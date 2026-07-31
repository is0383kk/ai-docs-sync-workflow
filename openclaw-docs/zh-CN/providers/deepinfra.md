---
read_when:
    - 你希望使用一个 API 密钥访问顶尖的开源大语言模型
    - 你想通过 DeepInfra 的 API 在 OpenClaw 中运行模型
summary: 在 OpenClaw 中使用 DeepInfra 的统一 API 访问最热门的开源模型和前沿模型
title: DeepInfra
x-i18n:
    generated_at: "2026-07-26T06:25:11Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: a63bdd4ffd2189cde50f0ee601fd7ee32ca86c943a9899072f0c140823608004
    source_path: providers/deepinfra.md
    workflow: 16
---

DeepInfra 通过单个兼容 OpenAI 的端点和 API key，将请求路由到热门开源模型和前沿模型。大多数 OpenAI SDK 只需切换基础 URL 即可使用它。

## 安装插件

```bash
openclaw plugins install @openclaw/deepinfra-provider
openclaw gateway restart
```

## 获取 API key

1. 在 [deepinfra.com](https://deepinfra.com/) 登录
2. 前往 Dashboard / Keys 并生成一个 key，或使用自动创建的 key

## CLI 设置

```bash
openclaw onboard --deepinfra-api-key <key>
```

或者设置环境变量：

```bash
export DEEPINFRA_API_KEY="<your-deepinfra-api-key>" # pragma: allowlist secret
```

## 配置片段

```json5
{
  env: { DEEPINFRA_API_KEY: "<your-deepinfra-api-key>" }, // pragma: allowlist secret
  agents: {
    defaults: {
      model: { primary: "deepinfra/deepseek-ai/DeepSeek-V4-Flash" },
    },
  },
}
```

## 支持的功能界面

配置 `DEEPINFRA_API_KEY` 后，聊天、图像生成和视频生成会从 `https://api.deepinfra.com/v1/openai/models?sort_by=openclaw&filter=with_meta` 实时刷新其模型目录。实时发现会扩展可选模型列表；每个功能界面的默认模型仍为下方的静态值。其他功能界面在迁移到同一实时目录之前，会继续使用静态目录。

| 功能界面                 | 默认模型                                                                       | OpenClaw 配置/工具                                    |
| ------------------------ | ------------------------------------------------------------------------------ | ----------------------------------------------------- |
| 聊天 / 模型提供商        | `deepseek-ai/DeepSeek-V4-Flash`（实时目录会添加更多聊天模型）                               | `agents.defaults.model`                                    |
| 图像生成/编辑            | `black-forest-labs/FLUX-1-schnell`（实时目录会添加更多 `image-gen` 模型）               | `image_generate`、`agents.defaults.mediaModels.image`                |
| 媒体理解                 | 图像使用 `moonshotai/Kimi-K2.5`                                                    | 入站图像理解                                          |
| 语音转文本               | `openai/whisper-large-v3-turbo`                                                             | 入站音频转录                                          |
| 文本转语音               | `hexgrad/Kokoro-82M`                                                             | `tts.provider: "deepinfra"`                                    |
| 视频生成                 | `Pixverse/Pixverse-T2V`（实时目录会添加更多 `video-gen` 模型）               | `video_generate`、`agents.defaults.mediaModels.video`                |
| 记忆嵌入                 | `BAAI/bge-m3`                                                             | `memory.search.provider: "deepinfra"`                                    |

DeepInfra 还提供重排序、分类、对象检测和其他原生模型类型。OpenClaw 尚未为这些类别提供提供商契约，因此此插件不会注册它们。

## 可用模型

配置 key 后，OpenClaw 会动态发现 DeepInfra 模型。使用 `/models deepinfra` 或 `openclaw models list --provider deepinfra` 查看当前列表。

[deepinfra.com](https://deepinfra.com/) 上的任何模型都可以使用 `deepinfra/` 前缀：

```text
deepinfra/deepseek-ai/DeepSeek-V4-Flash
deepinfra/deepseek-ai/DeepSeek-V3.2
deepinfra/MiniMaxAI/MiniMax-M2.5
deepinfra/moonshotai/Kimi-K2.5
deepinfra/nvidia/NVIDIA-Nemotron-3-Super-120B-A12B
deepinfra/zai-org/GLM-5.1
……以及更多模型
```

## 注意事项

- 模型引用格式为 `deepinfra/<provider>/<model>`（例如 `deepinfra/Qwen/Qwen3-Max`）。
- 默认聊天模型：`deepinfra/deepseek-ai/DeepSeek-V4-Flash`
- 基础 URL：`https://api.deepinfra.com/v1/openai`
- 视频生成使用兼容 OpenAI 的异步端点 `https://api.deepinfra.com/v1/openai/videos`（先提交，然后轮询）。配置的 `baseUrl` 会生效。`openclaw doctor --fix` 会在 `api.deepinfra.com` 上自动将旧版 `nativeBaseUrl` 或 `/v1/inference` 值迁移到 `baseUrl`；自定义原生端点已停用，Doctor 会发出通知，并且需要手动配置兼容 OpenAI 的 `baseUrl`。当 `baseUrl` 仍指向已停用的 `/v1/inference` 功能界面时，视频生成会在发送任何请求之前失败，并返回可操作的错误。

## 相关内容

- [模型提供商](/zh-CN/concepts/model-providers)
- [所有提供商](/zh-CN/providers/index)
