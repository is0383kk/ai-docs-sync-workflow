---
read_when:
    - 你想要從終端機讀取或寫入工作區檔案中的葉節點
    - 你正在編寫與工作區狀態互動的指令碼，並希望採用穩定且不受類型限制的定址方式
    - 你正在偵錯 `oc://` 路徑（驗證語法，查看它會解析成什麼）
summary: '`openclaw path` 的命令列介面參考（透過 `oc://` 定址方式檢查及編輯工作區檔案）'
title: 路徑
x-i18n:
    generated_at: "2026-07-26T08:19:50Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 7afe5bd1c3a5fca8dd22c7d807e390e751ae7e895c54bf0e10e2734f3889436c
    source_path: cli/path.md
    workflow: 16
---

# `openclaw path`

透過 Shell 存取 `oc://` 定址機制：一種依類型分派的路徑語法，
用於檢查及編輯可定址的工作區檔案（markdown、jsonc、
jsonl、yaml/yml/lobster）。自行託管者、外掛作者及編輯器擴充功能
可使用它讀取、尋找或更新特定位置，而無須為每種檔案
自行編寫剖析器。

`path` 由隨附的選用 `oc-path` 外掛提供。首次
使用前請先啟用：

```bash
openclaw plugins enable oc-path
```

命令列介面的動詞與定址模型相呼應：

- `resolve` 是具體且僅比對單一結果的動詞。
- `find` 是用於萬用字元、聯集、述詞及
  位置展開的多重比對動詞。
- `set` 僅接受具體路徑或插入標記；萬用字元模式
  會在寫入前遭到拒絕。
- `validate` 會剖析路徑，但不存取檔案系統。
- `emit` 會讓檔案經過剖析與輸出往返處理（位元組保真度診斷）。

## 為何使用它

OpenClaw 狀態分散於人工編輯的 markdown、帶註解的 JSONC
設定、僅附加的 JSONL 記錄，以及 YAML 工作流程／規格檔案中。指令碼、鉤子
及代理程式通常只需要這些檔案中的一個小值：frontmatter 鍵、
外掛設定、記錄欄位、YAML 步驟，或具名章節下的項目符號
項目。

`openclaw path` 為這些呼叫端提供穩定的位址，而不必針對每種檔案類型
使用臨時的 grep、規則運算式或剖析器。同一個 `oc://` 路徑可從終端機進行驗證、
解析、搜尋、試執行及寫入，讓小範圍的
自動化易於審查及重播。它會保留檔案的其餘部分，因此
寫入一個葉節點不會擾動其註解、換行字元或鄰近
格式。

當所需項目有邏輯位址，但檔案結構
各異時，請使用它：

- 鉤子從帶註解的 JSONC 讀取一項設定，並在
  寫回值時保留註解。
- 維護指令碼尋找 JSONL 記錄中每個相符的事件欄位，
  而不必將整份記錄載入自訂剖析器。
- 編輯器依 slug 跳至 markdown 章節或項目符號項目，接著呈現
  解析所得的確切行。
- 代理程式在套用小型工作區編輯前先試執行，並在
  審查中顯示變更的位元組。

一般的整份檔案編輯、複雜設定遷移或
記憶體專用寫入應略過 `openclaw path`；這些作業應使用擁有者命令或外掛。`path`
適用於小型、可定址的檔案作業，此時可重複執行的終端機命令
比再製作一個專用剖析器更合適。

## 使用方式

從人工編輯的設定檔讀取一個值：

```bash
openclaw path resolve 'oc://config.jsonc/plugins/github/enabled'
```

預覽寫入而不變更磁碟：

```bash
openclaw path set 'oc://config.jsonc/plugins/github/enabled' 'true' --dry-run
```

尋找僅附加 JSONL 記錄中的相符紀錄：

```bash
openclaw path find 'oc://session.jsonl/[event=tool_call]/name'
```

依章節及項目而非行號，定址 markdown 中的指示：

```bash
openclaw path resolve 'oc://AGENTS.md/runtime-safety/openclaw-gateway'
```

在指令碼讀取或寫入前，於 CI 或預檢指令碼中驗證路徑：

