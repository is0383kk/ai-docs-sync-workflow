---
read_when:
    - 設定安全二進位檔或自訂安全二進位檔設定檔
    - 將核准請求轉送至 Slack、Discord、Telegram 或其他聊天管道
    - 為頻道實作原生核准用戶端
summary: 進階執行核准：安全二進位檔、直譯器繫結、核准轉送、原生傳遞
title: 執行核准 — 進階
x-i18n:
    generated_at: "2026-07-26T07:36:23Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: ac90d41f867a8ae4f14b6c9c13f3732d102a65707f456623932b858145a9bf46
    source_path: tools/exec-approvals-advanced.md
    workflow: 16
---

進階執行核准主題：`safeBins` 快速路徑、直譯器／執行階段繫結，以及將核准轉送至聊天頻道（包括原生傳遞）。
核心政策與核准流程請參閱[執行核准](/zh-TW/tools/exec-approvals)。

## 安全執行檔（僅限 stdin）

`tools.exec.safeBins` 指定**僅限 stdin** 的二進位執行檔（例如 `cut`），這些執行檔可在允許清單模式下執行，**無須**明確的允許清單項目。安全執行檔會拒絕位置檔案引數與類似路徑的權杖，因此只能處理傳入的串流。請將其視為串流篩選器的有限快速路徑，而非一般用途的信任清單。

<Warning>
請**勿**將直譯器或執行階段二進位執行檔（例如 `python3`、`node`、`ruby`、`bash`、`sh`、`zsh`）加入 `safeBins`。如果命令本身可評估程式碼、執行子命令或讀取檔案，請優先使用明確的允許清單項目，並保持啟用核准提示。自訂安全執行檔必須在 `tools.exec.safeBinProfiles.<bin>` 中定義明確的設定檔。
</Warning>

預設安全執行檔：

