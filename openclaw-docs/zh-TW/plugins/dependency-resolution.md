---
read_when:
    - 你正在偵錯外掛套件安裝問題
    - 你正在變更外掛啟動、doctor 或套件管理器安裝行為
    - 你正在維護已封裝的 OpenClaw 安裝版本或內建外掛資訊清單
sidebarTitle: Dependencies
summary: OpenClaw 如何安裝外掛套件並解析外掛相依性
title: 外掛相依性解析
x-i18n:
    generated_at: "2026-07-26T07:25:14Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: ae24a82568e275399cb7b68729d2805956792852612f84d6918850305f0eb243
    source_path: plugins/dependency-resolution.md
    workflow: 16
---

OpenClaw 僅在安裝／更新時處理外掛相依套件。執行階段
載入絕不會執行套件管理器、修復相依性樹狀結構，或修改
OpenClaw 套件目錄。

## 職責劃分

外掛套件自行負責其相依性圖：

- 執行階段相依套件位於外掛套件的 `dependencies` 或
  `optionalDependencies` 中。
- SDK／核心匯入項目是對等相依套件，或由 OpenClaw 提供的匯入項目。
- 本機開發外掛自行提供已安裝的相依套件。
- npm 與 git 外掛會安裝至 OpenClaw 所管理的套件根目錄。

OpenClaw 僅負責外掛生命週期：

- 探索外掛來源。
- 僅在明確要求時安裝或更新套件。
- 記錄安裝中繼資料。
- 載入外掛進入點。
- 相依套件缺少時，以可採取行動的錯誤訊息結束。

## 安裝根目錄

OpenClaw 針對每個來源使用穩定的根目錄：

- npm 套件會安裝至
  `~/.openclaw/npm/projects/<encoded-package>` 下各外掛專用的專案中。
- git 套件會複製至 `~/.openclaw/git` 下。
- 本機／路徑／封存檔安裝會直接複製或參照，不會修復相依套件。

npm 安裝會在該外掛專用專案根目錄中執行：

```bash
cd ~/.openclaw/npm/projects/<encoded-package>
npm install --omit=dev --omit=peer --legacy-peer-deps --ignore-scripts --no-audit --no-fund
```

`openclaw plugins install npm-pack:<path.tgz>` 會針對本機 npm-pack tarball 使用相同的外掛專用 npm
專案根目錄：OpenClaw 會讀取 tarball 的 npm
中繼資料，將其作為複製的 `file:` 相依套件加入受管理專案，執行
上述一般 npm 安裝，接著驗證已安裝的鎖定檔中繼資料，
確認無誤後才信任該外掛。此路徑用於套件驗收及
候選版本驗證，其中本機 pack 成品的行為應與它所模擬的
登錄檔成品相同。

發布前測試官方或外部外掛套件時，請使用 `npm-pack:`。
原始封存檔或路徑安裝適合用於本機偵錯，但
無法證明其相依套件路徑與已安裝的 npm 或 ClawHub
套件相同。`npm-pack:` 可證明受管理套件的安裝形式；但其本身
不足以證明該外掛是與目錄連結的官方內容。

當行為取決於隨附外掛或受信任官方外掛狀態時，
請將本機套件驗證搭配由目錄支援的官方安裝，或
會記錄官方信任狀態的已發布套件路徑。具權限的輔助工具存取
與受信任官方範圍處理，應在該受信任的安裝
路徑上驗證，不應從本機 tarball 安裝推斷。

如果外掛在執行階段因匯入項目缺少而失敗，請修正套件資訊清單，
而不要手動修復受管理專案。執行階段匯入項目應位於
外掛套件的 `dependencies` 或 `optionalDependencies` 中；受管理的執行階段專案
不會安裝 `devDependencies`。`~/.openclaw/npm/projects/<encoded-package>` 內的本機 `npm install`
可暫時解除診斷阻礙，
但不能作為套件驗收證明，因為下次安裝或
更新時，會依據套件中繼資料重新建立專案。

npm 可能會將遞移相依套件提升至外掛套件旁的
外掛專用專案 `node_modules`。OpenClaw 會先掃描受管理專案
根目錄，再信任該安裝，並在解除安裝時移除該專案，因此
提升的執行階段相依套件仍會位於該外掛的清理邊界內。

已發布的 npm 外掛套件可以隨附 `npm-shrinkwrap.json`；npm 在安裝期間會使用該
可發布的鎖定檔，而 OpenClaw 的受管理 npm 專案根目錄
會透過一般安裝路徑支援它。OpenClaw 所管理的可發布
外掛套件必須包含根據該套件已發布相依性圖產生的套件本機 shrinkwrap：

```bash
pnpm deps:shrinkwrap:generate
pnpm deps:shrinkwrap:check
```

產生器會移除外掛的 `devDependencies`、套用工作區覆寫
政策，並為每個具有
`openclaw.release.publishToNpm: true` 的外掛寫入 `extensions/<id>/npm-shrinkwrap.json`。第三方外掛套件也可以
隨附 shrinkwrap；OpenClaw 不要求社群套件必須提供，但
若有提供，npm 會遵循它。

將本機套件視為候選版本證明前，請檢查
將要安裝的 tarball：

```bash
npm pack --pack-destination /tmp
tar -xOf /tmp/<plugin-package>.tgz package/package.json
tar -tf /tmp/<plugin-package>.tgz | grep '^package/dist/'
```

若有變更相依套件，也請驗證正式環境安裝能否在
沒有開發相依套件的情況下解析執行階段套件：

```bash
tmpdir=$(mktemp -d)
(
  cd "$tmpdir"
  npm init -y >/dev/null
  npm install --package-lock-only --omit=dev --omit=peer --legacy-peer-deps --ignore-scripts /tmp/<plugin-package>.tgz
)
rm -rf "$tmpdir"
```

