# Git 命令分发与进程模型

本文解释一个从构建产物中很容易产生的疑问：Git 是否采用多进程架构，以及为什么源码目录中会生成大量 `git-*` 可执行文件。

## 结论

Git 不是“每个子命令各有一份独立实现”的纯多进程架构。更准确地说，它采用混合方式：

1. 大部分核心子命令是编译进 `git` 主程序的 builtin，在当前 Git 进程内直接调用对应的 `cmd_*()` 函数。
2. 构建目录中的许多 `git-*` 文件只是 `git` 主程序的硬链接；硬链接不可用时，Makefile 才退化为符号链接或文件复制。
3. 少量辅助工具是真正单独编译的可执行程序，另有一些命令由 Shell、Perl 或 Python 脚本实现。
4. builtin 在执行过程中仍可能按需启动 transport、hook、pager、editor、credential helper 等子进程。

因此，“Git 会使用多个进程”和“每个 Git 子命令都是独立程序”是两个不同的问题。前者在许多操作中成立，后者对大部分核心命令并不成立。

## 三类 `git-*` 文件

当前构建目录中的命令大致可以分成三类：

| 类型 | 示例 | 实际实现 |
| --- | --- | --- |
| builtin 的兼容入口 | `git-status`、`git-diff`、`git-fetch`、`git-upload-pack` | 与 `git` 指向同一个 inode，代码已经链接进 `git` |
| 独立可执行程序 | `git-shell`、`git-http-backend`、`git-remote-http` | 拥有独立 inode 和程序映像 |
| 生成的脚本 | `git-submodule`、`git-filter-branch` | Shell、Perl 或 Python 脚本 |

可以直接检查文件类型和 inode：

```sh
ls -li git git-diff git-fetch git-status git-upload-pack \
  git-shell git-http-backend git-remote-http git-submodule

file git git-status git-shell git-http-backend git-submodule
```

本地构建结果中，`git`、`git-diff`、`git-fetch`、`git-status` 和 `git-upload-pack` 的 inode 相同，并共享同一个硬链接计数。这表示它们只是同一份文件内容的多个目录项，不会各自占用一份完整的可执行文件空间。

`git-shell` 和 `git-http-backend` 则拥有不同的 inode；`git-submodule` 会被 `file` 识别为 POSIX shell script。

## `git status` 如何在当前进程内执行

执行以下命令时：

```sh
./git status
```

Shell 首先创建进程并运行 `./git`。进入 Git 后，主要分发路径是：

```text
main()                         common-main.c
  └── cmd_main()               git.c
        └── run_argv()
              └── handle_builtin()
                    └── run_builtin()
                          └── cmd_status()
```

[git.c](../git.c) 中的 `commands[]` 表把命令名映射到函数及其仓库初始化选项，例如：

```c
{ "diff", cmd_diff, NO_PARSEOPT },
{ "fetch", cmd_fetch, RUN_SETUP },
{ "status", cmd_status, RUN_SETUP | NEED_WORK_TREE },
```

`handle_builtin()` 通过 `get_builtin()` 查表，找到命令后将其交给 `run_builtin()`。后者最终直接调用函数指针 `p->fn(...)`，没有为了执行 `status` 再启动一个 `git-status` 进程。

可以用 trace 验证这条路径：

```sh
GIT_TRACE=1 ./git status --short
```

输出包含：

```text
trace: built-in: git status --short
```

这里仍然存在 Shell 启动 `git` 的进程边界；“当前进程内执行”指的是 `git` 不会为了 builtin 再派生第二个命令进程。

## 为什么还要生成 `git-status`

[Makefile](../Makefile) 为所有 `BUILT_INS` 使用同一条规则：

```make
$(BUILT_INS): git$X
	$(QUIET_BUILT_IN)$(RM) $@ && \
	ln $< $@ 2>/dev/null || \
	ln -s $< $@ 2>/dev/null || \
	cp $< $@
```

它依次尝试：

1. 为 `git` 创建硬链接。
2. 硬链接不可用时创建符号链接。
3. 两种链接都不可用时复制文件。

这些 `git-*` 名称保留了传统的 dashed command 调用方式，也方便 Git 自带脚本和内部流程通过统一命名约定寻找命令。

直接执行：

```sh
./git-status --short
```

操作系统会启动一个进程，但加载的仍是与 `./git` 相同的程序映像。[git.c](../git.c) 的 `cmd_main()` 会查看 `argv[0]`：如果程序名以 `git-` 开头，就去掉该前缀，并把剩余的 `status` 直接交给 builtin 分发器。

