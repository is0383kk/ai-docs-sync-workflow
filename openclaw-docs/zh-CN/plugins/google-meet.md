---
read_when:
    - 你希望 OpenClaw 智能体加入 Google Meet 通话
    - 你希望 OpenClaw 智能体创建一个新的 Google Meet 通话
    - 你正在将 Chrome、Chrome 节点或 Twilio 配置为 Google Meet 传输方式
summary: Google Meet 插件：通过 Chrome 或 Twilio 加入明确指定的 Meet URL，并采用智能体语音回应默认设置
title: Google Meet 插件
x-i18n:
    generated_at: "2026-07-26T06:15:05Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 8a611e283fe900984a29b563969936a641c7af430b05933eb03b98dc93b5d0c8
    source_path: plugins/google-meet.md
    workflow: 16
---

`google-meet` 插件代表 OpenClaw 智能体加入明确指定的 Meet URL。其功能范围刻意保持精简：

- 它只加入 `https://meet.google.com/...` URL；绝不会使用自行发现的电话号码拨入会议。
- `googlemeet create` 可以通过 Google Meet API（或浏览器回退方案）创建新的 Meet URL，并默认加入该会议。
- 通过 Chrome 参会时使用已登录的 Chrome 配置文件，也可以选择在已配对节点上运行。通过 Twilio 参会时，会借助[语音通话插件](/zh-CN/plugins/voice-call)，使用电话号码加 PIN/DTMF 拨入；它无法直接拨打 Meet URL。
- `mode: "agent"`（默认）使用实时提供商转录参会者语音，将其路由到已配置的 OpenClaw 智能体，并使用常规 OpenClaw TTS 朗读回答。`mode: "bidi"` 允许实时语音模型直接回答。`mode: "transcribe"` 以仅观察模式加入，不进行语音回应。
- 插件加入通话时不会自动播放同意声明。
- CLI 命令为 `googlemeet`；`meet` 保留用于更广泛的智能体电话会议工作流。

## 快速开始

安装插件和本地音频依赖项，然后设置实时提供商密钥。OpenAI 是 `agent` 模式的默认转录提供商；Google Gemini Live 可用作 `bidi` 模式的语音提供商：

```bash
openclaw plugins install npm:@openclaw/google-meet
brew install blackhole-2ch sox
export OPENAI_API_KEY=sk-...
# 仅当 bidi 模式的 realtime.voiceProvider 为 "google" 时才需要
export GEMINI_API_KEY=...
```

`blackhole-2ch` 会安装供 Chrome 路由音频的 `BlackHole 2ch` 虚拟音频设备。Homebrew 安装程序要求重新启动后，macOS 才会显示该设备：

```bash
sudo reboot
```

重启后，验证这两个组件：

```bash
system_profiler SPAudioDataType | grep -i BlackHole
command -v sox
```

安装后，插件默认启用。仅在需要自定义时添加配置项：

```json5
{
  plugins: {
    entries: {
      "google-meet": {
        config: {},
      },
    },
  },
}
```

如果不希望启用该插件，请运行 `openclaw plugins disable google-meet`。

检查设置，然后加入会议：

```bash
openclaw googlemeet setup
openclaw googlemeet join https://meet.google.com/abc-defg-hij
```

`setup` 的输出可供智能体读取，并能感知模式和传输方式：它会报告 Chrome 配置文件、节点固定情况；对于通过 Chrome 实时加入的会话，还会报告 BlackHole/SoX 音频桥接和延迟开场白检查。仅观察模式的加入会跳过实时处理的前置条件：

```bash
openclaw googlemeet setup --transport chrome-node --mode transcribe
```

配置 Twilio 委派后，`setup` 还会报告 `voice-call`、Twilio 凭据和公共 webhook 暴露是否就绪。在智能体加入之前，应将任何 `ok: false` 检查视为相应传输方式/模式的阻塞项。使用 `--json` 获取机器可读输出，并使用 `--transport chrome|chrome-node|twilio` 提前预检特定传输方式：

```bash
openclaw googlemeet setup --transport twilio
```

也可以让智能体通过 `google_meet` 工具加入：

```json
{
  "action": "join",
  "url": "https://meet.google.com/abc-defg-hij",
  "transport": "chrome-node",
  "mode": "agent"
}
```

在非 macOS Gateway 网关主机上，`google_meet` 仍可用于工件、日历、设置、转录、Twilio 和 `chrome-node` 操作，但本地 Chrome 语音回应（使用 `mode: "agent"` 或 `"bidi"` 的 `transport: "chrome"`）会在到达音频桥接前被阻止，因为该路径目前依赖 macOS `BlackHole 2ch`。请改用 `mode: "transcribe"`、Twilio 拨入或 macOS `chrome-node` 主机。

### 创建会议

```bash
openclaw googlemeet create --transport chrome-node --mode agent
openclaw googlemeet create --no-join
```

`create` 有两条路径，结果的 `source` 字段会报告所用路径：

- **`api`**：配置 Google Meet OAuth 凭据时使用。行为确定，不依赖浏览器 UI 状态。
- **`browser`**：没有 OAuth 凭据时使用。OpenClaw 会在固定的 Chrome 节点上打开 `https://meet.google.com/new`，并等待 Google 重定向到实际的会议代码 URL；该节点上的 OpenClaw Chrome 配置文件必须已登录 Google。加入和创建操作都会先复用现有的 Meet 标签页（或正在进行的 `.../new` / Google 账号提示标签页），再打开新标签页；标签页匹配会忽略 `authuser` 等无关紧要的查询字符串。

`create` 默认加入会议，并返回 `joined: true` 和加入会话。传入 `--no-join`（CLI）或 `"join": false`（工具）可仅创建 URL。

对于通过 API 创建的会议室，请设置明确的访问策略，而不是继承 Google 账号的默认值：

```bash
openclaw googlemeet create --access-type OPEN --transport chrome-node --mode agent
```

| `--access-type` | 无需请求准入即可加入的人员                                              |
| --------------- | ------------------------------------------------------------------- |
| `OPEN`          | 任何拥有 Meet URL 的人员                                            |
| `TRUSTED`       | 主办方组织的受信任用户、受邀的外部用户和拨入用户 |
| `RESTRICTED`    | 仅受邀者                                                       |

此设置仅适用于通过 API 创建的会议室，因此必须配置 OAuth。如果在此选项推出之前已完成身份验证，请在 OAuth 同意屏幕中添加 `meetings.space.settings` 范围后，重新运行 `openclaw googlemeet auth login --json`。

如果浏览器回退方案遇到 Google 登录或 Meet 权限阻塞，工具会返回 `manualActionRequired: true`，其中包含 `manualActionReason`、`manualActionMessage` 和 `browser.nodeId`/`browser.targetId`/`browserUrl`。请报告该消息，并停止打开新的 Meet 标签页，直到操作员完成浏览器步骤。

### 以仅观察模式加入

将 `"mode": "transcribe"` 设置为跳过双工实时桥接（无需 BlackHole/SoX，也不进行语音回应）。转录模式下通过 Chrome 加入也会跳过 OpenClaw 的麦克风/摄像头权限授予和 Meet 的 **Use microphone** 流程；如果 Meet 显示音频选择中间页面，自动化会先尝试 **Continue without microphone**。托管的 Chrome 传输方式会在所有模式下尽力安装 Meet 字幕观察器，以便在不改变实时智能体咨询路径的情况下提供持久化笔记。`googlemeet status --json` 和 `googlemeet doctor` 会报告 `captioning`、`captionsEnabledAttempted`、`transcriptLines`、`lastCaptionAt`、`lastCaptionSpeaker`、`lastCaptionText` 和一个 `recentTranscript` 尾部记录。

要读取有界会话转录，请读取所跟踪的确切 Meet 标签页：

```bash
openclaw googlemeet transcript <session-id>
openclaw googlemeet transcript <session-id> --since <next-index> --json
```

观察器在 Meet 页面中最多保留 2,000 行已完成字幕。可见的渐进式文本会保留在状态健康尾部记录中，直到字幕行完成，因此保存 `nextIndex` 不会漏掉稍后的文本扩展；离开会议时，会先将可见行标记为完成，再创建快照。超过上限时，`droppedLines` 会报告从开头丢失的行数。有界的 `googlemeet transcript` 尾部记录仍只保留最近结束的四个会话，并随 Gateway 网关重置。另外，OpenClaw 会在整个会议期间将已完成的字幕行追加到共享状态数据库，并在离开时写入派生摘要。使用 [`openclaw transcripts`](/zh-CN/cli/transcripts) 检查或导出这些持久化笔记。

自动笔记默认启用。将 `transcripts.enabled: false` 设置为
可在全局禁用持久化笔记；显式 `transcribe` 模式仍只会公开
其有界实时尾部记录。Twilio 加入没有浏览器字幕流，因此
不会通过此路径捕获。

