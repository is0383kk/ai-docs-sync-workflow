---
read_when:
    - 你需要一份面向初学者的 OpenClaw 日志概览
    - 你想配置日志级别、格式或脱敏处理
    - 你正在进行故障排查，需要快速找到日志
summary: 文件日志、控制台输出、CLI 尾随查看和 Control UI 的 Logs 选项卡
title: 日志
x-i18n:
    generated_at: "2026-07-26T06:18:25Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 2c9303c4bc9c0797ca9c5775a281dce95229661b61d710425b2f7bec182b2e75
    source_path: logging.md
    workflow: 16
---

OpenClaw 有两个主要的日志界面：

- **文件日志**（JSON 行），由 Gateway 网关写入。
- 运行 Gateway 网关的终端中的**控制台输出**。

Control UI 的**日志**选项卡会持续读取 Gateway 网关文件日志。本页说明日志的
存储位置、读取方法，以及如何配置日志级别和格式。

## 日志的存储位置

默认情况下，Gateway 网关每天写入一个滚动日志文件。默认配置文件
保留历史路径：

`/tmp/openclaw/openclaw-YYYY-MM-DD.log`

命名配置文件在同一目录中使用包含配置文件限定信息的文件名：

`/tmp/openclaw/openclaw-<profile>-YYYY-MM-DD.log`

文件名中的配置文件段使用小写，并且仅限字母、数字和
短横线。简单的小写名称保持可读，因此 `--dev` 简写会写入
`openclaw-dev-YYYY-MM-DD.log`。大小写、下划线和字面短横线使用
可逆的短横线转义，因此不同的配置文件名称绝不会共用一个日志文件。
直接通过环境设置的超长值会使用长度受限的哈希后缀，
以保持在文件系统的文件名长度限制内。显式设置的 `logging.file` 会覆盖
这些默认值。

日期使用 Gateway 网关主机的本地时区。当 `/tmp/openclaw` 不安全
或不可用时（在 Windows 上始终如此），OpenClaw 会改用操作系统临时目录下
用户范围的 `openclaw-<uid>` 目录。带日期的日志文件会在
24 小时后清理。

当下一次写入会导致文件超过 `logging.maxFileBytes`
（默认值：100 MB）时，每个文件都会轮转。OpenClaw 会在活动文件旁最多保留
五个带编号的归档，例如 `openclaw-YYYY-MM-DD.1.log` 或
`openclaw-dev-YYYY-MM-DD.1.log`，并继续写入新的活动日志，而不是
停止记录诊断信息。

可以在 `~/.openclaw/openclaw.json` 中覆盖路径：

```json
{
  "logging": {
    "file": "/path/to/openclaw.log"
  }
}
```

## 如何读取日志

### CLI：实时跟踪（推荐）

通过 RPC 跟踪 Gateway 网关日志文件：

```bash
openclaw logs --follow
openclaw --dev logs --follow
openclaw --profile work logs --follow
```

根配置文件选择器会解析为 Gateway 网关使用的同一个配置文件专属文件，
包括本地 RPC 不可用时的 CLI 回退读取。

选项：

| 标志                | 默认值  | 行为                                                                              |
| ------------------- | -------- | ------------------------------------------------------------------------------------- |
| `--follow`          | 关闭      | 持续跟踪；断开连接时采用退避策略重新连接                                   |
| `--limit <n>`       | `200`    | 每次获取的最大行数                                                                   |
| `--max-bytes <n>`   | `250000` | 每次获取最多读取的字节数                                                           |
| `--interval <ms>`   | `1000`   | 跟踪时的轮询间隔                                                         |
| `--json`            | 关闭      | 以行分隔的 JSON（每行一个事件）                                              |
| `--plain`           | 关闭      | 在 TTY 会话中强制使用纯文本                                                      |
| `--no-color`        | —        | 禁用 ANSI 颜色                                                                   |
| `--utc`             | 关闭      | 使用 UTC 呈现时间戳（默认为本地时间）                                      |
| `--local-time`      | 关闭      | 本地时间默认值所接受的兼容拼写；除此之外无其他作用       |
| `--url` / `--token` | —        | 标准 Gateway 网关 RPC 标志                                                            |
| `--timeout <ms>`    | `30000`  | Gateway 网关 RPC 超时时间                                                                   |
| `--expect-final`    | 关闭      | 由智能体支持的 RPC 最终响应等待标志（此处通过共享客户端层接受） |

