---
read_when:
    - 新增或修改訊息卡片、圖表、表格、按鈕或選擇器的呈現方式
    - 建置支援豐富外送訊息的頻道外掛
    - 變更訊息工具的呈現方式或傳遞功能
    - 偵錯特定提供者的卡片／區塊／元件呈現迴歸問題
summary: 適用於頻道外掛的語意訊息卡片、圖表、表格、控制項、備援文字與傳遞提示
title: 訊息呈現
x-i18n:
    generated_at: "2026-07-26T08:27:20Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 1fce3874c99627eb87ceb83aebe381b8a8466722703ec6322c609f187d15d9ae
    source_path: plugins/message-presentation.md
    workflow: 16
---

訊息呈現是 OpenClaw 用於豐富外送聊天 UI 的共用契約。
它讓代理程式、命令列介面命令、核准流程和外掛只需描述一次訊息
意圖，而各個頻道外掛則以其可提供的最佳原生形式進行轉譯。

使用呈現功能建立可攜式訊息 UI：文字區段、小型情境／頁尾
文字、分隔線、圖表、表格、按鈕、選取選單，以及卡片標題／語氣。

請勿將 Discord `components`、Slack
`blocks`、Telegram `buttons`、Teams `card` 或 Feishu `card` 等新的供應商原生欄位新增至共用
訊息工具。這些是由頻道外掛擁有的轉譯器輸出。

## 契約

外掛作者從以下位置匯入公開契約：

```ts
import type {
  MessagePresentation,
  ReplyPayloadDelivery,
} from "openclaw/plugin-sdk/interactive-runtime";
```

結構：

```ts
type MessagePresentation = {
  title?: string;
  tone?: "neutral" | "info" | "success" | "warning" | "danger";
  blocks: MessagePresentationBlock[];
};

type MessagePresentationBlock =
  | { type: "text"; text: string }
  | { type: "context"; text: string }
  | { type: "divider" }
  | { type: "buttons"; buttons: MessagePresentationButton[] }
  | { type: "select"; placeholder?: string; options: MessagePresentationOption[] }
  | {
      type: "chart";
      chartType: "pie";
      title: string;
      segments: Array<{ label: string; value: number }>;
    }
  | {
      type: "chart";
      chartType: "bar" | "area" | "line";
      title: string;
      categories: string[];
      series: Array<{ name: string; values: number[] }>;
      xLabel?: string;
      yLabel?: string;
    }
  | {
      type: "table";
      caption: string;
      headers: string[];
      rows: Array<Array<string | number>>;
      rowHeaderColumnIndex?: number;
    };

type MessagePresentationAction =
  | { type: "command"; command: string }
  | { type: "callback"; value: string }
  | {
      type: "approval";
      approvalId: string;
      approvalKind: "exec" | "plugin";
      decision: "allow-once" | "allow-always" | "deny";
    }
  | {
      type: "question";
      questionId: string;
      optionValue: string;
    }
  | { type: "url"; url: string }
  | {
      type: "web-app";
      url: string;
      widgetId?: string;
    }
  | {
      type: "web-app";
      url?: string;
      widgetId: string;
    };

type MessagePresentationButton = {
  label: string;
  action?: MessagePresentationAction;
  /** 舊版回呼值。新控制項應優先使用 action。 */
  value?: string;
  /** @deprecated 請使用 type 為 "url" 的 action。 */
  url?: string;
  /** @deprecated 請使用 type 為 "web-app" 的 action。 */
  webApp?: { url: string };
  /** @deprecated 請使用 type 為 "web-app" 的 action。 */
  web_app?: { url: string };
  priority?: number;
  disabled?: boolean;
  reusable?: boolean;
  style?: "primary" | "secondary" | "success" | "danger";
};

type MessagePresentationOption = {
  label: string;
  action?: Extract<MessagePresentationAction, { type: "command" | "callback" }>;
  /** 舊版回呼值。新控制項應優先使用 action。 */
  value?: string;
};

type ReplyPayloadDelivery = {
  pin?:
    | boolean
    | {
        enabled: boolean;
        notify?: boolean;
        required?: boolean;
      };
};
```

