---
read_when:
    - 你正在建置本機 AI 命令列介面後端外掛
    - 你想要為 `acme-cli/model` 之類的模型參照註冊後端
    - 你需要將第三方命令列介面對應至 OpenClaw 的文字備援執行器
sidebarTitle: CLI backend plugins
summary: 建置一個註冊本機 AI 命令列介面後端的外掛
title: 建置命令列介面後端外掛
x-i18n:
    generated_at: "2026-07-26T08:26:18Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 1923b0829b46a309e4b5a6cbbbfd3dcb76a1e14fe4106310d7a9fb37bca41d70
    source_path: plugins/cli-backend-plugins.md
    workflow: 16
---

CLI 後端外掛可讓 OpenClaw 呼叫本機 AI 命令列介面，作為文字推論
後端。此後端會在模型參照中顯示為提供者前綴：

```text
acme-cli/acme-large
```

當上游整合已透過本機命令提供、命令列介面管理本機登入狀態，或在 API
提供者無法使用時作為備援，可使用 CLI 後端。

<Info>
  如果上游服務提供一般的 HTTP 模型 API，請改為編寫
  [提供者外掛](/zh-TW/plugins/sdk-provider-plugins)。如果上游執行階段管理完整的代理程式工作階段、工具事件、壓縮或背景
  工作狀態，請使用[代理程式控管介面](/zh-TW/plugins/sdk-agent-harness)。
</Info>

## 外掛負責的項目

CLI 後端外掛包含三項合約：

| 合約                 | 檔案                   | 用途                                                      |
| -------------------- | ---------------------- | --------------------------------------------------------- |
| 套件進入點           | `package.json`         | 將 OpenClaw 指向外掛執行階段模組                          |
| 資訊清單所有權       | `openclaw.plugin.json` | 在載入執行階段前宣告後端 ID                               |
| 執行階段註冊         | `index.ts`             | 使用命令預設值呼叫 `api.registerCliBackend(...)`                     |

資訊清單是探索中繼資料：它不會執行命令列介面或註冊
執行階段行為。外掛進入點呼叫
`api.registerCliBackend(...)` 時，執行階段行為才會開始。

## 最小後端外掛

