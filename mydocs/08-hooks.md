# 八、Hooks 系统深度解析

OpenClaw 存在**两套独立的 Hook 系统**，它们服务于完全不同的场景，且架构设计理念迥异。理解这两套系统的差异，对于正确使用 OpenClaw 的扩展机制至关重要。本文档将深入分析 Internal Hooks（内部事件钩子）和 Plugin Hooks（插件扩展钩子）的架构设计、API 风格和使用场景。

## 8.1 两套 Hook 系统概览

在深入代码之前，首先需要建立对两套 Hook 系统的整体认知。Internal Hooks 是 OpenClaw 内部的事件驱动机制，用于在特定系统事件发生时执行自定义逻辑；而 Plugin Hooks 则是 Plugin SDK 提供的扩展点，允许插件在 Agent 生命周期、消息处理等关键节点注入代码。这两套系统不仅位置不同（分别位于 `src/hooks/` 和 `src/plugins/hooks.ts`），设计哲学也截然不同：Internal Hooks 追求简洁的事件广播模型，Plugin Hooks 则提供精细的上下文控制和结果处理能力。

```typescript
// Internal Hooks - 内部事件系统
// 位置: src/hooks/internal-hooks.ts
export type InternalHookEventType = "command" | "session" | "agent" | "gateway";

// Plugin Hooks - 插件扩展系统  
// 位置: src/plugins/types.ts
export type PluginHookName =
  | "before_agent_start" | "agent_end"
  | "before_compaction" | "after_compaction"
  | "message_received" | "message_sending" | "message_sent"
  | "before_tool_call" | "after_tool_call" | "tool_result_persist"
  | "session_start" | "session_end"
  | "gateway_start" | "gateway_stop";
```

**重要澄清**：Memory Flush 机制完全独立于 Hook 系统，它通过 `runMemoryFlushIfNeeded()` 直接函数调用实现，不涉及任何 Hook 机制。这一设计决策体现了 OpenClaw 对系统边界的清晰划分：Memory Flush 是 Auto-Reply 子系统的核心功能，不需要也不应该通过 Hook 触发。

## 8.2 Internal Hooks 系统详解

Internal Hooks 是 OpenClaw 的轻量级事件广播系统，设计目标是让用户和开发者能够在特定系统事件发生时执行自定义逻辑。这套系统主要服务于三类场景：会话生命周期管理（如 `/new` 命令触发会话保存）、命令审计（如记录所有命令执行日志）、以及系统启动行为（如 gateway 启动时执行 BOOT.md）。

### 8.2.1 核心架构设计

Internal Hooks 的核心是一个基于事件类型的注册表（Registry），支持两种粒度的监听：通用事件类型监听（如监听所有 `command` 事件）和特定动作监听（如仅监听 `command:new` 事件）。这种设计允许开发者根据需求选择合适的监听粒度，既可以捕获所有同类事件进行统一处理，也可以精确拦截特定操作。

```typescript
// src/hooks/internal-hooks.ts
const handlers = new Map<string, InternalHookHandler[]>();

export function registerInternalHook(eventKey: string, handler: InternalHookHandler): void {
  if (!handlers.has(eventKey)) {
    handlers.set(eventKey, []);
  }
  handlers.get(eventKey)!.push(handler);
}

export async function triggerInternalHook(event: InternalHookEvent): Promise<void> {
  const typeHandlers = handlers.get(event.type) ?? [];
  const specificHandlers = handlers.get(`${event.type}:${event.action}`) ?? [];
  const allHandlers = [...typeHandlers, ...specificHandlers];

  for (const handler of allHandlers) {
    try {
      await handler(event);
    } catch (err) {
      console.error(`Hook error [${event.type}:${event.action}]:`, err);
    }
  }
}
```

当事件被触发时，系统首先获取该事件类型的所有处理器，然后追加特定动作的处理器（如果有）。所有处理器按注册顺序依次执行，错误被捕获并记录但不会中断其他处理器的执行。这种「 fire-and-forget 」的错误处理模式确保了单个 Hook 的失败不会影响整个系统。

### 8.2.2 事件类型与数据结构

Internal Hooks 定义了四种事件类型，每种类型对应不同的系统组件和生命周期阶段。事件对象包含类型（type）、动作（action）、会话键（sessionKey）、上下文（context）、时间戳（timestamp）以及用于向用户传递消息的数组（messages）。这种设计允许 Hook 在处理过程中积累多个消息，最终统一展示给用户。

```typescript
// 事件类型定义
export type InternalHookEventType = "command" | "session" | "agent" | "gateway";

export interface InternalHookEvent {
  type: InternalHookEventType;
  action: string;
  sessionKey: string;
  context: Record<string, unknown>;
  timestamp: Date;
  messages: string[];
}
```