```bash
openclaw path validate 'oc://AGENTS.md/tools/$last/risk'
```

這些命令可直接複製到 Shell 指令碼中。呼叫端需要結構化輸出時，
請使用 `--json`；人員檢查結果時則使用 `--human`。

## 運作方式

1. 將 `oc://` 位址剖析成欄位：檔案、章節、項目、欄位及
   選用的工作階段查詢。
2. 依目標副檔名選擇檔案類型配接器（`.md`、`.jsonc`、
   `.json`、`.jsonl`、`.ndjson`、`.yaml`、`.yml`、`.lobster`）。
3. 依該檔案類型的結構解析欄位：markdown
   標題／項目、JSONC 物件鍵／陣列索引、JSONL 行紀錄，或
   YAML 對應／序列節點。
4. 針對 `set`，透過同一個配接器輸出已編輯的位元組，使檔案中
   未變更的部分在該類型支援時保留其註解、換行字元及鄰近
   格式。

`resolve` 與 `set` 需要單一具體目標。`find` 是探索用
動詞：它會將萬用字元、聯集、述詞及序數展開成具體
比對結果，讓你在選擇要寫入的目標前先行檢查。

## 子命令

| 子命令                  | 用途                                                                        |
| ----------------------- | --------------------------------------------------------------------------- |
| `resolve <oc-path>`     | 輸出路徑上的具體比對結果（或「找不到」）。                                  |
| `find <pattern>`        | 列舉萬用字元／聯集／述詞路徑的比對結果。                                    |
| `set <oc-path> <value>` | 在具體路徑寫入葉節點或插入目標。支援 `--dry-run`。                   |
| `validate <oc-path>`    | 僅剖析；輸出結構分解（檔案／章節／項目／欄位）。                            |
| `emit <file>`           | 讓檔案經過剖析與輸出往返處理（位元組保真度診斷）。                          |

## 全域旗標

| 旗標            | 適用於                           | 用途                                                                     |
| --------------- | -------------------------------- | ------------------------------------------------------------------------ |
| `--cwd <dir>`   | `resolve`、`find`、`set`、`emit` | 相對於此目錄解析檔案欄位（預設值：`process.cwd()`）。                 |
| `--file <path>` | `resolve`、`find`、`set`、`emit` | 覆寫檔案欄位解析後的路徑（絕對路徑存取）。                               |
| `--json`        | 全部                             | 強制輸出 JSON（stdout 不是 TTY 時的預設值）。                             |
| `--human`       | 全部                             | 強制輸出供人閱讀的內容（stdout 是 TTY 時的預設值）。                     |
| `--value-json`  | `set`                            | 將 `<value>` 剖析為 JSON，以替換 JSON/JSONC/JSONL 葉節點。       |
| `--dry-run`     | `set`                            | 輸出原本會寫入的位元組，但不實際寫入。                                   |
| `--diff`        | `set`（需要 `--dry-run`） | 輸出統一格式差異，而非完整位元組。                                       |

`validate` 僅接受 `--json`／`--human`；它不會存取檔案系統，因此
`--cwd` 與 `--file` 不適用。

## `oc://` 語法

```text
oc://FILE/SECTION/ITEM/FIELD?session=SCOPE
```

欄位規則：`field` 需要 `item`，而 `item` 需要 `section`。四個
欄位均適用以下規則：

- **加引號的區段** — `"a/b.c"` 可跨越 `/` 與 `.` 分隔符號。內容是
  位元組常值；引號內不允許 `"` 與 `\`。檔案欄位也能辨識引號：`oc://"skills/email-drafter"/Tools/$last` 會將
  `skills/email-drafter` 視為單一檔案路徑。
- **述詞** — `[k=v]`、`[k!=v]`、`[k<v]`、`[k<=v]`、`[k>v]`、`[k>=v]`。
  數值運算子要求兩側均可強制轉換為有限數值。
- **聯集** — `{a,b,c}` 會比對任一替代項目。
- **萬用字元** — `*`（單一子區段）及 `**`（零個或多個，
  遞迴）。`find` 接受這些語法；`resolve` 與 `set` 會因語意
  不明確而拒絕。
