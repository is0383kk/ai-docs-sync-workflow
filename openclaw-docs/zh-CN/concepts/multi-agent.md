---
read_when: You want multiple agents with separate workspaces, auth, and sessions in one Gateway process.
sidebarTitle: Multi-agent routing
status: active
summary: 多 Agent 路由：Agent 边界、渠道账户和绑定
title: 多智能体路由
x-i18n:
    generated_at: "2026-07-26T06:40:15Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 46df162388205e46d5a4ea3567c8c8f7016117d2ecafe1184a35b4c95798fd80
    source_path: concepts/multi-agent.md
    workflow: 16
---

在一个 Gateway 网关进程中运行多个相互_隔离_的智能体，每个智能体都有自己的工作区、状态目录（`agentDir`）和由 SQLite 支持的会话历史记录，以及多个渠道账户（例如两个 WhatsApp 号码）。入站消息通过**绑定**路由到正确的智能体。

**智能体**是每个人格角色的完整作用域：工作区文件、身份验证配置文件、模型注册表和会话存储。**绑定**将渠道账户（Slack 工作区、WhatsApp 号码等）映射到其中一个智能体。

## 什么是一个智能体

每个智能体都有自己的：

- **工作区**：文件、`AGENTS.md`/`SOUL.md`/`USER.md`、本地笔记、人格角色规则。
- **状态目录**（`agentDir`）：身份验证配置文件、模型注册表、每智能体配置。
- **会话存储**：`~/.openclaw/agents/<agentId>/agent/openclaw-agent.sqlite` 中的聊天历史记录和路由状态。

身份验证配置文件按智能体隔离，从以下位置读取：

```text
~/.openclaw/agents/<agentId>/agent/auth-profiles.json
```

<Note>
`sessions_history` 是更安全的跨会话回忆路径：它返回有界且经过脱敏的视图，而不是原始对话记录转储。它会移除思考块签名、工具结果载荷详情、`<relevant-memories>` 脚手架、工具调用 XML 标签（`<tool_call>`、`<function_call>` 及其复数形式和降级形式）和 MiniMax 工具调用 XML，然后截断输出并按字节大小限制输出上限。
</Note>

<Warning>
切勿在智能体之间复用 `agentDir`，否则会导致身份验证/会话状态冲突。当辅助智能体的本地 OAuth 凭据已过期或刷新失败时，OpenClaw 会透传读取默认/主智能体中具有相同配置文件 ID 的凭据，并采用其中最新的令牌，而不会将刷新令牌复制到辅助智能体的存储中。如果需要完全独立的 OAuth 账户，请从该智能体登录。如果手动复制凭据，只能复制可移植的静态 `api_key` 或 `token` 配置文件——默认情况下，OAuth 刷新材料不可移植（`copyToAgents` 可显式允许某个配置文件进行移植）。
</Warning>

