---
read_when:
    - 執行或偵錯閘道程序
summary: 閘道服務、生命週期與維運操作手冊
title: 閘道操作手冊
x-i18n:
    generated_at: "2026-07-26T08:24:02Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: d8b50b6041905c321887ea0f579f8d4c3b74552b2b72c37ec655e43a53dfc130
    source_path: gateway/index.md
    workflow: 16
---

此頁面適用於閘道服務的首日啟動與後續日常維運。

<CardGroup cols={2}>
  <Card title="深入疑難排解" icon="siren" href="/zh-TW/gateway/troubleshooting">
    以症狀為起點的診斷，提供確切的命令步驟與日誌特徵。
  </Card>
  <Card title="設定" icon="sliders" href="/zh-TW/gateway/configuration">
    以任務為導向的設定指南，以及完整的設定參考。
  </Card>
  <Card title="密鑰管理" icon="key-round" href="/zh-TW/gateway/secrets">
    SecretRef 合約、執行階段快照行為，以及遷移／重新載入操作。
  </Card>
  <Card title="密鑰計畫合約" icon="shield-check" href="/zh-TW/gateway/secrets-plan-contract">
    確切的 `secrets apply` 目標／路徑規則，以及僅使用參照的驗證設定檔行為。
  </Card>
</CardGroup>

## 5 分鐘本機啟動

<Steps>
  <Step title="啟動閘道">

```bash
openclaw gateway --port 18789
# 除錯／追蹤同步輸出至 stdio
openclaw gateway --port 18789 --verbose
# 強制終止所選連接埠上的監聽程序，然後啟動
openclaw gateway --force
```

  </Step>

  <Step title="驗證服務健康狀態">

```bash
openclaw gateway status
openclaw status
openclaw logs --follow
```

健康基準：`Runtime: running`、`Connectivity probe: ok`，以及符合預期的 `Capability` 行。請使用 `openclaw gateway status --require-rpc` 證明唯讀範圍 RPC 正常，而不只是可連線。

  </Step>

  <Step title="驗證頻道就緒狀態">

```bash
openclaw channels status --probe
```

閘道可連線時，此命令會即時執行各帳號的頻道探測與選用稽核。若無法連線至閘道，命令列介面會退回僅依設定產生頻道摘要。

  </Step>
</Steps>

<Note>
閘道設定重新載入功能會監看使用中的設定檔路徑（從設定檔／狀態預設值解析，或在有設定時使用 `OPENCLAW_CONFIG_PATH`）。預設模式為 `gateway.reload.mode="hybrid"`。首次成功載入後，執行中的程序會使用目前有效的記憶體內設定快照提供服務；成功重新載入時，會以不可分割方式替換該快照。
</Note>

## 執行階段模型

- 一個持續執行的程序，負責路由、控制平面與頻道連線。
- 單一多工連接埠用於：
  - WebSocket 控制／RPC
  - HTTP API（`/v1/models`、`/v1/embeddings`、`/v1/chat/completions`、`/v1/responses`、`/tools/invoke`）
  - 外掛 HTTP 路由，例如選用的 `/api/v1/admin/rpc`
  - 控制介面與鉤子
- 預設繫結模式：`loopback`。在偵測到的容器環境內，有效預設值為 `auto`（解析為 `0.0.0.0`，以便進行連接埠轉送）；但若啟用了 Tailscale serve／funnel，則一律強制使用 `loopback`。
- 預設需要驗證。共用密鑰設定使用 `gateway.auth.token`／`gateway.auth.password`（或 `OPENCLAW_GATEWAY_TOKEN`／`OPENCLAW_GATEWAY_PASSWORD`），非迴環反向 Proxy 設定則可使用 `gateway.auth.mode: "trusted-proxy"`。

## OpenAI 相容端點

OpenClaw 最具效益的相容性介面：

- `GET /v1/models`
- `GET /v1/models/{id}`
- `POST /v1/embeddings`
- `POST /v1/chat/completions`
- `POST /v1/responses`

此端點組合的重要性：

- 大多數 Open WebUI、LobeChat 與 LibreChat 整合會先探測 `/v1/models`。
- 許多 RAG 與記憶管線預期使用 `/v1/embeddings`。
- 代理程式原生用戶端日益偏好 `/v1/responses`。