- **位置** — `$first`／`$last` 會解析為第一個／最後一個索引或
  已宣告的鍵。
- **序數** — `#N` 代表依文件順序排列的第 N 個比對結果。
- **插入標記** — `+`、`+key`、`+nnn` 用於依鍵／索引插入
  （搭配 `set` 使用）。
- **工作階段範圍** — `?session=cron-daily` 等。與欄位巢狀結構
  彼此獨立。工作階段值為原始值，不會進行百分比解碼；其中不得包含控制
  字元或保留的查詢分隔符號（`?`、`&`、`%`）。

位於加引號、述詞或聯集區段之外的保留字元（`?`、`&`、`%`）
會遭到拒絕。控制字元（U+0000-U+001F、U+007F）在任何位置均
會遭到拒絕，包括 `session` 查詢值。

標準路徑保證支援 `formatOcPath(parseOcPath(path)) === path`。
除第一個非空白的 `session=` 值外，非標準查詢參數會被忽略。

硬性限制：路徑上限為 4096 位元組，最多 4 個欄位（檔案／章節／項目／
欄位），每個欄位最多 64 個以點分隔的子區段，而深層 JSON 路徑最多
256 層巢狀周遊。另外，對於任何會載入 JSONC/JSON 檔案的動詞，
超過 16 MiB 的檔案輸入都會遭到拒絕並傳回剖析診斷，而不會進行剖析。

## 依檔案類型定址

| 類型          | 副檔名                      | 定址模型                                                                                            |
| ------------- | --------------------------- | --------------------------------------------------------------------------------------------------- |
| Markdown      | `.md`                       | 依 slug 定址 H2 章節、依 slug 或 `#N` 定址項目符號項目，以及透過 `[frontmatter]` 定址 frontmatter。 |
| JSONC/JSON    | `.jsonc`、`.json`           | 物件鍵及陣列索引；除非加上引號，否則點會分隔巢狀子區段。                                           |
| JSONL         | `.jsonl`、`.ndjson`         | 頂層行位址（`L1`、`L2`、`$first`、`$last`），接著在行內依 JSONC 樣式向下存取。 |
| YAML/.lobster | `.yaml`、`.yml`、`.lobster` | 對應鍵及序列索引；註解及流式樣式由 YAML 文件 API 處理。                                            |

`resolve` 會傳回結構化比對結果：`root`、`node`、`leaf` 或
`insertion-point`，並包含從 1 起算的行號。葉節點值會以
文字加上 `leafType` 的形式呈現，讓外掛作者無須依賴
各檔案類型的 AST 結構即可呈現預覽。

## 變更合約

`set` 會寫入一個具體目標：

- Markdown frontmatter 值和 `- key: value` 項目欄位都是字串
  葉節點。Markdown 插入會附加章節、frontmatter 鍵或章節
  項目，並為已變更的檔案呈現標準 Markdown 形式。無法透過 `set`
  將章節本文當作整體寫入。
- JSONC 葉節點寫入會將字串值強制轉換為現有葉節點型別
  （`string`、有限 `number`、`true`/`false` 或 `null`）。當 JSONC/JSON/JSONL 葉節點替換應將 `<value>` 剖析為 JSON，
  並且可能改變結構時（例如以物件取代字串形式的 secret-ref 簡寫），請使用 `--value-json`。
  JSONC 物件和陣列插入會將 `<value>` 剖析為 JSON，並對一般葉節點寫入使用
  `jsonc-parser` 編輯路徑，以保留註解
  和鄰近格式。
- JSONL 葉節點寫入會在單行內比照 JSONC 進行強制轉換。整行替換
  和附加會將 `<value>` 剖析為 JSON。呈現後的 JSONL 會保留檔案的
  主要 LF/CRLF 行尾慣例（依檔案中的換行採多數決，
  因此主要使用 CRLF 的檔案即使夾雜少數 LF，仍會維持 CRLF）。
