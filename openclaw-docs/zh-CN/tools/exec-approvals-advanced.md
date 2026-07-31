---
read_when:
    - 配置安全二进制文件或自定义安全二进制文件配置文件
    - 将审批转发到 Slack、Discord、Telegram 或其他聊天渠道
    - 为渠道实现原生审批客户端
summary: 高级 Exec 审批：安全二进制文件、解释器绑定、审批转发、原生交付
title: Exec 审批 — 高级配置
x-i18n:
    generated_at: "2026-07-26T06:03:07Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: ac90d41f867a8ae4f14b6c9c13f3732d102a65707f456623932b858145a9bf46
    source_path: tools/exec-approvals-advanced.md
    workflow: 16
---

高级 Exec 审批主题：`safeBins` 快速路径、解释器/运行时
绑定，以及将审批转发到聊天渠道（包括原生交付）。
有关核心策略和审批流程，请参阅 [Exec 审批](/zh-CN/tools/exec-approvals)。

## 安全二进制文件（仅限 stdin）

`tools.exec.safeBins` 指定**仅限 stdin** 的二进制文件（例如 `cut`），这些文件
无需显式白名单条目即可在白名单模式下运行。安全二进制文件会拒绝
位置文件参数和类似路径的令牌，因此它们只能处理
传入的数据流。应将其视为流过滤器的有限快速路径，而非
通用信任列表。

<Warning>
**不要**将解释器或运行时二进制文件（例如 `python3`、`node`、
`ruby`、`bash`、`sh`、`zsh`）添加到 `safeBins`。如果命令按设计可以求值代码、
执行子命令或读取文件，请优先使用显式白名单条目，
并保持启用审批提示。自定义安全二进制文件必须在
`tools.exec.safeBinProfiles.<bin>` 中定义显式配置文件。
</Warning>

默认安全二进制文件：

