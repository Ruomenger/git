# Git 架构与核心概念导览

本文面向"命令用得久、内部原理没读过"的情况，目标不是罗列命令用法，而是回答：Git 到底把数据存成了什么、源码怎么分层、一条日常命令在代码里走了哪些系统。

所有结论基于当前工作区快照：

```text
版本：2.55.0
提交：e9019fcafe0040228b8631c30f97ae1adb61bcdc
平台：macOS arm64
```

规模参考：顶层 `*.c` / `*.h` 共 472 个文件、约 24.9 万行；`builtin/` 下 130 个命令实现文件；`src/` 下 11 个 Rust 文件。

## 一、先建立三个核心抽象

Git 的复杂功能几乎都是这三层的组合。理解它们之后，大部分"高级命令"会变成"在这三层上做一次特定操作"。

### 1. 对象库：内容寻址的不可变存储

Git 不存"文件的第 N 个版本"，只存**内容**，用内容的哈希当地址。对象类型定义在 [object.h](../object.h) 的 `enum object_type`（object.h:98）：

```c
OBJ_COMMIT = 1,
OBJ_TREE = 2,
OBJ_BLOB = 3,
OBJ_TAG = 4,
/* 5 for future expansion */
OBJ_OFS_DELTA = 6,
OBJ_REF_DELTA = 7,
```

前四种是真实对象；6 和 7 是 **pack 文件内部的 delta 编码**，不是独立的对象类型，解包后仍然是 commit/tree/blob/tag 之一。这个枚举值是 pack 格式的一部分，所以不能随意改动。

对象 ID 的表示见 [hash.h](../hash.h) 的 `struct object_id`（hash.h:212）：

```c
struct object_id {
	unsigned char hash[GIT_MAX_RAWSZ];
	uint32_t algo;	/* XXX requires 4-byte alignment */
};
```

注意这里带了 `algo` 字段——Git 已经为 SHA-1 / SHA-256 双算法做好了结构准备，但**默认仍是 SHA-1**（hash.h:179）：

```c
#ifdef WITH_BREAKING_CHANGES
# define GIT_HASH_DEFAULT GIT_HASH_SHA256
#else
# define GIT_HASH_DEFAULT GIT_HASH_SHA1
#endif
```

哈希怎么算出来的，可以直接验证。对象的存储格式是 `<类型> <字节数>\0<内容>`：

```sh
printf 'hello\n' | ./git hash-object --stdin
# ce013625030ba8dba906f756967f9e9ca394464a

printf 'blob 6\0hello\n' | shasum -a 1
# ce013625030ba8dba906f756967f9e9ca394464a  -
```

两者完全一致。这就是"内容寻址"的全部魔法：**同样的内容在任何仓库、任何时间都得到同一个 ID**，因此去重、完整性校验、分布式同步全部免费得到。

对象的物理存放有两种形态，可以在本仓库看到：

```sh
./git count-objects -v
# count: 18            ← 松散对象（.git/objects/xx/yyyy...）
# in-pack: 421340      ← pack 文件中的对象
# packs: 3
# size-pack: 329009    ← KiB
```

松散对象一个文件一个对象、zlib 压缩；pack 把大量对象打包并做 delta 压缩，是网络传输和长期存储的形态。42 万个对象压到约 321 MiB，靠的就是 delta + zlib。

**当前源码正在做的一件事**：把"对象从哪来"抽象成可插拔的 source。见 [odb/source.h](../odb/source.h)（odb/source.h:7）：

```c
enum odb_source_type {
	ODB_SOURCE_UNKNOWN,
	ODB_SOURCE_FILES,     /* 松散对象 + packfile */
	ODB_SOURCE_LOOSE,     /* 仅松散对象 */
	ODB_SOURCE_INMEMORY,  /* 内存中的对象 */
};
```

`struct odb_source` 带有 `read_object_info` / `read_object_stream` / `write_object` / `for_each_object` 等函数指针，是一套 vtable。这是较新的重构（`odb/` 目录 2026-06 仍在活跃改动），属于"已进入仓库但还在推进中"的基础设施——阅读时要区分它和已经稳定多年的 [packfile.c](../packfile.c)、[object-file.c](../object-file.c)。

### 2. 引用：给对象起可变的名字

对象 ID 不可变也不好记，所以需要一层"可变指针"：`refs/heads/master`、`refs/tags/v1.0`、`HEAD`。分支的本质就是**一个存着 40 位十六进制字符串的名字**，这也是为什么 Git 建分支这么快。

