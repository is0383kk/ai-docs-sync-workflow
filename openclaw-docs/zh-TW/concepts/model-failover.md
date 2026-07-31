---
read_when:
    - 診斷認證設定檔輪替、冷卻時間或模型備援行為
    - 更新認證設定檔或模型的容錯移轉規則
    - 瞭解工作階段模型覆寫如何與備援重試互動
sidebarTitle: Model failover
summary: OpenClaw 如何輪替驗證設定檔並在模型之間進行備援切換
title: 模型容錯移轉
x-i18n:
    generated_at: "2026-07-26T07:40:06Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 3dfedbc85038eebb5be056a7b3ffa3275b4329a0b0d791e1a2b4701cbaa4b595
    source_path: concepts/model-failover.md
    workflow: 16
---

OpenClaw 分兩個階段處理失敗：

1. 目前供應商內的**認證設定檔輪替**。
2. **模型後援**至 `agents.defaults.model.fallbacks` 中的下一個模型。

## 執行階段流程

<Steps>
  <Step title="解析工作階段狀態">
    解析使用中的工作階段模型與認證設定檔偏好。
  </Step>
  <Step title="建立候選鏈">
    根據目前的模型選擇，以及該選擇來源的後援原則，建立模型候選鏈。已設定的預設值、排程工作的主要模型，以及自動選取的後援模型可以使用已設定的後援；使用者明確選取的工作階段模型則採嚴格模式。
  </Step>
  <Step title="嘗試目前的供應商">
    依照認證設定檔輪替／冷卻規則嘗試目前的供應商。
  </Step>
  <Step title="發生值得容錯移轉的錯誤時繼續">
    如果該供應商因值得容錯移轉的錯誤而用盡可用選項，則移至下一個模型候選項目。
  </Step>
  <Step title="在目前回合使用後援">
    執行勝出的後援候選項目，而不變更工作階段選取的供應商／模型。
  </Step>
  <Step title="安全重試單純的過載耗盡">
    如果每個候選項目都只因供應商過載而失敗，且尚未開始執行工具或輸出助理內容，則以指數退避方式重試完整的回合區域鏈，最多 10 次。30 秒後傳送一次狀態通知，避免使用者在沒有任何訊息的情況下持續等待。
  </Step>
  <Step title="若全部耗盡則擲回 FallbackSummaryError">
    如果每個候選項目都失敗，則擲回 `FallbackSummaryError`，其中包含各次嘗試的詳細資料，以及已知情況下最早的冷卻到期時間。
  </Step>
</Steps>

後援執行僅適用於目前回合。回覆執行器只會保存後援通知狀態，讓 `/status` 與轉換通知能區分選取的模型和實際回答的模型；它不會將後援保存為下一回合的模型選擇。

## 選擇來源原則

選擇來源會控制是否允許後援鏈：

- **已設定的預設值**：`agents.defaults.model.primary` 使用 `agents.defaults.model.fallbacks`。
- **代理程式主要模型**：`agents.entries.*.model` 採嚴格模式，除非該代理程式的模型物件包含自己的 `fallbacks`。使用 `fallbacks: []` 可明確指定嚴格行為，或使用非空白清單，讓該代理程式選擇加入模型後援。
- **執行階段後援**：後援候選項目只適用於目前回合。下一回合會再次從選取的主要模型開始。OpenClaw 仍會辨識先前儲存的 `modelOverrideSource: "auto"` 項目，每 5 分鐘探測其設定的來源，並在來源恢復後清除這些項目。`/new`、`/reset` 和 `sessions.reset` 也會清除這些項目。
- **使用者工作階段覆寫**：`/model`、模型選擇器、`session_status(model=...)` 和 `sessions.patch` 會寫入 `modelOverrideSource: "user"`。這是精確的工作階段選擇。如果選取的供應商／模型在產生回覆前失敗，OpenClaw 會回報失敗，而不會改用不相關的已設定後援來回答。
- **舊版工作階段覆寫**：較舊的工作階段項目可能有 `modelOverride`，但沒有 `modelOverrideSource`。OpenClaw 會將這些項目視為使用者覆寫，避免明確的舊選擇在未告知的情況下轉換成後援行為。
- **排程承載資料模型**：排程工作的 `payload.model`／`--model` 是工作的主要模型，而非使用者工作階段覆寫。除非工作提供 `payload.fallbacks`，否則它會使用已設定的後援；`payload.fallbacks: []` 會讓排程執行採嚴格模式。

