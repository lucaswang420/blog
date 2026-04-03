---
tags:
  - Docker
  - OpenClaw
  - 容器化部署
  - AI Agent
title: OpenClaw容器化部署完全手册 Docker Compose实战教程
categories:
  - 08-云原生运维
  - 01-容器化
description: >-
  保姆级OpenClaw Docker部署教程，详细介绍环境准备、Docker
  Compose部署、配置文件设置、IM机器人接入(飞书/Telegram)及安全隧道配置，包含完整的故障排除指南。
abbrlink: 2ef3f333
date: 2026-04-03 10:32:00
---

# OpenClaw容器化部署完全手册：Docker Compose实战教程

> 本教程详细介绍如何使用 Docker Compose 部署 OpenClaw AI Agent Gateway，实现多渠道消息接入与模型路由统一管理。

---

## 一、概述

### 1.1 背景介绍

AI Agent 落地到企业内部，第一个挡在路上的问题就是**接入层**。微信、Telegram、企业微信，每个渠道的协议不一样，消息格式不一样，认证方式不一样。自己写网关代码？写到第三个渠道就想掀桌了。

**OpenClaw 就是解决这个问题的。**

它是一个开源 AI Agent Gateway，用 Node.js 运行时构建，把多渠道消息接入、模型路由、Token 认证这些脏活累活全包了。后端对接任意 OpenAI 兼容 API（包括 vLLM、Ollama 等自托管方案），前端对接各种 IM 渠道，中间做协议转换和流量管理。

---

### 1.2 技术特点

| 特性 | 说明 |
|------|------|
| **多渠道网关统一接入** | HTTP API、WebSocket 双协议复用同一端口（`18789`），支持 Telegram Bot、企业微信、Slack 等 IM 渠道接入。消息格式在网关层统一转换，后端模型服务不需要关心渠道差异。 |
| **插件式模型路由** | 通过 `models.providers` 配置自定义 Provider，支持同时对接多个推理后端。模型标识使用 `provider/model` 格式（如 `google/gemini-3.1-flash-lite-preview`），路由逻辑在配置文件中声明，不需要写代码。 |
| **轻量级 Token 认证** | Gateway 内置 Token 认证机制（`gateway.auth.token`），每个接入方分配独立 Token，在网关层完成鉴权。生产环境可配合 K8s NetworkPolicy 实现网络层隔离。 |

---

### 1.3 适用场景

| 场景 | 描述 |
|------|------|
| **企业内部 AI 助手平台** | 统一管理多个部门的 AI 助手，每个部门分配独立 Token 和模型配额。 |
| **多平台消息聚合** | 同一个 AI Agent 同时在 Telegram、企业微信、Slack 上提供服务。OpenClaw 在网关层做消息协议转换，后端只需要维护一套模型服务。 |

---

### 1.4 环境要求

| 组件 | 版本要求 | 说明 |
|:-----|:---------|------|
| **操作系统** | Ubuntu 22.04+ / Debian 12+ | 推荐 Ubuntu 22.04 LTS，内核 5.15+ |
| **Docker** | 24.0+ | 需要 Docker Compose V2 |
| **Node.js** | 24.x | 容器内置，无需手动安装 |
| **GPU (可选)** | NVIDIA A100/H100 | 仅推理服务需要 |

---

## 二、详细步骤

### 2.1 准备工作

先远程连接到服务器，推荐使用**非 root 用户**连接。

#### 2.1.1 系统检查

```bash
# 系统与软件包更新
sudo apt update && sudo apt upgrade -y

# 安装必要依赖项（部署 OpenClaw 需要 git 和 curl）
sudo apt install -y git curl

# 检查系统版本
cat /etc/os-release

# 检查内核版本
uname -r

# 检查 Docker 版本
docker --version
docker compose version

# 检查系统资源
free -h
df -h
```

> [!NOTE]
> **预期输出**：Docker 版本 24.0 以上，Docker Compose V2 能正常输出版本号。

---

#### 2.1.2 安装 Docker（如果尚未安装）

```bash
# Ubuntu/Debian 安装 Docker
curl -fsSL https://get.docker.com | bash -s docker

# 将当前用户加入 docker 组
sudo usermod -aG docker $USER
newgrp docker

# 启动并设置开机自启
sudo systemctl enable --now docker

# 验证安装
docker run hello-world
```

---

#### 2.1.3 准备 OpenClaw 配置目录

```bash
# 创建配置目录与工作空间
mkdir -p ~/openclaw-home/{config,workspace,npm-global,pip-global}

# 确保目录权限正确（容器内部以 node 用户运行，uid 1000）
sudo chown -R 1000:1000 ~/openclaw-home
```

