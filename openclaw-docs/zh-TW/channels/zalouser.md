---
read_when:
    - 為 OpenClaw 設定 Zalo Personal
    - 偵錯 Zalo Personal 登入或訊息流程
summary: 透過原生 zca-js（QR 登入）支援 Zalo Personal 帳號、功能與設定
title: Zalo Personal
x-i18n:
    generated_at: "2026-07-26T08:26:29Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 09cecad1a9a5b34b932c5e68e2b3164b360fb6af1dcd2fd5b5979d1b2a1bd62b
    source_path: channels/zalouser.md
    workflow: 16
---

狀態：實驗性。此整合透過原生 `zca-js` 在程序內自動操作**個人 Zalo 帳號**，不需要外部命令列介面執行檔。

<Warning>
這是非官方整合，可能導致帳號遭停權或封鎖。使用風險由你自行承擔。
</Warning>

## 安裝

Zalo Personal 是官方外部外掛，並未隨核心一同提供。使用前請先安裝：

```bash
openclaw plugins install @openclaw/zalouser
```

- 鎖定版本：`openclaw plugins install @openclaw/zalouser@<version>`
- 從原始碼簽出版本安裝：`openclaw plugins install ./path/to/local/zalouser-plugin`
- 詳細資訊：[外掛](/zh-TW/tools/plugin)

## 快速設定

1. 安裝外掛（如上所述）。
2. 登入（在閘道機器上使用 QR Code）：
   - `openclaw channels login --channel zalouser`
   - 使用 Zalo 行動應用程式掃描 QR Code。
3. 啟用頻道：

```json5
{
  channels: {
    zalouser: {
      enabled: true,
      dmPolicy: "pairing",
    },
  },
}
```

4. 重新啟動閘道（或完成設定）。
5. 私訊存取預設使用配對；第一次聯絡時請核准配對碼。

## 功能說明

- 完全透過 `zca-js` 程式庫在程序內執行（不需要外部 `zca`/`openzca` 執行檔）。
- 使用原生事件監聽器（`message`、`error`）接收傳入訊息。
- 透過 JS API 直接傳送回覆（文字／媒體／連結）。
- 專為無法使用 Zalo Bot API 的「個人帳號」使用情境所設計。

## 命名

頻道 ID 為 `zalouser`，以明確表示其自動操作的是**個人 Zalo 使用者帳號**（非官方）。`zalo` 保留供未來可能推出的官方 Zalo API 整合使用。

## 尋找 ID（目錄）

```bash
openclaw directory self --channel zalouser
openclaw directory peers list --channel zalouser --query "name"
openclaw directory groups list --channel zalouser --query "work"
```

## 限制

- 傳出文字會依 2000 個字元分段（Zalo 用戶端限制）。
- 不支援串流。
- 已完成處理的傳入訊息 ID 會保留 30 天，每個帳號最多保留最近的 1000 筆項目。

## 傳入訊息持久性

OpenClaw 會在處理每個原始 `zca-js` 訊息回呼之前，先將其儲存。閘道重新啟動後，待處理訊息會從帳號佇列繼續處理，且每個直接聊天或群組的處理會維持循序執行。

`zca-js` 通訊端監聽器不會提供送達確認，也不會在重新連線後自動重播舊訊息。因此，持久性佇列只能防範回呼送達 OpenClaw 後的本機當機空窗；無法復原通訊端從未送達的訊息。重播刪除標記主要用於防範同一個 Zalo 訊息 ID 的回呼重複出現。

## 存取控制（私訊）

`channels.zalouser.dmPolicy`：`pairing | allowlist | open | disabled`（預設：`pairing`）。

`channels.zalouser.allowFrom` 應使用穩定的 Zalo 使用者 ID。它也可以參照靜態傳送者存取群組（`accessGroup:<name>`）。在互動式設定期間，可使用外掛的程序內聯絡人查詢，將輸入的名稱解析為 ID。

如果設定中仍有原始名稱，只有啟用 `channels.zalouser.dangerouslyAllowNameMatching: true` 時，啟動程序才會解析該名稱。若未明確啟用此選項，執行階段的傳送者檢查只會使用 ID，且會忽略原始名稱，不將其用於授權。

核准方式：

- `openclaw pairing list zalouser`
- `openclaw pairing approve zalouser <code>`

## 群組存取（選用）

- 預設：`channels.zalouser.groupPolicy = "allowlist"`（群組必須有明確的允許清單項目）。
- 開放所有群組：`channels.zalouser.groupPolicy = "open"`。
- 封鎖所有群組：`channels.zalouser.groupPolicy = "disabled"`。
- 搭配 `groupPolicy = "allowlist"` 時：
  - `channels.zalouser.groups` 的鍵應為穩定的群組 ID；只有啟用 `channels.zalouser.dangerouslyAllowNameMatching: true` 時，才會在啟動時將名稱解析為 ID。
  - `channels.zalouser.groupAllowFrom` 控制允許群組中的哪些傳送者可以觸發機器人；可使用 `accessGroup:<name>` 參照靜態傳送者存取群組。
