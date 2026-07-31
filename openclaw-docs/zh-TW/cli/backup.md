---
read_when:
    - 你想要為本機 OpenClaw 狀態建立一流的備份封存檔
    - 你需要一份精簡且經過驗證的單一 OpenClaw SQLite 資料庫快照
    - 你想在重設或解除安裝前預覽將包含哪些路徑
summary: '`openclaw backup` 的命令列介面參考（封存檔與 SQLite 快照）'
title: 備份
x-i18n:
    generated_at: "2026-07-26T08:12:23Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: dfb5a118545589b181cede26dab72e9d029d98a1cac5cfccedd9d9cf2c56d3b5
    source_path: cli/backup.md
    workflow: 16
---

# `openclaw backup`

為 OpenClaw 狀態、設定、驗證設定檔、頻道／提供者認證資訊、工作階段，以及選用的工作區建立本機備份封存檔。

```bash
openclaw backup create
openclaw backup create --output ~/Backups
openclaw backup create --dry-run --json
openclaw backup create --verify
openclaw backup create --no-include-workspace
openclaw backup create --only-config
openclaw backup verify ./2026-03-09T08-00-00.000+08-00-openclaw-backup.tar.gz
openclaw backup sqlite create --global --repository ~/Backups/openclaw-sqlite
openclaw backup sqlite create --agent main --repository ~/Backups/openclaw-sqlite
openclaw backup sqlite list --repository ~/Backups/openclaw-sqlite
openclaw backup sqlite verify ~/Backups/openclaw-sqlite/<snapshot-id>
openclaw backup sqlite verify ~/Backups/openclaw-sqlite/<snapshot-id> --scratch ~/Private/openclaw-scratch
openclaw backup sqlite restore ~/Backups/openclaw-sqlite/<snapshot-id> --target ./restored/openclaw.sqlite
```

## 注意事項

- 封存檔內嵌一份 `manifest.json`，其中包含解析後的來源路徑與封存檔配置。
- 預設輸出是在目前工作目錄中建立帶有時間戳記的 `.tar.gz` 封存檔。帶有時間戳記的檔名會使用你機器的本機時區，並包含 UTC 偏移量。如果目前工作目錄位於已備份的來源樹狀結構內，OpenClaw 會改用你的家目錄作為預設封存檔位置。
- 絕不覆寫現有封存檔。為避免將封存檔本身納入，會拒絕位於來源狀態／工作區樹狀結構內的輸出路徑。
- `openclaw backup verify <archive>` 會檢查封存檔是否只包含一份根資訊清單、拒絕路徑遍歷形式的封存路徑與 SQLite 附屬檔案、確認資訊清單宣告的每個承載內容都存在、驗證每個 SQLite 快照的檔案格式，並對標準 OpenClaw 資料庫執行完整的完整性與角色檢查。專用外掛結構描述會維持不透明，因為它們可能需要擁有者定義的 SQLite 功能。`openclaw backup create --verify` 會在寫入封存檔後立即執行該驗證。
- `openclaw backup create --only-config` 只會備份使用中的 JSON 設定檔。

## SQLite 快照

當你需要的是單一 OpenClaw 所擁有 SQLite 資料庫的可攜式成品，而非廣泛的狀態封存檔時，請使用 `openclaw backup sqlite`。

建立快照時，只接受一個具名來源：

| 命令                                                         | 資料庫               |
| --------------------------------------------------------------- | ---------------------- |
| `openclaw backup sqlite create --global --repository <dir>`     | 共用 OpenClaw 狀態  |
| `openclaw backup sqlite create --agent <id> --repository <dir>` | 每個代理程式各一個資料庫 |

儲存庫會為每個已提交的快照包含一個目錄。每個快照目錄僅包含：

- `manifest.json`
- `database.sqlite`