> [!TIP]
> OpenClaw 的配置文件是 JSON5 格式，文件路径固定在 `~/.openclaw/openclaw.json`。由于容器内部以 `node` 用户（uid 1000）运行，挂载目录时需要特别注意权限问题。

---

### 2.2 Docker Compose 官方部署方案（推荐）

> [!TIP]
> 这是官方推荐的部署方式，适合开发测试和小规模生产环境。
> 2.2 和 2.3 选择一种方式即可

#### 2.2.1 使用官方安装脚本

OpenClaw 提供了 `setup.sh` 一键安装脚本，流程是：拉取镜像 → 初始化配置（onboard）→ 启动服务。

```bash
# 克隆官方仓库
git clone https://github.com/openclaw/openclaw.git ~/openclaw-source
cd ~/openclaw-source

# 环境变量配置
export OPENCLAW_IMAGE="ghcr.io/openclaw/openclaw:latest"
export OPENCLAW_CONFIG_DIR="~/openclaw-home/config"
export OPENCLAW_WORKSPACE_DIR="~/openclaw-home/workspace"
export OPENCLAW_TZ="Asia/Shanghai"
export OPENCLAW_EXTRA_MOUNTS="~/openclaw-home/npm-global:/home/node/.npm-global,~/openclaw-home/pip-global:/home/node/.local"

# 一键安装部署
chmod +x ./scripts/docker/setup.sh
./scripts/docker/setup.sh
```

> [!WARNING]
> 如果网络环境不方便直接执行远程脚本，也可以手动操作（详见下文 2.3 节）。

---

##### 环境变量配置说明

`setup.sh` 支持通过环境变量定制行为：

| 环境变量 | 默认值 | 说明 |
|:---------|:-------|------|
| `OPENCLAW_IMAGE` | `ghcr.io/openclaw/openclaw:latest` | 镜像地址，私有仓库可修改此项 |
| `OPENCLAW_GATEWAY_BIND` | `127.0.0.1:18789` | 网关监听地址和端口 |
| `OPENCLAW_HOME_VOLUME` | `~/.openclaw` | 配置文件挂载路径（容器内） |
| `OPENCLAW_CONFIG_DIR` | `~/openclaw-home/config` | 配置文件目录挂载路径（宿主机） |
| `OPENCLAW_WORKSPACE_DIR` | `~/openclaw-home/workspace` | Agent 工作目录挂载路径（宿主机） |
| `OPENCLAW_EXTRA_MOUNTS` | `~/openclaw-home/npm-global:~/npm-global` | 额外挂载路径，用于后续安装 Skills |
| `OPENCLAW_SANDBOX` | `0` | 设为 `1` 启用沙箱模式 |

---

#### 2.2.2 初始化配置（Onboard）

建议先只配置模型供应商（Provider）及其 API Key，并选择默认模型。其余步骤可暂且忽略，后续再按需配置。
> [!WARNING]
> skills安装步骤一定要先跳过，因为Onboard环节使用npm全局安装skills需要root权限，而我们用的时非root用户，所以需要下一步的设置后，再进行Onboard安装skills。

**1. 在宿主机预先准备环境初始化脚本**

为了避免以后容器重启导致 `~/.bashrc` 环境变量丢失，你可以先在宿主机挂载的 `config` 目录下打包好一个快捷配置脚本 `init-env.sh`：

```bash
# 在宿主机终端执行：生成并赋予权限
cat << 'EOF' > ~/openclaw-home/config/init-env.sh
#!/bin/bash
# 变更 npm 安装路径并配置 pip
npm config set prefix ~/.npm-global
pip config set global.user true

# 更新 PATH 并在 .bashrc 注册（具备去重检测，防止多次执行重复注入）
if ! grep -q ".npm-global/bin" ~/.bashrc; then
  echo 'export PATH=$HOME/.npm-global/bin:$HOME/.local/bin:$PATH' >> ~/.bashrc
fi
echo "✅ 初始化脚本注入成功！"
EOF

chmod +x ~/openclaw-home/config/init-env.sh
```

**2. 运行进入容器执行该脚本**

现在你只需要进入容器，调用这个长期跟随文件挂载的脚本即可，哪怕后续容器重建，一键执行也很方便：

```bash
docker exec -it openclaw-openclaw-gateway-1 bash

# 在容器内调用挂载进去的脚本
/home/node/.openclaw/init-env.sh

# 手动让环境变量立即生效
source ~/.bashrc
```

