---
read_when:
    - 新增或變更外部命令列介面整合功能
    - 偵錯 RPC 轉接器（signal-cli、imsg）
summary: 外部命令列介面（signal-cli、imsg）的 RPC 介接器與閘道模式
title: RPC 轉接器
x-i18n:
    generated_at: "2026-07-26T08:06:13Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 7deee8154dc824db4eccca9a26381711693972ba2606aec47d657e3724b3a5dd
    source_path: reference/rpc.md
    workflow: 16
---

OpenClaw 透過 JSON-RPC 整合外部命令列介面。目前使用兩種模式。

## 模式 A：HTTP 常駐程式（signal-cli）

- `signal-cli` 以常駐程式形式執行，透過 HTTP 使用 JSON-RPC。
- 事件串流採用 SSE（`/api/v1/events`）。
- 健康狀態探測：`/api/v1/check`。
- 當 `channels.signal.transport.kind="managed-native"`（預設值）時，OpenClaw 會管理生命週期。

設定與端點請參閱 [Signal](/zh-TW/channels/signal)。

## 模式 B：stdio 子行程（imsg）

- OpenClaw 會產生 `imsg rpc` 子行程以處理 [iMessage](/zh-TW/channels/imessage)。
- JSON-RPC 透過 stdin/stdout 以行分隔（每行一個 JSON 物件）。
- 不使用 TCP 連接埠，也不需要常駐程式。

使用的核心方法：

- `watch.subscribe` → 通知（`method: "message"`）
- `watch.unsubscribe`
- `send`
- `chats.list`（探測／診斷）

設定與定址方式請參閱 [iMessage](/zh-TW/channels/imessage)（建議優先使用 `chat_id`，而非顯示字串）。

## 轉接器準則

- 閘道管理行程（啟動／停止與提供者生命週期連動）。
- 確保 RPC 用戶端具備韌性：設定逾時，並在結束時重新啟動。
- 優先使用穩定 ID（例如 `chat_id`），而非顯示字串。

## 相關內容

- [閘道通訊協定](/zh-TW/gateway/protocol)