关键在于当前有**两套引用存储后端**，注册在 [refs.c](../refs.c)（refs.c:38）：

```c
static const struct ref_storage_be *refs_backends[] = {
	[REF_STORAGE_FORMAT_FILES] = &refs_be_files,
	[REF_STORAGE_FORMAT_REFTABLE] = &refs_be_reftable,
};
```

- `files` 后端（[refs/files-backend.c](../refs/files-backend.c)）：传统方案，一个 ref 一个文件，加上 `.git/packed-refs` 做批量压缩。本仓库用的就是它：

  ```sh
  ./git rev-parse --show-ref-format
  # files
  ```

- `reftable` 后端（[refs/reftable-backend.c](../refs/reftable-backend.c) + [reftable/](../reftable/)）：来自 JGit 的二进制格式，解决 files 后端在巨量 ref、大小写不敏感文件系统、原子批量更新上的固有问题。

两个后端通过统一的 `struct ref_storage_be` 接口接入，上层命令基本无感。**所有引用修改都走事务**（`ref_transaction_begin` / `ref_transaction_update` / `ref_transaction_commit`），这是 Git 保证"要么全改要么不改"的机制，后面 commit 流程里会看到。

引用还带 reflog——`.git/logs/` 记录了每个 ref 的历史取值。这是 `git reset --hard` 之后还能救回来的原因，而它和对象库的不可变性是配合关系：**对象删不掉，只是没有名字指向它了**。

### 3. 索引：工作区和对象库之间的那层缓存

`.git/index` 是最容易被忽略但最能解释 Git 行为的结构。定义在 [read-cache-ll.h](../read-cache-ll.h)（read-cache-ll.h:22）：

```c
struct cache_entry {
	struct hashmap_entry ent;
	struct stat_data ce_stat_data;   /* ctime/mtime/dev/ino/uid/gid/size */
	unsigned int ce_mode;
	unsigned int ce_flags;
	unsigned int mem_pool_allocated;
	unsigned int ce_namelen;
	unsigned int index;	/* for link extension */
	struct object_id oid;
	char name[FLEX_ARRAY]; /* more */
};
```

它同时承担两个角色：

1. **暂存区（staging area）**：`ce_flags` 中的 `CE_STAGEMASK` 保存 stage 号，合并冲突时同一路径会有 stage 1/2/3 三条记录（base/ours/theirs）。这就是冲突时 `git ls-files -u` 能看到的东西。
2. **stat 缓存**：`ce_stat_data` 缓存了每个文件的 inode/mtime/size。`git status` 之所以不用读取每个文件内容重算哈希，是因为它先比 stat；stat 没变就认为文件没变。相关的边界情况见 [Documentation/technical/racy-git.adoc](../Documentation/technical/racy-git.adoc)。

实际查看：

```sh
./git ls-files --stage | head -3
# 100644 fef04a38402fee6465a6a4225374d493b47421c0 0	.cirrus.yml

./git ls-files --debug | head -8   # 能看到 ctime/mtime/dev/ino/size
```

索引格式版本范围是 2~4（read-cache-ll.h:19）：

```c
#define INDEX_FORMAT_LB 2
#define INDEX_FORMAT_UB 4
```

v4 用 varint 做路径前缀压缩——这正是 Rust 已经接管的部分，见 [rust-in-git.md](rust-in-git.md)。

索引还挂了一堆扩展段（[read-cache.c](../read-cache.c):69）：

| 扩展 | 用途 |
| --- | --- |
| `TREE` | cache-tree，缓存目录对应的 tree 对象，让 `git commit` 不必重算整棵树 |
| `REUC` | resolve-undo，支持 `git checkout -m` 重新触发冲突 |
| `UNTR` | untracked cache，加速 `git status` 对未跟踪文件的扫描 |
| `FSMN` | fsmonitor，配合文件系统监视进程进一步跳过扫描 |
| `EOIE` / `IEOT` | 支持多线程并行读取索引 |
| `sdir` | sparse directory，sparse index 的核心 |

**这张表基本就是 `git status` 性能优化史。**

### 三者的关系

```text
对象库 (.git/objects)      不可变、内容寻址、只增不减
      ▲
      │ 引用指向 commit
引用   (.git/refs, packed-refs, reftable)   可变的名字
      ▲
      │ commit 指向 tree
索引   (.git/index)         下一次 commit 的草稿 + stat 缓存
      ▲
工作区  实际的文件
```

