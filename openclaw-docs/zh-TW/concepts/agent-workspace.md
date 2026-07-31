---
read_when:
    - 你需要說明代理程式工作區或其檔案配置
    - 你想要備份或移轉代理程式工作區
sidebarTitle: Agent workspace
summary: 代理程式工作區：位置、配置與備份策略
title: 代理程式工作區
x-i18n:
    generated_at: "2026-07-26T08:20:50Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: b58ead9079c3dda4bcaec3253f8d55e67e7e554d5c5b87ccfec6b08ec4ba038f
    source_path: concepts/agent-workspace.md
    workflow: 16
---

工作區是代理程式的家：檔案工具與工作區情境所使用的工作目錄。請將其保持私密，並視為記憶。

這與儲存設定、認證資訊和工作階段的 `~/.openclaw/` 分開。

<Warning>
工作區是**預設 cwd**，不是強制沙箱。工具會相對於工作區解析相對路徑，但除非啟用沙箱，否則絕對路徑仍可存取主機上的其他位置。若需要隔離，請使用 [`agents.defaults.sandbox`](/zh-TW/gateway/sandboxing)（及／或個別代理程式的沙箱設定）。

啟用沙箱且 `workspaceAccess` 不是 `"rw"` 時，工具會在 `~/.openclaw/sandboxes` 下的沙箱工作區內運作，而不是你的主機工作區。
</Warning>

## 預設位置

- 預設值：`~/.openclaw/workspace`
- 如果已設定 `OPENCLAW_PROFILE` 且其值不是 `"default"`，預設值會變成 `~/.openclaw/workspace-<profile>`。
- 設定 `OPENCLAW_WORKSPACE_DIR` 後，會覆寫上述兩者。
- 沒有明確工作區的非預設代理程式（`agents.entries.*`）會解析至 `<state-dir>/workspace-<agentId>`，而不是共用的預設工作區。

在 `~/.openclaw/openclaw.json` 中覆寫：

```json5
{
  agents: {
    defaults: {
      workspace: "~/.openclaw/workspace",
    },
  },
}
```

個別代理程式覆寫：`agents.entries.*.workspace`。

`openclaw onboard`、`openclaw configure` 或 `openclaw setup` 會建立工作區，並在啟動引導檔案不存在時植入這些檔案。

<Note>
沙箱植入副本只接受工作區內的一般檔案；解析至來源工作區外部的符號連結／硬連結別名會被忽略。
</Note>

如果你已自行管理工作區檔案，請停用啟動引導檔案建立功能：

```json5
{ agents: { defaults: { skipBootstrap: true } } }
```

## 額外工作區資料夾

較舊的安裝可能建立了 `~/openclaw`。保留多個工作區目錄可能造成令人困惑的驗證或狀態偏移，因為同一時間只會啟用一個工作區。

<Note>
**建議：**只保留一個使用中的工作區。如果你已不再使用額外資料夾，請將其封存或移至垃圾桶（例如 `trash ~/openclaw`）。如果你刻意保留多個工作區，請確保 `agents.defaults.workspace`（或個別代理程式的 `workspace` 鍵）指向使用中的工作區。
</Note>

## 工作區檔案對照表

OpenClaw 預期工作區內包含的標準檔案：