- YAML 葉節點寫入會強制轉換為現有純量型別（`string`、有限
  `number`、`true`/`false` 或 `null`）。YAML 插入會使用隨附的
  `yaml` 套件文件 API 更新對應表／序列。若 YAML
  文件格式錯誤且剖析器回報錯誤，系統會在變更前拒絕操作並回傳
  `parse-error`。

若使用者可見的寫入內容必須維持完全相同的位元組，請先使用 `--dry-run`。JSONC
和 YAML 編輯會修補現有文件（透過 `jsonc-parser` 或 `yaml`
文件 API），因此未變更的位元組通常會保留；Markdown 則會在任何編輯時
依剖析後的結構重建檔案，這可能會將已變更葉節點以外的非必要
格式標準化。若希望預覽聚焦於變更前後的修補內容，而非完整呈現的檔案，
請加上 `--diff`。

## 範例

```bash
# 驗證路徑（不存取檔案系統）
openclaw path validate 'oc://AGENTS.md/Tools/$last/risk'

# 讀取葉節點
openclaw path resolve 'oc://gateway.jsonc/version'

# 萬用字元搜尋
openclaw path find 'oc://session.jsonl/*/event' --file ./logs/session.jsonl

# 試執行寫入
openclaw path set 'oc://gateway.jsonc/version' '2.0' --dry-run

# 將寫入試執行為統一差異
openclaw path set 'oc://gateway.jsonc/version' '2.0' --dry-run --diff

# 套用寫入
openclaw path set 'oc://gateway.jsonc/version' '2.0'

# 位元組保真往返轉換（診斷）
openclaw path emit ./AGENTS.md
```

更多語法範例：

```bash
# 為包含 / 或 . 的鍵加上引號
openclaw path resolve 'oc://config.jsonc/agents.defaults.models/"anthropic/claude-opus-4-7"/alias'

# 深層 JSON/JSONC 路徑可使用斜線區段；這些區段會標準化為以點分隔的子區段
openclaw path set 'oc://openclaw.json/agents/list/0/tools/exec/security' 'allowlist' --dry-run

# 以剖析後的物件取代 JSONC 葉節點
openclaw path set 'oc://openclaw.json/gateway/auth/token' '{"source":"file","provider":"secrets","id":"/test"}' --value-json --dry-run

# 對 JSONC 子節點執行述詞搜尋
openclaw path find 'oc://config.jsonc/plugins/[enabled=true]/id'

# 插入 JSONC 陣列
openclaw path set 'oc://config.jsonc/items/+1' '{"id":"new","enabled":true}' --dry-run

# 插入 JSONC 物件鍵
openclaw path set 'oc://config.jsonc/plugins/+github' '{"enabled":true}' --dry-run

# 附加 JSONL 事件
openclaw path set 'oc://session.jsonl/+' '{"event":"checkpoint","ok":true}' --file ./logs/session.jsonl

# 解析最後一個 JSONL 值所在的行
openclaw path resolve 'oc://session.jsonl/$last/event' --file ./logs/session.jsonl

# 解析 YAML 工作流程步驟
openclaw path resolve 'oc://workflow.yaml/steps/0/id'

# 更新 YAML 純量
openclaw path set 'oc://workflow.yaml/steps/$last/id' 'classify-renamed' --dry-run

# 定址 Markdown frontmatter
openclaw path resolve 'oc://AGENTS.md/[frontmatter]/name'

# 插入 Markdown frontmatter
openclaw path set 'oc://AGENTS.md/[frontmatter]/+description' 'Agent instructions' --dry-run

# 尋找 Markdown 項目欄位
openclaw path find 'oc://SKILL.md/Tools/*/send_email'

# 驗證工作階段範圍的路徑
openclaw path validate 'oc://AGENTS.md/Tools/$last/risk?session=cron-daily'
```

## 依檔案種類分類的作法

相同的五個動詞適用於所有種類；定址配置會依
副檔名分派。

### Markdown

```text
<!-- frontmatter.md -->
---
name: drafter
description: 電子郵件草稿代理程式
tier: core
---
## 工具
- gh: GitHub 命令列介面
- curl: HTTP 用戶端
- send_email: 已啟用
```

