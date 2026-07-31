---
read_when:
    - 設計或重構媒體理解功能
    - 調校傳入音訊／視訊／影像的前處理
sidebarTitle: Media understanding
summary: 傳入圖片／音訊／影片理解（選用），支援提供者與命令列介面備援方案
title: 媒體理解
x-i18n:
    generated_at: "2026-07-26T08:37:18Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 38e9a0f89607bb9c4af85689ef0fbd3df9234b36e06d86c129e0d823d6e05143
    source_path: nodes/media-understanding.md
    workflow: 16
---

OpenClaw 可在回覆流水線執行前摘要傳入的媒體（圖片／音訊／影片），讓命令解析與路由使用簡短文字，而非原始位元組。理解功能會自動偵測本機工具或供應商金鑰，你也可以設定明確的模型。原始媒體一律會照常傳送給模型；當理解失敗或停用時，回覆流程會維持不變並繼續執行。

供應商外掛會註冊能力中繼資料（哪個供應商支援哪種媒體類型、預設模型、優先順序）。OpenClaw 核心負責共用的 `tools.media` 設定、備援順序，以及回覆流水線整合。

## 運作方式

<Steps>
  <Step title="收集附件">
    收集依序排列的傳入媒體資訊（`path`、`url`、`contentType` 和 `kind`）。
  </Step>
  <Step title="依能力選取">
    對每項已啟用的能力（圖片／音訊／影片），依 `attachments` 政策選取附件（預設：僅第一個附件）。
  </Step>
  <Step title="選擇模型">
    選取第一個符合資格的模型項目（大小、能力及可用的驗證）。
  </Step>
  <Step title="失敗時備援">
    如果模型發生錯誤、逾時，或媒體超過 `maxBytes`，則嘗試下一個項目。
  </Step>
  <Step title="成功時套用">
    `Body` 會成為 `[Image]`、`[Audio]` 或 `[Video]` 區塊。音訊也會設定 `{{Transcript}}`；若有字幕文字，命令解析會使用字幕文字，否則使用逐字稿。字幕會在區塊內保留為 `User text:`。
  </Step>
</Steps>

## 設定

`tools.media` 包含一份標註能力的模型清單，以及少量的個別能力控制項：

```json5
{
  tools: {
    media: {
      concurrency: 2, // 同時執行能力工作的最大數量（預設）
      models: [
        { provider: "openai", model: "gpt-4o-mini-transcribe", capabilities: ["audio"] },
        { provider: "google", model: "gemini-3-flash-preview", capabilities: ["image", "video"] },
      ],
      image: { preferredModel: "google/gemini-3-flash-preview" },
      audio: { enabled: true },
      video: { enabled: true },
    },
  },
}
```

個別能力（`image`/`audio`/`video`）的鍵：

| 鍵               | 類型      | 預設值                                 | 備註                                                                 |
| ---------------- | --------- | -------------------------------------- | -------------------------------------------------------------------- |
| `enabled`        | `boolean` | 自動（`false` 會停用）                | 設定 `false` 以關閉此能力的自動偵測              |
| `preferredModel` | `string`  | 第一個相容項目                 | 優先使用 `provider/model`、模型 ID、`provider:<id>` 或 `cli:command` |
| `prompt`         | `string`  | 能力預設值                     | 當項目未覆寫提示詞時使用的預設提示詞                    |
| `maxChars`       | `number`  | 圖片／影片為 `500`，音訊未設定         | 預設輸出限制                                                 |
| `maxBytes`       | `number`  | 圖片 10MB、音訊 20MB、影片 50MB     | 預設輸入限制                                                  |
| `timeoutSeconds` | `number`  | 圖片／音訊為 `60`，影片為 `120`          | 預設請求逾時                                              |
| `language`       | `string`  | 未設定                                  | 音訊轉錄提示                                             |
| `scope`          | object    | 未設定                                  | 依頻道／聊天類型／來源鍵設限                                 |
| `attachments`    | object    | `{ mode: "first", maxAttachments: 1 }` | 選取要處理的相符附件                      |
| `echoTranscript` | `boolean` | `false`                                | 僅限音訊：在代理處理前回顯逐字稿              |
| `echoFormat`     | `string`  | `'📝 "{transcript}"'`                  | 僅限音訊：回顯逐字稿的格式                         |

提示詞、限制、語言提示、請求覆寫和供應商選項，可設為能力預設值，或在個別 `tools.media.models[]` 項目上覆寫。未設定明確模型時，能力預設值也適用於自動偵測到的供應商。

