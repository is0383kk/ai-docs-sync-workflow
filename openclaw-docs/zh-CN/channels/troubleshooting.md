---
read_when:
    - 渠道传输显示已连接，但回复失败
    - 在深入阅读提供商文档之前，你需要先进行特定于渠道的检查
summary: 按渠道快速排查故障，提供各渠道的故障特征和修复方法
title: 渠道故障排查
x-i18n:
    generated_at: "2026-07-26T06:42:04Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 3891595e4b5aca9de7997a6e908fa1c9246579032bfdfa1656a6992d644c3ecc
    source_path: channels/troubleshooting.md
    workflow: 16
---

当渠道已连接但行为异常时，请使用此页面。

## 命令执行顺序

首先按顺序运行以下命令：

```bash
openclaw status
openclaw gateway status
openclaw logs --follow
openclaw doctor
openclaw channels status --probe
```

健康基线：

- `Runtime: running`
- `Connectivity probe: ok`
- `Capability: read-only`、`write-capable` 或 `admin-capable`
- 渠道探测显示传输已连接，并在支持的情况下显示 `works` 或 `audit ok`

## 更新后

当 Telegram、iMessage、BlueBubbles 时代的配置或其他插件渠道在更新后消失时，
请使用以下命令。

```bash
openclaw status --all
openclaw doctor --fix
openclaw gateway restart
openclaw status --all
```

在 `openclaw
status --all` 中查找 `plugin load failed: dependency tree corrupted; run openclaw doctor --fix`。这表示渠道已配置，但插件设置/加载遇到了损坏的
依赖树，因而未能注册渠道。`openclaw doctor --fix` 会清除过期的
插件运行时依赖符号链接和过期的身份验证影子，然后 `openclaw gateway restart` 会重新加载
干净状态。

## WhatsApp

### WhatsApp 故障特征

| 症状                                | 最快检查方式                                          | 修复方法                                                                                                                                |
| ----------------------------------- | --------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------- |
| 已连接但不回复私信                  | `openclaw pairing list whatsapp`                    | 批准发送者，或切换私信策略/允许列表。                                                                                    |
| 忽略群组消息                        | 检查配置中的 `requireMention` 和提及模式 | 提及机器人，或放宽该群组的提及策略。                                                                          |
| 二维码登录超时并返回 408            | 检查 Gateway 网关的 `HTTPS_PROXY` / `HTTP_PROXY` 环境变量      | 设置可访问的代理；仅在绕过限制时使用 `NO_PROXY`。                                                                         |
| 随机断开连接/重新登录循环           | `openclaw channels status --probe` 和日志           | 即使当前已连接，也会标记近期的重新连接；监视日志、重启 Gateway 网关，如果连接仍反复波动，则重新关联。 |
| `status=408 Request Time-out` 循环  | 依次执行探测、查看日志、运行 Doctor，然后检查 Gateway 网关状态            | 先修复主机连接/时序问题；如果循环仍然存在，请备份身份验证数据并重新关联账户。                                   |
| 回复延迟数秒/数分钟                 | `openclaw doctor --fix`                             | 当确认过期的本地 TUI 客户端正在降低 Gateway 网关事件循环性能时，Doctor 会将其停止。                                    |

