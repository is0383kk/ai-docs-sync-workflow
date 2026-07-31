---
read_when:
    - 設定執行核准或允許清單
    - 在 macOS 應用程式中實作 exec 核准使用者體驗
    - 審查沙箱逃逸提示及其影響
sidebarTitle: Exec approvals
summary: 主機執行核准：原則設定、允許清單，以及 YOLO／嚴格工作流程
title: 執行核准
x-i18n:
    generated_at: "2026-07-26T07:58:58Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 2bd09746375061232e9094b8803d33859cac4c13c7bde14a059b7d52e48b5de8
    source_path: tools/exec-approvals.md
    workflow: 16
---

Exec 核准是**輔助應用程式／節點主機的防護機制**，用於允許
沙箱化代理程式在實際主機（`gateway` 或 `node`）上執行命令。只有當政策、允許清單與（選用的）使用者核准全都同意時，命令
才會執行。
核准機制疊加在工具政策與提高權限閘控**之上**（提高權限
`full` 會略過這些機制）。

如需以模式為主的 `deny`、`allowlist`、`ask`、`auto`、`full`、
Codex Guardian 對應，以及 ACPX 控制框架權限概覽，請參閱
[權限模式](/zh-TW/tools/permission-modes)。

<Note>
有效政策是 `tools.exec.*` 與核准預設值之中**較嚴格**的一方：核准只能收緊由設定衍生的安全性／詢問政策，絕不能
放寬。如果省略核准欄位，會使用 `tools.exec` 值。
主機 Exec 也會使用該機器上的本機核准狀態——即使工作階段或設定預設值要求 `ask: "on-miss"`，執行主機核准檔案中的主機本機
`ask: "always"` 仍會持續提示。
</Note>

## 適用範圍

Exec 核准會在執行主機上於本機強制執行：

- **閘道主機** -> 閘道機器上的 `openclaw` 程序。
- **節點主機** -> 節點執行器（macOS 輔助應用程式或無介面節點主機）。

### 信任模型

- 經過閘道驗證的呼叫者，是該閘道受信任的操作者。
- 已配對節點會將該受信任操作者的能力延伸至節點主機。
- 核准可降低意外執行的風險，但**不是**個別使用者的驗證邊界或檔案系統唯讀政策。
- 命令一經核准，即可依照所選主機或沙箱檔案系統權限修改檔案。
- 已核准的節點主機執行會繫結標準執行環境：cwd、確切 argv、存在時的 env 繫結，以及適用時鎖定的可執行檔路徑。
- 對於 Shell 指令碼及直接以直譯器／執行階段叫用檔案的情況，OpenClaw 也會嘗試繫結一個明確的本機檔案運算元。如果該檔案在核准後、執行前發生變更，系統會拒絕執行，而非執行已偏移的內容。
- 檔案繫結採盡力而為，並非涵蓋所有直譯器／執行階段載入路徑的完整模型。如果無法識別恰好一個明確的本機檔案，OpenClaw 會拒絕建立由核准支援的執行，而不會假裝已完整涵蓋。

### macOS 職責劃分

- **節點主機服務**會透過本機 IPC，將 `system.run` 轉送至 **macOS 應用程式**。
- **macOS 應用程式**會強制執行核准，並在 UI 環境中執行命令。

## 檢查有效政策

| 命令                                                          | 顯示內容                                                                          |
| ---------------------------------------------------------------- | -------------------------------------------------------------------------------------- |
| `openclaw approvals get` / `--gateway` / `--node <id\|name\|ip>` | 要求的政策、主機政策來源，以及有效結果。                       |
| `openclaw exec-policy show`                                      | 本機的合併檢視。                                                             |
| `openclaw exec-policy set` / `preset`                            | 透過單一步驟，將本機要求的政策與本機主機核准檔案同步。 |

