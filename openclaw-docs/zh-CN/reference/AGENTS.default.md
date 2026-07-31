---
read_when:
    - 启动新的 OpenClaw 智能体会话
    - 启用或审核默认 Skills
summary: 个人助理设置的默认 OpenClaw 智能体指令和 Skills 列表
title: 默认 AGENTS.md
x-i18n:
    generated_at: "2026-07-26T06:27:01Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 645342f8c6e2805135817cf4bbc2c8bd1d57066054ed671eda93876b2762ffb1
    source_path: reference/AGENTS.default.md
    workflow: 16
---

## 首次运行（推荐）

OpenClaw 智能体使用一个工作区目录。默认值：`~/.openclaw/workspace`（可通过 `agents.defaults.workspace` 配置，支持 `~`）。

1. 创建工作区：

```bash
mkdir -p ~/.openclaw/workspace
```

2. 将默认工作区模板复制到其中：

```bash
cp docs/reference/templates/AGENTS.md ~/.openclaw/workspace/AGENTS.md
cp docs/reference/templates/SOUL.md ~/.openclaw/workspace/SOUL.md
cp docs/reference/templates/TOOLS.md ~/.openclaw/workspace/TOOLS.md
```

3. 可选：使用此文件的个人助理 Skills 清单，而不是通用模板：

```bash
cp docs/reference/AGENTS.default.md ~/.openclaw/workspace/AGENTS.md
```

4. 可选：指向其他工作区：

```json5
{
  agents: { defaults: { workspace: "~/.openclaw/workspace" } },
}
```

## 默认安全设置

- 不要将目录内容或机密信息倾倒到聊天中。
- 除非明确要求，否则不要运行破坏性命令。
- 更改配置或调度器（crontab、systemd 单元、nginx 配置、shell rc 文件）之前，先检查现有状态，并默认保留或合并原有内容。
- 不要向外部消息平台发送不完整或流式回复（仅发送最终回复）。

## 现有解决方案预检

在提议或构建自定义系统、功能、工作流、工具、集成或自动化方案之前，先检查是否已有开源项目、维护中的库、现有 OpenClaw 插件或免费平台能够充分解决该问题。如果已有方案足够适用，应优先采用。仅当现有选项不适用、过于昂贵、无人维护、不安全、不合规，或用户明确要求自定义方案时，才自行构建。除非用户明确同意付费，否则避免推荐付费服务。保持此检查轻量化，将其作为预检关卡，而不是研究任务。

## 会话开始（必需）

- 回复前，阅读 `SOUL.md`、`USER.md`，以及 `memory/` 中今天和昨天的内容。
- 如果存在 `MEMORY.md`，请阅读它。

## 灵魂设定（必需）

- `SOUL.md` 定义身份、语气和边界。保持其内容为最新状态。
- 如果更改了 `SOUL.md`，请告知用户。
- 每个会话中的你都是一个全新实例；连续性保存在这些文件中。

## 共享空间（推荐）

- 你不代表用户发言；在群聊或公共频道中应谨慎。
- 不要分享私人数据、联系信息或内部笔记。

## 记忆系统（推荐）

- 每日日志：`memory/YYYY-MM-DD.md`（如有需要，创建 `memory/`）。
- 长期记忆：使用 `MEMORY.md` 记录持久有效的事实、偏好和决定。
- 小写的 `memory.md` 仅作为旧版修复输入；不要有意同时保留两个根目录文件。
- 会话开始时，阅读今天、昨天以及 `MEMORY.md`（如果存在）的内容。
- 写入记忆文件前，先读取文件；只写入具体更新，绝不写入空占位内容。
- 记录：决定、偏好、约束和未完成事项。
- 除非明确要求，否则避免记录机密信息。

## 工具和 Skills

- 工具位于 Skills 中；需要使用某项 Skill 时，请遵循其 `SKILL.md`。
- 将特定于环境的说明保存在 `TOOLS.md` 中（Skills 相关说明）。

## 备份提示（推荐）

将此工作区视为助理的记忆：把它创建为 git 仓库（最好是私有仓库），以便备份 `AGENTS.md` 和记忆文件。

```bash
cd ~/.openclaw/workspace
git init
git add AGENTS.md
git commit -m "Add workspace"
# 可选：添加私有远程仓库并推送
```

