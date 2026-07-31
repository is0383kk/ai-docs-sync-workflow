---
read_when:
    - 透過代理程式產生影片
    - 設定影片生成供應商和模型
    - 了解 `video_generate` 工具參數
sidebarTitle: Video generation
summary: 透過 16 個供應商後端，使用 `video_generate` 從文字、圖片或影片參考素材生成影片
title: 影片生成
x-i18n:
    generated_at: "2026-07-26T08:50:47Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 4afc9338fdfdc269b50b949b6d1a186e3a2064ed4ee40a41722efea40ae791aa
    source_path: tools/video-generation.md
    workflow: 16
---

OpenClaw 代理程式透過 `video_generate`，從文字提示、參考圖片或
現有影片產生影片。支援十六種供應商後端；代理程式會根據設定和
可用的 API 金鑰自動選擇合適的後端。

<Note>
只有在至少有一個影片產生供應商可用時，才會顯示
`video_generate`。如果代理程式工具中沒有此項目，請設定供應商 API 金鑰或
設定 `agents.defaults.mediaModels.video`。
</Note>

`video_generate` 有三種執行階段模式，會根據呼叫中的參考輸入
決定：

- `generate` - 無參考媒體（文字轉影片）。
- `imageToVideo` - 一張或多張參考圖片。
- `videoToVideo` - 一部或多部參考影片。

供應商可支援這些模式的任意子集。工具會在提交前驗證
目前模式，並在 `action=list` 中回報支援的模式。

## 快速開始

<Steps>
  <Step title="設定驗證">
    為任一支援的供應商設定 API 金鑰：

    ```bash
    export GEMINI_API_KEY="your-key"
    ```

  </Step>
  <Step title="選擇預設模型（選用）">
    ```bash
    openclaw config set agents.defaults.mediaModels.video.primary "google/veo-3.1-fast-generate-preview"
    ```
  </Step>
  <Step title="要求代理程式">
    > 產生一段 5 秒的電影風格影片，內容是一隻友善的龍蝦在夕陽下衝浪。

    代理程式會自動呼叫 `video_generate`。不需要將工具加入允許清單。

  </Step>
</Steps>

## 非同步產生的運作方式

影片產生採非同步方式：

1. OpenClaw 將要求提交給供應商，並立即傳回工作 ID。
2. 供應商在背景處理工作（通常需要 30 秒到數分鐘，視供應商和解析度而定；由慢速佇列支援的供應商可能會執行至設定的逾時時間）。
3. 影片準備就緒時，OpenClaw 會以內部完成事件喚醒同一個工作階段。
4. 代理程式會透過工作階段的一般可見回覆模式回報：
   自動最終回覆，或在工作階段要求使用訊息工具時透過 `message(action="send")` 回覆。
   如果要求者的工作階段未啟用，或喚醒失敗且完成回覆中仍缺少產生的媒體，OpenClaw 會
   直接傳送具等冪性的後援訊息及媒體。

工作進行期間，同一工作階段中重複的 `video_generate` 呼叫會
傳回目前工作狀態，而不會開始另一個產生工作。使用 `action: "status"`
可在不觸發新產生工作的情況下檢查，或從命令列介面使用 `openclaw tasks list` /
`openclaw tasks show <lookup>`（請參閱[背景工作](/zh-TW/automation/tasks)）。

在不以工作階段為基礎的代理程式執行環境外（例如直接呼叫工具），
工具會退回行內產生方式，並在同一輪中傳回最終媒體路徑。

當供應商傳回位元組時，產生的影片檔案會儲存在 OpenClaw 管理的媒體儲存空間中。
預設上限為 16MB（共用影片媒體限制）；`agents.defaults.mediaMaxMb` 可提高此上限，
以處理較大的算繪結果。如果供應商也傳回託管的輸出 URL，而本機持久化因檔案過大
遭拒，OpenClaw 會改為傳送該 URL，而不會讓工作失敗。

### 工作生命週期

| 狀態       | 意義                                                                                                |
| ----------- | ------------------------------------------------------------------------------------------------------ |
| `queued`    | 工作已建立，正在等待供應商接受。                                                   |
| `running`   | 供應商正在處理（通常需要 30 秒到數分鐘，視供應商和解析度而定）。 |
| `succeeded` | 影片已準備就緒；代理程式會被喚醒，並將影片發布至對話中。                                         |
| `failed`    | 供應商發生錯誤或逾時；代理程式會被喚醒並提供錯誤詳細資料。                                         |