要执行“是/否”监听探测：

```bash
openclaw googlemeet test-listen <meet-url> --transport chrome-node
```

它会以转录模式加入，等待新的字幕/转录发生变化，并返回 `listenVerified`、`listenTimedOut`、手动操作字段和当前字幕健康状态。

### 实时会话健康状态

在语音回应会话期间，`google_meet` 状态会报告 Chrome/音频桥接健康状态：`inCall`、`manualActionRequired`、`providerConnected`、`realtimeReady`、`audioInputActive`、`audioOutputActive`、最后输入/输出时间戳、字节计数器和桥接关闭状态。托管的 Chrome 会话只会在健康状态报告 `inCall: true` 后朗读开场白/测试短语；否则会返回 `speechReady: false` 并阻止语音尝试，而不是静默地不执行任何操作。

本地 Chrome 通过已登录的 OpenClaw 浏览器配置文件加入，并且麦克风/扬声器路径需要 `BlackHole 2ch`。首次冒烟测试只需一个 BlackHole 设备，但可能产生回声；要获得干净的双工音频，请使用不同的虚拟设备或 Loopback 风格的音频图。

## 本地 Gateway 网关 + Parallels Chrome

如果只是要为 macOS 虚拟机提供 Chrome，则无需在虚拟机中运行完整的 Gateway 网关或配置模型 API key。在本地运行 Gateway 网关和智能体；在虚拟机中运行节点主机。

| 运行位置           | 运行内容                                                                                            |
| -------------------- | ----------------------------------------------------------------------------------------------- |
| Gateway 网关主机         | OpenClaw Gateway 网关、Agent 工作区、模型/API key、实时提供商、Google Meet 插件配置 |
| Parallels macOS 虚拟机   | OpenClaw CLI/节点主机、Chrome、SoX、BlackHole 2ch、已登录 Google 的 Chrome 配置文件        |
| 虚拟机中不需要 | Gateway 网关服务、智能体配置、模型提供商设置                                             |

安装虚拟机依赖项、重启并验证：

```bash
brew install blackhole-2ch sox
sudo reboot
system_profiler SPAudioDataType | grep -i BlackHole
command -v sox
```

在虚拟机中安装插件（默认启用），然后启动节点主机：

```bash
openclaw plugins install npm:@openclaw/google-meet
openclaw node run --host <gateway-host> --port 18789 --display-name parallels-macos
```

如果 `<gateway-host>` 是不使用 TLS 的 LAN IP，请针对该受信任的专用网络显式启用：

```bash
OPENCLAW_ALLOW_INSECURE_PRIVATE_WS=1 \
  openclaw node run --host <gateway-lan-ip> --port 18789 --display-name parallels-macos
```

安装为 LaunchAgent 时请使用相同的标志（它是进程环境变量，如果安装命令中存在，则会存储在 LaunchAgent 环境中，而不是 `openclaw.json` 设置）：

```bash
OPENCLAW_ALLOW_INSECURE_PRIVATE_WS=1 \
  openclaw node install --host <gateway-lan-ip> --port 18789 --display-name parallels-macos --force
openclaw node restart
```

从 Gateway 网关主机批准节点，然后确认它同时公布 `googlemeet.chrome` 和浏览器能力/`browser.proxy`：

```bash
openclaw devices list
openclaw devices approve <requestId>
openclaw nodes status
```

通过该节点路由 Meet：

```json5
{
  gateway: {
    nodes: {
      commands: { allow: ["googlemeet.chrome", "browser.proxy"] },
    },
  },
  plugins: {
    entries: {
      "google-meet": {
        enabled: true,
        config: {
          defaultTransport: "chrome-node",
          chrome: {
            guestName: "OpenClaw Agent",
            autoJoin: true,
            reuseExistingTab: true,
          },
          chromeNode: {
            node: "parallels-macos",
          },
        },
      },
    },
  },
}
```

现在可以从 Gateway 网关主机正常加入：

```bash
openclaw googlemeet join https://meet.google.com/abc-defg-hij
```

要运行单命令冒烟测试以创建或复用会话、朗读已知短语并打印会话健康状态：

```bash
openclaw googlemeet test-speech https://meet.google.com/abc-defg-hij
```

在实时加入期间，浏览器自动化会填写访客姓名，点击 Join/Ask to join，并在 Meet 首次运行时出现“Use microphone”提示时接受该提示（在仅观察模式加入和仅通过浏览器创建会议时，则选择“Continue without microphone”）。如果配置文件已退出登录、Meet 正在等待主持人准入、Chrome 需要麦克风/摄像头权限，或者 Meet 卡在未解决的提示上，结果会报告 `manualActionRequired: true`，并包含 `manualActionReason` 和 `manualActionMessage`。停止重试，报告该消息以及 `browserUrl`/`browserTitle`，并且仅在手动操作完成后重试。

如果省略 `chromeNode.node`，OpenClaw 仅会在恰好有一个已连接节点同时声明 `googlemeet.chrome` 和浏览器控制能力时自动选择；当连接了多个具备相应能力的节点时，请固定指定 `chromeNode.node`（节点 ID、显示名称或远程 IP）。

### 常见故障检查

| 症状                                                  | 修复方法                                                                                                                                                                                                                                                                                   |
| -------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `Configured Google Meet node ... is not usable: offline` | 已知固定指定的节点，但该节点不可用。报告设置阻碍；除非收到明确要求，否则不要静默回退到其他传输方式。                                                                                                                                                      |
| `No connected Google Meet-capable node`                  | 在虚拟机中安装 `npm:@openclaw/google-meet`，运行 `openclaw plugins enable browser`，启动 `openclaw node run`，然后批准配对。如果已明确禁用 Google Meet，也请将其启用。确认 `gateway.nodes.commands.allow` 包含 `googlemeet.chrome` 和 `browser.proxy`。 |
| `BlackHole 2ch audio device not found`                   | 在要检查的主机上安装 `blackhole-2ch`，然后重启。                                                                                                                                                                                                                         |
| `BlackHole 2ch audio device not found on the node`       | 在虚拟机中安装 `blackhole-2ch`，然后重启虚拟机。                                                                                                                                                                                                                                  |
| Chrome 已打开但无法加入                             | 在虚拟机中登录浏览器配置文件，或保持设置 `chrome.guestName`。访客自动加入通过节点浏览器代理使用 OpenClaw 浏览器自动化；将节点的 `browser.defaultProfile`（或命名的现有会话配置文件）指向所需的配置文件。                   |
| 重复的 Meet 标签页                                      | 保持 `chrome.reuseExistingTab: true`。在打开其他标签页之前，OpenClaw 会激活相同 URL 的现有标签页，并且创建操作会复用正在进行的 `.../new` 或 Google 账号提示标签页。                                                                                        |
| 无音频                                                 | 通过 OpenClaw 使用的虚拟音频路径路由 Meet 麦克风/扬声器；使用独立的虚拟设备或 Loopback 风格的路由，以获得清晰的双工音频。                                                                                                                                |

## 安装说明

Chrome 回传语音的默认方式使用两种 OpenClaw 不内置或再分发的外部工具；请通过 Homebrew 将它们安装为主机依赖项：

- `sox`：命令行音频工具。该插件为默认的 24 kHz PCM16 音频桥接发出明确的 CoreAudio 设备命令。
- `blackhole-2ch`：macOS 虚拟音频驱动程序，提供 Chrome/Meet 用于路由的 `BlackHole 2ch` 设备。

SoX 采用 `LGPL-2.0-only AND GPL-2.0-only` 许可；BlackHole 采用 GPL-3.0 许可。如果构建的安装程序或设备将 BlackHole 与 OpenClaw 捆绑，请审查 BlackHole 的上游许可，或从 Existential Audio 获取单独的许可证。

## 传输方式

| 传输方式     | 使用场景                                                                                     |
| ------------- | -------------------------------------------------------------------------------------------- |
| `chrome`      | Chrome/音频运行在 Gateway 网关主机上                                                        |
| `chrome-node` | Chrome/音频运行在已配对节点上（例如 Parallels macOS 虚拟机）                        |
| `twilio`      | 无法通过 Chrome 参与时，通过语音通话插件使用电话拨入回退方案 |

### Chrome

通过 OpenClaw 浏览器控制打开 Meet URL，并以已登录的 OpenClaw 浏览器配置文件加入。在 macOS 上，该插件会在启动前检查 `BlackHole 2ch`，并且在已配置时，于打开 Chrome 前运行音频桥接健康检查/启动命令。对于本地 Chrome，请使用 `browser.defaultProfile` 选择配置文件；`chrome.browserProfile` 则会传递给 `chrome-node` 主机。

