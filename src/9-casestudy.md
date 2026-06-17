# Chapter 9. Case Study

## 9.1 Asterinas

[Asterinas](https://github.com/asterinas/asterinas) uses an older toolchain (2025-02-01).
To apply RAPx, a few minor modifications are required (see an example [here](https://github.com/Artisan-Lab/asterinas-rapx)). Then, RAPx can be applied using the following command.

```shell
cd ostd
cargo rapx check -f -- --target x86_64-unknown-none
```


