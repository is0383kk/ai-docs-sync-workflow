---
read_when:
    - 你想瞭解記憶的運作方式
    - 你想知道要寫入哪些記憶檔案
summary: OpenClaw 如何跨工作階段記住資訊
title: 記憶體概覽
x-i18n:
    generated_at: "2026-07-26T08:21:39Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: cdfd5276d6289a4ee38b5203eb5443312c4b040d4ea67abe4a9c579703136339
    source_path: concepts/memory.md
    workflow: 16
---

OpenClaw 會將內容寫入你代理程式工作區中的純 Markdown 檔案（預設為 `~/.openclaw/workspace`）來記住資訊。模型只會記得已儲存至磁碟的內容；不存在隱藏狀態。

## 運作方式

你的代理程式有三個與記憶相關的檔案：

- **`MEMORY.md`** — 長期記憶。持久保存的事實、偏好和決策。會在工作階段開始時載入。
- **`memory/YYYY-MM-DD.md`**（或 `memory/YYYY-MM-DD-<slug>.md`）— 每日筆記。
  持續累積的情境資訊和觀察。只使用 `/new` 或 `/reset` 時，今天和昨天含日期的筆記會自動載入；帶有簡短名稱的變體（例如隨附的 session-memory 鉤子所寫入的檔案）則會連同只有日期的檔案一起載入。
- **`DREAMS.md`**（選用）— 供人工審閱的夢境日誌和夢境整理摘要，包括以事實為依據的歷史回填項目。

<Tip>
如果希望代理程式記住某件事，只要告訴它：“記住我偏好 TypeScript。”它會將筆記寫入適當的檔案。
</Tip>

## 各類內容的存放位置

`MEMORY.md` 是精簡且經過整理的層級：包含持久保存的事實、偏好、常設決策，以及應在工作階段開始時即可取得的簡短摘要。它不是原始逐字稿、每日日誌或完整封存。

`memory/YYYY-MM-DD.md` 檔案是工作層級：包含詳細的每日筆記、觀察、工作階段摘要，以及日後可能仍有用的原始情境資訊。這些內容會編入索引，供 `memory_search` 和 `memory_get` 使用，但不會在每次互動時注入啟動提示詞。

隨著時間經過，代理程式會從每日筆記中提煉有用內容並寫入 `MEMORY.md`，同時移除過時的長期項目。產生的工作區指示和心跳偵測流程會定期執行此作業；你不需要為每個細節手動編輯 `MEMORY.md`。

如果 `MEMORY.md` 超出啟動檔案的預算，OpenClaw 會完整保留磁碟上的檔案，但截斷注入情境的副本。應將此視為一項提示：把詳細內容移至 `memory/*.md`、在 `MEMORY.md` 中僅保留持久摘要，或在願意使用更多提示詞預算時提高啟動限制。使用 `/context list`、`/context detail` 或 `openclaw doctor`，即可查看原始大小、注入大小和截斷狀態。

## 從程式設計助理匯入

Control UI 可以從 Codex 和 Claude Code 匯入現有的本機記憶。
開啟 **Settings** → **Import Memory**，選擇目的地代理程式、檢閱偵測到的檔案，然後確認匯入。OpenClaw 只會複製 Markdown 記憶：

- Codex：位於 `~/.codex/memories`（或 `CODEX_HOME/memories`）下整合後的 `MEMORY.md` 和 `memory_summary.md` 檔案。不會匯入原始 rollout 和逐字稿檔案。
- Claude Code：每個專案在 `~/.claude/projects/*/memory` 下的自動記憶目錄中的 Markdown 檔案，以及存在時由使用者設定的 `autoMemoryDirectory`。專案指示、工作階段、設定和認證資訊不屬於這項僅限記憶的操作。

匯入的檔案會分別保存在所選代理程式工作區的 `memory/imports/codex/` 和 `memory/imports/claude-code/` 下。這些檔案會編入索引，供 `memory_search` 使用，並可透過 `memory_get` 取得；它們不會合併至代理程式的啟動 `MEMORY.md`。來源檔案不會變更。

預覽畫面會標示目的地衝突。啟用 **Replace existing imports** 即可取代這些檔案；套用時會建立經驗證的匯入前備份，並在遷移報告中保留遭覆寫檔案的逐項副本。

## 動作敏感型記憶

大多數記憶都是一般的 Markdown 筆記。有些記憶會影響代理程式日後應採取的動作；針對這些記憶，不只要記錄事實本身，也要記錄何時可安全地根據該筆記採取動作。

當筆記涉及以下內容時，請記錄該動作界線：

