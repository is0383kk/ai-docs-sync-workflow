---
read_when:
    - 你希望使用 Deepgram 为音频附件提供语音转文字功能
    - 你希望为语音通话使用 Deepgram 流式转录
    - 你需要一个简短的 Deepgram 配置示例
summary: 使用 Deepgram 转录传入的语音留言
title: Deepgram
x-i18n:
    generated_at: "2026-07-26T06:29:56Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: c00473762c3bede1f6de9230043827d90daefd68d05e67ed4b3e3026b9d6ba4f
    source_path: providers/deepgram.md
    workflow: 16
---

Deepgram 是一个语音转文本 API。OpenClaw 通过 `tools.media.audio` 使用它转录入站音频/语音消息，并通过 `plugins.entries.voice-call.config.streaming` 将其用于语音通话流式 STT。

批量转录会将完整音频文件上传到 Deepgram，并将转录文本注入回复流水线（`{{Transcript}}` + `[Audio]` 块）。
语音通话流式传输通过 Deepgram 的 WebSocket `listen` 端点转发实时 G.711 u-law 帧，并在 Deepgram 返回部分/最终转录文本时将其发出。

| 详情          | 值                                                         |
| ------------- | ---------------------------------------------------------- |
| 网站          | [deepgram.com](https://deepgram.com)                       |
| 文档          | [developers.deepgram.com](https://developers.deepgram.com) |
| 身份验证      | `DEEPGRAM_API_KEY`                                         |
| 默认模型      | `nova-3`                                         |

## 入门指南

<Steps>
  <Step title="设置 API key">
    ```bash
    DEEPGRAM_API_KEY=dg_...
    ```
  </Step>
  <Step title="启用音频提供商">
    ```json5
    {
      tools: {
        media: {
          audio: {
            enabled: true,
            models: [{ provider: "deepgram", model: "nova-3" }],
          },
        },
      },
    }
    ```
  </Step>
  <Step title="发送语音消息">
    通过任意已连接的渠道发送一条音频消息。OpenClaw 会通过 Deepgram 转录该消息，
    并将转录文本注入回复流水线。
  </Step>
</Steps>

## 配置选项

| 选项       | 路径                            | 说明                                  |
| ---------- | ------------------------------- | ------------------------------------- |
| `model`    | `tools.media.models[].model`    | Deepgram 模型 ID（默认值：`nova-3`） |
| `language` | `tools.media.models[].language` | 语言提示（可选）                      |

`providerOptions.deepgram` 会将额外查询参数直接合并到
Deepgram `/listen` 请求中，因此可以使用 Deepgram 支持的任意参数名称
（例如 `detect_language`、`punctuate`、`smart_format`）：

<Tabs>
  <Tab title="包含语言提示">
    ```json5
    {
      tools: {
        media: {
          audio: {
            enabled: true,
            models: [{ provider: "deepgram", model: "nova-3", language: "en" }],
          },
        },
      },
    }
    ```
  </Tab>
  <Tab title="包含 Deepgram 选项">
    ```json5
    {
      tools: {
        media: {
          audio: {
            enabled: true,
            providerOptions: {
              deepgram: {
                detect_language: true,
                punctuate: true,
                smart_format: true,
              },
            },
            models: [{ provider: "deepgram", model: "nova-3" }],
          },
        },
      },
    }
    ```
  </Tab>
</Tabs>

## 语音通话流式 STT

内置的 `deepgram` 插件还会为语音通话插件注册实时转录提供商。

| 设置            | 配置路径                                                                | 默认值                                      |
| --------------- | ----------------------------------------------------------------------- | ------------------------------------------- |
| API key         | `plugins.entries.voice-call.config.streaming.providers.deepgram.apiKey` | 回退到 `DEEPGRAM_API_KEY`                   |
| 基础 URL        | `...deepgram.baseUrl`                                                   | `DEEPGRAM_BASE_URL` 或 Deepgram 的公共 API   |
| 模型            | `...deepgram.model`                                                     | `nova-3`                          |
| 语言            | `...deepgram.language`                                                  | （未设置）                                  |
| 编码            | `...deepgram.encoding`                                                  | `mulaw`                          |
| 采样率          | `...deepgram.sampleRate`                                                | `8000`                          |
| 端点检测        | `...deepgram.endpointingMs`                                             | `800`                          |
| 中间结果        | `...deepgram.interimResults`                                            | `true`                          |

```json5
{
  plugins: {
    entries: {
      "voice-call": {
        config: {
          streaming: {
            enabled: true,
            provider: "deepgram",
            providers: {
              deepgram: {
                apiKey: "${DEEPGRAM_API_KEY}",
                model: "nova-3",
                endpointingMs: 800,
                language: "en-US",
              },
            },
          },
        },
      },
    },
  },
}
```

对于 [Deepgram 自定义端点](https://developers.deepgram.com/reference/custom-endpoints)，
将 `baseUrl` 设置为端点根地址，包括任何基础路径，但不包括 `/listen`。
实时端点接受 `http://`、`https://`、`ws://` 和 `wss://`。HTTP
映射到 WS，HTTPS 映射到 WSS，而显式 WebSocket 协议保持不变。
格式错误的 URL 和其他协议会导致会话设置失败。

<Note>
语音通话接收 8 kHz G.711 u-law 格式的电话音频。Deepgram
流式传输提供商默认使用 `encoding: "mulaw"` 和 `sampleRate: 8000`，因此
可以直接转发 Twilio 媒体帧。
</Note>

## 注意事项

<AccordionGroup>
  <Accordion title="身份验证">
    身份验证遵循标准的提供商身份验证顺序。`DEEPGRAM_API_KEY` 是
    最简单的方式。
  </Accordion>
  <Accordion title="代理和自定义端点">
    使用代理时，请在 Deepgram `tools.media.models[]` 条目中覆盖端点或标头。
  </Accordion>
  <Accordion title="输出行为">
    输出遵循与其他提供商相同的音频规则（大小上限、超时、
    转录文本注入）。
  </Accordion>
</AccordionGroup>

## 相关内容

<CardGroup cols={2}>
  <Card title="媒体工具" href="/zh-CN/tools/media-overview" icon="photo-film">
    音频、图像和视频处理流水线概览。
  </Card>
  <Card title="配置" href="/zh-CN/gateway/configuration" icon="gear">
    完整配置参考，包括媒体工具设置。
  </Card>
  <Card title="故障排查" href="/zh-CN/help/troubleshooting" icon="wrench">
    常见问题和调试步骤。
  </Card>
  <Card title="常见问题" href="/zh-CN/help/faq" icon="circle-question">
    有关 OpenClaw 设置的常见问题。
  </Card>
</CardGroup>
