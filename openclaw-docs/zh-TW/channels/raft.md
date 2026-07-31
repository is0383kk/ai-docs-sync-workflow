---
read_when:
    - 你想要將 OpenClaw 連接至 Raft 工作區
    - 你正在設定 Raft 外部代理程式
    - 你正在偵錯 Raft 喚醒傳遞
sidebarTitle: Raft
summary: 透過 Raft 命令列介面喚醒橋接支援 Raft 外部代理程式
title: Raft
x-i18n:
    generated_at: "2026-07-26T08:16:52Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 454d92d764a4ec3b0ec52467cba254dcad795870e04d1d32d4cf65d8b451a0de
    source_path: channels/raft.md
    workflow: 16
---

Raft 透過本機 Raft 命令列介面，將 OpenClaw 代理連接至 Raft External Agent。Raft 會將經過驗證的喚醒提示傳送至閘道；接著代理會使用 Raft 命令列介面檢查及傳送訊息。僅支援直接聊天（不支援群組）。

## 安裝

Raft 是官方外部外掛。請將它安裝在閘道主機上：

```bash
openclaw plugins install @openclaw/raft
openclaw gateway restart
```

詳細資訊：[外掛](/zh-TW/tools/plugin)

## 先決條件

- 具有 External Agent 的 Raft 工作區。
- Raft 命令列介面已安裝在 OpenClaw 閘道所在的同一部主機上，並位於服務的
  `PATH` 中。
- 已登入且與該 External Agent 關聯的 Raft 命令列介面設定檔。

此外掛不會儲存 Raft 認證資訊；Raft 命令列介面會將該驗證資訊保留在自己的設定檔中。

## 設定

在設定中指定設定檔：

```json5
{
  channels: {
    raft: {
      enabled: true,
      profile: "openclaw",
    },
  },
}
```

若為預設帳號，也可以改為在閘道環境中設定 `RAFT_PROFILE`：

```bash
RAFT_PROFILE=openclaw
```

當一個閘道連接至多個 Raft External Agent 時，請使用具名帳號：

```json5
{
  channels: {
    raft: {
      accounts: {
        support: {
          profile: "support-agent",
        },
        engineering: {
          profile: "engineering-agent",
        },
      },
    },
  },
}
```

互動式設定會記錄相同的設定檔：

```bash
openclaw channels add --channel raft
```

## 運作方式

閘道啟動時，此外掛會：

1. 在臨時連接埠上開啟僅限迴路介面的 HTTP 喚醒端點。
2. 使用該端點及每個處理程序專屬的權杖啟動 `raft --profile <profile> agent bridge`。
3. 僅接受來自本機橋接器、經過驗證、不含內容且具有重播識別資訊的喚醒提示。
4. 要求每個喚醒承載資料都必須包含 `eventId`、`attemptId`、`messageId`、`delivery_id`、
   `wake_id` 或 `id` 其中之一。
5. 依橋接器事件 ID，對重試的喚醒傳遞進行 24 小時去重，且跨閘道重新啟動仍然有效。
6. 為目前的橋接器傳回穩定的執行階段工作階段，並為 Raft 命令列介面通訊協定傳回空的活動排空批次。
7. 每次接受喚醒時，啟動一個依序執行的 OpenClaw 代理回合。

橋接器負責 Raft 傳遞重試及重新連線。OpenClaw 回合只會收到喚醒通知，不會收到複製的 Raft 訊息本文。它會使用命令列介面讀取待處理訊息並傳送回覆：

```bash
raft --profile openclaw message check
raft --profile openclaw message send
```

<Note>
Raft 不是推播訊息傳輸機制。OpenClaw 不會自動透過橋接器傳回模型的最終文字，因此代理處理喚醒後必須使用 Raft 命令列介面。
</Note>

## 驗證

檢查 OpenClaw 是否能找到命令列介面，且已設定設定檔：

```bash
openclaw channels status --probe
openclaw plugins inspect raft --runtime --json
```

接著傳送訊息給 Raft External Agent。閘道記錄應先顯示 Raft 橋接器啟動，然後顯示傳入的喚醒。代理應使用已設定的 Raft 設定檔檢查待處理訊息。

## 疑難排解

<AccordionGroup>
  <Accordion title="缺少 Raft 命令列介面">
    請在閘道主機上安裝 Raft 命令列介面，並確保服務的 `PATH` 中可使用 `raft`。使用 `raft --help` 驗證，然後重新啟動閘道。
  </Accordion>
  <Accordion title="橋接器立即結束">
    確認已設定的設定檔處於登入狀態，且屬於預期的 Raft External Agent。直接執行 `raft --profile <profile> agent bridge`
    以查看命令列介面診斷資訊。
  </Accordion>
  <Accordion title="收到喚醒，但未傳送 Raft 回覆">
    若代理未叫用 Raft 命令列介面，這是預期行為。喚醒橋接器不會攜帶訊息本文，也不會自動傳送最終回覆。請檢查代理的工具政策，並確保它能執行 `raft --profile <profile>
    message check` 和 `message send`。
  </Accordion>
</AccordionGroup>

## 參考資料

- [Raft](https://raft.build/)
- [Raft 文件](https://docs.raft.build/welcome/)
- [Hermes Raft 整合](https://hermes-agent.nousresearch.com/docs/user-guide/messaging/raft)
