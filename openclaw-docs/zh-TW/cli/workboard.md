---
read_when:
    - 你想要從終端機檢視或建立 Workboard 卡片
    - 你想要從命令列介面分派 Workboard 工作節點執行作業
    - 你正在偵錯 Workboard 命令列介面或斜線命令的行為
summary: '`openclaw workboard` 卡片、分派與工作執行的命令列介面參考資料'
title: 工作看板命令列介面
x-i18n:
    generated_at: "2026-07-26T07:16:17Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 640260ea6f5959b3aee1cdce76f2501097bff79e9bf1741bdd9ff7a8b43e1a7f
    source_path: cli/workboard.md
    workflow: 16
---

`openclaw workboard` 是隨附 [Workboard 外掛](/zh-TW/plugins/workboard)的終端操作介面。操作員可用它列出卡片、建立卡片、檢視單一卡片，並要求執行中的閘道將就緒工作分派為子代理工作執行。

使用此命令前，請先啟用外掛：

```bash
openclaw plugins enable workboard
openclaw gateway restart
```

## 使用方式

```bash
openclaw workboard list [--board <id>] [--status <status>] [--include-archived] [--json]
openclaw workboard create <title...> [--notes <text>] [--status <status>] [--priority <priority>] [--agent <id>] [--board <id>] [--labels <items>] [--json]
openclaw workboard show <id> [--json]
openclaw workboard move <id> --status <status> [--json]
openclaw workboard dispatch [--board <id>] [--max-starts <count>] [--admin] [--url <url>] [--token <token>] [--timeout <ms>] [--json]
```

此命令會讀寫儀表板與 Workboard 代理工具所使用的同一個外掛自有 SQLite 資料庫。卡片 ID 是 UUID；接受卡片 ID 的命令也接受不具歧義的 ID 前綴（精簡文字輸出會顯示前 8 個字元）。

有效的 `status` 值：`triage`、`backlog`、`todo`、`scheduled`、`ready`、`running`、`review`、`blocked`、`done`。有效的 `priority` 值：`low`、`normal`、`high`、`urgent`。

## `list`

```bash
openclaw workboard list
openclaw workboard list --board default --status ready
openclaw workboard list --json
```

文字輸出採用精簡格式：

```text
7f4a2c10  ready     high    default agent-a  修正過期的工作程式心跳偵測
```

各欄依序為 ID 前綴、狀態、優先順序、看板 ID、選用的代理 ID，以及標題。

| 旗標                 | 用途                                       |
| -------------------- | --------------------------------------------- |
| `--board <id>`       | 將結果限制於單一看板命名空間          |
| `--status <status>`  | 將結果限制於單一 Workboard 狀態         |
| `--include-archived` | 在精簡文字輸出中包含已封存的卡片 |
| `--json`             | 以機器可讀 JSON 輸出完整卡片清單      |

精簡文字輸出預設會隱藏已封存的卡片，使命令列介面與 `/workboard list` 保持一致。傳入 `--include-archived` 即可顯示這些卡片。為支援現有自動化，JSON 輸出一律保留包含已封存卡片在內的完整卡片清單。

## `create`

```bash
openclaw workboard create "修正過期的工作程式心跳偵測" --priority high --labels bug,workboard
openclaw workboard create "撰寫 Workboard 文件" --status ready --agent docs-agent --board docs --notes "涵蓋命令列介面、斜線命令、分派與 SQLite 狀態。"
```

| 旗標                    | 用途                                 |
| ----------------------- | --------------------------------------- |
| `--notes <text>`        | 卡片的初始備註                      |
| `--status <status>`     | 初始狀態，預設為 `todo`          |
| `--priority <priority>` | 優先順序，預設為 `normal`              |
| `--agent <id>`          | 將卡片指派給代理或擁有者 ID |
| `--board <id>`          | 將卡片儲存於看板命名空間     |
| `--labels <items>`      | 以逗號分隔的標籤                  |
| `--json`                | 以機器可讀 JSON 輸出已建立的卡片  |

