# 编写一个简单的 Rust 内核模块

> 模块：rfl-drivers-01 | 难度：进阶 | 预计时间：4 小时
> 前置要求：[kernel Crate：核心抽象](rfl-arch-02.md)、[构建系统](kernel-basics-02.md)

---

## 目录

1. [最小内核模块：rust_minimal](#1-最小内核模块rust_minimal)
2. [module! 宏展开](#2-module-宏展开)
3. [Module trait 与 Drop](#3-module-trait-与-drop)
4. [模块参数](#4-模块参数)
5. [MiscDevice：字符设备驱动](#5-miscdevice字符设备驱动)
6. [InPlaceModule：就地初始化模块](#6-inplacemodule就地初始化模块)
7. [将模块加入 Kbuild](#7-将模块加入-kbuild)
8. [实战：从零到 QEMU 测试](#8-实战从零到-qemu-测试)
9. [练习](#9-练习)
10. [知识检查](#10-知识检查)
11. [参考资源](#11-参考资源)

---

## 1. 最小内核模块：rust_minimal

内核源码中自带的最简示例：

```rust
// samples/rust/rust_minimal.rs
use kernel::prelude::*;

module! {
    type: RustMinimal,
    name: "rust_minimal",
    author: "Rust for Linux Contributors",
    description: "Rust minimal sample",
    license: "GPL",
}

struct RustMinimal;

impl kernel::Module for RustMinimal {
    fn init(_module: &'static ThisModule) -> Result<Self> {
        pr_info!("Rust minimal sample (init)\n");
        Ok(RustMinimal)
    }
}

impl Drop for RustMinimal {
    fn drop(&mut self) {
        pr_info!("Rust minimal sample (exit)\n");
    }
}
```

这段代码做了什么：

| 部分 | 作用 | C 等价物 |
|------|------|---------|
| `module! { ... }` | 声明模块元数据 | `MODULE_LICENSE`、`MODULE_AUTHOR` 等 |
| `impl Module` | 模块加载时执行 | `module_init()` |
| `impl Drop` | 模块卸载时执行 | `module_exit()` |
| `pr_info!` | 内核日志输出 | `printk(KERN_INFO ...)` |

---

## 2. module! 宏展开

`module!` 宏生成 C 兼容的模块元数据，让 `insmod`/`modprobe` 能识别：

```rust
module! {
    type: RustMinimal,        // 实现 Module trait 的类型
    name: "rust_minimal",     // 模块名（insmod 显示的名字）
    author: "...",            // 作者信息
    description: "...",       // 模块描述
    license: "GPL",           // 许可证（必须是 GPL 兼容的）
}
```

展开后大致生成：

```rust
// 模块元数据（链接到 .modinfo 段）
#[link_section = ".modinfo"]
static __MODULE_LICENSE: ... = "license=GPL\0";
#[link_section = ".modinfo"]
static __MODULE_AUTHOR: ... = "author=...\0";

// 模块入口/出口函数（C ABI 兼容）
#[no_mangle]
pub extern "C" fn init_module() -> core::ffi::c_int { ... }
#[no_mangle]
pub extern "C" fn cleanup_module() { ... }
```

> 注意：`license: "GPL"` 不是可选的。非 GPL 模块不能使用 `EXPORT_SYMBOL_GPL` 导出的内核符号，而大部分 Rust 绑定依赖这些符号。

---

## 3. Module trait 与 Drop

### 3.1 Module trait

```rust
pub trait Module: Sized + Sync {
    fn init(module: &'static ThisModule) -> Result<Self>;
}
```

- 返回 `Result<Self>` → 初始化可以失败，返回 `Err(e)` 内核就不加载模块
- `&'static ThisModule` → 内核模块的生命周期是 `'static`
- `Sized + Sync` → 模块实例必须是固定大小且线程安全的

### 3.2 Drop 替代 module_exit

C 内核模块需要手写 `module_exit()` 清理函数。Rust 利用 `Drop` trait：

```rust
impl Drop for RustMinimal {
    fn drop(&mut self) {
        // 模块卸载时自动调用
        // 释放 init 中申请的所有资源
    }
}
```

优势：如果你的模块持有的资源（如注册的设备）本身实现了 `Drop`，卸载时会**自动级联释放**，不需要手动逐一清理。

### 3.3 对比 C 的 goto 清理

```c
// C：init 失败需要手动 goto 清理
static int __init my_init(void) {
    ret = alloc_a();
    if (ret) return ret;
    ret = alloc_b();
    if (ret) goto err_free_a;
    ret = register_c();
    if (ret) goto err_free_b;
    return 0;
err_free_b: free_b();
err_free_a: free_a();
    return ret;
}
```

```rust
// Rust：? 自动传播错误，Drop 自动清理
impl Module for MyModule {
    fn init(_module: &'static ThisModule) -> Result<Self> {
        let a = alloc_a()?;     // 失败直接返回
        let b = alloc_b()?;     // 失败自动 drop a
        let c = register_c()?;  // 失败自动 drop b 和 a
        Ok(MyModule { a, b, c })
    }
}
```

---

## 4. 模块参数

Rust 内核模块可以接受参数，类似 C 的 `module_param()`：

```rust
module! {
    type: RustMyModule,
    name: "rust_mymod",
    author: "Me",
    description: "Module with params",
    license: "GPL",
    params: {
        my_param: u32 {
            default: 42,
            permissions: 0o644,
            description: "A sample parameter",
        },
    },
}
```

使用参数：

```rust
impl kernel::Module for RustMyModule {
    fn init(module: &'static ThisModule) -> Result<Self> {
        let val = {
            let lock = module.my_param.read();
            *lock
        };
        pr_info!("my_param = {}\n", val);
        Ok(RustMyModule)
    }
}
```

加载时传参：

```bash
insmod rust_mymod.ko my_param=100
```

---

## 5. MiscDevice：字符设备驱动

从简单的日志模块进阶到能和用户空间通信的设备驱动：

```rust
use kernel::prelude::*;
use kernel::miscdevice::MiscDevice;

#[pin_data]
struct RustEcho {
    #[pin]
    dev: MiscDeviceRegistration<RustEchoDevice>,
}

struct RustEchoDevice;

#[vtable]
impl MiscDevice for RustEchoDevice {
    type Ptr = KBox<Self>;

    fn open() -> Result<KBox<Self>> {
        Ok(KBox::new(RustEchoDevice, GFP_KERNEL)?)
    }

    fn read(_this: &Self, _file: &File, buf: &mut UserSliceWriter, _offset: u64) -> Result<usize> {
        let msg = b"Hello from kernel!\n";
        buf.write(msg)?;
        Ok(msg.len())
    }

    fn write(_this: &Self, _file: &File, buf: &mut UserSliceReader, _offset: u64) -> Result<usize> {
        let len = buf.len();
        let data = buf.read_all(GFP_KERNEL)?;
        pr_info!("Received {} bytes from userspace\n", len);
        Ok(len)
    }
}
```

关键概念：

| 概念 | 说明 |
|------|------|
| `MiscDevice` trait | 定义字符设备的操作（open/read/write/ioctl） |
| `#[vtable]` | 生成 C 兼容的函数指针表（`struct file_operations`） |
| `UserSliceReader/Writer` | 安全的用户空间内存访问（封装 `copy_from_user`/`copy_to_user`） |
| `MiscDeviceRegistration` | RAII 注册——构造时注册设备，Drop 时自动注销 |

### 5.1 用户空间交互

设备注册后会出现在 `/dev/` 下，用户空间程序可以正常读写：

```bash
# 读取
cat /dev/rust-echo

# 写入
echo "hello" > /dev/rust-echo

# 或者用自定义工具
./my_tool /dev/rust-echo
```

---

## 6. InPlaceModule：就地初始化模块

当模块包含需要 pin 的字段时（如 `MiscDeviceRegistration`），不能用普通的 `Module` trait，要用 `InPlaceModule`：

```rust
impl kernel::InPlaceModule for RustEcho {
    fn init(_module: &'static ThisModule) -> impl PinInit<Self, Error> {
        try_pin_init!(Self {
            dev <- MiscDeviceRegistration::register(fmt!("rust-echo")),
        })
    }
}
```

对比两种 trait：

| | Module | InPlaceModule |
|---|---|---|
| 返回值 | `Result<Self>` | `impl PinInit<Self, Error>` |
| 适用场景 | 简单模块（无 pin 字段） | 含注册型资源的模块 |
| 构造方式 | 栈上构造，返回值 | 就地初始化，返回初始化器 |
| Drop | `impl Drop` | `impl PinnedDrop`（通过宏） |

当使用 `InPlaceModule` 时，清理用 `PinnedDrop` 而不是普通 `Drop`：

```rust
#[pinned_drop]
impl PinnedDrop for RustEcho {
    fn drop(self: Pin<&mut Self>) {
        pr_info!("Echo module unloaded\n");
        // MiscDeviceRegistration 的 Drop 会自动注销设备
    }
}
```

---

## 7. 将模块加入 Kbuild

### 7.1 Kconfig

在 `samples/rust/Kconfig` 或你的子系统 Kconfig 中添加：

```kconfig
config SAMPLE_RUST_MYMOD
    tristate "My Rust module"
    depends on RUST
    help
      A sample Rust kernel module.
```

### 7.2 Makefile

```makefile
# samples/rust/Makefile
obj-$(CONFIG_SAMPLE_RUST_MYMOD) += rust_mymod.o
```

### 7.3 编译和加载

```bash
# 配置（启用你的模块）
make menuconfig   # 找到 Kernel hacking → Sample kernel code → Rust samples

# 编译
make -j$(nproc) M=samples/rust

# 在 QEMU 中加载
insmod samples/rust/rust_mymod.ko
dmesg | tail

# 卸载
rmmod rust_mymod
dmesg | tail
```

---

## 8. 实战：从零到 QEMU 测试

完整流程回顾：

```
1. 编写 .rs 文件
   └→ samples/rust/rust_mymod.rs

2. 添加构建配置
   ├→ samples/rust/Kconfig（添加 config 条目）
   └→ samples/rust/Makefile（添加 obj- 行）

3. 编译
   └→ make -j$(nproc)

4. 在 QEMU 中测试
   ├→ virtme-ng 或手动 QEMU 启动
   ├→ insmod rust_mymod.ko [参数]
   ├→ dmesg 查看输出
   ├→ 如果是设备驱动：ls /dev/ 确认设备、cat/echo 测试
   └→ rmmod rust_mymod

5. 迭代修改
   └→ 改代码 → make → 重新 insmod
```

> 踩坑提示：如果 `insmod` 报 `Invalid module format`，检查模块是否和当前运行的内核版本一致。使用 `virtme-ng` 可以确保模块和内核始终匹配。

---

## 9. 练习

1. **编译并加载 rust_minimal**：在 QEMU 中 `insmod`，观察 `dmesg` 输出，然后 `rmmod`，再看 `dmesg`
2. **添加模块参数**：修改 rust_minimal，添加一个字符串参数，加载时传入并打印
3. **从零创建模块**：不参考示例，自己写一个 Rust 内核模块，包含 init 日志和 exit 日志，加入 Kbuild 编译

---

## 10. 知识检查

1. `module!` 宏展开后生成了什么？
2. Rust 模块的清理（卸载）和 C 的 `module_exit` 有什么不同？
3. 如何将一个新的 Rust 模块添加到内核构建系统？

---

## 11. 参考资源

- [samples/rust/rust_minimal.rs](https://github.com/Rust-for-Linux/linux/blob/rust-next/samples/rust/rust_minimal.rs) — 最小示例
- [samples/rust/rust_print.rs](https://github.com/Rust-for-Linux/linux/blob/rust-next/samples/rust/rust_print.rs) — 打印示例
- [Rust kernel module guide](https://docs.kernel.org/rust/quick-start.html) — 官方快速上手
- [rust/kernel/miscdevice.rs](https://github.com/Rust-for-Linux/linux/blob/rust-next/rust/kernel/miscdevice.rs) — MiscDevice 抽象
