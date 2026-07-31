---
read_when:
    - 你想要分析來自代理程式的 PDF 檔案
    - 你需要確切的 PDF 工具參數與限制
    - 你正在偵錯原生 PDF 模式與擷取備援機制的差異
summary: 使用原生供應商支援及擷取備援來分析一份或多份 PDF 文件
title: PDF 工具
x-i18n:
    generated_at: "2026-07-26T08:46:48Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: e0e5b897e1e122af4b2f6f9a3eaeb73f6e93af1051d306ad82539b258de90c49
    source_path: tools/pdf.md
    workflow: 16
---

`pdf` 會分析一或多份 PDF 文件並回傳文字。它在 Anthropic 和 Google 模型上使用原生文件輸入，對其他所有供應商則改用文字／影像擷取。

## 可用性

只有當 OpenClaw 能為代理程式解析出支援 PDF 的模型時，才會註冊此工具。解析順序如下：

1. `agents.defaults.pdfModel`（明確指定的主要模型／備援模型）
2. `agents.defaults.imageModel`（明確指定的主要模型／備援模型）
3. 代理程式解析出的工作階段／預設模型，前提是其供應商支援原生 PDF 輸入（Anthropic、Google），或已設定視覺模型
4. 自動偵測具有可用認證且支援影像／視覺的供應商，優先選用原生支援 PDF 的供應商

每個備援候選模型都會在使用前檢查認證，因此已設定的 `provider/model` 只有在 OpenClaw 能為代理程式向該供應商進行驗證時才算有效。若無法解析出可用模型，便不會公開 `pdf` 工具。

## 輸入參考

<ParamField path="pdf" type="string">
一個 PDF 路徑或 URL。
</ParamField>

<ParamField path="pdfs" type="string[]">
多個 PDF 路徑或 URL，總計最多 10 個。
</ParamField>

<ParamField path="prompt" type="string" default="Analyze this PDF document.">
分析提示詞。
</ParamField>

<ParamField path="pages" type="string">
頁面篩選條件，例如 `1-5` 或 `1,3,7-9`。原生供應商模式不支援此項目。
</ParamField>

<ParamField path="password" type="string">
加密 PDF 的密碼。套用至要求中的每份 PDF；僅供擷取備援模式使用。
</ParamField>

<ParamField path="model" type="string">
選用的模型覆寫值，格式為 `provider/model`。
</ParamField>

<ParamField path="maxBytesMb" type="number">
每份 PDF 的大小上限（MB）。預設為 `agents.defaults.pdfMaxMb`；若未設定，則為 `10`。
</ParamField>

注意事項：

- `pdf` 和 `pdfs` 會在載入前合併並去除重複項目；至少需要提供其中一項。
- `pages` 會解析為從 1 開始的頁碼、去除重複、排序，並限制在 `agents.defaults.pdfMaxPages`（預設為 `20`）範圍內。若範圍未符合任何界內頁面，會在呼叫模型前發生錯誤。

## 支援的 PDF 參照

- 本機檔案路徑（包括 `~` 展開）
- `file://` URL
- `http://` 和 `https://` URL
- 由 OpenClaw 管理的傳入參照，例如 `media://inbound/<id>`

其他 URI 配置（例如 `ftp://`）會回傳 `details.error = "unsupported_pdf_reference"`。當工具在沙箱中執行時，會拒絕遠端 `http(s)` URL。啟用僅限工作區的檔案政策時，會拒絕允許根目錄以外的本機路徑；仍允許受管理的傳入參照，以及 OpenClaw 傳入媒體儲存區下的重播路徑。

## 執行模式

### 原生供應商模式

用於供應商 `anthropic` 和 `google`（目前僅有這些供應商宣告支援原生 PDF 文件）。每個檔案的原始 PDF 位元組會以原生文件／內嵌 PDF 部分直接傳送至供應商 API。

限制：

- 不支援 `pages`；若已設定，工具會擲回 `pages is not supported with native PDF providers`。
- 不支援 `password`；若已設定，工具會擲回 `password is not supported with native PDF providers`。加密 PDF 請使用非原生模型。

### 擷取備援模式

用於其他所有供應商。