當某個回合轉移至後援時，OpenClaw 會傳送可見通知；當後續回合在選取的主要模型上成功時，也會傳送另一則通知。保存的通知狀態可避免連續回合使用相同的選取／使用中配對時重複通知，而模型選擇本身維持不變。

## 認證失敗略過快取

依預設，每個新回合都會保留現有的後援重試行為：OpenClaw 會再次嘗試每個已設定的後援候選項目，包括最近因 `auth` 或 `auth_permanent` 而失敗的非主要候選項目。

若要選擇啟用並抑制重複的認證失敗，請使用：

```bash
OPENCLAW_FALLBACK_SKIP_TTL_MS=60000
```

啟用後，當非主要後援候選項目發生認證類別的失敗時，OpenClaw 會記錄一個位於記憶體中、以工作階段為範圍的略過標記，其索引鍵由工作階段 ID、供應商和模型組成。主要候選項目絕不會被略過，因此明確的使用者模型選擇仍會顯示真正的認證錯誤。此快取僅限處理程序本機，並會在閘道重新啟動時清除。

此值是以毫秒為單位的 TTL。`0` 或未設定時會停用快取。正值會限制在 1 秒到 10 分鐘之間。

## 使用者可見的後援通知

當工作階段移至自動選取的後援時，OpenClaw 會在相同的回覆介面中傳送狀態通知：

```text
↪️ 模型後援：<fallback>（已選取 <primary>；<reason>）
```

當後續探測成功，且工作階段返回選取的主要模型時，OpenClaw 會傳送：

```text
↪️ 已清除模型後援：<primary>（先前為 <fallback>）
```

這些通知是操作訊息，而非助理內容。每次狀態變更只會傳送一次，包括在可行時僅有副作用的回合，但重複的回合區域後援轉換不會重複傳送通知。傳遞會略過一般的來源回覆抑制，不會占用串接式頻道的第一個助理回覆位置，且不會納入文字轉語音和承諾擷取。

## 認證儲存空間（金鑰 + OAuth）

OpenClaw 對 API 金鑰和 OAuth 權杖都使用**認證設定檔**。

- 密鑰和執行階段認證路由狀態位於 `~/.openclaw/agents/<agentId>/agent/openclaw-agent.sqlite`。
- 設定 `auth.profiles`／`auth.order` **僅包含中繼資料與路由**（不含密鑰）。
- 僅供舊版匯入的 OAuth 檔案：`~/.openclaw/credentials/oauth.json`（首次使用時匯入每個代理程式的認證儲存區）。
- 舊版 `auth-profiles.json`、`auth-state.json` 和每個代理程式的 `auth.json` 檔案會由 `openclaw doctor --fix` 匯入。

更多詳細資訊：[OAuth](/zh-TW/concepts/oauth)

認證資訊類型：

- `type: "api_key"` → `{ provider, key }`
- `type: "oauth"` → `{ provider, access, refresh, expires, email? }`（部分供應商另有 `projectId`/`enterpriseUrl`）
- `type: "token"` → 靜態持有人式權杖，可選擇設定到期時間；OpenClaw 不會重新整理它（用於 `aws-sdk` 和其他認證資訊鏈認證模式）

## 設定檔 ID

OAuth 登入會建立不同的設定檔，讓多個帳號可以共存。

- 預設：沒有可用的電子郵件地址時使用 `provider:default`。
- 具有電子郵件地址的 OAuth：`provider:<email>`（例如 `google-antigravity:user@gmail.com`）。

設定檔位於每個代理程式的 `openclaw-agent.sqlite` 認證設定檔儲存區中。

## 輪替順序

當供應商有多個設定檔時，OpenClaw 會依下列方式選擇順序：

<Steps>
  <Step title="明確設定">
    `auth.order[provider]`（若已設定）。
  </Step>
  <Step title="已設定的設定檔">
    依供應商篩選的 `auth.profiles`。
  </Step>
  <Step title="已儲存的設定檔">
    該供應商在每個代理程式 SQLite 中的認證設定檔項目。
  </Step>
</Steps>

如果未明確設定順序，OpenClaw 會使用循環輪替順序：

- **主要索引鍵：**設定檔類型（**OAuth、靜態權杖，然後是 API 金鑰**）。
- **OAuth 的次要索引鍵：**目前有可用存取權杖的設定檔，優先於
  存取權杖已過期的設定檔。已過期的 OAuth 設定檔仍符合資格，因此
  當沒有可用的同類設定檔時，執行階段可以重新整理它們。
