# 二、Memory 模块深度分析

Memory 模块是 OpenClaw 的核心功能之一，提供语义化的长期记忆存储与检索能力。

## 2.1 整体架构

Memory 模块位于 `src/memory/` 目录下，核心文件包括：

| 文件 | 功能描述 |
|------|----------|
| `manager.ts` | 核心管理器 MemoryIndexManager（约 900+ 行） |
| `search-manager.ts` | 工厂模式与单例缓存管理 |
| `embeddings.ts` | Embedding Provider 抽象层 |
| `embeddings-openai.ts` | OpenAI 实现 |
| `embeddings-gemini.ts` | Gemini 实现 |
| `manager-search.ts` | 向量搜索实现 |
| `hybrid.ts` | 混合搜索（向量 + 关键词） |
| `internal.ts` | 核心工具函数 |
| `session-files.ts` | 会话文件处理 |
| `sqlite-vec.ts` | sqlite-vec 扩展加载 |
| `memory-schema.ts` | 数据库 Schema 定义 |

## 2.2 混合搜索架构

Memory 模块采用**向量搜索与关键词搜索相结合**的混合检索策略：

```typescript
vectorWeight（默认 0.7）+ textWeight（默认 0.3）
```

这种设计充分利用了两种搜索方式的各自优势：
- **向量搜索**：善于捕捉语义相似性
- **关键词搜索**：确保精确匹配的召回率

**Hybrid Search 实现**（[`src/memory/hybrid.ts`](./src/memory/hybrid.ts)）：

```typescript
export function mergeHybridResults(params: {
  vector: HybridVectorResult[];
  keyword: HybridKeywordResult[];
  vectorWeight: number;
  textWeight: number;
})
```

## 2.3 Embedding Provider 抽象层

系统定义了统一的 Embedding Provider 接口：

```typescript
type EmbeddingProvider = {
  id: string;
  model: string;
  embedQuery: (text: string) => Promise<number[]>;
  embedBatch: (texts: string[]) => Promise<number[][]>;
};
```

### 2.3.1 支持的后端

| 提供商 | 默认模型 | 特点 |
|--------|----------|------|
| OpenAI | `text-embedding-3-small` | 云服务，精度高 |
| Gemini | `gemini-embedding-001` | Google 云服务 |
| Local | `embeddinggemma-300M-Q8_0.gguf` | 本地运行（node-llama-cpp） |

### 2.3.2 自动降级机制

当主 Provider（如 API Key 失效）时会自动切换到备用方案：

```typescript
async function createEmbeddingProvider(options: EmbeddingProviderOptions)
```

## 2.4 向量数据库选型

OpenClaw 采用 **SQLite + sqlite-vec** 组合作为向量存储方案：

- 使用 `vec_distance_cosine` 计算余弦相似度
- 单文件存储，零配置部署
- 比 Chroma、Pinecone 等云服务轻量得多

### 2.4.1 核心表结构

```sql
CREATE TABLE files (path TEXT, source TEXT, hash TEXT, mtime_ms REAL, size REAL);
CREATE TABLE chunks (id TEXT PRIMARY KEY, path TEXT, source TEXT, start_line INTEGER, end_line INTEGER, text TEXT, model TEXT, hash TEXT, embedding TEXT, updated_at INTEGER);
CREATE VIRTUAL TABLE chunks_vec USING vec0(embedding FLOAT[768]);
CREATE VIRTUAL TABLE chunks_fts USING fts5(path, text, tokenize='porter');
CREATE TABLE embedding_cache (hash TEXT PRIMARY KEY, embedding BLOB);
```

### 2.4.2 向量查询示例

```sql
SELECT c.id, c.path, c.start_line, c.end_line,
       vec_distance_cosine(v.embedding, ?) AS dist
  FROM chunks_vec v
  JOIN chunks c ON c.id = v.id
 WHERE c.model = ?
 ORDER BY dist ASC
 LIMIT ?;
```

## 2.5 智能文件监听与增量同步

Memory 模块实现了智能的文件变化监听机制：

