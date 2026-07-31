---
read_when:
    - 添加扩大访问范围或自动化程度的功能
summary: 运行具备 Shell 访问权限的 AI Gateway 网关时的安全注意事项与威胁模型
title: 安全性
x-i18n:
    generated_at: "2026-07-26T06:10:07Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 8cdf1b1455ecb35a3cf5b9ab968a55c89b7b7c283231b99d4d740bb75fa11700
    source_path: gateway/security/index.md
    workflow: 16
---

<Warning>
  **个人助理信任模型。** 本指南假定每个 Gateway 网关对应一个可信的
  操作员边界（单用户、个人助理模型）。
  OpenClaw **不是**供多个敌对用户共享同一智能体或 Gateway 网关的恶意多租户安全边界。
  对于混合信任或敌对用户操作，应拆分信任边界：使用独立的 Gateway 网关和
  凭据，最好还使用独立的操作系统用户或主机。
</Warning>

## 范围：个人助理安全模型

- 支持：每个 Gateway 网关一个用户/信任边界（每个边界最好使用一个操作系统用户/主机/VPS）。
- 不支持：互不信任或相互敌对的用户共用一个 Gateway 网关/智能体。
- 敌对用户隔离需要使用独立的 Gateway 网关（最好还使用独立的操作系统用户/主机）。
- 如果多个不受信任的用户可以向同一个启用了工具的智能体发送消息，他们会共享该智能体被委派的工具权限。
- 如果某人可以修改 Gateway 网关主机的状态/配置（`~/.openclaw`，包括 `openclaw.json`），应将其视为可信操作员。
- 在一个 Gateway 网关内，经过身份验证的操作员访问是一种可信的控制平面角色，而不是按用户划分的租户角色。
- `sessionKey`（会话 ID、标签）是路由选择器，而不是授权令牌。

要托管多个用户或组织？请为每个租户运行一个隔离的 Gateway 网关单元，而不是共享 Gateway 网关。请参阅[多租户托管](/zh-CN/gateway/multi-tenant-hosting)。

在更改远程访问、私信策略、反向代理或公网暴露之前，请按照 [Gateway 暴露运行手册](/zh-CN/gateway/security/exposure-runbook)完成预检/回滚检查。

## `openclaw security audit`

在任何配置更改后或暴露网络接口前运行：

```bash
openclaw security audit
openclaw security audit --deep    # 尝试实时探测 Gateway 网关
openclaw security audit --fix     # 应用安全的修复措施
openclaw security audit --json
```

`--fix` 的范围有意保持狭窄：它会将开放的群组策略改为允许列表、恢复 `logging.redactSensitive: "tools"`、收紧状态/配置/包含文件的权限（`600` 文件、`700` 目录），并在 Windows 上使用 ACL 重置而不是 POSIX `chmod`。

### 审计检查的内容（概要）

- **入站访问** - 私信/群组策略、允许列表：陌生人能否触发该 Bot？
- **工具影响范围** - 提升权限的工具 + 开放房间：提示词注入是否可能转化为 shell/文件/网络操作？
- **Exec 文件系统策略偏移** - 拒绝会修改文件系统的工具，但 `exec`/`process` 在不受沙箱约束的情况下仍然可用。
- **Exec 审批策略偏移** - `security="full"`、`autoAllowSkills`、缺少 `strictInlineEval` 的解释器允许列表。仅存在 `security="full"` 是一项宽泛的安全态势警告，并不能证明存在错误——它是可信个人助理设置所选用的默认值；仅当你的威胁模型需要审批或允许列表防护时才应收紧。
- **网络暴露** - Gateway 网关绑定/身份验证、Tailscale Serve/Funnel、较弱或过短的身份验证令牌。
- **浏览器控制暴露** - 远程节点、中继端口、远程 CDP 端点。
- **本地磁盘安全卫生** - 权限、符号链接、配置包含项、同步文件夹路径。
- **插件** - 未设置明确允许列表便加载插件。
- **策略偏移** - 已配置沙箱 Docker 设置，但沙箱模式处于关闭状态；`gateway.nodes.commands.deny` 条目看似有效，实际只匹配确切的命令 ID（例如 `system.run`），而不匹配载荷中的 shell 文本；危险的 `gateway.nodes.commands.allow` 条目；全局 `tools.profile="minimal"` 被按智能体配置覆盖；插件所属工具在宽松策略下可访问。
- **运行时预期偏移** - 在 `tools.exec.host` 现在默认设为 `auto` 时，仍假定隐式 Exec 表示 `sandbox`；或在沙箱模式关闭时设置 `tools.exec.host="sandbox"`。
- **模型安全卫生** - 对已配置的旧版模型发出警告（软警告，不是硬性阻止）。

每项发现都有一个结构化的 `checkId`（例如 `gateway.bind_no_auth`、`tools.exec.security_full_configured`）。前缀：`fs.*`（权限）、`gateway.*`（绑定/身份验证/Tailscale/Control UI/可信代理）、`hooks.*`/`browser.*`/`sandbox.*`/`tools.exec.*`（按接口加固）、`plugins.*`/`skills.*`（供应链）、`security.exposure.*`（访问策略 × 工具影响范围）。包含严重性和自动修复支持情况的完整目录，请参阅[安全审计检查](/zh-CN/gateway/security/audit-checks)。另请参阅[形式化验证](/zh-CN/security/formal-verification)。

### 对发现进行分类处理时的优先顺序

1. 任何“开放”且启用了工具的环境：首先限制私信/群组（配对/允许列表），然后收紧工具策略/沙箱隔离。
2. 公网暴露（LAN 绑定、Funnel、缺少身份验证）：立即修复。
3. 浏览器控制远程暴露：将其视同操作员访问（仅限 tailnet、有意配对节点、不得公开暴露）。
4. 权限：状态/配置/凭据/身份验证信息不得允许组内其他用户或所有用户读取。
5. 插件：仅加载你明确信任的插件。
6. 模型选择：对于任何配备工具的 Bot，优先使用现代且经过指令加固的模型。

## 60 秒完成加固基线配置

```json5
{
  gateway: {
    mode: "local",
    bind: "loopback",
    auth: { mode: "token", token: "replace-with-long-random-token" },
  },
  session: {
    dmScope: "per-channel-peer",
  },
  tools: {
    profile: "messaging",
    deny: ["group:automation", "group:runtime", "group:fs", "sessions_spawn", "sessions_send"],
    fs: { workspaceOnly: true },
    exec: { security: "deny", ask: "always" },
    elevated: { enabled: false },
  },
  channels: {
    whatsapp: { dmPolicy: "pairing", groups: { "*": { requireMention: true } } },
  },
}
```

使 Gateway 网关仅在本地可用、隔离私信，并默认禁用控制平面/运行时工具。此后可按可信智能体选择性地重新启用工具。

聊天驱动的智能体轮次内置基线：无论如何配置，非所有者发送者都无法使用 `cron` 或 `gateway` 工具。

### 请求者范围的控制和提示词上下文

`tools.toolsBySender`、发送者所有权以及仅限所有者的工具清单，均针对当前轮次的原始请求者进行评估。它们不会对该模型提示词中的其他内容进行身份验证或清理，包括引用文本、此前的共享房间历史记录、转发内容、获取的内容、附件、工具结果或其他提示词输入。因此，当其他人的内容包含在某个由所有者触发的轮次上下文中时，该内容可能会影响该轮次。

应将这些控制措施视为减少请求者直接能力的纵深防御，而不是敌对多用户隔离。使用 `contextVisibility` 过滤受支持渠道提供的上下文、限制工具并对智能体实施沙箱隔离；当参与者相互敌对时，应使用独立的 Gateway 网关，最好还使用独立的操作系统用户或主机。

## 信任边界矩阵

用于分类处理风险报告的快速模型：

| 边界或控制                                                | 含义                                              | 常见误解                                                                      |
| --------------------------------------------------------- | ------------------------------------------------- | ----------------------------------------------------------------------------- |
| `gateway.auth`（令牌/密码/可信代理/设备身份验证）     | 对 Gateway 网关 API 的调用者进行身份验证          | “为确保安全，每个帧中的每条消息都需要签名”                                    |
| `sessionKey`                                        | 用于上下文/会话选择的路由键                       | “会话键是用户身份验证边界”                                                     |
| 提示词/内容防护措施                                       | 降低模型被滥用的风险                              | “仅凭提示词注入就能证明身份验证被绕过”                                         |
| `canvas.eval` / 浏览器 evaluate                      | 启用后提供有意授予的操作员能力                    | “在此信任模型中，任何 JS eval 原语都会自动构成漏洞”                            |
| 本地 TUI `!` shell                         | 由操作员明确触发的本地执行                        | “本地 shell 便捷命令属于远程注入”                                              |
| 节点配对和节点命令                                        | 在已配对设备上执行操作员级远程操作                | “默认情况下，应将远程设备控制视为不受信任的用户访问”                           |
| `gateway.nodes.pairing.autoApproveCidrs`                                        | 可选启用的可信网络节点注册策略                    | “默认禁用的允许列表自动构成配对漏洞”                                           |
| `gateway.nodes.pairing.sshVerify`                                        | 通过操作员 SSH 进行密钥验证的节点注册             | “默认启用的自动批准自动构成配对漏洞”                                           |