<Steps>
  <Step title="建立套件中繼資料">
    ```json package.json
    {
      "name": "@acme/openclaw-acme-cli",
      "version": "1.0.0",
      "type": "module",
      "openclaw": {
        "extensions": ["./index.ts"],
        "compat": {
          "pluginApi": ">=2026.3.24-beta.2",
          "minGatewayVersion": "2026.3.24-beta.2"
        },
        "build": {
          "openclawVersion": "2026.3.24-beta.2",
          "pluginSdkVersion": "2026.3.24-beta.2"
        }
      },
      "dependencies": {
        "openclaw": "^2026.3.24"
      },
      "devDependencies": {
        "typescript": "^5.9.0"
      }
    }
    ```

    已發布的套件必須包含建置完成的 JavaScript 執行階段檔案。如果你的原始碼
    進入點是 `./src/index.ts`，請新增指向
    建置完成之對應 JavaScript 檔案的 `openclaw.runtimeExtensions`。請參閱[進入點](/zh-TW/plugins/sdk-entrypoints)。

  </Step>

  <Step title="宣告後端所有權">
    ```json openclaw.plugin.json
    {
      "id": "acme-cli",
      "name": "Acme CLI",
      "description": "Run Acme's local AI CLI through OpenClaw",
      "cliBackends": ["acme-cli"],
      "setup": {
        "cliBackends": ["acme-cli"],
        "requiresRuntime": false
      },
      "activation": {
        "onStartup": false
      },
      "configSchema": {
        "type": "object",
        "additionalProperties": false
      }
    }
    ```

    `cliBackends` 是執行階段所有權清單；當模型選擇或 `agentRuntime.id` 提及 `acme-cli` 時，
    它可讓 OpenClaw 自動載入
    外掛。

    `setup.cliBackends` 是描述元優先的設定介面。當
    模型探索、初始設定或狀態應在不載入外掛執行階段的情況下辨識後端時，請新增此介面。
    只有當這些靜態描述元足以完成設定時，才使用 `requiresRuntime: false`。

  </Step>

  <Step title="註冊後端">
    ```typescript index.ts
    import { definePluginEntry } from "openclaw/plugin-sdk/plugin-entry";
    import {
      CLI_FRESH_WATCHDOG_DEFAULTS,
      CLI_RESUME_WATCHDOG_DEFAULTS,
      type CliBackendPlugin,
    } from "openclaw/plugin-sdk/cli-backend";

    function buildAcmeCliBackend(): CliBackendPlugin {
      return {
        id: "acme-cli",
        liveTest: {
          defaultModelRef: "acme-cli/acme-large",
          defaultImageProbe: false,
          defaultMcpProbe: false,
          docker: {
            npmPackage: "@acme/acme-cli",
            binaryName: "acme",
          },
        },
        config: {
          command: "acme",
          args: ["chat", "--output-format", "stream-json", "--prompt", "{prompt}"],
          resumeArgs: [
            "chat",
            "--resume",
            "{sessionId}",
            "--output-format",
            "stream-json",
            "--prompt",
            "{prompt}",
          ],
          output: "jsonl",
          resumeOutput: "jsonl",
          jsonlDialect: "gemini-stream-json",
          input: "arg",
          modelArg: "--model",
          modelAliases: {
            large: "acme-large-2026",
            fast: "acme-fast-2026",
          },
          sessionArgs: ["--session", "{sessionId}"],
          sessionMode: "existing",
          sessionIdFields: ["session_id", "conversation_id"],
          systemPromptFileArg: "--system-file",
          systemPromptWhen: "first",
          imageArg: "--image",
          imageMode: "repeat",
          imagePathScope: "workspace",
          reliability: {
            watchdog: {
              fresh: { ...CLI_FRESH_WATCHDOG_DEFAULTS },
              resume: { ...CLI_RESUME_WATCHDOG_DEFAULTS },
            },
          },
          serialize: true,
        },
      };
    }

    export default definePluginEntry({
      id: "acme-cli",
      name: "Acme CLI",
      description: "Run Acme's local AI CLI through OpenClaw",
      register(api) {
        api.registerCliBackend(buildAcmeCliBackend());
      },
    });
    ```

    後端 ID 必須符合資訊清單的 `cliBackends` 項目。已註冊的
    轉接器是具權威性的外掛程式碼；OpenClaw 設定會選擇後端，
    但不會改寫其命令合約。

  </Step>
</Steps>

## 設定結構

`CliBackendConfig` 說明 OpenClaw 應如何啟動及剖析命令列介面。上述
完整範例刻意使用與內建
`google-gemini-cli` 轉接器相同的命令、恢復、JSONL、
模型別名、工作階段、影像及監控程序欄位：

