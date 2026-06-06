---
title: "通过 SSH 从 Mac 远程控制 Windows"
date: 2026-06-06
categories: ["技术"]
tags: ["SSH", "Mac", "Windows", "远程控制"]
summary: "在同一局域网下，通过 OpenSSH 从 Mac 终端远程登录 Windows 台式机，执行命令行操作。"
draft: false
---

## 背景

手里有一台 MacBook Air 和一台 Windows 台式机，连接着同一个 Wi-Fi。很多时候我想在 Mac 上直接操作 Windows——比如跑个脚本、传个文件——但又不想走到台式机前去操作。SSH 是最轻量的解决方案。

## 什么是 SSH

SSH（Secure Shell，安全外壳协议）让你**安全地远程登录另一台电脑**，并在上面执行命令。所有传输的数据都是加密的，不会被窃听或篡改。

一个类比：SSH 就像一条加密的隧道，你在这头输入命令，另一头的电脑收到并执行，然后把结果安全地传回来。

### SSH vs 远程桌面

| 对比 | SSH | 远程桌面 (RDP) |
|------|-----|----------------|
| 传输内容 | 命令行（纯文本） | 图形界面（屏幕画面） |
| 带宽要求 | 极低 | 较高 |
| 操作方式 | 键盘敲命令 | 鼠标点击图形界面 |
| 适用场景 | 服务器、开发、自动化 | 日常办公、图形软件 |

SSH 更轻量高效，适合命令行操作；远程桌面适合需要图形界面的场景。

## Windows 端：启用 OpenSSH 服务器

Windows 10/11 自带 OpenSSH，只需要几步开启。

### 1. 安装并启动 SSH 服务

以管理员身份打开 PowerShell，执行：

```powershell
# 安装 OpenSSH 服务器
Add-WindowsCapability -Online -Name OpenSSH.Server~~~~0.0.1.0

# 启动 SSH 服务
Start-Service sshd

# 设置开机自启
Set-Service -Name sshd -StartupType Automatic
```

### 2. 确认防火墙放行端口 22

安装时通常会自动配置防火墙规则，可以验证一下：

```powershell
# 查看已有的 SSH 防火墙规则
Get-NetFirewallRule -Name *ssh*

# 如果没有规则，手动添加
New-NetFirewallRule -Name sshd -DisplayName 'OpenSSH Server' -Enabled True -Direction Inbound -Protocol TCP -Action Allow -LocalPort 22
```

### 3. 查看 Windows 的 IP 地址

```powershell
ipconfig
```

找到无线局域网适配器（或以太网适配器）下的 IPv4 地址，类似 `192.168.x.x`，后面连接要用。

### 4. 确认用户名

```powershell
whoami
# 例如输出 DESKTOP-ABC\pkz → 用户名为 pkz
```

## Mac 端：连接

打开终端，先测试网络是否通：

```bash
nc -zv 192.168.x.x 22
```

然后连接：

```bash
ssh 用户名@192.168.x.x
# 例如：ssh pkz@192.168.31.143
```

首次连接会提示确认主机指纹，输入 `yes` 回车，然后输入 Windows 账户的密码即可。连接成功后，终端就变成了 Windows 的命令行。

## 进阶：密钥登录（免密码）

每次连接都要输入密码很麻烦，而且密码可能在输入时被看到。SSH 密钥登录通过一对密钥（公钥 + 私钥）来认证，配置好之后连接不再需要密码。

### 原理

简单来说：你在 Mac 上生成一对密钥，把公钥放到 Windows 上，私钥留在 Mac 上。连接时，SSH 用私钥做签名，Windows 用公钥验证——匹配则通过，整个过程不需要传输密码。

类比：公钥像是一把锁（装在 Windows 上），私钥是唯一的钥匙（留在 Mac 上）。只有拿着钥匙的人才能开锁。

### 1. 在 Mac 上生成密钥对

```bash
ssh-keygen -t ed25519 -C "mac-to-windows"
```

一路回车即可（使用默认路径 `~/.ssh/id_ed25519`，不设密码短语）。

生成两个文件：
- `~/.ssh/id_ed25519` — 私钥（绝不能给别人）
- `~/.ssh/id_ed25519.pub` — 公钥（可以放到任何服务器上）

### 2. 把公钥复制到 Windows

Windows 的 OpenSSH 默认从 `C:\Users\用户名\.ssh\authorized_keys` 读取公钥。在 Mac 上执行：

