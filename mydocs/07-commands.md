# 七、Commands 模块深度解析

## 7.1 系统定位与设计目标

Commands 模块是 OpenClaw 架构中一个常被误解但至关重要的组件。表面上看，它提供了 `openclaw` 命令行工具的全部命令支持；更深层次地，它是 OpenClaw 系统的"运维操作层"，封装了所有与系统配置、管理、诊断相关的业务逻辑。这个模块的精妙之处在于它的双重身份：既是对外的 CLI 接口，又是对内的公共 API 库。

理解 Commands 模块的关键在于认识到它的设计哲学是将"运维操作"抽象为独立的、可复用的函数单元。每个 Command 都被实现为一个导出的异步函数，接受配置参数和运行时环境，然后执行特定的操作并输出结果。这种设计使得同一套逻辑既可以被用户在命令行手动调用，也可以被 OpenClaw 内部的 Wizard 向导、Doctor 诊断工具等其他流程复用。

Commands 模块的目标用户有两类。第一类是系统管理员和高级用户，他们通过 CLI 直接与 OpenClaw 交互，执行配置、监控、故障排除等操作。第二类是 OpenClaw 内部的其他模块，如 Wizard 向导、Doctor 诊断、Configure 配置工具等，它们调用 Commands 提供的函数来完成各自的工作。这种双重定位要求 Commands 模块既要提供友好的 CLI 体验，又要保持良好的 API 可编程性。

## 7.2 架构层次与入口机制

### 7.2.1 CLI 入口架构

Commands 模块的 CLI 入口遵循标准的 Commander.js 模式，整个架构分为四个层次：程序入口层、命令注册层、命令实现层和路由匹配层。这种分层设计确保了命令注册的灵活性和命令执行的一致性。

程序入口层位于 `src/cli/program.ts`，它导出了 `buildProgram` 函数用于创建 Commander 程序实例。这个函数首先创建程序上下文，然后配置帮助信息，注册前置钩子，最后调用命令注册层完成所有命令的注册。程序的执行入口在 `src/cli/run-main.ts`，它解析命令行参数并分发到对应的命令处理函数。

命令注册层是 CLI 架构的核心，位于 `src/cli/program/` 目录下。`build-program.ts` 是程序构建的入口文件，它组装各个注册器并生成完整的命令树。`command-registry.ts` 定义了命令注册表和注册机制，它是整个注册系统的枢纽。`register.setup.ts`、`register.onboard.ts`、`register.configure.ts` 等文件分别注册不同类别的命令组。每个注册器都是一个函数，接受 Commander 程序实例并修改它，添加新的命令和选项。

命令实现层位于 `src/commands/` 目录下，包含了所有命令的实际业务逻辑。这个目录按照功能划分为多个子目录：`agent/` 包含 Agent 相关的命令实现，`channels/` 包含通道管理命令，`models/` 包含模型配置命令，`onboard-non-interactive/` 和 `onboarding/` 包含向导流程命令。每个命令实现都遵循统一的模式：导出一个异步函数作为命令处理函数，接受参数和运行时环境，执行操作并输出结果。

路由匹配层提供了命令路由的能力，支持将特定路径的请求路由到对应的处理函数。这在处理复合命令时特别有用，如 `openclaw memory status` 需要路由到 `memory` 命令组下的 `status` 子命令。路由机制在 `command-registry.ts` 中实现，通过 `RouteSpec` 类型定义了匹配规则和处理函数。

### 7.2.2 命令注册机制详解

命令注册机制是 Commands 模块的核心机制之一，它负责将命令实现与 CLI 接口连接起来。注册机制的核心是 `CommandRegistration` 类型，它定义了一个命令注册项的结构，包括标识符和注册函数。命令注册表 `commandRegistry` 是一个数组，包含了所有已注册的命令项。

