---
read_when:
    - 你想要將 OpenClaw 連線至微信或微信
    - 你正在安裝或疑難排解 openclaw-weixin 頻道外掛
    - 你需要瞭解外部頻道外掛如何在閘道旁執行
summary: 透過外部 openclaw-weixin 外掛設定微信頻道
title: 微信
x-i18n:
    generated_at: "2026-07-26T07:34:02Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 98faf95f9fb76deedb7df9adf3092083722a77bdd793de98c41a6f715cc0d14a
    source_path: channels/wechat.md
    workflow: 16
---

OpenClaw 透過騰訊的外部
`@tencent-weixin/openclaw-weixin` 頻道外掛連線至微信。

狀態：由騰訊微信團隊維護的外部外掛。支援私人聊天和
媒體。外掛能力中繼資料未宣告支援群組聊天
（僅宣告私人聊天）。

## 命名

- **微信**是這些文件中面向使用者的名稱。
- **微信**是騰訊套件和外掛 ID 使用的名稱。
- `openclaw-weixin` 是 OpenClaw 頻道 ID（`weixin` 和 `wechat` 可作為別名）。
- `@tencent-weixin/openclaw-weixin` 是 npm 套件。

請在命令列介面命令和設定路徑中使用 `openclaw-weixin`。

## 運作方式

微信程式碼不在 OpenClaw 核心儲存庫中。OpenClaw 提供
通用頻道外掛合約，而外部外掛提供
微信專用執行階段：

1. `openclaw plugins install` 會安裝 `@tencent-weixin/openclaw-weixin`。
2. 閘道會探索外掛資訊清單並載入外掛進入點。
3. 外掛會註冊頻道 ID `openclaw-weixin`。
4. `openclaw channels login --channel openclaw-weixin` 會啟動 QR Code 登入。
5. 外掛會將帳戶認證資訊儲存在 OpenClaw 狀態目錄下
   （預設為 `~/.openclaw`）。
6. 閘道啟動時，外掛會為每個
   已設定的帳戶啟動其微信監控程式。
7. 傳入的微信訊息會透過頻道合約正規化、路由至
   所選的 OpenClaw 代理程式，並透過外掛的傳出路徑傳回。

這種分離方式很重要：OpenClaw 核心不依賴特定頻道。微信登入、
騰訊 iLink API 呼叫、媒體上傳／下載、情境權杖和帳戶
監控皆由外部外掛負責。

## 安裝

快速安裝：

```bash
npx -y @tencent-weixin/openclaw-weixin-cli install
```

手動安裝：

```bash
openclaw plugins install "@tencent-weixin/openclaw-weixin"
openclaw config set plugins.entries.openclaw-weixin.enabled true
```

安裝後重新啟動閘道：

```bash
openclaw gateway restart
```

## 登入

請在執行閘道的同一台機器上執行 QR Code 登入：

```bash
openclaw channels login --channel openclaw-weixin
```

使用手機上的微信掃描 QR Code 並確認登入。掃描成功後，外掛會將
帳戶權杖儲存在本機。

若要新增另一個微信帳戶，請再次執行相同的登入命令。使用多個
帳戶時，請依帳戶、頻道和傳送者隔離私人訊息工作階段：

```bash
openclaw config set session.dmScope per-account-channel-peer
```

## 存取控制

私人訊息使用頻道外掛的一般 OpenClaw 配對與允許清單模型。

核准新的傳送者：

```bash
openclaw pairing list openclaw-weixin
openclaw pairing approve openclaw-weixin <CODE>
```

如需完整的存取控制模型，請參閱[配對](/zh-TW/channels/pairing)。

## 相容性

外掛會在啟動時檢查主機的 OpenClaw 版本。

| 外掛版本線 | OpenClaw 版本                                                | npm 標籤  |
| ----------- | --------------------------------------------------------------- | -------- |
| `2.x`       | `>=2026.5.12`（目前為 2.4.6；早期 2.x 接受 `>=2026.3.22`） | `latest` |
| `1.x`       | `>=2026.1.0 <2026.3.22`                                         | `legacy` |

若外掛回報你的 OpenClaw 版本過舊，請更新
OpenClaw，或安裝舊版外掛版本線：

```bash
openclaw plugins install @tencent-weixin/openclaw-weixin@legacy
```

## Sidecar 程序

微信外掛在監控騰訊 iLink API 時，可於閘道旁執行輔助工作。
在 issue #68451 中，該輔助程式路徑暴露了 OpenClaw
通用過時閘道清理機制中的錯誤：子程序可能嘗試清理父
閘道程序，導致 systemd 等程序管理工具下發生重新啟動迴圈。

目前的 OpenClaw 啟動清理會排除目前程序及其祖先程序，
因此頻道輔助程式無法終止啟動它的閘道。此修正
具通用性；它不是核心中的微信專用路徑。

## 疑難排解

檢查安裝和狀態：

```bash
openclaw plugins list
openclaw channels status --probe
openclaw --version
```

如果頻道顯示為已安裝但未連線，請確認外掛
已啟用，然後重新啟動：

```bash
openclaw config set plugins.entries.openclaw-weixin.enabled true
openclaw gateway restart
```

如果啟用微信後閘道反覆重新啟動，請同時更新 OpenClaw 和
外掛：

```bash
npm view @tencent-weixin/openclaw-weixin version
openclaw plugins install "@tencent-weixin/openclaw-weixin" --force
openclaw gateway restart
```

如果啟動時回報已安裝的外掛套件 `requires compiled runtime
output for TypeScript entry`，表示該 npm 套件發佈時未包含 OpenClaw 所需的已編譯
JavaScript 執行階段檔案。請在外掛
發佈者推出修正後的套件後更新／重新安裝，或暫時停用／解除安裝外掛。

暫時停用：

```bash
openclaw config set plugins.entries.openclaw-weixin.enabled false
openclaw gateway restart
```

## 相關文件

- 頻道概覽：[聊天頻道](/zh-TW/channels)
- 配對：[配對](/zh-TW/channels/pairing)
- 頻道路由：[頻道路由](/zh-TW/channels/channel-routing)
- 外掛架構：[外掛架構](/zh-TW/plugins/architecture)
- 頻道外掛 SDK：[頻道外掛 SDK](/zh-TW/plugins/sdk-channel-plugins)
- 外部套件：[@tencent-weixin/openclaw-weixin](https://www.npmjs.com/package/@tencent-weixin/openclaw-weixin)
