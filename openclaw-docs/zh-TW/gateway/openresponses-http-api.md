---
read_when:
    - 整合使用 OpenResponses API 的用戶端
    - 你想要以項目為基礎的輸入、用戶端工具呼叫或 SSE 事件
summary: 從閘道公開與 OpenResponses 相容的 `/v1/responses` HTTP 端點
title: OpenResponses API
x-i18n:
    generated_at: "2026-07-26T08:33:29Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 5bfd6ca3bf0cecd761fde865b41a95cff3fc5681f74f31b3adae5cd2e0b0be95
    source_path: gateway/openresponses-http-api.md
    workflow: 16
---

閘道可提供與 OpenResponses 相容的 `POST /v1/responses` 端點。此端點**預設為停用**，並與閘道共用連接埠（WS + HTTP 多工）：`http://<gateway-host>:<port>/v1/responses`。

請求會以一般閘道代理程式執行的方式運作（與 `openclaw agent` 使用相同程式碼路徑），因此路由、權限與設定皆與你的閘道一致。

使用 `gateway.http.endpoints.responses.enabled` 啟用或停用。啟用後，同一相容介面也會提供 `GET /v1/models`、`GET /v1/models/{id}`、`POST /v1/embeddings` 與 `POST /v1/chat/completions`。

## 驗證、安全性與路由

運作行為與 [OpenAI Chat Completions](/zh-TW/gateway/openai-http-api) 一致：

- 驗證路徑與 `gateway.auth.mode` 一致：共用密鑰（`token`/`password`）使用 `Authorization: Bearer <token-or-password>`；受信任 Proxy 使用可識別身分的 Proxy 標頭（同一主機的迴路 Proxy 需要 `gateway.auth.trustedProxy.allowLoopback = true`；若不存在 `Forwarded`/`X-Forwarded-*`/`X-Real-IP` 標頭，則可透過 `gateway.auth.password` / `OPENCLAW_GATEWAY_PASSWORD` 直接從同一主機備援）；私人輸入端的 `none` 不需要驗證標頭。請參閱[受信任 Proxy 驗證](/zh-TW/gateway/trusted-proxy-auth)。
- 請將此端點視為對閘道執行個體具有完整操作員存取權。
- 共用密鑰驗證模式會忽略 Bearer 宣告的較窄 `x-openclaw-scopes`，並還原完整的預設操作員權限範圍集合：`operator.admin`、`operator.approvals`、`operator.pairing`、`operator.read`、`operator.talk.secrets`、`operator.write`。此端點上的聊天回合會被視為擁有者傳送者回合。
- 攜帶受信任身分的 HTTP 模式（受信任 Proxy 或 `gateway.auth.mode="none"`）會在存在 `x-openclaw-scopes` 時採用其值，否則回復為操作員預設權限範圍集合。只有當呼叫端明確縮小權限範圍且省略 `operator.admin` 時，才會失去擁有者語意。
- 使用 `model: "openclaw"`、`"openclaw/default"`、`"openclaw/<agentId>"` 或 `x-openclaw-agent-id` 標頭選取代理程式。
- 使用 `x-openclaw-model` 覆寫所選代理程式的後端模型（在攜帶身分的驗證路徑上需要 `operator.admin`）。
- 使用 `x-openclaw-session-key` 明確指定工作階段路由（若使用保留命名空間 `subagent:`、`cron:`、`acp:`，則會以 `400 invalid_request_error` 拒絕）。
- 使用 `x-openclaw-message-channel` 指定非預設的合成輸入通道情境。

