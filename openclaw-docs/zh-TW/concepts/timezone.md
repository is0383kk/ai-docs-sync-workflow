---
read_when:
    - 你想快速掌握時區處理的基本概念
    - 你正在決定要在哪裡設定或覆寫時區
summary: 時區在 OpenClaw 中出現的位置 — 封裝、工具承載資料、系統提示詞
title: 時區
x-i18n:
    generated_at: "2026-07-26T07:50:31Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 9d1620b4b2cedba89bd6ab4392018cd48d0ef92a6abc1744011d482557e2c4fc
    source_path: concepts/timezone.md
    workflow: 16
---

OpenClaw 會將時間戳記標準化，讓模型看到**單一參考時間**，而不是混雜各提供者本地時鐘的時間。以下三個介面會顯示時區，各有不同用途：

## 三個時區介面

| 介面              | 顯示內容                                                                                                   | 預設值                                      | 設定方式                                               |
| ----------------- | ---------------------------------------------------------------------------------------------------------- | ------------------------------------------- | ------------------------------------------------------ |
| 訊息封裝          | 包裝傳入的頻道訊息：`[Signal +1555 Sun 2026-01-18 00:19:42 PST] hello`                                                                    | 主機本地時區                                | `agents.defaults.envelopeTimezone`                                     |
| 工具承載資料      | 頻道 `readMessages` 類型的工具會傳回原始提供者時間，以及標準化的 `timestampMs` / `timestampUtc` | 一律包含 UTC 欄位                           | 無法設定；保留提供者原生時間戳記                       |
| 系統提示          | 一小段 `Current Date & Time` 區塊，**僅含時區**（不含時鐘值，以維持快取穩定性）                               | 若未設定 `userTimezone`，則使用主機時區 | `agents.defaults.userTimezone`                                     |

系統提示刻意省略即時時鐘，以維持各輪對話之間提示快取的穩定性。代理程式需要目前時間時，會呼叫 `session_status`。

## 設定使用者時區

```json5
{
  agents: {
    defaults: {
      userTimezone: "America/Chicago",
    },
  },
}
```

若未設定 `userTimezone`，OpenClaw 會在執行階段透過 `Intl.DateTimeFormat().resolvedOptions().timeZone` 解析主機時區（不寫入設定）。`agents.defaults.timeFormat`（`auto` | `12` | `24`）控制封裝及下游介面的 12 小時制／24 小時制顯示方式，但不影響系統提示區段。

## 封裝時區值

`agents.defaults.envelopeTimezone` 接受：

- `"local"`（預設）或 `"host"` - 主機的時區。
- `"utc"` 或 `"gmt"` - UTC。
- `"user"` - 已解析的 `agents.defaults.userTimezone`（若未設定，則退回使用主機時區）。
- 任何明確的 IANA 時區字串，例如 `"Europe/Vienna"`。

## 何時應覆寫

- **使用 `"utc"`**，可讓不同地區的主機使用一致的時間戳記，或與採用 UTC 的診斷／日誌輸出保持一致。
- **使用 `"user"`**，可讓封裝一律與設定的使用者時區保持一致，不受閘道主機所在時區影響。
- **使用固定的 IANA 時區**，適用於閘道主機位於某個時區，但無論主機如何遷移，封裝都應一律顯示另一個時區的情況。
- 當時間戳記情境對對話沒有幫助時，請**設定 `envelopeTimestamp: "off"`**。這會移除封裝、直接代理程式提示前綴，以及嵌入式模型輸入前綴中的絕對時間戳記。

如需完整的行為參考、各提供者的範例，以及經過時間的格式設定，請參閱[日期與時間](/zh-TW/date-time)。

## 相關內容

- [日期與時間](/zh-TW/date-time) - 完整的封裝／工具／提示行為與範例。
- [心跳偵測](/zh-TW/gateway/heartbeat) - 活躍時段會使用時區進行排程。
- [排程工作](/zh-TW/automation/cron-jobs) - 排程運算式會使用時區進行排程。
