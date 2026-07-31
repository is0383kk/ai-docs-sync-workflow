---
read_when:
    - 设置或排查 Discord Activity 小组件故障
summary: 在 Discord Activities 中启动独立运行的 OpenClaw HTML 小组件
title: Discord 活动
x-i18n:
    generated_at: "2026-07-26T06:08:17Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: b1bc04443aef89fd514290c3bebdbdd3e9972298b45cae3806bec99344f6d8cd
    source_path: channels/discord-activities.md
    workflow: 16
---

Discord Activities 允许智能体向当前 Discord 频道发布一个交互式、自包含的 HTML 小组件。消息中包含一个 **Open widget** 按钮；点击后会在 Discord 内启动该小组件。

此功能默认关闭。仅当存在 `channels.discord.activities` 且能够解析到客户端密钥时，OpenClaw 才会注册 Activity HTTP 路由、`show_widget` 智能体工具和启动按钮处理程序。已弃用的 `discord_widget` 别名将在一个发布版本内继续可用。

## 前置条件

- 一个现有的 [OpenClaw Discord Bot](/zh-CN/channels/discord)
- 一个可访问 OpenClaw Gateway 网关的公共 HTTPS 主机名
- 为 Bot 的 Discord 应用配置 Activities 和 OAuth2 的权限

任何 HTTPS 反向代理或隧道都可以使用。命名的 Cloudflare Tunnel 可以提供稳定的主机名，而无需直接暴露 Gateway 网关端口。

```yaml
# ~/.cloudflared/config.yml
tunnel: openclaw-discord
credentials-file: /home/you/.cloudflared/TUNNEL-ID.json
ingress:
  - hostname: openclaw.example.com
    service: http://127.0.0.1:18789
  - service: http_status:404
```

```bash
cloudflared tunnel login
cloudflared tunnel create openclaw-discord
cloudflared tunnel route dns openclaw-discord openclaw.example.com
cloudflared tunnel run openclaw-discord
```

保持正常的 Gateway 网关身份验证处于启用状态。只有 Activity 前缀是公开的，插件会自行验证 OAuth、Activity 实例成员身份、频道绑定、会话和一次性文档能力。

## 设置

<Steps>
  <Step title="通过 HTTPS 公开 Gateway 网关">
    启动隧道或反向代理，并在添加 Activities 配置后验证 `https://openclaw.example.com/discord/activity/` 能够访问 Gateway 网关。请将示例主机名替换为你自己的主机名。
  </Step>

  <Step title="在 Discord 中启用 Activities">
    在 [Discord Developer Portal](https://discord.com/developers/applications) 中打开现有的 Bot 应用。打开 **Activities**，启用 Activities，然后创建 URL 映射：

    - 前缀：`ROOT`（`/`）
    - 目标：`openclaw.example.com/discord/activity`

    目标是公共主机名加上 `/discord/activity`，末尾不带斜杠。

  </Step>

  <Step title="复制 OAuth2 客户端密钥">
    在 Developer Portal 中打开 **OAuth2**。Discord 要求至少配置一个重定向 URI，因此如果应用尚未配置，请添加一个本地占位地址，例如 loopback 地址；Embedded App SDK 会处理 Activity 返回流程。复制或重置应用客户端密钥。请将其视为凭据：不要将其粘贴到聊天、日志或已提交的配置文件中。
  </Step>

  <Step title="配置 OpenClaw">
    将以下配置块添加到需要提供小组件的 Discord 账户中：

    ```json5
    {
      channels: {
        discord: {
          token: "${DISCORD_BOT_TOKEN}",
          activities: {
            clientSecret: "${DISCORD_CLIENT_SECRET}",
            // 可选。默认使用启动时获取的 Bot 应用 ID。
            applicationId: "YOUR_DISCORD_APPLICATION_ID",
          },
        },
      },
    }
    ```

    设置 `DISCORD_CLIENT_SECRET` 后，可以从该配置块中省略 `clientSecret`。该配置块本身必须保留，以明确选择启用此功能。

    常规 Discord 访问设置仍然相互独立。例如，`allowFrom` 仍控制谁可以向智能体发送私信；它不控制谁可以打开已发布到频道中的小组件。

  </Step>

  <Step title="重启并测试">
    重启 Gateway 网关。在 Discord 对话中，让智能体显示一个交互式小组件。智能体会调用 `show_widget`；点击已发布消息中的 **Open widget**。
  </Step>
</Steps>

## 安全模型

- 在返回小组件元数据之前，OAuth 会识别 Discord 用户。
- Discord 的 Get Activity Instance API 必须确认 OAuth 用户存在于当前 Activity 实例中。实例频道必须与发布小组件的频道一致。
- Discord 允许进入该频道的所有人都可以打开其中的小组件。如需缩小受众范围，请使用 Discord 频道权限。OpenClaw 的命令和私信允许列表不会授予或移除对已发布频道内容的访问权限。
- OAuth 会话在 15 分钟后过期。小组件文档能力在 60 秒后过期，并且只能使用一次。
- 小组件在七天后过期，每个 Discord 插件实例最多保留 64 个小组件。
- 小组件 HTML 由你的智能体编写，应将其视为可信内容。不要嵌入你不希望因小组件缺陷而泄露的秘密。
- 小组件可以在其自己的嵌套框架内导航。`sandbox="allow-scripts"` iframe 会阻止顶层导航、弹出窗口和同源访问，其内容安全策略会阻止网络连接和外部资源。这些控制属于纵深防御措施，并不能构成防范小组件编写智能体的安全边界。
- 禁用 Activities 时，完全不会注册 `/discord/activity`。

启用后，公共 Activity 外壳和令牌交换路由可通过隧道访问。如果没有有效的 OAuth 会话和一次性文档能力，它们不会公开小组件 HTML。

## 故障排查

### Activity 显示“Gateway offline”

- 确认隧道正在运行，并路由到 Gateway 网关实际绑定的端口
- 确认 Developer Portal 中的目标包含 `/discord/activity`
- 更改 Discord 或 OpenClaw 配置后重启 Gateway 网关
- 检查 Gateway 网关日志中有关缺少 Activities 客户端密钥的单行警告

### Discord 打开空白页面或报告 `blocked:csp`

- 验证 URL 映射使用 `ROOT`，且未添加第二个 `/discord/activity` 路径段
- 确认外壳、`shell.js` 和 SDK 模块都通过 Discord 代理返回
- 检查 Gateway 网关日志中 `/discord/activity/` 下的请求

小组件的网络请求会被有意阻止。请内联小组件所需的所有 CSS、JavaScript、图像和数据。

### “Widget unavailable”

请从智能体发布小组件的频道中点击启动按钮。OpenClaw 会在按钮被点击时在服务器端记录启动，因此即使 Discord 省略或破坏了按钮的自定义 ID，最新的启动记录也能解析出确切的小组件。当自定义 ID 和启动记录都无法解析时，OpenClaw 会打开该频道中最近发布且仍有效的小组件。通过保留了自定义 ID 的按钮，仍然可以访问较早的小组件。

### “You cannot launch Activities in this channel”

Discord 不支持从论坛帖子主题中启动 Activities。OpenClaw 可以在其中发布小组件消息和按钮，但请改为从普通文本频道启动 Activity。此限制来自 Discord，而非 OpenClaw。
