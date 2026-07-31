---
read_when:
    - 你希望 OpenClaw 代理程式能使用大型工具目錄，而不必將每個工具結構描述都加入提示詞中
    - 你希望透過單一精簡的執行階段介面公開 OpenClaw 工具、MCP 工具和用戶端工具
    - 你正在為 OpenClaw 執行實作或偵錯工具探索功能
summary: 工具搜尋：透過搜尋、描述與呼叫，精簡大型 OpenClaw 工具目錄
title: 工具搜尋
x-i18n:
    generated_at: "2026-07-26T08:10:30Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: d31322d5ef108c52fd14d48771cc3c6c43fcfbc4bfb95652bc29a55fd706c903
    source_path: tools/tool-search.md
    workflow: 16
---

Tool Search 是一項實驗性的 OpenClaw 代理程式執行階段功能。它為代理程式提供一種
精簡的方式，用來探索及呼叫大型工具目錄。當執行作業有許多可用工具，
但模型可能只需要其中少數工具時，此功能特別實用。

本頁說明 OpenClaw Tool Search。這不是 Codex 原生的工具
搜尋或動態工具介面。Codex 原生程式碼模式、工具搜尋、延遲載入的
動態工具及巢狀工具呼叫，都是穩定的 Codex 控制框架介面，
不依賴 `tools.toolSearch`。

若要瞭解公開 QuickJS-WASI `exec`/`wait`
介面而非 Tool Search 控制項的一般 OpenClaw 執行階段，請參閱[程式碼模式](/zh-TW/tools/code-mode)。

為 OpenClaw 執行作業啟用此功能時，模型預設會收到一個 `tool_search_code` 工具，
以及結構化結果無法通過精簡橋接器的所有僅限直接使用工具。
程式碼工具會在隔離的 Node 子程序中，透過 `openclaw.tools` 橋接器
執行一小段 JavaScript 主體：

```js
const hits = await openclaw.tools.search("建立 GitHub 議題");
const tool = await openclaw.tools.describe(hits[0].id);
return await openclaw.tools.call(tool.id, {
  title: "啟動時當機",
  body: "重現步驟...",
});
```

目錄可包含符合目錄資格的 OpenClaw 工具、外掛工具、MCP
工具及用戶端提供的工具。模型不會預先看到每個已編入目錄的結構描述。
它會改為搜尋精簡描述元，在需要確切結構描述時說明一項選定的
工具，然後透過 OpenClaw 呼叫該工具。僅限直接使用的工具仍會顯示給模型，
且不會加入目錄。

Codex 控制框架執行作業不會收到這些實驗性的 OpenClaw Tool Search
控制項。OpenClaw 會將產品能力以動態工具形式傳遞給 Codex，而
Codex 負責穩定的原生程式碼模式、原生工具搜尋、延遲載入的動態
工具及巢狀工具呼叫。

## 單一回合的執行方式

在規劃階段，OpenClaw 內嵌執行器會為執行作業建立有效目錄：

1. 解析代理程式、設定檔、沙箱及工作階段的有效工具政策。
2. 列出符合資格的 OpenClaw 與外掛工具。
3. 透過工作階段 MCP 執行階段列出符合資格的 MCP 工具。
4. 加入為目前執行作業提供的符合資格用戶端工具。
5. 讓僅限直接使用的工具保持對模型可見，並為其餘符合目錄資格的工具
   建立精簡描述元索引。
6. 在這些僅限直接使用的工具旁，公開 OpenClaw 程式碼橋接器、結構化備援工具或
   精簡目錄介面。

在執行階段，每次實際工具呼叫都會返回 OpenClaw。隔離的 Node
執行階段不會持有外掛實作、MCP 用戶端物件或機密。
`openclaw.tools.call(...)` 會透過橋接器返回閘道，並繼續套用
一般的政策、核准、掛鉤、記錄及結果處理。

## 模式

`tools.toolSearch` 有三種面向模型的模式：

- `code`：公開 `tool_search_code`（預設的精簡 JavaScript 橋接器）
  及僅限直接使用的工具。
- `tools`：將 `tool_search`、`tool_describe` 及 `tool_call` 公開為一般
  結構化工具，供不應接收程式碼的供應商使用，並同時公開
  僅限直接使用的工具。
- `directory`：公開 `tool_search`、`tool_describe` 及 `tool_call`，並提供
  有界限的提示目錄，其中包含可用工具的名稱與說明，供應讓
  供應商可看到工具名稱，而不需看到所有完整結構描述。OpenClaw 也可以
  直接公開目前回合可能需要或必要之工具結構描述的小型有界限集合。
  在此模式下，僅限直接使用的工具也仍然可見。

