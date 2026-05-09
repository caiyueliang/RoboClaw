# AGENTS.md

本文档面向在本仓库内工作的代理，帮助快速理解 RoboClaw 的真实结构、运行入口和开发注意事项。内容以当前代码为准，而不是只参考上游 `nanobot` 的默认说明。

## 项目概览

RoboClaw 是一个基于 `nanobot` 扩展出来的机器人智能控制项目。它把通用 AI Agent 框架、多消息通道、网关服务、机器人动作控制、图像服务、避障服务和设备部署脚本整合在同一个仓库里。

当前仓库的核心目标不是单纯聊天，而是打通这条链路：

`自然语言 / App / 多渠道消息 -> Agent 推理与工具调用 -> 网关接口 -> 机器人底层能力 / 外围服务`

从代码现状看，当前已经明显偏向机器人场景，尤其是 `Unitree G1` 的移动、动作、抓取、视觉和安全控制。

## 你会先看到的几个关键目录

- `nanobot/`：主 Python 包，仍然是整个项目的核心。
- `nanobot/agent/`：Agent 主循环、上下文拼装、记忆、子代理、工具注册。
- `nanobot/gateway/`：FastAPI 网关，提供 App 聊天、控制器、技能、能力等 HTTP 路由。
- `nanobot/api/`：一个单独的 OpenAI 兼容 HTTP API，走 `aiohttp`，不是 FastAPI 网关。
- `nanobot/channels/`：多消息渠道接入，包括 Slack、Telegram、Discord、Email、WhatsApp、飞书、钉钉、QQ、微信、Matrix、App 等。
- `nanobot/providers/`：大模型供应商适配层，统一走 provider 抽象。
- `nanobot/skills/`：内置技能目录。除通用技能外，已经包含 `robot-move`、`robot-action`、`robot-camera`、`robot-grasp`、`robot-navigate` 等机器人技能。
- `robot/teleimager/`：图像采集与图传服务，独立 Python 子项目。
- `robot/obstacle_avoid/`：避障与安全状态通知服务，独立 Python 子项目。
- `bridge/`：Node.js/TypeScript 的 WhatsApp Bridge。
- `systemd/`：部署到机器人设备时使用的用户级 systemd 服务文件。
- `scripts/setup/unitree_g1.sh`：面向 Unitree G1 的一键部署脚本，依赖较多，包含系统包、`uv`、Miniconda、CycloneDDS、抓取相关环境准备。

## 项目怎么运行

### Python 主项目

常用命令：

```bash
uv sync
uv sync --extra dev
uv run nanobot onboard
uv run nanobot agent
uv run nanobot gateway
```

如果走 `pyproject.toml` 的可编辑安装方式，也可使用：

```bash
uv pip install -e ".[dev]"
nanobot agent
nanobot gateway
```

### 机器人相关子项目

图像服务与避障服务是独立子项目，各自有自己的 `pyproject.toml`：

```bash
(cd robot/teleimager && uv sync --extra server)
(cd robot/obstacle_avoid && uv sync)
```

### 测试与格式化

```bash
pytest
ruff check nanobot
ruff format nanobot
```

仓库测试覆盖面很广，包含：

- `tests/agent/`：Agent 循环、记忆、技能、心跳、会话等。
- `tests/channels/`：多渠道接入。
- `tests/tools/`：文件、shell、web、MCP、sandbox 等工具层。
- `tests/providers/`：供应商适配。
- `tests/cli/`、`tests/config/`、`tests/cron/`、`tests/security/`：CLI、配置、定时任务、安全边界。

## 核心架构

### 1. Agent 主循环

`nanobot/agent/loop.py` 是系统核心。

主流程可以概括为：

1. 接收消息或直接请求。
2. 从上下文、会话、记忆、技能构造 prompt。
3. 调用 LLM。
4. 解析并执行工具调用。
5. 把结果写回渠道、会话和后续流程。

这里已经不是一个极简 loop，而是带有这些重要能力：

- `ToolRegistry` 动态注册默认工具。
- `SessionManager` 管理多会话历史。
- `SubagentManager` 支持子代理。
- `CronService` 支持定时任务。
- `HeartbeatService` 支持心跳跟进。
- `Consolidator` 和 `Dream` 负责记忆整理。
- 支持 MCP server 接入。
- 通过环境变量 `NANOBOT_MAX_CONCURRENT_REQUESTS` 控制并发请求数，默认值是 `3`。

### 2. 默认工具集合

主循环里默认注册的工具主要包括：

