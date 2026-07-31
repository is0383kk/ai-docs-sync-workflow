---
read_when:
    - 为 OpenClaw 设置 Zalo Personal
    - 调试 Zalo Personal 登录或消息流程
summary: 通过原生 zca-js（二维码登录）支持 Zalo Personal 账号，包括功能和配置
title: Zalo Personal
x-i18n:
    generated_at: "2026-07-26T06:42:40Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 09cecad1a9a5b34b932c5e68e2b3164b360fb6af1dcd2fd5b5979d1b2a1bd62b
    source_path: channels/zalouser.md
    workflow: 16
---

状态：实验性。此集成通过原生 `zca-js` 在进程内自动操作**个人 Zalo 账户**，无需外部 CLI 二进制文件。

<Warning>
这是非官方集成，可能导致账户被暂停或封禁。使用风险自负。
</Warning>

## 安装

Zalo Personal 是官方外部插件，不内置于核心中。使用前请先安装：

```bash
openclaw plugins install @openclaw/zalouser
```

- 固定版本：`openclaw plugins install @openclaw/zalouser@<version>`
- 从源代码检出安装：`openclaw plugins install ./path/to/local/zalouser-plugin`
- 详细信息：[插件](/zh-CN/tools/plugin)

## 快速设置

1. 安装插件（见上文）。
2. 登录（使用二维码，在 Gateway 网关所在机器上）：
   - `openclaw channels login --channel zalouser`
   - 使用 Zalo 移动应用扫描二维码。
3. 启用渠道：

```json5
{
  channels: {
    zalouser: {
      enabled: true,
      dmPolicy: "pairing",
    },
  },
}
```

4. 重启 Gateway 网关（或完成设置）。
5. 私信访问默认采用配对；首次联系时批准配对码。

## 功能说明

- 完全通过 `zca-js` 库在进程内运行（无需外部 `zca`/`openzca` 二进制文件）。
- 使用原生事件监听器（`message`、`error`）接收入站消息。
- 通过 JS API 直接发送回复（文本/媒体/链接）。
- 专为无法使用 Zalo Bot API 的“个人账户”使用场景设计。

## 命名

渠道 ID 为 `zalouser`，以明确表示此集成自动操作的是**个人 Zalo 用户账户**（非官方）。`zalo` 保留给未来可能推出的官方 Zalo API 集成。

## 查找 ID（目录）

```bash
openclaw directory self --channel zalouser
openclaw directory peers list --channel zalouser --query "name"
openclaw directory groups list --channel zalouser --query "work"
```

## 限制

- 出站文本按 2000 个字符分块（Zalo 客户端限制）。
- 不支持流式传输。
- 已完成处理的入站消息 ID 保留 30 天，每个账户最多保留最近的 1000 条记录。

## 入站消息持久性

OpenClaw 会在处理前存储每个原始 `zca-js` 消息回调。Gateway 网关重启后，待处理消息会从账户队列恢复，并且每个私聊或群组中的处理始终保持串行。

`zca-js` 套接字监听器不提供送达确认，也不会在重新连接后自动重放旧消息。因此，持久队列只能防止回调到达 OpenClaw 后发生本地崩溃时丢失消息；无法恢复套接字从未送达的消息。重放墓碑记录主要用于防止具有相同 Zalo 消息 ID 的回调被重复处理。

## 访问控制（私信）

`channels.zalouser.dmPolicy`：`pairing | allowlist | open | disabled`（默认值：`pairing`）。

`channels.zalouser.allowFrom` 应使用稳定的 Zalo 用户 ID。它也可以引用静态发送者访问组（`accessGroup:<name>`）。在交互式设置期间，可通过插件的进程内联系人查找功能将输入的姓名解析为 ID。

如果配置中仍有原始姓名，仅当启用 `channels.zalouser.dangerouslyAllowNameMatching: true` 时，启动过程才会解析该姓名。未明确启用此选项时，运行时发送者检查仅使用 ID，并在授权时忽略原始姓名。

批准方式：

- `openclaw pairing list zalouser`
- `openclaw pairing approve zalouser <code>`

## 群组访问（可选）

- 默认值：`channels.zalouser.groupPolicy = "allowlist"`（群组需要明确的允许列表条目）。
- 开放所有群组：`channels.zalouser.groupPolicy = "open"`。
- 阻止所有群组：`channels.zalouser.groupPolicy = "disabled"`。
- 使用 `groupPolicy = "allowlist"` 时：
  - `channels.zalouser.groups` 的键应为稳定的群组 ID；仅当启用 `channels.zalouser.dangerouslyAllowNameMatching: true` 时，才会在启动时将名称解析为 ID。
  - `channels.zalouser.groupAllowFrom` 控制允许群组中的哪些发送者可以触发机器人；可通过 `accessGroup:<name>` 引用静态发送者访问组。
