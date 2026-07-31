---
read_when:
    - 你想要使用官方 Codex app-server 測試框架
    - 你需要 Codex 控制框架的設定範例
    - 你希望僅限 Codex 的部署在失敗時直接終止，而不是回退到 OpenClaw
summary: 透過官方 Codex app-server 測試框架執行 OpenClaw 內嵌代理程式輪次
title: Codex 測試框架
x-i18n:
    generated_at: "2026-07-26T07:48:07Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: e016a1689af65c5520d529ce22a87bd25ee29369f7aedca77b27f943a7f21b0f
    source_path: plugins/codex-harness.md
    workflow: 16
---

官方 `codex` 外掛透過 Codex
app-server 執行內嵌的 OpenAI 代理程式回合，而非使用內建的 OpenClaw 控制框架。Codex 負責
底層代理程式工作階段：原生執行緒續接、原生工具續接、
原生壓縮，以及 app-server 執行。OpenClaw 仍負責聊天
頻道、工作階段檔案、模型選擇、OpenClaw 動態工具、核准、
媒體傳遞，以及可見的逐字稿鏡像。

請使用標準 OpenAI 模型參照，例如 `openai/gpt-5.6-sol`。請勿設定
舊版 Codex GPT 參照；請將 OpenAI 代理程式驗證順序放在 `auth.order.openai` 下。
舊版 Codex 驗證設定檔 ID 與舊版 Codex 驗證順序項目會由
`openclaw doctor --fix` 修復。

