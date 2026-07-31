---
read_when:
    - 你想從終端機搜尋即時 OpenClaw 文件
    - 你需要知道文件命令列介面會呼叫哪個託管搜尋 API
summary: '`openclaw docs` 的命令列介面參考（搜尋即時文件索引）'
title: 文件說明
x-i18n:
    generated_at: "2026-07-26T08:27:57Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: b0b575f0b76d40a53dd4f79c55fd65969a24eae27e27bd1c46d395f61fe89e42
    source_path: cli/docs.md
    workflow: 16
---

# `openclaw docs`

從終端機搜尋即時 OpenClaw 文件索引。

## 使用方式

```bash
openclaw docs                       # 顯示文件進入點與搜尋範例
openclaw docs <query...>            # 搜尋即時文件索引
```

| 引數     | 說明                                                                        |
| ------------ | ---------------------------------------------------------------------------------- |
| `[query...]` | 自由格式搜尋查詢。多字詞查詢會以空格連接，並作為單一查詢傳送。 |

未提供查詢時，`openclaw docs` 會顯示文件進入點 URL 與搜尋命令範例，而不執行搜尋。

## 範例

```bash
openclaw docs browser existing-session
openclaw docs sandbox allowHostControl
openclaw docs gateway token secretref
```

## 運作方式

`openclaw docs` 會呼叫 `https://docs.openclaw.ai/api/search` 並呈現 JSON 結果。搜尋請求使用固定的 30 秒逾時。

## 輸出

在富格式（TTY）終端機中，結果會呈現為標題，後接項目符號清單：頁面標題、含連結的文件 URL，以及下一行的簡短摘錄。沒有結果時會顯示 "No results."。

在非富格式輸出（透過管線傳送、`--no-color`、指令碼）中，相同資料會呈現為 Markdown：

```markdown
# 文件搜尋：<query>

- [標題](https://docs.openclaw.ai/...) - 摘錄
- [標題](https://docs.openclaw.ai/...) - 摘錄
```

## 結束代碼

| 代碼 | 意義                                                                  |
| ---- | ------------------------------------------------------------------------ |
| `0`  | 搜尋成功，包括零結果的回應。                       |
| `1`  | 託管文件搜尋 API 呼叫失敗；stderr 會顯示錯誤訊息。 |

## 相關內容

- [命令列介面參考](/zh-TW/cli)
- [即時文件](https://docs.openclaw.ai)
