---
read_when:
    - 變更輸入指示器的行為或預設值
summary: OpenClaw 何時顯示輸入中指示，以及如何調整其設定
title: 輸入中指示器
x-i18n:
    generated_at: "2026-07-26T08:16:44Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: b3c66d61ea7e3e809b8e88ae2eabb9794f0886b629094753716ed02912843ffc
    source_path: concepts/typing-indicators.md
    workflow: 16
---

執行進行期間，系統會將輸入指示傳送至聊天頻道。使用 `agents.defaults.typingMode` 控制輸入狀態在**何時**開始，並使用 `typingIntervalSeconds` 控制其重新整理**頻率**（保持連線的週期，預設為 6 秒）。

## 預設值

當 `agents.defaults.typingMode` **未設定**時：

- **直接聊天**：模型迴圈開始後立即顯示輸入狀態。
- **含提及的群組聊天**：立即顯示輸入狀態。
- **不含提及的群組聊天**：獲准執行出現使用者可見的活動時，才顯示輸入狀態，例如執行工具框架或產生訊息文字。
- **心跳偵測執行**：若解析出的心跳偵測目標是支援輸入狀態的聊天，且未停用輸入狀態，則在心跳偵測執行開始時顯示輸入狀態。

## 模式

將 `agents.defaults.typingMode` 設為以下其中一項：

- `never` - 永遠不顯示輸入指示。
- `instant` - **模型迴圈一開始**就顯示輸入狀態，即使該次執行最後只傳回靜默回覆權杖。
- `thinking` - 在出現**第一個推理增量**時，或輪次獲接受後開始執行工具框架時，顯示輸入狀態。
- `message` - 在出現**第一個使用者可見的回覆活動**時顯示輸入狀態，例如執行工具框架或出現非靜默的文字增量。`NO_REPLY` 等靜默回覆權杖不算文字活動。

依「觸發時間由早至晚」排序：`never` -> `message`/`thinking` -> `instant`。

## 設定

設定代理程式層級的預設值：

```json5
{
  agents: {
    defaults: {
      typingMode: "thinking",
      typingIntervalSeconds: 6,
    },
  },
}
```

覆寫單一代理程式的政策：

```json5
{
  agents: {
    entries: {
      support: {
        typingMode: "message",
      },
    },
  },
}
```

## 注意事項

- `message` 模式不會因靜默回覆權杖而開始顯示輸入狀態，但在任何助理文字可用之前，執行中的活動仍可能觸發輸入狀態。
- `thinking` 仍會回應串流推理（`reasoningLevel: "stream"`），也可在推理增量抵達前，因執行中的活動而開始顯示輸入狀態。
- 心跳偵測輸入狀態是針對解析後傳遞目標的存活訊號。它會在心跳偵測執行開始時啟動，而不會依循 `message` 或 `thinking` 的串流時序。設定 `typingMode: "never"` 可將其停用。
- 當心跳偵測目標為 `"none"`、無法解析目標、已停用該心跳偵測的聊天傳遞，或頻道不支援輸入狀態時，心跳偵測不會顯示輸入狀態。
- `agents.defaults.typingIntervalSeconds` 控制每個代理程式的**重新整理週期**，而非開始時間。預設值：6 秒。

## 相關內容

<CardGroup cols={2}>
  <Card title="上線狀態" href="/zh-TW/concepts/presence" icon="signal">
    閘道如何追蹤已連線的用戶端，以供控制介面的「裝置」頁面和 macOS「執行個體」分頁使用。
  </Card>
  <Card title="串流與分塊" href="/zh-TW/concepts/streaming" icon="bars-staggered">
    傳出串流行為、區塊邊界，以及頻道專屬的傳遞方式。
  </Card>
</CardGroup>
