---
read_when:
    - 你看到 OPENCLAW_PLUGIN_SDK_COMPAT_DEPRECATED 警告
    - 你看到 OPENCLAW_EXTENSION_API_DEPRECATED 警告
    - 你在 OpenClaw 2026.4.25 之前使用了 api.registerEmbeddedExtensionFactory
    - 你正在將外掛更新為現代外掛架構
    - 你維護一個外部 OpenClaw 外掛
sidebarTitle: Migrate to SDK
summary: 從舊版向下相容層遷移至現代外掛 SDK
title: 外掛 SDK 遷移
x-i18n:
    generated_at: "2026-07-26T07:29:58Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: a483f9c0f8409505fc2688872995382944e002520ceb651214dbc5ad8e3554fb
    source_path: plugins/sdk-migration.md
    workflow: 16
---

OpenClaw 已將廣泛的向後相容層替換為現代化的外掛架構，
此架構由小型且聚焦的匯入項目組成。如果你的外掛早於該項
變更，本指南可協助它遷移至目前的合約。

## 變更內容

過去有數個過度開放的匯入介面，讓外掛能從單一進入點存取
幾乎所有內容：

- **`openclaw/plugin-sdk`** 和 **`openclaw/plugin-sdk/compat`** - 在建置聚焦的 SDK
  期間，曾重新匯出數十個輔助工具。這兩個根路徑現已移除；
  請改為匯入文件記載的子路徑。
- **`openclaw/plugin-sdk/infra-runtime`** - 混合了系統事件、心跳偵測狀態、
  傳遞佇列、擷取／Proxy 輔助工具、檔案輔助工具、核准類型及
  不相關公用程式的廣泛彙總匯出。
- **`openclaw/plugin-sdk/config-runtime`** - 廣泛的設定彙總匯出，僅為後續的
  相容期間而保留；直接的執行階段載入／寫入輔助工具已移除。
- **`openclaw/extension-api`** - 已移除的橋接層，先前讓外掛可直接
  存取主機端的輔助工具，例如內嵌代理程式執行器。
