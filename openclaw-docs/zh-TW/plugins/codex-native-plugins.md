---
read_when:
    - 你希望 Codex 模式的 OpenClaw 代理使用原生 Codex 外掛
    - 你正在遷移從原始碼安裝、由 OpenAI 精選的 Codex 外掛
    - 你正在設定現有工作區目錄中的 Codex 外掛
    - 你正在針對 codexPlugins、應用程式清單、破壞性操作或外掛應用程式診斷進行疑難排解
summary: 為 Codex 模式的 OpenClaw 代理程式設定原生 Codex 外掛
title: 原生 Codex 外掛
x-i18n:
    generated_at: "2026-07-26T08:02:35Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 0b1cfa39838d4dbd1f33a1e5b7f52faec4b033f9fa98ef5c029003177c2e27e5
    source_path: plugins/codex-native-plugins.md
    workflow: 16
---

原生 Codex 外掛支援可讓採用 Codex 模式的 OpenClaw 代理程式，在處理 OpenClaw 回合的同一個 Codex 執行緒內，使用 Codex app-server 自有的應用程式與外掛功能。外掛呼叫會保留在原生 Codex 逐字稿中；由 Codex app-server 負責執行應用程式支援的 MCP。OpenClaw 不會將 Codex 外掛轉換成合成的 `codex_plugin_*` OpenClaw 動態工具。

請在基礎 [Codex 控制框架](/zh-TW/plugins/codex-harness)正常運作後使用本頁。

## 需求

- 代理程式執行階段必須是原生 Codex 控制框架。
- `plugins.entries.codex.enabled` 為 `true`。
- `plugins.entries.codex.config.codexPlugins.enabled` 為 `true`。
- 目標 Codex app-server 必須能看見預期的市集、外掛與
  應用程式清單。
- 遷移僅支援它在來源 Codex 主目錄中觀察到以原始碼安裝的
  `openai-curated` 外掛。
- 手動設定的 `workspace-directory` 外掛需要使用符合以下條件的 Codex app-server：
  其 `plugin/list` 接受 `marketplaceKinds`，且其無路徑工作區摘要包含
  `remotePluginId`。此外，外掛必須已安裝並啟用，且其所屬應用程式必須能在
  `app/list` 中存取。

`codexPlugins` 不會影響 OpenClaw 提供者執行、ACP 對話繫結或其他控制框架，因為這些路徑絕不會建立具有原生 `apps` 設定的 Codex app-server 執行緒。

