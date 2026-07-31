---
read_when:
    - 你想了解 Task Flow 与后台任务之间的关系
    - 你在发布说明或文档中看到 Task Flow 或 openclaw tasks flow
    - 你想要检查或管理持久化的流程状态
summary: 后台任务之上的 Task Flow 编排层
title: 任务流程
x-i18n:
    generated_at: "2026-07-26T06:40:23Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 5ccc6acf58b4b44c2989e3061bff08dabce8ef385706102360c756a1286ddd1b
    source_path: automation/taskflow.md
    workflow: 16
---

Task Flow 是位于[后台任务](/zh-CN/automation/tasks)之上的编排层。Flow 是多步骤工作的持久记录，拥有自己的状态、JSON 状态数据、修订计数器和关联的任务记录。Flow 可在 Gateway 网关重启后继续存在；各个任务仍是分离式工作的基本单位。

## 何时使用 Task Flow

| 场景                                      | 使用方式                                    |
| ----------------------------------------- | ------------------------------------------- |
| 单个后台作业                              | 普通任务                                    |
| 由插件代码驱动的多步骤流水线              | Task Flow（托管模式）                       |
| 分离式 ACP 或子智能体生成                 | Task Flow（镜像模式，自动创建）             |
| 一次性提醒                                | 定时任务                                    |

## 同步模式

### 托管模式

托管 Flow 有一个控制器：插件代码通过插件运行时 Task Flow API，使用目标和必需的控制器 ID 创建 Flow，然后显式驱动它。

- 每个步骤都作为在该 Flow 下创建的后台任务运行；Flow 的所有者键和请求方来源会传递给子任务。
- 控制器在 `running`、`waiting` 和终止状态之间推进 Flow，并在 Flow 记录中存储任意 JSON 步骤状态。
- 每次变更都会传入 Flow 的预期修订版本。过期写入会因修订冲突而被拒绝，而不会覆盖更新的状态。
- 请求取消后，系统会拒绝新的子任务；当没有子任务仍处于活动状态时，Flow 会最终变为 `cancelled`。

示例：一个每周报告 Flow，其中 (1) 收集数据，(2) 生成报告，(3) 交付报告，每个步骤对应一个后台任务：

```
Flow：weekly-report
  步骤 1：gather-data     → 已创建任务 → 已成功
  步骤 2：generate-report → 已创建任务 → 已成功
  步骤 3：deliver         → 已创建任务 → 运行中
```

### 镜像模式

当分离式 ACP 或子智能体运行开始时（具有可交付完成结果的会话范围任务），OpenClaw 会自动创建一个镜像的单任务 Flow。Flow 记录会镜像其唯一的后端任务——状态、目标和时间信息——因此分离式生成操作无需控制器，即可获得稳定的 Flow 句柄，用于状态查询和重试界面。镜像 Flow 在 CLI 中显示同步模式 `task_mirrored`。

## Flow 状态

| 状态        | 含义                                                                       |
| ----------- | -------------------------------------------------------------------------- |
| `queued`    | 已创建，尚未开始推进                                                       |
| `running`   | Flow 正在主动推进                                                          |
| `waiting`   | 托管 Flow 停留在等待元数据上（计时器、外部事件）                           |
| `blocked`   | 某个步骤已结束，但没有可用结果；`blockedTaskId`/摘要会说明具体步骤      |
| `succeeded` | 已成功完成                                                                 |
| `failed`    | 因错误而完成                                                               |
| `cancelled` | 已请求取消，且所有子任务均已结束                                           |
| `lost`      | Flow 丢失了其权威后端状态                                                  |

## 持久状态和修订跟踪

Flow 记录与任务记录一起持久化到共享 SQLite 状态数据库（`~/.openclaw/state/openclaw.sqlite`，`flow_runs` 表）中，因此进度可在 Gateway 网关重启后继续保留。每次写入都会递增 Flow 的 `revision`；传入过期预期修订版本的并发写入方会遇到冲突，必须重新读取。SQLite 自动检查点和定期被动检查点会限制 WAL 的增长，并在关闭时执行截断检查点。旧安装中的旧版 `flows/registry.sqlite` 辅助文件由 `openclaw doctor` 导入。

## 取消行为

`openclaw tasks flow cancel` 会在 Flow 上设置持久的取消意图，取消其活动子任务，并拒绝新的托管子任务。当没有子任务仍处于活动状态时，Flow 会最终变为 `cancelled`——可能立即完成，也可能在子任务需要更长时间才能结束时，由维护扫描完成。该意图会持久化，因此即使所有子任务终止前 Gateway 网关重启，已取消的 Flow 仍会保持取消状态。

## CLI 命令

