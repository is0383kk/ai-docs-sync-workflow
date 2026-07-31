---
read_when:
    - 你想要建立語意記憶索引或搜尋語意記憶
    - 你正在偵錯記憶可用性或索引問題
    - 你想將回想起的短期記憶提升為 `MEMORY.md`
summary: '`openclaw memory` 的命令列介面參考（status/index/search/promote/promote-explain/rem-harness/rem-backfill）'
title: 記憶
x-i18n:
    generated_at: "2026-07-26T07:46:28Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 6354745f8622ee80345325fa6f3e7d6c5f280cb63b9cdb100a766cf9e300af59
    source_path: cli/memory.md
    workflow: 16
---

# `openclaw memory`

管理語意記憶的索引、搜尋，以及提升至 `MEMORY.md`。
由隨附的 `memory-core` 外掛提供，當
`plugins.slots.memory` 選取 `memory-core`（預設值）時可用。其他記憶
外掛會公開各自的命令列介面命名空間。

相關內容：[記憶](/zh-TW/concepts/memory)概念、[夢境整理](/zh-TW/concepts/dreaming)、
[記憶設定參考](/zh-TW/reference/memory-config)、[記憶 Wiki](/zh-TW/plugins/memory-wiki)、
[Wiki](/zh-TW/cli/wiki)、[外掛](/zh-TW/tools/plugin)。

## `memory status`

```bash
openclaw memory status [--agent <id>] [--deep] [--index] [--fix] [--json] [--verbose]
```

未指定 `--agent` 時，會對 `agents.entries` 中的每個代理執行；若未設定代理清單，
則改用預設代理。

| 旗標        | 效果                                                                                                                                                                                                                                                                                                    |
| ----------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `--deep`    | 探測向量儲存區、嵌入提供者與語意搜尋是否就緒（會產生額外的提供者呼叫）。一般的 `memory status` 會保持快速並略過此探測；向量／語意狀態未知表示未經探測。QMD 詞彙 `searchMode: "search"` 一律略過語意向量探測，即使搭配 `--deep` 也是如此。 |
| `--index`   | 若儲存區處於髒狀態，則重新建立索引。包含 `--deep`。                                                                                                                                                                                                                                                          |
| `--fix`     | 修復過時的回憶鎖定，並正規化提升中繼資料。                                                                                                                                                                                                                                               |
| `--json`    | 輸出 JSON。                                                                                                                                                                                                                                                                                               |
| `--verbose` | 輸出各階段的詳細記錄。                                                                                                                                                                                                                                                                             |

如果 `Dreaming` 行即使搭配 `dreaming.enabled: true` 仍維持 `off`，或
排程掃描似乎從未執行，受管理的夢境整理排程會依賴
預設代理的心跳偵測觸發，以啟動協調。排程詳情請參閱
[夢境整理](/zh-TW/concepts/dreaming)。

狀態也會列出來自 `memory.search.extraPaths` 的任何額外搜尋路徑。

## `memory index`

```bash
openclaw memory index [--agent <id>] [--force] [--verbose]
```

代理範圍與 `status` 相同。`--force` 會執行完整重新索引，而非
增量索引。`--verbose` 會先輸出各代理的提供者、模型、來源與
額外路徑詳細資訊，再顯示索引進度。

## `memory search`

```bash
openclaw memory search [query] [--query <text>] [--agent <id>] [--max-results <n>] [--min-score <n>] [--json]
```

- 查詢：位置引數 `[query]` 或 `--query <text>`。若兩者皆設定，以 `--query`
  為準。若皆未設定，命令會發生錯誤。
- `--agent <id>`：預設使用預設代理（而非完整代理清單）。
- `--max-results <n>`：限制結果數量（正整數）。
- `--min-score <n>`：篩除分數低於此值的相符項目。

## `memory promote`

對來自 `memory/YYYY-MM-DD.md` 的短期候選項目進行排名，並可選擇將
排名最高的項目附加至 `MEMORY.md`。

```bash
openclaw memory promote [--agent <id>] [--limit <n>] [--min-score <n>] \
  [--min-recall-count <n>] [--min-unique-queries <n>] [--apply] [--include-promoted] [--json]
```

| 旗標                       | 預設值      | 效果                                                            |
| -------------------------- | ------------ | ----------------------------------------------------------------- |
| `--limit <n>`              |              | 可傳回／套用的候選項目上限。                                   |
| `--min-score <n>`          | `0.75`       | 加權提升分數下限。                                 |
| `--min-recall-count <n>`   | `3`          | 所需的最低回憶次數。                                    |
| `--min-unique-queries <n>` | `2`          | 所需的最低相異查詢數量。                            |
| `--apply`                  | 僅預覽 | 將選取的候選項目附加至 `MEMORY.md`，並將其標記為已提升。 |
| `--include-promoted`       |              | 納入先前週期中已提升的候選項目。           |
| `--json`                   |              | 輸出 JSON。                                                       |

