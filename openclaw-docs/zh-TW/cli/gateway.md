---
read_when:
    - 從命令列介面執行閘道（開發環境或伺服器）
    - 偵錯閘道驗證、繫結模式與連線能力
    - 透過 Bonjour 探索閘道（區域網路 + 廣域 DNS-SD）
    - 整合外部閘道程序監督器
sidebarTitle: Gateway
summary: OpenClaw 閘道命令列介面（`openclaw gateway`）— 執行、查詢及探索閘道
title: 閘道
x-i18n:
    generated_at: "2026-07-26T07:46:21Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 0188d7c79571ebf8f350295775625533a83cb2eb909bcc8763e8ce81806d2214
    source_path: cli/gateway.md
    workflow: 16
---

閘道是 OpenClaw 的 WebSocket 伺服器（頻道、節點、工作階段、鉤子）。以下所有子命令都位於 `openclaw gateway ...` 之下。

<CardGroup cols={3}>
  <Card title="Bonjour 探索" href="/zh-TW/gateway/bonjour">
    本機 mDNS + 廣域 DNS-SD 設定。
  </Card>
  <Card title="探索概覽" href="/zh-TW/gateway/discovery">
    OpenClaw 如何公告及尋找閘道。
  </Card>
  <Card title="設定" href="/zh-TW/gateway/configuration">
    頂層閘道設定鍵。
  </Card>
</CardGroup>

## 執行閘道

```bash
openclaw gateway
openclaw gateway run   # 等效的明確形式
```

<AccordionGroup>
  <Accordion title="啟動行為">
    - 除非已在 `~/.openclaw/openclaw.json` 中設定 `gateway.mode=local`，否則會拒絕啟動。臨時／開發執行請使用 `--allow-unconfigured`；它會略過防護機制，而不寫入或修復設定。
    - 啟動時若發現可修復的無效設定，互動式終端會提議執行 `openclaw doctor --fix`，並在取得同意後重試啟動一次。非互動式執行絕不會自動修復，而是印出該命令。若修復後的設定仍然無效，啟動仍會停止。
    - `openclaw onboard --mode local` 和 `openclaw setup` 會寫入 `gateway.mode=local`。若設定檔存在，但缺少 `gateway.mode`，系統會將其視為已損壞／遭覆寫的設定，且閘道不會替你猜測 `local`——請重新執行初始設定、手動設定該鍵，或傳入 `--allow-unconfigured`。
    - 禁止在未經驗證的情況下繫結至回送介面以外的位址。
    - `--bind` 的值 `lan`、`tailnet` 和 `custom` 目前僅透過 IPv4 路徑解析；僅限 IPv6 的自備主機設定需要在閘道前方放置 IPv4 輔助服務或 Proxy。
    - 獲得授權時，`SIGUSR1` 會觸發程序內重新啟動。`commands.restart`（預設：啟用）會控管外部傳送的 `SIGUSR1`；將其設為 `false` 可封鎖手動作業系統訊號重新啟動。面向代理程式的 `gateway` 工具為唯讀；代理程式透過經人員核准的 `openclaw` 委派工具要求重新啟動。
    - `SIGINT`/`SIGTERM` 會停止程序，但不會還原自訂終端狀態——若你將命令列介面包裝在終端介面或原始模式輸入中，請在結束前自行還原終端。

  </Accordion>
</AccordionGroup>

### 選項

<ParamField path="--port <port>" type="number">
  WebSocket 連接埠（預設取自設定／環境；通常為 `18789`）。
</ParamField>
<ParamField path="--bind <mode>" type="string">
  繫結模式：`loopback`（預設）、`lan`、`tailnet`、`auto`、`custom`。
</ParamField>
<ParamField path="--token <token>" type="string">
  `connect.params.auth.token` 的共用權杖。設定 `OPENCLAW_GATEWAY_TOKEN` 時預設使用該值。
</ParamField>
<ParamField path="--auth <mode>" type="string">
  驗證模式：`none`、`token`、`password`、`trusted-proxy`。
</ParamField>
<ParamField path="--password <password>" type="string">
  `--auth password` 的密碼。
</ParamField>
<ParamField path="--password-file <path>" type="string">
  從檔案讀取閘道密碼。
</ParamField>
<ParamField path="--tailscale <mode>" type="string">
  Tailscale 公開模式：`off`、`serve`、`funnel`。
</ParamField>
<ParamField path="--tailscale-reset-on-exit" type="boolean">
  關閉時重設 Tailscale serve/funnel 設定。
</ParamField>
<ParamField path="--allow-unconfigured" type="boolean">
  啟動時不強制要求 `gateway.mode=local`。僅供臨時／開發環境啟動；不會保存或修復設定。
</ParamField>
<ParamField path="--dev" type="boolean">
  若缺少開發設定與工作區，則建立它們（略過 `BOOTSTRAP.md`）。
