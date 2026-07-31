---
read_when:
    - 你正在 OpenClaw、Codex、ACP 或其他原生代理程式執行環境之間做選擇
    - 你對狀態或設定中的提供者／模型／執行階段標籤感到困惑
    - 你正在記錄原生測試框架的支援一致性
summary: OpenClaw 如何區分模型供應商、模型、頻道與代理程式執行環境
title: 代理程式執行環境
x-i18n:
    generated_at: "2026-07-26T08:14:22Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 980d112946535df1566f2df4e3e71abacc2b073b51717c1e85fbb678691d39cb
    source_path: concepts/agent-runtimes.md
    workflow: 16
---

**代理執行階段**負責一個已準備好的模型迴圈：它接收提示詞、
驅動模型輸出、處理原生工具呼叫，並將完成的回合
傳回 OpenClaw。

執行階段很容易與供應商混淆，因為兩者都會出現在模型
設定附近。它們屬於不同的層級：

| 層級          | 範例                                         | 意義                                                                  |
| ------------- | -------------------------------------------- | --------------------------------------------------------------------- |
| 供應商        | `anthropic`, `github-copilot`, `openai`      | OpenClaw 如何進行驗證、探索模型，以及命名模型參照。                  |
| 模型          | `claude-opus-4-6`, `gpt-5.6-sol`             | 為代理回合選取的模型。                                                |
| 代理執行階段  | `claude-cli`, `codex`, `copilot`, `openclaw` | 執行已準備回合的低階迴圈或後端。                                      |
| 頻道          | Discord, Slack, Telegram, WhatsApp           | 訊息進出 OpenClaw 的位置。                                            |

**代理框架**是提供代理執行階段的實作（程式碼
術語）。例如，隨附的 Codex 代理框架實作了 `codex` 執行階段。
公開設定會在供應商或模型項目上使用 `agentRuntime.id`；整個代理的
執行階段鍵屬於舊版設定，會被忽略。`openclaw doctor --fix` 會移除舊有的
整個代理執行階段固定設定，並視需要將舊版執行階段模型參照改寫為標準的
供應商／模型參照及模型範圍的執行階段原則。

兩類執行階段：

- **嵌入式代理框架**會在 OpenClaw 已準備好的代理迴圈內執行：包括
  內建的 `openclaw` 執行階段，以及已註冊的外掛代理框架，例如
  `codex` 和 `copilot`。
- **命令列介面後端**會執行本機命令列介面程序，同時維持標準模型參照。
  例如，`anthropic/claude-opus-5` 搭配模型範圍的
  `agentRuntime.id: "claude-cli"`，表示「選取 Anthropic 模型，透過
  Claude 命令列介面執行。」`claude-cli` 不是嵌入式代理框架 ID，不得
  傳入 AgentHarness 選取程序。

`copilot` 代理框架是供 GitHub Copilot 命令列介面使用的獨立選用外部外掛代理框架；
關於 PI、Codex 與 GitHub Copilot 代理執行階段之間面向使用者的選擇，
請參閱 [GitHub Copilot 代理執行階段](/zh-TW/plugins/copilot)。

## Codex 介面

數個介面共用 Codex 名稱：

| 介面                                             | OpenClaw 名稱／設定                    | 功能                                                                                                            |
| ------------------------------------------------ | -------------------------------------- | --------------------------------------------------------------------------------------------------------------- |
| 原生 Codex app-server 執行階段                   | `openai/*` 模型參照                   | 透過 Codex app-server 執行 OpenAI 嵌入式代理回合。這是一般的 ChatGPT/Codex 訂閱設定。                           |
| Codex OAuth 驗證設定檔                           | `openai` OAuth 設定檔                 | 儲存供 Codex app-server 代理框架使用的 ChatGPT/Codex 訂閱驗證資訊。                                             |
| Codex ACP 轉接器                                 | `runtime: "acp"`, `agentId: "codex"` | 透過外部 ACP/acpx 控制平面執行 Codex。僅在明確要求 ACP/acpx 時使用。                                           |
| 原生 Codex 聊天控制命令集                        | `/codex ...`                            | 從聊天繫結、繼續、引導、停止及檢查 Codex app-server 執行緒。                                                   |
| 非代理介面的 OpenAI Platform API 路由            | `openai/*` 加上 API 金鑰驗證           | 直接使用 OpenAI API，例如影像、嵌入、語音及即時功能。                                                          |

