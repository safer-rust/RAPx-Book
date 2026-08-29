# Chapter 2. Installation and Usage Guide

## Platform Support
RAPx supports the following three platforms:
* Linux (x86_64)
* macOS (x86_64)
* macOS (aarch64/Apple Silicon)

## Preparation

RAPx requires a nightly Rust toolchain with three components: `rustc-dev`, `rust-src`, `llvm-tools-preview`.

Three toolchain versions are supported:

| Toolchain | Status | CI Job |
|-----------|--------|--------|
| `nightly` (latest) | Default / always up-to-date | `latest` |
| `nightly-2026-07-21` | Pinned / tested | `asterinas` |
| `nightly-2025-11-25` | Pinned / tested | `verify-std` |

Install the recommended (latest) toolchain:

```shell
rustup toolchain install nightly --profile minimal --component rustc-dev,rust-src,llvm-tools-preview
```

The `main` branch pins `nightly` in its `rust-toolchain.toml`, so RAPx tracks the
latest nightly. If you need a specific pinned version instead (e.g. to match the
`asterinas` or `verify-std` CI jobs), install it explicitly:

```shell
rustup toolchain install nightly-2026-07-21 --profile minimal --component rustc-dev,rust-src,llvm-tools-preview
```

If you have multiple Rust versions, please ensure the correct version is set as default:
```shell
rustup show
```

## Install
### Download the project
```shell
git clone https://github.com/safer-rust/RAPx.git
```

### Build and install RAPx

```shell
./install.sh
```

You can combine the previous two steps into a single command:

```shell
cargo +nightly install rapx --git https://github.com/safer-rust/RAPx.git
```

For macOS users, you may encounter compilation errors related to Z3 headers and libraries. There are two solutions:

The first one is to manually export the headers and libraries as follows:
```
export C_INCLUDE_PATH=/opt/homebrew/Cellar/z3/VERSION/include:$C_INCLUDE_PATH
ln -s /opt/homebrew/Cellar/z3/VERSION/lib/libz3.dylib /usr/local/lib/libz3.dylib
```

