# Git 源码阅读路线

Git 源码规模较大，直接按文件名字母顺序阅读很容易失去上下文。更有效的方法是选一个命令，从进程入口开始沿调用链追踪，再逐步扩展到索引、对象库、引用和传输层。

## 第一阶段：程序入口与命令分发

建议先理解下面这条主线：

```text
main()
  │
  ▼
common-main.c
  │
  ▼
cmd_main() in git.c
  │
  ├── handle_builtin()
  │      └── cmd_status / cmd_commit / cmd_log / ...
  │
  └── execv_dashed_builtin() / run_argv()
```

关键文件：

- [common-main.c](../common-main.c)：所有主要 C 程序共享的 `main()`。
- [common-init.c](../common-init.c)：进程级初始化。
- [git.c](../git.c)：Git 顶层参数解析和命令分发。
- [builtin.h](../builtin.h)：内建命令函数声明。
- [command-list.txt](../command-list.txt)：命令分类和生成输入。
- `builtin/*.c`：各内建命令实现。

第一个调试练习：

```sh
lldb -- ./git status
```

```text
(lldb) breakpoint set --name main
(lldb) breakpoint set --name cmd_main
(lldb) breakpoint set --name handle_builtin
(lldb) breakpoint set --name cmd_status
(lldb) run
```

目标是回答：

1. `git status` 如何从字符串变成 `cmd_status()` 调用？
2. 哪些命令编译进同一个 `git` 可执行文件？
3. 哪些命令通过脚本或独立程序执行？

## 第二阶段：仓库发现和配置读取

以以下命令为入口：

```sh
./git rev-parse --show-toplevel
```

关键文件：

- [setup.c](../setup.c)：从当前目录向上发现 `.git`、处理 worktree 和环境变量。
- [repository.c](../repository.c)：`struct repository` 的初始化与生命周期。
- [repository.h](../repository.h)：仓库级状态的核心结构。
- [config.c](../config.c)：配置文件读取和解析。
- [environment.c](../environment.c)：全局环境和仓库相关路径。

建议搜索：

```sh
rg -n 'setup_git_directory|discover_git_directory|repo_init' \
  setup.c repository.c '*.h'
```

重点理解普通仓库、裸仓库、linked worktree 和 `GIT_DIR` 之间的关系。

## 第三阶段：索引与工作区状态

以 `git status`、`git add` 或 `git diff --cached` 为入口。

关键文件：

- [read-cache.c](../read-cache.c)：index 的读取、写入和 cache entry 操作。
- [read-cache.h](../read-cache.h)：索引 API。
- [cache-tree.c](../cache-tree.c)：索引对应的 tree 缓存。
- [name-hash.c](../name-hash.c)：路径查找加速。
- [wt-status.c](../wt-status.c)：工作区状态收集和输出。
- [builtin/add.c](../builtin/add.c)：`git add`。
- [builtin/commit.c](../builtin/commit.c)：`git status` 和 `git commit` 的命令入口。

阅读目标：

1. `.git/index` 如何进入内存？
2. `struct cache_entry` 保存了什么？
3. Git 如何区分 HEAD、index 和 working tree 的变化？
4. index v4 的路径前缀压缩如何使用 varint？

## 第四阶段：对象库

以以下命令为入口：

```sh
./git cat-file -t HEAD
./git cat-file -p HEAD
./git hash-object README.md
```

关键文件：

- [object.c](../object.c)：对象类型和内存对象。
- [object-file.c](../object-file.c)：对象读取、解析和哈希相关入口。
- [object-name.c](../object-name.c)：把 revision/object name 解析为 object ID。
- [odb.c](../odb.c) 与 `odb/`：对象数据库和数据源抽象。
- [packfile.c](../packfile.c)：pack 文件管理。
- [builtin/cat-file.c](../builtin/cat-file.c)：`git cat-file`。
- [builtin/hash-object.c](../builtin/hash-object.c)：`git hash-object`。

建议先追踪 loose object，再追踪 packed object。不要一开始同时研究 delta、midx、commit-graph 和 partial clone。

一个适合调试的主线是：

```text
cmd_cat_file()
  └── repo_get_oid()
        └── object name resolution
  └── object_info / read_object
        ├── loose object source
        └── packed object source
```

