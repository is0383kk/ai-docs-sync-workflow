---
read_when:
    - 你想要配置记忆搜索提供商或嵌入模型
    - 你想设置 QMD 后端
    - 你想启用混合搜索、MMR 或时间衰减
    - 你想启用多模态记忆索引
sidebarTitle: Memory config
summary: 记忆搜索提供商、检索模式、QMD 和多模态索引
title: 记忆配置参考
x-i18n:
    generated_at: "2026-07-26T06:27:43Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 91f843b1516093c49e18b3d659ab24ea9cb7be32aaaac722205eca8bc3f2ca5b
    source_path: reference/memory-config.md
    workflow: 16
---

此页面列出了 OpenClaw 记忆搜索的所有配置项。有关概念性概览，请参阅：

<CardGroup cols={2}>
  <Card title="记忆概览" href="/zh-CN/concepts/memory">
    记忆的工作原理。
  </Card>
  <Card title="内置引擎" href="/zh-CN/concepts/memory-builtin">
    默认 SQLite 后端。
  </Card>
  <Card title="QMD 引擎" href="/zh-CN/concepts/memory-qmd">
    本地优先的边车服务。
  </Card>
  <Card title="记忆搜索" href="/zh-CN/concepts/memory-search">
    搜索流程和调优。
  </Card>
  <Card title="主动记忆" href="/zh-CN/concepts/active-memory">
    用于交互式会话的记忆子智能体。
  </Card>
</CardGroup>

所有共享记忆设置都位于 `openclaw.json` 的顶层 `memory` 下。搜索默认值使用 `memory.search`；每个智能体的搜索覆盖项使用 `agents.entries.*.memory.search`。

<Note>
对于推荐的个人智能体工作流，请使用
`memory.search.rememberAcrossConversations`。高级主动记忆的目标选择、
模型、提示词和延迟控制位于 `plugins.entries.active-memory` 下。

有关两种激活路径、记录持久化和安全上线指南，请参阅[主动记忆](/zh-CN/concepts/active-memory)。
</Note>

---

## 跨对话记忆

| 键                            | 类型      | 默认值                                                     | 描述                                                                         |
| ----------------------------- | --------- | ---------------------------------------------------------- | ------------------------------------------------------------------------------ |
| `rememberAcrossConversations` | `boolean` | 个人安装时开启；配置私信隔离时关闭 | 使用此智能体其他已识别私密对话中的相关上下文。 |

如果只有受信任的个人智能体应使用跨对话的记录回忆，请为该智能体配置：

```json5
{
  agents: {
    entries: {
      personal: {
        memory: {
          search: {
            rememberAcrossConversations: true,
          },
        },
      },
    },
  },
}
```

该值遵循常规 `memory.search` 继承规则，并可按智能体覆盖。未设置时，仅当全局
`session.dmScope` 未设置或为 `"main"`，且没有任何绑定包含 `session.dmScope`
覆盖项时，才默认开启。配置任何私信隔离都会使其默认关闭。显式设置的 `true` 或
`false` 始终优先。启用此设置意味着为会话记录建立索引，并将
`sessions` 添加到智能体解析后的记忆来源中。使用 QMD 时，还会启用该智能体的会话导出；此模式不需要单独配置
`memory.qmd.sessions.enabled`。

OpenClaw 的内置记忆提供商通过内置和 QMD 后端支持此受保护路径。其他记忆提供商仍可使用其自身的回忆钩子和高级主动记忆工具，但除非当前提供商支持受保护的私密记录回忆，否则会跳过此设置。
`openclaw doctor` 会报告不受支持的提供商，或报告显式主动记忆
`toolsAllow` 列表遗漏了 `memory_search`。

此检索边界比常规会话搜索更窄：

- 只有同一智能体已识别的私密对话才符合条件
- 排除当前正在回复的对话
- 群组和频道均不能作为来源或目标
- 遇到未知对话类型时以关闭状态失败
- 沙箱隔离的回忆无法使用特殊的跨对话授权

此设置不会更改 `tools.sessions.visibility`、会话键、记录存储、投递路由，也不会更改
`sessions_list`、`sessions_history` 和 `sessions_send` 的权限。主动记忆会执行有界的只读检索过程；检索不可用或超时不会阻止回复。