從命令列介面檢查狀態：

```bash
openclaw tasks list
openclaw tasks show <lookup>
openclaw tasks cancel <lookup>
```

## 支援的供應商

| 供應商              | 預設模型                   | 文字 | 圖片參考                                            | 影片參考                                       | 驗證                                     |
| --------------------- | ------------------------------- | :--: | ---------------------------------------------------- | ----------------------------------------------- | ---------------------------------------- |
| Alibaba               | `wan2.6-t2v`                    |  ✓   | 是（遠端 URL）                                     | 是（遠端 URL）                                | `MODELSTUDIO_API_KEY`                    |
| BytePlus（內建）    | `seedance-1-0-pro-250528`       |  ✓   | 最多 2 張圖片（第一幀 + 最後一幀）                  | -                                               | `BYTEPLUS_API_KEY`                       |
| BytePlus 1.5 外掛   | `seedance-1-5-pro-251215`       |  ✓   | 最多 2 張圖片（透過角色指定第一幀 + 最後一幀）         | -                                               | `BYTEPLUS_API_KEY`                       |
| BytePlus Seedance 2.0 | `dreamina-seedance-2-0-260128`  |  ✓   | 最多 9 張參考圖片                             | 最多 3 部影片                                  | `BYTEPLUS_API_KEY`                       |
| ComfyUI               | `workflow`                      |  ✓   | 1 張圖片                                              | -                                               | `COMFY_API_KEY` 或 `COMFY_CLOUD_API_KEY` |
| DeepInfra             | `Pixverse/Pixverse-T2V`         |  ✓   | -                                                    | -                                               | `DEEPINFRA_API_KEY`                      |
| fal                   | `fal-ai/minimax/video-01-live`  |  ✓   | 1 張圖片；使用 Seedance 參考轉影片時最多 9 張    | 使用 Seedance 參考轉影片時最多 3 部影片 | `FAL_KEY`                                |
| Google                | `veo-3.1-fast-generate-preview` |  ✓   | 1 張圖片                                              | 1 部影片                                         | `GEMINI_API_KEY`                         |
| MiniMax               | `MiniMax-Hailuo-2.3`            |  ✓   | 1 張圖片                                              | -                                               | `MINIMAX_API_KEY` 或 MiniMax OAuth       |
| OpenAI                | `sora-2`                        |  ✓   | 1 張圖片                                              | 1 部影片                                         | `OPENAI_API_KEY`                         |
| OpenRouter            | `google/veo-3.1-fast`           |  ✓   | 最多 4 張圖片（第一幀／最後一幀或參考圖片）      | -                                               | `OPENROUTER_API_KEY`                     |
| Qwen                  | `wan2.6-t2v`                    |  ✓   | 是（遠端 URL）                                     | 是（遠端 URL）                                | `QWEN_API_KEY`                           |
| Runway                | `gen4.5`                        |  ✓   | 1 張圖片                                              | 1 部影片                                         | `RUNWAYML_API_SECRET`                    |
| Together              | `Wan-AI/Wan2.2-T2V-A14B`        |  ✓   | 僅限 `Wan-AI/Wan2.2-I2V-A14B`                        | -                                               | `TOGETHER_API_KEY`                       |
| Vydra                 | `veo3`                          |  ✓   | 1 張圖片（`kling`）                                    | -                                               | `VYDRA_API_KEY`                          |
| xAI                   | `grok-imagine-video`            |  ✓   | Classic：1 個第一幀或 7 張參考圖片；1.5：1 幀 | Classic：1 部影片                                | `XAI_API_KEY`                            |

