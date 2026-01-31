# 五、Agent System Prompt 完整解析

System Prompt 是 OpenClaw Agent 的「灵魂」，决定了 AI 的行为规范、思考方式和响应模式。

## 5.1 核心入口

System Prompt 构建入口位于 [system-prompt.ts](file:///Users/waterwu/www/github/openclaw/src/agents/system-prompt.ts)：

```typescript
export function buildAgentSystemPrompt(params: {
  workspaceDir: string;
  defaultThinkLevel?: ThinkLevel;
  reasoningLevel?: ReasoningLevel;
  extraSystemPrompt?: string;
  ownerNumbers?: string[];
  reasoningTagHint?: boolean;
  toolNames?: string[];
  toolSummaries?: Record<string, string>;
  modelAliasLines?: string[];
  userTimezone?: string;
  userTime?: string;
  userTimeFormat?: ResolvedTimeFormat;
  contextFiles?: EmbeddedContextFile[];
  skillsPrompt?: string;
  heartbeatPrompt?: string;
  docsPath?: string;
  workspaceNotes?: string[];
  ttsHint?: string;
  promptMode?: PromptMode;
  runtimeInfo?: {
    agentId?: string;
    host?: string;
    os?: string;
    arch?: string;
    node?: string;
    model?: string;
    defaultModel?: string;
    channel?: string;
    capabilities?: string[];
    repoRoot?: string;
  };
  messageToolHints?: string[];
  sandboxInfo?: {
    enabled: boolean;
    workspaceDir?: string;
    workspaceAccess?: "none" | "ro" | "rw";
    agentWorkspaceMount?: string;
    browserBridgeUrl?: string;
    browserNoVncUrl?: string;
    hostBrowserAllowed?: boolean;
    elevated?: {
      allowed: boolean;
      defaultLevel: "on" | "off" | "ask" | "full";
    };
  };
  reactionGuidance?: {
    level: "minimal" | "extensive";
    channel: string;
  };
})
```

## 5.2 Prompt 组成结构

System Prompt 由多个模块拼接而成：

```
[1. 基础指令] 基础行为规范、角色定义
    ↓
[2. 强制规则] 记忆召回、工具使用（CRITICAL）
    ↓
[3. 记忆召回] Memory Recall（强制步骤）
    ↓
[4. Skills] 可选的行为规范
    ↓
[5. 工具列表] 可用工具及描述
    ↓
[6. 运行时信息] Agent ID、系统环境、模型信息
    ↓
[7. 通道配置] 消息格式、回复规则
    ↓
[8. 心跳/健康检查] 系统状态监控
    ↓
[9. 额外指令] 用户自定义
```

## 5.3 关键模块详解

### 5.3.1 基础指令 (Foundation)

```typescript
// 默认基础指令
export const DEFAULT_FOUNDATION_PROMPT = [
  "You are OpenClaw, an advanced AI assistant running on the user's own machine.",
  "Think carefully about your response and plan your actions.",
  "Prefer to ask clarifying questions rather than make assumptions.",
  "Admit what you don't know rather than hallucinate.",
].join("\n");
```

### 5.3.2 强制规则 (Critical)

**记忆召回强制规则**：

```typescript
export const DEFAULT_CRITICAL_PROMPT = [
  "Critical: ALWAYS FIRST perform memory recall before answering questions about prior work, decisions, dates, people, preferences, or todos. Use the memory_search tool for this. The tool returns results with file path and line numbers. Quote relevant parts in your response.",
  "Memory recall is NOT optional. You MUST do it first.",
].join("\n");
```

**这条规则确保 AI 在回答任何历史相关问题前，一定会先搜索记忆库。**

### 5.3.3 记忆召回 (Memory Recall)

```typescript
export const DEFAULT_MEMORY_RECALL_PROMPT = [
  "Mandatory recall step: semantically search MEMORY.md + memory/*.md (and optional session transcripts) before answering questions about prior work, decisions, dates, people, preferences, or todos; returns top snippets with path + lines.",
].join("\n");
```

### 5.3.4 工具使用 (Tool Use)

```typescript
export const DEFAULT_TOOL_USE_PROMPT = [
  "Use tools proactively to accomplish tasks. Don't ask the user to do things you can do yourself.",
  "When using tools, clearly explain what you're doing and why.",
  "If a tool fails, explain the error and suggest alternatives.",
  "After using a tool, always check the output and respond accordingly.",
].join("\n");
```

### 5.3.5 思考过程 (Thinking)

```typescript
export const DEFAULT_THINKING_PROMPT = [
  "You can think silently inside <think> tags.",
  "Keep your thinking focused and concise.",
  "Only include thoughts that help solve the user's request.",
  "Don't reveal your full chain of thought if not asked.",
].join("\n");
```

### 5.3.6 静默回复 (Silent Reply)

OpenClaw 支持一种特殊的静默回复机制：

```typescript
export const SILENT_REPLY_TOKEN = "__SILENT_REPLY__";

export const DEFAULT_SILENT_REPLY_PROMPT = [
  `If you have nothing to say, reply with exactly "${SILENT_REPLY_TOKEN}" and nothing else.`,
  "This is useful when memory flush finds nothing to store, or when no response is needed.",
].join("\n");
```

**使用场景**：Memory Flush 时，如果 AI 判断没有重要信息需要保存，就返回这个 Token 而不是生成无意义的文本。

### 5.3.7 心跳机制 (Heartbeat)

```typescript
export const DEFAULT_HEARTBEAT_PROMPT = [
  "Periodically report your status with <heartbeat> tags.",
  "Include system status, memory usage, and any ongoing operations.",
].join("\n");
```

## 5.4 工具列表动态生成

System Prompt 中的工具列表是根据实际启用的 Skills 动态生成的：

```typescript
// 从 skills/ 目录加载
const skillTools = await loadSkillTools(skills, workspaceDir);

// 构建工具描述
const toolDescriptions = skillTools.map((tool) => {
  return `- ${tool.name}: ${tool.description}`;
}).join("\n");

// 插入到 System Prompt
if (toolDescriptions.length > 0) {
  parts.push(`## Tools\n\n${toolDescriptions}`);
}
```

## 5.5 运行时信息注入

Agent 运行时会注入详细的系统信息：

```typescript
const runtimeInfoPrompt = [
  "## Runtime Info",
  "",
  `Agent ID: ${runtimeInfo.agentId}`,
  `Host: ${runtimeInfo.host}`,
  `OS: ${runtimeInfo.os} (${runtimeInfo.arch})`,
  `Node: ${runtimeInfo.node}`,
  `Model: ${runtimeInfo.model}`,
  `Channel: ${runtimeInfo.channel}`,
  "",
  runtimeInfo.capabilities?.length
    ? `Capabilities: ${runtimeInfo.capabilities.join(", ")}`
    : "",
].filter(Boolean).join("\n");
```

## 5.6 沙箱环境配置

如果启用了沙箱，System Prompt 会包含相关配置：

```typescript
if (sandboxInfo?.enabled) {
  parts.push("## Sandbox");
  parts.push(`Workspace: ${sandboxInfo.workspaceDir}`);
  parts.push(`Access: ${sandboxInfo.workspaceAccess}`);
  if (sandboxInfo.elevated?.allowed) {
    parts.push(`Elevated: ${sandboxInfo.elevated.defaultLevel}`);
  }
}
```

## 5.7 多通道适配

针对不同通讯平台，System Prompt 会有细微调整：

| 平台 | 特殊处理 |
|------|----------|
| Telegram | 支持 `/command` 格式 |
| Slack | 支持 `/openclaw` 提及格式 |
| Discord | 支持 `@OpenClaw` 提及格式 |
| iMessage | 文本格式简化 |

## 5.8 完整的 System Prompt 示例

```markdown
You are OpenClaw, an advanced AI assistant running on the user's own machine.

