---
read_when:
    - 你想從指令碼執行一次代理程式回合（可選擇傳送回覆）
summary: '`openclaw agent` 的命令列介面參考（透過閘道傳送一個代理程式回合）'
title: 代理程式
x-i18n:
    generated_at: "2026-07-26T08:27:05Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 1a4c139a3b235d6a56ba63063737b80f93448c2dbb7a92c6d0756fb19a9f95e4
    source_path: cli/agent.md
    workflow: 16
---

# `openclaw agent`

透過閘道執行一個代理程式回合。明確的 `--local` 旗標是唯一的嵌入式執行路徑。

至少傳入一個工作階段選擇器：`--to`、`--session-key`、`--session-id` 或 `--agent`。

相關：[代理程式傳送工具](/zh-TW/tools/agent-send)

## 選項

- `-m, --message <text>`：訊息本文
- `--message-file <path>`：從 UTF-8 檔案讀取訊息本文
- `-t, --to <dest>`：用於衍生工作階段金鑰的收件者
- `--session-key <key>`：用於路由的明確工作階段金鑰
- `--session-id <id>`：明確的工作階段 ID
- `--agent <id>`：代理程式 ID；覆寫路由繫結
- `--model <id>`：覆寫此次執行的模型（`provider/model` 或模型 ID）
- `--thinking <level>`：代理程式思考層級（`off`、`minimal`、`low`、`medium`、`high`，以及供應商支援的自訂層級，例如 `xhigh`、`adaptive` 或 `max`）
- `--verbose <on|off>`：保存工作階段的詳細程度
- `--channel <channel>`：傳遞頻道；省略時使用主要工作階段頻道
- `--reply-to <target>`：覆寫傳遞目標
- `--reply-channel <channel>`：覆寫傳遞頻道
- `--reply-account <id>`：覆寫傳遞帳號
- `--local`：直接執行嵌入式代理程式（預先載入外掛登錄檔後）
- `--deliver`：將回覆傳回所選頻道／目標
- `--timeout <seconds>`：覆寫此命令的代理程式回合截止時間（預設為 600 或 `agents.defaults.timeoutSeconds`）；`0` 會停用整體截止時間。600 秒的備援值屬於此命令列介面命令，而非一般閘道回合；後者的預設值為 48 小時。
- `--json`：輸出 JSON

## 範例

```bash
openclaw agent --to +15555550123 --message "狀態更新" --deliver
openclaw agent --agent ops --message "摘要記錄"
openclaw agent --agent ops --message-file ./task.md
openclaw agent --agent ops --model openai/gpt-5.4 --message "摘要記錄"
openclaw agent --session-key agent:ops:incident-42 --message "摘要狀態"
openclaw agent --agent ops --session-key incident-42 --message "摘要狀態"
openclaw agent --session-id 1234 --message "摘要收件匣" --thinking medium
openclaw agent --to +15555550123 --message "追蹤記錄" --verbose on --json
openclaw agent --agent ops --message "產生報告" --deliver --reply-channel slack --reply-to "#reports"
openclaw agent --agent ops --message "在本機執行" --local
```

## 注意事項

