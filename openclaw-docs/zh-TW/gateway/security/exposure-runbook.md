---
read_when:
    - 透過區域網路、tailnet、Tailscale Serve、Funnel 或反向代理公開閘道
    - 在允許真實訊息使用者使用前審查部署作業
    - 回復有風險的遠端存取或私訊設定
sidebarTitle: Exposure runbook
summary: 將 OpenClaw 閘道公開至迴路介面以外之前的預檢與復原檢查清單
title: 閘道公開存取操作手冊
x-i18n:
    generated_at: "2026-07-26T07:20:55Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: fb8e66af57e804325afc91281122b822183337177c734efe065c5fc18b175e72
    source_path: gateway/security/exposure-runbook.md
    workflow: 16
---

<Warning>
只有在你能說明誰可以連線、他們如何通過驗證、可以觸發哪些代理程式，以及這些代理程式可以使用哪些工具之後，才可公開閘道。若有疑問，請恢復為僅限回送存取，並重新執行稽核。
</Warning>

本操作手冊將較廣泛的[安全性](/zh-TW/gateway/security)指引轉化為遠端存取與訊息暴露的管理員檢查清單。

## 選擇暴露模式

優先選擇能滿足工作流程的最小範圍模式。

| 模式                       | 建議使用時機                                    | 必要控制措施                                                                                                                    |
| -------------------------- | ----------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------- |
| 回送 + SSH 通道            | 個人使用、管理員存取、偵錯                      | 保留 `gateway.bind: "loopback"` 並建立通道 `127.0.0.1:18789`                                                                    |
| 回送 + Tailscale Serve     | 從個人 tailnet 存取控制介面/WebSocket           | 閘道維持僅限回送；Tailscale 身分標頭只驗證控制介面 WebSocket 介面，不驗證其他身分驗證路徑 |
| Tailnet/LAN 繫結           | 具有已知裝置的專用私人網路                      | 閘道身分驗證、防火牆允許清單、不可公開轉送連接埠                                                                                |
| 受信任的反向 Proxy         | 在閘道前方使用組織 SSO/OIDC                     | `trusted-proxy` 身分驗證、嚴格的 `trustedProxies`、標頭覆寫/移除規則、明確允許的使用者                             |
| 公開網際網路               | 少見的高風險部署                                | 可感知身分的 Proxy、TLS、速率限制、嚴格允許清單、在沙箱中執行的非主要工作階段                                                |

避免將公開連接埠直接轉送至閘道。如果需要公開存取，請在閘道前方放置可感知身分的 Proxy，並將其設為通往閘道的唯一網路路徑。

## 執行前盤點

變更繫結、Proxy、Tailscale 或頻道原則之前，請記錄以下項目：

- 閘道主機、作業系統使用者和狀態目錄（預設為 `~/.openclaw`）。
- 閘道 URL 和繫結模式（`gateway.bind`；預設連接埠為 `18789`）。
- 身分驗證模式、權杖/密碼來源，或受信任的 Proxy 身分來源。
- 每個已啟用的頻道，以及其是否接受私人訊息、群組或網路鉤子。
- 非本機傳送者可存取的代理程式。
- 每個可存取代理程式的工具設定檔、沙箱模式和提高權限工具原則。
- 這些代理程式可用的外部認證資訊。
- `~/.openclaw/openclaw.json` 和認證資訊的備份位置。

如果有多人可以傳送訊息給機器人，請將此情況視為共用的委派工具權限，而不是個別使用者的主機隔離。

## 基準檢查

請在開放存取前執行：

```bash
openclaw doctor
openclaw security audit
openclaw security audit --deep
openclaw health
```

請先解決重大發現。只有當警告是部署時刻意接受且已有文件記錄時，才能接受警告。請參閱[安全性稽核檢查](/zh-TW/gateway/security/audit-checks)，瞭解每個 `checkId` 的含義及其修正鍵。

若要進行遠端命令列介面驗證，請明確傳入認證資訊：

```bash
openclaw gateway probe --url ws://127.0.0.1:18789 --token "$OPENCLAW_GATEWAY_TOKEN"
```

不要假設本機設定的認證資訊適用於明確指定的遠端 URL。

## 最低安全基準

請將此設定形式作為暴露式部署的起點：

```json5
{
  gateway: {
    bind: "loopback",
    auth: {
      mode: "token",
      token: "replace-with-a-long-random-token",
    },
  },
  session: {
    dmScope: "per-channel-peer",
  },
  agents: {
    defaults: {
      sandbox: { mode: "non-main" },
    },
  },
  tools: {
    profile: "messaging",
    exec: { security: "deny", ask: "always" },
    elevated: { enabled: false },
  },
}
```

每次只放寬一項控制：先新增特定頻道的允許清單，再啟用可寫入的工具；或先啟用反向 Proxy，再接受遠端控制介面流量。

