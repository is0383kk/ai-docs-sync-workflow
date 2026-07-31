---
read_when:
    - 你使用語音通話外掛，並希望每個命令列介面進入點
    - 你需要 setup、smoke、call、continue、speak、dtmf、end、status、tail、latency、expose 和 start 的旗標表格與預設值
summary: '`openclaw voicecall` 的命令列介面參考（語音通話外掛命令介面）'
title: 語音通話
x-i18n:
    generated_at: "2026-07-26T08:14:06Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: aec445886cccb79c9212dd9f1f448ff9634274deb380632be786478c9bb29670
    source_path: cli/voicecall.md
    workflow: 16
---

# `openclaw voicecall`

`voicecall` 是由外掛提供的命令。只有在安裝並啟用 voice-call
外掛後才會顯示。

閘道執行時，操作命令（`call`、`start`、
`continue`、`speak`、`dtmf`、`end`、`status`）會路由至該閘道的
voice-call 執行階段。如果無法連線至任何閘道，則會改用獨立的
命令列介面執行階段。

## 子命令

```bash
openclaw voicecall setup    [--json]
openclaw voicecall smoke    [-t <phone>] [--message <text>] [--mode <m>] [--yes] [--json]
openclaw voicecall call     -m <text> [-t <phone>] [--mode <m>]
openclaw voicecall start    --to <phone> [--message <text>] [--mode <m>]
openclaw voicecall continue --call-id <id> --message <text>
openclaw voicecall speak    --call-id <id> --message <text>
openclaw voicecall dtmf     --call-id <id> --digits <digits>
openclaw voicecall end      --call-id <id>
openclaw voicecall status   [--call-id <id>] [--json]
openclaw voicecall tail     [--file <path>] [--since <n>] [--poll <ms>]
openclaw voicecall latency  [--file <path>] [--last <n>]
openclaw voicecall expose   [--mode <m>] [--path <p>] [--port <port>] [--serve-path <p>]
```

| 子命令 | 說明                                                            |
| ---------- | --------------------------------------------------------------- |
| `setup`    | 顯示供應商與網路鉤子的就緒狀態檢查。                     |
| `smoke`    | 執行就緒狀態檢查；只有搭配 `--yes` 才會撥打即時測試電話。 |
| `call`     | 發起撥出語音通話。                                |
| `start`    | `call` 的別名，必須指定 `--to`，`--message` 則為選用。 |
| `continue` | 說出訊息並等待下一個回應。                 |
| `speak`    | 說出訊息，但不等待回應。                 |
| `dtmf`     | 將 DTMF 數字傳送至進行中的通話。                             |
| `end`      | 掛斷進行中的通話。                                         |
| `status`   | 檢查進行中的通話（或透過 `--call-id` 檢查單一通話）。                   |
| `tail`     | 追蹤 `calls.jsonl`（適合在供應商測試期間使用）。              |
| `latency`  | 摘要整理 `calls.jsonl` 的輪次延遲指標。              |
| `expose`   | 切換網路鉤子端點的 Tailscale serve/funnel。         |

## 設定與冒煙測試

### `setup`

預設會輸出便於閱讀的就緒狀態檢查結果。指令碼請傳入 `--json`。

```bash
openclaw voicecall setup
openclaw voicecall setup --json
```

### `smoke`

執行相同的就緒狀態檢查。只有同時提供
`--to` 和 `--yes` 時，才會實際撥打電話。

| 旗標               | 預設值                           | 說明                             |
| ------------------ | --------------------------------- | --------------------------------------- |
| `-t, --to <phone>` | （無）                            | 用於即時冒煙測試的撥打電話號碼。  |
| `--message <text>` | `OpenClaw voice call smoke test.` | 冒煙測試通話期間要說出的訊息。 |
| `--mode <mode>`    | `notify`                          | 通話模式：`notify` 或 `conversation`。  |
| `--yes`            | `false`                           | 實際撥打即時外撥電話。  |
| `--json`           | `false`                           | 輸出機器可讀的 JSON。            |

```bash
openclaw voicecall smoke
openclaw voicecall smoke --to "+15555550123"        # 試執行
openclaw voicecall smoke --to "+15555550123" --yes  # 即時通知通話
```

<Note>
對於外部供應商（`plivo`、`telnyx`、`twilio`），`setup` 和 `smoke` 需要來自 `publicUrl`、通道或 Tailscale 公開功能的公用網路鉤子 URL。系統會拒絕迴路介面或私有 serve 後援，因為電信業者無法連線至該位址。
</Note>

## 通話生命週期

### `call`

發起撥出語音通話。