`create` 會直接寫入 Workboard SQLite 狀態。卡片會立即顯示於控制介面的 Workboard 分頁中，Workboard 工具也能立即存取。

## `show`

```bash
openclaw workboard show 7f4a2c10
openclaw workboard show 7f4a2c10 --json
```

文字輸出會顯示精簡卡片列與備註。JSON 輸出會傳回完整卡片記錄，包括執行中繼資料、嘗試次數、留言、連結、證明、成品、工作程式記錄、通訊協定狀態、診斷資訊與自動化中繼資料。

JSON 中的證明狀態是由工作程式回報的結果。`passed` 會記錄工作程式對所附命令或檢查的
自我評估；這不是獨立驗證的
結果。

## `move`

```bash
openclaw workboard move 7f4a2c10 --status review
openclaw workboard move 7f4a2c10 --status done --json
```

`move` 會使用與在儀表板中拖曳卡片相同的手動操作員路徑來變更卡片狀態。它接受完整卡片 ID 或不具歧義的前綴。作用中的相依性與排程暫停仍然適用。操作員可以移動已被領取的卡片，無須提供其代理領取權杖；領取權杖仍僅適用於代理工具的變更操作，且會從 JSON 輸出中遮蔽。

## `dispatch`

```bash
openclaw workboard dispatch
openclaw workboard dispatch --json
openclaw workboard dispatch --max-starts 10
openclaw workboard dispatch --admin
openclaw workboard dispatch --url http://127.0.0.1:18789 --token "$OPENCLAW_GATEWAY_TOKEN"
```

`dispatch` 會先呼叫執行中閘道的 RPC 方法 `workboard.cards.dispatch`。此方法使用與儀表板分派動作相同的子代理執行環境，因此就緒卡片會成為受任務追蹤的工作程式執行，並連結工作階段金鑰。`--max-starts` 使用附加的 `workboard.cards.dispatchWithOptions` 方法，因此較舊的閘道會在啟動任何工作程式前拒絕此選項；升級後，使用此旗標前請重新啟動閘道。已指派代理的卡片會使用代理範圍的子代理工作階段金鑰；未指派的卡片則保留不限定範圍的子代理金鑰，以維持閘道所設定的預設代理。

分派迴圈會：

1. 將相依性已就緒的子卡片提升為 `ready`。
2. 封鎖已過期的領取或逾時的工作程式執行。
3. 在就緒卡片上記錄分派中繼資料。
4. 選取一小批尚未被領取的就緒卡片。
5. 由分派程式或獲指派的代理領取每張所選卡片。
6. 使用受限的卡片內容與卡片領取權杖啟動子代理工作程式執行。
7. 將工作程式執行 ID、工作階段金鑰、閘道任務帳本回報的任務連結、執行狀態，以及工作程式記錄儲存於卡片上。

選取方式較為保守：單次分派預設最多啟動三個工作程式、略過已封存或已被領取的卡片，並且每次只為每位擁有者或代理啟動一張卡片。若某位擁有者已有作用中的執行中或審查中工作，其卡片會留待後續分派。傳入帶有正整數的 `--max-starts <count>` 可變更每次分派的上限；每位擁有者一張卡片的規則仍然適用，因此實際啟動數量可能較少。

如果卡片被領取後無法啟動工作程式，Workboard 會封鎖該卡片、清除領取狀態，並在卡片執行資訊與工作程式記錄中繼資料內記錄失敗，使啟動失敗保持可見，而不是默默將卡片退回佇列。

若未明確指定閘道目標，且本機閘道無法使用或尚未公開 Workboard 分派方法，命令列介面會改用本機 Workboard 狀態進行僅資料分派。僅資料分派仍可提升相依項目、清理過期領取狀態，以及封鎖逾時執行，但不會啟動工作程式。驗證、權限與資料驗證失敗，以及明確指定 `--url` 或 `--token` 目標時發生的失敗，都會直接回報，而不會觸發後援機制。

