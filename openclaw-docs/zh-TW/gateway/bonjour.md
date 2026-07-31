---
read_when:
    - 偵錯 macOS/iOS 上的 Bonjour 探索問題
    - 變更 mDNS 服務類型、TXT 記錄或探索使用者體驗
summary: Bonjour/mDNS 探索與偵錯（閘道信標、用戶端及常見故障模式）
title: Bonjour 探索
x-i18n:
    generated_at: "2026-07-26T07:18:27Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: f43ef71b323b59362655c390a4df621c2571abbe3b2c1cd2728918c6f76d6f99
    source_path: gateway/bonjour.md
    workflow: 16
---

OpenClaw 可以使用 Bonjour (mDNS/DNS-SD) 探索使用中的閘道（WebSocket 端點）。多點傳播 `local.` 瀏覽是一項**僅限區域網路的便利功能**：隨附的 `bonjour` 外掛負責區域網路廣告，會在 macOS 主機上自動啟動，而在 Linux、Windows 與容器化閘道部署中則需選擇啟用。同一個信標也能透過已設定的廣域 DNS-SD 網域發布，以進行跨網路探索。探索功能採盡力而為，且**無法**取代以 SSH 或 Tailnet 為基礎的連線方式。

## 透過 Tailscale 使用廣域 Bonjour（單點傳播 DNS-SD）

如果節點與閘道位於不同網路，多點傳播 mDNS 就無法跨越網路邊界。可透過 Tailscale 改用**單點傳播 DNS-SD**（「廣域 Bonjour」），以維持相同的探索使用體驗：

1. 在閘道主機上執行可透過 Tailnet 存取的 DNS 伺服器。
2. 在專用區域下發布 `_openclaw-gw._tcp` 的 DNS-SD 記錄（範例：`openclaw.internal.`）。
3. 設定 Tailscale **分割 DNS**，讓用戶端（包括 iOS）透過該 DNS 伺服器解析你選擇的網域。

上述 `openclaw.internal.` 只是一個範例——OpenClaw 支援任何探索網域。iOS/Android 節點會同時瀏覽 `local.` 與你設定的廣域網域。

### 閘道設定

```json5
{
  gateway: { bind: "tailnet" }, // 僅限 tailnet（建議）
  discovery: { wideArea: { enabled: true, domain: "openclaw.internal" } },
}
```

未設定時，`discovery.wideArea.domain` 也接受 `OPENCLAW_WIDE_AREA_DOMAIN` 環境變數作為備用值。

### 一次性 DNS 伺服器設定（閘道主機，僅限 macOS）

```bash
openclaw dns setup --apply
```

此命令僅適用於 macOS，並需要 Homebrew 與執行中的 Tailscale 連線。它會安裝 CoreDNS（`brew install coredns`）並將其設定為：

- 僅在閘道的 Tailscale 介面上監聽連接埠 53
- 從 `~/.openclaw/dns/<domain>.db` 提供你選擇的網域（範例：`openclaw.internal.`）

先不加 `--apply` 執行，即可在不安裝任何項目的情況下預覽計畫（網域、區域檔案路徑、偵測到的 Tailnet IP、建議設定）。

從已連線至 Tailnet 的機器進行驗證：

```bash
dns-sd -B _openclaw-gw._tcp openclaw.internal.
dig @<TAILNET_IPV4> -p 53 _openclaw-gw._tcp.openclaw.internal PTR +short
```

### Tailscale DNS 設定

在 Tailscale 管理主控台中：

- 新增指向閘道 Tailnet IP（UDP/TCP 53）的名稱伺服器。
- 新增分割 DNS，讓探索網域使用該名稱伺服器。

用戶端接受 Tailnet DNS 後，iOS 節點與命令列介面探索功能便可在你的探索網域中瀏覽 `_openclaw-gw._tcp`，無須使用多點傳播。

### 閘道接聽程式安全性

閘道 WS 連接埠（預設為 `18789`）預設繫結至迴送介面。若要從區域網路/Tailnet 存取，請明確設定繫結並保持驗證功能啟用。若設定為僅限 Tailnet，請在 `~/.openclaw/openclaw.json` 中設定 `gateway.bind: "tailnet"`，然後重新啟動閘道（或 macOS 選單列應用程式）。

