---
read_when:
    - 你想通过脚本运行一个智能体轮次（可选择发送回复）
summary: '`openclaw agent` 的 CLI 参考（通过 Gateway 网关发送一个智能体轮次）'
title: 智能体
x-i18n:
    generated_at: "2026-07-26T06:43:13Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 1a4c139a3b235d6a56ba63063737b80f93448c2dbb7a92c6d0756fb19a9f95e4
    source_path: cli/agent.md
    workflow: 16
---

# `openclaw agent`

通过 Gateway 网关运行一个智能体轮次。显式的 `--local` 标志是唯一的嵌入式执行路径。

至少传入一个会话选择器：`--to`、`--session-key`、`--session-id` 或 `--agent`。

相关内容：[智能体发送工具](/zh-CN/tools/agent-send)

## 选项

- `-m, --message <text>`：消息正文
- `--message-file <path>`：从 UTF-8 文件读取消息正文
- `-t, --to <dest>`：用于派生会话键的接收者
- `--session-key <key>`：用于路由的显式会话键
- `--session-id <id>`：显式会话 ID
- `--agent <id>`：智能体 ID；覆盖路由绑定
- `--model <id>`：本次运行的模型覆盖值（`provider/model` 或模型 ID）
- `--thinking <level>`：智能体思考级别（`off`、`minimal`、`low`、`medium`、`high`，以及提供商支持的自定义级别，例如 `xhigh`、`adaptive` 或 `max`）
- `--verbose <on|off>`：为会话持久保存详细输出级别
- `--channel <channel>`：投递渠道；省略时使用主会话渠道
- `--reply-to <target>`：投递目标覆盖值
- `--reply-channel <channel>`：投递渠道覆盖值
- `--reply-account <id>`：投递账户覆盖值
- `--local`：直接运行嵌入式智能体（预加载插件注册表后）
- `--deliver`：将回复发回所选渠道/目标
- `--timeout <seconds>`：覆盖此命令的智能体轮次截止时间（默认 600，或 `agents.defaults.timeoutSeconds`）；`0` 会禁用总截止时间。600 秒的回退值属于此 CLI 命令，而非普通 Gateway 网关轮次；后者的默认值为 48 小时。
- `--json`：输出 JSON

## 示例

```bash
openclaw agent --to +15555550123 --message "状态更新" --deliver
openclaw agent --agent ops --message "汇总日志"
openclaw agent --agent ops --message-file ./task.md
openclaw agent --agent ops --model openai/gpt-5.4 --message "汇总日志"
openclaw agent --session-key agent:ops:incident-42 --message "汇总状态"
openclaw agent --agent ops --session-key incident-42 --message "汇总状态"
openclaw agent --session-id 1234 --message "汇总收件箱" --thinking medium
openclaw agent --to +15555550123 --message "跟踪日志" --verbose on --json
openclaw agent --agent ops --message "生成报告" --deliver --reply-channel slack --reply-to "#reports"
openclaw agent --agent ops --message "在本地运行" --local
```

## 注意事项

