---
read_when:
    - 调度后台任务或唤醒操作
    - 将外部触发器（Webhooks、Gmail）接入 OpenClaw
    - 为定时任务选择 Heartbeat 还是 cron
sidebarTitle: Scheduled tasks
summary: Gateway 网关调度器的定时任务、Webhooks 和 Gmail PubSub 触发器
title: 定时任务
x-i18n:
    generated_at: "2026-07-26T06:40:19Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: dd889cf8e45196eda3ec7c2af930abcb2cc2bae8bad2dbdcaf3cd521a9e884b2
    source_path: automation/cron-jobs.md
    workflow: 16
---

Cron 是 Gateway 网关的内置调度器。它会持久化任务，在适当的时间唤醒智能体，并可将输出发送到聊天渠道、Webhook，或不发送到任何位置。

## 快速开始

<Steps>
  <Step title="添加一次性提醒">
    ```bash
    openclaw cron create "2027-02-01T16:00:00Z" \
      --name "Reminder" \
      --session main \
      --system-event "Reminder: check the cron docs draft" \
      --wake now \
      --delete-after-run
    ```
  </Step>
  <Step title="检查你的任务">
    ```bash
    openclaw cron list
    openclaw cron get <job-id>
    openclaw cron show <job-id>
    ```
  </Step>
  <Step title="查看运行历史">
    ```bash
    openclaw cron runs --id <job-id>
    ```
  </Step>
</Steps>

## Cron 的工作原理

- Cron 在 **Gateway 网关进程内部**运行，而不是在模型内部。Gateway 网关必须处于运行状态，调度才能触发。
- 任务定义、运行时状态和运行历史会持久化到 OpenClaw 的共享 SQLite 状态数据库中，因此重启不会丢失调度。
- 每次 Cron 执行都会创建一条[后台任务](/zh-CN/automation/tasks)记录。
- 一次性任务（`--at`）默认在成功后自动删除；传入 `--keep-after-run` 可保留它们。
- 每次运行的实际时间预算：设置后使用 `--timeout-seconds`。否则，隔离式/分离式智能体轮次任务受 Cron 自身的 60 分钟看门狗限制，该限制会在底层智能体轮次超时（`agents.defaults.timeoutSeconds`，默认 48 小时）生效前触发；命令任务默认为 10 分钟，脚本载荷默认为 5 分钟。
- Gateway 网关启动时，逾期的隔离式智能体轮次任务会被重新调度，而不是立即重放，从而避免模型/工具的引导工作占用渠道连接窗口。
- 如果你通过系统 Cron 或其他外部调度器驱动 `openclaw agent`，即使 CLI 已处理 `SIGTERM`/`SIGINT`，也要用硬终止升级机制封装它。由 Gateway 网关支持的运行会请求 Gateway 网关中止已接受的运行；`--local` 运行也会收到相同的中止信号。对于 GNU `timeout`，应优先使用 `timeout -k 60 600 openclaw agent ...`，而不是普通的 `timeout 600 ...`——如果进程无法及时完成清理退出，`-k` 值将作为最后保障。对于 systemd 单元，使用 `SIGTERM` 停止信号，并在最终终止前设置宽限窗口（`TimeoutStopSec`）。如果原始 Gateway 网关运行仍处于活动状态时重复使用 `--run-id`，系统会将重复请求报告为正在运行，而不会启动第二次运行。

<AccordionGroup>
  <Accordion title="隔离式运行加固">
    - 隔离式运行完成时，会尽力关闭其 `cron:<jobId>` 会话中受跟踪的浏览器标签页/进程，并通过主会话和自定义会话运行所使用的同一共享拆卸路径，释放为该任务创建的所有内置 MCP 运行时实例。清理失败会被忽略，以确保 Cron 结果仍然有效。
    - 具有有限 Cron 自清理授权的隔离式运行可以读取调度器状态、仅包含自身任务的自过滤列表以及该任务的运行历史，并且只能删除自身任务。
    - 隔离式运行会防范过时的确认回复：如果第一个结果只是临时状态更新（`on it`、`pulling everything together` 以及类似提示），且没有任何后代子智能体仍负责最终答案，OpenClaw 会再次提示一次，以在交付前获取实际结果。
    - 系统会识别结构化的执行拒绝元数据（包括嵌套错误以 `SYSTEM_RUN_DENIED` 或 `INVALID_REQUEST` 开头的节点主机 `UNAVAILABLE` 包装器），因此被阻止的命令不会被报告为成功运行，同时普通的助手文本也不会被误判为拒绝。
    - 即使没有回复载荷，运行级智能体故障也会计为任务错误，因此模型/提供商故障会增加错误计数并触发失败通知，而不会将任务清除为成功状态。
    - 当任务达到 `timeoutSeconds` 时，Cron 会中止运行并提供一个短暂的清理窗口。如果运行未能完成清理退出，Gateway 网关负责的清理会在 Cron 记录超时之前强制清除该运行的会话所有权，从而避免排队的聊天工作被过时的处理中会话阻塞。
    - 设置/启动停滞会获得特定于阶段的超时（例如 `cron: isolated agent setup timed out before runner start` 或 `cron: isolated agent run stalled before execution start (last phase: context-engine)`）。即使外部 CLI 进程尚未启动，这些看门狗也会覆盖嵌入式和由 CLI 支持的提供商，并且其上限独立于较长的 `timeoutSeconds` 值，因此冷启动、身份验证或上下文故障可迅速显现。

  </Accordion>
  <Accordion title="任务协调">
    Cron 任务协调首先由运行时负责，其次以持久化历史为依据：只要 Cron 运行时仍将相应任务跟踪为运行中，活动的 Cron 任务就会保持活动状态，即使旧的子会话行仍然存在。一旦运行时不再拥有该任务且 5 分钟宽限窗口到期，维护检查就会检查匹配 `cron:<jobId>:<startedAt>` 运行的持久化运行日志和任务状态。其中的终态结果会完成任务账本记录；否则，由 Gateway 网关负责的维护可以将任务标记为 `lost`。离线 CLI 审计可以从持久化历史中恢复，但其自身进程内活动任务集为空，并不能证明由 Gateway 网关负责的运行已不存在。
  </Accordion>
</AccordionGroup>

## 调度类型