## 第五阶段：引用系统

以以下命令为入口：

```sh
./git show-ref
./git symbolic-ref HEAD
./git update-ref refs/heads/example HEAD
```

关键文件：

- [refs.c](../refs.c)：引用系统公共 API。
- [refs.h](../refs.h)：公开数据结构和函数。
- `refs/files-backend.c`：传统 files 引用后端。
- `refs/reftable-backend.c`：reftable 引用后端。
- [builtin/show-ref.c](../builtin/show-ref.c)：查看引用。
- [builtin/update-ref.c](../builtin/update-ref.c)：引用事务更新。

重点理解：

- 普通 ref 与 symbolic ref。
- reflog。
- ref transaction 和锁。
- files 与 reftable 后端如何通过统一接口接入。

## 第六阶段：提交图遍历

以以下命令为入口：

```sh
./git log --oneline --graph -20
./git rev-list --parents HEAD
./git merge-base HEAD HEAD~10
```

关键文件：

- [revision.c](../revision.c)：revision 参数解析和遍历框架。
- [commit.c](../commit.c)：commit 对象处理。
- [commit-reach.c](../commit-reach.c)：可达性和 merge-base。
- [commit-graph.c](../commit-graph.c)：commit-graph 加速结构。
- [list-objects.c](../list-objects.c)：对象遍历。
- [builtin/rev-list.c](../builtin/rev-list.c)：`git rev-list`。
- [builtin/log.c](../builtin/log.c)：`git log`。

`git rev-list` 是理解 Git 图遍历的好入口，因为它比 `git log` 的格式化和用户界面逻辑更集中。

## 第七阶段：diff 和 merge

关键文件：

- [diff.c](../diff.c)：diff 核心配置和输出。
- [diff-lib.c](../diff-lib.c)：工作区、索引和 tree 之间的 diff。
- `xdiff/`：底层文本差异算法。
- [merge-ort.c](../merge-ort.c)：当前主要 merge 策略实现。
- [merge-ll.c](../merge-ll.c)：低层文件合并和 merge driver。

建议先比较：

```sh
./git diff
./git diff --cached
./git diff HEAD
```

观察三条命令分别比较哪两个状态，以及最终如何汇合到 diffcore 和 xdiff。

## 第八阶段：网络传输

关键文件：

- [connect.c](../connect.c)：连接和传输建立。
- [transport.c](../transport.c)：传输层抽象。
- [fetch-pack.c](../fetch-pack.c)：fetch 协商和 pack 获取。
- [send-pack.c](../send-pack.c)：push 发送端。
- [upload-pack.c](../upload-pack.c)：服务端 fetch/upload-pack。
- [receive-pack.c](../builtin/receive-pack.c)：服务端 push/receive-pack。
- [pkt-line.c](../pkt-line.c)：Git 协议 pkt-line 编解码。

网络部分建议最后阅读，因为它同时依赖引用、对象遍历、pack 和子进程管理等多个系统。

## 通用阅读方法

### 从命令入口反向建立范围

查找命令实现：

```sh
rg -n '^int cmd_[a-zA-Z0-9_]+\(' builtin
rg -n 'cmd_status|cmd_cat_file|cmd_rev_list' builtin git.c
```

### 用调用点限制阅读范围

例如研究 `repo_get_oid()`：

```sh
rg -n '\brepo_get_oid\s*\(' --glob '*.[ch]'
```

先阅读声明、实现和两三个代表性调用者，不必立刻阅读所有调用点。

### 将动态调试和静态索引结合

clangd 适合完成：

- 跳转到定义和声明。
- 查找引用。
- 查看宏展开后的类型信息。

LLDB 适合确认：

- 实际走到了哪个后端或分支。
- 结构体字段在真实仓库中的值。
- 子命令是在当前进程还是子进程执行。

### 记录版本边界

Git 的内部结构持续重构，尤其是对象数据库、引用后端和 Rust 接入。形成结论后建议记录：

```sh
git rev-parse HEAD
git log -1 --format='%h %ad %s' --date=short -- path/to/file
```

这样未来切换分支或版本时，可以用 `git log` 和 `git blame` 快速定位变化来源。
