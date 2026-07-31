---
read_when:
    - 你正在變更對外傳送頻道的 Markdown 格式或分段方式
    - 你正在新增頻道格式化工具或樣式對應關係
    - 你正在偵錯各通道的格式回歸問題
summary: 外送頻道的 Markdown 格式化流水線
title: Markdown 格式設定
x-i18n:
    generated_at: "2026-07-26T07:17:11Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: f9a35fd9a6386068e1e3bec73ec6e692f49239b468f42dd737f919b1c6a88e41
    source_path: concepts/markdown-formatting.md
    workflow: 16
---

OpenClaw 會先將傳出的 Markdown 轉換為共用的中介表示法
(IR)，再渲染成各頻道專用的輸出。IR 會保留純文字以及
樣式／連結範圍，因此只需解析一次即可供所有頻道使用，且分塊絕不會
在範圍中途切斷格式。

## 流水線

1. **將 Markdown 解析為 IR** (`markdownToIR`) - 純文字加上樣式範圍
   （粗體、斜體、刪除線、程式碼、程式碼區塊、劇透、引用區塊、
   1-6 級標題）和連結範圍。位移量使用 UTF-16 程式碼單元，因此 Signal 樣式
   範圍可直接與其 API 對齊。只有在頻道選用表格模式時才會解析表格。
2. **對 IR 分塊** (`chunkMarkdownIR` / `renderMarkdownIRChunksWithinLimit`)
   - 分割會在渲染前對 IR 文字進行，因此行內樣式和
     連結會依各區塊切分，而不會跨越邊界而遭到破壞。
3. **依頻道渲染** (`renderMarkdownWithMarkers`) - 樣式標記對應表會
   將範圍轉換為頻道的原生標記。

| 頻道                                                          | 渲染器                                                                             | 備註                                                                                    |
| ---------------------------------------------------------------- | ------------------------------------------------------------------------------------ | ---------------------------------------------------------------------------------------- |
| Slack                                                            | mrkdwn 權杖（`*bold*`、`_italic_`、`` `code` ``、程式碼圍欄）                      | 連結會變為 `<url\|label>`；解析時停用自動連結，以避免重複建立連結      |
| Telegram                                                         | HTML 標籤（`<b>`、`<i>`、`<s>`、`<code>`、`<pre><code>`、`<a href>`、`<tg-spoiler>`） | `richMessages` 開啟時，也支援豐富訊息表格和標題（`<h1>`-`<h6>`） |
| Signal                                                           | 純文字 + `text-style` 範圍                                                     | 當標籤與 URL 不同時，連結會渲染為 `label (url)`                        |
| Discord、WhatsApp、iMessage、Microsoft Teams 和其他頻道 | 純文字                                                                           | 不使用基於 IR 的樣式；Markdown 表格轉換仍會透過 `convertMarkdownTables` 執行    |

## IR 範例

輸入的 Markdown：

```markdown
你好，**世界** - 請參閱[文件](https://docs.openclaw.ai)。
```

IR（示意）：

```json
{
  "text": "你好，世界 - 請參閱文件。",
  "styles": [{ "start": 6, "end": 11, "style": "bold" }],
  "links": [{ "start": 19, "end": 23, "href": "https://docs.openclaw.ai" }]
}
```

## 表格處理

`markdown.tables` 控制頻道如何轉換 Markdown 表格，可針對
各頻道設定，也可選擇針對各帳號設定：

| 模式      | 行為                                                                             |
| --------- | ------------------------------------------------------------------------------------ |
| `code`    | 在程式碼區塊內渲染為對齊的 ASCII 表格（備援預設值）              |
| `bullets` | 將每一列轉換為 `label: value` 項目符號                                   |
| `block`   | 在傳輸支援時保留原生表格；否則退回使用 `code` |
| `off`     | 停用表格解析；原始表格文字不經變更直接傳遞                       |

各頻道外掛的預設值：Signal、WhatsApp 和 Matrix 預設為
`bullets`；Mattermost 預設為 `off`；Telegram 預設為 `block`（除非
帳號已啟用 `richMessages`，否則會解析為 `code`）。任何
未明確設定外掛預設值的頻道都會退回使用 `code`。

```yaml
channels:
  discord:
    markdown:
      tables: code
    accounts:
      work:
        markdown:
          tables: off
```

## 分塊規則

- 分塊限制來自頻道轉接器／設定，並套用至 IR 文字，而非
  渲染後的輸出。
- 圍欄程式碼區塊會保留為單一區塊並帶有結尾換行，讓
  頻道能正確渲染結束圍欄。
- 清單和引用區塊前綴是 IR 文字的一部分，因此分塊絕不會
  在前綴中途切分。
- 行內樣式絕不會跨區塊切斷；渲染器會在下一個區塊開頭
  重新開啟尚未結束的樣式。

如需了解各頻道的分塊邊界和
傳遞行為，請參閱[串流與分塊](/concepts/streaming)。

## 連結政策

- **Slack：** `[label](url)` -> `<url|label>`；裸露 URL 維持原樣。
- **Telegram：** `[label](url)` -> `<a href="url">label</a>`（HTML 解析模式）。
- **Signal：** `[label](url)` -> `label (url)`，除非標籤已
  與 URL 相符。

## 劇透

Signal 會解析劇透標記（`||spoiler||`，對應至 `SPOILER`
樣式範圍），Telegram 也會解析（對應至 `<tg-spoiler>`）。其他頻道會將
`||...||` 視為純文字。

## 新增或更新頻道格式器

1. **解析一次**：使用 `markdownToIR(...)`，並傳入適合頻道的
   選項（`autolink`、`headingStyle`、`blockquotePrefix`、`tableMode`）。
2. **渲染**：使用 `renderMarkdownWithMarkers(...)` 和樣式標記對應表（或
   對 Signal 之類的傳輸使用自訂樣式範圍邏輯）。
3. **分塊**：在渲染各區塊前使用 `chunkMarkdownIR(...)` 或
   `renderMarkdownIRChunksWithinLimit(...)`。
4. **連接轉接器**：讓傳出訊息的傳送路徑呼叫新的分塊器和渲染器。
5. **測試**：使用格式測試；若頻道會分塊，另加傳出訊息傳遞測試。

## 常見陷阱

- Slack 尖括號權杖（`<@U123>`、`<#C123>`、`<https://...>`）必須
  在逸出處理後仍予以保留；原始 HTML 仍需安全地逸出。
- Telegram HTML 必須逸出標籤外的文字，以避免標記損壞。
- Signal 樣式範圍使用 UTF-16 位移量，而非碼點位移量。
- 保留圍欄程式碼區塊的結尾換行，讓結束標記
  獨占一行。

## 相關內容

<CardGroup cols={2}>
  <Card title="串流與分塊" href="/zh-TW/concepts/streaming" icon="bars-staggered">
    傳出串流行為、分塊邊界和各頻道專用的傳遞方式。
  </Card>
  <Card title="系統提示詞" href="/zh-TW/concepts/system-prompt" icon="message-lines">
    模型在對話前看到的內容，包括注入的工作區檔案。
  </Card>
</CardGroup>
