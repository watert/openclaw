# 六、Plugin 系统与 Skills 系统对比分析

## 6.1 核心定位差异

Plugin 和 Skills 是 OpenClaw 中两个相互独立又协同工作的扩展机制，但它们解决的完全是不同层面的问题。Plugin 是**底层代码扩展框架**，为开发者提供编程式的能力注入；Skills 是**LLM 教学材料框架**，告诉 AI 应该在什么场景下、如何使用工具。理解这两者的本质区别，是掌握 OpenClaw 扩展体系的关键所在。

Plugin 系统赋予开发者直接在运行时环境中注册工具、拦截消息、注入生命周期钩子的能力，相当于为 OpenClaw 添加新的"器官"；而 Skills 系统则更像是给 AI 编写使用手册，告诉它这些新"器官"应该在什么情况下使用、如何使用。这种职责分离的设计非常精妙：Plugin 解决"能不能做"的问题，Skills 解决"该不该做"的问题。

从技术实现角度来看，Plugin 是纯代码层面的扩展，需要编写 TypeScript 或 JavaScript 代码，通过manifest 定义和 runtime API 完成注册；Skills 则是纯文档层面的指导，使用 Markdown 配合 YAML frontmatter 编写，通过特定的目录结构和加载机制被 Agent 读取。两者完全不在同一个抽象层次，却在功能上形成了完美的互补。

## 6.2 Plugin 系统架构详解

### 6.2.1 系统架构总览

Plugin 系统的核心设计目标是提供一个安全、可控、可扩展的代码加载机制。整体架构可以分为四个核心层次：Manifest 定义层、Loader 加载层、Registry 注册层和 Runtime 运行时层。这四个层次各司其职，共同构成了 Plugin 系统的完整生命周期管理。

Manifest 定义层负责描述插件的元信息和配置规范，所有插件必须在根目录包含 `openclaw.plugin.json` 文件，该文件声明了插件的唯一标识、配置模式、支持的功能类型（kind）、绑定的技能目录等信息。Loader 加载层负责发现、验证和加载插件代码，使用 jiti 运行时进行动态导入，支持 TypeScript 和 JavaScript 混合开发。Registry 注册层负责收集插件注册的各种能力（工具、钩子、命令、频道等），并维护全局的单例注册表。Runtime 运行时层则提供插件在执行期间所需的各种上下文和服务。

这种分层设计的好处是职责清晰、边界明确。Manifest 只负责"说什么"，Loader 只负责"怎么读"，Registry 只负责"怎么存"，Runtime 只负责"怎么用"。任何一层的变更都不会影响其他层，这种松耦合的设计让整个系统具有很强的可维护性和可扩展性。

### 6.2.2 Manifest 定义与配置模式

Manifest 文件是插件与 OpenClaw 框架之间的契约，其定义位于 `src/plugins/manifest.ts` 中。核心结构包括以下几个关键字段：id 作为插件的唯一标识，必须全局唯一且符合命名规范；configSchema 定义了配置的模式，采用 JSON Schema 语法，允许插件声明自己需要的配置参数；kind 标记插件的类型，目前支持 "memory" 类型用于指定内存管理插件；skills 数组声明插件绑定的技能目录路径；channels 和 providers 分别声明支持的通讯平台和模型提供商。

```json
{
  "id": "github-integration",
  "name": "GitHub Integration",
  "description": "Enhanced GitHub integration for OpenClaw",
  "version": "1.0.0",
  "configSchema": {
    "type": "object",
    "properties": {
      "accessToken": { "type": "string" },
      "defaultRepo": { "type": "string" }
    },
    "required": ["accessToken"],
    "additionalProperties": false
  },
  "kind": "memory",
  "skills": ["skills/github"],
  "channels": ["github"],
  "providers": ["github-models"]
}
```

配置模式支持多种验证方式，包括 safeParse、parse、validate 和 jsonSchema 四种模式。safeParse 是推荐的验证方式，返回包含 success 布尔值、data 数据和 error 信息的结构化结果；parse 用于简单的类型转换；validate 返回更详细的错误信息数组；jsonSchema 则允许直接使用 JSON Schema 进行验证。插件可以根据自己的复杂度选择合适的验证模式。

