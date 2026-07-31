---
read_when:
    - 更改日志输出或格式
    - 调试 CLI 或 Gateway 网关输出
summary: 日志界面、文件日志、WS 日志样式和控制台格式化
title: Gateway 网关日志
x-i18n:
    generated_at: "2026-07-26T06:09:21Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: f0b11a68611032c29c31091b2411982487e7f5df3ecf4f1e3b586e7d21e543d3
    source_path: gateway/logging.md
    workflow: 16
---

# 日志

有关面向用户的概览（CLI + Control UI + 配置），请参阅 [/logging](/zh-CN/logging)。

OpenClaw 有两个日志界面：

- **控制台输出** - 你在终端 / Debug UI 中看到的内容。
- **文件日志** - Gateway 网关日志记录器写入的 JSON 行。

启动时，Gateway 网关会记录解析后的默认智能体模型，以及影响新会话的模式默认值：

```text
智能体模型：openai/gpt-5.6-sol（思考=中等，快速=开启）
```

`thinking` 来自默认智能体、模型参数或全局智能体默认值；未设置时显示 `medium`。`fast` 来自默认智能体或模型的 `fastMode` 参数。

## 基于文件的日志记录器

- 默认滚动日志文件位于 `/tmp/openclaw/` 下（每天一个文件），日期采用 Gateway 网关主机的本地时区。默认配置文件使用 `openclaw-YYYY-MM-DD.log`；命名配置文件使用 `openclaw-<profile>-YYYY-MM-DD.log`（例如 `openclaw-dev-YYYY-MM-DD.log`）。如果该目录不安全或不可写（所有者错误、任何人均可写或为符号链接），OpenClaw 会改用用户范围的 `os.tmpdir()/openclaw-<uid>` 路径；在 Windows 上始终使用该操作系统临时目录回退路径。
- 活动日志文件达到 `logging.maxFileBytes`（默认：100 MB）时轮转，最多保留五个带编号的归档（`.1` 至 `.5`），并继续写入新的活动文件。
- 通过 `~/.openclaw/openclaw.json` 配置日志文件路径和级别：`logging.file`、`logging.level`。
- 文件格式为每行一个 JSON 对象。

Talk、实时语音和托管房间代码路径使用共享文件日志记录器，记录有界的生命周期信息，用于运维调试和 OTLP 日志导出。转录文本、音频载荷、轮次 ID、通话 ID 和提供商项目 ID 绝不会复制到日志记录中。

Control UI 的 Logs 选项卡通过 Gateway 网关跟踪此文件（`logs.tail`）。CLI 也采用相同方式：

```bash
openclaw logs --follow
```

### 详细模式与日志级别

- **文件日志**完全由 `logging.level` 控制。
- `--verbose` 仅影响**控制台详细程度**（以及 WS 日志样式），**不会**提高文件日志级别。
- 若要在文件日志中捕获仅详细模式下提供的信息，请将 `logging.level` 设置为 `debug` 或 `trace`。
- 跟踪日志还包括所选热路径的诊断计时摘要，例如插件工具工厂准备过程。请参阅 [/tools/plugin#slow-plugin-tool-setup](/zh-CN/tools/plugin#slow-plugin-tool-setup)。

## 控制台捕获

CLI 捕获 `console.log/info/warn/error/debug/trace`，将其写入文件日志，同时仍输出到 stdout/stderr。

可独立调整控制台详细程度：

- `logging.consoleLevel`（默认 `info`）
- `logging.consoleStyle`（`pretty` | `compact` | `json`；在 TTY 上默认为 `pretty`，否则默认为 `compact`）

## 脱敏

OpenClaw 会在日志或转录输出离开进程前遮蔽敏感令牌。此脱敏策略适用于控制台、文件日志、OTLP 日志记录和会话转录文本输出位置，因此匹配的机密值会在 JSONL 行或消息写入磁盘前被遮蔽。

- 敏感值脱敏始终启用。
- `logging.redactPatterns`：正则表达式字符串数组（覆盖默认值）
  - 使用原始正则表达式字符串（自动 `gi`），或使用 `/pattern/flags` 指定自定义标志。
  - 匹配项会被遮蔽，但保留前 6 个和后 4 个字符（值长度 >= 18 个字符）；较短的值会变为 `***`。
  - 默认规则涵盖常见的密钥赋值、CLI 标志、JSON 字段、Bearer 请求头、PEM 块、常见供应商令牌前缀，以及支付凭据字段名称（卡号、CVC/CVV、共享支付令牌、支付凭据）。

Control UI 工具调用事件、`sessions_history` 输出、诊断导出、提供商错误、Exec 审批显示和 Gateway WebSocket 日志等安全边界始终执行脱敏。`logging.redactPatterns` 可添加特定于部署的模式。

## Gateway WebSocket 日志

Gateway 网关以两种模式输出 WebSocket 协议日志：

- **普通模式（无 `--verbose`）**：仅输出“值得关注的”RPC 结果——错误（`ok=false`）、慢调用（默认阈值：`>= 50ms`）和解析错误。
- **详细模式（`--verbose`）**：输出所有 WS 请求/响应流量。

### WS 日志样式

`openclaw gateway` 支持按 Gateway 网关切换样式：

- `--ws-log auto`（默认）：普通模式经过优化；详细模式使用紧凑输出。
- `--ws-log compact`：在详细模式下使用紧凑输出（请求/响应成对显示）。
- `--ws-log full`：在详细模式下使用完整的逐帧输出。
- `--compact`：`--ws-log compact` 的别名。

```bash
# 优化模式（仅错误/慢调用）
openclaw gateway

# 显示所有 WS 流量（成对显示）
openclaw gateway --verbose --ws-log compact

# 显示所有 WS 流量（完整元数据）
openclaw gateway --verbose --ws-log full
```

## 控制台格式（子系统日志记录）

控制台格式化程序可**感知 TTY**，并输出格式一致且带前缀的行。子系统日志记录器使输出保持分组且易于浏览：

- 每行都有**子系统前缀**（例如 `[gateway]`、`[canvas]`、`[tailscale]`）。
- **子系统颜色**（每个子系统保持稳定，根据名称哈希生成）以及级别颜色。
- 当输出为 TTY 或环境类似富功能终端（`TERM`/`COLORTERM`/`TERM_PROGRAM`）时**使用颜色**；遵循 `NO_COLOR` 和 `FORCE_COLOR`。
- **缩短的子系统前缀**：移除开头的 `gateway/`、`channels/` 或 `providers/` 段，然后最多保留剩余段中的最后 2 段（例如 `channels/turn/kernel` 显示为 `turn/kernel`）。已知渠道子系统（`telegram`、`whatsapp`、`slack` 等）始终折叠为仅显示渠道名称。
- **按子系统划分的子日志记录器**（自动添加前缀 + 结构化字段 `{ subsystem }`）。
- 用于二维码/用户体验输出的 **`logRaw()`**（无前缀、无格式）。
- **控制台样式**：`pretty` | `compact` | `json`。
- **控制台日志级别**与文件日志级别相互独立（当 `logging.level` 为 `debug`/`trace` 时，文件仍保留完整详细信息）。
- **WhatsApp 消息正文**以 `debug` 级别记录（使用 `--verbose` 查看）。

这样既能保持文件日志稳定，又能使交互式输出易于浏览。

## 相关内容

- [日志](/zh-CN/logging)
- [OpenTelemetry 导出](/zh-CN/gateway/opentelemetry)
- [诊断导出](/zh-CN/gateway/diagnostics)
