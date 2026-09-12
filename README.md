# DLUT-EDA 自动登录 - Rust 版本

大连理工大学（DLUT）**开发区校区**校园网自动登录工具的 Rust 实现。

## 功能特性

- 完全模拟官网des.js加密
- 自动检测网络状态
- 适用于嵌入式系统（如 OpenWrt）的轻量级 CLI
- 同步 HTTP 请求（最小化二进制体积）
- 自动重试机制，适合无人值守运行

## 构建

```bash
cargo build --release
# 可选：去除符号表以减小二进制文件大小
strip target/release/dlut_auto_login_rs
```

## 使用方法

### 基本用法（单次登录）：

```bash
./dlut_auto_login_rs -u <username> -p <password>
```

### 获取当前网络状态：

```bash
./dlut_auto_login_rs --info
```

### 强制登录（跳过在线状态检查）：

```bash
./dlut_auto_login_rs -u <username> -p <password> --force
```

### 守护进程模式（定期检查并登录）：

```bash
./dlut_auto_login_rs -u <username> -p <password> --daemon
```

### 指定 IP 地址：

```bash
./dlut_auto_login_rs -u <username> -p <password> -i <ip_address>
```

### 获取帮助：

```bash
./dlut_auto_login_rs --help
```

## 命令行选项

- `-u, --username <USERNAME>`: 用户名
- `-p, --password <PASSWORD>`: 密码
- `-i, --ip <IP>`: IPV4 地址（可选，未提供时将自动检测）
- `--pid <PATH>`: PID 文件路径（用于确保单实例运行）
- `--info`: 获取 drcom 信息并退出
- `--force`: 登录前跳过状态检查
- `--daemon`: 守护进程模式（定期检查状态并在掉线时登录）

## 进程管理

### 使用 PID 文件

使用 `--pid` 选项确保只有一个实例在运行：

```bash
./dlut_auto_login_rs -u <username> -p <password> --daemon --pid /tmp/dlut_login.pid
```

### 在 Linux 上使用 (systemd)

创建 `/etc/systemd/system/dlut-login.service`：

```bash
sudo tee /etc/systemd/system/dlut-login.service > /dev/null <<'EOF'
[Unit]
Description=DLUT Campus Network Auto Login Daemon
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
ExecStart=/path/to/dlut_auto_login_rs -u 你的学号 -p 你的密码 --daemon
Restart=always
RestartSec=10

[Install]
WantedBy=multi-user.target
EOF
```

然后启用并启动：

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now dlut-login
```


### 在 OpenWrt 上使用 (procd)

创建文件 `/etc/init.d/dlut_login`：

```bash
#!/bin/sh /etc/rc.common

START=99
USE_PROCD=1

start_service() {
    procd_open_instance
    procd_set_param command /usr/bin/dlut_auto_login_rs -u "你的用户名" -p "你的密码" --daemon
    procd_set_param respawn
    procd_close_instance
}
```

然后启用服务：

```bash
chmod +x /etc/init.d/dlut_login
/etc/init.d/dlut_login enable
/etc/init.d/dlut_login start
```

### 在 Windows 上使用

1.  **任务计划程序 (推荐)**：
    - 打开“任务计划程序”。
    - 创建新任务，设置触发器为“登录时”。
    - 操作设置为“启动程序”，选择 `dlut_auto_login_rs.exe`。
    - 添加参数：`-u 用户名 -p 密码 --daemon --pid C:\path\to\login.pid`。
2.  **启动文件夹**：
    - 按下 `Win + R`，输入 `shell:startup`。
    - 在该目录下创建一个指向 `dlut_auto_login_rs.exe` 的快捷方式。
    - 右键快捷方式 -> 属性 -> 目标，在路径后添加参数。

### 在 macOS 上使用 (launchd)

创建 `~/Library/LaunchAgents/com.user.dlut_login.plist`：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>Label</key>
    <string>com.user.dlut_login</string>
    <key>ProgramArguments</key>
    <array>
        <string>/path/to/dlut_auto_login_rs</string>
        <string>-u</string>
        <string>你的用户名</string>
        <string>-p</string>
        <string>你的密码</string>
        <string>--daemon</string>
        <string>--pid</string>
        <string>/tmp/dlut_login.pid</string>
    </array>
    <key>RunAtLoad</key>
    <true/>
    <key>KeepAlive</key>
    <true/>
</dict>
</plist>
```

然后加载任务：

```bash
launchctl load ~/Library/LaunchAgents/com.user.dlut_login.plist
```

## 项目结构

- `src/main.rs` - 入口点和 CLI 处理
- `src/des_crypto.rs` - 3DES 加密/解密实现
- `src/drcom.rs` - DrCOM 协议处理（网络状态检查）
- `src/login.rs` - 登录逻辑和表单处理

## 依赖项

- `ureq` - 极简同步 HTTP 客户端（支持 JSON 和 Cookies）
- `cookie_store` - Cookie 存储和管理
- `scraper` - HTML 解析和 CSS 选择器
- `serde` - 序列化框架
- `serde_json` - JSON 解析
- `argh` - 极简命令行参数解析
- `ctrlc` - Ctrl-C 信号处理

## 注意事项

- 本工具针对嵌入式系统（如 OpenWrt）进行了优化
- 移除了交互模式，以便于作为守护进程/脚本使用
- 工具在尝试登录前会检查是否已经在线
- 在守护进程模式下，工具会定期检查网络状态，如果掉线则自动执行登录。

## 特别感谢

原始 Python 项目(https://github.com/yangzhanwu/DLUT_auto_login)
