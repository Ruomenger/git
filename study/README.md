# Git 源码学习笔记

本目录用于记录阅读、构建和调试 Git 源码时得到的结论。

当前笔记基于以下源码快照：

- Git 版本：2.55.0
- 提交：`e9019fcafe0040228b8631c30f97ae1adb61bcdc`
- 平台：macOS arm64
- 构建系统：GNU Make

版本和提交可以通过以下命令重新确认：

```sh
./git version --build-options
git rev-parse HEAD
```

## 文档索引

- [架构与核心概念导览](architecture-overview.md)：对象库、引用、索引三大抽象，源码分层地图，以及 `git commit` 的完整调用链路。建议作为第一篇。
- [Debug 构建与 clangd 配置](debug-build.md)：使用 Make 构建 Debug 版本，并通过 Bear 生成 `compile_commands.json`。
- [Git 命令分发与进程模型](command-dispatch-and-process-model.md)：解释 builtin、`git-*` 硬链接、独立程序、脚本和按需子进程之间的关系。
- [Git 中的 Rust](rust-in-git.md)：Rust 的引入时间线、当前用途、C/Rust 边界和后续计划。
- [源码阅读路线](source-reading-roadmap.md)：从程序入口、命令分发到对象库、索引和引用系统的建议阅读顺序。

## 记录约定

阅读源码时尽量区分三类信息：

1. 当前代码已经启用的功能。
2. 已经进入仓库、但仍属于实验或基础设施的代码。
3. 文档或邮件列表中规划的未来变化。

涉及行为判断时，优先记录对应文件、函数和提交。例如：

```sh
rg -n "WITH_RUST|NO_RUST" Makefile repository.c help.c
git log --oneline --all -- src '*.rs'
git blame -L 490,510 Makefile
```

这样在升级源码后，可以快速确认原结论是否仍然成立。
