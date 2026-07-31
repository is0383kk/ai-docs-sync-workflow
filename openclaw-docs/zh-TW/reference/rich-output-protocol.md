---
read_when:
    - 變更控制介面中的助理輸出呈現方式
    - 偵錯 `[embed ...]`、結構化媒體、回覆或音訊呈現指令
summary: 結構化媒體、嵌入內容、音訊提示與回覆的豐富輸出通訊協定
title: 豐富輸出通訊協定
x-i18n:
    generated_at: "2026-07-26T08:13:32Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: cbfe68f38c871f5f6d2811eb52b18d0143606f30283023ae96db64543eed95a1
    source_path: reference/rich-output-protocol.md
    workflow: 16
---

助理輸出會透過數個專用管道攜帶傳遞／轉譯指令：

- 用於附件傳遞的結構化 `mediaUrl` / `mediaUrls` 欄位。
- `[[audio_as_voice]]` 用於音訊呈現提示。
- `[[reply_to_current]]` / `[[reply_to:<id>]]` 用於回覆中繼資料。
- `[embed ...]` 用於控制介面的豐富轉譯。

結構化媒體欄位和 `[[...]]` 標籤是傳遞中繼資料。`[embed ...]` 是獨立且僅限網頁使用的豐富轉譯路徑；它不是媒體別名。

## 媒體附件

遠端附件必須是公開的 `https:` URL。`http:`、迴路、連結本機、私有和內部主機名稱會被拒絕作為附件指令；伺服器端媒體擷取器還會另外套用自己的網路防護措施。

本機附件接受絕對路徑、工作區相對路徑或相對於家目錄的 `~/` 路徑。在傳遞前，它們仍須通過代理程式檔案讀取政策和媒體類型檢查。

<Warning>
請勿從工具、外掛、串流區塊、瀏覽器輸出或訊息動作發出附件的文字命令。請改用結構化媒體欄位：

```json
{ "message": "這是你的圖片。", "mediaUrl": "/workspace/image.png" }
```

為了相容性，舊版最終回覆文字仍可能會正規化，但這不是通用的外掛／工具通訊協定。
</Warning>

純 Markdown 圖片語法（`![alt](url)`）預設會保留為文字。若管道想將 Markdown 圖片視為媒體回覆，須在其傳出配接器中選擇啟用；Telegram 會這麼做，因此 `![alt](url)` 會成為媒體附件。

啟用區塊串流時，媒體必須透過結構化承載資料欄位傳送。如果相同的媒體 URL 出現在串流區塊中，之後又出現在最終助理承載資料中，OpenClaw 只會傳遞一次，並從最終承載資料中移除重複項目。

## `[embed ...]`

`[embed ...]` 是控制介面唯一面向代理程式的豐富轉譯語法。自閉合範例：

```text
[embed ref="cv_123" title="Status" /]
```

規則：

- `[view ...]` 不再適用於新的輸出。
- 嵌入短代碼只會在助理訊息介面中轉譯。
- 只有以 URL 為基礎的嵌入內容會轉譯；請使用 `ref="..."` 或 `url="..."`。
- 區塊形式的行內 HTML 嵌入短代碼不會轉譯。
- 網頁介面會從可見文字中移除短代碼，並在行內轉譯嵌入內容。

## 儲存的轉譯結構

正規化／儲存後的助理內容區塊是結構化的 `canvas` 項目：

```json
{
  "type": "canvas",
  "preview": {
    "kind": "canvas",
    "surface": "assistant_message",
    "render": "url",
    "viewId": "cv_123",
    "url": "/__openclaw__/canvas/documents/cv_123/index.html",
    "title": "狀態",
    "preferredHeight": 320
  }
}
```

系統不會識別 `present_view`；儲存／轉譯的豐富內容區塊一律使用此 `canvas` 結構。

## 相關內容

- [RPC 配接器](/zh-TW/reference/rpc)
- [Typebox](/zh-TW/concepts/typebox)
