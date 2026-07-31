---
read_when:
    - 你正在變更內嵌的代理執行階段或框架登錄表
    - 你正在從內建或受信任的外掛註冊代理程式工具框架
    - 你需要瞭解 Codex 外掛與模型提供者之間的關係
sidebarTitle: Agent Harness
summary: 供取代低階內嵌代理程式執行器之外掛使用的實驗性 SDK 介面
title: 代理程式框架外掛
x-i18n:
    generated_at: "2026-07-26T08:44:17Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 4ff4e41a46ba0074fc6c8bf46da813b58d074f5e6c5c1d236d7ab78e824bdc02
    source_path: plugins/sdk-agent-harness.md
    workflow: 16
---

**代理執行框架**是 OpenClaw 代理單次已準備回合的低階執行器。它不是模型供應商、不是頻道，也不是工具登錄庫。如需面向使用者的心智模型，請參閱[代理執行階段](/zh-TW/concepts/agent-runtimes)。

此介面僅供內建或受信任的原生外掛使用。此合約仍屬實驗性質，因為參數型別刻意對應目前的內嵌執行器。

## 何時使用執行框架

當模型系列擁有自己的原生工作階段執行階段，而一般 OpenClaw 供應商傳輸並非正確的抽象層時，請註冊代理執行框架：

- 擁有執行緒與壓縮功能的原生程式設計代理伺服器
- 必須串流原生計畫／推理／工具事件的本機命令列介面或常駐程式
- 除了 OpenClaw 工作階段逐字稿之外，還需要自有繼續執行 ID 的模型執行階段

請**不要**只為了新增 LLM API 而註冊執行框架。對於一般 HTTP 或 WebSocket 模型 API，請建立[供應商外掛](/zh-TW/plugins/sdk-provider-plugins)。

## 核心仍負責的項目

在選取執行框架之前，OpenClaw 已解析：

- 供應商與模型
- 執行階段驗證狀態，除非執行框架宣告由它負責驗證啟動程序
- 思考層級與情境預算
- OpenClaw 逐字稿／工作階段檔案
- 工作區、沙箱與工具政策
- 頻道回覆回呼與串流回呼
- 模型後援與即時模型切換政策

執行框架會執行已準備好的嘗試；它不會選擇供應商、取代頻道傳遞，或在未告知的情況下切換模型。

### 執行框架負責的驗證啟動程序

依預設，核心會先解析供應商認證資訊，再呼叫執行框架。可透過自身原生執行階段進行驗證的受信任執行框架，可在其靜態 `AgentHarness` 註冊上設定 `authBootstrap: "harness"`。之後，對於該執行框架承接的每次嘗試，核心都會略過通用供應商認證資訊啟動程序及缺少認證資訊的失敗處理。

若存在相容且已明確選取或排序的 OpenClaw 驗證設定檔及其限定範圍的儲存區，核心仍會轉交它們。執行框架必須在發出模型要求前解析該設定檔或其原生認證資訊、將機密限定於該次嘗試，並顯示可採取行動的驗證失敗資訊。若執行框架只在部分情況下負責驗證，請勿設定此能力。

### 已驗證的設定執行階段成品

能為首次執行設定提供推論功能的本機執行框架，必須證明完成探查的實作。當 `params.captureRuntimeArtifact` 為 true 時，請傳回具有穩定 ID 與內容指紋的不透明 `result.runtimeArtifact`。請註冊相符的 `runtimeArtifact.validate(...)` 能力，以便在不載入其他執行框架或掃描無關外掛的情況下重新檢查該繫結。

經過驗證的 OpenClaw 繼續執行也會傳入 `params.expectedRuntimeArtifact`。執行框架必須將其與實際取得的原生程序比較；若兩者不同，必須在啟動或繼續原生執行緒前失敗。一般代理回合會省略這兩個欄位，因此內容雜湊不會進入一般要求的熱路徑。遠端／WebSocket 執行框架必須先具備伺服器證明合約才能參與；僅有版本字串不足以識別成品。

已準備的嘗試也包含 `params.runtimePlan`，這是由 OpenClaw 擁有的政策套件，用於必須在 OpenClaw 與原生執行框架之間保持一致的執行階段決策：

