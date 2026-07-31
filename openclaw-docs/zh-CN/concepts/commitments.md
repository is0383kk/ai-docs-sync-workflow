---
read_when:
    - 你正在升级一个曾使用推断式跟进承诺的配置
    - 你想要查看或忽略之前存储的跟进记录
sidebarTitle: Commitments
summary: 已停用的推断式跟进承诺的状态与清理指南
title: 推断式跟进承诺
x-i18n:
    generated_at: "2026-07-26T06:11:20Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: cfaa8c44be4ffb8db48279dba5347d4f598a193bfc4e244aeaed7a93e00ffb79
    source_path: concepts/commitments.md
    workflow: 16
---

推断式跟进承诺实验已停用。OpenClaw 不再从新对话中提取后续事项，也不再通过 Heartbeat 交付这些事项；原有的
`commitments` 配置块会由 `openclaw doctor --fix` 移除。

精确提醒和计划工作继续使用
[定时任务](/zh-CN/automation/cron-jobs)。持久的对话事实应存入
[记忆](/zh-CN/concepts/memory)。

## 现有记录

先前存储的跟进承诺仍保留在共享 SQLite 状态数据库中，因此升级不会销毁操作员可见的历史记录。使用旧版维护
CLI 检查或解除这些记录：

```bash
openclaw commitments --all
openclaw commitments dismiss cm_abc123
```

有关维护命令的参考信息，请参阅 [`openclaw commitments`](/zh-CN/cli/commitments)。

## 相关内容

- [定时任务](/zh-CN/automation/cron-jobs)
- [记忆概览](/zh-CN/concepts/memory)
- [Heartbeat](/zh-CN/gateway/heartbeat)