OpenClaw 所管理的 npm 外掛套件也可以使用明確的
`bundledDependencies` 進行發布。npm 發布路徑會覆疊執行階段相依套件
名稱清單、從已發布的資訊清單中移除僅供開發使用的工作區中繼資料、
針對套件本機的執行階段相依套件執行不含指令碼的 npm 安裝，
接著將包含這些相依套件檔案的外掛 tarball
封裝或發布。大量使用原生元件的套件（Codex、ACPX、Copilot、llama.cpp、
memory-lancedb、Tlon）會透過
`openclaw.release.bundleRuntimeDependencies: false` 選擇不採用此方式；它們仍會隨附
shrinkwrap，但 npm 會在安裝期間解析執行階段相依套件，而不是
將每個平台的二進位檔都嵌入外掛 tarball。根 `openclaw`
套件不會綑綁其完整的相依性樹狀結構。

匯入 `openclaw/plugin-sdk/*` 的外掛會將 `openclaw` 宣告為對等
相依套件。OpenClaw 不允許 npm 將獨立的主機套件登錄檔副本
安裝至受管理專案，因為過時的主機套件可能會影響
npm 在該外掛內的對等相依套件解析。受管理的 npm 安裝會略過 npm 對等相依套件
解析／具體化，且在安裝或更新後，OpenClaw 會針對
宣告主機對等相依套件的已安裝套件，重新建立外掛本機的
`node_modules/openclaw` 連結。

git 安裝會複製或重新整理儲存庫，接著執行：

```bash
npm install --omit=dev --ignore-scripts --no-audit --no-fund
```

接著，已安裝的外掛會從該套件目錄載入，因此
套件本機與父層的 `node_modules` 解析方式
與一般 Node 套件相同。

## 本機外掛

本機外掛是由開發者控制的目錄。OpenClaw 絕不會為其執行
`npm install`、`pnpm install` 或相依套件修復；如果本機
外掛有相依套件，請先在該外掛中安裝它們，再載入外掛。

第三方 TypeScript 本機外掛會透過 Jiti 載入，作為緊急備援路徑。
已封裝的 JavaScript 外掛與隨附的內部外掛則透過原生
import／require 載入。

## 啟動與重新載入

閘道啟動與設定重新載入絕不會安裝外掛相依套件。它們會
讀取外掛安裝記錄、計算進入點，並載入該進入點。

執行階段缺少相依套件時，外掛載入會失敗，並顯示
引導操作人員採取明確修正方式的錯誤：

```bash
openclaw plugins update <id>
openclaw plugins install <source>
openclaw doctor --fix
```

`doctor --fix` 會清理舊版 OpenClaw 產生的相依套件狀態；當
設定仍參照可下載外掛，但本機安裝記錄中缺少該外掛時，
也可將其復原。Doctor 不會修復
已安裝本機外掛的相依套件。

## 隨附外掛

輕量且對核心至關重要的隨附外掛會作為 OpenClaw 的一部分提供。它們
不應包含龐大的執行階段相依性樹狀結構，否則應移至
ClawHub／npm 上的可下載套件。

如需目前產生的外掛清單，其中列出哪些外掛隨核心套件提供、
由外部安裝，或僅保留於原始碼中，請參閱
[外掛清單](/zh-TW/plugins/plugin-inventory)。

隨附外掛的資訊清單不得要求相依套件暫存。大型或
選用的外掛功能應封裝為一般外掛，並
透過與第三方外掛相同的 npm／git／ClawHub 路徑安裝。

在原始碼簽出中，OpenClaw 會將儲存庫視為 pnpm 單一儲存庫。
執行 `pnpm install` 後，隨附外掛會從 `extensions/<id>` 載入，因此
套件本機的工作區相依套件可供使用，且編輯內容會直接生效。
原始碼簽出開發僅支援 pnpm；在儲存庫根目錄執行一般的 `npm install`
不會準備隨附外掛的相依套件。

| 安裝形式                    | 隨附外掛位置               | 相依套件負責方                                                     |
| -------------------------------- | ------------------------------------- | -------------------------------------------------------------------- |
| `npm install -g openclaw`        | 套件內的已建置執行階段樹狀結構 | OpenClaw 套件與明確的外掛安裝／更新／Doctor 流程     |
| Git 簽出加上 `pnpm install` | `extensions/<id>` 工作區套件  | pnpm 工作區，包括每個外掛套件本身的相依套件 |
| `openclaw plugins install ...`   | 受管理的 npm 專案／git／ClawHub 根目錄  | 外掛安裝／更新流程                                       |

## 舊版清理

舊版 OpenClaw 會在啟動時或 Doctor 修復期間，產生隨附外掛的
相依套件根目錄。目前的 Doctor 清理會使用 `--fix` 移除這些過時的
目錄與符號連結，包括舊的 `plugin-runtime-deps`
根目錄、指向已刪減 `plugin-runtime-deps` 目標的全域 Node 前綴套件符號連結、
`.openclaw-runtime-deps*` 資訊清單、產生的外掛 `node_modules`、
安裝暫存目錄，以及套件本機的 pnpm
儲存區。已封裝的 postinstall 也會先移除這些全域符號連結，再
刪減舊版目標根目錄，因此升級不會留下失效的 ESM
套件匯入。

舊版 npm 安裝也曾使用共用的 `~/.openclaw/npm/node_modules` 根目錄。
目前的安裝、更新、解除安裝與 Doctor 流程仍會辨識該
舊版扁平根目錄，但僅用於復原與清理。新的 npm 安裝會改為建立
各外掛專用的專案根目錄。
