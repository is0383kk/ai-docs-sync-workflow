---
read_when:
    - 你希望自動執行記憶提升
    - 你想了解每個夢境整理階段的作用
    - 你想要調整整合機制，但不想污染 MEMORY.md
sidebarTitle: Dreaming
summary: 具備淺層、深層與快速動眼期階段，以及夢境日記的背景記憶整合
title: 夢境整理
x-i18n:
    generated_at: "2026-07-26T07:16:50Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 501ab42cfdfa0216c308896aa8c1719b06b49d64a62afdb004e097102a376eac
    source_path: concepts/dreaming.md
    workflow: 16
---

夢境整理是 `memory-core` 中的背景記憶整合系統。它會將強烈的短期訊號移入持久記憶，同時讓處理過程保持可解釋且可審查。

<Note>
夢境整理為**選用功能**，預設停用。
</Note>

## 夢境整理寫入的內容

- `memory/.dreams/` 中的**機器狀態**（回憶儲存區、階段訊號、擷取檢查點、鎖定）。
- `DREAMS.md`（或現有的 `dreams.md`）中的**人類可讀輸出**，以及 `memory/dreaming/<phase>/YYYY-MM-DD.md` 下的選用階段報告檔案。

長期提升仍只會寫入 `MEMORY.md`。

## 階段模型

夢境整理每次掃描會依序執行三個協作階段：淺層 -> REM -> 深層。這些是內部實作階段，而非由使用者分別設定的模式。

| 階段 | 用途                                   | 持久寫入     |
| ----- | ----------------------------------------- | ----------------- |
| 淺層 | 整理並暫存近期短期素材 | 否                |
| REM   | 反思主題與反覆出現的想法     | 否                |
| 深層  | 評分並提升持久候選項目      | 是（`MEMORY.md`） |

<AccordionGroup>
  <Accordion title="淺層階段">
    - 讀取近期短期回憶狀態、每日記憶檔案，以及可用時經過遮蔽處理的工作階段逐字稿。
    - 對訊號去重並暫存候選行。
    - 當儲存方式包含行內輸出時，寫入受管理的 `## Light Sleep` 區塊。
    - 記錄強化訊號，供之後的深層排序使用。
    - 絕不寫入 `MEMORY.md`。

  </Accordion>
  <Accordion title="REM 階段">
    - 根據近期短期軌跡建立主題與反思摘要。
    - 當儲存方式包含行內輸出時，寫入受管理的 `## REM Sleep` 區塊。
    - 記錄深層排序所使用的 REM 強化訊號。
    - 絕不寫入 `MEMORY.md`。

  </Accordion>
  <Accordion title="深層階段">
    - 使用加權評分與門檻條件對候選項目排序（`minScore`、`minRecallCount`、`minUniqueQueries` 必須全部通過）。
    - 寫入前會從即時每日檔案重新載入片段，因此會略過過時或已刪除的片段。
    - 將提升的項目附加至 `MEMORY.md`。
    - 將 `## Deep Sleep` 摘要寫入 `DREAMS.md`，並可選擇寫入 `memory/dreaming/deep/YYYY-MM-DD.md`。

  </Accordion>
</AccordionGroup>

## 工作階段逐字稿擷取

夢境整理可將經過遮蔽處理的工作階段逐字稿擷取至夢境整理語料庫。逐字稿可用時，會與每日記憶訊號及回憶軌跡一起提供給淺層階段。個人與敏感內容會在擷取前遮蔽。

## 夢境日誌

夢境整理會在 `DREAMS.md` 中保存敘事式的**夢境日誌**。每個階段累積足夠素材後，`memory-core` 會以盡力而為的方式執行背景子代理回合，並附加一則簡短日誌項目；除非已設定 `dreaming.model`，否則會使用預設執行階段模型。如果設定的模型無法使用，日誌執行會以工作階段預設模型重試一次；信任或允許清單失敗不會重試，而會保留在日誌中清楚顯示，不會無聲地改用通用日誌項目。

