---
read_when:
    - 設定長時間執行的聊天回合所顯示的進度更新
    - 在部分、區塊與進度串流模式之間選擇
    - 說明 OpenClaw 如何在工作進行期間更新單一頻道訊息
    - 疑難排解進度草稿、獨立進度訊息或完成作業的備援方案
summary: 進度草稿：一則可見的進行中訊息，會在代理程式執行期間持續更新
title: 進度草稿
x-i18n:
    generated_at: "2026-07-26T07:49:36Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 4ef66dd4d7a31c753f5faa0b88b83ec3760beecf3118cf8aae84f5e57652e809
    source_path: concepts/progress-drafts.md
    workflow: 16
---

進度草稿會在代理程式工作時，將一則頻道訊息轉換為即時狀態列，而不是堆疊一連串暫時的「仍在處理」回覆。設定
`channels.<channel>.streaming.mode: "progress"` 後，OpenClaw 會在實際工作開始時建立訊息，並隨著代理程式讀取、規劃、呼叫工具或等待核准而編輯訊息，最後再將其轉換為最終答案。

```text
處理中...
📖 來源：docs/concepts/progress-drafts.md
🔎 網頁搜尋："discord edit message"
🛠️ Bash：執行測試
```

<Note>
  當 `channels.discord.streaming` 未設定時，Discord 已預設為 `streaming.mode: "progress"`，因此無須任何設定即可在該處顯示進度草稿。其他所有頻道預設為 `partial`
  或 `off`；如需完整的各頻道預設值表格，請參閱[串流與分塊](/zh-TW/concepts/streaming#channel-mapping)。
</Note>

## 快速開始

```json5
{
  channels: {
    discord: {
      streaming: {
        mode: "progress",
      },
    },
  },
}
```

此處的預設值為：開始延遲 5 秒、在進行有用工作時顯示精簡的進度列，並在該輪對話中隱藏較舊的獨立進度訊息。原始工具列草稿會使用自動產生的單字標籤；除非明確設定標籤，否則狀態標題會省略這個多餘的標題。

本頁說明進度草稿體驗及其設定選項。如需完整的串流模式矩陣、各頻道執行階段注意事項及舊版鍵值遷移方式，請參閱[串流與分塊](/zh-TW/concepts/streaming)。

## 使用者會看到什麼

| 部分            | 用途                                                                           |
| --------------- | --------------------------------------------------------------------------------- |
| 狀態標題 | 在 Discord 和 Telegram 上顯示模型的前言；Discord 會加入輔助填充文字。       |
| 標籤           | 選用的起始／狀態列，例如 `Working`。                                   |
| 進度列  | 使用與 `/verbose` 相同的工具圖示和詳細資訊格式器，顯示精簡的執行更新。 |

對於原始工具進度，代理程式開始進行實質工作並持續忙碌超過初始延遲後，標籤就會出現。
它位於持續更新的進度列清單頂端，因此當具體工作列累積到一定數量後，便會隨捲動消失。除非明確設定標籤，否則狀態標題只會顯示代理程式以一般語言撰寫的狀態。純文字回覆絕不會顯示進度草稿；只有實際工作更新才會產生進度列，例如 `🛠️ Bash: run tests`、`🔎 Web Search: for "discord edit message"`
或 `✍️ Write: to /tmp/file`。

當頻道能安全地就地取代草稿時，最終答案會直接取代草稿；否則 OpenClaw 會透過一般傳送流程傳送最終答案，並清除草稿或停止更新草稿（請參閱[完成處理](#finalization)）。

## 選擇模式

`channels.<channel>.streaming.mode` 控制處理期間的可見行為：

| 模式       | 最適合                         | 聊天中顯示的內容                              |
| ---------- | -------------------------------- | ------------------------------------------------- |
| `off`      | 安靜的頻道                   | 僅顯示最終答案。                            |
| `partial`  | 觀看答案文字逐步出現      | 編輯同一份草稿，顯示最新的答案文字。     |
| `block`    | 較大的答案預覽區塊     | 以較大的區塊更新或附加同一份預覽。 |
| `progress` | 大量使用工具或長時間執行的對話輪次 | 顯示一份狀態草稿，接著顯示最終答案。          |

當使用者比起逐字元觀看答案文字串流，更關心「目前正在發生什麼」時，請選擇 `progress`；當答案文字本身就是進度訊號時，請選擇 `partial`；若要使用較大的預覽區塊，請選擇 `block`。在 Discord 和 Telegram 上，`streaming.mode: "block"` 仍屬於預覽串流，而不是一般的區塊回覆傳送方式——後者請使用 `streaming.block.enabled`。

## 設定標籤

進度標籤位於 `channels.<channel>.streaming.progress` 之下。預設的原始工具列標籤為 `"auto"`，它會使用內建的純文字 `Working` 標籤。狀態標題會隱藏這個隱含標籤；如果也想在其上方顯示標籤，請明確設定
`label: "auto"`：

```text
處理中
```

使用固定標籤：

```json5
{
  channels: {
    discord: {
      streaming: {
        mode: "progress",
        progress: {
          label: "調查中",
        },
      },
    },
  },
}
```

使用自訂標籤集（當 `label: "auto"` 時，仍會隨機／依種子選取）：

```json5
{
  channels: {
    discord: {
      streaming: {
        mode: "progress",
        progress: {
          label: "auto",
          labels: ["檢查中", "讀取中", "測試中", "收尾中"],
        },
      },
    },
  },
}
```

隱藏標籤，只顯示進度列：

```json5
{
  channels: {
    discord: {
      streaming: {
        mode: "progress",
        progress: {
          label: false,
        },
      },
    },
  },
}
```

## 控制進度列

進度列來自實際的執行事件：工具啟動、項目更新、工作計畫、核准、命令輸出、修補摘要及類似的代理程式活動。這些功能預設為啟用（`progress.toolProgress`，預設值為 `true`）。

工具也可以在單次呼叫仍在執行時發出具型別的進度。這能讓耗時的擷取或搜尋在工具傳回最終結果前，就更新可見草稿。進度更新是一項部分工具結果，其中模型內容為空，並包含明確的公開頻道中繼資料：

```json
{
  "content": [],
  "progress": {
    "text": "正在擷取頁面內容...",
    "visibility": "channel",
    "privacy": "public",
    "id": "web_fetch:fetching"
  }
}
```

OpenClaw 在頻道進度介面中只會呈現 `progress.text`。一般工具結果稍後仍會以 `content`/`details` 的形式到達，且只有該部分會傳回模型。

為工具新增進度時，請發出簡短、通用的訊息，並延遲到作業已等待足夠時間、顯示進度確實有用時才送出。`web_fetch` 會以 5 秒延遲完成此操作：

```typescript
const clearProgressTimer = scheduleToolProgress(
  onUpdate,
  { text: "正在擷取頁面內容...", id: "web_fetch:fetching" },
  5_000,
  { signal },
);

try {
  return await runToolWork();
} finally {
  clearProgressTimer();
}
```

快速呼叫不會顯示進度列；長時間呼叫會在仍處於等待狀態時顯示一列；已取消的呼叫會先清除計時器，避免顯示過時的進度。進度文字是公開的介面旁路頻道，因此絕不可包含祕密、原始引數、擷取到的內容、命令輸出或頁面文字。

### 詳細資訊模式

OpenClaw 對進度草稿和 `/verbose` 使用相同的格式器：

```json5
{
  agents: {
    defaults: {
      toolProgressDetail: "explain", // explain | raw
    },
  },
}
```

`"explain"` 是預設值，會使用精簡標籤來維持草稿穩定。`"raw"` 會在可用時附加底層命令，這在偵錯時很實用，但在聊天中較為嘈雜。例如，`node --check /tmp/app.js` 呼叫在不同模式下會呈現不同內容：

| 模式      | 進度列                                                   |
| --------- | --------------------------------------------------------------- |
| `explain` | `🛠️ check js syntax for /tmp/app.js`                            |
| `raw`     | `🛠️ check js syntax for /tmp/app.js · node --check /tmp/app.js` |

### 命令／執行文字

`streaming.progress.commandText`（預設值為 `"raw"`）控制 exec/bash 進度列旁顯示多少命令詳細資訊，且不受上方詳細資訊模式影響。將其設定為 `"status"`，即可在隱藏所有命令文字的同時保留工具進度列：

```json5
{
  channels: {
    discord: {
      streaming: {
        mode: "progress",
        progress: {
          commandText: "status",
        },
      },
    },
  },
}
```

### 評述通道

`streaming.progress.commentary`（預設值為 `false`）會在草稿中的工具列之間穿插模型在使用工具前的評述／前言敘述（💬，例如「我會先檢查……，接著……」）。如需各頻道共用的設定形式，請參閱[串流與分塊](/zh-TW/concepts/streaming#commentary-progress-lane)。

啟用評述通道後，前言只會呈現為這些穿插的 💬 列；下方的狀態標題不會顯示，以便該通道維持其記載的形式。

### 狀態標題

在 Discord 和 Telegram 的進度模式下，只要模型提供具型別的工具前前言，就會成為草稿的狀態標題。其他進度模式頻道會維持現有的狀態行為。標題預設為啟用，且短時間對話輪次仍須通過一般活動門檻；啟用 `streaming.progress.commentary` 後，前言會改交由穿插式評述通道處理。

在 Discord 上，當代理程式能解析出輔助模型時——明確設定的 [`utilityModel`](/zh-TW/gateway/config-agents#utilitymodel)，或主要供應商宣告的小型模型預設值（OpenAI → `gpt-5.6-luna`、Anthropic → `claude-haiku-4-5`）——若模型未發出前言或已沉默約 20 秒，該模型會提供簡短的一般語言填充文字（目前 Telegram 的標題只使用前言）：

```text
正在更新設定中的預設模型，接著重新啟動閘道以套用變更。
一次代理程式清單呼叫失敗，目前正在重試。
```

輔助敘述預設為啟用（`streaming.progress.narration`，預設值為 `true`），且絕不會退回使用主要模型：它只會在明確設定 `utilityModel`，或代理程式主要供應商宣告了預設值時執行。將 `utilityModel: ""` 設為停用，即可完全停用輔助路由。工具列會繼續在下方累積，若兩種狀態來源都停止，工具列便會重新出現。草稿編輯仍會等待通過一般活動門檻且文字確實發生變更，藉此避免短時間對話輪次出現閃爍，並減少繁忙頻道中的編輯頻率。將 `narration: false` 設為停用，可只停用輔助模型填充文字；模型前言標題仍會保持啟用：

```json5
{
  channels: {
    discord: {
      streaming: {
        mode: "progress",
        progress: {
          narration: false,
        },
      },
    },
  },
}
```

敘述輸入有長度限制且會經過遮蔽：輔助模型會收到傳入的請求文字，以及草稿原本會呈現的相同精簡遮蔽工具摘要——絕不包含原始命令輸出或工具結果。使用 `commandText: "status"` 時，敘述輸入也會省略 exec/bash 命令文字，與草稿顯示的內容一致。

### 行數限制

限制保持可見的行數（預設為 8）：

```json5
{
  channels: {
    discord: {
      streaming: {
        mode: "progress",
        progress: {
          maxLines: 4,
        },
      },
    },
  },
}
```

進度列會自動精簡，以減少編輯草稿時聊天泡泡重新排列的情況；OpenClaw 也會截斷過長的行，讓反覆編輯草稿時不會在每次更新時以不同方式換行。預設的單行上限為 120 個字元；一般文字會在單字邊界截斷，而路徑或原始命令等較長的詳細資訊則會以置中省略號縮短，讓結尾仍保持可見。

調整單行上限：

```json5
{
  channels: {
    discord: {
      streaming: {
        mode: "progress",
        progress: {
          maxLineChars: 160,
        },
      },
    },
  },
}
```

### 豐富呈現（Slack）

Slack 可以將進度列呈現為結構化 Block Kit 欄位，而不是純文字：

```json5
{
  channels: {
    slack: {
      streaming: {
        mode: "progress",
        progress: {
          render: "rich",
        },
      },
    },
  },
}
```

豐富呈現一律會在 Block Kit 欄位之外，同時傳送相同的純文字本文，因此無法呈現較豐富格式的用戶端仍可顯示精簡的進度文字。

### 隱藏工具／工作列

保留單一進度草稿，但隱藏工具與工作列：

```json5
{
  channels: {
    discord: {
      streaming: {
        mode: "progress",
        progress: {
          toolProgress: false,
        },
      },
    },
  },
}
```

使用 `toolProgress: false` 時，OpenClaw 仍會抑制該回合較舊的獨立
工具進度訊息——頻道在視覺上會保持安靜，直到顯示最終答案；
若已設定標籤，則標籤除外。

## 頻道行為

| 頻道            | 進度傳輸方式                           | 備註                                                                                                                                                                  |
| --------------- | -------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Discord         | 傳送一則訊息，然後編輯。               | 預設使用 `progress` 模式；最終答案會附帶 `-#` 活動摘要，並在答案送達後刪除狀態草稿。                                                             |
| Matrix          | 傳送一個事件，然後編輯。               | 帳號層級的串流設定會控制帳號層級的草稿。                                                                                                                              |
| Microsoft Teams | 個人聊天中的原生 Teams 串流。          | `streaming.mode: "block"` 會改為對應至 Teams 區塊傳送。                                                                                                                       |
| Slack           | 原生串流或可編輯的草稿貼文。           | 需要回覆討論串目標；沒有目標的頂層私訊仍會收到草稿預覽貼文及其編輯。                                                                                                  |
| Telegram        | 傳送一則訊息，然後編輯。               | 若進度草稿與答案之間出現另一則訊息，草稿會重新張貼在該訊息下方（先張貼新草稿，再刪除舊草稿），而不會讓用戶端捲動位置突然跳動。                                          |
| Mattermost      | 可編輯的草稿貼文。                     | `block` 模式會在已完成文字與工具活動貼文之間輪替；其他模式則會將工具活動整合至同一則草稿式貼文中。                                                          |

不支援安全編輯的頻道會退回使用輸入中指示器，或僅傳送
最終答案。請參閱[串流與分塊](/zh-TW/concepts/streaming)，瞭解各頻道完整的
執行階段行為解析。

## 完成處理

最終答案準備就緒時，OpenClaw 會嘗試保持聊天內容整潔：

- 在 Discord 的 `progress` 模式中，最終答案會以新訊息傳送，
  並附加簡短的 `-#` 活動摘要（例如
  `-# 🧠 2 thoughts · 🛠️ 5 tool calls · ⏱️ 12s`）；該答案送達後，
  狀態草稿即會刪除。繁忙的頻道不會在回覆上方留下無主的工具
  記錄；錯誤最終結果則會保留草稿，作為該失敗回合的可見記錄。
- 如果草稿可安全地直接成為最終答案（`partial`/`block` 模式），
  OpenClaw 會就地編輯草稿。
- 如果頻道使用原生進度串流，當原生傳輸接受最終文字時，
  OpenClaw 會完成該串流。
- 否則（媒體、核准提示、明確的回覆目標、分塊過多，
  或編輯／傳送失敗），OpenClaw 會透過一般頻道傳送路徑傳送
  最終答案，而不覆寫草稿。

此退回機制是刻意設計的：傳送新的最終答案，優於遺失文字、
將回覆放入錯誤的討論串，或以頻道無法安全呈現的內容覆寫草稿。

## 疑難排解

**我只看到最終答案。**

請檢查處理該訊息的帳號或頻道，其 `channels.<channel>.streaming.mode` 是否為
`progress`。當頻道無法安全編輯正確的訊息時，某些群組或
引用回覆路徑會停用該回合的草稿預覽。

**我看到標籤，但看不到工具行。**

請檢查 `streaming.progress.toolProgress`。若其值為 `false`，OpenClaw 會維持
單一草稿行為，但隱藏工具與工作進度行。

**我看到的是新的最終訊息，而非經過編輯的草稿。**

這就是[完成處理](#finalization)中所述的安全退回機制。媒體回覆、
較長的答案、明確的回覆目標、過舊的 Telegram 草稿、缺少 Slack
討論串目標、已刪除的預覽訊息，或原生串流完成失敗時，都可能發生此情況。

**我仍然看到獨立的進度訊息。**

草稿啟用時，進度模式會抑制預設的獨立工具進度訊息。若仍出現
獨立訊息，請確認該回合實際使用的是 `progress` 模式，而非
`streaming.mode: "off"`，也不是無法為該訊息建立草稿的頻道路徑。

**Teams 的行為與 Discord 或 Telegram 不同。**

Microsoft Teams 在個人聊天中使用原生串流，而非通用的
傳送後編輯預覽傳輸，並將 `streaming.mode: "block"` 對應至 Teams
區塊傳送，因為它沒有像 Discord 和 Telegram 那樣的草稿預覽區塊模式。

## 相關內容

- [串流與分塊](/zh-TW/concepts/streaming)
- [訊息](/zh-TW/concepts/messages)
- [頻道設定](/zh-TW/gateway/config-channels)
- [Discord](/zh-TW/channels/discord)
- [Matrix](/zh-TW/channels/matrix)
- [Microsoft Teams](/zh-TW/channels/msteams)
- [Slack](/zh-TW/channels/slack)
- [Telegram](/zh-TW/channels/telegram)
- [Mattermost](/zh-TW/channels/mattermost)