```typescript
private watcher: FSWatcher | null = null;
private dirty = false;
private sessionsDirty = false;
```

### 2.5.1 监听配置

```typescript
private ensureWatcher()
```

### 2.5.2 会话增量同步

- 使用 `chokidar` 监听文件变化
- 采用 5 秒 debounce 策略，避免频繁触发重索引
- 支持增量同步，会话文件只同步新增内容

详见 [04-memory-chunk.md](./04-memory-chunk.md) 中的更新逻辑。

## 2.6 Embedding 缓存机制

系统内置 LRU 缓存，避免重复调用 Embedding API：

```typescript
private readonly cache: { enabled: boolean; maxEntries?: number };
```

## 2.7 单例缓存管理

MemoryIndexManager 使用单例模式，配合全局缓存避免重复创建：

```typescript
const INDEX_CACHE = new Map<string, MemoryIndexManager>();
```

## 2.8 可配置参数

| 配置项 | 默认值 | 用途说明 |
|--------|--------|----------|
| `chunking.tokens` | 400 | 文本分块大小（Token 数） |
| `chunking.overlap` | 80 | 分块重叠量 |
| `query.maxResults` | 6 | 单次搜索返回结果数 |
| `query.minScore` | 0.35 | 相似度阈值 |
| `cache.maxEntries` | - | LRU 缓存大小限制 |
| `sync.sessions.deltaBytes` | 100KB | 会话增量同步阈值 |
| `sync.sessions.deltaMessages` | 50 | 会话增量消息数阈值 |

## 2.9 核心使用场景

Memory 模块主要通过 Agent Tool 暴露给 AI：

```typescript
"Mandatory recall step: semantically search MEMORY.md + memory/*.md 
(and optional session transcripts) before answering questions about 
prior work, decisions, dates, people, preferences, or todos; 
returns top snippets with path + lines."
```

## 2.10 Memory Flush 机制

Memory Flush 是 OpenClaw 的上下文压缩与记忆持久化机制。当对话上下文接近模型上下文窗口上限时，系统会自动触发一次特殊的 AI 调用周期：AI 评估当前对话中是否存在「值得持久化的记忆」，若有则写入 `<workspace>/memory/YYYY-MM-DD.md` 文件；若判断无重要信息，则返回 `__SILENT_REPLY__` Token 以避免产生无意义文本。

这一机制的核心价值在于：在有限的上下文窗口内，优先保留「对长期决策有价值的信息」，而非被短期对话细节淹没。Memory Flush 不同于主动写入 MEMORY.md 的显式操作，它是一种被动防御机制——当上下文即将溢出时，系统主动「抢救」高价值记忆。

### 2.10.1 触发条件与决策逻辑

Memory Flush 并非每次对话都会执行，而是基于严格的上下文使用量评估。决策入口位于 [`shouldRunMemoryFlush()`](src/auto-reply/reply/memory-flush.ts)，该函数综合考量三个维度：当前上下文已消耗 Token 数、模型上下文窗口上限、以及用户配置的水位线阈值。

```typescript
type ShouldRunMemoryFlushParams = {
  entry: SessionEntry | undefined;
  contextWindowTokens: number;
  reserveTokensFloor: number;
  softThresholdTokens: number;
};
function shouldRunMemoryFlush(params: ShouldRunMemoryFlushParams): boolean
```

**决策逻辑**：首先，系统从 `entry.tokens` 获取当前会话已消耗的 Token 数，将其与模型上下文窗口相减得到已用 Token；然后计算「可用 Token = 上下文窗口 - 已用 Token - 保留空间」；最后进行两层检查：硬阈值（可用 Token 为负，即已经溢出）直接触发，软阈值（可用 Token 低于用户配置的 `softThresholdTokens`）则在启用渐进压缩时触发。

**配置解析**（[`resolveMemoryFlushSettings()`](src/auto-reply/reply/memory-flush.ts)）：