### 模型項目

每個 `models[]` 項目都是**供應商**項目（預設）或**命令列介面**項目：

<Tabs>
  <Tab title="供應商項目">
    ```json5
    {
      type: "provider", // 省略時為預設值
      provider: "openai",
      model: "gpt-5.6-sol",
      prompt: "以不超過 500 個字元描述圖片。",
      maxChars: 500,
      maxBytes: 10485760,
      timeoutSeconds: 60,
      capabilities: ["image"],
      profile: "vision-profile",
      preferredProfile: "vision-fallback",
    }
    ```
  </Tab>
  <Tab title="命令列介面項目">
    ```json5
    {
      type: "cli",
      command: "gemini",
      args: [
        "-m",
        "gemini-3-flash",
        "--allowed-tools",
        "read_file",
        "讀取 {{AttachmentPath}} 的媒體，並以不超過 {{MaxChars}} 個字元描述。",
      ],
      maxChars: 500,
      maxBytes: 52428800,
      timeoutSeconds: 120,
      capabilities: ["video", "image"],
    }
    ```

    命令列介面範本也可使用 `{{AttachmentUrl}}`、`{{AttachmentContentType}}`、`{{AttachmentDir}}`、`{{AttachmentIndex}}`、`{{OutputDir}}`（為本次執行建立的暫存目錄）和 `{{OutputBase}}`（暫存檔案的基礎路徑，不含副檔名）。較舊的 `{{MediaPath}}`、`{{MediaUrl}}`、`{{MediaType}}` 和 `{{MediaDir}}` 名稱仍保留為已淘汰的相容性別名。

  </Tab>
</Tabs>

### 供應商認證資訊

供應商媒體理解功能採用與一般模型呼叫相同的驗證解析順序：驗證設定檔、環境變數，接著是 `models.providers.<providerId>.apiKey`。`tools.media.models[]` 項目不接受行內 `apiKey` 欄位。

```json5
{
  models: {
    providers: {
      openai: { apiKey: "<OPENAI_API_KEY>" },
      moonshot: { apiKey: "<MOONSHOT_API_KEY>" },
    },
  },
}
```

如需設定檔、環境變數及自訂基底 URL 的資訊，請參閱[工具和自訂供應商](/zh-TW/gateway/config-tools)。

## 規則與行為

- 超過 `maxBytes` 的媒體會略過該模型，並嘗試下一個模型。
- 小於 1024 位元組的音訊檔會視為空白／損毀，並在轉錄前略過；代理會改為取得確定性的預留位置逐字稿。
- 如果目前作用中的主要圖片模型已原生支援視覺，OpenClaw 會略過 `[Image]` 摘要區塊，直接將原始圖片傳入模型。MiniMax 是例外：`minimax`、`minimax-cn`、`minimax-portal` 和 `minimax-portal-cn` 一律透過外掛所擁有的 `MiniMax-VL-01` 媒體供應商路由圖片理解，即使舊版 MiniMax M2.x 聊天中繼資料宣稱支援圖片輸入也一樣（只有 `MiniMax-M3` 及更新版本會視為原生支援視覺）。
- 如果閘道／WebChat 的主要模型僅支援文字，圖片附件會保留為已卸載的 `media://inbound/*` 參照，讓圖片／PDF 工具或已設定的圖片模型仍可檢查附件，而不會遺失附件。
- 明確設定的 `openclaw infer image describe --file <path> --model <provider/model>`（別名：`openclaw capability image describe`）會直接執行該支援圖片的供應商／模型，包括 `ollama/qwen2.5vl:7b` 等 Ollama 參照，前提是在 `models.providers.ollama.models[]` 下設定了相符且支援圖片的模型。
- 如果 `<capability>.enabled` 不是 `false`，但未設定任何模型，OpenClaw 會在目前作用中的回覆模型之供應商支援該能力時嘗試使用該模型。

### 自動偵測（預設）

當 `tools.media.<capability>.enabled` 不是 `false`，且未設定任何模型時，OpenClaw 會依序嘗試下列選項，並在第一個可運作的選項停止：

