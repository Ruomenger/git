# Git 源码学习计划

制定日期：2026-09-07。基线：Git 2.55.0，提交 `e9019fcafe0040228b8631c30f97ae1adb61bcdc`，macOS arm64，Debug 构建见 [debug-build.md](debug-build.md)。

本文是学习路线本身，不记录源码结论。各阶段得到的结论应写入对应专题笔记，并在文末进度表中登记。文中所有练习命令均在制定日期用本仓库编译出的 `./git` 实际运行过。

## 起点与目标

起点：

- 多年终端手工使用 Git，分支管理、版本回退、rebase 熟练。
- 很少使用 merge；对 worktree、bisect、rerere、notes 等命令了解不多。
- 没有系统读过 Git 内部原理和源码。

目标按顺序递进：

1. 能用对象库、引用、索引三个结构解释每一条已经会用的命令。
2. 理解三方合并，并据此理解 rebase、cherry-pick、revert 共用的机制。
3. 能沿一条命令从 `main()` 读到具体子系统，并用 LLDB 验证实际路径。
4. 能独立挑一个专题深入，并把结论写回 `study/`。

原则：

- plumbing 先于 porcelain，数据先于代码。
- 每次只追一个可验证的问题。
- 一手资料优先：`Documentation/`、`t/` 测试、`git log`，其次才是外部文章。
- 结论必须落到 `study/`，附文件、函数、提交哈希或命令输出。

## 每次学习的循环

```text
提出一个具体问题
  │
  ▼
让模型从源码追出调用链，要求给出 文件:行号
  │
  ▼
在练手仓库用 ./git 复现；必要时用 LLDB 下断点确认实际走到的分支
  │
  ▼
验证通过的结论写入 study/ 对应笔记，更新 README 索引
```

提问示例：

- `git reset --hard HEAD~1` 依次修改了 HEAD、index、工作区中的哪些内容？源码里对应哪些函数？
- rebase 出现冲突时，index 里的 stage 1/2/3 分别来自哪三个 tree？
- `git branch foo` 在 files 后端下最终写了哪个文件？中间经过了哪些锁和事务步骤？

不接受没有文件和行号的解释。行号绑定在当前基线上，升级源码后用 `git blame` 复核。

## 环境准备

### 练手仓库

在源码树之外建练手仓库，全程使用本仓库编译出的 `./git`，避免混入系统 Git。

编译出的 `git` 内置 exec path 是 `$HOME/libexec/git-core`，本机并不存在，因此脚本命令和 helper 需要显式指定 `GIT_EXEC_PATH`。建议在 shell 配置中加入：

```sh
alias g='GIT_EXEC_PATH=/Volumes/Dev/src/c/git GIT_TEMPLATE_DIR=/Volumes/Dev/src/c/git/templates/blt /Volumes/Dev/src/c/git/git'
```

然后初始化练手仓库：

```sh
mkdir -p ~/tmp/git-lab && cd ~/tmp/git-lab
g init -b master
g config user.name lab
g config user.email lab@example.com
```

不设置 `GIT_TEMPLATE_DIR` 时 `g init` 会警告 `templates not found`，只影响默认 hooks 样例，不影响学习。

### neovim 与 clangd

`compile_commands.json` 已由 Bear 生成，见 [debug-build.md](debug-build.md)。16 GB 内存下建议限制 clangd 后台索引并发，nvim-lspconfig 示例：

```lua
require("lspconfig").clangd.setup({
  cmd = {
    "clangd",
    "-j=2",
    "--background-index",
    "--background-index-priority=background",
    "--malloc-trim",
    "--header-insertion=never",
  },
})
```

clangd 的索引缓存写在仓库根目录 `.cache/`，上游 [.gitignore](../.gitignore) 已忽略该目录，不需要额外处理。不要同时用 CLion 或 VS Code 索引本仓库。

### 调试与考古

- LLDB 直接在终端使用：`lldb -- ./git status`。断点清单见 [debug-build.md](debug-build.md) 和 [command-dispatch-and-process-model.md](command-dispatch-and-process-model.md)。
- 观察行为：`GIT_TRACE=1` 看进程边界，`GIT_TRACE2_PERF=1` 看各阶段耗时。
- 考古三件套，以 `cache_tree_update` 为例：

