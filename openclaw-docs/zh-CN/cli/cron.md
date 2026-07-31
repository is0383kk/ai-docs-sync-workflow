---
read_when:
    - 你需要定时任务和唤醒功能
    - 你正在调试定时任务的执行和日志
summary: '`openclaw cron` 的 CLI 参考（调度并运行后台任务）'
title: 定时任务
x-i18n:
    generated_at: "2026-07-26T05:42:27Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 5989a7558f4ae2f046480b6a52e3fa296c95d47b14b11c5bad709fea4af6af3e
    source_path: cli/cron.md
    workflow: 16
---

# `openclaw cron`

管理 Gateway 网关调度器的定时任务。

<Tip>
运行 `openclaw cron --help` 查看完整的命令功能。有关概念指南，请参阅[定时任务](/zh-CN/automation/cron-jobs)。
</Tip>

<Note>
所有定时任务变更操作（`add`/`create`、`update`/`edit`、`remove`、`run`）都需要 `operator.admin`。命令载荷直接在 Gateway 网关进程中运行，而不是作为智能体的 `tools.exec` 工具调用；`tools.exec.*` 和 Exec 审批仍会管控模型可见的 Exec 工具。
</Note>

## 快速创建任务

`openclaw cron create` 是 `openclaw cron add` 的别名。创建新任务时，先放调度表达式，再放提示词：

```bash
openclaw cron create "0 7 * * *" \
  "汇总夜间更新。" \
  --name "晨间简报" \
  --agent ops
```

当任务应通过 POST 发送最终载荷，而不是投递到聊天目标时，请使用 `--webhook <url>`：

```bash
openclaw cron create "0 18 * * 1-5" \
  "以 JSON 格式汇总今天的部署。" \
  --name "部署摘要" \
  --webhook "https://example.invalid/openclaw/cron"
```

对于在 OpenClaw 定时任务中运行、且不启动隔离智能体/模型运行的确定性 shell 风格任务，请使用 `--command`：

```bash
openclaw cron create "*/15 * * * *" \
  --name "队列深度探测" \
  --command "scripts/check-queue.sh" \
  --command-cwd "/srv/app" \
  --announce \
  --channel telegram \
  --to "-1001234567890"
```

`--command <shell>` 存储 `argv: ["sh", "-lc", <shell>]`。如需精确执行 argv，请使用 `--command-argv '["node","scripts/report.mjs"]'`。命令任务会捕获 stdout/stderr、记录常规定时任务历史记录，并通过与隔离任务相同的 `announce`、`webhook` 或 `none` 投递模式路由输出。仅打印 `NO_REPLY` 的命令将被抑制。

## 会话

`--session` 接受 `main`、`isolated`、`current` 或 `session:<id>`。

<AccordionGroup>
  <Accordion title="会话键">
    - `main` 绑定到智能体的主会话。
    - `isolated` 为每次运行创建新的对话记录和会话 ID。
    - `current` 绑定到创建时的活动会话。
    - `session:<id>` 固定到明确指定的持久会话键。

  </Accordion>
  <Accordion title="隔离会话语义">
    隔离运行会重置环境对话上下文。渠道和群组路由、发送/排队策略、权限提升、来源以及 ACP 运行时绑定都会针对新运行重置。安全偏好设置以及用户明确选择的模型或身份验证覆盖项可以跨运行保留。
  </Accordion>
</AccordionGroup>

## 投递

`openclaw cron list` 和 `openclaw cron show <job-id>` 可预览解析后的投递路由。对于 `channel: "last"`，预览会显示路由是从主会话还是当前会话解析的，或者是否会以关闭方式失败。

带提供商前缀的目标可以消除未解析公告渠道的歧义。例如，当省略 `delivery.channel` 或其值为 `last` 时，`to: "telegram:123"` 会选择 Telegram。只有已加载插件声明的前缀才是提供商选择器。如果明确指定了 `delivery.channel`，前缀必须与该渠道匹配；将 `channel: "whatsapp"` 与 `to: "telegram:123"` 组合使用会被拒绝。`imessage:` 和 `sms:` 等服务前缀仍属于渠道自有的目标语法。

