---
read_when:
    - 正在开发 Tlon/Urbit 渠道功能
summary: Tlon/Urbit 支持状态、能力和配置
title: Tlon
x-i18n:
    generated_at: "2026-07-26T06:39:04Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: d742628d6cf9aaf82d79a8d96b1685229905e9452c9fc4d3a494d2dee8d69943
    source_path: channels/tlon.md
    workflow: 16
---

Tlon 是一款基于 Urbit 构建的去中心化即时通讯工具。OpenClaw 会连接到你的 Urbit ship，并
响应私信和群聊消息。默认情况下，群组回复需要 @ 提及，同时还会应用
授权规则和所有者审批流程。

状态：内置插件。支持私信、群组提及、话题串、富文本、图片上传/下载以及
所有者审批系统。不支持表情回应和投票。

## 内置插件

当前 OpenClaw 版本已内置 Tlon；打包构建无需单独安装。

对于未包含该插件的旧版构建或自定义安装，请从 npm 安装：

```bash
openclaw plugins install @openclaw/tlon
```

使用不带版本号的包名可跟随当前发布标签。仅在需要可复现安装时固定版本（`@openclaw/tlon@x.y.z`）。

从本地检出安装：

```bash
openclaw plugins install ./path/to/local/tlon-plugin
```

详情：[插件](/zh-CN/tools/plugin)

## 设置

```bash
openclaw channels add --channel tlon --ship ~sampel-palnet --url https://your-ship-host --code lidlut-tabwed-pillex-ridrup
```

或者直接编辑配置：

```json5
{
  channels: {
    tlon: {
      enabled: true,
      ship: "~sampel-palnet",
      url: "https://your-ship-host",
      code: "lidlut-tabwed-pillex-ridrup",
      ownerShip: "~your-main-ship", // 建议：设为你的 ship，始终获得授权
    },
  },
}
```

直接编辑配置后，请重启 Gateway 网关。然后向机器人发送私信，或在群组
频道中 @ 提及它。

## 入站持久性

OpenClaw 会在分派给智能体之前持久化已接受的 Tlon 私信和群聊事件。待处理或可重试的轮次可在 Gateway 网关重启后继续执行，并且每个群组频道或私信对端的工作仍会串行处理。只要队列记录或保留的完成记录仍然存在，稳定的 Urbit 消息 ID 也会抑制重复投递的事件。

从队列到智能体的边界采用至少一次投递语义：交接期间发生崩溃可能导致轮次重放。因此，在可行的情况下，产生外部副作用的智能体操作应保持幂等。

## 私有网络/LAN 中的 ship

默认情况下，OpenClaw 会阻止私有/内部主机名和 IP 范围，以防范 SSRF。如果你的
ship 运行在私有网络上（localhost、LAN IP、内部主机名），请明确选择启用：

```json5
{
  channels: {
    tlon: {
      url: "http://localhost:8080",
      network: {
        dangerouslyAllowPrivateNetwork: true,
      },
    },
  },
}
```

适用于 `http://localhost:8080`、`http://192.168.x.x:8080` 和
`http://my-ship.local:8080` 等目标。仅对你信任的 ship URL 启用此选项；它会禁用
该账户 HTTP 请求的 SSRF 防护。

<Note>
`channels.tlon.allowPrivateNetwork`（扁平键）已停用。`openclaw doctor --fix` 会自动将其移至
`channels.tlon.network.dangerouslyAllowPrivateNetwork`。
</Note>

## 群组频道

手动固定频道，或开启自动发现：

```json5
{
  channels: {
    tlon: {
      groupChannels: ["chat/~host-ship/general", "chat/~host-ship/support"],
      autoDiscoverChannels: true,
    },
  },
}
```

未在配置中设置时，`autoDiscoverChannels` 默认为 `false`；设置向导中的
提示默认为“是”，并明确写入 `true`。启用后，OpenClaw 会在启动时探查已加入的群组，
在接受群组邀请时监视新频道，并每 2 分钟重新检查一次。

## 访问控制

私信允许列表（为空 = 除非发送者是 `ownerShip`，否则不允许私信）：

```json5
{
  channels: {
    tlon: {
      dmAllowlist: ["~zod", "~nec"],
    },
  },
}
```

默认情况下，每个频道的群组授权均为 `restricted`。通过 `defaultAuthorizedShips` 设置
基准规则，并按频道 nest 覆盖：

