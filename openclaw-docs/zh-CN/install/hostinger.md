---
read_when:
    - 在 Hostinger 上设置 OpenClaw
    - 正在寻找适用于 OpenClaw 的托管 VPS
    - 使用 Hostinger 一键部署 OpenClaw
summary: 在 Hostinger 上托管 OpenClaw
title: Hostinger
x-i18n:
    generated_at: "2026-07-26T06:49:35Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 7dc49e741f8581928553e2426ed91f92df6e7b0c31dd8780c0d6e891a07be263
    source_path: install/hostinger.md
    workflow: 16
---

在 [Hostinger](https://www.hostinger.com/openclaw) 上运行持久化的 OpenClaw Gateway 网关，可选择 **1-Click** 托管部署，也可选择自行管理的 **VPS** 安装。

## 前提条件

- Hostinger 账户（[注册](https://www.hostinger.com/openclaw)）
- 大约 5-10 分钟

## 选项 A：1-Click OpenClaw

Hostinger 负责基础设施、Docker 和自动更新。这是最快启动并运行实例的方式。

<Steps>
  <Step title="购买并启动">
    1. 在 [Hostinger OpenClaw 页面](https://www.hostinger.com/openclaw)中选择 Managed OpenClaw 套餐并完成结账。

    <Note>
    结账时，你可以选择预先购买并立即集成到 OpenClaw 中的 **Ready-to-Use AI** 点数，无需其他提供商的外部账户或 API 密钥。你可以立即开始聊天。或者，也可以在设置期间提供自己的 Anthropic、OpenAI、Google Gemini 或 xAI 密钥。
    </Note>

  </Step>

  <Step title="选择消息渠道">
    选择一个或多个要连接的渠道：

    - **WhatsApp** -- 扫描设置向导中显示的二维码。
    - **Telegram** -- 粘贴从 [BotFather](https://t.me/BotFather) 获取的 Bot 令牌。

  </Step>

  <Step title="完成安装">
    点击 **Finish** 部署实例。准备就绪后，从 hPanel 中的 **OpenClaw Overview** 访问 OpenClaw 控制面板。
  </Step>

</Steps>

## 选项 B：在 VPS 上运行 OpenClaw

获得对服务器的更多控制权。Hostinger 通过 Docker 将 OpenClaw 部署到你的 VPS；你可以通过 hPanel 中的 **Docker Manager** 进行管理。

<Steps>
  <Step title="购买 VPS">
    1. 在 [Hostinger OpenClaw 页面](https://www.hostinger.com/openclaw)中选择 OpenClaw on VPS 套餐并完成结账。

    <Note>
    你可以在结账时选择 **Ready-to-Use AI** 点数。这些点数已预先购买并立即集成到 OpenClaw 中，因此无需其他提供商的任何外部账户或 API 密钥即可开始聊天。
    </Note>

  </Step>

  <Step title="配置 OpenClaw">
    VPS 预配完成后，填写配置字段：

    - **Gateway token** -- 自动生成；请保存以供稍后使用。
    - **WhatsApp number** -- 带国家/地区代码的号码（可选）。
    - **Telegram bot token** -- 从 [BotFather](https://t.me/BotFather) 获取（可选）。
    - **API keys** -- 仅当结账时未选择 Ready-to-Use AI 点数时才需要。

  </Step>

  <Step title="启动 OpenClaw">
    点击 **Deploy**。运行后，在 hPanel 中点击 **Open**，打开 OpenClaw 控制面板。
  </Step>

</Steps>

日志、重启和更新操作均通过 hPanel 中的 Docker Manager 界面执行。如需更新，请在 Docker Manager 中按 **Update** 以拉取最新镜像。

## 验证设置

在已连接的渠道上向你的助手发送“Hi”。OpenClaw 会回复并引导你完成初始偏好设置。

## 故障排查

**控制面板无法加载** -- 等待几分钟，让容器完成预配，然后查看 hPanel 中的 Docker Manager 日志。

**Docker 容器持续重启** -- 打开 Docker Manager 日志并查找配置错误（缺少令牌、API 密钥无效）。

**Telegram Bot 无响应** -- 如果需要私信配对，未知发送者将收到一个简短的配对码，而不是回复。可通过 OpenClaw 控制面板聊天批准配对；如果你可以通过 shell 访问容器，也可使用 `openclaw pairing approve telegram <CODE>`。请参阅[配对](/zh-CN/channels/pairing)。

## 后续步骤

- [渠道](/zh-CN/channels) -- 连接 Telegram、WhatsApp、Discord 等
- [Gateway 配置](/zh-CN/gateway/configuration) -- 所有配置选项

## 相关内容

- [安装概览](/zh-CN/install)
- [VPS 托管](/zh-CN/vps)
- [DigitalOcean](/zh-CN/install/digitalocean)
