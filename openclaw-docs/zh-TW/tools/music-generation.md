---
read_when:
    - 透過代理程式產生音樂或音訊
    - 設定音樂生成提供者與模型
    - 瞭解 `music_generate` 工具參數
sidebarTitle: Music generation
summary: 透過 ComfyUI、fal、Google Lyria、MiniMax 和 OpenRouter 工作流程，使用 music_generate 產生音樂
title: 音樂生成
x-i18n:
    generated_at: "2026-07-26T08:16:38Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 3f2a8a4a36e47839c7896046a556f7bf84f6c168492e2de46736635fe2a9358e
    source_path: tools/music-generation.md
    workflow: 16
---

`music_generate` 工具會透過共用的音樂生成功能建立音樂或音訊，後端支援 ComfyUI、fal、Google、MiniMax 與 OpenRouter。

<Note>
只有在至少有一個音樂生成提供者可用時，才會顯示 `music_generate`：明確的 `agents.defaults.mediaModels.music` 設定，或已設定驗證的提供者（例如已設定 API 金鑰）。
</Note>

對於以工作階段為基礎的代理程式執行，`music_generate` 會以背景工作啟動、在工作帳本中追蹤進度，並在音軌就緒時喚醒代理程式，讓它通知使用者並附上完成的音訊。完成代理程式會遵循工作階段的可見回覆合約：設定時自動傳送最終回覆，或在工作階段要求使用訊息工具時使用 `message(action="send")`。如果請求者的工作階段未啟用，或喚醒失敗，且回覆中仍缺少生成的音訊，OpenClaw 會傳送僅包含缺漏音訊的冪等直接備援訊息。

## 快速開始

<Tabs>
  <Tab title="共用提供者後端">
    <Steps>
      <Step title="設定驗證">
        為至少一個提供者設定 API 金鑰，例如
        `GEMINI_API_KEY` 或 `MINIMAX_API_KEY`。
      </Step>
      <Step title="選擇預設模型（選用）">
        ```json5
        {
          agents: {
            defaults: {
              musicGenerationModel: {
                primary: "google/lyria-3-clip-preview",
              },
            },
          },
        }
        ```
      </Step>
      <Step title="向代理程式提出要求">
        _「生成一首關於夜間駕車穿越霓虹城市、節奏輕快的合成器流行音樂。」_

        代理程式會自動呼叫 `music_generate`，不需要將工具加入允許清單。
      </Step>
    </Steps>

    如果不是以工作階段為基礎的代理程式執行（直接／本機情境），工具會內嵌執行，並在相同的工具結果中傳回最終媒體路徑。

  </Tab>
  <Tab title="ComfyUI 工作流程">
    <Steps>
      <Step title="設定工作流程">
        使用工作流程 JSON 及提示詞／輸出節點設定 `plugins.entries.comfy.config.music`。
      </Step>
      <Step title="雲端驗證（選用）">
        若使用 Comfy Cloud，請設定 `COMFY_API_KEY` 或 `COMFY_CLOUD_API_KEY`。
      </Step>
      <Step title="呼叫工具">
        ```text
        /tool music_generate prompt="帶有柔和磁帶質感的溫暖環境合成器循環樂段"
        ```
      </Step>
    </Steps>
  </Tab>
</Tabs>

提示詞範例：

```text
生成一首帶有柔和弦樂且無人聲的電影感鋼琴曲。
```

```text
生成一段關於在日出時發射火箭、充滿活力的晶片音樂循環樂段。
```

使用 `action: "list"` 檢查可用的提供者／模型，並使用 `action: "status"` 檢查以工作階段為基礎且目前啟用的音樂工作：

```text
/tool music_generate action=list
/tool music_generate action=status
```

直接生成範例：

```text
/tool music_generate prompt="帶有黑膠質感與柔和雨聲的夢幻 lo-fi 嘻哈音樂" instrumental=true
```

## 支援的提供者