```bash
openclaw googlemeet join https://meet.google.com/abc-defg-hij --transport chrome
openclaw googlemeet join https://meet.google.com/abc-defg-hij --transport chrome-node
```

Chrome 麦克风/扬声器音频通过本地 OpenClaw 音频桥接进行路由。如果未安装 `BlackHole 2ch`，加入操作会失败并显示设置错误，而不是在没有音频路径的情况下加入。

### Twilio

委托给[语音通话插件](/zh-CN/plugins/voice-call)的严格拨号方案。它不会解析 Meet 页面以查找电话号码；Google Meet 必须为会议提供电话拨入号码和 PIN。

请在 Gateway 网关主机上启用语音通话，而不是在 Chrome 节点上启用：

```json5
{
  plugins: {
    allow: ["google-meet", "voice-call", "google"],
    entries: {
      "google-meet": {
        enabled: true,
        config: {
          defaultTransport: "chrome-node",
          // 或者，如果 Twilio 应作为默认方式，则设置为 "twilio"
        },
      },
      "voice-call": {
        enabled: true,
        config: {
          provider: "twilio",
          inboundPolicy: "allowlist",
          realtime: {
            enabled: true,
            provider: "google",
            instructions: "以 OpenClaw 智能体身份加入此 Google Meet。保持简洁。",
            toolPolicy: "safe-read-only",
            providers: {
              google: {
                silenceDurationMs: 500,
                startSensitivity: "high",
              },
            },
          },
        },
      },
      google: {
        enabled: true,
      },
    },
  },
}
```

通过环境提供 Twilio 凭据，以避免将机密信息写入 `openclaw.json`：

```bash
export TWILIO_ACCOUNT_SID=AC...
export TWILIO_AUTH_TOKEN=...
export TWILIO_FROM_NUMBER=+15550001234
export GEMINI_API_KEY=...
```

如果 OpenAI 是实时语音提供商，请改用 `realtime.provider: "openai"` 和 `OPENAI_API_KEY`。

启用 `voice-call` 后，重启或重新加载 Gateway 网关；插件配置更改在重新加载前不会生效。验证：

```bash
openclaw config validate
openclaw plugins list | grep -E 'google-meet|voice-call'
openclaw googlemeet setup
```

连接好 Twilio 委托后，`googlemeet setup` 会包含 `twilio-voice-call-plugin`、`twilio-voice-call-credentials` 和 `twilio-voice-call-webhook` 检查。

```bash
openclaw googlemeet join https://meet.google.com/abc-defg-hij \
  --transport twilio \
  --dial-in-number +15551234567 \
  --pin 123456
```

如需自定义序列，请使用 `--dtmf-sequence`；可在开头使用 `w` 或逗号，以便在输入 PIN 前暂停：

```bash
openclaw googlemeet join https://meet.google.com/abc-defg-hij \
  --transport twilio \
  --dial-in-number +15551234567 \
  --dtmf-sequence ww123456#
```

## OAuth 和预检

创建 Meet 链接时 OAuth 是可选的，因为 `googlemeet create` 可以回退到浏览器自动化。请为通过官方 API 创建、解析会议空间或 Meet Media API 预检配置 OAuth。Chrome/Chrome-node 加入从不依赖 OAuth；无论哪种情况，它们都使用已登录的 Chrome 配置文件、BlackHole/SoX，并且（对于 `chrome-node`）还使用一个已连接节点。

### 创建 Google 凭据

在 Google Cloud Console 中：

<Steps>
<Step title="创建或选择项目">
</Step>
<Step title="启用 Google Meet REST API">
</Step>
<Step title="配置 OAuth 同意屏幕">
对于 Google Workspace 组织，Internal 最简单。External 适用于个人/测试设置；当应用处于 Testing 状态时，请将每个要授权该应用的 Google 账号添加为测试用户。
</Step>
<Step title="添加请求的权限范围">
- `https://www.googleapis.com/auth/meetings.space.created`
- `https://www.googleapis.com/auth/meetings.space.readonly`
- `https://www.googleapis.com/auth/meetings.space.settings`
- `https://www.googleapis.com/auth/meetings.conference.media.readonly`
- `https://www.googleapis.com/auth/calendar.events.readonly`（日历查找）
- `https://www.googleapis.com/auth/drive.meet.readonly`（转录/智能笔记文档正文导出）

</Step>
<Step title="创建 OAuth 客户端 ID">
应用类型选择 **Web application**。已获授权的重定向 URI：

```text
http://localhost:8085/oauth2callback
```

</Step>
<Step title="复制客户端 ID 和客户端密钥">
</Step>
</Steps>

`spaces.create` 需要 `meetings.space.created`。`meetings.space.readonly` 将 Meet URL/代码解析为会议空间。`meetings.space.settings` 允许 OpenClaw 在通过 API 创建会议室时传递 `SpaceConfig` 设置，例如 `accessType`。`meetings.conference.media.readonly` 用于 Meet Media API 预检和媒体工作；实际使用 Media API 时，Google 可能要求加入 Developer Preview。仅在使用 `--today`/`--event` 进行日历查找时才需要 `calendar.events.readonly`。仅在导出 `--include-doc-bodies` 时才需要 `drive.meet.readonly`。如果只需要基于浏览器的 Chrome 加入，请完全跳过 OAuth。

### 生成刷新令牌

配置 `oauth.clientId`，并可选择配置 `oauth.clientSecret`（或将其作为环境变量传递），然后运行：

```bash
openclaw googlemeet auth login --json
```

这会通过 `http://localhost:8085/oauth2callback` 上的 localhost 回调运行 PKCE 流程，并打印包含刷新令牌的 `oauth` 配置块。当浏览器无法访问本地回调时，请添加 `--manual` 以使用复制/粘贴流程：

```bash
OPENCLAW_GOOGLE_MEET_CLIENT_ID="your-client-id" \
OPENCLAW_GOOGLE_MEET_CLIENT_SECRET="your-client-secret" \
openclaw googlemeet auth login --json --manual
```

JSON 输出：

```json
{
  "oauth": {
    "clientId": "your-client-id",
    "clientSecret": "your-client-secret",
    "refreshToken": "refresh-token",
    "accessToken": "access-token",
    "expiresAt": 1770000000000
  },
  "scope": "..."
}
```

将 `oauth` 对象存储在插件配置下：

```json5
{
  plugins: {
    entries: {
      "google-meet": {
        enabled: true,
        config: {
          oauth: {
            clientId: "your-client-id",
            clientSecret: "your-client-secret",
            refreshToken: "refresh-token",
          },
        },
      },
    },
  },
}
```

如果不希望刷新令牌出现在配置中，请优先使用环境变量；系统会先解析配置，然后以环境变量作为回退。如果在支持会议创建、日历查找或文档正文导出之前已完成身份验证，请重新运行 `openclaw googlemeet auth login --json`，使刷新令牌涵盖当前的权限范围集。

### 使用 Doctor 验证 OAuth

```bash
openclaw googlemeet doctor --oauth --json
```

此检查会验证 OAuth 配置是否存在，以及刷新令牌能否签发访问令牌，而无需加载 Chrome 运行时或要求节点已连接。报告仅包含状态字段（`ok`、`configured`、`tokenSource`、`expiresAt`、检查消息），绝不会输出访问令牌、刷新令牌或客户端密钥。

| 检查项                | 含义                                                                          |
| -------------------- | -------------------------------------------------------------------------------- |
| `oauth-config`       | 存在 `oauth.clientId` 和 `oauth.refreshToken`，或已缓存的访问令牌 |
| `oauth-token`        | 已缓存的访问令牌仍然有效，或刷新令牌已签发新令牌    |
| `meet-spaces-get`    | 可选的 `--meeting` 检查解析到了现有 Meet 空间                       |
| `meet-spaces-create` | 可选的 `--create-space` 检查创建了新的 Meet 空间                         |

使用会产生副作用的创建检查来验证 Meet API 已启用且具有 `spaces.create` 权限范围：

```bash
openclaw googlemeet doctor --oauth --create-space --json
```

验证对现有空间的读取权限：

```bash
openclaw googlemeet doctor --oauth --meeting https://meet.google.com/abc-defg-hij --json
openclaw googlemeet resolve-space --meeting https://meet.google.com/abc-defg-hij
```

这些检查返回 `403` 通常表示 Meet REST API 已禁用、刷新令牌缺少所需权限范围，或 Google 账号无法访问该空间。刷新令牌错误表示需要重新运行 `openclaw googlemeet auth login --json`，并存储新的 `oauth` 块。

浏览器回退不需要 OAuth；其中的 Google 身份验证来自所选节点上已登录的 Chrome 配置文件，而非 OpenClaw 配置。

