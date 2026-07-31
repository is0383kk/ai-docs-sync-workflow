---
read_when:
    - 你希望将 OpenClaw 模型使用情况、消息流或会话指标发送到 OpenTelemetry 收集器
    - 你正在将追踪、指标或日志接入 Grafana、Datadog、Honeycomb、New Relic、Tempo 或其他 OTLP 后端系统
    - 你需要准确的指标名称、Span 名称或属性结构来构建仪表板或警报
summary: 通过 diagnostics-otel 插件将 OpenClaw 诊断数据导出到 OpenTelemetry 收集器或 stdout JSONL
title: OpenTelemetry 导出
x-i18n:
    generated_at: "2026-07-26T06:09:56Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 6ed37f094c6c151379d8e0aaa2633b3ebebdb08b7dcbc9403c4bdeb6e5b8cf76
    source_path: gateway/opentelemetry.md
    workflow: 16
---

OpenClaw 通过官方 `diagnostics-otel` 插件导出诊断数据，
使用 **OTLP/HTTP (protobuf)**。日志也可以写入 stdout JSONL，以供
容器和沙箱日志流水线使用。任何接受
OTLP/HTTP 的收集器或后端都无需更改代码即可使用。有关本地文件日志，请参阅
[日志](/zh-CN/logging)。

- **诊断事件**是由 Gateway 网关和内置插件针对模型运行、消息流、会话、队列
  和 exec 发出的结构化进程内记录。
- **`diagnostics-otel`** 订阅这些事件，并通过 OTLP/HTTP 将其导出为
  OpenTelemetry **指标**、**追踪**和**日志**，还可以
  将日志记录镜像到 stdout JSONL。
- 当提供商传输支持自定义标头时，**提供商调用**会从 OpenClaw
  可信的模型调用 span 上下文中接收 W3C `traceparent` 标头。
  插件发出的追踪上下文不会被传播。
- 仅当诊断功能和插件都已启用时才会挂载导出器，
  因此默认情况下进程内开销接近于零。

## 快速开始

```bash
openclaw plugins install clawhub:@openclaw/diagnostics-otel
```

```json5
{
  plugins: {
    allow: ["diagnostics-otel"],
    entries: {
      "diagnostics-otel": { enabled: true },
    },
  },
  diagnostics: {
    enabled: true,
    otel: {
      enabled: true,
      endpoint: "http://otel-collector:4318",
      protocol: "http/protobuf",
      serviceName: "openclaw-gateway",
      traces: true,
      metrics: true,
      logs: true,
      sampleRate: 0.2,
      flushIntervalMs: 60000,
    },
  },
}
```

或者通过 CLI 启用插件：`openclaw plugins enable diagnostics-otel`。

<Note>
`protocol` 仅支持 `http/protobuf`。由于 `traces` 和 `metrics` 默认启用，任何其他值（包括 `grpc`）都会通过 `unsupported protocol` 警告中止整个 diagnostics-otel 订阅，这也会停止 stdout 日志导出。如果只希望将 `logsExporter: "stdout"` 与非 OTLP 协议值结合使用，请显式设置 `traces: false` 和 `metrics: false`。
</Note>

## 导出的信号

| 信号        | 包含的内容                                                                                                                                                                                              |
| ----------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| **指标** | 用于令牌用量、成本、运行时长、故障转移、技能使用情况、消息流、Talk 事件、队列通道、会话状态/恢复、工具执行、exec、记忆、存活状态和导出器健康状况的计数器/直方图。 |
| **追踪**  | 用于模型使用情况、模型调用、harness 生命周期、技能使用情况、工具执行、exec、webhook/消息处理、上下文组装和工具循环的 span。                                                      |
| **日志**    | 启用 `diagnostics.otel.logs` 时，通过 OTLP 或 stdout JSONL 导出的结构化 `logging.file` 记录；除非显式启用内容捕获，否则不会导出日志正文。                          |

可以独立切换 `traces`、`metrics` 和 `logs`。当 `diagnostics.otel.enabled` 为 true 时，追踪和指标
默认开启；日志默认关闭，
仅当 `diagnostics.otel.logs` 被显式设为 `true` 时才会导出。日志导出
默认为 OTLP；将 `diagnostics.otel.logsExporter` 设为 `stdout` 可在
stdout 上输出 JSONL，或设为 `both` 以同时使用两者。

## 配置参考