</ParamField>
<ParamField path="--dev-ambient-channels" type="boolean">
  允許開發用閘道從目前環境變數自動設定頻道。需要 `--dev`。
</ParamField>
<ParamField path="--reset" type="boolean">
  重設開發設定、認證資訊、工作階段和工作區。需要 `--dev`。
</ParamField>
<ParamField path="--force" type="boolean">
  啟動前終止目標連接埠上任何現有的接聽程式。在非互動式 Shell 中，此選項會拒絕終止已驗證的閘道接聽程式；請改用 `--dev`，或使用具有可用連接埠的隔離 `--profile`。
</ParamField>
<ParamField path="--verbose" type="boolean">
  將詳細記錄輸出至 stdout/stderr。
</ParamField>
<ParamField path="--cli-backend-logs" type="boolean">
  主控台中只顯示命令列介面後端記錄（也會啟用 stdout/stderr）。
</ParamField>
<ParamField path="--ws-log <style>" type="string" default="auto">
  WebSocket 記錄樣式：`auto`、`full`、`compact`。
</ParamField>
<ParamField path="--compact" type="boolean">
  `--ws-log compact` 的別名。
</ParamField>
<ParamField path="--raw-stream" type="boolean">
  將原始模型串流事件記錄至 JSONL。
</ParamField>
<ParamField path="--raw-stream-path <path>" type="string">
  原始串流 JSONL 路徑。
</ParamField>

`--claude-cli-logs` 是 `--cli-backend-logs` 已棄用的別名。

使用 `--bind custom` 時，請將 `gateway.customBindHost` 設為 IPv4 位址。除 `127.0.0.1` 或 `0.0.0.0` 以外的任何位址，也要求相同主機的用戶端在同一連接埠上使用 `127.0.0.1`；若任一接聽程式無法繫結，啟動就會失敗。萬用字元 `0.0.0.0` 不會新增另一個必要別名。僅限 IPv6 的自備主機設定需要在閘道前方放置 IPv4 輔助服務或 Proxy。

## 重新啟動閘道

```bash
openclaw gateway restart
openclaw gateway restart --safe
openclaw gateway restart --safe --skip-deferral
openclaw gateway restart --force
openclaw gateway restart --wait 30s
```

`--safe` 會要求執行中的閘道預先檢查進行中的工作，並排定在這些工作結束後進行一次合併的重新啟動。等待時間上限為 5 分鐘；超過時間額度時會強制重新啟動。`--safe` 無法與 `--force` 或 `--wait` 合併使用。

`--skip-deferral` 會略過安全重新啟動的進行中工作延後閘門，因此即使回報有阻礙因素，閘道仍會立即重新啟動。它需要 `--safe`——當延後流程卡在失控的工作上時使用。

`--wait <duration>` 會覆寫一般（非安全）重新啟動的清空時間額度。接受不含單位的毫秒值，或單位後綴 `ms`、`s`、`m`、`h`、`d`（例如 `30s`、`5m`、`1h30m`）；`--wait 0` 會無限期等待。無法與 `--force` 或 `--safe` 相容。

`--force` 會略過進行中工作的清空程序並立即重新啟動。一般的 `restart`（不含旗標）會保留現有的服務管理員重新啟動行為。

<Warning>
內嵌的 `--password` 可能會顯示在本機程序清單中。建議使用 `--password-file`、環境變數或由 SecretRef 支援的 `gateway.auth.password`。
</Warning>

### 外部監督程式

只有在另一個程序管理員負責閘道生命週期時，才設定 `OPENCLAW_SUPERVISOR_MODE=external`。在此模式下：

- `openclaw gateway restart` 會保留現有的安全、強制及限定等待行為，但目標改為經驗證、正在執行的閘道，而非 launchd、systemd 或 Task Scheduler。
- 系統會拒絕原生服務的安裝、啟動、停止和解除安裝作業，並提示使用外部監督程式。
- 系統會拒絕 OpenClaw 自我更新，讓監督程式可以停止閘道、取代並完成執行階段，然後安全地重新啟動。
- 全新程序重新啟動會在正常結束前寫入具有界限的 SQLite 交接資料。若保存失敗，閘道會改為程序內重新啟動，而非在沒有可用交接資料的情況下結束。

`OPENCLAW_SERVICE_REPAIR_POLICY=external` 仍是獨立的 Doctor 修復原則。它不宣告執行階段的擁有權；同時需要兩種行為的監督程式應設定這兩個變數。

外部監督程式可透過以下隱藏的機器合約協商並取用重新啟動交接資料：

```bash
openclaw gateway restart-handoff capabilities --json
openclaw gateway restart-handoff consume --expected-pid <pid> --json
```

