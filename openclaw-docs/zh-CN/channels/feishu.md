---
read_when:
    - 你想要连接一个 Feishu/Lark Bot
    - 你正在配置 Feishu 渠道
summary: Feishu 机器人概览、功能和配置
title: Feishu
x-i18n:
    generated_at: "2026-07-26T06:37:46Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 5e7c4cbb704ce266b7c7b0f6e160c36c873050fee8d5808965e15b56ad637f28
    source_path: channels/feishu.md
    workflow: 16
---

OpenClaw 通过官方 `@openclaw/feishu` 插件连接到 Feishu/Lark（一体化协作平台）：支持 Bot 私信、群聊、流式卡片回复，以及 Feishu 文档/wiki/云盘/多维表格工具。

**状态：**Bot 私信和群聊功能已可用于生产环境。WebSocket 是默认事件传输方式（无需公共 URL）；也可选择 webhook 模式。

## 快速开始

<Note>
需要 OpenClaw 2026.5.29 或更高版本。运行 `openclaw --version` 进行检查。使用 `openclaw update` 升级。
</Note>

<Steps>
  <Step title="运行渠道设置向导">
  ```bash
  openclaw channels login --channel feishu
  ```
  如果尚未安装 `@openclaw/feishu` 插件，此命令会先安装该插件，然后引导完成设置：

- **手动设置**：粘贴来自 Feishu Open Platform（`https://open.feishu.cn`）或 Lark Developer（`https://open.larksuite.com`）的 App ID 和 App Secret。
- **二维码设置**：在 Feishu 应用中扫描二维码以自动创建 Bot。此流程会将私信限定为仅你自己的账号可用（将 `dmPolicy: "allowlist"` 与你的 `open_id` 配合使用）。

向导还会询问 API 域（Feishu 或 Lark）和群组策略。如果中国大陆版 Feishu 移动应用扫描二维码后没有反应，请重新运行设置并选择手动设置。
</Step>

  <Step title="设置完成后，重启 Gateway 网关以应用更改">
  ```bash
  openclaw gateway restart
  ```
  </Step>
</Steps>

## 入站持久性

OpenClaw 会先将经过身份验证的 `im.message.receive_v1` 和 `drive.notice.comment_add_v1` 信封持久排队，然后再分派给智能体。待处理或可重试的事件可在 Gateway 网关重启后保留，并按聊天或文档保持串行处理；只要活动或保留的完成记录仍然存在，就会使用 Feishu 的事件 ID 阻止重复的队列条目。

如果 WebSocket 事件在有限次数的重试后仍无法持久化，OpenClaw 会关闭该套接字并强制建立新的已验证连接，而不会越过尚未提交的轮次继续处理。其他 Feishu 事件类型（包括表情回应和 VC 会议邀请）使用其常规事件路径，不享有此持久队列保证。

## 访问控制

### 私信

配置 `channels.feishu.dmPolicy`（默认值：`pairing`）以控制谁可以向 Bot 发送私信：

| 值         | 行为                                                                                                      |
| ------------- | ------------------------------------------------------------------------------------------------------------- |
| `"pairing"`   | 未知用户会收到配对码；通过 CLI 批准                                                         |
| `"allowlist"` | 仅 `allowFrom` 中列出的用户可以聊天                                                                     |
| `"open"`      | 公开私信；配置验证要求 `allowFrom` 包含 `"*"`。非通配符条目仍会缩小访问范围 |

**批准配对请求：**

```bash
openclaw pairing list feishu
openclaw pairing approve feishu <CODE>
```

### 群聊

**群组策略**（`channels.feishu.groupPolicy`，默认值：`allowlist`）：

| 值         | 行为                                                                                     |
| ------------- | -------------------------------------------------------------------------------------------- |
| `"open"`      | 回复群组中的所有消息                                                            |
| `"allowlist"` | 仅回复 `groupAllowFrom` 中的群组，或在 `groups.<chat_id>` 下明确配置的群组 |
| `"disabled"`  | 禁用所有群组消息；明确的 `groups.<chat_id>` 条目不会覆盖此设置         |

**提及要求**（`channels.feishu.requireMention`）：