```json5
{
  diagnostics: {
    enabled: true,
    otel: {
      enabled: true,
      endpoint: "http://otel-collector:4318",
      tracesEndpoint: "http://otel-collector:4318/v1/traces",
      metricsEndpoint: "http://otel-collector:4318/v1/metrics",
      logsEndpoint: "http://otel-collector:4318/v1/logs",
      protocol: "http/protobuf", // grpc 会禁用 OTLP 导出
      serviceName: "openclaw-gateway", // 未设置时回退到 OTEL_SERVICE_NAME，然后回退到 "openclaw"
      headers: { "x-collector-token": "..." },
      traces: true,
      metrics: true,
      logs: true,
      logsExporter: "otlp", // otlp | stdout | both
      sampleRate: 0.2, // 根 span 采样器，0.0..1.0
      flushIntervalMs: 60000, // 指标导出间隔（最小 1000ms）
      captureContent: {
        enabled: false,
        inputMessages: false,
        outputMessages: false,
        toolInputs: false,
        toolOutputs: false,
        systemPrompt: false,
        toolDefinitions: false,
      },
    },
  },
}
```

### 环境变量

| 变量                                                                                                          | 用途                                                                                                                                                                                                                                                                                                        |
| ----------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `OTEL_EXPORTER_OTLP_ENDPOINT`                                                                                     | 配置键未设置时，作为 `diagnostics.otel.endpoint` 的回退值。                                                                                                                                                                                                                                         |
| `OTEL_EXPORTER_OTLP_TRACES_ENDPOINT` / `OTEL_EXPORTER_OTLP_METRICS_ENDPOINT` / `OTEL_EXPORTER_OTLP_LOGS_ENDPOINT` | 对应的 `diagnostics.otel.*Endpoint` 配置键未设置时使用的信号专用端点回退值。信号专用配置优先于信号专用环境变量，而信号专用环境变量优先于共享端点。                                                                                                         |
| `OTEL_SERVICE_NAME`                                                                                               | 配置键未设置时，作为 `diagnostics.otel.serviceName` 的回退值。默认服务名称为 `openclaw`。                                                                                                                                                                                                  |
| `OTEL_EXPORTER_OTLP_PROTOCOL`                                                                                     | `diagnostics.otel.protocol` 未设置时，作为传输协议的回退值。只有 `http/protobuf` 会启用导出。                                                                                                                                                                                                 |
| `OTEL_SEMCONV_STABILITY_OPT_IN`                                                                                   | 设为 `gen_ai_latest_experimental` 可发出最新的 GenAI 推理 span 结构：`{gen_ai.operation.name} {gen_ai.request.model}` span 名称、`CLIENT` span 类型，并使用 `gen_ai.provider.name` 取代旧版 `gen_ai.system`。无论如何，GenAI 指标始终使用有界的低基数属性。 |
| `OPENCLAW_OTEL_PRELOADED`                                                                                         | 当其他预加载项或宿主进程已注册全局 OpenTelemetry SDK 时，设为 `1`。插件随后会跳过自己的 NodeSDK 生命周期，但仍会连接诊断监听器，并遵循 `traces`/`metrics`/`logs`。                                                                                    |

## 隐私和内容捕获

默认**不会**导出原始模型/工具内容。Span 携带有界
标识符（渠道、提供商、模型、错误类别、仅哈希请求 ID、
工具来源、工具所有者、技能名称/来源），绝不会包含提示词文本、
响应文本、工具输入、工具输出、技能文件路径或会话键。
类似带作用域 Agent 会话键的值（例如以
`agent:` 开头）在低基数属性中会被替换为 `unknown`。默认情况下，OTLP 日志
记录会保留严重性、日志记录器、代码位置、可信追踪上下文和
经过净化的属性；仅当 `diagnostics.otel.captureContent` 为布尔值 `true` 时，
才会导出原始日志消息正文。细粒度
`captureContent.*` 子键绝不会启用日志正文。Talk 指标仅导出
有界的事件元数据（模式、传输协议、提供商、事件类型），不会导出
文字记录、音频载荷、会话 ID、轮次 ID、调用 ID、房间 ID 或
交接令牌。

出站模型请求可能包含 W3C `traceparent` 标头，该标头仅
从当前模型调用的 OpenClaw 自有诊断追踪上下文生成。
现有调用方提供的 `traceparent` 标头会被替换，因此插件或
自定义提供商选项无法伪造跨服务追踪继承关系。

仅当你的收集器和保留策略获准存储提示词、响应、工具或
系统提示词文本时，才将 `diagnostics.otel.captureContent.*` 设为 `true`。
每个子键相互独立：

- `inputMessages` - 用户提示词内容。
- `outputMessages` - 模型响应内容。
- `toolInputs` - 工具参数载荷。
- `toolOutputs` - 工具结果载荷。
- `systemPrompt` - 组装后的系统/开发者提示词。
- `toolDefinitions` - 模型工具名称、描述和 schema。

启用任何子键后，模型和工具 span 仅会获得该类别对应的有界、经过脱敏的
`openclaw.content.*` 属性。

<Note>
布尔值 `captureContent: true` 会同时启用 `inputMessages`、`outputMessages`、`toolInputs`、`toolOutputs`、`toolDefinitions` 和 OTLP 日志正文，但**不会**启用 `systemPrompt`；如果还需要组装后的系统提示词，请显式设置 `captureContent.systemPrompt: true`。
</Note>

