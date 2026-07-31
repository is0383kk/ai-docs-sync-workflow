---
read_when:
    - 在 macOS/iOS/Android 上实现 Talk 模式
    - 更改语音/TTS/打断行为
summary: Talk 模式：通过本地 STT/TTS 和实时语音进行连续语音对话
title: Talk 模式
x-i18n:
    generated_at: "2026-07-26T06:13:29Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: b21319eee169ba898331f87279a2b2a5170441131a1e9cdc85c15b268d165e21
    source_path: nodes/talk.md
    workflow: 16
---

Talk 模式涵盖五种运行时形态：

- **原生 macOS/iOS/Android Talk**：原生语音识别、Gateway 网关聊天和 `talk.speak` TTS。macOS/iOS 上的 Apple Speech 识别可能使用网络服务；Android 的行为取决于已安装的语音服务。节点会公布 `talk` 能力，并声明其支持哪些 `talk.*` 命令。
- **iOS Talk（实时）**：对于选择 `webrtc` 传输方式或省略传输方式的 OpenAI 实时配置，使用客户端所有的 WebRTC。显式的 `gateway-relay`、`provider-websocket` 和非 OpenAI 实时配置仍使用 Gateway 网关所有的中继；非实时配置使用原生语音循环。
- **浏览器 Talk**：客户端所有的 `webrtc`/`provider-websocket` 会话使用 `talk.client.create`，Gateway 网关所有的 `gateway-relay` 会话使用 `talk.session.create`。`managed-room` 保留用于 Gateway 网关移交和对讲机房间。
- **Android Talk（实时）**：通过 `talk.realtime.mode: "realtime"` 和 `talk.realtime.transport: "gateway-relay"` 选择启用。否则，Android 继续使用原生语音识别、Gateway 网关聊天和 `talk.speak`。
- **仅转录客户端**：依次使用 `talk.session.create({ mode: "transcription", transport: "gateway-relay", brain: "none" })`、`talk.session.appendAudio`、`talk.session.cancelTurn` 和 `talk.session.close`，在没有助手语音回复的情况下提供字幕/听写。一次性上传的语音留言仍使用[媒体理解](/zh-CN/nodes/media-understanding)音频路径。

原生 Talk 是一个连续循环：监听语音，通过活动会话将转录文本发送给模型，等待响应，然后通过配置的 Talk 提供商（`talk.speak`）朗读响应。

客户端所有的实时 Talk 通过 `talk.client.toolCall` 转发提供商工具调用，而不是直接调用 `chat.send`。实时咨询处于活动状态时，客户端可以调用 `talk.client.steer` 或 `talk.session.steer`，将语音输入分类为 `status`、`steer`、`cancel` 或 `followup`。已接受的 Steering 会加入活动嵌入式运行的队列；被拒绝的 Steering 会返回原因，例如 `no_active_run`、`not_streaming` 或 `compacting`。

最终确定的实时用户和助手话语始终会实时追加到活动智能体会话中，因此后续的聊天和语音轮次共享同一份历史记录。客户端所有的传输方式使用稳定的条目 ID 报告其最终转录文本；Gateway 网关中继会话则在服务器端追加相同事件。提供商会话还会接收 Discord 语音所使用的有界实时配置文件上下文。

源自语音的咨询运行在执行发送消息、控制节点、浏览器/计算机操作、服务变更、破坏性 shell 命令或发布等高影响操作前，需要新的、明确无误的口头确认。该确认仅适用于被阻止工具的确切参数，并且仅使用一次；其他不相关的并发运行不受影响。通话结束时，OpenClaw 可以将变更型工具的精简版**语音通话变更**摘要发送到该会话最后一个非 WebChat 投递目标。

仅转录 Talk 会发出与实时会话和 STT/TTS 会话相同的 Talk 事件信封，但使用 `mode: "transcription"` 和 `brain: "none"`。所有 Talk 会话都会在 `talk.event` 渠道上广播事件；客户端订阅该渠道，以接收部分/最终转录文本更新（`transcript.delta`/`transcript.done`）及其他会话遥测数据。

浏览器视频 Talk 可用于 OpenAI Realtime WebRTC 和 Google Live
提供商 WebSocket 会话。当 `describe_view` 请求视觉上下文时，
OpenAI 会收到一张大小受限的 JPEG；它不会收到连续的摄像头轨道。
Google Live 直接从浏览器接收大小受限的 JPEG 帧，频率最高为每秒一帧，
同时 `describe_view` 会报告摄像头流状态。在这两种情况下，
摄像头帧均绕过 Gateway 网关，停止 Talk 会释放摄像头和麦克风轨道。

## 行为（macOS）

