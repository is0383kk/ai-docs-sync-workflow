---
read_when:
    - 你想要從終端機檢查或編輯工作區檔案中的單一葉節點
    - 你正在編寫針對工作區狀態的指令碼，並需要一套穩定且不受類型影響的定址機制
    - 你正在決定是否要在自行託管的閘道上啟用選用的 `oc-path` 外掛
summary: 隨附的 `oc-path` 外掛：提供用於 `oc://` 工作區檔案定址方案的 `openclaw path` 命令列介面
title: OC Path 外掛
x-i18n:
    generated_at: "2026-07-26T08:33:49Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: eb7bb1aacd37e5cc9c391372b871dc519f4048232d93a0016138ae00a6985a59
    source_path: plugins/oc-path.md
    workflow: 16
---

隨附的 `oc-path` 外掛新增了 [`openclaw path`](/zh-TW/cli/path) 命令列介面，用於
`oc://` 工作區檔案定址配置。它隨附於 OpenClaw 儲存庫的
`extensions/oc-path/` 中，但需選擇啟用：安裝／建置後會維持停用，直到你
啟用它。

`oc://` 位址會指向工作區檔案中的單一葉節點（或一組萬用字元比對的葉節點）。
此外掛支援四種檔案類型：

- **markdown**（`.md`）：frontmatter、區段、項目、欄位
- **jsonc**（`.jsonc`、`.json`）：保留註解與格式
- **jsonl**（`.jsonl`、`.ndjson`）：以行為單位的記錄
- **yaml**（`.yaml`、`.yml`、`.lobster`）：透過
  `yaml` 套件的 `Document` API 處理對應表／序列／純量節點

自行託管者與編輯器擴充功能使用命令列介面讀取或寫入單一葉節點，
而不必直接針對 SDK 編寫指令碼；代理程式與鉤子則將它視為
確定性基礎層，因此能在各種檔案類型間一致套用位元組保真往返處理
與遮罩哨兵防護。完整語法、各動詞的旗標清單，以及
各檔案類型的實作範例，請參閱
[命令列介面參考](/zh-TW/cli/path)；本頁說明為何以及如何啟用此外掛。

## 為何要啟用

當指令碼、鉤子或本機代理程式工具需要精確指向
工作區狀態的特定部分，而不想為每種檔案結構編寫專用剖析器時，請啟用 `oc-path`。
單一 `oc://` 位址可以指定 markdown frontmatter 鍵、區段項目、
JSONC 設定葉節點、JSONL 事件欄位或 YAML 工作流程步驟。

這對維護者工作流程很重要，因為變更應保持小範圍、
可稽核且可重複：檢查一個值、尋找相符記錄、試執行
寫入，然後只套用該葉節點，同時不變更註解、行尾格式及
附近的格式。

常見的啟用原因：

- **本機自動化**：Shell 指令碼使用 `openclaw path … --json` 解析或更新單一工作區值，
  不必分別攜帶 markdown、JSONC、
  JSONL 與 YAML 剖析程式碼。
- **代理程式可見的編輯**：代理程式在寫入前顯示單一指定
  葉節點的試執行差異，比任意形式的檔案
  重寫更容易審查。
- **編輯器整合**：編輯器將 `oc://AGENTS.md/tools/gh` 對應至
  確切的 markdown 節點與行號，不必根據標題文字猜測。
- **診斷**：`emit` 讓檔案經過剖析器與輸出器往返處理，
  因此你可以先檢查某種檔案類型的位元組是否穩定，再依賴
  自動化編輯。

```bash
# 此設定是否已啟用 GitHub 外掛？
openclaw path resolve 'oc://config.jsonc/plugins/github/enabled' --json

# 此工作階段記錄中出現了哪些工具呼叫名稱？
openclaw path find 'oc://session.jsonl/[event=tool_call]/name' --json

# 這項小型設定編輯會寫入哪些位元組？
openclaw path set 'oc://config.jsonc/plugins/github/enabled' 'true' --dry-run
```

`oc-path` 刻意不負責高階語意。記憶體
外掛仍負責記憶體寫入，設定命令仍負責完整的設定
管理，而上次已知正常（LKG）設定復原仍負責
還原／升級。`oc-path` 是狹義的定址與位元組保留
檔案操作層，讓這些高階工具可以建構於其上。

