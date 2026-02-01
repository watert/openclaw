# 一、项目概览

## 1.1 项目意图

OpenClaw 是一个**个人 AI 助手**项目，设计理念是在用户自有设备上运行的本地化 AI 助手。它回答问题的渠道涵盖用户已经使用的即时通讯平台（WhatsApp、Telegram、Slack、Discord、Google Chat、Signal、iMessage、Microsoft Teams、WebChat），还支持扩展渠道如 BlueBubbles、Matrix、Zalo 等。

核心特性包括：
- **多端支持**：可在 macOS/iOS/Android 上运行，支持语音输入输出
- **多渠道统一**：一个后端对接多个聊天平台
- **本地化运行**：数据存储在用户设备上，强调隐私和速度
- **可扩展性**：支持插件系统（Plugin）和技能系统（Skills）
- **Canvas 控制**：支持实时 Canvas 渲染和控制

推荐配置：**Anthropic Pro/Max (100/200) + Opus 4.5**，长上下文能力强，抗提示注入表现好。

## 1.2 目录结构

```
openclaw/
├── src/                    # 核心业务代码
│   ├── agents/            # AI Agent 逻辑（最核心）
│   ├── auto-reply/        # 消息自动回复引擎
│   ├── memory/            # 长期记忆系统
│   ├── commands/          # CLI 命令实现
│   ├── gateway/           # 网关控制平面
│   ├── channels/          # 多通道抽象层
│   ├── plugins/           # 插件系统
│   ├── hooks/             # 内部事件钩子
│   ├── browser/           # 浏览器控制
│   ├── infra/             # 基础设施
│   └── ...
├── apps/                  # 多平台原生应用
│   ├── macOS/            # macOS 应用
│   ├── iOS/              # iOS 应用
│   ├── Android/          # Android 应用
│   └── shared/           # 跨平台共享代码
├── skills/                # 技能定义文件
├── extensions/            # 浏览器扩展
├── docs/                  # 项目文档
├── mydocs/                # 代码分析文档（本仓库）
├── ui/                    # UI 组件库
├── assets/                # 静态资源
└── openclaw.mjs           # CLI 入口
```

## 1.3 核心架构

OpenClaw 采用**分层架构**，核心组件关系如下：

```
┌─────────────────────────────────────────────────────────────┐
│                      CLI / TUI                              │
├─────────────────────────────────────────────────────────────┤
│                      Gateway                                │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────┐ │
│  │ Auto-Reply  │  │  Commands   │  │  Channel Manager    │ │
│  │   Engine    │  │   Module    │  │                     │ │
│  └─────────────┘  └─────────────┘  └─────────────────────┘ │
├─────────────────────────────────────────────────────────────┤
│                      Agent Core                             │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────┐ │
│  │ Memory      │  │   Skills    │  │   Plugin System     │ │
│  │   System    │  │   Engine    │  │                     │ │
│  └─────────────┘  └─────────────┘  └─────────────────────┘ │
├─────────────────────────────────────────────────────────────┤
│                    Model Provider                           │
│        Anthropic / OpenAI / Gemini / Local                  │
└─────────────────────────────────────────────────────────────┘
```

**核心子系统**：
- **Agent Core**：AI 大脑，负责理解意图、决策、执行
- **Memory System**：长期记忆存储与检索（向量 + 关键词混合搜索）
- **Auto-Reply Engine**：消息处理、上下文管理、Memory Flush
- **Channel Manager**：多平台消息收发抽象
- **Plugin System**：插件扩展机制
- **Skills System**：LLM 技能教导框架

## 1.4 核心依赖库

| 分类 | 库名 | 版本 | 用途 |
|------|------|------|------|
| **Agent 核心** | `@mariozechner/pi-*` | 0.50.7 | Agent 运行时、工具调用、TUI |
| **消息平台** | `@whiskeysockets/baileys` | 7.0.0-rc.9 | WhatsApp 协议 |
| | `grammy` | 1.39.3 | Telegram 机器人框架 |
| | `@slack/bolt` | 4.6.0 | Slack 应用框架 |
| | `@line/bot-sdk` | 10.6.0 | LINE 机器人 SDK |
| **Web 框架** | `express` | 5.2.1 | HTTP 服务器 |
| | `hono` | 4.11.4 | 轻量 Web 框架 |
| **数据库** | `sqlite-vec` | 0.1.7-alpha.2 | SQLite 向量搜索扩展 |
| **配置** | `zod` | ^4.3.6 | 类型安全配置 |
| | `yaml` | ^2.8.2 | YAML 解析 |
| **工具** | `chokidar` | ^5.0.0 | 文件监听 |
| | `playwright-core` | 1.58.0 | 浏览器控制 |
| | `ws` | ^8.19.0 | WebSocket |