`/v1/models` 以代理程式為優先：它會針對每個已設定的代理程式傳回 `openclaw`、`openclaw/default` 與 `openclaw/<agentId>`。`openclaw/default` 是穩定別名，一律對應至已設定的預設代理程式。若要覆寫後端供應商／模型，請傳送 `x-openclaw-model`；否則仍由所選代理程式的一般模型與嵌入設定掌控。

所有這些端點都在主要閘道連接埠上執行，並與閘道 HTTP API 的其他部分共用相同的受信任操作員驗證邊界。

管理員 HTTP RPC（`POST /api/v1/admin/rpc`）是獨立且預設關閉的外掛路由，供無法使用 WebSocket RPC 的主機工具使用。請參閱[管理員 HTTP RPC](/zh-TW/plugins/admin-http-rpc)。

### 連接埠與繫結優先順序

| 設定         | 解析順序                                                             |
| ------------ | -------------------------------------------------------------------- |
| 閘道連接埠   | `--port` → `OPENCLAW_GATEWAY_PORT` → `gateway.port` → `18789`        |
| 繫結模式     | 命令列介面／覆寫值 → `gateway.bind` → `loopback`（容器內則為 `auto`） |

已安裝的閘道服務會將解析後的 `--port` 記錄於監督程式中繼資料。變更 `gateway.port` 後，請執行 `openclaw doctor --fix` 或 `openclaw gateway install --force`，讓 launchd／systemd／schtasks 在新連接埠上啟動程序。

針對非迴環繫結，閘道啟動時會使用相同的有效連接埠與繫結來植入本機控制介面來源。例如，`--bind lan --port 3000` 會在執行階段驗證前植入 `http://localhost:3000` 與 `http://127.0.0.1:3000`。請明確將任何遠端瀏覽器來源（例如 HTTPS Proxy URL）新增至 `gateway.controlUi.allowedOrigins`。

### 熱重新載入模式

| `gateway.reload.mode` | 行為                                       |
| --------------------- | ------------------------------------------ |
| `off`                 | 不重新載入設定                             |
| `hot`                 | 僅套用可安全熱更新的變更                   |
| `restart`             | 發生需要重新載入的變更時重新啟動           |
| `hybrid`（預設）     | 安全時熱套用，需要時重新啟動               |

## 操作員命令集

```bash
openclaw gateway status
openclaw gateway status --deep   # 新增系統層級服務掃描
openclaw gateway status --json
openclaw gateway install
openclaw gateway restart
openclaw gateway stop
openclaw secrets reload
openclaw logs --follow
openclaw doctor
```

`gateway status --deep` 用於額外服務探索（LaunchDaemons／systemd 系統單元／schtasks），而非更深入的 RPC 健康探測。

## 多個閘道（同一主機）

大多數安裝環境每台機器應執行一個閘道。單一閘道可承載多個代理程式與頻道。只有刻意需要隔離或救援機器人時，才需要多個閘道。

實用檢查：

```bash
openclaw gateway status --deep
openclaw gateway probe
```

預期情況：

- `gateway status --deep` 可在仍存在過時的 launchd／systemd／schtasks 安裝項目時回報 `Other gateway-like services detected (best effort)`，並顯示清理提示。
- 當不同閘道回應，或 OpenClaw 無法證明可連線的目標是同一個閘道時，`gateway probe` 可能會警告 `multiple reachable gateway identities`。即使傳輸連接埠不同，指向同一閘道的 SSH 通道、Proxy URL 或已設定的遠端 URL，仍是具有多種傳輸方式的單一閘道。
- 若這是刻意的安排，請為各閘道隔離連接埠、設定／狀態與工作區根目錄。

各執行個體檢查清單：

- 唯一的 `gateway.port`
- 唯一的 `OPENCLAW_CONFIG_PATH`
- 唯一的 `OPENCLAW_STATE_DIR`
- 唯一的 `agents.defaults.workspace`

範例：