Skills 从每个智能体工作区以及 `~/.openclaw/skills` 等共享根目录加载，然后根据智能体的有效技能允许列表进行筛选。使用 `agents.defaults.skills` 设置共享基线，使用 `agents.entries.*.skills` 设置每智能体替代项（显式条目会替代默认值，而不是与其合并）。请参阅 [Skills：每智能体与共享 Skills](/zh-CN/tools/skills#per-agent-vs-shared-skills)和 [Skills：智能体允许列表](/zh-CN/tools/skills#agent-allowlists)。

插件拥有的存储遵循该插件的配置；添加第二个智能体
不会自动拆分所有全局插件存储。例如，当人格角色不能共享
已编译的 wiki 知识时，请配置
[Memory Wiki 每智能体保管库](/zh-CN/concepts/multi-agent#per-agent-memory-wiki-vaults)。

<Note>
**工作区注意事项：**每个智能体的工作区是**默认 cwd**，而不是硬性沙箱。相对路径在工作区内解析，但除非启用沙箱隔离，否则绝对路径可以访问主机上的其他位置。请参阅[沙箱隔离](/zh-CN/gateway/sandboxing)。
</Note>

## 路径

| 内容                             | 默认值                                                                                | 覆盖方式                                                                                    |
| -------------------------------- | -------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------- |
| 配置                           | `~/.openclaw/openclaw.json`                                                            | `OPENCLAW_CONFIG_PATH`                                                                      |
| 状态目录                        | `~/.openclaw`                                                                          | `OPENCLAW_STATE_DIR`                                                                        |
| 默认智能体的工作区        | `~/.openclaw/workspace`（设置 `OPENCLAW_PROFILE` 时为 `workspace-<profile>`）      | `agents.entries.*.workspace`，然后是 `agents.defaults.workspace`，或 `OPENCLAW_WORKSPACE_DIR` |
| 其他智能体的工作区          | `<stateDir>/workspace-<agentId>`（设置时为 `<agents.defaults.workspace>/<agentId>`） | `agents.entries.*.workspace`                                                                |
| Agent 目录                        | `~/.openclaw/agents/<agentId>/agent`                                                   | `agents.entries.*.agentDir`                                                                 |
| 会话和对话记录         | `~/.openclaw/agents/<agentId>/agent/openclaw-agent.sqlite`                             | —                                                                                           |
| 旧版/归档会话工件 | `~/.openclaw/agents/<agentId>/sessions`                                                | —                                                                                           |

### 单智能体模式（默认）

如果未进行任何配置，OpenClaw 会运行一个智能体：

- `agentId` 默认为 `main`。
- 会话键为 `agent:main:<mainKey>`（默认 `mainKey` 为 `main`）。
- 工作区默认为 `~/.openclaw/workspace`（当 `OPENCLAW_PROFILE` 设置为 `default` 以外的值时，为 `workspace-<profile>`）。
- 状态默认为 `~/.openclaw/agents/main/agent`。

## 智能体辅助工具

添加新的隔离智能体：

```bash
openclaw agents add work
```

标志：`--workspace <dir>`、`--model <id>`、`--agent-dir <dir>`、`--bind <channel[:accountId]>`（可重复使用）、`--non-interactive`（需要 `--workspace`）。

添加 `bindings` 以路由入站消息（向导会询问是否代为执行），然后验证：

```bash
openclaw agents list --bindings
```

## 快速开始

<Steps>
  <Step title="创建每个智能体的工作区">
    ```bash
    openclaw agents add coding
    openclaw agents add social
    ```

    每个智能体都会获得自己的工作区，其中包含 `SOUL.md`、`AGENTS.md` 和可选的 `USER.md`，并在 `~/.openclaw/agents/<agentId>` 下拥有专用的 `agentDir` 和会话存储。

  </Step>
  <Step title="创建渠道账户">
    在首选渠道上为每个智能体创建一个账户：

    - Discord：每个智能体使用一个机器人，启用 Message Content Intent，并复制每个令牌。
    - Telegram：通过 BotFather 为每个智能体创建一个机器人，并复制每个令牌。
    - WhatsApp：为每个账户关联相应的电话号码。

    ```bash
    openclaw channels login --channel whatsapp --account work
    ```

    请参阅渠道指南：[Discord](/zh-CN/channels/discord)、[Telegram](/zh-CN/channels/telegram)、[WhatsApp](/zh-CN/channels/whatsapp)。

  </Step>
  <Step title="添加智能体、账户和绑定">
    在 `agents.entries` 下添加智能体，在 `channels.<channel>.accounts` 下添加渠道账户，并使用 `bindings` 将它们连接起来（示例如下）。
  </Step>
  <Step title="重启并验证">
    ```bash
    openclaw gateway restart
    openclaw agents list --bindings
    openclaw channels status --probe
    ```
  </Step>
</Steps>

## 多个智能体、多个角色人格

每个已配置的 `agentId` 都是核心智能体状态的独立人格角色边界：

- 每个渠道使用不同账户（按 `accountId`）。
- 不同人格（通过每智能体 `AGENTS.md`/`SOUL.md`）。
- 身份验证和会话彼此分离，仅通过显式功能或插件配置启用跨智能体访问。

这样，多个人就可以共享一个 Gateway 网关，同时保持核心智能体状态彼此分离。

## 每智能体 Memory Wiki 保管库

Memory Wiki 默认使用一个全局保管库。要使支持智能体的
已编译知识与营销智能体的知识分离，请将
`plugins.entries.memory-wiki.config.vault.scope` 设置为 `agent`：

```json5
{
  plugins: {
    entries: {
      "memory-wiki": {
        enabled: true,
        config: {
          vault: {
            scope: "agent",
            path: "~/.openclaw/wiki",
          },
        },
      },
    },
  },
}
```

配置的路径是父目录。OpenClaw 会追加规范化的
智能体 ID，从而生成 `~/.openclaw/wiki/support` 和
`~/.openclaw/wiki/marketing` 等路径。配置多个智能体时，智能体作用域的 CLI 和 Gateway 网关操作
需要显式指定智能体。有关桥接
筛选、迁移和信任边界的详情，请参阅
[Memory Wiki 每智能体保管库](/zh-CN/plugins/memory-wiki#per-agent-vaults)。

## 跨智能体 QMD 记忆搜索

要允许一个智能体搜索另一个智能体的 QMD 会话对话记录，请在 `agents.entries.*.memory.search.qmd.extraCollections` 下添加额外集合。当所有智能体都应共享相同集合时，请使用 `memory.search.qmd.extraCollections`。

```json5
{
  agents: {
    defaults: {
      workspace: "~/workspaces/main",
    },
    entries: {
      main: {
        workspace: "~/workspaces/main",
        memory: {
          search: {
            qmd: {
              extraCollections: [{ path: "notes" }], // 在工作区内解析 -> 名为 "notes-main" 的集合
            },
          },
        },
      },
      family: { workspace: "~/workspaces/family" },
    },
  },
  memory: {
    backend: "qmd",
    search: {
      qmd: {
        extraCollections: [{ path: "~/agents/family/sessions", name: "family-sessions" }],
      },
    },
    qmd: { includeDefaultMemory: false },
  },
}
```

额外集合路径可以在智能体之间共享，但当路径位于智能体工作区之外时，其 `name` 仍需显式指定。工作区内的路径仍按智能体划分作用域，因此每个智能体都保留自己的对话记录搜索集。

## 一个 WhatsApp 号码，多个人（私信拆分）

在**一个** WhatsApp 账户上，通过使用 `peer.kind: "direct"` 匹配发送者 E.164（`+15551234567`），将不同的 WhatsApp 私信路由到不同的智能体。回复仍从同一个 WhatsApp 号码发出——不存在每智能体发送者身份。

<Note>
默认情况下，直接聊天会归并到智能体的主会话键，因此真正的隔离要求每个人使用一个智能体。
</Note>

```json5
{
  agents: {
    list: [
      { id: "alex", workspace: "~/.openclaw/workspace-alex" },
      { id: "mia", workspace: "~/.openclaw/workspace-mia" },
    ],
  },
  bindings: [
    {
      agentId: "alex",
      match: { channel: "whatsapp", peer: { kind: "direct", id: "+15551230001" } },
    },
    {
      agentId: "mia",
      match: { channel: "whatsapp", peer: { kind: "direct", id: "+15551230002" } },
    },
  ],
  channels: {
    whatsapp: {
      dmPolicy: "allowlist",
      allowFrom: ["+15551230001", "+15551230002"],
    },
  },
}
```

私信访问控制（配对/允许列表）按 WhatsApp 账户全局生效，而不是按智能体生效。对于共享群组，请将群组绑定到一个智能体，或使用[广播群组](/zh-CN/channels/broadcast-groups)。

## 路由规则

绑定是确定性的，并以最具体的匹配为准。有关完整的层级顺序（精确对端、父对端、对端通配符、服务器 + 角色、服务器、团队、账户、渠道、默认智能体），请参阅[渠道路由](/zh-CN/channels/channel-routing#routing-rules-how-an-agent-is-chosen)。这里有几项值得特别说明的规则：

- 如果同一层级内有多个绑定匹配，则配置顺序中的第一个绑定获胜。
- 如果绑定设置了多个匹配字段（例如 `peer` + `guildId`），则所有指定字段都必须匹配（`AND` 语义）。
- 省略 `accountId` 的绑定只匹配默认账户，而不是所有账户。使用 `accountId: "*"` 设置渠道范围的回退，或使用 `accountId: "<name>"` 指定一个账户。再次添加同一绑定并显式指定账户 ID，会升级现有的仅渠道绑定，而不是创建重复绑定。

## 多个账户/电话号码

支持多个账户的渠道（例如 WhatsApp）使用 `accountId` 标识每次登录。每个 `accountId` 都会路由到自己的智能体，因此一台服务器可以托管多个电话号码而不会混用会话。

设置 `channels.<channel>.defaultAccount`，以选择省略 `accountId` 时使用的账户。未设置时，如果存在 `default`，OpenClaw 将回退到该账户；否则使用配置的第一个账户 ID（排序后）。

支持多个账户的渠道：`discord`、`feishu`、`googlechat`、`imessage`、`irc`、`line`、`mattermost`、`matrix`、`nextcloud-talk`、`nostr`、`signal`、`slack`、`telegram`、`whatsapp`、`zalo`、`zalouser`。

## 概念

- `agentId`：一个“大脑”（工作区、每个智能体的身份验证、每个智能体的会话存储）。
- `accountId`：一个渠道账户实例（例如 WhatsApp 账户 `personal` 与 `biz`）。
- `binding`：根据 `(channel, accountId, peer)` 将入站消息路由到 `agentId`，并可选择使用服务器/团队 ID。
- 直接聊天会归并到 `agent:<agentId>:<mainKey>`（每个智能体的“主”会话；参见 `session.mainKey`）。

## 平台示例

<AccordionGroup>
  <Accordion title="每个智能体使用一个 Discord Bot">
    每个 Discord Bot 账户都映射到唯一的 `accountId`。将每个账户绑定到一个智能体，并为每个 Bot 分别维护允许列表。

    ```json5
    {
      agents: {
        list: [
          { id: "main", workspace: "~/.openclaw/workspace-main" },
          { id: "coding", workspace: "~/.openclaw/workspace-coding" },
        ],
      },
      bindings: [
        { agentId: "main", match: { channel: "discord", accountId: "default" } },
        { agentId: "coding", match: { channel: "discord", accountId: "coding" } },
      ],
      channels: {
        discord: {
          groupPolicy: "allowlist",
          accounts: {
            default: {
              token: "DISCORD_BOT_TOKEN_MAIN",
              guilds: {
                "123456789012345678": {
                  channels: {
                    "222222222222222222": { allow: true, requireMention: false },
                  },
                },
              },
            },
            coding: {
              token: "DISCORD_BOT_TOKEN_CODING",
              guilds: {
                "123456789012345678": {
                  channels: {
                    "333333333333333333": { allow: true, requireMention: false },
                  },
                },
              },
            },
          },
        },
      },
    }
    ```

    - 邀请每个 Bot 加入服务器，并启用 Message Content Intent。
    - 令牌位于 `channels.discord.accounts.<id>.token` 中（默认账户可以使用 `DISCORD_BOT_TOKEN`）。

  </Accordion>
  <Accordion title="每个智能体使用一个 Telegram Bot">
    ```json5
    {
      agents: {
        list: [
          { id: "main", workspace: "~/.openclaw/workspace-main" },
          { id: "alerts", workspace: "~/.openclaw/workspace-alerts" },
        ],
      },
      bindings: [
        { agentId: "main", match: { channel: "telegram", accountId: "default" } },
        { agentId: "alerts", match: { channel: "telegram", accountId: "alerts" } },
      ],
      channels: {
        telegram: {
          accounts: {
            default: {
              botToken: "123456:ABC...",
              dmPolicy: "pairing",
            },
            alerts: {
              botToken: "987654:XYZ...",
              dmPolicy: "allowlist",
              allowFrom: ["tg:123456789"],
            },
          },
        },
      },
    }
    ```

    - 使用 BotFather 为每个智能体创建一个 Bot，并复制各自的令牌。
    - 令牌位于 `channels.telegram.accounts.<id>.botToken` 中（默认账户可以使用 `TELEGRAM_BOT_TOKEN`）。
    - 如果同一个 Telegram 群组中有多个 Bot，请邀请每个 Bot，并提及应当回答的那个 Bot。
    - 为每个群组 Bot 禁用 BotFather Privacy Mode（`/setprivacy` -> Disable），然后移除并重新添加该 Bot，以便 Telegram 应用此设置。
    - 使用 `channels.telegram.groups` 允许群组，或仅在可信的群组部署中使用 `groupPolicy: "open"`。
    - 将发送者用户 ID 放入 `groupAllowFrom`。群组和超级群组 ID 应放入 `channels.telegram.groups`，而不是 `groupAllowFrom`。
    - 按 `accountId` 绑定，使每个 Bot 都路由到其各自的智能体。

  </Accordion>
  <Accordion title="每个智能体使用一个 WhatsApp 号码">
    启动 Gateway 网关前，先关联每个账户：

    ```bash
    openclaw channels login --channel whatsapp --account personal
    openclaw channels login --channel whatsapp --account biz
    ```

    `~/.openclaw/openclaw.json`（JSON5）：

    ```js
    {
      agents: {
        list: [
          {
            id: "home",
            default: true,
            name: "Home",
            workspace: "~/.openclaw/workspace-home",
            agentDir: "~/.openclaw/agents/home/agent",
          },
          {
            id: "work",
            name: "Work",
            workspace: "~/.openclaw/workspace-work",
            agentDir: "~/.openclaw/agents/work/agent",
          },
        ],
      },

      // 确定性路由：第一个匹配项生效（最具体的规则优先）。
      bindings: [
        { agentId: "home", match: { channel: "whatsapp", accountId: "personal" } },
        { agentId: "work", match: { channel: "whatsapp", accountId: "biz" } },

        // 可选的单个对端覆盖规则（示例：将特定群组发送给工作智能体）。
        {
          agentId: "work",
          match: {
            channel: "whatsapp",
            accountId: "personal",
            peer: { kind: "group", id: "1203630...@g.us" },
          },
        },
      ],

      // 默认关闭：必须显式启用智能体间消息传递并将其加入允许列表。
      tools: {
        agentToAgent: {
          enabled: false,
          allow: ["home", "work"],
        },
      },

      channels: {
        whatsapp: {
          accounts: {
            personal: {
              // 可选覆盖。默认值：~/.openclaw/credentials/whatsapp/personal
              // authDir: "~/.openclaw/credentials/whatsapp/personal",
            },
            biz: {
              // 可选覆盖。默认值：~/.openclaw/credentials/whatsapp/biz
              // authDir: "~/.openclaw/credentials/whatsapp/biz",
            },
          },
        },
      },
    }
    ```

  </Accordion>
</AccordionGroup>

## 常见模式

<Tabs>
  <Tab title="WhatsApp 日常使用 + Telegram 深度工作">
    按渠道拆分：将 WhatsApp 路由到快速的日常智能体，将 Telegram 路由到 Opus 智能体。

    ```json5
    {
      agents: {
        list: [
          {
            id: "chat",
            name: "Everyday",
            workspace: "~/.openclaw/workspace-chat",
            model: "anthropic/claude-sonnet-4-6",
          },
          {
            id: "opus",
            name: "Deep Work",
            workspace: "~/.openclaw/workspace-opus",
            model: "anthropic/claude-opus-4-6",
          },
        ],
      },
      bindings: [
        { agentId: "chat", match: { channel: "whatsapp", accountId: "*" } },
        { agentId: "opus", match: { channel: "telegram", accountId: "*" } },
      ],
    }
    ```

    这些示例使用 `accountId: "*"`，因此以后添加账户时，绑定仍能正常工作。若要将单个私信/群组路由到 Opus，同时让其余消息继续由 chat 处理，请为该对端添加 `match.peer` 绑定——对端匹配始终优先于渠道级规则。

  </Tab>
  <Tab title="同一渠道中将一个对端路由到 Opus">
    让 WhatsApp 继续使用快速智能体，但将一条私信路由到 Opus：

    ```json5
    {
      agents: {
        list: [
          {
            id: "chat",
            name: "Everyday",
            workspace: "~/.openclaw/workspace-chat",
            model: "anthropic/claude-sonnet-4-6",
          },
          {
            id: "opus",
            name: "Deep Work",
            workspace: "~/.openclaw/workspace-opus",
            model: "anthropic/claude-opus-4-6",
          },
        ],
      },
      bindings: [
        {
          agentId: "opus",
          match: { channel: "whatsapp", accountId: "*", peer: { kind: "direct", id: "+15551234567" } },
        },
        { agentId: "chat", match: { channel: "whatsapp", accountId: "*" } },
      ],
    }
    ```

    对端绑定始终优先，因此请将其置于渠道级规则之前。

  </Tab>
  <Tab title="绑定到 WhatsApp 群组的家庭智能体">
    将专用的家庭智能体绑定到单个 WhatsApp 群组，并设置提及门控和更严格的工具策略：

    ```json5
    {
      agents: {
        list: [
          {
            id: "family",
            name: "Family",
            workspace: "~/.openclaw/workspace-family",
            identity: { name: "Family Bot" },
            groupChat: {
              mentionPatterns: ["@family", "@familybot", "@Family Bot"],
            },
            sandbox: {
              mode: "all",
              scope: "agent",
            },
            tools: {
              allow: [
                "exec",
                "read",
                "sessions_list",
                "sessions_history",
                "sessions_send",
                "sessions_spawn",
                "session_status",
              ],
              deny: ["write", "edit", "apply_patch", "browser", "canvas", "nodes", "cron"],
            },
          },
        ],
      },
      bindings: [
        {
          agentId: "family",
          match: {
            channel: "whatsapp",
            peer: { kind: "group", id: "120363999999999999@g.us" },
          },
        },
      ],
    }
    ```

    工具允许/拒绝列表针对的是**工具**，而不是 Skills。如果某个 Skill 需要运行二进制文件，请确保允许 `exec`，并且该二进制文件存在于沙箱中。若需要更严格的门控，请设置 `agents.entries.*.groupChat.mentionPatterns`，并为该渠道保持启用群组允许列表。

  </Tab>
</Tabs>

## 按智能体配置沙箱和工具

每个智能体都可以有自己的沙箱和工具限制：

```js
{
  agents: {
    list: [
      {
        id: "personal",
        workspace: "~/.openclaw/workspace-personal",
        sandbox: {
          mode: "off",  // 个人智能体不使用沙箱
        },
        // 无工具限制——所有工具均可用
      },
      {
        id: "family",
        workspace: "~/.openclaw/workspace-family",
        sandbox: {
          mode: "all",     // 始终使用沙箱隔离
          scope: "agent",  // 每个智能体一个容器
          docker: {
            // 容器创建后的可选一次性设置
            setupCommand: "apt-get update && apt-get install -y git curl",
          },
        },
        tools: {
          allow: ["read"],                    // 仅允许 read 工具
          deny: ["exec", "write", "edit", "apply_patch"],    // 拒绝其他工具
        },
      },
    ],
  },
}
```

<Note>
`setupCommand` 位于 `sandbox.docker` 下，并在创建容器时运行一次。当解析后的作用域为 `"shared"` 时，会忽略每个智能体的 `sandbox.docker.*` 覆盖设置。
</Note>

这样可以实现：

- **安全隔离**：限制不可信智能体可使用的工具。
- **资源控制**：对特定智能体进行沙箱隔离，同时让其他智能体在主机上运行。
- **灵活策略**：为不同智能体设置不同权限。

<Note>
`tools.elevated` 同时具有全局门控（`tools.elevated.enabled`/`allowFrom`）和每个智能体的门控（`agents.entries.*.tools.elevated.enabled`/`allowFrom`）。每个智能体的门控只能进一步限制全局门控——必须两者都允许某个发送者，该发送者才能运行提升权限的命令。对于群组定向，请使用 `agents.entries.*.groupChat.mentionPatterns`，以便将 @提及明确映射到目标智能体。
</Note>

有关详细示例，请参阅[多 Agent 沙盒和工具](/zh-CN/tools/multi-agent-sandbox-tools)。

## 相关内容

- [ACP 智能体](/zh-CN/tools/acp-agents) — 运行外部编码工具框架
- [渠道路由](/zh-CN/channels/channel-routing) — 消息如何路由到智能体
- [在线状态](/zh-CN/concepts/presence) — 智能体的在线状态和可用性
- [会话](/zh-CN/concepts/session) — 会话隔离和路由
- [子智能体](/zh-CN/tools/subagents) — 启动后台智能体运行
