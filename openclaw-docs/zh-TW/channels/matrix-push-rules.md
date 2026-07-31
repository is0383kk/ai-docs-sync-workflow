---
read_when:
    - 為自行託管的 Synapse 或 Tuwunel 設定 Matrix 靜默串流
    - 使用者只希望在區塊完成時收到通知，而不是每次編輯預覽時都收到通知
summary: 針對靜默完成預覽編輯的各收件者 Matrix 推播規則
title: Matrix 靜默預覽的推送規則
x-i18n:
    generated_at: "2026-07-26T07:43:52Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 1c58e7e796c3ae6d1ee25de229e4592ab8b4fb4d0d50a9cf868ab5ef35b1dab5
    source_path: channels/matrix-push-rules.md
    workflow: 16
---

當 `channels.matrix.streaming.mode` 為 `"quiet"` 時，OpenClaw 會透過就地編輯單一預覽事件來串流回覆。預覽會以不發出通知的 `m.notice` 事件傳送，而最終編輯則以 `content["com.openclaw.finalized_preview"] = true` 標記。只有在個別使用者的推播規則符合此標記時，Matrix 用戶端才會針對該最終編輯發出通知。本頁適用於自行託管 Matrix，並希望為每個接收者帳號安裝此規則的營運人員。

`streaming.mode: "progress"` 會透過相同路徑完成草稿，因此同一規則也會針對進度模式的最終編輯觸發。

若只需要 Matrix 的標準通知行為，請使用 `streaming.mode: "partial"` 或關閉串流。請參閱 [Matrix 頻道設定](/zh-TW/channels/matrix#streaming-previews)。

## 先決條件

- 接收者使用者 = 應收到通知的人
- 機器人使用者 = 傳送回覆的 OpenClaw Matrix 帳號
- 以下 API 呼叫請使用接收者使用者的存取權杖
- 在推播規則中，將 `sender` 與機器人使用者的完整 MXID 進行比對
- 接收者帳號必須已有正常運作的推播器；只有在一般 Matrix 推播傳送正常時，靜默預覽規則才能運作

## 步驟

<Steps>
  <Step title="設定靜默預覽">

```json5
{
  channels: {
    matrix: {
      streaming: { mode: "quiet" },
    },
  },
}
```

  </Step>

  <Step title="取得接收者的存取權杖">
    如有可能，請重複使用現有的用戶端工作階段權杖。若要建立新的權杖：

```bash
curl -sS -X POST \
  "https://matrix.example.org/_matrix/client/v3/login" \
  -H "Content-Type: application/json" \
  --data '{
    "type": "m.login.password",
    "identifier": { "type": "m.id.user", "user": "@alice:example.org" },
    "password": "REDACTED"
  }'
```

  </Step>

  <Step title="確認推播器存在">

```bash
curl -sS \
  -H "Authorization: Bearer $USER_ACCESS_TOKEN" \
  "https://matrix.example.org/_matrix/client/v3/pushers"
```

若未傳回任何推播器，請先修復此帳號的一般 Matrix 推播傳送，再繼續操作。

  </Step>

  <Step title="安裝覆寫推播規則">
    安裝同時比對最終預覽標記與機器人 MXID 傳送者的規則：

```bash
curl -sS -X PUT \
  "https://matrix.example.org/_matrix/client/v3/pushrules/global/override/openclaw-finalized-preview-botname" \
  -H "Authorization: Bearer $USER_ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  --data '{
    "conditions": [
      { "kind": "event_match", "key": "type", "pattern": "m.room.message" },
      {
        "kind": "event_property_is",
        "key": "content.m\\.relates_to.rel_type",
        "value": "m.replace"
      },
      {
        "kind": "event_property_is",
        "key": "content.com\\.openclaw\\.finalized_preview",
        "value": true
      },
      { "kind": "event_match", "key": "sender", "pattern": "@bot:example.org" }
    ],
    "actions": [
      "notify",
      { "set_tweak": "sound", "value": "default" },
      { "set_tweak": "highlight", "value": false }
    ]
  }'
```

    執行前請替換：

    - `https://matrix.example.org`：你的主伺服器基底 URL
    - `$USER_ACCESS_TOKEN`：接收者使用者的存取權杖
    - `openclaw-finalized-preview-botname`：每個接收者的每個機器人都必須使用唯一的規則 ID（模式：`openclaw-finalized-preview-<botname>`）
    - `@bot:example.org`：你的 OpenClaw 機器人 MXID，而非接收者的 MXID

  </Step>

  <Step title="驗證">

```bash
curl -sS \
  -H "Authorization: Bearer $USER_ACCESS_TOKEN" \
  "https://matrix.example.org/_matrix/client/v3/pushrules/global/override/openclaw-finalized-preview-botname"
```

接著測試串流回覆。在靜默模式下，聊天室會顯示靜默草稿預覽，並在區塊或對話輪次完成時發出一次通知。

  </Step>
</Steps>

若之後要移除規則，請使用接收者的權杖對相同的規則 URL 執行 `DELETE`。

## 多機器人注意事項

推播規則以 `ruleId` 作為索引鍵：針對同一 ID 重新執行 `PUT`，會更新單一規則。若有多個 OpenClaw 機器人要通知同一位接收者，請為每個機器人建立一條規則，並使用不同的傳送者比對條件。

新定義的使用者 `override` 規則會插入伺服器預設抑制規則之前，因此不需要額外的排序參數。此規則只會影響可就地完成的純文字預覽編輯；媒體回覆、過期預覽的備援處理，以及會啟用 Matrix 提及功能的最終文字，則會改以一般會發出通知的訊息傳送。

## 主伺服器注意事項

<AccordionGroup>
  <Accordion title="Synapse">
    不需要進行特殊的 `homeserver.yaml` 變更。若一般 Matrix 通知已能送達此使用者，主要設定步驟就是使用接收者權杖執行上述 `pushrules` 呼叫。

    若 Synapse 位於反向 Proxy 或工作處理程序之後，請確保 `/_matrix/client/.../pushrules/` 能正確抵達 Synapse。推播傳送由主程序或 `synapse.app.pusher`／已設定的推播器工作處理程序負責，請確保它們運作正常。

    此規則使用 `event_property_is` 推播規則條件（MSC3758，推播規則 v1.10），Synapse 於 2023 年加入此條件。較舊的 Synapse 版本會接受 `PUT pushrules/...` 呼叫，但不會顯示錯誤，也永遠不會符合該條件；若完成預覽編輯時未收到通知，請升級 Synapse。

  </Accordion>

  <Accordion title="Tuwunel">
    流程與 Synapse 相同；最終預覽標記不需要任何 Tuwunel 專用設定。

    若使用者在另一部裝置上處於使用中狀態時通知消失，請檢查是否已啟用 `suppress_push_when_active`。Tuwunel 在 1.4.2（2025 年 9 月）加入此選項；當其中一部裝置處於使用中狀態時，此選項可刻意抑制傳送至其他裝置的推播。

  </Accordion>
</AccordionGroup>

## 相關內容

- [Matrix 頻道設定](/zh-TW/channels/matrix)
- [串流概念](/zh-TW/concepts/streaming)