```sh
rg -n 'cache_tree_update\s*\(' --glob '*.[ch]'
git log -S'cache_tree_update' --oneline -- cache-tree.c
git blame -L 100,120 cache-tree.c
```

- `t/` 目录是可执行的规格说明。读一个命令前先翻它的测试。例如 [t/t7111-reset-table.sh](../t/t7111-reset-table.sh) 用一张表穷举了 `reset` 各模式下 HEAD、index、工作区的结果。

## 第一阶段：用 plumbing 建立数据模型

目标：手工造出一个 commit，然后把已经会用的操作翻译成三个结构上的变化。

### 练习 1：手工造 commit

```sh
cd ~/tmp/git-lab
printf 'hello\n' > a.txt
g hash-object -w a.txt
# ce013625030ba8dba906f756967f9e9ca394464a
find .git/objects -type f
# .git/objects/ce/013625030ba8dba906f756967f9e9ca394464a

g update-index --add a.txt
g write-tree
# 2e81171448eb9f2ee3821e3d447aa6b2fe3ddba1
g commit-tree 2e81171448eb9f2ee3821e3d447aa6b2fe3ddba1 -m init
# 输出 commit ID；作者和时间不同，每次都会变
g update-ref refs/heads/master <commit-id>

cat .git/refs/heads/master
g cat-file -p HEAD
g cat-file -p HEAD^{tree}
g ls-files --stage
```

每一步都对照 `.git` 目录观察：`hash-object -w` 让 blob 落盘；`update-index` 只改 `.git/index`；`write-tree` 才生成 tree 对象；`update-ref` 写的是一个 41 字节的文本文件。blob ID 与手工 `shasum` 的验证方法见 [architecture-overview.md](architecture-overview.md) 第一节。

### 练习 2：把熟悉的操作翻译到四层结构

| 日常操作 | 对象库 | 引用 | 索引 | 工作区 |
| --- | --- | --- | --- | --- |
| `git branch foo` | 不变 | 新增 `refs/heads/foo` | 不变 | 不变 |
| `git switch foo` | 不变 | HEAD 改指 `foo` | 重写为 `foo` 的 tree | 更新有差异的文件 |
| `git reset --soft X` | 不变 | 当前分支改指 X | 不变 | 不变 |
| `git reset --mixed X` | 不变 | 当前分支改指 X | 重写为 X 的 tree | 不变 |
| `git reset --hard X` | 不变 | 当前分支改指 X | 重写为 X 的 tree | 覆盖为 X 的 tree |
| `git commit` | 新增 tree 和 commit | 当前分支前进 | 不变 | 不变 |
| `git stash` | 新增 2 到 3 个 commit | `refs/stash` 更新 | 重写为 HEAD | 覆盖为 HEAD |
| `git checkout <commit>` | 不变 | HEAD 变为 detached | 重写 | 更新 |

每一行都要在练手仓库里做一遍，并用 `g reflog`、`g ls-files --stage`、`g cat-file -p refs/stash` 验证。注意"对象库"一列几乎总是"不变"或"新增"：Git 不删除对象，只是没有名字指向它，这是 reflog 能救回来的根本原因。

### 阅读

- [Documentation/gitcore-tutorial.adoc](../Documentation/gitcore-tutorial.adoc)：官方 plumbing 教程，与练习 1 同路。
- [architecture-overview.md](architecture-overview.md) 第一、二节。
- [Documentation/gitrevisions.adoc](../Documentation/gitrevisions.adoc)：`@{u}`、`^{tree}`、`:path`、`A...B` 等写法。
- [Documentation/git-reset.adoc](../Documentation/git-reset.adoc) 的 DISCUSSION 一节，用表格说明各模式。
- 测试：[t/t7111-reset-table.sh](../t/t7111-reset-table.sh)、[t/t1400-update-ref.sh](../t/t1400-update-ref.sh)、[t/t3903-stash.sh](../t/t3903-stash.sh)。

