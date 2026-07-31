---
read_when:
    - 新增或修改 doctor 遷移作業
    - 引入破壞性設定變更
sidebarTitle: Doctor
summary: Doctor 命令：健康檢查、設定遷移與修復步驟
title: 醫生
x-i18n:
    generated_at: "2026-07-26T08:32:38Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 3f599553a2455759cd0fe56bafbc16948f7ab4d381d344b08a496bf19c9dc636
    source_path: gateway/doctor.md
    workflow: 16
---

`openclaw doctor` 是 OpenClaw 的修復與遷移工具。它會修正過時的設定／狀態、檢查健康情況，並提供可採取行動的修復步驟。

## 快速開始

```bash
openclaw doctor
```

### 無介面與自動化模式

<Tabs>
  <Tab title="--yes">
    ```bash
    openclaw doctor --yes
    ```

    不提示並接受預設值（包括適用時的重新啟動／服務／沙箱修復步驟）。

  </Tab>
  <Tab title="--fix">
    ```bash
    openclaw doctor --fix
    ```

    不提示並套用建議的修復（`--repair` 是別名）。

  </Tab>
  <Tab title="--lint">
    ```bash
    openclaw doctor --lint
    openclaw doctor --lint --json
    ```

    執行適用於 CI 或前置檢查自動化的結構化健康檢查。唯讀：不會
    提示、修復、遷移、重新啟動或寫入狀態。

  </Tab>
  <Tab title="--fix --force">
    ```bash
    openclaw doctor --fix --force
    ```

    同時套用積極的修復（會覆寫自訂的監督程式設定）。

  </Tab>
  <Tab title="--non-interactive">
    ```bash
    openclaw doctor --non-interactive
    ```

    不提示執行，且僅套用安全的遷移（設定正規化 +
    磁碟上的狀態移動）。略過需要人工確認的重新啟動／服務／沙箱
    動作。偵測到舊版狀態遷移時，仍會自動執行。

  </Tab>
  <Tab title="--deep">
    ```bash
    openclaw doctor --deep
    ```

    掃描系統服務，尋找額外安裝的閘道（launchd/systemd/schtasks）。

  </Tab>
</Tabs>

若要在寫入前檢閱變更，請先開啟設定檔：

```bash
cat ~/.openclaw/openclaw.json
```

## 唯讀 lint 模式

`openclaw doctor --lint` 是
`openclaw doctor --fix` 適合自動化使用的同類模式。兩者共用相同的 Doctor 規則登錄庫，但
選取及執行規則的方式並不相同：

| 模式                     | 提示   | 寫入設定／狀態     | 輸出                 | 用途                      |
| ------------------------ | --------- | ----------------------- | ---------------------- | ------------------------------- |
| `openclaw doctor`        | 是       | 否                      | 易讀的健康報告 | 人工檢查狀態         |
| `openclaw doctor --fix`  | 有時 | 是，依照修復原則 | 易讀的修復記錄    | 套用已核准的修復       |
| `openclaw doctor --lint` | 否        | 否                      | 結構化發現項目    | CI、前置檢查及審查關卡 |

預設的 `doctor --lint` 會執行廣泛且安全的自動化設定檔：執行靜態、本機，
且有助於 CI 或前置檢查輸出的檢查。它會略過選擇加入的檢查，包括建議性、
對環境敏感、依賴即時服務、帳號／工作區清查，或歷史清理的檢查。若要執行
完整的已登錄 lint 稽核（包括這些選擇加入的檢查），請使用 `doctor --lint --all`；若要
執行特定檢查，則使用 `--only <id>`。

`doctor --fix` 不會使用 lint 預設設定檔，也不接受
`--all`。它會執行 Doctor 的依序修復路徑：現代健康檢查可以提供
選用的 `repair()` 實作，而較舊的區域仍使用其舊版
Doctor 修復流程。部分 lint 發現項目刻意僅供診斷，因此某項檢查出現在 `--lint --all` 中，
並不代表 `--fix` 會變更該區域。
此合約將 `detect()`（回報發現項目）與 `repair()`（回報
變更／差異／副作用）分開，為未來的
`doctor --fix --dry-run` 保留實作空間，而不會將 lint 檢查變成變更規劃器。

部分內建檢查在內部預設為停用，使其仍可供
`--all`、`--only` 及 Doctor 修復流程使用，而不會成為預設
`doctor --lint` 自動化設定檔的一部分。每個發現項目仍會輸出其嚴重性
（`info`、`warning` 或 `error`）；預設選取與否並非嚴重性
等級。

```bash
openclaw doctor --lint
openclaw doctor --lint --severity-min warning
openclaw doctor --lint --json
openclaw doctor --lint --all
openclaw doctor --lint --only core/doctor/gateway-config --json
```

JSON 輸出欄位：

- `ok`：是否有任何發現項目達到所選的嚴重性門檻
- `checksRun` / `checksSkipped`：計數（因設定檔、`--only` 或 `--skip` 而略過）
- `findings`：包含 `checkId`、`severity`、`message`，以及選用的 `path`、`line`、`column`、`ocPath`、`source`、`target`、`requirement`、`fixHint` 的結構化診斷

結束代碼：

| 代碼 | 意義                                                  |
| ---- | -------------------------------------------------------- |
| `0`  | 沒有任何發現項目達到或超過所選門檻           |
| `1`  | 一個或多個發現項目達到所選門檻          |
| `2`  | 發現項目輸出前發生命令／執行階段失敗 |

旗標：

- `--severity-min info|warning|error`（預設為 `warning`）：控制輸出的內容，以及哪些情況會造成非零結束代碼。
- `--all`：執行每個已登錄的 lint 檢查，包括未納入預設自動化集合的選擇加入檢查。
- `--only <id>`（可重複）：僅執行指定 ID 的檢查；未知的 ID 會回報為錯誤發現項目。
- `--skip <id>`（可重複）：排除某項檢查，同時繼續執行其餘檢查。
- `--json`、`--severity-min`、`--all`、`--only` 及 `--skip` 需要 `--lint`；純 `openclaw doctor` 與 `--fix` 執行會拒絕這些旗標。

## 功能摘要