按鈕語意：

- `action.type: "command"` 透過核心的命令路徑執行原生斜線命令。
  內建命令按鈕和選單應使用此類型。
- `action.type: "callback"` 透過頻道的互動路徑攜帶不透明的外掛資料。
  頻道外掛不得將回呼資料重新解讀為斜線
  命令。
- `action.type: "approval"` 識別一項持久的操作者核准、其明確的
  `exec` 或 `plugin` 類型，以及要求的決定。頻道外掛
  會將該動作編碼為傳輸層私有的回呼，並透過
  核准服務解析；它們不得剖析 `/approve` 命令文字或根據 ID 推斷
  類型。
- `action.type: "question"` 識別即時、由執行階段撰寫的
  `ask_user` 問題中的一個選項。與 `approval` 相同，這是 OpenClaw 執行階段動作；
  代理程式和外掛不得自行產生問題 ID。Telegram、Discord 和
  Slack 會將其對應至傳輸層私有的原生回呼，並透過
  閘道解析選項。當問題已回答、過期或
  取消時，這些頻道會編輯已傳送的訊息、移除其動作，
  並附加最終狀態。WhatsApp、Signal 和 iMessage 會將最多
  四個單選選項轉譯為 `1️⃣` 至 `4️⃣` 回應。其他問題
  形式會降級為標籤文字，而使用者可以用純文字
  回覆作答。
- `action.type: "url"` 會開啟一般連結。
- `action.type: "web-app"` 會啟動頻道原生的網頁應用程式。請為
  URL 支援的應用程式設定 `url`，或為啟動
  機制由頻道擁有、託管於 OpenClaw 的小工具設定 `widgetId`；至少必須提供其中一項。當兩者
  都存在時，頻道可以優先使用其原生託管小工具啟動機制，並在該機制
  不可用時使用 URL。
- `value` 是舊版不透明回呼值。新控制項應使用 `action`，
  讓頻道外掛無須根據文字猜測，即可對應命令和回呼。
- `url`、`webApp` 和 `web_app` 仍可作為已棄用的邊界輸入。
  正規化器會保留這些欄位，讓轉譯器能區分已發布的舊版
  語意與明確的型別化動作。新的產生端應使用 `action`。
- `label` 為必填，也會用於文字備援。
- `style` 僅供提示。轉譯器應將不支援的樣式對應至安全的
  預設值，而非使傳送失敗。
- `priority` 為選填。當頻道宣告動作限制且必須捨棄部分控制項時，
  核心會優先保留優先順序較高的按鈕，並在優先順序相同的按鈕之間維持
  原始順序。當所有控制項都容納得下時，會保留撰寫時的
  順序。
- `disabled` 為選填。頻道必須透過 `supportsDisabled` 明確選擇加入；否則
  核心會將停用的控制項降級為非互動式備援文字。
  即使停用的按鈕帶有 `command` 動作，在備援文字中也一律只轉譯
  標籤。
- `reusable` 為選填。支援可重複使用原生回呼的頻道，可以在
  互動成功後繼續提供該動作。它適用於
  重新整理、檢查或更多詳細資料等可重複或具冪等性的動作；
  一般的一次性核准和破壞性動作則不要設定。

選取語意：

- `options[].action` 僅接受 `command` 或 `callback`；核准和連結動作僅能用於按鈕。
- `options[].value` 是舊版選取的應用程式值。
- `placeholder` 僅供提示，不支援原生選取功能的頻道可以忽略。
- 如果頻道不支援選取功能，備援文字會列出標籤。

圖表語意：

- `pie` 要求區段值必須為正數。
- `bar`、`area` 和 `line` 使用一個有序的 `categories` 陣列。每個數列
  都必須依相同順序，為每個類別提供恰好一個有限值。