- 設定精靈可以提示輸入群組允許清單。
- 群組允許清單預設只依 ID 比對。除非啟用 `channels.zalouser.dangerouslyAllowNameMatching: true`，否則授權時會忽略未解析的名稱。
- `channels.zalouser.dangerouslyAllowNameMatching: true` 是緊急相容模式，會重新啟用可變動的啟動時名稱解析，以及執行階段群組名稱比對。
- 對一般群組訊息而言，`groupAllowFrom` **不會**退回使用 `allowFrom`：若允許清單中的群組將其留空，該群組會對任何傳送者開放。已授權的控制命令（例如 `/new`）是例外；當 `groupAllowFrom` 為空時，命令傳送者檢查會退回使用 `allowFrom`。

範例：

```json5
{
  channels: {
    zalouser: {
      groupPolicy: "allowlist",
      groupAllowFrom: ["1471383327500481391"],
      groups: {
        "123456789": { enabled: true },
        "Work Chat": { enabled: true },
      },
    },
  },
}
```

<Note>
`channels.zalouser.groups.<id>.allow` 是舊版欄位名稱；目前的設定使用 `enabled`。`openclaw doctor --fix` 會自動將 `allow` 遷移至 `enabled`。
</Note>

### 群組提及閘控

- `channels.zalouser.groups.<group>.requireMention` 控制群組回覆是否必須提及機器人。
- 解析順序：群組 ID -> `group:<id>` 別名 -> 群組名稱／slug（以名稱為基礎的候選項目僅在 `dangerouslyAllowNameMatching: true` 時適用）-> `*` -> 預設值（`true`）。
- 同時適用於允許清單群組及開放群組模式。
- 引用機器人訊息會視為隱含提及，並啟用群組處理。
- 已授權的控制命令（例如 `/new`）可以略過提及閘控。
- 如果群組訊息因需要提及而被略過，OpenClaw 會將其儲存為待處理群組記錄，並在下一則處理的群組訊息中納入該訊息。
- 群組記錄限制：依序使用 `channels.zalouser.historyLimit`、`messages.groupChat.historyLimit`，最後退回使用 `50`。

範例：

```json5
{
  channels: {
    zalouser: {
      groupPolicy: "allowlist",
      groups: {
        "*": { enabled: true, requireMention: true },
        "Work Chat": { enabled: true, requireMention: false },
      },
    },
  },
}
```

## 多帳號

帳號會對應至 OpenClaw 狀態中的 `zalouser` 設定檔。範例：

```json5
{
  channels: {
    zalouser: {
      enabled: true,
      defaultAccount: "default",
      accounts: {
        work: { enabled: true, profile: "work" },
      },
    },
  },
}
```

## 環境變數

也可以透過環境變數選取設定檔：

| 變數               | 用途                                                                    |
| ------------------ | -------------------------------------------------------------------------- |
| `ZALOUSER_PROFILE` | 當頻道或帳號設定中未設定 `profile` 時使用的設定檔名稱。 |
| `ZCA_PROFILE`      | 舊版備援，僅在未設定 `ZALOUSER_PROFILE` 時使用。             |

設定檔名稱用於選取 OpenClaw 狀態中已儲存的 Zalo 登入認證資訊。解析順序：

1. 設定中明確指定的 `profile`。
2. `ZALOUSER_PROFILE`。
3. `ZCA_PROFILE`。
4. 非預設帳號使用帳號 ID，預設帳號則使用 `default`。

對於多帳號設定，建議在設定中為每個帳號指定 `profile`，避免單一環境變數導致多個帳號共用同一個登入工作階段。

## 輸入狀態、表情回應與送達確認

- OpenClaw 會在分派回覆前傳送輸入狀態事件（盡力而為）。
- 頻道動作中的 `zalouser` 支援訊息表情回應動作 `react`。
  - 使用 `remove: true` 從訊息移除特定的表情回應 emoji。
  - 表情回應語意：[表情回應](/zh-TW/tools/reactions)
- 對於包含事件中繼資料的傳入訊息，OpenClaw 會傳送已送達及已查看確認（盡力而為）。

## 疑難排解

**登入狀態無法保留：**

- `openclaw channels status --probe`
- 重新登入：`openclaw channels logout --channel zalouser && openclaw channels login --channel zalouser`

**允許清單／群組名稱無法解析：**

- 在 `allowFrom`/`groupAllowFrom` 中使用數字 ID，並在 `groups` 中使用穩定的群組 ID。如果確實需要使用完全相符的好友／群組名稱，請啟用 `channels.zalouser.dangerouslyAllowNameMatching: true`。

**從舊版外部 `zca`/命令列介面式設定升級：**

- 移除任何依賴外部 `zca` 程序的假設；頻道現在完全透過 `zca-js` 在程序內執行，不需要外部命令列介面執行檔。

## 相關內容

- [頻道概覽](/zh-TW/channels) - 所有支援的頻道
- [配對](/zh-TW/channels/pairing) - 私訊驗證與配對流程
- [群組](/zh-TW/channels/groups) - 群組聊天行為與提及閘控
- [頻道路由](/zh-TW/channels/channel-routing) - 訊息的工作階段路由
- [安全性](/zh-TW/gateway/security) - 存取模型與強化措施