以下环境变量可用作回退：

- `OPENCLAW_GOOGLE_MEET_CLIENT_ID` 或 `GOOGLE_MEET_CLIENT_ID`
- `OPENCLAW_GOOGLE_MEET_CLIENT_SECRET` 或 `GOOGLE_MEET_CLIENT_SECRET`
- `OPENCLAW_GOOGLE_MEET_REFRESH_TOKEN` 或 `GOOGLE_MEET_REFRESH_TOKEN`
- `OPENCLAW_GOOGLE_MEET_ACCESS_TOKEN` 或 `GOOGLE_MEET_ACCESS_TOKEN`
- `OPENCLAW_GOOGLE_MEET_ACCESS_TOKEN_EXPIRES_AT` 或 `GOOGLE_MEET_ACCESS_TOKEN_EXPIRES_AT`
- `OPENCLAW_GOOGLE_MEET_DEFAULT_MEETING` 或 `GOOGLE_MEET_DEFAULT_MEETING`
- `OPENCLAW_GOOGLE_MEET_PREVIEW_ACK` 或 `GOOGLE_MEET_PREVIEW_ACK`

### 解析、预检和读取工件

```bash
openclaw googlemeet resolve-space --meeting https://meet.google.com/abc-defg-hij
openclaw googlemeet preflight --meeting https://meet.google.com/abc-defg-hij
```

Meet 创建会议记录后：

```bash
openclaw googlemeet artifacts --meeting https://meet.google.com/abc-defg-hij
openclaw googlemeet attendance --meeting https://meet.google.com/abc-defg-hij
openclaw googlemeet export --meeting https://meet.google.com/abc-defg-hij --output ./meet-export
```

使用 `--meeting` 时，`artifacts` 和 `attendance` 默认使用最新的会议记录；传入 `--all-conference-records` 可处理所有保留的记录。

日历查找会先从 Google Calendar 解析会议 URL，然后再读取工件（需要刷新令牌包含 Calendar 事件只读权限范围）：

```bash
openclaw googlemeet latest --today
openclaw googlemeet calendar-events --today --json
openclaw googlemeet artifacts --event "Weekly sync"
openclaw googlemeet attendance --today --format csv --output attendance.csv
```

`--today` 会在今天的 `primary` 日历中搜索包含 Meet 链接的事件；`--event <query>` 搜索匹配的事件文本；`--calendar <id>` 指定非主日历。`calendar-events` 会预览匹配的事件，并标记 `latest`/`artifacts`/`attendance`/`export` 将选择哪一个。

如果已知会议记录 ID，可直接指定：

```bash
openclaw googlemeet latest --meeting https://meet.google.com/abc-defg-hij
openclaw googlemeet artifacts --conference-record conferenceRecords/abc123 --json
openclaw googlemeet attendance --conference-record conferenceRecords/abc123 --json
```

关闭通过 API 创建的空间：

```bash
openclaw googlemeet end-active-conference https://meet.google.com/abc-defg-hij
```

此命令会调用 `spaces.endActiveConference`，并且对于已授权账号可管理的空间，需要具有 `meetings.space.created` 权限范围的 OAuth。它接受 Meet URL、会议代码或 `spaces/{id}`，并先将其解析为 API 空间资源。这与 `googlemeet leave` 不同：`leave` 会停止 OpenClaw 的本地/会话参与；`end-active-conference` 会请求 Google Meet 结束该空间的活动会议。

写入易读的报告：

```bash
openclaw googlemeet artifacts --conference-record conferenceRecords/abc123 \
  --format markdown --output meet-artifacts.md
openclaw googlemeet attendance --conference-record conferenceRecords/abc123 \
  --format csv --output meet-attendance.csv
openclaw googlemeet export --conference-record conferenceRecords/abc123 \
  --include-doc-bodies --zip --output meet-export
openclaw googlemeet export --conference-record conferenceRecords/abc123 \
  --include-doc-bodies --dry-run
```

`artifacts` 会返回会议记录元数据，以及 Google 提供的参与者、录制内容、转写文本、结构化转写条目和智能笔记资源元数据。`--no-transcript-entries` 会跳过大型会议的条目查找。`attendance` 会将参与者展开为参与者会话行，其中包含首次/最后出现时间、会话总时长、迟到/提前离开标志，并按已登录用户或显示名称合并重复的参与者资源；`--no-merge-duplicates` 会将原始资源分别保留，`--late-after-minutes`/`--early-before-minutes` 用于调整阈值。

`export` 会写入一个包含 `summary.md`、`attendance.csv`、`transcript.md`、`artifacts.json`、`attendance.json` 和 `manifest.json` 的文件夹。`manifest.json` 会记录所选输入、导出选项、会议记录、输出文件、计数、令牌来源、使用的任何 Calendar 事件以及部分检索警告。`--zip` 还会在文件夹旁写入一个可移植归档。`--include-doc-bodies` 会通过 Drive `files.export` 导出已链接的转写文本/智能笔记 Google Docs 文本（需要 Drive Meet 只读权限范围）；如果未启用，导出仅包含 Meet 元数据和结构化转写条目。如果部分工件失败（智能笔记列表、转写条目或文档正文错误），警告会保留在摘要/清单中，而不会导致整个导出失败。`--dry-run` 会获取相同的数据并输出清单 JSON，而不创建文件夹或 ZIP。

