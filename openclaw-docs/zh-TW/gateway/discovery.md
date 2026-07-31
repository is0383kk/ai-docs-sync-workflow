---
read_when:
    - 實作或變更 Bonjour 探索／廣告發布
    - 調整遠端連線模式（直接連線與 SSH）
    - 設計遠端節點的探索與配對機制
summary: 用於尋找閘道的節點探索與傳輸方式（Bonjour、Tailscale、SSH）
title: 探索與傳輸方式
x-i18n:
    generated_at: "2026-07-26T08:23:28Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 3a3f1a6a1212ab0bc7021e77c88de059edcb8e09eff90d3e1e59451b9b20876b
    source_path: gateway/discovery.md
    workflow: 16
---

OpenClaw 有兩個彼此相關但不同的探索問題：

1. **操作員遠端控制**：macOS 選單列應用程式控制在其他位置執行的閘道。
2. **節點配對**：iOS/Android（及未來的節點）尋找閘道並進行安全配對。

所有網路探索／廣告功能都位於**節點閘道**
（`openclaw gateway`）；用戶端（Mac 應用程式、iOS）只負責使用這些資訊。

## 術語

- **閘道**：單一長時間執行的程序，負責管理狀態（工作階段、
  配對、節點登錄檔）並執行頻道。大多數設定會在每部主機上使用一個；
  也可以設定彼此隔離的多個閘道。
- **閘道 WS（控制平面）**：預設位於 `127.0.0.1:18789`
  的 WebSocket 端點；透過 `gateway.bind` 將其繫結至區域網路／tailnet。
- **直接 WS 傳輸**：面向區域網路／tailnet 的閘道 WS 端點（不使用 SSH）。
- **SSH 傳輸（備援）**：透過 SSH 轉送
  `127.0.0.1:18789` 來進行遠端控制。
- **舊版 TCP 橋接器（已移除）**：較舊的節點傳輸方式（請參閱
  [橋接器通訊協定](/zh-TW/gateway/bridge-protocol)）；探索功能不再廣告此方式，
  目前的建置版本也不再包含此方式。

通訊協定詳細資料：[閘道通訊協定](/zh-TW/gateway/protocol)、
[橋接器通訊協定（舊版）](/zh-TW/gateway/bridge-protocol)。

## 為何直接連線和 SSH 會同時存在

- **直接 WS** 在相同網路及 tailnet 內可提供最佳使用者體驗：透過 Bonjour
  在區域網路上自動探索、由閘道管理配對權杖與 ACL，
  而且不需要 Shell 存取權。
- **SSH** 是通用的備援方式：只要能透過 SSH 存取，無論位於何處皆可使用，
  即使跨越互不相關的網路也可以運作，且不受多播／mDNS 問題影響，
  除了 SSH 之外不需要新的傳入連接埠。

## 探索輸入

### 1) Bonjour / DNS-SD

多播 Bonjour 是盡力而為的機制，無法跨越網路。OpenClaw 也支援透過已設定的廣域 DNS-SD
網域瀏覽相同的閘道信標，因此探索範圍可以同時涵蓋相同區域網路上的
`local.`，以及用於跨網路探索的已設定單播 DNS-SD 網域。

啟用隨附的 `bonjour` 外掛時，**閘道**會透過 Bonjour 廣告其 WS 端點；
用戶端會瀏覽並顯示「選擇閘道」清單，然後儲存所選端點。

疑難排解與信標詳細資料：[Bonjour](/zh-TW/gateway/bonjour)。

#### 服務信標詳細資料

- 服務類型：`_openclaw-gw._tcp`（閘道傳輸信標）。
- TXT 鍵（非機密）：

  | 鍵                          | 備註                                                                                                                                                             |
  | --------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------- |
  | `role=gateway`              | 一律存在。                                                                                                                                                       |
  | `transport=gateway`         | 一律存在。                                                                                                                                                       |
  | `displayName=<name>`        | 由操作員設定的顯示名稱。                                                                                                                                         |
  | `lanHost=<hostname>.local`  | 僅限區域網路 mDNS 廣告器；不由廣域 DNS-SD 寫入。                                                                                                                 |
  | `gatewayPort=18789`         | 閘道 WS + HTTP 連接埠。                                                                                                                                          |
  | `gatewayTls=1`              | 僅在啟用 TLS 時存在。                                                                                                                                            |
  | `gatewayTlsSha256=<sha256>` | 僅在啟用 TLS 且可取得指紋時存在。                                                                                                                                |
  | `tailnetDns=<magicdns>`     | 選用提示；可使用 Tailscale 時會自動偵測。                                                                                                                        |
  | `sshPort=<port>`            | 僅在 `discovery.mdns.mode="full"` 時存在；在預設 `"minimal"` 模式中會省略（SSH 預設為 `22`），區域網路廣告器與廣域 DNS-SD 皆是如此。 |
  | `cliPath=<path>`            | 使用與 `sshPort` 相同的 `discovery.mdns.mode="full"` 閘門；供命令列介面路徑使用的遠端安裝提示。                                                                  |

  外掛探索合約中定義了 `canvasPort` TXT 鍵，供未來的畫布主機連接埠使用，
  但目前沒有任何程式碼路徑設定其值，因此現今絕不會發出此鍵。