通訊協定版本 `1` 支援 `consume` 作業。取用程序會在單一即時 SQLite 交易中驗證預期 PID 與具有界限的交接欄位。接受交接後會先刪除資料再回傳成功，因此並行或重播的取用者無法同時接受該交接。PID 不相符的資料會保留給相符的擁有者；缺少、已過期或無效的資料列不會授權重新啟動。

有效的機器要求會傳回 JSON，結束碼為 `0`，包括不重新啟動的結果。無效引數會傳回 `reason: "invalid-expected-pid"`，結束碼為 `2`；狀態儲存區失敗會傳回 `reason: "store-unavailable"`，結束碼為 `1`。監督程式應在實際要使用的執行階段或啟動器上探測 `capabilities`，而非根據 OpenClaw 版本字串推斷支援情況，或直接讀取私有 SQLite 結構描述。

### 閘道效能分析

- `OPENCLAW_GATEWAY_STARTUP_TRACE=1` 會記錄啟動期間的階段計時，包括各階段的 `eventLoopMax` 延遲與外掛查閱表計時（已安裝索引、資訊清單登錄、啟動規劃、擁有者對應工作）。
- `OPENCLAW_GATEWAY_RESTART_TRACE=1` 會記錄重新啟動範圍內的 `restart trace:` 行：訊號處理、進行中工作清空、關閉階段、下次啟動、就緒計時和記憶體指標。
- `OPENCLAW_DIAGNOSTICS=timeline` 搭配 `OPENCLAW_DIAGNOSTICS_TIMELINE_PATH=<path>`，會為外部 QA 測試框架盡力寫入 JSONL 啟動診斷時間軸（等同於設定 `diagnostics.flags: ["timeline"]`；路徑仍只能透過環境變數設定）。加入 `OPENCLAW_DIAGNOSTICS_EVENT_LOOP=1` 可包含事件迴圈樣本。
- 先執行 `pnpm build`，再執行 `pnpm test:startup:gateway -- --runs 5 --warmup 1`，會根據已建置的命令列介面進入點測量閘道啟動效能：首次程序輸出、`/healthz`、`/readyz`、啟動追蹤計時、事件迴圈延遲及外掛查閱表計時。
- 先執行 `pnpm build`，再執行 `pnpm test:restart:gateway -- --case skipChannels --runs 1 --restarts 5`，會在 macOS 或 Linux 上測量程序內重新啟動效能（Windows 不支援；重新啟動需要 `SIGUSR1`）。它使用 `SIGUSR1`、在子程序中啟用兩項追蹤，並記錄下一次 `/healthz`、下一次 `/readyz`、停機時間、就緒計時、CPU、RSS 和重新啟動追蹤指標。
- `/healthz` 表示存活；`/readyz` 表示可用就緒。應將追蹤行和效能測試輸出視為擁有者歸因訊號，而非根據單一時間範圍或樣本得出的完整效能結論。

## 查詢執行中的閘道

所有查詢命令都使用 WebSocket RPC。

<Tabs>
  <Tab title="輸出模式">
    - 預設：人類可讀（在 TTY 中有色彩）。
    - `--json`：機器可讀的 JSON（無樣式／進度動畫）。
    - `--no-color`（或 `NO_COLOR=1`）：停用 ANSI，同時保留人類可讀的版面配置。

  </Tab>
  <Tab title="共用選項">
    - `--url <url>`：閘道 WebSocket URL。
    - `--token <token>`：閘道權杖。
    - `--password <password>`：閘道密碼。
    - `--timeout <ms>`：逾時／時間額度（預設值依命令而異；請參閱下方各命令）。
    - `--expect-final`：等待「最終」回應（代理程式呼叫）。

  </Tab>
</Tabs>

<Note>
設定 `--url` 時，命令列介面不會退回使用設定或環境中的認證資訊。請明確傳入 `--token` 或 `--password`。缺少明確認證資訊會視為錯誤。
</Note>

### `gateway health`

```bash
openclaw gateway health --url ws://127.0.0.1:18789
openclaw gateway health --port 18789
```

`/healthz` 是存活探測：只要伺服器能回應 HTTP，就會立即傳回。`/readyz` 的判定更嚴格；當啟動中的外掛 sidecar、頻道或已設定的掛鉤仍在穩定期間時，會持續顯示紅色。本機或經過驗證的詳細 `/readyz` 回應包含一個 `eventLoop` 診斷區塊（延遲、使用率、CPU 核心比率、`degraded` 旗標）。

<ParamField path="--port <port>" type="number">
  以此連接埠上的本機迴路閘道為目標。此呼叫會覆寫 `OPENCLAW_GATEWAY_URL` 和 `OPENCLAW_GATEWAY_PORT`。
</ParamField>

### `gateway usage-cost`

從工作階段日誌擷取用量成本摘要。

