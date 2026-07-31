---
read_when:
    - 你正在更改嵌入式 Agent 运行时或 harness 注册表
    - 你正在从内置或可信插件注册 Agent harness
    - 你需要了解 Codex 插件与模型提供商之间的关系
sidebarTitle: Agent Harness
summary: 供替换底层嵌入式智能体执行器的插件使用的实验性 SDK 接口面
title: Agent harness plugins
x-i18n:
    generated_at: "2026-07-26T06:58:20Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 4ff4e41a46ba0074fc6c8bf46da813b58d074f5e6c5c1d236d7ab78e824bdc02
    source_path: plugins/sdk-agent-harness.md
    workflow: 16
---

**Agent harness** 是单次已准备好的 OpenClaw 智能体轮次的底层执行器。它不是模型提供商，不是渠道，也不是工具注册表。有关面向用户的心智模型，请参阅 [Agent Runtimes](/zh-CN/concepts/agent-runtimes)。

此接口仅供内置或受信任的原生插件使用。该契约仍处于实验阶段，因为其参数类型有意与当前嵌入式运行器保持一致。

## 何时使用 harness

当某个模型系列拥有自己的原生会话运行时，并且常规 OpenClaw 提供商传输并非合适的抽象时，请注册 agent harness：

- 拥有线程和压缩功能的原生编码智能体服务器
- 必须流式传输原生计划、推理和工具事件的本地 CLI 或守护进程
- 除 OpenClaw 会话记录外，还需要自身恢复 ID 的模型运行时

请**勿**仅为了添加新的 LLM API 而注册 harness。对于常规 HTTP 或 WebSocket 模型 API，请构建[提供商插件](/zh-CN/plugins/sdk-provider-plugins)。

## 核心仍负责的内容

在选择 harness 之前，OpenClaw 已经解析了：

- 提供商和模型
- 运行时身份验证状态，除非 harness 声明由其负责身份验证引导
- 思考级别和上下文预算
- OpenClaw 会话记录/会话文件
- 工作区、沙箱和工具策略
- 渠道回复回调和流式传输回调
- 模型回退和实时模型切换策略

harness 运行已准备好的尝试；它不会选择提供商、取代渠道投递或静默切换模型。

### Harness 负责的身份验证引导

默认情况下，核心会在调用 harness 前解析提供商凭据。能够通过自身原生运行时进行身份验证的受信任 harness，可以在其静态 `AgentHarness` 注册中设置 `authBootstrap: "harness"`。随后，对于该 harness 声明处理的每次尝试，核心都会跳过通用提供商凭据引导和凭据缺失失败。

当存在兼容且已显式选择或排序的 OpenClaw 身份验证配置文件及其限定范围的存储时，核心仍会将其转发。harness 必须在发出模型请求前解析该配置文件或其原生凭据，将密钥的作用域限制在本次尝试内，并提供可操作的身份验证失败信息。如果 harness 只在部分情况下负责身份验证，请勿设置此能力。

### 已验证的设置运行时工件

能够为首次运行设置提供推理能力的本地 harness，必须证明完成探测的实现。当 `params.captureRuntimeArtifact` 为 true 时，返回具有稳定 ID 和内容指纹的不透明 `result.runtimeArtifact`。注册匹配的 `runtimeArtifact.validate(...)` 能力，以便在不加载其他 harness 或扫描无关插件的情况下重新检查该绑定。

已验证的 OpenClaw 延续操作还会传入 `params.expectedRuntimeArtifact`。harness 必须将其与实际获取的原生进程进行比较；如果二者不同，必须在启动或恢复原生线程前失败。普通智能体轮次会省略这两个字段，因此内容哈希不会进入正常请求的热路径。远程/WebSocket harness 必须先具备服务器证明契约才能参与；仅有版本字符串不足以构成工件身份。

已准备好的尝试还包括 `params.runtimePlan`，这是 OpenClaw 所有的策略包，用于处理必须在 OpenClaw 与原生 harness 之间保持一致的运行时决策：

