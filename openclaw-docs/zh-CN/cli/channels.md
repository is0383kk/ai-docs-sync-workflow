---
read_when:
    - 你想要添加或移除渠道账号（Discord、Google Chat、iMessage、Matrix、Signal、Slack、Telegram、WhatsApp 等）
    - 你想检查渠道状态或持续查看渠道日志
    - 你需要检查或重新提交失败的入站渠道事件
summary: '`openclaw channels` 的 CLI 参考（账户、状态、死信、能力、解析、日志、登录/退出）'
title: 渠道
x-i18n:
    generated_at: "2026-07-26T06:43:23Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 8e5b7d674264af51d6fec34c8c95256129d66918b7c4515ac0f2c2bd311f2c3b
    source_path: cli/channels.md
    workflow: 16
---

# `openclaw channels`

在 Gateway 网关上管理聊天渠道账号及其运行时状态。

相关文档：

- 渠道指南：[渠道](/zh-CN/channels)
- Gateway 配置：[配置](/zh-CN/gateway/configuration)

## 常用命令

```bash
openclaw channels list
openclaw channels list --all
openclaw channels status
openclaw channels capabilities
openclaw channels capabilities --channel discord --target channel:123
openclaw channels resolve --channel slack "#general" "@jane"
openclaw channels logs --channel all
openclaw channels dead-letters list --channel telegram --account default
```

`channels list` 仅显示聊天渠道：默认显示已配置的账号，并为每个账号附带 `installed`、`configured` 和 `enabled` 状态标签（使用 `--json` 获取机器可读输出）。传入 `--all` 还可显示尚未配置账号的内置渠道，以及尚未安装到磁盘的可安装目录渠道。提供商身份验证和模型用量位于其他位置：使用 `openclaw models auth list` 管理提供商身份验证配置文件，使用 `openclaw status` 或 `openclaw models list` 查看用量/配额。

## 状态 / 能力 / 解析 / 日志

- `channels status`：`--channel <name>`、`--probe`、`--timeout <ms>`（默认 `10000`）、`--json`
- `channels capabilities`：`--channel <name>`、`--account <id>`（需要 `--channel`）、`--target <dest>`（需要 `--channel`）、`--timeout <ms>`（默认 `10000`，上限为 `30000`）、`--json`
- `channels resolve <entries...>`：`--channel <name>`、`--account <id>`、`--kind <auto|user|group>`（默认 `auto`）、`--json`
- `channels logs`：`--channel <name|all>`（默认 `all`）、`--lines <n>`（默认 `200`）、`--json`

`channels status --probe` 是实时路径：在 Gateway 网关可访问时，它会对每个账号运行
`probeAccount` 和可选的 `auditAccount` 检查，因此输出可包含传输状态
以及 `works`、`probe failed`、`audit ok` 或 `audit failed` 等探测结果。
如果 Gateway 网关无法访问，`channels status` 会回退到仅基于配置的摘要，
而不输出实时探测结果。

## 入站死信

耗尽重试策略的入站事件会在队列现有的失败条目保留期内留在共享状态数据库中。使用以下命令检查一个渠道账号：

```bash
openclaw channels dead-letters list --channel telegram --account default
openclaw channels dead-letters list --channel telegram --account default --json
```

文本视图显示事件 ID、失败原因、尝试次数和失败时长。JSON 输出还包含保留的有效载荷、元数据、通道和尝试时间戳，以供诊断。

修正根本问题后，使用事件的原始事件 ID 将其重新入队：

```bash
openclaw channels dead-letters resubmit <event-id> --channel telegram --account default
```

请在 Gateway 网关主机上运行这些命令，以便它们访问与渠道运行时相同的共享状态数据库。重新提交会保留有效载荷、元数据和通道，但会重置尝试计数器和队列时长。它会以原子方式替换该事件的失败标记，因此当事件处于待处理或已认领状态时重复运行该命令会被拒绝，而不会创建第二次分派。运行中的渠道会在下次入口排空时接收该事件。已完成的事件保持终态，无法重新提交。在添加有效载荷保留功能之前创建的失败行仍可能出现在列表中，但由于其有效载荷不可用，重新提交会被拒绝。