| 类型      | CLI 标志           | 说明                                                                                              |
| --------- | ------------------ | -------------------------------------------------------------------------------------------------------- |
| `at`      | `--at`             | 一次性时间戳（ISO 8601，或类似 `20m` 的相对时间）                                                     |
| `every`   | `--every`          | 固定间隔（`10m`、`1h`、`1d`）                                                                       |
| `cron`    | `--cron`           | 5 字段或 6 字段 Cron 表达式，可选 `--tz`                                                  |
| `on-exit` | `--on-exit`        | 受监视的命令退出时触发一次（事件触发器；智能体轮次拆卸后仍然有效；可选 `--on-exit-cwd`） |
| `stream`  | `--stream-command` | 根据受监管的长期运行命令所生成的批量行触发                                      |

不含时区的时间戳按 UTC 处理。添加 `--tz America/New_York`，可在该 IANA 时区中解释不含偏移量的 `--at` 日期时间或计算 Cron 表达式。不含 `--tz` 的 Cron 表达式使用 Gateway 网关主机时区。`--tz` 不能与 `--every` 或 `--on-exit` 一起使用。

每小时整点重复执行的表达式（分钟字段为 `0`，小时字段为通配符）会自动错开最多 5 分钟，以减少负载峰值。使用 `--exact` 可强制精确计时，或使用 `--stagger 30s` 设置明确的时间窗口（仅适用于 Cron 调度）。

### Heartbeat 任务迁移

旧版 Heartbeat 临时配置支持结构化的 `tasks:` 块。升级后运行 `openclaw doctor --fix`，可将每个条目转换为普通的、可编辑的主会话 Cron 任务。Doctor 会保留时间间隔和上次运行时间，先创建任务再移除该块，并且重新运行时会安全地收敛相同的声明键。

这些迁移后的任务携带公开的 `systemEvent` 载荷，因此 `openclaw cron list`、`get`、`edit` 和 `remove` 以及 Cron 工具可以像管理其他任务一样管理它们。它们的执行使用受保护的 Heartbeat 任务唤醒机制：活动时段、最小间隔、防洪控制和忙碌重试仍然适用，而 Cron 负责每项任务的独立节奏。在同一合并窗口内到期的任务可以共享一个 Heartbeat 轮次。若计划执行时间位于 Heartbeat 活动时段之外，则会跳过该次执行，并在任务下一次到期时重试。

Heartbeat 临时配置现在仅作为监控说明文本。运行时 Heartbeat 不会将 `tasks:` 文本解析为调度；请使用 Cron 创建新的重复任务。

### 流数据源

流调度会在 Gateway 网关下持续运行由操作员编写的 argv 命令，并根据其 stdout 和 stderr 输出行触发任务。流调度由事件驱动，绝不会按时间到期触发，并且需要 `cron.triggers.enabled: true`，因为长期运行的命令与触发器脚本属于相同的无人值守信任类别。禁用或移除任务会停止该进程；Gateway 网关关闭时会等待进程树完成拆卸。快速失败会按照 Cron 的内置错误退避机制重启。若连续 5 次运行均短于 60 秒，任务将保持错误状态并使用常规失败提醒路径；手动重新启用任务可清除重启上限。

```bash
openclaw cron add \
  --name "Build event stream" \
  --stream-command '["node","scripts/build-events.mjs"]' \
  --stream-mode match \
  --stream-match '^(failed|recovered):' \
  --stream-batch-ms 250 \
  --session isolated \
  --message "Investigate these build events."
```

`mode: "line"`（默认值）接受每一行。`mode: "match"` 仅接受与已编译的 `match` 正则表达式匹配的行。批次会在静默达到 `batchMs` 后关闭（默认 250 毫秒，限制在 50–5000 之间），或在达到 `maxBatchBytes` 时关闭（默认 16384，限制在 1024–65536 之间）。达到字节上限时，批次会以 `[truncated]` 结束。匹配模式始终根据完整文本计算完整的行，即使超过 `maxBatchBytes` 也是如此（只有交付的批次会被截断）；在有界原始输入上限处被截断的行只是一个前缀，因此会被视为不匹配，避免让以结尾锚定的模式在截断内容上触发。批次会追加到系统事件文本或智能体轮次消息中。流调度会拒绝命令载荷，因为源命令和载荷命令的进程所有权会产生歧义。

每项任务只保留一次载荷触发和一个有界待处理批次。载荷运行期间到达的行，或在内置的 30 秒触发间隔结束前到达的行，会合并到该待处理批次中，而不会形成无界队列。单一串行所有者会在 `streamDroppedBatches` 中记录门控丢弃、载荷错误和非运行状态的调度；有界合并会递增 `streamCoalescedBatches`。失败的载荷不会重试，因为它们可能不具备幂等性。逻辑源身份在受监管的子进程重启期间保持稳定，但在源被禁用、移除或替换时会轮换，因此即使进行 A 到 B 再到 A 的编辑，已退役源的排队批次也无法触发。停止完成后，旧子进程的延迟回调将不再生效。V1 不包含原生 WebSocket 数据源；可使用 argv 命令（例如 `websocat wss://example.invalid/events`）进行桥接。

当流任务还包含 `trigger.script` 时，门控会对每个已关闭的批次运行一次。当前批次作为深度冻结的 `trigger.streamBatch` 字符串与 `trigger.state` 一起提供。`fire: false` 会在持久化门控状态后丢弃该批次。`fire: true` 会保留现有的触发器消息语义，然后将该批次追加到生成的载荷中。流任务也可以改用不带条件门控的脚本载荷；该脚本通过相同的 `trigger.streamBatch` 值接收批次。脚本载荷不能与条件门控组合使用，因为二者都会占用持久化的 `trigger.state` 槽位。

### 动态节奏（速率控制）

重复任务可以将 `pacing.min` 和/或 `pacing.max` 设置为类似 `15m` 或 `4h` 的时长字符串；必须至少设置一个边界。将 `--pacing-min` 和 `--pacing-max` 与 `cron add|edit` 一起使用（`--clear-pacing` 会移除两个边界）。

在隔离运行期间，节奏调度任务可以使用 `action: "next_check"` 和 `in: "30m"` 调用 `cron` 工具。该提议仅适用于当前正在运行的任务，并从运行成功完成时开始计时。OpenClaw 会静默地将其限制在配置的边界内。

