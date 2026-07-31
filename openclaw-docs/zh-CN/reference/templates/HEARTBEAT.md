---
read_when:
    - 手动引导工作区
summary: HEARTBEAT.md 的工作空间模板
title: HEARTBEAT.md 模板
x-i18n:
    generated_at: "2026-07-26T07:00:06Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: d5b02cd62708a87515c4ae59bd2ffab3e4c8ebf81f4126fdd43ced756241b151
    source_path: reference/templates/HEARTBEAT.md
    workflow: 16
---

# HEARTBEAT.md 模板

`HEARTBEAT.md` 位于 Agent 工作区中，用于保存定期 Heartbeat 检查清单。将其留空，或仅包含空白字符、Markdown 注释、ATX 标题、空列表框架（`- `、`* [ ]`）或围栏标记，即可让 OpenClaw 完全跳过 Heartbeat 模型调用（`reason=empty-heartbeat-file`）。

随附的默认内容：

```markdown
<!-- Heartbeat template; comments-only content prevents scheduled heartbeat API calls. -->

# 将此文件留空（或仅包含注释）可跳过 Heartbeat API 调用。

# 当 Heartbeat 应检查共享上下文时，请在下方添加简短的检查清单。
```

仅当一次 Heartbeat 轮次应同时检查这些项目时，才在注释行下方添加简短的检查清单。清单应保持精简：Heartbeat 每次触发时都会读取此文件（默认每 30 分钟一次），因此臃肿的指令会在每次唤醒时消耗 token。

对于独立调度或仅在到期时执行的检查，请创建[定时任务](/zh-CN/automation/cron-jobs)。Heartbeat 草稿不再支持调度器语法。运行 `openclaw doctor --fix` 可转换旧版 `tasks:` 块。

## 相关内容

- [Heartbeat](/zh-CN/gateway/heartbeat)
- [Heartbeat 配置](/zh-CN/gateway/config-agents)