[//]: # "SAFE_BIN_DEFAULTS:START"

`cut`、`uniq`、`head`、`tail`、`tr`、`wc`

[//]: # "SAFE_BIN_DEFAULTS:END"

`grep` 和 `sort` 不在默认列表中。如果选择启用，请为其非 stdin 工作流保留显式
白名单条目。对于安全二进制模式下的 `grep`，
请通过 `-e`/`--regexp` 提供模式；位置模式形式会被拒绝，
因此无法将文件操作数伪装成含义不明确的位置参数。

### Argv 验证和禁止的标志

验证仅根据 argv 结构确定（不检查主机文件系统中是否存在
文件），这可避免允许/拒绝差异造成文件存在性预言机
行为。默认安全二进制文件禁止面向文件的选项；长
选项采用失败关闭验证（拒绝未知标志和含义不明确的缩写）。
默认二进制文件中已识别的只读布尔标志（例如
`wc -l`、`tr -d`、`uniq -c`）会被接受，而无法识别的短标志仍然
采用失败关闭策略，并回退到手动审批。

按安全二进制配置文件列出的禁止标志：

[//]: # "SAFE_BIN_DENIED_FLAGS:START"

- `grep`: `--dereference-recursive`, `--directories`, `--exclude-from`, `--file`, `--recursive`, `-R`, `-d`, `-f`, `-r`
- `jq`: `--argfile`, `--from-file`, `--library-path`, `--rawfile`, `--slurpfile`, `-L`, `-f`
- `sort`: `--compress-program`, `--files0-from`, `--output`, `--random-source`, `--temporary-directory`, `-T`, `-o`
- `tail`: `--follow`, `--retry`, `-F`, `-f`
- `wc`: `--files0-from`

[//]: # "SAFE_BIN_DENIED_FLAGS:END"

安全二进制文件还会强制在执行时将 argv 令牌视为**字面文本**
（不进行 glob 展开，也不展开 `$VARS`），这适用于仅限 stdin 的段，因此
无法使用 `*` 或 `$HOME/...` 等模式夹带文件读取。`awk`、
`sed` 和 `jq` 始终不能作为安全二进制文件，因为无法验证其语义
仅限 stdin：`jq` 可以读取环境数据，并从
模块或启动文件加载 jq 代码。对于这些工具，请使用显式白名单条目或审批提示，
而不要使用 `safeBins`。

### 可信二进制目录

安全二进制文件必须解析自可信二进制目录（系统默认目录以及
可选的 `tools.exec.safeBinTrustedDirs`）。`PATH` 条目绝不会被自动信任。
默认可信目录有意保持最小范围：`/bin`、`/usr/bin`。如果
你的安全二进制可执行文件位于软件包管理器/用户路径中（例如
`/opt/homebrew/bin`、`/usr/local/bin`、`/opt/local/bin`、`/snap/bin`），请将它们
显式添加到 `tools.exec.safeBinTrustedDirs`。

### Shell 链式调用、包装器和多路复用器

当每个顶层段都满足白名单要求（包括安全二进制文件或 Skills 自动允许）时，
允许 Shell 链式调用（`&&`、`||`、`;`）。
白名单模式仍不支持重定向。白名单解析期间会拒绝命令替换
（`$()` / 反引号），包括双引号内的命令替换；如果需要字面
`$()` 文本，请使用单引号。

对于 macOS 配套应用审批，包含 Shell 控制或
展开语法（`&&`、`||`、`;`、`|`、`` ` ``、`$`、`<`、`>`、`(`、`)`）的原始 Shell 文本
会被视为白名单未命中，除非 Shell 二进制文件本身已加入白名单。

对于 Shell 包装器（`bash|sh|zsh ... -c/-lc`），请求范围内的环境变量覆盖
会缩减为一个较小的显式白名单（`TERM`、`LANG`、`LC_*`、`COLORTERM`、
`NO_COLOR`、`FORCE_COLOR`）。

对于白名单模式下的 `allow-always` 决策，透明调度包装器
（例如 `env`、`flock`、`nice`、`nohup`、`stdbuf`、`timeout`）会持久化
内部可执行文件路径，而非包装器路径。Shell 多路复用器
（`busybox`、`toybox`）也会以相同方式针对 Shell 小程序（`sh`、`ash` 等）解除包装。
如果无法安全地解除包装器或多路复用器，则不会自动持久化任何白名单
条目。

如果将 `python3` 或 `node` 等解释器加入白名单，请优先使用
`tools.exec.strictInlineEval=true`，这样内联求值仍需显式
审批。在严格模式下，`allow-always` 仍可持久化无害的
解释器/脚本调用，但不会自动持久化内联求值载体。

### 安全二进制文件与白名单的对比

| 主题             | `tools.exec.safeBins`                                  | 白名单（`exec-approvals.json`）                                                  |
| ---------------- | ------------------------------------------------------ | ---------------------------------------------------------------------------------- |
| 目标             | 自动允许有限的 stdin 过滤器                         | 显式信任特定可执行文件                                              |
| 匹配类型         | 可执行文件名称 + 安全二进制 argv 策略                 | 已解析的可执行文件路径 glob，或用于通过 PATH 调用的命令的裸命令名 glob |
| 参数范围         | 受安全二进制配置文件和字面令牌规则限制 | 默认匹配路径；可选的 `argPattern` 可限制解析后的 argv              |
| 典型示例         | `head`、`tail`、`tr`、`wc`                             | `jq`、`python3`、`node`、`ffmpeg`、自定义 CLI                                     |
| 最佳用途         | 流水线中的低风险文本转换                  | 任何行为范围更广或具有副作用的工具                                     |

配置位置：

- `safeBins` 来自配置（`tools.exec.safeBins` 或按 Agent 配置的 `agents.entries.*.tools.exec.safeBins`）。
- `safeBinTrustedDirs` 来自配置（`tools.exec.safeBinTrustedDirs` 或按 Agent 配置的 `agents.entries.*.tools.exec.safeBinTrustedDirs`）。
- `safeBinProfiles` 来自配置（`tools.exec.safeBinProfiles` 或按 Agent 配置的 `agents.entries.*.tools.exec.safeBinProfiles`）。按 Agent 配置的配置文件键会覆盖全局键。
- 白名单条目位于 `agents.<id>.allowlist` 下的主机本地审批文件中（也可通过 Control UI / `openclaw approvals allowlist ...` 管理）。
- 当解释器/运行时二进制文件出现在 `safeBins` 中但没有显式配置文件时，`openclaw security audit` 会通过 `tools.exec.safe_bins_interpreter_unprofiled` 发出警告。
- `openclaw doctor --fix` 可以将缺失的自定义 `safeBinProfiles.<bin>` 条目搭建为 `{}`（之后请检查并收紧配置）。不会自动搭建解释器/运行时二进制文件。

自定义配置文件示例：

```json5
{
  tools: {
    exec: {
      safeBins: ["myfilter"],
      safeBinProfiles: {
        myfilter: {
          minPositional: 0,
          maxPositional: 0,
          allowedValueFlags: ["-n", "--limit"],
          deniedFlags: ["-f", "--file", "-c", "--command"],
        },
      },
    },
  },
}
```

## 解释器/运行时命令

由审批支持的解释器/运行时执行有意采取保守策略：

- 始终绑定确切的 argv/cwd/env 上下文。
- 直接 Shell 脚本和直接运行时文件形式会尽最大努力绑定到一个具体的本地
  文件快照。
- 仍可解析到一个直接本地文件的常见软件包管理器包装形式（例如
  `pnpm exec`、`pnpm node`、`npm exec`、`npx`）会在绑定前解除包装。
- 如果 OpenClaw 无法为解释器/运行时命令识别出恰好一个具体的本地文件
  （例如软件包脚本、求值形式、运行时特有的加载器链或含义不明确的多文件
  形式），则会拒绝由审批支持的执行，而不会声称提供了实际上并不具备的语义
  覆盖。
- 对于这些工作流，请优先使用沙箱隔离、独立的主机边界，或显式可信的
  白名单/完整工作流，由操作员接受更广泛的运行时语义。

需要审批时，Exec 工具会立即返回审批 ID。使用该 ID
关联之后已获批准执行的系统事件（`Exec finished`，以及配置后的 `Exec running`）。
如果超时前未收到决定，则请求会被视为审批超时，并
作为最终主机命令拒绝呈现。对于具有来源会话的主 Agent 异步审批，
OpenClaw 还会通过内部后续消息恢复该会话，使 Agent 知道
命令并未执行，而不是之后再修复缺失的结果。默认情况下，待处理的 Exec 审批会在
30 分钟后过期。

### 后续交付行为

获批的异步 Exec 完成后，OpenClaw 会向同一会话发送后续 `agent` 轮次。
被拒绝的异步审批使用相同的主会话后续路径传递拒绝状态，但不会
注册提升权限的运行时交接，也不会运行命令。如果拒绝没有可恢复的
主会话，则会抑制该拒绝，或在存在安全直接路由时通过该路由报告。

- 如果存在有效的外部交付目标（可交付渠道加目标 `to`），后续交付将使用该渠道。
- 在没有外部目标、仅限 Webchat 或内部会话的流程中，后续交付仅保留在会话内（`deliver: false`）。
- 如果调用方在没有可解析外部渠道时明确请求严格外部交付，请求将以 `INVALID_REQUEST` 失败。
- 如果已启用 `bestEffortDeliver` 且无法解析外部渠道，交付会降级为仅限会话，而不是失败。

## 第三方客户端的最小权限范围

Gateway 网关审批解析受专用 `operator.approvals` 权限范围保护。这同时适用于所有者专用的 `exec.approval.resolve` 方法和与种类无关的 `approval.resolve` 方法；`operator.write` 并不包含该权限。仪表板和集成应仅请求其所用方法所需的权限范围。应将审批解析访问权限视为远程执行级权限，并有意授予 `operator.approvals`，即使客户端仅呈现一个小型审批 UI。

## 将审批转发到聊天渠道

你可以将 Exec 审批提示转发到任何聊天渠道（包括插件渠道），并通过 `/approve` 审批。这使用常规的出站投递流水线。

配置：

```json5
{
  approvals: {
    exec: {
      enabled: true,
      mode: "session", // "session" | "targets" | "both"
      agentFilter: ["main"],
      sessionFilter: ["discord"], // substring or regex
      targets: [
        { channel: "slack", to: "U12345678" },
        { channel: "telegram", to: "123456789" },
      ],
    },
  },
}
```

在聊天中回复：

```
/approve <id> allow-once
/approve <id> allow-always
/approve <id> deny
```

`/approve` 命令同时处理 Exec 审批和插件审批。如果 ID 与待处理的 Exec 审批不匹配，它会自动改为检查插件审批。此回退仅限于“找不到审批”故障；真正的 Exec 审批拒绝或错误不会静默地以插件审批方式重试。

### 插件审批转发

插件审批转发使用与 Exec 审批相同的投递流水线，但在 `approvals.plugin` 下具有独立配置。启用或禁用其中一项不会影响另一项。有关插件开发行为、请求字段和决策语义，请参阅[插件权限请求](/plugins/plugin-permission-requests)。

```json5
{
  approvals: {
    plugin: {
      enabled: true,
      mode: "targets",
      agentFilter: ["main"],
      targets: [
        { channel: "slack", to: "U12345678" },
        { channel: "telegram", to: "123456789" },
      ],
    },
  },
}
```

配置结构与 `approvals.exec` 相同：`enabled`、`mode`、`agentFilter`、`sessionFilter` 和 `targets` 的工作方式相同。

支持共享交互式回复的渠道会为 Exec 审批和插件审批呈现相同的审批按钮。没有共享交互式 UI 的渠道会回退到包含 `/approve` 指令的纯文本。插件审批请求可能会限制可用决策：审批界面使用请求中声明的决策集，Gateway 网关会拒绝提交未提供的决策。

### 任意渠道中的同一聊天审批

当 Exec 或插件审批请求源自可投递的聊天界面时，默认情况下，同一聊天可以使用 `/approve` 审批该请求。除了现有的 Web UI 和终端 UI 流程之外，这也适用于 Slack、Matrix、Microsoft Teams 及类似的可投递聊天，并使用该对话的常规渠道身份验证模型。如果来源聊天已经能够发送命令并接收回复，审批请求便不再需要单独的原生投递适配器来保持待处理状态。

Discord、Telegram 和 QQ Bot 也支持同一聊天中的 `/approve`，但即使原生审批投递已禁用，这些渠道仍会使用其解析出的审批人列表进行授权。

### 原生审批投递

某些渠道还可充当原生审批客户端：Discord、Slack、Telegram、Matrix 和 QQ Bot。原生客户端在共享的同一聊天 `/approve` 流程之上，增加审批人私信、来源聊天扇出和渠道特定的交互式审批体验。

当原生审批卡片或按钮可用时，该原生 UI 是面向智能体的主要路径。除非工具结果表明聊天审批不可用，或手动审批是唯一剩余路径，否则智能体不应再回显重复的纯聊天 `/approve` 命令。

如果已配置原生审批客户端，但来源渠道没有活跃的原生运行时，OpenClaw 会保持显示本地确定性的 `/approve` 提示。如果原生运行时处于活跃状态并尝试投递，但没有目标收到卡片，OpenClaw 会发送同一聊天回退通知，其中包含确切的 `/approve <id> <decision>` 命令，以便仍能处理该请求。

通用模型：

- 主机 Exec 策略仍决定是否需要 Exec 审批
- `approvals.exec` 控制是否将审批提示转发到其他聊天目的地
- `channels.<channel>.execApprovals` 控制是否启用 Discord、Slack、Telegram、QQ Bot 及类似的渠道特定原生客户端
- 当请求来自 Slack 且能够解析出 Slack 插件审批人时，Slack 插件审批可以使用 Slack 的原生审批客户端；即使 Slack Exec 审批已禁用，`approvals.plugin` 也可以将插件审批路由到 Slack 会话或目标
- 当能够从 `dm.allowFrom` 或 `defaultTo` 解析出稳定的 `users/<id>` 审批人时，Google Chat 原生审批卡片会处理源自 Google Chat 空间或话题串的 Exec 和插件审批；它们不使用表情回应事件来作出决策
- WhatsApp 和 Signal 表情回应审批投递受 `approvals.exec` 和 `approvals.plugin` 控制；它们没有 `channels.<channel>.execApprovals` 块

当以下所有条件均成立时，原生审批客户端会自动启用以私信优先的投递：

- 渠道支持原生审批投递
- 可以从显式的 `execApprovals.approvers` 或 `commands.ownerAllowFrom` 等所有者身份解析出审批人
- `channels.<channel>.execApprovals.enabled` 未设置或为 `"auto"`

将 `enabled: false` 设置为显式禁用原生审批客户端。将 `enabled: true` 设置为在能够解析出审批人时强制启用。公开的来源聊天投递仍需通过 `channels.<channel>.execApprovals.target` 显式启用。当原生 `target` 启用来源聊天投递时，审批提示会包含命令文本。

常见问题：[为什么聊天审批有两项 Exec 审批配置？](/help/faq-first-run)

- Discord：`channels.discord.execApprovals.*`
- Slack：`channels.slack.execApprovals.*`
- Telegram：`channels.telegram.execApprovals.*`
- QQ Bot：`channels.qqbot.execApprovals.*`
- Google Chat：使用 `channels.googlechat.dm.allowFrom` 或 `channels.googlechat.defaultTo` 配置稳定的审批人；不需要 `execApprovals` 块
- WhatsApp：使用 `approvals.exec` 和 `approvals.plugin` 将审批提示路由到 WhatsApp
- Signal：使用 `approvals.exec` 和 `approvals.plugin` 将审批提示路由到 Signal

原生客户端特定路由：

- Telegram 默认向审批人发送私信（`target: "dm"`）。切换为 `channel` 或 `both`，也可以在来源 Telegram 聊天或话题中显示审批提示。对于 Telegram 论坛话题，OpenClaw 会为审批提示和审批后的后续消息保留该话题。
- Discord 和 Telegram 审批人可以显式指定（`execApprovals.approvers`），也可以从 `commands.ownerAllowFrom` 推断；只有解析出的审批人才能批准或拒绝。
- Slack 审批人可以显式指定（`execApprovals.approvers`），也可以从 `commands.ownerAllowFrom` 推断。Slack 插件审批私信使用来自 `allowFrom` 的 Slack 插件审批人和账户默认路由，而不是 Slack Exec 审批人。Slack 原生按钮会保留审批 ID 类型，因此 `plugin:` ID 可以处理插件审批，而不需要第二层 Slack 本地回退。
- Google Chat 原生卡片会在消息文本中保留手动 `/approve` 回退，但卡片按钮回调仅携带不透明操作令牌；审批 ID 和决策会从服务器端待处理状态中恢复。
- 当匹配的顶层转发类别路由到 WhatsApp 时，WhatsApp 表情符号审批会处理 Exec 和插件提示。原生来源提示会直接绑定；共享目标模式投递会将相同的类型化审批元数据绑定到已接受的 WhatsApp 消息回执。
- 仅当匹配的顶层转发类别已启用并路由到 Signal 时，Signal 表情回应审批才会处理 Exec 和插件提示。直接在同一 Signal 聊天中进行的 Exec 审批可以在没有显式审批人的情况下抑制本地 `/approve` 回退；Signal 表情回应解析仍需要来自 `channels.signal.allowFrom` 或 `defaultTo` 的显式 Signal 审批人。
- Matrix 原生私信/渠道路由和表情回应快捷方式会处理 Exec 和插件审批；插件授权仍来自 `channels.matrix.dm.allowFrom`。Matrix 原生提示会在第一个提示事件中包含 `com.openclaw.approval` 自定义事件内容，以便支持 OpenClaw 的 Matrix 客户端读取结构化审批状态，而标准客户端仍保留纯文本 `/approve` 回退。
- 原生 Discord 和 Telegram 审批按钮在传输私有的回调数据中携带显式的 Exec 或插件所有者类型，并且仅处理该所有者。缺少类型的旧版 `/approve` 控件仍作为受限的兼容路径：它们只尝试操作者有权审批的所有者类型，仅在得到找不到审批的结果后继续，并且绝不会根据审批 ID 推断所有权。
- 请求者不需要是审批人。
- 如果没有操作员 UI 或已配置的审批客户端能够接受请求，提示会回退到 `askFallback`。

`/diagnostics` 和 `/export-trajectory` 等仅限所有者使用的敏感群组命令，会对审批提示和最终结果使用私有所有者路由。OpenClaw 首先尝试在所有者运行命令的同一界面上使用私有路由。如果该界面没有私有所有者路由，则会回退到 `commands.ownerAllowFrom` 中第一个可用的所有者路由，因此，当 Telegram 是已配置的主要私有界面时，Discord 群组命令仍可将审批和结果发送到所有者的 Telegram 私信。群聊只会收到简短的确认消息。

另请参阅：

- [Discord](/channels/discord)
- [Telegram](/channels/telegram)
- [QQ Bot](/channels/qqbot)

### 官方移动端操作员应用

官方 iOS 和 Android 应用也可以审查由 Gateway 网关所有的待处理 Exec 审批，前提是使用 `operator.admin` 连接，或者请求明确指定了其已配对的 `operator.approvals` 设备。它们读取 Control UI 所使用的同一份经过净化的持久记录，提交感知类型的决策，并显示 Gateway 网关规范的首次响应结果。Apple Watch 会通过已配对的 iPhone 镜像这些审批提示，并提供允许一次和拒绝操作。Watch 直连 Gateway 网关模式不会审查审批。

丢失处理确认并不会使已提交的选择成为权威结果：应用会禁用控件并再次读取记录。如果另一个界面先完成处理，应用会显示记录中的决策。待处理提示始终绑定到发出它们的 Gateway 网关，因此切换活跃的 Gateway 网关无法重定向旧的审批 ID。

### macOS IPC 流程

```
Gateway 网关 -> 节点服务 (WS)
                 |  IPC (UDS + 令牌 + HMAC + TTL)
                 v
             Mac 应用（UI + 审批 + system.run）
```

安全说明：

- Unix 套接字模式 `0600`，令牌存储在 `exec-approvals.json` 中。
- 同一 UID 对等方检查。
- 质询/响应（随机数 + HMAC 令牌 + 请求哈希）+ 短 TTL。

## 常见问题

### 何时会在审批目标上使用 `accountId` 和 `threadId`？

当渠道配置了多个身份，并且审批提示必须通过某个特定账户发出时，使用 `accountId`。当目的地支持话题或话题串，并且提示应保留在该话题串内而不是顶层聊天中时，使用 `threadId`。

一个具体的 Telegram 示例是具有论坛话题和两个 Telegram Bot 账户的运维超级群组。`to` 值指定超级群组，`accountId` 选择 Bot 账户，`threadId` 选择论坛话题：

```json5
{
  approvals: {
    exec: {
      enabled: true,
      mode: "targets",
      targets: [
        {
          channel: "telegram",
          to: "-1001234567890",
          accountId: "ops-bot",
          threadId: "77",
        },
      ],
    },
  },
  channels: {
    telegram: {
      accounts: {
        default: {
          name: "Primary bot",
          botToken: "env:TELEGRAM_PRIMARY_BOT_TOKEN",
        },
        "ops-bot": {
          name: "Operations bot",
          botToken: "env:TELEGRAM_OPS_BOT_TOKEN",
        },
      },
    },
  },
}
```

采用该设置后，转发的 Exec 审批会由 `ops-bot` Telegram 账号发布到聊天 `-1001234567890` 的话题
`77` 中。未指定 `accountId` 的目标使用该渠道的默认账号，未指定
`threadId` 的目标则发布到顶层目的地。

### 将审批发送到会话后，该会话中的任何人都能批准吗？

不能。会话投递仅控制提示出现的位置。它本身并不会授权该聊天中的所有
参与者进行批准。

对于通用的同聊天 `/approve`，发送者必须已经获得在该
渠道会话中使用命令的授权。如果渠道公开了明确的审批批准者，那么即使这些批准者在该会话中没有其他命令授权，也可以授权
`/approve` 操作。

某些渠道更为严格。Discord、Telegram、Matrix、Slack 原生审批私信以及类似的
原生审批客户端使用其解析出的批准者列表进行审批授权。例如，
Telegram 论坛话题中的审批提示可能对话题内的所有人可见，但只有从 `channels.telegram.execApprovals.approvers` 或
`commands.ownerAllowFrom` 解析出的数字 Telegram 用户 ID 才能批准或拒绝该提示。

## 相关内容

- [Exec 审批](/zh-CN/tools/exec-approvals) — 核心策略和审批流程
- [Exec 工具](/zh-CN/tools/exec)
- [提升权限模式](/zh-CN/tools/elevated)
- [Skills](/zh-CN/tools/skills) — 由技能支持的自动允许行为
