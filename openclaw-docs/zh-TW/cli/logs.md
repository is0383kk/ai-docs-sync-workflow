---
read_when:
    - 你需要從遠端持續查看閘道日誌（不使用 SSH）
    - 你需要供工具使用的 JSON 日誌行
summary: '`openclaw logs` 的命令列介面參考（透過 RPC 追蹤閘道日誌）'
title: 日誌
x-i18n:
    generated_at: "2026-07-26T07:37:59Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 7c8dc40e70f2eb4f8d6ba8b75b91a33337786a146abbe401079ee374daa5a0c6
    source_path: cli/logs.md
    workflow: 16
---

# `openclaw logs`

透過 RPC 追蹤閘道檔案日誌。支援遠端模式。

## 選項

- `--limit <n>`：要傳回的日誌行數上限（預設為 `200`）
- `--max-bytes <n>`：要從日誌檔案讀取的位元組數上限（預設為 `250000`）
- `--follow`：持續追蹤日誌串流
- `--interval <ms>`：追蹤期間的輪詢間隔（預設為 `1000`）
- `--json`：輸出以行分隔的 JSON 事件
- `--plain`：輸出不含樣式格式的純文字
- `--no-color`：停用 ANSI 色彩
- `--local-time`：以你的本機時區顯示時間戳記（預設）
- `--utc`：以 UTC 顯示時間戳記

## 共用閘道 RPC 選項

- `--url <url>`：閘道 WebSocket URL
- `--token <token>`：閘道權杖
- `--timeout <ms>`：逾時毫秒數（預設為 `30000`）
- `--expect-final`：當閘道呼叫由代理程式支援時，等待最終回應

傳入 `--url` 會略過自動套用的設定認證資訊；如果目標閘道要求驗證，請明確包含 `--token`。

## 範例

```bash
openclaw logs
openclaw logs --follow
openclaw --dev logs --follow
openclaw --profile work logs --follow
openclaw logs --follow --interval 2000
openclaw logs --limit 500 --max-bytes 500000
openclaw logs --json
openclaw logs --plain
openclaw logs --no-color
openclaw logs --utc
openclaw logs --follow --local-time
openclaw logs --url ws://127.0.0.1:18789 --token "$OPENCLAW_GATEWAY_TOKEN"
```

選取的根設定檔會與閘道的輪替檔案相符：預設
設定檔使用 `openclaw-YYYY-MM-DD.log`，具名設定檔則使用
`openclaw-<profile>-YYYY-MM-DD.log`（例如
`openclaw-dev-YYYY-MM-DD.log`）。

## 後援與復原行為

- 如果隱含的本機迴路閘道要求配對、在連線期間關閉，或在 `logs.tail` 回應前逾時，`openclaw logs` 會自動改用已設定的閘道檔案日誌。明確指定的 `--url` 目標絕不使用此後援機制。
- `--follow` 不會在隱含的本機閘道 RPC 失敗後改用該設定檔案，因為過時的並存檔案可能會誤導即時追蹤。在 Linux 上，它會改用可用的作用中使用者 systemd 閘道日誌（並顯示選取的來源），否則會持續重試即時閘道。
- 在 `--follow` 期間，暫時性中斷（WebSocket 關閉、逾時、連線中斷）會觸發採用指數退避的自動重新連線：最多重試 8 次，每次嘗試之間的等待時間上限為 30 秒。每次重試都會將警告輸出至 stderr，而輪詢成功後會輸出一次 `[logs] gateway reconnected` 通知。在 `--json` 模式下，兩者都會以 `{"type":"notice"}` 記錄輸出至 stderr。無法復原的錯誤（驗證失敗、設定錯誤）仍會立即結束。
- 在 `--follow --json` 模式下，日誌來源轉換會以 `{"type":"meta"}` 記錄輸出。請依每個 `sourceKind` 追蹤游標：串流可從閘道檔案輸出（`sourceKind: "file"`）切換至本機日誌後援（`sourceKind: "journal"`、`localFallback: true`，搭配 `service.pid`/`service.unit`），並在復原後切回閘道檔案輸出。請勿假設整個工作階段只會使用一個固定來源或游標，並應容許復原時重播閘道檔案游標所產生的重疊日誌行。

## 相關內容

- [日誌概覽](/zh-TW/logging)
- [閘道命令列介面](/zh-TW/cli/gateway)
- [命令列介面參考](/zh-TW/cli)
- [閘道日誌](/zh-TW/gateway/logging)