```bash
OPENCLAW_CONFIG_PATH=~/.openclaw/a.json OPENCLAW_STATE_DIR=~/.openclaw-a openclaw gateway --port 19001
OPENCLAW_CONFIG_PATH=~/.openclaw/b.json OPENCLAW_STATE_DIR=~/.openclaw-b openclaw gateway --port 19002
```

詳細設定：[/gateway/multiple-gateways](/zh-TW/gateway/multiple-gateways)。

## 遠端存取

首選：Tailscale／VPN。
替代方案：SSH 通道。

```bash
ssh -N -L 18789:127.0.0.1:18789 user@gateway-host
```

接著，讓用戶端在本機連線至 `ws://127.0.0.1:18789`。

<Warning>
SSH 通道不會略過閘道驗證。使用共用密鑰驗證時，即使透過通道，用戶端仍
必須傳送 `token`／`password`。使用帶有身分資訊的模式時，
要求仍必須符合該驗證路徑。
</Warning>

請參閱：[遠端閘道](/zh-TW/gateway/remote)、[驗證](/zh-TW/gateway/authentication)、[Tailscale](/zh-TW/gateway/tailscale)。

## 監督與服務生命週期

在類正式環境中，請使用受監督執行以提高可靠性。

<Tabs>
  <Tab title="macOS (launchd)">

```bash
openclaw gateway install
openclaw gateway status
openclaw gateway restart
openclaw gateway stop
```

請使用 `openclaw gateway restart` 重新啟動。請勿串接 `openclaw gateway stop` 與 `openclaw gateway start` 來取代重新啟動。

在 macOS 上，`gateway stop` 預設使用 `launchctl bootout`。這會從目前的開機工作階段移除 LaunchAgent，但不會永久停用，因此在非預期當機後，KeepAlive 自動復原功能仍會運作，且 `gateway start` 可順利重新啟用。若要跨重新開機持續抑制自動重新產生程序，請傳入 `--disable`：`openclaw gateway stop --disable`。

LaunchAgent 標籤為 `ai.openclaw.gateway`（預設）或 `ai.openclaw.<profile>`（具名設定檔）。`openclaw doctor` 會稽核並修復服務設定偏移。

  </Tab>

  <Tab title="Linux (systemd user)">

```bash
openclaw gateway install
systemctl --user enable --now openclaw-gateway[-<profile>].service
openclaw gateway status
```

若要在登出後持續執行，請啟用 lingering：

```bash
sudo loginctl enable-linger $(whoami)
```

在沒有桌面工作階段的無頭伺服器上，重試 `systemctl --user` 命令前，另請確認已設定 `XDG_RUNTIME_DIR`（`export XDG_RUNTIME_DIR=/run/user/$(id -u)`）。

需要自訂安裝路徑時的手動使用者單元範例：

```ini
[Unit]
Description=OpenClaw Gateway
After=network-online.target
Wants=network-online.target
StartLimitBurst=5
StartLimitIntervalSec=60

[Service]
ExecStart=/usr/local/bin/openclaw gateway --port 18789
Restart=always
RestartSec=5
RestartPreventExitStatus=78
TimeoutStopSec=30
TimeoutStartSec=30
SuccessExitStatus=0 143
OOMPolicy=continue
KillMode=control-group

[Install]
WantedBy=default.target
```

  </Tab>

  <Tab title="Windows (native)">

```powershell
openclaw gateway install
openclaw gateway status --json
openclaw gateway restart
openclaw gateway stop
```

Windows 原生受管理啟動功能使用名為 `OpenClaw Gateway`
（具名設定檔則為 `OpenClaw Gateway (<profile>)`）的排定工作。若建立排定工作
遭拒，OpenClaw 會退回使用每位使用者的 Startup 資料夾啟動器，
並指向狀態目錄內的 `gateway.cmd`。

  </Tab>

  <Tab title="Linux (system service)">