### 源码入口

| 练习命令 | 入口 | 核心实现 |
| --- | --- | --- |
| `hash-object` | [builtin/hash-object.c](../builtin/hash-object.c) | [object-file.c](../object-file.c) |
| `update-index` | [builtin/update-index.c](../builtin/update-index.c) | [read-cache.c](../read-cache.c) |
| `write-tree` | [builtin/write-tree.c](../builtin/write-tree.c) | [cache-tree.c](../cache-tree.c) |
| `commit-tree` | [builtin/commit-tree.c](../builtin/commit-tree.c) | [commit.c](../commit.c) |
| `update-ref` | [builtin/update-ref.c](../builtin/update-ref.c) | [refs.c](../refs.c)、[refs/files-backend.c](../refs/files-backend.c) |
| `reset` | [builtin/reset.c](../builtin/reset.c) | [reset.c](../reset.c)、[unpack-trees.c](../unpack-trees.c) |
| `stash` | [builtin/stash.c](../builtin/stash.c) | 同上，stash 本质是 commit 加 reset |

### 完成标准

- 不看资料能画出对象库、引用、索引、工作区的关系图，并说出 `diff`、`diff --cached`、`diff HEAD` 各比较哪两级。
- 能从机制层解释 `reset` 三种模式、detached HEAD、reflog 恢复。
- 能用 `cat-file` 读出任意 commit、tree、blob、tag 的原始内容并解释每个字段。

## 第二阶段：用 merge 讲透 rebase

目标：理解三方合并，并认识到 rebase、cherry-pick、revert 每一步都在复用同一套合并代码。

### 练习 3：制造冲突并观察

```sh
cd ~/tmp/git-lab
g switch -c topic
printf 'topic\n' >> a.txt && g commit -qam topic
g switch master
printf 'master\n' >> a.txt && g commit -qam master

g merge-base master topic
# 输出 init 那个 commit，即共同祖先

g merge-tree --write-tree master topic
# 不碰工作区和 index，直接输出合并结果：
# <合并后的 tree id>
# 100644 ce013625... 1	a.txt     stage 1：base
# 100644 858e207b... 2	a.txt     stage 2：ours
# 100644 1470cb6a... 3	a.txt     stage 3：theirs
# Auto-merging a.txt
# CONFLICT (content): Merge conflict in a.txt

g -c merge.conflictStyle=diff3 merge topic
g ls-files -u        # 同一路径三条记录，stage 1/2/3
cat a.txt            # diff3 风格能同时看到 base 版本
g merge --abort
```

### 关键概念

1. 三方合并的输入是三个 tree：merge-base、ours、theirs。输出是一个新 tree 加冲突列表。
2. 冲突存在 index 里，不在工作区。工作区里的 `<<<<<<<` 标记只是三个 stage 的一种渲染。
3. rebase 的每一步是一次 cherry-pick；cherry-pick 是一次三方合并，其中 base 是被 pick 的 commit 的父提交，ours 是当前 HEAD，theirs 是被 pick 的 commit。因此 rebase 冲突时的 ours/theirs 方向与 merge 相反，这是常见困惑的根源。
4. revert 也是三方合并，只是把 commit 和它的父提交在 base 与 theirs 的位置上互换。
5. 这些流程都由 [sequencer.c](../sequencer.c) 驱动，合并本身由 [merge-ort.c](../merge-ort.c) 完成，文本级冲突由 [merge-ll.c](../merge-ll.c) 和 [xdiff/](../xdiff) 处理。

### 阅读

- [Documentation/git-merge-tree.adoc](../Documentation/git-merge-tree.adoc)：不碰工作区的合并，是观察合并结果最干净的入口。
- [Documentation/technical/trivial-merge.adoc](../Documentation/technical/trivial-merge.adoc)：三方合并的 case 表。
- [Documentation/technical/api-merge.adoc](../Documentation/technical/api-merge.adoc)：合并 API 概览。
- [Documentation/technical/rerere.adoc](../Documentation/technical/rerere.adoc)：冲突解法记录与重放。
- [Documentation/git-rebase.adoc](../Documentation/git-rebase.adoc) 中 `--onto` 的图示。
- 测试：[t/t7600-merge.sh](../t/t7600-merge.sh)、[t/t1000-read-tree-m-3way.sh](../t/t1000-read-tree-m-3way.sh)、[t/t3400-rebase.sh](../t/t3400-rebase.sh)、[t/t3500-cherry.sh](../t/t3500-cherry.sh)。