所有模式都使用同一個經政策篩選的目錄及一般 OpenClaw 執行
路徑。標記為 `catalogMode: "direct-only"` 的工具會留在該目錄之外，
並保持對模型可見。如果目前執行階段無法啟動隔離的 Node 程式碼模式子
程序，預設 `code` 模式會在壓縮目錄前回復為 `tools`。
在 `directory` 模式下，用戶端提供的工具會在目前執行作業中保持直接可見，
而 OpenClaw 工具、外掛工具及 MCP 工具可以壓縮到目錄後方。
直接呼叫確切的隱藏目錄名稱時，會先從相同的已授權目錄載入該工具，
再執行呼叫。

所有模式均為實驗性質。對於小型 OpenClaw 工具目錄，建議直接公開工具；
對於 Codex 控制框架執行作業，則建議使用穩定的 Codex 原生介面。

沒有獨立的來源選取設定。啟用 Tool Search 時，目錄會在套用一般
政策篩選後，包含符合目錄資格的 OpenClaw、MCP 及用戶端工具；
僅限直接使用的工具則會另外保留。

## 此功能的用途

大型目錄很實用，但成本高昂。將每個工具結構描述傳送給模型，
會使要求變大、規劃變慢，並增加意外選取工具的機率。

Tool Search 會改變其形式：

- 直接工具：模型會在產生第一個權杖前看到每個選定的結構描述
- Tool Search 程式碼模式：模型會看到一個精簡的程式碼工具、一份簡短的 API
  合約，以及所有僅限直接使用的工具
- Tool Search 工具模式：模型會看到三個精簡的結構化備援
  工具，以及所有僅限直接使用的工具
- Tool Search 目錄模式：模型會看到有界限的目錄、
  搜尋／說明／呼叫控制項，以及可能需要或必要之結構描述的小型有界限集合，
  另外還有所有僅限直接使用的工具
- 回合進行期間：模型可視需要載入其餘結構描述

對小型目錄而言，直接公開工具仍是正確的預設選擇。當單一執行作業
可看到許多工具時，Tool Search 最為適合，尤其是來自 MCP 伺服器或
用戶端提供的應用程式工具。

## API

`openclaw.tools.search(query, options?)`

搜尋目前執行作業的有效目錄。結果精簡，且可安全地放回
提示內容中。每個命中項目都包含有界限的 TypeScript 風格
`input` 簽章，例如 `{ id: string; mode?: "drip" | "flood" }`，因此當該簽章
已足夠時，模型可略過 `describe`。受信任的 OpenClaw
核心或外掛工具也可包含精簡的 `output` 提示，例如
`Array<{ id: string; paid: boolean }>`。MCP 與用戶端的輸出結構描述宣告
不會提升為此受信任提示。其不受信任的輸入結構描述也會
延遲為 `input: "unknown"`；呼叫前請使用 `describe`。開放式、
過大或其他不完整的輸出結構描述會省略此提示，但仍可改由
`describe` 取得。

```js
const hits = await openclaw.tools.search("行事曆事件", { limit: 5 });
```

`openclaw.tools.describe(id)`

載入一個搜尋結果的完整中繼資料，包括確切的輸入結構描述，以及
工具有宣告時受信任的完整 `outputSchema`。

```js
const calendarCreate = await openclaw.tools.describe("mcp:calendar:create_event");
```

`openclaw.tools.call(id, args)`

透過 OpenClaw 呼叫選定的工具，並傳回原始 `{ tool, result }`
封套。傳回 JSON 的工具通常會將其值放在
`result.details` 中。如果受信任的工具宣告 `outputSchema`，OpenClaw 會在
執行前編譯該結構描述，並在一般工具掛鉤完成後驗證最終
`details`，再傳回目錄呼叫。

```js
await openclaw.tools.call(calendarCreate.id, {
  summary: "規劃",
  start: "2026-05-09T14:00:00Z",
});
```