## 廣告內容

只有閘道會廣告 `_openclaw-gw._tcp`。啟用後，區域網路多點傳播廣告由隨附的 `bonjour` 外掛負責；廣域 DNS-SD 發布仍由閘道負責。

## 服務類型

- `_openclaw-gw._tcp` - 閘道傳輸信標，由 macOS/iOS/Android 節點使用。

## TXT 鍵（非機密提示）

| 鍵                            | 出現時機                                                                       |
| ----------------------------- | ------------------------------------------------------------------------------ |
| `role=gateway`                | 一律出現。                                                                     |
| `displayName=<friendly name>` | 一律出現。                                                                     |
| `lanHost=<hostname>.local`    | 一律出現。                                                                     |
| `gatewayPort=<port>`          | 一律出現（閘道 WS + HTTP）。                                                   |
| `transport=gateway`           | 一律出現。                                                                     |
| `gatewayTls=1`                | 僅在啟用 TLS 時出現。                                                          |
| `gatewayTlsSha256=<sha256>`   | 僅在啟用 TLS 且有可用指紋時出現。                                              |
| `gatewayDirectReachable=1`    | 僅在閘道可直接連線時出現（而非只能透過轉送/Proxy 路徑）。                      |
| `canvasPort=<port>`           | 僅在啟用畫布主機時出現；目前與 `gatewayPort` 相同。                       |
| `tailnetDns=<magicdns>`       | 僅限 mDNS 完整模式；Tailnet 可用時的選用提示。                                 |
| `sshPort=<port>`              | 僅限完整模式；在最小與關閉模式中省略。                                         |
| `cliPath=<path>`              | 僅限完整模式；在最小與關閉模式中省略。                                         |

安全性注意事項：

- Bonjour/mDNS TXT 記錄**未經驗證**。用戶端不得將 TXT 視為具權威性的路由資訊。
- 用戶端應使用解析出的服務端點（SRV + A/AAAA）進行路由。`lanHost`、`tailnetDns`、`gatewayPort` 與 `gatewayTlsSha256` 僅應視為提示。
- SSH 自動選擇目標同樣應使用解析出的服務主機，而非僅使用 TXT 提示。
- TLS 固定絕不可讓廣告的 `gatewayTlsSha256` 覆寫先前儲存的固定值。
- iOS/Android 節點應將透過探索建立的直接連線視為**僅限 TLS**，並在信任首次出現的指紋前要求使用者明確確認。

## 在 macOS 上偵錯

內建工具：

```bash
# 瀏覽執行個體
dns-sd -B _openclaw-gw._tcp local.

# 解析一個執行個體（取代 <instance>）
dns-sd -L "<instance>" _openclaw-gw._tcp local.
```

如果瀏覽正常但解析失敗，通常是區域網路原則或 mDNS 解析程式發生問題。

## 在閘道記錄中偵錯

閘道會寫入輪替記錄檔（啟動時顯示為 `gateway log file: ...`）。請尋找 `bonjour:` 行，尤其是：

- `bonjour: advertise failed ...`
- `bonjour: suppressing ciao netmask assertion ...`
- `bonjour: ... name conflict resolved` / `hostname conflict resolved`

OpenClaw 只會啟動每個 Bonjour 服務一次，並將探測、重試、名稱衝突解決及介面變更後的重新發布交由 mDNS 回應程式處理。這可避免一般網路變動期間出現重疊的發布嘗試。系統會抑制重複的內部自我探測訊息，以免其大量湧入閘道記錄。

當多個 OpenClaw 閘道從同一主機發出廣告時，Bonjour 可能會附加 `(2)` 或 `(3)` 等尾碼，以維持服務執行個體名稱的唯一性。這些尾碼是正常的衝突解決機制，並不表示 OCM 受到重複監管。

當系統主機名稱是有效的 DNS 標籤時，Bonjour 會將其用於廣告的 `.local` 主機。如果系統主機名稱含有空格、底線或其他無效的 DNS 標籤字元，OpenClaw 會改用 `openclaw.local`。需要明確指定主機標籤時，請在啟動閘道前設定 `OPENCLAW_MDNS_HOSTNAME=<name>`。

## 在 iOS 節點上偵錯

iOS 節點使用 `NWBrowser` 探索 `_openclaw-gw._tcp`。