建立快照時，會先驗證使用中的資料庫，再進行讀取；使用 SQLite 的線上備份 API 擷取已提交的 WAL 狀態，而不會長時間持有單一讀取交易；接著關閉使用中的資料庫、使用 `VACUUM` 壓縮私有副本、再次驗證產生的資料庫，並在不覆寫現有路徑的情況下發布完成的目錄。全域快照會在壓縮前移除暫時性的傳遞佇列資料列，避免已刪除的佇列承載內容保留在可用頁面中。

請勿將使用中的 `.sqlite`、`-wal`、`-shm` 或 `-journal` 檔案複製為可攜式成品。只複製已完成的快照目錄。

SQLite 快照可能包含驗證設定檔、工作階段狀態、外掛狀態及其他敏感記錄。請使用與 OpenClaw 使用中狀態目錄相同的權限、加密、保留原則及目的地限制來保護儲存庫。

### 驗證與還原

```bash
openclaw backup sqlite verify <snapshot-directory>
openclaw backup sqlite restore <snapshot-directory> --target <new-database-path>
```

驗證會檢查嚴格的資訊清單格式、成品大小與 SHA-256、SQLite 完整性、外鍵、結構描述版本、資料庫角色與擁有者，以及 OpenClaw 所擁有的索引定義。

驗證會檢查內容固定的私有副本，因此路徑名稱競爭無法調換 SQLite 所檢查的位元組。預設情況下，該暫存副本會建立在快照儲存庫旁，並在命令返回前移除。暫存根目錄及其祖先鏈必須防止其他使用者將其替換。POSIX 根目錄必須由目前使用者擁有，且群組／所有人皆不可寫入；對於使用者擁有的子項目，會接受如 `/tmp` 之類設有黏著位元的祖先目錄。會拒絕導致暫存區暴露或可被替換的 macOS ACL 授權。Windows 根目錄與祖先目錄必須由目前使用者或受信任的作業系統主體擁有，且 ACL 必須拒絕不受信任的暫存區存取。若是唯讀掛載點或網路共用，請在具有同等加密與目的地控制措施的儲存空間上傳入 `--scratch <existing-private-directory>`。

建立快照時，會在暫存或發布資料庫位元組前，對儲存庫套用相同的擁有者、ACL、祖先目錄及路徑身分檢查。

還原會再次執行驗證，且只寫入全新的目標。它會拒絕現有目標、`-wal`、`-shm` 或 `-journal` 附屬檔案，且絕不就地替換使用中的 OpenClaw 資料庫。目標的父目錄必須符合與驗證暫存空間相同的路徑安全性要求。啟用還原後的資料庫仍須由操作人員明確執行離線步驟。

快照儲存庫是本機目錄。排程、上傳、保留、增量 WAL 組合包、容錯移轉及開機時還原行為均刻意不包含在此命令的範圍內。

## 備份內容

`openclaw backup create` 會根據你的本機 OpenClaw 安裝規劃來源：

- 狀態目錄（通常是 `~/.openclaw`）
- 使用中設定檔的路徑
- 解析後的 `credentials/` 目錄（當其存在於狀態目錄之外時）
- 從目前設定中探索到的工作區目錄，除非你傳入 `--no-include-workspace`

驗證設定檔及其他個別代理程式的執行階段狀態位於狀態目錄下的 SQLite 中（`agents/<agentId>/agent/openclaw-agent.sqlite`），因此會自動納入狀態備份項目。

`--only-config` 會略過狀態、認證資訊目錄與工作區探索，並只封存使用中設定檔的路徑。

OpenClaw 會先將路徑標準化，再建立封存檔：如果設定檔、認證資訊目錄或工作區已位於狀態目錄內，就不會將其重複納入為獨立的頂層備份來源。缺少的路徑會略過。

建立封存檔期間，OpenClaw 會在 `tar` 讀取已知會即時變動的路徑前將其排除。這可避免檔案記錄的大小與並行寫入之間發生競爭。此篩選器會在每個已備份的狀態目錄下套用下列相對於狀態目錄的規則：

