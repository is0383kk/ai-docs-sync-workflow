---
read_when:
    - 你看到一個 `.experimental` 設定鍵，想知道它是否穩定
    - 你想要試用預覽版執行階段功能，同時避免將其與一般預設值混淆
    - 你想要有一個地方可查找目前文件記載的實驗性旗標
summary: OpenClaw 中的實驗性旗標代表什麼，以及目前有哪些旗標已記載於文件中
title: 實驗性功能
x-i18n:
    generated_at: "2026-07-26T07:39:37Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 6c14b74bbafce77c0d1e1358ad94053675c4aad9e26be78719f58e78f455c3a2
    source_path: concepts/experimental-features.md
    workflow: 16
---

實驗性功能是置於明確旗標之後的預覽介面。它們需要更多實際使用驗證，才會獲得穩定的預設值或長期有效的契約。

- 預設關閉，除非文件說明了範圍明確的自動設定規則。
- 其結構與行為的變更速度可能比穩定設定更快。
- 若已有穩定路徑，請優先使用。
- 請先在較小的環境中測試，再廣泛部署。

## 目前已記錄的旗標

| 介面                     | 鍵                                                                                            | 適用時機                                                                                                                          | 更多資訊                                                                               |
| ------------------------ | --------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------- |
| 本機模型執行環境         | `agents.defaults.experimental.localModelLean`, `agents.entries.*.experimental.localModelLean` | 較小型或限制較嚴格的本機後端無法處理 OpenClaw 完整的預設工具介面                                                                 | [本機模型](/zh-TW/gateway/local-models)                                                      |
| Codex 執行框架            | `plugins.entries.codex.config.appServer.experimental.sandboxExecServer`                       | 你想讓原生 Codex app-server 0.132.0 或更新版本以 OpenClaw 沙箱支援的 exec-server 為目標，而不是停用 Code Mode                     | [Codex 執行框架參考](/zh-TW/plugins/codex-harness-reference#sandboxed-native-execution)      |
| 結構化規劃工具           | `tools.experimental.planTool`                                                                 | 你想在相容的執行環境與 UI 中公開結構化 `update_plan` 工具，以追蹤多步驟工作                                                       | [閘道設定參考](/zh-TW/gateway/config-tools#toolsexperimental)                               |
| Code Mode                | `tools.codeMode.enabled`                                                                      | 你想以精簡、由程式碼協調的方式存取隱藏的 OpenClaw 工具目錄                                                                        | [Code Mode](/zh-TW/tools/code-mode)                                                          |
| Swarm                    | `tools.swarm.enabled`                                                                         | 你想讓 Code Mode 指令碼平行協調範圍受限的子代理程式群組                                                                           | [Swarm](/zh-TW/tools/swarm)                                                                  |

## Control UI 實驗室

開啟 **Settings → Agents & Tools → Labs**，管理具有
Control UI 開關的實驗。啟用或停用實驗室功能時，系統會立即修補標準閘道
設定；只有功能需要重新啟動時，頁面才會顯示重新啟動提示。

Code Mode 和 Swarm 是目前已發布的 Labs 項目。這兩個開關都會
寫入現有且經過驗證的設定鍵，通常無須重新啟動閘道，即可在後續代理程式
執行時生效。

## 本機模型精簡模式

`agents.defaults.experimental.localModelLean: true` 會在每一輪中，從代理程式的直接介面移除重量級選用工具：`browser`、`cron`、`message`、`image_generate`、`music_generate`、`video_generate`、`tts` 與 `pdf`。明確允許或傳遞所需的工具仍可使用，但 Tool Search 可能會將它們納入目錄，而非直接公開。當尚未設定 `tools.toolSearch` 時，精簡模式也會將外掛／MCP／用戶端目錄預設為結構化 Tool Search（`tool_search`、`tool_describe`、`tool_call`）。使用 `agents.entries.*.experimental.localModelLean` 可將此設定範圍限制於單一代理程式。

在新手引導期間，當該值不存在時，經過驗證的 `ollama` 或 `lmstudio` 推論路由會自動設定 `agents.defaults.experimental.localModelLean: true`。OpenClaw 會記錄此設定來自新手引導，因此之後經過驗證的非本機路由只會移除這項自動設定。明確設定的 `true` 或 `false` 會予以保留。系統不會根據模型名稱或 URL 推斷其他自行託管及 OpenAI 相容的供應商。

如果你已全域調整 Tool Search，OpenClaw 不會變更該設定。設定 `tools.toolSearch: false` 可選擇不使用精簡模式的 Tool Search 預設值。

在結構化 `tools` 模式中，精簡執行會讓 `exec` 直接顯示在 Tool Search 控制項旁，讓針對程式設計調校的本機模型仍可選擇熟悉的 shell 路徑。這只會變更結構描述的可見性：一般工具政策、沙箱機制與 exec 核准仍然適用。明確的 `code` 與 `directory` 模式會維持一般的壓縮行為。

### 為何選擇這些工具

這些工具的說明最長、參數結構最廣，或最容易使小型模型偏離一般的程式設計與對話路徑。在上下文較小或限制較嚴格的 OpenAI 相容後端上，這會造成以下差異：

- 工具結構描述能容納於提示詞中，而非排擠對話記錄。
- 模型能選擇正確工具，而非因為有太多相似的結構描述而發出格式錯誤的工具呼叫。
- Chat Completions 轉接器能維持在結構化輸出限制內，而非因工具呼叫承載內容大小而收到 400 錯誤。

移除這些工具只會縮短直接工具清單。模型仍具有 `read`、`write`、`edit`、`exec`、`apply_patch`、影像理解、網路搜尋／擷取（若已設定）、記憶體，以及工作階段／代理程式工具。除非你設定 `tools.toolSearch: false`，否則額外目錄仍可透過 Tool Search 存取；明確允許工具可讓精簡代理程式重新使用已裁減的工作流程。

### 何時啟用

當你已確認模型可與閘道通訊，但完整的代理程式輪次發生異常時，請啟用精簡模式：

1. `openclaw infer model run --gateway --model <ref> --prompt "Reply with exactly: pong"` 成功。
2. 一般代理程式輪次因工具呼叫格式錯誤、提示詞過大，或模型忽略其工具而失敗。
3. 切換 `localModelLean: true` 後，失敗情況消失。

### 何時維持關閉

如果你的後端可順利處理完整的預設執行環境，請維持關閉。這是供需要較小工具介面的本機技術堆疊使用的因應措施，並非託管模型或資源充足之本機設備的預設選項。

精簡模式不會取代 `tools.profile`、`tools.allow`/`tools.deny`，或模型的 `compat.supportsTools: false` 備援機制。若要為特定代理程式永久使用範圍較窄的工具介面，請優先使用這些穩定的調整項目。

### 啟用

```json5
{
  agents: {
    defaults: {
      experimental: {
        localModelLean: true,
      },
    },
  },
}
```

僅套用於一個代理程式：

```json5
{
  agents: {
    list: [
      {
        id: "local",
        model: "lmstudio/gemma-4-e4b-it",
        experimental: {
          localModelLean: true,
        },
      },
    ],
  },
}
```

變更旗標後，請重新啟動閘道。除非你使用 `tools.allow` 或 `tools.alsoAllow` 明確保留，否則精簡篩選會移除 `browser`、`cron`、`message`、`image_generate`、`music_generate`、`video_generate`、`tts` 與 `pdf`；Tool Search 仍可能將保留的工具納入目錄，而非直接公開。

## 實驗性不代表隱藏

實驗性功能應在文件與設定路徑本身明確標示，不應藏在看似穩定的預設調整項目之後。

## 相關內容

- [功能](/zh-TW/concepts/features)
- [發布管道](/zh-TW/install/development-channels)
