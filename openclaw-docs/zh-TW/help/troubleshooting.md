---
read_when:
    - OpenClaw 無法運作，而你需要以最快的方式修復問題
    - 你需要在深入詳盡的操作手冊前先進行分流流程
summary: OpenClaw 依症狀優先的疑難排解中心
title: 一般疑難排解
x-i18n:
    generated_at: "2026-07-26T08:28:08Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: de3554ed680ac536d105017220b44d94456a4408916e949352500b046f4d5f17
    source_path: help/troubleshooting.md
    workflow: 16
---

分流入口。2 分鐘內完成診斷，接著前往深入說明頁面。

## 前 60 秒

依序執行以下檢查：

```bash
openclaw status
openclaw status --all
openclaw gateway probe
openclaw gateway status
openclaw doctor
openclaw channels status --probe
openclaw logs --follow
```

正常輸出，每項一行：

- `openclaw status` 顯示已設定的頻道，且沒有驗證錯誤。
- `openclaw status --all` 產生完整且可分享的報告。
- `openclaw gateway probe` 顯示 `Reachable: yes`。`Capability: ...` 是探測所證實的
  驗證層級；`Read probe: limited - missing scope:
operator.read` 表示診斷功能降級，而非連線失敗。
- `openclaw gateway status` 顯示 `Runtime: running`、`Connectivity probe:
ok`，以及合理的 `Capability: ...`。加上 `--require-rpc`，即可同時要求
  讀取範圍的 RPC 驗證。
- `openclaw doctor` 回報沒有阻礙運作的設定／服務錯誤。
- 閘道可連線時，`openclaw channels status --probe` 會傳回各帳號即時的傳輸狀態
  （`works`／`audit ok`）；無法連線時，則退回
  僅依設定產生的摘要。
- `openclaw logs --follow` 顯示活動穩定，且沒有重複發生的嚴重錯誤。

## 助理功能受限或缺少工具

檢查實際生效的工具設定檔：

```bash
openclaw status
openclaw status --all
openclaw doctor
```

常見原因：

- `tools.profile: "minimal"` 僅允許 `session_status`。
- `tools.profile: "messaging"` 範圍較窄，適用於僅進行聊天的代理程式。
- `tools.profile: "coding"` 是新的本機設定預設值（儲存庫、檔案、
  shell 和執行階段工作）。
- `tools.profile: "full"` 會移除設定檔限制；僅限由受信任的
  操作者控制之代理程式使用。
- 每個代理程式的 `agents.entries.*.tools` 可針對單一代理程式縮限或擴大根設定檔。

