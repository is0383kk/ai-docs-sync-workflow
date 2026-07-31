---
read_when:
    - 擷取 macOS 日誌或調查私密資料記錄問題
    - 偵錯語音喚醒／工作階段生命週期問題
summary: OpenClaw 記錄：滾動式診斷檔案記錄 + 統一記錄隱私權旗標
title: macOS 記錄功能
x-i18n:
    generated_at: "2026-07-26T08:01:06Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: ef0fd91bd7fc0a8b5f598cfe8f5de551795a4badd0f6634c5bcbd4f3916bfc64
    source_path: platforms/mac/logging.md
    workflow: 16
---

# 記錄（macOS）

## 輪替診斷檔案記錄（除錯窗格）

macOS 應用程式透過 swift-log 寫入記錄（預設使用統一記錄），也能寫入會輪替的本機檔案記錄，以便持久保留 (`DiagnosticsFileLog`)。

- 啟用：**Debug pane -> Logs -> App logging -> "Write rolling diagnostics log (JSONL)"**（預設關閉）。
- 詳細程度：**Debug pane -> Logs -> App logging -> Verbosity** 選擇器。
- 位置：`~/Library/Logs/OpenClaw/diagnostics.jsonl`。
- 輪替：達到 5 MB 時輪替；最多保留 5 份備份，後綴為 `.1`...`.5`（最舊的會被刪除）。
- 清除：**Debug pane -> Logs -> App logging -> "Clear"** 會刪除目前使用中的檔案及所有備份。

請將此檔案視為敏感資料；未經檢閱請勿分享。

## macOS 統一記錄中的私人資料

除非子系統選擇啟用 `privacy -off`，否則統一記錄會遮蔽大多數承載內容。此設定由 `/Library/Preferences/Logging/Subsystems/` 中以子系統名稱為鍵的 plist 控制。只有新的記錄項目會套用此旗標，因此請在重現問題前先啟用。背景資訊：[macOS 記錄隱私機制解析](https://steipete.me/posts/2025/logging-privacy-shenanigans)。

## 為 OpenClaw 啟用（`ai.openclaw`）

先將 plist 寫入暫存檔，再以 root 身分進行不可分割的安裝：

```bash
cat <<'EOF' >/tmp/ai.openclaw.plist
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>DEFAULT-OPTIONS</key>
    <dict>
        <key>Enable-Private-Data</key>
        <true/>
    </dict>
</dict>
</plist>
EOF
sudo install -m 644 -o root -g wheel /tmp/ai.openclaw.plist /Library/Preferences/Logging/Subsystems/ai.openclaw.plist
```

不需要重新啟動；logd 很快就會讀取此檔案，但只有新的記錄行會包含私人承載內容。使用 `./scripts/clawlog.sh --category WebChat --last 5m` 檢視更詳盡的輸出（`--last`/`-l` 設定時間範圍，預設為 `5m`；`--category`/`-c` 依類別篩選）。

## 除錯後停用

- 移除覆寫設定：`sudo rm /Library/Preferences/Logging/Subsystems/ai.openclaw.plist`。
- 也可執行 `sudo log config --reload`，強制 logd 立即捨棄覆寫設定。
- 此介面可能包含電話號碼和訊息本文；只有在確實需要使用時才保留此 plist。

## 相關內容

- [macOS 應用程式](/zh-TW/platforms/macos)
- [閘道記錄](/zh-TW/gateway/logging)
