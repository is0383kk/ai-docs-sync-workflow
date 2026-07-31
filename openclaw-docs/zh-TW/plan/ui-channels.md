---
read_when:
    - 重構頻道訊息 UI、互動式承載資料或原生頻道轉譯器
    - 變更訊息工具功能、傳遞提示或跨情境標記
    - 偵錯 Discord Carbon 匯入扇出或頻道外掛執行階段的延遲載入行為
summary: 將語意訊息呈現與頻道原生 UI 轉譯器解耦。
title: 頻道呈現重構計畫
x-i18n:
    generated_at: "2026-07-26T08:00:17Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 6b0f0c4f64e0c503209ac0a5b763b1b5483bf8d55a28ceacffbbcd1337d4371e
    source_path: plan/ui-channels.md
    workflow: 16
---

## 狀態

已在共用代理程式、命令列介面、外掛能力及出站傳遞介面中實作：

- `ReplyPayload.presentation` 承載語意式訊息 UI。
- `ReplyPayload.delivery.pin` 承載已傳送訊息的釘選要求。
- 共用訊息動作公開 `presentation`、`delivery` 和 `pin`，而非提供者原生的 `components`、`blocks`、`buttons` 或 `card`。
- 核心會透過外掛宣告的出站能力呈現內容，或自動降級呈現方式。
- Discord、Slack、Telegram、Mattermost、MS Teams 和 Feishu 的轉譯器會使用通用合約。
- Discord 頻道控制平面程式碼不再匯入以 Carbon 為基礎的 UI 容器。

標準文件現位於[訊息呈現](/zh-TW/plugins/message-presentation)。
請保留此計畫作為歷史實作背景；若合約、轉譯器或備援行為有所變更，
請更新標準指南。

## 問題

頻道 UI 目前分散於數個互不相容的介面：

- 核心透過 `buildCrossContextComponents` 擁有一個採用 Discord 形態的跨情境轉譯器掛鉤。
- Discord `channel.ts` 可透過 `DiscordUiContainer` 匯入原生 Carbon UI，因而將執行階段 UI 相依性引入頻道外掛的控制平面。
- 代理程式和命令列介面公開原生承載資料的逃生口，例如 Discord 的 `components`、Slack 的 `blocks`、Telegram 或 Mattermost 的 `buttons`，以及 Teams 或 Feishu 的 `card`。
- `ReplyPayload.channelData` 同時承載傳輸提示和原生 UI 封套。
- 通用的 `interactive` 模型已存在，但其涵蓋範圍比 Discord、Slack、Teams、Feishu、LINE、Telegram 和 Mattermost 已使用的豐富版面配置更窄。

這使核心必須感知原生 UI 形態、削弱外掛執行階段的延遲載入能力，並讓代理程式能以過多提供者特定方式表達相同的訊息意圖。

## 目標

- 核心會依據宣告的能力，決定訊息的最佳語意呈現方式。
- 擴充功能會宣告能力，並將語意呈現轉譯為原生傳輸承載資料。
- Web 控制 UI 與聊天原生 UI 維持分離。
- 共用代理程式或命令列介面的訊息介面不公開原生頻道承載資料。
- 不支援的呈現功能會自動降級為最佳文字表示形式。
- 釘選已傳送訊息等傳遞行為屬於通用傳遞中繼資料，而非呈現內容。

## 非目標

- 不為 `buildCrossContextComponents` 提供向後相容性轉接層。
- 不為 `components`、`blocks`、`buttons` 或 `card` 提供公開的原生逃生口。
- 核心不匯入頻道原生 UI 程式庫。
- 不為內建頻道提供提供者特定的 SDK 介面。

## 目標模型

將核心所擁有的 `presentation` 欄位新增至 `ReplyPayload`。