`toolInputs`/`toolOutputs` 内容会针对内置 Agent
运行时的工具执行进行捕获（在完成/错误 span 上为 `openclaw.content.tool_input` 和
`gen_ai.tool.call.arguments`；
在完成 span 上为 `openclaw.content.tool_output` 和 `gen_ai.tool.call.result`）。
`openclaw.content.*` 名称仍是稳定的 OpenClaw 属性名称；
`gen_ai.tool.call.*` 副本则为 semconv 原生查看器镜像这些属性。
外部 harness 工具调用（Codex、Claude CLI）会发出
不含内容载荷的 `tool.execution.*` span。捕获的内容通过
仅限可信监听器的渠道传输，绝不会放入公共诊断事件
总线。

## 采样和刷新

- **追踪：** `diagnostics.otel.sampleRate` 仅在根 span 上设置一个 `TraceIdRatioBasedSampler`
  （`0.0` 全部丢弃，`1.0` 全部保留）。未设置时使用
  OpenTelemetry SDK 默认值（始终开启）。
- **指标：** `diagnostics.otel.flushIntervalMs`（下限为
  `1000`）；未设置时使用 SDK 的周期性导出默认值。
- **日志：** OTLP 日志遵循 `logging.level`（文件日志级别），并使用
  诊断日志记录脱敏路径，而非控制台格式。高流量
  安装应优先使用 OTLP 收集器采样/过滤，而不是本地
  采样。当你的平台已将 stdout/stderr 发送到日志处理器
  且没有 OTLP 日志收集器时，请设置 `diagnostics.otel.logsExporter: "stdout"`。
  stdout 记录每行一个 JSON 对象，其中包含 `ts`、`signal`、
  `service.name`、严重性、正文、脱敏后的属性，以及可用时可信的追踪
  字段。
- **文件日志关联：** 当日志调用携带有效的
  诊断追踪上下文时，JSONL 文件日志会包含顶层 `traceId`、
  `spanId`、`parentSpanId` 和 `traceFlags`，使日志处理器能够将本地日志行与
  导出的 span 关联起来。
- **请求关联：** Gateway 网关 HTTP 请求和 WebSocket 帧会创建
  内部请求追踪作用域。该作用域内的日志和诊断事件
  默认继承请求追踪，而智能体运行和模型调用
  span 会创建为其子项，使提供商 `traceparent` 标头保持在同一
  追踪中。
- **模型调用关联：** `openclaw.model.call` span 默认包含安全的提示词
  组件大小，并在提供商结果公开用量时包含每次调用的 token 属性。`openclaw.model.usage` 仍是运行级
  计费 span，用于汇总成本、上下文和渠道仪表板；当发出该 span 的运行时具有可信的
  追踪上下文时，它会保持在同一诊断追踪中。

### 模型调用观测单位

每个 `openclaw.model.call` span 都通过
`openclaw.model_call.observation_unit` 标识其生命周期测量的内容：

- `request` - 一次可观测的模型/提供商请求。原生嵌入式模型
  调用使用此单位；为兼容较旧或外部的发出方，导出器会将缺失值视为 `request`。
- `turn` - 一次不透明的智能体 CLI 轮次，其中可能包含隐藏的模型请求、
  重试、工具工作或后台工作。Claude Code CLI 和 Codex app-server
  调用使用此单位。

这两个单位都仍属于模型调用 span，因此追踪后端可以呈现模型输入、
输出、用量和层级结构。请求 span 使用从 API 派生的 GenAI 操作
（`chat`、`generate_content` 或 `text_completion`），而轮次 span 使用
`gen_ai.operation.name = invoke_agent`。两者都计入
`gen_ai.client.operation.duration`，其中操作名称会将直接
请求延迟与完整轮次延迟区分开来。OpenClaw 的 OTEL 模型调用
指标还包含 `openclaw.model_call.observation_unit`；Prometheus
模型调用指标公开等效的 `observation_unit` 标签。

### Claude Code CLI 模型调用保真度

Claude Code CLI 轮次会发出一个合成的轮次级 `openclaw.model.call`
span。这些并非 Anthropic HTTP 请求 span。它们使用 `openclaw.api =
claude-code`、`openclaw.model_call.observation_unit = turn`，并将
操作标识为 `gen_ai.operation.name = invoke_agent`。它们通过
`openclaw.transport` 标识 OpenClaw 的 CLI 边界：

- `stdio` - 一次性本地 Claude Code 进程。
- `stdio-live` - 托管的持久 Claude stdio 会话中的一个轮次。
- `paired-node-cli` - 委托给已配对
  节点的一次性 Claude Code 执行。

