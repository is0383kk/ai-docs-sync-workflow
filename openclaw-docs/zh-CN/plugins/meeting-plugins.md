---
read_when:
    - 你希望 OpenClaw 智能体加入视频会议
    - 你正在 Google Meet、Microsoft Teams 会议和 Zoom 会议插件之间进行选择
    - 你需要共享的 Chrome、BlackHole、SoX 或会议模式设置
summary: 选择并配置 Google Meet、Microsoft Teams 或 Zoom 会议参与功能
title: 会议插件
x-i18n:
    generated_at: "2026-07-26T06:15:44Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: f41488de018402e3d5cfd01fa5351cdb6107412477d5d54e2d9e186e0fc8ee94
    source_path: plugins/meeting-plugins.md
    workflow: 16
---

OpenClaw 为 Google Meet、Microsoft Teams 会议和 Zoom 提供了不同的插件。这三个插件都可以通过 Chrome 加入会议，使用相同的参与模式，并且既可在 Gateway 网关主机上运行 Chrome，也可在已配对节点上运行。它们的平台 URL、安装模式和额外能力有所不同。

这些插件用于参与会议。它们与 [Microsoft Teams 频道](/zh-CN/channels/msteams)等消息渠道以及[语音通话插件](/zh-CN/plugins/voice-call)彼此独立。

## 选择插件

| 平台            | 插件                                        | 接受的会议链接                                                                                                  | 安装方式                                    | 参与路径                                                     | 平台特有能力                                                                                                      |
| --------------- | ------------------------------------------- | --------------------------------------------------------------------------------------------------------------- | ------------------------------------------- | ------------------------------------------------------------ | ----------------------------------------------------------------------------------------------------------------- |
| Google Meet     | [`google-meet`](/zh-CN/plugins/google-meet) | `meet.google.com/...`                                                                                              | 从 npm 或 ClawHub 安装；默认启用            | 本地 Chrome、已配对节点上的 Chrome 或 Twilio 电话拨入        | 可以通过 Meet API 或已登录的浏览器创建会议；可以使用 OAuth 读取受支持的 Meet 工件                                 |
| Microsoft Teams | [`teams-meetings`](/plugins/teams-meetings) | `teams.microsoft.com/l/meetup-join/...` 下的工作链接和 `teams.live.com/meet/...` 下的个人版链接                                             | 已内置；默认启用                            | 本地 Chrome 或已配对节点上的 Chrome                          | 以访客身份加入工作版和个人版会议                                                                                  |
| Zoom            | [`zoom-meetings`](/plugins/zoom-meetings) | `zoom.us/j/...` 和 `example.zoom.us/j/...` 等账户子域名                                                           | 已内置；默认启用                            | 本地 Chrome 或已配对节点上的 Chrome                          | 通过 Zoom Web App 以访客身份加入                                                                                   |

需要创建会议、使用 Google API 工件或通过 Twilio 电话参与时，请选择 Google Meet。若要在 Teams 或 Zoom 平台上直接通过浏览器以访客身份参与，请选择对应的插件。Teams 和 Zoom 插件不会创建会议、电话拨入、调用供应商 API，也不会录制音频或视频。

## 选择模式

这三个插件使用相同的模式：

| 模式                 | 行为                                                                                                          | 音频要求                                                |
| -------------------- | ------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------- |
| `agent`   | 实时转录内容会发送给已配置的 OpenClaw 智能体；常规 OpenClaw TTS 会朗读回复。                                  | Chrome 回传语音需要 BlackHole 和 SoX 桥接。             |
| `bidi`   | 实时语音模型直接收听并回复。                                                                                  | Chrome 回传语音需要 BlackHole 和 SoX 桥接。             |
| `transcribe`   | 以仅观察方式加入，并在平台提供字幕时公开一份有界的实时字幕转录。                                              | 不需要 BlackHole 或 SoX 回传语音桥接。                  |

当智能体只需要会议文本时，使用 `transcribe`。如需常规 OpenClaw 推理和工具，使用 `agent`。当低延迟直接语音比让每轮对话经过常规智能体更重要时，使用 `bidi`。

有界实时转录仅在 `transcribe` 模式下保持可用。在所有
三种模式下，通过浏览器加入还会将已完成的字幕行及其派生
摘要持久化到共享状态数据库。离开会议时会完成可见
字幕的收尾并写入摘要；使用 [`openclaw transcripts`](/zh-CN/cli/transcripts)
列出、检查或导出这些内容。此持久化笔记路径不会更改实时
智能体咨询转录，也不会创建音频或视频录制。

