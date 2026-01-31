# 三、Memory 文件保存方式

OpenClaw 的 Memory 文件采用「明文 Markdown + 向量索引」的双层存储架构。

## 3.1 文件存储位置

Memory 文件保存在工作区目录下，遵循以下层次结构：

```
workspace/
├── MEMORY.md                    # 主记忆文件（必读）
├── memory.md                    # 备选主文件
└── memory/
    ├── 2026-01-03-project-kickoff.md
    ├── 2026-01-15-api-design.md
    ├── 2026-01-31-user-feedback.md
    └── ...                      # 按日期组织的记忆文件
```

## 3.2 用户手动保存

Memory 文件就是普通的 Markdown 文件，用户可以随时手动编辑：

- 直接编辑 `MEMORY.md` 或 `memory.md`
- 在 `memory/` 目录下创建新的 `.md` 文件
- 文件变化会被 `chokidar` 监听并自动触发重索引

## 3.3 自动保存机制（Memory Flush）

当对话上下文接近 Token 限制时，系统会自动触发 Memory Flush，让 AI 将重要信息持久化。

### 3.3.1 触发条件

```typescript
// 当 context tokens 接近 compaction 阈值时触发
shouldRunMemoryFlush({
  contextWindowTokens: 128000,      // 模型上下文大小
  reserveTokensFloor: 20000,        // 保留空间
  softThresholdTokens: 4000,        // 软阈值
  totalTokens: 110000,              // 当前已用 Token 数
})
// 触发阈值：totalTokens >= contextWindow - reserveTokens - softThreshold
```

### 3.3.2 System Prompt 指令

```typescript
const DEFAULT_MEMORY_FLUSH_PROMPT = [
  "Pre-compaction memory flush.",
  "Store durable memories now (use memory/YYYY-MM-DD.md; create memory/ if needed).",
  `If nothing to store, reply with ${SILENT_REPLY_TOKEN}.`,
].join(" ");
```

AI 收到此提示后，会将重要信息写入 `memory/YYYY-MM-DD.md` 文件。

### 3.3.3 配置项

```typescript
type MemoryFlushSettings = {
  enabled: boolean;              // 默认 true
  softThresholdTokens: number;   // 默认 4000
  prompt: string;                // 用户 prompt
  systemPrompt: string;          // 系统 prompt
  reserveTokensFloor: number;    // 保留空间下限
};
```

## 3.4 会话摘要保存（Session Memory Hook）

当用户执行 `/new` 命令时，系统会自动保存当前会话摘要。

### 3.4.1 触发逻辑

```typescript
// 只在 /new 命令触发时执行
if (event.type !== "command" || event.action !== "new") {
  return;
}

// 读取最近 15 条消息
const sessionContent = await getRecentSessionContent(sessionFile, 15);

// 使用 LLM 生成描述性 slug
const slug = await generateSlugViaLLM({ sessionContent, cfg });

// 写入新文件
const filename = `${dateStr}-${slug}.md`;  // 如：2026-01-31-team-sync.md
await fs.writeFile(memoryFilePath, entry, "utf-8");
```

### 3.4.2 生成文件格式

```markdown
# Session: 2026-01-31 14:30:00 UTC

- **Session Key**: xxx
- **Session ID**: xxx
- **Source**: telegram

## Conversation Summary

user: 我们需要设计新的 API
assistant: 好的，我来设计...
user: ...
```

### 3.4.3 配置项

```typescript
type SessionMemoryHookConfig = {
  messages?: number;  // 读取的消息数量，默认 15
};
```

## 3.5 索引机制

Memory 文件本身是明文 Markdown，但系统会维护两套索引：

| 索引类型 | 技术实现 | 用途 |
|----------|----------|------|
| **向量索引** | SQLite + sqlite-vec | 高效语义检索 |
| **全文索引** | SQLite FTS5 | 关键词精确匹配 |

### 3.5.1 索引更新时机

1. **文件变化监听**：`chokidar` 检测到 `MEMORY.md` / `memory/*.md` 变化
2. **会话增量更新**：按 `deltaBytes` / `deltaMessages` 阈值触发
3. **手动执行**：`openclaw memory index` 命令

### 3.5.2 索引更新流程

详见 [04-memory-chunk.md](./04-memory-chunk.md)。

## 3.6 存储架构总结

| 层级 | 存储方式 | 特点 |
|------|----------|------|
| **原始文件** | 明文 Markdown | 可读性好，可直接编辑 |
| **向量索引** | SQLite + sqlite-vec | 高效语义检索 |
| **全文索引** | SQLite FTS5 | 关键词精确匹配 |
| **Embedding 缓存** | SQLite | 避免重复计算 |

## 3.7 文件来源配置

Memory 文件来源可通过配置扩展：

```typescript
memorySearch: {
  sources: ["memory", "sessions"],  // memory 文件 + 会话转录
  extraPaths: [
    "/shared/notes",           // 绝对路径
    "../team-knowledge",       // 相对路径（相对于 workspace）
  ]
}
```

## 3.8 相关文档

- **Chunk 机制** → [04-memory-chunk.md](./04-memory-chunk.md)
- **System Prompt** → [05-agent-system-prompt.md](./05-agent-system-prompt.md)