- 類別標籤和數列名稱必須是唯一的。無效或不完整的圖表
  區塊會在正規化期間遭到捨棄，而不會默默變更資料。
- 必須透過 `presentationCapabilities.charts` 明確選擇加入原生圖表轉譯。
  其他頻道會以確定性的文字接收圖表標題、座標軸、類別、數列和值。
  這也是無障礙備援。

表格語意：

- `caption` 是必填的簡短標題。`headers` 必須包含至少一個
  唯一且非空白的欄標籤。
- `rows` 必須包含至少一列。每列必須恰好為每個
  表頭提供一個儲存格，而每個儲存格都必須是非空白字串或有限數值。
- `rowHeaderColumnIndex` 是選填的零起始索引，用於識別原生轉譯器應將其
  儲存格公開為列標題的欄。
- 表格正規化具原子性。無效的說明文字、表頭、列寬、儲存格
  或列標題索引會使表格區塊遭到捨棄，而不是截斷或修復
  其資料。
- 必須透過 `presentationCapabilities.tables` 明確選擇加入原生表格轉譯。
  其他頻道會以確定性的線性文字接收說明文字和每一列，
  並摺疊內部空白：

  ```text
  開放中的銷售管線（表格）
  - 帳戶：Acme；階段：已贏得；ARR：125000
  - 帳戶：Globex；階段：審查；ARR：82000
  ```

沒有獨立的 `report` 判別欄位。請使用 `title`、
`tone`、`text`、`context`、`chart`、`table` 和動作區塊組成報告。這能讓每個
區塊都可獨立轉譯，並讓完整報告使用相同的
確定性文字備援。

## 產生端範例

簡易卡片：

```json
{
  "title": "部署核准",
  "tone": "warning",
  "blocks": [
    { "type": "text", "text": "Canary 已準備好升級。" },
    { "type": "context", "text": "組建 1234，預備環境已通過。" },
    {
      "type": "buttons",
      "buttons": [
        {
          "label": "核准",
          "action": { "type": "callback", "value": "deploy:approve" },
          "style": "success"
        },
        {
          "label": "拒絕",
          "action": { "type": "callback", "value": "deploy:decline" },
          "style": "danger"
        }
      ]
    }
  ]
}
```

僅 URL 的連結按鈕：

```json
{
  "blocks": [
    { "type": "text", "text": "版本資訊已準備就緒。" },
    {
      "type": "buttons",
      "buttons": [
        {
          "label": "開啟版本資訊",
          "action": { "type": "url", "url": "https://example.com/release" }
        }
      ]
    }
  ]
}
```

Telegram Mini App 按鈕：

```json
{
  "blocks": [
    {
      "type": "buttons",
      "buttons": [
        {
          "label": "啟動",
          "action": { "type": "web-app", "url": "https://example.com/app" }
        }
      ]
    }
  ]
}
```

選取選單：

```json
{
  "title": "選擇環境",
  "blocks": [
    {
      "type": "select",
      "placeholder": "環境",
      "options": [
        { "label": "Canary", "value": "env:canary" },
        { "label": "正式環境", "value": "env:prod" }
      ]
    }
  ]
}
```

圖表：

```json
{
  "blocks": [
    {
      "type": "chart",
      "chartType": "line",
      "title": "季度營收",
      "categories": ["第 1 季", "第 2 季", "第 3 季"],
      "series": [
        { "name": "產品", "values": [120, 145, 138] },
        { "name": "服務", "values": [80, 95, 104] }
      ],
      "xLabel": "季度",
      "yLabel": "營收"
    }
  ]
}
```

表格報告：

```json
{
  "title": "銷售管線報告",
  "tone": "info",
  "blocks": [
    { "type": "text", "text": "依階段顯示目前的商機。" },
    {
      "type": "table",
      "caption": "開放中的銷售管線",
      "headers": ["帳戶", "階段", "ARR"],
      "rows": [
        ["Acme", "已贏得", 125000],
        ["Globex", "審查", 82000]
      ],
      "rowHeaderColumnIndex": 0
    },
    { "type": "context", "text": "已從 CRM 快照更新。" }
  ]
}
```

