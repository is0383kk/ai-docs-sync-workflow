---
read_when:
    - 你想了解預設的記憶後端
    - 你想要設定嵌入向量提供者或混合式搜尋
summary: 具備關鍵字、向量與混合搜尋功能的預設 SQLite 記憶後端
title: 內建記憶引擎
x-i18n:
    generated_at: "2026-07-26T07:39:41Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: c3efb6f1449d9b55717b3c117444ba7d4519d0111b842b48790ad85551511433
    source_path: concepts/memory-builtin.md
    workflow: 16
---

內建引擎是預設的記憶後端。它會將你的記憶索引儲存在每個代理程式各自的 SQLite 資料庫中，無需額外相依套件即可開始使用。

## 提供的功能

- **關鍵字搜尋**：透過 FTS5 全文索引（BM25 評分）。
- **向量搜尋**：透過任何支援的提供者所產生的嵌入向量。
- **混合搜尋**：結合兩者以獲得最佳結果。
- **CJK 支援**：透過三元組斷詞支援中文、日文與韓文。
- **sqlite-vec 加速**：用於資料庫內向量查詢（選用）。

## 開始使用

依預設，內建引擎使用 OpenAI 嵌入向量。如果已設定 `OPENAI_API_KEY` 或
`models.providers.openai.apiKey`，向量搜尋無需額外的記憶設定即可運作。

若要明確設定提供者：

```json5
{
  memory: {
    search: {
      provider: "openai",
    },
  },
}
```

如果沒有嵌入向量提供者，則只能使用關鍵字搜尋。

若要強制使用本機 GGUF 嵌入向量，請安裝官方 llama.cpp 提供者外掛，然後將
`local.modelPath` 指向 GGUF 檔案：

```bash
openclaw plugins install @openclaw/llama-cpp-provider
```

```json5
{
  memory: {
    search: {
      provider: "local",
      fallback: "none",
      local: {
        modelPath: "~/.node-llama-cpp/models/embeddinggemma-300m-qat-Q8_0.gguf",
      },
    },
  },
}
```

## 支援的嵌入向量提供者

| 提供者            | ID                  | 備註                                  |
| ----------------- | ------------------- | ------------------------------------- |
| Bedrock           | `bedrock`           | 使用 AWS 認證資訊鏈                   |
| DeepInfra         | `deepinfra`         | 預設：`BAAI/bge-m3`              |
| Gemini            | `gemini`            | 支援多模態（影像 + 音訊）             |
| GitHub Copilot    | `github-copilot`    | 使用你的 Copilot 訂閱                 |
| LM Studio         | `lmstudio`          | 本機／自行託管                        |
| 本機              | `local`             | `@openclaw/llama-cpp-provider`      |
| Mistral           | `mistral`           |                                       |
| Ollama            | `ollama`            | 本機／自行託管                        |
| OpenAI            | `openai`            | 預設：`text-embedding-3-small`   |
| OpenAI 相容       | `openai-compatible` | 通用 `/v1/embeddings` 端點   |
| Voyage            | `voyage`            |                                       |

設定 `memory.search.provider` 即可改用 OpenAI 以外的提供者。

## 索引的運作方式

OpenClaw 會將 `MEMORY.md` 和 `memory/*.md` 建立為區塊索引（預設每個區塊 400 個權杖，重疊
80 個權杖），並將其儲存在每個代理程式各自的 SQLite 資料庫中。

- **索引位置：**所屬代理程式的資料庫，位於
  `~/.openclaw/agents/<agentId>/agent/openclaw-agent.sqlite`
- **儲存空間維護：**透過定期檢查點與關閉時檢查點，限制 SQLite WAL 附屬檔案的大小。
- **檔案監看：**記憶檔案的變更會觸發防抖動重新索引
  （預設 1.5 秒）。
- **自動重新索引：**嵌入向量提供者、模型、分塊設定、已設定的來源或範圍變更時，索引會自動重建。
- **隨選重新索引：**`openclaw memory index --force`

<Info>
你也可以使用 `memory.search.extraPaths`，為工作區外的 Markdown 檔案建立索引。請參閱
[設定參考](/zh-TW/reference/memory-config#additional-memory-paths)。
</Info>

## 適用時機

內建引擎是大多數使用者的正確選擇：

- 無需額外相依套件，開箱即用。
- 能妥善處理關鍵字與向量搜尋。
- 支援所有嵌入向量提供者。
- 混合搜尋結合了兩種擷取方式的優點。

如果你需要重新排序、查詢擴展，或想為工作區外的目錄建立索引，請考慮切換至
[QMD](/zh-TW/concepts/memory-qmd)。

如果你想要具備自動使用者建模的跨工作階段記憶，請考慮使用
[Honcho](/zh-TW/concepts/memory-honcho)。

## 疑難排解

**記憶搜尋已停用？** 檢查 `openclaw memory status`。如果未偵測到提供者，請明確設定一個提供者或新增 API 金鑰。

**未偵測到本機提供者？** 確認本機路徑存在，並執行：

```bash
openclaw memory status --deep --agent main
openclaw memory index --force --agent main
```

獨立的命令列介面命令和閘道都使用相同的 `local` 提供者 ID。
若要使用本機嵌入向量，請設定 `memory.search.provider: "local"`。

**結果過期？** 執行 `openclaw memory index --force` 以重建索引。在罕見的極端情況下，監看器可能會漏掉變更。

**sqlite-vec 無法載入？** OpenClaw 會自動改用程序內餘弦相似度。`openclaw memory status --deep` 會分別回報本機向量儲存區與嵌入向量提供者，因此 `Vector store:
unavailable` 指向 sqlite-vec 載入狀態，而 `Embeddings: unavailable`
則指向提供者／驗證或模型就緒狀態。請檢查日誌中的特定載入錯誤。

## 設定

如需瞭解嵌入向量提供者設定、混合搜尋調校（權重、MMR、時間衰減）、批次索引、多模態記憶、sqlite-vec、額外路徑及所有其他設定選項，請參閱
[記憶設定參考](/zh-TW/reference/memory-config)。

## 相關內容

- [記憶概覽](/zh-TW/concepts/memory)
- [記憶搜尋](/zh-TW/concepts/memory-search)
- [主動記憶](/zh-TW/concepts/active-memory)
