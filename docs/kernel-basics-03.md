# 内核数据结构与模式

> 模块：kernel-basics-03 | 难度：入门 | 预计时间：3 小时
> 前置要求：[内核架构概览](kernel-basics-01.md)

---

## 目录

1. [container_of 宏](#1-container_of-宏)
2. [侵入式链表（struct list_head）](#2-侵入式链表struct-list_head)
3. [task_struct 中的链表实例](#3-task_struct-中的链表实例)
4. [kref 引用计数](#4-kref-引用计数)
5. [errno 错误约定](#5-errno-错误约定)
6. [其他重要数据结构](#6-其他重要数据结构)
7. [练习](#7-练习)
8. [知识检查](#8-知识检查)
9. [参考资源](#9-参考资源)

---

## 1. container_of 宏

`container_of` 是内核中使用最广泛的宏之一。它的作用是：**已知一个结构体成员的指针，反推出包含它的结构体的指针。**

### 为什么需要它？

内核的通用数据结构（如链表）只操作嵌入的小结构体（如 `list_head`），不知道外层结构体是什么。当你从链表中取出一个 `list_head` 指针时，需要通过 `container_of` 找回外层的完整结构体。

### 内存布局与原理

```c
struct task_struct {
    pid_t           pid;        // offset: 0
    char            comm[16];   // offset: 4
    struct list_head tasks;     // offset: 20  ← 你拿到了这个指针
    int             prio;       // offset: 36
};
```

```
内存布局：

地址低 ──────────────────────────────────────────► 地址高

┌──────┬──────────────┬──────────────────┬──────┐
│ pid  │  comm[16]    │  tasks           │ prio │
│ 4B   │  16B         │  (list_head)16B  │ 4B   │
└──────┴──────────────┴──────────────────┴──────┘
▲                      ▲
│                      │
│ container_of 要找的   │ 你手上有的指针
│ （结构体起始地址）     │ （成员地址）
│                      │
│◄──── offset = 20 ────┤

结构体起始地址 = 成员地址 - 成员在结构体中的偏移量
```

### 实现

```c
// include/linux/container_of.h
#define container_of(ptr, type, member) ({                  \
    void *__mptr = (void *)(ptr);                           \
    ((type *)(__mptr - offsetof(type, member)));             \
})
```

**拆解**：
- `ptr`：成员的指针（如 `list_head *`）
- `type`：外层结构体类型（如 `struct task_struct`）
- `member`：成员在结构体中的名称（如 `tasks`）
- `offsetof(type, member)`：编译期计算成员偏移量
- 最终：`成员地址 - 偏移量 = 结构体起始地址`

### 使用示例

```c
// 已知 list_head 指针，反推 task_struct 指针
struct list_head *pos = /* 从链表中获取 */;
struct task_struct *task = container_of(pos, struct task_struct, tasks);
printf("PID: %d\n", task->pid);
```

> **这是一个纯指针算术操作**——不分配内存、不做类型检查（C 的限制），只是简单的地址减法。如果传入错误的类型，编译器不会报错，但运行时会得到垃圾数据。这是 C 语言类型安全缺失的一个典型例子，也是 Rust 抽象想要解决的问题之一。

---

## 2. 侵入式链表（struct list_head）

### 2.1 两种链表设计

**非侵入式链表**（标准库风格）：

```
链表节点持有数据指针，数据不知道自己在链表中

┌──────────┐     ┌──────────┐     ┌──────────┐
│ next ────────►│ next ────────►│ next ──► NULL
│ prev     │◄────── prev    │◄────── prev    │
│ data ─┐  │     │ data ─┐  │     │ data ─┐  │
└───────│──┘     └───────│──┘     └───────│──┘
        │                │                │
        ▼                ▼                ▼
  ┌──────────┐     ┌──────────┐     ┌──────────┐
  │ task A   │     │ task B   │     │ task C   │
  └──────────┘     └──────────┘     └──────────┘
```

**侵入式链表**（Linux 内核风格）：

```
数据结构内部嵌入链表节点，节点直接是数据的一部分

┌─────────────────┐     ┌─────────────────┐     ┌─────────────────┐
│ pid = 100       │     │ pid = 200       │     │ pid = 300       │
│ comm = "bash"   │     │ comm = "vim"    │     │ comm = "gcc"    │
│ ┌─────────────┐ │     │ ┌─────────────┐ │     │ ┌─────────────┐ │
│ │ tasks.next ───────────► tasks.next ───────────► tasks.next ──────► ...
│ │ tasks.prev  │◄─────────── tasks.prev │◄─────────── tasks.prev │ │
│ └─────────────┘ │     │ └─────────────┘ │     │ └─────────────┘ │
│ prio = 120      │     │ prio = 100      │     │ prio = 139      │
└─────────────────┘     └─────────────────┘     └─────────────────┘
     task_struct              task_struct              task_struct
```

### 2.2 为什么内核选择侵入式？

| 特性 | 非侵入式 | 侵入式（Linux） |
|------|---------|----------------|
| 内存分配 | 每个节点需额外 `malloc` | 零额外分配（嵌入在结构体内） |
| 缓存友好性 | 节点和数据可能不相邻 | 节点就是数据的一部分，天然相邻 |
| 多链表成员 | 每个链表都需要独立的包装器 | 嵌入多个 `list_head` 即可 |
| 删除操作 | 需要遍历找节点 | O(1)，直接操作 prev/next |
| 类型通用性 | 需要泛型或 `void*` | `container_of` 解决 |

### 2.3 struct list_head 定义

```c
// include/linux/types.h
struct list_head {
    struct list_head *next;
    struct list_head *prev;
};
```

只有两个指针，共 16 字节（64 位系统）。它是一个**双向循环链表**——最后一个节点的 `next` 指向头节点，头节点的 `prev` 指向最后一个节点。

```
循环结构：

        ┌─────────────────────────────────────────────┐
        │                                              │
        ▼                                              │
  ┌──────────┐     ┌──────────┐     ┌──────────┐      │
  │   head   │────►│  node1   │────►│  node2   │──────┘
  │          │◄────│          │◄────│          │
  └──────────┘     └──────────┘     └──────────┘
  ▲                                              │
  │                                              │
  └──────────────────────────────────────────────┘
```

### 2.4 常用操作

```c
// 初始化
LIST_HEAD(my_list);                           // 声明并初始化一个链表头
INIT_LIST_HEAD(&my_struct.list);              // 初始化嵌入的 list_head

// 添加
list_add(&new->list, &head);                  // 在 head 后面插入（头插法）
list_add_tail(&new->list, &head);             // 在 head 前面插入（尾插法）

// 删除
list_del(&entry->list);                       // 从链表中移除

// 遍历
list_for_each_entry(pos, &head, member) {     // 遍历每个元素
    // pos 是 container_of 后的结构体指针
    // member 是 list_head 在结构体中的字段名
}

// 判空
list_empty(&head);                            // 链表是否为空
```

### 2.5 一个结构体中嵌入多个 list_head

这是侵入式链表的一个强大特性——同一个对象可以同时属于多个链表：

```c
struct task_struct {
    struct list_head tasks;       // 全局进程链表
    struct list_head children;    // 子进程链表
    struct list_head sibling;     // 兄弟进程链表
    struct list_head run_list;    // 调度运行队列
    // ...
};
```

```
同一个 task_struct 同时在多个链表中：

全局进程链表（通过 tasks 字段）：
  init ◄──► bash ◄──► vim ◄──► gcc ◄──► ...

bash 的子进程链表（通过 children 字段）：
  bash.children ◄──► vim.sibling ◄──► gcc.sibling

调度队列（通过 run_list 字段）：
  runqueue ◄──► bash.run_list ◄──► gcc.run_list
```

> 每个 `list_head` 只有 16 字节，所以嵌入多个链表头的内存开销很小，但换来了 O(1) 的插入/删除和零额外内存分配。

---

## 3. task_struct 中的链表实例

`task_struct` 是内核中最重要的结构体（定义在 `include/linux/sched.h`），它至少包含 **14 个 `list_head`** 成员：

| 字段 | 用途 |
|------|------|
| `tasks` | 全局进程链表，串联系统中所有进程 |
| `children` | 该进程的子进程链表头 |
| `sibling` | 兄弟进程链表（挂在父进程的 children 上） |
| `thread_group` | 线程组（同一进程的所有线程） |
| `thread_node` | 信号相关的线程节点 |
| `ptraced` | 被 ptrace 追踪的进程链表 |
| `ptrace_entry` | ptrace 链表节点 |
| `cg_list` | cgroup 相关链表 |
| `pi_state_list` | 优先级继承状态链表 |
| `numa_entry` | NUMA 迁移链表 |
| `rcu_tasks_holdout_list` | RCU tasks 相关 |
| `cpu_timers[3]` | CPU 定时器链表（3 个） |

这展示了侵入式链表在实际代码中的威力——一个对象可以高效地参与十几个不同的数据结构，而无需任何额外的内存分配。

---

## 4. kref 引用计数

### 4.1 问题：何时释放内存？

当多个组件共享同一个对象时，如何知道何时安全释放它？

```
组件 A ──┐
         │
组件 B ──┼──► 共享对象    谁来释放？什么时候释放？
         │
组件 C ──┘
```

答案：**引用计数**——每个使用者增加计数，用完减少计数，计数归零时释放。

### 4.2 kref 结构

```c
// include/linux/kref.h
struct kref {
    refcount_t refcount;    // 原子计数器
};
```

### 4.3 API

```c
// 初始化（计数设为 1）
kref_init(&obj->kref);

// 增加引用（计数 +1）
kref_get(&obj->kref);

// 减少引用（计数 -1，归零时调用 release 回调）
kref_put(&obj->kref, my_release_fn);
```

### 4.4 完整示例

```c
struct my_device {
    char name[32];
    int  status;
    struct kref kref;      // 嵌入引用计数
};

// 释放函数（计数归零时被调用）
static void my_device_release(struct kref *kref)
{
    struct my_device *dev = container_of(kref, struct my_device, kref);
    pr_info("releasing device: %s\n", dev->name);
    kfree(dev);
}

// 使用流程
void example(void)
{
    struct my_device *dev = kmalloc(sizeof(*dev), GFP_KERNEL);
    kref_init(&dev->kref);          // refcount = 1

    // 组件 B 也要用
    kref_get(&dev->kref);           // refcount = 2

    // 组件 A 用完了
    kref_put(&dev->kref, my_device_release);  // refcount = 1（不释放）

    // 组件 B 也用完了
    kref_put(&dev->kref, my_device_release);  // refcount = 0 → 调用 release → kfree
}
```

### 4.5 kref 的生命周期

```
kref_init()     kref_get()     kref_put()     kref_put()
    │               │              │              │
    ▼               ▼              ▼              ▼
 refcount=1  →  refcount=2  →  refcount=1  →  refcount=0
                                                  │
                                                  ▼
                                          release() → kfree()
```

> **与 Rust 的联系**：kref 解决的问题正是 Rust `Arc`（原子引用计数）解决的问题。但 kref 完全靠程序员手动调用 `kref_get/kref_put`，忘记调用就会内存泄漏或 use-after-free。Rust 的 `Arc` 在编译期和 `Drop` trait 自动管理引用计数，从根本上杜绝这类错误。

---

## 5. errno 错误约定

### 5.1 内核函数的错误返回

内核 C 代码使用**负整数**表示错误，正整数或零表示成功：

```c
// 返回 0 表示成功
int my_init(void) {
    void *ptr = kmalloc(1024, GFP_KERNEL);
    if (!ptr)
        return -ENOMEM;   // 内存不足
    return 0;              // 成功
}
```

### 5.2 常见 errno 值

| errno | 值 | 含义 |
|-------|-----|------|
| `EPERM` | -1 | 操作不允许 |
| `ENOENT` | -2 | 文件不存在 |
| `EINTR` | -4 | 系统调用被信号中断 |
| `EIO` | -5 | I/O 错误 |
| `ENOMEM` | -12 | 内存不足 |
| `EACCES` | -13 | 权限不足 |
| `EBUSY` | -16 | 设备或资源忙 |
| `EEXIST` | -17 | 文件已存在 |
| `ENODEV` | -19 | 没有该设备 |
| `EINVAL` | -22 | 无效参数 |
| `ENOSPC` | -28 | 磁盘空间不足 |
| `ENOSYS` | -38 | 函数未实现 |
| `ENODATA` | -61 | 无可用数据 |

### 5.3 指针与错误：ERR_PTR / IS_ERR / PTR_ERR

返回指针的内核函数用特殊约定表示错误——将错误码编码到指针值中：

```c
// 返回指针的函数
struct file *filp_open(const char *filename, int flags, umode_t mode);

// 使用
struct file *f = filp_open("/dev/null", O_RDONLY, 0);
if (IS_ERR(f)) {
    int err = PTR_ERR(f);           // 提取错误码
    pr_err("open failed: %d\n", err);
    return err;
}
// 正常使用 f ...
```

**原理**：内核虚拟地址空间的最高几页永远不会被映射，所以值在 `[-4095, -1]` 范围内的指针不可能是合法地址，可以安全地用来编码错误：

```c
// include/linux/err.h
#define MAX_ERRNO   4095

static inline void *ERR_PTR(long error) {
    return (void *)error;           // -ENOMEM → 0xFFFFFFFFFFFFFFF4
}

static inline long PTR_ERR(const void *ptr) {
    return (long)ptr;               // 0xFFFFFFFFFFFFFFF4 → -ENOMEM
}

static inline bool IS_ERR(const void *ptr) {
    return (unsigned long)ptr >= (unsigned long)-MAX_ERRNO;
}
```

> **Rust-for-Linux 用 `Result<T, Error>` 替代这套约定**，将错误处理从运行时约定变为编译期类型检查。忘记检查 `IS_ERR`？C 编译器不会告诉你。Rust 的 `Result` 必须被处理，否则编译器发出警告。

---

## 6. 其他重要数据结构

### 6.1 红黑树（rbtree）

自平衡二叉搜索树，保证 O(log n) 的查找、插入、删除。

```c
#include <linux/rbtree.h>

struct my_node {
    int key;
    struct rb_node rb;    // 嵌入红黑树节点
};
```

**用途**：CFS 调度器用红黑树按虚拟运行时间排列进程，内存管理用红黑树管理 VMA（虚拟内存区域）。

### 6.2 哈希表（hashtable）

内核提供基于链表的开放寻址哈希表：

```c
#include <linux/hashtable.h>

DEFINE_HASHTABLE(my_table, 10);    // 2^10 = 1024 个桶

struct my_entry {
    int id;
    char name[32];
    struct hlist_node hash;         // 嵌入哈希链表节点
};

// 添加
hash_add(my_table, &entry->hash, entry->id);

// 查找
hash_for_each_possible(my_table, entry, hash, target_id) {
    if (entry->id == target_id)
        return entry;
}
```

**用途**：PID 查找表、inode 缓存、网络连接跟踪。

### 6.3 XArray

XArray 是基数树（radix tree）的现代替代，提供自动扩展的稀疏数组：

```c
#include <linux/xarray.h>

DEFINE_XARRAY(my_xa);

// 存储
xa_store(&my_xa, index, ptr, GFP_KERNEL);

// 加载
void *entry = xa_load(&my_xa, index);
```

**用途**：页缓存（page cache）用 XArray 按文件偏移索引页面。

### 6.4 位图（bitmap）

用于高效管理布尔标志集合：

```c
#include <linux/bitmap.h>

DECLARE_BITMAP(my_bits, 256);     // 256 位的位图

set_bit(42, my_bits);              // 设置第 42 位
clear_bit(42, my_bits);            // 清除第 42 位
test_bit(42, my_bits);             // 测试第 42 位

// 查找第一个为 0 的位
int pos = find_first_zero_bit(my_bits, 256);
```

**用途**：CPU 掩码（cpumask）、IRQ 分配、内存页帧跟踪。

### 数据结构选择指南

| 场景 | 推荐结构 | 时间复杂度 |
|------|---------|-----------|
| 频繁遍历、前后插入 | `list_head` | 插入/删除 O(1) |
| 有序数据、范围查询 | `rbtree` | 查找/插入 O(log n) |
| 精确键查找 | `hashtable` | 平均 O(1) |
| 稀疏索引、整数键 | `XArray` | O(log n) |
| 布尔标志集合 | `bitmap` | O(1) per bit |

---

## 7. 练习

### 练习 1：手算 container_of

给定以下结构体：

```c
struct student {
    int   id;          // 4 bytes
    char  name[20];    // 20 bytes
    float gpa;         // 4 bytes
    struct list_head courses;  // 16 bytes
    int   year;        // 4 bytes
};
```

如果 `courses` 字段的地址是 `0x7fff1028`，`container_of(ptr, struct student, courses)` 的返回值是什么？（假设无填充对齐，即 packed 布局）

### 练习 2：阅读 list.h

```bash
# 在内核源码中查看 list_head 的实现
less ~/linux/include/linux/list.h
```

回答：
1. `list_add` 和 `list_add_tail` 有什么区别？
2. `list_for_each_entry` 展开后是什么样的循环？
3. 为什么有 `list_for_each_entry_safe` 这个变体？

### 练习 3：查找内核中的 list_head 使用

```bash
# 在内核源码中搜索 list_head 的使用
grep -r "struct list_head" ~/linux/include/linux/sched.h | head -20
```

找出 `task_struct` 中至少 5 个 `list_head` 字段，并说明每个的用途。

---

## 8. 知识检查

1. **`container_of` 宏在底层是如何工作的？（用指针算术解释）**

2. **Linux 为什么使用侵入式链表而不是包装式链表？**

3. **当 kref 计数减到零时，会发生什么？**

4. **内核函数返回 `-ENOMEM` 意味着什么？Rust-for-Linux 如何替代这种错误约定？**

5. **如果一个结构体需要同时出现在 3 个不同的链表中，侵入式链表如何实现？**

---

## 9. 参考资源

| 资源 | 说明 |
|------|------|
| `include/linux/list.h` | 链表完整实现 |
| `include/linux/kref.h` | kref 引用计数实现 |
| `include/linux/container_of.h` | container_of 宏定义 |
| `include/linux/rbtree.h` | 红黑树实现 |
| `include/linux/hashtable.h` | 哈希表实现 |
| `include/linux/xarray.h` | XArray 实现 |
| [Kernel API documentation](https://docs.kernel.org/core-api/kernel-api.html) | 内核 API 文档 |
| *Linux Kernel Development* (Robert Love) Ch 6 | 内核数据结构章节 |