| 提供者   | 預設模型                | 參考輸入 | 支援的控制項                                    | 驗證                                   |
| ---------- | ---------------------------- | ---------------- | ----------------------------------------------------- | -------------------------------------- |
| ComfyUI    | `workflow`                   | 最多 1 張圖片    | 由工作流程定義的音樂或音訊                       | `COMFY_API_KEY`、`COMFY_CLOUD_API_KEY` |
| fal        | `fal-ai/minimax-music/v2.6`  | 無             | `lyrics`、`instrumental`、`durationSeconds`、`format` | `FAL_KEY` 或 `FAL_API_KEY`             |
| Google     | `lyria-3-clip-preview`       | 最多 10 張圖片  | `lyrics`、`instrumental`、`format`                    | `GEMINI_API_KEY`、`GOOGLE_API_KEY`     |
| MiniMax    | `music-2.6`                  | 無             | `lyrics`、`instrumental`、`format`（僅限 mp3）         | `MINIMAX_API_KEY` 或 MiniMax OAuth     |
| OpenRouter | `google/lyria-3-pro-preview` | 最多 1 張圖片    | `lyrics`、`instrumental`、`durationSeconds`、`format` | `OPENROUTER_API_KEY`                   |

MiniMax 會註冊兩個共用相同模型的提供者 ID：`minimax` 用於 API 金鑰驗證，`minimax-portal` 用於 OAuth。模型參照會依循驗證路徑（`minimax/music-2.6` 與 `minimax-portal/music-2.6`）；請參閱 [MiniMax](/zh-TW/providers/minimax#music-generation)。

除了預設的 MiniMax 後端模型外，fal 也提供 `fal-ai/ace-step/prompt-to-audio`（wav、無歌詞、無純器樂切換選項）與 `fal-ai/stable-audio-25/text-to-audio`（wav、僅限提示詞）。Google 的預設 `lyria-3-clip-preview` 僅輸出 mp3；`lyria-3-pro-preview` 也支援 wav。MiniMax 也提供 `music-2.6-free`、`music-cover` 與 `music-cover-free`。OpenRouter 也提供 `google/lyria-3-clip-preview`。

### 功能矩陣

`music_generate`、合約測試與共用即時掃描所使用的明確模式合約：

| 提供者   | `generate` | `edit` | 編輯限制 | 共用即時測試通道                                                         |
| ---------- | :--------: | :----: | ---------- | ------------------------------------------------------------------------- |
| ComfyUI    |     ✓      |   ✓    | 1 張圖片    | 不包含在共用掃描中；由 `extensions/comfy/comfy.live.test.ts` 涵蓋 |
| fal        |     ✓      |   —    | 無       | `generate`                                                                |
| Google     |     ✓      |   ✓    | 10 張圖片  | `generate`、`edit`                                                        |
| MiniMax    |     ✓      |   —    | 無       | `generate`                                                                |
| OpenRouter |     ✓      |   ✓    | 1 張圖片    | `generate`、`edit`                                                        |

## 工具參數

<ParamField path="prompt" type="string" required>
  音樂生成提示詞。`action: "generate"` 必須提供此參數。
</ParamField>
<ParamField path="action" type='"generate" | "status" | "list"' default="generate">
  `"status"` 會傳回目前的工作階段工作；`"list"` 會檢查提供者。
</ParamField>
<ParamField path="model" type="string">
  覆寫提供者／模型（例如 `google/lyria-3-pro-preview`、
  `comfy/workflow`）。
</ParamField>
<ParamField path="lyrics" type="string">
  當提供者支援明確輸入歌詞時，可選擇提供歌詞。
</ParamField>
<ParamField path="instrumental" type="boolean">
  當提供者支援時，要求僅輸出純器樂。
</ParamField>
<ParamField path="image" type="string">
  單一參考圖片路徑或 URL。
</ParamField>
<ParamField path="images" type="string[]">
  多張參考圖片（支援的提供者最多可使用 10 張）。
</ParamField>
<ParamField path="durationSeconds" type="number">
  當提供者支援時，以秒為單位指定目標時長提示。
</ParamField>
<ParamField path="format" type='"mp3" | "wav"'>
  當提供者支援時，指定輸出格式提示。
</ParamField>
<ParamField path="filename" type="string">輸出檔名提示。</ParamField>

<Note>
並非所有提供者都支援所有參數。OpenClaw 仍會在提交前驗證輸入數量等硬性限制。當提供者支援指定時長，但其最大值短於要求的值時，OpenClaw 會將其限制為最接近的支援時長。若選取的提供者或模型無法採用確實不受支援的選用提示，系統會忽略這些提示並發出警告。工具結果會報告實際套用的設定；`details.normalization` 會記錄從要求值到套用值的任何對應。
</Note>

提供者要求逾時僅能由操作人員設定。若已設定，OpenClaw 會使用 `agents.defaults.mediaModels.music.timeoutMs`，將低於 120000ms 的值提高至 120000ms；否則，提供者要求的預設逾時為 300000ms。

## 非同步行為

以工作階段為基礎的音樂生成會以背景工作執行：

- **背景工作：**`music_generate` 會建立背景工作、立即傳回已啟動／工作回應，並於稍後在後續代理程式訊息中張貼完成的音軌。
- **防止重複：**工作處於 `queued` 或 `running` 狀態時，同一工作階段中後續的 `music_generate` 呼叫會傳回工作狀態，而不是啟動另一個生成工作。使用 `action: "status"` 可明確檢查。最近完成且相符的要求也會在 2 分鐘內進行去重。
- **狀態查詢：**`openclaw tasks list` 或 `openclaw tasks show <taskId>` 會檢查已排入佇列、執行中及終止狀態。
- **完成喚醒：**OpenClaw 會將內部完成事件注入回相同的工作階段，讓模型自行撰寫面向使用者的後續回覆。
- **提示詞提醒：**如果音樂工作已在執行，同一工作階段中後續的使用者／手動回合會收到簡短的執行階段提示，避免模型盲目地再次呼叫 `music_generate`。
- **無工作階段備援：**沒有實際代理程式工作階段的直接／本機情境會內嵌執行，並在相同回合中傳回最終音訊結果。

### 工作生命週期

音樂工作會呈現與一般工作登錄檔相同的狀態（如需包含 `timed_out`、`cancelled` 與 `lost` 在內的完整狀態機，請參閱[背景工作](/zh-TW/automation/tasks#task-lifecycle)）。大多數音樂執行會經過：

| 狀態       | 意義                                                                                        |
| ----------- | ---------------------------------------------------------------------------------------------- |
| `queued`    | 工作已建立，正在等待提供者接受。                                           |
| `running`   | 提供者正在處理（通常需要 30 秒至 3 分鐘，視提供者與時長而定）。 |
| `succeeded` | 音軌已就緒；代理程式會被喚醒並將其張貼至對話。                                 |
| `failed`    | 提供者發生錯誤或逾時；代理程式會被喚醒並取得錯誤詳細資料。                                 |

從命令列介面檢查狀態：

```bash
openclaw tasks list
openclaw tasks show <taskId>
openclaw tasks cancel <taskId>
```

## 設定

### 模型選擇

```json5
{
  agents: {
    defaults: {
      musicGenerationModel: {
        primary: "google/lyria-3-clip-preview",
        fallbacks: ["fal/fal-ai/minimax-music/v2.6", "minimax/music-2.6"],
      },
    },
  },
}
```

### 提供者選擇順序

OpenClaw 會依照下列順序嘗試提供者：

1. 工具呼叫中的 `model` 參數（如果代理程式有指定）。
2. 設定中的 `musicGenerationModel.primary`。
3. 依序使用 `musicGenerationModel.fallbacks`。
4. 僅使用具備驗證資訊的提供者預設值進行自動偵測：
   - 如果目前的預設文字模型提供者也提供音樂生成功能，則優先使用；
   - 其餘已註冊的音樂生成提供者，依提供者 ID 的字母順序排列。

如果提供者失敗，系統會自動嘗試下一個候選項目。如果全部失敗，錯誤會包含每次嘗試的詳細資料。

一律啟用已驗證提供者之間的自動備援。每次呼叫指定的 `model` 仍具有最高決定權。

## 提供者注意事項

<AccordionGroup>
  <Accordion title="ComfyUI">
    由工作流程驅動，並取決於已設定的圖形以及提示詞／輸出欄位的節點對應。
    內建的 `comfy` 外掛會透過音樂生成提供者登錄檔，接入共用的
    `music_generate` 工具。
  </Accordion>
  <Accordion title="fal">
    透過共用的提供者驗證路徑使用 fal 模型端點。內建提供者預設使用
    `fal-ai/minimax-music/v2.6`，並且也公開
    `fal-ai/ace-step/prompt-to-audio` 和
    `fal-ai/stable-audio-25/text-to-audio`，以處理提示詞轉音訊請求。
    歌詞與純音樂模式僅適用於 MiniMax 模型；另外兩個模型僅支援提示詞。
  </Accordion>
  <Accordion title="Google (Lyria 3)">
    使用 Lyria 3 批次生成。目前的內建流程支援提示詞、選用的歌詞文字，以及選用的參考圖片。
    預設的 `lyria-3-clip-preview` 模型僅輸出 mp3；
    `lyria-3-pro-preview` 模型也支援 wav。
  </Accordion>
  <Accordion title="MiniMax">
    使用批次 `music_generation` 端點。支援提示詞、選用的歌詞、純音樂模式，
    以及透過 `minimax` API 金鑰驗證或 `minimax-portal` OAuth
    輸出 mp3。也公開 `music-2.6-free`、
    `music-cover` 和 `music-cover-free` 模型。
  </Accordion>
  <Accordion title="OpenRouter">
    使用已啟用串流的 OpenRouter 聊天補全音訊輸出。內建提供者預設使用
    `google/lyria-3-pro-preview`，並且也公開
    `openrouter/google/lyria-3-clip-preview`。
  </Accordion>
</AccordionGroup>

## 選擇正確的路徑

- 若你需要模型選擇、提供者容錯移轉，以及內建的非同步任務／狀態流程，請選擇**共用提供者支援路徑**。
- 若你需要自訂工作流程圖形，或使用不屬於共用內建音樂功能的提供者，請選擇**外掛路徑（ComfyUI）**。

若你正在偵錯 ComfyUI 特有的行為，請參閱
[ComfyUI](/zh-TW/providers/comfy)。若你正在偵錯共用提供者的行為，請先查看 [fal](/zh-TW/providers/fal)、[Google (Gemini)](/zh-TW/providers/google)、
[MiniMax](/zh-TW/providers/minimax) 或 [OpenRouter](/zh-TW/providers/openrouter)。

## 提供者功能模式

共用音樂生成合約支援明確的模式宣告：

- `generate` 用於僅依提示詞生成。
- `edit` 用於請求包含一張或多張參考圖片時。

新的提供者實作應優先使用明確的模式區塊：

```typescript
capabilities: {
  generate: {
    maxTracks: 1,
    supportsLyrics: true,
    supportsFormat: true,
  },
  edit: {
    enabled: true,
    maxTracks: 1,
    maxInputImages: 1,
    supportsFormat: true,
  },
}
```

`maxInputImages`、`supportsLyrics` 和
`supportsFormat` 等舊版扁平欄位，**不足以**表明支援編輯。提供者應明確宣告
`generate` 和 `edit`，讓即時測試、合約測試以及共用的
`music_generate` 工具能以確定性方式驗證模式支援。

## 即時測試

選用的共用內建提供者（fal、Google、MiniMax、OpenRouter）即時涵蓋範圍：

```bash
OPENCLAW_LIVE_TEST=1 pnpm test:live -- extensions/music-generation-providers.live.test.ts
```

作用相同的儲存庫包裝命令，會執行同一個測試檔案：

```bash
pnpm test:live:media:music
```

此即時測試檔案預設會優先使用已匯出的提供者環境變數，而非儲存的驗證設定檔；
當提供者啟用編輯模式時，會同時執行 `generate` 和已宣告的
`edit` 涵蓋範圍。目前的涵蓋範圍：

- `google`：`generate` 加上 `edit`
- `fal`：僅 `generate`
- `minimax`：僅 `generate`
- `openrouter`：`generate` 加上 `edit`
- `comfy`：獨立的 Comfy 即時測試涵蓋範圍，不屬於共用提供者全面測試

選用的內建 ComfyUI 音樂路徑即時涵蓋範圍：

```bash
OPENCLAW_LIVE_TEST=1 COMFY_LIVE_TEST=1 pnpm test:live -- extensions/comfy/comfy.live.test.ts
```

設定相關區段後，Comfy 即時測試檔案也會涵蓋 Comfy 圖片和影片工作流程。

## 相關內容

- [背景任務](/zh-TW/automation/tasks) — 追蹤分離執行的 `music_generate` 任務
- [ComfyUI](/zh-TW/providers/comfy)
- [設定參考](/zh-TW/gateway/config-agents#agent-defaults) — `musicGenerationModel` 設定
- [Google (Gemini)](/zh-TW/providers/google)
- [MiniMax](/zh-TW/providers/minimax)
- [模型](/zh-TW/concepts/models) — 模型設定與容錯移轉
- [工具概覽](/zh-TW/tools)