```bash
# 在 Windows 上创建 .ssh 目录（如果不存在）
ssh 用户名@192.168.x.x "mkdir C:\Users\用户名\.ssh 2>nul"

# 把公钥追加到 authorized_keys
cat ~/.ssh/id_ed25519.pub | ssh 用户名@192.168.x.x "cat >> C:\Users\用户名\.ssh\authorized_keys"
```

或者手动方式：把 `id_ed25519.pub` 的内容复制出来，在 Windows 上用记事本粘贴到 `C:\Users\用户名\.ssh\authorized_keys`。

### 3. 测试免密登录

```bash
ssh 用户名@192.168.x.x
```

不再提示输入密码，直接进入——配置成功。

如果仍然要求密码，检查 Windows 端：
- `authorized_keys` 文件内容是否正确（一整行，不能有换行断裂）
- 路径中的用户名是否和你登录的用户名一致

## 进阶：用 SCP 传文件

SSH 连通之后，可以用 `scp`（Secure Copy）在两台电脑之间安全地传输文件，原理和 SSH 相同——数据全程加密。

### 从 Mac 传文件到 Windows

```bash
scp 本地文件路径 用户名@192.168.x.x:目标路径

# 例：把 Mac 桌面上的报告传到 Windows 桌面
scp ~/Desktop/report.pdf pkz@192.168.31.143:C:/Users/pkz/Desktop/
```

### 从 Windows 传文件到 Mac

```bash
scp 用户名@192.168.x.x:远程文件路径 本地路径

# 例：把 Windows 上的实验结果拉到 Mac
scp pkz@192.168.31.143:C:/Users/pkz/Desktop/results.csv ~/Downloads/
```

### 传整个文件夹

加 `-r`（recursive）参数即可递归传输整个目录：

```bash
# 把整个项目文件夹从 Mac 推到 Windows
scp -r ~/projects/my_project pkz@192.168.31.143:C:/Users/pkz/Desktop/

# 把 Windows 上的文件夹拉到 Mac
scp -r pkz@192.168.31.143:C:/Users/pkz/Desktop/data ~/Downloads/
```

### SCP vs 其他传文件方式

| 方式 | 优点 | 缺点 |
|------|------|------|
| SCP | 加密、命令行一行搞定 | 传大量小文件较慢 |
| 微信/QQ传文件 | 有图形界面 | 大小限制、文件名会变、慢 |
| U盘 | 不依赖网络 | 要走过去插 |
| 共享文件夹（SMB） | 可在 Finder 中直接访问 | 配置稍复杂 |

日常传几个文件用 SCP 就够了。如果频繁双向同步大量文件，可以考虑配置 SMB 共享文件夹。

## 进阶：SSH 配置文件简化连接

每次都要输入 `ssh pkz@192.168.31.143` 太长了。可以在 `~/.ssh/config` 中写好别名：

### 配置方法

在 Mac 上创建或编辑 `~/.ssh/config`：

```bash
nano ~/.ssh/config
```

添加以下内容：

```
Host win
    HostName 192.168.31.143
    User pkz
    IdentityFile ~/.ssh/id_ed25519
```

保存后，连接只需：

```bash
ssh win
```

SCP 也同样简化：

```bash
scp report.pdf win:C:/Users/pkz/Desktop/
scp win:C:/Users/pkz/Desktop/results.csv ~/Downloads/
```

### 如果 IP 会变

家庭路由器通常用 DHCP，重启后 IP 可能变化。两个解决办法：

1. **在路由器管理页面给 Windows 绑定固定 IP**（推荐，一劳永逸）
2. **每次连接前先在 Windows 上 `ipconfig` 确认 IP**，然后更新 config

## 常见问题

**连接超时？** 检查两台电脑是否在同一局域网、Windows 防火墙是否放行了 22 端口、sshd 服务是否在运行。

**密码正确但被拒绝？** 确认使用的是 Windows 本地账户的用户名和密码，而不是 Microsoft 账户。如果用的是 Microsoft 账户登录 Windows，SSH 的密码也是那个 Microsoft 账户密码。

**密钥登录不生效，还是要求密码？** 检查 `authorized_keys` 的内容是否完整（一整行），以及 Windows 端 `sshd_config` 中 `PubkeyAuthentication` 是否为 `yes`。

**Windows 重启后连不上？** 确认 `sshd` 的启动类型是 `Automatic`（前面配置过），重启后服务会自动启动。