<Note>
隔离的 `cron add` 任务默认使用 `--announce` 投递。使用 `--no-deliver` 可将输出保留在内部。`--deliver` 仍作为 `--announce` 的已弃用别名保留。
</Note>

### 投递所有权

隔离定时任务的聊天投递由智能体和运行器共同负责：

- 当聊天路由可用时，智能体可以使用 `message` 工具直接发送。
- 仅当智能体未直接发送到解析后的目标时，`announce` 才会回退投递最终回复。
- `webhook` 将最终载荷发布到 URL。
- `none` 禁用运行器的回退投递。

使用 `cron add|create --webhook <url>` 或 `cron edit <job-id> --webhook <url>` 设置 webhook 投递。不要将 `--webhook` 与 `--announce`、`--no-deliver`、`--channel`、`--to`、`--thread-id` 或 `--account` 等聊天投递标志组合使用。

`cron edit <job-id>` 可以使用 `--clear-channel`、`--clear-to`、`--clear-thread-id` 和 `--clear-account` 取消设置各个投递路由字段（每个选项与其对应的设置标志组合时都会被拒绝）。`--no-deliver` 仅禁用运行器回退投递，与之不同的是，这些选项会移除已存储的字段，使任务再次从默认值解析其路由的相应部分。

`--announce` 是最终回复的运行器回退投递。`--no-deliver` 会禁用该回退，但在聊天路由可用时不会移除智能体的 `message` 工具。

从活动聊天创建的提醒会保留实时聊天投递目标，用于回退公告投递。内部会话键可以是小写；不要将它们作为 Matrix 房间 ID 等区分大小写的提供商 ID 的事实依据。

### 失败投递

失败通知按以下顺序解析：

1. 任务上的 `delivery.failureDestination`。
2. 全局 `cron.failureDestination`。
3. 任务的主要公告目标（当上述两项均未解析为具体目的地时）。

<Note>
仅当主要投递模式为 `webhook` 时，主会话任务才可以使用 `delivery.failureDestination`。隔离任务在所有模式下均接受它。
</Note>

隔离定时任务运行会将运行级智能体故障视为任务错误，即使没有生成回复载荷也是如此，因此模型/提供商故障仍会增加错误计数器并触发失败通知。

命令定时任务不会启动隔离的智能体轮次。退出代码为零时记录为 `ok`；非零退出、信号、超时或无输出超时记录为 `error`，并且可以触发相同的失败通知路径。

如果隔离运行在首次模型请求前超时，`openclaw cron show` 和 `openclaw cron runs` 会包含特定于阶段的错误，例如 `setup timed out before runner start`，或者包含一条指出最后已知启动阶段的停滞消息（例如 `context-engine`）。对于基于 CLI 的提供商，预模型看门狗会保持活动状态，直到外部 CLI 轮次开始，因此会话查找、钩子、身份验证、提示词和 CLI 设置停滞都会被报告为预模型定时任务故障。

## 调度

### 单次任务

`--at <datetime>` 安排单次运行。没有偏移量的日期时间会被视为 UTC，除非你同时传入 `--tz <iana>`，它会按给定时区解释挂钟时间。

<Note>
默认情况下，单次任务成功后会被删除。使用 `--keep-after-run` 可保留任务。
</Note>

### 重复任务

重复任务在连续发生错误后使用指数重试退避：30s、1m、5m、15m、60m。下一次运行成功后，调度恢复正常。

跳过的运行与执行错误分开跟踪。它们不会影响重试退避，但 `openclaw cron edit <job-id> --failure-alert-include-skipped` 可以让失败警报包含重复的运行跳过通知。

对于以本地已配置模型提供商（基准 URL 位于环回地址、专用网络或 `.local`）为目标的隔离任务，定时任务会在启动智能体轮次前执行轻量级提供商预检：在 `/api/tags` 探测 `api: "ollama"` 提供商；在 `/models` 探测其他与 OpenAI 兼容的本地提供商（`api: "openai-completions"`，例如 vLLM、SGLang、LM Studio）。如果端点不可访问，该运行会记录为 `skipped`，并在后续调度时重试；每个端点的可达性结果会缓存 5 分钟，以免针对同一本地服务器的大量任务通过重复探测对其造成压力。