當供應商／模型執行階段原則未設定或為 `auto` 時，僅有 `openai/*` 前綴
絕不會選取此控制框架。只有在路由為完全相符的官方 HTTPS Platform Responses 或 ChatGPT Responses，且
沒有自行設定的要求覆寫時，OpenAI 才可能隱式選取 Codex。請參閱
[OpenAI 隱式代理程式執行階段](/zh-TW/providers/openai#implicit-agent-runtime)。
如果在確定 Platform 與 ChatGPT 路由之前，驗證由 Codex 負責，OpenClaw
仍要求每個候選路由宣告與 Codex 相容。僅由原生功能負責
驗證絕不會略過該路由檢查。

當沒有啟用 OpenClaw 沙箱時，OpenClaw 會以啟用 Codex 原生程式碼模式的方式啟動 Codex app-server 執行緒
（預設仍會關閉僅限程式碼模式），因此原生工作區／程式碼功能可與透過 app-server `item/tool/call` 橋接器
路由的 OpenClaw 動態工具並存。除非你選擇加入實驗性的沙箱 exec-server 路徑，
否則啟用中的 OpenClaw 沙箱或受限工具原則會完全停用原生程式碼模式。

使用預設的 `tools.exec.host: "auto"` 且沒有啟用 OpenClaw 沙箱時，
Codex 也會收到用於在已配對節點上執行命令的 `node_exec` 和 `node_process` 工具。
原生殼層仍位於 Codex app-server 主機與工作區
（預設 stdio 部署是在閘道本機）；`node_exec` 會依
名稱或 ID 選取節點，並持續套用 OpenClaw 的節點核准原則。如果有限的
執行階段允許清單停用原生程式碼模式，導致該回合沒有
執行環境，OpenClaw 會改為保留經原則篩選的 `exec` 和 `process`
工具，以供直接、未經沙箱隔離的執行。

此 Codex 原生功能與
[OpenClaw 程式碼模式](/zh-TW/tools/code-mode)不同；後者是選用的 QuickJS-WASI 執行階段，
供一般 OpenClaw 執行使用，且具有不同的 `exec` 輸入格式。若要瞭解
更廣泛的模型／供應商／執行階段劃分，請先參閱
[代理程式執行階段](/zh-TW/concepts/agent-runtimes)：`openai/gpt-5.6-sol` 是模型
參照、`codex` 是執行階段，而 Telegram、Discord、Slack 或其他
頻道則是通訊介面。

## 需求

- 已安裝官方 `@openclaw/codex` 外掛。如果你的設定使用允許清單，請在
  `plugins.allow` 中包含 `codex`。
- 穩定版 Codex app-server，版本範圍為 `0.143.0` 至 `0.145.0`。此外掛預設會管理相容的
  二進位檔，因此 `PATH` 上的 `codex` 命令不會影響正常
  啟動。
- 透過 `openclaw models auth login --provider openai` 進行 Codex 驗證、
  代理程式 Codex 主目錄中已有的 app-server 帳號，或
  明確的 Codex API 金鑰驗證設定檔。

如需驗證優先順序、環境隔離、自訂 app-server 命令、
模型探索及完整設定欄位清單，請參閱
[Codex 控制框架參考資料](/zh-TW/plugins/codex-harness-reference)。

## 快速開始

安裝官方外掛，然後使用 Codex OAuth 登入：

```bash
openclaw plugins install @openclaw/codex
openclaw models auth login --provider openai
```

啟用 `codex` 外掛並選取 OpenAI 代理程式模型：

```json5
{
  plugins: {
    entries: {
      codex: {
        enabled: true,
      },
    },
  },
  agents: {
    defaults: {
      model: "openai/gpt-5.6-sol",
    },
  },
}
```

如果你的設定使用 `plugins.allow`，也請在其中加入 `codex`：

```json5
{
  plugins: {
    allow: ["codex"],
    entries: {
      codex: {
        enabled: true,
      },
    },
  },
}
```

變更外掛設定後，請重新啟動閘道。如果聊天已有
工作階段，請先執行 `/new` 或 `/reset`，讓下一個回合能依目前設定解析控制框架。

## 與 Codex Desktop 和命令列介面共用執行緒

預設的 `appServer.homeScope: "agent"` 會將每個 OpenClaw 代理程式與
操作者的原生 Codex 狀態隔離。若要讓擁有者檢查及管理
Codex Desktop 與 Codex 命令列介面顯示的相同原生執行緒，請選擇使用
使用者 Codex 主目錄：

```json5
{
  plugins: {
    entries: {
      codex: {
        enabled: true,
        config: {
          appServer: {
            homeScope: "user",
          },
        },
      },
    },
  },
}
```

使用者主目錄模式支援本機受管理的 stdio 程序或共用 Unix socket
傳輸。設定時使用 `$CODEX_HOME`，否則使用 `~/.codex`，包括
該主目錄的原生 Codex 驗證、設定、外掛及執行緒儲存區。OpenClaw 不會
將 OpenClaw 驗證設定檔注入此 app-server。

擁有者回合會取得 `codex_threads` 工具：列出、搜尋、讀取、分叉、重新命名、
封存及還原原生執行緒。分叉執行緒可在
OpenClaw 中繼續執行；分叉會附加至目前的 OpenClaw 工作階段，並持續
對其他原生 Codex 用戶端可見。封存前必須明確
確認該執行緒已在其他位置關閉。若同時啟用監督功能，
逐字稿欄位與變更操作需要啟用相符的
`supervision.allowRawTranscripts` 或 `supervision.allowWriteControls` 選項。

請勿透過各自獨立管理的
stdio App Server 同時續接或寫入同一執行緒。Codex 只會在單一 App Server 內協調即時寫入者，
不會跨不同程序協調。對一般
使用者主目錄 stdio 工作階段而言，分叉是安全的共存方式。

僅設定 `appServer.homeScope: "user"` 不會控制機群目錄。外掛啟用時，
原生工作階段探索功能即會啟用；若要將它從 OpenClaw 側邊欄移除，但不
停用 Codex，請設定 `sessionCatalog.enabled: false`。該目錄使用獨立的監督連線；若沒有
明確的 `appServer` 連線設定，該連線預設會使用受管理的
使用者主目錄 stdio，而一般控制框架仍維持代理程式範圍。兩條路徑都會採用明確的
`appServer` 設定。如上所示，當一般控制框架也應共用原生狀態時，
請明確設定 `homeScope: "user"`。

## 監督 Codex 工作階段

相同的 `codex` 外掛可以列出閘道
電腦與選擇加入之已配對節點上的未封存 Codex 工作階段。已儲存或閒置的閘道本機工作階段可以
建立模型鎖定的聊天，鏡像其有界限的持續保存使用者與助理
歷史記錄。其私有繫結會使用監督連線取得原生
快照、標準分支及後續回合，而一般 Codex 工作階段仍維持
代理程式範圍。第一次標準啟動會完全採用 Codex 為
快照分叉傳回的模型和供應商。後續續接會交由 Codex 的
原生設定選擇；外層 OpenClaw 模型與備援鏈絕不會取代
它。明確確認沒有其他執行者後，即可封存已儲存和閒置的資料列。
使用中的來源無法建立分支或封存；仍可開啟現有的
受監督聊天。已配對節點的工作階段仍僅提供中繼資料。

如需設定、分支
規則、已配對節點限制、中繼資料揭露及疑難排解，請參閱[監督 Codex 工作階段](/zh-TW/plugins/codex-supervision)。

## 設定

| 需求                                                | 設定                                                                                              | 位置                              |
| --------------------------------------------------- | ------------------------------------------------------------------------------------------------ | ---------------------------------- |
| 啟用控制框架                                  | `plugins.entries.codex.enabled: true`                                                            | OpenClaw 設定                    |
| 隱藏原生 Codex 工作階段探索                 | `plugins.entries.codex.config.sessionCatalog.enabled: false`                                     | Codex 外掛設定                |
| 保留採用允許清單的外掛安裝                  | 在 `plugins.allow` 中包含 `codex`                                                               | OpenClaw 設定                    |
| 允許符合資格的 OpenAI 回合隱式使用 Codex | 完全相符的官方 HTTPS Responses／ChatGPT 路由、沒有自行設定的要求覆寫、執行階段未設定／`auto` | OpenAI 供應商／模型設定       |
| 使用 ChatGPT／Codex OAuth 登入                    | `openclaw models auth login --provider openai`                                                   | 命令列介面驗證設定檔                   |
| 為 Codex 執行新增 API 金鑰備援                   | 在 `auth.order.openai` 中，將 `openai:*` API 金鑰設定檔列於訂閱驗證之後                 | 命令列介面驗證設定檔 + OpenClaw 設定 |
| Codex 無法使用時採取失敗關閉               | 供應商或模型 `agentRuntime.id: "codex"`                                                     | OpenClaw 模型／供應商設定     |
| 使用直接 OpenAI API 流量                       | 供應商或模型 `agentRuntime.id: "openclaw"` 搭配一般 OpenAI 驗證                          | OpenClaw 模型／供應商設定     |
| 調整 app-server 行為                            | `plugins.entries.codex.config.appServer.*`                                                       | Codex 外掛設定                |
| 啟用原生 Codex 外掛應用程式                     | `plugins.entries.codex.config.codexPlugins.*`                                                    | Codex 外掛設定                |
| 啟用 Codex Computer Use                           | `plugins.entries.codex.config.computerUse.*`                                                     | Codex 外掛設定                |

若要採用訂閱優先、API 金鑰備援的順序，建議使用 `auth.order.openai`。
現有舊版 Codex 驗證設定檔 ID 與舊版 Codex 驗證順序屬於
僅供 doctor 處理的舊版狀態；請勿寫入新的舊版 Codex GPT 參照。

```json5
{
  auth: {
    order: {
      openai: ["openai:user@example.com", "openai:api-key-backup"],
    },
  },
}
```

對於與 Codex 相容的有效路由，上述兩個設定檔仍會是
同一次 Codex 執行的候選項目。設定檔順序選擇的是認證資訊，而非執行階段。
變更驗證順序不會使自訂、Completions、HTTP 或
已覆寫要求的路由變得與 Codex 相容。

### 壓縮

請勿在 Codex 支援的
代理程式上設定 `compaction.model` 或 `compaction.provider`。Codex 會透過其原生 app-server 執行緒狀態進行壓縮，因此
OpenClaw 在執行階段會忽略這些本機摘要器覆寫，且當代理程式使用 Codex 時，
`openclaw doctor --fix` 會移除它們。

Lossless 仍可作為 Codex 回合周圍的組合、擷取及
維護所使用的上下文引擎，並透過
`plugins.slots.contextEngine: "lossless-claw"` 和
`plugins.entries.lossless-claw.config.summaryModel` 設定，而不是透過
`agents.defaults.compaction.provider`。當 Codex 是啟用中的執行階段時，`openclaw doctor --fix` 會將
舊的 `compaction.provider: "lossless-claw"` 格式遷移至 Lossless
上下文引擎位置，但壓縮仍由原生 Codex
負責。原生 app-server 控制框架支援需要在提示前組合內容的上下文引擎；
一般命令列介面後端（包括 `codex-cli`）不提供該主機功能。

對於 Codex 支援的代理程式，`/compact` 會在已繫結的執行緒上啟動原生 Codex app-server
壓縮，並等待其終止結果。共用的
`agents.defaults.compaction.timeoutSeconds` 預算適用；逾時時，
OpenClaw 會要求 Codex 中斷原生回合，並保留每個執行緒的隔離柵欄，
直到確認終止為止。它絕不會退回使用上下文引擎或
公開 OpenAI 摘要器。如果原生 Codex 執行緒繫結遺失或
過期，該命令會採取失敗關閉，而不會無聲切換壓縮
後端。

### 直接 API 長上下文

Codex 訂閱與直接 OpenAI API 流量是分開的合約。即時
ChatGPT/Codex 目錄通常提供 `272000` 個權杖的模型視窗，
而 OpenAI 文件記載的平台 API 視窗為 `1050000` 個權杖，GPT-5.5 和 GPT-5.6 的
最大輸出為 `128000`。保留完整輸出額度後，
推導出的輸入預算為 `922000` 個權杖。輸入超過 `272000`
個權杖的要求會採用 OpenAI 較高的長上下文定價。

請從與已安裝 Codex 版本相容的完整 Codex 模型目錄開始。對於每個應使用長上下文的
直接 GPT-5.5 或 GPT-5.6 項目，請保留描述項的其餘內容並設定：

```json
{
  "context_window": 922000,
  "max_context_window": 922000,
  "auto_compact_token_limit": 700000
}
```

Codex 會對目錄中的 `922000` 值套用一般的 95% 有效視窗保留量，
因此會回報約 `875900` 個可用權杖。在 `700000`
時進行壓縮，會在該有效防護限制前留下 `175900` 個權杖，並在
供應商安全輸入額度前留下 `222000` 個權杖。這個較大的餘裕是刻意設計的：Codex 會在加入下一則使用者訊息和上下文
更新前，檢查已記錄的上下文，因此此門檻除了工具、
指示、序列化和壓縮輪次本身之外，還必須涵蓋一次大型的傳入輪次。

若要獨立使用 Codex 命令列介面或 Desktop，命令驗證自訂供應商可以
從系統鑰匙圈或密鑰管理員讀取 API 金鑰，同時保留一般
ChatGPT 登入供連接器使用：

```toml
model = "gpt-5.6-terra"
model_provider = "openai_api_direct"
model_context_window = 922000
model_auto_compact_token_limit = 700000
model_auto_compact_token_limit_scope = "total"
model_catalog_json = "/absolute/path/to/models-api-1m.json"

[model_providers.openai_api_direct]
name = "OpenAI API direct"
base_url = "https://api.openai.com/v1"
wire_api = "responses"
requires_openai_auth = false

[model_providers.openai_api_direct.auth]
command = "/absolute/path/to/read-openai-inference-key"
timeout_ms = 5000
refresh_interval_ms = 300000
```

驗證輔助程式必須只將金鑰輸出至 stdout。請勿將其放入 TOML。

對於 OpenClaw Codex app-server 執行框架，請保留預設的代理程式範圍 Codex
主目錄，並讓 OpenClaw 注入 `openai` API 金鑰設定檔。請將目錄和
上下文限制作為原生 Codex app-server 引數傳入：

```json5
{
  auth: {
    order: {
      openai: ["openai:api-key"],
    },
  },
  plugins: {
    entries: {
      codex: {
        enabled: true,
        config: {
          appServer: {
            args: [
              "app-server",
              "--listen",
              "stdio://",
              "-c",
              'model_catalog_json="/absolute/path/to/models-api-1m.json"',
              "-c",
              "model_context_window=922000",
              "-c",
              "model_auto_compact_token_limit=700000",
              "-c",
              "model_auto_compact_token_limit_scope=total",
            ],
          },
        },
      },
    },
  },
  agents: {
    defaults: {
      model: "openai/gpt-5.6-terra",
      models: {
        "openai/gpt-5.6-terra": { agentRuntime: { id: "codex" } },
      },
    },
  },
}
```

如有需要，請將 `openai:api-key` 替換為實際的 API 金鑰設定檔 ID。此
代理程式範圍的 app-server 只會收到準備好的金鑰；操作者原生的
`~/.codex` ChatGPT 登入、外掛、連接器和討論串儲存區皆不受影響。
Codex app-server `0.144.6` 不會在 app-server 輪次中附加命令驗證自訂
供應商的 bearer，因此此路徑請使用上述注入的 API 金鑰路徑，
而不要使用 `homeScope: "user"`。

變更目錄或 app-server 引數後，請重新啟動閘道並
開始新的聊天。現有的原生討論串會保留其已記錄的供應商
和模型設定。請使用 `/status` 和 `/codex status` 驗證執行階段，然後
在開始長時間工作階段前，傳送一次無害的直接 API 輪次。

<Warning>
長上下文是刻意設為選擇性啟用。輸入超過 `272000` 個權杖後，OpenAI 會針對整個要求收取
2× 輸入費率和 1.5× 輸出費率。存取權、實際限制和計費仍以 API
為準。請參閱
[OpenAI 模型限制](https://developers.openai.com/api/docs/models/compare)和
[API 定價](https://developers.openai.com/api/docs/pricing)。
</Warning>

本頁其餘內容涵蓋部署形式、失敗時關閉路由、守護程式
核准政策、原生 Codex 外掛和 Computer Use。如需完整的選項
清單、預設值、列舉、探索、環境隔離、逾時和
app-server 傳輸欄位，請參閱
[Codex 執行框架參考](/zh-TW/plugins/codex-harness-reference)。

## 驗證 Codex 執行階段

請在預期使用 Codex 的聊天中使用 `/status`。由 Codex 支援的 OpenAI
代理程式輪次會顯示：

```text
執行階段：OpenAI Codex
```

接著檢查 Codex app-server 狀態：

```text
/codex status
/codex models
/codex binding
```

`/codex binding` 會回報已連結的原生討論串和目前的模型設定。
`/codex status` 會回報 app-server 連線能力、帳號、速率限制、MCP
伺服器和 Skills。`/codex models` 會列出執行框架和帳號的即時 Codex app-server 目錄。
如果 `/status` 的結果出乎預期，請參閱
[疑難排解](#troubleshooting)。

## 路由與模型選擇

請將供應商參照和執行階段政策分開：

- 請使用 `openai/gpt-*` 進行標準 OpenAI 模型選擇。僅有前綴
  絕不會選取 Codex。
- 當執行階段未設定或為 `auto` 時，只有完全相符的官方 HTTPS Platform Responses
  或 ChatGPT Responses 路由，且沒有自行撰寫的要求覆寫，才能隱式選取 Codex。
- 請勿在設定中使用舊版 Codex GPT 參照；請執行 `openclaw doctor --fix`
  以修復舊版參照和過時的工作階段路由固定設定。
- `agentRuntime.id: "codex"` 會使 Codex 成為相容路由的失敗時關閉必要條件。
  它不會讓不相容的有效路由變得相容。
- `agentRuntime.id: "openclaw"` 會在有意如此設定時，讓供應商或模型選擇加入內嵌的
  OpenClaw 執行階段。
- `/codex ...` 可從聊天控制原生 Codex app-server 對話。
- ACP/acpx 是獨立的外部執行框架路徑。只有在使用者
  要求 ACP/acpx 或外部執行框架轉接器時才使用。

| 使用者意圖                                                | 使用                                                                                                   |
| ---------------------------------------------------------- | ----------------------------------------------------------------------------------------------------- |
| 連結目前的聊天                                    | `/codex bind [thread-id] [--cwd <path>] [--model <model>] [--provider <provider>]`                    |
| 繼續現有的 Codex 討論串                            | `/codex resume <thread-id>`                                                                           |
| 列出或篩選 Codex 討論串                               | `/codex threads [filter]`                                                                             |
| 讀取或更新已綁定討論串的原生目標              | `/codex goal [status\|set <objective>\|pause\|resume\|block\|complete\|clear]`                        |
| 列出原生 Codex 外掛                                  | `/codex plugins list`                                                                                 |
| 啟用或停用已設定的原生 Codex 外掛         | `/codex plugins enable <name>`、`/codex plugins disable <name>`                                       |
| 將已儲存的 Codex 命令列介面工作階段作為配對節點輪次繼續    | `/codex sessions --host <node> [filter]`，接著使用 `/codex resume <session-id> --host <node> --bind here` |
| 檢視跨電腦的非封存 Codex 工作階段          | 啟用 Codex 監督並開啟 **Codex 工作階段**                                                  |
| 變更已綁定討論串的模型、快速模式或權限 | `/codex model <model>`、`/codex fast [on\|off\|status]`、`/codex permissions [default\|yolo\|status]` |
| 停止或引導進行中的輪次                              | `/codex stop`、`/codex steer <text>`                                                                  |
| 解除目前的綁定                                 | `/codex detach`（別名 `/codex unbind`）                                                               |
| 僅傳送 Codex 意見回饋                                   | `/codex diagnostics [note]`                                                                           |
| 啟動 ACP/acpx 工作                                     | ACP/acpx 工作階段命令，而非 `/codex`                                                               |

| 使用案例                                        | 設定                                                                                                   | 驗證                                  | 備註                                      |
| ----------------------------------------------- | ----------------------------------------------------------------------------------------------------------- | --------------------------------------- | ------------------------------------------ |
| 具備原生 Codex 執行階段資格的 OpenAI 路由 | 完全相符的官方 HTTPS Responses/ChatGPT 路由，且沒有自行撰寫的要求覆寫，另加已啟用的 `codex` 外掛 | `/status` 顯示 `Runtime: OpenAI Codex` | 執行階段未設定或為 `auto` 時使用的隱式路徑 |
| Codex 無法使用時失敗並關閉             | 供應商或模型 `agentRuntime.id: "codex"`                                                                | 輪次失敗，而非退回內嵌模式 | 用於僅限 Codex 的部署             |
| 透過 OpenClaw 傳送直接 OpenAI API 金鑰流量  | 供應商或模型 `agentRuntime.id: "openclaw"` 和一般 OpenAI 驗證                                      | `/status` 顯示 OpenClaw 執行階段        | 僅在有意使用 OpenClaw 時使用      |
| 舊版設定                                   | 舊版 Codex GPT 參照                                                                                       | `openclaw doctor --fix` 會將其重寫     | 請勿以此方式撰寫新設定           |
| ACP/acpx Codex 轉接器                          | ACP `sessions_spawn({ runtime: "acp" })`                                                                    | ACP 工作／工作階段狀態                 | 與原生 Codex 執行框架分開         |

`agents.defaults.imageModel` 遵循相同的前綴拆分方式。請使用 `openai/gpt-*`
作為一般 OpenAI 路由，只有在影像理解應透過有界限的 Codex app-server 輪次執行時，才使用 `codex/gpt-*`。
Doctor 會將舊版 Codex GPT 參照重寫為 `openai/gpt-*`。

## 部署模式

### 基本 Codex 部署

對於有效官方 HTTPS 路由符合隱式選取 Codex 資格的 OpenAI 模型，
請使用快速入門設定：

```json5
{
  plugins: {
    entries: {
      codex: {
        enabled: true,
      },
    },
  },
  agents: {
    defaults: {
      model: "openai/gpt-5.6-sol",
    },
  },
}
```

### 混合供應商部署

將 Claude 保持為預設代理程式，並新增具名 Codex 代理程式：

```json5
{
  plugins: {
    entries: {
      codex: {
        enabled: true,
      },
    },
  },
  agents: {
    defaults: {
      model: "anthropic/claude-opus-4-6",
    },
    list: [
      {
        id: "main",
        default: true,
        model: "anthropic/claude-opus-4-6",
      },
      {
        id: "codex",
        name: "Codex",
        model: "openai/gpt-5.6-sol",
      },
    ],
  },
}
```

`main` 代理程式使用其一般供應商路徑。當 `codex` 代理程式的有效 OpenAI 路由保持相容時，
它會使用 Codex app-server；如果這應是失敗時關閉的必要條件，請新增明確的模型範圍
`agentRuntime.id: "codex"`。

### 失敗時關閉的 Codex 部署

當隨附的外掛可用時，符合資格且完全相符的官方 HTTPS OpenAI 路由可以解析至 Codex。
若要制定明文的失敗時關閉規則，請新增明確的執行階段政策：

```json5
{
  models: {
    providers: {
      openai: {
        agentRuntime: {
          id: "codex",
        },
      },
    },
  },
  agents: {
    defaults: {
      model: "openai/gpt-5.6-sol",
    },
  },
  plugins: {
    entries: {
      codex: {
        enabled: true,
      },
    },
  },
}
```

強制使用 Codex 時，如果有效路由未宣告為與 Codex 相容、外掛已停用、app-server 版本過舊，或 app-server 無法啟動，OpenClaw 會提早失敗。

## App-server 政策

依預設，外掛會在本機啟動由 OpenClaw 管理的 Codex 二進位檔，並使用 stdio 傳輸。只有在刻意執行其他可執行檔時，才設定 `appServer.command`。Codex 將 WebSocket 傳輸歸類為實驗性且不受支援；僅將它用於針對已在其他位置執行之 app-server 的非正式環境測試：

```json5
{
  plugins: {
    entries: {
      codex: {
        enabled: true,
        config: {
          appServer: {
            transport: "websocket",
            url: "ws://gateway-host:39175",
            authToken: "${CODEX_APP_SERVER_TOKEN}",
          },
        },
      },
    },
  },
}
```

本機 stdio app-server 工作階段預設採用受信任的本機操作員態勢：`approvalPolicy: "never"`、`approvalsReviewer: "user"` 和 `sandbox: "danger-full-access"`。如果本機 Codex 要求不允許該隱含的 YOLO 態勢，OpenClaw 會改為選擇允許的 Guardian 權限。當工作階段啟用 OpenClaw 沙箱時，OpenClaw 會在該回合停用 Codex 原生 Code Mode、使用者 MCP 伺服器，以及由應用程式支援的外掛執行，而不依賴 Codex 主機端沙箱。當一般的 exec/process 工具可用時，Shell 存取會改為透過由 OpenClaw 沙箱支援的動態工具，例如 `sandbox_exec` 和 `sandbox_process`。

在沙箱逸出或額外權限之前，對 Codex 原生自動審查使用標準化的 OpenClaw exec 模式：

```json5
{
  tools: {
    exec: {
      mode: "auto",
    },
  },
  plugins: {
    entries: {
      codex: {
        enabled: true,
      },
    },
  },
}
```

對於 Codex app-server 工作階段，`tools.exec.mode: "auto"` 會對應至經 Codex Guardian 審查的核准：當本機要求允許這些值時，通常是 `approvalPolicy: "on-request"`、`approvalsReviewer: "auto_review"` 和 `sandbox: "workspace-write"`。在 `tools.exec.mode: "auto"` 中，OpenClaw 不會保留舊版不安全的 Codex `approvalPolicy: "never"` 或 `sandbox: "danger-full-access"` 覆寫；若要刻意採用 Codex 無須核准態勢，請使用 `tools.exec.mode: "full"`。舊版 `plugins.entries.codex.config.appServer.mode: "guardian"` 預設集仍可運作，但 `tools.exec.mode: "auto"` 是標準化的 OpenClaw 介面。

如需與主機 exec 核准及 ACPX 權限進行模式層級的比較，請參閱[權限模式](/zh-TW/tools/permission-modes)。如需瞭解每個 app-server 欄位、驗證順序、環境隔離和逾時行為，請參閱 [Codex 操作框架參考](/zh-TW/plugins/codex-harness-reference)。

## 指令與診斷

`codex` 外掛會在任何支援 OpenClaw 文字指令的頻道上，將 `/codex` 註冊為斜線指令。

原生執行與控制需要擁有者或 `operator.admin` 閘道用戶端：包括繫結或恢復執行緒、傳送或停止回合、變更模型、快速模式或權限狀態、進行壓縮或審查，以及中斷繫結。其他已獲授權的傳送者仍可使用唯讀的狀態、說明、帳戶、模型、執行緒、原生目標、MCP 伺服器、技能和繫結檢查指令。

常見形式：

- `/codex status` 檢查 app-server 連線能力、模型、帳戶、速率限制、MCP 伺服器和技能。
- `/codex models` 列出即時 Codex app-server 模型。
- `/codex threads [filter]` 列出最近的 Codex app-server 執行緒。
- `/codex goal` 讀取或更新所附加執行緒的原生 Codex 目標。Codex 自動目標接續仍維持停用；OpenClaw 尚未負責自主後續回合。
- `/codex resume <thread-id>` 將目前的 OpenClaw 工作階段附加至現有 Codex 執行緒。
- `/codex bind [thread-id] [--cwd <path>] [--model <model>] [--provider <provider>]`
  附加目前的聊天。
- `/codex detach`（或 `/codex unbind`）中斷目前的繫結。
- `/codex binding` 說明目前的繫結。
- `/codex stop` 停止進行中的回合；`/codex steer <text>` 會引導該回合。
- `/codex model <model>`、`/codex fast [on|off|status]` 和
  `/codex permissions [default|yolo|status]` 變更每個對話的狀態。
- `/codex compact` 要求 Codex app-server 壓縮所附加的執行緒。
- `/codex review` 為所附加的執行緒啟動 Codex 原生審查。
- `/codex diagnostics [note]` 會先詢問，再傳送所附加執行緒的 Codex 意見回饋。
- `/codex account` 顯示帳戶和速率限制狀態。
- `/codex mcp` 列出 Codex app-server MCP 伺服器狀態。
- `/codex skills` 列出 Codex app-server 技能。
- `/codex plugins list`、`/codex plugins enable <name>` 和
  `/codex plugins disable <name>` 管理已設定的原生 Codex 外掛。
- `/codex computer-use [status|install]` 管理 Codex Computer Use。
- `/codex help` 列出完整的指令樹。

對於大多數支援報告，請先在發生錯誤的對話中使用 `/diagnostics [note]`。它會建立一份閘道診斷報告，並針對 Codex 操作框架工作階段，要求核准傳送相關的 Codex 意見回饋套件。如需瞭解隱私權模型和群組聊天行為，請參閱[診斷匯出](/zh-TW/gateway/diagnostics)。只有在你明確想要上傳目前所附加執行緒的 Codex 意見回饋，而不包含完整的閘道診斷套件時，才使用 `/codex diagnostics [note]`。

### 在本機檢查 Codex 執行緒

檢查有問題之 Codex 執行的最快方式，通常是直接開啟原生 Codex 執行緒：

```bash
codex resume <thread-id>
```

從已完成的 `/diagnostics` 回覆、`/codex binding` 或 `/codex threads [filter]` 取得執行緒 ID。

如需瞭解上傳機制和執行階段層級的診斷界線，請參閱 [Codex 操作框架執行階段](/zh-TW/plugins/codex-harness-runtime#codex-feedback-upload)。

### 驗證順序

在預設的個別代理程式主目錄中，驗證會依下列順序選取：

1. 代理程式的有序 OpenAI 驗證設定檔，最好位於 `auth.order.openai` 下。執行 `openclaw doctor --fix`，以遷移較舊的 Codex 驗證設定檔 ID 和舊版 Codex 驗證順序。
2. 該代理程式 Codex 主目錄中的 app-server 現有帳戶。
3. 僅限本機 stdio app-server 啟動：當不存在 app-server 帳戶且仍需要 OpenAI 驗證時，依序使用 `CODEX_API_KEY`，再使用 `OPENAI_API_KEY`。

當 OpenClaw 偵測到 ChatGPT 訂閱形式的 Codex 驗證設定檔時，會從衍生的 Codex 子程序移除 `CODEX_API_KEY` 和 `OPENAI_API_KEY`。這可讓閘道層級的 API 金鑰繼續供嵌入或直接 OpenAI 模型使用，同時避免原生 Codex app-server 回合意外透過 API 計費。明確的 Codex API 金鑰設定檔和本機 stdio 環境金鑰後援會使用 app-server 登入，而不會使用繼承的子程序環境。WebSocket app-server 連線不會收到閘道環境的 API 金鑰後援；請使用明確的驗證設定檔或遠端 app-server 自己的帳戶。

如果訂閱設定檔達到 Codex 使用量限制，當 Codex 回報重設時間時，OpenClaw 會記錄該時間，並為同一次 Codex 執行嘗試下一個有序驗證設定檔。重設時間過後，訂閱設定檔會再次符合使用資格，而無須變更所選的 `openai/gpt-*` 模型或 Codex 執行階段。

設定原生 Codex 外掛後，OpenClaw 會先透過已連線的 app-server 安裝或重新整理這些外掛，再將外掛擁有的應用程式公開給 Codex 執行緒。`app/list` 仍是應用程式 ID、可存取性和中繼資料的事實來源，但 OpenClaw 負責每個執行緒的啟用決策：如果政策允許列出的可存取應用程式，即使 `app/list` 目前回報該應用程式已停用，OpenClaw 仍會傳送 `thread/start.config.apps[appId].enabled = true`。此路徑不會為未知 ID 虛構應用程式安裝；OpenClaw 只會使用 `plugin/install` 啟用市集外掛，然後重新整理清單。

### 環境隔離

對於本機 stdio app-server 啟動，OpenClaw 會將 `CODEX_HOME` 設為每個代理程式專屬的目錄，因此 Codex 設定、驗證／帳戶檔案、外掛快取／資料和原生執行緒狀態，依預設不會讀寫操作員個人的 `~/.codex`。OpenClaw 會保留一般程序的 `HOME`；Codex 執行的子程序仍可找到使用者主目錄中的設定和權杖，且 Codex 可能會探索共用的 `$HOME/.agents/skills` 和 `$HOME/.agents/plugins/marketplace.json` 項目。使用 `appServer.homeScope: "user"` 時，OpenClaw 會改用原生使用者 Codex 主目錄及其現有帳戶，而不會注入 OpenClaw 驗證設定檔。

如果部署需要額外的環境隔離，請將這些變數加入 `appServer.clearEnv`：

```json5
{
  plugins: {
    entries: {
      codex: {
        enabled: true,
        config: {
          appServer: {
            clearEnv: ["CODEX_API_KEY", "OPENAI_API_KEY"],
          },
        },
      },
    },
  },
}
```

`appServer.clearEnv` 只會影響衍生的 Codex app-server 子程序。OpenClaw 會在本機啟動標準化期間，從此清單移除 `CODEX_HOME` 和 `HOME`：`CODEX_HOME` 會繼續指向所選代理程式或使用者範圍，而 `HOME` 會繼續繼承，讓子程序能使用一般使用者主目錄狀態。

### 動態工具與網頁搜尋

Codex 動態工具預設採用 `searchable` 載入。OpenClaw 通常不會公開與 Codex 原生工作區操作重複的動態工具：`read`、`write`、`edit`、`apply_patch`、`exec`、`process`、`update_plan`、`get_goal`、`create_goal`、`update_goal`、`tool_call`、`tool_describe`、`tool_search` 和 `tool_search_code`。目標操作仍由 Codex 原生處理，因此 OpenClaw 不會將第二個目標儲存區投射至 Codex 回合。大多數其餘 OpenClaw 整合工具，例如訊息、媒體、排程、瀏覽器、節點、閘道和 `heartbeat_respond`，都可透過 `openclaw` 命名空間下的 Codex 工具搜尋使用，藉此縮小初始模型上下文。當有限允許清單停用原生 Code Mode 時，受限回合的 Shell 後援是 `exec` 和 `process` 的例外情況；執行階段允許清單和 `codexDynamicToolsExclude` 仍會套用。

標記為 `catalogMode: "direct-only"` 的工具（包括 OpenClaw `computer` 工具）會改用 `openclaw_direct` 命名空間。Codex 將該命名空間視為 `DirectModelOnly`，因此這些工具在一般和僅限 Code Mode 的執行緒中，會持續直接對模型可見，而不會跨越巢狀 Code Mode `tools.*` 呼叫。

啟用搜尋且未選取受管理的供應商時，網頁搜尋依預設會使用 Codex 託管的 `web_search` 工具。原生託管搜尋與 OpenClaw 受管理的 `web_search` 動態工具互斥，因此受管理的搜尋無法繞過原生網域限制。當託管搜尋無法使用、明確停用，或由選定的受管理供應商取代時，OpenClaw 會使用受管理的工具。OpenClaw 會維持停用 Codex 的獨立 `web.run` 擴充功能，因為正式環境 app-server 流量會拒絕其使用者定義的 `web` 命名空間。`tools.web.search.enabled: false` 會停用這兩條路徑，停用工具、僅使用 LLM 的執行亦同。Codex 將 `"cached"` 視為偏好設定，並將其解析為不受限制之 app-server 回合的即時外部存取。設定原生 `allowedDomains` 時，自動受管理後援會以失敗關閉方式處理，確保無法繞過允許清單。持續性的有效搜尋政策變更，會在下一個回合前輪替已繫結的 Codex 執行緒；暫時的每回合限制則使用暫時受限的執行緒，並保留現有繫結供稍後恢復。

`sessions_yield`、`sessions_spawn` 和僅限訊息工具的來源回覆會維持
直接呼叫，因為它們屬於回合控制或委派契約。指引仍
偏好將 Codex 原生的 `spawn_agent` 作為主要的 Codex 子代理介面，
而明確的 OpenClaw 或 ACP 委派仍可透過
`sessions_spawn` 直接呼叫。在 Codex 程式碼模式中，通用 OpenClaw
動態工具的結果是 JSON 文字，而非 JavaScript 物件，因此在讀取欄位前，
請先剖析看似 JSON 的結果。Codex 也會依序執行巢狀
動態呼叫；請在有界迴圈中提交多個 `sessions_spawn` 呼叫，而不要
預期 `Promise.all` 會並行啟動它們。已接受的
子代理在提交後續呼叫時仍可重疊執行。完整模式請參閱
[群集](/zh-TW/tools/swarm#use-swarm-from-other-harnesses)。
心跳偵測協作指示會要求 Codex 在結束心跳偵測回合前搜尋
`heartbeat_respond`，前提是該工具尚未載入。

僅在連線至無法搜尋延後載入動態工具的自訂
Codex app-server，或偵錯完整工具承載資料時，才設定 `codexDynamicToolsLoading: "direct"`。

### 設定欄位

支援的頂層 Codex 外掛欄位：

| 欄位                      | 預設值        | 意義                                                                                  |
| -------------------------- | -------------- | ---------------------------------------------------------------------------------------- |
| `codexDynamicToolsLoading` | `"searchable"` | 使用 `"direct"` 將 OpenClaw 動態工具直接放入初始 Codex 工具情境中。 |
| `codexDynamicToolsExclude` | `[]`           | 要從 Codex app-server 回合中省略的其他 OpenClaw 動態工具名稱。              |
| `codexPlugins`             | 已停用       | 對已遷移、從原始碼安裝之精選外掛的原生 Codex 外掛／應用程式支援。           |
| `sessionCatalog`           | 已啟用        | 在此閘道及符合資格的已配對節點上探索原生 Codex 工作階段的側邊欄。   |
| `supervision`              | 已停用       | 面向代理的原生工作階段逐字稿與寫入控制政策。                         |

支援的 `appServer` 欄位：

| 欄位                                         | 預設值                                                | 說明                                                                                                                                                                                                                                                                                                                                                                                         |
| --------------------------------------------- | ------------------------------------------------------ | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `transport`                                   | `"stdio"`                                              | `"stdio"` 會啟動 Codex；明確指定 `"unix"` 會連線至本機控制通訊端；`"websocket"` 會連線至 `url`。                                                                                                                                                                                                                                                                                |
| `homeScope`                                   | `"agent"`                                              | `"agent"` 會依各 OpenClaw 代理程式隔離一般測試框架狀態。`"user"` 是明確選擇加入的設定，會共用原生 `$CODEX_HOME` 或 `~/.codex`、使用原生驗證，並啟用僅限擁有者的執行緒管理。使用者範圍支援本機 stdio 或 Unix 傳輸。對於獨立的監督連線，未設定的值在 stdio 或 Unix 下會解析為 `"user"`，在 WebSocket 下則會解析為 `"agent"`。     |
| `command`                                     | 受管理的 Codex 二進位檔                                   | stdio 傳輸所使用的可執行檔。保留未設定即可使用受管理的二進位檔；僅在明確覆寫時才設定。                                                                                                                                                                                                                                                                                    |
| `args`                                        | `["app-server", "--listen", "stdio://"]`               | stdio 傳輸的引數。                                                                                                                                                                                                                                                                                                                                                                  |
| `url`                                         | 未設定                                                  | WebSocket App Server URL 或 `unix://` URL。明確指定空白的 Unix 路徑時，會選取標準的使用者家目錄控制通訊端。                                                                                                                                                                                                                                                                          |
| `authToken`                                   | 未設定                                                  | WebSocket 傳輸的 Bearer 權杖。接受常值字串或 SecretInput，例如 `${CODEX_APP_SERVER_TOKEN}`。                                                                                                                                                                                                                                                                              |
| `headers`                                     | `{}`                                                   | 額外的 WebSocket 標頭。標頭值接受常值字串或 SecretInput 值，例如 `x-codex-client-session-token: "${CODEX_CLIENT_SESSION_TOKEN}"`。                                                                                                                                                                                                                               |
| `clearEnv`                                    | `[]`                                                   | OpenClaw 建立繼承環境後，從啟動的 stdio app-server 程序中移除的額外環境變數名稱。對於本機啟動，OpenClaw 會保留所選的 `CODEX_HOME` 與繼承的 `HOME`。                                                                                                                                                                           |
| `codeModeOnly`                                | `false`                                                | 選擇加入 Codex 僅限程式碼模式的工具介面。一般 OpenClaw 動態工具仍可透過巢狀 `tools.*` 呼叫使用；`openclaw_direct` 工具仍會直接顯示給模型。                                                                                                                                                                                                             |
| `remoteWorkspaceRoot`                         | 未設定                                                  | 遠端 Codex app-server 工作區根目錄。設定後，OpenClaw 會從解析出的 OpenClaw 工作區推斷本機工作區根目錄、在此遠端根目錄下保留目前的 cwd 後綴，並只將最終的 app-server cwd 傳送給 Codex。若 cwd 位於解析出的 OpenClaw 工作區根目錄之外，OpenClaw 會採取失敗關閉，而不會將閘道本機路徑傳送至遠端 app-server。 |
| `requestTimeoutMs`                            | `60000`                                                | app-server 控制平面呼叫的逾時時間。                                                                                                                                                                                                                                                                                                                                                     |
| `turnCompletionIdleTimeoutMs`                 | `60000`                                                | Codex 接受一個回合後，或發出回合範圍的 app-server 要求後，OpenClaw 等待 `turn/completed` 時的靜默時段。                                                                                                                                                                                                                                                                    |
| `turnAssistantCompletionIdleTimeoutMs`        | `10000`                                                | 最終／非評論助理項目或工具執行前的原始助理完成訊號啟動助理輸出釋放後，OpenClaw 仍等待 `turn/completed` 時的靜默時段。提高此值可讓 Codex 有更多時間發出 `turn/completed`，之後 OpenClaw 才會中斷並釋放工作階段通道。                                                                                            |
| `postToolRawAssistantCompletionIdleTimeoutMs` | `300000`                                               | OpenClaw 等待 `turn/completed` 時，在工具交接、原生工具完成、工具執行後的原始助理進度、原始推理完成或推理進度之後使用的完成閒置與進度防護。適用於受信任或繁重的工作負載，因為工具執行後的整合可合理地維持靜默，且時間長於最終助理釋放預算。                                |
| `mode`                                        | `"yolo"`，除非本機 Codex 要求不允許 YOLO | YOLO 或經守護者審查執行的預設設定。本機 stdio 要求若省略 `danger-full-access`、`never` 核准或 `user` 審查者，隱含預設值會是守護者。                                                                                                                                                                                                           |
| `approvalPolicy`                              | `"never"` 或允許的守護者核准政策       | 傳送至執行緒啟動／繼續／回合的原生 Codex 核准政策。若允許，守護者預設值偏好 `"on-request"`。                                                                                                                                                                                                                                                                            |
| `sandbox`                                     | `"danger-full-access"` 或允許的守護者沙箱  | 傳送至執行緒啟動／繼續的原生 Codex 沙箱模式。若允許，守護者預設值偏好 `"workspace-write"`，否則使用 `"read-only"`。當 OpenClaw 沙箱處於作用中時，`danger-full-access` 回合會使用 Codex `workspace-write`，其網路存取權限取決於 OpenClaw 沙箱輸出設定。                                                                                     |
| `approvalsReviewer`                           | `"user"` 或允許的守護者審查者               | 若允許，使用 `"auto_review"` 讓 Codex 審查原生核准提示，否則使用 `guardian_subagent` 或 `user`。`guardian_subagent` 仍是舊版別名。                                                                                                                                                                                                                              |
| `serviceTier`                                 | 未設定                                                  | 選用的 Codex app-server 服務層級。`"priority"` 會啟用快速模式路由，`"flex"` 會要求彈性處理，`null` 會清除覆寫，而舊版 `"fast"` 會視為 `"priority"` 接受。                                                                                                                                                                                                 |
| `networkProxy`                                | 已停用                                               | 選擇加入 Codex 權限設定檔網路功能，以供 app-server 命令使用。OpenClaw 會定義所選的 `permissions.<profile>.network` 設定，並以 `default_permissions` 選取，而非傳送 `sandbox`。                                                                                                                                                                             |
| `experimental.sandboxExecServer`              | `false`                                                | 預覽版選擇加入設定，會向支援的 Codex app-server 註冊由 OpenClaw 沙箱支援的 Codex 環境，使原生 Codex 執行可在作用中的 OpenClaw 沙箱內運行。                                                                                                                                                                                                            |

`appServer.networkProxy` 是明確設定，因為它會變更 Codex 沙箱合約。啟用後，OpenClaw 也會在 Codex 執行緒設定中設定 `features.network_proxy.enabled`
和 `default_permissions`，讓產生的權限設定檔可以啟動 Codex 受管理網路。依預設，OpenClaw
會根據設定檔內容產生抗碰撞的 `openclaw-network-<fingerprint>` 設定檔
名稱；只有在需要穩定的本機名稱時，才使用 `profileName`。

```json5
{
  plugins: {
    entries: {
      codex: {
        config: {
          appServer: {
            sandbox: "workspace-write",
            networkProxy: {
              enabled: true,
              domains: {
                "api.openai.com": "allow",
                "blocked.example.com": "deny",
              },
              unixSockets: {
                "/tmp/proxy.sock": "allow",
                "/tmp/blocked.sock": "none",
              },
              allowUpstreamProxy: true,
              proxyUrl: "http://127.0.0.1:3128",
            },
          },
        },
      },
    },
  },
}
```

如果一般應用程式伺服器執行階段原本會是 `danger-full-access`，啟用
`networkProxy` 後，產生的權限設定檔會使用工作區型檔案系統存取：
Codex 受管理網路的強制執行屬於沙箱化網路，因此完整存取設定檔無法保護對外流量。
網域項目使用 `allow` 或 `deny`；Unix 通訊端項目使用 Codex 的
`allow` 或 `none` 值。

### 動態工具呼叫逾時

OpenClaw 所擁有的動態工具呼叫有獨立於
`appServer.requestTimeoutMs` 的限制：Codex `item/tool/call` 請求依預設使用 90
秒的 OpenClaw 看門狗。每次呼叫的正值 `timeoutMs`
引數可延長或縮短該次工具的時間預算，上限為 600000 ms。
當工具呼叫未提供自己的逾時值時，`image_generate` 工具會使用 `agents.defaults.mediaModels.image.timeoutMs`；
否則會使用 120 秒的圖片產生預設值。媒體理解 `image` 工具
會使用所選具圖片處理能力之 `tools.media.models[]` 項目的 `timeoutSeconds`，或其 60 秒媒體預設值；對於
圖片理解，該逾時套用於請求本身，不會因先前的準備工作而縮短。逾時時，OpenClaw
會在支援的情況下中止工具訊號，並向 Codex 傳回失敗的動態工具回應，
讓該回合得以繼續，而不會讓工作階段停留在 `processing`。
此看門狗是外層動態 `item/tool/call` 預算；供應商特定的
請求逾時會在該呼叫內執行，並保留各自的逾時語意。

Codex 接受回合後，以及 OpenClaw 回應回合範圍的
應用程式伺服器請求後，控制框架會預期 Codex 在目前回合取得進展，
並最終以 `turn/completed` 完成原生回合。如果
應用程式伺服器靜默達 `appServer.turnCompletionIdleTimeoutMs`，OpenClaw
會盡力中斷 Codex 回合、記錄診斷逾時，並
釋放 OpenClaw 工作階段通道，避免後續聊天訊息
排在過期的原生回合之後。同一回合的大多數非終止通知
會解除這個短期看門狗，因為 Codex 已證明該回合仍在運作。

工具交接使用較長的工具執行後閒置預算：在 OpenClaw 傳回
`item/tool/call` 回應後、在 `commandExecution`
等原生工具項目完成後、在原始 `custom_tool_call_output`
完成後，以及在工具執行後的原始助理進度、原始推理
完成或推理進度之後。此防護在設定時使用
`appServer.postToolRawAssistantCompletionIdleTimeoutMs`，否則預設為五分鐘；在 Codex 發出
下一個目前回合事件前的靜默合成期間，相同的預算也會延長
進度看門狗。全域應用程式伺服器通知（例如
速率限制更新）不會重設回合閒置進度。推理完成、
評論 `agentMessage` 完成，以及工具執行前的原始推理或
助理進度之後可能接續自動最終回覆，因此它們會使用
進度後回覆防護，而不是立即釋放工作階段通道。

只有最終／非評論且已完成的 `agentMessage` 項目，以及工具執行前的原始
助理完成，會啟用助理輸出釋放：如果 Codex 隨後在沒有
`turn/completed` 的情況下靜默，OpenClaw 會盡力中斷原生
回合並釋放工作階段通道。如果另一個回合監看先贏得該釋放
競爭，只要不再有原生請求、項目或動態工具完成處於作用中，
且助理輸出釋放仍屬於最新完成的項目，並且
之後沒有其他項目完成，OpenClaw 仍會接受已完成的最終助理項目。
這可在已完成工具工作後保留最終答案，而不必重播回合。
部分助理差異更新、較早的過期回覆，以及較晚的空白完成均不符合資格。

可安全重播的 stdio 應用程式伺服器失敗，包括沒有助理、工具、
作用中項目或副作用證據的回合完成閒置逾時，會在新的應用程式伺服器嘗試中重試一次。
不安全的逾時仍會汰除卡住的應用程式伺服器用戶端並釋放
OpenClaw 工作階段通道；它們也會清除過期的原生執行緒繫結，
而不會自動重播。完成監看逾時會顯示 Codex 特定的逾時
文字：可安全重播的情況會表示回應可能不完整，而不安全的
情況會要求使用者在重試前確認目前狀態。公開逾時
診斷包含結構化欄位，例如最後一個應用程式伺服器
通知方法、原始助理回應項目的 ID／類型／角色、作用中
請求／項目數量，以及已啟用的監看狀態；當最後一個通知是
原始助理回應項目時，也會包含有長度限制的助理文字
預覽。其中不會包含原始提示或工具內容。

### 本機測試環境覆寫

- `OPENCLAW_CODEX_APP_SERVER_BIN` 會在
  `appServer.command` 未設定時略過受管理的二進位檔。
- `OPENCLAW_CODEX_APP_SERVER_ARGS`
- `OPENCLAW_CODEX_APP_SERVER_MODE=yolo|guardian`
- `OPENCLAW_CODEX_APP_SERVER_APPROVAL_POLICY`
- `OPENCLAW_CODEX_APP_SERVER_SANDBOX`

`OPENCLAW_CODEX_APP_SERVER_GUARDIAN=1` 已移除。請改用
`plugins.entries.codex.config.appServer.mode: "guardian"`，或使用
`OPENCLAW_CODEX_APP_SERVER_MODE=guardian` 進行單次本機測試。對於可重複的部署，
建議使用設定，因為這可讓外掛行為與 Codex 控制框架的其餘設定
保存在同一個經過審查的檔案中。

## 原生 Codex 外掛

原生 Codex 外掛支援會在與 OpenClaw 控制框架回合相同的 Codex 執行緒中，
使用 Codex 應用程式伺服器本身的應用程式與外掛功能。OpenClaw
不會將 Codex 外掛轉換為合成的 `codex_plugin_*` OpenClaw
動態工具。

`codexPlugins` 僅影響選用原生 Codex 控制框架的工作階段。
它不會影響內建控制框架執行、一般 OpenAI 供應商執行、ACP
對話繫結或其他控制框架。

最小遷移設定：

```json5
{
  plugins: {
    entries: {
      codex: {
        enabled: true,
        config: {
          codexPlugins: {
            enabled: true,
            allow_destructive_actions: true,
            plugins: {
              "google-calendar": {
                enabled: true,
                marketplaceName: "openai-curated",
                pluginName: "google-calendar",
              },
            },
          },
        },
      },
    },
  },
}
```

OpenClaw 建立 Codex 控制框架工作階段或取代過期的 Codex 執行緒繫結時，
會計算執行緒應用程式設定；不會在每個回合重新計算。
變更 `codexPlugins` 後，請使用 `/new`、`/reset`，或重新啟動
閘道，讓未來的 Codex 控制框架工作階段以更新後的應用程式
集合啟動。

如需了解遷移資格、應用程式清單、破壞性動作原則、
資訊徵詢及原生外掛診斷，請參閱
[原生 Codex 外掛](/zh-TW/plugins/codex-native-plugins)。

OpenAI 端的應用程式與外掛存取權由已登入的 Codex
帳號控制；對 Business 與 Enterprise/Edu 工作區而言，亦受工作區應用程式
控制項管理。如需 OpenAI 的帳號與工作區控制概覽，請參閱
[搭配你的 ChatGPT 方案使用 Codex](https://help.openai.com/en/articles/11369540-using-codex-with-your-chatgpt-plan)。

## 電腦操作

電腦操作有專屬的設定指南：
[Codex 電腦操作](/zh-TW/plugins/codex-computer-use)。

簡短說明：OpenClaw 不會內建桌面控制應用程式，也不會自行執行
桌面動作。它會準備 Codex 應用程式伺服器、確認
`computer-use` MCP 伺服器可用，然後讓 Codex 在 Codex 模式回合期間負責原生
MCP 工具呼叫。

## 執行階段邊界

Codex 控制框架只會變更底層嵌入式代理程式執行器。

- 支援 OpenClaw 動態工具。Codex 會要求 OpenClaw 執行
  這些工具，因此 OpenClaw 仍位於執行路徑中。
- Codex 原生 shell、修補、MCP 與原生應用程式工具由 Codex 負責。
  OpenClaw 可透過支援的轉送機制觀察或封鎖特定原生事件，
  但不會改寫原生工具引數。
- Codex 負責原生壓縮。OpenClaw 會保留文字記錄鏡像，用於
  頻道歷史記錄、搜尋、`/new`、`/reset`，以及未來切換模型或控制框架，
  但不會以 OpenClaw 或上下文引擎摘要器取代 Codex 壓縮。
- 媒體產生、媒體理解、TTS、核准與訊息工具
  輸出會繼續使用相符的 OpenClaw 供應商／模型設定。
- `tool_result_persist` 適用於 OpenClaw 所擁有的文字記錄工具結果，
  不適用於 Codex 原生工具結果記錄。

如需了解鉤點層、支援的 V1 介面、原生權限處理、佇列
導向、Codex 意見回饋上傳機制與壓縮詳細資料，請參閱
[Codex 控制框架執行階段](/zh-TW/plugins/codex-harness-runtime)。

## 疑難排解

**Codex 未顯示為一般 `/model` 供應商：**這是新
設定的預期行為。請選取 `openai/gpt-*` 模型、啟用
`plugins.entries.codex.enabled`，並檢查 `plugins.allow` 是否排除
`codex`。

**OpenClaw 使用內建控制框架而非 Codex：**確認有效
路由是完全相符的官方 HTTPS Platform Responses 或 ChatGPT Responses 路由，
沒有自行編寫的請求覆寫，且 Codex 外掛已安裝並
啟用。只有 `openai/gpt-*` 前綴並不足夠。若要在
測試時取得嚴格證明，請設定供應商或模型 `agentRuntime.id: "codex"`；強制使用 Codex 時，
若路由或控制框架不相容，會直接失敗而非回復到備援方案。

**OpenAI Codex 執行階段回復到 API 金鑰路徑：**收集經過遮蔽的
閘道摘錄，其中應顯示模型、執行階段、所選供應商及
失敗資訊。請受影響的協作者在其 OpenClaw 主機上執行以下唯讀命令：

```bash
(
  pattern='openai/gpt-5\.[45]|openai[-]codex|agentRuntime(\.id)?|harnessRuntime|Runtime: OpenAI Codex|legacy OpenAI Codex prefix|resolveSelectedOpenAIRuntimeProvider|candidateProvider[": ]+openai|status[": ]+401|Incorrect API key|No API key|api-key path|API-key path|OAuth'

  if ls /tmp/openclaw/openclaw-*.log >/dev/null 2>&1; then
    grep -E -i -n "$pattern" /tmp/openclaw/openclaw-*.log 2>/dev/null || true
  else
    journalctl --user -u openclaw-gateway --since today --no-pager 2>/dev/null \
      | grep -E -i "$pattern" || true
  fi
) | sed -E \
    -e 's/(Authorization: Bearer )[A-Za-z0-9._~+\/-]+/\1[REDACTED]/Ig' \
    -e 's/(Bearer )[A-Za-z0-9._~+\/-]+/\1[REDACTED]/Ig' \
    -e 's/(api[_ -]?key[=: ]+)[^ ,}"]+/\1[REDACTED]/Ig' \
    -e 's/(OPENAI_API_KEY[=: ]+)[^ ,}"]+/\1[REDACTED]/Ig' \
    -e 's/sk-[A-Za-z0-9_-]{12,}/sk-[REDACTED]/g' \
    -e 's/[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Za-z]{2,}/[EMAIL-REDACTED]/g' \
  | tail -200
```

有用的摘錄通常包含 `openai/gpt-5.6-sol` 或 `openai/gpt-5.6-luna`、
`Runtime: OpenAI Codex`、`agentRuntime.id` 或 `harnessRuntime`、
`candidateProvider: "openai"`，以及 `401`、`Incorrect API key` 或
`No API key` 結果。修正後的執行應顯示 OpenAI OAuth 路徑，
而不是一般的 OpenAI API 金鑰失敗。

**舊版 Codex 模型參照設定仍然存在：**執行 `openclaw doctor --fix`。
Doctor 會將舊版模型參照改寫為 `openai/*`、移除過時的工作階段與
整個代理程式的執行階段固定設定，並保留現有的驗證設定檔覆寫。

**app-server 遭到拒絕：**請使用 `0.143.0` 中的穩定版 Codex app-server，
並透過隨附的 `0.145.0` 執行。預發行版、含建置後綴的版本，以及較新但
尚未驗證的版本都會遭到拒絕，因為 OpenClaw 會根據隨附的 app-server 版本
驗證產生的結構描述。

**`/codex status` 無法連線：**檢查 `codex` 外掛
是否已啟用、設定允許清單時 `plugins.allow` 是否包含該外掛，以及任何自訂的
`appServer.command`、`url`、`authToken` 或標頭是否有效。

**Codex app-server 使用過多記憶體：**請先區分這兩個處理程序。
OpenClaw 會將本機 Codex app-server 作為獨立的 Rust 子處理程序執行。
`NODE_OPTIONS=--max-old-space-size=...` 只會變更閘道的 Node.js V8
堆積；它不會限制或擴大 Codex。受管理的閘道安裝已會選擇
自適應 V8 堆積，提高此值可能會減少 Codex 可用的主機記憶體。若是閘道壓力，請參閱
[閘道記憶體疑難排解](/zh-TW/gateway/troubleshooting#gateway-exits-during-high-memory-use)；
若是 Codex 子處理程序，請檢查主機或容器記憶體。

隨附的 Codex 沒有堆積或 RSS 限制，也沒有可設定的閒置卸載
延遲。最後一個用戶端取消訂閱後，非作用中的執行緒仍可能維持載入
最長 30 分鐘。在資源受限的主機上，增加閘道堆積前，請先減少原生 Codex 子代理程式的扇出：

```json5
{
  plugins: {
    entries: {
      codex: {
        config: {
          appServer: {
            args: ["-c", "agents.max_threads=3", "app-server", "--listen", "stdio://"],
          },
        },
      },
    },
  },
}
```

該設定會限制隨附 Codex 預設
多代理程式後端的原生子執行緒。如果明確啟用 Codex 多代理程式 v2，請改用
`features.multi_agent_v2.max_concurrent_threads_per_session=3`；v2
限制包含根執行緒，且無法與 `agents.max_threads` 結合使用。
若要為 Codex 提供更多餘裕，請提高主機、容器或 cgroup 的記憶體
配置。作業系統的硬性限制可能會終止 Codex，而非對其施加背壓。

**模型探索速度緩慢：**請降低
`plugins.entries.codex.config.discovery.timeoutMs` 或停用探索。
請參閱 [Codex 控制框架參考](/zh-TW/plugins/codex-harness-reference#model-discovery)。

**WebSocket 傳輸立即失敗：**檢查 `appServer.url`、
`authToken`、標頭，以及遠端 app-server 使用的 Codex
app-server 通訊協定版本是否相同。Codex WebSocket 傳輸仍屬實驗性功能，
且不受支援；請優先使用受管理的 stdio 或本機 Unix 控制通訊端。

**原生 shell 或修補工具遭 `Native hook relay
unavailable` 封鎖：**Codex 執行緒仍在嘗試使用
OpenClaw 已不再註冊的原生掛鉤轉送
ID。這是原生 Codex 掛鉤
傳輸問題，並非 ACP 後端、供應商、GitHub 或 shell 命令
失敗。請在受影響的聊天中使用 `/new` 或 `/reset` 啟動新的工作階段，
然後重試無害的命令。如果這次有效，但下一次原生工具
呼叫又失敗，請僅將 `/new` 視為暫時因應措施：重新啟動 Codex app-server 或
OpenClaw 閘道後，將提示詞複製到新的工作階段，讓舊執行緒被捨棄，並重新建立
原生掛鉤註冊。

**Codex 工具呼叫建立過多短生命週期的掛鉤處理程序：**請設定
`plugins.entries.codex.config.appServer.loopDetectionPreToolUseRelay: false`
並重新啟動閘道。這只會停用用於 OpenClaw 迴圈偵測及其無原則標記的 Codex
`PreToolUse` 子處理程序。必要的
`before_tool_call` 與受信任工具原則轉送仍會保持啟用。

**非 Codex 模型使用內建控制框架：**除非供應商或模型執行階段原則將其路由至其他
控制框架，否則這是預期行為。在 `auto` 模式下，單純的非 OpenAI
供應商參照會繼續使用其一般供應商路徑。

**Computer Use 已安裝，但工具未執行：**請在新的工作階段中檢查
`/codex computer-use status`。如果工具回報
`Native hook relay unavailable`，請使用上述原生掛鉤轉送復原方式。
請參閱 [Codex Computer Use](/zh-TW/plugins/codex-computer-use#troubleshooting)。

## 相關內容

- [Codex 控制框架參考](/zh-TW/plugins/codex-harness-reference)
- [Codex 控制框架執行階段](/zh-TW/plugins/codex-harness-runtime)
- [Codex 監督](/zh-TW/plugins/codex-supervision)
- [原生 Codex 外掛](/zh-TW/plugins/codex-native-plugins)
- [Codex Computer Use](/zh-TW/plugins/codex-computer-use)
- [代理程式執行階段](/zh-TW/concepts/agent-runtimes)
- [模型供應商](/zh-TW/concepts/model-providers)
- [OpenAI 供應商](/zh-TW/providers/openai)
- [OpenAI Codex 說明](https://help.openai.com/en/collections/14937394-codex)
- [代理程式控制框架外掛](/zh-TW/plugins/sdk-agent-harness)
- [外掛掛鉤](/zh-TW/plugins/hooks)
- [診斷匯出](/zh-TW/gateway/diagnostics)
- [狀態](/zh-TW/cli/status)
- [測試](/zh-TW/help/testing-live#live-codex-app-server-harness-smoke)
