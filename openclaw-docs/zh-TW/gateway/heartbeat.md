---
read_when:
    - 調整心跳偵測頻率或訊息內容
    - 在心跳偵測與排程之間選擇以執行定時任務
sidebarTitle: Heartbeat
summary: 心跳偵測輪詢訊息與通知規則
title: 心跳偵測
x-i18n:
    generated_at: "2026-07-26T08:18:06Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 44c78e797987d8dccab910cd82fc1f482df86afce40677846d8f26522d33f6fa
    source_path: gateway/heartbeat.md
    workflow: 16
---

<Note>
**心跳偵測還是排程？** 請參閱[自動化](/zh-TW/automation)，以瞭解各自的適用時機。
</Note>

心跳偵測會在主要工作階段中執行**週期性的代理程式回合**，讓模型能提出任何需要注意的事項，而不會以大量訊息打擾你。

心跳偵測是排定的主要工作階段回合，**不會**建立[背景任務](/zh-TW/automation/tasks)記錄。任務記錄用於分離式工作（ACP 執行、子代理程式、隔離的排程工作）。

在底層，心跳偵測的頻率由排程器管理：閘道會為每個已啟用心跳偵測的代理程式維護一個系統擁有的排程工作（在 `openclaw cron list --all` 中顯示為 `Heartbeat (agent-id)`）。心跳偵測設定仍是期望狀態的輸入，而持久化的監控排程則控制實際的觸發時間點及執行器後續的冷卻時間。閘道會在啟動及重新載入設定時寫入設定變更；`openclaw doctor --fix` 可在下次閘道啟動前，建立缺少或過期的監控資料列。請編輯 `agents.*.heartbeat`，而非排程工作。

排定的心跳偵測需要排程功能。當 `cron.enabled` 為 `false` 或 `OPENCLAW_SKIP_CRON=1` 時，閘道會記錄啟動警告，且不會執行排定的心跳偵測；手動及事件驅動的心跳偵測喚醒仍可使用。沒有獨立的心跳偵測備援計時器。

