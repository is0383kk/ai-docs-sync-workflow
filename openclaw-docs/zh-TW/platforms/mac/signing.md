---
read_when:
    - 建置或簽署 Mac 偵錯版本
summary: 由封裝指令碼產生的 macOS 偵錯組建簽署步驟
title: macOS 簽署
x-i18n:
    generated_at: "2026-07-26T07:24:47Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 406211dadc9293cf7983e75ae7dd98234f9088351234cf06c33df2f63d1b9b97
    source_path: platforms/mac/signing.md
    workflow: 16
---

# Mac 簽署（偵錯組建）

[`scripts/package-mac-app.sh`](https://github.com/openclaw/openclaw/blob/main/scripts/package-mac-app.sh) 會將應用程式組建並封裝至固定路徑（`dist/OpenClaw.app`），然後呼叫 [`scripts/codesign-mac-app.sh`](https://github.com/openclaw/openclaw/blob/main/scripts/codesign-mac-app.sh) 進行簽署。TCC 權限會綁定套件識別碼和程式碼簽章；在每次重新組建時維持兩者穩定（並將應用程式保留在固定路徑），可防止 macOS 遺忘 TCC 授權（通知、輔助使用、螢幕錄製、麥克風、語音）。

- 偵錯套件識別碼預設為 `ai.openclaw.mac.debug`（可使用 `BUNDLE_ID=...` 覆寫）。
- Node：`>=22.22.3 <23`、`>=24.15.0 <25` 或 `>=25.9.0`（儲存庫 `package.json` `engines`）。封裝程式也會組建控制介面（`pnpm ui:build`）。
- 預設需要真實的簽署身分；如果找不到簽署身分，且未設定 `ALLOW_ADHOC_SIGNING`，程式碼簽署指令碼會以錯誤結束。臨時簽署（`SIGN_IDENTITY="-"`）必須明確選擇啟用，且無法在重新組建後保留 TCC 權限。請參閱 [macOS 權限](/zh-TW/platforms/mac/permissions)。
- 從環境讀取 `SIGN_IDENTITY`（例如 `export SIGN_IDENTITY="Apple Development: Your Name (TEAMID)"` 或 Developer ID Application 憑證）。若未提供，`codesign-mac-app.sh` 會依下列順序自動選取身分：Developer ID Application、Apple Distribution、Apple Development，最後是找到的第一個有效程式碼簽署身分。
- `CODESIGN_TIMESTAMP=auto`（預設）僅為 Developer ID Application 簽章啟用受信任的時間戳記。設定 `on`/`off` 可強制啟用或停用。
- 在 Info.plist 中寫入 `OpenClawBuildTimestamp`（ISO8601 UTC）和 `OpenClawGitCommit`（短雜湊；若無法取得則為 `unknown`），讓「關於」分頁能顯示組建、Git 和偵錯／發行通道。
- 簽署後執行 Team ID 稽核；如果套件內任何 Mach-O 的 Team ID 不同，則會失敗。設定 `SKIP_TEAM_ID_CHECK=1` 可略過此檢查。

## 使用方式

```bash
# 從儲存庫根目錄執行
scripts/package-mac-app.sh                                                      # 自動選取身分；若找不到則發生錯誤
SIGN_IDENTITY="Developer ID Application: Your Name" scripts/package-mac-app.sh   # 真實憑證
ALLOW_ADHOC_SIGNING=1 scripts/package-mac-app.sh                                 # 臨時簽署（權限不會保留）
SIGN_IDENTITY="-" scripts/package-mac-app.sh                                     # 明確使用臨時簽署（注意事項相同）
DISABLE_LIBRARY_VALIDATION=1 scripts/package-mac-app.sh                          # 僅供開發使用的 Sparkle Team ID 不相符因應措施
```

### 臨時簽署注意事項

`SIGN_IDENTITY="-"` 會停用強化執行階段（`--options runtime`），以防應用程式載入未使用相同 Team ID 的內嵌框架（例如 Sparkle）時當機。臨時簽章也會破壞 TCC 權限的持續保留；復原步驟請參閱 [macOS 權限](/zh-TW/platforms/mac/permissions)。

## 「關於」的組建中繼資料

「關於」分頁會從 Info.plist 讀取 `OpenClawBuildTimestamp` 和 `OpenClawGitCommit`，以顯示版本、組建日期、Git 提交，以及此組建是否為 DEBUG（透過 `#if DEBUG`）。程式碼變更後，請重新執行封裝程式以更新這些值。

## 相關內容

- [macOS 應用程式](/zh-TW/platforms/macos)
- [macOS 權限](/zh-TW/platforms/mac/permissions)