Alternatively, you can modify the [Cargo.toml](https://github.com/safer-rust/RAPx/blob/main/rapx/Cargo.toml) file to change the dependency of Z3 to use static linkage. However, this may significantly slow down the installation process, so we do not recommend enabling this option by default.

```
[dependencies]
z3 = {version="0.13.3", features = ["static-link-z3"]}
```

After this step, you should be able to see the RAPx plugin for cargo.
```shell
cargo --list
```

Execute the following command to run RAPx and print the help message:
```shell
cargo rapx --help
Usage: cargo rapx [OPTIONS] <COMMAND> [-- [CARGO_FLAGS]]

Commands:
  analyze  perform various analyses on the crate, e.g., alias analysis, callgraph generation
  check    check potential vulnerabilities in the crate, e.g., use-after-free, memory leak
  opt      detect code optimization opportunities
  verify   verify annotated functions in the crate, e.g., identify #[rapx::verify] targets
  help     Print this message or the help of the given subcommand(s)

Options:
      --timeout <TIMEOUT>        specify the timeout seconds in running rapx
      --test-crate <TEST_CRATE>  specify the tested package in the workspace
  -h, --help                     Print help
  -V, --version                  Print version

NOTE: multiple detections can be processed in single run by 
appending the options to the arguments. Like `cargo rapx check -f -m`
will perform two kinds of detection in a row.

Examples:

  1. detect use-after-free and memory leak:
     cargo rapx check -f -m
  2. detect optimization opportunities:
     cargo rapx opt
  3. perform alias analysis:
     cargo rapx analyze alias
  4. verify annotated functions:
     cargo rapx verify --prepare-targets
```

### `analyze` command

```
Usage: cargo rapx analyze <COMMAND>

Commands:
  alias         alias analysis (meet-over-paths by default)
  adg           API dependency graphs
  safetyflow    unsafety propagation graph (safety flow analysis)
  safetyflowstd unsafety propagation graph on the standard library
  callgraph     callgraph generation
  dataflow      dataflow graphs
  heapowner     analyze heap-owning types
  paths         path-sensitive CFG paths
  pathcond      path constraint extraction
  range         range analysis
  scan          basic crate info
  ssa           print the SSA form of the crate
  mir           print MIR
  dot-mir       print MIR as DOT
  help          Print this message or the help of the given subcommand(s)

Options:
  -h, --help  Print help
```

Notable subcommand options:
- `alias -s <STRATEGY>` / `--strategy <STRATEGY>` — choose alias analysis strategy: `mop` (meet-over-paths, default) or `mfp` (maximum-fixed-point)
- `dataflow -d` / `--debug` — print debug information during dataflow analysis
- `dataflow --draw` — render dataflow graphs as PNG images (requires Graphviz)
- `safetyflow --draw` — render safety flow graphs as PNG images (requires Graphviz)
- `safetyflowstd --draw` — render safety flow graphs on stdlib as PNG images
- `paths --postfix-repeat <N>` — allow repeated SCC postfix segments (default 0)
- `range -d` / `--debug` — print debug information during range analysis
- `adg --include-private` — include private APIs in the API graph
- `adg --include-unsafe` — include unsafe APIs in the API graph
- `adg --include-drop` — include Drop impls in the API graph
- `adg --max-iteration <N>` — maximum generic iteration count (default 10)
- `adg --dump [PATH]` — dump API graph to file (default `./api_graph.dot`)

### `check` command

```
Usage: cargo rapx check [OPTIONS]

Options:
  -f, --uaf [<UAF>]    detect use-after-free/double-free (optional level, default 1)
  -m, --mleak          detect memory leakage
  -h, --help           Print help
```

### `opt` command

```
Usage: cargo rapx opt
```

### `verify` command

The `verify` command provides a contract-based verification pipeline for functions annotated with `#[rapx::verify]`. It uses path-sensitive backward/forward analysis and Z3-based SMT solving to prove safety properties. RAPx supports two verification modes (see [Chapter 8](./8-verification.md) for details).

```
Usage: cargo rapx verify [OPTIONS]

Options:
      --prepare-targets            identify #[rapx::verify] functions and list their safety contracts
      --debug-contracts            print contract resolutions per target and exit (no verification)
      --skip-invariant             skip struct invariant checks, derive safety via
                              constructor-mutator-method chains (works with both modes)
      --postfix-repeat <N|auto>
                              extra SCC postfix repetitions during path enumeration;
                              `auto` is the default planner, a number fixes the repeat count
      --mode <MODE>               verification mode: scan, targeted (default scan)
      --crate <CRATE>             filter verification targets to the named crate
      --module <PATH>             filter verification targets to the named module path
  -h, --help                      Print help
```

Verification modes:
- `scan` — auto-detect: verify all functions with unsafe callees or struct invariants
- `targeted` — only verify functions annotated with `#[rapx::verify]`

The `--skip-invariant` flag works with both modes to skip struct invariant checks and instead derive safety through constructor-mutator-method chains.

The `--postfix-repeat` option controls loop unrolling depth:
- `auto` (default) — automatic loop-depth detection for `InBound` and `Align` contracts
- A fixed number `N` ≥ 0 — repeat loop body up to N+1 times

**Filtering targets:** `--crate` and `--module` work in any mode and can be combined:

```bash
cargo rapx verify --module my_mod::sub
cargo rapx verify --crate core --module ptr::NonNull
```

### Environment Variables

| var             | default when absent | possible values     | description                  |
|-----------------|---------------------|---------------------|------------------------------|
| `RAPX_LOG`       | info                | trace, debug, info, warn | verbosity of logging   |
| `RAPX_CLEAN`     | true                | true, false         | run cargo clean before check |
| `RAPX_RECURSIVE` | none                | none, shallow, deep | scope of packages to check   |
| `RAPXFLAGS`      | (unset)             | CLI arguments       | arguments passed to `rapx` binary directly |

For `RAPX_RECURSIVE`:
* `none`: check for current folder
* `shallow`: check for current workspace members
* `deep`: check for all workspaces from current folder

### Uninstall
```shell
cargo uninstall rapx
```


