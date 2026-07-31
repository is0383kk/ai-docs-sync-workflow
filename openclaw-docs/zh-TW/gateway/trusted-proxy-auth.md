---
read_when:
    - 在身分感知代理伺服器後方執行 OpenClaw
    - 在 OpenClaw 前端設定使用 OAuth 的 Pomerium、Caddy 或 nginx
    - 修正反向代理設定中的 WebSocket 1008 未授權錯誤
    - 決定要在哪裡設定 HSTS 與其他 HTTP 強化標頭
sidebarTitle: Trusted proxy auth
summary: 將閘道驗證委派給受信任的反向代理伺服器（Pomerium、Caddy、nginx + OAuth）
title: 受信任的代理伺服器驗證
x-i18n:
    generated_at: "2026-07-26T07:43:16Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 39bf8f12b3ae95f53b21bfed12deb1c8ed8f767711955bbee52c74538052a89f
    source_path: gateway/trusted-proxy-auth.md
    workflow: 16
---

<Warning>
**安全性敏感功能。** 此模式會將驗證完全委派給你的反向代理。設定錯誤可能使你的閘道遭到未經授權的存取。啟用前請仔細閱讀此頁面。
</Warning>

## 使用時機

- 你在具備**身分感知功能的代理**（Pomerium、Caddy + OAuth、nginx + oauth2-proxy、Traefik + forward auth）後方執行 OpenClaw。
- 你的代理會處理所有驗證，並透過標頭傳遞使用者身分。
- 你位於 Kubernetes 或容器環境中，且代理是通往閘道的唯一路徑。
- 你遇到 WebSocket `1008 unauthorized` 錯誤，因為瀏覽器無法在 WS 承載資料中傳遞權杖。

## 不應使用的時機

- 你的代理不會驗證使用者（僅作為 TLS 終結器或負載平衡器）。
- 存在任何可略過代理而存取閘道的路徑（防火牆漏洞、內部網路存取）。
- 你不確定代理是否會正確移除或覆寫轉送標頭。
- 你只需要個人單一使用者存取（請改用 Tailscale Serve + 回送位址）。

## 運作方式

<Steps>
  <Step title="代理驗證使用者">
    你的反向代理會驗證使用者（OAuth、OIDC、SAML 等）。
  </Step>
  <Step title="代理加入身分標頭">
    代理會加入包含已驗證使用者身分的標頭（例如 `x-forwarded-user: nick@example.com`）。
  </Step>
  <Step title="閘道驗證可信任來源">
    OpenClaw 會檢查要求是否來自**可信任的代理 IP**（`gateway.trustedProxies`），且不是閘道本身的回送位址或本機介面位址。
  </Step>
  <Step title="閘道擷取身分">
    OpenClaw 會先讀取必要標頭，再從已設定的標頭讀取使用者身分。
  </Step>
  <Step title="授權">
    如果所有檢查皆通過，且使用者通過 `allowUsers`（若已設定），便會授權該要求。
  </Step>
</Steps>

## 設定

```json5
{
  gateway: {
    // 可信任代理驗證預設要求代理的來源 IP 不是回送位址
    bind: "lan",

    // 重要：此處僅加入你的代理 IP
    trustedProxies: ["10.0.0.1", "172.17.0.1"],

    auth: {
      mode: "trusted-proxy",
      trustedProxy: {
        // 包含已驗證使用者身分的標頭（必要）
        userHeader: "x-forwarded-user",

        // 選用：必須存在的標頭（代理驗證）
        requiredHeaders: ["x-forwarded-proto", "x-forwarded-host"],

        // 選用：限制為特定使用者（空白 = 允許所有使用者）
        allowUsers: ["nick@example.com", "admin@company.org"],

        // 選用：明確選擇啟用後，允許同一主機上的回送代理
        allowLoopback: false,

        // 選用：允許已透過代理驗證的使用者註冊新的瀏覽器裝置
        deviceAutoApprove: {
          enabled: false,
          scopes: ["operator.read", "operator.write", "operator.approvals"],
        },
      },
    },
  },
}
```

<Warning>
**執行階段規則（依評估順序）**