- 文件工具：读、写、编辑、列目录。
- 搜索工具：`glob`、`grep`。
- Shell 工具：可执行命令，是否启用和沙箱策略由配置控制。
- Web 工具：网页抓取、网页搜索。
- 消息工具。
- 子代理工具。
- Cron 工具。
- MCP 工具。
- 知识库相关工具。

这意味着很多“能力”并不写死在 Agent 里，而是由工具层提供。

### 3. 配置层

`nanobot/config/schema.py` 很关键。它定义了：

- `agents.defaults`：默认模型、工作区、最大 token、上下文窗口、时区、思维强度等。
- `providers`：大量模型供应商配置，包含 `openai`、`anthropic`、`openrouter`、`azure_openai`、`gemini`、`ollama`、`siliconflow`、`volcengine` 等。
- `gateway`：FastAPI 网关监听地址和端口，默认端口是 `18790`。
- `api`：OpenAI 兼容 API 的监听地址和端口，默认端口是 `8900`。
- `tools.exec` / `tools.web` / `tools.mcp_servers`：工具开关、超时、代理、MCP server 配置。
- `tools.restrict_to_workspace`：是否把工具访问限制在工作区内。

配置对象同时兼容 `camelCase` 和 `snake_case`。

## 网关与 API

### 1. `nanobot gateway`

CLI 中的 `gateway` 命令会启动 FastAPI 网关，并顺带拉起或接入这些组件：

- `MessageBus`
- LLM provider
- `SessionManager`
- `CronService`
- `AgentLoop`
- `ChannelManager`
- `HeartbeatService`

它是本项目最重要的运行入口之一。

### 2. Gateway 路由

`nanobot/gateway/routes/` 当前至少有这些模块：

- `chat.py`
- `controller.py`
- `ability.py`
- `skills.py`
- `home.py`

其中最值得先理解的是：

#### `chat.py`

面向 App 通道，提供文本、文件、语音、轮询回复、历史记录等接口。特点：

- 上传文件会先落盘到媒体目录。
- 语音会先转录，再进入 Agent 流程。
- 回复通过队列轮询拉取。
- 数据会落到 SQLite。

#### `controller.py`

这是机器人控制核心 HTTP 路由，前缀是 `/api/controller`。当前支持：

- 获取和设置速度。
- 旋转。
- 移动。
- 停止。
- 获取动作列表。
- 执行动作。
- 通过 `/external` 接收外部安全状态。

这里有一个非常重要的安全逻辑：

- 当前仅在“前进”动作时检查 `_safe`。
- 当避障服务报告不安全时，前进会被阻止，并返回 `code=2004`。

### 3. OpenAI 兼容 API

`nanobot/api/server.py` 提供 `/v1/chat/completions`、`/v1/models`、`/health`。

重要特点：

- 基于 `aiohttp`，不是 FastAPI。
- 目前只支持单条 user message。
- 不支持 `stream=true`。
- 可以通过 `session_id` 维持不同 API 会话。
- 所有请求最终仍然落到同一个 `AgentLoop.process_direct(...)`。

## 机器人执行层

### 1. Unitree G1 控制

`nanobot/gateway/unitree_g1.py` 是当前机器人动作控制的重要桥梁。它通过调用本地二进制程序来控制机器人，例如：

- `g1_loco_client`
- `g1_arm_action`
- `g1_audio_client`

代码里已经做了“无硬件环境可降级”的处理：

- 如果二进制不在 `PATH` 中，不会直接崩，而是记日志并跳过执行。
- 因此在开发机上可以启动 gateway，但很多硬件动作只是 no-op。

当前内置动作列表里已经有多种命名动作，例如：

- `shake_hand`
- `both_hands_up`
- `right_hand_up`
- `refuse`
- `box_both_hand_win`
- `blow_kiss_with_left_hand`
- `blow_kiss_with_right_hand`

### 2. 图像服务

`robot/teleimager/` 是独立子项目，来自 Unitree 生态，负责图像输入/传输。仓库里既有英文 README，也有中文 README。

### 3. 避障与安全状态

`robot/obstacle_avoid/safety_service_notify.py` 是一个 FastAPI 服务，用于安全检测，并在安全状态变化时通知控制器：

- 它会向 `GET /api/controller/external` 对应的控制器服务发通知。
- 目的是让网关知道当前是否允许继续前进。

### 4. 抓取与视觉相关服务

`nanobot/skills/robot-grasp/scripts/` 下有大量抓取和感知脚本，包含：

- `arm_ik_server.py`
- `yolo_detector_service.py`
- `grasp.py`
- `handover.py`
- `retract.py`