<AccordionGroup>
  <Accordion title="AGENTS.md - 操作指示">
    代理程式的操作指示，以及應如何使用記憶。每次工作階段開始時載入。適合放置規則、優先順序和「應如何行事」等細節。
  </Accordion>
  <Accordion title="SOUL.md - 人格與語氣">
    人格、語氣和界限。每次工作階段都會載入。指南：[SOUL.md 人格指南](/zh-TW/concepts/soul)。
  </Accordion>
  <Accordion title="USER.md - 使用者身分">
    使用者是誰，以及應如何稱呼。每次工作階段都會載入。
  </Accordion>
  <Accordion title="IDENTITY.md - 名稱、風格、表情符號">
    代理程式的名稱、風格和表情符號。在啟動引導儀式期間建立／更新。
  </Accordion>
  <Accordion title="TOOLS.md - 本機工具慣例">
    關於本機工具和慣例的備註。這不會控制工具的可用性；僅供指引。
  </Accordion>
  <Accordion title="HEARTBEAT.md - 心跳偵測檢查清單">
    心跳偵測執行的選用精簡檢查清單。請保持簡短，以免消耗權杖。
  </Accordion>
  <Accordion title="BOOT.md - 啟動檢查清單">
    閘道重新啟動時自動執行的選用啟動檢查清單（需啟用[內部鉤子](/zh-TW/automation/hooks)）。請保持簡短；對外傳送請使用訊息工具。
  </Accordion>
  <Accordion title="BOOTSTRAP.md - 首次執行儀式">
    僅執行一次的首次執行儀式。只會為全新的工作區建立。儀式完成後請將其刪除。
  </Accordion>
  <Accordion title="memory/YYYY-MM-DD.md - 每日記憶日誌">
    每日記憶日誌（每天一個檔案）。建議在工作階段開始時讀取今天與昨天的日誌。
  </Accordion>
  <Accordion title="MEMORY.md - 整理過的長期記憶（選用）">
    整理過的長期記憶：持久的事實、偏好、決策和簡短摘要。將詳細日誌保留在 `memory/YYYY-MM-DD.md`，讓記憶工具可按需擷取，而不必將其注入每個提示。只在主要的私人工作階段載入 `MEMORY.md`（不要在共用／群組情境中載入）。工作流程和自動記憶清除請參閱[記憶](/zh-TW/concepts/memory)。
  </Accordion>
  <Accordion title="skills/ - 工作區 Skills（選用）">
    工作區專用 Skills。當名稱衝突時，這是該工作區優先順序最高的 Skills 位置，優先於專案代理程式 Skills、個人代理程式 Skills、受管理的 Skills、隨附 Skills 和 `skills.load.extraDirs`。
  </Accordion>
  <Accordion title="canvas/ - Canvas UI 檔案（選用）">
    用於節點顯示的 Canvas UI 檔案（例如 `canvas/index.html`）。
  </Accordion>
</AccordionGroup>

<Note>
如果缺少啟動引導檔案，OpenClaw 會將「缺少檔案」標記注入工作階段並繼續執行。大型啟動引導檔案在注入時會被截斷；可使用 `agents.defaults.bootstrapMaxChars`（預設值：`20000`）和 `agents.defaults.bootstrapTotalMaxChars`（預設值：`60000`）調整限制。`openclaw setup` 可重新建立缺少的預設檔案，而不會覆寫現有檔案。
</Note>

## 工作區中「不」包含的內容

以下內容位於 `~/.openclaw/` 下，且「不應」提交至工作區儲存庫：

- `~/.openclaw/openclaw.json`（設定）
- `~/.openclaw/state/openclaw.sqlite`（共用工作區設定狀態和驗證聲明）
- `~/.openclaw/agents/<agentId>/agent/auth-profiles.json`（模型驗證設定檔：OAuth + API 金鑰）
- `~/.openclaw/agents/<agentId>/agent/openclaw-agent.sqlite`（工作階段資料列、逐字記錄和個別代理程式執行階段狀態）
- `~/.openclaw/agents/<agentId>/agent/codex-home/`（個別代理程式的 Codex 執行階段帳號、設定、Skills、外掛和原生對話串狀態）
- `~/.openclaw/credentials/`（頻道／提供者狀態及舊版 OAuth 匯入資料）
- `~/.openclaw/agents/<agentId>/sessions/`（舊版遷移來源和封存／支援成品）
- `~/.openclaw/skills/`（受管理的 Skills）

如果需要遷移工作階段或設定，請分別複製，並避免將其納入版本控制。

較舊的 OpenClaw 版本會寫入 `openclaw-workspace-state.json`、
`.openclaw/workspace-state.json` 和 `.attested` 工作區附屬檔案。目前的
執行階段只會使用共用 SQLite 資料庫儲存該狀態。如果診斷工具回報
其中一個檔案，請執行 `openclaw doctor --fix`；診斷工具會匯入有效的舊版
狀態，且只會在驗證資料庫資料列後刪除來源。