UI Hints 是 Manifest 的另一个重要特性，允许插件为配置项提供更友好的界面描述，包括 label 标签、help 帮助文本、advanced 是否属于高级选项、sensitive 是否敏感字段（不显示明文）、placeholder 占位符等。这些信息会被 UI 层消费，用于生成配置表单。

### 6.2.3 Loader 加载流程详解

Loader 是 Plugin 系统的核心引擎，负责将插件代码从磁盘加载到内存并完成注册。其主要流程可以分为发现阶段、解析阶段、验证阶段、执行阶段和注册阶段五个步骤。整个流程在 `src/plugins/loader.ts` 中实现，支持缓存、验证模式等多种选项。

发现阶段首先扫描预定义的目录位置，包括 OpenClaw 内置目录、工作区目录和用户自定义目录。对于每个候选路径，Loader 会尝试解析 Manifest 文件，如果找不到有效的 Manifest，则跳过该路径。解析阶段将 Manifest 文件解析为结构化的 PluginManifest 对象，验证必要字段的有效性，并进行规范化处理（如字符串 trim、数组过滤等）。

验证阶段是 Loader 的核心安全环节，它首先检查插件是否被明确启用（通过 allowlist 或 enabled 标志），然后验证配置是否符合 configSchema 定义。Loader 使用 schema-validator 模块进行 JSON Schema 验证，支持缓存以提升重复加载的性能。如果配置验证失败，插件状态会被标记为 "error"，并记录详细的错误信息。

执行阶段使用 jiti 运行时动态导入插件模块。jiti 是一个支持 TypeScript、ESM、CJS 的通用模块加载器，能够在运行时直接导入 .ts 文件而无需预编译。Loader 会根据模块的导出类型判断插件的注册方式：如果导出的是函数，则该函数被视为 register 回调；如果导出的是对象，则查找其 register 或 activate 方法。

注册阶段是 Loader 的最后一步，它创建 OpenClawPluginApi 对象并调用插件的注册回调。API 对象包含了插件所需的所有运行时能力，包括配置访问、工具注册、钩子注册、命令注册等。注册回调可以同步或异步执行，但如果返回 Promise，Loader 会发出警告并忽略异步结果。

### 6.2.4 Registry 注册机制

Registry 是 Plugin 系统的数据枢纽，负责收集和存储所有插件注册的各种能力。核心数据结构定义在 `src/plugins/registry.ts` 中，包括 plugins、tools、hooks、channels、providers、gatewayHandlers、httpHandlers、httpRoutes、cliRegistrars、services、commands 等多个数组和映射。

每种注册类型都有对应的 Registration 结构，例如 PluginToolRegistration 记录了插件 ID、工厂函数、工具名称列表、是否可选等信息；PluginHookRegistration 记录了插件 ID、钩子条目、关注的事件列表等。这些 Registration 结构不仅存储注册信息，还包含源文件路径等元数据，便于后续的诊断和调试。

Registry 还实现了注册冲突检测机制。对于工具名称、钩子名称、HTTP 路由路径、Gateway 方法名等可能产生冲突的资源，Registry 会在注册时进行唯一性检查。如果检测到重复注册，会生成相应的诊断信息并拒绝注册。这种设计防止了插件之间的无意覆盖，提高了系统的可预测性。

### 6.2.5 Runtime API 能力矩阵

Runtime API 是插件与 OpenClaw 框架交互的主要接口，定义在 `src/plugins/types.ts` 的 OpenClawPluginApi 类型中。这个 API 提供了十二种核心能力，覆盖了插件开发的大部分场景。

registerTool 允许插件注册 Agent 工具。工具可以是 AnyAgentTool 实例或工厂函数（接受上下文参数返回工具），支持通过 names 数组声明多个名称，通过 optional 标记为可选工具。可选工具默认不会被加载，除非在配置中显式声明。

registerHook 和 on 允许插件注入生命周期钩子。registerHook 是传统的钩子注册方式，支持通过 HookEntry 定义钩子的完整元信息；on 是更简洁的 API，通过 hookName 直接指定钩子类型。OpenClaw 提供了 14 个钩子点，包括 before_agent_start、agent_end、before_compaction、message_received、message_sending、message_sent、before_tool_call、after_tool_call、tool_result_persist、session_start、session_end、gateway_start、gateway_stop 等。