- 核准或權限要求，
- 暫時性限制，
- 移交給另一個工作階段、討論串或人員，
- 到期條件，
- 可安全採取動作的時機，
- 來源或擁有者權限，
- 避免執行某個誘人動作的指示。

實用的動作敏感型記憶會清楚說明：

- 哪些內容會改變未來行為，
- 何時或在什麼條件下適用，
- 何時到期，或什麼條件會允許採取動作，
- 代理程式應避免執行哪些動作，
- 來源或擁有者是誰（如果這會影響信任或權限）。

記憶可以保留核准情境，但不會強制執行政策。請使用 OpenClaw 的核准設定、沙箱隔離和排程工作來實施嚴格的操作控制。

範例：

```md
API 遷移正在另一個工作階段中進行設計。未來的互動不應從這個討論串編輯 API 實作；在遷移計畫確定之前，僅將這裡的發現用作設計輸入。
```

另一個範例：

```md
來自不受信任來源的報告必須先經過審閱才能提升採用。未來的互動應僅將它視為證據；在受信任的審閱者確認內容之前，請勿將它儲存為持久記憶。
```

這並非每筆記憶都必須遵循的結構；簡單的事實可以保持精簡。當遺失時機、權限、到期時間或可安全採取動作的情境，可能導致代理程式日後採取錯誤動作時，請使用動作敏感型界線。

針對精確提醒、定時檢查和週期性工作，請使用[排程工作](/zh-TW/automation/cron-jobs)。記憶仍可摘要這些工作周邊的持久情境。

## 已淘汰的推斷承諾

某些未來的後續事項並非持久事實。如果你提到明天有一場面試，實用的記憶可能是“面試後關心進展”，而不是“將此永久儲存在 `MEMORY.md` 中”。

推斷承諾實驗已淘汰。OpenClaw 不再擷取或傳遞這些後續事項。請使用[排程工作](/zh-TW/automation/cron-jobs)處理未來動作；舊版 `openclaw commitments` 命令仍可用於檢查或捨棄現有的已儲存資料列。

## 記憶工具

代理程式有兩個用於處理記憶的工具：

- **`memory_search`** — 使用語意搜尋尋找相關筆記，即使措辭與原文不同也能找到。
- **`memory_get`** — 讀取特定的記憶檔案或行範圍。

這兩個工具都由作用中的記憶外掛提供（預設：`memory-core`）。

## 記憶搜尋

設定嵌入模型提供者後，`memory_search` 會使用混合搜尋：結合向量相似度（語意）與關鍵字比對（ID 和程式碼符號等確切詞彙）。只要具備任何受支援提供者的 API 金鑰，即可直接使用。

<Info>
OpenClaw 預設使用 OpenAI 嵌入模型。明確設定
`memory.search.provider` 即可使用 Gemini、Voyage、Mistral、Bedrock、DeepInfra、本機 GGUF、Ollama、LM Studio、GitHub Copilot，或通用的 OpenAI 相容端點。
</Info>

如需瞭解搜尋的運作方式、調整選項和提供者設定，請參閱[記憶搜尋](/zh-TW/concepts/memory-search)。

## 記憶後端

<CardGroup cols={3}>
<Card title="內建（預設）" icon="database" href="/zh-TW/concepts/memory-builtin">
以 SQLite 為基礎。無須設定即可使用關鍵字搜尋、向量相似度和混合搜尋。不需要額外相依套件。
</Card>
<Card title="QMD" icon="search" href="/zh-TW/concepts/memory-qmd">
本機優先的附屬程序，支援重新排序、查詢擴展，以及為工作區外的目錄建立索引。
</Card>
<Card title="Honcho" icon="brain" href="/zh-TW/concepts/memory-honcho">
AI 原生的跨工作階段記憶，具備使用者建模、語意搜尋和多代理程式感知能力。需安裝外掛。
</Card>
<Card title="LanceDB" icon="layers" href="/zh-TW/plugins/memory-lancedb">
以 LanceDB 為後端的記憶，支援 OpenAI 相容嵌入模型、自動回想、自動擷取，以及本機 Ollama 嵌入模型。需安裝外掛。
</Card>
</CardGroup>

## 知識 Wiki 層

如果希望持久記憶的運作方式更像有人維護的知識庫，而不是原始筆記，請使用隨附的 `memory-wiki` 外掛。它會將持久知識編譯成具有確定性頁面結構的 Wiki 儲存庫，並包含結構化主張與證據、矛盾與新鮮度追蹤、產生的儀表板、編譯摘要，以及 Wiki 原生工具（`wiki_status`、`wiki_search`、`wiki_get`、`wiki_apply`、`wiki_lint`）。