1. 要求的來源 IP 必須符合 `gateway.trustedProxies`（支援 CIDR），否則會遭到拒絕（`trusted_proxy_untrusted_source`）。
2. 除非已設定 `gateway.auth.trustedProxy.allowLoopback = true`，且該回送位址也位於 `trustedProxies` 中（`trusted_proxy_loopback_source`），否則會拒絕來源為回送位址（`127.0.0.1`、`::1`）的要求。此檢查會在標頭檢查前執行，因此即使必要標頭也遺失，來源為回送位址的要求仍會以此方式失敗。
3. 為防止偽造，若非回送來源符合閘道主機本身的任一本機網路介面位址，該要求會遭到拒絕（`trusted_proxy_local_interface_source`）。如果介面探索本身失敗，也會拒絕該要求（`trusted_proxy_local_interface_check_failed`）。
4. `requiredHeaders` 和 `userHeader` 必須存在且不可為空白。
5. 若 `allowUsers` 不為空，則必須包含擷取出的使用者。

**轉送標頭證據會在本機直接後援機制中覆寫回送位址的本機性判定。** 如果要求從回送位址抵達，但帶有 `Forwarded`、任一 `X-Forwarded-*` 或 `X-Real-IP` 標頭，該證據會使其無法使用本機直接密碼後援與裝置身分閘控，即使它仍會因來源為回送位址而無法通過可信任代理驗證。

`allowLoopback` 對閘道主機上本機處理程序的信任程度與反向代理相同。僅當閘道仍由防火牆阻擋直接遠端存取，且本機代理會移除或覆寫用戶端提供的身分標頭時才啟用。

未經反向代理傳輸的內部閘道用戶端應使用 `gateway.auth.password` / `OPENCLAW_GATEWAY_PASSWORD`，而非可信任代理身分標頭。非回送位址的 Control UI 部署仍需明確設定 `gateway.controlUi.allowedOrigins`。
</Warning>

### 設定參考

<ParamField path="gateway.trustedProxies" type="string[]" required>
  要信任的代理 IP 位址（或 CIDR）陣列。來自其他 IP 的要求會遭到拒絕。
</ParamField>
<ParamField path="gateway.auth.mode" type="string" required>
  必須是 `"trusted-proxy"`。
</ParamField>
<ParamField path="gateway.auth.trustedProxy.userHeader" type="string" required>
  包含已驗證使用者身分的標頭名稱。
</ParamField>
<ParamField path="gateway.auth.trustedProxy.requiredHeaders" type="string[]">
  要信任要求而必須存在的其他標頭。
</ParamField>
<ParamField path="gateway.auth.trustedProxy.allowUsers" type="string[]">
  使用者身分允許清單。空白表示允許所有已驗證的使用者。
</ParamField>
<ParamField path="gateway.auth.trustedProxy.allowLoopback" type="boolean" default="false">
  選擇啟用對同一主機上回送反向代理的支援。
</ParamField>
<ParamField path="gateway.auth.trustedProxy.deviceAutoApprove.enabled" type="boolean" default="false">
  在可信任代理驗證後，自動核准新的 Control UI 與 WebChat 裝置身分。
</ParamField>
<ParamField path="gateway.auth.trustedProxy.deviceAutoApprove.scopes" type="string[]" default='["operator.read", "operator.write", "operator.approvals"]'>
  授予自動核准瀏覽器裝置的最大範圍。明確列出 `operator.admin` 會讓每位透過代理驗證的使用者要求自動授予裝置完整管理員權限，使未指定範圍的要求自動獲得完整管理員權限，並觸發嚴重的 `gateway.trusted_proxy_device_auto_approve_admin` 安全稽核發現及閘道啟動警告。
</ParamField>

<Warning>
僅當本機反向代理是預期的信任邊界時，才啟用 `allowLoopback`。任何能連線至閘道的本機處理程序都可嘗試傳送代理身分標頭，因此請確保只有主機本身能直接存取閘道，並要求由代理擁有的標頭（例如 `x-forwarded-proto`），或在代理支援時使用已簽署的宣告標頭。
</Warning>

## 自動裝置核准

可信任代理驗證可選擇使用代理身分作為新瀏覽器裝置的核准邊界：

```json5
{
  gateway: {
    auth: {
      mode: "trusted-proxy",
      trustedProxy: {
        userHeader: "x-forwarded-user",
        allowUsers: ["operator@example.com"],
        deviceAutoApprove: {
          enabled: true,
          scopes: ["operator.read", "operator.write", "operator.approvals"],
        },
      },
    },
  },
}
```

