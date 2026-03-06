# Pin-init 与就地初始化

> 模块：rfl-memory-02 | 难度：高级 | 预计时间：4 小时
> 前置要求：[内核内存分配](rfl-memory-01.md)、[并发原语](rfl-concurrency-01.md)

---

## 目录

1. [为什么需要就地初始化](#1-为什么需要就地初始化)
2. [PinInit 和 Init trait](#2-pininit-和-init-trait)
3. [#\[pin_data\] 宏](#3-pin_data-宏)
4. [try_pin_init! 宏语法](#4-try_pin_init-宏语法)
5. [InPlaceInit trait：完整流程](#5-inplaceinit-trait完整流程)
6. [结构化 pinning](#6-结构化-pinning)
7. [设计哲学总结](#7-设计哲学总结)
8. [练习](#8-练习)
9. [知识检查](#9-知识检查)
10. [参考资源](#10-参考资源)

---

## 1. 为什么需要就地初始化

### 1.1 问题：Default::default() 为什么不行？

考虑一个包含 Mutex 的结构体：

```rust
struct MyDevice {
    data: Mutex<Vec<u8>>,
    name: CString,
}
```

你可能想这样初始化：

```rust
let dev = MyDevice {
    data: Mutex::new(Vec::new()),  // 在栈上构造
    name: CString::try_from_fmt(fmt!("my_device"))?,
};
let pinned = KBox::new(dev)?;  // 移动到堆上
```

**这行不通！** 原因：

1. `Mutex` 内部包含 C 的 `struct mutex`，初始化后会建立内部指针（等待队列等）
2. `KBox::new(dev)` 会把 `dev` 从栈上**移动**到堆上
3. 移动后，内部指针指向的还是栈上的旧地址 → **悬垂指针**

```
栈上构造:                        移动到堆后:
┌──────────────┐                ┌──────────────┐
│ Mutex        │                │ Mutex        │
│  waiters ──────→ [栈地址A]    │  waiters ──────→ [栈地址A] ← 已失效！
│  ...         │                │  ...         │
└──────────────┘                └──────────────┘
  (栈，已释放)                     (堆)
```

### 1.2 解决方案：就地初始化

不在栈上构造再移动，而是**直接在堆上的最终位置初始化**：

```
1. kmalloc 分配堆内存（未初始化）
2. 在堆上的内存地址直接初始化每个字段
3. 包装为 Pin<KBox<T>>，保证不再移动
```

这就是 pin-init 子系统要解决的问题。

---

## 2. PinInit 和 Init trait

### 2.1 两个核心 trait

```rust
// rust/kernel/init.rs

/// 可以就地初始化 T 的"初始化器"
/// 初始化后的 T 不能被移动（pinned）
pub unsafe trait PinInit<T, E = Error> {
    unsafe fn __pinned_init(self, slot: *mut T) -> Result<(), E>;
}

/// 可以就地初始化 T 的"初始化器"
/// 初始化后的 T 可以被移动
pub unsafe trait Init<T, E = Error>: PinInit<T, E> {
    unsafe fn __init(self, slot: *mut T) -> Result<(), E>;
}
```

关键设计：

| 特性 | PinInit | Init |
|------|---------|------|
| 初始化后可移动？ | ❌ 不可移动 | ✅ 可移动 |
| 用于什么类型？ | 含 Mutex、ListHead 等自引用字段 | 普通字段（u32、CString 等） |
| 继承关系 | 基础 trait | 继承自 PinInit |

**为什么 `Init: PinInit`？** 如果一个类型初始化后可以移动（Init），那它当然也可以不移动（PinInit）。反之不成立。

### 2.2 为什么用 `*mut T` 而不是 `&mut T`？

```rust
unsafe fn __pinned_init(self, slot: *mut T) -> Result<(), E>;
//                             ^^^^^^^^
//                             裸指针，不是引用
```

因为 `slot` 指向的内存是**未初始化的**。Rust 的 `&mut T` 要求指向的内存必须是有效的、已初始化的值。对未初始化内存创建 `&mut T` 是**未定义行为（UB）**。所以必须用裸指针 `*mut T`。

---

## 3. #[pin_data] 宏

### 3.1 标记哪些字段需要 pin

```rust
#[pin_data]
struct MyDevice {
    #[pin]
    data: Mutex<Vec<u8>>,   // 需要 pin（含自引用）
    name: CString,           // 不需要 pin（可移动）
}
```

`#[pin_data]` 是一个过程宏，它做两件事：

1. **生成 per-field 的 pin 信息**：记录哪些字段标记了 `#[pin]`
2. **供 `try_pin_init!` 宏使用**：在初始化时，对 `#[pin]` 字段使用 `PinInit`，对普通字段使用 `Init`

### 3.2 不标 #[pin] 会怎样？

如果你的 Mutex 字段没有标 `#[pin]`，编译器会报错——因为 Mutex 的初始化器只实现了 `PinInit`，不能当作 `Init` 使用。编译器帮你守住了正确性。

---

## 4. try_pin_init! 宏语法

### 4.1 基本语法

```rust
let initializer = try_pin_init!(MyDevice {
    data <- new_mutex!(KVec::new()),   // <- 表示就地初始化（PinInit）
    name: CString::try_from_fmt(fmt!("my_device"))?,  // : 表示普通赋值（Init）
});
```

两种语法的区别：

| 语法 | 含义 | 用于 |
|------|------|------|
| `field <- expr` | 就地初始化（调用 `__pinned_init`） | `#[pin]` 字段 |
| `field: expr` | 普通赋值（值已构造好，直接写入） | 非 pin 字段 |

### 4.2 错误处理

`try_pin_init!` 中的 `try` 意味着初始化可以失败：

```rust
let init = try_pin_init!(MyDevice {
    data <- new_mutex!(KVec::new()),
    name: CString::try_from_fmt(fmt!("my_device"))?,  // ? 可以传播错误
});
// init 的类型是 impl PinInit<MyDevice, Error>
// 注意：这里还没有分配内存！只是构建了一个"初始化器"
```

如果某个字段初始化失败，已经初始化的字段会被**正确 drop**——宏生成的代码处理了这个清理逻辑。

### 4.3 对应的非 pin 版本

```rust
// 不需要 pin 的场景
let init = try_init!(SimpleStruct {
    x: 42,
    y: compute_y()?,
});
```

---

## 5. InPlaceInit trait：完整流程

### 5.1 把初始化器变成实际对象

初始化器本身不分配内存，它只是一个"配方"。要真正创建对象，需要通过 `InPlaceInit` trait：

```rust
// rust/kernel/init.rs
pub trait InPlaceInit<T> {
    fn try_pin_init<E>(init: impl PinInit<T, E>) -> Result<Pin<Self>, E>;
    fn try_init<E>(init: impl Init<T, E>) -> Result<Self, E>;
}
```

`KBox` 实现了 `InPlaceInit`。

### 5.2 KBox::try_pin_init 的完整流程

```rust
let dev: Pin<KBox<MyDevice>> = KBox::try_pin_init(
    try_pin_init!(MyDevice {
        data <- new_mutex!(KVec::new()),
        name: CString::try_from_fmt(fmt!("my_device"))?,
    }),
    GFP_KERNEL,
)?;
```

执行步骤：

```
1. KBox 调用 kmalloc(size_of::<MyDevice>(), GFP_KERNEL)
   → 得到 *mut MyDevice（未初始化的堆内存）

2. 调用初始化器的 __pinned_init(slot: *mut MyDevice)
   ├─ 对 data 字段：调用 new_mutex! 的 __pinned_init
   │   → 直接在 slot.data 地址上初始化 Mutex
   └─ 对 name 字段：把构造好的 CString 写入 slot.name

3. 包装为 Pin<KBox<MyDevice>>
   → Pin 保证 MyDevice 不会再被移动
   → Mutex 内部指针永远有效
```

图示：

```
步骤 1：kmalloc                步骤 2：就地初始化            步骤 3：Pin 封装
┌──────────────┐            ┌──────────────┐            Pin<KBox<MyDevice>>
│ ???????????? │  ────→     │ Mutex        │  ────→     不可移动，安全使用
│ ???????????? │            │  waiters → [自身]│
│ ???????????? │            │ name: "my_dev" │
└──────────────┘            └──────────────┘
  堆上未初始化                 堆上已初始化
```

---

## 6. 结构化 pinning

### 6.1 什么是结构化 pinning？

当整个结构体被 pin 住时，每个字段是否也被 pin 取决于你的标记：

```rust
#[pin_data]
struct MyDevice {
    #[pin]
    mutex: Mutex<u32>,    // 结构化 pin → 如果 MyDevice 被 pin，mutex 也被 pin
    counter: u32,          // 非 pin → 即使 MyDevice 被 pin，counter 仍可移动
}
```

这就是为什么需要 `#[pin_data]` + `#[pin]`：**pin 不是全有或全无的，而是按字段选择的**。

### 6.2 Pin 投影

有了 `Pin<&mut MyDevice>`，如何访问字段？

- `#[pin]` 字段 → 得到 `Pin<&mut Mutex<u32>>`（保持 pin）
- 普通字段 → 得到 `&mut u32`（可以自由使用）

`#[pin_data]` 宏会生成安全的投影方法，确保 pin 语义传递正确。

### 6.3 与 C 的对比

| | C 内核 | Rust 内核 (pin-init) |
|---|---|---|
| 分配 | `kmalloc` | `KBox::try_pin_init` |
| 初始化 | 手动调用 `mutex_init()` 等 | `try_pin_init!` 宏统一处理 |
| 防止移动 | 靠程序员自觉 | `Pin<T>` 编译期保证 |
| 初始化失败 | `goto err` 手动清理 | 宏自动 drop 已初始化字段 |
| 漏初始化字段 | 运行时 bug | 编译期报错（宏检查所有字段） |

---

## 7. 设计哲学总结

| 设计决策 | 原因 |
|---------|------|
| 用 `*mut T` 而不是 `&mut T` | 内存未初始化，`&mut T` 是 UB |
| `Init: PinInit` 继承关系 | 可移动的初始化器天然是不可移动初始化器的子类型 |
| `<-` vs `:` 语法 | 明确区分就地初始化和普通赋值，防止误用 |
| `#[pin_data]` + `#[pin]` | 按字段选择 pin，不是全有或全无 |
| 初始化器是惰性的 | 先构建配方，再选择分配方式（KBox/VBox/...） |
| 失败时自动清理 | 宏生成 drop 代码，不依赖程序员 goto |

---

## 8. 练习

1. **写一个需要 pin-init 的结构体**：包含一个 `Mutex` 和一个 `ListHead`，用 `try_pin_init!` 初始化，观察 `#[pin]` 和 `<-` 语法的使用
2. **追踪宏展开**：用 `cargo expand`（如果可用）或手动推理 `try_pin_init!` 对一个两字段结构体展开后的代码
3. **对比 C 的方式**：找一个 C 驱动中初始化嵌入 mutex 的结构体，和 Rust 的 pin-init 方式对比

---

## 9. 知识检查

1. 为什么不能用 `Default::default()` 初始化包含 Mutex 的内核结构体？
2. `Init` 和 `PinInit` 的区别是什么？
3. pin-init 如何保证结构体在初始化后不被移动？

---

## 10. 参考资源

- [rust/kernel/init.rs](https://github.com/Rust-for-Linux/linux/blob/rust-next/rust/kernel/init.rs) — pin-init 核心实现
- [rust/kernel/init/macros.rs](https://github.com/Rust-for-Linux/linux/blob/rust-next/rust/kernel/init/macros.rs) — 宏定义
- [pin-init crate (standalone)](https://github.com/Rust-for-Linux/pin-init) — 独立版本，可在用户态实验
- [Rust 官方 Pin 文档](https://doc.rust-lang.org/std/pin/index.html) — 理解 Pin 的基础