- 請只傳入 `--message` 或 `--message-file` 其中一個。`--message-file` 會移除開頭的 UTF-8 BOM 並保留多行內容；非有效 UTF-8 的檔案會遭拒絕。大於 4 MiB 的檔案會在分派前遭拒絕。
- 斜線命令（例如 `/compact`）無法透過 `--message` 執行。命令列介面會拒絕這些命令，並改為指向其一級命令（壓縮使用 `openclaw sessions compact <key>`）。
- `--local` 執行為單次性：為該次執行開啟的內建 MCP 回送資源與暖啟 Claude stdio 工作階段，會在回覆後停用，因此指令碼叫用不會留下仍在執行的本機子行程。以閘道為後端的執行，則會將閘道所擁有的 MCP 回送資源保留在執行中的閘道行程下。
- 當重新啟動復原仍在等待中時，使用 `--local` 的獨立嵌入式執行會拒絕重複使用現有的主要工作階段。請透過運作正常的閘道執行該回合，或在該處使用 `/new` 或 `/reset` 重設；獨立的嵌入式行程無法安全地與閘道掃描器協調該復原擁有者。
- 同時使用 `--agent`、`--channel` 和 `--to` 時，工作階段路由會遵循頻道的標準收件者與 `session.dmScope`。具有穩定僅限傳出收件者身分的頻道，會使用供應商擁有的工作階段，並與代理程式的主要工作階段隔離。`--reply-channel` 和 `--reply-account` 僅影響傳遞。
- `--session-key` 會選取明確的工作階段金鑰。以代理程式為前綴的金鑰必須使用 `agent:<agent-id>:<session-key>`；若兩者皆有提供，`--agent` 必須符合金鑰的代理程式 ID。若有提供 `--agent`，不含前綴且非哨兵值的金鑰會限定於該值；否則限定於已設定的預設代理程式。例如，`--agent ops --session-key incident-42` 會路由至 `agent:ops:incident-42`。只有在未提供 `--agent` 時，常值金鑰 `global` 和 `unknown` 才會保持不限定範圍。
- `--json` 會將 stdout 保留給 JSON 回應；閘道、外掛和 `--local` 診斷資訊會寫入 stderr，讓指令碼能直接剖析 stdout。
- 暫時性握手重試用盡後，閘道逾時或連線關閉會使命令失敗；命令列介面絕不會在未告知的情況下，改以嵌入式方式重新執行該回合。傳輸中斷的結果並不確定——閘道可能已接受並仍會完成該回合——因此 stderr 提示會要求先檢查 `openclaw gateway status` 與工作階段逐字稿，再重試或使用 `--local` 重新執行，以免執行該回合兩次。
- `SIGTERM`/`SIGINT` 會中斷等待中的閘道後端要求；如果閘道已接受該次執行，命令列介面也會在結束前，針對該執行 ID 傳送 `chat.abort`。`--local` 執行會收到相同訊號，但不會傳送 `chat.abort`。啟動器子行程若因第一個轉送的 `SIGINT` 或 `SIGTERM` 而終止，將分別以狀態 130 或 143 結束。若內部執行去重金鑰已對此工作階段有作用中的執行，回應會回報 `status: "in_flight"`，且非 JSON 命令列介面會列印 stderr 診斷資訊，而非空白回覆。對於外部 cron/systemd 包裝函式，請保留如 `timeout -k 60 600 openclaw agent ...` 的強制終止後援機制，以便在關閉程序無法排空時，由監督程式回收該行程。
- 當此命令觸發重新產生 `models.json` 時，由 SecretRef 管理的供應商認證資訊會保存為非機密標記（例如環境變數名稱、`secretref-env:ENV_VAR_NAME` 或 `secretref-managed`），絕不會解析為機密純文字。標記寫入內容來自作用中的來源設定快照，而非已解析的執行階段機密值。

## JSON 傳遞狀態

搭配 `--json --deliver` 使用時，命令列介面 JSON 回應會包含頂層的 `deliveryStatus`，讓指令碼能區分已傳遞、已抑制、部分傳遞及傳送失敗：

```json
{
  "payloads": [{ "text": "報告已就緒", "mediaUrl": null }],
  "meta": { "durationMs": 1200 },
  "deliveryStatus": {
    "requested": true,
    "attempted": true,
    "status": "sent",
    "succeeded": true,
    "resultCount": 1
  }
}
```

以閘道為後端的命令列介面回應，也會在 `result.deliveryStatus` 保留原始閘道結果格式。

`deliveryStatus.status` 為下列其中之一：

| 狀態             | 意義                                                                                                                                       |
| ---------------- | ------------------------------------------------------------------------------------------------------------------------------------------ |
| `sent`           | 傳遞完成。                                                                                                                                 |
| `suppressed`     | 刻意未傳送（例如傳訊鉤子已取消傳送，或沒有可見結果）。此為終止狀態，不重試。                                                               |
| `partial_failed` | 至少一個承載資料已傳送，之後的承載資料才失敗。                                                                                             |
| `failed`         | 沒有完成任何持久傳送，或傳遞預檢失敗。                                                                                                     |

常見欄位：

- `requested`：物件存在時一律為 `true`。
- `attempted`：持久傳送路徑執行後為 `true`；預檢失敗或沒有可見承載資料時為 `false`。
- `succeeded`：`true`、`false` 或 `"partial"`；`"partial"` 與 `status: "partial_failed"` 配對。
- `reason`：來自持久傳遞或預檢驗證的小寫 snake-case 原因。已知值包括 `cancelled_by_message_sending_hook`、`no_visible_payload`、`no_visible_result`、`channel_resolved_to_internal`、`unknown_channel`、`invalid_delivery_target` 和 `no_delivery_target`；失敗的持久傳送也可能回報失敗階段。由於此集合可能擴充，請將未知值視為不透明值。
- `resultCount`：頻道傳送結果的數量（若有）。
- `sentBeforeError`：部分失敗在發生錯誤前已傳送至少一個承載資料時為 `true`。
- `error`：傳送失敗或部分失敗時為 `true`。
- `errorMessage`：僅在擷取到底層傳遞錯誤訊息時存在。預檢失敗會包含 `error`/`reason`，但不包含 `errorMessage`。
- `payloadOutcomes`：選用的個別承載資料結果，可能包含 `index`、`status`、`reason`、`resultCount`、`error`、`stage`、`sentBeforeError`，或鉤子中繼資料（若有）。

## 相關

- [命令列介面參考](/zh-TW/cli)
- [代理程式執行階段](/zh-TW/concepts/agent)
