---
read_when:
    - 你想了解智能体具备哪些会话工具
    - 你想要配置跨会话访问或生成子智能体
    - 你想检查已生成子智能体的状态
summary: 用于跨会话状态、回忆、消息传递和子智能体编排的智能体工具
title: 会话工具
x-i18n:
    generated_at: "2026-07-26T06:40:55Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: ceaf48addc9fc57afe2f6428cda03ed8b19f4efce93b13b58b7ef493a41c62fe
    source_path: concepts/session-tool.md
    workflow: 16
---

OpenClaw 为智能体提供跨会话工作、检查状态和编排子智能体的工具。

## 可用工具

| 工具                 | 功能                                                                |
| -------------------- | --------------------------------------------------------------------------- |
| `sessions`           | 修补可见会话设置并管理全局会话组目录  |
| `sessions_list`      | 使用可选筛选条件（类型、标签、智能体、归档、预览）列出会话  |
| `sessions_search`    | 搜索可见会话的转录文本并返回匹配的摘录             |
| `sessions_history`   | 读取特定会话的转录文本                                   |
| `sessions_send`      | 在同一 Gateway 网关上运行另一个会话，并可选择等待                 |
| `conversations_list` | 列出稳定的外部对话地址                                 |
| `conversations_send` | 向一个确切的外部对话发送消息，而不运行本地会话     |
| `conversations_turn` | 向一个确切的外部对话发送消息，并等待与之关联的回复   |
| `sessions_spawn`     | 生成隔离的子智能体会话以执行后台工作                     |
| `sessions_yield`     | 结束当前轮次并等待后续子智能体结果               |
| `subagents`          | 列出或取消此会话树中的后台工作                         |
| `session_status`     | 显示 `/status` 风格的卡片，并可选择设置每会话模型覆盖 |

这些工具仍受当前工具配置文件和允许/拒绝策略约束。`tools.profile: "coding"` 包含完整的会话编排工具集。`tools.profile: "messaging"` 包含会话自助服务、设备发现、回忆、跨会话消息传递、外部对话工具，以及完整的生成生命周期（`sessions_spawn`、`sessions_yield` 和 `subagents`）。仅供 UI 使用的任务建议工具 `spawn_task` 和 `dismiss_task` 仍属于编码配置文件工具。

组、提供商、沙箱和按智能体配置的策略仍可在配置文件阶段之后移除这些工具。请从受影响的会话中使用 `/tools` 检查实际生效的工具列表。

## 列出和读取会话

`sessions_list` 返回聚焦于设备发现的行：会话键、智能体、类型、渠道、标签/标题/预览字段、父子关系、最后更新时间、归档/固定状态、状态版本、模型、上下文/总 token 数、运行状态，以及最后一次运行是否中止。可按 `kinds`（数组；接受的值：`main`、`group`、`cron`、`hook`、`node`、`other`）、确切的 `label`、确切的 `agentId`、`search` 文本或最近更新时间（`activeMinutes`）进行筛选。默认返回活动会话；改为传入 `archived: true` 可检查已归档会话。当需要邮箱式分类处理时，可设置 `includeDerivedTitles`、`includeLastMessage` 或 `messageLimit`（上限为 20）：分别为受可见性范围约束的派生标题、最后一条消息的预览片段，或每行有限数量的近期消息。交付路由、内部会话 ID、每次运行的计时/设置、成本估算和转录文本路径会被有意省略；这些所有者特定的详细信息请使用 `session_status`、对话工具和 `sessions_history`。仅为调用者根据已配置的会话工具可见性策略本就可见的会话生成派生标题和预览，因此不相关的会话仍会隐藏。当可见性受限时，`sessions_list` 会返回可选的 `visibility` 元数据，其中显示实际生效的模式，并警告结果可能受范围限制。

`sessions_history` 获取特定会话的对话转录文本。默认排除工具结果；传入 `includeTools: true` 可查看这些结果。使用 `limit` 获取最新且有界的尾部内容。需要分页元数据时传入 `offset: 0`，然后传入返回的 `nextOffset` 值，以便在较旧的 OpenClaw 转录文本窗口中向后翻页，而无需读取原始转录文本文件。显式偏移页面不会合并外部 CLI 回退导入；需要这种合并显示历史记录时，请使用默认的最新尾部视图（不传 `offset`）。

