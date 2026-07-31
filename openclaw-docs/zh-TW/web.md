---
read_when:
    - 你想透過 Tailscale 存取閘道
    - 你需要瀏覽器控制介面與設定編輯功能
summary: 閘道網頁介面：控制介面、繫結模式與安全性
title: 網頁
x-i18n:
    generated_at: "2026-07-26T08:04:35Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 413fb029d95241f5c6043b28825727cdee52b2fa8cbe998fbbd6e3ff7b81467b
    source_path: web/index.md
    workflow: 16
---

閘道會從與閘道 WebSocket 相同的連接埠提供小型的**瀏覽器控制介面**（Vite + Lit）：

- 預設：`http://<host>:18789/`
- 搭配 `gateway.tls.enabled: true`：`https://<host>:18789/`
- 選用前綴：設定 `gateway.controlUi.basePath`（例如 `/openclaw`）

功能詳見[控制介面](/zh-TW/web/control-ui)。本頁涵蓋繫結模式、安全性及其他面向 Web 的介面。

## 設定（預設開啟）

資產存在時（`dist/control-ui`），控制介面會**預設啟用**：

```json5
{
  gateway: {
    controlUi: { enabled: true, basePath: "/openclaw" }, // basePath 選用
  },
}
```

## 網路鉤子

設定 `hooks.enabled=true` 後，閘道也會在同一個 HTTP 伺服器上公開網路鉤子端點。認證與承載內容請參閱[閘道設定參考](/zh-TW/gateway/configuration-reference#hooks)中的 `hooks`。

## 管理 HTTP RPC

`POST /api/v1/admin/rpc` 會透過 HTTP 公開所選的閘道控制平面方法。預設關閉；只有啟用 `admin-http-rpc` 外掛時才會註冊。認證模型、允許的方法，以及與 WebSocket API 的比較，請參閱[管理 HTTP RPC](/zh-TW/plugins/admin-http-rpc)。

## Tailscale 存取

<Tabs>
  <Tab title="整合式 Serve（建議）">
    將閘道維持在回送介面，並讓 Tailscale Serve 代理該閘道：

    ```json5
    {
      gateway: {
        bind: "loopback",
        tailscale: { mode: "serve" },
      },
    }
    ```

    啟動閘道：

    ```bash
    openclaw gateway
    ```

    開啟 `https://<magicdns>/`（或你設定的 `gateway.controlUi.basePath`）。

  </Tab>
  <Tab title="Tailnet 繫結 + 權杖">
    ```json5
    {
      gateway: {
        bind: "tailnet",
        controlUi: { enabled: true },
        auth: { mode: "token", token: "your-token" },
      },
    }
    ```

    啟動閘道（此非回送介面範例使用共用密鑰權杖認證）：

    ```bash
    openclaw gateway
    ```

    開啟 `http://<tailscale-ip>:18789/`（或你設定的 `gateway.controlUi.basePath`）。

  </Tab>
  <Tab title="公用網際網路（Funnel）">
    ```json5
    {
      gateway: {
        bind: "loopback",
        tailscale: { mode: "funnel" },
        auth: { mode: "password" }, // 或 OPENCLAW_GATEWAY_PASSWORD
      },
    }
    ```

    `tailscale.mode: "funnel"` 需要 `gateway.auth.mode: "password"`；Serve 與 Funnel 都需要 `gateway.bind: "loopback"`。

  </Tab>
</Tabs>

## 安全性注意事項

- 預設需要閘道認證：權杖、密碼、受信任的代理，或啟用時的 Tailscale Serve 身分識別標頭。
- 非回送介面繫結仍然**需要**閘道認證：權杖／密碼認證，或搭配 `gateway.auth.mode: "trusted-proxy"` 的身分識別感知反向代理。
- 新手引導精靈預設會建立共用密鑰認證，而且通常會產生閘道權杖，即使在回送介面上也是如此。
- 在共用密鑰模式下，介面會在 WebSocket 交握期間傳送 `connect.params.auth.token` 或 `connect.params.auth.password`。
- 使用 `gateway.tls.enabled: true` 時，本機儀表板／狀態輔助工具會呈現 `https://` URL 與 `wss://` WebSocket URL。
- 在帶有身分識別資訊的模式下（Tailscale Serve、`trusted-proxy`），WebSocket 認證檢查會從要求標頭獲得滿足，而不是使用共用密鑰。
- 若將控制介面部署在公用的非回送介面上，請明確設定 `gateway.controlUi.allowedOrigins`（完整來源）。對於回送介面、RFC1918／連結本機、`.local`、`.ts.net` 和 Tailscale CGNAT 主機，無須設定即可接受私有同源載入。
- `gateway.controlUi.dangerouslyAllowHostHeaderOriginFallback: true` 會啟用 Host 標頭來源後援；這是危險的安全性降級。
- 使用 Serve 時，若 `gateway.auth.allowTailscale: true`，Tailscale 身分識別標頭會滿足控制介面／WebSocket 認證要求（不需要權杖／密碼）。HTTP API 端點不使用 Tailscale 身分識別標頭；它們一律遵循閘道的一般 HTTP 認證模式。設定 `gateway.auth.allowTailscale: false`，即可要求即使透過 Serve 也必須提供明確認證資訊。此無權杖流程假設閘道主機本身受到信任。請參閱 [Tailscale](/zh-TW/gateway/tailscale)與[安全性](/zh-TW/gateway/security)。

## 建置介面

閘道會從 `dist/control-ui` 提供靜態檔案：

```bash
pnpm ui:build
```
