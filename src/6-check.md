# Chapter 6. Check

This chapter introduces the bug detection modules of RAPx, covering security-focused use cases. RAPx supports detection of dangling pointers and memory leaks. Performance optimization is covered as a standalone module in [Chapter 7](./7-opt.md), and unsafe code verification in [Chapter 8](./8-verification.md).

## Module Overview

The `check` module at [`rapx/src/check/`](https://github.com/safer-rust/RAPx/blob/main/rapx/src/check/) contains two main sub-modules:

| Module | Purpose | Command | Chapter |
|--------|---------|---------|---------|
| `safedrop/` | Dangling pointer (use-after-free/double-free) detection | `cargo rapx check -f` | [6.1](./6.1-dangling.md) |
| `rcanary/` | Memory leak detection via SMT-based analysis | `cargo rapx check -m` | [6.2](./6.2-memleak.md) |

All check modules are invoked through the `cargo rapx check` sub-command with appropriate flags. Multiple flags can be combined to run several detectors simultaneously.

## Architecture

Each check module follows a common architecture:

1. Alias Preparation: The MOP-based [alias analysis](./5.2-alias.md) is run first to compute function-level alias summaries.
2. Path-Sensitive Traversal: Using [`PathGraph`](./5.1-path.md), each function's execution paths are enumerated and analyzed independently.
3. Bug Pattern Matching: Each module applies domain-specific rules along each path to identify bug patterns (dangling pointer usage, leaked allocations).
4. Warning Emission: Detected bugs are reported with file locations, function names, and diagnostic messages.

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
cargo rapx check -f -m
```