```bash
openclaw gateway usage-cost
openclaw gateway usage-cost --days 7
openclaw gateway usage-cost --agent work --json
openclaw gateway usage-cost --all-agents
openclaw gateway usage-cost --json
```

<ParamField path="--days <days>" type="number" default="30">
  要納入的天數。
</ParamField>
<ParamField path="--agent <id>" type="string">
  將摘要範圍限於一個已設定的代理程式 ID。
</ParamField>
<ParamField path="--all-agents" type="boolean">
  彙總所有已設定的代理程式。無法與 `--agent` 搭配使用。
</ParamField>

### `gateway stability`

從執行中的閘道擷取近期診斷穩定性記錄器。

```bash
openclaw gateway stability
openclaw gateway stability --type payload.large
openclaw gateway stability --bundle latest
openclaw gateway stability --bundle latest --export
openclaw gateway stability --json
```

<ParamField path="--limit <limit>" type="number" default="25">
  要納入的近期事件數量上限（最大值為 `1000`）。
</ParamField>
<ParamField path="--type <type>" type="string">
  依診斷事件類型篩選，例如 `payload.large` 或 `diagnostic.memory.pressure`。
</ParamField>
<ParamField path="--since-seq <seq>" type="number">
  僅納入診斷序號之後的事件。
</ParamField>
<ParamField path="--bundle [path]" type="string">
  讀取持久保存的穩定性套件，而非呼叫執行中的閘道。`--bundle latest`（或不帶值的 `--bundle`）會選取狀態目錄下最新的套件；你也可以直接傳入套件 JSON 路徑。
</ParamField>
<ParamField path="--export" type="boolean">
  寫入可分享的支援診斷 ZIP 檔，而非列印穩定性詳細資料。
</ParamField>
<ParamField path="--output <path>" type="string">
  `--export` 的輸出路徑。
</ParamField>

<AccordionGroup>
  <Accordion title="隱私權與套件行為">
    - 記錄會保留操作中繼資料：事件名稱、計數、位元組大小、記憶體讀數、佇列／工作階段狀態、核准 ID、頻道／外掛名稱，以及已遮蔽的工作階段摘要。其中不包含聊天文字、網路鉤子本文、工具輸出、原始要求／回應本文、權杖、Cookie、密鑰值、主機名稱和原始工作階段 ID。設定 `diagnostics.enabled: false` 可完全停用記錄器。
    - 若記錄器中有事件，閘道嚴重錯誤退出、關閉逾時和重新啟動時的啟動失敗，會將相同的診斷快照寫入 `~/.openclaw/logs/stability/openclaw-stability-*.json`。使用 `openclaw gateway stability --bundle latest` 檢查最新套件；`--limit`、`--type` 和 `--since-seq` 也適用於套件輸出。

  </Accordion>
</AccordionGroup>

### `gateway diagnostics export`

寫入專為錯誤報告設計的本機診斷 ZIP 檔。如需瞭解隱私權模型和套件內容，請參閱[診斷匯出](/zh-TW/gateway/diagnostics)。

```bash
openclaw gateway diagnostics export
openclaw gateway diagnostics export --output openclaw-diagnostics.zip
openclaw gateway diagnostics export --json
```

<ParamField path="--output <path>" type="string">
  輸出 ZIP 路徑。預設為狀態目錄下的支援匯出檔案。
</ParamField>
<ParamField path="--log-lines <count>" type="number" default="5000">
  要納入的已清理日誌行數上限。
</ParamField>
<ParamField path="--log-bytes <bytes>" type="number" default="1000000">
  要檢查的日誌位元組數上限。
</ParamField>
<ParamField path="--url <url>" type="string">
  用於健康狀態快照的閘道 WebSocket URL。
</ParamField>
<ParamField path="--token <token>" type="string">
  用於健康狀態快照的閘道權杖。
</ParamField>
<ParamField path="--password <password>" type="string">
  用於健康狀態快照的閘道密碼。
</ParamField>
<ParamField path="--timeout <ms>" type="number" default="3000">
  狀態／健康狀態快照的逾時時間。
</ParamField>
<ParamField path="--no-stability-bundle" type="boolean">
  略過持久保存的穩定性套件查詢。
</ParamField>
<ParamField path="--json" type="boolean">
  以 JSON 格式列印寫入的路徑、大小和資訊清單。
</ParamField>

匯出套件包含：`manifest.json`（檔案清單）、`summary.md`（Markdown 摘要）、`diagnostics.json`（頂層設定／日誌／探索／穩定性／狀態／健康狀態摘要）、`config/sanitized.json`、`status/gateway-status.json`、`health/gateway-health.json`、`logs/openclaw-sanitized.jsonl`，以及套件存在時的 `stability/latest.json`。