<Steps>
  <Step title="已設定的圖片模型（僅限圖片）">
    `agents.defaults.imageModel` 主要／備援參照，但目前作用中的回覆模型已原生支援視覺時除外。優先使用 `provider/model` 參照；只有在相符項目唯一時，才會從已設定且支援圖片的供應商模型項目補全未限定的參照。
  </Step>
  <Step title="目前作用中的回覆模型">
    當目前作用中的回覆模型之供應商支援該能力時，使用該模型。
  </Step>
  <Step title="供應商驗證（僅限音訊，優先於本機命令列介面）">
    支援音訊的已設定 `models.providers.*` 項目會優先於本機命令列介面嘗試。內建供應商的優先順序（優先順序相同時，依供應商 ID 的字母順序決定）：Groq/OpenAI &rarr; xAI &rarr; Deepgram &rarr; OpenRouter &rarr; Google/SenseAudio &rarr; Deepinfra/ElevenLabs &rarr; Mistral。
  </Step>
  <Step title="本機命令列介面（僅限音訊）">
    已就緒的本機二進位檔會成為依序排列的備援清單：
    - `whisper-cli` 僅在目前程序中先前的模型叫用觀察到 Metal 或 CUDA 後排在第一位
    - 預設使用 CPU 的 `sherpa-onnx-offline`（需要 `SHERPA_ONNX_MODEL_DIR`，並搭配 `tokens.txt`/`encoder.onnx`/`decoder.onnx`/`joiner.onnx`）
    - 當加速僅具備建置支援或尚未觀察到時，使用 `whisper-cli`
    - 在 Apple Silicon 上使用 `parakeet-mlx`（支援 MLX，尚未觀察到裝置使用情況）
    - `whisper`（Python 命令列介面；預設使用 `turbo` 模型，並自動下載）

    後端能力檢查結果會被快取，且不會載入模型。建置能力、要求的後端旗標，以及從實際叫用觀察到的後端，會維持彼此分離。自動偵測到的 whisper.cpp 會維持啟用模型執行記錄，以便記錄上游所選後端的那一行。明確設定的命令列介面項目會保留其設定順序、後端旗標和輸出旗標。

  </Step>
  <Step title="供應商驗證（圖片／影片）">
    支援該能力的已設定 `models.providers.*` 項目會優先於內建備援順序嘗試。僅設定圖片的供應商如果具有支援圖片的模型，即使不是內建供應商外掛，也會自動註冊至媒體理解功能。

    內建供應商的優先順序（優先順序相同時，依供應商 ID 的字母順序決定）：
    - 圖片：Anthropic/OpenAI &rarr; Google &rarr; MiniMax &rarr; Deepinfra &rarr; MiniMax Portal &rarr; Z.AI
    - 影片：Google &rarr; Qwen &rarr; Moonshot

  </Step>
  <Step title="Antigravity 命令列介面（僅限圖片／影片）">
    使用第一個已安裝的 `agy` 或 `antigravity` 二進位檔（以 `OPENCLAW_ANTIGRAVITY_CLI` 覆寫），並以媒體所在目錄作為沙箱範圍。
  </Step>
</Steps>

若要停用某項能力的自動偵測：

```json5
{
  tools: {
    media: {
      audio: {
        enabled: false,
      },
    },
  },
}
```

<Note>
在 macOS/Linux/Windows 上，二進位檔偵測採盡力而為；請確保命令列介面位於 `PATH`（會展開 `~`），或以完整命令路徑設定明確的命令列介面模型項目。
</Note>

### Proxy 支援（音訊／影片供應商呼叫）

以供應商為基礎的**音訊**與**影片**理解功能會遵循標準輸出 Proxy 環境變數，包括 `NO_PROXY`/`no_proxy` 略過規則：`HTTPS_PROXY`、`HTTP_PROXY`、`ALL_PROXY`、`https_proxy`、`http_proxy`、`all_proxy`。小寫變數的優先順序高於大寫變數。如果均未設定，媒體理解功能會直接向外連線；如果 Proxy 值格式錯誤，OpenClaw 會記錄警告並改用直接擷取。圖片理解不會經過此 Proxy 路徑。

## 能力

在 `models[]` 項目上設定 `capabilities`，以將其限制為特定媒體類型。對於共用清單，OpenClaw 會依各內建供應商推斷預設值：

