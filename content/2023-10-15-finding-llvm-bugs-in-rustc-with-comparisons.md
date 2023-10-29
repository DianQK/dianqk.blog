+++
title = "Finding LLVM bugs in rust using good-bad comparisons"
description = "Finding problems in large projects is always complicated, and bugs in LLVM mixed with rust are one such case. This article describes how the author finds the rust unit test failure problem under stage2."
+++

# Abstract

Finding problems in large projects is always complicated, and the LLVM bugs mixed in with rust is one such case. In this article, I will describe how I solved the rust unit test failure issue under stage2.
I'll discuss the issues around [Failing tests when rustc is compiled with 1 CGU](https://rust-lang.zulipchat.com/#narrow/stream/131828-t-compiler/topic/Failing.20tests.20when.20rustc.20is.20compiled.20with.201.20CGU) and [Implementing niche checks](https://rust-lang.zulipchat.com/#narrow/stream/131828-t-compiler/topic/Implementing.20niche.20checks) to document my process for solving this type of issue, which I hope will help solve this type of issue in the future.

# Before We Start

- Since this is a summary post of my solution, I may have omitted some details of the process, so please forgive me.
- Feel free to submit [an issue](https://github.com/DianQK/dianqk.blog/issues) or [PR](https://github.com/DianQK/dianqk.blog/pulls) to correct errors in the article.

I have prepared a corresponding project for this article, see:

- [https://github.com/DianQK/rust/tree/blog/repro-1-cgu](https://github.com/DianQK/rust/tree/blog/repro-1-cgu)
- [https://github.com/DianQK/llvm-project/tree/blog/repro-1-cgu](https://github.com/DianQK/llvm-project/tree/blog/repro-1-cgu)
- [https://github.com/DianQK/rust/tree/blog/mir-niche-checks](https://github.com/DianQK/rust/tree/blog/mir-niche-checks)
- [https://github.com/DianQK/llvm-project/tree/blog/mir-niche-checks](https://github.com/DianQK/llvm-project/tree/blog/mir-niche-checks)

The PR to fix these two issues is:

- [https://github.com/llvm/llvm-project/pull/67539](https://github.com/llvm/llvm-project/pull/67539)
- [https://github.com/llvm/llvm-project/pull/68190](https://github.com/llvm/llvm-project/pull/68190)

# How to reproduce the first problem

First, we need to switch to a revision that can be reproduced:

```bash
git clone https://github.com/rust-lang/rust.git
git checkout 085acd02d4abaf2ccaf629134caa83cfe23283c8
```

Then we need to change `config.toml`:

```toml
profile = "dist"

[build]
profiler = true

[rust]
codegen-units = 1
optimize = 2
```

We also need to understand how to use [rustc-perf](https://github.com/rust-lang/rustc-perf) and then build rustc with PGO optimization using the following script:

```bash
./x build --rust-profile-generate=/tmp/profiles --stage 2 library
cargo run --bin collector bench_local --include serde,syn <path to stage2/bin/rustc>
./build/ci-llvm/bin/llvm-profdata merge -o profiles.profdata /tmp/profiles
./x build --rust-profile-use=profiles.profdata --stage 2 library
```

The problem can be reproduced by compiling the following code with this version of rustc: 

```rust
#![feature(inline_const)]

fn main() {
    const {
        assert!(-9.223372036854776e18f64 as i64 == 0x8000000000000000u64 as i64);
    }
}
```

The reproduced error log is as follows:

```bash
error[E0080]: evaluation of `main::{constant#0}` failed
 --> ./test.rs:5:9
  |
5 |         assert!(-9.223372036854776e18f64 as i64 == 0x8000000000000000u64 as i64);
  |         ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^ the evaluated program panicked at 'assertion failed: -9.223372036854776e18f64 as i64 == 0x8000000000000000u64 as i64', ./test.rs:5:9
  |
  = note: this error originates in the macro `assert` (in Nightly builds, run with -Z macro-backtrace for more info)

note: erroneous constant encountered
 --> ./test.rs:4:5
  |
4 | /     const {
5 | |         assert!(-9.223372036854776e18f64 as i64 == 0x8000000000000000u64 as i64);
6 | |     }
  | |_____^

error: aborting due to previous error

For more information about this error, try `rustc --explain E0080`.
```

## Do some preliminary analysis

## Use the error stack to determine a problematic crate

We can use `-Z treat-err-as-bug` to get the error stack, where the incorrectly compiled function is likely to be.

```bash
› ./build/host/stage2/bin/rustc ./test.rs -Z treat-err-as-bug
error[E0080]: evaluation of `main::{constant#0}` failed
 --> ./test.rs:5:9
  |
5 |         assert!(-9.223372036854776e18f64 as i64 == 0x8000000000000000u64 as i64);
  |         ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^ the evaluated program panicked at 'assertion failed: -9.223372036854776e18f64 as i64 == 0x8000000000000000u64 as i64', ./test.rs:5:9
  |
  = note: this error originates in the macro `assert` (in Nightly builds, run with -Z macro-backtrace for more info)

thread 'rustc' panicked at compiler/rustc_errors/src/lib.rs:1724:30:
aborting due to `-Z treat-err-as-bug=1`
stack backtrace:
  ...
  23:     0x7f32189f3bd7 - rustc_const_eval[7551efff2730a760]::const_eval::eval_queries::eval_to_const_value_raw_provider
  ...
```

By examining the stack, I think `rustc_const_eval` is the crate to focus on, and we can do a simple verification to prove my guess. Change `Cargo.toml` to the following:

```toml
[profile.release.package.rustc_const_eval]
codegen-units = 16
```

We can see that the problem is no longer reproducing. I think rustc on stage1 compiles `rustc_const_eval` erroneously.

## Simplify `test.rs`

After some tweaking of the reproduction use case, I found that the following code can also be reproduced:

```rust
#![crate_type = "lib"]

const _: u32 = -1.1f32 as i32 as u32 - 1 as u32;
```

Also, we can see that the issue here behaves as if any negative floating-point number converted to a signed integer will go straight to 0.

## Roughly identify the problem function

By reading the `rustc_const_eval` code and analyzing the call stack, I guess the issue is in [float_to_float_or_int](https://github.com/rust-lang/rust/blob/085acd02d4abaf2ccaf629134caa83cfe23283c8/compiler/rustc_const_eval/src/interpret/cast.rs#L183) and [cast_from_float](https://github.com/rust-lang/rust/blob/085acd02d4abaf2ccaf629134caa83cfe23283c8/compiler/rustc_const_eval/src/interpret/cast.rs#L309) nearby.

To verify this, we can use `#[inline(never)]` to prevent partial optimization. I found that adding `#[inline(never)]` to `float_to_float_or_int` is still an issue while adding to `cast_from_float` makes the test code compile correctly. I guess the issue is in `float_to_float_or_int` and the inline function `cast_from_float`.

## Is it really about PGO?

In fact, we can reproduce the issue directly using the stage2 version of rustc used to generate PGO:

```bash
./x build --rust-profile-generate=/tmp/profiles --stage 2 library
```

# Use git bisect to find which commit caused the problem

While this result may not be a faulty commit, it can give us a concrete way to control the compilation of errors. We can further pinpoint the problem by tuning this commit.

## Pick a good commit

In execute git bisect, we need to find a good commit.
If we can't find a good commit between major releases, I'm going to give up on git bisect because a commit that's too far back in time might not make sense. And the more commits we ignore, the more likely we are to have other unrelated issues.
Here's a simple approach: LLVM's tags have a creation rule where we create a tag that raises the major version at the same time we create a new release branch, which is `llvmorg-{version}-init`, which has a linear relationship and is very bisect-friendly.
I would treat `llvmorg-18-init` and `llvmorg-17.0.1` as consistent code.

Here we have chosen the comparison version:

- Good commit: [b0daacf5](https://github.com/llvm/llvm-project/commit/b0daacf58f417634f7c7c9496589d723592a8f5a), which is the previous commit of `llvmorg-17-init`, and is similar to `llvmorg-16.0.0`. Since rust uses version numbers to adapt to LLVM API changes, we need to keep the version numbers consistent with the API.
- Bad commit: [d0b54bb5](https://github.com/llvm/llvm-project/commit/d0b54bb50e5110a004b41fc06dadf3fee70834b7).

## Preparing the LLVM build configuration

Since each step of the bisect takes a long time, I first recommend using a higher-performance computer to shorten this time.

Then change `config.toml` to reduce the repetitive build time, my changes are as follows:

```toml
[build]
submodules = false

[llvm]
download-ci-llvm = false
assertions = false
ccache = "sccache"
targets = "X86"
experimental-targets = ""

[target.x86_64-unknown-linux-gnu]
# The commit of the patch will be removed after using bisect.
llvm-has-rust-patches = false

[rust]
codegen-units = 256
```

Also, change `Cargo.toml`:

```toml
[profile.release.package.rustc_const_eval]
codegen-units = 1
```

## Reducing functions processed by PGO

This reduces build time and further identifies the problem.

We change [PGOInstrumentation.cpp#L1761](https://github.com/llvm/llvm-project/blob/llvmorg-17.0.2/llvm/lib/Transforms/Instrumentation/PGOInstrumentation.cpp#L1761) filtering rules, for example:

```cpp
static bool skipPGO(const Function &F) {
  // ...
  if (!F.getName().contains("rustc_const_eval"))
    return true;
  if (!F.getName().contains("float_to_float_or_int"))
    return true;
  if (!F.getName().contains("cast_from_float"))
    return true;
  // ...
}
```

## Execute git bisect

The bisect process is a bit different from the standard LLVM project. Instead of using `git bisect skip`, we need to manually adapt API changes when we encounter a `PassWrapper.cpp` compilation failure. Instead of using `git bisect skip`.
Since `PassWrapper.cpp` has to be modified so often, we can't automate the process using `git bisect run`, but have to do it manually and check the results each time. With any luck, it takes less than 12 runs to get the results.

After running it for a while, I got a bisect result of [361464c0](https://github.com/llvm/llvm-project/commit/361464c027239a70d66fb7790032b23696d5b303).
Because the LLVM is so complex, I also generally can't tell what the problem is directly from this commit, and this commit may not necessarily be a faulty commit.

I generally categorize bisect results into several categories:

- It was a standalone commit that caused the mis-compilation to occur.
- This is a standalone commit that caused a mis-compilation to occur. It was a commit with misleading information that caused a later Pass to produce a "mis" compilation (this is the first type of problem).
- it's just a coincidence that a later mis-compiled Pass provided matching input or exposed an existing mis-compilation (this is the second type of problem).

At this point, however, we can't categorize the result of this one. But we can use this result to continue debugging the problem.

# Finding the transformation of interest by changing the LLVM source code

At this point, we don't know where the mis-compilation is, and we can't debug it by getting an IR directly. We still compile and run rustc to locate the problem until we have a clear conclusion.

Based on the bisect result, **we need to find which function ended up mis-compiling after `processImmutArgument`**.

Using code like the following can help us step-by-step to pinpoint which function is affected.

```cpp
bool MemCpyOptPass::processImmutArgument(CallBase &CB, unsigned ArgNo) {
    // ...
    auto FnName = CB.getFunction()->getName();
    if (FnName.contains("rustc_const_eval") &&
        FnName.contains("CompileTimeInterpreter") &&
        FnName.contains("float_to_float_or_int")) {
      errs() << "LLVMLOG: Skip " << FnName << "\n";
      return false;
    // ...
}
```

Looking through the log I can find that the function impacted in `MemCpyOptPass` is:

```
<rustc_const_eval::interpret::eval_context::InterpCx<rustc_const_eval::const_eval::machine::CompileTimeInterpreter>>::float_to_float_or_int
```

From the log I found that this function completes multiple `memcpy` transformations.

So we can continue filtering to find which transformation is causing the problem.

```cpp
auto FnName = CB.getFunction()->getName();
bool IsKeyFunction = FnName.contains("rustc_const_eval") &&
                     FnName.contains("CompileTimeInterpreter") &&
                     FnName.contains("float_to_float_or_int");
if (!IsKeyFunction)
  return false;
static int Count = 0;
Count += 1;
if (Count != 3) {
  errs() << "LLVMLOG: Skip " << Count << "\n";
  return false;
}
errs() << "LLVMLOG: Use " << Count << "\n";
```
When I got to this point, I began to suspect that the mis-compilation had nothing to do with PGO. At this point, I tried canceling the PGO which also reproduced the problem.
My guess is that PGO was just a coincidence that exposed the mis-compilation to the runtime.
But we don't yet know what type of `MemCpyOptPass` it is, and it could be a new coincidence, or a mis-compilation, or a misdirected subsequent Pass. 

## Using `-opt-bisect-limit` 

We can use `-opt-bisect-limit` to locate which pass changes instructions that cause problems at runtime.

There are two types of passes found using `-opt-bisect-limit`:

- The previous pass performed the correct transformation but did not update the metadata and other information in time, resulting in a problem with the found passes.
- There is a mis-compilation in the found pass itself.

## Execute `-opt-bisect-limit` for a specific crate.

Trivia: This real debugging was done by changing `OptBisect.cpp`. However, while writing this article, I found a simpler and more efficient solution.

If we set `-Cllvm-args=-opt-bisect-limit=-1` via `RUSTFLAGS_NOT_BOOTSTRAP` directly, we will get a lot of invalid logs. We want to apply this only to `rustc_const_eval`.

The nightly version of `cargo` provides this. We need to switch to nightly first. make the following changes:

```toml
[build]
cargo = "<path to home>/.rustup/toolchains/nightly-x86_64-unknown-linux-gnu/bin/cargo"
```

Then update `Cargo.toml`.

```toml
cargo-features = ["profile-rustflags"]
# ...
[profile.release.package.rustc_const_eval]
rustflags = [
  "-C", "llvm-args=-opt-bisect-limit=-1",
]
```

Next, we can build using `--keep-stage`:

```bash
./x build --stage 2 library --keep-stage 0 --keep-stage-std 1 2> build.log
./build/host/stage2/bin/rustc ./test.rs
```

My final bisect result was:

```
BISECT: running pass (560953) MemCpyOptPass on _RNvMNtNtCsiODAygBxQYA_16rustc_const_eval9interpret4castINtNtB4_12eval_context8InterpCxNtNtNtB6_10const_eval7machine22CompileTimeInterpreterE21float_to_float_or_intB6_
BISECT: running pass (560954) DSEPass on _RNvMNtNtCsiODAygBxQYA_16rustc_const_eval9interpret4castINtNtB4_12eval_context8InterpCxNtNtNtB6_10const_eval7machine22CompileTimeInterpreterE21float_to_float_or_intB6_
BISECT: NOT running pass (560955) MoveAutoInitPass on _RNvMNtNtCsiODAygBxQYA_16rustc_const_eval9interpret4castINtNtB4_12eval_context8InterpCxNtNtNtB6_10const_eval7machine22CompileTimeInterpreterE21float_to_float_or_intB6_
```

It's weird to me. If I set the limit to `560953`, the result does not stop at `MemCpyOptPass`. The result is a bit untrustworthy.

```
BISECT: running pass (560953) BDCEPass on _RNvXsg_NtNtCsjGTw0T6X7N4_16rustc_const_eval9interpret7operandNtB5_9ImmediateNtNtCs6sCMhXnFQZh_4core3fmt5Debug3fmtB9_
BISECT: NOT running pass (560954) InstCombinePass on _RNvXsg_NtNtCsjGTw0T6X7N4_16rustc_const_eval9interpret7operandNtB5_9ImmediateNtNtCs6sCMhXnFQZh_4core3fmt5Debug3fmtB9_
```

However, we can verify that the problem is related to `DSEPass` in a similar way, by changing the code to skip `DSEPass`.
Next, as with debugging `MemCpyOptPass`, we can find out exactly which instruction `DSEPass` transformation is causing the final mis-compilation.

Update: For more stable results with `-opt-bisect-limit`, we can try `-Z no-parallel-llvm`. Also, `rustc` tries to perform ThinLTO by default, which can be turned off with `-C lto=false`.

# Getting IR to prepare for LLVM debugging

This part is related to LLVM debugging, and I don't have any good experience to share yet.

But the two important points are:
- We know exactly which transformation affects the result in `MemCpyOptPass`.
- We also know which transformation in `DSEPass` caused the final mis-compilation.

With these two transformations, we can use `opt` to debug IR. We don't need to use `rustc` to compile as often!

The recommended way to get the IR is by changing `Cargo.toml`:

```toml
[profile.release.package.rustc_const_eval]
codegen-units = 1
rustflags = [
  "-C", "save-temps",
]
```

Then find `*.no-opt.bc` to debug.

Of course, during this debugging process, we need to know the details of the logic of the two transformations of `MemCpyOptPass` and `DSEPass`. Here we'll learn that this is a transformation via alias analysis, which ultimately pinpoints the fact that `MemCpyOptPass` doesn't update the corresponding alias information when replacing the value used by the instruction.


# Next question - mir niche checks

Although this problem is easier to reproduce than the previous one, it will be more troublesome to solve.
First, let's switch to [cf8d85e4](https://github.com/DianQK/rust/commit/cf8d85e49c6f3a5802549055f983fd0ba3f89952).

This can be reproduced by running a test with `--stage 2`:

```bash
./x test tests/ui --stage 2
```

`config.toml` reference:

```toml
profile = "codegen"

[llvm]
assertions = false
enable-warnings = false
download-ci-llvm = false
ccache = "sccache"
targets = "X86"
experimental-targets = ""
link-shared = true
use-linker = "lld"
optimize = true
release-debuginfo = true

[rust]
debug = false
incremental = false
optimize = 3
debug-logging = false
deny-warnings = false
codegen-backends = ["llvm"]
use-lld = true
lto = "off"
debug-assertions = true
debug-assertions-std = false
```

The error log is as follows:

```
thread 'rustc' panicked at compiler/rustc_mir_build/src/thir/pattern/deconstruct_pat.rs:560:22:
occupied niche: found 0x7f7700000000 but must be in 0x0..=0x2 in type std::option::Option<thir::pattern::deconstruct_pat::SliceKind> at offset 0 with type Int(I64, false)
stack backtrace:
  ...
  13:     0x7f77e92ade54 - <rustc_mir_build[48ebee3fe6c2e2b3]::thir::pattern::deconstruct_pat::Constructor>::split::<core[8d828210e7f791ba]::iter::adapters::map::Map<core[8d828210e7f791ba]::iter::adapters::map::Map<core[8d828210e7f791ba]::slice::iter::Iter<rustc_mir_build[48ebee3fe6c2e2b3]::thir::pattern::usefulness::PatStack>, <rustc_mir_build[48ebee3fe6c2e2b3]::thir::pattern::usefulness::Matrix>::heads::{closure#0}>, <rustc_mir_build[48ebee3fe6c2e2b3]::thir::pattern::deconstruct_pat::DeconstructedPat>::ctor>>
  ...
```

I found that if I set `codegen-units=1`, the panic disappears. So we can use `codegen-units` to determine which crate is impacted. I guessed `rustc_mir_build` based on the stack and tried toggling `codegen-units=1` to verify. Since this issue occurred at roughly the same point in time as the previous one, we used the same LLVM commit at the start of the bisect. Unfortunately, with the `optimize=3` set, we can't find a good commit in LLVM 16. But I also tried setting `optimize=2` and can find LLVM 16 is a good commit.


## Tricky git bisect

During the bisect process, I get an unexpected result:

```
---- [ui] tests/ui/issue-11881.rs stdout ----

error: test compilation failed although it shouldn't!
status: exit status: 1
command: RUSTC_ICE="0" "/home/dianqk/rust-workspace/rust/build/x86_64-unknown-linux-gnu/stage2/bin/rustc" "/home/dianqk/rust-workspace/rust/tests/ui/issue-11881.rs" "-Zthreads=1" "-Zsimulate-remapped-rust-src-base=/rustc/FAKE_PREFIX" "-Ztranslate-remapped-path-to-local-path=no" "-Z" "ignore-directory-in-diagnostics-source-blocks=/home/dianqk/.cargo" "--sysroot" "/home/dianqk/rust-workspace/rust/build/x86_64-unknown-linux-gnu/stage2" "--target=x86_64-unknown-linux-gnu" "-O" "--error-format" "json" "--json" "future-incompat" "-Ccodegen-units=1" "-Zui-testing" "-Zdeduplicate-diagnostics=no" "-Zwrite-long-types-to-disk=no" "-Cstrip=debuginfo" "-C" "prefer-dynamic" "-o" "/home/dianqk/rust-workspace/rust/build/x86_64-unknown-linux-gnu/test/ui/issue-11881/a" "-A" "internal_features" "-Crpath" "-Cdebuginfo=0" "-Lnative=/home/dianqk/rust-workspace/rust/build/x86_64-unknown-linux-gnu/native/rust-test-helpers" "-Clink-arg=-fuse-ld=lld" "-Clink-arg=-Wl,--threads=1" "-L" "/home/dianqk/rust-workspace/rust/build/x86_64-unknown-linux-gnu/test/ui/issue-11881/auxiliary"
stdout: none
--- stderr -------------------------------
error: unexpected token: `&`
  --> /home/dianqk/rust-workspace/rust/tests/ui/issue-11881.rs:18:25
   |
LL |     fn encode(&self, s: &mut S) -> Result<(), S::Error>;
   |                         ^ unexpected token after this

error: unexpected token: `&`
  --> /home/dianqk/rust-workspace/rust/tests/ui/issue-11881.rs:33:23
   |
LL |     fn fmt(&self, _f: &mut fmt::Formatter<'_>) -> fmt::Result {
   |                       ^ unexpected token after this
```

For this result, we can't use good/bad to execute bisect, which leads bisect to the wrong result.

Unfortunately, even with a skip, we can't get a bisect result.
When bisect is in [f7deb69f2...7c78cb4b](https://github.com/llvm/llvm-project/compare/f7deb69f22b93d7411d08db14b50aae5caf40fcb...7c78cb4b1f4993a84bf2b46b197d90dcabb9f8c5) internally, it is possible to get this unexpected error. If it's an earlier commit, it's good, and if it's a later commit, it's bad.
This is because new issues were introduced with the change in the semantics of `nonnull` and so on, and we overlooked some issue fixes during bisect that led to the exposure of a new issue.

The history of this commit is as follows:

```
bad
Revert "[SimplifyCFG][LICM] Preserve nonnull, range and align metadat…  7c78cb4
skip
[SimplifyCFG][LICM] Preserve nonnull, range and align metadata when s…  78b1fbc
good
```
Since we ignored some commits that caused a new problem, bisect had no results.
Luckily, this time the problem was exceptional and we were able to drop `78b1fbc` and `7c78cb4` using rebase.

The final bisect result is [[AggressiveInstCombine] Enable also for -O2](https://github.com/DianQK/llvm-project/commit/2f70f65246e987842cfae495839975eaa6245a0c).
I also found the critical transformation in `AggressiveInstCombine` by changing the code, but I can't see anything wrong with it from the code or IR. It could be that I'm missing something, or it could be that this is just a lucky opportunity for a triggering.
We need to keep this point of suspicion in mind and continue troubleshooting.

# `codegen-units=256` & `-opt-bisect-limit=n`

This time we have no way to use `-opt-bisect-limit` because `codegen-units` is not equal to 1. 
It doesn't make sense to bisect multiple IRs at the same time. We need to modify the rust code to support selecting a specific CGU to bisect under multiple CGUs.

First use `-C save-temps` to find the corresponding IR. See [bfd759b7](https://github.com/DianQK/rust/commit/bfd759b704b77d0ff390cc172a62c3d9a0c36986) for the details.

Trivia: Here I also tried to find the one transformation in AggressiveInstCombine that had an effect. It was a headache that adding `-C save-temps` changed the symbol names, which made me look again for the associated symbols.

I wrote a simple script to find this IR:

```bash
for bitcode in build/x86_64-unknown-linux-gnu/stage1-rustc/x86_64-unknown-linux-gnu/release/deps/rustc_mir_build-*-cgu.*.rcgu.no-opt.bc; do
    if llvm-nm -U $bitcode | grep -q "example"; then
        echo $bitcode
    fi
done
```

I viewed the implementation of `OptBisect`, and although the `-opt-bisect-limit` option is global, we can replace an empty `OptBisect` with a separate one for the CGU. The full modification is at [a9f62a4a](https://github.com/DianQK/rust/commit/a9f62a4a634aef8f792e9cb6eb4573b4b9fc79d2).

Some key changes are referenced below:

```cpp
struct RunAllOptPassGate : public OptPassGate {
  bool shouldRunPass(const StringRef PassName, StringRef IRDescription) override {
    return true;
  }
  bool isEnabled() const override { return true; }
};

static RunAllOptPassGate &getRunAllOptPassGate() {
  static RunAllOptPassGate RunAllOptPassGate;
  return RunAllOptPassGate;
}

extern "C" void LLVMRustContextSetSetRunAllOptPassGate(LLVMContextRef C) {
  unwrap(C)->setOptPassGate(getRunAllOptPassGate());
}
```

I also added a command line argument `-Z llvm-opt-bisect-limit-cgu` so that I could bisect using the following script:

```bash
export RUSTFLAGS_NOT_BOOTSTRAP="-C llvm-args=-opt-bisect-limit=-1 -Z llvm-opt-bisect-limit-cgu=rustc_mir_build.63d28fcded2a05ed-cgu.007"
./x build --stage 2 library --keep-stage 0 --keep-stage-std 1 2> build.log
./build/host/stage2/bin/rustc ./tests/ui/consts/const_prop_slice_pat_ice.rs
```

I also wrote a simple automated bisect script:

```bash
function iterate() {
    local good=`sed -n '1p' bisect_result`
    local bad=`sed -n '2p' bisect_result`
    local result=$((bad - good))
    echo "good: $good, bad: $bad"
    if [ $result -eq 1 ]; then
        echo "done"
        exit 0
    else
        local next=$((good + (result / 2)))
        echo $next
        bash bisect.sh $next
        exit_code=$?
        case $exit_code in
            0)
                good=$next
                ;;
            1)
                bad=$next
                ;;
            *)
                echo "failed: $exit_code"
                exit 1
                ;;
        esac
    fi
    echo $good > bisect_result
    echo $bad >> bisect_result
}

while true; do
    iterate
    sleep 1
done
```
But I got a weird result:

```
ISECT: running pass (13444) InlinerPass on (symbol)
BISECT: NOT running pass (13445) PostOrderFunctionAttrsPass on (symbol)
```

I don't think inline is directly related to this mis-compilation.
I simply changed `OptBisect.cpp` to skip `InlinerPass`:

```cpp
bool OptBisect::shouldRunPass(const StringRef PassName,
                              StringRef IRDescription) {
  if (PassName == "InlinerPass") {
      printPassMessage(PassName, -1, IRDescription, true);
      return true;
  }
  // ...
}
```

Eventually, I got:

```
BISECT: running pass (10040) CorrelatedValuePropagationPass on symbol
BISECT: NOT running pass (10041) SimplifyCFGPass on symbol
```

To verify that `CorrelatedValuePropagationPass` is related, it is still done by removing unrelated code, see [a08f2c14](https://github.com/DianQK/llvm-project/commit/a08f2c14b1d501aa9b76b96e26c7366c5a6e6e9b).
I also added a line of logging for simple validation:

```
LLVMLOG: Delete   %102 = and i64 %101, 4294967295   -> and i64 %101, 0xffffffff
...
occupied niche: found 0x7fba00000001 but must be in 0x0..=0x2 in type std::option::Option<thir::pattern::deconstruct_pat::SliceKind> at offset 0 with type Int(I64, false)
0x7fba00000001 & 0xffffffff = 0x1
```

I'm pretty sure this is what we're looking for! This correlates very well with rustc's panic log.
Eventually, I realized that `%101` could get `undef` results under certain control flows. In this case, we should not remove `%102`.

Here I am curious why it is related to `AggressiveInstCombine`, if we remove this Pass, the instruction to delete becomes `%123 = and i64 %122, 72057594037927935(0xffffffffffffff)`. We still can't delete the instruction, it's just that this huge value makes it hard for the program to encounter during runtime.

# Summary

What have we compared?

- Finding the affected crate by comparing different `codegen-units`.
- Compare results by adding `[inline(never)]` to find the affected functions.
- git bisect to find out which commit affected the result.
- by changing the code to find out which transformations were affected.
- Finding related passes through `-opt-bisect-limit` results.

I'm getting closer to the truth with these approaches. By the way, luck is also important, I don't keep records of my missteps :].

In these methods, I have also introduced some specific methods:

- git bisect can find results by removing some commits when it doesn't (this may be a case-specific trick)
- cargo provides the ability to apply rustc parameters to a given crate
- Changing the rustc code to apply `opt-bisect-limit` to a specific CGU should make this useful for large projects.

The generalized process for solving this type of problem is:

1. first find a way to shorten the time to single replication, too long debugging time is annoying
2. the key objective is to **find the key transformations that are causing the mis-compilation by comparing the above methods**.
3. extract the IR and locate the problem based on the crucial transformations
4. fix the problem

I don't have any experience with step 3 yet, but I would use `llvm-extract` and `llvm-reduce` to reduce the amount of IRs I get, which would be helpful for debugging. 
I would also use `-opt-bisect-limit` to extract the IR for intermediate processes and manually remove some functions or instructions to debug the problem.

I don't have a clear idea of how to submit a proper fix yet, and currently, I rarely manage to get PRs for LGTM without modifications. l need to learn and practice more ;).

# Refer

- [https://rustc-dev-guide.rust-lang.org/compiler-debugging.html](https://rustc-dev-guide.rust-lang.org/compiler-debugging.html)
- [https://rustc-dev-guide.rust-lang.org/backend/debugging.html](https://rustc-dev-guide.rust-lang.org/backend/debugging.html)
- [https://llvm.org/docs/GitBisecting.html](https://llvm.org/docs/GitBisecting.html)
- [https://doc.rust-lang.org/cargo/reference/unstable.html#profile-rustflags-option](https://doc.rust-lang.org/cargo/reference/unstable.html#profile-rustflags-option)
- [https://www.llvm.org/docs/AliasAnalysis.html](https://www.llvm.org/docs/AliasAnalysis.html)
- [https://doc.rust-lang.org/rustc/codegen-options/index.html#lto](https://doc.rust-lang.org/rustc/codegen-options/index.html#lto)
