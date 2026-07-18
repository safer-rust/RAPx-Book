# Chapter 10. Case Study

## 10.1 Verifying `core::slice` Safety (Challenge 17)

This case study demonstrates how RAPx verifies safety contracts in the Rust standard library's `slice` module, targeting [Challenge 17: Verify the safety of `slice` functions](https://model-checking.github.io/verify-rust-std/challenges/0017-slice.html) from the [Verify Rust Std Lib](https://github.com/model-checking/verify-rust-std) project.

### Goal

Prove that 30+ unsafe and safe functions in `library/core/src/slice/mod.rs` are free of undefined behavior by:

- Writing `#[rapx::requires(...)]` safety preconditions on unsafe callees
- Adding `#[rapx::verify]` annotations to trigger verification
- Running RAPx in `targeted` mode against the annotated functions

### Tool Integration

Following the same conditional pattern as Kani (`#[cfg_attr(kani, ...)]`) and Flux (`#[cfg(flux)]`), the project uses `cfg_attr` to avoid hard-coding `register_tool` in source code:

- `core/src/lib.rs`: No `#![register_tool(rapx)]` — tool registration is injected via `RUSTFLAGS`
- `core/Cargo.toml`: `['cfg(rapx)']` declared in `[lints.rust.unexpected_cfgs]` to suppress unknown cfg warnings
- Source files: `#[cfg_attr(rapx, rapx::verify)]` and `#[cfg_attr(rapx, rapx::requires(...))]` conditionally activate only when RAPx runs

```rust
// Example from core/src/slice/mod.rs
#[cfg_attr(rapx, rapx::verify)]
#[cfg_attr(rapx, rapx::requires(InBound(index_access(self, a))))]
pub fn swap(&mut self, a: usize, b: usize) { /* ... */ }
```

```rust
// Example from core/src/ptr/mod.rs
#[cfg_attr(rapx, rapx::requires(Align(src, T)))]
#[cfg_attr(rapx, rapx::requires(Align(dst, T)))]
#[cfg_attr(rapx, rapx::requires(ValidPtr(src, T, count)))]
#[cfg_attr(rapx, rapx::requires(ValidPtr(dst, T, count)))]
pub unsafe fn copy_nonoverlapping<T>(src: *const T, dst: *mut T, count: usize) { /* ... */ }
```

### Commands

**Prepare targets** (preview what will be verified without running full verification):

```shell
cd library/core
RUSTFLAGS="--cfg=rapx -Zcrate-attr=feature(register_tool) -Zcrate-attr=register_tool(rapx)" \
  cargo rapx verify --module slice --prepare-targets
```

Output:

```
[rapx::verify] total: 80 free function(s), 21 method(s), 9 struct(s), 0 trait(s)
```

**Run verification:**

```shell
cd library/core
RUSTFLAGS="--cfg=rapx -Zcrate-attr=feature(register_tool) -Zcrate-attr=register_tool(rapx)" \
  cargo rapx verify --module slice --mode targeted
```

> **Note:** Commands must be run from `library/core`, not the `library` workspace
> root. In the `library` workspace, `core` is only a *dependency* (the members
> are `std`, `sysroot`, `coretests`, `alloctests`), and RAPx analyzes a crate
> only when it is compiled as the primary package. Running from `library/core`
> makes `core` local, so its `#[rapx::verify]` annotations are visible without
> needing the `--crate core` filter.


The RUSTFLAGS environment variable provides three things:

| Flag | Purpose |
|------|---------|
| `--cfg=rapx` | Activates `cfg_attr(rapx, ...)` conditional expansion |
| `-Zcrate-attr=feature(register_tool)` | Enables the `register_tool` nightly feature |
| `-Zcrate-attr=register_tool(rapx)` | Registers `rapx` as a recognized tool namespace |

### CI Integration

A GitHub Actions workflow (`.github/workflows/rapx.yml`) runs verification on every push and pull request, using the same RUSTFLAGS setup. It pins a specific RAPx commit via `RAPX_VERSION`, `cd`s into `library/core`, and invokes `cargo rapx verify --module slice --mode targeted`.

### Verification Results

Running `cargo rapx verify --module slice --mode targeted` produces output grouped by function. Below is a typical result excerpt:

```
============================================================
[rapx::verify] function: core::slice::<impl [T]>::swap
============================================================
  --- unsafe checkpoints ---
      unsafe checkpoint: bb3 -> core::intrinsics::transmute_unchecked
        path [0, 1, 2, 3]:
          ValidPtr | Proved
          Align | Proved
      unsafe checkpoint: bb13 -> core::slice::get_unchecked_mut
        path [0, 1, 5, 6, 7, 8, 9, 10, 13]:
          InBound | Proved
          NonOverlap | Proved
  --- struct invariants ---
      checkpoint bb14:
        path [0, 1, 5, 6, 7, 8, 9, 10, 11, 12, 14]:
          ValidPtr | Proved
          Align | Proved
  result: SOUND
============================================================

[rapx::verify] function: core::slice::<impl [T]>::get_unchecked_mut
============================================================
  --- unsafe checkpoints ---
      unsafe checkpoint: bb2 -> core::ptr::mut_ptr::add
        path [0, 1, 2]:
          Align | Proved
          InBound | Proved
  result: SOUND
```

### Code Reference

- **Source:** [`safer-rust/rapx-verify-rust-std`](https://github.com/safer-rust/rapx-verify-rust-std)
- **Challenge:** [#281 / 0017-slice](https://github.com/model-checking/verify-rust-std/issues/281)

## 10.2 Asterinas

[Asterinas](https://github.com/asterinas/asterinas) uses an older toolchain (2025-02-01).
To apply RAPx, a few minor modifications are required (see an example [here](https://github.com/Artisan-Lab/asterinas-rapx)). Then, RAPx can be applied using the following command.

```shell
cd ostd
cargo rapx check -f -- --target x86_64-unknown-none
```