此匯出檔專為分享而設計。它會保留對偵錯有用的操作詳細資料，包括安全的日誌欄位、子系統名稱、狀態碼、持續時間、已設定的模式、連接埠、外掛／供應商 ID、非密鑰功能設定，以及已遮蔽的操作日誌訊息；同時省略或遮蔽聊天文字、網路鉤子本文、工具輸出、認證資訊、Cookie、帳號／訊息識別碼、提示／指示文字、主機名稱和密鑰值。當日誌訊息看似使用者／聊天／工具承載資料文字（例如 “user said”、“chat text”、“tool output”、“webhook body”）時，匯出檔只會保留訊息已省略的事實及其位元組數。

### `gateway status`

顯示閘道服務（launchd/systemd/schtasks），並可選擇執行連線能力／驗證探測。

```bash
openclaw gateway status
openclaw gateway status --json
openclaw gateway status --require-rpc
```

<ParamField path="--url <url>" type="string">
  新增明確的探測目標。仍會探測已設定的遠端目標和 localhost。
</ParamField>
<ParamField path="--token <token>" type="string">
  探測所使用的權杖驗證。
</ParamField>
<ParamField path="--password <password>" type="string">
  探測所使用的密碼驗證。
</ParamField>
<ParamField path="--timeout <ms>" type="number" default="10000">
  探測逾時時間。
</ParamField>
<ParamField path="--no-probe" type="boolean">
  略過連線能力探測（僅顯示服務）。
</ParamField>
<ParamField path="--deep" type="boolean">
  同時掃描系統層級的服務。
</ParamField>
<ParamField path="--require-rpc" type="boolean">
  將連線能力探測升級為讀取探測，並在失敗時以非零狀態退出。無法與 `--no-probe` 搭配使用。
</ParamField>

<AccordionGroup>
  <Accordion title="狀態語意">
    - 即使本機命令列介面設定遺失或無效，仍可用於診斷。
    - 預設輸出會證明服務狀態、WebSocket 連線，以及交握時可見的驗證能力，而非讀取／寫入／管理員操作。
    - 首次裝置驗證的探測不會變更狀態：若已有快取的裝置權杖，就會重複使用，但絕不會只為了檢查狀態而建立新的命令列介面裝置身分或唯讀配對記錄。
    - 在可能的情況下，解析已設定的驗證 SecretRef 以供探測驗證使用。如果必要的 SecretRef 尚未解析，當探測連線能力／驗證失敗時，`--json` 會回報 `rpc.authWarning`；請明確傳入 `--token`/`--password`，或修正密鑰來源。探測成功後，未解析驗證警告就會受到抑制。
    - 當執行中的閘道回報 `gateway.version` 時，JSON 輸出會包含該資訊；如果交握探測無法提供版本中繼資料，`--require-rpc` 可以改用 `status.runtimeVersion` RPC 承載資料。
    - 當服務僅處於監聽狀態仍不足以滿足需求，且你也需要讀取範圍 RPC 保持健康時，請在指令碼／自動化中使用 `--require-rpc`。
    - `--deep` 會掃描額外的 launchd/systemd/schtasks 安裝項目；若找到多個類似閘道的服務，人類可讀輸出會列印清理提示（通常每台機器執行一個閘道），並在適用時回報近期的監督程式重新啟動交接。
    - `--deep` 也會以外掛感知模式（`pluginValidation: "full"`）執行設定驗證，並顯示外掛資訊清單警告（例如缺少頻道設定中繼資料）。預設的 `gateway status` 會保留略過外掛驗證的快速唯讀路徑。
    - 人類可讀輸出包含解析後的檔案日誌路徑，以及命令列介面與服務的設定路徑／有效性，有助於診斷設定檔或狀態目錄的偏移。
    - 人類可讀輸出包含 `Gateway heap:`，其中有已套用的限制及其調適性推導。JSON 輸出會將相同報告公開為 `service.gatewayHeap`。

  </Accordion>
  <Accordion title="Linux systemd 驗證偏移檢查">
    - 服務驗證偏移檢查會從單元讀取 `Environment=` 和 `EnvironmentFile=`（包括 `%h`、加上引號的路徑、多個檔案，以及選用的 `-` 檔案）。
    - 使用合併後的執行階段環境解析 `gateway.auth.token` SecretRef（優先使用服務命令環境，其次回退到程序環境）。
    - 當權杖驗證實際上未啟用時，權杖偏移檢查會略過設定權杖解析（`gateway.auth.mode` 明確設為 `password`/`none`/`trusted-proxy`；或模式未設定、密碼可能勝出，且沒有權杖候選項能勝出）。

  </Accordion>
</AccordionGroup>

### `gateway probe`

「偵錯所有項目」命令。它一律會探測：

- 你已設定的遠端閘道（若有設定），以及
- localhost（迴路），**即使已設定遠端目標**。