> [!TIP]
> **最佳不修改 YML 的持久化方案：利用 `.env` 文件**
> Docker 默认会自动读取与 `docker-compose.yml` 同级别的 `.env` 隐藏文件来注入环境变量。
>
> **核心原则：`.env` 必须和 `docker-compose.yml` 放在同一个目录下！**
> 找到 `setup.sh` 为你生成的 `docker-compose.yml` 文件所在目录（通常在使用官方安装脚本时，yml 文件会留在你刚刚执行命令的 `~/openclaw-source` 下，或者在指定的 `~/openclaw-home` 中）。
>
> 请使用 `cd` 命令进入到包含 `docker-compose.yml` 的那个目录中，并在**宿主机（退出容器后）**运行以下指令，即可一劳永逸：
>
> ```bash
> # 仅示范：请确保当前路径下 `ls` 能看到 docker-compose.yml
> echo 'NPM_CONFIG_PREFIX=/home/node/.npm-global' >> .env
> echo 'PATH=/home/node/.npm-global/bin:/home/node/.local/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin' >> .env
> docker compose restart
> ```

**3. 退出容器 bash**

```bash
exit
```

**4. 重启服务**

```bash
docker compose restart
```

#### 2.2.3 检查 OpenClaw 网关服务

```bash
# 查看 OpenClaw 网关服务容器状态
docker ps  # 显示网关名为 openclaw-openclaw-gateway-1
docker compose ps

# 检查 CLI 版本
docker compose run --rm openclaw-cli --version

# 查看网关实时日志
docker logs -f openclaw-openclaw-gateway-1

# 检查状态、安全诊断以及快速修复
docker compose run --rm openclaw-cli status --all
docker compose run --rm openclaw-cli security audit --deep
docker compose run --rm openclaw-cli doctor
docker compose run --rm openclaw-cli doctor --fix
```

#### 2.2.4 Terminal UI (TUI) 验证网关连通性

**1. 启动 TUI**

```bash
docker compose run --rm openclaw-cli tui
```

**2. 和 ClawBot 对话**

在界面中发送 `hi`，验证机器人是否能正常回复。

#### 2.2.5 使用 Control UI 及 Dashboard

**1. 建立 SSH 安全隧道**

> [!IMPORTANT]
> 切记，**不要**开放服务器的 `18789` 端口。

在你的**本地电脑**（客户端）上执行以下命令建立端口转发：

```bash
# 密码认证方式
ssh -N -L 18789:127.0.0.1:18789 user@<服务器IP>

# 密钥认证方式（推荐）
ssh -i /path/to/your/private_key -N -L 18789:127.0.0.1:18789 user@<服务器IP>
```

**2. 本地访问 Dashboard 并配对**

在服务器环境中获取 Dashboard 的访问链接：

```bash
docker compose run --rm openclaw-cli dashboard --no-open
```

你将得到一个类似 `http://127.0.0.1:18789/#token=xxxxx` 的链接。将其复制并在**本地浏览器**中打开。此时页面会提示 **Pairing required**（需要配对）。

回到服务器终端，执行配对批准：

```bash
# 获取待配对设备列表（找到对应的 requestId）
docker compose run --rm openclaw-cli devices list

# 批准配对请求
docker compose run --rm openclaw-cli devices approve <requestId>
```

显示配对成功后，刷新浏览器页面即可正常访问 Dashboard。

---

### 2.3 自定义 Docker Compose 部署方案

#### 2.3.1 编写 Docker Compose 文件

生产环境建议直接通过 `docker-compose.yml` 进行管理，比裸 `docker run` 更易于维护。
进入 `~/openclaw-home/` 目录并创建文件：

```yaml
# ~/openclaw-home/docker-compose.yml
services:
  openclaw-gateway:
    image: ghcr.io/openclaw/openclaw:latest
    container_name: openclaw-gateway
    restart: unless-stopped
    ports:
      # 强烈建议绑定到 127.0.0.1，只允许通过 SSH 隧道访问，防止公网端口扫描
      - "127.0.0.1:18789:18789"
    environment:
      - TZ=Asia/Shanghai # 强制设置容器时区，保证聊天记录和日志时间正确
      - NPM_CONFIG_PREFIX=/home/node/.npm-global # 重定向 npm 全局安装路径
      - PATH=/home/node/.npm-global/bin:/home/node/.local/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin # 追加 npm 和 pip 执行路径
    volumes:
      - ./config:/home/node/.openclaw
      - ./workspace:/home/node/.openclaw/workspace
      - ./npm-global:/home/node/.npm-global
      - ./pip-global:/home/node/.local # 增加：持久化存储 node 用户的 pip 全局包
    user: "1000:1000"
  openclaw-cli:
    image: ghcr.io/openclaw/openclaw:latest
    container_name: openclaw-cli
    entrypoint: ["openclaw"]
    volumes:
      - ./config:/home/node/.openclaw
      - ./workspace:/home/node/.openclaw/workspace
      - ./npm-global:/home/node/.npm-global
      - ./pip-global:/home/node/.local
    environment:
      - NODE_ENV=production
      - TZ=Asia/Shanghai
      - NPM_CONFIG_PREFIX=/home/node/.npm-global
      - PATH=/home/node/.npm-global/bin:/home/node/.local/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
    user: "1000:1000"
    profiles: ["cli"]
```