預設值為 `enabled: false`。啟用後，以下規則全數適用：

1. WebSocket 必須已透過 `trusted-proxy` 方法完成驗證，具有非空白的使用者身分，且在設定允許清單時已通過 `allowUsers`。權杖、密碼、Tailscale 及未驗證連線絕不會使用此原則。
2. 只有新的 Control UI 或 WebChat 瀏覽器裝置能獲得自動核准。針對既有裝置的任何要求（包括提升範圍）都會維持待處理狀態，等候使用 `openclaw devices approve <requestId>` 手動核准。
3. 裝置會以 `operator` 角色獲得核准。如果連線要求包含範圍，授權內容會是要求範圍與 `deviceAutoApprove.scopes` 的精確交集。如果要求省略範圍，則會授予已設定的清單；若省略該清單，預設為 `operator.read`、`operator.write` 和 `operator.approvals`。若連線中存在 [`x-openclaw-scopes`](#control-ui-pairing-behavior) 代理標頭，最終授權還會受其限制，因此縮減使用者範圍的代理也會限制**持久性**裝置授權，而不只是工作階段；存在但為空白的標頭不會產生任何範圍。即使用戶端省略自己的範圍清單，此限制仍會套用。
4. 只有明確列於 `deviceAutoApprove.scopes` 中時，才允許 `operator.admin`。列出後，每位透過代理驗證的使用者都能為新的瀏覽器裝置要求並自動獲得完整管理員權限；未指定範圍的要求會自動獲得完整管理員權限。`openclaw security audit` 會回報嚴重的 `gateway.trusted_proxy_device_auto_approve_admin` 發現，且閘道會在啟動時記錄一次警告。在個別身分角色可用之前，請優先使用 `openclaw devices approve` 或 `openclaw devices rotate` 手動核准管理員權限。

<Warning>
啟用此選項會將新瀏覽器裝置的註冊完全委派給反向代理身分。遭入侵的代理帳號可註冊具有所有已設定範圍的持久性裝置。列出 `operator.admin` 會使該裝置在未經手動核准的情況下成為完整管理員。請確保只能透過代理存取閘道、要求強式代理驗證、覆寫身分標頭，並使用範圍受限的 `allowUsers` 清單。
</Warning>

## Control UI 配對行為

當 `gateway.auth.mode = "trusted-proxy"` 啟用且要求通過可信任代理檢查時，Control UI WebSocket 工作階段可在沒有裝置配對身分的情況下連線。

範圍影響：

- 沒有裝置的 Control UI WebSocket 工作階段可以連線，但預設不會獲得任何操作員範圍。OpenClaw 會將要求的範圍清單清除為 `[]`，因此未繫結至已核准配對裝置／權杖的工作階段無法自行宣告權限。
- 如果 WebSocket 成功連線後，方法因 `missing scope` 而失敗，請使用 HTTPS，讓瀏覽器可以產生裝置身分並完成配對。請參閱 [Control UI 不安全的 HTTP](/zh-TW/web/control-ui#insecure-http)。
- 仍包含已淘汰
  `gateway.controlUi.dangerouslyDisableDeviceAuth=true` 金鑰的舊版設定會使用受限的
  [Control UI 升級遷移](/zh-TW/web/control-ui#device-pairing-first-connection)。

反向代理範圍限制：如果你的代理在 Control UI WebSocket 升級要求中傳送 `x-openclaw-scopes`，OpenClaw 會將工作階段範圍限制為要求範圍與宣告範圍的交集。此標頭不會授予範圍，只會縮減工作階段可持有的範圍。當 `deviceAutoApprove.enabled` 為 true 時，相同限制也會套用至由[自動裝置核准](#automatic-device-approval)寫入的持久性裝置授權，因此自動核准的裝置絕不會持有超出代理所宣告的範圍。

影響：

- 配對不再是無裝置 Control UI 存取的主要閘門。當 `deviceAutoApprove.enabled` 為 true 時，代理身分也會成為新瀏覽器裝置註冊的核准閘門。
- 你的反向代理驗證原則和 `allowUsers` 會成為實際的存取控制。
- 請將閘道輸入流量限制為僅允許可信任的代理 IP（`gateway.trustedProxies` + 防火牆）。

自訂 WebSocket 用戶端並非 Control UI 工作階段。已淘汰的 Control UI
升級輸入不會向任意
`client.mode: "backend"` 或命令列介面形式的用戶端授予暫時存取權。自訂自動化應使用
裝置身分／配對、保留供本機直接使用的 `client.id: "gateway-client"`
後端輔助程式路徑，或在 HTTP 要求／回應介面更合適時使用[管理員 HTTP RPC 外掛](/zh-TW/plugins/admin-http-rpc)。

## 操作員範圍標頭

受信任代理驗證是一種**帶有身分資訊**的 HTTP 模式，因此呼叫端可選擇在 HTTP API 請求中使用 `x-openclaw-scopes` 宣告操作者範圍。

注意：WebSocket 範圍由閘道通訊協定交握與裝置身分繫結決定。在 Control UI WebSocket 升級請求中，`x-openclaw-scopes` 只會限制協商後的工作階段範圍，並不會授予範圍。請參閱 [Control UI 配對行為](#control-ui-pairing-behavior)。

範例：

- `x-openclaw-scopes: operator.read`
- `x-openclaw-scopes: operator.read,operator.write`
- `x-openclaw-scopes: operator.admin,operator.write`

行為：

- 標頭存在時，OpenClaw 會採用所宣告的範圍集合。
- 標頭存在但為空時，請求會宣告**沒有**操作者範圍。
- 標頭不存在時，一般帶有身分資訊的 HTTP API 會退回使用標準操作者預設範圍集合（`operator.admin`、`operator.read`、`operator.write`、`operator.approvals`、`operator.pairing`、`operator.talk.secrets`）。
- 閘道驗證的**外掛 HTTP 路由**預設範圍較窄：當 `x-openclaw-scopes` 不存在時，其執行階段範圍僅退回使用 `operator.write`。
- 即使受信任代理驗證成功，來自瀏覽器來源的 HTTP 請求仍必須通過 `gateway.controlUi.allowedOrigins`（或刻意啟用的 Host 標頭備援模式）。

實用規則：若要讓受信任代理請求的範圍比預設值更窄，或閘道驗證的外掛路由需要比寫入範圍更強的權限，請明確傳送 `x-openclaw-scopes`。

## TLS 終止與 HSTS

使用單一 TLS 終止點，並在該處套用 HSTS。

<Tabs>
  <Tab title="代理 TLS 終止（建議）">
    當反向代理為 `https://control.example.com` 處理 HTTPS 時，請在代理上為該網域設定 `Strict-Transport-Security`。

    - 適合面向網際網路的部署。
    - 將憑證與 HTTP 強化原則集中在同一處。
    - OpenClaw 可在代理後方繼續使用迴路 HTTP。

    標頭值範例：

    ```text
    Strict-Transport-Security: max-age=31536000; includeSubDomains
    ```

  </Tab>
  <Tab title="閘道 TLS 終止">
    如果 OpenClaw 本身直接提供 HTTPS（沒有終止 TLS 的代理），請設定：

    ```json5
    {
      gateway: {
        tls: { enabled: true },
        http: {
          securityHeaders: {
            strictTransportSecurity: "max-age=31536000; includeSubDomains",
          },
        },
      },
    }
    ```

    `strictTransportSecurity` 接受字串標頭值，或使用 `false` 明確停用。

  </Tab>
</Tabs>

### 推出指引

- 驗證流量時，先從較短的最大存留時間開始（例如 `max-age=300`）。
- 只有在充分確認後，才提高為長期值（例如 `max-age=31536000`）。
- 只有在每個子網域都已準備好使用 HTTPS 時，才新增 `includeSubDomains`。
- 只有在你刻意符合完整網域集合的預載要求時，才使用預載。
- 僅限迴路的本機開發無法從 HSTS 獲益。

## 代理設定範例

<AccordionGroup>
  <Accordion title="Pomerium">
    Pomerium 會在 `x-pomerium-claim-email`（或其他宣告標頭）中傳遞身分，並在 `x-pomerium-jwt-assertion` 中傳遞 JWT。

    ```json5
    {
      gateway: {
        bind: "lan",
        trustedProxies: ["10.0.0.1"], // Pomerium 的 IP
        auth: {
          mode: "trusted-proxy",
          trustedProxy: {
            userHeader: "x-pomerium-claim-email",
            requiredHeaders: ["x-pomerium-jwt-assertion"],
          },
        },
      },
    }
    ```

    Pomerium 設定片段：

    ```yaml
    routes:
      - from: https://openclaw.example.com
        to: http://openclaw-gateway:18789
        policy:
          - allow:
              or:
                - email:
                    is: nick@example.com
        pass_identity_headers: true
    ```

  </Accordion>
  <Accordion title="搭配 OAuth 的 Caddy">
    搭配 `caddy-security` 外掛的 Caddy 可以驗證使用者並傳遞身分標頭。

    ```json5
    {
      gateway: {
        bind: "lan",
        trustedProxies: ["10.0.0.1"], // Caddy/附屬代理 IP
        auth: {
          mode: "trusted-proxy",
          trustedProxy: {
            userHeader: "x-forwarded-user",
          },
        },
      },
    }
    ```

    Caddyfile 片段：

    ```caddy
    openclaw.example.com {
        authenticate with oauth2_provider
        authorize with policy1

        reverse_proxy openclaw:18789 {
            header_up X-Forwarded-User {http.auth.user.email}
        }
    }
    ```

  </Accordion>
  <Accordion title="nginx + oauth2-proxy">
    oauth2-proxy 會驗證使用者，並在 `x-auth-request-email` 中傳遞身分。

    ```json5
    {
      gateway: {
        bind: "lan",
        trustedProxies: ["10.0.0.1"], // nginx/oauth2-proxy IP
        auth: {
          mode: "trusted-proxy",
          trustedProxy: {
            userHeader: "x-auth-request-email",
          },
        },
      },
    }
    ```

    nginx 設定片段：

    ```nginx
    location / {
        auth_request /oauth2/auth;
        auth_request_set $user $upstream_http_x_auth_request_email;

        proxy_pass http://openclaw:18789;
        proxy_set_header X-Auth-Request-Email $user;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
    }
    ```

  </Accordion>
  <Accordion title="搭配轉送驗證的 Traefik">
    ```json5
    {
      gateway: {
        bind: "lan",
        trustedProxies: ["172.17.0.1"], // Traefik 容器 IP
        auth: {
          mode: "trusted-proxy",
          trustedProxy: {
            userHeader: "x-forwarded-user",
          },
        },
      },
    }
    ```
  </Accordion>
</AccordionGroup>

## 混合權杖設定

如果同時設定了共用權杖（`gateway.auth.token` 或 `OPENCLAW_GATEWAY_TOKEN`），閘道啟動時會拒絕受信任代理驗證。兩者互斥，因為共用權杖會讓同一主機上的呼叫端，透過與此模式原本要強制執行的代理驗證身分完全不同的路徑進行驗證。

如果啟動失敗，並顯示類似 `gateway auth mode is trusted-proxy, but a shared token is also configured` 的錯誤：

- 使用受信任代理模式時移除共用權杖，或
- 如果你想使用權杖式驗證，請將 `gateway.auth.mode` 切換為 `"token"`。

迴路受信任代理身分標頭仍會採取封閉式失敗：同一主機上的呼叫端不會被默默驗證為代理使用者。繞過代理的 OpenClaw 內部呼叫端可改用 `gateway.auth.password` / `OPENCLAW_GATEWAY_PASSWORD` 進行驗證。受信任代理模式刻意不支援權杖備援。

## 安全性檢查清單

啟用受信任代理驗證前，請確認：

- [ ] **代理是唯一路徑**：閘道連接埠的防火牆只允許你的代理存取。
- [ ] **trustedProxies 維持最小範圍**：僅包含實際代理 IP，而非整個子網路。
- [ ] **迴路代理來源是刻意設定的**：除非為同一主機代理明確啟用 `gateway.auth.trustedProxy.allowLoopback`，否則來自迴路來源的請求會使受信任代理驗證採取封閉式失敗。
- [ ] **代理會移除標頭**：你的代理會覆寫（而非附加）來自用戶端的 `x-forwarded-*` 標頭。
- [ ] **TLS 終止**：你的代理處理 TLS；使用者透過 HTTPS 連線。
- [ ] **明確設定 allowedOrigins**：非迴路的 Control UI 使用明確的 `gateway.controlUi.allowedOrigins`。
- [ ] **已設定 allowUsers**（建議）：限制為已知使用者，而非允許任何通過驗證的人。
- [ ] **沒有混合權杖設定**：不要同時設定 `gateway.auth.token` 與 `gateway.auth.mode: "trusted-proxy"`。
- [ ] **本機密碼備援保持私密**：如果為內部直接呼叫端設定 `gateway.auth.password`，請以防火牆保護閘道連接埠，避免非代理遠端用戶端直接存取。
- [ ] **裝置自動核准是刻意啟用的**：如果 `deviceAutoApprove.enabled` 為 true，請將反向代理帳號安全性視為裝置註冊邊界，並將授予的範圍清單維持為非管理員且最小化。

## 安全性稽核

`openclaw security audit` 會以**重大**嚴重性發現標記受信任代理驗證。這是刻意設計的；用來提醒你已將安全性委派給代理設定。

稽核會檢查：

- 基本 `gateway.trusted_proxy_auth` 警告／重大提醒。
- 缺少 `trustedProxies` 設定。
- 缺少 `userHeader` 設定。
- `allowUsers` 為空（允許任何通過驗證的使用者）。
- 為同一主機代理來源啟用了 `allowLoopback`。
- 啟用了瀏覽器裝置自動核准（將新裝置配對委派給代理身分）。

只要 Control UI 對外公開，其他不專屬於受信任代理的發現也同樣適用：萬用字元或缺少 `gateway.controlUi.allowedOrigins`，以及 Host 標頭來源備援。

## 疑難排解

<AccordionGroup>
  <Accordion title="trusted_proxy_untrusted_source">
    請求並非來自 `gateway.trustedProxies` 中的 IP。請檢查：

    - 代理 IP 是否正確？（Docker 容器 IP 可能會變更。）
    - 代理前方是否有負載平衡器？
    - 使用 `docker inspect` 或 `kubectl get pods -o wide` 找出實際 IP。

  </Accordion>
  <Accordion title="trusted_proxy_loopback_source">
    OpenClaw 拒絕了來自迴路來源的受信任代理請求。

    請檢查：

    - 代理是否從 `127.0.0.1` / `::1` 連線？
    - 你是否嘗試搭配同一主機的迴路反向代理使用受信任代理驗證？

    修正方式：

    - 對於未經代理的同一主機內部用戶端，優先使用權杖／密碼驗證，或
    - 透過非迴路的受信任代理位址路由，並將該 IP 保留在 `gateway.trustedProxies` 中，或
    - 若是刻意設定的同一主機反向代理，請設定 `gateway.auth.trustedProxy.allowLoopback = true`、將迴路位址保留在 `gateway.trustedProxies` 中，並確保代理會移除或覆寫身分標頭。

  </Accordion>
  <Accordion title="trusted_proxy_local_interface_source / trusted_proxy_local_interface_check_failed">
    請求的來源 IP 符合閘道主機自身的其中一個非迴路網路介面位址（而非代理）；這是一項防護措施，用來阻止 tailnet 或 Docker 橋接網路上偽造的同一主機流量。`..._check_failed` 表示介面探索本身發生錯誤，因此 OpenClaw 會採取封閉式失敗。

    請檢查：

    - 閘道主機本身是否有處理程序繞過代理，直接傳送身分標頭？
    - 代理是否與閘道在相同的網路命名空間中執行，而且其 IP 也顯示為本機介面？

    修正方式：透過未同時繫結至閘道主機本機的位址路由代理流量，或僅在真正的同一主機代理設定中使用 `allowLoopback`。

  </Accordion>
  <Accordion title="trusted_proxy_user_missing">
    使用者標頭為空或不存在。請檢查：

    - 你的代理是否已設定為傳遞身分標頭？
    - 標頭名稱是否正確？（不區分大小寫，但拼字必須正確）
    - 使用者是否確實已在代理完成驗證？

  </Accordion>
  <Accordion title="trusted_proxy_missing_header_*">
    必要標頭不存在。請檢查：

    - 代理中這些特定標頭的設定。
    - 標頭是否在鏈路中的某處遭到移除。

  </Accordion>
  <Accordion title="trusted_proxy_user_not_allowed">
    使用者已通過驗證，但不在 `allowUsers` 中。請將其加入，或移除允許清單。
  </Accordion>
  <Accordion title="trusted_proxy_no_proxies_configured / trusted_proxy_config_missing">
    `gateway.auth.mode` 為 `"trusted-proxy"`，但 `gateway.trustedProxies` 是空的，或缺少 `gateway.auth.trustedProxy` 本身。在兩者都設定完成前，所有要求都會遭到拒絕。
  </Accordion>
  <Accordion title="trusted_proxy_origin_not_allowed">
    受信任 Proxy 驗證成功，但瀏覽器的 `Origin` 標頭未通過 Control UI 的來源檢查。

    請檢查：

    - `gateway.controlUi.allowedOrigins` 是否包含確切的瀏覽器來源。
    - 除非你有意允許所有來源，否則不應依賴萬用字元來源。
    - 如果你有意使用 Host 標頭備援模式，請確認 `gateway.controlUi.dangerouslyAllowHostHeaderOriginFallback=true` 是刻意設定的。

  </Accordion>
  <Accordion title="連線成功，但方法回報缺少範圍">
    WebSocket 已連線，但 `chat.history`、`sessions.list` 或
    `models.list` 因 `missing scope: operator.read` 而失敗。

    常見原因：

    - 沒有裝置的 Control UI 工作階段：受信任 Proxy 驗證可以在沒有裝置身分的情況下允許 WebSocket 連線，但 OpenClaw 依設計會清除沒有裝置之工作階段的範圍。
    - 自訂後端用戶端：已淘汰的 Control UI 升級輸入絕不會授予任意後端或命令列介面形式的 WebSocket 用戶端存取權。
    - 過度限縮的 `x-openclaw-scopes`：如果你的 Proxy 在 Control UI 的 WebSocket 升級要求中注入此標頭，工作階段範圍會以該集合為上限。空白標頭值不會產生任何範圍。

    修正方式：

    - 針對 Control UI，請使用 HTTPS，讓瀏覽器能產生裝置身分並完成配對。
    - 針對自訂自動化，請使用裝置身分／配對、保留的直接本機 `gateway-client` 後端輔助程式路徑，或[管理員 HTTP RPC](/zh-TW/plugins/admin-http-rpc)。
    - 請勿將已淘汰的 `gateway.controlUi.dangerouslyDisableDeviceAuth` 鍵加入目前的設定。較舊的安裝會自動使用一次性的自我配對遷移。

  </Accordion>
  <Accordion title="WebSocket 仍然失敗">
    請確認你的 Proxy：

    - 支援 WebSocket 升級（`Upgrade: websocket`、`Connection: upgrade`）。
    - 在 WebSocket 升級要求中傳遞身分標頭（不僅限於 HTTP）。
    - 未對 WebSocket 連線使用個別的驗證路徑。

  </Accordion>
</AccordionGroup>

## 從權杖驗證遷移

<Steps>
  <Step title="設定 Proxy">
    設定你的 Proxy，以驗證使用者並傳遞標頭。
  </Step>
  <Step title="獨立測試 Proxy">
    獨立測試 Proxy 設定（使用帶有標頭的 curl）。
  </Step>
  <Step title="更新 OpenClaw 設定">
    使用受信任 Proxy 驗證更新 OpenClaw 設定。
  </Step>
  <Step title="重新啟動閘道">
    重新啟動閘道。
  </Step>
  <Step title="測試 WebSocket">
    從 Control UI 測試 WebSocket 連線。
  </Step>
  <Step title="稽核">
    執行 `openclaw security audit` 並檢視發現的問題。
  </Step>
</Steps>

## 相關內容

- [設定](/zh-TW/gateway/configuration) — 設定參考
- [操作員範圍](/zh-TW/gateway/operator-scopes) — 角色、範圍與核准檢查
- [遠端存取](/zh-TW/gateway/remote) — 其他遠端存取模式
- [安全性](/zh-TW/gateway/security) — 完整的安全性指南
- [Tailscale](/zh-TW/gateway/tailscale) — 僅限 tailnet 存取的較簡單替代方案