- **下一個索引鍵：**`usageStats.lastUsed`（在各類型／狀態層級中，最舊者優先）。
- **處於冷卻／停用狀態的設定檔**會移至末尾，並按最早到期時間排序。

### 工作階段黏著性（有利於快取）

OpenClaw 會**為每個工作階段固定選定的認證設定檔**，以保持供應商快取的熱度。它**不會**在每個要求中輪替。固定的設定檔會重複使用，直到：

- 工作階段重設（`/new`／`/reset`）
- 壓縮完成（壓縮計數增加）
- 設定檔處於冷卻／停用狀態

透過 `/model …@<profileId>` 手動選取時，會為該工作階段設定**使用者覆寫**，且在新工作階段開始前不會自動輪替。

<Note>
由工作階段路由器選取並自動固定的設定檔會被視為一種**偏好**：OpenClaw 會先嘗試這些設定檔，但遇到速率限制／逾時時，可能會輪替至其他設定檔。當原始設定檔恢復可用時，新的執行可以再次優先使用它，而不變更選取的模型或執行階段。使用者固定的設定檔會鎖定在該設定檔；如果它失敗且已設定模型後援，OpenClaw 會移至下一個模型，而不是切換設定檔。
</Note>

### OpenAI Codex 訂閱加上 API 金鑰備援

對於 OpenAI 代理程式模型，認證與執行階段彼此分離。`openai/gpt-*` 會留在 Codex 控制框架中，而認證可以在 Codex 訂閱設定檔與 OpenAI API 金鑰備援之間輪替。

使用 `auth.order.openai` 設定面向使用者的順序：

```json5
{
  auth: {
    order: {
      openai: ["openai:user@example.com", "openai:api-key-backup"],
    },
  },
}
```

對 ChatGPT/Codex OAuth 設定檔和 OpenAI API 金鑰設定檔都使用 `openai:*`。當訂閱達到 Codex 使用量限制時，如果 Codex 提供確切的重設時間，OpenClaw 會記錄該時間、嘗試下一個依序排列的認證設定檔，並讓執行維持在 Codex 控制框架中。重設時間過後，訂閱設定檔會再次符合資格，下一次自動選取即可返回該設定檔。

只有在想要強制該工作階段使用某個帳號／金鑰時，才使用由使用者固定的設定檔。使用者固定的設定檔會刻意採嚴格模式，不會在未告知的情況下跳到其他設定檔。

## 冷卻

當設定檔因認證／速率限制錯誤（或看似速率限制的逾時）而失敗時，OpenClaw 會將其標記為冷卻狀態，並移至下一個設定檔。

<AccordionGroup>
  <Accordion title="哪些情況會歸入速率限制／逾時類別">
    該速率限制類別比單純的 `429` 更廣：它也包含 `Too many concurrent requests`、`ThrottlingException`、`concurrency limit reached`、`workers_ai ... quota limit exceeded`、`throttled`、`resource exhausted` 等供應商訊息，以及 `weekly limit reached` 或 `monthly limit exhausted` 等週期性用量時段限制。

    格式／無效要求錯誤通常是終止性錯誤，因為重試相同的承載資料仍會以相同方式失敗，所以 OpenClaw 會顯示這些錯誤，而不是輪替認證設定檔。已知的重試修復路徑可以明確選擇加入：例如，Cloud Code Assist 工具呼叫 ID 驗證失敗會經過清理，並透過 `allowFormatRetry` 原則重試一次。

    與 OpenAI 相容的**供應商已完成**停止／結束原因，例如 `Unhandled stop reason: error`、`stop reason: error`、`reason: error` 和 `Provider finish_reason: error`，會分類為 **`server_error`**（類似 HTTP 的狀態 500），而非逾時。它們仍符合模型／設定檔輪替的容錯移轉資格，但診斷資訊會保留供應商的結束原因文字，而不是將使用者看到的文字改寫為「LLM 要求逾時」。類似傳輸問題的結束原因，例如 `Provider finish_reason: abort`、`network_error` 和 `malformed_response`，則仍歸入逾時／容錯移轉類別（狀態 408）。

    當來源符合已知的暫時性模式時，一般伺服器文字也可能歸入該逾時類別。例如，單獨出現的模型執行階段串流包裝器訊息 `An unknown error occurred`，對所有供應商都會視為值得容錯移轉，因為當供應商串流以 `stopReason: "aborted"` 或 `stopReason: "error"` 結束且沒有具體詳細資料時，共用模型執行階段會發出此訊息。若 JSON `api_error` 承載資料包含 `internal server error`、`unknown error, 520`、`upstream error` 或 `backend error` 等暫時性伺服器文字，也會視為值得容錯移轉的逾時。

    OpenRouter 專用的一般上游文字（例如單獨的 `Provider returned error`）只有在供應商情境確實是 OpenRouter 時，才會被視為逾時。一般內部備援文字（例如 `LLM request failed with an unknown error.`）則維持保守處理，本身不會觸發容錯移轉。

  </Accordion>
  <Accordion title="SDK retry-after 上限">
    否則，某些供應商 SDK 可能會在將控制權交還 OpenClaw 前，於很長的 `Retry-After` 時間範圍內休眠。對於 Anthropic 和 OpenAI 等以 Stainless 為基礎的 SDK，OpenClaw 預設會將 SDK 內部的 `retry-after-ms` / `retry-after` 等待時間限制為 60 秒，並立即呈現等待時間更長且可重試的回應，讓此容錯移轉路徑得以執行。可使用 `OPENCLAW_SDK_RETRY_MAX_WAIT_SECONDS` 調整或停用此上限；請參閱[重試行為](/zh-TW/concepts/retry)。
  </Accordion>
  <Accordion title="模型範圍的冷卻">
    速率限制冷卻也可以限定於模型範圍：

    - 當已知失敗的模型 ID 時，OpenClaw 會針對速率限制失敗記錄 `cooldownModel`。
    - 當冷卻限定於其他模型時，仍可嘗試相同供應商的同層模型。
    - 計費／停用時段仍會封鎖所有模型的整個設定檔。

  </Accordion>