| 提供者                                                                 | 功能          |
| ------------------------------------------------------------------------ | --------------------- |
| `openai`, `anthropic`, `minimax`                                         | 影像                 |
| `minimax-portal`                                                         | 影像                 |
| `moonshot`                                                               | 影像 + 影片         |
| `openrouter`                                                             | 影像 + 音訊         |
| `google`（Gemini API）                                                    | 影像 + 音訊 + 影片 |
| `qwen`                                                                   | 影像 + 影片         |
| `deepinfra`                                                              | 影像 + 音訊         |
| `mistral`                                                                | 音訊                 |
| `zai`                                                                    | 影像                 |
| `groq`, `xai`, `deepgram`, `senseaudio`                                  | 音訊                 |
| 任何具有影像功能模型的 `models.providers.<id>.models[]` 目錄 | 影像                 |

對於命令列介面項目，請明確設定 `capabilities`，以避免意外的比對結果；若省略，該項目將符合其出現於其中的每個功能清單。

## 提供者支援矩陣

| 功能 | 提供者                                                                                                                                               | 備註                                                                                                                                                                                   |
| ---------- | ------------------------------------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 影像      | Anthropic, Codex app-server, Deepinfra, Google, MiniMax, MiniMax Portal, Moonshot, OpenAI, OpenAI Codex OAuth, OpenRouter, Qwen, Z.AI, 設定提供者 | 供應商外掛會註冊影像支援；`openai/*` 可使用 API 金鑰或 Codex OAuth 路由；`codex/*` 使用有界限的 Codex app-server 回合；支援影像的設定提供者會自動註冊。 |
| 音訊      | Deepgram, Deepinfra, ElevenLabs, Google, Groq, Mistral, OpenAI, OpenRouter, SenseAudio, xAI                                                             | 提供者轉錄（Whisper/Groq/xAI/Deepgram/OpenRouter STT/Gemini/SenseAudio/Scribe/Voxtral）。                                                                                     |
| 影片      | Google, Moonshot, Qwen                                                                                                                                  | 透過供應商外掛提供影片理解功能；Qwen 影片理解使用標準 DashScope 端點。                                                                        |

<Note>
**MiniMax 備註**：`minimax`、`minimax-cn`、`minimax-portal` 和 `minimax-portal-cn` 的影像理解一律來自外掛所擁有的 `MiniMax-VL-01` 媒體提供者，即使舊版 MiniMax M2.x 聊天中繼資料聲稱支援影像輸入亦然。
</Note>

## 模型選擇指南

- 當品質與安全性至關重要時，請為各項媒體功能優先選用目前最強的新一代模型。
- 對於處理不受信任輸入且啟用工具的代理程式，請避免使用較舊或較弱的媒體模型。
- 每項功能至少保留一個後援模型，以確保可用性（高品質模型 + 較快或較便宜的模型）。
- 當提供者 API 無法使用時，命令列介面後援（`whisper-cli`、`whisper`、`gemini`）可提供協助。
- 已知的檔案輸出模式具有決定性：若推斷出的逐字稿檔案為空或不存在，則不會產生逐字稿，而不會退回使用命令列介面的進度輸出。
- `parakeet-mlx`：搭配 `--output-dir` 和預設的 `{filename}` 輸出範本使用 `--output-format txt`（或 `all`）。上游的 `PARAKEET_OUTPUT_FORMAT` 和 `PARAKEET_OUTPUT_TEMPLATE` 環境變數也會受到支援。OpenClaw 會讀取 `<output-dir>/<media-basename>.txt`；預設的 `srt` 格式、其他格式及自訂輸出範本仍會使用標準輸出。

## 附件政策

各功能的 `attachments` 控制要處理哪些附件：

<ParamField path="mode" type='"first" | "all"' default="first">
  僅處理第一個選取的附件，或處理所有附件。
</ParamField>
<ParamField path="maxAttachments" type="number" default="1">
  限制處理數量。
</ParamField>
<ParamField path="prefer" type='"first" | "last" | "path" | "url"'>
  候選附件之間的選取偏好。
</ParamField>

當 `mode: "all"` 時，輸出會標示為 `[Image 1/2]`、`[Audio 2/2]` 等。

### 檔案附件擷取

- 擷取出的檔案文字會先包裝為不受信任的外部內容，再附加至媒體提示詞；包裝會使用 `<<<EXTERNAL_UNTRUSTED_CONTENT id="...">>>` / `<<<END_EXTERNAL_UNTRUSTED_CONTENT id="...">>>` 等邊界標記，並加上一行 `Source: External` 中繼資料。
- 此路徑刻意省略較長的 `SECURITY NOTICE:` 橫幅，以保持媒體提示詞簡短；邊界標記和中繼資料仍會套用。
- 無法擷取任何文字的檔案會得到 `[No extractable text]`。
- 如果 PDF 退回使用轉譯後的頁面影像，OpenClaw 會將這些影像轉送至具視覺功能的回覆模型，並在檔案區塊中保留預留位置 `[PDF content rendered to images]`。