文字輸出會回報工作程式啟動結果：

```text
分派完成：已啟動=2 失敗=0
```

後援輸出會明確標示：

```text
閘道無法使用；僅執行資料分派：已提升=1 已封鎖=0
```

JSON 輸出包含分派結果。由閘道支援的分派可能包含 `started` 與 `startFailures`；僅資料後援則包含 `gatewayUnavailable: true`。卡片 JSON 輸出會遮蔽領取權杖。

在儀表板中，相同的分派結果會顯示為簡短摘要，讓操作員無須開啟卡片詳細資料，即可查看已啟動、提升、封鎖、重新領取或失敗的卡片數量。

## 斜線命令功能一致性

支援命令的頻道可使用對應的斜線命令：

```text
/workboard list
/workboard show 7f4a2c10
/workboard create 修正過期的工作程式心跳偵測
/workboard move 7f4a2c10 --status review
/workboard dispatch
```

斜線命令分派也會使用閘道的子代理執行環境，因此其領取、工作程式啟動與失敗行為，會與儀表板及命令列介面的閘道路徑相同。

`/workboard list` 與 `/workboard show` 是供獲授權命令傳送者使用的讀取命令。`/workboard create`、`/workboard move` 與 `/workboard dispatch` 會變更看板狀態，且在聊天介面上需要擁有者身分，或需要具備 `operator.write` 或 `operator.admin` 的閘道用戶端。

## 權限

命令列介面分派路徑通常會要求閘道的 `operator.write` 與 `operator.read` 範圍。繫結至工作區的卡片會直接在完全相符的已設定代理工作區中執行；工作樹要求會縮限至該目錄，而不允許主機具現化由儲存庫控制的程式碼。所選工作程式必須具備對該確切工作區的可寫入、非共用 Docker 沙箱存取權；必須有與所要求掛載和原則相符的作用中容器雜湊；且不得具備主機逃逸能力。傳入 `--admin` 可明確要求 `operator.admin`、允許使用另一個主機簽出，並採用一般受管理的工作樹設定；若用戶端未獲核准該範圍，連線就會失敗。唯讀閘道權杖可透過讀取方法檢視 Workboard 資料，但無法建立卡片或分派工作程式。對於具備 Workboard 變更權限的呼叫者，工作區限制不會以其他方式影響手動移動卡片。

本機 `list`、`create`、`show` 與 `move` 命令會操作目前設定檔所使用的本機 OpenClaw 狀態目錄。需要不同的狀態根目錄時，請在最上層 `openclaw` 命令上使用 `--dev` 或 `--profile <name>`。

## 疑難排解

### 未顯示任何卡片

確認已針對相同設定檔與狀態根目錄啟用此外掛：

```bash
openclaw plugins inspect workboard --runtime --json
```

如果儀表板顯示卡片，但命令列介面沒有顯示，請檢查兩個命令是否使用相同的 `--dev` 或 `--profile` 設定。

### 分派顯示僅資料模式

啟動或重新啟動閘道：

```bash
openclaw gateway restart
openclaw gateway status --deep
```

接著重試 `openclaw workboard dispatch`。僅資料後援適合清理本機狀態，但工作程式執行需要作用中的閘道。

### 分派未啟動任何項目

檢查是否至少有一張沒有作用中領取狀態的 `ready` 卡片：

```bash
openclaw workboard list --status ready
```

如果同一位擁有者已有執行中或審查中的工作，也可能略過其他卡片。請將已完成的工作移至 `done`、透過 Workboard 工具解除過期領取狀態，或在作用中的工作程式完成後再次執行分派。

## 相關內容

- [Workboard 外掛](/zh-TW/plugins/workboard)
- [命令列介面參考](/zh-TW/cli)
- [斜線命令](/zh-TW/tools/slash-commands)
- [控制介面](/zh-TW/web/control-ui)
