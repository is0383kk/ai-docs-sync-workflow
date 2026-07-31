---
read_when:
    - 你想通过 Ollama 使用云端或本地模型运行 OpenClaw
    - 你需要 Ollama 的设置和配置指南
    - 你想使用 Ollama 视觉模型进行图像理解
summary: 使用 Ollama（云端和本地模型）运行 OpenClaw
title: Ollama
x-i18n:
    generated_at: "2026-07-26T06:26:10Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 80ae833d006ce307406fac11fe3457809165035a38b7e0a970777baf126cc9cb
    source_path: providers/ollama.md
    workflow: 16
---

OpenClaw 使用 Ollama 的原生 API（`/api/chat`），而不是兼容 OpenAI 的
`/v1` 端点。支持三种模式：

| 模式          | 使用的资源                                                                     |
| ------------- | -------------------------------------------------------------------------------- |
| 云端 + 本地 | 一个可访问的 Ollama 主机，用于提供本地模型以及（如果已登录）`:cloud` 模型 |
| 仅云端    | 直接使用 `https://ollama.com`，无需本地守护进程                                   |
| 仅本地    | 一个可访问的 Ollama 主机，仅使用本地模型                                       |

如需使用专用 `ollama-cloud` 提供商 ID 进行纯云端设置，请参阅
[Ollama Cloud](/zh-CN/providers/ollama-cloud)。如果你希望将云端路由与本地 `ollama` 提供商分开，
请使用 `ollama-cloud/<model>` 引用。

<Warning>
请勿使用 `/v1` 的 OpenAI 兼容 URL（`http://host:11434/v1`）。它会导致工具调用失效，并且模型可能会将原始工具调用 JSON 作为纯文本输出。请使用原生 URL：`baseUrl: "http://host:11434"`（不含 `/v1`）。
</Warning>

规范配置键为 `baseUrl`。为了兼容 OpenAI SDK 风格的示例，也接受
`baseURL`，但新配置应使用 `baseUrl`。

## 身份验证规则

<AccordionGroup>
  <Accordion title="本地和局域网主机">
    环回地址、专用网络、`.local` 和仅含主机名的 Ollama URL 不需要真实的持有者令牌。OpenClaw 对这些地址使用 `ollama-local` 标记。
  </Accordion>
  <Accordion title="远程和 Ollama Cloud 主机">
    公共远程主机和 `https://ollama.com` 需要真实凭据：`OLLAMA_API_KEY`、身份验证配置文件或提供商的 `apiKey`。对于直接托管使用，首选 `ollama-cloud` 提供商。
  </Accordion>
  <Accordion title="自定义提供商 ID">
    使用 `api: "ollama"` 的自定义提供商遵循相同规则。例如，指向专用局域网主机的 `ollama-remote` 提供商可以使用 `apiKey: "ollama-local"`；子智能体会通过 Ollama 提供商钩子解析该标记，而不会将其视为缺少凭据。`memory.search.provider` 也可以指向自定义提供商 ID，使嵌入使用相应的 Ollama 端点。
  </Accordion>
  <Accordion title="身份验证配置文件">
    `auth-profiles.json` 存储提供商 ID 的凭据；请将端点设置（`baseUrl`、`api`、模型、请求头、超时时间）放在 `models.providers.<id>` 中。`{ "ollama-windows": { "apiKey": "ollama-local" } }` 等旧版扁平文件不是运行时格式；`openclaw doctor --fix` 会将它们重写为规范的 `ollama-windows:default` API 密钥配置文件并创建备份。该旧版文件中的 `baseUrl` 值是无效信息，应移至提供商配置。
  </Accordion>
  <Accordion title="记忆嵌入范围">
    Ollama 记忆嵌入的持有者身份验证仅适用于声明它的主机：

    - 提供商级密钥仅发送到该提供商的主机。
    - `memory.search.remote.apiKey` 和每智能体覆盖项仅发送到各自的远程嵌入主机。
    - 纯 `OLLAMA_API_KEY` 环境变量值会被视为 Ollama Cloud 约定，默认不会发送到本地或自行托管的主机。

  </Accordion>
</AccordionGroup>

## 入门指南