- 启用 Talk 模式时始终显示浮层。
- **正在聆听 &rarr; 正在思考 &rarr; 正在朗读**阶段转换。
- 短暂停顿（静音窗口）后，会发送当前转录文本。
- 回复会写入 WebChat（与键入文本相同）。
- **语音打断**（默认启用）：如果用户在助手朗读时说话，播放会停止，并记录打断时间戳以供下一个提示使用。

## 回复中的语音指令

助手可以在回复开头添加一行 JSON 来控制语音：

```json
{ "voice": "<voice-id>", "once": true }
```

规则：

- 仅限第一个非空行；TTS 播放前会移除该 JSON 行。
- 未知键会被忽略。
- `once: true` 仅应用于当前回复；如果没有该键，此语音会成为新的 Talk 模式默认值。

支持的键：`voice` / `voice_id` / `voiceId`、`model` / `model_id` / `modelId`、`speed`、`rate`（WPM）、`stability`、`similarity`、`style`、`speakerBoost`、`seed`、`normalize`、`lang`、`output_format`、`latency_tier`、`once`。

## 配置（`~/.openclaw/openclaw.json`）

```json5
{
  talk: {
    provider: "elevenlabs",
    providers: {
      elevenlabs: {
        voiceId: "elevenlabs_voice_id",
        modelId: "eleven_v3",
        outputFormat: "mp3_44100_128",
        apiKey: "elevenlabs_api_key",
      },
      mlx: {
        modelId: "mlx-community/Soprano-80M-bf16",
      },
      system: {},
    },
    speechLocale: "ru-RU",
    silenceTimeoutMs: 1500,
    interruptOnSpeech: true,
    realtime: {
      provider: "openai",
      providers: {
        openai: {
          apiKey: "openai_api_key",
          model: "gpt-realtime-2.1",
          speakerVoice: "cedar",
        },
      },
      instructions: "以温和的语气说话，并保持回答简短。",
      mode: "realtime",
      transport: "webrtc",
      brain: "agent-consult",
    },
  },
}
```

| 键                                       | 默认值                                     | 说明                                                                                                                                                                                                                                                                       |
| ---------------------------------------- | ------------------------------------------ | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `provider`                               | -                                          | 当前使用的 Talk TTS 提供商。对于 macOS 本地播放路径，请使用 `elevenlabs`、`mlx` 或 `system`。                                                                                                                                                                             |
| `providers.<id>.voiceId`                 | -                                          | ElevenLabs 会回退到 `ELEVENLABS_VOICE_ID` / `SAG_VOICE_ID`，或使用第一个具有 API key 的可用语音。                                                                                                                                                             |
| `speechLocale`                           | 设备默认值                             | Android、iOS 和 macOS 原生语音识别所用的 BCP 47 区域设置。Apple Speech 可能会使用网络服务；Android 还会将语言部分转发给实时输入转录。                                                                                  |
| `providers.elevenlabs.modelId`           | `eleven_v3`                                |                                                                                                                                                                                                                                                                            |
| `providers.mlx.modelId`                  | `mlx-community/Soprano-80M-bf16`           |                                                                                                                                                                                                                                                                            |
| `providers.elevenlabs.apiKey`            | -                                          | 回退到 `ELEVENLABS_API_KEY`（如果可用，则使用 Gateway 网关 shell 配置文件）。                                                                                                                                                                                                |
| `silenceTimeoutMs`                       | macOS/Android 为 `700` ms，iOS 为 `900` ms       | Talk 发送转录文本前的暂停时长。                                                                                                                                                                                                                             |
| `interruptOnSpeech`                      | `true`                                     |                                                                                                                                                                                                                                                                            |
| `outputFormat`                           | macOS/iOS 为 `pcm_44100`，Android 为 `pcm_24000` | 设置 `mp3_*` 以强制使用 MP3 流式传输。                                                                                                                                                                                                                                        |
| `consultThinkingLevel`                   | 未设置                                      | 实时 `openclaw_agent_consult` 调用背后的智能体运行所用的思考级别覆盖值。                                                                                                                                                                                  |
| `consultFastMode`                        | 未设置                                      | 实时 `openclaw_agent_consult` 调用的快速模式覆盖值。                                                                                                                                                                                                            |
| `realtime.provider`                      | -                                          | `openai` 用于 WebRTC，`google` 用于提供商 WebSocket，也可使用通过 Gateway 网关中继的仅桥接提供商。                                                                                                                                                                     |
| `realtime.providers.<id>`                | -                                          | 由提供商管理的实时配置。浏览器只会收到临时或受限的会话凭据，绝不会收到标准 API key。                                                                                                                                                 |
| `realtime.providers.openai.speakerVoice` | `alloy`                                    | 内置 OpenAI Realtime 语音 ID（旧版 `voice` 键仍可使用，但已弃用）。当前的 `gpt-realtime-2.1` 语音包括：`alloy`、`ash`、`ballad`、`cedar`、`coral`、`echo`、`marin`、`sage`、`shimmer`、`verse`；建议使用 `marin` 和 `cedar` 以获得最佳质量。 |
| `realtime.transport`                     | -                                          | `webrtc`：在 iOS 和浏览器中由客户端管理的 OpenAI WebRTC。`provider-websocket`：由浏览器管理，在 iOS 上仍通过 Gateway 网关中继。`gateway-relay`：将提供商音频保留在 Gateway 网关上；Android 仅在使用此传输方式时支持实时功能。                                  |
| `realtime.brain`                         | -                                          | `agent-consult` 通过 Gateway 网关策略路由实时工具调用；`direct-tools` 是旧版直接工具兼容模式；`none` 用于转录或外部编排。                                                                                                 |
| `realtime.consultRouting`                | -                                          | 当提供商跳过 `openclaw_agent_consult` 时，`provider-direct` 会保留提供商的直接回复；`force-agent-consult` 则通过 OpenClaw 路由已定稿的用户转录文本。                                                                                          |
| `realtime.instructions`                  | -                                          | 将面向提供商的系统指令附加到 OpenClaw 的内置实时提示词中（语音风格/语气）；默认的 `openclaw_agent_consult` 指引保持不变。                                                                                                                |

