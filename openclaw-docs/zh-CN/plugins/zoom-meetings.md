---
read_when:
    - 你希望 OpenClaw 智能体加入 Zoom 会议
    - 你正在为 Zoom 会议的回传语音配置 Chrome、BlackHole 或 SoX
summary: Zoom 会议插件：以 Chrome 浏览器访客身份加入会议
title: Zoom 会议插件
x-i18n:
    generated_at: "2026-07-26T05:58:11Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: d91e57cccb163f634c6eaee71dd3832fc7b9e783fc5cd02601572b302d0d25e8
    source_path: plugins/zoom-meetings.md
    workflow: 16
---

`zoom-meetings` 插件通过 OpenClaw Chrome 配置文件中的 Zoom Web App，以访客身份加入 Zoom 会议链接。它接受 `zoom.us/j/...` 下的会议链接以及 `example.zoom.us/j/...` 等账户子域名。它不会创建会议、电话接入、使用 Zoom Meeting SDK，也不会捕获音频/视频录制内容。

## 设置

语音回复使用与 [Google Meet 插件](/zh-CN/plugins/google-meet)相同的本地音频前提条件：macOS、`BlackHole 2ch` 虚拟音频设备和 SoX。

```bash
brew install blackhole-2ch sox
sudo reboot
system_profiler SPAudioDataType | grep -i BlackHole
command -v sox
```

该插件已内置并默认启用。仅在需要自定义时添加条目，然后检查设置：

```json5
{
  plugins: {
    entries: {
      "zoom-meetings": {
        config: {
          defaultMode: "agent",
          chrome: { guestName: "OpenClaw Agent" },
        },
      },
    },
  },
}
```

如果不希望启用该插件，请运行 `openclaw plugins disable zoom-meetings`。

```bash
openclaw zoommeetings setup
openclaw zoommeetings join 'https://zoom.us/j/1234567890'
```

使用 `chromeNode.node` 可在已配对的 macOS 节点上运行 Chrome、BlackHole 和 SoX。该节点必须允许 `zoommeetings.chrome` 和 `browser.proxy`。

## 模式

| 模式         | 行为                                                                    |
| ------------ | --------------------------------------------------------------------------- |
| `agent`      | 实时转录会咨询已配置的 OpenClaw 智能体；通过 TTS 回复。 |
| `bidi`       | 实时语音模型直接监听并回复。                        |
| `transcribe` | 仅观察加入，并提供实时字幕转录快照。                   |

在所有模式下，获准加入后都会启用 Zoom 实时字幕，以便 OpenClaw
持久保存会议记录。`transcript` 操作仍然仅为 `transcribe`
会话返回有界的实时缓冲区。离开时，OpenClaw 会将持久化转录和派生摘要
存储在共享状态数据库中；可使用 [`openclaw transcripts`](/zh-CN/cli/transcripts)
列出或导出它们。

默认启用自动记录。将 `transcripts.enabled: false` 设置为
禁用全局持久化记录；显式的 `transcribe` 模式仍然只公开
其有界的实时尾部内容。

## 访客加入限制

浏览器适配器会选择 **Join from browser**，填写访客姓名，关闭摄像头，根据所选模式配置麦克风，然后点击 **Join**。Zoom Web App 在 `app.zoom.us` 下运行；插件会在导航前为该来源授予麦克风和扬声器选择权限。通话中状态使用 Zoom 的 Leave 控件。等候室、登录、密码、CAPTCHA 和设备权限状态会返回明确的手动操作原因。

Zoom 主持人和账户策略可能会禁用浏览器加入、要求身份验证或电子邮件验证、显示 CAPTCHA，或要求主持人批准加入。请在 OpenClaw Chrome 配置文件中完成该步骤，然后重试状态检查或语音操作。该插件不会绕过 Zoom 策略。

已使用 Zoom 官方测试会议对 Zoom Web App 进行实际验证，涵盖应用中间页、iframe 访客姓名输入、加入前麦克风和摄像头控件、加入、浏览器和 macOS 媒体权限、通话中检测、启用实时字幕以及主持人结束会议检测。等候室和身份验证状态取决于主持人策略；在没有稳定 DOM 标识符时，会保留文本回退方案。

## 工具和 Gateway 网关接口

`zoom_meetings` 智能体工具支持 `join`、`leave`、`status`、`transcript` 和 `speak`。Gateway 网关方法使用 `zoommeetings.*` 前缀。节点命令为 `zoommeetings.chrome`。

## 相关内容

- [会议插件概览](/plugins/meeting-plugins)
