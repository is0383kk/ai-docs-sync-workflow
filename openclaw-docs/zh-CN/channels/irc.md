---
read_when:
    - 你想将 OpenClaw 连接到 IRC 频道或私信
    - 你正在配置 IRC 允许列表、群组策略或提及门控
summary: IRC 插件设置、访问控制和故障排查
title: IRC
x-i18n:
    generated_at: "2026-07-26T06:38:10Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 85c3da80b45d6611872ddbd10b3be4a5742b46e355e8bb554353a478f2a1702f
    source_path: channels/irc.md
    workflow: 16
---

当你希望在传统频道（`#room`）和私信中使用 OpenClaw 时，请使用 IRC。
安装官方 IRC 插件，然后在 `channels.irc` 下进行配置。

## 快速开始

1. 安装插件：

```bash
openclaw plugins install @openclaw/irc
```

2. 至少在 `~/.openclaw/openclaw.json` 中设置主机、昵称和要加入的频道：

```json5
{
  channels: {
    irc: {
      enabled: true,
      host: "irc.example.com",
      port: 6697,
      tls: true,
      nick: "openclaw-bot",
      channels: ["#openclaw"],
    },
  },
}
```

3. 启动或重启 Gateway 网关：

```bash
openclaw gateway run
```

建议使用私有 IRC 服务器进行 Bot 协调。如果有意使用公共 IRC 网络，常见选择包括 Libera.Chat、OFTC 和 Snoonet。避免使用名称可预测的公共频道传输 Bot 或 Swarm 的后端通信流量。

## 入站持久性

OpenClaw 会先将每个已接受的 IRC `PRIVMSG` 写入持久化入口队列，然后再执行常规策略检查和智能体分派。待处理或可重试的消息可在 Gateway 网关重启后保留，并继续按频道或私信对端串行处理。

IRC 不提供可重放的投递 ID，也不会重新发送客户端断开连接期间错过的消息。因此，OpenClaw 会分配一个仅在当前 TCP 连接内保持稳定的本地 ID。该队列会保护从本地接受消息到分派消息之间的窗口；它无法恢复从未到达 OpenClaw 的消息，也无法对服务器跨连接重发的消息进行去重。

## 连接设置

| 键                            | 默认值                        | 说明                                                        |
| ----------------------------- | ----------------------------- | ----------------------------------------------------------- |
| `host`                        | 无（必填）                    | IRC 服务器主机名                                            |
| `port`                        | 使用 TLS 时为 `6697`，明文时为 `6667` | 1-65535                                                     |
| `tls`                         | `true`                        | 仅在有意使用明文连接时设置 `false`               |
| `nick`                        | 无（必填）                    | Bot 昵称                                                    |
| `username`                    | 昵称，否则为 `openclaw`      | IRC 用户名                                                  |
| `realname`                    | `OpenClaw`                    | 真实姓名/GECOS 字段                                         |
| `password` / `passwordFile`   | 无                            | 服务器密码；文件必须是常规文件                              |
| `channels`                    | 无                            | 要加入的频道（`["#openclaw"]`）                          |
| `accounts` / `defaultAccount` | 无                            | 多账户设置；环境变量仅填充默认账户                          |

## 安全默认设置

- IRC 使用 OpenClaw 操作员管理的正向代理路由之外的原始 TCP/TLS 套接字。如果部署要求所有出站流量都经过该正向代理，请设置 `channels.irc.enabled=false`，除非已明确批准 IRC 直接出站。
- `channels.irc.dmPolicy` 默认为 `"pairing"`：未知的私信发送者会收到配对码，你可以使用 `openclaw pairing approve irc <code>` 批准。
- `channels.irc.groupPolicy` 默认为 `"allowlist"`。
- 使用 `groupPolicy="allowlist"` 时，设置 `channels.irc.groups` 以定义允许的频道。
- 除非有意接受明文传输，否则请使用 TLS（`channels.irc.tls=true`）。

## 访问控制

IRC 频道有两个独立的“关卡”：

1. **频道访问权限**（`groupPolicy` + `groups`）：Bot 是否接受来自某个频道的消息。
2. **发送者访问权限**（`groupAllowFrom` / 每频道的 `groups["#channel"].allowFrom`）：允许哪些人在该频道中触发 Bot。

配置键：

- 私信允许列表（私信发送者访问权限）：`channels.irc.allowFrom`
- 群组发送者允许列表（频道发送者访问权限）：`channels.irc.groupAllowFrom`
- 每频道控制（频道、发送者和提及规则）：`channels.irc.groups["#channel"]`，包含 `requireMention`、`allowFrom`、`enabled`、`tools`、`toolsBySender`、`skills` 和 `systemPrompt`
- `channels.irc.groupPolicy="open"` 允许未配置的频道（**默认情况下仍需提及**）

允许列表条目应使用稳定的发送者身份（`nick!user@host`）。
仅当 `channels.irc.dangerouslyAllowNameMatching: true` 时，才会启用可变的纯昵称匹配。

### 常见误区：`allowFrom` 用于私信，而非频道

如果看到类似以下内容的日志：

- `irc: drop group sender alice!ident@host (policy=allowlist)`

……这表示该发送者无权发送**群组/频道**消息。可通过以下任一方式修复：

