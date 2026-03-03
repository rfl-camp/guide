# Rust 与内核：全景图

> 模块：rfl-arch-01 | 难度：入门 | 预计时间：2 小时
> 前置要求：[内核架构概览](kernel-basics-01.md)

---

## 目录

1. [为什么内核需要 Rust](#1-为什么内核需要-rust)
2. [三层架构：从 C 到 safe Rust](#2-三层架构从-c-到-safe-rust)
3. [rust/ 目录结构](#3-rust-目录结构)
4. [Mutex\<T\>：数据内置锁设计](#4-mutext数据内置锁设计)
5. [module! 宏与 no_std 环境](#5-module-宏与-no_std-环境)
6. [Rust 模块的 Kbuild 编译流程](#6-rust-模块的-kbuild-编译流程)
7. [当前主线内核 Rust 状态](#7-当前主线内核-rust-状态)
8. [练习](#8-练习)
9. [知识检查](#9-知识检查)
10. [参考资源](#10-参考资源)

---

## 1. 为什么内核需要 Rust

### 1.1 CVE 数据说话

根据微软、Google、Android 团队的统计：

| 来源 | 数据 |
|------|------|
| 微软 | 其产品 **~70%** 的安全漏洞是内存安全问题 |
| Google Chrome | **~70%** 的高严重度漏洞是内存安全问题 |
| Android | 引入 Rust 后，内存安全漏洞从 **223 个**（2019）降至 **85 个**（2022） |
| Linux 内核 CVE | 大量 use-after-free、缓冲区溢出、空指针解引用 |

**内存安全问题的典型类型**：

```
┌─────────────────────────────────────────────────────────┐
│                  C 代码中常见的内存安全 bug               │
│                                                           │
│  use-after-free     释放后还在用，导致数据损坏或提权       │
│  buffer overflow    数组越界写入，可能执行任意代码         │
│  double free        重复释放同一块内存                     │
│  null deref         空指针解引用，导致 kernel panic        │
│  data race          多线程无锁访问共享数据                 │
│  uninitialized mem  使用未初始化的内存                     │
└─────────────────────────────────────────────────────────┘
```

### 1.2 C 语言的根本限制

C 语言在设计上缺乏内存安全保证：

```c
// C 代码：编译通过，运行出问题
void dangerous(void) {
    int *p = kmalloc(sizeof(int), GFP_KERNEL);
    kfree(p);
    *p = 42;           // use-after-free！编译器不报错

    int buf[10];
    buf[15] = 0;        // buffer overflow！编译器不报错

    // data race：两个线程同时修改，没有锁
    shared_counter++;   // 编译器不报错
}
```

C 依赖程序员自觉遵守规则，编译器不做强制检查。在百万行规模的代码库（Linux 内核 3000 万行）中，人类不可能不犯错。

### 1.3 Rust 的解决方案

Rust 将内存安全检查移到**编译期**：

```rust
fn safe_example() {
    let p = Box::new(42);
    drop(p);
    // *p = 42;          // 编译错误！所有权已转移

    let buf = [0i32; 10];
    // buf[15] = 0;      // 编译错误！索引越界

    // data race：
    // let counter = Arc::new(Mutex::new(0));
    // 必须通过 Mutex::lock() 才能修改，编译器强制执行
}
```

**Rust 在内核中并不是要替换所有 C 代码**。目标是：
- **新驱动**用 Rust 编写（驱动占内核 ~70% 代码，bug 密度最高）
- **安全抽象层**封装已有的 C API
- 逐步在 bug 热点区域引入 Rust

---

## 2. 三层架构：从 C 到 safe Rust

Rust-for-Linux 的核心设计是**三层架构**：

```
┌─────────────────────────────────────────────────────────┐
│                       第三层                              │
│                  Rust 驱动 / 模块                         │
│              （safe Rust，无 unsafe）                     │
│                                                           │
│  samples/rust/rust_minimal.rs                             │
│  drivers/gpu/nova/                                        │
│  drivers/net/phy/ax88796b_rust.rs                        │
│                                                           │
│  目标：驱动开发者只写 safe Rust，不接触 unsafe            │
├───────────────────────────────────────────────────────────┤
│                       第二层                              │
│              Rust safe 抽象层                             │
│          （包含 unsafe，封装 C API）                      │
│                                                           │
│  rust/kernel/sync/mutex.rs     → Mutex<T>                │
│  rust/kernel/error.rs          → Error, Result            │
│  rust/kernel/alloc/            → 内存分配                 │
│  rust/kernel/init.rs           → pin-init 初始化          │
│                                                           │
│  目标：unsafe 代码集中在这里，提供 safe 接口给上层        │
├───────────────────────────────────────────────────────────┤
│                       第一层                              │
│              bindgen 生成的 FFI 绑定                      │
│            （自动生成，全部 unsafe）                       │
│                                                           │
│  rust/bindings/bindings_generated.rs                      │
│                                                           │
│  bindgen 读取 C 头文件 → 输出 extern "C" { fn ... }      │
│  所有函数都是 unsafe 的，直接对应 C ABI                   │
└───────────────────────────────────────────────────────────┘
         │
         │  bindgen 工具
         ▼
┌───────────────────────────────────────────────────────────┐
│                     C 头文件                               │
│  include/linux/mutex.h                                    │
│  include/linux/slab.h                                     │
│  include/linux/device.h                                   │
│  ...                                                      │
└───────────────────────────────────────────────────────────┘
```

### 各层的角色

| 层次 | 谁写 | 包含 unsafe？ | 复杂度 |
|------|------|-------------|--------|
| 第三层：驱动/模块 | 驱动开发者 | **否**（理想状态） | 低 |
| 第二层：safe 抽象 | 内核 Rust 专家 | **是**（受控） | 高 |
| 第一层：FFI 绑定 | bindgen 自动生成 | **全部** | — |

### 为什么这种分层有意义？

- **unsafe 代码集中**：只有少数专家编写和审核 unsafe 代码
- **驱动开发门槛低**：驱动作者不需要理解内核 C API 的细节，用 safe Rust 接口即可
- **bug 发现更容易**：如果出了内存安全 bug，99% 的概率在第二层（unsafe 抽象层），搜索范围极大缩小

### bindgen 做了什么？

```bash
# bindgen 的工作
bindgen include/linux/mutex.h \
    --output rust/bindings/bindings_generated.rs
```

输入（C 头文件）：
```c
// include/linux/mutex.h
struct mutex {
    atomic_long_t owner;
    raw_spinlock_t wait_lock;
    struct list_head wait_list;
};

void mutex_lock(struct mutex *lock);
void mutex_unlock(struct mutex *lock);
```

输出（Rust FFI）：
```rust
// rust/bindings/bindings_generated.rs（自动生成）
#[repr(C)]
pub struct mutex {
    pub owner: atomic_long_t,
    pub wait_lock: raw_spinlock_t,
    pub wait_list: list_head,
}

extern "C" {
    pub fn mutex_lock(lock: *mut mutex);
    pub fn mutex_unlock(lock: *mut mutex);
}
```

> 这些 FFI 函数都是 `unsafe` 的——bindgen 不知道调用它们需要满足什么前提条件（如"必须在进程上下文"、"不能持有自旋锁"等）。第二层的 safe 抽象层就是要把这些隐含的规则编码为 Rust 类型系统的约束。

---

## 3. rust/ 目录结构

```
rust/
├── kernel/                 # 核心抽象层（第二层）
│   ├── lib.rs              # crate 根，re-export 所有模块
│   ├── error.rs            # Error 类型，对应 -errno
│   ├── print.rs            # pr_info! 等打印宏
│   ├── sync/               # 同步原语
│   │   ├── mutex.rs        # Mutex<T> 抽象
│   │   ├── lock.rs         # 锁的通用框架
│   │   └── arc.rs          # Arc 引用计数
│   ├── alloc/              # 内存分配
│   ├── init.rs             # pin-init 初始化宏
│   ├── types.rs            # 基础类型
│   ├── str.rs              # 字符串处理（CStr 等）
│   ├── module.rs           # module! 宏定义
│   └── ...
│
├── bindings/               # bindgen 生成的 FFI（第一层）
│   └── bindings_generated.rs
│
├── helpers/                # C 辅助函数
│   └── helpers.c           # 无法通过 bindgen 转换的内联函数
│
├── macros/                 # 过程宏
│   └── lib.rs              # module! 宏的实现
│
└── uapi/                   # 用户空间 API 绑定
```

### 关键文件说明

| 文件 | 作用 |
|------|------|
| `kernel/lib.rs` | crate 入口，`pub mod` 声明所有子模块 |
| `kernel/error.rs` | 将 C 的 `-errno` 映射为 Rust 的 `Error` 类型 |
| `kernel/print.rs` | `pr_info!`、`pr_err!` 等宏，底层调用 C 的 `printk` |
| `kernel/sync/mutex.rs` | 将 C 的 `struct mutex` 封装为 safe 的 `Mutex<T>` |
| `kernel/init.rs` | pin-init 系统，解决内核结构体的就地初始化问题 |
| `helpers/helpers.c` | 有些 C 函数是 `static inline`，bindgen 无法处理，需要手写 C 包装 |

---

## 4. Mutex\<T\>：数据内置锁设计

这是理解 Rust-for-Linux 设计哲学最好的例子。

### 4.1 C 的做法：锁和数据分离

```c
// C 风格：锁和数据是两个独立的东西
struct mutex lock;
int shared_data;

// 正确用法：先锁再访问
mutex_lock(&lock);
shared_data = 42;
mutex_unlock(&lock);

// 错误用法：忘记锁！编译通过，运行时 data race
shared_data = 42;    // 没有锁保护！编译器不报错
```

问题：C 编译器**无法检查**你是否在持有锁的情况下访问数据。

### 4.2 Rust 的做法：锁包裹数据

```rust
// Rust 风格：数据在锁里面，必须先获取锁才能访问数据
let mutex: Mutex<i32> = Mutex::new(0);

// 获取锁，返回 Guard，通过 Guard 访问数据
{
    let mut guard = mutex.lock();
    *guard = 42;    // 只有通过 Guard 才能访问数据
}   // guard 离开作用域 → 自动解锁

// 不获取锁就无法访问数据
// mutex.get_data()  ← 不存在这样的方法！

// 忘记解锁？不可能！Guard 的 Drop 自动解锁
```

### 4.3 设计对比

```
C 风格：
┌──────────┐     ┌──────────┐
│  mutex   │     │  data    │     锁和数据是分开的
│  lock    │     │  int x   │     你可以不锁就访问 data
└──────────┘     └──────────┘     编译器不管

Rust 风格：
┌──────────────────────┐
│  Mutex<T>            │
│  ┌────────────────┐  │     数据在锁里面
│  │  T (data)      │  │     没有锁的 Guard 就没法碰 T
│  └────────────────┘  │     编译器强制执行
│  lock state           │
└──────────────────────┘
```

### 4.4 内核 Rust Mutex 的实现概要

```rust
// rust/kernel/sync/mutex.rs（简化）

/// 包裹一个 C struct mutex 和被保护的数据
pub struct Mutex<T> {
    mutex: Opaque<bindings::mutex>,  // C 的 struct mutex
    data: UnsafeCell<T>,             // 被保护的数据
}

impl<T> Mutex<T> {
    /// 获取锁，返回 Guard
    pub fn lock(&self) -> Guard<'_, T> {
        // SAFETY: mutex 已正确初始化
        unsafe { bindings::mutex_lock(self.mutex.get()) };
        Guard { mutex: self }
    }
}

/// Guard 持有锁，允许访问数据
pub struct Guard<'a, T> {
    mutex: &'a Mutex<T>,
}

impl<T> Deref for Guard<'_, T> {
    type Target = T;
    fn deref(&self) -> &T {
        // SAFETY: Guard 存在意味着锁已被持有
        unsafe { &*self.mutex.data.get() }
    }
}

impl<T> Drop for Guard<'_, T> {
    fn drop(&mut self) {
        // Guard 离开作用域时自动解锁
        // SAFETY: 锁是由这个 Guard 获取的
        unsafe { bindings::mutex_unlock(self.mutex.mutex.get()) };
    }
}
```

**关键设计点**：

1. **`UnsafeCell<T>`**：数据被 `UnsafeCell` 包裹，Rust 的借用规则阻止直接访问
2. **`Guard` 的 `Deref`**：只有通过 Guard 才能解引用到数据
3. **`Guard` 的 `Drop`**：离开作用域自动解锁，不可能忘记
4. **unsafe 集中在抽象层**：使用者只看到 safe API

> 这就是"safe 抽象"的本质：在第二层用 `unsafe` 调用 C 函数，但暴露给第三层的 API 是完全 safe 的。类型系统在编译期保证正确使用。

---

## 5. module! 宏与 no_std 环境

### 5.1 no_std：没有标准库

内核代码不能使用 Rust 标准库（`std`），因为：

- `std` 依赖操作系统提供的功能（文件 I/O、线程、网络），但内核**就是**操作系统
- `std` 的内存分配可能 panic（OOM），内核分配必须是 fallible（可失败）的
- `std` 的线程 API 不适用于内核的调度模型

内核 Rust 代码使用 `#![no_std]`，只依赖：

| 库 | 提供 |
|----|------|
| `core` | 基础类型、trait、Option、Result（不分配内存） |
| `alloc` | Vec、Box、Arc（内核提供自定义分配器） |
| `kernel` | 内核专用 crate（RFL 提供） |

### 5.2 module! 宏

每个 Rust 内核模块的入口：

```rust
// samples/rust/rust_minimal.rs
//! Rust minimal sample.

use kernel::prelude::*;

module! {
    type: RustMinimal,
    name: "rust_minimal",
    author: "Rust for Linux Contributors",
    description: "Rust minimal sample",
    license: "GPL",
}

struct RustMinimal {
    numbers: KVec<i32>,
}

impl kernel::Module for RustMinimal {
    fn init(_module: &'static ThisModule) -> Result<Self> {
        pr_info!("Rust minimal sample (init)\n");
        pr_info!("Am I built-in or a module? {}\n",
            if cfg!(MODULE) { "Module" } else { "Built-in" });

        let mut numbers = KVec::new();
        numbers.push(72, GFP_KERNEL)?;
        numbers.push(108, GFP_KERNEL)?;
        numbers.push(200, GFP_KERNEL)?;

        Ok(RustMinimal { numbers })
    }
}

impl Drop for RustMinimal {
    fn drop(&mut self) {
        pr_info!("My numbers are {:?}\n", self.numbers);
        pr_info!("Rust minimal sample (exit)\n");
    }
}
```

### 5.3 module! 宏做了什么

`module!` 是一个过程宏（定义在 `rust/macros/lib.rs`），它展开后生成：

```rust
// 伪展开（实际更复杂）

// 1. 模块元数据（对应 C 的 MODULE_INFO）
#[link_section = ".modinfo"]
static __MODULE_NAME: &str = "rust_minimal";
static __MODULE_LICENSE: &str = "GPL";

// 2. init 函数入口（对应 C 的 module_init）
#[no_mangle]
unsafe extern "C" fn init_module() -> i32 {
    match <RustMinimal as kernel::Module>::init(...) {
        Ok(instance) => { /* 保存 instance */ 0 }
        Err(e) => e.to_errno()
    }
}

// 3. exit 函数入口（对应 C 的 module_exit）
#[no_mangle]
unsafe extern "C" fn cleanup_module() {
    drop(instance);  // 调用 Drop::drop
}
```

**关键对比**：

| C 模块 | Rust 模块 |
|--------|----------|
| `module_init(fn)` + `module_exit(fn)` | `module!` 宏 + `Module` trait + `Drop` |
| init 返回 int（0 或 -errno） | init 返回 `Result<Self>` |
| exit 手动释放所有资源 | `Drop` 自动释放 |
| 忘记释放 → 内存泄漏 | 编译器保证 Drop 被调用 |

---

## 6. Rust 模块的 Kbuild 编译流程

```
源文件                       中间产物                    最终产物
┌─────────────────┐
│ C 头文件         │
│ include/linux/  │──── bindgen ────► bindings_generated.rs
└─────────────────┘                         │
                                            ▼
┌─────────────────┐                  ┌──────────────┐
│ rust/kernel/*.rs │──── rustc ─────►│ libkernel.o  │
│ (safe 抽象层)    │                  └──────┬───────┘
└─────────────────┘                         │
                                            │
┌─────────────────┐                         │
│ rust_minimal.rs  │──── rustc ─────►┌──────┴───────┐
│ (驱动模块)       │  (链接 kernel)   │rust_minimal.o│
└─────────────────┘                  └──────┬───────┘
                                            │
┌─────────────────┐                         │
│ C 代码 (.c)      │──── clang ─────►┌──────┴───────┐
│ (其余内核代码)    │                  │ 其余 .o 文件 │
└─────────────────┘                  └──────┬───────┘
                                            │
                                     llvm-ld (链接)
                                            │
                                            ▼
                                     ┌──────────────┐
                                     │   bzImage    │
                                     │ (内核映像)    │
                                     └──────────────┘
```

### Kbuild 中的 Rust 规则

```makefile
# samples/rust/Makefile
obj-$(CONFIG_SAMPLE_RUST_MINIMAL) += rust_minimal.o

# Kbuild 看到 .rs 源文件时，自动使用 rustc 而不是 gcc/clang
# 规则定义在 scripts/Makefile.build 中
```

### 编译命令详解

```bash
# 完整编译（包含 Rust）
make LLVM=1 -j$(nproc)

# 只看 Rust 相关的编译输出
make LLVM=1 -j$(nproc) 2>&1 | grep -E 'RUSTC|BINDGEN'

# 输出示例：
# BINDGEN rust/bindings/bindings_generated.rs
# RUSTC   rust/kernel.o
# RUSTC   samples/rust/rust_minimal.o
# RUSTC   samples/rust/rust_print.o
```

### make 目标

| 命令 | 作用 |
|------|------|
| `make LLVM=1 rustavailable` | 检查 Rust 工具链是否满足要求 |
| `make LLVM=1 rustfmt` | 检查 Rust 代码格式 |
| `make LLVM=1 rustfmtfix` | 自动修复格式 |
| `make LLVM=1 CLIPPY=1` | 编译时启用 Clippy 检查 |
| `make LLVM=1 rustdoc` | 生成 Rust 文档 |
| `make LLVM=1 rust-analyzer` | 生成 rust-project.json |

---

## 7. 当前主线内核 Rust 状态

### 时间线

| 时间 | 里程碑 |
|------|--------|
| 2020 | Miguel Ojeda 提出 RFC，开始在内核中实验 Rust |
| 2021 | Rust for Linux RFC v2，基础设施搭建 |
| 2022-12 | **Linux 6.1**：Rust 基础支持合入主线（`CONFIG_RUST`、`rust/kernel/`） |
| 2023 | 持续完善抽象层：sync、alloc、init |
| 2024 | 首个 Rust PHY 驱动合入主线；nova GPU 驱动开发中 |
| 2025 | pin-init 独立 crate、更多子系统采纳 Rust |

### 已合入主线的 Rust 组件

| 组件 | 说明 |
|------|------|
| `rust/kernel/` 基础抽象 | Module、Error、print、str、types |
| 同步原语 | Mutex、SpinLock、Arc、CondVar、LockedBy |
| 内存分配 | fallible Box/Vec、GFP flags |
| pin-init | 就地初始化框架 |
| PHY 驱动 | `drivers/net/phy/` 中的 Rust PHY 驱动 |
| DRM (nova) | `drivers/gpu/nova/` NVIDIA GPU 驱动 |

### 活跃开发中

| 方向 | 状态 |
|------|------|
| 块设备抽象 | 补丁审核中 |
| 网络设备抽象 | 早期开发 |
| 文件系统 | 实验阶段（Rust ext2） |
| 固件加载 | 补丁审核中 |
| DMA 抽象 | 开发中 |

### 社区与维护者

| 角色 | 人物 |
|------|------|
| RFL 维护者 | Miguel Ojeda |
| DRM/nova | Danilo Krummrich |
| PHY 驱动 | FUJITA Tomonori |
| pin-init | Benno Lossin |

---

## 8. 练习

### 练习 1：浏览 rust/kernel/ 目录

```bash
cd ~/linux

# 列出 rust/kernel/ 的文件结构
find rust/kernel/ -name "*.rs" | head -30

# 查看 lib.rs 中声明了哪些子模块
grep "^pub mod" rust/kernel/lib.rs
```

回答：`rust/kernel/` 目前包含多少个 `.rs` 文件？列出你认为最重要的 5 个并说明理由。

### 练习 2：追踪 Rust 模块编译

```bash
# 清理并重新编译，观察 Rust 编译步骤
make LLVM=1 clean
make LLVM=1 defconfig
scripts/config --enable CONFIG_RUST
scripts/config --enable CONFIG_SAMPLE_RUST_MINIMAL
make LLVM=1 olddefconfig
make LLVM=1 -j$(nproc) 2>&1 | grep -E 'RUSTC|BINDGEN|EXPORTS'
```

记录输出中出现的每一步，说明该步骤的作用。

### 练习 3：阅读 rust_minimal.rs

```bash
cat ~/linux/samples/rust/rust_minimal.rs
```

回答：
1. `module!` 宏中每个字段的作用是什么？
2. `init` 函数返回 `Result<Self>` 意味着什么？
3. 资源清理在哪里发生？为什么不需要显式调用 cleanup 函数？

---

## 9. 知识检查

1. **bindgen 在 Rust-for-Linux 中的角色是什么？它的输入和输出分别是什么？**

2. **"safe 抽象"在 RFL 中意味着什么？为什么 unsafe 代码集中在第二层？**

3. **为什么 Rust 内核代码不能使用标准库（std）？它用什么替代？**

4. **C 的 Mutex 和 Rust 的 Mutex\<T\> 在设计上有什么根本区别？这如何提高安全性？**

5. **Rust 内核模块的 init/exit 与 C 内核模块有什么区别？**

---

## 10. 参考资源

| 资源 | 说明 |
|------|------|
| [Rust in the kernel docs](https://docs.kernel.org/rust/index.html) | 内核官方 Rust 文档 |
| [RFL upstream tree](https://github.com/Rust-for-Linux/linux) | RFL 开发仓库 |
| [Kangrejos conference](https://kangrejos.com) | RFL 年度技术会议 |
| [rust-for-linux.com](https://rust-for-linux.com) | RFL 官网 |
| [Zulip chat](https://rust-for-linux.zulipchat.com) | RFL 社区聊天 |
| `rust/kernel/` 源码 | 最好的学习资料就是代码本身 |
| `samples/rust/` | 示例模块，入门必读 |