如需代理程式目標模型、`openclaw/default`、嵌入向量傳遞與後端模型覆寫的標準說明，請參閱 [OpenAI Chat Completions](/zh-TW/gateway/openai-http-api#agent-first-model-contract)。

請參閱[操作員權限範圍](/zh-TW/gateway/operator-scopes)與[安全性](/zh-TW/gateway/security)。

## 工作階段行為

端點預設為**每個請求皆無狀態**（每次呼叫都會產生新的工作階段金鑰）。

如果請求包含 OpenResponses `user` 字串，閘道會從中衍生穩定的工作階段金鑰，讓重複呼叫可以共用代理程式工作階段。

當請求維持在同一代理程式／使用者／所要求的工作階段範圍內（依驗證主體、代理程式 ID 與 `x-openclaw-session-key` 比對）時，`previous_response_id` 會重複使用先前回應的工作階段。

## 請求格式

| 欄位                                                            | 支援                                                                                                                        |
| ---------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------ |
| `input`                                                          | 字串或項目物件陣列。                                                                                               |
| `instructions`                                                   | 合併至系統提示詞。                                                                                                 |
| `tools`                                                          | 用戶端工具定義（函式工具）。                                                                                      |
| `tool_choice`                                                    | 使用 `"auto"`、`"none"`、`"required"` 或 `{ "type": "function", "name": "..." }` 篩選或要求使用用戶端工具。                |
| `stream`                                                         | 啟用 SSE 串流。                                                                                                         |
| `max_output_tokens`                                              | 盡力而為的輸出限制（取決於供應商）。                                                                                 |
| `temperature`                                                    | 盡力而為的取樣溫度。以 ChatGPT 為基礎的 Codex Responses 後端會忽略此值，因為該後端使用固定的伺服器端取樣。 |
| `top_p`                                                          | 盡力而為的核心取樣。與 `temperature` 相同，受 Codex Responses 的限制。                                                    |
| `user`                                                           | 穩定的工作階段路由。                                                                                                        |
| `previous_response_id`                                           | 工作階段連續性（見上文）。                                                                                                |
| `max_tool_calls`、`reasoning`、`metadata`、`store`、`truncation` | 接受但目前會忽略。                                                                                                |

## 項目（輸入）

### `message`

角色：`system`、`developer`、`user`、`assistant`。

- `system` 與 `developer` 會附加至系統提示詞。
- 最新的 `user` 或 `function_call_output` 項目會成為「目前訊息」。
- 較早的使用者／助理訊息會納入歷史記錄以提供情境。

### `function_call_output`（以回合為基礎的工具）

將工具結果傳回模型：

```json
{
  "type": "function_call_output",
  "call_id": "call_123",
  "output": "{\"temperature\": \"72F\"}"
}
```

### `reasoning` 與 `item_reference`

為了結構描述相容性而接受，但建立提示詞時會忽略。

## 工具（用戶端函式工具）

使用 `tools: [{ type: "function", name, description?, parameters? }]` 提供工具。

如果代理程式呼叫工具，回應會傳回 `function_call` 輸出項目。傳送包含 `function_call_output` 的後續請求以繼續該回合。

對於 `tool_choice: "required"` 與由函式固定的 `tool_choice`，端點會縮小公開的用戶端函式工具集合、指示執行階段在回應前呼叫用戶端工具，並在回合不含相符的結構化用戶端工具呼叫時拒絕該回合，以符合 `/v1/chat/completions` 合約。非串流請求會傳回包含 `api_error` 的 `502`；串流請求則會發出 `response.failed` 事件。

## 圖片（`input_image`）

支援 base64 或 URL 來源：

```json
{
  "type": "input_image",
  "source": { "type": "url", "url": "https://example.com/image.png" }
}
```

允許的 MIME 類型（預設）：`image/jpeg`、`image/png`、`image/gif`、`image/webp`、`image/heic`、`image/heif`。大小上限（預設）：10MB。

## 檔案（`input_file`）

支援 base64 或 URL 來源：

```json
{
  "type": "input_file",
  "source": {
    "type": "base64",
    "media_type": "text/plain",
    "data": "SGVsbG8gV29ybGQh",
    "filename": "hello.txt"
  }
}
```

允許的 MIME 類型（預設）：`text/plain`、`text/markdown`、`text/html`、`text/csv`、`application/json`、`application/pdf`。大小上限（預設）：5MB。

目前行為：

- 檔案內容會解碼並加入**系統提示詞**，而非使用者訊息，因此會維持暫時性（不會保存於工作階段歷史記錄中）。
- 解碼後的檔案文字會先包裝成**不受信任的外部內容**再加入，因此檔案位元組會被視為資料，而非受信任的指示。注入的區塊會使用明確的邊界標記（`<<<EXTERNAL_UNTRUSTED_CONTENT id="...">>>` / `<<<END_EXTERNAL_UNTRUSTED_CONTENT id="...">>>`）與一行 `Source: External` 中繼資料。為保留提示詞預算，該區塊刻意省略較長的 `SECURITY NOTICE:` 橫幅；邊界標記與中繼資料仍然適用。
- PDF 會先解析文字。若找到的文字很少，前幾頁會柵格化為圖片並傳遞給模型，而注入的檔案區塊會使用預留位置 `[PDF content rendered to images]`。

PDF 解析由內建的 `document-extract` 外掛提供；該外掛使用 `clawpdf` 及其封裝的 PDFium WebAssembly 執行階段擷取文字並轉譯頁面。

URL 擷取預設值：

- `files.allowUrl`：`true`
- `images.allowUrl`：`true`
- `maxUrlParts`：`8`（每個請求中，基於 URL 的 `input_file` + `input_image` 部分總數）
- 請求受到防護（DNS 解析、私人 IP 封鎖、重新導向上限、逾時）。
- 每種輸入類型皆支援選用的主機名稱允許清單（`files.urlAllowlist`、`images.urlAllowlist`）：完全相符的主機（`"cdn.example.com"`）或萬用字元子網域（`"*.assets.example.com"`，不會比對頂層網域本身）。允許清單為空或省略時，表示不限制主機名稱允許清單。
- 若要完全停用基於 URL 的擷取，請設定 `files.allowUrl: false` 和／或 `images.allowUrl: false`。

## 檔案與圖片限制

此端點使用內建的 20 MB 請求本文限制。檔案與圖片來源
原則仍可在 `gateway.http.endpoints.responses` 下設定：

```json5
{
  gateway: {
    http: {
      endpoints: {
        responses: {
          enabled: true,
          maxUrlParts: 8,
          files: {
            allowUrl: true,
            urlAllowlist: ["cdn.example.com", "*.assets.example.com"],
            allowedMimes: [
              "text/plain",
              "text/markdown",
              "text/html",
              "text/csv",
              "application/json",
              "application/pdf",
            ],
            maxBytes: 5242880,
            maxChars: 60000,
            maxRedirects: 3,
            timeoutMs: 10000,
            pdf: {
              maxPages: 4,
              maxPixels: 4000000,
              minTextChars: 200,
            },
          },
          images: {
            allowUrl: true,
            urlAllowlist: ["images.example.com"],
            allowedMimes: [
              "image/jpeg",
              "image/png",
              "image/gif",
              "image/webp",
              "image/heic",
              "image/heif",
            ],
            maxBytes: 10485760,
            maxRedirects: 3,
            timeoutMs: 10000,
          },
        },
      },
    },
  },
}
```

省略時的預設值：

| 金鑰                      | 預設值   |
| ------------------------ | --------- |
| `maxUrlParts`            | 8         |
| `files.maxBytes`         | 5MB       |
| `files.maxChars`         | 60k       |
| `files.maxRedirects`     | 3         |
| `files.timeoutMs`        | 10s       |
| `files.pdf.maxPages`     | 4         |
| `files.pdf.maxPixels`    | 4,000,000 |
| `files.pdf.minTextChars` | 200       |
| `images.maxBytes`        | 10MB      |
| `images.maxRedirects`    | 3         |
| `images.timeoutMs`       | 10s       |

HEIC/HEIF `input_image` 來源在透過共用的 OpenClaw 圖片處理器（Rastermill）傳送給提供者之前，會正規化為 JPEG；對於需要外部編解碼器支援的格式，則會改用系統轉換器（`sips`、ImageMagick、GraphicsMagick 或 ffmpeg）。

安全性注意事項：系統會在擷取前及每次重新導向時強制套用 URL 允許清單。將主機名稱加入允許清單不會略過私人／內部 IP 封鎖。對於暴露於網際網路的閘道，除了應用程式層級的防護措施外，也請套用網路輸出流量控制。請參閱[安全性](/zh-TW/gateway/security)。

## 串流（SSE）

設定 `stream: true` 以接收伺服器傳送事件：

- `Content-Type: text/event-stream`
- 每個事件行都是 `event: <type>` 和 `data: <json>`
- 串流以 `data: [DONE]` 結束

目前發出的事件類型：`response.created`、`response.in_progress`、`response.output_item.added`、`response.content_part.added`、`response.output_text.delta`、`response.output_text.done`、`response.content_part.done`、`response.output_item.done`、`response.completed`、`response.failed`（發生錯誤時）。

## 使用量

當底層提供者回報權杖數量時，系統會填入 `usage`。在這些計數器傳至下游狀態／工作階段介面之前，OpenClaw 會正規化常見的 OpenAI 風格別名，包括 `input_tokens` / `output_tokens` 和 `prompt_tokens` / `completion_tokens`。

## 錯誤

錯誤使用如下的 JSON 物件：

```json
{ "error": { "message": "...", "type": "invalid_request_error" } }
```

常見情況：`400` 無效的請求本文、`401` 缺少／無效的驗證、`403` 缺少操作員範圍、`405` 方法錯誤、`429` 驗證失敗次數過多（包含 `Retry-After`）。

## 範例

非串流：

```bash
curl -sS http://127.0.0.1:18789/v1/responses \
  -H 'Authorization: Bearer YOUR_TOKEN' \
  -H 'Content-Type: application/json' \
  -H 'x-openclaw-agent-id: main' \
  -d '{
    "model": "openclaw",
    "input": "hi"
  }'
```

串流：

```bash
curl -N http://127.0.0.1:18789/v1/responses \
  -H 'Authorization: Bearer YOUR_TOKEN' \
  -H 'Content-Type: application/json' \
  -H 'x-openclaw-agent-id: main' \
  -d '{
    "model": "openclaw",
    "stream": true,
    "input": "hi"
  }'
```

## 相關內容

- [OpenAI 聊天補全](/zh-TW/gateway/openai-http-api)
- [操作員範圍](/zh-TW/gateway/operator-scopes)
- [OpenAI](/zh-TW/providers/openai)
