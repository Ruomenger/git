# Git 源码学习仓库说明

## 仓库用途

这是 Git 上游源码的学习仓库。`study` 分支用于阅读、调试和记录 Git 内部实现，尽量保持上游源码不被无关修改污染。

当前学习基线：

- 上游版本：Git 2.55.0
- 上游基线提交：`e9019fcafe0040228b8631c30f97ae1adb61bcdc`
- 学习分支：`study`
- 构建系统：顶层 GNU Makefile，不使用 CMake
- 主要平台：macOS arm64

开始工作前先执行：

```sh
git branch --show-current
git status --short --branch
git log -1 --oneline
```

不要覆盖用户未提交的修改，也不要清理不属于当前任务的构建产物。

## 首要阅读入口

学习笔记统一放在 `study/`：

- `study/README.md`：笔记索引和记录约定。
- `study/debug-build.md`：Make Debug 构建、Bear、clangd 和 LLDB。
- `study/rust-in-git.md`：Git 引入 Rust 的时间线、用途和 C/Rust 边界。
- `study/source-reading-roadmap.md`：程序入口、索引、对象库、引用、diff 和网络代码的阅读路线。

处理相关任务时先阅读对应文档，不要重复从零调研。新结论应补充到合适的 Markdown 文件，并更新 `study/README.md` 索引。

## 本地 Debug 构建上下文

仓库根目录使用被 `.gitignore` 忽略的 `config.mak` 保存本地 Debug 配置。当前配置意图是：

```make
DEBUG = 1
CFLAGS = -g3 -O0 -Wall -fno-omit-frame-pointer
RUST_TARGET_DIR = target/debug
CARGO_ARGS = --quiet
```

注意：当前 Makefile 在包含 `config.mak` 之前就计算 Rust profile，因此不能只设置 `DEBUG = 1`；还要显式覆盖 `RUST_TARGET_DIR` 和 `CARGO_ARGS`，否则可能得到 C Debug、Rust Release 的混合产物。

常用命令：

```sh
make -j8
./git version --build-options
```

仓库根目录的 `compile_commands.json` 由 Bear 从真实 Make 编译过程生成，也被 `.gitignore` 忽略。需要完整重建数据库时使用：

```sh
make clean
bear --output compile_commands.json -- make -j8
```

Bear 只能捕获实际执行的编译命令，所以生成完整数据库前必须确保发生全量编译。`make clean` 会删除旧的 `compile_commands.json`。在受限沙箱中 Bear 可能因进程拦截权限报 `Operation not permitted`，此时应请求必要权限后重试，不要改用虚构或手工拼接的数据库。

clangd 快速验证：

```sh
clangd --check=hex-ll.c --compile-commands-dir=. --log=error
```

## Rust 调研摘要

旧版 Git 的普通 Make 构建不要求 Rust。当前 Git 2.55 是首个默认启用 Rust 的版本，仍可通过 `NO_RUST = YesPlease` 退出；Git 3.0 计划将 Rust 变成强制构建依赖，但官方尚未给出确定发布日期。

当前源码中：

- `src/varint.rs` 是已通过 C ABI 进入实际 C 调用路径的主要 Rust 模块，用于 index v4 路径压缩和 untracked cache。
- `src/hash.rs`、`src/csum_file.rs`、`src/loose.rs` 是 SHA-1/SHA-256 互操作和对象映射基础设施。
- 不要把已经进入源码的基础设施误写成已经全面接管对象数据库；需要区分已启用功能、实验代码和未来计划。

详细结论、提交时间线和上游资料见 `study/rust-in-git.md`。

## 学习记录规范

1. 优先使用 `rg`、`git log`、`git show` 和 `git blame` 从当前源码和历史中取证。
2. 明确区分当前实现、已合入但未完全接线的基础设施、以及邮件或文档中的未来规划。
3. 记录结论时尽量给出文件、函数、提交哈希和上游一手资料。
4. 文档使用中文，代码、函数名、构建参数和命令保持原文。
5. 新增专题优先放在 `study/`，并从 `study/README.md` 链接。
6. 除非用户明确要求修改实现，否则源码学习和调研任务只修改学习文档。
7. `config.mak` 和 `compile_commands.json` 是本地忽略文件，不要加入提交。

文档修改后至少检查：

```sh
git diff --check
git status --short
```

还应确认 Markdown 中新增的仓库内相对链接确实存在，并确保代码围栏成对出现。

## Git 提交规范

- 只有用户明确要求提交时才创建 Git commit。
- 永远不要在提交信息中添加 `Co-Authored-By`。
- 标题使用约定式提交前缀加简洁中文描述，例如：

```text
docs: 添加 Git 源码学习笔记
fix: 修正 Debug 构建参数
```

- 提交正文使用 `1. 2. 3.` 编号列表逐条说明改动，例如：

```text
1. 记录构建配置和验证步骤
2. 补充相关源码入口
3. 更新学习文档索引
```

- 提交前检查暂存区，只提交当前任务范围内的文件：

```sh
git diff --cached --check
git diff --cached --stat
git diff --cached --name-status
```