<AccordionGroup>
  <Accordion title="健康狀態、UI 與更新">
    - 針對 git 安裝的選用前置更新（僅限互動模式）。
    - UI 通訊協定時效檢查（當通訊協定結構描述較新時，重新建置 Control UI）。
    - 健康檢查 + 重新啟動提示。
    - 僅顯示有問題的 skill 與外掛附註；健康的清查資訊保留在 `openclaw skills check` 與 `openclaw plugins list` 中。

  </Accordion>
  <Accordion title="設定與遷移">
    - 舊版值結構的設定正規化。
    - 將 Talk 設定從舊版扁平 `talk.*` 欄位遷移至 `talk.provider` + `talk.providers.<provider>`。
    - 舊版 Chrome 擴充功能設定及 Chrome MCP 就緒狀態的瀏覽器遷移檢查。
    - OpenCode 提供者覆寫警告（`models.providers.opencode` / `opencode-zen` / `opencode-go`）。
    - 舊版 OpenAI Codex 提供者／設定檔遷移（`openai-codex` → `openai`），以及過時 `models.providers.openai-codex` 的遮蔽警告。
    - OpenAI Codex OAuth 設定檔的 OAuth TLS 先決條件檢查。
    - 當 `plugins.allow` 有限制，但工具原則仍要求萬用字元或外掛擁有的工具時，顯示外掛／工具允許清單警告。
    - 舊版磁碟狀態遷移（工作階段／代理程式目錄／WhatsApp 驗證）。
    - 舊版外掛資訊清單合約鍵遷移（`speechProviders`、`realtimeTranscriptionProviders`、`realtimeVoiceProviders`、`mediaUnderstandingProviders`、`imageGenerationProviders`、`videoGenerationProviders`、`webFetchProviders`、`webSearchProviders` → `contracts`）。
    - 舊版排程儲存區遷移（`jobId`、`schedule.cron`、頂層傳遞／承載資料欄位、承載資料 `provider`、`notify: true` 網路鉤子後援工作）。
    - 在 `agents.defaults`、`agents.entries.*` 及 `models.providers.*`（包括各模型項目）中修復 Codex 命令列介面執行階段釘選（`agentRuntime.id: "codex-cli"` → `"codex"`）。
    - 啟用外掛時清理過時的外掛設定；當 `plugins.enabled=false` 時，過時的外掛參照會保留為非作用中的隔離設定。

  </Accordion>
  <Accordion title="狀態與完整性">
    - 檢查工作階段鎖定檔並清理過時的鎖定。
    - 修復受影響的 2026.4.24 組建所建立、含重複提示重寫分支的工作階段文字記錄。
    - 偵測卡住的主要工作階段與子代理程式重新啟動復原墓碑。Doctor 會回報遭封鎖的工作階段，且僅修復與現有墓碑衝突的過時中止旗標；不會重新啟用自動復原。
    - 狀態完整性與權限檢查（工作階段、文字記錄、狀態目錄）。
    - 在本機執行時檢查設定檔權限（chmod 600）。
    - 模型驗證健康狀態：檢查 OAuth 到期時間、可重新整理即將到期的權杖，並回報驗證設定檔的冷卻／停用狀態。

  </Accordion>
  <Accordion title="閘道、服務與監督程式">
    - 啟用沙箱時修復沙箱映像檔。
    - 舊版服務遷移及額外閘道偵測。
    - Matrix 頻道舊版狀態遷移（在 `--fix` / `--repair` 模式中）。
    - 閘道執行階段檢查（服務已安裝但未執行；快取的 launchd 標籤）。
    - 頻道狀態警告（從執行中的閘道探查）。
    - 頻道特定的權限檢查位於 `openclaw channels capabilities` 下；例如，使用 `openclaw channels capabilities --channel discord --target channel:<channel-id>` 稽核 Discord 語音頻道權限。
    - 當閘道事件迴圈健康狀態降級，但本機終端介面用戶端仍在執行時，檢查 WhatsApp 回應能力；`--fix` 僅停止已驗證的本機終端介面用戶端。
    - 修復主要模型、後援模型、影像／影片生成模型、心跳偵測／子代理程式／壓縮覆寫、掛鉤、頻道模型覆寫及工作階段路由釘選中的舊版 `openai-codex/*` 模型參照；`--fix` 會將其重寫為 `openai/*`、將 `openai-codex:*` 驗證設定檔／順序遷移至 `openai:*`、移除過時的工作階段／整個代理程式執行階段釘選，並由修復後的有效路由判斷 Codex 是否相容。
    - 監督程式設定稽核（launchd/systemd/schtasks），可選擇進行修復。
    - 清理在安裝或更新期間擷取到 shell `HTTP_PROXY` / `HTTPS_PROXY` / `NO_PROXY` 值的閘道服務內嵌 Proxy 環境。
    - 閘道執行階段檢查（不支援的舊版 Bun 服務、版本管理工具路徑）。
    - 閘道連接埠衝突診斷（預設 `18789`）。

  </Accordion>
  <Accordion title="驗證、安全性與配對">
    - 開放 DM 原則的安全性警告。
    - 本機權杖模式的閘道驗證檢查（當不存在權杖來源時提供產生權杖的選項；不會覆寫權杖 SecretRef 設定）。
    - 裝置配對問題偵測（待處理的首次配對要求、待處理的角色／範圍升級、過時的本機裝置權杖快取偏移，以及已配對記錄的驗證偏移）。

  </Accordion>
  <Accordion title="工作區與 shell">
    - Linux 上的 systemd linger 檢查。
    - 工作區啟動檔案大小檢查（內容檔案的截斷／接近上限警告）。
    - 預設代理程式的 Skills 就緒狀態檢查；回報缺少執行檔、環境、設定或作業系統需求的已允許 skill，而 `--fix` 可在 `skills.entries` 中停用不可用的 skill。
    - Shell 自動補齊狀態檢查與自動安裝／升級。
    - 記憶搜尋嵌入提供者就緒狀態檢查（本機模型、遠端 API 金鑰或 QMD 執行檔）。
    - 原始碼安裝檢查（pnpm 工作區不符、缺少 UI 資產、缺少 tsx 執行檔）。
    - 寫入更新後的設定 + 精靈中繼資料。

  </Accordion>
</AccordionGroup>