未提供提议的节奏调度不会更改正常计划。失败、超时和跳过的运行会丢弃该提议，因此现有的重试和错误退避行为优先。手动强制运行定时任务属于带外操作，会保留其待执行的自然或节奏调度时段。对于条件触发的任务，即使提议要求更早检查，内置最小间隔仍是下限。

### 月中日期和星期使用 OR 逻辑

Cron 表达式由 [croner](https://github.com/Hexagon/croner) 解析。当月中日期和星期字段均不是通配符时，只要**任一**字段匹配，croner 就会匹配，而不是要求两者都匹配。这是标准的 Vixie cron 行为。

```bash
# 预期：“仅当 15 日是星期一时，在上午 9 点运行”
# 实际：“每月 15 日上午 9 点运行，并且每个星期一上午 9 点运行”
0 9 15 * 1
```

这会导致每月大约触发 5-6 次，而不是 0-1 次。若要同时满足两个条件，请使用 croner 的 `+` 星期修饰符（`0 9 15 * +1`），或者按其中一个字段进行调度，并在任务的提示词或命令中检查另一个字段。

## 事件触发器（条件监视器）

事件触发器会向 `every`、`cron` 或 `stream` 计划添加无界面条件脚本。时间计划会在到期时对其求值；流计划会对每个已关闭的批次进行求值。仅当脚本返回 `fire: true` 时，Cron 才运行正常载荷：

```json5
{
  schedule: { kind: "every", everyMs: 30000 },
  trigger: {
    // 仅当观察到的状态与上次求值不同时触发。
    script: "const res = await tools.call('exec', { command: 'gh pr checks 123 --json state -q \\'.[].state\\' | sort -u' }); const status = String(res?.result?.details?.aggregated ?? '').trim(); json({ fire: status !== trigger.state?.status, message: `PR 123 CI: ${trigger.state?.status ?? 'unknown'} -> ${status}`, state: { status } });",
    once: false,
  },
  payload: { kind: "agentTurn", message: "调查 CI 状态变化。" },
}
```

脚本必须返回 `{ fire, message?, state? }`。之前的 JSON 状态以深度冻结的 `trigger.state` 形式提供；流门控还会通过 `trigger.streamBatch` 接收当前批次。返回新的 `state` 值以将其持久化。状态上限为 16 KB。当触发结果包含 `message` 时，Cron 会在执行前将其附加到系统事件文本或智能体轮次消息中。`once: true` 会在首次成功执行已触发载荷后禁用该任务。

`fire: false` 会持久化求值状态和计数器，然后重新调度，而不会创建运行历史记录。如果已触发的载荷运行失败，返回的 `state` **不会**被持久化——下一次求值会看到之前的状态并可再次触发，因此应将脚本编写为只读检查，并将操作放在载荷中。触发器计划具有内置的 30 秒最小间隔。每次求值具有 30 秒的实际时间预算，最多可调用工具 5 次。

应围绕**可操作状态**编写监视器，而不应只关注成功：如果检查失败或超时时监视器悄无声息，它在故障时看起来仍然正常。将观察结果与 `trigger.state` 比较，并返回最新状态以进行去重；不要依赖模型或进程内存。触发时，应确保 `message` 自包含，因为它会成为已触发运行的完整事件上下文。

<Warning>
启用 `cron.triggers.enabled` 后，条件触发脚本和 `script` 载荷均可使用所属智能体的**完整工具策略（包括 `exec`）**以无界面方式运行。应将其视为使用该智能体权限的无人值守代码执行；除非所有获准创建定时任务的智能体均受到相应信任，否则请保持禁用。
</Warning>

从本地脚本文件创建监视器（`-` 从标准输入读取脚本）：

```bash
openclaw cron add \
  --name "PR CI watcher" \
  --every 30s \
  --trigger-script ./watch-pr-ci.js \
  --message "Respond to the CI status change" \
  --session isolated
```

## 载荷

每个任务都恰好携带一种载荷类型，由标志选择：

| 载荷          | 标志                                           | 运行方式                                                   |
| ------------- | ---------------------------------------------- | ---------------------------------------------------------- |
| 系统事件      | `--system-event <text>`                        | 加入主会话队列，本身不调用模型                             |
| 智能体消息    | `--message <text>`                             | 由模型支持的智能体轮次                                     |
| 命令          | `--command <shell>` 或 `--command-argv <json>` | Gateway 网关主机上的 shell/进程，不调用模型                |
| 脚本          | `--script <file\|->`                           | 使用所属智能体工具的无界面代码模式脚本                     |

另有一种载荷类型 `heartbeat` 由系统所有：Gateway 网关会为每个启用了 Heartbeat 的智能体收敛为一个 Heartbeat 监控任务（参见 [Heartbeat](/zh-CN/gateway/heartbeat)）。它会显示在 `cron list --all` 中，但无法通过 CLI 或 API 创建或编辑。启动、配置重新加载或执行 `openclaw doctor --fix` 时，Heartbeat 配置会直写到持久化的监控计划中。当 Cron 被禁用时，监控器不会触发，也不会运行备用 Heartbeat 计时器。

### 智能体轮次选项

<ParamField path="--message" type="string" required>
  提示词文本（隔离任务、当前会话任务或自定义会话任务必须提供）。
</ParamField>
<ParamField path="--model" type="string">
  模型覆盖；必须解析为允许的模型，否则运行会因验证错误而失败。
</ParamField>
<ParamField path="--fallbacks" type="string">
  每任务备用模型列表，例如 `--fallbacks openai/gpt-5.6-sol,openrouter/meta-llama/llama-3.3-70b-instruct:free`。传入 `--fallbacks ""` 可执行不使用备用模型的严格运行。
</ParamField>
<ParamField path="--clear-fallbacks" type="boolean">
  在 `cron edit` 中，移除每任务备用模型覆盖，使任务遵循配置的备用模型优先级。不能与 `--fallbacks` 组合使用。
</ParamField>
<ParamField path="--clear-model" type="boolean">
  在 `cron edit` 中，移除每任务模型覆盖，使任务遵循正常的 Cron 模型优先级（已存储的 Cron 会话覆盖，否则使用智能体/默认模型）。不能与 `--model` 组合使用。
</ParamField>
<ParamField path="--thinking" type="string">
  思考级别覆盖（`off|minimal|low|medium|high|xhigh|adaptive|max|ultra`）。可用级别仍取决于所选模型和智能体运行时。
</ParamField>
<ParamField path="--clear-thinking" type="boolean">
  在 `cron edit` 中，移除每任务思考覆盖。不能与 `--thinking` 组合使用。
</ParamField>
<ParamField path="--light-context" type="boolean">
  跳过工作区引导文件注入。
</ParamField>
<ParamField path="--tools" type="string">
  限制任务可以使用的工具，例如 `--tools exec,read`。
</ParamField>

能够运行工具的新任务始终存储显式工具策略。由智能体创建的任务
仅能使用创建该任务的轮次可用的工具，且智能体无法扩大
已存储的列表。由经过身份验证且未使用 `--tools` 的操作员创建的任务会存储
不受限制的 `*` 策略；`cron edit --clear-tools` 会恢复该显式不受限制
策略。早于显式工具策略而存在的任务会保留其当前行为，
直到显式编辑其工具策略或重新创建该任务。

`--model` 设置任务的主要模型；它不会取代会话 `/model` 覆盖，因此配置的备用模型链仍会叠加应用。无法解析或不允许的模型会导致运行出现明确的验证错误，而不会静默回退到默认模型。如果任务具有 `--model`，但没有显式或已配置的备用模型列表，OpenClaw 会传入空的备用模型覆盖，而不会静默附加智能体的主要模型作为隐藏重试目标。

隔离任务的模型选择优先级从高到低如下：

1. 每任务载荷 `model`（显式配置；不允许的模型会导致运行失败）
2. Gmail 钩子模型覆盖（仅当运行来自 Gmail 且允许该覆盖时）
3. 用户选择并存储的 Cron 会话模型覆盖
4. 智能体/默认模型选择

快速模式遵循解析后的实时选择。如果所选模型配置具有 `params.fastMode`，隔离 Cron 默认使用它；已存储的会话 `fastMode` 覆盖（其次是智能体 `fastModeDefault`）在任一方向上仍优先于模型配置。自动模式使用模型的 `params.fastAutoOnSeconds` 截止值，默认为 60 秒。

如果运行遇到实时模型切换交接，Cron 会使用切换后的提供商/模型重试，并为当前运行持久化该选择（以及任何新的身份验证配置文件）。重试次数有上限：初始尝试加 2 次切换重试后，Cron 会中止，而不会循环。

隔离运行开始前，OpenClaw 会检查已配置的 `api: "ollama"` 和 `api: "openai-completions"` 提供商的可达本地端点，这些提供商的 `baseUrl` 为环回地址、专用网络或 `.local`。此预检会遍历任务配置的备用模型链，只有当所有候选项均不可达时，才将运行标记为 `skipped`；`--fallbacks ""` 会将该遍历严格限制为主要模型。端点宕机会将运行记录为 `skipped`，并提供明确错误，而不会启动模型调用。结果按端点缓存 5 分钟（而非按任务或模型缓存），因此共享已宕机的本地 Ollama/vLLM/SGLang/LM Studio 服务器的多个到期任务只会产生一次探测，而不会引发请求风暴。跳过的预检运行不会增加执行错误退避；设置 `failureAlert.includeSkipped` 可选择接收重复的跳过警报。

### 命令载荷

命令载荷在 Gateway 网关调度器内运行确定性脚本，而不会启动由模型支持的轮次。它们在 Gateway 网关主机上执行，捕获 stdout/stderr，在 Cron 历史记录中记录运行，并复用与智能体轮次任务相同的 `announce`、`webhook` 和 `none` 交付模式。

<Note>
命令 Cron 是操作员管理员使用的 Gateway 网关自动化界面，而不是智能体的 `tools.exec` 调用。创建、更新、移除或手动运行定时任务需要 `operator.admin`；之后，计划命令运行会在 Gateway 网关进程内作为该管理员编写的自动化执行。智能体 Exec 策略（`tools.exec.mode`、审批提示、按智能体配置的工具允许列表）管理模型可见的 Exec 工具，而不管理命令 Cron 载荷。
</Note>

```bash
openclaw cron create "*/15 * * * *" \
  --name "Queue depth probe" \
  --command "scripts/check-queue.sh" \
  --command-cwd "/srv/app" \
  --announce \
  --channel telegram \
  --to "-1001234567890"
```

`--command <shell>` 存储 `argv: ["sh", "-lc", <shell>]`。使用 `--command-argv '["node","scripts/report.mjs"]'` 可在不经过 shell 解析的情况下精确执行 argv。可选的 `--command-env KEY=VALUE`（可重复）、`--command-input`、`--timeout-seconds`（默认 10 分钟）、`--no-output-timeout-seconds` 和 `--output-max-bytes` 用于控制进程环境、标准输入和输出限制。

交付文本从进程输出派生：非空 stdout 优先；如果 stdout 为空而 stderr 非空，则交付 stderr；如果两者均存在，Cron 会发送一个简短的 `stdout:` / `stderr:` 块。退出代码 `0` 会将运行记录为 `ok`；非零退出、信号、超时或无输出超时会记录为 `error`，并可能触发失败警报。仅输出 `NO_REPLY` 的命令会使用正常的 Cron 静默令牌抑制机制，不会向聊天发回任何内容。

### 脚本载荷

脚本载荷会在与触发器脚本相同的代码模式执行器中以无头方式运行，不会启动对话式智能体轮次。创建或运行它们之前，请启用 `cron.triggers.enabled`；这一危险自动化门控同时涵盖触发器脚本和脚本载荷。脚本任务仅支持 `main` 和 `isolated` 会话目标。

```bash
openclaw cron create "0 * * * *" \
  --name "Hourly queue check" \
  --script ./automation/check-queue.js \
  --script-timeout-seconds 300 \
  --script-tool-budget 50 \
  --session isolated \
  --announce
```

使用 `--script <file|->` 从文件或标准输入读取 JavaScript。超时时间默认为 300 秒，上限为 900 秒；工具预算默认为 50 次调用，上限为 200 次。这些载荷预算与更小的触发器门控评估预算相互独立。

脚本可以返回包含以下可选字段的对象：

- `notify`：通过任务的 `announce`、`webhook` 或 `none` 交付模式交付的文本。如果省略，则不交付任何内容。对于 `main` 任务，该文本会成为系统事件。
- `wake`：`"now"` 请求在将 `notify`（或精简的完成事件）加入队列后立即触发一次 Heartbeat；`"next-heartbeat"` 则将该事件加入队列，供下一次 Heartbeat 处理。
- `state`：JSON 状态，上限为 16 KB，并且仅在成功运行后持久化。下一次运行会以 `trigger.state` 的形式接收其冻结副本，与触发器脚本一致。由于该命名空间只有一个持久化所有者，因此同一任务中的脚本载荷不能与条件触发器结合使用。
- `nextCheck`：持续时间，例如 `"15m"`。它仅适用于启用了节奏控制的任务，并使用与智能体轮次提案相同的节奏限制。

抛出异常、超时、工具预算耗尽、无效结果，以及未启用节奏控制时返回 `nextCheck`，都属于正常的定时任务运行错误：它们会进入运行历史记录、退避和失败警报处理流程，且不会持久化返回的状态。

## 执行方式

| 方式           | `--session` 值   | 运行位置                  | 最适合                        |
| --------------- | ------------------- | ------------------------ | ------------------------------- |
| 主会话    | `main`              | 专用定时唤醒通道 | 提醒、系统事件        |
| 隔离        | `isolated`          | 专用 `cron:<jobId>` | 报告、后台杂务      |
| 当前会话 | `current`           | 创建时绑定   | 感知上下文的重复工作    |
| 自定义会话  | `session:custom-id` | 持久化命名会话 | 基于历史记录持续推进的工作流 |

<AccordionGroup>
  <Accordion title="主会话、隔离会话与自定义会话">
    **主会话**任务会将系统事件加入定时任务自有的运行通道，并可选择唤醒 Heartbeat（`--wake now` 或 `--wake next-heartbeat`）。它们可以使用目标主会话的最近交付上下文来回复，但不会将常规定时任务轮次追加到人类聊天通道，也不会延长目标会话每日重置或空闲重置的新鲜度。**隔离**任务使用全新会话运行专用的智能体轮次。**自定义会话**（`session:xxx`）会跨运行持久保留上下文，从而支持每日站会等基于先前摘要持续推进的工作流。

    主会话定时任务事件是内容完备的系统事件提醒。它们不会自动包含默认 Heartbeat 提示词或 Heartbeat 监控暂存内容；如果提醒应查阅该上下文，请在定时任务事件文本中明确说明。

  </Accordion>
  <Accordion title="隔离任务中的“全新会话”含义">
    每次运行都会使用新的转录记录/会话 ID。OpenClaw 会保留安全偏好设置（思考/快速/详细程度设置、标签、用户明确选择的模型/身份验证覆盖），但不会从较旧的定时任务记录继承环境对话上下文：渠道/群组路由、发送或队列策略、权限提升、来源或 ACP 运行时绑定。如果重复任务需要有意基于同一对话上下文持续推进，请使用 `current` 或 `session:<id>`。
  </Accordion>
  <Accordion title="无人值守运行契约">
    隔离定时任务和钩子智能体轮次明确为无人值守：现场无人进行澄清或审批。最终回复必须是交付成果，而不是计划、确认或输入请求。无需执行任何操作时，智能体返回 `HEARTBEAT_OK`，并明确说明失败情况；重试和失败警报策略由定时任务负责。

    对于受信任的定时任务，如果任务自身的指令有意要求提出问题或制定计划，则这些指令优先，并且智能体可以移除不再需要的任务。外部钩子轮次只接收通用的无人值守契约；跨越外部内容边界时，它们不会收到这种覆盖规则或自我移除指导。

  </Accordion>
  <Accordion title="子智能体和 Discord 交付">
    当隔离定时任务运行协调子智能体时，交付会优先采用最终后代输出，而不是过时的父级中间文本。如果后代仍在运行，OpenClaw 会抑制该父级的部分更新，而不是将其公告出去。

    对于纯文本 Discord 公告目标，OpenClaw 只发送一次规范的最终助手文本，而不会同时重放流式/中间文本和最终答案。媒体和结构化 Discord 载荷仍会单独交付，以免丢失附件和组件。

  </Accordion>
</AccordionGroup>

## 交付和输出

| 模式       | 行为                                                        |
| ---------- | ------------------------------------------------------------------- |
| `announce` | 如果智能体未发送，则将最终文本回退交付至目标 |
| `webhook`  | 将完成事件载荷 POST 到 URL                                |
| `none`     | 运行器不执行回退交付                                         |

使用 `--announce --channel telegram --to "-1001234567890"` 进行渠道交付。对于 Telegram 论坛主题，请使用 `-1001234567890:topic:123`；OpenClaw 也接受 Telegram 自有的 `-1001234567890:123` 简写形式。直接 RPC/配置调用方可以将 `delivery.threadId` 作为字符串或数字传递。Slack/Discord/Mattermost 目标使用明确的前缀（`channel:<id>`、`user:<id>`）。Matrix 房间 ID 区分大小写；请使用 Matrix 提供的确切房间 ID 或 `room:!room:server` 形式。

当公告交付使用 `channel: "last"` 或省略 `channel` 时，像 `telegram:123` 这样的提供商前缀目标可以在定时任务回退到会话历史记录或唯一已配置渠道之前选择渠道。只有已加载插件声明的前缀才是提供商选择器。如果显式指定了 `delivery.channel`，目标前缀必须指定同一提供商；`channel: "whatsapp"` 与 `to: "telegram:123"` 的组合会被拒绝，而不是让 WhatsApp 将 Telegram ID 解释为电话号码。目标类型和服务前缀（`channel:<id>`、`user:<id>`、`imessage:<handle>`、`sms:<number>`）仍然是渠道自有的目标语法，而不是提供商选择器。

对于隔离任务，聊天交付是共享的：如果有可用的聊天路由，即使使用 `--no-deliver`，智能体也可以使用 `message` 工具。如果智能体发送到已配置的目标或当前目标，OpenClaw 会跳过回退公告。否则，`announce`、`webhook` 和 `none` 只控制智能体轮次结束后运行器如何处理最终回复。

当智能体从活跃聊天创建隔离提醒时，OpenClaw 会存储保留下来的实时交付目标，用作回退公告路由。内部会话键可以是小写；如果当前聊天上下文可用，则不会根据这些键重建提供商交付目标。

隐式公告交付使用已配置的渠道允许列表来验证过时目标并重新路由。私信配对存储中的审批对象不是回退自动化接收方；如果定时任务应主动向私信发送消息，请设置 `delivery.to` 或配置渠道的 `allowFrom` 条目。

### 失败通知

失败通知使用独立的目标路径：

- `cron.failureDestination` 设置失败通知的全局默认值。
- `job.delivery.failureDestination` 为每个任务覆盖该设置。
- 如果两者均未设置，并且任务已通过 `announce` 交付，则失败通知会回退到该主要公告目标。
- `delivery.failureDestination` 仅支持 `sessionTarget="isolated"` 任务，除非主要交付模式为 `webhook`。
- `failureAlert.includeSkipped: true` 使任务或全局定时任务警报策略选择接收重复的已跳过运行警报。已跳过的运行使用单独的连续跳过计数器，因此不会影响执行错误退避。
- `openclaw cron edit` 提供每个任务的警报调优选项：`--failure-alert`/`--no-failure-alert`、`--failure-alert-after <n>`、`--failure-alert-channel`、`--failure-alert-to`、`--failure-alert-cooldown`、`--failure-alert-include-skipped`/`--failure-alert-exclude-skipped`、`--failure-alert-mode` 和 `--failure-alert-account-id`。

### 输出语言

定时任务不会根据渠道、区域设置或先前消息推断回复语言。请将语言规则写入定时消息或模板：

```bash
openclaw cron edit <jobId> \
  --message "总结更新。使用中文回复；URL、代码和产品名称保持不变。"
```

对于模板文件，请在渲染后的提示词中保留语言指令，并在任务运行前确认 `{{language}}` 等占位符已填充。如果输出混合了多种语言，请明确规则，例如：“叙述性文本使用中文，技术术语保留英文。”

## CLI 示例

<Tabs>
  <Tab title="一次性提醒">
    ```bash
    openclaw cron add \
      --name "Calendar check" \
      --at "20m" \
      --session main \
      --system-event "Next heartbeat: check calendar." \
      --wake now
    ```
  </Tab>
  <Tab title="重复运行的隔离任务">
    ```bash
    openclaw cron create "0 7 * * *" \
      "Summarize overnight updates." \
      --name "Morning brief" \
      --tz "America/Los_Angeles" \
      --session isolated \
      --announce \
      --channel slack \
      --to "channel:C1234567890"
    ```
  </Tab>
  <Tab title="模型和思考覆盖">
    ```bash
    openclaw cron add \
      --name "Deep analysis" \
      --cron "0 6 * * 1" \
      --tz "America/Los_Angeles" \
      --session isolated \
      --message "Weekly deep analysis of project progress." \
      --model "opus" \
      --thinking high \
      --announce
    ```
  </Tab>
  <Tab title="Webhook 输出">
    ```bash
    openclaw cron create "0 18 * * 1-5" \
      "Summarize today's deploys as JSON." \
      --name "Deploy digest" \
      --webhook "https://example.invalid/openclaw/cron"
    ```
  </Tab>
  <Tab title="命令输出">
    ```bash
    openclaw cron create "*/15 * * * *" \
      --name "Queue depth probe" \
      --command "scripts/check-queue.sh" \
      --command-cwd "/srv/app" \
      --announce \
      --channel telegram \
      --to "-1001234567890"
    ```
  </Tab>
</Tabs>

## 管理任务

```bash
# 列出已启用的任务
openclaw cron list

# 包含已禁用的任务
openclaw cron list --all

# 以 JSON 格式获取一个已存储的任务
openclaw cron get <jobId>

# 显示一个任务，包括解析后的投递路由
openclaw cron show <jobId>

# 启用/禁用而不删除
openclaw cron enable <jobId>
openclaw cron disable <jobId>

# 编辑任务
openclaw cron edit <jobId> --message "更新后的提示词" --model "opus"

# 立即强制运行任务
openclaw cron run <jobId>

# 立即强制运行任务并等待其终止状态
openclaw cron run <jobId> --wait --wait-timeout 10m --poll-interval 2s

# 仅在到期时运行
openclaw cron run <jobId> --due

# 查看运行历史记录
openclaw cron runs --id <jobId> --limit 50

# 查看一次确切的运行
openclaw cron runs --id <jobId> --run-id <runId>

# 删除任务
openclaw cron remove <jobId>

# 选择智能体（多智能体设置）
openclaw cron create "0 6 * * *" "检查运维队列" --name "运维巡检" --session isolated --agent ops
openclaw cron edit <jobId> --clear-agent
```

归档会话（在 Control UI 中，或由操作员管理员调用方通过 `sessions.patch { archived: true }`）会禁用绑定到该会话的所有已启用定时任务：其隔离的 `cron:<jobId>` 会话、`session:<key>` 目标或投递/唤醒 `sessionKey` 通道。恢复会话不会重新启用这些任务；请使用 `openclaw cron enable <jobId>`。存在已启用绑定任务的会话会在 Control UI 侧边栏中显示时钟徽章。

`openclaw cron run <jobId>` 会在手动运行入队后返回。对于关停钩子、维护脚本或其他必须阻塞至已入队运行完成的自动化，请使用 `--wait`；它会轮询返回的 `runId`（默认超时 `10m`，轮询间隔 `2s`），状态为 `ok` 时以 `0` 退出，状态为 `error`、`skipped` 或等待超时时以非零值退出。

智能体 `cron` 工具从 `cron(action: "list")` 返回精简的任务摘要（`id`、`name`、`enabled`、`nextRunAtMs`、`scheduleKind`、`lastRunStatus`）；使用 `cron(action: "get", jobId: "...")` 获取单个任务的完整定义。直接调用 Gateway 网关的调用方可以将 `compact: true` 传给 `cron.list`；省略该参数则保留包含投递预览的完整响应。

`openclaw cron create` 是 `openclaw cron add` 的别名。新任务可以依次使用位置式计划（`"0 9 * * 1"`、`"every 1h"`、`"20m"` 或 ISO 时间戳）和位置式智能体提示词。在 `cron add|create` 或 `cron edit` 上使用 `--webhook <url>`，将已完成运行的有效负载 POST 到 HTTP 端点；webhook 投递不能与聊天投递标志（`--announce`、`--channel`、`--to`、`--thread-id`、`--account`）组合使用。在 `cron edit`、`--clear-channel`、`--clear-to`、`--clear-thread-id` 和 `--clear-account` 上，可分别清除这些路由字段（每个选项与其对应的设置标志同时使用时都会被拒绝）——这与 `--no-deliver` 不同，后者仅禁用运行器的回退投递。

<Note>
模型覆盖说明：

- `openclaw cron add|edit --model ...` 会更改任务选择的模型。
- 如果允许使用该模型，隔离的智能体运行会使用这个确切的提供商/模型。
- 如果不允许使用或无法解析该模型，cron 会因明确的验证错误而使运行失败。
- API `cron.update` 有效负载补丁可将 `model: null` 设为空，以清除已存储任务的模型覆盖。
- `openclaw cron edit <job-id> --clear-model` 会从 CLI 清除该覆盖（效果与 `model: null` 补丁相同），且不能与 `--model` 组合使用。
- 配置的回退链仍然适用，因为 cron `--model` 是任务主模型，而不是会话 `/model` 覆盖。
- `openclaw cron add|edit --fallbacks ...` 会设置有效负载 `fallbacks`，替换该任务已配置的回退模型；`--fallbacks ""` 会禁用回退并使运行采用严格模式。`openclaw cron edit <job-id> --clear-fallbacks` 会清除每任务覆盖。
- 如果普通的 `--model` 没有显式或已配置的回退列表，则不会将智能体主模型静默作为额外的重试目标。

</Note>

## Webhooks

Gateway 网关可以公开 HTTP webhook 端点以接收外部触发。在配置中启用：

```json5
{
  hooks: {
    enabled: true,
    token: "shared-secret",
    path: "/hooks",
  },
}
```

### 身份验证

每个请求都必须通过请求头包含钩子令牌：

- `Authorization: Bearer <token>`（推荐）
- `x-openclaw-token: <token>`

查询字符串令牌会被拒绝。

<AccordionGroup>
  <Accordion title="POST /hooks/wake">
    为主会话将系统事件加入队列：

    ```bash
    curl -X POST http://127.0.0.1:18789/hooks/wake \
      -H 'Authorization: Bearer SECRET' \
      -H 'Content-Type: application/json' \
      -d '{"text":"收到新电子邮件","mode":"now"}'
    ```

    <ParamField path="text" type="string" required>
      事件描述。
    </ParamField>
    <ParamField path="mode" type="string" default="now">
      `now` 或 `next-heartbeat`。
    </ParamField>

  </Accordion>
  <Accordion title="POST /hooks/agent">
    运行一个隔离的智能体轮次：

    ```bash
    curl -X POST http://127.0.0.1:18789/hooks/agent \
      -H 'Authorization: Bearer SECRET' \
      -H 'Content-Type: application/json' \
      -d '{"message":"汇总收件箱","name":"电子邮件","model":"openai/gpt-5.6-sol"}'
    ```

    字段：`message`（必填）、`name`、`agentId`、`sessionKey`（需要 `hooks.allowRequestSessionKey=true`）、`idempotencyKey`、`wakeMode`、`deliver`、`channel`、`to`、`model`、`thinking`、`timeoutSeconds`。

  </Accordion>
  <Accordion title="映射钩子（POST /hooks/<name>）">
    自定义钩子名称通过配置中的 `hooks.mappings` 解析。映射可以使用模板或代码转换，将任意有效负载转换为 `wake` 或 `agent` 操作。
  </Accordion>
</AccordionGroup>

<Warning>
请将钩子端点置于环回地址、tailnet 或受信任的反向代理之后。

- 使用专用的钩子令牌；不要重复使用 Gateway 网关身份验证令牌。
- 将 `hooks.path` 保持在专用子路径上；`/` 会被拒绝。
- 设置 `hooks.allowedAgentIds`，限制钩子可指定的有效智能体；这也包括省略 `agentId` 时使用的默认智能体。
- 除非需要由调用方选择会话，否则请保留 `hooks.allowRequestSessionKey=false`。
- 如果启用 `hooks.allowRequestSessionKey`，还应设置 `hooks.allowedSessionKeyPrefixes`，以约束允许的会话键格式。
- 默认情况下，钩子有效负载会被安全边界包裹。

</Warning>

## Gmail PubSub 集成

通过 Google PubSub 将 Gmail 收件箱触发器连接到 OpenClaw。

<Note>
**先决条件：** `gcloud` CLI、`gog`（gogcli）、已启用 OpenClaw 钩子，以及用于公共 HTTPS 端点的 Tailscale。
</Note>

### 向导设置（推荐）

```bash
openclaw webhooks gmail setup --account openclaw@gmail.com
```

此命令会写入 `hooks.gmail` 配置，启用 Gmail 预设，并默认使用 Tailscale Funnel 作为推送端点（`--tailscale funnel|serve|off`）。

<Warning>
Gmail 预设的每消息会话会分离对话上下文；它不会限制目标智能体的工具或工作区。如果没有设置 `agentId` 的自定义映射，Gmail 钩子将以默认智能体身份运行。

对于不受信任的收件箱，请将钩子路由到专用的读取智能体，仅授予该智能体只读工作区访问权限或完全不授予工作区访问权限，并禁止文件系统写入、shell、浏览器及其他不必要的工具。如果它需要通知主智能体，只允许所需的智能体间移交。请参阅[提示词注入](/zh-CN/gateway/security#prompt-injection)、[多 Agent 沙盒和工具](/zh-CN/tools/multi-agent-sandbox-tools)和 [`tools.agentToAgent`](/zh-CN/gateway/config-tools#toolsagenttoagent)。
</Warning>

### Gateway 网关自动启动

当已设置 `hooks.enabled=true` 和 `hooks.gmail.account` 时，Gateway 网关会在启动时运行 `gog gmail watch serve`，并自动续订监视。设置 `OPENCLAW_SKIP_GMAIL_WATCHER=1` 可选择退出。

### 手动一次性设置

<Steps>
  <Step title="选择 GCP 项目">
    选择拥有 `gog` 所用 OAuth 客户端的 GCP 项目：

    ```bash
    gcloud auth login
    gcloud config set project <project-id>
    gcloud services enable gmail.googleapis.com pubsub.googleapis.com
    ```

  </Step>
  <Step title="创建主题并授予 Gmail 推送访问权限">
    ```bash
    gcloud pubsub topics create gog-gmail-watch
    gcloud pubsub topics add-iam-policy-binding gog-gmail-watch \
      --member=serviceAccount:gmail-api-push@system.gserviceaccount.com \
      --role=roles/pubsub.publisher
    ```
  </Step>
  <Step title="启动监视">
    ```bash
    gog gmail watch start \
      --account openclaw@gmail.com \
      --label INBOX \
      --topic projects/<project-id>/topics/gog-gmail-watch
    ```
  </Step>
</Steps>

### Gmail 模型覆盖

```json5
{
  hooks: {
    gmail: {
      model: "openai/gpt-5.6-sol",
      thinking: "high",
    },
  },
}
```

对于不受信任的收件箱，请使用提供商所提供的最新一代、最高级别模型。以上值仅为示例；该模型必须存在于已配置的目录和允许列表中。

## 配置

```json5
{
  cron: {
    enabled: true,
    store: "~/.openclaw/cron/jobs.json",
    triggers: {
      enabled: false,
    },
    webhookToken: "replace-with-dedicated-webhook-token",
    sessionRetention: "24h",
  },
}
```

`webhookToken` 会在 cron webhook POST 中作为 `Authorization: Bearer <token>` 发送。

`cron.store` 是逻辑存储键和 Doctor 迁移路径，而不是可手动编辑的实时 JSON 文件。任务数据存储在 SQLite 中；请使用 CLI 或 Gateway 网关 API 进行更改。

禁用 cron：`cron.enabled: false` 或 `OPENCLAW_SKIP_CRON=1`。

<AccordionGroup>
  <Accordion title="重试行为">
    **一次性任务重试**：瞬时错误（速率限制、过载、网络、超时、服务器错误）使用内置重试计划。永久性错误会立即禁用任务。

    **周期任务重试**：连续执行错误会按延长的计划退避（30s、60s、5m、15m、60m）。下次成功运行后，退避会重置。

  </Accordion>
  <Accordion title="维护">
    `cron.sessionRetention`（默认为 `24h`，`false` 表示禁用）会清理隔离的运行会话条目。运行历史记录会为每个任务保留最新的 2000 条终止记录；丢失的记录仍保留其 24 小时清理窗口。
  </Accordion>
  <Accordion title="旧版存储迁移">
    升级时，运行 `openclaw doctor --fix`，将旧版 `~/.openclaw/cron/jobs.json`、`jobs-state.json` 和 `runs/*.jsonl` 文件导入 SQLite，并使用 `.migrated` 后缀重命名这些文件。格式错误的任务记录会从运行时中跳过，并复制到 `jobs-quarantine.json`，供以后修复或审查。
  </Accordion>
</AccordionGroup>

## 故障排查

### 命令排查顺序

```bash
openclaw status
openclaw gateway status
openclaw cron status
openclaw cron list
openclaw cron runs --id <jobId> --limit 20
openclaw system heartbeat last
openclaw logs --follow
openclaw doctor
```

<AccordionGroup>
  <Accordion title="Cron 未触发">
    - 检查 `cron.enabled` 和 `OPENCLAW_SKIP_CRON` 环境变量。
    - 确认 Gateway 网关持续运行。
    - 对于 `cron` 计划，请核对时区（`--tz`）与主机时区。
    - 运行输出中的 `reason: not-due` 表示手动运行使用了 `openclaw cron run <jobId> --due` 进行检查，而任务尚未到期。

  </Accordion>
  <Accordion title="定时任务已触发但未投递">
    - 投递模式 `none` 表示不会执行运行器回退发送。当聊天路由可用时，智能体仍可使用 `message` 工具直接发送。
    - 投递目标缺失或无效（`channel`/`to`）表示已跳过出站发送。
    - 对于 Matrix，复制的任务或旧版任务若使用小写的 `delivery.to` 房间 ID，可能会失败，因为 Matrix 房间 ID 区分大小写。请将任务编辑为 Matrix 中确切的 `!room:server` 或 `room:!room:server` 值。
    - 渠道身份验证错误（`unauthorized`、`Forbidden`）表示投递因凭据问题而被阻止。
    - 如果隔离运行仅返回静默令牌（`NO_REPLY` / `no_reply`），OpenClaw 会阻止直接出站投递和回退的排队摘要路径，因此不会向聊天发送任何内容。
    - 如果智能体应自行向用户发送消息，请检查任务是否具有可用路由（`channel: "last"` 且存在先前聊天，或显式指定渠道/目标）。

  </Accordion>
  <Accordion title="定时任务或 Heartbeat 似乎阻止 /new 样式的滚动切换">
    - 每日重置和空闲重置的新鲜度并非基于 `updatedAt`；请参阅[会话管理](/zh-CN/concepts/session#session-lifecycle)。
    - 定时任务唤醒、Heartbeat 运行、Exec 通知和 Gateway 网关记账可能会更新用于路由/状态的会话行，但不会延长 `sessionStartedAt` 或 `lastInteractionAt`。
    - 对于在这些字段存在之前创建的旧版行，如果文件仍然可用，OpenClaw 可以从会话记录 JSONL 的会话标头中恢复 `sessionStartedAt`。没有 `lastInteractionAt` 的旧版空闲行使用该恢复的开始时间作为空闲基准。

  </Accordion>
  <Accordion title="时区注意事项">
    - 未设置 `--tz` 的定时任务使用 Gateway 网关主机的时区。
    - 未指定时区的 `at` 计划按 UTC 处理。
    - Heartbeat `activeHours` 使用已配置的时区解析方式。

  </Accordion>
</AccordionGroup>

## 相关内容

- [自动化](/zh-CN/automation) — 快速了解所有自动化机制
- [后台任务](/zh-CN/automation/tasks) — 定时任务执行的任务账本
- [Heartbeat](/zh-CN/gateway/heartbeat) — 周期性的主会话轮次
- [时区](/zh-CN/concepts/timezone) — 时区配置