| 欄位                                                      | 用途                                                                               |
| --------------------------------------------------------- | --------------------------------------------------------------------------------- |
| `command`                                                 | 二進位檔名稱或命令的絕對路徑                                                      |
| `args`                                                    | 全新執行的基礎 argv                                                               |
| `resumeArgs`                                              | 已恢復工作階段的替代 argv；支援 `{sessionId}`                                |
| `output` / `resumeOutput`                                 | 剖析器：`json`、`jsonl` 或 `text`              |
| `jsonlDialect`                                            | JSONL 事件方言：`claude-stream-json` 或 `gemini-stream-json`                          |
| `liveSession`                                             | 長時間執行的 CLI 處理程序模式（`claude-stdio`）                               |
| `input`                                                   | 提示詞傳輸方式：`arg` 或 `stdin`                          |
| `maxPromptArgChars`                                       | `arg` 模式在改用標準輸入前允許的提示詞長度上限                       |
| `env` / `clearEnv`                                        | 要注入的額外環境變數，或啟動前要移除的名稱                                        |
| `modelArg`                                                | 模型 ID 前使用的旗標                                                              |
| `modelAliases`                                            | 將 OpenClaw 模型 ID 對應至 CLI 原生 ID                                             |
| `sessionArgs`                                             | 如何使用 `{sessionId}` 傳遞工作階段 ID                                       |
| `sessionMode`                                             | `always`、`existing` 或 `none`                      |
| `sessionIdFields`                                         | OpenClaw 從 CLI 輸出讀取的 JSON 欄位                                               |
| `systemPromptArg` / `systemPromptFileArg`                 | 系統提示詞傳輸方式                                                                |
| `systemPromptFileConfigArg` / `systemPromptFileConfigKey` | 系統提示詞檔案的設定覆寫傳輸方式（例如 `-c`）                       |
| `systemPromptMode`                                        | `append` 或 `replace`                                          |
| `systemPromptWhen`                                        | `first`、`always` 或 `never`                      |
| `imageArg` / `imageMode`                                  | 影像路徑旗標，以及如何傳遞多張影像（`repeat` 或 `list`）     |
| `imagePathScope`                                          | 交接前暫存影像檔案的所在位置：`temp` 或 `workspace`             |
| `serialize`                                               | 讓相同後端的執行保持依序進行                                                      |
| `reseedFromRawTranscriptWhenUncompacted`                  | 選擇啟用在壓縮前對原始逐字稿進行有界限的重新植入，以安全重設工作階段              |
| `reliability.watchdog`                                    | 無輸出逾時調整，分別設定全新與已恢復的執行                                        |

應優先使用符合命令列介面的最小靜態設定。僅針對確實屬於後端的行為
新增外掛回呼。

## 進階後端掛鉤

`CliBackendPlugin` 也可定義：

| 掛鉤                               | 用途                                                                         |
| ---------------------------------- | --------------------------------------------------------------------------- |
| `normalizeConfig(config, context)` | 使用執行階段內容正規化已註冊的靜態轉接器                                    |
| `resolveExecutionArgs(ctx)`        | 新增要求範圍的旗標，例如思考強度或旁支問題隔離                              |
| `prepareExecution(ctx)`            | 在啟動前建立暫時的驗證、設定或環境橋接                                      |
| `transformSystemPrompt(ctx)`       | 套用最終的 CLI 特定系統提示詞轉換                                           |
| `textTransforms`                   | 雙向提示詞／輸出替換                                                        |
| `defaultAuthProfileId`             | 優先使用特定的 OpenClaw 驗證設定檔                                           |
| `authEpochMode`                    | 決定驗證變更使已儲存 CLI 工作階段失效的方式                                 |
| `nativeToolMode`                   | 宣告原生工具是不存在、永遠開啟，或可由主機選擇                              |
| `toolAvailabilityEnforcement`      | 宣告確切的工具上限是在 argv 或執行暫存階段中強制執行                         |
| `sideQuestionToolMode`             | 宣告 `/btw` 旁支問題所停用的原生工具                             |
| `bundleMcp` / `bundleMcpMode`      | 選擇啟用 OpenClaw 的迴送 MCP 工具橋接                                        |
| `ownsNativeCompaction`             | 後端自行管理壓縮 — OpenClaw 會延後處理                                       |
| `subscriptionAuthDispatch`         | 已選擇啟用、使用訂閱認證資訊的嵌入式執行會透過此後端執行                     |
| `runtimeArtifact`                  | 將指令碼啟動器限定於其完整的內建套件樹狀結構                                |

這些掛鉤應由提供者負責。如果後端掛鉤可表達該行為，
請勿在核心中新增 CLI 特定分支。

`prepareExecution(ctx)` 會接收 `ctx.contextTokenBudget`，也就是為該次執行選定的有效權杖
限制。自行處理原生壓縮的後端，可將此
預算對應至其命令列介面專用的啟動合約。

`runtimeArtifact` 由外掛擁有。僅在即時推論回合
建立或重新驗證已確認的設定授權時才會查詢；
一般命令列介面執行不需要它。沒有此宣告的後端無法
建立已驗證的命令列介面設定授權。`bundled-package-tree` 宣告會指定
確切的 `package.json` 擁有者，並要求套件進入點必須是該
命令。OpenClaw 會對有界且完整的已安裝套件樹狀結構進行雜湊，包括
巢狀相依套件；若遇到重新導向的符號連結、
宣告套件外的啟動器、必要的外部相依性
宣告、過大的樹狀結構或未知指令碼，則會採取失敗關閉。僅當該
樹狀結構包含完整的推論實作時才宣告此項；選用的工具整合
不會讓外部實作圖變得安全。