- 设置 `channels.irc.groupAllowFrom`（全局应用于所有频道），或
- 设置每频道发送者允许列表：`channels.irc.groups["#channel"].allowFrom`

示例（允许 `#openclaw` 中的任何人与 Bot 对话）：

```json5
{
  channels: {
    irc: {
      groupPolicy: "allowlist",
      groups: {
        "#openclaw": { allowFrom: ["*"] },
      },
    },
  },
}
```

## 回复触发（提及）

即使频道已获准访问（通过 `groupPolicy` + `groups`），且发送者也获准访问，OpenClaw 在群组上下文中默认仍会**要求提及**。当消息包含已连接 Bot 的昵称，或与配置的提及模式匹配时，即视为提及了 Bot。

这意味着，除非消息包含与 Bot 匹配的提及模式，否则可能会看到类似 `drop channel … (missing-mention)` 的日志。

要使 Bot 在 IRC 频道中回复时**无需提及**，请为该频道禁用提及限制：

```json5
{
  channels: {
    irc: {
      groupPolicy: "allowlist",
      groups: {
        "#openclaw": {
          requireMention: false,
          allowFrom: ["*"],
        },
      },
    },
  },
}
```

或者，要允许**所有** IRC 频道（不使用每频道允许列表）并且仍能在未提及时回复：

```json5
{
  channels: {
    irc: {
      groupPolicy: "open",
      groups: {
        "*": { requireMention: false, allowFrom: ["*"] },
      },
    },
  },
}
```

## 安全说明（建议用于公共频道）

如果在公共频道中允许 `allowFrom: ["*"]`，任何人都可以向 Bot 发送提示词。
为降低风险，请限制该频道可用的工具。

### 频道中的所有人使用相同工具

```json5
{
  channels: {
    irc: {
      groups: {
        "#openclaw": {
          allowFrom: ["*"],
          tools: {
            deny: ["group:runtime", "group:fs", "gateway", "nodes", "cron", "browser"],
          },
        },
      },
    },
  },
}
```

### 不同发送者使用不同工具（所有者拥有更多权限）

使用 `toolsBySender` 对 `"*"` 应用更严格的策略，并对你的昵称应用更宽松的策略：

```json5
{
  channels: {
    irc: {
      groups: {
        "#openclaw": {
          allowFrom: ["*"],
          toolsBySender: {
            "*": {
              deny: ["group:runtime", "group:fs", "gateway", "nodes", "cron", "browser"],
            },
            "id:alice": {
              deny: ["gateway", "nodes", "cron"],
            },
          },
        },
      },
    },
  },
}
```

说明：

- `toolsBySender` 键应使用显式前缀（`channel:`、`id:`、`e164:`、`username:`、`name:`）。对于 IRC，请将 `id:` 与发送者身份值配合使用：使用 `id:alice` 或 `id:alice!~alice@203.0.113.7` 可实现更严格的匹配。
- 仍接受旧版无前缀键，但仅按 `id:` 匹配，并会发出弃用警告。
- 第一个匹配的发送者策略生效；`"*"` 是通配符回退项。

要详细了解群组访问权限与提及限制及其交互方式，请参阅：[/channels/groups](/zh-CN/channels/groups)。

## NickServ

要在连接后向 NickServ 进行身份识别：

```json5
{
  channels: {
    irc: {
      nickserv: {
        enabled: true,
        service: "NickServ",
        password: "your-nickserv-password",
      },
    },
  },
}
```

只要设置了密码，默认就会执行 NickServ 身份识别（仅当将 `enabled` 设置为 `false` 时才会选择退出）。`service` 默认为 `NickServ`；`passwordFile` 可替代内联的 `password`。

连接时可选择执行一次性注册（`register: true` 需要 `registerEmail`）：

```json5
{
  channels: {
    irc: {
      nickserv: {
        register: true,
        registerEmail: "bot@example.com",
      },
    },
  },
}
```

昵称注册完成后，请禁用 `register`，以避免重复尝试 REGISTER。

## 环境变量

默认账户支持：

- `IRC_HOST`
- `IRC_PORT`
- `IRC_TLS`
- `IRC_NICK`
- `IRC_USERNAME`
- `IRC_REALNAME`
- `IRC_PASSWORD`
- `IRC_CHANNELS`（逗号分隔）
- `IRC_NICKSERV_PASSWORD`
- `IRC_NICKSERV_REGISTER_EMAIL`

无法通过工作区 `.env` 设置 `IRC_HOST`；请参阅[工作区 `.env` 文件](/zh-CN/gateway/security)。

## 故障排查

- 如果 Bot 已连接但从不在频道中回复，请同时检查 `channels.irc.groups` **以及**提及限制是否丢弃了消息（`missing-mention`）。如果希望它无需被提及即可回复，请为该频道设置 `requireMention:false`。
- 如果登录失败，请检查昵称是否可用以及服务器密码是否正确。
- 如果自定义网络上的 TLS 连接失败，请检查主机、端口和证书设置。

## 相关内容

- [频道概览](/zh-CN/channels) — 所有受支持的频道
- [配对](/zh-CN/channels/pairing) — 私信身份验证和配对流程
- [群组](/zh-CN/channels/groups) — 群聊行为和提及限制
- [频道路由](/zh-CN/channels/channel-routing) — 消息的会话路由
- [安全](/zh-CN/gateway/security) — 访问模型和安全加固
