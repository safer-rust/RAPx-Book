# Chapter 3. Framework of RAPx
Traditionally, performing code analysis requires modifying the compiler source code to add new passes. 
Developers then need to recompile the compiler to activate these new passes, which can be cumbersome. 
The Rust compiler offers a more portable way to conduct code analysis using the [rustc_driver](https://rustc-dev-guide.rust-lang.org/rustc-driver.html).
We refer to this approach as the **frontend** method because it allows developers to directly access internal compiler data and perform analysis as callbacks.

## Custom Cargo Commands
To support project-level program analysis, we want the analysis tool to be integrated into cargo as [subcommands](https://doc.rust-lang.org/cargo/reference/external-tools.html). To this end, we can name the tool as cargo-toolname and place it in $CARGO_HOME/bin or $PATH. Then we can execute the tool via the following command.
```shell
cargo toolname -more_arguments
```
Cargo will automatically search the binaries named cargo-toolname from the paths. The following figure demonstrates the whole process before reaching our analysis program.
![Workflow of how cargo dispatches the analysis command to rapx.](figure/cargoflow.png)
Note that we cannot directly invoke rapx in the first round but through cargo check because we need cargo to manage the project-level compilation and append detailed compilation options for launching rustc. However, we want to hook rustc execution and execute rapx instead for analysis. Therefore, we set [RUSTC_WRAPPER](https://doc.rust-lang.org/cargo/reference/environment-variables.html) with the value of cargo-rapx. In this way, cargo check will actually run `cargo-rapx rustc appended_options`. We then dispath the execution to rapx with appended options.

## Register Analysis Callbacks
Supposing the purpose is to execute a function named my_analysis, developers should design a new struct and implement the [Callbacks Trait](https://doc.rust-lang.org/nightly/nightly-rustc/rustc_driver/trait.Callbacks.html) for the struct.
```rust
pub struct MyCallback {...}
impl Callbacks for MyCallback {
    fn after_analysis<'tcx>(&mut self, compiler: &Compiler, queries: &'tcx Queries<'tcx>) -> Compilation {
        compiler.session().abort_if_errors();
        queries.global_ctxt().unwrap().enter(
            |tcx| my_analysis(tcx, *self) // the analysis function to execute after compilation.
        );
        compiler.session().abort_if_errors();
        Compilation::Continue
    }
}
```

To execute the compiler and callback function, developers can employ the APIs [rustc_driver::RunCompiler](https://doc.rust-lang.org/nightly/nightly-rustc/rustc_driver/struct.RunCompiler.html) provided by Rust.
```rust

let mut callback = MyCallback::default();
let run_compiler = rustc_driver::RunCompiler::new(&args, callback);
let exit_code = rustc_driver::catch_with_exit_code(move || run_compiler.run());
```


