---
read_when:
    - 你希望通过本地 Inferrs 服务器运行 OpenClaw
    - 你正在通过 Inferrs 提供 Gemma 或其他模型服务
    - 你需要 inferrs 的确切 OpenClaw 兼容性标志位
summary: 通过 inferrs（兼容 OpenAI 的本地服务器）运行 OpenClaw
title: Inferrs
x-i18n:
    generated_at: "2026-07-26T06:25:35Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 8b9b6fe337a2ec6536332dd62840052fd802fad0a5f3d885ce137523266ff3c9
    source_path: providers/inferrs.md
    workflow: 16
---

[inferrs](https://github.com/ericcurtin/inferrs) 通过兼容 OpenAI 的 `/v1` API 提供本地模型。OpenClaw 通过通用 `openai-completions` 适配器与其通信。

| 属性               | 值                                                                   |
| ------------------ | -------------------------------------------------------------------- |
| 提供商 ID          | `inferrs`（自定义；在 `models.providers.inferrs` 下配置）           |
| 插件               | 无 — 不是 OpenClaw 内置提供商插件                                    |
| 身份验证环境变量   | 无需提供；如果你的 inferrs 服务器未启用身份验证，任意值均可          |
| API                | 兼容 OpenAI（`openai-completions`）                                    |
| 建议的基础 URL     | `http://127.0.0.1:8080/v1`（或你的 inferrs 服务器监听的任意位置）            |

<Note>
  `inferrs` 是自托管的自定义 OpenAI 兼容后端，而不是专用的 OpenClaw 提供商插件：应在 `models.providers.inferrs` 下配置，而不是选择新手引导中的身份验证选项。有关支持自动发现的内置插件，请参阅 [SGLang](/zh-CN/providers/sglang) 或 [vLLM](/zh-CN/providers/vllm)。
</Note>

## 入门指南

<Steps>
  <Step title="使用模型启动 inferrs">
    ```bash
    inferrs serve google/gemma-4-E2B-it \
      --host 127.0.0.1 \
      --port 8080 \
      --device metal
    ```
  </Step>
  <Step title="验证服务器是否可访问">
    ```bash
    curl http://127.0.0.1:8080/health
    curl http://127.0.0.1:8080/v1/models
    ```
  </Step>
  <Step title="添加 OpenClaw 提供商条目">
    添加明确的提供商条目，并将默认模型指向该条目。请参阅下方的配置示例。
  </Step>
</Steps>

## 完整配置示例

本地 `inferrs` 服务器上的 Gemma 4：

```json5
{
  agents: {
    defaults: {
      model: { primary: "inferrs/google/gemma-4-E2B-it" },
      models: {
        "inferrs/google/gemma-4-E2B-it": {
          alias: "Gemma 4 (inferrs)",
        },
      },
    },
  },
  models: {
    mode: "merge",
    providers: {
      inferrs: {
        baseUrl: "http://127.0.0.1:8080/v1",
        apiKey: "inferrs-local",
        api: "openai-completions",
        models: [
          {
            id: "google/gemma-4-E2B-it",
            name: "Gemma 4 E2B (inferrs)",
            reasoning: false,
            input: ["text"],
            cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 },
            contextWindow: 131072,
            maxTokens: 4096,
            compat: {
              requiresStringContent: true,
            },
          },
        ],
      },
    },
  },
}
```

## 按需启动

仅当选择了 `inferrs/...` 模型时，OpenClaw 才能自行启动 `inferrs`。将 `localService` 添加到同一提供商条目：

```json5
{
  models: {
    providers: {
      inferrs: {
        baseUrl: "http://127.0.0.1:8080/v1",
        apiKey: "inferrs-local",
        api: "openai-completions",
        timeoutSeconds: 300,
        localService: {
          command: "/opt/homebrew/bin/inferrs",
          args: [
            "serve",
            "google/gemma-4-E2B-it",
            "--host",
            "127.0.0.1",
            "--port",
            "8080",
            "--device",
            "metal",
          ],
          healthUrl: "http://127.0.0.1:8080/v1/models",
          readyTimeoutMs: 180000,
          idleStopMs: 0,
        },
        models: [
          {
            id: "google/gemma-4-E2B-it",
            name: "Gemma 4 E2B (inferrs)",
            reasoning: false,
            input: ["text"],
            cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 },
            contextWindow: 131072,
            maxTokens: 4096,
            compat: {
              requiresStringContent: true,
            },
          },
        ],
      },
    },
  },
}
```

`command` 必须是绝对路径。在 Gateway 网关主机上运行 `which inferrs` 并使用该路径。完整字段参考：[本地模型服务](/zh-CN/gateway/local-model-services)。

## 高级配置

<AccordionGroup>
  <Accordion title="requiresStringContent 为何重要">
    某些 `inferrs` Chat Completions 路由仅接受字符串 `messages[].content`，不接受结构化的内容部分数组。

    <Warning>
    如果 OpenClaw 运行失败并显示：

    ```text
    messages[1].content: 类型无效：应为字符串，却收到序列
    ```

    请在模型条目中设置 `compat.requiresStringContent: true`。随后，OpenClaw 会在发送请求前将纯文本内容部分展平为普通字符串。
    </Warning>

  </Accordion>

  <Accordion title="Gemma 与工具架构注意事项">
    某些 `inferrs` + Gemma 组合能够接受小型的直接 `/v1/chat/completions` 请求，但在完整的 OpenClaw Agent 运行时轮次中失败。请先尝试禁用工具架构表面：

    ```json5
    compat: {
      requiresStringContent: true,
      supportsTools: false
    }
    ```

    这可以减轻更严格的本地后端所承受的提示词压力。如果微型直接请求仍然有效，但正常的 OpenClaw Agent 轮次仍持续在 `inferrs` 内崩溃，应将其视为上游模型或服务器限制，而不是 OpenClaw 传输问题。

  </Accordion>

  <Accordion title="手动冒烟测试">
    配置后测试两个层级：

    ```bash
    curl http://127.0.0.1:8080/v1/chat/completions \
      -H 'content-type: application/json' \
      -d '{"model":"google/gemma-4-E2B-it","messages":[{"role":"user","content":"2 + 2 等于多少？"}],"stream":false}'
    ```

    ```bash
    openclaw infer model run \
      --model inferrs/google/gemma-4-E2B-it \
      --prompt "2 + 2 等于多少？请用一个简短句子回答。" \
      --json
    ```

    如果第一个命令有效但第二个失败，请参阅下方的故障排查。

  </Accordion>

  <Accordion title="代理式行为">
    由于 `inferrs` 使用通用 `openai-completions` 适配器（而非 `openai-responses`），因此绝不会应用仅限原生 OpenAI 的请求整形：不会发送 `service_tier`、Responses `store`、提示词缓存提示或 OpenAI 推理兼容性载荷整形。
  </Accordion>
</AccordionGroup>

## 故障排查

<AccordionGroup>
  <Accordion title="curl /v1/models 失败">
    `inferrs` 未运行、无法访问，或未绑定到所配置的主机或端口。请确认服务器已启动并正在该地址上监听。
  </Accordion>

  <Accordion title="messages[].content 应为字符串">
    在模型条目中设置 `compat.requiresStringContent: true`（见上文）。
  </Accordion>

  <Accordion title="直接调用 /v1/chat/completions 成功，但 openclaw infer model run 失败">
    设置 `compat.supportsTools: false` 以禁用工具架构表面（请参阅上方的 Gemma 注意事项）。
  </Accordion>

  <Accordion title="inferrs 在较大的 Agent 轮次中仍会崩溃">
    如果架构错误已消失，但 `inferrs` 在较大的 Agent 轮次中仍会崩溃，请将其视为上游 `inferrs` 或模型限制。请减轻提示词压力，或更换后端或模型。
  </Accordion>
</AccordionGroup>

<Tip>
如需常规帮助，请参阅[故障排查](/zh-CN/help/troubleshooting)和[常见问题](/zh-CN/help/faq)。
</Tip>

## 相关内容

<CardGroup cols={2}>
  <Card title="本地模型" href="/zh-CN/gateway/local-models" icon="server">
    使用本地模型服务器运行 OpenClaw。
  </Card>
  <Card title="本地模型服务" href="/zh-CN/gateway/local-model-services" icon="play">
    为已配置的提供商按需启动本地模型服务器。
  </Card>
  <Card title="Gateway 网关故障排查" href="/zh-CN/gateway/troubleshooting#local-openai-compatible-backend-passes-direct-probes-but-agent-runs-fail" icon="wrench">
    调试能够通过探测但在 Agent 运行时失败的本地 OpenAI 兼容后端。
  </Card>
  <Card title="模型选择" href="/zh-CN/concepts/model-providers" icon="layers">
    所有提供商、模型引用和故障转移行为的概览。
  </Card>
</CardGroup>