傳入 `--url` 會在這兩者之前新增該明確目標。人類可讀輸出會將目標標示為 `URL (explicit)`、`Remote (configured)` / `Remote (configured, inactive)` 和 `Local loopback`。

<Note>
如果有多個探測目標可連線，會全部列印。SSH 通道、TLS／Proxy URL 和已設定的遠端 URL 即使使用不同的傳輸連接埠，也可能指向同一個閘道；`multiple_gateways` 保留給不同或身分不明確的可連線閘道。系統支援為隔離的設定檔執行多個閘道（例如救援機器人），但大多數安裝只會執行單一閘道。
</Note>

```bash
openclaw gateway probe
openclaw gateway probe --json
openclaw gateway probe --port 18789
```

<ParamField path="--port <port>" type="number">
  將此連接埠用於本機迴路探測目標和 SSH 通道遠端連接埠。若沒有 `--url`，這只會選取本機迴路目標，而非已設定的閘道環境 URL、環境連接埠或遠端目標。
</ParamField>

<AccordionGroup>
  <Accordion title="解讀方式">
    - `Reachable: yes` 表示至少有一個目標接受 WebSocket 連線。
    - `Capability: read-only|write-capable|admin-capable|pairing-pending|connect-only` 會回報探測能確認的驗證資訊，並與可連線性分開顯示。
    - `Read probe: ok` 表示讀取範圍的詳細 RPC 呼叫（`health`/`status`/`system-presence`/`config.get`）也成功。
    - `Read probe: limited - missing scope: operator.read` 表示連線成功，但讀取範圍 RPC 受限。這會回報為可連線性**降級**，而非完全失敗。
    - `Connect: ok` 之後的 `Read probe: failed` 表示 WebSocket 已連線，但後續讀取診斷逾時或失敗；這同樣是**降級**，而非無法連線。
    - 與 `gateway status` 相同，探測會重複使用現有的快取裝置驗證，但不會建立首次使用的裝置身分或配對狀態。
    - 只有在所有探測目標都無法連線時，退出代碼才會是非零值。

  </Accordion>
  <Accordion title="JSON 輸出">
    頂層：

    - `ok`：至少有一個目標可連線。
    - `degraded`：至少有一個目標接受了連線，但未完成完整的詳細 RPC 診斷。
    - `capability`：在可連線目標中觀察到的最佳能力（`read_only`、`write_capable`、`admin_capable`、`pairing_pending`、`connected_no_operator_scope` 或 `unknown`）。
    - `primaryTargetId`：最適合視為目前有效勝出者的目標，依序為：明確 URL、SSH 通道、已設定的遠端目標、本機迴路。
    - `warnings[]`：盡力提供的警告記錄，包含 `code`、`message`，以及選用的 `targetIds`。
    - `network`：根據目前設定與主機網路推導出的本機迴路／tailnet URL 提示。
    - `discovery.timeoutMs` / `discovery.count`：此輪探測實際使用的探索預算／結果數量。

    每個目標（`targets[].connect`）：`ok`（可連線性 + 降級分類）、`rpcOk`（完整詳細 RPC 成功）、`scopeLimited`（因缺少操作者範圍而導致詳細 RPC 失敗）。

    每個目標（`targets[].auth`）：可用時會在 `hello-ok` 中回報 `role` 與 `scopes`，以及呈現的 `capability` 分類。

  </Accordion>
  <Accordion title="常見警告代碼">
    - `ssh_tunnel_failed`：SSH 通道設定失敗；命令已改用直接探測。
    - `multiple_gateways`：可連線到不同的閘道身分，或 OpenClaw 無法證明可連線的目標是同一個閘道。指向同一閘道的 SSH 通道、代理 URL 或已設定的遠端 URL 不會觸發此警告。
    - `auth_secretref_unresolved`：無法解析失敗目標所設定的驗證 SecretRef。
    - `probe_scope_limited`：WebSocket 連線成功，但因缺少 `operator.read`，讀取探測受到限制。
    - `local_tls_runtime_unavailable`：已啟用本機閘道 TLS，但 OpenClaw 無法載入本機憑證指紋。

  </Accordion>
</AccordionGroup>

#### 透過 SSH 遠端連線（與 Mac 應用程式一致）

macOS 應用程式的「Remote over SSH」模式使用本機連接埠轉送，讓僅限迴路連線的遠端閘道可在 `ws://127.0.0.1:<port>` 連線。

對應的命令列介面命令：

```bash
openclaw gateway probe --ssh user@gateway-host
```

<ParamField path="--ssh <target>" type="string">
  `user@host` 或 `user@host:port`（連接埠預設為 `22`）。
</ParamField>
<ParamField path="--ssh-identity <path>" type="string">
  身分檔案。
