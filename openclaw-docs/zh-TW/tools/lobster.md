---
read_when:
    - 你想要具備明確核准機制的確定性多步驟工作流程
    - 你需要在不重新執行先前步驟的情況下繼續工作流程
summary: 具備可續接核准閘門的 OpenClaw 型別化工作流程執行階段。
title: Lobster
x-i18n:
    generated_at: "2026-07-26T08:41:25Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 85b7900f86bfedc9d73fcc91c3d0dac37b81f7413b1e68c54dd8a797b70f79fc
    source_path: tools/lobster.md
    workflow: 16
---

Lobster 會將多步驟工具流水線作為一次確定性的工具呼叫執行，並提供
明確的核准檢查點與恢復權杖。它位於分離式背景工作的上一層：若要協調
跨越多個分離式任務的流程，請參閱 [Task Flow](/zh-TW/automation/taskflow) (`openclaw tasks flow`)；若要查看任務
活動帳本，請參閱[背景任務](/zh-TW/automation/tasks)。

## 原因

若沒有 Lobster，多步驟工作就意味著多次往返工具呼叫，且由
模型協調每個步驟。Lobster 將該協調工作移至具型別的
執行階段：

- **一次呼叫取代多次呼叫**：單次 Lobster 工具呼叫會傳回整個流水線的結構化
  結果。
- **內建核准機制**：副作用（傳送、發布、刪除）會暫停工作流程，
  直到取得明確核准。
- **可恢復**：暫停的工作流程會傳回權杖；核准後即可恢復，無須
  重新執行先前的步驟。

Lobster 是小型、受限的 DSL，而非通用指令碼語言：
核准／恢復是持久且內建的基本操作；流水線是資料（易於
記錄、比對差異、重播、審查）；精簡的語法限制了「創意」程式碼路徑，
使驗證保持務實；逾時、輸出上限、沙箱檢查和
允許清單由執行階段強制執行，而非由各個指令碼執行。每個步驟仍可
呼叫任何命令列介面或指令碼——如果你想使用功能更豐富的編寫語言，可透過其他工具產生 `.lobster` 檔案。

若沒有 Lobster，週期性的電子郵件分類會如下所示：

```text
使用者：「檢查我的電子郵件並擬定回覆」
→ openclaw 呼叫 gmail.list
→ LLM 摘要
→ 使用者：「為第 2 封和第 5 封擬定回覆」
→ LLM 擬稿
→ 使用者：「傳送第 2 封」
→ openclaw 呼叫 gmail.send
（每天重複，且不記得哪些郵件已分類）
```

使用 Lobster 後，同一項工作只需一次呼叫，並會暫停以等待核准，之後再恢復：

```json
{ "action": "run", "pipeline": "email.triage --limit 20", "timeoutMs": 30000 }
```

```json
{
  "ok": true,
  "status": "needs_approval",
  "output": [{ "summary": "5 封需要回覆，2 封需要處理" }],
  "requiresApproval": {
    "type": "approval_request",
    "prompt": "要傳送 2 封回覆草稿嗎？",
    "items": [],
    "resumeToken": "..."
  }
}
```

## 運作方式

OpenClaw 使用隨附的
`@clawdbot/lobster` 套件作為嵌入式執行器，**在處理程序內**執行 Lobster 工作流程。不會產生外部 `lobster`
子處理程序；工具呼叫會直接傳回 JSON 封套。如果
流水線暫停以等待核准，封套會攜帶恢復權杖（或簡短的
核准 ID），以便你稍後繼續。

## 啟用

Lobster 是**選用**的外掛工具，預設不會啟用。它已隨附
在內，因此不需要另外安裝——只要允許該工具即可：

```json
{
  "tools": {
    "alsoAllow": ["lobster"]
  }
}
```

或針對個別代理：

```json
{
  "agents": {
    "list": [
      {
        "id": "main",
        "tools": {
          "alsoAllow": ["lobster"]
        }
      }
    ]
  }
}
```

