---
read_when:
    - 選擇或切換模型、設定別名
    - 偵錯模型容錯移轉／“所有模型皆失敗”
    - 瞭解驗證設定檔及其管理方式
sidebarTitle: Models FAQ
summary: 常見問題：模型預設值、選擇、別名、切換、容錯移轉與驗證設定檔
title: 常見問題：模型與驗證
x-i18n:
    generated_at: "2026-07-26T08:27:55Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 0c46d99352c5e51af5917c6f62b897dfa4030cb0201b8235e28f2f81f2954544
    source_path: help/faq-models.md
    workflow: 16
---

模型與驗證設定檔問答。如需設定、工作階段、閘道、頻道及
疑難排解資訊，請參閱主要的[常見問題](/zh-TW/help/faq)。

## 模型：預設值、選擇、別名、切換

<AccordionGroup>
  <Accordion title='什麼是「預設模型」？'>
    設定方式：

    ```text
    agents.defaults.model.primary
    ```

    模型是 `provider/model` 參照（例如：`openai/gpt-5.5`、
    `anthropic/claude-sonnet-4-6`）。務必明確設定 `provider/model`。如果
    省略提供者，OpenClaw 會先嘗試比對別名，再針對該模型 ID 比對唯一的
    已設定提供者，接著退回使用已設定的預設提供者（已淘汰的相容性路徑）。
    如果該提供者已不再擁有已設定的預設模型，OpenClaw 會改用第一個已設定的
    提供者／模型，而不是過時的預設值。

  </Accordion>

  <Accordion title="建議使用哪個模型？">
    使用你的提供者堆疊所提供、最新世代中能力最強的模型，
    尤其是用於啟用工具或處理不受信任輸入的代理程式時——較弱或
    過度量化的模型更容易受到提示詞注入及不安全
    行為影響（請參閱[安全性](/zh-TW/gateway/security)）。依代理程式角色，將較便宜的模型
    分配給例行／低風險的聊天。

    依代理程式分配模型，並使用子代理程式平行處理耗時任務（每個
    子代理程式各自消耗權杖）。請參閱[模型](/zh-TW/concepts/models)、
    [子代理程式](/zh-TW/tools/subagents)、[MiniMax](/zh-TW/providers/minimax)及
    [本機模型](/zh-TW/gateway/local-models)。

  </Accordion>

  <Accordion title="如何在不清除設定的情況下切換模型？">
    只變更模型欄位——避免完整取代設定。

    - `/model`：在聊天中使用（每個工作階段，請參閱[斜線命令](/zh-TW/tools/slash-commands)）
    - `openclaw models set ...`（僅更新模型設定）
    - `openclaw configure --section model`（互動式）
    - 直接編輯 `~/.openclaw/openclaw.json` 中的 `agents.defaults.model`

    進行 RPC 編輯時，先使用 `config.schema.lookup` 檢查（正規化
    路徑、簡要結構描述文件、子項摘要），然後針對部分物件，優先使用 `config.patch`
    而不是 `config.apply`。如果確實覆寫了設定，
    請從備份還原，或執行 `openclaw doctor` 進行修復。

    文件：[模型](/zh-TW/concepts/models)、[設定](/zh-TW/cli/configure)、
    [組態](/zh-TW/cli/config)、[診斷](/zh-TW/gateway/doctor)。

  </Accordion>

  <Accordion title="可以使用自行託管的模型（llama.cpp、vLLM、Ollama）嗎？">
    可以——Ollama 是最簡單的做法。快速設定：

    1. 從 `https://ollama.com/download` 安裝 Ollama
    2. 拉取本機模型，例如 `ollama pull gemma4`
    3. 如也要使用雲端模型，請執行 `ollama signin`
    4. 執行 `openclaw onboard`，選擇 `Ollama`，然後選擇 `Local` 或 `Cloud + Local`

    `Cloud + Local` 可讓你同時使用雲端模型及本機 Ollama 模型；
    `kimi-k2.5:cloud` 等雲端模型不需要在本機拉取。若要手動切換：
    `openclaw models list`，然後 `openclaw models set ollama/<model>`。

    較小型／高度量化的模型更容易受到提示詞注入攻擊。
    任何可存取工具的機器人都應使用大型模型；若仍要使用小型模型，
    請啟用沙箱隔離及嚴格的工具允許清單。

    文件：[Ollama](/zh-TW/providers/ollama)、[本機模型](/zh-TW/gateway/local-models)、
    [模型提供者](/zh-TW/concepts/model-providers)、[安全性](/zh-TW/gateway/security)、
    [沙箱隔離](/zh-TW/gateway/sandboxing)。

  </Accordion>

  <Accordion title="如何即時切換模型（無須重新啟動）？">
    將 `/model <name>` 作為獨立訊息傳送。請參閱
    [斜線命令](/zh-TW/tools/slash-commands)以取得
    完整命令清單，包括編號選擇器（`/model`、`/model
    list`、`/model 3`）、用於清除工作階段覆寫值的 `/model default`，以及
    用於查看端點／API 模式詳細資料的 `/model status`。

    使用 `@profile` 為每個工作階段強制指定驗證設定檔：

    ```text
    /model opus@anthropic:default
    /model opus@anthropic:work
    ```

    若要取消固定透過 `@profile` 設定的設定檔，請重新執行不含
    後綴的 `/model`（例如 `/model anthropic/claude-opus-4-6`），或從
    `/model` 選擇預設值。使用 `/model status` 確認使用中的驗證設定檔。

  </Accordion>

  <Accordion title="如果兩個提供者公開相同的模型 ID，/model 會使用哪一個？">
    `/model provider/model` 會選擇該確切的提供者路由。例如，
    即使模型 ID 相同，`qianfan/deepseek-v4-flash` 與 `deepseek/deepseek-v4-flash` 仍是不同的
    參照——OpenClaw 不會僅因未限定的 ID 相符就悄悄切換
    提供者。

    使用者選取的 `/model` 參照會嚴格套用容錯移轉規則：如果該
    提供者／模型無法使用，回覆會明確失敗，而不會
    容錯移轉至 `agents.defaults.model.fallbacks`。已設定的容錯移轉
    鏈仍會套用至已設定的預設值、排程工作的主要模型，以及
    自動選取的容錯移轉狀態。當未覆寫工作階段的執行可
    使用容錯移轉時，OpenClaw 會先嘗試要求的提供者／模型，接著
    嘗試已設定的容錯移轉，最後才嘗試已設定的主要模型——因此重複的未限定
    模型 ID 絕不會直接跳回預設提供者。

    請參閱[模型](/zh-TW/concepts/models)及[模型容錯移轉](/zh-TW/concepts/model-failover)。

  </Accordion>

  <Accordion title="可以使用 GPT 5.5 處理日常工作，並使用 Codex 5.5 編寫程式碼嗎？">
    可以——模型選擇與執行階段選擇互相獨立：

    - **原生 Codex 程式設計代理程式：**將 `agents.defaults.model.primary` 設為
      `openai/gpt-5.5`。使用 `openclaw models auth login --provider
      openai` 登入，以進行 ChatGPT／Codex 訂閱驗證。
    - **代理程式迴圈以外的直接 OpenAI API 工作：**為影像、
      嵌入、語音、即時處理及其他非代理程式 OpenAI API 介面設定
      `OPENAI_API_KEY`。
    - **OpenAI 代理程式 API 金鑰驗證：**使用 `/model openai/gpt-5.5` 搭配依序排列的
      `openai` API 金鑰設定檔。
    - **子代理程式：**將程式設計工作分配給專注於 Codex 的代理程式，並為其設定
      專屬的 `openai/gpt-5.5` 模型。

    請參閱[模型](/zh-TW/concepts/models)及[斜線命令](/zh-TW/tools/slash-commands)。

  </Accordion>

  <Accordion title="如何為 GPT 5.5 設定快速模式？">
    - **每個工作階段：**使用 `openai/gpt-5.5` 時傳送 `/fast on`。
    - **每個模型的預設值：**將
      `agents.defaults.models["openai/gpt-5.5"].params.fastMode` 設為 `true`。
    - **自動截止：**`/fast auto` 或 `params.fastMode: "auto"` 會讓新的
      模型呼叫在截止時間前以快速模式執行，之後的重試、容錯移轉、
      工具結果或接續呼叫則不使用快速模式。截止時間預設為
      60 秒；可在模型上使用 `params.fastAutoOnSeconds` 覆寫。

    ```json5
    {
      agents: {
        defaults: {
          models: {
            "openai/gpt-5.5": {
              params: {
                fastMode: "auto",
                fastAutoOnSeconds: 30,
              },
            },
          },
        },
      },
    }
    ```

    快速模式會對應至原生 OpenAI Responses
    要求中的 `service_tier = "priority"`；現有的 `service_tier` 值會保留，且快速模式不會
    重寫 `reasoning` 或 `text.verbosity`。工作階段的 `/fast` 覆寫值優先於
    設定預設值。

    請參閱[思考與快速模式](/zh-TW/tools/thinking)，以及
    [OpenAI](/zh-TW/providers/openai) 提供者頁面「進階設定」下的快速模式章節。

  </Accordion>

  <Accordion title='為什麼會看到「Model ... is not allowed」，接著沒有回覆？'>
    如果 `agents.defaults.modelPolicy.allow` 非空白，它會成為
    `/model`、工作階段覆寫及 `--model` 的
    **允許清單**。選擇該清單以外的模型時，會傳回以下內容，而不是正常回覆：

    ```text
    Model override "provider/model" is not allowed by agents.defaults.modelPolicy.allow.
    ```

    修正方式：將確切模型或 `"provider/*"` 等提供者萬用字元加入
    指定的 `modelPolicy.allow` 清單、移除／清空該清單，或從
    `/model list` 選擇模型。如果命令也
    包含 `--runtime codex`，請先更新允許清單，再重試相同的
    `/model provider/model --runtime codex` 命令。

  </Accordion>

  <Accordion title='為什麼會看到「Unknown model: minimax/MiniMax-M3」？'>
    如果使用的是較舊的 OpenClaw 版本，請先升級（或從原始碼執行
    `main`）並重新啟動閘道——你的已安裝版本目錄中可能尚未包含
    `MiniMax-M3`。否則表示 MiniMax 提供者尚未
    設定（找不到提供者項目或驗證設定檔），因此無法解析模型。
    請參閱 [MiniMax](/zh-TW/providers/minimax) 提供者頁面的疑難排解章節，
    以取得完整的修正檢查清單、提供者／模型 ID 表格及設定區塊範例。

  </Accordion>

  <Accordion title="可以將 MiniMax 設為預設模型，並使用 OpenAI 處理複雜工作嗎？">
    可以。將 MiniMax 設為預設值，並依工作階段切換模型——容錯移轉
    是用於錯誤，而非「困難工作」，因此請使用 `/model` 或另一個代理程式。

    **選項 A：依工作階段切換**

    ```json5
    {
      env: { MINIMAX_API_KEY: "sk-...", OPENAI_API_KEY: "sk-..." },
      agents: {
        defaults: {
          model: { primary: "minimax/MiniMax-M3" },
          models: {
            "minimax/MiniMax-M3": { alias: "minimax" },
            "openai/gpt-5.5": { alias: "gpt" },
          },
        },
      },
    }
    ```

    接著使用 `/model gpt`。

    **選項 B：使用不同的代理程式**——代理程式 A 預設使用 MiniMax，代理程式 B
    預設使用 OpenAI；依代理程式進行路由，或使用 `/agent` 切換。

    文件：[模型](/zh-TW/concepts/models)、[多代理程式路由](/zh-TW/concepts/multi-agent)、
    [MiniMax](/zh-TW/providers/minimax)、[OpenAI](/zh-TW/providers/openai)。

  </Accordion>

  <Accordion title="opus／sonnet／gpt 是內建捷徑嗎？">
    是——它們是內建簡寫，且只會在 `agents.defaults.models` 中存在目標模型時套用：

    | 別名 | 解析為 |
    | --- | --- |
    | `opus` | `anthropic/claude-opus-5` |
    | `sonnet` | `anthropic/claude-sonnet-5` |
    | `gpt` | `openai/gpt-5.4` |
    | `gpt-mini` | `openai/gpt-5.4-mini` |
    | `gpt-nano` | `openai/gpt-5.4-nano` |
    | `gemini` | `google/gemini-3.1-pro-preview` |
    | `gemini-flash` | `google/gemini-3-flash-preview` |
    | `gemini-flash-lite` | `google/gemini-3.1-flash-lite` |

    同名的自訂別名會覆寫內建別名。

  </Accordion>

  <Accordion title="如何定義／覆寫模型捷徑（別名）？">
    別名位於 `agents.defaults.models.<modelId>.alias`：

    ```json5
    {
      agents: {
        defaults: {
          model: { primary: "anthropic/claude-opus-4-6" },
          models: {
            "anthropic/claude-opus-4-6": { alias: "opus" },
            "anthropic/claude-sonnet-4-6": { alias: "sonnet" },
          },
        },
      },
    }
    ```

    接著，`/model sonnet`（或在支援時使用 `/<alias>`）會解析為該
    模型 ID。

  </Accordion>

  <Accordion title="如何新增 OpenRouter 或 Z.AI 等其他提供者的模型？">
    OpenRouter（按權杖計費；提供多種模型）：

    ```json5
    {
      agents: {
        defaults: {
          model: { primary: "openrouter/anthropic/claude-sonnet-4-6" },
          models: { "openrouter/anthropic/claude-sonnet-4-6": {} },
        },
      },
      env: { OPENROUTER_API_KEY: "sk-or-..." },
    }
    ```

    Z.AI（GLM 模型）：

    ```json5
    {
      agents: {
        defaults: {
          model: { primary: "zai/glm-5.1" },
          models: { "zai/glm-5.1": {} },
        },
      },
      env: { ZAI_API_KEY: "..." },
    }
    ```

    如果參照的提供者／模型缺少提供者金鑰，執行階段會引發
    驗證錯誤（例如 `No API key found for provider "zai"`）。

    **新增代理程式後找不到 API 金鑰**

    新代理程式的驗證儲存區為空——驗證是每個代理程式各自獨立的，儲存位置為：

    ```text
    ~/.openclaw/agents/<agentId>/agent/auth-profiles.json
    ```

    修正：執行 `openclaw agents add <id>` 並在精靈中設定驗證，或
    僅從主要代理程式的儲存區複製可攜式靜態
    `api_key`/`token` 設定檔。若使用 OAuth，當新代理程式需要
    自己的帳戶時，請從該代理程式登入。完整的 `agentDir` 重複使用與認證資訊
    共用規則請參閱[多代理程式路由](/zh-TW/concepts/multi-agent)——切勿在代理程式之間重複使用
    `agentDir`。

  </Accordion>
