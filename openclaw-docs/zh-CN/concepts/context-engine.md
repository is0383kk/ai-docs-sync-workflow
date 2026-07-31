---
read_when:
    - 你想了解 OpenClaw 如何组装模型上下文
    - 你正在旧版引擎和插件引擎之间切换
    - 你正在构建上下文引擎插件
sidebarTitle: Context engine
summary: 上下文引擎：可插拔的上下文组装、压缩和子智能体生命周期
title: 上下文引擎
x-i18n:
    generated_at: "2026-07-26T06:39:31Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 721780790dacebec44e3c7540b225bd853ee66bf5ae066b84df4344614d93a62
    source_path: concepts/context-engine.md
    workflow: 16
---

**上下文引擎**控制 OpenClaw 如何为每次运行构建模型上下文：包含哪些消息、如何总结较早的历史记录，以及如何跨子智能体边界管理上下文。

OpenClaw 内置 `legacy` 引擎并默认使用它。仅当需要不同的组装、压缩或跨会话回忆行为时，才安装并选择插件引擎。

## 快速开始

<Steps>
  <Step title="检查当前启用的引擎">
    ```bash
    openclaw doctor
    # 或直接检查配置：
    cat ~/.openclaw/openclaw.json | jq '.plugins.slots.contextEngine'
    ```
  </Step>
  <Step title="安装插件引擎">
    上下文引擎插件的安装方式与其他 OpenClaw 插件相同。

    <Tabs>
      <Tab title="从 npm 安装">
        ```bash
        openclaw plugins install @martian-engineering/lossless-claw
        ```
      </Tab>
      <Tab title="从本地路径安装">
        ```bash
        openclaw plugins install -l ./my-context-engine
        ```
      </Tab>
    </Tabs>

  </Step>
  <Step title="启用并选择引擎">
    ```json5
    // openclaw.json
    {
      plugins: {
        slots: {
          contextEngine: "lossless-claw", // 必须与插件注册的引擎 id 匹配
        },
        entries: {
          "lossless-claw": {
            enabled: true,
            // 在此填写插件特有的配置（请参阅插件文档）
          },
        },
      },
    }
    ```

    安装并配置后，重启 Gateway 网关。

  </Step>
  <Step title="切换回旧版引擎（可选）">
    将 `contextEngine` 设置为 `"legacy"`（或完全删除该键——默认值为 `"legacy"`）。
  </Step>
</Steps>

## 工作原理

每当 OpenClaw 运行模型提示词时，上下文引擎都会参与四个生命周期阶段：

<AccordionGroup>
  <Accordion title="1. 摄取">
    在向会话添加新消息时调用。引擎可以将消息存储到自己的数据存储中或为其建立索引。
  </Accordion>
  <Accordion title="2. 组装">
    在每次模型运行前调用。引擎返回符合令牌预算的有序消息集（以及可选的 `systemPromptAddition`）。
  </Accordion>
  <Accordion title="3. 压缩">
    当上下文窗口已满，或用户运行 `/compact` 时调用。引擎总结较早的历史记录以释放空间。
  </Accordion>
  <Accordion title="4. 轮次结束后">
    在运行完成后调用。引擎可以持久化状态、触发后台压缩或更新索引。
  </Accordion>
</AccordionGroup>

引擎还可以实现可选的 `maintain()` 方法，用于在引导启动、轮次成功完成或压缩后维护对话记录（通过 `runtimeContext.rewriteTranscriptEntries()` 安全重写）。设置 `info.turnMaintenanceMode: "background"` 可将其作为延迟任务运行，而不是阻塞回复。

对于内置的非 ACP Codex harness，OpenClaw 通过将组装后的上下文投影到 Codex 开发者指令和当前轮次提示词中，应用相同的生命周期。Codex 仍负责管理其原生线程历史记录和原生压缩器。

### 子智能体生命周期（可选）

OpenClaw 会调用两个可选的子智能体生命周期钩子：

<ParamField path="prepareSubagentSpawn" type="method">
  在子运行开始前准备共享上下文状态。该钩子接收父会话和子会话键、`contextMode`（`isolated` 或 `fork`）、可用的对话记录 id/文件以及可选的 TTL。如果它返回回滚句柄，当准备成功后生成失败时，OpenClaw 会调用该句柄。请求 `lightContext` 且解析为 `contextMode="isolated"` 的原生子智能体生成会有意跳过此钩子，使子智能体从轻量级引导上下文开始，而不使用上下文引擎管理的生成前状态。
</ParamField>
<ParamField path="onSubagentEnded" type="method">
  在子智能体会话完成或被清理时执行清理。
</ParamField>

### 系统提示词附加内容

