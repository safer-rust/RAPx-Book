# Chapter 6. RAPx Applications
This section introduces several applications of RAPx, covering both security and performance use cases. For security, RAPx currently supports the detection of dangling pointer and memory leak bugs, and it can be used to audit unsafe code through three verification modes. Please refer to the sub-chapters for more information.

## Default Options for Rust Project Compilation

To analyze system software without std (e.g., [Asterinas](https://github.com/asterinas/asterinas)), try the following command:
```shell
cargo rapx check -f -- --target x86_64-unknown-none
```

To analyze the Rust standard library, try the following command:
```shell
cargo rapx check -f -- -Z build-std --target x86_64-unknown-linux-gnu
```

## Verification Modes Overview

RAPx's unsafe code verification (Chapter 6.4) supports three modes for different use cases:

| Mode | Command | Use Case |
|------|---------|----------|
| `scan` (default) | `cargo rapx verify` | Auto-detect and verify all functions with unsafe code or struct invariants |
| `targeted` | `cargo rapx verify --mode targeted` | Only verify functions annotated with `#[rapx::verify]` |
| `invless` | `cargo rapx verify --mode invless` | Verify without struct invariants, deriving safety requirements automatically |