`talk.catalog` 会公开规范提供商 ID 和注册表别名、每个提供商的有效模式/传输方式/思维策略/实时音频格式/能力标志，以及运行时选择的就绪状态结果。第一方 Talk 客户端应读取该目录，而不是在本地维护提供商别名；如果旧版 Gateway 网关未提供分组就绪状态，应将其视为未经验证，而不是明确判定为未配置。流式转录提供商通过 `talk.catalog.transcription` 发现；在专用 Talk 转录配置界面发布之前，当前 Gateway 网关中继使用语音通话流式提供商配置。

## macOS UI

- 菜单栏开关：**Talk**
- 配置标签页：**Talk Mode** 组（语音 ID + 打断开关）
- 浮层：圆球会呈现通用 Talk 波形（与 iOS、watchOS 和 Android 共用）。聆听时跟随实时麦克风电平，发声时跟随实际 TTS 播放包络，思考时轻柔呼吸。单击圆球可暂停/继续，双击可停止发声，单击 X 可退出 Talk 模式。

## Android UI

- Android 的主导航为 **Home**、**Chat** 和 **Settings**。语音输入
  位于 Chat 编辑器中，而不是单独的 Voice 标签页中。
- 点按编辑器中的麦克风可使用设备端听写。长按可录制
  语音消息附件。从 Talk 波形启动连续 Talk。
- 听写、语音消息录制和 Talk 是互斥的麦克风
  路径；启动其中一项会停止或阻止其他路径。
- 实时 Talk 优先使用已连接的 Bluetooth Classic 或 BLE 耳机
  麦克风；如果连接断开，应用会请求另一个耳机输入或
  回退到默认麦克风，并在采集停止后
  恢复默认偏好设置。
- 当应用离开前台或
  用户离开 Chat 时，听写和语音消息录制会停止。
- Talk 模式会持续运行，直到关闭开关或节点断开连接；运行期间使用 Android 的麦克风前台服务类型。
- Android 支持 `pcm_16000`、`pcm_22050`、`pcm_24000` 和 `pcm_44100` 输出格式，以实现低延迟 `AudioTrack` 流式传输。

## 注意事项

- 需要语音和麦克风权限。
- 原生 Talk 使用当前 Gateway 网关会话，仅当响应事件不可用时才回退到历史记录轮询。
- Gateway 网关使用当前 Talk 提供商，通过 `talk.speak` 解析 Talk 播放。仅当该 RPC 不可用时，Android 才会回退到本地系统 TTS。
- macOS 本地 MLX 播放会使用内置的 `openclaw-mlx-tts` 辅助程序（如果存在），或使用 `PATH` 上的可执行文件。开发期间可设置 `OPENCLAW_MLX_TTS_BIN`，使其指向自定义辅助程序二进制文件。
- 语音指令值范围（ElevenLabs）：`stability`、`similarity` 和 `style` 接受 `0..1`；`speed` 接受 `0.5..2`；`latency_tier` 接受 `0..4`。

## 相关内容

- [语音唤醒](/zh-CN/nodes/voicewake)
- [音频和语音消息](/zh-CN/nodes/audio)
- [媒体理解](/zh-CN/nodes/media-understanding)
