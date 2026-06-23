# Chapter 4. Preliminary: Compiler Internals

This chapter introduces the Rust compiler internals that RAPx builds upon. Understanding these concepts is essential for reading the analysis and verification chapters that follow.

## Intermediate Representations

RAPx primarily operates on two levels of Rust's intermediate representation:

### HIR (High-Level IR)

[HIR](https://rustc-dev-guide.rust-lang.org/hir.html) is a type-checked, desugared AST. Macro expansion, name resolution, and type checking have completed by this stage. HIR is used in RAPx for:

- **Fast pre-filtering**: `hir_contains_unsafe()` in the Safety-Flow and Verification modules checks whether a function body contains `unsafe fn` declarations or `unsafe { }` blocks without descending into MIR.
- **Attribute scanning**: `#[rapx::verify]`, `#[rapx::requires(...)]`, and `#[rapx::invariant(...)]` are read from HIR attributes via `tcx.hir_attrs()`.
- **Struct definition inspection**: Checking whether a struct has `PhantomData` fields or specific type invariants.

```
cargo rustc -- -Z unpretty=hir-tree
```

### MIR (Mid-Level IR)

[MIR](https://rustc-dev-guide.rust-lang.org/mir/index.html) is a control-flow-graph-based IR where each function is decomposed into **basic blocks** connected by **terminators**. MIR is not in SSA form by default. RAPx uses MIR for all path-sensitive and dataflow analyses because:

- MIR has a finite, well-defined set of statement and terminator kinds, making exhaustive pattern matching feasible.
- Every loop in MIR is a **natural loop** (has a single dominator), enabling SCC-based path enumeration.
- MIR preserves type information and place projections, enabling field-sensitive analysis.

```
cargo rustc -- -Zunpretty=mir
```

To obtain the optimized MIR for a function:

```rust
let body: &Body<'tcx> = tcx.optimized_mir(def_id);
```

RAPx compiles with `-Zmir-opt-level=0` to preserve the original MIR structure (no inlining, no optimization passes that could obscure unsafe operations).

## Key Data Structures

### TyCtxt

[`TyCtxt<'tcx>`](https://doc.rust-lang.org/nightly/nightly-rustc/rustc_middle/ty/struct.TyCtxt.html) is the central context object of the Rust compiler. It provides access to virtually all compiler state: type information, MIR bodies, HIR bodies, crate metadata, and trait resolution. Every RAPx analysis module receives a `TyCtxt<'tcx>` and uses it as the entry point for all queries.

```rust
// Get the MIR body for a function
let body = tcx.optimized_mir(def_id);

// Get the type of a MIR local
let ty = body.local_decls[local].ty;

// Iterate over all local crate definitions
for local_def_id in tcx.iter_local_def_id() {
    let def_kind = tcx.def_kind(local_def_id);
    // ...
}
```

### DefId

[`DefId`](https://doc.rust-lang.org/beta/nightly-rustc/rustc_span/def_id/struct.DefId.html) is a globally unique identifier for any item (function, struct, trait, impl, etc.) across all crates. It pairs a crate number (`CrateNum`) with a definition index.

```rust
let def_path = tcx.def_path_str(def_id);  // e.g. "std::vec::Vec::new"
let crate_name = tcx.crate_name(def_id.krate);  // "std", "core", etc.
```

[`LocalDefId`](https://doc.rust-lang.org/beta/nightly-rustc/rustc_span/def_id/struct.LocalDefId.html) is a crate-local variant that does not carry a crate identifier:

```rust
let def_id: DefId = local_def_id.to_def_id();
```

### MIR Body Structure

A [`Body<'tcx>`](https://doc.rust-lang.org/beta/nightly-rustc/rustc_middle/mir/struct.Body.html) represents the MIR of a single function. Its key fields:

```rust
pub struct Body<'tcx> {
    pub basic_blocks: IndexVec<BasicBlock, BasicBlockData<'tcx>>,
    pub local_decls: IndexVec<Local, LocalDecl<'tcx>>,
    pub arg_count: usize,          // number of arguments
    pub spread_arg: Option<Local>, // variadic spread argument
    pub var_debug_info: Vec<VarDebugInfo<'tcx>>,
    pub span: Span,
    // ...
}
```

- `arg_count`: Arguments occupy `Local(1)` through `Local(arg_count)`. `Local(0)` is the return value.
- `local_decls`: Maps each `Local` to its type, mutability, and source span.

### BasicBlock and BasicBlockData

A [`BasicBlock`](https://doc.rust-lang.org/beta/nightly-rustc/rustc_middle/mir/struct.BasicBlock.html) is an index into the `basic_blocks` vector. [`BasicBlockData`](https://doc.rust-lang.org/beta/nightly-rustc/rustc_middle/mir/struct.BasicBlockData.html) contains:

```rust
pub struct BasicBlockData<'tcx> {
    pub statements: Vec<Statement<'tcx>>,  // zero or more statements
    pub terminator: Option<Terminator<'tcx>>, // exactly one, except for unreachable blocks
    pub is_cleanup: bool,
}
```

Statements execute sequentially within a block. The terminator determines which block(s) execute next.

### Statements

[`Statement`](https://doc.rust-lang.org/beta/nightly-rustc/rustc_middle/mir/enum.StatementKind.html) kinds most relevant to RAPx:

| Kind | Example | Used By |
|------|---------|---------|
| `Assign(place, rvalue)` | `_3 = _1 + _2` | All analyses — this is the primary dataflow source |
| `StorageLive(local)` | begin lifetime | rCanary (memleak) |
| `StorageDead(local)` | end lifetime, drop value | rCanary, SafeDrop |
| `FakeRead` | borrow checker artifact | Generally ignored |
| `SetDiscriminant` | set enum variant tag | Alias analysis (discriminant tracking) |

### Terminators

[`Terminator`](https://doc.rust-lang.org/beta/nightly-rustc/rustc_middle/mir/enum.TerminatorKind.html) kinds most relevant to RAPx:

| Kind | Description | Used By |
|------|-------------|---------|
| `Call { func, args, destination, target, unwind }` | Function call | All analyses — unsafe callee detection, dataflow, alias interproc |
| `SwitchInt { discr, targets }` | Branch on integer/enum value | Path analysis (constraint tracking), range analysis |
| `Assert { cond, expected, target, unwind }` | Runtime assertion | Path analysis (branch conditions) |
| `Goto { target }` | Unconditional branch | CFG construction |
| `Return` | Function return | Path extraction, dataflow |
| `Drop { place, target, unwind }` | Drop a value | SafeDrop, rCanary |
| `Unreachable` | Unreachable code | CFG construction |

### Place and Local

[`Place<'tcx>`](https://doc.rust-lang.org/beta/nightly-rustc/rustc_middle/mir/struct.Place.html) represents a memory location: a base `Local` plus a sequence of projections.

[`Local`](https://doc.rust-lang.org/beta/nightly-rustc/rustc_middle/mir/struct.Local.html) is an index into `body.local_decls`. `Local(0)` is the return value; `Local(1..=arg_count)` are function arguments; higher indices are temporaries and user variables.

```rust
let place = Place { local: Local::from_usize(3), projection: vec![] };
let place_with_field = place.project_field(1);  // _3.1
```

Projections applied in sequence:

| Projection | Meaning | MIR Syntax |
|-----------|---------|------------|
| `Deref` | Pointer dereference | `(*_3)` |
| `Field(idx, ty)` | Struct/tuple field access | `_3.1` |
| `Downcast(variant)` | Enum variant projection | `_3 as VariantName` |
| `Index(local)` | Array/slice indexing | `_3[_4]` |
| `ConstantIndex { offset, .. }` | Constant index into array | `_3[2]` |
| `Subslice { from, to }` | Slice sub-range | `_3[1..3]` |

### Operand and Rvalue

[`Operand<'tcx>`](https://doc.rust-lang.org/beta/nightly-rustc/rustc_middle/mir/enum.Operand.html) is the right-hand side source in assignments and call arguments:

```rust
pub enum Operand<'tcx> {
    Copy(Place<'tcx>),    // copy the value at place
    Move(Place<'tcx>),    // move the value at place
    Constant(Box<ConstOperand<'tcx>>),  // compile-time constant
}
```

[`Rvalue<'tcx>`](https://doc.rust-lang.org/beta/nightly-rustc/rustc_middle/mir/enum.Rvalue.html) is the right-hand side of an assignment, describing how the value is produced:

| Rvalue | Description | Example |
|--------|-------------|---------|
| `Use(operand)` | Copy or move | `_3 = move _2` |
| `Ref(region, kind, place)` | Create reference | `_3 = &_2` |
| `RawPtr(mutability, place)` | Create raw pointer | `_3 = &raw const _2` |
| `Cast(kind, operand, ty)` | Type cast | `_3 = _2 as *const u32` |
| `BinaryOp(op, (lhs, rhs))` | Binary operation | `_3 = Add(_2, _1)` |
| `UnaryOp(op, operand)` | Unary operation | `_3 = Not(_2)` |
| `Discriminant(place)` | Get enum discriminant | `_3 = discriminant(_2)` |
| `Aggregate(kind, operands)` | Construct value | `_3 = Foo { x: _1, y: _2 }` |
| `CopyForDeref(place)` | Copy for dereference | Used in deref patterns |

## How RAPx Hooks into rustc

RAPx runs as a custom rustc driver via the `RUSTC_WRAPPER` mechanism, similar to Miri. The entry point is `cargo-rapx` which:

1. **Phase 1 (`cargo rapx ...`)**: Sets `RUSTC_WRAPPER=cargo-rapx` and invokes `cargo check`.
2. **Phase 2 (rustc wrapper)**: Cargo invokes `cargo-rapx path/to/rustc ...`. RAPx intercepts this, runs the requested analysis, then calls the real rustc to complete compilation.

The `RapCallback` struct in `lib.rs` implements `rustc_driver::Callbacks` with two hooks:

- **`config`**: Injects `-Zalways-encode-mir`, `-Zmir-opt-level=0` and other flags to ensure MIR is available and unoptimized.
- **`after_analysis`**: After type checking and MIR construction, RAPx dispatches to the requested command (`Analyze`, `Check`, or `Verify`), each of which runs within `rustc_public::rustc_internal::run(tcx, ...)` to access the `TyCtxt`.

## The Analysis Trait

Every RAPx analysis module implements the [`Analysis`] trait defined in [`rapx/src/analysis/mod.rs`](https://github.com/safer-rust/RAPx/blob/main/rapx/src/analysis/mod.rs):

```rust
pub trait Analysis {
    fn name(&self) -> &'static str;
    fn run(&mut self);
    fn reset(&mut self);
}
```

- `name()`: Returns a human-readable name for logging.
- `run()`: Executes the analysis. This is the main entry point called by the dispatcher.
- `reset()`: Clears internal state for re-execution.

Each analysis module defines a subtrait (e.g., `AliasAnalysis`, `DataflowAnalysis`, `RangeAnalysis`) that extends `Analysis` with query methods. A default implementation struct (e.g., `AliasAnalyzer`, `DataflowAnalyzer`) provides concrete algorithms.

## Navigating the Compiler API

Common patterns used throughout RAPx:

```rust
// Iterate all function definitions in the local crate
for local_def_id in tcx.iter_local_def_id() {
    if matches!(tcx.def_kind(local_def_id), DefKind::Fn | DefKind::AssocFn) {
        let def_id = local_def_id.to_def_id();
        // ...
    }
}

// Get a function's MIR body
let body: &Body<'_> = tcx.optimized_mir(def_id);

// Walk all basic blocks
for (block_idx, bb) in body.basic_blocks.iter().enumerate() {
    for stmt in &bb.statements {
        if let StatementKind::Assign(box (place, rvalue)) = &stmt.kind {
            // handle assignment
        }
    }
    if let Some(terminator) = &bb.terminator {
        match &terminator.kind {
            TerminatorKind::Call { func, args, destination, .. } => { /* ... */ }
            TerminatorKind::SwitchInt { discr, targets } => { /* ... */ }
            TerminatorKind::Return => { /* ... */ }
            _ => {}
        }
    }
}

// Check if a function is unsafe or contains unsafe blocks (HIR-level)
let body_id = tcx.hir_body_id_if_owned(local_def_id);
let is_unsafe = hir_contains_unsafe(tcx, body_id);

// Get function argument count
let argc = body.arg_count;
// _0 = return value, _1.._argc = parameters

// Resolve a callee DefId from a Call terminator
if let Operand::Constant(c) = func {
    if let TyKind::FnDef(callee_def_id, _) = c.const_.ty().kind() {
        let callee_name = tcx.def_path_str(*callee_def_id);
    }
}
```

## Stable MIR and Compatibility

### The Nightly Dependency

RAPx depends on `#![feature(rustc_private)]` — it links against the Rust compiler's internal crates (`rustc_middle`, `rustc_hir`, `rustc_driver`, etc.). These APIs are **unstable** and change with every nightly compiler release. There is no stable compiler plugin API in Rust.

This has practical consequences:

- **Pinned toolchain**: RAPx ships a `rust-toolchain.toml` specifying an exact nightly version (e.g., `nightly-2026-04-03`). Users must install this version.
- **Breakage on upgrade**: Bumping the nightly version typically requires updating dozens of API calls across the codebase, as MIR statement/terminator variants, field names, and module paths shift.
- **Conditional compilation**: The [`compat.rs`](https://github.com/safer-rust/RAPx/blob/main/rapx/src/compat.rs) module (`rapx/src/compat.rs`) centralizes version-gated re-exports. `build.rs` detects the rustc version at build time and sets `cfg` flags like `rapx_rustc_ge_193`, `rapx_rustc_ge_196`, `rapx_rustc_ge_198`, `rustc_spanned_at_root`, etc. Source files use these flags to adapt to API changes without duplicating version checks.

Example from [`compat.rs`](https://github.com/safer-rust/RAPx/blob/main/rapx/src/compat.rs):

```rust
// Spanned moved from rustc_span::source_map to rustc_span root in rustc 1.97
#[cfg(rustc_spanned_at_root)]
pub use rustc_span::Spanned;
#[cfg(not(rustc_spanned_at_root))]
pub use rustc_span::source_map::Spanned;
```

And a typical usage pattern in analysis code:

```rust
#[cfg(rapx_rustc_ge_198)]
let field_ty = field.ty(self.tcx, substs).skip_norm_wip();
#[cfg(not(rapx_rustc_ge_198))]
let field_ty = field.ty(self.tcx, substs);
```

The CI ([`rapx/.github/workflows/test.yml`](https://github.com/safer-rust/RAPx/blob/main/.github/workflows/test.yml)) tests against multiple nightly versions to catch regressions early.

### Stable MIR (SMI)

The Rust compiler team is developing [Stable MIR](https://rust-lang.github.io/project-stable-mir/), an alternative API that provides a versioned, stable interface to compiler internals. It exposes MIR bodies, types, and associated metadata through a crate (`rustc_smir`) that abstracts over the internal representation. The goal is to allow external tools to query MIR without tracking nightly churn.

However, Stable MIR has limitations that make it unsuitable for RAPx's current needs:

- **Read-only access**: SMI provides MIR introspection but does not expose the full compiler callback infrastructure (`rustc_driver::Callbacks`, `after_analysis`, `RUSTC_WRAPPER` hook). RAPx needs to intercept compilation and run analyses between type checking and codegen.
- **Limited HIR access**: RAPx uses HIR for attribute scanning (`#[rapx::verify]`, `#[rapx::requires]`) and fast pre-filtering (`hir_contains_unsafe`). SMI focuses on MIR and does not expose HIR.
- **Evolving coverage**: SMI does not yet expose all MIR constructs that RAPx depends on (e.g., `TerminatorKind::InlineAsm`, `StatementKind::Intrinsic`, certain `AggregateKind` variants).
- **No standard library integration**: RAPx links against `core`/`alloc`/`std` to build contract databases and call effect summaries. SMI targets external tooling that operates on already-compiled crates.

RAPx tracks the Stable MIR effort and may adopt it for MIR querying in the future, while retaining nightly-only hooks for compilation interception and HIR access. The [`compat.rs`](https://github.com/safer-rust/RAPx/blob/main/rapx/src/compat.rs) abstraction layer is designed to make such a migration feasible: most analysis code already imports compiler types through centralized re-exports rather than directly from `rustc_middle`.