- `runtimePlan.tools.normalize(...)` 與 `runtimePlan.tools.logDiagnostics(...)`，用於供應商感知的工具結構描述政策
- `runtimePlan.transcript.resolvePolicy(...)`，用於逐字稿清理與工具呼叫修復政策
- `runtimePlan.delivery.isSilentPayload(...)`，用於共用 `NO_REPLY` 與媒體傳遞抑制
- `runtimePlan.outcome.classifyRunResult(...)`，用於模型後援分類
- `runtimePlan.observability`，用於已解析的供應商／模型／執行框架中繼資料

執行框架可以將此計畫用於必須與 OpenClaw 行為一致的決策，但應將它視為主機擁有的嘗試狀態：請勿修改它，也不要用它在回合內切換供應商／模型。

### 要求傳輸合約

`supports(ctx)` 會在 `ctx.modelProvider` 中接收已解析的模型傳輸。以下兩項不含機密且由供應商擁有的事實，用於描述選定的路由：

- `runtimePolicy.compatibleIds` 列出供應商宣告與該具體路由相容的執行階段 ID。缺少政策表示供應商未宣告路由層級的相容性；這不代表可以假設支援。
- `requestTransportOverrides: "none"` 表示不需要重現任何人工設定的供應商／模型要求覆寫。`"present"` 表示存在人工設定的標頭、驗證傳輸、Proxy、TLS、本機服務、私人網路行為或要求參數。此事實不會揭露這些值。

當執行框架無法重現已準備的傳輸時，請傳回 `{ supported: false, reason }`。請勿在選取完成後透過讀取原始設定來推斷支援情況。若驗證準備產生多個重試路由，單一執行框架必須支援全部路由，才能進行分派。若沒有外掛能負責完整路由集合，隱含選取會使用 OpenClaw；明確或已保存的外掛選取則會採取失敗即關閉。

## 註冊執行框架

**匯入：** `openclaw/plugin-sdk/agent-harness`

```typescript
import type { AgentHarness } from "openclaw/plugin-sdk/agent-harness";
import { definePluginEntry } from "openclaw/plugin-sdk/plugin-entry";

const myHarness: AgentHarness = {
  id: "my-harness",
  label: "My native agent harness",

  supports(ctx) {
    const routeSupportsHarness =
      ctx.modelProvider?.runtimePolicy?.compatibleIds.includes("my-harness") === true;
    const canReproduceRequest = ctx.modelProvider?.requestTransportOverrides !== "present";
    return ctx.provider === "my-provider" && routeSupportsHarness && canReproduceRequest
      ? { supported: true, priority: 100 }
      : { supported: false, reason: "effective route is not harness-compatible" };
  },

  async runAttempt(params) {
    // 啟動或繼續你的原生執行緒。
    // 使用 params.prompt、params.tools、params.images、params.onPartialReply、
    // params.onAgentEvent，以及其他已準備的嘗試欄位。
    return await runMyNativeTurn(params);
  },
};

export default definePluginEntry({
  id: "my-native-agent",
  name: "My Native Agent",
  description: "Runs selected models through a native agent daemon.",
  register(api) {
    api.registerAgentHarness(myHarness);
  },
});
```

此通用範例刻意未包含 `authBootstrap`。只有在執行框架符合上述合約時，才可新增 `authBootstrap: "harness"`。

### 委派執行

執行框架擁有者可以將 `delegatedExecutionPluginIds` 設為需要執行現有模型鎖定工作階段的受信任外掛 ID，例如由語音傳輸繼續 Codex 支援的對話。這是擁有者的靜態同意，而非核心允許清單。請將範圍保持在最低限度。

受委派者只會取得工作接納與內嵌執行權限。OpenClaw 要求提供完全相符的已儲存工作階段金鑰、儲存區路徑與工作階段 ID；`modelSelectionLocked:
true`；以及相符的 `agentHarnessId` 與 `agentHarnessRuntimeOverride` 值。接著，執行會限定於執行框架擁有者的範圍。工作階段建立、修補、重設、刪除、封存及閘道變更仍僅限擁有者執行。

## 選取政策

OpenClaw 會在解析供應商／模型後選擇執行框架：

1. 模型範圍的執行階段政策優先。
2. 其次是供應商範圍的執行階段政策。
3. `auto` 會詢問已註冊的執行框架是否支援已解析的有效路由。僅有供應商／模型前置詞絕不會選取執行框架。
4. 若沒有相符的已註冊執行框架，OpenClaw 會使用其內嵌執行階段。