`openclaw health` 会报告每个渠道账号的死信数量和最早失败时长。`openclaw doctor` 会列出受影响的账号，并指向上述检查命令。

不要将 `openclaw sessions`、Gateway 网关 `sessions.list` 或智能体
`sessions_list` 工具用作渠道套接字健康状态的信号。这些界面报告的是
存储的对话行，而不是提供商运行时状态。Discord 提供商重启后，
已连接但没有活动的账号可能仍然健康，而在发生下一次入站或出站对话事件之前，
不会出现 Discord 会话行。

## 添加 / 移除账号

```bash
openclaw channels add --channel telegram --token <bot-token>
openclaw channels add --channel nostr --private-key "$NOSTR_PRIVATE_KEY"
openclaw channels remove --channel telegram --delete
```

<Tip>
`openclaw channels add telegram --help` 或 `openclaw channels add --channel telegram --help` 仅显示 Telegram 的设置标志。`openclaw channels add --help` 仅显示共享命令封装。
</Tip>

`channels remove` 仅对已安装/已配置的渠道插件生效。对于可安装的目录渠道，请先使用 `channels add`。不带 `--delete` 时，它会询问是否禁用账号并保留其配置；`--delete` 会直接移除配置条目而不提示。
对于由运行时支持的渠道插件，`channels remove` 还会在更新配置前请求运行中的 Gateway 网关停止所选账号，因此禁用或删除账号后，旧监听器不会一直保持活动状态直到重启。

共享控制封装仅包含 `--channel`、`--account` 和可选的账号显示 `--name`。每个现代渠道插件都拥有自己的凭据、传输方式和提供商特定语义。通过位置 ID 或 `--channel <id>` 选择渠道后，CLI 只会根据内置或已安装插件包的元数据构建该渠道的选项，而不会加载渠道运行时代码。

当现代契约处理 `--token`、`--url` 或 `--use-env` 等看似通用的标志时，它们仍归渠道所有。当所选第三方插件仍使用旧版共享设置适配器时，核心只会为该渠道注册已发布的兼容性标志集及其旧版 `cliAddOptions`。无关的旧版字段不会泄漏到其他渠道中，现代渠道也会拒绝其未声明的兼容性标志。

渠道自有标志示例包括：

| 渠道        | 标志                                                                                                 |
| ----------- | ---------------------------------------------------------------------------------------------------- |
| Google Chat | `--webhook-path`、`--webhook-url`、`--audience-type`、`--audience`                                   |
| iMessage    | `--cli-path`、`--db-path`、`--service`、`--region`                                                   |
| Matrix      | `--homeserver`、`--user-id`、`--access-token`、`--password`、`--device-name`、`--initial-sync-limit` |
| Nostr       | `--private-key`、`--relay-urls`                                                                      |
| Signal      | `--signal-number`、`--signal-transport`、`--cli-path`、`--http-url`、`--http-host`、`--http-port`    |
| Tlon        | `--ship`、`--url`、`--code`、`--group-channels`、`--dm-allowlist`、`--auto-discover-channels`        |
| WhatsApp    | `--auth-dir`                                                                                         |

如果通过标志驱动的添加命令需要安装渠道插件，OpenClaw 会使用该渠道的默认安装源，而不会打开交互式插件安装提示。

引导式设置和标志驱动的设置都会经过所选渠道的解析器、验证、账号解析、配置写入器和写入后钩子。不支持的标志会触发所属渠道的设置错误，而不会通过全局输入容器被接受。

当你运行 `openclaw channels add` 且未提供直接账号、凭据或渠道配置标志时，交互式向导可以发出提示。位置渠道 ID 和 `--channel <id>` 都可以预选该渠道，而不会绕过引导：

```bash
openclaw channels add telegram
openclaw channels add --channel telegram
```

向导可以提示输入：

- 每个所选渠道的账号 ID
- 这些账号的可选显示名称
- `Route these channel accounts to agents now?`

如果确认立即绑定，向导会询问应由哪个智能体拥有各个已配置的渠道账号，并写入账号范围的路由绑定。

之后也可以使用 `openclaw agents bindings`、`openclaw agents bind` 和 `openclaw agents unbind` 管理相同的路由规则（参见 [智能体](/zh-CN/cli/agents)）。