**当前支持的事件类型**：`command`（所有命令事件）、`command:new`（/new 命令）、`command:reset`（/reset 命令）、`command:stop`（/stop 命令）、`agent:bootstrap`（Agent 启动前）、`gateway:startup`（Gateway 启动）。系统文档指出未来将扩展更多事件类型，包括会话生命周期和 Agent 错误处理等。

### 8.2.3 Bundled Hooks 实现分析

OpenClaw 预置了四个 Bundled Hooks，直接打包在安装中，无需额外安装即可启用。这些 Hooks 覆盖了常见的扩展需求，每个 Hook 都是一个独立目录，包含 HOOK.md（元数据+文档）和 handler.ts（处理逻辑）。

**session-memory Hook** 是最具代表性的实现，它监听 `command:new` 事件，在用户执行 `/new` 命令开始新会话时，自动将上一会话的对话内容保存到 `memory/YYYY-MM-DD-slug.md` 文件。Hook 使用 LLM 生成会话摘要的 slug（标识符），确保文件名既有时间信息又有语义含义。

```typescript
// src/hooks/bundled/session-memory/handler.ts
const saveSessionToMemory: HookHandler = async (event) => {
  if (event.type !== "command" || event.action !== "new") return;

  // 读取会话文件内容
  const sessionContent = await getRecentSessionContent(sessionFile, messageCount);
  
  // 使用 LLM 生成 slug
  const slug = await generateSlugViaLLM({ sessionContent, cfg });
  
  // 写入 memory/YYYY-MM-DD-slug.md
  const filename = `${dateStr}-${slug}.md`;
  await fs.writeFile(memoryFilePath, entry, "utf-8");
};
```

**boot-md Hook** 监听 `gateway:startup` 事件，在 Gateway 启动后自动执行 BOOT.md 文件中的指令。这种设计允许用户在系统启动时执行初始化操作，如发送欢迎消息、运行诊断命令等。**command-logger Hook** 监听所有 `command` 事件，将命令执行记录以 JSONL 格式写入日志文件，用于审计和调试。**soul-evil Hook** 是一个有趣的「彩蛋」实现，它监听 `agent:bootstrap` 事件，有概率将注入的 SOUL.md 内容替换为 SOUL_EVIL.md，为长期运行的 Agent 会话增添一些不可预测性。

### 8.2.4 HOOK.md 元数据格式

每个 Bundled Hook 必须包含一个 HOOK.md 文件，使用 YAML 前matter 定义 Hook 的元数据，前matter 之后是 Hook 的详细文档。这种设计将元数据与文档紧密结合，便于开发者理解和配置 Hook，同时支持 CLI 工具解析和展示 Hook 信息。

```yaml
---
name: session-memory
description: "Automatically saves session context to memory when you issue /new"
homepage: https://docs.openclaw.ai/hooks/session-memory
metadata:
  { "openclaw": { "emoji": "💾", "events": ["command:new"], "requires": { "bins": ["node"] } } }
---
# Session Memory Hook

Documentation goes here...
```

元数据支持丰富的配置项：`emoji` 用于 CLI 展示、`events` 定义监听的事件数组、`requires` 指定运行依赖（如二进制命令、环境变量、配置文件等）、`os` 限制支持的操作系统、`install` 定义安装方式。Bundled Hooks 的 install 配置通常为 `[{"kind":"bundled"}]`，表明它们已内置在 OpenClaw 中。

### 8.2.5 自定义 Hook 开发

开发者可以在两个位置创建自定义 Hook：工作区 hooks 目录（`<workspace>/hooks/`）和用户级 hooks 目录（`~/.openclaw/hooks/`）。工作区 hooks 具有最高优先级，会覆盖同名的系统 hooks。自定义 Hook 的结构与 Bundled Hooks 相同，包含 HOOK.md 和 handler.ts 两个必需文件。

```typescript
// 自定义 Hook handler.ts 示例
import type { HookHandler } from "../../src/hooks/hooks.js";

const myHandler: HookHandler = async (event) => {
  if (event.type !== "command" || event.action !== "new") {
    return;
  }

  // 业务逻辑
  console.log("New command triggered!");
  
  // 可选：向用户发送消息
  event.messages.push("✨ Hook executed!");
};

export default myHandler;
```

## 8.3 Plugin Hooks 系统详解

Plugin Hooks 是 Plugin SDK 的核心扩展机制，与 Internal Hooks 面向最终用户不同，Plugin Hooks 主要服务于插件开发者。它提供了更精细的上下文控制、更丰富的扩展点和更强大的结果处理能力。这套系统的设计理念是让插件能够在 Agent 生命周期的关键节点进行拦截、修改和扩展。

### 8.3.1 架构设计理念