`assemble` 方法可以返回 `systemPromptAddition` 字符串。OpenClaw 会将其添加到本次运行的系统提示词之前。这样，引擎无需静态工作区文件，即可注入动态回忆指导、检索指令或上下文感知提示。

## 旧版引擎

内置的 `legacy` 引擎保留 OpenClaw 的原始行为：

- **摄取**：不执行任何操作（会话管理器直接处理消息持久化）。
- **组装**：直接传递（运行时中现有的清理 → 验证 → 限制流水线负责上下文组装）。
- **压缩**：委托给内置的总结压缩机制，该机制会为较早的消息生成一份摘要，并完整保留近期消息。
- **轮次结束后**：不执行任何操作。

旧版引擎不会注册工具，也不提供 `systemPromptAddition`。

未设置 `plugins.slots.contextEngine`（或将其设置为 `"legacy"`）时，会自动使用此引擎。

## 插件引擎

插件可以使用插件 API 注册上下文引擎：

```ts
import { buildMemorySystemPromptAddition } from "openclaw/plugin-sdk/core";

export default function register(api) {
  api.registerContextEngine("my-engine", (ctx) => ({
    info: {
      id: "my-engine",
      name: "My Context Engine",
      ownsCompaction: true,
    },

    async ingest({ sessionId, message, isHeartbeat }) {
      // 将消息存储到你的数据存储中
      return { ingested: true };
    },

    async assemble({
      sessionId,
      sessionKey,
      messages,
      tokenBudget,
      availableTools,
      citationsMode,
    }) {
      // 返回符合预算的消息
      return {
        messages: buildContext(messages, tokenBudget),
        estimatedTokens: countTokens(messages),
        systemPromptAddition: buildMemorySystemPromptAddition({
          availableTools: availableTools ?? new Set(),
          citationsMode,
          agentSessionKey: sessionKey,
        }),
      };
    },

    async compact({ sessionId, force }) {
      // 总结较早的上下文
      return { ok: true, compacted: true };
    },
  }));
}
```

工厂 `ctx` 包含可选的 `config`、`agentDir` 和 `workspaceDir` 值，使插件能够在首次调用生命周期钩子前初始化每个智能体或每个工作区的状态。在调用非旧版的 `assemble()` 前，宿主会完成已注册的异步记忆提示词准备。同步的 `buildMemorySystemPromptAddition(...)` 辅助函数会读取该不可变的运行快照；请原样传递所提供的工具、引用、智能体和会话上下文。

然后在配置中启用它：

```json5
{
  plugins: {
    slots: {
      contextEngine: "my-engine",
    },
    entries: {
      "my-engine": {
        enabled: true,
      },
    },
  },
}
```

### ContextEngine 接口

必需成员：

| 成员               | 类型     | 用途                                                 |
| ------------------ | -------- | -------------------------------------------------------- |
| `info`             | 属性 | 引擎 id、名称、版本，以及是否由其负责压缩 |
| `ingest(params)`   | 方法   | 存储单条消息                                   |
| `assemble(params)` | 方法   | 为模型运行构建上下文（返回 `AssembleResult`） |
| `compact(params)`  | 方法   | 总结/精简上下文                                 |

`assemble` 返回一个 `AssembleResult`，其中包含：

<ParamField path="messages" type="Message[]" required>
  要发送给模型的有序消息。
</ParamField>
<ParamField path="estimatedTokens" type="number" required>
  引擎对组装后上下文中令牌总数的估算。OpenClaw 使用该值进行压缩阈值判断和诊断报告。
</ParamField>
<ParamField path="systemPromptAddition" type="string">
  添加到系统提示词之前。
</ParamField>
<ParamField path="promptAuthority" type='"assembled" | "preassembly_may_overflow"'>
  控制运行器在预防性溢出预检查中使用哪一个令牌估算值。默认值为 `"assembled"`，这意味着对于不负责压缩的引擎，仅检查组装后提示词的估算值。设置 `ownsCompaction: true` 的引擎自行管理提示词准入，因此 OpenClaw 默认跳过通用的提示前预检查。仅当组装后的视图可能掩盖底层对话记录中的溢出风险时，才设置 `"preassembly_may_overflow"`；此时运行器会保持通用预检查启用，并在决定是否进行预防性压缩时，取组装后估算值与组装前（未进行窗口裁剪的）会话历史记录估算值中的较大值。无论采用哪种方式，模型看到的仍然是你返回的消息——`promptAuthority` 仅影响预检查。