```typescript
// 命令注册项类型定义
type CommandRegistration = {
  id: string;                              // 命令唯一标识
  register: (params: CommandRegisterParams) => void;  // 注册函数
  routes?: RouteSpec[];                    // 可选的路由规则
};

// 命令注册表示例
export const commandRegistry: CommandRegistration[] = [
  { id: "setup", register: ({ program }) => registerSetupCommand(program) },
  { id: "onboard", register: ({ program }) => registerOnboardCommand(program) },
  { id: "configure", register: ({ program }) => registerConfigureCommand(program) },
  { id: "config", register: ({ program }) => registerConfigCli(program) },
  { id: "maintenance", register: ({ program }) => registerMaintenanceCommands(program) },
  { id: "message", register: ({ program, ctx }) => registerMessageCommands(program, ctx) },
  { id: "memory", register: ({ program }) => registerMemoryCli(program) },
  { id: "agent", register: ({ program, ctx }) => registerAgentCommands(program, ctx) },
  { id: "subclis", register: ({ program, argv }) => registerSubCliCommands(program, argv) },
  { id: "status-health-sessions", register: ({ program }) => registerStatusHealthSessionsCommands(program) },
  { id: "browser", register: ({ program }) => registerBrowserCli(program) },
];
```

注册函数接收一个 `CommandRegisterParams` 参数，包含程序实例、上下文和命令行参数。注册函数负责向程序实例添加新的命令，包括命令名称、描述、选项和处理器。这种设计使得注册逻辑完全解耦，每个命令组可以独立定义自己的注册行为。

路由机制为复合命令提供了支持。通过 `RouteSpec` 类型，可以定义更复杂的命令匹配规则和执行逻辑。路由机制允许命令处理函数访问原始命令行参数，并根据参数内容决定是否处理该请求。这种设计特别适用于需要解析嵌套参数的命令场景。

### 7.2.3 命令实现标准模式

每个命令实现都遵循统一的标准模式，这确保了命令行为的一致性和可维护性。标准模式包括函数签名、参数解析、错误处理和结果输出四个部分。

命令函数的标准签名是 `async function xxxCommand(opts: XxxOptions, runtime: RuntimeEnv): Promise<void>`。第一个参数是命令特定的选项对象，第二个参数是运行时环境，返回 Promise<void> 表示异步执行且不返回具体值。选项对象通常定义为一个 TypeScript 接口，包含命令的所有参数。

```typescript
// 命令实现标准模式示例
export async function setupCommand(
  opts?: { workspace?: string },
  runtime: RuntimeEnv = defaultRuntime,
) {
  // 参数解析
  const desiredWorkspace = typeof opts?.workspace === "string" && opts.workspace.trim()
    ? opts.workspace.trim()
    : undefined;

  // 业务逻辑
  const io = createConfigIO();
  const configPath = io.configPath;
  const existingRaw = await readConfigFileRaw(configPath);
  const cfg = existingRaw.parsed;
  const defaults = cfg.agents?.defaults ?? {};

  const workspace = desiredWorkspace ?? defaults.workspace ?? DEFAULT_AGENT_WORKSPACE_DIR;

  // 结果输出
  if (!existingRaw.exists || defaults.workspace !== workspace) {
    await writeConfigFile(next);
    runtime.log(`Wrote ${formatConfigPath(configPath)}`);
  } else {
    runtime.log(`Config OK: ${formatConfigPath(configPath)}`);
  }
}
```

错误处理是命令实现的重要组成部分。命令应该捕获并处理预期的错误情况，向用户提供有意义的错误信息。对于未预期的错误，应该记录详细信息并抛出异常，让上层处理机制决定如何响应。运行时环境 `runtime` 提供了 `log`、`warn`、`error` 等方法用于输出不同级别的信息。

结果输出通过运行时环境的日志方法完成。命令不应该直接使用 `console.log`，而应该通过 `runtime.log` 输出信息。这样做的好处是可以统一管理输出格式，并且支持不同环境下的输出重定向。某些命令还支持 `--json` 选项，以结构化格式输出机器可读的结果。

## 7.3 命令分类详解

### 7.3.1 初始化与配置类命令

初始化与配置类命令是用户首次使用 OpenClaw 时最常用的命令，它们负责系统的基础配置和初始设置。这类命令包括 setup、onboard、configure、config 等，它们共同构成了 OpenClaw 的配置管理层。

setup 命令是系统初始化的核心命令，负责创建默认配置文件和工作区目录。当用户首次运行 `openclaw setup` 时，该命令会执行以下操作：首先检查配置文件是否存在，如果不存在则创建默认配置；然后验证工作区目录，如果不存在则创建；最后确保会话目录也存在。setup 命令支持 `--workspace` 选项，允许用户指定自定义的工作区目录。

```bash
# 默认初始化
openclaw setup

# 指定工作区
openclaw setup --workspace /path/to/workspace
```

