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