`git diff` 比工作区和索引，`git diff --cached` 比索引和 HEAD，`git diff HEAD` 比工作区和 HEAD——三条命令的差别就是选了这个梯子上的哪两级。

## 二、把它们串起来：`git commit` 走过的路

以 [builtin/commit.c](../builtin/commit.c) 为例，主干调用是：

```text
cmd_commit()                          builtin/commit.c
  ├── repo_read_index()               commit.c:1629    读 .git/index 进内存
  ├── prepare_to_commit()             commit.c:1842
  │     ├── cache_tree_update()       commit.c:1111    由索引构建 tree 对象并写入对象库
  │     └── run_commit_hook()         commit.c:1133    prepare-commit-msg / commit-msg
  ├── commit_tree_extended()          commit.c:1938 → commit.c:1736
  │                                                    写 commit 对象（指向 tree + parent）
  └── update_head_with_reflog()       commit.c:1945 → sequencer.c:1259
        └── ref_store_transaction_begin()
            ref_transaction_update("HEAD", ...)
            ref_transaction_commit()                   事务性更新 HEAD 与 reflog
```

值得注意的几点：

1. **tree 对象是由索引生成的，不是由工作区生成的**。这就是"没 `git add` 就不会被提交"的机制层原因。
2. `cache_tree_update()` 增量利用索引里的 `TREE` 扩展，只重算被改动的目录。
3. 更新分支指针走的是 ref transaction，不是简单写文件——所以中断不会留下半更新状态，同时 reflog 也在同一事务里写入。
4. `update_head_with_reflog()` 定义在 [sequencer.c](../sequencer.c) 而不是 refs.c，因为 commit / cherry-pick / rebase / revert 共享这段逻辑。[sequencer.c](../sequencer.c) 有 6917 行，是 rebase、cherry-pick、revert、`--continue`/`--abort` 状态机的共同实现。

## 三、源码分层地图

| 层 | 位置 | 职责 |
| --- | --- | --- |
| 进程入口 | [common-main.c](../common-main.c)、[common-init.c](../common-init.c) | 所有 C 程序共享的 `main()` 与进程初始化 |
| 命令分发 | [git.c](../git.c)、[command-list.txt](../command-list.txt) | `commands[]` 表（约 149 条）、alias 展开、外部命令回退 |
| 命令实现 | `builtin/*.c`（130 个文件） | 每个 `cmd_*()` 是一条命令的入口 |
| 核心子系统 | `odb/`、`refs/`、[read-cache.c](../read-cache.c)、[revision.c](../revision.c)、[diff.c](../diff.c)、[merge-ort.c](../merge-ort.c)、[transport.c](../transport.c) | 对象库、引用、索引、遍历、差异、合并、传输 |
| 算法库 | [xdiff/](../xdiff)、[ewah/](../ewah)、[reftable/](../reftable)、[sha1dc/](../sha1dc) | 文本 diff、位图、reftable、SHA-1 碰撞检测 |
| Rust 组件 | [src/](../src)（`varint.rs`、`hash.rs`、`csum_file.rs`、`loose.rs`） | 通过 C ABI 接入，详见 [rust-in-git.md](rust-in-git.md) |
| 通用数据结构 | [strbuf.h](../strbuf.h)、[string-list.h](../string-list.h)、[hashmap.h](../hashmap.h)、[strmap.h](../strmap.h)、[strvec.h](../strvec.h)、[oidset.h](../oidset.h)、[mem-pool.h](../mem-pool.h)、[prio-queue.h](../prio-queue.h) | Git 自己的"标准库"，读任何模块前都会撞上 |
| 进程与可观测性 | [run-command.c](../run-command.c)、[trace2/](../trace2) | 子进程管理、结构化 trace |
| 平台兼容 | [compat/](../compat) | 各平台差异垫片 |

`struct repository`（[repository.h](../repository.h)）是把这些串起来的容器：

```c
char *gitdir;                        /* repository.h:47  */
char *commondir;                     /* repository.h:53  */
struct object_database *objects;     /* repository.h:58  */
struct ref_store *refs_private;      /* repository.h:74  */
struct index_state *index;           /* repository.h:144 */
char *worktree;                      /* repository.h:116 */
```

