---
read_when:
    - 开发 Discord 频道功能
summary: Discord Bot 设置、配置键、组件、语音和故障排查
title: Discord
x-i18n:
    generated_at: "2026-07-26T06:32:52Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 52a2926217f3a8dfb9398551ddacb0bc6aae6de0a164b215c55256eda9b6245e
    source_path: channels/discord.md
    workflow: 16
---

OpenClaw 通过官方 Discord Gateway 网关以机器人的身份连接到 Discord。支持私信和服务器频道。

<CardGroup cols={3}>
  <Card title="配对" icon="link" href="/zh-CN/channels/pairing">
    Discord 私信默认使用配对模式。
  </Card>
  <Card title="斜杠命令" icon="terminal" href="/zh-CN/tools/slash-commands">
    原生命令行为和命令目录。
  </Card>
  <Card title="渠道故障排查" icon="wrench" href="/zh-CN/channels/troubleshooting">
    跨渠道诊断和修复流程。
  </Card>
</CardGroup>

## 快速设置

创建一个带机器人的 Discord 应用，将机器人添加到你的服务器，然后将其与 OpenClaw 配对。如果可以，请使用私人服务器；如有需要，请先[创建一个服务器](https://support.discord.com/hc/en-us/articles/204849977-How-do-I-create-a-server)（**Create My Own > For me and my friends**）。

<Steps>
  <Step title="创建 Discord 应用和机器人">
    在 [Discord Developer Portal](https://discord.com/developers/applications) 中，点击 **New Application** 并为其命名（例如“OpenClaw”）。

    在侧边栏中打开 **Bot**，并将 **Username** 设置为你的智能体名称。

  </Step>

  <Step title="启用特权意图">
    仍在 **Bot** 页面上的 **Privileged Gateway Intents** 下，启用：

    - **Message Content Intent**（必需）
    - **Server Members Intent**（推荐；角色允许列表、名称到 ID 的匹配以及频道受众访问组需要此项）
    - **Presence Intent**（可选；仅用于在线状态更新）

  </Step>

  <Step title="复制机器人令牌">
    在 **Bot** 页面上，点击 **Reset Token** 并复制令牌。

    <Note>
    尽管名称如此，这会生成你的第一个令牌——并没有“重置”任何内容。
    </Note>

  </Step>

  <Step title="生成邀请 URL 并将机器人添加到服务器">
    在侧边栏中打开 **OAuth2**。在 **OAuth2 URL Generator** 中启用以下作用域：

    - `bot`
    - `applications.commands`

    在随即出现的 **Bot Permissions** 部分中，至少启用：

    **General Permissions**
      - View Channels

    **Text Permissions**
      - Send Messages
      - Read Message History
      - Embed Links
      - Attach Files
      - Add Reactions（可选）

    这些是普通文本频道的基本权限。如果机器人将在帖子串中发帖——包括创建或延续帖子串的论坛或媒体频道工作流——还应启用 **Send Messages in Threads**。

    复制生成的 URL，在浏览器中打开它，选择你的服务器，然后点击 **Continue**。此时机器人应会出现在你的服务器中。

  </Step>

  <Step title="启用开发者模式并收集 ID">
    在 Discord 应用中启用开发者模式，以便复制 ID：

    1. **User Settings**（齿轮图标）→ **Developer** → 开启 **Developer Mode**
       *（移动端：**App Settings** → **Advanced**）*
    2. 右键点击你的**服务器图标** → **Copy Server ID**
    3. 右键点击你**自己的头像** → **Copy User ID**

    将服务器 ID 和用户 ID 与机器人令牌一起妥善保存；下一步需要这三项信息。

  </Step>

  <Step title="允许接收服务器成员的私信">
    要使配对正常工作，Discord 必须允许机器人向你发送私信。右键点击你的**服务器图标** → **Privacy Settings** → 开启 **Direct Messages**。

    如果你通过 Discord 私信使用 OpenClaw，请保持此项开启。如果你只使用服务器频道，可在配对后将其关闭。

  </Step>

  <Step title="安全设置机器人令牌（不要在聊天中发送）">
    机器人令牌是机密信息。在向智能体发送消息之前，请在运行 OpenClaw 的机器上进行设置：

```bash
export DISCORD_BOT_TOKEN="YOUR_BOT_TOKEN"
cat > discord.patch.json5 <<'JSON5'
{
  channels: {
    discord: {
      enabled: true,
      token: { source: "env", provider: "default", id: "DISCORD_BOT_TOKEN" },
    },
  },
}
JSON5
openclaw config patch --file ./discord.patch.json5 --dry-run
openclaw config patch --file ./discord.patch.json5
openclaw gateway
```

    如果 OpenClaw 已作为后台服务运行，请通过 OpenClaw Mac 应用重启它，或停止并重新启动 `openclaw gateway run` 进程。
    对于托管服务安装，请在已设置 `DISCORD_BOT_TOKEN` 的 shell 中运行 `openclaw gateway install`，或将该变量存储在 `~/.openclaw/.env` 中，以便服务在重启后能够解析环境变量 SecretRef。
    如果你的主机在 Discord 启动应用查询时被阻止或受到速率限制，请设置 Developer Portal 中的应用/客户端 ID，以便启动时跳过该 REST 调用：默认账户使用 `channels.discord.applicationId`，每个机器人则使用 `channels.discord.accounts.<accountId>.applicationId`。

  </Step>

  <Step title="配置 OpenClaw 并配对">

    <Tabs>
      <Tab title="让你的智能体完成">
        在现有渠道（例如 Telegram）中与你的 OpenClaw 智能体聊天，并告诉它完成设置。如果 Discord 是你的第一个渠道，请改用 CLI / 配置选项卡。

        > “我已经在配置中设置了 Discord 机器人令牌。请使用用户 ID `<user_id>` 和服务器 ID `<server_id>` 完成 Discord 设置。”
      </Tab>
      <Tab title="CLI / 配置">
        基于文件的配置：

```json5
{
  channels: {
    discord: {
      enabled: true,
      token: {
        source: "env",
        provider: "default",
        id: "DISCORD_BOT_TOKEN",
      },
    },
  },
}
```

        默认账户的环境变量回退：

```bash
DISCORD_BOT_TOKEN=...
```

        对于脚本化或远程设置，请使用 `openclaw config patch --file ./discord.patch.json5 --dry-run` 写入相同的 JSON5 块，然后在不带 `--dry-run` 的情况下重新运行。也可使用明文 `token` 字符串，并且 `channels.discord.token` 支持 env/file/exec 提供商的 SecretRef 值。请参阅[机密信息管理](/zh-CN/gateway/secrets)。

        对于多个 Discord 机器人，请将每个机器人的令牌和应用 ID 保存在对应账户下。账户会继承顶层的 `channels.discord.applicationId`，因此仅当每个账户都使用相同的应用 ID 时，才在顶层设置该值。

```json5
{
  channels: {
    discord: {
      enabled: true,
      accounts: {
        personal: {
          token: { source: "env", provider: "default", id: "DISCORD_PERSONAL_TOKEN" },
          applicationId: "111111111111111111",
        },
        work: {
          token: { source: "env", provider: "default", id: "DISCORD_WORK_TOKEN" },
          applicationId: "222222222222222222",
        },
      },
    },
  },
}
```

      </Tab>
    </Tabs>

  </Step>

  <Step title="批准首次私信配对">
    Gateway 网关运行后，在 Discord 中向机器人发送私信。它会回复一个配对码。

    <Tabs>
      <Tab title="让你的智能体完成">
        通过现有渠道将配对码发送给你的智能体：

        > “批准此 Discord 配对码：`<CODE>`”
      </Tab>
      <Tab title="CLI">

```bash
openclaw pairing list discord
openclaw pairing approve discord <CODE>
```

      </Tab>
    </Tabs>

    配对码会在 1 小时后过期。批准后，即可在 Discord 私信中与你的智能体聊天。

  </Step>
</Steps>

<Note>
令牌解析会区分账户。配置中的令牌值优先于环境变量回退，并且 `DISCORD_BOT_TOKEN` 仅用于默认账户。
如果两个已启用的 Discord 账户解析到相同的机器人令牌，OpenClaw 只会为该令牌启动一个 Gateway 网关监控器：配置来源的令牌优先于环境变量回退；否则第一个启用的账户优先，重复账户会被报告为已禁用，原因为 `duplicate bot token`。
对于高级出站调用（消息工具/渠道操作），每次调用显式指定的 `token` 将用于该调用。这适用于发送操作以及读取/探测类操作（读取/搜索/获取/帖子串/置顶消息/权限）。账户策略和重试设置仍来自活动运行时快照中所选的账户。
</Note>

## 推荐：设置服务器工作区

私信正常工作后，你可以将服务器转变为完整的工作区，其中每个频道都有一个具备独立上下文的智能体会话。推荐用于只有你和机器人的私人服务器。

<Steps>
  <Step title="将服务器添加到服务器允许列表">
    这样智能体便可在服务器的任何频道中回复，而不仅限于私信。

    <Tabs>
      <Tab title="让你的智能体完成">
        > “将我的 Discord 服务器 ID `<server_id>` 添加到服务器允许列表”
      </Tab>
      <Tab title="配置">

```json5
{
  channels: {
    discord: {
      groupPolicy: "allowlist",
      guilds: {
        YOUR_SERVER_ID: {
          requireMention: true,
          users: ["YOUR_USER_ID"],
        },
      },
    },
  },
}
```

      </Tab>
    </Tabs>

  </Step>

  <Step title="允许无需 @提及即可回复">
    默认情况下，只有在服务器频道中 @提及智能体时，它才会回复。在私人服务器中，你可能希望它回复每条消息。

    默认情况下，普通回复会自动发布到服务器频道。对于共享的常驻房间，可选择启用 `messages.groupChat.visibleReplies: "message_tool"`，让智能体静默观察，并且仅在认为频道回复有用时才发布消息。此功能最适合 GPT-5.6 Sol 等最新一代且工具调用可靠的模型。除非工具发送消息，否则环境房间事件会保持静默。有关完整的静默观察模式配置，请参阅[环境房间事件](/zh-CN/channels/ambient-room-events)。

    如果 Discord 显示正在输入，日志也显示使用了令牌，但没有发布消息，请检查该轮次是否被配置为环境房间事件，或是否选择了通过消息工具发布可见回复。

    <Tabs>
      <Tab title="让你的智能体完成">
        > “允许我的智能体在此服务器上无需 @提及即可回复”
      </Tab>
      <Tab title="配置">
        在服务器配置中设置 `requireMention: false`：

```json5
{
  channels: {
    discord: {
      guilds: {
        YOUR_SERVER_ID: {
          requireMention: false,
        },
      },
    },
  },
}
```

        若要要求通过消息工具发送可见的群组/频道回复，请设置 `messages.groupChat.visibleReplies: "message_tool"`。

      </Tab>
    </Tabs>

  </Step>

  <Step title="规划服务器频道中的记忆使用方式">
    长期记忆（MEMORY.md）仅在私信会话中自动加载；服务器频道不会加载它。

    <Tabs>
      <Tab title="让你的智能体完成">
        > “当我在 Discord 频道中提问时，如果需要 MEMORY.md 中的长期上下文，请使用 memory_search 或 memory_get。”
      </Tab>
      <Tab title="手动">
        若要在每个频道中共享上下文，请将稳定的指令放入 `AGENTS.md` 或 `USER.md`（会注入每个会话）。将长期笔记保存在 `MEMORY.md` 中，并根据需要使用记忆工具访问。
      </Tab>
    </Tabs>

  </Step>
</Steps>

现在创建频道并开始聊天。智能体可以看到频道名称，并且每个频道都是独立会话——你可以设置 `#coding`、`#home`、`#research`，或任何适合你工作流的频道。

## 运行时模型

- Gateway 网关负责 Discord 连接。
- 回复路由是确定性的：来自 Discord 的入站消息会回复到 Discord。
- Discord 服务器/频道元数据会作为不受信任的上下文添加到模型提示词中，而不是作为用户可见的回复前缀。如果模型将该封装内容复制到回复中，OpenClaw 会从出站回复和未来的重放上下文中移除复制的元数据。
- 默认情况下（`session.dmScope=main`），直接聊天共享智能体的主会话（`agent:main:main`）。
- 服务器频道使用彼此隔离的会话键（`agent:<agentId>:discord:channel:<channelId>`）。
- 默认忽略群组私信（`channels.discord.dm.groupEnabled=false`）。
- 原生斜杠命令在隔离的命令会话（`agent:<agentId>:discord:slash:<userId>`）中运行，同时仍会将 `CommandTargetSessionKey` 传递给路由后的对话会话。
- 仅含文本的定时任务/Heartbeat 公告投递到 Discord 时，会合并为智能体最终可见的回答，并且只发送一次。当智能体生成多个可投递载荷时，媒体和结构化组件载荷仍会以多条消息发送。

## 论坛频道

Discord 论坛和媒体频道仅接受主题帖。OpenClaw 支持通过两种方式创建主题帖：

- 向论坛父频道（`channel:<forumId>`）发送消息以自动创建主题帖。主题帖标题取自消息的第一个非空行（截断至 Discord 的 100 字符主题帖名称限制）。
- 使用 `openclaw message thread create` 直接创建主题帖。对于论坛频道，请勿传递 `--message-id`。

向论坛父频道发送消息以创建主题帖：

```bash
openclaw message send --channel discord --target channel:<forumId> \
  --message "主题标题\n帖子的正文"
```

显式创建论坛主题帖：

```bash
openclaw message thread create --channel discord --target channel:<forumId> \
  --thread-name "主题标题" --message "帖子的正文"
```

论坛父频道不接受 Discord 组件。如果需要使用组件，请发送到主题帖本身（`channel:<threadId>`）。

## 交互式组件

OpenClaw 支持在智能体消息中使用 Discord components v2 容器。请通过消息工具传入 `components` 载荷。交互结果会作为普通入站消息路由回智能体，并遵循现有的 Discord `replyToMode` 设置。

支持的块：

- `text`、`section`、`separator`、`actions`、`media-gallery`、`file`
- 操作行最多允许 5 个按钮或一个选择菜单
- 选择类型：`string`、`user`、`role`、`mentionable`、`channel`

默认情况下，组件只能使用一次。设置 `components.reusable=true` 后，按钮、选择菜单和表单可在过期前多次使用。

若要限制谁可以点击按钮，请在该按钮上设置 `allowedUsers`（Discord 用户 ID、标签或 `*`）。不匹配的用户会收到仅自己可见的拒绝消息。

组件回调默认在 30 分钟后过期。设置 `channels.discord.agentComponents.ttlMs` 可更改默认账户的回调注册表生存期，也可使用 `channels.discord.accounts.<accountId>.agentComponents.ttlMs` 为每个账户单独设置。该值以毫秒为单位，必须为正整数，上限为 `86400000`（24 小时）。较长的 TTL 适用于需要让按钮持续可用的审查或审批工作流，但也会延长旧 Discord 消息仍能触发操作的时间窗口。应优先使用满足需求的最短 TTL；如果过期回调可能导致意外行为，请保留默认值。

`/model` 和 `/models` 斜杠命令会打开交互式模型选择器，其中包含提供商、模型和兼容运行时的下拉菜单，以及一个提交步骤。`/models add` 已弃用，它不会再从聊天中注册模型，而是返回弃用消息。选择器回复仅调用者可见，也只能由调用者使用。Discord 选择菜单最多包含 25 个选项，因此，如果希望选择器仅为 `openai` 或 `vllm` 等指定提供商显示动态发现的模型，请向 `agents.defaults.modelPolicy.allow` 添加 `provider/*` 条目。

文件附件：

- `file` 块必须指向附件引用（`attachment://<filename>`）
- 通过 `media`/`path`/`filePath` 提供附件（单个文件）；多个文件请使用 `media-gallery`
- 如果上传名称应与附件引用匹配，请使用 `filename` 覆盖上传名称

模态表单：

- 添加 `components.modal`，最多包含 5 个字段
- 字段类型：`text`、`checkbox`、`radio`、`select`、`role-select`、`user-select`
- OpenClaw 会自动添加触发按钮

示例：

```json5
{
  channel: "discord",
  action: "send",
  to: "channel:123456789012345678",
  message: "可选的后备文本",
  components: {
    reusable: true,
    text: "选择一个路径",
    blocks: [
      {
        type: "actions",
        buttons: [
          {
            label: "批准",
            style: "success",
            allowedUsers: ["123456789012345678"],
          },
          { label: "拒绝", style: "danger" },
        ],
      },
      {
        type: "actions",
        select: {
          type: "string",
          placeholder: "选择一个选项",
          options: [
            { label: "选项 A", value: "a" },
            { label: "选项 B", value: "b" },
          ],
        },
      },
    ],
    modal: {
      title: "详细信息",
      triggerLabel: "打开表单",
      fields: [
        { type: "text", label: "请求者" },
        {
          type: "select",
          label: "优先级",
          options: [
            { label: "低", value: "low" },
            { label: "高", value: "high" },
          ],
        },
      ],
    },
  },
}
```

## 访问控制和路由

<Tabs>
  <Tab title="私信策略">
    `channels.discord.dmPolicy` 控制私信访问。`channels.discord.allowFrom` 是规范的私信允许列表。

    - `pairing`（默认）
    - `allowlist`（至少需要一个 `allowFrom` 发送者）
    - `open`（要求 `channels.discord.allowFrom` 包含 `"*"`）
    - `disabled`

    如果私信策略不是开放模式，未知用户将被阻止（或在 `pairing` 模式下收到配对提示）。

    多账户优先级：

    - `channels.discord.accounts.default.allowFrom` 仅适用于 `default` 账户。
    - 对于单个账户，`allowFrom` 的优先级高于旧版 `dm.allowFrom`。
    - 当命名账户自身的 `allowFrom` 和旧版 `dm.allowFrom` 均未设置时，会继承 `channels.discord.allowFrom`。
    - 命名账户不会继承 `channels.discord.accounts.default.allowFrom`。

    为了兼容，仍会读取旧版 `channels.discord.dm.policy` 和 `channels.discord.dm.allowFrom`。如果能够在不改变访问权限的情况下完成迁移，`openclaw doctor --fix` 会将它们迁移到 `dmPolicy` 和 `allowFrom`。

    用于投递的私信目标格式：

    - `user:<id>`
    - `<@id>` 提及

    当启用默认频道时，纯数字 ID 通常解析为频道 ID；但为了兼容，账户有效私信 `allowFrom` 中列出的 ID 会被视为用户私信目标。

  </Tab>

  <Tab title="访问组">
    Discord 私信和文本命令授权可以使用 `channels.discord.allowFrom` 中的动态 `accessGroup:<name>` 条目。

    访问组名称在各消息渠道之间共享。如果要创建静态组，其成员以各渠道常规的 `allowFrom` 语法表示，请使用 `type: "message.senders"`；如果应根据 Discord 频道当前的 `ViewChannel` 受众动态确定成员资格，请使用 `type: "discord.channelAudience"`。共享访问组行为：[访问组](/zh-CN/channels/access-groups)。

```json5
{
  accessGroups: {
    operators: {
      type: "message.senders",
      members: {
        "*": ["global-owner-id"],
        discord: ["discord:123456789012345678"],
        telegram: ["987654321"],
      },
    },
  },
  channels: {
    discord: {
      dmPolicy: "allowlist",
      allowFrom: ["accessGroup:operators"],
    },
  },
}
```

    Discord 文本频道没有单独的成员列表。`type: "discord.channelAudience"` 对成员资格的定义为：私信发送者是已配置服务器的成员，并且在应用角色和频道覆盖设置后，当前对已配置频道拥有有效的 `ViewChannel` 权限。

    示例：允许任何能够看到 `#maintainers` 的人向 Bot 发送私信，同时对其他所有人关闭私信。

```json5
{
  accessGroups: {
    maintainers: {
      type: "discord.channelAudience",
      guildId: "1456350064065904867",
      channelId: "1456744319972282449",
      membership: "canViewChannel",
    },
  },
  channels: {
    discord: {
      dmPolicy: "allowlist",
      allowFrom: ["accessGroup:maintainers"],
    },
  },
}
```

    可以混合使用动态和静态条目：

```json5
{
  accessGroups: {
    maintainers: {
      type: "discord.channelAudience",
      guildId: "1456350064065904867",
      channelId: "1456744319972282449",
    },
  },
  channels: {
    discord: {
      dmPolicy: "allowlist",
      allowFrom: ["accessGroup:maintainers", "discord:123456789012345678"],
    },
  },
}
```

    查找失败时默认拒绝访问。如果 Discord 返回 `Missing Access`、成员查找失败，或频道属于其他服务器，则该私信发送者会被视为未获授权。

    使用频道受众访问组时，请在 Discord Developer Portal 中启用 **Server Members Intent**。私信不包含服务器成员状态，因此 OpenClaw 会在授权时通过 Discord REST 解析该成员。

  </Tab>

  <Tab title="服务器策略">
    服务器处理由 `channels.discord.groupPolicy` 控制：

    - `open`
    - `allowlist`
    - `disabled`

    当 `channels.discord` 存在时，安全基线为 `allowlist`。

    `allowlist` 行为：

    - 服务器必须匹配 `channels.discord.guilds`（首选 `id`，也接受 slug）
    - 可选的发送者允许列表：`users`（建议使用稳定 ID）和 `roles`（仅限角色 ID）；如果配置了其中任何一个，则发送者匹配 `users` 或 `roles` 时获准访问
    - 默认禁用直接名称/标签匹配；仅在应急兼容模式下启用 `channels.discord.dangerouslyAllowNameMatching: true`
    - `users` 支持名称/标签，但使用 ID 更安全；使用名称/标签条目时，`openclaw security audit` 会发出警告
    - 如果服务器配置了 `channels`，则拒绝未列出的频道
    - 如果服务器没有 `channels` 块，则允许该允许列表服务器中的所有频道

    示例：

```json5
{
  channels: {
    discord: {
      groupPolicy: "allowlist",
      guilds: {
        "123456789012345678": {
          requireMention: true,
          ignoreOtherMentions: true,
          users: ["987654321098765432"],
          roles: ["123456789012345678"],
          channels: {
            general: { enabled: true },
            help: { enabled: true, requireMention: true },
          },
        },
      },
    },
  },
}
```

    `openclaw doctor --fix` 会将旧版的每频道 `allow` 键迁移到 `enabled`。

    如果仅设置 `DISCORD_BOT_TOKEN` 而未创建 `channels.discord` 块，则运行时后备值为 `groupPolicy="allowlist"`（日志中会显示警告），即使 `channels.defaults.groupPolicy` 为 `open` 也是如此。

  </Tab>

  <Tab title="提及和群组私信">
    默认情况下，服务器消息必须提及 Bot 才会触发处理。

    提及检测包括：

    - 显式提及 Bot
    - 已配置的提及模式（`agents.entries.*.groupChat.mentionPatterns`，后备为 `messages.groupChat.mentionPatterns`）
    - 在支持的情况下，隐式回复 Bot 的行为

    编写 Discord 出站消息时，请使用规范的提及语法：用户使用 `<@USER_ID>`，频道使用 `<#CHANNEL_ID>`，角色使用 `<@&ROLE_ID>`。请勿使用旧版 `<@!USER_ID>` 昵称提及形式。

    `requireMention` 按服务器/频道配置（`channels.discord.guilds...`）。
    `ignoreOtherMentions` 可选择丢弃提及其他用户/角色但未提及 Bot 的消息（不包括 @everyone/@here）。

    群组私信：

    - 默认：忽略（`dm.groupEnabled=false`）
    - 可通过 `dm.groupChannels` 配置可选允许列表（频道 ID 或 slug）

  </Tab>
</Tabs>

### 基于角色的智能体路由

使用 `bindings[].match.roles`，可按角色 ID 将 Discord 服务器成员路由到不同的智能体。基于角色的绑定仅接受角色 ID，并在对等方或父级对等方绑定之后、仅服务器绑定之前进行求值。如果绑定还设置了其他匹配字段（例如 `peer` + `guildId` + `roles`），则所有已配置字段都必须匹配。

```json5
{
  bindings: [
    {
      agentId: "opus",
      match: {
        channel: "discord",
        guildId: "123456789012345678",
        roles: ["111111111111111111"],
      },
    },
    {
      agentId: "sonnet",
      match: {
        channel: "discord",
        guildId: "123456789012345678",
      },
    },
  ],
}
```

## 原生命令和命令身份验证

- `commands.native` 默认为 `"auto"`，并为 Discord 启用。
- 按渠道覆盖：`channels.discord.commands.native`。
- `commands.native=false` 会在启动期间跳过 Discord 斜杠命令注册和清理。之前注册的命令可能仍会显示在 Discord 中，直到你从 Discord 应用中将其移除。
- 原生命令身份验证使用与普通消息处理相同的 Discord 允许列表和策略。
- 未经授权的用户可能仍能在 Discord UI 中看到命令；执行时会强制实施 OpenClaw 身份验证，并回复“未授权”。
- 默认斜杠命令设置：`ephemeral: true`（`channels.discord.slashCommand.ephemeral`）。

有关命令目录和行为，请参阅[斜杠命令](/zh-CN/tools/slash-commands)。

## 功能详情

<AccordionGroup>
  <Accordion title="回复标签和原生回复">
    Discord 支持在智能体输出中使用回复标签：

    - `[[reply_to_current]]`
    - `[[reply_to:<id>]]`

    由 `channels.discord.replyToMode` 控制：

    - `off`（默认）：不进行隐式回复串联；仍会遵循显式的 `[[reply_to_*]]` 标签
    - `first`：将隐式原生回复引用附加到该轮次的第一条 Discord 出站消息
    - `all`：将其附加到每条出站消息
    - `batched`：仅当入站事件是由多条消息组成的防抖批次时附加；如果你主要希望在含义不明确的突发聊天中使用原生回复，而不是每个单消息轮次都使用，此选项很有用

    消息 ID 会显示在上下文/历史记录中，以便智能体定位特定消息。

  </Accordion>

  <Accordion title="链接预览">
    Discord 默认会为 URL 生成富链接嵌入。OpenClaw 默认会在 Discord 出站消息中抑制这些自动生成的嵌入，因此智能体发送的 URL 会保持为普通链接，除非你选择启用：

```json5
{
  channels: {
    discord: {
      suppressEmbeds: false,
    },
  },
}
```

    设置 `channels.discord.accounts.<id>.suppressEmbeds` 可覆盖单个账户。智能体通过消息工具发送时，也可以为单条消息传递 `suppressEmbeds: false`。显式的 Discord `embeds` 载荷不受默认链接预览设置的抑制。

  </Accordion>

  <Accordion title="实时流式预览">
    OpenClaw 可以通过发送临时消息并在文本到达时编辑该消息，以流式传输回复草稿。`channels.discord.streaming.mode` 接受 `off` | `partial` | `block` | `progress`（未设置 `streaming`/旧版 `streamMode` 键时的默认值）。`streamMode` 是旧版别名；运行 `openclaw doctor --fix` 可将持久化配置重写为规范的嵌套 `streaming` 结构。

```json5
{
  channels: {
    discord: {
      streaming: {
        mode: "progress",
        progress: {
          maxLines: 8,
          maxLineChars: 120,
          toolProgress: false,
          commentary: false,
        },
      },
    },
  },
}
```

    - `off` 禁用 Discord 预览编辑。
    - `partial` 会在令牌到达时编辑一条预览消息。
    - `block` 会生成草稿大小的分块；可使用 `streaming.preview.chunk`（`minChars`、`maxChars`、`breakPreference`）调整大小和断点，并限制在 `textChunkLimit` 以内。显式启用分块流式传输时，OpenClaw 会跳过预览流，以避免重复流式传输。
    - `progress` 会保留一个可编辑的状态草稿，直到最终交付。默认情况下，它显示智能体最新前置说明或叙述中的一行，不包含生成的标签、间隔行或工具行。
    - 包含媒体、错误或显式回复的最终消息会取消待处理的预览编辑。
    - `streaming.preview.toolProgress` 在 `partial`/`block` 模式下默认为 `true`。Discord 进度模式默认不显示工具行；设置 `streaming.progress.toolProgress: true` 可选择启用。
    - 设置 `streaming.progress.toolProgress: true` 可添加紧凑的工具/进度行，例如 `🛠️ Bash: run tests` 或 `🔎 Web Search: for "query"`。为保持兼容性，现有的 `progress.label` 或 `progress.labels` 配置会保留之前的工具行默认设置；设置 `toolProgress: false` 可使用自定义标签而不显示工具行。
    - `streaming.progress.commentary`（默认 `false`）可选择在临时进度草稿中显示原始助手评论。默认的前置说明/叙述状态行不受此选项影响。评论在显示前会经过清理，仅临时存在，并且不会改变最终答案的交付。
    - `streaming.progress.maxLineChars` 控制每行进度预览的字符预算。普通文本会在单词边界处缩短；命令和路径详情会保留有用的后缀。
    - `streaming.preview.commandText` / `streaming.progress.commandText` 控制紧凑进度行中的命令/执行详情：`raw`（默认）或 `status`（仅工具标签）。

    隐藏原始命令/执行文本，同时保留紧凑进度行：

    ```json
    {
      "channels": {
        "discord": {
          "streaming": {
            "mode": "progress",
            "progress": {
              "toolProgress": true,
              "commandText": "status"
            }
          }
        }
      }
    }
    ```

    预览流式传输仅支持文本；媒体回复会回退到普通交付方式。

  </Accordion>

  <Accordion title="历史记录、上下文和线程行为">
    服务器历史记录上下文：

    - `channels.discord.historyLimit` 默认值为 `20`
    - 回退：`messages.groupChat.historyLimit`
    - `0` 表示禁用

    私信历史记录控制：

    - `channels.discord.dmHistoryLimit`
    - `channels.discord.dms["<user_id>"].historyLimit`

    线程行为：

    - Discord 线程作为频道会话进行路由，并继承父频道配置，除非进行了覆盖。
    - 线程会话继承父频道的会话级 `/model` 选择，作为仅用于模型的回退；线程本地的 `/model` 选择优先级更高，并且除非启用了记录继承，否则不会复制父级记录历史。
    - `channels.discord.thread.inheritParent`（默认 `false`）可选择让新自动线程使用父级记录进行初始化。按账户覆盖：`channels.discord.accounts.<id>.thread.inheritParent`。
    - 消息工具的表情回应可以解析 `user:<id>` 私信目标。
    - `guilds.<guild>.channels.<channel>.requireMention: false` 会在回复阶段激活回退期间保留。

    频道主题会作为**不受信任的**上下文注入。允许列表只限制谁能触发智能体，并不能作为完整的补充上下文脱敏边界。

  </Accordion>

  <Accordion title="子智能体的线程绑定会话">
    Discord 可以将线程绑定到会话目标，使该线程中的后续消息继续路由到同一会话，包括子智能体会话。

    命令：

    - `/focus <target>` 将当前/新线程绑定到子智能体/会话目标
    - `/unfocus` 移除当前线程绑定
    - `/agents` 显示活动运行和绑定状态
    - `/session idle <duration|off>` 检查/更新聚焦绑定因不活跃而自动取消聚焦的设置
    - `/session max-age <duration|off>` 检查/更新聚焦绑定的硬性最大时长

    配置：

```json5
{
  session: {
    threadBindings: {
      enabled: true,
      idleHours: 24,
      maxAgeHours: 0,
      spawnSessions: true,
      defaultSpawnContext: "fork",
    },
  },
}
```

    注意：

    - `session.threadBindings.*` 是 Discord 和 Telegram 的规范策略。
    - `spawnSessions` 控制为 `sessions_spawn({ thread: true })` 和 ACP 线程派生自动创建/绑定线程。默认值：`true`。
    - `defaultSpawnContext` 控制线程绑定派生的原生子智能体上下文。默认值：`"fork"`。
    - 已弃用的 `spawnSubagentSessions`/`spawnAcpSessions` 键由 `openclaw doctor --fix` 迁移。
    - 如果禁用线程绑定，`/focus` 及相关操作将不可用。

    请参阅[子智能体](/zh-CN/tools/subagents)、[ACP 智能体](/zh-CN/tools/acp-agents)和[配置参考](/zh-CN/gateway/configuration-reference)。

  </Accordion>

  <Accordion title="源消息上的子智能体进度">
    设置 `channels.discord.subagentProgress: true`，可在启动父运行的 Discord 消息上显示后台子级活动。

```json5
{
  channels: {
    discord: {
      subagentProgress: true,
    },
  },
}
```

    子运行处于活动状态时，OpenClaw 会使 Discord 的输入状态最多保持一小时，并随着并发数量变化替换一个计数表情回应（从 `1️⃣` 到 `🔟`）；`🔟` 也表示 10 个或更多。最后一个子级结束后，计数表情回应会被移除。失败、超时或被终止的子级会留下 `🔴` 表情回应。

    此功能需选择启用，并使用固定的内部计时和表情符号默认值。Bot 需要 **Add Reactions** 权限才能提供表情回应反馈。账户级 `channels.discord.accounts.<id>.subagentProgress` 会覆盖顶层值。

  </Accordion>

  <Accordion title="持久 ACP 频道绑定">
    对于稳定的“始终在线”ACP 工作区，请配置以 Discord 对话为目标的顶层类型化 ACP 绑定。

    配置路径：`bindings[]`，包含 `type: "acp"` 和 `match.channel: "discord"`。

```json5
{
  agents: {
    entries: {
      codex: {
        runtime: {
          type: "acp",
          acp: {
            agent: "codex",
            backend: "acpx",
            mode: "persistent",
            cwd: "/workspace/openclaw",
          },
        },
      },
    },
  },
  bindings: [
    {
      type: "acp",
      agentId: "codex",
      match: {
        channel: "discord",
        accountId: "default",
        peer: { kind: "channel", id: "222222222222222222" },
      },
      acp: { label: "codex-main" },
    },
  ],
  channels: {
    discord: {
      guilds: {
        "111111111111111111": {
          channels: {
            "222222222222222222": {
              requireMention: false,
            },
          },
        },
      },
    },
  },
}
```

    注意：

    - `/acp spawn codex --bind here` 会就地绑定当前频道或线程，并使未来消息继续使用同一 ACP 会话。线程消息会继承父频道绑定。
    - 在已绑定的频道或线程中，`/new` 和 `/reset` 会就地重置同一 ACP 会话。临时线程绑定在活动期间可以覆盖目标解析。
    - `spawnSessions` 通过 `--thread auto|here` 控制子线程创建/绑定。

    有关绑定行为的详情，请参阅 [ACP 智能体](/zh-CN/tools/acp-agents)。

  </Accordion>

  <Accordion title="表情回应通知">
    按服务器配置的表情回应通知模式（`guilds.<id>.reactionNotifications`）：

    - `off`
    - `own`（默认）
    - `all`
    - `allowlist`（使用 `guilds.<id>.users`）

    表情回应事件会转换为系统事件，并附加到路由后的 Discord 会话。

  </Accordion>

  <Accordion title="在线状态事件">
    可选择让服务器在某个人类成员从离线状态切换为在线状态时触发路由后的智能体唤醒：

    ```json5
    {
      channels: {
        discord: {
          intents: { presence: true },
          guilds: {
            "111111111111111111": {
              presenceEvents: {
                channelId: "222222222222222222",
                users: ["333333333333333333"], // 可选；进一步缩小频道查看者范围
                reconnectSuppressSeconds: 300, // 可选；新会话静默窗口（0 表示禁用）
                burstLimit: 8, // 可选；每个突发窗口的最大事件数
                burstWindowSeconds: 60, // 可选；滑动突发检测窗口
              },
            },
          },
        },
      },
    }
    ```

    `presenceEvents` 要求路由到的智能体已启用 Heartbeat，并且已在 Discord Developer Portal 中应用的 Bot 页面上启用特权 **Presence Intent**。OpenClaw 根据每个完整的 `GUILD_CREATE` 快照获取当前在线成员，路由观察到的离线到在线转换，并将稍后首次收到的未见成员在线信号也视为该成员刚刚变为可用。该成员可能是在快照之后上线或加入，因此此事件并不表明其此前的确切状态。只有能够查看 `channelId` 的人类用户才符合条件：频道和公开帖子要求对该频道或父级拥有 **View Channel** 权限，而私密帖子还要求具备成员身份或 **Manage Threads** 权限。`users` 可以进一步缩小该受众范围。OpenClaw 会忽略 Bot 和未变化的在线状态，并在 Gateway 网关重启后继续保留每位用户八小时的冷却时间。当 Discord 建立新的 Gateway 网关会话并发送 `READY` 时，OpenClaw 会在 `reconnectSuppressSeconds` 期间（默认 300，`0` 表示禁用）抑制由在线状态产生的事件，同时重建服务器在线状态，以免再次观察到的成员逐个唤醒智能体。它还会按服务器对成功排队的事件进行速率限制：每个 `burstWindowSeconds` 滑动窗口（默认 60）最多 `burstLimit` 个事件（默认 8），并且每次服务器抑制期仅记录一次日志。恢复的会话不视为新会话。Discord 会限制成员数超过 75,000 的服务器快照；在此情况下，OpenClaw 要求先收到明确的离线更新，之后才会问候。系统事件携带不可变的用户、服务器和频道 ID，不会嵌入可变的显示名称。智能体决定是否问候以及如何问候。

  </Accordion>

  <Accordion title="确认表情回应">
    `ackReaction` 会在 OpenClaw 处理入站消息时发送确认表情符号。

    解析顺序：

    - `channels.discord.accounts.<accountId>.ackReaction`
    - `channels.discord.ackReaction`
    - `messages.ackReaction`
    - 智能体身份表情符号回退值（`agents.entries.*.identity.emoji`，否则为 "👀"）

    注意事项：

    - Discord 接受 Unicode 表情符号或自定义表情符号名称。
    - 使用 `""` 可为某个频道或账户禁用该表情回应。

    **范围（`messages.ackReactionScope`）：**

    值：`"all"`（私信 + 群组，包括环境房间事件）、`"direct"`（仅私信）、`"group-all"`（除环境房间事件外的每条群组消息，不包括私信）、`"group-mentions"`（Bot 被提及时的群组；**不包括私信**，默认值）、`"off"` / `"none"`（禁用）。

    <Note>
    默认范围（`"group-mentions"`）不会在私信或环境房间事件中触发确认表情回应。要对入站 Discord 私信和安静房间事件添加确认表情回应，请将 `messages.ackReactionScope` 设置为 `"all"`。
    </Note>

  </Accordion>

  <Accordion title="写入配置">
    默认启用由频道发起的配置写入。这会影响 `/config set|unset` 流程（启用命令功能时）。

    禁用方法：

```json5
{
  channels: {
    discord: {
      configWrites: false,
    },
  },
}
```

  </Accordion>

  <Accordion title="Gateway 网关代理">
    使用 `channels.discord.proxy` 通过 HTTP(S) 代理路由 Discord Gateway 网关 WebSocket 流量和启动时的 REST 查询（应用 ID + 允许列表解析）。
    Discord Gateway 网关 WebSocket 代理需要显式配置；WebSocket 连接不会继承 Gateway 网关进程环境中的代理环境变量。配置 `channels.discord.proxy` 后，启动时的 REST 查询会使用此代理。

```json5
{
  channels: {
    discord: {
      proxy: "http://proxy.example:8080",
    },
  },
}
```

    按账户覆盖：

```json5
{
  channels: {
    discord: {
      accounts: {
        primary: {
          proxy: "http://proxy.example:8080",
        },
      },
    },
  },
}
```

  </Accordion>

  <Accordion title="PluralKit 支持">
    启用 PluralKit 解析，将代理消息映射到系统成员身份：

```json5
{
  channels: {
    discord: {
      pluralkit: {
        enabled: true,
        token: "pk_live_...", // 可选；私密系统需要
      },
    },
  },
}
```

    注意事项：

    - 允许列表可以使用 `pk:<memberId>`
    - 仅当 `channels.discord.dangerouslyAllowNameMatching: true` 时，才按名称/slug 匹配成员显示名称
    - 查询会使用原始消息 ID 调用 PluralKit API
    - 如果查询失败，代理消息将被视为 Bot 消息并丢弃，除非 `allowBots` 允许其通过

  </Accordion>

  <Accordion title="出站提及别名">
    当智能体需要确定性地出站提及已知 Discord 用户时，请使用 `mentionAliases`。键是不含开头 `@` 的用户名；值是 Discord 用户 ID。未知用户名、`@everyone`、`@here` 以及 Markdown 代码跨度内的提及将保持不变。

```json5
{
  channels: {
    discord: {
      mentionAliases: {
        SupportLead: "123456789012345678",
      },
      accounts: {
        ops: {
          mentionAliases: {
            OpsLead: "234567890123456789",
          },
        },
      },
    },
  },
}
```

  </Accordion>

  <Accordion title="在线状态配置">
    设置状态或活动字段，或者启用自动在线状态后，在线状态更新将生效。

    仅状态：

```json5
{
  channels: {
    discord: {
      status: "idle",
    },
  },
}
```

    活动（设置 `activity` 时，自定义状态是默认活动类型）：

```json5
{
  channels: {
    discord: {
      activity: "专注时间",
      activityType: 4,
    },
  },
}
```

    直播：

```json5
{
  channels: {
    discord: {
      activity: "实时编码",
      activityType: 1,
      activityUrl: "https://twitch.tv/openclaw",
    },
  },
}
```

    活动类型映射：

    - 0：正在玩
    - 1：直播（需要 `activityUrl`；而 `activityUrl` 又需要 `activityType: 1`）
    - 2：正在听
    - 3：正在看
    - 4：自定义（使用活动文本作为状态内容；表情符号可选）
    - 5：正在竞赛

    自动在线状态（运行时健康信号）：

```json5
{
  channels: {
    discord: {
      autoPresence: {
        enabled: true,
        intervalMs: 30000,
        minUpdateIntervalMs: 15000,
        exhaustedText: "令牌已耗尽",
      },
    },
  },
}
```

    自动在线状态将运行时可用性映射到 Discord 状态：健康 => 在线，降级或未知 => 空闲，已耗尽或不可用 => 请勿打扰。默认值：`intervalMs` 为 30000，`minUpdateIntervalMs` 为 15000（必须小于或等于 `intervalMs`）。可选文本覆盖项：

    - `autoPresence.healthyText`
    - `autoPresence.degradedText`
    - `autoPresence.exhaustedText`（支持 `{reason}` 占位符）

  </Accordion>

  <Accordion title="Discord 中的审批">
    Discord 支持在私信中使用按钮处理审批，也可以选择在发起操作的频道中发布审批提示。

    配置路径：

    - `channels.discord.execApprovals.enabled`
    - `channels.discord.execApprovals.approvers`（可选；在可行时回退到 `commands.ownerAllowFrom`）
    - `channels.discord.execApprovals.target`（`dm` | `channel` | `both`，默认值：`dm`）
    - `agentFilter`、`sessionFilter`、`cleanupAfterResolve`

    当 `enabled` 未设置或为 `"auto"`，并且至少能解析出一名审批者时，Discord 会自动启用原生 Exec 审批；审批者可从 `execApprovals.approvers` 或 `commands.ownerAllowFrom` 解析。Discord 不会从频道 `allowFrom`、旧版 `dm.allowFrom` 或私信 `defaultTo` 推断 Exec 审批者。将 `enabled: false` 设置为禁用，可明确禁止 Discord 充当原生审批客户端。

    对于 `/diagnostics` 和 `/export-trajectory` 等仅限所有者执行的敏感群组命令，OpenClaw 会私下发送审批提示和最终结果。当发起调用的所有者拥有 Discord 所有者路由时，它会先尝试发送 Discord 私信；否则会回退到 `commands.ownerAllowFrom` 中第一个可用的所有者路由，例如 Telegram。

    当 `target` 为 `channel` 或 `both` 时，审批提示在频道中可见。只有解析出的审批者可以使用按钮；其他用户会收到仅自己可见的拒绝消息。审批提示包含命令文本，因此仅应在可信频道中启用频道投递。如果无法从会话键推导出频道 ID，OpenClaw 会回退到私信投递。

    Discord 会呈现其他聊天渠道共用的审批按钮；Discord 原生适配器主要添加审批者私信路由和频道扇出功能。当这些按钮存在时，它们是主要的审批用户体验；仅当工具结果表明聊天审批不可用或手动审批是唯一途径时，OpenClaw 才应包含手动 `/approve` 命令。如果 Discord 原生审批运行时未激活，OpenClaw 会继续显示本地确定性的 `/approve <id> <decision>` 提示。如果运行时已激活，但无法将原生卡片投递到任何目标，OpenClaw 会在同一聊天中发送回退通知，其中包含待处理审批中的确切 `/approve` 命令。

    Gateway 网关身份验证和审批解析遵循共享的 Gateway 网关客户端契约（`plugin:` ID 通过 `plugin.approval.resolve` 解析；其他 ID 通过 `exec.approval.resolve` 解析）。默认情况下，审批会在 30 分钟后过期。

    请参阅 [Exec 审批](/zh-CN/tools/exec-approvals)。

  </Accordion>
</AccordionGroup>

## 工具和操作门控

Discord 消息操作涵盖消息传递、频道管理、内容审核、在线状态和元数据。

核心示例：

- 消息传递：`sendMessage`、`readMessages`、`editMessage`、`deleteMessage`、`threadReply`
- 表情回应：`react`、`reactions`、`emojiList`
- 内容审核：`timeout`、`kick`、`ban`
- 在线状态：`setPresence`

`event-create` 操作接受可选的 `image` 参数（URL 或本地文件路径），用于设置定时事件的封面图像。

操作门控位于 `channels.discord.actions.*` 下。

默认门控行为：

| 操作组                                                                                                                                                                   | 默认值 |
| ------------------------------------------------------------------------------------------------------------------------------------------------------------------------ | ------ |
| reactions, messages, threads, pins, polls, search, memberInfo, roleInfo, channelInfo, channels, voiceStatus, events, stickers, emojiUploads, stickerUploads, permissions | 启用   |
| roles                                                                                                                                                                    | 禁用   |
| moderation                                                                                                                                                               | 禁用   |
| presence                                                                                                                                                                 | 禁用   |

## Components v2 UI

OpenClaw 使用 Discord components v2 实现 Exec 审批和跨上下文标记。Discord 消息操作也可以接受 `components` 以使用自定义 UI（高级功能；需要通过 discord 工具构造组件负载），旧版 `embeds` 仍然可用，但不建议使用。

- `channels.discord.ui.components.accentColor` 设置 Discord 组件容器使用的强调色（十六进制）。每个账户：`channels.discord.accounts.<id>.ui.components.accentColor`。
- `channels.discord.agentComponents.ttlMs` 控制已发送的 Discord 组件回调保持注册的时长（默认 `1800000`，最长 `86400000`）。每个账户：`channels.discord.accounts.<id>.agentComponents.ttlMs`。
- 存在 components v2 时，`embeds` 会被忽略。
- 默认禁止普通 URL 预览。当单个出站链接需要展开时，请在消息操作上设置 `suppressEmbeds: false`。

示例：

```json5
{
  channels: {
    discord: {
      ui: {
        components: {
          accentColor: "#5865F2",
        },
      },
    },
  },
}
```

## 语音

Discord 有两种不同的语音界面：实时 **语音频道**（持续对话）和 **语音消息附件**（波形预览格式）。Gateway 网关同时支持这两者。

### 语音频道

设置检查清单：

1. 在 Discord Developer Portal 中启用 Message Content Intent。
2. 使用角色/用户允许列表时，启用 Server Members Intent。
3. 使用 `bot` 和 `applications.commands` 权限范围邀请 Bot。
4. 在目标语音频道中授予 Connect、Speak、Send Messages 和 Read Message History 权限。
5. 启用原生命令（`commands.native` 或 `channels.discord.commands.native`）。
6. 配置 `channels.discord.voice`。

使用 `/vc join|leave|status` 控制会话。该命令使用账户的默认智能体，并遵循与其他 Discord 命令相同的允许列表和群组策略规则。

```bash
/vc join channel:<voice-channel-id>
/vc status
/vc leave
```

要在加入前检查 Bot 的有效权限：

```bash
openclaw channels capabilities --channel discord --target channel:<voice-channel-id>
```

自动加入示例：

```json5
{
  channels: {
    discord: {
      voice: {
        enabled: true,
        model: "openai/gpt-5.6-sol",
        autoJoin: [
          {
            guildId: "123456789012345678",
            channelId: "234567890123456789",
          },
        ],
        allowedChannels: [
          {
            guildId: "123456789012345678",
            channelId: "234567890123456789",
          },
        ],
        daveEncryption: true,
        decryptionFailureTolerance: 24,
        connectTimeoutMs: 30000,
        reconnectGraceMs: 15000,
        realtime: {
          provider: "openai",
          model: "gpt-realtime-2.1",
          speakerVoice: "cedar",
        },
      },
    },
  },
}
```

注意事项：

- 对于仅文本配置，Discord 语音功能是选择启用的；设置 `channels.discord.voice.enabled=true`（或保留现有的 `channels.discord.voice` 块）即可启用 `/vc` 命令、语音运行时和 `GuildVoiceStates` Gateway 网关意图。`channels.discord.intents.voiceStates` 可显式覆盖意图订阅；不设置则遵循实际的语音启用状态。
- `voice.mode` 控制对话路径。默认为 `agent-proxy`：实时语音前端负责轮次时序、打断和播放，通过 `openclaw_agent_consult` 将实质性工作委托给路由到的 OpenClaw 智能体，并将结果视为该说话者在 Discord 中输入的提示。`stt-tts` 保留较旧的批量 STT 加 TTS 流程。`bidi` 允许实时模型直接对话，同时通过 `openclaw_agent_consult` 提供 OpenClaw 大脑。
- `voice.agentSession` 控制哪个 OpenClaw 对话接收语音轮次。不设置时使用语音频道自身的会话；也可设置 `{ mode: "target", target: "channel:<text-channel-id>" }`，使语音频道充当现有 Discord 文本频道会话（例如 `#maintainers`）的麦克风/扬声器扩展。
- `voice.model` 覆盖用于 Discord 语音响应和实时咨询的 OpenClaw 智能体大脑。不设置则继承路由到的智能体模型。它与 `voice.realtime.model` 相互独立。
- `voice.followUsers` 允许 Bot 跟随选定用户加入、切换和离开 Discord 语音频道。参阅[在语音频道中跟随用户](#follow-users-in-voice)。
- `agent-proxy` 通过 `discord-voice` 路由语音，该路径保留说话者和目标会话的常规所有者/工具授权，但会隐藏智能体的 `tts` 工具，因为播放由 Discord 语音负责。默认情况下，`agent-proxy` 为所有者说话者提供与所有者等效的完整工具访问权限（`voice.realtime.toolPolicy: "owner"`），并强烈倾向于在给出实质性回答前咨询 OpenClaw 智能体（`voice.realtime.consultPolicy: "always"`）。在默认的 `always` 模式下，实时层不会在咨询答案前自动说出填充语；它会捕获并转录语音，然后朗读路由到的 OpenClaw 答案。如果在 Discord 仍在播放第一个答案时有多个强制咨询答案完成，后续精确语音答案会排队等待播放空闲，而不会在句子中途替换语音。
- 在 `stt-tts` 模式下，STT 使用 `tools.media.audio`；`voice.model` 不影响转录。
- 在实时模式下，`voice.realtime.provider`、`voice.realtime.model` 和 `voice.realtime.speakerVoice` 用于配置实时音频会话。要将 OpenAI Realtime 2.1 与 Codex 大脑搭配使用，请使用 `voice.realtime.model: "gpt-realtime-2.1"` 和 `voice.model: "openai/gpt-5.6-sol"`。
- 默认情况下，实时语音模式会在实时提供商指令中包含较小的 `IDENTITY.md`、`USER.md` 和 `SOUL.md` 配置文件，使快速直接轮次与路由到的 OpenClaw 智能体保持相同的身份、用户背景和角色设定。将 `voice.realtime.bootstrapContextFiles` 设置为一个子集可自定义此行为，或设置 `[]` 将其禁用。仅支持这些配置文件；`AGENTS.md` 仍保留在常规智能体上下文中。注入的配置文件上下文不能取代 `openclaw_agent_consult`，后者仍用于工作区工作、当前事实、记忆查询或工具支持的操作。
- 在 OpenAI `agent-proxy` 实时模式下，唤醒名称门控默认会根据房间情况调整：只有一名用户时，无需唤醒名称即可自然交谈；有两名或更多用户时，每个轮次的开头或结尾必须包含唤醒名称。其他 Bot 不计入人数。设置 `voice.realtime.requireWakeName: true` 可始终要求唤醒名称，设置 `false` 则从不要求。配置的唤醒名称必须由一到两个单词组成。如果未设置 `voice.realtime.wakeNames`，OpenClaw 会使用路由到的智能体 `name` 加 `OpenClaw`，并在无法使用时回退到智能体 ID 加 `OpenClaw`。启用唤醒名称门控后，实时提供商的自动响应会被禁用，已接受的轮次将通过 OpenClaw 智能体咨询路径进行路由；如果在最终转录到达前，从部分转录中识别出位于开头的唤醒名称，还会给出简短的语音确认。该策略会实时跟随用户的加入和离开，无需重新连接语音。
- OpenAI 实时提供商接受当前的 Realtime 2 事件名称，以及用于输出音频和转录事件的旧版 Codex 兼容别名，因此兼容的提供商快照即使出现差异，也不会丢失助手音频。
- `voice.realtime.bargeIn` 控制 Discord 的说话开始事件是否中断正在进行的实时播放。如果未设置，则遵循实时提供商的输入音频中断设置。
- `voice.realtime.minBargeInAudioEndMs` 控制 OpenAI 实时插话截断音频前，助手至少需要播放多长时间。默认值：`250`。在低回声房间中，可设置 `0` 以立即中断；对于回声较大的扬声器设置，则可提高该值。
- `voice.tts` 仅覆盖 `stt-tts` 语音播放的 `tts`；实时模式改用 `voice.realtime.speakerVoice`。要在 Discord 播放中使用 OpenAI 语音，请设置 `voice.tts.provider: "openai"`，并在 `voice.tts.providers.openai.speakerVoice` 下选择一种文本转语音音色。在当前 OpenAI TTS 模型中，`cedar` 是一个不错的男性音色选项。
- 每频道的 Discord `systemPrompt` 覆盖设置会应用于该语音频道的语音转录轮次。
- 当 OpenClaw 加入语音频道时，路由到的智能体会话会收到一条包含当前参与者名单的静默系统事件。之后参与者的加入和离开会更新该会话，但不会触发未经请求的语音回复；Discord 显示名称会被视为不可信标签。已授权的语音轮次也会收到最新的参与者名单快照。
- 语音转录轮次和 `/vc` 命令使用 `commands.ownerAllowFrom` 中的 Discord 条目确定所有者状态。如果未配置 Discord 命令所有者，所选 Discord 账号的 `allowFrom`（或旧版 `dm.allowFrom`）仍可授权语音访问，但不会授予所有者状态。智能体工具的可见性遵循路由会话所配置的工具策略。
- 如果 `voice.autoJoin` 中同一服务器有多个条目，OpenClaw 会加入该服务器最后配置的频道。
- `voice.allowedChannels` 是可选的驻留允许列表。不设置时，允许 `/vc join` 进入任何已授权的 Discord 语音频道。设置后，`/vc join`、启动时自动加入和 Bot 语音状态切换仅限于列出的 `{ guildId, channelId }` 条目。将其设置为空数组可拒绝加入所有 Discord 语音频道。如果 Discord 将 Bot 移至允许列表之外，OpenClaw 会离开该频道，并在存在已配置的自动加入目标时重新加入该目标。
- `voice.daveEncryption` 和 `voice.decryptionFailureTolerance` 会原样传递给 `@discordjs/voice` 加入选项；上游默认值为 `daveEncryption=true` 和 `decryptionFailureTolerance=24`。
- OpenClaw 使用内置的 `libopus-wasm` 编解码器接收 Discord 语音并实时播放原始 PCM。它随附固定版本的 libopus WebAssembly 构建，无需原生 opus 插件。
- `voice.connectTimeoutMs` 控制 `/vc join` 和自动加入尝试最初等待 `@discordjs/voice` Ready 状态的时长。默认值：`30000`。
- `voice.reconnectGraceMs` 控制 OpenClaw 在销毁已断开连接的语音会话前，等待该会话开始重新连接的时长。默认值：`15000`。
- 在 `stt-tts` 模式下，语音播放不会仅仅因为其他用户开始说话而停止。为避免反馈循环，OpenClaw 会在 TTS 播放期间忽略新的语音捕获；请在播放结束后再说出下一轮内容。实时模式会将说话开始事件作为插话信号转发给实时提供商。
- 在实时模式下，扬声器声音进入开启的麦克风所产生的回声可能会被视为插话并中断播放。对于回声较大的 Discord 房间，请设置 `voice.realtime.providers.openai.interruptResponseOnInputAudio: false`，阻止 OpenAI 因输入音频而自动中断。如果仍希望 Discord 说话开始事件中断正在进行的播放，请添加 `voice.realtime.bargeIn: true`。OpenAI 实时桥接器会将短于 `voice.realtime.minBargeInAudioEndMs` 的播放截断视为可能的回声/噪声并忽略，将其记录为已跳过，而不是清除 Discord 播放。
- `voice.captureSilenceGraceMs` 控制 Discord 报告说话者停止说话后，OpenClaw 等待多长时间再最终确定该音频片段并交给 STT。默认值：`2000`；如果 Discord 将正常停顿切割成断断续续的部分转录，请提高该值。
- 当选择 ElevenLabs 作为 TTS 提供商时，Discord 语音播放使用流式 TTS，并直接从提供商响应流开始播放。不支持流式传输的提供商会回退到合成临时文件路径。
- OpenClaw 会监控接收解密失败，并在短时间内反复失败后，通过离开并重新加入语音频道自动恢复。
- 如果更新后接收日志反复显示 `DecryptionFailed(UnencryptedWhenPassthroughDisabled)`，请收集依赖项报告和日志。内置的 `@discordjs/voice` 版本包含 discord.js PR #11449 中的上游填充修复，该修复关闭了 discord.js issue #11419。
- 当 OpenClaw 最终确定捕获的说话者片段时，出现 `The operation was aborted` 接收事件是正常现象；它们是详细诊断信息，而不是警告。
- 详细的 Discord 语音日志会为每个已接受的说话者片段提供有长度限制的单行 STT 转录预览，使调试时能够同时看到用户端和智能体回复端，而不会转储无限长度的转录文本。
- 在 `agent-proxy` 模式下，强制咨询回退会跳过可能不完整的转录片段，例如以 `...` 结尾的文本、以 “and” 等连接词结尾的文本，以及 “be right back” 或 “bye” 等明显无法执行操作的结束语。当此机制阻止过时的排队答案时，日志会显示 `forced agent consult skipped reason=...`。

### 在语音频道中跟随用户

如果希望 Discord 语音 Bot 跟随一个或多个已知 Discord 用户，而不是在启动时加入固定频道或等待 `/vc join`，请使用 `voice.followUsers`。

```json5
{
  channels: {
    discord: {
      voice: {
        enabled: true,
        followUsersEnabled: true,
        followUsers: ["discord:123456789012345678"],
        allowedChannels: [
          {
            guildId: "123456789012345678",
            channelId: "234567890123456789",
          },
        ],
      },
    },
  },
}
```

行为：

- `followUsers` 接受原始 Discord 用户 ID 和 `discord:<id>` 值。OpenClaw 会在匹配语音状态事件前规范化这两种形式。
- 配置 `followUsers` 后，`followUsersEnabled` 默认为 `true`。将其设为 `false` 可保留已保存的列表，但停止自动跟随语音。
- `followUsers` 仅控制语音驻留。它不授予发言者访问权限或所有者权限；请分别配置 `commands.ownerAllowFrom` 以及服务器或频道的用户和角色。
- 当被跟随用户加入允许的语音频道时，OpenClaw 会加入该频道。当用户移动时，OpenClaw 会随之移动。当当前被跟随用户断开连接时，OpenClaw 会离开。
- 如果同一服务器中有多个被跟随用户，并且当前被跟随用户离开，OpenClaw 会先移至另一个受跟踪的被跟随用户所在的频道，再考虑离开该服务器。如果多个被跟随用户同时移动，则以最后观察到的语音状态事件为准。
- `allowedChannels` 仍然适用。处于不允许频道中的被跟随用户会被忽略，由跟随功能所有的会话会移至另一个被跟随用户处或离开。
- OpenClaw 会在启动时以及按有界间隔协调遗漏的语音状态事件。协调过程会对已配置的服务器进行采样，并限制每次运行的 REST 查询次数，因此非常大的 `followUsers` 列表可能需要多个间隔才能收敛。
- 如果 Discord 或管理员在机器人跟随用户期间移动机器人，OpenClaw 会重建语音会话，并在目标位置被允许时保留跟随所有权。如果机器人被移至 `allowedChannels` 之外，OpenClaw 会离开，并在存在已配置目标时重新加入该目标。
- DAVE 接收恢复可能会在重复解密失败后离开并重新加入同一频道。由跟随功能所有的会话会在该恢复路径中保留跟随所有权，因此被跟随用户之后断开连接时，机器人仍会离开频道。

在以下加入模式中选择：

- 对于个人或操作员设置，如果希望你进入语音时机器人也自动进入，请使用 `followUsers`。
- 对于即使没有受跟踪用户进入语音也应保持在线的固定房间机器人，请使用 `autoJoin`。
- 对于一次性加入或不应自动保持语音在线状态的房间，请使用 `/vc join`。

Discord 语音编解码器：

- 语音接收日志会显示 `discord voice: opus decoder: libopus-wasm`。
- 实时播放使用同一个内置 `libopus-wasm` 软件包，将原始 48 kHz 立体声 PCM 编码为 Opus，然后再将数据包交给 `@discordjs/voice`。
- 文件和提供商流播放使用 ffmpeg 转码为原始 48 kHz 立体声 PCM，然后使用 `libopus-wasm` 生成发送至 Discord 的 Opus 数据包流。

STT 加 TTS 流水线：

- Discord PCM 捕获内容会转换为临时 WAV 文件。
- `tools.media.audio` 处理 STT，例如 `openai/gpt-4o-mini-transcribe`。
- 转录文本会经由 Discord 入口和路由发送；与此同时，响应 LLM 会按照一项语音输出策略运行，该策略会隐藏智能体的 `tts` 工具并要求返回文本，因为最终的 TTS 播放由 Discord 语音负责。
- 设置 `voice.model` 后，它仅覆盖此语音频道轮次使用的响应 LLM。
- `voice.tts` 会合并并覆盖 `tts`；支持流式传输的提供商会直接向播放器馈送内容，否则会在已加入的频道中播放生成的音频文件。

默认的智能体代理语音频道会话示例：

```json5
{
  channels: {
    discord: {
      voice: {
        enabled: true,
        model: "openai/gpt-5.6-sol",
        followUsersEnabled: true,
        followUsers: ["123456789012345678"],
        realtime: {
          provider: "openai",
          model: "gpt-realtime-2.1",
          speakerVoice: "cedar",
        },
      },
    },
  },
}
```

如果没有 `voice.agentSession` 块，每个语音频道都会获得自己的已路由 OpenClaw 会话。例如，`/vc join channel:234567890123456789` 会与该 Discord 语音频道对应的会话交互。实时模型仅充当语音前端；实质性请求会转交给已配置的 OpenClaw 智能体。如果实时模型未调用咨询工具便生成最终转录文本，OpenClaw 会强制执行咨询作为后备措施，使默认行为仍与直接和智能体交谈一致。

旧版 STT 加 TTS 示例：

```json5
{
  channels: {
    discord: {
      voice: {
        enabled: true,
        mode: "stt-tts",
        model: "openai/gpt-5.4-mini",
        tts: {
          provider: "openai",
          providers: {
            openai: {
              model: "gpt-4o-mini-tts",
              speakerVoice: "cedar",
            },
          },
        },
      },
    },
  },
}
```

实时双向示例：

```json5
{
  channels: {
    discord: {
      voice: {
        enabled: true,
        mode: "bidi",
        model: "openai/gpt-5.6-sol",
        realtime: {
          provider: "openai",
          model: "gpt-realtime-2.1",
          speakerVoice: "cedar",
          toolPolicy: "safe-read-only",
          consultPolicy: "always",
        },
      },
    },
  },
}
```

将语音用作现有 Discord 频道会话的扩展：

```json5
{
  channels: {
    discord: {
      voice: {
        enabled: true,
        mode: "agent-proxy",
        model: "openai/gpt-5.6-sol",
        agentSession: {
          mode: "target",
          target: "channel:123456789012345678",
        },
        realtime: {
          provider: "openai",
          model: "gpt-realtime-2.1",
          speakerVoice: "cedar",
        },
      },
    },
  },
}
```

在 `agent-proxy` 模式下，机器人会加入已配置的语音频道，但 OpenClaw 智能体轮次使用目标频道正常路由的会话和智能体。实时语音会话会将返回的结果在语音频道中朗读出来。监督智能体仍可根据其工具策略使用常规消息工具，包括在适当时另行发送 Discord 消息。

当委派的 OpenClaw 运行处于活动状态时，新的 Discord 语音转录文本会在启动另一个智能体轮次前被视为实时运行控制输入。“状态”“取消那个操作”“使用更小的修复方案”或“完成后也检查测试”等短语会被分类为当前会话的状态、取消、Steering queue 或后续输入。状态、取消、已接受的 Steering queue 和后续结果会在语音频道中朗读出来，使调用者知道 OpenClaw 是否已处理该请求。

可用的目标形式：

- `target: "channel:123456789012345678"` 通过 Discord 文本频道会话进行路由。
- `target: "123456789012345678"` 被视为频道目标。
- `target: "dm:123456789012345678"` 或 `target: "user:123456789012345678"` 通过相应的私信会话进行路由。

回声较严重时的 OpenAI Realtime 示例：

```json5
{
  channels: {
    discord: {
      voice: {
        enabled: true,
        mode: "bidi",
        model: "openai/gpt-5.6-sol",
        realtime: {
          provider: "openai",
          model: "gpt-realtime-2.1",
          speakerVoice: "cedar",
          bargeIn: true,
          minBargeInAudioEndMs: 500,
          consultPolicy: "always",
          providers: {
            openai: {
              interruptResponseOnInputAudio: false,
            },
          },
        },
      },
    },
  },
}
```

当模型通过开启的麦克风听到自己在 Discord 中播放的声音，但你仍希望通过说话打断它时，请使用此配置。OpenClaw 会阻止 OpenAI 因原始输入音频而自动中断，同时 `bargeIn: true` 允许 Discord 发言者开始事件和已处于活动状态的发言者音频在下一段捕获的轮次到达 OpenAI 前取消当前实时响应。当 `audioEndMs` 低于 `minBargeInAudioEndMs` 时，过早的插话信号会被视为可能的回声或噪声并予以忽略，以免模型在第一个播放帧出现时便中断。

预期的语音日志：

- 加入时：`discord voice: joining ... voiceSession=... supervisorSession=... agentSessionMode=... voiceModel=... realtimeModel=...`
- 实时处理开始时：`discord voice: realtime bridge starting ... autoRespond=false interruptResponse=false bargeIn=false minBargeInAudioEndMs=...`
- 收到发言者音频时：`discord voice: realtime speaker turn opened ...`、`discord voice: realtime input audio started ... outputAudioMs=... outputActive=...` 和 `discord voice: realtime speaker turn closed ... chunks=... discordBytes=... realtimeBytes=... interruptedPlayback=...`
- 跳过过期语音时：`discord voice: realtime forced agent consult skipped reason=incomplete-transcript ...` 或 `reason=non-actionable-closing ...`
- 实时响应完成时：`discord voice: realtime audio playback finishing reason=response.done ... audioMs=... chunks=...`
- 播放停止或重置时：`discord voice: realtime audio playback stopped reason=... audioMs=... elapsedMs=... chunks=...`
- 实时咨询时：`discord voice: realtime consult requested ... voiceSession=... supervisorSession=... question=...`
- 智能体回答时：`discord voice: agent turn answer ...`
- 精确语音进入队列时：`discord voice: realtime exact speech queued ... queued=... outputAudioMs=... outputActive=...`，随后是 `discord voice: realtime exact speech dequeued reason=player-idle ...`
- 检测到插话时：`discord voice: realtime barge-in detected source=speaker-start ...` 或 `discord voice: realtime barge-in detected source=active-speaker-audio ...`，随后是 `discord voice: realtime barge-in requested reason=... outputAudioMs=... outputActive=...`
- 实时中断时：`discord voice: realtime model interrupt requested client:response.cancel reason=barge-in`，随后是 `discord voice: realtime model audio truncated client:conversation.item.truncate reason=barge-in audioEndMs=...` 或 `discord voice: realtime model interrupt confirmed server:response.done status=cancelled ...`
- 忽略回声或噪声时：`discord voice: realtime model interrupt ignored client:conversation.item.truncate.skipped reason=barge-in audioEndMs=0 minAudioEndMs=250`
- 插话功能已禁用时：`discord voice: realtime capture ignored during playback (barge-in disabled) ...`
- 播放空闲时：`discord voice: realtime barge-in ignored reason=... outputActive=false ... playbackChunks=0`

要调试音频被截断的问题，请将实时语音日志按时间线阅读：

1. `realtime audio playback started` 表示 Discord 已开始播放助手音频。从此时起，桥接器开始统计助手输出分块、Discord PCM 字节数、提供商实时字节数和合成音频时长。
2. `realtime speaker turn opened` 表示一名 Discord 发言者开始活动。如果播放已处于活动状态且 `bargeIn` 已启用，之后可能会出现 `barge-in detected source=speaker-start`。
3. `realtime input audio started` 表示收到该发言轮次的第一个实际音频帧。此处出现 `outputActive=true` 或非零的 `outputAudioMs`，表示助手播放仍处于活动状态时，麦克风正在发送输入。
4. `barge-in detected source=active-speaker-audio` 表示 OpenClaw 在助手播放处于活动状态时检测到实时发言者音频。这有助于区分真正的中断与没有有效音频的 Discord 发言者开始事件。
5. `barge-in requested reason=...` 表示 OpenClaw 已要求实时提供商取消或截断当前响应。其中包含 `outputAudioMs`、`outputActive` 和 `playbackChunks`，以便查看中断前实际已播放了多少助手音频。
6. `realtime audio playback stopped reason=...` 是本地 Discord 播放重置点。原因会说明是谁停止了播放：`barge-in`、`player-idle`、`provider-clear-audio`、`forced-agent-consult`、`stream-close` 或 `session-close`。
7. `realtime speaker turn closed` 汇总了捕获的输入轮次。`chunks=0` 或 `hasAudio=false` 表示发言轮次已开始，但没有可用音频到达实时桥接器。`interruptedPlayback=true` 表示该输入轮次与助手输出重叠，并触发了插话逻辑。

可用字段：

- `outputAudioMs`：实时提供商在该日志行之前生成的助手音频时长。
- `audioMs`：OpenClaw 在播放停止前统计的助手音频时长。
- `elapsedMs`：播放流或发言轮次从开始到关闭之间的实际时钟时间。
- `discordBytes`：发送至 Discord 语音或从中接收的 48 kHz 立体声 PCM 字节数。
- `realtimeBytes`：发送至实时提供商或从中接收的提供商格式 PCM 字节数。
- `playbackChunks`：当前响应中转发至 Discord 的助手音频分块数。
- `sinceLastAudioMs`：最后捕获的发言者音频帧与发言轮次关闭之间的时间间隔。

常见模式：

- 立即中断，并伴有 `source=active-speaker-audio`、较小的 `outputAudioMs`，且同一用户仍在附近，通常表明扬声器回声进入了麦克风。提高 `voice.realtime.minBargeInAudioEndMs`、降低扬声器音量、使用耳机，或设置 `voice.realtime.providers.openai.interruptResponseOnInputAudio: false`。
- `source=speaker-start` 后紧跟 `speaker turn closed ... hasAudio=false`，表示 Discord 报告扬声器开始发声，但没有音频传到 OpenClaw。这可能是短暂的 Discord 语音事件、噪声门行为，或客户端短暂开启了麦克风。
- 如果出现 `audio playback stopped reason=stream-close`，但附近没有插话或 `provider-clear-audio`，表示本地 Discord 播放流意外结束。检查此前的提供商和 Discord 播放器日志。
- `capture ignored during playback (barge-in disabled)` 表示 OpenClaw 在助手音频播放期间有意丢弃了输入。如果希望语音中断播放，请启用 `voice.realtime.bargeIn`。
- `barge-in ignored ... outputActive=false` 表示 Discord 或提供商的 VAD 报告检测到语音，但 OpenClaw 没有可中断的活动播放。这不应中断音频。

凭据按组件解析：`voice.model` 使用 LLM 路由身份验证，`tools.media.audio` 使用 STT 身份验证，`tts`/`voice.tts` 使用 TTS 身份验证，而 `voice.realtime.providers` 使用实时提供商身份验证或该提供商的常规身份验证配置。

### 语音消息

Discord 语音消息会显示波形预览，并且需要 OGG/Opus 音频。OpenClaw 会自动生成波形，但需要 Gateway 网关主机上安装 `ffmpeg` 和 `ffprobe`，以便检查和转换音频。

- 提供一个**本地文件路径**（不接受 URL）。
- 省略文本内容（Discord 不接受在同一有效载荷中同时包含文本和语音消息）。
- 接受任何音频格式；OpenClaw 会按需将其转换为 OGG/Opus。

```bash
message(action="send", channel="discord", target="channel:123", path="/path/to/audio.mp3", asVoice=true)
```

## 故障排查

<AccordionGroup>
  <Accordion title="使用了不允许的 Intent，或 Bot 看不到服务器消息">

    - 启用 Message Content Intent
    - 依赖用户/成员解析时，启用 Server Members Intent
    - 更改 Intent 后重启 Gateway 网关

  </Accordion>

  <Accordion title="服务器消息意外被阻止">

    - 验证 `groupPolicy`
    - 验证 `channels.discord.guilds` 下的服务器允许列表
    - 如果存在服务器 `channels` 映射，则仅允许其中列出的频道
    - 验证 `requireMention` 的行为和提及模式

    实用检查：

```bash
openclaw doctor
openclaw channels status --probe
openclaw logs --follow
```

  </Accordion>

  <Accordion title="Require mention 为 false，但仍被阻止">
    常见原因：

    - 使用了 `groupPolicy="allowlist"`，但没有匹配的服务器/频道允许列表
    - `requireMention` 配置在了错误位置（必须位于 `channels.discord.guilds` 或某个频道条目下）
    - 发送者被服务器/频道的 `users` 允许列表阻止

  </Accordion>

  <Accordion title="长时间运行的 Discord 轮次或重复回复">

    典型日志：

    - `Slow listener detected ...`
    - `stuck session: sessionKey=agent:...:discord:... state=processing ...`

    Discord 不会对排队的智能体轮次应用由频道控制的超时。消息监听器会立即移交任务，排队的 Discord 运行会保持每个会话的顺序，直到会话/工具/运行时生命周期完成或中止工作。

  </Accordion>

  <Accordion title="Gateway 网关元数据查询超时警告">
    OpenClaw 会在连接前获取 Discord 的 `/gateway/bot` 元数据。发生短暂故障时，会回退到 Discord 的默认 Gateway 网关 URL，并对相关日志进行速率限制。

    元数据超时默认为 30 秒。对于特殊的主机环境，可以使用 `OPENCLAW_DISCORD_GATEWAY_INFO_TIMEOUT_MS` 覆盖它。

  </Accordion>

  <Accordion title="Gateway 网关 READY 超时重启">
    OpenClaw 会在启动期间和运行时重新连接后等待 Discord Gateway 网关的 `READY` 事件。采用交错启动的多账户设置可能需要比默认值更长的启动 READY 等待窗口。

    启动等待时间为 15 秒，运行时重新连接等待时间为 30 秒。对于特殊的主机环境，仍可使用 `OPENCLAW_DISCORD_READY_TIMEOUT_MS` 和 `OPENCLAW_DISCORD_RUNTIME_READY_TIMEOUT_MS`。

  </Accordion>

  <Accordion title="权限审计不匹配">
    `channels status --probe` 权限检查仅适用于数字频道 ID。

    如果使用 slug 键，运行时匹配仍可正常工作，但探测无法完整验证权限。

  </Accordion>

  <Accordion title="私信和配对问题">

    - 私信已禁用：`channels.discord.dm.enabled=false`
    - 私信策略已禁用：`channels.discord.dmPolicy="disabled"`（旧版：`channels.discord.dm.policy`）
    - 在 `pairing` 模式下等待配对批准

  </Accordion>

  <Accordion title="Bot 间循环">
    默认情况下，会忽略由 Bot 发出的消息。

    如果设置了 `channels.discord.allowBots=true`，请使用严格的提及和允许列表规则来避免循环行为。
    建议使用 `channels.discord.allowBots="mentions"`，仅接受提及该 Bot 的 Bot 消息。

    OpenClaw 还提供共享的 [Bot 循环保护](/zh-CN/channels/bot-loop-protection)。每当 `allowBots` 允许 Bot 发出的消息进入分派流程时，Discord 会将入站事件映射为 `(account, channel, bot pair)` 事实；当这对 Bot 超过配置的事件预算后，通用配对防护机制就会抑制它们之间的交互。该防护机制可防止失控的双 Bot 循环，这类循环过去只能依靠 Discord 速率限制来停止；它不会影响单 Bot 部署，也不会影响未超过预算的一次性 Bot 回复。

    默认设置（设置 `allowBots` 时生效）：

    - `maxEventsPerWindow: 20` -- Bot 对可在滑动窗口内交换 20 条消息
    - `windowSeconds: 60` -- 滑动窗口长度
    - `cooldownSeconds: 60` -- 一旦触发预算限制，任一方向后续的每条 Bot 间消息都会被丢弃一分钟

    在 `channels.defaults.botLoopProtection` 下配置一次共享默认值，然后在合法工作流需要更多余量时为 Discord 覆盖该值。优先级为：

    - `channels.discord.accounts.<account>.botLoopProtection`
    - `channels.discord.botLoopProtection`
    - `channels.defaults.botLoopProtection`
    - 内置默认值

    Discord 使用通用的 `maxEventsPerWindow`、`windowSeconds` 和 `cooldownSeconds` 键。

```json5
{
  channels: {
    defaults: {
      botLoopProtection: {
        maxEventsPerWindow: 20,
        windowSeconds: 60,
        cooldownSeconds: 60,
      },
    },
    discord: {
      // 可选的 Discord 全局覆盖。账户块会覆盖各个
      // 字段，并从此处继承省略的字段。
      botLoopProtection: {
        maxEventsPerWindow: 4,
      },
      accounts: {
        alpha: {
          // Alpha 仅在其他 Bot 提及它时才监听其消息。
          allowBots: "mentions",
        },
        bravo: {
          // Bravo 监听 Discord 中由 Bot 发出的所有消息。
          allowBots: true,
          mentionAliases: {
            // 允许 Bravo 使用配置的用户 ID 写入对 Alpha 的 Discord 提及。
            Alpha: "ALPHA_DISCORD_USER_ID",
          },
          botLoopProtection: {
            // 每分钟最多允许五条消息，之后抑制这对 Bot。
            maxEventsPerWindow: 5,
            windowSeconds: 60,
            cooldownSeconds: 90,
          },
        },
      },
    },
  },
}
```

  </Accordion>

  <Accordion title="语音 STT 因 DecryptionFailed(...) 而丢失">

    - 保持 OpenClaw 为最新版本（`openclaw update`），以确保包含 Discord 语音接收恢复逻辑
    - 确认 `channels.discord.voice.daveEncryption=true`（默认值）
    - 从 `channels.discord.voice.decryptionFailureTolerance=24`（上游默认值）开始，仅在需要时调整
    - 留意日志中的以下内容：
      - `discord voice: DAVE decrypt failures detected`
      - `discord voice: repeated decrypt failures; attempting rejoin`
    - 如果自动重新加入后故障仍然存在，请收集日志，并与 [discord.js #11419](https://github.com/discordjs/discord.js/issues/11419) 和 [discord.js #11449](https://github.com/discordjs/discord.js/pull/11449) 中的上游 DAVE 接收历史记录进行比较

  </Accordion>
</AccordionGroup>

## 配置参考

主要参考：[配置参考 - Discord](/zh-CN/gateway/config-channels#discord)。

<Accordion title="高价值 Discord 字段">

- 启动/身份验证：`enabled`、`token`、`applicationId`、`accounts.*`、`allowBots`
- 策略：`groupPolicy`、`dmPolicy`、`allowFrom`、`dm.*`、`guilds.*`、`guilds.*.channels.*`
- 命令：`commands.native`、`commands.useAccessGroups`（全局）、`configWrites`、`slashCommand.ephemeral`
- Gateway 网关：`proxy`
- 回复/历史记录：`replyToMode`、`historyLimit`、`dmHistoryLimit`、`dms.*.historyLimit`
- 传递：`textChunkLimit`（默认值 `2000`）、`maxLinesPerMessage`（默认值 `17`）
- 流式传输：`streaming.mode`、`streaming.chunkMode`、`streaming.preview.*`、`streaming.progress.*`、`streaming.block.*`（旧版扁平 `streamMode`、`draftChunk`、`blockStreaming`、`blockStreamingCoalesce`、`chunkMode` 键会由 `openclaw doctor --fix` 迁移到 `streaming.*`）
- 媒体：`mediaMaxMb`（限制 Discord 出站上传大小，默认值为 `100`）
- 操作：`actions.*`
- 在线状态：`activity`、`status`、`activityType`、`activityUrl`、`autoPresence.*`
- UI：`ui.components.accentColor`
- 功能：`threadBindings`、顶层 `bindings[]`（`type: "acp"`）、`pluralkit`、`execApprovals`、`intents`、`agentComponents.enabled`、`agentComponents.ttlMs`、`activities`、`heartbeat`、`responsePrefix`

</Accordion>

### Discord Activities

设置 `channels.discord.activities`，允许智能体发布可在 Discord 内打开的独立 HTML 小组件。此配置块为可选启用；若不存在，OpenClaw 不会注册任何 Activity 路由、工具或交互处理程序。有关 Developer Portal、隧道、安全和故障排查设置，请参阅 [Discord Activities](/channels/discord-activities)。

- `activities.clientSecret`：Discord 应用程序的 OAuth2 客户端密钥；回退到 `DISCORD_CLIENT_SECRET`
- `activities.applicationId`：可选的 Activity 应用程序 ID；默认使用 Gateway 网关启动时获取的 Bot 应用程序 ID

## 安全与运维

- 将 Bot 令牌视为机密信息（在受监管环境中建议使用 `DISCORD_BOT_TOKEN`）。
- 授予最小权限的 Discord 权限。
- 如果命令部署/状态已过期，请重启 Gateway 网关，并使用 `openclaw channels status --probe` 重新检查。

## 相关内容

<CardGroup cols={2}>
  <Card title="Discord Activities" icon="window" href="/channels/discord-activities">
    在 Discord 内启动交互式 HTML 小组件。
  </Card>
  <Card title="配对" icon="link" href="/zh-CN/channels/pairing">
    将 Discord 用户与 Gateway 网关配对。
  </Card>
  <Card title="群组" icon="users" href="/zh-CN/channels/groups">
    群聊和允许列表行为。
  </Card>
  <Card title="频道路由" icon="route" href="/zh-CN/channels/channel-routing">
    将入站消息路由到智能体。
  </Card>
  <Card title="安全" icon="shield" href="/zh-CN/gateway/security">
    威胁模型和安全加固。
  </Card>
  <Card title="多智能体路由" icon="sitemap" href="/zh-CN/concepts/multi-agent">
    将服务器和频道映射到智能体。
  </Card>
  <Card title="斜杠命令" icon="terminal" href="/zh-CN/tools/slash-commands">
    原生命令行为。
  </Card>
</CardGroup>
