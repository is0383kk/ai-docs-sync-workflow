---
read_when:
    - 将 OpenClaw 连接到 ClickClack 工作区
    - 测试 ClickClack Bot 身份
summary: ClickClack Bot Token 渠道设置和目标语法
title: ClickClack
x-i18n:
    generated_at: "2026-07-26T06:06:00Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 761538cdd7a916415719131b9ff2f40bf3e3e0eab0f7bda450250886acde8a64
    source_path: channels/clickclack.md
    workflow: 16
---

ClickClack 通过一等支持的 ClickClack Bot 令牌，将 OpenClaw 连接到自托管的 ClickClack 工作区。

当你希望 OpenClaw 智能体以 ClickClack Bot 用户身份出现时，请使用此方式。ClickClack 支持独立服务 Bot 和用户所有的 Bot；用户所有的 Bot 会保留 `owner_user_id`，并且仅获得你授予的令牌权限范围。

## 快速设置

在 ClickClack 中，打开 **Workspace settings → Integrations → OpenClaw**，使用 **Setup code (recommended)** 创建
Bot，然后复制生成的命令：

```bash
openclaw channels add clickclack --code 'https://clickclack.example.com/#XXXX-XXXX-XXXX'
```

对于前端与 API 使用不同来源或 API 挂载在路径下的情况，ClickClack 会改为生成
精确的申领端点：

```bash
openclaw channels add clickclack --code 'https://api.example.com/services/clickclack/api/bot-setup-codes/claim#XXXX-XXXX-XXXX'
```

设置码只能使用一次，并会在 10 分钟后过期。OpenClaw 会申领该设置码，
接收新生成的 Bot 令牌和工作区设置，保存账户，
验证连接，并报告正在运行的 Gateway 网关是否已加载该账户。
对于带版本的精确端点，OpenClaw 会验证并保存 ClickClack 返回的规范 API
基础地址，包括所有路径前缀。设置码本身
不会存储在 OpenClaw 配置中。

公共服务器上的设置码申领使用 HTTPS。对于
`localhost` 和 `127.0.0.1` 等环回地址上的本地安装，也支持普通 HTTP。

如果 OpenClaw 已在运行，ClickClack 会自动连接，无需执行第二条
命令。否则，请使用以下命令启动：

```bash
openclaw gateway
```

也可以将设置码与服务器 URL 分开传递：

```bash
openclaw channels add clickclack --code XXXX-XXXX-XXXX --base-url https://clickclack.example.com
```

如需引导式设置，请运行：

```bash
openclaw onboard
```

选择 ClickClack，然后在出现提示时输入服务器 URL、Bot 令牌和工作区。
引导式设置会在保存后检查服务器、令牌和工作区；检查
失败不会丢弃配置。

### 替代方式：手动令牌

配置非 OpenClaw 客户端，或明确需要自行管理令牌时，
请在 ClickClack 中选择 **Manual token**：

```bash
openclaw channels add clickclack --base-url https://clickclack.example.com --token ccb_... --workspace default
```

`workspace` 接受工作区 ID（`wsp_...`）、别名或显示名称。
`--code` 不能与 `--token`、`--token-file` 或 `--use-env` 组合使用。

### 替代方式：基于环境变量的令牌

默认账户可以读取 `CLICKCLACK_BOT_TOKEN`，而不在
配置中存储令牌：

```bash
export CLICKCLACK_BOT_TOKEN="ccb_..."
openclaw channels add clickclack --base-url https://clickclack.example.com --workspace default --use-env
openclaw gateway
```

命名账户必须使用已配置的令牌或令牌文件；共享
环境变量被有意限制为仅供默认账户使用。

### JSON5 参考

等效的配置结构为：

```json5
{
  channels: {
    clickclack: {
      enabled: true,
      baseUrl: "https://clickclack.example.com",
      token: { source: "env", provider: "default", id: "CLICKCLACK_BOT_TOKEN" },
      workspace: "default",
      defaultTo: "channel:general",
    },
  },
}
```

只有同时设置 `baseUrl`、令牌来源和
`workspace` 时，账户才会被视为已配置。对于默认账户，令牌来源可以是 `token`、`tokenFile` 或
`CLICKCLACK_BOT_TOKEN`。`workspace` 接受工作区
ID（`wsp_...`）、别名或名称；Gateway 网关会在启动时将其解析为 ID。

### 账户配置键