- 默认：需要 @提及，但有效群组策略为 `"open"` 时除外；此时默认值为 `false`，以便无法携带提及的消息（例如图片）仍能到达智能体。
- 明确设置 `true` 或 `false` 可覆盖默认值；按群组覆盖：`channels.feishu.groups.<chat_id>.requireMention`。
- 仅广播的 `@all` 和 `@_all` 不视为提及 Bot。同时直接提及 `@all` 和 Bot 的消息仍视为提及 Bot。

## 群组配置示例

### 允许所有群组，无需 @提及

```json5
{
  channels: {
    feishu: {
      groupPolicy: "open", // 在 "open" 下，requireMention 默认为 false
    },
  },
}
```

### 允许所有群组，但仍要求 @提及

```json5
{
  channels: {
    feishu: {
      groupPolicy: "open",
      requireMention: true,
    },
  },
}
```

### 仅允许特定群组

```json5
{
  channels: {
    feishu: {
      groupPolicy: "allowlist",
      // 群组 ID 的格式类似：oc_xxx
      groupAllowFrom: ["oc_xxx", "oc_yyy"],
    },
  },
}
```

在 `allowlist` 模式下，还可以通过添加明确的 `groups.<chat_id>` 条目来准入群组。明确条目不会覆盖 `groupPolicy: "disabled"`。`groups.*` 下的通配符默认设置会配置匹配的群组，但其本身不会准入群组。

```json5
{
  channels: {
    feishu: {
      groupPolicy: "allowlist",
      groups: {
        oc_xxx: {
          requireMention: false,
        },
      },
    },
  },
}
```

### 限制群组内的发送者

```json5
{
  channels: {
    feishu: {
      groupPolicy: "allowlist",
      groupAllowFrom: ["oc_xxx"],
      groups: {
        oc_xxx: {
          // 用户 open_id 的格式类似：ou_xxx
          allowFrom: ["ou_user1", "ou_user2"],
        },
      },
    },
  },
}
```

`channels.feishu.groupSenderAllowFrom` 为所有群组设置相同的发送者允许列表；按群组设置的 `allowFrom` 优先级更高。

### Bot 发送的消息

默认情况下，Feishu 会忽略其他 Bot 发送的消息。要允许 Bot 之间进行群组对话，请向应用授予 `im:message.group_at_msg.include_bot:readonly` 和 `im:message:readonly` 权限范围，然后设置 `allowBots`：

```json5
{
  channels: {
    feishu: {
      allowBots: true,
    },
  },
}
```

仅当另一个 Bot 提及当前 Bot 时，Feishu 才会传递由 Bot 发送的群组事件。现有的群组策略、发送者允许列表和提及要求仍然适用。OpenClaw 会丢弃自己发送的消息，在每个文本或卡片回复中提及对端 Bot，并应用共享的 [`channels.defaults.botLoopProtection`](/zh-CN/channels/bot-loop-protection) 防护机制。

<a id="get-groupuser-ids"></a>

## 获取群组/用户 ID

### 群组 ID（`chat_id`，格式：`oc_xxx`）

在 Feishu/Lark 中打开群组，单击右上角的菜单图标，然后转到 **Settings**。群组 ID（`chat_id`）列在设置页面中。

![获取群组 ID](/images/feishu-get-group-id.png)

### 用户 ID（`open_id`，格式：`ou_xxx`）

启动 Gateway 网关，向 Bot 发送私信，然后检查日志：

```bash
openclaw logs --follow
```

在日志输出中查找 `open_id`。也可以检查待处理的配对请求：

```bash
openclaw pairing list feishu
```

## 常用命令

| 命令   | 描述                 |
| --------- | --------------------------- |
| `/status` | 显示 Bot 状态             |
| `/reset`  | 重置当前会话   |
| `/model`  | 显示或切换 AI 模型 |

<Note>
Feishu/Lark 不支持原生斜杠命令菜单，因此请将这些命令作为纯文本消息发送。
</Note>

## 故障排查

### Bot 在群聊中没有响应

1. 确保已将 Bot 添加到群组
2. 确保 @提及 Bot（默认要求）
3. 确认 `groupPolicy` 不是 `"disabled"`
4. 检查日志：`openclaw logs --follow`

### Bot 未收到消息

