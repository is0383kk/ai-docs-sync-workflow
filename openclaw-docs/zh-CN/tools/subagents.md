---
read_when:
    - 你希望通过智能体执行后台或并行工作
    - 你正在更改 `sessions_spawn` 或子智能体工具策略
    - 你正在实现或排查与线程绑定的子智能体会话问题
sidebarTitle: Sub-agents
summary: 生成隔离的后台智能体运行，并将结果通知回请求者聊天会话
title: 子智能体
x-i18n:
    generated_at: "2026-07-26T07:04:34Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: e45b32fdb177c52ed785287712b9b6c2c30bbe392f0ce975970910ff91ed30ed
    source_path: tools/subagents.md
    workflow: 16
---

子智能体是从现有智能体运行中派生的后台智能体运行。
每个子智能体都在自己的会话（`agent:<agentId>:subagent:<uuid>`）中运行，并在完成后将结果**通知**回请求方的聊天渠道。
每个子智能体运行都会作为[后台任务](/zh-CN/automation/tasks)进行跟踪。

目标：

- 并行执行研究、长时间任务和缓慢的工具工作，而不阻塞主运行。
- 默认保持子智能体相互隔离（会话分离，可选沙箱隔离）。
- 降低工具面的误用风险：默认情况下，子智能体不会获得会话或消息工具。
- 支持可配置的嵌套深度，以适应编排器模式。

<Note>
**成本说明：**默认情况下，每个子智能体都有自己的上下文和 token 用量。对于繁重或重复性任务，请为子智能体设置成本较低的模型，并通过 `agents.defaults.subagents.model` 或按智能体覆盖，让主智能体继续使用质量较高的模型。当子智能体确实需要请求方当前的记录时，请使用 `context: "fork"` 派生它。绑定到话题串的子智能体会话默认使用 `context: "fork"`，因为它们会将当前对话分支到后续话题串中。
</Note>

## 斜杠命令

`/subagents` 可检查**当前会话**的子智能体运行：

```text
/subagents list
/subagents log <id|#> [limit] [tools]
/subagents info <id|#>
```

`/subagents info` 显示运行元数据（状态、时间戳、会话 ID、记录路径、清理）。`/subagents log` 输出某次运行最近的聊天轮次；添加 `tools` token 可包含工具调用/结果消息（默认省略）。在智能体轮次中，使用 `sessions_history` 获取有边界且经过安全过滤的回溯视图，或检查磁盘上的记录路径以查看原始完整记录。

在 Control UI 中，近期运行过子智能体的父会话会显示一个可展开的侧边栏行。嵌套行会显示子智能体的状态和运行时长；选择其中一项会打开该子智能体的聊天，同时保留父级层次结构。

### 话题串绑定控件