這些介面刻意彼此獨立。啟用 `codex` 外掛
會提供原生 app-server 功能；`openclaw doctor --fix` 負責
修復舊版 Codex 路由及清理過時的工作階段固定設定。現在為代理模型選取 `openai/*`
表示「透過 Codex 執行此模型」，除非使用的是非代理
OpenAI API 介面。

常見的 ChatGPT/Codex 訂閱設定使用 Codex OAuth 進行驗證，但
會將模型參照維持為 `openai/*`，並選取 `codex` 執行階段：

```json5
{
  agents: {
    defaults: {
      model: "openai/gpt-5.6-sol",
    },
  },
}
```

這表示 OpenClaw 會選取 OpenAI 模型參照，接著要求 Codex
app-server 執行階段執行嵌入式代理回合。這並不表示「使用 API
計費」，也不表示頻道、模型供應商目錄或
OpenClaw 工作階段儲存區會變成 Codex。

啟用隨附的 `codex` 外掛時，請使用原生 `/codex` 命令
介面（`/codex bind`、`/codex threads`、`/codex resume`、`/codex steer`、
`/codex stop`）以自然語言控制 Codex，而不要使用 ACP。只有在
使用者明確要求 ACP/acpx，或正在測試 ACP 轉接器路徑時，才對
Codex 使用 ACP。Claude Code、Gemini 命令列介面、OpenCode、Cursor 及類似的外部
代理框架仍使用 ACP。

決策樹：

1. **Codex 繫結／控制／執行緒／繼續／引導／停止** -> 啟用隨附的 `codex` 外掛時，使用原生 `/codex` 命令介面。
2. **將 Codex 作為嵌入式執行階段**，或一般由訂閱支援的 Codex 代理體驗 -> `openai/<model>`。
3. **明確選擇 OpenClaw 執行 OpenAI 模型** -> 將模型參照維持為 `openai/<model>`，並將供應商／模型執行階段原則設為 `agentRuntime.id: "openclaw"`。選取的 `openai` OAuth 設定檔會在內部透過 OpenClaw 的 Codex 驗證傳輸進行路由。
4. **設定中的舊版 Codex 模型參照** -> 使用 `openclaw doctor --fix` 修復為 `openai/<model>`；若舊模型參照隱含該意圖，doctor 會加入供應商／模型範圍的 `agentRuntime.id: "codex"`，以保留 Codex 驗證路由。舊版 **`codex-cli/*`** 模型參照會修復為相同的 `openai/<model>` Codex app-server 路由；OpenClaw 不再保留隨附的 Codex 命令列介面後端。
5. **明確要求 ACP、acpx 或 Codex ACP 轉接器** -> `runtime: "acp"` 和 `agentId: "codex"`。
6. **Claude Code、Gemini 命令列介面、OpenCode、Cursor、Droid 或其他外部代理框架** -> ACP/acpx，而非原生子代理執行階段。

| 你的意思是……                           | 使用……                                               |
| --------------------------------------- | ---------------------------------------------------- |
| Codex app-server 聊天／執行緒控制       | 隨附的 `codex` 外掛所提供的 `/codex ...` |
| Codex app-server 嵌入式代理執行階段     | `openai/*` 代理模型參照                              |
| OpenAI Codex OAuth                      | `openai` OAuth 設定檔                             |
| Claude Code 或其他外部代理框架          | ACP/acpx                                             |