外掛執行框架失敗會顯示為執行失敗。在 `auto` 模式下，只有當沒有已註冊的外掛執行框架支援已解析的供應商／模型時，才會套用內嵌後援。一旦外掛執行框架承接某次執行，OpenClaw 就不會透過其他執行階段重播同一回合，因為這可能改變驗證／執行階段語意或重複產生副作用。

所設定的執行階段政策仍是所需執行階段的權威依據。在路由／驗證準備仍待完成時，已保存工作階段的 `agentHarnessId` 會保留其原生逐字稿的擁有權。兩者都不能讓不相容的路由變得相容：一旦準備好的事實存在，選定或固定的執行框架就必須支援它們，否則執行會採取失敗即關閉。`/status` 顯示根據政策、已保存擁有權與路由支援選定的有效執行階段。
準備狀態是明確的：缺少 `runtimePolicy` 時會維持未宣告狀態，而不會根據剛好存在的傳輸欄位進行推斷。
當執行框架負責的驗證留下多個尚未解析的實體路由時，準備好的支援事實是其相容執行階段 ID 的交集，且若任何候選項具有要求覆寫，也會加以回報。因此，只要有一個未宣告的候選項，原生相容性就會變成空集合；`preparedAuth.source: "harness"` 是驗證擁有者，並不代表可以推斷路由支援。

若選定的執行框架出乎預期，請啟用 `agents/harness` 偵錯記錄，並檢查閘道的結構化 `agent harness selected` 記錄：其中包含選定的執行框架 ID、選取原因、執行階段／後援政策，以及在 `auto` 模式下各外掛候選項的支援結果。

內建 Codex 外掛會將 `codex` 註冊為其執行框架 ID。核心會將其視為一般外掛執行框架 ID；Codex 專用別名應位於外掛或操作人員設定中，而非共用執行階段選取器中。

## 供應商與執行框架配對

大多數執行框架也應註冊供應商。供應商會讓 OpenClaw 的其餘部分看見模型參照、驗證狀態、模型中繼資料與 `/model` 選取。接著，執行框架會在 `supports(...)` 中承接該供應商。

內建 Codex 外掛遵循此模式：

- 偏好的使用者模型參照：`openai/gpt-5.6-sol`
- 相容性參照：仍接受舊版 `codex/gpt-*` 參照，但新設定不應將其用作一般供應商／模型參照
- 執行框架 ID：`codex`
- 驗證：合成的供應商可用性，因為 Codex 執行框架負責原生 Codex 登入／工作階段
- 應用程式伺服器要求：OpenClaw 會將純模型 ID 傳送給 Codex，並由執行框架與原生應用程式伺服器通訊協定通訊