```typescript
type MemoryFlushSettings = {
  prompt: string;
  systemPrompt: string;
  reserveTokensFloor: number;
  softThresholdTokens: number;
};
function resolveMemoryFlushSettings(cfg: OpenClawConfig): MemoryFlushSettings | null
```

默认配置下 `softThresholdTokens` 为 0，意味着渐进压缩默认关闭，系统只在上下文真正溢出时才触发 Memory Flush。这是一种保守策略——避免过早触发导致频繁的压缩中断用户体验。

### 2.10.2 Prompt 指令与 AI 行为引导

当 Memory Flush 被触发后，系统会向 AI 发送特殊的 Prompt 指令，引导其完成记忆抢救任务。指令定义在 [`memory-flush.ts`](src/auto-reply/reply/memory-flush.ts)：

```typescript
const DEFAULT_MEMORY_FLUSH_PROMPT = [
  "Pre-compaction memory flush.",
  "Store durable memories now (use memory/YYYY-MM-DD.md; create memory/ if needed).",
  `If nothing to store, reply with ${SILENT_REPLY_TOKEN}.`,
].join(" ");

const DEFAULT_MEMORY_FLUSH_SYSTEM_PROMPT = [
  "Pre-compaction memory flush turn.",
  "The session is near auto-compaction; capture durable memories to disk.",
  `You may reply, but usually ${SILENT_REPLY_TOKEN} is correct.`,
].join(" ");
```

User Prompt 明确告知 AI 三个关键信息：当前是「压缩前的记忆 flush 阶段」，应该「将持久化记忆存储到 memory/YYYY-MM-DD.md 文件」，如果「没有需要存储的内容」则回复 `__SILENT_REPLY__`。System Prompt 则从系统层面强调这是「压缩前的特殊轮次」，会话即将自动压缩，AI「可以回复，但通常 `__SILENT_REPLY__` 才是正确选择」。

这里的设计精髓在于「双层确认」机制。User Prompt 和 System Prompt 各自强调了一次静默回复的正确性，确保 AI 不会在无内容可写时强行生成无意义文本。

### 2.10.3 文件名约定与格式来源

OpenClaw 对 Memory 文件的命名有明确约定：`memory/YYYY-MM-DD.md`。这一约定直接编码在 `DEFAULT_MEMORY_FLUSH_PROMPT` 中，AI 被告知「use memory/YYYY-MM-DD.md」。文件名使用 ISO 日期格式（YYYY-MM-DD），不含时间戳，暗示同一日期的多条记忆应当合并写入同一文件——这是符合直觉的设计，因为日期相近的记忆往往具有关联性。

**关键发现**：OpenClaw 源码中**并未显式定义** Memory 文件的内容格式。AI 使用 `createWriteTool`（来自 `@mariozechner/pi-coding-agent` 包）执行写入操作，但该工具的描述来自外部依赖，OpenClaw 层面没有额外的格式约束。

格式风格主要来源于两个方面：其一，AI 模型的预训练知识使其天然具备「摘要写作」的能力；其二，Session Memory Hook（见 [03-memory-files.md](./03-memory-files.md)）提供了另一种记忆持久化范式，其输出格式为「标题 + 元数据 + 对话摘要」的三段式结构。OpenClaw 对 Memory 文件内容秉持「简约摘要风格」的定位——Memory 文件应当是「轻量级、易检索的摘要」，而非完整对话转录。

### 2.10.4 执行流程与状态持久化

Memory Flush 的完整执行流程由 [`runMemoryFlushIfNeeded()`](src/auto-reply/reply/agent-runner-memory.ts) 编排。该函数首先检查是否满足触发条件，若满足则执行一次特殊的 AI 调用周期，最后将执行结果持久化到 Session Store。

```typescript
type RunMemoryFlushIfNeededParams = {
  cfg: OpenClawConfig;
  followupRun: FollowupRun;
  sessionCtx: TemplateContext;
  // ...
};
async function runMemoryFlushIfNeeded(params: RunMemoryFlushIfNeededParams): Promise<SessionEntry | undefined>
```

**关键设计决策**：

