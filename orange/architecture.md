# OpenClaw 项目架构分析文档

## 1. 项目概述

### 1.1 项目定位

**OpenClaw** 是一个运行在用户自有设备上的**多通道 AI 网关**（Multi-channel AI Gateway），它是一个**本地优先（local-first）**的 AI 助手控制平面。

### 1.2 核心功能

- **多通道收件箱**（Multi-channel inbox）：支持 WhatsApp、Telegram、Slack、Discord、Signal、iMessage、Microsoft Teams、Matrix 等多个消息平台
- **多代理路由**（Multi-agent routing）：支持配置多个 AI 代理，每个代理可独立响应
- **语音唤醒 + 对话模式**（Voice Wake + Talk Mode）
- **实时 Canvas**（Live Canvas）
- **桌面/移动端配套应用**：macOS 菜单栏应用、iOS/Android 节点

### 1.3 运行时要求

- Node.js 22+
- pnpm 包管理器

---

## 2. 目录结构说明

```
/Users/orange/code/ai/openclaw/
├── src/                          # 核心源代码
│   ├── gateway/                  # 网关服务器实现
│   │   ├── server.impl.ts        # 网关服务器启动入口
│   │   ├── server-http.ts        # HTTP API
│   │   ├── server-ws-runtime.ts  # WebSocket 连接
│   │   └── ...
│   ├── agents/                   # AI 代理运行时
│   │   ├── pi-embedded-runner/   # Pi Agent 核心运行器
│   │   ├── subagent-registry.ts  # 子代理注册表
│   │   └── ...
│   ├── channels/                 # 通道基础设施（插件系统）
│   │   ├── plugins/              # 通道共享逻辑
│   │   ├── channel-manager.ts    # 通道管理器
│   │   └── ...
│   ├── cli/                      # CLI 程序构建
│   │   ├── program.ts            # Commander 程序
│   │   ├── run-main.ts           # 主运行逻辑
│   │   └── ...
│   ├── commands/                 # CLI 命令实现
│   ├── config/                   # 配置管理
│   │   ├── config.ts             # 配置加载
│   │   ├── loader.ts             # 配置加载器
│   │   └── validation.ts         # 配置验证
│   ├── infra/                    # 基础设施工具
│   ├── plugins/                  # 插件系统核心
│   ├── plugin-sdk/               # 插件 SDK（导出给扩展）
│   ├── providers/                # AI 供应商认证
│   ├── sessions/                 # 会话管理
│   ├── memory/                   # 记忆/向量存储
│   ├── telegram/                 # Telegram 通道实现
│   │   ├── bot.ts               # Bot 主类
│   │   ├── bot-handlers.ts      # 消息处理器
│   │   └── bot-message-dispatch.ts # 消息分发
│   ├── discord/                  # Discord 通道实现
│   ├── slack/                    # Slack 通道实现
│   ├── signal/                   # Signal 通道实现
│   ├── whatsapp/                 # WhatsApp 通道实现
│   ├── imessage/                 # iMessage 通道实现
│   ├── web/                      # Web 通道实现
│   └── ...
├── extensions/                   # 扩展插件（workspace packages）
│   ├── whatsapp/                 # WhatsApp 扩展
│   ├── telegram/                 # Telegram 扩展
│   ├── msteams/                  # Microsoft Teams
│   ├── matrix/                   # Matrix 协议
│   ├── zalo/                     # Zalo 越南通讯
│   ├── voice-call/               # 语音通话
│   └── ...
├── apps/                         # 桌面/移动应用
│   ├── macos/                    # macOS 菜单栏应用
│   ├── ios/                      # iOS 应用
│   ├── android/                  # Android 应用
│   └── shared/                   # 共享代码
├── ui/                           # Web 控制界面
├── docs/                         # 文档
├── skills/                       # 技能定义
├── scripts/                      # 构建/部署脚本
├── package.json                   # 项目配置
└── openclaw.mjs                  # npm bin 入口
```

---

## 3. 核心概念详解

### 3.1 Channel（通道）

通道是 OpenClaw 与外部消息平台集成的接口。每个通道都是一个独立的集成模块，负责：

- 接收来自平台的消息
- 将消息路由到内部处理流程
- 将响应发送回平台

**内置通道**（src/）：

