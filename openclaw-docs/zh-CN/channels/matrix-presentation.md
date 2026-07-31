---
read_when:
    - 构建可呈现 OpenClaw 富响应的 Matrix 客户端
    - 调试 com.openclaw.presentation 事件内容
summary: 面向 OpenClaw 感知客户端的 Matrix 呈现元数据
title: Matrix 呈现元数据
x-i18n:
    generated_at: "2026-07-26T06:06:52Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: c0de4d13c6cefc6f91dcc7a4b0edeea6bf001f3bd71f52c9f0498ad422783d8a
    source_path: channels/matrix-presentation.md
    workflow: 16
---

OpenClaw 会将规范化的 `MessagePresentation` 元数据附加到出站 Matrix `m.room.message` 事件的 `com.openclaw.presentation` 内容键下。

标准 Matrix 客户端会继续呈现纯文本 `body`。支持 OpenClaw 的客户端可以读取结构化元数据，并呈现按钮、选择器、上下文行和分隔线等原生 UI。

## 事件内容

```json
{
  "msgtype": "m.text",
  "body": "选择模型\n\n选择模型：\n- DeepSeek",
  "com.openclaw.presentation": {
    "version": 1,
    "type": "message.presentation",
    "title": "选择模型",
    "tone": "info",
    "blocks": [
      {
        "type": "select",
        "placeholder": "选择模型",
        "options": [
          {
            "label": "DeepSeek",
            "value": "/model deepseek/deepseek-chat"
          }
        ]
      }
    ]
  }
}
```

- `version` 是元数据架构版本；当前版本为 `1`。`type` 是稳定的判别字段，始终为 `"message.presentation"`。Matrix 适配器只会发出版本和类型与此完全一致的载荷；客户端同样应忽略无法安全解析的未知版本、未知的 `type` 值和未知的块类型。
- `title` 和 `tone`（`info`、`success`、`warning`、`danger`、`neutral`）是可选提示。
- 按钮和选择选项可在旧版字符串 `value` 之外携带类型化的 `action`（`{ "type": "command", "command": "/..." }` 或 `{ "type": "callback", "value": "..." }`）。两者同时存在时，优先使用 `action`。

## 回退行为

OpenClaw 始终会将可读的纯文本回退内容呈现到 `body` 中。结构化元数据是附加信息，不得作为实现基本 Matrix 互操作性的必要条件。

回退呈现规则：

- `title`、`text` 和 `context` 内容呈现为纯文本行。
- 带有 `command` 操作的按钮呈现为 ``label: `/command` ``，使命令保持可复制。带有 `callback` 操作或只有旧版 `value` 的按钮仅呈现标签，从而使不透明的回调值保持私密；禁用的按钮始终只呈现标签。URL 和 Web 应用按钮呈现为 `label: URL`。
- 选择块将占位符（或 `Options:`）呈现为标题，并在其后显示仅含标签的选项行。
- 如果没有任何内容可呈现（例如仅包含分隔线的呈现内容），正文将回退为 `---`。

不受支持的客户端会继续显示回退文本。支持 OpenClaw 的客户端可优先使用结构化元数据显示内容，同时保留回退文本，以供复制、搜索、通知和无障碍功能使用。

## 支持的块

Matrix 出站适配器声明原生支持：

- `buttons`
- `select`
- `context`
- `divider`

`text` 块始终通过回退正文获得支持。应将所有块视为尽力呈现的提示；遇到未知字段和块类型时应将其忽略，而不是使整条消息失败。

## 交互

此元数据不会添加 Matrix 回调语义。按钮和选择项的值是回退交互载荷，通常为斜杠命令或文本命令。希望支持交互的 Matrix 客户端会解析控件值（依次为 `action.command`、`action.value`、`value`），并将其作为普通消息发送回房间。

例如，可以通过在同一房间中将值 `/model deepseek/deepseek-chat` 作为加密的 Matrix 文本消息发送，来处理带有该值的按钮。

## 与审批元数据的关系

`com.openclaw.presentation` 用于常规富消息呈现。

审批提示使用专用的 `com.openclaw.approval` 元数据，因为审批包含安全敏感的状态、决策以及 Exec/插件详情。如果同一事件中同时存在两个元数据键，客户端应优先使用专用的审批呈现器。

## 媒体消息

当回复包含多个媒体 URL 时，OpenClaw 会为每个媒体 URL 发送一个 Matrix 事件。说明文字和呈现元数据仅附加到第一个事件，使客户端获得一个稳定的结构化载荷，而不会出现重复的呈现器。长文本被分块到多个事件中时也适用同一规则：元数据仅随第一个事件发送。

呈现元数据应保持紧凑。较长的用户可见文本应保留在 `body` 中，并使用常规的 Matrix 文本分块路径。