registerCommand 允许插件注册斜杠命令。与 Agent 工具不同，命令是绕过 LLM 直接执行的，这适用于简单的状态切换、状态查询等功能。命令处理器接收 PluginCommandContext 参数，包含发送者 ID、频道、授权状态、原始参数等信息。

registerChannel 允许插件注册通讯平台适配器，这是 OpenClaw 多平台支持的核心扩展机制。registerProvider 允许插件注册模型提供商，用于扩展支持的 AI 模型。registerGatewayMethod 允许插件扩展 Gateway API，提供新的远程调用能力。

registerHttpHandler 和 registerHttpRoute插件提供 HTTP 服务能力。registerHttpHandler 处理原始 允许的 HTTP 请求，registerHttpRoute 则提供更结构化的路由处理。registerService 允许插件注册后台服务，支持 start/stop 生命周期管理。registerCli 允许插件扩展命令行界面。

### 6.2.6 Plugin Origin 与加载优先级

Plugin 系统支持四种来源（origin），每种来源有不同的加载优先级和权限级别。bundled 插件随 OpenClaw 发行版一起提供，具有最高优先级，不能被其他同名插件覆盖。global 插件安装在全局目录，优先级次之。workspace 插件位于工作区内，优先级较低但对用户更友好。config 插件通过配置直接指定路径，优先级最低但灵活性最高。

当多个来源存在同名插件时，只有优先级最高的版本会被加载，其他版本会被标记为 disabled 并记录 "overridden by <origin> plugin" 错误。这种设计确保了核心功能的安全性，同时为用户提供了可控的覆盖机制。

## 6.3 Skills 系统架构详解

### 6.3.1 系统定位与设计目标

Skills 系统是 OpenClaw 的 LLM 教学框架，核心目标是将工具的使用方法以 AI 可理解的方式传递给模型。每个 Skill 是一个独立的教学单元，包含工具描述、使用示例、最佳实践等内容，Agent 在推理时会读取这些信息来决定是否以及如何使用特定工具。

Skills 的设计理念来源于 AgentSkills 规范，这是一套专门为 AI Agent 设计的技术文档格式。OpenClaw 完全兼容这套规范，并在此基础上进行了扩展，增加了 OpenClaw 特有的元数据机制（如二进制依赖、环境变量需求、配置要求等）。

与 Plugin 的代码扩展不同，Skills 完全是声明式的。开发者不需要编写任何代码，只需要按照规定的格式编写 Markdown 文档，OpenClaw 会自动解析这些文档并将其注入到 Agent 的系统提示中。这种设计大大降低了技能开发的门槛，让非程序员也能为 OpenClaw 贡献新的能力。

### 6.3.2 技能目录结构与加载层次