定时任务、待处理运行时状态和运行历史记录存储在共享 SQLite 状态数据库中。旧版 `jobs.json`、`<name>-state.json` 和 `runs/*.jsonl` 文件会被导入一次，并使用 `.migrated` 后缀重命名。导入后，请使用 `openclaw cron add|edit|remove` 编辑调度，而不是编辑 JSON 文件。

### 手动运行

`openclaw cron run <job-id>` 默认强制运行，并在手动运行进入队列后立即返回。成功响应包含 `{ ok: true, enqueued: true, runId }`。使用返回的 `runId` 查看后续结果：

```bash
openclaw cron run <job-id>
openclaw cron runs --id <job-id> --run-id <run-id>
```

当脚本需要阻塞，直到该次入队运行记录终止状态时，请添加 `--wait`：

```bash
openclaw cron run <job-id> --wait --wait-timeout 10m --poll-interval 2s
```

使用 `--wait` 时，CLI 仍会先调用 `cron.run`，然后针对返回的 `runId` 轮询 `cron.runs`。仅当运行以 `ok` 状态结束时，命令才以 `0` 退出。如果运行以 `error` 或 `skipped` 结束、Gateway 网关响应不包含 `runId`，或者 `--wait-timeout` 到期（默认为 `10m`，默认每 `2s` 轮询一次），命令将以非零状态退出。`--poll-interval` 必须大于零。

<Note>
如果你只想在任务当前已到期时运行手动命令，请使用 `--due`。如果 `--due --wait` 未将运行加入队列，命令会返回常规的未运行响应，而不会进行轮询。
</Note>

## Models

`cron add|edit --model <ref>` 为任务选择允许使用的模型。`cron add|edit --fallbacks <list>` 设置每个任务的回退模型，例如 `--fallbacks openrouter/gpt-4.1-mini,openai/gpt-5`；传入 `--fallbacks ""` 可执行没有回退模型的严格运行。`cron edit <job-id> --clear-fallbacks` 移除每个任务的回退覆盖项。`cron edit <job-id> --clear-model` 移除每个任务的模型覆盖项，使任务遵循正常的定时任务模型选择优先级（优先使用已存储的定时任务会话覆盖项，否则使用智能体/默认模型）；它不能与 `--model` 组合使用。`cron add|edit --thinking <level>` 设置每个任务的思考覆盖项；`cron edit <job-id> --clear-thinking` 将其移除，使任务遵循正常的定时任务思考优先级，并且不能与 `--thinking` 组合使用。

<Warning>
如果模型不被允许或无法解析，定时任务会使该次运行失败并给出明确的验证错误，而不会回退到任务的智能体或默认模型选择。
</Warning>

定时任务的 `--model` 是**任务主模型**，而不是聊天会话的 `/model` 覆盖项。这意味着：

- 选定的任务模型失败时，已配置的模型回退仍然适用。
- 如果存在每个任务的载荷 `fallbacks`，它会替换已配置的回退列表。
- 空的每任务回退列表（任务载荷/API 中的 `--fallbacks ""` 或 `fallbacks: []`）会使定时任务严格运行。
- 当任务包含 `--model` 但未配置回退列表时，OpenClaw 会传入明确的空回退覆盖项，避免将智能体主模型作为隐藏重试目标追加。
- 本地提供商预检会逐个检查已配置的回退项，然后才将定时任务运行标记为 `skipped`。

`openclaw doctor` 会报告已设置 `payload.model` 的任务，包括提供商命名空间计数以及与 `agents.defaults.model` 的不匹配项。当实时聊天与调度任务之间的身份验证、提供商或计费行为看起来不同时，请使用此检查。

### 隔离定时任务模型优先级

隔离定时任务按以下顺序解析活动模型：

1. Gmail 钩子覆盖项。
2. 每个任务的 `--model`。
3. 已存储的定时任务会话模型覆盖项（当用户选择过时）。
4. 智能体或默认模型选择。

