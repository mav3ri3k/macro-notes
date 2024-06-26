[29 May]

- Testing wasmtime api -> This runs in same process as the compiler ( it would likely have to be separate process )
    - Examples make sense 
- Built a simple proc macro at [commit](https://github.com/mav3ri3k/proc-macro-server/commit/3c7ade5490473d8913a7db32bba22644f0e38b21)
    - Learnt about use of `extern crate proc_macro`. See [pr](https://github.com/rust-lang/reference/pull/1504)
- [!Ques] Ambiguity over how proc macros are linked
    - They have their own `--crate-type=proc-macro, #![crate_type = "proc-macro"]`, but seem to produce `dylib`
        Having own separate type may be a curse or a boon :pray:
    - This is important as when building for wasm, docs mention building for `cdylib` or `bin`
    - Currently naively building proc macro to `--target wasm32-unknown-unknown` builds without error but does not produce output

[30 May]

- [Zulip Message](https://rust-lang.zulipchat.com/#narrow/stream/131828-t-compiler/topic/wasm.20proc.20macros/near/291100839) provides more context for previous ques last day.
    > The main bits I remember not being sure about from when I last looked into this were mostly how to get rustc to build a proper .wasm file for a proc-macro crate. IIRC from some conversations with @eddyb there's currently some hard-coded flags to prevent rustc from building anything other than a cstaticlib for proc-macro, which needs to be changed, and then some work to the loader could be done to force-loading proc-macro crates with the given target.
Find out about these flags.

# Going through proc_macro library
I spend the past couple of days doing through the proc_macro code, trying to make sense of it.

Observations:
- Client does the bulk of the work. The server is quite thin (relatively) in its responsibilities.
- `bridge/{client.rs, server.rs, rpc.rs, bridge.rs` are clear in their roles. The code is personally a little complex. As its turns out, it was mostly because of the work done around using threads. This is new to me. Removing that, the control flow is quite "direct" (famous last words).
- There is entire section of code for `bridge/symbols.rs`, `bridge/arena.rs` and `fxhash.rs` which doesn't quite make sense in what their roles are. These also seem to be least updated code over the years. **I would appreciate some context.**

# Wasm related stuff
## Wasmtime
- I tried basic examples with wasmtime api
- Wasmtime provides safe methods for interacting with the wasm linear memory. However it does not come with a gc. So that would have to be managed manually, eventually. 
- Initial plan is to utilize wasmtime api without any gc or custom ipc.
## Compiling to wasm
- Naively compiling a simple proc_macro crate to `wasm32-unknown-unknown` does not work correctly. This was expected otherwise what will I do this summer ? :laugh: It builds without error but does not emit wasm bin. Interesting. I **believe** currently the compiler only build wasm for `staticlib`, thus above case does not work. Need to look into this.
- Compiling `hello-word.rs` to wasm and then turning it into a c code using `wasm2c` generated 12k loc which is just WoW :pinch:.

# Next immediate goals
- Gain clarity over building wasm bin for proc macro crates
- Build program which allows communicating with wasm bin by exposing host functions.

# Question
I would like to know ideas/thoughts around general implementation details as in how people expect wasm proc_macro to behave or be implemented.

[31 May]

Thoughts on how the implementation should look like
[bjorn3](https://rust-lang.zulipchat.com/#narrow/stream/421156-gsoc/topic/Project.3A.20Sandboxed.20and.20Deterministic.20Proc.20Macro.20using.20Wasm/near/441542623)
>I did personally expect the wasm proc macros to use an entirely new target with it's own abi. This way for example printing to stdout/stderr and reading env vars could be supported. wasm32-unknown-unknown can't support either as it has to assume that there is no environment to implement this functionality.

[bjorn3](https://rust-lang.zulipchat.com/#narrow/stream/421156-gsoc/topic/Project.3A.20Sandboxed.20and.20Deterministic.20Proc.20Macro.20using.20Wasm/near/441543634)
>Wasmtime is working on implementing the GC proposal. I don't think it is necessary however. Maybe using the component model would be the easiest way of implementing the ABI between the proc macro and rustc? Or alternatively we could reuse the exact same methods that we have for native proc macros and just export an additional memory allocation method. Freeing it technically not even necessary. You could throw away the entire wasm instance and restart from scratch every time. Wasmtime is designed for things like recreating the wasm instance on every http request in a web server.

After discussion on zulip, implementation design:
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
    1. - `wasm32-unknown-unknown` has no abi at all, so anything that interfaces with the OS will return an error or panic. A custom target would allow custom abi for this functionality. **However, initial plan is to support only pure macros which means no custom abi is required. Thus step 1 is skipped as of now.** 
       - It also seem to have some errors: [Use correct ABI for wasm32 by default #79998](https://github.com/rust-lang/rust/pull/79998) 
       - [Is this intended C ABI for wasm ?](https://github.com/WebAssembly/tool-conventions/blob/main/BasicCABI.md)
    2. When custom target is used, **refer** to: 
        - [Tracking issue for the unstable "wasm" ABI #83788](https://github.com/rust-lang/rust/issues/83788). 
        **bjorn3** also mentioned something regarding this: [link to message](https://rust-lang.zulipchat.com/#narrow/stream/421156-gsoc/topic/Project.3A.20Sandboxed.20and.20Deterministic.20Proc.20Macro.20using.20Wasm/near/441685050). Quote:
        > Also a custom target would allow setting the abi for `extern "C"` to match the official C abi for wasm, allowing compatibility with C wasm libraries. wasm32-unknown-unknown currently uses a different abi, though it is going to switch to the official C abi in the future.
        - [Target Tier Policy](https://rust-lang.github.io/rfcs/2803-target-tier-policy.html?highlight=sandbox#)

        - Potential Garbage:
            - [Hot Takes](https://users.rust-lang.org/t/fixing-rusts-webassembly-targets/88947)
            - [WASM-PACK FROM SCRATCH](https://khangdinh.wordpress.com/2022/05/06/wasm-pack-from-scratch/)

## Step 2
What does this even mean ??
- [Original RFC for proc_macro in 2016](https://rust-lang.github.io/rfcs/1681-macros-1.1.html?highlight=sandbox#)
    Why is it called `rustc_macro`. There exist `rustc_macro` and `rustc_expand`. I have currently only checked `rustc_expeand` which conveniently has a file named `proc_macro_server.rs`. :cry: 
    It should be `rustc_expand` [as mentioned in the compiler](https://arc.net/l/quote/ybomdhve). 
    [Link to full guide page if prev quote link breaks](https://rustc-dev-guide.rust-lang.org/macro-expansion.html?highlight=macro#macro-expansion)
    - [Tracking issue for RFC 1566: Procedural macros #38356](https://github.com/rust-lang/rust/issues/38356): Leaving it here so I don't forget.

## Step 3
Focus on this step at the moment. **Targets:**
- [x] Identify functions exported by the server
- [ ] Build mock setup. [Nice little blog I found](https://petermalmgren.com/serverside-wasm-data/) 
    - [x] A lib crate accepts an integer and returns an integer.
    - [ ] Identify what `[#proc_macro]` does ? May be helpful in identifying **Step 2**
    - [x] Export host functions through wasmtime api for sending integer between sever and client.

[ 1 June ]
# Mostly learning day
- Learned Git! :celebration:. Was not bad since I have been using sappling-scm. But god does the ui in sp look good. I don't like git branches.
- Checked out progress on [wasm-c-api](https://doc.rust-lang.org/beta/unstable-book/compiler-flags/wasm-c-abi.html). I don't think it would pose a problem as of now. It mostly affects not being able to inter-op between different wasi specs.
- Found funny [video](https://www.youtube.com/watch?v=OZnPY6xptjg&t=506s&pp=ygUMZGF2aWQgdG9sbmF5). The duo is funny as hell :lol:.
- Did some experiments with `where` keyword. New toy I found reading `proc_macro` code.
- Read [comment by eddyb](https://github.com/rust-lang/rust/issues/54722#issuecomment-488386448). The discussion is not very relevant but the comment clarifies overarching design. Very helpful in understand the code.
- First commit [introducing proc_macro bridge](https://github.com/mav3ri3k/rust/commit/e305994beb1347e2fcadf5c84acec60fb6902551). A lot of changes to various other parts of the compiler. :face_with_monocle: 

[ 2 June ] 
# Big W
Compiler side, the expansion function lives at `/rust/compiler/rustc_expand/src/proc_macro.rs`.
I mean I knew this but this time reading it makes and sense. I can actually **read the code** now.
Yipee!

# Small little hiccup
There is a `TokenStream` definition at `compiler/rustc_ast` which has different impl 
And one at `library/proc_macro::TokenStream` to be consumed by the proc_macro. 
Now it based on `bridge::client::TokenStream` but I can not seem to find where `TokenStream` is defined in `client.rs`.
:salute: Good luck finding that!

# Tit-bit
context: cx, ctx ( this might be trivial but I have never quite seen it )

# Another W to end the day
My goal, for every invocation of a bang_proc_macro, it emits same expansion
- My lsp stopped because it could not parse some manifest in some rustbook folder,
    I don't know why it is even need when I have set my config for compiler
    So finally went around reading the Cargo.toml file, made some changes. Now it works. ( deleted everything related to rustdoc ).
- [This is the entry point for bang_proc_macro](https://github.com/mav3ri3k/rust/blob/20be84a7e62af0a623af4bfb75df6d30cb39d6d0/compiler/rustc_expand/src/proc_macro.rs#L48)
    It required changes to the `run` function for `self.client.run(&strategy, server, input, proc_macro_backtrace)`
    Now if you will do around finding definition for this function. It does not live in `/proc_macro/bridge/client.rs` which you would expect.
    It lives in `/proc_macro/bridge/server.rs` which is interesting.
    Then the `run` function calls `run_server()` which does main work.
    What I did was not a proper changes but at [the last implicit return](https://github.com/mav3ri3k/rust/blob/20be84a7e62af0a623af4bfb75df6d30cb39d6d0/library/proc_macro/src/bridge/server.rs#L388): `Result::decode(&mut &buf[..], &mut dispatcher.handle_store)`. I changed it to always return same `a + b`.
    The compiler was not particularly happy with this. It does not expand correctly and throws some soft errors but it works with `cargo expand` ( little cargo helper command by dtolnay ) and that all I care about. 