若要擷取記錄：Settings -> Gateway -> Advanced -> **Discovery Debug Logs**，接著前往 Settings -> Gateway -> Advanced -> **Discovery Logs** -> 重現問題 -> **Copy**。記錄包含瀏覽器狀態轉換與結果集變更。

## 何時啟用 Bonjour

在 macOS 主機以空白設定啟動閘道時，Bonjour 會自動啟動，因為本機應用程式與附近的 iOS/Android 節點通常依賴同一區域網路內的探索功能。

在 Linux、Windows 或其他非 macOS 主機上，若同一區域網路內的自動探索功能實用，請明確啟用：

```bash
openclaw plugins enable bonjour
```

啟用後，Bonjour 會使用 `discovery.mdns.mode` 決定要發布多少 TXT 中繼資料；同一模式也會控制廣域 DNS-SD 記錄中的選用 TXT 提示。模式如下：

| 模式                | 行為                                                                                                                                     |
| ------------------- | ---------------------------------------------------------------------------------------------------------------------------------------- |
| `minimal`（預設） | 僅包含核心 TXT 鍵；省略 `sshPort`、`cliPath`、`tailnetDns`。                                                    |
| `full`              | 新增 `sshPort`、`cliPath`、`tailnetDns`——用戶端需要這些提示時使用。                                             |
| `off`               | 在不變更外掛啟用狀態的情況下抑制區域網路多點傳播；設定 `discovery.wideArea.domain` 時，廣域 DNS-SD 仍可發布。                                     |

## 何時停用 Bonjour

若區域網路多點傳播廣告不必要、無法使用或有害，請維持 Bonjour 停用——常見情況包括非 macOS 伺服器、Docker 橋接網路、WSL，或會捨棄 mDNS 多點傳播的網路原則。閘道仍可透過其發布的 URL、SSH、Tailnet 或廣域 DNS-SD 存取；只有區域網路自動探索功能不可靠。

對於部署範圍內的問題，請使用環境變數覆寫（適用於 Docker 映像、服務檔案、啟動指令碼與一次性偵錯——環境消失時，此設定也會消失）：

```bash
OPENCLAW_DISABLE_BONJOUR=1
```

如果你有意針對該 OpenClaw 設定關閉隨附的區域網路探索外掛，請使用外掛設定：

```bash
openclaw plugins disable bonjour
```

## Docker 注意事項

偵測到容器時，若未設定 `OPENCLAW_DISABLE_BONJOUR`，隨附的 Bonjour 外掛會自動停用區域網路多點傳播廣告。Docker 橋接網路通常不會在容器與區域網路之間轉送 mDNS 多點傳播（`224.0.0.251:5353`），因此從容器發出廣告通常無法讓探索功能正常運作。

注意事項：

- Bonjour 會在 macOS 主機上自動啟動，而在其他平台則需選擇啟用。維持停用並不會停止閘道——只會略過區域網路多點傳播廣告。
- 停用 Bonjour 不會變更 `gateway.bind`；Docker 仍預設使用 `OPENCLAW_GATEWAY_BIND=lan`，因此發布的主機連接埠仍可運作。
- 停用 Bonjour 不會停用廣域 DNS-SD。當閘道與節點不在同一區域網路時，請使用廣域探索或 Tailnet。
- 在 Docker 外重複使用相同的 `OPENCLAW_CONFIG_DIR`，不會保留容器自動停用原則。
- 僅在主機網路、macvlan 或其他已知可傳遞 mDNS 多點傳播的網路中設定 `OPENCLAW_DISABLE_BONJOUR=0`；將其設為 `1` 可強制停用。

## 針對已停用的 Bonjour 進行疑難排解

如果設定 Docker 後，節點不再自動探索閘道：

1. 確認閘道目前是以自動、強制開啟或強制關閉模式執行：

   ```bash
   docker compose config | grep OPENCLAW_DISABLE_BONJOUR
   ```

2. 確認可透過發布的連接埠存取閘道本身：

   ```bash
   curl -fsS http://127.0.0.1:18789/healthz
   ```

3. 停用 Bonjour 時，請使用直接目標：
   - 控制介面或本機工具：`http://127.0.0.1:18789`
   - 區域網路用戶端：`http://<gateway-host>:18789`
   - 跨網路用戶端：Tailnet MagicDNS、Tailnet IP、SSH 通道或廣域 DNS-SD

