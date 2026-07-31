---
read_when:
    - 你想通过智能体分析 PDF 文件
    - 你需要准确了解 PDF 工具的参数和限制
    - 你正在调试原生 PDF 模式与提取回退机制的差异
summary: 使用原生提供商支持和提取回退机制分析一个或多个 PDF 文档
title: PDF 工具
x-i18n:
    generated_at: "2026-07-26T07:04:02Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: e0e5b897e1e122af4b2f6f9a3eaeb73f6e93af1051d306ad82539b258de90c49
    source_path: tools/pdf.md
    workflow: 16
---

`pdf` 分析一个或多个 PDF 文档并返回文本。它在 Anthropic 和 Google 模型上使用原生文档输入，对其他所有提供商则回退到文本/图像提取。

## 可用性

仅当 OpenClaw 能为智能体解析到支持 PDF 的模型时，才会注册该工具。解析顺序如下：

1. `agents.defaults.pdfModel`（显式指定的主模型/回退模型）
2. `agents.defaults.imageModel`（显式指定的主模型/回退模型）
3. 智能体解析出的会话/默认模型，前提是其提供商支持原生 PDF 输入（Anthropic、Google），或已配置视觉模型
4. 自动检测具有可用身份验证的图像/视觉模型提供商，并优先选择原生支持 PDF 的提供商

使用每个回退候选模型前都会检查身份验证，因此已配置的 `provider/model` 只有在 OpenClaw 能为该智能体通过对应提供商的身份验证时才算有效。如果无法解析到可用模型，则不会公开 `pdf` 工具。

## 输入参考

<ParamField path="pdf" type="string">
一个 PDF 路径或 URL。
</ParamField>

<ParamField path="pdfs" type="string[]">
多个 PDF 路径或 URL，总计最多 10 个。
</ParamField>

<ParamField path="prompt" type="string" default="Analyze this PDF document.">
分析提示词。
</ParamField>

<ParamField path="pages" type="string">
页面筛选条件，例如 `1-5` 或 `1,3,7-9`。原生提供商模式不支持此参数。
</ParamField>

<ParamField path="password" type="string">
加密 PDF 的密码。应用于请求中的每个 PDF；仅供提取回退模式使用。
</ParamField>

<ParamField path="model" type="string">
可选的模型覆盖，格式为 `provider/model`。
</ParamField>

<ParamField path="maxBytesMb" type="number">
每个 PDF 的大小上限，以 MB 为单位。默认为 `agents.defaults.pdfMaxMb`；若未设置，则为 `10`。
</ParamField>

注意：

- 加载前会合并 `pdf` 和 `pdfs` 并去重；至少需要提供其中一个。
- `pages` 会解析为从 1 开始的页码，经过去重、排序，并限制在 `agents.defaults.pdfMaxPages`（默认值为 `20`）以内。如果某个范围没有匹配任何有效页码，则会在调用模型前报错。

## 支持的 PDF 引用

- 本地文件路径（包括 `~` 展开）
- `file://` URL
- `http://` 和 `https://` URL
- OpenClaw 管理的入站引用，例如 `media://inbound/<id>`

其他 URI 方案（例如 `ftp://`）会返回 `details.error = "unsupported_pdf_reference"`。工具在沙箱隔离环境中运行时，会拒绝远程 `http(s)` URL。启用仅限工作区的文件策略后，允许根目录之外的本地路径会被拒绝；OpenClaw 入站媒体存储区下由系统管理的入站引用和重放路径仍然允许使用。

## 执行模式

### 原生提供商模式

用于提供商 `anthropic` 和 `google`（目前仅这两个提供商声明支持原生 PDF 文档）。每个文件的原始 PDF 字节会作为原生文档/内联 PDF 部分直接发送到提供商 API。

限制：

- 不支持 `pages`；如果设置此参数，工具会抛出 `pages is not supported with native PDF providers`。
- 不支持 `password`；如果设置此参数，工具会抛出 `password is not supported with native PDF providers`。对于加密 PDF，请使用非原生模型。

### 提取回退模式

用于其他所有提供商。

