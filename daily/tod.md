# [Main RFC issue:#475](https://github.com/rust-lang/compiler-team/issues/475)
Add support for sandboxed build scripts and proc macros. Wasm Proc Macros are intended to be MVP for the proposal:
1. Allow opt-in full sandboxing of proc macros via wasm
2. Extend the system with capabilities requested via Cargo.toml for access to different system resources

# Current design goals:
1. A new Tier 2 target platform is introduced, `wasm32-macro`. Almost everything about it is similar to `wasm32-unknown-unknown`. 
2. A build of proc_macro library is available for this target.
3. A proc-macro crate for `wasm32-macro` is spawned as a sub-process. This is loaded with some host functions to act as proc macro bridge (likely similar to current proc-macro-bridge implementation).

## Step 1
- What are problem with `wasm32-unknown-unknown` ?
    1. `wasm32-unknown-unknown` has no abi at all, so anything that interfaces with the OS will return an error or panic. A custom target would allow custom abi for this functionality. **However, initial plan is to support only pure macros which means no custom abi is required. Thus step 1 is skipped as of now.** 
    2. When custom target is used, **refer** to: [Tracking issue for the unstable "wasm" ABI #83788](https://github.com/rust-lang/rust/issues/83788). **bjorn3** also mentioned something regarding this: [link to message](https://rust-lang.zulipchat.com/#narrow/stream/421156-gsoc/topic/Project.3A.20Sandboxed.20and.20Deterministic.20Proc.20Macro.20using.20Wasm/near/441685050). Quote:
    > Also a custom target would allow setting the abi for `extern "C"` to match the official C abi for wasm, allowing compatibility with C wasm libraries. wasm32-unknown-unknown currently uses a different abi, though it is going to switch to the official C abi in the future.

## Step 2
What does this even mean ??

## Step 3
Focus on this step at the moment. **Targets:**
- [ ] Identify functions exported by the server
- [ ] Build mock setup. 
    - [ ] A lib crate accepts an integer and returns an integer.
    - [ ] Identify what `[#proc_macro]` does ? May be helpful in identifying **Step 2**
    - [ ] Export host functions through wasmtime api for sending integer between sever and client.

