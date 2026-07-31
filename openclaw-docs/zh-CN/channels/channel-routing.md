---
read_when:
    - 更改渠道路由或收件箱行为
summary: 每个渠道（WhatsApp、Telegram、Discord、Slack）的路由规则和共享上下文
title: 渠道路由
x-i18n:
    generated_at: "2026-07-26T06:02:04Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: aa03f04a55015bf17e0fe1f3a9bc422875124bb64af5891c898a98bc6917d9e8
    source_path: channels/channel-routing.md
    workflow: 16
---

# 渠道与路由

OpenClaw 将回复路由**回消息来源的渠道**。模型不会选择渠道；路由是确定性的，并由主机配置控制。在默认私信范围下，来自各个渠道的私信都会汇聚到智能体的[主会话](/concepts/main-session)。

## 关键术语

- **渠道**：内置渠道插件，例如 `discord`、`googlechat`、`imessage`、`irc`、`line`、`signal`、`slack`、`telegram` 或 `whatsapp`，以及已安装的插件渠道。`webchat` 是内部 WebChat UI 渠道，不是可配置的出站渠道。
- **AccountId**：每个渠道的账户实例（如果支持）。
- 可选的渠道默认账户：`channels.<channel>.defaultAccount` 用于选择
  在出站路径未指定 `accountId` 时使用哪个账户。
  - 在多账户设置中，如果配置了两个或更多账户，请设置显式默认账户（`defaultAccount` 或名为 `default` 的账户）。如果未设置，回退路由可能会选择第一个规范化的账户 ID。
- **AgentId**：隔离的工作区和会话存储（“大脑”）。
- **SessionKey**：用于存储上下文和控制并发的分桶键。

## 出站目标前缀

显式出站目标可以包含提供商前缀，例如 `telegram:123` 或 `tg:123`。仅当所选渠道为 `last` 或尚未解析，并且加载的插件声明了该前缀时，核心才会将该前缀视为渠道选择提示。如果调用方已显式选择渠道，则提供商前缀必须与该渠道匹配；诸如将 WhatsApp 消息投递到 `telegram:123` 之类的跨渠道组合，会在插件专属的目标规范化之前失败。

目标类型和服务前缀（例如 `channel:<id>`、`user:<id>`、`room:<id>`、`thread:<id>`、`imessage:<handle>` 和 `sms:<number>`）仍属于所选渠道的语法。它们本身不会选择提供商。

## 会话键格式（示例）

默认情况下，私信会归并到智能体的**主**会话：

- `agent:<agentId>:<mainKey>`（默认值：`agent:main:main`）

`session.dmScope` 控制私信归并：`main`（默认值）共享一个主
会话，而 `per-peer`、`per-channel-peer` 和 `per-account-channel-peer`
则将私信保留在独立会话中。路由绑定可以通过 `bindings[].session.dmScope`
覆盖其匹配对等方的范围。

即使私信对话历史与主会话共享，来自外部的私信也会使用派生的每账户私聊运行时键来应用沙箱和
工具策略，因此来自渠道的消息不会被当作本地主会话运行。

群组和渠道按渠道保持隔离：

- 群组：`agent:<agentId>:<channel>:group:<id>`
- 渠道/房间：`agent:<agentId>:<channel>:channel:<id>`

话题串：

- Slack/Discord 话题串会将 `:thread:<threadId>` 追加到基础键。
- Telegram 论坛话题会将 `:topic:<topicId>` 嵌入群组键。

示例：

- `agent:main:telegram:group:-1001234567890:topic:42`
- `agent:main:discord:channel:123456:thread:987654`

## 主私信路由固定

当 `session.dmScope` 为 `main` 时，私信可以共享一个主会话。
为防止会话的 `lastRoute` 被非所有者的私信覆盖，在以下所有条件均满足时，
OpenClaw 会从 `allowFrom` 推断固定的所有者：

- `allowFrom` 恰好包含一个非通配符条目。
- 该条目可以规范化为该渠道的具体发送者 ID。
- 入站私信发送者与该固定所有者不匹配。

在这种不匹配情况下，OpenClaw 仍会记录入站会话元数据，但会
跳过更新主会话的 `lastRoute`。

## 受保护的入站记录