OpenAI 端的 Codex 帳戶、應用程式可用性及工作區應用程式／外掛控制項，均來自已登入的 Codex 帳戶。關於 OpenAI 帳戶與管理員模型，請參閱[透過 ChatGPT 方案使用 Codex](https://help.openai.com/en/articles/11369540-using-codex-with-your-chatgpt-plan)。

## 快速入門

預覽從來源 Codex 主目錄進行的遷移：

```bash
openclaw migrate codex --dry-run
```

加入 `--verify-plugin-apps`，讓遷移呼叫來源 `app/list`，並要求每個所屬應用程式都存在、已啟用且可存取，之後才規劃原生啟用：

```bash
openclaw migrate codex --dry-run --verify-plugin-apps
```

確認計畫無誤後套用遷移：

```bash
openclaw migrate apply codex --yes
```

遷移會為符合資格的外掛寫入明確的 `codexPlugins` 項目，並針對選取的外掛呼叫 Codex app-server `plugin/install`。遷移後的設定如下：

```json5
{
  plugins: {
    entries: {
      codex: {
        enabled: true,
        config: {
          codexPlugins: {
            enabled: true,
            allow_destructive_actions: true,
            plugins: {
              "google-calendar": {
                enabled: true,
                marketplaceName: "openai-curated",
                pluginName: "google-calendar",
              },
            },
          },
        },
      },
    },
  },
}
```

遷移仍僅限於 `openai-curated`。若要使用現有的 `workspace-directory` 外掛，請使用 `plugin/list` 傳回的確切市集限定 `summary.id` 手動新增。例如，若 Codex 傳回 `example-plugin@workspace-directory`，請設定該完整值，而非其顯示名稱：

```json5
{
  plugins: {
    entries: {
      codex: {
        enabled: true,
        config: {
          codexPlugins: {
            enabled: true,
            plugins: {
              "example-plugin": {
                enabled: true,
                marketplaceName: "workspace-directory",
                pluginName: "example-plugin@workspace-directory",
              },
            },
          },
        },
      },
    },
  },
}
```

OpenClaw 不會呼叫 `plugin/install`，也不會為 `workspace-directory` 外掛啟動驗證。請先在 Codex 中安裝、啟用並驗證該外掛，再新增或啟用 OpenClaw 原則。當回應缺少確切的市集、外掛 ID、詳細資料 ID 或應用程式就緒證據時，OpenClaw 會持續隱藏應用程式。若 Codex 拒絕明確的工作區 `plugin/list` 要求，OpenClaw 會針對每個已啟用的工作區外掛回報 `marketplace_missing`，並維持任何獨立探索到的精選外掛可用。

`codexPlugins` 變更後，新的 Codex 對話會自動採用更新後的應用程式集合。執行 `/new` 或 `/reset` 可重新整理目前對話。啟用或停用外掛不需要重新啟動閘道。

## 從聊天管理外掛

`/codex plugins` 可從你操作 Codex 控制框架的同一個聊天中，檢查或變更已設定的原生 Codex 外掛：

```text
/codex plugins
/codex plugins list
/codex plugins disable google-calendar
/codex plugins enable google-calendar
```

`/codex plugins` 是 `/codex plugins list` 的別名。清單會顯示每個已設定外掛的索引鍵、開啟／關閉狀態、Codex 外掛名稱，以及來自 `plugins.entries.codex.config.codexPlugins.plugins` 的市集。

`enable`/`disable` 只會寫入 `~/.openclaw/openclaw.json`；絕不會編輯 `~/.codex/config.toml` 或安裝新的 Codex 外掛。只有擁有者或具備 `operator.admin` 範圍的閘道用戶端才能執行這些操作。

啟用已設定的外掛時，也會開啟全域 `codexPlugins.enabled` 開關。若精選外掛因遷移傳回 `auth_required` 而被寫入為停用，請先在 Codex 中重新授權應用程式，再於 OpenClaw 中啟用。對於 `workspace-directory` 項目，在此啟用只會變更 OpenClaw 原則；外掛與應用程式必須已在 Codex 中啟用。

## 原生外掛設定的運作方式

此整合會追蹤三種狀態：

| 狀態      | 含義                                                                                                                            |
| ---------- | ---------------------------------------------------------------------------------------------------------------------------------- |
| 已安裝  | Codex 在目標 app-server 執行階段中有該外掛套件。                                                                      |
| 已啟用    | Codex 回報該外掛已啟用，且 OpenClaw 設定允許 Codex 控制框架回合使用它。                                           |
| 可存取 | Codex app-server 確認該外掛的應用程式項目可供使用中帳戶存取，且可對應至設定的外掛身分。 |

對於 `openai-curated` 外掛，遷移是持久的安裝／資格確認步驟：

- 規劃期間，OpenClaw 會讀取來源 Codex `plugin/read` 詳細資料，並
  檢查來源 Codex app-server 帳戶是否為 ChatGPT 訂閱帳戶。非 ChatGPT
  帳戶或缺少帳戶回應時，會以 `codex_subscription_required` 略過應用程式支援的外掛。
- 依預設，遷移會略過來源 `app/list` 呼叫：通過帳戶關卡的應用程式支援來源外掛
  會在未驗證來源應用程式可存取性的情況下納入規劃；帳戶查詢傳輸失敗則會以
  `codex_account_unavailable` 略過。
- 使用 `--verify-plugin-apps` 時，遷移會取得新的來源 `app/list`
  快照，並要求每個所屬應用程式都存在、已啟用且可存取，之後才規劃原生啟用。此時帳戶查詢傳輸失敗
  會改為進入來源應用程式清單關卡，而非直接略過。

對於 `workspace-directory` 外掛，設定會在 OpenClaw 外部完成。只有至少設定一個已啟用的工作區項目時，OpenClaw 才會查詢該市集，依確切的 `summary.id` 解析各外掛，並重複使用現有的 `plugin/read` 所有權與 `app/list` 就緒檢查。未安裝、已停用、無法存取或未驗證的外掛不會公開任何應用程式；OpenClaw 不會嘗試安裝或驗證。

對於遷移後的精選外掛與手動設定的工作區外掛，執行階段應用程式清單都是目標工作階段的可存取性檢查。Codex 控制框架工作階段設定會依已啟用且可存取的外掛應用程式，計算限制性的執行緒應用程式設定；此設定不會在每個回合重新計算，因此 `/codex plugins enable`/`disable` 只會影響新的 Codex 對話。使用 `/new` 或 `/reset` 可讓目前對話採用變更。

## V1 支援範圍

- 只有已安裝在來源 Codex app-server 清單中的 `openai-curated` 外掛
  符合遷移資格。
- 在 app-server 建置的 `plugin/list` 實作 `marketplaceKinds`，且針對無路徑工作區摘要傳回
  `remotePluginId` 時，執行階段也支援明確的 `workspace-directory` 項目。這些項目必須使用其確切的市集限定
  `summary.id`，且必須已安裝、已啟用並可存取應用程式。遭拒絕的工作區清單要求會產生現有的個別外掛
  `marketplace_missing` 診斷；缺少市集、外掛、詳細資料或應用程式證據時，不會公開任何工作區應用程式。預設清單要求中的精選清單仍可使用。
- 應用程式支援的來源外掛必須通過遷移時的訂閱關卡。
  `--verify-plugin-apps` 會加入來源應用程式清單關卡。受訂閱限制的帳戶，以及在驗證模式中無法存取、已停用或缺少來源應用程式，或應用程式清單重新整理失敗的情況，都會回報為已略過的手動項目，而非已啟用的設定項目。無法讀取的外掛詳細資料會在應用程式清單關卡前略過。
- 遷移會寫入明確的外掛身分（`marketplaceName` 和
  `pluginName`）；不會寫入本機 `marketplacePath` 快取路徑。
- `codexPlugins.enabled` 是唯一的全域啟用開關；沒有
  `plugins["*"]` 萬用字元或設定鍵可授予任意安裝權限。
- 非精選市集、已快取的外掛套件、鉤子及 Codex 設定檔會保留在遷移報告中供手動審查，不會自動啟用。執行階段接受手動設定的 `workspace-directory` 項目；其他市集仍不受支援。

## 應用程式清單與所有權

OpenClaw 透過 app-server `app/list` 讀取 Codex 應用程式清單，在記憶體中快取一小時，並以非同步方式重新整理過期或缺少的項目。快取位於處理程序本機；重新啟動命令列介面或閘道會將其清除，而 OpenClaw 會在下一次讀取 `app/list` 時重建。

遷移與執行階段使用不同的快取索引鍵：

- 來源遷移驗證使用來源 Codex 主目錄與啟動選項。它只會搭配 `--verify-plugin-apps` 執行，並強制為該次規劃執行全新的來源 `app/list` 周遊。
- 目標執行階段設定在建置執行緒應用程式設定時，使用目標代理程式的 Codex app-server 身分。精選外掛啟用會使該目標快取索引鍵失效，接著在 `plugin/install` 後強制重新整理。
  `workspace-directory` 設定絕不會執行此啟用路徑。

只有當 OpenClaw 能透過穩定的所有權將外掛應用程式對應回已設定的外掛時，才會公開該應用程式：外掛詳細資料中的確切應用程式 ID、已知的 MCP 伺服器名稱，或唯一的穩定中繼資料。僅有顯示名稱或所有權不明確的項目會被排除，直到下次清單重新整理證實其所有權。

## 已連線帳戶的應用程式

由擁有者操作的代理程式可以選擇使用已連線至其 Codex 帳戶的所有應用程式，而不需要相符的外掛套件：

```json5
{
  plugins: {
    entries: {
      codex: {
        enabled: true,
        config: {
          codexPlugins: {
            enabled: true,
            allow_all_plugins: true,
            allow_destructive_actions: "auto",
          },
        },
      },
    },
  },
}
```

`allow_all_plugins: true` 會在建立新的原生 Codex 執行緒時取得完整的 `app/list` 快照，且只允許該帳戶中標示為可存取的應用程式。它不會安裝、驗證或全域啟用應用程式。現有執行緒會保留其持久化的應用程式集合；使用 `/new`、`/reset` 或重新啟動閘道，即可採用新連線或已撤銷的應用程式。

帳戶應用程式會繼承全域 `codexPlugins.allow_destructive_actions` 值，
其可接受 `true`、`false`、`"auto"` 或 `"ask"`。明確的各外掛政策
會在應用程式 ID 重疊時覆寫全域政策。清查失敗時會採取
封閉式失敗，而不會退回不受限制的預設值。

## 執行緒應用程式設定

OpenClaw 會為 Codex 執行緒注入限制性的 `config.apps` 修補：
停用 `_default`，且僅啟用由已啟用且已設定之外掛擁有的應用程式，或
經 `allow_all_plugins` 准許存取的帳戶應用程式。

每個應用程式上的 `destructive_enabled` 來自有效的全域或
各外掛 `allow_destructive_actions` 政策；`true`、`"auto"` 和 `"ask"`
都會設定 `destructive_enabled: true`，而 `false` 會將其設定為 `false`。Codex 仍會
強制執行其原生應用程式工具註解中的破壞性工具中繼資料。
`_default` 會透過 `open_world_enabled: false` 停用；已啟用的外掛應用程式
會取得 `open_world_enabled: true`。OpenClaw 不會公開獨立的
外掛層級開放世界政策控制項，也不會維護各外掛的
破壞性工具名稱拒絕清單。

對已准許的應用程式，工具核准模式預設為自動，因此非破壞性的
讀取工具無須同一執行緒中的核准提示即可執行。破壞性工具仍由
各應用程式的 `destructive_enabled` 政策控制。

## 破壞性動作政策

對已設定的 Codex 外掛，預設允許破壞性外掛引導要求，
但不安全的結構描述和模稜兩可的擁有權會採取封閉式失敗：

- 全域 `allow_destructive_actions` 預設為 `true`。
- 各外掛 `allow_destructive_actions` 會覆寫該外掛的
  全域政策。
- `false`：OpenClaw 會傳回確定性的拒絕。
- `true`：OpenClaw 只會自動接受能對應至核准
  回應的安全結構描述，例如布林值核准欄位。
- `"auto"`：OpenClaw 會向 Codex 公開破壞性外掛動作，接著
  將已證明擁有權的 MCP 核准引導要求轉換為 OpenClaw 外掛
  核准，再傳回 Codex 核准回應。
- `"ask"`：OpenClaw 會使用與
  `"auto"` 相同的 Codex 寫入／破壞性閘控，在執行緒啟動前清除該應用程式
  持久保存的 Codex 各工具核准覆寫，並且只提供單次核准或拒絕，讓
  持久核准無法抑制之後的寫入動作提示。對於每個使用
  `"ask"` 的已准許應用程式，OpenClaw 會為該應用程式選取 Codex 的人工核准
  審查者，讓 Codex 將其核准引導要求傳送至
  OpenClaw；其他應用程式和非應用程式執行緒核准則保留其設定的
  審查者和政策。
- 缺少外掛身分、擁有權模稜兩可、缺少或不相符的
  回合 ID，或不安全的引導要求結構描述，都會直接拒絕而不提示。

## 疑難排解

| 代碼                                              | 含義                                                                                                                              | 修正方式                                                                                                                    |
| ------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------ | ---------------------------------------------------------------------------------------------------------------------- |
| `auth_required`                                   | 遷移已安裝外掛，但其中一個應用程式仍需要驗證。該項目會以停用狀態寫入，直到你重新授權。 | 在 Codex 中重新授權應用程式，然後在 OpenClaw 中啟用外掛。                                                      |
| `app_inaccessible`、`app_disabled`、`app_missing` | 使用 `--verify-plugin-apps` 時，來源 Codex 應用程式清查未顯示所有擁有的應用程式均存在、已啟用且可存取。         | 在 Codex 中重新授權或啟用應用程式，然後使用 `--verify-plugin-apps` 重新執行遷移。                              |
| `app_inventory_unavailable`                       | 已要求嚴格驗證來源應用程式，但重新整理來源 Codex 應用程式清查時失敗。                                      | 修復來源 Codex 應用程式伺服器的存取問題，或不使用 `--verify-plugin-apps` 重試，以接受較快、由帳戶閘控的方案。   |
| `codex_subscription_required`                     | 來源 Codex 應用程式伺服器帳戶不是 ChatGPT 訂閱帳戶。                                                          | 使用訂閱驗證登入 Codex 應用程式，然後重新執行遷移。                                                  |
| `codex_account_unavailable`                       | 無法讀取來源 Codex 應用程式伺服器帳戶。                                                                               | 修復來源 Codex 應用程式伺服器的驗證，或使用 `--verify-plugin-apps` 重新執行，讓來源應用程式清查決定資格。 |
| `marketplace_missing`、`plugin_missing`           | 市集或指定外掛無法使用；明確的工作區目錄要求可能已遭拒絕；工作區應用程式會採取封閉式失敗。  | 驗證下方所述的相容應用程式伺服器合約和確切 ID。                                                |
| `plugin_detail_unavailable`                       | OpenClaw 無法讀取外掛擁有權詳細資料。                                                                                    | 檢查目標應用程式伺服器的 `plugin/list` 和 `plugin/read` 回應。                                             |
| `plugin_disabled`                                 | Codex 回報外掛已安裝但已停用。                                                                                     | 策選啟用程序可能會修復此問題；重試前，請先在 Codex 中啟用工作區外掛。                                  |
| `plugin_activation_failed`                        | 外掛啟用未完成。                                                                                                  | 使用隨附的診斷資訊，區分市集、驗證、重新整理或工作區就緒失敗。                |
| `app_inventory_missing`、`app_inventory_stale`    | 應用程式就緒狀態來自空白或過時的快取。                                                                                     | OpenClaw 會自動排程非同步重新整理；在擁有權和就緒狀態確認前，外掛應用程式會維持排除狀態。  |
| `app_ownership_ambiguous`                         | 應用程式清查僅依顯示名稱相符。                                                                                          | 在後續重新整理證明擁有權前，該應用程式會持續從 Codex 執行緒中隱藏。                                     |

**工作區外掛已安裝但不可見：**確認工作區
`plugin/list` 結果將確切的已設定 ID 回報為已安裝且已啟用，
接著確認 `app/list` 回報同一個 Codex
帳戶可以存取每個擁有的應用程式。即使帳戶清查目前回報該應用程式已停用，
OpenClaw 仍可為執行緒啟用可存取的應用程式。如果你在閘道快取應用程式
清查後變更該狀態，請等待一小時的快取重新整理或重新啟動閘道，然後使用
`/new` 或 `/reset`。OpenClaw 不會修復工作區外掛，也不會為其進行驗證。
如果明確的工作區清單要求遭拒，每個已啟用的工作區
項目都會回報 `marketplace_missing`；不相關的策選項目仍會根據
預設清單回應繼續處理。

對於 `plugin_detail_unavailable`，不含路徑的工作區摘要必須包含
`remotePluginId`；如果該選擇器或後續的
`plugin/read` 結果無法使用，OpenClaw 會繼續隱藏擁有的應用程式。對於
`plugin_activation_failed`，策選外掛可能會回報市集、驗證或
安裝後重新整理失敗。當工作區外掛尚未啟用時，會回報此代碼；
請在 OpenClaw 外部安裝、啟用並驗證該外掛。

**設定已變更，但代理程式看不到外掛：**執行 `/codex plugins
list` 以確認設定狀態，然後執行 `/new` 或 `/reset`。現有的
Codex 執行緒繫結會保留其啟動時的應用程式設定，直到 OpenClaw
建立新的控管器工作階段或取代過時的繫結。

**破壞性動作遭拒絕：**檢查全域和各外掛的
`allow_destructive_actions` 值。即使使用 `true`、`"auto"` 或 `"ask"`，
不安全的引導要求結構描述和模稜兩可的外掛身分仍會採取封閉式失敗。

## 相關內容

- [Codex 控管器](/zh-TW/plugins/codex-harness)
- [Codex 控管器參考](/zh-TW/plugins/codex-harness-reference)
- [Codex 控管器執行階段](/zh-TW/plugins/codex-harness-runtime)
- [設定參考](/zh-TW/gateway/configuration-reference#codex-harness-plugin-config)
- [遷移命令列介面](/zh-TW/cli/migrate)
