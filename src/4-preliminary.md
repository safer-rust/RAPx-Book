# Chatper 4. Preliminary: Compiler Internals

## IR
Our analysis platform is primarily based on two forms of Rust intermediate representations: HIR and MIR.
 - [HIR](https://rustc-dev-guide.rust-lang.org/hir.html): An AST-like, type-checked, high-level intermediate representation that closely follows Rust’s surface syntax while eliminating syntactic sugar.
 - [MIR](https://rustc-dev-guide.rust-lang.org/mir/index.html): A LLVM-IR-like, mid-level intermediate representation designed to simplify borrow checking, optimization, and program analysis. Note that MIR is not SSA.

You can obtain the HIR or MIR of a Rust source file by executing the following commands:
```shell
cargo rustc -- -Z unpretty=hir-tree
cargo rustc -- -Zunpretty=mir
```

## Key Terms and Usages
- [TyCtxt](https://doc.rust-lang.org/nightly/nightly-rustc/rustc_middle/ty/struct.TyCtxt.html): It is the central data structure of Rust compilers. We can obtain the hir or mir of a function based on the object.
- [DefId](https://doc.rust-lang.org/beta/nightly-rustc/rustc_span/def_id/struct.DefId.html): It is a globally unique identifier that identifies an item (such as a function, type, trait, or impl) across all crates by pairing a crate identifier with a definition index.
- [LocalDefId](https://doc.rust-lang.org/beta/nightly-rustc/rustc_span/def_id/struct.LocalDefId.html): A crate-local identifier that is equivalent to a DefId with `krate == LOCAL_CRATE`. Unlike `DefId`, it does not carry a crate identifier and therefore cannot refer to definitions from external crates.
- [Place](https://doc.rust-lang.org/beta/nightly-rustc/rustc_middle/mir/struct.Place.html): An abstract memory location in MIR. A Place describes where a value is stored or accessed, consisting of a base (such as a local variable or static) and a projection path (e.g., field access, dereference, indexing). 
- [Local](https://doc.rust-lang.org/beta/nightly-rustc/rustc_middle/mir/struct.Place.html): An index identifying a local variable in a MIR body, such as function parameters, temporaries, or user-declared variables.

To obtain the mir of a function.
```rust
let mir = optimized_mir(def_id); 
```

To obtain the body of a function.
```rust
let body = self.tcx.optimized_mir(def_id);
```

```rust
let mut ty = body.local_decls[local].ty;
```
        