智能体通过 `google_meet` 工具使用相同的操作（`export`、带 `accessType` 的 `create`、`end_active_conference`、`test_listen`）；参见[工具](#tool)。

### 实时冒烟测试

```bash
OPENCLAW_LIVE_TEST=1 \
OPENCLAW_GOOGLE_MEET_LIVE_MEETING=https://meet.google.com/abc-defg-hij \
pnpm test:live -- extensions/google-meet/google-meet.live.test.ts
```

```bash
openclaw googlemeet setup --transport chrome-node --mode transcribe
openclaw googlemeet test-listen https://meet.google.com/abc-defg-hij --transport chrome-node --timeout-ms 30000
```

| 变量                                                                                                                  | 用途                                                                |
| ------------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------- |
| `OPENCLAW_LIVE_TEST=1`                                                                                                    | 启用受保护的实时测试                                             |
| `OPENCLAW_GOOGLE_MEET_LIVE_MEETING`                                                                                       | 保留的 Meet URL、代码或 `spaces/{id}`                              |
| `OPENCLAW_GOOGLE_MEET_CLIENT_ID` / `GOOGLE_MEET_CLIENT_ID`                                                                | OAuth 客户端 ID                                                        |
| `OPENCLAW_GOOGLE_MEET_REFRESH_TOKEN` / `GOOGLE_MEET_REFRESH_TOKEN`                                                        | 刷新令牌                                                          |
| `OPENCLAW_GOOGLE_MEET_CLIENT_SECRET`、`OPENCLAW_GOOGLE_MEET_ACCESS_TOKEN`、`OPENCLAW_GOOGLE_MEET_ACCESS_TOKEN_EXPIRES_AT` | 可选；不带 `OPENCLAW_` 前缀的相同回退名称也可用 |

基础工件/出席情况冒烟测试需要 `meetings.space.readonly` 和 `meetings.conference.media.readonly`。日历查找需要 `calendar.events.readonly`。Drive 文档正文导出需要 `drive.meet.readonly`。

### 创建示例

```bash
openclaw googlemeet create
```

输出新会议 URI、来源和加入会话。使用 OAuth 时会使用 Meet API；未使用 OAuth 时，则使用固定 Chrome 节点的已登录配置文件。浏览器回退 JSON：

```json
{
  "source": "browser",
  "meetingUri": "https://meet.google.com/abc-defg-hij",
  "joined": true,
  "browser": {
    "nodeId": "ba0f4e4bc...",
    "targetId": "tab-1"
  },
  "join": {
    "session": {
      "id": "meet_...",
      "url": "https://meet.google.com/abc-defg-hij"
    }
  }
}
```

如果浏览器回退首先遇到 Google 登录或 Meet 权限阻止页面，`google_meet` 会返回结构化详细信息，而非纯字符串：

```json
{
  "source": "browser",
  "error": "google-login-required: 登录 OpenClaw 浏览器配置文件中的 Google 账号，然后重试创建会议。",
  "manualActionRequired": true,
  "manualActionReason": "google-login-required",
  "manualActionMessage": "登录 OpenClaw 浏览器配置文件中的 Google 账号，然后重试创建会议。",
  "browser": {
    "nodeId": "ba0f4e4bc...",
    "targetId": "tab-1",
    "browserUrl": "https://accounts.google.com/signin",
    "browserTitle": "登录 - Google 账号"
  }
}
```

API 创建 JSON：

```json
{
  "source": "api",
  "meetingUri": "https://meet.google.com/abc-defg-hij",
  "joined": true,
  "space": {
    "name": "spaces/abc-defg-hij",
    "meetingCode": "abc-defg-hij",
    "meetingUri": "https://meet.google.com/abc-defg-hij"
  },
  "join": {
    "session": {
      "id": "meet_...",
      "url": "https://meet.google.com/abc-defg-hij"
    }
  }
}
```

创建后默认加入，但 Chrome/Chrome 节点仍需要已登录的 Google 配置文件才能通过浏览器加入；如果已退出登录，OpenClaw 会报告 `manualActionRequired: true` 或浏览器回退错误，并要求操作员完成 Google 登录后再重试。

仅在确认你的 Cloud 项目、OAuth 主体和会议参与者均已加入 Google Workspace Developer Preview Program for Meet media APIs 后，才设置 `preview.enrollmentAcknowledged: true`。

## 配置

常规 Chrome 智能体路径只需要启用插件、BlackHole、SoX、实时提供商密钥，以及已配置的 OpenClaw TTS 提供商：

```json5
{
  plugins: {
    entries: {
      "google-meet": {
        enabled: true,
        config: {},
      },
    },
  },
}
```

### 默认值

| 键                               | 默认值                                  | 说明                                                                                                                                                                                                             |
| --------------------------------- | ---------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `defaultTransport`                | `"chrome"`                               |                                                                                                                                                                                                                   |
| `defaultMode`                     | `"agent"`                                | 接受 `"realtime"` 作为 `"agent"` 的旧版别名；新调用方应使用 `"agent"`                                                                                                                        |
| `chromeNode.node`                 | 未设置                                    | `chrome-node` 的节点 ID/名称/IP；当可能连接多个具有相应能力的节点时必填                                                                                                                      |
| `chrome.launch`                   | `true`                                   | 启动 Chrome 以加入；仅在复用已打开的会话时设置 `false`                                                                                                                                 |
| `chrome.audioBackend`             | `"blackhole-2ch"`                        |                                                                                                                                                                                                                   |
| `chrome.guestName`                | `"OpenClaw Agent"`                       | 显示在已退出登录的 Meet 访客界面上                                                                                                                                                                         |
| `chrome.autoJoin`                 | `true`                                   | 在 `chrome-node` 上尽力填写访客姓名并点击 Join Now                                                                                                                                                   |
| `chrome.reuseExistingTab`         | `true`                                   | 激活现有的 Meet 标签页，而不是打开重复标签页                                                                                                                                                      |
| `chrome.waitForInCallMs`          | `20000`                                  | 等待 Meet 标签页报告已进入通话，然后再触发回话介绍                                                                                                                                          |
| `chrome.audioFormat`              | `"pcm16-24khz"`                          | 命令对音频格式；`"g711-ulaw-8khz"` 仅用于输出电话音频的旧版/自定义命令对                                                                                                   |
| `chrome.audioBufferBytes`         | `4096`                                   | 生成的命令对音频命令使用的 SoX 处理缓冲区（为 SoX 默认 8192 字节缓冲区的一半，以降低管道延迟）；值会被限制为至少 17 字节                                         |
| `chrome.audioInputCommand`        | 生成的 SoX 命令                    | 从 CoreAudio `BlackHole 2ch` 读取，并以 `chrome.audioFormat` 格式写入音频                                                                                                                                        |
| `chrome.audioOutputCommand`       | 生成的 SoX 命令                    | 读取 `chrome.audioFormat` 格式的音频，并写入 CoreAudio `BlackHole 2ch`                                                                                                                                          |
| `chrome.bargeInInputCommand`      | 未设置                                    | 可选的本地麦克风命令，用于写入有符号 16 位小端序单声道 PCM，以便在助手播放期间检测人工插话；适用于由 Gateway 网关托管的命令对桥接                          |
| `chrome.bargeInRmsThreshold`      | `650`                                    | 视为人工打断的 RMS 电平                                                                                                                                                                           |
| `chrome.bargeInPeakThreshold`     | `2500`                                   | 视为人工打断的峰值电平                                                                                                                                                                          |
| `chrome.bargeInCooldownMs`        | `900`                                    | 重复清除打断状态之间的最短延迟                                                                                                                                                                |
| `mode`（按请求）              | `"agent"`                                | 回话模式；参见 [Agent 和双向模式](#agent-and-bidi-modes)表                                                                                                                                       |
| `realtime.provider`               | `"openai"`                               | 当下方作用域字段未设置时使用的兼容性回退值                                                                                                                                                |
| `realtime.transcriptionProvider`  | `"openai"`                               | `agent` 模式用于实时转录的提供商 ID                                                                                                                                                       |
| `realtime.voiceProvider`          | 未设置                                    | `bidi` 模式用于直接实时语音的提供商 ID；设为 `"google"` 可使用 Gemini Live，同时让 Agent 模式转录继续使用 OpenAI。与 `realtime.model` 配合使用以选择特定的 Gemini Live 模型。 |
| `realtime.toolPolicy`             | `"safe-read-only"`                       | 参见 [Agent 和双向模式](#agent-and-bidi-modes)                                                                                                                                                                 |
| `realtime.instructions`           | 简短的语音回复指令          | 指示模型简短作答，并使用 `openclaw_agent_consult` 提供更深入的回答                                                                                                                              |
| `realtime.introMessage`           | `"Say exactly: I'm here and listening."` | 实时桥接连接时播报一次；设为 `""` 可静默加入                                                                                                                                       |
| `realtime.agentId`                | `"main"`                                 | `openclaw_agent_consult` 使用的 OpenClaw 智能体 ID                                                                                                                                                               |
| `voiceCall.enabled`               | `true`                                   | 将 Twilio PSTN 通话、DTMF 和开场问候委托给语音通话插件                                                                                                                                 |
| `voiceCall.dtmfDelayMs`           | `12000`                                  | 通过 Twilio 播放由 PIN 派生的 DTMF 序列前的等待时间                                                                                                                                               |
| `voiceCall.postDtmfSpeechDelayMs` | `5000`                                   | 语音通话启动 Twilio 通话线路后，请求实时开场问候前的延迟                                                                                                                        |

`chrome.audioBridgeCommand` 和 `chrome.audioBridgeHealthCommand` 允许外部桥接接管整个本地音频路径，而不使用 `chrome.audioInputCommand`/`chrome.audioOutputCommand`；有关哪些模式可以使用它们的限制，请参见[说明](#notes)。

存在针对旧版 `realtime.provider: "google"` 结构的 `openclaw doctor --fix` 迁移：当 `realtime.voiceProvider: "google"` 和 `realtime.transcriptionProvider: "openai"` 尚未设置时，它会将该意图迁移到这两个字段。

### 可选覆盖项

```json5
{
  defaults: {
    meeting: "https://meet.google.com/abc-defg-hij",
  },
  browser: {
    defaultProfile: "openclaw",
  },
  chrome: {
    guestName: "OpenClaw Agent",
    waitForInCallMs: 30000,
    bargeInInputCommand: [
      "sox",
      "-q",
      "-t",
      "coreaudio",
      "External Microphone",
      "-r",
      "24000",
      "-c",
      "1",
      "-b",
      "16",
      "-e",
      "signed-integer",
      "-t",
      "raw",
      "-",
    ],
  },
  chromeNode: {
    node: "parallels-macos",
  },
  defaultMode: "agent",
  realtime: {
    provider: "openai",
    transcriptionProvider: "openai",
    voiceProvider: "google",
    model: "gemini-3.1-flash-live-preview",
    agentId: "jay",
    toolPolicy: "owner",
    introMessage: "Say exactly: I'm here.",
    providers: {
      google: {
        speakerVoice: "Kore",
      },
    },
  },
}
```

使用 ElevenLabs 进行 Agent 模式的聆听和语音输出：

```json5
{
  tts: {
    provider: "elevenlabs",
    providers: {
      elevenlabs: {
        modelId: "eleven_v3",
        speakerVoiceId: "pMsXgVXv3BLzUgSXRplE",
      },
    },
  },
  plugins: {
    entries: {
      "google-meet": {
        config: {
          realtime: {
            transcriptionProvider: "elevenlabs",
            providers: {
              elevenlabs: {
                modelId: "scribe_v2_realtime",
                audioFormat: "ulaw_8000",
                sampleRate: 8000,
                commitStrategy: "vad",
              },
            },
          },
        },
      },
    },
  },
}
```

Meet 的持久语音来自 `tts.providers.elevenlabs.speakerVoiceId`。启用 TTS 模型覆盖后，Agent 回复也可以使用按回复指定的 `[[tts:speakerVoiceId=... model=eleven_v3]]` 指令，但对于会议而言，配置是确定性的默认设置。加入时，日志会显示 `transcriptionProvider=elevenlabs`；每次语音回复都会记录 `provider=elevenlabs model=eleven_v3 speakerVoiceId=<voiceId>`。

仅限 Twilio 的配置：

```json5
{
  defaultTransport: "twilio",
  twilio: {
    defaultDialInNumber: "+15551234567",
    defaultPin: "123456",
  },
  voiceCall: {
    gatewayUrl: "ws://127.0.0.1:18789",
  },
}
```

使用 `voiceCall.enabled: true`（默认值）和 Twilio 传输时，语音通话会在打开实时媒体流之前发送 DTMF 序列，然后将已保存的开场文本用作初始实时问候。如果未启用 `voice-call`，Google Meet 仍可验证并记录拨号方案，但无法发起 Twilio 通话。

将 `voiceCall.gatewayUrl` 保持未设置即可使用本地可信的 Gateway 网关运行时，该运行时会在整个调用期间保留发起调用的智能体。已配置的 Gateway 网关 URL 仍是显式的 WebSocket 目标，且无法验证插件来源；非默认智能体加入时会以失败关闭，而不会静默改用其他智能体。当需要按智能体路由时，请在同一 Gateway 网关进程中运行 Google Meet 和 Voice Call。

## 工具

智能体使用 `google_meet` 工具：

```json
{
  "action": "join",
  "url": "https://meet.google.com/abc-defg-hij",
  "transport": "chrome-node",
  "mode": "agent"
}
```

| `action`                | 用途                                                                                           |
| ----------------------- | ------------------------------------------------------------------------------------------------- |
| `join`                  | 加入显式指定的 Meet URL                                                                         |
| `create`                | 创建空间（默认同时加入）；支持 `accessType`/`entryPointAccess`                    |
| `status`                | 列出活跃会话，或通过 `sessionId` 检查某个会话                                               |
| `setup_status`          | 运行与 `googlemeet setup` 相同的检查                                                         |
| `resolve_space`         | 通过 `spaces.get` 解析 URL、代码或 `spaces/{id}`                                                 |
| `preflight`             | 验证 OAuth 和会议解析的先决条件                                                 |
| `latest`                | 查找某场会议的最新会议记录                                                   |
| `calendar_events`       | 预览包含 Meet 链接的日历事件                                                           |
| `artifacts`             | 列出会议记录以及参与者、录制、文字记录和智能笔记元数据                  |
| `attendance`            | 列出参与者和参与者会话                                                        |
| `export`                | 写入工件、出席记录、文字记录和清单包；设置 `"dryRun": true` 可仅生成清单 |
| `recover_current_tab`   | 聚焦或检查现有 Meet 标签页，而不打开新标签页                                      |
| `transcript`            | 读取有界字幕文字记录；`sinceIndex` 从上一个 `nextIndex` 继续           |
| `leave`                 | 结束会话（Chrome 点击 Leave；仅关闭由其打开的标签页；Twilio 挂断）                  |
| `end_active_conference` | 结束 API 管理空间中活跃的 Google Meet 会议                                    |
| `speak`                 | 在给定 `sessionId` 和 `message` 后，让实时智能体立即说话                        |
| `test_speech`           | 创建或复用会话，触发已知短语，并返回 Chrome 健康状态                              |
| `test_listen`           | 创建或复用仅观察会话，并等待字幕或文字记录发生变化                        |

`test_speech` 始终强制使用 `mode: "agent"` 或 `"bidi"`；如果要求在 `mode: "transcribe"` 中运行，则会失败，因为仅观察会话无法发出语音。`speechOutputVerified` 要求同时存在新的实时输出字节，以及该输出期间通过桥接器麦克风捕获路径返回的新的非静音音频。复用会话中的旧输出或回环信号不计入验证，仅有接收端字节增长也不再表示语音已通过验证。

对于 Chrome 传输，`leave` 会在点击 Meet 的 Leave 通话按钮后，让复用的用户自有标签页保持打开。由 OpenClaw 打开的标签页会在离开后关闭。

当 Chrome 在 Gateway 网关主机上运行时使用 `transport: "chrome"`，在已配对节点上运行时使用 `transport: "chrome-node"`。在这两种情况下，模型提供商和 `openclaw_agent_consult` 都在 Gateway 网关主机上运行，因此模型凭据会保留在那里。智能体模式日志会在桥接器启动时包含解析后的转录提供商和模型，并在每次合成回复后包含 TTS 提供商、模型、语音、输出格式和采样率。原始 `mode: "realtime"` 仍作为 `mode: "agent"` 的旧版兼容别名被接受，但工具的 `mode` 枚举中不再公布该值。

使用 API 支持的房间和显式访问策略的 `create`：

```json
{
  "action": "create",
  "transport": "chrome-node",
  "mode": "agent",
  "accessType": "OPEN"
}
```

结束已知房间中的活跃会议：

```json
{
  "action": "end_active_conference",
  "meeting": "https://meet.google.com/abc-defg-hij"
}
```

在声称会议可用之前，先执行收听验证：

```json
{
  "action": "test_listen",
  "url": "https://meet.google.com/abc-defg-hij",
  "transport": "chrome-node",
  "timeoutMs": 30000
}
```

按需说话：

```json
{
  "action": "speak",
  "sessionId": "meet_...",
  "message": "准确地说：我已加入，正在聆听。"
}
```

`status` 会在可用时包含 Chrome 健康状态：

| 字段                                                                 | 含义                                                                                                                |
| --------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------- |
| `inCall`                                                              | Chrome 看起来已进入 Meet 通话                                                                              |
| `micMuted`                                                            | 尽力检测的 Meet 麦克风状态                                                                                      |
| `manualActionRequired` / `manualActionReason` / `manualActionMessage` | 在语音能够正常工作之前，浏览器配置文件需要手动登录、Meet 主持人准入、权限授权或浏览器控制修复 |
| `speechReady` / `speechBlockedReason` / `speechBlockedMessage`        | 当前是否允许受管理的 Chrome 语音；`speechReady: false` 表示 OpenClaw 未发送开场或测试短语   |
| `providerConnected` / `realtimeReady`                                 | 实时语音桥接器状态                                                                                            |
| `lastInputAt` / `lastOutputAt`                                        | 最近从桥接器接收或向其发送的音频                                                                                |
| `audioOutputRouted` / `audioOutputDeviceLabel`                        | Meet 标签页的媒体输出是否已主动路由至桥接器的 BlackHole 设备                               |
| `lastOutputLoopbackAt` / `outputLoopbackSignalBytes`                  | 波形指纹已在 BlackHole 麦克风捕获路径上建立关联的新输出                        |
| `lastOutputLoopbackCorrelation`                                       | 将捕获的信号与当前助手输出生成关联起来的相关性分数                                 |
| `outputGeneration` / `verifiedOutputGeneration`                       | 单调递增 ID；二者相等表示通过回环验证的是当前输出，而不是较早的话语                |
| `lastOutputLoopbackRms` / `lastOutputLoopbackPeak`                    | 最近一次通过验证的回环捕获音频块的音频能量诊断                                                |
| `lastSuppressedInputAt` / `suppressedInputBytes`                      | 助手播放音频时忽略回环输入                                                              |

## 智能体和双向模式

| 模式    | 由谁决定回答        | 语音输出路径                     | 适用场景                                              |
| ------- | ----------------------------- | -------------------------------------- | ----------------------------------------------------- |
| `agent` | 已配置的 OpenClaw 智能体 | 常规 OpenClaw TTS 运行时            | 需要“我的智能体正在会议中”的行为        |
| `bidi`  | 实时语音模型      | 实时语音提供商的音频响应 | 需要最低延迟的对话式语音循环 |

`agent` 模式：实时转录提供商收听会议音频，参与者的最终文字记录经由已配置的 OpenClaw 智能体进行路由，回答则通过常规 OpenClaw TTS 播放。相邻的最终文字记录片段会在咨询前合并，避免一次口语轮次产生多个过时的局部回答；当队列中的助手音频仍在播放时，实时输入会被抑制，并且在咨询前会忽略最近类似助手语音的文字记录回声，从而避免 BlackHole 回环导致智能体回答自己的语音。

`bidi` 模式：实时语音模型直接回答，并可调用 `openclaw_agent_consult` 进行更深入的推理、获取当前信息或使用常规 OpenClaw 工具。咨询工具会在后台使用最近的会议文字记录上下文运行常规 OpenClaw 智能体，并返回简洁的口语回答；在 `agent` 模式下，OpenClaw 会将该回答直接发送至 TTS；在 `bidi` 模式下，实时语音模型可以将其说出。它与 Voice Call 使用相同的共享咨询机制。

默认情况下，咨询针对 `main` 智能体运行；设置 `realtime.agentId` 可将 Meet 通道指向专用的智能体工作区、模型默认值、工具策略、记忆和会话历史。智能体模式咨询使用按会议区分的 `agent:<id>:subagent:google-meet:<session>` 会话键，因此后续问题可以保留会议上下文，同时继承常规智能体策略。当智能体在智能体模式下调用 `google_meet` 时，顾问会话会先派生调用方当前的文字记录，然后再回答参与者的发言；Meet 会话保持独立，因此会议中的后续提问不会直接修改调用方的文字记录。

`realtime.toolPolicy` 控制咨询运行：

| 策略           | 行为                                                                                                                         |
| ---------------- | -------------------------------------------------------------------------------------------------------------------------------- |
| `safe-read-only` | 提供咨询工具；将常规智能体限制为使用 `read`、`web_search`、`web_fetch`、`x_search`、`memory_search`、`memory_get` |
| `owner`          | 提供咨询工具；允许常规智能体使用其正常的工具策略                                                        |
| `none`           | 不向实时语音模型提供咨询工具                                                                       |

咨询会话键按 Meet 会话划分作用域，因此在同一场会议期间，后续咨询调用会复用先前的咨询上下文。

在 Chrome 完全加入后，强制执行口头就绪检查：

```bash
openclaw googlemeet speak meet_... "准确地说：我已加入，正在聆听。"
```

完整的加入并说话冒烟测试：

```bash
openclaw googlemeet test-speech https://meet.google.com/abc-defg-hij \
  --transport chrome-node \
  --message "准确地说：我已加入，正在聆听。"
```

## 实时测试检查清单

将会议交给无人值守的智能体之前：

```bash
openclaw googlemeet setup
openclaw nodes status
openclaw googlemeet test-speech https://meet.google.com/abc-defg-hij \
  --transport chrome-node \
  --message "准确地说：Google Meet 语音测试完成。"
```

预期的 Chrome 节点状态：

- `googlemeet setup` 全部显示为绿色，并且当 Chrome 节点是默认传输方式或固定了某个节点时，其中会包含 `chrome-node-connected`。
- `nodes status` 显示所选节点已连接，并同时公布 `googlemeet.chrome` 和 `browser.proxy`。
- Meet 标签页成功加入，并且 `test-speech` 返回包含 `inCall: true` 的 Chrome 健康状态。

对于 Parallels macOS 虚拟机等远程 Chrome 主机，更新 Gateway 网关或虚拟机后最简短且安全的检查方法如下：

```bash
openclaw googlemeet setup
openclaw nodes status --connected
openclaw nodes invoke \
  --node parallels-macos \
  --command googlemeet.chrome \
  --params '{"action":"setup"}'
```

这可以证明 Gateway 网关插件已加载、虚拟机节点已使用当前令牌连接，并且在智能体打开真实会议标签页之前，Meet 音频桥接已可用。

对于 Twilio 冒烟测试，请使用提供电话拨入信息的会议：

```bash
openclaw googlemeet setup
openclaw googlemeet join https://meet.google.com/abc-defg-hij \
  --transport twilio \
  --dial-in-number +15551234567 \
  --pin 123456
```

预期的 Twilio 状态：

- `googlemeet setup` 包含显示为绿色的 `twilio-voice-call-plugin`、`twilio-voice-call-credentials` 和 `twilio-voice-call-webhook` 检查。
- Gateway 网关重新加载后，CLI 中可以使用 `voicecall`。
- 返回的会话具有 `transport: "twilio"` 和一个 `twilio.voiceCallId`。
- `openclaw logs --follow` 显示先提供 DTMF TwiML，再提供实时 TwiML，随后建立实时桥接并将初始问候语加入队列。
- `googlemeet leave <sessionId>` 挂断委托的语音通话。

## 故障排查

### 智能体看不到 Google Meet 工具

确认插件已启用并重新加载 Gateway 网关；正在运行的智能体只能看到当前 Gateway 网关进程注册的插件工具：

```bash
openclaw plugins list | grep google-meet
openclaw googlemeet setup
```

在非 macOS 的 Gateway 网关主机上，`google_meet` 仍然可见，但本地 Chrome 回话操作会在到达音频桥接之前被阻止。请使用 `mode: "transcribe"`、Twilio 拨入，或 macOS `chrome-node` 主机，而不要使用默认的本地 Chrome 智能体路径。

### 没有已连接且支持 Google Meet 的节点

在节点主机上：

```bash
openclaw plugins install npm:@openclaw/google-meet
openclaw plugins enable browser
OPENCLAW_ALLOW_INSECURE_PRIVATE_WS=1 \
  openclaw node run --host <gateway-lan-ip> --port 18789 --display-name parallels-macos
```

在 Gateway 网关主机上：

```bash
openclaw devices list
openclaw devices approve <requestId>
openclaw nodes status
```

节点必须已连接并列出 `googlemeet.chrome` 和 `browser.proxy`；Gateway 网关配置必须同时允许两者：

```json5
{
  gateway: {
    nodes: {
      commands: { allow: ["browser.proxy", "googlemeet.chrome"] },
    },
  },
}
```

如果 `googlemeet setup` 未通过 `chrome-node-connected`，或 Gateway 网关日志报告 `gateway token mismatch`，请使用当前 Gateway 网关令牌重新安装或重启节点：

```bash
OPENCLAW_ALLOW_INSECURE_PRIVATE_WS=1 \
  openclaw node install \
  --host <gateway-lan-ip> \
  --port 18789 \
  --display-name parallels-macos \
  --force
```

然后重新加载节点服务并再次运行：

```bash
openclaw googlemeet setup
openclaw nodes status --connected
```

### 浏览器已打开，但智能体无法加入

对于仅观察加入，请运行 `googlemeet test-listen`；对于实时加入，请运行 `googlemeet test-speech`，然后检查返回的 Chrome 健康状态。如果任一操作报告 `manualActionRequired: true`，请向操作员显示 `manualActionMessage`，并停止重试，直到浏览器操作完成。

常见的手动操作包括：登录 Chrome 配置文件；通过 Meet 主持人账户准许访客加入；出现原生提示时授予 Chrome 麦克风/摄像头权限；关闭或修复卡住的 Meet 权限对话框。

不要仅仅因为 Meet 询问“是否希望会议中的其他人听到你的声音？”就报告“未登录”；这是 Meet 的音频选择过渡页面。可用时，OpenClaw 会通过浏览器自动化点击 **使用麦克风**，并继续等待真实会议状态；对于仅创建的浏览器回退，它可能改为点击 **不使用麦克风继续**，因为生成 URL 不需要实时音频路径。

### 创建会议失败

配置 OAuth 时，`googlemeet create` 使用 Meet API `spaces.create`；否则使用固定 Chrome 节点上的浏览器。请确认：

- **通过 API 创建**：存在 `oauth.clientId` 和 `oauth.refreshToken`（或对应的 `OPENCLAW_GOOGLE_MEET_*` 环境变量），并且刷新令牌是在添加创建支持后生成的；旧令牌可能缺少 `meetings.space.created`，因此请重新运行 `openclaw googlemeet auth login --json`。
- **浏览器回退**：`defaultTransport: "chrome-node"` 和 `chromeNode.node` 指向一个已连接的节点，该节点具有 `browser.proxy` 和 `googlemeet.chrome`；该节点上的 OpenClaw Chrome 配置文件已登录且能够打开 `https://meet.google.com/new`。
- **浏览器回退重试**：打开新标签页之前，复用现有的 `.../new` 或 Google 账户提示标签页；应重试工具调用，而不是手动再打开一个标签页。
- **手动操作**：如果工具返回 `manualActionRequired: true`，请使用 `browser.nodeId`、`browser.targetId`、`browserUrl` 和 `manualActionMessage` 指导操作员；不要循环重试。
- **音频选择过渡页面**：如果 Meet 显示“是否希望会议中的其他人听到你的声音？”，请保持该标签页打开。OpenClaw 应点击 **使用麦克风** 或（仅创建时）**不使用麦克风继续**，并继续等待生成的 URL；如果无法执行，错误应提及 `meet-audio-choice-required`，而不是 `google-login-required`。

### 智能体已加入但不说话

```bash
openclaw googlemeet setup
openclaw googlemeet doctor
```

对于 STT -> OpenClaw 智能体 -> TTS 路径，请使用 `mode: "agent"`；对于直接实时语音回退，请使用 `mode: "bidi"`。`mode: "transcribe"` 按设计不会启动回话桥接。对于仅观察调试，请在参与者发言后运行 `openclaw googlemeet status --json <session-id>`，并检查 `captioning`、`transcriptLines`、`lastCaptionText`。如果 `inCall` 为 true，但 `transcriptLines` 一直为 `0`，则可能是 Meet 字幕已禁用、安装观察器后无人发言、Meet UI 已发生变化，或该会议语言/账户无法使用实时字幕。

`googlemeet test-speech` 始终检查实时路径，并报告在该次调用中是否观察到桥接输出字节。如果 `speechOutputVerified` 为 false 且 `speechOutputTimedOut` 为 true，则实时提供商可能已接受该话语，但 OpenClaw 未观察到新的输出字节到达 Chrome 音频桥接。

还应验证：Gateway 网关主机上存在实时提供商密钥（`OPENAI_API_KEY` 或 `GEMINI_API_KEY`）；Chrome 主机上可以看到 `BlackHole 2ch`；该主机上存在 `sox`；Meet 麦克风/扬声器通过虚拟音频路径路由（对于本地 Chrome 实时加入，`doctor` 应显示 `meet output routed: yes`）。

`googlemeet doctor [session-id]` 会输出会话、节点、通话中状态、手动操作原因、实时提供商连接、`realtimeReady`、音频输入/输出活动、最后音频时间戳、字节计数器和浏览器 URL。使用 `googlemeet status [session-id] --json` 获取原始 JSON，并使用 `googlemeet doctor --oauth`（添加 `--meeting` 或 `--create-space`）验证 OAuth 刷新，而不暴露令牌。

如果智能体已超时且 Meet 标签页已打开，请在不打开其他标签页的情况下检查它：

```bash
openclaw googlemeet recover-tab
openclaw googlemeet recover-tab https://meet.google.com/abc-defg-hij
```

对应的工具操作是 `recover_current_tab`：它会针对所选传输方式（`chrome` 使用本地浏览器控制，`chrome-node` 使用已配置的节点）聚焦并检查现有 Meet 标签页，而不会打开新标签页或新会话，并报告当前阻塞原因（登录、准入、权限、音频选择状态）。CLI 命令会与已配置的 Gateway 网关通信，因此该网关必须正在运行；`chrome-node` 还要求节点已连接。

### Twilio 设置检查失败

当不允许或未启用 `voice-call` 时，`twilio-voice-call-plugin` 会失败：将其添加到 `plugins.allow`，启用 `plugins.entries.voice-call`，然后重新加载 Gateway 网关。

当 Twilio 后端缺少账户 SID、身份验证令牌或主叫号码时，`twilio-voice-call-credentials` 会失败：

```bash
export TWILIO_ACCOUNT_SID=AC...
export TWILIO_AUTH_TOKEN=...
export TWILIO_FROM_NUMBER=+15550001234
```

当 `voice-call` 没有公开 Webhook 暴露，或 `publicUrl` 指向环回/专用网络空间时，`twilio-voice-call-webhook` 会失败。请勿将 `localhost`、`127.0.0.1`、`0.0.0.0`、`10.x`、`172.16.x`-`172.31.x`、`192.168.x`、`169.254.x`、`fc00::/7` 或 `fd00::/8` 用作 `publicUrl`；运营商回调无法访问这些地址。请将 `plugins.entries.voice-call.config.publicUrl` 设置为公共 URL，或配置隧道/Tailscale 暴露：

```json5
{
  plugins: {
    entries: {
      "voice-call": {
        enabled: true,
        config: {
          provider: "twilio",
          fromNumber: "+15550001234",
          publicUrl: "https://voice.example.com/voice/webhook",
        },
      },
    },
  },
}
```

对于本地开发，请使用隧道或 Tailscale 暴露，而不是专用主机 URL：

```json5
{
  plugins: {
    entries: {
      "voice-call": {
        config: {
          tunnel: { provider: "ngrok" },
          // 或
          tailscale: { mode: "funnel", path: "/voice/webhook" },
        },
      },
    },
  },
}
```

重启或重新加载 Gateway 网关，然后运行：

```bash
openclaw googlemeet setup --transport twilio
openclaw voicecall setup
openclaw voicecall smoke
```

默认情况下，`voicecall smoke` 仅检查就绪状态。对特定号码执行试运行：

```bash
openclaw voicecall smoke --to "+15555550123"
```

仅在有意发起真实呼出通话时添加 `--yes`：

```bash
openclaw voicecall smoke --to "+15555550123" --yes
```

### Twilio 通话已开始，但始终未进入会议

确认 Meet 活动提供电话拨入信息，并传入准确的拨入号码及 PIN，或自定义 DTMF 序列：

```bash
openclaw googlemeet join https://meet.google.com/abc-defg-hij \
  --transport twilio \
  --dial-in-number +15551234567 \
  --dtmf-sequence ww123456#
```

在 `--dtmf-sequence` 中使用开头的 `w` 或逗号，以便在输入 PIN 前暂停。

如果通话已创建，但 Meet 名单始终未显示拨入参与者：

- `openclaw googlemeet doctor <session-id>`：确认委托的 Twilio 通话 ID、DTMF 是否已加入队列，以及是否已请求介绍问候语。
- `openclaw voicecall status --call-id <id>`：确认通话仍处于活动状态。
- `openclaw voicecall tail`：确认 Twilio Webhook 正在到达 Gateway 网关。
- `openclaw logs --follow`：查找 Twilio Meet 序列：Google Meet 委托加入操作，语音通话存储并提供连接前 DTMF TwiML，语音通话为 Twilio 通话提供实时 TwiML，随后 Google Meet 使用 `voicecall.speak` 请求介绍语音。
- 重新运行 `openclaw googlemeet setup --transport twilio`；设置检查显示为绿色是必需条件，但不能证明会议 PIN 序列正确。
- 确认拨入号码与 PIN 属于同一个 Meet 邀请和区域。
- 如果 Meet 应答缓慢，或发送连接前 DTMF 后通话转录仍显示 PIN 提示，请将 `voiceCall.dtmfDelayMs` 从默认的 12 秒调大。
- 如果参与者已加入但你听不到问候语，请检查 `openclaw logs --follow` 中是否存在 DTMF 后的 `voicecall.speak` 请求，以及媒体流 TTS 播放或 Twilio `<Say>` 回退。如果转录仍显示“输入会议 PIN”，则电话线路尚未加入 Meet 会议室，因此参与者听不到语音。

如果 Webhooks 未到达，请先调试语音通话插件：提供商必须能够访问 `plugins.entries.voice-call.config.publicUrl` 或已配置的隧道。请参阅[语音通话故障排查](/zh-CN/plugins/voice-call#troubleshooting)。

## 注意事项

Google Meet 的官方媒体 API 以接收为主，因此要在通话中发言，仍需通过参与者路径。此插件明确保留了这一边界：Chrome 负责浏览器参会和本地音频路由；Twilio 负责电话拨入参会。

Chrome 回话模式需要 `BlackHole 2ch`，并加上以下任一选项：

- `chrome.audioInputCommand` 加 `chrome.audioOutputCommand`：OpenClaw 负责桥接，并在这些命令与所选提供商之间通过 `chrome.audioFormat` 传输音频。`agent` 模式使用实时转录加常规 TTS；`bidi` 模式使用实时语音提供商。默认路径为采用 `chrome.audioBufferBytes: 4096` 的 24 kHz PCM16；8 kHz G.711 mu-law 仍可用于旧版命令对。
- `chrome.audioBridgeCommand`：外部桥接命令负责整个本地音频路径，并且必须在启动或验证其守护进程后退出。仅适用于 `bidi`，因为 `agent` 模式需要直接访问命令对来执行 TTS。

使用命令对 Chrome 桥接时，`chrome.bargeInInputCommand` 可以监听单独的本地麦克风，并在人开始说话时清除智能体的播放内容。这样，即使智能体播放期间共享的 BlackHole 回环输入被暂时抑制，也能让人的语音优先于智能体输出。与 `chrome.audioInputCommand`/`chrome.audioOutputCommand` 一样，它是由操作员配置的本地命令：请使用明确且可信的命令路径或参数列表，绝不要使用来自不可信位置的脚本。

为获得清晰的双工音频，请通过不同的虚拟设备或 Loopback 风格的虚拟设备拓扑，分别路由 Meet 输出和 Meet 麦克风；单个共享的 BlackHole 设备可能会将其他参与者的声音回传到通话中。

`googlemeet speak` 会触发 Chrome 会话的活动回话音频桥接；`googlemeet leave` 会将其停止（对于通过语音通话插件委派的 Twilio 会话，还会挂断底层通话）。对于由 API 管理的空间，可使用 `googlemeet end-active-conference` 同时关闭活动的 Google Meet 会议。

## 相关内容

- [会议插件概览](/zh-CN/plugins/meeting-plugins)
- [语音通话插件](/zh-CN/plugins/voice-call)
- [Talk 模式](/zh-CN/nodes/talk)
- [构建插件](/zh-CN/plugins/building-plugins)