自动笔记默认开启。设置 `transcripts.enabled: false` 可在全局禁用
持久化笔记。明确选择的 `transcribe` 会话会保留其
有界实时字幕尾部，但不会写入持久化数据行。字幕是否可用
仍取决于会议平台、账户、语言和主持人策略。

## 准备 Chrome 和音频

Chrome 可以在 Gateway 网关主机上运行，也可以在已配对节点上运行。远程 Chrome 节点必须允许 `browser.proxy` 以及对应的平台命令：

| 插件            | 节点命令               |
| --------------- | ---------------------- |
| Google Meet     | `googlemeet.chrome`     |
| Microsoft Teams | `teamsmeetings.chrome`     |
| Zoom            | `zoommeetings.chrome`     |

要通过 Chrome 使用 `agent` 或 `bidi` 模式，请在 macOS 上运行 Chrome，并在同一主机上安装共享音频依赖项：

```bash
brew install blackhole-2ch sox
sudo reboot
system_profiler SPAudioDataType | grep -i BlackHole
command -v sox
```

当 Chrome 在已配对节点上运行时，Gateway 网关主机仍负责 OpenClaw 智能体和模型凭证。在 `agent` 模式下配置实时转录提供商和 OpenClaw TTS，或在 `bidi` 模式下配置实时语音提供商。各平台指南包含提供商和音频命令选项。

## 安装或禁用插件

Google Meet 需要单独安装；安装后默认启用。Teams 会议和 Zoom 已内置于 OpenClaw 并默认启用：

```bash
# 仅 Google Meet
openclaw plugins install npm:@openclaw/google-meet
```

禁用任何不使用的会议插件：

```bash
openclaw plugins disable google-meet
openclaw plugins disable teams-meetings
openclaw plugins disable zoom-meetings
```

如果你的插件管理路径不会自动重启 Gateway 网关，请手动重启。然后在加入会议之前运行平台设置检查。

## 验证并加入

| 平台            | 设置检查                 | 加入命令                  |
| --------------- | ------------------------ | ------------------------- |
| Google Meet     | `openclaw googlemeet setup`       | `openclaw googlemeet join 'https://meet.google.com/abc-defg-hij'`        |
| Microsoft Teams | `openclaw teamsmeetings setup`       | `openclaw teamsmeetings join 'https://teams.microsoft.com/l/meetup-join/...'`        |
| Zoom            | `openclaw zoommeetings setup`       | `openclaw zoommeetings join 'https://zoom.us/j/1234567890'`        |

任何失败的设置检查都应视为使用相应传输方式和模式的阻断因素。要执行仅观察冒烟测试，请选择 `transcribe` 模式，并在预期出现字幕文本之前，确认状态报告存在通话中的会话。

对于回传语音冒烟测试，验证语音需要的不只是播放命令接受了字节。共享命令对桥接会将当前输出生成内容的有界波形指纹与 BlackHole 麦克风采集路径返回的音频相关联；如果只有输出字节计数器增加或存在无关的参与者音频，Google Meet、Teams 和 Zoom 不会报告 `speechOutputVerified: true`。

## 处理平台策略提示

浏览器自动化可以处理常规的访客姓名、加入前摄像头和麦克风、加入、通话中及离开控件。它不会绕过平台或组织者策略。

- Google Meet 可能要求登录 Google、等待主持人准入或作出浏览器权限决定。
- Microsoft Teams 可能要求租户登录、电子邮件验证或组织者准入。
- Zoom 可能要求身份验证、电子邮件验证、密码、完成 CAPTCHA 或主持人准入；账户也可能禁用通过浏览器加入。

当加入或状态结果报告 `manualActionRequired` 时，请先在同一 OpenClaw Chrome 配置文件中完成所报告的步骤，然后再重试。反复打开新标签页无法解决账户、租户、大厅或 CAPTCHA 门槛。

只能加入操作员获授权添加智能体的会议。当本地政策或同意规则要求披露自动参与、转录或合成语音时，请告知参与者。

## Discord 语音聊天

[Discord 语音频道](/zh-CN/channels/discord#voice-channels)无需浏览器会议自动化即可提供原生的纯音频实时对话。OpenClaw 可以加入语音频道、收听内容、通过 OpenClaw 智能体或实时语音模型处理各轮对话，并朗读回复。即使参与者在同一 Discord 频道中使用视频，它也不会发送或接收摄像头视频或屏幕共享，因此 Discord 语音是相关的实时对话界面，而不是第四个浏览器会议插件。

## 平台指南

- [Google Meet 插件](/zh-CN/plugins/google-meet)
- [Microsoft Teams 会议插件](/plugins/teams-meetings)
- [Zoom 会议插件](/plugins/zoom-meetings)
- [管理插件](/zh-CN/plugins/manage-plugins)
- [浏览器控制](/zh-CN/tools/browser)