安全性注意事項：

- Bonjour/mDNS TXT 記錄**未經驗證**。用戶端必須僅將 TXT
  值視為使用者體驗提示。
- 路由（主機／連接埠）應優先採用**解析後的服務端點**
  （SRV + A/AAAA），而非 TXT 提供的 `lanHost`、`tailnetDns` 或 `gatewayPort`。
- TLS 釘選絕不可讓廣告的 `gatewayTlsSha256` 覆寫
  先前儲存的釘選。
- 只要選擇的路由是安全／以 TLS 為基礎，iOS/Android 節點在儲存首次使用的釘選前，
  應要求明確確認「信任此指紋」（頻外驗證）。

啟用、停用及覆寫：

- `openclaw plugins enable bonjour` 會啟用區域網路多播廣告。
- `openclaw.json` 中的 `discovery.mdns.mode` 控制 mDNS 廣播：
  `"minimal"`（預設）、`"full"`（將 `cliPath`/`sshPort` 同時加入區域網路
  信標與任何廣域 DNS-SD 區域），或 `"off"`（停用 mDNS）。
- `OPENCLAW_DISABLE_BONJOUR=1` 會強制停用廣告；`discovery.mdns.mode="off"`
  則會獨立停用廣告。`OPENCLAW_DISABLE_BONJOUR=0` 是明確的選擇加入設定，
  可覆寫外掛在偵測到容器（Docker、containerd、Kubernetes、LXC）內時的自動停用行為；
  但不會覆寫 `discovery.mdns.mode="off"`。隨附的 `bonjour` 外掛會在
  macOS 主機（`enabledByDefaultOnPlatforms: ["darwin"]`）上自動啟動，並在偵測到容器內時自動停用；
  Linux、Windows 及其他容器化部署需要明確設定 `plugins enable bonjour`。
- `~/.openclaw/openclaw.json` 中的 `gateway.bind` 控制閘道繫結模式。
- `OPENCLAW_SSH_PORT` 會覆寫廣告的 SSH 連接埠（僅在
  `discovery.mdns.mode="full"` 時生效）。
- `OPENCLAW_TAILNET_DNS` 會發布 `tailnetDns` 提示（MagicDNS）。
- `OPENCLAW_CLI_PATH` 會覆寫廣告的命令列介面路徑。

### 2) Tailnet（跨網路）

對位於不同實體網路上的閘道而言，Bonjour 無法提供協助。建議的直接目標是
Tailscale MagicDNS 名稱（優先）或穩定的 tailnet IP。

如果閘道偵測到自己在 Tailscale 下執行，便會發布
`tailnetDns` 作為用戶端的選用提示（包括廣域信標）。
macOS 應用程式在探索閘道時，會優先採用 MagicDNS 名稱而非原始 Tailscale IP；
因此當 tailnet IP 因節點重新啟動或 CGNAT 重新指派而變更時，仍可維持可靠性，
因為 MagicDNS 會自動解析至目前的 IP。

對於行動節點配對，探索提示絕不會放寬 tailnet／公用路由上的傳輸安全性：

- iOS/Android 仍需要安全的首次 tailnet／公用連線路徑
  （`wss://` 或 Tailscale Serve/Funnel）。
- 探索到的原始 tailnet IP 是路由提示，並非使用
  明文遠端 `ws://` 的許可。
- 仍支援私人區域網路直接連線 `ws://`。
- 若要在行動節點上使用最簡單的 Tailscale 路徑，請使用 Tailscale Serve，
  讓探索與設定都解析至相同的安全 MagicDNS 端點。

### 3) 手動／SSH 目標

沒有直接路由（或直接連線已停用）時，用戶端仍可透過 SSH 轉送
迴路閘道連接埠來連線。請參閱[遠端存取](/zh-TW/gateway/remote)。

## 傳輸選擇（用戶端政策）

1. 如果已設定配對的直接端點且可以連線，則使用該端點。
2. 否則，如果探索功能在 `local.` 或已設定的廣域
   網域中找到閘道，請提供一鍵「使用此閘道」選項，並將其儲存為
   直接端點。
3. 否則，如果已設定 tailnet DNS/IP，則嘗試直接連線。對使用
   tailnet／公用路由的行動節點而言，直接連線表示安全端點，而非明文
   遠端 `ws://`。
4. 否則，回退至 SSH。

## 配對與驗證（直接傳輸）

閘道是節點／用戶端准入的真實資料來源：

- 配對要求會在閘道中建立／核准／拒絕（請參閱
  [閘道配對](/zh-TW/gateway/pairing)）。
- 閘道會強制執行驗證（權杖／金鑰組）、範圍／ACL（並非
  對所有方法提供原始代理），以及速率限制。

## 各元件的責任

- **閘道**：廣告探索信標、負責配對決策，並託管
  WS 端點。
- **macOS 應用程式**：協助你選擇閘道、顯示配對提示，且僅將 SSH
  作為備援方式。
- **iOS/Android 節點**：為了便利性而瀏覽 Bonjour，並連線至
  已配對的閘道 WS。

## 相關內容

- [遠端存取](/zh-TW/gateway/remote)
- [Tailscale](/zh-TW/gateway/tailscale)
- [Bonjour 探索](/zh-TW/gateway/bonjour)
