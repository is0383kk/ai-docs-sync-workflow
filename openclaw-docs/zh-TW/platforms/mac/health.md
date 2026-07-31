---
read_when:
    - 偵錯 Mac 應用程式的健康狀態指示器
summary: macOS 應用程式如何回報閘道／頻道的健康狀態
title: 健康檢查（macOS）
x-i18n:
    generated_at: "2026-07-26T07:24:39Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 095abdbefa7db7c0d14435e2c5db7d1ebc03afa0c539555a7abdd9170d015fb8
    source_path: platforms/mac/health.md
    workflow: 16
---

# macOS 上的健康檢查

如何從選單列 App 讀取已連結頻道的健康狀態。

## 選單列

狀態圓點：

- 綠色：已連結 + 探測正常。
- 橘色：已連結，但頻道探測回報效能降低／未連線。
- 紅色：尚未連結。

次要資訊列會顯示「已連結 · 認證 12 分鐘」，或顯示失敗原因。
選單中的「立即執行健康檢查」會觸發隨選探測。

## 設定

- 「一般」分頁會顯示健康狀態卡片：狀態圓點、摘要資訊列（連結狀態 +
  認證時間），以及選用的失敗詳細資訊列，並提供「**立即重試**」與
  「**開啟日誌**」按鈕。
- 「**頻道分頁**」會顯示 WhatsApp 和 Telegram 各頻道的狀態與控制項目（登入 QR Code、
  登出、探測、上次中斷連線／錯誤）。

## 探測的運作方式

App 會透過現有的 WebSocket 連線（而非呼叫命令列介面殼層），每隔約 60 秒及隨選呼叫閘道的 `health` RPC。
此 RPC 會載入認證資訊並回報狀態，而不會傳送訊息。App 會分別快取最後一次正常的
快照與最後一次錯誤，讓 UI 能立即載入，且離線時不會閃爍。

## 有疑問時

使用[閘道健康狀態](/zh-TW/gateway/health)中的命令列介面流程（`openclaw status`、
`openclaw status --deep`、`openclaw health --json`），並執行
`openclaw logs --follow`，篩選 `web-heartbeat` / `web-reconnect`。

## 相關內容

- [閘道健康狀態](/zh-TW/gateway/health)
- [macOS App](/zh-TW/platforms/macos)
