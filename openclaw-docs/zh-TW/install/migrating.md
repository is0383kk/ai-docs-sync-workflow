---
read_when:
    - 你正在將 OpenClaw 移至新的筆記型電腦或伺服器
    - 你正從另一個代理系統轉移過來，並希望保留狀態
    - 你正在就地升級外掛
summary: 遷移中心：跨系統匯入、機器間移轉與外掛升級
title: 遷移指南
x-i18n:
    generated_at: "2026-07-26T08:29:39Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: e9ceb80045ab082c9cfc9e1aca59e079b6bf28b1d047265a0be40c03ebe5dac6
    source_path: install/migrating.md
    workflow: 16
---

OpenClaw 支援三種移轉路徑：從另一個代理系統匯入、將現有安裝移至新機器，以及就地升級外掛。

## 從另一個代理系統匯入

內建的移轉提供者可將指示、MCP 伺服器、skills、模型設定，以及（選擇性加入的）API 金鑰帶入 OpenClaw。任何變更執行前都會先預覽計畫，且報告中的秘密會經過遮蔽。獨立執行的 `openclaw migrate` 會以經過驗證的備份為依據；全新初始設定中的匯入則會先暫存並驗證本機成品，提交設定後才發布，並且會在進行任何不可逆的外部啟用之前完成。

<CardGroup cols={2}>
  <Card title="從 Claude 移轉" href="/zh-TW/install/migrating-claude" icon="brain">
    匯入 Claude Code 與 Claude Desktop 狀態，包括 `CLAUDE.md`、MCP 伺服器、skills 和專案命令。
  </Card>
  <Card title="從 Hermes 移轉" href="/zh-TW/install/migrating-hermes" icon="feather">
    匯入 Hermes 設定、提供者、MCP 伺服器、記憶、skills，以及支援的 `.env` 金鑰。
  </Card>
</CardGroup>

命令列介面的進入點是 [`openclaw migrate`](/zh-TW/cli/migrate)。初始設定偵測到已知來源（`openclaw onboard --flow import`）時，也可以提供移轉選項。

## 將 OpenClaw 移至新機器

複製**狀態目錄**（預設為 `~/.openclaw/`）及你的**工作區**，以保留下列內容：

- **設定** — `openclaw.json` 及所有閘道設定。
- **驗證** — 每個代理的 `auth-profiles.json`（API 金鑰與 OAuth），以及 `credentials/` 下的任何頻道或提供者狀態。
- **工作階段** — 對話記錄與代理狀態。
- **頻道狀態** — WhatsApp 登入、Telegram 工作階段及類似內容。
- **工作區檔案** — `MEMORY.md`、`USER.md`、skills 和提示詞。

<Tip>
在舊機器上執行 `openclaw status`，確認你的狀態目錄路徑。自訂設定檔會使用 `~/.openclaw-<profile>/`，或使用透過 `OPENCLAW_STATE_DIR` 設定的路徑。
</Tip>

### 移轉步驟

<Steps>
  <Step title="停止閘道並備份">
    在**舊**機器上停止閘道，以免檔案在複製過程中變更，然後封存：

    ```bash
    openclaw gateway stop
    cd ~
    tar -czf openclaw-state.tgz .openclaw
    ```

    若使用多個設定檔（例如 `~/.openclaw-work`），請分別封存每一個設定檔。

  </Step>

  <Step title="在新機器上安裝 OpenClaw">
    在新機器上[安裝](/zh-TW/install)命令列介面（若需要，也安裝節點）。初始設定即使建立了全新的 `~/.openclaw/` 也沒關係，你接下來會覆寫它。
  </Step>

  <Step title="複製狀態目錄與工作區">
    透過 `scp`、`rsync -a` 或外接硬碟傳輸封存檔，然後解壓縮：

    ```bash
    cd ~
    tar -xzf openclaw-state.tgz
    ```

    確認其中包含隱藏目錄，且檔案擁有權與將執行閘道的使用者相符。

  </Step>

  <Step title="執行 Doctor 並驗證">
    在新機器上執行 [Doctor](/zh-TW/gateway/doctor)，以套用設定移轉並修復服務：

    ```bash
    openclaw doctor
    openclaw gateway restart
    openclaw status
    ```

  </Step>
</Steps>

若 Telegram 或 Discord 使用預設環境變數後援（`TELEGRAM_BOT_TOKEN` 或 `DISCORD_BOT_TOKEN`），請確認已移轉的狀態目錄中，`.env` 包含這些金鑰，且不會印出秘密值：

```bash
awk -F= '/^(TELEGRAM_BOT_TOKEN|DISCORD_BOT_TOKEN)=/ { print $1 "=present" }' ~/.openclaw/.env
```

當已啟用的預設 Telegram 或 Discord 帳號未設定權杖，且 Doctor 程序無法取得對應的環境變數時，`openclaw doctor` 也會發出警告。

### 常見問題

<AccordionGroup>
  <Accordion title="設定檔或狀態目錄不一致">
    若舊閘道使用 `--profile` 或 `OPENCLAW_STATE_DIR`，而新閘道未使用，頻道會顯示為已登出，且工作階段將為空。請使用與已移轉內容**相同**的設定檔或狀態目錄啟動閘道，然後重新執行 `openclaw doctor`。
  </Accordion>

  <Accordion title="只複製 openclaw.json">
    僅有設定檔並不足夠。模型驗證設定檔位於 `agents/<agentId>/agent/auth-profiles.json` 下，頻道與提供者狀態則位於 `credentials/` 下。請一律移轉**整個**狀態目錄。
  </Accordion>

  <Accordion title="權限與擁有權">
    若以 root 身分複製或切換了使用者，閘道可能無法讀取認證資訊。請確保狀態目錄與工作區由執行閘道的使用者擁有。
  </Accordion>

  <Accordion title="遠端模式">
    若你的使用者介面指向**遠端**閘道，工作階段與工作區由遠端主機持有。請移轉閘道主機本身，而不是你的本機筆記型電腦。請參閱[常見問題](/zh-TW/help/faq#where-things-live-on-disk)。
  </Accordion>

  <Accordion title="備份中的秘密">
    狀態目錄包含驗證設定檔、頻道認證資訊及其他提供者狀態。請以加密方式儲存備份、避免使用不安全的傳輸管道，若懷疑資料外洩，請輪替金鑰。
  </Accordion>
</AccordionGroup>

### 驗證檢查清單

在新機器上確認：

- [ ] `openclaw status` 顯示閘道正在執行。
- [ ] 頻道仍保持連線（不需要重新配對）。
- [ ] 儀表板可正常開啟並顯示現有工作階段。
- [ ] 工作區檔案（記憶、設定）均存在。

## 就地升級外掛

就地外掛升級會保留相同的外掛 ID 與設定鍵，但可能會將磁碟上的狀態移至目前的配置。外掛專用的升級指南與其頻道文件放在一起：

- [Matrix 移轉](/zh-TW/channels/matrix-migration)：加密狀態的復原限制、自動快照行為，以及手動復原命令。

## 相關內容

- [`openclaw migrate`](/zh-TW/cli/migrate)：跨系統匯入的命令列介面參考。
- [安裝概覽](/zh-TW/install)：所有安裝方式。
- [Doctor](/zh-TW/gateway/doctor)：移轉後健康狀態檢查。
- [解除安裝](/zh-TW/install/uninstall)：完整移除 OpenClaw。
