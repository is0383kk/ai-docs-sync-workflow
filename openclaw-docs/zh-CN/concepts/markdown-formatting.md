---
read_when:
    - 你正在更改出站渠道的 Markdown 格式或分块方式
    - 你正在添加新的渠道格式化程序或样式映射
    - 你正在调试各渠道中的格式回归问题
summary: 出站渠道的 Markdown 格式化流水线
title: Markdown 格式设置
x-i18n:
    generated_at: "2026-07-26T05:46:37Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: f9a35fd9a6386068e1e3bec73ec6e692f49239b468f42dd737f919b1c6a88e41
    source_path: concepts/markdown-formatting.md
    workflow: 16
---

OpenClaw 在渲染特定渠道的输出之前，会将出站 Markdown 转换为共享的中间表示
（IR）。IR 保留纯文本以及样式/链接跨度，因此一次解析即可供所有渠道使用，并且分块绝不会
在跨度中间拆分格式。

## 流程

1. **将 Markdown 解析为 IR**（`markdownToIR`）- 纯文本加样式跨度
   （粗体、斜体、删除线、代码、代码块、剧透、块引用、
   1-6 级标题）和链接跨度。偏移量采用 UTF-16 代码单元，因此 Signal 样式
   范围可直接与其 API 对齐。仅当渠道选择使用表格模式时，
   才会解析表格。
2. **对 IR 进行分块**（`chunkMarkdownIR` / `renderMarkdownIRChunksWithinLimit`）
   - 分割发生在渲染前的 IR 文本上，因此内联样式和
     链接会按块切分，而不会在边界处断裂。
3. **按渠道渲染**（`renderMarkdownWithMarkers`）- 样式标记映射
   将跨度转换为渠道的原生标记。

| 渠道                                                              | 渲染器                                                                               | 说明                                                                                         |
| ---------------------------------------------------------------- | ------------------------------------------------------------------------------------ | ---------------------------------------------------------------------------------------- |
| Slack                                                            | mrkdwn 标记（`*bold*`、`_italic_`、`` `code` ``、代码围栏）                      | 链接转换为 `<url\|label>`；解析时禁用自动链接，以避免重复添加链接      |
| Telegram                                                         | HTML 标签（`<b>`、`<i>`、`<s>`、`<code>`、`<pre><code>`、`<a href>`、`<tg-spoiler>`） | 启用 `richMessages` 时，还支持富消息表格和标题（`<h1>`-`<h6>`） |
| Signal                                                           | 纯文本 + `text-style` 范围                                                     | 当标签与 URL 不同时，链接渲染为 `label (url)`                        |
| Discord、WhatsApp、iMessage、Microsoft Teams 和其他渠道 | 纯文本                                                                           | 不使用基于 IR 的样式；Markdown 表格转换仍通过 `convertMarkdownTables` 运行    |

## IR 示例

输入 Markdown：

```markdown
你好，**世界** - 请参阅[文档](https://docs.openclaw.ai)。
```

IR（示意）：

```json
{
  "text": "你好，世界 - 请参阅文档。",
  "styles": [{ "start": 3, "end": 5, "style": "bold" }],
  "links": [{ "start": 12, "end": 14, "href": "https://docs.openclaw.ai" }]
}
```

## 表格处理

`markdown.tables` 控制渠道如何转换 Markdown 表格，可按
渠道配置，也可选择按账户配置：

| 模式      | 行为                                                                             |
| --------- | ------------------------------------------------------------------------------------ |
| `code`    | 在代码块中渲染为对齐的 ASCII 表格（后备默认值）              |
| `bullets` | 将每一行转换为 `label: value` 项目符号列表项                                   |
| `block`   | 在传输方式支持时保留原生表格；否则回退到 `code` |
| `off`     | 禁用表格解析；原始表格文本不作更改地传递                       |

各渠道插件的默认值：Signal、WhatsApp 和 Matrix 默认使用
`bullets`；Mattermost 默认使用 `off`；Telegram 默认使用 `block`（除非账户启用了 `richMessages`，否则
会解析为 `code`）。任何
未明确设置插件默认值的渠道都会回退到 `code`。

```yaml
channels:
  discord:
    markdown:
      tables: code
    accounts:
      work:
        markdown:
          tables: off
```

## 分块规则

- 分块限制来自渠道适配器/配置，并应用于 IR 文本，而非
  渲染后的输出。
- 围栏代码块会作为一个整体保留，并带有结尾换行符，以便
  渠道正确渲染结束围栏。
- 列表和块引用前缀属于 IR 文本的一部分，因此分块绝不会
  在前缀中间拆分。
- 内联样式绝不会跨块拆分；渲染器会在下一个块的开头重新开启
  尚未闭合的样式。

有关各渠道的分块边界和
交付行为，请参阅[流式传输和分块](/concepts/streaming)。

## 链接策略

- **Slack：** `[label](url)` -> `<url|label>`；裸 URL 保持原样。
- **Telegram：** `[label](url)` -> `<a href="url">label</a>`（HTML 解析模式）。
- **Signal：** `[label](url)` -> `label (url)`，除非标签已经
  与 URL 匹配。

## 剧透

Signal 会解析剧透标记（`||spoiler||`）并映射到 `SPOILER`
样式范围，Telegram 会将其映射到 `<tg-spoiler>`。其他渠道将
`||...||` 视为纯文本。

## 添加或更新渠道格式化程序

1. 使用 `markdownToIR(...)` **仅解析一次**，并传入适合渠道的
   选项（`autolink`、`headingStyle`、`blockquotePrefix`、`tableMode`）。
2. 使用 `renderMarkdownWithMarkers(...)` 和样式标记映射进行**渲染**（对于
   Signal 等传输方式，则使用自定义样式范围逻辑）。
3. 在渲染每个块之前，使用 `chunkMarkdownIR(...)` 或
   `renderMarkdownIRChunksWithinLimit(...)` 进行**分块**。
4. **连接适配器**，使出站发送路径调用新的分块器和渲染器。
5. 使用格式测试进行**测试**；如果渠道会分块，还需添加出站交付测试。

## 常见注意事项

- Slack 尖括号标记（`<@U123>`、`<#C123>`、`<https://...>`）必须
  在转义后保留下来；原始 HTML 仍需安全转义。
- Telegram HTML 需要转义标签外的文本，以避免标记损坏。
- Signal 样式范围使用 UTF-16 偏移量，而非码点偏移量。
- 保留围栏代码块末尾的换行符，使结束标记
  单独占一行。

## 相关内容

<CardGroup cols={2}>
  <Card title="流式传输和分块" href="/zh-CN/concepts/streaming" icon="bars-staggered">
    出站流式传输行为、分块边界和特定于渠道的交付。
  </Card>
  <Card title="系统提示词" href="/zh-CN/concepts/system-prompt" icon="message-lines">
    模型在对话开始前看到的内容，包括注入的工作区文件。
  </Card>
</CardGroup>