近年的大方向重构就是**消除全局变量、把状态收进 `struct repository`**（历史上是 `the_repository` 一个全局单例）。读旧文章时看到的 `the_index`、`the_repository` 直接引用，在当前源码里很多已经改成显式传参。

命令注册时会声明自己需要什么初始化（[git.c](../git.c):21）：

```c
#define RUN_SETUP		(1<<0)   /* 必须在仓库里 */
#define RUN_SETUP_GENTLY	(1<<1)   /* 不在仓库里也能跑 */
#define NEED_WORK_TREE		(1<<3)   /* 必须有工作区，裸仓库不行 */
#define DELAY_PAGER_CONFIG	(1<<4)
#define NO_PARSEOPT		(1<<5)
```

所以"为什么 `git status` 在裸仓库里报错、`git init` 在哪都能跑"，答案就写在这张表里。命令分发的完整机制见 [command-dispatch-and-process-model.md](command-dispatch-and-process-model.md)。

## 四、porcelain 与 plumbing

Git 官方把命令分成两类，分类数据在 [command-list.txt](../command-list.txt)：

| 类别 | 数量 | 含义 |
| --- | --- | --- |
| `mainporcelain` | 46 | 日常用户命令（`commit`、`log`、`merge`…） |
| `plumbinginterrogators` | 25 | 底层查询（`cat-file`、`rev-list`、`ls-files`…） |
| `plumbingmanipulators` | 20 | 底层修改（`update-ref`、`hash-object`、`write-tree`…） |
| `ancillaryinterrogators` | 17 | 辅助查询（`blame`、`whatchanged`…） |
| `purehelpers` | 19 | 纯助手（`upload-pack`、`receive-pack`…） |
| `ancillarymanipulators` | 12 | 辅助修改（`config`、`gc`、`remote`…） |
| `foreignscminterface` | 10 | 与其他 SCM 互通（`svn`、`p4`、`fast-import`…） |

**这个区分对读源码非常重要**：porcelain 命令的输出格式会变、面向人；plumbing 命令的输出格式是稳定契约、面向脚本。学习内部原理时应该优先用 plumbing，因为它们的实现更薄，更接近数据结构本身：

```sh
./git cat-file -t HEAD          # commit
./git cat-file -p HEAD          # 看 commit 对象的原始内容
./git cat-file -p HEAD^{tree}   # 看 tree 的原始内容
./git rev-list --parents HEAD -5
./git ls-files --stage
./git show-ref
```

例如 `./git cat-file -p HEAD` 的输出就是 commit 对象在磁盘上的真实文本：

```text
tree 7385aa8dbd9b1de970ea9d6ff4c76306ab104909
parent fae97fc383c7ff11cbdd9167e969823a9d898719
author Ruomenger <ruomenger@foxmail.com> 1785834963 +0800
committer Ruomenger <ruomenger@foxmail.com> 1785834963 +0800

docs: 记录 Git 命令分发与进程模型
...
```

没有任何"diff"，没有任何"变更集"——**commit 存的是完整快照的树根**，diff 是 Git 在读取时现算的。这一点和 SVN、Perforce 的心智模型完全不同，也是理解 rebase / cherry-pick / merge 的前提。

## 五、常被忽略但值得了解的能力

按"日常命令之外"排列，附上源码位置，方便按需深入：

| 能力 | 一句话说明 | 源码 |
| --- | --- | --- |
| `git worktree` | 一个仓库同时挂多个工作区，共享对象库 | [worktree.c](../worktree.c)、[builtin/worktree.c](../builtin/worktree.c) |
| `git rerere` | 记录冲突解法并自动重放，长期 rebase 分支的利器 | [rerere.c](../rerere.c) |
| sparse-checkout / sparse index | 只检出部分目录；索引中用 `sdir` 条目代表整个未展开目录 | [builtin/sparse-checkout.c](../builtin/sparse-checkout.c)、[sparse-index.c](../sparse-index.c) |
| partial clone | 按需惰性获取 blob（`--filter=blob:none`） | [promisor-remote.c](../promisor-remote.c) |
| commit-graph | 预计算提交图与 generation number，加速可达性判断 | [commit-graph.c](../commit-graph.c) |
| multi-pack-index | 跨多个 pack 的统一对象索引 | [midx.c](../midx.c) |
| 可达性位图 | 用 EWAH 位图加速 `rev-list` 与 pack 生成 | [pack-bitmap.c](../pack-bitmap.c)、[ewah/](../ewah) |
| `git maintenance` | 后台增量维护，取代粗放的 `gc` | [builtin/gc.c](../builtin/gc.c) |
| fsmonitor | 与文件系统监视守护进程协作跳过全量扫描 | [fsmonitor.c](../fsmonitor.c) |
| `git notes` | 给已有 commit 附加信息而不改变其哈希 | [notes.c](../notes.c) |
| `git replace` | 在读取层"伪装"对象替换，不重写历史 | [replace-object.c](../replace-object.c) |
| bundle / bundle-uri | 把仓库打包成单文件传输，或用 CDN 加速克隆 | [bundle-uri.c](../bundle-uri.c) |
| gitattributes / 内容过滤 | `text`、`diff=` 驱动、clean/smudge filter | [attr.c](../attr.c)、[convert.c](../convert.c) |
| merge-ort | 当前默认合并策略，取代 recursive | [merge-ort.c](../merge-ort.c) |
| `git bisect` | 二分查找引入问题的提交，支持 `run` 自动化 | [bisect.c](../bisect.c) |
| trace2 | 结构化性能与事件日志，调试性能问题的正规入口 | [trace2/](../trace2)、[Documentation/technical/api-trace2.adoc](../Documentation/technical/api-trace2.adoc) |