- `runtimePlan.tools.normalize(...)` 和 `runtimePlan.tools.logDiagnostics(...)`，用于感知提供商的工具 schema 策略
- `runtimePlan.transcript.resolvePolicy(...)`，用于会话记录清理和工具调用修复策略
- `runtimePlan.delivery.isSilentPayload(...)`，用于共享 `NO_REPLY` 和媒体投递抑制
- `runtimePlan.outcome.classifyRunResult(...)`，用于模型回退分类
- `runtimePlan.observability`，用于已解析的提供商/模型/harness 元数据

harness 可以使用该计划做出需要与 OpenClaw 行为一致的决策，但应将其视为主机所有的尝试状态：请勿修改它，也不要在轮次内使用它切换提供商/模型。

### 请求传输契约

`supports(ctx)` 通过 `ctx.modelProvider` 接收已解析的模型传输。以下两个不含密钥且由提供商所有的事实描述了所选路由：

- `runtimePolicy.compatibleIds` 列出提供商声明与该具体路由兼容的运行时 ID。缺少策略意味着提供商未声明路由级兼容性；这并不表示可以假定支持。
- `requestTransportOverrides: "none"` 表示无需重现人为配置的提供商/模型请求覆盖。`"present"` 表示存在人为配置的请求头、身份验证传输、代理、TLS、本地服务、私有网络行为或请求参数。该事实不会暴露这些值。

当 harness 无法重现已准备好的传输时，返回 `{ supported: false, reason }`。选择完成后，请勿通过读取原始配置推断支持情况。当身份验证准备产生多个重试路由时，一个 harness 必须支持所有路由才能分派。如果没有插件能够负责完整集合，隐式选择将使用 OpenClaw；显式或持久化的插件选择则会按失败关闭原则处理。

## 注册 harness

**导入：** `openclaw/plugin-sdk/agent-harness`

```typescript
import type { AgentHarness } from "openclaw/plugin-sdk/agent-harness";
import { definePluginEntry } from "openclaw/plugin-sdk/plugin-entry";

const myHarness: AgentHarness = {
  id: "my-harness",
  label: "我的原生 agent harness",

  supports(ctx) {
    const routeSupportsHarness =
      ctx.modelProvider?.runtimePolicy?.compatibleIds.includes("my-harness") === true;
    const canReproduceRequest = ctx.modelProvider?.requestTransportOverrides !== "present";
    return ctx.provider === "my-provider" && routeSupportsHarness && canReproduceRequest
      ? { supported: true, priority: 100 }
      : { supported: false, reason: "有效路由与 harness 不兼容" };
  },

  async runAttempt(params) {
    // 启动或恢复你的原生线程。
    // 使用 params.prompt、params.tools、params.images、params.onPartialReply、
    // params.onAgentEvent 以及其他已准备好的尝试字段。
    return await runMyNativeTurn(params);
  },
};

export default definePluginEntry({
  id: "my-native-agent",
  name: "我的原生智能体",
  description: "通过原生智能体守护进程运行所选模型。",
  register(api) {
    api.registerAgentHarness(myHarness);
  },
});
```

此通用示例有意省略 `authBootstrap`。仅当 harness 满足上述契约时，才添加 `authBootstrap: "harness"`。

### 委托执行

harness 所有者可以将 `delegatedExecutionPluginIds` 设置为需要执行现有模型锁定会话的受信任插件 ID，例如继续由 Codex 支持的对话的语音传输。这是所有者的静态同意，而不是核心允许列表。请将其范围保持在最小限度。

委托方仅获得工作准入和嵌入式执行权限。OpenClaw 要求提供完全一致的已存储会话键、存储路径和会话 ID；`modelSelectionLocked:
true`；以及匹配的 `agentHarnessId` 和 `agentHarnessRuntimeOverride` 值。随后，运行会通过 harness 所有者限定作用域。会话创建、修补、重置、删除、归档和 Gateway 网关变更仍仅限所有者执行。

## 选择策略

OpenClaw 在解析提供商/模型后选择 harness：

1. 模型范围的运行时策略优先。
2. 其次是提供商范围的运行时策略。
3. `auto` 会询问已注册的 harness 是否支持已解析的有效路由。仅凭提供商/模型前缀绝不会选择 harness。
4. 如果没有匹配的已注册 harness，OpenClaw 将使用其嵌入式运行时。