Plugin Hooks 的核心是一个注册表（PluginRegistry），存储所有插件注册的 Hook 处理器。与 Internal Hooks 的简单注册表不同，Plugin Hooks 支持优先级排序（priority）、精细的错误处理（catchErrors）和两种执行模式：并行执行（fire-and-forget）和顺序执行（sequential with result merging）。这种设计允许插件开发者根据 Hook 的特性选择合适的执行策略。

```typescript
// src/plugins/hooks.ts
function getHooksForName<K extends PluginHookName>(
  registry: PluginRegistry,
  hookName: K,
): PluginHookRegistration<K>[] {
  return (registry.typedHooks as PluginHookRegistration<K>[])
    .filter((h) => h.hookName === hookName)
    .sort((a, b) => (b.priority ?? 0) - (a.priority ?? 0));
}

export function createHookRunner(registry: PluginRegistry, options: HookRunnerOptions = {}) {
  // 并行执行：fire-and-forget
  async function runVoidHook<K extends PluginHookName>(
    hookName: K,
    event: Parameters<NonNullable<PluginHookRegistration<K>["handler"]>>[0],
    ctx: Parameters<NonNullable<PluginHookRegistration<K>["handler"]>>[1],
  ): Promise<void> {
    const hooks = getHooksForName(registry, hookName);
    const promises = hooks.map(async (hook) => {
      try {
        await hook.handler(event, ctx);
      } catch (err) {
        // 错误被捕获，不会中断其他处理器
      }
    });
    await Promise.all(promises);
  }

  // 顺序执行：结果合并
  async function runModifyingHook<K extends PluginHookName, TResult>(
    hookName: K,
    event: Parameters<NonNullable<PluginHookRegistration<K>["handler"]>>[0],
    ctx: Parameters<NonNullable<PluginHookRegistration<K>["handler"]>>[1],
    mergeResults?: (accumulated: TResult | undefined, next: TResult) => TResult,
  ): Promise<TResult | undefined> {
    const hooks = getHooksForName(registry, hookName);
    let result: TResult | undefined;

    for (const hook of hooks) {
      const handlerResult = await hook.handler(event, ctx);
      if (handlerResult !== undefined && handlerResult !== null) {
        result = mergeResults ? mergeResults(result, handlerResult) : handlerResult;
      }
    }
    return result;
  }
}
```

### 8.3.2 完整的 Hook 扩展点列表

Plugin Hooks 定义了 14 个扩展点，覆盖 Agent、Message、Tool、Session 和 Gateway 五个维度的生命周期。这些扩展点分为三类：拦截类（before_*）、结果类（after_*）和持久化类（persist）。拦截类 Hook 可以在动作发生前修改参数或取消动作，结果类 Hook 可以在动作发生后处理结果，持久化类 Hook 可以将结果写入外部存储。

**Agent 生命周期 Hooks**：`before_agent_start` 在 Agent 启动前触发，允许插件修改 System Prompt 和注入上下文；`agent_end` 在 Agent 结束（无论成功或失败）时触发，用于资源清理和状态持久化。

**消息处理 Hooks**：`message_received` 在消息到达时触发（fire-and-forget），适合日志记录和统计；`message_sending` 在消息发送前触发，可以修改消息内容或取消发送；`message_sent` 在消息发送完成后触发，用于确认和后续处理。

**工具调用 Hooks**：`before_tool_call` 在工具调用前触发，可修改参数或阻止调用；`after_tool_call` 在工具调用完成后触发，可处理返回结果；`tool_result_persist` 用于自定义工具结果的持久化方式。

**会话管理 Hooks**：`session_start` 在新会话开始时触发；`session_end` 在会话结束时触发。

**系统生命周期 Hooks**：`gateway_start` 在 Gateway 启动后触发；`gateway_stop` 在 Gateway 停止前触发。

```typescript
// src/plugins/types.ts
export type PluginHookName =
  | "before_agent_start" | "agent_end"
  | "before_compaction" | "after_compaction"
  | "message_received" | "message_sending" | "message_sent"
  | "before_tool_call" | "after_tool_call" | "tool_result_persist"
  | "session_start" | "session_end"
  | "gateway_start" | "gateway_stop";
```

### 8.3.3 上下文对象与结果类型

Plugin Hooks 为每类 Hook 定义了专门的上下文对象，包含该场景下所有可用的信息。这种设计比 Internal Hooks 的通用 `context: Record<string, unknown>` 类型安全得多，开发者可以直接访问所需的属性而无需进行类型断言。

```typescript
// Agent 上下文
export type PluginHookAgentContext = {
  agentId?: string;
  sessionKey?: string;
  workspaceDir?: string;
  messageProvider?: string;
};

// Message 上下文
export type PluginHookMessageContext = {
  channelId: string;
  accountId?: string;
  conversationId?: string;
};

// Tool 上下文
export type PluginHookToolContext = {
  agentId?: string;
  sessionKey?: string;
  toolName: string;
};
```