---

## 提供商选择

| 键         | 类型      | 默认值           | 描述                                                                                                                                                                                                                                                                                        |
| ---------- | --------- | ---------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `enabled`  | `boolean` | `true`           | 启用或禁用记忆搜索                                                                                                                                                                                                                                                             |
| `provider` | `string`  | `"openai"`       | 嵌入适配器 ID，例如 `bedrock`、`deepinfra`、`gemini`、`github-copilot`、`local`、`mistral`、`ollama`、`openai`、`openai-compatible` 或 `voyage`；也可以是已配置的 `models.providers.<id>`，其 `api` 指向记忆嵌入适配器或兼容 OpenAI 的模型 API |
| `model`    | `string`  | 提供商默认值 | 嵌入模型名称                                                                                                                                                                                                                                                                        |
| `fallback` | `string`  | `"none"`         | 主适配器失败时使用的后备适配器 ID                                                                                                                                                                                                                                                  |

未设置 `provider` 时，OpenClaw 使用 OpenAI 嵌入。显式设置 `provider`
以使用 Bedrock、DeepInfra、Gemini、GitHub Copilot、Mistral、Ollama、Voyage、本地 GGUF 模型或兼容 OpenAI 的 `/v1/embeddings` 端点。
仍使用 `provider: "auto"` 的旧版配置会解析为 `openai`。

<Warning>
更改嵌入提供商、模型、提供商设置、来源、作用域、分块方式或分词器可能导致现有 SQLite 向量索引不兼容。
OpenClaw 会暂停向量搜索并报告索引标识警告，而不是自动重新嵌入所有内容。准备就绪后，使用
`openclaw memory status --index --agent <id>` 或
`openclaw memory index --force --agent <id>` 重建。
</Warning>

当 `provider` 未设置、存在旧版 `provider: "auto"`，或
`provider: "none"` 有意选择仅 FTS 模式时，嵌入不可用的情况下，记忆回忆仍可使用词法 FTS 排序。

显式指定的非本地提供商会以关闭状态失败。如果将 `memory.search.provider` 设置为具体的远程后端提供商，例如 Bedrock、DeepInfra、Gemini、GitHub Copilot、LM Studio、Mistral、Ollama、OpenAI、Voyage 或兼容 OpenAI 的自定义提供商，而该提供商在运行时不可用，则 `memory_search`
会返回不可用结果，而不会静默改用仅 FTS 回忆。请修复提供商或身份验证配置、切换到可访问的提供商，或者如果希望有意使用仅 FTS 回忆，请设置
`provider: "none"`。

### 自定义提供商 ID

`memory.search.provider` 可以指向自定义 `models.providers.<id>` 条目，用于 `ollama` 等记忆专用提供商适配器，或 `openai-responses` / `openai-completions` 等兼容 OpenAI 的模型 API。OpenClaw 会为嵌入适配器解析该提供商的 `api` 所有者，同时保留自定义提供商 ID，以便处理端点、身份验证和模型前缀。这样，多 GPU 或多主机设置就能将记忆嵌入专用于特定本地端点：

```json5
{
  models: {
    providers: {
      "ollama-5080": {
        api: "ollama",
        baseUrl: "http://gpu-box.local:11435",
        apiKey: "ollama-local",
        models: [{ id: "qwen3-embedding:0.6b", name: "Qwen3 Embedding 0.6B" }],
      },
    },
  },
  memory: {
    search: {
      provider: "ollama-5080",
      model: "qwen3-embedding:0.6b",
    },
  },
}
```

### API 密钥解析

远程嵌入需要 API 密钥。Bedrock 则使用 AWS SDK 默认凭证链（实例角色、SSO、访问密钥或 Bedrock API 密钥）。