## 執行位置

此外掛會在你叫用命令的主機上，**於 `openclaw` 命令列介面處理程序內執行**。
它不需要執行中的閘道，也不會開啟任何
網路通訊端；每個動詞都只會對你指定的檔案執行純轉換。

外掛中繼資料位於 `extensions/oc-path/openclaw.plugin.json`：

```json
{
  "id": "oc-path",
  "name": "OC Path",
  "activation": {
    "onStartup": false,
    "onCommands": ["path"]
  },
  "commandAliases": [{ "name": "path", "kind": "cli" }]
}
```

`onStartup: false` 會讓此外掛不進入閘道啟動路徑。
`commandAliases` 與 `activation.onCommands` 會指示命令列介面，在你第一次執行
`openclaw path …` 時才延遲載入此外掛，因此從未使用
此動詞的安裝環境不會產生任何成本。

## 啟用

```bash
openclaw plugins enable oc-path
```

重新啟動閘道（如果你有執行），讓資訊清單快照取得新的
狀態。在同一台主機上直接叫用 `openclaw path` 會立即生效；
命令列介面會視需要載入此外掛。

停用方式：

```bash
openclaw plugins disable oc-path
```

## 相依套件

所有剖析器相依套件皆為此外掛本機專用；啟用 `oc-path` 不會將
新套件引入核心執行階段：

| 相依套件       | 用途                                                                   |
| -------------- | ---------------------------------------------------------------------- |
| `commander`    | 為 `resolve`、`find`、`set`、`validate`、`emit` 連接子命令。    |
| `jsonc-parser` | 剖析與編輯 JSONC 葉節點，同時保留註解及尾端逗號。     |
| `markdown-it`  | 為區段／項目／欄位模型進行 Markdown 詞法分析。            |
| `yaml`         | 剖析／輸出／編輯 YAML `Document`，同時保留註解與流程樣式。 |

JSONL 維持手動實作：以行為單位的剖析比使用任何
相依套件都更簡單，而且每行剖析已經會經過 `jsonc-parser`。

## 提供的功能

| 介面                           | 提供者                                                  |
| ------------------------------ | ------------------------------------------------------- |
| `openclaw path` 命令列介面            | `extensions/oc-path/cli-registration.ts`                |
| `oc://` 剖析器／格式化工具     | `extensions/oc-path/src/oc-path/oc-path.ts`             |
| 各類型的剖析／輸出／編輯       | `extensions/oc-path/src/oc-path/{md,jsonc,jsonl,yaml}`  |
| 通用解析／尋找／設定            | `extensions/oc-path/src/oc-path/{resolve,find,edit}.ts` |
| 遮罩哨兵防護                    | `extensions/oc-path/src/oc-path/sentinel.ts`            |

命令列介面是目前唯一的公開介面。基礎層動詞為
此外掛私有；使用者透過命令列介面操作（或根據
SDK 建置自己的外掛）。

## 與其他外掛的關係

- **`memory-*`**：記憶體寫入會經過記憶體外掛，而非
  `oc-path`。`oc-path` 是通用檔案基礎層；記憶體外掛會在其上
  疊加自己的語意。
- **LKG**：`path` 不知道上次已知正常設定的還原機制。如果你
  透過 `path` 編輯的檔案也由 LKG 追蹤，下一次設定觀察
  週期會決定要升級還是復原；請將 `path` 編輯視為
  對該檔案的任何其他直接寫入。

## 安全性

`set` 會透過基礎層的輸出路徑寫入原始位元組，該路徑會自動套用
遮罩哨兵防護。若葉節點包含
`__OPENCLAW_REDACTED__`（逐字相同或作為子字串），系統會在寫入時拒絕，
並顯示 `OC_EMIT_SENTINEL`。命令列介面也會從它印出的任何
人類可讀或 JSON 輸出中清除該哨兵字面值，並以 `[REDACTED]` 取代，
確保終端擷取內容與管線絕不洩漏此標記。

## 相關內容

- [`openclaw path` 命令列介面參考](/zh-TW/cli/path)
- [管理外掛](/zh-TW/plugins/manage-plugins)
- [建置外掛](/zh-TW/plugins/building-plugins)