```bash
# 列出活动和最近的 Flow
openclaw tasks flow list [--status <status>] [--json]

# 显示特定 Flow 的详细信息
openclaw tasks flow show <lookup> [--json]

# 取消运行中的 Flow 及其活动任务
openclaw tasks flow cancel <lookup>
```

| 命令                              | 描述                                                                      |
| --------------------------------- | ------------------------------------------------------------------------- |
| `openclaw tasks flow list`        | 列出受跟踪的 Flow，包括同步模式、状态、修订版本、控制器和任务数量         |
| `openclaw tasks flow show <id>`   | 按 Flow ID 或所有者键检查单个 Flow，包括关联的任务                        |
| `openclaw tasks flow cancel <id>` | 取消运行中的 Flow 及其活动任务                                            |

`openclaw tasks audit`（过期或损坏的 Flow 检查结果）和 `openclaw tasks maintenance`（完成卡住的取消操作，并在 7 天后清理终止的 Flow）也涵盖 Flow。

## 可靠的定时工作流模式

对于市场情报简报等重复工作流，应将调度、编排和可靠性检查视为独立的层：

1. 使用[定时任务](/zh-CN/automation/cron-jobs)安排时间。
2. 当工作流需要基于先前上下文继续执行时，使用持久 cron 会话。
3. 使用 [Lobster](/zh-CN/tools/lobster) 实现确定性步骤、审批关卡和恢复令牌。
4. 使用 Task Flow 跨子任务、等待、重试和 Gateway 网关重启跟踪多步骤运行。

cron 示例结构：

```bash
openclaw cron add \
  --name "Market intelligence brief" \
  --cron "0 7 * * 1-5" \
  --tz "America/New_York" \
  --session session:market-intel \
  --message "Run the market-intel Lobster workflow. Verify source freshness before summarizing." \
  --announce \
  --channel slack \
  --to "channel:C1234567890"
```

当重复工作流需要有意保留历史记录、先前运行摘要或长期上下文时，使用 `--session session:<id>`，而不是 `isolated`。当每次运行都应从全新状态开始，并且工作流中已显式提供所有必需状态时，使用 `isolated`。

在工作流内部，将可靠性检查放在 LLM 摘要步骤之前：

```yaml
name: market-intel-brief
steps:
  - id: preflight
    command: market-intel check --json
  - id: collect
    command: market-intel collect --json
    stdin: $preflight.json
  - id: summarize
    command: market-intel summarize --json
    stdin: $collect.json
  - id: approve
    command: market-intel deliver --preview
    stdin: $summarize.json
    approval: required
  - id: deliver
    command: market-intel deliver --execute
    stdin: $summarize.json
    condition: $approve.approved
```

建议的预检项目：

- 浏览器可用性和配置文件选择，例如使用 `openclaw` 管理状态，或在需要已登录的 Chrome 会话时使用 `user`。请参阅[浏览器](/zh-CN/tools/browser)。
- 各个来源的 API 凭据和配额。
- 所需端点的网络可达性。
- 为智能体启用所需工具，例如 `lobster`、`browser` 和 `llm-task`。
- 为 cron 配置失败目标，以便预检失败可见。请参阅[定时任务](/zh-CN/automation/cron-jobs#delivery-and-output)。

建议为每个收集的项目提供以下数据溯源字段：

```json
{
  "sourceUrl": "https://example.com/report",
  "retrievedAt": "2026-04-24T12:00:00Z",
  "asOf": "2026-04-24",
  "title": "Example report",
  "content": "..."
}
```

让工作流在摘要生成前拒绝过期项目或将其标记为过期。LLM 步骤应仅接收结构化 JSON，并应要求其在输出中保留 `sourceUrl`、`retrievedAt` 和 `asOf`。当工作流中需要经过架构验证的模型步骤时，请使用 [LLM 任务](/zh-CN/tools/llm-task)。

对于可供团队或社区复用的工作流，将 CLI、`.lobster` 文件和所有设置说明打包为 Skill 或插件，并通过 [ClawHub](/clawhub) 发布。除非插件 API 缺少所需的通用能力，否则应将工作流特定的防护措施保留在该包中。

## Flow 与任务的关系

Flow 协调任务，而不是取代任务。单个 Flow 在其生命周期内可以驱动多个后台任务。使用 `openclaw tasks` 检查各个任务记录，使用 `openclaw tasks flow` 检查负责协调的 Flow。

## 相关内容

- [后台任务](/zh-CN/automation/tasks)——由 Flow 协调的分离式工作账本
- [CLI：任务](/zh-CN/cli/tasks)——`openclaw tasks flow` 的 CLI 命令参考
- [自动化概览](/zh-CN/automation)——快速了解所有自动化机制
- [定时任务](/zh-CN/automation/cron-jobs)——可将工作送入 Flow 的定时作业
