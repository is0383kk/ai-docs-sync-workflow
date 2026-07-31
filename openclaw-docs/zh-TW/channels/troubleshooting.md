---
read_when:
    - 頻道傳輸顯示已連線，但回覆失敗
    - 深入閱讀提供者文件前，你需要先進行頻道特定檢查
summary: 透過各頻道的失敗特徵與修正方式，快速排解頻道層級問題
title: 頻道疑難排解
x-i18n:
    generated_at: "2026-07-26T08:25:51Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 3891595e4b5aca9de7997a6e908fa1c9246579032bfdfa1656a6992d644c3ecc
    source_path: channels/troubleshooting.md
    workflow: 16
---

當頻道已連線但行為不正確時，請使用此頁面。

## 命令執行順序

請先依序執行以下命令：

```bash
openclaw status
openclaw gateway status
openclaw logs --follow
openclaw doctor
openclaw channels status --probe
```

正常基準：

- `Runtime: running`
- `Connectivity probe: ok`
- `Capability: read-only`、`write-capable` 或 `admin-capable`
- 頻道探測顯示傳輸已連線，並在支援的情況下顯示 `works` 或 `audit ok`

## 更新後

若更新後 Telegram、iMessage、BlueBubbles 時期的設定或其他外掛頻道消失，請使用以下命令。

```bash
openclaw status --all
openclaw doctor --fix
openclaw gateway restart
openclaw status --all
```

在 `openclaw
status --all` 中尋找 `plugin load failed: dependency tree corrupted; run openclaw doctor --fix`。這表示頻道已設定，但外掛設定／載入遇到損壞的相依性樹，而未註冊頻道。`openclaw doctor --fix` 會清除過時的外掛執行階段相依性符號連結和過時的驗證陰影，接著 `openclaw gateway restart` 會重新載入乾淨狀態。

## WhatsApp

### WhatsApp 失敗徵兆

| 症狀                                | 最快檢查方式                                          | 修正方式                                                                                                               |
| ----------------------------------- | --------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------- |
| 已連線但沒有私訊回覆                | `openclaw pairing list whatsapp`                                  | 核准傳送者，或切換私訊政策／允許清單。                                                                                 |
| 群組訊息遭忽略                      | 檢查設定中的 `requireMention` 和提及模式           | 提及機器人，或放寬該群組的提及政策。                                                                                   |
| QR 登入逾時並顯示 408               | 檢查閘道的 `HTTPS_PROXY`／`HTTP_PROXY` 環境 | 設定可連線的 Proxy；僅在略過限制時使用 `NO_PROXY`。                                                            |
| 隨機中斷連線／重新登入循環          | `openclaw channels status --probe` 和日誌                           | 即使目前已連線，近期的重新連線仍會被標記；監看日誌、重新啟動閘道，若連線持續不穩定則重新連結。                          |
| `status=408 Request Time-out` 循環             | 依序探測、檢查日誌、執行 doctor，再檢查閘道狀態      | 先修正主機連線能力／時序；若循環持續，請備份驗證資料並重新連結帳號。                                                    |
| 回覆延遲數秒／數分鐘才送達          | `openclaw doctor --fix`                                  | 當經確認過時的本機終端介面用戶端正在降低閘道事件迴圈效能時，Doctor 會將其停止。                                         |