工具作者會在工具的 `outputSchema` 屬性上宣告輸出合約。
它描述的是 `AgentToolResult.details`，而非轉譯後的內容區塊。請納入
所有不會擲回例外的變體；若結果不穩定，則省略此屬性。請參閱
[程式碼模式輸出合約](/zh-TW/tools/code-mode#declared-output-contracts)及
[工具外掛](/zh-TW/plugins/tool-plugins#output-contracts)。

結構化備援模式會將相同的操作公開為工具：

- `tool_search`
- `tool_describe`
- `tool_call`

目錄模式會公開：

- `tool_search`
- `tool_describe`
- `tool_call`

它也會讓用戶端提供的工具及所有僅限直接使用的工具保持直接可見，
並可能直接公開目前回合可能需要或必要之目錄工具結構描述的小型
有界限集合。如果有界限的目錄省略了項目，請使用
`tool_search` 尋找它們。如果模型直接要求確切的隱藏目錄
工具名稱，OpenClaw 會先從已授權目錄載入該工具，再進行
一般執行。
目錄模式的用戶端工具名稱不得與 OpenClaw、外掛或 MCP
工具名稱衝突，因為確切的延遲分派會使用這些名稱。

## 執行階段邊界

程式碼橋接器在短期存在的 Node 子程序中執行。子程序啟動時
會啟用 Node 權限模式、使用空白環境，不授予檔案系統或
網路權限，也不授予子程序或工作執行緒權限。OpenClaw 會強制執行
父程序的實際經過時間逾時，並在逾時時終止子程序，包括
非同步延續執行之後。

執行階段只公開：

- `console.log`、`console.warn` 及 `console.error`
- `openclaw.tools.search`
- `openclaw.tools.describe`
- `openclaw.tools.call`

最終呼叫仍會套用一般 OpenClaw 行為：

- 工具允許與拒絕政策
- 每個代理程式及每個沙箱的工具限制
- 頻道／執行階段工具政策
- 核准掛鉤
- 外掛 `before_tool_call` 掛鉤
- 工作階段身分、記錄及遙測

## 設定

使用預設程式碼橋接器，為 OpenClaw 執行作業啟用 Tool Search：

```bash
openclaw config set tools.toolSearch true
```

等效的 JSON：

```json5
{
  tools: {
    toolSearch: true,
  },
}
```

若是 OpenClaw 執行作業，也可以改用結構化備援工具：

```json5
{
  tools: {
    toolSearch: {
      mode: "tools",
    },
  },
}
```

若是 OpenClaw 執行作業，也可以改用精簡目錄介面：

```json5
{
  tools: {
    toolSearch: {
      mode: "directory",
    },
  },
}
```

調整程式碼模式逾時及搜尋結果限制（顯示的值為預設值）：

```json5
{
  tools: {
    toolSearch: {
      mode: "code",
      codeTimeoutMs: 10000,
      searchDefaultLimit: 8,
      maxSearchLimit: 20,
    },
  },
}
```

執行階段會將 `codeTimeoutMs` 限制在 1000-60000、將 `maxSearchLimit` 限制在 1-50，並將
`searchDefaultLimit` 限制在 1..`maxSearchLimit`。

停用此功能：

```json5
{
  tools: {
    toolSearch: false,
  },
}
```

## 提示與遙測

Tool Search 會記錄足夠的遙測資料，以便與直接公開工具進行比較：

- 傳送至控制框架的序列化工具與提示位元組總數
- 目錄大小及來源分布
- 搜尋、說明及呼叫次數
- 透過 OpenClaw 執行的最終工具呼叫
- 選定的工具 ID 及來源

工作階段記錄應可用來回答：

- 模型預先看到了多少個工具結構描述
- 模型執行了多少次搜尋及說明操作
- 呼叫了哪個最終工具
- 結果來自 OpenClaw、MCP 還是用戶端工具

## 端對端驗證

QA Lab 閘道情境會使用 OpenClaw 執行階段驗證兩條路徑：

```bash
pnpm openclaw qa suite --provider-mode mock-openai --scenario tool-search-gateway-e2e
```

它會建立具有大型工具目錄的暫時假外掛、啟動模擬
OpenAI 供應商、分別以直接模式及啟用 Tool Search 的模式啟動一次閘道，
然後比較供應商要求承載資料及工作階段記錄。

此迴歸測試會驗證：

1. 直接模式可以呼叫假的外掛工具。
2. 工具搜尋可以呼叫相同的假外掛工具。
3. 直接模式會將假外掛工具的結構描述直接提供給供應商。
4. 工具搜尋僅提供精簡橋接器，以及任何僅限直接模式使用的工具。
5. 對於大型假目錄，工具搜尋的請求承載資料較小。
6. 工作階段記錄會顯示預期的工具呼叫次數與橋接呼叫遙測資料。

## 失敗行為

工具搜尋應採取封閉式失敗：

- 如果工具不在有效政策中，搜尋不應傳回該工具
- 如果選取的工具變得無法使用，`tool_call` 應失敗
- 如果政策或核准阻止執行，呼叫結果應回報該
  阻擋，而不是繞過它
- 如果程式碼橋接器無法建立隔離的執行環境，請使用 `mode: "tools"`，或
  為該部署停用工具搜尋

## 相關內容

- [工具與外掛](/zh-TW/tools)
- [多代理程式沙箱與工具](/zh-TW/tools/multi-agent-sandbox-tools)
- [執行工具](/zh-TW/tools/exec)
- [ACP 代理程式設定](/zh-TW/tools/acp-agents-setup)
- [建置外掛](/zh-TW/plugins/building-plugins)
