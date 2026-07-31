---
read_when:
    - 手動啟動工作區建置
summary: HEARTBEAT.md 的工作區範本
title: HEARTBEAT.md 範本
x-i18n:
    generated_at: "2026-07-26T08:36:34Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: d5b02cd62708a87515c4ae59bd2ffab3e4c8ebf81f4126fdd43ced756241b151
    source_path: reference/templates/HEARTBEAT.md
    workflow: 16
---

# HEARTBEAT.md 範本

`HEARTBEAT.md` 位於代理程式工作區中，並保存定期心跳偵測檢查清單。請將其保持空白，或僅包含空白字元、Markdown 註解、ATX 標題、空白清單框架（`- `、`* [ ]`）或圍欄標記，讓 OpenClaw 完全略過心跳偵測模型呼叫（`reason=empty-heartbeat-file`）。

隨附的預設內容：

```markdown
<!-- Heartbeat template; comments-only content prevents scheduled heartbeat API calls. -->

# 將此檔案保持空白（或僅包含註解），即可略過心跳偵測 API 呼叫。

# 當心跳偵測需要檢查共用內容時，請在下方新增簡短的檢查清單。
```

只有當一次心跳偵測回合應一併檢查這些項目時，才在註解行下方新增簡短的檢查清單。請保持精簡：心跳偵測每次觸發時都會讀取此檔案（預設每 30 分鐘一次），因此過度冗長的指示會在每次喚醒時耗用權杖。

若是獨立排程或僅在到期時執行的檢查，請建立[排程工作](/zh-TW/automation/cron-jobs)。心跳偵測暫存內容不再支援排程器語法。執行 `openclaw doctor --fix` 以轉換舊版 `tasks:` 區塊。

## 相關內容

- [心跳偵測](/zh-TW/gateway/heartbeat)
- [心跳偵測設定](/zh-TW/gateway/config-agents)
