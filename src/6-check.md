# Chapter 6. Check

This chapter introduces the bug detection modules of RAPx, covering both security and performance use cases. RAPx supports detection of dangling pointers, memory leaks, and performance anti-patterns. Unsafe code verification is covered separately in Chapter 7.

## Module Overview

The `check` module at `rapx/src/check/` contains three main sub-modules:

| Module | Purpose | Command | Chapter |
|--------|---------|---------|---------|
| `safedrop/` | Dangling pointer (use-after-free/double-free) detection | `cargo rapx check -f` | 6.1 |
| `rcanary/` | Memory leak detection via SMT-based analysis | `cargo rapx check -m` | 6.2 |
| `opt/` | Performance anti-pattern detection | `cargo rapx check -o` | 6.3 |

All check modules are invoked through the `cargo rapx check` sub-command with appropriate flags. Multiple flags can be combined to run several detectors simultaneously.

## Architecture

Each check module follows a common architecture:

1. **Alias Preparation**: The MOP-based alias analysis (Chapter 5.2.2) is run first to compute function-level alias summaries. These summaries enable inter-procedural tracking without re-analyzing callees repeatedly.
2. **Path-Sensitive Traversal**: Using `PathGraph` (Chapter 5.1), each function's execution paths are enumerated and analyzed independently. The path-sensitive approach enables precise, context-aware bug detection that avoids false positives from path merging.
3. **Bug Pattern Matching**: Each module applies domain-specific rules along each path to identify bug patterns (dangling pointer usage, leaked allocations, performance anti-patterns, lock misuse).
4. **Warning Emission**: Detected bugs are reported with file locations, function names, and diagnostic messages.

## Default Options for Rust Project Compilation

To analyze system software without std (e.g., [Asterinas](https://github.com/asterinas/asterinas)), try:

```shell
cargo rapx check -f -- --target x86_64-unknown-none
```

To analyze the Rust standard library:

```shell
cargo rapx check -f -- -Z build-std --target x86_64-unknown-linux-gnu
```

To run all check modules at once:

```shell
cargo rapx check -f -m -o
```