Codex 外掛採用附加方式運作。若未設定執行階段政策或設為 `auto`，OpenAI 只有在其供應商擁有的路由合約宣告與 `codex` 相容時，才能選取 Codex：也就是完全相符的官方 HTTPS Platform Responses 或 ChatGPT Responses 路由，且沒有人工設定的要求覆寫。僅有 `openai/*` 前置詞絕不會選取 Codex。自訂端點、Completions 轉接器與人工設定的要求行為會繼續由 OpenClaw 處理。純文字官方 HTTP 端點會遭拒絕。較舊的 `codex/gpt-*` 參照仍保留為相容性輸入。請參閱
[OpenAI 隱含代理執行階段](/zh-TW/providers/openai#implicit-agent-runtime)。

如需操作人員設定、模型前置詞範例與 Codex 專用設定，請參閱
[Codex 執行框架](/zh-TW/plugins/codex-harness)。

Codex 外掛會強制執行 [Codex 執行框架](/zh-TW/plugins/codex-harness)中記載的最低應用程式伺服器版本。它會檢查初始化交握，並阻擋較舊或未提供版本的伺服器，以確保 OpenClaw 只會針對已測試的通訊協定介面執行。

### 工具結果中介軟體

若內建外掛及明確啟用的已安裝外掛具有相符的資訊清單合約，且其資訊清單在 `contracts.agentToolResultMiddleware` 中宣告目標執行階段 ID，便可透過 `api.registerAgentToolResultMiddleware(...)` 附加與執行階段無關的工具結果中介軟體。此受信任介面用於非同步工具結果轉換，且必須在 OpenClaw 或 Codex 將工具輸出送回模型之前執行。

舊版內建外掛仍可將
`api.registerCodexAppServerExtensionFactory(...)` 用於僅限 Codex app-server 的
中介軟體，但新的結果轉換應使用執行階段中立 API。僅限
內嵌執行器的 `api.registerEmbeddedExtensionFactory(...)` 鉤子已
移除；內嵌工具結果轉換必須使用執行階段中立中介軟體。

### 終止結果分類

擁有自身協定投影的原生框架，可在已完成的輪次未產生
可見的助理文字時，使用
`openclaw/plugin-sdk/agent-harness-runtime` 中的
`classifyAgentHarnessTerminalOutcome(...)`。此輔助函式會傳回 `empty`、`reasoning-only` 或
`planning-only`，讓 OpenClaw 的備援政策可決定是否改用
其他模型重試。`planning-only` 需要框架明確提供 `planText`
欄位；OpenClaw 不會從助理文字推斷該欄位。此輔助函式
刻意不對提示錯誤、進行中的輪次，以及 `NO_REPLY` 等刻意保持靜默的
回覆進行分類。

### 代理程式結束時的副作用

原生框架在完成一次嘗試的最終處理後，必須呼叫
`openclaw/plugin-sdk/agent-harness-runtime` 中的
`runAgentEndSideEffects(...)`。它會分派可攜式
`agent_end` 鉤子和 OpenClaw 的研究擷取作業，且不會延遲互動式回覆。
對於必須等到這些副作用完成後才能結束嘗試的
本機非互動式執行，請使用 `awaitAgentEndSideEffects(...)`。這兩個輔助函式接受與
`runAgentHarnessAgentEndHook(...)` 相同的 `{ event, ctx }` 承載資料；
其失敗不會改變已完成嘗試的結果。

### 使用者輸入與工具介面

公開執行階段層級使用者輸入要求的原生框架，應使用
`openclaw/plugin-sdk/agent-harness-runtime` 中的使用者輸入輔助函式來格式化
提示、透過 OpenClaw 的阻塞式回覆路徑傳送提示，並將
選項／自由格式答案正規化回執行階段的原生回應結構。此
輔助函式可維持頻道／終端介面的呈現一致，同時讓各框架保有自己的
協定剖析與待處理要求生命週期。

需要類似 PI 之精簡工具路由的原生框架，應使用
`openclaw/plugin-sdk/agent-harness-tool-runtime` 中的
`createAgentHarnessToolSurfaceRuntime(...)`。它負責
工具搜尋／程式碼模式控制選擇、本機模型的精簡預設值、
與執行階段相容的結構描述篩選、隱藏目錄執行、目錄
填入，以及目錄清理。框架仍負責其 SDK 專屬的工具
轉換與原生執行回呼。

### 原生 Codex 框架模式

內建的 `codex` 框架是內嵌 OpenClaw
代理程式輪次的原生 Codex 模式。請先啟用內建的 `codex` 外掛；若設定使用限制性允許清單，請在
`plugins.allow` 中加入 `codex`。原生 app-server
設定應使用 `openai/gpt-*`；只有在有效路由宣告與 Codex 相容時，OpenAI 代理程式輪次
才會選取 Codex 框架。舊版 Codex 模型
參照應使用 `openclaw doctor --fix` 修復，而舊版 `codex/*`
模型參照仍是原生框架的相容性別名。

此模式執行時，由 Codex 負責原生執行緒 ID、繼續執行行為、
壓縮及 app-server 執行。OpenClaw 仍負責聊天頻道、
可見記錄鏡像、工具政策、核准、媒體傳送和工作階段
選擇。需要證明只有 Codex app-server 路徑能接管該次執行時，請使用提供者／模型
`agentRuntime.id: "codex"`。明確指定的外掛
執行階段會採失敗即關閉；Codex app-server 選取失敗與執行階段失敗
不會透過其他執行階段重試。

## 執行階段嚴格模式

OpenClaw 預設使用 `auto` 提供者／模型執行階段政策：已註冊的
外掛框架可接管相容的有效路由，而在沒有任何框架符合時，內嵌
執行階段會處理該輪次。僅有提供者／模型前綴絕不會
選取框架。若缺少框架選取時應直接失敗，而不是
路由至內嵌執行階段，請使用明確的提供者／模型外掛執行階段，例如
`agentRuntime.id: "codex"`。明確選取不會讓
不相容的路由變得相容。選定的外掛框架失敗時一律
立即失敗。這不會阻止明確的提供者／模型
`agentRuntime.id: "openclaw"`。

僅限 Codex 的內嵌執行：

```json
{
  "models": {
    "providers": {
      "openai": {
        "agentRuntime": {
          "id": "codex"
        }
      }
    }
  },
  "agents": {
    "defaults": {
      "model": "openai/gpt-5.6-sol"
    }
  }
}
```

若要為單一標準模型使用命令列介面後端，請將執行階段放在該
模型項目中：

```json
{
  "agents": {
    "defaults": {
      "model": "anthropic/claude-opus-5",
      "models": {
        "anthropic/claude-opus-5": {
          "agentRuntime": {
            "id": "claude-cli"
          }
        }
      }
    }
  }
}
```

每個代理程式的覆寫使用相同的模型範圍結構：

```json
{
  "agents": {
    "list": [
      {
        "id": "codex-only",
        "model": "openai/gpt-5.6-sol",
        "models": {
          "openai/gpt-5.6-sol": {
            "agentRuntime": { "id": "codex" }
          }
        }
      }
    ]
  }
}
```

如下所示的舊版整體代理程式執行階段範例會被忽略：

```json
{
  "agents": {
    "defaults": {
      "agentRuntime": {
        "id": "codex"
      }
    }
  }
}
```

使用明確的外掛執行階段時，若要求的框架尚未註冊、不支援解析後的
提供者／模型，或在產生輪次副作用之前失敗，工作階段就會提早
失敗。這是 Codex 專用
部署及必須證明 Codex app-server 路徑確實正在使用中的
即時測試所刻意採用的行為。

此設定僅控制內嵌代理程式框架，不會停用
圖片、影片、音樂、TTS、PDF 或其他提供者專屬模型路由。

## 原生工作階段與記錄鏡像

框架可以保留原生工作階段 ID、執行緒 ID 或常駐程式端的繼續執行
權杖。請明確將該繫結與 OpenClaw 工作階段建立關聯，並
持續將使用者可見的助理／工具輸出鏡像至 OpenClaw
記錄。

OpenClaw 記錄仍是以下功能的相容層：

- 頻道可見的工作階段歷程
- 記錄搜尋與建立索引
- 在後續輪次切換回內建 OpenClaw 框架
- 一般的 `/new`、`/reset` 及工作階段刪除行為

若框架儲存側載繫結，請實作 `reset(...)`，讓 OpenClaw
可在重設所屬的 OpenClaw 工作階段時清除該繫結。

## 工具與媒體結果

核心會建構 OpenClaw 工具清單，並將其傳入已準備的
嘗試。框架執行動態工具呼叫時，請透過框架結果
結構傳回工具結果，而不要自行傳送頻道媒體。

這可讓文字、圖片、影片、音樂、TTS、核准及訊息工具
輸出，與 OpenClaw 支援的執行使用相同的傳送路徑。

僅針對受信任框架執行階段自行建立並保存的原生成品，才可設定
`AgentHarnessAttemptResult.hostOwnedToolMediaUrls`。每個項目也都必須
出現在 `toolMediaUrls` 中。絕不可包含模型選取的動態工具或
OpenClaw 工具媒體。在 `message_tool_only` 路由上，這種狹義的來源資訊可讓
原生執行階段成品在來源回覆遭抑制時仍能保留；一般傳送政策
與環境聊天室准入規則仍然適用。

### 終止工具結果

`AgentHarnessAttemptParams.observeToolTerminal` 是由主機擁有的終止
結果累加器。執行 OpenClaw 動態工具或原生
工具的框架，必須在每個工具達到唯一一個終止結果時、且在
嘗試結果完成最終處理之前呼叫它。不執行工具的框架無須
呼叫它。

請回報執行邊界上的事實：

- 若有協定呼叫 ID，請傳入該 ID、標準工具名稱，以及在準備或鉤子重寫後實際
  傳入工具的引數。
- 當驗證、核准或其他防護措施在工具實作開始之前
  阻止呼叫時，請設定 `executionStarted: false`。只要可能已進行分派，
  就應保守地回報 `true`。
- 回報 `outcome: "success"` 或 `outcome: "failure"`。請納入執行階段提供的結構化
  失敗欄位，而不要從顯示文字推斷失敗。
- 僅針對未使用 OpenClaw 工具
  定義的原生工具使用 `nativeMutation`。請在其中提供協定擁有的變更與重播事實；不要
  將 OpenClaw 的變更分類器複製到框架中。

此回呼會傳回該呼叫的標準解析結果。請將其
`lastToolError` 帶入 `AgentHarnessAttemptResult`，並在框架投影中使用其執行、
引數和副作用事實，而不要另外推導平行狀態。主機會在無關工具
成功後繼續保留尚未解決的變更失敗，並且只在相符動作成功後
清除該失敗。

為了與較舊的實驗性框架保持原始碼相容，此回呼仍為選用。
對會執行工具的框架而言，選用不代表可忽略：
若未回報終止結果，OpenClaw 就無法在之後的工具呼叫中保留變更工具失敗的
真實狀態，包括靜默的心跳偵測完成情況。

### 已完成工具的最終處理

當框架已完成所有
工具呼叫，但其原生輪次結束時沒有助理文字，OpenClaw 可能需要最後一個可見答案。框架可透過
實作 `finalizeSettledTurn({ attempt,
settledAttempt })` 選擇加入該復原機制。

此回呼是獨立的能力，而不是另一個一般嘗試。它必須：

- 使用精確且受限制的原生記錄，或凍結至已完成工具結果邊界的完整應用程式
  記錄；
- 不得公開任何工具、權限授予或使用者輸入能力、原生執行
  鉤子、代理程式、Skills、記憶、排程、擴充功能或遠端控制；
- 僅傳送主機提供的最終處理提示；以及
- 若其選取的記錄／隔離策略無法強制執行
  這些限制，則採失敗即關閉。

OpenClaw 會在一般
嘗試與重試迴圈之外，將此回呼作為終止子作業呼叫一次。若失敗，該次執行會以
可感知副作用的未完成輪次警告結束；它不能進入一般的
驗證／設定檔輪替、模型備援、內容復原、壓縮
接續或鉤子要求的修訂路徑。最終處理也會略過外掛
提示變更、`before_agent_run`、LLM 輸入／輸出、終止修訂及
`agent_end` 鉤子。核心診斷仍會記錄此作業及其失敗。

此回呼會傳回 `AgentHarnessSettledTurnFinalizationResult`，而不是
一般嘗試結果。其公開欄位僅限於已完成的
助理訊息、最終處理呼叫用量、記錄擁有權中繼資料及
診斷追蹤。工具、傳送、媒體、衍生、生命週期、重播、工作階段及
備援狀態都不能跨越此結果邊界。未知欄位與助理
工具呼叫會採失敗即關閉。

內部重複使用完整嘗試引擎的框架，可在傳回前呼叫
`projectSettledTurnFinalizationAttemptResult(...)`。此輔助函式
會拒絕標準失敗、工具、傳送、重播及生命週期證據，然後
僅投影狹義結果。它是原生隔離後的縱深防禦，
不能取代移除原生能力介面。

以投影為基礎的框架必須將完整內容放在
`settledAttempt.settledTurnFinalizationContext` 上，並搭配
`source: "openclaw-transcript"`。它必須在
已完成的輪次鏡像後擷取作用中分支、證明目前提示與所有目前工具
呼叫／結果皆完整包含至該邊界，並在傳回嘗試之前凍結產生的訊息
陣列。最終處理器必須拒絕缺失、不支援、模稜兩可或過大的內容。它不得截斷訊息、
捨棄較早的歷程，或將此應用程式記錄描述為精確的原生
歷程。繼續使用單一受限制原生工作階段的框架無須提供此
投影欄位。

請勿透過以盡力而為的
`disableTools` 提示呼叫 `runAttempt` 來實作此回呼。框架擁有者必須強制執行完整的原生
能力邊界。OpenClaw 不提供一般備援，因為它
無法證明任意原生執行階段有遵守這些限制。

回呼對於實驗性的第三方執行框架相容性仍為選用項目。當所選的執行框架省略回呼時，OpenClaw 會保留現有的回合未完成錯誤，而不冒著重複產生副作用的風險。

## 目前的限制

- 公開匯入路徑採用通用名稱，但部分嘗試／結果型別別名
  為了相容性仍沿用舊有名稱。
- 第三方執行框架的安裝仍處於實驗階段。在需要原生工作階段執行階段之前，
  請優先使用供應商外掛。
- 支援在不同回合之間切換執行框架。當原生工具、核准、助理文字或訊息
  傳送已開始後，請勿在回合進行途中切換執行框架。

## 相關內容

- [SDK 概觀](/zh-TW/plugins/sdk-overview)
- [執行階段輔助工具](/zh-TW/plugins/sdk-runtime)
- [供應商外掛](/zh-TW/plugins/sdk-provider-plugins)
- [Codex 執行框架](/zh-TW/plugins/codex-harness)
- [模型供應商](/zh-TW/concepts/model-providers)