多使用者／持續執行的主機請使用系統單元。

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now openclaw-gateway[-<profile>].service
```

使用與使用者單元相同的服務內容，但將其安裝於
`/etc/systemd/system/openclaw-gateway[-<profile>].service`，若你的 `openclaw` 二進位檔位於其他位置，
請調整 `ExecStart=`。

請勿同時讓 `openclaw doctor --fix` 為相同設定檔／連接埠安裝使用者層級的閘道服務。Doctor 發現系統層級 OpenClaw 閘道服務時，會拒絕該自動安裝；若生命週期由系統單元管理，請使用 `OPENCLAW_SERVICE_REPAIR_POLICY=external`。

  </Tab>
</Tabs>

無效設定錯誤會以代碼 `78` 結束。Linux systemd 單元使用 `RestartPreventExitStatus=78`，在設定修正前停止重新啟動。launchd 與 Windows Task Scheduler 沒有依退出代碼停止的對等規則，因此閘道也會保存短時間內未正常啟動的歷程，並在重複啟動失敗後抑制頻道／供應商帳號的自動啟動。在該安全模式下，控制平面仍會啟動以供檢查與修復；設定熱重新載入與 `secrets.reload` 會拒絕自動重新啟動頻道，而操作員明確提出的 `channels.start` 要求可以覆寫此抑制。

## 開發設定檔快速流程

```bash
openclaw --dev setup
openclaw --dev gateway --allow-unconfigured
openclaw --dev status
```

預設值包含隔離的狀態／設定，以及基礎閘道連接埠 `19001`。

## 通訊協定快速參考（操作員視角）

- 第一個用戶端框架必須是 `connect`。
- 閘道會傳回 `hello-ok` 框架，其中包含 `snapshot`（`presence`、`health`、`stateVersion`、`uptimeMs`）以及 `policy` 限制（`maxPayload`、`maxBufferedBytes`、`tickIntervalMs`）。
- `hello-ok.features.methods` / `events` 是保守的探索清單，而非
  每個可呼叫輔助路由的自動產生完整傾印。
- 請求：`req(method, params)` → `res(ok/payload|error)`。
- 常見事件包括 `connect.challenge`、`agent`、`chat`、
  `session.message`、`session.operation`、`session.tool`、選擇啟用的
  `session.approval`、`sessions.changed`、`presence`、`tick`、`health`、
  `heartbeat`、配對／核准生命週期事件，以及 `shutdown`。

代理程式執行分為兩個階段：

1. 立即接受確認（`status:"accepted"`）
2. 最終完成回應（`status:"ok"|"error"`），期間會串流 `agent` 事件。

請參閱完整通訊協定文件：[閘道通訊協定](/zh-TW/gateway/protocol)。

## 操作檢查

### 存活狀態

- 開啟 WS 並傳送 `connect`。
- 預期收到包含快照的 `hello-ok` 回應。

### 就緒狀態

```bash
openclaw gateway status
openclaw channels status --probe
openclaw health
```

### 缺口復原

事件不會重播。遇到序列缺口時，請先重新整理狀態（`health`、`system-presence`），再繼續執行。

## 常見失敗特徵

| 特徵                                                           | 可能的問題                                                                        |
| -------------------------------------------------------------- | ----------------------------------------------------------------------------- |
| `refusing to bind gateway ... without auth`                    | 綁定至非迴路介面，但沒有有效的閘道驗證路徑                                           |
| `another gateway instance is already listening` / `EADDRINUSE` | 連接埠衝突                                                                        |
| `Gateway start blocked: set gateway.mode=local`                | 設定為遠端模式，或損壞的設定中缺少 `gateway.mode` |
| 連線期間出現 `unauthorized`                                  | 用戶端與閘道之間的驗證不相符                                                        |

如需完整的診斷步驟，請使用[閘道疑難排解](/zh-TW/gateway/troubleshooting)。

## 安全保證

- 閘道通訊協定用戶端會在閘道無法使用時快速失敗（不會隱含地直接退回頻道路徑）。
- 無效或非連線類型的第一個框架會被拒絕並關閉。
- 正常關閉會在通訊端關閉前發出 `shutdown` 事件。

## 相關內容

- [設定](/zh-TW/gateway/configuration)
- [閘道疑難排解](/zh-TW/gateway/troubleshooting)
- [背景程序](/zh-TW/gateway/background-process)
- [健康狀態](/zh-TW/gateway/health)
- [診斷工具](/zh-TW/gateway/doctor)
- [驗證](/zh-TW/gateway/authentication)
- [遠端存取](/zh-TW/gateway/remote)
- [機密管理](/zh-TW/gateway/secrets)
