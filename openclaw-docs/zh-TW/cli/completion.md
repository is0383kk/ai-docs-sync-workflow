---
read_when:
    - 你想要 zsh/bash/fish/PowerShell 的 shell 自動補全功能
    - 你需要將補全指令碼快取在 OpenClaw 狀態目錄下
summary: '`openclaw completion` 的命令列介面參考（產生／安裝 Shell 自動完成指令碼）'
title: 完成
x-i18n:
    generated_at: "2026-07-26T07:45:44Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 67cb52a47036745150887c752d18e2dfa84fab2722c27c696142d23080bb2efd
    source_path: cli/completion.md
    workflow: 16
---

# `openclaw completion`

產生 Shell 自動補全指令碼、將其快取至 OpenClaw 狀態目錄，並可選擇安裝至你的 Shell 設定檔。

## 用法

```bash
openclaw completion                          # 將 zsh 指令碼輸出至標準輸出
openclaw completion --shell fish             # 輸出 fish 指令碼
openclaw completion --write-state            # 快取所有 Shell 的指令碼
openclaw completion --write-state --install  # 快取後一次完成安裝
openclaw completion --shell bash --write-state
```

## 選項

- `-s, --shell <shell>`：目標 Shell（`zsh`、`bash`、`powershell`、`fish`；預設：`zsh`）
- `-i, --install`：在你的 Shell 設定檔中加入載入快取指令碼的來源行，以安裝自動補全
- `--write-state`：將自動補全指令碼寫入 `$OPENCLAW_STATE_DIR/completions`（預設為 `~/.openclaw/completions`），而不輸出至標準輸出；搭配 `--shell` 時只寫入該 Shell，否則寫入全部四種 Shell
- `-y, --yes`：略過安裝確認提示（非互動式）

## 安裝流程

`--install` 會讓你的設定檔指向快取的指令碼，因此快取必須先存在：若不存在，命令會失敗並提示你執行 `openclaw completion --write-state`。搭配 `--write-state --install` 可一次完成兩者。未指定 `--shell` 時，`--install` 會從 `$SHELL` 偵測 Shell（偵測失敗時使用 zsh）。

安裝程序會將一小段 `# OpenClaw Completion` 區塊寫入你的 Shell 設定檔，並以快取來源行取代任何較舊且緩慢的 `source <(openclaw completion ...)` 行：

| Shell      | 設定檔                                                                                                                                                                                     |
| ---------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| bash       | `~/.bashrc`（缺少 `~/.bashrc` 時改用 `~/.bash_profile`）                                                                                                                   |
| fish       | `~/.config/fish/config.fish`                                                                                                                                                                        |
| powershell | `~/.config/powershell/Microsoft.PowerShell_profile.ps1`（在 Windows 上：`Documents/PowerShell/Microsoft.PowerShell_profile.ps1`，或 Windows PowerShell 使用 `Documents/WindowsPowerShell/...`）                                                                                     |
| zsh        | `~/.zshrc`                                                                                                                                                                        |

## 注意事項

- 未指定 `--install` 或 `--write-state` 時，命令會將指令碼輸出至標準輸出。
- 產生自動補全時會立即載入完整的命令樹，包括外掛的命令列介面命令，因此也會包含巢狀子命令。
- `openclaw update` 會在成功更新後自動重新整理自動補全快取；`openclaw doctor` 可修復缺失或過期的自動補全設定。

## 相關內容

- [命令列介面參考](/zh-TW/cli)
