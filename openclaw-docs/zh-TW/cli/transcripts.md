---
read_when:
    - 你想要從終端機讀取已儲存的逐字稿摘要
    - 你需要逐字稿 Markdown 摘要的路徑
    - 你正在偵錯核心逐字稿的儲存配置
summary: '`openclaw transcripts` 的命令列介面參考（列出、顯示及匯出已儲存的逐字稿）'
title: 對話記錄命令列介面
x-i18n:
    generated_at: "2026-07-26T07:15:43Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: c04ba637fb46ec271383b2f0d17655e18018e07f489c34dc3fd14ad926f27aa4
    source_path: cli/transcripts.md
    workflow: 16
---

# `openclaw transcripts`

用於持久化會議逐字稿的檢查與匯出命令。Google Meet、
Microsoft Teams 和 Zoom 瀏覽器參與者會自動擷取筆記；
`transcripts` 代理程式工具也支援提供者擷取與手動匯入。

標準逐字稿狀態位於共享 SQLite 資料庫的
`$OPENCLAW_STATE_DIR/state/openclaw.sqlite`。`show` 和 `path` 會明確地將
面向使用者的成品具體化至狀態目錄下：

```text
$OPENCLAW_STATE_DIR/transcripts/YYYY-MM-DD/<session>/
  metadata.json
  transcript.jsonl
  summary.json
  summary.md
```

這些檔案是匯出成品，而非第二個執行階段儲存區。OpenClaw 不會在
擷取、摘要或列出期間讀回這些檔案。預設狀態目錄為
`~/.openclaw`；可使用 `OPENCLAW_STATE_DIR` 覆寫。日期目錄取自
工作階段開始時間；工作階段目錄則是由工作階段 ID 衍生的檔案系統安全 slug。

## 命令

```bash
openclaw transcripts list
openclaw transcripts show <session>
openclaw transcripts show YYYY-MM-DD/<session>
openclaw transcripts path <session>
openclaw transcripts path YYYY-MM-DD/<session>
openclaw transcripts path <session> --dir
openclaw transcripts path <session> --metadata
openclaw transcripts path <session> --transcript
openclaw transcripts list --json
openclaw transcripts show <session> --json
openclaw transcripts path <session> --json
```

| 命令                          | 說明                                                 |
| ----------------------------- | ---------------------------------------------------- |
| `list`                        | 列出已儲存的工作階段。                               |
| `show <session>`              | 輸出並具體化 `summary.md`。                  |
| `path <session>`              | 具體化並輸出 `summary.md` 路徑。             |
| `path <session> --dir`        | 具體化所有成品並輸出其目錄。                         |
| `path <session> --metadata`   | 具體化並輸出 `metadata.json`。               |
| `path <session> --transcript` | 具體化並輸出 `transcript.jsonl`。            |
| `--json`                      | 輸出機器可讀的結果（適用於任何子命令）。             |

`<session>` 接受單獨的工作階段 ID 或包含日期的選擇器
（`YYYY-MM-DD/<session>`）。當同一工作階段 ID 出現在多個日期時，
請使用包含日期的形式，例如 `openclaw transcripts show
2026-05-22/standup`。預設工作階段 ID 包含時間戳記和隨機
尾碼；只有在該 ID 於當日唯一時，才應為工作階段指定固定 ID。

## 輸出

`list` 會為每個工作階段輸出一行以定位字元分隔的內容：選擇器、開始時間、標題、
摘要路徑。

```text
2026-05-22/standup  2026-05-22T09:00:00.000Z  每週站立會議  /Users/user/.openclaw/transcripts/2026-05-22/standup/summary.md
```

選擇器是傳回給 `show` 或 `path` 最安全的值。

`list --json` 會傳回包含 `sessionId`、`selector`、`date`、`title`、
`startedAt`、`stoppedAt`、`source`、`path`、`summaryPath`、`hasSummary` 的物件。
儲存的會議來源 URL 僅包含來源和路徑；查詢字串、
片段和內嵌認證資訊會在持久化前移除。

`show --json` 會傳回已儲存的工作階段中繼資料、選擇器、工作階段
目錄、摘要路徑和 Markdown 摘要文字。

`path --json` 會傳回所選路徑，以及該成品是否能夠
具體化。已儲存的工作階段一律會有中繼資料與逐字稿匯出；
在工作階段具有摘要之前，摘要路徑會回報 `exists: false`。

## 每日多個工作階段

工作階段先依日期分組，再依工作階段 ID 分組。一天內的十場會議會成為
十個同層資料夾：

```text
~/.openclaw/transcripts/2026-05-22/
  transcript-2026-05-22T09-00-00-000Z-a1b2c3d4/
  transcript-2026-05-22T10-30-00-000Z-b2c3d4e5/
  standup/
```

自動化請使用預設產生的 ID。只有在同一日期不會重複時，才使用
`standup` 這類固定 ID。

## 缺少摘要

即時工作階段會在工作階段停止時儲存並具體化 `summary.md`；
匯入的逐字稿則會在匯入後立即執行此作業。若擷取仍在進行中、提供者
在停止期間失敗，或中繼資料在任何發言抵達前就已儲存，工作階段可能會
出現在 `list` 中但沒有摘要。

使用 `path <session> --transcript` 檢查原始的僅附加逐字稿，
或執行 `transcripts` 工具的 `summarize` 動作以重新產生 Markdown
摘要。

## 升級舊版檔案儲存區

早於 SQLite 儲存區的 OpenClaw 版本會將標準執行階段狀態
直接寫入 `$OPENCLAW_STATE_DIR/transcripts/` 下方。執行：

```bash
openclaw doctor --fix
```

Doctor 會將完整的舊版目錄樹匯入 SQLite、驗證資料列數量與
順序、記錄移轉收據，並將已驗證的來源目錄樹移至帶有時間戳記的
`transcripts.migrated-*` 封存。執行階段命令不會退回使用
舊版檔案。在確認匯入的工作階段以及你所依賴的任何匯出成品之前，
請保留該封存。

## 設定

會議逐字稿擷取預設為啟用。若要全域停用：

```json
{
  "transcripts": {
    "enabled": false
  }
}
```

- `enabled`（預設 `true`）：啟用自動會議筆記、逐字稿
  工具，以及已設定的自動啟動來源。當會議筆記不應持久化於主機上時，
  將其設為 `false`。明確要求的會議
  `transcribe` 模式會保留既有、有界限的即時字幕尾端，但此設定為 false 時
  不會寫入持久化資料列。
  使用 `transcripts.autoStart` 設定自動啟動來源。每個項目只要存在
  即為啟用；省略項目即可停用該來源。`discord-voice`
  是隨附且支援自動啟動的來源，並需要 `guildId` 和
  `channelId`：

```json
{
  "transcripts": {
    "enabled": true,
    "autoStart": [
      {
        "providerId": "discord-voice",
        "guildId": "1234567890",
        "channelId": "2345678901"
      }
    ]
  }
}
```

會議提供者 ID 為 `google-meet`、`teams` 和 `zoom`。其別名
分別為 `googlemeet`/`meet`、`teams-meetings`/`microsoft-teams`/`msteams`，以及
`zoom-meetings`。會議提供者會附加至已啟用的
會議機器人工作階段；一般加入會議不需要 `autoStart` 項目。
