# Chapter 7. Verification

Unsafe code enables low-level operations while circumventing Rust's safety guarantees, and may introduce undefined behavior (UB) if misused.
The verification module (`rapx::verify`) provides a staged pipeline for checking that unsafe call sites satisfy their callee's safety preconditions.

## Design Principles

The verification is grounded on three assumptions:

* **Origin of Unsafe Code:** All instances of undefined behavior arise from unsafe code.
* **Explicit Safety Properties:** Safety properties of unsafe code are documented through `#[rapx::requires(...)]` annotations on unsafe functions and `#[rapx::invariant(...)]` annotations on structs. See the [Rust Safety Standard](https://safer-rust.github.io/rust-safety-standard/rust-safety-standard.html#23-design-choices) for the underlying methodology.
* **Soundness of Unsafe Code Usage:** Unsafe code is considered safe if every possible execution path satisfies the safety properties of every unsafe callee it invokes.

## 7.1 Verification Modes

RAPx supports three verification modes, selectable via `cargo rapx verify --mode <MODE>`.

### `scan` Mode (Default)

Fully automatic. The `VerifyTargetCollector` (in `target.rs:64`) walks all function bodies in the crate using a HIR visitor. It applies a pre-filter (`hir_contains_unsafe` from `root.rs:38`):
- the function is declared `unsafe fn`, or
- the function body contains an `unsafe { }` block.

If the pre-filter passes, `build_function_target` collects all unsafe call sites from the MIR body and resolves each callee's safety contracts from `#[rapx::requires]` annotations or a bundled JSON database for standard library functions (`target.rs:91-122`).

```shell
cargo rapx verify --mode scan
```

### `targeted` Mode

Only verifies functions explicitly annotated with `#[rapx::verify]`. No pre-filter is applied; the collector simply checks for the attribute presence (`target.rs:172-189`).

```shell
cargo rapx verify --mode targeted
```

The annotation pattern:
```rust
#![feature(register_tool)]
#![register_tool(rapx)]

#[rapx::verify]
#[rapx::requires(ValidPtr(self.0, T, 1))]
unsafe fn my_function(ptr: *mut u32, len: usize) {
    unsafe { ptr::write(ptr.add(len - 1), 0); }
}
```

### `invless` Mode

Similar to `scan` but skips struct invariant checks. Instead, it chains constructors → mutable methods → read methods to derive implicit safety requirements (`driver.rs:194-223`). Each sequence propagates the constructor's `#[rapx::requires]` contracts through the mutator chain, filtering out contracts invalidated by field mutations.

```shell
cargo rapx verify --mode invless
```

### Mode Comparison

| Feature | `scan` | `targeted` | `invless` |
|---------|--------|------------|-----------|
| Annotation required | No | `#[rapx::verify]` | No |
| Scope | Functions with unsafe code or struct invariants | Annotated functions | Functions with unsafe code |
| Struct invariants | Checked | Checked | Ignored |
| Best for | Auditing unknown codebases | Focused verification | Discovering implicit safety requirements |

The `--prepare-targets` flag lists all verification targets and their safety contracts without running verification:
```shell
cargo rapx verify --prepare-targets
```

The `--allow-pathseg-repeat` flag controls how many extra SCC postfix repetitions are allowed during path enumeration (default 0, meaning each SCC postfix segment appears once; set to 1 for one extra repetition, etc.).

## 7.2 Verification Target Collection

The `VerifyTargetCollector` (`target.rs`) implements `rustc_hir::intravisit::Visitor` and visits every function body. For each function, it builds a `FunctionTarget` containing:

* **`def_id`**: The function's DefId.
* **`callsites`**: All call terminators whose callee is an `unsafe fn`, collected from MIR by `collect_unsafe_callsites` (`helpers.rs`). Each `Callsite` records the caller DefId, callee DefId, basic block, and argument operands.
* **`callee_requires`**: A `HashMap<DefId, Vec<Property>>` mapping each unsafe callee to its required safety properties, resolved from `#[rapx::requires]` annotations or the bundled `std-contracts.json` asset (`target.rs:91-122`).
* **`caller_requires`**: The caller's own `#[rapx::requires]` contracts, used as entry assumptions during verification.
* **`struct_invariants`**: If the function is a method on a struct with `#[rapx::invariant]` annotations, those invariants are included for struct-soundness checking.
* **`raw_ptr_deref_checks`**: Pseudo-callsites for raw pointer dereferences, paired with `ValidPtr`, `Align`, and (for reads) `Typed` properties (`target.rs:build_raw_ptr_deref_checks`).

**Free functions** are targets if they contain internal `unsafe` code. **Associated functions** are additionally targeted if their owning struct defines type invariants.

## 7.3 Safety Property Contracts

Safety properties are defined in `contract.rs` as the `PropertyKind` enum. Key property kinds include:

| Property | Arguments | Description |
|----------|-----------|-------------|
| `ValidPtr` | `(target, Ty, count)` | Pointer is valid for reads/writes of the given type and count |
| `Align` | `(target, Ty)` | Pointer satisfies the alignment of the given type |
| `InBound` | `(target, Ty, count)` | Pointer plus `count * size_of(Ty)` stays within the allocation |
| `Init` | `(target, Ty, count)` | Memory is initialized for the given type and count |
| `NonNull` | `(target)` | Pointer is not null |
| `Allocated` | `(target, Ty, count)` | Memory is allocated for the given type and count |
| `Typed` | `(target, Ty)` | Memory holds a valid value of the given type |
| `ValidNum` | `(predicates)` | Numeric constraints (e.g., `index < len`) |
| `Owning` | `(target)` | The value has unique ownership |
| `Deref` | `(target)` | The value is safe to dereference |

Contracts are expressed through `#[rapx::requires(...)]` attributes on function definitions. Each argument can be a place expression (e.g., `self.0`, `Arg_0`), a type, or a numeric expression. For example:

```rust
#[rapx::requires(ValidPtr(ptr, u32, 1))]
#[rapx::requires(Align(ptr, u32))]
#[rapx::requires(InBound(ptr, u32, len))]
pub unsafe fn write_slice(ptr: *mut u32, len: usize) {
    // ...
}
```

For standard library functions, contracts are maintained in a bundled JSON asset (`std-contracts.json`) parsed by `get_contract_from_entry` in `target.rs`. This avoids requiring annotations on upstream code.

## 7.4 The Verification Pipeline

The core verification flow is implemented in `driver.rs` as `VerifyDriver`, which orchestrates three stages:

```
VerifyDriver::verify_function()
  │
  ├─ 1. Path Extraction (path.rs)
  │     Build finite, acyclic paths from CFG to each callsite
  │     using SCC-aware enumeration (PathGraph).
  │
  ├─ 2. Backward Visit (path_refine/visitor.rs)
  │     Walk backward along the path from the callsite
  │     to function entry, collecting relevant MIR items.
  │
  ├─ 3. Forward Visit (forward_visit.rs)
  │     Walk forward along the path, building abstract state
  │     (values, facts) from the collected MIR items.
  │
  └─ 4. SMT Check (smt_check/)
        For each property kind, encode the forward facts and
        the property as SMT assertions and check satisfiability.
```

The engine is wrapped in `engine.rs` as `VerifyEngine`, whose `check_callsite` method runs the full pipeline for a single (callsite, path, property) triple. The `check_invariant` method handles struct invariant verification with constructor-specific logic.

### Path Extraction

`PathExtractor` (`path.rs`) builds finite verification paths from the function's CFG to each unsafe callsite. It uses `PathGraph` (from `analysis::path_analysis::graph`) to:

1. Build the CFG and detect strongly connected components (SCCs).
2. Enumerate acyclic paths through the CFG, allowing controlled SCC postfix repetition via `allow_pathseg_repeat`.
3. Filter paths that reach the target callsite block.
4. Truncate at the target block and deduplicate by prefix.

Each `Path` is a sequence of `PathStep::Block` steps ending with a `PathStep::Callsite`. Path enumeration is capped at `PATH_LIMIT` (1024).

### Backward Visit

The `BackwardVisitor` (`path_refine/visitor.rs`) starts from the callsite and walks backward along the path, collecting only MIR statements and terminators that are *relevant* to the property being checked. Items are classified as:

- **`Statement`**: A MIR statement whose def-use chain reaches the property's target place.
- **`Terminator`**: A MIR terminator (call, switch, assert) that influences control-flow constraints.
- **`ContractFact`**: A caller's `#[rapx::requires]` contract injected as an entry assumption.
- **`Forget`**: A conservatively dropped fact (e.g., after an unknown call).

The backward visitor uses def-use analysis (`def_use.rs`) to trace value flow from the callsite arguments back to their origins, filtering out irrelevant operations.

### Forward Visit

The `ForwardVisitor` (`forward_visit.rs`) walks forward along the retained MIR items, building an abstract state consisting of:

- **`values`**: A map from MIR locals to `AbstractValue` (Place, Ref, RawPtr, ConstInt, Cast, Binary op, CallResult, etc.).
- **`facts`**: A list of `StateFact` entries recording:
  - `PointsTo` — pointer-to-source relationships.
  - `Cast` — type cast information.
  - `KnownAligned` — alignment guarantees derived from operations or calls.
  - `KnownInit` / `KnownAllocated` — initialization and allocation status.
  - `KnownNonZero` / `KnownConst` — numeric constraints.
  - `BranchEq` — branch conditions from `SwitchInt` and `Assert` terminators.
  - `Call` / `CallEffect` — summarized call semantics from `call_summary.rs`.

The `call_summary.rs` module provides pre-defined effect summaries for standard library functions. For example, `ptr::read` is summarized as a `ReadMemory` effect; `ptr::write` as `WriteMemory`; `Vec::as_ptr` as `ReturnPointerFromArg { arg: 0 }`. These summaries are critical for tracking pointer properties across function boundaries without inter-procedural MIR analysis.

### SMT Check

The `SmtChecker` (`smt_check/`) encodes the forward state facts and the target property as SMT-LIB assertions and invokes a Z3 solver to check whether the facts logically imply the property. Each property kind has a dedicated checking module:

| Module | Checks |
|--------|--------|
| `valid_ptr.rs` | Pointer validity for the given type and count |
| `align.rs` | Pointer alignment matches the type requirement |
| `in_bound.rs` | Pointer arithmetic stays within allocation bounds |
| `init.rs` | Memory initialization state |
| `non_null.rs` | Pointer non-nullness |
| `allocated.rs` | Memory allocation tracking |
| `valid_num.rs` | Numeric range constraints |

The checker returns a `CheckResult::Proved` if the SMT solver confirms the property holds under all modeled assumptions, or `CheckResult::Unproved` otherwise.

### Call Effect Summaries

`call_summary.rs` maintains a hardcoded database of call effects for common unsafe functions. Each entry maps a function path to a `CallEffectSummary` with effects such as:

- `ReturnAliasArg` / `ReturnPointerFromArg` — the return value aliases an argument.
- `ReturnPointerAdd` / `ReturnPointerSub` — the return value is an offset of an argument (for `ptr::add`, `ptr::offset`, `ptr::sub`, `ptr::wrapping_add`, etc. and their `byte_*` variants).
- `ReturnAligned` — the return value has a known alignment.
- `WriteMemory` — the call writes to a pointed-to memory region (establishes `KnownInit`).
- `ReadMemory` — the call reads from a pointed-to memory region.
- `ReturnNonZero` — the return value is non-zero.
- `ReturnConst` — the return value is a known constant.
- `ForgetArgFacts` — the call invalidates previously known facts about an argument.

These summaries bridge the gap between intra-procedural path analysis and the semantics of standard library functions.

## 7.5 Struct Invariant Verification

When a function target is a method on a struct with `#[rapx::invariant(...)]` annotations, the verifier additionally checks that the struct invariants are maintained at every return point (for constructors) or assumed at entry and preserved at exit (for methods).

The `verify_struct_invariants` method in `driver.rs` handles this:

- For **constructors** (functions returning `Self`): Paths are filtered to return blocks. Invariants must be proved at each return checkpoint, with only the constructor's own `#[rapx::requires]` as entry assumptions.
- For **methods**: Invariants are assumed at function entry (prepended as `ContractFact` items) and checked at the end of every complete CFG path.

This analysis ensures that struct soundness is preserved across method sequences.

## 7.6 Verification Report

The `VerificationReport` (`report.rs`) aggregates results for all (callsite, path, property) triples. Each `PropertyCheckResult` records:

- The callsite location (`CallsiteLocation`: caller DefId + basic block).
- Path and property indices.
- The property being checked and the verification result (`Proved` / `Unproved`).
- Forward and backward visit diagnostics.

The `emit_verify_summary` function in `driver.rs` groups results by callsite and prints a compact summary:

```
============================================================
[rapx::verify] function: my_crate::my_function
============================================================
  --- unsafe callsites ---
      unsafe callsite: bb3 -> std::ptr::read
        path [0, 1, 3]:
          ValidPtr | Proved
          Align | Proved
          Typed | Proved
  result: SOUND
```

If any property is unproved, the function is reported as `UNSOUND` with a count of unproved properties.

## 7.7 Example: Verifying `Vec::into_raw_parts_with_alloc`

Consider `Vec::into_raw_parts_with_alloc` from the standard library. Its MIR contains a `ptr::read` call on the allocator field:

```
bb9:  _13 = &raw const (*_14)
      _12 = std::ptr::read::<A>(move _13)
```

The verifier collects this callsite, resolves `ptr::read`'s safety contracts (`ValidPtr`, `Align`, `Typed`), and verifies them along paths reaching bb9.

The `call_summary.rs` entry for `Vec::allocator` returns a `ReturnPointerFromArg` effect, establishing that `_14` points into the Vec's allocator field. The `alloc` field is always initialized (Rust's RAII guarantee), so:

- **ValidPtr**: The pointer returned by `Vec::allocator` points to live memory; the fact chain confirms non-null dereferenceability.
- **Align**: The pointer type matches the allocator type, satisfying alignment.
- **Typed**: The allocator maintains correct typing throughout the Vec's lifetime.

If the Vec struct had `#[rapx::invariant]` annotations, the verifier would additionally check construtor and method boundaries.

## 7.8 Invless Mode Sequences

In `invless` mode, when a method has constructors, the verifier generates verification sequences of the form:

```
Constructor → Method
Constructor → Mutator → Method
```

Each sequence propagates the constructor's `#[rapx::requires]` contracts through the mutator chain. Mutated fields are tracked via `get_mutated_fields` (`fn_info`), and contracts referencing those fields are filtered out.

For each sequence, the verifier builds a `FunctionTarget` with the accumulated `caller_requires`, then runs the standard pipeline with incremented `allow_pathseg_repeat`. Results are emitted per sequence with a chain label like `new_in -> push -> into_raw_parts_with_alloc`.

## 7.9 Limitations and Current Status

- Path enumeration is capped at 1024 paths per callsite.
- Inter-procedural analysis is limited to pre-defined call summaries; full cross-function verification is not supported.
- The SMT encoding approximates numeric constraints and may produce false positives for complex arithmetic.
- Struct invariant verification for method sequences is under active development.
- Safety contracts for standard library functions are maintained manually in `std-contracts.json` and may be incomplete.