<Tabs>
  <Tab title="新手引导（推荐）">
    <Steps>
      <Step title="运行新手引导">
        ```bash
        openclaw onboard
        ```

        选择 **Ollama**，然后选择一种模式：**云端 + 本地**、**仅云端** 或 **仅本地**。

        在全新的引导式设置中，OpenClaw 首先检查默认或已配置的
        Ollama 主机。仅当 `/api/show` 确认模型支持工具且上下文窗口至少为 16K 时，
        才会自动提供已安装的模型；如果缺少上下文元数据或其值更小，
        则继续使用手动设置流程。共享的 CLI/macOS 设置阶梯仍会通过一次
        真实补全验证所选路由，然后再保存。此自动检查绝不会拉取
        模型；如果没有合适的已安装模型，新手引导会继续进入
        常规 Ollama 选择器。
      </Step>
      <Step title="选择模型">
        `Cloud only` 会提示输入 `OLLAMA_API_KEY`，并建议托管云端默认值。`Cloud + Local` 和 `Local only` 会提示输入 Ollama 基础 URL、发现可用模型，并在缺少所选本地模型时自动拉取。已安装的 `:latest` 标签（例如 `gemma4:latest`）只显示一次，不会与 `gemma4` 重复。`Cloud + Local` 还会检查主机是否已登录以访问云端。
      </Step>
      <Step title="验证">
        ```bash
        openclaw models list --provider ollama
        ```
      </Step>
    </Steps>

    非交互模式：

    ```bash
    openclaw onboard --non-interactive \
      --auth-choice ollama \
      --custom-base-url "http://ollama-host:11434" \
      --custom-model-id "qwen3.5:27b" \
      --accept-risk
    ```

    `--custom-base-url` 和 `--custom-model-id` 是可选的；省略它们将使用本地默认主机和 `gemma4` 建议模型。

  </Tab>

  <Tab title="手动设置">
    <Steps>
      <Step title="安装并启动 Ollama">
        从 [ollama.com/download](https://ollama.com/download) 获取，然后拉取模型：

        ```bash
        ollama pull gemma4
        ```

        如需混合云端访问，请在同一主机上运行 `ollama signin`。
      </Step>
      <Step title="设置凭据">
        ```bash
        export OLLAMA_API_KEY="ollama-local"    # 本地/局域网主机，任意值均可
        export OLLAMA_API_KEY="your-real-key"   # 仅用于 https://ollama.com
        ```

        或在配置中设置：`openclaw config set models.providers.ollama.apiKey "OLLAMA_API_KEY"`。
      </Step>
      <Step title="选择模型">
        ```bash
        openclaw models list
        openclaw models set ollama/gemma4
        ```

        或在配置中设置：

        ```json5
        {
          agents: {
            defaults: {
              model: { primary: "ollama/gemma4" },
            },
          },
        }
        ```
      </Step>
    </Steps>

  </Tab>
</Tabs>

## 通过本地主机使用云端模型

`Cloud + Local` 通过一个可访问的 Ollama 主机同时路由本地模型和
`:cloud` 模型——这是 Ollama 的混合流程；如果你希望同时使用两者，
应在设置期间选择此模式。

OpenClaw 会提示输入基础 URL、发现本地模型并检查
`ollama signin` 状态。登录后，它会建议托管默认模型
（`kimi-k2.5:cloud`、`minimax-m2.7:cloud`、`glm-5.1:cloud`、`glm-5.2:cloud`）。如果
未登录，设置会保持仅本地模式，直到你运行 `ollama signin`。

如需在没有本地守护进程的情况下仅访问云端，请使用 `openclaw onboard --auth-choice ollama-cloud` 并参阅 [Ollama Cloud](/zh-CN/providers/ollama-cloud)——此路径不需要 `ollama signin` 或正在运行的服务器：

```bash
openclaw onboard --auth-choice ollama-cloud
openclaw models set ollama-cloud/kimi-k2.5:cloud
```

在 `openclaw onboard` 期间显示的云端模型列表会从
`https://ollama.com/api/tags` 实时填充，最多包含 500 个条目，因此选择器会反映
当前托管目录。如果设置时无法访问 `ollama.com` 或其未返回任何
模型，OpenClaw 会回退到硬编码的建议列表，以便
新手引导仍能完成。

## 模型发现（隐式提供商）

当设置了 `OLLAMA_API_KEY`（或身份验证配置文件），且既未定义
`models.providers.ollama`，也未定义其他使用 `api: "ollama"` 的自定义提供商时，
OpenClaw 会从 `http://127.0.0.1:11434` 发现模型：

| 行为             | 详情                                                                                                                                                                                                                                                                                        |
| -------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 目录查询        | `/api/tags`                                                                                                                                                                                                                                                                                   |
| 能力检测 | 尽力通过 `/api/show` 读取 `contextWindow`、`num_ctx` Modelfile 参数和能力（视觉/工具/思考）                                                                                                                                                                       |
| 视觉模型        | `/api/show` 中的 `vision` 能力会将模型标记为支持图像（`input: ["text", "image"]`）                                                                                                                                                                                             |
| 推理检测  | 可用时使用 `/api/show` 中的 `thinking` 能力；当 Ollama 省略能力时，回退到名称启发式规则（`r1`、`reason`、`reasoning`、`think`）。无论报告的能力如何，`glm-5.2:cloud` 和 `deepseek-v4-flash\|pro:cloud` 始终视为推理模型。 |
| 令牌限制         | `maxTokens` 默认为 OpenClaw 的 Ollama 最大令牌上限                                                                                                                                                                                                                                       |
| 费用                | 所有费用均为 `0`                                                                                                                                                                                                                                                                             |

```bash
ollama list
openclaw models list
```

设置包含显式 `models` 数组的 `models.providers.ollama`，或者设置使用
`api: "ollama"` 且 `baseUrl` 非环回的自定义提供商，会禁用
自动发现；之后必须手动定义模型（参阅
[配置](#configuration)）。指向托管 `https://ollama.com` 的
`models.providers.ollama` 条目也会跳过发现，因为 Ollama Cloud 模型
由提供商管理。`http://127.0.0.2:11434` 等环回自定义提供商仍视为本地提供商，并保留自动发现。

你可以使用 `ollama/<pulled-model>:latest` 等完整引用，而无需手写
`models.json` 条目；OpenClaw 会实时解析它。对于已登录的
主机，选择未列出的 `ollama/<model>:cloud` 引用时，会通过 `/api/show` 验证该
确切模型，并且仅在 Ollama 确认元数据后才将其添加到运行时目录中——输入错误的模型名称仍会因模型未知而失败。

### 冒烟测试

如需跳过完整智能体工具界面的精简文本探测：

```bash
OLLAMA_API_KEY=ollama-local \
  openclaw infer model run \
    --local \
    --model ollama/llama3.2:latest \
    --prompt "Reply with exactly: pong" \
    --json
```

添加 `--file` 并提供图像，可进行精简的视觉模型探测（接受 PNG/JPEG/WebP；
非图像文件会在调用 Ollama 之前被拒绝——音频请使用
`openclaw infer audio transcribe`）：

```bash
OLLAMA_API_KEY=ollama-local \
  openclaw infer model run \
    --local \
    --model ollama/qwen2.5vl:7b \
    --prompt "Describe this image in one sentence." \
    --file ./photo.jpg \
    --json
```

这两种路径都不会加载聊天工具、记忆或会话上下文。如果它能成功，
而普通智能体回复失败，则问题很可能在于模型的工具/智能体能力，
而不是端点。

使用 `/model ollama/<model>` 选择模型是用户的明确选择：如果已配置的 `baseUrl` 无法访问，下一次回复会因提供商错误而失败，而不会静默回退到另一个已配置的模型。

隔离的定时任务会在启动智能体轮次前增加一项本地安全检查：如果所选模型解析到本地/专用网络/`.local` Ollama 提供商，而 `/api/tags` 无法访问，OpenClaw 会将该次运行记录为 `skipped`，并在错误文本中包含模型。此端点检查按主机缓存 5 分钟，因此针对已停止守护进程重复运行的定时任务不会全部发起注定失败的请求。

实时验证：

```bash
OPENCLAW_LIVE_TEST=1 OPENCLAW_LIVE_OLLAMA=1 OPENCLAW_LIVE_OLLAMA_WEB_SEARCH=0 \
  pnpm test:live -- extensions/ollama/ollama.live.test.ts
```

对于 Ollama Cloud，将同一实时测试指向托管端点（默认跳过嵌入；由于云端密钥可能没有 `/api/embed` 的授权，可使用 `OPENCLAW_LIVE_OLLAMA_EMBEDDINGS=1` 强制启用）：

```bash
export OLLAMA_API_KEY='<your-ollama-cloud-api-key>'
OPENCLAW_LIVE_TEST=1 OPENCLAW_LIVE_OLLAMA=1 \
OPENCLAW_LIVE_OLLAMA_BASE_URL=https://ollama.com \
OPENCLAW_LIVE_OLLAMA_MODEL=glm-5.1:cloud \
OPENCLAW_LIVE_OLLAMA_WEB_SEARCH=1 \
pnpm test:live -- extensions/ollama/ollama.live.test.ts
```

要添加模型，请拉取该模型，系统会自动发现它：

```bash
ollama pull mistral
```

## 节点本地推理

智能体可以将短任务委派给已配对桌面设备或服务器节点上的 Ollama 模型。提示词和响应通过现有的已认证 Gateway 网关/节点连接传输；请求在节点自身的 local loopback Ollama 端点（`http://127.0.0.1:11434`）上运行。

<Steps>
  <Step title="在节点上启动 Ollama">
    ```bash
    ollama pull qwen3:0.6b
    ollama list
    ```
  </Step>
  <Step title="连接节点主机">
    ```bash
    openclaw node run \
      --host <gateway-host> \
      --port 18789 \
      --display-name "Local inference"
    ```

    在 Gateway 网关主机上批准设备及其节点命令，然后验证：

    ```bash
    openclaw devices list
    openclaw devices approve <deviceRequestId>
    openclaw nodes pending
    openclaw nodes approve <nodeRequestId>
    openclaw nodes status --connected
    ```

    首次连接或添加 Ollama 命令的升级可能会触发节点命令审批。如果节点连接后未公布 `ollama.models` 和 `ollama.chat`，请再次检查 `openclaw nodes pending`。

  </Step>
  <Step title="从智能体中使用">
    内置 Ollama 插件提供 `node_inference` 工具。智能体先调用 `action: "discover"`，然后使用该结果中的节点和模型调用 `action: "run"`（当恰好连接了一个具备相应能力的节点时，`run` 可以省略节点）。例如：“发现我的节点上的 Ollama 模型，然后使用已加载且速度最快的模型总结此文本。”
  </Step>
</Steps>

设备发现会读取 `/api/tags`、检查 `/api/show` 能力，并在可用时使用 `/api/ps`，以优先排列已加载的模型。它仅返回 Ollama 报告为支持聊天的本地模型（`completion` 能力）——Ollama Cloud 条目和仅支持嵌入的模型会被排除。每次运行都会禁用模型思考，且输出默认为 512 个令牌（硬上限为 8192），除非工具调用请求不同的 `maxTokens`；某些模型（例如 GPT-OSS）不支持禁用思考，因此仍可能输出推理令牌。

要让 Ollama 在节点上保持运行但不向智能体开放：

```bash
openclaw config set plugins.entries.ollama.config.nodeInference.enabled false
```

重启节点（`openclaw node restart`，或者对于前台会话，停止并重新运行 `openclaw node run`）。节点将停止公布 `ollama.models` 和 `ollama.chat`；Ollama 本身以及 Gateway 网关的 Ollama 提供商不受影响。将值改回 `true` 并重启即可重新启用；重新连接后，发生变化的命令表面可能需要再次批准 `openclaw nodes pending`。

无需启动智能体轮次即可直接验证节点命令：

```bash
openclaw nodes invoke \
  --node "Local inference" \
  --command ollama.models \
  --params '{}' \
  --invoke-timeout 90000 \
  --timeout 100000

openclaw nodes invoke \
  --node "Local inference" \
  --command ollama.chat \
  --params '{"model":"qwen3:0.6b","prompt":"Reply with exactly: pong","maxTokens":32,"timeoutMs":120000}' \
  --invoke-timeout 130000 \
  --timeout 140000
```

`--invoke-timeout` 限制节点运行命令的时长；`--timeout` 限制整个 Gateway 网关调用的时长，并且应设置得更大。

节点本地推理始终使用节点自身的 local loopback 端点——不会复用已配置的远程/云端 `models.providers.ollama.baseUrl`。节点命令默认可用于 macOS、Linux 和 Windows 节点主机，并且仍受常规节点配对/命令策略约束。

## 视觉和图像描述

内置 Ollama 插件会将 Ollama 注册为具备图像能力的媒体理解提供商，因此 OpenClaw 可以通过本地或托管的 Ollama 视觉模型路由明确的图像描述请求和已配置的图像模型默认值。

```bash
ollama pull qwen2.5vl:7b
export OLLAMA_API_KEY="ollama-local"
openclaw infer image describe --file ./photo.jpg --model ollama/qwen2.5vl:7b --json
```

`--model` 必须是完整的 `<provider/model>` 引用；设置后，`infer image
describe` 会先尝试该模型，而不是因为模型已原生支持视觉就跳过描述。如果调用失败，OpenClaw 可以继续执行 `agents.defaults.imageModel.fallbacks`；文件/URL 准备错误会在尝试回退前导致失败。对 OpenClaw 的图像理解流程和已配置的 `imageModel` 使用 `infer image describe`；对带自定义提示词的原始多模态探测使用 `infer model run
--file`。

要将 Ollama 设置为入站媒体的默认图像理解提供商：

```json5
{
  agents: {
    defaults: {
      imageModel: {
        primary: "ollama/qwen2.5vl:7b",
      },
    },
  },
}
```

优先使用完整的 `ollama/<model>` 引用。仅当 `qwen2.5vl:7b` 等裸 `imageModel` 引用以该确切模型列在 `models.providers.ollama.models` 下、带有 `input: ["text", "image"]`，并且没有其他已配置的图像提供商公开相同裸 ID 时，才会将其规范化为 `ollama/qwen2.5vl:7b`；否则请显式使用提供商前缀。

与云端模型相比，较慢的本地视觉模型可能需要更长的图像理解超时；如果 Ollama 尝试分配模型所公布的完整视觉上下文，模型还可能在资源受限的硬件上崩溃。请设置能力超时并限制 `num_ctx`：

```json5
{
  models: {
    providers: {
      ollama: {
        models: [
          {
            id: "qwen2.5vl:7b",
            name: "qwen2.5vl:7b",
            input: ["text", "image"],
            params: { num_ctx: 2048, keep_alive: "1m" },
          },
        ],
      },
    },
  },
  tools: {
    media: {
      image: {
        timeoutSeconds: 180,
        models: [{ provider: "ollama", model: "qwen2.5vl:7b", timeoutSeconds: 300 }],
      },
    },
  },
}
```

此超时适用于入站图像理解和显式 `image` 工具。对于常规模型调用，`models.providers.ollama.timeoutSeconds` 仍控制底层 Ollama HTTP 请求保护时限。

实时验证：

```bash
OPENCLAW_LIVE_TEST=1 OPENCLAW_LIVE_OLLAMA_IMAGE=1 \
  pnpm test:live -- src/agents/tools/image-tool.ollama.live.test.ts
```

如果手动定义 `models.providers.ollama.models`，请显式标记视觉模型：

```json5
{
  id: "qwen2.5vl:7b",
  name: "qwen2.5vl:7b",
  input: ["text", "image"],
  contextWindow: 128000,
  maxTokens: 8192,
}
```

OpenClaw 会拒绝针对未标记为具备图像能力的模型发出的图像描述请求。使用隐式设备发现时，此信息来自 `/api/show` 的视觉能力。

## 配置

<Tabs>
  <Tab title="基础（隐式设备发现）">
    ```bash
    export OLLAMA_API_KEY="ollama-local"
    ```

    <Tip>
    如果设置了 `OLLAMA_API_KEY`，则可以在提供商条目中省略 `apiKey`；OpenClaw 会为可用性检查自动填充它。
    </Tip>

  </Tab>

  <Tab title="显式（手动模型）">
    对于托管云端设置、非默认主机/端口、强制上下文窗口或完全手动的模型列表，请使用显式配置：

    ```json5
    {
      models: {
        providers: {
          ollama: {
            baseUrl: "https://ollama.com",
            apiKey: "OLLAMA_API_KEY",
            api: "ollama",
            models: [
              {
                id: "kimi-k2.5:cloud",
                name: "kimi-k2.5:cloud",
                reasoning: false,
                input: ["text", "image"],
                cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 },
                contextWindow: 128000,
                maxTokens: 8192
              }
            ]
          }
        }
      }
    }
    ```

  </Tab>

  <Tab title="自定义基础 URL">
    显式配置会禁用自动发现，因此必须列出模型：

    ```json5
    {
      models: {
        providers: {
          ollama: {
            apiKey: "ollama-local",
            baseUrl: "http://ollama-host:11434", // 无 /v1——原生 Ollama API URL
            api: "ollama", // 显式指定：保证原生工具调用行为
            timeoutSeconds: 300, // 可选：为冷启动的本地模型提供更长的连接/流式传输时间预算
            models: [
              {
                id: "qwen3:32b",
                name: "qwen3:32b",
                params: {
                  keep_alive: "15m", // 可选：在轮次之间保持模型已加载
                },
              },
            ],
          },
        },
      },
    }
    ```

    <Warning>
    不要添加 `/v1`。该路径会选择 OpenAI 兼容模式，而该模式下的工具调用并不可靠。
    </Warning>

  </Tab>
</Tabs>

## 常用方案

请使用 `ollama list` 或 `openclaw models list --provider ollama` 中的确切名称替换模型 ID。

<AccordionGroup>
  <Accordion title="使用自动发现的本地模型">
    Ollama 与 Gateway 网关位于同一台机器上，并自动被发现：

    ```bash
    ollama serve
    ollama pull gemma4
    export OLLAMA_API_KEY="ollama-local"
    openclaw models list --provider ollama
    openclaw models set ollama/gemma4
    ```

    除非需要手动指定模型，否则不要添加 `models.providers.ollama` 块。

  </Accordion>

  <Accordion title="使用手动模型的局域网 Ollama 主机">
    ```json5
    {
      models: {
        providers: {
          ollama: {
            baseUrl: "http://gpu-box.local:11434",
            apiKey: "ollama-local",
            api: "ollama",
            timeoutSeconds: 300,
            contextWindow: 32768,
            maxTokens: 8192,
            models: [
              {
                id: "qwen3.5:9b",
                name: "qwen3.5:9b",
                reasoning: true,
                input: ["text"],
                params: {
                  num_ctx: 32768,
                  thinking: false,
                  keep_alive: "15m",
                },
              },
            ],
          },
        },
      },
      agents: {
        defaults: {
          model: { primary: "ollama/qwen3.5:9b" },
        },
      },
    }
    ```

    `contextWindow` 是 OpenClaw 的上下文预算；`params.num_ctx` 会发送给 Ollama。当硬件无法运行模型所公布的完整上下文时，请使两者保持一致。

  </Accordion>

  <Accordion title="仅使用 Ollama Cloud">
    无需本地守护进程，直接使用托管模型：

    ```bash
    export OLLAMA_API_KEY="your-ollama-api-key"
    ```

    ```json5
    {
      models: {
        providers: {
          ollama: {
            baseUrl: "https://ollama.com",
            apiKey: "OLLAMA_API_KEY",
            api: "ollama",
            models: [
              {
                id: "kimi-k2.5:cloud",
                name: "kimi-k2.5:cloud",
                reasoning: false,
                input: ["text", "image"],
                contextWindow: 128000,
                maxTokens: 8192,
              },
            ],
          },
        },
      },
      agents: {
        defaults: {
          model: { primary: "ollama/kimi-k2.5:cloud" },
        },
      },
    }
    ```

    如需使用专用的 `ollama-cloud` 提供商 ID，而不是此结构，请参阅
    [Ollama Cloud](/zh-CN/providers/ollama-cloud)。

  </Accordion>

  <Accordion title="通过已登录的守护进程同时使用云端和本地模型">
    ```bash
    ollama signin
    ollama pull gemma4
    ```

    ```json5
    {
      models: {
        providers: {
          ollama: {
            baseUrl: "http://127.0.0.1:11434",
            apiKey: "ollama-local",
            api: "ollama",
            timeoutSeconds: 300,
            models: [
              { id: "gemma4", name: "gemma4", input: ["text"] },
              { id: "kimi-k2.5:cloud", name: "kimi-k2.5:cloud", input: ["text", "image"] },
            ],
          },
        },
      },
      agents: {
        defaults: {
          model: {
            primary: "ollama/gemma4",
            fallbacks: ["ollama/kimi-k2.5:cloud"],
          },
        },
      },
    }
    ```

  </Accordion>

  <Accordion title="多个 Ollama 主机">
    运行多个 Ollama 服务器时可使用自定义提供商 ID；每个提供商都有自己的
    主机、模型、身份验证和超时设置。

    ```json5
    {
      models: {
        providers: {
          "ollama-fast": {
            baseUrl: "http://mini.local:11434",
            apiKey: "ollama-local",
            api: "ollama",
            contextWindow: 32768,
            models: [{ id: "gemma4", name: "gemma4", input: ["text"] }],
          },
          "ollama-large": {
            baseUrl: "http://gpu-box.local:11434",
            apiKey: "ollama-local",
            api: "ollama",
            timeoutSeconds: 420,
            contextWindow: 131072,
            maxTokens: 16384,
            models: [{ id: "qwen3.5:27b", name: "qwen3.5:27b", input: ["text"] }],
          },
        },
      },
      agents: {
        defaults: {
          model: {
            primary: "ollama-fast/gemma4",
            fallbacks: ["ollama-large/qwen3.5:27b"],
          },
        },
      },
    }
    ```

    OpenClaw 在调用 Ollama 前会移除当前提供商前缀（找不到时回退到裸
    `ollama/` 前缀），因此 `ollama-large/qwen3.5:27b`
    到达 Ollama 时会变为 `qwen3.5:27b`。

  </Accordion>

  <Accordion title="精简本地模型配置">
    某些本地模型可以处理简单提示词，但难以应对完整的智能体
    工具界面。请先限制工具和上下文，再调整全局运行时
    设置：

    ```json5
    {
      agents: {
        list: [
          {
            id: "local",
            experimental: {
              localModelLean: true,
            },
            model: { primary: "ollama/gemma4" },
          },
        ],
      },
      models: {
        providers: {
          ollama: {
            baseUrl: "http://127.0.0.1:11434",
            apiKey: "ollama-local",
            api: "ollama",
            contextWindow: 32768,
            models: [
              {
                id: "gemma4",
                name: "gemma4",
                input: ["text"],
                params: { num_ctx: 32768 },
                compat: { supportsTools: false },
              },
            ],
          },
        },
      },
    }
    ```

    仅当模型或服务器在处理工具架构时持续
    失败，才使用 `compat.supportsTools: false`——它以牺牲智能体能力换取稳定性。
    除非明确需要，否则 `localModelLean` 会从智能体的直接工具界面中移除重量级的浏览器、定时任务、消息、媒体生成、
    语音和 PDF 工具，
    并将较大的工具目录置于工具搜索之后。它不会改变 Ollama 的
    运行时上下文或思考模式。对于会循环或
    将预算消耗在隐藏推理上的小型 Qwen 风格思考模型，请将它与 `params.num_ctx` 和
    `params.thinking: false` 搭配使用。

  </Accordion>
</AccordionGroup>

### 模型选择

```json5
{
  agents: {
    defaults: {
      model: {
        primary: "ollama/gpt-oss:20b",
        fallbacks: ["ollama/llama3.3", "ollama/qwen2.5-coder:32b"],
      },
    },
  },
}
```

自定义提供商 ID 的工作方式相同：对于使用当前提供商
前缀的引用（例如 `ollama-spark/qwen3:32b`），OpenClaw 会在
调用 Ollama 前移除该前缀，并发送 `qwen3:32b`。

对于速度较慢的本地模型，请优先调整提供商范围的设置，而不是提高整个
智能体运行时的超时时间：

```json5
{
  models: {
    providers: {
      ollama: {
        timeoutSeconds: 300,
        models: [
          {
            id: "gemma4:26b",
            name: "gemma4:26b",
            params: { keep_alive: "15m" },
          },
        ],
      },
    },
  },
}
```

`timeoutSeconds` 涵盖模型 HTTP 请求的全过程：连接建立、标头、
正文流式传输以及受保护提取操作的总中止时间。原生 `/api/chat` 请求会将
`params.keep_alive` 作为顶层 `keep_alive` 转发；当首次轮次的加载时间成为瓶颈时，请按
模型设置它。

### 快速验证

```bash
# 此计算机能够访问 Ollama 守护进程
curl http://127.0.0.1:11434/api/tags

# OpenClaw 目录和所选模型
openclaw models list --provider ollama
openclaw models status

# 直接模型冒烟测试
openclaw infer model run \
  --model ollama/gemma4 \
  --prompt "仅回复：ok"
```

对于远程主机，请将 `127.0.0.1` 替换为 `baseUrl` 主机。如果 `curl`
可以工作，但 OpenClaw 无法工作，请检查 Gateway 网关是否运行在其他
计算机、容器或服务账户下。

## Ollama Web 搜索

OpenClaw 将 **Ollama Web 搜索**内置为 `web_search` 提供商。

| 属性        | 详情                                                                                                                                                       |
| ----------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 主机        | 设置时使用 `models.providers.ollama.baseUrl`，否则使用 `http://127.0.0.1:11434`；`https://ollama.com` 直接使用托管 API                          |
| 身份验证    | 已登录的本地主机无需密钥；直接执行 `https://ollama.com` 搜索或使用受身份验证保护的主机时，使用 `OLLAMA_API_KEY` 或已配置的提供商身份验证           |
| 要求        | 本地/自行托管的主机必须正在运行，并已使用 `ollama signin` 登录；直接托管搜索需要 `baseUrl: "https://ollama.com"` 和真实的 API 密钥 |

在 `openclaw onboard` 或 `openclaw configure --section web` 期间选择它，或设置：

```json5
{
  tools: {
    web: {
      search: {
        provider: "ollama",
      },
    },
  },
}
```

要通过 Ollama Cloud 直接执行托管搜索：

```json5
{
  models: {
    providers: {
      ollama: {
        baseUrl: "https://ollama.com",
        apiKey: "OLLAMA_API_KEY",
        api: "ollama",
        models: [{ id: "kimi-k2.5:cloud", name: "kimi-k2.5:cloud", input: ["text"] }],
      },
    },
  },
  tools: {
    web: {
      search: { provider: "ollama" },
    },
  },
}
```

对于自行托管的主机，OpenClaw 会先尝试本地 `/api/experimental/web_search`
代理，然后回退到同一主机上的托管 `/api/web_search` 路径；
已登录的本地守护进程通常会通过本地代理响应。直接
`https://ollama.com` 调用始终使用托管的 `/api/web_search` 端点。

<Note>
有关完整设置和行为，请参阅 [Ollama Web 搜索](/zh-CN/tools/ollama-search)。
</Note>

## 高级配置

<AccordionGroup>
  <Accordion title="旧版 OpenAI 兼容模式">
    <Warning>
    **此模式下的工具调用并不可靠。** 仅当代理需要 OpenAI 格式，并且你不依赖原生工具调用时才使用此模式。
    </Warning>

    对于位于 `/v1/chat/completions` 后面的代理，请显式设置
    `api: "openai-completions"`：

    ```json5
    {
      models: {
        providers: {
          ollama: {
            baseUrl: "http://ollama-host:11434/v1",
            api: "openai-completions",
            injectNumCtxForOpenAICompat: true, // 默认值：true
            apiKey: "ollama-local",
            models: [...]
          }
        }
      }
    }
    ```

    此模式可能不支持同时进行流式传输和工具调用；你
    可能需要在模型上设置 `params: { streaming: false }`。

    OpenClaw 默认会在此模式下注入 `options.num_ctx`，以免 Ollama
    静默回退到 4096 token 的上下文。如果你的代理拒绝
    未知的 `options` 字段，请将其禁用：

    ```json5
    {
      models: {
        providers: {
          ollama: {
            baseUrl: "http://ollama-host:11434/v1",
            api: "openai-completions",
            injectNumCtxForOpenAICompat: false,
            apiKey: "ollama-local",
            models: [...]
          }
        }
      }
    }
    ```

  </Accordion>

  <Accordion title="上下文窗口">
    对于自动发现的模型，OpenClaw 使用 `/api/show`
    报告的上下文窗口，包括来自自定义
    Modelfile 的较大 `PARAMETER num_ctx` 值；否则会回退到 OpenClaw 的默认 Ollama 上下文
    窗口。

    提供商级别的 `contextWindow`、`contextTokens` 和 `maxTokens` 会为
    该提供商下的每个模型设置默认值，并可按
    模型覆盖。`contextWindow` 是 OpenClaw 自身的提示词/压缩预算。除非你显式设置
    `params.num_ctx`，否则原生
    `/api/chat` 请求不会设置 `options.num_ctx`，因此 Ollama 会应用自己的模型、
    `OLLAMA_CONTEXT_LENGTH` 或基于 VRAM 的默认值；无效、零值、负值
    或非有限的 `params.num_ctx` 值会被忽略。如果旧配置仅使用
    `contextWindow`/`maxTokens` 强制设置原生请求上下文，请运行
    `openclaw doctor --fix` 将它们复制到 `params.num_ctx`。OpenAI
    兼容适配器仍会默认根据已配置的 `params.num_ctx` 或
    `contextWindow` 注入 `options.num_ctx`；如果上游拒绝
    `options`，请使用 `injectNumCtxForOpenAICompat: false` 禁用此行为。

    原生模型条目还接受 `params` 下的常用 Ollama 运行时选项，
    并将其作为原生 `/api/chat` `options` 转发：`num_keep`、`seed`、
    `num_predict`、`top_k`、`top_p`、`min_p`、`typical_p`、`repeat_last_n`、
    `temperature`、`repeat_penalty`、`presence_penalty`、`frequency_penalty`、
    `stop`、`num_batch`、`num_gpu`、`main_gpu`、`use_mmap` 和 `num_thread`。
    少数键（`format`、`keep_alive`、`truncate`、`shift`）会作为
    顶层请求字段转发，而不是嵌套在 `options` 中。OpenClaw 仅
    转发这些 Ollama 请求键，因此 `streaming` 等仅限运行时的参数
    永远不会发送给 Ollama。使用 `params.think`（或
    `params.thinking`）设置顶层 `think`；`false` 会为 Qwen 风格的思考模型禁用 API 级
    思考。

    ```json5
    {
      models: {
        providers: {
          ollama: {
            contextWindow: 32768,
            models: [
              {
                id: "llama3.3",
                contextWindow: 131072,
                maxTokens: 65536,
                params: {
                  num_ctx: 32768,
                  temperature: 0.7,
                  top_p: 0.9,
                  thinking: false,
                },
              }
            ]
          }
        }
      }
    }
    ```

    每个模型的 `agents.defaults.models["ollama/<model>"].params.num_ctx` 也
    有效；如果两者都已设置，则显式的提供商模型条目优先。

  </Accordion>

  <Accordion title="思考控制">
    OpenClaw 会按 Ollama 预期的方式传递思考设置：使用顶层 `think`，而不是
    `options.think`。自动发现的模型中，如果其 `/api/show` 报告具有
    `thinking` 能力，则会提供 `/think low`、`/think medium`、`/think high`
    和 `/think max`；非思考模型仅提供 `/think off`。

    ```bash
    openclaw agent --model ollama/gemma4 --thinking off
    openclaw agent --model ollama/gemma4 --thinking low
    ```

    或设置模型默认值：

    ```json5
    {
      agents: {
        defaults: {
          models: {
            "ollama/gemma4": {
              thinking: "low",
            },
          },
        },
      },
    }
    ```

    每个模型的 `params.think`/`params.thinking` 可以为特定模型禁用或强制启用 API
    思考。仅当当前运行使用隐式的 `off` 默认值时，OpenClaw 才会保留该显式配置；
    非关闭状态的运行时命令（如 `/think medium`）仍会覆盖它。对于明确标记为
    `reasoning: false` 的模型，绝不会发送真值思考请求；无论如何都会发送
    `think: false` 请求。

  </Accordion>

  <Accordion title="推理模型">
    名为 `deepseek-r1`、`reasoning`、`reason` 或 `think` 的模型默认被视为
    具备推理能力，无需额外配置：

    ```bash
    ollama pull deepseek-r1:32b
    ```

  </Accordion>

  <Accordion title="模型费用">
    Ollama 在本地运行且免费，因此自动发现和手动定义的模型费用均为 `0`。
  </Accordion>

  <Accordion title="记忆嵌入">
    内置的 Ollama 插件为
    [记忆搜索](/zh-CN/concepts/memory)注册了记忆嵌入提供商。它使用已配置的 Ollama 基础 URL
    和 API key，调用 `/api/embed`，并在可能的情况下将多个记忆块合并到
    一个 `input` 请求中。

    当 `proxy.enabled=true` 时，向根据已配置的 `baseUrl` 得出的精确主机本地
    loopback 源发送的嵌入请求，会使用 OpenClaw 的受保护直连路径，而不是托管转发代理。
    配置的主机名本身必须是 `localhost` 或 loopback IP 字面量——仅通过 DNS
    解析到 loopback 的名称仍会使用托管代理路径。局域网、tailnet、专用网络和公共
    Ollama 主机始终使用托管代理路径，并且重定向到其他主机或端口时不会继承信任。
    `proxy.loopbackMode: "proxy"` 仍会通过代理路由 loopback 流量；`proxy.loopbackMode: "block"`
    会在连接前拒绝该流量——参见[托管代理](/zh-CN/security/network-proxy#gateway-loopback-mode)。

    | 属性 | 值 |
    | --- | --- |
    | 默认模型 | `nomic-embed-text` |
    | 自动拉取 | 是，如果本地不存在 |
    | 默认内联并发数 | 1（其他提供商的默认值更高；如果主机能够承受，可使用 `nonBatchConcurrency` 提高） |

    对需要或建议使用检索前缀的模型，查询时嵌入会使用这些前缀：
    `nomic-embed-text`、`qwen3-embedding` 和
    `mxbai-embed-large`。文档批次保持原始格式，因此现有索引
    无需格式迁移。

    ```json5
    {
      memory: {
        search: {
          provider: "ollama",
          remote: {
            // Ollama 的默认值。如果重新索引速度过慢，可在更大的主机上提高此值。
            nonBatchConcurrency: 1,
          },
        },
      },
    }
    ```

    对于远程嵌入主机，请将身份验证限定在该主机范围内：

    ```json5
    {
      memory: {
        search: {
          provider: "ollama",
          model: "nomic-embed-text",
          remote: {
            baseUrl: "http://gpu-box.local:11434",
            apiKey: "ollama-local",
            nonBatchConcurrency: 2,
          },
        },
      },
    }
    ```

  </Accordion>

  <Accordion title="流式传输配置">
    Ollama 默认使用**原生 API**（`/api/chat`），它同时支持
    流式传输和工具调用，无需特殊配置。

    对于原生请求，思考控制会直接传递：除非配置了显式的
    `params.think`/`params.thinking`，否则 `/think off`
    和 `openclaw agent --thinking off` 会发送顶层 `think: false`；`/think
    low|medium|high`
    会发送匹配的强度字符串；`/think max` 映射到 Ollama 的最高强度
    `think: "high"`。

    <Tip>
    如果要改用 OpenAI 兼容端点，请参阅上文“旧版 OpenAI 兼容模式”——在该模式下，流式传输和工具调用可能无法同时工作。
    </Tip>

  </Accordion>
</AccordionGroup>

## 故障排查

<AccordionGroup>
  <Accordion title="WSL2 崩溃循环（反复重启）">
    在使用 NVIDIA/CUDA 的 WSL2 上，官方 Ollama Linux 安装程序会创建一个
    带有 `Restart=always` 的 `ollama.service` systemd 单元。如果该服务
    自动启动并在 WSL2 启动期间加载由 GPU 支持的模型，Ollama 可能会在加载时锁定
    主机内存；Hyper-V 内存回收并不总能回收这些页面，因此 Windows 可能会终止
    WSL2 虚拟机，systemd 随后重启 Ollama，从而不断循环。

    迹象包括：WSL2 反复重启或终止、WSL2 启动后 `app.slice` 或
    `ollama.service` 的 CPU 占用率很高，以及 SIGTERM 来自 systemd 而不是
    Linux OOM killer。

    当 OpenClaw 检测到 WSL2、已使用 `Restart=always` 启用
    `ollama.service` 且存在可见的 CUDA 标记时，会记录启动警告。

    缓解方法：

    ```bash
    sudo systemctl disable ollama
    ```

    在 Windows 端，将以下内容添加到 `%USERPROFILE%\.wslconfig`，然后运行
    `wsl --shutdown`：

    ```ini
    [experimental]
    autoMemoryReclaim=disabled
    ```

    或缩短保活时间，并仅在需要时手动启动 Ollama：

    ```bash
    export OLLAMA_KEEP_ALIVE=5m
    ollama serve
    ```

    参见 [ollama/ollama#11317](https://github.com/ollama/ollama/issues/11317)。

  </Accordion>

  <Accordion title="未检测到 Ollama">
    确认 Ollama 正在运行，已设置 `OLLAMA_API_KEY`（或身份验证配置文件），
    并且未显式定义 `models.providers.ollama`：

    ```bash
    ollama serve
    curl http://localhost:11434/api/tags
    ```

  </Accordion>

  <Accordion title="没有可用模型">
    在本地拉取模型，或在
    `models.providers.ollama` 中显式定义模型：

    ```bash
    ollama list  # 查看已安装的模型
    ollama pull gemma4
    ollama pull gpt-oss:20b
    ollama pull llama3.3     # 或其他模型
    ```

  </Accordion>

  <Accordion title="连接被拒绝">
    ```bash
    # 检查 Ollama 是否正在运行
    ps aux | grep ollama

    # 或重启 Ollama
    ollama serve
    ```

  </Accordion>

  <Accordion title="远程主机可通过 curl 访问，但 OpenClaw 无法访问">
    在运行 Gateway 网关的同一台机器和运行时中验证：

    ```bash
    openclaw gateway status --deep
    curl http://ollama-host:11434/api/tags
    ```

    常见原因：

    - `baseUrl` 指向 `localhost`，但 Gateway 网关在 Docker 中或其他主机上运行。
    - URL 使用 `/v1`，因此选择了 OpenAI 兼容行为，而不是原生 Ollama。
    - 远程主机需要更改防火墙或局域网绑定设置。
    - 模型位于你的笔记本电脑守护进程上，而不在远程守护进程上。

  </Accordion>

  <Accordion title="模型将工具 JSON 作为文本输出">
    通常是因为提供商处于 OpenAI 兼容模式，或模型无法
    处理工具 schema。应优先使用原生模式：

    ```json5
    {
      models: {
        providers: {
          ollama: {
            baseUrl: "http://ollama-host:11434",
            api: "ollama",
          },
        },
      },
    }
    ```

    如果小型本地模型仍无法处理工具 schema，请在该模型条目上设置
    `compat.supportsTools: false`，然后重新测试。

  </Accordion>

  <Accordion title="Kimi 或 GLM 返回乱码符号">
    如果托管的 Kimi/GLM 响应是长串非语言符号，则会被视为
    提供商调用失败，而不是成功回复，因此系统会执行正常的
    重试、回退或错误处理，而不会将损坏的文本持久化到会话中。

    如果问题再次出现，请记录模型名称、当前会话文件，以及
    此次运行使用的是 `Cloud + Local` 还是 `Cloud only`，然后尝试新
    会话和回退模型：

    ```bash
    openclaw infer model run --model ollama/kimi-k2.5:cloud --prompt "Reply with exactly: ok" --json
    openclaw models set ollama/gemma4
    ```

  </Accordion>

  <Accordion title="冷启动的本地模型超时">
    大型本地模型首次加载可能需要很长时间。将超时范围限定到
    Ollama 提供商，并可选择让模型在轮次之间保持加载状态：

    ```json5
    {
      models: {
        providers: {
          ollama: {
            timeoutSeconds: 300,
            models: [
              {
                id: "gemma4:26b",
                name: "gemma4:26b",
                params: { keep_alive: "15m" },
              },
            ],
          },
        },
      },
    }
    ```

    如果主机本身接受连接的速度较慢，`timeoutSeconds` 还会
    延长此提供商的受保护连接超时时间。

  </Accordion>

  <Accordion title="大上下文模型速度过慢或内存不足">
    许多模型声明的上下文大小超出了硬件能够舒适运行的范围。
    除非设置了 `params.num_ctx`，否则原生 Ollama 会使用其自身的运行时默认值。
    同时限制 OpenClaw 的预算和 Ollama 的请求上下文，可获得可预测的首 token 延迟：

    ```json5
    {
      models: {
        providers: {
          ollama: {
            contextWindow: 32768,
            maxTokens: 8192,
            models: [
              {
                id: "qwen3.5:9b",
                name: "qwen3.5:9b",
                params: { num_ctx: 32768, thinking: false },
              },
            ],
          },
        },
      },
    }
    ```

    如果 OpenClaw 发送的提示词过多，请降低 `contextWindow`。
    如果 Ollama 的运行时上下文对该机器而言过大，请降低
    `params.num_ctx`。如果生成运行时间过长，请降低 `maxTokens`。

  </Accordion>
</AccordionGroup>

<Note>
更多帮助：[故障排查](/zh-CN/help/troubleshooting)和[常见问题](/zh-CN/help/faq)。
</Note>

## 相关内容

<CardGroup cols={2}>
  <Card title="Ollama Cloud" href="/zh-CN/providers/ollama-cloud" icon="cloud">
    使用专用 `ollama-cloud` 提供商的纯云端设置。
  </Card>
  <Card title="模型提供商" href="/zh-CN/concepts/model-providers" icon="layers">
    所有提供商、模型引用和故障转移行为的概览。
  </Card>
  <Card title="模型选择" href="/zh-CN/concepts/models" icon="brain">
    如何选择和配置模型。
  </Card>
  <Card title="Ollama Web 搜索" href="/zh-CN/tools/ollama-search" icon="magnifying-glass">
    由 Ollama 提供支持的 Web 搜索的完整设置和行为详情。
  </Card>
  <Card title="配置" href="/zh-CN/gateway/configuration" icon="gear">
    完整的配置参考。
  </Card>
</CardGroup>