输出模式：

- **TTY 会话**：美化、彩色的结构化日志行。
- **非 TTY 会话**：纯文本。

传递显式的 `--url` 时，CLI 不会自动应用配置或
环境凭据；需要自行包含 `--token`，否则调用会失败并显示
`gateway url override requires explicit credentials`。

在 JSON 模式下，CLI 会发出带 `type` 标签的对象：

- `meta`：流元数据（文件、来源、来源类型、服务、游标、大小）
- `log`：解析后的日志条目
- `notice`：截断/轮转提示
- `raw`：未解析的日志行
- `error`：Gateway 网关连接失败（写入 stderr）

如果隐式 local loopback Gateway 网关要求配对、在连接期间关闭，
或在 `logs.tail` 响应前超时，`openclaw logs` 会自动回退到
已配置的 Gateway 网关文件日志。显式的 `--url` 目标不会使用
此回退。`openclaw logs --follow` 更为严格：在 Linux 上，如果可用，它会按 PID 使用活动的
用户 systemd Gateway 网关日志；否则会采用退避策略重试
实时 Gateway 网关，而不是跟踪可能过时的并列
文件。

如果无法访问 Gateway 网关，CLI 会打印一条简短提示，建议运行：

```bash
openclaw doctor
```

### Control UI（Web）

Control UI 的**日志**选项卡使用 `logs.tail` 跟踪同一个文件。
有关如何打开它的信息，请参阅 [Control UI](/zh-CN/web/control-ui)。

### 仅渠道日志

要筛选渠道活动（WhatsApp/Telegram 等），请使用：

```bash
openclaw channels logs --channel whatsapp
```

`--channel` 默认为 `all`；还可以使用 `--lines <n>`（默认值为 200）和 `--json`。

## 日志格式

### 文件日志（JSONL）

日志文件中的每一行都是一个 JSON 对象。CLI 和 Control UI 会解析这些
条目，以呈现结构化输出（时间、级别、子系统、消息）。

文件日志 JSONL 记录还会在可用时包含可供机器筛选的顶层字段：

- `hostname`：Gateway 网关主机名。
- `message`：用于全文搜索的扁平化日志消息文本。
- `agent_id`：日志调用携带智能体上下文时的活动智能体 ID。
- `session_id`：日志调用携带会话上下文时的活动会话 ID/键。
- `channel`：日志调用携带渠道上下文时的活动渠道。

OpenClaw 会在这些字段旁保留原始的结构化日志参数，
因此读取带编号 tslog 参数键的现有解析器仍可继续工作。

Talk、实时语音和托管房间活动会通过同一文件日志管道发出长度受限的生命周期日志
记录。这些记录在可用时包含事件类型、
模式、传输方式、提供商以及大小/计时测量值，但不包含
转录文本、音频载荷、轮次 ID、通话 ID 和提供商项目 ID。

### 控制台输出

控制台日志可感知 **TTY**，并经过格式化以提高可读性：

- 子系统前缀（例如 `gateway/channels/whatsapp`）
- 级别着色（info/warn/error）
- 可选的紧凑模式或 JSON 模式

控制台格式由 `logging.consoleStyle` 控制。

### Gateway 网关 WebSocket 日志

`openclaw gateway` 还为 RPC 流量提供 WebSocket 协议日志：

- 普通模式：仅记录值得关注的结果（错误、解析错误、慢调用）
- `--verbose`：所有请求/响应流量
- `--ws-log auto|compact|full`：选择详细呈现样式
- `--compact`：`--ws-log compact` 的别名

示例：

```bash
openclaw gateway
openclaw gateway --verbose --ws-log compact
openclaw gateway --verbose --ws-log full
```

## 配置日志

所有日志配置都位于 `~/.openclaw/openclaw.json` 的 `logging` 下。

```json
{
  "logging": {
    "level": "info",
    "file": "/path/to/openclaw.log",
    "consoleLevel": "info",
    "consoleStyle": "pretty",
    "redactSensitive": "tools",
    "redactPatterns": ["sk-.*"]
  }
}
```

### 日志级别

级别：`silent`、`fatal`、`error`、`warn`、`info`、`debug`、`trace`。

