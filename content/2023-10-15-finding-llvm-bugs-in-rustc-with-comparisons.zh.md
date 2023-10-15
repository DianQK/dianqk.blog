+++
title = "使用好坏比较发现 rust 中的 LLVM bug"
description = "在大型工程中定位问题总是很复杂的，掺杂在 rust 中的 LLVM 的 bug 就是这类情况。本文介绍笔者如何定位 stage2 下 rust 单元测试失败问题。"
+++

# 摘要

在大型工程中定位问题总是很复杂的，掺杂在 rust 中的 LLVM 的 bug 就是这类情况。我将在本文介绍我如何定位 stage2 下 rust 单元测试失败问题。
我将围绕 [Failing tests when rustc is compiled with 1 CGU](https://rust-lang.zulipchat.com/#narrow/stream/131828-t-compiler/topic/Failing.20tests.20when.20rustc.20is.20compiled.20with.201.20CGU) 和 [Implementing niche checks](https://rust-lang.zulipchat.com/#narrow/stream/131828-t-compiler/topic/Implementing.20niche.20checks) 记录我解决这类问题的过程，希望能对后续解决该类问题有所帮助。

# 开始之前

- 由于这是我解决后的总结文章，我可能会疏漏一些详细过程，还请见谅。
- 欢迎提交 [issue](https://github.com/DianQK/dianqk.blog/issues) 或 [PR](https://github.com/DianQK/dianqk.blog/pulls) 改正文章的错误

我为本文准备了对应的工程，参见：

- [https://github.com/DianQK/rust/tree/blog/repro-1-cgu](https://github.com/DianQK/rust/tree/blog/repro-1-cgu)
- [https://github.com/DianQK/llvm-project/tree/blog/repro-1-cgu](https://github.com/DianQK/llvm-project/tree/blog/repro-1-cgu)
- [https://github.com/DianQK/rust/tree/blog/mir-niche-checks](https://github.com/DianQK/rust/tree/blog/mir-niche-checks)
- [https://github.com/DianQK/llvm-project/tree/blog/mir-niche-checks](https://github.com/DianQK/llvm-project/tree/blog/mir-niche-checks)

修复这两个问题的 PR 是：

- [https://github.com/llvm/llvm-project/pull/67539](https://github.com/llvm/llvm-project/pull/67539)
- [https://github.com/llvm/llvm-project/pull/68190](https://github.com/llvm/llvm-project/pull/68190)

# 第 1 个问题的复现方式

首先我们需要切换到可以复现的版本：

```bash
git clone https://github.com/rust-lang/rust.git
git checkout 085acd02d4abaf2ccaf629134caa83cfe23283c8
```

然后需要修改 `config.toml`：

```toml
profile = "dist"

[build]
profiler = true

[rust]
codegen-units = 1
optimize = 2
```

我们还需要了解 [rustc-perf](https://github.com/rust-lang/rustc-perf) 如何使用，然后使用如下脚本构建带有 PGO 优化的 rustc：

```bash
./x build --rust-profile-generate=/tmp/profiles --stage 2 library
cargo run --bin collector bench_local --include serde,syn <path to stage2/bin/rustc>
./build/ci-llvm/bin/llvm-profdata merge -o profiles.profdata /tmp/profiles
./x build --rust-profile-use=profiles.profdata --stage 2 library
```

接下来使用该版本 rustc 编译如下代码即可复现该问题：

```rust
#![feature(inline_const)]

fn main() {
    const {
        assert!(-9.223372036854776e18f64 as i64 == 0x8000000000000000u64 as i64);
    }
}
```

复现的错误日志如下：

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

# 做一些初步的分析

## 使用错误堆栈判断存在问题的 crate

我们可以使用 `-Z treat-err-as-bug` 获取报错堆栈，被错误编译的函数很可能在这里。

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

通过观察堆栈，我认为 `rustc_const_eval` 是值得关注的 crate。我们可以做个简单的验证来证明我猜测。修改 `Cargo.toml` 为如下内容：

```toml
[profile.release.package.rustc_const_eval]
codegen-units = 16
```

我们可以发现问题不再复现。我认为是 stage1 的 rustc 错误编译了 `rustc_const_eval`。

## 简化 `test.rs`

经过一些对复现用例的调整，我发现如下代码也可以复现：

```rust
#![crate_type = "lib"]

const _: u32 = -1.1f32 as i32 as u32 - 1 as u32;
```

同时我们可以发现，这里的问题表现是任意一个负浮点数转换为有符号整数都会直接变成 0。

## 大致定位问题函数

通过阅读 `rustc_const_eval` 代码和分析调用堆栈，我猜测问题出在 [float_to_float_or_int](https://github.com/rust-lang/rust/blob/085acd02d4abaf2ccaf629134caa83cfe23283c8/compiler/rustc_const_eval/src/interpret/cast.rs#L183) 和 [cast_from_float](https://github.com/rust-lang/rust/blob/085acd02d4abaf2ccaf629134caa83cfe23283c8/compiler/rustc_const_eval/src/interpret/cast.rs#L309) 附近。

为了验证这一点，我们可以使用 `#[inline(never)]` 阻止部分优化。我通过尝试发现，添加 `#[inline(never)]` 到 `float_to_float_or_int` 仍然有问题，而添加到 `cast_from_float` 后，测试代码可以正常编译。我猜测问题出现在 `float_to_float_or_int` 和内联的函数 `cast_from_float` 中。

## 真的和 PGO 有关吗？

事实上，我们可以直接使用用于生成 PGO 的 stage2 版本 rustc 复现该问题：

```bash
./x build --rust-profile-generate=/tmp/profiles --stage 2 library
```

# 使用 git bisect 寻找是哪个提交触发了这个问题

尽管这一结果未必是有问题的提交，但它可以给我们提供一个具体的控制错误编译的方式。我们可以通过调整这个提交进一步定位问题。

## 选取一个好的提交

为了执行 git bisect，我们需要找到一个好的提交。
如果我们不能在一个大版本之间找到一个好的提交，我会放弃使用 git bisect。因为太久远的提交可能没有意义。而且随着我们忽略的提交越多，越可能出现其他不相关的问题。
这里我有一个简单的选择方式。LLVM 的 tag 有一个创建规则，我们在创建新的 release 分支时，同时创建一个提高大版本的 tag，这个 tag 规则为 `llvmorg-{version}-init`，这类 tag 的关系是线性的，对于 bisect 非常友好。
我会把 `llvmorg-18-init` 和 `llvmorg-17.0.1` 当作一致的代码。

这里我们选择的对比版本为：

- 好的提交: [b0daacf5](https://github.com/llvm/llvm-project/commit/b0daacf58f417634f7c7c9496589d723592a8f5a)，这是 `llvmorg-17-init` 的前一个提交，和 `llvmorg-16.0.0` 相似。由于 rust 使用版本号适配 LLVM 的 API 变更，我们需要保持和 API 一致的版本号。
- 坏的提交: [d0b54bb5](https://github.com/llvm/llvm-project/commit/d0b54bb50e5110a004b41fc06dadf3fee70834b7)。

## 准备 LLVM 的构建配置

由于每一步的 bisect 要花很长时间，首先我推荐使用更高性能的计算机缩短这个时间。

然后修改 `config.toml` 减少重复构建时间，我的修改如下：

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
# 使用 bisect 后 patch 的 commit 会被移除
llvm-has-rust-patches = false

[rust]
codegen-units = 256
```

同时修改 `Cargo.toml`：

```toml
[profile.release.package.rustc_const_eval]
codegen-units = 1
```

## 减少 PGO 处理的函数

这即可以减少构建时间，也可以进一步明确问题所在。

我们修改 [PGOInstrumentation.cpp#L1761](https://github.com/llvm/llvm-project/blob/llvmorg-17.0.2/llvm/lib/Transforms/Instrumentation/PGOInstrumentation.cpp#L1761) 的过滤规则，比如：

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

## 执行 git bisect

和标准的 LLVM 工程的 bisect 过程有些不同。当遇到 `PassWrapper.cpp` 编译失败时，我们需要手动适配 API 变更。而不是使用 `git bisect skip`。
由于要频繁修改 `PassWrapper.cpp`，我们不能使用 `git bisect run` 自动完成这一过程，只能手动执行并检查每次的结果。运气好的话，不超过 12 次就可以得到结果。

经过一段时间的运行，我获得的 bisect 结果是 [361464c0](https://github.com/llvm/llvm-project/commit/361464c027239a70d66fb7790032b23696d5b303)。
由于 LLVM 非常复杂，我还一般不能直接从这一提交中判断问题所在，同时这个提交未必是有问题的提交。

我一般将 bisect 结果分为几类：

- 这是一个导致错误编译发生的独立 commit
- 提交了一些误导信息，导致后面的 Pass 产生了“错误”编译（这是第一个问题的类型）
- 只是巧合，为后面的错误编译的 Pass 提供了匹配的输入或者是暴露了已有的错误编译（这是第二个问题的类型）

不过此时我们还无法给这次的结果进行分类。但我们可以使用这个结果继续调试定位。

# 通过修改 LLVM 源码定位感兴趣的转换

此时我们不知道错误编译在哪里，我们无法通过直接获取一个 IR 调试。在有明确的结论前，我们仍然编译运行 rustc 定位问题。

根据 bisect 结果，**我们需要找到是哪个函数经过 `processImmutArgument` 后，最终出现错误编译。**

使用类似下面的代码可以帮助我们逐步定位哪个函数被影响了。

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

通过日志我可以发现在 `MemCpyOptPass` 中影响的函数是：

```
<rustc_const_eval::interpret::eval_context::InterpCx<rustc_const_eval::const_eval::machine::CompileTimeInterpreter>>::float_to_float_or_int
```

从日志中我发现这个函数完成了多次 `memcpy` 转换。

所以我们可以继续过滤找到是哪一次转换导致了这个问题。

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

当我走到这一步时，我开始怀疑这次错误编译和 PGO 无关。此时我尝试取消 PGO 也可以复现该问题。
我猜测 PGO 只是一个巧合，将这个错误编译暴露到运行时。
但我们还不知道 `MemCpyOptPass` 是那种类型，可能是新的巧合，也可能是错误编译，或者是误导了后续 Pass。 

## 使用 `-opt-bisect-limit` 

我们可以使用 `-opt-bisect-limit` 定位是哪个 Pass 修改指令导致运行时出现问题。

使用 `-opt-bisect-limit` 发现的 Pass 有两种类型：

- 前面的 pass 执行了正确的转换，但是没有及时更新 metadata 等信息，导致找到的 Pass 出现问题
- 找到的 Pass 本身存在错误编译

## 为特定 crate 执行 `-opt-bisect-limit`

小插曲：本次真实调试的过程是通过修改 `OptBisect.cpp` 完成的。但在写这篇文章时，我发现了更简单高效的方法。

如果我们直接通过 `RUSTFLAGS_NOT_BOOTSTRAP` 设置 `-Cllvm-args=-opt-bisect-limit=-1` 将得到大量的无效日志。我们希望只应用到 `rustc_const_eval`。

nightly 版本的 `cargo` 提供了这个功能。我们需要先切换到 nightly。修改如下：


```toml
[build]
cargo = "<path to home>/.rustup/toolchains/nightly-x86_64-unknown-linux-gnu/bin/cargo"
```

然后更新 `Cargo.toml`。

```toml
cargo-features = ["profile-rustflags"]
# ...
[profile.release.package.rustc_const_eval]
rustflags = [
  "-C", "llvm-args=-opt-bisect-limit=-1",
]
```

接下来我们可以使用 `--keep-stage` 构建：

```bash
./x build --stage 2 library --keep-stage 0 --keep-stage-std 1 2> build.log
./build/host/stage2/bin/rustc ./test.rs
```

最终我的 bisect 结果为：

```
BISECT: running pass (560953) MemCpyOptPass on _RNvMNtNtCsiODAygBxQYA_16rustc_const_eval9interpret4castINtNtB4_12eval_context8InterpCxNtNtNtB6_10const_eval7machine22CompileTimeInterpreterE21float_to_float_or_intB6_
BISECT: running pass (560954) DSEPass on _RNvMNtNtCsiODAygBxQYA_16rustc_const_eval9interpret4castINtNtB4_12eval_context8InterpCxNtNtNtB6_10const_eval7machine22CompileTimeInterpreterE21float_to_float_or_intB6_
BISECT: NOT running pass (560955) MoveAutoInitPass on _RNvMNtNtCsiODAygBxQYA_16rustc_const_eval9interpret4castINtNtB4_12eval_context8InterpCxNtNtNtB6_10const_eval7machine22CompileTimeInterpreterE21float_to_float_or_intB6_
```

有一个让我感觉奇怪的地方是，如果我设置 limit 为 `560953`，结果不会停到 `MemCpyOptPass`。这是个奇怪的结果让本次的 bisect 结果有些不可信。

```
BISECT: running pass (560953) BDCEPass on _RNvXsg_NtNtCsjGTw0T6X7N4_16rustc_const_eval9interpret7operandNtB5_9ImmediateNtNtCs6sCMhXnFQZh_4core3fmt5Debug3fmtB9_
BISECT: NOT running pass (560954) InstCombinePass on _RNvXsg_NtNtCsjGTw0T6X7N4_16rustc_const_eval9interpret7operandNtB5_9ImmediateNtNtCs6sCMhXnFQZh_4core3fmt5Debug3fmtB9_
```

不过我们可以使用类似的方法验证问题是否与 `DSEPass` 有关，通过修改代码跳过 `DSEPass` 即可。
接下来和调试 `MemCpyOptPass` 一样，我们可以找到具体是哪个指令 `DSEPass` 转换导致了最终的错误编译。

# 获取 IR 准备 LLVM 的调试

这部分涉及到 LLVM 的具体调试，我暂时还没有什么好的经验分享。

但重要的两点是：
- 我们知道了在 `MemCpyOptPass` 中，具体是哪次转换影响了结果
- 我们也知道了在 `DSEPass` 中，具体是哪次转换导致了最终的错误编译

通过这两此转换，我们就可以使用 `opt` 调试 IR 了。我们不需要再使用 `rustc` 频繁编译了！

对于获取 IR 的方式我推荐通过修改 `Cargo.toml` 获得：

```toml
[profile.release.package.rustc_const_eval]
codegen-units = 1
rustflags = [
  "-C", "save-temps",
]
```

然后找到 `*.no-opt.bc` 进行调试。

当然在这个调试过程，我们需要知道 `MemCpyOptPass` 和 `DSEPass` 的具体两个转换的逻辑。在这里我们会了解到这是通过别名分析进行的转换，最终定位到 `MemCpyOptPass` 在替换指令使用的值时，没有更新对应的别名信息。

# 下一个问题 - mir niche checks

尽管这个问题的复现比前者要简单，但定位起来会更麻烦。
首先让我们切换到 [cf8d85e4](https://github.com/DianQK/rust/commit/cf8d85e49c6f3a5802549055f983fd0ba3f89952)。

使用 `--stage 2` 执行测试即可复现：

```bash
./x test tests/ui --stage 2
```

`config.toml` 参考：

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

错误日志如下：

```
thread 'rustc' panicked at compiler/rustc_mir_build/src/thir/pattern/deconstruct_pat.rs:560:22:
occupied niche: found 0x7f7700000000 but must be in 0x0..=0x2 in type std::option::Option<thir::pattern::deconstruct_pat::SliceKind> at offset 0 with type Int(I64, false)
stack backtrace:
  ...
  13:     0x7f77e92ade54 - <rustc_mir_build[48ebee3fe6c2e2b3]::thir::pattern::deconstruct_pat::Constructor>::split::<core[8d828210e7f791ba]::iter::adapters::map::Map<core[8d828210e7f791ba]::iter::adapters::map::Map<core[8d828210e7f791ba]::slice::iter::Iter<rustc_mir_build[48ebee3fe6c2e2b3]::thir::pattern::usefulness::PatStack>, <rustc_mir_build[48ebee3fe6c2e2b3]::thir::pattern::usefulness::Matrix>::heads::{closure#0}>, <rustc_mir_build[48ebee3fe6c2e2b3]::thir::pattern::deconstruct_pat::DeconstructedPat>::ctor>>
  ...
```

我发现如果设置 `codegen-units=1`，这个 panic 就会消失。
那么我们正好可以借助 `codegen-units` 判断是哪个 crate 被影响了。根据堆栈猜测是 `rustc_mir_build`，尝试切换 `codegen-units=1` 验证。
由于这个问题发生和上一个时间点基本一致，我们使用相同的 LLVM 提交作为 bisect 开始。
遗憾的是，如果设置 `optimize=3`，我们无法在 LLVM 16 找到一个好的提交。但我还尝试了设置 `optimize=2` 可以找到 LLVM 16 是好的提交。

## 棘手的 git bisect

在 bisect 过程中，我得到一个意外结果：

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

面对这个结果，我们不可使用 good/bad 执行 bisect，这将 bisect 导向错误的结果。

遗憾的是，即便使用 skip，我们也无法得到 bisect 结果。
当 bisect 在 [f7deb69f2...7c78cb4b](https://github.com/llvm/llvm-project/compare/f7deb69f22b93d7411d08db14b50aae5caf40fcb...7c78cb4b1f4993a84bf2b46b197d90dcabb9f8c5) 内部时，就可以得到这个预期之外的错误。如果是更早的提交，是 good，更晚的提交，是 bad。
这是因为 `nonnull` 等语义的变更后引入了新的问题，我们在 bisect 期间忽略了一些问题修复，导致暴露了一个新的问题。

此处提交历史如下：

```
bad
Revert "[SimplifyCFG][LICM] Preserve nonnull, range and align metadat…  7c78cb4
skip
[SimplifyCFG][LICM] Preserve nonnull, range and align metadata when s…  78b1fbc
good
```

由于我们忽略了一些提交导致新问题发生，bisect 没有结果。
幸运的是，这次的问题很特别的，我们可以使用 rebase 将 `78b1fbc` 和 `7c78cb4` drop 掉。

最终 bisect 结果是 [[AggressiveInstCombine] Enable also for -O2](https://github.com/DianQK/llvm-project/commit/2f70f65246e987842cfae495839975eaa6245a0c)。
我也通过修改代码找到了在 `AggressiveInstCombine` 的关键转换，但从代码和 IR 上，我看不出什么问题。可能是我疏漏了什么，也可能是这只是个幸运的触发机会。
我们需要记住这个怀疑点，继续排查。

# `codegen-units=256` & `-opt-bisect-limit=n`

这次我们没有办法使用 `-opt-bisect-limit`，因为 `codegen-units` 不等于 1。 
同时对多个 IR 进行 bisect 没有意义。我们需要修改 rust 代码支持在多个 CGU 下选择特定的 CGU 进行 bisect。

首先使用 `-C save-temps` 找到对应的 IR。修改方式参考 [bfd759b7](https://github.com/DianQK/rust/commit/bfd759b704b77d0ff390cc172a62c3d9a0c36986)。

小插曲：这里我也尝试了在 AggressiveInstCombine 中寻找有影响的那一次转换。很头痛的是，添加 `-C save-temps` 后，符号名会变，这让我重新找了一下有关联的符号。

我编写了一个简单的脚本找到这个 IR：

```bash
for bitcode in build/x86_64-unknown-linux-gnu/stage1-rustc/x86_64-unknown-linux-gnu/release/deps/rustc_mir_build-*-cgu.*.rcgu.no-opt.bc; do
    if llvm-nm -U $bitcode | grep -q "example"; then
        echo $bitcode
    fi
done
```

我查看了 `OptBisect` 的实现，尽管 `-opt-bisect-limit` 参数是全局的，但我们可以为 CGU 单独替换一个空的 `OptBisect`。完整修改在 [a9f62a4a](https://github.com/DianQK/rust/commit/a9f62a4a634aef8f792e9cb6eb4573b4b9fc79d2)。

一些关键修改参考如下：

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

我还增加了一个命令行参数 `-Z llvm-opt-bisect-limit-cgu`，这样我就可以使用下面的脚本进行 bisect：

```bash
export RUSTFLAGS_NOT_BOOTSTRAP="-C llvm-args=-opt-bisect-limit=-1 -Z llvm-opt-bisect-limit-cgu=rustc_mir_build.63d28fcded2a05ed-cgu.007"
./x build --stage 2 library --keep-stage 0 --keep-stage-std 1 2> build.log
./build/host/stage2/bin/rustc ./tests/ui/consts/const_prop_slice_pat_ice.rs
```

我还编写了一个简单的自动 bisect 脚本：

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

但我得到了一个奇怪的结果：

```
ISECT: running pass (13444) InlinerPass on (symbol)
BISECT: NOT running pass (13445) PostOrderFunctionAttrsPass on (symbol)
```

我认为 inline 与这个错误编译无直接关系。
我简单地修改了 `OptBisect.cpp` 跳过 `InlinerPass`：

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

最终我得到：

```
BISECT: running pass (10040) CorrelatedValuePropagationPass on symbol
BISECT: NOT running pass (10041) SimplifyCFGPass on symbol
```

为了验证 `CorrelatedValuePropagationPass` 是否有关，仍然是通过删除不相关的代码，参见：[a08f2c14](https://github.com/DianQK/llvm-project/commit/a08f2c14b1d501aa9b76b96e26c7366c5a6e6e9b)。
我还添加了一行日志进行简单的验证：

```
LLVMLOG: Delete   %102 = and i64 %101, 4294967295   -> and i64 %101, 0xffffffff
...
occupied niche: found 0x7fba00000001 but must be in 0x0..=0x2 in type std::option::Option<thir::pattern::deconstruct_pat::SliceKind> at offset 0 with type Int(I64, false)
0x7fba00000001 & 0xffffffff = 0x1
```

我敢肯定这就是我们要找的！这和 rustc 的 panic 日志有非常大的相关性。
最终我发现 `%101` 在特定的控制流下，可能获得 `undef` 结果。在这种情况下，我们不应该删除 `%102`。

这里我很好奇为什么会和 `AggressiveInstCombine` 有关，如果我们去掉这个 Pass，要删除的指令变为 `%123 = and i64 %122, 72057594037927935(0xffffffffffffff)`。我们仍然不能删除这个指令，只是这个巨大的数值让程序在运行期间很难遇到。

# 总结

我们比较了什么？

- 通过比较不同 `codegen-units`，定位被影响的 crate
- 通过添加 `[inline(never)]` 比较结果，定位被影响的函数
- 通过 git bisect 定位是哪个 commit 影响了结果
- 通过修改代码，定位是哪次转换影响了结果
- 通过 `-opt-bisect-limit` 结果定位有关联的 Pass

我通过这些方法逐渐接近真相。对了，运气也很重要，我没有记录我走错路的经历 :]。

在这些方法中，我也介绍了一些具体技巧：

- git bisect 拿不到结果时可以通过移除一些 commit 找到结果（这可能只是特定场景的技巧）
- cargo 提供了应用 rustc 参数到指定 crate 的功能
- 修改 rustc 代码，将 `opt-bisect-limit` 应用到具体的 CGU，让这个功能在大型项目上应当很有用

解决这类问题概括流程是：

1. 首先想办法缩短单次复现的时间，太长的调试时间令人恼火
2. 关键目标是**通过上述比较方法，找到导致错误编译的关键转换**
3. 提取 IR，根据关键转换定位问题
4. 修复问题

暂时我还没有第三步骤的心得与经验，但我会使用 `llvm-extract` 和 `llvm-reduce` 减少获得的 IR，这对于调试会有些帮助。 
我也会使用 `-opt-bisect-limit` 提取中间过程的 IR，并手动删除一些函数或指令定位问题。

对于如何提交一个合适的修复，我还没有清晰的思路，目前我很少做到无修改获得 LGTM 的 PR。我还需要更多的学习与实践 ;)。

# 参考

- [https://rustc-dev-guide.rust-lang.org/compiler-debugging.html](https://rustc-dev-guide.rust-lang.org/compiler-debugging.html)
- [https://rustc-dev-guide.rust-lang.org/backend/debugging.html](https://rustc-dev-guide.rust-lang.org/backend/debugging.html)
- [https://llvm.org/docs/GitBisecting.html](https://llvm.org/docs/GitBisecting.html)
- [https://doc.rust-lang.org/cargo/reference/unstable.html#profile-rustflags-option](https://doc.rust-lang.org/cargo/reference/unstable.html#profile-rustflags-option)
- [https://www.llvm.org/docs/AliasAnalysis.html](https://www.llvm.org/docs/AliasAnalysis.html)