## 按设计不属于漏洞

<Accordion title="通常无需采取行动而关闭的发现">

- 仅涉及提示词注入、但未绕过策略、身份验证或沙箱的攻击链。
- 假定在一个共享主机或配置上进行敌对多租户操作的主张。
- 在共享 Gateway 网关设置中，将正常的操作员读取路径访问（例如 `sessions.list` / `sessions.preview` / `chat.history`）归类为 IDOR。
- 仅限 localhost 部署的发现（例如，仅限 local loopback 的 Gateway 网关缺少 HSTS）。
- 针对本仓库中不存在的入站路径提出的 Discord 入站 Webhook 签名问题。
- 将节点配对元数据视为 `system.run` 的隐藏第二层逐命令审批；真正的执行边界是 Gateway 网关的全局节点命令策略以及节点自身的 Exec 审批。
- 因为默认启用而将 `gateway.nodes.pairing.sshVerify` 视为漏洞。它绝不会仅根据网络位置或 SSH 可达性进行批准：Gateway 网关会通过 SSH（BatchMode、严格主机密钥）回读设备身份，并且仅在设备密钥与待处理请求完全匹配时才批准；这要求连接密钥对已经存在于操作员所控制主机上的操作员账户中。探测范围仅限私有/CGNAT 源地址，并遵循可信 CIDR 的资格下限（仅限新近且无权限范围的 `role: node`），而 `sshVerify: false` 可关闭此功能。
- 将 `gateway.nodes.pairing.autoApproveCidrs` 本身视为漏洞。它默认禁用，需要显式配置 CIDR/IP 条目，仅适用于首次配对且未请求权限范围的 `role: node`，并且绝不会自动批准操作员/浏览器/Control UI、WebChat、角色/权限范围升级、元数据或公钥变更，也不会自动批准同一主机的 local loopback 可信代理标头路径（即使已启用 local loopback 可信代理身份验证）。
- 将 `sessionKey` 视为身份验证令牌的“缺少按用户授权”发现。

</Accordion>

## Gateway 网关和节点信任

将 Gateway 网关和节点视为同一操作员信任域中承担不同角色的组件：

- **Gateway 网关**：控制平面和策略界面（`gateway.auth`、工具策略、路由）。
- **节点**：与该 Gateway 网关配对的远程执行界面（命令、设备操作、主机本地能力）。
- 通过 Gateway 网关身份验证的调用方在 Gateway 网关范围内受到信任；配对后，节点操作将被视为该节点上受信任的操作员操作。请参阅[操作员权限范围](/zh-CN/gateway/operator-scopes)。
- 使用共享 Gateway 网关令牌/密码进行身份验证的直接 local loopback 后端客户端，无需提供用户设备身份即可调用内部控制平面 RPC。这并非绕过远程或浏览器配对——网络客户端、节点客户端、设备令牌客户端和显式设备身份仍需经过配对和权限范围升级强制检查。
- Exec 审批（允许列表 + 询问）是保障操作员意图的防护措施，而不是用于隔离恶意多租户。它们绑定确切的请求上下文，并尽力绑定直接的本地文件操作数；但不会在语义上对每条运行时/解释器加载路径建模。请使用沙箱隔离和主机隔离来建立强边界。
- 受信任的单操作员默认设置：在 `gateway`/`node` 上执行主机命令无需审批提示（`security="full"`、`ask="off"`）。这是有意设计的用户体验，其本身并非漏洞。

要隔离恶意用户，请按操作系统用户/主机划分信任边界，并运行独立的 Gateway 网关。

## 威胁模型

你的 AI 助手可以执行任意 shell 命令、读写文件、访问网络服务，并向任何人发送消息（如果获得了渠道访问权限）。向它发送消息的人可能会试图诱骗它执行恶意操作、通过社会工程手段获取你的数据访问权限，或探查基础设施详情。

这里的大多数故障并非奇特的漏洞利用，而是“有人向 Bot 发送了消息，Bot 按照对方的要求执行了操作”。OpenClaw 按以下优先顺序采取应对策略：

1. **身份优先**——决定谁可以与 Bot 交互（私信配对/允许列表/显式“开放”）。
2. **其次限定范围**——决定 Bot 可以在哪里执行操作（群组允许列表 + 提及门控、工具、沙箱隔离、设备权限）。
3. **最后考虑模型**——假设模型可能受到操纵；进行相应设计，使操纵造成的影响范围有限。

## 私信访问：配对、允许列表、开放、禁用

每个支持私信的渠道都支持 `dmPolicy`（或 `*.dm.policy`），在处理消息之前对入站私信进行门控：

| 策略      | 行为                                                                                                                                                                                                             |
| ----------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `pairing`   | 默认。未知发送者会收到配对码；在获得批准之前，Bot 会忽略其消息。配对码将在 1 小时后过期；在创建新请求之前，重复发送私信不会再次发送配对码。每个渠道最多可有 3 个待处理请求。 |
| `allowlist` | 阻止未知发送者，不进行配对握手。                                                                                                                                                                       |
| `open`      | 任何人都可以发送私信（公开）。要求渠道允许列表包含 `"*"`（显式选择启用）。                                                                                                                           |
| `disabled`  | 完全忽略入站私信。                                                                                                                                                                                        |

```bash
openclaw pairing list <channel>
openclaw pairing approve <channel> <code>
```

详细信息及磁盘上的文件：[配对](/zh-CN/channels/pairing)

请将 `dmPolicy="open"` 和 `groupPolicy="open"` 视为最后手段；除非你完全信任聊天室中的每个成员，否则应优先使用配对 + 允许列表。

### 允许列表（两层）

- **私信允许列表**（`allowFrom` / `channels.discord.allowFrom` / `channels.slack.allowFrom`；旧版：`channels.discord.dm.allowFrom`、`channels.slack.dm.allowFrom`）：哪些人可以向 Bot 发送私信。当 `dmPolicy="pairing"` 时，批准结果会写入 `~/.openclaw/credentials/<channel>-allowFrom.json`（默认账户）或 `<channel>-<accountId>-allowFrom.json`（非默认账户），并与配置中的允许列表合并。
- **群组允许列表**（因渠道而异）：Bot 接受哪些群组/渠道/服务器。
  - `channels.whatsapp.groups`、`channels.telegram.groups`、`channels.imessage.groups`：每个群组的默认设置，例如 `requireMention`；设置后也会充当群组允许列表（包含 `"*"` 可保留允许所有群组的行为）。使用 `agents.entries.*.groupChat.mentionPatterns`（例如 `["@openclaw", "@mybot"]`）自定义提及触发条件，使 `requireMention` 根据你自己的 Bot 名称进行门控。
  - `groupPolicy="allowlist"` + `groupAllowFrom`：限制谁可以在群组会话中触发 Bot（WhatsApp/Telegram/Signal/iMessage/Microsoft Teams）。
  - `channels.discord.guilds` / `channels.slack.channels`：每个界面的允许列表 + 提及默认设置。
  - 检查顺序：先检查 `groupPolicy`/群组允许列表，再检查提及/回复激活条件。回复 Bot 消息（隐式提及）**不会**绕过 `groupAllowFrom`。

详细信息：[配置](/zh-CN/gateway/configuration)和[群组](/zh-CN/channels/groups)

### 私信会话隔离（多用户模式）

默认情况下，OpenClaw 会将所有私信路由到主会话，以保持跨设备连续性。如果多人可以向 Bot 发送私信（开放私信或包含多人的允许列表），请隔离私信会话：

```json5
{ session: { dmScope: "per-channel-peer" } }
```

`session.dmScope` 的值：

| 值                      | 范围                                                                  |
| -------------------------- | ---------------------------------------------------------------------- |
| `main`（配置默认值）    | 所有私信共享一个会话。                                             |
| `per-channel-peer`         | 每个渠道+发送者组合都获得独立的私信上下文（安全私信模式）。 |
| `per-account-channel-peer` | 与上一项类似，但会进一步按账户拆分（多账户渠道）。         |
| `per-peer`                 | 每个发送者在所有同类型渠道中共享一个会话。     |

本地 CLI 新手引导会保留显式设置的 `session.dmScope`，否则将其留空，从而应用 `"main"` 默认值：所有渠道中的私信共享该智能体持续更新的主会话（个人智能体默认设置）。对于共享或多用户收件箱，请设置 `session.dmScope: "per-channel-peer"`；当 `openclaw security audit` 检测到多用户私信流量时，会建议启用隔离。

这是消息上下文边界，而不是主机管理员边界。如果用户之间相互敌对，且共享同一 Gateway 网关主机/配置，请为每个信任边界运行独立的 Gateway 网关。

如果同一个人通过多个渠道与你联系，请使用 `session.identityLinks` 将这些私信会话合并到一个规范身份中。请参阅[会话管理](/zh-CN/concepts/session)和[配置](/zh-CN/gateway/configuration)。

## 上下文可见性与触发授权

这是两个不同的概念：

- **触发授权**：谁可以触发智能体（`dmPolicy`、`groupPolicy`、允许列表、提及门控）。
- **上下文可见性**：哪些补充上下文会传递给模型（回复正文、引用文本、话题串历史记录、转发元数据）。

