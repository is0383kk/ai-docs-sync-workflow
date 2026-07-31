---
doc-schema-version: 1
read_when:
    - 你想在控制介面中瀏覽、安裝、啟用或停用外掛
    - 你想快速查看外掛清單、安裝、更新、檢查或解除安裝的範例
    - 你想要選擇外掛安裝來源
    - 你需要正確的外掛套件發布參考資料
sidebarTitle: Manage plugins
summary: 從控制介面或命令列介面管理 OpenClaw 外掛
title: 管理外掛
x-i18n:
    generated_at: "2026-07-26T08:41:30Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 9101d5c3630b618a043f1e71fdf5fa083698cc23694ccdc773d295a37c4c1ef3
    source_path: plugins/manage-plugins.md
    workflow: 16
---

控制介面涵蓋常見的探索、安裝、啟用與停用工作流程。命令列介面則增加更新、解除安裝、進階設定，以及明確的安裝來源控制。如需完整的命令契約、旗標、來源選擇規則與邊界情況，請參閱 [`openclaw plugins`](/zh-TW/cli/plugins)。

典型的命令列介面工作流程：尋找套件，從 ClawHub、npm、git 或本機路徑安裝，讓受管理的閘道自動重新啟動（或手動重新啟動），然後驗證外掛的執行階段註冊項目。

## 使用控制介面

在控制介面中開啟 **Plugins**，或使用相對於已設定控制介面基底路徑的 `/settings/plugins`。例如，基底路徑為 `/openclaw` 時會使用 `/openclaw/settings/plugins`。此頁面有兩個分頁：

- **Installed** 顯示按類別分組的完整本機清單（頻道、模型提供者、記憶、工具）。每一列都可開啟詳細資料檢視；其更多選項
  (`…`) 選單可啟用或停用外掛，並針對外部安裝的外掛提供 **Remove**。此分頁也會列出已設定的
  [MCP 伺服器](/zh-TW/cli/mcp)，並提供相同的選單式啟用、停用與移除操作，同時編輯閘道設定中的 `mcp.servers`。