| 提供商         | 环境变量                                             | 配置键                              |
| -------------- | --------------------------------------------------- | ----------------------------------- |
| Bedrock        | AWS 凭证链或 `AWS_BEARER_TOKEN_BEDROCK` | 不需要 API 密钥                     |
| DeepInfra      | `DEEPINFRA_API_KEY`                                 | `models.providers.deepinfra.apiKey` |
| Gemini         | `GEMINI_API_KEY`                                    | `models.providers.google.apiKey`    |
| GitHub Copilot | `COPILOT_GITHUB_TOKEN`、`GH_TOKEN`、`GITHUB_TOKEN`  | 通过设备登录获取身份验证配置文件       |
| Mistral        | `MISTRAL_API_KEY`                                   | `models.providers.mistral.apiKey`   |
| Ollama         | `OLLAMA_API_KEY`（占位符）                      | --                                  |
| OpenAI         | `OPENAI_API_KEY`                                    | `models.providers.openai.apiKey`    |
| Voyage         | `VOYAGE_API_KEY`                                    | `models.providers.voyage.apiKey`    |

<Note>
Codex OAuth 仅涵盖聊天/补全，不能满足嵌入请求。
</Note>

---

## 远程端点配置

对于不应继承全局 OpenAI 聊天凭证的通用兼容 OpenAI 的
`/v1/embeddings` 服务器，请使用 `provider: "openai-compatible"`。

<ParamField path="remote.baseUrl" type="string">
  自定义 API 基础 URL。
</ParamField>
<ParamField path="remote.apiKey" type="string">
  覆盖 API 密钥。
</ParamField>
<ParamField path="remote.headers" type="object">
  额外的 HTTP 标头（与提供商默认值合并）。
</ParamField>

```json5
{
  memory: {
    search: {
      provider: "openai-compatible",
      model: "text-embedding-3-small",
      remote: {
        baseUrl: "https://api.example.com/v1/",
        apiKey: "YOUR_KEY",
      },
    },
  },
}
```

---

## 提供商特定配置