插件 harness 失败会显示为运行失败。在 `auto` 模式下，仅当没有已注册的插件 harness 支持已解析的提供商/模型时，才会应用嵌入式回退。一旦某个插件 harness 声明处理一次运行，OpenClaw 就不会通过其他运行时重放同一轮次，因为这可能改变身份验证/运行时语义或产生重复的副作用。

已配置的运行时策略始终是所需运行时的权威来源。在路由/身份验证准备仍待完成时，持久化会话的 `agentHarnessId` 会保留其原生会话记录的所有权。两者都无法让不兼容的路由变得兼容：一旦准备好的事实存在，所选或固定的 harness 就必须支持它们，否则运行将按失败关闭原则处理。`/status` 显示根据策略、持久化所有权和路由支持情况选择的有效运行时。
准备状态是显式的：缺少 `runtimePolicy` 时，状态会保持未声明，而不是根据碰巧存在的传输字段进行推断。
当 harness 所有的身份验证仍有多个物理路由未解析时，准备好的支持事实是这些路由兼容运行时 ID 的交集；如果任一候选路由具有请求覆盖，也会进行报告。因此，一个未声明的候选路由就会使原生兼容性为空；`preparedAuth.source: "harness"` 是身份验证所有者，并不表示可以推断路由支持。

如果所选 harness 出乎预期，请启用 `agents/harness` 调试日志，并检查 Gateway 网关的结构化 `agent harness selected` 记录：其中包括所选 harness ID、选择原因、运行时/回退策略，以及在 `auto` 模式下各插件候选项的支持结果。

内置 Codex 插件将 `codex` 注册为其 harness ID。核心将其视为普通的插件 harness ID；Codex 专用别名应位于插件或操作员配置中，而不是共享运行时选择器中。

## 提供商与 harness 配对

大多数 harness 还应注册提供商。提供商会让模型引用、身份验证状态、模型元数据和 `/model` 选择对 OpenClaw 的其余部分可见。随后，harness 在 `supports(...)` 中声明处理该提供商。

内置 Codex 插件遵循此模式：

- 首选用户模型引用：`openai/gpt-5.6-sol`
- 兼容性引用：仍接受旧版 `codex/gpt-*` 引用，但新配置不应将其用作常规模型提供商/模型引用
- harness ID：`codex`
- 身份验证：合成的提供商可用性，因为 Codex harness 负责原生 Codex 登录/会话
- app-server 请求：OpenClaw 将裸模型 ID 发送给 Codex，并由 harness 与原生 app-server 协议通信