部分 Hook 支持返回值来修改系统行为。例如，`before_agent_start` 可以返回 `{ systemPrompt?: string; prependContext?: string }` 来修改 System Prompt；`message_sending` 可以返回 `{ content?: string; cancel?: boolean }` 来修改或取消消息发送。

```typescript
// before_agent_start 结果类型
export type PluginHookBeforeAgentStartResult = {
  systemPrompt?: string;
  prependContext?: string;
};

// message_sending 结果类型
export type PluginHookMessageSendingResult = {
  content?: string;
  cancel?: boolean;
};
```

### 8.3.4 优先级与错误处理

Plugin Hooks 支持为每个注册处理器指定 `priority`（优先级），数值越高的处理器越先执行。这在多个插件注册同名 Hook 时非常重要，可以控制处理器之间的执行顺序。例如，两个插件都注册了 `before_agent_start` Hook，优先级高的插件会先运行，可能影响后续插件看到的数据。

```typescript
export type PluginHookRegistration<K extends PluginHookName = PluginHookName> = {
  hookName: K;
  pluginId: string;
  priority?: number;
  handler: PluginHookHandlerMap[K];
};
```

错误处理通过 `HookRunnerOptions.catchErrors` 配置。当 `catchErrors` 为 `true`（默认值）时，单个 Hook 处理器的错误会被捕获并记录，不会影响其他处理器和主流程。当设为 `false` 时，任何 Hook 错误都会向上抛出，中断当前操作。这种设计允许插件开发者在开发和调试阶段启用严格模式，在生产环境中使用宽容模式。

## 8.4 两套系统对比分析

Internal Hooks 和 Plugin Hooks 虽然名称相似，但服务于完全不同的场景和用户群体。理解它们的差异是正确使用 OpenClaw 扩展机制的基础。

### 8.4.1 设计目标对比

Internal Hooks 的设计目标是让最终用户和系统集成者能够在不修改核心代码的情况下扩展系统行为。它的 API 简洁直观，学习成本低，适合实现「即插即用」的功能扩展。Plugin Hooks 的设计目标是让插件开发者能够在 Agent 生命周期的关键节点进行深度定制，需要处理更复杂的上下文和返回值，但提供了更强大的控制能力。

### 8.4.2 技术架构对比

| 维度 | Internal Hooks | Plugin Hooks |
|------|----------------|---------------|
| 位置 | `src/hooks/` | `src/plugins/hooks.ts` |
| 目标用户 | 最终用户、系统集成者 | 插件开发者 |
| 事件模型 | 通用事件广播 | 精细化扩展点 |
| 错误处理 | 捕获并记录 | 可配置捕获或抛出 |
| 执行模式 | 并行 | 并行或顺序 |
| 结果处理 | 无（fire-and-forget） | 支持返回值合并 |
| 优先级 | 无 | 支持 |
| 上下文类型 | 通用 Record | 专用类型 |

### 8.4.3 典型使用场景对比

**Internal Hooks 适用场景**：会话保存与恢复（session-memory）、命令审计日志（command-logger）、系统启动初始化（boot-md）、开发者实验性功能。**Plugin Hooks 适用场景**：消息内容过滤与修改、工具调用拦截与增强、Agent 行为深度定制、跨插件协同扩展。

### 8.4.4 混淆风险与澄清

在实际使用中，最常见的混淆是将 Memory Flush 误认为是一种 Hook 机制。这是因为 Memory Flush 涉及「触发条件检查」和「执行时机控制」，看起来像是某种事件驱动的逻辑。但经过代码分析，Memory Flush 完全通过 `runMemoryFlushIfNeeded()` 直接函数调用实现，与 Hook 系统完全无关。它位于 `src/auto-reply/reply/` 目录下，属于 Auto-Reply 子系统的核心功能。

另一个需要澄清的概念是「Bundled Hooks」与「Plugin Hooks」的关系。Bundled Hooks 是 Internal Hooks 系统的预置实现（位于 `src/hooks/bundled/`），而 Plugin Hooks 是完全独立的插件扩展机制。两者虽然都叫「Hook」，但分别属于不同的扩展体系。

## 8.5 总结与最佳实践

OpenClaw 的两套 Hook 系统体现了「简单扩展」与「深度定制」的平衡。Internal Hooks 提供了低门槛的扩展方式，适合大多数用户和集成场景；Plugin Hooks 提供了强大的定制能力，适合需要深度集成的插件开发。正确选择使用哪套系统，关键在于理解你的需求是「在特定事件发生时执行逻辑」还是「在关键节点拦截和修改系统行为」。

**最佳实践建议**：优先使用 Internal Hooks 实现简单扩展；仅当需要修改系统行为、拦截关键操作时才使用 Plugin Hooks；永远不要假设 Memory Flush 是一种 Hook 机制——它是 Auto-Reply 子系统的独立功能。

