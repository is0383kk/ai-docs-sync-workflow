---
read_when:
    - 建置或偵錯 OpenClaw 原生外掛
    - 瞭解外掛能力模型或所有權邊界
    - 處理外掛載入流水線或登錄檔時
    - 實作提供者執行階段掛鉤或頻道外掛
sidebarTitle: Internals
summary: 外掛內部機制：功能模型、擁有權、契約、載入流水線與執行階段輔助工具
title: 外掛內部機制
x-i18n:
    generated_at: "2026-07-26T07:25:17Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: d47551b1bc2f71ce2ade3dfdd14bff8ee187616c3807f8101c1a3236e1443cc1
    source_path: plugins/architecture.md
    workflow: 16
---

這是 OpenClaw 外掛系統的**深度架構參考**。如需實作指南，請從下方其中一個聚焦頁面開始。

<CardGroup cols={2}>
  <Card title="安裝及使用外掛" icon="plug" href="/zh-TW/tools/plugin">
    新增、啟用及疑難排解外掛的終端使用者指南。
  </Card>
  <Card title="建置外掛" icon="rocket" href="/zh-TW/plugins/building-plugins">
    使用最小可運作資訊清單的第一個外掛教學。
  </Card>
  <Card title="頻道外掛" icon="comments" href="/zh-TW/plugins/sdk-channel-plugins">
    建置訊息頻道外掛。
  </Card>
  <Card title="供應商外掛" icon="microchip" href="/zh-TW/plugins/sdk-provider-plugins">
    建置模型供應商外掛。
  </Card>
  <Card title="SDK 概覽" icon="book" href="/zh-TW/plugins/sdk-overview">
    匯入對照表與註冊 API 參考。
  </Card>
</CardGroup>

## 公開能力模型

能力是 OpenClaw 內部公開的**原生外掛**模型。每個原生 OpenClaw 外掛都會註冊一種或多種能力類型：

| 能力                   | 註冊方法                                         | 外掛範例                                                    |
| ---------------------- | ------------------------------------------------ | ----------------------------------------------------------- |
| 文字推論               | `api.registerProvider(...)`                      | `anthropic`, `openai`                                       |
| 命令列介面推論後端     | `api.registerCliBackend(...)`                    | `anthropic`, `openai`                                       |
| 嵌入                   | `api.registerEmbeddingProvider(...)`             | 供應商擁有的向量外掛                                        |
| 語音                   | `api.registerSpeechProvider(...)`                | `elevenlabs`, `microsoft`                                   |
| 即時轉錄               | `api.registerRealtimeTranscriptionProvider(...)` | `openai`                                                    |
| 即時語音               | `api.registerRealtimeVoiceProvider(...)`         | `google`, `openai`                                          |
| 媒體理解               | `api.registerMediaUnderstandingProvider(...)`    | `google`, `openai`                                          |
| 轉錄稿來源             | `api.registerTranscriptSourceProvider(...)`      | `discord`, `google-meet`, `teams-meetings`, `zoom-meetings` |
| 圖片生成               | `api.registerImageGenerationProvider(...)`       | `fal`, `google`, `openai`                                   |
| 音樂生成               | `api.registerMusicGenerationProvider(...)`       | `fal`, `google`, `minimax`                                  |
| 影片生成               | `api.registerVideoGenerationProvider(...)`       | `fal`, `google`, `qwen`                                     |
| 網頁擷取               | `api.registerWebFetchProvider(...)`              | `firecrawl`                                                 |
| 網頁搜尋               | `api.registerWebSearchProvider(...)`             | `brave`, `firecrawl`, `google`                              |
| 頻道／訊息             | `api.registerChannel(...)`                       | `matrix`, `msteams`                                         |
| 閘道探索               | `api.registerGatewayDiscoveryService(...)`       | `bonjour`                                                   |

<Note>
註冊零項能力，但提供鉤子、工具、探索服務或背景服務的外掛，是**僅限舊版鉤子**外掛。此模式仍受到完整支援。
</Note>

### 外部相容性立場

能力模型已整合至核心，且目前由內建／原生外掛使用；但外部外掛相容性仍需要比“只要已匯出，就代表已凍結”更嚴格的標準。