</AccordionGroup>

一般（非計費、非永久驗證）冷卻會隨設定檔近期的錯誤次數調整：

- 第 1 次失敗：30 秒
- 第 2 次失敗：1 分鐘
- 第 3 次以上失敗：5 分鐘（上限）

設定檔內建的失敗時段結束後，計數器便會重設。

狀態會儲存在每個代理程式的 SQLite 驗證狀態中，位於 `usageStats`：

```json
{
  "usageStats": {
    "provider:profile": {
      "lastUsed": 1736160000000,
      "cooldownUntil": 1736160600000,
      "errorCount": 2
    }
  }
}
```

## 因計費而停用

計費／額度失敗（例如「額度不足」／「額度餘額過低」）會被視為應進行容錯移轉，但通常並非暫時性問題。OpenClaw 不會套用短暫冷卻，而是將設定檔標記為**已停用**（採用較長的退避時間），並輪替至下一個設定檔／供應商。

<Note>
並非每個看似計費問題的回應都是 `402`，也並非每個 HTTP `402` 都會歸入此處。即使供應商傳回 `401` 或 `403`，OpenClaw 仍會將明確的計費文字保留在計費處理路徑中，但供應商專用的比對器只會套用於其所屬供應商（例如 OpenRouter `403 Key limit exceeded`）。

同時，當訊息看來可重試時，暫時性的 `402` 使用時段與組織／工作區支出上限錯誤會被分類為 `rate_limit`（例如 `weekly usage limit exhausted`、`daily limit reached, resets tomorrow` 或 `organization spending limit exceeded`）。這些錯誤會繼續使用短暫冷卻／容錯移轉路徑，而非長時間的計費停用路徑。
</Note>

可信度高的永久驗證失敗（已撤銷／停用的金鑰、已停用的工作區）會進入類似的停用路徑，但其恢復時間比計費問題短得多，因為部分供應商在事故期間可能會暫時呈現看似驗證問題的承載資料。

狀態會儲存在每個代理程式的 SQLite 驗證狀態中：

```json
{
  "usageStats": {
    "provider:profile": {
      "disabledUntil": 1736178000000,
      "disabledReason": "billing"
    }
  }
}
```

相較於計費冷卻，系統會更積極地處理過載與速率限制錯誤：OpenClaw 預設允許重試一次相同供應商的驗證設定檔，接著不等待便切換至下一個已設定的模型備援。

## 模型備援

如果某個供應商的所有設定檔都失敗，OpenClaw 會移至 `agents.defaults.model.fallbacks` 中的下一個模型。這適用於驗證失敗、速率限制，以及設定檔輪替耗盡的逾時（其他錯誤不會推進備援）。對於未提供足夠詳細資訊的供應商錯誤，備援狀態仍會精確標記：`empty_response` 表示供應商未傳回可用的訊息或狀態、`no_error_details` 表示供應商明確傳回 `Unknown error (no error details in response)`，而 `unclassified` 表示 OpenClaw 保留了原始預覽，但尚無分類器與其相符。

