---
read_when:
    - 你需要回答是谁运行了智能体或工具、何时运行，以及运行结果如何
    - 你需要不含内容的入站或出站消息生命周期元数据
    - 你需要一个有范围限制且可安全脱敏的活动导出功能
summary: 仅包含元数据的运行、工具和消息生命周期审计记录 CLI 参考
title: 审计记录
x-i18n:
    generated_at: "2026-07-26T06:10:21Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: da9df6f388b0a24c3b79d755fa59d047cce99262bc6d9c890be7a83da75693a8
    source_path: cli/audit.md
    workflow: 16
---

# `openclaw audit`

查询 Gateway 网关中仅含元数据的审计账本，以获取智能体运行、工具操作和选择启用的消息生命周期记录。

运行和工具事件的账本默认启用。设置
[`audit.enabled: false`](/zh-CN/gateway/configuration-reference#audit) 并重启
Gateway 网关，即可停止记录所有新事件。消息记录默认单独禁用；将 `audit.messages` 设置为 `direct` 或 `all`，并重启 Gateway 网关即可
记录消息。现有记录在过期前仍可查询（30 天）。

该账本与对话记录相互独立：它记录身份、顺序、来源、操作、状态和规范化结果代码，但绝不
存储内容；消息标识符仅以安装实例本地的
带密钥假名形式出现。[审计历史](/zh-CN/gateway/audit)定义完整的数据模型、
隐私语义、存储/保留期限和覆盖范围限制；本页介绍命令界面。

```bash
openclaw audit
openclaw audit --agent main --status failed
openclaw audit --session "agent:main:main" --after 2026-07-01T00:00:00Z
openclaw audit --run 8c69f72e-8b11-4c54-98d5-1a3dd67450c3
openclaw audit --kind tool_action --limit 50 --json
openclaw audit --kind message --direction outbound --channel telegram --json
```

## 筛选器

- `--agent <id>`：精确的智能体 ID
- `--session <key>`：精确的会话键
- `--run <id>`：精确的运行 ID
- `--kind <kind>`：`agent_run`、`tool_action` 或 `message`
- `--status <status>`：`started`、`succeeded`、`failed`、`cancelled`、
  `timed_out`、`blocked` 或 `unknown`
- `--direction <direction>`：消息方向，`inbound` 或 `outbound`
- `--channel <channel>`：精确的消息渠道
- `--after <timestamp>` / `--before <timestamp>`：包含边界值的 ISO 时间戳或
  Unix 毫秒时间戳
- `--limit <count>`：页面大小，范围为 1 到 500；默认为 `100`
- `--cursor <sequence>`：继续之前按从新到旧顺序执行的查询
- `--json`：以 JSON 格式输出有界页面

CLI 查询带版本的活动 RPC，因此一个命令即可显示完整的
已配置账本。文本输出显示时间、类型、方向、渠道、状态、
智能体、运行和操作。缺失的消息来源信息显示为 `-`；OpenClaw
不会虚构智能体或运行 ID。工具操作还会显示工具名称。JSON
输出会在存在下一页时包含 `nextCursor`。将该值传递给
`--cursor` 即可继续查询，同时不会对分页期间新到达的记录重新排序。

即使不包含消息正文和原始消息身份字段，这些导出内容仍属于敏感的运维元数据。智能体、会话和运行 ID、时间、
渠道、结果以及稳定的 HMAC 引用均可用于关联活动。应使用与其他操作员记录相同的访问控制和保留措施
对其进行保护。

## 记录的事件

Gateway 网关将可信的生命周期流投射为六种操作：

- `agent.run.started`
- `agent.run.finished`
- `tool.action.started`
- `tool.action.finished`
- `message.inbound.processed`
- `message.outbound.finished`

每条返回的记录都有稳定的事件 ID、单调递增的账本
序号、生命周期时间戳、执行者、操作、状态、
`schemaVersion: 1` 标记、来源序号以及 `redaction: "metadata_only"`。
仅当可信来源提供相关信息时，才会包含智能体/会话/运行来源信息和事件特有字段。消息记录会有意省略
`sessionKey` 和 `sessionId`，因此 `--session` 筛选器仅适用于运行和工具记录。

终止的运行和工具记录使用闭合状态和错误代码区分成功、失败、取消、
超时以及策略阻止。当上游运行时未公开
权威的终止结果时，`unknown` 是明确的非成功结果。工具调用 ID 仅以稳定的
指纹形式导出。工具名称必须符合面向模型的紧凑名称
约定；其他值将变为 `unknown`。

消息记录还会添加方向、渠道、对话类型、结果，以及可选的交付类型、失败阶段、持续时间、结果数量、规范化
原因代码和带密钥的账户/对话/消息/目标假名。当前入站边界涵盖到达核心分派的已接受消息，
包括核心重复处理结果和终止处理结果。对于到达
共享持久交付层的每个原始逻辑回复载荷，出站
边界会写入一行终止记录；分块和适配器扇出会聚合到
`resultCount` 中。对于进入队列且可重试或结果不明确的发送操作，仅在
确认、死信或协调过程使结果进入终止状态后才会记录。
目前尚未覆盖绕过这些共享边界的插件本地路径和直接发送路径；缺少记录并不能证明消息从未存在。

审计账本不能替代对话记录、任务历史、定时任务运行历史或
日志。它提供一个小型的跨运行索引，用于回答操作员问题，而无需
将对话内容复制到另一个存储中。

对于入站记录，`durationMs` 衡量核心分派，`resultCount` 统计
已完成的排队工具、分块和回复载荷。对于出站记录，
`durationMs` 包括直至终止状态的交付所有权时长（因此也包括
排队等待时间），而 `resultCount` 统计可识别的物理平台
发送次数。`deliveryKind`（如果存在）描述钩子处理后、
渲染后的有效载荷；被抑制和崩溃导致结果不明确的记录会省略该字段。

## Gateway RPC

`audit.activity.list` 需要 `operator.read`，并接受相同的筛选器。它
返回命名的 V1 活动事件联合类型，包括运行、工具、入站消息和
出站消息记录。

```bash
openclaw gateway call audit.activity.list --params '{"channel":"telegram","limit":50}'
```

结果为 `{ "events": AuditActivityEventV1[], "nextCursor"?: string }`。
结果按从新到旧排列，每个请求最多返回 500 条记录。

已发布的 `audit.list` RPC 对旧版运行/工具客户端保持不变。当
旧版 Gateway 网关不支持 `audit.activity.list` 时，仅当旧方法支持所有请求的筛选器，CLI 才会重试
`audit.list`。在旧版 Gateway 网关上，`--kind message`、
`--direction` 和 `--channel` 会失败并显示升级消息，而不会被静默丢弃。

## 相关内容

- [审计历史](/zh-CN/gateway/audit)
- [Gateway 网关协议](/zh-CN/gateway/protocol#audit-ledger-rpc)
- [会话](/zh-CN/cli/sessions)
- [任务](/zh-CN/cli/tasks)
- [定时任务](/zh-CN/automation/cron-jobs)