</ParamField>
<ParamField path="contextProjection" type="ContextEngineProjection">
  为具有持久后端线程的宿主（例如 Codex app-server）提供的可选投影生命周期。带有稳定 `epoch` 的 `mode: "thread_bootstrap"` 会要求宿主在每个纪元中仅注入一次组装后的上下文，并在纪元发生变化前复用后端线程，而不是每轮都重新投影。对于常规的逐轮投影，请省略此字段。
</ParamField>

`compact` 返回一个 `CompactResult`。当压缩更改活动会话标识时，`result.sessionTarget`（携带会话标识和存储作用域的类型化 `ContextEngineSessionTarget`）会标识下一次重试或轮次必须使用的后继会话；`result.sessionId` 对应后继会话 id。

可选成员：

| 成员                         | 类型   | 用途                                                                                                                                      |
| ------------------------------ | ------ | -------------------------------------------------------------------------------------------------------------------------------------------- |
| `bootstrap(params)`            | 方法 | 初始化会话的引擎状态。引擎首次看到某个会话时调用一次（例如导入历史记录）。                              |
| `maintain(params)`             | 方法 | 在引导启动、轮次成功完成或压缩后维护对话记录。使用 `runtimeContext.rewriteTranscriptEntries()` 进行安全重写。 |
| `ingestBatch(params)`          | 方法 | 批量摄取一个已完成的轮次。在运行完成后调用，一次传入该轮次的所有消息。                                  |
| `afterTurn(params)`            | 方法 | 运行后的生命周期工作（持久化状态、触发后台压缩）。                                                                      |
| `prepareSubagentSpawn(params)` | 方法 | 在子会话开始前设置共享状态。                                                                                    |
| `onSubagentEnded(params)`      | 方法 | 在子智能体结束后执行清理。                                                                                                              |
| `dispose()`                    | 方法 | 释放资源。在 Gateway 网关关闭或插件重新加载期间调用，而非按会话调用。                                                        |

### 运行时设置

在 OpenClaw 内运行的生命周期钩子会接收一个可选的 `runtimeSettings` 对象。它是一个带版本号的只读内部生产者/消费者 API 表面：OpenClaw 为所选上下文引擎生成该对象，上下文引擎则在生命周期钩子中使用它。它不会直接呈现给用户，也不会创建专用的报告界面。

- `schemaVersion`：当前为 `1`
- `runtime`：OpenClaw 主机、运行时模式（`normal`、`fallback` 或
  `degraded`），以及可选的 harness/运行时 ID
- `contextEngineSelection`：所选上下文引擎 ID 和选择来源
- `executionHost`：调用该钩子的界面对应的主机 ID 和标签
- `model`：请求的模型、解析后的模型、提供商，以及可选的模型系列
- `limits`：已知时的提示词 token 预算和最大输出 token 数
- `diagnostics`：已知时的封闭式回退和降级原因代码

可能未知的字段表示为 `null`；运行时模式和选择来源等判别字段
仍不可为 null。旧版引擎仍保持兼容：如果严格的旧版引擎将
`runtimeSettings` 作为未知属性拒绝，OpenClaw 会在不携带该属性的情况下重试生命周期调用，
而不是隔离该引擎。

### 主机要求

上下文引擎可以在 `info.hostRequirements` 上声明主机能力要求。
OpenClaw 会在开始操作前检查这些要求；如果所选运行时无法满足要求，
则会以描述性错误执行封闭式失败。

对于智能体运行，当引擎必须通过 `assemble()` 控制
实际模型提示词时，请声明 `assemble-before-prompt`：

```ts
info: {
  id: "my-context-engine",
  name: "My Context Engine",
  hostRequirements: {
    "agent-run": {
      requiredCapabilities: ["assemble-before-prompt"],
      unsupportedMessage:
        "使用原生 Codex 或 OpenClaw 嵌入式运行时，或者选择旧版上下文引擎。",
    },
  },
}
```

原生 Codex 和 OpenClaw 嵌入式智能体运行满足 `assemble-before-prompt`。
通用 CLI 后端不满足此要求，因此需要该能力的引擎会在
CLI 进程启动前被拒绝。

### 故障隔离

OpenClaw 将所选插件引擎与核心回复路径隔离。如果非旧版引擎缺失、
未通过契约验证、在创建工厂时抛出异常，或从生命周期方法中抛出异常，
OpenClaw 会在当前 Gateway 网关进程中隔离该引擎，并将上下文引擎工作
降级到内置的 `legacy` 引擎。系统会连同失败的操作记录该错误，
以便操作员修复、更新或禁用插件，而不会导致智能体无响应。

主机要求失败则有所不同：当引擎声明某个运行时缺少必需能力时，
OpenClaw 会在开始运行前执行封闭式失败。这样可以保护那些在不受支持的主机中运行时
会破坏状态的引擎。

### ownsCompaction