<AccordionGroup>
  <Accordion title="Gemini">
    | 键                     | 类型     | 默认值                 | 描述                                      |
    | ---------------------- | -------- | ---------------------- | ------------------------------------------- |
    | `model`                | `string` | `gemini-embedding-001` | 还支持 `gemini-embedding-2-preview` |
    | `outputDimensionality` | `number` | `3072`                 | 对于 Embedding 2：768、1536 或 3072        |

    <Warning>
    更改模型或 `outputDimensionality` 会改变索引标识。OpenClaw
    会暂停向量搜索，直到你显式重建记忆索引。
    </Warning>

  </Accordion>
  <Accordion title="兼容 OpenAI 的输入类型">
    兼容 OpenAI 的嵌入端点可以选择使用提供商特定的 `input_type` 请求字段。这适用于要求查询嵌入和文档嵌入使用不同标签的非对称嵌入模型。

    | 键                  | 类型     | 默认值 | 描述                                                   |
    | ------------------- | -------- | ------- | -------------------------------------------------------- |
    | `inputType`         | `string` | 未设置   | 查询嵌入和文档嵌入共享的 `input_type`   |
    | `queryInputType`    | `string` | 未设置   | 查询时的 `input_type`；覆盖 `inputType`          |
    | `documentInputType` | `string` | 未设置   | 索引/文档的 `input_type`；覆盖 `inputType`      |

    ```json5
    {
      memory: {
        search: {
          provider: "openai-compatible",
          remote: {
            baseUrl: "https://embeddings.example/v1",
            apiKey: "${EMBEDDINGS_API_KEY}",
          },
          model: "asymmetric-embedder",
          queryInputType: "query",
          documentInputType: "passage",
        },
      },
    }
    ```

    更改这些值会影响提供商批量索引的嵌入缓存标识；如果上游模型对这些标签的处理方式不同，则更改后应重新索引记忆。

  </Accordion>
  <Accordion title="Bedrock">
    ### Bedrock 嵌入配置

    Bedrock 使用 AWS SDK 默认凭证链以及经 OpenClaw 检查的持有者令牌，因此配置中不会存储 API 密钥。如果 OpenClaw 在具有 Bedrock 使用权限的实例角色所对应的 EC2 上运行，只需设置提供商和模型：

    ```json5
    {
      memory: {
        search: {
          provider: "bedrock",
          model: "amazon.titan-embed-text-v2:0",
        },
      },
    }
    ```

    | 键                     | 类型     | 默认值                          | 描述                            |
    | ---------------------- | -------- | ------------------------------- | -------------------------------- |
    | `model`                | `string` | `amazon.titan-embed-text-v2:0` | 任意 Bedrock 嵌入模型 ID        |
    | `outputDimensionality` | `number` | 模型默认值                      | 对于 Titan V2：256、512 或 1024 |

    **支持的模型**（含系列检测和默认维度）：

    | 模型 ID                                     | 提供商      | 默认维度      | 可配置维度                      |
    | ------------------------------------------- | ---------- | ------------- | -------------------------- |
    | `amazon.titan-embed-text-v2:0`             | Amazon     | 1024         | 256, 512, 1024             |
    | `amazon.titan-embed-text-v1`               | Amazon     | 1536         | --                          |
    | `amazon.titan-embed-g1-text-02`            | Amazon     | 1536         | --                          |
    | `amazon.titan-embed-image-v1`              | Amazon     | 1024         | --                          |
    | `amazon.nova-2-multimodal-embeddings-v1:0` | Amazon     | 1024         | 256, 384, 1024, 3072       |
    | `cohere.embed-english-v3`                  | Cohere     | 1024         | --                          |
    | `cohere.embed-multilingual-v3`             | Cohere     | 1024         | --                          |
    | `cohere.embed-v4:0`                        | Cohere     | 1536         | 256, 384, 512, 768, 1024, 1536 |
    | `twelvelabs.marengo-embed-3-0-v1:0`        | TwelveLabs | 512          | --                          |
    | `twelvelabs.marengo-embed-2-7-v1:0`        | TwelveLabs | 1024         | --                          |

    带吞吐量后缀的变体（例如 `amazon.titan-embed-text-v1:2:8k`）和带区域前缀的推理配置文件 ID（例如 `us.amazon.titan-embed-text-v2:0`）会继承基础模型的配置。

    **区域：**按以下顺序解析：`memory.search.remote.baseUrl` 覆盖值、`models.providers.amazon-bedrock.baseUrl` 配置、`AWS_REGION`、`AWS_DEFAULT_REGION`，最后采用默认值 `us-east-1`。

    **身份验证：**OpenClaw 首先检查 `AWS_ACCESS_KEY_ID` + `AWS_SECRET_ACCESS_KEY` 或 `AWS_BEARER_TOKEN_BEDROCK`，然后回退到标准 AWS SDK 默认凭证提供商链：

    1. 环境变量（`AWS_ACCESS_KEY_ID` + `AWS_SECRET_ACCESS_KEY`），除非还设置了 `AWS_PROFILE`
    2. SSO（仅当配置了 SSO 字段时）
    3. 共享凭证和配置文件（`fromIni`，包括 `AWS_PROFILE`）
    4. 凭证进程（AWS 配置文件中的 `credential_process`）
    5. Web 身份令牌凭证
    6. ECS 或 EC2 实例元数据凭证

    **IAM 权限：**IAM 角色或用户需要：

    ```json
    {
      "Effect": "Allow",
      "Action": "bedrock:InvokeModel",
      "Resource": "*"
    }
    ```

    为遵循最小权限原则，请将 `InvokeModel` 限定到特定模型：

    ```text
    arn:aws:bedrock:*::foundation-model/amazon.titan-embed-text-v2:0
    ```

  </Accordion>
  <Accordion title="本地（GGUF + llama.cpp）">
    | 键                    | 类型               | 默认值                  | 描述                                                                                                                                                                                                                                                                                                          |
    | --------------------- | ------------------ | ----------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
    | `local.modelPath`     | `string`           | 自动下载                | GGUF 模型文件的路径                                                                                                                                                                                                                                                                                              |
    | `local.modelCacheDir` | `string`           | node-llama-cpp 默认值   | 已下载模型的缓存目录                                                                                                                                                                                                                                                                                      |
    | `local.contextSize`   | `number \| "auto"` | `4096`                 | 嵌入上下文的上下文窗口大小。4096 可覆盖典型分块（128-512 个令牌），同时限制非权重显存占用。在资源受限的主机上可降至 1024-2048。`"auto"` 使用模型训练时的最大值——不建议用于 8B+ 模型（Qwen3-Embedding-8B：最高 40 960 个令牌可能会将显存占用推高至约 32 GB）。 |

    请先安装官方 llama.cpp 提供商：`openclaw plugins install @openclaw/llama-cpp-provider`。
    默认模型：`embeddinggemma-300m-qat-Q8_0.gguf`（约 0.6 GB，自动下载）。源码检出仍需要原生构建批准：先执行 `pnpm approve-builds`，再执行 `pnpm rebuild node-llama-cpp`。

    使用独立 CLI 验证 Gateway 网关所使用的同一提供商路径：

    ```bash
    openclaw memory status --deep --agent main
    openclaw memory index --force --agent main
    ```

    数值型 `local.contextSize` 值还会影响 node-llama-cpp 的 GPU 层自动放置，以便同时容纳模型权重和所请求的嵌入上下文。运行时加载后，`openclaw memory status --deep` 会报告最近一次已知的 llama.cpp 后端、设备、卸载情况、所请求的上下文以及带时间戳的内存信息；被动状态检查不会加载模型。

    对于本地 GGUF 嵌入，请显式设置 `provider: "local"`。显式本地配置支持 `hf:` 和 HTTP(S) 模型引用（通过 node-llama-cpp 的模型解析），但它们不会更改默认提供商。

  </Accordion>