Critical: ALWAYS FIRST perform memory recall before answering questions about prior work, decisions, dates, people, preferences, or todos. Use the memory_search tool for this. The tool returns results with file path and line numbers. Quote relevant parts in your response.

Mandatory recall step: semantically search MEMORY.md + memory/*.md (and optional session transcripts) before answering questions about prior work, decisions, dates, people, preferences, or todos; returns top snippets with path + lines.

## Tools

- memory_search: Search memory files for relevant information
- read_file: Read the contents of a file
- write_file: Write content to a file
- list_files: List files in a directory
- exec: Execute shell commands
...

## Runtime Info

Agent ID: openclaw-mac
Host: MacBook-Pro.local
OS: macOS (arm64)
Node: v20.10.0
Model: gpt-4o
Channel: telegram

## Guidelines

- Think carefully about your response and plan your actions.
- Prefer to ask clarifying questions rather than make assumptions.
- Use tools proactively to accomplish tasks.

If you have nothing to say, reply with exactly "__SILENT_REPLY__" and nothing else.
```

## 5.9 思考层级配置 (ThinkLevel)

System Prompt 支持配置不同的思考深度：

```typescript
enum ThinkLevel {
  "off" = 0,      // 不使用 <think> 标签
  "low" = 1,      // 简要思考
  "normal" = 2,   // 标准思考深度
  "high" = 3,     // 深度思考
}
```

## 5.10 设计亮点

| 特性 | 实现方式 | 优点 |
|------|----------|------|
| **强制记忆召回** | System Prompt 硬编码规则 | 确保可靠性 |
| **模块化拼接** | 函数式构建，各模块独立 | 易维护、易测试 |
| **动态工具列表** | 根据 Skills 动态生成 | 高度可定制 |
| **运行时注入** | 动态拼接系统信息 | 信息始终最新 |
| **静默回复机制** | 特殊 Token 处理 | 避免无意义回复 |
| **多通道适配** | 条件性 Prompt 调整 | 一套代码多平台 |

## 5.11 相关文档

- **Memory 系统** → [02-memory-system.md](./02-memory-system.md)
- **文件保存** → [03-memory-files.md](./03-memory-files.md)
- **Chunk 机制** → [04-memory-chunk.md](./04-memory-chunk.md)