命令列介面傳送：

```bash
openclaw message send --channel slack \
  --target channel:C123 \
  --message "部署核准" \
  --presentation '{"title":"部署核准","tone":"warning","blocks":[{"type":"text","text":"Canary 已準備就緒。"},{"type":"buttons","buttons":[{"label":"核准","value":"deploy:approve","style":"success"},{"label":"拒絕","value":"deploy:decline","style":"danger"}]}]}'
```

釘選傳送：

```bash
openclaw message send --channel telegram \
  --target -1001234567890 \
  --message "已開啟主題" \
  --pin
```

使用明確 JSON 的釘選傳送：

```json
{
  "pin": {
    "enabled": true,
    "notify": true,
    "required": false
  }
}
```

## 轉譯器合約

頻道外掛會在其輸出配接器上宣告轉譯支援：

```ts
const adapter: ChannelOutboundAdapter = {
  deliveryMode: "direct",
  presentationCapabilities: {
    supported: true,
    buttons: true,
    selects: true,
    context: true,
    divider: true,
    charts: false,
    tables: false,
    limits: {
      actions: {
        maxActions: 25,
        maxActionsPerRow: 5,
        maxRows: 5,
        maxLabelLength: 80,
        maxValueBytes: 100,
        supportsStyles: true,
        supportsDisabled: false,
      },
      selects: {
        maxOptions: 25,
        maxLabelLength: 100,
        maxValueBytes: 100,
      },
      text: {
        maxLength: 2000,
        encoding: "characters",
        markdownDialect: "discord-markdown",
      },
    },
  },
  deliveryCapabilities: {
    pin: true,
  },
  renderPresentation({ payload, presentation, ctx }) {
    return renderNativePayload(payload, presentation, ctx);
  },
  async pinDeliveredMessage({ target, messageId, pin }) {
    await pinNativeMessage(target, messageId, { notify: pin.notify === true });
  },
};
```

能力布林值描述轉譯器能讓哪些項目具備互動功能。選用的
`limits` 描述核心在呼叫轉譯器前可調整的通用封套：

```ts
type ChannelPresentationCapabilities = {
  supported?: boolean;
  buttons?: boolean;
  selects?: boolean;
  context?: boolean;
  divider?: boolean;
  charts?: boolean;
  tables?: boolean;
  limits?: {
    actions?: {
      maxActions?: number;
      maxActionsPerRow?: number;
      maxRows?: number;
      maxLabelLength?: number;
      maxValueBytes?: number;
      supportsStyles?: boolean;
      supportsDisabled?: boolean;
      supportsLayoutHints?: boolean;
    };
    selects?: {
      maxOptions?: number;
      maxLabelLength?: number;
      maxValueBytes?: number;
    };
    text?: {
      maxLength?: number;
      encoding?: "characters" | "utf8-bytes" | "utf16-units";
      markdownDialect?: "plain" | "markdown" | "html" | "slack-mrkdwn" | "discord-markdown";
      supportsEdit?: boolean;
    };
  };
};
```

核心會在轉譯前對語意控制項套用通用限制。轉譯器仍負責最終的供應商特定驗證與截斷，包括原生區塊數量、卡片大小、URL 限制，以及無法在通用合約中表達的供應商特殊行為。如果限制移除了區塊中的所有控制項，核心會將標籤保留為非互動式情境文字，使傳送的訊息仍有可見的備援內容。

## 核心轉譯流程

在命令列介面與標準訊息動作所使用的標準輸出路徑上，核心會：

1. 正規化呈現內容承載。
2. 解析目標頻道的輸出配接器。
3. 讀取 `presentationCapabilities`。
4. 當配接器宣告相關限制時，套用動作數量、標籤長度與選取選項數量等通用能力限制。除非配接器分別明確宣告
   `charts: true` 或 `tables: true`，否則圖表與表格區塊會轉為具確定性的文字。
