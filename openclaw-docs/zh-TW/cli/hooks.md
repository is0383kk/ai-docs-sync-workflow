---
read_when:
    - 你想要管理代理程式鉤子
    - 你想要檢查鉤子的可用性，或啟用工作區鉤子
summary: '`openclaw hooks`（代理程式鉤子）的命令列介面參考資料'
title: 鉤子
x-i18n:
    generated_at: "2026-07-26T08:19:20Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: d4d58ea2270cf5122018f7be2943401229929f48f448b15fdd126d1cc99e1e56
    source_path: cli/hooks.md
    workflow: 16
---

# `openclaw hooks`

管理代理程式鉤子（針對 `/new`、`/reset` 等命令及閘道啟動的事件驅動自動化）。單獨使用 `openclaw hooks` 等同於 `openclaw hooks list`。

相關內容：[鉤子](/zh-TW/automation/hooks) - [外掛鉤子](/zh-TW/plugins/hooks)

## 列出鉤子

```bash
openclaw hooks list [--eligible] [--json] [-v|--verbose]
```

列出從工作區、受管理、額外及隨附目錄中探索到的鉤子。

- `--eligible`：僅顯示符合需求的鉤子。
- `--json`：結構化輸出。
- `-v, --verbose`：包含顯示未符合需求的 Missing 欄。

```
鉤子（4/5 個就緒）

就緒：
  🚀 boot-md ✓ - 閘道啟動時執行 BOOT.md
  📎 bootstrap-extra-files ✓ - 在代理程式啟動載入期間注入額外的工作區啟動載入檔案
  📝 command-logger ✓ - 將所有命令事件記錄至集中式稽核檔案
  💾 session-memory ✓ - 發出 /new 或 /reset 命令時，將工作階段情境儲存至記憶
```

## 取得鉤子資訊

```bash
openclaw hooks info <name> [--json]
```

`<name>` 是鉤子名稱或鉤子索引鍵（例如 `session-memory`）。顯示來源、檔案／處理常式路徑、首頁、事件，以及各項需求的狀態（二進位檔、環境變數、設定、作業系統）。

## 檢查資格

```bash
openclaw hooks check [--json]
```

輸出就緒／未就緒的數量摘要；若有未就緒的鉤子，則逐一列出及其阻擋原因。

## 啟用鉤子

```bash
openclaw hooks enable <name>
```

在設定中新增／更新 `hooks.internal.entries.<name>.enabled = true`，並同時開啟 `hooks.internal.enabled` 主開關（在至少設定一個鉤子前，閘道不會載入任何內部鉤子處理常式）。若鉤子不存在、由外掛管理，或不符合資格（缺少需求），則會失敗。

由外掛管理的鉤子會在 `hooks list` 中顯示 `plugin:<id>`，且無法在此啟用／停用；請改為啟用或停用所屬的外掛。

啟用後請重新啟動閘道（重新啟動 macOS 選單列應用程式，或在開發環境中重新啟動你的閘道程序），以便重新載入鉤子。

## 停用鉤子

```bash
openclaw hooks disable <name>
```

設定 `hooks.internal.entries.<name>.enabled = false`。之後請重新啟動閘道。

## 安裝及更新鉤子套件

```bash
openclaw plugins install <package>        # 預設使用 npm
openclaw plugins install npm:<package>    # 僅使用 npm
openclaw plugins install <package> --pin  # 鎖定解析出的版本
openclaw plugins install <path>           # 本機目錄或封存檔
openclaw plugins install -l <path>        # 連結本機目錄而非複製

openclaw plugins update <id>
openclaw plugins update --all
openclaw plugins update --dry-run
```

鉤子套件透過統一的外掛安裝程式／更新程式安裝；`openclaw hooks install`／`openclaw hooks update` 仍可作為已棄用的別名使用，它們會顯示警告並轉送至 `plugins` 命令。

- Npm 規格僅限登錄檔：套件名稱加上選用的確切版本或 dist-tag。不接受 Git／URL／檔案規格及 semver 範圍。相依套件會以 `--ignore-scripts` 在專案本機安裝。
- 單獨規格及 `@latest` 會維持在穩定版軌道；若 npm 解析到預發行版本，OpenClaw 會停止並要求你明確選擇加入（`@beta`、`@rc`，或確切的預發行版本）。
- 支援的封存格式：`.zip`、`.tgz`、`.tar.gz`、`.tar`。
- `-l, --link` 會連結本機目錄而非複製（將其加入 `hooks.internal.load.extraDirs`）；連結的鉤子套件是來自操作人員所設定目錄的受管理鉤子，而非工作區鉤子。
- `--pin` 會將 npm 安裝以確切解析出的 `name@version` 記錄於共用 SQLite 狀態中。
- 安裝作業會將套件複製至 `~/.openclaw/hooks/<id>`、在 `hooks.internal.entries.*` 下啟用其鉤子，並在共用 SQLite 狀態中記錄安裝來源。
- 若儲存的完整性雜湊不再與擷取的成品相符，OpenClaw 會發出警告並在繼續前提示確認；可傳入全域 `--yes` 以略過提示（例如在 CI 中）。

## 隨附鉤子

| 鉤子                  | 事件                                            | 功能                                                                                       |
| --------------------- | ------------------------------------------------- | -------------------------------------------------------------------------------------------------- |
| boot-md               | `gateway:startup`                                 | 閘道啟動時，針對每個已設定的代理程式範圍執行 `BOOT.md`                                  |
| bootstrap-extra-files | `agent:bootstrap`                                 | 在代理程式啟動載入期間注入額外的啟動載入檔案（例如 monorepo 的 `AGENTS.md`／`TOOLS.md`） |
| command-logger        | `command`                                         | 將命令事件記錄至 `~/.openclaw/logs/commands.log`                                             |
| compaction-notifier   | `session:compact:before`、`session:compact:after` | 在工作階段壓縮開始及完成時傳送可見的聊天通知                             |
| session-memory        | `command:new`、`command:reset`                    | 在 `/new` 或 `/reset` 時將工作階段情境儲存至記憶                                              |

使用 `openclaw hooks enable <hook-name>` 啟用任何隨附鉤子。完整詳細資料、設定鍵及預設值：[隨附鉤子](/zh-TW/automation/hooks#bundled-hooks)。

### command-logger 記錄檔

```bash
tail -n 20 ~/.openclaw/logs/commands.log        # 最近的命令
cat ~/.openclaw/logs/commands.log | jq .          # 美化輸出
grep '"action":"new"' ~/.openclaw/logs/commands.log | jq .   # 依動作篩選
```

## 注意事項

- `hooks list --json`、`info --json` 及 `check --json` 會將結構化 JSON 直接寫入標準輸出。

## 相關內容

- [命令列介面參考](/zh-TW/cli)
- [自動化鉤子](/zh-TW/automation/hooks)
