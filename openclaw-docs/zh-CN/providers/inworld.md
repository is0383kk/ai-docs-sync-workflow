---
read_when:
    - 你希望使用 Inworld 语音合成生成出站回复
    - 你需要 Inworld 输出 PCM 电话音频或 OGG_OPUS 语音留言格式
summary: 用于 OpenClaw 回复的 Inworld 流式文本转语音
title: Inworld
x-i18n:
    generated_at: "2026-07-26T06:57:04Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 09560f5beda3b40d9c67f9408d34446f28ecddb8235fc0725c4265c813302946
    source_path: providers/inworld.md
    workflow: 16
---

Inworld 是流式文本转语音（TTS）提供商。在 OpenClaw 中，它为出站回复合成音频（默认为 MP3，语音消息使用 OGG_OPUS），并为语音通话等电话渠道合成原始 PCM 音频。

OpenClaw 向 Inworld 的流式 TTS 端点发出 POST 请求，将返回的 Base64 音频块拼接为单个缓冲区，然后将结果交给标准回复音频流水线。

| 属性          | 值                                                              |
| ------------- | --------------------------------------------------------------- |
| 提供商 ID     | `inworld`                                              |
| 插件          | 官方外部软件包（`@openclaw/inworld-speech`）                            |
| 契约          | `speechProviders`（仅 TTS）                                    |
| 身份验证环境变量 | `INWORLD_API_KEY`（HTTP Basic，Base64 控制面板凭据）         |
| 基础 URL      | `https://api.inworld.ai`                                              |
| 默认语音      | `Sarah`                                              |
| 默认模型      | `inworld-tts-1.5-max`                                              |
| 输出          | MP3（默认）、OGG_OPUS（语音消息）、PCM 22050 Hz（电话）         |
| 网站          | [inworld.ai](https://inworld.ai)                                |
| 文档          | [docs.inworld.ai/tts/tts](https://docs.inworld.ai/tts/tts)      |

## 安装插件

```bash
openclaw plugins install @openclaw/inworld-speech
openclaw gateway restart
```

## 入门指南

<Steps>
  <Step title="设置 API 密钥">
    从 Inworld 控制面板（Workspace > API Keys）复制凭据，并将其设置为环境变量。该值会原样作为 HTTP Basic 凭据发送，因此不要再次对其进行 Base64 编码，也不要将其转换为不记名令牌。

    ```bash
    INWORLD_API_KEY=<base64-credential-from-dashboard>
    ```

  </Step>
  <Step title="在 tts 中选择 Inworld">
    ```json5
    {
      tts: {
        auto: "always",
        provider: "inworld",
        providers: {
          inworld: {
            voiceId: "Sarah",
            modelId: "inworld-tts-1.5-max",
          },
        },
      },
    }
    ```
  </Step>
  <Step title="发送消息">
    通过任意已连接的渠道发送回复。OpenClaw 使用 Inworld 合成音频，并以 MP3 格式传送（当渠道要求语音消息时则使用 OGG_OPUS）。
  </Step>
</Steps>

## 配置选项

| 选项          | 路径                                | 说明                                                                |
| ------------- | ----------------------------------- | ------------------------------------------------------------------- |
| `apiKey`      | `tts.providers.inworld.apiKey`      | Base64 控制面板凭据。回退到 `INWORLD_API_KEY`。                     |
| `baseUrl`     | `tts.providers.inworld.baseUrl`     | 覆盖 Inworld API 基础 URL（默认为 `https://api.inworld.ai`）。             |
| `voiceId`     | `tts.providers.inworld.voiceId`     | 语音标识符（默认为 `Sarah`）。旧版别名：`speakerVoiceId`。 |
| `modelId`     | `tts.providers.inworld.modelId`     | TTS 模型 ID（默认为 `inworld-tts-1.5-max`）。                           |
| `temperature` | `tts.providers.inworld.temperature` | 采样温度，范围为 `0`（不含）至 `2`（可选）。 |

## 注意事项

<AccordionGroup>
  <Accordion title="身份验证">
    Inworld 使用 HTTP Basic 身份验证，其中包含一个经过 Base64 编码的凭据字符串。请从 Inworld 控制面板原样复制该字符串。提供商将其作为 `Authorization: Basic <apiKey>` 发送，不会进行任何进一步编码，因此不要自行对其进行 Base64 编码，也不要传入不记名令牌类型的令牌。相同的注意事项请参阅 [TTS 身份验证说明](/zh-CN/tools/tts#inworld-primary)。
  </Accordion>
  <Accordion title="模型">
    支持的模型 ID：`inworld-tts-1.5-max`（默认）、`inworld-tts-1.5-mini`、`inworld-tts-1-max`、`inworld-tts-1`。
  </Accordion>
  <Accordion title="音频输出">
    回复默认使用 MP3。当渠道目标为 `voice-note` 时，OpenClaw 会请求 Inworld 返回 `OGG_OPUS`，以便音频作为原生语音气泡播放。电话合成使用 22050 Hz 的原始 `PCM` 音频输入电话桥接器。
  </Accordion>
  <Accordion title="自定义端点">
    使用 `tts.providers.inworld.baseUrl` 覆盖 API 主机。发送请求前会移除末尾的斜杠。
  </Accordion>
</AccordionGroup>

## 相关内容

<CardGroup cols={2}>
  <Card title="文本转语音" href="/zh-CN/tools/tts" icon="waveform-lines">
    TTS 概览、提供商和 `tts` 配置。
  </Card>
  <Card title="配置" href="/zh-CN/gateway/configuration" icon="gear">
    完整的配置参考，包括 `tts` 设置。
  </Card>
  <Card title="提供商" href="/zh-CN/providers" icon="grid">
    OpenClaw 支持的所有提供商。
  </Card>
  <Card title="故障排查" href="/zh-CN/help/troubleshooting" icon="wrench">
    常见问题和调试步骤。
  </Card>
</CardGroup>