---

#### 2.3.2 编写 OpenClaw 配置文件

在 `~/openclaw-home/config/` 目录下创建 `openclaw.json`（该文件使用 JSON5 格式，支持注释）：

```json5
{
  "gateway": {
    "port": 18789,
    "mode": "local",
    "bind": "lan", // 监听容器内所有网卡
    "auth": {
      "mode": "token",
      "token": "YOUR_STRONG_RANDOM_TOKEN" // 请务必修改为你自己生成的强验证 Token
    },
    "controlUi": {
      "allowedOrigins": [
        "http://localhost:18789",
        "http://127.0.0.1:18789" // 若后续你采用公网 IP 直连访问后台，请在此处加上 "http://你的公网IP:18789" (前提是开放了安全组)
      ]
    }
  },
  "rateLimit": {
    // 增加基础防护：限制高频恶意调用，保护你的大模型计费
    "maxRequestsPerMinute": 600,
    "maxRequestsPerMinutePerToken": 60
  },
  "agents": {
    "defaults": {
      // 声明工作空间路径，与 docker-compose 的 volumes 映射保持一致
      "workspace": "/home/node/.openclaw/workspace",
      "maxConcurrent": 4
    }
  },
  "models": {
    "default": "google/gemini-3.1-flash-lite-preview",
    "providers": {
      "google": {
        "type": "openai",
        "baseUrl": "https://generativelanguage.googleapis.com/v1beta/openai/",
        "apiKey": "YOUR_GOOGLE_API_KEY", // 你的谷歌 API Key
        "models": [
          "google/gemini-3.1-flash-lite-preview"
        ]
      }
    }
  }
}
```

> [!CAUTION]
> `gateway.auth.token` 的值必须是**强随机字符串**，绝不要使用示例中的说明值！
>
> 可以使用以下命令生成一个安全的 32 字节随机 Token：
>
> ```bash
> openssl rand -hex 32
> ```

---

#### 2.3.3 启动并验证

进入 `~/openclaw-home/` 目录执行以下操作：

```bash
# 启动服务（后台运行）
docker compose up -d

# 查看日志，确认启动成功
docker compose logs -f openclaw-openclaw-gateway-1

# 健康检查
curl -s http://localhost:18789/healthz
# 预期输出：{"ok":true,"status":"live"}

# 就绪检查
curl -s http://localhost:18789/readyz
# 预期输出：{"ready":true}
```

> [!NOTE]
> **验证说明**：
>
> - 如果 `/healthz` 返回 `{"ok":true,"status":"live"}`，说明网关已成功启动。
> - `/readyz` 还会去检查后端模型的连通性。如果你的后端推理服务（例如 vLLM、Ollama）还没部署好，`/readyz` 可能会返回 `not ready`，这是正常现象。

---

### 2.4 IM机器人接入

> [!NOTE]
> 配置新的渠道接入后，需要在 `~/openclaw-home/config/openclaw.json` 中的 `channels` 字段中进行声明，并执行 `docker compose restart` 重启网关容器使之生效。

#### 2.4.1 接入飞书

飞书非常适合国内企业和团队协作环境。你需要前往 **飞书开放平台** 创建一个企业自建应用，以获取机器人的身份凭证。

**步骤 1：获取应用凭证 (App ID / App Secret)**