`contextVisibility` 控制第二项：

- `"all"`（默认）：按接收时的原样保留补充上下文。
- `"allowlist"`：将补充上下文过滤为通过当前允许列表检查的发送者所提供的内容。
- `"allowlist_quote"`：与 `allowlist` 类似，但仍保留一个显式引用回复。

可按渠道或聊天室/对话设置——请参阅[群组](/zh-CN/channels/groups#context-visibility-and-allowlists)。如果报告仅表明“模型可以看到来自不在允许列表中的发送者的引用文本/历史文本”，这属于可通过 `contextVisibility` 解决的安全加固问题，其本身并非身份验证或沙箱绕过；要认定报告具有安全影响，仍需证明存在信任边界绕过。

## 提示词注入

攻击者会精心构造消息，操纵模型执行不安全的操作（“忽略你的指令”“转储你的文件系统”“访问此链接并运行命令”）。仅依靠系统提示词防护**无法解决**提示词注入——这些只是软性指导；强制执行依赖工具策略、Exec 审批、沙箱隔离和渠道允许列表（操作员仍可按设计禁用这些机制）。

提示词注入并不要求开放公开私信：即使只有你能向 Bot 发送消息，它读取的任何**不受信任内容**（Web 搜索/获取结果、浏览器页面、电子邮件、文档、附件、粘贴的日志/代码）也可能携带恶意指令。内容本身就是攻击面，而不仅仅是发送者。

应视为不受信任的危险信号：

- “读取此文件/URL，并完全按照其中的要求操作。”
- “忽略你的系统提示词或安全规则。”
- “泄露你的隐藏指令或工具输出。”
- “粘贴 ~/.openclaw 或日志的完整内容。”

实践中有效的措施：

- 严格限制入站私信（配对/允许列表）；在群组中优先使用提及门控；避免在公共聊天室中运行始终在线的 Bot。
- 默认将链接、附件和粘贴的指令视为恶意内容。
- 在沙箱中执行敏感工具；确保智能体可访问的文件系统中不包含密钥。沙箱隔离需显式启用：如果沙箱模式关闭，隐式 `host=auto` 会解析为 Gateway 网关主机，而显式 `host=sandbox` 仍会以关闭方式失败（没有可用的沙箱运行时）。设置 `host=gateway` 可在配置中明确指定此行为。
- 将高风险工具（`exec`、`browser`、`web_fetch`、`web_search`）限制为仅供受信任的智能体或显式允许列表使用。
- 如果将解释器（`python`、`node`、`ruby`、`perl`、`php`、`lua`、`osascript`）加入允许列表，请启用 `tools.exec.strictInlineEval`，使内联求值形式（`-c`、`-e` 及类似形式）仍需显式批准。在允许列表模式下，任何 here-document 片段（`<<`）无论如何引用，都始终需要审查者批准或显式批准——已加入允许列表的命令不能利用 here-document 正文绕过允许列表审查。
- 使用只读或禁用工具的**读取智能体**汇总不受信任的内容，然后将摘要传递给主智能体，以缩小影响范围。
- 对于 Gmail Hooks，内置的逐消息会话可以隔离对话上下文，但不会移除目标智能体的工具或工作区权限。请将不受信任的邮件路由到专用读取智能体，应用[按 Agent 配置的沙箱和工具限制](/zh-CN/tools/multi-agent-sandbox-tools)，并使用 [`tools.agentToAgent`](/zh-CN/gateway/config-tools#toolsagenttoagent) 限制向主智能体进行的任何移交。请参阅 [Gmail 集成](/zh-CN/gateway/configuration-reference#gmail-integration)。
- 除非确有需要，否则对于启用了工具的智能体，请关闭 `web_search` / `web_fetch` / `browser`。
- 对于 OpenResponses URL 输入（`input_file` / `input_image`），请严格限制 `gateway.http.endpoints.responses.files.urlAllowlist` / `images.urlAllowlist`，并将 `maxUrlParts` 保持在较低值（空允许列表视为未设置）。使用 `files.allowUrl: false` / `images.allowUrl: false` 可完全禁用 URL 获取。
- 不要在提示词中包含密钥；改为通过 Gateway 网关主机上的环境变量/配置传递。

**模型选择很重要。** 不同模型层级抵御提示词注入的能力并不相同——在对抗性提示词下，较小/较便宜的模型更容易被诱导滥用工具或劫持指令。

<Warning>
对于启用了工具的智能体或会读取不受信任内容的智能体，使用较旧/较小模型带来的提示词注入风险通常过高。不要在能力较弱的模型层级上运行这些工作负载。
</Warning>

- 对于任何能够运行工具或访问文件/网络的 Bot，请使用最新一代的最高层级模型。
- 不要为启用了工具的智能体或不受信任的收件箱使用较旧/较弱/较小的模型层级。
- 如果必须使用较小模型，请缩小影响范围：使用只读工具、强沙箱隔离、最少的文件系统访问权限和严格的允许列表。为所有会话启用沙箱隔离，并禁用 `web_search`/`web_fetch`/`browser`，除非输入受到严格控制。
- 对于仅用于聊天、输入可信且不使用工具的个人助理，较小模型通常没有问题。

### 外部内容和不受信任输入的封装

即使 Gateway 网关在本地解码 OpenResponses `input_file` 文本，它仍会作为不受信任的外部内容注入——该块包含 `<<<EXTERNAL_UNTRUSTED_CONTENT ...>>>` 边界标记以及 `Source: External` 元数据（此路径省略了其他位置使用的较长 `SECURITY NOTICE:` 横幅）。当媒体理解从附加文档中提取文本并将其追加到媒体提示词之前，也会采用相同的基于标记的封装方式。

OpenClaw 还会在封装后的外部内容和元数据到达模型之前，移除常见自托管 LLM 聊天模板中的特殊令牌字面量（Qwen/ChatML、Llama、Gemma、Mistral、Phi、GPT-OSS 的角色/轮次令牌）。自托管的 OpenAI 兼容后端（vLLM、SGLang、TGI、LM Studio、自定义 Hugging Face 分词器栈）有时会将用户内容中类似 `<|im_start|>` 或 `<|start_header_id|>` 的字面量字符串标记化为结构化聊天模板令牌；如果不进行这种净化，所获取页面、电子邮件正文或文件内容工具输出中的不受信任文本就可能伪造合成的 `assistant`/`system` 角色边界。净化发生在外部内容封装层，因此会统一应用于获取/读取工具以及入站渠道内容。托管提供商（OpenAI、Anthropic）已在请求侧应用各自的净化措施；请保持启用外部内容封装，并在可用时优先选择能够拆分/转义特殊令牌的后端设置。

出站模型响应具有独立的净化器，会在最终渠道交付边界移除泄漏的 `<tool_call>`、`<function_calls>`、`<system-reminder>`、`<previous_response>` 以及类似的内部脚手架，使其不会出现在用户可见的回复中。

这不能取代 `dmPolicy`、允许列表、Exec 审批、沙箱隔离或 `contextVisibility`——它只封堵一个特定的分词器层绕过路径。

### 绕过标志（在生产环境中保持关闭）

- `hooks.mappings[].allowUnsafeExternalContent`
- `hooks.gmail.allowUnsafeExternalContent`
- 定时任务负载字段 `allowUnsafeExternalContent`

仅在范围严格受限的调试中临时启用；如果启用，请隔离该智能体（沙箱 + 最少工具 + 专用会话命名空间）。

即使交付来自你控制的系统，钩子负载也属于不受信任内容（邮件/文档/Web 内容可能携带提示词注入）。较弱的模型层级会增加此风险——对于钩子驱动的自动化，请优先使用能力强大的现代模型层级并严格限制工具策略（`tools.profile: "messaging"` 或更严格），同时尽可能启用沙箱隔离。

### 群组中的推理和详细输出

`/reasoning`、`/verbose` 和 `/trace` 可能暴露不应出现在公共渠道中的内部推理、工具输出或插件诊断信息——其中可能包含工具参数、URL、插件诊断信息以及模型看到的数据。请在公共房间中禁用这些功能；仅在可信私信或受到严格控制的房间中启用。

## 命令授权

只有获得授权的发送者发出的斜杠命令和指令才会生效；授权依据为渠道允许列表/配对以及 `commands.useAccessGroups`（请参阅[配置](/zh-CN/gateway/configuration)和[斜杠命令](/zh-CN/tools/slash-commands)）。如果渠道允许列表为空或包含 `"*"`，则该渠道实际上对所有人开放命令。

`/exec` 只是为已授权操作员提供的会话级便利功能——它不会写入配置，也不会更改其他会话。

## 控制平面工具

两个内置工具仍涉及敏感的控制平面操作：

- `gateway` 使用 `config.schema.lookup` / `config.get` 读取配置。它无法写入配置、更新 OpenClaw 或重启 Gateway 网关。
- `cron` 创建在原始聊天/任务结束后仍会继续运行的定时任务。

`gateway` 工具仍仅限所有者使用，因为读取配置可能暴露密钥和主机拓扑。智能体通过 `openclaw` 委托工具请求持久配置或生命周期变更；OpenClaw 会将这些请求映射为类型化操作，并要求人工批准后才会应用。请参阅 [OpenClaw 设置智能体](/zh-CN/cli/openclaw#operations-and-approval)。

对于任何处理不受信任内容的智能体/界面，默认拒绝以下工具：

```json5
{
  tools: {
    deny: ["gateway", "cron", "sessions_spawn", "sessions_send"],
  },
}
```

`commands.restart=false` 会禁用 `/restart` 和外部 `SIGUSR1` 重启请求。`gateway` 智能体工具没有重启操作。

## 节点执行（`system.run`）

如果已配对 macOS 节点，Gateway 网关便可在该节点上调用 `system.run`——这相当于在该 Mac 上远程执行代码。

- 需要节点配对（批准 + 令牌）。配对会建立节点身份/信任关系并签发令牌；它不是逐命令审批机制。
- Gateway 网关通过 `gateway.nodes.commands.allow` / `gateway.nodes.commands.deny` 应用粗粒度的全局节点命令策略。拒绝列表仅匹配确切的节点命令名称（例如 `system.run`），而不匹配命令负载中的 Shell 文本——如果重新连接的节点公布了不同的命令列表，只要 Gateway 网关的全局策略和节点自身的 Exec 审批仍能强制执行边界，这本身并不构成漏洞。
- 每个节点的 `system.run` 策略是节点自身的 Exec 审批文件（`exec.approvals.node.*`），可在 Mac 上通过 Settings -> Exec approvals（安全 + 询问 + 允许列表）进行控制；它可以比 Gateway 网关的全局命令 ID 策略更严格或更宽松。
- 运行 `security="full"` 和 `ask="off"` 的节点遵循默认的可信操作员模型——这是预期行为，并非 Bug，除非你的部署需要采取更严格的立场。
- 审批模式会绑定确切的请求上下文，并在可能时绑定一个具体的本地脚本/文件操作数。如果 OpenClaw 无法为解释器/运行时命令准确识别唯一一个直接本地文件，则会拒绝依赖审批的执行，而不会承诺提供完整的语义覆盖。
- 对于 `host=node`，依赖审批的运行还会存储规范化且已准备好的 `systemRunPlan`；后续已批准的转发会复用该已存储计划，而 Gateway 网关验证会拒绝调用方修改审批请求创建后的命令/cwd/会话上下文。
- 要完全禁用远程执行：将安全性设置为 `deny`，并移除该 Mac 的节点配对。

## 动态 Skills（监视器/远程节点）

OpenClaw 可以在会话进行期间刷新 Skills 列表：当 `SKILL.md` 发生变化时，Skills 监视器会在智能体的下一轮更新快照；连接 macOS 节点还可能使仅限 macOS 的 Skills 符合条件（基于二进制文件探测）。应将 Skill 文件夹视为可信代码，并限制其修改权限。

## 插件

插件与 Gateway 网关在同一进程中运行——应将其视为可信代码。

- 仅从你信任的来源安装；优先使用明确的 `plugins.allow` 允许列表；启用前审查插件配置；更改插件后重启 Gateway 网关。
- 安装/更新插件会运行可执行代码：
  - 安装路径是当前插件安装根目录下每个插件各自的目录。
  - ClawHub 软件包以及 OpenClaw 的内置/官方目录属于可信来源。安装新的任意 npm、`npm-pack:`、git、本地路径/归档或市场来源前会发出警告；非交互式安装要求你在审查并信任该来源后使用 `--force`。`--force` 会确认来源并允许覆盖；它不会绕过 `security.installPolicy` 或其余安装安全检查。更新会复用已选择的来源。
  - OpenClaw 在安装/更新期间不会运行内置的本地危险代码阻止机制。使用 `security.installPolicy` 执行由操作员管理的本地允许/阻止决策，并使用 `openclaw security audit --deep` 进行诊断扫描。
  - npm 和 git 插件安装仅在明确的安装/更新流程中运行包管理器依赖收敛。本地路径和归档会被视为自包含软件包；OpenClaw 会复制/引用它们，而不会运行 `npm install`。
  - 优先使用锁定的精确版本（`@scope/pkg@1.2.3`），并在启用前检查解包后的代码。
  - `--dangerously-force-unsafe-install` 已弃用，不再改变安装/更新行为。
  - `security.installPolicy` 允许操作员运行可信的本地命令，针对 Skills 和插件安装做出特定于主机的允许/阻止决策。它会在来源材料暂存后、安装继续前运行，也适用于 ClawHub Skills，并且无法通过已弃用的不安全标志绕过。

详情：[插件](/zh-CN/tools/plugin)

## 沙箱隔离

专门文档：[沙箱隔离](/zh-CN/gateway/sandboxing)

两种互补方式：

- **在 Docker 中运行完整 Gateway 网关**（容器边界）：[Docker](/zh-CN/install/docker)
- **工具沙箱**（`agents.defaults.sandbox`；主机 Gateway 网关 + 由沙箱隔离的工具；Docker 是默认后端）：[沙箱隔离](/zh-CN/gateway/sandboxing)

<Note>
为防止智能体之间相互访问，请将 `agents.defaults.sandbox.scope` 保持为 `"agent"`（默认值），或使用 `"session"` 实现更严格的逐会话隔离。`scope: "shared"` 使用单个容器或工作区。
</Note>

沙箱内的 Agent 工作区访问权限（`agents.defaults.sandbox.workspaceAccess`）：

- `"none"`（默认）：工具可以看到 `~/.openclaw/sandboxes` 下的沙箱工作区；无法访问 Agent 工作区。
- `"ro"`：将 Agent 工作区以只读方式挂载到 `/agent`（禁用 `write`/`edit`/`apply_patch`）。
- `"rw"`：将 Agent 工作区以读写方式挂载到 `/workspace`。

额外的 `sandbox.docker.binds` 会依据经过规范化和标准化的源路径进行验证。阻止路径拒绝列表涵盖 `/etc`、`/private/etc`、`/proc`、`/sys`、`/dev`、`/root`、`/boot`，以及通常包含 Docker 套接字或作为其别名的目录（其中的 `/run`、`/var/run` 和 `docker.sock`），此外还包括 HOME 凭据子路径（`.aws`、`.cargo`、`.config`、`.docker`、`.gnupg`、`.netrc`、`.npm`、`.ssh`）。系统会通过现有祖先目录解析父级符号链接伎俩和规范化的主目录别名，然后重新检查；因此，如果它们最终解析到被阻止的根目录，仍会以失败关闭方式拒绝。

<Warning>
`tools.elevated` 是全局基线逃生通道，会在沙箱外运行 Exec。默认情况下，有效主机为 `gateway`；当 Exec 目标配置为 `node` 时，则为 `node`。请严格限制 `tools.elevated.allowFrom`，不要为陌生人启用。可通过 `agents.entries.*.tools.elevated` 按智能体进一步限制。请参阅[提升权限模式](/zh-CN/tools/elevated)。
</Warning>

### 子智能体委托防护措施

如果允许使用会话工具，请将委派子智能体运行视为另一项边界决策：

- 除非智能体确实需要委派，否则请拒绝 `sessions_spawn`。
- 将 `agents.defaults.subagents.allowAgents` 和任何按智能体配置的 `agents.entries.*.subagents.allowAgents` 覆盖限制为已知安全的目标智能体。
- 对于必须保持沙箱隔离的工作流，请调用 `sessions_spawn` 并传入 `sandbox: "require"`（默认为 `"inherit"`）；当目标子运行时未沙箱隔离时，`"require"` 会快速失败。

### 只读模式

通过将 `agents.defaults.sandbox.workspaceAccess: "ro"`（或使用 `"none"` 禁止访问工作区）与阻止 `write`、`edit`、`apply_patch`、`exec`、`process` 等工具的允许/拒绝列表结合，构建只读配置文件。

- `tools.exec.applyPatch.workspaceOnly: true`（默认）：即使关闭沙箱隔离，也会阻止 `apply_patch` 在工作区目录外写入或删除内容。仅当你有意让 `apply_patch` 操作工作区外的文件时，才设置 `false`。
- `tools.fs.workspaceOnly: true`（可选）：将 `read`/`write`/`edit`/`apply_patch` 路径及原生提示词图像自动加载路径限制在工作区目录内。
- 保持文件系统根目录范围狭窄——避免将主目录等宽泛路径用作智能体/沙箱工作区，因为这可能会向文件系统工具暴露敏感的本地文件（例如 `~/.openclaw` 下的状态/配置）。

## 按智能体配置的访问配置文件（多智能体）

每个智能体都可以拥有自己的沙箱和工具策略：完全访问、只读或禁止访问。有关优先级规则，请参阅[多 Agent 沙盒和工具](/zh-CN/tools/multi-agent-sandbox-tools)。

常见模式：个人智能体（完全访问，不使用沙箱）、家庭/工作智能体（沙箱隔离 + 只读工具）、公共智能体（沙箱隔离 + 无文件系统/Shell 工具）。

### 完全访问（无沙箱）

```json5
{
  agents: {
    list: [
      { id: "personal", workspace: "~/.openclaw/workspace-personal", sandbox: { mode: "off" } },
    ],
  },
}
```

### 只读工具 + 只读工作区

```json5
{
  agents: {
    list: [
      {
        id: "family",
        workspace: "~/.openclaw/workspace-family",
        sandbox: { mode: "all", scope: "agent", workspaceAccess: "ro" },
        tools: {
          allow: ["read"],
          deny: ["write", "edit", "apply_patch", "exec", "process", "browser"],
        },
      },
    ],
  },
}
```

### 无文件系统/Shell 访问权限（允许提供商消息传递）

```json5
{
  agents: {
    list: [
      {
        id: "public",
        workspace: "~/.openclaw/workspace-public",
        sandbox: { mode: "all", scope: "agent", workspaceAccess: "none" },
        tools: {
          // 会话工具可能会暴露转录数据。默认范围为当前会话 + 派生会话；
          // 读取还包括通过环境群组感知监视的同智能体群组。
          // 使用 visibility: "self" 可排除这些受监视的会话。
          sessions: { visibility: "tree" }, // self | tree | agent | all
          allow: [
            "sessions_list",
            "sessions_history",
            "sessions_send",
            "sessions_spawn",
            "session_status",
            "discord",
            "slack",
            "telegram",
            "whatsapp",
          ],
          deny: [
            "apply_patch",
            "browser",
            "canvas",
            "cron",
            "edit",
            "exec",
            "gateway",
            "image",
            "nodes",
            "process",
            "read",
            "write",
          ],
        },
      },
    ],
  },
}
```

## 浏览器控制风险

启用浏览器控制会为模型提供一个真实浏览器。如果该配置文件中已有登录会话，模型便可访问这些账户和数据——应将浏览器配置文件视为敏感状态。

- 优先为智能体使用专用配置文件（默认的 `openclaw` 配置文件）；避免使用你的个人日常配置文件。
- 除非你信任沙箱隔离的智能体，否则应禁用其主机浏览器控制。
- 独立的 local loopback 浏览器控制 API 仅支持共享密钥身份验证（Gateway 网关令牌持有者身份验证或 Gateway 网关密码）——它不会使用受信任代理或 Tailscale Serve 身份标头。
- 将浏览器下载内容视为不受信任的输入；优先使用隔离的下载目录。
- 如有可能，请在智能体配置文件中禁用浏览器同步/密码管理器。
- 对于远程 Gateway 网关，“浏览器控制”等同于对该配置文件可访问的所有内容拥有“操作员访问权限”。
- 仅允许通过 tailnet 访问 Gateway 网关和节点主机；避免将浏览器控制端口暴露给 LAN 或公共互联网。
- 不需要时禁用浏览器代理路由（`gateway.nodes.browser.mode="off"`）。
- Chrome MCP 的现有会话模式并不“更安全”——它可以在该主机 Chrome 配置文件可访问的任何位置以你的身份执行操作。
- 当 Gateway 网关与浏览器不在同一台机器上时，请在浏览器机器上运行**节点主机**，并让 Gateway 网关代理浏览器操作（请参阅[浏览器工具](/zh-CN/tools/browser)）；将节点配对视同管理员访问权限，让 Gateway 网关与节点主机处于同一 tailnet，并避免通过 LAN、公共互联网或 Tailscale Funnel 暴露中继/控制端口。

### 浏览器 SSRF 策略（默认严格）

除非明确选择启用，否则私有/内部目标始终会被阻止。

- 默认：未设置 `browser.ssrfPolicy.dangerouslyAllowPrivateNetwork`，因此私有/内部/特殊用途目标仍会被阻止。仍接受旧版别名 `allowPrivateNetwork`。
- 选择启用：设置 `dangerouslyAllowPrivateNetwork: true` 以允许这些目标。
- 在严格模式下，使用 `hostnameAllowlist`（例如 `*.example.com` 的模式）和 `allowedHostnames`（精确主机例外，包括 `localhost` 等原本会被阻止的名称）配置显式例外。
- 直接导航请求会接受预检。在操作期间及操作后有限的宽限期内，受保护的 Playwright 交互（点击、坐标点击、悬停、拖动、滚动、选择、按键、输入、填写表单和求值）会在发送 HTTP 请求字节之前拦截策略拒绝的顶级和子框架文档加载，然后尽力重新检查最终的 `http(s)` URL。
- 每次全新启动托管 Chrome 前，OpenClaw 都会尽力禁用网络预测，以抑制所观察到的 Chromium 针对这些被拒绝加载的推测性预连接。这是纵深防御，而非策略边界：跨控制服务重启复用的浏览器以及其他浏览器后端可能不会采用相同的加固措施。页面路由仍是请求级拦截，而不是网络防火墙：重定向跳转、弹出窗口的首次请求、Service Worker 流量、有限保护窗口结束后运行的页面代码，以及某些后台/子资源路径都可能绕过它。最终 URL 检查仍属于检测/隔离防御；要实现完全阻止，需要所有者侧出口隔离或强制实施策略的代理。

```json5
{
  browser: {
    ssrfPolicy: {
      dangerouslyAllowPrivateNetwork: false,
      hostnameAllowlist: ["*.example.com", "example.com"],
      allowedHostnames: ["localhost"],
    },
  },
}
```

## 网络暴露

### 绑定、端口、防火墙

Gateway 网关在同一端口上多路复用 WebSocket + HTTP（默认为 `18789`；配置/标志/环境变量：`gateway.port`、`--port`、`OPENCLAW_GATEWAY_PORT`）。该 HTTP 接口包括 Control UI（SPA 资源，默认基础路径为 `/`）和 canvas 主机（`/__openclaw__/canvas` 和 `/__openclaw__/a2ui`——任意 HTML/JS；在普通浏览器中加载时应视为不受信任的内容；不要将其暴露给不受信任的网络/用户，也不要让其与具有特权的 Web 接口共享来源）。

`gateway.bind` 控制 Gateway 网关的监听位置：

- `"loopback"`（默认）：仅本地客户端可以连接。
- `"lan"`、`"tailnet"`、`"custom"`：会扩大攻击面。仅可在启用 Gateway 网关身份验证（共享令牌/密码或正确配置的受信任代理）和真正防火墙的情况下使用。

经验法则：优先使用 Tailscale Serve，而不是绑定 LAN（Serve 会让 Gateway 网关继续监听回环地址，并由 Tailscale 处理访问）；如果必须绑定 LAN，请使用防火墙将端口限制为严格的源 IP 允许列表，而不要进行宽泛的端口转发；绝不要在 `0.0.0.0` 上未经身份验证地暴露 Gateway 网关。

### 使用 UFW 发布 Docker 端口

已发布的容器端口（`-p HOST:CONTAINER` 或 Compose `ports:`）通过 Docker 的转发链路由，而不只通过主机 `INPUT` 规则。请在 `DOCKER-USER` 中强制实施规则（其求值先于 Docker 自身的接受规则）；大多数现代发行版使用 `iptables-nft` 前端，该前端仍会将这些规则应用于 nftables 后端。

```bash
# /etc/ufw/after.rules（作为独立的 *filter 节追加）
*filter
:DOCKER-USER - [0:0]
-A DOCKER-USER -m conntrack --ctstate ESTABLISHED,RELATED -j RETURN
-A DOCKER-USER -s 127.0.0.0/8 -j RETURN
-A DOCKER-USER -s 10.0.0.0/8 -j RETURN
-A DOCKER-USER -s 172.16.0.0/12 -j RETURN
-A DOCKER-USER -s 192.168.0.0/16 -j RETURN
-A DOCKER-USER -s 100.64.0.0/10 -j RETURN
-A DOCKER-USER -p tcp --dport 80 -j RETURN
-A DOCKER-USER -p tcp --dport 443 -j RETURN
-A DOCKER-USER -m conntrack --ctstate NEW -j DROP
-A DOCKER-USER -j RETURN
COMMIT
```

IPv6 使用单独的表——如果启用了 Docker IPv6，请在 `/etc/ufw/after6.rules` 中添加相匹配的策略。避免硬编码接口名称（`eth0`），因为不同 VPS 镜像中的名称各不相同（`ens3`、`enp*` 等），名称不匹配可能导致拒绝规则被静默跳过。

```bash
ufw reload
iptables -S DOCKER-USER
ip6tables -S DOCKER-USER
nmap -sT -p 1-65535 <public-ip> --open
```

预期的外部端口应仅包含你有意暴露的端口（对于大多数设置：SSH + 反向代理端口）。

### mDNS/Bonjour 设备发现

启用内置 `bonjour` 插件后，Gateway 网关会通过 mDNS（`_openclaw-gw._tcp`，端口 5353）广播在线状态，以供本地设备发现。完整模式包含会暴露运行信息的 TXT 记录：`cliPath`（会泄露用户名和安装位置的文件系统路径）、`sshPort`（宣告 SSH 可用）、`displayName`/`lanHost`（主机名信息）。广播基础设施详细信息会使 LAN 侦察更加容易。

- 除非需要 LAN 设备发现，否则请禁用 Bonjour——它会在 macOS 主机上自动启动，在其他系统上则需主动启用；使用直接 Gateway 网关 URL、Tailnet、SSH 或广域 DNS-SD 可以避免本地多播。
- **最小模式**（启用 Bonjour 时的默认模式，建议用于对外暴露的 Gateway 网关）会省略敏感字段：

  ```json5
  { discovery: { mdns: { mode: "minimal" } } }
  ```

- **关闭**会在保持插件启用的同时禁止本地设备发现：

  ```json5
  { discovery: { mdns: { mode: "off" } } }
  ```

- **完整模式**（主动启用）包含 `cliPath` + `sshPort`：

  ```json5
  { discovery: { mdns: { mode: "full" } } }
  ```

- 也可以设置 `OPENCLAW_DISABLE_BONJOUR=1`，无需更改配置即可禁用 mDNS。

在最小模式下，Gateway 网关会广播 `role`、`gatewayPort`、`transport`，但省略 `cliPath`/`sshPort`；需要 CLI 路径的应用可以改为通过经过身份验证的 WebSocket 连接获取它。

### Gateway 网关 WebSocket 身份验证

默认要求 Gateway 网关身份验证——如果未配置有效的身份验证路径，Gateway 网关会拒绝 WebSocket 连接（故障时关闭）。新手引导默认会生成令牌（即使使用回环地址），因此本地客户端也必须进行身份验证。

```json5
{ gateway: { auth: { mode: "token", token: "your-token" } } }
```

`openclaw doctor --generate-gateway-token` 可以为你生成令牌。

<Note>
`gateway.remote.token` 和 `gateway.remote.password` 是客户端凭据来源——它们本身并不能保护本地 WS 访问。本地调用路径仅在未设置 `gateway.auth.*` 时使用 `gateway.remote.*` 作为回退。如果通过 SecretRef 显式配置了 `gateway.auth.token` 或 `gateway.auth.password`，但无法解析，则解析会以关闭方式失败（不会通过远程回退掩盖问题）。
</Note>

使用 `wss://` 时，通过 `gateway.remote.tlsFingerprint` 固定远程 TLS。明文 `ws://` 可用于环回、私有 IP 字面量、`.local` 和 Tailnet `*.ts.net` Gateway 网关 URL；对于其他受信任的私有 DNS 名称，请在客户端进程上设置 `OPENCLAW_ALLOW_INSECURE_PRIVATE_WS=1` 作为紧急解锁措施（仅限进程环境，而不是 `openclaw.json` 键）。移动端配对以及 Android 手动输入/扫描的 Gateway 网关路由更为严格：仅环回可使用明文，而私有 LAN、链路本地地址、`.local` 和无点主机名必须使用 TLS，除非你明确选择使用受信任私有网络明文路径。

对于直接本地环回连接，设备配对会自动获批（此外，还包括一条范围很窄的后端/容器本地自连接路径，用于受信任的共享密钥辅助程序流程）；Tailnet 和 LAN 连接（包括连接到 Tailnet 地址的同主机连接）会被视为远程连接，仍需审批。解析后的 `tailnet` 地址，或除 `127.0.0.1` 和 `0.0.0.0` 之外的 `custom` 地址，会添加单独的 `127.0.0.1` 监听器；只有连接到该本地监听器的连接才具有环回语义。环回请求中存在转发标头证据时，该请求不再被视为本地环回；元数据升级的自动审批范围受到严格限制。请参阅 [Gateway 网关配对](/zh-CN/gateway/pairing)。

身份验证模式：

- `"token"`：共享 Bearer 令牌（推荐用于大多数设置）。
- `"password"`：建议通过 `OPENCLAW_GATEWAY_PASSWORD` 设置。
- `"trusted-proxy"`：信任具备身份感知能力的反向代理，由其对用户进行身份验证并通过标头传递身份。请参阅[受信任代理身份验证](/zh-CN/gateway/trusted-proxy-auth)。

轮换检查清单（令牌/密码）：生成/设置新密钥（`gateway.auth.token` 或 `OPENCLAW_GATEWAY_PASSWORD`）；重启 Gateway 网关（如果由 macOS 应用监管 Gateway 网关，则重启该应用）；更新远程客户端（`gateway.remote.token`/`.password`）；验证旧凭据已无法使用。

### Tailscale Serve 身份标头

当 `gateway.auth.allowTailscale` 为 `true`（Serve 的默认值）时，OpenClaw 接受 Tailscale Serve 身份标头 `tailscale-user-login`，用于 Control UI/WebSocket 身份验证。它通过本地 Tailscale 守护进程（`tailscale whois`）解析 `x-forwarded-for` 地址，并将其与标头匹配来验证身份——仅当环回请求携带由 Tailscale 注入的 `x-forwarded-for`、`x-forwarded-proto` 和 `x-forwarded-host` 时，才会触发此流程。对于此异步检查，在限流器记录失败之前，来自同一 `{scope, ip}` 的失败尝试会被串行处理，因此来自同一个 Serve 客户端的并发错误重试可能会立即锁定第二次尝试。

HTTP API 端点（`/v1/*`、`/tools/invoke`、`/api/channels/*`）不使用 Tailscale 身份标头身份验证——它们遵循 Gateway 网关已配置的 HTTP 身份验证模式。

Gateway 网关 HTTP Bearer 身份验证实际上提供全有或全无的操作员访问权限。能够调用 `/v1/chat/completions`、`/v1/responses`、`/api/v1/admin/rpc` 等插件路由或 `/api/channels/*` 的凭据，都是该 Gateway 网关的完全访问权限操作员密钥：共享密钥 Bearer 身份验证会恢复完整的默认操作员权限范围（`operator.admin`、`operator.approvals`、`operator.pairing`、`operator.read`、`operator.talk.secrets`、`operator.write`）以及智能体轮次的所有者语义，而范围更窄的 `x-openclaw-scopes` 值不会限制该共享密钥路径。仅当请求来自携带身份的模式（受信任代理身份验证）或明确无身份验证的私有入口时，按请求设置的权限范围语义才适用；在这些模式下，省略 `x-openclaw-scopes` 时会回退到常规的默认操作员权限范围集合，而在权限范围缩窄时，`x-openclaw-model` 等所有者级标头需要 `operator.admin`。`/tools/invoke` 和 HTTP 会话历史记录端点遵循相同的共享密钥规则。不要与不受信任的调用方共享这些凭据；最好为每个信任边界使用单独的 Gateway 网关。

无令牌 Serve 身份验证假定 Gateway 网关主机本身可信——它无法防御同一主机上的恶意进程。如果 Gateway 网关主机上可能运行不受信任的本地代码，请禁用 `allowTailscale`，并要求显式共享密钥身份验证（`token` 或 `password`）。

不要从你自己的反向代理转发这些标头。如果你在 Gateway 网关前终止 TLS 或使用代理，请禁用 `allowTailscale`，改用共享密钥身份验证或[受信任代理身份验证](/zh-CN/gateway/trusted-proxy-auth)。

请参阅 [Tailscale](/zh-CN/gateway/tailscale) 和 [Web 概览](/zh-CN/web)。

### 反向代理配置

在 nginx/Caddy/Traefik 等之后部署时，设置 `gateway.trustedProxies` 以正确处理转发的客户端 IP。当 Gateway 网关检测到来自**不在** `trustedProxies` 中的地址所发送的代理标头时，它不会将连接视为本地连接；如果已禁用 Gateway 网关身份验证，该连接会被拒绝。这可以防止代理连接伪装成来自 localhost 并获得自动信任。

`trustedProxies` 还会供要求更严格的 `gateway.auth.mode: "trusted-proxy"` 使用：默认情况下，它会对环回来源的代理以关闭方式失败。同主机环回反向代理可以使用 `trustedProxies` 进行本地客户端检测和转发 IP 处理，但仅当 `gateway.auth.trustedProxy.allowLoopback = true` 时才能满足 `trusted-proxy` 身份验证模式；否则，请使用令牌/密码身份验证。

```yaml
gateway:
  trustedProxies:
    - "10.0.0.1" # 反向代理 IP
  allowRealIpFallback: false # 默认为 false；仅当代理无法提供 X-Forwarded-For 时启用
  auth:
    mode: password
    password: ${OPENCLAW_GATEWAY_PASSWORD}
```

设置 `trustedProxies` 后，Gateway 网关使用 `X-Forwarded-For` 确定客户端 IP；除非显式设置 `gateway.allowRealIpFallback: true`，否则会忽略 `X-Real-IP`。确保你的代理**覆盖** `X-Forwarded-For`/`X-Real-IP`，而不是向其中追加内容：

```nginx
# 正确
proxy_set_header X-Forwarded-For $remote_addr;
proxy_set_header X-Real-IP $remote_addr;

# 错误：保留/追加客户端提供的不受信任值
proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
```

受信任代理标头不会让节点设备配对自动获得信任——`gateway.nodes.pairing.autoApproveCidrs` 是一项单独且默认禁用的操作员策略；即使启用了环回受信任代理身份验证，来自环回来源的受信任代理标头路径仍不会触发节点自动审批（因为本地调用方可以伪造这些标头）。

### HSTS 和来源说明

- OpenClaw 的 Gateway 网关优先用于本地/环回。如果在反向代理处终止 TLS，请在那里设置 HSTS。
- 如果由 Gateway 网关本身终止 HTTPS，`gateway.http.securityHeaders.strictTransportSecurity` 会让 OpenClaw 响应发出 HSTS 标头。
- 默认情况下，非环回 Control UI 部署需要 `gateway.controlUi.allowedOrigins`；`allowedOrigins: ["*"]` 是显式的全部允许策略，并非经过加固的默认设置——除非是在严格受控的本地测试中，否则请避免使用。
- 即使启用了常规环回豁免，环回上的浏览器来源身份验证失败仍会受到速率限制，但锁定键会按规范化后的 `Origin` 值分别划分，而不是共用一个 localhost 存储桶。
- `gateway.controlUi.dangerouslyAllowHostHeaderOriginFallback=true` 会启用 Host 标头来源回退模式；应将其视为由操作员选择的危险策略。
- 将 DNS 重绑定和代理 Host 标头行为视为部署加固事项；严格限制 `trustedProxies`，并避免将 Gateway 网关直接暴露到公共互联网。
- 详细部署指南：[受信任代理身份验证](/zh-CN/gateway/trusted-proxy-auth#tls-termination-and-hsts)。

### 通过 HTTP 使用 Control UI

Control UI 需要安全上下文（HTTPS 或 localhost）才能生成设备身份。

- `gateway.controlUi.allowInsecureAuth`：本地兼容性开关。在 localhost 上，当页面通过不安全的 HTTP 加载时，允许 Control UI 在没有设备身份的情况下进行身份验证。不会绕过配对检查，也不会放宽远程（非 localhost）设备身份要求。建议使用 HTTPS（Tailscale Serve），或在 `127.0.0.1` 上打开 UI。
- `gateway.controlUi.dangerouslyDisableDeviceAuth`：已停用的紧急解锁输入。旧配置会保留经过身份验证且仅限配对的 Control UI 访问权限，以便进行修复，直到在 HTTPS 或 localhost 上重新打开浏览器并完成有界且显式的自配对迁移；不要将其添加到当前配置中。
- 除这些标志外，成功的 `gateway.auth.mode: "trusted-proxy"` 可以允许没有设备身份的**操作员** Control UI 会话进入——这是有意设计的身份验证模式行为，并非 `allowInsecureAuth` 捷径，且不适用于节点角色的 Control UI 会话。

启用 `allowInsecureAuth` 时，`openclaw security audit` 会发出警告。

### 不安全/危险标志

对于每个已启用且已知不安全/危险的调试开关，`openclaw security audit` 都会引发一项 `config.insecure_or_dangerous_flags`（每个标志一项发现）。在生产环境中不要设置这些标志。如果配置了审计抑制，即使匹配的发现移至 `suppressedFindings`，`security.audit.suppressions.active` 仍会保留在活动输出中。

<AccordionGroup>
  <Accordion title="当前审计跟踪的标志">
    - `gateway.controlUi.allowInsecureAuth=true`
    - `gateway.controlUi.dangerouslyAllowHostHeaderOriginFallback=true`
    - 从已停用的 `gateway.controlUi.dangerouslyDisableDeviceAuth=true` 导入的待处理 Control UI 设备身份验证迁移
    - `security.audit.suppressions configured (<count>)`
    - `hooks.gmail.allowUnsafeExternalContent=true`
    - `hooks.mappings[<index>].allowUnsafeExternalContent=true`
    - `tools.exec.applyPatch.workspaceOnly=false`
    - `plugins.entries.acpx.config.permissionMode=approve-all`

  </Accordion>

  <Accordion title="配置架构中的所有 dangerous*/dangerously* 键">
    Control UI 和浏览器：
    - `gateway.controlUi.dangerouslyAllowHostHeaderOriginFallback`
    - `gateway.controlUi.dangerouslyDisableDeviceAuth`（已停用的升级输入）
    - `browser.ssrfPolicy.dangerouslyAllowPrivateNetwork`

    渠道名称匹配（内置渠道和插件渠道；适用时也包括每个 `accounts.<accountId>`）：
    - `channels.discord.dangerouslyAllowNameMatching`
    - `channels.googlechat.dangerouslyAllowNameMatching`
    - `channels.msteams.dangerouslyAllowNameMatching`
    - `channels.slack.dangerouslyAllowNameMatching`
    - `channels.irc.dangerouslyAllowNameMatching`（插件渠道）
    - `channels.mattermost.dangerouslyAllowNameMatching`（插件渠道）
    - `channels.synology-chat.dangerouslyAllowNameMatching`（插件渠道）
    - `channels.synology-chat.dangerouslyAllowInheritedWebhookPath`（插件渠道）
    - `channels.zalouser.dangerouslyAllowNameMatching`（插件渠道）

    网络暴露：
    - `channels.telegram.network.dangerouslyAllowPrivateNetwork`（也可按账户设置）

    沙箱 Docker（默认值 + 按智能体设置）：
    - `agents.defaults.sandbox.docker.dangerouslyAllowReservedContainerTargets`
    - `agents.defaults.sandbox.docker.dangerouslyAllowExternalBindSources`
    - `agents.defaults.sandbox.docker.dangerouslyAllowContainerNamespaceJoin`

  </Accordion>
</AccordionGroup>

## 部署和主机信任

- Gateway 网关主机应启用全盘加密；如果主机由多人共享，建议为 Gateway 网关使用专用的操作系统用户账户。
- 已发布软件包的依赖锁定：源代码检出使用 `pnpm-lock.yaml`；已发布的 `openclaw` npm 软件包和 OpenClaw 自有的 npm 插件软件包包含 `npm-shrinkwrap.json`，因此安装时使用发布版本中经过审核的传递依赖关系图，而不是在安装时重新解析依赖关系图。这是供应链加固和发布可复现性的边界，并非沙箱——请参阅 [npm shrinkwrap](/zh-CN/gateway/security/shrinkwrap)。
- 安全文件操作：OpenClaw 使用 `@openclaw/fs-safe` 实现受根目录边界约束的文件访问、原子写入、归档提取、临时工作区和机密文件辅助功能。可选的 POSIX Python 辅助程序默认**关闭**；仅当需要额外的基于文件描述符相对路径的变更加固，并且能够支持 Python 运行时时，才设置 `OPENCLAW_FS_SAFE_PYTHON_MODE=auto` 或 `require`。详情请参阅：[安全文件操作](/zh-CN/gateway/security/secure-file-operations)。
- 共享 Slack 工作区的风险：如果 Slack 中的所有人都能向 Bot 发送消息，核心风险在于委托的工具权限——任何获准的发送者都可以在智能体策略范围内触发工具调用（`exec`、浏览器、网络/文件工具），来自某个发送者的提示词/内容注入可能会影响共享状态、设备和输出；如果共享智能体拥有敏感凭据或文件，任何获准的发送者都可能通过使用工具来实施数据外泄。团队工作流应使用仅配备必要工具的独立智能体/Gateway 网关；包含个人数据的智能体应保持私有。
- 公司共享智能体（可接受的模式）：如果使用智能体的所有人都处于同一信任边界内（例如同一个公司团队），并且智能体严格限定于业务用途，则可以采用此模式。应在专用计算机、虚拟机或容器中运行它，使用专用的操作系统用户以及专用的浏览器、配置文件和账户，并且不要让该运行时登录个人 Apple/Google 账户，也不要使用个人密码管理器或浏览器配置文件。在同一运行时中混用个人身份和公司身份会破坏这种隔离，并增加个人数据暴露的风险。

## 磁盘上的机密信息

假定 `~/.openclaw/`（或 `$OPENCLAW_STATE_DIR/`）下的任何内容都可能包含机密信息或私有数据：

| 路径                                           | 内容                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                          |
| ---------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `openclaw.json`                                | 配置可能包含令牌（Gateway 网关、远程 Gateway 网关）、提供商设置和允许列表。                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                           |
| `credentials/**`                               | 渠道凭据（例如 WhatsApp 凭据）、配对允许列表、旧版 OAuth 导入。                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                       |
| `state/openclaw.sqlite`                        | 共享运行时状态，包括原生 MCP OAuth 访问/刷新令牌、动态客户端注册密钥和设备发现状态。                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                 |
| `agents/<agentId>/agent/openclaw-agent.sqlite` | 每个智能体的运行时状态，包括模型身份验证配置文件。                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                           |
| `agents/<agentId>/agent/auth-profiles.json`    | 旧版模型身份验证迁移源；Doctor 将受支持的记录导入每个智能体的 SQLite 数据库。                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                          |
| `agents/<agentId>/agent/codex-home/**`         | 每个智能体的 Codex app-server 账户、配置、Skills、插件、原生线程状态和诊断（默认）。                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                          |
| `$CODEX_HOME/**` 或 `~/.codex/**`              | 原生 Codex 运行时状态。普通 Codex harness 仅在显式设置 `plugins.entries.codex.config.appServer.homeScope: "user"` 时访问该状态。独立的监管连接会在其解析后的主目录作用域为 `"user"` 时访问该状态；未设置时，这是 stdio 或 Unix 的默认值。其中包含原生 Codex 账户、配置、插件和线程存储。监管会列出源元数据，并在该连接上保留续接 Chat 的规范原生分支及后续轮次；分支操作会将有限的持久化用户和助手历史记录复制到经过身份验证且锁定模型的 OpenClaw Chat 中。仅为由所有者控制的 Gateway 网关启用此功能。请参阅 [Codex harness](/zh-CN/plugins/codex-harness#share-threads-with-codex-desktop-and-cli) 和 [Codex 监管](/zh-CN/plugins/codex-supervision)。 |
| `secrets.json`（可选）                      | `file` SecretRef 提供商使用的文件后端密钥载荷（`secrets.providers`）。                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                              |
| `agents/<agentId>/agent/auth.json`             | 旧版兼容文件；发现静态 `api_key` 条目时会将其清除。                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                 |
| `agents/<agentId>/agent/openclaw-agent.sqlite` | 每个智能体的运行时状态，包括可能包含私信和工具输出的会话行及转录记录。                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                |
| `agents/<agentId>/sessions/**`                 | 旧版会话迁移源和归档，其中可能包含私信和工具输出。                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                  |
| 内置插件包                        | 已安装的插件（及其 `node_modules/`）。                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                   |
| `sandboxes/**`                                 | 工具沙箱工作区；可能会积累在沙箱内读取或写入的文件副本。                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                          |

### 凭据存储位置图

也可用于帮助制定备份决策：

- WhatsApp：`~/.openclaw/credentials/whatsapp/<accountId>/creds.json`
- Telegram Bot 令牌：配置/环境变量或 `channels.telegram.tokenFile`（仅限常规文件；拒绝符号链接）
- Discord Bot 令牌：配置/环境变量或 SecretRef（环境变量/文件/exec 提供商）
- Slack 令牌：配置/环境变量（`channels.slack.*`）
- 配对允许列表：`~/.openclaw/credentials/<channel>-allowFrom.json`（默认账户）/ `<channel>-<accountId>-allowFrom.json`（非默认账户）
- 模型身份验证配置文件：`~/.openclaw/agents/<agentId>/agent/openclaw-agent.sqlite`（`auth_profile_store`）
- MCP OAuth 会话：`~/.openclaw/state/openclaw.sqlite`（`mcp_oauth_stores`）
- 旧版 OAuth 导入：`~/.openclaw/credentials/oauth.json`

安全加固：严格限制权限（目录使用 `700`，文件使用 `600`）；在 Gateway 网关主机上使用全盘加密；如果主机由多人共享，优先使用专用的操作系统用户账户。

### 文件权限

- `~/.openclaw/openclaw.json`：`600`（仅允许用户读写）
- `~/.openclaw`：`700`（仅限用户）

`openclaw doctor` 可以发出警告，并提供收紧这些权限的选项。

### 工作区 `.env` 文件

OpenClaw 会为智能体和工具加载工作区本地的 `.env` 文件，但绝不允许它们静默覆盖 Gateway 网关运行时控制：

- 不受信任的工作区 `.env` 文件不得设置提供商凭据环境变量，例如 `GEMINI_API_KEY`、`GOOGLE_API_KEY`、`XAI_API_KEY`、`MISTRAL_API_KEY`、`GROQ_API_KEY`、`DEEPSEEK_API_KEY`、`PERPLEXITY_API_KEY`、`BRAVE_API_KEY`、`TAVILY_API_KEY`、`EXA_API_KEY`、`FIRECRAWL_API_KEY`，以及已安装的受信任插件声明的提供商身份验证键。请改为将提供商凭据放入 Gateway 网关进程环境、`~/.openclaw/.env`（`$OPENCLAW_STATE_DIR/.env`）、配置的 `env` 块或可选的登录 shell 导入中。
- 不受信任的工作区 `.env` 文件不得设置任何以 `OPENCLAW_` 开头的键，从而保留整个运行时命名空间。这样，未来的 `OPENCLAW_*` 控制默认采用故障关闭策略，而不会从已提交或攻击者提供的 `.env` 内容中被静默继承。
- 工作区 `.env` 也不得覆盖渠道和提供商的端点路由设置（例如 `MATRIX_HOMESERVER`、`MATTERMOST_URL`、`IRC_HOST`、`SYNOLOGY_CHAT_INCOMING_URL`、`AZURE_SPEECH_ENDPOINT`，以及其他以 `_ENDPOINT` 结尾的键），因此克隆的工作区无法通过本地端点配置重定向内置连接器流量。这些设置必须来自 Gateway 网关进程环境、全局运行时 dotenv、显式配置或 `env.shellEnv`。
- 受信任的进程/操作系统环境变量、全局运行时 dotenv、配置 `env` 和已启用的登录 shell 导入仍然有效——此限制仅适用于工作区 `.env` 文件的加载。

工作区 `.env` 文件通常与智能体代码放在一起，可能被意外提交或由工具写入；禁止其中设置提供商凭据，可防止克隆的工作区替换为攻击者控制的提供商账户。

### 日志和转录记录

OpenClaw 将会话转录记录存储在磁盘的 `~/.openclaw/agents/<agentId>/sessions/*.jsonl` 下，以保持会话连续性并支持可选的记忆索引——任何拥有文件系统访问权限的进程/用户都可以读取这些记录。请将磁盘访问视为信任边界，并严格限制 `~/.openclaw` 的权限；如需更强的隔离，请在不同的操作系统用户或主机下运行智能体。

Gateway 网关日志可能包含工具摘要、错误和 URL；会话转录记录可能包含粘贴的机密、文件内容、命令输出和链接。

- 保持日志/转录记录脱敏功能开启（`logging.redactSensitive: "tools"`，默认值）。
- 通过 `logging.redactPatterns` 添加适用于你的环境的自定义模式（令牌、主机名、内部 URL）。
- 共享诊断信息时，优先使用 `openclaw status --all`（可直接粘贴，机密已脱敏），而不是原始日志。
- 如果不需要长期保留，请清理旧的会话转录记录和日志文件。

详情：[日志](/zh-CN/gateway/logging)

## 安全基线（复制/粘贴）

```json5
{
  gateway: {
    mode: "local",
    bind: "loopback",
    port: 18789,
    auth: { mode: "token", token: "your-long-random-token" },
  },
  channels: {
    whatsapp: {
      dmPolicy: "pairing",
      groups: { "*": { requireMention: true } },
    },
  },
}
```

这会使 Gateway 网关保持私有、要求私信配对，并避免群组 Bot 始终处于活动状态。如需提高工具执行的安全性，还应为所有非所有者智能体添加沙箱并拒绝危险工具（参见上文“按智能体配置的访问权限配置文件”）。

### 使用独立号码（WhatsApp、Signal、Telegram）

对于基于电话号码的渠道，建议让助手使用与你个人号码不同的独立号码，以确保个人对话保持私密，并让 Bot 号码在其自身边界内处理自动化任务。

## 事件响应

### 遏制

1. 停止运行：停止 macOS 应用（如果它负责监管 Gateway 网关），或终止你的 `openclaw gateway` 进程。
2. 关闭暴露：在查明事件原因之前，设置 `gateway.bind: "loopback"`（或禁用 Tailscale Funnel/Serve）。
3. 冻结访问：将高风险私信/群组切换为 `dmPolicy: "disabled"` / 要求提及，并移除所有 `"*"` 全部允许条目。

### 轮换（如果机密泄露，应假定已遭入侵）

1. 轮换 Gateway 网关身份验证凭据（`gateway.auth.token` / `OPENCLAW_GATEWAY_PASSWORD`）并重启。
2. 在任何能够调用 Gateway 网关的机器上轮换远程客户端机密（`gateway.remote.token` / `.password`）。
3. 轮换提供商/API 凭据（WhatsApp 凭据、Slack/Discord 令牌、`auth-profiles.json` 中的模型/API 密钥，以及使用加密机密载荷时其中的值）。

### 审计

1. 使用 `openclaw logs` 检查 Gateway 网关日志（对于命名配置文件，使用 `openclaw --profile <profile> logs`）。默认路径为 `/tmp/openclaw/openclaw-YYYY-MM-DD.log`；除非 `logging.file` 覆盖，否则命名配置文件使用 `/tmp/openclaw/openclaw-<profile>-YYYY-MM-DD.log`。
2. 检查相关转录记录：`~/.openclaw/agents/<agentId>/sessions/*.jsonl`。
3. 检查近期可能扩大访问范围的配置更改：`gateway.bind`、`gateway.auth`、私信/群组策略、`tools.elevated`、插件更改。
4. 重新运行 `openclaw security audit --deep`，并确认严重问题均已解决。

### 收集报告所需信息

- 时间戳、Gateway 网关主机操作系统和 OpenClaw 版本。
- 会话转录记录和一小段日志末尾内容（完成脱敏后）。
- 攻击者发送的内容以及智能体执行的操作。
- Gateway 网关是否暴露在 local loopback 之外（LAN/Tailscale Funnel/Serve）。

## 机密扫描

CI 会对仓库运行预提交 `detect-private-key` 钩子。如果该检查失败，请移除或轮换已提交的密钥材料，然后在本地复现：

```bash
pre-commit run --all-files detect-private-key
```

## 报告安全问题

发现 OpenClaw 中的漏洞？请以负责任的方式报告：

1. 电子邮件：[security@openclaw.ai](mailto:security@openclaw.ai)
2. 在修复之前，请勿公开发布。
3. 我们会向你致谢（除非你希望匿名）。
