# Debug 构建与 clangd 配置

本文记录如何使用 Git 原生的 Make 构建系统生成 Debug 版本，并用 Bear 捕获真实编译命令，供 clangd 索引源码。

`compile_commands.json` 是 Clang Compilation Database 的标准文件，并不要求项目使用 CMake。Bear 可以从 Make 的实际子进程中生成该文件。

## 本地工具

本次配置使用的工具版本如下：

```text
GNU Make 3.81
Bear 4.2.1
Homebrew clangd 23.1.0
Apple clang 17.0.0 (/usr/bin/cc)
```

可以重新检查本机环境：

```sh
make --version
bear --version
clangd --version
cc --version
```

## Debug 配置

Git 的顶层 [Makefile](../Makefile) 会读取仓库根目录的 `config.mak`。该文件专门用于本地构建配置，并且已经被 `.gitignore` 忽略。

当前使用的 `config.mak`：

```make
# Local debug build settings. This file is intentionally ignored by Git.
DEBUG = 1
CFLAGS = -g3 -O0 -Wall -fno-omit-frame-pointer

# These are selected before config.mak is included, so override them here too.
RUST_TARGET_DIR = target/debug
CARGO_ARGS = --quiet
```

各选项的作用：

- `-g3`：生成完整调试信息，包括宏定义信息。
- `-O0`：关闭优化，使源码、变量和执行顺序更适合单步调试。
- `-fno-omit-frame-pointer`：保留栈帧指针，改善调用栈回溯。
- `DEBUG = 1`：声明 Debug 构建模式。
- `RUST_TARGET_DIR = target/debug`：让内部 Rust 静态库使用 Debug profile。
- `CARGO_ARGS = --quiet`：覆盖 Makefile 早期追加的 `--release`，同时保留安静输出。

### Rust 配置的易错点

当前 [Makefile](../Makefile) 在包含 `config.mak` 之前，就根据 `DEBUG` 计算了 `RUST_TARGET_DIR` 和 `CARGO_ARGS`。因此，仅在 `config.mak` 中设置：

```make
DEBUG = 1
```

只能可靠改变后续读取的配置，不能撤销此前已经追加到 `CARGO_ARGS` 的 `--release`。这也是配置中显式覆盖两个 Rust 变量的原因。

可以用下面的命令检查 Make 最终采用的值：

```sh
make -pn 2>/dev/null |
  rg '^(DEBUG|CFLAGS|CARGO_ARGS|RUST_TARGET_DIR) ?[:+?]?='
```

预期结果：

```text
DEBUG = 1
CFLAGS = -g3 -O0 -Wall -fno-omit-frame-pointer
RUST_TARGET_DIR = target/debug
CARGO_ARGS = --quiet
```

## 全量构建并生成编译数据库

Bear 只能捕获本次构建真正执行的编译命令。如果 `.o` 文件已经是最新状态，直接运行 Bear 得到的数据库可能不完整。因此先清理，再执行全量构建：

```sh
make clean
bear --output compile_commands.json -- make -j8
```

注意：Git 的 `make clean` 会删除旧的 `compile_commands.json`，所以应当先清理，再启动 Bear。

Bear 需要拦截编译器子进程。在受限沙箱、终端安全策略或 macOS 权限控制下，可能出现：

```text
Failed to create collector: Operation not permitted
```

这不是 Git 的编译错误，需要允许 Bear 进行进程拦截，或者在具有相应权限的普通终端中重新执行。

成功后，仓库根目录会生成：

```text
compile_commands.json
```

该文件也已经被 Git 的 `.gitignore` 忽略，不会污染源码提交。

## 验证构建结果

### 验证 Git 可执行文件

```sh
./git --version
./git version --build-options
```

本次构建结果包含：

```text
git version 2.55.0
cpu: arm64
rust: enabled
```

### 验证编译数据库

统计编译单元：

```sh
jq 'length' compile_commands.json
jq '[.[].file] | unique | length' compile_commands.json
```

本次生成了 565 个唯一 C 编译单元。

检查是否所有条目都包含 Debug 参数：

```sh
jq '[.[] | select((.arguments | index("-O0")) == null)] | length' \
  compile_commands.json

jq '[.[] | select((.arguments | index("-g3")) == null)] | length' \
  compile_commands.json
```

两条命令都应输出 `0`。

### 验证 clangd

clangd 会从源文件所在目录向父目录查找 `compile_commands.json`，因此数据库放在仓库根目录即可：

```sh
clangd --check=hex-ll.c --compile-commands-dir=. --log=error
```

成功时退出码为 `0`。

`clangd --check` 还会测试部分代码操作。复杂宏有时会触发 `SwapBinaryOperands` 等 tweak 自测失败，这不一定代表语法分析或编译数据库有问题；可以先换一个较小的编译单元验证，再检查完整日志中的语法诊断。

## 使用 LLDB 调试

应明确运行当前源码构建出的 `./git`，避免误调试系统安装的 Git：

```sh
lldb -- ./git status
```

常用操作：

```text
(lldb) breakpoint set --name main
(lldb) breakpoint set --name cmd_main
(lldb) run
(lldb) next
(lldb) step
(lldb) bt
(lldb) frame variable
```

调试具体内建命令时，可以在对应的 `cmd_*` 函数设置断点。例如：

```text
(lldb) breakpoint set --name cmd_status
(lldb) breakpoint set --name cmd_commit
(lldb) breakpoint set --name cmd_cat_file
```

Git 的部分命令会继续执行其他 Git 子程序。如果需要确保子程序也来自当前构建目录，可以在 LLDB 中设置：

```text
(lldb) settings set target.env-vars GIT_EXEC_PATH=/Volumes/Dev/src/c/git
```

## 日常增量构建

修改源码后通常只需要：

```sh
make -j8
```

Bear 数据库记录的是编译参数，不需要每次修改 `.c` 文件都重新生成。以下情况建议重新生成：

- 修改了 `config.mak`、编译器或 CFLAGS。
- 切换到构建结构变化较大的分支。
- 新增或删除了编译单元。
- clangd 找不到新文件的编译命令。

此时重新执行：

```sh
make clean
bear --output compile_commands.json -- make -j8
```

## 可选：纯 C 构建

Git 2.55 仍允许关闭 Rust：

```make
NO_RUST = YesPlease
```

关闭后，Make 会编译 [varint.c](../varint.c) 作为 Rust 实现的替代。但 SHA-1/SHA-256 兼容对象格式的部分实验功能也会被关闭。切换该配置后必须清理并重新生成编译数据库。