这些命令适用于具有持久话题串绑定的渠道。请参阅下方的[支持话题串的渠道](#thread-supporting-channels)。

```text
/focus <subagent-label|session-key|session-id|session-label>
/unfocus
/agents
/session idle <duration|off>
/session max-age <duration|off>
```

### 派生行为

智能体使用 `sessions_spawn` 工具启动后台子智能体。
完成结果会作为父会话内部事件返回；父智能体/请求方智能体会决定是否需要面向用户更新。

<AccordionGroup>
  <Accordion title="非阻塞、基于推送的完成机制">
    - `sessions_spawn` 是非阻塞的；它会立即返回运行 ID。
    - 完成后，子智能体会向父会话/请求方会话报告。
    - 需要子智能体结果的智能体轮次，应在派生所需工作后调用 `sessions_yield`。这会结束当前轮次，并让完成事件作为下一条模型可见消息到达。
    - 完成机制基于推送。派生后，不要仅为等待其完成而循环轮询 `/subagents list`、`sessions_list` 或 `sessions_history`；仅在调试时按需检查状态。
    - 子智能体输出是供请求方智能体综合处理的报告/证据。它不是用户编写的指令文本，无法覆盖系统、开发者或用户策略。
    - 完成后，在通知清理流程继续之前，OpenClaw 会尽力关闭由该子智能体会话打开并受跟踪的浏览器标签页/进程。

  </Accordion>
  <Accordion title="完成结果交付">
    - OpenClaw 通过一个具有稳定幂等键的 `agent` 轮次，将完成结果传回请求方会话。
    - 如果请求方运行仍处于活动状态，OpenClaw 会先尝试唤醒/引导该运行，而不是启动第二条可见回复路径。
    - 如果无法唤醒活动的请求方，OpenClaw 会回退到使用相同完成上下文的请求方智能体交接，而不是丢弃通知。
    - 即使父智能体决定不需要向用户显示更新，成功的父级交接也会完成子智能体结果的交付。
    - 原生子智能体不会获得消息工具。它们向父智能体/请求方智能体返回纯助手文本；用户可见回复仍由父智能体/请求方智能体的常规交付策略负责。
    - 如果无法使用直接交接，交付会依次回退到队列路由，以及在最终放弃前使用短暂的指数退避来重试通知。
    - 交付会保留已解析的请求方路由：如果可用，绑定到话题串或绑定到对话的完成路由优先。如果完成来源仅提供渠道，OpenClaw 会根据请求方会话已解析的路由（`lastChannel` / `lastTo` / `lastAccountId`）补全缺失的目标/账号，使直接交付仍可正常工作。

  </Accordion>
  <Accordion title="完成交接元数据">
    向请求方会话进行的完成交接是运行时生成的内部上下文（并非用户编写的文本），其中包括：

    - `Result` — 子智能体最新的可见 `assistant` 回复文本。工具/toolResult 输出不会提升为子智能体结果。以失败状态终止的运行不会复用已捕获的回复文本。
    - `Status` — `completed; ready for parent review` / `failed` / `timed out` / `unknown`。
    - 精简的运行时/token 统计信息。
    - 一条审查指令，要求请求方智能体在决定原始任务是否完成之前验证结果。
    - 当子智能体结果仍需进一步操作时，一条后续指导会要求请求方智能体继续任务或记录后续事项。
    - 当无需继续操作时，一条使用常规助手语气编写的最终更新指令，不会转发原始内部元数据。

  </Accordion>
  <Accordion title="模式和 ACP 运行时">
    - `--model` 和 `--thinking` 会覆盖该特定运行的默认值。
    - 使用 `info`/`log` 检查完成后的详细信息和输出。
    - 对于持久的绑定话题串会话，请将 `sessions_spawn` 与 `thread: true` 和 `mode: "session"` 搭配使用。
    - 如果请求方渠道不支持话题串绑定，请使用 `mode: "run"`，而不要重试不可能实现的绑定话题串组合。
    - 对于 ACP harness 会话（Claude Code、Gemini CLI、OpenCode 或显式 Codex ACP/acpx），当工具声明支持该运行时时，请将 `sessions_spawn` 与 `runtime: "acp"` 搭配使用。调试完成结果或智能体间循环时，请参阅 [ACP 交付模型](/zh-CN/tools/acp-agents#delivery-model)。启用 `codex` 插件后，除非用户明确要求使用 ACP/acpx，否则 Codex 聊天/话题串控制应优先使用 `/codex ...`，而非 ACP。
    - 在 ACP 未启用、请求方处于沙箱隔离状态或尚未加载 `acpx` 等后端插件时，OpenClaw 会隐藏 `runtime: "acp"`。`runtime: "acp"` 需要一个外部 ACP harness ID，或一个带有 `runtime.type="acp"` 的 `agents.entries.*` 条目；对于来自 `agents_list` 的普通 OpenClaw 配置智能体，请使用默认子智能体运行时。

  </Accordion>
</AccordionGroup>

## 上下文模式

除非调用方明确要求派生当前记录，否则原生子智能体启动时相互隔离。

| 模式       | 使用场景                                                                                                                         | 行为                                                                          |
| ---------- | -------------------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------- |
| `isolated` | 全新研究、独立实现、缓慢的工具工作，或任何可在任务文本中完整说明的工作                           | 创建干净的子记录。这是默认行为，可降低 token 用量。  |
| `fork`     | 依赖当前对话、先前工具结果，或请求方记录中已有细微指令的工作 | 在子智能体启动之前，将请求方记录分支到子会话中。 |

请谨慎使用 `fork`。它用于对上下文敏感的委派，不能替代清晰的任务提示词。

## 工具：`sessions_spawn`

在全局 `subagent` 通道上使用 `deliver: false` 启动子智能体运行，然后执行通知步骤，并将通知回复发布到请求方聊天渠道。

是否可用取决于调用方的有效工具策略。内置 `coding` 和 `messaging` 配置文件包含 `sessions_spawn`、`sessions_yield` 和 `subagents`；`minimal` 不包含这些工具。`full` 允许所有工具。对于使用自定义、更窄配置文件但仍需委派工作的智能体，请通过 `tools.alsoAllow` 添加这些工具，或使用上述配置文件之一。
在配置文件阶段之后，渠道/群组、提供商、沙箱及按智能体设置的允许/拒绝策略仍可能移除该工具。使用同一会话中的 `/tools` 确认有效工具列表。

**默认值：**

- **模型：**除非设置 `agents.defaults.subagents.model`（或按智能体设置 `agents.entries.*.subagents.model`），否则原生子智能体会继承调用方的模型。存在已配置的子智能体模型时，ACP 运行时派生也会使用相同模型；否则，ACP harness 会保留自己的默认模型。显式设置的 `sessions_spawn.model` 仍具有最高优先级。
- **思考：**除非设置 `agents.defaults.subagents.thinking`（或按智能体设置 `agents.entries.*.subagents.thinking`），否则原生子智能体会继承调用方的设置。ACP 运行时派生也会为所选模型应用 `agents.defaults.models["provider/model"].params.thinking`。显式设置的 `sessions_spawn.thinking` 仍具有最高优先级。
- **运行超时：**设置后，OpenClaw 使用 `agents.defaults.subagents.runTimeoutSeconds`；否则回退到 `0`（无超时）。`sessions_spawn` 不接受按调用设置的超时覆盖。
- **进程生命周期：**分离的 OpenClaw 子智能体拥有自己的运行生命周期。在外部 CLI 后端中创建的后台任务则有所不同：它会与父 CLI 子进程共享生命周期，并在父进程达到 `agents.defaults.timeoutSeconds` 时停止。
- **任务交付：**原生子智能体通过其第一条可见的 `[Subagent Task]` 消息接收委派任务。子智能体系统提示词包含运行时规则和路由上下文，不包含任务的隐藏副本。

已接受的原生子智能体派生会在工具结果中包含已解析的子智能体模型元数据：`resolvedModel` 包含已应用的模型引用；如果该引用带有提供商前缀，`resolvedProvider` 会包含该前缀。

### 委派提示模式

`agents.defaults.subagents.delegationMode` 仅控制提示词指导；它不会更改工具策略，也不会强制执行委派。

- `suggest`（默认）：保留标准提示，建议对规模较大或耗时较长的工作使用子智能体。
- `prefer`：要求主智能体保持响应，并通过 `sessions_spawn` 委派任何比直接回复更复杂的工作。

按智能体覆盖：`agents.entries.*.subagents.delegationMode`。

```json5
{
  agents: {
    defaults: {
      subagents: {
        delegationMode: "prefer",
        maxConcurrent: 4,
      },
    },
    list: [
      {
        id: "coordinator",
        subagents: { delegationMode: "prefer" },
      },
    ],
  },
}
```

### 工具参数

<ParamField path="task" type="string" required>
  子智能体的任务描述。
</ParamField>
<ParamField path="taskName" type="string">
  可选的稳定句柄，用于在后续状态输出中识别特定子项。必须匹配 `[a-z][a-z0-9_-]{0,63}`，且不能是 `last` 或 `all` 等保留目标。
</ParamField>
<ParamField path="label" type="string">
  可选的易读标签。
</ParamField>
<ParamField path="agentId" type="string">
  在 `subagents.allowAgents` 允许时，在另一个已配置的智能体 ID 下生成。
</ParamField>
<ParamField path="cwd" type="string">
  子运行的可选任务工作目录。原生子智能体仍从目标智能体工作区加载引导文件；`cwd` 仅更改运行时工具和 CLI harness 执行委派工作的目录。
</ParamField>
<ParamField path="runtime" type='"subagent" | "acp"' default="subagent">
  `acp` 仅用于外部 ACP harness（`claude`、`droid`、`gemini`、`opencode` 或明确请求的 Codex ACP/acpx），以及 `runtime.type` 为 `acp` 的 `agents.entries.*` 条目。
</ParamField>
<ParamField path="resumeSessionId" type="string">
  仅限 ACP。当 `runtime: "acp"` 时恢复现有 ACP harness 会话；原生子智能体生成会忽略此项。
</ParamField>
<ParamField path="streamTo" type='"parent"'>
  仅限 ACP。当 `runtime: "acp"` 时，将 ACP 运行输出流式传输到父会话；原生子智能体生成应省略此项。
</ParamField>
<ParamField path="model" type="string">
  覆盖子智能体模型。无效值会被跳过，子智能体将使用默认模型运行，并在工具结果中显示警告。
</ParamField>
<ParamField path="thinking" type="string">
  覆盖子智能体运行的思考级别。使用 `visible: true` 时不可用。
</ParamField>
<ParamField path="thread" type="boolean" default="false">
  当 `true` 时，请求为此子智能体会话绑定渠道线程。
</ParamField>
<ParamField path="mode" type='"run" | "session"' default="run">
  如果 `thread: true` 且省略 `mode`，默认值将变为 `session`。`mode: "session"` 需要 `thread: true`。
  如果请求方渠道无法使用线程绑定，请改用 `mode: "run"`。
  使用 `visible: true` 时，请省略 `mode`；可见会话是持久会话，不支持 `mode: "run"`。
</ParamField>
<ParamField path="cleanup" type='"delete" | "keep"' default="keep">
  `"delete"` 会在公告后立即归档会话（仍会通过重命名保留会话记录）。
</ParamField>
<ParamField path="sandbox" type='"inherit" | "require"' default="inherit">
  除非目标子运行时已进行沙箱隔离，否则 `require` 会拒绝生成。
</ParamField>
<ParamField path="context" type='"isolated" | "fork"' default="isolated">
  `fork` 将请求方的当前会话记录分支到子会话中。仅限原生子智能体。线程绑定生成默认为 `fork`；非线程生成默认为 `isolated`。可见分支必须以与请求方相同的智能体为目标。
</ParamField>
<ParamField path="visible" type="boolean" default="false">
  创建一个用户可在 Control UI 中打开的持久仪表板会话。可见生成仅支持 `runtime: "subagent"`，并且始终保留所创建的会话。
</ParamField>
<ParamField path="worktree" type="boolean" default="false">
  为新仪表板会话配置托管的 git 工作树。需要 `visible: true`。
</ParamField>
<ParamField path="worktreeName" type="string">
  可选的托管工作树名称。需要 `visible: true` 和 `worktree: true`。
</ParamField>
<ParamField path="worktreeBaseRef" type="string">
  托管工作树的可选 git 基础引用。需要 `visible: true` 和 `worktree: true`。
</ParamField>

<Warning>
`sessions_spawn` **不**接受渠道投递参数（`target`、
`channel`、`to`、`threadId`、`replyTo`、`transport`）。原生子智能体会将
其最新的助手轮次报告给请求方；外部投递仍由
父智能体/请求方智能体负责。
</Warning>

支持 `visible: true`、`model`、`cwd` 和同智能体的 `context: "fork"`。沙箱隔离的目标会将 `cwd` 限制在该智能体的工作区内。此路径不支持线程绑定、`mode`、思考覆盖、`lightContext`、`attachments` 和 `attachAs`，因为可见会话是通过 `sessions.create` 创建的持久仪表板会话。如果请求方本身是使用继承的工具允许列表或拒绝列表生成的，则会拒绝可见生成；此限制在生成时固定，无法通过配置覆盖。会话列表和寻址遵循 `tools.sessions.visibility`；默认的 `tree` 范围涵盖当前会话及其自己的生成子树。有关检出命名、设置、清理和恢复行为，请参阅[托管工作树](/zh-CN/concepts/managed-worktrees)。

### 任务名称和目标定位

`taskName` 是面向模型的编排句柄，而不是会话键。
当协调器之后可能需要检查某个子项时，可使用它为子项指定稳定名称，例如 `review_subagents`、
`linux_validation` 或 `docs_update`。

目标解析接受完全匹配的 `taskName` 和无歧义的
前缀。匹配范围与编号 `/subagents` 目标使用的当前活跃/近期目标窗口相同，
因此过期的已完成子项不会使复用的句柄
产生歧义。如果两个活跃或近期子项共享同一个
`taskName`，则目标存在歧义；请改用列表索引、会话键或
运行 ID。

保留目标 `last` 和 `all` 不能作为有效的 `taskName` 值，
因为它们已有控制含义。

## 工具：`sessions_yield`

结束当前模型轮次并等待运行时事件（主要是
子智能体完成事件）作为下一条消息到达。在生成必需的子任务后，
如果请求方必须等这些任务完成才能生成最终
答案，请使用此工具。

`sessions_yield` 是等待原语。不要仅为了检测子任务完成，
就用对 `subagents`、`sessions_list`、`sessions_history` 的轮询循环、shell
`sleep` 或进程轮询来代替它。

仅当会话的有效工具列表包含
`sessions_yield` 时才使用它。某些精简或自定义工具配置文件可能会提供 `sessions_spawn` 和
`subagents`，但不提供 `sessions_yield`；在这种情况下，不要仅为等待完成
而构造轮询循环。

当存在活跃子项时，OpenClaw 会在正常轮次中注入一个紧凑的、由运行时生成的
`Active Subagents` 提示块，让请求方无需轮询即可查看
当前子会话、运行 ID、状态、标签、任务和
`taskName` 别名。该块中的任务和标签字段会作为数据加引号，
而不是作为指令，因为它们可能来自
用户/模型提供的生成参数。

## 工具：`subagents`

列出由请求方会话树拥有的已生成子智能体运行和后台任务记录。
任务行涵盖原生子智能体、ACP 运行、
Gateway CLI/媒体工作和定时任务执行。其范围限于当前
请求方；子项只能查看自己控制的子项。

使用 `subagents` 进行按需状态检查和调试。使用 `sessions_yield`
等待完成事件。

使用 `action: "cancel"` 和 `action: "list"` 返回的 `taskId` 来停止
任务。取消操作仅限于受控会话树；叶级
子智能体无法取消其他会话拥有的工作。

## 线程绑定会话

为某个渠道启用线程绑定后，子智能体可以持续绑定到
一个线程，使该线程中的后续用户消息继续路由到
同一个子智能体会话。

### 支持线程的渠道

当渠道注册了对话
绑定适配器时，它就支持持久的线程绑定子智能体会话
（`sessions_spawn` 与 `thread: true`）。支持此功能的内置渠道：**Discord**、
**iMessage**、**Matrix** 和 **Telegram**。Discord 和 Matrix 默认
创建子线程；Telegram 和 iMessage 默认绑定
当前对话。使用各渠道的 `threadBindings` 配置键设置
启用状态、超时和 `spawnSessions`。

### 快速流程

<Steps>
  <Step title="生成">
    使用 `sessions_spawn` 和 `thread: true`（以及可选的 `mode: "session"`）。
  </Step>
  <Step title="绑定">
    OpenClaw 在活跃渠道中创建线程或将线程绑定到该会话目标。
  </Step>
  <Step title="路由后续消息">
    该线程中的回复和后续消息会路由到绑定的会话。
  </Step>
  <Step title="检查超时">
    使用 `/session idle` 检查/更新非活跃状态下的自动取消聚焦，并使用
    `/session max-age` 控制硬性上限。
  </Step>
  <Step title="解除绑定">
    使用 `/unfocus` 手动解除绑定。
  </Step>
</Steps>

### 手动控制

| 命令            | 效果                                                                                    |
| ------------------ | ----------------------------------------------------------------------------------------- |
| `/focus <target>`  | 将当前线程绑定到子智能体/会话目标（或创建一个线程）                     |
| `/unfocus`         | 移除当前已绑定线程的绑定                                           |
| `/agents`          | 列出活跃运行和绑定状态（`binding:<id>`、`unbound` 或 `bindings unavailable`） |
| `/session idle`    | 检查/更新空闲时自动取消聚焦（仅限已聚焦的绑定线程）                             |
| `/session max-age` | 检查/更新硬性上限（仅限已聚焦的绑定线程）                                      |

### 配置开关

- **全局默认值：** `session.threadBindings.enabled`、`session.threadBindings.idleHours`、`session.threadBindings.maxAgeHours`。
- **渠道覆盖和生成时自动绑定键**取决于适配器。请参阅上文的[支持线程的渠道](#thread-supporting-channels)。

有关当前适配器的详细信息，请参阅[配置参考](/zh-CN/gateway/configuration-reference)和
[斜杠命令](/zh-CN/tools/slash-commands)。

### 允许列表

<ParamField path="agents.entries.*.subagents.allowAgents" type="string[]">
  可通过显式 `agentId` 指定的已配置智能体 ID 列表（`["*"]` 允许任意已配置目标）。默认值：仅限请求方智能体。如果设置了列表，但仍希望请求方通过 `agentId` 生成自身，请将请求方 ID 包含在列表中。
</ParamField>
<ParamField path="agents.defaults.subagents.allowAgents" type="string[]">
  当请求方智能体未设置自己的 `subagents.allowAgents` 时使用的默认已配置目标智能体允许列表。
</ParamField>
<ParamField path="agents.defaults.subagents.requireAgentId" type="boolean" default="false">
  阻止省略 `agentId` 的 `sessions_spawn` 调用（强制显式选择配置文件）。按智能体覆盖：`agents.entries.*.subagents.requireAgentId`。
</ParamField>
<ParamField path="agents.defaults.subagents.announceTimeoutMs" type="number" default="120000">
  Gateway 网关 `agent` 公告投递尝试的单次调用超时。值必须为正整数毫秒，并限制在平台安全的计时器最大值以内。瞬态重试可能使公告总等待时间超过单个配置的超时时间。
</ParamField>

如果请求方会话已进行沙箱隔离，`sessions_spawn` 会拒绝将以
非沙箱隔离方式运行的目标。

### 设备发现

使用 `agents_list` 查看当前允许用于
`sessions_spawn` 的 Agent ID。响应包含列出的每个 Agent 的实际生效
模型和嵌入式运行时元数据，以便调用方区分 OpenClaw、Codex
app-server 和其他已配置的原生运行时。

`allowAgents` 条目必须指向 `agents.entries.*` 中已配置的 Agent ID。
`["*"]` 表示任意已配置的目标 Agent 加上请求方。如果删除了某个 Agent 配置，
但其 ID 仍保留在 `allowAgents` 中，`sessions_spawn` 会拒绝该 ID，
而 `agents_list` 会将其省略。运行 `openclaw doctor --fix` 以清理过期的
允许列表条目；如果目标应在继承默认值的同时保持可生成，则添加最小化的
`agents.entries.*` 条目。

### 自动归档

- 子智能体会话会在 `agents.defaults.subagents.archiveAfterMinutes` 后自动归档（默认值为 `60`）。
- 归档使用 `sessions.delete`，并将记录重命名为 `*.deleted.<timestamp>`（位于同一文件夹）。
- `cleanup: "delete"` 会在通知后立即归档（仍通过重命名保留记录）。
- 自动归档是尽力而为的；如果 Gateway 网关重启，待处理的计时器将丢失。
- 已配置的运行超时**不会**自动归档；它们只会停止运行。会话会一直保留到自动归档。
- 自动归档同样适用于深度为 1 和深度为 2 的会话。
- 浏览器清理与归档清理相互独立：运行结束时，会尽力关闭被跟踪的浏览器标签页/进程，即使记录/会话记录仍被保留。

## 嵌套子智能体

默认情况下，子智能体无法生成自己的子智能体
（`maxSpawnDepth: 1`）。将 `maxSpawnDepth: 2` 设为启用一层
嵌套，即**编排器模式**：主智能体 → 编排器子智能体 →
工作子智能体。

```json5
{
  agents: {
    defaults: {
      subagents: {
        maxSpawnDepth: 2, // 允许子智能体生成子级（默认值：1，范围 1-5）
        maxChildrenPerAgent: 5, // 每个智能体会话的最大活跃子级数（默认值：5，范围 1-20）
        maxConcurrent: 8, // 全局并发通道上限（默认值：8）
        runTimeoutSeconds: 900, // sessions_spawn 的默认超时时间（0 = 不超时）
        announceTimeoutMs: 120000, // 每次调用的 Gateway 网关通知超时时间
      },
    },
  },
}
```

### 深度级别

| 深度 | 会话键格式                                   | 角色                                          | 能否生成？                   |
| ----- | -------------------------------------------- | --------------------------------------------- | ---------------------------- |
| 0     | `agent:<id>:main`                            | 主智能体                                      | 始终可以                     |
| 1     | `agent:<id>:subagent:<uuid>`                 | 子智能体（允许深度 2 时为编排器）             | 仅当 `maxSpawnDepth >= 2` 时 |
| 2     | `agent:<id>:subagent:<uuid>:subagent:<uuid>` | 孙级智能体（叶级工作单元）                    | 永远不能                     |

### 通知链

结果沿链向上返回：

1. 深度为 2 的工作单元完成 → 通知其父级（深度为 1 的编排器）。
2. 深度为 1 的编排器收到通知、综合结果并完成 → 通知主智能体。
3. 主智能体收到通知并将结果交付给用户。

每一级只能看到来自其直接子级的通知。

<Note>
**操作指导：**只启动一次子级工作并等待完成
事件，不要围绕 `sessions_list`、
`sessions_history`、`/subagents list` 或 `exec` 休眠命令构建轮询循环。
`sessions_list` 和 `/subagents list` 使子会话关系
聚焦于实时工作——运行中的子级保持关联，已结束的子级会在较短的近期窗口内
保持可见，而仅存在于存储中的过期子级链接会在超过其新鲜度窗口后
被忽略。这样可以防止重启后旧的 `spawnedBy` /
`parentSessionKey` 元数据使幽灵子级复现。
如果在你已发送最终答案后收到子级完成事件，正确的后续操作是输出完全一致的静默令牌
`NO_REPLY` / `no_reply`。
</Note>

### 按深度划分的工具策略

- 子级生成时会捕获请求方当时实际生效的发送方策略。即使 `toolsBySender` 随后发生变化，无发送方的子级运行和经过身份验证的操作员恢复操作仍会保留该快照；当前的全局、Agent、提供商、沙箱和子智能体限制仍然适用。如果新的外部渠道轮次以该子级为目标，则会改为重新解析当前发送方策略。
- 角色和控制范围会在生成时写入会话元数据。这样可防止扁平化或恢复后的会话键意外重新获得编排器权限。
- **深度 1（当 `maxSpawnDepth >= 2` 时为编排器）：**获得 `sessions_spawn`、`subagents`、`sessions_list`、`sessions_history`，以便生成子级并检查其状态。其他会话/系统工具仍被拒绝。
- **深度 1（当 `maxSpawnDepth == 1` 时为叶级）：**没有会话工具（当前默认行为）。
- **深度 2（叶级工作单元）：**没有会话工具——在深度 2 时始终拒绝 `sessions_spawn`。无法继续生成子级。

### 每个 Agent 的生成限制

每个 Agent 会话（无论深度如何）同时最多可有 `maxChildrenPerAgent`
个活跃子级（默认值为 `5`）。这可以防止单个编排器不受控制地扇出。

### 级联停止

停止深度为 1 的编排器会自动停止其所有深度为 2 的
子级：

- 主聊天中的 `/stop` 会停止所有深度为 1 的 Agent，并级联停止其深度为 2 的子级。

## 身份验证

子智能体的身份验证按 **Agent ID** 解析，而不是按会话类型解析：

- 子智能体会话键为 `agent:<agentId>:subagent:<uuid>`。
- 身份验证存储从该 Agent 的 `agentDir` 加载。
- 主智能体的身份验证配置文件作为**后备**合并；发生冲突时，Agent 配置文件会覆盖主智能体的配置文件。

该合并是增量式的，因此主智能体配置文件始终可作为
后备使用。目前尚不支持每个 Agent 完全隔离的身份验证。

## 通知

子智能体通过通知步骤报告结果：

- 通知步骤在子智能体会话中运行（而不是在请求方会话中）。
- 如果子智能体的回复与 `ANNOUNCE_SKIP` 完全一致，则不会发布任何内容。
- 如果最新的助手文本是完全一致的静默令牌 `NO_REPLY` / `no_reply`，即使此前存在可见进度，也会抑制通知输出。

交付方式取决于请求方的深度：

- 顶层请求方会话使用后续 `agent` 调用进行外部交付（`deliver=true`）。
- 嵌套的请求方子智能体会话接收内部后续注入（`deliver=false`），以便编排器在会话内综合子级结果。
- 如果嵌套的请求方子智能体会话已不存在，OpenClaw 会在可用时回退到该会话的请求方。

对于顶层请求方会话，完成模式的直接交付会先
解析任何已绑定的对话/话题串路由和钩子覆盖，然后从请求方会话存储的路由中
补充缺失的渠道目标字段。
这样，即使完成事件的来源只标识了渠道，也能确保完成结果发送到正确的聊天/主题。

构建嵌套完成结果时，子级完成聚合仅限于当前请求方运行，
可防止先前运行中过期的子级输出泄漏到当前通知中。渠道适配器支持时，
通知回复会保留话题串/主题路由。

### 通知上下文

通知上下文会被规范化为稳定的内部事件块：

| 字段           | 来源                                                                                                   |
| -------------- | -------------------------------------------------------------------------------------------------------- |
| 来源           | `subagent` 或 `cron`                                                                                     |
| 会话 ID        | 子级会话键/ID                                                                                           |
| 类型           | 通知类型 + 任务标签                                                                                     |
| 状态           | 从运行时结果派生（`ok`、`error`、`timeout` 或 `unknown`）——**不会**根据模型文本推断 |
| 结果内容       | 子级最新的可见助手文本                                                                                  |
| 后续操作       | 说明何时回复以及何时保持静默的指令                                                                      |

以失败告终的运行会报告失败状态，而不会重放捕获的
回复文本。工具/工具结果输出不会被提升为子级结果文本。

### 统计行

通知载荷末尾包含统计行（即使经过包装）：

- 运行时长（例如 `runtime 5m12s`）。
- Token 用量（输入/输出/总计）。
- 配置模型定价时的预估成本（`models.providers.*.models[].cost`）。
- `sessionKey`、`sessionId` 和记录路径，以便主智能体通过 `sessions_history` 获取历史记录或检查磁盘上的文件。

内部元数据仅用于编排；面向用户的回复
应使用正常的助手语气重写。

### 为什么首选 `sessions_history`

`sessions_history` 是在 Agent 轮次中读取子级
记录的更安全编排路径：

- 即使通用日志脱敏功能已禁用，也会隐去类似凭据/Token 的文本。
- 截断较长的文本块（每块 4000 个字符），并丢弃思维签名、推理重放载荷和内联图像数据。
- 强制执行 80 KB 的响应上限；超大行会替换为 `[sessions_history omitted: message too large]`。
- 如果存在 `nextOffset`，使用它向后翻页查看更早的记录窗口。
- `sessions_history` **不会**从消息文本中去除推理标签、`<relevant-memories>` 脚手架或工具调用 XML——它返回接近原始记录格式的结构化内容块，只进行脱敏并限制大小。`/subagents log` 会应用更严格的文本清理器（去除推理标签、记忆脚手架和工具调用 XML），因为它呈现的是纯聊天行，而不是结构化块。
- 当需要完整、逐字节一致的记录时，直接检查磁盘上的原始记录是后备方案。

## 工具策略

子智能体首先使用与父级或目标 Agent 相同的配置文件和工具策略管线。
之后，OpenClaw 会应用子智能体限制层。

无论深度或角色如何，子智能体始终无法使用 `gateway`、`agents_list`、`session_status` 和
`cron`（系统级/交互式工具，或应由主智能体协调的
工具）。叶级子智能体（默认的深度 1 行为，以及深度 2 下的所有情况）
还无法使用 `subagents`、`sessions_list`、`sessions_history` 和 `sessions_spawn`。子智能体永远
不会获得 `message` 工具——它在生成时即被禁用，而不是由
此拒绝列表过滤——并且 `sessions_send` 始终被拒绝，因此子智能体
只能通过通知链通信。

`sessions_history` 在此处仍是一个有界且经过清理的回忆视图——它
不是原始记录转储。

当 `maxSpawnDepth >= 2` 时，深度为 1 的编排器子智能体还会
获得 `sessions_spawn`、`subagents`、`sessions_list` 和
`sessions_history`，以便管理其子级。

### 通过配置覆盖

```json5
{
  agents: {
    defaults: {
      subagents: {
        maxConcurrent: 1,
      },
    },
  },
  tools: {
    subagents: {
      tools: {
        // 拒绝规则优先
        deny: ["gateway", "cron"],
        // 如果设置了 allow，则仅允许其中所列工具（拒绝规则仍然优先）
        // allow: ["read", "exec", "process"]
      },
    },
  },
}
```

`tools.subagents.tools.allow` 是最终的仅允许过滤器。它可以缩小
已经解析的工具集，但无法**重新添加**被
`tools.profile` 移除的工具。例如，`tools.profile: "coding"` 包含
`web_search`/`web_fetch`，但不包含 `browser` 工具。要允许
编码配置的子智能体使用浏览器自动化，请在
配置阶段添加 browser：

```json5
{
  tools: {
    profile: "coding",
    alsoAllow: ["browser"],
  },
}
```

如果只有一个智能体应获得浏览器自动化能力，请使用按智能体配置的
`agents.entries.*.tools.alsoAllow: ["browser"]`。

## 并发

子智能体使用专用的进程内队列通道：

- **通道名称：** `subagent`
- **并发数：** `agents.defaults.subagents.maxConcurrent`（默认值为 `8`）

## 活跃性与恢复

OpenClaw 不会将缺少 `endedAt` 视为子智能体仍然存活的
永久证据。超过陈旧运行时间窗口仍未结束的运行
（2 小时，或配置的运行超时时间加上一小段宽限期，
取两者中较长者）将不再计入 `/subagents list`、
状态摘要、后代完成门控和按会话执行的
并发检查中的活动或待处理运行。

Gateway 网关重启后，会清理已恢复但陈旧且未结束的运行，除非
其子会话被标记为 `abortedLastRun: true`。因重启中止的
运行仍会注册在子智能体孤立运行恢复流程中：陈旧
运行会直接结束而不恢复，而较新的子会话会先收到
一条合成的恢复消息，之后才会清除中止标记。

自动重启恢复按子会话设有次数限制。如果同一个
子智能体的子会话在快速反复卡死窗口内多次被接受进行孤立运行恢复，
OpenClaw 会在该会话上持久化恢复墓碑，并在后续重启时
停止自动恢复它。运行
`openclaw tasks maintenance --apply` 以协调任务记录，或运行
`openclaw doctor --fix` 以清除带有墓碑的会话上的陈旧中止恢复标志。

<Note>
如果生成子智能体时出现 Gateway 网关 `PAIRING_REQUIRED` /
`scope-upgrade` 错误，请先检查 RPC 调用方，再编辑配对状态。
当调用方已在 Gateway 网关请求上下文中运行时，内部
`sessions_spawn` 协调会在进程内分派，因此不会
打开 loopback WebSocket，也不依赖 CLI 的已配对设备权限范围
基线。Gateway 网关进程外的调用方仍会通过 WebSocket
回退，以 `client.id: "gateway-client"` 身份使用 `client.mode: "backend"`，
通过直接 loopback 共享令牌/密码进行身份验证。远程调用方、显式
`deviceIdentity`、显式设备令牌路径以及浏览器/节点客户端
在提升权限范围时仍需正常的设备审批。
</Note>

## 停止

- 在请求方聊天中发送 `/stop` 会中止请求方会话，并停止由其生成的所有活动子智能体运行，此操作会级联至嵌套子智能体。

## 限制

- 子智能体通知是**尽力而为**的。如果 Gateway 网关重启，待处理的“回传通知”工作将丢失。
- 子智能体仍共享同一个 Gateway 网关进程的资源；应将 `maxConcurrent` 视为安全阀。
- `sessions_spawn` 始终是非阻塞的：它会立即返回 `{ status: "accepted", runId, childSessionKey }`。
- 子智能体上下文仅注入 `AGENTS.md` 和 `TOOLS.md`（不注入 `SOUL.md`、`IDENTITY.md`、`USER.md`、`MEMORY.md`、`HEARTBEAT.md` 或 `BOOTSTRAP.md`）。Codex 原生子智能体遵循相同的边界：`TOOLS.md` 保留在继承的 Codex 线程指令中，而仅属于父级的角色设定、身份和用户文件会作为轮次范围的协作指令注入，因此子智能体不会复制它们。
- 最大嵌套深度为 5（`maxSpawnDepth` 范围：1-5）。对于大多数使用场景，建议使用深度 2。
- `maxChildrenPerAgent` 限制每个会话的活动子智能体数量（默认值为 `5`，范围为 `1-20`）。

## 相关内容

- [会话工具和状态变更](/zh-CN/concepts/session-tool)
- [ACP 智能体](/zh-CN/tools/acp-agents)
- [智能体发送](/zh-CN/tools/agent-send)
- [后台任务](/zh-CN/automation/tasks)
- [多 Agent 沙盒工具](/zh-CN/tools/multi-agent-sandbox-tools)
