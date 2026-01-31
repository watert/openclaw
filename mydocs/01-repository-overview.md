# 一、仓库体积分析

## 1.1 总体概况

OpenClaw 项目总体积约为 65MB，对于一个多端 AI Agent 项目而言属于中等规模。以下是各主要目录的占用分布：

| 目录 | 大小 | 占比 | 主要构成 |
|------|------|------|----------|
| `src/` | 18MB | 27.7% | 核心业务代码 |
| `apps/` | 9.6MB | 14.8% | 多平台应用（macOS/iOS/Android） |
| `docs/` | 6.9MB | 10.6% | 文档及截图资源 |
| `extensions/` | 4.2MB | 6.5% | 浏览器扩展 |
| `vendor/` | 1.9MB | 2.9% | 第三方依赖 |
| `assets/` | 1.2MB | 1.8% | 静态资源 |
| `ui/` | 1.1MB | 1.7% | UI 组件库 |

## 1.2 核心代码结构

`src/` 目录是整个项目的核心，其内部结构如下：

| 子目录 | 大小 | 描述 |
|--------|------|------|
| `agents/` | 3.1MB | 最重量级模块，包含 AI Agent 逻辑 |
| `auto-reply/` | 1.7MB | 自动回复相关功能 |
| `commands/` | 1.6MB | 命令行指令实现 |
| `gateway/` | 1.5MB | 网关服务 |
| `infra/` | 1.2MB | 基础设施 |
| `cli/` | 1.2MB | 命令行工具 |
| `telegram/` | 748KB | Telegram 平台集成 |
| `browser/` | 692KB | 浏览器控制模块 |
| `channels/` | 576KB | 多通道支持 |
| `web/` | 564KB | Web 相关功能 |
| `discord/` | 428KB | Discord 集成 |
| `slack/` | 412KB | Slack 集成 |
| `plugins/` | 300KB | 插件系统 |
| `line/` | 296KB | LINE 平台集成 |

## 1.3 多平台应用分布

`apps/` 目录包含各平台的原生应用代码：

| 子目录 | 大小 | 说明 |
|--------|------|------|
| `macOS/` | 5.7MB | 包含应用图标等资源 |
| `iOS/` | 2.0MB | iOS 应用 |
| `Android/` | 1.3MB | Android 应用 |
| `shared/` | 544KB | 跨平台共享代码 |

## 1.4 大文件清单

| 文件 | 大小 | 类型 |
|------|------|------|
| `README-header.png` | - | 项目头图 |
| `docs/assets/showcase/oura-health.png` | - | 文档截图 |
| `.git/objects/pack/` | - | Git 对象包 |
| `apps/macos/Resources/OpenClaw.icns` | - | macOS 图标 |
| `apps/*/Icon*/Assets/*.png` | - | 各平台图标资源 |

## 1.5 分析结论

从体积分布来看，OpenClaw 的代码组织非常清晰：

**优点**：
- 依赖管理克制（`vendor/` 仅 1.9MB）
- 模块职责清晰，目录结构合理
- 核心代码集中在 `src/agents/` 和 `src/memory/`

**优化空间**：
- 文档截图占比较高（约 11%），部分截图可压缩
- 项目总体积 65MB 对于"轻量个人助手"略重

## 1.6 核心模块一览

| 模块 | 重要性 | 核心职责 |
|------|--------|----------|
| `agents/` | ⭐⭐⭐⭐⭐ | AI Agent 大脑 |
| `memory/` | ⭐⭐⭐⭐⭐ | 长期记忆系统 |
| `auto-reply/` | ⭐⭐⭐⭐⭐ | 消息处理引擎 |
| `gateway/` | ⭐⭐⭐⭐ | 网关控制平面 |
| `channels/` | ⭐⭐⭐⭐ | 多通道抽象 |
| `browser/` | ⭐⭐⭐⭐ | 浏览器控制 |
| `plugins/` | ⭐⭐⭐⭐ | 可扩展系统 |

详见 [02-memory-system.md](./02-memory-system.md) 和 [05-agent-system-prompt.md](./05-agent-system-prompt.md)。