### 源码入口

| 命令 | 入口 | 核心实现 |
| --- | --- | --- |
| `merge` | [builtin/merge.c](../builtin/merge.c) | [merge-ort.c](../merge-ort.c)、[merge-ll.c](../merge-ll.c) |
| `merge-tree` | [builtin/merge-tree.c](../builtin/merge-tree.c) | [merge-ort.c](../merge-ort.c) |
| `rebase` | [builtin/rebase.c](../builtin/rebase.c) | [sequencer.c](../sequencer.c) |
| `cherry-pick` / `revert` | [builtin/revert.c](../builtin/revert.c) | [sequencer.c](../sequencer.c) |
| `rerere` | [builtin/rerere.c](../builtin/rerere.c) | [rerere.c](../rerere.c) |

读 `merge-ort.c` 时从 `merge_incore_nonrecursive()`、`merge_incore_recursive()` 和 `merge_switch_to_result()` 三个公开函数入手；读 `sequencer.c` 时从 `sequencer_pick_revisions()` 追到 `pick_commits()` 和 `do_pick_commit()`。

### 完成标准

- 能说出 merge-base 如何求得，以及 `A...B` 语法与它的关系。
- 看到 `ls-files -u` 输出能说出每条记录来自哪个 tree。
- 能解释 rebase 冲突时 ours/theirs 为什么和 merge 相反。
- 能在 LLDB 中于 `merge_incore_nonrecursive` 下断点，跑一次冲突合并并观察三个输入 tree。

## 第三阶段：沿一条命令读进源码

目标：从 `main()` 读到具体子系统，并用断点确认实际路径。按 [source-reading-roadmap.md](source-reading-roadmap.md) 的第一至第三阶段进行。

1. 命令分发：读 [command-dispatch-and-process-model.md](command-dispatch-and-process-model.md)，用 `GIT_TRACE=1` 确认 builtin 与外部命令的边界。
2. 仓库发现：以 `g rev-parse --show-toplevel` 为入口读 [setup.c](../setup.c)，理解普通仓库、裸仓库、linked worktree 和 `GIT_DIR` 的关系。
3. commit 全链路：按 [architecture-overview.md](architecture-overview.md) 第二节，在四个函数下断点跑一次真实提交：

```text
(lldb) settings set target.env-vars GIT_EXEC_PATH=/Volumes/Dev/src/c/git
(lldb) breakpoint set --name cmd_commit
(lldb) breakpoint set --name cache_tree_update
(lldb) breakpoint set --name commit_tree_extended
(lldb) breakpoint set --name ref_transaction_commit
(lldb) run
```

4. 从读到改：跟着 [Documentation/MyFirstObjectWalk.adoc](../Documentation/MyFirstObjectWalk.adoc) 写一个遍历提交图的 builtin。这是上游维护者写的源码级教程，完成后即掌握 [revision.c](../revision.c) 的基本用法。

### 完成标准

- 不看笔记能说出 `git status` 从 `main()` 到 `cmd_status()` 的路径。
- 能解释 commit 的 tree 为什么来自 index 而不是工作区。
- MyFirstObjectWalk 的练习能编译运行。

## 第四阶段：专题与命令盲区

完成前三阶段后，按兴趣选一个专题深入。命令盲区在日常遇到时逐个补，不必按表背。

### 专题候选