1. 确保 Bot 已在 Feishu Open Platform / Lark Developer 中发布并获得批准
2. 确保事件订阅包含 `im.message.receive_v1`
3. 如需自动加入会议邀请，还要订阅 `vc.bot.meeting_invited_v1`
4. 确保已选择 **persistent connection**（WebSocket）
5. 确保已授予所有必需的权限范围
6. 确保 Gateway 网关正在运行：`openclaw gateway status`
7. 检查日志：`openclaw logs --follow`

订阅 `vc.bot.meeting_invited_v1` 只会传递事件。自动加入功能
默认关闭。要全局启用：

```json5
{
  channels: {
    feishu: {
      vcAutoJoin: true,
    },
  },
}
```

如果只为一个账号启用，请省略顶层开关并设置账号覆盖项：

```json5
{
  channels: {
    feishu: {
      accounts: {
        meetings: { vcAutoJoin: true },
      },
    },
  },
}
```

在智能体收到加入轮次之前，邀请者仍需经过正常的 Feishu 私信策略、允许列表/配对、会话和回复
路由。加入会议还要求配置一个可用的 Feishu VC 加入
工具，该工具须使用应用身份并具有
`vc:meeting.bot.join:write` 权限范围。例如，官方
[`lark-cli` VC 智能体技能](https://github.com/larksuite/cli/tree/main/skills/lark-vc-agent)
提供 `vc +meeting-join`。

<Warning>
官方 `lark-cli` VC 智能体技能目前将会议 Bot 操作标记为有限测试版。如果工具返回 `ErrNotInGray` 或错误代码 `20017`，则表示该应用或租户尚未获准使用此测试版；在排查常规权限范围授予问题之前，请先按照所链接技能中的抢先体验指南操作。
</Warning>

### Feishu 移动应用扫描二维码后没有反应

1. 重新运行设置：`openclaw channels login --channel feishu`
2. 选择手动设置
3. 在 Feishu Open Platform 中创建自建应用，并复制其 App ID 和 App Secret
4. 将这些凭据粘贴到设置向导中

### App Secret 泄露

1. 在 Feishu Open Platform / Lark Developer 中重置 App Secret
2. 更新配置中的值
3. 重启 Gateway 网关：`openclaw gateway restart`

## 高级配置

### 多账号

```json5
{
  channels: {
    feishu: {
      defaultAccount: "main",
      accounts: {
        main: {
          appId: "cli_xxx",
          appSecret: "xxx",
          name: "Primary bot",
          tts: {
            providers: {
              openai: { voice: "shimmer" },
            },
          },
        },
        backup: {
          appId: "cli_yyy",
          appSecret: "yyy",
          name: "Backup bot",
          enabled: false,
        },
      },
    },
  },
}
```

当出站 API 未指定 `accountId` 时，`defaultAccount` 控制使用哪个账号。账号条目会继承顶层设置；大多数顶层键都可以按账号覆盖。
`accounts.<id>.tts` 与 `tts` 使用相同的结构，并在全局 TTS 配置上进行深度合并，因此多 Bot Feishu 设置可以将共享的提供商凭据保留在全局配置中，而仅按账号覆盖语音、模型、角色设定或自动模式。

### 消息限制

- `textChunkLimit` - 出站文本分块大小（默认：`4000` 个字符）
- `streaming.chunkMode` - `"length"`（默认）在达到限制时拆分；`"newline"` 优先在换行边界处拆分
- `mediaMaxMb` - 媒体上传/下载限制（默认：`30` MB）

### 流式传输

Feishu/Lark 支持通过交互式卡片进行流式回复（Card Kit 流式 API）。启用后，Bot 会在生成文本时实时更新卡片。

```json5
{
  channels: {
    feishu: {
      streaming: {
        mode: "partial", // 流式卡片输出（默认："partial"）
        block: { enabled: true }, // 选择启用已完成块的流式传输
      },
    },
  },
}
```

将 `streaming.mode: "off"` 设置为在一条消息中发送完整回复；`renderMode: "raw"`（使用纯文本而非卡片）也会禁用流式卡片。`streaming.block.enabled` 默认关闭；仅当你希望在最终回复之前发送已完成的助手内容块时才启用它。旧版布尔值 `streaming` 以及扁平的 `blockStreaming` / `blockStreamingCoalesce` / `chunkMode` 键会通过 `openclaw doctor --fix` 迁移到这一嵌套结构。

### 配额优化

使用两个可选标志减少 Feishu/Lark API 调用次数：

- `typingIndicator`（默认值为 `true`）：设置为 `false` 可跳过输入状态表情回应调用
- `resolveSenderNames`（默认值为 `true`）：设置为 `false` 可跳过发送者资料查询

```json5
{
  channels: {
    feishu: {
      typingIndicator: false,
      resolveSenderNames: false,
    },
  },
}
```

### 群组会话范围和话题线程

`channels.feishu.groupSessionScope`（顶层、每个账户或每个群组）控制群组消息如何映射到 Agent 会话：

| 值                     | 会话                                                             |
| ---------------------- | ---------------------------------------------------------------- |
| `"group"`（默认）    | 每个群聊一个会话                                       |
| `"group_sender"`       | 每个（群组 + 发送者）一个会话                                 |
| `"group_topic"`        | 每个话题线程一个会话；回退到群组会话    |
| `"group_topic_sender"` | 每个（话题 + 发送者）一个会话；回退到（群组 + 发送者） |

对于话题范围，Feishu/Lark 原生话题群组使用事件 `thread_id`（`omt_*`）作为规范的话题会话键。如果原生话题发起事件省略了 `thread_id`，OpenClaw 会先从 Feishu 补充该值，再路由该轮次。由 OpenClaw 转换为线程的普通群组回复继续使用回复根消息 ID（`om_*`），以便首轮和后续轮次保持在同一会话中。

设置 `replyInThread: "enabled"`（顶层或每个群组），可让 Bot 回复创建或继续 Feishu 话题线程，而不是进行内联回复。`topicSessionMode` 是 `groupSessionScope` 的已弃用前身；请优先使用 `groupSessionScope`。

### Feishu 工作区工具

该插件提供用于 Feishu 文档、聊天、知识库、云存储、权限和 Bitable 的 Agent 工具，以及对应的 Skills（`feishu-doc`、`feishu-drive`、`feishu-perm`、`feishu-wiki`）。工具族由 `channels.feishu.tools` 控制：

| 键              | 工具                                          | 默认值              |
| --------------- | --------------------------------------------- | ------------------- |
| `tools.doc`     | `feishu_doc` 文档操作              | `true`              |
| `tools.chat`    | `feishu_chat` 聊天信息 + 成员查询      | `true`              |
| `tools.wiki`    | `feishu_wiki` 知识库（需要 `doc`） | `true`              |
| `tools.drive`   | `feishu_drive` 云存储                  | `true`              |
| `tools.perm`    | `feishu_perm` 权限管理           | `false`（敏感） |
| `tools.scopes`  | `feishu_app_scopes` 应用权限范围诊断     | `true`              |
| `tools.bitable` | `feishu_bitable_*` Bitable/Base 操作    | `true`              |

`tools.base` 是 `tools.bitable` 的别名；同时设置两者时，以显式的 `bitable` 值为准。每账户控制项位于 `accounts.<id>.tools` 下。

对于根目录之外的直接 `feishu_drive info` 查询，请授予 `drive:drive.metadata:readonly`，除非应用已具有完整的 `drive:drive` 权限范围。如果这两个权限范围均未授予，`info`
仍可通过 `drive:drive:readonly` 使用旧版根目录查询。

### ACP 会话

Feishu/Lark 支持在私信和群组线程消息中使用 ACP。Feishu/Lark ACP 由文本命令驱动——没有原生斜杠命令菜单，因此请直接在对话中使用 `/acp ...` 消息。

#### 持久 ACP 绑定

```json5
{
  agents: {
    list: [
      {
        id: "codex",
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
    ],
  },
  bindings: [
    {
      type: "acp",
      agentId: "codex",
      match: {
        channel: "feishu",
        accountId: "default",
        peer: { kind: "direct", id: "ou_1234567890" },
      },
    },
    {
      type: "acp",
      agentId: "codex",
      match: {
        channel: "feishu",
        accountId: "default",
        peer: { kind: "group", id: "oc_group_chat:topic:om_topic_root" },
      },
      acp: { label: "codex-feishu-topic" },
    },
  ],
}
```

#### 从聊天中生成 ACP

在 Feishu/Lark 私信或线程中：

```text
/acp spawn codex --thread here
```

`--thread here` 适用于私信和 Feishu/Lark 线程消息。绑定对话中的后续消息会直接路由到该 ACP 会话。

### 多 Agent 路由

使用 `bindings` 将 Feishu/Lark 私信或群组路由到不同的 Agent。

```json5
{
  agents: {
    list: [
      { id: "main" },
      { id: "agent-a", workspace: "/home/user/agent-a" },
      { id: "agent-b", workspace: "/home/user/agent-b" },
    ],
  },
  bindings: [
    {
      agentId: "agent-a",
      match: {
        channel: "feishu",
        peer: { kind: "direct", id: "ou_xxx" },
      },
    },
    {
      agentId: "agent-b",
      match: {
        channel: "feishu",
        peer: { kind: "group", id: "oc_zzz" },
      },
    },
  ],
}
```

路由字段：

- `match.channel`：`"feishu"`
- `match.peer.kind`：`"direct"`（私信）或 `"group"`（群聊）
- `match.peer.id`：用户 Open ID（`ou_xxx`）或群组 ID（`oc_xxx`）

有关查询提示，请参阅[获取群组/用户 ID](#get-groupuser-ids)。

## 每用户 Agent 隔离（动态 Agent 创建）

启用 `dynamicAgentCreation`，可为每个私信用户自动创建**隔离的 Agent 实例**。每个用户都有自己的：

- 独立工作区目录
- 单独的 `USER.md` / `SOUL.md` / `MEMORY.md`
- 私有对话历史记录
- 隔离的 Skills 和状态

对于希望为每个用户提供专属私有 AI 助手体验的公共 Bot，此功能至关重要。

<Note>
动态绑定包含规范化的 Feishu `accountId`，因此默认账户和命名账户会将每位发送者路由到正确的动态 Agent。

如果某个命名账户在旧版本中创建了无范围限定的动态 Agent，该旧版 Agent 仍计入 `maxAgents`。移除它之前，请确认默认账户未使用它；或者暂时提高 `maxAgents`。OpenClaw 无法安全推断存在歧义的旧版状态属于哪个账户。
</Note>

### 快速设置

```json5
{
  channels: {
    feishu: {
      dmPolicy: "open",
      allowFrom: ["*"],
      dynamicAgentCreation: {
        enabled: true,
        workspaceTemplate: "~/.openclaw/workspace-{agentId}",
        agentDirTemplate: "~/.openclaw/agents/{agentId}/agent",
      },
    },
  },
  session: {
    // 关键：将每个用户的私信设为其“主会话”
    // 自动加载 USER.md / SOUL.md / MEMORY.md
    // 如需更强的隔离，请改用 "per-channel-peer"
    dmScope: "main",
  },
}
```

### 工作原理

当新用户发送第一条私信时：

1. 渠道生成唯一的 `agentId`：默认账户使用 `feishu-{user_open_id}`，命名账户则使用长度受限且带账户前缀的身份摘要
2. 在 `workspaceTemplate` 路径创建新工作区
3. 注册 Agent，并为该用户创建绑定
4. 工作区辅助程序会在首次访问时确保引导文件（`AGENTS.md`、`SOUL.md`、`USER.md` 等）存在
5. 将该用户今后的所有消息路由到其专用 Agent

### 配置选项

| 设置                                                      | 描述                                      | 默认值                               |
| -------------------------------------------------------- | ------------------------------------------ | ------------------------------------ |
| `channels.feishu.dynamicAgentCreation.enabled`           | 启用自动创建每用户 Agent   | `false`                              |
| `channels.feishu.dynamicAgentCreation.workspaceTemplate` | 动态 Agent 工作区的路径模板 | `~/.openclaw/workspace-{agentId}`    |
| `channels.feishu.dynamicAgentCreation.agentDirTemplate`  | Agent 目录名称模板              | `~/.openclaw/agents/{agentId}/agent` |
| `channels.feishu.dynamicAgentCreation.maxAgents`         | 可创建的动态 Agent 最大数量 | 无限制                            |

模板变量：

- `{agentId}` - 生成的 Agent ID（例如 `feishu-ou_xxxxxx` 或 `feishu-support-<identity_digest>`）
- `{userId}` - 发送者的 Feishu open_id（例如 `ou_xxxxxx`）

### 会话范围

`session.dmScope` 控制私信如何映射到 Agent 会话。这是一项影响所有渠道的**全局设置**。

| 值                           | 行为                                                                | 最适合                                                           |
| ---------------------------- | ------------------------------------------------------------------- | ------------------------------------------------------------------ |
| `"main"`                     | 每个用户的私信映射到其 Agent 的主会话                   | 希望自动加载 `USER.md` / `SOUL.md` 的单用户 Bot |
| `"per-peer"`                 | 每个对端都有单独的会话（无论渠道如何）           | 仅按发送者身份进行隔离                            |
| `"per-channel-peer"`         | 每个（渠道 + 用户）组合都有单独的会话           | 需要更强隔离的公共多用户 Bot                  |
| `"per-account-channel-peer"` | 每个（账户 + 渠道 + 用户）组合都有单独的会话 | 需要账户级会话隔离的多账户 Bot         |

**权衡**：使用 `"main"` 可以自动加载引导文件（`USER.md`、`SOUL.md`、`MEMORY.md`），但这意味着所有渠道中的全部私信都共享相同的会话键模式。对于隔离比自动加载引导文件更重要的公共多用户 Bot，请考虑使用 `"per-channel-peer"`，并手动管理引导文件。

<Note>
当命名 Feishu 账户需要为同一发送者保留单独会话时，请使用 `"per-account-channel-peer"`。动态绑定会保留账户范围。
</Note>

### 典型的多用户部署

```json5
{
  channels: {
    feishu: {
      appId: "cli_xxx",
      appSecret: "xxx",
      dmPolicy: "open",
      allowFrom: ["*"],
      groupPolicy: "open",
      requireMention: true,
      dynamicAgentCreation: {
        enabled: true,
        workspaceTemplate: "~/.openclaw/workspace-{agentId}",
        agentDirTemplate: "~/.openclaw/agents/{agentId}/agent",
      },
    },
  },
  session: {
    // 根据你的隔离需求选择 dmScope：
    // 使用 "main" 自动加载引导文件，使用 "per-channel-peer" 实现更强隔离
    dmScope: "main",
  },
  bindings: [], // 留空——动态 Agent 会自动绑定
}
```

### 验证

检查 Gateway 网关日志，确认动态创建功能正常工作：

```text
feishu：正在为用户 ou_xxxxxx 创建动态 Agent "feishu-ou_xxxxxx"
  工作区：/home/user/.openclaw/workspace-feishu-ou_xxxxxx
  Agent 目录：/home/user/.openclaw/agents/feishu-ou_xxxxxx/agent
```

列出所有已创建的工作区：

```bash
ls -la ~/.openclaw/workspace-*
```

### 注意事项

- **工作区隔离**：每个用户都有自己的工作区目录和智能体实例。在正常消息流程中，用户无法查看彼此的对话历史记录或文件。
- **安全边界**：这是一种消息上下文隔离机制，而不是针对不受信任的共租户的安全边界。智能体进程和主机环境是共享的。
- **必须保持启用配置写入**：动态 Agent 创建会将智能体和绑定写入配置；当 `channels.feishu.configWrites` 为 `false` 时会跳过此操作（默认：启用）。
- **`bindings` 应为空**：动态智能体会自动注册自己的绑定
- **升级路径**：现有手动绑定可继续与动态智能体配合使用
- **`session.dmScope` 是全局设置**：这会影响所有渠道，而不仅仅是 Feishu

## 配置参考

完整配置：[Gateway 配置](/zh-CN/gateway/configuration)

| 设置                                                     | 描述                                                                                 | 默认值                               |
| -------------------------------------------------------- | ------------------------------------------------------------------------------------ | ------------------------------------ |
| `channels.feishu.enabled`                                | 启用/禁用渠道                                                                        | `true`                               |
| `channels.feishu.domain`                                 | API 域（`feishu`、`lark` 或 `https://` 基础 URL）                             | `feishu`                             |
| `channels.feishu.connectionMode`                         | 事件传输方式（`websocket` 或 `webhook`）                                           | `websocket`                          |
| `channels.feishu.defaultAccount`                         | 出站路由的默认账户                                                                   | `default`                            |
| `channels.feishu.verificationToken`                      | webhook 模式必需                                                                    | -                                    |
| `channels.feishu.encryptKey`                             | webhook 模式必需                                                                    | -                                    |
| `channels.feishu.webhookPath`                            | Webhook 路由路径                                                                     | `/feishu/events`                     |
| `channels.feishu.webhookHost`                            | Webhook 绑定主机                                                                     | `127.0.0.1`                          |
| `channels.feishu.webhookPort`                            | Webhook 绑定端口                                                                     | `3000`                               |
| `channels.feishu.accounts.<id>.appId`                    | 应用 ID                                                                              | -                                    |
| `channels.feishu.accounts.<id>.appSecret`                | 应用 Secret                                                                          | -                                    |
| `channels.feishu.accounts.<id>.domain`                   | 每账户域覆盖                                                                         | `feishu`                             |
| `channels.feishu.accounts.<id>.tts`                      | 每账户 TTS 覆盖                                                                      | `tts`                                |
| `channels.feishu.dmPolicy`                               | 私信策略（`pairing`、`allowlist`、`open`）                                           | `pairing`                            |
| `channels.feishu.allowFrom`                              | 私信允许列表（open_id 列表）                                                         | -                                    |
| `channels.feishu.groupPolicy`                            | 群组策略（`open`、`allowlist`、`disabled`）                                       | `allowlist`                          |
| `channels.feishu.groupAllowFrom`                         | 群组允许列表                                                                         | -                                    |
| `channels.feishu.groupSenderAllowFrom`                   | 应用于所有群组的发送者允许列表                                                       | -                                    |
| `channels.feishu.requireMention`                         | 要求在群组中 @提及                                                                   | `true`（策略为 `open` 时为 `false`）  |
| `channels.feishu.allowBots`                              | 接受提及此机器人的其他机器人，并提供机器人循环保护                                   | `false`                              |
| `channels.feishu.groups.<chat_id>.requireMention`        | 每群组 @提及覆盖；在允许列表模式下，显式 ID 也会准许该群组                          | 继承                                 |
| `channels.feishu.groups.<chat_id>.enabled`               | 启用/禁用特定群组                                                                    | `true`                               |
| `channels.feishu.groups.<chat_id>.allowFrom`             | 每群组发送者允许列表（覆盖 `groupSenderAllowFrom`）                        | -                                    |
| `channels.feishu.groupSessionScope`                      | 群组会话映射（`group`、`group_sender`、`group_topic`、`group_topic_sender`） | `group`                              |
| `channels.feishu.replyInThread`                          | Bot 回复创建/继续话题线程（`disabled`、`enabled`）                    | `disabled`                           |
| `channels.feishu.reactionNotifications`                  | 入站表情回应事件（`off`、`own`、`all`）                                        | `own`                                |
| `channels.feishu.vcAutoJoin`                             | 通过正常私信授权后加入受邀的 VC 会议                                                 | `false`                              |
| `channels.feishu.dynamicAgentCreation.enabled`           | 启用自动创建每用户智能体                                                             | `false`                              |
| `channels.feishu.dynamicAgentCreation.workspaceTemplate` | 动态 Agent 工作区的路径模板                                                          | `~/.openclaw/workspace-{agentId}`    |
| `channels.feishu.dynamicAgentCreation.agentDirTemplate`  | Agent 目录名称模板                                                                   | `~/.openclaw/agents/{agentId}/agent` |
| `channels.feishu.dynamicAgentCreation.maxAgents`         | 可创建的动态智能体最大数量                                                           | 无限制                               |
| `channels.feishu.textChunkLimit`                         | 消息分块大小                                                                         | `4000`                               |
| `channels.feishu.streaming.chunkMode`                    | 分块拆分（`length` 或 `newline`）                                              | `length`                             |
| `channels.feishu.mediaMaxMb`                             | 媒体大小限制                                                                         | `30`                                 |
| `channels.feishu.renderMode`                             | 回复渲染（`auto`、`raw`、`card`）                                              | `auto`                               |
| `channels.feishu.streaming.mode`                         | 流式卡片输出（`partial` 或 `off`）                                           | `partial`                            |
| `channels.feishu.streaming.block.enabled`                | 已完成块的回复流式传输                                                               | `false`                              |
| `channels.feishu.typingIndicator`                        | 发送输入状态表情回应                                                                 | `true`                               |
| `channels.feishu.resolveSenderNames`                     | 解析发送者显示名称                                                                   | `true`                               |
| `channels.feishu.configWrites`                           | 允许渠道发起配置写入（动态智能体需要）                                               | `true`                               |
| `channels.feishu.tools.doc`                              | 启用文档工具                                                                         | `true`                               |
| `channels.feishu.tools.chat`                             | 启用聊天信息工具                                                                     | `true`                               |
| `channels.feishu.tools.wiki`                             | 启用知识库工具（需要 `doc`）                                         | `true`                               |
| `channels.feishu.tools.drive`                            | 启用云存储工具                                                                       | `true`                               |
| `channels.feishu.tools.perm`                             | 启用权限管理工具                                                                     | `false`                              |
| `channels.feishu.tools.scopes`                           | 启用应用权限范围诊断工具                                                             | `true`                               |
| `channels.feishu.tools.bitable`                          | 启用 Bitable/Base 工具                                                               | `true`                               |
| `channels.feishu.tools.base`                             | `channels.feishu.tools.bitable` 的别名；同时设置时以显式的 `bitable` 为准     | `true`                               |
| `channels.feishu.accounts.<id>.tools.bitable`            | 每账户 Bitable/Base 工具开关                                                         | 继承                                 |
| `channels.feishu.accounts.<id>.tools.base`               | `tools.bitable` 的每账户别名                                                | 继承                                 |

## 支持的消息类型

### 接收

- ✅ 文本
- ✅ 富文本（帖子）
- ✅ 图片
- ✅ 文件
- ✅ 音频
- ✅ 视频/媒体
- ✅ 贴纸

入站 Feishu/Lark 音频消息会被规范化为媒体占位符，而不是原始
`file_key` JSON。配置 `tools.media.audio` 后，OpenClaw
会下载语音留言资源，并在智能体轮次开始前运行共享音频转录，使智能体
收到语音转录文本。如果 Feishu 在音频载荷中直接包含转录文本，则会使用
该文本，而不会再次调用 ASR。若没有音频转录提供商，智能体仍会收到
`<media:audio>` 占位符和已保存的附件，而不是原始 Feishu
资源载荷。

### 发送

- ✅ 文本
- ✅ 图片
- ✅ 文件
- ✅ 音频
- ✅ 视频/媒体
- ✅ 交互式卡片（包括流式更新）
- ⚠️ 富文本（帖子式格式；不支持 Feishu/Lark 的全部创作功能）

Feishu/Lark 原生音频气泡使用 Feishu `audio` 消息类型，并要求上传
Ogg/Opus 媒体（`file_type: "opus"`）。现有的 `.opus` 和 `.ogg` 媒体
会直接作为原生音频发送。仅当回复请求以语音形式
发送时（`audioAsVoice` / 消息工具 `asVoice`，包括 TTS 语音消息
回复），才会使用 `ffmpeg` 将 MP3/WAV/M4A 和其他可能的音频格式
转码为 48kHz Ogg/Opus。普通 MP3 附件仍作为常规文件发送。如果缺少 `ffmpeg` 或
转换失败，OpenClaw 会回退为文件附件并记录原因。

### 话题串和回复

- ✅ 行内回复
- ✅ 话题串回复
- ✅ 回复话题串消息时，媒体回复仍会保留话题串上下文

主题群组会话路由详见
[群组会话范围和主题话题串](#group-session-scope-and-topic-threads)。

## 相关内容

- [渠道概览](/zh-CN/channels) - 所有支持的渠道
- [配对](/zh-CN/channels/pairing) - 私信身份验证和配对流程
- [群组](/zh-CN/channels/groups) - 群聊行为和提及门控
- [频道路由](/zh-CN/channels/channel-routing) - 消息的会话路由
- [安全](/zh-CN/gateway/security) - 访问模型和安全加固