完整故障排查：[WhatsApp 故障排查](/zh-CN/channels/whatsapp#troubleshooting)

## Telegram

### Telegram 故障特征

| 症状                                 | 最快检查方式                                       | 修复方法                                                                                                                       |
| ------------------------------------ | ------------------------------------------------ | ---------------------------------------------------------------------------------------------------------------------- |
| `/start` 但没有可用的回复流程    | `openclaw pairing list telegram`                 | 批准配对或更改私信策略。                                                                                   |
| 机器人在线但群组中保持静默           | 验证提及要求和机器人的隐私模式                   | 禁用隐私模式以获得群组可见性，或提及机器人。                                                              |
| 发送失败并出现网络错误               | 检查日志中的 Telegram API 调用失败记录           | 修复通往 `api.telegram.org` 的 DNS/IPv6/代理路由。                                                                      |
| 启动时报告 `getMe returned 401` | 检查配置的令牌来源                               | 重新复制或生成 BotFather 令牌，并更新 `botToken`、`tokenFile` 或默认账户的 `TELEGRAM_BOT_TOKEN`。 |
| 轮询停滞或重新连接缓慢               | 查看 `openclaw logs --follow` 中的轮询诊断信息 | 升级；持续停滞通常表示代理/DNS/IPv6 存在问题。                                                            |
| 启动时拒绝 `setMyCommands`  | 检查日志中的 `BOT_COMMANDS_TOO_MUCH`         | 减少插件、Skills 或自定义 Telegram 命令，或者禁用原生菜单。                                                  |
| 升级后允许列表将你拦截               | `openclaw security audit` 和配置允许列表  | 运行 `openclaw doctor --fix`，或将 `@username` 替换为数字发送者 ID。                                            |

完整故障排查：[Telegram 故障排查](/zh-CN/channels/telegram#troubleshooting)

## Discord

### Discord 故障特征

| 症状                                      | 最快检查方式                                                                                                                   | 修复方法                                                                                                                                                                                                                                                                      |
| ----------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 机器人在线但不回复服务器消息              | `openclaw channels status --probe`                                                                                           | 允许该服务器/频道，并验证消息内容 intent。                                                                                                                                                                                                                |
| 忽略群组消息                              | 检查日志中因提及门控而丢弃消息的记录                                                                                          | 提及机器人，或设置服务器/频道的 `requireMention: false`。                                                                                                                                                                                                             |
| 有输入状态/令牌用量但没有 Discord 消息    | 检查这是环境房间事件，还是模型遗漏 `message(action=send)` 的已选择加入 `message_tool` 房间 | 检查 Gateway 网关详细日志中被抑制的最终负载元数据，验证 `messages.groupChat.unmentionedInbound`，阅读[环境房间事件](/zh-CN/channels/ambient-room-events)，或为普通群组请求保留 `messages.groupChat.visibleReplies: "automatic"`。 |
| 缺少私信回复                              | `openclaw pairing list discord`                                                                                              | 批准私信配对或调整私信策略。                                                                                                                                                                                                                               |

完整故障排查：[Discord 故障排查](/zh-CN/channels/discord#troubleshooting)

## Slack

### Slack 故障特征

| 症状                                   | 最快检查方式                                | 修复方法                                                                                                                                                     |
| -------------------------------------- | ----------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------- |
| Socket 模式已连接但无响应              | `openclaw channels status --probe`        | 验证应用令牌、机器人令牌和所需权限范围；在由 SecretRef 支持的设置中留意 `botTokenStatus` / `appTokenStatus = configured_unavailable`。 |
| 私信被阻止                             | `openclaw pairing list slack`             | 批准配对或放宽私信策略。                                                                                                                  |
| 频道消息被忽略                         | 检查 `groupPolicy` 和频道允许列表 | 允许该频道，或将策略切换为 `open`。                                                                                                        |

完整故障排查：[Slack 故障排查](/zh-CN/channels/slack#troubleshooting)

## iMessage

### iMessage 故障特征

| 症状                                  | 最快检查方式                                              | 修复方法                                                                      |
| ------------------------------------ | ------------------------------------------------------- | --------------------------------------------------------------------- |
| 非 macOS 上缺少 `imsg` 或运行失败 | `openclaw channels status --probe --channel imessage`   | 在运行“信息”应用的 Mac 上运行 OpenClaw，或为 `cliPath` 使用 SSH 包装器。 |
| 在 macOS 上可以发送但无法接收         | 检查“信息”自动化的 macOS 隐私权限                        | 重新授予 TCC 权限并重启渠道进程。                 |
| 私信发送者被阻止                      | `openclaw pairing list imessage`                        | 批准配对或更新允许列表。                                  |

完整故障排查：[iMessage 故障排查](/zh-CN/channels/imessage#troubleshooting)

## Signal

### Signal 故障特征

| 症状                            | 最快检查方式                                 | 修复方法                                                         |
| ------------------------------- | ------------------------------------------ | -------------------------------------------------------- |
| 守护进程可访问但机器人保持静默  | `openclaw channels status --probe`         | 验证 `signal-cli` 守护进程 URL/账户和接收模式。 |
| 私信被阻止                      | `openclaw pairing list signal`             | 批准发送者或调整私信策略。                      |
| 群组回复未触发                  | 检查群组允许列表和提及模式                 | 添加发送者/群组或放宽门控。                       |

完整故障排查：[Signal 故障排查](/zh-CN/channels/signal#troubleshooting)

## QQ Bot

### QQ Bot 故障特征

| 症状                              | 最快检查方式                                  | 修复方法                                                                |
| ------------------------------- | ------------------------------------------- | --------------------------------------------------------------- |
| 机器人回复“去了火星”             | 验证配置中的 `appId` 和 `clientSecret` | 设置凭据或重启 Gateway 网关。                         |
| 没有入站消息                     | `openclaw channels status --probe`          | 在 QQ 开放平台上验证凭据。                     |
| 语音未转录                       | 检查 STT 提供商配置                         | 配置 `channels.qqbot.stt` 或 `tools.media.audio`。          |
| 主动消息未送达                   | 检查 QQ 平台的交互要求                      | 如果近期没有交互，QQ 可能会阻止机器人发起的消息。 |

完整故障排查：[QQ Bot 故障排查](/zh-CN/channels/qqbot#troubleshooting)

## Matrix

### Matrix 故障特征

| 症状                             | 最快检查方式                          | 修复方法                                                                       |
| ----------------------------------- | -------------------------------------- | ------------------------------------------------------------------------- |
| 已登录但忽略房间消息 | `openclaw channels status --probe`     | 检查 `groupPolicy`、房间允许列表和提及门控。                  |
| 私信未被处理                  | `openclaw pairing list matrix`         | 批准发送者或调整私信策略。                                       |
| 加密房间失败                | `openclaw matrix verify status`        | 重新验证设备，然后检查 `openclaw matrix verify backup status`。  |
| 备份恢复处于待处理状态或已损坏    | `openclaw matrix verify backup status` | 运行 `openclaw matrix verify backup restore`，或使用恢复密钥重新运行。 |
| 交叉签名/引导看起来不正确 | `openclaw matrix verify bootstrap`     | 一次性修复机密存储、交叉签名和备份状态。       |

完整设置和配置：[Matrix](/zh-CN/channels/matrix)

## 相关内容

- [配对](/zh-CN/channels/pairing)
- [渠道路由](/zh-CN/channels/channel-routing)
- [Gateway 网关故障排查](/zh-CN/gateway/troubleshooting)