**推荐运行时**：Node.js ≥ 22.12.0

## 1.5 核心逻辑流程

### 消息处理流程

```
用户消息 ──► Channel Adapter ──► Gateway ──► Auto-Reply Engine
                                              │
                      ┌───────────────────────┼───────────────────────┐
                      ▼                       ▼                       ▼
              Memory Search          Agent Decision           Command Parse
              (向量+关键词)           (LLM 调用)              (/命令)
                      │                       │                       │
                      └───────────────────────┴───────────────────────┘
                                              │
                                              ▼
                                      响应生成 ──► Memory Flush（如需要）
                                              │
                                              ▼
                                      Channel Adapter ──► 用户
```

### Memory Flush 触发逻辑

```
检查条件：上下文 Token 接近限制
         │
         ▼
┌─────────────────────────────────────────────────────────────┐
│  shouldRunMemoryFlush()                                     │
│  - 硬阈值：可用 Token < 0（已溢出）                          │
│  - 软阈值：可用 Token < softThresholdTokens（渐进压缩）      │
└─────────────────────────────────────────────────────────────┘
         │
         ▼
  触发 Memory Flush 轮次
         │
         ▼
  AI 收到 Prompt："Store durable memories now"
         │
         ├──► 有内容 ──► 写入 memory/YYYY-MM-DD.md
         │
         └──► 无内容 ──► 回复 __SILENT_REPLY__
```

## 1.6 核心代码位置

| 功能 | 路径 | 说明 |
|------|------|------|
| **Agent 核心** | `src/agents/` | Agent 逻辑、工具定义、Pi 集成 |
| **Memory 系统** | `src/memory/` | 向量存储、混合搜索、索引管理 |
| **自动回复** | `src/auto-reply/` | 消息处理、Memory Flush、Session 管理 |
| **插件系统** | `src/plugins/` | Plugin manifest、loader、runtime API |
| **Skills** | `src/skills/` | Skill 加载、过滤、执行 |
| **Hooks** | `src/hooks/` | Internal Hooks、Bundled Hooks |
| **通道适配** | `src/channels/` | 多平台抽象、Telegram/Slack/Discord 等 |
| **浏览器控制** | `src/browser/` | Playwright 集成、页面操作 |
| **配置** | `src/config/` | Zod Schema、配置解析 |
| **CLI** | `src/cli/` | 命令行入口、命令注册 |

## 1.7 体积分析

### 总体概况

OpenClaw 项目总体积约为 65MB，对于一个多端 AI Agent 项目而言属于中等规模。

| 目录 | 大小 | 占比 | 主要构成 |
|------|------|------|----------|
| `src/` | 18MB | 27.7% | 核心业务代码 |
| `apps/` | 9.6MB | 14.8% | 多平台应用 |
| `docs/` | 6.9MB | 10.6% | 文档及截图资源 |
| `extensions/` | 4.2MB | 6.5% | 浏览器扩展 |
| `vendor/` | 1.9MB | 2.9% | 第三方依赖 |
| `assets/` | 1.2MB | 1.8% | 静态资源 |
| `ui/` | 1.1MB | 1.7% | UI 组件库 |

### 核心代码结构（src/）

| 子目录 | 大小 | 描述 |
|--------|------|------|
| `agents/` | 3.1MB | 最重量级模块，AI Agent 逻辑 |
| `auto-reply/` | 1.7MB | 自动回复相关功能 |
| `commands/` | 1.6MB | 命令行指令实现 |
| `gateway/` | 1.5MB | 网关服务 |
| `infra/` | 1.2MB | 基础设施 |
| `cli/` | 1.2MB | 命令行工具 |

### 观察与结论

**优点**：
- 依赖管理克制（`vendor/` 仅 1.9MB）
- 模块职责清晰，目录结构合理
- 核心代码集中在 `agents/` 和 `memory/`

**优化空间**：
- 文档截图占比较高（约 11%）
- 项目总体积 65MB 对于「轻量个人助手」略重

## 1.8 文档索引

| 文档 | 内容 |
|------|------|
| [02-memory-system.md](./02-memory-system.md) | Memory 模块架构、Hybrid Search、Memory Flush |
| [03-memory-files.md](./03-memory-files.md) | Memory 文件保存方式 |
| [04-memory-chunk.md](./04-memory-chunk.md) | Chunk 机制、更新逻辑 |
| [05-agent-system-prompt.md](./05-agent-system-prompt.md) | System Prompt 完整解析 |
| [06-plugin-vs-skills.md](./06-plugin-vs-skills.md) | Plugin 系统与 Skills 系统对比 |
| [07-commands.md](./07-commands.md) | Commands 模块、CLI 命令、内部运维 API |
| [08-hooks.md](./08-hooks.md) | Internal Hooks 与 Plugin Hooks 系统对比 |
