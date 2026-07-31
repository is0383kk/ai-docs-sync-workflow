---
read_when:
    - 你想要將 Gmail Pub/Sub 事件串接至 OpenClaw
    - 你需要完整的旗標清單和預設值
summary: '`openclaw webhooks` 的命令列介面參考（Gmail Pub/Sub 設定與執行程式）'
title: 網路鉤子
x-i18n:
    generated_at: "2026-07-26T08:20:38Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 83fff0ac2ce247402f45523eda0b5cdd551bd65212636118698e45cb8740236c
    source_path: cli/webhooks.md
    workflow: 16
---

# `openclaw webhooks`

網路鉤子輔助工具與整合。目前此介面僅適用於以內建 `gog` 監看器建構的 Gmail Pub/Sub 流程。

## 子命令

```bash
openclaw webhooks gmail setup --account <email> [...]
openclaw webhooks gmail run   [--account <email>] [...]
```

| 子命令    | 說明                                                                           |
| ------------- | ------------------------------------------------------------------------------------- |
| `gmail setup` | 一次性精靈：設定 Gmail 監看、Pub/Sub 主題／訂閱，以及 OpenClaw 鉤子傳遞。 |
| `gmail run`   | 在前景執行 `gog watch serve` 與監看自動續訂迴圈。               |

