---
read_when:
    - 你想要針對 SSRF 和 DNS 重新綁定攻擊採取縱深防禦措施
    - 為 OpenClaw 執行階段流量設定外部正向代理伺服器
summary: 如何透過由操作者管理的篩選 Proxy 路由 OpenClaw 執行階段的 HTTP 與 WebSocket 流量
title: 網路代理伺服器
x-i18n:
    generated_at: "2026-07-26T08:06:46Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: e948189d691e2cfe32e911e24071fd77157397b510d606423ef738c2565071b5
    source_path: security/network-proxy.md
    workflow: 16
---

OpenClaw 可透過由營運者管理的正向 Proxy 路由執行階段的 HTTP 與 WebSocket 流量。這是選用的縱深防禦措施：可集中控管對外流量、提供更強的 SSRF 防護，並在網路邊界稽核目的地。由於 Proxy 會在連線時、DNS 解析之後且即將開啟上游連線之前評估目的地，因此也能縮小 DNS 重新綁定攻擊所依賴的時間差，也就是先前應用程式層級的 DNS 檢查與實際對外連線之間的落差。單一 Proxy 政策也讓營運者能在同一處強制執行目的地規則、網路分段、速率限制或對外允許清單，而無須重新建置 OpenClaw。

OpenClaw 不會隨附、下載、啟動、設定或認證任何 Proxy。你應執行適合自身環境的 Proxy 技術；OpenClaw 會透過該 Proxy 路由自身的 HTTP 與 WebSocket 用戶端。

## 設定

```yaml
proxy:
  proxyUrl: http://127.0.0.1:3128
```

你也可以透過環境變數設定 URL：

```bash
OPENCLAW_PROXY_URL=http://127.0.0.1:3128 openclaw gateway run
```

`proxy.proxyUrl` 的優先順序高於 `OPENCLAW_PROXY_URL`。設定 URL 後會啟用受管理的 Proxy 路由；移除這兩個 URL 則會停用它。

| 鍵                   | 類型                                 | 預設值         | 備註                                                                                                                                 |
| -------------------- | ------------------------------------ | -------------- | ------------------------------------------------------------------------------------------------------------------------------------- |
| `proxy.proxyUrl`     | 字串                                 | 未設定          | `http://` 或 `https://` 正向 Proxy URL。嵌入 URL 的認證資訊會視為敏感資訊，並從快照／記錄中遮蔽。 |
| `proxy.tls.caFile`   | 字串                               | 未設定          | 用於驗證由私有 CA 簽署之 `https://` Proxy 端點的 CA 套件。                                                          |
| `proxy.loopbackMode` | `gateway-only` \| `proxy` \| `block` | `gateway-only` | 控制迴路位址略過行為；請參閱下文。                                                                                         |

對於受管理的閘道服務，請將 URL 儲存在設定中，使其在重新安裝後仍能保留，而非依賴前景環境變數：

```bash
openclaw config set proxy.proxyUrl http://127.0.0.1:3128
openclaw gateway install --force
openclaw gateway start
```

`OPENCLAW_PROXY_URL` 環境變數後援最適合前景執行。若要搭配已安裝的服務使用，請將它放入服務的持久環境（`$OPENCLAW_STATE_DIR/.env`，預設為 `~/.openclaw/.env`），然後重新安裝，讓 launchd/systemd/Scheduled Tasks 載入它。

### 使用私有 CA 的 HTTPS Proxy 端點

```yaml
proxy:
  proxyUrl: https://proxy.corp.example:8443
  tls:
    caFile: /etc/openclaw/proxy-ca.pem
```

`proxy.tls.caFile` 會驗證 Proxy 端點本身的 TLS 憑證。它不是目的地 MITM 信任設定、用戶端憑證，也不能取代 Proxy 的目的地政策。只有當整個 Node 程序必須從啟動時信任額外的 CA（例如企業 TLS 檢查系統會重新簽署每個 HTTPS 目的地憑證）時，才應改用 `NODE_EXTRA_CA_CERTS`；該變數作用於整個程序，且必須在 Node 啟動前設定，因此 OpenClaw 無法像套用 `proxy.tls.caFile` 一樣在執行期間套用它。HTTPS Proxy 端點信任應優先使用 `proxy.tls.caFile`：它的作用範圍僅限受管理的 Proxy 路由，而非整個程序。