仅当进程诊断分发器已启用，且已附加内部或可信事件监听器时，
才会实例化 Claude CLI 诊断。没有可观测性插件或其他活动监听器时，
Claude CLI 轮次会跳过合成追踪层级、内容缓冲区和诊断流字节
计数。启用内容捕获时，提示词和系统提示词字段
各自上限为 128 KiB；助手输出在最多 200 个信封中总计上限为 128 KiB，
其中为最终可见的回退响应预留 16 KiB 和一个条目。
达到限制时会记录一个截断标记。

OpenClaw 为 Claude CLI 轮次提供与其他
智能体运行时相同的所有权层级：`openclaw.harness.run`（`openclaw.harness.id = claude-cli`）
包含 `openclaw.run`，后者包含 Claude `openclaw.model.call`
span。harness 和运行 span 是合成的 OpenClaw 轮次边界，而不是
Claude Code 的内部阶段。一次性轮次和托管 stdio 轮次使用相同的
层级；真实的新会话重试会在同一次 OpenClaw 运行中创建另一个模型调用子项。

span 在 OpenClaw 接纳准备好的 CLI 轮次时开始，并仅在
该轮次成功或失败后结束。对于托管会话，当 Claude 报告仍持有结果的后台智能体或
工作流时，临时成功结果不会结束该 span；排空后的最终结果才会结束它。中止、超时、进程失败、
输出/解析失败及其他轮次失败，都会使同一个 span 以错误状态结束。

Claude Code 会报告每条助手消息的用量，也可能在其终止结果中报告累计
用量。OpenClaw 回复计费继续使用
最后一条助手消息，因此现有成本语义不会改变；轮次级模型调用 span 在可用时使用终止时的累计用量，
包括缓存读取和缓存创建 token。

对于这些 CLI span，字节和计时字段描述的是可观测的 OpenClaw
CLI 边界：

- `openclaw.model_call.request_bytes` 是通过一次性 stdin/argv 发送的提示词值，
  或托管 stdio JSONL 用户信封的 UTF-8 大小。它
  不是 Claude Code 隐藏模型请求的大小。
- `openclaw.model_call.response_bytes` 是轮次期间观察到的 Claude CLI stdout 的 UTF-8 大小。
  它不是 Anthropic HTTP 响应大小。
- `openclaw.model_call.time_to_first_byte_ms` 是首次观察到
  Claude CLI stdout 或 stderr 输出所需的时间。它不是网络 TTFB。

启用匹配的细粒度 `captureContent` 字段后，该 span 会通过
`gen_ai.input.messages`、`gen_ai.output.messages` 和
`gen_ai.system_instructions` 导出 OpenClaw 发送给 Claude Code 的实际提示词、OpenClaw 追加的系统
提示词，以及可见的助手文本/推理/工具调用标识。
Claude 助手信封会省略工具参数、不透明的思维签名和
工具结果。OpenClaw 不声称能够访问 Claude Code 的私有系统提示词、隐藏的已恢复或
已压缩请求负载、原生内部工具 schema、原始 Anthropic HTTP
请求、内部重试、上游请求 ID 或真实网络 TTFB。由于
Claude Code 无法准确公开其实际原生工具定义，
这些 span 不会填充 `gen_ai.tool.definitions`。

即使启用了工具内容捕获，外部 Claude harness 工具 span 仍仅包含元数据。
与所有模型 span 一样，捕获的 Claude CLI 内容使用
仅限可信监听器的路径，以及导出器现有的脱敏和大小
限制；内容默认保持关闭。

## 导出的指标

### 模型用量

- `openclaw.tokens`（计数器，属性：`openclaw.token`、`openclaw.channel`、`openclaw.provider`、`openclaw.model`、`openclaw.agent`）
- `openclaw.cost.usd`（计数器，属性：`openclaw.channel`、`openclaw.provider`、`openclaw.model`）
- `openclaw.run.duration_ms`（直方图，属性：`openclaw.channel`、`openclaw.provider`、`openclaw.model`）
- `openclaw.context.tokens`（直方图，属性：`openclaw.context`、`openclaw.channel`、`openclaw.provider`、`openclaw.model`）
- `gen_ai.client.token.usage`（直方图，GenAI 语义约定指标，属性：`gen_ai.token.type` = `input`/`output`、`gen_ai.provider.name`、`gen_ai.operation.name`、`gen_ai.request.model`）
- `gen_ai.client.operation.duration`（直方图，秒，用于模型请求和合成智能体轮次的 GenAI 语义约定指标；属性：`gen_ai.provider.name`、`gen_ai.operation.name`、`gen_ai.request.model`、可选的 `error.type`；轮次观测使用 `gen_ai.operation.name = invoke_agent`）
- `openclaw.model_call.duration_ms`（直方图，属性：`openclaw.provider`、`openclaw.model`、`openclaw.api`、`openclaw.transport`、`openclaw.model_call.observation_unit`，以及分类错误上的 `openclaw.errorCategory` 和 `openclaw.failureKind`）
- `openclaw.model_call.request_bytes`（直方图，最终模型请求负载的 UTF-8 字节大小；对于 Claude Code CLI，指上述可观测的提示词输入/信封；不含原始负载内容）
- `openclaw.model_call.response_bytes`（直方图，流式响应分块负载的 UTF-8 字节大小；高频文本、思考和工具调用增量仅计算递增的 `delta` 字节；对于 Claude Code CLI，指观察到的 stdout 字节；不含原始响应内容）
- `openclaw.model_call.time_to_first_byte_ms`（直方图，首个流式响应事件前经过的时间；对于 Claude Code CLI，指首次可观测的 CLI 输出，而非网络 TTFB）
- `openclaw.model.failover`（计数器，属性：`openclaw.provider`、`openclaw.model`、`openclaw.failover.to_provider`、`openclaw.failover.to_model`、`openclaw.failover.reason`、`openclaw.failover.suspended`、`openclaw.lane`）
- `openclaw.skill.used`（计数器，属性：`openclaw.skill.name`、`openclaw.skill.source`、`openclaw.skill.activation`、可选的 `openclaw.agent`、可选的 `openclaw.toolName`）

