---
read_when:
    - 配置渠道插件（身份验证、访问控制、多账号）
    - 排查各渠道配置键问题
    - 审计私信策略、群组策略或提及门控机制
summary: 频道配置：Slack、Discord、Telegram、WhatsApp、Matrix、iMessage 等渠道的访问控制、配对和各渠道密钥
title: 配置 — 渠道
x-i18n:
    generated_at: "2026-07-26T06:47:40Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: e346648287d275d84a9c082a3bb13edaee751d53546d8231dcf1525bf9adafc2
    source_path: gateway/config-channels.md
    workflow: 16
---

`channels.*` 下的各渠道配置键：私信和群组访问、多账户设置、提及门控，以及 Slack、Discord、Telegram、WhatsApp、Matrix、iMessage 和其他渠道插件的各渠道专用键。

有关智能体、工具、Gateway 网关运行时和其他顶层键，请参阅[配置参考](/zh-CN/gateway/configuration-reference)。

## 渠道

当渠道的配置节存在时，每个渠道都会自动启动（除非 `enabled: false`）。Telegram 和 iMessage 随核心 `openclaw` 包一起提供。其他官方渠道（Discord、Slack、WhatsApp、Matrix、Microsoft Teams、IRC、Google Chat、Signal、Mattermost 等）需使用 `openclaw plugins install <spec>` 作为独立插件安装；完整列表和安装规范请参阅[渠道](/zh-CN/channels)。

### 私信和群组访问

所有渠道都支持私信策略和群组策略：

| 私信策略           | 行为                                                        |
| ------------------- | --------------------------------------------------------------- |
| `pairing`（默认） | 未知发送者会收到一次性配对码；所有者必须批准 |
| `allowlist`         | 仅允许 `allowFrom` 中的发送者（或已配对的允许存储中的发送者）             |
| `open`              | 允许所有入站私信（需要 `allowFrom: ["*"]`）             |
| `disabled`          | 忽略所有入站私信                                          |

| 群组策略          | 行为                                               |
| --------------------- | ------------------------------------------------------ |
| `allowlist`（默认） | 仅允许与已配置允许列表匹配的群组          |
| `open`                | 绕过群组允许列表（提及门控仍然适用） |
| `disabled`            | 阻止所有群组/房间消息                          |

<Note>
当提供商的 `groupPolicy` 未设置时，`channels.defaults.groupPolicy` 会设定默认值。
配对码将在 1 小时后过期。待处理的配对请求上限为**每个账户 3 个**（按渠道和账户 ID 划分）。
如果提供商配置块完全缺失（不存在 `channels.<provider>`），运行时群组策略会回退到 `allowlist`（故障时关闭），并在启动时发出警告。
</Note>

### 渠道模型覆盖

使用 `channels.modelByChannel` 将特定渠道 ID 或私信对端固定到某个模型。值可以是 `provider/model` 或已配置的模型别名。仅当会话尚无有效的模型覆盖时，渠道映射才会应用（例如，通过 `/model` 设置的覆盖）。

对于群组/话题会话，键是渠道特定的群组 ID、主题 ID 或渠道名称。对于私信（DM）会话，键是从渠道发送者身份派生的对端标识符（`nativeDirectUserId`、`origin.from`、`origin.to`、`OriginatingTo`、`From` 或 `SenderId`）。确切的键格式取决于渠道：

| 渠道  | 私信键格式         | 示例                                      |
| -------- | ------------------- | -------------------------------------------- |
| Discord  | 原始用户 ID         | `987654321`                                  |
| Feishu   | `feishu:ou_...`     | `feishu:ou_a8b6cab7e945387de5f253775d9b4d85` |
| Matrix   | Matrix 用户 ID      | `@user:matrix.org`                           |
| Slack    | `user:U...`         | `user:U12345`                                |
| Telegram | 原始用户 ID         | `123456789`                                  |
| WhatsApp | 电话号码或 JID | `15551234567`                                |

```json5
{
  channels: {
    modelByChannel: {
      discord: {
        "123456789012345678": "anthropic/claude-opus-4-6",
      },
      slack: {
        C1234567890: "openai/gpt-5.6-sol",
        "user:U12345": "openai/gpt-5.4-mini",
      },
      telegram: {
        "-1001234567890": "openai/gpt-5.4-mini",
        "-1001234567890:topic:99": "anthropic/claude-sonnet-4-6",
        "123456789": "openai/gpt-4.1",
      },
    },
  },
}
```

私信专用键仅在私信会话中匹配；它们不会影响群组/话题路由。

### 渠道默认值和 Heartbeat

使用 `channels.defaults` 在各提供商之间共享群组策略、隐式提及和 Heartbeat 行为：

```json5
{
  channels: {
    defaults: {
      groupPolicy: "allowlist", // open | allowlist | disabled
      contextVisibility: "all", // all | allowlist | allowlist_quote
      implicitMentions: {
        replyToBot: true,
        quotedBot: true,
        threadParticipation: true,
      },
      heartbeat: {
        showOk: false,
        showAlerts: true,
        useIndicator: true,
      },
    },
  },
}
```

