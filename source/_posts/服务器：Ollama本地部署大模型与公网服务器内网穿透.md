---
title: 服务器：Ollama本地部署大模型与公网服务器内网穿透
date: 2025-12-01 17:58:55
tags: 服务器
---

# 概览

Windows 机使用 `ollama` 本地部署 `qwen`，云服务器使用内网穿透与 Windows 机连接，实现其他机器（如 Mac）远程调用 Windows 运行的大模型服务。

## ollama 本地部署大模型

### 安装与验证

1. 在 [ollama 官网](https://ollama.com/) 下载对应操作系统版本
2. 本地 `cmd` 输入 `ollama`，检测是否安装成功

```bash
ollama
```

### 拉取与运行模型

3. 拉取模型到本地，以 `qwen2.5:3b-instruct` 为例：

```bash
ollama pull qwen2.5:3b-instruct
```

4. 拉取完成后，运行模型，检查模型是否正常输出，检查本地 GPU 是否有资源占用（防止模型使用 CPU 运行，降低运行效率）：

```bash
ollama run qwen2.5:3b-instruct
```

### 测试

5. 直接使用命令行交互式测试，或使用 `curl` 命令调用 API 测试：

```bash
curl http://localhost:11434/api/generate -d '{
  "model": "qwen2.5:3b-instruct",
  "prompt": "你好"
}'
```

## 内网穿透

### 原理

本质就是以有公网 IP 的服务器作为跳板机，使得访问服务器的服务，转发到本地 Windows 机的服务。

这里使用 `frp` 作为内网穿透工具，来高效轻量搭建反向代理隧道。

**下载地址**：https://github.com/fatedier/frp/releases

### 架构流程

```
┌─────────────┐
│  公网用户    │ (Mac/其他设备)
│  (Client)   │
└──────┬──────┘
       │ HTTP请求
       │ <server-ip>:11434
       ▼
┌─────────────────────────┐
│   公网服务器 (frps)      │
│   - 监听端口: 11434      │
│   - 控制端口: 7000       │
└──────┬──────────────────┘
       │ 转发请求
       │ (TCP隧道)
       ▼
┌─────────────────────────┐
│  Windows 机 (frpc)       │
│   - 连接服务器: 7000     │
│   - 本地服务: 11434      │
└──────┬──────────────────┘
       │ 访问本地服务
       ▼
┌─────────────────────────┐
│   本地 Ollama 服务       │
│   - 端口: 11434          │
│   - 模型: qwen2.5:3b     │
└─────────────────────────┘
```

**数据流向**：`公网用户 → 公网服务器(frps) → Windows(frpc) → 本地 ollama`

### 服务器配置

#### 1. 下载与解压

从下载地址下载对应版本，如 `frp_0.65.0_linux_amd64.tar.gz` 版本，并解压：

```bash
# 下载对应版本的压缩包
curl -fsLO https://github.com/fatedier/frp/releases/download/v0.65.0/frp_0.65.0_linux_amd64.tar.gz

# 解压压缩包
tar -xzf frp_0.65.0_linux_amd64.tar.gz
```

#### 2. 配置参数

在 `frps.toml` 中配置参数：

```toml
bindPort = 7000
bindAddr = "0.0.0.0"

auth.method = "token"
auth.token = "root1234"

transport.tls.force = false

[[allowPorts]]
single = 11434
```

#### 3. 启动 frps

```bash
./frps -c frps.toml
```

### Windows 本机配置

#### 1. 下载与解压

从下载地址下载对应版本，如 `frp_0.65.0_windows_amd64.zip` 版本，并解压。

#### 2. 配置参数

在 `frpc.toml` 中配置参数：

```toml
serverAddr = <server ip>
serverPort = 7000
auth.token = "root1234"

[[proxies]]
name = "ollama run qwen2.5:3b-instruct"
type = "tcp"
localIP = "0.0.0.0"
localPort = 11434
remotePort = 11434
```

#### 3. 开放 Ollama 端口

在 Windows 中配置环境变量 `OLLAMA_HOST=0.0.0.0:11434`，并**重启电脑**使环境变量生效（一定要重启电脑）。

> **注意**：不配置这个环境变量的话，`ollama` 的服务默认运行在 `127.0.0.1:11434`，不对外暴露服务。

#### 4. 启动 frpc

```bash
.\frpc -c frpc.toml
```

> **注意**：Windows 与 Linux 命令的差异，Windows 使用 `.\frpc`，Linux 使用 `./frps`。

#### 5. 测试

调用公网 IP 的端口，测试内网穿透到本机大模型服务是否成功：

```bash
curl http://<server ip>:11434/api/generate -d '{
  "model": "qwen2.5:3b-instruct",
  "prompt": "你好"
}'
```

### 编写脚本快速启动和关闭

#### 服务端脚本

**特性**：

- 使用 `nohup` 命令运行，确保关闭 SSH 会话后进程继续运行
- 启动前会自动检查是否已有进程在运行

新建启动文件 `start_frps.sh`：

```bash
#!/bin/bash

# FRP 服务端启动脚本
# 使用 nohup 确保关闭会话后进程继续运行

SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
FRPS_BIN="$SCRIPT_DIR/frps"
FRPS_CONFIG="$SCRIPT_DIR/frps.toml"

# 检查 frps 可执行文件是否存在
if [ ! -f "$FRPS_BIN" ]; then
    echo "错误: 找不到 frps 可执行文件: $FRPS_BIN"
    exit 1
fi

# 检查配置文件是否存在
if [ ! -f "$FRPS_CONFIG" ]; then
    echo "错误: 找不到配置文件: $FRPS_CONFIG"
    exit 1
fi

# 检查进程是否在运行（精确匹配进程名）
if pgrep -x "frps" > /dev/null; then
    echo "警告: 检测到 frps 进程正在运行，请先使用 stop_frps.sh 停止"
    exit 1
fi

# 启动 frps
echo "正在启动 frps..."
cd "$SCRIPT_DIR"
nohup "$FRPS_BIN" -c "$FRPS_CONFIG" > /dev/null 2>&1 &
FRPS_PID=$!

# 等待一下，检查进程是否成功启动
sleep 2
if ps -p "$FRPS_PID" > /dev/null 2>&1; then
    echo "frps 启动成功!"
    echo "PID: $FRPS_PID"
    echo "配置文件: $FRPS_CONFIG"
    echo ""
    echo "提示: 使用 './stop_frps.sh' 停止服务"
else
    echo "错误: frps 启动失败"
    exit 1
fi
```

新建关闭文件 `stop_frps.sh`：

```bash
#!/bin/bash

# FRP 服务端停止脚本

# 检查 frps 是否在运行（精确匹配进程名）
if pgrep -x "frps" > /dev/null; then
    echo "正在停止 frps..."
    # 先尝试正常终止
    pkill -x "frps"

    # 等待进程结束
    sleep 2

    # 如果进程仍在运行，强制终止
    if pgrep -x "frps" > /dev/null; then
        echo "进程未正常退出，强制终止..."
        pkill -9 -x "frps"
        sleep 1
    fi

    # 确认进程已停止
    if pgrep -x "frps" > /dev/null; then
        echo "错误: 无法停止 frps"
        exit 1
    else
        echo "frps 已成功停止"
    fi
else
    echo "frps 未运行"
fi

exit 0
```

**添加文件操作权限**，后续直接运行文件即可：

```bash
chmod +x start_frps.sh stop_frps.sh
```

**使用方法**：

```bash
# 启动服务
./start_frps.sh

# 停止服务
./stop_frps.sh
```

#### 客户端脚本

**特性**：

- 自动重连机制

新建启动文件 `start_frpc.bat`：

```batch
@echo off
REM FRP Client Startup Script (with auto-reconnection mechanism)
REM When frpc exits due to network fluctuations or other reasons, it will automatically restart

setlocal enabledelayedexpansion

REM Get script directory
set "SCRIPT_DIR=%~dp0"
set "FRPC_BIN=%SCRIPT_DIR%frpc.exe"
set "FRPC_CONFIG=%SCRIPT_DIR%frpc.toml"
set "RESTART_DELAY=5"

REM Check if frpc.exe exists
if not exist "%FRPC_BIN%" (
    echo Error: frpc.exe file not found: %FRPC_BIN%
    pause
    exit /b 1
)

REM Check if config file exists
if not exist "%FRPC_CONFIG%" (
    echo Error: Config file not found: %FRPC_CONFIG%
    pause
    exit /b 1
)

REM Check if frpc is already running
tasklist /FI "IMAGENAME eq frpc.exe" 2>NUL | find /I /N "frpc.exe">NUL
if "%ERRORLEVEL%"=="0" (
    echo Warning: Detected frpc.exe is already running, please stop it first using stop_frpc.bat
    pause
    exit /b 1
)

echo ========================================
echo FRP Client Startup Script (with auto-reconnection)
echo ========================================
echo Config file: %FRPC_CONFIG%
echo Restart delay: %RESTART_DELAY% seconds
echo ========================================
echo.
echo Tip: Press Ctrl+C to stop the service
echo.

REM Set loop counter
set "RESTART_COUNT=0"

:LOOP
set /a RESTART_COUNT+=1

REM Display startup information
set "CURRENT_TIME=%DATE% %TIME%"
echo [%CURRENT_TIME%] Starting frpc (Attempt %RESTART_COUNT%)...

REM Run frpc
cd /d "%SCRIPT_DIR%"
"%FRPC_BIN%" -c "%FRPC_CONFIG%"
set "EXIT_CODE=%ERRORLEVEL%"

REM Display exit information
set "CURRENT_TIME=%DATE% %TIME%"
echo [%CURRENT_TIME%] frpc exited with code: %EXIT_CODE%

REM If user presses Ctrl+C, exit code may be 0 or non-zero, but mainly check if interrupted
REM We always try to restart here (unless user stops manually)

REM Wait for specified time before restarting
echo [%CURRENT_TIME%] Auto-restarting in %RESTART_DELAY% seconds...
timeout /t %RESTART_DELAY% /nobreak >nul

REM Check if should continue (can add conditional checks here)
REM If user wants to stop, should use stop_frpc.bat

goto LOOP
```

**使用方法**：直接双击对应的 `.bat` 文件即可启动, 关闭 cmd 窗口即可关闭。
