# kernel Crate：核心抽象

> 模块：rfl-arch-02 | 难度：进阶 | 预计时间：4 小时
> 前置要求：[Rust 与内核：全景图](rfl-arch-01.md)、[数据结构与模式](kernel-basics-03.md)

---

## 目录

1. [kernel crate 的整体结构](#1-kernel-crate-的整体结构)
2. [Error 类型与 Result](#2-error-类型与-result)
3. [Module trait 与 module! 宏](#3-module-trait-与-module-宏)
4. [内核中的 Arc](#4-内核中的-arc)
5. [KVec 与 fallible allocation](#5-kvec-与-fallible-allocation)
6. [CStr 与字符串处理](#6-cstr-与字符串处理)
7. [Opaque\<T\> 与自引用结构体](#7-opaquettypedef-与自引用结构体)
8. [设计哲学总结](#8-设计哲学总结)
9. [练习](#9-练习)
10. [知识检查](#10-知识检查)
11. [参考资源](#11-参考资源)

---

## 1. kernel crate 的整体结构

`rust/kernel/` 是整个 Rust 内核生态的核心 crate，位于三层架构的第二层（safe 抽象层）。

```
┌─────────────────────────────┐
│  Rust 内核模块 / 驱动       │  ← 尽量不写 unsafe
├─────────────────────────────┤
│  rust/kernel/ (safe 抽象)   │  ← 本模块学习的这一层
├─────────────────────────────┤
│  bindgen FFI (unsafe 绑定)  │  ← 自动生成
├─────────────────────────────┤
│  C 内核代码                 │
└─────────────────────────────┘
```

`rust/kernel/lib.rs` 是 crate 的入口，主要子模块：

| 文件/目录 | 职责 |
|-----------|------|
| `error.rs` | Error 类型，errno 到 Rust 的映射 |
| `init.rs` + `init/` | pin-init 系统，就地初始化 |
| `sync/` | 锁、Arc、CondVar 等并发原语 |
| `alloc/` | 内核内存分配器，Vec 扩展 |
| `str.rs` | CStr、BStr 等字符串处理 |
| `print.rs` | pr_info!、pr_err! 等打印宏 |
| `types.rs` | Opaque、ARef 等基础类型 |

### 内核场景的硬约束

这些约束直接决定了 kernel crate 的设计：

1. **分配可能失败** — 内核没有 OOM killer 兜底（对内核自身而言），所有内存分配必须是 fallible 的
2. **上下文敏感** — 同样是分配内存，在进程上下文可以睡眠等待（`GFP_KERNEL`），在中断上下文不能（`GFP_ATOMIC`）
3. **生命周期由 C 侧控制** — 很多内核对象的创建和销毁由 C 框架管理
4. **初始化必须就地完成** — 含锁的结构体不能先在栈上构造再 move

---

## 2. Error 类型与 Result

### 2.1 C 内核的错误约定

C 内核用**负数 errno** 表示错误：

```c
int ret = do_something();
if (ret < 0)
    return ret;  // 传播错误，如 -ENOMEM, -EINVAL
```

### 2.2 Rust 的 Error 封装

`rust/kernel/error.rs` 的核心定义：

```rust
// 内核错误类型，封装一个非零负 errno
pub struct Error(NonZeroI32);
```

**为什么用 `NonZeroI32` 而不是 `c_int`？**

- `NonZeroI32` 使用 `#[repr(transparent)]`，内存布局和 `i32` 完全一样 — 4 字节
- "非零"是编译期约束，运行时零开销
- 编译器可以做 **niche optimization**：`Option<Error>` 和 `Error` 大小相同（零值表示 `None`）
- 和 C 的 errno 二进制兼容，跨 FFI 边界**零开销**

**为什么不用 enum（如 `enum Error { NoMem, Inval, ... }`）？**

1. **FFI 转换代价高** — enum 的内部编号和 errno 数值不对应，每次跨 FFI 边界都要 match 转换
2. **无法穷尽** — Linux 有几百个 errno，不同架构还有私有的
3. **不需要穷尽匹配** — 内核中大多数时候对错误是直接传播（`?`），不是逐个 match

### 2.3 错误码定义

错误码定义在独立的 `code` 子模块中：

```rust
pub mod code {
    declare_err!(EPERM, "Operation not permitted.");
    declare_err!(ENOMEM, "Out of memory.");
    declare_err!(EINVAL, "Invalid argument.");
    declare_err!(EBUSY, "Device or resource busy.");
    declare_err!(EPROBE_DEFER, "Driver requests probe retry.");
    // ... 50+ 个错误码
}
```

`declare_err!` 宏通过 `crate::bindings::$err` 直接引用 bindgen 从 C 头文件生成的 errno 值，保证和 C 侧完全一致。

### 2.4 构造时检查

```rust
const fn try_from_errno(errno: crate::ffi::c_int) -> Option<Error> {
    if errno < -(bindings::MAX_ERRNO as i32) || errno >= 0 {
        return None;  // 0 和正数都拒绝
    }
    Some(unsafe { Error::from_errno_unchecked(errno) })
}
```

先检查范围（包含了 0），通过后才调用 unsafe 的 `new_unchecked`。**safe 外壳 + unsafe 内核，边界处检查一次。**

### 2.5 FFI 桥接函数

三个关键函数覆盖了内核中所有的错误传递模式：

```
C 返回 int       ──→  to_result()     ──→  Rust Result
C 返回 error ptr ──→  from_err_ptr()  ──→  Rust Result<*mut T>
Rust Result      ──→  from_result()   ──→  C int
```

### 2.6 From trait 实现

常见 Rust 错误自动映射到 errno：

| Rust 错误 | 映射到 |
|-----------|--------|
| `AllocError` | `ENOMEM` |
| `TryFromIntError` | `EINVAL` |
| `Utf8Error` | `EINVAL` |
| `LayoutError` | `ENOMEM` |
| `fmt::Error` | `EINVAL` |

让 `?` 运算符可以自动转换这些错误类型。

### 2.7 统一 Result

```rust
pub type Result<T = (), E = Error> = core::result::Result<T, E>;
```

默认 `T = ()`，所以 `Result` 等同于 `Result<()>`，表示"只关心成功与否"。

文档中**明确禁止 `unwrap()`** — 类比 C 的 `BUG()` 宏，在内核中是 "heavily discouraged"。

---

## 3. Module trait 与 module! 宏

### 3.1 基本用法

```rust
module! {
    type: RustMinimal,
    name: "rust_minimal",
    license: "GPL",
}

struct RustMinimal {
    numbers: KVec<i32>,
}

impl kernel::Module for RustMinimal {
    fn init(_module: &'static ThisModule) -> Result<Self> {
        let mut numbers = KVec::new();
        numbers.push(72, GFP_KERNEL)?;
        Ok(RustMinimal { numbers })
    }
}

impl Drop for RustMinimal {
    fn drop(&mut self) {
        pr_info!("Goodbye\n");
    }
}
```

### 3.2 和 C 模块的对比

| 概念 | C 内核 | Rust 内核 |
|------|--------|-----------|
| 初始化 | `module_init(fn)` 返回 int | `Module::init()` 返回 `Result<Self>` |
| 清理 | `module_exit(fn)` 手动释放 | `Drop::drop()` 自动调用 |
| 模块状态 | 全局变量 | struct 的字段 |
| 错误时清理 | 手动 goto 链 | `?` 自动 drop 已构造的部分 |

### 3.3 告别 goto 清理链

C 内核中最常见的 bug 来源之一：

```c
static int __init my_init(void) {
    ret = alloc_resource_a(&res_a);
    if (ret < 0) return ret;

    ret = alloc_resource_b(&res_b);
    if (ret < 0) goto err_free_a;   // 跳错标签？忘记释放？

    ret = alloc_resource_c(&res_c);
    if (ret < 0) goto err_free_b;

    return 0;
err_free_b: free_resource_b(res_b);
err_free_a: free_resource_a(res_a);
    return ret;
}
```

Rust 的 `Result<Self>` + `Drop`：

```rust
fn init() -> Result<Self> {
    let res_a = alloc_resource_a()?;  // 失败 → 直接返回
    let res_b = alloc_resource_b()?;  // 失败 → res_a 自动 drop
    let res_c = alloc_resource_c()?;  // 失败 → res_b, res_a 自动 drop
    Ok(MyModule { res_a, res_b, res_c })
}
```

编译器保证已构造的资源**一定会被释放**，释放顺序是**构造的逆序**。

---

## 4. 内核中的 Arc

### 4.1 和标准库的 5 个差异

源码文件 `rust/kernel/sync/arc.rs` 的文档注释明确列出：

| # | 标准库 Arc | 内核 Arc |
|---|-----------|----------|
| 1 | Rust 原子操作 | 内核 `refcount_t`（带安全检测） |
| 2 | 支持 Weak（两个计数器） | **不支持**（体积减半） |
| 3 | 溢出时 abort | **饱和**（卡在最大值不溢出） |
| 4 | 有 `get_mut` | **没有**（隐式 pin） |
| 5 | 显式 Pin | **隐式 pin**（Arc 内对象始终 pinned） |

### 4.2 内部结构

```rust
struct ArcInner<T: ?Sized> {
    refcount: Opaque<bindings::refcount_t>,  // C 内核的引用计数
    data: T,
}
```

用 C 的 `refcount_t` 而非 Rust 的 `AtomicUsize`，因为内核的 `refcount_t` 集成了 **REFCOUNT_FULL** 安全检测。

### 4.3 创建时的差异

```rust
// 标准库
let a = Arc::new(42);                        // 永远成功

// 内核：多了 GFP flags 和 ?
let a = Arc::new(42, GFP_KERNEL)?;           // 可能失败
```

两处差异对应内核的两个硬约束：**分配可能失败** + **上下文敏感**。

---

## 5. KVec 与 fallible allocation

```rust
// 标准库 — 分配失败就 panic
let mut v = Vec::new();
v.push(42);              // 永远成功

// 内核 — 每次操作都可能失败
let mut v = KVec::new();
v.push(42, GFP_KERNEL)?; // 返回 Result，? 传播错误
```

**每一个涉及内存分配的操作都带 `Flags` 参数，都返回 `Result`。** 这是整个 kernel crate 中最普遍的模式。

---

## 6. CStr 与字符串处理

内核中的字符串是 C 风格的 null-terminated 字节串：

```rust
use kernel::str::CStr;
let s = c_str!("hello");          // 编译期构造，零开销
pr_info!("message: {}\n", s);
```

`c_str!` 宏在编译期生成 null-terminated 的字节串，运行时没有任何分配或转换。

---

## 7. Opaque\<T\> 与自引用结构体

### 7.1 Opaque\<T\>

```rust
pub struct Opaque<T> {
    value: UnsafeCell<MaybeUninit<T>>,
    _pin: PhantomPinned,
}
```

作用：**包装一个 C 内核的数据结构，让 Rust 侧不要动它。**

| 组成 | 作用 |
|------|------|
| `UnsafeCell` | 允许通过共享引用修改内部值（C 代码会直接操作它） |
| `MaybeUninit` | 允许未初始化状态（由 C 侧负责初始化） |
| `PhantomPinned` | 标记不可移动（C 侧可能持有指向它的指针） |

### 7.2 自引用结构体

当一个对象（如包含 `list_head` 的结构体）插入链表后，其他节点的指针指向它。如果 Rust 把这个对象 **move** 到新地址，外部指针变成悬空指针 — use-after-free。

内核中这种模式非常普遍：链表节点、互斥锁（等待队列指针）、工作队列等。

### 7.3 Pin 的本质

```
普通引用 &T      →  只保证 T 有效
Pin<&T>          →  保证 T 有效 且 不会被移动
```

内核 Arc 不提供 `get_mut()` — 有 `&mut T` 就能 `mem::swap()` 移走对象，破坏 Pin 保证。

### 7.4 pin-init 的由来

正常 Rust 的构造流程：栈上构造 → move 到堆上。但含锁的结构体不能 move，所以需要 pin-init：

```
先在堆上分配内存 → 在原地（in-place）初始化 → 永不 move
```

---

## 8. 设计哲学总结

```
标准库的假设               内核的现实               kernel crate 的应对
────────────             ─────────             ──────────────
分配永远成功              分配可能失败              所有分配返回 Result
一种分配方式              上下文敏感(GFP flags)     分配操作带 Flags 参数
可以 panic               不能 panic              禁止 unwrap()
有 std                   no_std                  重新实现核心类型
对象可以 move             含自引用结构不能 move      隐式 Pin + pin-init
独立的内存模型             C 管理生命周期            Opaque<T> + FFI 桥接
```

---

## 9. 练习

### 练习 1：读 error.rs

阅读 `rust/kernel/error.rs`，关注第 69-87 行的内核特有错误码（如 `EPROBE_DEFER`、`ERESTARTSYS`）。查阅 `EPROBE_DEFER` 在内核设备模型中的作用。

### 练习 2：写一个 Rust 内核模块

目标：
- 模块加载时打印 "Hello from my module"
- 模块卸载时打印 "Goodbye from my module"
- 内部用 KVec 存储几个数字并在卸载时打印它们的和
- 添加一个模块参数

参考 `samples/rust/rust_minimal.rs`。

### 练习 3：追踪 pin_init! 宏

在 `rust/kernel/init.rs` 中，找到 `pin_init!` 的定义。对一个包含 `Mutex` 字段的结构体，理解为什么需要用 `<-` 而不是 `:` 来初始化 Mutex 字段。

---

## 10. 知识检查

1. **kernel::Error 和 std::io::Error 有什么不同？为什么内核选择封装 `NonZeroI32` 而不是用 enum？**

2. **为什么需要 pin-init？它解决什么问题？**

3. **内核 Arc 和标准库 Arc 有哪些差异？各自的设计原因是什么？**

---

## 11. 参考资源

| 资源 | 说明 |
|------|------|
| `rust/kernel/error.rs` | Error 类型与 FFI 桥接函数 |
| `rust/kernel/sync/arc.rs` | 内核 Arc 实现 |
| `rust/kernel/types.rs` | Opaque、ARef 等基础类型 |
| `rust/kernel/init.rs` | pin-init 系统 |
| `samples/rust/rust_minimal.rs` | 最小内核模块示例 |
| [Rust in the kernel docs](https://docs.kernel.org/rust/index.html) | 内核官方 Rust 文档 |
