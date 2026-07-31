---
read_when:
    - 你想将 Cerebras 与 OpenClaw 搭配使用
    - 你需要设置 Cerebras API key 环境变量或选择 CLI 身份验证方式
summary: Cerebras 设置（身份验证 + 模型选择）
title: Cerebras
x-i18n:
    generated_at: "2026-07-26T06:29:31Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 716eef83155ef80d9aa61bd55ed83e3e38ad22720ae055bce7eb9c2cbfb6cf41
    source_path: providers/cerebras.md
    workflow: 16
---

[Cerebras](https://www.cerebras.ai) 在自定义推理硬件上提供高速、兼容 OpenAI 的推理服务。该插件附带包含两个模型的静态目录（不支持实时发现）。

| 属性            | 值                                                        |
| --------------- | --------------------------------------------------------- |
| 提供商 ID       | `cerebras`                                        |
| 插件            | 官方外部软件包（`@openclaw/cerebras-provider`）                      |
| 身份验证环境变量 | `CEREBRAS_API_KEY`                                        |
| 新手引导标志    | `--auth-choice cerebras-api-key`                                        |
| 直接 CLI 标志   | `--cerebras-api-key <key>`                                        |
| API             | 兼容 OpenAI（`openai-completions`）                         |
| 基础 URL        | `https://api.cerebras.ai/v1`                                        |
| 默认模型        | `cerebras/zai-glm-4.7`                                        |

## 安装插件

```bash
openclaw plugins install @openclaw/cerebras-provider
openclaw gateway restart
```

## 入门指南

<Steps>
  <Step title="获取 API key">
    在 [Cerebras Cloud Console](https://cloud.cerebras.ai) 中创建 API key。
  </Step>
  <Step title="运行新手引导">
    <CodeGroup>

```bash Onboarding
openclaw onboard --auth-choice cerebras-api-key
```

```bash Direct flag
openclaw onboard --non-interactive \
  --auth-choice cerebras-api-key \
  --cerebras-api-key "$CEREBRAS_API_KEY"
```

```bash Env only
export CEREBRAS_API_KEY=csk-...
```

    </CodeGroup>

  </Step>
  <Step title="验证模型是否可用">
    ```bash
    openclaw models list --provider cerebras
    ```

    列出两个静态模型。如果 `CEREBRAS_API_KEY` 未解析，`openclaw models status --json` 会在 `auth.unusableProfiles` 下报告缺失的凭据。

  </Step>
</Steps>

## 非交互式设置

```bash
openclaw onboard --non-interactive \
  --mode local \
  --auth-choice cerebras-api-key \
  --cerebras-api-key "$CEREBRAS_API_KEY"
```

## 内置目录

两个模型都具有 128k 上下文窗口和最多 8,192 个输出 token。

| 模型引用                | 名称         | 推理 | 说明                     |
| ----------------------- | ------------ | ---- | ------------------------ |
| `cerebras/zai-glm-4.7`      | Z.ai GLM 4.7 | 是   | 默认模型；预览版推理模型 |
| `cerebras/gpt-oss-120b`      | GPT OSS 120B | 是   | 生产级推理模型           |

## 手动配置

大多数设置只需要 API key。使用显式 `models.providers.cerebras` 配置可覆盖模型元数据，或通过 `mode: "merge"` 基于静态目录运行：

```json5
{
  env: { CEREBRAS_API_KEY: "csk-..." },
  agents: {
    defaults: {
      model: { primary: "cerebras/zai-glm-4.7" },
    },
  },
  models: {
    mode: "merge",
    providers: {
      cerebras: {
        baseUrl: "https://api.cerebras.ai/v1",
        apiKey: "${CEREBRAS_API_KEY}",
        api: "openai-completions",
        models: [
          { id: "zai-glm-4.7", name: "Z.ai GLM 4.7" },
          { id: "gpt-oss-120b", name: "GPT OSS 120B" },
        ],
      },
    },
  },
}
```

<Note>
如果 Gateway 网关以守护进程（launchd、systemd、Docker）的形式运行，请确保 `CEREBRAS_API_KEY` 对该进程可用——例如在 `~/.openclaw/.env` 中设置，或通过 `env.shellEnv` 提供。仅在交互式 shell 中导出的密钥无法供托管服务使用，除非单独导入该环境变量。
</Note>

## 相关内容

<CardGroup cols={2}>
  <Card title="模型提供商" href="/zh-CN/concepts/model-providers" icon="layers">
    选择提供商、模型引用和故障转移行为。
  </Card>
  <Card title="思考模式" href="/zh-CN/tools/thinking" icon="brain">
    两个支持推理的 Cerebras 模型的推理投入级别。
  </Card>
  <Card title="配置参考" href="/zh-CN/gateway/config-agents#agent-defaults" icon="gear">
    Agent 默认设置和模型配置。
  </Card>
  <Card title="模型常见问题" href="/zh-CN/help/faq-models" icon="circle-question">
    身份验证配置文件、切换模型以及解决“无配置文件”错误。
  </Card>
</CardGroup>
