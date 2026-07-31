---
read_when:
    - 你正從 Hermes 遷移而來，並想保留你的模型設定、提示詞、記憶與 Skills
    - 你想知道 OpenClaw 會自動匯入哪些內容，以及哪些內容僅保留於封存中
    - 你需要一個乾淨、可透過指令碼執行的移轉路徑（CI、全新筆電、自動化）
summary: 透過可預覽、可復原的匯入作業，從 Hermes 移轉至 OpenClaw
title: 從 Hermes 遷移
x-i18n:
    generated_at: "2026-07-26T08:21:41Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: f8cdb7a77cfb8ecb0504ccc322b5600c6ed671a8bf9ac866d964fdf4b3494000
    source_path: install/migrating-hermes.md
    workflow: 16
---

隨附的 Hermes 遷移提供者會遵循 `HERMES_HOME` 與使用中的 Hermes 設定檔，並在 macOS/Linux 上回退至 `~/.hermes`，或在 Windows 上回退至 `%LOCALAPPDATA%\hermes`。它會在套用前預覽每項變更，並在計畫與報告中遮蔽祕密。獨立執行的 `openclaw migrate` 會寫入經驗證的備份；全新初始設定路徑會暫存設定、認證資訊與檔案，並且只在匯入的推論通過驗證後才發布。明確指定的 `--from` 路徑一律優先。

<Note>
匯入需要全新的 OpenClaw 設定。如果你已有本機 OpenClaw 狀態，請先重設設定、認證資訊、工作階段與工作區，或在檢閱計畫後，直接搭配 `--overwrite` 使用 `openclaw migrate apply hermes`。
</Note>

## 兩種匯入方式

<Tabs>
  <Tab title="初始設定精靈">
    偵測使用中的 Hermes 主目錄／設定檔，並在套用前顯示預覽。

    ```bash
    openclaw onboard --flow import
    ```

    或指定特定來源：

    ```bash
    openclaw onboard --import-from hermes --import-source ~/.hermes
    ```

  </Tab>
  <Tab title="命令列介面">
    若要執行腳本化或可重複的作業，請使用 `openclaw migrate`。完整參考資料請參閱 [`openclaw migrate`](/zh-TW/cli/migrate)。

    ```bash
    openclaw migrate hermes --dry-run    # 僅預覽
    openclaw migrate apply hermes --yes  # 套用並略過確認
    ```

    加入 `--from <path>` 可覆寫 Hermes 主目錄／設定檔探索。

  </Tab>
</Tabs>

## 匯入的內容