## 設定範例

<Tabs>
  <Tab title="共用模型 + 覆寫">
    ```json5
    {
      tools: {
        media: {
          models: [
            { provider: "openai", model: "gpt-5.6-sol", capabilities: ["image"] },
            {
              provider: "google",
              model: "gemini-3-flash-preview",
              capabilities: ["image", "audio", "video"],
            },
            {
              type: "cli",
              command: "gemini",
              args: [
                "-m",
                "gemini-3-flash",
                "--allowed-tools",
                "read_file",
                "讀取位於 {{AttachmentPath}} 的媒體，並用不超過 {{MaxChars}} 個字元描述它。",
              ],
              capabilities: ["image", "video"],
            },
          ],
          audio: {
            attachments: { mode: "all", maxAttachments: 2 },
          },
          video: {
            maxChars: 500,
          },
        },
      },
    }
    ```
  </Tab>
  <Tab title="僅音訊 + 影片">
    ```json5
    {
      tools: {
        media: {
          audio: {
            enabled: true,
            models: [
              { provider: "openai", model: "gpt-4o-mini-transcribe" },
              {
                type: "cli",
                command: "whisper",
                args: ["--model", "base", "{{AttachmentPath}}"],
              },
            ],
          },
          video: {
            enabled: true,
            maxChars: 500,
            models: [
              { provider: "google", model: "gemini-3-flash-preview" },
              {
                type: "cli",
                command: "gemini",
                args: [
                  "-m",
                  "gemini-3-flash",
                  "--allowed-tools",
                  "read_file",
                  "讀取位於 {{AttachmentPath}} 的媒體，並用不超過 {{MaxChars}} 個字元描述它。",
                ],
              },
            ],
          },
        },
      },
    }
    ```
  </Tab>
  <Tab title="僅影像">
    ```json5
    {
      tools: {
        media: {
          image: {
            enabled: true,
            maxBytes: 10485760,
            maxChars: 500,
            models: [
              { provider: "openai", model: "gpt-5.6-sol" },
              { provider: "anthropic", model: "claude-opus-5" },
              {
                type: "cli",
                command: "gemini",
                args: [
                  "-m",
                  "gemini-3-flash",
                  "--allowed-tools",
                  "read_file",
                  "讀取位於 {{AttachmentPath}} 的媒體，並用不超過 {{MaxChars}} 個字元描述它。",
                ],
              },
            ],
          },
        },
      },
    }
    ```
  </Tab>
  <Tab title="單一多模態項目">
    ```json5
    {
      tools: {
        media: {
          image: {
            models: [
              {
                provider: "google",
                model: "gemini-3.1-pro-preview",
                capabilities: ["image", "video", "audio"],
              },
            ],
          },
          audio: {
            models: [
              {
                provider: "google",
                model: "gemini-3.1-pro-preview",
                capabilities: ["image", "video", "audio"],
              },
            ],
          },
          video: {
            models: [
              {
                provider: "google",
                model: "gemini-3.1-pro-preview",
                capabilities: ["image", "video", "audio"],
              },
            ],
          },
        },
      },
    }
    ```
  </Tab>
</Tabs>

## 狀態輸出

媒體理解執行時，`/status` 會包含每項功能的摘要行：

```
📎 媒體：影像成功（openai/gpt-5.6-sol）· 音訊成功（whisper-cli observed=metal）
```

若要執行預檢清查，請執行 `openclaw capability audio providers`。本機資料列會將本機後援勝出項目，與全域提供者選擇、就緒狀態，以及各自獨立的可支援／已要求／已觀察後端欄位分開顯示。相同的本機選擇也會以資訊性 doctor 發現項目的形式提供：

```bash
openclaw doctor --lint --only core/doctor/local-audio-acceleration --severity-min info
```

## 備註

- 理解功能採盡力而為。錯誤不會阻止回覆。
- 即使理解功能已停用，附件仍會傳遞給模型。
- 使用 `scope` 限制理解功能的執行位置（例如僅限私訊）。

## 相關內容

- [設定](/zh-TW/gateway/configuration)
- [影像與媒體支援](/zh-TW/nodes/images)