</AccordionGroup>

## 模型容錯移轉與「所有模型皆失敗」

<AccordionGroup>
  <Accordion title="容錯移轉如何運作？">
    分為兩個階段：

    1. 在同一供應商內進行**驗證設定檔輪替**。
    2. **模型備援**至 `agents.defaults.model.fallbacks` 中的下一個模型。

    系統會對失敗的設定檔套用冷卻期（指數退避），因此當供應商受到速率限制或暫時失敗時，
    OpenClaw 仍能持續回應。

    速率限制類別涵蓋的不只是單純的 `429`：`Too many concurrent
    requests`、`ThrottlingException`、`concurrency limit reached`、`workers_ai
    ... quota limit exceeded`、`resource exhausted`，以及週期性
    使用量時窗限制（`weekly/monthly limit reached`）全都視為
    值得進行容錯移轉的速率限制。

    計費回應不一定都是 `402`，而且部分 `402` 會留在
    暫時性／速率限制類別，而非計費處理路徑。`401`/`403` 上明確的
    計費文字仍可路由至計費處理；供應商特定的文字比對器（例如 OpenRouter
    `Key limit exceeded`）仍僅限用於其所屬供應商。若 `402` 看起來像是可重試的
    使用量時窗或組織／工作區支出限制（`daily limit reached, resets tomorrow`、
    `organization spending limit exceeded`），則會視為 `rate_limit`，而不是
    長時間停用計費功能。

    上下文溢位錯誤完全不會進入備援路徑——像是 `request_too_large`、`input exceeds the maximum number of tokens`、
    `input token count exceeds the maximum number of input tokens`、`input is
    too long for the model` 或 `ollama error: context length exceeded` 等特徵，
    會進入壓縮／重試流程，而非繼續進行模型備援。

    一般伺服器錯誤文字的範圍，比「任何包含 unknown/error
    的文字」更窄。以下受供應商範圍限制的暫時性形式確實會視為容錯移轉
    訊號：Anthropic 的純 `An unknown error occurred`、OpenRouter 的純
    `Provider returned error`、像 `Unhandled stop reason:
    error` 這類停止原因錯誤、含有暫時性伺服器文字
    （`internal
    server error`、`unknown error, 520`、`upstream error`、`backend error`）的 JSON
    `api_error` 承載資料，以及在供應商
    上下文相符時，像 `ModelNotReadyException` 這類供應商忙碌錯誤。像 `LLM request failed
    with an unknown error.`
    這類一般內部備援文字仍採保守處理，單獨出現時不會觸發備援。

  </Accordion>

  <Accordion title='「找不到設定檔 anthropic:default 的認證資訊」代表什麼？'>
    驗證設定檔 ID `anthropic:default` 在預期的驗證儲存區中沒有認證資訊。

    **修正檢查清單：**

    - 確認設定檔的儲存位置——目前位置：
      `~/.openclaw/agents/<agentId>/agent/auth-profiles.json`；舊版位置：
      `~/.openclaw/agent/*`（由 `openclaw doctor` 遷移）。
    - 確認閘道會載入你的環境變數。僅在
      你的 shell 中設定的 `ANTHROPIC_API_KEY`，不會傳遞至透過 systemd/launchd 執行的閘道——請將其放入
      `~/.openclaw/.env`，或啟用 `env.shellEnv`。
    - 確認你正在編輯正確的代理程式——多代理程式設定會有
      多個 `auth-profiles.json` 檔案。
    - 執行 `openclaw models status`，查看已設定的模型與供應商
      驗證狀態。

    **若為「找不到設定檔 anthropic 的認證資訊」（沒有電子郵件尾碼）：**

    此次執行已固定使用閘道找不到的 Anthropic 設定檔。

    - 使用 Claude 命令列介面：在閘道主機上執行 `openclaw models auth login --provider anthropic
      --method cli --set-default`。
    - 若偏好使用 API 金鑰：請在閘道主機上將 `ANTHROPIC_API_KEY` 放入
      `~/.openclaw/.env`，然後清除任何強制使用遺失設定檔的固定順序：

      ```bash
      openclaw models auth order clear --provider anthropic
      ```

    - 遠端模式：驗證設定檔位於閘道機器，而非你的
      筆記型電腦——請確認你是在該機器上執行命令。

  </Accordion>

  <Accordion title="為什麼它也嘗試了 Google Gemini 並失敗？">
    如果你的模型設定包含 Google Gemini 作為備援（或你
    切換至 Gemini 簡寫），OpenClaw 會在備援期間嘗試使用它。未設定
    Google 認證資訊時會產生 `No API key found for provider
    "google"`。修正方式：新增 Google 驗證，或從
    `agents.defaults.model.fallbacks`/別名中移除 Google 模型。

    **LLM 要求遭拒：需要思考簽章（Google Antigravity）**

    原因：工作階段歷程包含沒有簽章的思考區塊（通常
    源自中止／不完整的串流）；Google Antigravity 要求思考區塊
    必須具有簽章。OpenClaw 會為 Google Antigravity Claude 移除未簽章的思考區塊；若問題仍然出現，請啟動新的工作階段，或為該代理程式設定
    `/thinking off`。

  </Accordion>