```bash
$ openclaw path resolve 'oc://x.md/[frontmatter]/tier' --file frontmatter.md --human
葉節點 @ L4: "core"（字串）

$ openclaw path resolve 'oc://x.md/tools/gh/gh' --file frontmatter.md --human
葉節點 @ L9: "GitHub CLI"（字串）

$ openclaw path find 'oc://x.md/tools/*' --file frontmatter.md --human
oc://x.md/tools/* 有 3 個相符項目：
  oc://x.md/tools/gh           →  節點 @ L9 [md-item]
  oc://x.md/tools/curl         →  節點 @ L10 [md-item]
  oc://x.md/tools/send-email   →  節點 @ L11 [md-item]
```

`[frontmatter]` 述詞會定址 YAML frontmatter 區塊；`tools`
會透過 slug 比對 `## Tools` 標題，而項目葉節點即使原始內容使用底線，
仍會保留其 slug 形式（`send_email` 會變成 `send-email`）。

### JSONC

```text
// config.jsonc
{
  "plugins": {
    "github": {"enabled": true, "role": "vcs"},
    "slack":  {"enabled": false, "role": "chat"}
  }
}
```

```bash
$ openclaw path resolve 'oc://config.jsonc/plugins/github/enabled' --file config.jsonc --human
葉節點 @ L4: "true"（布林值）

$ openclaw path set 'oc://config.jsonc/plugins/slack/enabled' 'true' --file config.jsonc --dry-run
--dry-run: 將寫入 142 位元組至 /…/config.jsonc
{
  "plugins": {
    "github": {"enabled": true, "role": "vcs"},
    "slack":  {"enabled": true, "role": "chat"}
  }
}
```

JSONC 編輯會經由 `jsonc-parser`，因此執行 `set`
後仍會保留註解和空白。請先搭配 `--dry-run` 執行，以便在提交前檢查位元組。
`.json` 檔案使用與 `.jsonc` 相同的配接器和編輯路徑。

### JSONL

```text
{"event":"start","userId":"u1","ts":1}
{"event":"action","userId":"u1","ts":2}
{"event":"end","userId":"u1","ts":3}
```

```bash
$ openclaw path find 'oc://session.jsonl/[event=action]/userId' --file session.jsonl --human
oc://session.jsonl/[event=action]/userId 有 1 個相符項目：
  oc://session.jsonl/L2/userId  →  葉節點 @ L2: "u1"（字串）

$ openclaw path resolve 'oc://session.jsonl/L2/ts' --file session.jsonl --human
葉節點 @ L2: "2"（數字）
```

每一行都是一筆記錄。不知道行號時，請透過述詞（`[event=action]`）定址；
知道行號時，則使用標準 `LN` 區段。
`.ndjson` 檔案使用與 `.jsonl` 相同的配接器。

### YAML

```text
# workflow.yaml
name: inbox-triage
steps:
  - id: fetch
    command: gmail.search
  - id: classify
    command: openclaw.invoke
```

```bash
$ openclaw path resolve 'oc://workflow.yaml/steps/0/id' --file workflow.yaml --human
葉節點 @ L3: "fetch"（字串）

$ openclaw path set 'oc://workflow.yaml/steps/$last/id' 'classify-renamed' --file workflow.yaml --dry-run
--dry-run: 將寫入 99 位元組至 /…/workflow.yaml
name: inbox-triage
steps:
  - id: fetch
    command: gmail.search
  - id: classify-renamed
    command: openclaw.invoke
```

YAML 使用 `yaml` 套件的 `Document` API，而不是自行編寫的
剖析器，因此一般的剖析／輸出往返轉換會保留註解和撰寫
形式，而解析後的路徑則使用與 JSONC 相同的對應表鍵／序列索引模型。
相同的配接器會處理 `.yaml`、`.yml` 和 `.lobster` 檔案。

## 子命令參考

### `resolve <oc-path>`

讀取單一葉節點或節點。不接受萬用字元，請改用 `find`。
相符時以 `0` 結束，確定無相符項目時以 `1` 結束，遇到剖析錯誤或遭拒絕的
模式時則以 `2` 結束。

