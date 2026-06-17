# Chapter 6. Check

This chapter introduces the bug detection modules of RAPx, covering both security and performance use cases. RAPx supports the detection of dangling pointer and memory leak bugs, as well as performance anti-pattern checks. Unsafe code verification is covered separately in Chapter 7.

## Default Options for Rust Project Compilation

To analyze system software without std (e.g., [Asterinas](https://github.com/asterinas/asterinas)), try the following command:
```shell
cargo rapx check -f -- --target x86_64-unknown-none
```

To analyze the Rust standard library, try the following command:
```shell
cargo rapx check -f -- -Z build-std --target x86_64-unknown-linux-gnu
```