向仍使用单账号顶层设置的渠道添加非默认账号时，OpenClaw 会先将这些顶层值提升到渠道的账号映射中，再写入新账号。当渠道恰好只有一个现有命名账号，或 `defaultAccount` 指向某个账号时，提升过程会复用该账号；否则，这些值会写入 `channels.<channel>.accounts.default`。

路由行为保持一致：

- 现有的仅渠道绑定（无 `accountId`）会继续匹配默认账号。
- `channels add` 在非交互模式下不会自动创建或重写绑定。
- 交互式设置可以选择添加账号范围的绑定。

如果配置已经处于混合状态（同时存在命名账号和顶层单账号值），请运行 `openclaw doctor --fix`，将账号范围的值移入为该渠道选定的提升账号中。

## 登录和退出登录（交互式）

```bash
openclaw channels login --channel whatsapp
openclaw channels logout --channel whatsapp
```

- `channels login` 支持 `--account <id>` 和 `--verbose`；`channels logout` 支持 `--account <id>`。
- 当只有一个已配置渠道支持相应操作时，`channels login` 和 `logout` 可以推断渠道；如果有多个渠道，请传入 `--channel`。
- `channels logout` 会在 Gateway 网关可访问时优先使用实时 Gateway 网关路径，以便退出登录在清除渠道身份验证状态前停止所有活动监听器。如果本地 Gateway 网关不可访问，它会回退到本地身份验证清理；使用 `gateway.mode: "remote"` 时，Gateway 网关错误会使命令失败。
- 成功登录后，CLI 会请求可访问的本地 Gateway 网关启动该账号；在远程模式下，它会在本地保存身份验证信息，并注明远程运行时未重启。
- 请在 Gateway 网关主机的终端中运行 `channels login`。智能体 `exec` 会阻止此交互式登录流程；如果渠道提供原生智能体登录工具（例如 `whatsapp_login`），则应在聊天中使用这些工具。

## 故障排查

- 运行 `openclaw status --deep` 进行全面探测。
- 使用 `openclaw doctor` 获取引导式修复。
- 当 Gateway 网关无法访问时，`openclaw channels status` 会回退到仅基于配置的摘要。如果受支持的渠道凭据通过 SecretRef 配置，但在当前命令路径中不可用，它会将该账号报告为已配置并附带降级说明，而不会显示为未配置。

## 能力探测

获取提供商能力提示（可用时包括 intent/scope）以及静态功能支持：

```bash
openclaw channels capabilities
openclaw channels capabilities --channel discord --target channel:123
```

注意事项：

- `--channel` 是可选的；省略它可列出所有渠道（包括插件提供的渠道）。
- `--account` 仅可与 `--channel` 一起使用。
- `--target` 接受 `channel:<id>` 或原始数字渠道 ID，且仅适用于 Discord。对于 Discord 语音频道，权限检查会标记缺少的 `ViewChannel`、`Connect`、`Speak`、`SendMessages` 和 `ReadMessageHistory`。
- 探测因提供商而异：Discord Bot 身份和 intents，以及可选的渠道权限；Slack Bot 和用户权限范围；Telegram Bot 标志和 webhook；Signal 守护进程版本；Microsoft Teams 应用令牌和 Graph 角色/权限范围（已知时会标注）。没有探测功能的渠道会报告 `Probe: unavailable`。

## 将名称解析为 ID

使用提供商目录将渠道/用户名解析为 ID：

```bash
openclaw channels resolve --channel slack "#general" "@jane"
openclaw channels resolve --channel discord "My Server/#support" "@someone"
openclaw channels resolve --channel matrix "Project Room"
```

注意：

- 使用 `--kind user|group|auto` 强制指定目标类型。
- 当多个条目同名时，解析会优先选择活跃的匹配项。
- `channels resolve` 是只读的。如果所选账户通过 SecretRef 配置，但当前命令路径中无法使用该凭据，命令将返回带有说明的降级未解析结果，而不是中止整个运行。
- `channels resolve` 不会安装渠道插件。对于可从目录安装的渠道，请先使用 `channels add --channel <name>`，再解析名称。

## 相关内容

- [CLI 参考](/zh-CN/cli)
- [渠道概览](/zh-CN/channels)