5. 當配接器能轉譯內容承載時，呼叫 `renderPresentation`。
6. 當配接器不存在或無法轉譯時，退回保守的文字格式。
7. 透過一般頻道傳送路徑傳送產生的內容承載。
8. 在第一則訊息成功送出後，套用 `delivery.pin` 等傳送中繼資料。

直接使用 `ReplyPayload` 的頻道本地回覆或預覽匯流流程，必須進入該標準路徑，或在將內容承載投射為純文字／媒體前，具體化相同的呈現備援內容。

核心負責備援行為，讓產生端可維持與頻道無關。頻道外掛負責原生轉譯與互動處理。

## 降級規則

呈現內容必須能安全地傳送至能力受限的頻道。

備援文字包括：

- 第一行的 `title`
- 以一般段落呈現的 `text` 區塊
- 以精簡情境行呈現的 `context` 區塊
- 以視覺分隔線呈現的 `divider` 區塊
- 按鈕標籤，包括連結按鈕的 URL
- 選取選項標籤
- 圖表標題、類型、座標軸、類別、數列和值
- 表格標題、標頭及每一列的值

### 按鈕值的備援可見性

當頻道無法轉譯互動式控制項時，按鈕與選取值會退回純文字。此備援行為會維持可用性，同時將不透明的回呼資料保留為私密資訊：

- **`command` 類型的動作**會轉譯為 `` label: `command` ``，讓使用者能複製命令，並在頻道輸入欄中手動執行。
- **`callback` 類型的動作**與舊版 **`value`** 欄位只會轉譯標籤。不透明的回呼值不會公開於備援文字中。
- **`approval` 類型的動作**只會轉譯標籤。核准 ID 與決策屬於傳輸資料，不會透過通用純量輔助函式或備援文字公開。
- **`url` 動作**、由 URL 支援的 **`web-app` 動作**，以及已棄用的 **`url` /
  `webApp` / `web_app`** 輸入，會在按鈕標籤旁轉譯 URL 文字，因為 URL 是面向使用者的內容。僅限託管小工具的動作，在沒有原生小工具啟動功能的頻道上只會轉譯標籤。
- **選取選項**只會轉譯標籤。底層選項值不會公開於備援文字中。

在備援 UI 中新增手動命令指引的頻道配接器（例如 Feishu 文件留言指示），必須根據備援轉譯器使用的相同呈現區塊，判斷是否存在命令，使指引文字只在確實顯示手動命令時出現。

不受支援的原生控制項應降級，而不是讓整次傳送失敗。例如：

- 停用行內按鈕的 Telegram 會傳送文字備援內容。
- 不支援選取功能的頻道會以文字列出選取選項。
- 不支援原生圖表的頻道會以文字列出圖表資料。
- 不支援原生表格的頻道會以文字列出每一個表格列。
- 僅含 URL 的按鈕會變成原生連結按鈕或備援 URL 行。
- 選用的釘選失敗不會導致已傳送訊息失敗。

主要例外是 `delivery.pin.required: true`；如果要求必須釘選，而頻道無法釘選已傳送的訊息，傳送作業會回報失敗。

## 供應商對應

目前內建的轉譯器：