- 必须且只能传入 `--message` 或 `--message-file` 中的一个。`--message-file` 会移除开头的 UTF-8 BOM 并保留多行内容；它会拒绝非有效 UTF-8 的文件。大于 4 MiB 的文件会在分派前被拒绝。
- 斜杠命令（例如 `/compact`）无法通过 `--message` 运行。CLI 会拒绝这些命令，并引导你改用对应的一等命令（压缩操作使用 `openclaw sessions compact <key>`）。
- `--local` 运行是一次性的：为该运行打开的内置 MCP loopback 资源和预热的 Claude stdio 会话会在回复后停用，因此脚本调用不会留下仍在运行的本地子进程。由 Gateway 网关支持的运行则会把 Gateway 网关拥有的 MCP loopback 资源保留在运行中的 Gateway 网关进程下。
- 当重启恢复仍待处理时，使用 `--local` 的独立嵌入式执行会拒绝复用现有主会话。请通过健康的 Gateway 网关运行该轮次，或在那里使用 `/new` 或 `/reset` 重置；独立的嵌入式进程无法与 Gateway 网关扫描器安全协调该恢复所有者。
- 同时使用 `--agent`、`--channel` 和 `--to` 时，会话路由遵循该渠道的规范接收者和 `session.dmScope`。具有稳定的仅出站接收者身份的渠道会使用提供商拥有的会话，该会话与智能体的主会话隔离。`--reply-channel` 和 `--reply-account` 仅影响投递。
- `--session-key` 用于选择显式会话键。带智能体前缀的键必须使用 `agent:<agent-id>:<session-key>`；同时提供两者时，`--agent` 必须与键中的智能体 ID 匹配。未带前缀且非哨兵值的键在提供 `--agent` 时归属于该智能体，否则归属于已配置的默认智能体；例如，`--agent ops --session-key incident-42` 会路由到 `agent:ops:incident-42`。仅当未提供 `--agent` 时，字面键 `global` 和 `unknown` 才保持无作用域状态。
- `--json` 会将 stdout 专用于 JSON 响应；Gateway 网关、插件和 `--local` 的诊断信息会写入 stderr，使脚本能够直接解析 stdout。
- 瞬时握手重试耗尽后，Gateway 网关超时或连接关闭会导致命令失败；CLI 绝不会悄然以嵌入式方式重新运行该轮次。传输中断的结果具有不确定性——Gateway 网关可能已接受该轮次并且仍会完成它——因此 stderr 提示会要求先检查 `openclaw gateway status` 和会话记录，再重试或使用 `--local` 重新运行，以免该轮次执行两次。
- `SIGTERM`/`SIGINT` 会中断正在等待的 Gateway 网关支持请求；如果 Gateway 网关已经接受该运行，CLI 还会在退出前针对该运行 ID 发送 `chat.abort`。`--local` 运行会收到相同信号，但不会发送 `chat.abort`。启动器子进程如果因首次转发的 `SIGINT` 或 `SIGTERM` 而终止，将分别以状态码 130 或 143 退出。如果内部运行去重键已存在此会话的活动运行，响应会报告 `status: "in_flight"`，非 JSON CLI 则会向 stderr 输出诊断信息，而不是输出空回复。对于外部 cron/systemd 包装器，请保留类似 `timeout -k 60 600 openclaw agent ...` 的强制终止后备机制，以便在关闭过程无法排空时由监督程序回收该进程。
- 当此命令触发 `models.json` 重新生成时，由 SecretRef 管理的提供商凭据会以非机密标记形式持久保存（例如环境变量名称、`secretref-env:ENV_VAR_NAME` 或 `secretref-managed`），绝不会保存解析后的机密明文。标记写入内容来自活动源配置快照，而非解析后的运行时机密值。

## JSON 投递状态

使用 `--json --deliver` 时，CLI JSON 响应包含顶层 `deliveryStatus`，使脚本能够区分已投递、已抑制、部分投递和投递失败：

```json
{
  "payloads": [{ "text": "报告已就绪", "mediaUrl": null }],
  "meta": { "durationMs": 1200 },
  "deliveryStatus": {
    "requested": true,
    "attempted": true,
    "status": "sent",
    "succeeded": true,
    "resultCount": 1
  }
}
```

由 Gateway 网关支持的 CLI 响应还会在 `result.deliveryStatus` 中保留原始 Gateway 网关结果结构。

`deliveryStatus.status` 是以下值之一：

| 状态           | 含义                                                                                                                                    |
| ---------------- | ------------------------------------------------------------------------------------------------------------------------------------------ |
| `sent`           | 投递已完成。                                                                                                                        |
| `suppressed`     | 有意不发送投递（例如，消息发送钩子取消了投递，或没有可见结果）。终止状态，不重试。 |
| `partial_failed` | 至少一个载荷已发送，之后的载荷发送失败。                                                                                   |
| `failed`         | 没有完成持久投递，或投递预检失败。                                                                                   |

通用字段：

- `requested`：存在此对象时始终为 `true`。
- `attempted`：执行持久投递路径后为 `true`；预检失败或没有可见载荷时为 `false`。
- `succeeded`：`true`、`false` 或 `"partial"`；`"partial"` 与 `status: "partial_failed"` 配对。
- `reason`：来自持久投递或预检验证的小写蛇形命名原因。已知值包括 `cancelled_by_message_sending_hook`、`no_visible_payload`、`no_visible_result`、`channel_resolved_to_internal`、`unknown_channel`、`invalid_delivery_target` 和 `no_delivery_target`；持久投递失败时还可能报告失败阶段。由于该集合可能扩展，请将未知值视为不透明值。
- `resultCount`：渠道发送结果的数量（如果可用）。
- `sentBeforeError`：部分失败时，如果出错前至少发送了一个载荷，则为 `true`。
- `error`：投递失败或部分失败时为 `true`。
- `errorMessage`：仅当捕获到基础投递错误消息时存在。预检失败包含 `error`/`reason`，但不包含 `errorMessage`。
- `payloadOutcomes`：可选的逐载荷结果，在可用时包含 `index`、`status`、`reason`、`resultCount`、`error`、`stage`、`sentBeforeError` 或钩子元数据。

## 相关内容

- [CLI 参考](/zh-CN/cli)
- [智能体运行时](/zh-CN/concepts/agent)