因此，下面两种调用最终都会进入 `cmd_status()`：

```text
./git status                   ./git-status
     │                              │
     └──────► git 主程序 ◄─────────┘
                    │
                    ▼
               cmd_status()
```

两者的主要差异是参数入口形式：`git status` 可以在 `git` 和 `status` 之间放置顶层选项，而 `git-status` 没有这个位置。

## 非 builtin 命令如何执行

`cmd_main()` 会先通过 `setup_path()` 设置命令搜索路径，然后由 `run_argv()` 按大致顺序处理：

```text
命令名
  ├── alias 展开
  ├── 在 commands[] 中找到 ──► 当前进程调用 builtin
  └── 未找到 ─────────────────► 查找并启动 git-<命令>
```

外部命令由 `execv_dashed_external()` 构造 `git-<命令>` 名称，再通过 `run_command()` 启动子进程。例如 PATH 或 Git exec path 中存在 `git-example` 时：

```sh
git example arg1
```

可以转化为对下列外部命令的查找和执行：

```sh
git-example arg1
```

命令搜索会考虑 `--exec-path`、`GIT_EXEC_PATH`、构建时配置的 `gitexecdir` 和 PATH。当前默认 exec path 可以这样查看：

```sh
./git --exec-path
```

这套机制让核心命令可以逐步 builtin 化，同时继续支持脚本、辅助程序和用户自行安装的 `git-*` 扩展命令。

## 真正独立的程序和脚本

Makefile 用不同变量区分产物。

`PROGRAM_OBJS` 中的对象会生成独立程序，当前包括：

```make
PROGRAM_OBJS += daemon.o
PROGRAM_OBJS += http-backend.o
PROGRAM_OBJS += imap-send.o
PROGRAM_OBJS += sh-i18n--envsubst.o
PROGRAM_OBJS += shell.o
```

对应产物包括 `git-daemon`、`git-http-backend`、`git-imap-send` 和 `git-shell` 等。这类程序虽然也会链接 Git 的公共库，但不是 `git` 文件的硬链接。

Makefile 还维护 `SCRIPT_SH`、`SCRIPT_PERL` 和 `SCRIPT_PYTHON`。例如源码中的 `git-submodule.sh` 会在构建时生成可直接执行的 `git-submodule`。

独立程序存在的常见原因包括：

- 需要被远端协议或服务器环境直接调用，例如 `git-http-backend`。
- 具有与普通客户端命令不同的进程入口或链接依赖。
- 仍以脚本形式维护，尚未转换为 builtin。
- 属于 remote helper、credential helper 或用户扩展一类的可插拔接口。

## builtin 仍然可能产生子进程

builtin 描述的是“命令入口在 `git` 主程序内部”，不代表它的整个执行过程只有一个进程。例如，根据协议和配置，`git fetch` 可能需要启动：

```text
git fetch
  ├── ssh 或 git-remote-http
  ├── 远端 git-upload-pack
  ├── pack/index 处理程序
  └── credential helper
```

其他常见子进程还包括：

- pager，例如 `less`。
- 用户配置的 editor。
- hook 脚本。
- 外部 diff 或 merge driver。
- credential 和 remote helper。
- 后台 maintenance 或 fsmonitor 相关程序。

[run-command.c](../run-command.c) 和 [run-command.h](../run-command.h) 提供了 Git 通用的子进程启动与管理设施。具体命令是否产生子进程，要根据执行路径、协议和配置判断，不能只看命令本身是不是 builtin。

## 阅读和调试入口

建议按以下顺序验证命令分发：

1. 阅读 [common-main.c](../common-main.c) 的 `main()`。
2. 阅读 [git.c](../git.c) 的 `cmd_main()`、`run_argv()` 和 `handle_builtin()`。
3. 在 `commands[]` 中找到目标命令对应的 `cmd_*()` 函数。
4. 检查 Makefile 的 `BUILT_INS`、`PROGRAMS` 和 `SCRIPT_*` 分类。
5. 使用 `GIT_TRACE=1` 观察 builtin 和外部命令的运行边界。

以 `status` 为例，可以设置以下 LLDB 断点：

```text
(lldb) breakpoint set --name main
(lldb) breakpoint set --name cmd_main
(lldb) breakpoint set --name handle_builtin
(lldb) breakpoint set --name run_builtin
(lldb) breakpoint set --name cmd_status
(lldb) run
```

如果要研究子进程边界，可以继续关注 `run_command()`、`start_command()` 以及 trace 输出中的 `exec`、`run_command` 和 child 事件。
