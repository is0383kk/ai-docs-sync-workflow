---
read_when:
    - 你需要登入網站以進行瀏覽器自動化
    - 你想要將更新發佈到 X/Twitter
summary: 瀏覽器自動化與 X/Twitter 發文的手動登入
title: 瀏覽器登入
x-i18n:
    generated_at: "2026-07-26T07:35:31Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: bccd363cf7c9611f4687d50a92f7fb3e2fd1c1d67bb27a80c892f7ac58ae1f8f
    source_path: tools/browser-login.md
    workflow: 16
---

## 手動登入（建議）

當網站要求登入時，請在主機瀏覽器的 `openclaw`
設定檔中手動登入。請勿將你的認證資訊提供給模型：自動登入通常會
觸發反機器人防禦機制，並可能導致帳號遭到鎖定。

在 X/Twitter 及其他對機器人敏感的網站上，無論是讀取（搜尋／討論串）或
發布內容，都請使用主機瀏覽器（手動登入）。沙箱瀏覽器工作階段
更容易觸發機器人偵測。

返回瀏覽器主要文件：[瀏覽器](/zh-TW/tools/browser)。

## 使用哪個 Chrome 設定檔？

OpenClaw 會控制名為 `openclaw` 的專用 Chrome 設定檔（使用橘色色調的
使用者介面），與你的日常瀏覽器設定檔分開。

對於代理程式的瀏覽器工具呼叫：

- 預設選擇：代理程式使用其隔離的 `openclaw` 瀏覽器。
- 僅當現有的已登入工作階段相當重要，且你在電腦前能夠點選／核准任何附加提示時，
  才使用 `profile="user"`。
- 如果你有多個使用者瀏覽器設定檔，請明確指定設定檔，
  而不要猜測。

有兩種方式可存取 `openclaw` 設定檔：

1. 要求代理程式開啟瀏覽器，然後由你自行登入。
2. 透過命令列介面開啟：

```bash
openclaw browser start
openclaw browser open https://x.com
```

若要使用非預設設定檔，請將 `--browser-profile <name>` 放在
子命令之前（預設為 `openclaw`）：

```bash
openclaw browser --browser-profile <name> open https://x.com
```

## 沙箱：允許存取主機瀏覽器

如果代理程式在沙箱中執行，其 `browser` 工具呼叫預設會使用沙箱
瀏覽器，而非主機瀏覽器。若要讓代理程式改為以主機瀏覽器為目標：

```json5
{
  agents: {
    defaults: {
      sandbox: {
        mode: "non-main",
        browser: {
          allowHostControl: true,
        },
      },
    },
  },
}
```

命令列介面呼叫一律以主機瀏覽器為目標，絕不使用沙箱，因此無論此設定為何，你都可以
自行開啟主機瀏覽器：

```bash
openclaw browser --browser-profile openclaw open https://x.com
```

設定 `sandbox.browser.allowHostControl: true` 後，代理程式的 `browser`
工具呼叫也能以主機為目標。或者，也可以為發布更新的
代理程式停用沙箱。

## 相關內容

- [瀏覽器](/zh-TW/tools/browser)
- [瀏覽器 Linux 疑難排解](/zh-TW/tools/browser-linux-troubleshooting)
- [瀏覽器 WSL2 疑難排解](/zh-TW/tools/browser-wsl2-windows-remote-cdp-troubleshooting)
