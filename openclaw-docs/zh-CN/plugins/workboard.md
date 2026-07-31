---
read_when:
    - 你想在 Control UI 中使用看板式工作板
    - 你正在启用或禁用内置的 Workboard 插件
    - 你希望在不使用外部项目管理器的情况下跟踪计划中的智能体工作
summary: 可选的仪表板工作看板，用于 Agent 自主管理卡片和会话交接
title: Workboard 插件
x-i18n:
    generated_at: "2026-07-26T06:54:59Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 8ec05c990c3559015780d9cb80f3ceedd7cc79db89ccf1afd65906c8c7630331
    source_path: plugins/workboard.md
    workflow: 16
---

Workboard 插件为 [Control UI](/zh-CN/web/control-ui) 添加了一个可选的看板式工作面板：适合智能体处理的工作卡片、向智能体分配任务，以及返回卡片对应任务、运行和仪表板会话的链接。

Workboard 刻意保持轻量：它跟踪一个 OpenClaw Gateway 网关的本地运维工作。它不能替代 GitHub Issues、Linear、Jira 或其他团队项目管理系统。

## 启用插件

Workboard 已内置，但默认禁用：

1. 在 Control UI 中打开 **Plugins**，或使用相对于已配置 Control UI 基础路径的 `/settings/plugins`。例如，基础路径为 `/openclaw` 时，使用 `/openclaw/settings/plugins`。
2. 找到 **Workboard** 并选择 **Enable**。由于 Workboard 已包含在 OpenClaw 中，因此无需执行 **Install** 操作。
3. 如果 UI 提示需要重启，请重启 Gateway 网关。

插件运行时加载后，Workboard 选项卡会出现在仪表板导航中。禁用时，该选项卡不会显示在导航中。如果插件已禁用或被 `plugins.allow`/`plugins.deny` 阻止，直接打开 `/workboard` 路由会显示插件不可用状态，而不是卡片数据。

对应的 CLI 工作流如下：

```bash
openclaw plugins enable workboard
openclaw gateway restart
openclaw dashboard
```

## 配置

Workboard 没有插件专用配置。使用标准插件条目启用或禁用它：

```json5
{
  plugins: {
    entries: {
      workboard: {
        enabled: true,
        config: {},
      },
    },
  },
}
```

```bash
openclaw plugins disable workboard
openclaw gateway restart
```

## 卡片字段

| 字段       | 值                                                                                                            |
| ----------- | ------------------------------------------------------------------------------------------------------------- |
| `status`    | `triage`、`backlog`、`todo`、`scheduled`、`ready`、`running`、`review`、`blocked`、`done`                     |
| `priority`  | `low`、`normal`、`high`、`urgent`                                                                             |
| `labels`    | 自由格式字符串                                                                                             |
| `agentId`   | 可选的已分配智能体                                                                                       |
| 关联引用 | 可选的任务、运行、会话或源 URL                                                                    |
| `execution` | 从卡片启动的 Codex/Claude 运行的可选元数据（引擎、模式、模型、会话、运行 ID、状态） |

卡片还包含简洁的元数据，用于记录尝试、评论、链接、验证材料、工件、自动化设置、附件、工作器日志、工作器协议状态、认领、诊断、通知、模板 ID、归档状态和过期会话检测，以及最近事件列表（`created`、`edited`、`moved`、`linked`、`specified`、`decomposed`、`claimed`、`heartbeat`、`execution_updated`、`attempt_started`、`attempt_updated`、`comment_added`、`link_added`、`proof_added`、`artifact_added`、`attachment_added`、`diagnostic`、`notification`、`dispatch`、`orchestration`、`protocol_violation`、`archived`、`unarchived`、`stale`）。借助这些元数据，操作员无需打开关联会话即可查看卡片在工作面板中的流转过程；它们是本地运维上下文，不能替代会话记录或 GitHub Issue 历史记录。

插件和 Control UI 使用同一套 Workboard 卡片契约。因此，仪表板刷新时会保留工作区来源和权限、认领状态、诊断操作以及通知序列号，而不是投影一份字段更少且仅供 UI 使用的卡片副本。未知的诊断类型、诊断严重性和通知类型会被忽略，直到两个界面均支持它们；它们绝不会被改写为其他有效状态。