Codex 插件是增量式的。在未设置运行时策略或策略为 `auto` 时，仅当 OpenAI 由提供商所有的路由契约声明 `codex` 兼容时，OpenAI 才可能选择 Codex：即完全匹配的官方 HTTPS Platform Responses 或 ChatGPT Responses 路由，且不存在人为配置的请求覆盖。仅凭 `openai/*` 前缀绝不会选择 Codex。自定义端点、Completions 适配器和人为配置的请求行为仍由 OpenClaw 处理。官方明文 HTTP 端点会被拒绝。旧版 `codex/gpt-*` 引用仍作为兼容性输入。请参阅
[OpenAI 隐式 agent runtime](/zh-CN/providers/openai#implicit-agent-runtime)。

有关操作员设置、模型前缀示例和仅使用 Codex 的配置，请参阅
[Codex harness](/zh-CN/plugins/codex-harness)。

Codex 插件会强制执行 [Codex harness](/zh-CN/plugins/codex-harness) 中记录的最低 app-server 版本。它会检查初始化握手并阻止旧版或无版本信息的服务器，从而确保 OpenClaw 仅针对其已测试的协议接口运行。

### 工具结果中间件

当内置插件和显式启用的已安装插件具有匹配的清单契约时，如果其清单在 `contracts.agentToolResultMiddleware` 中声明了目标运行时 ID，它们便可通过 `api.registerAgentToolResultMiddleware(...)` 附加与运行时无关的工具结果中间件。此受信任接口用于必须在 OpenClaw 或 Codex 将工具输出反馈给模型前运行的异步工具结果转换。

旧版内置插件仍可将
`api.registerCodexAppServerExtensionFactory(...)` 用于仅限 Codex app-server 的
中间件，但新的结果转换应使用运行时中立 API。仅限嵌入式运行器的 `api.registerEmbeddedExtensionFactory(...)` 钩子已被
移除；嵌入式工具结果转换必须使用运行时中立中间件。

### 终止结果分类

拥有自身协议投影的原生 harness 可以在已完成的轮次未产生
可见智能体文本时，使用
`openclaw/plugin-sdk/agent-harness-runtime` 中的
`classifyAgentHarnessTerminalOutcome(...)`。该辅助函数返回 `empty`、`reasoning-only` 或
`planning-only`，以便 OpenClaw 的回退策略决定是否使用
其他模型重试。`planning-only` 需要 harness 提供明确的 `planText`
字段；OpenClaw 不会根据智能体文本推断该字段。该辅助函数
有意不对提示错误、进行中的轮次以及 `NO_REPLY` 等刻意静默的
回复进行分类。

### 智能体结束时的副作用

原生 harness 必须在最终确定一次尝试后，调用
`openclaw/plugin-sdk/agent-harness-runtime` 中的 `runAgentEndSideEffects(...)`。它会
分派可移植的 `agent_end` 钩子和 OpenClaw 的研究捕获，
且不会延迟交互式回复。对于本地非交互式运行，如果必须等到这些
副作用完成后才能结束尝试，请使用 `awaitAgentEndSideEffects(...)`。
两个辅助函数都接受与 `runAgentHarnessAgentEndHook(...)` 相同的 `{ event, ctx }`
载荷；它们发生故障不会改变已完成的尝试结果。

### 用户输入和工具表面

暴露运行时级用户输入请求的原生 harness 应使用
`openclaw/plugin-sdk/agent-harness-runtime` 中的用户输入辅助函数来设置
提示格式、通过 OpenClaw 的阻塞式回复路径发送提示，并将
选择题/自由形式答案规范化为运行时的原生响应结构。该
辅助函数可保持渠道/TUI 呈现一致，同时各 harness 继续负责自身的
协议解析和待处理请求生命周期。

需要类似 PI 的紧凑工具路由的原生 harness 应使用
`openclaw/plugin-sdk/agent-harness-tool-runtime` 中的
`createAgentHarnessToolSurfaceRuntime(...)`。它负责
工具搜索/代码模式控制选择、本地模型精简默认值、
运行时兼容的 schema 过滤、隐藏目录执行、目录
水合和目录清理。Harness 仍负责其 SDK 特定的工具
转换和原生执行回调。

### Native Codex harness 模式

内置的 `codex` harness 是嵌入式 OpenClaw
智能体轮次的原生 Codex 模式。请先启用内置的 `codex` 插件；如果配置使用限制性允许列表，还需在
`plugins.allow` 中加入 `codex`。原生 app-server
配置应使用 `openai/gpt-*`；仅当有效路由声明与 Codex 兼容时，OpenAI 智能体轮次才会选择 Codex harness。
旧版 Codex 模型引用应使用 `openclaw doctor --fix` 修复，而旧版 `codex/*`
模型引用仍作为原生 harness 的兼容性别名。

运行此模式时，Codex 负责原生线程 ID、恢复行为、
压缩和 app-server 执行。OpenClaw 仍负责聊天渠道、
可见转录副本、工具策略、审批、媒体交付和会话
选择。如果需要证明只有 Codex app-server 路径能够接管运行，请使用
提供商/模型 `agentRuntime.id: "codex"`。显式插件
运行时会采用失败关闭策略；Codex app-server 选择失败和运行时失败
不会通过其他运行时重试。

## 运行时严格性

默认情况下，OpenClaw 使用 `auto` 提供商/模型运行时策略：已注册的
插件 harness 可以接管兼容的有效路由，如果没有匹配项，则由嵌入式
运行时处理该轮次。仅凭提供商/模型前缀绝不会
选择 harness。如果缺少 harness 选择时应失败，而不是
通过嵌入式运行时进行路由，请使用显式提供商/模型插件运行时，例如
`agentRuntime.id: "codex"`。显式选择不会使
不兼容的路由变为兼容。所选插件 harness 的故障始终会导致
硬失败。这不会阻止显式提供商/模型
`agentRuntime.id: "openclaw"`。

对于仅限 Codex 的嵌入式运行：

```json
{
  "models": {
    "providers": {
      "openai": {
        "agentRuntime": {
          "id": "codex"
        }
      }
    }
  },
  "agents": {
    "defaults": {
      "model": "openai/gpt-5.6-sol"
    }
  }
}
```

如果希望一个规范模型使用 CLI 后端，请将运行时放在该
模型条目中：

```json
{
  "agents": {
    "defaults": {
      "model": "anthropic/claude-opus-5",
      "models": {
        "anthropic/claude-opus-5": {
          "agentRuntime": {
            "id": "claude-cli"
          }
        }
      }
    }
  }
}
```

每个智能体的覆盖项使用相同的模型作用域结构：

```json
{
  "agents": {
    "list": [
      {
        "id": "codex-only",
        "model": "openai/gpt-5.6-sol",
        "models": {
          "openai/gpt-5.6-sol": {
            "agentRuntime": { "id": "codex" }
          }
        }
      }
    ]
  }
}
```

如下所示的旧版全智能体运行时示例会被忽略：

```json
{
  "agents": {
    "defaults": {
      "agentRuntime": {
        "id": "codex"
      }
    }
  }
}
```

使用显式插件运行时后，如果请求的
harness 未注册、不支持解析后的提供商/模型，或在产生轮次副作用之前
失败，会话将提前失败。这是 Codex 专属
部署和必须证明 Codex app-server 路径确实
正在使用的实时测试的预期行为。

此设置仅控制嵌入式智能体 harness。它不会禁用
图像、视频、音乐、TTS、PDF 或其他提供商特定的模型路由。

## 原生会话和转录副本

Harness 可以保留原生会话 ID、线程 ID 或守护进程端恢复
令牌。请明确地将该绑定与 OpenClaw 会话关联，并
继续将用户可见的智能体/工具输出同步至 OpenClaw
转录。

OpenClaw 转录仍是以下功能的兼容层：

- 渠道可见的会话历史记录
- 转录搜索和索引
- 在后续轮次切换回内置 OpenClaw harness
- 通用 `/new`、`/reset` 和会话删除行为

如果 harness 存储边车绑定，请实现 `reset(...)`，以便 OpenClaw
在重置所属 OpenClaw 会话时将其清除。

## 工具和媒体结果

核心会构建 OpenClaw 工具列表并将其传入已准备的
尝试。当 harness 执行动态工具调用时，应通过 harness 结果结构
返回工具结果，而不是自行发送渠道媒体。

这样可以让文本、图像、视频、音乐、TTS、审批和消息工具
输出使用与 OpenClaw 支持的运行相同的交付路径。

仅对受信任的 harness 运行时自行创建并持久化的原生工件
设置 `AgentHarnessAttemptResult.hostOwnedToolMediaUrls`。每个条目还必须
出现在 `toolMediaUrls` 中。切勿包含模型选择的动态工具或
OpenClaw 工具媒体。在 `message_tool_only` 路由上，这种严格限定的来源证明可使
原生运行时工件在源回复受到抑制时继续保留；常规发送策略
和环境房间准入规则仍然适用。

### 工具终止结果

`AgentHarnessAttemptParams.observeToolTerminal` 是由宿主负责的终止
结果累加器。执行 OpenClaw 动态工具或原生
工具的 harness 必须在每个工具达到一个终止结果时、且在
最终确定尝试结果之前调用它。不执行工具的 harness 无需
调用它。

报告执行边界处的事实：

- 如果存在协议调用 ID，请传入该 ID、规范工具名称，以及
  经准备或钩子重写后实际传递给工具的参数。
- 当验证、审批或其他防护机制
  在工具实现开始前阻止调用时，请设置 `executionStarted: false`。一旦可能已发生分派，
  应保守地报告 `true`。
- 报告 `outcome: "success"` 或 `outcome: "failure"`。请包含运行时提供的结构化
  失败字段，而不是根据显示文本推断失败。
- 仅对未使用 OpenClaw 工具
  定义的原生工具使用 `nativeMutation`。在此提供协议所属的变更和重放事实；不要
  将 OpenClaw 的变更分类器复制到 harness 中。

回调会返回该调用的规范解析结果。将其
`lastToolError` 传入 `AgentHarnessAttemptResult`，并在 harness 投影中使用其执行、
参数和副作用事实，而不是派生
并行状态。宿主会在无关工具成功后继续保留未解析的变更失败，
并且仅在匹配的操作成功后清除它。

为与较旧的实验性 harness 保持源代码兼容，该回调仍是可选的。
但对于执行工具的 harness，“可选”并不意味着可以忽略：
如果没有终止报告，OpenClaw 无法在后续工具调用中保留变更工具失败的真实状态，
其中包括静默完成的 Heartbeat。

### 已结算工具最终处理

当 harness 已完成每个
工具调用，但其原生轮次结束时没有智能体文本，OpenClaw 可能需要一个最终可见答案。Harness 可以通过实现 `finalizeSettledTurn({ attempt,
settledAttempt })`
选择启用该恢复机制。

该回调是一项独立能力，而不是另一项普通尝试。它必须：

- 使用确切的受限原生转录，或完整且冻结至已结算工具结果边界的应用
  转录；
- 不暴露任何工具、权限授予或用户输入能力、原生执行
  钩子、智能体、Skills、记忆、调度、扩展或远程控制；
- 仅发送宿主提供的最终处理提示；并且
- 如果所选的转录/隔离策略无法实施
  这些限制，则采用失败关闭策略。

OpenClaw 会在普通尝试和重试循环之外，将该回调调用一次作为终止子操作。
失败会使运行以
可感知副作用的未完成轮次警告结束；它无法进入普通的
身份验证/配置文件轮换、模型回退、上下文恢复、压缩
延续或钩子请求的修订路径。最终处理还会跳过插件
提示变更、`before_agent_run`、LLM 输入/输出、终止修订和
`agent_end` 钩子。核心诊断仍会记录该操作及其失败。

该回调返回 `AgentHarnessSettledTurnFinalizationResult`，而不是
普通尝试结果。其公共字段仅限于已完成的
智能体消息、最终处理调用用量、转录所有权元数据和
诊断跟踪。工具、交付、媒体、生成、生命周期、重放、会话和
回退状态均不能越过此结果边界。未知字段和智能体
工具调用会采用失败关闭策略。

在内部复用完整尝试引擎的 harness 可以在返回前调用
`projectSettledTurnFinalizationAttemptResult(...)`。该辅助函数
会拒绝规范失败、工具、交付、重放和生命周期证据，然后
仅投影严格限定的结果。它是在原生隔离之后提供的纵深防御，
不能替代移除原生能力表面。

基于投影的 harness 必须将完整上下文放入
`settledAttempt.settledTurnFinalizationContext`，并使用
`source: "openclaw-transcript"`。它必须在已结算轮次完成同步后捕获活动分支，
证明当前提示以及每个当前工具
调用/结果都已包含至该边界，并在返回尝试之前冻结生成的消息
数组。最终处理器必须拒绝缺失、不受支持、有歧义或过大的上下文。
它不得截断消息、
丢弃较早的历史记录，也不得将此应用转录描述为确切的原生
历史记录。恢复单个受限原生会话的 harness 不需要此
投影字段。

请勿通过使用尽力而为的
`disableTools` 提示调用 `runAttempt` 来实现此回调。Harness 所有者必须实施完整的原生
能力边界。OpenClaw 不提供通用回退，因为它
无法证明任意原生运行时遵守了这些限制。

对于实验性第三方 harness 兼容性，该回调仍为可选项。当所选 harness 省略该回调时，OpenClaw 会保留现有的轮次未完成错误，而不是冒着重复产生副作用的风险。

## 当前限制

- 公共导入路径是通用的，但为保持兼容性，某些尝试/结果类型别名仍沿用旧名称。
- 第三方 harness 安装仍处于实验阶段。在需要原生会话运行时之前，优先使用提供商插件。
- 支持在不同轮次之间切换 harness。在原生工具、审批、助手文本或消息发送已经开始后，不要在轮次中途切换 harness。

## 相关内容

- [SDK 概览](/zh-CN/plugins/sdk-overview)
- [运行时辅助函数](/zh-CN/plugins/sdk-runtime)
- [提供商插件](/zh-CN/plugins/sdk-provider-plugins)
- [Codex harness](/zh-CN/plugins/codex-harness)
- [模型提供商](/zh-CN/concepts/model-providers)