返回的视图会被有意限制范围并经过安全筛选：

- 回忆前会规范化 assistant 文本：
  - 移除 thinking 标签
  - 移除 `<relevant-memories>` / `<relevant_memories>` 脚手架块
  - 移除纯文本工具调用 XML 载荷块，例如 `<tool_call>...</tool_call>`、`<function_call>...</function_call>`、`<tool_calls>...</tool_calls>` 和 `<function_calls>...</function_calls>`，包括从未正常闭合的截断载荷
  - 移除降级后的工具调用/结果脚手架，例如 `[Tool Call: ...]`、`[Tool Result ...]` 和 `[Historical context ...]`
  - 移除泄漏的模型控制 token，例如 `<|assistant|>`、其他 ASCII `<|...|>` token，以及全角 `<｜...｜>` 变体
  - 移除格式错误的 MiniMax 工具调用 XML，例如 `<invoke ...>` / `</minimax:tool_call>`
- 返回前会遮盖类似凭证/token 的文本
- 截断长文本块
- 非常大的历史记录可能会丢弃较旧的行，或用 `[sessions_history omitted: message too large]` 替换过大的行
- 该工具会报告摘要标志，例如 `truncated`、`droppedMessages`、`contentTruncated`、`contentRedacted`、`bytes`，以及分页元数据

将返回的**会话键**（例如 `"main"`）与 `sessions_history`、`sessions_send` 和 `session_status` 一起使用。这些目标工具也可以解析已知的会话 ID，但 `sessions_list` 不会公开内部 ID。

如果需要确切的原始转录文本，请检查限定范围的 SQLite 转录文本行，而不要将 `sessions_history` 视为未经筛选的转储。

使用 [`sessions_search`](/zh-CN/concepts/session-search) 对可见的用户和 assistant 转录文本进行精确全文回忆。其结果包含用于后续 `sessions_history` 调用的 `sessionKey`；可见性筛选、片段遮盖和输出边界与历史记录边界一致。

## 管理会话设置和组

受所有者权限限制的 `sessions` 工具提供两个有界的自助服务界面：

- `action: "patch"` 默认更改当前会话，也可通过 `sessionKey` 选择另一个可见会话。它可以设置标签、侧边栏图标、固定/归档状态、模型和思考级别。它不提供重置、删除或压缩操作。
- `group_list`、`group_set`、`group_rename` 和 `group_delete` 管理全局有序会话组目录。`group_set` 会替换有序名称列表，而不是修补单个条目。

智能体选择的模型修补在该选择成功完成一次运行之前始终可以撤销。如果所选模型因身份验证、计费或找不到模型的故障而确定无法使用，OpenClaw 会恢复之前的模型并写入可见的系统注释。暂时性的速率限制、过载、超时、网络和服务器故障不会撤销该选择。

## 会话与对话

**会话**是本地模型上下文。**对话**是确切的外部地址，例如某个对等方、频道或线程。二者相关联，但不可互换：直接消息可以共享一个 `main` 会话，同时保留各自独立的对话地址。

`conversations_list` 为活动智能体返回不透明的 `conversationRef` 值。显式指定 `channel` 后，Gateway 网关还会从该渠道的本地目录刷新地址，例如已批准的 Reef 对等方；使用 `query` 可查找当前结果页以外的特定对等方。设备发现会将地址编入目录，但不会创建模型上下文会话；只有在交付或入站上下文需要时，才会创建支持会话。对话设备发现和交付仅限所有者使用，因为它们会使用 Gateway 网关的渠道凭证。使用 `conversations_send` 进行发送后不等待的交付。当远程回复属于当前模型轮次时，使用 `conversations_turn`：Gateway 网关会预留一个传输消息 ID，在传输 I/O 之前持久化交付操作和队列意图，并由工具返回关联的回复，而不是启动第二个本地智能体轮次。交付操作存在于模型转录文本之外；捕获的回复仅作为辅助工件保留，而模型上下文由工具结果持有。如果 Gateway 网关在排队后重启，交付可以恢复，但之后的回复会遵循普通入站分发流程，因为进程本地的等待器已不存在。未经请求的入站消息始终继续通过正常的渠道分发路径处理。