| 通道 | 目录 | SDK/框架 |
|------|------|----------|
| Telegram | `src/telegram/` | `grammy` |
| Discord | `src/discord/` | `discord.js` |
| Slack | `src/slack/` | `@slack/bolt` |
| Signal | `src/signal/` | 自定义实现 |
| WhatsApp | `src/whatsapp/` | `@whiskeysockets/baileys` |
| iMessage | `src/imessage/` | BlueBubbles API |
| LINE | `src/line/` | `@line/bot-sdk` |
| Web | `src/web/` | WebSocket |

**扩展通道**（extensions/）：

- Microsoft Teams
- Matrix
- Zalo
- BlueBubbles
- Feishu
- IRC
- Nostr
- Twitch

### 3.2 Agent（代理）

代理是执行 AI 对话的实体，基于 **Pi Agent Core** 运行。每个代理：

- 有独立的会话存储
- 支持工具调用（tools）
- 支持子代理（subagents）
- 支持技能（skills）

**关键文件**：
- `src/agents/pi-embedded-runner/` - Pi Agent 核心运行器
- `src/agents/subagent-registry.ts` - 子代理注册表
- `src/agents/tool-registry.ts` - 工具注册表

### 3.3 Gateway（网关）

网关是整个系统的**控制平面**，管理所有通道、代理、会话和配置。

**主要职责**：
- 启动和管理 HTTP/WebSocket 服务器
- 加载和初始化通道
- 管理代理生命周期
- 处理配置加载和验证
- 提供 RESTful API 和 WebSocket API

**关键文件**：
- `src/gateway/server.impl.ts` - 网关服务器启动入口
- `src/gateway/server-http.ts` - HTTP API
- `src/gateway/server-ws-runtime.ts` - WebSocket 运行时

### 3.4 Provider（供应商）

AI 模型供应商，支持多种认证方式和模型。

**支持的供应商**：
- OpenAI (ChatGPT/Codex)
- Anthropic (Claude)
- Google Gemini
- GitHub Copilot
- Ollama (本地模型)
- Azure OpenAI
- 以及更多...

---

## 4. 入口点和主要流程

### 4.1 启动流程

```
openclaw.mjs (bin entry)
    ↓
src/entry.ts (CLI entry point)
    ↓
src/cli/program.ts (Commander program)
    ↓
src/cli/run-main.ts
    ↓
启动各种命令 (gateway, agent, message, etc.)
```

**关键入口文件**：

| 文件 | 职责 |
|------|------|
| `openclaw.mjs` | npm bin 入口 |
| `src/entry.ts` | CLI 主入口 |
| `src/cli/program.ts` | 命令行程序定义 |
| `src/gateway/server.impl.ts` | 网关服务器实现 |

### 4.2 网关启动流程

```
startGatewayServer()
    ├── 加载配置 (loadConfig)
    │   └── src/config/loader.ts
    ├── 初始化通道 (createChannelManager)
    │   └── src/channels/channel-manager.ts
    ├── 初始化代理 (initSubagentRegistry)
    │   └── src/agents/subagent-registry.ts
    ├── 加载插件 (loadGatewayPlugins)
    │   └── src/plugins/
    ├── 启动 HTTP/WebSocket 服务器
    │   ├── server-http.ts - HTTP API
    │   └── server-ws-runtime.ts - WebSocket 连接
    ├── 启动定时任务 (startGatewayMaintenanceTimers)
    ├── 启动通道健康监控 (startChannelHealthMonitor)
    └── 等待连接...
```

### 4.3 消息处理流程

```
外部消息 (Telegram/Discord/WhatsApp/etc.)
    ↓
通道 bot handlers (e.g., telegram/bot-handlers.ts)
    ↓
消息分发 (bot-message-dispatch.ts)
    ↓
检查 allowlist/DM 策略
    ↓
路由到对应代理 (resolve session key)
    ↓
代理处理 (agents/pi-embedded-runner/)
    ↓
生成 AI 响应
    ↓
发送响应回通道 (outbound handlers)
```

---

## 5. 架构设计模式

### 5.1 分层架构

OpenClaw 采用清晰的分层架构：

```
┌─────────────────────────────────────┐
│         CLI 层                      │  用户命令行界面
├─────────────────────────────────────┤
│         网关层                      │  核心控制平面（HTTP/WebSocket API）
├─────────────────────────────────────┤
│         通道层                      │  消息平台集成
├─────────────────────────────────────┤
│         代理层                      │  AI 对话运行时
├─────────────────────────────────────┤
│         供应商层                    │  AI 模型调用
└─────────────────────────────────────┘
```

