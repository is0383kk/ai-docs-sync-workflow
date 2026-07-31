---
read_when:
    - 你想要使用 memory-wiki 命令列介面
    - 你正在記錄或變更 `openclaw wiki`
summary: '`openclaw wiki` 的命令列介面參考（memory-wiki 資料庫狀態、搜尋、編譯、檢查、套用、橋接、ChatGPT 匯入及 Obsidian 輔助工具）'
title: 維基百科
x-i18n:
    generated_at: "2026-07-26T08:29:25Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 1f793d52de270068cf3a06b13f52242bb66738235718639486e090a2de213e73
    source_path: cli/wiki.md
    workflow: 16
---

# `openclaw wiki`

檢查並維護 `memory-wiki` 資料庫。由隨附的選用 `memory-wiki` 外掛提供。首次使用前請先啟用：

```bash
openclaw plugins enable memory-wiki
openclaw gateway restart
```

相關內容：[記憶 Wiki 外掛](/zh-TW/plugins/memory-wiki)、[記憶概覽](/zh-TW/concepts/memory)、[命令列介面：memory](/zh-TW/cli/memory)

## 常用命令

```bash
openclaw wiki status
openclaw wiki doctor
openclaw wiki init
openclaw wiki ingest ./notes/alpha.md
openclaw wiki okf import ./knowledge-catalog/okf/bundles/ga4
openclaw wiki compile
openclaw wiki lint
openclaw wiki search "alpha"
openclaw wiki search "who should I ask about Teams?" --mode route-question
openclaw wiki get entity.alpha --from 1 --lines 80

openclaw wiki apply synthesis "Alpha Summary" \
  --body "Short synthesis body" \
  --source-id source.alpha

openclaw wiki apply metadata entity.alpha \
  --source-id source.alpha \
  --status review \
  --question "Still active?"

openclaw wiki bridge import
openclaw wiki unsafe-local import
openclaw wiki chatgpt import --export ./chatgpt-export --dry-run
openclaw wiki chatgpt rollback <run-id>

openclaw wiki obsidian status
openclaw wiki obsidian search "alpha"
openclaw wiki obsidian open syntheses/alpha-summary.md
openclaw wiki obsidian command workspace:quick-switcher
openclaw wiki obsidian daily
```

## Agent 選擇

當 `plugins.entries.memory-wiki.config.vault.scope` 為 `agent` 時，請使用頂層 `--agent <id>` 選項選取資料庫：

```bash
openclaw wiki --agent support status
openclaw wiki --agent support search "refund policy"
openclaw wiki --agent marketing ingest ./campaign-notes.md
```

在設定多個 Agent 的環境中，命令列介面操作必須提供 `--agent`，以免命令讀取或寫入任意的預設資料庫。若僅設定一個 Agent，該 Agent 仍為預設值。未知的 Agent ID 會在資料庫操作開始前失敗。當 `vault.scope` 為 `global` 時，此選項不會變更所選路徑。

閘道用戶端遵循相同規則：在 Agent 範圍的多 Agent 設定中，對以資料庫為基礎的 `wiki.*` 要求傳入 `agentId`。缺少或未知的 ID 會導致錯誤。Agent 回合、Wiki 工具、記憶語料庫補充內容，以及編譯後的提示摘要，均已攜帶作用中的執行階段 Agent 情境。

## 命令

### `wiki status`

顯示資料庫模式與範圍、解析後的 Agent、健康狀態，以及 Obsidian 命令列介面的可用性。請先使用此命令，檢查預期的資料庫是否已初始化、橋接模式是否正常，或 Obsidian 整合是否可用。

當橋接模式處於作用中且設定為讀取記憶成品時，此命令會查詢執行中的閘道，因此能看到與 Agent／執行階段記憶相同的作用中記憶外掛情境。

### `wiki doctor`

執行 Wiki 健康檢查並回報可採取的修正措施。狀態不正常時會以非零狀態結束。

當橋接模式處於作用中且設定為讀取記憶成品時，此命令會先查詢執行中的閘道，再建立報告。停用的橋接匯入，以及未讀取記憶成品的橋接設定，會維持本機／離線運作。

常見問題：

- 已啟用橋接模式，但沒有公開記憶成品
- 資料庫配置無效或遺失
- 預期使用 Obsidian 模式時，缺少外部 Obsidian 命令列介面

### `wiki init`

