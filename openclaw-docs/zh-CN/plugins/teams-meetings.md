---
read_when:
    - 你希望让 OpenClaw 智能体加入 Microsoft Teams 会议
    - 你正在为 Teams 会议回传语音配置 Chrome、BlackHole 或 SoX
summary: Microsoft Teams 会议插件：以 Chrome 浏览器访客身份加入工作或个人会议
title: Microsoft Teams 会议插件
x-i18n:
    generated_at: "2026-07-26T05:57:51Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 6f84e58d478185d026dd79a02a8500af48f51689ef6865d56badb0e27c6d2814
    source_path: plugins/teams-meetings.md
    workflow: 16
---

`teams-meetings` 插件以访客身份在 OpenClaw Chrome 配置文件中加入 Microsoft Teams 链接。它接受 `teams.microsoft.com/l/meetup-join/...` 下的工作版链接和 `teams.live.com/meet/...` 下的个人版链接。它不会创建会议、拨号接入、调用 Microsoft Graph，也不会捕获音频/视频录制内容。

## 设置

语音回应使用与 [Google Meet 插件](/zh-CN/plugins/google-meet)相同的本地音频前置条件：macOS、`BlackHole 2ch` 虚拟音频设备和 SoX。

```bash
brew install blackhole-2ch sox
sudo reboot
system_profiler SPAudioDataType | grep -i BlackHole
command -v sox
```

该插件已包含在内，并默认启用。仅在需要自定义时添加条目，然后检查设置：

```json5
{
  plugins: {
    entries: {
      "teams-meetings": {
        config: {
          defaultMode: "agent",
          chrome: { guestName: "OpenClaw Agent" },
        },
      },
    },
  },
}
```

如果不希望启用该插件，请运行 `openclaw plugins disable teams-meetings`。

```bash
openclaw teamsmeetings setup
openclaw teamsmeetings join 'https://teams.microsoft.com/l/meetup-join/...'
```

使用 `chromeNode.node` 在已配对的 macOS 节点上运行 Chrome、BlackHole 和 SoX。该节点必须允许 `teamsmeetings.chrome` 和 `browser.proxy`。

## 模式

| 模式         | 行为                                                                    |
| ------------ | --------------------------------------------------------------------------- |
| `agent`      | 实时转写会咨询已配置的 OpenClaw 智能体；通过 TTS 回应。 |
| `bidi`       | 实时语音模型直接监听并回应。                        |
| `transcribe` | 以仅观察方式加入，并获取实时字幕转写快照。                   |

在所有模式下，获准加入后都会启用 Teams 实时字幕，以便 OpenClaw
持久保存带发言人归属的笔记。`transcript` 操作仍然仅为
`transcribe` 会话返回有界实时缓冲区。离开时，OpenClaw 会将
持久转写记录和派生摘要存储到共享状态数据库中；可使用
[`openclaw transcripts`](/zh-CN/cli/transcripts)列出或导出它们。

自动笔记默认启用。将 `transcripts.enabled: false` 设置为
禁用全局持久笔记；显式 `transcribe` 模式仍然仅公开
其有界实时尾部内容。

## 访客加入限制

浏览器适配器会关闭应用启动提示页、填写访客姓名、关闭摄像头、根据所选模式配置麦克风，并点击加入按钮。通话中状态使用挂断控件；处于大厅、租户登录和设备权限状态时，会返回明确的手动操作原因。支持个人版会议启动器重定向，以及 Chrome 显示的 `BlackHole 2ch (Virtual)` 标签。

Teams 租户策略可能要求登录、电子邮件验证或组织者批准。请在 OpenClaw Chrome 配置文件中完成该步骤，然后重试状态检查或语音操作。该插件不会绕过租户策略。

个人版 Teams Web 客户端已对以下流程进行实时验证：应用启动提示页、访客姓名输入、加入前的麦克风/摄像头开关、加入、大厅准入、媒体权限、通话中状态检测、实时字幕、BlackHole 输入/输出路由、离开以及通话后状态检测。工作版租户可能实施不同的登录、电子邮件验证、准入和离开确认策略；请在 OpenClaw Chrome 配置文件中完成所有已报告的手动操作。

## 工具和 Gateway 网关接口

`teams_meetings` 智能体工具支持 `join`、`leave`、`status`、`transcript` 和 `speak`。Gateway 网关方法使用 `teamsmeetings.*` 前缀。节点命令为 `teamsmeetings.chrome`。

## 相关内容

- [会议插件概览](/plugins/meeting-plugins)
- [Microsoft Teams 频道](/zh-CN/channels/msteams)
