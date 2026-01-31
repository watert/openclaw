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
// 向量搜索（语义相似度）+ 关键词搜索（BM25）加权融合
vectorWeight（默认 0.7）+ textWeight（默认 0.3）
```

这种设计充分利用了两种搜索方式的各自优势：
- **向量搜索**：善于捕捉语义相似性
- **关键词搜索**：确保精确匹配的召回率

**Hybrid Search 实现** ([hybrid.ts](file:///Users/waterwu/www/github/openclaw/src/memory/hybrid.ts))：

```typescript
export function mergeHybridResults(params: {
  vector: HybridVectorResult[];
  keyword: HybridKeywordResult[];
  vectorWeight: number;
  textWeight: number;
}) {
  const byId = new Map<string, HybridEntry>();
  
  // 合并向量结果和关键词结果
  for (const r of params.vector) {
    byId.set(r.id, { ..., vectorScore: r.vectorScore, textScore: 0 });
  }
  for (const r of params.keyword) {
    const existing = byId.get(r.id);
    if (existing) {
      existing.textScore = r.textScore;
    } else {
      byId.set(r.id, { ..., vectorScore: 0, textScore: r.textScore });
    }
  }
  
  // 加权计算最终分数
  return Array.from(byId.values()).map((entry) => ({
    ...,
    score: params.vectorWeight * entry.vectorScore + params.textWeight * entry.textScore,
  }));
}
```

## 2.3 Embedding Provider 抽象层

系统定义了统一的 Embedding Provider 接口：

```typescript
type EmbeddingProvider = {
  id: string;                                    // 提供商标识
  model: string;                                 // 模型名称
  embedQuery: (text: string) => Promise<number[]>;   // 单条文本向量化
  embedBatch: (texts: string[]) => Promise<number[][]>;  // 批量向量化
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
// 从 embeddings.ts
async function createEmbeddingProvider(options: EmbeddingProviderOptions) {
  // 尝试主 Provider
  try {
    return await createOpenAiEmbeddingProvider(options);
  } catch (err) {
    if (options.fallback === "none") throw err;
    // 自动降级
    if (options.fallback === "gemini") {
      return createGeminiEmbeddingProvider(options);
    }
    // ...
  }
}
```

## 2.4 向量数据库选型

OpenClaw 采用 **SQLite + sqlite-vec** 组合作为向量存储方案：

- 使用 `vec_distance_cosine` 计算余弦相似度
- 单文件存储，零配置部署
- 比 Chroma、Pinecone 等云服务轻量得多

### 2.4.1 核心表结构

```sql
-- 文件表：记录已索引的文件
CREATE TABLE files (
  path TEXT,
  source TEXT,      -- 'memory' 或 'sessions'
  hash TEXT,        -- 文件内容哈希
  mtime_ms REAL,    // 修改时间
  size REAL
);

-- Chunks 表：文本分块信息
CREATE TABLE chunks (
  id TEXT PRIMARY KEY,
  path TEXT,
  source TEXT,
  start_line INTEGER,
  end_line INTEGER,
  text TEXT,
  source TEXT,
  model TEXT,       -- 使用的 Embedding 模型
  hash TEXT,        -- chunk 内容哈希
  embedding TEXT,   -- JSON 序列化的向量
  updated_at INTEGER
);

-- 向量索引：使用 sqlite-vec 扩展
CREATE VIRTUAL TABLE chunks_vec USING vec0(embedding FLOAT[768]);

-- 全文搜索：使用 FTS5
CREATE VIRTUAL TABLE chunks_fts USING fts5(path, text, tokenize='porter');

-- Embedding 缓存：避免重复计算
CREATE TABLE embedding_cache (
  hash TEXT PRIMARY KEY,
  embedding BLOB
);
```

### 2.4.2 向量查询示例

```sql
-- 余弦相似度查询（距离越小越相似）
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
private watcher: FSWatcher | null = null;  // chokidar 文件监听器
private dirty = false;          // 内存文件标记为脏
private sessionsDirty = false;  // 会话文件标记为脏
```

### 2.5.1 监听配置

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
  
  this.watcher.on("add", markDirty);
  this.watcher.on("change", markDirty);
  this.watcher.on("unlink", markDirty);
}
```

### 2.5.2 会话增量同步

- 使用 `chokidar` 监听文件变化
- 采用 5 秒 debounce 策略，避免频繁触发重索引
- 支持增量同步，会话文件只同步新增内容

```typescript
const SESSION_DELTA_BYTES = 100_000;      // 默认 100KB
const SESSION_DELTA_MESSAGES = 50;        // 默认 50 条消息
```

详见 [04-memory-chunk.md](./04-memory-chunk.md) 中的更新逻辑。

## 2.6 Embedding 缓存机制

系统内置 LRU 缓存，避免重复调用 Embedding API：

```typescript
private readonly cache: { enabled: boolean; maxEntries?: number };
// 数据存储在 EMBEDDING_CACHE_TABLE 表中
```

## 2.7 单例缓存管理

MemoryIndexManager 使用单例模式，配合全局缓存避免重复创建：

```typescript
const INDEX_CACHE = new Map<string, MemoryIndexManager>();
// 同一个 agent + workspace 只创建一个实例
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
// memory_search 工具描述
"Mandatory recall step: semantically search MEMORY.md + memory/*.md 
(and optional session transcripts) before answering questions about 
prior work, decisions, dates, people, preferences, or todos; 
returns top snippets with path + lines."
```

这意味着 AI 在回答任何关于历史工作、决策、日期、人员偏好或待办事项的问题前，都**强制**需要先执行记忆搜索。

## 2.10 设计亮点总结

| 特性 | 实现方式 | 优点 |
|------|----------|------|
| 混合搜索架构 | 向量 + 关键词融合 | 兼顾语义与精确匹配 |
| Provider 抽象层 | 三种后端 + 自动降级 | 易于扩展 |
| SQLite 轻量方案 | 单文件存储 | 运维成本低 |
| 智能增量同步 | 按 deltaBytes/deltaMessages | 避免全量重索引 |
| 强制记忆召回 | System Prompt 强制规则 | 确保可靠性 |

详见 [03-memory-files.md](./03-memory-files.md) 和 [04-memory-chunk.md](./04-memory-chunk.md)。