```bash
openclaw config set proxy.proxyUrl https://proxy.corp.example:8443
openclaw config set proxy.tls.caFile /etc/openclaw/proxy-ca.pem
openclaw gateway run
```

## 路由運作方式

設定有效的 Proxy URL 後，受保護的執行階段程序（`openclaw gateway run`、`openclaw node run`、`openclaw agent --local`）會透過 Proxy 路由一般 HTTP 與 WebSocket 對外流量：

```text
OpenClaw 程序
  fetch、node:http、node:https、WebSocket 用戶端  -> 營運者 Proxy -> 目的地
```

在內部，OpenClaw 會安裝 [Proxyline](https://github.com/openclaw/proxyline) 作為程序層級的路由執行階段。它涵蓋 `fetch`、以 undici 為基礎的用戶端、`node:http`/`node:https`、常見的 WebSocket 用戶端，以及由輔助程式建立的 `CONNECT` 通道；同時還會取代呼叫端提供的 Node HTTP 代理程式，因此明確指定的代理程式（包括 `axios`、`got`、`node-fetch` 及類似的 Node 代理程式型用戶端）無法在未察覺的情況下略過 Proxy。

Proxy URL 配置的通訊協定描述的是 OpenClaw 到 Proxy 之間的區段，而非前往最終目的地的區段：

- `http://proxy.example:3128` — 以純 TCP 連線至 Proxy；OpenClaw 會傳送 HTTP Proxy 請求，包括針對 HTTPS 目的地的 `CONNECT`。
- `https://proxy.example:8443` — OpenClaw 會對 Proxy 本身開啟 TLS（並驗證 Proxy 的憑證），然後在該工作階段內傳送 HTTP Proxy 請求。

目的地 TLS 與 Proxy 端點 TLS 彼此獨立：對於 HTTPS 目的地，OpenClaw 一律會要求 Proxy 建立 `CONNECT` 通道，並透過該通道啟動目的地 TLS。

Proxy 啟用期間，OpenClaw 會清除 `no_proxy`/`NO_PROXY`。這些略過清單以目的地為依據；若其中保留 `localhost` 或 `127.0.0.1`，SSRF 目標就能完全略過 Proxy。關閉時，OpenClaw 會還原先前的 Proxy 環境，並重設快取的路由狀態。

即使已啟用程序層級路由，部分外掛仍擁有需要個別連接 Proxy 的自訂傳輸。Telegram 的 Bot API 用戶端使用自己的 HTTP/1 undici 分派器，並另行遵循程序 Proxy 環境及 `OPENCLAW_PROXY_URL` 後援。

### 閘道迴路位址模式

本機閘道控制平面用戶端通常會連線至迴路位址 WebSocket，例如 `ws://127.0.0.1:18789`。`proxy.loopbackMode` 控制該流量是否略過受管理的 Proxy：

```yaml
proxy:
  proxyUrl: http://127.0.0.1:3128
  loopbackMode: gateway-only # gateway-only、proxy 或 block
```

設定 `proxyUrl` 或 `OPENCLAW_PROXY_URL` 後會啟用受管理的路由。只有在需要保留已儲存的 URL
但不啟用它時，才應將 `proxy.enabled: false` 設為進階退出選項。

| 模式                     | 行為                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                            |
| ------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `gateway-only`（預設） | OpenClaw 會將使用中的閘道迴路位址授權單位註冊為直接連線例外，因此本機閘道 WebSocket 流量無須透過 Proxy 即可連線。自訂迴路位址連接埠也能運作，因為例外會精確指定已設定的主機／連接埠。隨附的瀏覽器外掛會針對由 OpenClaw 啟動之受管理瀏覽器的確切本機 CDP 就緒狀態與 DevTools WebSocket URL，註冊相同類型的例外；隨附的 Ollama 記憶嵌入提供者則為其確切設定的主機本機迴路位址嵌入來源，提供範圍更窄且受保護的直接路徑。 |
| `proxy`                  | 不會註冊任何迴路位址例外；閘道與 Ollama 迴路位址流量會透過 Proxy。遠端 Proxy 必須能將流量路由回 OpenClaw 主機的迴路位址服務（例如透過可連線的主機名稱、IP 或通道）——標準遠端 Proxy 會相對於自身而非 OpenClaw 主機解析 `127.0.0.1`/`localhost`。                                                                                                                                                                                                                |
| `block`                  | OpenClaw 會在開啟通訊端之前拒絕閘道迴路位址控制平面連線，以及受保護的 Ollama 迴路位址嵌入連線。                                                                                                                                                                                                                                                                                                                                                                                                                               |

閘道控制平面略過僅限於 `localhost` 與明確的迴路位址 IP URL——請使用 `ws://127.0.0.1:18789`、`ws://[::1]:18789` 或 `ws://localhost:18789`。其他主機名稱會像一般流量一樣路由。

### 容器

對於 `openclaw --container ...` 命令，設定 `OPENCLAW_PROXY_URL` 時，OpenClaw 會將其轉送給以容器為目標的子命令列介面。該 URL 必須能從容器內部連線——容器中的 `127.0.0.1` 指的是容器本身，而非主機。除非你設定 `OPENCLAW_CONTAINER_ALLOW_LOOPBACK_PROXY_URL=1` 以明確覆寫此檢查，否則 OpenClaw 會拒絕以容器為目標之命令所使用的迴路位址 Proxy URL。

## 相關 Proxy 詞彙

- `proxy.enabled` / `proxy.proxyUrl` — 執行階段對外流量的正向 Proxy 路由。本頁所述主題。
- `gateway.auth.mode: "trusted-proxy"` — 用於存取閘道的入站身分感知反向 Proxy 驗證。請參閱[受信任的 Proxy 驗證](/zh-TW/gateway/trusted-proxy-auth)。
- `openclaw proxy` — 用於開發與支援的本機偵錯 Proxy 及擷取檢查器。請參閱 [openclaw proxy](/zh-TW/cli/proxy)。
- `tools.web.fetch.useTrustedEnvProxy` — `web_fetch` 的選擇性啟用項目，允許營運者控制的 HTTP(S) 環境 Proxy 解析 DNS，同時預設維持嚴格的 DNS 固定與主機名稱政策。請參閱[網頁擷取](/zh-TW/tools/web-fetch#trusted-env-proxy)。
- 頻道或提供者專用的 Proxy 設定 — 單一傳輸的擁有者專用覆寫。若要集中控管整個執行階段的對外流量，應優先使用受管理的網路 Proxy。

## 驗證 Proxy

Proxy 的目的地政策才是實際的安全邊界；OpenClaw 無法驗證你的 Proxy 是否封鎖了正確的目標。請將其設定為：

- 僅繫結至迴路位址或私有受信任介面，且只有 OpenClaw 程序／主機／容器／服務帳戶能夠連線。
- 自行解析目的地，並在 DNS 解析後、連線時依 IP 封鎖，且同時套用於純 HTTP 與 HTTPS `CONNECT` 通道。
- 拒絕針對迴路位址、私有、連結本機、中繼資料、多點傳播、保留及文件用途範圍的目的地型略過規則。
- 除非你完全信任 DNS 解析路徑，否則請避免使用主機名稱允許清單。
- 記錄目的地、決策、狀態與原因——絕不可記錄請求本文、授權標頭、Cookie 或其他機密。
- 將政策納入版本控制，並將其變更視為安全性敏感項目進行審查。

請從執行 OpenClaw 的相同主機／容器／服務帳戶進行驗證：

```bash
openclaw proxy validate --proxy-url http://127.0.0.1:3128
```

使用私有 CA 的 HTTPS Proxy 端點：

```bash
openclaw proxy validate --proxy-url https://proxy.corp.example:8443 --proxy-ca-file /etc/openclaw/proxy-ca.pem
```

| 旗標                     | 用途                                                              |
| ------------------------ | -------------------------------------------------------------------- |
| `--proxy-url <url>`      | 驗證此 URL，而不是解析設定／環境變數。                   |
| `--proxy-ca-file <path>` | HTTPS Proxy 端點的 CA 組合包。                               |
| `--allowed-url <url>`    | 預期可成功連線的目的地（可重複指定）。                        |
| `--denied-url <url>`     | 預期會遭封鎖的目的地（可重複指定）。                     |
| `--apns-reachable`       | 另行驗證 Proxy 是否能透過隧道直接執行沙箱 APNs HTTP/2 探測。 |
| `--apns-authority <url>` | 覆寫使用 `--apns-reachable` 探測的 APNs authority。          |
| `--timeout-ms <ms>`      | 每個要求的逾時時間。                                                 |
| `--json`                 | 機器可讀的輸出。                                             |

如果沒有可用的設定、環境變數或 `--proxy-url` 值，此命令會回報設定問題；在變更設定之前，請傳入 `--proxy-url` 進行一次性預檢。

未指定 `--allowed-url`/`--denied-url` 時，預設檢查為：`https://example.com/` 必須成功，且 Proxy 不得連線至為此臨時建立的回送金絲雀伺服器，該連線必須遭到封鎖。回送檢查在傳輸失敗時會通過；若收到不含金絲雀單次執行權杖的非 2xx 回應，也會通過。若收到缺少權杖的 2xx 回應，則檢查失敗（表示非金絲雀的某個服務意外成功回應）；尤其是收到任何帶有相符權杖的回應時，也會失敗，因為這證明 Proxy 確實轉送了原應拒絕的回送目的地。自訂 `--denied-url` 目標沒有這類金絲雀權杖，因此會採取失敗關閉策略：任何 HTTP 回應都代表可連線（失敗）；傳輸錯誤則回報為無法判定，而非證實已封鎖，因為 OpenClaw 無法確認究竟是你的 Proxy 拒絕了可連線的來源，還是發生其他錯誤。`--apns-reachable` 會傳送刻意無效的提供者權杖，因此 `403 InvalidProviderToken` 回應可作為隧道已連線至 Apple 的證明。任何驗證失敗時，命令皆以 `1` 結束；文字與 JSON 輸出都會遮蔽 Proxy URL 中的認證資訊。

```json
{
  "ok": true,
  "config": {
    "enabled": true,
    "proxyUrl": "http://127.0.0.1:3128/",
    "source": "override",
    "errors": []
  },
  "checks": [
    { "kind": "allowed", "url": "https://example.com/", "ok": true, "status": 200 },
    { "kind": "apns", "url": "https://api.sandbox.push.apple.com", "ok": true, "status": 403 }
  ]
}
```

手動 `curl` 檢查（公開要求應成功；回送和中繼資料要求應由 Proxy 本身封鎖——單靠 `curl` 無法像 `openclaw proxy validate` 的內建金絲雀一樣，區分 Proxy 拒絕與來源無法連線）：

```bash
curl -x http://127.0.0.1:3128 https://example.com/
curl -x http://127.0.0.1:3128 http://127.0.0.1/
curl -x http://127.0.0.1:3128 http://169.254.169.254/
```

## 建議封鎖的目的地

適用於任何正向 Proxy、防火牆或輸出流量原則的初始拒絕清單。OpenClaw 自有的 SSRF 分類器位於 `src/infra/net/ssrf.ts` 和 `packages/net-policy/src/ip.ts`（`BLOCKED_HOSTNAMES`、`BLOCKED_IPV4_SPECIAL_USE_RANGES`、`BLOCKED_IPV6_SPECIAL_USE_RANGES`、RFC 2544 基準測試前綴，以及針對 NAT64/6to4/Teredo/ISATAP/IPv4 對應格式的內嵌 IPv4 處理）——這些是實用的參考資料，但 OpenClaw 不會將這些規則匯出或套用至你的外部 Proxy。

| 範圍或主機                                                                        | 封鎖原因                                      |
| ------------------------------------------------------------------------------------ | ------------------------------------------------- |
| `127.0.0.0/8`、`localhost`、`localhost.localdomain`                                  | IPv4 回送                                     |
| `::1/128`                                                                            | IPv6 回送                                     |
| `0.0.0.0/8`、`::/128`                                                                | 未指定／本網路位址              |
| `10.0.0.0/8`、`172.16.0.0/12`、`192.168.0.0/16`                                      | RFC 1918 私有網路                         |
| `169.254.0.0/16`、`fe80::/10`                                                        | 連結本機，包括常見的雲端中繼資料路徑 |
| `169.254.169.254`、`metadata.google.internal`                                        | 雲端中繼資料服務                           |
| `100.64.0.0/10`                                                                      | 電信業者級 NAT 共用位址空間            |
| `198.18.0.0/15`、`2001:2::/48`                                                       | 基準測試範圍                               |
| `192.0.0.0/24`、`192.0.2.0/24`、`198.51.100.0/24`、`203.0.113.0/24`、`2001:db8::/32` | 特殊用途及文件範例範圍              |
| `224.0.0.0/4`、`ff00::/8`                                                            | 多播                                         |
| `240.0.0.0/4`                                                                        | 保留的 IPv4                                     |
| `fc00::/7`、`fec0::/10`                                                              | IPv6 本機／私有範圍                         |
| `100::/64`、`2001:20::/28`                                                           | IPv6 丟棄及 ORCHIDv2 範圍                  |
| `64:ff9b::/96`、`64:ff9b:1::/48`                                                     | 含有內嵌 IPv4 的 NAT64 前綴                 |
| `2002::/16`、`2001::/32`                                                             | 含有內嵌 IPv4 的 6to4 和 Teredo                |
| `::/96`、`::ffff:0:0/96`                                                             | IPv4 相容及 IPv4 對應 IPv6              |

加入你的雲端提供者或網路平台文件中列出的任何其他中繼資料主機或保留範圍。

## 限制

| 表面                                                      | 受管理 Proxy 狀態                                                                                                                                     |
| ------------------------------------------------------------ | -------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `fetch`、`node:http`、`node:https`、常見 WebSocket 用戶端 | 設定後，會透過受管理的 Proxy 掛鉤路由。                                                                                                      |
| APNs 直接 HTTP/2                                           | 透過 APNs 受管理的 `CONNECT` 輔助函式路由。                                                                                                        |
| 閘道控制平面回送                               | 僅針對確切設定的本機回送閘道 URL 直接連線。                                                                                         |
| 偵錯 Proxy 上游轉送                              | 受管理 Proxy 模式啟用時停用，除非為了本機診斷明確啟用。                                                             |
| IRC                                                          | 原始 TCP/TLS；不由受管理 HTTP Proxy 模式代理。如果你的部署要求所有輸出流量都透過正向 Proxy，請設定 `channels.irc.enabled: false`。 |
| 其他原始 `net`、`tls` 或 `http2` 用戶端呼叫              | 必須先由原始 Socket 防護機制分類，才能合併。                                                                                               |

- 這是 JavaScript HTTP/WebSocket 用戶端的處理程序層級涵蓋範圍，不是作業系統層級的網路沙箱。
- 除非繼承並遵循 Proxy 環境變數，否則原始 `net`、`tls`、`http2` Socket、原生附加元件，以及非 OpenClaw 子處理程序都可能繞過 Node 層級路由。分叉的 OpenClaw 子命令列介面會繼承受管理的 Proxy URL 與 `proxy.loopbackMode` 狀態。
- 使用者的本機 WebUI 與本機模型伺服器不適用一般的區域網路略過規則——如有需要，請在營運者的 Proxy 原則中將其加入允許清單。唯一例外是隨附的 Ollama 記憶嵌入提供者受防護的直接路徑，其範圍限於所設定 `baseUrl` 中確切的主機本機回送來源；區域網路、Tailnet、私有網路和公開 Ollama 主機仍使用受管理的 Proxy。
- 受管理 Proxy 模式啟用時，本機偵錯 Proxy 的直接上游轉送（適用於 Proxy 要求與 `CONNECT` 隧道）預設為停用；僅限經核准的本機診斷使用情境才應啟用。
- OpenClaw 不會檢查、測試或認證你的 Proxy 原則。請將 Proxy 原則變更視為安全性敏感的營運變更。
