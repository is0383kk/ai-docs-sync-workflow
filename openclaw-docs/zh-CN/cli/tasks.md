---
read_when:
    - 你想要检查、审计或取消后台任务记录
    - 你正在 `openclaw tasks flow` 下记录 Task Flow 命令的文档
summary: '`openclaw tasks` 的 CLI 参考（后台任务账本和 Task Flow 状态）'
title: '`openclaw tasks`'
x-i18n:
    generated_at: "2026-07-26T06:44:58Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: b03a4aa9fab12b6e5773259a76a1e89fd6e6398c73e5b0533a31e5e3a3894f9c
    source_path: cli/tasks.md
    workflow: 16
---

检查持久化后台任务和 Task Flow 状态。不带子命令时，
`openclaw tasks` 等同于 `openclaw tasks list`。

有关生命周期和交付模型，请参阅[后台任务](/zh-CN/automation/tasks)；有关完整的问题描述，请参阅其中的 `tasks audit` 部分。

## 用法

```bash
openclaw tasks
openclaw tasks list
openclaw tasks list --runtime acp
openclaw tasks list --status running
openclaw tasks show <lookup>
openclaw tasks notify <lookup> state_changes
openclaw tasks cancel <lookup>
openclaw tasks audit
openclaw tasks maintenance
openclaw tasks maintenance --apply
openclaw tasks flow list
openclaw tasks flow show <lookup>
openclaw tasks flow cancel <lookup>
```

## 根选项

| 标志               | 描述                                                                                        |
| ------------------ | -------------------------------------------------------------------------------------------------- |
| `--json`           | 输出 JSON。                                                                                       |
| `--runtime <name>` | 按类型筛选：`subagent`、`acp`、`cron` 或 `cli`。                                               |
| `--status <name>`  | 按状态筛选：`queued`、`running`、`succeeded`、`failed`、`timed_out`、`cancelled` 或 `lost`。 |

## 子命令

### `list`

```bash
openclaw tasks list [--runtime <name>] [--status <name>] [--json]
```

按从新到旧的顺序列出已跟踪的后台任务。

### `show`

```bash
openclaw tasks show <lookup> [--json]
```

按任务 ID、运行 ID 或会话键显示单个任务。

### `notify`

```bash
openclaw tasks notify <lookup> <done_only|state_changes|silent>
```

更改正在运行的任务的通知策略。

### `cancel`

```bash
openclaw tasks cancel <lookup>
```

取消正在运行的后台任务。

### `audit`

```bash
openclaw tasks audit [--severity <warn|error>] [--code <name>] [--limit <n>] [--json]
```

显示过期、丢失、交付失败或存在其他不一致情况的任务和
Task Flow 记录。保留至 `cleanupAfter` 的丢失任务属于警告；
已过期或未标记时间戳的丢失任务属于错误。

`--code` 接受任务代码（`stale_queued`、`stale_running`、`lost`、
`delivery_failed`、`missing_cleanup`、`inconsistent_timestamps`）和 Task
Flow 代码（`restore_failed`、`stale_waiting`、`stale_blocked`、
`cancel_stuck`、`missing_linked_tasks`、`blocked_task_missing`）。有关各代码的
严重级别和触发条件详情，请参阅[后台任务](/zh-CN/automation/tasks)。

### `maintenance`

```bash
openclaw tasks maintenance [--apply] [--json]
```

预览或应用任务和 Task Flow 协调、清理标记、修剪以及过期的 cron 运行会话注册表清理。

对于 cron 任务，协调过程会先使用持久化的运行日志/任务状态，再将旧的活动任务标记为
`lost`，因此已完成的 cron 运行不会仅因内存中的 Gateway 网关运行时状态消失
而成为误报的审计错误。离线 CLI 审计无法作为 Gateway 网关进程本地 cron
活动任务集的权威依据。带有运行 ID/来源 ID 的 CLI 任务在其活动 Gateway 网关运行上下文
消失时会被标记为 `lost`，即使旧的子会话行仍然存在。

应用维护时，还会修剪超过 7 天的 `cron:<jobId>:run:<uuid>` 会话
注册表行，同时保留当前正在运行的 cron
任务，并且不会改动非 cron 会话行。

### `flow`

```bash
openclaw tasks flow list [--status <name>] [--json]
openclaw tasks flow show <lookup> [--json]
openclaw tasks flow cancel <lookup>
```

检查或取消任务账本下的持久化 Task Flow 状态。
`flow list --status` 接受 `queued`、`running`、`waiting`、`blocked`、
`succeeded`、`failed`、`cancelled` 或 `lost`。

## 相关内容

- [CLI 参考](/zh-CN/cli)
- [后台任务](/zh-CN/automation/tasks)