4. 如果你特意在 Docker 中啟用 Bonjour 外掛，並使用 `OPENCLAW_DISABLE_BONJOUR=0` 強制發出廣告，請從主機測試多點傳播：

   ```bash
   dns-sd -B _openclaw-gw._tcp local.
   ```

   如果瀏覽結果為空，或閘道記錄顯示重複的 ciao 探測失敗，請還原 `OPENCLAW_DISABLE_BONJOUR=1`，並改用直接路由或 Tailnet 路由。

## 常見失敗模式

- **Bonjour 無法跨越網路**：請使用 Tailnet 或 SSH。
- **多點傳播遭封鎖**：部分 Wi-Fi 網路會停用 mDNS。
- **廣告端卡在探測／宣告階段**：多點傳播遭封鎖的主機、容器橋接網路、WSL 或介面頻繁變動，可能使回應端停留在未宣告狀態。仍可透過直接連線、SSH、Tailnet 或廣域 DNS-SD 路由使用閘道；無法使用多點傳播時，請使用 `discovery.mdns.mode: "off"` 或 `OPENCLAW_DISABLE_BONJOUR=1` 停用區域網路 Bonjour。
- **Docker 橋接網路**：在偵測到的容器中，Bonjour 會自動停用。僅針對主機、macvlan 或其他支援 mDNS 的網路設定 `OPENCLAW_DISABLE_BONJOUR=0`。
- **睡眠／介面頻繁變動**：macOS 可能暫時無法取得 mDNS 結果；請重試。
- **可瀏覽但解析失敗**：請使用簡單的機器名稱（避免表情符號或標點符號），然後重新啟動閘道。服務執行個體名稱衍生自主機名稱，因此過度複雜的名稱可能使某些解析器無法正確處理。

## 跳脫的執行個體名稱（`\032`）

Bonjour/DNS-SD 通常會將服務執行個體名稱中的位元組跳脫為十進位 `\DDD` 序列（空格會變成 `\032`）。這在通訊協定層級屬於正常現象；使用者介面應解碼後再顯示（iOS 使用 `BonjourEscapes.decode`）。

## 啟用／停用／設定

| 設定                                              | 效果                                                                            |
| ---------------------------------------------------- | --------------------------------------------------------------------------------- |
| `openclaw plugins enable bonjour`                    | 在預設未啟用的主機上啟用隨附的區域網路探索外掛。 |
| `openclaw plugins disable bonjour`                   | 透過停用隨附的外掛，停用區域網路多點傳播廣告。               |
| `OPENCLAW_DISABLE_BONJOUR=1`（或 `true`/`yes`/`on`）  | 在不變更外掛設定的情況下停用區域網路多點傳播廣告。                |
| `OPENCLAW_DISABLE_BONJOUR=0`（或 `false`/`no`/`off`） | 強制啟用區域網路多點傳播廣告，包括在偵測到的容器內。        |
| `discovery.mdns.mode`                                | `off` \| `minimal`（預設）\| `full` — 請參閱上述模式。                         |
| `gateway.bind`                                       | 控制 `~/.openclaw/openclaw.json` 中的閘道繫結模式。                    |
| `OPENCLAW_SSH_PORT`                                  | 在宣告 `sshPort` 時覆寫 SSH 連接埠（完整模式）。                  |
| `OPENCLAW_TAILNET_DNS`                               | 啟用 mDNS 完整模式時，在 TXT 中發布 MagicDNS 提示。                  |
| `OPENCLAW_CLI_PATH`                                  | 覆寫所宣告的命令列介面路徑（完整模式）。                                    |

macOS 主機預設會自動啟動隨附的區域網路探索外掛。啟用 Bonjour 外掛且未設定 `OPENCLAW_DISABLE_BONJOUR` 時，Bonjour 會在一般主機上進行廣告，並在偵測到的容器（Docker、Fly.io 機器和常見容器執行階段）內自動停用。

## 相關文件

- 探索原則與傳輸方式選擇：[探索](/zh-TW/gateway/discovery)
- 節點配對與核准：[閘道配對](/zh-TW/gateway/pairing)
