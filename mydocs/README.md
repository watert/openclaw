# OpenClaw 项目文档

本文档整理自对 OpenClaw 开源项目的代码分析。

## 文档结构

| 文档 | 内容 |
|------|------|
| [01-repository-overview.md](./01-repository-overview.md) | 仓库体积分析、目录结构 |
| [02-memory-system.md](./02-memory-system.md) | Memory 模块架构、Hybrid Search |
| [03-memory-files.md](./03-memory-files.md) | Memory 文件保存方式 |
| [04-memory-chunk.md](./04-memory-chunk.md) | Chunk 机制、更新逻辑 |
| [05-agent-system-prompt.md](./05-agent-system-prompt.md) | System Prompt 完整解析 |
| [06-plugin-vs-skills.md](./06-plugin-vs-skills.md) | Plugin 系统与 Skills 系统对比 |
| [07-commands.md](./07-commands.md) | Commands 模块、CLI 命令、内部运维 API |
| [08-hooks.md](./08-hooks.md) | Internal Hooks 与 Plugin Hooks 系统对比 |

## 快速索引

### 核心模块

- **Memory 系统** → `02-memory-system.md` + `03-memory-files.md` + `04-memory-chunk.md`
- **Agent** → `05-agent-system-prompt.md`
- **Plugin/Skills** → `06-plugin-vs-skills.md`
- **Commands** → `07-commands.md`
- **Hooks** → `08-hooks.md`
- **仓库概览** → `01-repository-overview.md`

### 关键特性

| 特性 | 文档位置 |
|------|----------|
| Hybrid Search (向量 + 关键词) | `02-memory-system.md` |
| 文件保存机制 | `03-memory-files.md` |
| Chunk 分块算法 | `04-memory-chunk.md` |
| 强制记忆召回 | `05-agent-system-prompt.md` |
| Plugin vs Skills 架构对比 | `06-plugin-vs-skills.md` |
| Commands 模块架构与内部调用 | `07-commands.md` |

## 相关链接

- [OpenClaw 官网](https://openclaw.ai)
- [官方文档](https://docs.openclaw.ai)
- [GitHub 仓库](https://github.com/openclaw/openclaw)

---

## TODOs

> 待补充的重要文档，记录未来值得深入分析的模块。

### 🔴 高优先级

- [x] **Memory Flush 机制** — 触发条件、Prompt 指令、格式来源 (`src/auto-reply/reply/memory-flush.ts`)：直接函数调用，非 Hook
- [x] **Plugin 系统** — 插件架构、manifest 定义、loader 流程、runtime API (`src/plugins/`, `src/plugin-sdk/`)
- [x] **Tools/Skills 系统** — Agent 工具调用机制、Skill 加载与执行流程 (`src/agents/tools/`, `src/skills/`)
- [x] **Hooks 系统** — Internal Hooks（事件驱动）与 Plugin Hooks（插件扩展）是两套独立系统，Memory Flush 与它们完全无关
- [ ] 自主性是如何实现的？ — 系统提示词中包含 `Allow autonomous actions` 指令， cron job 会触发内存 flush 操作。
- [ ] **Channel 模块** — 多平台通讯适配（Telegram/Slack/Discord/iMessage） (`src/channels/`, `src/channels/plugins/`)
- [ ] **Sandbox 沙箱** — 沙箱环境管理、文件系统隔离 (`src/sandbox/`)
- [ ] **Config 配置** — 配置文件解析、YAML/JSON 配置加载 (`src/config/`)
- [ ] **Message Queue** — 消息队列、异步处理 (`src/queue/`)
- [ ] **Auto-Reply 机制** — 自动回复触发条件、内存 flush 触发逻辑 (`src/auto-reply/`, `src/memory/flush.ts`)