`ModelNotReadyException` 等供應商忙碌訊號會歸入過載類別，並採用與速率限制相同的「輪替一次後備援」原則（請參閱上方的預設值表格）。

如果整個候選鏈僅因過載失敗而耗盡，回覆執行器會在同一輪中重試該鏈，最多 10 次。只有在工具執行或助理輸出開始前，才允許重試整輪，以免在可觀察工作完成後發生過載時造成重複異動或訊息。退避從 2.5 秒開始並倍增，最高為 30 秒。一旦此輪已等待 30 秒，OpenClaw 會傳送一次暫時性狀態通知：`The AI service is temporarily overloaded. I’m still retrying; this may take a few minutes.`。重試及任何勝出的備援都僅限於該輪；一般暫時性伺服器錯誤則維持其獨立的單次重試原則。

當執行從已設定的預設主要模型、排程工作的主要模型、具有明確備援的代理程式主要模型，或自動選取的備援覆寫開始時，OpenClaw 可以依序走訪相符的已設定備援鏈。沒有明確備援的代理程式主要模型，以及使用者明確選取的項目（例如 `/model ollama/qwen3.5:27b`、模型選擇器、`sessions.patch`，或一次性的命令列介面供應商／模型覆寫）會採取嚴格模式：如果該供應商／模型無法連線或在產生回覆前失敗，OpenClaw 會回報失敗，而不會改用不相關的備援作答。

### 候選鏈規則

OpenClaw 會根據目前要求的 `provider/model` 加上已設定的備援，建立候選清單。

<AccordionGroup>
  <Accordion title="規則">
    - 要求的模型一律排在第一位。
    - 明確設定的備援會去除重複項目，但不會依模型允許清單篩選。這些備援會被視為操作人員的明確意圖。
    - 如果目前執行已位於相同供應商系列中的已設定備援，OpenClaw 會繼續使用完整的已設定鏈。
    - 未提供明確的備援覆寫時，即使要求的模型使用不同供應商，也會先嘗試已設定的備援，再嘗試已設定的主要模型。
    - 未向備援執行器提供明確的備援覆寫時，已設定的主要模型會附加至末尾，讓較早的候選項目耗盡後，候選鏈可以回到一般預設值。
    - 當呼叫端提供 `fallbacksOverride` 時，執行器只會使用要求的模型及該覆寫清單。空白清單會停用模型備援，並防止已設定的主要模型被附加為隱藏的重試目標。

  </Accordion>
</AccordionGroup>

### 哪些錯誤會推進備援