当受保护的路径不得创建新的 OpenClaw 会话时，渠道插件可以将入站会话记录标记为 `createIfMissing: false`。
在此模式下，OpenClaw 可以更新现有会话的元数据和 `lastRoute`，但不会
仅仅因为观察到一条消息就创建只有路由信息的会话条目。

## 路由规则（如何选择智能体）

路由会为每条入站消息选择**一个智能体**：

1. **精确对等方匹配**（`bindings`，包含 `peer.kind` + `peer.id`）。
2. **父级对等方匹配**（话题串继承）。
3. **对等方通配符匹配**（对某个对等方类型使用 `peer.id: "*"`）。
4. **服务器 + 角色匹配**（Discord），通过 `guildId` + `roles`。
5. **服务器匹配**（Discord），通过 `guildId`。
6. **团队匹配**（Slack），通过 `teamId`。
7. **账户匹配**（渠道上的 `accountId`）。
8. **渠道匹配**（该渠道上的任意账户，`accountId: "*"`）。
9. **默认智能体**（`agents.entries.*.default`，否则使用列表中的第一个条目，最后回退到 `main`）。

当绑定包含多个匹配字段（`peer`、`guildId`、`teamId`、`roles`）时，**所有提供的字段都必须匹配**，该绑定才会生效。

匹配的智能体决定使用哪个工作区和会话存储。

## 广播群组（运行多个智能体）

广播群组允许针对同一个对等方运行**多个智能体**，但仅在 **OpenClaw 通常会回复时**（例如：在 WhatsApp 群组中通过提及/激活门控后）。

配置：

```json5
{
  broadcast: {
    strategy: "parallel",
    "120363403215116621@g.us": ["alfred", "baerbel"],
    "+15555550123": ["support", "logger"],
  },
}
```

另请参阅：[广播群组](/zh-CN/channels/broadcast-groups)。

## 配置概览

- `agents.entries`：命名的智能体定义（工作区、模型等）。
- `bindings`：将入站渠道/账户/对等方映射到智能体。

示例：

```json5
{
  agents: {
    list: [{ id: "support", name: "Support", workspace: "~/.openclaw/workspace-support" }],
  },
  bindings: [
    { match: { channel: "slack", teamId: "T123" }, agentId: "support" },
    { match: { channel: "telegram", peer: { kind: "group", id: "-100123" } }, agentId: "support" },
  ],
}
```

## 会话存储

运行时会话行位于每个智能体的 SQLite 数据库中，存放在状态
目录下（默认值为 `~/.openclaw`）：

- `~/.openclaw/agents/<agentId>/agent/openclaw-agent.sqlite`

较早的安装可能在 `~/.openclaw/agents/<agentId>/sessions/` 下包含旧版对话记录 JSONL 文件和 `sessions.json` 行
存储。Gateway 网关启动时和
`openclaw doctor --fix` 会自动将活跃的旧版行/历史记录导入 SQLite。
需要明确的迁移证据时，请使用 `openclaw doctor --session-sqlite inspect
--session-sqlite-all-agents` 和
[Doctor](/zh-CN/cli/doctor#session-sqlite-migration) 验证流程。
对于迁移和离线维护工作流，仍可通过 `session.store` 和 `{agentId}`
模板选择旧版存储路径。

Gateway 网关和 ACP 会话发现还会扫描默认 `agents/` 根目录以及模板化
`session.store` 根目录下基于磁盘的智能体存储。发现的存储必须位于解析后的智能体根目录内，
并使用常规旧版 `sessions.json` 文件。符号链接和根目录之外的路径会被忽略。

## WebChat 行为

WebChat 会连接到**所选智能体**，并默认使用该智能体的主
会话。因此，WebChat 允许你在一个位置查看该
智能体的跨渠道上下文。

## 回复上下文

入站回复包括：

- 可用时包括 `ReplyToId`、`ReplyToBody` 和 `ReplyToSender`。
- 引用的上下文会作为 `[Replying to ...]` 块追加到 `Body`。

此行为在所有渠道中保持一致。

## 相关内容

- [群组](/zh-CN/channels/groups)
- [广播群组](/zh-CN/channels/broadcast-groups)
- [配对](/zh-CN/channels/pairing)