</AccordionGroup>

## 索引行为

记忆引擎负责同步、批处理、监视以及压缩后的
索引启发式策略。OpenClaw 使用持续维护的默认值启用这些行为，
而不会公开按安装实例设置的时序开关。

## 混合搜索配置

以下各项均位于 `memory.search.query` 下：

| 键           | 类型     | 默认值 | 描述                                   |
| ------------ | -------- | ------- | ----------------------------------------- |
| `maxResults` | `number` | `6`     | 注入前返回的最大记忆命中数               |
| `minScore`   | `number` | `0.35`  | 纳入命中结果所需的最低相关性分数         |

混合检索保持启用；内置引擎策略仍会禁用 MMR 和时间衰减。

### 完整示例

```json5
{
  memory: {
    search: {
      query: {
        maxResults: 6,
        minScore: 0.35,
      },
    },
  },
}
```

---

## 其他记忆路径

| 键           | 类型       | 描述                                   |
| ------------ | ---------- | ---------------------------------------- |
| `extraPaths` | `string[]` | 要索引的其他目录或文件                 |

```json5
{
  memory: {
    search: {
      extraPaths: ["../team-docs", "/srv/shared-notes"],
    },
  },
}
```

路径可以是绝对路径，也可以相对于工作区。系统会递归扫描目录中的 `.md` 文件。符号链接的处理方式取决于当前后端：内置引擎会跳过符号链接，而 QMD 则遵循底层 QMD 扫描器的行为。

对于 Agent 范围内的跨 Agent 记录搜索，请使用 `agents.entries.*.memory.search.qmd.extraCollections`，而不是 `memory.qmd.paths`。这些额外集合遵循相同的 `{ path, name, pattern? }` 结构，但它们会按 Agent 合并；当路径指向当前工作区之外时，还可以保留显式共享名称。如果同一解析路径同时出现在 `memory.qmd.paths` 和 `memory.search.qmd.extraCollections` 中，QMD 会保留第一个条目并跳过重复项。

---

## 多模态记忆（Gemini）

使用 Gemini Embedding 2 将图像和音频与 Markdown 一同索引：

| 键                        | 类型       | 默认值     | 描述                                   |
| ------------------------- | ---------- | ---------- | -------------------------------------- |
| `multimodal.enabled`      | `boolean`  | `false`    | 启用多模态索引                         |
| `multimodal.modalities`   | `string[]` | --         | `["image"]`、`["audio"]` 或 `["all"]` |
| `multimodal.maxFileBytes` | `number`   | `10485760` | 索引文件的最大大小（10 MiB）           |

<Note>
仅适用于 `extraPaths` 中的文件。默认记忆根目录仍仅支持 Markdown。需要 `gemini-embedding-2-preview`。`fallback` 必须为 `"none"`。
</Note>