- **Discover** 是外掛商店：包含 OpenClaw 隨附的精選外掛、官方外部外掛，以及經策展的連接器專區。連接器卡片可按一下就新增託管的 MCP 伺服器（GitHub、Notion、Linear、Sentry、
  Home Assistant），或前往已預填內容的 ClawHub 搜尋。在搜尋框中輸入內容時，會直接查詢 [ClawHub](https://clawhub.ai/plugins)，並附加包含下載次數與來源驗證徽章的 **From
  ClawHub** 區段。

隨附的外掛不需要安裝套件。其選單操作為 **Enable** 或 **Disable**。例如，Workboard 隨 OpenClaw 提供且預設停用，因此請選擇 **Enable** 將其開啟。無法移除隨附的外掛，只能將其停用。

存取目錄與搜尋需要 `operator.read`。安裝、啟用、停用、移除及變更 MCP 伺服器需要 `operator.admin`。ClawHub 安裝由閘道執行，並保留其信任、完整性及外掛安裝政策檢查。管理員啟用已安裝的外掛時，也會將所選外掛新增至現有的限制性 `plugins.allow` 清單，以記錄該明確信任。明確的 `plugins.deny` 項目仍具有決定權，必須先移除才能啟用該外掛。

安裝或移除外掛程式碼需要重新啟動閘道。當已安裝的外掛與目前閘道執行階段支援時，啟用狀態變更可不重新啟動就套用；否則介面會告知你需要重新啟動。以 OAuth 為基礎的 MCP 連接器新增後，仍需從命令列介面執行一次 `openclaw mcp login <name>`。

控制介面不會從任意 npm、git 或本機路徑來源安裝、不會更新外掛，也不會提供完整的外掛設定。這些操作請使用下方的命令列介面工作流程。

## 列出及搜尋外掛

```bash
openclaw plugins list
openclaw plugins list --enabled
openclaw plugins list --verbose
openclaw plugins list --json
openclaw plugins search "calendar"
```

供指令碼使用的 `--json`：

```bash
openclaw plugins list --json \
  | jq '.plugins[] | {id, enabled, format, source, dependencyStatus}'
```

`plugins list` 是冷清單檢查：檢查 OpenClaw 可從設定、資訊清單與持久化外掛登錄中探索到的項目。它無法證明已在執行的閘道已匯入外掛執行階段。JSON 輸出包含登錄診斷，以及每個外掛的 `dependencyStatus`（宣告的
`dependencies`/`optionalDependencies` 是否可在磁碟上解析）。

`plugins search` 會向 ClawHub 查詢可安裝的外掛套件，並為每個結果輸出安裝提示（`openclaw plugins install clawhub:<package>`）。

## 啟用及停用外掛

```bash
openclaw plugins enable <plugin-id>
openclaw plugins disable <plugin-id>
```

切換外掛的設定項目，而不會動到已安裝的檔案。部分隨附外掛（隨附的模型／語音提供者及隨附的瀏覽器外掛）預設啟用；其他外掛安裝後需要 `enable`。

## 安裝外掛

```bash
# 在 ClawHub 搜尋外掛套件。
openclaw plugins search "calendar"

# 從 ClawHub 安裝。
openclaw plugins install clawhub:<package>
openclaw plugins install clawhub:<package>@1.2.3
openclaw plugins install clawhub:<package>@beta

# 從 npm 安裝。
openclaw plugins install npm:<package>
openclaw plugins install npm:@scope/openclaw-plugin@1.2.3
openclaw plugins install npm:@openclaw/codex

# 從本機 npm-pack 成品安裝。
openclaw plugins install npm-pack:<path.tgz>

# 從 git 或本機開發簽出目錄安裝。
openclaw plugins install git:github.com/acme/openclaw-plugin@v1.0.0
openclaw plugins install ./my-plugin
openclaw plugins install --link ./my-plugin
```

在啟動切換期間，未加前綴的套件規格會從 npm 安裝；但若名稱符合隨附或官方外掛 ID，OpenClaw 會改用該本機／官方副本。使用 `clawhub:`、`npm:`、`git:` 或
`npm-pack:`，以確定性方式選擇來源。OpenClaw 的隨附與官方目錄套件和 ClawHub 套件一樣受信任。對於新的任意 npm、git、本機路徑／封存檔、`npm-pack:` 或市集來源，在你審查並信任來源後，非互動式安裝需要 `--force`。

`--force` 會在不提示的情況下確認非 ClawHub 來源，並視需要覆寫現有的安裝目標。若要例行升級受追蹤的 npm、ClawHub 或 hook-pack 安裝，請改用 `openclaw plugins update`。搭配
`--link` 時，`--force` 只會確認來源；不會複製或覆寫連結的目錄。

如果新安裝的外掛需要尚未提供的設定，OpenClaw 會記錄該安裝，但讓外掛保持停用。設定
`plugins.entries.<id>.config`，然後執行 `openclaw plugins enable <id>`。如果現有設定項目存在但無效，安裝會失敗且不會重寫該項目。

## 重新啟動及檢查

若執行中的受管理閘道已啟用設定重新載入，則在安裝、更新或解除安裝外掛程式碼後會自動重新啟動。如果閘道不受管理或已停用重新載入，請先自行重新啟動，再檢查即時執行階段介面：

```bash
openclaw gateway restart
openclaw plugins inspect <plugin-id> --runtime --json
```

`inspect --runtime` 會載入外掛模組，並證明它已註冊執行階段介面（工具、鉤子、服務、閘道方法、HTTP 路由、外掛擁有的命令列介面命令）。單獨使用 `inspect` 與 `list` 只會執行冷資訊清單／設定／登錄檢查。

## 更新外掛

```bash
openclaw plugins update <plugin-id>
openclaw plugins update <npm-package-or-spec>
openclaw plugins update --all
openclaw plugins update <plugin-id> --dry-run
```

傳入外掛 ID 會重複使用其受追蹤的安裝規格：儲存的 dist-tag（`@beta`）與明確釘選的版本會延續到後續的 `update <plugin-id>` 執行。

`openclaw plugins update --all` 是大量維護路徑。它仍會遵循一般受追蹤的安裝規格，但受信任的官方 OpenClaw 外掛記錄會同步至目前官方目錄目標，而不是持續釘選至過時的明確官方套件；當 `update.channel` 為
`beta` 時，該同步會優先採用 beta 發行系列。使用指定目標的
`update <plugin-id>`，可讓明確版本或帶標籤的官方規格維持不變。

若為 npm 安裝，請傳入明確的套件規格以切換受追蹤的記錄：

```bash
openclaw plugins update @scope/openclaw-plugin@beta
openclaw plugins update @scope/openclaw-plugin
```

若外掛先前釘選至明確版本或標籤，第二個命令會將其移回登錄的預設發行系列。

如需確切的備援與釘選規則，請參閱 [`openclaw plugins`](/zh-TW/cli/plugins#update)。

## 解除安裝外掛

```bash
openclaw plugins uninstall <plugin-id> --dry-run
openclaw plugins uninstall <plugin-id>
openclaw plugins uninstall <plugin-id> --keep-files
```

解除安裝會移除外掛的設定項目、持久化外掛索引記錄、允許／拒絕清單項目，以及適用時已連結的 `plugins.load.paths` 項目。除非你傳入
`--keep-files`，否則受管理的安裝目錄也會被移除。當解除安裝變更外掛來源時，執行中的受管理閘道會自動重新啟動。

在 Nix 模式（`OPENCLAW_NIX_MODE=1`）下，外掛的安裝、更新、解除安裝、啟用與停用功能全都會停用；請改在該安裝的 Nix 來源中管理這些選擇。

## 選擇來源

| 來源        | 適用時機                                                                    | 範例                                                           |
| ----------- | --------------------------------------------------------------------------- | -------------------------------------------------------------- |
| ClawHub     | 你需要 OpenClaw 原生探索、掃描摘要、版本與提示                              | `openclaw plugins install clawhub:<package>`                   |
| git         | 你需要儲存庫中的分支、標籤或提交                                            | `openclaw plugins install git:github.com/<owner>/<repo>@<ref>` |
| 本機路徑    | 你正在同一台機器上開發或測試外掛                                            | `openclaw plugins install --link ./my-plugin`                  |
| 市集        | 你正在安裝與 Claude 相容的市集外掛                                          | `openclaw plugins install <plugin> --marketplace <source>`     |
| npm pack    | 你要透過 npm 安裝語意驗證本機套件成品                                       | `openclaw plugins install npm-pack:<path.tgz>`                 |
| npmjs.com   | 你已發佈 JavaScript 套件，或需要 npm dist-tag／私有登錄                      | `openclaw plugins install npm:@acme/openclaw-plugin`           |

受管理的本機路徑安裝必須是外掛目錄或封存檔。請將獨立外掛檔案放在 `plugins.load.paths`，而不是使用 `plugins install` 安裝。

## 發佈外掛

ClawHub 是 OpenClaw 外掛的主要公開探索介面。如果你希望使用者在安裝前能找到外掛中繼資料、版本歷程、登錄掃描結果與安裝提示，請在該處發佈。

```bash
npm i -g clawhub
clawhub login
clawhub package publish your-org/your-plugin --dry-run
clawhub package publish your-org/your-plugin
clawhub package publish your-org/your-plugin@v1.0.0
```

原生 npm 外掛在發佈前必須提供外掛資訊清單（`openclaw.plugin.json`）及
`package.json` 中繼資料：

```json package.json
{
  "name": "@acme/openclaw-plugin",
  "version": "1.0.0",
  "type": "module",
  "openclaw": {
    "extensions": ["./dist/index.js"]
  }
}
```

```bash
npm publish --access public
openclaw plugins install npm:@acme/openclaw-plugin
openclaw plugins install npm:@acme/openclaw-plugin@beta
openclaw plugins install npm:@acme/openclaw-plugin@1.0.0
```

請使用以下頁面取得完整的發佈契約，而不要將本頁視為發佈參考：

- [ClawHub 發佈](/zh-TW/clawhub/publishing)說明擁有者、範圍、發行、審查、套件驗證與套件轉移。
- [建置外掛](/zh-TW/plugins/building-plugins)展示完整的外掛套件結構（包括 `openclaw.plugin.json`）與首次發佈工作流程。
- [外掛資訊清單](/zh-TW/plugins/manifest)定義原生外掛資訊清單欄位。

如果同一套件同時可從 ClawHub 與 npm 取得，請使用明確的
`clawhub:` 或 `npm:` 前綴來強制選用其中一個來源。

## 相關內容

- [外掛](/zh-TW/tools/plugin) - 安裝、設定、重新啟動及疑難排解
- [`openclaw plugins`](/zh-TW/cli/plugins) - 完整的命令列介面參考
- [社群外掛](/zh-TW/plugins/community) - 公開探索與 ClawHub 發布
- [ClawHub](/zh-TW/clawhub/cli) - 登錄檔命令列介面操作
- [建置外掛](/zh-TW/plugins/building-plugins) - 建立外掛套件
- [外掛資訊清單](/zh-TW/plugins/manifest) - 資訊清單與套件中繼資料
