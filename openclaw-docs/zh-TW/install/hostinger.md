---
read_when:
    - 在 Hostinger 上設定 OpenClaw
    - 正在尋找適合 OpenClaw 的代管 VPS
    - 使用 Hostinger 一鍵式 OpenClaw
summary: 在 Hostinger 上託管 OpenClaw
title: Hostinger
x-i18n:
    generated_at: "2026-07-26T08:29:40Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 7dc49e741f8581928553e2426ed91f92df6e7b0c31dd8780c0d6e891a07be263
    source_path: install/hostinger.md
    workflow: 16
---

在 [Hostinger](https://www.hostinger.com/openclaw) 上執行持續運作的 OpenClaw 閘道，可選擇 **1-Click** 代管部署，或自行管理的 **VPS** 安裝。

## 事前準備

- Hostinger 帳號（[註冊](https://www.hostinger.com/openclaw)）
- 約 5-10 分鐘

## 選項 A：1-Click OpenClaw

Hostinger 負責基礎架構、Docker 及自動更新。這是最快讓執行個體開始運作的方式。

<Steps>
  <Step title="購買並啟動">
    1. 前往 [Hostinger OpenClaw 頁面](https://www.hostinger.com/openclaw)，選擇 Managed OpenClaw 方案並完成結帳。

    <Note>
    結帳時，你可以選擇預先購買並立即整合至 OpenClaw 的 **Ready-to-Use AI** 點數，不需要其他供應商的外部帳號或 API 金鑰，即可立即開始聊天。你也可以在設定期間提供自己的 Anthropic、OpenAI、Google Gemini 或 xAI 金鑰。
    </Note>

  </Step>

  <Step title="選擇訊息頻道">
    選擇要連線的一個或多個頻道：

    - **WhatsApp** -- 掃描設定精靈中顯示的 QR code。
    - **Telegram** -- 貼上來自 [BotFather](https://t.me/BotFather) 的機器人權杖。

  </Step>

  <Step title="完成安裝">
    按一下 **Finish** 以部署執行個體。準備完成後，從 hPanel 的 **OpenClaw Overview** 存取 OpenClaw 儀表板。
  </Step>

</Steps>

## 選項 B：VPS 上的 OpenClaw

提供更多伺服器控制權。Hostinger 會透過 Docker 將 OpenClaw 部署至你的 VPS；你可透過 hPanel 中的 **Docker Manager** 進行管理。

<Steps>
  <Step title="購買 VPS">
    1. 前往 [Hostinger OpenClaw 頁面](https://www.hostinger.com/openclaw)，選擇 OpenClaw on VPS 方案並完成結帳。

    <Note>
    結帳時，你可以選擇 **Ready-to-Use AI** 點數。這些點數會預先購買並立即整合至 OpenClaw，因此不需要其他供應商的任何外部帳號或 API 金鑰，即可開始聊天。
    </Note>

  </Step>

  <Step title="設定 OpenClaw">
    VPS 佈建完成後，填寫設定欄位：

    - **Gateway token** -- 自動產生；請儲存以供稍後使用。
    - **WhatsApp number** -- 包含國碼的電話號碼（選填）。
    - **Telegram bot token** -- 來自 [BotFather](https://t.me/BotFather)（選填）。
    - **API keys** -- 僅在結帳時未選擇 Ready-to-Use AI 點數的情況下需要。

  </Step>

  <Step title="啟動 OpenClaw">
    按一下 **Deploy**。開始運作後，在 hPanel 中按一下 **Open**，即可開啟 OpenClaw 儀表板。
  </Step>

</Steps>

日誌、重新啟動及更新皆透過 hPanel 中的 Docker Manager 介面執行。若要更新，請在 Docker Manager 中按下 **Update**，以提取最新映像檔。

## 驗證設定

在已連線的頻道上向助理傳送「Hi」。OpenClaw 會回覆，並引導你完成初始偏好設定。

## 疑難排解

**儀表板無法載入** -- 等待幾分鐘，讓容器完成佈建，然後檢查 hPanel 中的 Docker Manager 日誌。

**Docker 容器持續重新啟動** -- 開啟 Docker Manager 日誌，並檢查設定錯誤（缺少權杖、API 金鑰無效）。

**Telegram 機器人沒有回應** -- 如果需要進行私訊配對，未知傳送者會收到簡短的配對碼，而不是回覆。請透過 OpenClaw 儀表板聊天核准，或在你擁有容器 Shell 存取權時使用 `openclaw pairing approve telegram <CODE>`。請參閱[配對](/zh-TW/channels/pairing)。

## 後續步驟

- [頻道](/zh-TW/channels) -- 連接 Telegram、WhatsApp、Discord 等服務
- [閘道設定](/zh-TW/gateway/configuration) -- 所有設定選項

## 相關內容

- [安裝概覽](/zh-TW/install)
- [VPS 主機託管](/zh-TW/vps)
- [DigitalOcean](/zh-TW/install/digitalocean)