支持的格式：`.jpg`、`.jpeg`、`.png`、`.webp`、`.gif`、`.heic`、`.heif`（图像）；`.mp3`、`.wav`、`.ogg`、`.opus`、`.m4a`、`.aac`、`.flac`（音频）。

---

## 嵌入缓存

| 键              | 类型      | 默认值 | 描述                          |
| --------------- | --------- | ------- | -------------------------------- |
| `cache.enabled` | `boolean` | `true`  | 在 SQLite 中缓存分块嵌入       |

避免在重新索引或更新记录期间对未更改的文本重复生成嵌入。

---

## 批量索引

| 键                           | 类型      | 默认值 | 描述                         |
| ---------------------------- | --------- | ------- | -------------------------- |
| `remote.nonBatchConcurrency` | `number`  | `4`     | 并行内联嵌入               |
| `remote.batch.enabled`       | `boolean` | `false` | 启用批量嵌入 API           |

适用于 `gemini`、`openai` 和 `voyage`。对于大规模回填，OpenAI 批处理通常速度最快且成本最低。

并发、轮询和超时行为由提供商负责。

---

## 会话记忆搜索

索引会话记录，并通过 `memory_search` 提供：

| 键                            | 类型       | 默认值       | 描述                                   |
| ----------------------------- | ---------- | ------------ | ---------------------------------------- |
| `rememberAcrossConversations` | `boolean`  | `false`      | 允许跨私密对话进行回忆                   |
| `sources`                     | `string[]` | `["memory"]` | 添加 `"sessions"` 以包含会话记录  |

<Warning>
会话索引是可选功能，并且异步运行。结果可能略有滞后。会话日志存储在磁盘上，因此应将文件系统访问权限视为信任边界。
</Warning>