onboard 命令启动交互式配置向导，引导用户完成更复杂的初始配置过程。向导会依次询问用户要配置本地 Gateway 还是连接远程 Gateway，选择通讯平台，配置认证信息等。onboard 命令支持多种模式，包括交互模式和非交互模式，可以通过 `--non-interactive` 选项禁用交互提示。

```bash
# 交互式向导
openclaw onboard

# 非交互模式，连接到远程 Gateway
openclaw onboard --mode remote --remote-url ws://gateway.example.com
```

configure 命令启动配置向导，允许用户在已安装的系统上进行配置修改。它可以配置模型、认证、通道等各个方面。configure 命令是一个复合命令，包含多个子命令：`configure auth` 配置认证信息，`configure models` 配置模型设置，`configure channels` 配置通讯平台等。

config 命令提供直接编辑配置文件的能力。`openclaw config edit` 打开默认配置文件进行编辑，`openclaw config show` 显示当前配置内容。这个命令适合高级用户直接操作配置文件，绕过向导的引导。

### 7.3.2 Agent 管理类命令

Agent 管理类命令负责 Agent 的创建、配置、监控和管理。这类命令位于 `src/commands/agent/` 和 `src/commands/agents.*` 文件中，提供了完整的 Agent 生命周期管理能力。

agent 命令是 Agent 管理的核心入口，它本身是一个复合命令，包含多个子命令。`openclaw agent send` 向指定 Agent 发送消息并获取响应，这是在 CLI 中与 Agent 交互的主要方式。`openclaw agent spawn` 创建新的 Agent 会话。`openclaw agent list` 列出所有 Agent。`openclaw agent delete` 删除指定的 Agent。

```bash
# 发送消息给默认 Agent
openclaw agent send "Hello, help me debug this issue"

# 指定 Agent 发送消息
openclaw agent send --agent myagent "What's the weather?"

# 列出所有 Agent
openclaw agents list

# 添加新 Agent
openclaw agents add myagent --model gpt-4

# 删除 Agent
openclaw agents delete myagent
```

agents 命令组提供更高级的 Agent 管理功能。`agents add` 添加新的 Agent 配置，`agents delete` 删除 Agent 配置，`agents list` 列出所有 Agent，`agents identity` 设置 Agent 的身份标识。这些命令操作的是 Agent 的配置元数据，而不是运行时的会话状态。

### 7.3.3 通道管理类命令

通道管理类命令负责通讯平台的添加、配置和状态查看。OpenClaw 支持多种通讯平台，包括 WhatsApp、Telegram、Slack、Discord、iMessage 等，通道管理命令提供了统一的管理接口。

channels 命令是通道管理的核心入口。`channels add` 添加新的通讯平台配置，这会启动交互式向导引导用户完成认证和配置流程。`channels list` 列出所有已配置的通道及其状态。`channels remove` 删除指定的通道配置。`channels status` 查看通道的详细运行状态。`channels capabilities` 查询平台支持的功能特性。`channels logs` 查看通道的运行日志。`channels resolve` 解析通道目标标识。

```bash
# 添加新通道
openclaw channels add telegram

# 列出所有通道
openclaw channels list

# 查看通道状态
openclaw channels status --deep

# 删除通道
openclaw channels remove telegram:myaccount
```

每个通道添加流程的具体步骤因平台而异。以 Telegram 为例，添加流程会要求用户提供 Bot Token，然后验证 Token 的有效性，最后保存配置并启动通道服务。添加完成后，通道会自动连接到 Telegram 服务器，开始接收消息。

### 7.3.4 模型管理类命令

模型管理类命令负责 AI 模型的配置、认证和状态查看。OpenClaw 支持多种模型提供商，包括 OpenAI、Anthropic、Google、GitHub Copilot 等，模型管理命令提供了统一的管理接口。

models 命令是模型管理的核心入口。`models list` 列出所有已配置的模型，显示每个模型的提供商、模型 ID 和认证状态。`models set` 设置默认模型或为特定提供商设置模型。`models scan` 扫描可用的模型。`models auth` 管理认证信息。`models status` 查看模型的运行状态。`models aliases` 管理模型别名。`models fallbacks` 管理模型回退配置。

```bash
# 列出所有模型
openclaw models list

# 设置默认模型
openclaw models set openai:gpt-4

# 扫描可用模型
openclaw models scan

# 配置认证
openclaw models auth openai

# 管理模型别名
openclaw models aliases add mymodel openai:gpt-4
openclaw models aliases list
```