建立 Wiki 資料庫配置與入門頁面，包括頂層索引與快取目錄。

### `wiki ingest <path>`

將本機 Markdown 或文字檔案匯入 Wiki 的 `sources/` 資料夾，作為來源頁面。`<path>` 必須是本機檔案路徑；目前不支援從 URL 擷取。二進位檔案會遭拒絕。

匯入的來源頁面會帶有來源追溯 frontmatter（`sourceType: local-file`、`sourcePath`、`ingestedAt`）。擷取後一律會重新編譯資料庫。

旗標：`--title <title>` 會覆寫來源標題（預設值：從檔名衍生）。

### `wiki okf import <path>`

將解壓縮的 Open Knowledge Format 套件匯入 Wiki 概念頁面。

匯入工具會讀取 OKF 目錄樹中每個非保留的 `.md` 概念文件、要求 `type` 欄位不得為空，並將未知的 OKF `type` 值視為一般概念。保留的 OKF `index.md` 與 `log.md` 檔案不會匯入為概念。

匯入的頁面會扁平化存放於 `concepts/` 下，讓現有的 Wiki 編譯、搜尋、取得、摘要及儀表板流程能立即存取。原始 OKF 概念 ID、`type`、`resource`、`tags`、時間戳記、來源路徑及完整 frontmatter，都會保留在頁面的 frontmatter 中。內部 OKF Markdown 連結會改寫為產生的 Wiki 頁面；失效或外部連結則維持不變。匯入後一律會重新編譯資料庫。

範例：

```bash
openclaw wiki okf import ./bundles/ga4
openclaw wiki okf import ./bundles/ga4 --json
openclaw wiki search "BigQuery Table" --mode source-evidence --json
openclaw wiki get <path-from-json-result>
```

### `wiki compile`

重建索引、相關內容區塊、儀表板，以及編譯後的查詢／提示快照。快照會保存於 OpenClaw 的共用 SQLite 外掛狀態，並保留在記憶體中以同步投射提示；它不會在資料庫中建立快取檔案。

若已啟用 `render.createDashboards`，編譯也會重新整理報告頁面。

### `wiki lint`

檢查資料庫並寫入涵蓋下列項目的報告：

- 結構問題（失效連結、遺失／重複 ID、缺少頁面類型或標題、無效 frontmatter）
- 來源追溯缺口（缺少來源 ID、缺少匯入來源資訊）
- 矛盾（標記的矛盾、互相衝突的主張）
- 未解問題
- 低信心頁面與主張
- 過時頁面與主張

完成重要的 Wiki 更新後，請執行此命令。

### `wiki search <query>`

搜尋 Wiki 內容。行為取決於設定：

- `search.backend`：`shared` 或 `local`
- `search.corpus`：`wiki`、`memory` 或 `all`
- `--mode`：`auto`、`find-person`、`route-question`、`source-evidence` 或 `raw-claim`

需要 Wiki 專用的排序與來源追溯時，請使用 `wiki search`。若要執行一次廣泛的共用回憶查詢，當作用中的記憶外掛公開共用搜尋時，請優先使用 `openclaw memory search`。

搜尋模式：

- `find-person`：別名、帳號名稱、社群帳號、標準 ID 及人物頁面
- `route-question`：詢問對象／最適用途提示及關係情境
- `source-evidence`：來源頁面與結構化證據欄位
- `raw-claim`：含主張／證據中繼資料的結構化主張文字

範例：

```bash
openclaw wiki search "bgroux" --mode find-person
openclaw wiki search "who knows Teams rollout?" --mode route-question
openclaw wiki search "maintainer-whois" --mode source-evidence
openclaw wiki search "strong route Teams" --mode raw-claim --json
```

當結果符合結構化主張時，文字輸出會包含 `Claim:` 與 `Evidence:` 行。JSON 輸出另會公開 `matchedClaimId`、`matchedClaimStatus`、`matchedClaimConfidence`、`evidenceKinds` 及 `evidenceSourceIds`，供 Agent 端深入探索。

### `wiki get <lookup>`

依 ID 或相對路徑讀取 Wiki 頁面。

```bash
openclaw wiki get entity.alpha
openclaw wiki get syntheses/alpha-summary.md --from 1 --lines 80
```

### `wiki apply`

套用範圍明確的變更，無須任意修改頁面：

- `apply synthesis <title>`：以受管理的摘要內文建立或重新整理綜合頁面
- `apply metadata <lookup>`：更新現有頁面的中繼資料