| 旗標                   | 必要 | 預設值           | 說明                                                                |
| ---------------------- | -------- | ----------------- | -------------------------------------------------------------------------- |
| `-m, --message <text>` | 是      | （無）            | 通話接通時要說出的訊息。                                   |
| `-t, --to <phone>`     | 否       | 設定 `toNumber` | 要撥打的 E.164 電話號碼。                                                |
| `--mode <mode>`        | 否       | `conversation`    | 通話模式：`notify`（說完訊息後掛斷）或 `conversation`（保持通話）。 |

```bash
openclaw voicecall call --to "+15555550123" --message "Hello"
openclaw voicecall call -m "Heads up" --mode notify
```

### `start`

`call` 的別名，使用不同的預設旗標形式。

| 旗標               | 必要 | 預設值        | 說明                              |
| ------------------ | -------- | -------------- | ---------------------------------------- |
| `--to <phone>`     | 是      | （無）         | 要撥打的電話號碼。                    |
| `--message <text>` | 否       | （無）         | 通話接通時要說出的訊息。 |
| `--mode <mode>`    | 否       | `conversation` | 通話模式：`notify` 或 `conversation`。   |

### `continue`

說出訊息並等待回應。

| 旗標               | 必要 | 說明       |
| ------------------ | -------- | ----------------- |
| `--call-id <id>`   | 是      | 通話 ID。          |
| `--message <text>` | 是      | 要說出的訊息。 |

### `speak`

說出訊息，但不等待回應。

| 旗標               | 必要 | 說明       |
| ------------------ | -------- | ----------------- |
| `--call-id <id>`   | 是      | 通話 ID。          |
| `--message <text>` | 是      | 要說出的訊息。 |

### `dtmf`

將 DTMF 數字傳送至進行中的通話。

| 旗標                | 必要 | 說明                                      |
| ------------------- | -------- | ------------------------------------------------ |
| `--call-id <id>`    | 是      | 通話 ID。                                         |
| `--digits <digits>` | 是      | DTMF 數字（例如使用 `ww123456#` 表示等待）。 |

### `end`

掛斷進行中的通話。

| 旗標             | 必要 | 說明 |
| ---------------- | -------- | ----------- |
| `--call-id <id>` | 是      | 通話 ID。    |

### `status`

檢查進行中的通話。

| 旗標             | 預設值 | 說明                  |
| ---------------- | ------- | ---------------------------- |
| `--call-id <id>` | （無）  | 將輸出限制為單一通話。 |
| `--json`         | `false` | 輸出機器可讀的 JSON。 |

```bash
openclaw voicecall status
openclaw voicecall status --json
openclaw voicecall status --call-id <id>
```

## 記錄與指標

### `tail`

追蹤 voice-call JSONL 記錄。啟動時輸出最後 `--since` 行，接著在寫入新行時
串流輸出。

| 旗標            | 預設值                    | 說明                    |
| --------------- | -------------------------- | ------------------------------ |
| `--file <path>` | 從外掛儲存區解析 | `calls.jsonl` 的路徑。         |
| `--since <n>`   | `25`                       | 開始追蹤前要輸出的行數。 |
| `--poll <ms>`   | `250`（最小值 50）         | 輪詢間隔（毫秒）。 |

### `latency`

摘要整理 `calls.jsonl` 中的輪次延遲與聆聽等待指標。輸出為
JSON，包含 `recordsScanned`、`turnLatency` 和 `listenWait` 摘要。

| 旗標            | 預設值                    | 說明                          |
| --------------- | -------------------------- | ------------------------------------ |
| `--file <path>` | 從外掛儲存區解析 | `calls.jsonl` 的路徑。               |
| `--last <n>`    | `200`（最小值 1）          | 要分析的近期記錄數量。 |

## 公開網路鉤子

### `expose`

啟用、停用或變更語音網路鉤子的 Tailscale serve/funnel
設定。

| 旗標                  | 預設值                                   | 說明                                     |
| --------------------- | ----------------------------------------- | ----------------------------------------------- |
| `--mode <mode>`       | `funnel`                                  | `off`、`serve`（tailnet）或 `funnel`（公開）。 |
| `--path <path>`       | 設定 `tailscale.path` 或 `--serve-path` | 要公開的 Tailscale 路徑。                       |
| `--port <port>`       | 設定 `serve.port` 或 `3334`             | 本機網路鉤子連接埠。                             |
| `--serve-path <path>` | 設定 `serve.path` 或 `/voice/webhook`   | 本機網路鉤子路徑。                             |

```bash
openclaw voicecall expose --mode serve
openclaw voicecall expose --mode funnel
openclaw voicecall expose --mode off
```

<Warning>
只將網路鉤子端點公開至你信任的網路。可行時，優先使用 Tailscale Serve，而非 Funnel。
</Warning>

## 相關內容

- [命令列介面參考](/zh-TW/cli)
- [語音通話外掛](/zh-TW/plugins/voice-call)
