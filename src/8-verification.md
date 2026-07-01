# Chapter 8. Verification

Unsafe code enables low-level operations while circumventing Rust's safety guarantees, and may introduce undefined behavior (UB) if misused. The verification module ([`rapx::verify`](https://github.com/safer-rust/RAPx/tree/main/rapx/src/verify/)) provides a staged pipeline for checking that unsafe call sites satisfy their callee's safety preconditions. It combines path-sensitive MIR traversal, abstract interpretation, and Z3-based SMT solving to produce verdicts of **Proved** or **Unproved** for each safety property at each unsafe call site.

## 8.1 Design Principles

The verification is grounded on three assumptions:

* **Origin of Unsafe Code:** All instances of undefined behavior arise from unsafe code — safe Rust's type system already rules out data races, use-after-free, null-pointer dereferences, and buffer overflows. Verification focuses exclusively on the `unsafe` boundary where the compiler's guarantees end.
* **Explicit Safety Properties:** Safety properties of unsafe code are documented through `#[rapx::requires(...)]` annotations on unsafe functions and `#[rapx::invariant(...)]` annotations on structs. See the [Rust Safety Standard](https://safer-rust.github.io/rust-safety-standard/rust-safety-standard.html#23-design-choices) for the underlying methodology. Each annotation declares a contract that callers must uphold, making safety obligations machine-checkable rather than implicit in documentation comments.
* **Soundness of Unsafe Code Usage:** Unsafe code is considered safe if every possible execution path satisfies the safety properties of every unsafe callee it invokes. If a path exists that would violate even one property, the call site is flagged as unproved.

### 8.1.1 Contract-Based vs. Full Functional Verification

RAPx's verification is **contract-based** rather than full functional verification. This means:

1. Callees declare their *preconditions* (what must hold before the call) via `#[rapx::requires]`.
2. The verifier checks that the caller's abstract state at the call site entails these preconditions.
3. Callees do not need to declare their *postconditions* — the caller's state after the call is inferred from a library of pre-defined call effect summaries.

This design trades full functional correctness for scalability: it avoids the need to annotate every function with pre/post-conditions, and it handles standard library calls through a curated database rather than through inter-procedural MIR analysis.

## 8.2 Verification Modes

RAPx supports three verification modes, selectable via `cargo rapx verify --mode <MODE>`. Each mode targets a different use case in the development workflow.

### 8.2.1 `scan` Mode (Default)

Fully automatic. The `VerifyTargetCollector` ([`target.rs`](https://github.com/safer-rust/RAPx/blob/main/rapx/src/verify/target.rs)) walks all function bodies in the crate using a HIR visitor. It applies a pre-filter (`hir_contains_unsafe`):

- the function is declared `unsafe fn`, or
- the function body contains an `unsafe { }` block, or
- the function is a method on a struct with `#[rapx::invariant]` annotations.

If the pre-filter passes, `build_function_target` collects all unsafe call sites from the MIR body and resolves each callee's safety contracts from `#[rapx::requires]` annotations or a bundled JSON database for standard library functions (`std-contracts.json`).

```shell
cargo rapx verify --mode scan
```

**Use case:** Auditing an unknown or untrusted codebase where you don't know in advance which functions contain unsafe code. `scan` mode discovers all unsafe call sites automatically and checks them against available contracts.

**What you'll see:** For each function containing unsafe code, the verifier emits a summary showing every unsafe call site, the paths reaching it, and the Proved/Unproved status of each required property.

### 8.2.2 `targeted` Mode

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

**Use case:** Focused verification during development. You annotate the function you're currently working on, and the verifier checks only that function, ignoring the rest of the crate. This keeps verification fast and focused.

**Workflow:** Annotate the function with `#[rapx::verify]`, run `cargo rapx verify --mode targeted`, inspect results, fix any unproved properties, and repeat. When all properties are Proved, the function is considered sound.

### 8.2.3 `invless` Mode

Similar to `scan` but skips struct invariant checks. Instead, it chains constructors → mutable methods → read methods to derive implicit safety requirements. Each sequence propagates the constructor's `#[rapx::requires]` contracts through the mutator chain, filtering out contracts invalidated by field mutations.

```shell
cargo rapx verify --mode invless
```

**Use case:** Discovering implicit safety requirements when struct invariants are not explicitly annotated. Useful for legacy code or libraries where you haven't yet written `#[rapx::invariant]` annotations but want to understand the de facto safety contracts implied by construction and method chaining.

**How it works in detail:**

1. The verifier identifies all constructors (functions returning `Self`) and all methods (`&self` and `&mut self` functions) on each struct.
2. For each constructor, it extracts the constructor's own `#[rapx::requires]` contracts and begins a chain.
3. It appends each `&mut self` method to the chain, tracking which fields are mutated by the method body.
4. At each step, it filters out contracts that reference mutated fields (since the mutation may invalidate the contract).
5. It terminates the chain at each `&self` method, forming a complete sequence like `Constructor → Mutator₁ → Mutator₂ → Reader`.
6. Each sequence is verified as a single `FunctionTarget` with the accumulated contracts as entry assumptions.

### 8.2.4 Mode Comparison

| Feature | `scan` | `targeted` | `invless` |
|---------|--------|------------|-----------|
| Annotation required | No | `#[rapx::verify]` | No |
| Scope | Functions with unsafe code or struct invariants | Annotated functions | Functions with unsafe code |
| Struct invariants | Checked | Checked | Ignored |
| Implicit sequences | No | No | Constructor → Method chains |
| Best for | Auditing unknown codebases | Focused verification | Discovering implicit safety requirements |

### 8.2.5 Common Options

The `--prepare-targets` flag lists all verification targets and their safety contracts without running verification:

```shell
cargo rapx verify --prepare-targets
```

Output example:

```
[rapx::verify] prepare targets: function my_crate::my_function
  unsafe callsite: bb3 -> std::ptr::read
    contract: ValidPtr(ptr, u32, 1)
    contract: Align(ptr, u32)
    contract: Typed(ptr, u32)
```

This is useful for understanding which contracts the verifier will attempt to prove before running the full (potentially time-consuming) verification.

The `--postfix-repeat` flag controls how many extra SCC postfix repetitions are allowed during path enumeration. If not explicitly set, the verifier automatically expands `postfix_repeat` until convergence — deeper loop unrolling continues until no new information is gained. The auto-expansion terminates immediately when any property becomes `Unproved` or `Unknown`: once a violation is discovered, further unrolling serves no purpose. When explicitly set via `--postfix-repeat=N`, auto-expansion is disabled and the verifier uses exactly N repeat values. The full convergence mechanism is described in [8.5.1 Path Extraction](#851-path-extraction).

## 8.3 Verification Target Collection

The `VerifyTargetCollector` ([`target.rs`](https://github.com/safer-rust/RAPx/blob/main/rapx/src/verify/target.rs)) implements `rustc_hir::intravisit::Visitor` and visits every function body. For each function, it builds a `FunctionTarget` containing:

* **`def_id`**: The function's DefId.
* **`callsites`**: All call terminators whose callee is an `unsafe fn`, collected from MIR by `collect_unsafe_callsites` ([`helpers.rs`](https://github.com/safer-rust/RAPx/blob/main/rapx/src/verify/helpers.rs)). Each `Callsite` records the caller DefId, callee DefId, basic block, and argument operands.
* **`callee_requires`**: A `HashMap<DefId, Vec<Property>>` mapping each unsafe callee to its required safety properties, resolved from `#[rapx::requires]` annotations or the bundled `std-contracts.json` asset.
* **`caller_requires`**: The caller's own `#[rapx::requires]` contracts, used as entry assumptions during verification.
* **`struct_invariants`**: If the function is a method on a struct with `#[rapx::invariant]` annotations, those invariants are included for struct-soundness checking.
* **`raw_ptr_deref_checks`**: Pseudo-callsites for raw pointer dereferences (`*ptr` and `*ptr = val` in unsafe blocks), paired with `ValidPtr`, `Align`, and (for reads) `Typed` properties. These are built by `build_raw_ptr_deref_checks()` which scans the MIR for `PlaceElem::Deref` projections on raw pointer types.

**Free functions** are targets if they contain internal `unsafe` code. **Associated functions** are additionally targeted if their owning struct defines type invariants.

### 8.3.1 The Collection Pipeline in Detail

```
VerifyTargetCollector::visit_fn(fn_kind, def_id)
  │
  ├─ 1. Pre-filter
  │     - hir_contains_unsafe: checks HIR for unsafe blocks/fn declarations
  │     - struct_invariants check: is this a method on a struct with invariants?
  │     - If neither passes → skip
  │
  ├─ 2. Build callsites
  │     - collect_unsafe_callsites: scan MIR terminators for unsafe fn calls
  │     - For each callsite: record (callee DefId, basic block, args)
  │
  ├─ 3. Build raw pointer checks
  │     - build_raw_ptr_deref_checks: scan MIR for *ptr and *ptr = val
  │     - Create pseudo-callsites with implicit contracts
  │
  ├─ 4. Resolve contracts
  │     - For each callee DefId:
  │        a. get_contract_from_annotation: parse #[rapx::requires] from source
  │        b. If not found: get_contract_from_entry: load from std-contracts.json
  │
  ├─ 5. Build struct invariants (if applicable)
  │     - Parse #[rapx::invariant] annotations on the struct
  │
  └─ 6. Return FunctionTarget
```

### 8.3.2 Contract Resolution

Contracts are resolved through a two-tier system:

1. **Annotation-based**: `get_contract_from_annotation()` parses `#[rapx::requires(...)]` attributes from the callee's source code, constructing `Property` instances via `Property::new()`. This is the primary mechanism for user-defined unsafe functions.

2. **JSON fallback**: For standard library functions (no annotations in upstream code), `get_contract_from_entry()` builds properties from the bundled `std-contracts.json` database, normalizing argument tokens (`arg:N`, `const:N`, `ty:T`) into Rust expression syntax.

The resolution order is: annotations first, then JSON. If neither provides a contract, the callee is skipped (no properties to verify). This means **unannotated third-party unsafe functions are not verified** — you must either annotate them or add entries to a custom contracts database.

### 8.3.3 std-contracts.json Format

The standard library contract database maps fully-qualified function paths to lists of contracts. Each entry uses a compact token form that is normalized during resolution:

```json
{
  "std::ptr::read": [
    "ValidPtr(arg:0, ty:T0, const:1)",
    "Align(arg:0, ty:T0)",
    "Typed(arg:0, ty:T0)"
  ],
  "std::ptr::write": [
    "ValidPtr(arg:0, ty:T0, const:1)",
    "Align(arg:0, ty:T0)"
  ],
  "std::ptr::add": [
    "ValidPtr(arg:0, ty:T0, const:1)",
    "Align(arg:0, ty:T0)"
  ]
}
```

The normalization step replaces:
- `arg:N` → the Nth function argument
- `ty:TN` → the Nth generic type parameter
- `const:N` → a literal constant N

## 8.4 Safety Property Contracts

Safety properties are defined in [`contract.rs`](https://github.com/safer-rust/RAPx/blob/main/rapx/src/verify/contract.rs) as the `PropertyKind` enum. Each `Property` consists of a `PropertyKind` and a vector of `PropertyArg` values.

### 8.4.1 Property Kinds

| Property | Arguments | Description |
|----------|-----------|-------------|
| `ValidPtr` | `(target, Ty, count)` | Pointer is valid for reads/writes of the given type and count |
| `Align` | `(target, Ty)` | Pointer satisfies the alignment of the given type |
| `InBound` | `(target, Ty, count)` or `(IndexAccess)` | Pointer offset/index stays within the allocation bounds |
| `Init` | `(target, Ty, count)` | Memory is initialized for the given type and count |
| `NonNull` | `(target)` | Pointer is not null |
| `Allocated` | `(target, Ty, count)` or `(target)` | Memory is allocated for the given type and count |
| `Typed` | `(target, Ty)` | Memory holds a valid value of the given type |
| `ValidNum` | `(predicates)` or `(predicate)` | Numeric constraints (e.g., `index < len`, `offset <= cap`, `n != 0`) |
| `Owning` | `(target)` | The value has unique ownership |
| `Deref` | `(target)` or `(target, Ty, count)` | The pointer can be safely dereferenced; composite of `Allocated` + `InBound` |
| `Ptr2Ref` | `(target)` | A raw pointer can be safely converted to a reference |
| `Layout` | `(target)` | A memory layout value (e.g., `Layout`) matches a prior allocation's size and alignment |
| `Size` | `(target)` | The value has valid DST (dynamically-sized type) metadata |
| `NonSize` | *(none)* | The type is `Sized` (not a DST), required for pointer offset arithmetic with generic types |
| `NonOverlap` | `(targets...)` | The specified memory ranges do not overlap |
| `NoPadding` | `(target)` | The type has no padding bytes |
| `ValidString` | `(target)` | String/slice data is valid UTF-8 |
| `ValidCStr` | `(target)` | C string is valid (null-terminated, no interior nulls) |
| `ValidSlice` | `(target)` or `(target, Ty)` | A slice reference (pointer + length) is valid |
| `Unwrap` | `(target)` | The `Option`/`Result` is in the expected variant (e.g., `Some` or `Ok`) |
| `ValidTransmute` | `(Src, Dst)` | Transmuting from type `Src` to type `Dst` is safe |
| `Alias` | `(target)` | No other live pointer aliases this memory (exclusive access) |
| `Alive` | `(target)` | The allocation is still live (not freed) |
| `Pinned` | `(target)` | The target is pinned (its memory address will not change) |
| `NonVolatile` | `(target)` | Memory is not volatile / not externally modified |
| `Opened` | `(target)` | An OS resource (e.g., file descriptor) is valid and open |
| `Trait` | `(target)` | Trait object safety contract |
| `Unreachable` | *(none)* | The code path is unreachable |
| `Null` | `(target)` | The pointer may be null (safe for operations like `as_ref` that handle null) |
| `Unknown` | `(tag)` | A contract tag not yet supported by the verifier |

### 8.4.2 When Each Property Applies

Different unsafe operations require different combinations of properties:

| Unsafe Operation | Required Properties |
|-----------------|-------------------|
| `ptr::read(ptr)` | `ValidPtr(ptr, T, 1)`, `Align(ptr, T)`, `Typed(ptr, T)` |
| `ptr::write(ptr, val)` | `ValidPtr(ptr, T, 1)`, `Align(ptr, T)` |
| `ptr::copy(src, dst, count)` | `ValidPtr(src, T, count)`, `Align(src, T)`, `ValidPtr(dst, T, count)`, `Align(dst, T)` |
| `ptr::copy_nonoverlapping(src, dst, count)` | Same as `ptr::copy`, plus `NonOverlap(src, dst)` |
| `*raw_ptr` (read) | `ValidPtr(ptr, T, 1)`, `Align(ptr, T)`, `Typed(ptr, T)` |
| `*raw_ptr = val` (write) | `ValidPtr(ptr, T, 1)`, `Align(ptr, T)` |
| `NonNull::new_unchecked(ptr)` | `NonNull(ptr)` |
| `slice::from_raw_parts(ptr, len)` | `ValidPtr(ptr, T, len)`, `Align(ptr, T)` |
| `Vec::from_raw_parts(ptr, len, cap)` | `ValidPtr(ptr, T, cap)`, `Align(ptr, T)`, `Allocated(ptr, T, cap)` |
| `Box::from_raw(ptr)` | `ValidPtr(ptr, T, 1)`, `Align(ptr, T)`, `Allocated(ptr, T, 1)` |
| `str::from_utf8_unchecked(v)` | `ValidString(v)` — the byte slice must be valid UTF-8 |
| `CStr::from_ptr(ptr)` | `ValidCStr(ptr)`, `ValidPtr(ptr, c_char, 1)`, `NonNull(ptr)` |
| `Pin::new_unchecked(ptr)` | `Pinned(ptr)` — the pointee must not be moved |
| `MaybeUninit::assume_init()` | `Init(self, T, 1)` — the memory must be initialized |
| `transmute::<Src, Dst>(src)` | `ValidTransmute(Src, Dst)` — bitwise reinterpretation must be valid |
| `ptr::add(ptr, count)` / `ptr::offset` | `InBound(ptr, T, count)`, `NonNull(ptr)`, `Align(ptr, T)` |
| GlobalAlloc/Allocator methods | `Allocated(ptr, u8, layout.size())`, `Layout(layout)` — memory must match the given layout |
| `std::os::unix::io::from_raw_fd(fd)` | `Opened(fd)` — the file descriptor must be valid and open |
| `hint::unreachable_unchecked()` | `Unreachable` — the code path must actually be unreachable |

### 8.4.3 Property Arguments

`PropertyArg` is polymorphic:

- **`Place(ContractPlace)`**: A program place expression (e.g., `self.0`, `Arg_0`, `_1.len`), with field projections. These use MIR place syntax to refer to function arguments and struct fields.
- **`Ty(Ty<'tcx>)`**: A Rust type used for size/alignment computations. For generic functions, this is the monomorphized type resolved at the call site.
- **`Expr(ContractExpr)`**: A numeric expression involving constants, size_of, align_of, arithmetic, and place values.
- **`Predicates(Vec<NumericPredicate>)`**: For `ValidNum`, a list of relational constraints (`lhs op rhs`) forming a numeric interval or relation.
- **`Ident(String)`**: An unresolved identifier, used during parsing before resolution.

### 8.4.4 Contract Expressions

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

Examples of contract expressions:

```
size_of::<T>()                    → SizeOf(T)
size_of::<T>() * len              → Binary(Mul, SizeOf(T), Place(len))
align_of::<T>() + 1               → Binary(Add, AlignOf(T), Const(1))
(len - 1) * size_of::<u32>()      → Binary(Mul, Binary(Sub, Place(len), Const(1)), SizeOf(u32))
```

### 8.4.5 ValidNum Predicates

`ValidNum` contracts use `NumericPredicate` to express range constraints:

```rust
pub struct NumericPredicate {
    pub lhs: ContractExpr,
    pub op: RelOp,          // Eq, Ne, Lt, Le, Gt, Ge
    pub rhs: ContractExpr,
}
```

For example, `#[rapx::requires(ValidNum(index < len))]` parses into a predicate `index < len`, which the SMT checker encodes as an SMT-LIB assertion:

```smt2
(assert (< index len))
```

Multiple predicates in a single `ValidNum` combine conjunctively:

```rust
#[rapx::requires(ValidNum(index < len, index >= 0, offset <= cap))]
```

This asserts that `index` is in the range `[0, len)` and `offset` is at most `cap`. The SMT encoding converts these to three separate assertions.

### 8.4.6 Contract Annotation Syntax

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

Each `#[rapx::requires]` attribute contains one or more comma-separated property declarations. You can use multiple attributes or group properties in a single attribute:

```rust
// Multiple attributes (equivalent to the grouped form below)
#[rapx::requires(ValidPtr(ptr, u32, 1))]
#[rapx::requires(Align(ptr, u32))]

// Grouped in a single attribute
#[rapx::requires(ValidPtr(ptr, u32, 1), Align(ptr, u32))]
```

### 8.4.7 Struct Invariant Annotations

Struct-level invariants use `#[rapx::invariant(...)]`:

```rust
#[rapx::invariant(ValidPtr(self.ptr, T, 1))]
#[rapx::invariant(Align(self.ptr, T))]
pub struct MyBuf<T> {
    ptr: *mut T,
    len: usize,
    allocated: usize,
}
```

These invariants must hold at all times for any instance of the struct. The verifier checks that:
1. Constructors establish all invariants before returning.
2. Mutating methods preserve all invariants.
3. Methods that take `&self` can assume invariants hold on entry.

## 8.5 The Verification Pipeline

The core verification flow is implemented across three stages orchestrated by `VerifyEngine` ([`engine.rs`](https://github.com/safer-rust/RAPx/blob/main/rapx/src/verify/engine.rs)) and driven by `VerifyDriver` ([`driver.rs`](https://github.com/safer-rust/RAPx/blob/main/rapx/src/verify/driver.rs)):

```
VerifyDriver::verify_function()
  │
  ├─ 1. Path Extraction (path.rs / PathGraph)
  │     Build finite, acyclic paths from CFG to each callsite
  │     using SCC-aware enumeration (PathGraph — [Chapter 5.1](./5.1-path.md)).
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

### 8.5.1 Path Extraction

[`PathExtractor`](https://github.com/safer-rust/RAPx/blob/main/rapx/src/verify/path_extractor.rs) ([`path.rs`](https://github.com/safer-rust/RAPx/blob/main/rapx/src/verify/path.rs)) builds finite verification paths from the function's CFG to each unsafe callsite. It uses `PathGraph` (from [`analysis::path_analysis::graph`](https://github.com/safer-rust/RAPx/blob/main/rapx/src/analysis/path_analysis/graph.rs)) to:

1. Build the CFG and detect strongly connected components (SCCs).
2. Enumerate acyclic paths through the CFG, allowing controlled SCC postfix repetition via `postfix_repeat`.
3. Filter paths that reach the target callsite block.
4. Truncate at the target block and deduplicate by prefix, storing them in a `PathTree` per callsite.

Each `Path` is a sequence of `PathStep::Block` steps ending with a `PathStep::Callsite`. Path enumeration is capped at `PATH_LIMIT` (1024) per callsite.

#### Why Path Sensitivity Matters

Consider a function with two branches that lead to the same unsafe call, but one branch establishes the needed preconditions and the other does not:

```rust
unsafe fn example(cond: bool, ptr: *mut u32, len: usize) {
    if cond {
        assert!(len > 0);
        let _ = ptr::read(ptr);  // Safe: len > 0 guarantees at least one element
    } else {
        let _ = ptr::read(ptr);  // Unsafe: len may be 0
    }
}
```

A path-insensitive analysis would merge the states from both branches, losing the `len > 0` fact from the first branch and incorrectly flagging both call sites as unproved. Path sensitivity preserves each branch's context, allowing the verifier to prove the first `ptr::read` while correctly identifying the second as unproved.

#### SCC Decomposition and Path Reachability

SCC decomposition handles loops by identifying the dominator (enter block) of each SCC and decomposing the SCC tree hierarchically. Path reachability filtering ([Chapter 5.1](./5.1-path.md)) further prunes paths that are structurally possible but semantically impossible — for example, when two `SwitchInt` branches on the same enum discriminant would produce contradictory constraints.

The combination of SCC decomposition and reachability filtering ensures that the verifier explores a manageable, relevant subset of paths rather than all combinatorially possible CFG traversals.

#### SCC-Aware Path Expansion and Multi-Occurrence Checkpoint Enumeration

When a verification checkpoint lies inside a loop body (an SCC), the path tree contains **multiple occurrences** of the same basic block at different loop iteration depths. For example, with a checkpoint at block 8 inside a loop body `[5,6,7,8,9,10]`:

```
PathTree (annotated):
  Node(0) → Node(1) → Node(3) → Node(5) → Node(6) → Node(7) → Node(8)[checkpoint, iter=1]
    → Node(9) → Node(10) → Node(5) → ... → Node(8)[checkpoint, iter=2]
      → ... → Node(8)[checkpoint, iter=N]
```

The backward slicer's `build_leaf_items` function processes the path tree in post-order. To capture deeper SCC occurrences (iterations 2..N), it continues processing children even at the checkpoint block — a technique called **child-path expansion**:

```
build_leaf_items(node=Node(8)[iter=1]):
  1. Build checkpoint_items for this occurrence (leaf)
  2. for child in node.children:  // continues past the checkpoint!
       recursively process Node(9), ..., Node(8)[iter=2]
       thread through parent block's terminator/statements
       produce another leaf for the deeper occurrence
  3. return all leaves
```

Each leaf represents one verification path corresponding to a specific loop iteration count. This enables the verifier to check safety properties at multiple iteration depths — critical for detecting off-by-one errors that only manifest after many iterations.

The `walk_all_prefixes` method (`PathTree`) provides an alternative enumeration strategy: instead of child-path expansion, it walks the tree and calls a callback at **every** occurrence of a target block (not just the first), continuing past the block into its children. Each callback receives the complete path prefix from the root, including all intermediate loop headers. This is useful when the child-path expansion may omit constraints from the last partial iteration's loop header.

#### Postfix Repeat Convergence

When `--postfix-repeat` is not explicitly set, the verifier automatically expands the postfix repeat value, exploring progressively deeper loop unrollings. The auto-expansion terminates under one of three convergence patterns:

```
for repeat in 0, 1, 2, ...:
  verify with postfix_repeat = repeat
  compare results with prior repeat
  apply convergence check → terminate or continue
```

| `postfix_repeat` | Max loop iterations in path |
|-----------------|---------------------------|
| 0 | 1 |
| 1 | 2 |
| 2 | 3 |
| ... | ... |
| k | k + 1 |

**Convergence pattern 1 — No change:** Deeper unrolling produces the same results as a single expansion — the state is fully determined at shallow depth, and further unrolling cannot change any verdict. The verifier terminates early.

**Convergence pattern 2 — Diverging toward violation:** Deeper unrolling reveals additional path instances where previously `Proved` properties become `Unproved` or `Unknown` — for example, an InBound check that passes at shallow depth but fails as the loop counter approaches the bound (see examples below). The auto-expansion terminates the moment any property is unprovable: discovering the violation is sufficient, and deeper unrolling that would produce even more violations is unnecessary.

**Convergence pattern 3 — Oscillating / bounded cycle:** Deeper unrolling alternates between a fixed set of states without reaching new conclusions (e.g., results at `repeat=3` match `repeat=1`, `repeat=4` matches `repeat=2`, repeating). The verifier detects the cycle and stops.

**InBound examples for pattern 2 (diverging toward violation):**

When an InBound property is `Proved` at `repeat=0` but the verifier detects that deeper unrolling may reveal violations, auto-expansion progressively probes deeper until convergence. Two categories of violation illustrate why convergence is necessary:

- **Category 1 (static)**: The off-by-one is detectable regardless of unrolling depth. For example, `ptr.wrapping_add(i + 1)` with guard `i < data.len()` — the SMT can symbolically reason that `i + 1` may exceed `len` even with a single loop body execution. These are caught in the initial pass (`repeat=0`), not by convergence. The canonical example is `inbound_unsound_6`:

  ```rust
  #[rapx::verify]
  pub fn unsound_scc_off_by_one(data: &[u32]) {
      let ptr = data.as_ptr();
      let mut i = 0usize;
      while i < data.len() {
          let current = ptr.wrapping_add(i + 1);
          unsafe { require_scc_inbound(current); }
          i += 1;
      }
  }
  ```

  Here `data.len()` is the direct slice length. The SMT models `i < len` and the offset `i + 1`, detecting that `i+1 < len` does not logically follow from `i < len` when `i == len - 1`. The violation is caught immediately — no deep unrolling required.

- **Category 2 (diverging)**: The violation only manifests at deeper iterations because an early-return guard anchors a minimum length. The canonical example is `inbound_unsound_7`:

  ```rust
  #[rapx::verify]
  pub fn unsound_len_guard_off_by_one(data: &[u32]) {
      let len = data.len();
      if len < 10 { return; }        // anchors len >= 10
      let ptr = data.as_ptr();
      let mut i = 0usize;
      while i < len {
          let current = ptr.wrapping_add(i + 1);
          unsafe { require_scc_inbound(current); }
          i += 1;
      }
  }
  ```

  At `repeat=0` (1 body execution): the path constrains `len == 1` via the loop exit, contradicting `len >= 10` from the if-check. This makes the path condition **unsatisfiable** → the SMT vacuously proves InBound. All InBound Proved → auto-expansion continues.

  At `repeat=9` (10 body executions): the path has `i=9` entering the 10th iteration. The InBound check for `ptr.wrapping_add(10)` requires `10 < len` (i.e., `len >= 11`). But only `len >= 10` is known from the early-return guard. The SMT cannot prove the tighter bound → **Unproved → auto-expansion terminates, UNSOUND reported**.

The postfix-repeat convergence is implemented in `VerifyRun::run()` ([`driver.rs`](https://github.com/safer-rust/RAPx/blob/main/rapx/src/verify/driver.rs)) at lines 653-724.

### 8.5.2 Backward Visit

The `BackwardVisitor` ([`path_refine/visitor.rs`](https://github.com/safer-rust/RAPx/blob/main/rapx/src/verify/path_refine/visitor.rs)) starts from the callsite and walks backward along the path, collecting only MIR statements and terminators that are *relevant* to the property being checked. Items are classified as:

- **`Statement`**: A MIR statement whose def-use chain reaches the property's target place. For example, `_5 = _4 + 1` is retained if `_5` flows into a `ValidPtr` target argument.
- **`Terminator`**: A MIR terminator (call, switch, assert) that influences control-flow constraints. `SwitchInt` and `Assert` terminators constrain branch conditions; `Call` terminators for safe functions are retained because their return values may participate in def-use chains.
- **`ContractFact`**: A caller's `#[rapx::requires]` contract injected as an entry assumption at the start of each path. These are prepended to the retained items to seed the forward analysis.
- **`Forget`**: A conservatively dropped fact (e.g., after an unknown call that may mutate a tracked pointer). This models the verifier's incomplete knowledge — when a call's effect on a tracked value is unknown, the verifier erases facts about that value rather than making unsound assumptions.

The backward visitor uses def-use analysis ([`def_use.rs`](https://github.com/safer-rust/RAPx/blob/main/rapx/src/verify/def_use.rs)) to trace value flow from the callsite arguments back to their origins, filtering out irrelevant operations. It also consults `CallDependencySummary` (from [`call_summary.rs`](https://github.com/safer-rust/RAPx/blob/main/rapx/src/verify/call_summary.rs)) to determine which callee arguments the return value depends on.

#### Def-Use Chain Tracing

The def-use analysis works by building a use-def chain (mapping each use of a local to its defining statement) from the MIR body, then walking backward from the property's target variables:

1. Identify the MIR locals that appear in the property's arguments (e.g., the pointer in `ValidPtr(ptr, T, 1)`).
2. For each local, find its defining statement via the use-def chain.
3. If the defining statement references other locals (e.g., `_5 = _4 + _3`), recursively trace those locals.
4. Stop at function arguments (origins), constants, and callsite results.

This filters out statements that compute unrelated values, keeping only the slice of MIR that is relevant to the property at hand. For a function with hundreds of lines of MIR but only one unsafe call, this dramatically reduces the state space that the forward visitor must model.

### 8.5.3 Forward Visit

The `ForwardVisitor` ([`forward_visit.rs`](https://github.com/safer-rust/RAPx/blob/main/rapx/src/verify/forward_visit.rs)) walks forward along the retained MIR items, building an abstract state consisting of:

- **`values`**: A map from MIR locals to `AbstractValue` (Place, Ref, RawPtr, ConstInt, Cast, Binary op, CallResult, etc.). This is the value-tracked state — what each local currently represents.
- **`facts`**: A list of `StateFact` entries recording semantic properties inferred from operations and calls.

#### State Facts

| Fact | Meaning | How It's Established |
|------|---------|---------------------|
| `PointsTo` | A pointer local points to a specific place (e.g., a struct field) | Assignment from an address-of expression |
| `Cast` | A pointer has been type-cast (e.g., `*mut T` to `*const T`) | `as` casts in MIR |
| `KnownAligned` | A pointer satisfies the alignment of a given type | Calls to allocation functions, `NonNull::new_unchecked` |
| `KnownInit` | Memory is initialized (result of `ptr::write`, `copy_nonoverlapping` dst) | Writing operations |
| `KnownAllocated` | Memory is allocated (result of `alloc::alloc`, `Box::new`) | Allocation calls |
| `KnownNonZero` | A value is guaranteed non-zero (result of `NonNull::new_unchecked`) | Non-null constructors |
| `KnownConst` | A value is a known constant (e.g., return of `size_of`) | Calls to sizing functions |
| `BranchEq` | A branch condition constrains a value (e.g., `_5 == 0` in a true branch) | `SwitchInt`, `Assert` terminators |
| `Call` / `CallEffect` | Summarized call semantics (pointer arithmetic, alias relationships) | Call effect summaries |

#### Abstract Value Tracking

The forward visitor maintains a map from MIR locals to abstract values. When it encounters an assignment `_3 = _2`, it copies the abstract value of `_2` to `_3`. When it encounters `_4 = ptr::add(_3, _5)`, it consults the call effect summary to determine that `_4` is a pointer derived from `_3` with an offset of `_5 * size_of::<T>()`, and records the appropriate `PointsTo` and offset facts.

The abstract domain is path-sensitive: the visitor processes each path independently, so facts established on one branch do not leak into another. This is what enables proving call sites on one branch even when another branch would fail.

### 8.5.4 SMT Check

The `SmtChecker` ([`smt_check/`](https://github.com/safer-rust/RAPx/tree/main/rapx/src/verify/smt_check/)) encodes the forward state facts and the target property as SMT-LIB assertions and invokes a Z3 solver to check whether the facts logically imply the property. Each property kind has a dedicated checking module:

| Module | Checks |
|--------|--------|
| [`valid_ptr.rs`](https://github.com/safer-rust/RAPx/blob/main/rapx/src/verify/smt_check/valid_ptr.rs) | Pointer validity for the given type and count |
| [`align.rs`](https://github.com/safer-rust/RAPx/blob/main/rapx/src/verify/smt_check/align.rs) | Pointer alignment matches the type requirement |
| [`in_bound.rs`](https://github.com/safer-rust/RAPx/blob/main/rapx/src/verify/smt_check/in_bound.rs) | Pointer arithmetic stays within allocation bounds |
| [`init.rs`](https://github.com/safer-rust/RAPx/blob/main/rapx/src/verify/smt_check/init.rs) | Memory initialization state |
| [`non_null.rs`](https://github.com/safer-rust/RAPx/blob/main/rapx/src/verify/smt_check/non_null.rs) | Pointer non-nullness |
| [`allocated.rs`](https://github.com/safer-rust/RAPx/blob/main/rapx/src/verify/smt_check/allocated.rs) | Memory allocation tracking |
| [`valid_num.rs`](https://github.com/safer-rust/RAPx/blob/main/rapx/src/verify/smt_check/valid_num.rs) | Numeric range constraints |
| [`deref.rs`](https://github.com/safer-rust/RAPx/blob/main/rapx/src/verify/smt_check/deref.rs) | Composite: `Allocated` + `InBound` for dereference safety |
| [`non_overlap.rs`](https://github.com/safer-rust/RAPx/blob/main/rapx/src/verify/smt_check/non_overlap.rs) | Memory region non-overlap constraints |
| [`typed.rs`](https://github.com/safer-rust/RAPx/blob/main/rapx/src/verify/smt_check/typed.rs) | Memory holds a valid bit-pattern for the type |
| [`alias.rs`](https://github.com/safer-rust/RAPx/blob/main/rapx/src/verify/smt_check/alias.rs) | No other live pointer aliases the memory |
| [`alive.rs`](https://github.com/safer-rust/RAPx/blob/main/rapx/src/verify/smt_check/alive.rs) | Allocation liveness tracking |

Properties without dedicated SMT modules (e.g., `ValidString`, `ValidCStr`, `Pinned`, `Unwrap`, `ValidTransmute`, `NonSize`, `Null`, `Size`, `NoPadding`, `Owning`, `Layout`, `Ptr2Ref`, `Opened`, `Trait`, `NonVolatile`, `Unreachable`) produce `CheckResult::Unknown` and do not contribute to soundness verdicts.

The checker returns a `CheckResult::Proved` if the SMT solver confirms the property holds under all modeled assumptions, or `CheckResult::Unproved` otherwise.

#### SMT Encoding Example

For a `ValidPtr(ptr, u32, 1)` check where the forward facts establish:
- `ptr` points to `self.buf` (a heap-allocated field)
- `self.buf` was allocated with `alloc::alloc(size)` where `size >= size_of::<u32>()`

The encoding produces SMT-LIB assertions like:

```smt2
(declare-const ptr (_ BitVec 64))
(declare-const buf_base (_ BitVec 64))
(declare-const buf_size (_ BitVec 64))
(assert (= ptr buf_base))
(assert (>= buf_size 4))           ; size_of::<u32>() = 4
(assert (>= (- ptr buf_base) 0))   ; ptr is within the allocation
(assert (<= (- (+ ptr 3) buf_base) buf_size))  ; ptr + 3 is within bounds
```

If Z3 determines these assertions are satisfiable (i.e., it is possible for the pointer to be valid), the check passes. If Z3 finds a counterexample, the check fails and is marked `Unproved`.

#### Solver Integration

RAPx uses the `z3` crate for native Rust bindings to the Z3 solver. The solver is invoked via a timeout-limited context to prevent infinite loops on complex encodings. The default solver timeout is 5 seconds per check; if Z3 times out, the result is `CheckResult::Unproved` (the verifier errs on the side of caution).

### 8.5.5 Call Effect Summaries

[`call_summary.rs`](https://github.com/safer-rust/RAPx/blob/main/rapx/src/verify/call_summary.rs) maintains a database of call effects for common unsafe and standard library functions. Each entry maps a function path to a `CallEffectSummary` with effects such as:

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

#### How Call Effect Summaries are Used

During **backward analysis**, `dependency_summary()` provides `CallDependencySummary` with:
- `return_depends_on_args`: Which callee arguments the return value depends on (so the backward visitor knows which arguments to trace).
- `may_write_args`: Which arguments may be mutated by the call (so the backward visitor can insert `Forget` markers to invalidate tracked facts).

During **forward analysis**, `effect_summary()` provides `CallEffectSummary` with concrete effects to apply:
- `ReturnPointerAdd(arg=0, arg=1)` means the return value = `arg0 + arg1 * size_of::<T>()`, so the forward visitor records a `PointsTo` fact linking the return value to `arg0` with an offset.
- `WriteMemory(arg=0)` means the call initializes the memory pointed to by `arg0`, so the forward visitor records a `KnownInit` fact.
- `ForgetArgFacts(arg=0)` means the call may invalidate facts about `arg0`, so the forward visitor removes existing `PointsTo` and `KnownInit` facts for that argument.

#### Extending Call Summaries

Users can extend the call summary database for their own libraries by adding entries to a custom JSON file. The format mirrors `std-contracts.json` with function paths mapped to effect lists. This is necessary for inter-procedural verification of custom unsafe abstractions.

## 8.6 Struct Invariant Verification

When a function target is a method on a struct with `#[rapx::invariant(...)]` annotations, the verifier additionally checks that the struct invariants are maintained:

- **Constructors** (functions returning `Self`): Paths are filtered to return blocks. Invariants must be proved at each return checkpoint, with only the constructor's own `#[rapx::requires]` as entry assumptions.
- **Methods**: Invariants are assumed at function entry (prepended as `ContractFact` items in the backward visitor) and verified at all return points.

The `verify_struct_invariants` method in [`driver.rs`](https://github.com/safer-rust/RAPx/blob/main/rapx/src/verify/driver.rs) handles this via `build_invariant_paths()`:

- For constructors: prefixes paths to return blocks with the constructor's contracts.
- For methods: builds whole-CFG paths with invariants assumed at entry and checked at every return block.

This analysis ensures that struct soundness is preserved across method sequences — invariants established by the constructor are maintained through mutations and respected by accessors.

### 8.6.1 Worked Example: Bounded Buffer

```rust
#[rapx::invariant(ValidPtr(self.ptr, u8, self.cap))]
#[rapx::invariant(Align(self.ptr, u8))]
pub struct Buf {
    ptr: *mut u8,
    len: usize,
    cap: usize,
}

impl Buf {
    #[rapx::requires(ValidPtr(ptr, u8, cap))]
    #[rapx::requires(Align(ptr, u8))]
    pub unsafe fn new(ptr: *mut u8, cap: usize) -> Self {
        Buf { ptr, len: 0, cap }
    }

    pub unsafe fn push(&mut self, val: u8) {
        // Verifier assumes invariants hold on entry:
        //   ValidPtr(self.ptr, u8, self.cap)
        //   Align(self.ptr, u8)
        assert!(self.len < self.cap);
        self.ptr.add(self.len).write(val);
        self.len += 1;
        // Verifier checks invariants hold on exit.
        // ValidPtr still holds (cap unchanged, ptr unchanged).
        // Align still holds (ptr unchanged).
    }
}
```

The verifier traces through `push`:
1. At entry, invariants `ValidPtr(self.ptr, u8, self.cap)` and `Align(self.ptr, u8)` are assumed.
2. The `self.ptr.add(self.len).write(val)` call requires `ValidPtr` and `Align` for the computed pointer. The `assert!(self.len < self.cap)` provides the `ValidNum` constraint needed to prove `ValidPtr` for the offset pointer.
3. At exit, invariants are re-checked: `self.ptr` and `self.cap` are unchanged, so both invariants still hold. Proved.

## 8.7 Invless Mode Sequences

In `invless` mode (`run_invless_sequences` in [`driver.rs`](https://github.com/safer-rust/RAPx/blob/main/rapx/src/verify/driver.rs)), when a struct has constructors and methods, the verifier generates verification sequences:

```
Constructor → Method
Constructor → Mutator → Method
Constructor → Mutator₁ → Mutator₂ → Method
```

Each sequence propagates the constructor's `#[rapx::requires]` contracts through the mutator chain. Mutated fields are tracked via `get_mutated_fields`, and contracts referencing those fields are filtered out at each step.

For each sequence, the verifier builds a `FunctionTarget` with the accumulated `caller_requires`, then runs the standard pipeline with incremented `postfix_repeat`. Results are emitted per sequence with a chain label like `new_in → push → into_raw_parts_with_alloc`.

### 8.7.1 Sequence Generation Algorithm

```
For each struct S:
  For each constructor C of S (returns Self):
    contracts = C's #[rapx::requires]
    
    // Sequence: Constructor → Reader (one step)
    For each &self method R of S:
      Build FunctionTarget with contracts as caller_requires
      Verify R with these entry assumptions
    
    // Sequences: Constructor → Mutators → Reader (multi-step)
    For each &mut self method M of S:
      mutated = get_mutated_fields(M's body)
      next_contracts = filter contracts: drop those referencing mutated fields
      
      // Recurse with M's exit state as the next entry state
      For each &self method R of S:
        Build FunctionTarget with next_contracts as caller_requires
        Verify R with these entry assumptions
      
      // Continue chaining mutators
      For each subsequent &mut self method M2 of S:
        ... (continue recursion up to MAX_CHAIN_DEPTH)
```

The `MAX_CHAIN_DEPTH` limits sequence length to avoid combinatorial explosion. Contracts that reference fields mutated by a method are conservatively dropped rather than attempting to re-prove them (since the verifier cannot know the exact new state of the field without analyzing the mutator's body in full inter-procedural mode).

### 8.7.2 When to Use invless Mode

`invless` mode is best used as a scoping tool — it tells you *which* constructor contracts are implicitly assumed by each method sequence, even without explicit `#[rapx::invariant]` annotations. If a sequence produces an unproved result, it indicates that either:
1. The constructor's contracts are insufficient for the reader method's needs, or
2. A mutator in the chain invalidates a contract the reader depends on.

This helps you identify where `#[rapx::invariant]` annotations are needed and what they should contain.

## 8.8 Raw Pointer Dereference Checking

In addition to explicit unsafe call sites, the verifier checks raw pointer dereferences (`*ptr` and `*ptr = val`) in unsafe blocks. `build_raw_ptr_deref_checks()` ([`target.rs`](https://github.com/safer-rust/RAPx/blob/main/rapx/src/verify/target.rs)) scans the MIR for dereference projections on raw pointer types and creates pseudo-callsites with the following properties:

- **Read dereferences** (`*ptr`): Requires `ValidPtr(ptr, T, 1)`, `Align(ptr, T)`, and `Typed(ptr, T)`.
- **Write dereferences** (`*ptr = val`): Requires `ValidPtr(ptr, T, 1)` and `Align(ptr, T)`.

These pseudo-callsites are integrated into the same `callsite → property` verification pipeline, so they benefit from the same path sensitivity and abstract state tracking as regular unsafe calls.

### 8.8.1 Why Separate Checking for Dereferences?

In safe Rust, `*ptr` on a raw pointer is forbidden. In unsafe blocks, `*ptr` is allowed but must satisfy the same safety conditions as `ptr::read` and `ptr::write`. By treating raw pointer dereferences as pseudo-callsites, the verifier ensures they are subject to the same rigorous checking as explicit unsafe function calls.

### 8.8.2 Example

```rust
unsafe fn write_first_element(slice: &mut [u32]) {
    let ptr = slice.as_mut_ptr();
    // Verifier collects: *ptr as a write dereference pseudo-callsite
    *ptr = 42;
    // Required: ValidPtr(ptr, u32, 1), Align(ptr, u32)
}
```

The verifier traces `ptr` back to `slice.as_mut_ptr()`, whose call effect summary establishes that `ptr` points into `slice`'s allocation, that `slice.len() > 0` (implied by the reference's validity), and that `ptr` is properly aligned. Both properties are Proved.

## 8.9 Verification Report

The `VerificationReport` ([`report.rs`](https://github.com/safer-rust/RAPx/blob/main/rapx/src/verify/report.rs)) aggregates results for all (callsite, path, property) triples. Each `PropertyCheckResult` records:

- The callsite location (`CallsiteLocation`: caller DefId + basic block).
- Path and property indices.
- The property being checked and the verification result (`Proved` / `Unproved`).
- Forward and backward visit diagnostics.

The `emit_verify_summary` function in [`driver.rs`](https://github.com/safer-rust/RAPx/blob/main/rapx/src/verify/driver.rs) groups results by callsite and prints a compact summary:

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

If any property is unproved, the function is reported as `UNSOUND` with a count of unproved properties:

```
============================================================
[rapx::verify] function: my_crate::buggy_function
============================================================
  --- unsafe callsites ---
      unsafe callsite: bb5 -> std::ptr::read
        path [0, 2, 4, 5]:
          ValidPtr | Proved
          Align | Proved
          Typed | Unproved
  result: UNSOUND (1 property unproved)
============================================================
```

### 8.9.1 Interpreting the Report

| Result | Meaning | Action |
|--------|---------|--------|
| All `Proved` → `SOUND` | The verifier confirmed all contracts for all paths. | No action needed; the function is verified. |
| Some `Unproved` → `UNSOUND` | At least one property could not be proved for at least one path. | Inspect the unproved property and the path. The verifier may lack necessary facts (e.g., missing `assert!`, unknown call effect), or there may be a genuine UB. |
| `CheckResult::Unknown` | The contract tag is not yet supported. | File an issue or contribute support for the contract tag. |

### 8.9.2 Forward/Backward Visit Diagnostics

When a property is `Unproved`, the report includes diagnostics from the backward and forward visitors. These show which MIR items were collected and which facts were established, helping you understand *why* the proof failed:

```
        path [0, 2, 4, 5]:
          ValidPtr | Proved
          Align | Proved
          Typed | Unproved
          backward items: [
            Statement(_4 = ptr::read(move _3)),
            Statement(_3 = &raw const (*_2)),
            Statement(_2 = &(*_1).0),
            ContractFact(ValidPtr(self.0, T, 1)),
          ]
          forward facts: [
            PointsTo(_3, (*_2)),
            PointsTo(_2, (*_1).0),
          ]
```

In this example, the `Typed` check failed because the forward facts don't establish that `(*_1).0` holds a valid value of type `T`. The missing piece might be a `KnownInit` fact that could be supplied by adding `#[rapx::requires(Init(self.0, T, 1))]` to the function, or by annotating a preceding `ptr::write` call in the call summary database.

## 8.10 Example: Verifying `Vec::into_raw_parts_with_alloc`

Consider `Vec::into_raw_parts_with_alloc` from the standard library. Its MIR contains a `ptr::read` call on the allocator field:

```
bb9:  _13 = &raw const (*_14)
      _12 = std::ptr::read::<A>(move _13) -> bb10
```

The verifier collects this callsite, resolves `ptr::read`'s safety contracts (`ValidPtr`, `Align`, `Typed`), and verifies them along paths reaching bb9.

### 8.10.1 Step-by-Step Verification

**1. Path Extraction:** The verifier enumerates paths from the function entry to bb9. For a typical `Vec`, the paths include:
- The direct path through `Vec::into_raw_parts_with_alloc`'s body.
- Paths that go through the Vec's fields (`ptr`, `len`, `cap`, `alloc`).

**2. Backward Visit:** Starting from `ptr::read(move _13)` in bb9, the backward visitor traces `_13`:
- `_13 = &raw const (*_14)` → `_13` is a raw pointer to `*_14`.
- `_14` is the Vec's allocator field (through prior assignments).
- `_14` is initialized at construction time (RAII guarantees).

Retained items include these assignments and the entry contract (if the function has `#[rapx::requires]`).

**3. Forward Visit:** The forward visitor walks forward, establishing:
- `PointsTo(_13, _14.alloc)` — `_13` points into the Vec's allocator field.
- `KnownInit(_14.alloc, A, 1)` — the allocator field is initialized (from RAII).
- `KnownAligned(_14.alloc, A)` — the field satisfies alignment (guaranteed by the struct layout).

**4. SMT Check:** For each property:

- **ValidPtr**: The pointer returned by `&raw const (*_14)` points to live, initialized memory within the Vec's allocation. The fact chain (`PointsTo` + `KnownInit`) confirms non-null dereferenceability. **Proved.**

- **Align**: The pointer's address matches the alignment of type `A`, guaranteed by the struct layout (Rust ensures all struct fields are properly aligned). **Proved.**

- **Typed**: The allocator maintains correct typing throughout the Vec's lifetime — it was set at construction and not invalidated by subsequent operations. **Proved.**

If the Vec struct had `#[rapx::invariant]` annotations, the verifier would additionally check constructor and method boundaries, ensuring that invariants like `ValidPtr(self.ptr, T, self.cap)` are maintained across all safe mutations.

### 8.10.2 Extended Example: With a Bug

Now consider a buggy version where a branch fails:

```rust
unsafe fn buggy_read(buffer: *const u8, len: usize, index: usize, cond: bool) -> u8 {
    if cond {
        if index < len {
            return *buffer.add(index);  // Safe path
        }
    }
    // Bug: no bounds check on this path
    *buffer.add(index)  // Unsafe path
}
```

The verifier produces:

```
============================================================
[rapx::verify] function: my_crate::buggy_read
============================================================
  --- raw pointer dereferences ---
      *buffer.add(index) -> bb7
        path [0, 1, 2, 3, 5, 7]:          // cond=false → no bounds check
          ValidPtr | Unproved
          Align | Proved
          Typed | Proved
        path [0, 1, 2, 4, 7]:             // cond=true, index<len
          ValidPtr | Proved
          Align | Proved
          Typed | Proved
  result: UNSOUND (1 property unproved)
============================================================
```

The report reveals that on the `cond=false` path, no `ValidNum(index < len)` constraint is established, so `ValidPtr(buffer.add(index), u8, 1)` cannot be proved. The developer can fix this by adding the bounds check to all paths or restructuring the control flow.

## 8.11 Limitations and Current Status

### 8.11.1 Current Limitations

- **Path enumeration is capped** at 1024 paths per callsite. Functions with very complex control flow (e.g., deeply nested loops with many conditional branches) may exceed this limit, in which case the verifier processes the first 1024 paths and reports a warning.
- **Postfix repeat convergence** may not exhaustively unroll deeply nested SCCs within a practical bound. Off-by-one bugs requiring more than 16 loop iterations to manifest may escape detection if convergence terminates earlier (e.g., via pattern 1 or 3).
- **Inter-procedural analysis is limited** to pre-defined call summaries; full cross-function MIR-level verification is not supported. This means custom unsafe functions that call other custom unsafe functions require either explicit contracts on both or manual entry in the call summary database.
- **The SMT encoding approximates** numeric constraints and may produce false positives for complex arithmetic. Specifically, non-linear arithmetic (e.g., multiplication of two variables) is not well-supported by the underlying SMT theories and may result in `Unproved` results even when the property holds.
- **Struct invariant verification** for method sequences exceeding the maximum chain depth is not explored, and field mutations in deeply nested method chains may be over-approximated.
- **Safety contracts for standard library functions** are maintained manually in [`std-contracts.json`](https://github.com/safer-rust/RAPx/blob/main/rapx/src/verify/attribute/assets/std-contracts.json). The database covers 313 functions with 737 contract entries across 30 property kinds. Functions not covered by the database are silently skipped.
- **The verifier supports 30 `PropertyKind` variants** (see Section 8.4.1). Of these, 12 have dedicated SMT solver modules (see Section 8.5.4); the remaining produce `CheckResult::Unknown` and do not block soundness. Contract tags not matching any recognized variant are classified as `Unknown` and silently skipped.
- **Concurrent code** (e.g., `Arc`, `Mutex`, atomics) is not modeled. The verification assumes single-threaded execution and does not account for data races or memory ordering.
- **Inline assembly** (`asm!` blocks) are not analyzed. Functions containing inline assembly are conservatively treated as having unknown effects.

### 8.11.2 Performance Characteristics

| Factor | Impact |
|--------|--------|
| Number of paths per callsite | Linear in verification time — each path is analyzed independently |
| Path length (MIR statements) | Linear in backward/forward visit time |
| Number of property kinds | Linear — each property is checked independently |
| SMT encoding complexity | Dominated by `ValidNum` and `InBound` checks, which encode integer arithmetic |
| Solver timeout | Default 5 seconds per check; complex numeric constraints may time out |

### 8.11.3 Future Work

- **Deep matching support**: Extend path tracking through complex `match` statements. Supported: nested enum destructuring (`Some(Ok(v))`) via type-based variant count lookup, `switchInt` on dereferenced enum pointers (`*ptr`), and guard-clause comparison source tracking (infrastructure in place for relational constraint propagation). Remaining: guard clause relational constraint integration in verification pipeline.
- **Full inter-procedural verification**: Replace call summaries with MIR-level cross-function analysis for custom unsafe abstractions.
- **Postcondition inference**: Automatically derive postconditions from function bodies so that callers of safe wrappers can benefit from the verifier's analysis results without manual annotation.
- **Lifetime-aware pointer analysis**: Integrate borrow-checker information to more precisely model borrow lifetimes and stack-vs-heap allocation.
- **Concurrency model**: Extend the abstract domain to model thread-local state and detect data races across unsafe boundaries.
