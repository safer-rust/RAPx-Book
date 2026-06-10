# Chapter 2. Installation and Usage Guide

## Platform Support
* Linux
* macOS (both x86_64 and aarch64 version)

## Preparation
The latest RAPx is developped based on Rust version nightly-2025-12-06. You can install this version using the following command.
```shell
rustup toolchain install nightly-2025-12-06 --profile minimal --component rustc-dev,rust-src,llvm-tools-preview
```

If you have multiple Rust versions, please ensure the default version is set to nightly-2025-12-06.
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
cargo +nightly-2025-12-06 install rapx --git https://github.com/safer-rust/RAPx.git
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
     cargo rapx check -o report
  3. perform alias analysis:
     cargo rapx analyze alias
  4. verify annotated functions:
     cargo rapx verify --prepare-targets
```

### Uninstall
```shell
cargo uninstall rapx
```