疑難排解：[排定的任務](/zh-TW/automation/cron-jobs#troubleshooting)

## 快速開始（初學者）

<Steps>
  <Step title="選擇頻率">
    保持啟用心跳偵測（預設為 `30m`；設定 Anthropic OAuth／權杖驗證時則為 `1h`，包括重複使用 Claude CLI），或設定你自己的頻率。
  </Step>
  <Step title="新增監控暫存內容（選用）">
    使用 `openclaw cron scratch <jobId> --set "..."`，在心跳偵測監控的暫存內容中儲存一份精簡的檢查清單。
  </Step>
  <Step title="決定心跳偵測訊息的傳送位置">
    預設為 `target: "none"`；將 `target: "last"` 設為傳送至最後聯絡人。
  </Step>
  <Step title="選用調整">
    - 如果心跳偵測執行只需要監控暫存內容，請使用輕量啟動內容。
    - 啟用隔離的工作階段，以避免每次心跳偵測都傳送完整的對話記錄。
    - 將心跳偵測限制在活躍時段（當地時間）。

  </Step>
</Steps>

設定範例：

```json5
{
  agents: {
    defaults: {
      heartbeat: {
        every: "30m",
        target: "last", // 明確傳送至最後聯絡人（預設為 "none"）
        directPolicy: "allow", // 預設：允許直接／私訊目標；設為 "block" 即可抑制
        lightContext: true, // 選用：心跳偵測執行時略過工作區啟動檔案
        isolatedSession: true, // 選用：每次執行都使用全新工作階段（無對話記錄）
        // activeHours: { start: "08:00", end: "24:00" },
      },
    },
  },
}
```

## 預設值

- 間隔：`30m`。套用 Anthropic 提供者預設值時，若解析出的驗證模式為 OAuth／權杖（包括重複使用 Claude CLI），會將此值提高至 `1h`，但僅限 `heartbeat.every` 未設定時。設定 `agents.defaults.heartbeat.every` 或每個代理程式的 `agents.entries.*.heartbeat.every`；使用 `0m` 可停用。
- 提示詞本文（可透過 `agents.defaults.heartbeat.prompt` 設定）：`Follow the heartbeat monitor scratch context when provided. Recurring tasks are cron jobs; create or change their schedules with cron tools or the openclaw cron CLI, not heartbeat scratch. Do not infer or repeat old tasks from prior chats. If nothing needs attention, reply HEARTBEAT_OK.`
- 逾時：未設定的心跳偵測回合會在已設定 `agents.defaults.timeoutSeconds` 時使用該值。否則，它們會使用心跳偵測頻率，且上限為 600 秒。設定 `agents.defaults.heartbeat.timeoutSeconds` 或每個代理程式的 `agents.entries.*.heartbeat.timeoutSeconds`，以執行較長的心跳偵測工作。
- 心跳偵測提示詞會**逐字**作為使用者訊息傳送。當預設代理程式啟用心跳偵測時，系統提示詞會包含「心跳偵測」區段，且該次執行會在內部加上標記。
- 使用 `0m` 停用心跳偵測時，監控排程工作會保留但處於停用狀態，其暫存內容也會保留，以供你重新啟用該頻率時使用。
- 當排程功能本身停用時，即使心跳偵測頻率仍保持啟用，排定的心跳偵測也不會執行。
- 活躍時段（`heartbeat.activeHours`）會依設定的時區檢查。在時段之外，心跳偵測會略過，直到時段內的下一個觸發時間點。
- 當排程工作正在執行或排隊中，或該代理程式以工作階段索引鍵為依據的子代理程式或巢狀命令通道忙碌時，心跳偵測會自動延後。同層代理程式不會彼此暫停。

## 心跳偵測提示詞的用途

預設提示詞刻意保持廣泛：

- **背景任務**：「考量尚未完成的任務」會提示代理程式檢查後續事項（收件匣、行事曆、提醒事項、排隊中的工作），並提出任何緊急事項。
- **關心使用者**：「白天偶爾關心你的使用者」會提示偶爾傳送一則輕量的「有什麼需要嗎？」訊息，但會使用你設定的當地時區來避免夜間傳送大量訊息（請參閱[時區](/zh-TW/concepts/timezone)）。

心跳偵測可以回應已完成的[背景任務](/zh-TW/automation/tasks)，但心跳偵測執行本身不會建立任務記錄。

如果你希望心跳偵測執行非常具體的動作（例如「檢查 Gmail PubSub 統計資料」或「驗證閘道健康狀態」），請將 `agents.defaults.heartbeat.prompt`（或 `agents.entries.*.heartbeat.prompt`）設定為自訂本文（會逐字傳送）。

## 回應契約

- 如果沒有任何事項需要注意，請回覆 **`HEARTBEAT_OK`**。
- 心跳偵測執行也可以改為呼叫 `heartbeat_respond`，並搭配 `notify: false` 表示沒有可見更新；或搭配 `notify: true` 與 `notificationText` 發出警示。如果存在結構化工具回應，該回應的優先順序高於文字備援。
- 具有 `notify: false` 的有效 `heartbeat_respond` 結果會保持靜默，但會被記錄為有限的內部內容，以供該工作階段中的下一個使用者回合使用。`no_change` 確認及可見通知不會以此方式儲存。
- 在心跳偵測執行期間，當 `HEARTBEAT_OK` 出現在回覆的**開頭或結尾**時，OpenClaw 會將其視為確認。系統會移除該權杖；如果剩餘內容最多為 300 個字元，則會捨棄該回覆。
- 如果 `HEARTBEAT_OK` 出現在回覆的**中間**，系統不會對其進行特殊處理。
- 若為警示，**請勿**包含 `HEARTBEAT_OK`；只傳回警示文字。

在心跳偵測之外，訊息開頭／結尾意外出現的 `HEARTBEAT_OK` 會被移除並記錄；僅包含 `HEARTBEAT_OK` 的訊息會被捨棄。

## 設定

```json5
{
  agents: {
    defaults: {
      heartbeat: {
        every: "30m", // 預設：30m（0m 會停用）
        model: "anthropic/claude-opus-4-6",
        lightContext: false, // 預設：false；true 會在心跳偵測執行時略過工作區啟動檔案
        isolatedSession: false, // 預設：false；true 會讓每次心跳偵測在全新工作階段中執行（無對話記錄）
        target: "last", // 預設：none | 選項：last | none | <channel id>（核心或外掛，例如 "imessage"）
        to: "+15551234567", // 選用的頻道專屬覆寫值
        accountId: "ops-bot", // 選用的多帳號頻道 ID
        prompt: "提供心跳偵測監控暫存內容時，請遵循該內容。週期性任務是排程工作；請使用排程工具或 openclaw cron 命令列介面建立或變更其排程，而非使用心跳偵測暫存內容。請勿根據先前的聊天推斷或重複舊任務。如果沒有任何事項需要注意，請回覆 HEARTBEAT_OK。",
      },
    },
  },
}
```

### 範圍與優先順序

- `agents.defaults.heartbeat` 設定全域心跳偵測行為。
- `agents.entries.*.heartbeat` 會合併於其上；如果任何代理程式具有 `heartbeat` 區塊，則**只有這些代理程式**會執行心跳偵測。
- `channels.defaults.heartbeatVisibility` 設定所有頻道的可見性預設值。
- `channels.<channel>.heartbeatVisibility` 會覆寫頻道預設值。
- `channels.<channel>.accounts.<id>.heartbeatVisibility`（多帳號頻道）會覆寫每個頻道的設定。

### 每個代理程式的心跳偵測

如果任何 `agents.entries.*` 項目包含 `heartbeat` 區塊，則**只有這些代理程式**會執行心跳偵測。每個代理程式的區塊會合併於 `agents.defaults.heartbeat` 之上（因此你可以只設定一次共用預設值，再針對每個代理程式覆寫）。

範例：兩個代理程式，只有第二個代理程式執行心跳偵測。

```json5
{
  agents: {
    defaults: {
      heartbeat: {
        every: "30m",
        target: "last", // 明確傳送至最後聯絡人（預設為 "none"）
      },
    },
    list: [
      { id: "main", default: true },
      {
        id: "ops",
        heartbeat: {
          every: "1h",
          target: "whatsapp",
          to: "+15551234567",
          timeoutSeconds: 45,
          prompt: "提供心跳偵測監控暫存內容時，請遵循該內容。週期性任務是排程工作；請使用排程工具或 openclaw cron 命令列介面建立或變更其排程，而非使用心跳偵測暫存內容。請勿根據先前的聊天推斷或重複舊任務。如果沒有任何事項需要注意，請回覆 HEARTBEAT_OK。",
        },
      },
    ],
  },
}
```

### 活躍時段範例

將心跳偵測限制在特定時區的營業時間內：

```json5
{
  agents: {
    defaults: {
      heartbeat: {
        every: "30m",
        target: "last", // 明確傳送至最後聯絡人（預設為 "none"）
        activeHours: {
          start: "09:00",
          end: "22:00",
          timezone: "America/New_York", // 選用；若已設定 userTimezone，則使用該值，否則使用主機時區
        },
      },
    },
  },
}
```

在此時段之外（美東時間上午 9 點之前或晚上 10 點之後），會略過心跳偵測。時段內的下一個排定觸發時間點將正常執行。

### 24/7 設定

如果你希望心跳偵測全天執行，請使用下列其中一種模式：

- 完全省略 `activeHours`（不限制時段；這是預設行為）。
- 設定全天時段：`activeHours: { start: "00:00", end: "24:00" }`。

<Warning>
請勿將 `start` 和 `end` 設為相同時間（例如從 `08:00` 到 `08:00`）。這會被視為零寬度時段，因此一律略過心跳偵測。
</Warning>

### 多帳號範例

使用 `accountId`，以指定 Telegram 等多帳號頻道上的特定帳號：

```json5
{
  agents: {
    list: [
      {
        id: "ops",
        heartbeat: {
          every: "1h",
          target: "telegram",
          to: "12345678:topic:42", // 選用：傳送至特定主題／討論串
          accountId: "ops-bot",
        },
      },
    ],
  },
  channels: {
    telegram: {
      accounts: {
        "ops-bot": { botToken: "YOUR_TELEGRAM_BOT_TOKEN" },
      },
    },
  },
}
```

### 欄位說明

<ParamField path="every" type="string">
  心跳偵測間隔（持續時間字串；預設單位 = 分鐘）。
</ParamField>
<ParamField path="model" type="string">
  心跳偵測執行的選用模型覆寫值（`provider/model`）。
</ParamField>
<ParamField path="lightContext" type="boolean" default="false">
  設為 true 時，心跳偵測執行會使用輕量啟動內容，並略過工作區啟動檔案。無論如何，心跳偵測執行器都會注入監控暫存內容。
</ParamField>
<ParamField path="isolatedSession" type="boolean" default="false">
  設為 true 時，每次心跳偵測都會在沒有先前對話記錄的全新工作階段中執行。使用與排程 `sessionTarget: "isolated"` 相同的隔離模式。可大幅降低每次心跳偵測的權杖成本。搭配 `lightContext: true` 使用可獲得最大節省效果。傳送路由仍會使用主要工作階段內容。
</ParamField>
<ParamField path="session" type="string">
  心跳偵測執行的選用工作階段索引鍵。

- `main`（預設）：代理程式主要工作階段。
- 明確的工作階段索引鍵（從 `openclaw sessions --json` 或[工作階段命令列介面](/zh-TW/cli/sessions)複製）。
- 工作階段索引鍵格式：請參閱[工作階段](/zh-TW/concepts/session)與[群組](/zh-TW/channels/groups)。

</ParamField>
<ParamField path="target" type="string">
- `last`：傳送至上次使用的外部頻道。
- 明確指定頻道：任何已設定的頻道或外掛 ID，例如 `discord`、`matrix`、`telegram` 或 `whatsapp`。
- `none`（預設）：執行心跳偵測，但**不對外傳送**。

</ParamField>
<ParamField path="directPolicy" type='"allow" | "block"' default="allow">
  控制直接訊息／私訊的傳送行為。`allow`：允許傳送直接訊息／私訊心跳偵測。`block`：禁止傳送直接訊息／私訊（`reason=dm-blocked`）。

</ParamField>
<ParamField path="to" type="string">
  選用的收件者覆寫設定（頻道特定 ID，例如 WhatsApp 的 E.164 或 Telegram 聊天 ID）。對於 Telegram 主題／討論串，請使用 `<chatId>:topic:<messageThreadId>`。

</ParamField>
<ParamField path="accountId" type="string">
  多帳號頻道的選用帳號 ID。當 `target: "last"` 時，若解析出的上次使用頻道支援帳號，便會對其套用帳號 ID；否則忽略此設定。如果帳號 ID 與解析出之頻道的任何已設定帳號皆不相符，則略過傳送。

</ParamField>
<ParamField path="prompt" type="string">
  覆寫預設提示詞本文（不會合併）。

</ParamField>
<ParamField path="timeoutSeconds" type="number" default="global timeout or min(every, 600)">
  心跳偵測代理程式回合在中止前允許執行的秒數上限。若已設定 `agents.defaults.timeoutSeconds`，請留空以使用該值；否則使用心跳偵測週期，且上限為 600 秒。

</ParamField>
<ParamField path="activeHours" type="object">
  將心跳偵測執行限制在一段時間範圍內。此物件包含 `start`（HH:MM，包含該時間；使用 `00:00` 表示一天的開始）、`end`（HH:MM，不包含該時間；允許使用 `24:00` 表示一天的結束），以及選用的 `timezone`。

- 省略或設為 `"user"`：若已設定你的 `agents.defaults.userTimezone`，則使用該值；否則改用主機系統時區。
- `"local"`：一律使用主機系統時區。
- 任何 IANA 識別碼（例如 `America/New_York`）：直接使用；若無效，則改用上述 `"user"` 行為。
- 對於有效時間範圍，`start` 與 `end` 不得相同；相同的值會視為寬度為零（一律位於範圍外）。
- 在有效時間範圍外，會略過心跳偵測，直到下一個位於範圍內的觸發時間點。

</ParamField>

## 傳送行為

<AccordionGroup>
  <Accordion title="工作階段與目標路由">
    - 心跳偵測預設在代理程式的主要工作階段中執行（`agent:<id>:<mainKey>`）；當 `session.scope = "global"` 時，則在 `global` 中執行。設定 `session` 可覆寫為特定頻道工作階段（Discord／WhatsApp 等）。
    - `session` 僅影響執行環境；傳送由 `target` 和 `to` 控制。
    - 若要傳送至特定頻道／收件者，請設定 `target` + `to`。使用 `target: "last"` 時，傳送作業會使用該工作階段上次使用的外部頻道。
    - 心跳偵測傳送預設允許直接訊息／私訊目標。設定 `directPolicy: "block"` 可禁止傳送至直接訊息目標，同時仍執行心跳偵測回合。
    - 如果主要佇列、目標工作階段通道、排程通道或進行中的排程工作正忙碌，則會略過心跳偵測，並於稍後重試。
    - 如果 `target` 未解析出任何外部目的地，仍會執行該回合，但不會傳送任何外送訊息。

  </Accordion>
  <Accordion title="可見性與略過行為">
    - 如果 `showOk`、`showAlerts` 和 `useIndicator` 全部停用，則一開始就會以 `reason=alerts-disabled` 為由略過該回合。
    - 如果只有警示傳送遭停用，OpenClaw 仍可執行心跳偵測、更新到期工作的時間戳記、還原工作階段閒置時間戳記，並抑制對外警示承載內容。
    - 如果解析出的心跳偵測目標支援輸入中狀態，OpenClaw 會在心跳偵測執行期間顯示輸入中。此功能使用心跳偵測原本會將聊天輸出傳送至的相同目標，且會由 `typingMode: "never"` 停用。

  </Accordion>
  <Accordion title="工作階段生命週期與稽核">
    - 僅限心跳偵測的回覆**不會**讓工作階段保持有效。心跳偵測中繼資料可能會更新工作階段資料列，但閒置到期會使用上次真實使用者／頻道訊息的 `lastInteractionAt`，每日到期則使用 `sessionStartedAt`。
    - 控制介面和 WebChat 歷程記錄會隱藏心跳偵測提示詞與僅含 OK 的確認訊息。底層工作階段對話記錄仍可能包含這些回合，以供稽核／重播。
    - 分離式[背景工作](/zh-TW/automation/tasks)可將系統事件加入佇列，並在主要工作階段需要迅速注意某件事時喚醒心跳偵測。此喚醒不會使心跳偵測以背景工作執行。

  </Accordion>
</AccordionGroup>

## 可見性控制

預設會抑制 `HEARTBEAT_OK` 確認訊息，同時傳送警示內容。你可以針對每個頻道或每個帳號調整此設定：

```yaml
channels:
  defaults:
    heartbeat:
      showOk: false # 隱藏 HEARTBEAT_OK（預設）
      showAlerts: true # 顯示警示訊息（預設）
      useIndicator: true # 發出指示器事件（預設）
  telegram:
    heartbeat:
      showOk: true # 在 Telegram 上顯示 OK 確認訊息
  whatsapp:
    accounts:
      work:
        heartbeat:
          showAlerts: false # 禁止為此帳號傳送警示
```

優先順序：每個帳號 → 每個頻道 → 頻道預設值 → 內建預設值。

### 每個旗標的作用

- `showOk`：當模型僅回覆 OK 時，傳送 `HEARTBEAT_OK` 確認訊息。
- `showAlerts`：當模型回覆非 OK 內容時，傳送警示內容。
- `useIndicator`：為介面狀態表面發出指示器事件。

如果**三者皆為** false，OpenClaw 會完全略過心跳偵測執行（不呼叫模型）。

### 每個頻道與每個帳號的範例

```yaml
channels:
  defaults:
    heartbeat:
      showOk: false
      showAlerts: true
      useIndicator: true
  slack:
    heartbeat:
      showOk: true # 所有 Slack 帳號
    accounts:
      ops:
        heartbeat:
          showAlerts: false # 僅禁止 ops 帳號的警示
  telegram:
    heartbeat:
      showOk: true
```

### 常見模式

| 目標                                     | 設定                                                                                   |
| ---------------------------------------- | ---------------------------------------------------------------------------------------- |
| 預設行為（靜默處理 OK，啟用警示） | _（不需要設定）_                                                                     |
| 完全靜默（無訊息、無指示器） | `channels.defaults.heartbeat: { showOk: false, showAlerts: false, useIndicator: false }` |
| 僅限指示器（無訊息）             | `channels.defaults.heartbeat: { showOk: false, showAlerts: false, useIndicator: true }`  |
| 僅在單一頻道顯示 OK                  | `channels.telegram.heartbeat: { showOk: true }`                                          |

## 監視器暫存文件（選用）

每個心跳偵測監視器排程工作都擁有一份儲存在共用狀態資料庫中的私有暫存文件。可將它視為你的「心跳偵測檢查清單」：精簡、穩定，且適合每 30 分鐘檢視一次。暫存文件存在時，其內容會附加至心跳偵測提示詞。

使用排程命令列介面管理它（工作 ID 來自 `openclaw cron list --all`）：

```bash
openclaw cron scratch <jobId>                 # 顯示目前的暫存文件
openclaw cron scratch <jobId> --set "..."     # 以指定文字完整取代
openclaw cron scratch <jobId> --file notes.md # 使用檔案內容取代（- 表示標準輸入）
openclaw cron scratch <jobId> --unset         # 移除
```

寫入受到比較並交換機制保護：傳入 `--expected-revision <n>`，即可在發生並行編輯時失敗，而非覆寫該編輯。暫存文件上限為 256 KiB，且絕不會出現在 `cron list`/`cron runs` 輸出中。

代理程式也可更新自己的暫存文件：在心跳偵測回合期間，`heartbeat_respond` 接受選用的 `scratch` 字串，以完整取代監視器的暫存文件，供未來心跳偵測使用。

<Note>
**要從 HEARTBEAT.md 或僅使用設定的週期遷移嗎？** 請執行 `openclaw doctor --fix`。Doctor 會先根據 `agents.*.heartbeat` 建立或更新系統擁有的監視器資料列，接著將每個代理程式工作區的 `HEARTBEAT.md` 匯入監視器的暫存文件，將所有有效的舊版 `tasks:` 項目轉換為排程工作，把原始檔案封存至狀態目錄下（`backups/heartbeat-migration/`），然後移除該檔案。執行階段心跳偵測指示僅來自資料庫暫存文件；執行階段絕不會讀取 `HEARTBEAT.md`。
</Note>

如果暫存文件存在但實際上為空（僅包含空白行、Markdown／HTML 註解、如 `# Heading` 的 Markdown 標題、程式碼圍欄標記或空白檢查清單項目），OpenClaw 會略過心跳偵測執行，以節省 API 呼叫。該略過會回報為 `reason=empty-heartbeat-file`。如果暫存文件不存在，心跳偵測仍會執行，並由模型決定要採取的動作。

請保持精簡（簡短的檢查清單或提醒），以避免提示詞過度膨脹。

暫存文件範例：

```md
# 心跳偵測檢查清單

- 快速查看：收件匣中是否有任何緊急事項？
- 如果是白天，且沒有其他待處理事項，進行一次簡單的問候。
- 如果工作受阻，記下_缺少什麼_，並在下次詢問 Peter。
```

### 使用排程安排定期檢查

心跳偵測暫存文件是提示詞環境，而不是排程器。請將每項定期檢查建立為[排程工作](/zh-TW/automation/cron-jobs)，使其擁有自己的週期、啟用／停用狀態和執行歷程記錄。當檢查需要使用一般對話環境時，排程工作仍可將主要工作階段設為目標。

舊版暫存文件可能包含結構化的 `tasks:` 區塊。升級後請執行一次 `openclaw doctor --fix`：Doctor 會將每個有效項目轉換為獨立排程的排程工作、保留其間隔與先前的上次執行時間，並移除已淘汰的區塊，同時保留周圍的暫存文件文字。執行階段心跳偵測回合不會將 `tasks:` 文字解析為排程。

Doctor 建立的心跳偵測工作會保留心跳偵測的有效時段、冷卻、防洪與忙碌防護。到期時間相同的工作可合併為單一心跳偵測回合。在有效時段外發生的執行會被略過，並在其下一次排程發生時間再次嘗試。

### 代理程式可以更新自己的暫存文件嗎？

可以。在心跳偵測回合期間，代理程式可將 `scratch` 值傳給 `heartbeat_respond`，以完整取代供未來心跳偵測使用的監視器文字。你也可以在一般聊天中要求它執行 `openclaw cron scratch <jobId> --set ...`，或使用相同命令自行編輯暫存文件。請使用排程管理定期排程，而非將排程器語法寫入暫存文件。

<Warning>
請勿將祕密（API 金鑰、電話號碼、私人權杖）放入監視器暫存文件中，因為它會成為提示詞環境的一部分。
</Warning>

## 手動喚醒（隨需）

使用 `openclaw system event` 將系統事件加入佇列，並選擇是否立即觸發心跳偵測：

```bash
openclaw system event --text "檢查是否有緊急的後續事項" --mode now
```

| 旗標                         | 說明                                                                                      |
| ---------------------------- | ------------------------------------------------------------------------------------------------ |
| `--text <text>`              | 系統事件文字（必填）。                                                                    |
| `--mode <mode>`              | `now` 會立即執行一次心跳偵測；`next-heartbeat`（預設）會等待下一個排定的觸發時點。 |
| `--session-key <sessionKey>` | 將事件指定給特定工作階段；預設為代理程式的主要工作階段。                   |
| `--json`                     | 輸出 JSON。                                                                                     |

若未提供 `--session-key`，且有多個代理程式設定了 `heartbeat`，則 `--mode now` 會立即執行其中每個代理程式的心跳偵測。

同一命令列介面群組中的相關心跳偵測控制：

```bash
openclaw system heartbeat last     # 顯示最近一次心跳偵測事件
openclaw system heartbeat enable   # 啟用心跳偵測
openclaw system heartbeat disable  # 停用心跳偵測
```

## 成本考量

心跳偵測會執行完整的代理程式回合。間隔越短，消耗的權杖越多。若要降低成本：

- 使用 `isolatedSession: true`，避免傳送完整的對話歷史記錄（每次執行可從約 100K 個權杖降至約 2–5K 個）。
- 使用 `lightContext: true`，在執行心跳偵測時略過工作區啟動載入檔案。
- 設定成本較低的 `model`（例如 `ollama/llama3.2:1b`）。
- 保持監控暫存內容精簡。
- 若只需要更新內部狀態，請使用 `target: "none"`。

## 心跳偵測後的上下文溢位

心跳偵測執行完成後，會保留共用工作階段中既有的執行階段模型。因此，若心跳偵測曾將工作階段切換至較小的本機模型（例如具有 32k 上下文視窗的 Ollama 模型），下一個主要工作階段回合仍可能沿用該模型。若下一回合接著回報上下文溢位，且該工作階段最近使用的執行階段模型符合已設定的 `heartbeat.model`，OpenClaw 的復原訊息會指出心跳偵測模型殘留可能是原因，並建議修正方式。

若要避免此問題：使用 `isolatedSession: true`，在全新的工作階段中執行心跳偵測（也可搭配 `lightContext: true`，將提示詞縮至最小），或選擇上下文視窗足以容納共用工作階段的心跳偵測模型。

## 相關內容

- [自動化](/zh-TW/automation) - 一覽所有自動化機制
- [背景任務](/zh-TW/automation/tasks) - 如何追蹤分離執行的工作
- [時區](/zh-TW/concepts/timezone) - 時區如何影響心跳偵測排程
- [疑難排解](/zh-TW/automation/cron-jobs#troubleshooting) - 偵錯自動化問題