- **CLI Provider 跳过**：通过命令行直接交互的会话不触发 Memory Flush——CLI 场景通常短平快，压缩价值有限。
- **System Prompt 拼接**：用户原本的 System Prompt 并追加 flush 相关指令，确保 AI 在 flush 轮次仍能遵守用户的全局行为规范。
- **状态持久化**：系统更新 Session Store 中的 `memoryFlushAt`（时间戳）和 `memoryFlushCompactionCount`（压缩计数）字段。这些元数据可用于审计和统计。

### 2.10.5 静默回复 Token 的设计意义

`SILENT_REPLY_TOKEN`（`__SILENT_REPLY__`）是 OpenClaw 对「无操作」进行建模的巧妙设计。在传统对话系统中，AI 必须生成文本响应任何输入；而在 Memory Flush 场景下，「无需响应」恰恰是正确的行为——没有值得持久化的记忆时，说什么都是噪音。

```typescript
const DEFAULT_SILENT_REPLY_PROMPT = [
  `If you have nothing to say, reply with exactly "${SILENT_REPLY_TOKEN}" and nothing else.`,
  "This is useful when memory flush finds nothing to store, or when no response is needed.",
].join("\n");
```

该指令被嵌入默认的 Foundation Prompt 中，意味着 AI 在任何轮次都可以使用静默回复——不仅限于 Memory Flush。这为系统层面的「降噪」提供了统一的机制基础。

### 2.10.6 与 Session Memory Hook 的协同

OpenClaw 实际上存在**三种记忆持久化路径**：

| 路径 | 触发时机 | 文件格式 | 典型内容 |
|------|----------|----------|----------|
| 主动写入 | 用户显式请求 | 自由格式 | 重要决策、约定、偏好 |
| Memory Flush | 上下文溢出 | 简约摘要 | 值得长期保留的关键信息 |
| Session Hook | `/new` 命令 | 标题+元数据+摘要 | 会话级别的历史记录 |

Memory Flush 与 Session Memory Hook 在文件命名上形成互补：Flush 使用 `YYYY-MM-DD.md`（按日期聚合），Hook 使用 `YYYY-MM-DD-slug.md`（按会话描述）。前者适合积累同一天的零散记忆，后者适合保留完整会话的历史切片。

### 2.10.7 风险与局限性

Memory Flush 机制虽然优雅，但存在几个潜在风险值得注意。

**格式不稳定性**：由于 OpenClaw 未显式定义 Memory 文件的内容格式，AI 生成的摘要风格可能因模型版本、temperature 设置等因素而波动。同一用户在模型升级后可能发现新写入的记忆与旧记忆格式迥异，影响可读性和检索一致性。

**触发时机滞后**：默认配置下，系统只在上下文「真正溢出」时才触发 Flush，这意味着在触发前的最后几个轮次中，AI 可能因上下文紧张而表现欠佳——思考受限、记忆召回不全、输出质量下降。渐进压缩（`softThresholdTokens`）可以缓解这一问题，但需要用户主动配置。

**静默回复滥用**：如果 AI 对「什么值得持久化」的理解过于保守，可能导致 Flush 轮次频繁返回 `__SILENT_REPLY__`，使该机制形同虚设。这一问题依赖于模型能力，无法在系统层面根治。

## 2.11 设计亮点总结

| 特性 | 实现方式 | 优点 |
|------|----------|------|
| 混合搜索架构 | 向量 + 关键词融合 | 兼顾语义与精确匹配 |
| Provider 抽象层 | 三种后端 + 自动降级 | 易于扩展 |
| SQLite 轻量方案 | 单文件存储 | 运维成本低 |
| 智能增量同步 | 按 deltaBytes/deltaMessages | 避免全量重索引 |
| 强制记忆召回 | System Prompt 强制规则 | 确保可靠性 |
| Memory Flush | 上下文溢出抢救机制 | 保护高价值记忆 |

详见 [03-memory-files.md](./03-memory-files.md) 和 [04-memory-chunk.md](./04-memory-chunk.md)。
