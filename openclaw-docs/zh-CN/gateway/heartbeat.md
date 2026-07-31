---
read_when:
    - 调整 Heartbeat 频率或消息传递
    - 为定时任务选择 Heartbeat 还是定时任务机制
sidebarTitle: Heartbeat
summary: Heartbeat 轮询消息和通知规则
title: Heartbeat
x-i18n:
    generated_at: "2026-07-26T06:42:43Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 44c78e797987d8dccab910cd82fc1f482df86afce40677846d8f26522d33f6fa
    source_path: gateway/heartbeat.md
    workflow: 16
---

<Note>
**Heartbeat 还是定时任务？** 请参阅[自动化](/zh-CN/automation)，了解何时使用哪一种。
</Note>

Heartbeat 会在主会话中运行**周期性的智能体轮次**，让模型能够呈现任何需要关注的事项，同时避免向你发送过多消息。

Heartbeat 是计划执行的主会话轮次——它**不会**创建[后台任务](/zh-CN/automation/tasks)记录。任务记录用于分离式工作（ACP 运行、子智能体、隔离的定时任务）。

在底层，Heartbeat 周期由定时任务调度器管理：Gateway 网关为每个启用了 Heartbeat 的智能体维护一个系统所有的定时任务（在 `openclaw cron list --all` 中显示为 `Heartbeat (agent-id)`）。Heartbeat 配置仍是期望状态输入，而持久化的监控计划负责实际触发时刻以及运行器后续的冷却时间。Gateway 网关会在启动和重新加载配置时写入配置变更；`openclaw doctor --fix` 可在下次 Gateway 网关启动前创建缺失或过期的监控行。请编辑 `agents.*.heartbeat`，而不是定时任务。

计划执行的 Heartbeat 需要启用定时任务。当 `cron.enabled` 为 `false` 或 `OPENCLAW_SKIP_CRON=1` 时，Gateway 网关会记录启动警告，并且不运行计划执行的 Heartbeat；手动和事件驱动的 Heartbeat 唤醒仍然可用。不存在单独的 Heartbeat 后备计时器。