- `logging.level`：**文件日志**（JSONL）级别（默认值：`info`）。
- `logging.consoleLevel`：**控制台**详细程度级别。

可以通过 **`OPENCLAW_LOG_LEVEL`** 环境变量覆盖两者（例如 `OPENCLAW_LOG_LEVEL=debug`）。环境变量的优先级高于配置文件，因此无需编辑 `openclaw.json`，即可为单次运行提高详细程度。也可以传递全局 CLI 选项 **`--log-level <level>`**（例如 `openclaw --log-level debug gateway run`），它会为该命令覆盖环境变量。

`--verbose` 仅影响控制台输出和 WS 日志的详细程度；它不会更改
文件日志级别。

### 定向模型传输诊断

调试提供商调用时，应使用定向环境标志，而不是将
所有日志提高到 `debug`：

```bash
OPENCLAW_DEBUG_MODEL_TRANSPORT=1 openclaw gateway
OPENCLAW_DEBUG_MODEL_PAYLOAD=tools OPENCLAW_DEBUG_SSE=events openclaw gateway
```

可用标志：

- `OPENCLAW_DEBUG_MODEL_TRANSPORT=1`：以 `info` 级别发出请求开始、fetch 响应、SDK
  标头、第一个流式事件、流完成和传输错误。
- `OPENCLAW_DEBUG_MODEL_PAYLOAD=summary`：在模型请求日志中包含长度受限的请求载荷
  摘要。
- `OPENCLAW_DEBUG_MODEL_PAYLOAD=tools`：在载荷摘要中包含所有面向模型的工具名称。
- `OPENCLAW_DEBUG_MODEL_PAYLOAD=full-redacted`：包含经过脱敏且大小受限的 JSON
  载荷快照。仅在调试期间使用；机密信息会被脱敏，但提示词
  和消息文本可能仍然存在。
- `OPENCLAW_DEBUG_SSE=events`：发出首个事件和流完成的计时信息。
- `OPENCLAW_DEBUG_SSE=peek`：还发出前五个经过脱敏的 SSE 事件
  载荷，每个事件的大小均受限制。
- `OPENCLAW_DEBUG_CODE_MODE=1`：发出代码模式的模型界面诊断信息，
  包括因代码模式拥有工具界面而隐藏原生提供商工具的情况。

这些标志通过 OpenClaw 的常规日志机制记录，因此 `openclaw logs --follow`
和 Control UI 的“日志”选项卡都会显示它们。不使用这些标志时，同样的诊断信息
仍可在 `debug` 级别获取。

无论 `OPENCLAW_DEBUG_MODEL_TRANSPORT` 如何，`[model-fetch]` 的开始和响应元数据（提供商、API、模型、状态、
延迟以及方法、URL、超时、代理和策略等请求字段）
始终以 `info` 级别发出，因此无需调试标志
也能看到基本的模型传输健康状况。

### 跟踪关联

文件日志采用 JSONL 格式。当日志调用携带有效的诊断跟踪上下文时，
OpenClaw 会将跟踪字段写为顶层 JSON 键（`traceId`、`spanId`、
`parentSpanId`、`traceFlags`），以便外部日志处理器可以将该日志行
与 OTEL span 和提供商的 `traceparent` 传播关联起来。

Gateway 网关 HTTP 请求和 Gateway 网关 WebSocket 帧会建立内部请求
跟踪作用域。在该异步作用域内发出的日志和诊断事件如果未传递显式跟踪上下文，
则会继承请求跟踪。智能体运行和模型调用跟踪会成为活动请求跟踪的子级，因此本地日志、
诊断快照、OTEL span 和受信任的提供商 `traceparent` 标头可以
通过 `traceId` 关联，而无需记录原始请求或模型内容。

启用 OpenTelemetry 日志导出时，Talk 生命周期日志记录也会流向 diagnostics-otel 日志导出，
并使用与文件日志相同的长度受限属性。
配置 `diagnostics.otel.logsExporter` 以选择 OTLP、标准输出 JSONL 或
同时使用两种接收端。

### 模型调用大小和计时

模型调用诊断会记录长度受限的请求/响应测量数据，而不会
捕获原始提示词或响应内容：