由模型调用的普通会话记录搜索遵循
[`tools.sessions.visibility`](/zh-CN/gateway/config-tools#toolssessions)。默认的
`tree` 可见性会公开当前会话、由其派生的会话，以及
通过环境群组感知功能监视的同一智能体群组会话。其他
不相关的会话需要 `agent` 可见性（仅当还需要跨智能体
回忆，并且智能体间策略允许时，才需要 `all` 可见性）。

`rememberAcrossConversations` 不会扩大该设置的范围。它提供一项
仅限运行时的独立授权，在有界的主动记忆处理过程中，仅允许访问同一智能体的私有
会话记录。

以下示例将这些设置放在顶层 `memory.search` 下。如果只有一个
智能体应索引和搜索会话记录，也可以在该智能体的 `memory.search` 覆盖配置中
应用等效设置。

对于同一智能体从 Gateway 网关到私信的回忆：

<Tabs>
  <Tab title="内置后端">
    ```json5
    {
      memory: {
        search: {
          experimental: { sessionMemory: true },
          sources: ["memory", "sessions"],
        },
      },
      tools: {
        sessions: { visibility: "agent" },
      },
    }
    ```
  </Tab>
  <Tab title="QMD 后端">
    ```json5
    {
      memory: {
        backend: "qmd",
        search: {
          experimental: { sessionMemory: true },
          sources: ["memory", "sessions"],
        },
        qmd: {
          sessions: { enabled: true },
        },
      },
      tools: {
        sessions: { visibility: "agent" },
      },
    }
    ```
  </Tab>
</Tabs>

使用 QMD 时，仅设置 `sources: ["sessions"]` 不会将会话记录导出到 QMD。还需设置
`memory.qmd.sessions.enabled: true`。更高层级的
`rememberAcrossConversations: true` 设置是例外：它隐含启用该智能体所需的
QMD 会话导出。隐含导出始终保持私有：
它们始终使用默认的内部导出位置（配置的
`sessions.exportDir` 仅适用于显式导出），仅在该智能体进行跨对话回忆时
被搜索，并且普通的 `memory_get`
无法读取它们。显式设置
`memory.qmd.sessions.enabled: true` 会保持其现有行为，使
导出的会话记录成为普通记忆语料库的一部分。

---

## SQLite 向量加速（sqlite-vec）

| 键                           | 类型      | 默认值 | 说明                              |
| ---------------------------- | --------- | ------- | --------------------------------- |
| `store.vector.enabled`       | `boolean` | `true`  | 使用 sqlite-vec 执行向量查询 |
| `store.vector.extensionPath` | `string`  | 内置 | 覆盖 sqlite-vec 路径          |

当 sqlite-vec 不可用时，OpenClaw 会自动回退到进程内余弦相似度计算。

---

## 索引存储

内置记忆索引存储在每个智能体的 OpenClaw SQLite 数据库中，路径为
`agents/<agentId>/agent/openclaw-agent.sqlite`。

| 键                    | 类型     | 默认值      | 说明                                      |
| --------------------- | -------- | ----------- | ----------------------------------------- |
| `store.fts.tokenizer` | `string` | `unicode61` | FTS5 分词器（`unicode61` 或 `trigram`） |

---

## QMD 后端配置

将 `memory.backend = "qmd"` 设为启用。所有 QMD 设置均位于 `memory.qmd` 下：

| 键                       | 类型      | 默认值   | 说明                                                                                         |
| ------------------------ | --------- | -------- | -------------------------------------------------------------------------------------------- |
| `command`                | `string`  | `qmd`    | QMD 可执行文件路径；当服务的 `PATH` 与你的 shell 不同时，请设置绝对路径 |
| `searchMode`             | `string`  | `search` | 搜索命令：`search`、`vsearch`、`query`                                         |
| `rerank`                 | `boolean` | --       | 配合 `searchMode: "query"` 和 QMD 2.1+ 设为 `false`，以跳过 QMD 重排序          |
| `includeDefaultMemory`   | `boolean` | `true`   | 自动索引 `MEMORY.md` + `memory/**/*.md`                                             |
| `paths[]`                | `array`   | --       | 额外路径：`{ name, path, pattern? }`                                               |
| `sessions.enabled`       | `boolean` | `false`  | 将会话记录导出到 QMD                                                   |
| `sessions.retentionDays` | `number`  | --       | 会话记录保留期限                                                                  |
| `sessions.exportDir`     | `string`  | --       | 导出目录                                                                      |

`searchMode: "search"` 仅使用词法/BM25。在该模式下，OpenClaw 不会运行语义向量就绪探测或 QMD 嵌入维护，包括在 `memory status --deep` 期间；`vsearch` 和 `query` 仍然要求 QMD 向量就绪并具备嵌入。

`rerank: false` 仅更改 QMD 的 `query` 模式，并要求 QMD 2.1 或更高版本。在直接 CLI 模式下，OpenClaw 会传递 `--no-rerank`；在由 mcporter 支持的 MCP 模式下，它会将 `rerank: false` 传递给 QMD 的统一查询工具。保持未设置即可使用 QMD 默认的查询重排序行为。

OpenClaw 优先使用当前的 QMD 集合和 MCP 查询结构，但会在需要时尝试兼容的集合模式标志和旧版 MCP 工具名称，以保持旧版 QMD 可用。当 QMD 声明支持多个集合筛选器时，同源集合会由一个 QMD 进程搜索；旧版 QMD 构建仍使用按集合处理的兼容路径。“同源”是指持久记忆集合（默认记忆文件加自定义路径）会被分为一组，而会话记录集合仍作为单独的一组，以便来源多样化仍可同时使用这两类输入。

<Note>
QMD 模型覆盖配置保留在 QMD 端，而不在 OpenClaw 配置中。如果需要全局覆盖 QMD 的模型，请在 Gateway 网关运行时环境中设置 `QMD_EMBED_MODEL`、`QMD_RERANK_MODEL` 和 `QMD_GENERATE_MODEL` 等环境变量。
</Note>

<AccordionGroup>
  <Accordion title="限制">
    | 键                        | 类型     | 默认值 | 说明                       |
    | --------------------------- | -------- | ------- | ------------------------------ |
    | `limits.maxResults`       | `number` | `4`     | 最大搜索结果数         |
    | `limits.maxSnippetChars`  | `number` | `450`   | 限制片段长度       |
    | `limits.maxInjectedChars` | `number` | `2200`  | 限制注入字符总数 |
    | `limits.timeoutMs`        | `number` | `4000`  | 使用 QMD 后端进行搜索时的 QMD 命令超时时间，包括 `memory_search`；设置、同步、内置回退和补充工作仍使用默认工具截止时间 |
  </Accordion>
  <Accordion title="范围">
    控制哪些会话可以接收 QMD 搜索结果。其结构与 [`session.sendPolicy`](/zh-CN/gateway/config-agents#session) 相同：

    ```json5
    {
      memory: {
        qmd: {
          scope: {
            default: "deny",
            rules: [{ action: "allow", match: { chatType: "direct" } }],
          },
        },
      },
    }
    ```

    随附的默认设置仅允许私信/直接会话，拒绝群组和其他渠道类型。`match.keyPrefix` 匹配规范化的会话键；`match.rawKeyPrefix` 匹配包括 `agent:<id>:` 在内的原始键。

  </Accordion>
  <Accordion title="引用">
    `memory.citations` 适用于所有后端：

    | 值               | 行为                                                   |
    | ------------------ | ------------------------------------------------------ |
    | `auto`（默认） | 在片段中包含 `Source: <path#line>` 页脚    |
    | `on`             | 始终包含页脚                               |
    | `off`            | 省略页脚（路径仍在内部传递给智能体） |

  </Accordion>
</AccordionGroup>

QMD 在首次使用记忆时延迟初始化；其适配器负责刷新和嵌入调度。

### 完整 QMD 示例

```json5
{
  memory: {
    backend: "qmd",
    citations: "auto",
    qmd: {
      includeDefaultMemory: true,
      update: { interval: "5m", debounceMs: 15000 },
      limits: { maxResults: 4, timeoutMs: 4000 },
      scope: {
        default: "deny",
        rules: [{ action: "allow", match: { chatType: "direct" } }],
      },
      paths: [{ name: "docs", path: "~/notes", pattern: "**/*.md" }],
    },
  },
}
```

---

## Dreaming

Dreaming 配置在 `plugins.entries.memory-core.config.dreaming` 下，而不是 `memory.search` 下。

Dreaming 作为一次定时扫描运行，并将内部的轻度/深度/REM 阶段作为实现细节。

有关概念行为和斜杠命令，请参阅 [Dreaming](/zh-CN/concepts/dreaming)。

### 用户设置

| 键                                     | 类型      | 默认值        | 说明                                                                                                                              |
| -------------------------------------- | --------- | ------------- | -------------------------------------------------------------------------------------------------------------------------------- |
| `enabled`                              | `boolean` | `false`       | 完全启用或禁用 Dreaming                                                                                              |
| `frequency`                            | `string`  | `0 3 * * *`   | 完整 Dreaming 扫描的可选 cron 周期                                                                                |
| `model`                                | `string`  | 默认模型 | 可选的 Dream Diary 子智能体模型覆盖                                                                                     |
| `phases.deep.maxPromotedSnippetTokens` | `number`  | `160`         | 从提升到 `MEMORY.md` 的每个短期回忆片段中保留的最大估算 token 数；溯源元数据仍然可见 |

### 示例

```json5
{
  plugins: {
    entries: {
      "memory-core": {
        subagent: {
          allowModelOverride: true,
          allowedModels: ["anthropic/claude-sonnet-4-6"],
        },
        config: {
          dreaming: {
            enabled: true,
            frequency: "0 3 * * *",
            model: "anthropic/claude-sonnet-4-6",
          },
        },
      },
    },
  },
}
```

<Note>
- Dreaming 将机器状态写入 `memory/.dreams/`。
- Dreaming 将人类可读的叙述输出写入 `DREAMS.md`（或现有的 `dreams.md`）。
- `dreaming.model` 使用现有的插件子智能体信任门槛；启用前请设置 `plugins.entries.memory-core.subagent.allowModelOverride: true`。
- 当配置的模型不可用时，Dream Diary 会使用会话默认模型重试一次。信任或允许列表失败会被记录，且不会静默重试。
- 轻度/深度/REM 阶段策略和阈值属于内部行为，不是面向用户的配置。

</Note>

## 相关内容

- [配置参考](/zh-CN/gateway/configuration-reference)
- [记忆概览](/zh-CN/concepts/memory)
- [记忆搜索](/zh-CN/concepts/memory-search)