變更設定檔、重新啟動或重新載入閘道，然後使用
`openclaw status --all` 再次檢查。完整設定檔／群組表格：[工具設定檔](/zh-TW/gateway/config-tools#tool-profiles)。

## Anthropic 長上下文 429

`HTTP 429: rate_limit_error: Extra usage is required for long context requests`
→ [Anthropic 429：長上下文需要額外用量](/zh-TW/gateway/troubleshooting#anthropic-429-extra-usage-required-for-long-context)。

## 本機 OpenAI 相容後端可直接運作，但在 OpenClaw 中失敗

你的本機／自架 `/v1` 後端可回應直接的 `/v1/chat/completions`
探測，但在 `openclaw infer model run` 或一般代理程式回合中失敗：

1. 錯誤提到 `messages[].content` 預期收到字串：請設定
   `models.providers.<provider>.models[].compat.requiresStringContent: true`。
2. 仍然只在 OpenClaw 代理程式回合失敗：請設定
   `models.providers.<provider>.models[].compat.supportsTools: false`，然後重試。
3. 小型直接呼叫可運作，但較大的 OpenClaw 提示詞會使後端當機：這是
   上游模型／伺服器的限制，並非 OpenClaw 錯誤。請繼續參閱
   [本機 OpenAI 相容後端通過直接探測，但代理程式執行失敗](/zh-TW/gateway/troubleshooting#local-openai-compatible-backend-passes-direct-probes-but-agent-runs-fail)。

## 安裝外掛時因缺少 openclaw extensions 而失敗

`package.json missing openclaw.extensions` 表示外掛套件使用了
OpenClaw 已不再接受的結構。

請在外掛套件中修正：

1. 將 `openclaw.extensions` 加入 `package.json`，並指向建置完成的執行階段
   檔案（通常是 `./dist/index.js`）。
2. 重新發布，然後再次執行 `openclaw plugins install <package>`。

```json
{
  "name": "@openclaw/my-plugin",
  "version": "1.2.3",
  "openclaw": {
    "extensions": ["./dist/index.js"]
  }
}
```

參考：[外掛架構](/zh-TW/plugins/architecture)

## 安裝政策封鎖外掛安裝或更新

更新完成，但外掛仍為舊版、遭停用，或顯示 `blocked by install
policy`、`install policy failed closed` 或 `Disabled "<plugin>" after plugin
update failure`：請檢查 `security.installPolicy`。

安裝政策會套用於外掛安裝與更新。`@openclaw/*` 外掛
版本通常會隨 OpenClaw 發行版本變動，因此 OpenClaw 更新後，
可能需要在更新後同步期間進行相符的外掛更新。

除非也維護相符的升級規則，否則請避免下列政策形式：

- 將 OpenClaw 擁有的外掛固定於某個確切的舊版本（例如只允許
  `@openclaw/*@2026.5.3`）。
- 僅依來源類型封鎖（所有 npm、網路或 `request.mode:
"update"` 請求）。
- 將政策命令視為選用：啟用 `security.installPolicy` 時，
  政策執行檔若缺少、過慢、無法讀取或因權限遭封鎖，
  皆會採取失敗時封鎖。
- 核准版本時，未將請求的 `openclawVersion` 與
  外掛候選項目的中繼資料進行比對。

請優先採用允許受信任且與目前主機相容的 `@openclaw/*` 更新之規則，
而非永久固定於單一發行版本。若預設封鎖 npm，
請針對你使用的外掛 ID 新增範圍有限的例外，並對 `request.mode: "update"`
套用與安裝相同的信任規則。

復原：

```bash
openclaw doctor --deep
openclaw plugins update --all
openclaw status --all
```

若政策刻意設為嚴格，請在受信任的升級
時段暫時放寬政策，重新執行 `openclaw plugins update --all`，再恢復較嚴格的規則。
若更新失敗導致外掛遭停用，請先檢查再重新啟用：

```bash
openclaw plugins inspect <plugin-id> --runtime --json
openclaw plugins enable <plugin-id>
```

參考：[操作者安裝政策](/zh-TW/tools/skills-config#operator-install-policy-securityinstallpolicy)

## 外掛存在，但因可疑的擁有權而遭封鎖

`openclaw doctor`、設定或啟動警告顯示：

```text
遭封鎖的外掛候選項目：擁有權可疑（... uid=1000，預期 uid=0 或 root）
外掛存在，但遭到封鎖
```

外掛檔案的擁有者與載入這些檔案的程序所屬 Unix 使用者不同。
請勿移除外掛設定；請修正檔案擁有權，或以狀態目錄擁有者的身分
執行 OpenClaw。

Docker 安裝會以 `node`（uid `1000`）執行。請修復主機的繫結掛載：

```bash
sudo chown -R 1000:1000 /path/to/openclaw-config /path/to/openclaw-workspace
openclaw doctor --fix
```

若你刻意以 root 身分執行 OpenClaw，請改為修復受管理的外掛根目錄：

```bash
sudo chown -R root:root /path/to/openclaw-config/npm
openclaw doctor --fix
```

深入說明：[遭封鎖之外掛路徑的擁有權](/zh-TW/tools/plugin#blocked-plugin-path-ownership)、[Docker：權限與 EACCES](/zh-TW/install/docker#shell-helpers-optional)

## 決策樹

```mermaid
flowchart TD
  A[OpenClaw 無法運作] --> B{最先發生哪種問題}
  B --> C[沒有回覆]
  B --> D[儀表板或 Control UI 無法連線]
  B --> E[閘道無法啟動或服務未執行]
  B --> F[頻道已連線，但訊息未傳遞]
  B --> G[排程或心跳偵測未觸發或未送達]
  B --> H[節點已配對，但相機、畫布、螢幕或 exec 失敗]
  B --> I[瀏覽器工具失敗]

  C --> C1[/沒有回覆章節/]
  D --> D1[/Control UI 章節/]
  E --> E1[/閘道章節/]
  F --> F1[/頻道流程章節/]
  G --> G1[/自動化章節/]
  H --> H1[/節點工具章節/]
  I --> I1[/瀏覽器章節/]
```

<AccordionGroup>
  <Accordion title="沒有回覆">
    ```bash
    openclaw status
    openclaw gateway status
    openclaw channels status --probe
    openclaw pairing list --channel <channel> [--account <id>]
    openclaw logs --follow
    ```

    正常輸出：

    - `Runtime: running`
    - `Connectivity probe: ok`
    - `Capability: read-only`、`write-capable` 或 `admin-capable`
    - 頻道顯示傳輸已連線，且在支援的情況下，`channels status --probe` 中顯示 `works` 或
      `audit ok`
    - 傳送者已核准（或私訊政策設為開放／允許清單）

    記錄特徵：

    - `drop guild message (mention required` → Discord 提及限制封鎖了該訊息。
    - `pairing request` → 傳送者尚未核准，正在等待私訊配對核准。
    - 頻道記錄中的 `blocked`／`allowlist` → 傳送者、聊天室或群組遭到篩除。

    深入說明：[沒有回覆](/zh-TW/gateway/troubleshooting#no-replies)、[頻道疑難排解](/zh-TW/channels/troubleshooting)、[配對](/zh-TW/channels/pairing)

  </Accordion>

  <Accordion title="儀表板或 Control UI 無法連線">
    ```bash
    openclaw status
    openclaw gateway status
    openclaw logs --follow
    openclaw doctor
    openclaw channels status --probe
    ```

    正常輸出：

    - `openclaw gateway status` 中顯示 `Dashboard: http://...`
    - `Connectivity probe: ok`
    - `Capability: read-only`、`write-capable` 或 `admin-capable`
    - 記錄中沒有驗證迴圈

    記錄特徵：

    - `device identity required` → HTTP／非安全內容無法完成裝置驗證。
    - `origin not allowed` → Control UI 閘道目標不允許瀏覽器 `Origin`。
    - `AUTH_TOKEN_MISMATCH` 搭配 `canRetryWithDeviceToken=true` → 系統可能會自動重試一次受信任的裝置權杖，並重複使用已配對權杖的快取範圍。
    - 該次重試後仍重複出現 `unauthorized` → 權杖／密碼錯誤、驗證模式不符，或已配對的裝置權杖過時。
    - `too many failed authentication attempts (retry later)` → 來自該瀏覽器 `Origin` 的重複失敗暫時遭到鎖定；其他 localhost 來源使用獨立的區間。關於 Tailscale Serve 同時重試的細節，請參閱[儀表板／Control UI 連線能力](/zh-TW/gateway/troubleshooting#dashboard-control-ui-connectivity)。
    - `gateway connect failed:` → UI 指向錯誤的 URL／連接埠，或閘道無法連線。

    深入說明：[儀表板／Control UI 連線能力](/zh-TW/gateway/troubleshooting#dashboard-control-ui-connectivity)、[Control UI](/zh-TW/web/control-ui)、[驗證](/zh-TW/gateway/authentication)

  </Accordion>

  <Accordion title="閘道無法啟動，或服務已安裝但未執行">
    ```bash
    openclaw status
    openclaw gateway status
    openclaw logs --follow
    openclaw doctor
    openclaw channels status --probe
    ```

    正常輸出：

    - `Service: ... (loaded)`
    - `Runtime: running`
    - `Connectivity probe: ok`
    - `Capability: read-only`、`write-capable` 或 `admin-capable`

    記錄特徵：

    - `Gateway start blocked: set gateway.mode=local` 或 `existing config is missing gateway.mode` → 閘道模式為遠端，或設定缺少本機模式標記而需要修復。
    - `refusing to bind gateway ... without auth` → 綁定非回送位址，但沒有有效的驗證路徑（權杖／密碼，或已設定的受信任 Proxy）。
    - `another gateway instance is already listening` 或 `EADDRINUSE` → 連接埠已被占用。

    深入說明：[閘道服務未執行](/zh-TW/gateway/troubleshooting#gateway-service-not-running)、[背景程序](/zh-TW/gateway/background-process)、[設定](/zh-TW/gateway/configuration)

  </Accordion>

  <Accordion title="頻道已連線，但訊息未傳遞">
    ```bash
    openclaw status
    openclaw gateway status
    openclaw logs --follow
    openclaw doctor
    openclaw channels status --probe
    ```

    正常輸出：

    - 頻道傳輸已連線。
    - 配對／允許清單檢查通過。
    - 需要提及時，已偵測到提及。

    記錄特徵：

    - `mention required` → 群組提及限制封鎖了處理。
    - `pairing`／`pending` → 私訊傳送者尚未核准。
    - `not_in_channel`、`missing_scope`、`Forbidden`、`401/403` → 頻道權限權杖問題。

    深入說明：[頻道已連線，但訊息未傳遞](/zh-TW/gateway/troubleshooting#channel-connected-messages-not-flowing)、[頻道疑難排解](/zh-TW/channels/troubleshooting)

  </Accordion>

  <Accordion title="排程或心跳偵測未觸發或未送達">
    ```bash
    openclaw status
    openclaw gateway status
    openclaw cron status
    openclaw cron list
    openclaw cron runs --id <jobId> --limit 20
    openclaw logs --follow
    ```

    正常輸出：

    - `cron status` 顯示排程器已啟用，並有下一次喚醒時間。
    - `cron runs` 顯示最近的 `ok` 項目。
    - 心跳偵測已啟用，且目前在作用時段內。

    日誌特徵：

    - `cron: scheduler disabled; jobs will not run automatically` → 排程已停用。
    - `heartbeat skipped` 原因 `quiet-hours` → 不在設定的作用時段內。
    - `heartbeat skipped` 原因 `empty-heartbeat-file` → 心跳偵測監控暫存內容只有空白、註解、標頭、圍欄或空白檢查清單的鷹架。
    - `heartbeat skipped` 原因 `alerts-disabled` → `showOk`、`showAlerts` 和 `useIndicator` 均已關閉。
    - `requests-in-flight` → 主要通道忙碌中；心跳偵測喚醒已延後。
    - `unknown accountId` → 心跳偵測傳遞目標帳號不存在。

    深入頁面：[排程與心跳偵測傳遞](/zh-TW/gateway/troubleshooting#cron-and-heartbeat-delivery)、[排定的工作：疑難排解](/zh-TW/automation/cron-jobs#troubleshooting)、[心跳偵測](/zh-TW/gateway/heartbeat)

  </Accordion>

  <Accordion title="節點已配對，但工具執行 camera canvas screen exec 失敗">
    ```bash
    openclaw status
    openclaw gateway status
    openclaw nodes status
    openclaw nodes describe --node <idOrNameOrIp>
    openclaw logs --follow
    ```

    正常輸出：

    - 節點顯示為已連線，且已針對角色 `node` 完成配對。
    - 你正在叫用的命令具備所需功能。
    - 工具的權限狀態為已授予。

    日誌特徵：

    - `NODE_BACKGROUND_UNAVAILABLE` → 將節點應用程式切換至前景。
    - `*_PERMISSION_REQUIRED` → 作業系統權限遭拒或缺少。
    - `SYSTEM_RUN_DENIED: approval required` → exec 核准待處理。
    - `SYSTEM_RUN_DENIED: allowlist miss` → 命令不在 exec 允許清單中。

    深入頁面：[節點已配對，但工具失敗](/zh-TW/gateway/troubleshooting#node-paired-tool-fails)、[節點疑難排解](/zh-TW/nodes/troubleshooting)、[Exec 核准](/zh-TW/tools/exec-approvals)

  </Accordion>

  <Accordion title="Exec 突然要求核准">
    ```bash
    openclaw config get tools.exec.host
    openclaw config get tools.exec.security
    openclaw config get tools.exec.ask
    openclaw gateway restart
    ```

    變更內容：

    - 未設定的 `tools.exec.host` 預設為 `auto`；當沙箱執行階段處於作用中時，
      會解析為 `sandbox`，否則為 `gateway`。
    - `host=auto` 只負責路由；不顯示提示的行為來自閘道／節點上的
      `security=full` 加上 `ask=off`。
    - 在 `gateway`/`node` 上，未設定的 `tools.exec.security` 預設為 `full`。
    - 未設定的 `tools.exec.ask` 預設為 `off`。
    - 如果出現核准要求，表示某個主機本機或個別工作階段的原則
      已收緊 exec 設定，使其偏離這些預設值。

    還原目前無須核准的預設值：

    ```bash
    openclaw config set tools.exec.host gateway
    openclaw config set tools.exec.security full
    openclaw config set tools.exec.ask off
    openclaw gateway restart
    ```

    更安全的替代方案：

    - 若要穩定地將工作路由至主機，僅設定 `tools.exec.host=gateway`。
    - 使用 `security=allowlist` 搭配 `ask=on-miss`，即可在允許清單未命中時，
      對主機 exec 進行審查。
    - 啟用沙箱模式，讓 `host=auto` 重新解析為 `sandbox`。

    日誌特徵：

    - `Approval required.` → 命令正在等待 `/approve ...`。
    - `SYSTEM_RUN_DENIED: approval required` → 節點主機 exec 核准待處理。
    - `exec host=sandbox requires a sandbox runtime for this session` → 已隱含或明確選取沙箱，但沙箱模式已關閉。

    深入頁面：[Exec](/zh-TW/tools/exec)、[Exec 核准](/zh-TW/tools/exec-approvals)、[安全性：稽核檢查的項目](/zh-TW/gateway/security#what-the-audit-checks-high-level)

  </Accordion>

  <Accordion title="瀏覽器工具失敗">
    ```bash
    openclaw status
    openclaw gateway status
    openclaw browser status
    openclaw logs --follow
    openclaw doctor
    ```

    正常輸出：

    - 瀏覽器狀態顯示 `running: true`，以及所選的瀏覽器／設定檔。
    - `openclaw` 設定檔可以啟動，或 `user` 設定檔可以看到本機 Chrome 分頁。

    日誌特徵：

    - `unknown command "browser"` → 已設定 `plugins.allow`，且其中排除了 `browser`。
    - `Failed to start Chrome CDP on port` → 本機瀏覽器啟動失敗。
    - `browser.executablePath not found` → 設定的二進位檔路徑錯誤。
    - `browser.cdpUrl must be http(s) or ws(s)` → 設定的 CDP URL 使用不支援的配置。
    - `browser.cdpUrl has invalid port` → 設定的 CDP URL 連接埠無效或超出範圍。
    - `No Chrome tabs found for profile="user"` → Chrome MCP 附加設定檔沒有任何開啟的本機 Chrome 分頁。
    - `Remote CDP for profile "<name>" is not reachable` → 無法從此主機連線至設定的遠端 CDP 端點。
    - `Browser attachOnly is enabled ... not reachable` → 僅附加設定檔沒有即時 CDP 目標。
    - 僅附加或遠端 CDP 設定檔上有過時的檢視區／深色模式／地區設定／離線覆寫 → 執行 `openclaw browser stop --browser-profile <name>`，無須重新啟動閘道即可關閉控制工作階段並釋放模擬狀態。

    深入頁面：[瀏覽器工具失敗](/zh-TW/gateway/troubleshooting#browser-tool-fails)、[缺少瀏覽器命令或工具](/zh-TW/tools/browser#missing-browser-command-or-tool)、[瀏覽器：Linux 疑難排解](/zh-TW/tools/browser-linux-troubleshooting)、[瀏覽器：WSL2/Windows 遠端 CDP 疑難排解](/zh-TW/tools/browser-wsl2-windows-remote-cdp-troubleshooting)

  </Accordion>

</AccordionGroup>

## 相關內容

- [常見問題](/zh-TW/help/faq) — 常見問題與解答
- [閘道疑難排解](/zh-TW/gateway/troubleshooting) — 閘道特有的問題
- [Doctor](/zh-TW/gateway/doctor) — 自動化健康狀態檢查與修復
- [通道疑難排解](/zh-TW/channels/troubleshooting) — 通道連線問題
- [排定的工作：疑難排解](/zh-TW/automation/cron-jobs#troubleshooting) — 排程與心跳偵測問題