认证管理是模型配置的重要组成部分。`models auth add` 添加新的认证凭证，`models auth login` 启动交互式登录流程，`models auth order` 设置认证优先级。OpenClaw 支持多种认证方式，包括 API Key、OAuth、OAuth Device Flow 等，不同的提供商支持不同的认证方式。

### 7.3.5 系统维护类命令

系统维护类命令用于系统诊断、健康检查、状态监控和故障排除。这类命令对于运维管理至关重要，帮助用户快速定位和解决问题。

health 命令检查 OpenClaw 系统的整体健康状态。它会检查 Gateway 服务运行状态、通道连接状态、内存使用情况等。`openclaw health` 输出简洁的健康摘要，`openclaw health --verbose` 输出详细信息，`openclaw health --json` 输出机器可读的 JSON 格式结果。

```bash
# 基础健康检查
openclaw health

# 详细输出
openclaw health --verbose

# JSON 格式输出（适合脚本处理）
openclaw health --json
```

status 命令查看系统的详细状态信息。`openclaw status` 显示 Gateway 和通道的状态概览，`openclaw status --all` 显示所有组件的详细状态，`openclaw status --usage` 显示资源使用情况，`openclaw status --deep` 进行深度检查。这个命令通常用于故障排除和状态诊断。

sessions 命令管理 Agent 会话。`openclaw sessions --store all --active` 列出所有活动会话，`openclaw sessions --store myagent --active false` 查看已结束的会话历史。

doctor 命令是系统诊断的终极工具，它会运行一系列检查来发现潜在问题。检查内容包括配置完整性、通道状态、认证有效性、磁盘空间、安全设置等。如果发现问题，doctor 会提供修复建议。

```bash
# 运行诊断检查
openclaw doctor

# 详细输出
openclaw doctor --verbose

# 修复模式（尝试自动修复）
openclaw doctor --fix
```

### 7.3.6 开发调试类命令

开发调试类命令主要用于开发测试和高级调试场景。这类命令包括浏览器控制、节点操作、Gateway 调试等。

browser 命令提供浏览器控制能力，用于测试浏览器相关的功能。`openclaw browser inspect` 打开浏览器检查器，`openclaw browser manage` 管理浏览器配置。这些命令主要用于开发测试，不建议在生产环境使用。

nodes 命令提供节点操作能力，用于管理远程节点。`openclaw nodes status` 查看节点状态，`openclaw nodes invoke` 在节点上执行命令，`openclaw nodes canvas` 控制节点的画布功能。

gateway 命令提供 Gateway 直接操作能力。`openclaw gateway call` 调用 Gateway API 方法，`openclaw gateway dev` 以开发模式启动 Gateway，`openclaw gateway discover` 发现网络中的 Gateway 服务。

## 7.4 内部调用机制

### 7.4.1 Commands 作为公共 API

Commands 模块的独特之处在于它不仅是对外的 CLI 接口，还是 OpenClaw 内部的公共 API。许多内部模块调用 Commands 提供的函数来完成各自的工作，这种设计实现了代码复用和功能一致性。

内部调用的典型模式是导入命令函数，传入选项参数和运行时环境，然后等待执行完成。由于命令函数返回 Promise，调用代码可以使用 `await` 或 `.then()` 来处理异步结果。这种模式与 CLI 调用完全一致，确保了行为的一致性。

```typescript
// 内部调用示例：Wizard 向导中调用 healthCommand
import { healthCommand } from "../commands/health.js";

export async function finalizeOnboardingWizard(options: FinalizeOnboardingOptions) {
  // ... 前置逻辑 ...

  // 在向导最后验证系统健康状态
  await healthCommand({ json: false, timeoutMs: 10_000 }, runtime);

  // ... 后续逻辑 ...
}
```

### 7.4.2 Wizard 向导流程中的 Commands 调用

Wizard 向导是 OpenClaw 的交互式配置系统，它大量复用 Commands 模块的逻辑来完成配置任务。向导流程的设计原则是"只做一次"，即在初始配置时完成所有必要的设置，配置完成后用户就可以正常使用了。

在 `onboarding.finalize.ts` 中，向导的最后阶段调用 `healthCommand` 来验证系统配置是否正确。这个调用确保了用户在完成向导后，系统已经处于健康运行状态。如果健康检查失败，向导会给出相应的错误提示和修复建议。

