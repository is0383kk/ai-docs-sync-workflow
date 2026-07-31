---
read_when:
    - 你需要在部署前驗證由操作人員管理的 Proxy 路由設定
    - 你需要在本機擷取 OpenClaw 的傳輸流量以進行偵錯
    - 你想要檢查偵錯 Proxy 工作階段、Blob 或內建查詢預設集
summary: '`openclaw proxy` 的命令列介面參考，包括由操作員管理的 Proxy 驗證與本機偵錯 Proxy 擷取檢查器'
title: 代理伺服器
x-i18n:
    generated_at: "2026-07-26T07:47:10Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 91583f785032bfffe455a1963804108550f6fbb735ac4de1dd91d0ca5ae0df35
    source_path: cli/proxy.md
    workflow: 16
---

# `openclaw proxy`

驗證由操作人員管理的代理路由，或執行本機明確指定的偵錯代理並檢查擷取的流量。

```bash
openclaw proxy validate [--json] [--proxy-url <url>] [--proxy-ca-file <path>] [--allowed-url <url>] [--denied-url <url>] [--apns-reachable] [--apns-authority <url>] [--timeout-ms <ms>]
openclaw proxy start [--host <host>] [--port <port>]
openclaw proxy run [--host <host>] [--port <port>] -- <cmd...>
openclaw proxy coverage
openclaw proxy sessions [--limit <count>]
openclaw proxy query --preset <name> [--session <id>]
openclaw proxy blob --id <blobId>
openclaw proxy purge
```

`validate` 會對由操作人員管理的正向代理執行預檢。其餘則是用於傳輸層調查的偵錯工具：啟動本機流量擷取代理、透過該代理執行子命令、列出擷取工作階段、查詢流量模式、讀取擷取的二進位大型物件，以及清除本機擷取資料。

## 驗證

依優先順序從 `--proxy-url`、設定（`proxy.proxyUrl`）或 `OPENCLAW_PROXY_URL` 檢查實際生效的操作人員管理代理 URL。如果未啟用及設定代理，則回報設定問題；傳入 `--proxy-url` 可執行一次性預檢，而不變更設定。

受管理的代理 URL 使用 `http://` 連線至一般正向代理接聽器；若 OpenClaw 必須先對代理端點本身建立 TLS 連線，再傳送代理請求，則使用 `https://`。使用 `--proxy-ca-file` 可讓該 TLS 連線信任私有 CA。

預設會執行：

- 對 `https://example.com/` 執行一次預期**允許**的檢查（可使用 `--allowed-url` 覆寫或新增，可重複指定）
- 對暫時性的迴路 canary 執行一次預期**拒絕**的檢查（可使用 `--denied-url` 覆寫，可重複指定）

自訂 `--denied-url` 目標採取失敗時關閉策略：HTTP 回應與意義不明的傳輸失敗都會視為失敗，除非你能獨立驗證部署特定的拒絕訊號。內建的迴路 canary 是唯一會將傳輸錯誤視為封鎖證明的目標。

加入 `--apns-reachable`，還可透過代理建立 APNs HTTP/2 CONNECT 通道，並確認 APNs 沙箱會回應。此探測會傳送刻意無效的提供者權杖，因此 APNs 的 `403 InvalidProviderToken` 回應會視為成功的可連線性訊號（而非失敗）。

### 選項

| 旗標                     | 效果                                                                                                             |
| ------------------------ | ------------------------------------------------------------------------------------------------------------------ |
| `--json`                 | 輸出機器可讀的 JSON                                                                                        |
| `--proxy-url <url>`      | 驗證此 `http://`/`https://` 代理 URL，而非設定或環境變數                                              |
| `--proxy-ca-file <path>` | 信任此 PEM CA 檔案，以便對 HTTPS 代理端點進行 TLS 驗證                                             |
| `--allowed-url <url>`    | 預期可透過代理成功連線的目的地（可重複指定）                                                     |
| `--denied-url <url>`     | 預期會遭代理封鎖的目的地（可重複指定）                                                       |
| `--apns-reachable`       | 同時驗證是否可透過代理連線至 APNs 沙箱的 HTTP/2                                                     |
| `--apns-authority <url>` | 要探測的 APNs authority（預設為 `https://api.sandbox.push.apple.com`；正式環境為 `https://api.push.apple.com`） |
| `--timeout-ms <ms>`      | 每個請求的逾時時間                                                                                                |

代理設定或目的地檢查失敗時，以代碼 1 結束。

如需部署指引與拒絕語意，請參閱[網路代理](/zh-TW/security/network-proxy)。

## 偵錯代理

`start` 會啟動本機流量擷取代理，並輸出其 URL、CA 憑證路徑及擷取資料庫路徑；按 Ctrl+C 即可停止。除非設定了 `--host`，否則預設繫結至 `127.0.0.1`。

`run` 會啟動本機偵錯代理，然後套用代理環境變數，在其專屬的擷取工作階段中執行 `<cmd...>`（位於 `--` 之後）。

偵錯代理的直接上游轉送會開啟上游通訊端以供診斷。啟用 OpenClaw 受管理代理模式時，預設會停用代理請求與 CONNECT 通道的直接轉送；只有在已核准的本機診斷情境下，才設定 `OPENCLAW_DEBUG_PROXY_ALLOW_DIRECT_CONNECT_WITH_MANAGED_PROXY=1`。

`coverage` 會輸出 JSON 報告（`summary` + 各傳輸方式的 `entries`），指出哪些傳輸方式會被擷取、僅能透過代理，或尚未涵蓋。

`sessions` 會列出最近的擷取工作階段（`--limit`，預設 20）。

`query --preset <name>` 會對擷取的流量執行內建查詢，並可選擇將範圍限定於 `--session <id>`。預設查詢：

- `double-sends`
- `retry-storms`
- `cache-busting`
- `ws-duplicate-frames`
- `missing-ack`
- `error-bursts`

`blob --id <blobId>` 會輸出擷取之承載資料二進位大型物件的原始內容。

`purge` 會刪除所有擷取的流量中繼資料與二進位大型物件。擷取內容屬於本機偵錯資料；完成後請清除。

## 相關內容

- [命令列介面參考](/zh-TW/cli)
- [網路代理](/zh-TW/security/network-proxy)
- [受信任代理驗證](/zh-TW/gateway/trusted-proxy-auth)
