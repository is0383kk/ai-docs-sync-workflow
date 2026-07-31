---
read_when:
    - 开发 Telegram 功能或 Webhooks
summary: Telegram Bot 支持状态、功能和配置
title: Telegram
x-i18n:
    generated_at: "2026-07-26T06:34:16Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: f34067478f4a5a71ed8f18503234b4cfcf573ac740aa887b65d13d0e1f09ba54
    source_path: channels/telegram.md
    workflow: 16
---

已可通过 grammY 用于生产环境中的 Bot 私信和群组。默认传输方式为长轮询；也可选择 webhook 模式。

<CardGroup cols={3}>
  <Card title="配对" icon="link" href="/zh-CN/channels/pairing">
    Telegram 的默认私信策略是配对。
  </Card>
  <Card title="渠道故障排除" icon="wrench" href="/zh-CN/channels/troubleshooting">
    跨渠道诊断和修复操作手册。
  </Card>
  <Card title="Gateway 配置" icon="settings" href="/zh-CN/gateway/configuration">
    完整的渠道配置模式和示例。
  </Card>
</CardGroup>

## 快速设置

<Steps>
  <Step title="在 BotFather 中创建 Bot 令牌">
    两种流程最终都会生成一个需要粘贴到 OpenClaw 中的令牌，请任选其一：

    - **聊天流程**：打开 Telegram，与 **@BotFather** 聊天（确认其用户名准确无误地为 `@BotFather`），运行 `/newbot`，按照提示操作并保存令牌。
    - **网页流程**：打开 [BotFather 的 Web 应用](https://t.me/BotFather?startapp)——它可在所有 Telegram 客户端中运行，包括 [web.telegram.org](https://web.telegram.org)——在界面中创建 Bot，然后复制其令牌。

  </Step>

  <Step title="配置令牌和私信策略">

```json5
{
  channels: {
    telegram: {
      enabled: true,
      botToken: "123:abc",
      dmPolicy: "pairing",
      groups: { "*": { requireMention: true } },
    },
  },
}
```

    环境变量回退：`TELEGRAM_BOT_TOKEN`（仅限默认账号；命名账号必须使用 `botToken` 或 `tokenFile`）。
    Telegram **不**使用 `openclaw channels login telegram`；请在配置或环境变量中设置令牌，然后启动 Gateway 网关。

  </Step>

  <Step title="启动 Gateway 网关并批准第一条私信">

```bash
openclaw gateway
openclaw pairing list telegram
openclaw pairing approve telegram <CODE>
```

    配对码会在 1 小时后过期。

  </Step>

  <Step title="将 Bot 添加到群组">
    将 Bot 添加到群组，然后获取群组访问所需的两个 ID：

    - 你的 Telegram 用户 ID，用于 `allowFrom` / `groupAllowFrom`
    - Telegram 群聊 ID，作为 `channels.telegram.groups` 下的键

    可通过 `openclaw logs --follow`、转发消息查 ID 的 Bot 或 Bot API 的 `getUpdates` 获取群聊 ID。允许该群组后，`/whoami@<bot_username>` 可确认用户 ID 和群组 ID。

    以 `-100` 开头的负数超级群组 ID 是群聊 ID。它们应放在 `channels.telegram.groups` 下，而不是 `groupAllowFrom` 下。

  </Step>
</Steps>

<Note>
令牌解析会区分账号：`tokenFile` 优先于 `botToken`，后者又优先于环境变量；配置始终优先于 `TELEGRAM_BOT_TOKEN`（后者仅解析默认账号）。成功启动后，OpenClaw 会将 Bot 身份缓存最多 24 小时，使重启时跳过一次额外的 `getMe` 调用；更改或移除令牌会清除该缓存。
</Note>

## Telegram 端设置

<AccordionGroup>
  <Accordion title="隐私模式和群组可见性">
    Telegram Bot 默认启用 **Privacy Mode**，这会限制它们能够接收的群组消息。

    要查看所有群组消息，可采用以下任一方式：

    - 通过 `/setprivacy` 禁用隐私模式，或
    - 将 Bot 设为群组管理员。

    切换隐私模式后，请在每个群组中移除并重新添加 Bot，以便 Telegram 应用此更改。

  </Accordion>

  <Accordion title="群组权限">
    管理员状态在 Telegram 群组设置中控制。管理员 Bot 会接收所有群组消息，适合需要始终响应的群组行为。
  </Accordion>

  <Accordion title="实用的 BotFather 开关">

    - `/setjoingroups` — 允许/拒绝添加到群组
    - `/setprivacy` — 群组可见性行为

    如果你更喜欢使用界面而不是聊天命令，也可以在 [BotFather 的 Web 应用](https://t.me/BotFather?startapp)中使用相同的设置。

  </Accordion>
</AccordionGroup>

## 仪表板 Mini App

在与 Bot 的私信中运行 `/dashboard`，即可在 Telegram 内打开 OpenClaw 仪表板。

要求：

- 为已发布的 HTTPS Mini App URL 配置 `gateway.tailscale.mode: "serve"` 或 `"funnel"`。
- 你的数字 Telegram 用户 ID 必须位于所选账号的有效 `allowFrom` 或 `commands.ownerAllowFrom` 中。
- 请使用私信。在群组中，`/dashboard` 会回复 `open this in a DM with the bot`，且不会发送按钮。
- Docker 安装：Serve/Funnel 模式要求 Gateway 网关在 `tailscaled` 旁绑定到环回地址，而使用已发布端口的桥接网络无法满足此要求。请使用 `network_mode: host` 运行 Gateway 网关容器，并将主机的 `tailscaled` 套接字（`/var/run/tailscale`）和 `tailscale` CLI 挂载到容器中。

Mini App 是仅限 Tailscale 的 v1 路径，不支持 Telegram Web iframe。

## 访问控制和激活

### 群组中的 Bot 身份

在群组和论坛主题中，明确提及已配置的 Bot 用户名（例如 `@my_bot`）即表示呼叫所选的 OpenClaw 智能体，即使智能体的人设名称与 Telegram 用户名不同。群组静默策略仍适用于不相关的消息，但 Bot 用户名本身绝不会被视为“其他人”。

<Tabs>
  <Tab title="私信策略">
    `channels.telegram.dmPolicy` 控制私信访问：

    - `pairing`（默认）
    - `allowlist`（要求 `allowFrom` 中至少有一个发送者 ID）
    - `open`（要求 `allowFrom` 包含 `"*"`）
    - `disabled`

    将 `dmPolicy: "open"` 与 `allowFrom: ["*"]` 配合使用，会使任何找到或猜到 Bot 用户名的 Telegram 账号都能向 Bot 发出命令。仅应将其用于工具受到严格限制且有意公开的 Bot；单所有者 Bot 应使用 `allowlist` 并配置数字用户 ID。

    `channels.telegram.allowFrom` 接受数字 Telegram 用户 ID。接受 `telegram:` / `tg:` 前缀，并会将其规范化。
    在多账号配置中，限制严格的顶层 `channels.telegram.allowFrom` 是一道安全边界：除非合并后的有效允许列表仍包含显式通配符，否则账号级 `allowFrom: ["*"]` 不会使该账号公开。
    `dmPolicy: "allowlist"` 与空的 `allowFrom` 组合会阻止所有私信，并且无法通过配置验证。
    设置流程仅询问数字用户 ID。如果你的配置包含旧版设置遗留的 `@username` 允许列表条目，请运行 `openclaw doctor --fix` 将其解析为数字 ID（尽力而为；需要 Telegram Bot 令牌）。
    如果你之前依赖配对存储的允许列表文件，`openclaw doctor --fix` 可以将条目恢复到 `channels.telegram.allowFrom`，以用于允许列表流程（例如 `dmPolicy: "allowlist"` 尚无显式 ID 时）。

    对于单所有者 Bot，建议使用 `dmPolicy: "allowlist"` 并配置显式数字 `allowFrom` ID，而不是依赖以前的配对批准。

    常见误解：批准私信配对并不意味着“此发送者在任何地方都有权限”。配对仅授予私信访问权限。如果尚不存在命令所有者，首次批准配对还会设置 `commands.ownerAllowFrom`，从而为仅限所有者的命令和 Exec 审批指定一个明确的操作员账号。群组发送者的授权仍来自显式配置的允许列表。
    要使用同一身份同时获得私信和群组命令的授权：将你的数字 Telegram 用户 ID 放入 `channels.telegram.allowFrom`；对于仅限所有者的命令，请确保 `commands.ownerAllowFrom` 包含 `telegram:<your user id>`。

    ### 查找你的 Telegram 用户 ID

    更安全的方式（无需第三方 Bot）：向你的 Bot 发送私信，运行 `openclaw logs --follow`，读取 `from.id`。

    官方 Bot API 方法：

```bash
curl "https://api.telegram.org/bot<bot_token>/getUpdates"
```

    第三方方式（隐私性较低）：`@userinfobot` 或 `@getidsbot`。

  </Tab>

  <Tab title="群组策略和允许列表">
    以下两项控制同时生效：

    1. **允许哪些群组**（`channels.telegram.groups`）
       - 无 `groups` 配置，`groupPolicy: "open"`：任何群组都能通过群组 ID 检查
       - 无 `groups` 配置，`groupPolicy: "allowlist"`（默认）：所有群组都会被阻止，直到添加 `groups` 条目（或 `"*"`）
       - 已配置 `groups`：其作用相当于允许列表（显式 ID 或 `"*"`）

    2. **允许群组中的哪些发送者**（`channels.telegram.groupPolicy`）
       - `open` / `allowlist`（默认）/ `disabled`

    `groupAllowFrom` 会筛选群组发送者；如果未设置，Telegram 会回退到 `allowFrom`（而不是配对存储——群组发送者授权绝不会继承私信配对存储的批准，这是自 `2026.2.25` 起设立的安全边界）。
    `groupAllowFrom` 条目应为数字 Telegram 用户 ID（`telegram:` / `tg:` 前缀会被规范化）；非数字条目会被忽略。不要在此处放置群组或超级群组的聊天 ID——负数聊天 ID 应放在 `channels.telegram.groups` 下。
    单所有者 Bot 的实用模式：在 `channels.telegram.allowFrom` 中设置你的用户 ID，不设置 `groupAllowFrom`，并在 `channels.telegram.groups` 下允许目标群组。
    如果配置中完全缺少 `channels.telegram`，除非显式设置 `channels.defaults.groupPolicy`，否则运行时默认采用故障关闭的 `groupPolicy="allowlist"`。

    仅限所有者的群组设置：

```json5
{
  channels: {
    telegram: {
      enabled: true,
      dmPolicy: "pairing",
      allowFrom: ["<YOUR_TELEGRAM_USER_ID>"],
      groupPolicy: "allowlist",
      groups: {
        "<GROUP_CHAT_ID>": {
          requireMention: true,
        },
      },
    },
  },
}
```

    在群组中使用 `@<bot_username> ping` 进行测试。启用 `requireMention: true` 时，普通群组消息不会触发 Bot。

    允许某个特定群组中的任何成员：

```json5
{
  channels: {
    telegram: {
      groups: {
        "-1001234567890": {
          groupPolicy: "open",
          requireMention: false,
        },
      },
    },
  },
}
```

    仅允许某个特定群组中的指定用户：

```json5
{
  channels: {
    telegram: {
      groups: {
        "-1001234567890": {
          requireMention: true,
          allowFrom: ["8734062810", "745123456"],
        },
      },
    },
  },
}
```

    <Warning>
      常见错误：`groupAllowFrom` 不是群组允许列表。

      - 负数 Telegram 群组/超级群组聊天 ID（`-1001234567890`）应放在 `channels.telegram.groups` 下。
      - Telegram 用户 ID（`8734062810`）应放在 `groupAllowFrom` 下，用于限制允许群组中哪些人可以触发 Bot。
      - 仅在希望允许群组中的任何成员与 Bot 对话时使用 `groupAllowFrom: ["*"]`。

    </Warning>

  </Tab>

  <Tab title="提及行为">
    默认情况下，群组回复需要提及 Bot。提及可以来自：

    - 原生 `@botusername` 提及，或
    - `agents.entries.*.groupChat.mentionPatterns` 或 `messages.groupChat.mentionPatterns` 中的提及模式

    会话级开关（仅改变状态，不持久化）：`/activation always`、`/activation mention`。请使用配置实现持久化：

```json5
{
  channels: {
    telegram: {
      groups: {
        "*": { requireMention: false },
      },
    },
  },
}
```

    群组历史上下文始终启用，并受 `historyLimit` 限制。将 `channels.telegram.historyLimit: 0` 设为禁用群组历史窗口。`openclaw doctor --fix` 会移除已停用的 `includeGroupHistoryContext` 键。

    获取群聊 ID：将群组消息转发给 `@userinfobot` / `@getidsbot`，从 `openclaw logs --follow` 中读取 `chat.id`，检查 Bot API 的 `getUpdates`，或在允许该群组后运行 `/whoami@<bot_username>`。

  </Tab>
</Tabs>

## 运行时行为

- Telegram 在 Gateway 网关进程内运行。
- 路由是确定性的：Telegram 入站消息的回复会返回 Telegram（模型不会选择渠道）。
- 入站消息会规范化为共享的渠道信封，其中包含回复元数据、媒体占位符，以及 Gateway 网关已观察到的回复所对应的持久化回复链上下文。
- 群组会话按群组 ID 隔离。论坛话题会附加 `:topic:<threadId>`。
- 私信消息可以携带 `message_thread_id`；OpenClaw 会为回复保留它。仅当 Telegram `getMe` 为该 Bot 报告 `has_topics_enabled: true` 时，私信话题才会拆分为不同会话；否则私信仍使用扁平会话。
- 长轮询使用 grammY runner，并按聊天/线程排序。Runner sink 并发使用 `agents.defaults.maxConcurrent`。
- 多账户启动会限制并发 `getMe` 探测，避免大型 Bot 集群同时扇出所有账户探测。
- 每个 Gateway 网关进程都会保护长轮询，确保同一时间只有一个活跃轮询器可以使用某个 Bot Token。持续出现的 `getUpdates` 409 冲突表明另一个 OpenClaw Gateway 网关、脚本或外部轮询器正在使用同一 Token。
- 如果连续 120 秒没有完成 `getUpdates` 活性检测，轮询看门狗会重启。
- Telegram Bot API 不支持已读回执（`sendReadReceipts` 不适用）。

<Note>
  `channels.telegram.dm.threadReplies` 和 `channels.telegram.direct.<chatId>.threadReplies` 已移除。如果升级后配置中仍包含这些键，请运行 `openclaw doctor --fix`。私信话题路由现在遵循 Telegram `getMe.has_topics_enabled`（由 BotFather 的线程模式控制）：启用话题的 Bot 在 Telegram 发送 `message_thread_id` 时使用线程范围的私信会话；其他私信仍使用扁平会话。
</Note>

## 功能参考

<AccordionGroup>
  <Accordion title="实时流式预览（消息编辑）">
    OpenClaw 会在私聊、群组和话题中实时流式传输部分回复：先发送一条预览消息，然后反复执行 `editMessageText`，最后在原消息中完成回复。

    - `channels.telegram.streaming` 为 `off | partial | block | progress`（默认值：`partial`）
    - 简短的初始回答预览会进行防抖；如果运行仍处于活跃状态，则会在有限延迟后将其实际发送
    - `progress` 会为工具进度保留一条可编辑的状态草稿；当回答活动先于工具进度到达时，它会显示稳定的状态标签；完成时清除该草稿，并将最终回答作为普通消息发送
    - `streaming.preview.toolProgress` 控制工具/进度更新是否复用同一条经过编辑的预览消息（默认值：预览流式传输处于活跃状态时为 `true`）
    - `streaming.preview.commandText` 控制这些行中的命令/Exec 详情：`raw`（默认）或 `status`（仅显示工具标签）
    - `streaming.progress.commentary`（默认值：`false`）用于选择在临时进度草稿中包含助手的评论/前置说明文本
    - 系统会检测旧版 `channels.telegram.streamMode`、布尔型 `streaming` 值以及已停用的原生草稿预览键；请运行 `openclaw doctor --fix` 进行迁移

    工具进度行是在工具运行时显示的简短状态更新（命令执行、文件读取、计划更新、补丁摘要，以及 app-server 模式下的 Codex 前置说明/评论）。Telegram 默认启用这些更新（与 `v2026.4.22`+ 版本中已发布的行为一致）。

    保留回答预览编辑，但隐藏工具进度行：

    ```json
    {
      "channels": {
        "telegram": {
          "streaming": {
            "mode": "partial",
            "preview": { "toolProgress": false }
          }
        }
      }
    }
    ```

    保持工具进度可见，但隐藏命令/Exec 文本：

    ```json
    {
      "channels": {
        "telegram": {
          "streaming": {
            "mode": "partial",
            "preview": { "commandText": "status" }
          }
        }
      }
    }
    ```

    `progress` 模式会显示工具进度，但不会通过编辑该消息来写入最终回答。请将命令文本策略置于 `streaming.progress` 下：

    ```json
    {
      "channels": {
        "telegram": {
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

    `streaming.mode: "off"` 会禁用预览编辑，并抑制通用工具/进度消息，而不是将其作为独立状态消息发送；审批提示、媒体和错误仍通过正常的最终交付路径发送。`streaming.preview.toolProgress: false` 仅保留回答预览编辑。

    <Note>
      选定文本引用回复属于例外情况。当 `replyToMode` 为 `first`、`all` 或 `batched`，且入站消息包含选定的引用文本时，OpenClaw 会通过 Telegram 的原生引用回复路径发送最终回答，而不是编辑回答预览，因此 `streaming.preview.toolProgress` 无法在该轮显示状态行。不含选定引用文本的当前消息回复仍会进行流式传输。如果工具进度可见性比原生引用回复更重要，请设置 `replyToMode: "off"`；或者设置 `streaming.preview.toolProgress: false` 以接受这一取舍。
    </Note>

    对于纯文本回复：短预览会在原消息中完成最终编辑；拆分为多条消息的长篇最终回答会复用预览作为第一个分块，然后仅发送剩余内容；进度模式的最终回答会清除状态草稿并使用正常的最终交付路径；如果在确认完成前最终编辑失败，OpenClaw 会回退到正常的最终交付路径，并清理过期预览。对于复杂回复（媒体载荷），OpenClaw 始终回退到正常的最终交付路径并清理预览。

    预览流式传输和分块流式传输互斥——显式启用分块流式传输时，OpenClaw 会跳过预览流，以避免重复流式传输。

    推理：`/reasoning stream` 会在生成过程中将推理内容流式传输到实时预览中，然后在最终交付后删除推理预览（使用 `/reasoning on` 可使其保持可见）。最终回答中不会包含推理文本。

  </Accordion>

  <Accordion title="富消息格式">
    默认情况下，出站文本使用标准 Telegram HTML 消息，可在当前客户端中正常阅读：粗体、斜体、链接、代码、剧透、引用——而不是仅限 Bot API 10.2 富消息的块（原生表格、详情、富媒体、公式）。

    选择启用 Bot API 10.2 富消息：

```json5
{
  channels: {
    telegram: {
      richMessages: true,
    },
  },
}
```

    启用后：系统会告知智能体此 Bot/账户可使用富消息（以及受支持的 Markdown + HTML 岛式编写约定）；Markdown 文本会通过 OpenClaw 的 Markdown IR 渲染为带类型的 Bot API 10.2 富消息块（标题、表格、详情、检查清单、富媒体、公式、地图、拼贴）；媒体说明仍使用 Telegram HTML 说明（富消息不会取代说明，并且说明上限为 1024 个字符）。

    这样可以避免模型文本接触 Telegram 的富 Markdown 标记，因此像 `$400-600K` 这样的货币不会被解析为数学表达式。较长的富文本会根据 Telegram 的限制自动拆分。超过 20 列限制的表格会回退为代码块。

    默认值：关闭，以确保客户端兼容性——一些当前的 Desktop、Web、Android 和第三方客户端会将已接受的富消息渲染为不受支持。除非该 Bot 使用的每个客户端都能渲染富消息，否则请保持关闭。`/status` 会显示当前会话的富消息处于开启还是关闭状态。

    链接预览默认开启。`channels.telegram.linkPreview: false` 会禁用富文本的自动实体检测。

  </Accordion>

  <Accordion title="原生命令和自定义命令">
    Telegram 的命令菜单会在启动时通过 `setMyCommands` 注册。`commands.native: "auto"` 会为 Telegram 启用原生命令。

    添加自定义命令菜单项：

```json5
{
  channels: {
    telegram: {
      customCommands: [
        { command: "backup", description: "Git 备份" },
        { command: "generate", description: "创建图像" },
      ],
    },
  },
}
```

    规则：名称会进行规范化（移除开头的 `/`，转换为小写）；有效模式为 `a-z`、`0-9`、`_`，长度为 1-32；自定义命令不能覆盖原生命令；冲突项/重复项会被跳过并记录到日志中。

    自定义命令只是菜单项——它们不会自动实现行为。即使 Telegram 菜单中未显示插件/技能命令，手动输入时仍可生效。如果禁用原生命令，内置命令会被移除；如已配置，自定义命令/插件命令仍可注册。

    常见设置失败：

    - 在尝试精简后，`setMyCommands failed` 仍出现 `BOT_COMMANDS_TOO_MUCH`，表示菜单仍然超出限制；请减少插件/技能/自定义命令，或禁用 `channels.telegram.commands.native`。
    - 当直接使用 Bot API curl 命令可以正常工作，但 `deleteWebhook`、`deleteMyCommands` 或 `setMyCommands` 因 `404: Not Found` 失败时，通常表示 `channels.telegram.apiRoot` 被设置为完整的 `/bot<TOKEN>` 端点。`apiRoot` 必须仅为 Bot API 根地址；`openclaw doctor --fix` 会移除意外添加的末尾 `/bot<TOKEN>`。
    - `getMe returned 401` 表示 Telegram 拒绝了所配置的 Bot Token。请使用当前 BotFather Token 更新 `botToken`、`tokenFile` 或 `TELEGRAM_BOT_TOKEN`（默认账户）；OpenClaw 会在轮询前停止，因此不会将其报告为 Webhook 清理失败。
    - `setMyCommands failed` 伴随网络/获取错误时，通常表示到 `api.telegram.org` 的出站 DNS/HTTPS 连接被阻止。

    ### 设备配对命令（`device-pair` 插件）

    安装后：

    1. `/pair` 会生成设置代码
    2. 将代码粘贴到 iOS 应用中
    3. `/pair pending` 会列出待处理请求（包括角色/权限范围）
    4. 批准：`/pair approve <requestId>`、`/pair approve`（仅有一个待处理请求时）或 `/pair approve latest`

    如果设备使用已更改的身份验证详情（角色、权限范围、公钥）重试，先前的待处理请求将被新的 `requestId` 取代；批准前请重新运行 `/pair pending`。

    更多详情：[配对](/zh-CN/channels/pairing#pair-via-telegram)。

  </Accordion>

  <Accordion title="内联按钮">
    配置内联键盘范围：

```json5
{
  channels: {
    telegram: {
      capabilities: {
        inlineButtons: "allowlist",
      },
    },
  },
}
```

    按账户覆盖：

```json5
{
  channels: {
    telegram: {
      accounts: {
        main: {
          capabilities: {
            inlineButtons: "allowlist",
          },
        },
      },
    },
  },
}
```

    范围：`off`、`dm`、`group`、`all`、`allowlist`（默认）。旧版 `capabilities: ["inlineButtons"]` 映射到 `"all"`。

    消息操作示例：

```json5
{
  action: "send",
  channel: "telegram",
  to: "123456789",
  message: "选择一个选项：",
  buttons: [
    [
      { text: "是", callback_data: "yes" },
      { text: "否", callback_data: "no" },
    ],
    [{ text: "取消", callback_data: "cancel" }],
  ],
}
```

    Mini App 按钮示例：

```json5
{
  action: "send",
  channel: "telegram",
  to: "123456789",
  message: "打开应用：",
  presentation: {
    blocks: [
      {
        type: "buttons",
        buttons: [{ label: "启动", web_app: { url: "https://example.com/app" } }],
      },
    ],
  },
}
```

    `web_app` 按钮仅适用于用户与 Bot 之间的私聊。

    未被已注册插件的交互处理程序认领的回调点击会作为文本传递给智能体：`callback_data: <value>`。

  </Accordion>

  <Accordion title="供智能体和自动化使用的 Telegram 消息操作">
    操作：

    - `sendMessage`（`to`、`content`、可选的 `mediaUrl`、`replyToMessageId`、`messageThreadId`）
    - `react`（`chatId`、`messageId`、`emoji`）
    - `deleteMessage`（`chatId`、`messageId`）
    - `editMessage`（`chatId`、`messageId`、`content` 或 `caption`、可选的 `presentation` 内联按钮；仅编辑按钮时会更新回复标记）
    - `createForumTopic`（`chatId`、`name`、可选的 `iconColor`、`iconCustomEmojiId`）

    易用别名：`send`、`react`、`delete`、`edit`、`sticker`、`sticker-search`、`topic-create`。

    功能开关：`channels.telegram.actions.sendMessage`、`deleteMessage`、`reactions`、`sticker`（默认：禁用）。`edit`、`createForumTopic` 和 `editForumTopic` 默认启用，没有专用开关。
    运行时发送使用启动或重新加载时的活动配置/密钥快照，因此操作路径不会在每次发送时重新解析 `SecretRef` 值。

    移除表情回应的语义：[/tools/reactions](/zh-CN/tools/reactions)。

  </Accordion>

  <Accordion title="回复线程标签">
    生成输出中的显式回复线程标签：

    - `[[reply_to_current]]` — 回复触发消息
    - `[[reply_to:<id>]]` — 回复指定消息 ID

    `channels.telegram.replyToMode`：`off`（默认）、`first`、`all`。

    启用回复线程且原始文本/说明文字可用时，OpenClaw 会自动添加原生引用摘录。Telegram 将原生引用文本限制为 1024 个 UTF-16 代码单元；较长消息会从开头开始引用，如果 Telegram 拒绝该引用，则回退为普通回复。

    `off` 仅禁用隐式回复线程；显式 `[[reply_to_*]]` 标签仍会生效。

  </Accordion>

  <Accordion title="论坛话题和线程行为">
    论坛超级群组：话题会话键会追加 `:topic:<threadId>`；回复和输入状态以话题线程为目标；话题配置路径为 `channels.telegram.groups.<chatId>.topics.<threadId>`。

    常规话题（`threadId=1`）是一种特殊情况：发送消息时省略 `message_thread_id`（Telegram 会拒绝带有 `sendMessage(...thread_id=1)` 的消息并返回“找不到线程”），但输入状态操作仍包含 `message_thread_id`（根据实际观察，这是显示输入指示器所必需的）。

    除非被覆盖，否则话题条目会继承群组设置（`requireMention`、`allowFrom`、`skills`、`systemPrompt`、`enabled`、`groupPolicy`）。`agentId` 仅适用于话题，不会继承群组默认值。`topics."*"` 为该群组中的每个话题设置默认值；精确的话题 ID 仍优先于 `"*"`。

    **按话题路由智能体**：每个话题都可以通过话题配置中的 `agentId` 路由到不同的智能体，使其拥有独立的工作区、记忆和会话：

    ```json5
    {
      channels: {
        telegram: {
          groups: {
            "-1001234567890": {
              topics: {
                "1": { agentId: "main" },      // 常规话题 -> 主智能体
                "3": { agentId: "zu" },        // 开发话题 -> zu 智能体
                "5": { agentId: "coder" }      // 代码审查 -> coder 智能体
              }
            }
          }
        }
      }
    }
    ```

    随后，每个话题都有自己的会话键，例如 `agent:zu:telegram:group:-1001234567890:topic:3`。

    **持久 ACP 话题绑定**：论坛话题可以通过顶层类型化绑定固定 ACP harness 会话（`bindings[]`，包含 `type: "acp"`、`match.channel: "telegram"`、`peer.kind: "group"`，以及类似 `-1001234567890:topic:42` 的话题限定 ID）。目前范围仅限群组/超级群组中的论坛话题。参阅 [ACP 智能体](/zh-CN/tools/acp-agents)。

    **从聊天生成线程绑定的 ACP 会话**：`/acp spawn <agent> --thread here|auto` 将当前话题绑定到新的 ACP 会话；后续消息会直接路由到该会话，OpenClaw 还会在话题中固定生成确认消息。由 `session.threadBindings.spawnSessions` 控制（默认：`true`）。

    模板上下文会公开 `MessageThreadId` 和 `IsForum`。带有 `message_thread_id` 的私信聊天会保留回复元数据，但仅当 Telegram `getMe` 报告 `has_topics_enabled: true` 时，才会使用线程感知会话键。
    已弃用的 `dm.threadReplies` 和 `direct.*.threadReplies` 覆盖项已移除；BotFather 线程模式是唯一事实来源。运行 `openclaw doctor --fix` 以移除过时的配置键。

  </Accordion>

  <Accordion title="音频、视频和贴纸">
    ### 音频消息

    Telegram 会区分语音留言和音频文件。默认使用音频文件行为；在智能体回复中添加 `[[audio_as_voice]]` 标签可强制以语音留言形式发送。入站语音留言的转写文本在智能体上下文中会被标记为机器生成且不可信的文本，但提及检测仍会使用原始转写文本，因此受提及条件限制的语音消息仍可正常工作。

```json5
{
  action: "send",
  channel: "telegram",
  to: "123456789",
  media: "https://example.com/voice.ogg",
  asVoice: true,
}
```

    ### 视频消息

    Telegram 会区分视频文件和视频留言。视频留言不支持说明文字；提供的消息文本会单独发送。

```json5
{
  action: "send",
  channel: "telegram",
  to: "123456789",
  media: "https://example.com/video.mp4",
  asVideoNote: true,
}
```

    ### 位置和地点

    使用现有的 `send` 操作，并提供一个独立的 `location` 对象。坐标会发送原生位置图钉；同时添加 `name` 和 `address` 会发送原生地点卡片。位置发送不能与消息文本或媒体组合使用。

```json5
{
  action: "send",
  channel: "telegram",
  to: "123456789",
  location: {
    latitude: 48.858844,
    longitude: 2.294351,
    accuracy: 12,
    name: "埃菲尔铁塔",
    address: "巴黎战神广场",
  },
}
```

    ### 贴纸

    入站：会下载并处理静态 WEBP（占位符 `<media:sticker>`）；会跳过动画 TGS 和视频 WEBM。

    贴纸上下文字段：`Sticker.emoji`、`Sticker.setName`、`Sticker.fileId`、`Sticker.fileUniqueId`、`Sticker.cachedDescription`。描述会缓存在 OpenClaw SQLite 插件状态中，以减少重复的视觉调用。

    启用贴纸操作：

```json5
{
  channels: {
    telegram: {
      actions: {
        sticker: true,
      },
    },
  },
}
```

    发送：

```json5
{
  action: "sticker",
  channel: "telegram",
  to: "123456789",
  fileId: "CAACAgIAAxkBAAI...",
}
```

    搜索已缓存的贴纸：

```json5
{
  action: "sticker-search",
  channel: "telegram",
  query: "挥手的猫",
  limit: 5,
}
```

  </Accordion>

  <Accordion title="表情回应通知">
    Telegram 表情回应以 `message_reaction` 更新的形式到达，与消息负载分开。启用后，OpenClaw 会将类似 `Telegram reaction added: 👍 by Alice (@alice) on msg 42` 的系统事件加入队列。

    - `channels.telegram.reactionNotifications`：`off | own | all`（默认：`own`）
    - `channels.telegram.reactionLevel`：`off | ack | minimal | extensive`（默认：`minimal`）

    `own` 表示仅处理用户对机器人所发消息的表情回应（通过已发送消息缓存尽力实现）。表情回应事件仍遵循 Telegram 访问控制（`dmPolicy`、`allowFrom`、`groupPolicy`、`groupAllowFrom`）；未经授权的发送者会被丢弃。

    Telegram 不会在表情回应更新中提供线程 ID：非论坛群组路由到群聊会话；论坛群组路由到常规话题会话（`:topic:1`），而不是确切的原始话题。

    用于轮询/webhook 的 `allowed_updates` 会自动包含 `message_reaction`。

  </Accordion>

  <Accordion title="确认表情回应">
    `ackReaction` 会在 OpenClaw 处理入站消息时发送确认表情符号。`messages.ackReactionScope` 决定发送的*时机*。

    **表情符号解析顺序：**

    - `channels.telegram.accounts.<accountId>.ackReaction`
    - `channels.telegram.ackReaction`
    - `messages.ackReaction`
    - 智能体身份表情符号回退值（`agents.entries.*.identity.emoji`，否则为“👀”）

    Telegram 要求使用 Unicode 表情符号（例如“👀”）；使用 `""` 可对某个渠道或账户禁用该表情回应。

    **范围（`messages.ackReactionScope`，默认为 `"group-mentions"`；目前没有 Telegram 账户或 Telegram 渠道级覆盖项）：**

    `all`（私信 + 群组，包括环境房间事件）、`direct`（仅私信）、`group-all`（除环境房间事件外的每条群组消息，不包括私信）、`group-mentions`（在群组中提及机器人时；**不包括私信** — 默认）、`off` / `none`（禁用）。

    <Note>
    默认范围（`group-mentions`）不会在私信或环境房间事件中触发确认表情回应。私信请使用 `direct` 或 `all`；只有 `all` 会确认环境房间事件。此值在 Telegram provider 启动时读取，因此更改后需要重启 Gateway 网关才能生效。
    </Note>

  </Accordion>

  <Accordion title="通过 Telegram 事件和命令写入配置">
    默认启用渠道配置写入（`configWrites !== false`）。由 Telegram 触发的写入包括群组迁移事件（`migrate_to_chat_id`，更新 `channels.telegram.groups`）以及 `/config set` / `/config unset`（需要启用命令）。

    禁用：

```json5
{
  channels: {
    telegram: {
      configWrites: false,
    },
  },
}
```

  </Accordion>

  <Accordion title="长轮询与 webhook 对比">
    默认使用长轮询。要使用 webhook 模式，请设置 `channels.telegram.webhookUrl` 和 `channels.telegram.webhookSecret`；可选项包括 `webhookPath`（默认 `/telegram-webhook`）、`webhookHost`（默认 `127.0.0.1`）、`webhookPort`（默认 `8787`）、`webhookCertPath`（用于直接 IP 或无域名设置的自签名证书 PEM）。

    在长轮询模式下，OpenClaw 仅会在更新成功分派后持久化其重启水位；处理程序失败时，该更新在同一进程中仍可重试，而不会被标记为已完成。

    本地监听器默认绑定到 `127.0.0.1:8787`。对于公共入口，请在本地端口前配置反向代理，或有意设置 `webhookHost: "0.0.0.0"`。

    Webhook 模式会验证请求防护、Telegram 密钥令牌和 JSON 正文，然后将更新提交到持久入口队列，再返回空的 `200`。成功持久接管时会包含 `x-openclaw-delivery-accepted: durable`；健康、路由、身份验证、验证和存储错误响应会省略此标头。反向代理和主机控制器可以要求此标头，以区分 OpenClaw 接管与通用的空 `200`，而无需根据响应时机推断是否已接受。

    完成持久写入后，OpenClaw 会通过核心渠道入口排空流程认领并处理更新（按聊天/话题划分通道，在轮次接管时完成，接管前停滞超时）。耗时较长的智能体轮次不会阻塞 Telegram 的投递 ACK。

  </Accordion>

  <Accordion title="限制和 CLI 目标">
    - `channels.telegram.textChunkLimit` 默认值为 4000；`streaming.chunkMode="newline"` 在按长度拆分前优先按段落边界（空行）拆分。
    - `channels.telegram.mediaMaxMb`（默认值为 100）限制入站和出站媒体的大小。
    - 群组上下文历史记录使用 `channels.telegram.historyLimit` 或 `messages.groupChat.historyLimit`（默认值为 50）；`0` 可将其禁用。
    - 当 Gateway 网关已观察到父消息时，回复/引用/转发的补充上下文会被规范化到一个选定的对话上下文窗口中；已观察消息的缓存位于 OpenClaw SQLite 插件状态中，`openclaw doctor --fix` 用于导入旧版边车文件。Telegram 每次更新仅包含一个浅层 `reply_to_message`，因此早于缓存的链只能使用该载荷。
    - Telegram 允许列表主要用于限制谁可以触发智能体，而不是完整的补充上下文脱敏边界。
    - 私信历史记录：`channels.telegram.dmHistoryLimit`、`channels.telegram.dms["<user_id>"].historyLimit`。

    CLI 和消息工具的发送目标接受数字聊天 ID、用户名或论坛主题目标：

```bash
openclaw message send --channel telegram --target 123456789 --message "hi"
openclaw message send --channel telegram --target @name --message "hi"
openclaw message send --channel telegram --target -1001234567890:topic:42 --message "hi topic"
```

    投票使用 `openclaw message poll`，并支持论坛主题：

```bash
openclaw message poll --channel telegram --target 123456789 \
  --poll-question "Ship it?" --poll-option "Yes" --poll-option "No"
openclaw message poll --channel telegram --target -1001234567890:topic:42 \
  --poll-question "Pick a time" --poll-option "10am" --poll-option "2pm" \
  --poll-duration-seconds 300 --poll-public
```

    仅限 Telegram 的投票标志：`--poll-duration-seconds`（5-600）、`--poll-anonymous`、`--poll-public`、`--thread-id`（或 `:topic:` 目标）。`--poll-option` 重复 2-12 次（Telegram 的选项上限）。

    Telegram 发送还支持：使用带 `buttons` 块的 `--presentation` 创建内联键盘（当 `channels.telegram.capabilities.inlineButtons` 允许时）；使用 `--pin` 或 `--delivery '{"pin":true}'` 在机器人可在该聊天中置顶消息时请求置顶投递；使用 `--force-document` 将出站图片、GIF 和视频作为文档发送，而不是采用压缩图片、动画或视频上传方式。

    操作限制：`channels.telegram.actions.sendMessage=false` 禁用所有出站消息，包括投票；`channels.telegram.actions.poll=false` 禁用创建投票，但仍允许常规发送。

  </Accordion>

  <Accordion title="Telegram 中的 Exec 审批">
    Telegram 支持在审批者私信中进行 Exec 审批，也可以选择在发起请求的聊天或主题中发布提示。审批者必须使用数字 Telegram 用户 ID。

    - `channels.telegram.execApprovals.enabled`（至少有一个审批者可解析时，`"auto"` 会启用）
    - `channels.telegram.execApprovals.approvers`（回退到 `commands.ownerAllowFrom` 中的数字所有者 ID）
    - `channels.telegram.execApprovals.target`：`dm`（默认）| `channel` | `both`
    - `agentFilter`、`sessionFilter`

    `channels.telegram.allowFrom`、`groupAllowFrom` 和 `defaultTo` 控制谁可以与机器人交互，以及机器人将常规回复发送到哪里——它们不会使某人成为 Exec 审批者。如果尚不存在命令所有者，首次获批的私信配对会初始化 `commands.ownerAllowFrom`，因此单所有者设置无需在 `execApprovals.approvers` 下重复填写 ID 即可工作。

    渠道投递会在聊天中显示命令文本；仅在受信任的群组/主题中启用 `channel` 或 `both`。当提示发送到论坛主题时，OpenClaw 会为审批提示和后续消息保留该主题。Exec 审批默认在 30 分钟后过期。

    内联审批按钮还要求 `channels.telegram.capabilities.inlineButtons` 允许目标界面（`dm`、`group` 或 `all`）。带有 `plugin:` 前缀的审批 ID 通过插件审批解析；其他 ID 会先通过 Exec 审批解析。

    请参阅 [Exec 审批](/zh-CN/tools/exec-approvals)。

  </Accordion>
</AccordionGroup>

## 错误回复控制

当智能体遇到投递或提供商错误时，错误策略控制错误消息是否会发送到 Telegram 聊天：

| 键                             | 值                     | 默认值  | 说明                                                                                                                                                                |
| ------------------------------- | -------------------------- | -------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `channels.telegram.errorPolicy` | `always`、`once`、`silent` | `always` | `always` 会将每条错误消息发送到聊天。`once` 会在每个内置冷却时间窗口内，将每条不同的错误消息发送一次。`silent` 绝不会将错误消息发送到聊天。 |

支持按账户、群组和主题进行覆盖（继承方式与其他 Telegram 配置键相同）。

```json5
{
  channels: {
    telegram: {
      errorPolicy: "always",
      groups: {
        "-1001234567890": {
          errorPolicy: "silent", // 禁止在此群组中显示错误
        },
      },
    },
  },
}
```

## 故障排查

<AccordionGroup>
  <Accordion title="机器人不响应未提及它的群组消息">

    - 如果 `requireMention=false`，Telegram 隐私模式必须允许完整可见性：BotFather `/setprivacy` -> Disable，然后从群组中移除机器人并重新添加。
    - 当配置预期接收未提及机器人的群组消息时，`openclaw channels status` 会发出警告。
    - `openclaw channels status --probe` 会检查显式数字群组 ID；无法探测通配符 `"*"` 的成员资格。
    - 快速会话测试：`/activation always`。

  </Accordion>

  <Accordion title="机器人完全看不到群组消息">

    - 当 `channels.telegram.groups` 存在时，必须列出该群组（或包含 `"*"`）。
    - 验证机器人是否为群组成员。
    - 查看 `openclaw logs --follow` 以了解跳过原因。

  </Accordion>

  <Accordion title="命令只能部分工作或完全无效">

    - 授权你的发送者身份（配对和/或数字 `allowFrom`）；即使群组策略为 `open`，命令授权仍然适用。
    - `setMyCommands failed` 与 `BOT_COMMANDS_TOO_MUCH` 同时出现表示原生菜单中的条目过多；请减少插件/技能/自定义命令，或禁用原生菜单。
    - `deleteMyCommands` / `setMyCommands` 启动调用和 `sendChatAction` 输入状态调用均有时限，并会在请求超时时通过 Telegram 的传输回退机制重试一次。持续出现网络/提取错误通常表示无法通过 DNS/HTTPS 访问 `api.telegram.org`。

  </Accordion>

  <Accordion title="启动报告令牌未经授权">

    - `getMe returned 401` 表示已配置的机器人令牌未通过 Telegram 身份验证。请在 BotFather 中重新复制或生成令牌，然后更新 `channels.telegram.botToken`、`tokenFile`、`accounts.<id>.botToken` 或 `TELEGRAM_BOT_TOKEN`（默认账户）。
    - 启动期间出现 `deleteWebhook 401 Unauthorized` 也表示身份验证失败；将其视为“没有 Webhook”只会把同一错误令牌导致的失败推迟到后续 API 调用。

  </Accordion>

  <Accordion title="轮询或网络不稳定">

    - 使用自定义 fetch/代理时，如果 `AbortSignal` 类型不匹配，Node 22+ 可能会触发立即中止行为。
    - 某些主机会优先将 `api.telegram.org` 解析为 IPv6；IPv6 出站连接异常会导致间歇性 API 失败。
    - 日志中的 `TypeError: fetch failed` 或 `Network request for 'getUpdates' failed!` 会被视为可恢复的网络错误并重试。
    - 在轮询启动期间，OpenClaw 会为 grammY 复用成功的启动 `getMe` 探测，因此运行器无需在首次 `getUpdates` 前再次调用 `getMe`。
    - 如果轮询启动期间 `deleteWebhook` 因暂时性网络错误而失败，OpenClaw 会直接进入长轮询，而不是再次进行轮询前的控制平面调用。仍处于活动状态的 Webhook 随后会表现为 `getUpdates` 冲突；OpenClaw 会重建传输并重试清理 Webhook。
    - 日志中的 `Polling stall detected` 表示默认情况下，长轮询活跃性连续 120 秒未完成后，OpenClaw 会重启轮询并重建传输。
    - 当正在运行的轮询账户在启动宽限期后尚未完成 `getUpdates`、正在运行的 Webhook 账户在启动宽限期后尚未完成 `setWebhook`，或上次成功的轮询传输活动已过期时，`openclaw channels status --probe` 和 `openclaw doctor` 会发出警告。
    - Telegram 的 Bot API 传输遵循进程代理环境变量：`HTTP_PROXY`、`HTTPS_PROXY`、`ALL_PROXY` 及其小写变体。`NO_PROXY` / `no_proxy` 仍可绕过 `api.telegram.org`。
    - 如果服务环境设置了 `OPENCLAW_PROXY_URL`，且不存在标准代理环境变量，Telegram 也会将该 URL 用于 Bot API 传输。
    - 在直接出站连接/TLS 不稳定的 VPS 主机上，通过代理路由 Telegram API 调用：

```yaml
channels:
  telegram:
    proxy: socks5://<user>:<password>@proxy-host:1080
```

    - Node 22+ 默认使用 `autoSelectFamily=true`（WSL2 除外）。Telegram DNS 结果顺序依次遵循 `OPENCLAW_TELEGRAM_DNS_RESULT_ORDER`、`channels.telegram.network.dnsResultOrder` 和进程默认值（例如 `NODE_OPTIONS=--dns-result-order=ipv4first`）；如果均不适用，在 Node 22+ 上会回退到 `ipv4first`。
    - 在 WSL2 上，或仅使用 IPv4 的效果更好时，强制选择地址族：

```yaml
channels:
  telegram:
    network:
      autoSelectFamily: false
```

    - 默认情况下，Telegram 媒体下载已允许 RFC 2544 基准测试地址范围的响应（`198.18.0.0/15`）。如果在媒体下载期间，受信任的 fake-IP 或透明代理将 `api.telegram.org` 重写为其他私有/内部/特殊用途地址，请选择启用仅限 Telegram 的绕过机制：

```yaml
channels:
  telegram:
    network:
      dangerouslyAllowPrivateNetwork: true
```

    - 还可以在 `channels.telegram.accounts.<accountId>.network.dangerouslyAllowPrivateNetwork` 中按账户选择启用相同选项。
    - 如果你的代理将 Telegram 媒体主机解析到 `198.18.x.x`，请先保持危险标志关闭——该范围默认已允许。

    <Warning>
      `channels.telegram.network.dangerouslyAllowPrivateNetwork` 会削弱 Telegram 媒体 SSRF 防护。仅可用于受信任且由操作员控制的代理环境（Clash、Mihomo、Surge fake-IP 路由），这些环境会合成 RFC 2544 基准测试范围之外的私有或特殊用途响应。正常通过公共互联网访问 Telegram 时，请保持关闭。
    </Warning>

    - 临时环境覆盖：`OPENCLAW_TELEGRAM_DISABLE_AUTO_SELECT_FAMILY=1`、`OPENCLAW_TELEGRAM_ENABLE_AUTO_SELECT_FAMILY=1`、`OPENCLAW_TELEGRAM_DNS_RESULT_ORDER=ipv4first`。
    - 验证 DNS 响应：

```bash
dig +short api.telegram.org A
dig +short api.telegram.org AAAA
```

  </Accordion>
</AccordionGroup>

更多帮助：[渠道故障排除](/zh-CN/channels/troubleshooting)。

## 配置参考

主要参考：[配置参考 - Telegram](/zh-CN/gateway/config-channels#telegram)。

<Accordion title="重要的 Telegram 字段">

- 启动/身份验证：`enabled`、`botToken`、`tokenFile`（必须是常规文件；符号链接会被拒绝）、`accounts.*`
- 访问控制：`dmPolicy`、`allowFrom`、`groupPolicy`、`groupAllowFrom`、`groups`、`groups.*.topics.*`、顶层 `bindings[]`（`type: "acp"`）
- 话题默认值：`groups.<chatId>.topics."*"` 适用于未匹配的论坛话题；精确的话题 ID 会覆盖它
- Exec 审批：`execApprovals`、`accounts.*.execApprovals`
- 命令/菜单：`commands.native`、`commands.nativeSkills`、`customCommands`
- 线程/回复：`replyToMode`、`threadBindings`
- 流式传输：`streaming`（模式 `off | partial | block | progress`）、`streaming.preview.toolProgress`
- 格式化/递送：`textChunkLimit`、`streaming.chunkMode`、`richMessages`、`markdown.tables`（`off | bullets | code | block`）、`linkPreview`、`responsePrefix`
- 媒体/网络：`mediaMaxMb`、`network.autoSelectFamily`、`network.dangerouslyAllowPrivateNetwork`、`proxy`
- 自定义 API 根地址：`apiRoot`（仅限 Bot API 根地址；不要包含 `/bot<TOKEN>`）、`trustedLocalFileRoots`（自托管 Bot API 的绝对 `file_path` 根地址）
- Webhook：`webhookUrl`、`webhookSecret`、`webhookPath`、`webhookHost`、`webhookPort`、`webhookCertPath`
- 操作/能力：`capabilities.inlineButtons`、`actions.sendMessage|editMessage|deleteMessage|reactions|sticker|createForumTopic|editForumTopic`
- 表情回应：`reactionNotifications`、`reactionLevel`
- 错误：`errorPolicy`、`silentErrorReplies`
- 写入/历史记录：`configWrites`、`historyLimit`、`dmHistoryLimit`、`dms.*.historyLimit`

</Accordion>

<Note>
多账户优先级：配置两个或更多账户 ID 时，请设置 `channels.telegram.defaultAccount`（或包含 `channels.telegram.accounts.default`），以明确指定默认路由。否则，OpenClaw 会回退到第一个规范化的账户 ID，并由 `openclaw doctor` 发出警告。命名账户会继承 `channels.telegram.allowFrom` / `groupAllowFrom`，但不会继承 `accounts.default.*` 的值。
</Note>

## 相关内容

<CardGroup cols={2}>
  <Card title="配对" icon="link" href="/zh-CN/channels/pairing">
    将 Telegram 用户与 Gateway 网关配对。
  </Card>
  <Card title="群组" icon="users" href="/zh-CN/channels/groups">
    群组和话题的允许列表行为。
  </Card>
  <Card title="频道路由" icon="route" href="/zh-CN/channels/channel-routing">
    将入站消息路由到智能体。
  </Card>
  <Card title="安全性" icon="shield" href="/zh-CN/gateway/security">
    威胁模型与安全加固。
  </Card>
  <Card title="多智能体路由" icon="sitemap" href="/zh-CN/concepts/multi-agent">
    将群组和话题映射到智能体。
  </Card>
  <Card title="故障排查" icon="wrench" href="/zh-CN/channels/troubleshooting">
    跨渠道诊断。
  </Card>
</CardGroup>
