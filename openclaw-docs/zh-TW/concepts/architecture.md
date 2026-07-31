---
read_when:
    - 處理閘道通訊協定、用戶端或傳輸層相關工作
summary: WebSocket 閘道架構、元件與用戶端流程
title: 閘道架構
x-i18n:
    generated_at: "2026-07-26T07:16:14Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: f8054bd87f738b957c24f8d6965d55365de2293d44902530a9ba778afa597cc7
    source_path: concepts/architecture.md
    workflow: 16
---

## 概觀

- 單一長期執行的**閘道**負責所有訊息介面（透過 Baileys 的 WhatsApp、透過 grammY 的 Telegram、Slack、Discord、Signal、iMessage、WebChat）。
- 控制平面用戶端（macOS 應用程式、命令列介面、Web UI、自動化）透過設定的繫結主機上的 **WebSocket** 連線至閘道（預設為 `127.0.0.1:18789`）。
- **節點**（macOS/iOS/Android/無介面）也透過 **WebSocket** 連線，但會宣告 `role: node`，並明確列出功能與命令。
- 每台主機僅有一個閘道；它是唯一會開啟 WhatsApp 工作階段的位置。
- **畫布主機**由閘道 HTTP 伺服器透過以下路徑提供：
  - `/__openclaw__/canvas/`（代理程式可編輯的 HTML/CSS/JS）
  - `/__openclaw__/a2ui/`（A2UI 主機）

  它與閘道使用相同的連接埠（預設為 `18789`）。

## 元件與流程

### 閘道（常駐程式）

- 維護提供者連線。
- 公開具型別的 WS API（請求、回應、伺服器推送事件）。
- 依據 JSON Schema 驗證傳入的訊框。
- 發出 `agent`、`chat`、`presence`、`health`、`heartbeat`、`cron` 等事件。

### 用戶端（Mac 應用程式／命令列介面／Web 管理介面）

- 每個用戶端各有一個 WS 連線。
- 傳送請求（`health`、`status`、`send`、`agent`、`system-presence`）。
- 訂閱事件（`tick`、`agent`、`presence`、`shutdown`）。

### 節點（macOS／iOS／Android／無介面）

- 使用 `role: node` 連線至**同一個 WS 伺服器**。
- 在 `connect` 中提供裝置身分；配對**以裝置為基礎**（角色 `node`），核准資訊儲存在裝置配對儲存區中。
- 公開 `canvas.*`、`camera.*`、`screen.record`、`location.get` 等命令。

通訊協定詳細資訊：[閘道通訊協定](/zh-TW/gateway/protocol)

### WebChat

- 使用閘道 WS API 取得聊天記錄及傳送訊息的靜態 UI。
- 在遠端設定中，透過與其他用戶端相同的 SSH/Tailscale 通道連線。

## 連線生命週期（單一用戶端）

```mermaid
sequenceDiagram
    participant Client
    participant Gateway

    Client->>Gateway: req:connect
    Gateway-->>Client: res (成功)
    Note right of Gateway: 或回應錯誤並關閉
    Note left of Client: payload=hello-ok<br>快照：目前狀態 + 健康狀態

    Gateway-->>Client: event:presence
    Gateway-->>Client: event:tick

    Client->>Gateway: req:agent
    Gateway-->>Client: res:agent<br>確認 {runId, status:"accepted"}
    Gateway-->>Client: event:agent<br>（串流中）
    Gateway-->>Client: res:agent<br>最終結果 {runId, status, summary}
```

## 線路通訊協定（摘要）

- 傳輸：WebSocket，包含 JSON 承載資料的文字訊框。
- 第一個訊框**必須**是 `connect`。
- 交握後：
  - 請求：`{type:"req", id, method, params}` → `{type:"res", id, ok, payload|error}`
  - 事件：`{type:"event", event, payload, seq?, stateVersion?}`
- `hello-ok.features.methods`／`events` 是探索中繼資料，而不是每個可呼叫輔助路由的自動產生傾印。
- 共用密鑰驗證會依設定的閘道驗證模式使用 `connect.params.auth.token` 或 `connect.params.auth.password`。
- 具有身分資訊的模式，例如 Tailscale Serve（`gateway.auth.allowTailscale: true`）或非回送 `gateway.auth.mode: "trusted-proxy"`，會從請求標頭完成驗證，而非使用 `connect.params.auth.*`。
- 私有輸入的 `gateway.auth.mode: "none"` 會完全停用共用密鑰驗證；請勿在公開／不受信任的輸入上啟用該模式。
- 具有副作用的方法（`send`、`agent`）必須使用冪等性金鑰，才能安全地重試；伺服器會保留短期的去重快取。
- 節點必須在 `connect` 中包含 `role: "node"`，以及功能、命令與權限。

## 配對與本機信任

- 所有 WS 用戶端（操作端 + 節點）都會在 `connect` 中包含**裝置身分**。
- 新的裝置 ID 需要取得配對核准；閘道會簽發**裝置權杖**，供後續連線使用。
- 直接的本機回送連線可自動核准，以維持同一主機上的流暢使用體驗。
- OpenClaw 也針對受信任的共用密鑰輔助流程，提供範圍有限的後端／容器本機自我連線路徑。
- Tailnet 與 LAN 連線（包括同一主機的 Tailnet 繫結）仍需要明確的配對核准。
- 所有連線都必須簽署 `connect.challenge` nonce。簽章承載資料 `v3` 也會繫結 `platform` 與 `deviceFamily`；閘道會在重新連線時固定已配對的中繼資料，若中繼資料變更，則需要修復配對。
- **非本機**連線仍需要明確核准。
- 閘道驗證（`gateway.auth.*`）仍適用於**所有**連線，無論是本機或遠端。

詳細資訊：[閘道通訊協定](/zh-TW/gateway/protocol)、[配對](/zh-TW/channels/pairing)、
[安全性](/zh-TW/gateway/security)。

## 通訊協定型別與程式碼產生

- TypeBox 結構描述定義通訊協定。
- JSON Schema 由這些結構描述產生。
- Swift 模型由 JSON Schema 產生。

## 遠端存取

- 建議使用：Tailscale 或 VPN。
- 替代方案：SSH 通道

  ```bash
  ssh -N -L 18789:127.0.0.1:18789 user@gateway-host
  ```

- 透過通道時，仍適用相同的交握與驗證權杖。
- 遠端設定可為 WS 啟用 TLS 與選用的固定驗證。

## 操作概況

- 啟動：`openclaw gateway`（前景執行，記錄至標準輸出）。
- 健康狀態：透過 WS 使用 `health`（也包含在 `hello-ok` 中）。
- 監督：使用 launchd/systemd 自動重新啟動。

## 不變條件

- 每台主機只能由一個閘道控制單一 Baileys 工作階段。
- 交握為強制要求；任何非 JSON 或第一個訊框不是 connect 的情況，都會立即強制關閉連線。
- 事件不會重播；若有缺漏，用戶端必須重新整理。

## 相關內容

- [代理程式迴圈](/zh-TW/concepts/agent-loop) — 詳細的代理程式執行週期
- [閘道通訊協定](/zh-TW/gateway/protocol) — WebSocket 通訊協定契約
- [佇列](/zh-TW/concepts/queue) — 命令佇列與並行處理
- [安全性](/zh-TW/gateway/security) — 信任模型與強化措施