| 相對於狀態目錄的範圍                         | 略過的檔案副檔名         |
| -------------------------------------------- | ----------------------------- |
| `sessions/**`                                | `.jsonl`、`.log`              |
| `agents/<agentId>/sessions/**`               | `.jsonl`、`.log`              |
| `cron/runs/**`                               | `.jsonl`、`.log`              |
| `logs/**`                                    | `.jsonl`、`.log`              |
| `delivery-queue/**`                          | `.json`、`.delivered`、`.tmp` |
| `session-delivery-queue/**`                  | `.json`、`.delivered`、`.tmp` |
| 已備份狀態目錄下的任何路徑 | `.sock`、`.pid`、`.tmp`       |

這些規則不會篩除狀態目錄外的工作區檔案。它們也會略過符合表格條件的已完成文字記錄與記錄檔，因此需要時請另行保留這些記錄。JSON 結果中的 `skippedVolatileCount` 會回報刻意略過的檔案數量。

狀態目錄下的 SQLite 資料庫會使用 SQLite 的線上備份 API 擷取，並透過 `VACUUM` 離線壓縮，避免已刪除頁面的殘留內容進入封存檔，且不會複製使用中的 WAL／SHM 檔案。需要目前無法使用之擁有者定義 SQLite 功能的外掛自有資料庫，會採取封閉式失敗，而不會退回直接複製檔案。透過工作區備份納入的 SQLite 檔案會以工作區檔案形式複製，且不在壓縮保證範圍內。

狀態目錄的 `extensions/` 樹狀結構下，已安裝外掛的原始碼與資訊清單檔案會納入備份，但其中巢狀的 `node_modules/` 相依性樹狀結構會略過，因為它們是可重新建置的安裝成品。還原封存檔後，若還原的外掛回報缺少相依性，請使用 `openclaw plugins update <id>`，或透過 `openclaw plugins install <spec> --force` 重新安裝。

狀態目錄下由安裝程式管理且可重新建置的執行階段根目錄也會略過：`dev/`、`git/`、`npm/`、舊版 `npm-runtime/`，以及 `tools/`。這些目錄包含受管理的簽出內容、套件樹狀結構及已下載的執行階段，而不是權威性的使用者狀態；還原後請重新安裝或更新對應的執行階段或外掛。位於這些根目錄之一內、經明確設定的設定檔、認證資訊目錄或工作區仍會納入。

## 無效設定行為

`openclaw backup` 會略過一般設定預先檢查，因此在復原期間仍可提供協助。工作區探索依賴有效的設定，因此當設定檔存在但無效，且工作區備份仍啟用時，`openclaw backup create` 會快速失敗。

若要在這種情況下進行部分備份，請搭配 `--no-include-workspace` 重新執行：它會保留狀態、設定及外部認證資訊目錄，同時完全略過工作區探索。

設定格式錯誤時，`--only-config` 也能運作，因為它不會剖析設定以探索工作區。

## 大小與效能

OpenClaw 不會強制執行內建的最大備份大小或單一檔案大小限制。如果封存檔寫入作業持續五分鐘未產生任何資料，就會失敗並移除其未完成的暫存檔，而不是無限期停滯。其他實際限制取決於：

- 暫存封存檔寫入及最終封存檔所需的可用空間
- 走訪大型工作區樹狀結構並將其壓縮至 `.tar.gz` 所需的時間
- 使用 `--verify` 或 `openclaw backup verify` 重新掃描封存檔所需的時間
- 目的地檔案系統行為：OpenClaw 要求以不覆寫的硬連結發布，確保最終封存檔路徑絕不會暴露尚未完成的副本；不支援的檔案系統會失敗並提供可採取行動的錯誤訊息

若發布後無法確認最終目錄的耐久性，命令會回報失敗，但會保留完整的最終項目，以免冒險刪除並行作業建立的替代項目。

大型工作區通常是影響封存檔大小的主要因素。請使用 `--no-include-workspace` 進行更小、更快的備份，或使用 `--only-config` 建立最小的封存檔。

## 相關內容

- [命令列介面參考](/zh-TW/cli)