对应的 systemd 服务也已经给出：

- `systemd/arm-ik-server.service`
- `systemd/yolo-detector.service`

这说明“技能”在本仓库里不仅是 prompt，也可能是带脚本、带服务、带外部依赖的完整机器人能力包。

## 消息渠道与桥接

### 1. 多渠道

仓库支持的消息渠道很多，至少包括：

- Slack
- Telegram
- Discord
- Email
- WhatsApp
- 飞书
- 钉钉
- QQ
- 微信
- Matrix
- WeCom
- App

如果改渠道层，优先查看：

- `nanobot/channels/base.py`
- `nanobot/channels/manager.py`
- 对应渠道的实现文件

### 2. WhatsApp Bridge

`bridge/` 是单独的 Node.js/TypeScript 服务，使用：

- `@whiskeysockets/baileys`
- `ws`
- `typescript`

`bridge/src/server.ts` 里可以确认的安全边界：

- 仅绑定 `127.0.0.1`
- 要求 `BRIDGE_TOKEN`
- 拒绝浏览器 `Origin` 头的 WebSocket 连接

这部分是 Python 与 WhatsApp 客户端之间的本地桥接层，不是对外暴露的公网服务。

## 数据与持久化

### 1. 会话

`nanobot/session/manager.py` 管理会话历史。

### 2. Gateway 数据库

`nanobot/gateway/database.py` 使用 `SQLModel + SQLite`，数据库文件位于数据目录下的 `nanobot.db`。

### 3. 工作区

默认工作区来自配置，一般是 `~/.nanobot/workspace`。很多模板、记忆、cron 数据都会同步或落在这个工作区下，而不是仓库目录本身。

## 部署相关

### 1. systemd 服务

`systemd/` 目录下当前可见服务包括：

- `roboclaw.service`
- `teleimager.service`
- `obstacle-avoid.service`
- `arm-ik-server.service`
- `yolo-detector.service`

其中：

- `roboclaw.service` 会启动 `nanobot gateway`
- `teleimager.service` 会启动图像服务
- `obstacle-avoid.service` 会启动安全监测服务

### 2. Unitree G1 一键部署脚本

`scripts/setup/unitree_g1.sh` 会做很多重活，不只是安装 Python 依赖。它会处理：

- 系统包安装
- `uv` 安装
- 仓库拉取或更新
- `CycloneDDS` 编译安装
- `Miniconda` 安装
- `ik_env` conda 环境创建
- 抓取脚本环境准备
- systemd 用户服务准备

如果只是本地读代码或跑单测，不要轻易执行这个脚本。

## 开发时的判断原则

- 这个仓库虽然叫 RoboClaw，但骨架仍然是 `nanobot`，多数通用 Agent 行为仍在 `nanobot/agent`、`nanobot/tools`、`nanobot/providers` 中。
- 真正体现“机器人化”的部分主要集中在 `gateway`、机器人技能目录、`robot/` 子项目、`systemd/` 和部署脚本。
- 改聊天逻辑时，优先看 `agent loop -> channels -> gateway chat route` 之间的衔接。
- 改运动控制时，优先看 `gateway/routes/controller.py -> gateway/unitree_g1.py -> 本地机器人二进制`。
- 改安全逻辑时，要同时关注 `robot/obstacle_avoid` 和 `/api/controller/external` 的交互。
- 改抓取/视觉能力时，不要只看 `SKILL.md`，还要看同目录下的 `scripts/` 与 systemd 服务定义。
- 改配置行为时，优先看 `nanobot/config/schema.py`，不要只在 README 里找答案。
- 很多功能在没有真实机器人硬件时只能做静态检查、单测或接口级验证，不能假设本机具备完整执行环境。

## 对后续代理最有用的阅读顺序

如果你刚进入仓库，建议按这个顺序建立上下文：

1. `README.md`
2. `pyproject.toml`
3. `nanobot/cli/commands.py`
4. `nanobot/agent/loop.py`
5. `nanobot/config/schema.py`
6. `nanobot/gateway/routes/controller.py`
7. `nanobot/gateway/routes/chat.py`
8. `nanobot/gateway/unitree_g1.py`
9. `robot/obstacle_avoid/safety_service_notify.py`
10. 与当前任务相关的 `nanobot/skills/*` 或 `robot/*`

## 一句话总结

这是一个“以 `nanobot` 为内核、以 `gateway + robot skills + device services` 为外延”的机器人智能控制仓库；理解它时，既要把它当作 Agent 框架看，也要把它当作一个面向 Unitree G1 的机器人系统集成项目看。