<Note>
不包含各工作階段的 `/exec` 覆寫。請在相關工作階段中執行 `/exec`，以檢查其目前的預設值。請參閱[工作階段覆寫](/zh-TW/tools/exec#session-overrides-exec)。
</Note>

完整的命令列介面參考（旗標、JSON 輸出、允許清單新增／移除）：[核准命令列介面](/zh-TW/cli/approvals)。

當本機範圍要求 `host=node` 時，`exec-policy show` 會在執行階段將該
範圍回報為由節點管理，而不會將本機核准
檔案視為事實來源。

如果輔助應用程式 UI **無法使用**，任何通常會
提示的要求，都會由**詢問備援**處理（預設：`deny`）。

<Tip>
原生聊天核准用戶端可在待處理核准訊息中預先提供各頻道專屬的操作方式。Matrix 會提供回應捷徑（`✅` 允許一次、
`♾️` 永遠允許、`❌` 拒絕），同時仍在
訊息中保留 `/approve ...` 作為備援。
</Tip>

## 設定與儲存空間

核准資訊存放於執行主機上的本機 JSON 檔案。設定
`OPENCLAW_STATE_DIR` 時，檔案會跟隨該狀態目錄；
否則會使用預設的 OpenClaw 狀態目錄：

```text
$OPENCLAW_STATE_DIR/exec-approvals.json
# 否則
~/.openclaw/exec-approvals.json
```

預設核准 Socket 會使用相同的根目錄：
`$OPENCLAW_STATE_DIR/exec-approvals.sock`；若未設定變數，則為
`~/.openclaw/exec-approvals.sock`。

各狀態目錄是彼此獨立的信任範圍。當 `OPENCLAW_STATE_DIR`
指向其他位置時，OpenClaw 絕不會匯入或封存
`~/.openclaw/exec-approvals.json`；請為自訂狀態目錄
另行設定核准。Doctor 也只會在舊版
`plugin-binding-approvals.json` 屬於作用中狀態目錄時匯入該檔案。

結構描述範例：

```json
{
  "version": 1,
  "socket": {
    "path": "~/.openclaw/exec-approvals.sock",
    "token": "base64url-token"
  },
  "defaults": {
    "security": "deny",
    "ask": "on-miss",
    "askFallback": "deny",
    "autoAllowSkills": false
  },
  "agents": {
    "main": {
      "security": "allowlist",
      "ask": "on-miss",
      "askFallback": "deny",
      "autoAllowSkills": true,
      "allowlist": [
        {
          "id": "B0C8C0B3-2C2D-4F8A-9A3C-5A4B3C2D1E0F",
          "pattern": "~/Projects/**/bin/rg",
          "argPattern": "sha256:argv:...",
          "source": "allow-always",
          "lastUsedAt": 1737150000000,
          "lastResolvedPath": "/Users/user/Projects/.../bin/rg"
        },
        {
          "pattern": "~/Projects/**/bin/git"
        }
      ]
    }
  }
}
```

## 政策控制項

### `tools.exec.mode`

`tools.exec.mode` 是主機 Exec 的建議正規化政策介面：

| 值       | 行為                                                                                                                                                                  |
| ----------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `deny`      | 封鎖主機 Exec。                                                                                                                                                          |
| `allowlist` | 僅執行允許清單中的命令，不進行詢問。                                                                                                                             |
| `ask`       | 使用允許清單政策，並在未命中時詢問。                                                                                                                                   |
| `auto`      | 使用允許清單政策，直接執行確定性比對，並先將核准未命中項目交由 OpenClaw 的原生自動審查器處理，再備援至人工核准途徑。 |
| `full`      | 執行主機 Exec，不顯示核准提示。                                                                                                                                   |

Doctor 會將已淘汰的持續保存 `tools.exec.security` / `tools.exec.ask`
配對遷移至 `tools.exec.mode`。

### `exec.security`

<ParamField path="security" type='"deny" | "allowlist" | "full"'>
  - `deny` - 封鎖所有主機 Exec 要求。
  - `allowlist` - 僅允許允許清單中的命令。
  - `full` - 全部允許（等同於提高權限）。

閘道／節點主機的預設值為 `full`；`sandbox` 主機的預設值則為
`deny`。
</ParamField>

### `exec.ask`

<ParamField path="ask" type='"off" | "on-miss" | "always"'>
  為主機 Exec 設定的詢問政策。控制來自 `tools.exec.ask` 與主機核准預設值的基準核准
  提示行為。
  預設值為 `off`。每次呼叫的 `ask` 工具參數（請參閱
  [Exec 工具](/zh-TW/tools/exec#parameters)）只能加強該基準；當有效的主機詢問政策為 `off` 時，
  來自頻道的模型呼叫會忽略此參數。

- `off` - 永不提示。
- `on-miss` - 僅在允許清單未比對成功時提示。
- `always` - 每個命令都提示。當有效詢問模式為 `always` 時，`allow-always` 永久信任**不會**抑制提示。

</ParamField>

### `askFallback`

<ParamField path="askFallback" type='"deny" | "allowlist" | "full"'>
  需要提示但無法連線至任何 UI（或提示逾時）時的處理方式。省略時預設為 `deny`。

- `deny` - 封鎖。
- `allowlist` - 僅在符合允許清單時允許。
- `full` - 允許。

</ParamField>

### `tools.exec.strictInlineEval`

<ParamField path="strictInlineEval" type="boolean">
  當 `true` 時，即使直譯器二進位檔本身位於允許清單中，也會將行內程式碼求值形式視為僅能透過核准執行。這可為無法明確對應至單一穩定檔案運算元的
  直譯器載入器提供縱深防禦。
</ParamField>

嚴格模式會攔截的範例：`python -c`、`node -e`/`--eval`/`-p`、
`ruby -e`、`perl -e`/`-E`、`php -r`、`lua -e`、`osascript -e`（亦包括 `awk`、
`sed`、`make`、`find -exec` 與 `xargs` 的行內形式）。

在嚴格模式下，這些命令需要審查器或明確核准。使用
`tools.exec.mode: "auto"` 時，如果命令具備可強制執行的計畫，審查器可授予一次低風險執行；否則 OpenClaw 會詢問人工操作者。
到達審查器備援的 `Codex app-server` 命令核准會詢問
人工操作者，因為其核准要求不會公開可強制執行且已解析的
可執行檔。
`allow-always` 不會為行內求值命令永久保存新的允許清單項目。

### `tools.exec.commandHighlighting`

<ParamField path="commandHighlighting" type="boolean" default="false">
  僅影響呈現：啟用後，OpenClaw 可附加由剖析器衍生的
  命令範圍，讓 Web 核准提示能醒目標示命令詞元。這
  **不會**變更 `security`、`ask`、允許清單比對、嚴格行內求值
  行為、核准轉送或命令執行。
</ParamField>

可在 `tools.exec.commandHighlighting` 下進行全域設定，或在
`agents.entries.*.tools.exec.commandHighlighting` 下為各代理程式設定。

## YOLO 模式（無需核准）

若要執行主機 Exec 而不顯示核准提示，必須同時開放**兩個**政策層：
OpenClaw 設定中要求的 Exec 政策（`tools.exec.*`）**以及**
執行主機核准檔案中的主機本機核准政策。

省略 `askFallback` 時，預設為 `deny`。如果在沒有 UI 的情況下，核准提示應備援為允許，請明確將主機 `askFallback` 設為 `full`。

| 層級              | YOLO 設定               |
| ------------------ | -------------------------- |
| `tools.exec.mode`  | `gateway`/`node` 上的 `full` |
| 主機 `askFallback` | `full`                     |

<Warning>
**重要區別：**

- `tools.exec.host=auto` 選擇 exec 在**何處**執行：可用時在沙箱中，否則在閘道上。
- YOLO 選擇主機 exec 核准的**方式**：`security=full` 加上 `ask=off`。
- YOLO **不會**在已設定的主機 exec 政策之上，另加啟發式命令混淆核准閘門或指令碼預先檢查拒絕層。
- `auto` 不會讓閘道路由成為沙箱工作階段可任意覆寫的選項。來自 `auto` 的每次呼叫 `host=node` 請求皆獲允許；只有在沒有啟用沙箱執行階段時，才允許從 `auto` 使用 `host=gateway`。若要使用穩定的非自動預設值，請設定 `tools.exec.host`，或明確使用 `/exec host=...`。

</Warning>

公開自身非互動式權限模式、由命令列介面支援的供應商
可以遵循此政策。當 OpenClaw 的有效 exec
政策為 YOLO 時，Claude 命令列介面會加入
`--permission-mode bypassPermissions`。對於由 OpenClaw 管理的 Claude 即時工作階段，OpenClaw
的有效 exec 政策優先於 Claude 的原生權限模式：
YOLO 會將即時啟動正規化為 `--permission-mode bypassPermissions`，而
限制性的有效 exec 政策會將即時啟動正規化為
`--permission-mode default`，即使原始 Claude 後端引數指定其他
模式亦然。

若需要更保守的設定，請將 OpenClaw exec 政策收緊回
`allowlist` / `on-miss` 或 `deny`。

### 永久性的閘道主機「永不提示」設定

<Steps>
  <Step title="設定要求的設定政策">
    ```bash
    openclaw config set tools.exec.host gateway
    openclaw config set tools.exec.mode full
    openclaw gateway restart
    ```
  </Step>
  <Step title="配合主機核准檔案">
    ```bash
    openclaw approvals set --stdin <<'EOF'
    {
      version: 1,
      defaults: {
        security: "full",
        ask: "off",
        askFallback: "full"
      }
    }
    EOF
    ```
  </Step>
</Steps>

### 本機捷徑

```bash
openclaw exec-policy preset yolo
```

同時更新本機 `tools.exec.host/security/ask` 與本機核准
檔案的預設值（包括 `askFallback: "full"`）。此功能刻意
僅限本機。若要從遠端變更閘道主機或節點主機的核准，請使用
`openclaw approvals set --gateway` 或 `openclaw approvals set --node
<id|name|ip>`。

其他內建預設組合：`cautious`（`host=gateway`、`security=allowlist`、
`ask=on-miss`、`askFallback=deny`）與 `deny-all`（`host=gateway`、
`security=deny`、`ask=off`、`askFallback=deny`）。套用方式相同：
`openclaw exec-policy preset cautious`。

若要設定個別欄位，而非完整的預設組合，請使用
`openclaw exec-policy set --host <auto|sandbox|gateway|node> --security
<deny|allowlist|full> --ask <off|on-miss|always> --ask-fallback
<deny|allowlist|full>`，並搭配這些旗標的任意子集。

### 節點主機

改為在節點上套用相同的核准檔案：

```bash
openclaw approvals set --node <id|name|ip> --stdin <<'EOF'
{
  version: 1,
  defaults: {
    security: "full",
    ask: "off",
    askFallback: "full"
  }
}
EOF
```

<Note>
**僅限本機的限制：**

- `openclaw exec-policy` 不會同步節點核准。
- `openclaw exec-policy set --host node` 會遭到拒絕。
- 節點 exec 核准會在執行階段從節點擷取，因此以節點為目標的更新必須使用 `openclaw approvals --node ...`。

</Note>

### 僅限工作階段的捷徑

- `/exec security=full ask=off` 只會變更目前的工作階段。
- `/elevated full` 是緊急使用的捷徑，只有在要求的政策與主機核准檔案皆解析為
  `security: "full"` 和 `ask: "off"` 時，才會略過 exec 核准。較嚴格的主機檔案（例如 `ask:
"always"`）仍會提示。

若主機核准檔案維持比設定更嚴格的狀態，仍以較嚴格的主機
政策為準。

## 允許清單（每個代理程式）

允許清單是**每個代理程式各自獨立**的。若有多個代理程式，請在 macOS
應用程式中切換目前要編輯的代理程式。模式採用 glob 比對。

模式可以是解析後二進位檔路徑的 glob，也可以是純命令名稱的 glob。
純名稱只會比對透過 `PATH` 叫用的命令，因此當命令為 `rg` 時，`rg` 可以比對
`/opt/homebrew/bin/rg`，但**不能**比對 `./rg` 或
`/tmp/rg`。請使用路徑 glob 來信任單一特定位置的二進位檔。

舊版 `agents.default` 項目會在載入時移轉至 `agents.main`。
`echo ok && pwd` 等 Shell 鏈結仍要求每個頂層區段
都符合允許清單規則。

範例：

- `rg`
- `~/Projects/**/bin/peekaboo`
- `~/.local/bin/*`
- `/opt/homebrew/bin/rg`

### 使用 argPattern 限制引數

當允許清單項目應比對某個二進位檔及
特定引數形式時，請加入 `argPattern`。OpenClaw 在每個主機上使用 ECMAScript（JavaScript）規則
運算式語意，並針對剖析後的命令引數評估運算式，但不包括可執行檔權杖（`argv[0]`）。
對於手動撰寫的項目，引數會以單一空格連接，因此
需要完全比對時，請錨定模式。

```json
{
  "version": 1,
  "agents": {
    "main": {
      "allowlist": [
        {
          "pattern": "python3",
          "argPattern": "^safe\\.py$"
        }
      ]
    }
  }
}
```

該項目允許 `python3 safe.py`；`python3 other.py` 不符合允許
清單。若同一個二進位檔也有僅限路徑的項目，不符合的
引數仍可退回使用該僅限路徑的項目。若目標是將二進位檔限制為宣告的引數，請省略僅限路徑的
項目。

由核准流程儲存的項目會使用內部分隔符號格式，以精確
比對 argv。請優先使用使用者介面或核准流程重新產生這些項目，
而不要手動編輯編碼後的值。若 OpenClaw 無法剖析某個命令區段的 argv，
具有 `argPattern` 的項目不會相符。

產生的 `allow-always` 項目會繫結至 argv。新產生的項目包含
`argPattern`；較舊的產生式僅限路徑項目會被忽略，且需要重新
核准。若是手動的僅限路徑規則，請同時省略 `source` 與 `argPattern`。

每個允許清單項目支援：

| 欄位              | 意義                                                                  |
| ------------------ | ------------------------------------------------------------------------ |
| `pattern`          | 解析後二進位檔路徑 glob 或純命令名稱 glob                      |
| `argPattern`       | ECMAScript argv 規則運算式或產生的精確 argv 雜湊；省略時僅比對路徑 |
| `id`               | 穩定的不透明 ID；不存在時產生為 UUID                        |
| `source`           | 產生項目的來源，例如 `allow-always`；手動項目請省略  |
| `commandText`      | 舊版純文字輸入；載入期間捨棄                            |
| `lastUsedAt`       | 上次使用時間戳記                                                      |
| `lastUsedCommand`  | 上次相符的命令；產生的雜湊 argv 項目會省略     |
| `lastResolvedPath` | 上次解析出的二進位檔路徑                                                |

## 自動允許 Skills 命令列介面

啟用 **自動允許 Skills 命令列介面**（`autoAllowSkills`）後，已知 Skills
所參照的可執行檔在節點上（macOS 節點或無介面節點主機）
會視為已列入允許清單。此功能透過閘道 RPC 使用 `skills.bins`
擷取 Skill 二進位檔清單。若需要嚴格的手動
允許清單，請停用此功能。

<Warning>
- 這是一份**隱含的便利允許清單**，與手動路徑允許清單項目分開。
- 其適用於閘道與節點位於相同信任邊界內的受信任操作員環境。
- 若要求嚴格的明確信任，請維持 `autoAllowSkills: false`，並且只使用手動路徑允許清單項目。

</Warning>

## 安全二進位檔與核准轉送

如需瞭解安全二進位檔（僅限 stdin 的快速路徑）、直譯器繫結詳細資訊，以及
如何將核准提示轉送至 Slack/Discord/Telegram（或作為
原生核准用戶端執行），請參閱
[Exec 核准－進階](/zh-TW/tools/exec-approvals-advanced)。

## Control UI 編輯

使用 **Control UI -> Nodes -> Exec approvals** 卡片編輯預設值、
每個代理程式的覆寫值，以及允許清單。選擇範圍（Defaults 或某個代理程式）、
調整政策、新增或移除允許清單模式，然後按 **Save**。使用者介面
會顯示每個模式的上次使用中繼資料，方便維持清單整潔。

目標選擇器可選擇 **Gateway**（本機核准）或 **Node**。
節點必須公告 `system.execApprovals.get/set`（macOS 應用程式或無介面
節點主機）。若節點尚未公告 exec 核准，請直接編輯其
本機核准檔案。

部分節點主機（包括 Windows 夥伴應用程式）使用不同的核准
政策格式。Control UI 會以唯讀方式顯示這些主機原生政策。請使用
夥伴應用程式或搭配原生政策形式的 `openclaw approvals set --node <id|name|ip>`
進行編輯；請參閱[核准命令列介面](/zh-TW/cli/approvals)。

命令列介面：`openclaw approvals` 支援閘道或節點編輯－請參閱
[核准命令列介面](/zh-TW/cli/approvals)。

## 核准流程

需要提示時，閘道會向操作員用戶端廣播
`exec.approval.requested`。Control UI 與 macOS
應用程式會透過 `exec.approval.resolve` 解決該提示，接著閘道會將
已核准的請求轉送至節點主機。

對於 `host=node`，核准請求包含標準化的 `systemRunPlan`
承載資料。轉送已核准的 `system.run` 請求時，閘道會將該計畫用作具權威性的命令／cwd／工作階段
內容：

- 節點 exec 路徑會預先準備一份標準計畫。
- 核准記錄會儲存該計畫及其繫結中繼資料。
- 核准後，最終轉送的 `system.run` 呼叫會重複使用已儲存的計畫，而不信任呼叫端後續的編輯。
- 若呼叫端在核准請求建立後變更 `command`、`rawCommand`、`cwd`、`agentId` 或 `sessionKey`，閘道會因核准不符而拒絕轉送的執行。

## 系統事件與拒絕

節點回報完成後，exec 生命週期會將 `Exec finished` 系統訊息張貼至代理程式的
工作階段。OpenClaw 也可以在核准授予後，
經過 `tools.exec.approvalRunningNoticeMs`（預設為 `10000`，`0` 會停用
此功能）時發出進行中通知。遭拒絕的 exec 核准對主機命令而言是終止狀態：命令
不會執行。

- 對於具有來源工作階段的主要代理程式非同步核准，OpenClaw
  會將拒絕作為內部後續訊息張貼回該工作階段，讓代理程式
  可以停止等待非同步命令，並避免缺少結果的修復作業。
- 若沒有工作階段或無法恢復工作階段，OpenClaw 仍可
  向操作員或直接聊天路由回報簡短的拒絕訊息。
- 子代理程式與排程工作階段的拒絕不會張貼回該
  工作階段。

閘道主機 exec 核准會發出相同的完成生命週期事件。
受核准管制的 exec 會重複使用核准 ID，以將待處理
請求與其完成／拒絕訊息建立關聯（`Exec finished (gateway
id=...)` / `Exec denied (gateway id=...)`）。

## 影響

- **`full`** 功能強大；請盡可能優先使用允許清單。
- **`ask`** 讓你持續掌握狀況，同時仍可快速核准。
- 每個代理程式各自獨立的允許清單，可防止某個代理程式的核准洩漏至其他代理程式。
- 核准只適用於來自**已授權傳送者**的主機 exec 請求。未授權的傳送者無法發出 `/exec`。
- `/exec security=full` 是供已授權操作員使用的工作階段層級便利功能，依設計會略過核准。若要硬性封鎖主機 exec，請將核准安全性設為 `deny`，或透過工具政策拒絕 `exec` 工具。

## 相關內容

<CardGroup cols={2}>
  <Card title="執行核准 - 進階" href="/zh-TW/tools/exec-approvals-advanced" icon="gear">
    安全執行檔、直譯器繫結，以及將核准轉送至聊天。
  </Card>
  <Card title="執行工具" href="/zh-TW/tools/exec" icon="terminal">
    Shell 命令執行工具。
  </Card>
  <Card title="提升權限模式" href="/zh-TW/tools/elevated" icon="shield-exclamation">
    同時略過核准的緊急存取路徑。
  </Card>
  <Card title="沙箱隔離" href="/zh-TW/gateway/sandboxing" icon="box">
    沙箱模式與工作區存取權限。
  </Card>
  <Card title="安全性" href="/zh-TW/gateway/security" icon="lock">
    安全性模型與強化。
  </Card>
  <Card title="沙箱、工具政策與提升權限的比較" href="/zh-TW/gateway/sandbox-vs-tool-policy-vs-elevated" icon="sliders">
    何時該採用各項控制措施。
  </Card>
  <Card title="Skills" href="/zh-TW/tools/skills" icon="sparkles">
    由 Skill 支援的自動允許行為。
  </Card>
</CardGroup>
