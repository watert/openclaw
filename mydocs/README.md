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

## 快速索引

### 核心模块

- **Memory 系统** → `02-memory-system.md` + `03-memory-files.md` + `04-memory-chunk.md`
- **Agent** → `05-agent-system-prompt.md`
- **仓库概览** → `01-repository-overview.md`

### 关键特性

| 特性 | 文档位置 |
|------|----------|
| Hybrid Search (向量 + 关键词) | `02-memory-system.md` |
| 文件保存机制 | `03-memory-files.md` |
| Chunk 分块算法 | `04-memory-chunk.md` |
| 强制记忆召回 | `05-agent-system-prompt.md` |

## 相关链接

- [OpenClaw 官网](https://openclaw.ai)
- [官方文档](https://docs.openclaw.ai)
- [GitHub 仓库](https://github.com/openclaw/openclaw)

---

## TODOs

> 待补充的重要文档，记录未来值得深入分析的模块。

### 🔴 高优先级

- [ ] **Plugin 系统** — 插件架构、manifest 定义、loader 流程、runtime API (`src/plugins/`, `src/plugin-sdk/`)
- [ ] **Tools/Skills 系统** — Agent 工具调用机制、Skill 加载与执行流程 (`src/agents/tools/`, `src/skills/`)
- [ ] **Channel 模块** — 多平台通讯适配（Telegram/Slack/Discord/iMessage） (`src/channels/`, `src/channels/plugins/`)

### 🟡 中优先级

- [ ] **Auto-Reply 机制** — 自动回复触发条件、内存 flush 触发逻辑 (`src/auto-reply/`, `src/memory/flush.ts`)
- [ ] **Commands 模块** — 命令解析、斜杠命令执行 (`src/commands/`)
- [ ] **Hooks 系统** — 钩子注册、事件触发机制 (`src/hooks/`, `src/hooks/bundled/`)

### 🟢 低优先级

- [ ] **Config 配置** — 配置文件解析、YAML/JSON 配置加载 (`src/config/`)
- [ ] **Sandbox 沙箱** — 沙箱环境管理、文件系统隔离 (`src/sandbox/`)
- [ ] **Message Queue** — 消息队列、异步处理 (`src/queue/`)
