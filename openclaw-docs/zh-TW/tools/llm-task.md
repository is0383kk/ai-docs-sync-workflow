---
read_when:
    - 你想在工作流程中加入僅輸出 JSON 的 LLM 步驟
    - 你需要經過結構描述驗證的 LLM 輸出，以用於自動化作業
summary: 適用於工作流程的純 JSON LLM 任務（選用外掛工具）
title: LLM 任務
x-i18n:
    generated_at: "2026-07-26T08:09:39Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 78ea533f43546fbdd66c7f7138b8dea0b12b02d38925689324b390a12d0c4c5a
    source_path: tools/llm-task.md
    workflow: 16
---

`llm-task` 是一個隨附的**選用外掛工具**，會執行一次僅限 JSON 的
LLM 呼叫並傳回結構化輸出，也可選擇依據 JSON
Schema 驗證輸出。它讓 Lobster 等工作流程引擎能使用 LLM 步驟，而不必為每個工作流程撰寫自訂
OpenClaw 程式碼。

## 啟用

1. 啟用外掛：

```json
{
  "plugins": {
    "entries": {
      "llm-task": { "enabled": true }
    }
  }
}
```

2. 允許此工具：

```json
{
  "tools": {
    "alsoAllow": ["llm-task"]
  }
}
```

`alsoAllow` 會在現用工具設定檔上加入 `llm-task`，而不會
限制其他核心工具。只有在需要限制性
允許清單模式時，才改用 `tools.allow`。

## 設定（選用）

```json
{
  "plugins": {
    "entries": {
      "llm-task": {
        "enabled": true,
        "config": {
          "defaultProvider": "openai",
          "defaultModel": "gpt-5.6-sol",
          "defaultAuthProfileId": "main",
          "allowedModels": ["openai/gpt-5.6-sol"],
          "maxTokens": 800,
          "timeoutMs": 30000
        }
      }
    }
  }
}
```

`allowedModels` 是由 `provider/model` 字串組成的允許清單；任何要求使用
其他模型的請求都會遭到拒絕。其餘所有鍵都是各次呼叫的備用值，會在
工具呼叫省略該參數時使用。

## 工具參數

| 參數       | 類型   | 備註                                                                                                                                         |
| --------------- | ------ | --------------------------------------------------------------------------------------------------------------------------------------------- |
| `prompt`        | string | 必填。提供給 LLM 的任務指示。                                                                                                       |
| `input`         | any    | 選用承載資料；序列化為 JSON 後附加至提示詞。                                                                              |
| `schema`        | object | 選用的 JSON Schema，剖析後的輸出必須通過其驗證。                                                                                 |
| `provider`      | string | 覆寫 `defaultProvider`／代理程式的預設供應商。                                                                                   |
| `model`         | string | 覆寫 `defaultModel`；接受不含前綴的模型 ID、別名或 `provider/model` 參照（重複的供應商前綴會自動移除）。 |
| `thinking`      | string | 推理層級（例如 `low`、`medium`）；必須是解析後模型支援的層級。                                                          |
| `authProfileId` | string | 覆寫 `defaultAuthProfileId`。                                                                                                             |
| `temperature`   | number | 僅盡力套用；並非所有供應商都會遵循。                                                                                                      |
| `maxTokens`     | number | 輸出權杖數的盡力上限。                                                                                                             |
| `timeoutMs`     | number | 執行逾時時間；預設為 `30000`。                                                                                                                 |

## 輸出

傳回 `details.json`（經剖析並通過結構描述驗證的 JSON），以及指出實際執行項目的 `details.provider`
和 `details.model`。

## 範例：Lobster 工作流程步驟

### 重要限制

以下範例假設**獨立版 Lobster 命令列介面**執行所在環境中的
`openclaw.invoke` 已具備正確的閘道 URL／驗證情境。

對於 OpenClaw 內隨附的**內嵌式** Lobster 執行器，這種巢狀命令列介面
模式**目前並不可靠**：

```lobster
openclaw.invoke --tool llm-task --action json --args-json '{ ... }'
```

在內嵌式 Lobster 尚未提供支援此流程的橋接機制前，建議優先使用：

- 在 Lobster 之外直接呼叫 `llm-task` 工具，或
- 不依賴巢狀 `openclaw.invoke` 呼叫的 Lobster 步驟。

獨立版 Lobster 命令列介面範例：

```lobster
openclaw.invoke --tool llm-task --action json --args-json '{
  "prompt": "根據輸入的電子郵件，傳回意圖和草稿。",
  "thinking": "low",
  "input": {
    "subject": "您好",
    "body": "可以幫忙嗎？"
  },
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

## 安全注意事項

- **僅限 JSON**：模型會收到只傳回 JSON 值的指示，不得包含程式碼
  圍欄或任何說明。
- **無工具**：底層執行已停用工具，因此模型無法在
  任務執行期間進行外部呼叫。
- 除非使用 `schema` 驗證輸出，否則應將其視為不受信任。
- 任何使用此輸出並會產生副作用的步驟（傳送、發布、執行）之前，
  都應先進行核准。

## 相關內容

- [推理層級](/zh-TW/tools/thinking)
- [子代理程式](/zh-TW/tools/subagents)
- [斜線命令](/zh-TW/tools/slash-commands)