如果已有明确的原始渠道目标，或需要渠道特定操作，请使用共享的 `message` 工具。对话引用的范围限于活动智能体，应通过 `conversations_list` 获取，而不是根据会话键构造。

在代码模式下，对话工具会复用其确切的 Gateway 网关输出契约。单个 `exec` 单元格可以列出地址、选择返回的 `conversationRef`，并调用 `conversations_send` 或 `conversations_turn`；常规工具策略和审批仍适用于嵌套调用。

## 发送跨会话消息

`sessions_send` 在同一 Gateway 网关上运行另一个会话，并可选择等待响应。其 `sessionKey`、`label` 或 `agentId` 选择的是本地模型上下文，而不是外部目标。生成的回复仍可通过已建立的请求方或目标交付上下文发布；这一现有行为保持不变。若要进行确切的外部交付，请使用对话工具，或使用带有明确渠道和目标的 `message`。

- **发送后不等待：**将 `timeoutSeconds: 0` 设置为入队后立即返回。
- **等待回复：**设置超时时间并以内联方式获取响应。

线程范围的聊天会话（例如键以 `:thread:<id>` 结尾的会话）不是有效的 `sessions_send` 目标。请使用父频道会话键进行智能体间协调，以免工具路由的消息出现在面向用户的活动线程中。

消息和 A2A 后续回复会在接收提示词（`[Inter-session message ... isUser=false]`）和转录文本来源信息中标记为会话间数据。接收方智能体应将其视为通过工具路由的数据，而不是最终用户直接撰写的指令。

目标响应后，OpenClaw 可以运行**回复循环**，让智能体轮流发送消息，直至达到内置限制。目标智能体可以回复 `REPLY_SKIP` 以提前停止。

传入 `watch: true`，还可将发送方注册为目标的状态变更观察者：当其他参与者之后向目标发送直接的人工消息或更改其目标时，发送方会收到一条系统通知，其中指向 `session_status` `changesSince`。注册会在成功分发后进行，目标是实际收到消息的会话，并从其当前状态版本开始，因此只有后续更改才会产生通知。注册成功时，结果会报告 `watched: true`。请参阅[会话状态感知](/zh-CN/concepts/session-state)。

## 状态和编排辅助工具

`session_status` 是针对当前会话或另一个可见会话的轻量级 `/status` 等效工具。它会报告用量、时间、模型/运行时状态，以及存在关联后台任务时的上下文。与 `/status` 类似，它可以根据最新转录文本用量条目回填缺失的 token/缓存计数器，而 `model=default` 会清除每会话覆盖。使用 `sessionKey="current"` 表示调用者的当前会话；诸如 `openclaw-tui` 的可见客户端标签并不是会话键。

当路由元数据可用时，`session_status` 还会包含一个可见的 `Route context` JSON 块以及对应的结构化 `details` 字段。这些字段可区分会话键与当前正在处理实时运行的路由：

- `origin` 表示会话的创建位置；当旧状态缺少已存储的来源元数据时，则表示根据可投递会话键前缀推断出的提供商。
- `active` 表示当前的实时运行路由。仅针对当前正在处理的实时会话或当前会话报告该字段。
- `deliveryContext` 表示会话中存储的持久化投递路由。即使活跃界面不同，OpenClaw 仍可复用该路由进行后续投递。

## 会话状态变更

OpenClaw 会持久记录重要会话状态变更的信号日志（发送到受监视会话的人工直接消息、子运行结果、目标变更、压缩）。`sessions_list` 行和 `session_status` 会公开会话的 `stateVersion`，而 `session_status` 接受 `changesSince: <version>`，以返回该版本之后的类型化事件；当请求的版本早于保留的历史记录时，会通过准确的 `historyGap` 发出信号。当其他参与者更改受监视的会话时，监视者（自动监视的生成父会话，以及通过 `sessions_send watch: true` 显式注册的监视者）会收到一条合并后的状态过期通知。

状态变更事件会省略重复的会话/智能体 ID，仅公开对模型有用的有效负载字段（`outcome`、`channel` 或 `turns`）。事件摘要以及参与者/运行标识符仍可用于对账。