1. 使用管理员或有权限的账号登录 [飞书开放平台](https://open.feishu.cn/)。
2. 点击 **创建企业自建应用**，填写应用名称（例如：`OpenClaw Assistant`）和描述。
3. 在左侧菜单栏进入 **凭证与基础信息** 页面，获取并复制 **App ID** 和 **App Secret**。
4. 在 **添加应用能力** 中，确保开启 **机器人** 能力。需要开启哪些权限，建议自行搜索。

**步骤 2：在 OpenClaw 中配置**
编辑 `~/openclaw-home/config/openclaw.json`，在顶级配置下（与 `"gateway"`、`"models"` 平级）添加 `"channels"` 字段及其 `feishu` 对象：

```json5
{
  // ... 其他配置项（gateway, models）
  "channels": {
    "feishu": {
      "enabled": true,
      "appId": "cli_你的飞书AppID",
      "appSecret": "你的飞书AppSecret"
    }
  }
}
```

**步骤 3：重启连接通道**

执行以下命令重启网关服务使飞书配置生效：

```bash
docker compose restart
```

然后在飞书客户端内搜索你的机器人名称，打开聊天窗口并发送任意一条消息（例如 `你好`），以此建立双方的第一次通信。受默认安全机制保护，此时你需要在后台进行"通过配对授权"的操作（详见下方 **2.4.3 节**的演示说明）。

---

#### 2.4.2 接入 Telegram

Telegram 的接入最为直接和简单，非常适合极客个人或开放环境。OpenClaw 支持长轮询模式（Polling），无需配置复杂的 Webhook 和公网 IP 限制。

**步骤 1：通过 BotFather 获取 Bot Token**

1. 在 Telegram 搜索栏中寻找官方机器人 **@BotFather**。
2. 发送命令 `/newbot`，根据提示给你的机器人设置一个显示名称和用户名（要求以 `bot` 结尾，如 `my_openclaw_bot`）。
3. BotFather 会生成一串 **HTTP API Token**（例：`123456:ABC-DEF1234ghIkl-zyx57...`），请务必妥善保存泄漏的风险。

**步骤 2：在 OpenClaw 中配置**
依然是编辑 `~/openclaw-home/config/openclaw.json`，在 `"channels"` 中配置 `telegram` 账号：

```json5
{
  // ... 其他配置项（gateway, models）
  "channels": {
    "telegram": {
      "enabled": true,
      "botToken": "填入你的_TELEGRAM_BOT_TOKEN",
      "dmPolicy": "pairing" // pairing 表示新用户第一次发送消息需要在 Dashboard 手动通过请求授权
    }
  }
}
```

> [!TIP]
> 如果你需要同时接入飞书和 Telegram，只需在 `"channels"` 对象下平级包含 `"feishu": {...}` 和 `"telegram": {...}` 即可，OpenClaw 可以极其稳定地同时驱动多端收发。

**步骤 3：重启连接通道**

首先执行重启命令使配置生效：

```bash
docker compose restart
```

打开 Telegram 搜索机器人的用户名并点击 **Start**。这会触发首次通讯。由于代码配置了 `dmPolicy: "pairing"` 安全隔离，机器人为了防止你的 API 被盗用，仅会发送一条申请中的提醒，或完全不予置答。请参考下一步 **2.4.3 节** 的操作批准请求！

---

#### 2.4.3 如何在后台审核并通过配对授权（小白必操作）

如果你希望这台设备（比如你的手机 Telegram 或你的飞书独立客户端）能正常开始调用模型沟通，你需要让服务器这端"同意"它的请求。这也是你保护自己 API 开销不被别人无成本消耗的核心防线。

当你刚在前面两步发送了"第一次联系"的消息后，你可以选择以下两种方式之一进行审批通过操作：

**方式一、命令行一键通过（如果你还在服务器终端内，这个最快）：**

1. 提取正在排队的设备：在你的 Linux 黑框终端里，运行以下指令：

   ```bash
   docker compose run --rm openclaw-cli devices list
   ```

   *(这时候你会看到一排列表，里面记录了你的用户名请求，找到一长串形如 `req_xyz123...` 的字符组合，那个就是它的 **requestId** )*。
2. 运行通过指令：复制那个字符并粘贴在其后：

   ```bash
   docker compose run --rm openclaw-cli devices approve 你的这串requestId
   ```

   审批一旦完成，设备就被拉入了白名单，随时可以畅快交流！

**方式二、可视化看板通过（如果你连好了本地终端）：**

1. 就像我们之前的 **2.2.5 章节** 讲的，确保你的安全隧道开着，使用电脑浏览器去访问你的类似 `http://127.0.0.1:18789/#token=xxx...` 的本地管理面板（Dashboard）。
2. 在主管理面板导航栏寻找诸如 **Devices** 或 **Pending Requests（待授权请求）** 这类菜单页面。
3. 你的帐号刚刚敲出的请求就安安静静躺在那儿等通知。找到它，点旁边那个醒目的 **Approve (通过)** 绿色按钮。妥了！

> [!TIP]
> 针对已经授权的设备通道，往后在任何时间点继续发送消息，服务器都会直接接盘并转到对应模型了，不用重复验证。

---

> 🎉 **恭喜！** 你已全部完成 OpenClaw 的部署及渠道调试闭环！接下来你可以根据业务需要，在 `openclaw.json` 继续随意切换、接入自己想使用的不同的大规模多模态 Provider。
