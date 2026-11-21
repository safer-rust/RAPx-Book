# Chapter 6. Symbolic VM

RAPx uses a shared symbolic VM to execute Rust MIR along selected paths. The VM is implemented in [`rapx/src/vm.rs`](https://github.com/safer-rust/RAPx/blob/main/rapx/src/vm.rs) and [`rapx/src/vm/`](https://github.com/safer-rust/RAPx/tree/main/rapx/src/vm). [SafeDrop](./7.1-dangling.md), [rCanary](./7.2-memleak.md), and [verification](./8-verification.md) use its execution and memory model for different checks.

## Execution Model

Path selection happens outside the VM. A checker provides MIR items from a selected path, or a backward slice of that path, to `VmExecutor`. The executor initializes symbolic parameters and processes statements and terminators in execution order. Assignments update local values; calls use modeled effects or available MIR, with caller frames saved during inlining. When execution branches, the VM can produce separate states with their own path conditions.

```text
Selected MIR path or slice
          │
          ▼
     VmExecutor
          │
          ├─ symbolic values and path constraints
          ├─ allocations, contents, and pointer provenance
          └─ call frames and optional memory events
          │
          ▼
   Path-local VM states ──► SafeDrop / rCanary / verifier
```

The VM does not choose which functions to analyze or which safety properties to prove. Each caller decides how to enumerate paths, which MIR items to execute, and how to interpret the resulting states.

### Executing a path

[`RelevantItem`](https://github.com/safer-rust/RAPx/blob/main/rapx/src/vm/step.rs) is the input to [`VmExecutor::execute_items`](https://github.com/safer-rust/RAPx/blob/main/rapx/src/vm/exec.rs). It has four forms:

| Item | What the VM does |
|------|------------------|
| `Statement` | Loads one statement from its function, basic block, and statement index, then applies its MIR rule. |
| `Terminator` | Loads the block terminator and applies its rule. For `SwitchInt`, `switch_succ` identifies the successor selected by the supplied path. |
| `CalleeEntry` | Saves the caller frame, enters the callee frame, and binds call arguments to parameters. |
| `CalleeExit` | Restores the caller frame and writes the callee return value to the call destination. |

`execute_items` initializes symbolic parameters, then replays these items in order over a vector of VM states. A conditional inside an inlined callee can add pending states; the executor carries each surviving state forward separately. A selected branch that contradicts a known condition is pruned. If a MIR operation is unsupported, the state records a diagnostic and stops executing later items; it cannot be treated as a proved path. The verifier can reuse VM states for a common path prefix through its [`VmCache`](https://github.com/safer-rust/RAPx/blob/main/rapx/src/verify/proof_cache.rs).

### MIR execution rules

The [MIR dispatch table](https://github.com/safer-rust/RAPx/blob/main/rapx/src/vm/mir_rules/dispatch.rs) selects a rule by variant. The following groups describe how the main supported operations change a state:

| MIR operation | State update |
|---------------|--------------|
| `Assign(place, rvalue)` | Evaluates the rvalue, then stores the resulting `VmValue` in the destination local, field, or pointed-to memory. `Use` copies or moves an operand; `Ref` and `RawPtr` construct pointer values with provenance when known. Arithmetic and casts produce symbolic terms and propagate supported facts. `Aggregate` and `Discriminant` build or inspect aggregate and enum values. |
| `SetDiscriminant` | Sets an enum's active variant and checks whether its payload was materialized. |
| `StorageLive` / `StorageDead` | Creates or ends a local's stack storage and updates its lifetime. |
| Statement intrinsics `Assume` / `CopyNonOverlapping` | Adds the assumption to path constraints, or copies the modeled memory values from source to destination. These are MIR statements, even when the source program called an intrinsic function. |
| `SwitchInt` / `Assert` | Adds the chosen branch or assertion condition to path constraints; a known incompatible `SwitchInt` successor is pruned. |
| `Drop` | Applies the modeled destruction and lifetime effects, including optional memory events. |
| `Call` / `TailCall` | Selects a call model: a registered effect or postcondition, constructor result, available callee MIR for inlining, or a diagnostic for an unsupported or missing implementation. `TailCall` writes to the current frame's return local. |

Other supported rvalues include unary operations, repeats, and thread-local references. Bookkeeping statements and control-only terminators such as `Goto` and `Return` need no additional value rule: the supplied item sequence already fixes their order. The dispatch table records an unsupported operation when no rule applies, and some recognized variants can still report that their particular operand or layout is unsupported. [§8.5](./8.5-verification-approach.md) describes how the verifier checks properties against the resulting states.

## Values and Memory

`VmValue` carries a symbolic Z3 term together with its Rust type and tracked facts. Memory is represented by allocation units identified by `AllocId`. A pointer can retain its allocation provenance and offset as it moves through locals, fields, calls, and returns. The VM records allocation size, alignment, contents, initialization, and lifetime information where its MIR rules can establish them.

Path conditions and value facts are kept with each state. A checker can ask Z3 whether a property follows from those constraints, or use structural facts from the VM directly. For example, the verifier checks a pointer precondition at an unsafe call; SafeDrop observes uses and drops of an allocation; rCanary follows an allocation after ownership is converted to a raw pointer.

Memory events are optional. SafeDrop enables them to receive use and drop records with source locations. The other clients can inspect allocation state without turning every memory operation into a reported event.

## VM State

`VmExecutor` owns immutable compiler and Z3 context references plus a path-local `VmState`. The main parts of that state are:

- `current_frame.local_alloc`: each MIR local's stack allocation in the current frame. `caller_frames` saves frames while callees execute.
- `mem_units`: allocation shape, contents, and facts indexed by `AllocId`; `static_allocs` maps statics to stable allocations.
- `constraints`: path conditions and cached symbolic terms.
- `inline`: recursion depth, pending branch states, argument referents, and deferred writes during inlining.
- `alias_hazard_allocs`: allocation roots licensed by declared Alias contracts.
- `memory_events`: optional use/drop records enabled by SafeDrop.

When inlining reaches `MAX_INLINE_DEPTH`, the destination receives a tracked placeholder. A dependent verification obligation becomes `Unknown` (`InlineDepthExceeded`).

### Values and facts

A MIR local's whole value (its rvalue) and each of its field values are both symbolic values (`VmValue`), stored in `MemoryContent::values` keyed by `(view_ty, path)` within the local's allocation (each `MemoryUnit` *is* one allocation, so the `AllocId` is the unit index, not part of the key) — `path == []` is the whole value, `path == [i, ..]` is field `i` (viewed as `view_ty`, so a reinterpret cast resolves to the right field view):

```text
VmValue { z3_term, ty, provenance, facts, source }
```

- `z3_term` — a Z3 term. For pointer/reference types (`&T`, `*const T`, `Box<T>`, …) it is the address; for scalar types it is the value; for aggregate types it is the base address of the stack slot backing the aggregate.
- `ty` — the Rust type, for layout queries.
- `provenance` — `Option<Provenance>` (see below); `None` for a scalar.
- `facts` — the derived value facts (`ValueFacts`) that serve as the "fast-path flags" of [§8.5.1](./8.5-verification-approach.md#851-deduction-rules):
  - `non_null: bool`, `init: bool`, `in_bounds: bool`.
  - `align_n: Option<Int>` — the pointer's *alignment lower bound*. Here `Int` is `z3::ast::Int`, a **Z3 term**, not a Rust integer: `Some(n)` asserts `z3_term % n == 0`, i.e. the pointer is at least `n`-aligned (alignments are powers of two, so "at least `n`" ⇔ "a multiple of `n`"). For a *concrete* type `n` is a constant term (`Some(8)`); for a *generic* type `n` is the *symbolic* constant `align_T` — a fresh Z3 constant shared per type — so a generic `T` is carried as a symbol, not a number. `None` records **no** alignment fact (e.g. a raw pointer `*mut T`), independent of whether `T` is generic.
- `source` — a `ValueSource` carrying any extra semantics beyond z3_term/type/provenance:
  - `None` — a plain value.
  - `FieldOffset` — a compile-time `offset_of!` field offset, propagated to a pointer's provenance when used as an `add`/`byte_add` offset.
  - `KnownDiscriminant(Int)` — a symbolic enum variant index (e.g. `Iterator::next` returns `Some(x)` iff `!is_empty`), tying `switchInt` branches to the real condition instead of a fresh symbol.
  - `Comparison { lhs, rhs, op, cond }` — a comparison result (`Le`/`Lt`/`Ge`/`Gt`/`Eq`/`Ne`): the operands plus the direct boolean condition (`offset <= len`), so `switchInt`/`Assert` emit a precise path condition instead of `ite(cond, 1, 0) != 0`.
  - `BinaryOp { lhs, rhs, op }` — a non-comparison binary-op result (`Add`/`Sub`/…/`Div`/`Rem`), used for guard inference (tracing a switch/assert guard back to the pointer it null-checks/alignment-checks) and division-axiom injection.

These facts come from MIR operations or declared contracts. A raw-pointer field (`*mut T`, `*const T`, or `NonNull<T>`) carries no implicit `non_null`, `init`, or alignment fact. An invariant can supply such a fact, while a null guard can establish non-nullness on a particular path. A concrete `Align` invariant also adds `p.z3_term % A == 0` to the path constraints, so the alignment fact can survive pointer arithmetic that drops the value-level flag.

### Allocation table

Every allocation `a` the VM tracks is one `MemoryUnit` (`vm::state::MemoryUnit`, stored at `mem_units[a]`), split into its shape (`allocation`) and its contents (`content`). Its fields, grouped as in the code:

- **Shape** (always present):
  - `base: Int` — base address (a fresh Z3 constant).
  - `size: Int` — byte size (may be symbolic; an external parameter is sized as "unbounded").
  - `align: Int` — alignment in bytes (may be symbolic for a generic element type).
  - `element_ty: ElementTy` — the contents' type: `Typed(T)` for a concrete `Ty` (or `u8` for a raw byte buffer), or `Generic` for a symbolic `T`.
  - `kind: AllocKind` — `Object` (one object or raw buffer), `Slice { len }` (a slice/array buffer), `External` (caller-owned memory with unknown extent), or `ExternalSlice { len }` (a caller-owned slice with a tracked length). Nullability is tracked on the pointer term, not on the allocation kind.
- **Lifecycle facts** (`allocation.facts: AllocFacts`) — cross-cutting per-allocation *lifecycle* facts, kept apart from the shape metadata above:
  - `dead: bool` — freed (`StorageDead`/`Drop`).
  - `liveness: Option<Region>` — the region an *external* allocation is assumed alive for (`Alive(p, 'a)` contract / `'static`); only consulted for external allocations, whose memory is caller-owned (so `dead` carries no guarantee). `None` = no assumption.
  - `for_each: ForEachFacts` — uniform facts about the container's *pointer elements* (`x.iter()` for_each invariants), anchored to the allocation so a pointer loaded from it resolves the fact via its provenance: `target_ty` (Typed), `aligned_ty` (Align), `allocated: (Ty, n)` (Allocated, `n` may be symbolic), `owning` (Owning).
- **Content facts** (`content.facts: ContentFacts`) — per-allocation *content* facts, kept next to the stored bytes/values rather than on the lifecycle layer:
  - `initialized: bool` — contents hold a readable value (`Box::new`/`Vec`/reference referent/callee return/write/`ValidCStr`/const data); `false` for `new_uninit`/`uninit`.
  - `cstr_trusted: bool` — a null-terminated byte buffer (valid C string).
  - `utf8_trusted: bool` — a valid UTF-8 buffer (`ValidString`).
  - `split_transmute_to: Option<Ty>` — the destination element type this allocation is licensed to be re-interpreted into by a `SplitTransmute([T], [U])` contract (the slice's data buffer, reached through the slice value's provenance); `None` = no license.
- **Relationships** (optional):
  - `parent: Option<AllocId>` — the source allocation of a sub-view (`s[i..j]`/`get`/`split_at`/`align_to`/`as_chunks`/`from_raw_parts`); `root_alloc` follows this edge to group aliasing views.

The data allocation of a two-part value (`Vec`/`String`/`CString`) is **not** stored as an edge: it is re-derived on demand from the header's *owning pointer field* — the field whose provenance names the heap buffer, located generically by `container_ptr_field` (`Box.0.pointer` → `[0, 0]`, `Vec.buf.ptr.pointer` → `[0, 0, 0]`). `data_alloc_of` reads that field's provenance; for a slice view whose header `AllocId` no longer carries a type key it falls back to `header_data_alloc` (scan the header's materialized fields for the single owning-pointer provenance). A `&[T]` reference needs neither — its provenance already names the data.

The entry differs by how the allocation was created. With `--expand-ctor=default`, a constructor directly returning a struct with a source or JSON invariant can be summarized; the VM materializes its result and asserts the declared invariant. `--expand-ctor=true` executes available constructor MIR instead.

| creation | kind | element_ty | parent |
|---|---|---|---|
| `Box::new(v)` — heap object | `Object` | `Typed(T)` | — |
| Inlined `Vec` constructor's heap buffer | `Slice { len }` | `Typed(T)` | — |
| `Vec`/`String`/`CString` parameter or summarized constructor's heap buffer | `External` (symbolic capacity or unbounded) | `Typed(T)` | — |
| `&[T]` reference's stack slot | `Object` | `Typed(T)` | — |
| `split_at` half / `from_raw_parts` view | `Slice { mid }` | `Typed(T)` | `Some(src)` |
| external `*mut T` parameter | `External` | `Typed(T)` | — |

### Provenance

A pointer with known provenance carries the allocation it points into, the byte offset within it, and (when known) the structure of that offset:

```text
provenance(p) = { alloc_id: a, offset: off, offset_kind: k }   ⇔   p points into allocation a at byte offset off
```

`offset_kind` classifies the offset so the checker can use a cheaper, more precise proof: `Field` carries the "field in-bounds" guarantee (`offset + size_of(field) <= size_of(container)`, e.g. `Option::as_slice`); `Element(i)` tracks the element index so `InBound` checks `i + count <= slice_len` linearly instead of the non-linear byte form `(i+count)·S <= len·S` (undecidable in Z3 NIA for a generic `S`); `Byte` marks a byte-strided (`byte_add`) or unclassifiable offset. `offset_kind` is `None` at the base (`offset == 0`).

A cast (`&mut → *mut`, `*mut → *const`) preserves provenance; a plain copy `q = p` copies provenance (so `p` and `q` are aliases).

### Provenance recording rules

The VM maintains provenance in execution order, one transfer per MIR statement/terminator. Below, `prov(p) = (a, off)` abbreviates the full `provenance(p) = { alloc_id: a, offset: off, offset_kind }`, and `slice_len` denotes `AllocKind::Slice { len }` (read via `Allocation::slice_len()`):

```text
── Allocation creation (inlined or summarized Box::new / Vec::new / …) ──
a := fresh
──────────────────────────────────────────
prov(p) = (a, 0)      a.parent = ⊥         (a.kind = Slice { len } for a Vec/slice)

── View of a pointer (&*p / &mut *p / as_ptr / as_mut_ptr / into_raw) ──
prov(p) = (a, off)
────────────────────────
prov(&*p) = prov(p) = (a, off)          (the view inherits p's provenance)

── Slice view (from_raw_parts(ptr, n)) ──
prov(ptr) = (a, off)
────────────────────────────────────────
b := fresh   prov(s) = (b, 0)   b.kind = Slice { len: n }   b.parent = a

── Split (split_at(s, mid)) ──
prov(s) = (a, 0)   a.kind = Slice { len }   0 ≤ mid ≤ len
──────────────────────────────────────────────────────────────
prov(s[..mid]) = (b₁, 0)   b₁.kind = Slice { len: mid }      b₁.parent = a
prov(s[mid..]) = (b₂, 0)   b₂.kind = Slice { len: len−mid }   b₂.parent = a

── Element arithmetic (p.add(k)) ──
prov(p) = (a, off)
──────────────────────────────
prov(p.add(k)) = (a, off + k·S)      S = sizeof(element)

── Byte arithmetic (p.byte_add(k)) ──
prov(p) = (a, off)
──────────────────────────────
prov(p.byte_add(k)) = (a, off + k)

── Cast reinterprets a buffer (byte ↔ field) ──
A `*mut u8` ↔ `*mut T` cast switches the *view* of an allocation without
changing its bytes.  A *scalar* field round-trips across the two views
(little-endian, concrete size); a *pointer* field does not — provenance is not a
byte, so a byte→field read of a pointer field mints a fresh symbol:

  byte → field:  field(i).z3_term = Σⱼ byte[off(i) + j] · 256ʲ
  field → byte:  byte[off(i) + j]  = (field(i).z3_term ÷ 256ʲ) mod 256

where `off(i)` is field `i`'s byte offset and `256ʲ` is its byte-weight.
```

### Allocation-root grouping

The verifier groups *live* reference views `p` and `q` as potentially aliasing when they name the same allocation or sub-allocations of it:

```text
prov(p) = (a, _)    prov(q) = (b, _)
root(a) == root(b)  where root(a) follows parent(a) to the top (⊥)
```

A view is *live* when its MIR local is between its `StorageLive` and `StorageDead` (and not moved). The `parent` chain makes a slice view (`from_raw_parts`) or a `split_at` half share a grouping key with its source, even though each has a *distinct* `AllocId`. The verifier performs further alias hazard checks on those candidates; sharing a root does not by itself prove that byte ranges overlap.

## State Construction

The VM adds constraints and memory facts as it executes. The following relations summarize representative rules. They describe the model that the verifier reads; they are not separate proof obligations. Here `S = sizeof(T)` and `A = align(T)`.

```text
── Slice type invariant ──
NonNull(s) ∧ Align(s, T) ∧ ValidNum(len·S ≤ isize::MAX) ∧ Allocated(s, T, len) ∧ Init(s, T, len)

── Layout ──
──────────        ──────────        ──────────────────         ────────
S ≥ 0             A ≥ 1             S % A == 0                 len ≥ 0

For a *generic* `T` (no concrete layout), `S` and `A` are the shared symbolic
constants `sizeof_T` / `align_T`.  `size_of::<T>()` / `align_of::<T>()` on such a
`T` bind to these same symbols (not `0`/`1`), so they agree with allocation
sizes and pointer strides; `align_of::<T>() ≥ 1` keeps a cast like
`align as *const T` in `NonNull::dangling` non-null.

── Allocation (s : &[T]) ──
prov(s) = (a, 0)   a.kind = Slice { len }   a.size = len·S   a.align = A   a.initialized = true

── Pointer identity ──
prov(p) = (a, off)
─────────────────────
p.z3_term = a.base + off

── Pointer arithmetic ──
prov(p) = (a, off)                      prov(p) = (a, off)
──────────────────────────────          ──────────────────────────────
prov(p.add(k)) = (a, off + k·S)         (p.add(k)).z3_term = p.z3_term + k·S

── Alignment offset ──
prov(p) = (a, off)    S = sizeof(pointee)    A' = align_of::<U>()
────────────────────────────────────────────────────────────────────────────
k = align_offset(p, A')
k = usize::MAX  ∨  (0 ≤ k < A' ∧ (p.z3_term + k·S) % A' == 0)

`align_offset` is modelled as returning the offset in *elements* of the pointee
type (not bytes), so the aligned address is `p.z3_term + k·S`; a downstream
`p.add(k) as *const U` can then discharge `Align(p, U)`.  The bound `k < A'`
covers successful offsets, but `A' / S` is not their general upper bound:
the stride and requested alignment determine which offsets are reachable.
The VM also permits `usize::MAX` as the failure result when the alignment cannot be reached.

── Byte-range case split (generic S) ──
S = 0                                          S ≥ 1
──────────────────────                        ──────────────────────
x·S = y·S = 0   (byte range vacuous)           x·S ≤ y·S ⟺ x ≤ y

── Integer arithmetic ──
i < len
─────────────
i + 1 ≤ len
```

## Scope and Clients

| Client | Provides | Uses from the VM |
|--------|----------|------------------|
| [SafeDrop](./7.1-dangling.md) | Paths through eligible local functions | Allocation identity, lifetime, and use/drop events |
| [rCanary](./7.2-memleak.md) | Paths from ownership-release candidates to return | Allocation provenance and lifetime; the checker handles escape |
| [Verifier](./8.4-pipeline.md) | Paths and relevant MIR items at unsafe checkpoints | Symbolic values, memory facts, and constraints for property checks |

The results are bounded by the paths explored and the MIR operations modeled. A truncated path set or unsupported operation cannot justify a claim about all executions. The verifier represents incomplete proofs as `Unknown`; each bug checker applies its own reporting conditions to supported paths.
