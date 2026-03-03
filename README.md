# RFL Camp

> Rust-for-Linux 中文社区 — 从零到第一个内核补丁的最短路径

**[在线阅读](https://rfl-camp.github.io/guide/)**

## 这是什么？

RFL Camp 帮助中文开发者快速融入 [Rust-for-Linux](https://rust-for-linux.com) 社区。我们提供：

- **环境搭建指南** — 跳过所有踩坑，一天搭好开发环境
- **结构化学习路径** — 从内核基础到 Rust 驱动开发，循序渐进
- **贡献工作流** — 从第一个补丁到被合并的完整指南
- **社区动态** — RFL 上游进展的中文解读

## 快速开始

### 1. 搭建开发环境

> [环境搭建完整指南](docs/env-setup-guide.md)

你需要一台 x86_64 Linux 机器（物理机、VM 或远程服务器），推荐 Fedora 或 Ubuntu。

```bash
# 安装依赖（Fedora）
sudo dnf install -y git make gcc flex bison bc kmod cpio \
  ncurses-devel openssl-devel dwarves elfutils-libelf-devel \
  clang lld llvm rsync zstd

# 克隆 RFL 内核源码
git clone --depth=1 https://github.com/Rust-for-Linux/linux.git ~/linux

# 安装 Rust 工具链
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
rustup component add rust-src rustfmt clippy
cargo install --locked bindgen-cli

# 验证
cd ~/linux && make LLVM=1 rustavailable
# 输出 "Rust is available!" 即成功
```

### 2. 编译并测试内核

```bash
cd ~/linux

# 配置
make LLVM=1 defconfig
scripts/config --enable CONFIG_RUST
scripts/config --enable CONFIG_SAMPLES --enable CONFIG_SAMPLES_RUST \
  --enable CONFIG_SAMPLE_RUST_MINIMAL --enable CONFIG_SAMPLE_RUST_PRINT
make LLVM=1 olddefconfig

# 编译
make LLVM=1 -j$(nproc)

# 测试（需要 QEMU + virtme-ng）
pip3 install virtme-ng
vng -r arch/x86/boot/bzImage --force-initramfs \
  --exec 'dmesg | grep rust'
```

看到 `rust_minimal: Rust minimal sample (init)` 就说明 Rust 在内核中跑起来了。

### 3. 学习路径

| 阶段 | 模块 | 内容 |
|------|------|------|
| **内核基础** | [内核架构概览](docs/kernel-basics-01.md) | 宏内核、五大子系统、系统调用 |
| | [构建系统](docs/kernel-basics-02.md) | Kbuild、Kconfig、.config |
| | [数据结构](docs/kernel-basics-03.md) | list_head、kref、container_of |
| **RFL 架构** | [Rust 与内核](docs/rfl-arch-01.md) | bindgen、安全抽象、rust/kernel/ |
| | [核心抽象](docs/rfl-arch-02.md) | Module、Error、pin_init、Arc |
| **并发** | [锁原语](docs/rfl-concurrency-01.md) | Mutex、SpinLock、Guard |
| **内存** | [内核分配](docs/rfl-memory-01.md) | fallible allocation、GFP flags |
| **驱动** | [第一个模块](docs/rfl-drivers-01.md) | 编写、编译、加载 Rust 内核模块 |
| **贡献** | [贡献工作流](docs/contrib-01.md) | git format-patch、send-email |
| | [RFL 规范](docs/contrib-02.md) | SAFETY 注释、no unwrap、fallible alloc |

### 4. 提交你的第一个补丁

> [补丁提交完整指南](docs/patch-guide.md)

## 资源链接

| 资源 | 链接 |
|------|------|
| RFL 上游仓库 | https://github.com/Rust-for-Linux/linux |
| 内核文档（Rust） | https://docs.kernel.org/rust/index.html |
| 邮件列表 | rust-for-linux@vger.kernel.org |
| Zulip 聊天 | https://rust-for-linux.zulipchat.com |
| RFL 官网 | https://rust-for-linux.com |

## 参与贡献

RFL Camp 本身也是开源项目，欢迎贡献：

- 完善文档和教程
- 翻译上游文档
- 分享你的学习笔记
- 报告错误或过时信息

## License

CC BY-SA 4.0
