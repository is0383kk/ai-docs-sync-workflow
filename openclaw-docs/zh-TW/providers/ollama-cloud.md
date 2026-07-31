---
read_when:
    - 你想要在沒有本機 Ollama 伺服器的情況下使用託管的 Ollama 模型
    - 你需要 ollama-cloud 提供者 ID、金鑰或端點
summary: 直接在 OpenClaw 中使用 Ollama Cloud
title: Ollama Cloud
x-i18n:
    generated_at: "2026-07-26T08:10:41Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 966e5237e37134cef109979079db390e9844714001e921e7976dc8ca7f58bcc4
    source_path: providers/ollama-cloud.md
    workflow: 16
---

Ollama Cloud 是 Ollama 的託管模型 API。`ollama-cloud` 提供者會透過 Ollama 原生的 `/api/chat` API，
直接呼叫 `https://ollama.com`，不需要本機 Ollama 伺服器，也不需要登入雲端模式的本機 Ollama 應用程式。請使用
`ollama-cloud/kimi-k2.6` 之類的模型參照。

OpenClaw 將 `ollama-cloud` 註冊為獨立的提供者 ID，讓僅限雲端的
認證資訊、即時目錄探索與模型選擇不會和本機 `ollama` 主機混在一起。如需本機 Ollama、雲端加本機的混合路由、
嵌入及自訂主機詳細資訊，請參閱 [Ollama](/zh-TW/providers/ollama)。

## 設定

在 [ollama.com/settings/keys](https://ollama.com/settings/keys) 建立 Ollama Cloud API 金鑰，然後執行：

```bash
openclaw onboard --auth-choice ollama-cloud
```

或設定：

```bash
export OLLAMA_API_KEY="<your-ollama-cloud-api-key>" # pragma: allowlist secret
```

非互動式初始設定可直接接受金鑰：

```bash
openclaw onboard --auth-choice ollama-cloud --ollama-cloud-api-key "<key>"
```

初始設定會將預設模型設為 `ollama-cloud/kimi-k2.5:cloud`。

## 預設值

- 提供者：`ollama-cloud`
- 基底 URL：`https://ollama.com`
- 環境變數：`OLLAMA_API_KEY`
- API 樣式：Ollama 原生 `/api/chat`
- 初始設定預設模型：`ollama-cloud/kimi-k2.5:cloud`

## 何時選擇 Ollama Cloud

- 你想使用託管的 Ollama 模型，而不在本機執行 `ollama serve`。
- 你想使用與 OpenClaw 本機 Ollama 相同的原生 Ollama 聊天 API 格式，
  但將其指向 `https://ollama.com`。
- 你想要一條簡單的雲端路徑，以使用 Ollama 託管目錄中已有的
  模型。
- 你不需要在本機拉取模型、控制本機 GPU，或進行僅限區域網路的推論。

若你想透過已登入的 Ollama 主機使用僅限本機或
雲端加本機的路由，請改用 [Ollama](/zh-TW/providers/ollama)。若你需要 `/v1/chat/completions`
語意或提供者特定的 OpenAI 風格功能，請改用
OpenAI 相容提供者。

## 模型

此提供者需要 API 金鑰；沒有金鑰時會保持停用狀態。提供金鑰後，
OpenClaw 會從託管目錄即時探索 Ollama Cloud 模型：

```bash
openclaw models list --provider ollama-cloud
openclaw models set ollama-cloud/kimi-k2.6
```

即時目錄中的託管 ID 包括 `deepseek-v4-flash`、`glm-5`、
`gpt-oss:20b`、`kimi-k2.6` 和 `minimax-m2.7`。即時探索未傳回
任何內容時，OpenClaw 會回退至內建項目 `kimi-k2.5:cloud`、
`minimax-m2.7:cloud`、`glm-5.1:cloud` 和 `glm-5.2:cloud`。

模型 ID 是雲端目錄 ID，而非本機拉取名稱。如果某個模型名稱可在
本機 Ollama 主機上使用，但未出現在託管目錄中，請改用 `ollama`
提供者搭配該本機主機。

## 即時測試

若要進行 Ollama Cloud API 金鑰的冒煙測試，請將 Ollama 即時測試指向託管
端點，並從目前的目錄中選擇模型：

```bash
export OLLAMA_API_KEY="<your-ollama-cloud-api-key>" # pragma: allowlist secret

OPENCLAW_LIVE_TEST=1 \
OPENCLAW_LIVE_OLLAMA=1 \
OPENCLAW_LIVE_OLLAMA_BASE_URL=https://ollama.com \
OPENCLAW_LIVE_OLLAMA_MODEL=kimi-k2.6 \
pnpm test:live -- extensions/ollama/ollama.live.test.ts
```

雲端冒煙測試會執行文字、原生串流和網頁搜尋；設定
`OPENCLAW_LIVE_OLLAMA_WEB_SEARCH=0` 可略過網頁搜尋。對於 `https://ollama.com`，測試預設會略過
嵌入，因為 Ollama Cloud API 金鑰可能未獲授權使用
`/api/embed`；若要強制執行，請設定 `OPENCLAW_LIVE_OLLAMA_EMBEDDINGS=1`。

## 疑難排解

- `Ollama Cloud requires an API key` / `Set OLLAMA_API_KEY` 錯誤：請提供
  真實的雲端 API 金鑰。本機 `ollama-local` 標記僅適用於本機或
  私有 Ollama 主機。
- 未知模型錯誤：執行 `openclaw models list --provider ollama-cloud`，並
  完整複製託管模型 ID。
- 自訂 Ollama 主機上的工具呼叫或原始 JSON 問題：請檢查是否
  不慎使用了 OpenAI 相容的 `/v1` URL。Ollama 路由應使用
  不含 `/v1` 後綴的原生基底 URL。

## 相關內容

- [Ollama](/zh-TW/providers/ollama)
- [模型提供者](/zh-TW/concepts/model-providers)
- [所有提供者](/zh-TW/providers/index)