| 頻道         | 原生轉譯目標                      | 備註                                                                                                                                                                                                             |
| --------------- | ----------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Discord         | 元件與元件容器       | 為現有供應商原生內容承載產生端保留舊版 `channelData.discord.components`，但新的共用傳送應使用 `presentation`。                                                                 |
| Feishu          | 互動式卡片                         | 卡片標頭可使用 `title`；本文會避免重複該標題。                                                                                                                                                  |
| Matrix          | 文字備援加結構化事件欄位 | 按鈕／選取功能會宣告為受支援，但目前每個區塊都會轉譯為包含在 `com.openclaw.presentation` 事件欄位中的 `renderMessagePresentationFallbackText` 輸出，而非原生互動式小工具。 |
| Mattermost      | 文字加互動式屬性               | 不支援選取功能與分隔線；這些區塊會降級為文字。                                                                                                                                             |
| Microsoft Teams | Adaptive Cards                            | 同時提供卡片和純 `message` 文字時，卡片中會包含該文字。不支援選取功能、樣式與停用狀態。                                                                                     |
| Slack           | Block Kit                                 | 將 `chart` 轉譯為原生 `data_visualization`，並將 `table` 轉譯為原生 `data_table`；保留舊版 `channelData.slack.blocks`，但新的共用傳送應使用 `presentation`。                                   |
| Telegram        | 文字加行內鍵盤                | 按鈕／選取功能要求目標介面具備行內按鈕能力；否則會使用文字備援內容。                                                                                                         |
| 純文字頻道  | 文字備援                             | 沒有轉譯器的頻道仍會獲得可讀的輸出。                                                                                                                                                            |

供應商原生內容承載相容性是為現有回覆產生端提供的過渡機制。這不構成新增共用原生欄位的理由。

## Presentation 與 InteractiveReply

`InteractiveReply` 是核准與互動輔助函式使用的較舊內部子集。它支援：

- 文字
- 按鈕
- 選取功能

`MessagePresentation` 是標準的共用傳送合約。它新增：

- 標題
- 語調
- 情境
- 分隔線
- 圖表
- 表格
- 僅含 URL 的按鈕
- 透過 `ReplyPayload.delivery` 提供的通用傳送中繼資料

橋接舊版程式碼時，請使用 `openclaw/plugin-sdk/interactive-runtime` 中的輔助函式：

```ts
import {
  adaptMessagePresentationForChannel,
  applyPresentationActionLimits,
  hasMessagePresentationBlocks,
  interactiveReplyToPresentation,
  isMessagePresentationInteractiveBlock,
  normalizeMessagePresentation,
  presentationPageSize,
  presentationToInteractiveControlsReply,
  presentationToInteractiveReply,
  renderMessagePresentationChartFallbackText,
  renderMessagePresentationFallbackText,
  renderMessagePresentationTableFallbackText,
  resolveMessagePresentationActionValue,
  resolveMessagePresentationButtonAction,
  resolveMessagePresentationControlValue,
  resolveMessagePresentationOptionAction,
} from "openclaw/plugin-sdk/interactive-runtime";
```

新程式碼應直接接受或產生 `MessagePresentation`。現有的
`interactive` 內容承載是 `presentation` 的已棄用子集；仍保留執行階段支援供舊版產生端使用。

值得了解的未棄用輔助函式：

- `normalizeMessagePresentation(raw)` / `hasMessagePresentationBlocks(value)`
  驗證未指定型別的承載資料（例如來自命令列介面
  `--presentation` 旗標的 JSON）並進行型別強制轉換，以產生 `MessagePresentation`。
- `isMessagePresentationInteractiveBlock(block)` 將區塊的型別縮限為
  `buttons` | `select` 聯集。
- `resolveMessagePresentationButtonAction(button)` 和
  `resolveMessagePresentationOptionAction(option)` 會傳回標準型別化動作，
  同時接受已淘汰的邊界欄位。明確的 `action`
  一律優先。
- `resolveMessagePresentationActionValue(action)` /
  `resolveMessagePresentationControlValue(control)` 僅讀取命令／回呼的
  純量值。非純量的標準動作絕不會遞降至舊版影子
  `value`，因此核准 ID 和連結目標會維持其型別。
- `renderMessagePresentationChartFallbackText(block)` /
  `renderMessagePresentationTableFallbackText(block)` 會將單一結構化
  資料區塊呈現為確定性文字，供頻道專屬的後援路徑使用。

舊版 `InteractiveReply*` 型別和轉換輔助函式在 SDK 中標記為
`@deprecated`：

