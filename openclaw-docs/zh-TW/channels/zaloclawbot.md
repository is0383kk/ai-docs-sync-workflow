---
read_when:
    - 你想要一個使用 QR Code 登入的個人 Zalo 助理機器人
    - 你正在安裝或疑難排解 openclaw-zaloclawbot 頻道外掛
summary: 透過外部 openclaw-zaloclawbot 外掛設定 Zalo ClawBot 頻道
title: Zalo ClawBot
x-i18n:
    generated_at: "2026-07-26T08:17:33Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 76c9f79d114856b86026a5e4b98a43f451b0d3f16dd41a67e9226da4f8b37b33
    source_path: channels/zaloclawbot.md
    workflow: 16
---

OpenClaw 透過目錄中列出的外部 `@zalo-platforms/openclaw-zaloclawbot` 外掛連線至 Zalo ClawBot。登入使用 Zalo Mini App QR code；設定中的外掛 ID 為 `openclaw-zaloclawbot`。

## 相容性

| 外掛版本 | OpenClaw 版本 | npm dist-tag | 狀態          |
| -------------- | ---------------- | ------------ | ------------- |
| 0.1.4          | >=2026.4.10      | `latest`     | 啟用中 / Beta |

## 必要條件

- Node.js >= 22
- 已安裝 [OpenClaw](https://docs.openclaw.ai/install)（可使用 `openclaw` 命令列介面）
- 行動裝置上的 Zalo 帳號，用於掃描登入 QR code

## 使用 onboard 安裝（建議）

```bash
openclaw onboard
```

從頻道選單選擇 **Zalo ClawBot**。精靈會從官方目錄安裝外掛（並驗證完整性）、在終端機中顯示登入 QR code，並在你使用 Zalo 應用程式掃描後完成頻道設定。

## 手動安裝

若要將頻道新增至已完成 onboard 的閘道：

### 1. 安裝外掛

```bash
openclaw plugins install "@zalo-platforms/openclaw-zaloclawbot@0.1.4"
```

請使用確切鎖定的版本，以便 OpenClaw 在安裝期間根據目錄完整性雜湊驗證套件。

### 2. 在設定中啟用外掛

```bash
openclaw config set plugins.entries.openclaw-zaloclawbot.enabled true
```

### 3. 產生 QR code 並登入

```bash
openclaw channels login --channel openclaw-zaloclawbot
```

使用 Zalo 行動應用程式掃描終端機顯示的 QR code，在 Zalo Mini App 內接受使用條款，並授權工作階段。

### 4. 重新啟動閘道

```bash
openclaw gateway restart
```

## 運作方式

標準 Zalo 頻道需要註冊自己的 Zalo Official Account（OA）並設定靜態開發者認證資訊；與其不同的是，Zalo ClawBot 是共用官方基礎架構上的**擁有者綁定個人助理**：

1. **新手設定：** QR code 會導向 Zalo Mini App，該應用程式會在共用官方 OA 下繫結新佈建的私人機器人，並將其直接連結至你的 Zalo 使用者 ID。
2. **擁有者綁定隱私：**機器人只會與其擁有者通訊。其他使用者的訊息會在平台層級遭到捨棄。
3. **官方 API 路徑：**此外掛使用 Zalo Bot Platform API，而非瀏覽器或 Web 工作階段自動化。

## 底層運作

此外掛透過持續的長輪詢迴圈（`getUpdates`）與 Zalo 通訊。針對本機桌面／終端機閘道執行，網路鉤子預設為停用。訊息會在用戶端處理，並對應至你的本機代理程式執行環境。

此外掛會在 OpenClaw 狀態目錄下管理機器人認證資訊。請將該目錄視為敏感資料，並套用與其餘 OpenClaw 狀態相同的存取控制與備份政策。

此外掛的執行環境完全位於外部 `@zalo-platforms/openclaw-zaloclawbot` 套件中；以下安裝／設定以外的行為細節由外掛維護者提供，尚未根據 OpenClaw 核心原始碼驗證。

## 疑難排解

- **QR 登入逾時：**基於安全考量，登入權杖（`zbsk`）會在 5 分鐘後到期。如果 QR code 在你掃描前到期，請重新執行登入命令以產生新的 QR code。
- **閘道載入失敗：**確認你的 OpenClaw 主機版本為 `2026.4.10` 或更高版本。舊版不支援此 ID 所需的外部 npm 外掛安裝帳本。

## 相關內容

- [頻道概覽](/zh-TW/channels) - 所有支援的頻道
- [Zalo](/zh-TW/channels/zalo) - 內建的 Zalo Bot Creator / Marketplace 頻道
- [配對](/zh-TW/channels/pairing) - 私訊驗證與配對流程
- [外掛](/zh-TW/tools/plugin) - 安裝及管理外掛
