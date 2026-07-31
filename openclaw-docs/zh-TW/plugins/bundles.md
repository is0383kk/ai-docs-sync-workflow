---
read_when:
    - 你想安裝與 Codex、Claude 或 Cursor 相容的套件組合
    - 你需要瞭解 OpenClaw 如何將套件組合內容對應至原生功能
    - 你正在偵錯套件組合偵測或功能缺失問題
summary: 安裝並使用 Codex、Claude 與 Cursor 套件組，作為 OpenClaw 外掛
title: 外掛套件組合
x-i18n:
    generated_at: "2026-07-26T07:56:56Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: d44006866238f53ee2e3e8126cc4f7ed6f7413534257775f7904c9b877778c59
    source_path: plugins/bundles.md
    workflow: 16
---

OpenClaw 可以從三個外部生態系統安裝外掛：**Codex**、**Claude**
和 **Cursor**。這些稱為 **套件組合**，是由內容與中繼資料組成的套件，
OpenClaw 會將其對應至 Skills、鉤子和 MCP 工具等原生功能。

<Info>
  套件組合與 OpenClaw 原生外掛**不同**。原生外掛會在處理程序內執行，
  並可註冊任何能力。套件組合則是內容套件，只會選擇性地對應功能，
  且信任邊界較窄。
</Info>

## 為何需要套件組合

許多實用的外掛是以 Codex、Claude 或 Cursor 格式發布。OpenClaw 不要求
作者將它們重寫為原生 OpenClaw 外掛，而是偵測這些格式，並將其支援的內容
對應至原生功能集。你可以安裝 Claude 命令套件或 Codex Skill 套件組合，
並立即使用。

## 安裝套件組合

<Steps>
  <Step title="從目錄、封存檔或市集安裝">
    ```bash
    # 本機目錄
    openclaw plugins install ./my-bundle

    # 封存檔
    openclaw plugins install ./my-bundle.tgz

    # Claude 市集
    openclaw plugins marketplace list <source>
    openclaw plugins install <plugin> --marketplace <source>
    ```

    `<source>` 是本機市集路徑／儲存庫，或 git/GitHub 來源。

  </Step>

  <Step title="驗證偵測結果">
    ```bash
    openclaw plugins list
    openclaw plugins inspect <id>
    ```

    套件組合會顯示 `Format: bundle`，以及值為 `codex`、
    `claude` 或 `cursor` 的 `Bundle format:`。

  </Step>

  <Step title="重新啟動並使用">
    ```bash
    openclaw gateway restart
    ```

    對應後的功能（Skills、鉤子、MCP 工具、LSP 預設值）會在下一個工作階段中可用。

  </Step>
</Steps>

## OpenClaw 會從套件組合對應哪些內容

目前並非所有套件組合功能都能在 OpenClaw 中執行。以下列出已可運作的功能，
以及已偵測但尚未接通的功能。

### 目前支援

| 功能          | 對應方式                                                                                          | 適用格式       |
| ------------- | ------------------------------------------------------------------------------------------------- | -------------- |
| Skill 內容    | 套件組合的 Skill 根目錄會載入為一般 OpenClaw Skills                                               | 所有格式       |
| 命令          | 將 `commands/` 和 `.cursor/commands/` 視為 Skill 根目錄                                     | Claude、Cursor |
| 鉤子套件      | OpenClaw 樣式的 `HOOK.md` + `handler.ts` 配置                                      | Codex          |
| MCP 工具      | 將套件組合的 MCP 設定合併至內嵌 OpenClaw 設定；載入支援的 stdio 和 HTTP 伺服器                    | 所有格式       |
| LSP 伺服器    | 將 Claude `.lsp.json` 和資訊清單宣告的 `lspServers` 合併至內嵌 OpenClaw LSP 預設值   | Claude         |
| 設定          | 將 Claude `settings.json` 匯入為內嵌 OpenClaw 預設值                                           | Claude         |

#### Skill 內容

- 套件組合的 Skill 根目錄會載入為一般 OpenClaw Skill 根目錄。
- Claude `commands/` 根目錄會視為額外的 Skill 根目錄。
- Cursor `.cursor/commands/` 根目錄會視為額外的 Skill 根目錄。

Claude Markdown 命令檔和 Cursor 命令 Markdown 都能透過一般的
OpenClaw Skill 載入器運作。

#### 鉤子套件

套件組合鉤子根目錄**僅**在使用一般 OpenClaw 鉤子套件配置時才會運作：
`HOOK.md` 加上 `handler.ts` 或 `handler.js`。目前主要適用於
與 Codex 相容的情況。

#### 內嵌 OpenClaw 的 MCP

- 已啟用的套件組合可以提供 MCP 伺服器設定。
- OpenClaw 會將套件組合的 MCP 設定合併至有效的內嵌 OpenClaw
  設定中，作為 `mcpServers`。
