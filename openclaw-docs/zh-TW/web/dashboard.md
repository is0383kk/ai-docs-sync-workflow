---
read_when:
    - 變更儀表板驗證或公開模式
summary: 閘道儀表板（控制介面）的存取與驗證
title: 儀表板
x-i18n:
    generated_at: "2026-07-26T08:51:29Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: ca531ad2943dfdee1cd90a4efdc1fb69c4517780e2be52237fd558b8638e7cd0
    source_path: web/dashboard.md
    workflow: 16
---

閘道儀表板是預設由 `/` 提供服務的瀏覽器控制介面（可使用 `gateway.controlUi.basePath` 覆寫）。

快速開啟（本機閘道）：

- [http://127.0.0.1:18789/](http://127.0.0.1:18789/)（或 [http://localhost:18789/](http://localhost:18789/)）
- 使用 `gateway.tls.enabled: true` 時，請使用 `https://127.0.0.1:18789/` 和 `wss://127.0.0.1:18789` 作為 WebSocket 端點。

重要參考資料：

- [控制介面](/zh-TW/web/control-ui)：使用方式與介面功能。
- [Tailscale](/zh-TW/gateway/tailscale)：Serve/Funnel 自動化。
- [Web 介面](/zh-TW/web)：繫結模式與安全性注意事項。

系統會在 WebSocket 交握期間，透過已設定的閘道驗證路徑強制執行驗證：

- `connect.params.auth.token`
- `connect.params.auth.password`
- 當 `gateway.auth.allowTailscale: true` 時，使用 Tailscale Serve 身分識別標頭
- 當 `gateway.auth.mode: "trusted-proxy"` 時，使用受信任 Proxy 身分識別標頭

請參閱[閘道設定](/zh-TW/gateway/configuration)中的 `gateway.auth`。

<Warning>
控制介面是**管理介面**（聊天、設定、執行核准）。請勿將其公開。此介面會依目前的瀏覽器分頁及所選的閘道 URL，將儀表板 URL 權杖保留在 sessionStorage 中，並在載入後從 URL 中移除權杖。建議使用 localhost、Tailscale Serve 或 SSH 通道。
</Warning>

## 快速方式（建議）

- 完成初始設定後，命令列介面會自動開啟儀表板，並顯示不含權杖的乾淨連結。
- 隨時重新開啟：`openclaw dashboard`（複製連結、盡可能開啟瀏覽器，若為無頭環境則顯示 SSH 提示）。
- 如果剪貼簿和瀏覽器傳遞都失敗，`openclaw dashboard` 仍會顯示乾淨的 URL，並指示你將權杖（取自 `OPENCLAW_GATEWAY_TOKEN` 或 `gateway.auth.token`）附加為 URL 片段鍵 `token`；它絕不會在日誌中顯示權杖值。
- 如果介面提示輸入共用密鑰驗證，請將已設定的權杖或密碼貼到控制介面設定中。

## 驗證基礎（本機與遠端）

- **Localhost**：開啟 `http://127.0.0.1:18789/`。
- **閘道 TLS**：當 `gateway.tls.enabled: true` 時，儀表板／狀態連結會使用 `https://`，而控制介面的 WebSocket 連結會使用 `wss://`。
- **共用密鑰權杖來源**：`gateway.auth.token`（或 `OPENCLAW_GATEWAY_TOKEN`）。`openclaw dashboard` 可透過 URL 片段傳遞權杖以進行一次性啟動；控制介面會依目前的分頁及所選的閘道 URL，將權杖保留在 sessionStorage，而不是 localStorage。
- **缺少設定時的執行階段權杖**：如果啟動訊息指出已產生執行階段權杖，該權杖是暫時性的，無法透過 `openclaw config get gateway.auth.token` 取得。回送介面仍需要驗證。請執行 `openclaw doctor --generate-gateway-token`、重新啟動閘道，然後將已設定的權杖貼到控制介面設定中。
- 如果 `gateway.auth.token` 由 SecretRef 管理，為避免在 Shell 日誌、剪貼簿歷程或瀏覽器啟動引數中暴露外部管理的權杖，`openclaw dashboard` 依設計會顯示、複製並開啟不含權杖的 URL。如果目前的 Shell 無法解析該參照，它仍會顯示不含權杖的 URL，以及可據以執行的驗證設定指引。
- **共用密鑰密碼**：使用已設定的 `gateway.auth.password`（或 `OPENCLAW_GATEWAY_PASSWORD`）。儀表板不會在重新載入後保留密碼。
- **具身分識別資訊的模式**：當 `gateway.auth.allowTailscale: true` 時，Tailscale Serve 會透過身分識別標頭滿足控制介面／WebSocket 驗證；非回送且可感知身分識別的反向 Proxy 會滿足 `gateway.auth.mode: "trusted-proxy"`。兩者的 WebSocket 都不需要貼上共用密鑰。
- **非 localhost**：請使用 Tailscale Serve、非回送共用密鑰繫結、搭配 `gateway.auth.mode: "trusted-proxy"` 的非回送身分識別感知反向 Proxy，或 SSH 通道。除非你刻意執行私人輸入流量的 `gateway.auth.mode: "none"` 或受信任 Proxy HTTP 驗證，否則 HTTP API 仍會使用共用密鑰驗證。請參閱 [Web 介面](/zh-TW/web)。

## 在 Telegram 中開啟

Telegram Bot 可使用 `/dashboard`，將儀表板開啟為 Telegram Mini App。

需求：

- `gateway.tailscale.mode: "serve"` 或 `"funnel"`，以便 Telegram 取得 HTTPS Mini App URL。
- Telegram 傳送者必須是 Bot 擁有者：即 `commands.ownerAllowFrom` 中的數字 Telegram 使用者 ID，或所選帳號的有效 `channels.telegram.allowFrom`。
- 在與 Bot 的私訊中執行 `/dashboard`。在群組中叫用時，只會指示你在私訊中開啟該命令，不會包含按鈕。
- Docker 安裝：Serve/Funnel 模式要求閘道在 `tailscaled` 旁繫結回送介面，而使用已發布連接埠的橋接網路無法滿足此需求。請使用 `network_mode: host` 執行閘道容器，並將主機的 `tailscaled` Socket（`/var/run/tailscale`）及 `tailscale` 命令列介面掛載至容器中。

Mini App 會執行一次性的擁有者交接，並使用短效啟動權杖重新導向至控制介面。它不會在 URL 中暴露共用閘道權杖。

v1 不涵蓋的目標：

- 不支援 Telegram Web iframe。
- Tailscale Serve/Funnel 是唯一受支援的已發布 URL 路徑。

<a id="if-you-see-unauthorized-1008"></a>

## 如果看到 “unauthorized”／1008

- 確認閘道可連線：本機使用 `openclaw status`；遠端則建立 SSH 通道 `ssh -N -L 18789:127.0.0.1:18789 user@gateway-host`，然後開啟 `http://127.0.0.1:18789/`。
- 對於 `AUTH_TOKEN_MISMATCH`，當閘道傳回重試提示時，用戶端可使用快取的裝置權杖進行一次受信任的重試；該次重試會重複使用權杖中快取且已核准的範圍（明確的 `deviceToken`／`scopes` 呼叫端會保留其要求的範圍集合）。如果該次重試後驗證仍失敗，請手動解決權杖偏移問題。
- 對於 `AUTH_SCOPE_MISMATCH`，系統已識別裝置權杖，但它未包含所要求的範圍；請重新配對或核准新的範圍集合，而非輪替共用閘道權杖。
- 在該重試路徑之外，連線驗證的優先順序為：明確指定的共用權杖／密碼、明確指定的 `deviceToken`、已儲存的裝置權杖，最後是啟動權杖。
- 在非同步 Tailscale Serve 路徑中，系統會先將相同 `{scope, ip}` 的失敗嘗試依序處理，再由驗證失敗限制器記錄，因此第二個同時發生的錯誤重試可能已顯示 `retry later`。
- 權杖偏移修復步驟請參閱[權杖偏移復原檢查清單](/zh-TW/cli/devices#token-drift-recovery-checklist)。
- 從閘道主機取得或提供共用密鑰：
  - 權杖：`openclaw config get gateway.auth.token`
  - 密碼：解析已設定的 `gateway.auth.password` 或 `OPENCLAW_GATEWAY_PASSWORD`
  - SecretRef 管理的權杖：解析外部密鑰提供者，或在此 Shell 中匯出 `OPENCLAW_GATEWAY_TOKEN`，然後重新執行 `openclaw dashboard`
  - 因未設定共用密鑰而產生的執行階段權杖：執行 `openclaw doctor --generate-gateway-token`、重新啟動閘道，然後使用已設定的權杖
- 在儀表板設定中，將權杖或密碼貼到驗證欄位，然後連線。
- 介面語言選擇器位於 **Settings -> General -> Language**，而非 Appearance 下方。

## 相關內容

- [控制介面](/zh-TW/web/control-ui)
- [WebChat](/zh-TW/web/webchat)