## OpenClaw 的作用

- 运行一个消息渠道 Gateway 网关（WhatsApp、Telegram、Discord、Signal、iMessage、Slack 等）以及一个嵌入式智能体，使助理能够读取和写入聊天、获取上下文，并通过宿主机器运行 Skills。
- macOS 应用管理权限（屏幕录制、通知、麦克风），并通过其内置二进制文件提供 `openclaw` CLI。
- 默认情况下，直接聊天会归入智能体的 `main` 会话；群组和频道/房间则各自使用独立的会话键。确切的键格式请参阅[频道路由](/zh-CN/channels/channel-routing)。Heartbeat 会让后台任务保持运行。

## 核心 Skills（在 Settings → Skills 中启用）

以下是个人助理工作区的示例清单；可根据你的设置换成合适的 Skills。

- **mcporter** - 用于管理外部 Skill 后端的工具服务器运行时/CLI。
- **Peekaboo** - 快速截取 macOS 屏幕截图，并可选择使用 AI 视觉分析。
- **camsnap** - 从 RTSP/ONVIF 安防摄像头捕获画面、视频片段或移动警报。
- **oracle** - 支持 OpenAI 的智能体 CLI，提供会话回放和浏览器控制。
- **eightctl** - 从终端控制你的睡眠。
- **imsg** - 发送、读取和流式传输 iMessage 与 SMS。
- **wacli** - WhatsApp CLI：同步、搜索和发送。
- **discord** - Discord 操作：表情回应、贴纸、投票。使用 `user:<id>` 或 `channel:<id>` 目标（不带前缀的纯数字 ID 含义不明确）。
- **gog** - Google 套件 CLI：Gmail、Calendar、Drive、Contacts。
- **spotify-player** - 终端 Spotify 客户端，用于搜索、加入队列和控制播放。
- **sag** - ElevenLabs 语音工具，采用 macOS 风格的 say 交互；默认将音频流式传输到扬声器。
- **Sonos CLI** - 通过脚本控制 Sonos 扬声器（发现/状态/播放/音量/分组）。
- **blucli** - 通过脚本播放、分组和自动控制 BluOS 播放器。
- **OpenHue CLI** - 控制 Philips Hue 灯光的场景和自动化。
- **OpenAI Whisper** - 本地语音转文字，用于快速听写和生成语音邮件转录文本。
- **Gemini CLI** - 从终端使用 Google Gemini 模型进行快速问答。
- **agent-tools** - 用于自动化和辅助脚本的实用工具包。

## 使用说明

- 编写脚本时优先使用 `openclaw` CLI；桌面应用负责处理权限。
- 从 Skills 选项卡运行安装；如果所需二进制文件已经存在，安装按钮会隐藏。
- 保持 Heartbeat 启用，以便助理安排提醒、监控收件箱并触发摄像头拍摄。
- Canvas UI 以全屏模式运行，并带有原生叠加层。避免将关键控件放在左上角、右上角或底部边缘；应添加明确的布局留白，而不是依赖安全区域边距。
- 进行浏览器驱动的验证时，请使用 `openclaw browser` CLI（内置 `browser` 插件），并配合由 OpenClaw 管理的 Chrome/Brave/Edge/Chromium 配置文件。
- 管理：`status`、`doctor [--deep]`、`start [--headless]`、`stop`、`tabs`、`tab [new|select|close]`、`open <url>`、`focus <id>`、`close <id>`。
- 检查：`screenshot [--full-page|--ref|--labels]`、`snapshot [--format ai|aria|--interactive|--efficient]`、`console`、`errors`、`requests`、`pdf`、`responsebody`。
- 操作：`navigate`、`click <ref>`、`type <ref> <text>`、`press`、`hover`、`drag`、`select`、`upload`、`download`、`fill`、`dialog`、`wait`、`evaluate --fn <js>`、`highlight`。操作需要来自 `snapshot` 的 `ref`（操作不接受 CSS 选择器）；需要采用 `document.querySelector` 风格的目标定位时，请使用 `evaluate`。
- 在任何检查命令中添加 `--json`，即可获得机器可读的输出。

## 相关内容

- [Agent 工作区](/zh-CN/concepts/agent-workspace)
- [智能体运行时](/zh-CN/concepts/agent)
- [频道路由](/zh-CN/channels/channel-routing)