```json5
{
  channels: {
    tlon: {
      defaultAuthorizedShips: ["~zod"],
      authorization: {
        channelRules: {
          "chat/~host-ship/general": {
            mode: "restricted",
            allowedShips: ["~zod", "~nec"],
          },
          "chat/~host-ship/announcements": {
            mode: "open",
          },
        },
      },
    },
  },
}
```

机器人在话题串内回复后，会继续响应该话题串中的后续消息，
无需再次提及。

将 `channels.tlon.implicitMentions.threadParticipation: false` 设为要求这些后续消息
重新明确提及。账户级覆盖使用 `channels.tlon.accounts.<id>.implicitMentions`。Tlon
目前不会产生 `replyToBot` 或 `quotedBot` 事实，因此这些标志在此处不起作用。

## 所有者和审批系统

```json5
{
  channels: {
    tlon: {
      ownerShip: "~your-main-ship",
    },
  },
}
```

所有者 ship 在所有位置均获得授权：私信邀请始终自动接受，群组邀请
始终自动接受，频道消息也始终通过授权。所有者无需位于
`dmAllowlist`、`defaultAuthorizedShips` 或 `groupInviteAllowlist` 中。

设置 `ownerShip` 后，未经授权的请求不会被直接丢弃，而是进入待处理
审批队列，并向所有者发送私信：

- 来自不在 `dmAllowlist` 中的 ship 的私信请求
- 发送者未通过授权的频道提及
- 来自不在 `groupInviteAllowlist` 中的 ship 的群组邀请（关闭自动接受时，或已开启但
  邀请者不在允许列表中时）

所有者通过私信回复来处理请求：

| 所有者回复                   | 效果                                                 |
| ---------------------------- | ---------------------------------------------------- |
| `approve` / `deny` / `block` | 处理最近一项待审批请求                               |
| `approve <id>` / `deny <id>` | 按 ID 处理特定审批请求                               |
| `block`                      | 同时在原生层面阻止该 ship，使其无法重新连接          |
| `unblock ~ship`              | 解除原生阻止                                         |
| `blocked`                    | 列出当前被阻止的 ship                                |
| `pending`                    | 列出待处理的审批请求                                 |

如果未配置 `ownerShip`，未经授权的私信和频道提及只会被丢弃并记录日志；
不会出现审批提示。

## 自动接受设置

自动接受来自已在 `dmAllowlist` 中的 ship 的私信邀请（无论此标志如何设置，所有者
始终会被自动接受）：

```json5
{
  channels: {
    tlon: {
      autoAcceptDmInvites: true,
    },
  },
}
```

自动接受来自允许列表的群组邀请（失败时关闭：当 `autoAcceptGroupInvites: true` 已启用且
`groupInviteAllowlist` 为空时，不接受任何非所有者邀请）：

```json5
{
  channels: {
    tlon: {
      autoAcceptGroupInvites: true,
      groupInviteAllowlist: ["~zod"],
    },
  },
}
```

## 通过 Urbit 设置存储热重载

上述大多数设置（`dmAllowlist`、`groupInviteAllowlist`、`groupChannels`、
`defaultAuthorizedShips`、`autoDiscoverChannels`、`autoAcceptDmInvites`、
`autoAcceptGroupInvites`、`ownerShip`、`showModelSignature`）会在首次运行时镜像到 ship 的
`%settings` 智能体（desk 为 `moltbot`，bucket 为 `tlon`），之后会从中实时读取，
因此通过 Landscape 客户端或内置 Skills 的设置命令所做的更改无需重启
Gateway 网关即可生效。`channelRules` 和待处理审批也会以 JSON 形式持久化到其中。对于从未写入设置存储的值，
文件配置仍是事实来源。

## 投递目标（CLI/定时任务）

与 `openclaw message send` 或定时任务投递配合使用：

- 私信：`~sampel-palnet` 或 `dm/~sampel-palnet`
- 群组：`chat/~host-ship/channel` 或 `group:~host-ship/channel`

## 内置 Skills