`ownsCompaction` 控制 OpenClaw 运行时内置的单次尝试内自动压缩功能是否在该次运行中保持启用：

<AccordionGroup>
  <Accordion title="ownsCompaction: true">
    引擎负责压缩行为。OpenClaw 会为该次运行禁用 OpenClaw 运行时内置的自动压缩和通用提示词前溢出预检，而引擎的 `compact()` 实现负责 `/compact`、提供商溢出恢复压缩，以及它希望在 `afterTurn()` 中执行的任何主动压缩。当引擎从 `assemble()` 返回 `promptAuthority: "preassembly_may_overflow"` 时，OpenClaw 仍会运行提示词前溢出保护措施。
  </Accordion>
  <Accordion title="ownsCompaction: false 或未设置">
    OpenClaw 运行时内置的自动压缩仍可能在执行提示词期间运行，但对于 `/compact` 和溢出恢复，仍会调用活动引擎的 `compact()` 方法。
  </Accordion>
</AccordionGroup>

<Warning>
`ownsCompaction: false` **并不**意味着 OpenClaw 会自动回退到旧版引擎的压缩路径。
</Warning>

这意味着插件有两种有效模式：

<Tabs>
  <Tab title="自主管理模式">
    实现自己的压缩算法并设置 `ownsCompaction: true`。
  </Tab>
  <Tab title="委托模式">
    设置 `ownsCompaction: false`，并让 `compact()` 调用 `openclaw/plugin-sdk/core` 中的 `delegateCompactionToRuntime(...)`，以使用 OpenClaw 的内置压缩行为。
  </Tab>
</Tabs>

对于活动的非自主管理引擎，无操作的 `compact()` 并不安全，因为它会禁用该引擎槽位的正常 `/compact` 和溢出恢复压缩路径。

## 配置参考

```json5
{
  plugins: {
    slots: {
      // 选择活动的上下文引擎。默认值："legacy"。
      // 设置为插件 ID 以使用插件引擎。
      contextEngine: "legacy",
    },
  },
}
```

<Note>
该槽位在运行时具有排他性——对于给定的运行或压缩操作，只会解析一个已注册的上下文引擎。其他已启用的 `kind: "context-engine"` 插件仍可加载并运行其注册代码；`plugins.slots.contextEngine` 仅选择 OpenClaw 在需要上下文引擎时解析的已注册引擎 ID。
</Note>

<Note>
**卸载插件：**卸载当前被选为 `plugins.slots.contextEngine` 的插件时，OpenClaw 会将该槽位重置为默认值（`legacy`）。相同的重置行为也适用于 `plugins.slots.memory`。无需手动编辑配置。
</Note>

## 与压缩和记忆的关系

<AccordionGroup>
  <Accordion title="压缩">
    压缩是上下文引擎的职责之一。旧版引擎委托给 OpenClaw 的内置摘要功能。插件引擎可以实现任意压缩策略（DAG 摘要、向量检索等）。
  </Accordion>
  <Accordion title="记忆插件">
    记忆插件（`plugins.slots.memory`）与上下文引擎相互独立。记忆插件提供搜索/检索功能；上下文引擎控制模型看到的内容。两者可以协同工作——上下文引擎可以在组装期间使用记忆插件数据。希望使用活动记忆提示词路径的插件引擎应使用 `openclaw/plugin-sdk/core` 中的 `buildMemorySystemPromptAddition(...)`，它会将主机准备的记忆提示词各部分转换为可直接前置的 `systemPromptAddition`，而不会暴露记忆插件的布局。
  </Accordion>
  <Accordion title="会话裁剪">
    无论哪个上下文引擎处于活动状态，仍会在内存中裁剪旧的工具结果。
  </Accordion>
</AccordionGroup>

## 提示

- 使用 `openclaw doctor` 验证引擎是否正确加载。
- 切换引擎时，现有会话会继续保留其当前历史记录。新引擎将接管后续运行。
- 系统会记录引擎错误，并在当前 Gateway 网关进程中隔离所选插件引擎。OpenClaw 会在用户轮次中回退到 `legacy`，以便继续回复，但你仍应修复、更新、禁用或卸载损坏的插件。
- 在开发过程中，使用 `openclaw plugins install -l ./my-engine` 链接本地插件目录，无需复制。

## 相关内容

- [压缩](/zh-CN/concepts/compaction) - 对长对话进行摘要
- [上下文](/zh-CN/concepts/context) - 如何为智能体轮次构建上下文
- [插件架构](/zh-CN/plugins/architecture) - 注册上下文引擎插件
- [插件清单](/zh-CN/plugins/manifest) - 插件清单字段
- [插件](/zh-CN/tools/plugin) - 插件概览