Skills 支持从多个目录加载，不同来源的技能有不同的加载顺序和覆盖规则。加载层次从高到低依次为：workspace/skills 位于工作区根目录下，具有最高优先级，用于存放特定 Agent 的专属技能；~/.openclaw/skills 是用户管理目录，对所有 Agent 可见，用于存放共享技能；bundled skills 随 OpenClaw 发行版提供，包含基础技能集合；plugins/*/skills 是插件绑定的技能目录，随插件启用而加载；skills.load.extraDirs 是用户配置的额外目录，优先级最低。

当同名技能存在于多个来源时，高优先级的版本会覆盖低优先级的版本。这种设计允许用户在工作区中覆盖默认技能，实现定制化而不修改原始文件。覆盖机制对于 bug 修复、功能增强等场景非常有用。

加载流程由 `src/agents/skills/workspace.ts` 中的 loadSkillEntries 函数实现。它首先收集所有来源的技能目录，然后使用 @mariozechner/pi-coding-agent 的 loadSkillsFromDir 函数加载每个目录中的技能，最后按照优先级合并去重。合并后的技能会进行 frontmatter 解析和元数据提取，为后续的过滤和注入做准备。

### 6.3.3 SKILL.md 文件格式规范

每个 Skill 必须包含一个 SKILL.md 文件作为核心描述文档。该文件采用 Markdown 语法，并使用 YAML frontmatter 定义元数据。frontmatter 位于文件顶部，使用三横线包裹，包含 name、description、metadata 等必需或可选字段。

name 是技能的标识名称，通常使用 kebab-case 格式，如 "github"、"web-search"、"image-generation" 等。description 是技能的简短描述，会被注入到 Agent 的系统提示中，帮助 AI 理解技能的用途。metadata 是一个 JSON 对象，包含 OpenClaw 特有的扩展信息。

```markdown
---
name: github
description: "Interact with GitHub using the `gh` CLI. Use `gh issue`, `gh pr`, `gh run`, and `gh api` for issues, PRs, CI runs, and advanced queries."
metadata: {"openclaw":{"emoji":"🐙","requires":{"bins":["gh"]},"install":[{"id":"brew","kind":"brew","formula":"gh","bins":["gh"],"label":"Install GitHub CLI (brew)"}]}}
---

# GitHub Skill

Use the `gh` CLI to interact with GitHub. Always specify `--repo owner/repo` when not in a git directory, or use URLs directly.

## Pull Requests

Check CI status on a PR:
```bash
gh pr checks 55 --repo owner/repo
```

List recent workflow runs:
```bash
gh run list --repo owner/repo --limit 10
```
```

metadata.openclaw 包含以下主要字段：always 设为 true 时技能始终加载，跳过其他条件检查；emoji 用于 macOS Skills UI 显示；os 指定技能支持的操作系统列表；requires.bins 声明必需的二进制命令；requires.anyBins 声明任一满足即可的命令；requires.env 声明必需的环境变量；requires.config 声明必需的配置项为真；primaryEnv 声明主要的环境变量名（用于 skills.entries.<name>.apiKey 注入）；install 声明一键安装规格；homepage 声明技能的主页 URL。

### 6.3.4 技能过滤与注入机制

Skills 在加载后会经过多层过滤，只有满足条件的技能才会被注入到 Agent 的系统提示中。过滤发生在 buildWorkspaceSkillSnapshot 函数中，它调用 filterSkillEntries 应用所有过滤条件。

过滤条件包括：enabled 状态（skills.entries.<name>.enabled 必须为 true 或未设置）；二进制依赖（requires.bins 声明的命令必须存在于 PATH）；环境变量（requires.env 声明的变量必须存在或由配置提供）；配置项（requires.config 声明的配置路径必须为真）；操作系统（os 列表必须包含当前系统）。

通过过滤的技能会被格式化为 XML 列表并注入到系统提示中。注入的格式由 pi-coding-agent 的 formatSkillsForPrompt 函数生成，包含每个技能的名称、描述、位置等信息。注入开销是确定的：基础开销 195 字符，每技能约 97 字符加字段长度。这为 Token 预算规划提供了可预测性。

### 6.3.5 技能命令与工具派发

Skills 系统支持将技能暴露为用户可调用的斜杠命令。通过 user-invocable 字段控制，默认值为 true。当启用时，系统会自动注册 /<skill-name> 格式的命令，用户可以直接调用。

command-dispatch 字段提供了更高级的派发机制。当设为 "tool" 时，命令会绕过 LLM 直接派发到指定工具。配合 command-tool 字段可以指定目标工具名称，command-arg-mode 控制参数传递方式（支持 "raw" 原始传递）。这种机制适用于需要确定性行为的场景，如文件操作、系统命令等。

```markdown
---
name: summarize
description: "Summarize text content using the summarize CLI tool"
user-invocable: true
command-dispatch: tool
command-tool: bash
command-arg-mode: raw
---
```

### 6.3.6 插件绑定技能

Plugin 可以通过 Manifest 的 skills 字段绑定技能目录。当插件启用时，其绑定的技能会自动加入技能加载池，参与正常的优先级排序和过滤流程。这种机制允许插件既提供底层工具实现，又提供对应的使用说明。

绑定流程在 `src/agents/skills/plugin-skills.ts` 的 resolvePluginSkillDirs 函数中实现。它首先加载所有启用的插件 Manifest，然后解析 skills 数组中的路径，将相对路径转换为绝对路径，最后检查路径是否存在且可访问。只有通过验证的技能目录才会被加入加载列表。

这种设计实现了 Plugin 与 Skills 的解耦：插件只需要声明技能路径，实际的加载和注入由 Skills 系统统一处理。插件启用时技能自动可用，插件禁用时技能自动移除，无需额外的同步逻辑。

## 6.4 Plugin 与 Skills 协同机制

### 6.4.1 能力提供与使用指导的分离

Plugin 和 Skills 的关系本质上是"能力提供"与"使用指导"的分离。Plugin 负责提供实际的工具实现、HTTP 服务、通讯适配等能力，这些能力以代码形式存在于运行时中。Skills 负责描述这些能力的使用方法，包括什么场景下应该使用、如何使用、有哪些注意事项等，这些描述以文档形式存在于文件系统中。

这种分离带来了几个显著好处。首先是关注点分离，Plugin 开发者专注于功能实现，Skill 编写者专注于使用体验，职责清晰。其次是复用性，同一个工具实现可以被多个 Skill 教学（不同场景、不同语言），同一个 Skill 可以引用多个工具实现。最后是安全性，Skill 只是文档，不会直接执行代码，避免了恶意 Skill 执行任意操作的风险。

### 6.4.2 工具注册与工具教学的对齐

在实际开发中，Plugin 注册的工具名称、参数格式需要与 Skill 描述保持一致。这种对齐通常通过约定俗成的命名规范来实现：Plugin 注册的工具名与 Skill 的 name 保持一致，Skill 中的命令示例使用工具的真实名称和参数格式。

当工具名称发生变更时，Plugin 开发者应该同步更新 Skill 描述。如果需要保持向后兼容，可以在 Plugin 中注册别名工具指向新工具实现。Skills 系统也支持通过 frontmatter 的 skillKey 字段自定义配置键名，允许工具名和配置键名不一致。

### 6.4.3 命令派发与工具绑定的联动

command-dispatch 机制是 Plugin 与 Skills 联动的关键点。当 Skill 声明 command-dispatch: tool 时，系统会查找同名或指定名称的工具，并将用户输入的原始参数直接传递给它。这个过程中，Plugin 负责提供工具实现，Skill 负责定义派发规则，OpenClaw 负责将两者连接起来。

这种联动机制支持多种使用模式。对于简单工具，Skill 可以声明 command-dispatch: tool 和 command-tool: <tool-name>，实现命令到工具的直接映射。对于复杂工具，Skill 可以只声明 command-dispatch: tool，让系统自动查找同名工具。对于需要参数处理的工具，Skill 可以使用 command-arg-mode: raw 获取原始输入，由工具自行解析。

## 6.5 总结与最佳实践

### 6.5.1 核心差异总结

Plugin 和 Skills 在抽象层次、实现方式、功能定位上有着本质的区别。Plugin 是代码扩展框架，面向开发者，使用 TypeScript/JavaScript 编写，提供运行时能力扩展；Skills 是文档教学框架，面向 LLM，使用 Markdown/YAML 编写，提供使用指导。在选择使用哪种机制时，应该首先明确需求：如果是添加新功能、新能力，应该使用 Plugin；如果是改进 AI 对现有功能的使用理解，应该使用 Skills。

### 6.5.2 插件开发最佳实践

开发 Plugin 时应该遵循以下原则：保持单一职责，每个插件专注于一个功能领域；正确声明依赖，使用 configSchema 声明配置需求，使用 requires 声明系统依赖；提供清晰的错误信息，在 register 阶段进行充分的验证和错误处理；遵守安全规范，对于敏感操作添加授权检查，不要在日志中输出敏感信息。

### 6.5.3 技能编写最佳实践

编写 Skill 时应该遵循以下原则：保持简洁明确，技能描述应该直截了当告诉 AI 什么情况下使用、怎么使用；提供实用示例，使用真实场景的代码示例，帮助 AI 理解使用方法；声明依赖条件，明确告诉 AI 这个技能需要哪些前置条件（二进制、环境变量、配置等）；避免过度注入，只包含必要的技能，过多的技能会增加上下文长度和推理复杂度。
