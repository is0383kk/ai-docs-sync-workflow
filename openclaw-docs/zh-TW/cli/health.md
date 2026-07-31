---
read_when:
    - 你想要快速檢查執行中閘道的健康狀態
summary: '`openclaw health` 的命令列介面參考（透過 RPC 取得閘道健康狀態快照）'
title: 健康狀態
x-i18n:
    generated_at: "2026-07-26T08:13:13Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 51cc0e3dd61af3e6fa460dd646bfa1c3e5bd1a52da860eac26c12101151d081d
    source_path: cli/health.md
    workflow: 16
---

# `openclaw health`

透過 WebSocket RPC 從執行中的閘道擷取健康狀態快照（命令列介面不會直接使用頻道通訊端）。

## 選項

| 旗標             | 預設值 | 說明                                                                       |
| ---------------- | ------- | --------------------------------------------------------------------------------- |
| `--json`         | `false` | 輸出機器可讀的 JSON，而非文字。                                      |
| `--timeout <ms>` | `10000` | 連線逾時時間（毫秒）。                                               |
| `--verbose`      | `false` | 強制執行即時探測，並展開所有已設定帳號與代理程式的輸出。 |
| `--debug`        | `false` | `--verbose` 的別名。                                                            |

範例：

```bash
openclaw health
openclaw health --json
openclaw health --timeout 2500
openclaw health --verbose
openclaw health --debug
```

## 行為

- 未使用 `--verbose` 時，閘道可傳回快取的快照（最長 60 秒內視為最新，且與即時頻道執行階段狀態相同），並在背景重新整理，以供下一個呼叫者使用。
- `--verbose` 會強制執行即時探測（逐一探測各頻道帳號）、輸出閘道連線詳細資訊，並展開所有已設定帳號與代理程式的可讀文字輸出，而非僅顯示預設代理程式。
- `--json` 一律傳回完整快照：頻道、各帳號探測結果、外掛載入狀態、上下文引擎隔離狀態、模型定價快取狀態、事件迴圈健康狀態、傳遞佇列的無法處理項目，以及各代理程式的工作階段儲存區。
- 當外送傳遞或傳入頻道事件被標記為無法處理時，文字輸出會回報其數量及最早失敗項目的經過時間。傳入數量會依頻道帳號分組；可使用 [`openclaw channels dead-letters`](/zh-TW/cli/channels#inbound-dead-letters) 檢查或復原個別事件。

## 相關內容

- [命令列介面參考](/zh-TW/cli)
- [`openclaw status`](/zh-TW/cli/status) — 在不取得完整健康狀態快照的情況下，執行本機診斷與頻道探測
- [閘道健康狀態](/zh-TW/gateway/health)