| 外掛情況                                          | 指引                                                                                     |
| ------------------------------------------------- | ---------------------------------------------------------------------------------------- |
| 現有的外部外掛                                    | 維持以鉤子為基礎的整合運作；這是相容性基準。                                             |
| 新的內建／原生外掛                                | 優先使用明確的能力註冊，而非供應商專屬的深入存取或新的僅限鉤子設計。                     |
| 採用能力註冊的外部外掛                            | 允許採用，但除非文件標記為穩定，否則應將能力專屬輔助介面視為仍在演進。                   |

能力註冊是預定的發展方向。在轉換期間，舊版鉤子仍是外部外掛最安全且不會造成破壞的途徑。匯出的輔助子路徑並非全都同等穩定——請優先使用範圍明確且已有文件記載的合約，而非附帶匯出的輔助項目。

### 外掛形態

OpenClaw 會依據每個已載入外掛的實際註冊行為（而不只是靜態中繼資料），將其分類為一種形態：

<AccordionGroup>
  <Accordion title="單一能力">
    僅註冊一種能力類型（例如只有供應商能力的外掛，如 `arcee` 或 `chutes`）。
  </Accordion>
  <Accordion title="混合能力">
    註冊多種能力類型（例如 `openai` 擁有文字推論、語音、媒體理解及圖片生成能力）。
  </Accordion>
  <Accordion title="僅限鉤子">
    僅註冊鉤子（具型別或自訂），不註冊能力、工具、命令或服務。
  </Accordion>
  <Accordion title="非能力">
    註冊工具、命令、服務或路由，但不註冊能力。
  </Accordion>
</AccordionGroup>