關於 OpenAI 系列前綴的區分，請參閱 [OpenAI](/zh-TW/providers/openai) 和
[模型供應商](/zh-TW/concepts/model-providers)。關於 Codex 執行階段支援
合約，請參閱 [Codex 代理框架執行階段](/zh-TW/plugins/codex-harness-runtime#v1-support-contract)。

## 執行階段所有權

不同的執行階段負責迴圈中不同範圍的工作：

| 介面                        | OpenClaw 嵌入式                                  | Codex app-server                                                               |
| --------------------------- | ------------------------------------------------ | ------------------------------------------------------------------------------ |
| 模型迴圈擁有者              | OpenClaw，透過 OpenClaw 嵌入式執行器             | Codex app-server                                                               |
| 標準執行緒狀態              | OpenClaw 記錄                                    | Codex 執行緒，加上 OpenClaw 記錄鏡像                                          |
| OpenClaw 動態工具           | 原生 OpenClaw 工具迴圈                           | 透過 Codex 轉接器橋接                                                          |
| 原生 shell 與檔案工具       | OpenClaw 路徑                                    | Codex 原生工具，在支援時透過原生掛鉤橋接                                      |
| 上下文引擎                  | 原生 OpenClaw 上下文組合                         | OpenClaw 將組合完成的上下文投射至 Codex 回合                                   |
| 壓縮                        | OpenClaw 或選取的上下文引擎                      | Codex 原生壓縮，搭配 OpenClaw 通知及鏡像維護                                  |
| 頻道傳遞                    | OpenClaw                                         | OpenClaw                                                                       |

設計規則：若介面由 OpenClaw 負責，即可提供一般的外掛掛鉤
行為。若介面由原生執行階段負責，OpenClaw 就需要執行階段
事件或原生掛鉤。若標準執行緒狀態由原生執行階段負責，
OpenClaw 會鏡像及投射上下文，而不是改寫不支援的
內部機制。

## 執行階段選取

OpenClaw 會在解析供應商和模型後，依照
下列順序解析嵌入式執行階段：

1. **模型範圍的執行階段原則**優先。它位於已設定的供應商
   模型項目中，或 `agents.defaults.models["provider/model"].agentRuntime`
   ／`agents.entries.*.models["provider/model"].agentRuntime` 中。像
   `agents.defaults.models["vllm/*"].agentRuntime` 這樣的供應商萬用字元會在精確模型原則之後套用，
   因此動態探索到的供應商模型可共用一個執行階段，而不會
   覆寫精確的個別模型例外。
2. **供應商範圍的執行階段原則**：`models.providers.<provider>.agentRuntime`。
3. **`auto` 模式**：已註冊的外掛執行階段可宣告支援的供應商／模型組合。
4. 若在 `auto` 模式中沒有任何項目接管該回合，OpenClaw 會回退至
   `openclaw`，作為相容性執行階段。執行作業必須嚴格符合要求時，
   請使用明確的執行階段 ID。

整個工作階段和整個代理的執行階段固定設定會被忽略：`OPENCLAW_AGENT_RUNTIME`、
工作階段 `agentHarnessId`/`agentRuntimeOverride` 狀態、`agents.defaults.agentRuntime`
和 `agents.entries.*.agentRuntime`。執行 `openclaw doctor --fix` 可移除過時的
整個代理執行階段設定，並在能保留原意時轉換舊版執行階段模型參照。

明確設定的供應商／模型外掛執行階段會採取封閉式失敗：供應商或模型上的 `agentRuntime.id: "codex"`
表示 Codex，否則會產生明確的選取／執行階段錯誤——絕不會
無提示地路由回 OpenClaw。只有 `auto` 可將不相符的
回合路由至 OpenClaw。

命令列介面後端別名與嵌入式代理框架 ID 不同。建議的 Claude 命令列介面形式：

```json5
{
  agents: {
    defaults: {
      model: "anthropic/claude-opus-5",
      models: {
        "anthropic/claude-opus-5": {
          agentRuntime: { id: "claude-cli" },
        },
      },
    },
  },
}
```

為了相容性，仍支援 `claude-cli/claude-opus-4-7` 等舊版參照，
但新設定應維持供應商／模型的標準形式，並將執行後端
放入供應商／模型執行階段原則中。

舊版 `codex-cli/*` 參照則不同：doctor 會將其遷移至 `openai/*`，
使其透過 Codex app-server 代理框架執行，而不是保留 Codex
命令列介面後端。

對大多數供應商而言，`auto` 模式刻意採取保守策略。OpenAI 代理
模型是例外：未設定執行階段及 `auto` 都會解析至 Codex
代理框架。明確的 OpenClaw 執行階段設定仍是 `openai/*` 代理回合的選用相容
路由；與選取的 `openai` OAuth 設定檔搭配時，OpenClaw 會在內部透過 Codex 驗證
傳輸來路由該路徑，同時將公開模型參照維持為 `openai/*`。過時的 OpenAI
執行階段工作階段固定設定會被執行階段選取程序忽略，並可使用
`openclaw doctor --fix` 清理。

如果 `openclaw doctor` 警告已啟用 `codex` 外掛，但設定中仍有舊版
Codex 模型參照，請將其視為舊版路由狀態，並執行
`openclaw doctor --fix`，使用 Codex 執行階段將其改寫為 `openai/*`。

## GitHub Copilot 代理程式執行階段

外部 `@openclaw/copilot` 外掛會註冊由 GitHub Copilot 命令列介面
（`@github/copilot-sdk`）支援、需選擇啟用的 `copilot` 執行階段。它會宣告
標準訂閱 `github-copilot` 提供者，且**絕不會**由
`auto` 選取。透過 `agentRuntime.id` 依模型或提供者選擇啟用：

```json5
{
  agents: {
    defaults: {
      model: "github-copilot/gpt-5.5",
      models: {
        "github-copilot/gpt-5.5": {
          agentRuntime: { id: "copilot" },
        },
      },
    },
  },
}
```

此框架會在 `extensions/copilot/doctor-contract-api.ts` 中宣告其提供者、執行階段、命令列介面工作階段金鑰和驗證設定檔
前綴，而 `openclaw doctor` 會自動載入該項目。關於設定、驗證、逐字稿鏡像、壓縮、
宣告式 doctor 合約，以及更廣泛的 PI、Codex 與 Copilot SDK
選擇，請參閱 [GitHub Copilot 代理程式執行階段](/zh-TW/plugins/copilot)。

## 相容性合約

當執行階段不是 OpenClaw 時，其文件應說明支援哪些 OpenClaw 介面：

| 問題                                   | 重要性                                                                                              |
| -------------------------------------- | --------------------------------------------------------------------------------------------------- |
| 誰負責模型迴圈？                       | 決定重試、工具延續和最終答案決策在何處進行。                                                        |
| 誰負責標準執行緒歷程？                 | 決定 OpenClaw 能否編輯歷程，或只能建立鏡像。                                                        |
| OpenClaw 動態工具是否可用？            | 訊息、工作階段、排程和 OpenClaw 所屬工具依賴此功能。                                                |
| 動態工具鉤子是否可用？                 | 外掛需要 `before_tool_call`、`after_tool_call`，以及圍繞 OpenClaw 所屬工具的中介軟體。              |
| 原生工具鉤子是否可用？                 | Shell、修補程式和執行階段所屬工具需要原生鉤子支援，才能套用政策並進行觀察。                          |
| 內容引擎生命週期是否會執行？           | 記憶與內容外掛依賴組裝、擷取、回合後處理和壓縮生命週期。                                            |
| 會公開哪些壓縮資料？                   | 某些外掛只需要通知；其他外掛則需要保留／捨棄項目的中繼資料。                                        |
| 哪些功能刻意不受支援？                 | 當原生執行階段擁有更多狀態時，使用者不應假定其等同於 OpenClaw。                                     |

Codex 執行階段支援合約記載於
[Codex 框架執行階段](/zh-TW/plugins/codex-harness-runtime#v1-support-contract)。

## 狀態標籤

狀態輸出可以同時顯示 `Execution` 和 `Runtime` 標籤。請將其視為
診斷資訊，而非提供者名稱：

- 模型參照（例如 `openai/gpt-5.6-sol`）是所選的提供者／模型。
- 執行階段 ID（例如 `codex`）是執行該回合的迴圈。
- 頻道標籤（例如 Telegram 或 Discord）代表對話發生的位置。

如果某次執行顯示非預期的執行階段，請先檢查所選提供者／模型的
執行階段政策。舊版工作階段執行階段固定設定已不再決定路由。

## 相關內容

- [Codex 框架](/zh-TW/plugins/codex-harness)
- [Codex 框架執行階段](/zh-TW/plugins/codex-harness-runtime)
- [GitHub Copilot 代理程式執行階段](/zh-TW/plugins/copilot)
- [OpenAI](/zh-TW/providers/openai)
- [代理程式框架外掛](/zh-TW/plugins/sdk-agent-harness)
- [代理程式迴圈](/zh-TW/concepts/agent-loop)
- [模型](/zh-TW/concepts/models)
- [狀態](/zh-TW/cli/status)
