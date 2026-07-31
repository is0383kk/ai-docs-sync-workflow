---
read_when:
    - 为 OpenClaw 设置 Twitch 聊天集成
sidebarTitle: Twitch
summary: Twitch 聊天机器人：安装、凭据、访问控制、令牌刷新
title: Twitch
x-i18n:
    generated_at: "2026-07-26T05:39:52Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: d827c742ded5fd0b071443dead27b975e2414419b0facb486d7f9c0c9800b060
    source_path: channels/twitch.md
    workflow: 16
---

通过 Twurple 客户端使用 Twitch 的聊天（IRC）接口提供 Twitch 聊天支持。OpenClaw 以 Twitch Bot 账号登录，每个已配置账号加入一个频道，并在该频道中回复。

## 安装

Twitch 作为官方插件发布；它不属于核心安装的一部分。

<Tabs>
  <Tab title="npm 注册表">
    ```bash
    openclaw plugins install @openclaw/twitch
    ```
  </Tab>
  <Tab title="本地检出">
    ```bash
    openclaw plugins install ./path/to/local/twitch-plugin
    ```
  </Tab>
</Tabs>

`plugins install` 会注册并启用该插件。在 `openclaw onboard` 或 `openclaw channels add` 期间选择 Twitch，会按需安装该插件。使用不带版本的包名可跟随当前版本；仅在需要可复现安装时固定确切版本。需要 OpenClaw 2026.4.10 或更高版本。

详情：[插件](/zh-CN/tools/plugin)

## 快速设置