打开的仪表板通过 `plugin.workboard.changed` 失效事件进行更新。每个事件仅包含存储纪元和修订号；随后 UI 会通过常规 `operator.read` RPC 重新读取规范卡片。多个修订会合并为一次后续读取。拖动、编辑或写入卡片时，Workboard 会推迟该读取，并在本地交互完成后恢复。重新连接始终会执行规范重新加载。系统不会例行轮询完整卡片，同时仍可使用 **Refresh** 进行手动恢复。

存在多个工作面板时，工具栏会提供 **Board** 筛选器；该筛选器由持久化的工作面板元数据提供支持，而不是只依据当前可见的卡片。因此，空工作面板和已归档工作面板仍可选择。没有显式工作面板 ID 的卡片属于规范的 `default` 工作面板。每个工作面板都有一个规范的 `/workboard/<boardId>` 页面，可以添加书签、共享或固定到侧边栏。此前发布的 `/workboard?board=<boardId>` 形式仍作为兼容性别名，并会在保留其他查询参数的同时重定向到该页面。选择 **All boards** 会返回 `/workboard`。

卡片存储在插件自己的 Gateway 网关状态中，并会与该 Gateway 网关的其他 OpenClaw 状态一起迁移（请参阅[存储](#storage)）。

## 从卡片开始工作

未关联的卡片可以直接开始工作：

- **Run Codex** / **Run Claude** 会使用显式指定的引擎启动一个由任务跟踪的智能体运行，发送卡片提示词，并将卡片标记为 `running`。Codex 运行使用 `openai/gpt-5.6-sol`；Claude 运行使用 `anthropic/claude-sonnet-4-6`。
- **Open Codex** / **Open Claude** 会创建一个关联的仪表板会话，但不会发送卡片提示词或移动卡片，适用于需要保持附加到工作面板的手动工作。

自主启动使用 Gateway 网关由任务跟踪的智能体运行路径（除非显式选择 Codex/Claude，否则使用默认智能体和模型）；随后 Workboard 会将生成的任务、运行 ID 和会话键关联回卡片。每次关联执行还会记录尝试摘要（引擎、模式、模型、运行 ID、时间戳、状态、滚动失败次数），以便持续显示重复失败。

仪表板会从 Gateway 网关任务账本刷新任务状态，并通过任务 ID、运行 ID 或关联会话键将任务与卡片匹配。处于排队或运行状态的任务会使卡片的生命周期保持活跃；已完成、失败、超时或取消的任务会使用与关联会话相同的同步规则，将卡片移向 `review` 或 `blocked`（请参阅[会话生命周期同步](#session-lifecycle-sync)）。

## 智能体工具

| 工具                                                                                                                                             | 用途                                                                                                                                                                                   |
| ------------------------------------------------------------------------------------------------------------------------------------------------ | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `workboard_list`                                                                                                                                 | 列出包含认领/诊断状态的紧凑卡片；可选择按看板筛选。                                                                                                                    |
| `workboard_read`                                                                                                                                 | 返回一张卡片及有界的工作节点上下文（备注、尝试、评论、链接、证明、工件、父级结果、受派人近期工作、活动诊断）。                               |
| `workboard_create`                                                                                                                               | 创建一张卡片，可选择指定父级、租户、Skills、看板、工作区元数据、幂等键、运行时限和重试预算。                                                             |
| `workboard_link`                                                                                                                                 | 将父卡片链接到子卡片。所有父级达到 `done` 之前，子卡片保持 `todo`；之后，调度提升会将其移至 `ready`。                                                     |
| `workboard_claim`                                                                                                                                | 为调用方智能体认领卡片；将 `backlog`/`todo`/`ready` 移至 `running`。                                                                                                        |
| `workboard_heartbeat`                                                                                                                            | 在较长时间的运行期间刷新认领心跳。                                                                                                                                          |
| `workboard_release`                                                                                                                              | 在完成、暂停或移交后释放认领；可将卡片移至下一个状态。                                                                                                |
| `workboard_complete` / `workboard_block`                                                                                                         | 用于最终摘要、证明、工件、已创建卡片清单（必须引用链接回已完成卡片的卡片）或阻塞原因的结构化生命周期工具。                 |
| `workboard_attachment_add` / `workboard_attachment_read` / `workboard_attachment_delete`                                                         | 在插件 SQLite 状态中存储小型卡片附件、在卡片上建立索引，并在工作节点上下文中公开。                                                                                         |
| `workboard_worker_log` / `workboard_protocol_violation`                                                                                          | 记录工作节点日志行，并在自动化工作节点停止且未调用 `workboard_complete`/`workboard_block` 时阻塞卡片。                                                           |
| `workboard_board_create` / `workboard_board_archive` / `workboard_board_delete`                                                                  | 管理持久化的看板元数据（显示名称、描述、归档状态、默认工作区）。                                                                                            |
| `workboard_runs`                                                                                                                                 | 返回卡片的持久化运行尝试历史记录。                                                                                                                                      |
| `workboard_specify`                                                                                                                              | 将粗略的分诊/待办卡片转化为明确的 `todo` 卡片；在卡片上记录规格摘要。                                                                                      |
| `workboard_decompose`                                                                                                                            | 将父级编排卡片拆分为相互链接的子卡片，并继承看板/租户元数据；可使用已创建卡片清单完成父级卡片。                                             |
| `workboard_notify_subscribe` / `workboard_notify_list` / `workboard_notify_events` / `workboard_notify_advance` / `workboard_notify_unsubscribe` | 管理通知订阅。事件读取可安全重放；`advance` 会移动持久化游标，使调用方能够恢复读取，而不会丢失或重复读取已完成/失败/过期的卡片事件。 |
| `workboard_boards` / `workboard_stats`                                                                                                           | 检查看板命名空间和队列统计信息。                                                                                                                                                 |
| `workboard_promote` / `workboard_reassign` / `workboard_reclaim`                                                                                 | 恢复或移交卡住的工作。                                                                                                                                                           |
| `workboard_comment` / `workboard_proof`                                                                                                          | 添加移交备注或附加证明/工件引用。                                                                                                                                    |
| `workboard_unblock`                                                                                                                              | 将阻塞的工作移回 `todo`。                                                                                                                                                         |
| `workboard_move`                                                                                                                                 | 将卡片移至其他状态；对于已认领的卡片，调用方必须具有其智能体认领权限范围。                                                                                                      |
| `workboard_dispatch`                                                                                                                             | 在不启动工作节点的情况下推动依赖提升或清理过期认领；工作节点通过 Gateway 网关或斜杠命令调度启动。                                                        |

证明状态是工作节点报告的结果，而非独立验证。`passed`
条目表示工作节点报告其命令或检查已成功；需要独立质量门禁的使用方应检查
所附的命令、URL 或工件，并运行自己的验证程序。`workboard_proof`
返回新记录的 `proofId`。当 `workboard_complete` 报告同一证明的
终止状态时，传入 `proofId`，以便原地解析待处理记录，而不丢失
其标识或时间戳。已有相同终止状态的证明会原样复用。未提供
`proofId` 的完成证明保持仅追加，因此后续重试不能仅因其命令或
备注相同而改写较早的历史记录。

除非调用方持有 `workboard_claim` 返回的认领令牌，否则已认领的卡片会
拒绝其他智能体发起的智能体工具变更。智能体工具或 Gateway RPC 调用返回的
每张卡片都会将 `metadata.claim.token` 脱敏为 `[redacted]`
（令牌本身仅由 `workboard_claim` 在顶层返回一次），因此仪表板操作员和
其他智能体可以检查认领状态，而永远不会看到可用的令牌。恢复通过
`workboard_promote`/`workboard_reassign`/`workboard_reclaim` 进行，这些操作
不需要令牌。

## 调度

调度在 Gateway 网关本地进行：它不会生成任意操作系统进程。执行仍由常规
OpenClaw 子智能体会话负责。一次调度流程：

1. 提升依赖项已就绪的卡片。
2. 在就绪卡片上记录调度元数据。
3. 阻塞认领已过期或运行已超时的卡片。
4. 将看板配置的分诊卡片标记为编排候选项。
5. 认领一小批就绪卡片，并通过
   Gateway 网关子智能体运行时启动工作节点运行。

工作节点会获得有界的卡片上下文，以及通过 Workboard 工具发送心跳、完成或
阻塞卡片所需的认领令牌。

工作区路径遵循调用方已有的文件系统权限。具有 `operator.write` 的
Gateway 网关客户端可以使用已配置的智能体工作区；`operator.admin`
客户端可以使用其他主机检出目录。沙箱隔离的智能体工具使用其沙箱工作区访问权限，
而未进行沙箱隔离且仅限工作区的工具使用其配置的工作区根目录。Workboard 在分配
工作区时记录该权限，并在调度时再次将其与当前调用方的权限取交集，因此持久化的
卡片无法扩大后续调用方的访问权限。对于具有显式主机工作区但未记录权限的旧卡片，
必须先重新保存该工作区，才能执行完整主机调度；没有主机路径的卡片在首次调度时
采用当前调用方的权限。

仅当目录或 Git 检出目录的仓库根目录与目标智能体工作区完全匹配时，绑定工作区的
调度才会接受它。工作树请求会限定到该目录并持久化为目录工作区，因此主机不会具现化
检出目录或执行仓库设置代码。目标工作节点必须为该确切工作区使用可写、非共享的
Docker 沙箱，且不得使用提升权限的执行、持久化的主机/节点 Exec 覆盖，也不得使用
未分类的插件和 MCP 工具。Workboard 会枚举其注册工具，而不是信任
`workboard_*` 前缀；如果热 Docker 容器的实时挂载/配置哈希已过期，调度将拒绝
该容器。调度会报告不兼容的目标策略，而不会启动隔离程度较低的工作节点。
完整主机调度可以面向其他本地检出目录，并保留常规的托管工作树设置。

工作区权限不会创建第二套卡片生命周期权限模型。获准变更 Workboard 卡片的调用方
可以在所有界面上手动将其移经相同的状态；只读工作区访问权限仅会阻止需要写入的
工作节点调度。

### 工作节点选择

每轮默认启动**最多 3 个工作进程**。就绪卡片依次按优先级、位置和创建时间排序。每轮对每个所有者/智能体只启动一张卡片，并跳过看板上已有运行中或审查中工作的所有者。已归档的卡片、有活动认领的卡片，以及状态不是 `ready` 的卡片，绝不会被选中用于启动工作进程（但仍可能受到调度数据侧操作的影响：清理过期认领、提升依赖项、清理超时）。

会话键按看板/卡片确定性生成，因此重复调度会路由回同一工作进程通道，而不是创建无关会话：

- 已分配的卡片：`agent:<agentId>:subagent:workboard-<boardId>-<cardId>`
- 未分配的卡片：`subagent:workboard-<boardId>-<cardId>`（Gateway 网关会解析已配置的默认智能体）

如果卡片被认领后无法启动工作进程，Workboard 会阻塞该卡片、清除认领、记录运行启动失败，并追加一行工作进程日志——可在仪表板、CLI JSON、智能体工具和卡片诊断中查看。

### 入口点

- 仪表板调度操作
- `openclaw workboard dispatch`
- 支持命令的渠道上的 `/workboard dispatch`

当 Gateway 网关可用时，这三种入口都使用 Gateway 网关的子智能体运行时。CLI 提供一种操作员回退机制：如果 Gateway 网关调用因连接/不可用错误而失败（对于旧版 Gateway 网关，也包括 `unknown method` 错误），并且未指定显式 `--url`/`--token` 目标，也没有已配置的远程 Gateway 网关（`OPENCLAW_GATEWAY_URL` 或 `gateway.mode: remote`）适用，CLI 会针对本地 SQLite 状态执行仅数据调度——它可以提升依赖项、清理过期认领并阻塞超时运行，但无法启动工作进程。可访问的 Gateway 网关所返回的身份验证、权限和验证失败不会被视为不可用；这些失败会显示为命令错误。如果指定了显式 `--url`/`--token` 目标，任何 Gateway 网关失败也都会显示为命令错误。

看板元数据可以设置 `autoDecompose`、`autoDecomposePerDispatch`、`defaultAssignee` 和 `orchestratorProfile`。OpenClaw 会记录此意图并将其公开到工作进程上下文中；实际的规格制定/任务拆分仍通过常规 Workboard 工具进行。

## CLI 和斜杠命令

```bash
openclaw workboard list [--board <id>] [--status <status>] [--include-archived] [--json]
openclaw workboard create "修复过期卡片生命周期" --priority high --labels bug,workboard
openclaw workboard show <card-id> [--json]
openclaw workboard move <card-id> --status <status> [--json]
openclaw workboard dispatch [--board <id>] [--json]
```

`list` 文本输出默认隐藏已归档卡片（`--include-archived` 可覆盖此行为）；`--json` 始终包含已归档卡片，与现有脚本使用的完整卡片契约一致。`show` 和 `move` 接受无歧义的 ID 前缀。`list`、`create`、`show` 和 `move` 始终直接读写本地插件状态。只有 `dispatch` 会调用正在运行的 Gateway 网关，并使用上述回退机制。

有关完整标志、JSON 输出、Gateway 网关回退行为、ID 前缀处理、调度选择规则和故障排查，请参阅 [Workboard CLI](/zh-CN/cli/workboard)。

`/workboard list`、`/workboard show <card-id>`、`/workboard create <title>`、`/workboard move <card-id> --status <status>` 和 `/workboard dispatch` 与 CLI 对应。对于任何获得授权的命令发送者，列出和显示都是读取操作。在聊天界面中，创建、移动和调度要求拥有所有者身份；Gateway 网关客户端则需要 `operator.write`/`operator.admin`。操作员手动移动卡片时，使用与仪表板拖放相同的认领覆盖行为。其工作树访问仍遵循上述相同的工作区边界。

## 会话生命周期同步

卡片可以链接到现有仪表板会话，也可以链接到从卡片启动工作时创建的会话。已链接的卡片会内联显示会话生命周期：运行中、已过期、已链接但空闲、已完成、失败或缺失。还可以在会话选项卡中使用**添加到 Workboard**捕获现有会话；卡片会链接到该会话，使用会话标签或最近的用户提示作为标题，并在可用时使用最近的用户提示和最新的助手回复初始化备注。

如果已链接的会话缺失，卡片会为保留上下文而继续保持链接，并仍提供启动控件，以便在新会话中重新启动。如果活动的已链接会话停止报告近期活动，Workboard 会将卡片标记为 `stale`，并将其存储为元数据，直到生命周期将其清除。

当卡片处于活动工作状态时，Workboard 会跟随已链接会话：

| 已链接会话状态                        | 卡片状态 |
| ------------------------------------- | ----------- |
| 活动中                                | `running`   |
| 已完成                                | `review`    |
| 失败、被终止、超时或中止              | `blocked`   |

**手动审查状态优先。** 将卡片移至 `review`、`blocked` 或 `done` 会停止该卡片的自动同步，直到将其移回 `todo` 或 `running`。

启动卡片时使用常规 Gateway 网关会话；Workboard 仅存储卡片元数据和链接。对话记录、模型选择和运行生命周期仍由常规会话系统负责。对活动的已链接卡片使用**停止**可中止活动运行——Workboard 会将该卡片标记为 `blocked`，使其继续显示以便后续处理。

新卡片可以从 Workboard 模板（`bugfix`、`docs`、`release`、`pr_review`、`plugin`）开始创建。模板会预填标题、备注、标签和优先级；模板 ID 会存储为卡片元数据。

## 仪表板工作流

1. 在 Control UI 中打开 Workboard 选项卡。
2. 创建一张卡片，填写标题、备注、优先级、标签、可选智能体和可选的已链接会话；或者打开会话并为现有会话选择**添加到 Workboard**。
3. 在列之间拖动卡片，或聚焦其紧凑状态控件，然后使用菜单或 ArrowLeft/ArrowRight。拖动期间，源卡片会变暗，可用的放置列会显示轮廓。
4. 从卡片启动工作，以创建或复用仪表板会话。
5. 智能体工作期间，从卡片打开已链接会话。
6. 让生命周期同步将运行中的工作移至 `review`/`blocked`，然后在接受后手动将卡片移至 `done`。

### 会话看板小组件

Workboard 为会话仪表板提供两个原生小组件（参阅[仪表板](/web/dashboards)）。智能体使用其 `dashboard` 工具并通过 `content: { kind: "plugin", pluginKind, props }` 固定这些小组件；它们会使用实时数据渲染为第一方 UI——无需沙箱框架或能力授权：

- `workboard:card` 与 `props: { cardId }` 结合使用时，会显示一张卡片及其状态控件、优先级和已分配智能体。
- `workboard:mini` 与可选的 `props: { boardId, limit }` 结合使用时，会显示各状态的数量以及排名靠前的就绪/运行中卡片，并链接到完整看板页面。如果没有 `boardId`，它会聚合所有看板；如果有 `boardId`，则限定到该看板（未指定显式看板 ID 而创建的卡片位于 `default`）。

## 诊断

诊断根据本地卡片元数据计算。内置检查会标记：

| 类型                        | 条件                                                                      |
| --------------------------- | ------------------------------------------------------------------------------ |
| `stranded_ready`            | 已分配的 `todo`/`backlog`/`ready` 卡片超过 1 小时未更新。             |
| `running_without_heartbeat` | `running` 卡片超过 20 分钟没有认领心跳或执行更新。 |
| `blocked_too_long`          | `blocked` 卡片超过 24 小时未更新。                                   |
| `repeated_failures`         | 卡片跟踪的失败次数达到 2 次或更多。                                |
| `missing_proof`             | `done` 卡片没有证明、工件或附件。                          |
| `orphaned_session`          | `running` 卡片有 `sessionKey`，但没有 `execution` 元数据。                |

## 权限

Gateway 网关 RPC 方法位于 `workboard.*` 下：

| 权限范围            | 方法                                                                                                                                                                                                                                                                                                                                                                            |
| ---------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `operator.read`  | `cards.list`、`cards.export`、`cards.diagnostics`、附件列表/获取、通知事件读取、`boards.list`、`cards.stats`、`cards.runs`                                                                                                                                                                                                                                       |
| `operator.write` | `cards.diagnostics.refresh`、创建/更新/移动/删除/评论/链接/链接依赖项/证明/工件、附件添加/删除、工作进程日志、协议违规、认领/心跳/释放/提升/重新分配/重新认领/完成/阻塞/解除阻塞、`cards.dispatch`、`cards.bulk`、归档、`boards.upsert`/`archive`/`delete`、`cards.specify`/`decompose`、通知订阅/删除/推进 |

没有 RPC 方法需要 `operator.admin`。以只读操作员权限连接的浏览器可以查看看板，但不能修改卡片。管理员权限范围会扩大可接受的 Workboard 主机路径，但不会改变可用的方法。

## 存储

Workboard 将持久数据存储在 OpenClaw 状态目录下由插件所有的关系型 SQLite 数据库中：看板、卡片、标签、生命周期事件、运行尝试、评论、依赖链接、证明、工件引用、附件元数据和二进制大对象、诊断、通知、工作进程日志、协议状态和订阅均存储在 Workboard 表中（而不是插件键值条目中）。卡片导出会保留看板叙事，但不会内联附件二进制大对象的内容。

在 `.28` 版本中使用过 Workboard 的安装可以运行 `openclaw doctor --fix`，将已发布的旧版插件状态命名空间（`workboard.cards`、`workboard.boards`、`workboard.notify`，以及存在时的 `workboard.attachments`）迁移到关系型数据库中。

## 故障排查

**选项卡显示 Workboard 不可用**

```bash
openclaw plugins inspect workboard --runtime --json
```

如果配置了 `plugins.allow`，请将 `workboard` 添加到其中。如果 `plugins.deny` 包含 `workboard`，请在启用插件前将其移除。

**卡片无法保存**

确认浏览器连接具有 `operator.write` 访问权限。只读操作员会话可以列出卡片，但无法创建、编辑、移动或删除卡片。

**启动卡片后未打开预期会话**

检查卡片的智能体 ID 和已链接会话，然后打开会话或聊天以查看实际运行状态。

**调度未启动工作进程**

确认至少有一张没有活动认领的 `ready` 卡片：

```bash
openclaw workboard list --status ready
```

如果 CLI 报告仅数据分派，请启动或重启 Gateway 网关并
重试——仅数据分派会更新本地工作看板状态，但无法启动
子智能体工作进程运行。如果同一负责人或智能体的另一张卡片
已在运行或等待审查，也可能跳过当前卡片；请先完成、
阻塞或释放该活动工作，再为同一负责人分派更多
工作。

## 相关内容

- [Control UI](/zh-CN/web/control-ui)
- [Workboard CLI](/zh-CN/cli/workboard)
- [插件](/zh-CN/tools/plugin)
- [管理插件](/zh-CN/plugins/manage-plugins)
- [会话](/zh-CN/concepts/session)