</ParamField>
<ParamField path="--ssh-auto" type="boolean">
  從已解析的探索端點中，選擇第一個探索到的閘道主機作為 SSH 目標（`local.`，以及已設定的廣域網域（若有））。僅有 TXT 的提示會被忽略。
</ParamField>

設定預設值（選用）：`gateway.remote.sshTarget`、`gateway.remote.sshIdentity`。

### `gateway call <method>`

低階 RPC 輔助工具。

```bash
openclaw gateway call status
openclaw gateway call logs.tail --params '{"limit": 200}'
```

<ParamField path="--params <json>" type="string" default="{}">
  用於參數的 JSON 物件字串。
</ParamField>
<ParamField path="--url <url>" type="string">
  閘道 WebSocket URL。
</ParamField>
<ParamField path="--token <token>" type="string">
  閘道權杖。
</ParamField>
<ParamField path="--password <password>" type="string">
  閘道密碼。
</ParamField>
<ParamField path="--timeout <ms>" type="number" default="10000">
  逾時預算。
</ParamField>
<ParamField path="--expect-final" type="boolean">
  主要用於會在最終承載內容之前串流中間事件的代理程式型 RPC。
</ParamField>
<ParamField path="--json" type="boolean">
  機器可讀的 JSON 輸出。
</ParamField>

<Note>
`--params` 必須是有效的 JSON，且每個方法都會驗證自己的參數形狀（額外或名稱錯誤的欄位會遭拒絕）。
</Note>

## 管理閘道服務

```bash
openclaw gateway install
openclaw gateway start
openclaw gateway stop
openclaw gateway restart
openclaw gateway uninstall
```

### 使用包裝程式安裝

當受管理的服務必須透過另一個可執行檔啟動時，請使用 `--wrapper`，例如密鑰管理器轉接程式或指定執行身分的輔助工具。包裝程式會接收一般的閘道引數，並負責最終以這些引數執行 `openclaw` 或 Node。

```bash
cat > ~/.local/bin/openclaw-doppler <<'EOF'
#!/usr/bin/env bash
set -euo pipefail
exec doppler run --project my-project --config production -- openclaw "$@"
EOF
chmod +x ~/.local/bin/openclaw-doppler

openclaw gateway install --wrapper ~/.local/bin/openclaw-doppler --force
openclaw gateway restart
```

你也可以透過環境設定包裝程式。`gateway install` 會驗證該路徑是可執行檔、將包裝程式寫入服務的 `ProgramArguments`，並將 `OPENCLAW_WRAPPER` 保留在服務環境中，以供之後的強制重新安裝、更新及 doctor 修復使用。

```bash
OPENCLAW_WRAPPER="$HOME/.local/bin/openclaw-doppler" openclaw gateway install --force
openclaw doctor
```

若要移除已保留的包裝程式，請在重新安裝時清除 `OPENCLAW_WRAPPER`：

```bash
OPENCLAW_WRAPPER= openclaw gateway install --force
openclaw gateway restart
```