```ts
type MessagePresentationTone = "neutral" | "info" | "success" | "warning" | "danger";

type MessagePresentation = {
  tone?: MessagePresentationTone;
  title?: string;
  blocks: MessagePresentationBlock[];
};

type MessagePresentationBlock =
  | { type: "text"; text: string }
  | { type: "context"; text: string }
  | { type: "divider" }
  | { type: "buttons"; buttons: MessagePresentationButton[] }
  | { type: "select"; placeholder?: string; options: MessagePresentationOption[] };

type MessagePresentationButton = {
  label: string;
  value?: string;
  url?: string;
  style?: "primary" | "secondary" | "success" | "danger";
};

type MessagePresentationOption = {
  label: string;
  value: string;
};
```

遷移期間，`interactive` 會成為 `presentation` 的子集：

- `interactive` 文字區塊會對應至 `presentation.blocks[].type = "text"`。
- `interactive` 按鈕區塊會對應至 `presentation.blocks[].type = "buttons"`。
- `interactive` 選取區塊會對應至 `presentation.blocks[].type = "select"`。

外部代理程式和命令列介面結構描述現在使用 `presentation`；`interactive` 仍是供現有回覆產生器使用的內部舊版剖析／轉譯輔助工具。
面向公開產生端的 API 將 `interactive` 視為已棄用。仍會保留執行階段
支援，讓現有核准輔助工具和較舊的外掛能繼續運作，同時讓新程式碼發出
`presentation`。

## 傳遞中繼資料

新增核心所擁有的 `delivery` 欄位，用於非 UI 的傳送行為。

