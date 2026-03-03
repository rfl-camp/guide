# 内核构建系统

> 模块：kernel-basics-02 | 难度：入门 | 预计时间：2 小时
> 前置要求：[内核架构概览](kernel-basics-01.md)

---

## 目录

1. [为什么需要专用构建系统](#1-为什么需要专用构建系统)
2. [Kconfig：内核配置系统](#2-kconfig内核配置系统)
3. [Kbuild：内核构建引擎](#3-kbuild内核构建引擎)
4. [.config 文件](#4-config-文件)
5. [配置命令对比](#5-配置命令对比)
6. [CONFIG_RUST 的启用](#6-config_rust-的启用)
7. [Rust 在 Kbuild 中的编译流程](#7-rust-在-kbuild-中的编译流程)
8. [练习](#8-练习)
9. [知识检查](#9-知识检查)
10. [参考资源](#10-参考资源)

---

## 1. 为什么需要专用构建系统

Linux 内核有 **3000 万+** 行代码、**2 万+** 个可配置选项、支持 **20+** 种 CPU 架构。普通的 Makefile 或 CMake 无法管理这种规模。

内核使用两套协作系统：

| 系统 | 职责 | 类比 |
|------|------|------|
| **Kconfig** | 决定**编译什么** | 菜单 / 订单 |
| **Kbuild** | 决定**怎么编译** | 厨房 / 厨师 |

```
用户选择配置         配置决定编译目标       编译输出
┌──────────┐       ┌──────────────┐       ┌──────────┐
│ Kconfig  │ ────► │   .config    │ ────► │  Kbuild  │ ────► bzImage
│ 菜单系统  │       │ CONFIG_XX=y  │       │ Makefile │       + 模块
└──────────┘       └──────────────┘       └──────────┘
```

---

## 2. Kconfig：内核配置系统

### 2.1 配置类型

每个内核选项有三种可能的值：

| 值 | 含义 | 说明 |
|----|------|------|
| `y` | 内置 (built-in) | 编译进 bzImage，启动即加载 |
| `m` | 模块 (module) | 编译为 `.ko` 文件，按需加载 |
| `n` | 不编译 | 完全排除 |

> **bool 和 tristate 的区别**：`bool` 类型只能选 `y` 或 `n`，`tristate` 可以选 `y`、`m` 或 `n`。有些功能（如核心调度器）必须内置，不能做成模块，所以用 `bool`。

### 2.2 Kconfig 语法

每个子目录都有一个 `Kconfig` 文件，定义该目录下的配置选项。以 Rust 支持为例：

```kconfig
# init/Kconfig（简化）

config RUST
    bool "Rust support"
    depends on HAVE_RUST
    depends on RUST_IS_AVAILABLE
    depends on !MODVERSIONS
    depends on !GCC_PLUGINS
    depends on !RANDSTRUCT
    default n
    help
      Enables Rust support in the kernel. This allows writing kernel
      modules and other components in Rust.

      If unsure, say N.
```

**关键语法**：

| 关键字 | 作用 | 示例 |
|--------|------|------|
| `config` | 定义一个配置选项 | `config RUST` → 产生 `CONFIG_RUST` |
| `bool` / `tristate` | 值类型 | `bool "Rust support"` |
| `depends on` | 依赖条件 | `depends on HAVE_RUST` → 只有 HAVE_RUST=y 时才可见 |
| `select` | 反向依赖（自动开启） | `select CONSTRUCTORS` → 选中 RUST 时自动开启 CONSTRUCTORS |
| `default` | 默认值 | `default n` |
| `help` | 帮助文本 | 在 menuconfig 中按 `?` 查看 |

### 2.3 depends on vs select

```
depends on：我需要你（被动等待）
    CONFIG_RUST depends on HAVE_RUST
    → HAVE_RUST 没开？RUST 选项直接隐藏，用户看不到

select：我帮你开（主动强制）
    CONFIG_RUST select CONSTRUCTORS
    → 选中 RUST 后，自动把 CONSTRUCTORS 设为 y
```

> **踩坑**：`select` 是强制行为，会无视目标的 `depends on`，可能导致配置不一致。内核社区倾向于**少用 select**，因为它绕过了依赖检查。

### 2.4 menuconfig 界面

```bash
make menuconfig
```

```
┌───────────────── Linux/x86 6.x Kernel Configuration ──────────────────┐
│                                                                         │
│  General setup  --->                                                    │
│  [*] 64-bit kernel                                                      │
│  Processor type and features  --->                                      │
│  [*] Rust support                              ← bool 选项，[*]=y [ ]=n │
│  <M> XFS filesystem support                    ← tristate，<M>=m <*>=y  │
│  < > Btrfs filesystem support                  ← tristate，< >=n        │
│  Device Drivers  --->                                                   │
│                                                                         │
│         <Select>    < Exit >    < Help >    < Save >    < Load >        │
└─────────────────────────────────────────────────────────────────────────┘
```

快捷键：
- `空格` / `回车`：切换选项
- `/`：搜索配置项
- `?`：查看帮助（shows depends on、selected by 信息）

---

## 3. Kbuild：内核构建引擎

### 3.1 核心变量

Kbuild 使用分布在各子目录的 `Makefile` 来控制编译。核心机制是三个变量：

```makefile
# drivers/net/phy/Makefile

obj-$(CONFIG_PHYLIB)     += libphy.o       # 看 CONFIG_PHYLIB 的值
obj-$(CONFIG_RUST_PHYLIB_ABSTRACTIONS) += abstractions.o
```

**变量替换机制**：

| CONFIG 值 | `$(CONFIG_XXX)` 替换为 | 变量变成 | 效果 |
|-----------|----------------------|----------|------|
| `y` | `y` | `obj-y += foo.o` | 编译并链接进内核 |
| `m` | `m` | `obj-m += foo.o` | 编译为 `foo.ko` 模块 |
| `n` 或未定义 | `n` 或空 | `obj-n += foo.o` 或 `obj- += foo.o` | **不编译** |

这就是 Kconfig 和 Kbuild 的连接点——`.config` 中的 `CONFIG_XXX=y/m` 决定了 Makefile 中哪些文件被编译。

### 3.2 完整示例

假设 `.config` 中有：
```
CONFIG_EXT4_FS=y
CONFIG_BTRFS_FS=m
CONFIG_XFS_FS=n
```

对应 `fs/Makefile`（简化）中：
```makefile
obj-$(CONFIG_EXT4_FS)   += ext4/       # → obj-y += ext4/   → 编译并内置
obj-$(CONFIG_BTRFS_FS)  += btrfs/      # → obj-m += btrfs/  → 编译为模块
obj-$(CONFIG_XFS_FS)    += xfs/        # → obj-n += xfs/    → 不编译
```

### 3.3 目录递归

当 `obj-y` 或 `obj-m` 指向一个**目录**时，Kbuild 会进入该目录继续处理其 `Makefile`：

```makefile
# 顶层 Makefile
obj-y += kernel/
obj-y += mm/
obj-y += fs/
obj-y += drivers/
obj-y += net/
```

每个子目录的 Makefile 继续用 `obj-y/m` 指定编译目标，形成树状递归。

---

## 4. .config 文件

`.config` 是内核配置的最终产物，位于源码根目录。它是一个纯文本文件：

```bash
# .config 片段
#
# General setup
#
CONFIG_LOCALVERSION=""
CONFIG_DEFAULT_HOSTNAME="(none)"
CONFIG_SYSVIPC=y
CONFIG_POSIX_MQUEUE=y

#
# Rust support
#
CONFIG_RUST_IS_AVAILABLE=y
CONFIG_RUST=y

# CONFIG_XFS_FS is not set        ← 这行表示 CONFIG_XFS_FS=n
```

> **注意**：`.config` 被 `.gitignore` 忽略，不会提交到版本控制。它是**本机特有**的配置，不同机器的硬件和需求不同。

### .config 的生命周期

```
defconfig ─── 生成 ──► .config ─── 用户修改 ──► .config'
                                                    │
                          make olddefconfig ◄────────┘
                              │
                              ▼
                          .config'' （补全新增选项的默认值）
                              │
                              ▼
                          make -j$(nproc)  （用 .config'' 驱动编译）
```

---

## 5. 配置命令对比

| 命令 | 功能 | 使用场景 |
|------|------|---------|
| `make defconfig` | 生成当前架构的默认配置 | 从零开始 |
| `make oldconfig` | 逐一询问新增选项 | 升级内核版本后 |
| `make olddefconfig` | 新增选项用默认值 | 自动化脚本，不想交互 |
| `make menuconfig` | 图形化菜单配置 | 手动调整特定选项 |
| `make allnoconfig` | 所有选项设为 n | 最小内核（调试用） |
| `make allyesconfig` | 所有选项设为 y | 最大编译覆盖（CI 用） |
| `scripts/config --enable XXX` | 命令行修改单个选项 | 脚本中精确控制 |

### defconfig 是什么？

`arch/x86/configs/x86_64_defconfig` 是预定义的配置模板。`make defconfig` 会读取它生成 `.config`。

不同架构有不同的 defconfig：
```
arch/x86/configs/x86_64_defconfig
arch/arm64/configs/defconfig
arch/riscv/configs/defconfig
```

> **oldconfig vs olddefconfig**：升级内核版本后，新版本可能增加了新的配置选项。`oldconfig` 会逐一问你每个新选项选什么（交互式），`olddefconfig` 则自动用默认值填充所有新选项（非交互）。日常开发推荐 `olddefconfig`。

---

## 6. CONFIG_RUST 的启用

启用 Rust 支持的完整依赖链：

```
CONFIG_RUST=y
   │
   ├── depends on HAVE_RUST      ← 架构支持（目前 x86_64、arm64、riscv 等）
   ├── depends on RUST_IS_AVAILABLE ← make rustavailable 检查通过
   ├── depends on !MODVERSIONS    ← 模块版本控制与 Rust 不兼容
   ├── depends on !GCC_PLUGINS   ← GCC 插件与 Rust 不兼容
   └── depends on !RANDSTRUCT    ← 结构体随机化与 Rust 不兼容
```

```bash
# 完整启用流程
cd ~/linux

# 1. 确认 Rust 工具链可用
make LLVM=1 rustavailable
# ✓ 输出 "Rust is available!"

# 2. 生成默认配置
make LLVM=1 defconfig

# 3. 启用 Rust
scripts/config --enable CONFIG_RUST

# 4. 解决依赖、补全新选项
make LLVM=1 olddefconfig

# 5. 确认
grep CONFIG_RUST=y .config
```

> **为什么需要 LLVM=1？** Rust 的后端是 LLVM，内核的 Rust 代码需要和 C 代码链接在一起。如果 C 代码用 GCC 编译，可能产生与 LLVM 不兼容的目标文件。`LLVM=1` 让 C 代码也用 Clang（LLVM 前端）编译，确保整个内核使用同一套编译器后端。

---

## 7. Rust 在 Kbuild 中的编译流程

当 `CONFIG_RUST=y` 时，Kbuild 如何编译 Rust 代码：

```
                              编译流程
┌─────────────────────────────────────────────────────────────────┐
│                                                                   │
│  C 头文件                     bindgen                             │
│  include/linux/*.h  ──────────────────►  rust/bindings/          │
│                                          bindings_generated.rs   │
│                                               │                  │
│  Rust 核心库                                   │                  │
│  rust/kernel/*.rs  ◄──────────────────────────┘                  │
│       │                                                          │
│       │ rustc                                                    │
│       ▼                                                          │
│  rust/kernel.o                                                   │
│       │                                                          │
│       │                                                          │
│  Rust 模块代码                                                   │
│  samples/rust/rust_minimal.rs                                    │
│       │                                                          │
│       │ rustc                                                    │
│       ▼                                                          │
│  samples/rust/rust_minimal.o                                     │
│       │                                                          │
│       │ llvm-ld (链接器)                                         │
│       ▼                                                          │
│  ┌────────────────────────────────────┐                          │
│  │           bzImage                   │                          │
│  │  C 代码(.o) + Rust 代码(.o) 链接    │                          │
│  └────────────────────────────────────┘                          │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

### 关键步骤详解

**Step 1: bindgen 生成 FFI 绑定**

```bash
# bindgen 读取 C 头文件，输出 Rust FFI 声明
bindgen include/linux/slab.h --output rust/bindings/bindings_generated.rs
```

生成的内容类似：
```rust
// rust/bindings/bindings_generated.rs（自动生成，不要手动编辑）
extern "C" {
    pub fn kmalloc(size: usize, flags: gfp_t) -> *mut c_void;
    pub fn kfree(ptr: *const c_void);
}
```

**Step 2: 构建 Rust 核心库**

`rust/kernel/` 中的 safe 抽象层封装 bindgen 生成的 unsafe FFI：

```rust
// rust/kernel/allocator.rs（简化示意）
use crate::bindings;

pub unsafe fn kmalloc(size: usize, flags: u32) -> *mut u8 {
    // SAFETY: caller must ensure flags are valid for current context
    unsafe { bindings::kmalloc(size, flags) as *mut u8 }
}
```

**Step 3: 编译 Rust 模块**

```makefile
# samples/rust/Makefile
obj-$(CONFIG_SAMPLE_RUST_MINIMAL)   += rust_minimal.o
```

Kbuild 对 `.rs` 文件调用 `rustc` 而非 `gcc`/`clang`：

```bash
# Kbuild 内部执行（简化）
rustc --edition=2021 \
      --crate-type obj \
      --extern kernel=rust/libkernel.rmeta \
      -o samples/rust/rust_minimal.o \
      samples/rust/rust_minimal.rs
```

**Step 4: 链接**

所有 `.o` 文件（无论来自 C 还是 Rust）由链接器合并为最终的内核映像。

---

## 8. 练习

### 练习 1：探索 .config

```bash
cd ~/linux

# 生成默认配置
make LLVM=1 defconfig

# 回答以下问题：
# 1. .config 文件有多少行？
wc -l .config

# 2. 有多少个 CONFIG_XXX=y？多少个 =m？
grep -c '=y' .config
grep -c '=m' .config

# 3. CONFIG_RUST 当前的值是什么？
grep CONFIG_RUST .config
```

### 练习 2：理解 Kconfig 依赖

```bash
# 在 menuconfig 中搜索 RUST
make LLVM=1 menuconfig
# 按 / 键，输入 RUST，查看它的 depends on 和 selected by
```

回答：如果 `MODVERSIONS=y`，能启用 `CONFIG_RUST` 吗？为什么？

### 练习 3：追踪 Kbuild 变量替换

阅读 `samples/rust/Makefile`，列出所有 Rust 示例模块。对每个模块，说明：
- 它由哪个 `CONFIG_` 选项控制
- 当该选项设为 `y` 时，编译结果是什么

---

## 9. 知识检查

1. **Kconfig 中 `bool` 和 `tristate` 的区别是什么？**

2. **Kbuild 中 `obj-y`、`obj-m`、`obj-n` 分别意味着什么？**

3. **`make defconfig` 和 `make olddefconfig` 各在什么场景下使用？**

4. **为什么编译 Rust 内核代码必须加 `LLVM=1`？**

5. **bindgen 在内核 Rust 编译流程中的角色是什么？**

---

## 10. 参考资源

| 资源 | 说明 |
|------|------|
| [Kbuild documentation](https://docs.kernel.org/kbuild/index.html) | 内核官方 Kbuild 文档 |
| [Kconfig language](https://docs.kernel.org/kbuild/kconfig-language.html) | Kconfig 语法规范 |
| 内核源码 `Makefile`（顶层） | 构建系统入口 |
| 内核源码 `scripts/Makefile.build` | Kbuild 的核心实现 |
| 内核源码 `init/Kconfig` | CONFIG_RUST 的定义位置 |
