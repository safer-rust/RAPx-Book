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

## Values and Memory

`VmValue` carries a symbolic Z3 term together with its Rust type and tracked facts. Memory is represented by allocation units identified by `AllocId`. A pointer can retain its allocation provenance and offset as it moves through locals, fields, calls, and returns. The VM records allocation size, alignment, contents, initialization, and lifetime information where its MIR rules can establish them.

Path conditions and value facts are kept with each state. A checker can ask Z3 whether a property follows from those constraints, or use structural facts from the VM directly. For example, the verifier checks a pointer precondition at an unsafe call; SafeDrop observes uses and drops of an allocation; rCanary follows an allocation after ownership is converted to a raw pointer.

Memory events are optional. SafeDrop enables them to receive use and drop records with source locations. The other clients can inspect allocation state without turning every memory operation into a reported event.

## Scope and Clients

| Client | Provides | Uses from the VM |
|--------|----------|------------------|
| [SafeDrop](./7.1-dangling.md) | Paths through eligible local functions | Allocation identity, lifetime, and use/drop events |
| [rCanary](./7.2-memleak.md) | Paths from ownership-release candidates to return | Allocation provenance and lifetime; the checker handles escape |
| [Verifier](./8.4-pipeline.md) | Paths and relevant MIR items at unsafe checkpoints | Symbolic values, memory facts, and constraints for property checks |

The results are bounded by the paths explored and the MIR operations modeled. A truncated path set or unsupported operation cannot justify a claim about all executions. The verifier represents incomplete proofs as `Unknown`; each bug checker applies its own reporting conditions to supported paths.