### 消息流

- `openclaw.webhook.received`（计数器，属性：`openclaw.channel`、`openclaw.webhook`）
- `openclaw.webhook.error`（计数器，属性：`openclaw.channel`、`openclaw.webhook`）
- `openclaw.webhook.duration_ms`（直方图，属性：`openclaw.channel`、`openclaw.webhook`）
- `openclaw.message.queued`（计数器，属性：`openclaw.channel`、`openclaw.source`）
- `openclaw.message.received`（计数器，属性：`openclaw.channel`、`openclaw.source`）
- `openclaw.message.dispatch.started`（计数器，属性：`openclaw.channel`、`openclaw.source`）
- `openclaw.message.dispatch.completed`（计数器，属性：`openclaw.channel`、`openclaw.outcome`、`openclaw.reason`、`openclaw.source`）
- `openclaw.message.dispatch.duration_ms`（直方图，属性：`openclaw.channel`、`openclaw.outcome`、`openclaw.reason`、`openclaw.source`）
- `openclaw.message.processed`（计数器，属性：`openclaw.channel`、`openclaw.outcome`）
- `openclaw.message.duration_ms`（直方图，属性：`openclaw.channel`、`openclaw.outcome`）
- `openclaw.message.delivery.started`（计数器，属性：`openclaw.channel`、`openclaw.delivery.kind`）
- `openclaw.message.delivery.duration_ms`（直方图，属性：`openclaw.channel`、`openclaw.delivery.kind`、`openclaw.outcome`、`openclaw.errorCategory`）

### Talk

- `openclaw.talk.event`（计数器，属性：`openclaw.talk.event_type`、`openclaw.talk.mode`、`openclaw.talk.transport`、`openclaw.talk.brain`、`openclaw.talk.provider`）
- `openclaw.talk.event.duration_ms`（直方图，属性：与 `openclaw.talk.event` 相同；当 Talk 事件报告持续时间时发出）
- `openclaw.talk.audio.bytes`（直方图，属性：与 `openclaw.talk.event` 相同；针对报告字节长度的 Talk 音频帧事件发出）

### 队列和会话

- `openclaw.queue.lane.enqueue`（计数器，属性：`openclaw.lane`）
- `openclaw.queue.lane.dequeue`（计数器，属性：`openclaw.lane`）
- `openclaw.queue.depth`（直方图，属性：`openclaw.lane` 或 `openclaw.channel=heartbeat`）
- `openclaw.queue.wait_ms`（直方图，属性：`openclaw.lane`）
- `openclaw.session.state`（计数器，属性：`openclaw.state`、`openclaw.reason`）
- `openclaw.session.stuck`（计数器，属性：`openclaw.state`；针对可恢复的过期会话记录状态发出）
- `openclaw.session.stuck_age_ms`（直方图，属性：`openclaw.state`；针对可恢复的过期会话记录状态发出）
- `openclaw.session.turn.created`（计数器，属性：`openclaw.agent`、`openclaw.channel`、`openclaw.trigger`）
- `openclaw.session.recovery.requested`（计数器，属性：`openclaw.state`、`openclaw.action`、`openclaw.active_work_kind`、`openclaw.reason`）
- `openclaw.session.recovery.completed`（计数器，属性：`openclaw.state`、`openclaw.action`、`openclaw.status`、`openclaw.active_work_kind`、`openclaw.reason`）
- `openclaw.session.recovery.age_ms`（直方图，属性：与对应的恢复计数器相同）
- `openclaw.run.attempt`（计数器，属性：`openclaw.attempt`）

### 会话活性遥测

当 OpenClaw 观察到回复、工具、状态、分块或 ACP 运行时进度时，`processing` 会话不会逐渐接近内置活性阈值。输入状态保活不算作进度，因此仍然可以检测到无响应的模型或 harness。