- `InteractiveReply`、`InteractiveReplyBlock`、`InteractiveReplyButton` 和
  `InteractiveReplyOption`
- `normalizeInteractiveReply(...)`
- `hasInteractiveReplyBlocks(...)`
- `interactiveReplyToPresentation(...)`
- `presentationToInteractiveReply(...)`
- `presentationToInteractiveControlsReply(...)`
- `resolveInteractiveTextFallback(...)`
- `reduceInteractiveReply(...)`

`presentationToInteractiveReply(...)` 和
`presentationToInteractiveControlsReply(...)` 仍可作為舊版頻道實作的呈現器
橋接。新的產生端程式碼不應呼叫它們；請傳送 `presentation`，並讓核心／頻道調適機制處理呈現。

核准輔助函式也有以呈現為優先的替代項目：

- 使用 `buildApprovalPresentation(...)`，而非
  `buildApprovalInteractiveReply(...)`
- 使用 `buildExecApprovalPresentation(...)`，而非
  `buildExecApprovalInteractiveReply(...)`

為了維持外掛相容性，這些已發布的建構器仍以命令為基礎。擁有持久核准種類的閘道
和內建頻道程式碼應使用
`buildTypedApprovalPresentation(...)`、
`buildTypedExecApprovalPendingReplyPayload(...)` 或
`buildTypedPluginApprovalPendingReplyPayload(...)`，讓傳輸層收到
明確的 `approval` 動作，而不是從 `/approve` 文字推斷語意。

對於沒有文字後援的呈現區塊（例如僅含分隔線的
呈現），`renderMessagePresentationFallbackText(...)` 會傳回空字串。
需要非空白傳送本文的傳輸層可以傳入
`emptyFallback`，選擇使用最小本文，而不變更預設後援
合約。

## 傳遞釘選

釘選是傳遞行為，而非呈現。請使用 `delivery.pin`，而不是
`channelData.telegram.pin` 等供應商原生欄位。

語意：

- `pin: true` 會釘選第一則成功傳遞的訊息。
- `pin.notify` 預設為 `false`。
- `pin.required` 預設為 `false`。
- 選用的釘選失敗會降級處理，並保留已傳送的訊息。
- 必要的釘選失敗會使傳遞失敗。
- 分塊訊息會釘選第一個已傳遞的區塊，而非末尾區塊。

對於供應商支援這些操作的現有
訊息，手動 `pin`、`unpin` 和 `pins` 訊息動作仍然存在。

## 外掛作者檢查清單

- 當頻道可呈現語意式呈現內容，或可安全地將其降級時，請從 `describeMessageTool(...)`
  宣告 `presentation`。
- 將 `presentationCapabilities` 新增至執行階段輸出配接器。
- 在執行階段程式碼中實作 `renderPresentation`，而非在控制平面的外掛
  設定程式碼中實作。
- 不要讓原生 UI 程式庫進入高頻設定／目錄路徑。
- 已知通用能力限制時，請在 `presentationCapabilities.limits` 上
  宣告這些限制。
- 在呈現器和測試中保留最終平台限制。
- 為不支援的圖表、表格、按鈕、選取項、URL
  按鈕、標題／文字重複，以及混合 `message` 與 `presentation`
  傳送新增後援測試。
- 僅當供應商能釘選已傳送的訊息 ID 時，才透過 `deliveryCapabilities.pin` 和
  `pinDeliveredMessage` 新增傳遞釘選支援。
- 不要透過共用訊息動作結構描述公開新的供應商原生資訊卡／區塊／元件／按鈕
  欄位。

## 相關文件

- [訊息命令列介面](/zh-TW/cli/message)
- [外掛 SDK 概觀](/zh-TW/plugins/sdk-overview)
- [外掛架構](/zh-TW/plugins/architecture-internals#message-tool-schemas)
- [頻道呈現重構計畫](/zh-TW/plan/ui-channels)
