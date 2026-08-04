# Git 中的 Rust

## 结论

旧版 Git 的普通 Make 构建确实不需要 Rust。当前源码是 Git 2.55.0，这是第一个默认启用 Rust 的版本，所以执行普通 `make` 时会运行 Cargo，并将 `libgitcore.a` 链接到 Git 可执行文件。

Git 并没有突然使用 Rust 重写整个项目。当前策略是从边界清楚的小模块开始，验证构建系统、C/Rust FFI 和平台工具链，再逐步把 Rust 用于新功能或适合迁移的子系统。

## 时间线

| 时间或版本 | 变化 |
| --- | --- |
| 2024 年 1 月 | Git 社区正式讨论引入 Rust，涉及内存安全、可维护性、贡献者生态和老平台支持。 |
| Git 2.49，2025-03-14 | `contrib/libgit-sys` 和 `contrib/libgit-rs` 进入仓库，提供围绕 `libgit.a` 的 Rust 包装；不参与普通 Git 主程序构建。 |
| Git 2.52，2025-11-17 | 建立内部 `gitcore` Rust 静态库，并用 `varint` 作为 C/Rust 互操作试验；Make 默认仍不启用。 |
| Git 2.54，2026-04-20 | 加入 Object ID、哈希算法抽象和 SHA-1/SHA-256 对象映射相关 Rust 基础设施。 |
| Git 2.55，2026-06-29 | Make 和 Meson 默认启用 Rust；仍可显式设置 `NO_RUST` 退出。 |
| Git 3.0 | 当前计划移除构建开关，使 Rust 成为强制依赖；尚无确定发布日期，官方也保留根据发行版影响调整计划的可能。 |

本地提交历史可以验证几个关键节点：

```sh
git show e7f8bf125c  # libgit-sys: introduce Rust wrapper for libgit.a
git show 8832e728d3  # varint: reimplement as test balloon for Rust
git show 40a1b4fb2b  # rust: add a new binary object map format
git show 32d5b90590  # Enable Rust by default
```

上游资料：

