---
read_when:
    - 你需要调试会话 ID、转录事件或会话行字段
    - 你正在更改自动压缩行为或添加“压缩前”整理操作
    - 你想要实现记忆刷新或静默系统轮次
summary: 深入解析：会话存储与转录记录、生命周期及（自动）压缩内部机制
title: 会话管理深入解析
x-i18n:
    generated_at: "2026-07-26T07:01:11Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 4ae02d49245768831abd17e1c2e5adacfa1a36673cef2a8a7a06a5300392b104
    source_path: reference/session-management-compaction.md
    workflow: 16
---

单个 **Gateway 网关进程**端到端管理会话状态。UI（macOS 应用、Web Control UI、TUI）从 Gateway 网关查询会话列表和令牌计数。在远程模式下，会话文件位于远程主机上，因此检查本地 Mac 上的文件无法反映 Gateway 网关实际使用的内容。

请先阅读概览文档：[会话管理](/zh-CN/concepts/session)、[压缩](/zh-CN/concepts/compaction)、[记忆概览](/zh-CN/concepts/memory)、[记忆搜索](/zh-CN/concepts/memory-search)、[会话修剪](/zh-CN/concepts/session-pruning)、[转录记录卫生](/zh-CN/reference/transcript-hygiene)；完整配置参考见 [Agent 配置](/zh-CN/gateway/config-agents)。

## 两个持久化层

1. **会话行（每个 Agent 独立的 SQLite）** - 键值映射 `sessionKey -> SessionEntry`。由 Gateway 网关管理的可变运行时状态。跟踪元数据：当前会话 ID、最近活动、开关和令牌计数器。
2. **转录事件（每个 Agent 独立的 SQLite）** - 仅追加的树形结构（条目包含 `id` + `parentId`）。存储对话、工具调用和压缩摘要；为后续轮次重建模型上下文。压缩检查点是已压缩后继转录记录上的元数据——新的压缩不会再写入一份 `.checkpoint.*.jsonl` 副本。