### 快速模式

隔离的定时任务快速模式遵循解析后的实时模型选择。默认应用模型配置 `params.fastMode`，但存储的会话 `fastMode` 覆盖仍优先于配置。当解析后的模式为 `auto` 时，截止时间使用所选模型的 `params.fastAutoOnSeconds` 值，默认为 60 秒。

### 实时模型切换重试

如果隔离运行抛出 `LiveSessionModelSwitchError`，定时任务会在重试前，为当前运行持久化切换后的提供商和模型（以及存在时切换后的身份验证配置文件覆盖）。外层重试循环在初次尝试后最多进行两次切换重试，随后中止，而不是无限循环。

## 运行输出和拒绝

### 抑制过时确认

隔离的定时任务轮次会抑制仅包含过时确认的回复。如果第一个结果只是临时状态更新，并且没有任何后代子智能体运行负责生成最终答案，定时任务会在交付前重新提示一次，以获取实际结果。

### 抑制静默令牌

如果隔离的定时任务运行仅返回静默令牌（`NO_REPLY` 或 `no_reply`），定时任务会同时抑制直接出站交付和后备排队摘要路径，因此不会向聊天回发任何内容。

### 结构化拒绝

隔离的定时任务运行将嵌入式运行提供的结构化执行拒绝元数据（编码为 `SYSTEM_RUN_DENIED` 或 `INVALID_REQUEST` 的致命 Exec 工具错误）用作权威拒绝信号。它们也会识别节点主机的 `UNAVAILABLE` 包装器，其中嵌套的结构化错误携带上述代码之一。

除非嵌入式运行也提供结构化拒绝元数据，否则定时任务不会将最终输出中的文字或看似要求审批的拒绝语句归类为拒绝，因此普通智能体文本不会被视为受阻命令。

`cron list` 和运行历史记录会显示拒绝原因，而不是将受阻命令报告为 `ok`。

## 保留

保留行为：

- `cron.sessionRetention`（默认值为 `24h`，或设为 `false` 以禁用）会清理已完成的隔离运行会话。
- 运行历史记录为每个定时任务保留最新的 2000 条终态记录。丢失的记录仍采用标准的 24 小时丢失任务清理窗口。

## 迁移旧任务

<Note>
如果你的定时任务创建于当前交付和存储格式之前，请运行 `openclaw doctor --fix`。Doctor 会规范化旧版定时任务字段（`jobId`、`schedule.cron`、包括旧版 `threadId` 在内的顶层交付字段、有效负载 `provider` 交付别名），并在移除该配置键之前，将 `notify: true` Webhook 后备任务从已停用的原始 `cron.webhook` 值迁移为显式 Webhook 交付。已经向聊天发送通知的任务会保留该交付方式，并获得一个完成 Webhook 目标。若没有旧版 Webhook，对于没有迁移目标的任务，会移除不起作用的顶层 `notify` 标记（现有交付方式保持不变），因此 `doctor --fix` 不会再反复发出相关警告。
</Note>

## 常用编辑操作

在不更改消息的情况下更新交付设置：

```bash
openclaw cron edit <job-id> --announce --channel telegram --to "123456789"
```

禁用隔离任务的交付：

```bash
openclaw cron edit <job-id> --no-deliver
```

为隔离任务启用轻量级引导上下文：

```bash
openclaw cron edit <job-id> --light-context
```

通知到特定渠道：

```bash
openclaw cron edit <job-id> --announce --channel slack --to "channel:C1234567890"
```

通知到 Telegram 论坛话题：

```bash
openclaw cron edit <job-id> --announce --channel telegram --to "-1001234567890" --thread-id 42
```

创建采用轻量级引导上下文的隔离任务：

```bash
openclaw cron create "0 7 * * *" \
  "汇总夜间更新。" \
  --name "轻量级晨间简报" \
  --session isolated \
  --light-context \
  --no-deliver
```

`--light-context` 仅适用于隔离的智能体轮次任务。对于定时任务运行，轻量级模式会保持引导上下文为空，而不是注入完整的工作区引导集合。

