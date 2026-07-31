---
read_when:
    - 你正在變更時間戳記向模型或使用者顯示的方式
    - 你正在偵錯訊息或系統提示詞輸出中的時間格式設定
summary: 信封、提示詞、工具與連接器的日期和時間處理
title: 日期與時間
x-i18n:
    generated_at: "2026-07-26T08:31:58Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: e6f923022c021c1cf18ba306cd7b9a4873f5df947bb9a8fae9c737a89f64cbf2
    source_path: date-time.md
    workflow: 16
---

OpenClaw 使用**主機本機時間作為傳輸時間戳記**，並且在系統提示詞中**只放入時區**。
系統會保留提供者時間戳記，讓工具維持其原生語意。當代理程式需要目前
時間時，會執行 `session_status` 工具。

## 訊息信封（預設使用本機時間）

傳入訊息會以星期幾加上精確到秒的時間戳記包裝：

```
[WhatsApp +1555 Mon 2026-01-05 16:26:34 PST] 訊息文字
```

無論提供者的時區為何，信封時間戳記**預設使用主機本機時間**。
可在 `agents.defaults` 下覆寫：

```json5
{
  agents: {
    defaults: {
      envelopeTimezone: "local", // "utc" | "local" | "user" | IANA 時區
      envelopeTimestamp: "on", // "on" | "off"
      envelopeElapsed: "on", // "on" | "off"
    },
  },
}
```

| 鍵                  | 值                                                   | 行為                                                                                                                                                                            |
| ------------------- | ---------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `envelopeTimezone`  | `local`（預設）、`utc`、`user`、明確的 IANA 名稱 | `user` 使用 `agents.defaults.userTimezone`（未設定時使用主機時區）。明確的 IANA 名稱（例如 `"America/Chicago"`）會固定使用特定時區；無法辨識的名稱則回退至 UTC。 |
| `envelopeTimestamp` | `on`（預設）、`off`                                | `off` 會從信封標頭、直接代理程式提示詞前綴，以及嵌入的模型輸入前綴中移除絕對時間戳記。                                                       |
| `envelopeElapsed`   | `on`（預設）、`off`                                | `off` 會移除自工作階段中上一則訊息以來顯示的經過時間後綴（`+30s` / `+2m` 樣式）。                                                               |

### 範例

**本機時間（預設）：**

```
[WhatsApp +1555 Sun 2026-01-18 00:19:42 PST] 你好
```

**使用者時區：**

```
[WhatsApp +1555 Sun 2026-01-18 00:19:42 CST] 你好
```

**搭配 `envelopeTimezone: "utc"` 的經過時間：**

```
[WhatsApp +1555 +30s Sun 2026-01-18T05:19:00Z] 後續訊息
```

## 系統提示詞：目前日期與時間

系統提示詞包含一個**目前日期與時間**區段，其中**只有時區**
（不含時鐘時間或時間格式），使提示詞快取保持穩定：

```
時區：America/Chicago
```

若已設定，時區會使用 `agents.defaults.userTimezone`，否則使用主機時區。
提示詞也會指示代理程式，每當需要目前日期、時間或星期幾時，
執行 `session_status` 工具。

## 系統事件行（預設使用本機時間）

插入代理程式情境的佇列系統事件，會使用與訊息信封相同的
`envelopeTimezone` 選項加上時間戳記前綴（預設：主機本機時間）。

```
系統：[2026-01-12 12:19:17 PST] 模型已切換。
```

### 設定使用者時區與格式

```json5
{
  agents: {
    defaults: {
      userTimezone: "America/Chicago",
      timeFormat: "auto", // auto | 12 | 24
    },
  },
}
```

- `userTimezone` 會設定提示詞情境（以及 `envelopeTimezone: "user"`）所使用的**使用者本機時區**。
- `timeFormat` 控制提示詞所呈現時間的 **12 小時制／24 小時制顯示**。`auto` 會遵循作業系統偏好設定。

## 時間格式偵測（自動）

當 `timeFormat: "auto"` 時，OpenClaw 會檢查作業系統偏好設定（macOS 與 Windows），
並在無法判定時回退至地區設定格式。偵測到的值會**依處理程序快取**，
以避免重複呼叫系統。

## 工具承載資料與連接器（原始提供者時間與正規化欄位）

頻道工具會傳回**提供者原生時間戳記**，並加入正規化欄位以維持一致性：

- `timestampMs`：Epoch 毫秒數（UTC）
- `timestampUtc`：ISO 8601 UTC 字串

系統會保留原始提供者欄位，確保不會遺失任何內容。

- Discord：UTC ISO 時間戳記
- Slack：來自 API、類似 Epoch 的字串
- Telegram/WhatsApp：提供者特有的數值／ISO 時間戳記

如果需要本機時間，請在下游使用已知時區進行轉換。

## 相關文件

- [系統提示詞](/zh-TW/concepts/system-prompt)
- [時區](/zh-TW/concepts/timezone)
- [訊息](/zh-TW/concepts/messages)
