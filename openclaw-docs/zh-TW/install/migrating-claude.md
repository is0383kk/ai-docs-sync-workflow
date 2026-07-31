---
read_when:
    - 你正在從 Claude Code 或 Claude Desktop 轉移而來，並希望保留指示、MCP 伺服器和 Skills
    - 你需要了解 OpenClaw 會自動匯入哪些內容，以及哪些內容只會保留在封存中
summary: 透過可預覽的匯入，將 Claude Code 和 Claude Desktop 的本機狀態移至 OpenClaw
title: 從 Claude 遷移
x-i18n:
    generated_at: "2026-07-26T07:54:42Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 0d5a5e63727e1583fc3fa27ac45215c72df9074b21d7c5f6b33800bec916769b
    source_path: install/migrating-claude.md
    workflow: 16
---

OpenClaw 透過內建的 Claude 遷移提供者匯入本機 Claude 狀態。提供者會在變更狀態前預覽每個項目，並在計畫與報告中遮蔽密鑰。獨立執行 `openclaw migrate` 會建立經驗證的備份；全新設定精靈流程會暫存匯入內容，且僅在驗證成功後才發布。

<Note>
透過設定精靈匯入需要全新的 OpenClaw 設定。如果已有本機 OpenClaw 狀態，請先重設設定、認證資訊、工作階段與工作區；或者在檢閱計畫後，直接搭配 `--overwrite` 使用 `openclaw migrate`。
</Note>

## 兩種匯入方式

<Tabs>
  <Tab title="設定精靈">
    精靈偵測到本機 Claude 狀態時，會提供 Claude 選項。

    ```bash
    openclaw onboard --flow import
    ```

    或指定特定來源：

    ```bash
    openclaw onboard --import-from claude --import-source ~/.claude
    ```

  </Tab>
  <Tab title="命令列介面">
    使用 `openclaw migrate` 執行指令碼化或可重複的作業。完整參考資料請參閱 [`openclaw migrate`](/zh-TW/cli/migrate)。

    ```bash
    openclaw migrate claude --dry-run
    openclaw migrate apply claude --yes
    ```

    加上 `--from <path>`，即可匯入特定的 Claude Code 主目錄或專案根目錄。

  </Tab>
</Tabs>

## 匯入的內容

<AccordionGroup>
  <Accordion title="指示與記憶">
    - 專案 `CLAUDE.md` 與 `.claude/CLAUDE.md` 的內容會複製或附加至 OpenClaw 代理程式工作區的 `AGENTS.md`。
    - 使用者 `~/.claude/CLAUDE.md` 的內容會附加至工作區的 `USER.md`。

  </Accordion>
  <Accordion title="MCP 伺服器">
    如果存在 MCP 伺服器定義，則會從專案的 `.mcp.json`、Claude Code 的 `~/.claude.json` 及 Claude Desktop 的 `claude_desktop_config.json` 匯入。
  </Accordion>
  <Accordion title="Skills 與命令">
    - 含有 `SKILL.md` 檔案的 Claude Skills 會複製至 OpenClaw 工作區的 Skills 目錄。
    - `.claude/commands/` 或 `~/.claude/commands/` 下的 Claude 命令 Markdown 檔案，會透過 `disable-model-invocation: true` 轉換為 OpenClaw Skills。

  </Accordion>
</AccordionGroup>

## 僅保留於封存中的內容

提供者會將下列內容複製到遷移報告中供手動檢閱，但**不會**將其載入即時 OpenClaw 設定：

- Claude 鉤子
- Claude 權限與廣泛的工具允許清單
- Claude 環境預設值
- `CLAUDE.local.md`
- `.claude/rules/`
- `.claude/agents/` 或 `~/.claude/agents/` 下的 Claude 子代理程式
- Claude Code 快取、計畫與專案歷史記錄目錄
- Claude Desktop 擴充功能與作業系統儲存的認證資訊

OpenClaw 拒絕自動執行鉤子、信任權限允許清單，或解碼不透明的 OAuth 與 Desktop 認證資訊狀態。請在檢閱封存內容後，手動移動所需項目。

## 來源選取