`memory-wiki` 不會取代作用中的記憶外掛；作用中的記憶外掛仍負責回想、提升和夢境整理。`memory-wiki` 會在其旁加入具有豐富來源追溯資訊的知識層。

<CardGroup cols={1}>
<Card title="記憶 Wiki" icon="book" href="/zh-TW/plugins/memory-wiki">
將持久記憶編譯成具有豐富來源追溯資訊的 Wiki 儲存庫，並提供主張、儀表板、橋接模式，以及適合 Obsidian 的工作流程。
</Card>
</CardGroup>

## 自動記憶排清

在[壓縮](/zh-TW/concepts/compaction)摘要你的對話之前，OpenClaw 會執行一個無聲互動，提醒代理程式將重要情境儲存至記憶檔案。此功能預設啟用；設定 `agents.defaults.compaction.memoryFlush.enabled: false` 可將其關閉。

若要讓該維護互動使用本機模型，請設定只套用於記憶排清互動的明確覆寫值（它不會繼承作用中工作階段的模型備援鏈）：

```json
{
  "agents": {
    "defaults": {
      "compaction": {
        "memoryFlush": {
          "model": "ollama/qwen3:8b"
        }
      }
    }
  }
}
```

<Tip>
記憶排清可防止壓縮期間遺失情境。如果對話中有尚未寫入檔案的重要事實，系統會在產生摘要前自動儲存。
</Tip>

## 夢境整理

夢境整理是選用的背景記憶整合階段。它會收集短期回想訊號、為候選項目評分，並只將符合資格的項目提升至長期記憶（`MEMORY.md`）：

- **選擇啟用**：預設停用。
- **排程執行**：啟用後，`memory-core` 會自動管理一項週期性排程工作，以執行完整的夢境整理掃描。
- **設有門檻**：提升項目必須通過分數、回想頻率和查詢多樣性門檻。
- **可供審閱**：階段摘要和日誌項目會寫入 `DREAMS.md`，供人工審閱。

如需瞭解階段行為、評分訊號和夢境日誌詳細資訊，請參閱[夢境整理](/zh-TW/concepts/dreaming)。

## 以事實為依據的回填與即時提升

夢境整理系統有兩個相關的審閱管道：

- **即時夢境整理**會使用 `memory/.dreams/` 下的短期夢境整理儲存區，而一般深度階段會據此決定哪些內容可提升至 `MEMORY.md`。
- **以事實為依據的回填**會將歷史 `memory/YYYY-MM-DD.md` 筆記作為獨立的每日檔案讀取，並將結構化審閱輸出寫入 `DREAMS.md`。

以事實為依據的回填適合用於重播較舊的筆記，並檢查系統認為哪些內容值得持久保存，而不必手動編輯 `MEMORY.md`。

```bash
openclaw memory rem-backfill --path ./memory --stage-short-term
```

`--stage-short-term` 旗標會將以事實為依據的持久候選項目暫存到一般深度階段已使用的同一個短期夢境整理儲存區；它不會直接提升這些項目。因此：

- `DREAMS.md` 仍是人工審閱介面。
- 短期儲存區仍是供機器使用的排序介面。
- `MEMORY.md` 仍只會由深度提升流程寫入。

若要復原重播，但不影響一般日誌項目或正常回想狀態：

```bash
openclaw memory rem-backfill --rollback
openclaw memory rem-backfill --rollback-short-term
```

## 命令列介面

```bash
openclaw memory status          # 檢查索引狀態和提供者
openclaw memory search "query"  # 從命令列搜尋
openclaw memory index --force   # 重建索引
```

## 延伸閱讀

- [記憶搜尋](/zh-TW/concepts/memory-search)：搜尋流程、提供者與調校。
- [內建記憶引擎](/zh-TW/concepts/memory-builtin)：預設的 SQLite 後端。
- [QMD 記憶引擎](/zh-TW/concepts/memory-qmd)：進階的本機優先輔助程序。
- [Honcho 記憶](/zh-TW/concepts/memory-honcho)：AI 原生的跨工作階段記憶。
- [Memory LanceDB](/zh-TW/plugins/memory-lancedb)：以 LanceDB 為後端、採用 OpenAI 相容嵌入向量的外掛。
- [Memory Wiki](/zh-TW/plugins/memory-wiki)：編譯式知識庫與 Wiki 原生工具。
- [夢境整理](/zh-TW/concepts/dreaming)：在背景將短期回憶提升為長期記憶。
- [記憶設定參考](/zh-TW/reference/memory-config)：所有設定選項。
- [壓縮](/zh-TW/concepts/compaction)：壓縮如何與記憶互動。
- [主動記憶](/zh-TW/concepts/active-memory)：用於互動式聊天工作階段的子代理程式記憶。
