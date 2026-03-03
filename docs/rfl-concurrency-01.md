# 内核并发原语的 Rust 封装

> 模块：rfl-concurrency-01 | 难度：进阶 | 预计时间：4 小时
> 前置要求：[kernel Crate：核心抽象](rfl-arch-02.md)

---

## 目录

1. [设计总览：Lock\<T, B\> 策略模式](#1-设计总览lockt-b-策略模式)
2. [Backend trait：统一的锁后端](#2-backend-trait统一的锁后端)
3. [Guard：RAII 守卫](#3-guardraii-守卫)
4. [Mutex vs SpinLock](#4-mutex-vs-spinlock)
5. [CondVar：条件变量](#5-condvar条件变量)
6. [LockedBy：外部锁保护](#6-lockedby外部锁保护)
7. [设计哲学总结](#7-设计哲学总结)
8. [练习](#8-练习)
9. [知识检查](#9-知识检查)
10. [参考资源](#10-参考资源)

---

## 1. 设计总览：Lock\<T, B\> 策略模式

内核有两种主要的锁：Mutex（互斥锁，可睡眠）和 SpinLock（自旋锁，不可睡眠）。它们的 API 模式高度相似：加锁 → 访问数据 → 解锁。

kernel crate 用**策略模式（Strategy Pattern）**统一了两者：

```rust
// rust/kernel/sync/lock.rs
pub struct Lock<T: ?Sized, B: Backend> {
    state: B::State,           // 锁的内部状态（由后端决定）
    data: UnsafeCell<T>,       // 被保护的数据
}
```

```
┌──────────────────────────────┐
│    Lock<T, B: Backend>       │  ← 通用框架
│  ┌─────────┬──────────────┐  │
│  │  state   │  data: T     │  │
│  │(B::State)│(UnsafeCell)  │  │
│  └─────────┴──────────────┘  │
└──────────────────────────────┘
         │
   ┌─────┴──────┐
   ▼             ▼
MutexBackend  SpinLockBackend   ← 具体实现
```

关键设计：**数据在锁内部**（`UnsafeCell<T>`）。和 C 不同——C 的 mutex 和它保护的数据是分离的，靠程序员"记住"哪把锁保护哪个数据。Rust 把数据**物理上**放在锁里面，编译器就能保证：**不加锁就不能访问数据。**

---

## 2. Backend trait：统一的锁后端

```rust
pub unsafe trait Backend {
    /// 锁的内部状态类型
    type State;

    /// 加锁后的上下文（SpinLock 可能需要保存 flags）
    type GuardState;

    /// 初始化锁（配合 pin-init 使用）
    unsafe fn init(
        ptr: *mut Self::State,
        name: *const c_char,
        key: *mut bindings::lock_class_key,
    );

    /// 加锁，返回上下文
    unsafe fn lock(ptr: *mut Self::State) -> Self::GuardState;

    /// 解锁
    unsafe fn unlock(ptr: *mut Self::State, guard_state: &Self::GuardState);

    /// 非阻塞尝试加锁
    unsafe fn try_lock(ptr: *mut Self::State) -> Option<Self::GuardState>;
}
```

这个设计的好处：

| 好处 | 说明 |
|------|------|
| **代码复用** | Lock、Guard 的公共逻辑只写一份 |
| **稳定接口** | 上层代码对 `Lock<T, B>` 编程，不关心具体用的是哪种锁 |
| **易于扩展** | 新增锁类型（如 RwLock）只需实现 Backend trait |
| **零开销** | Backend 是 trait，编译时单态化，没有虚函数开销 |

为什么是 `unsafe trait`？因为实现方必须保证：加锁/解锁的配对正确性，lock 返回后确实持有锁，unlock 确实释放了锁。

---

## 3. Guard：RAII 守卫

```rust
#[must_use]
pub struct Guard<'a, T: ?Sized, B: Backend> {
    lock: &'a Lock<T, B>,
    state: B::GuardState,
    _not_send: NotThreadSafe,
}
```

### 3.1 RAII 的三个保证

```rust
// Deref — 通过 Guard 读取数据
impl<T: ?Sized, B: Backend> Deref for Guard<'_, T, B> {
    type Target = T;
    fn deref(&self) -> &T {
        unsafe { &*self.lock.data.get() }
    }
}

// DerefMut — 通过 Guard 修改数据（注意 T: Unpin 约束）
impl<T: ?Sized + Unpin, B: Backend> DerefMut for Guard<'_, T, B> {
    fn deref_mut(&mut self) -> &mut T {
        unsafe { &mut *self.lock.data.get() }
    }
}

// Drop — 离开作用域自动解锁
impl<T: ?Sized, B: Backend> Drop for Guard<'_, T, B> {
    fn drop(&mut self) {
        unsafe { B::unlock(self.lock.state.as_mut_ptr(), &self.state) };
    }
}
```

### 3.2 Guard 为什么可以返回 &T？

```
Guard 持有 &'a Lock<T, B>
  → Lock 内部有 UnsafeCell<T>
    → Guard 的 Deref 返回 &T
      → &T 的生命周期绑定到 Guard 的生命周期 'a
        → Guard drop 时解锁
          → 引用失效 = 锁释放，同步保证
```

Guard 存在 → 锁被持有 → 可以安全访问数据。Guard 被 drop → 锁被释放 → 引用自动失效。这是**类型驱动的正确性保证**。

### 3.3 三个细节

| 特性 | 作用 |
|------|------|
| `#[must_use]` | 如果 `lock()` 的返回值被忽略，编译器警告（`try_lock` 尤其重要） |
| `NotThreadSafe` | Guard 不能跨线程传递（Send），防止"在 A 线程加锁，在 B 线程解锁" |
| `T: Unpin` for DerefMut | 有 `&mut T` 就能 `mem::swap()` 移走对象，对 pinned 的数据（如含锁的结构体）这是不安全的 |

---

## 4. Mutex vs SpinLock

### 4.1 Mutex

```rust
// rust/kernel/sync/lock/mutex.rs
pub type Mutex<T> = Lock<T, MutexBackend>;

pub struct MutexBackend;

unsafe impl Backend for MutexBackend {
    type State = Opaque<bindings::mutex>;
    type GuardState = ();

    unsafe fn lock(ptr: *mut Self::State) -> Self::GuardState {
        bindings::mutex_lock(/* ... */);
    }

    unsafe fn unlock(ptr: *mut Self::State, _: &Self::GuardState) {
        bindings::mutex_unlock(/* ... */);
    }
}
```

特点：
- 底层调用 C 的 `mutex_lock` / `mutex_unlock`
- **可以睡眠** — 等不到锁时，当前线程进入睡眠队列
- GuardState 是 `()` — 不需要额外上下文

### 4.2 SpinLock

```rust
// rust/kernel/sync/lock/spinlock.rs
pub type SpinLock<T> = Lock<T, SpinLockBackend>;

pub struct SpinLockBackend;

unsafe impl Backend for SpinLockBackend {
    type State = Opaque<bindings::spinlock_t>;
    type GuardState = ();

    unsafe fn lock(ptr: *mut Self::State) -> Self::GuardState {
        bindings::spin_lock(/* ... */);
    }

    unsafe fn unlock(ptr: *mut Self::State, _: &Self::GuardState) {
        bindings::spin_unlock(/* ... */);
    }
}
```

特点：
- 底层调用 C 的 `spin_lock` / `spin_unlock`
- **不能睡眠** — 忙等待（busy-wait），适合短临界区
- 获取自旋锁时 **禁用内核抢占**（preempt_disable）

### 4.3 为什么 SpinLock 持有期间不能睡眠？

```
CPU 0
──────
spin_lock(&lock)
  → preempt_disable()           // 关闭抢占
  → 进入临界区
  → 调用了某个可睡眠函数
    → schedule()  ← 💥 BUG！
```

关键链条：

1. `spin_lock` 会调用 `preempt_disable()`，禁用当前 CPU 的抢占
2. 如果在持锁期间调用了可以睡眠的函数（如 `kmalloc(GFP_KERNEL)`、`mutex_lock`）
3. 这些函数内部会调用 `schedule()` 让出 CPU
4. 但 `schedule()` 发现抢占被禁用 → **BUG / 死锁**

**即使是多核系统**，问题不在"没有其他 CPU 来运行"，而在于**当前 CPU 自身进入了矛盾状态**：自旋锁要求"禁用抢占，快速完成"，睡眠要求"让出 CPU，等待唤醒"。

### 4.4 使用场景对比

| | Mutex | SpinLock |
|---|-------|----------|
| **阻塞方式** | 睡眠（进入等待队列） | 自旋（busy-wait） |
| **能否睡眠** | ✅ 可以 | ❌ 不可以 |
| **临界区长度** | 可以长 | 应该短 |
| **中断上下文** | ❌ 不能用 | ✅ 可以用 |
| **分配用的 GFP** | `GFP_KERNEL` | `GFP_ATOMIC` |
| **典型场景** | 文件操作、设备配置 | 中断处理、计数器更新 |

---

## 5. CondVar：条件变量

```rust
// rust/kernel/sync/condvar.rs
pub struct CondVar {
    inner: Opaque<bindings::wait_queue_head>,
}
```

CondVar 是对 C 内核 `wait_queue_head` 的封装，用于"等待某个条件成立"。

### 5.1 核心 API

```rust
impl CondVar {
    /// 等待条件（可中断）
    pub fn wait_interruptible<T: ?Sized, B: Backend>(
        &self,
        guard: &mut Guard<'_, T, B>,
    ) -> Result;

    /// 唤醒一个等待者
    pub fn notify_one(&self);

    /// 唤醒所有等待者
    pub fn notify_all(&self);
}
```

### 5.2 为什么 wait 要传 Guard？

```rust
fn wait_interruptible(&self, guard: &mut Guard<'_, T, B>) -> Result;
```

Guard 参数是**持有锁的证明**。wait 的典型模式：

1. 持有锁，检查条件
2. 条件不满足 → 释放锁，进入等待
3. 被唤醒后 → 重新获取锁，返回

传入 Guard 让编译器保证：调用 wait 时你确实持有锁。这避免了 C 中忘记加锁就 wait 的 bug。

---

## 6. LockedBy：外部锁保护

```rust
// rust/kernel/sync/locked_by.rs
pub struct LockedBy<T: ?Sized, U: ?Sized> {
    owner: *const U,    // 指向锁中数据的指针（用于运行时检查）
    data: UnsafeCell<T>,
}
```

### 6.1 使用场景

有时候数据 T 不在锁内部，而是"由某个外部的锁保护"。比如：

```
Mutex<A> 保护 A
  └─ A 中有个指针 → B
                     └─ B 本身不在 Mutex 里，但也受该 Mutex 保护
```

LockedBy 让编译器知道这种关系：

```rust
// T: 要保护的数据
// U: 锁中的数据类型（用于身份验证）
let locked_data: LockedBy<MyData, LockOwner> = ...;

// 访问时，必须提供 &LockOwner 的引用（证明你持有锁）
let data = locked_data.access(&owner_ref);
```

### 6.2 运行时检查

LockedBy 在 `access()` 时做**指针比较**：传入的引用是否指向构造时记录的 owner。这是运行时检查（debug 时 panic），而不是编译时检查——但它是对 C "文档约定" 的显著改进。

---

## 7. 设计哲学总结

```
C 内核的做法                    Rust kernel crate 的做法
─────────                    ─────────────────
锁和数据分离，靠文档关联        数据在锁内部（UnsafeCell<T>）
手动 lock/unlock 配对          Guard RAII 自动解锁
忘记解锁 → 死锁               编译器保证 Drop
不同锁各自独立 API             Backend trait 统一抽象
SpinLock 不能睡眠靠文档说明     类型系统 + GFP flags 辅助
```

核心思想：**把 C 内核靠"约定"和"文档"保证的规则，编码到类型系统中，让编译器帮你检查。**

---

## 8. 练习

### 练习 1：对比 Rust Mutex 和 C mutex API

阅读 `rust/kernel/sync/lock/mutex.rs` 和 `include/linux/mutex.h`，列出 C mutex 中哪些不变量在 Rust 中变成了编译时保证。

### 练习 2：设计一个 SpinLock 保护的结构体

写一个 Rust 结构体，内部用 SpinLock 保护一个计数器。展示：

- 如何在持锁期间读写数据
- 为什么不持锁时无法访问数据（编译错误）
- Guard 出作用域后自动解锁

### 练习 3：解释 LockedBy 的使用场景

在内核源码中找到 LockedBy 的实际使用案例，解释为什么不直接把数据放在锁里。

---

## 9. 知识检查

1. **Guard 类型如何保证锁一定会被释放？**

2. **在 Rust 中，为什么不持有锁就无法访问 Mutex 保护的数据？请描述类型系统是如何防止这一点的。**

3. **SpinLock 和 Mutex 在内核中的使用场景有什么区别？为什么 SpinLock 持有期间不能睡眠？**

---

## 10. 参考资源

| 资源 | 说明 |
|------|------|
| `rust/kernel/sync/lock.rs` | Lock<T,B> 和 Guard 泛型框架 |
| `rust/kernel/sync/lock/mutex.rs` | Mutex 后端实现 |
| `rust/kernel/sync/lock/spinlock.rs` | SpinLock 后端实现 |
| `rust/kernel/sync/condvar.rs` | CondVar 条件变量封装 |
| `rust/kernel/sync/locked_by.rs` | LockedBy 外部锁保护模式 |
| [LWN: Rust locking abstractions](https://lwn.net/Articles/863459/) | Rust 锁抽象的设计讨论 |
