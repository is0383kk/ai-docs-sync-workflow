---
read_when:
    - 你想要能跨工作階段與頻道使用的持久記憶體
    - 你想要由 AI 驅動的回憶與使用者建模
summary: 透過 Honcho 外掛實現 AI 原生的跨工作階段記憶功能
title: Honcho 記憶體
x-i18n:
    generated_at: "2026-07-26T07:49:06Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: fadcf6d8e2505ab4fe6a81340695b7c8fee49c3cb4889665af13389941619117
    source_path: concepts/memory-honcho.md
    workflow: 16
---

[Honcho](https://honcho.dev) 透過外部外掛為 OpenClaw 新增 AI 原生記憶。它會將對話持久儲存至專用服務，並隨時間建立使用者與代理模型，讓你的代理取得跨工作階段的情境，範圍超越工作區 Markdown 檔案。

## 功能

- **跨工作階段記憶** - 每輪對話後都會持久保存，因此即使重設工作階段、進行壓縮或切換頻道，情境仍會延續。
- **使用者建模** - Honcho 會為每位使用者維護一份設定檔（偏好、事實、溝通風格），也會為代理維護設定檔（個性、習得行為）。
- **語意搜尋** - 搜尋過往對話中的觀察結果，而不僅限於目前的工作階段。
- **多代理感知** - 父代理會自動追蹤所產生的子代理，並在子工作階段中將父代理新增為觀察者。

## 可用工具

Honcho 會註冊代理可在對話期間使用的工具：

**資料擷取（快速，不呼叫 LLM）：**

| 工具                        | 功能                                           |
| --------------------------- | ------------------------------------------------------ |
| `honcho_context`            | 跨工作階段的完整使用者表徵               |
| `honcho_search_conclusions` | 對已儲存的結論進行語意搜尋                |
| `honcho_search_messages`    | 尋找跨工作階段的訊息（依傳送者、日期篩選） |
| `honcho_session`            | 目前工作階段的歷程記錄與摘要                    |

**問答（由 LLM 驅動）：**

| 工具         | 功能                                                              |
| ------------ | ------------------------------------------------------------------------- |
| `honcho_ask` | 詢問使用者相關資訊。使用 `depth='quick'` 查詢事實，使用 `'thorough'` 進行綜合分析 |

## 開始使用

安裝外掛並執行設定：

```bash
openclaw plugins install @honcho-ai/openclaw-honcho
openclaw honcho setup
openclaw gateway --force
```

設定命令會提示你輸入 API 認證資訊、寫入設定，並可選擇遷移現有的工作區記憶檔案。

<Info>
Honcho 可以完全在本機執行（自行託管），也可以透過 `api.honcho.dev` 的受管理 API 執行。自行託管選項不需要任何外部相依項目。
</Info>

## 設定

設定位於 `plugins.entries["openclaw-honcho"].config` 之下：

```json5
{
  plugins: {
    entries: {
      "openclaw-honcho": {
        config: {
          apiKey: "your-api-key", // 自行託管時省略
          workspaceId: "openclaw", // 記憶隔離
          baseUrl: "https://api.honcho.dev",
        },
      },
    },
  },
}
```

若使用自行託管的執行個體，請將 `baseUrl` 指向你的本機伺服器（例如 `http://localhost:8000`），並省略 API 金鑰。

## 遷移現有記憶

如果你已有工作區記憶檔案（`USER.md`、`MEMORY.md`、`IDENTITY.md`、`memory/`、`canvas/`），`openclaw honcho setup` 會偵測這些檔案並提供遷移選項。

<Info>
遷移不具破壞性——檔案會上傳至 Honcho，原始檔案絕不會遭到刪除或移動。
</Info>

## 運作方式

每次 AI 回合結束後，對話都會持久儲存至 Honcho。系統會觀察使用者和代理雙方的訊息，讓 Honcho 能夠隨時間建立並精進其模型。

對話期間，Honcho 工具會在 OpenClaw 的 `before_prompt_build` 外掛鉤點執行時查詢服務，並在模型看到提示之前注入相關情境。

## Honcho 與內建記憶的比較

|                   | 內建 / QMD                | Honcho                              |
| ----------------- | ---------------------------- | ----------------------------------- |
| **儲存空間**       | 工作區 Markdown 檔案     | 專用服務（本機或託管） |
| **跨工作階段** | 透過記憶檔案             | 自動且內建                 |
| **使用者建模** | 手動（寫入 MEMORY.md）  | 自動建立設定檔                  |
| **搜尋**        | 向量 + 關鍵字（混合）    | 對觀察結果進行語意搜尋          |
| **多代理**   | 不追蹤                  | 父子代理感知              |
| **相依項目**  | 無（內建）或 QMD 二進位檔 | 安裝外掛                      |

Honcho 與內建記憶系統可以搭配使用。設定 QMD 後，會提供額外工具，讓你能在使用 Honcho 跨工作階段記憶的同時搜尋本機 Markdown 檔案。

## 命令列介面命令

```bash
openclaw honcho setup                        # 設定 API 金鑰並遷移檔案
openclaw honcho status                       # 檢查連線狀態
openclaw honcho ask <question>               # 向 Honcho 查詢使用者相關資訊
openclaw honcho search <query> [-k N] [-d D] # 對記憶進行語意搜尋
```

## 延伸閱讀

- [外掛原始碼](https://github.com/plastic-labs/openclaw-honcho)
- [Honcho 文件](https://docs.honcho.dev)
- [Honcho OpenClaw 整合指南](https://docs.honcho.dev/v3/guides/integrations/openclaw)

## 相關內容

- [記憶概覽](/zh-TW/concepts/memory)
- [內建記憶引擎](/zh-TW/concepts/memory-builtin)
- [QMD 記憶引擎](/zh-TW/concepts/memory-qmd)
- [情境引擎](/zh-TW/concepts/context-engine)