- **`api.registerEmbeddedExtensionFactory(...)`** - 已移除且僅供內嵌執行器使用的
  掛鉤，先前用於觀察內嵌執行器事件，例如 `tool_result`。請改用代理程式
  工具結果中介軟體（請參閱[將內嵌工具結果擴充功能遷移至中介軟體](#how-to-migrate)）。

根 SDK、相容性彙總匯出、擴充功能橋接層及內嵌擴充功能工廠
均已移除。`infra-runtime` 和 `config-runtime` 僅為各自另行記錄的後續
期間而保留；新外掛應使用聚焦的子路徑。

<Warning>
  匯入已移除根路徑、相容性或擴充功能介面的外掛將不再
  載入。升級前，請依照下方對應關係進行調整。
</Warning>

OpenClaw 不會在引入替代方案的同一項變更中，移除或重新解讀文件記載的
外掛行為。破壞合約的變更會先經過相容性配接器、診斷、文件及
淘汰期間。這適用於 SDK 匯入、資訊清單欄位、設定 API、掛鉤及執行階段
註冊行為。

### 原因

- **啟動緩慢** - 匯入一個輔助工具會載入數十個不相關的模組。
- **循環相依性** - 廣泛的重新匯出使匯入循環很容易
  產生。
- **API 介面不明確** - 無法分辨穩定匯出項目與內部匯出項目。

每個 `openclaw/plugin-sdk/<subpath>` 現在都是小型、獨立且
具有文件化合約的模組。

綁定通道的舊版供應商便利介面也已移除 -
帶有通道品牌的輔助工具捷徑是私有單一儲存庫的便利功能，而非
穩定的外掛合約。請改用範圍狹窄的通用 SDK 子路徑。在
綁定的外掛工作區內，請將供應商擁有的輔助工具保留在該外掛自身的
`api.ts` 或 `runtime-api.ts` 中：

- Anthropic 將 Claude 專用的串流輔助工具保留在自己的 `api.ts` /
  `contract-api.ts` 介面中。
- OpenAI 將供應商建構器、預設模型輔助工具及即時供應商
  建構器保留在自己的 `api.ts` 中。
- OpenRouter 將供應商建構器及新手引導／設定輔助工具保留在自己的
  `api.ts` 中。

## 相容性政策

外部外掛的相容性工作依照以下順序進行：

1. 新增合約。
2. 透過相容性配接器維持舊有行為的連接。
3. 發出診斷或警告，指出舊路徑及其替代方案。
4. 在測試中涵蓋兩條路徑。
5. 記錄淘汰及遷移路徑。
6. 僅在公告的遷移期間結束後移除，通常會在主要
   版本中進行。

如果資訊清單欄位仍被接受，請持續使用，直到文件及
診斷另有說明為止。新程式碼應優先使用文件記載的替代方案；
現有外掛不應在一般次要版本發布期間中斷。

### 已發布通道的設定相容性

透過 `2026.7.1` 發布的 Slack、Discord、Signal 和 Microsoft Teams
套件，會從 `openclaw/plugin-sdk/bundled-channel-config-schema` 匯入通道專用的設定結構描述。
已發布的 Slack 和 Discord 套件也會從
`openclaw/plugin-sdk/setup-runtime` 匯入 `createLegacyCompatChannelDmPolicy` 和
`promptLegacyChannelAllowFromForAccount`。

這些匯出項目仍以已淘汰的執行階段相容性配接器形式提供。
新外掛及重新發布的外掛應在本機擁有其設定結構描述及設定政策，
並使用 `channel-config-schema` 和 `setup-runtime` 的通用基礎功能。
只有在支援的最低已發布套件版本不再匯入這些項目後，
才能移除相容性匯出項目。

### 通道設定輸入欄位的相容性

`ChannelSetupInput` 現在只會永久保留跨通道設定封套的型別。
通道專用欄位仍在已淘汰的相容性層中保有型別，讓現有外部外掛
在外掛作者將這些欄位移入外掛本機設定輸入型別時仍可編譯。

OpenClaw 不發布主要版本。2026-07-22 的登錄檔掃描檢查了
426 個已發布的樹外通道外掛，並移除 21 個沒有讀取端的欄位。
保留的 22 個欄位各自都有已知的已發布讀取端。只要沒有已發布外掛
讀取某個欄位，就會立即刪除該欄位；隨著外掛作者遷移至外掛本機
設定輸入型別，保留集合會持續縮小。

同一次掃描也移除了 23 個沒有已發布相依項目的舊版未宣告配接器
提升索引鍵。目前保留六個常用索引鍵及僅供設定使用的 `rooms`
索引鍵。隨著已發布外掛宣告 `singleAccountKeysToMove`，該集合也會持續縮小。

共用型別沒有索引簽章。外掛擁有的索引鍵仍可存在於執行階段
輸入物件中；請在外掛本機的交集型別中宣告它們，或透過擁有該索引鍵的
外掛設定結構描述縮小其型別。

| `code`                                  | `owner`   | `replacement`                                                                                    | 移除條件                                                     |
| --------------------------------------- | --------- | ------------------------------------------------------------------------------------------------ | --------------------------------------------------------------------- |
| `plugin-sdk-channel-setup-input-fields` | `channel` | 將 `ChannelSetupInput` 與宣告所屬通道欄位的外掛本機型別建立交集 | 當已發布外掛登錄檔掃描找不到讀取端時刪除欄位 |

舊版未宣告配接器提升層採用相同的讀取端驅動
政策。請宣告 `singleAccountKeysToMove`；若外掛不需要額外的提升索引鍵，
也應包含空陣列，以便逐一淘汰共用備援中的索引鍵。

#### 驗證讀取端

1. 使用每個 `nextCursor` 分頁瀏覽 `https://clawhub.ai/api/v1/packages?family=code-plugin&limit=100`，並保留其 `categories` 包含 `channels` 的套件。
2. 從 `npm search --json --searchlimit=1000 "openclaw channel plugin"` 新增 npm 候選項目。從 GitHub 中針對 `openclaw/plugin-sdk/channel-setup`、`openclaw/plugin-sdk/setup` 和 `openclaw/plugin-sdk/core` 的程式碼搜尋新增僅原始碼候選項目。
3. 解析每個候選項目的最新已發布版本。執行 `npm pack <package>@<version> --json --pack-destination <temp-dir>`、解壓縮套件，並檢查已出貨的 `dist` JavaScript 和宣告，尋找直接或解構式欄位讀取。如果套件沒有 npm 版本，請下載 ClawHub 成品。
4. 記錄套件、版本、欄位或提升索引鍵，以及相符的檔案。只有在沒有任何已發布外掛成品讀取該欄位或索引鍵時，才能刪除它。請讓保留欄位及索引鍵清單旁程式碼註解中的讀取端名稱與掃描結果保持同步。

這僅是原始碼／型別相容性記錄。它沒有執行階段配接器或
相容性登錄項目，因為執行階段設定輸入物件及設定
行為並未變更。

使用 `pnpm plugins:boundary-report` 稽核目前的遷移佇列：

| 旗標                                                    | 效果                                                                         |
| ------------------------------------------------------- | ------------------------------------------------------------------------------ |
| `--summary`（或 `pnpm plugins:boundary-report:summary`） | 顯示精簡計數，而非完整詳細資料。                                         |
| `--json`                                                | 機器可讀報告。                                                       |
| `--owner <id>`                                          | 篩選為單一外掛或相容性擁有者。                                   |
| `--fail-on-cross-owner`                                 | 遇到跨擁有者的保留 SDK 匯入時，以非零狀態結束。                             |
| `--fail-on-eligible-compat`                             | 當已淘汰相容性記錄的 `removeAfter` 日期已過時，以非零狀態結束。 |
| `--fail-on-unclassified-unused-reserved`                | 遇到未使用的保留 SDK 墊片時，以非零狀態結束。                                    |

`pnpm plugins:boundary-report:ci` 會啟用全部三個失敗旗標。已淘汰的
記錄通常具有明確的 `removeAfter` 日期，而非含糊的「下一個
主要版本」。擁有者尚未核准日期的記錄不會包含
`removeAfter`，會顯示為 `no-date`，且永遠不符合移除資格。
報告會依日期將已淘汰記錄分組、計算本機程式碼／文件參照、
顯示跨擁有者的保留 SDK 匯入，並摘要私有的
記憶體主機 SDK 橋接層。保留的 SDK 子路徑必須有追蹤記錄的擁有者使用情形；
未使用的保留匯出項目應從公開 SDK 中移除。

### 媒體舊版投影

`media-legacy-projection` 相容性記錄涵蓋舊有的平行
媒體欄位、酬載建構器、掛鉤中繼資料別名及媒體範本
名稱。其核准的 `removeAfter` 日期為 **2026-10-01**（以事實優先的替代方案
出貨後兩個發布週期）。屆時要移除它，還必須完成沒有問題的
已發布外掛成品掃描；請在該日期前完成遷移。

針對通道輸入，請將單數／複數的 `MediaPath`、`MediaUrl`、
`MediaType`、`MediaPaths`、`MediaUrls`、`MediaTypes`、
`MediaTranscribedIndexes`、`MediaWorkspaceDir` 和 `MediaStaged` 替換為有序
事實：

```ts
import { toInboundMediaFacts } from "openclaw/plugin-sdk/channel-inbound";

const media = toInboundMediaFacts([
  { path: saved.path, url: nativeUrl, contentType: saved.contentType, messageId },
]);

const ctx = finalizeInboundContext({ Body: caption, media });
```

在 `inbound_claim` 和 `message_received` 掛鉤中使用 `event.media`。如果遠端
媒體尚未在本機暫存，請使用 `event.originalMedia` 進行身分識別／診斷，
並等待 `event.media`；`event.mediaStagingPending` 可區分該
狀態。請勿從 `event.metadata` 讀取已淘汰的單數／複數屬性。

針對命令列介面的媒體模型，請將 `{{MediaPath}}`、`{{MediaUrl}}`、`{{MediaType}}`
和 `{{MediaDir}}` 替換為 `{{AttachmentPath}}`、`{{AttachmentUrl}}`、
`{{AttachmentContentType}}` 和 `{{AttachmentDir}}`。當附件位置很重要時，
請使用 `{{AttachmentIndex}}`。

針對本機媒體讀取政策，請從
`openclaw/plugin-sdk/media-local-roots` 匯入 `getAgentScopedMediaLocalRoots(...)` 或
`getAgentScopedMediaLocalRootsForSources(...)`。
`openclaw/plugin-sdk/agent-media-payload` 外觀介面及其
`buildAgentMediaPayload(...)` 投影均已淘汰。

## 如何遷移

<Steps>
  <Step title="遷移執行階段設定載入／寫入輔助工具">
    綁定的外掛應停止直接呼叫 `api.runtime.config.loadConfig()` 和
    `api.runtime.config.writeConfigFile(...)`。請優先使用已傳入目前呼叫路徑的
    設定。需要目前處理程序快照的長期執行處理常式可以使用
    `api.runtime.config.current()`。長期執行的代理程式工具應在 `execute` 內讀取
    `ctx.getRuntimeConfig()`，如此一來，在設定寫入前建立的工具仍可取得重新整理後的設定。

    設定寫入會透過交易式輔助工具進行，並使用明確的
    寫入後政策：

    ```typescript
    await api.runtime.config.mutateConfigFile({
      afterWrite: { mode: "auto" },
      mutate(draft) {
        draft.plugins ??= {};
      },
    });
    ```

    當變更需要
    乾淨地重新啟動閘道時，使用 `afterWrite: { mode: "restart", reason: "..." }`；只有在呼叫端負責後續處理，且刻意停用
    重新載入規劃器時，才使用 `afterWrite: { mode: "none", reason: "..." }`。
    變更結果包含具型別的 `followUp` 摘要，供測試與記錄使用；
    閘道仍負責套用或排程重新啟動。

    `loadConfig` 和 `writeConfigFile` 已從外掛
    執行階段移除。內建外掛與儲存庫執行階段程式碼受到
    `pnpm check:deprecated-api-usage` 和
    `pnpm check:no-runtime-action-load-config` 的防護：新的正式環境外掛用法
    會直接失敗、直接寫入設定會失敗、閘道伺服器方法必須使用
    請求執行階段快照、執行階段頻道傳送／動作／用戶端輔助函式
    必須從其邊界接收設定，而長期存續的執行階段模組
    不允許任何環境式 `loadConfig()` 呼叫。

    新的外掛程式碼應避免使用廣泛的 `openclaw/plugin-sdk/config-runtime`
    桶狀匯出。請針對工作使用範圍較窄的子路徑：

    | 需求 | 匯入 |
    | --- | --- |
    | `OpenClawConfig` 等設定型別 | `openclaw/plugin-sdk/config-contracts` |
    | 外掛進入點設定查詢 | `api.pluginConfig` |
    | 設定合併 | 設定邊界上的外掛本機邏輯 |
    | 目前執行階段快照讀取 | `openclaw/plugin-sdk/runtime-config-snapshot` |
    | 設定寫入 | `openclaw/plugin-sdk/config-mutation` |
    | 工作階段存放區輔助函式 | `openclaw/plugin-sdk/session-store-runtime` |
    | Markdown 表格設定 | `openclaw/plugin-sdk/markdown-table-runtime` |
    | 群組原則執行階段輔助函式 | `openclaw/plugin-sdk/runtime-group-policy` |
    | 密鑰輸入解析 | `openclaw/plugin-sdk/secret-input-runtime` |
    | 模型／工作階段覆寫 | `openclaw/plugin-sdk/model-session-runtime` |

    內建外掛及其測試受到掃描器防護，不得使用廣泛的
    桶狀匯出，使匯入與模擬都維持在所需行為的本機範圍內。此
    桶狀匯出仍為外部相容性而存在，但新程式碼不應
    依賴它。

  </Step>

  <Step title="將嵌入式工具結果擴充功能遷移至中介軟體">
    內建外掛必須將僅限嵌入式執行器的
    `api.registerEmbeddedExtensionFactory(...)` 工具結果處理常式替換為
    不依賴執行階段的中介軟體：

    ```typescript
    // OpenClaw 執行階段工具與 Codex 執行階段動態工具（結果可能會被
    // 轉換）。Codex 原生工具結果也會被轉送以供觀察，
    // 但其轉換後的輸出絕不會傳達給模型：Codex
    // PostToolUse 鉤子合約無法取代原生工具回應。
    api.registerAgentToolResultMiddleware(async (event) => {
      return compactToolResult(event);
    }, {
      runtimes: ["openclaw", "codex"],
    });
    ```

    同時更新外掛資訊清單：

    ```json
    {
      "contracts": {
        "agentToolResultMiddleware": ["openclaw", "codex"]
      }
    }
    ```

    已安裝的外掛在明確啟用，且每個目標執行階段都已於
    `contracts.agentToolResultMiddleware` 中宣告時，也可以註冊工具結果中介軟體。
    未宣告的已安裝中介軟體註冊會遭到拒絕。

  </Step>

  <Step title="將原生核准處理常式遷移至能力事實">
    具核准能力的頻道外掛透過
    `approvalCapability.nativeRuntime` 加上共用執行階段情境
    登錄檔，公開原生核准行為：

    - 將 `approvalCapability.handler.loadRuntime(...)` 替換為
      `approvalCapability.nativeRuntime`。
    - 將核准專用的驗證／傳遞從舊版 `plugin.auth` /
      `plugin.approvals` 配線移至 `approvalCapability`。
    - `ChannelPlugin.approvals` 已從公開
      頻道外掛合約中移除；請將傳遞／原生／算繪欄位移至
      `approvalCapability`。
    - `plugin.auth` 僅保留供頻道登入／登出流程使用；核心不再
      從該處讀取核准驗證鉤子。
    - 透過 `openclaw/plugin-sdk/channel-runtime-context` 註冊頻道擁有的執行階段物件（用戶端、權杖、Bolt 應用程式）。
    - 請勿從原生核准處理常式傳送外掛擁有的重新路由通知；
      核心會依實際傳遞結果負責傳送已路由至其他位置的通知。
    - 將 `channelRuntime` 傳入 `createChannelManager(...)` 時，請提供
      真正的 `createPluginRuntime().channel` 介面，不完整的虛設常式會
      遭到拒絕。

    如需目前的核准能力配置，請參閱[頻道外掛](/zh-TW/plugins/sdk-channel-plugins)。

  </Step>

  <Step title="稽核 Windows 包裝函式的後援行為">
    如果你的外掛使用 `openclaw/plugin-sdk/windows-spawn`，無法解析的 Windows
    `.cmd`/`.bat` 包裝函式現在會採取封閉式失敗，除非你明確傳入
    `allowShellFallback: true`：

    ```typescript
    // 之前
    const program = applyWindowsSpawnProgramPolicy({ candidate });

    // 之後
    const program = applyWindowsSpawnProgramPolicy({
      candidate,
      // 僅為刻意接受由 shell 媒介後援的受信任相容性呼叫端
      // 設定此項目。
      allowShellFallback: true,
    });
    ```

    如果你的呼叫端並非刻意依賴 shell 後援，請勿設定
    `allowShellFallback`，而應改為處理拋出的錯誤。

  </Step>

  <Step title="尋找已淘汰的匯入">
    ```bash
    grep -r "plugin-sdk/compat" my-plugin/
    grep -r "plugin-sdk/infra-runtime" my-plugin/
    grep -r "plugin-sdk/config-runtime" my-plugin/
    grep -r "openclaw/extension-api" my-plugin/
    ```
  </Step>

  <Step title="替換為聚焦的匯入">
    舊介面的每個匯出都對應至特定的現代匯入路徑：

    ```typescript
    // 之前（已淘汰的向後相容層）
    import {
      createChannelReplyPipeline,
      createPluginRuntimeStore,
      resolveControlCommandGate,
    } from "openclaw/plugin-sdk/compat";

    // 之後（現代的聚焦匯入）
    import { createChannelReplyPipeline } from "openclaw/plugin-sdk/channel-reply-pipeline";
    import { createPluginRuntimeStore } from "openclaw/plugin-sdk/runtime-store";
    import { resolveControlCommandGate } from "openclaw/plugin-sdk/command-auth";
    ```

    對於主機端輔助函式，請使用注入的外掛執行階段，而不是
    直接匯入：

    ```typescript
    // 之前（已淘汰的 extension-api 橋接）
    import { runEmbeddedAgent } from "openclaw/extension-api";
    const result = await runEmbeddedAgent({ sessionId, prompt });

    // 之後（注入的執行階段）
    const result = await api.runtime.agent.runEmbeddedAgent({ sessionId, prompt });
    ```

    其他舊版橋接輔助函式也採用相同模式：

    | 舊匯入 | 現代對應項目 |
    | --- | --- |
    | `resolveAgentDir` | `api.runtime.agent.resolveAgentDir` |
    | `resolveAgentWorkspaceDir` | `api.runtime.agent.resolveAgentWorkspaceDir` |
    | `resolveAgentIdentity` | `api.runtime.agent.resolveAgentIdentity` |
    | `resolveThinkingDefault` | `api.runtime.agent.resolveThinkingDefault` |
    | `resolveAgentTimeoutMs` | `api.runtime.agent.resolveAgentTimeoutMs` |
    | `ensureAgentWorkspace` | `api.runtime.agent.ensureAgentWorkspace` |
    | 工作階段存放區輔助函式 | `api.runtime.agent.session.*` |

  </Step>

  <Step title="替換廣泛的 infra-runtime 匯入">
    `openclaw/plugin-sdk/infra-runtime` 仍為外部
    相容性而存在，但新程式碼應匯入實際
    需要的聚焦介面：

    | 需求 | 匯入 |
    | --- | --- |
    | 系統事件佇列輔助函式 | `openclaw/plugin-sdk/system-event-runtime` |
    | 心跳偵測喚醒、事件與可見性輔助函式 | `openclaw/plugin-sdk/heartbeat-runtime` |
    | 清空待處理傳遞佇列 | `openclaw/plugin-sdk/delivery-queue-runtime` |
    | 頻道活動遙測 | `openclaw/plugin-sdk/channel-activity-runtime` |
    | 記憶體內與持久化後端的重複資料刪除快取 | `openclaw/plugin-sdk/dedupe-runtime` |
    | 安全的本機檔案／媒體路徑輔助函式 | `openclaw/plugin-sdk/file-access-runtime` |
    | 可感知分派器的擷取 | `openclaw/plugin-sdk/runtime-fetch` |
    | Proxy 與受防護的擷取輔助函式 | `openclaw/plugin-sdk/fetch-runtime` |
    | SSRF 分派器原則型別 | `openclaw/plugin-sdk/ssrf-dispatcher` |
    | 核准請求／解析型別 | `openclaw/plugin-sdk/approval-runtime` |
    | 核准回覆承載資料與命令輔助函式 | `openclaw/plugin-sdk/approval-reply-runtime` |
    | 錯誤格式化輔助函式 | `openclaw/plugin-sdk/error-runtime` |
    | 等待傳輸就緒 | `openclaw/plugin-sdk/transport-ready-runtime` |
    | 安全權杖輔助函式 | `openclaw/plugin-sdk/secure-random-runtime` |
    | 有界非同步工作並行 | `openclaw/plugin-sdk/concurrency-runtime` |
    | 可證明不變條件的必要值斷言 | `openclaw/plugin-sdk/expect-runtime` |
    | 數值強制轉型 | `openclaw/plugin-sdk/number-runtime` |
    | 程序本機非同步鎖定 | `openclaw/plugin-sdk/async-lock-runtime` |
    | 檔案鎖定 | `openclaw/plugin-sdk/file-lock` |

    內建外掛受到掃描器防護，不得使用 `infra-runtime`，因此儲存庫程式碼
    無法退回使用廣泛的桶狀匯出。

  </Step>

  <Step title="遷移頻道路由輔助函式">
    新的頻道路由程式碼使用 `openclaw/plugin-sdk/channel-route`。較舊的
    路由鍵名稱仍保留為相容性別名：

    | 舊輔助函式 | 現代輔助函式 |
    | --- | --- |
    | `channelRouteIdentityKey(...)` | `channelRouteDedupeKey(...)` |
    | `channelRouteKey(...)` | `channelRouteCompactKey(...)` |

    現代路由輔助函式會在原生核准、回覆抑制、輸入重複資料刪除、
    排程傳遞與工作階段路由之間，一致地正規化 `{ channel, to, accountId, threadId }`。

    請勿新增使用
    `plugin-sdk/channel-route` 中的 `ChannelMessagingAdapter.parseExplicitTarget` 或
    `resolveChannelRouteTargetWithParser(...)`，這些項目已淘汰，僅為較舊的
    外掛保留。新的頻道外掛應使用
    `messaging.targetResolver.resolveTarget(...)` 進行目標 ID 正規化
    與目錄查無結果時的後援；核心需要提早取得對等端種類時，使用
    `messaging.inferTargetChatType(...)`；而提供者原生
    工作階段與討論串身分則使用 `messaging.resolveOutboundSessionRoute(...)`。

  </Step>

  <Step title="建置與測試">
    ```bash
    pnpm build
    pnpm test my-plugin/
    ```
  </Step>
</Steps>

## 匯入路徑參考

公開套件匯出對應表是可匯入 SDK
子路徑的唯一事實來源。請使用 [SDK 概覽](/zh-TW/plugins/sdk-overview)
所連結的主題式 SDK 指南，並優先採用範圍最窄且有文件記載的公開子路徑。
`scripts/lib/plugin-sdk-entrypoints.json` 中的編譯器清單也包含用於
建置內建外掛的私有本機項目；它們出現在該處並不表示其為公開套件匯出。

此表是常見的遷移子集，而非完整的 SDK 介面。編譯器進入點清單位於
`scripts/lib/plugin-sdk-entrypoints.json`；套件匯出則由公開子集產生。

為內建外掛保留的輔助接縫已從公開 SDK
匯出對應表中淘汰，但明確記載的相容性 facade 除外，例如保留給仍
直接匯入已發布 `@openclaw/discord` 套件之外部外掛的
已淘汰 `plugin-sdk/discord` 相容層。擁有者專用
輔助函式位於所屬外掛套件內；共用主機行為則透過
`plugin-sdk/gateway-runtime`、
`plugin-sdk/security-runtime` 與注入的外掛 API 等通用 SDK 合約傳遞。

請使用符合工作需求且範圍最窄的匯入。如果找不到匯出，
請檢查 `src/plugin-sdk/` 的原始碼，或詢問維護者應由哪個通用
合約負責。

## 已移除的相容性介面

2026 年 7 月的清理移除了根 SDK 與 compat 桶狀匯出、擴充功能 API
橋接、已到期的 SDK 子路徑別名、未使用的 SDK 子路徑，以及僅供內建使用之 SDK 模組的公開
匯出。僅供內建使用的模組仍可由其儲存庫擁有者透過私有本機
建置對應使用；但無法從已發布的套件匯入。

### 程序全域 API 提供者發布

`registerApiProvider(...)` 和 `unregisterApiProviders(...)` 已從
`openclaw/plugin-sdk/llm` 移除。它們會將 API 傳輸發布至程序全域
狀態，導致由生命週期擁有的模型執行階段之後必須將其複製到每個準備完成的
登錄檔中。

提供者外掛應透過 `api.registerProvider(...)` 註冊文字推論提供者。
建立 `ApiRegistry` 的主機擁有程式碼與測試，應直接在該登錄檔上
註冊，使提供者擁有權與拆卸作業都維持在準備完成之執行階段的範圍內。

### 私有測試桶狀匯出

`openclaw/plugin-sdk/testing` 僅供儲存庫本機使用，且不包含在發布的套件
成品中，因此已在其 2026-07-28 `removeAfter` 日期前移除。儲存庫
測試會使用 `plugin-sdk/plugin-test-runtime`、
`plugin-sdk/channel-test-helpers`、`plugin-sdk/channel-target-testing`、
`plugin-sdk/test-env` 和 `plugin-sdk/test-fixtures` 等聚焦子路徑。

## 遷移參考

  這些對應關係涵蓋已於 2026 年 7 月移除的介面，以及較晚時程中仍在進行的
  淘汰項目。對應關係是遷移指引，不代表舊
  介面仍然可用；請查閱相容性登錄檔與移除
  時程表以瞭解目前狀態。

  <AccordionGroup>
  <Accordion title="command-auth 說明建構器 -> command-status">
    **舊版（`openclaw/plugin-sdk/command-auth`）**：`buildCommandsMessage`、
    `buildCommandsMessagePaginated`、`buildHelpMessage`。

    **新版（`openclaw/plugin-sdk/command-status`）**：簽章相同，改從
    範圍較窄的子路徑匯入。`command-auth` 相容性重新匯出
    已遭移除。

    ```typescript
    // 之前
    import { buildHelpMessage } from "openclaw/plugin-sdk/command-auth";

    // 之後
    import { buildHelpMessage } from "openclaw/plugin-sdk/command-status";
    ```

  </Accordion>

  <Accordion title="提及閘控輔助函式 -> resolveInboundMentionDecision">
    **舊版**：來自
    `openclaw/plugin-sdk/channel-inbound` 或
    `openclaw/plugin-sdk/channel-mention-gating` 的 `resolveMentionGating(params)` 與
    `resolveMentionGatingWithBypass(params)`。

    **新版**：`resolveInboundMentionDecision({ facts, policy })`——使用單一決策
    物件，取代兩種分開的呼叫形式。

    Discord、iMessage、Matrix、MS Teams、QQ Bot、Signal、
    Telegram、WhatsApp 與 Zalo 均已採用。Slack 自有的 `app_mention` 事件模型
    不使用此輔助函式。

  </Accordion>

  <Accordion title="頻道執行階段相容層與頻道動作輔助函式">
    `openclaw/plugin-sdk/channel-runtime` 已遭移除。請使用
    `openclaw/plugin-sdk/channel-runtime-context` 註冊執行階段
    物件。

    `openclaw/plugin-sdk/channel-actions` 中的原生訊息結構描述輔助函式
    已與原始的 “actions” 頻道匯出一併移除。請改為透過語意化的
    `presentation` 介面公開功能——頻道外掛應宣告其呈現的內容
    （卡片、按鈕、選取項目），而不是其接受的原始
    動作名稱。

  </Accordion>

  <Accordion title="網頁搜尋供應商 tool() 輔助函式 -> 外掛上的 createTool()">
    **舊版**：來自 `openclaw/plugin-sdk/provider-web-search` 的 `tool()` 工廠函式。

    **新版**：直接在供應商外掛上實作 `createTool(...)`。
    OpenClaw 不再需要 SDK 輔助函式來註冊工具包裝函式。

  </Accordion>

  <Accordion title="純文字頻道封裝 -> BodyForAgent">
    **舊版**：使用 `api.runtime.channel.reply.formatInboundEnvelope(...)`（以及輸入訊息物件上的
    `channelEnvelope` 欄位），從輸入頻道訊息建立扁平的
    純文字提示封裝。

    **新版**：`BodyForAgent` 加上結構化使用者情境區塊。頻道
    外掛會將路由中繼資料（討論串、主題、回覆目標、回應）附加為
    具型別的欄位，而非串接到提示字串中。
    `formatAgentEnvelope(...)` 輔助函式仍支援合成的
    助理端封裝，但輸入純文字封裝正逐步淘汰。

    受影響的區域：`inbound_claim`、`message_received`，以及任何會對
    舊封裝文字進行後處理的自訂
    頻道外掛。

  </Accordion>

  <Accordion title="deactivate 鉤子 -> gateway_stop">
    **舊版**：`api.on("deactivate", handler)`。

    **新版**：`api.on("gateway_stop", handler)`。關閉時的清理
    合約相同；只有鉤子名稱變更。

    ```typescript
    // 之前
    api.on("deactivate", async (event, ctx) => {
      await stopPluginService(ctx);
    });

    // 之後
    api.on("gateway_stop", async (event, ctx) => {
      await stopPluginService(ctx);
    });
    ```

    `deactivate` 仍以已淘汰的相容性別名接線，直到
    2026-08-16 後移除。

  </Accordion>

  <Accordion title="subagent_spawning 鉤子 -> 核心討論串繫結">
    **舊版**：`api.on("subagent_spawning", handler)` 會傳回
    `threadBindingReady` 或 `deliveryOrigin`。

    **新版**：讓核心透過頻道工作階段繫結轉接器，準備 `thread: true` 子代理
    繫結。`api.on("subagent_spawned", handler)`
    僅用於啟動後觀察。

    ```typescript
    // 之前
    api.on("subagent_spawning", async () => ({
      status: "ok",
      threadBindingReady: true,
      deliveryOrigin: { channel: "discord", to: "channel:123", threadId: "456" },
    }));

    // 之後
    api.on("subagent_spawned", async (event) => {
      await observeSubagentLaunch(event);
    });
    ```

    `subagent_spawning`、`PluginHookSubagentSpawningEvent`、
    `PluginHookSubagentSpawningResult` 與
    `SubagentLifecycleHookRunner.runSubagentSpawning(...)` 僅在外部外掛遷移期間
    保留為已淘汰的相容性介面，並將於
    2026-08-30 後移除。

  </Accordion>

  <Accordion title="供應商探索型別 -> 供應商目錄型別">
    四個探索型別別名現在只是目錄時代型別的薄層
    包裝：

    | 舊別名                    | 新型別                    |
    | ------------------------- | ------------------------- |
    | `ProviderDiscoveryOrder`  | `ProviderCatalogOrder`    |
    | `ProviderDiscoveryContext`| `ProviderCatalogContext`  |
    | `ProviderDiscoveryResult` | `ProviderCatalogResult`   |
    | `ProviderPluginDiscovery` | `ProviderPluginCatalog`   |

    這些別名與舊版 `ProviderCapabilities` 靜態集合
    已遭移除。供應商外掛
    應使用明確的供應商鉤子，例如 `buildReplayPolicy`、
    `normalizeToolSchemas` 與 `wrapStreamFn`，而非靜態物件。

  </Accordion>

  <Accordion title="思考策略鉤子 -> resolveThinkingProfile">
    **舊版**（`ProviderThinkingPolicy` 上的三個獨立鉤子）：
    `isBinaryThinking(ctx)`、`supportsXHighThinking(ctx)` 與
    `resolveDefaultThinkingLevel(ctx)`。

    **新版**：單一 `resolveThinkingProfile(ctx)`，會傳回
    `ProviderThinkingProfile`，其中包含標準的 `id`、選用的 `label`，以及
    按順位排列的層級清單。OpenClaw 會依設定檔順位，自動
    降級過時的已儲存值。

    情境包含 `provider`、`modelId`、選用的合併後 `reasoning`，
    以及選用的合併後模型 `compat` 資訊。供應商外掛可以使用這些
    目錄資訊，僅在已設定的要求合約支援時，公開模型專屬的設定檔。

    請實作一個鉤子，而非三個。舊版鉤子已遭移除。

  </Accordion>

  <Accordion title="外部驗證供應商 -> contracts.externalAuthProviders">
    **舊版**：實作外部驗證鉤子，但未在外掛資訊清單中宣告
    供應商。

    **新版**：在外掛資訊清單中宣告 `contracts.externalAuthProviders`
    **並且**實作 `resolveExternalAuthProfiles(...)`。

    ```json
    {
      "contracts": {
        "externalAuthProviders": ["anthropic", "openai"]
      }
    }
    ```

  </Accordion>

  <Accordion title="供應商環境變數查詢 -> setup.providers[].envVars">
    **舊版**資訊清單欄位：`providerAuthEnvVars: { anthropic: ["ANTHROPIC_API_KEY"] }`。

    **新版**：將相同的環境變數查詢對應到資訊清單上的 `setup.providers[].envVars`。
    這會將設定／狀態環境中繼資料整合到同一處，
    並避免僅為回應環境變數查詢而啟動外掛執行階段。

    不再接受 `providerAuthEnvVars`。

  </Accordion>

  <Accordion title="記憶體外掛註冊 -> registerMemoryCapability">
    **舊版**：三個獨立呼叫——`api.registerMemoryPromptSection(...)`、
    `api.registerMemoryFlushPlan(...)`、`api.registerMemoryRuntime(...)`。

    **新版**：記憶體狀態 API 上的一個呼叫——
    `registerMemoryCapability(pluginId, { promptBuilder, flushPlanResolver, runtime })`。

    插槽相同，僅使用單一註冊呼叫。附加式提示與語料庫輔助函式
    （`registerMemoryPromptSupplement`、`registerMemoryCorpusSupplement`）
    不受影響。

  </Accordion>

  <Accordion title="記憶體嵌入供應商 API">
    **舊版**：`api.registerMemoryEmbeddingProvider(...)` 加上
    `contracts.memoryEmbeddingProviders`。

    **新版**：`api.registerEmbeddingProvider(...)` 加上
    `contracts.embeddingProviders`。

    通用嵌入供應商合約可在記憶體之外重複使用，並且是
    新供應商所支援的路徑。記憶體專用註冊 API
    在現有供應商遷移期間，仍以已淘汰的相容性方式接線。
    外掛檢查會將非內建使用情況回報為相容性
    技術債。

  </Accordion>

  <Accordion title="原始頻道傳送結果 -> OutboundDeliveryResult">
    **舊版**：透過
    `ChannelSendRawResult` 傳回 `{ ok, messageId, error }`，並使用
    `createRawChannelSendResultAdapter(...)` 將其標準化。

    **新版**：傳回 `OutboundDeliveryResult` 欄位，並使用
    `createAttachedChannelResultAdapter(...)` 附加頻道。傳送失敗時應擲回例外，
    而不是傳回錯誤字串。原始結果型別會持續可用，直到
    下一個外掛 SDK 主要版本。

  </Accordion>

  <Accordion title="子代理工作階段訊息型別重新命名">
    仍從 `src/plugins/runtime/types.ts` 匯出的兩個舊版型別別名：

    | 舊版                          | 新版                            |
    | ----------------------------- | ------------------------------- |
    | `SubagentReadSessionParams`   | `SubagentGetSessionMessagesParams` |
    | `SubagentReadSessionResult`   | `SubagentGetSessionMessagesResult` |

    建議使用 `getSessionMessages`，而非已淘汰的執行階段方法
    `readSession`。簽章相同；舊方法會轉呼叫
    新方法。

  </Accordion>

  <Accordion title="已移除的工作階段與逐字稿檔案 API">
    工作階段／逐字稿改用 SQLite，因而移除或淘汰了會公開使用中
    `sessions.json` 儲存區、JSONL 逐字稿路徑或工作階段檔案清單的
    外掛端 API。執行階段外掛應使用工作階段識別資訊與 SDK 執行階段
    輔助函式，而非解析或修改使用中的檔案。

    | 遷移中的介面 | 替代方案 |
    | ----------------- | ----------- |
    | 已淘汰的 `loadSessionStore(...)`、`updateSessionStore(...)` 與 `resolveSessionStoreEntry(...)` | `getSessionEntry(...)`、`listSessionEntries(...)`，以及資料列層級的工作階段變更操作。 |
    | 已淘汰的 `resolveSessionFilePath(...)` | 工作階段識別資訊（`sessionKey`、`sessionId` 與 SDK 執行階段目標輔助函式），加上對目前工作階段進行操作的閘道方法。 |
    | 已移除的 `saveSessionStore(...)` | 由閘道擁有的工作階段執行階段 API；外掛程式碼應透過已記錄的執行階段／情境輔助函式要求或修改工作階段狀態，而非寫入使用中的儲存區檔案。 |
    | 已移除的 `resolveSessionTranscriptPathInDir(...)` 與 `resolveAndPersistSessionFile(...)` | 工作階段識別資訊，以及對目前工作階段進行操作的閘道方法。 |
    | `readLatestAssistantTextFromSessionTranscript(...)` | 目前執行階段情境公開的識別資訊型逐字稿讀取器；若外掛位於逐字稿擁有者路徑之外，則使用閘道歷程／工作階段方法。 |
    | `SessionTranscriptUpdate.sessionFile` | 搭配 `agentId`、`sessionKey` 與 `sessionId` 的 `SessionTranscriptUpdate.target`。 |
    | `sessionFiles` 等記憶體同步輸入 | 主機提供的識別資訊型逐字稿／工作階段來源；請勿為即時工作階段查找使用中的 JSONL 檔案。 |
    | 使用中工作階段裡名為 `transcriptPath` 或 `sessionFile` 的執行階段選項 | 攜帶不受儲存方式影響之工作階段識別資訊的 `sessionTarget`／執行階段目標物件。 |

    舊版 JSONL 逐字稿檔案作為匯入、封存、匯出與
    支援成品時仍然有效。它們不再是使用中工作階段的常態執行階段
    合約。

    以 `v2026.7.1-beta.5` 發布的官方外掛匯入了上述四個
    已淘汰的輔助函式。`openclaw/plugin-sdk/session-store-runtime` 會原封不動地保留
    該橋接至 2026-10-12；新外掛必須使用替代方案。
    `resolveStorePath(...)` 仍是受支援的 SDK 輔助函式，不屬於
    此淘汰範圍。

    `openclaw plugins inspect --all --runtime` 會回報載入錯誤或
    診斷資訊仍參照這些已移除檔案 API 的非內建外掛。
    `@openclaw/plugin-inspector` 諮詢掃描必須使用 `0.3.17` 或
    更新版本，讓外部套件掃描也能在發布前標記完整儲存區工作階段輔助函式、
    工作階段檔案路徑輔助函式、舊版逐字稿檔案目標，以及低階
    逐字稿輔助函式。

  </Accordion>

  <Accordion title="runtime.tasks.flow -> runtime.tasks.managedFlows">
    **舊版**：`runtime.tasks.flow`（單數）會傳回即時的 TaskFlow
    存取器。

    **新版**：`runtime.tasks.managedFlows` 保留受管理的 TaskFlow 變更
    執行階段，供從流程建立、更新、取消或執行子任務的外掛使用。
    當外掛只需要以 DTO 為基礎的讀取功能時，請使用 `runtime.tasks.flows`。

    ```typescript
    // 之前
    const flow = api.runtime.tasks.flow.fromToolContext(ctx);
    // 之後
    const flow = api.runtime.tasks.managedFlows.fromToolContext(ctx);
    ```

    舊版別名已於 2026 年 7 月移除。

  </Accordion>

  <Accordion title="嵌入式擴充功能工廠 -> 代理程式工具結果中介軟體">
    已於上方的[如何遷移](#how-to-migrate)中說明。為求完整，另於此處列出：
    已移除、僅供嵌入式執行器使用的
    `api.registerEmbeddedExtensionFactory(...)` 路徑，已由
    `api.registerAgentToolResultMiddleware(...)` 取代，並在
    `contracts.agentToolResultMiddleware` 中明確指定執行階段清單。
  </Accordion>

  <Accordion title="OpenClawSchemaType 別名 -> OpenClawConfig">
    `OpenClawSchemaType` 根 SDK 別名已移除。請使用標準
    `OpenClawConfig` 名稱。

    ```typescript
    // 之前
    import type { OpenClawSchemaType } from "openclaw/plugin-sdk";
    // 之後
    import type { OpenClawConfig } from "openclaw/plugin-sdk/config-contracts";
    ```

  </Accordion>
</AccordionGroup>

<Note>
擴充功能層級的棄用項目（位於
`extensions/` 下的內建頻道／供應商外掛內）會在其各自的 `api.ts` 與 `runtime-api.ts`
匯出入口中追蹤。這些項目不影響第三方外掛合約，因此未列於此處。
如果你直接使用內建外掛的本機匯出入口，升級前請先閱讀該匯出入口中的
棄用註解。
</Note>

## Talk 與即時語音遷移

即時語音、電話、會議及瀏覽器 Talk 程式碼共用一個由
`openclaw/plugin-sdk/realtime-voice` 匯出的 Talk 工作階段控制器。此
控制器擁有共用的 Talk 事件封裝、作用中輪次狀態、擷取
狀態、輸出音訊狀態、近期事件歷程及過期輪次拒絕機制。
供應商外掛擁有供應商專屬的即時工作階段。瀏覽器會議外掛
使用 `openclaw/plugin-sdk/meeting-runtime` 處理工作階段、瀏覽器、音訊、節點主機、
代理程式諮詢及語音通話機制，接著實作 `MeetingPlatformAdapter`
以處理 URL 規則、DOM 指令碼、手動動作對應、字幕、建立及撥入
方案。平台 REST API、OAuth、成品、選擇器及連線名稱仍保留在
外掛中。瀏覽器權限方案會收到所要求的會議 URL，讓各
平台僅授予其確切支援的來源。工作階段執行階段也必須在確認瀏覽器離開後，
正規化平台專屬的即時健康狀態；
歷史逐字稿欄位可以保留，但離開後字幕與音訊就緒狀態
不得繼續保持作用中。

所有內建介面都在共用控制器上執行：瀏覽器轉送、
受管理房間移交、語音通話即時處理、語音通話串流 STT、Google
Meet 即時處理及原生按下通話。閘道會在 `hello-ok.features.events` 中公布一個即時 Talk 事件
頻道：`talk.event`。

除非正在實作低階介接器或測試固定資料，否則新程式碼不應直接呼叫
`createTalkEventSequencer(...)`。請使用共用控制器，確保
輪次範圍事件無法在沒有輪次 ID 的情況下發出、過期的 `turnEnd` /
`turnCancel` 呼叫無法清除較新的作用中輪次，且輸出音訊
生命週期事件在電話、會議、瀏覽器轉送、
受管理房間移交及原生 Talk 用戶端之間保持一致。

公開 API 形式：

```typescript
// 閘道擁有的 Talk 工作階段 API。
await gateway.request("talk.session.create", {
  mode: "realtime",
  transport: "gateway-relay",
  brain: "agent-consult",
  sessionKey: "main",
});
await gateway.request("talk.session.appendAudio", { sessionId, audioBase64 });
await gateway.request("talk.session.cancelOutput", { sessionId, reason: "barge-in" });
await gateway.request("talk.session.submitToolResult", {
  sessionId,
  callId,
  result: { status: "working" },
  options: { willContinue: true },
});
await gateway.request("talk.session.submitToolResult", {
  sessionId,
  callId,
  result: { status: "already_delivered" },
  options: { suppressResponse: true },
});
await gateway.request("talk.session.submitToolResult", { sessionId, callId, result });
await gateway.request("talk.session.close", { sessionId });

// 用戶端擁有的供應商工作階段 API。
await gateway.request("talk.client.create", {
  mode: "realtime",
  transport: "webrtc",
  brain: "agent-consult",
  sessionKey: "main",
});
await gateway.request("talk.client.toolCall", { sessionKey, callId, name, args });
await gateway.request("talk.client.steer", { sessionKey, text, mode: "steer" });
```

由瀏覽器擁有的 WebRTC／供應商 WebSocket 工作階段使用 `talk.client.create`，
因為瀏覽器擁有供應商協商與媒體傳輸，而
閘道擁有認證資訊、指示及工具政策。`talk.session.*` 是
由閘道管理的共用介面，用於閘道轉送即時處理、閘道轉送
轉錄，以及受管理房間的原生 STT/TTS 工作階段。

將即時選擇器與 `talk.provider` /
`talk.providers` 並列的舊版設定，應使用 `openclaw doctor --fix` 修復；Talk 執行階段
不會將語音／TTS 供應商設定重新解讀為即時供應商設定。

支援的 `talk.session.create` 組合刻意維持精簡：

| 模式            | 傳輸       | 核心           | 擁有者              | 備註                                                                                                              |
| --------------- | --------------- | --------------- | ------------------ | ------------------------------------------------------------------------------------------------------------------ |
| `realtime`      | `gateway-relay` | `agent-consult` | 閘道            | 透過閘道橋接的全雙工供應商音訊；工具呼叫會透過代理程式諮詢工具路由。           |
| `transcription` | `gateway-relay` | `none`          | 閘道            | 僅串流 STT；呼叫端傳送輸入音訊並接收逐字稿事件。                                        |
| `stt-tts`       | `managed-room`  | `agent-consult` | 原生／用戶端房間 | 按下通話及對講機形式的房間，由用戶端擁有擷取／播放，閘道擁有輪次狀態。 |
| `stt-tts`       | `managed-room`  | `direct-tools`  | 原生／用戶端房間 | 僅限管理員的房間模式，供直接執行閘道工具動作的受信任第一方介面使用。                  |

供從較舊的 `talk.realtime.*` /
`talk.transcription.*` / `talk.handoff.*` 系列（均已移除）遷移的讀者參考的方法對照表：

| 舊                              | 新                                                      |
| -------------------------------- | -------------------------------------------------------- |
| `talk.realtime.session`          | `talk.client.create`                                     |
| `talk.realtime.toolCall`         | `talk.client.toolCall`                                   |
| `talk.realtime.relayAudio`       | `talk.session.appendAudio`                               |
| `talk.realtime.relayCancel`      | `talk.session.cancelOutput` 或 `talk.session.cancelTurn` |
| `talk.realtime.relayToolResult`  | `talk.session.submitToolResult`                          |
| `talk.realtime.relayStop`        | `talk.session.close`                                     |
| `talk.transcription.session`     | `talk.session.create({ mode: "transcription" })`         |
| `talk.transcription.relayAudio`  | `talk.session.appendAudio`                               |
| `talk.transcription.relayCancel` | `talk.session.cancelTurn`                                |
| `talk.transcription.relayStop`   | `talk.session.close`                                     |
| `talk.handoff.create`            | `talk.session.create({ transport: "managed-room" })`     |
| `talk.handoff.join`              | `talk.session.join`                                      |
| `talk.handoff.revoke`            | `talk.session.close`                                     |

統一控制詞彙也刻意限制在精簡範圍內：

| 方法                          | 適用於                                              | 合約                                                                                                                                                                                                                  |
| ------------------------------- | ------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `talk.session.appendAudio`      | `realtime/gateway-relay`、`transcription/gateway-relay` | 將 base64 PCM 音訊區塊附加至由同一個閘道連線擁有的供應商工作階段。                                                                                                                             |
| `talk.session.startTurn`        | `stt-tts/managed-room`                                  | 開始受管理房間的使用者輪次。                                                                                                                                                                                           |
| `talk.session.endTurn`          | `stt-tts/managed-room`                                  | 通過過期輪次驗證後結束作用中輪次。                                                                                                                                                                          |
| `talk.session.cancelTurn`       | 所有由閘道擁有的工作階段                              | 取消某輪次作用中的擷取／供應商／代理程式／TTS 工作。                                                                                                                                                                 |
| `talk.session.cancelOutput`     | `realtime/gateway-relay`                                | 停止助理音訊輸出，而不一定要結束使用者輪次。                                                                                                                                                     |
| `talk.session.submitToolResult` | `realtime/gateway-relay`                                | 在其橋接層公開的任何非同步完成作業結束後，完成供應商工具呼叫；傳入 `options.willContinue` 以取得中繼輸出，或在支援時傳入 `options.suppressResponse`，以避免再次產生助理回應。 |
| `talk.session.steer`            | 由代理程式支援的 Talk 工作階段                              | 將語音 `status`、`steer`、`cancel` 或 `followup` 控制傳送至從 Talk 工作階段解析出的作用中嵌入式執行。                                                                                                 |
| `talk.session.close`            | 所有統一工作階段                                    | 停止轉送工作階段或撤銷受管理房間狀態，然後清除統一工作階段 ID。                                                                                                                                     |

請勿為了讓此功能運作而在核心中加入供應商或平台特例。
核心擁有 Talk 工作階段語意。供應商外掛擁有供應商工作階段設定。
語音通話與 Google Meet 擁有電話／會議介接器。瀏覽器與原生
應用程式擁有裝置擷取／播放使用者體驗。

## 移除時程

| 時間                                        | 會發生什麼事                                                                                                                              |
| ------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------- |
| **現在**                                     | 可發出警告的已棄用介面會在執行階段發出警告；儲存庫防護會拒絕核心與隨附外掛匯入已棄用的 SDK。 |
| **等待擁有者決定**                  | 未標示日期的記錄會維持棄用狀態，且無法移除，直到其擁有者發布 `removeAfter` 日期為止。                          |
| **各相容性記錄的 `removeAfter` 日期** | 該特定介面可移除；日期一過，`pnpm plugins:boundary-report --fail-on-eligible-compat` 就會使 CI 失敗。    |
| **下一個主要版本**                      | 已標示日期的介面只能在其 `removeAfter` 日期之後移除；未標示日期的記錄仍需擁有者核准並發布日期。   |

下列其餘公開 SDK 子路徑具有由登錄檔支援的移除時程。
7 月 30 日的資料列已在維護者提前授權的清理後移除：
未使用的子路徑已刪除、較早的相容性別名已刪除，且
僅供隨附模組使用的模組已降級為私有本機建置對應。

| `removeAfter` | 層級                               | SDK 子路徑                                                                                                                                                                        |
| ------------- | ---------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `2026-08-15`  | 較早的相容性棄用項目 | `agent-config-primitives`、`channel-logging`、`channel-secret-runtime`、`channel-streaming`、`group-access`、`inbound-reply-dispatch`、`matrix`、`text-runtime`、`zod`              |
| `2026-09-01`  | 較早的相容性棄用項目 | `channel-lifecycle`、`channel-message`、`channel-reply-pipeline`、`config-runtime`、`infra-runtime`                                                                                 |
| `2026-10-01`  | 媒體舊版投影            | `agent-media-payload`，以及非子路徑的 `MsgContext Media*` 欄位、頻道傳入媒體承載資料建構器、`buildMediaPayload`、鉤子媒體別名與 `{{Media*}}` 範本 |

所有核心外掛皆已完成遷移。外部外掛應在
下一個主要版本之前遷移。執行 `pnpm plugins:boundary-report`，即可查看你的外掛所使用介面中
哪些相容性記錄最接近到期。

## 暫時抑制警告

```bash
OPENCLAW_SUPPRESS_PLUGIN_SDK_COMPAT_WARNING=1 openclaw gateway run
OPENCLAW_SUPPRESS_EXTENSION_API_WARNING=1 openclaw gateway run
```

這是暫時的緊急應變機制，並非永久解決方案。

## 相關內容

- [開始使用](/zh-TW/plugins/building-plugins) - 建置你的第一個外掛
- [SDK 概觀](/zh-TW/plugins/sdk-overview) - 完整的子路徑匯入參考
- [頻道外掛](/zh-TW/plugins/sdk-channel-plugins) - 建置頻道外掛
- [供應商外掛](/zh-TW/plugins/sdk-provider-plugins) - 建置供應商外掛
- [外掛內部機制](/zh-TW/plugins/architecture) - 深入探討架構
- [外掛資訊清單](/zh-TW/plugins/manifest) - 資訊清單結構描述參考