OpenClaw 根据仍可观察到的工作对会话进行分类：

- `session.long_running`：活跃的嵌入式工作、模型调用或工具调用
  仍在取得进展。由所有者管理的无输出模型调用在达到内置中止阈值前也会报告为长时间运行，因此，只要仍可观察中止状态，缓慢或非流式传输的模型提供商就不会看起来像停滞的 Gateway 网关会话。
- `session.stalled`：存在活跃工作，但活跃运行近期未报告
  进度。由所有者管理的模型调用会在达到或超过内置中止阈值时从 `session.long_running` 切换到
  `session.stalled`；没有所有者的
  过期模型/工具活动不会被视为无害的长时间运行工作。
  停滞的嵌入式运行最初仅受观察；当超过中止阈值且仍无进度时，
  将进入中止排空状态，以便该通道后方排队的轮次可以恢复。
- `session.stuck`：没有活跃工作的过期会话记录状态，或存在过期且
  无所有者模型/工具活动的空闲排队会话。恢复门控通过后，会立即释放
  受影响的会话通道。

恢复会发出结构化的 `session.recovery.requested` 和
`session.recovery.completed` 事件。仅在发生会改变状态的恢复结果（`aborted` 或 `released`）后，并且
同一处理代仍为当前代时，诊断会话状态才会标记为空闲。