<Note>
設定 `hooks.enabled=true` 與 `hooks.gmail.account` 後（由 `gmail setup` 設定），閘道也會在啟動時自動啟動 `gog gmail watch serve`。`gmail run` 會在前景執行相同邏輯，適合用於偵錯，或在停用閘道監看器時使用。自動啟動的詳細資訊與 `OPENCLAW_SKIP_GMAIL_WATCHER` 的退出設定，請參閱 [Gmail Pub/Sub 整合](/zh-TW/automation/cron-jobs#gmail-pubsub-integration)。
</Note>

## `webhooks gmail setup`

```bash
openclaw webhooks gmail setup --account you@example.com
openclaw webhooks gmail setup --account you@example.com --project my-gcp-project --json
openclaw webhooks gmail setup --account you@example.com --hook-url https://gateway.example.com/hooks/gmail
```

若缺少 `gcloud` 與 `gog`，會加以安裝，並驗證 `gcloud`、建立 Pub/Sub 主題與訂閱、啟動 Gmail 監看，以及寫入包含 `hooks.enabled=true` 的 `hooks.gmail` 設定。輸出 `Next: openclaw webhooks gmail run`。

### 必填

| 旗標                | 說明             |
| ------------------- | ----------------------- |
| `--account <email>` | 要監看的 Gmail 帳戶。 |

### Pub/Sub 選項

| 旗標                    | 預設值                | 說明                                                                                                                             |
| ----------------------- | ---------------------- | --------------------------------------------------------------------------------------------------------------------------------------- |
| `--project <id>`        | （無）                 | GCP 專案 ID（OAuth 用戶端的擁有者）。依序回退至主題本身的專案 ID，以及從 `gog` 認證資訊解析出的專案。 |
| `--topic <name>`        | `gog-gmail-watch`      | Pub/Sub 主題名稱。                                                                                                                     |
| `--subscription <name>` | `gog-gmail-watch-push` | Pub/Sub 訂閱名稱。                                                                                                              |
| `--label <label>`       | `INBOX`                | 要監看的 Gmail 標籤。                                                                                                                   |
| `--push-endpoint <url>` | （無）                 | 明確指定的 Pub/Sub 推送端點。覆寫 Tailscale。                                                                                    |

### OpenClaw 傳遞選項

| 旗標                   | 預設值                                      | 說明                                |
| ---------------------- | -------------------------------------------- | ------------------------------------------ |
| `--hook-url <url>`     | 由 `hooks.path` 與閘道連接埠建構 | OpenClaw 網路鉤子 URL。                      |
| `--hook-token <token>` | `hooks.token`，或產生的權杖          | OpenClaw 網路鉤子權杖。                    |
| `--push-token <token>` | 產生的權杖                              | 轉送至 `gog watch serve` 的推送權杖。 |

### `gog watch serve` 選項

| 旗標                  | 預設值         | 說明                                                                                                                                  |
| --------------------- | --------------- | -------------------------------------------------------------------------------------------------------------------------------------------- |
| `--bind <host>`       | `127.0.0.1`     | `gog watch serve` 繫結主機。                                                                                                                 |
| `--port <port>`       | `8788`          | `gog watch serve` 連接埠。                                                                                                                      |
| `--path <path>`       | `/gmail-pubsub` | `gog watch serve` 路徑。在啟用 Tailscale 且未明確指定目標時，會強制設為 `/`，因為 Tailscale 會在代理前移除路徑。 |
| `--include-body`      | `true`          | 包含電子郵件內文片段。沒有可將其關閉的命令列介面旗標；請改為在設定中設置 `hooks.gmail.includeBody: false`。                  |
| `--max-bytes <n>`     | `20000`         | 每個內文片段的位元組上限。                                                                                                                  |
| `--renew-minutes <n>` | `720`（12 小時）     | 每 N 分鐘續訂 Gmail 監看。                                                                                                           |

### Tailscale 公開存取

| 旗標                      | 預設值  | 說明                                                      |
| ------------------------- | -------- | ---------------------------------------------------------------- |
| `--tailscale <mode>`      | `funnel` | 透過 Tailscale 公開推送端點：`funnel`、`serve` 或 `off`。 |
| `--tailscale-path <path>` | （無）   | tailscale serve/funnel 的路徑。                                 |
| `--tailscale-target <t>`  | （無）   | Tailscale serve/funnel 目標（連接埠、`host:port` 或 URL）。       |

### 輸出

| 旗標     | 說明                                       |
| -------- | ------------------------------------------------- |
| `--json` | 輸出機器可讀的摘要，而非文字。 |

## `webhooks gmail run`

```bash
openclaw webhooks gmail run --account you@example.com
```

在前景執行 `gog watch serve` 與監看自動續訂迴圈；若 `gog watch serve` 意外結束，會延遲 2 秒後重新啟動。

`run` 接受與 `setup` 相同的 Pub/Sub、OpenClaw 傳遞、`gog watch serve` 與 Tailscale 旗標，但有以下例外：

- `--account` 在 `run` 上為**選填**；它會回退至 `hooks.gmail.account`。
- `run` **不**接受 `--project`、`--push-endpoint` 或 `--json`。
- 每個旗標都會先回退至對應的 `hooks.gmail.*` 設定值（由 `setup` 寫入），再回退至 `setup` 使用的相同內建預設值，但有一項例外：若旗標與 `hooks.gmail.tailscale.mode` 均未設定，`--tailscale` 在 `run` 上的預設值為 `off`（而非 `funnel`）。

| 類別          | 旗標                                                                            |
| ----------------- | -------------------------------------------------------------------------------- |
| Pub/Sub           | `--account`、`--topic`、`--subscription`、`--label`                              |
| OpenClaw 傳遞 | `--hook-url`、`--hook-token`、`--push-token`                                     |
| `gog watch serve` | `--bind`、`--port`、`--path`、`--include-body`、`--max-bytes`、`--renew-minutes` |
| Tailscale         | `--tailscale`、`--tailscale-path`、`--tailscale-target`                          |

<Note>
對於 `run`，`--topic` 值是完整的 Pub/Sub 主題路徑（`projects/.../topics/...`），而非僅有簡短主題名稱。
</Note>

## 相關內容

- [命令列介面參考](/zh-TW/cli)
- [網路鉤子自動化](/zh-TW/automation/cron-jobs)
- [Gmail Pub/Sub 整合](/zh-TW/automation/cron-jobs#gmail-pubsub-integration)
