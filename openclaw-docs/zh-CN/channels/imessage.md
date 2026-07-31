---
read_when:
    - 设置 iMessage 支持
    - 调试 iMessage 收发
summary: 通过 imsg（基于 stdio 的 JSON-RPC）提供原生 iMessage 支持，并通过私有 API 支持回复、点按回应、特效、投票、附件和群组管理操作。若主机满足要求，建议新的 OpenClaw iMessage 设置优先采用此方式。
title: iMessage
x-i18n:
    generated_at: "2026-07-26T06:08:45Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: f3e8b1a65c76b25d03615c06a976f86a8af555cd96d5bfdb10cef9c955893ddc
    source_path: channels/imessage.md
    workflow: 16
---

<Note>
对于常规的 OpenClaw iMessage 部署，请在同一台已登录 macOS Messages 的主机上运行 Gateway 网关和 `imsg`。如果 Gateway 网关在其他位置运行，请将 `channels.imessage.cliPath` 指向一个透明的 SSH 包装脚本，由它在 Mac 上运行 `imsg`。

**入站恢复会自动进行。** bridge 或 Gateway 网关重启后，iMessage 会重放停机期间错过的消息，并抑制 Apple 在 Push 恢复后可能集中刷出的陈旧“积压消息炸弹”，同时进行去重，确保不会重复分发任何消息。无需通过配置启用此功能——请参阅 [bridge 或 Gateway 网关重启后的入站恢复](#inbound-recovery-after-a-bridge-or-gateway-restart)。
</Note>

<Warning>
BlueBubbles 支持已移除。请将 `channels.bluebubbles` 配置迁移到 `channels.imessage`；OpenClaw 仅通过 `imsg` 支持 iMessage。简要公告请先阅读 [BlueBubbles removal and the imsg iMessage path](/zh-CN/announcements/bluebubbles-imessage)，完整迁移表请参阅 [Coming from BlueBubbles](/zh-CN/channels/imessage-from-bluebubbles)。
</Warning>

状态：原生外部 CLI 集成。Gateway 网关会启动 `imsg rpc`，并通过 stdio 使用 JSON-RPC 通信，无需单独的守护进程或端口。强烈建议使用私有 API 模式，以获得完整的 iMessage 渠道功能；回复、点按回应、特效、投票、附件回复和群组操作都需要 `imsg launch`，并且私有 API 探测必须成功。

对于常见的本地设置，OpenClaw 设置流程可以在已登录 Messages 的 Mac 上，经用户确认后通过 Homebrew 安装或更新 `imsg`。手动设置和 SSH 包装脚本拓扑仍由操作员管理：请在将要运行 Gateway 网关或包装脚本的同一用户上下文中安装或更新 `imsg`。

<CardGroup cols={3}>
  <Card title="私有 API 操作" icon="wand-sparkles" href="#private-api-actions">
    回复、点按回应、特效、投票、附件和群组管理。
  </Card>
  <Card title="配对" icon="link" href="/zh-CN/channels/pairing">
    iMessage 私信默认使用配对模式。
  </Card>
  <Card title="远程 Mac" icon="terminal" href="#remote-mac-over-ssh">
    当 Gateway 网关未在 Messages Mac 上运行时，请使用 SSH 包装脚本。
  </Card>
  <Card title="配置参考" icon="settings" href="/zh-CN/gateway/config-channels#imessage">
    完整的 iMessage 字段参考。
  </Card>
</CardGroup>

## 快速设置

<Tabs>
  <Tab title="本地 Mac（快速路径）">
    <Steps>
      <Step title="安装并验证 imsg">

```bash
brew install steipete/tap/imsg
brew update && brew upgrade imsg
imsg rpc --help
imsg launch
openclaw channels status --probe
```

        当本地设置向导检测到缺少默认的 `imsg` 命令时，可以提示通过 Homebrew 安装 `steipete/tap/imsg`。如果检测到由 Homebrew 管理的 `imsg`，则可以提示重新安装或更新它。不会修改自定义的 `cliPath` 包装脚本。

      </Step>

      <Step title="配置 OpenClaw">

```json5
{
  channels: {
    imessage: {
      enabled: true,
      cliPath: "/usr/local/bin/imsg",
      dbPath: "/Users/user/Library/Messages/chat.db",
    },
  },
}
```

      </Step>

      <Step title="启动 Gateway 网关">

```bash
openclaw gateway
```

      </Step>

      <Step title="批准首次私信配对（默认 dmPolicy）">

```bash
openclaw pairing list imessage
openclaw pairing approve imessage <CODE>
```

        配对请求将在 1 小时后过期。
      </Step>
    </Steps>

  </Tab>

  <Tab title="通过 SSH 连接远程 Mac">
    大多数设置不需要 SSH。仅当 Gateway 网关无法在已登录 Messages 的 Mac 上运行时，才使用此拓扑。OpenClaw 只需要一个兼容 stdio 的 `cliPath`，因此可以将 `cliPath` 指向一个包装脚本，由该脚本通过 SSH 连接远程 Mac 并运行 `imsg`。
    请在该远程 Mac 上安装和更新 `imsg`，而不是在 Gateway 网关主机上：

```bash
ssh messages-mac 'brew install steipete/tap/imsg && brew update && brew upgrade imsg'
```

```bash
#!/usr/bin/env bash
exec ssh -T messages-mac imsg "$@"
```

    启用附件时的推荐配置：

```json5
{
  channels: {
    imessage: {
      enabled: true,
      cliPath: "~/.openclaw/scripts/imsg-ssh",
      remoteHost: "user@gateway-host", // 用于通过 SCP 获取附件
      includeAttachments: true,
      // 可选：额外允许的附件根目录（与默认的
      // /Users/*/Library/Messages/Attachments 合并）。
      attachmentRoots: ["/Users/*/Library/Messages/Attachments"],
      remoteAttachmentRoots: ["/Users/*/Library/Messages/Attachments"],
    },
  },
}
```

    如果未设置 `remoteHost`，OpenClaw 会尝试通过解析 SSH 包装脚本自动检测它。
    `remoteHost` 必须是 `host` 或 `user@host`（不得包含空格或 SSH 选项）；不安全的值将被忽略。
    OpenClaw 对 SCP 使用严格的主机密钥检查，因此中继主机密钥必须已存在于 `~/.ssh/known_hosts` 中。
    附件路径会根据允许的根目录（`attachmentRoots` / `remoteAttachmentRoots`）进行验证。

<Warning>
放在 `imsg` 前面的任何 `cliPath` 包装脚本或 SSH 代理都**必须**像透明的 stdio 管道一样处理长连接 JSON-RPC。在渠道的整个生命周期内，OpenClaw 会通过包装脚本的 stdin/stdout 交换以换行符分帧的小型 JSON-RPC 消息：

- 一旦有字节可用，就立即转发每个 stdin 数据块/行——不要等待 EOF。
- 及时沿反方向转发每个 stdout 数据块/行。
- 保留换行符。
- 避免使用可能导致小型帧无法得到处理的固定大小阻塞读取（`read(4096)`、`cat | buffer`、shell 默认的 `read`）。
- 将 stderr 与 JSON-RPC stdout 流分开。

如果包装脚本一直缓冲 stdin，直到填满一个大数据块，就会产生类似 iMessage 中断的症状——`imsg rpc timeout (chats.list)` 或渠道反复重启——即使 `imsg rpc` 本身运行正常。上面的 `ssh -T host imsg "$@"` 是安全的，因为它会转发 OpenClaw 的 `cliPath` 参数，例如 `rpc` 和 `--db`。类似 `ssh host imsg | grep -v '^DEBUG'` 的管道则**不安全**——即使采用行缓冲，工具仍可能滞留帧；如果必须进行过滤，请在每个阶段使用 `stdbuf -oL -eL`。
</Warning>

  </Tab>
</Tabs>

## 要求和权限（macOS）

- 必须在运行 `imsg` 的 Mac 上登录 Messages。
- 运行 OpenClaw/`imsg` 的进程上下文需要拥有“完全磁盘访问权限”（用于访问 Messages 数据库）。
- 通过 Messages.app 发送消息需要“自动化”权限。
- 高级操作（回应 / 编辑 / 撤回 / 线程回复 / 特效 / 投票 / 群组操作）要求禁用系统完整性保护——请参阅[启用 imsg 私有 API](#enabling-the-imsg-private-api)。无需禁用该功能即可进行基本的文本和媒体收发。

<Tip>
权限按进程上下文授予。如果 Gateway 网关以无头方式运行（LaunchAgent/SSH），请在同一上下文中运行一次交互式命令来触发权限提示：

```bash
imsg chats --limit 1
# 或
imsg send <handle> "测试"
```

</Tip>

<Accordion title="SSH 包装脚本发送失败，出现 AppleEvents -1743">
  远程 SSH 设置可以读取聊天、通过 `channels status --probe` 并处理入站消息，但出站发送仍可能因 AppleEvents 授权错误而失败：

```text
无权向 Messages 发送 Apple 事件。(-1743)
```

请检查已登录 Mac 用户的 TCC 数据库，或打开 System Settings > Privacy & Security > Automation。如果“自动化”条目记录的是 `/usr/libexec/sshd-keygen-wrapper`，而不是 `imsg` 或本地 shell 进程，macOS 可能不会为该 SSH 服务端客户端显示可用的 Messages 开关：

```text
kTCCServiceAppleEvents | /usr/libexec/sshd-keygen-wrapper | auth_value=0 | com.apple.MobileSMS
```

在这种状态下，重复执行 `tccutil reset AppleEvents` 或通过同一 SSH 包装脚本重新运行 `imsg send` 可能仍会失败，因为需要 Messages“自动化”权限的进程上下文是 SSH 包装脚本，而不是 UI 可以向其授权的应用。

请改用以下受支持的 `imsg` 进程上下文之一：

- 在已登录 Messages 用户的本地会话中运行 Gateway 网关，或至少运行 `imsg` bridge。
- 在同一会话中授予“完全磁盘访问权限”和“自动化”权限后，使用该用户的 LaunchAgent 启动 Gateway 网关。
- 如果保留双用户 SSH 拓扑，请先验证真实的出站 `imsg send` 能否通过确切的包装脚本成功执行，再启用渠道。如果无法为其授予“自动化”权限，请改为单用户 `imsg` 设置，不要依赖 SSH 包装脚本进行发送。

</Accordion>

## 启用 imsg 私有 API

`imsg` 提供两种运行模式。对于 OpenClaw，推荐使用私有 API 模式，因为它能让渠道获得用户期望的原生 iMessage 操作。基本模式仍适用于低风险安装、初始验证，或无法禁用 SIP 的主机。

- **基本模式**（默认，无需更改 SIP）：通过 `send` 发送文本和媒体、入站监听/历史记录、聊天列表。这是全新安装 `brew install steipete/tap/imsg` 并授予上述标准 macOS 权限后即可获得的功能。
- **私有 API 模式**：`imsg` 会将辅助 dylib 注入 `Messages.app`，以调用内部 `IMCore` 函数。这样可解锁 `react`、`edit`、`unsend`、`reply`（线程式）、`sendWithEffect`、`poll` 和 `poll-vote`（Messages 原生投票）、`renameGroup`、`setGroupIcon`、`addParticipant`、`removeParticipant`、`leaveGroup`，以及正在输入指示和已读回执。

本页面推荐的操作功能需要私有 API 模式。`imsg` README 明确说明了这一要求：

> `read`、`typing`、`launch`、由 bridge 支持的富媒体发送、消息修改和聊天管理等高级功能需要选择性启用。它们要求禁用 SIP，并将辅助 dylib 注入 `Messages.app`。启用 SIP 时，`imsg launch` 会拒绝注入。

这种辅助注入技术使用 `imsg` 自身的 dylib 来访问 Messages 私有 API。OpenClaw iMessage 路径中不存在第三方服务器或 BlueBubbles 运行时。

<Warning>
**禁用 SIP 会带来切实的安全权衡。** SIP 是 macOS 防止运行经过修改的系统代码的核心保护机制之一；在整个系统中关闭它会扩大攻击面并带来额外副作用。特别需要注意的是，**在 Apple 芯片 Mac 上禁用 SIP 还会导致无法在 Mac 上安装和运行 iOS 应用**。

请将其视为需要慎重决定的运维选择，尤其是在主要个人 Mac 上。对于生产级 OpenClaw iMessage，最好使用一台专用 Mac 或专用机器人 macOS 用户，并确保可以接受启用该 bridge。如果你的威胁模型无法容忍在任何位置关闭 SIP，则内置 iMessage 仅限基本模式——只能发送和接收文本及媒体，不支持回应 / 编辑 / 撤回 / 特效 / 群组操作。
</Warning>

### 设置

1. 在运行 Messages.app 的 Mac 上**安装（或升级）`imsg`**：

   ```bash
   brew install steipete/tap/imsg
   brew update && brew upgrade imsg
   imsg --version
   imsg status --json
   ```

   `imsg status --json` 输出会报告 `bridge_version`、`rpc_methods` 和每种方法的 `selectors`，以便在开始之前查看当前构建支持的功能。

2. **禁用系统完整性保护，并且（在现代 macOS 上）禁用库验证。** 将非 Apple 的辅助 dylib 注入 Apple 签名的 `Messages.app`，需要关闭 SIP **并**放宽库验证。恢复模式下的 SIP 操作因 macOS 版本而异：
   - **macOS 10.13-10.15（Sierra-Catalina）：**通过终端禁用库验证，重新启动进入恢复模式，运行 `csrutil disable`，然后重新启动。
   - **macOS 11+（Big Sur 及更高版本），Intel：**进入恢复模式（或互联网恢复），运行 `csrutil disable`，然后重新启动。
   - **macOS 11+，Apple Silicon：**使用电源按钮启动流程进入恢复模式；在较新的 macOS 版本上，点击 Continue 时按住 **Left Shift** 键，然后运行 `csrutil disable`。虚拟机设置采用单独的流程，因此请先创建虚拟机快照。

   **在 macOS 11 及更高版本上，仅使用 `csrutil disable` 通常不够。** Apple 仍会针对作为平台二进制文件的 `Messages.app` 强制执行库验证，因此即使已关闭 SIP，临时签名的辅助程序也会被拒绝（`Library Validation failed: ... platform binary, but mapped file is not`）。禁用 SIP 后，还需禁用库验证并重新启动：

   ```bash
   sudo defaults write /Library/Preferences/com.apple.security.libraryvalidation.plist DisableLibraryValidation -bool true
   ```

   **macOS 26（Tahoe），已在 26.5.1 上验证：**关闭 SIP **并**执行上面的 `DisableLibraryValidation` 命令，即可在 26.0 到 26.5.x 的所有版本中注入辅助程序。**不需要 boot-args。**该 plist 是决定性因素，也是 Tahoe 上注入失败时最常遗漏的步骤：
   - **存在该 plist 时：**`imsg launch` 可以完成注入，且 `imsg status` 会报告 `advanced_features: true`。
   - **不存在该 plist 时（即使已关闭 SIP）：**`imsg launch` 会失败并显示 `Failed to launch: Timeout waiting for Messages.app to initialize`。AMFI 会在加载时拒绝临时签名的辅助程序，因此桥接永远无法就绪，启动最终超时。这是大多数人在 Tahoe 上遇到的症状；修复方法是添加上面的 plist，而不是采取更激进的措施。

   如果 macOS 升级后，`imsg launch` 注入或特定 `selectors` 开始返回 false，通常是此门控导致的。在认定 SIP 步骤本身失败之前，请检查 SIP 和库验证状态。如果这些设置正确，但桥接仍无法注入，请收集 `imsg status --json` 以及 `imsg launch` 的输出，并将其报告给 `imsg` 项目，而不是进一步削弱系统范围的安全控制。

3. **注入辅助程序。**在 SIP 已禁用且 Messages.app 已登录的情况下：

   ```bash
   imsg launch
   ```

   SIP 仍处于启用状态时，`imsg launch` 会拒绝注入，因此这也可用于确认第 2 步已生效。

4. **通过 OpenClaw 验证桥接：**

   ```bash
   openclaw channels status --probe
   ```

   iMessage 条目应报告 `works`，而 `imsg status --json | jq '{rpc_methods, selectors}'` 应显示你的 macOS 构建所公开的能力。创建投票需要 `selectors.pollPayloadMessage`；投票操作同时需要 `selectors.pollVoteMessage` 和 `poll.vote` RPC 方法。OpenClaw 插件仅公布缓存探测所支持的操作；缓存为空时则保持乐观，并在首次分派时进行探测。

如果 `openclaw channels status --probe` 将渠道报告为 `works`，但特定操作在分派时抛出“iMessage `<action>` requires the imsg private API bridge”，请再次运行 `imsg launch`——辅助程序可能会脱离（Messages.app 重新启动、操作系统更新等），而缓存的 `available: true` 状态会继续公布这些操作，直到下一次探测刷新状态。

### SIP 保持启用时

如果你的威胁模型不允许禁用 SIP：

- `imsg` 会回退到基本模式——仅支持文本、媒体和接收。
- OpenClaw 插件仍会公布文本/媒体发送和入站监控功能；它会从操作界面中隐藏 `react`、`edit`、`unsend`、`reply`、`sendWithEffect` 和群组操作（依据逐方法能力门控）。
- 你可以使用另一台关闭 SIP 的非 Apple Silicon Mac（或专用机器人 Mac）承载 iMessage 工作负载，同时在主要设备上保持 SIP 启用。请参阅下方的[专用机器人 macOS 用户（独立 iMessage 身份）](#deployment-patterns)。

## 访问控制和路由

<Tabs>
  <Tab title="私信策略">
    `channels.imessage.dmPolicy` 控制私信：

    - `pairing`（默认）
    - `allowlist`（至少需要一个 `allowFrom` 条目）
    - `open`（要求 `allowFrom` 包含 `"*"`）
    - `disabled`

    允许列表字段：`channels.imessage.allowFrom`。

    允许列表条目必须标识发送者：句柄或静态发送者访问组（`accessGroup:<name>`）。对于 `chat_id:*`、`chat_guid:*` 或 `chat_identifier:*` 等聊天目标，请使用 `channels.imessage.groupAllowFrom`；对于数字形式的 `chat_id` 注册表键，请使用 `channels.imessage.groups`。

  </Tab>

  <Tab title="群组策略 + 提及">
    `channels.imessage.groupPolicy` 控制群组处理：

    - `allowlist`（默认）
    - `open`
    - `disabled`

    群组发送者允许列表：`channels.imessage.groupAllowFrom`。

    `groupAllowFrom` 条目还可以引用静态发送者访问组（`accessGroup:<name>`）。

    运行时回退：如果未设置 `groupAllowFrom`，iMessage 群组发送者检查将使用 `allowFrom`；当私信和群组的准入规则应不同时，请设置 `groupAllowFrom`。显式为空的 `groupAllowFrom: []` 不会回退——它会在 `allowlist` 下阻止所有群组发送者。
    运行时说明：如果完全缺少 `channels.imessage`，运行时会回退到 `groupPolicy="allowlist"` 并记录警告（即使已设置 `channels.defaults.groupPolicy`）。

    <Warning>
    `groupPolicy: "allowlist"` 下的群组路由会连续执行**两道**门控：

    1. **发送者允许列表**（`channels.imessage.groupAllowFrom`）——句柄、`accessGroup:<name>`、`chat_guid`、`chat_identifier` 或 `chat_id`。有效列表为空（没有 `groupAllowFrom`，也没有 `allowFrom` 回退）时，会阻止所有群组发送者。
    2. **群组注册表**（`channels.imessage.groups`）——该映射存在条目后即会强制执行：聊天必须匹配一个显式的逐 `chat_id` 条目或 `groups: { "*": { ... } }` 通配符。当 `groups` 为空或缺失时，仅由发送者允许列表决定是否准入。

    如果未配置有效的群组发送者允许列表，所有群组消息都会在注册表门控之前被丢弃。每道门控在默认日志级别下都有各自的 `warn` 级信号，并分别指出不同的修复方法：

    - 每个账号在启动时记录一次：当有效群组发送者允许列表为空时，显示 `imessage: groupPolicy="allowlist" for account "<id>" but no group sender allowlist is configured ...`——通过设置 `channels.imessage.groupAllowFrom`（或 `allowFrom`）修复；仅添加 `groups` 条目仍会导致门控 1 阻止所有发送者。
    - 运行时每个 `chat_id` 记录一次：当发送者通过门控 1，但聊天未包含在已有内容的 `groups` 注册表中时，显示 `imessage: dropping group message from chat_id=<id> ...`——通过在 `channels.imessage.groups` 下添加该 `chat_id`（或 `"*"`）来修复。

    私信不受影响——它们采用不同的代码路径。

    `groupPolicy: "allowlist"` 下群组流程的推荐配置：

    ```json5
    {
      channels: {
        imessage: {
          groupPolicy: "allowlist",
          groupAllowFrom: ["+15555550123"],
          groups: { "*": { "requireMention": true } },
        },
      },
    }
    ```

    仅使用 `groupAllowFrom` 即可允许这些发送者进入任何群组；添加 `groups` 块可限定允许哪些聊天（并设置 `requireMention` 等逐聊天选项）。
    </Warning>

    群组的提及门控：

    - iMessage 没有原生提及元数据
    - 提及检测使用正则表达式模式（`agents.entries.*.groupChat.mentionPatterns`，回退为 `messages.groupChat.mentionPatterns`）
    - 未配置模式时，无法强制执行提及门控
    - 来自已授权发送者的控制命令可绕过提及门控

    逐群组 `systemPrompt`：

    `channels.imessage.groups.*` 下的每个条目都接受一个可选的 `systemPrompt` 字符串；每当处理该群组中的消息时，都会将其注入智能体的系统提示词。解析方式与 `channels.whatsapp.groups` 一致：

    1. **群组专用系统提示词**（`groups["<chat_id>"].systemPrompt`）：当映射中存在特定群组条目，**并且**已定义其 `systemPrompt` 键时使用。如果 `systemPrompt` 是空字符串（`""`），则会抑制通配符，并且不对该群组应用任何系统提示词。
    2. **群组通配符系统提示词**（`groups["*"].systemPrompt`）：当映射中完全不存在特定群组条目，或该条目存在但未定义 `systemPrompt` 键时使用。

    ```json5
    {
      channels: {
        imessage: {
          groupPolicy: "allowlist",
          groupAllowFrom: ["+15555550123"],
          groups: {
            "*": { systemPrompt: "使用英式拼写。" },
            "8421": {
              requireMention: true,
              systemPrompt: "这是值班轮换聊天。回复不超过 3 句话。",
            },
            "9907": {
              // 显式抑制：此处不应用通配符“使用英式拼写。”
              systemPrompt: "",
            },
          },
        },
      },
    }
    ```

    逐群组提示词仅适用于群组消息——私信不受影响。

  </Tab>

  <Tab title="会话和确定性回复">
    - 私信使用直接路由；群组使用群组路由。
    - 使用默认的 `session.dmScope=main` 时，iMessage 私信会归并到智能体主会话中。
    - 群组会话相互隔离（`agent:<agentId>:imessage:group:<chat_id>`）。
    - 回复会使用来源渠道/目标元数据路由回 iMessage。

    类群组线程行为：

    一些多人 iMessage 线程可能会携带 `is_group=false` 到达。
    如果该 `chat_id` 已在 `channels.imessage.groups` 下显式配置，OpenClaw 会将其视为群组流量（群组门控 + 群组会话隔离）。

  </Tab>
</Tabs>

## ACP 对话绑定

iMessage 聊天可以绑定到 ACP 会话。

快速操作流程：

- 在私信或允许的群聊中运行 `/acp spawn codex --bind here`。
- 同一 iMessage 对话中的后续消息将路由到新生成的 ACP 会话。
- `/new` 和 `/reset` 会就地重置同一个已绑定 ACP 会话。
- `/acp close` 会关闭 ACP 会话并移除绑定。

配置的持久绑定使用顶层 `bindings[]` 条目，并包含 `type: "acp"` 和 `match.channel: "imessage"`。

`match.peer.id` 可以使用：

- 规范化的私信句柄，例如 `+15555550123` 或 `user@example.com`
- `chat_id:<id>`（建议用于稳定的群组绑定）
- `chat_guid:<guid>`
- `chat_identifier:<identifier>`

示例：

```json5
{
  agents: {
    list: [
      {
        id: "codex",
        runtime: {
          type: "acp",
          acp: { agent: "codex", backend: "acpx", mode: "persistent" },
        },
      },
    ],
  },
  bindings: [
    {
      type: "acp",
      agentId: "codex",
      match: {
        channel: "imessage",
        accountId: "default",
        peer: { kind: "group", id: "chat_id:123" },
      },
      acp: { label: "codex-group" },
    },
  ],
}
```

有关共享 ACP 绑定行为，请参阅 [ACP 智能体](/zh-CN/tools/acp-agents)。

## 部署模式

<AccordionGroup>
  <Accordion title="专用机器人 macOS 用户（独立 iMessage 身份）">
    使用专用 Apple ID 和 macOS 用户，使机器人流量与个人 Messages 配置文件相隔离。

    典型流程：

    1. 创建/登录一个专用的 macOS 用户。
    2. 在该用户中，使用机器人的 Apple ID 登录 Messages。
    3. 在该用户中安装 `imsg`。
    4. 创建一个 SSH 包装脚本，使 OpenClaw 能在该用户上下文中运行 `imsg`。
    5. 将 `channels.imessage.accounts.<id>.cliPath` 和 `.dbPath` 指向该用户配置文件。

    首次运行时，可能需要在该机器人用户会话中通过 GUI 授予权限（Automation + Full Disk Access）。

  </Accordion>

  <Accordion title="通过 Tailscale 连接远程 Mac（示例）">
    常见拓扑：

    - Gateway 网关运行在 Linux/VM 上
    - iMessage + `imsg` 运行在 tailnet 中的一台 Mac 上
    - `cliPath` 包装脚本使用 SSH 运行 `imsg`
    - `remoteHost` 启用通过 SCP 获取附件

    示例：

    ```json5
    {
      channels: {
        imessage: {
          enabled: true,
          cliPath: "~/.openclaw/scripts/imsg-ssh",
          remoteHost: "bot@mac-mini.tailnet-1234.ts.net",
          includeAttachments: true,
          dbPath: "/Users/bot/Library/Messages/chat.db",
        },
      },
    }
    ```

    ```bash
    #!/usr/bin/env bash
    exec ssh -T bot@mac-mini.tailnet-1234.ts.net imsg "$@"
    ```

    使用 SSH 密钥，使 SSH 和 SCP 均可非交互式运行。
    首先确保主机密钥已受信任（例如 `ssh bot@mac-mini.tailnet-1234.ts.net`），以便填充 `known_hosts`。

  </Accordion>

  <Accordion title="多账户模式">
    iMessage 支持在 `channels.imessage.accounts` 下进行按账户配置。

    每个账户都可以覆盖 `cliPath`、`dbPath`、`allowFrom`、`groupPolicy`、`mediaMaxMb`、历史记录设置和附件根目录允许列表等字段。

  </Accordion>

  <Accordion title="私信历史记录">
    设置 `channels.imessage.dmHistoryLimit`，使用该对话最近解码的 `imsg` 历史记录为新的私信会话提供初始上下文。使用 `channels.imessage.dms["<sender>"].historyLimit` 进行按发送者覆盖，包括使用 `0` 为某个发送者禁用历史记录。

    iMessage 私信历史记录会按需从 `imsg` 获取。不设置 `dmHistoryLimit` 会禁用全局私信历史记录的初始填充，但正数的按发送者 `channels.imessage.dms["<sender>"].historyLimit` 仍会为该发送者启用初始填充。

  </Accordion>
</AccordionGroup>

## 媒体、分块和投递目标

<AccordionGroup>
  <Accordion title="附件和媒体">
    - 入站附件摄取**默认关闭**——设置 `channels.imessage.includeAttachments: true`，将照片、语音备忘录、视频和其他附件转发给智能体。禁用此项后，仅包含附件的 iMessage 会在到达智能体之前被丢弃，并且可能完全不会生成 `Inbound message` 日志行。
    - 设置 `remoteHost` 后，可通过 SCP 获取远程附件路径
    - 附件路径必须匹配允许的根目录：
      - `channels.imessage.attachmentRoots`（本地）
      - `channels.imessage.remoteAttachmentRoots`（远程 SCP 模式）
      - 配置的根目录会扩展默认根目录模式 `/Users/*/Library/Messages/Attachments`（合并，而非替换）
    - SCP 使用严格主机密钥检查（`StrictHostKeyChecking=yes`）
    - 出站媒体大小使用 `channels.imessage.mediaMaxMb`（默认 16 MB）

  </Accordion>

  <Accordion title="出站文本和分块">
    - 文本分块限制：`channels.imessage.textChunkLimit`（默认 4000）
    - 分块模式：`channels.imessage.streaming.chunkMode`
      - `length`（默认）
      - `newline`（优先按段落拆分）
    - 出站 Markdown 粗体/斜体/下划线/删除线会转换为原生样式文本（macOS 15+ 接收方会显示相应样式；旧版本接收方看到的是不含标记的纯文本）；Markdown 表格会根据该渠道的 Markdown 表格模式进行转换
    - `channels.imessage.sendTransport`（默认 `auto`、`bridge`、`applescript`）选择 `imsg` 如何执行发送

  </Accordion>

  <Accordion title="寻址格式">
    首选的显式目标：

    - `chat_id:123`（建议用于稳定路由）
    - `chat_guid:...`
    - `chat_identifier:...`

    也支持句柄目标：

    - `imessage:+1555...`
    - `sms:+1555...`
    - `user@example.com`

    ```bash
    imsg chats --limit 20
    ```

  </Accordion>
</AccordionGroup>

## 私有 API 操作

当 `imsg launch` 正在运行，且 `openclaw channels status --probe` 报告 `privateApi.available: true` 时，消息工具除发送普通文本外，还可以使用 iMessage 原生操作。

所有操作默认启用；使用 `channels.imessage.actions` 可关闭单项操作：

```json5
{
  channels: {
    imessage: {
      actions: {
        reactions: true,
        edit: true,
        unsend: true,
        reply: true,
        sendWithEffect: true,
        sendAttachment: true,
        renameGroup: true,
        setGroupIcon: true,
        addParticipant: true,
        removeParticipant: true,
        leaveGroup: true,
        polls: true,
      },
    },
  },
}
```

<AccordionGroup>
  <Accordion title="可用操作">
    - **react**：添加/移除 iMessage 点按回应（`messageId`、`emoji`、`remove`）。支持的点按回应分别映射为爱心、赞、踩、笑、强调和疑问。不指定表情符号进行移除时，会清除当前设置的任何点按回应。
    - **reply**：向现有消息发送线程式回复（`messageId`、`text` 或 `message`，以及 `chatGuid`、`chatId`、`chatIdentifier` 或 `to`）。带附件回复还需要一个 `imsg` 构建版本，且其 `send-rich` 支持 `--file`。
    - **sendWithEffect**：使用 iMessage 效果发送文本（`text` 或 `message`、`effect` 或 `effectId`）。短名称：slam、loud、gentle、invisibleink、confetti、lasers、fireworks、balloon、heart、echo、happybirthday、shootingstar、sparkles、spotlight。
    - **edit**：在受支持的 macOS/私有 API 版本上编辑已发送的消息（`messageId`、`text` 或 `newText`）。只有 Gateway 网关自身发送的消息才能编辑。
    - **unsend**：在受支持的 macOS/私有 API 版本上撤回已发送的消息（`messageId`）。只有 Gateway 网关自身发送的消息才能撤回。
    - **upload-file**：发送媒体/文件（`buffer` 采用 base64，或已填充的 `media`/`path`/`filePath`、`filename`，可选 `asVoice`）。旧版别名：`sendAttachment`。
    - **renameGroup**、**setGroupIcon**、**addParticipant**、**removeParticipant**、**leaveGroup**：当前目标为群组对话时，用于管理群聊。这些操作会修改主机的 Messages 身份，因此需要所有者发送者或 `operator.admin` Gateway 网关客户端。
    - **poll**：创建原生 Apple Messages 投票（`pollQuestion`、重复 2 至 12 次的 `pollOption`，以及 `chatGuid`、`chatId`、`chatIdentifier` 或 `to`）。使用 iOS/iPadOS/macOS 26+ 的接收方可以原生查看并投票；旧版操作系统会收到“已发送投票”的文本回退消息。需要 `selectors.pollPayloadMessage`。
    - **poll-vote**：对现有投票进行投票（`pollId` 或 `messageId`，并且必须恰好指定 `pollOptionIndex`、`pollOptionId` 或 `pollOptionText` 之一）。需要 `selectors.pollVoteMessage` 和 `poll.vote` RPC 方法。

    已接受的入站投票会呈现给智能体，其中包含问题、带编号的选项标签、票数，以及 `poll-vote` 所需的投票消息 ID。

  </Accordion>

  <Accordion title="消息 ID">
    入站 iMessage 上下文会包含短 `MessageSid` 值，并在可用时包含完整消息 GUID（`MessageSidFull`）。短 ID 的作用域限于近期由 SQLite 支持的回复缓存，并且使用前会根据当前聊天进行检查。如果短 ID 已过期，请在以提供该 ID 的对话为目标时，使用其 `MessageSidFull` 重试。完整 ID 不会绕过对话或账户绑定，因此应将来自其他聊天的 ID 替换为来自当前目标的 ID。如果缺少当前对话的证据，远程委托调用可能会拒绝过期的完整 ID。

  </Accordion>

  <Accordion title="能力检测">
    只有当缓存的探测状态表明桥接不可用时，OpenClaw 才会隐藏私有 API 操作。如果状态未知，操作仍然可见，并在分派时延迟执行探测，因此在 `imsg launch` 之后，首次操作无需单独手动刷新状态即可成功。

  </Accordion>

  <Accordion title="已读回执和正在输入状态">
    私有 API 桥接启动后，已接受的入站聊天会被标记为已读；私聊在轮次被接受后会立即显示正在输入气泡，同时智能体准备上下文并生成内容。使用以下配置禁用已读标记：

    ```json5
    {
      channels: {
        imessage: {
          sendReadReceipts: false,
        },
      },
    }
    ```

    在按方法能力列表出现之前的旧版 `imsg` 构建会静默关闭正在输入状态/已读功能；OpenClaw 每次重启会记录一次警告，以便确定回执缺失的原因。

  </Accordion>

  <Accordion title="入站点按回应">
    OpenClaw 会订阅 iMessage 点按回应，并将已接受的表情回应作为系统事件路由，而不是作为普通消息文本处理，因此用户的点按回应不会触发常规回复循环。

    通知模式由 `channels.imessage.reactionNotifications` 控制：

    - `"own"`（默认）：仅当用户回应机器人发送的消息时通知。
    - `"all"`：对已授权发送者的所有入站点按回应发出通知。
    - `"off"`：忽略入站点按回应。

    按账户覆盖使用 `channels.imessage.accounts.<id>.reactionNotifications`。

  </Accordion>

  <Accordion title="审批表情回应（👍 / 👎）">
    当 `approvals.exec.enabled` 或 `approvals.plugin.enabled` 为 true，并且请求路由到 iMessage 时，Gateway 网关会以原生方式投递审批提示，并接受点按回应来处理该请求：

    - `👍`（赞点按回应）→ `allow-once`
    - `👎`（踩点按回应）→ `deny`
    - `allow-always` 仍作为手动回退：将 `/approve <id> allow-always` 作为普通回复发送。

    表情回应处理要求作出回应的用户句柄必须是显式审批者。审批者列表从 `channels.imessage.allowFrom`（或 `channels.imessage.accounts.<id>.allowFrom`）读取；添加 E.164 格式的用户电话号码或其 Apple ID 电子邮件地址（`chat_id:*` 等聊天目标不是有效的审批者条目）。通配符条目 `"*"` 会生效，但会允许任何发送者进行审批；空审批者列表会完全禁用表情回应快捷方式。表情回应快捷方式会特意绕过 `reactionNotifications`、`dmPolicy` 和 `groupAllowFrom`，因为显式审批者允许列表是解决审批请求时唯一有效的门控条件。

    `/approve` 文本命令授权遵循同一列表：当 `channels.imessage.allowFrom` 非空时，`/approve <id> <decision>` 会根据该审批者列表进行授权（而不是范围更广的私信允许列表），私信允许列表中获准但未包含在 `allowFrom` 中的发送者会收到明确的拒绝。当 `allowFrom` 为空时，同一聊天回退仍然有效，并且 `/approve` 会授权私信允许列表准许的任何人。将所有应当能够审批的操作员——无论是通过 `/approve` 还是通过表情回应——都添加到 `allowFrom`。

    操作员说明：
    - 表情回应绑定会同时存储在内存和 Gateway 网关的持久化键值存储中（TTL 与审批过期时间一致），Gateway 网关还会轮询待处理提示中的点回表情，因此即使点回表情在 Gateway 网关重启后不久到达，仍可完成审批。
    - 当操作员自己的 `is_from_me=true` 点回表情（例如来自已配对的 Apple 设备）所对应的句柄是明确指定的审批者时，该点回表情会完成审批。
    - 仅当配置了明确的审批者时，审批提示才会路由到群组会话中；否则任何群组成员都可以批准。
    - 旧版文本样式的点回表情（来自非常旧的 Apple 客户端的 `Liked "…"` 纯文本）无法完成审批，因为它们不携带消息 GUID；表情回应解析需要当前 macOS / iOS 客户端发出的结构化点回表情元数据。

  </Accordion>

  <Accordion title="问题表情回应（1️⃣ / 2️⃣ / 3️⃣ / 4️⃣）">
    对于包含一个非敏感单选问题及一到四个选项的 `ask_user` 提示，OpenClaw 会添加带编号的表情符号选项。使用匹配的数字对已送达的提示作出表情回应即可回答。该表情回应必须携带由 Bot 编写的消息的稳定 GUID；随后 OpenClaw 会通过 Gateway 网关将该数字映射到规范选项。过期或重复的点击会被忽略。

    多问题、多选和自由文本提示仍然只能通过文本回复。问题表情回应遵循常规的 iMessage 私信/群组准入规则。即使常规 `reactionNotifications` 为 `"off"`，也能识别这些表情回应，而不会将无关的表情回应转换为智能体事件。

  </Accordion>
</AccordionGroup>

## 配置写入

默认情况下，iMessage 允许由渠道发起配置写入（适用于 `commands.config: true` 时的 `/config set|unset`）。

禁用：

```json5
{
  channels: {
    imessage: {
      configWrites: false,
    },
  },
}
```

<a id="coalescing-split-send-dms-command--url-in-one-composition"></a>

## 合并拆分发送的私信（同一条编辑内容中的命令 + URL）

Apple 可能会将命令及其 URL 预览存储为单独的物理 `chat.db` 行。`imsg` 0.13.1 及更高版本会在监视、历史记录或搜索返回消息前合并这些行，因此 OpenClaw 会收到一条逻辑入站消息，而无需增加渠道特定的私信延迟。

无需设置 iMessage 合并选项。已停用的 `channels.imessage.coalesceSameSenderDms` 键由 `openclaw doctor --fix` 移除。如果你有意希望批量处理某个渠道中快速连续发送的文本消息，仍可使用通用 `messages.inbound` 防抖。

如果命令加 URL 的发送内容作为单独的智能体轮次到达，请在 Messages Mac 上更新 `imsg`：

```bash
brew update && brew upgrade imsg
```

## 桥接器或 Gateway 网关重启后的入站恢复

iMessage 会恢复 Gateway 网关停机期间错过的消息，同时抑制 Apple 在 Push 恢复后可能刷出的陈旧“积压消息炸弹”。默认行为始终启用，基于持久化入口和时间范围防线构建。

- **持久化重放保护。** 在推进恢复游标之前，OpenClaw 会将每个原始行记录到共享 SQLite 入口队列中，并使用其 Apple GUID 作为事件 ID。已完成的行会留下约 4 小时的墓碑记录，上限为 10,000 个条目，因此即使重启后，具有相同 GUID 的重放也会被丢弃。待处理行会一直保持可恢复状态，直到分发接管它。
- **停机恢复。** 启动时，监视器会记住最后一个持久化准入的 `chat.db` 行 ID（持久化的每账户游标），并将其作为 `since_rowid` 传递给 `imsg watch.subscribe`，因此 imsg 会重放尚未记录的行，然后跟踪实时消息。在崩溃前已记录的行会从 SQLite 恢复。重放范围限制为最近 500 行以及最多约 2 小时前的消息，GUID 墓碑记录会丢弃所有已处理的内容。
- **陈旧积压消息时间范围防线。** 启动边界以上的行是真正的实时消息；如果某行的发送日期比其到达时间早约 15 分钟以上，则它属于 Push 刷出的积压消息，会被抑制。重放的行（位于边界或边界以下）改用更宽的恢复窗口，因此最近错过的消息会得到送达，而久远的历史消息不会。

本地和远程 `cliPath` 设置均支持恢复，因为 `since_rowid` 重放通过同一条 `imsg` RPC 连接运行。两者的区别在于窗口：当 Gateway 网关可以读取 `chat.db`（本地）时，它会锚定启动行 ID 边界、限制重放跨度，并送达最多约数小时前错过的消息。通过远程 SSH `cliPath` 时，它无法读取数据库，因此重放不设上限，并且每一行都使用实时消息时间范围防线——它仍会恢复最近错过的消息并抑制旧积压消息，只是实时窗口较窄。要获得更宽的恢复窗口，请在 Messages Mac 上运行 Gateway 网关。

### 操作员可见信号

被抑制的积压消息会以默认级别记录，绝不会静默丢弃（`recovery` 标志指明应用了哪个窗口）：

```text
imessage：已抑制陈旧的入站积压消息 account=<id> sent=<iso> recovery=<bool>（启动后已抑制 <N> 条）
```

### 迁移

`channels.imessage.catchup.*` 已弃用——停机恢复会自动进行，新设置无需配置。包含 `catchup.enabled: true` 的现有配置仍会作为恢复重放窗口的兼容性配置文件受到支持。已禁用的追赶块（`enabled: false` 或缺少 `enabled: true`）已停用；`openclaw doctor --fix` 会移除这些块。

## 故障排查

<AccordionGroup>
  <Accordion title="找不到 imsg 或不支持 RPC">
    验证二进制文件和 RPC 支持：

    ```bash
    imsg rpc --help
    imsg status --json
    openclaw channels status --probe
    ```

    如果探测报告不支持 RPC，请更新 `imsg`。如果私有 API 操作不可用，请在已登录的 macOS 用户会话中运行 `imsg launch`，然后再次探测。如果 Gateway 网关未在 macOS 上运行，请使用上文的通过 SSH 连接远程 Mac 的设置，而不是默认的本地 `imsg` 路径。

  </Accordion>

  <Accordion title="可以发送 Messages，但收不到入站 iMessage">
    首先确认消息是否已到达本地 Mac。如果 `chat.db` 没有变化，那么即使 `imsg status --json` 报告桥接器健康，OpenClaw 也无法接收消息。

```bash
imsg chats --limit 10 --json
imsg watch --chat-id <chat-id> --json
sqlite3 ~/Library/Messages/chat.db \
  "select datetime(max(date)/1000000000 + 978307200, 'unixepoch', 'localtime'), max(ROWID) from message;"
```

    如果从手机发送的消息未创建新行，请先修复 macOS Messages 和 Apple Push 层，再更改 OpenClaw 配置。执行一次服务刷新通常就足够：

```bash
launchctl kickstart -k system/com.apple.apsd
launchctl kickstart -k gui/$(id -u)/com.apple.CommCenter
launchctl kickstart -k gui/$(id -u)/com.apple.identityservicesd
launchctl kickstart -k gui/$(id -u)/com.apple.imagent
imsg launch
openclaw gateway restart
```

    从手机发送一条新的 iMessage，并确认出现新的 `chat.db` 行或 `imsg watch` 事件，然后再调试 OpenClaw 会话。不要将此操作作为定期重新启动桥接器的循环；在工作进行期间重复执行 `imsg launch` 并重启 Gateway 网关，可能会中断消息送达并使正在进行的渠道运行陷入停滞。

  </Accordion>

  <Accordion title="Gateway 网关未在 macOS 上运行">
    默认的 `cliPath: "imsg"` 必须在已登录 Messages 的 Mac 上运行。在 Linux 或 Windows 上，请将 `channels.imessage.cliPath` 设置为一个包装脚本，通过 SSH 连接到该 Mac 并运行 `imsg "$@"`。

```bash
#!/usr/bin/env bash
exec ssh -T messages-mac imsg "$@"
```

    然后运行：

```bash
openclaw channels status --probe --channel imessage
```

  </Accordion>

  <Accordion title="私信被忽略">
    检查：

    - `channels.imessage.dmPolicy`
    - `channels.imessage.allowFrom`
    - 配对审批（`openclaw pairing list imessage`）

  </Accordion>

  <Accordion title="群组消息被忽略">
    检查：

    - `channels.imessage.groupPolicy`
    - `channels.imessage.groupAllowFrom`
    - `channels.imessage.groups` 允许列表行为
    - 提及模式配置（`agents.entries.*.groupChat.mentionPatterns`）

  </Accordion>

  <Accordion title="远程附件失败">
    检查：

    - `channels.imessage.remoteHost`
    - `channels.imessage.remoteAttachmentRoots`
    - 来自 Gateway 网关主机的 SSH/SCP 密钥身份验证
    - Gateway 网关主机上的 `~/.ssh/known_hosts` 中存在主机密钥
    - 运行 Messages 的 Mac 上的远程路径可读性

  </Accordion>

  <Accordion title="错过了 macOS 权限提示">
    在同一用户/会话上下文的交互式 GUI 终端中重新运行，并批准提示：

    ```bash
    imsg chats --limit 1
    imsg send <handle> "test"
    ```

    确认运行 OpenClaw/`imsg` 的进程上下文已获得完全磁盘访问权限和自动化权限。

  </Accordion>
</AccordionGroup>

## 配置参考链接

- [Configuration reference - iMessage](/zh-CN/gateway/config-channels#imessage)
- [Gateway 配置](/zh-CN/gateway/configuration)
- [配对](/zh-CN/channels/pairing)

## 相关内容

- [渠道概览](/zh-CN/channels) — 所有支持的渠道
- [BlueBubbles removal and the imsg iMessage path](/zh-CN/announcements/bluebubbles-imessage) — 公告和迁移摘要
- [Coming from BlueBubbles](/zh-CN/channels/imessage-from-bluebubbles) — 配置转换表和分步切换指南
- [配对](/zh-CN/channels/pairing) — 私信身份验证和配对流程
- [群组](/zh-CN/channels/groups) — 群聊行为和提及门控
- [频道路由](/zh-CN/channels/channel-routing) — 消息的会话路由
- [安全性](/zh-CN/gateway/security) — 访问模型和安全加固