這些命令列介面預設值不同於排程夢境整理掃描的深層階段
門檻（請參閱下方的[夢境整理](#dreaming)）；若要讓一次性手動執行
符合掃描行為，請明確傳入旗標。

排名訊號包括：回憶頻率、擷取相關性、查詢多樣性、
時間新近度、跨日整合，以及衍生概念的豐富度；這些訊號取自
記憶回憶與每日擷取流程，另加上重複夢境整理回訪所獲得的輕度／REM 階段
強化加成。寫入前，提升流程會重新讀取即時每日筆記，因此
排名後對短期片段所做的編輯或刪除都會受到尊重，而不會從過時的快照
進行提升。

## `memory promote-explain`

說明單一提升候選項目的分數明細。

```bash
openclaw memory promote-explain <selector> [--agent <id>] [--include-promoted] [--json]
```

`<selector>` 會比對候選項目的索引鍵（完全相符或子字串）、路徑或片段
文字。

## `memory rem-harness`

預覽 REM 反思、候選真實資訊與深層階段提升輸出，
但不寫入任何內容。

```bash
openclaw memory rem-harness [--agent <id>] [--path <file-or-dir>] [--grounded] [--include-promoted] [--json]
```

- `--path <file-or-dir>`：從歷史 `YYYY-MM-DD.md`
  每日檔案植入測試框架資料，而非使用即時工作區。
- `--grounded`：也根據歷史筆記呈現有依據的 `What Happened`／`Reflections`／
  `Possible Lasting Updates` 預覽。

## `memory rem-backfill`

將有依據的歷史 REM 摘要寫入 `DREAMS.md`，以供 UI 檢視。
可還原。

```bash
openclaw memory rem-backfill --path <file-or-dir> [--agent <id>] [--stage-short-term] [--json]
openclaw memory rem-backfill --rollback [--rollback-short-term] [--json]
```

- `--path <file-or-dir>`：除非設定了 `--rollback`/`--rollback-short-term`，否則為必要項目。
  要作為回填來源的歷史每日記憶檔案或目錄。
- `--stage-short-term`：也將有依據的持久候選項目植入即時
  短期提升儲存區，使一般深層階段可以為其排名。
- `--rollback`：從 `DREAMS.md` 移除先前寫入的有依據日記項目。
- `--rollback-short-term`：移除先前暫存的有依據短期
  候選項目。

## 夢境整理

夢境整理是背景記憶整合系統，包含三個協同
階段，依序在同一排程中執行：**輕度**（整理／暫存短期
素材）、**REM**（反思並呈現主題）、**深層**（將持久
事實提升至 `MEMORY.md`）。只有深層階段會寫入 `MEMORY.md`。

- 使用 `plugins.entries.memory-core.config.dreaming.enabled: true` 啟用
  （預設為 `false`）；`memory-core` 會自動管理掃描排程工作，不需要手動
  設定 `openclaw cron add`。
- 在聊天中使用 `/dreaming on|off` 切換；使用 `/dreaming status`
  （或 `/dreaming`/`/dreaming help`）檢查。`on`/`off` 需要頻道擁有者身分
  或閘道 `operator.admin`；`status` 與說明仍對任何
  能叫用此命令的人開放。
- 人類可讀的階段輸出會寫入 `DREAMS.md`（或現有的 `dreams.md`）。
  依預設（`dreaming.storage.mode: "separate"`），每個階段也會將
  獨立報告寫入 `memory/dreaming/<phase>/YYYY-MM-DD.md`；設定 `mode:
"inline"` 可改為將報告合併至每日記憶檔案，或設定 `"both"`
  同時使用兩者。
- 排程與手動 `memory promote` 執行會共用相同的深層階段
  排名訊號；只有預設門檻不同（請比較上表與
  下方的排程預設值）。
- 排程執行會分散至每個已設定代理的記憶工作區。

排程預設值（`plugins.entries.memory-core.config.dreaming`）：

| 索引鍵                                    | 預設值     |
| -------------------------------------- | ----------- |
| `frequency`                            | `0 3 * * *` |
| `phases.deep.minScore`                 | `0.8`       |
| `phases.deep.minRecallCount`           | `3`         |
| `phases.deep.minUniqueQueries`         | `3`         |
| `phases.deep.recencyHalfLifeDays`      | `14`        |
| `phases.deep.maxAgeDays`               | `30`        |
| `phases.deep.maxPromotedSnippetTokens` | `160`       |

```json
{
  "plugins": {
    "entries": {
      "memory-core": {
        "config": {
          "dreaming": {
            "enabled": true
          }
        }
      }
    }
  }
}
```

完整索引鍵清單與階段詳細資訊：[夢境整理](/zh-TW/concepts/dreaming)、
[記憶設定參考](/zh-TW/reference/memory-config#dreaming)。

## SecretRef 閘道相依性

如果主動記憶遠端 API 金鑰欄位設定為 SecretRef，`memory`
命令會從作用中的閘道快照解析這些欄位；如果閘道
無法使用，命令會立即失敗。這需要閘道支援
`secrets.resolve` 方法；較舊的閘道會傳回未知方法錯誤。

## 相關內容

- [命令列介面參考](/zh-TW/cli)
- [記憶概覽](/zh-TW/concepts/memory)