| 专题 | 入口文件 | 一手文档 |
| --- | --- | --- |
| 引用与 reftable | [refs.c](../refs.c)、[refs/files-backend.c](../refs/files-backend.c)、[refs/reftable-backend.c](../refs/reftable-backend.c) | [technical/reftable.adoc](../Documentation/technical/reftable.adoc) |
| pack 与 delta | [packfile.c](../packfile.c)、[builtin/pack-objects.c](../builtin/pack-objects.c) | [gitformat-pack.adoc](../Documentation/gitformat-pack.adoc)、[technical/pack-heuristics.adoc](../Documentation/technical/pack-heuristics.adoc) |
| index 与 status 性能 | [read-cache.c](../read-cache.c)、[wt-status.c](../wt-status.c)、[dir.c](../dir.c) | [gitformat-index.adoc](../Documentation/gitformat-index.adoc)、[technical/racy-git.adoc](../Documentation/technical/racy-git.adoc) |
| revision 遍历 | [revision.c](../revision.c)、[commit-reach.c](../commit-reach.c)、[commit-graph.c](../commit-graph.c) | [technical/commit-graph.adoc](../Documentation/technical/commit-graph.adoc) |
| 传输协议 | [connect.c](../connect.c)、[fetch-pack.c](../fetch-pack.c)、[upload-pack.c](../upload-pack.c)、[pkt-line.c](../pkt-line.c) | [gitprotocol-v2.adoc](../Documentation/gitprotocol-v2.adoc) |
| Rust 边界 | [src/](../src)、[varint.h](../varint.h) | [rust-in-git.md](rust-in-git.md) |

### 命令盲区

| 命令 | 一句话原理 | 源码 | 测试 |
| --- | --- | --- | --- |
| `worktree` | 多个工作区共享一个对象库，各自有 HEAD 和 index | [builtin/worktree.c](../builtin/worktree.c)、[worktree.c](../worktree.c) | [t/t2400-worktree-add.sh](../t/t2400-worktree-add.sh) |
| `stash` | 一个带 2 到 3 个父提交的 commit，挂在 `refs/stash`，靠 reflog 存栈 | [builtin/stash.c](../builtin/stash.c) | [t/t3903-stash.sh](../t/t3903-stash.sh) |
| `bisect` | 在提交图上二分，状态存在 `.git/BISECT_*` | [builtin/bisect.c](../builtin/bisect.c)、[bisect.c](../bisect.c) | [t/t6030-bisect-porcelain.sh](../t/t6030-bisect-porcelain.sh) |
| `rerere` | 记录冲突 hunk 的解法并自动重放 | [rerere.c](../rerere.c) | [t/t4200-rerere.sh](../t/t4200-rerere.sh) |
| `range-diff` | 比较两段提交序列，rebase 前后自审 | [range-diff.c](../range-diff.c) | [t/t3206-range-diff.sh](../t/t3206-range-diff.sh) |
| `notes` | 用独立 tree 给 commit 挂附注，不改哈希 | [notes.c](../notes.c) | [t/t3301-notes.sh](../t/t3301-notes.sh) |
| `reflog` | 每个 ref 的取值历史，reset 和 rebase 后的后悔药 | [reflog.c](../reflog.c)、[refs/files-backend.c](../refs/files-backend.c) | [t/t1410-reflog.sh](../t/t1410-reflog.sh) |
| `fsck` | 校验对象图完整性与可达性 | [builtin/fsck.c](../builtin/fsck.c)、[fsck.c](../fsck.c) | [t/t1450-fsck.sh](../t/t1450-fsck.sh) |

### 完成标准

- 至少一个专题形成独立笔记，含调用链、断点现场和一手文档链接。
- 命令盲区中至少一半能在练手仓库演示并解释其数据结构。

## 进度记录

| 阶段 | 状态 | 产出笔记 | 备注 |
| --- | --- | --- | --- |
| 环境准备 | 已完成 | [debug-build.md](debug-build.md) | 2026-08-04 完成 Debug 构建与 `compile_commands.json` |
| 第一阶段：数据模型 | 未开始 | | |
| 第二阶段：merge 与 rebase | 未开始 | | |
| 第三阶段：命令调用链 | 未开始 | | |
| 第四阶段：专题 | 未开始 | | |

每完成一个阶段，更新这张表，并在对应笔记中记录结论和验证命令。
