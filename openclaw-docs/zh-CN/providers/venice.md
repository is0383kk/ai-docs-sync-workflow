---
read_when:
    - 你希望在 OpenClaw 中使用注重隐私的推理
    - 你需要 Venice AI 设置指导
summary: 在 OpenClaw 中使用注重隐私的 Venice AI 模型
title: Venice AI
x-i18n:
    generated_at: "2026-07-26T06:59:10Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 13c32b783394eb3092ff94a532b69e34c00624127b0e76e4e2812751d39073a1
    source_path: providers/venice.md
    workflow: 16
---

[Venice AI](https://venice.ai) 提供注重隐私的推理服务：开放模型运行时
不记录日志，并提供对 Claude、GPT、Gemini 和 Grok 的匿名代理访问。
所有端点均兼容 OpenAI（`/v1`）。

## 隐私模式

| 模式           | 行为                                                         | 模型                                                        |
| -------------- | ---------------------------------------------------------------- | ------------------------------------------------------------- |
| **私密**    | 提示词和响应绝不会被存储或记录。仅短暂存在。         | Llama、Qwen、DeepSeek、Kimi、MiniMax、Venice Uncensored 等。 |
| **匿名化** | 通过 Venice 代理，并在转发前移除元数据。 | Claude、GPT、Gemini、Grok                                     |

<Warning>
匿名化模型并非完全私密。Venice 会在转发前移除元数据，但底层提供商（OpenAI、Anthropic、Google、xAI）仍会处理请求。需要完全隐私时，请使用私密模型。
</Warning>

## 入门指南

<Steps>
  <Step title="安装插件">
    ```bash
    openclaw plugins install @openclaw/venice-provider
    ```
  </Step>
  <Step title="获取 API 密钥">
    1. 在 [venice.ai](https://venice.ai) 注册
    2. 前往 **Settings > API Keys > Create new key**
    3. 复制 API 密钥（格式：`vapi_xxxxxxxxxxxx`）
  </Step>
  <Step title="配置 OpenClaw">
    <Tabs>
      <Tab title="交互式（推荐）">
        ```bash
        openclaw onboard --auth-choice venice-api-key
        ```

        系统会提示输入 API 密钥（或复用现有的 `VENICE_API_KEY`），列出可用的 Venice 模型，并设置默认模型。
      </Tab>
      <Tab title="环境变量">
        ```bash
        export VENICE_API_KEY="vapi_xxxxxxxxxxxx"
        ```
      </Tab>
      <Tab title="非交互式">
        ```bash
        openclaw onboard --non-interactive \
          --auth-choice venice-api-key \
          --venice-api-key "vapi_xxxxxxxxxxxx"
        ```
      </Tab>
    </Tabs>

  </Step>
  <Step title="验证设置">
    ```bash
    openclaw agent --model venice/kimi-k2-5 --message "你好，你能正常工作吗？"
    ```
  </Step>
</Steps>

## 模型选择

- **默认模型**：`venice/kimi-k2-5`（私密、推理、视觉）。
- **最强的匿名化选项**：`venice/claude-opus-4-6`。

```bash
openclaw models set venice/kimi-k2-5
openclaw models list --all --provider venice
```

也可以运行 `openclaw configure`，然后选择 **Model/auth provider > Venice AI**。

<Tip>
| 使用场景              | 模型                                        | 原因                                    |
| --------------------- | -------------------------------------------- | -------------------------------------- |
| 常规聊天（默认） | `kimi-k2-5`                                  | 强大的私密推理和视觉能力   |
| 最佳综合质量   | `claude-opus-4-6`                            | Venice 最强的匿名化选项     |
| 隐私保护 + 编程       | `qwen3-coder-480b-a35b-instruct-turbo`       | 具有大上下文窗口的私密编程模型 |
| 快速 + 低成本           | `llama-3.2-3b`                               | 紧凑的私密模型                  |
| 复杂的私密任务  | `deepseek-v3.2`                              | 强大的推理能力；工具调用已禁用 |
| 无审查             | `venice-uncensored-1-2`                      | Venice 当前的无审查模型        |
</Tip>

## 内置目录（30 个模型）

<AccordionGroup>
  <Accordion title="私密模型（20 个）— 完全私密，不记录日志">
    | 模型 ID                               | 名称                                 | 上下文 | 备注                      |
    | -------------------------------------- | ------------------------------------- | ------- | --------------------------- |
    | `kimi-k2-5`                            | Kimi K2.5                             | 256k    | 默认、推理、视觉  |
    | `llama-3.3-70b`                        | Llama 3.3 70B                         | 128k    | 通用                     |
    | `llama-3.2-3b`                         | Llama 3.2 3B                          | 128k    | 通用                     |
    | `hermes-3-llama-3.1-405b`              | Hermes 3 Llama 3.1 405B               | 128k    | 通用，工具已禁用     |
    | `qwen3-235b-a22b-thinking-2507`        | Qwen3 235B Thinking                   | 128k    | 推理                   |
    | `qwen3-235b-a22b-instruct-2507`        | Qwen3 235B Instruct                   | 128k    | 通用                     |
    | `qwen3-coder-480b-a35b-instruct-turbo` | Qwen3 Coder 480B Turbo                | 256k    | 编程                      |
    | `qwen3-5-35b-a3b`                      | Qwen3.5 35B A3B                       | 256k    | 推理、视觉           |
    | `qwen3-next-80b`                       | Qwen3 Next 80B                        | 256k    | 通用                     |
    | `qwen3-vl-235b-a22b`                   | Qwen3 VL 235B（视觉）                | 256k    | 视觉                      |
    | `deepseek-v3.2`                        | DeepSeek V3.2                         | 160k    | 推理，工具已禁用    |
    | `google-gemma-3-27b-it`                | Google Gemma 3 27B Instruct           | 198k    | 视觉                       |
    | `openai-gpt-oss-120b`                  | OpenAI GPT OSS 120B                   | 128k    | 通用                      |
    | `nvidia-nemotron-3-nano-30b-a3b`       | NVIDIA Nemotron 3 Nano 30B            | 128k    | 通用                      |
    | `olafangensan-glm-4.7-flash-heretic`   | GLM 4.7 Flash Heretic                 | 128k    | 推理                    |
    | `zai-org-glm-4.6`                      | GLM 4.6                               | 198k    | 通用                      |
    | `zai-org-glm-4.7`                      | GLM 4.7                               | 198k    | 推理                    |
    | `zai-org-glm-4.7-flash`                | GLM 4.7 Flash                         | 128k    | 推理                    |
    | `zai-org-glm-5`                        | GLM 5                                 | 198k    | 推理                    |
    | `minimax-m25`                          | MiniMax M2.5                          | 198k    | 推理                    |
  </Accordion>

  <Accordion title="匿名化模型（10 个）— 通过 Venice 代理">
    | 模型 ID                        | 名称                           | 上下文 | 备注                      |
    | -------------------------------- | -------------------------------- | ------- | ---------------------------- |
    | `claude-opus-4-6`               | Claude Opus 4.6（通过 Venice）    | 1M      | 推理、视觉            |
    | `claude-sonnet-4-6`             | Claude Sonnet 4.6（通过 Venice）  | 1M      | 推理、视觉            |
    | `openai-gpt-54`                 | GPT-5.4（通过 Venice）            | 1M      | 推理、视觉            |
    | `openai-gpt-53-codex`           | GPT-5.3 Codex（通过 Venice）      | 400k    | 推理、视觉、编程     |
    | `openai-gpt-52`                 | GPT-5.2（通过 Venice）            | 256k    | 推理                    |
    | `openai-gpt-52-codex`           | GPT-5.2 Codex（通过 Venice）      | 256k    | 推理、视觉、编程     |
    | `openai-gpt-4o-2024-11-20`      | GPT-4o（通过 Venice）             | 128k    | 视觉                        |
    | `openai-gpt-4o-mini-2024-07-18` | GPT-4o Mini（通过 Venice）        | 128k    | 视觉                        |
    | `gemini-3-1-pro-preview`        | Gemini 3.1 Pro（通过 Venice）     | 1M      | 推理、视觉             |
    | `gemini-3-flash-preview`        | Gemini 3 Flash（通过 Venice）     | 256k    | 推理、视觉             |
  </Accordion>
</AccordionGroup>

由 Grok 支持的 Venice 模型（`grok-4-3` 及类似模型）会获得与原生 xAI 提供商相同的工具架构
兼容性补丁，因为它们使用相同的上游
工具调用格式。

## 模型发现

上述内置目录是由清单支持的种子列表。在运行时，OpenClaw
会从 Venice 的 `/models` API 刷新该目录；如果
API 无法访问，则回退到种子列表。`/models` 端点是公开的（列出模型无需身份验证），
但推理需要有效的 API 密钥。

Venice 可能会继续接受已停用的模型 ID，将其作为提供商自有的别名。
OpenClaw 目录仅列出 `/models` 返回的规范模型 ID。

## DeepSeek V4 重放行为

如果 Venice 提供 DeepSeek V4 模型，例如 `deepseek-v4-pro` 或
`deepseek-v4-flash`，当 Venice 省略必需的 `reasoning_content` 重放
字段时，OpenClaw 会在助手消息中补充该字段，并从请求载荷中移除 `thinking`/
`reasoning`/`reasoning_effort`（Venice 会拒绝这些模型上的
DeepSeek 原生 `thinking` 控制项）。此重放修复与
原生 DeepSeek 提供商自身的思考控制项相互独立。

## 流式传输和工具支持

| 功能          | 支持情况                                           |
| ---------------- | ------------------------------------------------- |
| 流式传输        | 所有模型                                        |
| 函数调用 | 大多数模型；上述注明的模型已禁用 |
| 视觉/图像    | 上述标记为“视觉”的模型                      |
| JSON 模式        | 通过 `response_format`                             |

## 定价

Venice 使用积分制。匿名化模型的费用大致等于
直接 API 定价加上少量 Venice 费用。当前费率请参阅
[venice.ai/pricing](https://venice.ai/pricing)。

## 使用示例

```bash
# 默认私密模型
openclaw agent --model venice/kimi-k2-5 --message "快速健康检查"

# 通过 Venice 使用 Claude Opus（匿名化）
openclaw agent --model venice/claude-opus-4-6 --message "总结此任务"

# 无审查模型
openclaw agent --model venice/venice-uncensored-1-2 --message "拟定选项"

# 使用图像的视觉模型
openclaw agent --model venice/qwen3-vl-235b-a22b --message "审查附带的图像"

# 编程模型
openclaw agent --model venice/qwen3-coder-480b-a35b-instruct-turbo --message "重构此函数"
```

## 故障排查

<AccordionGroup>
  <Accordion title="无法识别 API 密钥">
    ```bash
    echo $VENICE_API_KEY
    openclaw models list | grep venice
    ```

    确认密钥以 `vapi_` 开头。

  </Accordion>

  <Accordion title="模型不可用">
    运行 `openclaw models list --all --provider venice` 查看当前
    可用的模型；随着 Venice 添加或停用模型，目录会随之变化。
  </Accordion>

  <Accordion title="连接问题">
    Venice API 位于 `https://api.venice.ai/api/v1`。确认你的网络允许通过 HTTPS 访问该主机。
  </Accordion>
</AccordionGroup>

<Note>
更多帮助：[故障排查](/zh-CN/help/troubleshooting)和[常见问题](/zh-CN/help/faq)。
</Note>

## 高级配置

<AccordionGroup>
  <Accordion title="配置文件示例">
    ```json5
    {
      env: { VENICE_API_KEY: "vapi_..." },
      agents: { defaults: { model: { primary: "venice/kimi-k2-5" } } },
      models: {
        mode: "merge",
        providers: {
          venice: {
            baseUrl: "https://api.venice.ai/api/v1",
            apiKey: "${VENICE_API_KEY}",
            api: "openai-completions",
            models: [
              {
                id: "kimi-k2-5",
                name: "Kimi K2.5",
                reasoning: true,
                input: ["text", "image"],
                cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 },
                contextWindow: 256000,
                maxTokens: 65536,
              },
            ],
          },
        },
      },
    }
    ```
  </Accordion>
</AccordionGroup>

## 相关内容

<CardGroup cols={2}>
  <Card title="模型选择" href="/zh-CN/concepts/model-providers" icon="layers">
    选择提供商、模型引用和故障转移行为。
  </Card>
  <Card title="Venice AI" href="https://venice.ai" icon="globe">
    Venice AI 主页和账户注册。
  </Card>
  <Card title="API 文档" href="https://docs.venice.ai" icon="book">
    Venice API 参考和开发者文档。
  </Card>
  <Card title="定价" href="https://venice.ai/pricing" icon="credit-card">
    当前 Venice 积分费率和套餐。
  </Card>
</CardGroup>
