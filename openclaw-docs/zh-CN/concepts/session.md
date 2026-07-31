---
read_when:
    - 你想了解会话路由和隔离机制
    - 你想为多用户设置配置私信范围
    - 你正在调试每日或空闲会话重置问题
summary: OpenClaw 如何管理对话会话
title: 会话管理
x-i18n:
    generated_at: "2026-07-26T06:43:16Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: de85fe5a623bdbc6d5564d822b39e9077a582b0816b62ab30d2f7245bd097000
    source_path: concepts/session.md
    workflow: 16
---

OpenClaw 会根据每条入站消息的来源（私信、群聊、定时任务等），将其路由到一个**会话**。所有会话状态均由 **Gateway 网关**管理；UI 客户端通过查询 Gateway 网关获取会话数据。

对于个人智能体的默认模式——所有私信渠道共享一个持续滚动的对话，群组活动和后台工作也会汇入其中——请参阅[主会话](/concepts/main-session)。

## 消息的路由方式

| 来源          | 行为                  |
| --------------- | ------------------------- |
| 私信 | 默认共享会话 |
| 群聊     | 按群组隔离        |
| 房间/频道  | 按房间隔离         |
| 定时任务       | 每次运行使用新会话     |
| Webhooks        | 按 hook 隔离         |

## 私信隔离

默认情况下，所有私信共享一个会话以保持连续性，这适用于单用户设置。

<Warning>
如果有多个人可以向你的智能体发送消息，请启用私信隔离。否则，所有用户都会共享同一个对话上下文，因此 Alice 的私信会对 Bob 可见。
</Warning>

```json5
{
  session: {
    dmScope: "per-channel-peer", // 按渠道 + 发送者隔离
  },
}
```

`session.dmScope` 选项：

| 值                      | 行为                                                 |
| -------------------------- | -------------------------------------------------------- |
| `main`（默认）           | 所有私信共享[主会话](/concepts/main-session) |
| `per-peer`                 | 跨渠道按发送者隔离                       |
| `per-channel-peer`         | 按渠道 + 发送者隔离（推荐）                |
| `per-account-channel-peer` | 按账号 + 渠道 + 发送者隔离                    |

<Tip>
如果同一个人通过多个渠道联系你，请使用 `session.identityLinks` 将其身份映射到一个规范的对端 ID，使其共享一个会话。
</Tip>

### 停靠已关联的渠道

停靠命令可将当前直接聊天会话的回复路由移至另一个已关联的渠道，而无需启动新会话。有关示例、配置和故障排查，请参阅[频道停靠](/zh-CN/concepts/channel-docking)。

使用 `openclaw security audit` 验证你的设置。

## 无痕会话

无痕会话只能从 Control UI 的 **New thread** 屏幕创建。在启动线程前开启 **Incognito**，即可将其会话条目、记录和压缩状态保存在进程内存中，而不是写入磁盘。Gateway 网关重启后，该线程会消失；它不会运行 OpenClaw 的自动记忆刷新，并且在重置或删除时不会创建记录归档。由 Codex 支持的运行也会以临时模式启动其 harness 线程，因此 Codex 不会写入 rollout 或本地会话状态文件；其他模型提供商使用 HTTP API，不会在 OpenClaw 中保留本地提供商记录。

`incognito-` 段保留用于仪表板、子智能体和隐藏的内部会话键；`openclaw doctor --fix` 会重命名任何发生冲突的旧版持久键。

无痕模式不会限制智能体的常规工具。明确要求保存信息或任何由工具驱动的文件写入，仍可能将数据持久化到无痕会话存储之外。你配置的模型提供商仍会处理你发送的消息，诊断日志记录保持不变，OpenClaw 仍会记录不含内容的审计元数据，例如 HMAC 引用。

在多用户 Gateway 网关上，无痕线程仅对具有管理员权限范围的连接可见，并且绝不会通过其他会话的智能体会话工具或记录搜索显示。这可以防止存储系统和其他由 Gateway 网关中介的用户访问这些线程，但无法防止 Gateway 网关所有者或进程操作员访问，因为他们始终可以观察实时会话。