[//]: # "SAFE_BIN_DEFAULTS:START"

`cut`、`uniq`、`head`、`tail`、`tr`、`wc`

[//]: # "SAFE_BIN_DEFAULTS:END"

`grep` 與 `sort` 不在預設清單中。如果選擇啟用，請為其非 stdin 工作流程保留明確的允許清單項目。在安全執行檔模式中使用 `grep` 時，請透過 `-e`/`--regexp` 提供模式；系統會拒絕位置模式形式，以免檔案運算元偽裝成語意不明的位置引數。

### Argv 驗證與拒絕的旗標

驗證僅根據 argv 形態以確定性方式進行（不檢查主機檔案系統中是否存在檔案），以防止允許／拒絕差異形成檔案存在性預言機。預設安全執行檔會拒絕檔案導向選項；長選項採取失敗時關閉原則進行驗證（拒絕未知旗標與語意不明的縮寫）。預設執行檔可辨識的唯讀布林旗標（例如 `wc -l`、`tr -d`、`uniq -c`）會獲准，而無法辨識的短旗標則維持失敗時關閉，並轉由手動核准。

各安全執行檔設定檔拒絕的旗標：

[//]: # "SAFE_BIN_DENIED_FLAGS:START"

- `grep`：`--dereference-recursive`、`--directories`、`--exclude-from`、`--file`、`--recursive`、`-R`、`-d`、`-f`、`-r`
- `jq`：`--argfile`、`--from-file`、`--library-path`、`--rawfile`、`--slurpfile`、`-L`、`-f`
- `sort`：`--compress-program`、`--files0-from`、`--output`、`--random-source`、`--temporary-directory`、`-T`、`-o`
- `tail`：`--follow`、`--retry`、`-F`、`-f`
- `wc`：`--files0-from`

[//]: # "SAFE_BIN_DENIED_FLAGS:END"

安全執行檔還會強制在執行時將 argv 權杖視為**常值文字**（在僅限 stdin 的區段中不進行萬用字元展開，也不進行 `$VARS` 展開），因此無法利用 `*` 或 `$HOME/...` 之類的模式偷渡檔案讀取。`awk`、`sed` 與 `jq` 一律不得作為安全執行檔，因為無法驗證其語意是否僅限 stdin：`jq` 可以讀取環境資料，並從模組或啟動檔案載入 jq 程式碼。請為這些工具使用明確的允許清單項目或核准提示，而非 `safeBins`。

### 受信任的二進位執行檔目錄

安全執行檔必須解析自受信任的二進位執行檔目錄（系統預設值加上選用的 `tools.exec.safeBinTrustedDirs`）。系統絕不會自動信任 `PATH` 項目。預設受信任目錄刻意維持精簡：`/bin`、`/usr/bin`。如果你的安全執行檔位於套件管理員／使用者路徑中（例如 `/opt/homebrew/bin`、`/usr/local/bin`、`/opt/local/bin`、`/snap/bin`），請明確將其加入 `tools.exec.safeBinTrustedDirs`。

### Shell 串接、包裝器與多工器

只要每個頂層區段都符合允許清單（包括安全執行檔或 Skills 自動允許），即可使用 Shell 串接（`&&`、`||`、`;`）。允許清單模式仍不支援重新導向。系統會在解析允許清單時拒絕命令替換（`$()`／反引號），包括位於雙引號內的情況；若需要 `$()` 常值文字，請使用單引號。

在 macOS 輔助 App 核准中，如果原始 Shell 文字包含 Shell 控制或展開語法（`&&`、`||`、`;`、`|`、`` ` ``、`$`、`<`、`>`、`(`、`)`），除非 Shell 二進位執行檔本身已列入允許清單，否則會視為未命中允許清單。

對於 Shell 包裝器（`bash|sh|zsh ... -c/-lc`），要求範圍內的環境覆寫會縮減為一份精簡且明確的允許清單（`TERM`、`LANG`、`LC_*`、`COLORTERM`、`NO_COLOR`、`FORCE_COLOR`）。

對於允許清單模式中的 `allow-always` 決策，透明分派包裝器（例如 `env`、`flock`、`nice`、`nohup`、`stdbuf`、`timeout`）會保存內部執行檔路徑，而非包裝器路徑。Shell 多工器（`busybox`、`toybox`）也會以相同方式針對 Shell 小程式（`sh`、`ash` 等）進行解包裝。如果無法安全地解包裝某個包裝器或多工器，系統不會自動保存允許清單項目。

如果將 `python3` 或 `node` 等直譯器列入允許清單，請優先使用 `tools.exec.strictInlineEval=true`，如此一來，行內評估仍需要明確核准。在嚴格模式下，`allow-always` 仍可保存無害的直譯器／指令碼叫用，但不會自動保存行內評估載體。

### 安全執行檔與允許清單的比較

| 主題             | `tools.exec.safeBins`                                    | 允許清單（`exec-approvals.json`）                                                        |
| ---------------- | ------------------------------------------------------ | ---------------------------------------------------------------------------------- |
| 目標             | 自動允許範圍有限的 stdin 篩選器                       | 明確信任特定執行檔                                                               |
| 比對類型         | 執行檔名稱 + 安全執行檔 argv 政策                     | 已解析執行檔路徑 glob，或針對透過 PATH 叫用之命令的裸命令名稱 glob                |
| 引數範圍         | 受安全執行檔設定檔與常值權杖規則限制                 | 預設進行路徑比對；選用的 `argPattern` 可限制已解析的 argv                    |
| 典型範例         | `head`、`tail`、`tr`、`wc` | `jq`、`python3`、`node`、`ffmpeg`、自訂命令列介面 |
| 最佳用途         | 流水線中的低風險文字轉換                              | 任何具有較廣泛行為或副作用的工具                                                 |

設定位置：

- `safeBins` 來自設定（`tools.exec.safeBins` 或各代理程式的 `agents.entries.*.tools.exec.safeBins`）。
- `safeBinTrustedDirs` 來自設定（`tools.exec.safeBinTrustedDirs` 或各代理程式的 `agents.entries.*.tools.exec.safeBinTrustedDirs`）。
- `safeBinProfiles` 來自設定（`tools.exec.safeBinProfiles` 或各代理程式的 `agents.entries.*.tools.exec.safeBinProfiles`）。各代理程式設定檔的鍵會覆寫全域鍵。
- 允許清單項目位於 `agents.<id>.allowlist` 下的主機本機核准檔案中（或透過控制介面／`openclaw approvals allowlist ...` 管理）。
- 當直譯器／執行階段執行檔出現在 `safeBins` 中，但沒有明確設定檔時，`openclaw security audit` 會透過 `tools.exec.safe_bins_interpreter_unprofiled` 發出警告。
- `openclaw doctor --fix` 可將缺少的自訂 `safeBinProfiles.<bin>` 項目建構為 `{}`（之後請檢閱並收緊設定）。系統不會自動建構直譯器／執行階段執行檔的設定。

自訂設定檔範例：

```json5
{
  tools: {
    exec: {
      safeBins: ["myfilter"],
      safeBinProfiles: {
        myfilter: {
          minPositional: 0,
          maxPositional: 0,
          allowedValueFlags: ["-n", "--limit"],
          deniedFlags: ["-f", "--file", "-c", "--command"],
        },
      },
    },
  },
}
```

## 直譯器／執行階段命令

由核准支援的直譯器／執行階段執行方式刻意採取保守策略：

- 一律繫結確切的 argv/cwd/env 情境。
- 直接 Shell 指令碼與直接執行階段檔案形式會盡可能繫結至單一具體的本機檔案快照。
- 仍可解析至單一直接本機檔案的常見套件管理員包裝器形式（例如 `pnpm exec`、`pnpm node`、`npm exec`、`npx`）會先解包裝，再進行繫結。
- 如果 OpenClaw 無法為直譯器／執行階段命令精確識別唯一一個具體的本機檔案（例如套件指令碼、評估形式、特定執行階段載入器鏈，或語意不明的多檔案形式），則會拒絕由核准支援的執行，而不會宣稱提供實際上並不具備的語意涵蓋能力。
- 對於這些工作流程，請優先使用沙箱、獨立的主機邊界，或由操作人員接受較廣泛執行階段語意的明確受信任允許清單／完整工作流程。

需要核准時，執行工具會立即傳回核准 ID。請使用該 ID 關聯之後的已核准執行系統事件（`Exec finished`，以及設定時的 `Exec running`）。
如果在逾時前未收到決策，要求會視為核准逾時，並呈現為終止性的主機命令拒絕。對於具有來源工作階段的主要代理程式非同步核准，OpenClaw 也會透過內部後續訊息恢復該工作階段，讓代理程式得知命令未執行，而不會在稍後嘗試修復缺少的結果。待處理的執行核准預設會在 30 分鐘後到期。

### 後續傳遞行為

核准的非同步執行完成後，OpenClaw 會將後續 `agent` 回合傳送至同一工作階段。
遭拒的非同步核准會透過相同的主要工作階段後續路徑傳遞拒絕狀態，但不會登錄提高權限的執行階段交接，也不會執行命令。若拒絕沒有可恢復的主要工作階段，系統會抑制該拒絕，或在存在安全的直接路徑時透過該路徑回報。

- 如果存在有效的外部傳遞目標（可傳遞的頻道以及目標 `to`），後續傳遞會使用該頻道。
- 在只有網頁聊天或沒有外部目標的內部工作階段流程中，後續傳遞僅保留於工作階段（`deliver: false`）。
- 如果呼叫端明確要求嚴格外部傳遞，但沒有可解析的外部頻道，要求會以 `INVALID_REQUEST` 失敗。
- 如果已啟用 `bestEffortDeliver`，且無法解析任何外部頻道，傳遞會降級為僅限工作階段，而非失敗。

## 第三方用戶端的最小範圍

閘道核准解析受專用的 `operator.approvals` 範圍保護。這同時適用於擁有者特定的 `exec.approval.resolve` 方法與不限定種類的 `approval.resolve` 方法；`operator.write` 不包含該範圍。儀表板與整合應僅要求其所用方法必要的範圍。請將核准解析存取權視為遠端執行等級的權限，並審慎授予 `operator.approvals`，即使用戶端只顯示小型核准介面亦然。

## 將核准轉送至聊天頻道

你可以將 exec 核准提示轉送到任何聊天頻道（包括外掛頻道），並使用 `/approve` 核准這些提示。這會使用一般的對外傳遞流水線。

設定：

```json5
{
  approvals: {
    exec: {
      enabled: true,
      mode: "session", // "session" | "targets" | "both"
      agentFilter: ["main"],
      sessionFilter: ["discord"], // 子字串或規則運算式
      targets: [
        { channel: "slack", to: "U12345678" },
        { channel: "telegram", to: "123456789" },
      ],
    },
  },
}
```

在聊天中回覆：

```
/approve <id> allow-once
/approve <id> allow-always
/approve <id> deny
```

`/approve` 命令同時處理 exec 核准與外掛核准。如果 ID 不符合待處理的 exec 核准，它會自動改為檢查外掛核准。此備援僅限於「找不到核准」失敗；真正的 exec 核准拒絕／錯誤不會在未提示的情況下重試為外掛核准。

### 外掛核准轉送

外掛核准轉送使用與 exec 核准相同的傳遞流水線，但在 `approvals.plugin` 下有自己的獨立設定。啟用或停用其中一者不會影響另一者。
如需了解外掛編寫行為、要求欄位及決策語意，請參閱
[外掛權限要求](/plugins/plugin-permission-requests)。

```json5
{
  approvals: {
    plugin: {
      enabled: true,
      mode: "targets",
      agentFilter: ["main"],
      targets: [
        { channel: "slack", to: "U12345678" },
        { channel: "telegram", to: "123456789" },
      ],
    },
  },
}
```

設定結構與 `approvals.exec` 相同：`enabled`、`mode`、`agentFilter`、
`sessionFilter` 和 `targets` 的運作方式相同。

支援共用互動式回覆的頻道，會為 exec 與外掛核准顯示相同的核准按鈕。沒有共用互動式 UI 的頻道會改用純文字，並附上 `/approve` 指示。外掛核准要求可能會限制可用的決策：核准介面會使用要求中宣告的決策集合，而閘道會拒絕提交未提供之決策的嘗試。

### 在任何頻道的相同聊天中核准

當 exec 或外掛核准要求源自可傳遞訊息的聊天介面時，預設可在同一個聊天中使用 `/approve` 核准。除了現有的網頁 UI 與終端 UI 流程外，這也適用於 Slack、Matrix、Microsoft Teams 及類似的可傳遞訊息聊天，並使用該對話的一般頻道驗證模型。如果來源聊天已可傳送命令並接收回覆，核准要求就不再需要僅為了保持待處理狀態而使用獨立的原生傳遞介接器。

Discord、Telegram 和 QQ Bot 也支援在相同聊天中使用 `/approve`，但即使原生核准傳遞已停用，這些頻道仍會使用其解析出的核准者清單進行授權。

### 原生核准傳遞

部分頻道也能作為原生核准用戶端：Discord、Slack、Telegram、Matrix 和 QQ Bot。
除了共用的相同聊天 `/approve` 流程外，原生用戶端還會加入核准者私訊、來源聊天扇出，以及頻道特有的互動式核准使用者體驗。

當原生核准卡片／按鈕可用時，該原生 UI 是面向代理程式的主要路徑。除非工具結果指出聊天核准不可用，或手動核准是唯一剩餘路徑，否則代理程式不應同時在一般聊天中重複顯示 `/approve` 命令。

如果已設定原生核准用戶端，但來源頻道沒有作用中的原生執行階段，OpenClaw 會保持顯示本機的確定性 `/approve` 提示。如果原生執行階段已作用並嘗試傳遞，但沒有任何目標收到卡片，OpenClaw 會在相同聊天中傳送備援通知，其中包含確切的 `/approve <id> <decision>` 命令，讓要求仍可獲得處理。

通用模型：

- 主機 exec 原則仍會決定是否需要 exec 核准
- `approvals.exec` 控制是否將核准提示轉送到其他聊天目的地
- `channels.<channel>.execApprovals` 控制是否啟用 Discord、Slack、Telegram、QQ Bot 及類似的頻道特定原生用戶端
- 當要求來自 Slack 且可解析出 Slack 外掛核准者時，Slack 外掛核准可以使用 Slack 的原生核准用戶端；即使 Slack exec 核准已停用，`approvals.plugin` 也可以將外掛核准路由至 Slack 工作階段或目標
- 當可從 `dm.allowFrom` 或 `defaultTo` 解析出穩定的 `users/<id>` 核准者時，Google Chat 原生核准卡片會處理源自 Google Chat 空間或討論串的 exec 與外掛核准；它們不會使用表情回應事件進行決策
- WhatsApp 與 Signal 的表情回應核准傳遞受 `approvals.exec` 和 `approvals.plugin` 控制；它們沒有 `channels.<channel>.execApprovals` 區塊

當下列所有條件皆成立時，原生核准用戶端會自動啟用私訊優先傳遞：

- 頻道支援原生核准傳遞
- 可從明確的 `execApprovals.approvers` 或擁有者身分（例如 `commands.ownerAllowFrom`）解析出核准者
- `channels.<channel>.execApprovals.enabled` 未設定或為 `"auto"`

設定 `enabled: false` 可明確停用原生核准用戶端。設定 `enabled: true` 可在解析出核准者時強制啟用。公開的來源聊天傳遞仍須透過 `channels.<channel>.execApprovals.target` 明確啟用。當原生 `target` 啟用來源聊天傳遞時，核准提示會包含命令文字。

常見問題：[為什麼聊天核准有兩個 exec 核准設定？](/help/faq-first-run)

- Discord：`channels.discord.execApprovals.*`
- Slack：`channels.slack.execApprovals.*`
- Telegram：`channels.telegram.execApprovals.*`
- QQ Bot：`channels.qqbot.execApprovals.*`
- Google Chat：使用 `channels.googlechat.dm.allowFrom` 或 `channels.googlechat.defaultTo` 設定穩定的核准者；不需要 `execApprovals` 區塊
- WhatsApp：使用 `approvals.exec` 和 `approvals.plugin` 將核准提示路由至 WhatsApp
- Signal：使用 `approvals.exec` 和 `approvals.plugin` 將核准提示路由至 Signal

原生用戶端特定路由：

- Telegram 預設傳送至核准者私訊（`target: "dm"`）。切換至 `channel` 或 `both`，也可在來源 Telegram 聊天／主題中顯示核准提示。對於 Telegram 論壇主題，OpenClaw 會保留核准提示及核准後後續訊息所屬的主題。
- Discord 和 Telegram 核准者可以明確指定（`execApprovals.approvers`），或從 `commands.ownerAllowFrom` 推斷；只有解析出的核准者才能核准或拒絕。
- Slack 核准者可以明確指定（`execApprovals.approvers`），或從 `commands.ownerAllowFrom` 推斷。Slack 外掛核准私訊會使用來自 `allowFrom` 的 Slack 外掛核准者與帳號預設路由，而不是 Slack exec 核准者。Slack 原生按鈕會保留核准 ID 的種類，因此 `plugin:` ID 可以解析外掛核准，無須第二層 Slack 本機備援。
- Google Chat 原生卡片會在訊息文字中保留手動 `/approve` 備援，但卡片按鈕回呼只會攜帶不透明的動作權杖；核准 ID 與決策會從伺服器端的待處理狀態中復原。
- 當相符的頂層轉送系列路由至 WhatsApp 時，WhatsApp 表情符號核准會同時處理 exec 與外掛提示。原生來源提示會直接繫結；共用目標模式傳遞則會將相同的具型別核准中繼資料，繫結至已接受的 WhatsApp 訊息收件結果。
- 只有當相符的頂層轉送系列已啟用並路由至 Signal 時，Signal 表情回應核准才會同時處理 exec 與外掛提示。直接在相同聊天中的 Signal exec 核准，可以在沒有明確核准者的情況下隱藏本機 `/approve` 備援；Signal 表情回應解析仍需要來自 `channels.signal.allowFrom` 或 `defaultTo` 的明確 Signal 核准者。
- Matrix 原生私訊／頻道路由及表情回應捷徑會同時處理 exec 與外掛核准；外掛授權仍來自 `channels.matrix.dm.allowFrom`。Matrix 原生提示會在第一個提示事件中包含 `com.openclaw.approval` 自訂事件內容，讓支援 OpenClaw 的 Matrix 用戶端能讀取結構化核准狀態，同時讓標準用戶端保留純文字 `/approve` 備援。
- 原生 Discord 與 Telegram 核准按鈕會在傳輸私有回呼資料中攜帶明確的 exec 或外掛擁有者種類，且只解析該擁有者。缺少種類的舊版 `/approve` 控制項仍是有限的相容性路徑：它們只會嘗試行為者可以核准的擁有者種類，僅在收到找不到核准的結果後才會繼續，而且絕不會從核准 ID 推斷擁有權。
- 要求者不需要是核准者。
- 如果沒有任何操作者 UI 或已設定的核准用戶端可以接受要求，提示會改用 `askFallback`。

`/diagnostics` 和 `/export-trajectory` 等敏感且僅限擁有者使用的群組命令，會對核准提示和最終結果使用私有擁有者路由。OpenClaw 會先嘗試在擁有者執行命令的相同介面上使用私有路由。如果該介面沒有私有擁有者路由，便會改用 `commands.ownerAllowFrom` 中第一個可用的擁有者路由，因此當 Telegram 是已設定的主要私有介面時，Discord 群組命令仍可將核准和結果傳送至擁有者的 Telegram 私訊。群組聊天只會收到簡短的確認訊息。

另請參閱：

- [Discord](/channels/discord)
- [Telegram](/channels/telegram)
- [QQ Bot](/channels/qqbot)

### 官方行動操作者應用程式

使用 `operator.admin` 連線，或要求明確指定其已配對的 `operator.approvals` 裝置時，官方 iOS 與 Android 應用程式也可以審查由閘道擁有的待處理 exec 核准。它們會讀取與 Control UI 相同的已清理持久記錄、提交可識別種類的決策，並顯示閘道的標準首次作答結果。Apple Watch 會透過配對的 iPhone 鏡像這些核准提示，並提供允許一次與拒絕動作。Watch 直接閘道模式不會審查核准。

遺失解析確認不會使已提交的選擇成為權威結果：應用程式會停用控制項並再次讀取記錄。如果另一個介面先完成處理，應用程式會顯示該筆已記錄的決策。待處理提示會持續繫結至發出提示的閘道，因此切換作用中的閘道無法重新導向舊的核准 ID。

### macOS IPC 流程

```
閘道 -> 節點服務 (WS)
                 |  IPC (UDS + 權杖 + HMAC + TTL)
                 v
             Mac 應用程式 (UI + 核准 + system.run)
```

安全性注意事項：

- Unix 通訊端模式 `0600`，權杖儲存於 `exec-approvals.json`。
- 相同 UID 的對等端檢查。
- 挑戰／回應（nonce + HMAC 權杖 + 要求雜湊）+ 短 TTL。

## 常見問題

### 在核准目標上，何時會使用 `accountId` 和 `threadId`？

當頻道設定了多個身分，且核准提示必須透過特定帳號送出時，請使用 `accountId`。當目的地支援主題或討論串，且提示應留在該討論串內而非頂層聊天時，請使用 `threadId`。

具體的 Telegram 案例是：一個具有論壇主題的營運超級群組，以及兩個 Telegram Bot 帳號。`to` 值指定超級群組，`accountId` 選取 Bot 帳號，而 `threadId` 選取論壇主題：

```json5
{
  approvals: {
    exec: {
      enabled: true,
      mode: "targets",
      targets: [
        {
          channel: "telegram",
          to: "-1001234567890",
          accountId: "ops-bot",
          threadId: "77",
        },
      ],
    },
  },
  channels: {
    telegram: {
      accounts: {
        default: {
          name: "Primary bot",
          botToken: "env:TELEGRAM_PRIMARY_BOT_TOKEN",
        },
        "ops-bot": {
          name: "Operations bot",
          botToken: "env:TELEGRAM_OPS_BOT_TOKEN",
        },
      },
    },
  },
}
```

使用此設定時，轉送的執行核准會由 `ops-bot` Telegram 帳號發佈至聊天 `-1001234567890` 的主題
`77`。未指定 `accountId` 的目標會使用該頻道的預設帳號，而未指定
`threadId` 的目標則會發佈至頂層目的地。

### 核准傳送至工作階段時，該工作階段中的任何人都能核准嗎？

不能。工作階段傳遞只控制提示出現的位置，本身並不授權該聊天中的每位
參與者進行核准。

對於一般的同聊天 `/approve`，傳送者必須已獲授權，能在該
頻道工作階段中執行命令。若頻道公開了明確的核准者，這些核准者即使在該工作階段中未獲得其他命令授權，
仍可授權 `/approve` 動作。

有些頻道的限制更嚴格。Discord、Telegram、Matrix、Slack 原生核准私訊，以及類似的
原生核准用戶端，會使用其解析出的核准者清單來進行核准授權。例如，
Telegram 論壇主題的核准提示可能對主題中的所有人可見，但只有從 `channels.telegram.execApprovals.approvers` 或
`commands.ownerAllowFrom` 解析出的數字 Telegram 使用者 ID 才能核准或拒絕。

## 相關內容

- [執行核准](/zh-TW/tools/exec-approvals) — 核心政策與核准流程
- [執行工具](/zh-TW/tools/exec)
- [提升模式](/zh-TW/tools/elevated)
- [Skills](/zh-TW/tools/skills) — Skills 支援的自動允許行為
