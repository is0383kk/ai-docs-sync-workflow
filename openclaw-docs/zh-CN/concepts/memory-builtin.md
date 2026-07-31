---
read_when:
    - 你想了解默认的记忆后端
    - 你想要配置嵌入提供商或混合搜索
summary: 基于 SQLite 的默认记忆后端，支持关键词、向量和混合搜索
title: 内置记忆引擎
x-i18n:
    generated_at: "2026-07-26T06:07:26Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: c3efb6f1449d9b55717b3c117444ba7d4519d0111b842b48790ad85551511433
    source_path: concepts/memory-builtin.md
    workflow: 16
---

内置引擎是默认的记忆后端。它将你的记忆索引存储在每个智能体各自的 SQLite 数据库中，无需额外依赖即可开始使用。

## 提供的功能

- 通过 FTS5 全文索引（BM25 评分）进行**关键词搜索**。
- 通过任意受支持提供商的嵌入进行**向量搜索**。
- 结合两者以获得最佳结果的**混合搜索**。
- 通过三元组分词支持中文、日文和韩文的 **CJK 支持**。
- 使用 **sqlite-vec 加速**数据库内向量查询（可选）。

## 入门指南

默认情况下，内置引擎使用 OpenAI 嵌入。如果已配置 `OPENAI_API_KEY` 或
`models.providers.openai.apiKey`，则无需额外的记忆配置即可使用向量搜索。

要显式设置提供商：

```json5
{
  memory: {
    search: {
      provider: "openai",
    },
  },
}
```

如果没有嵌入提供商，则只能使用关键词搜索。

要强制使用本地 GGUF 嵌入，请安装官方 llama.cpp 提供商插件，
然后将 `local.modelPath` 指向 GGUF 文件：

```bash
openclaw plugins install @openclaw/llama-cpp-provider
```

```json5
{
  memory: {
    search: {
      provider: "local",
      fallback: "none",
      local: {
        modelPath: "~/.node-llama-cpp/models/embeddinggemma-300m-qat-Q8_0.gguf",
      },
    },
  },
}
```

## 支持的嵌入提供商

| 提供商            | ID                  | 说明                                |
| ----------------- | ------------------- | ----------------------------------- |
| Bedrock           | `bedrock`           | 使用 AWS 凭证链                     |
| DeepInfra         | `deepinfra`         | 默认值：`BAAI/bge-m3`              |
| Gemini            | `gemini`            | 支持多模态（图像 + 音频）            |
| GitHub Copilot    | `github-copilot`    | 使用你的 Copilot 订阅                |
| LM Studio         | `lmstudio`          | 本地/自行托管                        |
| 本地              | `local`             | `@openclaw/llama-cpp-provider`      |
| Mistral           | `mistral`           |                                     |
| Ollama            | `ollama`            | 本地/自行托管                        |
| OpenAI            | `openai`            | 默认值：`text-embedding-3-small`   |
| OpenAI 兼容       | `openai-compatible` | 通用 `/v1/embeddings` 端点           |
| Voyage            | `voyage`            |                                     |

设置 `memory.search.provider` 可不再使用 OpenAI。

## 索引的工作原理

OpenClaw 将 `MEMORY.md` 和 `memory/*.md` 划分为分块（默认每块 400 个词元，
重叠 80 个词元），并将其存储在每个智能体各自的 SQLite 数据库中。

- **索引位置：**所属智能体的数据库，位于
  `~/.openclaw/agents/<agentId>/agent/openclaw-agent.sqlite`
- **存储维护：**通过定期检查点和
  关闭时检查点限制 SQLite WAL 辅助文件的大小。
- **文件监视：**记忆文件的更改会触发防抖重新索引
  （默认 1.5 秒）。
- **自动重新索引：**当嵌入
  提供商、模型、分块配置、已配置的数据源或作用域发生变化时，索引会自动重建。
- **按需重新索引：**`openclaw memory index --force`

<Info>
你还可以使用 `memory.search.extraPaths` 为工作区之外的 Markdown 文件建立索引。请参阅
[配置参考](/zh-CN/reference/memory-config#additional-memory-paths)。
</Info>

## 适用场景

内置引擎适合大多数用户：

- 无需额外依赖即可开箱即用。
- 能够很好地处理关键词搜索和向量搜索。
- 支持所有嵌入提供商。
- 混合搜索结合了两种检索方式的优势。

如果需要重新排序、查询扩展，或想为工作区之外的目录建立索引，请考虑切换到 [QMD](/zh-CN/concepts/memory-qmd)。

如果需要具有自动用户建模功能的跨会话记忆，请考虑使用 [Honcho](/zh-CN/concepts/memory-honcho)。

## 故障排查

**记忆搜索已禁用？**检查 `openclaw memory status`。如果未检测到提供商，
请显式设置一个提供商或添加 API key。

**未检测到本地提供商？**确认本地路径存在，然后运行：

```bash
openclaw memory status --deep --agent main
openclaw memory index --force --agent main
```

独立 CLI 命令和 Gateway 网关使用相同的 `local` 提供商 ID。
如需使用本地嵌入，请设置 `memory.search.provider: "local"`。

**结果已过时？**运行 `openclaw memory index --force` 以重建索引。在极少数边缘情况下，
监视器可能会遗漏更改。

**sqlite-vec 无法加载？**OpenClaw 会自动回退到进程内余弦
相似度计算。`openclaw memory status --deep` 会将本地
向量存储与嵌入提供商分别报告，因此 `Vector store:
unavailable` 指向 sqlite-vec 加载问题，而 `Embeddings: unavailable`
指向提供商/身份验证或模型就绪状态。请检查日志以了解具体的加载
错误。

## 配置

有关嵌入提供商设置、混合搜索调优（权重、MMR、时间
衰减）、批量索引、多模态记忆、sqlite-vec、额外路径以及所有
其他配置选项，请参阅
[记忆配置参考](/zh-CN/reference/memory-config)。

## 相关内容

- [记忆概览](/zh-CN/concepts/memory)
- [记忆搜索](/zh-CN/concepts/memory-search)
- [主动记忆](/zh-CN/concepts/active-memory)