```bash
openclaw path resolve 'oc://AGENTS.md/tools/gh/risk' --human
openclaw path resolve 'oc://gateway.jsonc/server/port' --json
```

### `find <pattern>`

列舉萬用字元／述詞／聯集模式的每個相符項目。至少有一個相符項目時以 `0`
結束，零個時以 `1` 結束。檔案位置萬用字元會以
`OC_PATH_FILE_WILDCARD_UNSUPPORTED` 拒絕，請傳入具體檔案（多檔案
glob 是後續功能）。

```bash
openclaw path find 'oc://AGENTS.md/tools/**/risk'
openclaw path find 'oc://session.jsonl/[event=action]/userId'
openclaw path find 'oc://config.jsonc/plugins/{github,slack}/enabled'
```

### `set <oc-path> <value>`

寫入葉節點。搭配 `--dry-run` 可預覽將寫入的位元組，
而不會變更檔案。加上 `--diff` 可預覽統一差異。
成功寫入時以 `0` 結束；基底拒絕時（例如觸發
哨兵防護）以 `1` 結束；發生剖析錯誤時以 `2` 結束。

```bash
openclaw path set 'oc://gateway.jsonc/version' '2.0' --dry-run
openclaw path set 'oc://gateway.jsonc/version' '2.0' --dry-run --diff
openclaw path set 'oc://gateway.jsonc/version' '2.0'
openclaw path set 'oc://AGENTS.md/Tools/+gh/risk' 'low'
```

如果指定名稱的子節點尚不存在，`+key` 插入標記會建立該節點；
`+nnn` 和單獨的 `+` 則分別用於索引插入和附加插入。

### `validate <oc-path>`

僅進行剖析檢查。不存取檔案系統。適合用於在替換變數前確認
範本路徑格式正確，或在偵錯時查看
結構分解：

```bash
$ openclaw path validate 'oc://AGENTS.md/tools/gh' --human
有效：oc://AGENTS.md/tools/gh
  檔案：    AGENTS.md
  章節： tools
  項目：    gh
```

有效時以 `0` 結束；無效時以 `1` 結束（並附帶結構化的 `code` 和
`message`）；發生引數錯誤時以 `2` 結束。

### `emit <file>`

讓檔案通過各種類型的剖析器和輸出器進行往返轉換。對於格式正確的檔案，輸出應與輸入
逐位元組完全相同；若有差異，表示存在
剖析器錯誤或觸發了哨兵。適合用於使用
真實輸入偵錯基底行為。

```bash
openclaw path emit ./AGENTS.md
openclaw path emit ./gateway.jsonc --json
```

## 結束代碼

| 代碼 | 含義                                                                    |
| ---- | -------------------------------------------------------------------------- |
| `0`  | 成功。（`resolve`／`find`：至少一個相符項目。`set`：寫入成功。） |
| `1`  | 無相符項目，或 `set` 遭基底拒絕（無系統層級錯誤）。      |
| `2`  | 引數或剖析錯誤。                                                   |

## 輸出模式

`openclaw path` 會感知 TTY：在終端機上輸出人類可讀的內容，stdout
透過管線傳送或重新導向時則輸出 JSON。`--json` 和 `--human` 會覆寫
自動偵測。

## 注意事項

- `set` 會透過底層基礎的 emit 路徑寫入位元組，該路徑會自動套用
  遮蔽哨兵防護。若葉節點包含
  `__OPENCLAW_REDACTED__`（完整原文或作為子字串），則會在寫入
  時遭拒絕。
- JSONC 解析與葉節點編輯使用外掛本機的 `jsonc-parser`
  相依套件，因此一般葉節點寫入會保留註解與格式，
  而不會經過自行實作的剖析器／重新呈現路徑。
- `path` 不會感知最後已知良好（LKG）設定的追蹤或復原；
  該生命週期由其他位置負責。如果透過 `path` 編輯的檔案
  同時也受 LKG 追蹤，下一次讀取設定時會決定要提升還是
  復原該檔案；請將 `path` 編輯視為對該檔案的任何其他直接寫入。

## 相關內容

- [命令列介面參考](/zh-TW/cli)
