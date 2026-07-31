---
read_when:
    - 你想查看 OpenClaw 支持内容的完整列表
summary: OpenClaw 在各渠道、路由、媒体和用户体验方面的能力。
title: 功能
x-i18n:
    generated_at: "2026-07-26T06:11:34Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 5bc3ebdd87a0f6ea0f3d75d029bf7cae469ecd9db84a165bd47c4896936fe303
    source_path: concepts/features.md
    workflow: 16
---

## 亮点

<Columns>
  <Card title="渠道" icon="message-square" href="/zh-CN/channels">
    通过单个 Gateway 网关连接 Discord、iMessage、Signal、Slack、Telegram、WhatsApp、WebChat 等渠道。
  </Card>
  <Card title="插件" icon="plug" href="/zh-CN/tools/plugin">
    使用一条安装命令即可通过官方插件添加 Matrix、Nextcloud Talk、Nostr、Twitch、Zalo 等数十种集成。
  </Card>
  <Card title="路由" icon="route" href="/zh-CN/concepts/multi-agent">
    支持会话隔离的多智能体路由。
  </Card>
  <Card title="媒体" icon="image" href="/zh-CN/nodes/images">
    图像、音频、视频、文档以及图像/视频生成。
  </Card>
  <Card title="应用和 UI" icon="monitor" href="/zh-CN/platforms">
    Windows Hub、浏览器 Control UI、macOS 菜单栏应用和移动节点。
  </Card>
  <Card title="移动节点" icon="smartphone" href="/zh-CN/nodes">
    支持配对、语音/聊天和丰富设备命令的 iOS 与 Android 节点。
  </Card>
</Columns>

## 完整列表

**渠道：**

- iMessage、Telegram 和 WebChat 随核心安装提供；其他所有渠道均为
  使用 `openclaw plugins install @openclaw/<id>` 安装的官方插件（也可在
  `openclaw onboard` / `openclaw channels add` 期间按需安装）
- 官方插件渠道：Discord、Feishu、Google Chat、IRC、LINE、Matrix、Mattermost、
  Microsoft Teams、Nextcloud Talk、Nostr、QQ Bot、Raft、Signal、Slack、SMS、Synology Chat、
  Tlon、Twitch、语音通话、WhatsApp、Zalo 和 Zalo Personal
- 在 OpenClaw 仓库外维护的外部插件渠道：微信、腾讯元宝和 Zalo ClawBot
- 支持通过提及触发的群聊
- 通过允许列表和配对保障私信安全

**智能体：**

- 支持工具流式传输的嵌入式智能体运行时
- 按工作区或发送者隔离会话的多智能体路由
- 会话：直接聊天合并到共享的 `main`；群组则相互隔离
- 针对长回复的流式传输和分块

**身份验证和提供商：**

- 35+ 个模型提供商（Anthropic、OpenAI、Google 等）
- 通过 OAuth 进行订阅身份验证（例如 OpenAI Codex）
- 支持自定义和自托管提供商（vLLM、SGLang、Ollama、llama.cpp、LM Studio 以及
  任何兼容 OpenAI 或 Anthropic 的端点）

**媒体：**

- 输入和输出图像、音频、视频及文档
- 共享的图像生成和视频生成能力接口
- 语音留言转录
- 支持多个提供商的文本转语音

**应用和界面：**

- WebChat 和浏览器 Control UI
- macOS 菜单栏配套应用
- 支持配对、Canvas、摄像头、屏幕录制、位置和语音的 iOS 节点
- 支持配对、聊天、语音、Canvas、摄像头和设备命令的 Android 节点

**工具和自动化：**

- 浏览器自动化、Exec、沙箱隔离
- Web 搜索（Brave、DuckDuckGo、Exa、Firecrawl、Gemini、Grok、Kimi、MiniMax Search、Ollama Web 搜索、Perplexity、SearXNG、Tavily）
- 定时任务和 Heartbeat 调度
- Skills、插件和工作流流水线（Lobster）

## 相关内容

<CardGroup cols={2}>
  <Card title="实验性功能" href="/zh-CN/concepts/experimental-features" icon="flask">
    尚未在默认功能界面中发布的可选功能。
  </Card>
  <Card title="智能体运行时" href="/zh-CN/concepts/agent" icon="robot">
    智能体运行时模型以及运行的分派方式。
  </Card>
  <Card title="渠道" href="/zh-CN/channels" icon="message-square">
    通过一个 Gateway 网关连接 Telegram、WhatsApp、Discord、Slack 等渠道。
  </Card>
  <Card title="插件" href="/zh-CN/tools/plugin" icon="plug">
    扩展 OpenClaw 的官方和外部插件。
  </Card>
</CardGroup>