`tools.exec.security: "deny"` 會封鎖所有 exec 呼叫，包括無害的診斷。如果需要診斷或低風險命令，只有在選定符合威脅模型的特定傳送者、代理程式、命令和核准模式後，才可放寬此設定。

## 私人訊息與群組暴露

訊息頻道是不受信任的輸入介面。允許私人訊息或群組之前：

- 優先使用 `dmPolicy: "pairing"` 或嚴格的 `allowFrom` 清單，而非 `dmPolicy: "open"`。
- 不要將 `"*"` 允許清單與廣泛的工具存取權限搭配使用。
- 除非聊天室受到嚴格控管，否則群組中必須提及機器人。
- 當多人可以向機器人傳送私人訊息時，請設定 `session.dmScope: "per-channel-peer"`（多帳號頻道則設定 `"per-account-channel-peer"`），以免私人訊息工作階段共用情境。
- 將共用頻道路由至只具備最少工具且無個人認證資訊的代理程式。

配對會核准傳送者觸發機器人，但不會使該傳送者成為獨立的主機安全邊界。

## 反向 Proxy 檢查

對於可感知身分的 Proxy：

- Proxy 必須先驗證使用者身分，才能轉送至閘道。
- 防火牆或網路原則必須封鎖對閘道連接埠的直接存取。
- `gateway.trustedProxies` 必須只列出 Proxy 來源 IP。
- Proxy 必須移除或覆寫用戶端提供的身分與轉送標頭。
- 當 Proxy 服務多個受眾時，請設定 `gateway.auth.trustedProxy.allowUsers`。
- 只有在 Proxy 位於同一主機、信任本機程序，且 Proxy 擁有身分標頭時，才可使用 `gateway.auth.trustedProxy.allowLoopback`。

變更 Proxy 後，請執行 `openclaw security audit --deep`。受信任 Proxy 相關發現具有高度指標性，因為 Proxy 會成為身分驗證邊界。

## 工具與沙箱審查

向遠端傳送者公開代理程式之前：

- 確認哪些工作階段在主機上執行，哪些在沙箱中執行。
- 拒絕主機 exec，或要求核准。
- 除非特定受信任的傳送者需要提高權限的工具，否則請將其保持停用。
- 對於開放或半開放的訊息介面，請避免使用瀏覽器、畫布、節點、排程、閘道和工作階段建立工具。
- 保持繫結掛載範圍狹窄；避免掛載認證資訊、家目錄、Docker 通訊端和系統路徑。
- 對於信任程度有實質差異的邊界，請使用不同的閘道、作業系統使用者或主機。

如果並不完全信任遠端使用者，隔離必須透過不同的部署實現，而不能只依賴提示詞或工作階段標籤。

## 變更後驗證

每次變更暴露方式後：

1. 重新執行 `openclaw security audit --deep`。
2. 確認已授權的連線能成功建立。
3. 確認未授權的傳送者或瀏覽器工作階段會遭到拒絕。
4. 確認記錄會遮蔽祕密。
5. 確認私人訊息/群組路由只會到達預定的代理程式。
6. 確認高影響力工具會要求核准或遭到拒絕。
7. 記錄已接受的殘餘警告。

在瞭解目前變更之前，請勿繼續下一項暴露變更。

## 復原計畫

如果閘道可能過度暴露：

```json5
{
  gateway: {
    bind: "loopback",
  },
  channels: {
    whatsapp: { dmPolicy: "disabled" },
    telegram: { dmPolicy: "disabled" },
    discord: { dmPolicy: "disabled" },
    slack: { dmPolicy: "disabled" },
  },
  tools: {
    exec: { security: "deny", ask: "always" },
    elevated: { enabled: false },
  },
}
```

接著：

1. 停止公開轉送、Tailscale Funnel 或反向 Proxy 路由。
2. 輪替閘道權杖/密碼，以及受影響整合的認證資訊。
3. 從允許清單中移除 `"*"` 和非預期的傳送者。
4. 檢查近期的稽核記錄、執行歷程、工具呼叫和設定變更。
5. 重新執行 `openclaw security audit --deep`。
6. 以能滿足工作流程的最小範圍模式重新啟用存取。

## 審查檢查清單

- 除非有已記錄的理由，否則閘道維持僅限回送。
- 非回送存取具有身分驗證和防火牆保護，且沒有直接公開路徑。
- 受信任 Proxy 部署具有嚴格的 Proxy IP 和標頭控制。
- 私人訊息使用配對或允許清單，預設不開放存取。
- 群組需要提及機器人或使用明確的允許清單。
- 共用頻道無法存取個人認證資訊。
- 非主要工作階段以沙箱模式執行。
- 主機 exec 和提高權限的工具遭到拒絕或受到核准管控。
- 記錄會遮蔽祕密。
- 重大稽核發現均已解決。
- 復原步驟已經過測試並記錄成文件。