在 `onboard-non-interactive/local.ts` 中，非交互式本地配置流程同样调用 `healthCommand` 来验证 Gateway 服务状态。这种模式确保了无论用户使用交互式还是非交互式配置，系统都会执行相同的验证逻辑。

### 7.4.3 Doctor 诊断工具中的 Commands 调用

Doctor 诊断工具是 OpenClaw 的系统健康检查工具，它调用多个 Commands 函数来执行全面的系统诊断。Doctor 的设计原则是"发现问题并提供解决方案"，它会尽可能自动修复问题，或者提供详细的修复指导。

在 `doctor-gateway-health.ts` 中，`checkGatewayHealth` 函数调用 `healthCommand` 来检查 Gateway 服务状态。这个函数被多个 Doctor 检查流程复用，确保了一致的健康检查逻辑。

在 `doctor-gateway-daemon-flow.ts` 中，Doctor 的 Gateway 服务检查流程调用 `healthCommand` 来验证服务是否正常运行。如果服务未运行，Doctor 会提示用户启动服务。

在 `configure.wizard.ts` 中，配置向导在关键步骤调用 `healthCommand` 来验证配置变更是否成功。这种即时验证机制帮助用户在配置过程中及时发现问题。

### 7.4.4 设计优势与复用模式

Commands 模块的双重定位带来了显著的设计优势。首先是代码复用，Wizard、Doctor、Configure 等模块复用 Commands 的逻辑，避免了代码重复。其次是一致性保证，内部调用和 CLI 调用使用相同的函数，确保了行为的一致性。第三是可测试性，作为独立函数的 Commands 更容易进行单元测试。第四是可维护性，业务逻辑集中在 Commands 模块中，变更影响范围可控。

复用模式遵循"纯函数加依赖注入"的原则。每个命令函数只接收必要的参数和运行时环境，不依赖全局状态。这种设计使得命令函数可以在任何上下文中执行，只需要提供正确的参数和环境。

```typescript
// 命令函数的依赖注入模式
export async function healthCommand(
  opts: HealthCommandOptions,
  runtime: RuntimeEnv,
): Promise<void> {
  // 依赖通过参数传入，不使用全局状态
  const gatewayDetails = buildGatewayConnectionDetails({ config: opts.config });
  const healthResult = await checkGatewayHealth({
    runtime,
    cfg: opts.config,
    timeoutMs: opts.timeoutMs,
  });

  // 业务逻辑
  if (healthResult.healthOk) {
    runtime.log("Gateway is healthy");
  } else {
    runtime.error("Gateway is not healthy");
  }
}
```

## 7.5 命令执行流程深度解析

### 7.5.1 命令解析与分发

命令执行流程从用户输入开始，经过解析、分发、执行和输出四个阶段。解析阶段由 Commander.js 框架处理，它解析命令行参数并构建参数对象。分发阶段将参数对象传递给对应的命令处理函数。执行阶段运行命令的业务逻辑。输出阶段通过运行时环境输出结果。

解析阶段的关键是参数规范化。CLI 工具使用 `src/cli/argv.ts` 中的辅助函数来解析命令行参数，包括 `hasFlag` 检查标志位，`getFlagValue` 获取字符串值，`getPositiveIntFlagValue` 获取正整数值，`getVerboseFlag` 获取详细输出标志。这些函数处理了各种边界情况，如参数缺失、类型错误、值越界等。

分发阶段由 Commander.js 框架自动处理。每个注册的命令都有对应的处理器函数，当用户输入匹配该命令时，框架会自动调用处理器并传入解析后的参数对象。处理器函数接收的参数结构由注册时定义的选项决定。

### 7.5.2 运行时环境的作用

运行时环境 `RuntimeEnv` 是命令执行的重要上下文，它提供了访问配置、日志、系统资源的能力。所有命令函数都接收 `runtime` 参数，这是命令与系统交互的主要接口。

运行时环境的核心职责包括配置访问、日志输出、资源管理。配置访问通过 `runtime.config` 提供，命令可以读取当前的系统配置。日志输出通过 `runtime.log`、`runtime.warn`、`runtime.error` 方法实现，这些方法会格式化输出信息并添加时间戳。资源管理通过 `runtime.env`、`runtime.cwd` 等属性提供访问系统环境和工作目录的能力。