<AccordionGroup>
  <Accordion title="模型設定">
    - 來自 Hermes `config.yaml` 的預設模型選擇。
    - 來自 `model`、`providers` 與 `custom_providers` 的已設定模型提供者及自訂端點，包括目前的 Hermes Chat Completions、Codex Responses 與 Anthropic Messages 傳輸方式。

  </Accordion>
  <Accordion title="MCP 伺服器">
    來自 `mcp_servers` 或 `mcp.servers` 的 MCP 伺服器定義，包括停用狀態、逾時、平行工具支援、OAuth 範圍、相容的 TLS 欄位，以及原生／資源／提示詞工具原則。常值環境變數與標頭需要取得匯入認證資訊的同意。Hermes 專屬的生命週期、取樣、引導式資訊擷取、預檢、保持連線、CA 套件組、受密碼保護的用戶端金鑰，以及預先註冊的 OAuth 用戶端設定，會成為需手動檢閱的項目，而非無效的 OpenClaw 設定。
  </Accordion>
  <Accordion title="工作區檔案">
    - `SOUL.md` 與 `AGENTS.md` 會複製到 OpenClaw 代理程式工作區。
    - `memories/MEMORY.md` 與 `memories/USER.md` 會**附加**至相符的 OpenClaw 記憶檔案，而非覆寫這些檔案。
    - 僅記憶體介面的行為不同：初始設定記憶體頁面與控制介面的記憶體匯入頁面會將這兩個檔案複製到 `memory/imports/hermes/` 下以供索引召回，並保持現有工作區記憶不變。

  </Accordion>
  <Accordion title="記憶體設定">
    OpenClaw 檔案記憶體的記憶體設定預設值。Honcho 等外部記憶體提供者會記錄為封存項目或需手動檢閱的項目，讓你能審慎地遷移它們。
  </Accordion>
  <Accordion title="Skills">
    系統會遞迴探索 `skills/` 下任何位置含有 `SKILL.md` 檔案的 Skills，將其扁平化至 OpenClaw 工作區的 skill 目錄，並連同其支援檔案一起複製。來自 `skills.config` 的各 skill 設定值會予以保留。
  </Accordion>
  <Accordion title="驗證認證資訊">
    互動式 `openclaw migrate` 會在匯入驗證認證資訊前詢問，且預設選取「是」。可接受的匯入項目包括目前的 Hermes OpenAI Codex OAuth 項目、OpenCode OpenAI OAuth 與 GitHub Copilot 項目，以及[支援的 Hermes `.env` 金鑰](/zh-TW/cli/migrate#supported-env-keys)。非互動式匯入請使用 `--include-secrets`，略過認證資訊請使用 `--no-auth-credentials`，或使用初始設定的 `--import-secrets` 旗標。匯入 Hermes OAuth 後，請勿讓 Hermes 與 OpenClaw 繼續使用相同的重新整理授權；在同時執行兩者前，請重新驗證其中一方。
  </Accordion>
</AccordionGroup>

## 僅保留於封存的內容

提供者會將下列項目複製到遷移報告目錄以供手動檢閱，但**不會**將其載入使用中的 OpenClaw 設定或認證資訊：

- `plugins/`
- `sessions/`
- `logs/`
- `cron/`
- `mcp-tokens/`
- `plans/`、`workspace/`、`skins/` 與 `kanban/`
- `pairing/` 與 `platforms/` 儲存區，以及閘道路由／程序狀態
- `state.db`、`hermes_state.db`、`projects.db`、`response_store.db`、`memory_store.db`、`verification_evidence.db`、`kanban.db` 與 `retaindb_queue.db`

OpenClaw 拒絕自動執行或信任此狀態，因為不同系統間的格式與信任假設可能會產生差異。請在檢閱封存內容後，手動移動所需項目。

## 建議流程

<Steps>
  <Step title="預覽計畫">
    ```bash
    openclaw migrate hermes --dry-run
    ```

    計畫會列出所有將變更的項目，包括衝突、略過的項目與敏感項目。輸出中巢狀且看似祕密的金鑰會被遮蔽。

  </Step>
  <Step title="建立備份並套用">
    ```bash
    openclaw migrate apply hermes --yes
    ```

    OpenClaw 會在套用前建立並驗證備份。此非互動式範例只會匯入非祕密狀態。若要以互動方式回答認證資訊提示，請在不加 `--yes` 的情況下執行；若要在無人值守的執行中納入支援的認證資訊，請加入 `--include-secrets`。

  </Step>
  <Step title="執行 doctor">
    ```bash
    openclaw doctor
    ```

    [Doctor](/zh-TW/gateway/doctor) 會重新套用任何待處理的設定遷移，並檢查匯入期間引入的問題。

  </Step>
  <Step title="重新啟動並驗證">
    ```bash
    openclaw gateway restart
    openclaw status
    ```

    確認閘道運作正常，且匯入的模型、記憶體與 Skills 均已載入。

  </Step>
</Steps>

## 衝突處理

當計畫回報衝突（目標位置已有檔案或設定值）時，套用作業會拒絕繼續。

<Warning>
只有在確定要取代現有目標時，才使用 `--overwrite` 重新執行。提供者仍可能在遷移報告目錄中，為遭覆寫的檔案寫入項目層級的備份。
</Warning>

全新安裝時很少發生衝突。通常是在已包含使用者編輯內容的設定上重新執行匯入時才會出現。

如果衝突在套用途中浮現（例如設定檔發生非預期的競爭情況），該項目會回報為衝突，而彼此獨立的檔案、Skills、認證資訊、封存內容與設定項目則會繼續處理。解決衝突項目後重新執行匯入；相同的記憶體匯入具有冪等性。

## 祕密

互動式 `openclaw migrate` 會詢問是否匯入偵測到的驗證認證資訊，且預設選取「是」。

- 接受後會匯入目前的 Hermes OpenAI Codex OAuth 項目、OpenCode OpenAI OAuth 與 GitHub Copilot 項目，以及[支援的 `.env` 金鑰](/zh-TW/cli/migrate#supported-env-keys)。
- 若只要匯入非祕密狀態，請使用 `--no-auth-credentials`，或在提示時回答「否」。
- 若要在無人值守的 `--yes` 執行中匯入認證資訊，請使用 `--include-secrets`。
- 若要從精靈匯入認證資訊，請使用初始設定精靈的 `--import-secrets` 旗標。

## 用於自動化的 JSON 輸出

```bash
openclaw migrate hermes --dry-run --json
openclaw migrate apply hermes --json --yes
```

使用 `--json` 且未使用 `--yes` 時，套用作業會輸出計畫但不變更狀態，這是 CI 與共用腳本最安全的模式。

## 疑難排解

<AccordionGroup>
  <Accordion title="套用因衝突而遭拒">
    檢查計畫輸出。每項衝突都會指出來源路徑與現有目標。請逐項決定要略過、編輯目標，或使用 `--overwrite` 重新執行。
  </Accordion>
  <Accordion title="Hermes 位於 ~/.hermes 以外的位置">
    傳入 `--from /actual/path`（命令列介面）或 `--import-source /actual/path`（初始設定）。
  </Accordion>
  <Accordion title="初始設定拒絕匯入至現有設定">
    初始設定匯入需要全新的設定。你可以重設狀態並重新執行初始設定，或直接使用 `openclaw migrate apply hermes`；後者支援 `--overwrite` 與明確的備份控制。
  </Accordion>
  <Accordion title="API 金鑰未匯入">
    互動式 `openclaw migrate` 只有在你接受認證資訊提示時才會匯入 API 金鑰。非互動式 `--yes` 執行需要 `--include-secrets`；初始設定匯入則需要 `--import-secrets`。系統只會識別[支援的 `.env` 金鑰](/zh-TW/cli/migrate#supported-env-keys)，其他 `.env` 變數會被忽略。
  </Accordion>
</AccordionGroup>

## 相關內容

- [`openclaw migrate`](/zh-TW/cli/migrate)：完整的命令列介面參考資料、外掛合約與 JSON 結構。
- [初始設定](/zh-TW/cli/onboard)：精靈流程與非互動式旗標。
- [遷移](/zh-TW/install/migrating)：在不同機器間移動 OpenClaw 安裝。
- [Doctor](/zh-TW/gateway/doctor)：遷移後的健康狀態檢查。
- [代理程式工作區](/zh-TW/concepts/agent-workspace)：`SOUL.md`、`AGENTS.md` 與記憶體檔案的存放位置。