<Steps>
  <Step title="安装插件">
    请参阅上文的[安装](#install)。
  </Step>
  <Step title="创建 Twitch Bot 账号">
    为 Bot 创建专用 Twitch 账号（也可以使用现有账号）。
  </Step>
  <Step title="生成凭据">
    使用 [Twitch Token Generator](https://twitchtokengenerator.com/)：

    - 选择 **Bot Token**
    - 确认已选择权限范围 `chat:read` 和 `chat:write`
    - 复制 **Client ID** 和 **Access Token**

  </Step>
  <Step title="查找你的 Twitch 用户 ID">
    使用 [https://www.streamweasels.com/tools/convert-twitch-username-to-user-id/](https://www.streamweasels.com/tools/convert-twitch-username-to-user-id/) 将用户名转换为 Twitch 用户 ID。
  </Step>
  <Step title="配置令牌">
    - 环境变量：`OPENCLAW_TWITCH_ACCESS_TOKEN=...`（仅适用于默认账号）
    - 或配置：`channels.twitch.accessToken`

    如果两者均已设置，配置优先（环境变量仅作为默认账号的回退值）。

  </Step>
  <Step title="启动 Gateway 网关">
    ```bash
    openclaw gateway run
    ```
  </Step>
</Steps>

<Warning>
添加访问控制（`allowFrom` 或 `allowedRoles`），防止未授权用户触发 Bot。`requireMention` 默认为 `true`。
</Warning>

最小配置：

```json5
{
  channels: {
    twitch: {
      enabled: true,
      username: "openclaw", // Bot 的 Twitch 账号（用于身份验证）
      accessToken: "oauth:abc123...", // OAuth 访问令牌（或使用 OPENCLAW_TWITCH_ACCESS_TOKEN 环境变量）
      clientId: "xyz789...", // 来自 Token Generator 的客户端 ID
      channel: "yourchannel", // 要加入哪个 Twitch 频道的聊天室（必需）
      allowFrom: ["123456789"], // （推荐）仅填写你的 Twitch 用户 ID
    },
  },
}
```

## 工作原理

- 由 Gateway 网关拥有的 Twitch 频道。
- 确定性路由：回复始终发回消息来源的 Twitch 频道。
- 每个已加入的频道都映射到一个隔离的群组会话键 `agent:<agentId>:twitch:group:<channel>`。
- `username` 是 Bot 的账号（用于身份验证），`channel` 是要加入的聊天室。每个账号条目只加入一个频道。
- 令牌无论是否带有 `oauth:` 前缀均可使用；OpenClaw 会规范化这两种形式（设置向导要求使用 `oauth:` 形式）。

## 入站持久性

OpenClaw 会在正常分发前，将每条已接受的 Twitch 聊天消息持久化到队列。待处理或可重试的消息在 Gateway 网关重启后仍会保留，并按已配置的频道串行处理；只要存在活动或保留的完成记录，就会使用 Twitch 的消息 ID 阻止重复的队列条目。

Twitch 聊天不会在客户端接受 `PRIVMSG` 后重放它。这可以防范从本地接受消息到分发消息期间的崩溃窗口，但无法恢复在持久化接纳前错过的消息。如果追加队列本身失败，OpenClaw 会记录该故障；重新连接不会要求 Twitch 重新发送该消息。

## 令牌刷新（可选）

[Twitch Token Generator](https://twitchtokengenerator.com/) 生成的令牌无法由 OpenClaw 刷新——过期后请重新生成（有效期为几小时；无需注册应用）。

如需自动刷新，请在 [Twitch Developer Console](https://dev.twitch.tv/console) 创建自己的应用，并添加：

```json5
{
  channels: {
    twitch: {
      clientSecret: "your_client_secret",
      refreshToken: "your_refresh_token",
    },
  },
}
```

两者均已设置时，插件会使用可刷新身份验证提供商，在令牌过期前进行续期，并记录每次刷新。如果没有 `refreshToken`，则记录 `token refresh disabled (no refresh token)`；如果没有 `clientSecret`，则回退到静态（不可刷新）令牌。

## 多账号支持

使用 `channels.twitch.accounts` 配置各账号的凭据。有关共用模式，请参阅[配置](/zh-CN/gateway/configuration)。

示例（一个 Bot 账号用于两个频道）：

```json5
{
  channels: {
    twitch: {
      accounts: {
        channel1: {
          username: "openclaw",
          accessToken: "oauth:abc123...",
          clientId: "xyz789...",
          channel: "yourchannel",
        },
        channel2: {
          username: "openclaw",
          accessToken: "oauth:def456...",
          clientId: "uvw012...",
          channel: "secondchannel",
        },
      },
    },
  },
}
```

<Note>
每个账号条目都需要自己的 `accessToken`（环境变量仅涵盖默认账号）。一个账号只加入一个频道，因此加入两个频道意味着需要两个账号。`channels.twitch.defaultAccount` 用于选择哪个账号作为默认账号。
</Note>

## 访问控制

`allowFrom` 是 Twitch 用户 ID 的硬性允许列表。设置后会忽略 `allowedRoles`；如需改用基于角色的访问控制，请不要设置 `allowFrom`。

**可用角色：** `"moderator"`、`"owner"`、`"vip"`、`"subscriber"`、`"all"`。

<Tabs>
  <Tab title="用户 ID 允许列表（最安全）">
    ```json5
    {
      channels: {
        twitch: {
          accounts: {
            default: {
              allowFrom: ["123456789", "987654321"],
            },
          },
        },
      },
    }
    ```
  </Tab>
  <Tab title="基于角色">
    ```json5
    {
      channels: {
        twitch: {
          accounts: {
            default: {
              allowedRoles: ["moderator", "vip"],
            },
          },
        },
      },
    }
    ```
  </Tab>
  <Tab title="禁用 @提及要求">
    默认情况下，`requireMention` 为 `true`。要响应所有获准的消息：

    ```json5
    {
      channels: {
        twitch: {
          accounts: {
            default: {
              requireMention: false,
            },
          },
        },
      },
    }
    ```

  </Tab>
</Tabs>

<Note>
**为什么使用用户 ID？** 用户名可以更改，从而可能被冒充。用户 ID 是永久不变的。

使用[用户名转 ID 工具](https://www.streamweasels.com/tools/convert-twitch-username-to-user-id/)查找你的用户 ID。
</Note>

## 故障排查

首先运行诊断命令：

```bash
openclaw doctor
openclaw channels status --probe
```

<AccordionGroup>
  <Accordion title="Bot 不响应消息">
    - **检查访问控制：** 确保你的用户 ID 位于 `allowFrom` 中，或者临时移除 `allowFrom` 并设置 `allowedRoles: ["all"]` 进行测试。
    - **检查提及门控：** 使用 `requireMention: true`（默认值）时，消息必须 @提及 Bot 用户名。
    - **检查 Bot 是否在频道中：** Bot 只会加入 `channel` 中指定的频道。

  </Accordion>
  <Accordion title="令牌问题">
    “连接失败”或身份验证错误：

    - 确认 `accessToken` 是 OAuth 访问令牌值（`oauth:` 前缀可选）
    - 检查令牌是否具有 `chat:read` 和 `chat:write` 权限范围
    - 如果使用令牌刷新，请确认已设置 `clientSecret` 和 `refreshToken`

  </Accordion>
  <Accordion title="令牌刷新不起作用">
    检查日志中的刷新事件：

    ```text
    对 mybot 使用环境变量令牌来源
    用户 123456 的访问令牌已刷新（将在 14400s 后过期）
    ```

    如果看到 `token refresh disabled (no refresh token)`：

    - 确保已提供 `clientSecret`
    - 确保已提供 `refreshToken`

  </Accordion>
</AccordionGroup>

## 配置

### 账号配置

<ParamField path="username" type="string" required>
  Bot 用户名（用于身份验证的账号）。
</ParamField>
<ParamField path="accessToken" type="string" required>
  具有 `chat:read` 和 `chat:write` 权限范围的 OAuth 访问令牌（默认账号可使用配置或环境变量）。
</ParamField>
<ParamField path="clientId" type="string" required>
  Twitch 客户端 ID（来自 Token Generator 或你的应用）。在模式中是可选项，但建立连接时必需。
</ParamField>
<ParamField path="channel" type="string" required>
  要加入的频道。
</ParamField>
<ParamField path="enabled" type="boolean" default="true">
  启用此账号。
</ParamField>
<ParamField path="clientSecret" type="string">
  可选：用于自动刷新令牌。
</ParamField>
<ParamField path="refreshToken" type="string">
  可选：用于自动刷新令牌。
</ParamField>
<ParamField path="expiresIn" type="number">
  令牌过期时间，以秒为单位（刷新跟踪）。
</ParamField>
<ParamField path="obtainmentTimestamp" type="number">
  获取令牌时的时间戳（刷新跟踪）。
</ParamField>
<ParamField path="allowFrom" type="string[]">
  用户 ID 允许列表。设置后将忽略角色。
</ParamField>
<ParamField path="allowedRoles" type='Array<"moderator" | "owner" | "vip" | "subscriber" | "all">'>
  基于角色的访问控制。
</ParamField>
<ParamField path="requireMention" type="boolean" default="true">
  要求使用 @提及才能触发 Bot。
</ParamField>
<ParamField path="responsePrefix" type="string">
  覆盖此账号的出站回复前缀。
</ParamField>

### 提供商选项

- `channels.twitch.enabled` - 启用/禁用频道启动
- `channels.twitch.username` / `accessToken` / `clientId` / `channel` - 简化的单账号配置（隐式 `default` 账号；优先于 `accounts.default`）
- `channels.twitch.accounts.<accountName>` - 多账号配置（包括上述所有账号字段）
- `channels.twitch.defaultAccount` - 哪个账号名称为默认账号
- `channels.twitch.markdown.tables` - Markdown 表格渲染模式（`off` | `bullets` | `code` | `block`）

完整示例：

```json5
{
  channels: {
    twitch: {
      enabled: true,
      username: "openclaw",
      accessToken: "oauth:abc123...",
      clientId: "xyz789...",
      channel: "yourchannel",
      clientSecret: "secret123...",
      refreshToken: "refresh456...",
      allowFrom: ["123456789"],
      accounts: {
        second: {
          username: "mybot",
          accessToken: "oauth:def456...",
          clientId: "uvw012...",
          channel: "your_channel",
          enabled: true,
          expiresIn: 14400,
          obtainmentTimestamp: 1706092800000,
          allowedRoles: ["moderator"],
        },
      },
    },
  },
}
```

## 工具操作

智能体可以通过消息工具的 `send` 操作发送 Twitch 消息：

```json5
{
  channel: "twitch",
  action: "send",
  to: "#mychannel",
  message: "Hello Twitch!",
}
```

`to` 是可选项，默认使用账号已配置的 `channel`。

## 安全与运维

- **将令牌视同密码** — 切勿将令牌提交到 git。
- 对于长期运行的 Bot，**使用自动令牌刷新**。
- 使用**用户 ID 允许列表**而非用户名进行访问控制。
- **监控日志**中的令牌刷新事件和连接状态。
- **尽可能缩小令牌权限范围** — 仅请求 `chat:read` 和 `chat:write`。
- **如果遇到问题**：确认没有其他进程占用该会话后，重启 Gateway 网关。

## 限制

- 每条消息不超过 **500 个字符**；较长的回复会在单词边界处分块。
- 发送前会移除 Markdown 格式（Twitch 聊天使用纯文本；换行符会转换为空格）。
- OpenClaw 本身不添加速率限制；Twurple 聊天客户端负责处理 Twitch 的速率限制。

## 相关内容

- [频道路由](/zh-CN/channels/channel-routing) — 消息的会话路由
- [渠道概览](/zh-CN/channels) — 所有受支持的渠道
- [群组](/zh-CN/channels/groups) — 群聊行为和提及门控
- [配对](/zh-CN/channels/pairing) — 私信身份验证和配对流程
- [安全](/zh-CN/gateway/security) — 访问模型和安全强化
