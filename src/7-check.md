# Chapter 7. Check

RAPx provides detectors for memory safety bugs and performance issues. [SafeDrop](./7.1-dangling.md) and [rCanary](./7.2-memleak.md) use the shared [symbolic VM](./6-vm.md) to follow allocations along paths. [Optimization](./7.3-opt.md) uses dataflow analysis to identify inefficient code. Unsafe code contract verification is covered in [Chapter 8](./8-verification.md).

| Section | Purpose | Command |
|---------|---------|---------|
| [7.1 Dangling Pointer Detection](./7.1-dangling.md) | Use-after-free and double-free detection | `cargo rapx check -f` |
| [7.2 Memory Leakage Detection](./7.2-memleak.md) | Leaks after ownership is released to a raw pointer | `cargo rapx check -m` |
| [7.3 Optimization](./7.3-opt.md) | Performance pattern checks | `cargo rapx opt` |

The two memory safety detectors run through `cargo rapx check`; their flags can be combined. Optimization is documented here as another checker but has its own `opt` command and analysis pipeline.
