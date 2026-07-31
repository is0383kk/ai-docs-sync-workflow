---
read_when: Browser control fails on Linux, especially with snap Chromium
summary: 修正 Linux 上 OpenClaw 瀏覽器控制的 Chrome/Brave/Edge/Chromium CDP 啟動問題
title: 瀏覽器疑難排解
x-i18n:
    generated_at: "2026-07-26T08:49:34Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 8e5db2da2d43129862f0c005213df828f6eae81f5561e57d41795ea90787822a
    source_path: tools/browser-linux-troubleshooting.md
    workflow: 16
---

## 問題：無法在連接埠 18800 上啟動 Chrome CDP

```json
{ "error": "錯誤：無法在連接埠 18800 上為設定檔 \"openclaw\" 啟動 Chrome CDP。" }
```

### 根本原因

在 Ubuntu 和大多數 Linux 發行版上，`apt install chromium` 安裝的是 snap
包裝程式，而不是真正的瀏覽器：

```text
注意，選擇 'chromium-browser' 而非 'chromium'
chromium-browser 已是最新版本 (2:1snap1-0ubuntu2)。
```

Snap 的 AppArmor 限制會干擾 OpenClaw 產生及監控
瀏覽器程序的方式。

其他常見的 Linux 啟動失敗原因：

- `The profile appears to be in use by another Chromium process`：受管理設定檔目錄中有過期的
  `Singleton*` 鎖定檔案。當鎖定指向已終止或
  不同主機的程序時，OpenClaw 會移除這些鎖定並重試一次。
- `Missing X server or $DISPLAY`：在沒有桌面工作階段的主機上，明確要求啟動可見的瀏覽器。
  在 Linux 上，若 `DISPLAY` 和 `WAYLAND_DISPLAY` 均未設定，本機受管理設定檔會退回
  無頭模式。若你設定了 `OPENCLAW_BROWSER_HEADLESS=0`、`browser.headless: false` 或
  `browser.profiles.<name>.headless: false`，請移除該有頭模式覆寫、設定
  `OPENCLAW_BROWSER_HEADLESS=1`、啟動 `Xvfb`、執行
  `openclaw browser start --headless` 進行一次性受管理啟動，或在
  真正的桌面工作階段中執行 OpenClaw。

### 解決方案 1：安裝 Google Chrome（建議）

```bash
wget https://dl.google.com/linux/direct/google-chrome-stable_current_amd64.deb
sudo dpkg -i google-chrome-stable_current_amd64.deb
sudo apt --fix-broken install -y  # 如果發生相依性錯誤
```

更新 `~/.openclaw/openclaw.json`：

```json
{
  "browser": {
    "enabled": true,
    "executablePath": "/usr/bin/google-chrome-stable",
    "headless": true,
    "noSandbox": true
  }
}
```

### 解決方案 2：以僅附加模式使用 snap Chromium

如果你必須保留 snap Chromium，請將 OpenClaw 設定為附加至
手動啟動的瀏覽器，而非自行啟動瀏覽器：

```json
{
  "browser": {
    "enabled": true,
    "attachOnly": true,
    "headless": true,
    "noSandbox": true
  }
}
```

手動啟動 Chromium：

```bash
chromium-browser --headless --no-sandbox --disable-gpu \
  --remote-debugging-port=18800 \
  --user-data-dir=$HOME/.openclaw/browser/openclaw/user-data \
  about:blank &
```

也可以選擇使用 systemd 使用者服務自動啟動：

```ini
# ~/.config/systemd/user/openclaw-browser.service
[Unit]
Description=OpenClaw 瀏覽器 (Chrome CDP)
After=network.target

[Service]
ExecStart=/snap/bin/chromium --headless --no-sandbox --disable-gpu --remote-debugging-port=18800 --user-data-dir=%h/.openclaw/browser/openclaw/user-data about:blank
Restart=on-failure
RestartSec=5

[Install]
WantedBy=default.target
```

```bash
systemctl --user enable --now openclaw-browser.service
```

### 驗證瀏覽器是否正常運作

```bash
curl -s http://127.0.0.1:18791/ | jq '{running, pid, chosenBrowser}'
curl -s -X POST http://127.0.0.1:18791/start
curl -s http://127.0.0.1:18791/tabs
```

### 設定參考

| 選項                      | 說明                                                          | 預設值                                                            |
| --------------------------- | -------------------------------------------------------------------- | ------------------------------------------------------------------ |
| `browser.enabled`           | 啟用瀏覽器控制                                               | `true`                                                             |
| `browser.executablePath`    | Chromium 架構瀏覽器二進位檔的路徑（Chrome/Brave/Edge/Chromium） | 自動偵測（若作業系統預設瀏覽器採用 Chromium 架構，則優先使用） |
| `browser.headless`          | 在沒有圖形使用者介面的情況下執行                                                      | `false`                                                            |
| `OPENCLAW_BROWSER_HEADLESS` | 針對本機受管理瀏覽器無頭模式的個別程序覆寫         | 未設定                                                              |
| `browser.noSandbox`         | 新增 `--no-sandbox` 旗標（部分 Linux 設定需要）               | `false`                                                            |
| `browser.attachOnly`        | 不啟動瀏覽器；僅附加至現有瀏覽器              | `false`                                                            |

在 Raspberry Pi、較舊的 VPS 主機或速度較慢的儲存裝置上，若 Chrome 公開其 CDP HTTP
端點或就緒所需的時間超過受管理瀏覽器允許的期限，請使用以 `attachOnly`
手動啟動的瀏覽器。

### 問題：找不到 profile="user" 的 Chrome 分頁

你正在使用 `user`（`existing-session` / Chrome MCP）設定檔，但沒有
可供附加的已開啟分頁。

修正選項：

1. 改用受管理瀏覽器：
   `openclaw browser --browser-profile openclaw start`（或設定
   `browser.defaultProfile: "openclaw"`）。
2. 讓本機 Chrome 保持執行且至少開啟一個分頁，然後使用
   `--browser-profile user` 重試。

注意事項：

- `user` 僅限主機使用。在 Linux 伺服器、容器或遠端主機上，請優先使用
  CDP 設定檔。
- `user` 和其他 `existing-session` 設定檔受限於目前的 Chrome MCP
  限制：僅支援以參照為基礎的動作、每次上傳一個檔案、不支援對話方塊 `timeoutMs`
  覆寫、不支援 `wait --load networkidle`，也不支援 `responsebody`、PDF 匯出、
  下載攔截或批次動作。
- 本機 `openclaw` 驅動程式設定檔會自動指派 `cdpPort`/`cdpUrl`；僅針對
  遠端 CDP 手動設定這些值。
- 遠端 CDP 設定檔接受 `http://`、`https://`、`ws://` 和 `wss://`。
  使用 HTTP(S) 進行 `/json/version` 探索；若瀏覽器
  服務提供直接的 DevTools 通訊端 URL，則使用 WS(S)。

## 相關內容

- [瀏覽器](/zh-TW/tools/browser)
- [瀏覽器登入](/zh-TW/tools/browser-login)
- [瀏覽器 WSL2 疑難排解](/zh-TW/tools/browser-wsl2-windows-remote-cdp-troubleshooting)