<Note>
日誌供人員在夢境 UI 中閱讀，並非提升來源。日誌／報告成品不會納入短期提升；只有具備依據的記憶片段才有資格提升至 `MEMORY.md`。
</Note>

另有一條具備依據的歷史回填路徑，供審查與復原工作使用：

<AccordionGroup>
  <Accordion title="回填命令">
    - `memory rem-harness --path ... --grounded` 預覽根據歷史 `YYYY-MM-DD.md` 筆記產生的具依據日誌輸出。
    - `memory rem-backfill --path ...` 將可還原且具依據的日誌項目寫入 `DREAMS.md`。
    - `memory rem-backfill --path ... --stage-short-term` 將具依據的持久候選項目暫存至一般深層階段使用的相同短期證據儲存區。
    - `memory rem-backfill --rollback` 和 `--rollback-short-term` 會移除這些暫存的回填成品，而不影響一般日誌項目或即時短期回憶。

  </Accordion>
</AccordionGroup>

控制 UI 會在代理的 Memory 分頁（Agents 頁面）中提供相同的日誌回填／重設流程，讓你能在決定具依據的候選項目是否值得提升前，先於夢境場景中檢查結果。獨立的具依據 Scene 路徑會顯示哪些暫存短期項目來自歷史重播、哪些已提升項目以具依據項目為主，並讓你只清除純具依據的暫存項目，而不影響即時短期狀態。

## 深層排序訊號

深層排序使用六個加權基礎訊號以及階段強化：

| 訊號              | 權重 | 說明                                       |
| ------------------- | ------ | ------------------------------------------------- |
| 相關性           | 0.30   | 該項目的平均擷取品質           |
| 頻率           | 0.24   | 該項目累積的短期訊號數量 |
| 查詢多樣性     | 0.15   | 使其浮現的不同查詢／日期情境      |
| 時效性             | 0.15   | 隨時間衰減的新鮮度分數                      |
| 整合       | 0.10   | 跨多日重複出現的強度                     |
| 概念豐富度 | 0.06   | 片段／路徑中的概念標籤密度             |

淺層與 REM 階段的命中會從 `memory/.dreams/phase-signals.json` 加上一小段隨時間衰減的加成。

影子試驗結果可疊加在基礎分數之上，作為任何持久寫入前的審查訊號：有幫助的試驗會為候選項目提供小幅且有上限的加成，中立試驗會使其保持延後狀態，而有害試驗會在該次評分中將其標記為拒絕。此訊號僅用於報告——它可以變更候選項目順序或審查中繼資料，但絕不會寫入 `MEMORY.md`，也不會自行提升候選項目。

### QA 影子試驗報告涵蓋範圍

QA Lab 包含一個僅供報告使用的情境，用於探索未來的夢境整理影子試驗可如何在提升前審查候選記憶：代理會比較基準答案與可使用候選記憶的答案，然後寫入包含判定、理由及風險旗標的本機報告。此涵蓋範圍僅限於 QA——它會驗證報告成品與 `MEMORY.md` 保持分離，且代理絕不會宣稱候選項目已提升。它不會新增正式環境的影子試驗行為，也不會變更深層階段提升引擎。

`memory-core` 影子試驗執行器對需要穩定成品的程式碼路徑維持相同的僅供報告合約。它接收候選項目、試驗提示、基準結果、候選結果、判定、理由、風險旗標及證據參照，接著使用 `promotion action: report-only` 寫入報告。有幫助的判定會對應至 `promote` 建議，中立判定會對應至 `defer`，而有害判定會對應至 `reject`——這些動作都不會寫入 `MEMORY.md` 或套用深層階段提升。

## 排程

啟用時，`memory-core` 會自動管理一個用於完整夢境整理掃描的排程工作，並在主要執行階段工作區及所有已設定的代理工作區間去重，因此子代理工作區的扇出不會排除主要代理的 `DREAMS.md` 與記憶狀態。