未指定 `--from` 時，OpenClaw 會檢查位於 `~/.claude` 的預設 Claude Code 主目錄、取樣的 Claude Code `~/.claude.json` 狀態檔，以及 macOS 上的 Claude Desktop MCP 設定。

當 `--from` 指向專案根目錄時，OpenClaw 僅匯入該專案的 Claude 檔案，例如 `CLAUDE.md`、`.claude/settings.json`、`.claude/commands/`、`.claude/skills/` 與 `.mcp.json`。從專案根目錄匯入時，不會讀取全域 Claude 主目錄。

## 建議流程

<Steps>
  <Step title="預覽計畫">
    ```bash
    openclaw migrate claude --dry-run
    ```

    計畫會列出所有將變更的內容，包括衝突、略過的項目，以及從巢狀 MCP `env` 或 `headers` 欄位中遮蔽的敏感值。

  </Step>
  <Step title="套用並建立備份">
    ```bash
    openclaw migrate apply claude --yes
    ```

    OpenClaw 會在套用前建立並驗證備份。

  </Step>
  <Step title="執行診斷">
    ```bash
    openclaw doctor
    ```

    [診斷](/zh-TW/gateway/doctor)會在匯入後檢查設定或狀態問題。

  </Step>
  <Step title="重新啟動並驗證">
    ```bash
    openclaw gateway restart
    openclaw status
    ```

    確認閘道運作正常，且匯入的指示、MCP 伺服器與 Skills 均已載入。

  </Step>
</Steps>

## 衝突處理

當計畫回報衝突（目標位置已存在檔案或設定值）時，套用作業會拒絕繼續。

<Warning>
僅在確定要取代現有目標時，才使用 `--overwrite` 重新執行。提供者仍可能在遷移報告目錄中，為遭覆寫的檔案建立項目層級的備份。
</Warning>

對全新安裝的 OpenClaw 而言，衝突並不常見。通常是在已有使用者編輯內容的設定上重新執行匯入時才會發生。

## 自動化用 JSON 輸出

```bash
openclaw migrate claude --dry-run --json
openclaw migrate apply claude --json --yes
```

在互動式終端機之外，`migrate apply` 必須搭配 `--yes`；若未指定，OpenClaw 會回報錯誤而非套用，因此指令碼與 CI 必須明確傳入 `--yes`。請先使用 `--dry-run --json` 預覽，確認計畫無誤後，再使用 `--json --yes` 套用。

## 疑難排解

<AccordionGroup>
  <Accordion title="Claude 狀態位於 ~/.claude 之外">
    傳入 `--from /actual/path`（命令列介面）或 `--import-source /actual/path`（設定精靈）。
  </Accordion>
  <Accordion title="設定精靈拒絕在現有設定上匯入">
    透過設定精靈匯入需要全新的設定。請重設狀態後重新執行設定精靈，或直接使用支援 `--overwrite` 與明確備份控制的 `openclaw migrate apply claude`。
  </Accordion>
  <Accordion title="未匯入 Claude Desktop 的 MCP 伺服器">
    Claude Desktop 會從平台特定路徑讀取 `claude_desktop_config.json`。如果 OpenClaw 未自動偵測到該檔案，請將 `--from` 指向其所在目錄。
  </Accordion>
  <Accordion title="Claude 命令轉為停用模型叫用的 Skills">
    這是預期設計。Claude 命令由使用者觸發，因此 OpenClaw 會將其匯入為含有 `disable-model-invocation: true` 的 Skills。如果希望代理程式自動叫用這些 Skills，請編輯各 Skill 的 frontmatter。
  </Accordion>
</AccordionGroup>

## 相關內容

- [`openclaw migrate`](/zh-TW/cli/migrate)：完整的命令列介面參考資料、外掛合約與 JSON 結構。
- [遷移指南](/zh-TW/install/migrating)：所有遷移路徑。
- [從 Hermes 遷移](/zh-TW/install/migrating-hermes)：另一個跨系統匯入路徑。
- [設定精靈](/zh-TW/cli/onboard)：精靈流程與非互動式旗標。
- [診斷](/zh-TW/gateway/doctor)：遷移後的健康狀態檢查。
- [代理程式工作區](/zh-TW/concepts/agent-workspace)：`AGENTS.md`、`USER.md` 與 Skills 的所在位置。
