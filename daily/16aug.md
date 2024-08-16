No code commit today
Reason:
    - Time spend on identifying how to add encode metadata to .wpm file

Timeline:
Identify how encoding in done.

The rustc dev guide does not explicitly mention this anywhere but after reading through `rustc_metadata` 
the metadata is always encoded in rmeta format which is later added on top of other crate files

*So theoretically I just have to prepend this file on top of a `.wasm` file,
Change the extension to `.wpm` and the it should work*

While going through `rustc_metadata::rmeta::encoder.rs`, I spent stupidly long on this [FIXME](https://github.com/mav3ri3k/rust/blob/3139ff09e9d07f7700f8d15ed25a231e29c43627/compiler/rustc_metadata/src/rmeta/encoder.rs#L159)
Not the proudest moment. Turns out, most of it is already done and does not affect me in the slightest.

Now before encoding, I need to work up so that a proc macro can be compiled to wasm.

Tomorrow:
Remove this error when compiling proc macro to wasm:
"the `#[{}]` attribute is only usable with crates of the `proc-macro` crate type"
Error defined `rustc_builtin_macros/src/proc_macro_harness.rs`

FUTURE_REFER([How DECLS works](https://github.com/mav3ri3k/rust/blame/3139ff09e9d07f7700f8d15ed25a231e29c43627/compiler/rustc_builtin_macros/src/proc_macro_harness.rs#L272))