| 設定              | 預設值       |
| -------------------- | ------------- |
| `dreaming.frequency` | `0 3 * * *`   |
| `dreaming.model`     | 預設模型 |

## 快速開始

<Tabs>
  <Tab title="啟用夢境整理">
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
  </Tab>
  <Tab title="自訂掃描頻率">
    ```json
    {
      "plugins": {
        "entries": {
          "memory-core": {
            "config": {
              "dreaming": {
                "enabled": true,
                "timezone": "America/Los_Angeles",
                "frequency": "0 */6 * * *"
              }
            }
          }
        }
      }
    }
    ```
  </Tab>
</Tabs>

## 斜線命令

```text
/dreaming status
/dreaming on
/dreaming off
/dreaming help
```

對頻道呼叫端而言，`/dreaming on` 和 `/dreaming off` 需要擁有者身分；對閘道用戶端而言，則需要 `operator.admin`。`/dreaming status` 和 `/dreaming help` 為唯讀。

## 命令列介面工作流程

<Tabs>
  <Tab title="提升預覽／套用">
    ```bash
    openclaw memory promote
    openclaw memory promote --apply
    openclaw memory promote --limit 5
    openclaw memory status --deep
    ```

    除非使用命令列介面旗標覆寫，手動 `memory promote` 預設會使用深層階段門檻。

  </Tab>
  <Tab title="說明提升">
    說明特定候選項目會或不會提升的原因：

    ```bash
    openclaw memory promote-explain "router vlan"
    openclaw memory promote-explain "router vlan" --json
    ```

  </Tab>
  <Tab title="REM 測試框架預覽">
    預覽 REM 反思、候選事實與深層提升輸出，且不寫入任何內容：

    ```bash
    openclaw memory rem-harness
    openclaw memory rem-harness --json
    ```

  </Tab>
</Tabs>

## 主要預設值

所有設定皆位於 `plugins.entries.memory-core.config.dreaming` 下。

<ParamField path="enabled" type="boolean" default="false">
  啟用或停用夢境整理掃描。
</ParamField>
<ParamField path="frequency" type="string" default="0 3 * * *">
  完整夢境整理掃描的排程頻率。
</ParamField>
<ParamField path="model" type="string">
  選用的夢境日誌子代理模型覆寫。同時設定子代理 `allowedModels` 允許清單時，請使用標準 `provider/model` 值。
</ParamField>
<ParamField path="phases.deep.maxPromotedSnippetTokens" type="number" default="160">
  每個提升至 `MEMORY.md` 的短期回憶片段所保留的估計權杖數上限。排序來源資訊仍然可見。
</ParamField>

<Warning>
`dreaming.model` 需要 `plugins.entries.memory-core.subagent.allowModelOverride: true`。若要限制它，另請設定 `plugins.entries.memory-core.subagent.allowedModels`。自動重試僅涵蓋模型無法使用的錯誤；信任或允許清單失敗會保留在日誌中清楚顯示，而不會無聲地改用備援。
</Warning>

<Note>
大多數階段政策、門檻及儲存行為皆為內部實作細節。如需完整鍵值清單，請參閱[記憶設定參考](/zh-TW/reference/memory-config#dreaming)。
</Note>

## 夢境 UI

啟用時，閘道的 **Dreams** 分頁會顯示：

- 目前的夢境整理啟用狀態
- 階段層級狀態以及受管理掃描是否存在
- 短期、具依據、訊號及今日已提升項目的計數
- 下次排程執行時間
- 用於暫存歷史重播項目的獨立具依據 Scene 路徑
- 由 `doctor.memory.dreamDiary` 支援的可展開夢境日誌閱讀器

## 相關內容

- [記憶](/zh-TW/concepts/memory)
- [記憶命令列介面](/zh-TW/cli/memory)
- [記憶設定參考](/zh-TW/reference/memory-config)
- [記憶搜尋](/zh-TW/concepts/memory-search)