故障排查：[定时任务](/zh-CN/automation/cron-jobs#troubleshooting)

## 快速开始（初学者）

<Steps>
  <Step title="选择周期">
    保持 Heartbeat 启用（默认为 `30m`；配置 Anthropic OAuth/令牌身份验证时，包括复用 Claude CLI，则为 `1h`），或设置自己的周期。
  </Step>
  <Step title="添加监控暂存内容（可选）">
    使用 `openclaw cron scratch <jobId> --set "..."` 在 Heartbeat 监控器的暂存区中存储一份简短的检查清单。
  </Step>
  <Step title="决定 Heartbeat 消息的发送位置">
    `target: "none"` 是默认值；设置 `target: "last"` 可将消息路由到最近的联系人。
  </Step>
  <Step title="可选调优">
    - 如果 Heartbeat 运行只需要监控暂存内容，请使用轻量级引导上下文。
    - 启用隔离会话，避免每次 Heartbeat 都发送完整的对话历史记录。
    - 将 Heartbeat 限制在活跃时段（本地时间）。

  </Step>
</Steps>

配置示例：

```json5
{
  agents: {
    defaults: {
      heartbeat: {
        every: "30m",
        target: "last", // 明确发送给最近的联系人（默认为 "none"）
        directPolicy: "allow", // 默认：允许直接/私信目标；设置为 "block" 可禁止
        lightContext: true, // 可选：Heartbeat 运行时跳过工作区引导文件
        isolatedSession: true, // 可选：每次运行使用新会话（无对话历史记录）
        // activeHours: { start: "08:00", end: "24:00" },
      },
    },
  },
}
```

## 默认值

- 间隔：`30m`。应用 Anthropic 提供商默认值时，如果解析后的身份验证模式为 OAuth/令牌（包括复用 Claude CLI），则会将其提高到 `1h`，但仅限 `heartbeat.every` 未设置时。设置 `agents.defaults.heartbeat.every` 或每个智能体的 `agents.entries.*.heartbeat.every`；使用 `0m` 可禁用。
- 提示正文（可通过 `agents.defaults.heartbeat.prompt` 配置）：`Follow the heartbeat monitor scratch context when provided. Recurring tasks are cron jobs; create or change their schedules with cron tools or the openclaw cron CLI, not heartbeat scratch. Do not infer or repeat old tasks from prior chats. If nothing needs attention, reply HEARTBEAT_OK.`
- 超时：未设置的 Heartbeat 轮次会在已设置 `agents.defaults.timeoutSeconds` 时使用它。否则，会使用 Heartbeat 周期，且上限为 600 秒。设置 `agents.defaults.heartbeat.timeoutSeconds` 或每个智能体的 `agents.entries.*.heartbeat.timeoutSeconds`，可执行耗时更长的 Heartbeat 工作。
- Heartbeat 提示会作为用户消息**原样**发送。为默认智能体启用 Heartbeat 时，系统提示中会包含“Heartbeat”部分，并在内部标记该次运行。
- 使用 `0m` 禁用 Heartbeat 时，监控定时任务仍会保留但处于禁用状态，其暂存内容也会保留，以便重新启用周期时使用。
- 如果定时任务本身已禁用，即使 Heartbeat 周期仍处于启用状态，计划执行的 Heartbeat 也不会运行。
- 活跃时段（`heartbeat.activeHours`）会按照配置的时区进行检查。在该时间窗口之外，Heartbeat 会被跳过，直到窗口内的下一个触发时刻。
- 当定时任务工作正在运行或排队，或者该智能体按会话键划分的子智能体或嵌套命令通道正忙时，Heartbeat 会自动推迟。同级智能体之间不会相互暂停。

## Heartbeat 提示的用途

默认提示有意采用宽泛的表述：

- **后台任务**：“考虑尚未完成的任务”会促使智能体检查后续事项（收件箱、日历、提醒、排队的工作），并呈现任何紧急事项。
- **询问用户近况**：“白天偶尔询问你的用户近况”会促使系统偶尔发送一条简短的“有什么需要吗？”消息，同时使用你配置的本地时区避免夜间打扰（请参阅[时区](/zh-CN/concepts/timezone)）。

Heartbeat 可以响应已完成的[后台任务](/zh-CN/automation/tasks)，但 Heartbeat 运行本身不会创建任务记录。

如果希望 Heartbeat 执行非常具体的操作（例如“检查 Gmail PubSub 统计信息”或“验证 Gateway 健康”），请将 `agents.defaults.heartbeat.prompt`（或 `agents.entries.*.heartbeat.prompt`）设置为自定义正文（将原样发送）。

## 响应约定

- 如果没有需要关注的事项，请回复 **`HEARTBEAT_OK`**。
- Heartbeat 运行也可以调用 `heartbeat_respond` 并使用 `notify: false` 表示没有可见更新，或使用 `notify: true` 加 `notificationText` 发出警报。如果存在结构化工具响应，它优先于文本后备响应。
- 有意义的 `heartbeat_respond` 结果与 `notify: false` 搭配时仍保持静默，但会被记作有界的内部上下文，供该会话中的下一个用户轮次使用。`no_change` 确认和可见通知不会以这种方式存储。
- 在 Heartbeat 运行期间，如果 `HEARTBEAT_OK` 出现在回复的**开头或结尾**，OpenClaw 会将其视为确认。该令牌会被移除；如果剩余内容不超过 300 个字符，回复会被丢弃。
- 如果 `HEARTBEAT_OK` 出现在回复的**中间**，则不会进行特殊处理。
- 对于警报，**不要**包含 `HEARTBEAT_OK`；仅返回警报文本。

在 Heartbeat 之外，消息开头/结尾意外出现的 `HEARTBEAT_OK` 会被移除并记录日志；仅包含 `HEARTBEAT_OK` 的消息会被丢弃。

## 配置

```json5
{
  agents: {
    defaults: {
      heartbeat: {
        every: "30m", // 默认：30m（0m 表示禁用）
        model: "anthropic/claude-opus-4-6",
        lightContext: false, // 默认：false；true 表示 Heartbeat 运行时跳过工作区引导文件
        isolatedSession: false, // 默认：false；true 表示每次 Heartbeat 都在新会话中运行（无对话历史记录）
        target: "last", // 默认：none | 选项：last | none | <channel id>（核心或插件，例如 "imessage"）
        to: "+15551234567", // 可选的渠道专用覆盖值
        accountId: "ops-bot", // 可选的多账户渠道 ID
        prompt: "在提供 Heartbeat 监控暂存上下文时遵循该上下文。周期性任务属于定时任务；请使用定时任务工具或 openclaw cron CLI 创建或更改其计划，而不是使用 Heartbeat 暂存内容。不要根据之前的聊天推断或重复旧任务。如果没有需要关注的事项，请回复 HEARTBEAT_OK。",
      },
    },
  },
}
```

### 范围和优先级

- `agents.defaults.heartbeat` 设置全局 Heartbeat 行为。
- `agents.entries.*.heartbeat` 在其基础上合并；如果任何智能体包含 `heartbeat` 块，则**只有这些智能体**会运行 Heartbeat。
- `channels.defaults.heartbeatVisibility` 设置所有渠道的可见性默认值。
- `channels.<channel>.heartbeatVisibility` 覆盖渠道默认值。
- `channels.<channel>.accounts.<id>.heartbeatVisibility`（多账户渠道）覆盖每个渠道的设置。

### 每个智能体的 Heartbeat

如果任何 `agents.entries.*` 条目包含 `heartbeat` 块，则**只有这些智能体**会运行 Heartbeat。每个智能体的配置块会合并到 `agents.defaults.heartbeat` 之上（因此可以只设置一次共享默认值，再按智能体进行覆盖）。

示例：两个智能体，只有第二个智能体运行 Heartbeat。

```json5
{
  agents: {
    defaults: {
      heartbeat: {
        every: "30m",
        target: "last", // 明确发送给最近的联系人（默认为 "none"）
      },
    },
    list: [
      { id: "main", default: true },
      {
        id: "ops",
        heartbeat: {
          every: "1h",
          target: "whatsapp",
          to: "+15551234567",
          timeoutSeconds: 45,
          prompt: "在提供 Heartbeat 监控暂存上下文时遵循该上下文。周期性任务属于定时任务；请使用定时任务工具或 openclaw cron CLI 创建或更改其计划，而不是使用 Heartbeat 暂存内容。不要根据之前的聊天推断或重复旧任务。如果没有需要关注的事项，请回复 HEARTBEAT_OK。",
        },
      },
    ],
  },
}
```

### 活跃时段示例

将 Heartbeat 限制在特定时区的工作时间内：

```json5
{
  agents: {
    defaults: {
      heartbeat: {
        every: "30m",
        target: "last", // 明确发送给最近的联系人（默认为 "none"）
        activeHours: {
          start: "09:00",
          end: "22:00",
          timezone: "America/New_York", // 可选；如果已设置 userTimezone，则使用它，否则使用主机时区
        },
      },
    },
  },
}
```

在此时间窗口之外（美国东部时间上午 9 点之前或晚上 10 点之后），Heartbeat 会被跳过。时间窗口内的下一个计划触发时刻会正常运行。

### 24/7 设置

如果希望 Heartbeat 全天运行，请使用以下模式之一：

- 完全省略 `activeHours`（不限制时间窗口；这是默认行为）。
- 设置全天时间窗口：`activeHours: { start: "00:00", end: "24:00" }`。

<Warning>
不要将 `start` 和 `end` 设置为相同时间（例如从 `08:00` 到 `08:00`）。这会被视为零宽度时间窗口，因此 Heartbeat 总会被跳过。
</Warning>

### 多账户示例

使用 `accountId` 可指定 Telegram 等多账户渠道中的特定账户：

```json5
{
  agents: {
    list: [
      {
        id: "ops",
        heartbeat: {
          every: "1h",
          target: "telegram",
          to: "12345678:topic:42", // 可选：路由到特定主题/话题串
          accountId: "ops-bot",
        },
      },
    ],
  },
  channels: {
    telegram: {
      accounts: {
        "ops-bot": { botToken: "YOUR_TELEGRAM_BOT_TOKEN" },
      },
    },
  },
}
```

### 字段说明

<ParamField path="every" type="string">
  Heartbeat 间隔（时长字符串；默认单位 = 分钟）。
</ParamField>
<ParamField path="model" type="string">
  Heartbeat 运行的可选模型覆盖值（`provider/model`）。
</ParamField>
<ParamField path="lightContext" type="boolean" default="false">
  为 true 时，Heartbeat 运行会使用轻量级引导上下文并跳过工作区引导文件。无论如何，监控暂存内容都会由 Heartbeat 运行器注入。
</ParamField>
<ParamField path="isolatedSession" type="boolean" default="false">
  为 true 时，每次 Heartbeat 都会在没有先前对话历史记录的新会话中运行。使用与定时任务 `sessionTarget: "isolated"` 相同的隔离模式。可大幅降低每次 Heartbeat 的令牌成本。与 `lightContext: true` 结合使用可最大程度节省成本。发送路由仍使用主会话上下文。
</ParamField>
<ParamField path="session" type="string">
  Heartbeat 运行的可选会话键。

- `main`（默认）：智能体主会话。
- 显式会话键（从 `openclaw sessions --json` 或[会话 CLI](/zh-CN/cli/sessions) 复制）。
- 会话键格式：请参阅[会话](/zh-CN/concepts/session)和[群组](/zh-CN/channels/groups)。

</ParamField>
<ParamField path="target" type="string">
- `last`：投递到上次使用的外部渠道。
- 显式渠道：任何已配置的渠道或插件 ID，例如 `discord`、`matrix`、`telegram` 或 `whatsapp`。
- `none`（默认）：运行 Heartbeat，但**不向外部投递**。

</ParamField>
<ParamField path="directPolicy" type='"allow" | "block"' default="allow">
  控制直接消息/私信投递行为。`allow`：允许直接消息/私信 Heartbeat 投递。`block`：禁止直接消息/私信投递（`reason=dm-blocked`）。

</ParamField>
<ParamField path="to" type="string">
  可选的接收者覆盖值（渠道特定 ID，例如 WhatsApp 的 E.164 号码或 Telegram 聊天 ID）。对于 Telegram 话题/线程，请使用 `<chatId>:topic:<messageThreadId>`。

</ParamField>
<ParamField path="accountId" type="string">
  多账户渠道的可选账户 ID。当使用 `target: "last"` 时，如果解析出的上次使用渠道支持账户，则该账户 ID 应用于该渠道；否则将被忽略。如果账户 ID 与解析出的渠道中已配置的账户不匹配，则跳过投递。

</ParamField>
<ParamField path="prompt" type="string">
  覆盖默认提示词正文（不合并）。

</ParamField>
<ParamField path="timeoutSeconds" type="number" default="global timeout or min(every, 600)">
  Heartbeat 智能体轮次中止前允许的最长秒数。保持未设置时，如果已设置 `agents.defaults.timeoutSeconds`，则使用该值；否则使用 Heartbeat 周期，且上限为 600 秒。

</ParamField>
<ParamField path="activeHours" type="object">
  将 Heartbeat 运行限制在一个时间窗口内。对象包含 `start`（HH:MM，含边界；一天开始时请使用 `00:00`）、`end`（HH:MM，不含边界；一天结束时允许使用 `24:00`），以及可选的 `timezone`。

- 省略或设为 `"user"`：如果已设置，则使用你的 `agents.defaults.userTimezone`；否则回退到主机系统时区。
- `"local"`：始终使用主机系统时区。
- 任何 IANA 标识符（例如 `America/New_York`）：直接使用；如果无效，则回退到上述 `"user"` 行为。
- 对于有效时间窗口，`start` 和 `end` 不得相等；相等的值将被视为零宽度（始终处于窗口之外）。
- 在有效时间窗口之外，将跳过 Heartbeat，直到窗口内的下一个触发时刻。

</ParamField>

## 投递行为

<AccordionGroup>
  <Accordion title="会话和目标路由">
    - 默认情况下，Heartbeat 在智能体的主会话中运行（`agent:<id>:<mainKey>`）；当 `session.scope = "global"` 时，则在 `global` 中运行。设置 `session` 可覆盖为特定渠道会话（Discord/WhatsApp 等）。
    - `session` 仅影响运行上下文；投递由 `target` 和 `to` 控制。
    - 要投递到特定渠道/接收者，请设置 `target` + `to`。使用 `target: "last"` 时，投递将使用该会话上次使用的外部渠道。
    - Heartbeat 投递默认允许直接消息/私信目标。设置 `directPolicy: "block"` 可禁止向直接目标发送，同时仍运行 Heartbeat 轮次。
    - 如果主队列、目标会话通道、定时任务通道或活动中的定时任务繁忙，则跳过 Heartbeat 并稍后重试。
    - 如果 `target` 未解析出外部目标，运行仍会进行，但不会发送出站消息。

  </Accordion>
  <Accordion title="可见性和跳过行为">
    - 如果 `showOk`、`showAlerts` 和 `useIndicator` 均已禁用，则会预先跳过运行并标记为 `reason=alerts-disabled`。
    - 如果仅禁用警报投递，OpenClaw 仍可运行 Heartbeat、更新到期任务时间戳、恢复会话空闲时间戳，并禁止向外发送警报载荷。
    - 如果解析出的 Heartbeat 目标支持正在输入状态，OpenClaw 会在 Heartbeat 运行期间显示正在输入状态。它使用 Heartbeat 原本要向其发送聊天输出的同一目标，并可通过 `typingMode: "never"` 禁用。

  </Accordion>
  <Accordion title="会话生命周期和审计">
    - 仅含 Heartbeat 的回复**不会**使会话保持活动。Heartbeat 元数据可能会更新会话行，但空闲过期使用最后一条真实用户/渠道消息中的 `lastInteractionAt`，每日过期则使用 `sessionStartedAt`。
    - Control UI 和 WebChat 历史记录会隐藏 Heartbeat 提示词和仅含 OK 的确认消息。底层会话记录仍可包含这些轮次，以供审计/重放。
    - 分离的[后台任务](/zh-CN/automation/tasks)可以将系统事件加入队列，并在主会话需要快速获知某些事项时唤醒 Heartbeat。该唤醒不会使 Heartbeat 作为后台任务运行。

  </Accordion>
</AccordionGroup>

## 可见性控制

默认情况下，会禁止显示 `HEARTBEAT_OK` 确认消息，同时投递警报内容。你可以按渠道或账户调整此行为：

```yaml
channels:
  defaults:
    heartbeat:
      showOk: false # 隐藏 HEARTBEAT_OK（默认）
      showAlerts: true # 显示警报消息（默认）
      useIndicator: true # 发出指示器事件（默认）
  telegram:
    heartbeat:
      showOk: true # 在 Telegram 上显示 OK 确认消息
  whatsapp:
    accounts:
      work:
        heartbeat:
          showAlerts: false # 禁止为此账户投递警报
```

优先级：按账户 → 按渠道 → 渠道默认值 → 内置默认值。

### 每个标志的作用

- `showOk`：当模型返回仅含 OK 的回复时，发送 `HEARTBEAT_OK` 确认消息。
- `showAlerts`：当模型返回非 OK 回复时，发送警报内容。
- `useIndicator`：为 UI 状态界面发出指示器事件。

如果**三者**均为 false，OpenClaw 将完全跳过 Heartbeat 运行（不调用模型）。

### 按渠道与按账户示例

```yaml
channels:
  defaults:
    heartbeat:
      showOk: false
      showAlerts: true
      useIndicator: true
  slack:
    heartbeat:
      showOk: true # 所有 Slack 账户
    accounts:
      ops:
        heartbeat:
          showAlerts: false # 仅禁止 ops 账户的警报
  telegram:
    heartbeat:
      showOk: true
```

### 常见模式

| 目标                                     | 配置                                                                                   |
| ---------------------------------------- | ---------------------------------------------------------------------------------------- |
| 默认行为（静默处理 OK，启用警报） | _（无需配置）_                                                                     |
| 完全静默（无消息、无指示器） | `channels.defaults.heartbeat: { showOk: false, showAlerts: false, useIndicator: false }` |
| 仅指示器（无消息）             | `channels.defaults.heartbeat: { showOk: false, showAlerts: false, useIndicator: true }`  |
| 仅在一个渠道中显示 OK                  | `channels.telegram.heartbeat: { showOk: true }`                                          |

## 监视器暂存文档（可选）

每个 Heartbeat 监视器定时任务都拥有一个存储在共享状态数据库中的私有暂存文档。可将其视为你的“Heartbeat 检查清单”：内容简短、稳定，并且适合每 30 分钟检查一次。暂存文档存在时，其内容将附加到 Heartbeat 提示词中。

使用定时任务 CLI 管理它（任务 ID 来自 `openclaw cron list --all`）：

```bash
openclaw cron scratch <jobId>                 # 输出当前暂存文档
openclaw cron scratch <jobId> --set "..."     # 替换为确切文本
openclaw cron scratch <jobId> --file notes.md # 使用文件内容替换（- 表示标准输入）
openclaw cron scratch <jobId> --unset         # 删除
```

写入受比较并交换机制保护：传入 `--expected-revision <n>` 可在发生并发编辑时失败，而不是覆盖该编辑。暂存文档大小上限为 256 KiB，且绝不会出现在 `cron list`/`cron runs` 输出中。

智能体也可以更新自己的暂存文档：在 Heartbeat 轮次中，`heartbeat_respond` 接受可选的 `scratch` 字符串，以完全替换监视器的暂存文档，供未来的 Heartbeat 使用。

<Note>
**正在从 HEARTBEAT.md 或仅配置周期的方式迁移？**运行 `openclaw doctor --fix`。Doctor 首先根据 `agents.*.heartbeat` 创建或更新系统拥有的监视器行，然后将每个智能体工作区的 `HEARTBEAT.md` 导入监视器的暂存文档，把所有有效的旧版 `tasks:` 条目转换为定时任务，将原文件归档到状态目录下（`backups/heartbeat-migration/`），并删除该文件。运行时 Heartbeat 指令仅来自数据库暂存文档；运行时绝不会读取 `HEARTBEAT.md`。
</Note>

如果暂存文档存在但实际上为空（仅包含空行、Markdown/HTML 注释、类似 `# Heading` 的 Markdown 标题、围栏标记或空的检查清单占位项），OpenClaw 会跳过 Heartbeat 运行以节省 API 调用。该跳过会报告为 `reason=empty-heartbeat-file`。如果不存在暂存文档，Heartbeat 仍会运行，由模型决定要执行的操作。

保持内容精简（简短的检查清单或提醒），以避免提示词膨胀。

暂存文档示例：

```md
# Heartbeat 检查清单

- 快速检查：收件箱中是否有紧急事项？
- 如果是白天，并且没有其他待办事项，进行一次简短的主动问候。
- 如果任务受阻，记下_缺少什么_，并在下次询问 Peter。
```

### 使用定时任务安排重复检查

Heartbeat 暂存文档是提示词上下文，而不是调度器。将每项重复检查创建为一个[定时任务](/zh-CN/automation/cron-jobs)，使其拥有自己的周期、启用/禁用状态和运行历史记录。当检查需要使用正常对话上下文时，定时任务仍可将主会话设为目标。

较旧的暂存文档可能包含结构化的 `tasks:` 块。升级后运行一次 `openclaw doctor --fix`：Doctor 会将每个有效条目转换为独立调度的定时任务，保留其间隔和先前的最后运行时间，移除已弃用的块，同时保留其周围的暂存文档正文。运行时 Heartbeat 轮次不会将 `tasks:` 文本解析为计划。

Doctor 创建的 Heartbeat 任务会保留 Heartbeat 的有效时段、冷却、防洪和繁忙保护。同时到期的任务可以合并到一个 Heartbeat 轮次中。有效时段之外的任务执行将被跳过，并在下一个定时任务触发时再次尝试。

### 智能体可以更新其暂存文档吗？

可以。在 Heartbeat 轮次中，智能体可以向 `heartbeat_respond` 传入 `scratch` 值，以完全替换供未来 Heartbeat 使用的监视器正文。你也可以在普通聊天中要求它运行 `openclaw cron scratch <jobId> --set ...`，或使用同一命令自行编辑暂存文档。请使用定时任务管理重复计划，不要将调度器语法写入暂存文档。

<Warning>
不要将机密信息（API 密钥、电话号码、私有令牌）放入监视器暂存文档，因为它会成为提示词上下文的一部分。
</Warning>

## 手动唤醒（按需）

使用 `openclaw system event` 将系统事件加入队列，并可选择立即触发 Heartbeat：

```bash
openclaw system event --text "检查紧急待跟进事项" --mode now
```

| 标志                         | 说明                                                                                      |
| ---------------------------- | ------------------------------------------------------------------------------------------------ |
| `--text <text>`              | 系统事件文本（必填）。                                                                    |
| `--mode <mode>`              | `now` 会立即运行一次 Heartbeat；`next-heartbeat`（默认）会等待下一个计划触发时刻。 |
| `--session-key <sessionKey>` | 将事件定向到特定会话；默认为该智能体的主会话。                   |
| `--json`                     | 输出 JSON。                                                                                     |

如果未提供 `--session-key`，并且多个智能体配置了 `heartbeat`，则 `--mode now` 会立即运行其中每个智能体的 Heartbeat。

同一 CLI 组中的相关 Heartbeat 控制命令：

```bash
openclaw system heartbeat last     # 显示上一个 Heartbeat 事件
openclaw system heartbeat enable   # 启用 Heartbeat
openclaw system heartbeat disable  # 禁用 Heartbeat
```

## 成本意识

Heartbeat 会运行完整的智能体轮次。间隔越短，消耗的 token 越多。要降低成本：

- 使用 `isolatedSession: true`，避免发送完整的对话历史记录（每次运行可从约 100K token 降至约 2-5K）。
- 使用 `lightContext: true`，在 Heartbeat 运行时跳过工作区引导文件。
- 设置成本更低的 `model`（例如 `ollama/llama3.2:1b`）。
- 保持监控器暂存内容精简。
- 如果只需要更新内部状态，请使用 `target: "none"`。

## Heartbeat 后上下文溢出

Heartbeat 运行完成后，会保留共享会话中现有的运行时模型。因此，如果 Heartbeat 将会话切换到上下文窗口较小的本地模型（例如具有 32k 窗口的 Ollama 模型），该模型可能会继续用于主会话的下一轮。如果下一轮随后报告上下文溢出，并且会话最后使用的运行时模型与配置的 `heartbeat.model` 匹配，OpenClaw 的恢复消息会指出 Heartbeat 模型渗漏可能是原因，并建议修复方法。

要避免此问题：使用 `isolatedSession: true` 在新会话中运行 Heartbeat（可选择与 `lightContext: true` 结合使用，以获得最小的提示词），或者选择上下文窗口足以容纳共享会话的 Heartbeat 模型。

## 相关内容

- [自动化](/zh-CN/automation) - 一览所有自动化机制
- [后台任务](/zh-CN/automation/tasks) - 如何跟踪脱离前台运行的工作
- [时区](/zh-CN/concepts/timezone) - 时区如何影响 Heartbeat 调度
- [故障排查](/zh-CN/automation/cron-jobs#troubleshooting) - 调试自动化问题