<Note>
`alsoAllow` 會在作用中的工具設定檔上新增 `lobster`，而不會
限制其他核心工具。只有在你想使用限制性的
允許清單模式時，才改用 `tools.allow`。
</Note>

在沙箱化工具內容中，此工具會完全停用。

如果你需要獨立版 Lobster 命令列介面進行開發或執行外部流水線
（位於嵌入式閘道執行器之外），請從
[Lobster 儲存庫](https://github.com/openclaw/lobster)安裝，並將 `lobster` 放入
`PATH`。

## 模式：小型命令列介面 + JSON 管道 + 核准

建置使用 JSON 通訊的小型命令，再將其串連成一次 Lobster 呼叫。
（以下為命令名稱範例——請替換成你自己的命令。）

```bash
inbox list --json
inbox categorize --json
inbox apply --json
```

```json
{
  "action": "run",
  "pipeline": "exec --json --shell 'inbox list --json' | exec --stdin json --shell 'inbox categorize --json' | exec --stdin json --shell 'inbox apply --json' | approve --preview-from-stdin --limit 5 --prompt '套用變更？'",
  "timeoutMs": 30000
}
```

如果流水線要求核准，請使用權杖恢復：

```json
{
  "action": "resume",
  "token": "<resumeToken>",
  "approve": true
}
```

範例：將輸入項目對應至工具呼叫：

```bash
gog.gmail.search --query 'newer_than:1d' \
  | openclaw.invoke --tool message --action send --each --item-key message --args-json '{"provider":"telegram","to":"..."}'
```

## 僅限 JSON 的 LLM 步驟（llm-task）

若要在工作流程中執行**結構化 LLM 步驟**，請啟用選用的
`llm-task` 外掛工具，並從 Lobster 呼叫它：

```json
{
  "plugins": {
    "entries": {
      "llm-task": { "enabled": true }
    }
  },
  "agents": {
    "list": [
      {
        "id": "main",
        "tools": { "alsoAllow": ["llm-task"] }
      }
    ]
  }
}
```

### 重要限制：嵌入式 Lobster 與 `openclaw.invoke`

隨附的 Lobster 外掛會在閘道內**以處理程序內方式**執行工作流程。
在該嵌入式模式下，`openclaw.invoke` **不會**自動繼承供巢狀 OpenClaw 命令列介面工具呼叫使用的
閘道 URL／驗證內容。

這表示此模式在**嵌入式執行器中目前並不可靠**：

```lobster
openclaw.invoke --tool llm-task --action json --args-json '{ ... }'
```

只有在已使用正確
閘道／驗證內容設定 `openclaw.invoke` 的環境中執行**獨立版 Lobster 命令列介面**時，才使用以下範例。

```lobster
openclaw.invoke --tool llm-task --action json --args-json '{
  "prompt": "根據輸入的電子郵件，傳回意圖與草稿。",
  "thinking": "low",
  "input": { "subject": "你好", "body": "可以幫忙嗎？" },
  "schema": {
    "type": "object",
    "properties": {
      "intent": { "type": "string" },
      "draft": { "type": "string" }
    },
    "required": ["intent", "draft"],
    "additionalProperties": false
  }
}'
```

如果你目前使用嵌入式 Lobster 外掛，請優先採用以下任一方式：

- 在 Lobster 外直接呼叫 `llm-task` 工具，或
- 在 Lobster 流水線中使用非 `openclaw.invoke` 步驟，直到新增受支援的
  嵌入式橋接為止。

如需詳細資訊與設定選項，請參閱 [LLM 任務](/zh-TW/tools/llm-task)。

## 工作流程檔案（.lobster）

Lobster 可執行具有 `name`、`args`、`steps`、`env`、
`condition` 和 `approval` 欄位的 YAML／JSON 工作流程檔案。請在工具
呼叫中將 `pipeline` 設為檔案路徑。

```yaml
name: inbox-triage
args:
  tag:
    default: "family"
steps:
  - id: collect
    command: inbox list --json
  - id: categorize
    command: inbox categorize --json
    stdin: $collect.stdout
  - id: approve
    command: inbox apply --approve
    stdin: $categorize.stdout
    approval: required
  - id: execute
    command: inbox apply --execute
    stdin: $categorize.stdout
    condition: $approve.approved
```

注意事項：

- `stdin: $step.stdout` 和 `stdin: $step.json` 會傳遞先前步驟的輸出。
- `condition`（或 `when`）可根據 `$step.approved` 控制步驟是否執行。

### 注入的環境變數

每個步驟的殼層都會繼承父環境以及下列由 Lobster 注入的
變數，因此命令可參照已解析的工作流程引數，而不必將
原始值嵌入命令字串：

- `LOBSTER_ARG_<NAME>`——每個工作流程引數各有一個。名稱會轉成大寫，並將每段
  連續的非英數字元縮減為 `_`，因此引數 `user-id` 會變成
  `LOBSTER_ARG_USER_ID`。
- `LOBSTER_ARGS_JSON`——所有已解析的引數會合併為單一 JSON 字串。

以上就是注入的完整集合。**沒有**任何逐步輸出變數，
例如 `LOBSTER_STEP_<id>_STDOUT` 或 `LOBSTER_STEP_<id>_JSON_<field>`；殼層會將這些名稱視為未設定，因此參數展開的預設值可能會掩蓋錯誤。
請改為透過步驟參照讀取先前步驟的輸出——在 `stdin:`、`env:` 或 `condition:`
值中使用 `$step.stdout`、
`$step.json` 或 `$step.json.<field>`。（`LOBSTER_STATE_DIR` 是狀態
目錄的獨立執行階段設定，而非逐次執行引數。）

## 工具參數

### `run`

```json
{
  "action": "run",
  "pipeline": "gog.gmail.search --query 'newer_than:1d' | email.triage",
  "cwd": "workspace",
  "timeoutMs": 30000,
  "maxStdoutBytes": 512000
}
```

使用引數執行工作流程檔案：

```json
{
  "action": "run",
  "pipeline": "/path/to/inbox-triage.lobster",
  "argsJson": "{\"tag\":\"family\"}"
}
```

| 欄位             | 預設值      | 注意事項                                                                                                      |
| ---------------- | ----------- | ------------------------------------------------------------------------------------------------------------ |
| `pipeline`       | 必填        | 行內流水線字串，或以 `.lobster`/`.yaml`/`.yml`/`.json` 結尾的工作流程檔案路徑。           |
| `cwd`            | 閘道 cwd    | 相對工作目錄；必須解析至閘道工作目錄內（絕對路徑會遭拒絕）。 |
| `timeoutMs`      | `20000`     | 超過此值時中止執行。                                                                                  |
| `maxStdoutBytes` | `512000`    | 擷取的 stdout 或 stderr 超過此大小時中止執行。                                               |
| `argsJson`       | -           | 工作流程檔案引數的 JSON 字串（行內流水線會忽略此值）。                                      |

### `resume`

```json
{
  "action": "resume",
  "token": "<resumeToken>",
  "approve": true
}
```

`resume` 接受 `token`（來自 `requiresApproval` 的完整恢復權杖）
或 `approvalId`（來自同一物件的簡短 ID）——請使用暫停的
執行所傳回的值。`approve` 為必填。

### 受管理的 Task Flow 模式

在 `run` 上傳遞 `flowControllerId` 和 `flowGoal`（或在 `resume` 上傳遞 `flowId` 和
`flowExpectedRevision`），會讓呼叫透過外掛
執行階段受管理的 [Task Flow](/zh-TW/automation/taskflow) API 執行，而非傳回
單純的封套：OpenClaw 會建立或恢復持久流程記錄，並將
Lobster 封套套用至該記錄（核准時為 `waiting`，完成時為 `succeeded`/`failed`），
然後傳回 `{ ok, envelope, flow, mutation }`。此模式需要
已繫結的 Task Flow 執行階段，適用於需要在
閘道重新啟動後仍保有持久流程狀態的外掛／控制器程式碼，而非一般的臨時代理用途。

## 輸出封套

Lobster 會傳回具有下列三種狀態之一的 JSON 封套：

- `ok`——成功完成
- `needs_approval`——已暫停；`requiresApproval` 會攜帶 `resumeToken` 和簡短的
  `approvalId`，兩者任一皆可恢復執行
- `cancelled`——已明確拒絕或取消

此工具會同時在 `content`（格式化 JSON）和 `details`
（原始物件）中呈現封套。

## 核准

如果存在 `requiresApproval`，請檢查提示並決定：

- `approve: true`——恢復並繼續執行副作用
- `approve: false`——取消並結束工作流程

使用 `approve --preview-from-stdin --limit N` 可將 JSON 預覽附加至
核准要求，而不需要自訂 jq／heredoc 黏合程式碼。恢復狀態會以
小型 JSON 檔案儲存在 Lobster 狀態目錄下（預設為 `~/.lobster/state`，
可使用 `LOBSTER_STATE_DIR` 覆寫）；權杖本身僅編碼
指向該狀態的指標，而非完整的流水線狀態。

## OpenProse

OpenProse 與 Lobster 能良好搭配：使用 `/prose` 協調多代理
準備工作，再執行 Lobster 流水線以進行確定性核准。如果 Prose
程式需要 Lobster，請透過 `tools.subagents.tools` 為子代理允許
`lobster` 工具。請參閱 [OpenProse](/zh-TW/prose)。

## 安全性

- **僅限本機程序內** - 工作流程在閘道程序內執行；外掛本身不會
  發出網路呼叫。
- **不含機密資訊** - Lobster 不管理 OAuth；它會呼叫負責此功能的
  OpenClaw 工具。
- **感知沙箱環境** - 工具內容處於沙箱環境時停用。
- **強化防護** - 內嵌執行器會強制執行逾時與輸出上限。

## 疑難排解

| 錯誤                                                         | 原因／修正方式                                                                      |
| ------------------------------------------------------------- | -------------------------------------------------------------------------------- |
| `lobster runtime timed out`                                   | 流水線超過 `timeoutMs`。請提高此值或拆分流水線。                |
| `lobster stdout exceeded maxStdoutBytes`（或 `stderr`）        | 擷取的輸出超過上限。請提高 `maxStdoutBytes` 或減少輸出。       |
| `run --args-json must be valid JSON`                          | `argsJson`（工作流程檔案執行）剖析失敗。請修正 JSON 字串。            |
| `lobster runtime failed`（或其他 `runtime_error` 訊息） | 內嵌執行階段傳回錯誤封裝。請查看閘道記錄以取得詳細資訊。 |

## 深入瞭解

- [外掛](/zh-TW/tools/plugin)
- [外掛工具編寫](/zh-TW/plugins/building-plugins#registering-agent-tools)

## 案例研究：社群工作流程

一個公開範例：使用「第二大腦」命令列介面與 Lobster 流水線管理三個
Markdown 儲存庫（個人、伴侶、共用）。命令列介面會輸出統計資料、
收件匣清單與過期掃描的 JSON；Lobster 將這些命令串連成工作流程，
例如 `weekly-review`、`inbox-triage`、`memory-consolidation` 和
`shared-task-sync`，每個工作流程都設有核准關卡。AI 可用時會負責判斷
（分類），無法使用時則退回確定性規則。

- 討論串：[https://x.com/plattenschieber/status/2014508656335770033](https://x.com/plattenschieber/status/2014508656335770033)
- 儲存庫：[https://github.com/bloomedai/brain-cli](https://github.com/bloomedai/brain-cli)

## 相關內容

- [自動化](/zh-TW/automation) - 所有自動化機制
- [工具概覽](/zh-TW/tools) - 所有可用的代理工具