- `requestPayloadBytes`：最终模型请求负载的 UTF-8 字节大小
- `responseStreamBytes`：流式模型响应分块负载的 UTF-8 字节大小。高频文本、思考和工具调用增量事件仅计算增量 `delta` 字节，而不是完整的 `partial` 快照。
- `timeToFirstByteMs`：首个流式响应事件前的已用时间
- `durationMs`：模型调用总持续时间

启用诊断导出后，这些字段可用于诊断快照、模型调用插件钩子以及
OTEL 模型调用 span/指标。

### 控制台样式

`logging.consoleStyle`：

- `pretty`：便于阅读、带颜色并包含时间戳。
- `compact`：输出更紧凑（最适合长会话）。
- `json`：每行一个 JSON（供日志处理器使用）。

### 脱敏

OpenClaw 可以在敏感令牌进入控制台输出、文件日志、OTLP 日志记录、
持久化会话转录文本或 Control UI 工具事件负载（工具启动参数、
部分/最终结果负载、派生的 Exec 输出和补丁摘要）之前对其进行脱敏：

- 敏感值脱敏始终启用。
- `logging.redactPatterns`：正则表达式字符串列表，用于替换日志/转录输出的默认集合。对于 Control UI 工具负载，自定义模式会叠加应用于内置默认模式之上，因此添加模式绝不会削弱对默认模式已经捕获到的值的脱敏。

文件日志和会话转录仍采用 JSONL 格式，但匹配的机密值会在行或消息写入磁盘前被遮盖。脱敏采用尽力而为的方式：它适用于包含文本的消息内容和日志字符串，而不适用于所有标识符或二进制负载字段。

内置默认模式涵盖常见 API 凭据，以及卡号、CVC/CVV、共享支付令牌和支付凭据等支付凭据字段名称，无论它们以 JSON 字段、URL 参数、CLI 标志还是赋值的形式出现。

OpenClaw 还会对向 UI 客户端、支持包、诊断观察器、审批提示或智能体工具显示的安全边界负载进行脱敏。自定义
`logging.redactPatterns` 可以为这些界面添加项目特定的模式。

## 诊断和 OpenTelemetry

诊断是用于模型运行和消息流遥测（Webhooks、排队、会话状态）的结构化机器可读事件。它们**不会**取代日志，而是为指标、跟踪和导出器提供数据。默认情况下，事件在进程内发出（将 `diagnostics.enabled: false` 设置为关闭即可禁用）；导出这些事件是另一项独立操作。

两个相邻界面：

- **OpenTelemetry 导出** — 通过 OTLP/HTTP 将指标、跟踪和日志发送到任何兼容 OpenTelemetry 的收集器或后端（Datadog、Grafana、Honeycomb、New Relic、Tempo 等）。完整配置、信号目录、指标/span 名称、环境变量和隐私模型位于专门页面：
  [OpenTelemetry 导出](/zh-CN/gateway/opentelemetry)。
- **诊断标志** — 定向调试日志标志，可将额外日志路由到
  `logging.file`，而无需提高 `logging.level`。标志不区分大小写并支持通配符（`telegram.*`、`*`）。可在 `diagnostics.flags` 下配置，或通过 `OPENCLAW_DIAGNOSTICS=...` 环境变量覆盖。完整指南：
  [诊断标志](/zh-CN/diagnostics/flags)。

有关将 OTLP 导出到收集器的信息，请参阅 [OpenTelemetry 导出](/zh-CN/gateway/opentelemetry)。

## 故障排查提示

- **无法访问 Gateway 网关？** 请先运行 `openclaw doctor`。
- **日志为空？** 检查 Gateway 网关是否正在运行，并写入
  `logging.file` 中的文件路径。
- **需要更多详细信息？** 将 `logging.level` 设置为 `debug` 或 `trace`，然后重试。

## 相关内容

- [OpenTelemetry 导出](/zh-CN/gateway/opentelemetry) — OTLP/HTTP 导出、指标/span 目录、隐私模型
- [诊断标志](/zh-CN/diagnostics/flags) — 定向调试日志标志
- [Gateway 网关日志内部机制](/zh-CN/gateway/logging) — WS 日志样式、子系统前缀和控制台捕获
- [配置参考](/zh-CN/gateway/configuration-reference#diagnostics) — 完整的 `diagnostics.*` 字段参考