```typescript
// RuntimeEnv 类型定义（简化）
type RuntimeEnv = {
  config: OpenClawConfig;      // 系统配置
  log: (msg: string) => void;  // 信息日志
  warn: (msg: string) => void; // 警告日志
  error: (msg: string) => void; // 错误日志
  env: Record<string, string>; // 环境变量
  cwd: () => string;           // 工作目录
  // ... 其他属性和方法
};
```

### 7.5.3 错误处理与结果输出

错误处理是命令执行的重要组成部分。命令应该区分预期错误和未预期错误，分别采用不同的处理方式。预期错误是指业务流程中可能出现的错误情况，如配置无效、资源不存在、认证失败等，这些错误应该被捕获并转换为用户友好的错误信息。未预期错误是指程序缺陷导致的错误，如空指针、类型错误等，这些错误应该记录详细上下文并抛出，让上层处理机制决定如何响应。

结果输出通过运行时环境的日志方法完成。命令应该输出一致的格式化信息，包括操作结果、关键数据、建议操作等。对于支持 JSON 输出的命令，输出应该包含完整的结构化数据，便于脚本处理。

```typescript
// 错误处理与结果输出示例
export async function channelsListCommand(
  opts: ChannelsListOptions,
  runtime: RuntimeEnv,
): Promise<void> {
  try {
    // 执行业务逻辑
    const channels = await listChannels(runtime.config);

    if (opts.json) {
      // JSON 格式输出
      console.log(JSON.stringify({ channels }, null, 2));
    } else {
      // 格式化表格输出
      for (const channel of channels) {
        runtime.log(`${channel.id.padEnd(20)} ${channel.status.padEnd(10)} ${channel.account}`);
      }
    }
  } catch (err) {
    // 预期错误：转换为友好信息
    if (err instanceof ChannelNotFoundError) {
      runtime.error(`Channel not found: ${err.channelId}`);
      return;
    }

    // 未预期错误：记录并抛出
    runtime.error(`Unexpected error: ${String(err)}`);
    throw err;
  }
}
```

## 7.6 与 Plugin/Skills 的关系

### 7.6.1 Commands 的独立性

Commands 模块与 Plugin/Skills 是完全独立的系统，它们服务于不同的目的和用户群体。Plugin 扩展 Agent 的能力，Skills 指导 Agent 的行为，而 Commands 管理系统的运维操作。这三个系统是正交的，没有直接的依赖关系。

Plugin 系统的注册函数通过 `registerTool`、`registerHook` 等 API 注入能力，这些 API 与 Commands 完全不同。Skills 系统的 SKILL.md 文件由专门的加载器解析，与 Commands 的命令行解析也没有关系。

Commands 模块对 Plugin/Skills 一无所知，它专注于自己的职责：提供运维操作的 CLI 接口和公共 API。这种设计遵循了单一职责原则，每个系统只做一件事并做好。

### 7.6.2 间接关系：Commands 对 Plugin 的管理

虽然 Commands 与 Plugin 是独立的系统，但 Commands 提供了管理 Plugin 的命令。通过 `openclaw plugins` 命令组，用户可以查看、安装、更新、禁用插件。这些命令操作 Plugin 的配置和文件系统，但它们不直接调用 Plugin 的注册函数。

```bash
# 插件管理命令
openclaw plugins list          # 列出插件
openclaw plugins install <id>  # 安装插件
openclaw plugins remove <id>   # 移除插件
openclaw plugins update        # 更新插件
openclaw plugins disable <id>  # 禁用插件
openclaw plugins enable <id>   # 启用插件
```

插件管理命令的实现位于 `src/cli/plugins-cli.ts`，它们调用 `src/plugins/` 目录下的模块来完成实际的插件操作。这些操作包括插件发现、Manifest 解析、文件复制、配置更新等。

## 7.7 最佳实践与设计模式

### 7.7.1 命令设计原则

设计新的命令时应该遵循以下原则。首先是单一职责，每个命令只做一件事，避免命令功能过于复杂。如果需要复合功能，应该拆分为多个子命令。其次是接口一致，所有命令使用相同的参数模式：`async function xxxCommand(opts: XxxOptions, runtime: RuntimeEnv): Promise<void>`。

第三是错误友好，命令应该提供清晰的错误信息和修复建议。错误信息应该指出问题所在和可能的解决方案，而不是仅仅显示错误代码。第四是输出可控，支持 `--json` 选项输出机器可读的结果，同时提供人类可读的默认输出。

