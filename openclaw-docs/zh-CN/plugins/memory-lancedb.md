---
read_when:
    - 你正在配置 memory-lancedb 插件
    - 你需要由 LanceDB 支持、具备自动回忆或自动捕获功能的长期记忆
    - 你正在使用与 OpenAI 兼容的本地嵌入模型，例如 Ollama
sidebarTitle: Memory LanceDB
summary: 配置官方外部 Memory LanceDB 插件，包括本地 Ollama 兼容嵌入模型
title: Memory LanceDB
x-i18n:
    generated_at: "2026-07-26T06:24:38Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: bdb7208925ac6c76430ee36dfcd9733041530e0f2ee175950b3cdb8010d67b24
    source_path: plugins/memory-lancedb.md
    workflow: 16
---

`memory-lancedb` 是一个官方外部插件，它使用 LanceDB 和向量搜索存储长期记忆。它可以在模型轮次前自动召回相关记忆，并在响应后自动捕获重要事实。

可将其用于本地向量数据库、兼容 OpenAI 的嵌入端点，或默认内置记忆后端之外的记忆存储。

## 安装

```bash
openclaw plugins install @openclaw/memory-lancedb
```

该插件发布在 npm 上；它并未内置于 OpenClaw 运行时镜像中。安装会写入插件条目、启用插件，并将 `plugins.slots.memory` 切换为 `memory-lancedb`。如果当前由另一个插件占用记忆槽位，系统会禁用该插件并发出警告。

<Note>
`memory-wiki` 等配套插件可以与 `memory-lancedb` 同时运行，但同一时间只能有一个插件占用活动记忆槽位。
</Note>

