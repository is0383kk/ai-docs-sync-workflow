---
read_when:
    - 你想要移除閘道服務和／或本機狀態
    - 你想先進行試執行
summary: '`openclaw uninstall` 的命令列介面參考（移除閘道服務與本機資料）'
title: 解除安裝
x-i18n:
    generated_at: "2026-07-26T07:48:10Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 1e2e3996cf6d5c0fd11e5054c8fe60f7f8d25047193bb13944ca170bf77b581a
    source_path: cli/uninstall.md
    workflow: 16
---

# `openclaw uninstall`

解除安裝閘道服務和／或本機資料。命令列介面本身不會被移除；請另外透過 npm/pnpm 解除安裝。

## 選項

| 旗標                | 預設值 | 說明                                          |
| ------------------- | ------- | ---------------------------------------------------- |
| `--service`         | `false` | 移除閘道服務。                          |
| `--state`           | `false` | 移除狀態與設定。                             |
| `--workspace`       | `false` | 移除工作區目錄。                        |
| `--app`             | `false` | 移除 macOS 應用程式。                                |
| `--all`             | `false` | `--service --state --workspace --app` 的簡寫。 |
| `--yes`             | `false` | 略過確認提示。                           |
| `--non-interactive` | `false` | 停用提示；需要 `--yes`。                   |
| `--dry-run`         | `false` | 列印預定執行的動作，但不移除檔案。        |

未指定範圍旗標時，互動式多選提示會詢問要移除哪些元件（預設預先選取服務、狀態和工作區）。

## 範例

```bash
openclaw backup create
openclaw uninstall
openclaw uninstall --service --yes --non-interactive
openclaw uninstall --state --workspace --yes --non-interactive
openclaw uninstall --all --yes
openclaw uninstall --dry-run
```

## 注意事項

- 移除狀態或工作區前，請先執行 `openclaw backup create`，以建立可還原的快照。
- `--state` 會保留已設定的工作區目錄，除非同時選取 `--workspace`。

## 相關內容

- [命令列介面參考](/zh-TW/cli)
- [解除安裝](/zh-TW/install/uninstall)