```typescript
// 良好的命令设计示例
export interface MemoryFlushOptions {
  agent?: string;     // 可选，指定 Agent
  force?: boolean;    // 可选，强制刷新
  json?: boolean;     // 可选，JSON 输出
}

export async function memoryFlushCommand(
  opts: MemoryFlushOptions,
  runtime: RuntimeEnv,
): Promise<void> {
  // 参数验证
  if (!opts.agent && !opts.force) {
    runtime.error("Please specify --agent or use --force");
    return;
  }

  try {
    // 业务逻辑
    const result = await flushMemory({
      agent: opts.agent,
      force: opts.force,
    });

    // 输出结果
    if (opts.json) {
      console.log(JSON.stringify(result, null, 2));
    } else {
      runtime.log(`Flushed ${result.count} memories for ${result.agent}`);
    }
  } catch (err) {
    runtime.error(`Failed to flush memory: ${String(err)}`);
    throw err;
  }
}
```

### 7.7.2 可复用命令的实现

当某个运维操作需要在多个上下文中使用时，应该将其实现为可复用的命令函数。这样既可以在 CLI 中使用，也可以在内部模块中复用。可复用命令的实现应该避免依赖特定的调用上下文，只依赖传入的参数和运行时环境。

```typescript
// 可复用命令函数的实现模式
export async function checkGatewayHealth(params: {
  runtime: RuntimeEnv;
  cfg: OpenClawConfig;
  timeoutMs?: number;
}): Promise<{ healthOk: boolean }> {
  const timeoutMs = params.timeoutMs ?? 10_000;
  let healthOk = false;

  try {
    await healthCommand({ json: false, timeoutMs, config: params.cfg }, params.runtime);
    healthOk = true;
  } catch (err) {
    const message = String(err);
    if (message.includes("gateway closed")) {
      params.runtime.note("Gateway not running.", "Gateway");
    } else {
      params.runtime.error(formatHealthCheckFailure(err));
    }
  }

  return { healthOk };
}
```

### 7.7.3 测试策略

Commands 模块的测试策略包括单元测试和集成测试两个层面。单元测试验证命令函数的业务逻辑，使用 Mock 模拟运行时环境和其他依赖。集成测试验证 CLI 接口的完整流程，使用真实的命令行参数调用程序。

单元测试使用 Vitest 框架，每个命令都有对应的测试文件。测试用例覆盖正常流程、边界条件、错误处理等场景。Mock 对象使用 `vi.mock` 创建，模拟运行时环境和外部服务。

集成测试在 `src/cli/program*.test.ts` 文件中，模拟完整的命令行调用流程。这些测试验证命令注册、参数解析、结果输出的完整性。

## 7.8 总结与扩展指南

### 7.8.1 Commands 模块核心要点

Commands 模块的核心要点可以总结为以下几点。第一，它是运维操作层的抽象，将所有系统管理功能封装为独立的命令函数。第二，它具有双重接口，既是 CLI 工具，又是内部公共 API。第三，它遵循统一的设计模式，包括参数模式、错误处理、结果输出等。第四，它与 Plugin/Skills 完全独立，通过配置管理间接关联。第五，它支持内部复用，Wizard、Doctor 等模块调用 Commands 函数完成各自的工作。

### 7.8.2 添加新命令的步骤

添加新命令需要遵循以下步骤。首先在 `src/commands/` 目录下创建命令实现文件，定义选项接口和命令函数。其次在 `src/cli/program/` 目录下创建注册器文件，注册命令到 CLI 程序。最后在 `command-registry.ts` 中注册命令注册器。

命令实现文件应该包含完整的业务逻辑，支持所有必要的参数和错误处理。注册器文件应该定义命令的名称、描述、选项和处理函数。命令注册表更新确保新命令被正确注册和分发。

### 7.8.3 与其他模块的协作关系

Commands 模块与以下模块有密切的协作关系。与 `config/` 模块协作读取和写入系统配置。与 `runtime/` 模块协作获取运行时环境。与 `gateway/` 模块协作调用 Gateway API。与 `channels/` 模块协作管理通讯平台。与 `agents/` 模块协作管理 Agent 配置。与 `memory/` 模块协作管理记忆存储。

这些协作关系通过参数传递和函数调用实现，Commands 模块不直接依赖其他模块的具体实现，而是通过接口抽象进行交互。这种设计确保了模块之间的松耦合和可测试性。
