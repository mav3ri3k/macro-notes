[ 2 June ]

Compiler side, the expansion function lives at `/rust/compiler/rustc_expand/src/proc_macro.rs`.
I mean I knew this but this time reading it makes and sense. I can actually read the code now.
Yipee!

There is a `TokenStream` definition at `compiler/rustc_ast` which has different impl 
And one at `library/proc_macro::TokenStream` to be consumed by the proc_macro. Now it based on `bridge::client::TokenStream` but I can not seem to find where `TokenStream` is defined in `client.rs`.

:salute: Good luck finding that!

context: cx, ctx ( this might be trivial but I never quite seen it