## 夢境 UI 回填與重設

  Control UI 的夢境場景包含用於落地夢境整理工作流程的 **回填**、**重設** 和 **清除落地項目** 操作。這些操作使用閘道的 doctor 風格 RPC 方法，但**不**屬於 `openclaw doctor` 命令列介面的修復／遷移功能。

  | 操作           | 功能                                                                                                                                                              |
  | -------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------- |
  | 回填           | 掃描作用中工作區內的歷史 `memory/YYYY-MM-DD.md` 檔案、執行落地 REM 日記處理，並將可還原的回填項目寫入 `DREAMS.md`。 |
  | 重設           | 僅從 `DREAMS.md` 移除已標記的回填日記項目。                                                                                                  |
  | 清除落地項目   | 僅移除歷史重播中已暫存、僅限落地的短期項目，且這些項目尚未累積即時回憶或每日支援。                           |

  這些操作都不會編輯 `MEMORY.md`、執行完整的 doctor 遷移，也不會自行將落地候選項目暫存至即時短期提升儲存區。若要將落地的歷史重播內容送入一般的深度提升管道，請改用命令列介面流程：

  ```bash
  openclaw memory rem-backfill --path ./memory --stage-short-term
  ```

  這會將落地的持久候選項目暫存至短期夢境整理儲存區，同時讓 `DREAMS.md` 保持作為審查介面。

  ## 詳細行為與設計理由

  <AccordionGroup>
  <Accordion title="0. 選用更新（git 安裝）">
    如果這是 git 簽出版本，且 doctor 以互動模式執行，則會在執行 doctor 前提供更新選項（擷取／重定基底／建置）。
  </Accordion>
  <Accordion title="1. 設定正規化">
    Doctor 會將舊版值格式正規化為目前的結構描述。目前的 Talk 語音設定為 `talk.provider` + `talk.providers.<provider>`，即時語音設定則位於 `talk.realtime.*` 下。Doctor 會將舊版 `talk.voiceId` / `talk.voiceAliases` / `talk.modelId` / `talk.outputFormat` / `talk.apiKey` 格式重寫至提供者對應表，並將舊版頂層即時選擇器（`talk.mode`、`talk.transport`、`talk.brain`、`talk.model`、`talk.voice`）重寫至 `talk.realtime`。

    當 `plugins.allow` 非空白，且工具原則使用萬用字元或外掛所擁有的工具項目時，Doctor 也會發出警告。`tools.allow: ["*"]` 僅比對來自實際載入之外掛的工具；它不會略過排他性的外掛允許清單。

  </Accordion>
  <Accordion title="2. 舊版設定鍵遷移">
    當設定包含具有有效遷移機制的已淘汰鍵時，其他命令會拒絕執行，並要求你執行 `openclaw doctor`。Doctor 會說明找到哪些舊版鍵、顯示所套用的遷移，並使用更新後的結構描述重寫 `~/.openclaw/openclaw.json`。閘道啟動時會拒絕舊版設定格式，並要求你執行 `openclaw doctor --fix`；它不會在啟動時重寫 `openclaw.json`。排程工作儲存區的遷移也由 `openclaw doctor --fix` 處理。

    <Note>
      Doctor 僅會在設定鍵淘汰後約兩個月內提供自動遷移。
      更早期的舊版鍵（例如原始的
      `routing.queue`、`routing.bindings`、`routing.agents`/`defaultAgentId`、
      `routing.transcribeAudio`、頂層 `agent.*`，或多代理程式設定格式推出前的頂層 `identity`）
      已不再提供遷移路徑；目前使用這些鍵的設定會直接驗證失敗，而不會進行重寫。
      請依照目前的設定參考資料手動修正這些鍵，doctor 才能繼續執行。
    </Note>

    有效的遷移：

    | 舊版鍵值                                                                                    | 目前鍵值                                                                 |
    | ----------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------- |
    | `routing.allowFrom`                                                                              | `channels.whatsapp.allowFrom`                                                |
    | `routing.groupChat.requireMention`                                                               | `channels.whatsapp/telegram/imessage.groups."*".requireMention`             |
    | `routing.groupChat.historyLimit`                                                                 | `messages.groupChat.historyLimit`                                            |
    | `routing.groupChat.mentionPatterns`                                                              | `messages.groupChat.mentionPatterns`                                         |
    | `channels.telegram.requireMention`                                                               | `channels.telegram.groups."*".requireMention`                               |
    | `channels.webchat`, `gateway.webchat`                                                            | 已移除（WebChat 已停用）                                                 |
    | `channels.feishu.accounts.<accountId>.botName`                                                   | `channels.feishu.accounts.<accountId>.name`                                 |
    | `session.threadBindings.ttlHours`, `channels.<id>.threadBindings.ttlHours`（以及各帳號）      | `...threadBindings.idleHours`                                               |
    | 舊版 `talk.voiceId`/`talk.voiceAliases`/`talk.modelId`/`talk.outputFormat`/`talk.apiKey`        | `talk.provider` + `talk.providers.<provider>`                               |
    | 舊版頂層即時 Talk 選擇器（`talk.mode`/`talk.transport`/`talk.brain`/`talk.model`/`talk.voice`） | `talk.realtime`                                                              |
    | `messages.tts`                                                                                  | 頂層 `tts`                                                              |
    | `messages.tts.<provider>`（`openai`/`elevenlabs`/`microsoft`/`edge`）                             | `tts.providers.<provider>`                                                   |
    | `messages.tts.provider: "edge"` / `messages.tts.providers.edge`                                  | `tts.provider: "microsoft"` / `tts.providers.microsoft`                    |
    | `tools.exec.security` + `tools.exec.ask`                                                         | `tools.exec.mode`                                                            |
    | `session.idleMinutes`                                                                            | `session.reset.idleMinutes`                                                  |
    | 包含明確頻道區塊的 `messages.responsePrefix`                                           | 複製至已設定頻道／帳號的 `responsePrefix`；為隱含／自訂頻道保留全域備援 |
    | `web.enabled`                                                                                    | `channels.whatsapp.enabled`                                                  |
    | `meta.lastTouchedAt`、鉤子安裝項目、排程儲存區、內建探索、全域 TTS 偏好設定路徑            | 共用 SQLite 狀態                                                       |
    | TTS 語音者欄位 `voice`/`voiceName`/`voiceId`                                                 | `speakerVoice`/`speakerVoiceId`                                              |
    | `channels.<id>.tts.<provider>` / `channels.<id>.accounts.<accountId>.tts.<provider>`（Discord 以外的所有頻道）                                          | `...tts.providers.<provider>`                                                |
    | `channels.<id>.voice.tts.<provider>` / `channels.<id>.accounts.<accountId>.voice.tts.<provider>`（包括 Discord 在內的所有頻道）                          | `...voice.tts.providers.<provider>`                                          |
    | `plugins.entries.voice-call.config.tts.<provider>`（`openai`/`elevenlabs`/`microsoft`/`edge`）     | `plugins.entries.voice-call.config.tts.providers.<provider>`                |
    | `plugins.entries.voice-call.config.tts.provider: "edge"` / `...tts.providers.edge`                | `provider: "microsoft"` / `...tts.providers.microsoft`                      |
    | `plugins.entries.voice-call.config.provider: "log"`                                              | `"mock"`                                                                      |
    | `plugins.entries.voice-call.config.twilio.from`                                                  | `plugins.entries.voice-call.config.fromNumber`                              |
    | `plugins.entries.voice-call.config.streaming.sttProvider`                                        | `plugins.entries.voice-call.config.streaming.provider`                      |
    | `plugins.entries.voice-call.config.streaming.openaiApiKey`/`sttModel`/`silenceDurationMs`/`vadThreshold` | `plugins.entries.voice-call.config.streaming.providers.openai.*`             |
    | `models.providers.*.api: "openai"`                                                               | `"openai-completions"`（閘道啟動時也會略過 `api` 為未來／未知列舉值的供應商，而不是採取封閉失敗） |
    | `browser.ssrfPolicy.allowPrivateNetwork`                                                         | `browser.ssrfPolicy.dangerouslyAllowPrivateNetwork`                          |
    | `browser.profiles.*.driver: "extension"`                                                         | `"existing-session"`                                                          |
    | `browser.relayBindHost`                                                                          | 已移除（舊版 Chrome 擴充功能轉送設定）                             |
    | `mcp.servers.*.type`（命令列介面原生別名）                                                        | `mcp.servers.*.transport`                                                    |
    | `mcp.servers.*.disabled`                                                                         | 反向 `mcp.servers.*.enabled`                                              |
    | MCP 逾時別名 `connectTimeout`/`connect_timeout`/`timeout`                                 | `connectionTimeoutMs`/`requestTimeoutMs`                                    |
    | MCP 蛇形命名伺服器欄位                                                                     | 駝峰命名 MCP 伺服器欄位                                                   |
    | `tools.media.image/audio/video.models`                                                           | 標有能力標籤的 `tools.media.models`                                        |
    | `tools.media.asyncCompletion`                                                                    | 已移除                                                                       |
    | `tools.message.allowCrossContextSend`                                                            | `tools.message.crossContext`                                                  |
    | 媒體模型 `deepgram` 選項                                                                   | `providerOptions.deepgram`                                                    |
    | `talk.realtime.voice`、Discord 即時 `voice`                                                 | `speakerVoice`                                                                |
    | `agents.defaults.pdfMaxBytesMb`                                                                  | `agents.defaults.pdfMaxMb`                                                    |
    | `tools.exec.timeoutSec`                                                                          | `tools.exec.timeoutSeconds`                                                   |
    | `browser.ssrfPolicy.hostnameAllowlist`                                                           | 支援萬用字元的 `browser.ssrfPolicy.allowedHostnames`                          |
    | 沙箱瀏覽器 `enableNoVnc`                                                                    | `noVncEnabled`                                                                |
    | 根層級 `media`                                                                                     | `attachments`                                                                |
    | 頻道／帳號 `heartbeat` 可見性區塊                                                   | `heartbeatVisibility`                                                         |
    | `channels.slack.identity`                                                                        | `channels.slack.postAs`                                                       |
    | 根層級 `audit`                                                                                     | `logging.audit`                                                               |
    | `gateway.nodes.skills.enabled`                                                                   | `gateway.nodes.allowSkills`                                                   |
    | `gateway.nodes.allowCommands`/`denyCommands`                                                    | `gateway.nodes.commands.allow`/`deny`                                         |
    | 生成模型預設值                                                                       | `agents.defaults.mediaModels.{image,video,music}`                              |
    | 已淘汰的最終版面調校控制項                                                               | 內建預設行為                                                     |
    | `channels.whatsapp.messagePrefix` 和舊版 `messages.messagePrefix`                            | `channels.whatsapp.responsePrefix`                                            |
    | `channels.whatsapp.ackReaction`                                                                  | 全域 `messages.ackReaction`，以及可翻譯處的 `ackReactionScope`        |
    | `cron.failureDestination`                                                                        | `cron.failureAlert` 上的目的地欄位                                     |
    | `gateway.controlUi.chatMessageMaxWidth`、僅供呈現的 `ui.prefs` 鍵值                       | 已移除（文字縮放、聊天寬度和即時側邊欄活動皆為瀏覽器本機設定） |
    | `agents.list`                                                                                    | 以鍵值索引的 `agents.entries`                                                        |
    | 頂層 `defaultModel`                                                                         | `agents.defaults.model`                                                      |
    | `messages.messagePrefix`                                                                         | `channels.whatsapp.responsePrefix`                                            |
    | `session.maintenance.pruneDays`, `session.resetByType.dm`                                        | `session.maintenance.pruneAfter`, `session.resetByType.direct`               |
    | 頂層 `tui`                                                                                  | 已移除（終端介面頁尾使用精簡預設值）                            |
    | `plugins.entries.codex.config.codexDynamicToolsProfile`                                          | 已移除（Codex app-server 一律讓 Codex 原生工作區工具維持原生） |
    | `commands.modelsWrite`                                                                           | 已移除（`/models add` 已棄用）                                       |
    | `agents.defaults/list[].silentReplyRewrite`, `surfaces.*.silentReplyRewrite`                     | 已移除（不再將完全相符的 `NO_REPLY` 改寫為可見的備援文字）  |
    | `agents.defaults/list[].systemPromptOverride`                                                    | 已移除（OpenClaw 負責產生系統提示詞）                        |
    | `agents.defaults/list[].embeddedPi`                                                              | `embeddedAgent`                                                              |
    | `agents.defaults/list[].sandbox.perSession`                                                      | `sandbox.scope`                                                              |
    | `agents.defaults.llm`                                                                             | 已移除（對緩慢模型／供應商的逾時請使用 `models.providers.<id>.timeoutSeconds`，並保持低於代理程式／執行逾時上限） |
    | 頂層 `memorySearch`、`agents.defaults.memorySearch`                                         | `memory.search`                                                             |
    | `agents.entries.*.memorySearch`                                                                     | `agents.entries.*.memory.search`                                               |
    | `memorySearch.provider: "auto"`                                                                  | `"openai"`                                                                    |
    | `memorySearch.store.path`（任何層級）                                                            | 已移除（記憶索引存放於各個代理程式資料庫中）                       |
    | 頂層 `heartbeat`                                                                            | `agents.defaults.heartbeat` / `channels.defaults.heartbeat`                 |
    | `plugins.openai-codex` 原則 ID                                                                | `plugins.openai`                                                             |
    | `tools.web.x_search.apiKey`                                                                      | `plugins.entries.xai.config.webSearch.apiKey`                               |
    | `session.maintenance.rotateBytes`、`session.parentForkMaxTokens`                                 | 已移除（已棄用）                                                        |
    | 於 2026.7 淘汰的執行階段與頻道調校選項                                               | 已移除（套用內建的正式環境預設值）                               |

    <Note>
      上方的 `plugins.entries.voice-call.config.*` 列會在每次載入設定時，由語音通話外掛本身正規化，而不是由 `openclaw
      doctor` 正規化。該外掛也會記錄啟動警告並指向 `openclaw
      doctor --fix`，但 doctor 目前不會針對這些鍵重寫
      `openclaw.json`；在執行階段套用變更的是外掛本身的正規化機制。
    </Note>

    多帳號頻道的帳號預設值指引：

    - 如果設定了兩個以上的 `channels.<channel>.accounts` 項目，但未設定 `channels.<channel>.defaultAccount` 或 `accounts.default`，doctor 會警告備援路由可能選到非預期的帳號。
    - 如果將 `channels.<channel>.defaultAccount` 設為未知的帳號 ID，doctor 會發出警告並列出已設定的帳號 ID。

  </Accordion>
  <Accordion title="2b. OpenCode 提供者覆寫">
    如果你手動新增了 `models.providers.opencode`、`opencode-zen` 或 `opencode-go`，它會覆寫來自 `openclaw/plugin-sdk/llm` 的內建 OpenCode 目錄。這可能會強制模型使用錯誤的 API，或將成本歸零。Doctor 會發出警告，讓你移除覆寫並還原各模型的 API 路由與成本。
  </Accordion>
  <Accordion title="2c. 瀏覽器遷移與 Chrome MCP 就緒狀態">
    如果你的瀏覽器設定仍指向已移除的 Chrome 擴充功能路徑，doctor 會將其正規化為目前主機本機的 Chrome MCP 附加模型（`browser.profiles.*.driver: "extension"` → `"existing-session"`；移除 `browser.relayBindHost`）。

    當你使用 `defaultProfile: "user"` 或已設定的 `existing-session` 設定檔時，doctor 也會稽核主機本機的 Chrome MCP 路徑：

    - 針對預設自動連線設定檔，檢查同一台主機上是否已安裝 Google Chrome
    - 檢查偵測到的 Chrome 版本，並在低於 Chrome 144 時發出警告
    - 提醒你在瀏覽器檢查頁面中啟用遠端偵錯（例如 `chrome://inspect/#remote-debugging`、`brave://inspect/#remote-debugging` 或 `edge://inspect/#remote-debugging`）

    Doctor 無法代你啟用 Chrome 端的設定。主機本機的 Chrome MCP 仍需要閘道／節點主機上本機執行的 Chromium 系瀏覽器 144+、已啟用遠端偵錯，並且已在瀏覽器中核准第一次附加的同意提示。

    此處的就緒狀態只涵蓋本機附加的必要條件。Existing-session 仍受目前的 Chrome MCP 路由限制；`responsebody`、PDF 匯出、下載攔截及批次動作等進階路由，仍需要受管理的瀏覽器或原始 CDP 設定檔。此檢查不適用於 Docker、沙箱、遠端瀏覽器或其他無頭流程；這些流程會繼續使用原始 CDP。

  </Accordion>
  <Accordion title="2d. OAuth TLS 必要條件">
    設定 OpenAI Codex OAuth 設定檔時，doctor 會探測 OpenAI 授權端點，以確認本機 Node/OpenSSL TLS 堆疊能夠驗證憑證鏈。如果探測因憑證錯誤而失敗（例如 `UNABLE_TO_GET_ISSUER_CERT_LOCALLY`、憑證已過期或自我簽署憑證），doctor 會輸出平台專屬的修正指引。在使用 Homebrew Node 的 macOS 上，修正方式通常是 `brew postinstall ca-certificates`。使用 `--deep` 時，即使閘道運作正常，也會執行探測。
  </Accordion>
  <Accordion title="2e. Codex OAuth 提供者覆寫">
    如果你先前在 `models.providers.openai-codex` 下新增了舊版 OpenAI 傳輸設定，這些設定可能會遮蔽內建的 Codex OAuth 提供者路徑。Doctor 同時看到這些舊傳輸設定與 Codex OAuth 時會發出警告，讓你移除或重寫過時的傳輸覆寫，並還原目前的路由行為。自訂 Proxy 與僅限標頭的覆寫仍受支援，且不會觸發此警告，但這些自行定義的要求路由不符合隱含 Codex 選擇的資格。
  </Accordion>
  <Accordion title="2f. Codex 路由修復">
    Doctor 會檢查舊版 `openai-codex/*` 模型參照。原生 Codex 控制框架路由使用標準的 `openai/*` 模型參照，但僅憑前綴絕不會選取 Codex。當執行階段政策未設定或為 `auto` 時，只有完全相符的官方 HTTPS Platform Responses 或 ChatGPT Responses 路由，且沒有自行定義的要求覆寫，才符合資格。請參閱 [OpenAI 隱含代理程式執行階段](/zh-TW/providers/openai#implicit-agent-runtime)。

    在 `--fix` / `--repair` 模式下，doctor 會重寫受影響的預設代理程式與個別代理程式參照，包括主要模型、備援模型、影像／影片生成模型、心跳偵測／子代理程式／壓縮覆寫、掛鉤、頻道模型覆寫，以及過時的持續保存工作階段路由狀態：

    - `openai-codex/gpt-*` 會變成 `openai/gpt-*`。
    - 對於已修復的代理程式模型參照，Codex 使用意圖會移至以提供者／模型為範圍的 `agentRuntime.id: "codex"` 項目。
    - 系統會移除過時的完整代理程式執行階段設定與持續保存的工作階段執行階段固定項目，因為執行階段選擇是以提供者／模型為範圍。
    - 除非已修復的舊版模型參照需要 Codex 路由以保留舊有驗證路徑，否則會保留現有的提供者／模型執行階段政策。
    - 會保留現有模型備援清單並重寫其中的舊版項目；複製的個別模型設定會從舊版鍵移至標準的 `openai/*` 鍵。
    - 系統會在所有探索到的代理程式工作階段儲存區中，修復持續保存的工作階段 `modelProvider`/`providerOverride`、`model`/`modelOverride`、備援通知及驗證設定檔固定項目。
    - Doctor 會另外在 `agents.defaults`、`agents.entries.*` 與 `models.providers.*` 模型項目中，將過時的 `agentRuntime.id: "codex-cli"` 固定項目（一個不同的舊版執行階段 ID）修復為 `"codex"`。
    - `/codex ...` 表示「從聊天中控制或繫結原生 Codex 對話」。
    - `/acp ...` 或 `runtime: "acp"` 表示「使用外部 ACP/acpx 轉接器」。

  </Accordion>
  <Accordion title="2g. 工作階段路由清理">
    在你將已設定的模型或執行階段移離 Codex 等由外掛擁有的路由後，doctor 也會掃描探索到的代理程式工作階段儲存區，尋找過時的自動建立路由狀態。

    當擁有這些狀態的路由已不再設定時，`openclaw doctor --fix` 可以清除自動建立的過時狀態，例如 `modelOverrideSource: "auto"` 模型固定項目、執行階段模型中繼資料、固定的控制框架 ID、命令列介面工作階段繫結，以及自動驗證設定檔覆寫。明確的使用者或舊版工作階段模型選擇會回報供人工審查，並保持不變；若不再需要該路由，請使用 `/model ...`、`/new` 切換，或重設工作階段。

  </Accordion>
  <Accordion title="3. 舊版狀態遷移（磁碟配置）">
    Doctor 可以將較舊的磁碟配置遷移至目前的結構：

    - 工作階段儲存區與對話記錄：從 `~/.openclaw/sessions/` 遷移至 `~/.openclaw/agents/<agentId>/sessions/`
    - 代理程式目錄：從 `~/.openclaw/agent/` 遷移至 `~/.openclaw/agents/<agentId>/agent/`
    - WhatsApp 驗證狀態（Baileys）：從舊版 `~/.openclaw/credentials/*.json`（`oauth.json` 除外）遷移至 `~/.openclaw/credentials/whatsapp/<accountId>/...`（預設帳號 ID：`default`）
    - 已簽署的裝置身分：從 `~/.openclaw/identity/device.json` 遷移至 `state/openclaw.sqlite` 中的 `primary` `device_identities` 列；獨立的裝置驗證檔案保持不變

    這些遷移採盡力而為且具等冪性；當 doctor 將任何舊版資料夾保留作為備份時，會發出警告。閘道／命令列介面也會在啟動時自動遷移舊版工作階段與代理程式目錄，因此無須手動執行 doctor，歷程記錄／驗證／模型就會移至個別代理程式路徑。WhatsApp 驗證刻意只透過 `openclaw doctor` 遷移。Talk 提供者／提供者對應表正規化會依結構相等性比較，因此僅鍵順序不同的差異不再重複觸發無實質變更的 `doctor --fix` 變更。

  </Accordion>
  <Accordion title="3a. 舊版外掛資訊清單遷移">
    Doctor 會掃描所有已安裝的外掛資訊清單，尋找已淘汰的頂層功能鍵（`speechProviders`、`realtimeTranscriptionProviders`、`realtimeVoiceProviders`、`mediaUnderstandingProviders`、`imageGenerationProviders`、`videoGenerationProviders`、`webFetchProviders`、`webSearchProviders`）。發現時，它會提議將這些鍵移至 `contracts` 物件，並就地重寫資訊清單檔案。此遷移具等冪性；如果 `contracts` 已有相同值，系統會移除舊版鍵，而不會重複資料。
  </Accordion>
  <Accordion title="3b. 舊版排程儲存區遷移">
    在將標準列匯入 SQLite 前，doctor 也會檢查舊版排程工作儲存區（`~/.openclaw/cron/jobs.json`）中的舊工作結構。

    目前的排程清理包括：

    - `jobId` → `id`
    - `schedule.cron` → `schedule.expr`
    - 頂層承載資料欄位（`message`、`model`、`thinking`，……）→ `payload`
    - 頂層遞送欄位（`deliver`、`channel`、`to`、`provider`，……）→ `delivery`
    - 承載資料中的 `provider` 遞送別名 → 明確的 `delivery.channel`
    - 舊版 `notify: true` 網路鉤子備援工作 → 若已淘汰的原始 `cron.webhook` 值有效，則改為明確的網路鉤子遞送；公告工作會保留其聊天遞送，並取得 `delivery.completionDestination`。接著 doctor 會移除舊設定鍵。如果沒有可用的舊版網路鉤子，對於沒有目標的工作，系統會移除無作用的頂層 `notify` 標記（保留現有遞送，包括公告），因為執行階段遞送從不讀取該標記。

    閘道也會在載入時清理格式錯誤的排程列，讓有效工作繼續執行。在從 `jobs.json` 移除之前，原始格式錯誤的列會複製到有效儲存區旁的 `jobs-quarantine.json`；doctor 會回報已隔離的列，讓你可以人工審查或修復。

    閘道啟動時會正規化執行階段投影並忽略頂層 `notify` 標記，但會保留持續保存的排程狀態供 doctor 修復。對於沒有遷移目標的工作（`delivery.mode` 為無／不存在、舊版網路鉤子目標無法使用，或已有公告／聊天遞送），doctor 會移除無作用的標記，並保持現有遞送不變，因此重複執行 `doctor --fix` 時，不再針對同一工作反覆發出警告。

    在 Linux 上，當使用者的 crontab 仍呼叫舊版 `~/.openclaw/bin/ensure-whatsapp.sh` 時，doctor 也會發出警告。目前的 OpenClaw 不維護該主機本機指令碼；當排程無法連線至 systemd 使用者匯流排時，它可能會將錯誤的 `Gateway inactive` 訊息寫入 `~/.openclaw/logs/whatsapp-health.log`。請使用 `crontab -e` 移除過時的 crontab 項目；目前的健康狀態檢查請使用 `openclaw channels status --probe`、`openclaw doctor` 與 `openclaw gateway status`。

  </Accordion>
  <Accordion title="3c. 工作階段鎖定清理">
    Doctor 會掃描每個代理程式工作階段目錄，尋找工作階段異常結束時遺留的過期寫入鎖定檔。對於找到的每個鎖定檔，它會回報：路徑、PID、PID 是否仍在執行、鎖定時間，以及是否視為過期（PID 已終止、擁有者中繼資料格式錯誤、超過 30 分鐘，或已證實執行中的 PID 屬於非 OpenClaw 程序）。在 `--fix` / `--repair` 模式下，它會自動移除擁有者已終止、孤立、遭重複使用、格式錯誤且過舊，或不屬於 OpenClaw 的鎖定。仍由執行中的 OpenClaw 程序擁有的舊鎖定會被回報但保留原狀，確保 doctor 不會中斷仍在活動中的逐字稿寫入程序。
  </Accordion>
  <Accordion title="3d. 工作階段逐字稿分支修復">
    Doctor 會掃描代理程式工作階段 JSONL 檔案，尋找由 2026.4.24 提示詞逐字稿重寫錯誤所建立的重複分支結構：一個含有 OpenClaw 內部執行階段情境、已遭捨棄的使用者回合，以及一個包含相同可見使用者提示詞的活動同層分支。在 `--fix` / `--repair` 模式下，doctor 會在每個受影響檔案的原始檔旁建立備份，並將逐字稿重寫至活動分支，使閘道歷程與記憶讀取器不再看到重複回合。
  </Accordion>
  <Accordion title="4. 狀態完整性檢查（工作階段持續保存、路由與安全性）">
    狀態目錄是操作中樞。若它消失，除非你在其他位置備有副本，否則將失去工作階段、認證資訊、日誌與設定。

    Doctor 會檢查：

    - **狀態目錄不存在**：警告災難性的狀態遺失、提示重新建立目錄，並提醒你它無法復原遺失的資料。
    - **狀態目錄權限**：驗證是否可寫入；提供修復權限的選項（偵測到擁有者／群組不符時，會顯示 `chown` 提示）。
    - **macOS 雲端同步狀態目錄**：當狀態解析至 iCloud Drive（`~/Library/Mobile Documents/com~apple~CloudDocs/...`）或 `~/Library/CloudStorage/...` 下方時發出警告，因為同步支援的路徑可能造成較慢的 I/O，以及鎖定／同步競爭。
    - **Linux SD 或 eMMC 狀態目錄**：當狀態解析至 `mmcblk*` 掛載來源時發出警告，因為以 SD／eMMC 為基礎的隨機 I/O 可能較慢，且在工作階段與認證資訊寫入時耗損得更快。
    - **Linux 揮發性狀態目錄**：當狀態解析至 `tmpfs` 或 `ramfs` 時發出警告，因為工作階段、認證資訊、設定與 SQLite 狀態（含 WAL／日誌附屬檔案）會在重新啟動時消失。Docker `overlay` 掛載不會刻意標示，因為只要容器仍存在，其可寫入層就會在主機重新啟動後持續保留。
    - **工作階段目錄不存在**：必須有 `sessions/` 與工作階段存放區目錄，才能持續保存歷程並避免 `ENOENT` 當機。
    - **逐字稿不符**：近期工作階段項目缺少逐字稿檔案時發出警告。
    - **主要工作階段「單行 JSONL」**：主要逐字稿只有一行時標示異常（歷程未持續累積）。
    - **多個狀態目錄**：當多個家目錄中存在多個 `~/.openclaw` 資料夾，或 `OPENCLAW_STATE_DIR` 指向其他位置時發出警告（歷程可能分散在不同安裝之間）。
    - **遠端模式提醒**：若為 `gateway.mode=remote`，doctor 會提醒你在遠端主機上執行（狀態儲存在該處）。
    - **設定檔權限**：若群組／所有人可讀取 `~/.openclaw/openclaw.json`，則發出警告並提供將權限收緊為 `600` 的選項。

  </Accordion>
  <Accordion title="5. 模型驗證健康狀態（OAuth 到期）">
    Doctor 會檢查驗證存放區中的 OAuth 設定檔，在權杖即將到期／已到期時發出警告，並可在安全的情況下重新整理權杖。若 Anthropic OAuth／權杖設定檔已過期，它會建議使用 Anthropic API 金鑰或 Anthropic 設定權杖路徑。重新整理提示只會在互動式執行（TTY）時顯示；`--non-interactive` 會略過重新整理嘗試。

    OAuth 重新整理永久失敗時（例如 `refresh_token_reused`、`invalid_grant`，或供應商要求你重新登入），doctor 會回報必須重新驗證，並顯示要執行的確切 `openclaw models auth login --provider ...` 命令。

    Doctor 也會回報因短暫冷卻期（速率限制／逾時／驗證失敗）或較長時間停用（帳務／額度失敗）而暫時無法使用的驗證設定檔。

    權杖儲存在 macOS Keychain 中的舊版 Codex OAuth 設定檔（採用檔案型附屬配置之前的舊版初始設定）只能由 doctor 修復。請從互動式終端機執行一次 `openclaw doctor --fix`，將由 Keychain 支援的舊版權杖就地遷移至 `auth-profiles.json`；之後，內嵌回合（Telegram、排程、子代理程式分派）會將其解析為標準 OpenAI OAuth 設定檔。

  </Accordion>
  <Accordion title="6. Hooks 模型驗證">
    若已設定 `hooks.gmail.model`，doctor 會根據目錄與允許清單驗證模型參照，並在無法解析或不允許時發出警告。
  </Accordion>
  <Accordion title="7. 沙箱映像修復">
    啟用沙箱時，doctor 會檢查 Docker 映像，若目前的映像不存在，會提供建置映像或切換至舊版名稱的選項。
  </Accordion>
  <Accordion title="7b. 外掛安裝清理">
    在 `openclaw doctor --fix` / `openclaw doctor --repair` 模式下，Doctor 會移除 OpenClaw 產生的舊版外掛相依套件暫存狀態：過期的已產生相依套件根目錄、舊安裝暫存目錄、先前內建外掛相依套件修復程式碼留下的套件本機殘留項目，以及孤立或已復原、受管理的內建 `@openclaw/*` 外掛 npm 副本，這些副本可能遮蔽目前的內建資訊清單。Doctor 也會將主機的 `openclaw` 套件重新連結至宣告 `peerDependencies.openclaw` 的受管理 npm 外掛，確保 `openclaw/plugin-sdk/*` 等套件本機執行階段匯入在更新或 npm 修復後仍能解析。

    當設定參照可下載的外掛，但本機外掛登錄找不到它們時，Doctor 也可以重新安裝這些遺失的外掛（實質 `plugins.entries`、已設定的頻道／供應商／搜尋設定、已設定的代理程式執行階段）。套件更新期間，doctor 會在核心套件正被替換時避免重新安裝外掛套件；若更新後已設定的外掛仍需復原，請再次執行 `openclaw doctor --fix`。除了下述容器映像啟動例外，閘道啟動與設定重新載入不會執行套件修復；外掛安裝仍須明確透過 doctor／安裝／更新作業執行。

    容器化閘道啟動有一項範圍有限的升級例外：當 `openclaw gateway run` 在新的 OpenClaw 版本上啟動時，它會先執行安全的狀態遷移與既有的核心更新後外掛收斂程序，再進入就緒狀態，之後記錄每個版本的檢查點。此啟動程序可清理過期的內建外掛記錄、修復本機外掛連結、在收斂路徑需要時重新安裝已設定的外掛套件，並檢查活動中的外掛承載內容。若啟動時無法安全修復，請先使用 `openclaw doctor --fix`，針對相同的已掛載狀態／設定執行同一映像一次，再正常重新啟動容器。

  </Accordion>
  <Accordion title="8. 閘道服務遷移與清理提示">
    Doctor 會偵測舊版閘道服務（launchd／systemd／schtasks），提供移除這些服務並使用目前閘道連接埠安裝 OpenClaw 服務的選項。它也可以掃描額外的類閘道服務並顯示清理提示。以設定檔命名的 OpenClaw 閘道服務視為正式服務，不會標示為「額外」。

    在 Linux 上，若缺少使用者層級的閘道服務，但存在系統層級的 OpenClaw 閘道服務，doctor 不會自動安裝第二個使用者層級服務。請使用 `openclaw gateway status --deep` 或 `openclaw doctor --deep` 檢查，然後移除重複項目；若閘道生命週期由系統監督程式管理，則設定 `OPENCLAW_SERVICE_REPAIR_POLICY=external`。

  </Accordion>
  <Accordion title="8b. 啟動時 Matrix 遷移">
    當 Matrix 頻道帳號有待處理或可執行的舊版狀態遷移時，doctor（在 `--fix` / `--repair` 模式下）會建立遷移前快照，然後執行盡力而為的遷移步驟：舊版 Matrix 狀態遷移與舊版加密狀態準備。這兩個步驟都不會造成致命錯誤；錯誤會記錄至日誌，啟動程序則繼續。在唯讀模式（使用 `openclaw doctor` 且未使用 `--fix`）下，會完全略過此檢查。
  </Accordion>
  <Accordion title="8c. 裝置配對與驗證偏移">
    Doctor 會在正常健康狀態檢查期間檢查裝置配對狀態，並回報：

    - 待處理的首次配對要求
    - 已配對裝置待處理的角色或範圍升級
    - 裝置 ID 仍相符，但裝置身分不再符合已核准記錄時的公開金鑰不符修復
    - 已配對記錄缺少已核准角色的活動權杖
    - 範圍偏離已核准配對基準的已配對權杖
    - 目前機器上早於閘道端權杖輪替，或帶有過期範圍中繼資料的本機快取裝置權杖項目

    Doctor 不會自動核准配對要求，也不會自動輪替裝置權杖。它會顯示確切的後續步驟：

    - 使用 `openclaw devices list` 檢查待處理要求
    - 使用 `openclaw devices approve <requestId>` 核准確切要求
    - 使用 `openclaw devices rotate --device <deviceId> --role <role>` 輪替出新權杖
    - 使用 `openclaw devices remove <deviceId>` 移除並重新核准過期記錄

    這可區分首次配對、待處理的角色／範圍升級，以及過期權杖／裝置身分偏移，補上常見的「已配對但仍顯示需要配對」漏洞。

  </Accordion>
  <Accordion title="9. 安全性警告">
    Doctor 只會在發現警告時顯示安全性注意事項，例如供應商允許私訊卻未設定允許清單，或原則設定存在危險。使用 `openclaw security audit` 可查看完整的安全性清單。
  </Accordion>
  <Accordion title="10. systemd 持續執行（Linux）">
    若以 systemd 使用者服務執行，doctor 會確保已啟用持續執行，使閘道在登出後仍保持運作。
  </Accordion>
  <Accordion title="11. 工作區狀態（Skills、外掛與 TaskFlow）">
    Doctor 會顯示預設代理程式的問題與處理方式，而非健康狀態清單：

    - **Skills**：列出已允許但無法使用的 skill 名稱；使用 `openclaw skills check` 可查看需求詳細資料與完整計數。
    - **外掛**：僅回報發生錯誤的外掛 ID；使用 `openclaw plugins list` 可查看已載入、已匯入、已停用及內建外掛清單。
    - **外掛相容性警告**：標示與目前執行階段存在相容性問題的外掛。
    - **外掛診斷**：呈現外掛登錄在載入時發出的任何警告或錯誤。
    - **TaskFlow 復原**：呈現需要手動檢查或取消的可疑受管理 TaskFlow。
    - **Claude 命令列介面**：僅回報二進位檔、驗證、設定檔、工作區或專案目錄問題；省略健康狀態探測詳細資料。

  </Accordion>
  <Accordion title="11b. 啟動載入檔案大小">
    Doctor 會檢查工作區啟動載入檔案（例如 `AGENTS.md`、`CLAUDE.md` 或其他注入的情境檔案）是否接近或超過設定的字元預算。它會回報每個檔案的原始字元數與注入字元數、截斷百分比、截斷原因（`max/file` 或 `max/total`），以及注入字元總數占總預算的比例。當檔案遭截斷或接近限制時，doctor 會顯示調整 `agents.defaults.bootstrapMaxChars` 與 `agents.defaults.bootstrapTotalMaxChars` 的提示。
  </Accordion>
  <Accordion title="11c. Shell 自動完成">
    Doctor 會檢查目前的 shell（zsh、bash、fish 或 PowerShell）是否已安裝 Tab 鍵自動完成：

    - 如果 shell 設定檔使用緩慢的動態補全模式（`source <(openclaw completion ...)`），doctor 會將其升級為速度更快的快取檔案版本。
    - 如果設定檔中已設定補全，但快取檔案遺失，doctor 會自動重新產生快取。
    - 如果完全未設定補全，doctor 會提示安裝（僅限互動模式；使用 `--non-interactive` 時會略過）。

    執行 `openclaw completion --write-state` 可手動重新產生快取。

  </Accordion>
  <Accordion title="11d. 清理過時的頻道外掛">
    當 `openclaw doctor --fix` 移除遺失的頻道外掛時，也會移除參照該外掛而懸空的頻道範圍設定：`channels.<id>` 項目、以該頻道為名的心跳偵測目標，以及 `agents.*.models["<channel>/*"]` 覆寫。這可避免頻道執行階段已不存在，但設定仍要求閘道繫結該頻道而導致的閘道啟動迴圈。
  </Accordion>
  <Accordion title="12. 閘道驗證檢查（本機權杖）">
    Doctor 會檢查本機閘道權杖驗證是否就緒。

    - 如果權杖模式需要權杖，但不存在任何權杖來源，doctor 會提供產生權杖的選項。
    - 如果 `gateway.auth.token` 由 SecretRef 管理但無法使用，doctor 會發出警告，且不會以純文字覆寫。
    - `openclaw doctor --generate-gateway-token` 僅會在未設定權杖 SecretRef 時強制產生權杖。

  </Accordion>
  <Accordion title="12b. 可感知 SecretRef 的唯讀修復">
    某些修復流程需要檢查已設定的認證資訊，同時不削弱執行階段快速失敗的行為。

    - `openclaw doctor --fix` 使用與狀態系列命令相同的唯讀 SecretRef 摘要模型，以進行特定設定修復。
    - 範例：Telegram `allowFrom` / `groupAllowFrom` `@username` 修復會嘗試使用可用的已設定機器人認證資訊。
    - 如果 Telegram 機器人權杖是透過 SecretRef 設定，但在目前的命令路徑中無法使用，doctor 會回報該認證資訊已設定但無法使用，並略過自動解析，而不是當機或誤報權杖遺失。

  </Accordion>
  <Accordion title="13. 閘道健康狀態檢查與重新啟動">
    Doctor 會執行健康狀態檢查，並在閘道看似不健康時提供重新啟動選項。
  </Accordion>
  <Accordion title="13b. 記憶搜尋就緒狀態">
    Doctor 會檢查已設定的記憶搜尋嵌入提供者是否已為預設代理程式就緒。其行為取決於已設定的後端與提供者：

    - **QMD 後端**：探測 `qmd` 二進位檔是否可用且可啟動。若不可用，會列印修正指引，包括 `npm install -g @tobilu/qmd`（或對應的 Bun 命令），以及手動指定二進位檔路徑的選項。
    - **明確指定的本機提供者**：檢查是否有本機模型檔案，或可辨識的遠端／可下載模型 URL。若遺失，會建議切換至遠端提供者。
    - **明確指定的遠端提供者**（`openai`、`voyage` 等）：確認環境或驗證儲存區中是否存在 API 金鑰。若遺失，會列印可採取行動的修正提示。
    - **舊版自動提供者**：將 `memorySearch.provider: "auto"` 視為 OpenAI、檢查 OpenAI 是否就緒，且 `doctor --fix` 會將其重寫為 `provider: "openai"`。

    當有快取的閘道探測結果可用時（檢查當下閘道處於健康狀態），doctor 會將其結果與命令列介面可見的設定交叉比對，並指出任何差異。Doctor 不會在預設路徑上啟動新的嵌入連線測試；若要即時檢查提供者，請使用深度記憶狀態命令。

    使用 `openclaw memory status --deep` 可在執行階段驗證嵌入功能是否就緒。

  </Accordion>
  <Accordion title="14. 頻道狀態警告">
    如果閘道狀態健康，doctor 會執行頻道狀態探測，並回報警告及建議的修正方式。
  </Accordion>
  <Accordion title="15. 監督程式設定稽核與修復">
    Doctor 會檢查已安裝的監督程式設定（launchd/systemd/schtasks）是否缺少預設值或使用過時的預設值（例如 systemd 的 network-online 相依性與重新啟動延遲）。發現不相符時，會建議更新，並可將服務檔案／工作重寫為目前的預設值。

    注意事項：

    - `openclaw doctor` 會在重寫監督程式設定前提示確認。
    - `openclaw doctor --yes` 會接受預設的修復提示。
    - `openclaw doctor --fix` 會在不提示的情況下套用建議的修正（`--repair` 是別名）。
    - `openclaw doctor --fix --force` 會覆寫自訂監督程式設定。
    - `OPENCLAW_SERVICE_REPAIR_POLICY=external` 會讓 doctor 對閘道服務生命週期維持唯讀。它仍會回報服務健康狀態並執行非服務修復，但會略過服務安裝／啟動／重新啟動／啟動程序、監督程式設定重寫，以及舊版服務清理，因為該生命週期由外部監督程式管理。
    - 在 Linux 上，當相符的 systemd 閘道單元處於作用中時，doctor 不會重寫命令／進入點中繼資料。它也會在重複服務掃描期間忽略非作用中、非舊版的額外類閘道單元，避免附屬服務檔案產生清理雜訊。
    - 如果權杖驗證需要權杖，且 `gateway.auth.token` 由 SecretRef 管理，doctor 的服務安裝／修復會驗證 SecretRef，但不會將解析出的純文字權杖值保存至監督程式服務環境中繼資料。
    - Doctor 會偵測舊版 LaunchAgent、systemd 或 Windows 排定工作安裝中以內嵌方式儲存的受管理 `.env`/SecretRef 支援服務環境值，並重寫服務中繼資料，讓這些值改由執行階段來源載入，而非從監督程式定義載入。
    - Doctor 會偵測服務命令是否在 `gateway.port` 變更後仍固定使用舊的 `--port`，並將服務中繼資料重寫為目前的連接埠。
    - 如果權杖驗證需要權杖，但已設定的權杖 SecretRef 無法解析，doctor 會封鎖安裝／修復路徑，並提供可採取行動的指引。
    - 如果同時設定了 `gateway.auth.token` 與 `gateway.auth.password`，但未設定 `gateway.auth.mode`，doctor 會封鎖安裝／修復，直到明確設定模式為止。
    - 對於 Linux 使用者 systemd 單元，doctor 在比較服務驗證中繼資料時，權杖漂移檢查會同時納入 `Environment=` 與 `EnvironmentFile=` 來源。
    - 如果設定最後是由較新的版本寫入，Doctor 服務修復會拒絕使用較舊的 OpenClaw 二進位檔重寫、停止或重新啟動閘道服務。請參閱[閘道疑難排解](/zh-TW/gateway/troubleshooting#split-brain-installs-and-newer-config-guard)。
    - 你隨時可以透過 `openclaw gateway install --force` 強制完整重寫。

  </Accordion>
  <Accordion title="16. 閘道執行階段與連接埠診斷">
    Doctor 會檢查服務執行階段（PID、上次結束狀態），並在服務已安裝但實際上未執行時發出警告。它也會檢查閘道連接埠（預設為 `18789`）是否發生連接埠衝突，並回報可能的原因（閘道已在執行、SSH 通道）。
  </Accordion>
  <Accordion title="17. 閘道執行階段最佳實務">
    當閘道服務在 Bun 或由版本管理工具管理的 Node 路徑（`nvm`、`fnm`、`volta`、`asdf` 等）上執行時，doctor 會發出警告。Bun 無法開啟 OpenClaw 的 `node:sqlite` 狀態儲存區，因此修復會將舊版 Bun 服務遷移至 Node。版本管理工具路徑可能在升級後失效，因為服務不會載入你的 shell 初始化設定。當有可用的系統 Node 安裝時（Homebrew/apt/choco），doctor 會提供遷移選項。

    新安裝或修復的 macOS LaunchAgent 會使用標準系統 PATH（`/opt/homebrew/bin:/opt/homebrew/sbin:/usr/local/bin:/usr/bin:/bin:/usr/sbin:/sbin`），而不是複製互動式 shell 的 PATH，讓 Homebrew 管理的系統二進位檔持續可用，同時避免 Volta、asdf、fnm、pnpm 及其他版本管理工具目錄改變 Node 子程序解析到的版本。Linux 服務仍會保留明確的環境根目錄（`NVM_DIR`、`FNM_DIR`、`VOLTA_HOME`、`ASDF_DATA_DIR`、`BUN_INSTALL`、`PNPM_HOME`）及穩定的使用者二進位檔目錄，但推測的版本管理工具備援目錄只有在磁碟上確實存在時，才會寫入服務 PATH。

  </Accordion>
  <Accordion title="18. 寫入設定與精靈中繼資料">
    Doctor 會保存所有設定變更，並寫入精靈中繼資料以記錄此次 doctor 執行。
  </Accordion>
  <Accordion title="19. 工作區提示（備份與記憶系統）">
    如果缺少工作區記憶系統，doctor 會提出建議；如果工作區尚未納入 git 管理，則會列印備份提示。

    如需工作區結構與 git 備份的完整指南（建議使用私人 GitHub 或 GitLab），請參閱 [/concepts/agent-workspace](/zh-TW/concepts/agent-workspace)。

  </Accordion>
</AccordionGroup>

## 相關內容

- [閘道操作手冊](/zh-TW/gateway)
- [閘道疑難排解](/zh-TW/gateway/troubleshooting)