完整疑難排解：[WhatsApp 疑難排解](/zh-TW/channels/whatsapp#troubleshooting)

## Telegram

### Telegram 失敗徵兆

| 症狀                                      | 最快檢查方式                                  | 修正方式                                                                                                                      |
| ----------------------------------------- | --------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------- |
| `/start`，但沒有可用的回覆流程 | `openclaw pairing list telegram`                            | 核准配對或變更私訊政策。                                                                                                      |
| 機器人在線上，但群組維持靜默              | 確認提及要求和機器人隱私模式                  | 停用隱私模式以允許群組可見性，或提及機器人。                                                                                  |
| 傳送失敗並出現網路錯誤                    | 檢查日誌中的 Telegram API 呼叫失敗            | 修正前往 `api.telegram.org` 的 DNS／IPv6／Proxy 路由。                                                                        |
| 啟動時回報 `getMe returned 401`             | 檢查已設定的 Token 來源                       | 重新複製或產生 BotFather Token，並更新 `botToken`、`tokenFile` 或預設帳號的 `TELEGRAM_BOT_TOKEN`。                |
| 輪詢停滯或重新連線緩慢                    | 使用 `openclaw logs --follow` 進行輪詢診斷          | 升級；持續停滯通常表示 Proxy／DNS／IPv6 有問題。                                                                               |
| 啟動時拒絕 `setMyCommands`             | 檢查日誌中的 `BOT_COMMANDS_TOO_MUCH`               | 減少外掛／Skill／自訂 Telegram 命令，或停用原生選單。                                                                         |
| 升級後允許清單將你封鎖                    | `openclaw security audit` 和設定允許清單             | 執行 `openclaw doctor --fix`，或將 `@username` 替換為數字傳送者 ID。                                                        |

完整疑難排解：[Telegram 疑難排解](/zh-TW/channels/telegram#troubleshooting)

## Discord

### Discord 失敗徵兆

| 症狀                                      | 最快檢查方式                                                                                                                  | 修正方式                                                                                                                                                                                                                                                             |
| ----------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 機器人在線上，但沒有伺服器回覆            | `openclaw channels status --probe`                                                                                                            | 允許伺服器／頻道，並確認訊息內容 Intent。                                                                                                                                                                                                                             |
| 群組訊息遭忽略                            | 檢查日誌中因提及閘控而捨棄的紀錄                                                                                              | 提及機器人，或設定伺服器／頻道的 `requireMention: false`。                                                                                                                                                                                                                 |
| 有輸入中／Token 用量，但沒有 Discord 訊息 | 檢查這是否為環境房間事件，或模型遺漏 `message(action=send)` 的已加入 `message_tool` 房間                                      | 檢查閘道詳細日誌中遭抑制的最終承載資料中繼資料、確認 `messages.groupChat.unmentionedInbound`、閱讀[環境房間事件](/zh-TW/channels/ambient-room-events)，或對一般群組要求保留 `messages.groupChat.visibleReplies: "automatic"`。                                                                 |
| 缺少私訊回覆                              | `openclaw pairing list discord`                                                                                                            | 核准私訊配對或調整私訊政策。                                                                                                                                                                                                                                         |

完整疑難排解：[Discord 疑難排解](/zh-TW/channels/discord#troubleshooting)

## Slack

### Slack 失敗徵兆

| 症狀                                | 最快檢查方式                                      | 修正方式                                                                                                                                                  |
| ----------------------------------- | ------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Socket 模式已連線但沒有回應         | `openclaw channels status --probe`                                | 確認應用程式 Token、機器人 Token 和必要的 Scope；使用 SecretRef 的設定請留意 `botTokenStatus`／`appTokenStatus = configured_unavailable`。                                      |
| 私訊遭封鎖                          | `openclaw pairing list slack`                                | 核准配對或放寬私訊政策。                                                                                                                                  |
| 頻道訊息遭忽略                      | 檢查 `groupPolicy` 和頻道允許清單            | 允許該頻道，或將政策切換為 `open`。                                                                                                           |

完整疑難排解：[Slack 疑難排解](/zh-TW/channels/slack#troubleshooting)

## iMessage

### iMessage 失敗徵兆

| 症狀                                      | 最快檢查方式                                        | 修正方式                                                                  |
| ----------------------------------------- | --------------------------------------------------- | ------------------------------------------------------------------------- |
| 非 macOS 上缺少 `imsg` 或執行失敗 | `openclaw channels status --probe --channel imessage`                              | 在「訊息」所在的 Mac 上執行 OpenClaw，或為 `cliPath` 使用 SSH 包裝器。 |
| 可傳送但無法在 macOS 上接收              | 檢查 macOS 對「訊息」自動化的隱私權限               | 重新授予 TCC 權限，並重新啟動頻道程序。                                   |
| 私訊傳送者遭封鎖                          | `openclaw pairing list imessage`                                  | 核准配對或更新允許清單。                                                   |

完整疑難排解：[iMessage 疑難排解](/zh-TW/channels/imessage#troubleshooting)

## Signal

### Signal 失敗徵兆

| 症狀                          | 最快檢查方式                                | 修正方式                                                        |
| ----------------------------- | ------------------------------------------- | --------------------------------------------------------------- |
| Daemon 可連線但機器人無回應   | `openclaw channels status --probe`                          | 確認 `signal-cli` Daemon URL／帳號和接收模式。            |
| 私訊遭封鎖                    | `openclaw pairing list signal`                          | 核准傳送者或調整私訊政策。                                      |
| 群組回覆未觸發                | 檢查群組允許清單和提及模式                  | 新增傳送者／群組，或放寬閘控。                                  |

完整疑難排解：[Signal 疑難排解](/zh-TW/channels/signal#troubleshooting)

## QQ Bot

### QQ Bot 失敗徵兆

| 症狀                         | 最快檢查方式                                      | 修正方式                                                      |
| ---------------------------- | ------------------------------------------------- | ------------------------------------------------------------- |
| 機器人回覆「去了火星」       | 確認設定中的 `appId` 和 `clientSecret` | 設定認證資訊或重新啟動閘道。                                  |
| 沒有傳入訊息                 | `openclaw channels status --probe`                                | 在 QQ 開放平台確認認證資訊。                                  |
| 語音未轉錄                   | 檢查 STT 提供者設定                               | 設定 `channels.qqbot.stt` 或 `tools.media.audio`。               |
| 主動訊息未送達               | 檢查 QQ 平台的互動要求                            | 若近期沒有互動，QQ 可能會封鎖由機器人發起的訊息。              |

完整疑難排解：[QQ Bot 疑難排解](/zh-TW/channels/qqbot#troubleshooting)

## Matrix

### Matrix 失敗徵兆

| 症狀                             | 最快速的檢查方式                          | 修正方式                                                                       |
| ----------------------------------- | -------------------------------------- | ------------------------------------------------------------------------- |
| 已登入但忽略聊天室訊息 | `openclaw channels status --probe`     | 檢查 `groupPolicy`、聊天室允許清單及提及閘控。                  |
| 私訊未處理                  | `openclaw pairing list matrix`         | 核准傳送者或調整私訊政策。                                       |
| 加密聊天室失敗                | `openclaw matrix verify status`        | 重新驗證裝置，然後檢查 `openclaw matrix verify backup status`。  |
| 備份還原處於待處理狀態或已損壞    | `openclaw matrix verify backup status` | 執行 `openclaw matrix verify backup restore`，或使用復原金鑰重新執行。 |
| 交叉簽署／啟動程序看起來不正確 | `openclaw matrix verify bootstrap`     | 一次修復祕密儲存空間、交叉簽署及備份狀態。       |

完整設定與組態：[Matrix](/zh-TW/channels/matrix)

## 相關內容

- [配對](/zh-TW/channels/pairing)
- [頻道路由](/zh-TW/channels/channel-routing)
- [閘道疑難排解](/zh-TW/gateway/troubleshooting)
