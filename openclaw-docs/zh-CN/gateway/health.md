---
read_when:
    - 诊断渠道连接或 Gateway 健康状况
    - 了解健康检查 CLI 命令和选项
summary: 健康检查命令和 Gateway 健康监控
title: 健康检查
x-i18n:
    generated_at: "2026-07-26T06:15:34Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 59a7fbfb7fb86be7dbd3a03f96c7328c2bc8cc851230c0bdd1b1b750b3014be4
    source_path: gateway/health.md
    workflow: 16
---

无需猜测即可验证渠道连接性的简短指南。

## 快速检查

- `openclaw status` - 本地摘要：Gateway 网关可达性/模式、更新提示、已关联渠道的身份验证时长、会话及近期活动。
- `openclaw status --all` - 完整本地诊断（只读、彩色输出，可安全粘贴用于调试）。
- `openclaw status --deep` - 请求正在运行的 Gateway 网关执行实时探测（`health` 与 `probe:true`），并在支持时执行各账户的渠道探测。
- `openclaw status --usage` - 显示模型提供商的用量/配额快照。
- `openclaw health` - 请求正在运行的 Gateway 网关提供其健康快照（仅限 WS；CLI 不会直接建立渠道套接字）。
- `openclaw health --verbose`（别名 `--debug`）- 强制执行实时健康探测并输出 Gateway 网关连接详情。
- `openclaw health --json` - 输出机器可读的健康快照。
- 在任意渠道中将 `/status` 作为独立聊天命令发送，无需调用智能体即可获得状态回复。
- 日志：运行 `openclaw logs --follow`（或 `openclaw --profile <profile> logs --follow`），并筛选 `web-heartbeat`、`web-reconnect`、`web-auto-reply`、`web-inbound`。

对于 Discord 和其他聊天提供商，会话行并不代表套接字存活状态。
`openclaw sessions`、Gateway 网关 `sessions.list` 和智能体的 `sessions_list` 工具
读取已存储的对话状态。提供商可能已重新连接并显示渠道状态正常，
而此时尚未生成任何新会话行。请使用上述渠道状态和
健康命令进行实时连接检查。

## 深度诊断

- 磁盘上的凭据：`ls -l ~/.openclaw/credentials/whatsapp/<accountId>/creds.json`（mtime 应为近期时间）。
- 会话存储：`ls -l ~/.openclaw/agents/<agentId>/agent/openclaw-agent.sqlite`。`status` 会显示数量和近期收件人。
- 重新关联流程：当日志中出现状态码 409-515 或 `loggedOut` 时，运行 `openclaw channels logout && openclaw channels login --verbose`。配对后，如果状态码为 515，二维码登录流程会自动重启一次。
- 诊断默认启用（`diagnostics.enabled: false` 可将其禁用）。内存事件记录 RSS/堆字节数以及阈值/增长压力。进程仍在运行但已饱和时，存活警告会记录事件循环延迟/利用率、CPU 核心比率，以及活动/等待/排队会话数。超大负载事件会记录被拒绝/截断/分块的内容类型及其大小和限制，但绝不会记录消息文本、附件内容、Webhook 正文、原始请求/响应正文、令牌、Cookie 或密钥值。
- 同一 Heartbeat 还会驱动有界稳定性记录器：`openclaw gateway stability`（或 `diagnostics.stability` Gateway RPC）。Gateway 网关致命退出、关闭超时和重启启动失败会将最新快照持久化到 `~/.openclaw/logs/stability/` 下。使用 `openclaw gateway stability --bundle latest` 检查最新的软件包。
- 提交错误报告时，请运行 `openclaw gateway diagnostics export` 并附上生成的 zip：其中包含 Markdown 摘要、最新稳定性软件包、经过清理的日志元数据、经过清理的 Gateway 网关状态/健康快照，以及配置结构。聊天文本、Webhook 正文、工具输出、凭据、Cookie、账户/消息标识符和密钥值会被省略或编辑。请参阅[诊断导出](/zh-CN/gateway/diagnostics)。

## 健康监控配置

- `channels.<provider>.healthMonitor.enabled`：在保持全局监控启用的同时，禁用特定渠道的健康监控重启。
- `channels.<provider>.accounts.<accountId>.healthMonitor.enabled`：优先于渠道级设置的多账户覆盖配置。
- 这些按渠道覆盖的配置目前适用于公开此功能的内置渠道：Discord、Google Chat、iMessage、IRC、Microsoft Teams、Signal、Slack、Telegram 和 WhatsApp。

## 运行时间监控

外部运行时间监控服务应使用专用的 `/health` 端点，而不是 `/v1/chat/completions`。

- **应使用：**`GET /health` - 立即响应、不创建会话、不调用 LLM，并返回 `{"ok":true,"status":"live"}`
- **不要使用：**不要使用 `/v1/chat/completions` 进行健康检查 - 每个请求都会创建完整的智能体会话，包括 Skills 快照、上下文组装和 LLM 调用

如果未提供 `x-openclaw-session-key` 标头或 `user` 字段，`/v1/chat/completions` 会为每个请求生成新的随机会话。每 15 分钟发起一次探测的监控服务每天会创建约 96 个会话，每个会话占用 4-22KB。长期如此会导致会话存储膨胀，并可能造成上下文窗口溢出。

### 监控服务设置示例

- **BetterStack：**将健康检查 URL 设置为 `https://<your-gateway-host>:<port>/health`
- **UptimeRobot：**添加一个新的 HTTP 监控器，URL 为 `https://<your-gateway-host>:<port>/health`
- **通用：**当 Gateway 网关健康时，对 `/health` 发起任意 HTTP GET 请求都会返回 200 和 `{"ok":true}`

## 出现故障时

- `logged out` 或状态码 409-515 -> 先运行 `openclaw channels logout`，再运行 `openclaw channels login` 以重新关联。
- Gateway 网关不可达 -> 启动它：`openclaw gateway --port 18789`（如果端口被占用，请使用 `--force`）。
- 没有入站消息 -> 确认已关联的手机在线且允许该发送者（`channels.whatsapp.allowFrom`）；对于群聊，请确保允许列表和提及规则相匹配（`channels.whatsapp.groups`、`agents.entries.*.groupChat.mentionPatterns`）。

## 专用“health”命令

`openclaw health` 请求正在运行的 Gateway 网关提供其健康快照（CLI 不会直接建立渠道
套接字）。默认情况下，它返回最新缓存的 Gateway 网关快照，并由
Gateway 网关在后台刷新该缓存；`--verbose` 则会强制执行实时探测。
该命令会报告已关联凭据/身份验证时长（如可用）、各渠道探测摘要、
会话存储摘要和探测耗时。如果 Gateway 网关不可达或探测失败/超时，
命令将以非零状态退出。

选项：

- `--json`：机器可读的 JSON 输出
- `--timeout <ms>`：覆盖默认的 10 秒探测超时
- `--verbose`：强制执行实时探测并输出 Gateway 网关连接详情
- `--debug`：`--verbose` 的别名

健康快照包括：`ok`（布尔值）、`ts`（时间戳）、`durationMs`（探测时间）、各渠道状态、智能体可用性和会话存储摘要。

## 相关内容

- [Gateway 网关运行手册](/zh-CN/gateway)
- [诊断导出](/zh-CN/gateway/diagnostics)
- [Gateway 网关故障排除](/zh-CN/gateway/troubleshooting)