创建具有精确 argv、cwd、env、stdin 和输出限制的命令任务：

```bash
openclaw cron create "*/30 * * * *" \
  --name "仓位导出" \
  --command-argv '["node","scripts/export-position.mjs"]' \
  --command-cwd "/srv/app" \
  --command-env "NODE_ENV=production" \
  --command-input '{"mode":"summary"}' \
  --timeout-seconds 120 \
  --no-output-timeout-seconds 30 \
  --output-max-bytes 65536 \
  --webhook "https://example.invalid/openclaw/cron"
```

## 常用管理命令

手动运行和检查：

```bash
openclaw cron list
openclaw cron list --agent ops
openclaw cron get <job-id>
openclaw cron show <job-id>
openclaw cron run <job-id>
openclaw cron run <job-id> --due
openclaw cron run <job-id> --wait --wait-timeout 10m
openclaw cron run <job-id> --wait --wait-timeout 10m --poll-interval 2s
openclaw cron runs --id <job-id> --limit 50
openclaw cron runs --id <job-id> --run-id <run-id>
```

`openclaw cron list` 默认显示已启用的任务。传入 `--all` 可包括已禁用的任务，或传入 `--agent <id>` 仅显示有效规范化智能体 ID 匹配的任务；未存储智能体 ID 的任务视为使用配置的默认智能体。

`openclaw cron get <job-id>` 直接返回存储的任务 JSON。如需包含交付路由预览的易读视图，请使用 `cron show <job-id>`。

`cron list --json` 和 `cron show <job-id> --json` 会在每个任务中包含一个顶层 `status` 字段，该字段根据 `enabled`、`state.runningAtMs` 和 `state.lastRunStatus` 计算。值包括：`disabled`、`running`、`ok`、`error`、`skipped` 或 `idle`。JSON 状态保持规范且不加修饰，以便外部工具无需重新推导即可读取任务状态；易读输出可以为重复的 `error` 状态附加失败次数。

`cron runs` 条目包含交付诊断信息，包括预期的定时任务目标、解析后的目标、消息工具发送情况、后备方案使用情况和已交付状态。

每个任务的私有暂存区（Heartbeat 检查清单和类似的监控上下文）：

```bash
openclaw cron scratch <job-id>                  # 输出当前暂存区内容
openclaw cron scratch <job-id> --json           # 暂存区及修订元数据
openclaw cron scratch <job-id> --set "text"     # 使用精确文本替换暂存区
openclaw cron scratch <job-id> --file notes.md  # 使用文件内容替换暂存区（- 表示 stdin）
openclaw cron scratch <job-id> --unset          # 移除暂存区记录
```

暂存区存储在共享状态数据库中，上限为 256 KiB，并且绝不会包含在 `cron list`/`cron get`/`cron runs` 输出中。写入操作通过比较并交换机制，基于命令启动时读取的修订版本提供保护；也可以传入 `--expected-revision <n>` 来固定显式修订版本。有关 Heartbeat 监控器如何使用暂存区，请参阅 [Heartbeat](/zh-CN/gateway/heartbeat#monitor-scratch-optional)。

重新指定智能体和会话：

```bash
openclaw cron edit <job-id> --agent ops
openclaw cron edit <job-id> --clear-agent
openclaw cron edit <job-id> --session current
openclaw cron edit <job-id> --session "session:daily-brief"
```

在智能体轮次任务中省略 `--agent` 时，`openclaw cron add` 会发出警告，并回退到默认智能体（`main`）。创建时传入 `--agent <id>` 可固定特定智能体。

交付调整：

```bash
openclaw cron edit <job-id> --announce --channel slack --to "channel:C1234567890"
openclaw cron edit <job-id> --webhook "https://example.invalid/openclaw/cron"
openclaw cron edit <job-id> --best-effort-deliver
openclaw cron edit <job-id> --no-best-effort-deliver
openclaw cron edit <job-id> --no-deliver
```

## 相关内容

- [CLI 参考](/zh-CN/cli)
- [定时任务](/zh-CN/automation/cron-jobs)
