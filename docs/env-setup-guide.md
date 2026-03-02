# Rust-for-Linux 开发环境搭建全记录

> 日期：2026-03-01 ~ 2026-03-02
> 硬件：Apple Silicon Mac（主力机） + 2018 MacBook Pro Intel i7（开发机）

---

## 目录

1. [整体架构](#1-整体架构)
2. [在 Intel Mac 上安装 Fedora](#2-在-intel-mac-上安装-fedora)
3. [配置 SSH 远程连接](#3-配置-ssh-远程连接)
4. [SSH 安全加固](#4-ssh-安全加固)
5. [合盖不休眠](#5-合盖不休眠)
6. [配置代理（mihomo）](#6-配置代理mihomo)
7. [安装内核编译工具链](#7-安装内核编译工具链)
8. [编译带 Rust 支持的内核](#8-编译带-rust-支持的内核)
9. [QEMU 测试内核](#9-qemu-测试内核)
10. [VS Code 远程开发环境](#10-vs-code-远程开发环境)

---

## 1. 整体架构

```
┌─────────────────────┐         SSH          ┌──────────────────────┐
│  Apple Silicon Mac   │ ◄──────────────────► │  Intel Mac (Fedora)  │
│  (日常使用 + 编辑)    │    <SERVER_IP>      │  (编译 + 测试)        │
│                      │                      │                      │
│  VS Code             │  Remote-SSH          │  内核源码 ~/linux     │
│  rust-analyzer       │ ◄──────────────────► │  Rust 工具链          │
│  Claude Code         │                      │  QEMU + virtme-ng    │
└─────────────────────┘                      └──────────────────────┘
```

**为什么用两台机器？**
- Intel Mac 是 x86_64 架构，内核文档/教程假设 x86_64，QEMU 无需模拟
- Apple Silicon Mac 日常使用舒适，VS Code Remote SSH 编辑代码零延迟
- 分离编辑和编译，互不干扰

---

## 2. 在 Intel Mac 上安装 Fedora

### 2.1 为什么选 Fedora + t2linux？

2018 MacBook Pro 有 **T2 安全芯片**，标准 Linux 发行版无法直接安装：
- WiFi（Broadcom BCM4364）需要特殊驱动
- SSD 走 T2 通道，需要专用驱动
- Secure Boot 默认阻止非 macOS 启动

[t2linux](https://t2linux.org) 社区专门为 T2 Mac 做了补丁，提供开箱即用的 Fedora ISO。

### 2.2 关闭 Secure Boot

1. 完全关机（长按电源 10 秒）
2. **先按住 Command + R 不放**，再按电源键
3. **不要松开**，看到 Apple logo 继续按，直到出现 Recovery 界面
4. 顶部菜单 → 实用工具 → 启动安全性实用工具
5. 安全启动 → **无安全性**
6. 外部启动 → **允许从外部介质启动**
7. 重启

> **踩坑**：很多人看到 Apple logo 就松手，太早了。必须等到 Recovery 界面出现才松。

### 2.3 制作 USB 启动盘

在 Apple Silicon Mac 上操作：

```bash
# 下载 t2linux Fedora ISO（分片格式）
cd ~/Downloads
curl -L -O https://github.com/t2linux/fedora-iso/releases/download/f42.2/Fedora-t2linux-Workstation-Live-42.2.x86_64.iso.00
curl -L -O https://github.com/t2linux/fedora-iso/releases/download/f42.2/Fedora-t2linux-Workstation-Live-42.2.x86_64.iso.01
curl -L -O https://github.com/t2linux/fedora-iso/releases/download/f42.2/Fedora-t2linux-Workstation-Live-42.2.x86_64.iso.sha256

# 合并分片
cat Fedora-t2linux-Workstation-Live-42.2.x86_64.iso.00 \
    Fedora-t2linux-Workstation-Live-42.2.x86_64.iso.01 \
    > Fedora-t2linux-Workstation-Live-42.2.x86_64.iso

# 校验完整性（两行输出的 hash 应一致）
shasum -a 256 Fedora-t2linux-Workstation-Live-42.2.x86_64.iso
cat Fedora-t2linux-Workstation-Live-42.2.x86_64.iso.sha256
```

> **为什么要校验 hash？** 下载过程中文件可能损坏（网络问题、磁盘问题），写入损坏的 ISO 会导致安装失败。SHA256 校验能确保文件完整。

```bash
# 查看 U 盘设备号
diskutil list
# 找到 U 盘（看大小和名字判断），假设是 /dev/disk5

# 卸载 U 盘（不是弹出，是卸载）
diskutil unmountDisk /dev/disk5

# 写入 ISO 到 U 盘
sudo dd if=Fedora-t2linux-Workstation-Live-42.2.x86_64.iso of=/dev/rdisk5 bs=4m status=progress
```

**dd 参数解释：**
| 参数 | 含义 |
|------|------|
| `dd` | 底层磁盘复制工具，逐字节写入 |
| `if=` | input file，数据来源 |
| `of=` | output file，写入目标 |
| `/dev/rdisk5` | U 盘的 raw 设备（比 `/dev/disk5` 快，跳过系统缓存） |
| `bs=4m` | 每次读写 4MB 块（提高速度） |
| `status=progress` | 实时显示写入进度 |

> **危险操作**：`of=` 写错设备号会毁掉数据！务必用 `diskutil list` 确认。

### 2.4 安装 Fedora

1. U 盘插入 Intel Mac
2. 关机 → 按电源键 → **立刻按住 Option (⌥) 不放** → 等待磁盘选择界面
3. 选 **EFI Boot**
4. 进入 Fedora Live 桌面 → 点 **Install to Hard Drive**
5. 安装选项：
   - **Use entire disk**（整台机器给 Linux）
   - **不加密**（开发机，避免额外开销）
6. 等待安装完成 → 重启 → 拔 U 盘

> **踩坑**：Option 键也需要一直按住直到出现选择界面，提前松手会直接进 macOS。某些 USB 口/转接器不兼容启动，多试几个口。

### 2.5 WiFi 驱动

T2 Mac 的 WiFi 装完系统不能直接用，需要：
1. 先用有线网络（USB 网线转接器 / 手机 USB 热点）临时联网
2. 运行 t2linux 提供的固件安装工具，选择 "Retrieve from EFI partition" 或 "Download from Apple"
3. 如果缺少工具：`sudo dnf install -y dmg2img curl`
4. 如果 DNS 不工作：`echo "nameserver 8.8.8.8" | sudo tee /etc/resolv.conf`

---

## 3. 配置 SSH 远程连接

### 3.1 原理

SSH（Secure Shell）通过加密通道远程控制另一台机器。认证方式有两种：
- **密码认证**：输入密码登录（不安全，可被暴力破解）
- **公钥认证**：本地生成密钥对，公钥放服务器，私钥留本地（安全）

```
私钥 (~/.ssh/id_rsa)          公钥 (~/.ssh/authorized_keys)
只在你的 Mac 上                放在 Fedora 服务器上
永远不泄露                      可以公开

连接时：
1. 客户端发送公钥指纹
2. 服务器在 authorized_keys 中查找匹配
3. 服务器发送随机挑战（challenge）
4. 客户端用私钥签名挑战
5. 服务器用公钥验证签名 → 通过 → 登录成功
```

### 3.2 Fedora 端：启用 SSH 服务

```bash
# 启用并启动 SSH 守护进程
sudo systemctl enable --now sshd

# 确认在运行
sudo systemctl status sshd

# 查看 IP 地址
ip addr show | grep 192
# 记下 192.168.x.x 的地址
```

### 3.3 Mac 端：配置 SSH config

编辑 `~/.ssh/config`，添加：

```
Host rfl-dev
    HostName <SERVER_IP>
    User youruser
    Port 22
    IdentityFile ~/.ssh/id_rsa
```

**config 文件的作用**：避免每次输入完整命令。配置后 `ssh rfl-dev` 等价于 `ssh -i ~/.ssh/id_rsa youruser@<SERVER_IP> -p 22`。

### 3.4 推送公钥

```bash
# 方法 1：ssh-copy-id（需要密码认证已开启）
ssh-copy-id -o PubkeyAuthentication=no -i ~/.ssh/id_rsa.pub youruser@<SERVER_IP>

# 方法 2：手动（如果 ssh-copy-id 不工作）
# 在 Fedora 上执行：
mkdir -p ~/.ssh && chmod 700 ~/.ssh
# 把公钥内容粘贴到 ~/.ssh/authorized_keys
chmod 600 ~/.ssh/authorized_keys
```

> **踩坑**：Fedora 默认开启 SELinux，可能阻止 SSH 读取 authorized_keys。修复：`restorecon -Rv ~/.ssh`

### 3.5 配置免密 sudo

开发机上频繁需要 sudo，配置免密：

```bash
# 在 Fedora 上执行
echo "youruser ALL=(ALL) NOPASSWD: ALL" | sudo tee /etc/sudoers.d/youruser
```

---

## 4. SSH 安全加固

开放 SSH 到网络就要做安全加固，防止未授权访问。

```bash
# 创建加固配置（数字越小优先级越高）
sudo tee /etc/ssh/sshd_config.d/01-hardening.conf << 'EOF'
PasswordAuthentication no      # 禁用密码登录，只允许公钥
PermitRootLogin no             # 禁止 root 远程登录
PermitEmptyPasswords no        # 禁用空密码
MaxAuthTries 3                 # 最多尝试 3 次
X11Forwarding no               # 禁用 X11 转发（不需要图形转发）
GSSAPIAuthentication no        # 禁用 GSSAPI（加快连接速度）
EOF

# 验证配置无语法错误
sudo sshd -t

# 重启 SSH
sudo systemctl restart sshd

# 验证生效
sudo sshd -T | grep -iE 'passwordauth|permitroot'
# 应输出：
# passwordauthentication no
# permitrootlogin no
```

> **为什么文件名是 01-hardening.conf？** `sshd_config.d/` 目录下的文件按文件名排序加载，后加载的覆盖先加载的。Fedora 自带 `50-redhat.conf`，我们用 `01-` 确保最先加载，不被覆盖。实际测试发现 `99-` 会被 `50-redhat.conf` 覆盖。

**加固效果**：
- 密码登录禁用 → 暴力破解无效
- 只有持有私钥的机器能登录
- root 不能远程登录 → 即使拿到密钥也只能操作普通用户

---

## 5. 合盖不休眠

笔记本当服务器用，合盖不能休眠：

```bash
# 禁用所有休眠相关目标
sudo systemctl mask sleep.target suspend.target hibernate.target hybrid-sleep.target

# 配置 logind 忽略合盖事件
sudo mkdir -p /etc/systemd/logind.conf.d
sudo tee /etc/systemd/logind.conf.d/lid.conf << 'EOF'
[Login]
HandleLidSwitch=ignore
HandleLidSwitchDocked=ignore
HandleLidSwitchExternalPower=ignore
EOF

# 重启 logind
sudo systemctl restart systemd-logind
```

**原理**：
- `systemctl mask` 创建到 `/dev/null` 的符号链接，使目标永远无法激活
- `HandleLidSwitch=ignore` 告诉 logind 合盖时什么都不做
- 三个 LidSwitch 分别对应：正常合盖、连接扩展坞合盖、接电源合盖

---

## 6. 配置代理（mihomo）

从中国访问 GitHub 需要代理。mihomo 是 Clash 的社区维护版本。

### 6.1 安装

```bash
# 在 Mac 上下载（Mac 有代理能访问 GitHub）
# 从 https://github.com/MetaCubeX/mihomo/releases 下载对应平台的包

# 通过 SCP 传到 Fedora
scp mihomo-linux-amd64.rpm rfl-dev:~/

# 在 Fedora 上安装
sudo rpm -i ~/mihomo-linux-amd64.rpm
```

### 6.2 配置

```bash
# 创建配置目录
mkdir -p ~/.config/mihomo

# 下载订阅配置（在 Mac 上下载再 SCP 过去）
# scp config.yaml rfl-dev:~/.config/mihomo/config.yaml
```

mihomo 启动需要 GeoIP 数据库，首次启动会尝试从 GitHub 下载（但没代理下不了）。解决方法：在 Mac 上下载再传过去。

```bash
# Mac 上下载
curl -sL -o geoip.metadb https://github.com/MetaCubeX/meta-rules-dat/releases/download/latest/geoip.metadb
curl -sL -o geosite.dat https://github.com/MetaCubeX/meta-rules-dat/releases/download/latest/geosite.dat

# 传到 Fedora
scp geoip.metadb geosite.dat rfl-dev:~/.config/mihomo/
```

### 6.3 设为系统服务

```bash
sudo tee /etc/systemd/system/mihomo.service << 'EOF'
[Unit]
Description=mihomo Daemon
After=network.target

[Service]
Type=simple
User=youruser
ExecStart=/usr/bin/mihomo -d /home/youruser/.config/mihomo
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable --now mihomo
```

**systemd 服务文件解释**：
- `After=network.target`：网络就绪后才启动
- `Type=simple`：直接运行，不 fork
- `Restart=on-failure`：崩溃自动重启
- `WantedBy=multi-user.target`：开机自启

### 6.4 配置环境变量

```bash
# 在 ~/.bashrc 中添加
export http_proxy=http://127.0.0.1:7893
export https_proxy=http://127.0.0.1:7893
export no_proxy=localhost,127.0.0.1,192.168.0.0/16
```

`no_proxy` 确保本地和局域网流量不走代理。

---

## 7. 安装内核编译工具链

### 7.1 基础编译依赖

```bash
sudo dnf install -y bc kmod cpio flex ncurses-devel openssl-devel \
  dwarves elfutils-libelf-devel bison git rsync clang lld llvm \
  perl-FindBin perl-File-Compare zstd make gcc curl
```

| 包 | 用途 |
|------|------|
| `gcc`, `make` | C 编译器和构建工具 |
| `clang`, `lld`, `llvm` | LLVM 工具链（Rust 内核构建必需，`make LLVM=1`） |
| `flex`, `bison` | 词法/语法分析器（内核构建脚本需要） |
| `ncurses-devel` | `make menuconfig` 的 TUI 界面 |
| `openssl-devel` | 内核签名 |
| `elfutils-libelf-devel` | ELF 格式处理 |
| `dwarves` | BTF（BPF Type Format）生成 |
| `bc` | 内核版本计算 |
| `zstd` | 内核压缩 |

### 7.2 克隆 RFL 内核源码

```bash
git clone --depth=1 https://github.com/Rust-for-Linux/linux.git ~/linux
```

`--depth=1` 只拉最新一个 commit，节省时间和空间（完整历史 >5GB）。

### 7.3 Rust 工具链（按 RFL 社区标准）

```bash
# 安装 rustup（Rust 版本管理器）
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh -s -- -y
source ~/.cargo/env

# 查看内核要求的最低版本
cat ~/linux/scripts/min-tool-version.sh | grep -E 'rustc|bindgen'
# 输出：rustc 1.78.0, bindgen 0.65.1

# 安装必要组件
rustup component add rust-src    # Rust 标准库源码（内核编译需要）
rustup component add rustfmt     # 代码格式化
rustup component add clippy      # 代码检查

# 安装 bindgen（C → Rust FFI 绑定生成器）
cargo install --locked bindgen-cli

# 验证内核是否认可这套工具链
cd ~/linux && make LLVM=1 rustavailable
# 输出 "Rust is available!" 即为成功
```

**关键概念**：
- **bindgen**：读取 C 头文件，自动生成 Rust FFI 绑定。内核用它让 Rust 代码调用 C 函数。
- **rust-src**：Rust 标准库源码。内核不用 std，但需要 core/alloc 库的源码来交叉编译。
- **LLVM=1**：告诉内核构建系统使用 Clang/LLVM 而非 GCC。Rust 内核支持需要 LLVM。

---

## 8. 编译带 Rust 支持的内核

### 8.1 生成配置

```bash
cd ~/linux

# 生成默认配置
make LLVM=1 defconfig

# 启用 Rust 支持
scripts/config --enable CONFIG_RUST

# 启用 Rust 示例模块（用于验证）
scripts/config --enable CONFIG_SAMPLES
scripts/config --enable CONFIG_SAMPLES_RUST
scripts/config --enable CONFIG_SAMPLE_RUST_MINIMAL
scripts/config --enable CONFIG_SAMPLE_RUST_PRINT

# 启用 virtio 文件系统（QEMU 测试需要）
scripts/config --enable CONFIG_NET_9P
scripts/config --enable CONFIG_NET_9P_VIRTIO
scripts/config --enable CONFIG_9P_FS
scripts/config --enable CONFIG_VIRTIO
scripts/config --enable CONFIG_VIRTIO_PCI
scripts/config --enable CONFIG_VIRTIO_FS
scripts/config --enable CONFIG_FUSE_FS
scripts/config --enable CONFIG_OVERLAY_FS

# 解决依赖关系
make LLVM=1 olddefconfig

# 确认 Rust 已启用
grep CONFIG_RUST=y .config
```

**内核配置系统**：
- `defconfig`：生成当前架构的默认配置
- `scripts/config --enable`：命令行修改 `.config` 文件
- `olddefconfig`：根据依赖关系自动补全新增的配置项
- 每个 `CONFIG_` 选项可以是 `y`（内置）、`m`（模块）、`n`（不编译）

### 8.2 编译

```bash
make LLVM=1 -j$(nproc)
# -j$(nproc) 使用所有 CPU 核心并行编译
# 12 线程大约 10-15 分钟

# 成功输出：
# Kernel: arch/x86/boot/bzImage is ready
```

`bzImage` 是压缩的内核映像，"bz" 代表 "big zImage"。

---

## 9. QEMU 测试内核

### 9.1 安装 QEMU + virtme-ng

```bash
sudo dnf install -y qemu-system-x86 busybox python3-pip
pip3 install virtme-ng
```

**工具说明**：
- **QEMU**：硬件模拟器/虚拟机，可以启动任意内核映像
- **virtme-ng**：封装 QEMU，自动处理 initramfs、rootfs 挂载，让测试内核变得一键化
- **busybox**：精简版 Unix 工具集，用于构建 initramfs（初始内存文件系统）

### 9.2 启动测试

```bash
cd ~/linux

# 用 virtme-ng 启动编译好的内核
vng -r arch/x86/boot/bzImage --force-initramfs --exec 'uname -a && dmesg | grep -i rust'
```

**成功输出**：
```
Linux version 7.0.0-rc1 ...
rust_minimal: Rust minimal sample (init)
rust_minimal: Am I built-in or a module? Built-in!
rust_minimal: My numbers are [72, 108, 200]
rust_print: Rust printing macros sample (init)
rust_print: "hello, world"
```

**这意味着什么**：
- Linux 7.0-rc1 内核启动成功
- Rust 运行时在内核中正常工作
- `rust_minimal` 和 `rust_print` 两个示例模块执行了 init 函数
- Rust 代码在内核空间中运行，可以使用 `pr_info!` 宏打印到 dmesg

---

## 10. VS Code 远程开发环境

### 10.1 安装 VS Code

```bash
# 下载
curl -sL -o ~/Downloads/VSCode-darwin-arm64.zip \
  "https://update.code.visualstudio.com/latest/darwin-arm64/stable"

# 解压到 Applications
unzip -qo ~/Downloads/VSCode-darwin-arm64.zip -d /tmp/vscode-install
mv "/tmp/vscode-install/Visual Studio Code.app" /Applications/
```

### 10.2 安装扩展

```bash
# 添加 code 到 PATH
export PATH="/Applications/Visual Studio Code.app/Contents/Resources/app/bin:$PATH"

# 安装 Remote-SSH（远程开发）
code --install-extension ms-vscode-remote.remote-ssh

# 安装 rust-analyzer（Rust 语言支持）
code --install-extension rust-lang.rust-analyzer
```

### 10.3 生成 rust-analyzer 配置

在 Fedora 上：

```bash
cd ~/linux
make LLVM=1 rust-analyzer
# 生成 rust-project.json（4.3MB，描述内核的 Rust crate 结构）
```

**为什么需要 rust-project.json？** 内核不使用 Cargo，所以 rust-analyzer 无法通过 `Cargo.toml` 理解项目结构。`make rust-analyzer` 生成的 `rust-project.json` 告诉 rust-analyzer 哪些 crate 存在、它们的依赖关系、编译选项等。

### 10.4 VS Code 工作区配置

`.vscode/settings.json`：

```json
{
    "rust-analyzer.linkedProjects": [
        "rust-project.json"
    ],
    "rust-analyzer.check.overrideCommand": [],
    "rust-analyzer.cargo.sysroot": null,
    "rust-analyzer.procMacro.enable": false,
    "editor.formatOnSave": true,
    "files.watcherExclude": {
        "**/.git/objects/**": true,
        "**/drivers/**": true,
        "**/arch/!(x86)/**": true,
        "**/tools/**": true,
        "**/Documentation/**": true
    }
}
```

- `linkedProjects`：指向 `rust-project.json` 而非 Cargo.toml
- `procMacro.enable: false`：内核的过程宏在 rust-analyzer 中不完全支持
- `files.watcherExclude`：排除大量无关目录，避免 VS Code 卡顿

### 10.5 连接使用

```bash
# 一键打开远程内核目录
code --remote ssh-remote+rfl-dev /home/youruser/linux
```

或在 VS Code 中：`Cmd+Shift+P` → Remote-SSH: Connect to Host → rfl-dev → 打开 `/home/youruser/linux`

### 10.6 个性化配置

全局设置 `~/Library/Application Support/Code/User/settings.json`：

```json
{
    "workbench.colorTheme": "One Dark Pro",
    "editor.fontFamily": "JetBrains Mono, Menlo, Monaco, monospace",
    "editor.fontSize": 16,
    "editor.lineHeight": 1.6,
    "editor.fontLigatures": false,
    "editor.cursorBlinking": "smooth",
    "editor.cursorSmoothCaretAnimation": "on",
    "editor.smoothScrolling": true,
    "editor.minimap.enabled": false,
    "editor.bracketPairColorization.enabled": true,
    "editor.renderWhitespace": "boundary",
    "terminal.integrated.fontSize": 15,
    "terminal.integrated.fontFamily": "JetBrains Mono",
    "window.zoomLevel": 1
}
```

---

## 附录：环境速查

### SSH 连接

```bash
ssh rfl-dev                    # 终端连接
code --remote ssh-remote+rfl-dev /home/youruser/linux  # VS Code 连接
```

### 常用内核开发命令

```bash
cd ~/linux

make LLVM=1 menuconfig        # 图形化配置
make LLVM=1 defconfig          # 默认配置
make LLVM=1 olddefconfig       # 更新配置依赖
make LLVM=1 rustavailable      # 检查 Rust 工具链
make LLVM=1 rust-analyzer      # 生成 rust-analyzer 配置
make LLVM=1 -j$(nproc)         # 编译
make LLVM=1 CLIPPY=1 -j$(nproc)  # 编译 + Clippy 检查
make LLVM=1 rustfmt            # 格式化检查

vng -r arch/x86/boot/bzImage --force-initramfs --exec 'command'  # QEMU 测试
```

### 目录结构

```
~/linux/
├── rust/kernel/       # Rust 内核抽象层（核心代码）
├── samples/rust/      # Rust 示例模块
├── drivers/           # 驱动（部分 Rust）
├── arch/x86/          # x86 架构相关
├── scripts/           # 构建脚本
├── .config            # 当前内核配置
├── rust-project.json  # rust-analyzer 配置
└── arch/x86/boot/bzImage  # 编译输出的内核映像
```

### 里程碑

- [x] 开发环境就绪（2026-03-01）
- [x] 首次编译带 Rust 的内核（2026-03-01）
- [x] 首次运行 Rust 内核模块（2026-03-01）
- [ ] 首个补丁发送到邮件列表
- [ ] 首次代码审查
