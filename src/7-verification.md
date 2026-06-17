# Chapter 7. Verification

Unsafe code enables low-level operations while circumventing Rust's safety guarantees, and may introduce undefined behavior (UB) if misused. The verification module (`rapx::verify`) provides a staged pipeline for checking that unsafe call sites satisfy their callee's safety preconditions.

## Design Principles

The verification is grounded on three assumptions:

* **Origin of Unsafe Code:** All instances of undefined behavior arise from unsafe code.
* **Explicit Safety Properties:** Safety properties of unsafe code are documented through `#[rapx::requires(...)]` annotations on unsafe functions and `#[rapx::invariant(...)]` annotations on structs. See the [Rust Safety Standard](https://safer-rust.github.io/rust-safety-standard/rust-safety-standard.html#23-design-choices) for the underlying methodology.
* **Soundness of Unsafe Code Usage:** Unsafe code is considered safe if every possible execution path satisfies the safety properties of every unsafe callee it invokes.

## 7.1 Verification Modes

RAPx supports three verification modes, selectable via `cargo rapx verify --mode <MODE>`.

### `scan` Mode (Default)

Fully automatic. The `VerifyTargetCollector` (`target.rs`) walks all function bodies in the crate using a HIR visitor. It applies a pre-filter (`hir_contains_unsafe`):
- the function is declared `unsafe fn`, or
- the function body contains an `unsafe { }` block, or
- the function is a method on a struct with `#[rapx::invariant]` annotations.

If the pre-filter passes, `build_function_target` collects all unsafe call sites from the MIR body and resolves each callee's safety contracts from `#[rapx::requires]` annotations or a bundled JSON database for standard library functions (`std-contracts.json`).

```shell
cargo rapx verify --mode scan
```

### `targeted` Mode

Only verifies functions explicitly annotated with `#[rapx::verify]`. No pre-filter is applied; the collector simply checks for the attribute presence.

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

Similar to `scan` but skips struct invariant checks. Instead, it chains constructors → mutable methods → read methods to derive implicit safety requirements. Each sequence propagates the constructor's `#[rapx::requires]` contracts through the mutator chain, filtering out contracts invalidated by field mutations.

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

The `--allow-pathseg-repeat` flag controls how many extra SCC postfix repetitions are allowed during path enumeration (default 0, meaning each SCC postfix segment appears once; set to 1 for one extra repetition, etc.). The `VerifyRun` analysis automatically increments this value across multiple rounds (0, 1, 2, ...) when earlier rounds fail, up to a maximum of `MAX_REPEAT` (3).

## 7.2 Verification Target Collection

The `VerifyTargetCollector` (`target.rs`) implements `rustc_hir::intravisit::Visitor` and visits every function body. For each function, it builds a `FunctionTarget` containing:

* **`def_id`**: The function's DefId.
* **`callsites`**: All call terminators whose callee is an `unsafe fn`, collected from MIR by `collect_unsafe_callsites` (`helpers.rs`). Each `Callsite` records the caller DefId, callee DefId, basic block, and argument operands.
* **`callee_requires`**: A `HashMap<DefId, Vec<Property>>` mapping each unsafe callee to its required safety properties, resolved from `#[rapx::requires]` annotations or the bundled `std-contracts.json` asset.
* **`caller_requires`**: The caller's own `#[rapx::requires]` contracts, used as entry assumptions during verification.
* **`struct_invariants`**: If the function is a method on a struct with `#[rapx::invariant]` annotations, those invariants are included for struct-soundness checking.
* **`raw_ptr_deref_checks`**: Pseudo-callsites for raw pointer dereferences (`*ptr` and `*ptr = val` in unsafe blocks), paired with `ValidPtr`, `Align`, and (for reads) `Typed` properties. These are built by `build_raw_ptr_deref_checks()` which scans the MIR for `PlaceElem::Deref` projections on raw pointer types.

**Free functions** are targets if they contain internal `unsafe` code. **Associated functions** are additionally targeted if their owning struct defines type invariants.

### Contract Resolution

Contracts are resolved through a two-tier system:

1. **Annotation-based**: `get_contract_from_annotation()` parses `#[rapx::requires(...)]` attributes from the callee's source code, constructing `Property` instances via `Property::new()`.
2. **JSON fallback**: For standard library functions (no annotations in upstream code), `get_contract_from_entry()` builds properties from the bundled `std-contracts.json` database, normalizing argument tokens (`arg:N`, `const:N`, `ty:T`) into Rust expression syntax.

## 7.3 Safety Property Contracts

Safety properties are defined in `contract.rs` as the `PropertyKind` enum. Each `Property` consists of a `PropertyKind` and a vector of `PropertyArg` values.

### Property Kinds

| Property | Arguments | Description |
|----------|-----------|-------------|
| `ValidPtr` | `(target, Ty, count)` | Pointer is valid for reads/writes of the given type and count |
| `Align` | `(target, Ty)` | Pointer satisfies the alignment of the given type |
| `InBound` | `(target, Ty, count)` | Pointer plus `count * size_of(Ty)` stays within the allocation |
| `Init` | `(target, Ty, count)` | Memory is initialized for the given type and count |
| `NonNull` | `(target)` | Pointer is not null |
| `Allocated` | `(target, Ty, count)` | Memory is allocated for the given type and count |
| `Typed` | `(target, Ty)` | Memory holds a valid value of the given type |
| `ValidNum` | `(predicates)` | Numeric constraints (e.g., `index < len`, `offset <= cap`) |
| `Owning` | `(target)` | The value has unique ownership |
| `Deref` | `(target)` | The value is safe to dereference |
| `Ptr2Ref` | `(target)` | A raw pointer can be safely converted to a reference |
| `Layout` | `(target)` | A memory layout value matches the given size and alignment |
| `Size` | `(target, size_expr)` | A value matches the expected size |
| `Unknown` | `(tag)` | A contract tag not yet supported by the verifier |

### Property Arguments

`PropertyArg` is polymorphic:

- **`Place(ContractPlace)`**: A program place expression (e.g., `self.0`, `Arg_0`, `_1.len`), with field projections.
- **`Ty(Ty<'tcx>)`**: A Rust type used for size/alignment computations.
- **`Expr(ContractExpr)`**: A numeric expression involving constants, size_of, align_of, arithmetic, and place values.
- **`Predicates(Vec<NumericPredicate>)`**: For `ValidNum`, a list of relational constraints (`lhs op rhs`) forming a numeric interval.
- **`Ident(String)`**: An unresolved identifier.

### Contract Expressions

`ContractExpr` models numeric computations appearing in contract arguments:

```rust
pub enum ContractExpr {
    Place(ContractPlace),         // A place value
    Const(u128),                  // A literal constant
    SizeOf(Ty<'tcx>),            // size_of::<T>()
    AlignOf(Ty<'tcx>),           // align_of::<T>()
    Binary(NumericOp, Box<ContractExpr>, Box<ContractExpr>),
    Unary(NumericUnaryOp, Box<ContractExpr>),
    Unknown(String),
}
```

### ValidNum Predicates

`ValidNum` contracts use `NumericPredicate` to express range constraints:

```rust
pub struct NumericPredicate {
    pub lhs: ContractExpr,
    pub op: RelOp,          // Eq, Ne, Lt, Le, Gt, Ge
    pub rhs: ContractExpr,
}
```

For example, `#[rapx::requires(ValidNum(index < len))]` parses into a predicate `index < len`, which the SMT checker encodes as an SMT-LIB assertion.

### Contract Annotation Syntax

Contracts are expressed through `#[rapx::requires(...)]` attributes:

```rust
#[rapx::requires(ValidPtr(ptr, u32, 1))]
#[rapx::requires(Align(ptr, u32))]
#[rapx::requires(InBound(ptr, u32, len))]
#[rapx::requires(ValidNum(index < len))]
pub unsafe fn write_slice(ptr: *mut u32, len: usize, index: usize) {
    // ...
}
```

## 7.4 The Verification Pipeline

The core verification flow is implemented across three stages orchestrated by `VerifyEngine` (`engine.rs`) and driven by `VerifyDriver` (`driver.rs`):

```
VerifyDriver::verify_function()
  │
  ├─ 1. Path Extraction (path.rs / PathGraph)
  │     Build finite, acyclic paths from CFG to each callsite
  │     using SCC-aware enumeration (PathGraph — Chapter 5.1).
  │     Stored in a FunctionPaths per-callsite PathTree.
  │
  └─ 2. Callsite Verification (engine.rs)
        │
        ├─ 2a. Backward Visit (path_refine/visitor.rs)
        │     Walk backward from the callsite along paths,
        │     collecting MIR items relevant to the property.
        │     Uses def-use chains + call dependency summaries.
        │
        ├─ 2b. Forward Visit (forward_visit.rs)
        │     Walk forward along retained MIR items,
        │     building abstract state (values, facts).
        │     Applies call effect summaries.
        │
        └─ 2c. SMT Check (smt_check/)
              Encode forward facts + property as SMT assertions.
              Invoke Z3 solver to check logical entailment.
              Returns Proved or Unproved.
```

### 7.4.1 Path Extraction

`PathExtractor` (`path.rs`) builds finite verification paths from the function's CFG to each unsafe callsite. It uses `PathGraph` (from `analysis::path_analysis::graph`) to:

1. Build the CFG and detect strongly connected components (SCCs).
2. Enumerate acyclic paths through the CFG, allowing controlled SCC postfix repetition via `allow_pathseg_repeat`.
3. Filter paths that reach the target callsite block.
4. Truncate at the target block and deduplicate by prefix, storing them in a `PathTree` per callsite.

Each `Path` is a sequence of `PathStep::Block` steps ending with a `PathStep::Callsite`. Path enumeration is capped at `PATH_LIMIT` (1024) per callsite.

### 7.4.2 Backward Visit

The `BackwardVisitor` (`path_refine/visitor.rs`) starts from the callsite and walks backward along the path, collecting only MIR statements and terminators that are *relevant* to the property being checked. Items are classified as:

- **`Statement`**: A MIR statement whose def-use chain reaches the property's target place.
- **`Terminator`**: A MIR terminator (call, switch, assert) that influences control-flow constraints.
- **`ContractFact`**: A caller's `#[rapx::requires]` contract injected as an entry assumption.
- **`Forget`**: A conservatively dropped fact (e.g., after an unknown call).

The backward visitor uses def-use analysis (`def_use.rs`) to trace value flow from the callsite arguments back to their origins, filtering out irrelevant operations. It also consults `CallDependencySummary` (from `call_summary.rs`) to determine which callee arguments the return value depends on.

### 7.4.3 Forward Visit

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

### 7.4.4 SMT Check

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

### 7.4.5 Call Effect Summaries

`call_summary.rs` maintains a database of call effects for common unsafe and standard library functions. Each entry maps a function path to a `CallEffectSummary` with effects such as:

| Effect | Description | Examples |
|--------|-------------|----------|
| `ReturnAliasArg` | Return value aliases an argument | `NonNull::as_ptr`, `MaybeUninit::as_ptr` |
| `ReturnPointerFromArg` | Return value is a pointer to an argument | `Vec::as_ptr`, `Box::as_mut_ptr`, `alloc::alloc` |
| `ReturnPointerAdd` | Return value = base + offset × stride | `ptr::add`, `ptr::offset(i)`, `ptr::wrapping_add`, `ptr::byte_add` |
| `ReturnPointerSub` | Return value = base − offset × stride | `ptr::sub`, `ptr::wrapping_sub`, `ptr::byte_sub` |
| `ReturnNonZero` | Return value is guaranteed non-zero | `NonNull::new_unchecked` |
| `ReturnAligned` | Return value has a known alignment | `alloc::alloc`, memory allocation functions |
| `ReturnConst` | Return value is a known constant | `size_of`, `align_of` |
| `ReturnLengthOfArg` | Return value = length of an argument slice | `Vec::len`, `slice::len` |
| `ReadMemory` | Call reads from a pointed-to region | `ptr::read`, `ptr::copy_nonoverlapping` (src) |
| `WriteMemory` | Call writes to a pointed-to region (establishes `KnownInit`) | `ptr::write`, `ptr::copy_nonoverlapping` (dst) |
| `ForgetArgFacts` | Call invalidates previously known facts about an argument | `ptr::swap`, functions that mutate through a pointer |

These summaries bridge the gap between intra-procedural path analysis and the semantics of standard library functions, avoiding the need for full inter-procedural MIR analysis of every unsafe callee.

For backward analysis, `dependency_summary()` provides `CallDependencySummary` with `return_depends_on_args` (which args the return value depends on) and `may_write_args` (which args may be mutated). For forward analysis, `effect_summary()` provides `CallEffectSummary` with the concrete effects to apply to the abstract state.

## 7.5 Struct Invariant Verification

When a function target is a method on a struct with `#[rapx::invariant(...)]` annotations, the verifier additionally checks that the struct invariants are maintained:

- **Constructors** (functions returning `Self`): Paths are filtered to return blocks. Invariants must be proved at each return checkpoint, with only the constructor's own `#[rapx::requires]` as entry assumptions.
- **Methods**: Invariants are assumed at function entry (prepended as `ContractFact` items in the backward visitor) and verified at all return points.

The `verify_struct_invariants` method in `driver.rs` handles this via `build_invariant_paths()`:

- For constructors: prefixes paths to return blocks with the constructor's contracts.
- For methods: builds whole-CFG paths with invariants assumed at entry and checked at every return block.

This analysis ensures that struct soundness is preserved across method sequences — invariants established by the constructor are maintained through mutations and respected by accessors.

## 7.6 Invless Mode Sequences

In `invless` mode (`run_invless_sequences` in `driver.rs`), when a struct has constructors and methods, the verifier generates verification sequences:

```
Constructor → Method
Constructor → Mutator → Method
Constructor → Mutator₁ → Mutator₂ → Method
```

Each sequence propagates the constructor's `#[rapx::requires]` contracts through the mutator chain. Mutated fields are tracked via `get_mutated_fields`, and contracts referencing those fields are filtered out at each step.

For each sequence, the verifier builds a `FunctionTarget` with the accumulated `caller_requires`, then runs the standard pipeline with incremented `allow_pathseg_repeat`. Results are emitted per sequence with a chain label like `new_in → push → into_raw_parts_with_alloc`.

## 7.7 Raw Pointer Dereference Checking

In addition to explicit unsafe call sites, the verifier checks raw pointer dereferences (`*ptr` and `*ptr = val`) in unsafe blocks. `build_raw_ptr_deref_checks()` (`target.rs`) scans the MIR for dereference projections on raw pointer types and creates pseudo-callsites with the following properties:

- **Read dereferences** (`*ptr`): Requires `ValidPtr(ptr, T, 1)`, `Align(ptr, T)`, and `Typed(ptr, T)`.
- **Write dereferences** (`*ptr = val`): Requires `ValidPtr(ptr, T, 1)` and `Align(ptr, T)`.

These pseudo-callsites are integrated into the same `callsite → property` verification pipeline, so they benefit from the same path sensitivity and abstract state tracking as regular unsafe calls.

## 7.8 Verification Report

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

## 7.9 Example: Verifying `Vec::into_raw_parts_with_alloc`

Consider `Vec::into_raw_parts_with_alloc` from the standard library. Its MIR contains a `ptr::read` call on the allocator field:

```
bb9:  _13 = &raw const (*_14)
      _12 = std::ptr::read::<A>(move _13) -> bb10
```

The verifier collects this callsite, resolves `ptr::read`'s safety contracts (`ValidPtr`, `Align`, `Typed`), and verifies them along paths reaching bb9.

The `call_summary.rs` entry for `Vec::allocator` returns a `ReturnPointerFromArg` effect, establishing that `_14` points into the Vec's allocator field. The `alloc` field is always initialized (Rust's RAII guarantee), so:

- **ValidPtr**: The pointer returned by `Vec::allocator` points to live memory; the fact chain confirms non-null dereferenceability.
- **Align**: The pointer type matches the allocator type, satisfying alignment.
- **Typed**: The allocator maintains correct typing throughout the Vec's lifetime.

If the Vec struct had `#[rapx::invariant]` annotations, the verifier would additionally check constructor and method boundaries.

## 7.10 Limitations and Current Status

- Path enumeration is capped at 1024 paths per callsite.
- Inter-procedural analysis is limited to pre-defined call summaries; full cross-function MIR-level verification is not supported.
- The SMT encoding approximates numeric constraints and may produce false positives for complex arithmetic.
- Struct invariant verification for method sequences is under active development.
- Safety contracts for standard library functions are maintained manually in `std-contracts.json` and may be incomplete.
- The verifier supports a fixed set of `PropertyKind` variants; unknown contract tags are classified as `Unknown` and produce `CheckResult::Unknown`.