- [Git BreakingChanges](https://git-scm.com/docs/BreakingChanges.html)
- [2024 年 Introducing Rust 讨论](https://lore.kernel.org/git/ZZ77NQkSuiRxRDwt@nand.local/)
- [引入 Rust 的补丁系列说明](https://www.spinics.net/lists/git/msg509633.html)

## 当前构建结构

Rust crate 定义在 [Cargo.toml](../Cargo.toml)，名称为 `gitcore`，产物类型为静态库：

```toml
[lib]
crate-type = ["staticlib"]
```

Make 构建的大致关系是：

```text
Git 的 C 源码                       src/*.rs
     │                                 │
     ▼                                 ▼
 C 编译器                          cargo build
     │                                 │
     ▼                                 ▼
  libgit.a          target/{debug,release}/libgitcore.a
     │                                 │
     └──────────────┬──────────────────┘
                    ▼
          git / scalar / helper programs
```

当前 `Cargo.toml` 声明的最低 Rust 版本是 1.49.0，并且没有第三方 crate 依赖。这让引入阶段的供应链和平台适配保持得比较简单。

可以用下面的命令确认构建结果是否启用了 Rust：

```sh
./git version --build-options
```

对应代码位于 [help.c](../help.c)：启用时输出 `rust: enabled`，否则输出 `rust: disabled`。

## 已实际进入 C 调用路径的 Rust：varint

[src/varint.rs](../src/varint.rs) 通过稳定的 C ABI 导出两个符号：

```rust
#[no_mangle]
pub unsafe extern "C" fn decode_varint(...)

#[no_mangle]
pub unsafe extern "C" fn encode_varint(...)
```

C 侧通过 [varint.h](../varint.h) 调用它们：

```text
C 调用者
   │
   ▼
varint.h
   │
   ├── Rust 已启用 ──► src/varint.rs ──► libgitcore.a
   │
   └── NO_RUST ─────► varint.c
```

主要调用位置：

- [read-cache.c](../read-cache.c)：Git index v4 路径前缀压缩的读取和写入。
- [dir.c](../dir.c)：untracked cache 的变长整数序列化和反序列化。

可用以下命令查看完整调用点：

```sh
rg -n '\b(decode_varint|encode_varint)\s*\(' --glob '*.[ch]' .
```

选择 `varint` 不是因为它是 Git 的主要性能瓶颈，而是因为它规模小、没有复杂依赖，适合验证：

- Make 与 Cargo 的联合构建。
- C 调用 Rust 导出函数。
- 静态链接和符号可见性。
- Windows、macOS 和 Linux 平台兼容性。
- 发行版对 Rust 工具链的准备情况。

这也是上游所说的 test balloon，而不是大规模重写。

## SHA-1/SHA-256 互操作基础设施

当前 `gitcore` crate 还包含以下模块：

### `hash.rs`

[src/hash.rs](../src/hash.rs) 定义：

- `ObjectID`：同时保存最多 32 字节的哈希值和算法编号。
- `HashAlgorithm`：描述 SHA-1 和 SHA-256。
- `CryptoHasher`：通过 FFI 调用现有 C 哈希实现。
- 对象 ID 的显示、排序、比较和空对象常量。

这里没有重复实现 Git 的所有哈希算法，而是用 Rust 类型封装现有 C 能力。

### `csum_file.rs`

[src/csum_file.rs](../src/csum_file.rs) 把 Git 的 C `hashfile` API 包装为实现 `std::io::Write` 的 Rust `HashFile`。

这样 Rust 对象映射代码可以面向标准 `Write` 接口工作，而不必直接复制 Git 的文件打开、锁定和落盘逻辑。

### `loose.rs`

[src/loose.rs](../src/loose.rs) 实现 SHA-1 与 SHA-256 对象 ID 的双向映射，以及新的 `LMAP` 二进制对象映射格式。

其目标是支持 `extensions.compatObjectFormat`：仓库使用一种哈希算法作为主格式，同时保留另一种算法的兼容对象名。

需要注意当前实现状态：

- Rust 侧已经有对象 ID、哈希和二进制映射结构及测试。
- 当前 C 主程序直接调用的 Rust C ABI 仍主要是 `varint`。
- `loose.rs` 的更高层 C/FFI 接入尚未全部完成，当前树中仍能看到 C 版 loose object map 逻辑。
- 未启用 Rust 时，[repository.c](../repository.c) 会拒绝兼容哈希算法模式，为后续迁移明确建立 Rust 依赖边界。

因此，应把这些模块描述为“已经进入源码和构建的互操作基础设施”，而不是误认为 Git 2.55 已经用 Rust 接管了整个对象数据库。

## 为什么 Git 选择 Rust

从邮件列表和 BreakingChanges 中可以归纳出几个主要动机：

1. 利用 Rust 的内存和类型安全减少新代码中的常见 C 内存错误。
2. 为未来的新功能提供比 C 更严格的抽象边界。
3. 吸引熟悉现代系统语言的新贡献者。
4. 在保留现有 C 投资的同时，逐步建立可用的 C/Rust FFI。
5. 提前验证各发行版、老平台和交叉编译环境的工具链问题。

阻力也很实际：

- 部分老平台没有 Rust 工具链。
- 某些发行版需要解决 Rust 编译器的 bootstrap 顺序。
- 引入 Cargo 可能增加依赖和供应链管理成本。
- 同一模块长期维护 C/Rust 两份实现不可持续。

因此 Git 采用了“先可选、再默认、最终可能强制”的多阶段方案。

## 关闭 Rust 后会发生什么

Git 2.55 可以在 Make 中设置：

```make
NO_RUST = YesPlease
```

此时：

- 不构建 `libgitcore.a`。
- 编译 [varint.c](../varint.c) 作为替代。
- `git version --build-options` 显示 `rust: disabled`。
- `extensions.compatObjectFormat` 相关互操作功能会被拒绝。

这说明 Git 2.55 是“默认需要 Rust，但仍可退出”；计划中的 Git 3.0 才是“构建开关也被删除”。

## 后续跟踪方法

查看 Rust 文件的演进：

```sh
git log --oneline --all -- Cargo.toml build.rs src '*.rs'
```

查看 C/Rust 构建边界：

```sh
rg -n 'WITH_RUST|NO_RUST|RUST_LIB|RUST_SOURCES' Makefile
```

查看 Rust 对 C 的调用：

```sh
rg -n 'extern "C"' src
```

查看 Rust 导出给 C 的接口：

```sh
rg -n 'no_mangle|extern "C" fn' src
```

查看最终可执行文件中的相关符号：

```sh
nm -gU ./git | rg 'decode_varint|encode_varint'
```
