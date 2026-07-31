---
read_when:
    - 你想在其他應用程式中重複使用 OpenClaw 的模型傳輸層
    - 你正在變更 `packages/ai` 或 AI 傳輸主機連接埠
    - 你正在審查 OpenClaw 發布版本除了根套件之外，還會將哪些內容發布至 npm
summary: '@openclaw/ai npm 套件：可重複使用的模型傳輸、隔離執行環境與主機政策連接埠'
title: '@openclaw/ai 套件'
x-i18n:
    generated_at: "2026-07-26T08:35:59Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 610057caae0a9bbf9f74074cda75fc40c0b9aa9d3441f8263151f08f1a3f35a8
    source_path: reference/openclaw-ai.md
    workflow: 16
---

`@openclaw/ai` 是 OpenClaw 模型執行層可供發布的函式庫形式：
提供者中立的訊息／工具／串流合約、驗證、診斷、事件串流、隔離的執行階段登錄檔，
以及八個內建 API 家族的延遲載入轉接器（Anthropic Messages、OpenAI Completions、OpenAI
Responses、Azure OpenAI Responses、ChatGPT/Codex Responses、Google Generative
AI、Google Vertex、Mistral Conversations）。

它會在每次發行時與根 `openclaw` 套件一同發布，固定為
相同版本，並具有自己的 `npm-shrinkwrap.json`，因此其遞移
相依性樹會在安裝時鎖定。安裝 `openclaw` 會自動安裝
相符的 `@openclaw/ai`；函式庫使用者可以直接相依於它，
無需任何 OpenClaw 應用程式程式碼。

## 快速開始

```js
import { createLlmRuntime } from "@openclaw/ai";
import { registerBuiltInApiProviders } from "@openclaw/ai/providers";

const runtime = createLlmRuntime();
registerBuiltInApiProviders(runtime.registry);

const stream = runtime.streamSimple(model, { messages }, { apiKey });
for await (const event of stream) {
  if (event.type === "text_delta") process.stdout.write(event.delta);
}
const result = await stream.result();
```

可執行版本位於儲存庫中的 `examples/ai-chat`。

## 設計合約

- **預設以執行個體為範圍。** 匯入套件不會在全域
登錄任何內容。`createApiRegistry()`／`createLlmRuntime()` 會傳回隔離的
執行個體；`registerBuiltInApiProviders(registry)` 讓一個登錄檔選擇使用
內建傳輸。提供者 SDK 模組會在首次使用時延遲載入。
- **主機政策透過注入提供，而非內建於套件中。** 請求 fetch 防護（例如
SSRF 政策）、工具結果重播文字的機密遮蔽、OpenAI
嚴格工具預設值，以及診斷記錄，都是使用 `configureAiTransportHost`
設定的 `AiTransportHost` 連接埠。函式庫預設值不會執行任何操作；
OpenClaw 會在其串流門面中安裝實際實作。
- **單一事件串流識別。** `@openclaw/ai/event-stream` 是由 OpenClaw 核心、
agent-core 與外部使用者共用的標準
`EventStream` 建構函式。
- **`internal/*` 子路徑並非 API。** 它們僅供 OpenClaw
應用程式本身使用，不提供任何語意化版本保證。
- 提供者 ID、認證資訊、模型目錄、重試與容錯移轉仍屬於
應用程式關注事項。OpenClaw 會在此套件外層加入這些功能；函式庫
使用者則直接提供 `Model` 物件與選項。

## 子路徑匯出

| 子路徑          | 內容                                                                       |
| ---------------- | ------------------------------------------------------------------------------ |
| `.`              | 合約、`createApiRegistry`、`createLlmRuntime`、`configureAiTransportHost` |
| `./providers`    | `registerBuiltInApiProviders`、`resetApiProviders`                             |
| `./types`        | 模型／訊息／工具／串流型別                                                |
| `./validation`   | 工具引數驗證                                                       |
| `./diagnostics`  | 診斷合約                                                          |
| `./event-stream` | 共用 `EventStream` 實作                                            |
| `./internal/*`   | OpenClaw 內部使用，不提供語意化版本保證                                         |