## 跨对话记忆

独立的记录控制每个对话的本地历史记录。对于个人智能体或完全可信的智能体，`memory.search.rememberAcrossConversations: true` 会添加一个可选的检索步骤，用于检索该智能体的其他私密对话；它不会合并这些对话的记录。

私密的直接对话和持久的显式 UI 对话可以相互提供相关上下文。群组和频道在两个方向上均保持隔离：其记录不能作为私密回忆来源，并且这些对话中的回复也不会接收私密记录上下文。当前对话也会被排除，因为其历史记录已加载。

此设置不会更改会话键、私信范围、路由、投递或 `tools.sessions.visibility`。`MEMORY.md` 和 `memory/*.md` 中的共享工作区记忆也会保持现有行为。当前记忆提供商必须支持受保护的私密记录回忆；Lossless Claw 等上下文引擎保持独立，并可与其同时运行。有关设置和运行时详情，请参阅[主动记忆](/zh-CN/concepts/active-memory#remember-across-conversations)。

## 会话生命周期

会话会持续复用，直到你手动重置，或选择启用自动重置策略：

- **不自动重置**（默认 `mode: "none"`）- 会话保持相同的 `sessionId`；随着对话增长，压缩会管理活跃上下文。
- **每日重置**（`mode: "daily"`）- 选择在 Gateway 网关主机上配置的本地小时（`session.reset.atHour`，默认 `4`，0-23）开始新会话。每日新鲜度基于当前 `sessionId` 的开始时间，而不是之后的元数据写入时间。
- **空闲重置**（`mode: "idle"`）- 选择在无活动达到 `session.reset.idleMinutes` 后开始新会话。空闲新鲜度基于上次真实的用户/渠道交互，因此 Heartbeat、定时任务和 Exec 系统事件不会使会话保持活跃。
- **手动重置** - 在聊天中输入 `/new` 或 `/reset`。`/new <model>` 还会切换模型。

同时配置每日重置和空闲重置时，以最先过期者为准。Heartbeat、定时任务、Exec 和其他系统事件轮次可能会写入会话元数据，但这些写入不会延长每日或空闲重置的新鲜度。当重置轮换会话时，旧会话排队中的系统事件通知会被丢弃，以免过时的后台更新被添加到新会话的第一个提示词之前。

具有活跃的提供商自有 CLI 会话的会话同样默认不自动重置。如果这些会话应按计时器过期，请使用 `/reset` 或显式配置 `session.reset`。

先全局选择启用自动重置，然后按聊天类型或渠道进行覆盖：

```json5
{
  session: {
    reset: { mode: "daily", atHour: 4 },
    resetByType: {
      group: { mode: "idle", idleMinutes: 120 },
      thread: { mode: "daily", atHour: 6 },
    },
    resetByChannel: {
      discord: { mode: "idle", idleMinutes: 10080 },
    },
  },
}
```

`resetByType` 支持 `direct`、`group` 和 `thread`。Doctor 会将旧版 `dm` 条目迁移到 `direct`，并将 `session.idleMinutes` 迁移到 `session.reset.idleMinutes`；schema 会拒绝这两种已停用的形式。

## 状态存储位置

- **运行时会话行：**`~/.openclaw/agents/<agentId>/agent/openclaw-agent.sqlite`
- **已归档的记录文件：**`~/.openclaw/agents/<agentId>/sessions/`
- **旧版行迁移来源：**`~/.openclaw/agents/<agentId>/sessions/sessions.json`

每个智能体的 SQLite 数据库中的会话行会分别保存以下生命周期时间戳：

- `sessionStartedAt`：当前 `sessionId` 的开始时间；每日重置使用此时间。
- `lastInteractionAt`：上次延长空闲生命周期的用户/渠道交互时间。
- `updatedAt`：上次存储行变更时间；可用于列出和清理，但不能作为每日/空闲重置新鲜度的权威依据。

从旧版安装迁移时，Gateway 网关启动和 `openclaw doctor
--fix` 会自动将旧版 `sessions.json` 行以及活跃的 JSONL 记录历史导入 SQLite。对于缺少 `sessionStartedAt` 的行，如果旧版记录 JSONL 会话标头可用，则会从中解析该值。如果较旧的行还缺少 `lastInteractionAt`，则空闲新鲜度会回退到该会话的开始时间，而不是之后的簿记写入时间。如果需要显式检查或验证证据，请使用 `openclaw doctor --session-sqlite inspect
--session-sqlite-all-agents` 和 [Doctor 迁移序列](/zh-CN/cli/doctor#session-sqlite-migration)。

## 会话维护

OpenClaw 通过 `session.maintenance` 限制会话存储随时间增长，默认值如下：

```json5
{
  session: {
    maintenance: {
      mode: "enforce", // "enforce" 执行清理；"warn" 仅报告
      pruneAfter: "30d",
      maxEntries: 500,
    },
  },
}
```

对于生产规模的 `maxEntries` 限制，Gateway 网关运行时写入会使用一个较小的高水位缓冲区，并分批清理回配置的上限。Gateway 网关启动期间，会话存储读取不会清理或限制条目，因此启动和隔离的定时任务会话无需承担完整存储清理的开销。`openclaw sessions cleanup --enforce` 会立即应用该上限。

Gateway 网关模型运行探测会话默认是短期会话。匹配 `agent:*:explicit:model-run-<uuid>` 的行使用固定的 `24h` 保留期，但清理受压力触发：仅当达到会话条目维护/上限压力时，才会删除过期的探测行，并且会先于更广泛的过期条目年龄截止条件和条目上限执行。普通的直接会话、群组会话、线程会话、定时任务会话、hook 会话、Heartbeat 会话、ACP 会话和子智能体会话不会继承此 24h 保留期。

维护过程会保留持久的外部对话指针，包括群组会话和线程范围的聊天会话，同时仍允许合成的定时任务、hook、Heartbeat、ACP 和子智能体条目随时间过期。

已归档会话由用户暂存，不受任何自动维护路径影响，包括按时间清理、条目上限、模型运行清理和磁盘预算驱逐。它们会一直保持归档状态，直到你取消归档或显式删除。

如果你之前使用了私信隔离，之后又将 `session.dmScope` 恢复为 `main`，请使用 `openclaw sessions cleanup --dry-run --fix-dm-scope` 预览过时的按对端键控的私信行。应用同一标志会停用这些旧的直接私信行，并将其记录保留为已删除归档。

使用 `openclaw sessions cleanup --dry-run` 预览任何维护运行。

## 检查会话

| 命令                    | 显示内容                                           |
| -------------------------- | ----------------------------------------------- |
| `openclaw status`          | 会话存储路径和最近活动          |
| `openclaw sessions --json` | 所有会话（使用 `--active <minutes>` 筛选） |
| 聊天中的 `/status`          | 上下文用量、模型和开关               |
| `/context list`            | 系统提示词中的内容                    |

## 延伸阅读

- [会话搜索](/zh-CN/concepts/session-search) - 在过去的记录中进行全文回忆
- [会话清理](/zh-CN/concepts/session-pruning) - 裁剪工具结果
- [压缩](/zh-CN/concepts/compaction) - 总结长对话
- [会话工具](/zh-CN/concepts/session-tool) - 用于跨会话工作的智能体工具
- [会话管理深度解析](/zh-CN/reference/session-management-compaction) -
  存储 schema、记录、发送策略、来源元数据和高级配置
- [多智能体](/zh-CN/concepts/multi-agent) - 跨智能体的路由和会话隔离
- [后台任务](/zh-CN/automation/tasks) - 分离式工作如何创建带有会话引用的任务记录
- [频道路由](/zh-CN/channels/channel-routing) - 入站消息如何路由到会话

## 相关内容

- [会话清理](/zh-CN/concepts/session-pruning)
- [会话工具](/zh-CN/concepts/session-tool)
- [命令队列](/zh-CN/concepts/queue)