<AccordionGroup>
  <Accordion title="命令選項">
    - `gateway status`：`--url`、`--token`、`--password`、`--timeout`、`--no-probe`、`--require-rpc`、`--deep`、`--json`
    - `gateway install`：`--port`、`--runtime <node>`（預設值：`node`）、`--token`、`--wrapper <path>`、`--force`、`--json`
    - `gateway restart`：`--safe`、`--skip-deferral`、`--force`、`--wait <duration>`、`--json`
    - `gateway uninstall|start`：`--json`
    - `gateway stop`：`--disable`、`--force`、`--json`

  </Accordion>
  <Accordion title="生命週期行為">
    - `gateway start` 是冪等的：當受管理的服務已在執行時，它會回報執行中的程序並保持不變。已載入但停止的服務仍會照常啟動。
    - 請使用 `gateway restart` 重新啟動受管理的服務。請勿串接 `gateway stop` 與 `gateway start` 來代替重新啟動。
    - 在非互動式 shell 中，`gateway stop` 需要 `--force`。互動式終端機則維持現有的無提示行為。對於自動化與測試，建議使用 `gateway run --dev`，或使用具有可用連接埠且隔離的 `--profile`。
    - 在 macOS 上，`gateway stop` 預設使用 `launchctl bootout`，這會從目前的開機工作階段移除 LaunchAgent，但不會持續停用它——KeepAlive 的自動復原仍會在未來當機時保持啟用，而 `gateway start` 無須手動執行 `launchctl enable` 即可重新啟用。傳入 `--disable` 可持續抑制 KeepAlive 與 RunAtLoad，使閘道在下一次明確執行 `gateway start` 前不會重新衍生；當手動停止需要在重新開機後仍然有效時，請使用此選項。
    - 閘道生命週期變更會盡力將鍵值稽核記錄附加至 `<state-dir>/logs/gateway-restart.log`，其中包括命令列介面的啟動、停止及重新啟動操作、安全重新啟動要求、監督程式重新啟動，以及分離式交接。
    - 生命週期命令接受 `--json`，以供指令碼使用。

  </Accordion>
  <Accordion title="受管理閘道的堆積大小設定">
    - `gateway install` 會為受管理的閘道服務寫入僅限堆積的 `NODE_OPTIONS` 值。當 Node 回報容器或服務限制時，目標設為受限記憶體的 50%；否則設為實體記憶體的 50%。
    - 名義目標範圍為 2048–8192 MiB，並另設 75% 的原生記憶體餘裕上限。在小型主機上，此餘裕上限可能使套用的限制低於名義上的 2048 MiB 下限。
    - 已儲存在所安裝服務中的有效明確 `--max-old-space-size`，會在強制重新安裝與 doctor 修復時保留。其他 `NODE_OPTIONS` 旗標不會帶入受管理的服務。
    - 周遭 shell 的 `NODE_OPTIONS` 不會覆寫此原則。請使用 `gateway status` 或 `doctor` 檢查已安裝的值；執行 `openclaw gateway install --force` 可重新產生沒有受管理堆積設定的舊版服務中繼資料。
    - 此原則僅套用於受管理的閘道服務。前景執行的 `gateway run`、節點服務及手動編寫的監督程式單元會保留各自的執行階段設定。

  </Accordion>
  <Accordion title="安裝時的驗證與 SecretRef">
    - 當權杖驗證需要權杖，且 `gateway.auth.token` 由 SecretRef 管理時，`gateway install` 會驗證 SecretRef 能否解析，但不會將解析出的權杖保留至服務環境中繼資料。
    - 如果權杖驗證需要權杖，但設定的權杖 SecretRef 無法解析，安裝會採取封閉式失敗，而不會保留後備純文字。
    - 對於 `gateway run` 上的密碼驗證，建議優先使用 `OPENCLAW_GATEWAY_PASSWORD`、`--password-file` 或由 SecretRef 支援的 `gateway.auth.password`，而非內嵌的 `--password`。
    - 在推斷驗證模式下，僅限 shell 的 `OPENCLAW_GATEWAY_PASSWORD` 不會放寬安裝權杖的要求；安裝受管理的服務時，請使用持久設定（`gateway.auth.password` 或設定中的 `env`）。
    - 如果同時設定了 `gateway.auth.token` 與 `gateway.auth.password`，而 `gateway.auth.mode` 未設定，安裝將被封鎖，直到明確設定模式為止。

  </Accordion>
</AccordionGroup>

## 探索閘道（Bonjour）

`gateway discover` 會掃描閘道信標（`_openclaw-gw._tcp`）。

- 多點傳送 DNS-SD：`local.`
- 單點傳送 DNS-SD（廣域 Bonjour）：選擇一個網域（例如：`openclaw.internal.`），並設定分割 DNS 與 DNS 伺服器；請參閱 [Bonjour](/zh-TW/gateway/bonjour)。

只有啟用 Bonjour 探索功能（預設啟用）的閘道會廣播信標。

每個信標上的 TXT 提示：`role`（閘道角色提示）、`transport`（傳輸提示，例如 `gateway`）、`gatewayPort`（WebSocket 連接埠，通常為 `18789`）、`tailnetDns`（MagicDNS 主機名稱，如有）、`gatewayTls` / `gatewayTlsSha256`（是否啟用 TLS + 憑證指紋）。`sshPort` 與 `cliPath` 僅在完整探索模式下發布（`discovery.mdns.mode: "full"`；預設為 `"minimal"`，會省略這兩者——此時用戶端會將 SSH 目標連接埠預設為 `22`）。

### `gateway discover`

```bash
openclaw gateway discover
```

<ParamField path="--timeout <ms>" type="number" default="2000">
  每個命令的逾時時間（瀏覽／解析）。
</ParamField>
<ParamField path="--json" type="boolean">
  機器可讀的輸出（也會停用樣式與旋轉指示器）。
</ParamField>

範例：

```bash
openclaw gateway discover --timeout 4000
openclaw gateway discover --json | jq '.beacons[].wsUrl'
```

<Note>
- 掃描 `local.`，以及啟用時已設定的廣域網域。
- JSON 輸出中的 `wsUrl` 是從解析後的服務端點推導，而不是來自僅有 TXT 的提示，例如 `lanHost` 或 `tailnetDns`。
- `discovery.mdns.mode` 控制 `local.` mDNS 與廣域 DNS-SD 上的 `sshPort`/`cliPath` 發布（請參閱上文）。

</Note>

## 相關內容

- [命令列介面參考](/zh-TW/cli)
- [閘道操作手冊](/zh-TW/gateway)