### 5.2 双层插件系统

OpenClaw 采用**双层插件系统**：

1. **核心插件**（`src/channels/plugins/`）
   - 内置通道共享逻辑
   - 紧耦合到核心代码

2. **扩展插件**（`extensions/`）
   - 独立发布的通道扩展
   - 通过 `plugin-sdk` 提供统一接口

**插件机制**：
- 基于 `plugin-sdk`（`src/plugin-sdk/`）提供统一接口
- 支持自定义工具、钩子（hooks）、配置
- 扩展可发布到 npm 独立发布

### 5.3 配置驱动设计

- **配置文件格式**：YAML（`~/.openclaw/config.yaml`）
- **配置加载**：`src/config/loader.ts`
- **配置验证**：`src/config/validation.ts`
- **支持功能**：
  - 配置继承和合并
  - 热重载（`config-reload` 命令）
  - 多代理配置（agent directories）

### 5.4 状态管理

| 状态类型 | 存储方式 |
|---------|---------|
| 会话存储 | JSON 文件（`~/.openclaw/sessions/`） |
| 健康状态 | 内存中的健康快照 |
| WebSocket 连接 | 实时状态同步 |
| 配置 | YAML 文件 |

---

## 6. 关键技术栈

### 6.1 核心语言和运行时

- **TypeScript** (ESM 模块系统)
- **Node.js** 22+
- **pnpm** 包管理器

### 6.2 消息平台 SDK

| 通道 | SDK | 目录 |
|------|-----|------|
| Telegram | `grammy` | `src/telegram/` |
| Slack | `@slack/bolt` | `src/slack/` |
| Discord | `discord.js` | `src/discord/` |
| WhatsApp | `@whiskeysockets/baileys` | `src/whatsapp/` |
| Signal | 自定义实现 | `src/signal/` |
| iMessage | BlueBubbles API | `src/imessage/` |
| LINE | `@line/bot-sdk` | `src/line/` |

### 6.3 AI 运行时

- `@mariozechner/pi-agent-core` - Pi Agent 核心
- `@mariozechner/pi-ai` - AI 调用封装
- `@mariozechner/pi-coding-agent` - 编码代理

### 6.4 数据存储

- **SQLite**（via `better-sqlite3`）
- **sqlite-vec** - 向量搜索

### 6.5 测试和构建

- **Vitest** - 测试框架
- **tsdown** - TypeScript 打包
- **tsx** - TypeScript 执行

### 6.6 其他关键依赖

- `express` - HTTP 服务器
- `ws` - WebSocket
- `zod` - 数据验证
- `@sinclair/typebox` - TypeBox 类型定义
- `ajv` - JSON Schema 验证

---

## 7. 关键模块说明

| 模块路径 | 职责 |
|---------|------|
| `src/gateway/server.impl.ts` | 网关服务器启动入口 |
| `src/gateway/server-http.ts` | HTTP API 实现 |
| `src/gateway/server-ws-runtime.ts` | WebSocket 运行时 |
| `src/agents/` | AI 代理的运行核心 |
| `src/channels/channel-manager.ts` | 通道管理器 |
| `src/channels/plugins/` | 通道共享逻辑 |
| `src/cli/program.ts` | CLI 命令注册 |
| `src/config/` | 配置加载、验证、合并 |
| `src/plugins/` | 插件系统核心实现 |
| `src/plugin-sdk/` | 导出给扩展使用的 SDK |
| `src/infra/` | 基础设施工具 |
| `src/sessions/` | 会话存储和会话键解析 |
| `src/memory/` | 记忆/向量存储 |

---

## 8. 总结

OpenClaw 是一个高度模块化的多通道 AI 网关项目，采用了清晰的分层架构：

1. **控制平面（Gateway）** - 统一的 HTTP/WebSocket API
2. **消息路由** - 支持多通道的消息接收和发送
3. **AI 运行时** - 基于 Pi Agent Core 的对话系统
4. **插件系统** - 支持扩展新通道和功能
5. **配套应用** - macOS/iOS/Android 客户端

项目使用 TypeScript 开发，采用 ESM 模块系统，支持 Node.js 22+ 运行时，具有良好的可扩展性和可维护性。

---

*本文档由 AI 生成，基于对 OpenClaw 项目源代码的分析。*