<Note>
LanceDB 的 `memory_recall` 不会获得 `memory.search.rememberAcrossConversations` 所使用的受保护私有对话记录授权。请通过[高级主动记忆](/zh-CN/concepts/active-memory#lancedb-memory)使用 LanceDB 的 `autoRecall` 或其 `memory_recall` 工具。当当前记忆提供商不支持跨对话记忆时，`openclaw doctor` 会进行报告。
</Note>

## 快速开始

```json5
{
  plugins: {
    slots: {
      memory: "memory-lancedb",
    },
    entries: {
      "memory-lancedb": {
        enabled: true,
        config: {
          embedding: {
            provider: "openai",
            model: "text-embedding-3-small",
          },
          autoRecall: true,
          autoCapture: false,
        },
      },
    },
  },
}
```

更改插件配置后重启 Gateway 网关，然后验证插件是否已加载：

```bash
openclaw gateway restart
openclaw plugins list
```

## 嵌入配置

`embedding` 是必需的，并且必须至少包含一个字段。`provider` 默认为 `openai`；`model` 默认为 `text-embedding-3-small`。

| 字段                  | 类型          | 说明                                                                    |
| ---------------------- | ------------- | ------------------------------------------------------------------------ |
| `embedding.provider`   | 字符串        | 适配器 ID，例如 `openai`、`github-copilot`、`ollama`。默认为 `openai`。 |
| `embedding.model`      | 字符串        | 默认为 `text-embedding-3-small`。                                        |
| `embedding.apiKey`     | 字符串        | 可选；支持 `${ENV_VAR}` 展开。                               |
| `embedding.baseUrl`    | 字符串        | 可选；支持 `${ENV_VAR}` 展开。                               |
| `embedding.dimensions` | 整数（>=1） | 不在内置表中的模型必须设置此项（见下文）。               |

有两种请求路径：

- **提供商适配器路径**（默认）：设置 `embedding.provider`，并省略
  `embedding.apiKey`/`embedding.baseUrl`。插件会通过 `memory-core` 所使用的同一套记忆嵌入适配器，解析提供商已配置的身份验证配置文件、环境变量或 `models.providers.<provider>.apiKey`。这是 `github-copilot`、`ollama` 以及其他任何支持嵌入的内置提供商所使用的路径。
- **直接兼容 OpenAI 的客户端路径**：不设置 `embedding.provider`（或设置为 `"openai"`），并设置 `embedding.apiKey` 和 `embedding.baseUrl`。此路径用于没有内置提供商适配器的原始兼容 OpenAI 的嵌入端点。

OpenAI Codex / ChatGPT OAuth 不是 OpenAI Platform 嵌入凭据。若要使用 OpenAI 嵌入，请使用 OpenAI API 密钥身份验证配置文件、`OPENAI_API_KEY` 或 `models.providers.openai.apiKey`。仅使用 OAuth 的用户应选择其他支持嵌入的提供商，例如 `github-copilot` 或 `ollama`。

```json5
{
  plugins: {
    entries: {
      "memory-lancedb": {
        enabled: true,
        config: {
          embedding: {
            provider: "github-copilot",
            model: "text-embedding-3-small",
          },
        },
      },
    },
  },
}
```

某些兼容 OpenAI 的嵌入端点会拒绝 `encoding_format` 参数；另一些则会忽略它，并始终返回 `number[]`。`memory-lancedb` 会在请求中省略 `encoding_format`，并接受浮点数组或采用 base64 编码的 float32 响应，因此两种响应格式都无需配置即可使用。

### 维度

OpenClaw 仅为 `text-embedding-3-small`（1536）和 `text-embedding-3-large`（3072）提供内置维度。其他任何模型都需要显式设置 `embedding.dimensions`，以便 LanceDB 创建向量列。例如，智谱 `embedding-3` 的维度为 2048：

```json5
{
  plugins: {
    entries: {
      "memory-lancedb": {
        enabled: true,
        config: {
          embedding: {
            apiKey: "${ZHIPU_API_KEY}",
            baseUrl: "https://open.bigmodel.cn/api/paas/v4",
            model: "embedding-3",
            dimensions: 2048,
          },
        },
      },
    },
  },
}
```

## Ollama 嵌入

使用内置 Ollama 提供商适配器路径（`embedding.provider: "ollama"`）。它会调用 Ollama 的原生 `/api/embed` 端点，并遵循与 [Ollama](/zh-CN/providers/ollama) 提供商相同的身份验证/基础 URL 规则。

```json5
{
  plugins: {
    slots: {
      memory: "memory-lancedb",
    },
    entries: {
      "memory-lancedb": {
        enabled: true,
        config: {
          embedding: {
            provider: "ollama",
            baseUrl: "http://127.0.0.1:11434",
            model: "mxbai-embed-large",
            dimensions: 1024,
          },
          recallMaxChars: 400,
          autoRecall: true,
          autoCapture: false,
        },
      },
    },
  },
}
```

`mxbai-embed-large` 不在内置维度表中，因此必须设置 `dimensions`。对于小型本地嵌入模型，如果本地服务器返回上下文长度错误，请降低 `recallMaxChars`。

## 召回和捕获限制

| 设置           | 默认值 | 范围                        | 适用于                                                 |
| ----------------- | ------- | ---------------------------- | ---------------------------------------------------------- |
| `recallMaxChars`  | `1000`  | 100-10000                    | 为召回而发送到嵌入 API 的文本。                 |
| `captureMaxChars` | `500`   | 100-10000                    | 符合自动捕获条件的消息长度。                  |
| `customTriggers`  | `[]`    | 0-50 项，每项 <=100 个字符 | 使自动捕获考虑某条消息的字面短语。 |

`recallMaxChars` 会限制 `before_prompt_build` 自动召回查询、`memory_recall` 工具、`memory_forget` 查询路径和 `openclaw ltm
search`。自动召回会嵌入该轮次中的最新用户消息；仅当不存在用户消息时，才回退到完整提示词，从而避免将渠道元数据和大型提示词块包含在嵌入请求中。

`captureMaxChars` 用于判断该轮次 `agent_end` 事件中的用户消息是否足够短，可以纳入自动捕获考虑；它不会影响召回查询。

`customTriggers` 可添加不使用正则表达式的字面自动捕获短语。内置触发器涵盖常见的英语、捷克语、中文、日语和韩语记忆短语（`remember`、`prefer`、`记住`、`覚えて`、`기억해` 等）。

自动捕获还会拒绝看起来像信封/传输元数据、提示词注入载荷或已注入的 `<relevant-memories>` 上下文的文本，并将每个智能体轮次捕获的记忆数量限制为最多 3 条。

每条记忆都归一个智能体所有。召回、重复检测、捕获、列出、原始查询和删除操作都会先强制检查该所有者，再返回或修改行。如果某个智能体的 `agents.entries.*` 条目中含有 `memory.search.enabled: false`，或者继承了已禁用的顶层搜索设置，那么即使插件级 `autoRecall`/`autoCapture` 标志已开启，该智能体也不会获得 `memory_recall`、`memory_store` 或 `memory_forget` 工具，也不会参与自动召回或捕获。

## 命令

只要安装了 `memory-lancedb`，它就会注册 `ltm` CLI 命名空间（并非仅在它占用活动记忆槽位时注册）：

```bash
openclaw ltm list [--agent <id>] [--limit <n>] [--order-by-created-at]
openclaw ltm search <query> [--agent <id>] [--limit <n>]
openclaw ltm stats [--agent <id>]
```

`ltm query` 会直接针对 LanceDB 表运行非向量查询：

```bash
openclaw ltm query --agent research --cols id,text,createdAt --limit 20
openclaw ltm query --filter "category = 'preference'" --order-by createdAt:desc
```

| 标志                              | 默认值                                 | 说明                                                                                                                                     |
| --------------------------------- | --------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------- |
| `--agent <id>`                    | 已配置的默认智能体                | 选择私有智能体命名空间。可用于 `list`、`search`、`query` 和 `stats`。                                                 |
| `--cols <columns>`                | `id,text,importance,category,createdAt` | 以逗号分隔的列允许列表。                                                                                                         |
| `--filter <condition>`            | 无                                    | 针对输出列的一项比较，例如 `category = 'preference'` 或 `importance >= 0.8`。字符串值必须用引号括起。             |
| `--limit <n>`                     | `10`                                    | 正整数。                                                                                                                         |
| `--order-by <column>:<asc\|desc>` | 无                                    | 筛选运行后在内存中排序；排序列会自动添加到投影中，如果未请求该列，则会从输出中移除。 |

智能体会从活动记忆插件获得三个工具：

- `memory_recall`：对已存储的记忆执行向量搜索。
- `memory_store`：保存事实、偏好、决策或实体（拒绝看起来像提示词注入载荷的文本；跳过近似重复的存储内容）。
- `memory_forget`：按 `memoryId` 或 `query` 删除（若单个匹配项的得分高于 90%，则自动删除；否则列出候选 ID 以消除歧义）。

## 存储

LanceDB 数据默认存储在 `~/.openclaw/memory/lancedb`。可通过 `dbPath` 覆盖：

```json5
{
  plugins: {
    entries: {
      "memory-lancedb": {
        enabled: true,
        config: {
          dbPath: "~/.openclaw/memory/lancedb",
          embedding: {
            apiKey: "${OPENAI_API_KEY}",
            model: "text-embedding-3-small",
          },
        },
      },
    },
  },
}
```

该插件维护一个 LanceDB 表，并在每行中存储规范化的智能体所有者。这是存储边界，而不是搜索后筛选器：智能体所有权会在向量排名之前应用，并包含在列表、查询、计数和删除谓词中。`ltm query --filter` 接受一项针对公共输出列且经过验证的比较。存储会将该比较与强制所有者谓词分开构建，因此筛选器无法将查询范围扩大到另一个智能体。

在引入按智能体所有权之前创建的数据库不包含可靠的行来源信息。升级时，`openclaw doctor --fix` 会将这些旧版行一次性分配给已配置的默认智能体。在该迁移完成之前，运行时访问会采用故障关闭策略；其他智能体绝不会继承旧的共享行。

`storageOptions` 接受用于 LanceDB 存储后端（例如 S3 兼容的对象存储）的字符串键值对，并支持 `${ENV_VAR}` 展开：

```json5
{
  plugins: {
    entries: {
      "memory-lancedb": {
        enabled: true,
        config: {
          dbPath: "s3://memory-bucket/openclaw",
          storageOptions: {
            access_key: "${AWS_ACCESS_KEY_ID}",
            secret_key: "${AWS_SECRET_ACCESS_KEY}",
            endpoint: "${AWS_ENDPOINT_URL}",
          },
          embedding: {
            apiKey: "${OPENAI_API_KEY}",
            model: "text-embedding-3-small",
          },
        },
      },
    },
  },
}
```

## 运行时依赖和平台支持

`memory-lancedb` 依赖由插件包（而非 OpenClaw 核心发行包）负责管理的原生 `@lancedb/lancedb` 软件包。Gateway 网关启动时不会修复插件依赖；如果缺少原生依赖或加载失败，请重新安装或更新插件包，然后重启 Gateway 网关。

`@lancedb/lancedb` 不提供适用于 `darwin-x64`（Intel Mac）的原生构建。在该平台上，插件会在加载时记录 LanceDB 不可用；请使用默认记忆后端、在受支持的平台/架构上运行 Gateway 网关，或禁用 `memory-lancedb`。

## 故障排除

### 输入长度超过上下文长度

嵌入模型拒绝了召回查询：

```text
memory-lancedb: 召回失败：错误：400 输入长度超过上下文长度
```

降低 `recallMaxChars`，然后重启 Gateway 网关：

```json5
{
  plugins: {
    entries: {
      "memory-lancedb": {
        config: {
          recallMaxChars: 400,
        },
      },
    },
  },
}
```

对于 Ollama，还应使用其原生嵌入端点验证能否从 Gateway 网关主机访问嵌入服务器：

```bash
curl http://127.0.0.1:11434/api/embed \
  -H "Content-Type: application/json" \
  -d '{"model":"mxbai-embed-large","input":"hello"}'
```

### 不受支持的嵌入模型

如果未设置 `embedding.dimensions`，则仅已知内置 OpenAI 嵌入模型的维度（`text-embedding-3-small`、`text-embedding-3-large`）。对于任何其他模型，请将 `embedding.dimensions` 设置为该模型报告的向量大小。

### 插件已加载，但未显示任何记忆

确认 `plugins.slots.memory` 指向 `memory-lancedb`，然后运行：

```bash
openclaw ltm stats
openclaw ltm search "recent preference"
```

如果已禁用 `autoCapture`，插件仍会召回现有记忆，但不会自动存储新记忆。请使用 `memory_store` 工具，或启用 `autoCapture`。

## 相关内容

- [记忆概览](/zh-CN/concepts/memory)
- [主动记忆](/zh-CN/concepts/active-memory)
- [记忆搜索](/zh-CN/concepts/memory-search)
- [Memory Wiki](/zh-CN/plugins/memory-wiki)
- [Ollama](/zh-CN/providers/ollama)