- `channels.defaults.groupPolicy`：当提供商级别的 `groupPolicy` 未设置时使用的后备群组策略。
- `channels.defaults.contextVisibility`：所有渠道的默认补充上下文可见性模式。值：`all`（默认，包含所有引用/话题/历史上下文）、`allowlist`（仅包含允许列表中发送者的上下文）、`allowlist_quote`（与允许列表相同，但保留明确的引用/回复上下文）。各渠道覆盖：`channels.<channel>.contextVisibility`。
- `channels.defaults.implicitMentions`：控制哪些受支持的入站事实算作提及。`replyToBot`、`quotedBot` 和 `threadParticipation` 均默认为 `true`，以保留当前行为。可通过 `channels.<channel>.implicitMentions` 按渠道覆盖，或通过 `channels.<channel>.accounts.<id>.implicitMentions` 按账户覆盖；每个标志均按账户 -> 渠道 -> 默认值的顺序独立解析。这些名称采用肯定形式：将某个标志设为 `false`，即可阻止该事实绕过提及门控。原生明确提及始终允许；当渠道不产生该事实时，相应标志不起作用。有关当前的产生方矩阵，请参阅[提及门控](/zh-CN/channels/groups#mention-gating-default)。这些设置不会更改出站回复/话题模式或已授权命令的处理方式。
- `channels.defaults.heartbeat.showOk`：在 Heartbeat 输出中包含健康的渠道状态（默认 `false`）。
- `channels.defaults.heartbeat.showAlerts`：在 Heartbeat 输出中包含性能下降/错误状态（默认 `true`）。
- `channels.defaults.heartbeat.useIndicator`：呈现紧凑的指示器式 Heartbeat 输出（默认 `true`）。

### WhatsApp

WhatsApp 通过 Gateway 网关的 Web 渠道（Baileys Web）运行。当存在已关联的会话时，它会自动启动。

```json5
{
  web: {
    enabled: true,
  },
  channels: {
    whatsapp: {
      dmPolicy: "pairing", // pairing | allowlist | open | disabled
      allowFrom: ["+15555550123", "+447700900123"],
      textChunkLimit: 4000,
      streaming: { chunkMode: "length" }, // length | newline
      mediaMaxMb: 50,
      sendReadReceipts: true, // 蓝色双勾（自聊模式下为 false）
      groups: {
        "*": { requireMention: true },
      },
      groupPolicy: "allowlist",
      groupAllowFrom: ["+15551234567"],
    },
  },
}
```

- 带有 `type: "acp"` 的顶层 `bindings[]` 条目用于配置 WhatsApp 私信和群组的持久 ACP 绑定。在 `match.peer.id` 中使用 E.164 直拨号码或 WhatsApp 群组 JID。字段语义在 [ACP 智能体](/zh-CN/tools/acp-agents#persistent-channel-bindings)中统一说明。

<Accordion title="多账户 WhatsApp">

```json5
{
  channels: {
    whatsapp: {
      accounts: {
        default: {},
        personal: {},
        biz: {
          // authDir: "~/.openclaw/credentials/whatsapp/biz",
        },
      },
    },
  },
}
```

- 出站命令默认使用账户 `default`（如果存在）；否则使用按顺序排列后的第一个已配置账户 ID。
- 当可选的 `channels.whatsapp.defaultAccount` 与已配置的账户 ID 匹配时，它会覆盖该后备默认账户选择。
- 旧版单账户 Baileys 身份验证目录由 `openclaw doctor` 迁移到 `whatsapp/default`。
- 各账户覆盖：`channels.whatsapp.accounts.<id>.sendReadReceipts`、`channels.whatsapp.accounts.<id>.dmPolicy`、`channels.whatsapp.accounts.<id>.allowFrom`。

</Accordion>

### Telegram

```json5
{
  channels: {
    telegram: {
      enabled: true,
      botToken: "your-bot-token",
      dmPolicy: "pairing",
      allowFrom: ["tg:123456789"],
      groups: {
        "*": { requireMention: true },
        "-1001234567890": {
          allowFrom: ["@admin"],
          systemPrompt: "保持回答简洁。",
          topics: {
            "99": {
              requireMention: false,
              skills: ["search"],
              systemPrompt: "紧扣主题。",
            },
          },
        },
      },
      customCommands: [
        { command: "backup", description: "Git 备份" },
        { command: "generate", description: "创建图像" },
      ],
      historyLimit: 50,
      replyToMode: "first", // off | first | all | batched
      linkPreview: true,
      streaming: { mode: "partial" }, // off | partial | block | progress（默认：partial）
      actions: { reactions: true, sendMessage: true },
      reactionNotifications: "own", // off | own | all
      mediaMaxMb: 100,
      retry: {
        attempts: 3,
        minDelayMs: 400,
        maxDelayMs: 30000,
        jitter: 0.1,
      },
      network: {
        autoSelectFamily: true,
        dnsResultOrder: "ipv4first",
      },
      apiRoot: "https://api.telegram.org",
      trustedLocalFileRoots: ["/srv/telegram-bot-api-data"],
      proxy: "socks5://localhost:9050",
      webhookUrl: "https://example.com/telegram-webhook",
      webhookSecret: "secret",
      webhookPath: "/telegram-webhook",
    },
  },
}
```

- Bot 令牌：`channels.telegram.botToken` 或 `channels.telegram.tokenFile`（仅限普通文件；拒绝符号链接），默认账户可后备使用 `TELEGRAM_BOT_TOKEN`。
- `apiRoot` 仅表示 Telegram Bot API 根地址。请使用 `https://api.telegram.org` 或你的自托管/代理根地址，而不是 `https://api.telegram.org/bot<TOKEN>`；`openclaw doctor --fix` 会移除意外附加的尾部 `/bot<TOKEN>` 后缀。
- 对于以 `--local` 模式运行的自托管 Bot API 服务器，`trustedLocalFileRoots` 会列出 OpenClaw 可以读取的主机路径。将服务器数据卷挂载到 OpenClaw 主机，并配置其数据根目录或每令牌目录；`/var/lib/telegram-bot-api` 下的容器路径会映射到这些根目录中。其他绝对路径仍会被拒绝。
- 当可选的 `channels.telegram.defaultAccount` 与已配置的账户 ID 匹配时，它会覆盖默认账户选择。
- 在多账户设置（2 个或更多账户 ID）中，请设置明确的默认账户（`channels.telegram.defaultAccount` 或 `channels.telegram.accounts.default`），以避免后备路由；当该设置缺失或无效时，`openclaw doctor` 会发出警告。
- `configWrites: false` 会阻止由 Telegram 发起的配置写入（超级群组 ID 迁移、`/config set|unset`）。
- 带有 `type: "acp"` 的顶层 `bindings[]` 条目用于配置论坛主题的持久 ACP 绑定（在 `match.peer.id` 中使用规范的 `chatId:topic:topicId`）。字段语义在 [ACP 智能体](/zh-CN/tools/acp-agents#persistent-channel-bindings)中统一说明。
- Telegram 流式预览使用 `sendMessage` + `editMessageText`（适用于私聊和群聊）。
- `network.dnsResultOrder` 默认为 `"ipv4first"`，以避免常见的 IPv6 获取失败。
- 重试策略：请参阅[重试策略](/zh-CN/concepts/retry)。

### Discord

```json5
{
  channels: {
    discord: {
      enabled: true,
      token: "your-bot-token",
      mediaMaxMb: 100,
      allowBots: false,
      actions: {
        reactions: true,
        stickers: true,
        polls: true,
        permissions: true,
        messages: true,
        threads: true,
        pins: true,
        search: true,
        memberInfo: true,
        roleInfo: true,
        roles: false,
        channelInfo: true,
        voiceStatus: true,
        events: true,
        moderation: false,
      },
      replyToMode: "off", // off | first | all | batched
      dmPolicy: "pairing",
      allowFrom: ["1234567890", "123456789012345678"],
      dm: { enabled: true, groupEnabled: false, groupChannels: ["openclaw-dm"] },
      guilds: {
        "123456789012345678": {
          slug: "friends-of-openclaw",
          requireMention: false,
          ignoreOtherMentions: true,
          reactionNotifications: "own",
          users: ["987654321098765432"],
          channels: {
            general: { allow: true },
            help: {
              allow: true,
              requireMention: true,
              users: ["987654321098765432"],
              skills: ["docs"],
              systemPrompt: "仅提供简短回答。",
            },
          },
        },
      },
      historyLimit: 20,
      textChunkLimit: 2000,
      suppressEmbeds: true,
      streaming: {
        mode: "progress", // off | partial | block | progress（Discord 默认值：progress）
        chunkMode: "length", // length | newline
        progress: {
          label: "auto",
          maxLines: 8,
          maxLineChars: 120,
          toolProgress: true,
        },
      },
      maxLinesPerMessage: 17,
      ui: {
        components: {
          accentColor: "#5865F2",
        },
      },
      threadBindings: {
        enabled: true,
        idleHours: 24,
        maxAgeHours: 0,
        spawnSessions: true,
        defaultSpawnContext: "fork",
      },
      voice: {
        enabled: true,
        autoJoin: [
          {
            guildId: "123456789012345678",
            channelId: "234567890123456789",
          },
        ],
        daveEncryption: true,
        decryptionFailureTolerance: 24,
        connectTimeoutMs: 30000,
        reconnectGraceMs: 15000,
        tts: {
          provider: "openai",
          openai: { voice: "alloy" },
        },
      },
      execApprovals: {
        enabled: "auto", // true | false | "auto"
        approvers: ["987654321098765432"],
        agentFilter: ["default"],
        sessionFilter: ["discord:"],
        target: "dm", // dm | channel | both
        cleanupAfterResolve: false,
      },
      retry: {
        attempts: 3,
        minDelayMs: 500,
        maxDelayMs: 30000,
        jitter: 0.1,
      },
    },
  },
}
```

- 令牌：`channels.discord.token`，默认账户以 `DISCORD_BOT_TOKEN` 作为后备。
- 提供显式 Discord `token` 的直接出站调用会为该调用使用相应令牌；账户重试/策略设置仍来自活动运行时快照中选定的账户。
- 可选的 `channels.discord.defaultAccount` 在与已配置的账户 ID 匹配时会覆盖默认账户选择。
- 使用 `user:<id>`（私信）或 `channel:<id>`（服务器频道）作为投递目标；不接受单独的数字 ID。
- 服务器 slug 使用小写，并将空格替换为 `-`；频道键使用 slug 化的名称（不含 `#`）。优先使用服务器 ID。
- 默认忽略由 Bot 发送的消息。`allowBots: true` 可启用此类消息；使用 `allowBots: "mentions"` 可仅接受提及该 Bot 的 Bot 消息（仍会过滤该 Bot 自己的消息）。
- 支持由 Bot 发送的入站消息的渠道可以使用共享的 [Bot 循环保护](/zh-CN/channels/bot-loop-protection)。通过 `channels.defaults.botLoopProtection` 设置基准配对预算，仅在某个渠道或账户需要不同限制时才进行覆盖。
- `channels.discord.guilds.<id>.ignoreOtherMentions`（以及频道级覆盖项）会丢弃提及其他用户或角色但未提及该 Bot 的消息（@everyone/@here 除外）。
- `channels.discord.mentionAliases` 在发送前将稳定的出站 `@handle` 文本映射到 Discord 用户 ID，因此即使临时目录缓存为空，也能确定性地提及已知团队成员。每账户覆盖项位于 `channels.discord.accounts.<accountId>.mentionAliases` 下。
- `maxLinesPerMessage`（默认值为 `17`）即使在消息少于 2000 个字符时，也会拆分行数过多的消息。
- `channels.discord.suppressEmbeds` 默认为 `true`，因此除非禁用，否则出站 URL 不会展开为 Discord 链接预览。显式的 `embeds` 载荷仍会正常发送；单条消息的工具调用可通过 `suppressEmbeds` 覆盖此行为。
- `channels.discord.threadBindings` 控制 Discord 线程绑定路由：
  - `enabled`：针对线程绑定会话功能的 Discord 覆盖项（`/focus`、`/unfocus`、`/agents`、`/session idle`、`/session max-age`，以及绑定投递/路由）
  - `idleHours`：以小时为单位的 Discord 非活动自动取消聚焦覆盖项（`0` 表示禁用）
  - `maxAgeHours`：以小时为单位的 Discord 最大存续时间硬限制覆盖项（`0` 表示禁用）
  - `spawnSessions`：控制 `sessions_spawn({ thread: true })` 和 ACP 线程派生时自动创建/绑定线程的开关（默认值：`true`）
  - `defaultSpawnContext`：线程绑定派生所使用的原生子智能体上下文（默认为 `"fork"`）
- 带有 `type: "acp"` 的顶层 `bindings[]` 条目用于配置频道和线程的持久 ACP 绑定（在 `match.peer.id` 中使用频道/线程 ID）。字段语义统一说明于 [ACP 智能体](/zh-CN/tools/acp-agents#persistent-channel-bindings)。
- `channels.discord.ui.components.accentColor` 设置 Discord components v2 容器的强调色。
- `channels.discord.agentComponents.ttlMs` 控制已发送的 Discord 组件回调保持注册的时长。默认值为 `1800000`（30 分钟），最大值为 `86400000`（24 小时）。每账户覆盖项位于 `channels.discord.accounts.<accountId>.agentComponents.ttlMs` 下。应优先选择能满足工作流需求的最短 TTL。
- `channels.discord.voice` 启用 Discord 语音频道对话，以及可选的自动加入、LLM 和 TTS 覆盖项。仅文本的 Discord 配置默认关闭语音功能；设置 `channels.discord.voice.enabled=true` 可选择启用。
- `channels.discord.voice.model` 可选择覆盖用于 Discord 语音频道响应的 LLM 模型。
- `channels.discord.voice.daveEncryption`（默认值为 `true`）和 `channels.discord.voice.decryptionFailureTolerance`（默认值为 `24`）会透传给 `@discordjs/voice` 的 DAVE 选项。
- `channels.discord.voice.connectTimeoutMs` 控制 `/vc join` 和自动加入尝试最初等待 `@discordjs/voice` Ready 的时长（默认值为 `30000`）。
- `channels.discord.voice.reconnectGraceMs` 控制断开连接的语音会话可用多长时间进入重连信令状态，超时后 OpenClaw 会将其销毁（默认值为 `15000`）。
- Discord 语音播放不会因另一位用户的开始说话事件而中断。为避免反馈循环，OpenClaw 会在播放 TTS 时忽略新的语音捕获。
- 此外，在反复发生解密失败后，OpenClaw 会通过退出并重新加入语音会话来尝试恢复语音接收。
- `channels.discord.streaming` 是规范的流式模式键。Discord 默认为 `streaming.mode: "progress"`，因此工具/工作进度会显示在一条持续编辑的预览消息中；设置 `streaming.mode: "off"` 可禁用此功能。旧版扁平键（`streamMode`、`chunkMode`、`blockStreaming`、`draftChunk`、`blockStreamingCoalesce`）在运行时已不再读取；运行 `openclaw doctor --fix` 可迁移持久化配置。
- `channels.discord.autoPresence` 将运行时可用性映射为 Bot 在线状态（正常 => 在线、降级 => 空闲、耗尽 => 请勿打扰），并允许选择性覆盖状态文本。
- `channels.discord.guilds.<id>.presenceEvents` 将人员可用性上线事件作为智能体系统事件路由到一个已配置的 Discord 频道。符合条件的成员必须能够查看 `channelId`；公共线程继承父级可见性，而私有线程还要求成员身份或 Manage Threads 权限。`users` 可进一步缩小该受众范围。它会根据完整的 `GUILD_CREATE` 快照初始化当前在线成员、路由观测到的离线到在线状态转换，并将此前未见成员后续首次出现的在线信号视为新近可用，但不会断言其是在快照后上线还是加入。成员数超过 Discord 75,000 人快照限制的服务器需要先收到一次显式离线更新。限流项：`reconnectSuppressSeconds`（新 Gateway 网关会话建立后、服务器在线状态重建期间的静默窗口，默认值为 300，`0` 表示禁用）以及 `burstLimit`/`burstWindowSeconds`（每个服务器成功入队事件的速率限制，默认值为每 60 秒滑动窗口 8 个事件）。恢复的会话不会启动重连抑制窗口。现有的每用户再次问候冷却时间仍为八小时。此功能要求 `channels.discord.intents.presence=true`、Discord Developer Portal 中的特权 Presence Intent，以及已启用的智能体 Heartbeat。
- `channels.discord.dangerouslyAllowNameMatching` 重新启用可变名称/标签匹配（紧急兼容模式）。
- `channels.discord.execApprovals`：Discord 原生 Exec 审批投递和审批者授权。
  - `enabled`：`true`、`false` 或 `"auto"`（默认值）。在自动模式下，当可以从 `approvers` 或 `commands.ownerAllowFrom` 解析审批者时，将激活 Exec 审批。
  - `approvers`：允许批准 Exec 请求的 Discord 用户 ID。省略时回退到 `commands.ownerAllowFrom`。
  - `agentFilter`：可选的智能体 ID 允许列表。省略则转发所有智能体的审批。
  - `sessionFilter`：可选的会话键模式（子字符串或正则表达式）。
  - `target`：审批提示的发送位置。`"dm"`（默认值）发送至审批者私信，`"channel"` 发送至发起请求的频道，`"both"` 发送至两者。当目标包含 `"channel"` 时，只有已解析的审批者可以使用按钮。
  - `cleanupAfterResolve`：当为 `true` 时，会在批准、拒绝或超时后删除审批私信。

**表情回应通知模式：**`off`（无）、`own`（Bot 的消息，默认值）、`all`（所有消息）、`allowlist`（所有消息中来自 `guilds.<id>.users` 的表情回应）。

### Google Chat

```json5
{
  channels: {
    googlechat: {
      enabled: true,
      serviceAccountFile: "/path/to/service-account.json",
      audienceType: "app-url", // app-url | project-number
      audience: "https://gateway.example.com/googlechat",
      webhookPath: "/googlechat",
      botUser: "users/1234567890",
      dmPolicy: "pairing",
      allowFrom: ["users/1234567890"],
      groupPolicy: "allowlist",
      groups: {
        "spaces/AAAA": { allow: true, requireMention: true },
      },
      actions: { reactions: true },
      typingIndicator: "message",
      mediaMaxMb: 20,
    },
  },
}
```

- 服务账户 JSON：内联（`serviceAccount`）或基于文件（`serviceAccountFile`）。
- `serviceAccount` 直接接受 SecretRef。
- 环境变量后备项：`GOOGLE_CHAT_SERVICE_ACCOUNT` 或 `GOOGLE_CHAT_SERVICE_ACCOUNT_FILE`（仅限默认账户）。
- 使用 `spaces/<spaceId>` 或 `users/<userId>` 作为投递目标。
- `channels.googlechat.dangerouslyAllowNameMatching` 重新启用可变电子邮件主体匹配（紧急兼容模式）。

### Slack

```json5
{
  channels: {
    slack: {
      enabled: true,
      botToken: "xoxb-...",
      appToken: "xapp-...",
      socketMode: {
        clientPingTimeout: 15000,
        serverPingTimeout: 30000,
        pingPongLoggingEnabled: false,
      },
      dmPolicy: "pairing",
      allowFrom: ["U123", "U456", "*"],
      dm: { enabled: true, groupEnabled: false, groupChannels: ["G123"] },
      channels: {
        C123: { enabled: true, requireMention: true, allowBots: false },
        "#general": {
          enabled: true,
          requireMention: true,
          allowBots: false,
          users: ["U123"],
          skills: ["docs"],
          systemPrompt: "仅提供简短回答。",
        },
      },
      historyLimit: 50,
      allowBots: false,
      reactionNotifications: "own",
      reactionAllowlist: ["U123"],
      replyToMode: "off", // off | first | all | batched
      thread: {
        historyScope: "thread", // thread | channel
        inheritParent: false,
        initialHistoryLimit: 20,
      },
      actions: {
        reactions: true,
        messages: true,
        pins: true,
        memberInfo: true,
        emojiList: true,
      },
      slashCommand: {
        enabled: true,
        name: "openclaw",
        sessionPrefix: "slack:slash",
        ephemeral: true,
      },
      typingReaction: "hourglass_flowing_sand",
      unfurlLinks: false,
      unfurlMedia: false,
      textChunkLimit: 4000,
      streaming: {
        mode: "partial", // off | partial | block | progress
        chunkMode: "length", // length | newline
        nativeTransport: true, // 当 mode=partial 时使用 Slack 原生流式 API
      },
      mediaMaxMb: 20,
      execApprovals: {
        enabled: "auto", // true | false | "auto"
        approvers: ["U123"],
        agentFilter: ["default"],
        sessionFilter: ["slack:"],
        target: "dm", // dm | channel | both
      },
    },
  },
}
```

- **Socket 模式**同时需要 `botToken` 和 `appToken`（默认账户的环境变量回退需要 `SLACK_BOT_TOKEN` + `SLACK_APP_TOKEN`）。
- **HTTP 模式**需要 `botToken` 以及 `signingSecret`（位于根级别或每个账户中）。
- **用户身份**（`identity: "user"`）以授权用户的身份发布和读取内容。在 Socket 模式下，它需要 `userToken` 以及 `appToken`；在 HTTP 模式下，则需要 `userToken` 以及 `signingSecret`。不需要机器人令牌或机器人用户。有关用户权限范围和事件订阅，请参阅[用户身份](/zh-CN/channels/slack#user-identity-post-as-a-real-person)。
- `enterpriseOrgInstall: true` 使账户选择使用 Slack Enterprise Grid
  的组织范围事件路径。启动时会使用 `auth.test` 验证机器人令牌，并且
  当配置的模式与 Slack 的安装身份不匹配时启动失败。
  Enterprise 私信必须禁用，或使用 `dmPolicy: "open"` 并配置有效的
  `allowFrom: ["*"]`。频道和用户策略必须使用稳定的 Slack ID；
  可变名称和不受支持的频道前缀会导致启动失败。V1 仅处理
  直接 Socket 模式或 HTTP `message` 和 `app_mention` 事件，并立即
  回复；中继、命令、交互、App Home、表情回应事件监听器、
  置顶、操作工具、原生审批、绑定、延迟投递和
  主动发送均不可用。由监听器负责的确认、输入状态和
  状态表情回应仍可通过 `reactions:write` 使用；入站表情回应
  通知和表情回应操作工具不可用。有关最小权限清单、设置工作流和完整限制，
  请参阅 [Enterprise Grid 组织范围安装](/zh-CN/channels/slack#enterprise-grid-org-wide-installs)。
- `socketMode` 将 Slack SDK Socket 模式的传输调优选项传递给公开的 Bolt 接收器 API。仅在调查 ping/pong 超时或 WebSocket 连接失效行为时使用。`clientPingTimeout` 默认为 `15000`；只有在配置后才会传递 `serverPingTimeout` 和 `pingPongLoggingEnabled`。
- `botToken`、`appToken`、`signingSecret` 和 `userToken` 接受明文
  字符串或 SecretRef 对象。
- Slack 账户快照会公开每项凭据的来源/状态字段，例如
  `botTokenSource`、`botTokenStatus`、`userTokenSource`、`userTokenStatus`、
  `appTokenStatus`，以及 HTTP 模式下的 `signingSecretStatus`。
  `configured_unavailable` 表示账户
  通过 SecretRef 配置，但当前命令/运行时路径无法
  解析密钥值。
- `configWrites: false` 阻止由 Slack 发起的配置写入。
- 当可选的 `channels.slack.defaultAccount` 与已配置的账户 ID 匹配时，它会覆盖默认账户选择。
- `channels.slack.streaming.mode` 是规范的 Slack 流模式键（默认值为 `"partial"`）。`channels.slack.streaming.nativeTransport` 控制 Slack 的原生流式传输（默认值为 `true`）。旧版 `streamMode`、布尔值 `streaming`、`chunkMode`、`blockStreaming`、`blockStreamingCoalesce` 和 `nativeStreaming` 在运行时不再读取；请运行 `openclaw doctor --fix`，将持久化配置迁移到 `streaming.{mode,chunkMode,block.enabled,block.coalesce,nativeTransport}`。
- `unfurlLinks` 和 `unfurlMedia` 会为机器人回复传递 Slack 的 `chat.postMessage` 链接和媒体展开布尔值。`unfurlLinks` 默认为 `false`，因此除非启用，否则机器人发出的链接不会在行内展开；除非已配置，否则会省略 `unfurlMedia`。可在 `channels.slack.accounts.<accountId>` 中设置任一值，为单个账户覆盖顶层值。
- 投递目标请使用 `user:<id>`（私信）或 `channel:<id>`。

**表情回应通知模式：**`off`、`own`（默认）、`all`、`allowlist`（来自 `reactionAllowlist`）。

**话题串会话隔离：**`thread.historyScope` 可按话题串隔离（默认），也可在频道内共享。`thread.inheritParent` 会将父频道的对话记录复制到新话题串。`thread.initialHistoryLimit`（默认值为 `20`）限制新话题串会话启动时获取的现有话题串消息数量；`0` 会禁用话题串历史记录获取。

- Slack 原生流式传输和 Slack 助手风格的“正在输入...”话题串状态都需要以回复话题串为目标。顶层私信默认不进入话题串，因此仍可通过 Slack 草稿发布和编辑预览进行流式传输，而不显示话题串风格的原生流式传输/状态预览。
- `typingReaction` 会在回复运行期间为入站 Slack 消息添加临时表情回应，并在完成后将其移除。请使用 Slack 表情符号短代码，例如 `"hourglass_flowing_sand"`。
- `channels.slack.execApprovals`：Slack 原生审批客户端投递和 Exec 审批者授权。架构与 Discord 相同：`enabled`（`true`/`false`/`"auto"`）、`approvers`（Slack 用户 ID）、`agentFilter`、`sessionFilter`，以及 `target`（`"dm"`、`"channel"` 或 `"both"`）。当能够解析 Slack 插件审批者时，插件审批可对源自 Slack 的请求使用此原生客户端路径；还可通过 `approvals.plugin`，为源自 Slack 的会话或 Slack 目标启用 Slack 原生插件审批投递。插件审批使用 `allowFrom` 中的 Slack 插件审批者和默认路由，而不是 Exec 审批者。

| 操作组       | 默认值 | 说明                     |
| ------------ | ------ | ------------------------ |
| reactions    | 已启用 | 添加表情回应 + 列出表情回应 |
| messages     | 已启用 | 读取/发送/编辑/删除       |
| pins         | 已启用 | 置顶/取消置顶/列出        |
| memberInfo   | 已启用 | 成员信息                 |
| emojiList    | 已启用 | 自定义表情符号列表        |

### Mattermost

Mattermost 作为独立插件安装，方式与 Discord、Slack 和 WhatsApp 相同：

```bash
openclaw plugins install @openclaw/mattermost
```

固定版本之前，请在 [npmjs.com/package/@openclaw/mattermost](https://www.npmjs.com/package/@openclaw/mattermost) 查看当前的 dist-tag。

```json5
{
  channels: {
    mattermost: {
      enabled: true,
      botToken: "mm-token",
      baseUrl: "https://chat.example.com",
      dmPolicy: "pairing",
      chatmode: "oncall", // oncall | onmessage | onchar
      oncharPrefixes: [">", "!"],
      groups: {
        "*": { requireMention: true },
        "team-channel-id": { requireMention: false },
      },
      commands: {
        native: true, // 选择启用
        nativeSkills: true,
        callbackPath: "/api/channels/mattermost/command",
        // 用于反向代理/公开部署的可选显式 URL
        callbackUrl: "https://gateway.example.com/api/channels/mattermost/command",
      },
      textChunkLimit: 4000,
      streaming: { chunkMode: "length" },
    },
  },
}
```

聊天模式：`oncall`（在 @提及时响应，默认）、`onmessage`（每条消息）、`onchar`（以触发前缀开头的消息）。

启用 Mattermost 原生命令时：

- `commands.callbackPath` 必须是路径（例如 `/api/channels/mattermost/command`），不能是完整 URL。
- `commands.callbackUrl` 必须解析到 OpenClaw Gateway 网关端点，并且 Mattermost 服务器必须能够访问它。
- 原生斜杠命令回调使用 Mattermost 在注册斜杠命令期间返回的
  每命令令牌进行身份验证。如果注册失败或没有激活任何
  命令，OpenClaw 将拒绝回调并返回
  `Unauthorized: invalid command token.`
- 对于私有/tailnet/内部回调主机，Mattermost 可能要求
  `ServiceSettings.AllowedUntrustedInternalConnections` 包含回调主机/域名。
  请使用主机/域名值，而不是完整 URL。
- `channels.mattermost.configWrites`：允许或拒绝由 Mattermost 发起的配置写入。
- `channels.mattermost.requireMention`：在频道中回复前要求 `@mention`。
- `channels.mattermost.groups.<channelId>.requireMention`：按频道覆盖提及门控（默认使用 `"*"`）。
- 当可选的 `channels.mattermost.defaultAccount` 与已配置的账户 ID 匹配时，它会覆盖默认账户选择。

### Signal

```json5
{
  channels: {
    signal: {
      enabled: true,
      account: "+15555550123", // 可选的账户绑定
      dmPolicy: "pairing",
      allowFrom: ["+15551234567", "uuid:123e4567-e89b-12d3-a456-426614174000"],
      configWrites: true,
      reactionNotifications: "own", // off | own | all | allowlist
      reactionAllowlist: ["+15551234567", "uuid:123e4567-e89b-12d3-a456-426614174000"],
      historyLimit: 50,
    },
  },
}
```

**表情回应通知模式：**`off`、`own`（默认）、`all`、`allowlist`（来自 `reactionAllowlist`）。

- `channels.signal.account`：将渠道启动固定到特定 Signal 账户身份。
- `channels.signal.configWrites`：允许或拒绝由 Signal 发起的配置写入。
- 当可选的 `channels.signal.defaultAccount` 与已配置的账户 ID 匹配时，它会覆盖默认账户选择。

### iMessage

OpenClaw 会启动 `imsg rpc`（通过 stdio 使用 JSON-RPC）。无需守护进程或端口。当主机能够授予“信息”数据库和“自动化”权限时，这是新 OpenClaw iMessage 设置的首选路径。

BlueBubbles 支持已移除。当前 OpenClaw 不支持将 `channels.bluebubbles` 作为运行时配置界面。请将旧配置迁移到 `channels.imessage`；简要说明请参阅 [BlueBubbles removal and the imsg iMessage path](/zh-CN/announcements/bluebubbles-imessage)，完整转换表请参阅 [Coming from BlueBubbles](/zh-CN/channels/imessage-from-bluebubbles)。

如果 Gateway 网关未在已登录“信息”的 Mac 上运行，请保留 `channels.imessage.enabled=true`，并将 `channels.imessage.cliPath` 设置为一个 SSH 包装器，以便在该 Mac 上运行 `imsg "$@"`。默认的本地 `imsg` 路径仅适用于 macOS。

在依赖 SSH 包装器进行生产发送之前，请通过该包装器本身验证一次出站 `imsg send`。某些 macOS TCC 状态会将 Messages 自动化权限分配给 `/usr/libexec/sshd-keygen-wrapper`，这可能导致读取和探测正常工作，但发送会因 AppleEvents `-1743` 而失败；请参阅 [iMessage](/zh-CN/channels/imessage) 中的 SSH 包装器故障排除部分。

```json5
{
  channels: {
    imessage: {
      enabled: true,
      cliPath: "imsg",
      dbPath: "~/Library/Messages/chat.db",
      remoteHost: "user@gateway-host",
      dmPolicy: "pairing",
      allowFrom: ["+15555550123", "user@example.com", "chat_id:123"],
      historyLimit: 50,
      includeAttachments: false,
      attachmentRoots: ["/Users/*/Library/Messages/Attachments"],
      remoteAttachmentRoots: ["/Users/*/Library/Messages/Attachments"],
      mediaMaxMb: 16,
      service: "auto",
      sendTransport: "auto",
      region: "US",
      actions: {
        reactions: true,
        edit: true,
        unsend: true,
        reply: true,
        sendWithEffect: true,
        sendAttachment: true,
      },
    },
  },
}
```

- 可选的 `channels.imessage.defaultAccount` 在与已配置的账户 ID 匹配时会覆盖默认账户选择。
- 需要对 Messages 数据库具有完全磁盘访问权限。
- 优先使用 `chat_id:<id>` 目标。使用 `imsg chats --limit 20` 列出聊天。
- `cliPath` 可以指向 SSH 包装器；设置 `remoteHost`（`host` 或 `user@host`）以通过 SCP 获取附件。
- `attachmentRoots` 和 `remoteAttachmentRoots` 用于限制入站附件路径（默认值：`/Users/*/Library/Messages/Attachments`）。
- SCP 使用严格的主机密钥检查，因此请确保中继主机密钥已存在于 `~/.ssh/known_hosts` 中。
- `channels.imessage.configWrites`：允许或拒绝由 iMessage 发起的配置写入。
- `channels.imessage.sendTransport`：常规出站回复首选的 `imsg` RPC 发送传输方式。`auto`（默认值）会在 IMCore 桥接器运行时对现有聊天使用该桥接器，然后回退到 AppleScript；`bridge` 要求通过私有 API 进行投递；`applescript` 强制使用公开的 Messages 自动化路径。
- `channels.imessage.actions.*`：启用同时受 `imsg status` / `openclaw channels status --probe` 限制的私有 API 操作。
- `channels.imessage.includeAttachments` 默认关闭；在期望智能体轮次中接收入站媒体之前，请将其设为 `true`。
- 桥接器/Gateway 网关重启后的入站恢复会自动进行（GUID 去重加过期积压消息的时间界限）。现有的 `channels.imessage.catchup.enabled: true` 配置仍会作为已弃用的兼容性配置生效；`catchup` 默认禁用。
- `channels.imessage.groups`：群组注册表和各群组设置。使用 `groupPolicy: "allowlist"` 时，请配置明确的 `chat_id` 键或一个 `"*"` 通配符条目，以便群组消息通过注册表门控。
- 带有 `type: "acp"` 的顶层 `bindings[]` 条目可以将 iMessage 对话绑定到持久 ACP 会话。在 `match.peer.id` 中使用规范化句柄或明确的聊天目标（`chat_id:*`、`chat_guid:*`、`chat_identifier:*`）。共享字段语义：[ACP 智能体](/zh-CN/tools/acp-agents#persistent-channel-bindings)。

<Accordion title="iMessage SSH 包装器示例">

```bash
#!/usr/bin/env bash
exec ssh -T gateway-host imsg "$@"
```

</Accordion>

### Matrix

Matrix 由插件支持，并在 `channels.matrix` 下配置。

```json5
{
  channels: {
    matrix: {
      enabled: true,
      homeserver: "https://matrix.example.org",
      accessToken: "syt_bot_xxx",
      proxy: "http://127.0.0.1:7890",
      encryption: true,
      initialSyncLimit: 20,
      defaultAccount: "ops",
      accounts: {
        ops: {
          name: "Ops",
          userId: "@ops:example.org",
          accessToken: "syt_ops_xxx",
        },
        alerts: {
          userId: "@alerts:example.org",
          password: "secret",
          proxy: "http://127.0.0.1:7891",
        },
      },
    },
  },
}
```

- 令牌身份验证使用 `accessToken`；密码身份验证使用 `userId` + `password`。
- `channels.matrix.proxy` 通过明确指定的 HTTP(S) 代理路由 Matrix HTTP 流量。命名账户可以使用 `channels.matrix.accounts.<id>.proxy` 覆盖它。
- `channels.matrix.network.dangerouslyAllowPrivateNetwork` 允许私有/内部主服务器。`proxy` 与此网络选择启用项是相互独立的控制项。
- `channels.matrix.defaultAccount` 用于在多账户设置中选择首选账户。
- `channels.matrix.autoJoin` 默认为 `"off"`，因此在使用 `autoJoinAllowlist` 或 `autoJoin: "always"` 设置 `autoJoin: "allowlist"` 之前，受邀房间和新的私信式邀请会被忽略。
- `channels.matrix.execApprovals`：Matrix 原生 Exec 审批投递和审批者授权。
  - `enabled`：`true`、`false` 或 `"auto"`（默认值）。在自动模式下，如果可以从 `approvers` 或 `commands.ownerAllowFrom` 解析审批者，Exec 审批便会启用。
  - `approvers`：允许审批 Exec 请求的 Matrix 用户 ID（例如 `@owner:example.org`）。
  - `agentFilter`：可选的智能体 ID 允许列表。省略时会转发所有智能体的审批。
  - `sessionFilter`：可选的会话键模式（子字符串或正则表达式）。
  - `target`：审批提示的发送位置。`"dm"`（默认值）、`"channel"`（发起房间）或 `"both"`。
  - 按账户覆盖：`channels.matrix.accounts.<id>.execApprovals`。
- `channels.matrix.dm.sessionScope` 控制 Matrix 私信如何分组到会话中：`per-user`（默认值）按路由对等方共享，而 `per-room` 会隔离每个私信房间。
- Matrix 状态探测和实时目录查询使用与运行时流量相同的代理策略。
- 完整的 Matrix 配置、目标规则和设置示例记录在 [Matrix](/zh-CN/channels/matrix) 中。

### Microsoft Teams

Microsoft Teams 由插件支持，并在 `channels.msteams` 下配置。

```json5
{
  channels: {
    msteams: {
      enabled: true,
      configWrites: true,
      // appId、appPassword、tenantId、webhook、团队/频道策略：
      // 请参阅 /channels/msteams
    },
  },
}
```

- 此处涵盖的核心键路径：`channels.msteams`、`channels.msteams.configWrites`。
- 完整的 Teams 配置（凭据、webhook、私信/群组策略、按团队/按频道覆盖）记录在 [Microsoft Teams](/zh-CN/channels/msteams) 中。

### IRC

IRC 由插件支持，并在 `channels.irc` 下配置。

```json5
{
  channels: {
    irc: {
      enabled: true,
      dmPolicy: "pairing",
      configWrites: true,
      nickserv: {
        enabled: true,
        service: "NickServ",
        password: "${IRC_NICKSERV_PASSWORD}",
        register: false,
        registerEmail: "bot@example.com",
      },
    },
  },
}
```

- 此处涵盖的核心键路径：`channels.irc`、`channels.irc.dmPolicy`、`channels.irc.configWrites`、`channels.irc.nickserv.*`。
- 可选的 `channels.irc.defaultAccount` 在与已配置的账户 ID 匹配时会覆盖默认账户选择。
- 完整的 IRC 渠道配置（主机/端口/TLS/频道/允许列表/提及门控）记录在 [IRC](/zh-CN/channels/irc) 中。

### 多账户（所有渠道）

每个渠道运行多个账户（每个账户都有自己的 `accountId`）：

```json5
{
  channels: {
    telegram: {
      accounts: {
        default: {
          name: "Primary bot",
          botToken: "123456:ABC...",
        },
        alerts: {
          name: "Alerts bot",
          botToken: "987654:XYZ...",
        },
      },
    },
  },
}
```

- 省略 `accountId` 时会使用 `default`（CLI + 路由）。
- 环境变量中的令牌仅适用于**默认**账户。
- 基础渠道设置适用于所有账户，除非按账户覆盖。
- 使用 `bindings[].match.accountId` 将每个账户路由到不同的智能体。
- 如果在仍使用单账户顶层渠道配置时，通过 `openclaw channels add`（或渠道新手引导）添加非默认账户，OpenClaw 会先将账户范围的顶层单账户值提升到渠道账户映射中，以便原始账户继续工作。大多数渠道会将这些值移入 `channels.<channel>.accounts.default`；Matrix 则可以保留现有的匹配命名/默认目标。
- 现有的仅渠道绑定（没有 `accountId`）仍会匹配默认账户；账户范围的绑定仍为可选项。
- `openclaw doctor --fix` 还会修复混合结构，将账户范围的顶层单账户值移入为该渠道选择的提升账户中。大多数渠道使用 `accounts.default`；Matrix 则可以保留现有的匹配命名/默认目标。

### 其他插件渠道

许多插件渠道配置为 `channels.<id>`，并记录在各自的专用渠道页面中（例如 Feishu、LINE、Nextcloud Talk、Nostr、QQ Bot、Synology Chat、Twitch 和 Zalo）。
请参阅完整的渠道索引：[渠道](/zh-CN/channels)。

### 群聊提及门控

群组消息默认**要求提及**（元数据提及或安全的正则表达式模式）。适用于 WhatsApp、Telegram、Discord、Google Chat 和 iMessage 群聊。

可见回复由单独的设置控制。常规群组、频道和内部 WebChat 直接请求默认自动投递最终内容：智能体最终文本通过旧版可见回复路径发布。如果模型编写的来源回复仅应在智能体调用 `message(action=send)` 后发布，请选择启用 `messages.visibleReplies: "message_tool"` 或 `messages.groupChat.visibleReplies: "message_tool"`。如果模型在已选择启用的仅工具模式下返回实质性的最终答案，却未调用消息工具，该最终文本会保持私密，Gateway 网关详细日志会记录被抑制的负载元数据，并且 OpenClaw 会加入一次恢复重试，请求模型通过 `message(action=send)` 投递相同的回复。

仅工具策略控制智能体来源回复和通用工具媒体。它不会抑制由运行时所有的终端输出，例如已授权的命令响应、持久完成通知，或所属 harness 明确归类为主机所有的提供商原生工件。主机所有的工件通过常规渠道分发路径投递，并且仍遵循出站 `sendPolicy` 拒绝规则。环境中的 `room_event` 轮次仍会保持安静，除非它们是明确的命令，即使运行时输出被标记为主机所有也是如此。

仅工具可见回复要求模型/运行时能够可靠地调用工具，建议在使用 GPT-5.6 Sol 等最新一代模型的共享环境房间中采用。某些能力较弱的模型可以返回最终文本，却无法理解来源可见的输出必须通过 `message(action=send)` 发送。默认情况下，OpenClaw 仅会在最终内容具有实质性、来源轮次不是房间事件、发送策略未拒绝投递且尚未发送来源回复时，恢复常见的最终回复搁置情况。恢复仅限一次重试；它会禁止持久化合成的重试提示，并使该重试不参与收集批处理，从而避免与其他无关的排队提示合并。如果重试仍被搁置或无法入队，OpenClaw 只会投递经过净化的诊断消息，例如“我生成了回复，但无法将其投递到此聊天。请重试。”原始的私密最终文本永远不会被标记为自动投递到来源。对于反复搁置回复的模型，请使用 `"automatic"`，使智能体最终轮次成为可见回复路径；或改用工具调用能力更强的模型；或检查 Gateway 网关详细日志中的被抑制负载摘要；或设置 `messages.groupChat.visibleReplies: "automatic"`，对每个群组/频道请求使用可见最终回复。

如果消息工具在当前工具策略下不可用，OpenClaw 会回退到自动发送可见回复，而不是静默抑制响应。`openclaw doctor` 会对此不匹配情况发出警告。

此规则适用于智能体的常规最终文本。插件拥有的对话绑定会将所属插件返回的回复用作已认领绑定线程轮次的可见响应；插件无需为这些绑定回复调用 `message(action=send)`。

**故障排查：群组 @提及触发正在输入后却无响应（无错误）**

症状：群组/频道中的 @提及会显示正在输入指示器，并且 Gateway 网关日志报告 `dispatch complete (queuedFinal=false, replies=0)`，但房间中没有收到任何消息。向同一智能体发送私信时可以正常回复。

原因：群组/频道的可见回复模式解析为 `"message_tool"`，因此 OpenClaw 会运行该轮次，但除非智能体调用 `message(action=send)`，否则会抑制最终助手文本。此模式下不存在 `NO_REPLY` 契约；未调用消息工具意味着原始最终文本是私密的。对于包含实质内容的来源轮次，OpenClaw 现在会尝试一次受保护的恢复重试；简短备注、明确静默、房间事件、发送策略拒绝的轮次以及已送达的轮次不会重试。普通群组和频道轮次默认使用 `"automatic"`，因此仅当将 `messages.groupChat.visibleReplies`（或全局 `messages.visibleReplies`）显式设置为 `"message_tool"` 时，才会出现此症状。Harness `defaultVisibleReplies` 在此不适用——群组/频道解析器会忽略它；它只影响直接/来源聊天（Codex harness 会以这种方式抑制直接聊天的最终文本）。

修复：选择工具调用能力更强的模型、移除显式的 `"message_tool"` 覆盖以回退到默认的 `"automatic"`，或者设置 `messages.groupChat.visibleReplies: "automatic"`，强制每个群组/频道请求都发送可见回复。包含实质内容但未能送达的最终文本不应再以静默成功告终；它应通过一次 `message(action=send)` 重试恢复，或显示经过清理的送达失败诊断。保存文件后，Gateway 网关会热重载 `messages` 配置；仅当部署中禁用了文件监视或配置重载时，才需要重启 Gateway 网关。

**提及类型：**

- **元数据提及**：平台原生 @提及。在 WhatsApp 自聊模式下会被忽略。
- **文本模式**：`agents.entries.*.groupChat.mentionPatterns` 中的安全正则表达式模式。无效模式和不安全的嵌套重复会被忽略。
- 仅当能够检测提及时（存在原生提及或至少一个模式），才会强制执行提及门控。

```json5
{
  messages: {
    visibleReplies: "automatic", // 强制直接/来源聊天使用旧版自动最终回复
    groupChat: {
      historyLimit: 50,
      unmentionedInbound: "room_event", // 将始终开启但未提及的房间闲聊转为安静上下文
      visibleReplies: "message_tool", // 选择启用；要求使用 message(action=send) 发送可见房间回复
    },
  },
  agents: {
    list: [{ id: "main", groupChat: { mentionPatterns: ["@openclaw", "openclaw"] } }],
  },
}
```

`messages.groupChat.historyLimit` 设置全局默认值。渠道可以使用 `channels.<channel>.historyLimit`（或按账户设置）进行覆盖。设置 `0` 可禁用。

`messages.groupChat.unmentionedInbound: "room_event"` 会在受支持的渠道上，将未提及但始终开启的群组/频道消息作为安静的房间上下文提交。被提及的消息、命令和私信仍会作为用户请求处理。有关完整的 Discord、Slack 和 Telegram 示例，请参阅[环境房间事件](/zh-CN/channels/ambient-room-events)。

`messages.visibleReplies` 是全局来源事件默认值；`messages.groupChat.visibleReplies` 会针对群组/频道来源事件覆盖它。当未设置 `messages.visibleReplies` 时，直接/来源聊天使用所选运行时或 harness 默认值，但内部 WebChat 直接轮次会使用自动最终送达，以保持 Pi/Codex 提示词一致。设置 `messages.visibleReplies: "message_tool"` 可有意要求使用 `message(action=send)` 才能产生可见输出。渠道允许列表和提及门控仍会决定是否处理事件。

#### 私信历史记录限制

```json5
{
  channels: {
    telegram: {
      dmHistoryLimit: 30,
      dms: {
        "123456789": { historyLimit: 50 },
      },
    },
  },
}
```

解析顺序：按私信覆盖 → 提供商默认值 → 无限制（全部保留）。

此解析器会为会话键遵循标准 `provider:direct:<id>`（或旧版 `provider:dm:<id>`）格式的任何渠道读取 `channels.<provider>.dmHistoryLimit` 和 `channels.<provider>.dms.<id>.historyLimit`，因此它同样适用于内置渠道和插件渠道，而不局限于固定列表。

#### 自聊模式

在 `allowFrom` 中包含你自己的号码即可启用自聊模式（忽略原生 @提及，仅响应文本模式）：

```json5
{
  channels: {
    whatsapp: {
      allowFrom: ["+15555550123"],
      groups: { "*": { requireMention: true } },
    },
  },
  agents: {
    list: [
      {
        id: "main",
        groupChat: { mentionPatterns: ["reisponde", "@openclaw"] },
      },
    ],
  },
}
```

### 命令（聊天命令处理）

```json5
{
  commands: {
    native: "auto", // 在支持时注册原生命令
    nativeSkills: "auto", // 在支持时注册原生技能命令
    text: true, // 解析聊天消息中的 /commands
    bash: false, // 允许 !（别名：/bash）
    bashForegroundMs: 2000,
    config: false, // 允许 /config
    mcp: false, // 允许 /mcp
    plugins: false, // 允许 /plugins
    debug: false, // 允许 /debug
    restart: true, // 允许 /restart + 外部 SIGUSR1 重启请求
    ownerAllowFrom: ["discord:123456789012345678"],
    ownerDisplay: "raw", // raw | hash
    ownerDisplaySecret: "${OWNER_ID_HASH_SECRET}",
    allowFrom: {
      "*": ["user1"],
      discord: ["user:123"],
    },
    useAccessGroups: true,
  },
}
```

<Accordion title="命令详情">

- 此配置块用于配置命令界面。有关当前内置和捆绑命令目录，请参阅[斜杠命令](/zh-CN/tools/slash-commands)。
- 此页面是**配置键参考**，并非完整的命令目录。由渠道/插件拥有的命令，例如 QQ Bot `/bot-ping` `/bot-help` `/bot-logs`、LINE `/card`、设备配对 `/pair`、记忆 `/dreaming`、手机控制 `/phone` 和 Talk `/voice`，记录在各自的渠道/插件页面以及[斜杠命令](/zh-CN/tools/slash-commands)中。
- 文本命令必须是以 `/` 开头的**独立**消息。
- `native: "auto"` 会为 Discord/Telegram 启用原生命令，而 Slack 保持禁用。
- `nativeSkills: "auto"` 会为 Discord/Telegram 启用原生技能命令，而 Slack 保持禁用。
- 按渠道覆盖：`channels.discord.commands.native`（布尔值或 `"auto"`）。对于 Discord，`false` 会在启动期间跳过原生命令注册和清理。
- 使用 `channels.<provider>.commands.nativeSkills` 可按渠道覆盖原生技能注册。
- `channels.telegram.customCommands` 会添加额外的 Telegram Bot 菜单项。
- `bash: true` 会为主机 shell 启用 `! <cmd>`。要求启用 `tools.elevated.enabled`，且发送者位于 `tools.elevated.allowFrom.<channel>` 中。
- `config: true` 会启用 `/config`（读取/写入 `openclaw.json`）。对于 Gateway 网关 `chat.send` 客户端，持久化 `/config set|unset` 写入还需要 `operator.admin`；只读 `/config show` 对具有普通写入权限范围的操作员客户端仍然可用。
- `mcp: true` 会为 `mcp.servers` 下由 OpenClaw 管理的 MCP 服务器配置启用 `/mcp`。
- `plugins: true` 会启用 `/plugins`，用于插件发现、安装以及启用/禁用控制。
- `channels.<provider>.configWrites` 会按渠道控制配置变更（默认值：true）。
- 对于多账户渠道，`channels.<provider>.accounts.<id>.configWrites` 还会控制以该账户为目标的写入（例如 `/allowlist --config --account <id>` 或 `/config set channels.<provider>.accounts.<id>...`）。
- `restart: false` 会禁用 `/restart` 和外部 `SIGUSR1` 重启请求。默认值：`true`。
- `ownerAllowFrom` 是仅限所有者命令和受所有者门控的渠道操作所使用的显式所有者允许列表。它与 `allowFrom` 相互独立。
- `ownerDisplay: "hash"` 会在系统提示词中对所有者 ID 进行哈希处理。设置 `ownerDisplaySecret` 可控制哈希处理。
- `allowFrom` 按提供商设置。设置后，它将成为**唯一**授权来源（渠道允许列表/配对和 `useAccessGroups` 会被忽略）。
- 当未设置 `allowFrom` 时，`useAccessGroups: false` 允许命令绕过访问组策略。
- 命令文档地图：
  - 内置和捆绑目录：[斜杠命令](/zh-CN/tools/slash-commands)
  - 渠道专属命令界面：[渠道](/zh-CN/channels)
  - QQ Bot 命令：[QQ Bot](/zh-CN/channels/qqbot)
  - 配对命令：[配对](/zh-CN/channels/pairing)
  - LINE 卡片命令：[LINE](/zh-CN/channels/line)
  - 记忆 Dreaming：[Dreaming](/zh-CN/concepts/dreaming)

</Accordion>

---

## 相关内容

- [配置参考](/zh-CN/gateway/configuration-reference) — 顶层键
- [配置 — 智能体](/zh-CN/gateway/config-agents)
- [渠道概览](/zh-CN/channels)