</AccordionGroup>

## 驗證設定檔：其定義與管理方式

相關內容：[/concepts/oauth](/zh-TW/concepts/oauth)（OAuth 流程、權杖儲存、多帳戶模式）

<AccordionGroup>
  <Accordion title="什麼是驗證設定檔？">
    與供應商繫結的具名認證資訊記錄（OAuth 或 API 金鑰），儲存於：

    ```text
    ~/.openclaw/agents/<agentId>/agent/auth-profiles.json
    ```

    在不輸出秘密的情況下檢查已儲存的設定檔：`openclaw models auth
    list`（可選用 `--provider <id>` 或 `--json`）。請參閱
    [模型命令列介面](/zh-TW/cli/models#auth-profiles)。

  </Accordion>

  <Accordion title="常見的設定檔 ID 有哪些？">
    以供應商為前綴：`anthropic:default`（沒有電子郵件身分時很常見）、
    OAuth 身分使用 `anthropic:<email>`，或使用你自行選擇的自訂 ID
    （例如 `anthropic:work`）。

  </Accordion>

  <Accordion title="我可以控制優先嘗試哪個驗證設定檔嗎？">
    可以。`auth.order.<provider>` 設定可指定各供應商的輪替順序
    （僅限中繼資料——不會儲存秘密）。

    OpenClaw 可能會略過處於短期**冷卻期**（速率限制、
    逾時、驗證失敗）或較長時間**停用**狀態
    （計費／額度不足）的設定檔。使用 `openclaw models status
    --json` 檢查，並查看 `auth.unusableProfiles`。速率限制冷卻期可以
    僅限特定模型——某個設定檔對一個模型處於冷卻期時，仍可為同一供應商的
    同系列模型提供服務；計費／停用時窗則會封鎖整個
    設定檔。

    設定各代理程式的順序覆寫（儲存於該代理程式的 `auth-state.json`）：

    ```bash
    # 預設使用已設定的預設代理程式（省略 --agent）
    openclaw models auth order get --provider anthropic

    # 將輪替鎖定至單一設定檔
    openclaw models auth order set --provider anthropic anthropic:default

    # 或設定明確順序（同一供應商內的備援）
    openclaw models auth order set --provider anthropic anthropic:work anthropic:default

    # 清除覆寫（回復使用設定中的 auth.order／循環輪替）
    openclaw models auth order clear --provider anthropic

    # 指定特定代理程式
    openclaw models auth order set --provider anthropic --agent main anthropic:default
    ```

    驗證實際會嘗試的項目：`openclaw models status --probe`。若已儲存的設定檔
    未包含在明確順序中，系統會回報
    `excluded_by_auth_order`，而不會在未告知的情況下嘗試。

  </Accordion>

  <Accordion title="OAuth 與 API 金鑰有何不同？">
    - 供應商支援時，**OAuth／命令列介面登入**通常使用訂閱存取。對 Anthropic 而言，OpenClaw 的 Claude 命令列介面後端
      使用 Claude Code `claude -p`，Anthropic 目前將其視為
      Agent SDK／程式化使用方式，並計入訂閱用量限制——目前的計費暫停
      狀態與來源連結請參閱 [Anthropic](/zh-TW/providers/anthropic)。
    - **API 金鑰**採用依權杖計費。

    精靈支援 Anthropic Claude 命令列介面、OpenAI Codex OAuth 與 API
    金鑰。

  </Accordion>
</AccordionGroup>

## 相關內容

- [常見問題](/zh-TW/help/faq)——主要的常見問題
- [常見問題——快速開始與首次執行設定](/zh-TW/help/faq-first-run)
- [模型選擇](/zh-TW/concepts/model-providers)
- [模型容錯移轉](/zh-TW/concepts/model-failover)
