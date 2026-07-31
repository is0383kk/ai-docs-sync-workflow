---
read_when:
    - 你希望智能体向用户提出一个结构化问题
    - 你正在回答或调试 ask_user 提示词
    - 你需要了解 `ask_user` 架构、超时或渠道行为
summary: ask_user 如何暂停智能体轮次以等待结构化的人工决策
title: 询问用户
x-i18n:
    generated_at: "2026-07-26T06:30:03Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 32556314a34c26054c3aabfdd8ecc474cf85196e5cc71adb833face596edbd24
    source_path: tools/ask-user.md
    workflow: 16
---

`ask_user` 允许智能体向用户提出一到三个结构化问题并
等待回答。它适用于确实应由用户作出的决定，
而不是例行确认，也不是智能体可根据请求、
代码或合理默认值推导出的信息。

该工具仅在主会话中可用。子智能体和其他非主要
运行无法使用它。

## 回答问题

你可以从任何受支持的对话界面回答：

- Web 端 Control UI 会将问题面板直接停靠在编辑器上方。对于
  包含多个问题的提示，面板会逐次显示一个问题，并通过简短的步骤条
  依次推进。完成回答后，面板会关闭，聊天中
  仅保留精简的回答摘要。
- 对于仅含一个问题且只能单选的提示，Telegram、Discord 和 Slack 会呈现原生按钮。
- 任何渠道都支持纯文本回复。回复数字、选项标签
  或你自己的答案即可。

OpenClaw 始终启用自由文本的 **Other** 回答。智能体不得在编写的选项列表中
添加 `Other` 选项。

## 平台行为

每个受支持的对话界面都可进行回答。Web 端 Control UI 使用
停靠式步骤条，展开时会取代编辑器；将其折叠后，会在精简的问题栏下方
恢复完整编辑器。iOS、macOS 和 Android 会显示
内联卡片；多个问题会保持堆叠，这是为便于触控而有意采用的交互方式。
每个平台都会在当前聊天时间线中保留问题与回答的摘要，
不会定时移除，并且所有平台都提供 **Skip**。

无法使用原生按钮的提示（包括多问题和
多选提示）在各渠道上会降级为易读的文本。Control UI
则会保留完整的结构化步骤条。

## 超时和未回答

默认超时时间为 900 秒。`timeoutSeconds` 会限制在
30 到 3600 秒的范围内。

如果问题在收到回答前过期或被取消，该工具会
返回 `status: "no_answer"`。随后智能体将根据自己的最佳判断继续。
智能体运行中止时，会取消其待处理的 Gateway 网关问题。

## 工具架构

```ts
{
  questions: Array<{
    id: string; // 唯一的 snake_case 回答键
    header: string; // 简短标签；截断至 12 个字符
    question: string; // 一个句子
    options: Array<{
      label: string;
      description?: string;
    }>; // 2-4 个选项
    multiSelect?: boolean;
  }>; // 1-3 个问题
  timeoutSeconds?: number; // 整数；默认为 900，限制在 30-3600
}
```

使用 `multiSelect: true` 时，用户可以选择多个选项。每个问题的回答
值都会以数组形式返回。

已回答结果示例：

```json
{
  "status": "answered",
  "answers": {
    "answers": {
      "deploy_target": ["暂存环境（推荐）"]
    }
  }
}
```

## 模型指导

面向模型的契约要求智能体：

- 仅在确实因应由用户作出的决定而受阻时提问；
- 优先只提一个问题，并且不得超过三个；
- 将推荐选项放在首位，并在其标签末尾添加 `(Recommended)`；
- 不要编写 `Other` 选项，因为系统会自动添加自由文本回答；
- 在 `no_answer` 后根据最佳判断继续。

智能体不应使用 `ask_user` 询问是否可以继续，也不应借此确认
自己的计划。