使用 `openclaw plugins inspect <id>` 查看外掛的形態與能力明細。詳情請參閱[命令列介面參考](/zh-TW/cli/plugins#inspect)。

### 相容性訊號

`openclaw doctor`、`openclaw plugins inspect <id>`、`openclaw status --all` 及 `openclaw plugins doctor` 會顯示以下相容性通知：

| 訊號                                       | 意義                                                                                               |
| ------------------------------------------ | -------------------------------------------------------------------------------------------------- |
| **設定有效**                               | 設定可正常剖析，且外掛可解析                                                                       |
| **僅限鉤子**（資訊）                       | 外掛僅註冊鉤子；這是受支援的途徑，但尚未遷移至能力註冊                                             |
| **已棄用的記憶體嵌入 API**（警告）         | 非內建外掛使用舊的記憶體專屬嵌入供應商 API，而非 `registerEmbeddingProvider` |
| **嚴重錯誤**                               | 設定無效或外掛載入失敗                                                                             |

目前所有建議／警告訊號都不會使你的外掛中斷。這些訊號也會出現在 `openclaw status --all` 和 `openclaw plugins doctor` 中。

## 架構概覽

OpenClaw 的外掛系統分為四層：

<Steps>
  <Step title="資訊清單與探索">
    OpenClaw 會從設定的路徑、工作區根目錄、全域外掛根目錄及內建外掛中尋找候選外掛。探索程序會優先讀取原生 `openclaw.plugin.json` 資訊清單及受支援的套件資訊清單。
  </Step>
  <Step title="啟用與驗證">
    核心會判定探索到的外掛是已啟用、已停用、遭封鎖，或已獲選使用記憶體等互斥插槽。
  </Step>
  <Step title="執行階段載入">
    原生 OpenClaw 外掛會在處理程序內載入，並將能力註冊至中央登錄檔。封裝的 JavaScript 透過原生 `require` 載入；第三方本機原始碼 TypeScript 則使用緊急備援的 Jiti。相容的套件會正規化為登錄檔記錄，而不匯入執行階段程式碼。
  </Step>
  <Step title="介面使用">
    OpenClaw 的其他部分會讀取登錄檔，以公開工具、頻道、供應商設定、鉤子、HTTP 路由、命令列介面命令及服務。
  </Step>
</Steps>

針對外掛命令列介面，根命令探索會分為兩個階段：

- 剖析階段的中繼資料來自 `registerCli(..., { descriptors: [...] })`
- 實際的外掛命令列介面模組可維持延遲載入，並於第一次叫用時註冊

如此可讓外掛擁有的命令列介面程式碼保留在外掛內，同時仍允許 OpenClaw 在剖析前保留根命令名稱。

重要的設計邊界如下：

- 資訊清單／設定驗證應可僅使用**資訊清單／結構描述中繼資料**運作，而無須執行外掛程式碼
- 原生能力探索可載入受信任的外掛進入點程式碼，以建立不啟用功能的登錄檔快照
- 原生執行階段行為來自外掛模組的 `register(api)` 路徑及 `api.registrationMode === "full"`

這項區分讓 OpenClaw 能在完整執行階段啟用前驗證設定、說明外掛遺失／停用的原因，並建立 UI／結構描述提示。

### 外掛中繼資料快照與查閱表

閘道啟動時，會為目前的設定快照建立一個 `PluginMetadataSnapshot`。此快照僅包含中繼資料：它會儲存已安裝外掛索引、資訊清單登錄檔、資訊清單診斷、擁有者對照表、外掛 ID 正規化器及資訊清單記錄。它不會保存已載入的外掛模組、供應商 SDK、套件內容或執行階段匯出項目。

可感知外掛的設定驗證、啟動時自動啟用，以及閘道外掛啟動程序，都會使用此快照，而不會各自重新建立資訊清單／索引中繼資料。`PluginLookUpTable` 衍生自同一份快照，並加入目前執行階段設定的啟動外掛計畫。

啟動後，閘道會將目前的中繼資料快照保留為可替換的執行階段產物。重複進行執行階段供應商探索時，可借用該快照，而不必在每次供應商目錄掃描時重建已安裝索引及資訊清單登錄檔。閘道關閉、設定／外掛清冊變更，以及寫入已安裝索引時，快照會遭清除或替換；如果不存在相容的目前快照，呼叫端會退回未快取的資訊清單／索引路徑。相容性檢查必須包含 `plugins.load.paths` 與預設代理程式工作區等外掛探索根目錄，因為工作區外掛屬於中繼資料範圍的一部分。

快照及查閱表可讓重複的啟動決策維持在快速路徑上：

- 頻道擁有權
- 延後頻道啟動
- 啟動外掛 ID
- 供應商及命令列介面後端擁有權
- 設定供應商、命令別名、模型目錄供應商及資訊清單合約擁有權
- 外掛設定結構描述及頻道設定結構描述驗證
- 啟動時自動啟用決策

安全邊界是替換快照，而不是修改快照。當設定、外掛清冊、安裝記錄或持久化索引政策變更時，請重建快照。請勿將其視為廣泛的可變全域登錄檔，也不要保留數量無上限的歷史快照。執行階段外掛載入仍與中繼資料快照分離，避免過時的執行階段狀態被隱藏在中繼資料快取後方。

快取規則記載於[外掛架構內部機制](/zh-TW/plugins/architecture-internals#plugin-cache-boundary)：除非呼叫端持有目前流程的明確快照、查閱表或資訊清單登錄檔，否則資訊清單與探索中繼資料一律為最新。隱藏的中繼資料快取及以實際時間為基礎的 TTL 並非外掛載入的一部分。只有執行階段載入器、模組及相依性產物快取，才能在程式碼或已安裝產物實際載入後持續存在。

有些冷路徑呼叫端仍直接從持久化的已安裝外掛索引重建資訊清單登錄表，而不是接收閘道 `PluginLookUpTable`。該路徑現在會依需求重建登錄表；若呼叫端已有目前的查找表或明確的資訊清單登錄表，應優先透過執行階段流程傳遞。

### 啟用規劃

啟用規劃是控制平面的一部分。呼叫端可在載入更廣泛的執行階段登錄表之前，查詢哪些外掛與具體命令、提供者、頻道、路由、代理程式框架或能力相關。

規劃器會維持目前資訊清單行為的相容性：

- `activation.*` 欄位是明確的規劃器提示
- `providers`、`channels`、`commandAliases`、`setup.providers`、`contracts.tools` 和掛鉤仍作為資訊清單擁有權的備援依據
- 僅傳回 ID 的規劃器 API 仍可供現有呼叫端使用
- 規劃 API 會回報原因標籤，讓診斷能區分明確提示與擁有權備援依據

<Warning>
請勿將 `activation` 視為生命週期掛鉤或 `register(...)` 的替代方案。它是用於縮小載入範圍的中繼資料。當擁有權欄位已能描述關係時，應優先使用這些欄位；僅將 `activation` 用於額外的規劃器提示。
</Warning>

### 頻道外掛與共用訊息工具

頻道外掛無須為一般聊天動作註冊獨立的傳送、編輯或回應工具。OpenClaw 在核心中保留一個共用的 `message` 工具，而頻道外掛負責其背後的頻道專屬探索與執行。

目前的邊界如下：

- 核心負責共用的 `message` 工具主機、提示詞接線、工作階段／討論串簿記及執行分派
- 頻道外掛負責有範圍限制的動作探索、能力探索及任何頻道專屬結構描述片段
- 頻道外掛負責提供者專屬的工作階段對話語法，例如對話 ID 如何編碼討論串 ID，或如何從父對話繼承
- 頻道外掛透過其動作配接器執行最終動作

對頻道外掛而言，SDK 介面是 `ChannelMessageActionAdapter.describeMessageTool(...)`。這個統一探索呼叫可讓外掛一併傳回其可見動作、能力及結構描述貢獻，避免這些部分彼此脫節。

訊息動作名稱採用刻意封閉且由核心擁有的詞彙集，讓每個傳輸層都能呈現所有動作。外掛需透過核心 PR 新增動作名稱；系統刻意不支援執行階段註冊。

當頻道專屬的訊息工具參數包含媒體來源（例如本機路徑或遠端媒體 URL）時，外掛也應從 `describeMessageTool(...)` 傳回 `mediaSourceParams`。核心會使用這份明確清單套用沙箱路徑正規化及傳出媒體存取提示，而無須硬式編碼由外掛擁有的參數名稱。此處應優先使用以動作為範圍的對應表，而非整個頻道共用的扁平清單，避免僅供個人檔案使用的媒體參數在 `send` 等不相關動作上遭到正規化。

核心會將執行階段範圍傳入該探索步驟。重要欄位包括：

- `accountId`
- `currentChannelId`
- `currentThreadTs`
- `currentMessageId`
- `sessionKey`
- `sessionId`
- `agentId`
- 受信任的傳入 `requesterSenderId`

這對具備情境感知能力的外掛非常重要。頻道可根據使用中的帳號、目前的聊天室／討論串／訊息或受信任的請求者身分，隱藏或顯示訊息動作，而無須在核心 `message` 工具中硬式編碼頻道專屬分支。

因此，內嵌執行器的路由變更仍屬於外掛工作：執行器負責將目前的聊天／工作階段身分轉送至外掛探索邊界，使共用的 `message` 工具能為目前回合公開正確且由頻道擁有的介面。

針對頻道擁有的執行輔助程式，頻道外掛應將執行執行階段保留在各自的外掛模組內。核心不再擁有 `src/agents/tools` 下的 Discord、Slack、Telegram 或 WhatsApp 訊息動作執行階段。我們不發布獨立的 `plugin-sdk/*-action-runtime` 子路徑，而這些外掛應直接從自身擁有的外掛模組匯入本機執行階段程式碼。

同樣的邊界普遍適用於以提供者命名的 SDK 介面：核心不應匯入 Discord、Signal、Slack、WhatsApp 或類似外掛的頻道專屬便利彙整模組。如果核心需要某項行為，應使用隨附外掛自身的 `api.ts`／`runtime-api.ts` 彙整模組，或將該需求提升為共用 SDK 中範圍明確的通用能力。

隨附外掛也遵循相同規則。隨附外掛的 `runtime-api.ts` 不應重新匯出自身品牌化的 `openclaw/plugin-sdk/<plugin-id>` 門面。這些品牌化門面仍作為外部外掛及舊版使用者的相容性墊片，但隨附外掛應使用本機匯出，以及 `openclaw/plugin-sdk/channel-policy`、`openclaw/plugin-sdk/runtime-store` 或 `openclaw/plugin-sdk/webhook-ingress` 等範圍明確的通用 SDK 子路徑。除非現有外部生態系統的相容性邊界有所要求，否則新程式碼不應新增外掛 ID 專屬的 SDK 門面。

特別就投票而言，有兩條執行路徑：

- `outbound.sendPoll` 是適用於符合通用投票模型之頻道的共用基準
- `actions.handleAction("poll")` 是處理頻道專屬投票語意或額外投票參數的首選路徑

核心現在會等到外掛投票分派拒絕該動作後，才進行共用投票剖析，使外掛擁有的投票處理常式能接受頻道專屬投票欄位，而不會先遭通用投票剖析器阻擋。

如需完整啟動順序，請參閱[外掛架構內部機制](/zh-TW/plugins/architecture-internals)。

## 能力擁有權模型

OpenClaw 將原生外掛視為**公司**或**功能**的擁有權邊界，而不是無關整合功能的雜物箱。

這表示：

- 公司外掛通常應擁有該公司所有面向 OpenClaw 的介面
- 功能外掛通常應擁有其引入的完整功能介面
- 頻道應使用共用核心能力，而非臨時重新實作提供者行為

<AccordionGroup>
  <Accordion title="廠商多重能力">
    `google` 負責文字推論、命令列介面後端、嵌入、語音、即時語音、媒體理解、圖片／音樂／影片生成及網頁搜尋。`openai` 負責文字推論、嵌入、語音、即時轉錄、即時語音、媒體理解及圖片／影片生成。`minimax` 負責文字推論，以及媒體理解、語音、圖片／音樂／影片生成和網頁搜尋。
  </Accordion>
  <Accordion title="廠商單一能力">
    `arcee` 和 `chutes` 僅負責文字推論；`microsoft` 僅負責語音。廠商外掛在需要涵蓋該廠商更多介面前，可以維持如此狹窄的範圍。
  </Accordion>
  <Accordion title="功能外掛">
    `voice-call` 負責通話傳輸、工具、命令列介面、路由及 Twilio 媒體串流橋接，但會使用共用的語音、即時轉錄及即時語音能力，而非直接匯入廠商外掛。
  </Accordion>
</AccordionGroup>

預期的最終狀態如下：

- 即使廠商面向 OpenClaw 的介面橫跨文字模型、語音、圖片及影片，也會集中在一個外掛中
- 其他廠商也能對自己的介面範圍採取相同做法
- 頻道不在意哪個廠商外掛擁有該提供者；頻道使用的是核心所公開的共用能力合約

關鍵差異如下：

- **外掛** = 擁有權邊界
- **能力** = 可由多個外掛實作或使用的核心合約

因此，如果 OpenClaw 新增影片等領域，第一個問題不是“哪個提供者應硬式編碼影片處理？”，而是“核心影片能力合約是什麼？”。該合約建立後，廠商外掛便可針對它註冊，而頻道／功能外掛則可使用它。

如果該能力尚不存在，正確做法通常是：

<Steps>
  <Step title="定義能力">
    在核心中定義缺少的能力。
  </Step>
  <Step title="透過 SDK 公開">
    以具型別的方式透過外掛 API／執行階段公開該能力。
  </Step>
  <Step title="接線使用端">
    將頻道／功能接線至該能力。
  </Step>
  <Step title="廠商實作">
    讓廠商外掛註冊實作。
  </Step>
</Steps>

如此可保持擁有權明確，同時避免核心行為依賴單一廠商或一次性的外掛專屬程式碼路徑。

### 能力分層

決定程式碼歸屬時，請使用以下心智模型：

<Tabs>
  <Tab title="核心能力層">
    共用協調、原則、備援、設定合併規則、傳遞語意及具型別合約。
  </Tab>
  <Tab title="廠商外掛層">
    廠商專屬 API、驗證、模型目錄、語音合成、圖片生成、影片後端及用量端點。
  </Tab>
  <Tab title="頻道／功能外掛層">
    使用核心能力，並在特定介面上呈現的 Discord／Slack／語音通話等整合。
  </Tab>
</Tabs>

例如，TTS 遵循以下形式：

- 核心負責回覆時的 TTS 原則、備援順序、偏好設定及頻道傳遞
- `elevenlabs`、`google`、`microsoft` 和 `openai` 負責合成實作
- `voice-call` 使用電話語音 TTS 執行階段輔助程式

未來的能力也應優先採用相同模式。

### 多重能力公司外掛範例

公司外掛從外部看來應具有一致性。如果 OpenClaw 為模型、語音、即時轉錄、即時語音、媒體理解、圖片生成、影片生成、網頁擷取及網頁搜尋提供共用合約，廠商便可在同一處擁有其所有介面：

```ts
import { definePluginEntry } from "openclaw/plugin-sdk/plugin-entry";
import { exampleAiMedia } from "./exampleai-media.js";

export default definePluginEntry({
  id: "exampleai",
  name: "ExampleAI",
  description: "ExampleAI 模型與媒體能力。",
  register(api) {
    api.registerProvider({
      id: "exampleai",
      // 驗證／模型目錄／執行階段掛鉤
    });

    api.registerSpeechProvider({
      id: "exampleai",
      // 廠商語音設定 — 直接實作 SpeechProviderPlugin 介面
    });

    api.registerMediaUnderstandingProvider({
      id: "exampleai",
      capabilities: ["image", "audio", "video"],
      describeImage: (req) => exampleAiMedia.describeImage(req),
      transcribeAudio: (req) => exampleAiMedia.transcribeAudio(req),
      describeVideo: (req) => exampleAiMedia.describeVideo(req),
    });

    api.registerWebSearchProvider({
      id: "exampleai-search",
      createTool() {
        // 傳回廠商擁有的網頁搜尋工具。
      },
    });
  },
});
```

重點不在確切的輔助程式名稱，而在整體形式：

- 由一個外掛擁有廠商介面
- 核心仍負責能力合約
- 提供者請求轉譯及 HTTP 輔助程式留在廠商外掛中
- 頻道及功能外掛使用 `api.runtime.*` 輔助程式，而非廠商程式碼
- 合約測試可斷言外掛已註冊其宣稱擁有的能力

### 能力範例：影片理解

OpenClaw 已將圖片／音訊／影片理解視為同一項共用能力。相同的擁有權模型也適用於此：

<Steps>
  <Step title="核心定義契約">
    核心定義媒體理解契約。
  </Step>
  <Step title="供應商外掛註冊">
    供應商外掛視需要註冊 `describeImage`、`transcribeAudio` 和 `describeVideo`。
  </Step>
  <Step title="取用端使用共用行為">
    頻道與功能外掛取用共用的核心行為，而非直接連接至供應商程式碼。
  </Step>
</Steps>

這可避免將單一供應商對影片的假設固化到核心中。外掛擁有供應商介面；核心則擁有能力契約與後援行為。

影片生成功能已採用相同順序：核心擁有具型別的能力契約與執行階段輔助函式，而供應商外掛則針對該契約註冊 `api.registerVideoGenerationProvider(...)` 實作。

需要具體的推出檢查清單嗎？請參閱[能力指南](/zh-TW/plugins/adding-capabilities)。

## 契約與強制執行

外掛 API 介面刻意集中於 `OpenClawPluginApi` 並採用型別。該契約定義支援的註冊點，以及外掛可依賴的執行階段輔助函式。

其重要性如下：

- 外掛作者可獲得一套穩定的內部標準
- 核心可拒絕重複的擁有權，例如兩個外掛註冊相同的供應商 id
- 啟動時可針對格式錯誤的註冊顯示可採取行動的診斷資訊
- 契約測試可強制執行內建外掛的擁有權，並防止無聲偏移

強制執行分為兩層：

<AccordionGroup>
  <Accordion title="執行階段註冊強制執行">
    外掛載入時，外掛登錄檔會驗證註冊。例如：重複的供應商 id、重複的語音供應商 id，以及格式錯誤的註冊，都會產生外掛診斷資訊，而非導致未定義行為。
  </Accordion>
  <Accordion title="契約測試">
    測試執行期間，內建外掛會記錄於契約登錄檔中，讓 OpenClaw 能明確斷言擁有權。目前這用於模型供應商、語音供應商、網頁搜尋供應商，以及內建註冊的擁有權。
  </Accordion>
</AccordionGroup>

實際效果是 OpenClaw 能預先得知哪個外掛擁有哪些介面。由於擁有權是明確宣告、具型別且可測試的，而非隱含的，因此核心與頻道得以無縫組合。

### 契約應包含的內容

<Tabs>
  <Tab title="良好的契約">
    - 具型別
    - 精簡
    - 針對特定能力
    - 由核心擁有
    - 可由多個外掛重複使用
    - 頻道／功能無須瞭解供應商即可取用

  </Tab>
  <Tab title="不良的契約">
    - 隱藏於核心中的供應商特定政策
    - 繞過登錄檔的一次性外掛逃生機制
    - 頻道程式碼直接存取供應商實作
    - 不屬於 `OpenClawPluginApi` 或 `api.runtime` 的臨時執行階段物件

  </Tab>
</Tabs>

如有疑問，請提高抽象層級：先定義能力，再讓外掛接入。

## 執行模型

原生 OpenClaw 外掛與閘道在**同一處理程序內**執行，並未受到沙箱隔離。載入的原生外掛與核心程式碼具有相同的處理程序層級信任邊界。

<Warning>
原生外掛的影響：外掛可以註冊工具、網路處理常式、掛鉤與服務；外掛錯誤可能造成閘道當機或不穩定；惡意原生外掛等同於在 OpenClaw 處理程序內任意執行程式碼。
</Warning>

相容套件預設較為安全，因為 OpenClaw 目前將其視為中繼資料／內容套件。在目前版本中，這主要是指內建 Skills。

對非內建外掛使用允許清單與明確的安裝／載入路徑。將工作區外掛視為開發期間的程式碼，而非生產環境的預設值。

對於內建工作區套件名稱，外掛 id 預設應以 npm 名稱為基準：`@openclaw/<id>`；若套件刻意公開範圍較窄的外掛角色，也可採用經核准的型別化後綴，例如 `-provider`、`-plugin`、`-speech`、`-sandbox` 或 `-media-understanding`。

<Note>
**信任注意事項：**`plugins.allow` 信任的是**外掛 id**，而非來源出處。啟用或列入允許清單後，若工作區外掛與內建外掛具有相同 id，便會刻意覆蓋內建副本。這是正常行為，且有助於本機開發、修補程式測試和緊急修正。內建外掛的信任是根據來源快照判定，也就是載入時磁碟上的資訊清單與程式碼，而非安裝中繼資料。遭損毀或替換的安裝記錄，無法悄悄將內建外掛的信任範圍擴大到實際來源所宣告的範圍之外。
</Note>

## 匯出邊界

OpenClaw 匯出的是能力，而非為實作提供便利的項目。

讓能力註冊維持公開。移除非契約輔助匯出：

- 內建外掛專用的輔助子路徑
- 不打算作為公用 API 的執行階段管線子路徑
- 供應商專用的便利輔助函式
- 屬於實作細節的設定／新手引導輔助函式

保留的內建外掛輔助子路徑已從產生的 SDK 匯出對應表中移除。將擁有者專用的輔助函式保留於其所屬外掛套件內；只將可重複使用的主機行為提升為通用 SDK 契約，例如 `plugin-sdk/gateway-runtime`、`plugin-sdk/security-runtime`，以及注入的外掛 API 能力。

## 內部機制與參考資料

如需瞭解載入流水線、登錄檔模型、供應商執行階段掛鉤、閘道 HTTP 路由、訊息工具結構描述、頻道目標解析、供應商目錄、內容引擎外掛，以及新增能力的指南，請參閱[外掛架構內部機制](/zh-TW/plugins/architecture-internals)。

## 相關內容

- [建置外掛](/zh-TW/plugins/building-plugins)
- [外掛資訊清單](/zh-TW/plugins/manifest)
- [外掛 SDK 設定](/zh-TW/plugins/sdk-setup)