快速体验 trace2（比 `GIT_TRACE=1` 信息量大得多）：

```sh
GIT_TRACE2_PERF=1 ./git status >/dev/null
```

## 六、当前基线（2.55）的时代特征

读源码时要意识到这个版本处在几个迁移的中途，容易和网上旧资料冲突：

1. **Rust 首次默认启用**。`./git version --build-options` 显示 `rust: enabled`。可用 `NO_RUST = YesPlease` 退出；Git 3.0 计划变为强制依赖。详见 [rust-in-git.md](rust-in-git.md)。
2. **reftable 已是一等后端**，但默认仍是 files（本仓库 `--show-ref-format` 输出 `files`）。
3. **SHA-256 已全面支持但非默认**，切换由 `WITH_BREAKING_CHANGES` 控制。参考 [Documentation/technical/hash-function-transition.adoc](../Documentation/technical/hash-function-transition.adoc)。
4. **对象库正在被重构为 source 抽象**（`odb/` 目录），旧代码路径（`object-file.c`、`packfile.c`）与新抽象并存。
5. **2.55 新增了几个 builtin**：`git format-rev`（`cmd_format_rev` 在 [builtin/name-rev.c](../builtin/name-rev.c):811，与 `name-rev` 共用实现）、`git history`（[builtin/history.c](../builtin/history.c)）、`git url-parse`（[builtin/url-parse.c](../builtin/url-parse.c)）。这也是一个例子：**builtin 文件数（130）少于 `commands[]` 条目数（149）**，因为一个文件可以注册多条命令。完整清单见 [Documentation/RelNotes/2.55.0.adoc](../Documentation/RelNotes/2.55.0.adoc)。

## 七、配套的一手资料

Git 自带的技术文档质量很高，且和源码同步更新，优先于外部博客：

- [Documentation/gitcore-tutorial.adoc](../Documentation/gitcore-tutorial.adoc)：官方的 plumbing 入门，和本文第一节高度互补。
- [Documentation/gitformat-loose.adoc](../Documentation/gitformat-loose.adoc)、[Documentation/gitformat-pack.adoc](../Documentation/gitformat-pack.adoc)、[Documentation/gitformat-index.adoc](../Documentation/gitformat-index.adoc)：三种磁盘格式的规范。
- [Documentation/technical/](../Documentation/technical)：reftable、sparse-index、partial-clone、commit-graph、multi-pack-index、racy-git 等专题。

## 八、建议的下一步

1. 先按 [source-reading-roadmap.md](source-reading-roadmap.md) 的第一、二阶段跑通"命令如何进入 `cmd_*()`"。
2. 用 [debug-build.md](debug-build.md) 的 LLDB 配置，在 `cmd_commit`、`cache_tree_update`、`commit_tree_extended`、`ref_transaction_commit` 四个点各下一个断点，跑一次真实提交，观察本文第二节的链路。
3. 之后再按兴趣挑一个方向深入：索引与 `status` 性能、对象库与 pack、引用与 reftable、revision 遍历、merge-ort。

不建议一开始就同时研究 delta 编码、commit-graph、midx 和 partial clone——它们都是在上述三个核心抽象之上的优化层，脱离基础读会很快失去上下文。