```ts
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

語意：

- `delivery.pin = true` 表示釘選第一則成功傳遞的訊息。
- `notify` 預設為 `false`。
- `required` 預設為 `false`；遇到不支援的頻道或釘選失敗時，會繼續傳遞以自動降級。
- 手動的 `pin`、`unpin` 和 `list-pins` 訊息動作仍保留供現有訊息使用。

目前的 Telegram ACP 主題繫結應從 `channelData.telegram.pin = true` 移至 `delivery.pin = true`。

## 執行階段能力合約

將呈現與傳遞轉譯掛鉤新增至執行階段出站配接器，而非控制平面的頻道外掛。

```ts
type ChannelPresentationCapabilities = {
  supported: boolean;
  buttons?: boolean;
  selects?: boolean;
  context?: boolean;
  divider?: boolean;
  tones?: MessagePresentationTone[];
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

type ChannelDeliveryCapabilities = {
  pinSentMessage?: boolean;
};

type ChannelOutboundAdapter = {
  presentationCapabilities?: ChannelPresentationCapabilities;

  renderPresentation?: (params: {
    payload: ReplyPayload;
    presentation: MessagePresentation;
    ctx: ChannelOutboundSendContext;
  }) => ReplyPayload | null;

  deliveryCapabilities?: ChannelDeliveryCapabilities;

  pinDeliveredMessage?: (params: {
    cfg: OpenClawConfig;
    accountId?: string | null;
    to: string;
    threadId?: string | number | null;
    messageId: string;
    notify: boolean;
  }) => Promise<void>;
};
```

核心行為：

- 解析目標頻道和執行階段配接器。
- 查詢呈現能力。
- 在轉譯前，先將不支援的區塊降級並套用通用能力限制。
- 呼叫 `renderPresentation`。
- 若沒有轉譯器，則將呈現內容轉換為文字備援。
- 成功傳送後，若要求且支援 `delivery.pin`，則呼叫 `pinDeliveredMessage`。

## 頻道對應

Discord：

- 在僅限執行階段的模組中，將 `presentation` 轉譯為 components v2 和 Carbon 容器。
- 將強調色彩輔助工具保留在輕量模組中。
- 從頻道外掛控制平面程式碼中移除 `DiscordUiContainer` 匯入。

Slack：

- 將 `presentation` 轉譯為 Block Kit。
- 移除代理程式和命令列介面的 `blocks` 輸入。

Telegram：

- 將文字、情境和分隔線轉譯為文字。
- 若已設定且目標介面允許，將動作和選取項目轉譯為行內鍵盤。
- 停用行內按鈕時使用文字備援。
- 將 ACP 主題釘選移至 `delivery.pin`。

Mattermost：

- 若已設定，將動作轉譯為互動式按鈕。
- 將其他區塊轉譯為文字備援。

MS Teams：

- 將 `presentation` 轉譯為 Adaptive Cards。
- 保留手動釘選／取消釘選／列出釘選項目的動作。
- 若 Graph 對目標對話的支援可靠，可選擇實作 `pinDeliveredMessage`。

Feishu：

- 將 `presentation` 轉譯為互動式卡片。
- 保留手動釘選／取消釘選／列出釘選項目的動作。
- 若 API 行為可靠，可選擇實作 `pinDeliveredMessage` 以釘選已傳送的訊息。

LINE：

- 盡可能將 `presentation` 轉譯為 Flex 或範本訊息。
- 對不支援的區塊改用文字備援。
- 從 `channelData` 移除 LINE UI 承載資料。

純文字或功能受限的頻道：

- 以保守格式將呈現內容轉換為文字。

## 重構步驟

1. 重新套用 Discord 發行版修正：將 `ui-colors.ts` 與以 Carbon 為基礎的 UI 分離，並從 `extensions/discord/src/channel.ts` 移除 `DiscordUiContainer`。
2. 將 `presentation` 和 `delivery` 新增至 `ReplyPayload`、出站承載資料正規化、傳遞摘要和掛鉤承載資料。
3. 在範圍狹窄的 SDK／執行階段子路徑中新增 `MessagePresentation` 結構描述和剖析輔助工具。
4. 以語意呈現能力取代訊息能力 `buttons`、`cards`、`components` 和 `blocks`。
5. 為呈現轉譯和傳遞釘選新增執行階段出站配接器掛鉤。
6. 以 `buildCrossContextPresentation` 取代跨情境元件建構。
7. 刪除 `src/infra/outbound/channel-adapters.ts`，並從頻道外掛型別移除 `buildCrossContextComponents`。
8. 變更 `maybeApplyCrossContextMarker`，使其附加 `presentation`，而非原生參數。
9. 更新外掛分派傳送路徑，使其僅使用語意呈現和傳遞中繼資料。
10. 移除代理程式和命令列介面的原生承載資料參數：`components`、`blocks`、`buttons` 和 `card`。
11. 移除建立原生訊息工具結構描述的 SDK 輔助工具，改以呈現結構描述輔助工具取代。
12. 從 `channelData` 移除 UI／原生封套；在檢視每個剩餘欄位前，僅保留傳輸中繼資料。
13. 遷移 Discord、Slack、Telegram、Mattermost、MS Teams、Feishu 和 LINE 的轉譯器。
14. 更新訊息命令列介面、頻道頁面、外掛 SDK 和能力操作指南的文件。
15. 對 Discord 和受影響的頻道進入點執行匯入扇出分析。

此次重構已針對共用代理程式、命令列介面、外掛能力和出站配接器合約實作步驟 1-11 與 13-14。步驟 12 仍是針對提供者私有 `channelData` 傳輸封套的更深入內部清理。若需要型別／測試閘門以外的量化匯入扇出數據，步驟 15 仍需後續驗證。

## 測試

新增或更新：

- 呈現正規化測試。
- 針對不支援區塊的呈現自動降級測試。
- 外掛分派和核心傳遞路徑的跨情境標記測試。
- Discord、Slack、Telegram、Mattermost、MS Teams、Feishu、LINE 和文字備援的頻道轉譯矩陣測試。
- 證明原生欄位已移除的訊息工具結構描述測試。
- 證明原生旗標已移除的命令列介面測試。
- 涵蓋 Carbon 的 Discord 進入點匯入延遲載入迴歸測試。
- 涵蓋 Telegram 和通用備援的傳遞釘選測試。

## 未決問題

- 第一階段應為 Discord、Slack、MS Teams 和 Feishu 實作 `delivery.pin`，還是先只支援 Telegram？
- `delivery` 最終是否應納入 `replyToId`、`replyToCurrent`、`silent` 和 `audioAsVoice` 等現有欄位，還是維持專注於傳送後的行為？
- 呈現功能是否應直接支援圖片或檔案參照，還是媒體目前應與 UI 版面配置分開處理？

## 相關內容

- [頻道概覽](/zh-TW/channels)
- [訊息呈現](/zh-TW/plugins/message-presentation)