部分供應商接受其他或替代的 API 金鑰環境變數。詳情請參閱
各個[供應商頁面](#related)。

執行 `video_generate action=list`，即可在執行階段檢查可用的供應商、模型和
執行階段模式。

### 功能矩陣

`video_generate`、契約測試和
共用即時掃描所使用的明確模式契約：

| 供應商   | `generate` | `imageToVideo` | `videoToVideo` | 目前的共用即時測試通道                                                                                                                 |
| ---------- | :--------: | :------------: | :------------: | --------------------------------------------------------------------------------------------------------------------------------------- |
| Alibaba    |     ✓      |       ✓        |       ✓        | `generate`、`imageToVideo`；略過 `videoToVideo`，因為此供應商需要遠端 `http(s)` 影片 URL                              |
| BytePlus   |     ✓      |       ✓        |       -        | `generate`、`imageToVideo`                                                                                                              |
| ComfyUI    |     ✓      |       ✓        |       -        | 不在共用掃描中；工作流程特定的涵蓋範圍由 Comfy 測試負責                                                              |
| DeepInfra  |     ✓      |       -        |       -        | `generate`；原生 DeepInfra 影片結構描述在外掛契約中為文字轉影片                                                     |
| fal        |     ✓      |       ✓        |       ✓        | `generate`、`imageToVideo`；僅在使用 Seedance 參考轉影片時執行 `videoToVideo`                                                  |
| Google     |     ✓      |       ✓        |       ✓        | `generate`、`imageToVideo`；略過共用 `videoToVideo`，因為目前以緩衝區為基礎的 Gemini/Veo 掃描不接受該輸入 |
| MiniMax    |     ✓      |       ✓        |       -        | `generate`、`imageToVideo`                                                                                                              |
| OpenAI     |     ✓      |       ✓        |       ✓        | `generate`、`imageToVideo`；略過共用 `videoToVideo`，因為此組織／輸入路徑目前需要供應商端的影片編輯存取權   |
| OpenRouter |     ✓      |       ✓        |       -        | `generate`、`imageToVideo`                                                                                                              |
| Qwen       |     ✓      |       ✓        |       ✓        | `generate`、`imageToVideo`；略過 `videoToVideo`，因為此供應商需要遠端 `http(s)` 影片 URL                              |
| Runway     |     ✓      |       ✓        |       ✓        | `generate`、`imageToVideo`；僅在所選模型為 `runway/gen4_aleph` 時執行 `videoToVideo`                                     |
| Together   |     ✓      |       ✓        |       -        | `generate`、`imageToVideo`                                                                                                              |
| Vydra      |     ✓      |       ✓        |       -        | `generate`；略過共用 `imageToVideo`，因為內建的 `veo3` 僅支援文字，而內建的 `kling` 需要遠端圖片 URL           |
| xAI        |     ✓      |       ✓        |       ✓        | Classic 支援所有模式；Video 1.5 僅支援圖片轉影片；遠端 MP4 輸入使 `videoToVideo` 不納入共用掃描             |

## 工具參數

### 必填

<ParamField path="prompt" type="string" required>
  要產生之影片的文字描述。`action: "generate"` 必填。
</ParamField>

### 內容輸入

<ParamField path="image" type="string">單一參考圖片（路徑或 URL）。</ParamField>
<ParamField path="images" type="string[]">多張參考圖片（最多 9 張）。</ParamField>
<ParamField path="imageRoles" type="string[]">
選填的位置角色提示，與合併後的圖片清單平行對應。
標準值：`first_frame`、`last_frame`、`reference_image`。
</ParamField>
<ParamField path="video" type="string">單一參考影片（路徑或 URL）。</ParamField>
<ParamField path="videos" type="string[]">多段參考影片（最多 4 段）。</ParamField>
<ParamField path="videoRoles" type="string[]">
選填的位置角色提示，與合併後的影片清單平行對應。
標準值：`reference_video`。
</ParamField>
<ParamField path="audioRef" type="string">
單一參考音訊（路徑或 URL）。當供應商支援音訊輸入時，
用於背景音樂或語音參考。
</ParamField>
<ParamField path="audioRefs" type="string[]">多段參考音訊（最多 3 段）。</ParamField>
<ParamField path="audioRoles" type="string[]">
選填的位置角色提示，與合併後的音訊清單平行對應。
標準值：`reference_audio`。
</ParamField>

<Note>
角色提示會原樣轉送給供應商。標準值來自
`VideoGenerationAssetRole` 聯集，但供應商可能接受其他
角色字串。`*Roles` 陣列的項目數不得超過
對應的參考清單；差一錯誤會以明確錯誤訊息告知失敗。
使用空字串可將欄位保留為未設定。若使用 xAI，請將每張圖片的角色設為
`reference_image`，以使用其 `reference_images` 產生模式；若要使用
單張圖片的圖片轉影片模式，請省略角色或使用 `first_frame`。
</Note>

### 樣式控制

<ParamField path="aspectRatio" type="string">
  長寬比提示，例如 `1:1`、`16:9`、`9:16`、`adaptive`，或供應商專用值。OpenClaw 會依供應商正規化或忽略不支援的值。
</ParamField>
<ParamField path="resolution" type="string">解析度提示，例如 `360P`、`480P`、`540P`、`720P`、`768P`、`1080P`、`4K`，或供應商專用值。OpenClaw 會依供應商正規化或忽略不支援的值。</ParamField>
<ParamField path="durationSeconds" type="number">
  目標持續時間（秒，四捨五入至最接近的供應商支援值）。
</ParamField>
<ParamField path="size" type="string">供應商支援時使用的尺寸提示。</ParamField>
<ParamField path="audio" type="boolean">
  支援時在輸出中啟用產生的音訊。這與 `audioRef*`（輸入）不同。
</ParamField>
<ParamField path="watermark" type="boolean">支援時切換供應商浮水印。</ParamField>

`adaptive` 是供應商專用的哨兵值：系統會將其原樣轉送給
在能力中宣告 `adaptive` 的供應商（例如 BytePlus
Seedance 會用它根據輸入圖片的尺寸自動偵測長寬比）。
未宣告此能力的供應商會透過工具結果中的
`details.ignoredOverrides` 呈現該值，讓捨棄此值的情況清楚可見。

### 進階

<ParamField path="action" type='"generate" | "status" | "list"' default="generate">
  `"status"` 會傳回目前工作階段的任務；`"list"` 會檢查供應商。
</ParamField>
<ParamField path="model" type="string">覆寫供應商／模型（例如 `runway/gen4.5`）。</ParamField>
<ParamField path="filename" type="string">輸出檔名提示。</ParamField>
<ParamField path="timeoutMs" type="number">選填的供應商作業逾時時間（毫秒）。若省略，OpenClaw 會在已設定時使用 `agents.defaults.mediaModels.video.timeoutMs`，否則會使用外掛作者定義的供應商預設值（若有）。</ParamField>
<ParamField path="providerOptions" type="object">
  以 JSON 物件提供供應商專用選項（例如 `{"seed": 42, "draft": true}`）。
  宣告型別結構描述的供應商會驗證鍵和值型別；未知的鍵或不相符的
  型別會在備援期間略過該候選項目。未宣告結構描述的供應商會
  原樣接收選項。執行 `video_generate action=list`
  可查看各供應商接受的選項。
</ParamField>

<Note>
並非所有供應商都支援所有參數。OpenClaw 會將持續時間正規化為
最接近的供應商支援值；當備援供應商提供不同的
控制介面時，也會重新對應已轉換的幾何提示，
例如將尺寸轉換為長寬比。對於確實不支援的覆寫，系統會盡力忽略，
並在工具結果中回報警告。硬性能力限制
（例如參考輸入過多）會在提交前導致失敗。工具結果會
回報套用的設定；`details.normalization` 會記錄所有
從要求值到套用值的轉換。
</Note>

參考輸入會選取執行階段模式：

- 無參考媒體 -> `generate`
- 有任何圖片參考 -> `imageToVideo`
- 有任何影片參考 -> `videoToVideo`
- 參考音訊輸入**不會**變更解析出的模式；它們會套用在
  圖片／影片參考所選模式之上，且僅適用於
  宣告 `maxInputAudios` 的供應商。

混合圖片與影片參考並非穩定的共用能力介面。
每個要求應優先只使用一種參考類型。

#### 備援與型別化選項

部分能力檢查是在備援層而非工具
邊界套用，因此超過主要供應商限制的要求，仍可
由具備相應能力的備援供應商執行：

- 當要求包含音訊參考時，若作用中的候選項目未宣告 `maxInputAudios`（或 `0`），
  將略過該項目並嘗試下一個候選項目。同一項
  防護也會依據 `maxInputImages`/`maxInputVideos`，套用於圖片和影片參考數量。
- 作用中候選項目的 `maxDurationSeconds` 低於所要求的 `durationSeconds`，
  且未宣告 `supportedDurationSeconds` 清單 -> 略過。
- 要求包含 `providerOptions`，且作用中的候選項目明確
  宣告型別化 `providerOptions` 結構描述 -> 若提供的鍵
  不在結構描述中或值型別不符，則略過。未
  宣告結構描述的供應商會原樣接收選項（向後相容的
  直接傳遞）。供應商可宣告空結構描述
  （`capabilities.providerOptions: {}`），選擇不接受任何供應商選項；
  這會造成與型別不符相同的略過結果。

要求中的第一個略過原因會以 `warn` 層級記錄，讓營運人員知道
主要供應商何時遭到略過；後續略過則以 `debug` 層級記錄，
避免冗長的備援鏈產生過多訊息。若所有候選項目均遭略過，
彙總錯誤會包含各候選項目的略過原因。

## 動作

| 動作     | 功能                                                                                             |
| ---------- | -------------------------------------------------------------------------------------------------------- |
| `generate` | 預設。根據指定提示詞與選填的參考輸入建立影片。                             |
| `status`   | 檢查目前工作階段中進行中影片任務的狀態，而不啟動另一次產生作業。 |
| `list`     | 顯示可用的供應商、模型及其能力。                                                |

## 模型選擇

OpenClaw 會依下列順序解析模型：

1. **`model` 工具參數** - 若代理程式在呼叫中指定此參數。
2. 設定中的 **`videoGenerationModel.primary`**。
3. 依序使用 **`videoGenerationModel.fallbacks`**。
4. **自動偵測** - 從目前的預設供應商開始，
   接著按字母順序檢查其餘具備有效驗證資訊的供應商。

若供應商失敗，系統會自動嘗試下一個候選項目。若所有
候選項目均失敗，錯誤會包含每次嘗試的詳細資訊。

已驗證供應商之間的自動備援一律啟用。每次呼叫的
`model` 仍具有最終決定權。

```json5
{
  agents: {
    defaults: {
      videoGenerationModel: {
        primary: "google/veo-3.1-fast-generate-preview",
        fallbacks: ["runway/gen4.5", "qwen/wan2.6-t2v"],
        timeoutMs: 180000, // 選填的單一工具供應商要求逾時覆寫
      },
    },
  },
}
```

## 供應商附註

<AccordionGroup>
  <Accordion title="Alibaba">
    使用 DashScope／Model Studio 非同步端點。參考圖片與
    影片必須是遠端 `http(s)` URL。
  </Accordion>
  <Accordion title="BytePlus（內建）">
    供應商 ID：`byteplus`。

    模型：`seedance-1-0-pro-250528`（預設）、
    `seedance-1-5-pro-251215`。

    使用統一的 `content[]` API。最多支援 2 張輸入圖片
    （`first_frame` + `last_frame`）。請依位置傳入圖片，或明確設定每張
    圖片的 `role`。

    支援的 `providerOptions` 鍵：`seed`（數字）、`draft`（布林值 -
    強制使用 480p）、`camera_fixed`（布林值）。

  </Accordion>
  <Accordion title="BytePlus Seedance 1.5 外掛">
    需要 [`@openclaw/byteplus-modelark`](https://www.npmjs.com/package/@openclaw/byteplus-modelark)
    外掛（外部提供，未內建）。供應商 ID：`byteplus-seedance15`。模型：
    `seedance-1-5-pro-251215`。

    使用統一的 `content[]` API。最多支援 2 張輸入圖片
    （`first_frame` + `last_frame`）。所有輸入都必須是遠端 `https://`
    URL。請在每張圖片上設定 `role: "first_frame"` / `"last_frame"`，或
    依位置傳入圖片。

    `aspectRatio: "adaptive"` 會根據輸入圖片自動偵測長寬比。
    `audio: true` 會對應至 `generate_audio`。`providerOptions.seed`
    （數字）會原樣轉送。

  </Accordion>
  <Accordion title="BytePlus Seedance 2.0">
    需要 [`@openclaw/byteplus-modelark`](https://www.npmjs.com/package/@openclaw/byteplus-modelark)
    外掛（外部提供，未內建）。供應商 ID：`byteplus-seedance2`。模型：
    `dreamina-seedance-2-0-260128`、
    `dreamina-seedance-2-0-fast-260128`。

    使用統一的 `content[]` API。最多支援 9 張參考圖片、
    3 段參考影片及 3 段參考音訊。所有輸入都必須是遠端
    `https://` URL。請在每個素材上設定 `role`，支援的值：
    `"first_frame"`、`"last_frame"`、`"reference_image"`、
    `"reference_video"`、`"reference_audio"`。

    `aspectRatio: "adaptive"` 會根據輸入圖片自動偵測長寬比。
    `audio: true` 會對應至 `generate_audio`。`providerOptions.seed`
    （數字）會原樣轉送。

  </Accordion>
  <Accordion title="ComfyUI">
    工作流程驅動的本機或雲端執行。透過已設定的圖形支援文字轉影片和
    圖片轉影片。
  </Accordion>
  <Accordion title="fal">
    對長時間執行的工作使用以佇列為基礎的流程。OpenClaw 預設最多等待 20
    分鐘，之後會將仍在進行中的 fal 佇列工作視為逾時。大多數 fal 影片模型
    接受單一圖片參照。Seedance 2.0 參照轉影片
    模型最多接受 9 張圖片、3 部影片和 3 個音訊參照，且
    參照檔案總數最多為 12 個。
  </Accordion>
  <Accordion title="Google (Gemini / Veo)">
    支援一個圖片或一個影片參照。在 Gemini API 路徑上，
    產生音訊的要求會被忽略並顯示警告，因為該 API 會拒絕
    目前 Veo 影片產生功能的 `generateAudio` 參數。
  </Accordion>
  <Accordion title="MiniMax">
    僅支援單一圖片參照。MiniMax 接受 `768P` 和 `1080P`
    解析度；提交前，`720P` 等要求會正規化為最接近的
    支援值。
  </Accordion>
  <Accordion title="OpenAI">
    僅轉送 `size` 覆寫。其他樣式覆寫
    （`aspectRatio`、`resolution`、`audio`、`watermark`）會被忽略並
    顯示警告。
  </Accordion>
  <Accordion title="OpenRouter">
    使用 OpenRouter 的非同步 `/videos` API。OpenClaw 會提交
    工作、輪詢 `polling_url`，並下載 `unsigned_urls` 或文件記載的
    工作內容端點。隨附的 `google/veo-3.1-fast` 預設值
    宣告 4/6/8 秒的持續時間、`720P`/`1080P` 解析度，以及
    `16:9`/`9:16` 長寬比。
  </Accordion>
  <Accordion title="Qwen">
    使用與 Alibaba 相同的 DashScope 後端。參照輸入必須是遠端
    `http(s)` URL；本機檔案會在一開始就被拒絕。
  </Accordion>
  <Accordion title="Runway">
    透過資料 URI 支援本機檔案。影片轉影片需要
    `runway/gen4_aleph`。僅文字執行會公開 `16:9` 和 `9:16` 長寬
    比。
  </Accordion>
  <Accordion title="Together">
    僅支援單一圖片參照。
  </Accordion>
  <Accordion title="Vydra">
    直接使用 `https://www.vydra.ai/api/v1` 以避免重新導向時
    遺失驗證。隨附的 `veo3` 僅支援文字轉影片；`kling` 需要
    遠端圖片 URL。
  </Accordion>
  <Accordion title="xAI">
    預設的 `grok-imagine-video` 模型支援文字轉影片、單一
    首幀圖片轉影片、透過 xAI `reference_images` 傳入最多 7 個 `reference_image` 輸入，
    以及遠端影片編輯／延長流程。產生功能預設為
    `480P`；若省略 `aspectRatio`，單一圖片轉影片會沿用來源比例。
    影片編輯／延長會沿用輸入的幾何尺寸，且
    不接受長寬比或解析度覆寫。延長功能接受 2-10
    秒。

    `grok-imagine-video-1.5` 僅支援圖片轉影片：請恰好提供一張圖片。
    它支援 1-15 秒以及 `480P`、`720P` 或 `1080P`，預設為
    `480P`；省略 `aspectRatio` 即可沿用來源圖片比例。預覽版
    和標有日期的 1.5 識別碼會接受相同的驗證，並原封不動地轉送。

  </Accordion>
</AccordionGroup>

## 供應商能力模式

共用影片產生合約支援特定模式的能力，
而不僅是扁平的彙總限制。新的供應商實作
應優先使用明確的模式區塊：

```typescript
capabilities: {
  generate: {
    maxVideos: 1,
    maxDurationSeconds: 10,
    supportsResolution: true,
  },
  imageToVideo: {
    enabled: true,
    maxVideos: 1,
    maxInputImages: 1,
    maxInputImagesByModel: { "provider/reference-to-video": 9 },
    maxDurationSeconds: 5,
  },
  videoToVideo: {
    enabled: true,
    maxVideos: 1,
    maxInputVideos: 1,
    maxDurationSeconds: 5,
  },
}
```

`maxInputImages` 和 `maxInputVideos` 等扁平彙總欄位
**不足以**宣告轉換模式支援。供應商應
明確宣告 `generate`、`imageToVideo` 和 `videoToVideo`，使即時
測試、合約測試和共用 `video_generate` 工具能以確定方式驗證
模式支援。

當供應商中的某個模型比其餘模型支援更廣泛的參照輸入時，
請使用 `maxInputImagesByModel`、`maxInputVideosByModel` 或
`maxInputAudiosByModel`，而不要提高整個模式的限制。

## 即時測試

共用隨附供應商的選擇性即時涵蓋範圍：

```bash
OPENCLAW_LIVE_TEST=1 pnpm test:live -- extensions/video-generation-providers.live.test.ts
```

儲存庫包裝指令：

```bash
pnpm test:live:media video
```

此即時測試檔預設會優先使用已匯出的供應商環境變數，而非已儲存的驗證
設定檔，並預設執行適合發布的冒煙測試：

- `generate`，套用於掃描中的每個非 FAL 供應商。
- 一秒鐘的龍蝦提示詞。
- 各供應商的操作上限取自
  `OPENCLAW_LIVE_VIDEO_GENERATION_TIMEOUT_MS`（預設為 `180000`）。

FAL 採選擇性啟用，因為供應商端的佇列延遲可能會主導發布
時間：

```bash
pnpm test:live:media video --video-providers fal
```

設定 `OPENCLAW_LIVE_VIDEO_GENERATION_FULL_MODES=1`，即可一併執行已宣告且共用掃描能以本機媒體安全測試的
轉換模式：

- `imageToVideo`，條件為 `capabilities.imageToVideo.enabled`。
- `videoToVideo`，條件為 `capabilities.videoToVideo.enabled`，且
  供應商／模型在共用掃描中接受以緩衝區為基礎的本機影片輸入。

目前僅在選取 `runway/gen4_aleph` 時，共用 `videoToVideo` 即時測試通道才會涵蓋 `runway`。

## 設定

在 OpenClaw 設定中設定預設影片產生模型：

```json5
{
  agents: {
    defaults: {
      videoGenerationModel: {
        primary: "qwen/wan2.6-t2v",
        fallbacks: ["qwen/wan2.6-r2v-flash"],
      },
    },
  },
}
```

或透過命令列介面：

```bash
openclaw config set agents.defaults.mediaModels.video.primary "qwen/wan2.6-t2v"
```

## 相關內容

- [Alibaba Model Studio](/zh-TW/providers/alibaba)
- [背景工作](/zh-TW/automation/tasks) - 非同步影片產生的工作追蹤
- [BytePlus](/zh-TW/concepts/model-providers#byteplus-international)
- [ComfyUI](/zh-TW/providers/comfy)
- [設定參考](/zh-TW/gateway/config-agents#agent-defaults)
- [fal](/zh-TW/providers/fal)
- [Google (Gemini)](/zh-TW/providers/google)
- [MiniMax](/zh-TW/providers/minimax)
- [模型](/zh-TW/concepts/models)
- [OpenAI](/zh-TW/providers/openai)
- [Qwen](/zh-TW/providers/qwen)
- [Runway](/zh-TW/providers/runway)
- [Together AI](/zh-TW/providers/together)
- [工具概覽](/zh-TW/tools)
- [Vydra](/zh-TW/providers/vydra)
- [xAI](/zh-TW/providers/xai)