1. 通过内置的 `document-extract` 插件，从选定页面（最多 `agents.defaults.pdfMaxPages` 页，默认 `20` 页）中提取文本。该插件使用 `clawpdf` 软件包（PDFium WebAssembly）提取文本和图像。
2. 如果提取的文本少于 `200` 个字符，则将相同页面渲染为 PNG 图像。总渲染预算为 `4,000,000` 像素，由所有需要图像的页面共享（按比例分配给剩余页面，而不是每页单独分配），因此已提取到足够文本的页面会完全跳过渲染。
3. 将提取的文本（以及所有已渲染图像）与提示词一并发送给选定模型。

详细信息：

- 加密 PDF 使用顶层 `password` 参数打开。
- 如果模型不支持图像输入，且无法提取任何文本，工具会报错。
- 如果图像渲染失败，OpenClaw 会丢弃图像并继续使用提取的文本。
- 如果目标模型仅支持文本输入，而提取过程生成了图像，OpenClaw 会丢弃图像并仅发送文本。

## 配置

```json5
{
  agents: {
    defaults: {
      pdfModel: {
        primary: "anthropic/claude-opus-4-6",
        fallbacks: ["openai/gpt-5.4-mini"],
      },
      pdfMaxBytesMb: 10,
      pdfMaxPages: 20,
    },
  },
}
```

| 键                            | 默认值  | 含义                                                                                      |
| ----------------------------- | ------- | ----------------------------------------------------------------------------------------- |
| `agents.defaults.pdfModel`    | 未设置  | 显式指定的主 PDF 模型/回退 PDF 模型；回退到 `imageModel`，然后回退到会话模型。 |
| `agents.defaults.pdfMaxMb`    | `10`    | 每个 PDF 的大小上限，以 MB 为单位。                                                       |
| `agents.defaults.pdfMaxPages` | `20`    | 每个 PDF 可处理的最大页数。                                                               |

有关字段的完整详细信息，请参阅[配置参考](/zh-CN/gateway/config-agents#agent-defaults)。

## 输出详细信息

该工具在 `content[0].text` 中返回文本，并在 `details` 中返回结构化元数据。

常见的 `details` 字段：

- `model`：解析出的模型引用（`provider/model`）
- `native`：原生提供商模式为 `true`，回退模式为 `false`
- `attempts`：成功前失败的回退尝试

路径字段：

- 单个 PDF 输入：`details.pdf`
- 多个 PDF 输入：`details.pdfs[]`，包含 `pdf` 条目
- 沙箱路径重写元数据（如适用）：`rewrittenFrom`

## 错误行为

| 条件                              | 结果                                                           |
| --------------------------------- | -------------------------------------------------------------- |
| 未提供 PDF 输入                   | 抛出 `pdf required: provide a path or URL to a PDF document` |
| PDF 数量超过 10 个                | `details.error = "too_many_pdfs"`                              |
| 不支持的引用方案                  | `details.error = "unsupported_pdf_reference"`                  |
| 对原生提供商使用 `pages`    | 抛出 `pages is not supported with native PDF providers`      |
| 对原生提供商使用 `password` | 抛出 `password is not supported with native PDF providers`   |

## 示例

单个 PDF：

```json
{
  "pdf": "/tmp/report.pdf",
  "prompt": "用 5 个要点总结此报告"
}
```

多个 PDF：

```json
{
  "pdfs": ["/tmp/q1.pdf", "/tmp/q2.pdf"],
  "prompt": "比较两份文档中的风险和时间线变化"
}
```

带页面筛选条件的回退模型：

```json
{
  "pdf": "https://example.com/report.pdf",
  "pages": "1-3,7",
  "model": "openai/gpt-5.4-mini",
  "prompt": "仅提取影响客户的事件"
}
```

使用提取回退模式处理加密 PDF：

```json
{
  "pdf": "/tmp/locked.pdf",
  "password": "example-password",
  "model": "openai/gpt-5.4-mini",
  "prompt": "总结此合同"
}
```

## 相关内容

- [工具概览](/zh-CN/tools) - 所有可用的智能体工具
- [配置参考](/zh-CN/gateway/config-agents#agent-defaults) - pdfMaxBytesMb 和 pdfMaxPages 配置