- 配置向导可以提示输入群组允许列表。
- 默认情况下，群组允许列表仅按 ID 匹配。除非启用 `channels.zalouser.dangerouslyAllowNameMatching: true`，否则授权时会忽略未解析的名称。
- `channels.zalouser.dangerouslyAllowNameMatching: true` 是一种紧急兼容模式，会重新启用可变的启动时名称解析和运行时群组名称匹配。
- 对于普通群组消息，`groupAllowFrom` **不会**回退到 `allowFrom`：在允许列表群组中将其留空，会允许任何发送者使用该群组。已获授权的控制命令（例如 `/new`）是例外；当 `groupAllowFrom` 为空时，命令发送者检查会回退到 `allowFrom`。

示例：

```json5
{
  channels: {
    zalouser: {
      groupPolicy: "allowlist",
      groupAllowFrom: ["1471383327500481391"],
      groups: {
        "123456789": { enabled: true },
        "Work Chat": { enabled: true },
      },
    },
  },
}
```

<Note>
`channels.zalouser.groups.<id>.allow` 是旧版字段名称；当前配置使用 `enabled`。`openclaw doctor --fix` 会自动将 `allow` 迁移到 `enabled`。
</Note>

### 群组提及门控

- `channels.zalouser.groups.<group>.requireMention` 控制群组回复是否需要提及。
- 解析顺序：群组 ID -> `group:<id>` 别名 -> 群组名称/短名称（仅当 `dangerouslyAllowNameMatching: true` 时才应用基于名称的候选项）-> `*` -> 默认值（`true`）。
- 同时适用于允许列表群组和开放群组模式。
- 引用机器人消息可视为用于激活群组的隐式提及。
- 已获授权的控制命令（例如 `/new`）可以绕过提及门控。
- 当群组消息因需要提及而被跳过时，OpenClaw 会将其存储为待处理群组历史记录，并在下一条被处理的群组消息中包含该消息。
- 群组历史记录限制：`channels.zalouser.historyLimit`，然后是 `messages.groupChat.historyLimit`，最后回退到 `50`。

示例：

```json5
{
  channels: {
    zalouser: {
      groupPolicy: "allowlist",
      groups: {
        "*": { enabled: true, requireMention: true },
        "Work Chat": { enabled: true, requireMention: false },
      },
    },
  },
}
```

## 多账户

账户映射到 OpenClaw 状态中的 `zalouser` 配置文件。示例：

```json5
{
  channels: {
    zalouser: {
      enabled: true,
      defaultAccount: "default",
      accounts: {
        work: { enabled: true, profile: "work" },
      },
    },
  },
}
```

## 环境变量

也可通过环境变量选择配置文件：

| 变量               | 用途                                                                       |
| ------------------ | -------------------------------------------------------------------------- |
| `ZALOUSER_PROFILE` | 当渠道或账户配置中未设置 `profile` 时使用的配置文件名称。 |
| `ZCA_PROFILE`      | 旧版回退项，仅在未设置 `ZALOUSER_PROFILE` 时使用。             |

配置文件名称用于选择 OpenClaw 状态中保存的 Zalo 登录凭据。解析顺序：

1. 配置中明确指定的 `profile`。
2. `ZALOUSER_PROFILE`。
3. `ZCA_PROFILE`。
4. 非默认账户使用账户 ID，默认账户使用 `default`。

对于多账户设置，建议在配置中为每个账户设置 `profile`，避免一个环境变量导致多个账户共享同一登录会话。

## 输入状态、表情回应和送达确认

- OpenClaw 会在分派回复前发送输入状态事件（尽力而为）。
- 渠道操作中的 `zalouser` 支持消息表情回应操作 `react`。
  - 使用 `remove: true` 从消息中移除特定的表情回应。
  - 表情回应语义：[表情回应](/zh-CN/tools/reactions)
- 对于包含事件元数据的入站消息，OpenClaw 会发送已送达和已读确认（尽力而为）。

## 故障排查

**登录状态无法保持：**

- `openclaw channels status --probe`
- 重新登录：`openclaw channels logout --channel zalouser && openclaw channels login --channel zalouser`

**允许列表/群组名称未解析：**

- 在 `allowFrom`/`groupAllowFrom` 中使用数字 ID，并在 `groups` 中使用稳定的群组 ID。如果确实需要使用精确的好友/群组名称，请启用 `channels.zalouser.dangerouslyAllowNameMatching: true`。

**从旧版外部 `zca`/基于 CLI 的设置升级：**

- 移除所有依赖外部 `zca` 进程的假设；该渠道现在完全通过 `zca-js` 在进程内运行，无需外部 CLI 二进制文件。

## 相关内容

- [渠道概览](/zh-CN/channels) - 所有支持的渠道
- [配对](/zh-CN/channels/pairing) - 私信身份验证和配对流程
- [群组](/zh-CN/channels/groups) - 群聊行为和提及门控
- [频道路由](/zh-CN/channels/channel-routing) - 消息的会话路由
- [安全](/zh-CN/gateway/security) - 访问模型和安全加固
