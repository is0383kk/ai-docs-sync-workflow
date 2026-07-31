---
read_when:
    - 你想要減少工具輸出造成的上下文增長
    - 你想瞭解 Anthropic 提示詞快取最佳化
summary: 修剪舊的工具結果，以精簡內容脈絡並提升快取效率
title: 工作階段修剪
x-i18n:
    generated_at: "2026-07-26T07:40:17Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: dd5cb4582cb8d9d7265213abe1f5b5893634882b9f8b3ce1deef746293dd07db
    source_path: concepts/session-pruning.md
    workflow: 16
---

工作階段修剪會在每次呼叫 LLM 前，從上下文中移除**舊的工具結果**。它能減少累積的工具輸出（執行結果、檔案讀取結果、搜尋結果）造成的上下文膨脹，而不會改寫一般對話文字。

<Info>
修剪僅在記憶體中進行，不會修改磁碟上的工作階段逐字記錄。你的完整歷史記錄一律會保留。
</Info>

## 為何重要

長時間的工作階段會累積工具輸出，使上下文視窗膨脹。這會增加成本，並可能迫使系統比必要時間更早進行[壓縮](/zh-TW/concepts/compaction)。

修剪對 **Anthropic 提示快取**尤其有價值。快取 TTL 到期後，下一個請求會重新快取完整提示。修剪可減少快取寫入大小，直接降低成本。

## 運作方式

修剪以 `cache-ttl` 模式執行，並同時受到時間檢查與上下文大小檢查的限制：

1. 等待快取 TTL 到期（手動設定時預設為 5 分鐘；Anthropic 的自動預設值請參閱[智慧型預設值](#smart-defaults)）。TTL 到期前會完全略過修剪，以保留相鄰輪次對提示快取的重複使用。
2. TTL 到期後，根據模型的上下文視窗估算上下文總大小。如果比例低於 `softTrimRatio`（預設為 0.3），則略過修剪並讓 TTL 計時繼續運作。
3. 對超過該比例的過大工具結果進行**軟修剪**：保留開頭與結尾（預設各 1500 個字元，合計上限為 4000 個字元），並在中間插入 `...`。
4. 如果比例仍等於或高於 `hardClearRatio`（預設為 0.5），且仍有至少 `minPrunableToolChars`（預設為 50,000）的可修剪工具內容，則**完全清除**這些結果：以預留位置取代其內容（預設為 `[Old tool result content cleared]`）。
5. 僅在修剪確實變更上下文時重設 TTL 計時，讓後續請求可重複使用新的快取。

無論臨界值為何，都會套用兩項安全規則：最近的 `keepLastAssistants` 個助理輪次（預設為 3）絕不會被修剪，而且工作階段第一則使用者訊息之前的內容也絕不會被修剪（用以保護 `SOUL.md`/`USER.md` 等啟動階段的讀取內容）。

只有 `toolResult` 訊息符合修剪資格；一般對話文字不會受到影響。使用 `agents.defaults.contextPruning.tools.{allow,deny}` 限定哪些工具名稱可被修剪。

## 舊版影像清理

OpenClaw 也會為歷史記錄中保存原始影像區塊或提示載入媒體標記的工作階段，建立獨立且具冪等性的重播檢視。

- 它會逐位元組保留**最近 3 個已完成輪次**，使最近後續互動的提示快取前綴維持穩定。此數量包含所有已完成輪次，而不僅是含有影像的輪次，因此純文字輪次也會占用此視窗。
- 在重播檢視中，來自 `user` 或 `toolResult` 歷史記錄、較舊且已處理的影像區塊會替換為 `[image data removed - already processed by model]`。
- 較舊的文字媒體參照（例如 `[media attached: ...]`、`[Image: source: ...]` 和 `media://inbound/...`）會替換為 `[media reference removed - already processed by model]`。目前輪次的附件標記會保持不變，讓視覺模型仍可載入新的影像。
- 原始工作階段逐字記錄不會被改寫，因此歷史記錄檢視器仍可呈現原始訊息項目及其影像。
- 這與上述一般快取 TTL 修剪是分開的機制。其用途是避免重複的影像承載資料或過時的媒體參照，在後續輪次破壞提示快取。

## 智慧型預設值

內建的 Anthropic 外掛第一次解析 Anthropic（或 Claude 命令列介面）認證設定檔時，會自動設定修剪與心跳偵測頻率，但只會設定你尚未明確指定的欄位：

| 認證模式                                 | `contextPruning.mode` | `contextPruning.ttl` | `heartbeat.every` |
| ---------------------------------------- | --------------------- | -------------------- | ----------------- |
| OAuth/權杖（包括重複使用 Claude 命令列介面） | `cache-ttl`           | `1h`                 | `1h`              |
| API 金鑰                                 | `cache-ttl`           | `1h`                 | `30m`             |

如果你自行設定 `agents.defaults.contextPruning.mode` 或 `agents.defaults.heartbeat.every`，OpenClaw 不會覆寫它們。此自動預設值僅會套用於 Anthropic 系列認證；除非你另行設定，否則其他提供者的修剪設定為 `off`。

## 啟用或停用

對非 Anthropic 提供者而言，修剪預設為停用。若要啟用：

```json5
{
  agents: {
    defaults: {
      contextPruning: { mode: "cache-ttl", ttl: "5m" },
    },
  },
}
```

若要停用：設定 `mode: "off"`。

## 修剪與壓縮的比較

|            | 修剪               | 壓縮                 |
| ---------- | ------------------ | ----------------------- |
| **內容**   | 修剪工具結果       | 摘要對話內容           |
| **是否儲存？** | 否（每個請求個別處理） | 是（儲存於逐字記錄中） |
| **範圍**   | 僅工具結果         | 整段對話               |

兩者相輔相成：修剪可在壓縮週期之間保持工具輸出精簡。

## 延伸閱讀

- [壓縮](/zh-TW/concepts/compaction)：透過摘要縮減上下文
- [閘道設定](/zh-TW/gateway/configuration)：所有修剪設定選項（`contextPruning.*`）

## 相關內容

- [工作階段管理](/zh-TW/concepts/session)
- [工作階段工具](/zh-TW/concepts/session-tool)
- [上下文引擎](/zh-TW/concepts/context-engine)