1. 透過隨附的 `document-extract` 外掛，從所選頁面（最多 `agents.defaults.pdfMaxPages` 頁，預設為 `20`）擷取文字；該外掛使用 `clawpdf` 套件（PDFium WebAssembly）進行文字與影像擷取。
2. 若擷取出的文字少於 `200` 個字元，會將相同頁面算繪為 PNG 影像。算繪預算總計為 `4,000,000` 像素，由所有需要影像的頁面共用（依每個剩餘頁面按比例分配，而非每頁各自分配），因此已有足夠文字的文字頁面會完全略過算繪。
3. 將擷取的文字（以及任何算繪出的影像）連同提示詞傳送至所選模型。

詳細資訊：

- 加密 PDF 會使用頂層 `password` 參數開啟。
- 若模型不支援影像輸入，且沒有可擷取的文字，工具會發生錯誤。
- 若影像算繪失敗，OpenClaw 會捨棄影像，並繼續使用擷取出的文字。
- 若目標模型僅支援文字，而擷取過程產生了影像，OpenClaw 會捨棄影像，只傳送文字。

## 設定

```json5
{
  agents: {
    defaults: {
      pdfModel: {
        primary: "anthropic/claude-opus-4-6",
        fallbacks: ["openai/gpt-5.4-mini"],
      },
      pdfMaxBytesMb: 10,
      pdfMaxPages: 20,
    },
  },
}
```

| 鍵                            | 預設值 | 意義                                                                                          |
| ----------------------------- | ------- | ----------------------------------------------------------------------------------------- |
| `agents.defaults.pdfModel`    | 未設定   | 明確指定主要／備援 PDF 模型；依序備援至 `imageModel`，再使用工作階段模型。 |
| `agents.defaults.pdfMaxMb`    | `10`    | 每份 PDF 的大小上限（MB）。                                                                   |
| `agents.defaults.pdfMaxPages` | `20`    | 每份 PDF 處理的最大頁數。                                                              |

如需完整欄位詳細資訊，請參閱[設定參考](/zh-TW/gateway/config-agents#agent-defaults)。

## 輸出詳細資訊

工具會在 `content[0].text` 中回傳文字，並在 `details` 中回傳結構化中繼資料。

常見的 `details` 欄位：

- `model`：解析出的模型參照（`provider/model`）
- `native`：原生供應商模式為 `true`，備援模式為 `false`
- `attempts`：成功前失敗的備援嘗試

路徑欄位：

- 單一 PDF 輸入：`details.pdf`
- 多個 PDF 輸入：`details.pdfs[]`，包含 `pdf` 項目
- 沙箱路徑重寫中繼資料（如適用）：`rewrittenFrom`

## 錯誤行為

| 條件                              | 結果                                                         |
| --------------------------------- | -------------------------------------------------------------- |
| 未提供 PDF 輸入                   | 擲回 `pdf required: provide a path or URL to a PDF document` |
| 超過 10 份 PDF                    | `details.error = "too_many_pdfs"`                              |
| 不支援的參照配置                  | `details.error = "unsupported_pdf_reference"`                  |
| 對原生供應商使用 `pages`    | 擲回 `pages is not supported with native PDF providers`      |
| 對原生供應商使用 `password` | 擲回 `password is not supported with native PDF providers`   |

## 範例

單一 PDF：

```json
{
  "pdf": "/tmp/report.pdf",
  "prompt": "以 5 個項目符號摘要這份報告"
}
```

多個 PDF：

```json
{
  "pdfs": ["/tmp/q1.pdf", "/tmp/q2.pdf"],
  "prompt": "比較這兩份文件中的風險與時程變更"
}
```

經頁面篩選的備援模型：

```json
{
  "pdf": "https://example.com/report.pdf",
  "pages": "1-3,7",
  "model": "openai/gpt-5.4-mini",
  "prompt": "僅擷取影響客戶的事件"
}
```

使用擷取備援模式的加密 PDF：

```json
{
  "pdf": "/tmp/locked.pdf",
  "password": "example-password",
  "model": "openai/gpt-5.4-mini",
  "prompt": "摘要這份合約"
}
```

## 相關內容

- [工具概覽](/zh-TW/tools) - 所有可用的代理程式工具
- [設定參考](/zh-TW/gateway/config-agents#agent-defaults) - pdfMaxBytesMb 和 pdfMaxPages 設定