如果同一後端也隨附獨立完備的原生可執行檔，請在
`nativeExecutableNames` 中列出其標準基底名稱。其他原生命令仍為
未驗證狀態。

一般回合的 `ctx.executionMode` 為 `"agent"`，暫時性
`/btw` 呼叫則為 `"side-question"`。當命令列介面需要不同的一次性旗標時使用它，
例如針對 BTW 停用原生工具、工作階段持久化或恢復行為。
如果後端通常具有 `nativeToolMode: "always-on"`，但其
附帶問題的 argv 能可靠地停用這些工具，也請設定
`sideQuestionToolMode: "disabled"`；否則，當 BTW
要求執行無工具的命令列介面時，OpenClaw 會採取失敗關閉。

僅當後端能針對單次執行停用所有
後端原生工具時，才設定 `nativeToolMode: "selectable"`。受限制的執行會收到標準
合約：`ctx.toolAvailability.native` 是確切的後端原生清單，而
`ctx.toolAvailability.openClaw` 是確切的 OpenClaw 工具名稱清單。
主機會獨立將產生的 MCP 設定與授權限制於該
OpenClaw 清單；外掛不得在核心中轉換此清單，也不得加入傳輸前綴。

請宣告後端如何強制執行該合約：

- `toolAvailabilityEnforcement: "execution-args"` 要求
  `resolveExecutionArgs`。此掛鉤必須取代互相衝突的工具旗標、停用
  可能在所選工具範圍外執行的自訂介面，並
  為全新與恢復的執行傳回可強制執行的 argv。
- `toolAvailabilityEnforcement: "prepare-execution"` 要求
  `prepareExecution`。此掛鉤必須暫存精確的單次執行原則，並傳回
  `toolAvailabilityEnforced: true`；缺少確認時會採取失敗關閉，而且
  OpenClaw 會在啟動前清理暫存的資源。

在建立此合約前，OpenClaw 會將排程 `toolsAllow` 等執行階段上限
正規化並展開群組。原生工具會停用，而
缺少完整已宣告強制執行路徑的後端會在執行前失敗。

針對 `v2026.7.2-beta.1` 至 `v2026.7.2-beta.3` 建置的外掛，仍可
讀取已淘汰的 `ctx.toolAvailability.mcp` 傳輸名稱投影；當可選取的後端實作
`resolveExecutionArgs` 時，也可省略 `toolAvailabilityEnforcement`。
OpenClaw 會根據外掛套件必要的 `openclaw.build.openclawVersion` 中繼資料
辨識這條已發布的 beta 路徑，並將其保留至
`2026.8.x` 系列。新的與更新後的外掛應使用標準
`ctx.toolAvailability.openClaw` 名稱，並明確宣告
`toolAvailabilityEnforcement: "execution-args"`；beta
相容路徑預定在該期限結束後移除。

### `ownsNativeCompaction`：選擇不使用 OpenClaw 壓縮

如果你的後端執行的代理程式會壓縮其**自身**對話記錄，請設定
`ownsNativeCompaction: true`，使 OpenClaw 的防護摘要器永遠不會對
其工作階段執行；命令列介面的壓縮生命週期會直接略過，而
回合會繼續進行。`claude-cli` 會宣告此項，因為 Claude Code 會
在內部進行壓縮，且沒有控制框架端點。Codex 等原生控制框架工作階段
則會繼續路由至其控制框架壓縮端點。

**僅在以下條件全部成立時才宣告此項**，否則延後處理且
超出預算的工作階段可能持續超出預算或變成過期狀態（OpenClaw 將不再
挽救它）：

- 後端會在接近其
  視窗上限時，可靠地壓縮或限制自己的對話記錄；