较旧的安装可能仍在 Agent 的 `sessions/`
目录下保留 `sessions.json` 文件。应将这些文件视为旧版会话行迁移输入或明确的
离线维护目标。Gateway 网关启动和 `openclaw doctor --fix` 会自动将活跃的
旧版行和转录历史导入每个 Agent 的 SQLite 存储。
需要明确的检查或验证证据时，请运行 `openclaw doctor --session-sqlite inspect
--session-sqlite-all-agents`，然后按照
[Doctor 迁移流程](/zh-CN/cli/doctor#session-sqlite-migration)操作。如果旧版转录
工件归档后迁移失败，请使用该流程中的 Doctor 恢复模式。
恢复过程使用迁移清单，仅还原受影响的已归档支持工件，并可在请求时准备
经过清理的 GitHub Issue 报告；它不会让活跃运行时重新读取 JSONL 文件。

除非相应界面需要任意历史访问，否则 Gateway 网关历史读取器不会将整个转录记录载入内存。首页历史记录、嵌入式聊天历史记录、重启恢复以及令牌/用量检查都使用来自 SQLite 的有界尾部读取。完整转录扫描通过异步转录索引执行，并在并发读取器之间共享。

## 磁盘位置

对于 Gateway 网关主机上的每个 Agent（通过 `src/config/sessions.ts` 解析）：

- 运行时会话行存储：`~/.openclaw/agents/<agentId>/agent/openclaw-agent.sqlite`
- 运行时转录行：`~/.openclaw/agents/<agentId>/agent/openclaw-agent.sqlite`
- 旧版/归档转录工件：`~/.openclaw/agents/<agentId>/sessions/`
- 旧版行迁移输入：`~/.openclaw/agents/<agentId>/sessions/sessions.json`

## 存储维护和磁盘控制

`session.maintenance` 控制 SQLite 会话行、SQLite 转录行、归档工件和轨迹附属文件的自动维护：

| 键                      | 默认值                | 说明                                                                                         |
| ----------------------- | --------------------- | -------------------------------------------------------------------------------------------- |
| `mode`                  | `"enforce"`           | 或 `"warn"`（仅报告，不修改）                                                       |
| `pruneAfter`            | `"30d"`               | 过期条目的时间阈值                                                                           |
| `maxEntries`            | `500`                 | 会话条目数量上限                                                                             |
| `resetArchiveRetention` | 保留（无时间阈值）    | `*.reset.*`/`*.deleted.*` 转录归档的时间阈值；设置持续时间即启用删除             |
| `maxDiskBytes`          | `10gb`                | 每个 Agent 的会话磁盘预算；`false` 表示禁用                                        |
| `highWaterBytes`        | `maxDiskBytes` 的 80% | 预算清理后的目标值                                                                           |

重置会推进活跃的 `sessionKey -> sessionId` 映射，但保留之前的 SQLite 会话、转录、轨迹和搜索行。该历史记录仍可通过相同的会话键搜索；普通条目和会话列表仅显示新的活跃映射。保留的重置历史记录受磁盘预算限制，而不受 `resetArchiveRetention` 限制；后者仅按时间清理归档工件。显式删除则不同：它会先写入并验证压缩的转录归档（zstd 可用时为 `*.jsonl.deleted.<timestamp>.zst`），然后才移除已删除会话的行。

`maxDiskBytes` 使用物理字节数执行限制：每个 Agent 的 SQLite 主文件、其 `-wal` 文件，以及 Agent 会话目录中计入统计的文件。它绝不会估算行 JSON 大小，也不会从总量中减去逻辑行大小。

Gateway 网关模型运行探测会话（键匹配 `agent:*:explicit:model-run-<uuid>`）使用独立且固定的 `24h` 保留期限。此修剪由压力触发：仅在达到会话条目维护/上限压力时运行，并且只在全局过期条目清理/上限步骤之前运行。其他显式会话不使用此保留期限。

当合计物理用量超过 `maxDiskBytes` 时，`mode: "enforce"` 首先回收可通过检查点释放的数据库空间，然后移除最旧的已保留重置/删除归档。如果用量仍高于 `highWaterBytes`，它会按 `sessions.updated_at` 遍历历史 SQLite 会话，从最旧的开始。历史会话是指其会话 ID 未被活跃会话条目、路由目标或已准入/进行中的运行引用。对于每个清理对象，清理过程会先写入压缩归档、执行 fsync 并回读，之后写事务才会移除会话行及其转录、轨迹、活跃状态、索引和 FTS 投影。其中包括含有轨迹事件但不含转录事件的会话。清理过程会在删除时重新检查路由、条目和准入引用，在处理每个归档或会话对象后重新测量物理用量，并在达到 `highWaterBytes` 时停止。

已提交的写入和删除会先落入 WAL。清理过程会对其执行检查点，使 WAL 能够立即缩小，然后使用增量 vacuum 从主文件中返还符合条件的空闲尾页；尚不可回收的页面会保留在主文件中，因此在下一次物理测量时仍计入统计。`mode: "warn"` 会报告当前物理超额量，但不会执行检查点、写入归档或删除行。

按需运行维护：

```bash
openclaw sessions cleanup --dry-run
openclaw sessions cleanup --enforce
```

维护过程会保留持久的外部对话指针，例如群组会话和线程范围的聊天会话；但合成运行时条目（定时任务、Hooks、Heartbeat、ACP、子智能体）在超过配置的时间、数量或磁盘预算后仍可能被移除。隔离的定时任务运行使用独立的 `cron.sessionRetention` 控制项，不受模型运行探测保留期限影响。

Gateway 网关的正常写入通过会话访问器进行，该访问器通过运行时写入器路径串行处理每个 Agent 的 SQLite 修改。运行时代码应优先使用 `src/config/sessions/session-accessor.ts` 中的访问器辅助函数；旧版 `sessions.json` 辅助函数是迁移和离线维护工具。当 Gateway 网关可访问时，非试运行的 `openclaw sessions cleanup` 和 `openclaw agents delete` 会将存储修改委托给 Gateway 网关，使清理任务加入同一写入器队列；`--store <path>` 是针对所选旧版存储的显式离线修复路径，并始终在本地运行（`--dry-run` 也是如此）。`maxEntries` 清理会针对生产规模的存储分批执行，因此存储可能暂时超过配置的上限，直到下一次高水位清理将其重写到上限以内。在 Gateway 网关启动期间，读取操作绝不会修剪条目或对条目应用上限——只有写入或 `openclaw sessions cleanup --enforce` 会执行这些操作；后者还会立即应用上限，并且即使未配置磁盘预算，也会修剪未被引用的旧版转录、检查点和轨迹工件。

OpenClaw 在 Gateway 网关写入期间不再自动创建 `sessions.json.bak.*` 轮换备份。当前架构拒绝旧版 `session.maintenance.rotateBytes` 键，`openclaw doctor --fix` 会从旧配置中移除该键。

转录修改使用会话写入队列处理 SQLite 转录目标：

会话写入锁使用固定的生产环境默认值。对应的
`OPENCLAW_SESSION_WRITE_LOCK_*` 环境变量仍可用于
进程级诊断和紧急覆盖。

### 切换到 SQLite 后降级

在运行较旧的文件存储型 OpenClaw 版本之前，请还原已归档的旧版转录工件：

```bash
openclaw doctor --session-sqlite restore --session-sqlite-all-agents
```

迁移会保留旧版 `sessions.json` 文件，以便支持和
回滚，但已导入 SQLite 的活跃转录 JSONL 文件会被
重命名到 `session-sqlite-import-archive/`。较旧的文件存储型运行时会沿用
`sessions.json` 中的 `sessionFile` 路径，因此必须在启动前还原这些工件。
还原过程使用迁移清单，仅移动原始路径缺失且已记录的归档
工件，并保留 SQLite 数据库以供后续恢复。

切换到 SQLite 后创建的会话仅存在于 SQLite 中，不会出现在
较旧的文件存储型运行时里。如果降级后再次升级，请重新运行 Doctor
检查和验证流程，使 OpenClaw 能够在导入前验证已还原的旧版
工件。

## 定时任务会话和运行日志

隔离的定时任务运行会创建自己的会话条目/转录记录，并使用专用的保留策略：

- `cron.sessionRetention`（默认 `"24h"`）会从存储中修剪旧的隔离定时任务运行会话；`false` 表示禁用。
- 运行历史记录为每个定时任务保留最新的 2000 个终止行。丢失的行仍保留其 24 小时清理窗口。

当定时任务强制创建新的隔离运行会话时，它会在写入新行之前清理先前的 `cron:<jobId>` 会话条目：它会保留安全偏好设置（思考/快速/详细/推理设置、标签、显示名称）以及用户明确选择的模型/身份验证覆盖，但丢弃环境对话上下文（渠道/群组路由、发送/队列策略、权限提升、来源、ACP 运行时绑定），从而避免新的隔离运行继承旧运行中过期的交付或运行时权限。

## 会话键（`sessionKey`）

`sessionKey` 标识当前所在的对话存储桶（路由 + 隔离）。规范规则见：[/concepts/session](/zh-CN/concepts/session)。

| 模式                         | 示例                                                        |
| ---------------------------- | ----------------------------------------------------------- |
| 主聊天/直接聊天（每个 Agent） | `agent:<agentId>:<mainKey>`（默认 `main`）                |
| 群组                         | `agent:<agentId>:<channel>:group:<id>`                                          |
| 房间/频道（Discord/Slack）   | `agent:<agentId>:<channel>:channel:<id>` 或 `...:room:<id>`                    |
| 定时任务                     | `cron:<job.id>`                                          |
| Webhook                      | `hook:<uuid>`（除非被覆盖）                            |

## 会话 ID（`sessionId`）

每个 `sessionKey` 都指向当前的 `sessionId`（继续该对话的 SQLite 转录标识）。决策逻辑位于 `src/auto-reply/reply/session.ts` 的 `initSessionState()` 中。

- **重置**（`/new`、`/reset`）会为该 `sessionKey` 创建新的 `sessionId`。
- **不自动重置**是默认设置。当前 `sessionId` 会继续，而压缩会将活动模型上下文限制在一定范围内。
- **每日重置**（`session.reset.mode: "daily"`）会在经过配置的本地小时边界（`session.reset.atHour`，默认值为 `4`）后的下一条消息到达时创建新的 `sessionId`。
- **空闲过期**（带有 `session.reset.idleMinutes` 的 `session.reset.mode: "idle"`，或旧版 `session.idleMinutes`）会在空闲时限过后有消息到达时创建新的 `sessionId`。如果同时配置了每日重置和空闲过期，则以先到期者为准。
- **Control UI 重连恢复**会在 Gateway 网关从操作员 UI 客户端收到匹配的 `sessionId` 时，为重连后的单次发送保留当前可见的会话。这是一次性信号；普通的过期发送仍会创建新的 `sessionId`。
- **系统事件**（Heartbeat、定时唤醒、Exec 通知、Gateway 网关内部记录）可能会修改会话行，但绝不会延长每日重置或空闲重置的新鲜度。重置轮换会在构建新提示词之前，丢弃上一会话中排队等待处理的系统事件通知。
- **父级分叉策略**在创建线程或子智能体分叉时使用 OpenClaw 的活动分支。如果该分支过大（超过固定的内部上限，目前为 100K tokens），OpenClaw 会使用隔离上下文启动子项，而不是失败或继承不可用的历史记录。大小计算是自动的且不可配置；旧版 `session.parentForkMaxTokens` 配置由 `openclaw doctor --fix` 移除。
- **操作员分叉**：`sessions.create { parentSessionKey, fork: true }` 会创建一个新会话，其转录从父级的当前状态分支而来（使用与生成子智能体相同的分叉机制，包括上述大小上限）。父级存在活动运行时会拒绝分叉；除非显式传入模型选择，否则会继承父级的模型选择；并将子项标记为 `forkedFromParent`，同时使用全新的 token 计数器。

## 会话存储架构

运行时存储将 `SessionEntry` 值保存在每个智能体的 SQLite 中。值类型为 `src/config/sessions.ts` 中的 `SessionEntry`。关键字段（非完整列表）：

- `sessionId`：用于寻址 SQLite 转录行的当前转录 ID
- `sessionStartedAt`：当前 `sessionId` 的开始时间戳；每日重置的新鲜度使用此值。旧版行可能会从 JSONL 会话标头中推导该值。
- `lastInteractionAt`：上次真实用户/渠道交互的时间戳；空闲重置的新鲜度使用此值，因此 Heartbeat、定时任务和 Exec 事件不会让会话持续保持活动状态。缺少此字段的旧版行会回退到恢复出的会话开始时间。
- `updatedAt`：上次修改存储行的时间戳，用于列表展示、修剪和内部记录，不是每日重置或空闲重置新鲜度的权威依据。
- `archivedAt`：可选的归档时间戳。已归档会话会连同其完整转录保留在存储中，并从常规活动列表中排除。
- `pinnedAt`：可选的置顶时间戳。活动的置顶会话排在未置顶会话之前；归档会话会清除其置顶状态。
- Codex 线程互操作：两个字段均遵循 Codex 线程管理结构——传输中的 `archived`/`pinned` 布尔值始终从时间戳派生并由服务端写入，与 Codex `threads.archived_at` 语义和 camelCase 序列化保持一致。OpenClaw 时间戳使用 Unix 纪元毫秒，而 Codex 使用 Unix 纪元秒，因此桥接器会在 `codex` 插件接缝处进行转换。Codex 尚无置顶 API（仅有 `thread/archive`/`thread/unarchive`）；在该 API 出现之前，置顶状态会保留在 OpenClaw 侧。届时，匹配的结构将使绑定会话能够以机械方式往返同步置顶状态。
- Codex 监管仅列出未归档的原生线程。只有在操作员明确确认没有其他 Codex 进程拥有某个 Gateway 网关本地的 `idle` 或 `notLoaded` 活动状态未知线程后，才能通过原生 `thread/archive` 将其归档；插件会先重新读取一次进程本地状态，随后该线程会从目录中消失。该读取无法证明另一个 App Server 进程没有使用此线程。OpenClaw 拒绝归档活动行和错误行；在节点桥接器能够负责完整的流式线程生命周期之前，无法使用配对节点归档。在原生 Codex 客户端中取消归档后，该线程可以再次显示。
- `lastReadAt` / `markedUnreadAt`：由 `sessions.patch { unread }` 在服务端写入的读取状态时间戳——`unread: false` 会记录一次读取（设置 `lastReadAt`，清除 `markedUnreadAt`）；`unread: true` 会将会话标记为未读，直到下一次读取。会话行会公开一个派生的 `unread` 布尔值：已显式标记为未读，或读取时间早于最近活动。尚未被标记为已读的会话保持 `unread: false`，因此现有安装升级后不会突然全部显示为未读。
- `lastActivityAt`：上次完成且应被视为未读活动的智能体运行时间戳（用户、渠道和定时任务运行）。Heartbeat 和内部事件轮次以及元数据补丁不会更新它；`updatedAt` 不是活动信号。
- `sessionFile`：为迁移/归档兼容性保留的旧版标记；活动运行时使用 SQLite 身份
- `chatType`：`direct | group | room`
- `provider`、`subject`、`room`、`space`、`displayName`：群组/渠道标签元数据
- 开关：`thinkingLevel`、`verboseLevel`、`reasoningLevel`、`elevatedLevel`、`sendPolicy`（按会话覆盖）
- 模型选择：`providerOverride`、`modelOverride`、`authProfileOverride`
- Token 计数器（尽力而为/取决于提供商）：`inputTokens`、`outputTokens`、`totalTokens`、`contextTokens`
- `compactionCount`：此会话键完成自动压缩的次数
- `memoryFlushAt` / `memoryFlushCompactionCount`：上次压缩前记忆刷新时的时间戳和压缩次数

Gateway 网关是权威来源：随着会话运行，它可能会重写或重新加载条目。对于旧版文件后端安装，请使用
`openclaw doctor --session-sqlite import --session-sqlite-all-agents` 进行迁移，而不是
编辑 `sessions.json` 并期望运行时继续读取该文件。

## 转录事件结构

转录由 OpenClaw 会话访问器管理，并通过基于身份的辅助函数公开给运行时代码。事件流仅可追加：

- 第一条：会话标头——`type: "session"`、`id`、`cwd`、`timestamp`，以及可选的 `parentSession`。
- 随后：包含 `id` + `parentId` 的条目（树结构）。

重要条目类型：

- `message`：用户/助手/toolResult 消息
- `custom_message`：扩展注入的消息，_会_进入模型上下文（当 `display: true` 时在 TUI 中呈现，当 `display: false` 时完全隐藏）
- `custom`：扩展状态，_不会_进入模型上下文（用于在重新加载后保留扩展状态）
- `compaction`：包含 `firstKeptEntryId` 和 `tokensBefore` 的持久化压缩摘要
- `branch_summary`：导航树分支时持久化的摘要

OpenClaw 有意不“修正”转录；Gateway 网关使用 `SessionManager` 读写它们。

## 上下文窗口与跟踪的 token

这是两个不同的概念：

1. **模型上下文窗口**：每个模型的硬性上限（模型可见的 token）。该值来自模型目录，并可通过配置覆盖。
2. **会话存储计数器**：写入会话行的滚动统计信息（用于 `/status` 和仪表板）。`contextTokens` 是运行时估算/报告值——不要将其视为严格保证。

有关限制的更多信息：[/reference/token-use](/zh-CN/reference/token-use)。

## 压缩：它是什么

压缩会将较早的对话总结为转录中持久化的 `compaction` 条目，并完整保留最近的消息。压缩后，后续轮次会看到压缩摘要以及 `firstKeptEntryId` 之后的消息。与会话修剪不同，压缩是**持久的**——请参阅 [/concepts/session-pruning](/zh-CN/concepts/session-pruning)。

嵌入式 OpenClaw 压缩默认继承会话的思考级别。设置 `agents.defaults.compaction.thinkingLevel` 可为摘要调用使用单独的级别；运行时会将其限制在各个具体压缩模型或回退模型支持的范围内。原生 Codex App Server 的压缩由其自身负责紧凑请求，且无法接受按压缩设置的思考级别覆盖，因此 OpenClaw 会发出警告，并将该设置留给 Codex。

压缩后重新注入 AGENTS.md 章节仍需通过 `agents.defaults.compaction.postCompactionSections` 选择启用。插件可以通过 `before_prompt_build` 添加其他提示词上下文。

### 分块边界和工具配对

将长转录拆分为压缩块时，OpenClaw 会将助手工具调用与其匹配的 `toolResult` 条目保持配对：

- 如果按 token 比例拆分的边界会落在工具调用及其结果之间，OpenClaw 会将边界移至助手工具调用消息处，而不是拆散二者。
- 如果末尾的工具结果块原本会导致该块超过目标大小，OpenClaw 会保留该待处理工具块，并保持未摘要的尾部完整。
- 已中止/出错的工具调用块不会使待定拆分保持开放状态。

## 自动压缩何时发生

嵌入式 OpenClaw 智能体中有两个触发条件：

1. **溢出恢复**：模型返回上下文溢出错误（`request_too_large`、`context length exceeded`、`input exceeds the maximum number of tokens`、`input token count exceeds the maximum number of input tokens`、`input is too long for the model`、`ollama error: context length exceeded` 以及其他提供商格式的变体）——先压缩，然后重试。当提供商报告尝试使用的 token 数量时，OpenClaw 会将观测到的数量转发给溢出恢复压缩；如果提供商确认发生溢出但没有提供可解析的数量，OpenClaw 会向压缩引擎和诊断功能传递一个仅略微超出预算的合成数量。如果溢出恢复仍然失败，OpenClaw 会显示明确的操作指导，并保留当前会话映射，而不是悄悄轮换到新的会话 ID——请重试该消息、运行 `/compact`，或运行 `/new`。
2. **阈值维护**：成功完成一个轮次后，当当前上下文超过模型窗口减去 OpenClaw 为提示词和下一次模型输出内置的预留空间时触发。

这两个触发条件之外还会运行另外两个保护机制：

- **预检本地压缩**：设置 `agents.defaults.compaction.maxActiveTranscriptBytes`（字节数或类似 `"20mb"` 的字符串），当活动记录达到该大小时，在开始下一次运行前触发本地压缩。这是用于控制本地重新打开成本的大小保护机制，而不是原始归档；常规语义压缩仍会运行，并且需要 `truncateAfterCompaction`，以便压缩后的摘要成为新的后继记录。
- **轮次中途预检**：设置 `agents.defaults.compaction.midTurnPrecheck.enabled: true`（默认值为 `false`）以添加工具循环保护机制。在追加工具结果后、下一次模型调用前，OpenClaw 会使用与轮次开始时相同的预检预算逻辑来估算提示词压力。如果上下文已无法容纳，保护机制不会就地压缩，而是发出结构化的轮次中途预检信号，停止当前提示词提交，并让外层运行循环使用现有恢复路径（当截断超大工具结果足以解决问题时进行截断，否则触发已配置的压缩模式并重试）。它同时适用于 `default` 和 `safeguard` 压缩模式，包括由提供商支持的保护性压缩。它独立于 `maxActiveTranscriptBytes`：字节大小保护机制在轮次开始前运行，而轮次中途预检则在之后、追加新工具结果后运行。

## 压缩设置

```json5
{
  agents: {
    defaults: {
      compaction: {
        enabled: true,
        keepRecentTokens: 20000,
      },
    },
  },
}
```

OpenClaw 会为嵌入式运行强制保留内置余量，并根据活动模型的上下文窗口设置上限，使其无法占用全部提示词预算。这样既可防止小上下文本地模型从第一个 token 起就进入压缩，又能为多轮维护操作（例如记忆刷新）留出足够空间。

手动 `/compact` 会遵循显式的 `agents.defaults.compaction.keepRecentTokens`，并保留运行时的近期尾部切分点。如果未显式指定保留预算，手动压缩将成为硬检查点，重建后的上下文会从新摘要开始。

启用 `truncateAfterCompaction` 后，OpenClaw 会在压缩后将活动记录轮换为压缩后的后继记录。分支/恢复检查点操作会使用该压缩后继记录；旧版压缩前检查点文件在仍被引用时保持可读。

## 可插拔压缩提供商

插件通过插件 API 上的 `registerCompactionProvider()` 注册压缩提供商。当 `agents.defaults.compaction.provider` 设置为已注册的提供商 ID 时，保护扩展会将摘要生成委托给该提供商，而不是使用内置的 `summarizeInStages` 流水线。

- `provider`：已注册压缩提供商插件的 ID。保持未设置可使用默认 LLM 摘要生成。设置 `provider` 会强制启用 `mode: "safeguard"`。
- 提供商会收到与内置路径相同的压缩指令和标识符保留策略，并且在提供商输出后，保护机制仍会保留近期轮次和拆分轮次的后缀上下文。
- 内置保护性摘要会结合新消息重新提炼先前的摘要，而不是逐字保留完整的先前摘要。
- 保护模式默认启用摘要质量审计；设置 `qualityGuard.enabled: false` 可跳过输出格式错误时的重试行为。
- 如果提供商失败或返回空结果，OpenClaw 会自动回退到内置 LLM 摘要生成。调用方显式触发的中止/超时信号会被重新抛出而非吞掉，因此始终会遵循取消操作。

来源：`src/plugins/compaction-provider.ts`、`src/agents/agent-hooks/compaction-safeguard.ts`。

## 用户可见界面

- 任何聊天会话中的 `/status`
- `openclaw status`（CLI）
- `openclaw sessions` / `openclaw sessions --json`
- Gateway 网关日志（`pnpm gateway:watch` 或 `openclaw logs --follow`）：`embedded run auto-compaction start` + `complete`
- 详细模式：`🧹 Auto-compaction complete` 加上压缩次数

## 静默维护（`NO_REPLY`）

OpenClaw 支持用于后台任务的“静默”轮次，此时用户不应看到中间输出。

- 助手以精确的静默 token `NO_REPLY` / `no_reply` 开始输出，表示“不向用户发送回复”。OpenClaw 会在传递层中移除/抑制此内容。
- 精确静默 token 抑制不区分大小写：当整个载荷仅包含静默 token 时，`NO_REPLY` 和 `no_reply` 都会被识别。
- 从 `2026.1.10` 起，如果部分分块以 `NO_REPLY` 开头，OpenClaw 还会抑制草稿/输入状态流式传输，避免静默操作在轮次中途泄露部分输出。
- 这仅适用于真正的后台/不传递轮次，不能作为处理普通可操作用户请求的捷径。

## 压缩前记忆刷新

在自动压缩前，OpenClaw 可以运行一个静默智能体轮次，将持久状态写入磁盘（例如 Agent 工作区中的 `memory/YYYY-MM-DD.md`），以免压缩清除关键上下文。它会监控会话上下文使用量；当使用量超过低于压缩阈值的软阈值时，会使用精确的静默 token `NO_REPLY` / `no_reply` 发送一条静默的“立即写入记忆”指令，因此用户不会看到任何内容。

配置（`agents.defaults.compaction.memoryFlush`），完整参考见 [/gateway/config-agents](/zh-CN/gateway/config-agents#agentsdefaultscompaction)：

| 键                          | 默认值           | 说明                                                                                                                                   |
| --------------------------- | ---------------- | -------------------------------------------------------------------------------------------------------------------------------------- |
| `enabled`                   | `true`           |                                                                                                                                        |
| `model`                     | 未设置           | 仅用于刷新轮次的精确提供商/模型覆盖，例如 `ollama/qwen3:8b`                                                   |
| `softThresholdTokens`       | `4000`           | 低于压缩阈值并触发刷新的间隔                                                                               |
| `forceFlushTranscriptBytes` | 未设置（已禁用） | 当记录文件达到此字节大小（或类似 `"2mb"` 的字符串）时强制刷新，即使 token 计数器已过期；`0` 表示禁用 |

注意：

- 内置提示词和系统提示词包含 `NO_REPLY` 提示，以抑制传递。
- 设置 `model` 后，刷新轮次会使用该模型，而不会继承活动会话的回退链，因此仅限本地的维护操作失败时不会静默回退到付费对话模型。
- 每个压缩周期仅运行一次刷新（在会话行中跟踪）。
- 刷新仅针对嵌入式 OpenClaw 会话运行；CLI 后端和 Heartbeat 轮次会跳过它。
- 当会话工作区为只读（`workspaceAccess: "ro"` 或 `"none"`）时，会跳过刷新。
- 有关工作区文件布局和写入模式，请参阅[记忆](/zh-CN/concepts/memory)。

OpenClaw 在扩展 API 中公开了 `session_before_compact` 钩子，但上述刷新逻辑位于 Gateway 网关侧（`src/auto-reply/reply/memory-flush.ts`、`src/auto-reply/reply/agent-runner-memory.ts`），而不在该钩子上。

## 故障排查清单

- **会话键错误？** 从[/concepts/session](/zh-CN/concepts/session)开始，并确认 `/status` 中的 `sessionKey`。
- **存储与记录不匹配？** 通过 `openclaw status` 确认 Gateway 网关主机和存储路径。
- **频繁压缩？** 检查模型的上下文窗口（过小会导致频繁压缩）和工具结果膨胀（调整会话修剪）。
- **小型本地模型的每个提示词似乎都会溢出？** 确认提供商报告了正确的模型上下文窗口。只有在已知该窗口时，OpenClaw 才能限制有效保留余量。
- **静默轮次发生泄露？** 确认回复以精确的静默 token `NO_REPLY` 开头（不区分大小写），并且当前使用的构建版本包含流式传输抑制修复（`2026.1.10`+）。

## 相关内容

- [会话管理](/zh-CN/concepts/session)
- [会话修剪](/zh-CN/concepts/session-pruning)
- [上下文引擎](/zh-CN/concepts/context-engine)
- [Agent 配置参考](/zh-CN/gateway/config-agents)
