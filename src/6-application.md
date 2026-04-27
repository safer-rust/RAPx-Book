# Chapter 6. RAPx Applications
This section introduces several applications of RAPx, covering both security and performance use cases. For security, RAP currently supports the detection of dangling pointer and memory leak bugs, and it can be used to audit unsafe code. Please refer to the sub-chapters for more information.

## Default Options for Rust Project Compilation

To analyze system software without std (e.g., [Asterinas](https://github.com/asterinas/asterinas)), try the following command:
```shell
cargo rapx check -f -- --target x86_64-unknown-none
```

To analyze the Rust standard library, try the following command:
```shell
cargo rapx check -f -- -Z build-std --target x86_64-unknown-linux-gnu
```