兩者皆接受 `--source-id`、`--contradiction`、`--question`（各自可重複指定）、`--confidence <n>`（0-1）及 `--status <status>`。`apply metadata` 也接受 `--clear-confidence`，以移除已儲存的信心值。這是演進 Wiki 頁面的支援方式，可確保受管理的產生區塊保持完整。

### `wiki bridge import`

將作用中記憶外掛的公開記憶成品匯入橋接後端的來源頁面。請在 `bridge` 模式中使用此命令，將最新匯出的記憶成品提取至 Wiki 資料庫。

讀取作用中的橋接成品時，命令列介面會透過閘道 RPC 路由匯入，因此會使用執行階段記憶外掛情境。若已停用橋接匯入或成品讀取，命令會維持本機／離線的零匯入行為。匯入後是否重新整理索引由 `ingest.autoCompile` 控制。

### `wiki unsafe-local import`

在 `unsafe-local` 模式中，從明確設定的本機路徑（`unsafeLocal.paths`）匯入。此功能刻意設為實驗性質，且僅限同一台電腦。匯入後是否重新整理索引由 `ingest.autoCompile` 控制。

### `wiki chatgpt import`

將 ChatGPT 匯出內容匯入為 Wiki 來源頁面草稿。

```bash
openclaw wiki chatgpt import --export ./chatgpt-export
openclaw wiki chatgpt import --export ./conversations.json --dry-run
```

| 旗標              | 預設值    | 說明                                                   |
| ----------------- | ---------- | ------------------------------------------------------------- |
| `--export <path>` | （必填） | ChatGPT 匯出目錄或 `conversations.json` 路徑。        |
| `--dry-run`       | `false`    | 在不寫入頁面的情況下預覽建立／更新／略過的數量。 |

非試執行的匯入只要變更任何頁面，就會記錄匯入執行 ID 並顯示於摘要中；回復操作需要此 ID。

### `wiki chatgpt rollback <run-id>`

回復先前套用的 ChatGPT 匯入執行，移除其建立的頁面，並還原其覆寫的頁面。若該執行已經回復，則不執行任何操作（並回報 `alreadyRolledBack`）。

### `wiki obsidian ...`

適用於以 Obsidian 友善模式運作之資料庫的 Obsidian 輔助命令：`status`、`search`、`open`、`command`、`daily`。啟用 `obsidian.useOfficialCli` 時，這些命令需要 `PATH` 上的官方 `obsidian` 命令列介面。

當 `vault.scope` 為 `agent` 時，設定驗證會拒絕 `obsidian.useOfficialCli: true`，因為 `obsidian.vaultName` 是一項全域設定，而非各 Agent 的對應設定。仍可使用 Obsidian 友善的 Markdown 呈現方式。

## 實務使用指南

- 當來源追溯與頁面身分很重要時，請使用 `wiki search` + `wiki get`。
- 請使用 `wiki apply`，而非手動編輯受管理的產生區段。
- 在信任互相矛盾或低信心的內容之前，請使用 `wiki lint`。
- 批次匯入或變更來源後，若要立即取得最新的儀表板與編譯摘要，請使用 `wiki compile`。
- 當資料目錄、文件匯出或 Agent 強化流水線已產生 OKF Markdown 套件時，請使用 `wiki okf import`。
- 當橋接模式仰賴新匯出的記憶成品時，請使用 `wiki bridge import`。

## 設定關聯

`openclaw wiki` 的行為受下列項目影響：

- `plugins.entries.memory-wiki.config.vaultMode`
- `plugins.entries.memory-wiki.config.vault.scope`
- `plugins.entries.memory-wiki.config.vault.path`
- `plugins.entries.memory-wiki.config.search.backend`
- `plugins.entries.memory-wiki.config.search.corpus`
- `plugins.entries.memory-wiki.config.bridge.*`
- `plugins.entries.memory-wiki.config.obsidian.*`
- `plugins.entries.memory-wiki.config.ingest.autoCompile`
- `plugins.entries.memory-wiki.config.render.*`
- `plugins.entries.memory-wiki.config.context.includeCompiledDigestPrompt`

完整設定模型請參閱[記憶 Wiki 外掛](/zh-TW/plugins/memory-wiki)。

## 相關內容

- [命令列介面參考](/zh-TW/cli)
- [記憶 Wiki](/zh-TW/plugins/memory-wiki)