有关完整模型，请参阅[会话状态感知](/zh-CN/concepts/session-state)：事件类型、监视者注册、防刷屏通知协议、对账流程以及当前限制。

`sessions_yield` 会有意结束当前轮次，以便下一条消息可以是你正在等待的后续事件。当你生成子智能体后，希望完成结果作为下一条消息到达，而不是构建轮询循环时，请使用它。

`subagents` 是基于原生子智能体运行和共享后台任务账本的会话树视图。`action: "list"` 会报告活跃/近期的子智能体，以及限定范围的 ACP、CLI/媒体和定时任务。`action: "cancel"` 接受返回的 `taskId`，并且只能停止调用者所控制会话树内的工作；叶级子智能体无法取消其他会话的任务。

## 生成子智能体

默认情况下，`sessions_spawn` 会为后台任务创建一个隔离会话。它始终非阻塞，并会立即返回 `runId` 和 `childSessionKey`。原生子智能体运行会在子会话的第一条可见 `[Subagent Task]` 消息中接收委派任务，而系统提示词仅包含子智能体运行时规则和路由上下文。

主要选项：

- `runtime: "subagent"`（默认）或用于外部 harness 智能体的 `"acp"`。
- 用于覆盖子会话的 `model` 和 `thinking`。
- `thread: true`，用于将生成操作绑定到聊天线程（Discord、Slack 等）。
- `sandbox: "require"`，用于对子会话强制执行沙箱隔离。
- 对于需要当前请求者对话记录的原生子智能体，使用 `context: "fork"`；若要创建干净的子会话，请省略该选项或使用 `context: "isolated"`。`context: "fork"` 仅在与 `runtime: "subagent"` 配合使用时有效。除非 `threadBindings.defaultSpawnContext` 另有指定，否则绑定到线程的原生子智能体默认为 `context: "fork"`。
- `visible: true`，用于创建持久化仪表板会话，而不是隐藏的子智能体会话。可见生成支持显式模型、工作目录、同一智能体的对话记录分叉，以及可选的[托管工作树](/zh-CN/concepts/managed-worktrees)；确切的兼容性限制请参阅[子智能体](/zh-CN/tools/subagents#tool-parameters)。

默认情况下，叶级子智能体不会获得会话工具。当 `maxSpawnDepth >= 2` 时，深度为 1 的编排器子智能体还会获得 `sessions_spawn`、`subagents`、`sessions_list` 和 `sessions_history`，以便管理自己的子智能体。叶级运行仍不会获得递归编排工具。

完成后，通知步骤会将结果发布到请求者的渠道。完成结果投递会尽可能保留已绑定的线程/主题路由；如果完成来源只能标识渠道，OpenClaw 仍可复用请求者会话中存储的路由（`lastChannel` / `lastTo`）进行直接投递。

有关 ACP 特有的行为，请参阅 [ACP 智能体](/zh-CN/tools/acp-agents)。

## 可见性

会话工具具有范围限制，以约束智能体可以看到的内容：

| 级别   | 范围                                                      |
| ------- | ---------------------------------------------------------- |
| `self`  | 仅当前会话                                   |
| `tree`  | 当前会话 + 已生成会话；读取范围包括受监视的同智能体组 |
| `agent` | 此智能体的所有会话                                |
| `all`   | 所有会话（如已配置，可跨智能体）                   |

默认为 `tree`。无论如何配置，沙箱隔离会话都会被限制为 `tree`。
使用默认的 `session.dmScope: "main"` 时，组活动会使主会话能够读取受监视的
同智能体组会话。

## 延伸阅读

- [会话管理](/zh-CN/concepts/session)：路由、生命周期、维护
- [子智能体](/zh-CN/tools/subagents)：子会话生命周期和投递
- [ACP 智能体](/zh-CN/tools/acp-agents)：外部 harness 生成
- [多智能体](/zh-CN/concepts/multi-agent)：多智能体架构
- [Gateway 配置](/zh-CN/gateway/configuration)：会话工具配置选项

## 相关内容

- [会话管理](/zh-CN/concepts/session)
- [会话修剪](/zh-CN/concepts/session-pruning)