- OpenClaw 會在內嵌 OpenClaw 代理程式回合中公開支援的套件組合 MCP 工具，
  方法是啟動 stdio 伺服器或連線至 HTTP 伺服器。
- `coding` 和 `messaging` 工具設定檔預設包含套件組合 MCP 工具；
  可使用 `tools.deny: ["bundle-mcp"]` 讓代理程式或閘道選擇不使用。
- 套用套件組合預設值後，專案本機的內嵌代理程式設定仍會生效，因此
  必要時工作區設定可以覆寫套件組合的 MCP 項目。
- 套件組合 MCP 工具目錄會在註冊前以確定性方式排序，因此
  上游 `listTools()` 的順序變更不會造成提示快取工具區塊反覆變動。

##### 傳輸方式

MCP 伺服器可以使用 stdio 或 HTTP 傳輸。

**Stdio** 會啟動子處理程序：

```json
{
  "mcp": {
    "servers": {
      "my-server": {
        "command": "node",
        "args": ["server.js"],
        "env": { "PORT": "3000" }
      }
    }
  }
}
```

**HTTP** 會連線至正在執行的 MCP 伺服器；除非要求
`streamable-http`，否則預設為 `sse`：

```json
{
  "mcp": {
    "servers": {
      "my-server": {
        "url": "http://localhost:3100/mcp",
        "transport": "streamable-http",
        "headers": {
          "Authorization": "Bearer ${MY_SECRET_TOKEN}"
        },
        "connectionTimeoutMs": 30000
      }
    }
  }
}
```

- `transport` 接受 `"streamable-http"` 或 `"sse"`；省略時預設為 `sse`。
- `type: "http"` 是命令列介面原生的下游格式；在 OpenClaw 設定中請使用 `transport: "streamable-http"`。`openclaw mcp set` 和 `openclaw doctor --fix` 會正規化常見別名。
- 僅允許 `http:` 和 `https:` URL 配置。
- `headers` 值支援 `${ENV_VAR}` 插值。
- 同時包含 `command` 和 `url` 的伺服器項目會遭拒絕。
- 工具說明和記錄中的 URL 認證資訊（使用者資訊和查詢參數）會經過遮蔽。
- `connectionTimeoutMs` 會覆寫 stdio 和 HTTP 傳輸預設的 30 秒連線逾時。
  請求逾時預設為 60 秒，並可使用 `requestTimeoutMs` 覆寫。

##### 工具命名

OpenClaw 會使用 `serverName__toolName` 格式的供應商安全名稱來註冊套件組合 MCP 工具。
例如，索引鍵為 `"vigil-harbor"` 的伺服器若公開 `memory_search` 工具，
會註冊為 `vigil-harbor__memory_search`。

- `A-Za-z0-9_-` 以外的字元會替換為 `-`。
- 會以非字母開頭的片段會加上字母前綴，因此像 `12306` 這類數字
  伺服器索引鍵會轉換為供應商安全的工具前綴。
- 伺服器前綴上限為 30 個字元。
- 完整工具名稱上限為 64 個字元。
- 空白伺服器名稱會退回使用 `mcp`。
- 清理後發生衝突的名稱會以數字後綴加以區別。
- 最終公開的工具會依安全名稱進行確定性排序，讓重複的
  內嵌代理程式回合維持快取穩定。
- 設定檔篩選會將同一套件組合 MCP 伺服器的所有工具視為
  由 `bundle-mcp` 外掛擁有，因此設定檔允許／拒絕清單可以參照
  個別公開的工具名稱或 `bundle-mcp` 外掛索引鍵。

#### 內嵌 OpenClaw 設定

啟用套件組合時，Claude `settings.json` 會匯入為預設的內嵌 OpenClaw 設定。
OpenClaw 會先清理 shell 覆寫索引鍵再套用：

- `shellPath`
- `shellCommandPrefix`

#### 內嵌 OpenClaw LSP

- 已啟用的 Claude 套件組合可以提供 LSP 伺服器設定。
- OpenClaw 會載入 `.lsp.json`，以及任何資訊清單宣告的 `lspServers` 路徑。
- 套件組合 LSP 設定會合併至有效的內嵌 OpenClaw LSP 預設值。
- 目前僅能執行支援且以 stdio 為基礎的 LSP 伺服器；不支援的
  傳輸仍會顯示於 `openclaw plugins inspect <id>` 中。

### 已偵測但未執行

下列項目會被辨識並顯示於診斷資訊中，但 OpenClaw 不會執行：

- Claude `agents`、`hooks/hooks.json` 自動化、`outputStyles`
- Cursor `.cursor/agents`、`.cursor/hooks.json`、`.cursor/rules`
- Codex `.app.json` 中能力報告以外的中繼資料