## Git 備份（建議使用私人儲存庫）

將工作區視為私人記憶。請將其放入**私人** Git 儲存庫，以便備份與復原。

請在執行閘道的機器上執行以下步驟（工作區位於該處）。

<Steps>
  <Step title="初始化儲存庫">
    如果已安裝 Git，全新的工作區會自動初始化。如果此工作區尚未成為儲存庫，請執行：

    ```bash
    cd ~/.openclaw/workspace
    git init
    git add AGENTS.md SOUL.md TOOLS.md IDENTITY.md USER.md HEARTBEAT.md memory/
    git commit -m "Add agent workspace"
    ```

  </Step>
  <Step title="新增私人遠端儲存庫">
    <Tabs>
      <Tab title="GitHub 網頁介面">
        1. 在 GitHub 建立新的**私人**儲存庫。
        2. 不要使用 README 初始化（以避免合併衝突）。
        3. 複製 HTTPS 遠端 URL。
        4. 新增遠端並推送：

        ```bash
        git branch -M main
        git remote add origin <https-url>
        git push -u origin main
        ```
      </Tab>
      <Tab title="GitHub 命令列介面 (gh)">
        ```bash
        gh auth login
        gh repo create openclaw-workspace --private --source . --remote origin --push
        ```
      </Tab>
      <Tab title="GitLab 網頁介面">
        1. 在 GitLab 建立新的**私人**儲存庫。
        2. 不要使用 README 初始化（以避免合併衝突）。
        3. 複製 HTTPS 遠端 URL。
        4. 新增遠端並推送：

        ```bash
        git branch -M main
        git remote add origin <https-url>
        git push -u origin main
        ```
      </Tab>
    </Tabs>

  </Step>
  <Step title="持續更新">
    ```bash
    git status
    git add .
    git commit -m "Update memory"
    git push
    ```
  </Step>
</Steps>

## 不要提交祕密

<Warning>
即使是私人儲存庫，也應避免在工作區中儲存祕密：

- API 金鑰、OAuth 權杖、密碼或私人認證資訊。
- `~/.openclaw/` 下的任何內容。
- 聊天的原始傾印或敏感附件。

如果必須儲存敏感參照，請使用預留位置，並將真正的祕密保存在其他位置（密碼管理器、環境變數或 `~/.openclaw/`）。
</Warning>

建議的 `.gitignore` 起始內容：

```gitignore
.DS_Store
.env
**/*.key
**/*.pem
**/secrets*
```

## 將工作區移至新機器

<Steps>
  <Step title="複製儲存庫">
    將儲存庫複製至所需路徑（預設為 `~/.openclaw/workspace`）。
  </Step>
  <Step title="更新設定">
    在 `~/.openclaw/openclaw.json` 中將 `agents.defaults.workspace` 設為該路徑。
  </Step>
  <Step title="植入缺少的檔案">
    執行 `openclaw setup --workspace <path>` 以植入任何缺少的檔案。
  </Step>
  <Step title="複製工作階段（選用）">
    如果需要工作階段，請另外從舊機器複製 `~/.openclaw/agents/<agentId>/agent/openclaw-agent.sqlite`。
    只有在也需要舊版遷移輸入或封存／支援成品時，才複製 `~/.openclaw/agents/<agentId>/sessions/`。
  </Step>
</Steps>

## 進階備註

- 多代理程式路由可透過 `agents.entries.*.workspace` 為每個代理程式使用不同的工作區。路由設定請參閱[頻道路由](/zh-TW/channels/channel-routing)。
- 如果已啟用 `agents.defaults.sandbox`，非主要工作階段可使用 `agents.defaults.sandbox.workspaceRoot` 下的個別工作階段沙箱工作區。

## 相關內容

- [心跳偵測](/zh-TW/gateway/heartbeat) - HEARTBEAT.md 工作區檔案
- [沙箱](/zh-TW/gateway/sandboxing) - 沙箱環境中的工作區存取
- [工作階段](/zh-TW/concepts/session) - 工作階段儲存路徑
- [常駐指令](/zh-TW/automation/standing-orders) - 工作區檔案中的持續性指示
