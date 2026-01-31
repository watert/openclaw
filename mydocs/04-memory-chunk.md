# 四、Memory Chunk 机制详解

本文档详细解析 Memory 模块的文本分块（Chunking）算法和索引更新逻辑。

## 4.1 Chunk 的依据

### 4.1.1 内容来源

Vector Memory 索引三类内容：

| 来源 | 路径规则 | 说明 |
|------|----------|------|
| **主记忆文件** | `MEMORY.md` / `memory.md` | 工作区根目录下的主记忆文件 |
| **记忆目录** | `memory/*.md` | `memory/` 子目录下的所有 Markdown |
| **扩展路径** | `extraPaths` 配置 | 用户自定义的其他目录 |
| **会话转录** | `sessions/*.jsonl` | 对话历史（可选） |

### 4.1.2 都是本地文件吗？

**是的，主要都是本地文件**：

```
workspace/
├── MEMORY.md           ← 本地 Markdown
├── memory/
│   ├── 2026-01-03-project-kickoff.md  ← 本地 Markdown
│   └── 2026-01-15-api-design.md
└── sessions/
    └── 2026-01-31-xxx.jsonl  ← 可选的会话转录（也是本地）
```

## 4.2 Chunk 算法详解

### 4.2.1 核心逻辑

Chunk 算法位于 [internal.ts:134-218](file:///Users/waterwu/www/github/openclaw/src/memory/internal.ts#L134-L218)：

```typescript
export function chunkMarkdown(
  content: string,
  chunking: { tokens: number; overlap: number },
): MemoryChunk[] {
  // 默认配置：tokens=400, overlap=80
  const maxChars = Math.max(32, chunking.tokens * 4);  // ~1600 chars
  const overlapChars = Math.max(0, chunking.overlap * 4);  // ~320 chars

  const lines = content.split("\n");
  const chunks: MemoryChunk[] = [];
  let current: Array<{ line: string; lineNo: number }> = [];
  let currentChars = 0;

  // 按行扫描，累积到 maxChars 时 flush 成一个 chunk
  for (let i = 0; i < lines.length; i++) {
    const line = lines[i] ?? "";
    const lineNo = i + 1;
    
    // 长行会拆分成多个 segment
    for (const segment of segments) {
      if (currentChars + segment.length > maxChars && current.length > 0) {
        flush();                    // 触发分块
        carryOverlap();             // 保留 overlap 部分
      }
      current.push({ line: segment, lineNo });
    }
  }
  flush();  // 最后一块

  return chunks;
}
```

### 4.2.2 Chunk 结构

```typescript
type MemoryChunk = {
  startLine: number;   // 起始行号
  endLine: number;     // 结束行号
  text: string;        // 块文本内容
  hash: string;        // SHA256 哈希
};
```

### 4.2.3 Overlap 机制

```
文件内容行号:  1    2    3    4    5    6    7    8    9
               ├──── Chunk 1 ────┤
                        ├──── Chunk 2 ────┤
                                 ├──── Chunk 3 ────┤
```

- Chunk 1: 行 1-5 (~400 tokens)
- Chunk 2: 行 3-7 (~320 chars overlap + ~330 chars 新内容)
- Chunk 3: 行 5-9 (~320 chars overlap + ...)

**Overlap 计算**：

```typescript
const carryOverlap = () => {
  if (overlapChars <= 0 || current.length === 0) {
    current = [];
    currentChars = 0;
    return;
  }
  
  let acc = 0;
  const kept: Array<{ line: string; lineNo: number }> = [];
  
  // 从当前块末尾向前保留 overlapChars 长度
  for (let i = current.length - 1; i >= 0; i--) {
    const entry = current[i];
    acc += entry.line.length + 1;  // +1 for newline
    kept.unshift(entry);
    if (acc >= overlapChars) break;
  }
  
  current = kept;
  currentChars = kept.reduce((sum, entry) => sum + entry.line.length + 1, 0);
};
```

### 4.2.4 分块示例

假设配置为 `tokens=400`（~1600 chars），`overlap=80`（~320 chars）：

```markdown
# Line 1: # Project Alpha
# Line 2: This is the first paragraph about the project goals.
# Line 3: We aim to build a scalable system.
# Line 4: The architecture is based on microservices.
# Line 5: Key components include API gateway, auth service.
# Line 6: Performance requirements are strict.
# Line 7: Response time must be under 100ms.
# Line 8: The team has 5 members.
# Line 9: Timeline: Q1 2026 launch.
```

**分块结果**：
- Chunk 1: 行 1-4 (~1200 chars)
- Chunk 2: 行 3-6 (~1100 chars，包含 overlap)
- Chunk 3: 行 5-7 (~900 chars，包含 overlap)
- Chunk 4: 行 7-9 (~600 chars，包含 overlap)

## 4.3 文件修改后的更新逻辑

### 4.3.1 监听机制

```typescript
private ensureWatcher() {
  this.watcher = chokidar.watch([
    path.join(this.workspaceDir, "MEMORY.md"),
    path.join(this.workspaceDir, "memory.md"),
    path.join(this.workspaceDir, "memory"),
    ...additionalPaths,
  ], {
    ignoreInitial: true,
    awaitWriteFinish: {
      stabilityThreshold: this.settings.sync.watchDebounceMs,  // 默认 1500ms
      pollInterval: 100,
    },
  });
  
  this.watcher.on("add", markDirty);    // 新增文件
  this.watcher.on("change", markDirty); // 文件修改
  this.watcher.on("unlink", markDirty); // 文件删除
}

private markDirty() {
  this.dirty = true;
  this.scheduleWatchSync();  // 延迟 5 秒后触发同步
}
```

### 4.3.2 Hash 对比机制

```typescript
const tasks = fileEntries.map((entry) => async () => {
  // 1. 读取数据库中的记录
  const record = this.db
    .prepare(`SELECT hash FROM files WHERE path = ? AND source = ?`)
    .get(entry.path, "memory") as { hash: string } | undefined;
  
  // 2. 对比文件 hash
  if (!params.needsFullReindex && record?.hash === entry.hash) {
    // Hash 没变，跳过索引
    return;
  }
  
  // 3. Hash 变了，重新索引
  await this.indexFile(entry, { source: "memory" });
});
```

**文件哈希计算**：

```typescript
export async function buildFileEntry(absPath: string, workspaceDir: string) {
  const content = await fs.readFile(absPath, "utf-8");
  const hash = hashText(content);  // SHA256 整个文件内容
  return {
    path: path.relative(workspaceDir, absPath),
    absPath,
    mtimeMs: stat.mtimeMs,
    size: stat.size,
    hash,
  };
}
```

### 4.3.3 完整索引流程

```typescript
private async indexFile(entry, options) {
  const content = await fs.readFile(entry.absPath, "utf-8");
  
  // Step 1: 按 Markdown 内容分块
  const chunks = chunkMarkdown(content, this.settings.chunking);
  
  // Step 2: 生成 Embedding
  const embeddings = this.batch.enabled
    ? await this.embedChunksWithBatch(chunks, entry, options.source)
    : await this.embedChunksInBatches(chunks);
  
  // Step 3: 删除旧索引
  this.db.prepare(`DELETE FROM ${VECTOR_TABLE} WHERE id IN ...`).run(...);
  this.db.prepare(`DELETE FROM ${FTS_TABLE} WHERE ...`).run(...);
  this.db.prepare(`DELETE FROM chunks WHERE ...`).run(...);
  
  // Step 4: 插入新 chunks + embeddings
  for (let i = 0; i < chunks.length; i++) {
    const chunk = chunks[i];
    const embedding = embeddings[i] ?? [];
    
    const id = hashText(
      `${options.source}:${entry.path}:${chunk.startLine}:${chunk.endLine}:${chunk.hash}:${this.provider.model}`
    );
    
    // 插入 chunks 表
    this.db.prepare(`INSERT INTO chunks ...`).run(...);
    
    // 插入向量表（如果可用）
    if (vectorReady && embedding.length > 0) {
      this.db.prepare(`INSERT INTO ${VECTOR_TABLE} ...`).run(...);
    }
    
    // 插入 FTS 表
    if (this.fts.enabled && this.fts.available) {
      this.db.prepare(`INSERT INTO ${FTS_TABLE} ...`).run(...);
    }
  }
  
  // Step 5: 更新 files 表
  this.db.prepare(`INSERT INTO files ...`).run(entry.path, entry.hash, ...);
}
```

### 4.3.4 垃圾清理

```typescript
// 删除已不存在的文件索引
const staleRows = this.db.prepare(`SELECT path FROM files WHERE source = ?`).all("memory");
for (const stale of staleRows) {
  if (activePaths.has(stale.path)) continue;  // 文件还存在，跳过
  
  // 删除相关索引
  this.db.prepare(`DELETE FROM files WHERE path = ?`).run(stale.path);
  this.db.prepare(`DELETE FROM ${VECTOR_TABLE} WHERE id IN ...`).run(...);
  this.db.prepare(`DELETE FROM chunks WHERE path = ?`).run(stale.path);
  this.db.prepare(`DELETE FROM ${FTS_TABLE} WHERE path = ?`).run(...);
}
```

## 4.4 更新流程图

```
┌─────────────────────────────────────────────────────────────────┐
│                        文件发生变化                               │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                    chokidar 监听 (debounce 1.5s)                 │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                   markDirty() → dirty = true                     │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│              scheduleWatchSync() → 5秒后触发 sync                  │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                        sync({ reason: "watch" })                 │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                   syncMemoryFiles()                              │
│                                                                 │
│  1. listMemoryFiles() → 扫描所有 .md 文件                        │
│  2. buildFileEntry() → 计算每个文件的 SHA256                     │
│  3. 对比 DB 中的 hash                                            │
│     ├── Hash 相同 → 跳过（快速路径）                              │
│     └── Hash 不同 → 重新索引                                     │
│                                                                 │
│  重新索引步骤:                                                   │
│  a) chunkMarkdown() → 按行分块 (~400 tokens, 80 overlap)        │
│  b) embedChunksWithBatch() → 调用 Embedding API 生成向量         │
│  c) 删除旧索引 (chunks + vector + fts)                          │
│  d) 插入新 chunks + embeddings                                   │
│  e) 更新 files 表                                                │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                   清理已删除文件的索引                             │
└─────────────────────────────────────────────────────────────────┘
```

## 4.5 关键设计点

| 特性 | 实现方式 | 优点 |
|------|----------|------|
| **变化检测** | 文件 SHA256 hash 对比 | 比 mtime 更准确，避免重复索引 |
| **增量更新** | 只重索引 hash 变化的文件 | 节省 API 调用和计算资源 |
| **Overlap** | 保留上个 chunk 的 ~320 chars | 防止信息跨块丢失 |
| **垃圾清理** | 删除不再存在的文件索引 | 保持数据库整洁 |
| **批量 Embedding** | 支持 OpenAI/Gemini Batch API | 降低成本，提高速度 |
| **Debounce** | 1.5s 文件稳定性等待 | 避免编辑过程中频繁触发 |

## 4.6 配置参数

| 参数 | 默认值 | 说明 |
|------|--------|------|
| `chunking.tokens` | 400 | 每个 chunk 的目标 token 数 |
| `chunking.overlap` | 80 | chunk 之间的重叠 token 数 |
| `sync.watch` | true | 是否启用文件监听 |
| `sync.watchDebounceMs` | 1500 | 文件写入稳定性等待时间 |
| `SESSION_DIRTY_DEBOUNCE_MS` | 5000 | 会话脏标记处理延迟 |

## 4.7 毒舌点评

**优点**：
1. **Hash 对比比 mtime 可靠** —— 有些场景下文件 mtime 不会变但内容变了，SHA256 是真的保险
2. **Overlap 机制合理** —— 320 chars 大约 50-80 个词，防止一个完整的意思被切开
3. **批量 Embedding 支持** —— 用 Batch API 可以显著降低成本

**槽点**：
1. **双重延迟** —— 1.5s + 5s = 6.5 秒的感知延迟略长
2. **全量 chunk 更新** —— 当前是全量删除+重新插入，如果文件只有一行变了，整个文件的所有 chunks 都要重新生成 embedding。这在文件很大时有点浪费，但实现简单可靠 👍

## 4.8 相关文档

- **Memory 系统架构** → [02-memory-system.md](./02-memory-system.md)
- **文件保存方式** → [03-memory-files.md](./03-memory-files.md)