- 後端會持久保存可恢復的工作階段，使壓縮後的狀態能跨回合保留
  （例如 `--resume` / `--session-id`）；
- 它不是原生控制框架壓縮工作階段；符合 `agentHarnessId` 的
  工作階段會改為路由至控制框架端點。

## MCP 工具橋接

命令列介面後端預設不會收到 OpenClaw 工具。如果命令列介面能使用
MCP 設定，請明確選擇啟用：

```typescript
return {
  id: "acme-cli",
  bundleMcp: true,
  bundleMcpMode: "codex-config-overrides",
  config: {
    command: "acme",
    args: ["chat", "--json"],
    output: "json",
  },
};
```

支援的橋接模式：

| 模式                     | 用途                                                              |
| ------------------------ | ---------------------------------------------------------------- |
| `claude-config-file`     | 接受 MCP 設定檔的命令列介面                              |
| `codex-config-overrides` | 接受 argv 設定覆寫的命令列介面                        |
| `gemini-system-settings` | 從其系統設定目錄讀取 MCP 設定的命令列介面 |

僅當命令列介面確實能使用橋接時才啟用。如果命令列介面具有
無法停用的內建工具層，請設定 `nativeToolMode:
"always-on"`，以便在呼叫端要求不得使用原生
工具時，OpenClaw 能採取失敗關閉。如果它能針對每次執行停用所有原生工具，請搭配上述
`resolveExecutionArgs` 合約使用 `"selectable"`。

## 選取後端

使用者透過模型參照前綴選取獨立後端。宣告了標準
`modelProvider` 的後端，則可改為透過該
供應商模型的 `agentRuntime.id` 選取。轉接器機制仍保留在外掛中：

```json5
{
  agents: {
    defaults: {
      model: {
        primary: "openai/gpt-5.6-sol",
        fallbacks: ["acme-cli/large"],
      },
    },
  },
}
```

請將認證資訊放在 OpenClaw 驗證設定檔或外掛擁有的設定中。請確保
已註冊的命令位於閘道服務的 `PATH` 中；需要不同
路徑或 argv 的部署應變更或包裝外掛註冊。

## 驗證

對於隨附的外掛，請為建構器與設定
註冊新增聚焦測試，然後執行該外掛的目標測試通道：

```bash
pnpm test extensions/acme-cli
```

對於本機或已安裝的外掛，請驗證探索功能並執行一次真實模型：

```bash
openclaw plugins inspect acme-cli --runtime --json
openclaw agent --message "完全照以下內容回覆：後端正常" --model acme-cli/acme-large
```

如果後端支援圖片或 MCP，請新增即時冒煙測試，以真實
命令列介面證明這些路徑可運作。請勿僅依賴靜態檢查來驗證提示詞、圖片、
MCP 或工作階段恢復行為。

## 檢查清單

<Check>`package.json` 具有 `openclaw.extensions`，且已發布套件具有建置完成的執行階段項目</Check>
<Check>`openclaw.plugin.json` 宣告 `cliBackends` 與刻意設定的 `activation.onStartup`</Check>
<Check>當設定／模型探索應在冷啟動時看到後端，必須提供 `setup.cliBackends`</Check>
<Check>`api.registerCliBackend(...)` 使用與資訊清單相同的後端 ID</Check>
<Check>後端模型前綴或模型範圍的 `agentRuntime.id` 會選取該註冊</Check>
<Check>工作階段、系統提示詞、圖片與輸出剖析器設定符合真實的命令列介面合約</Check>
<Check>目標測試與至少一次即時命令列介面冒煙測試證明後端路徑可運作</Check>

## 相關內容

- [命令列介面後端](/zh-TW/gateway/cli-backends)－執行階段選取與行為
- [建置外掛](/zh-TW/plugins/building-plugins)－套件與資訊清單基礎
- [外掛 SDK 概觀](/zh-TW/plugins/sdk-overview)－註冊 API 參考資料
- [外掛資訊清單](/zh-TW/plugins/manifest)－`cliBackends` 與設定描述元
- [代理程式控制框架](/zh-TW/plugins/sdk-agent-harness)－完整的外部代理程式執行階段