只有 `session.stuck` 会发出 `openclaw.session.stuck` 计数器、
`openclaw.session.stuck_age_ms` 直方图和 `openclaw.session.stuck`
span。当会话保持不变时，重复的 `session.stuck` 诊断会进行退避，因此仪表板应针对持续增长发出警报，而不是
针对每次 Heartbeat 触发警报。有关配置选项和默认值，请参阅
[配置参考](/zh-CN/gateway/configuration-reference#diagnostics)。

活性警告还会发出：

- `openclaw.liveness.warning`（计数器，属性：`openclaw.liveness.reason`）
- `openclaw.liveness.event_loop_delay_p99_ms`（直方图，属性：`openclaw.liveness.reason`）
- `openclaw.liveness.event_loop_delay_max_ms`（直方图，属性：`openclaw.liveness.reason`）
- `openclaw.liveness.event_loop_utilization`（直方图，属性：`openclaw.liveness.reason`）
- `openclaw.liveness.cpu_core_ratio`（直方图，属性：`openclaw.liveness.reason`）

### Harness 生命周期

- `openclaw.harness.duration_ms`（直方图，属性：`openclaw.harness.id`、`openclaw.harness.plugin`、`openclaw.outcome`，发生错误时还包括 `openclaw.harness.phase`）

### 工具执行和循环检测

- `openclaw.tool.execution.duration_ms`（直方图，属性：`gen_ai.tool.name`、`openclaw.toolName`、`openclaw.tool.source`、`openclaw.tool.owner`、`openclaw.tool.params.kind`，发生错误时还包括 `openclaw.errorCategory`）
- `openclaw.tool.execution.blocked`（计数器，属性：`gen_ai.tool.name`、`openclaw.toolName`、`openclaw.tool.source`、`openclaw.tool.owner`、`openclaw.tool.params.kind`、`openclaw.deniedReason`）
- `openclaw.tool.loop`（计数器，属性：`openclaw.toolName`、`openclaw.loop.level`、`openclaw.loop.action`、`openclaw.loop.detector`、`openclaw.loop.count`，可选 `openclaw.loop.paired_tool`；检测到重复的工具调用循环时发出）

### Exec

- `openclaw.exec.duration_ms`（直方图，属性：`openclaw.exec.target`、`openclaw.exec.mode`、`openclaw.outcome`、`openclaw.failureKind`）

### 诊断内部机制（内存、载荷、导出器健康状态）

- `openclaw.payload.large`（计数器，属性：`openclaw.payload.surface`、`openclaw.payload.action`、`openclaw.channel`、`openclaw.plugin`、`openclaw.reason`）
- `openclaw.payload.large_bytes`（直方图，属性：与 `openclaw.payload.large` 相同）
- `openclaw.memory.rss_bytes` / `openclaw.memory.heap_used_bytes` / `openclaw.memory.heap_total_bytes` / `openclaw.memory.external_bytes` / `openclaw.memory.array_buffers_bytes`（直方图，无属性；进程内存样本）
- `openclaw.memory.pressure`（计数器，属性：`openclaw.memory.level`、`openclaw.memory.reason`）
- `openclaw.diagnostic.async_queue.dropped`（计数器，属性：`openclaw.diagnostic.async_queue.drop_class`；内部诊断队列背压导致的丢弃）
- `openclaw.telemetry.exporter.events`（计数器，属性：`openclaw.exporter`、`openclaw.signal`、`openclaw.status`，可选 `openclaw.reason`、可选 `openclaw.errorCategory`；导出器生命周期/故障自遥测）

## 导出的 span

- `openclaw.model.usage`
  - `openclaw.channel`、`openclaw.provider`、`openclaw.model`
  - `openclaw.tokens.*`（输入/输出/缓存读取/缓存写入/总计）
  - 默认使用 `gen_ai.system`，选择启用最新的 GenAI 语义约定时使用 `gen_ai.provider.name`
  - `gen_ai.request.model`、`gen_ai.operation.name`、`gen_ai.usage.*`
- `openclaw.run`
  - `openclaw.outcome`、`openclaw.channel`、`openclaw.provider`、`openclaw.model`、`openclaw.errorCategory`
- `openclaw.model.call`
  - 默认使用 `gen_ai.system`，选择启用最新的 GenAI 语义约定时使用 `gen_ai.provider.name`
  - `gen_ai.request.model`、`gen_ai.operation.name`、`openclaw.provider`、`openclaw.model`、`openclaw.api`、`openclaw.transport`、`openclaw.model_call.observation_unit`（`request` 或 `turn`）
  - `openclaw.errorCategory`、`error.type`，发生错误时还包括可选的 `openclaw.failureKind`
  - `openclaw.model_call.request_bytes`、`openclaw.model_call.response_bytes`、`openclaw.model_call.time_to_first_byte_ms`
  - `openclaw.model_call.prompt.input_messages_count`、`openclaw.model_call.prompt.input_messages_chars`、`openclaw.model_call.prompt.system_prompt_chars`、`openclaw.model_call.prompt.tool_definitions_count`、`openclaw.model_call.prompt.tool_definitions_chars`、`openclaw.model_call.prompt.total_chars`（仅包含安全的组件大小，不包含提示词文本）
  - 当结果包含该请求或聚合轮次的用量时，包括 `openclaw.model_call.usage.*` 和 `gen_ai.usage.*`
  - 当上游提供商结果公开请求 ID 时，span 事件 `openclaw.provider.request` 带有属性 `openclaw.upstreamRequestIdHash`（有界、基于哈希）；绝不会导出原始 ID
  - 使用 `OTEL_SEMCONV_STABILITY_OPT_IN=gen_ai_latest_experimental` 时，请求 span 使用最新的 GenAI 推理 span 名称 `{gen_ai.operation.name} {gen_ai.request.model}`。轮次 span 使用 `invoke_agent`，因为 OpenClaw 不会从不透明的 CLI 边界声称原生智能体名称。两者都使用 `CLIENT` span 类型，而不是 `openclaw.model.call`。
- `openclaw.harness.run`
  - `openclaw.harness.id`、`openclaw.harness.plugin`、`openclaw.outcome`、`openclaw.provider`、`openclaw.model`、`openclaw.channel`
  - 完成时：`openclaw.harness.result_classification`、`openclaw.harness.yield_detected`、`openclaw.harness.items.started`、`openclaw.harness.items.completed`、`openclaw.harness.items.active`
  - 发生错误时：`openclaw.harness.phase`、`openclaw.errorCategory`、可选 `openclaw.harness.cleanup_failed`
- `openclaw.tool.execution`
  - `gen_ai.tool.name`、`gen_ai.operation.name`（`execute_tool`）、`openclaw.toolName`、`openclaw.tool.source`、可选 `gen_ai.tool.call.id`、`openclaw.tool.owner`、`openclaw.tool.params.*`
  - 发生错误时可选 `openclaw.errorCategory`/`openclaw.errorCode`，被策略或沙箱拒绝时包括 `openclaw.deniedReason` 和 `openclaw.outcome=blocked`
- `openclaw.exec`
  - `openclaw.exec.target`、`openclaw.exec.mode`、`openclaw.outcome`、`openclaw.failureKind`、`openclaw.exec.command_length`、`openclaw.exec.exit_code`、`openclaw.exec.exit_signal`、`openclaw.exec.timed_out`
- `openclaw.webhook.processed`
  - `openclaw.channel`、`openclaw.webhook`
- `openclaw.webhook.error`
  - `openclaw.channel`、`openclaw.webhook`、`openclaw.error`
- `openclaw.message.processed`
  - `openclaw.channel`、`openclaw.outcome`、`openclaw.reason`
- `openclaw.message.delivery`
  - `openclaw.channel`、`openclaw.delivery.kind`、`openclaw.outcome`、`openclaw.errorCategory`、`openclaw.delivery.result_count`
- `openclaw.session.stuck`
  - `openclaw.state`、`openclaw.ageMs`、`openclaw.queueDepth`
- `openclaw.context.assembled`
  - `openclaw.prompt.size`、`openclaw.history.size`、`openclaw.context.tokens`、`openclaw.errorCategory`（不包含提示词、历史记录、响应或会话键内容）
- `openclaw.tool.loop`
  - `openclaw.toolName`、`openclaw.loop.level`、`openclaw.loop.action`、`openclaw.loop.detector`、`openclaw.loop.count`、可选 `openclaw.loop.paired_tool`（不包含循环消息、参数或工具输出）
- `openclaw.memory.pressure`
  - `openclaw.memory.level`、`openclaw.memory.reason`、`openclaw.memory.rss_bytes`、`openclaw.memory.heap_used_bytes`、`openclaw.memory.heap_total_bytes`、`openclaw.memory.external_bytes`、`openclaw.memory.array_buffers_bytes`、可选 `openclaw.memory.threshold_bytes`/`openclaw.memory.rss_growth_bytes`/`openclaw.memory.window_ms`

明确启用内容捕获后，模型和工具 span 还可以
为你选择启用的特定内容类别包含有界且经过脱敏的 `openclaw.content.*`
属性。

## 诊断事件目录

以下事件为上述指标和 span 提供支持，或可供插件直接
订阅。`run.progress` 和 `run.execution_phase` 是仅供直接订阅的
生命周期信号；diagnostics-otel 插件不会将其作为
独立的 OTLP 信号导出。事件种类和 `run.execution_phase.phase` 值采用
增量扩展方式。TypeScript 使用方应保留默认分支，而不要假定
任一联合类型会永久穷尽所有情况。

**模型用量**

- `model.usage` - token、成本、持续时间、上下文、提供商/模型/渠道、
  会话 ID。`usage` 是用于成本和遥测的提供商/轮次计量；
  `context.used` 是当前提示词/上下文快照，在涉及缓存输入或工具循环调用时，
  可能低于提供商的 `usage.total`。

**消息流**

- `webhook.received` / `webhook.processed` / `webhook.error`
- `message.queued` / `message.processed`
- `message.delivery.started` / `message.delivery.completed` / `message.delivery.error`

**队列和会话**

- `queue.lane.enqueue` / `queue.lane.dequeue`
- `session.state` / `session.long_running` / `session.stalled` / `session.stuck`
- `run.attempt` / `run.progress`
- `run.execution_phase`（公开的、与会话关联的嵌入式运行器启动里程碑）
- `diagnostic.heartbeat`（聚合计数器：webhook/队列/会话）

**Harness 生命周期**

- `harness.run.started` / `harness.run.completed` / `harness.run.error` -
  智能体 harness 的每次运行生命周期。包括 `harnessId`、可选
  `pluginId`、提供商/模型/渠道和运行 ID。完成时会添加
  `durationMs`、`outcome`、可选 `resultClassification`、`yieldDetected`
  以及 `itemLifecycle` 计数。发生错误时会添加 `phase`
  （`prepare`/`start`/`send`/`resolve`/`cleanup`）、`errorCategory` 和
  可选 `cleanupFailed`。

**Exec**

- `exec.process.completed` - 终端结果、持续时间、目标、模式、退出
  代码和失败类型。不包括命令文本和工作目录。
- `exec.approval.followup_suppressed` - 会话重新绑定后丢弃的过期审批后续操作。
  包括 `approvalId`、`reason`
  （`session_rebound`）、`phase`（`direct_delivery` 或 `gateway_preflight`）
  以及调度器时间戳。不包括会话键、路由和命令文本。

## 不使用导出器

无需运行 `diagnostics-otel`，即可让插件或自定义接收端使用诊断事件：

```json5
{
  diagnostics: { enabled: true },
}
```

若要输出有针对性的调试信息而不提高 `logging.level`，请使用诊断
标志。标志不区分大小写，并支持通配符（`telegram.*` 或
`*`）：

```json5
{
  diagnostics: { flags: ["telegram.http"] },
}
```

或者使用一次性环境变量覆盖：

```bash
OPENCLAW_DIAGNOSTICS=telegram.http,telegram.payload openclaw gateway
```

标志输出会写入标准日志文件（`logging.file`），并且仍会由
`logging.redactSensitive` 进行脱敏。完整指南：
[诊断标志](/zh-CN/diagnostics/flags)。

## 禁用

```json5
{
  diagnostics: { otel: { enabled: false } },
}
```

或者从 `plugins.allow` 中省略 `diagnostics-otel`，或运行
`openclaw plugins disable diagnostics-otel`。

## 相关内容

- [日志](/zh-CN/logging) - 文件日志、控制台输出、CLI 尾随查看以及 Control UI 的 Logs 选项卡
- [Gateway 网关日志内部机制](/zh-CN/gateway/logging) - WS 日志样式、子系统前缀和控制台捕获
- [诊断标志](/zh-CN/diagnostics/flags) - 有针对性的调试日志标志
- [诊断导出](/zh-CN/gateway/diagnostics) - 面向操作员的支持包工具（与 OTEL 导出分开）
- [配置参考](/zh-CN/gateway/configuration-reference#diagnostics) - 完整的 `diagnostics.*` 字段参考