<Tabs>
  <Tab title="在下列情況下繼續">
    - 驗證失敗
    - 速率限制與冷卻耗盡
    - 過載／供應商忙碌錯誤
    - 逾時型容錯移轉錯誤
    - 因計費而停用
    - `LiveSessionModelSwitchError`，其會被正規化為容錯移轉路徑，避免過時的持久化模型產生外層重試迴圈
    - 仍有剩餘候選項目時的其他無法辨識錯誤

  </Tab>
  <Tab title="在下列情況下不繼續">
    - 非逾時／容錯移轉型的明確中止
    - 應留在壓縮／重試邏輯內的情境溢位錯誤（例如 `request_too_large`、`input token count exceeds the maximum number of input tokens`、`input exceeds the maximum number of tokens`、`input too long for the model` 或 `ollama error: context length exceeded`）
    - 沒有剩餘候選項目時的最終未知錯誤
    - Claude Fable 5 安全性拒絕；直接使用 API 金鑰的要求會改由供應商層級處理，透過 Anthropic 的伺服器端備援改用 `claude-opus-4-8`（請參閱 [Anthropic](/zh-TW/providers/anthropic#safety-refusal-fallback-claude-fable-5)）

  </Tab>
</Tabs>

### 冷卻略過與探測行為

當某個供應商的每個驗證設定檔都已進入冷卻時，OpenClaw 不會自動永遠略過該供應商，而是針對每個候選項目做出決策：

<AccordionGroup>
  <Accordion title="每個候選項目的決策">
    - 持續性的驗證失敗會立即略過整個供應商。
    - 因計費而停用通常會略過，但主要候選項目仍可依節流頻率進行探測，以便在無須重新啟動的情況下恢復。
    - 主要候選項目可在冷卻即將到期時進行探測，且每個供應商都有各自的節流限制。
    - 當失敗看似暫時性問題（`rate_limit`、`overloaded` 或未知）時，即使處於冷卻狀態，也可以嘗試相同供應商的備援同層模型。當速率限制限定於模型範圍，而同層模型可能立即恢復時，這點尤其重要。
    - 暫時性冷卻探測在每次備援執行中，每個供應商僅限一次，以免單一供應商阻礙跨供應商備援。

  </Accordion>
</AccordionGroup>

## 工作階段覆寫與即時模型切換

工作階段模型變更是共用狀態。作用中的執行器、`/model` 命令、壓縮／工作階段更新，以及即時工作階段協調，都會讀取或寫入同一工作階段項目的不同部分。備援執行不會寫入模型選取欄位，因此重試時無法取代較新的手動選取。

即時模型切換遵循下列規則：

- 只有由使用者明確驅動的模型變更才會標記待處理的即時切換。這包括 `/model`、`session_status(model=...)` 和 `sessions.patch`。
- 由系統驅動的模型變更（例如備援輪替、心跳偵測覆寫或壓縮）本身絕不會標記待處理的即時切換。
- 由使用者驅動的模型覆寫在備援原則中會被視為精確選取，因此無法連線至所選供應商時會呈現失敗，而不會被 `agents.defaults.model.fallbacks` 掩蓋。
- 執行階段備援候選項目僅限於該輪。下一輪會從目前選取的模型開始，包括在前一輪執行期間收到的手動選取。
- 仍支援先前儲存的自動備援覆寫：OpenClaw 會定期探測其已設定的來源，並在恢復時清除覆寫；`/new`、`/reset` 和 `sessions.reset` 會立即清除自動來源的覆寫。
- 每次狀態變更時，使用者回覆會宣告一次備援轉換及備援清除後的恢復。若連續多輪使用相同的選取／作用中配對，則不會重複顯示通知。
- `/status` 會顯示所選模型；當備援狀態不同時，也會顯示作用中的備援模型及原因。
- 即時工作階段協調會優先採用持久化的工作階段覆寫，而非過時的執行階段模型欄位。
- 如果即時切換錯誤指向作用中備援鏈內較後面的候選項目，OpenClaw 會直接跳至該選取模型，而不會先依序走訪不相關的候選項目。

作用中的執行會直接攜帶其所選候選項目。即時協調只會在有明確待處理的使用者切換時變更該候選項目，因此不需要暫時性的備援覆寫或復原。

## 可觀測性與失敗摘要

`runWithModelFallback(...)` 會記錄每次嘗試的詳細資訊，並用於記錄檔和面向使用者的冷卻訊息：

- 嘗試的供應商／模型
- 原因（`rate_limit`、`overloaded`、`billing`、`auth`、`model_not_found` 及類似的容錯移轉原因）
- 選用的狀態／代碼
- 人類可讀的錯誤摘要

當候選項目失敗、被略過，或後續備援成功時，結構化的 `model_fallback_decision` 記錄也會包含扁平的 `fallbackStep*` 欄位。這些欄位會明確呈現嘗試的轉換（`fallbackStepFromModel`、`fallbackStepToModel`、`fallbackStepFromFailureReason`、`fallbackStepFromFailureDetail`、`fallbackStepFinalOutcome`），讓記錄與診斷匯出工具即使在最終備援也失敗時，仍能重建主要失敗。

當所有候選項目都失敗時，OpenClaw 會擲回 `FallbackSummaryError`。外層回覆執行器可利用此資訊建構更具體的訊息，例如「所有模型目前暫時受到速率限制」，並在已知時包含最早的冷卻到期時間。

該冷卻摘要會考量模型：

- 會忽略與嘗試的供應商／模型鏈無關、限定於模型範圍的速率限制
- 如果剩餘的封鎖是相符且限定於模型範圍的速率限制，OpenClaw 會回報仍封鎖該模型的最後一個相符到期時間

## 相關設定

請參閱[閘道設定](/zh-TW/gateway/configuration)，以瞭解：

- `auth.profiles` / `auth.order`
- `agents.defaults.model.primary` / `agents.defaults.model.fallbacks`
- `agents.defaults.imageModel` 路由

如需更全面的模型選擇與備援機制概覽，請參閱[模型](/zh-TW/concepts/models)。
