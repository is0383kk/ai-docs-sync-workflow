---
read_when:
    - 你想查看 OpenClaw 支援功能的完整清單
summary: OpenClaw 在各通道、路由、媒體與使用者體驗方面的功能。
title: 功能
x-i18n:
    generated_at: "2026-07-26T07:48:40Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 5bc3ebdd87a0f6ea0f3d75d029bf7cae469ecd9db84a165bd47c4896936fe303
    source_path: concepts/features.md
    workflow: 16
---

## 特色亮點

<Columns>
  <Card title="頻道" icon="message-square" href="/zh-TW/channels">
    透過單一閘道使用 Discord、iMessage、Signal、Slack、Telegram、WhatsApp、WebChat 等服務。
  </Card>
  <Card title="外掛" icon="plug" href="/zh-TW/tools/plugin">
    只需一道安裝命令，官方外掛即可新增 Matrix、Nextcloud Talk、Nostr、Twitch、Zalo 等數十種服務。
  </Card>
  <Card title="路由" icon="route" href="/zh-TW/concepts/multi-agent">
    具備隔離工作階段的多代理程式路由。
  </Card>
  <Card title="媒體" icon="image" href="/zh-TW/nodes/images">
    圖片、音訊、影片、文件，以及圖片／影片生成。
  </Card>
  <Card title="應用程式與使用者介面" icon="monitor" href="/zh-TW/platforms">
    Windows Hub、瀏覽器控制介面、macOS 選單列應用程式，以及行動節點。
  </Card>
  <Card title="行動節點" icon="smartphone" href="/zh-TW/nodes">
    支援配對、語音／聊天及豐富裝置命令的 iOS 與 Android 節點。
  </Card>
</Columns>

## 完整清單

**頻道：**

- iMessage、Telegram 與 WebChat 隨核心安裝提供；其他所有頻道皆為
  使用 `openclaw plugins install @openclaw/<id>` 安裝的官方外掛（也可在
  `openclaw onboard`／`openclaw channels add` 期間視需要安裝）
- 官方外掛頻道：Discord、Feishu、Google Chat、IRC、LINE、Matrix、Mattermost、
  Microsoft Teams、Nextcloud Talk、Nostr、QQ Bot、Raft、Signal、Slack、SMS、Synology Chat、
  Tlon、Twitch、語音通話、WhatsApp、Zalo 與 Zalo Personal
- 由 OpenClaw 儲存庫以外的團隊維護之外部外掛頻道：微信、騰訊元寶與 Zalo ClawBot
- 支援透過提及觸發的群組聊天
- 透過允許清單與配對確保私訊安全

**代理程式：**

- 具備工具串流功能的內嵌代理程式執行階段
- 依工作區或傳送者隔離工作階段的多代理程式路由
- 工作階段：直接聊天會合併至共用 `main`；群組則彼此隔離
- 針對長篇回覆提供串流與分塊處理

**驗證與供應商：**

- 超過 35 個模型供應商（Anthropic、OpenAI、Google 等）
- 透過 OAuth 進行訂閱驗證（例如 OpenAI Codex）
- 支援自訂與自行託管的供應商（vLLM、SGLang、Ollama、llama.cpp、LM Studio，以及
  任何與 OpenAI 或 Anthropic 相容的端點）

**媒體：**

- 收發圖片、音訊、影片與文件
- 共用的圖片生成與影片生成能力介面
- 語音訊息轉錄
- 支援多個供應商的文字轉語音

**應用程式與介面：**

- WebChat 與瀏覽器控制介面
- macOS 選單列輔助應用程式
- 支援配對、Canvas、相機、螢幕錄影、定位與語音的 iOS 節點
- 支援配對、聊天、語音、Canvas、相機與裝置命令的 Android 節點

**工具與自動化：**

- 瀏覽器自動化、命令執行與沙箱隔離
- 網頁搜尋（Brave、DuckDuckGo、Exa、Firecrawl、Gemini、Grok、Kimi、MiniMax Search、Ollama Web Search、Perplexity、SearXNG、Tavily）
- 排程工作與心跳偵測排程
- Skills、外掛與工作流程流水線（Lobster）

## 相關內容

<CardGroup cols={2}>
  <Card title="實驗性功能" href="/zh-TW/concepts/experimental-features" icon="flask">
    尚未在預設介面中推出的選用功能。
  </Card>
  <Card title="代理程式執行階段" href="/zh-TW/concepts/agent" icon="robot">
    代理程式執行階段模型，以及執行作業的分派方式。
  </Card>
  <Card title="頻道" href="/zh-TW/channels" icon="message-square">
    透過單一閘道連接 Telegram、WhatsApp、Discord、Slack 等服務。
  </Card>
  <Card title="外掛" href="/zh-TW/tools/plugin" icon="plug">
    擴充 OpenClaw 的官方與外部外掛。
  </Card>
</CardGroup>
