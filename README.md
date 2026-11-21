# 在 VS Code Remote-SSH 中使用 Linux 服务器上的 Codex CLI：完整配置与排错指南

> **适用场景：** Windows 本地电脑可以正常访问 ChatGPT，并能通过 SSH 登录 Linux 服务器；Linux 服务器不能直接连接 OpenAI。希望 **Codex CLI 在服务器上运行**，直接读取、修改和测试远程项目代码。
>
> **本文方案：** Windows HTTP 代理 + SSH 反向端口转发 + Linux Codex CLI + VS Code Remote-SSH。
>
> **前提提醒：** 仅在服务器管理员、网络管理政策及数据处理规定允许的情况下使用。本文不是绕过组织安全限制的操作指南。如果服务器禁止对外发送代码/数据或禁止 SSH 转发，应先取得授权或使用获批的方案。

## 目录

- [1. 最终效果与核心原理](#1-最终效果与核心原理)
- [2. 环境与示例参数](#2-环境与示例参数)
- [3. Windows：检查本地 HTTP 代理](#3-windows检查本地-http-代理)
- [4. Windows：用 SSH 建立反向代理隧道](#4-windows用-ssh-建立反向代理隧道)
- [5. Linux：验证整条网络链路](#5-linux验证整条网络链路)
- [6. Linux：安装 Codex 并登录 ChatGPT](#6-linux安装-codex-并登录-chatgpt)
- [7. VS Code：使用远程终端中的 Codex](#7-vs-code使用远程终端中的-codex)
- [8. 一次设置快捷命令 cdx](#8-一次设置快捷命令-cdx)
- [9. 每次开机的使用流程](#9-每次开机的使用流程)
- [10. 常见问题与排查顺序](#10-常见问题与排查顺序)
- [11. 安全与使用边界](#11-安全与使用边界)
- [12. 参考文档](#12-参考文档)

## 1. 最终效果与核心原理

```text
┌──────────────────────── Windows 本地电脑 ────────────────────────┐
│                                                                │
│  VS Code + Remote-SSH               本地 HTTP/Mixed 代理        │
│       │                                 127.0.0.1:7890          │
│       │                                          ▲             │
│       │          SSH 反向端口转发 (-R)            │             │
└───────┼──────────────────────────────────────────┼─────────────┘
        │                                          │
        ▼                                          │
┌──────────────────────── Linux 服务器 ────────────┼─────────────┐
│                                                  │             │
│  远程项目目录 → VS Code 终端 → Codex CLI          │             │
│                                    │             │             │
│                                    ▼             │             │
│                         127.0.0.1:38890 ─────────┘             │
│                         （只监听服务器本机）                    │
└────────────────────────────────────────────────────────────────┘
                                                     │
                                                     ▼
                                               OpenAI 服务
```

**关键区别：**

- **Codex 运行在哪里？** 运行在 Linux 服务器上，因此可以直接处理服务器的代码和文件。
- **模型请求从哪里联网？** Codex 访问服务器本机的 `38890`，SSH 将其转给 Windows 的 `7890` HTTP 代理，再由 Windows 已获准使用的网络连接访问 OpenAI。
- **VS Code 扮演什么角色？** 用 Remote-SSH 编辑服务器文件，并提供远程集成终端。
- **为什么是 `ssh -R` 而非 `ssh -L`？** `-R` 让**远程服务器上的端口**通过 SSH 连接到**本地电脑上的代理**。
- **登录 ChatGPT 和网络连通是两件事：** 完成账号登录后，Codex 日常请求仍需要可用网络链路。

## 2. 环境与示例参数

| 项目 | 文中示例 | 说明 |
|---|---|---|
| 本地电脑 | Windows 10/11 | 安装 VS Code、OpenSSH 客户端 |
| 远程服务器 | Ubuntu/Linux | 已开启 SSH 服务 |
| 远程用户名 | `student` | **替换为自己的用户名** |
| 服务器地址 | `192.0.2.10` | 文档专用示例 IP，**替换为实际地址** |
| Windows HTTP/Mixed 代理 | `127.0.0.1:7890` | 按代理软件中的真实端口修改 |
| Linux 反向转发端口 | `127.0.0.1:38890` | 选择服务器上未占用的端口 |
| SSH 主机别名 | `myserver` | VS Code 中使用的连接别名 |
| Linux 项目目录 | `/home/student/project` | 按实际项目修改 |

> `192.0.2.10` 是示例地址，并非真实可登录的服务器。**不要把自己的公网地址、用户名密码、代理订阅、API 密钥、设备码或 `auth.json` 上传到公开 GitHub 仓库。**

准备条件：

1. Windows 能通过代理访问 OpenAI 服务，且 HTTP/Mixed 端口可用。
2. Windows 能通过 SSH 连接 Linux 服务器（如需校内 VPN，先正常连接）。
3. 服务器管理员允许相应的 SSH 转发和外部 AI 服务访问。
4. Linux 用户有安装 Codex 的权限，并能正常运行所需的 Node.js/npm 环境（若使用 npm 安装方式）。

## 3. Windows：检查本地 HTTP 代理

以下命令都在 **Windows PowerShell** 执行，不能在 Linux 的 SSH 终端中执行。

先在代理软件中核对 HTTP 或 Mixed 端口（本例为 `7890`），然后检查 TCP 监听：

```powershell
Test-NetConnection 127.0.0.1 -Port 7890
```

期望看到：

```text
TcpTestSucceeded : True
```

注意：这**仅表示端口可连接**，还不能证明代理能访问 OpenAI。

继续验证 HTTP CONNECT：

```powershell
curl.exe -I -x http://127.0.0.1:7890 https://api.openai.com/v1/models
```

若看到 `HTTP/1.1 200 Connection established`（代理隧道建立成功），随后收到 `HTTP/2 401` / `401 Unauthorized`，说明这条无凭据的测试请求已经到达 OpenAI API。`401` 在这个测试里是预期结果。

如果端口不对或不是 HTTP/Mixed 协议，请先修正代理软件设置，不要急着修改服务器。

## 4. Windows：用 SSH 建立反向代理隧道

在 **Windows PowerShell** 中运行（替换用户名与服务器地址）：

```powershell
ssh -o ExitOnForwardFailure=yes -o ServerAliveInterval=30 -o ServerAliveCountMax=3 -N -R 127.0.0.1:38890:127.0.0.1:7890 student@192.0.2.10
```

参数说明：

| 参数 | 作用 |
|---|---|
| `-R 127.0.0.1:38890:127.0.0.1:7890` | 让服务器的 `38890` 连接经 SSH 到达 Windows 的 `7890` |
| `-N` | 只建立隧道，不在该窗口打开远程 Shell |
| `ExitOnForwardFailure=yes` | 转发创建失败时直接报错退出 |
| `ServerAliveInterval=30` | 定期检查 SSH 连接活性 |
| `ServerAliveCountMax=3` | 连续多次无响应后结束会话 |

首次连接时，核对服务器 SSH 主机指纹后再确认；随后输入 SSH 密码（输入时不显示字符）。**成功后 PowerShell 窗口通常会一直运行，没有返回提示符，这是正常的。不要关闭它。**

### 可选：创建 Windows 一键启动 BAT

桌面新建 `启动Codex代理.bat`，内容如下：

```bat
@echo off
title Codex SSH Proxy
echo Connecting to Linux server...

ssh -o ExitOnForwardFailure=yes -o ServerAliveInterval=30 -o ServerAliveCountMax=3 -N -R 127.0.0.1:38890:127.0.0.1:7890 student@192.0.2.10

echo SSH tunnel ended. Check proxy, server and network.
pause
```

将示例账号和地址替换为自己的配置。下次双击 BAT、输入 SSH 密码即可。此脚本**不会自动启动 Windows 代理软件**，也**不会在断线后自动重连**。

> 不要同时启动两个都占用服务器 `38890` 端口的 BAT/SSH 转发进程。

## 5. Linux：验证整条网络链路

另开一个 SSH 终端（或通过 VS Code Remote-SSH）登录服务器。首先检查服务器上是否正在监听 `38890`：

```bash
ss -ltn | grep ':38890'
```

如看到 `127.0.0.1:38890` 的 `LISTEN`，说明转发端口已建立，但仍需测试实际请求。

运行：

```bash
curl -I --max-time 20 \
  -x http://127.0.0.1:38890 \
  https://api.openai.com/v1/models
```

常见正常结果：

```text
HTTP/1.1 200 Connection established
HTTP/2 401
```

**注意：** `curl -x` 是对本次请求显式指定代理；它成功**不代表**当前 Shell 已设置环境变量，更不能保证登录域名也一定可达。

为后续命令设置代理（只影响当前 Shell 及其子进程）：

```bash
export http_proxy=http://127.0.0.1:38890
export https_proxy="$http_proxy"
export HTTP_PROXY="$http_proxy"
export HTTPS_PROXY="$http_proxy"
export ALL_PROXY="$http_proxy"
export all_proxy="$http_proxy"
```

检查：

```bash
echo "$https_proxy"
```

若想取消当前 Shell 中的代理设置：

```bash
unset http_proxy https_proxy HTTP_PROXY HTTPS_PROXY ALL_PROXY all_proxy
```

## 6. Linux：安装 Codex 并登录 ChatGPT

### 6.1 安装与检查

如果已安装 Codex，可以跳过安装步骤：

```bash
node -v
npm -v
npm install -g @openai/codex@latest
codex --version
```

如果 Node.js/npm 不存在，先根据服务器系统、架构和管理员规定安装受支持的 Node.js 环境；共享服务器建议使用用户级安装方式，不随意改动系统 Python/Conda/CUDA 环境。

### 6.2 处理曾经使用的第三方 API 配置（按需）

Codex 的配置及认证文件可能位于 `~/.codex/`（或由 `CODEX_HOME` 指定）。常见文件包括：

- `auth.json`：可能包含访问令牌，属于**敏感凭据**。
- `config.toml`：可能含第三方 `model_provider`、`base_url` 等配置。

如果原来用过第三方密钥，**先备份再调整**，不要直接删除：

```bash
mkdir -p ~/.codex/backup
cp -p ~/.codex/auth.json ~/.codex/backup/auth.json.bak 2>/dev/null || true
cp -p ~/.codex/config.toml ~/.codex/backup/config.toml.bak 2>/dev/null || true
```

备份目录同样含敏感内容；请确保只有自己能访问，**不要提交 GitHub**。如有自定义服务商配置，检查并按需取消其覆盖。可以在当前终端取消旧 API 环境变量：

```bash
unset OPENAI_API_KEY OPENAI_BASE_URL
```

仅取消变量不会清空历史登录状态。可以通过 `codex login status` 查看认证方式，并在确有需要时执行 `codex logout` 后重登。

### 6.3 无图形界面服务器：设备码登录

在已设置代理环境变量的 Linux 终端运行：

```bash
codex login --device-auth
```

在自己的 Windows 浏览器中打开终端给出的授权网页，使用 ChatGPT 账号登录，输入终端显示的一次性设备码。**设备码不要截图公开**。

如果提示 403，除了网络问题，还需要确认 ChatGPT 账号/工作空间允许设备码登录。`api.openai.com` 能返回 401 **不代表**设备码认证服务一定可用。

完成后检查：

```bash
codex login status
```

显示使用 ChatGPT 登录即可。**ChatGPT 套餐中的 Codex 使用额度与 API Key 按量计费是不同机制；Codex 额度仍受套餐及工作空间限制。**

### 6.4 验证模型是否能真实回复

在暂时不修改文件的前提下测试：

```bash
codex exec --sandbox read-only --skip-git-repo-check "请只回复 OK，不运行命令，也不要修改文件"
```

如果回复 `OK`，说明认证、代理与至少一个模型请求已经能正常工作。

## 7. VS Code：使用远程终端中的 Codex

1. Windows 安装 VS Code 和官方 **Remote - SSH** 扩展。
2. 在 VS Code 中选择 `Remote-SSH: Connect to Host...`，连接 `myserver`。
3. 打开远程项目目录。
4. 点击 `Terminal → New Terminal`，确认命令提示符是 Linux 服务器上的用户，而不是 Windows PowerShell。
5. 进入项目目录并启动 Codex：

```bash
cd /home/student/project
codex --no-daemon --no-alt-screen
```

某些 SSH 终端/CLI 版本直接运行 `codex` 可能出现启动超时、错误退出或 `Conversation interrupted`。本次实践中，下面两个参数有效：

- `--no-daemon`：不使用共享后台服务运行交互式 CLI。
- `--no-alt-screen`：关闭备用屏幕模式，提高某些终端的兼容性。

这是**针对该环境验证有效的兼容方案**，不代表所有人都必须使用这两个参数。

首次建议输入：

```text
请先阅读当前项目的主要 Python 文件，介绍项目结构与训练流程。
暂时不要修改文件，也不要运行耗时实验。
```

**重要：本文已验证的是 VS Code 下方远程终端中的 Codex CLI。** VS Code 侧边栏的 Codex 图形扩展是另一种运行方式，可能有独立的扩展宿主和代理机制；仅设置 `~/.bashrc` 或 VS Code 的 `http.proxy` 并不能保证它自动生效。

## 8. 一次设置快捷命令 cdx

每次新建 Linux 终端，之前手动执行的 `export` 不一定保留。可以把代理注入与兼容启动参数做成一个 Bash 函数，只给 Codex 生效，避免影响训练程序和 Conda。

在 Linux 中编辑 `~/.bashrc`，**只追加一次**以下内容：

```bash
# Start Codex with an SSH-forwarded HTTP proxy.
cdx() (
    export http_proxy=http://127.0.0.1:38890
    export https_proxy="$http_proxy"
    export HTTP_PROXY="$http_proxy"
    export HTTPS_PROXY="$http_proxy"
    export ALL_PROXY="$http_proxy"
    export all_proxy="$http_proxy"

    command codex --no-daemon --no-alt-screen "$@"
)
```

加载：

```bash
source ~/.bashrc
type cdx
```

以后在项目目录中输入：

```bash
cd /home/student/project
cdx
```

即可。

> `cdx` 只负责设置 Codex 的代理环境和启动参数；它**不会自动建立 Windows SSH 隧道**。这个函数适用于 Bash。若使用 Zsh，需要改写到 `~/.zshrc`。

## 9. 每次开机的使用流程

初次安装、设备码登录和创建快捷函数完成后，日常流程通常只需：

1. **Windows：** 启动有权限使用的 HTTP/Mixed 代理，确认端口（示例 `7890`）。
2. **Windows：** 双击 `启动Codex代理.bat`，输入 SSH 密码，保持窗口开启。
3. **Windows：** 打开 VS Code，使用 Remote-SSH 连接服务器（示例别名 `myserver`）。
4. **VS Code：** 打开 Linux 项目目录，启动远程终端。
5. **Linux 终端：** 运行 `cdx`，即可开始使用 Codex CLI。

**通常不需要重复登录 ChatGPT，也不需要重新安装 Codex。** 但 BAT 对应的 SSH 隧道断开后必须重新连接，除非另行配置了获准使用的持久网络方案。

## 10. 常见问题与排查顺序

| 现象 | 可能原因 | 建议检查 |
|---|---|---|
| Windows `TcpTestSucceeded: False` | 7890 未监听 / 端口不正确 | 核对本机代理软件端口 |
| Windows `curl -x` 不通 | HTTP/Mixed 协议不匹配或上游网络问题 | 先修复 Windows 本地代理 |
| Linux `Connection refused` | 38890 无监听 / BAT 已关闭 | `ss -ltn \| grep ':38890'` |
| Linux `Proxy CONNECT aborted` / `Connection reset by peer` | 反向隧道或 Windows 7890 目标连接中断 | 先测 Windows `curl.exe`，再看 SSH 调试输出 |
| `remote port forwarding failed` | 服务器端口被占用或服务端禁止转发 | 查端口占用和 SSH 管理策略 |
| `device code ... 403 Forbidden` | 认证服务访问限制或设备码登录未获允许 | 检查账号/工作空间设备码权限和认证网络 |
| `codex` 一发消息就退出 | 后台服务或终端显示兼容问题 | 试 `codex --no-daemon --no-alt-screen` |
| `workspace routing discovery timed out` | 后台服务启动或认证网络请求超时 | 先测 `codex exec`，再尝试 `--no-daemon` |
| `Conversation interrupted` | 会话被中断，或 CLI/网络发生故障 | 检查隧道、代理和兼容启动参数 |
| VS Code 报 `Could not establish connection` | SSH 配置的重复转发或其他 SSH 错误 | 检查 Remote-SSH 输出与 SSH 配置 |
| `curl.exe: command not found` | 在 Linux 中运行了 Windows 命令 | Windows 用 `curl.exe`，Linux 用 `curl` |

### 特别重要：不要让 BAT 和 VS Code 抢同一个端口

本次实践曾经遇到：将下面两项添加到 Windows `~/.ssh/config` 中的 `Host myserver` 后，VS Code Remote-SSH 连接失败。

```sshconfig
RemoteForward 127.0.0.1:38890 127.0.0.1:7890
ExitOnForwardFailure yes
```

原因可能是：BAT 已经占用了服务器 `38890`，而 VS Code 的 SSH 会话再次尝试创建同名转发；在启用失败即退出时，会直接影响 VS Code 连接。

**本指南的稳定做法：**

- **BAT：专门负责创建反向隧道。**
- **VS Code Remote-SSH：只负责连接服务器，不重复配置该端口转发。**
- **`cdx`：仅负责给 Codex CLI 设置代理并启动。**

如果 VS Code 已经受影响，先从对应 `Host` 段删除上述额外转发设置，保留原来的 `HostName`、`User`、`Port` 等连接参数，随后重连。

### 推荐的最小排错顺序

```text
Windows 7890 是否监听？
    ↓
Windows curl.exe -x 是否能访问 api.openai.com？
    ↓
Windows SSH -R 隧道是否成功建立、且没有退出？
    ↓
Linux 38890 是否监听？
    ↓
Linux curl -x 是否返回 API 的 401？
    ↓
当前终端是否设置代理环境变量？
    ↓
codex login status / codex exec 是否成功？
    ↓
交互式 codex --no-daemon --no-alt-screen 是否正常？
```

按照这个顺序排查，比反复更换端口、卸载重装 Codex 或删除认证文件更容易定位问题。

## 11. 安全与使用边界

- **取得授权：** 服务器可能属于学校、实验室或公司。任何代理、SSH 转发、远程 AI 代码分析都应符合管理员要求及数据保密政策。
- **限制监听范围：** `-R 127.0.0.1:38890:...` 将远程监听限制在服务器回环地址；不要无必要地将代理开放到 `0.0.0.0` 或局域网。
- **保护认证信息：** `~/.codex/auth.json`、历史备份、API 密钥、会话令牌、设备码、代理订阅链接等绝不能上传到公开仓库。
- **脱敏截图：** 遮住账号密码、设备码、`Set-Cookie`、授权请求头和内部网络信息。
- **最小权限：** 在受控共享服务器上保留 Codex 的操作审批和沙盒限制；不要盲目开启危险的全权限执行。
- **先读后改：** 修改科研代码前先使用 Git 提交/备份，让 Codex 先解释计划，再审查 diff，最后进行测试。
- **代理只给需要的程序：** 优先使用 `cdx` 函数单独设置代理，而不是把代理写成所有会话和训练程序的全局默认值。
- **分享仓库时：** 本 README 中的 IP、用户名、项目路径都已使用示例值。建议把任何本机配置文件和真实凭据加入 `.gitignore`，并在 `git status` 中检查待提交内容。

## 12. 参考文档

- [OpenAI：Codex CLI](https://developers.openai.com/codex/cli/)
- [OpenAI：Codex Authentication（设备码登录、凭据安全）](https://developers.openai.com/codex/auth/)
- [OpenAI：Codex CLI 命令与参数（`--no-daemon`、`--no-alt-screen`）](https://developers.openai.com/codex/cli/reference/)
- [OpenAI：Using Codex with your ChatGPT plan](https://help.openai.com/en/articles/11369540-using-codex-with-your-chatgpt-plan)
- [VS Code：Remote Development using SSH](https://code.visualstudio.com/docs/remote/ssh)
- [VS Code：Remote Development Tips and Tricks](https://code.visualstudio.com/docs/remote/troubleshooting)

---

**一句话总结：** Windows 代理提供网络出口，SSH `-R` 将该出口映射为 Linux 服务器本机端口，Codex CLI 在项目目录运行并通过该端口联网，VS Code Remote-SSH 负责远程编辑。第一次配置完成后，通常只需 **启动代理 → 双击 BAT → VS Code 连服务器 → 输入 `cdx`**。

> 本文整理自一次实际的 Windows + Linux + VS Code + Codex CLI 配置与排错过程。CLI 参数、认证权限和软件界面可能随版本更新而变化；遇到差异时，请以最新官方文档和 `codex --help` 为准。
