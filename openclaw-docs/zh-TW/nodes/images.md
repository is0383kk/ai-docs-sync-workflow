---
read_when:
    - 修改媒體流水線或附件
summary: 傳送、閘道與代理程式回覆的圖片及媒體處理規則
title: 圖片與媒體支援
x-i18n:
    generated_at: "2026-07-26T07:55:15Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 71f5591f4268593c142056370802b702899787a79f9ca1fbde6ea8e422f34023
    source_path: nodes/images.md
    workflow: 16
---

WhatsApp 頻道在 Baileys Web 上執行。本頁說明傳送、閘道及代理程式回覆的媒體處理規則。

## 目標

- 透過 `openclaw message send --media` 傳送媒體，並可選擇附加說明文字。
- 允許來自網頁收件匣的自動回覆同時包含媒體與文字。
- 讓各類型的限制維持合理且可預期。

## 命令列介面介面

`openclaw message send --target <dest> --media <path-or-url> [--message <caption>]`

- `--media <path-or-url>` — 附加媒體（圖片／音訊／影片／文件）；接受本機路徑或 URL。此項為選填；僅傳送媒體時，說明文字可以留空。
- `--gif-playback` — 將影片媒體視為 GIF 播放（僅限 WhatsApp）。
- `--force-document` — 將媒體以文件形式傳送，以避免頻道壓縮（Telegram、WhatsApp）；適用於圖片、GIF 及影片。
- `--reply-to <id>`、`--thread-id <id>`、`--pin`、`--silent` — 與純文字傳送共用的遞送／討論串選項。
- `--dry-run` — 輸出解析後的承載資料並略過傳送。
- `--json` — 將結果輸出為 JSON：`{ action, channel, dryRun, handledBy, messageId?, payload }`（`payload` 包含頻道特定的傳送結果，包括任何媒體參照）。

## WhatsApp Web 頻道行為

- 輸入：本機檔案路徑**或** HTTP(S) URL。
- 流程：載入緩衝區、偵測媒體類型，然後依類型建立外送承載資料：
  - **圖片：**最佳化至低於 `channels.whatsapp.mediaMaxMb`（預設 50MB）。不透明圖片會重新壓縮為 JPEG（預設邊長階梯從 2048px 開始，若反覆未符合大小限制便逐步降低）；含透明度的圖片則保留為 PNG。如果來源已是符合大小及邊長預算的可接受 JPEG／PNG／WebP，系統會原封不動保留原始位元組，而不重新壓縮。動畫 GIF 絕不重新編碼，只檢查大小。
  - **音訊／語音：**除非已是原生語音音訊（`.ogg`/`.opus` 或 `audio/ogg`/`audio/opus`），否則外送音訊會在傳送前透過 `ffmpeg` 轉碼為 Opus/OGG（48kHz 單聲道、64kbps、最長 20 分鐘），並以語音訊息（`ptt: true`）形式傳送。
  - **影片：**不經處理直接傳送，上限為 16MB。
  - **文件：**其他任何類型，上限為 100MB；若有檔名則予以保留。
- WhatsApp GIF 樣式播放：傳送帶有 `gifPlayback: true` 的 MP4（命令列介面：`--gif-playback`），讓行動版用戶端在行內循環播放。
- MIME 偵測會依序優先採用探測到的魔術位元組、檔案副檔名，再來是回應標頭；探測到的一般容器（`application/octet-stream`、`zip`）絕不會覆寫更具體的副檔名對應（例如 XLSX 與 ZIP）。
- 說明文字來自 `--message` 或 `reply.text`；允許空白說明文字。
- 記錄：非詳細模式顯示 `↩️`/`✅`；詳細模式則包含大小及來源路徑／URL。

<Note>
上述 16MB 音訊／影片及 100MB 文件數值，是未傳入明確位元組上限時使用的各類型共用媒體預設值。WhatsApp 傳送會從 `channels.whatsapp.mediaMaxMb` 設定明確上限（預設 50MB），並一致套用至該帳號的所有類型。
</Note>

## 自動回覆流水線

- `getReplyFromConfig` 會傳回回覆承載資料（或承載資料陣列），其中包含 `text?`、`mediaUrl?`、`mediaUrls?` 等欄位。
- 若有媒體，網頁傳送端會使用與 `openclaw message send` 相同的流水線解析本機路徑或 URL。
- 若提供多個媒體項目，會依序傳送。

## 傳入媒體至命令

- 當傳入的網頁訊息包含媒體時，OpenClaw 會將其下載至暫存檔，並提供下列範本變數：
  - `{{AttachmentUrl}}` — 目前附件的原始 URL 或提供者參照。
  - `{{AttachmentPath}}` — 執行命令前寫入的本機暫存路徑。
  - `{{AttachmentContentType}}` — MIME 內容類型。
  - `{{AttachmentDir}}` — 包含本機路徑的目錄。
  - `{{AttachmentIndex}}` — 從零開始的來源事實索引。
- 啟用每工作階段 Docker 沙箱時，傳入媒體會複製到沙箱工作區，且附件路徑／參照會改寫為類似 `media/inbound/<filename>` 的沙箱相對路徑。
- 在外掛 SDK 遷移期間，`{{MediaPath}}`、`{{MediaUrl}}`、`{{MediaType}}` 和 `{{MediaDir}}` 仍作為已棄用的相容性別名。
- 媒體理解（透過 `tools.media.*` 或共用的 `tools.media.models` 設定）會在範本化之前執行，並可將 `[Image]`、`[Audio]` 和 `[Video]` 區塊插入 `Body`。
  - 音訊會設定 `{{Transcript}}`，並使用轉錄文字剖析命令，讓斜線命令仍可運作。
  - 影片及圖片描述會保留任何說明文字，以供命令剖析使用。
  - 如果目前使用中的主要模型已原生支援視覺，OpenClaw 會略過 `[Image]` 摘要區塊，改為將原始圖片傳給模型。
- 預設只處理第一個相符的圖片／音訊／影片附件；使用 `tools.media.<capability>.attachments` 可選取多個附件。

## 限制與錯誤

**外送傳送上限（WhatsApp 網頁傳送）**

- 圖片：最佳化後上限為 `channels.whatsapp.mediaMaxMb`（預設 50MB）。
- 音訊／影片：上限為 16MB（共用預設值；透過 WhatsApp 傳送時由 `mediaMaxMb` 覆寫）。
- 文件：上限為 100MB（共用預設值；透過 WhatsApp 傳送時由 `mediaMaxMb` 覆寫）。
- 媒體過大或無法讀取時，記錄中會產生清楚的錯誤，並略過該回覆。

**媒體理解上限（轉錄／描述）**

- 圖片預設值：10MB（可使用 `tools.media.image.maxBytes` 覆寫，或在每個
  `tools.media.models[]` 項目中使用 `maxBytes`）。
- 音訊預設值：20MB（可使用 `tools.media.audio.maxBytes` 或針對個別項目覆寫）。
- 影片預設值：50MB（可使用 `tools.media.video.maxBytes` 或針對個別項目覆寫）。
- 媒體過大時會略過理解，但仍會使用原始本文送出回覆。

## 測試注意事項

- 涵蓋圖片／音訊／文件案例的傳送與回覆流程。
- 驗證圖片最佳化後的大小範圍，以及音訊的語音訊息旗標。
- 確保多媒體回覆展開為依序傳送。

## 相關內容

- [相機擷取](/zh-TW/nodes/camera)
- [媒體理解](/zh-TW/nodes/media-understanding)
- [音訊與語音訊息](/zh-TW/nodes/audio)
