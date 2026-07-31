---
read_when:
    - 在代理程式已執行時使用 /steer 或 /tell
    - 比較 `/steer` 與 `/queue` 模式
    - 決定要引導目前的執行，還是 ACP 工作階段
sidebarTitle: Steer
summary: 在不變更佇列模式的情況下引導進行中的執行作業
title: 引導
x-i18n:
    generated_at: "2026-07-26T08:42:36Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: d420e14982d52520e415103ffa6d86923fad6f13c43ff7741ebbd8dde0d0073f
    source_path: tools/steer.md
    workflow: 16
---

`/steer` 會先嘗試將指引傳送給已在執行中的執行作業。這適用於
“趁此執行作業仍在運作時進行調整”的情況。如果目前的執行環境
無法接受引導，OpenClaw 會改將訊息作為一般提示傳送，
而不會捨棄訊息。

## 目前的工作階段

使用頂層的 `/steer`，以目前工作階段中正在執行的執行作業為目標：

```text
/steer 優先採用較小的修補程式，並讓測試保持聚焦
/tell 在進行下一次工具呼叫前先摘要
```

行為：

- 僅以目前工作階段中正在執行的執行作業為目標。
- 不受工作階段的 `/queue` 模式影響，獨立運作。
- 當工作階段閒置，或正在執行的執行作業無法接受引導時，
  會以相同訊息開始一般回合。
- 使用正在執行之執行環境的引導路徑，因此模型會在
  下一個受支援的執行環境邊界看到該指引。

## 引導與佇列的比較

當執行作業進行中收到一般傳入訊息時，`/queue steer` 會嘗試使用這些訊息引導正在執行的執行作業。`/steer <message>` 則是明確命令，
無論儲存的 `/queue` 設定為何，都會嘗試在下一個
受支援的執行環境邊界，將該命令的訊息注入正在執行的執行作業。當
無法注入時，系統會移除命令前綴，而 `<message>`
會繼續作為一般提示。

明確的 `/steer`（以及 `/tell`）命令由閘道支援。在
`openclaw chat` 或 `openclaw tui --local` 中，選取 `/queue steer`，並將
指引作為一般訊息傳送；內嵌的執行環境會套用相同的引導
原則，而不會轉送閘道命令。

使用方式：

- 想要立即引導正在執行的執行作業時，使用 `/steer <message>`。
- 想讓日後的一般訊息預設引導正在執行的執行作業時，使用 `/queue steer`。
- 日後的一般訊息應等待稍後的回合，而非引導正在執行的執行作業時，
  使用 `/queue collect` 或 `/queue followup`。
- 最新訊息應取代正在執行的執行作業，而非引導該執行作業時，使用 `/queue interrupt`。

如需佇列模式和引導邊界的相關資訊，請參閱[命令佇列](/zh-TW/concepts/queue)和
[引導佇列](/zh-TW/concepts/queue-steering)。

## 子代理程式

頂層的 `/steer` 以目前工作階段中正在執行的執行作業為目標。子代理程式會向
其父工作階段／要求者工作階段回報；`/subagents` 僅供查看。

## ACP 工作階段

當目標是 ACP 控制框架工作階段時，使用 `/acp steer`：

```text
/acp steer --session agent:main:acp:codex 縮小重現步驟
```

如需 ACP 工作階段選擇和執行環境行為的相關資訊，請參閱
[ACP 代理程式](/zh-TW/tools/acp-agents)。

## 相關內容

- [斜線命令](/zh-TW/tools/slash-commands)
- [命令佇列](/zh-TW/concepts/queue)
- [引導佇列](/zh-TW/concepts/queue-steering)
- [子代理程式](/zh-TW/tools/subagents)