该插件内置了 [`@tloncorp/tlon-skill`](https://github.com/tloncorp/tlon-skill)，这是一个用于
直接执行 Urbit 操作的 CLI，安装插件后即可自动使用：

- **动态**：提及、回复、未读消息
- **频道**：列出、创建、重命名
- **联系人**：列出/获取/更新个人资料
- **群组**：创建、加入、邀请/请求流程、角色
- **Hooks**：管理频道 Hooks
- **消息**：历史记录、搜索
- **私信**：发送、表情回应、接受/拒绝
- **帖子**：表情回应、删除
- **笔记本**：发布到日记频道
- **设置**：通过上述设置存储热重载插件配置

## 能力

| 功能            | 状态                                           |
| --------------- | ---------------------------------------------- |
| 私信            | 支持                                           |
| 群组/频道       | 支持（默认要求提及）                           |
| 话题串          | 支持（加入后会持续回复）                       |
| 富文本          | Markdown 会转换为 Tlon 原生格式                |
| 图片            | 入站时下载，出站时上传                         |
| 表情回应        | 仅可通过[内置 Skills](#bundled-skill)使用      |
| 投票            | 不支持                                         |
| 原生命令        | 默认仅限所有者                                 |

## 故障排查

```bash
openclaw status
openclaw gateway status
openclaw logs --follow
openclaw doctor
```

常见故障：

- **私信被忽略**：发送者不在 `dmAllowlist` 中，且未配置用于审批流程的 `ownerShip`。
- **群组消息被忽略**：频道未被发现/固定，或发送者未通过授权且没有
  `ownerShip` 可将请求加入审批队列。
- **连接错误**：检查 ship URL 是否可访问；对于本地 ship，请设置
  `network.dangerouslyAllowPrivateNetwork`。
- **身份验证错误**：登录代码会轮换，请从你的 ship 复制当前代码。

## 配置参考

完整配置：[配置](/zh-CN/gateway/configuration)

| 键                                                     | 含义                                                           |
| ------------------------------------------------------ | -------------------------------------------------------------- |
| `channels.tlon.enabled`                                | 启用/禁用频道启动。                                            |
| `channels.tlon.ship`                                   | 机器人的 Urbit ship 名称（例如 `~sampel-palnet`）。          |
| `channels.tlon.url`                                    | Ship URL（例如 `https://sampel-palnet.tlon.network`）。                           |
| `channels.tlon.code`                                   | Ship 登录代码。                                                 |
| `channels.tlon.network.dangerouslyAllowPrivateNetwork` | 允许 localhost/LAN ship URL（选择启用 SSRF）。                  |
| `channels.tlon.ownerShip`                              | 所有者 ship：始终获得授权并接收审批请求。                       |
| `channels.tlon.dmAllowlist`                            | 允许发送私信的 ship（为空 = 除所有者外均不允许）。              |
| `channels.tlon.autoAcceptDmInvites`                    | 自动接受来自 `dmAllowlist` 中 ship 的私信。                |
| `channels.tlon.autoAcceptGroupInvites`                 | 自动接受来自 `groupInviteAllowlist` 的群组邀请。                    |
| `channels.tlon.groupInviteAllowlist`                   | 群组邀请会被自动接受的 ship。                                  |
| `channels.tlon.autoDiscoverChannels`                   | 自动发现已加入的群组频道（默认值：`false`）。        |
| `channels.tlon.implicitMentions.threadParticipation`   | 允许已参与话题串中的后续消息绕过提及要求。                       |
| `channels.tlon.groupChannels`                          | 手动固定的频道 nest。                                          |
| `channels.tlon.defaultAuthorizedShips`                 | 获得所有频道授权的 ship（没有匹配规则时使用）。                 |
| `channels.tlon.authorization.channelRules`             | 每个频道 nest 的身份验证模式 + 允许列表。                       |
| `channels.tlon.showModelSignature`                     | 在回复中附加 `_[Generated by <model>]_`。                              |
| `channels.tlon.responsePrefix`                         | 添加到出站回复开头的静态前缀。                                 |
| `channels.tlon.accounts.<id>`                          | 其他命名账户（多 ship 设置）。                                 |

## 注意事项

- 群组回复需要 @ 提及（例如 `~your-bot-ship`），除非机器人已加入该话题串。
- 话题串回复会发送到话题串内；机器人还会获取该话题串最近 10 条消息的上下文，并将其前置
  提供给智能体。
- 富文本（粗体、斜体、代码、标题、列表）会转换为 Tlon 的原生格式。
- 发送要求生成频道摘要的入站消息（例如“总结此
  频道”）会触发内置的历史记录摘要功能，而不是常规回复流程。

## 相关内容

- [渠道概览](/zh-CN/channels) — 所有支持的渠道
- [配对](/zh-CN/channels/pairing) — 私信身份验证和配对流程
- [群组](/zh-CN/channels/groups) — 群聊行为和提及门控
- [频道路由](/zh-CN/channels/channel-routing) — 消息的会话路由
- [安全](/zh-CN/gateway/security) — 访问模型和安全加固
