---
read_when:
    - 從 GitHub 或本機執行 Mantis Slack 桌面版 QA
    - 偵錯緩慢的 Mantis Slack 桌面版執行作業
    - 選擇來源、預先水合或暖租約模式
    - 將螢幕截圖與影片證據發佈至 PR
summary: Mantis Slack 桌面版 QA 的操作手冊：GitHub 分派、本機命令列介面、預熱的 VNC 租用環境、載入模式、時間解讀、成品與失敗處理。
title: Mantis Slack 桌面版操作手冊
x-i18n:
    generated_at: "2026-07-26T08:21:32Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: b3e956d99fc43a7b6fe65e2e820812b0e0e8b9e32badd25be27c74d302ab30dc
    source_path: concepts/mantis-slack-desktop-runbook.md
    workflow: 16
---

Mantis Slack 桌面版 QA 是針對 Slack 類型錯誤的真實 UI 測試路徑，適用於需要
Linux 桌面、VNC 救援、Slack Web、真實 OpenClaw 閘道、螢幕截圖、
影片及 PR 證據留言的情況。當單元測試或無頭模式
Slack 即時測試路徑無法證明錯誤時，請使用此路徑。

## 儲存模型

Mantis 使用三個儲存層：

- **供應商映像檔** - 由 Crabbox 擁有，儲存在雲端供應商帳戶中。
  包含機器功能（Chrome/Chromium、ffmpeg、scrot、
  Node/corepack/pnpm、原生建置工具）及空白快取目錄。
- **暖租用狀態** - 由目前的操作員工作階段擁有。在租用存續期間，可包含
  已登入的瀏覽器設定檔、`/var/cache/crabbox/pnpm`，以及準備就緒的原始碼
  簽出內容。
- **Mantis 成品** - 由 OpenClaw 執行作業擁有。位於
  `.artifacts/qa-e2e/mantis/...` 下；GitHub Actions 會上傳這些成品，而 Mantis
  GitHub App 會在 PR 上留言提供行內證據。

絕不可將密鑰、瀏覽器 Cookie、Slack 登入狀態、儲存庫簽出內容、
`node_modules` 或 `dist/` 烘焙至供應商映像檔中。

## GitHub 分派

從 `main` 執行工作流程：

```bash
gh workflow run mantis-slack-desktop-smoke.yml \
  --ref main \
  -f candidate_ref=<trusted-ref-or-sha> \
  -f pr_number=<pr-number> \
  -f scenario_id=slack-canary \
  -f crabbox_provider=aws \
  -f keep_vm=false \
  -f hydrate_mode=source
```

由於工作流程使用即時認證資訊，`candidate_ref` 受到限制：它
必須解析為目前 `main` 的祖先、發行標籤，或
`openclaw/openclaw` 中開啟中 PR 的 head。

工作流程會產生：

- 已上傳的成品 `mantis-slack-desktop-smoke-<run-id>-<attempt>`
- 由 Mantis GitHub App 發布的行內 PR 留言
- `slack-desktop-smoke.png`、`slack-desktop-smoke.mp4`
- `slack-desktop-smoke-preview.gif`、`slack-desktop-smoke-change.mp4`
- `mantis-slack-desktop-smoke-summary.json`、`mantis-slack-desktop-smoke-report.md`
- 遠端日誌：`slack-desktop-command.log`、`openclaw-gateway.log`、`chrome.log`、`ffmpeg.log`

PR 留言會透過隱藏的 `<!-- mantis-slack-desktop-smoke -->` 標記就地更新。

## 本機命令列介面

冷啟動原始碼證明：

```bash
pnpm openclaw qa mantis slack-desktop-smoke \
  --provider aws \
  --class standard \
  --gateway-setup \
  --credential-source convex \
  --credential-role maintainer \
  --provider-mode live-frontier \
  --model openai/gpt-5.4 \
  --alt-model openai/gpt-5.4 \
  --scenario slack-canary \
  --hydrate-mode source
```

保留 VM 以進行 VNC 救援：

```bash
pnpm openclaw qa mantis slack-desktop-smoke \
  --provider aws \
  --class standard \
  --gateway-setup \
  --scenario slack-canary \
  --keep-lease
```

開啟 VNC：

```bash
crabbox vnc --provider aws --id <cbx_id> --open
```

重複使用暖租用：

```bash
pnpm openclaw qa mantis slack-desktop-smoke \
  --provider aws \
  --lease-id <cbx_id-or-slug> \
  --gateway-setup \
  --scenario slack-canary \
  --hydrate-mode source
```

僅在重複使用的遠端工作區已具有
`node_modules` 及已建置的 `dist/` 時，才能使用 `--hydrate-mode prehydrated`；否則 Mantis 會以封閉方式失敗。

證明原生 Slack 核准 UI：

```bash
pnpm openclaw qa mantis slack-desktop-smoke \
  --provider aws \
  --class standard \
  --approval-checkpoints \
  --credential-source convex \
  --credential-role maintainer \
  --hydrate-mode source
```

