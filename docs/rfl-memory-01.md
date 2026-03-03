# 内核内存分配的 Rust 封装

> 模块：rfl-memory-01 | 难度：进阶 | 预计时间：3 小时
> 前置要求：[kernel Crate：核心抽象](rfl-arch-02.md)

---

## 目录

1. [GFP Flags：分配上下文标记](#1-gfp-flags分配上下文标记)
2. [三种分配器：Kmalloc / Vmalloc / KVmalloc](#2-三种分配器kmalloc--vmalloc--kvmalloc)
3. [Allocator trait：零开销抽象](#3-allocator-trait零开销抽象)
4. [KBox / VBox / KVBox](#4-kbox--vbox--kvbox)
5. [KVec 与 fallible push](#5-kvec-与-fallible-push)
6. [设计哲学总结](#6-设计哲学总结)
7. [练习](#7-练习)
8. [知识检查](#8-知识检查)
9. [参考资源](#9-参考资源)

---

## 1. GFP Flags：分配上下文标记

### 1.1 Flags 类型

```rust
// rust/kernel/alloc.rs
pub struct Flags(u32);
```

`Flags` 是对 C 内核 GFP（Get Free Pages）标志的零开销封装。`#[repr(transparent)]` 保证 `Flags(u32)` 和 `u32` 内存布局完全一致。

### 1.2 三个核心 GFP 标志

| 标志 | 值 | 含义 | 使用场景 |
|------|---|------|----------|
| `GFP_KERNEL` | `___GFP_RECLAIM \| ___GFP_IO \| ___GFP_FS` | 常规分配，**可以睡眠** | 进程上下文（大多数场景） |
| `GFP_ATOMIC` | `___GFP_HIGH \| ___GFP_KSWAPD_RECLAIM` | 紧急分配，**不能睡眠** | 中断上下文、SpinLock 持有期间 |
| `GFP_NOWAIT` | `___GFP_KSWAPD_RECLAIM` | 尽量分配，不能睡眠，容忍失败 | 可以容忍失败的快速路径 |

### 1.3 为什么需要区分？

内核的内存分配器可能需要**回收内存**来满足请求。回收过程可能涉及：

- 写脏页到磁盘（`___GFP_IO`）
- 与文件系统交互（`___GFP_FS`）
- 等待 I/O 完成（睡眠）

在中断上下文或持有 SpinLock 时，**不能睡眠**。所以：

```
进程上下文（可睡眠）
  → GFP_KERNEL  → 分配器可以做任何回收 → 成功率高

中断/SpinLock（不可睡眠）
  → GFP_ATOMIC  → 只用预留内存 → 可能失败
```

### 1.4 Flags 的运算

```rust
impl core::ops::BitOr for Flags {
    type Output = Self;
    fn bitor(self, rhs: Self) -> Self::Output {
        Self(self.0 | rhs.0)
    }
}
```

支持 `|` 组合多个标志，和 C 的用法一致：`GFP_KERNEL | __GFP_ZERO`。

---

## 2. 三种分配器：Kmalloc / Vmalloc / KVmalloc

| 分配器 | 内存类型 | 适用大小 | 特点 |
|--------|----------|----------|------|
| **Kmalloc** | 物理连续 | 小块（≤ 几页） | 最快，但大块分配容易失败 |
| **Vmalloc** | 虚拟连续 | 大块 | 通过页表映射，物理不连续 |
| **KVmalloc** | 自动选择 | 任意 | 先试 Kmalloc，失败后 fallback 到 Vmalloc |

### 2.1 Kmalloc

从 slab 分配器获取物理连续内存。内核中最常用的分配方式。

```
┌──────────────────────────────┐
│       物理内存                │
│  ┌─────┬─────┬─────┬─────┐  │
│  │ p1  │ p2  │ p3  │ p4  │  │  ← 连续的物理页
│  └─────┴─────┴─────┴─────┘  │
└──────────────────────────────┘
```

优点：快，无页表开销，适合 DMA。
缺点：内存碎片化后，大块连续物理内存难以分配。

### 2.2 Vmalloc

分配虚拟连续但物理不连续的内存。

```
虚拟地址空间:  [ v1 ][ v2 ][ v3 ][ v4 ]  ← 连续
                 │      │      │      │
物理内存:      [ p7 ]  [p23]  [ p3 ]  [p51]  ← 不连续
```

优点：大块分配不受物理碎片影响。
缺点：需要页表映射，有 TLB 开销。

### 2.3 KVmalloc

智能选择：先尝试 Kmalloc（快），失败后自动 fallback 到 Vmalloc（保证成功率）。

```rust
// 简化逻辑
fn kvmalloc(size: usize, flags: Flags) -> *mut u8 {
    let ptr = kmalloc(size, flags);
    if !ptr.is_null() {
        return ptr;      // Kmalloc 成功，用物理连续内存
    }
    vmalloc(size, flags) // 退而求其次，用虚拟连续内存
}
```

---

## 3. Allocator trait：零开销抽象

### 3.1 trait 定义

```rust
pub unsafe trait Allocator {
    /// 分配或重新分配内存
    unsafe fn realloc(
        ptr: Option<NonNull<u8>>,
        layout: Layout,
        old_layout: Layout,
        flags: Flags,
    ) -> Result<NonNull<[u8]>, AllocError>;

    /// 释放内存
    unsafe fn free(ptr: NonNull<u8>, layout: Layout);
}
```

### 3.2 ZST（零大小类型）设计

三个分配器都是**零大小类型**：

```rust
pub struct Kmalloc;   // 没有字段 → 大小为 0
pub struct Vmalloc;   // 没有字段 → 大小为 0
pub struct KVmalloc;  // 没有字段 → 大小为 0
```

为什么用 ZST？

1. **零存储开销** — `KBox<T, Kmalloc>` 和 `*mut T` 大小相同，分配器不占空间
2. **编译时分派** — Rust 的单态化把 `Allocator` trait 方法内联，没有虚函数调用
3. **类型区分** — `KBox<T>` 用 Kmalloc，`VBox<T>` 用 Vmalloc，编译器知道该调哪个 free

```
KBox<T> = Box<T, Kmalloc>
  └─ sizeof = sizeof(*mut T)  // Kmalloc 是 ZST，不占空间
  └─ drop 时调用 Kmalloc::free → kfree()

VBox<T> = Box<T, Vmalloc>
  └─ sizeof = sizeof(*mut T)  // 同样
  └─ drop 时调用 Vmalloc::free → vfree()
```

### 3.3 Allocator 方法没有 &self

注意 `realloc` 和 `free` 的签名：没有 `&self` 参数。

```rust
// 标准库
fn allocate(&self, layout: Layout) -> Result<...>;  // 有 &self

// 内核
unsafe fn realloc(ptr: ..., layout: ..., flags: ...) -> Result<...>;  // 无 self
```

因为分配器是 ZST，没有实例数据，不需要 `&self`。所有行为在编译时确定，不需要运行时状态。这是**零开销抽象**的典型范例。

### 3.4 什么时候适合 ZST 设计？

满足以下条件时，可以考虑 ZST 设计：

- **无状态** — 所有行为由类型本身决定，不需要运行时数据
- **策略选择** — 多个"策略"之间的选择可以在编译时确定
- **嵌入其他类型** — 作为泛型参数嵌入时零开销

反例：如果分配器需要记录自己的内存池地址、统计信息等，就不能是 ZST。例如自定义的 bump allocator 需要存储当前指针位置，就必须有字段。

---

## 4. KBox / VBox / KVBox

```rust
// rust/kernel/alloc/kbox.rs
pub struct Box<T: ?Sized, A: Allocator>(NonNull<T>, PhantomData<A>);

// 类型别名
pub type KBox<T> = Box<T, Kmalloc>;
pub type VBox<T> = Box<T, Vmalloc>;
pub type KVBox<T> = Box<T, KVmalloc>;
```

### 4.1 和标准库 Box 的对比

| | std::Box | kernel::KBox |
|---|---------|-------------|
| 分配失败 | panic | 返回 `Result` |
| 分配标志 | 无 | 需要 `Flags` 参数 |
| 分配器 | 全局分配器 | 类型参数 `A: Allocator` |
| drop | 调用全局 dealloc | 调用 `A::free()` |

### 4.2 创建方式

```rust
// 标准库
let b = Box::new(42);                    // 永远成功

// 内核
let b = KBox::new(42, GFP_KERNEL)?;      // 可能失败
let b = VBox::new(big_data, GFP_KERNEL)?; // 大对象用 Vmalloc
```

---

## 5. KVec 与 fallible push

```rust
// rust/kernel/alloc/kvec.rs
pub struct Vec<T, A: Allocator> {
    ptr: NonNull<T>,
    len: usize,
    layout: ArrayLayout<T>,  // 包含 capacity 信息
}

pub type KVec<T> = Vec<T, Kmalloc>;
pub type VVec<T> = Vec<T, Vmalloc>;
pub type KVVec<T> = Vec<T, KVmalloc>;
```

### 5.1 Fallible push

```rust
// 标准库
v.push(42);                  // 分配失败 → panic

// 内核
v.push(42, GFP_KERNEL)?;    // 分配失败 → 返回 Err
```

每次可能涉及内存分配的操作都要传 `Flags` 并处理 `Result`。

### 5.2 push_within_capacity

```rust
pub fn push_within_capacity(&mut self, v: T) -> Result<(), T> {
    if self.len < self.capacity() {
        // 不需要分配，直接放入
        unsafe { self.as_mut_ptr().add(self.len).write(v) };
        self.len += 1;
        Ok(())
    } else {
        Err(v)  // 容量不够，把 v 还回去
    }
}
```

特点：
- **不需要 Flags 参数** — 不做分配
- **不会失败**（capacity 够的话）
- 适合在不能分配的上下文中使用（中断、SpinLock 内）
- 失败时返回 `Err(v)` — 把值还回调用方，不丢失数据

### 5.3 NUMA 支持

```rust
pub struct NumaNode(i32);

pub const NUMA_NO_NODE: NumaNode = NumaNode(-1);
```

Allocator trait 的实现支持 NUMA node 参数，可以指定在哪个 NUMA 节点上分配内存。默认 `NUMA_NO_NODE` 由内核自动选择。

---

## 6. 设计哲学总结

```
标准库的假设               内核的现实               kernel crate 的应对
────────────             ─────────             ──────────────
分配永远成功              分配可能失败              所有分配返回 Result
一种分配方式              上下文敏感              每次分配带 Flags 参数
一个全局分配器            三种分配器              Allocator trait + ZST
分配器有状态              分配器无状态             ZST → 零存储开销
Box/Vec 固定分配器        按需选择分配器           泛型参数 A: Allocator
```

**一句话总结：内核的每一次内存分配都必须回答两个问题 ——"可以睡眠吗？"（Flags）和"用哪个分配器？"（Allocator 泛型参数）。**

---

## 7. 练习

### 练习 1：追踪 KVec::push 的分配流程

从 `KVec::push(42, GFP_KERNEL)` 出发，追踪完整调用链直到 C 的 `krealloc`。记录中间经过的每一层抽象。

### 练习 2：GFP 标志选择

判断以下场景应该使用哪种 GFP 标志：

- A: 驱动 probe 函数中分配配置结构体
- B: 网卡中断处理中分配 sk_buff
- C: 持有 SpinLock 时需要分配临时缓冲区

### 练习 3：Box 类型选择

说明以下场景应该用 KBox、VBox 还是 KVBox，为什么：

- 分配一个 64 字节的设备寄存器映射结构
- 分配一个 1MB 的 DMA 缓冲区
- 分配一个大小不确定的用户数据缓冲区

---

## 8. 知识检查

1. **为什么内核的内存分配必须是 fallible 的？如果在中断上下文中使用 `GFP_KERNEL` 会怎样？**

2. **Kmalloc、Vmalloc、KVmalloc 三者的区别是什么？分别适合什么场景？**

3. **为什么 Allocator trait 的方法没有 `&self` 参数？这种 ZST 设计带来了什么好处？**

---

## 9. 参考资源

| 资源 | 说明 |
|------|------|
| `rust/kernel/alloc.rs` | Flags 类型与 Allocator trait |
| `rust/kernel/alloc/allocator.rs` | Kmalloc/Vmalloc/KVmalloc 实现 |
| `rust/kernel/alloc/kbox.rs` | Box 类型与 KBox/VBox/KVBox 别名 |
| `rust/kernel/alloc/kvec.rs` | Vec 类型与 KVec/VVec/KVVec 别名 |
| [Kernel memory allocation guide](https://docs.kernel.org/core-api/memory-allocation.html) | 内核官方内存分配文档 |
