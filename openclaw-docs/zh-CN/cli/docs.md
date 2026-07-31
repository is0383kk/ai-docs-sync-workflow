---
read_when:
    - 你想从终端搜索实时 OpenClaw 文档
    - 你需要知道文档 CLI 调用的是哪个托管搜索 API
summary: '`openclaw docs` 的 CLI 参考（搜索实时文档索引）'
title: 文档
x-i18n:
    generated_at: "2026-07-26T06:43:53Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: b0b575f0b76d40a53dd4f79c55fd65969a24eae27e27bd1c46d395f61fe89e42
    source_path: cli/docs.md
    workflow: 16
---

# `openclaw docs`

从终端搜索实时 OpenClaw 文档索引。

## 用法

```bash
openclaw docs                       # 输出文档入口点和搜索示例
openclaw docs <query...>            # 搜索实时文档索引
```

| 参数     | 描述                                                                        |
| ------------ | ---------------------------------------------------------------------------------- |
| `[query...]` | 自由格式的搜索查询。多词查询将以空格连接，并作为一个查询发送。 |

如果未提供查询，`openclaw docs` 会输出文档入口点 URL 和示例搜索命令，而不执行搜索。

## 示例

```bash
openclaw docs browser existing-session
openclaw docs sandbox allowHostControl
openclaw docs gateway token secretref
```

## 工作原理

`openclaw docs` 调用 `https://docs.openclaw.ai/api/search` 并渲染 JSON 结果。搜索请求使用固定的 30 秒超时时间。

## 输出

在富文本（TTY）终端中，结果显示为一个标题，后跟项目符号列表：页面标题、带链接的文档 URL，以及下一行的简短片段。结果为空时输出“No results.”。

在非富文本输出中（管道、`--no-color`、脚本），相同数据渲染为 Markdown：

```markdown
# 文档搜索：<query>

- [标题](https://docs.openclaw.ai/...) - 摘要
- [标题](https://docs.openclaw.ai/...) - 摘要
```

## 退出代码

| 代码 | 含义                                                                  |
| ---- | ------------------------------------------------------------------------ |
| `0`  | 搜索成功，包括结果为零的响应。                       |
| `1`  | 调用托管文档搜索 API 失败；stderr 会输出错误消息。 |

## 相关内容

- [CLI 参考](/zh-CN/cli)
- [实时文档](https://docs.openclaw.ai)