| 键                      | 默认值              | 说明                                                                                    |
| ----------------------- | ------------------- | --------------------------------------------------------------------------------------- |
| `baseUrl`               | 无（必填）          | 用于面向浏览器的链接的公开 ClickClack URL。                                             |
| `apiBaseUrl`            | `baseUrl`           | 用于 REST 和实时 WebSocket 流量的可选服务器间端点。                                     |
| `token`                 | 无                  | 纯字符串或机密引用（`source: "env" \| "file" \| "exec"`）形式的 Bot 令牌。        |
| `tokenFile`             | 无                  | Bot 令牌文件的路径；优先级高于 `token`。                                |
| `workspace`             | 无（必填）          | 工作区 ID、别名或名称。                                                                 |
| `replyMode`             | `"agent"`           | `"agent"` 运行完整的智能体流水线；`"model"` 发送简短的直接模型补全。 |
| `defaultTo`             | `"channel:general"` | 出站路径未给出目标时使用的目标。                                                        |
| `allowFrom`             | `["*"]`             | 用于入站私信和频道消息的用户 ID 允许列表。                                              |
| `botUserId`             | 自动检测            | 启动时根据 Bot 令牌身份解析。                                                           |
| `agentId`               | 路由默认值          | 将此账户的入站消息固定到一个智能体。                                                    |
| `toolsAllow`            | 无                  | 此账户的智能体回复可使用的工具允许列表。                                                |
| `model`、`systemPrompt` | 无                  | 由 `replyMode: "model"` 补全使用。                                               |
| `commandMenu`           | `true`              | 将原生命令发布到 ClickClack 编辑器自动补全。                                            |
| `reconnectMs`           | `1500`              | 实时连接的重连延迟（100 到 60000）。                                                    |
| `discussions`           | 已禁用              | 托管的按会话频道设置；请参阅[会话讨论](#session-discussions)。                          |

### 保留受身份验证保护的公共主机名

当 ClickClack 和 OpenClaw Gateway 网关运行在同一主机上，
但公开的 ClickClack 主机名受 Cloudflare Access 等身份验证网关保护时，请使用 `apiBaseUrl`：

```json5
{
  channels: {
    clickclack: {
      baseUrl: "https://clack.openclaw.ai",
      apiBaseUrl: "http://127.0.0.1:8484",
      token: { source: "env", provider: "default", id: "CLICKCLACK_BOT_TOKEN" },
      workspace: "default",
    },
  },
}
```

面向浏览器用户的公共主机名可以继续完全受身份验证保护。OpenClaw
使用环回端点处理 REST 请求、设置验证和
实时 WebSocket，而讨论中的 `embedUrl` 和 `openUrl` 链接仍继续
使用公开的 `baseUrl`。如果省略 `apiBaseUrl`，所有流量都使用
`baseUrl`，从而保留现有行为。

如果 `plugins.allow` 是一个非空的限制性列表，则在频道设置中明确选择
ClickClack 或运行 `openclaw plugins enable clickclack`
会将 `clickclack` 追加到该列表。新手引导安装采用相同的
明确选择行为。这些路径不会覆盖 `plugins.deny` 或
全局 `plugins.enabled: false` 设置。直接执行
`openclaw plugins install @openclaw/clickclack` 会遵循常规的
插件安装策略，并且也会将 ClickClack 记录到现有允许列表中。

## 多个 Bot

每个账户都会建立自己的 ClickClack 实时连接，并使用自己的 Bot 令牌。

```json5
{
  channels: {
    clickclack: {
      enabled: true,
      baseUrl: "https://clickclack.example.com",
      defaultAccount: "service",
      accounts: {
        service: {
          token: { source: "env", provider: "default", id: "CLICKCLACK_SERVICE_BOT_TOKEN" },
          workspace: "default",
          defaultTo: "channel:general",
          agentId: "service-bot",
        },
        support: {
          token: { source: "env", provider: "default", id: "CLICKCLACK_SUPPORT_BOT_TOKEN" },
          workspace: "default",
          defaultTo: "dm:usr_...",
          agentId: "support-bot",
        },
      },
    },
  },
}
```

## 会话讨论

在一个 ClickClack 账户上启用讨论，可为每个 OpenClaw 会话提供
专用 ClickClack 频道。账户令牌必须包含
`channels:write`（`bot:admin` 权限包包含该权限）；常规 `bot:write`
设置令牌无法创建或同步频道。

```json5
{
  channels: {
    clickclack: {
      enabled: true,
      baseUrl: "https://clickclack.example.com",
      token: { source: "env", provider: "default", id: "CLICKCLACK_BOT_TOKEN" },
      workspace: "default",
      discussions: {
        enabled: true,
        workspace: "default",
        controlUrlBase: "https://team.openclaw.ai",
        section: "Sessions",
      },
    },
  },
}
```

`discussions.workspace` 接受与账户级 `workspace` 相同的工作区 ID、别名或显示名称，
并默认使用该值。`section` 控制
ClickClack 侧边栏分区，默认值为 `Sessions`。设置
`controlUrlBase` 后，托管频道会链接回真实的 Control UI
会话路由 `/chat?session=<encoded-session-key>`。

仅在一个 ClickClack 账户上启用讨论。Gateway 网关提供商
没有账户选择器，因此如果多个账户启用了讨论，系统会拒绝配置，
而不是按配置顺序选择其中一个。

打开讨论会创建一个标记为外部托管的公共 ClickClack
频道。插件会同步会话标签、类别和归档状态。
恢复会话会恢复其频道；清除会话类别
会将频道移回已配置的默认分区。删除
OpenClaw 会话会归档 ClickClack 频道，而不是将其删除，因此其
历史记录仍然可用。使用讨论 RPC 时，以及存在任何绑定期间大约每分钟一次，
插件会协调绑定状态。

托管频道中的入站消息会在
与所附加主会话相同的智能体 ID 下使用确定性的辅助会话。系统会告知辅助智能体
需要观察哪个主会话，并且它可以使用 `sessions_history` 和 `session_status`
（`changesSince` 适用于增量检查）。只有当讨论中的人员要求它
中继或引导主会话时，它才会使用 `sessions_send`。
绑定、托管所有权引用和辅助会话对等身份包含
具体的 OpenClaw 会话 ID，以及固定的 ClickClack 服务器和
频道。重置可复用的会话键或重新指定账户目标时，系统会在本地撤销
旧频道，在旧凭据仍可用时将其归档，并且
无法复用其辅助会话记录。通过已归档、已重置、
已禁用或已重新指定目标的绑定到达的消息会被丢弃，而不会回退到
账户的常规频道路由。已释放的绑定会留下持久的
已撤销频道标记，从而使延迟到达的实时事件继续采用故障关闭策略。远程
所有权以 ClickClack 服务器和频道 ID 为键，因此重命名本地
账户无法将托管频道变成普通频道。

将 `tools.sessions.visibility` 保持为更安全的默认值 `tree`。插件
仅在每个辅助会话及其附加的主会话之间安装主机范围的授权，
并额外安装一个工具策略钩子，用于阻止会话发现和
跨会话目标。它仅允许对所附加的主会话使用 `sessions_history`、`session_status` 和
`sessions_send`，并阻止状态调用
更改该会话的模型。这些工具仍必须存在于
智能体的有效工具允许列表中。系统提示词仅提供指导；主机授权
和钩子才是授权边界。

ClickClack 服务器必须在创建和更新频道时支持托管频道字段（`external_managed`、
`external_ref`、`external_url` 和 `sidebar_section`），并在频道响应中返回这些字段。OpenClaw
会先验证该契约，再持久化绑定。如果创建响应丢失，下次打开时会根据服务器强制执行的
`external_ref` 接管该频道，而不是再创建一个频道。在该结果完成协调之前，待处理的预留会隔离
目标工作区中原本未绑定的事件。粗粒度协调器会在同一会话仍处于活动状态时接管频道，或在
重置后将其归档；如果未创建远程频道，则会清除预留。
该引用包含一个针对每个 OpenClaw 安装实例的持久命名空间，以及
会话键、具体会话 ID、ClickClack 目标位置和持久
绑定代次的哈希。不同的 Gateway 网关无法接管彼此的频道，
重置后的会话无法继承旧频道历史记录，并且账号或工作区
往返变更后无法重新接管之前的频道。绑定还会固定到
已配置的 ClickClack 服务器 URL；如果账号改为指向其他服务器，绑定将失效。
更改或移除 `controlUrlBase` 会在下一次协调过程中更新或清除托管
频道链接。更改
`discussions.workspace` 时，如果旧工作区凭据仍保持配置，则必须先归档并释放旧绑定，
之后才能在新工作区中打开频道。如果令牌被替换为
无法访问旧工作区的工作区范围凭据，OpenClaw 会将旧频道记录为已撤销并
释放绑定，而不会尝试使用替换令牌；请从 ClickClack 中归档该遗留
频道。

附加的主会话还会获得一个仅拉取的 `discussion` 工具。它将
最新消息和近期帖子回复读取为每条消息一条经过转义且带归属信息的记录，
并且不会产生写入或生命周期副作用。频道根消息和帖子
查询具有固定的请求预算；当该安全界限可能遗漏较早但仍活跃的帖子时，
结果会明确发出警告。

## 回复模式

- `replyMode: "agent"`（默认）通过常规智能体流水线分派入站消息，包括会话记录和工具策略。
- `replyMode: "model"` 跳过智能体流水线，并使用插件运行时的 `llm.complete` 直接回复 Bot，还可选择通过 `model` 和 `systemPrompt` 调整回复形式。所选提供商和模型负责控制补全预算。

模型模式会针对解析出的 Bot 智能体 ID 运行补全，这要求显式启用
`plugins.entries.clickclack.llm.allowAgentIdOverride: true` 信任
位：

```json5
{
  plugins: {
    entries: {
      clickclack: {
        llm: {
          allowAgentIdOverride: true,
        },
      },
    },
  },
}
```

如果只使用默认的 `agent` 回复模式，请保持该信任位关闭；
该模式不需要此设置。

## 命令菜单

在 Gateway 网关启动时，每个已配置的账号都会将 OpenClaw 的原生
命令发布到 ClickClack。这些命令会显示在编辑器自动补全中，并以
Bot 的账号名称作为标签。每次启动时都会整体替换已发布的命令集；
当原生命令目录为空时，也会清除过期菜单。

命令菜单同步默认启用。为账号设置 `commandMenu: false`
即可选择退出：

```json5
{
  channels: {
    clickclack: {
      enabled: true,
      token: { source: "env", provider: "default", id: "CLICKCLACK_BOT_TOKEN" },
      workspace: "default",
      commandMenu: false,
    },
  },
}
```

令牌需要 `commands:write`。当前 ClickClack 的 `bot:write` 和
`bot:admin` 权限包包含该权限范围，也可以单独授予。
在引入命令菜单之前创建的令牌可能需要添加该权限范围或更换令牌。

同步以尽力而为方式执行，每次 Gateway 网关启动时运行一次。缺少权限范围或网络
故障会记录警告；不具备该端点的旧版 ClickClack 服务器会记录调试级别日志。
这些故障都不会阻止实时服务启动。智能体离线期间菜单仍然
可用，当 Bot 离开工作区时则会被移除。

此版本仅发布原生命令规范。别名以及
技能、插件或自定义命令目录不会添加到菜单中。如果某个
名称也注册为 HTTP 斜杠命令，ClickClack 会优先分派该
注册项；其他菜单命令仍通过常规消息
传递。

使用 `agent` 模式获取跨服务关联证据。对于采用规范 `msg_<ulid>`
格式的权威 ClickClack 消息 ID，该渠道会派生
确定性的 OpenClaw 运行 ID `clickclack:<message-id>`。随后，每次模型调用
都会在诊断中显示为 `clickclack:<message-id>:model:<n>`；当该
轮次使用 ClawRouter 时，同一模型调用 ID 会作为 `X-Request-ID` 发送。
`model` 模式会绕过常规智能体运行/会话诊断，因此
不适用于此证据路径。

当实时事件包含经过验证的 `payload.correlation_id` 时，
该渠道会在权威消息获取以及由此产生的 ClickClack 回复请求中将其作为
`X-Correlation-ID` 传递。值使用 ClickClack 的安全
128 字符集（`A-Z`、`a-z`、`0-9`、`.`、`_`、`:` 和 `-`）；无效值
会被省略。这些关联仅包含标识符，绝不包含消息正文、
提示词、补全、凭据或工具输出。

## 持久媒体传递

包含媒体的智能体回复使用必需的持久传递。OpenClaw 会在
首次写入 ClickClack 之前，为每个部分分配稳定的消息和上传 nonce，因此
重试会复用相同的上传和消息，而不会消耗额外的存储配额
或发布重复内容。如果重启后上传已存在，
OpenClaw 不会重新读取原始本地路径或远程媒体 URL。

此恢复契约要求 ClickClack 服务器支持：

- `GET /api/uploads/by-nonce`，并在
  找到和未找到结果中包含 `X-ClickClack-Upload-Nonce: supported`。
- `GET /api/messages/by-nonce`，并在
  找到和未找到结果中包含 `X-ClickClack-Message-Nonce: supported`。
- 对于同一所有者范围 nonce 和上传，消息创建和附件关联必须具有幂等性。

旧版服务器返回的通用 404 不会被视为发送不存在的证据。
OpenClaw 会让传递保持未解决状态，而不会冒险产生重复内容；请先更新
ClickClack，再启用会生成媒体的智能体回复。

## 智能体活动行

默认情况下，智能体轮次运行期间 ClickClack 频道不会显示任何内容；只有最终回复会发送。为账号设置 `agentActivity: true`，即可在轮次进行期间发布持久的 `agent_commentary` 和 `agent_tool` 消息行：

```json5
{
  channels: {
    clickclack: {
      enabled: true,
      token: { source: "env", provider: "default", id: "CLICKCLACK_BOT_TOKEN" },
      workspace: "default",
      agentActivity: true,
    },
  },
}
```

要求和行为：

- **默认关闭。** 标准设置和旧版 ClickClack 服务器不受影响。
- **需要 `agent_activity:write` 令牌权限范围。** 此权限范围与 `bot:write` 分离，且不会从后者继承；启用该选项前，请使用 `--scopes bot:write,agent_activity:write` 创建 Bot 令牌（或将该权限范围授予现有令牌）。
- **尽力降级。** 如果令牌缺少 `agent_activity:write` 或服务器拒绝活动写入，则会记录故障，最终回复仍会正常传递；不会显示活动行。
- 各行按轮次分组（`turn_id`），并进行合并，使一个逻辑步骤对应一行；工具行使用与 Discord/Slack/Telegram 相同的进度格式（工具名称加命令详情）。
- **归属元数据。** 智能体发布的内容（活动行和最终回复）会携带根据该轮次实际使用的模型（包括回退后的模型）解析出的 `author_model` 和 `author_thinking` 字段。未定义这些列的服务器会忽略未知 JSON 字段；持久化这些字段的服务器可以针对每条消息回答“哪种模型以哪种思考级别说出了这一行”。

## 目标

- `channel:<name-or-id>` 发送到工作区频道。裸目标默认为 `channel:`。
- `dm:<user_id>` 创建或复用与该用户的私聊。
- `thread:<message_id>` 在以该消息为根的帖子中回复。

显式出站目标也可以携带 `clickclack:` 或 `cc:` 提供商前缀。

出站媒体使用 ClickClack 的上传 API，然后将持久上传内容附加到
已创建的频道消息、帖子回复或私信。对于本地文件和支持的
远程媒体 URL，会遵循 OpenClaw 的常规媒体访问策略，每个文件的限制为 64 MiB。
持久排队发送会为每个上传和消息部分使用单独的所有者范围 nonce，
然后使用这些相同对象重试附件关联。有关服务器
契约和恢复行为，请参阅[持久媒体传递](#durable-media-delivery)。

示例：

```bash
openclaw message send --channel clickclack --target channel:general --message "hello"
openclaw message send --channel clickclack --target dm:usr_123 --message "hello"
openclaw message send --channel clickclack --target thread:msg_123 --message "following up"
```

## 权限

ClickClack 令牌权限范围由 ClickClack API 强制执行。

- `bot:read`：读取工作区/频道/消息/帖子/私信/实时/Profile 数据。
- `bot:write`：`bot:read`，外加频道消息、帖子回复、私信、上传和命令菜单发布。
- `bot:admin`：`bot:write`，外加频道创建。
- `commands:write`：发布 Bot 的命令菜单。当前 `bot:write` 和 `bot:admin` 权限包包含此权限，也可单独授予。
- `agent_activity:write`：持久智能体活动行（`agent_commentary` / `agent_tool`）。不会从 `bot:write` 或 `bot:admin` 继承；仅在设置 `agentActivity: true` 时才需要。

OpenClaw 的常规智能体聊天和命令菜单同步只需要当前的 `bot:write`。启用[智能体活动行](#agent-activity-rows)时，请添加 `agent_activity:write`。

## 故障排查

- `ClickClack is not configured for account "<id>"`：为该账号设置 `baseUrl`、`token`（例如通过 `CLICKCLACK_BOT_TOKEN`）和 `workspace`。
- `ClickClack workspace not found: <value>`：将 `workspace` 设置为 ClickClack 返回的工作区 ID、slug 或名称。
- 无入站回复：确认令牌具有实时读取权限，并注意 Bot 会忽略自身消息以及其他 Bot 的消息。
- 频道发送失败：确认 Bot 是工作区成员并具有 `bot:write`。
- 没有命令菜单：确认 `commandMenu` 不是 `false`、ClickClack 服务器支持 `PUT /api/bots/self/commands`，并且令牌具有 `commands:write`。
