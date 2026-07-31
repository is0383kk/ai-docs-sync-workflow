---
read_when:
    - 使用者回報代理程式陷入重複呼叫工具的狀態
    - 你需要控制重複呼叫保護機制
    - 你正在編輯代理程式工具／執行階段政策
    - 在因上下文溢位而重試後，你遇到 `compaction_loop_persisted` 中止錯誤
summary: 如何啟用可偵測重複工具呼叫迴圈的防護機制
title: 工具迴圈偵測
x-i18n:
    generated_at: "2026-07-26T08:46:20Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 79b5aa1d85e02b8cf46a95b3bcebb255178b91456517cab804cce77b8f3b818e
    source_path: tools/loop-detection.md
    workflow: 16
---

OpenClaw 有兩道相互配合的防護機制，可避免重複的工具呼叫模式，
兩者皆設定於 `tools.loopDetection`：

1. **迴圈偵測**（`enabled`）— 預設停用。監控滾動式
   工具呼叫歷程中的重複模式與未知工具重試。
2. **壓縮後防護** — 只要
   `enabled` 未明確設為 `false`，即會啟用。每次壓縮重試後進入戒備狀態；若代理程式在時間窗內重複相同的 `(tool, args, result)` 三元組，
   便會中止執行。

將 `tools.loopDetection.enabled: false` 設為靜默停用這兩道防護機制。

## 為何需要此機制

- 偵測毫無進展的重複序列。
- 偵測高頻率且無結果的迴圈（相同工具、相同輸入、重複
  錯誤）。
- 偵測已知輪詢工具的特定重複呼叫模式。
- 中斷「內容超出上限 -> 壓縮 -> 相同迴圈」的循環，而非任其
  無限執行。

## 設定區塊

全域設定：

```json5
{
  tools: {
    loopDetection: {
      enabled: false, // 滾動歷程偵測器的主開關
    },
  },
}
```

個別代理程式覆寫（選用，位於 `agents.entries.*.tools.loopDetection`）：

```json5
{
  agents: {
    list: [
      {
        id: "safe-runner",
        tools: {
          loopDetection: {
            enabled: true,
          },
        },
      },
    ],
  },
}
```

個別代理程式設定會覆寫全域設定。

### 欄位行為

| 欄位     | 預設值 | 效果                                                                                            |
| --------- | ------- | ------------------------------------------------------------------------------------------------- |
| `enabled` | `false` | 滾動歷程偵測器的主開關。`false` 也會停用壓縮後防護。 |

對於 `exec`，無進展雜湊會比較穩定的命令結果（狀態、
結束代碼、逾時旗標、輸出），並忽略持續時間、PID、工作階段 ID 與工作目錄等
易變的執行階段中繼資料。外送訊息的傳送
結果在雜湊前會移除每次呼叫都會變動的 ID（訊息 ID、檔案 ID、時間戳記），
因此某次「已傳送」結果不會看起來與另一次不同的「已傳送」
結果完全相同。若有執行 ID，則只會在該次執行內評估歷程，
因此排程的心跳偵測週期與全新執行不會繼承
先前執行留下的過時迴圈計數。

## 建議設定

- 對於較小的模型，請設定 `enabled: true`。旗艦模型很少需要滾動歷程偵測，因此可
  將主開關保持為 `false`，同時仍受益於
  壓縮後防護。
- 若要停用所有機制（包括壓縮後防護），請明確設定
  `tools.loopDetection.enabled: false`。

## 壓縮後防護

在內容超出上限後進行壓縮重試時，執行器會針對接下來幾次工具呼叫啟用
短時間窗防護。若代理程式在該時間窗內多次發出相同的
`(toolName, argsHash, resultHash)` 三元組，防護機制便會判定壓縮未能中斷
迴圈，並以 `compaction_loop_persisted` 錯誤中止執行。

此防護受主 `tools.loopDetection.enabled` 旗標控制，但有一項
特殊之處：旗標未設定或為 `true` 時，防護會保持**啟用**；只有
旗標明確設為 `false` 時才會關閉。這是刻意的設計 — 此防護
用於脫離原本會無上限消耗權杖的壓縮迴圈，
因此未進行任何設定的使用者仍可獲得保護。

```json5
{
  tools: {
    loopDetection: {
      // 主開關；設為 false 可連同滾動偵測器一起停用此防護
      enabled: true,
    },
  },
}
```

- 結果仍在變化時，防護絕不會中止執行；只有時間窗內
  位元組完全相同的結果才會觸發。
- 此防護只會在壓縮重試後的緊接階段啟用，不會在一次執行的其他
  時點啟用。

<Note>
  只要主旗標未明確設為 `false`，壓縮後防護就會執行，即使你從未編寫 `tools.loopDetection` 區塊也是如此。若要驗證，請在壓縮事件後立即查看閘道日誌中是否出現 `post-compaction guard armed for N attempts`。
</Note>

## 日誌與預期行為

偵測到迴圈時，OpenClaw 會記錄迴圈事件，並依嚴重程度警告或封鎖
下一個工具週期，在保留正常工具存取能力的同時，防止權杖
失控消耗與系統卡死。

- 會先發出警告。
- 當模式持續超過警告門檻後，便會進行封鎖。
- 達到嚴重門檻時，會封鎖下一個工具週期，並在執行記錄中顯示明確的
  迴圈偵測原因。
- 壓縮後防護會發出 `compaction_loop_persisted` 錯誤，其中會指出
  涉事工具與相同呼叫次數。

## 相關內容

<CardGroup cols={2}>
  <Card title="執行核准" href="/zh-TW/tools/exec-approvals" icon="shield">
    Shell 執行的允許／拒絕政策。
  </Card>
  <Card title="思考層級" href="/zh-TW/tools/thinking" icon="brain">
    推理投入程度與供應商政策的互動。
  </Card>
  <Card title="子代理程式" href="/zh-TW/tools/subagents" icon="users">
    產生隔離的代理程式，以限制失控行為。
  </Card>
  <Card title="設定參考" href="/zh-TW/gateway/config-tools#toolsloopdetection" icon="gear">
    完整的 `tools.loopDetection` 結構描述與合併語意。
  </Card>
</CardGroup>