## 套件組合格式

<AccordionGroup>
  <Accordion title="Codex 套件組合">
    標記：`.codex-plugin/plugin.json`

    選用內容：`skills/`、`hooks/`、`.mcp.json`、`.app.json`

    當 Codex 套件組合使用 Skill 根目錄及 OpenClaw 樣式的
    鉤子套件目錄（`HOOK.md` + `handler.ts`）時，最適合搭配 OpenClaw 使用。

  </Accordion>

  <Accordion title="Claude 套件組合">
    有兩種偵測模式：

    - **以資訊清單為基礎：** `.claude-plugin/plugin.json`
    - **無資訊清單：** 預設 Claude 配置（`skills/`、`commands/`、`agents/`、`hooks/`、`.mcp.json`、`.lsp.json`、`settings.json`）

    Claude 特有行為：

    - `commands/` 會視為 Skill 內容
    - `settings.json` 會匯入內嵌 OpenClaw 設定中（shell 覆寫索引鍵會經過清理）
    - `.mcp.json` 會向內嵌 OpenClaw 公開支援的 stdio 工具
    - `.lsp.json` 加上資訊清單宣告的 `lspServers` 路徑會載入內嵌 OpenClaw LSP 預設值
    - `hooks/hooks.json` 會被偵測但不執行
    - 資訊清單中的自訂元件路徑採累加方式；它們會擴充而非取代預設值

  </Accordion>

  <Accordion title="Cursor 套件組合">
    標記：`.cursor-plugin/plugin.json`

    選用內容：`skills/`、`.cursor/commands/`、`.cursor/agents/`、`.cursor/rules/`、`.cursor/hooks.json`、`.mcp.json`

    - `.cursor/commands/` 會視為 Skill 內容
    - `.cursor/rules/`、`.cursor/agents/` 和 `.cursor/hooks.json` 僅供偵測

  </Accordion>
</AccordionGroup>

## 偵測優先順序

OpenClaw 會先檢查原生外掛格式：

1. `openclaw.plugin.json`，或具有 `openclaw.extensions` 的有效 `package.json`，會視為**原生外掛**
2. 套件組合標記（`.codex-plugin/`、`.claude-plugin/`，或預設 Claude/Cursor 配置），會視為**套件組合**

如果目錄同時包含兩種格式，OpenClaw 會使用原生路徑。這可防止
雙格式套件以套件組合形式被部分安裝。

## 執行階段相依性與清理

- 第三方相容套件組合在啟動時不會獲得 `npm install` 修復。
  它們應透過 `openclaw plugins install` 安裝，並將所需的一切
  隨附於已安裝的外掛目錄中。
- OpenClaw 擁有的隨附外掛，若非隨核心以輕量形式提供，
  就可透過外掛安裝程式下載。閘道啟動時絕不會為它們執行套件管理器。
- `openclaw doctor --fix` 會移除過期的本機隨附外掛安裝記錄，
  並且當設定仍參照可下載外掛，但本機外掛索引中缺少該外掛時，可將其復原。

## 安全性

套件組合的信任邊界比原生外掛更窄：

- OpenClaw **不會**在處理程序內載入任意套件組合執行階段模組。
- Skills 和鉤子套件路徑必須保持在外掛根目錄內（會檢查邊界）。
- 讀取設定檔時會執行相同的邊界檢查。
- 支援的 stdio MCP 伺服器可能會以子處理程序啟動。

這使套件組合預設更安全，但你仍應就其公開的功能，
將第三方套件組合視為受信任內容。

## 疑難排解

<AccordionGroup>
  <Accordion title="已偵測到套件，但功能無法執行">
    執行 `openclaw plugins inspect <id>`。如果某項功能已列出但標示為
    尚未接線，這是產品限制，而非安裝損壞。
  </Accordion>

  <Accordion title="Claude 命令檔案未出現">
    請確認套件已啟用，且 Markdown 檔案位於偵測到的
    `commands/` 或 `skills/` 根目錄內。
  </Accordion>

  <Accordion title="Claude 設定未生效">
    僅支援 `settings.json` 中內嵌的 OpenClaw 設定。OpenClaw
    不會將套件設定視為原始設定修補。
  </Accordion>

  <Accordion title="Claude 鉤子未執行">
    `hooks/hooks.json` 僅供偵測。如果需要可執行的鉤子，請使用
    OpenClaw 鉤子套件版面配置，或發布原生外掛。
  </Accordion>
</AccordionGroup>

## 相關內容

- [安裝與設定外掛](/zh-TW/tools/plugin)
- [建置外掛](/zh-TW/plugins/building-plugins) - 建立原生外掛
- [外掛資訊清單](/zh-TW/plugins/manifest) - 原生資訊清單結構描述