`--approval-checkpoints` 與 `--gateway-setup` 互斥。除非你傳入明確的核准檢查點
`--scenario`，否則它會執行選擇加入的 `slack-approval-exec-native` 和 `slack-approval-plugin-native`
情境；其他 Slack 情境會在 VM 啟動前遭到拒絕。Slack QA 執行器會根據它觀察到的真實 Slack API 訊息，
寫入每個檢查點 JSON 檔案，接著遠端監看器會將該訊息轉譯至
`approval-checkpoints/<scenario>-pending.png` 和
`approval-checkpoints/<scenario>-resolved.png`。如果任何
檢查點 JSON、訊息證據、確認 JSON 或轉譯後的螢幕截圖遺失
或為空，執行作業便會失敗。

冷啟動 GitHub Actions 租用沒有 Slack Web Cookie，因此瀏覽器擷取畫面
可能會停在 Slack 登入畫面。對於核准檢查點證明，應信任
轉譯後的檢查點影像及 Slack QA 成品，而非
`slack-desktop-smoke.png`。只有當瀏覽器螢幕截圖本身必須顯示
Slack Web 時，才使用已保留、且具有手動登入 Slack Web 設定檔的暖租用。

## 填入模式

| 模式          | 使用時機                                  | 遠端行為                                                                       | 取捨                                                 |
| ------------- | ----------------------------------------- | ------------------------------------------------------------------------------------- | -------------------------------------------------------- |
| `source`      | 一般 PR 證明、冷機器、CI        | 在 VM 內執行 `pnpm install --frozen-lockfile --prefer-offline` 和 `pnpm build` | 最慢，但原始碼簽出證明最有力                 |
| `prehydrated` | 你刻意準備了重複使用的租用 | 要求現有的 `node_modules` 和 `dist/`；略過安裝／建置                     | 速度快，但僅適用於由操作員控制的暖租用 |

GitHub Actions 一律會在 VM 執行前準備候選簽出內容。其
pnpm 儲存區會依作業系統、Node 版本及鎖定檔進行快取。若 VM 的 `source` 執行中
存在 `/var/cache/crabbox/pnpm`，也會重複使用它。

## 計時解讀

`mantis-slack-desktop-smoke-report.md` 包含各階段的計時：

- `crabbox.warmup` - 雲端供應商開機、桌面／瀏覽器就緒、SSH。
- `crabbox.inspect` - 租用中繼資料查詢。
- `credentials.prepare` - 取得 Convex 認證資訊租用。
- `crabbox.remote_run` - 同步、啟動瀏覽器、安裝／建置 OpenClaw 或
  驗證填入狀態、啟動閘道、擷取螢幕截圖及影片。
- `artifacts.copy` - 從 VM rsync 回本機。

當 Crabbox 傳回非零的遠端狀態，但 Mantis 已複製可證明 OpenClaw 閘道
設定已完成，或 Slack QA 命令本身已成功結束的中繼資料時，
`crabbox.remote_run` 可能會顯示 `accepted`。
請將 `accepted` 視為附帶說明的通過狀態，而不是失敗的情境。

如果執行速度緩慢：

- 暖機時間占比最高：預先烘焙或升級為更好的 Crabbox 供應商映像檔。
- `remote_run` 在 `source` 中占比最高：使用暖租用、改善 pnpm 儲存區
  重複使用，或將機器必要條件移入供應商映像檔。
- `remote_run` 在 `prehydrated` 中占比最高：遠端工作區實際上尚未
  準備就緒，或閘道／瀏覽器／Slack 設定速度緩慢。
- 成品複製時間占比最高：檢查影片大小及成品目錄內容。

## 證據檢查清單

良好的 PR 留言會顯示：

- 情境 ID 及候選 SHA
- GitHub Actions 執行 URL 及成品 URL
- 行內核准檢查點螢幕截圖，或來自已登入暖租用的 Slack Web
  螢幕截圖
- 可用時提供行內動畫預覽
- 完整 MP4 及裁剪後 MP4 的連結
- 通過／失敗狀態及報告中的計時摘要

請勿將螢幕截圖或影片提交至儲存庫。請將它們保留在 GitHub
Actions 成品或 PR 留言中。

## 失敗處理

如果工作流程在 VM 執行前失敗，請先檢查 Actions 作業。
常見原因：不受信任的 `candidate_ref`、缺少環境密鑰，或
候選項目安裝／建置失敗。

如果 VM 執行失敗但螢幕截圖已複製回來，請檢查：

```bash
cat mantis-slack-desktop-smoke-report.md
cat mantis-slack-desktop-smoke-summary.json
cat slack-desktop-command.log
cat openclaw-gateway.log
cat chrome.log
cat ffmpeg.log
```

如果執行作業保留了租用，請使用報告中的 `crabbox vnc ...`
命令開啟 VNC，完成後再停止租用：

```bash
crabbox stop --provider aws <cbx_id-or-slug>
```

如果 Slack 登入已過期，請在保留的租用中透過 VNC 修復，然後使用
`--lease-id` 重新執行。請勿將該瀏覽器設定檔烘焙至供應商映像檔中。

## 相關內容

- [QA 概觀](/zh-TW/concepts/qa-e2e-automation)
- [Slack 頻道](/zh-TW/channels/slack)
- [測試](/zh-TW/help/testing)
