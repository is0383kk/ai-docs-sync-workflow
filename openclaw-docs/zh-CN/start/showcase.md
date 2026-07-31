---
description: Real-world OpenClaw projects from the community
read_when:
    - 寻找真实的 OpenClaw 使用示例
    - 更新社区项目亮点
summary: 由 OpenClaw 驱动的社区构建项目和集成
title: 展示案例
x-i18n:
    generated_at: "2026-07-26T06:34:27Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 64af6f1da52ebdccff82fe2cdb0f7a5f0cd57627b08ee796369e2933f47fbae4
    source_path: start/showcase.md
    workflow: 16
---

OpenClaw 社区项目：PR 审查循环、移动应用、家庭自动化、语音系统、开发工具和记忆工作流，以 Telegram、WhatsApp、Discord 和终端上的聊天原生方式构建。

<Info>
**想在这里展示你的项目？** 在 [Discord 的 #self-promotion](https://discord.gg/clawd) 中分享你的项目，或[在 X 上标记 @openclaw](https://x.com/openclaw)。
</Info>

## Discord 新鲜动态

近期在编程、开发工具、移动应用和聊天原生产品构建领域脱颖而出的项目。

<CardGroup cols={2}>

<Card title="Dropage 即时 HTML 部署" icon="cloud-arrow-up" href="https://clawhub.ai/jiantoucn/skills/dropage-deploy">
  **@jiantoucn** • `deploy` `hosting` `skill`

告诉你的智能体“部署这个 HTML”，大约一秒后即可获得一个公开 URL。页面会在一小时后自动过期——无需服务器、无需配置、无需注册。
</Card>

<Card title="反诈骗 URL 检查器" icon="shield-halved" href="https://clawhub.ai/phishguard-niki/anti-scam-guard">
  **@phishguard-niki** • `security` `phishing` `skill`

粘贴任意 URL，即可获得判定结果。包含来自 38 个数据源（PhishTank、OpenPhish、CERT.PL 等）的 250 万多个诈骗域名，并在本地完成匹配，因此浏览历史绝不会离开设备。
</Card>

<Card title="产品设计推理 Skills" icon="pen-ruler" href="https://clawhub.ai/monikazapisekstudio/skills/socratic-dialog">
  **@monikazapisekstudio** • `product` `reasoning` `skills`

一套用于产品工作的三件套：[苏格拉底式对话](https://clawhub.ai/monikazapisekstudio/skills/socratic-dialog)会在回答前深入盘问问题，[卡诺模型策略师](https://clawhub.ai/monikazapisekstudio/skills/kano-model-strategist)会梳理哪些功能值得保留，而[易读的智能体输出](https://clawhub.ai/monikazapisekstudio/skills/legible-agent-output)会将智能体输出改写为通俗语言。
</Card>

<Card title="子智能体邮箱代理" icon="inbox" href="https://clawhub.ai/albzhu/skills/miab-broker">
  **@albzhu** • `multi-agent` `async` `skill`

避免编排器在子智能体工作期间空闲：通过异步回调机制将结果投递至邮箱，而不是阻塞父智能体。
</Card>

<Card title="适用于低内存设备的 lite-mode" icon="feather" href="https://clawhub.ai/skills/lite-mode">
  **@mirajmahmudul** • `performance` `skill`

让 OpenClaw 在 2-4 GB 内存的设备上仍可正常使用：检查可用内存，并在设备开始使用交换空间前精简高负载功能。[GitHub 源代码](https://github.com/mirajmahmudul/openclaw-lite-mode)。
</Card>

<Card title="tokenomics 成本跟踪器" icon="coins" href="https://github.com/ncz-os/tokenomics">
  **@ncz-os** • `devtools` `costs` `tokens`

由 NVIDIA 工程师打造的 Token 成本跟踪器，原生支持 OpenClaw：按模型和会话准确查看智能体费用的去向。
</Card>

<Card title="Excalidraw 图表生成器" icon="shapes" href="https://x.com/swiftlysingh/status/2009684853827281070">
  **@swiftlysingh** • `diagrams` `excalidraw` `devtools`

在聊天中描述一个图表，即可获得以编程方式生成的 Excalidraw 草图。
</Card>

<Card title="GA4 分析 Skill" icon="chart-column" href="https://x.com/jdrhyne/status/2012028725710192741">
  **@jdrhyne** • `analytics` `ga4` `skill`

让 OpenClaw 构建自己的 Google Analytics 查询工具，然后将其打包并发布到 ClawHub。
</Card>

<Card title="ClawEval 模型排名" icon="ranking-star" href="https://github.com/AIgenteur/ClawEval">
  **@AIgenteur** • `evals` `models` `devtools`

在 59 种智能体角色中对模型进行基准测试，以回答“我的 GPU 该用哪个 LLM？”这一问题。这是社区挑选本地模型时的热门选择。
</Card>

<Card title="Music Craft" icon="music" href="https://clawhub.ai/luischarro/music-craft">
  **@luischarro** • `music` `generation` `skill`

不依赖特定提供商的歌曲生成：规划曲目、组织歌词，并修订内容稀疏的结果，而不是只进行一次提示。还包括一个 [MiniMax 变体](https://clawhub.ai/luischarro/music-craft-minimax)，可控制 BPM、调性、结构和混搭。
</Card>

<Card title="从 PR 审查到 Telegram 反馈" icon="code-pull-request" href="https://x.com/i/status/2010878524543131691">
  **@bangnokia** • `review` `github` `telegram`

OpenCode 完成修改并创建 PR，OpenClaw 审查差异，然后在 Telegram 中回复建议以及明确的合并结论。

  <img src="/assets/showcase/pr-review-telegram.jpg" alt="通过 Telegram 发送的 OpenClaw PR 审查反馈" />
</Card>

<Card title="几分钟内创建酒窖 Skill" icon="wine-glass" href="https://x.com/i/status/2010916352454791216">
  **@prades_maxime** • `skills` `local` `csv`

请求“Robby”（@openclaw）创建本地酒窖 Skill。它会要求提供 CSV 导出样本和存储路径，然后构建并测试该 Skill（示例中有 962 瓶酒）。

  <img src="/assets/showcase/wine-cellar-skill.jpg" alt="OpenClaw 根据 CSV 构建本地酒窖 Skill" />
</Card>

<Card title="Tesco 购物自动驾驶" icon="cart-shopping" href="https://x.com/i/status/2009724862470689131">
  **@marchattonhere** • `automation` `browser` `shopping`

制定每周膳食计划、添加常购商品、预约配送时段、确认订单。无需 API，只需浏览器控制。

  <img src="/assets/showcase/tesco-shop.jpg" alt="通过聊天实现 Tesco 购物自动化" />
</Card>

<Card title="SNAG 截图转 Markdown" icon="scissors" href="https://github.com/am-will/snag">
  **@am-will** • `devtools` `screenshots` `markdown`

通过快捷键截取屏幕区域，使用 Gemini 视觉识别，并立即将 Markdown 写入剪贴板。

  <img src="/assets/showcase/snag.png" alt="SNAG 截图转 Markdown 工具" />
</Card>

<Card title="智能体 UI" icon="window-maximize" href="https://releaseflow.net/kitze/agents-ui">
  **@kitze** • `ui` `skills` `sync`

用于跨 Agents、Claude、Codex 和 OpenClaw 管理 Skills 与命令的桌面应用。

  <img src="/assets/showcase/agents-ui.jpg" alt="智能体 UI 应用" />
</Card>

<Card title="Telegram 语音消息（papla.media）" icon="microphone" href="https://papla.media/docs">
  **社区** • `voice` `tts` `telegram`

封装 papla.media TTS，并将结果作为 Telegram 语音消息发送（不会烦人地自动播放）。

  <img src="/assets/showcase/papla-tts.jpg" alt="TTS 生成的 Telegram 语音消息输出" />
</Card>

<Card title="CodexMonitor" icon="eye" href="https://clawhub.ai/odrobnik/skills/codexmonitor">
  **@odrobnik** • `devtools` `codex` `brew`

通过 Homebrew 安装的辅助工具，用于列出、检查和监视本地 OpenAI Codex 会话（CLI + VS Code）。

  <img src="/assets/showcase/codexmonitor.png" alt="ClawHub 上的 CodexMonitor" />
</Card>

<Card title="Bambu 3D 打印机控制" icon="print" href="https://clawhub.ai/tobiasbischoff/skills/bambu-cli">
  **@tobiasbischoff** • `hardware` `3d-printing` `skill`

控制 BambuLab 打印机并排查故障：状态、任务、摄像头、AMS、校准等。

  <img src="/assets/showcase/bambu-cli.png" alt="ClawHub 上的 Bambu CLI Skill" />
</Card>

<Card title="维也纳交通（Wiener Linien）" icon="train" href="https://clawhub.ai/hjanuschka/skills/wienerlinien">
  **@hjanuschka** • `travel` `transport` `skill`

提供维也纳公共交通的实时发车信息、运营中断、电梯状态和路线规划。

  <img src="/assets/showcase/wienerlinien.png" alt="ClawHub 上的 Wiener Linien Skill" />
</Card>

<Card title="ParentPay 学校餐食" icon="utensils">
  **@George5562** • `automation` `browser` `parenting`

通过 ParentPay 自动预订英国学校餐食。使用鼠标坐标可靠地点击表格单元格。
</Card>

<Card title="R2 上传（把文件发给我）" icon="cloud-arrow-up" href="https://clawhub.ai/julianengel/skills/r2-upload">
  **@julianengel** • `files` `r2` `presigned-urls`

上传到 Cloudflare R2/S3，并生成安全的预签名下载链接。适用于远程 OpenClaw 实例。

  <img src="/assets/showcase/r2-upload.png" alt="ClawHub 上的 R2 上传 Skill" />
</Card>

<Card title="通过 Telegram 构建 iOS 应用" icon="mobile">
  **@coard** • `ios` `xcode` `app-store`

完全通过 Telegram 聊天构建了一款包含地图和录音功能的完整 iOS 应用，并为 App Store 发布做好准备。
</Card>

<Card title="Oura Ring 健康助手" icon="heart-pulse">
  **@AS** • `health` `oura` `calendar`

个人 AI 健康助手，将 Oura Ring 数据与日历、预约和健身房日程集成。

  <img src="/assets/showcase/oura-health.png" alt="Oura Ring 健康助手" />
</Card>

<Card title="Kev 的梦之队（14+ 个智能体）" icon="robot" href="https://github.com/adam91holt/orchestrated-ai-articles">
  **@adam91holt** • `multi-agent` `orchestration`

一个 Gateway 网关下运行 14+ 个智能体，由 Opus 4.5 编排器将任务委派给 Codex 工作智能体。参阅[技术说明](https://github.com/adam91holt/orchestrated-ai-articles)，以及用于智能体沙箱隔离的 [Clawdspace](https://github.com/adam91holt/clawdspace)。
</Card>

<Card title="Linear CLI" icon="terminal" href="https://github.com/Finesssee/linear-cli">
  **@NessZerra** • `devtools` `linear` `cli`

与智能体工作流（Claude Code、OpenClaw）集成的 Linear CLI。通过终端管理议题、项目和工作流。
</Card>

<Card title="Beeper CLI" icon="message" href="https://github.com/blqke/beepcli">
  **@jules** • `messaging` `beeper` `cli`

通过 Beeper Desktop 读取、发送和归档消息。使用 Beeper 本地 MCP API，让智能体可以在一个地方管理你的所有聊天（iMessage、WhatsApp 等）。
</Card>

</CardGroup>

## 自动化和工作流

调度、浏览器控制、支持循环，以及产品中“直接替我完成任务”的一面。

<CardGroup cols={2}>

<Card title="Winix 空气净化器控制" icon="wind" href="https://x.com/antonplex/status/2010518442471006253">
  **@antonplex** • `automation` `hardware` `air-quality`

Claude Code 发现并确认了净化器控制方式，随后由 OpenClaw 接管室内空气质量管理。

  <img src="/assets/showcase/winix-air-purifier.jpg" alt="通过 OpenClaw 控制 Winix 空气净化器" />
</Card>

<Card title="拍摄漂亮的天空照片" icon="camera" href="https://x.com/signalgaining/status/2010523120604746151">
  **@signalgaining** • `automation` `camera` `skill`

由屋顶摄像头触发：让 OpenClaw 在天空看起来很美时拍一张照片。它设计了一个 Skill，并完成了拍摄。

  <img src="/assets/showcase/roof-camera-sky.jpg" alt="OpenClaw 通过屋顶摄像头拍摄的天空快照" />
</Card>

<Card title="可视化晨间简报场景" icon="robot" href="https://x.com/buddyhadry/status/2010005331925954739">
  **@buddyhadry** • `automation` `briefing` `telegram`

通过 OpenClaw 人设，使用定时提示每天早晨生成一张场景图片（天气、任务、日期、喜爱的帖子或引语）。
</Card>

<Card title="板式网球场预订" icon="calendar-check" href="https://github.com/joshp123/padel-cli">
  **@joshp123** • `automation` `booking` `cli`

Playtomic 空闲时段检查器和预订 CLI。再也不会错过空闲球场。

  <img src="/assets/showcase/padel-screenshot.jpg" alt="padel-cli 截图" />
</Card>

<Card title="会计资料收集" icon="file-invoice-dollar">
  **社区** • `automation` `email` `pdf`

收集电子邮件中的 PDF，为税务顾问准备文档。每月会计工作自动运行。
</Card>

<Card title="沙发土豆开发模式" icon="couch" href="https://davekiss.com">
  **@davekiss** • `telegram` `migration` `astro`

一边看 Netflix，一边通过 Telegram 重建了整个个人网站——从 Notion 迁移到 Astro，迁移了 18 篇文章，并将 DNS 转移到 Cloudflare。全程没有打开笔记本电脑。
</Card>

<Card title="求职智能体" icon="briefcase">
  **@attol8** • `automation` `api` `skill`

搜索招聘信息，根据简历关键词进行匹配，并返回带链接的相关机会。使用 JSearch API 在 30 分钟内构建完成。
</Card>

<Card title="Jira 技能构建器" icon="diagram-project" href="https://x.com/jdrhyne/status/2008336434827002232">
  **@jdrhyne** • `jira` `skill` `devtools`

OpenClaw 连接到 Jira 后，即时生成了一个新技能（当时 ClawHub 上还不存在该技能）。
</Card>

<Card title="通过 Telegram 使用 Todoist 技能" icon="list-check" href="https://x.com/iamsubhrajyoti/status/2009949389884920153">
  **@iamsubhrajyoti** • `todoist` `skill` `telegram`

自动处理 Todoist 任务，并让 OpenClaw 直接在 Telegram 聊天中生成该技能。
</Card>

<Card title="TradingView 分析" icon="chart-line">
  **@bheem1798** • `finance` `browser` `automation`

通过浏览器自动化登录 TradingView，截取图表，并按需执行技术分析。无需 API，只需浏览器控制。
</Card>

<Card title="汽车议价（节省 $4,200）" icon="car-side" href="https://x.com/astuyve/status/2014147784098681217">
  **@astuyve** • `negotiation` `email` `automation`

让 OpenClaw 自主与汽车经销商交涉：它处理了往返议价，并将价格降低了 $4,200。
</Card>

<Card title="航班值机自动驾驶" icon="plane-departure" href="https://x.com/armanddp/status/2008767951340794245">
  **@armanddp** • `travel` `email` `automation`

从电子邮件中找到下一趟航班，完成在线值机，并选择靠窗座位——无需航空公司应用。
</Card>

<Card title="提交保险理赔" icon="file-signature" href="https://x.com/avi_press/status/2013066316467560521">
  **@avi_press** • `automation` `insurance` `browser`

自主提交保险理赔并预约后续会面。
</Card>

<Card title="Idealista 房地产技能" icon="building" href="https://x.com/quifago/status/2012458753786859872">
  **@quifago** • `real-estate` `api` `skill`

用于房产查询和估值的 Idealista API CLI，封装为技能，使智能体能在聊天中寻找房源。
</Card>

<Card title="园艺企业后台" icon="seedling" href="https://news.ycombinator.com/item?id=47783940">
  **@mjsweet** • `automation` `email` `invoicing`

监控 Gmail 中的工作订单，分析通过 Telegram 发送的房产照片，编写多页 LaTeX 报价 PDF，并通过 Xero 开具发票。
</Card>

<Card title="Slack 自动支持" icon="slack">
  **@henrymascot** • `slack` `automation` `support`

监控公司 Slack 频道，提供有用的回复，并将通知转发到 Telegram。它还在无人要求的情况下，自主修复了已部署应用中的生产环境错误。
</Card>

</CardGroup>

## 知识与记忆

用于索引、搜索、记忆个人或团队知识并进行推理的系统。

<CardGroup cols={2}>

<Card title="xuezh 中文学习" icon="language" href="https://github.com/joshp123/xuezh">
  **@joshp123** • `learning` `voice` `skill`

通过 OpenClaw 提供发音反馈和学习流程的中文学习引擎。

  <img src="/assets/showcase/xuezh-pronunciation.jpeg" alt="xuezh 发音反馈" />
</Card>

<Card title="X 帖子分析流水线" icon="hashtag" href="https://x.com/andrewjiang/status/2008388427180630155">
  **@andrewjiang** • `analysis` `x` `pipeline`

从 100 个热门 X 账号中获取了 400 万篇帖子，并将其转化为可查询的分析流水线。
</Card>

<Card title="将化验结果导入 Notion" icon="flask" href="https://x.com/danpeguine/status/2013388700479058068">
  **@danpeguine** • `health` `notion` `organization`

将多年来的血液化验结果整理到结构化的 Notion 数据库中。
</Card>

<Card title="Obsidian 第二大脑" icon="book" href="https://notesbylex.com/openclaw-the-missing-piece-for-obsidians-second-brain">
  **@lexandstuff** • `obsidian` `whatsapp` `memory`

WhatsApp 上的日常主力助手，所有记忆均以 Markdown 格式存储在受版本控制的 Obsidian 仓库中：记录卡路里和锻炼、管理待办事项及生活事务。
</Card>

<Card title="家族史机器人" icon="people-roof" href="https://news.ycombinator.com/item?id=47783940">
  **@brtkwr** • `telegram` `memory` `family`

驻留在家庭 Telegram 群聊中，记录 50 多位亲属的故事，并提出有针对性的后续问题——为母语使用者使用尼泊尔语回复。
</Card>

<Card title="WhatsApp 记忆库" icon="vault">
  **社区** • `memory` `transcription` `indexing`

导入完整的 WhatsApp 导出内容，转录 1,000 多条语音消息，与 Git 日志交叉核对，并输出带链接的 Markdown 报告。
</Card>

<Card title="Karakeep 语义搜索" icon="magnifying-glass" href="https://github.com/jamesbrooksco/karakeep-semantic-search">
  **@jamesbrooksco** • `search` `vector` `bookmarks`

使用 Qdrant 加 OpenAI 或 Ollama 嵌入，为 Karakeep 书签添加向量搜索功能。
</Card>

<Card title="Inside-Out-2 记忆" icon="brain">
  **社区** • `memory` `beliefs` `self-model`

独立的记忆管理器，将会话文件转化为记忆，再转化为信念，最终形成不断演化的自我模型。
</Card>

</CardGroup>

## 语音与电话

语音优先的入口、电话桥接和大量使用转录的工作流。

<CardGroup cols={2}>

<Card title="Pebble Ring 一键语音" icon="ring" href="https://x.com/thekitze/status/2014765279650189578">
  **@thekitze** • `voice` `wearable` `hardware`

轻触一下 Pebble Ring 即可与 OpenClaw 开始语音对话——通过可穿戴设备访问智能体。
</Card>

<Card title="创作者媒体工作室" icon="clapperboard" href="https://x.com/cedric_chee/status/2014608153393168425">
  **@cedric_chee** • `media` `tts` `transcription`

聊天中的完整媒体工作室：将 TTS、转录和浏览器自动化接入 Codex 5.2 与 MiniMax。
</Card>

<Card title="操作按钮对讲机" icon="walkie-talkie" href="https://x.com/i/status/2072766510053888497">
  **@buddyhadry** • `voice` `ios` `mobile`

将 iPhone 操作按钮连接到 OpenClaw：按住、说话，智能体就会像对讲机一样回应。
</Card>

<Card title="Clawdia 电话桥接" icon="phone" href="https://github.com/alejandroOPI/clawdia-bridge">
  **@alejandroOPI** • `voice` `vapi` `bridge`

连接 Vapi 语音助手与 OpenClaw 的 HTTP 桥接。让你能与智能体进行近乎实时的电话通话。
</Card>

<Card title="OpenRouter 转录" icon="microphone" href="https://clawhub.ai/obviyus/skills/openrouter-transcribe">
  **@obviyus** • `transcription` `multilingual` `skill`

通过 OpenRouter（Gemini 等）进行多语言音频转录。可在 ClawHub 上获取。

  <img src="/assets/showcase/openrouter-transcribe.png" alt="ClawHub 上的 OpenRouter 转录技能" />
</Card>

</CardGroup>

## 基础设施与部署

使 OpenClaw 更易于运行和扩展的打包、部署与集成。

<CardGroup cols={2}>

<Card title="Home Assistant 附加组件" icon="home" href="https://github.com/ngutman/openclaw-ha-addon">
  **@ngutman** • `homeassistant` `docker` `raspberry-pi`

在 Home Assistant OS 上运行的 OpenClaw Gateway 网关，支持 SSH 隧道和持久化状态。
</Card>

<Card title="Home Assistant 技能" icon="toggle-on" href="https://clawhub.ai/homeofe/skills/openclaw-homeassistant">
  **@homeofe** • `homeassistant` `skill` `automation`

通过自然语言控制 Home Assistant 设备并实现自动化。

  <img src="/assets/showcase/homeassistant.png" alt="ClawHub 上的 Home Assistant 技能" />
</Card>

<Card title="macOS 菜单栏管理器" icon="desktop" href="https://x.com/MagiMetal/status/2009424267801485362">
  **@MagiMetal** • `macos` `swift` `ui`

原生 Swift 菜单栏应用，可显示智能体状态并提供快捷控制。
</Card>

<Card title="Nix 打包" icon="snowflake" href="https://github.com/openclaw/nix-openclaw">
  **@openclaw** • `nix` `packaging` `deployment`

功能完备、经过 Nix 化的 OpenClaw 配置，支持可复现部署。
</Card>

<Card title="CalDAV 日历" icon="calendar" href="https://clawhub.ai/asleep123/skills/caldav-calendar">
  **@asleep123** • `calendar` `caldav` `skill`

使用 khal 和 vdirsyncer 的日历技能。自托管日历集成。

  <img src="/assets/showcase/caldav-calendar.png" alt="ClawHub 上的 CalDAV 日历技能" />
</Card>

</CardGroup>

## 家庭与硬件

OpenClaw 在现实世界中的应用：住宅、传感器、摄像头、吸尘器及其他设备。

<CardGroup cols={2}>

<Card title="自建 HomePod 技能" icon="volume-high" href="https://x.com/localghost/status/2014763987683225685">
  **@localghost** • `homepod` `discovery` `skill`

OpenClaw 在本地网络中找到了 HomePod，并自行编写了控制它们的技能。
</Card>

<Card title="$35 全息立方体界面" icon="cube" href="https://x.com/andrewjiang/status/2013140793649734032">
  **@andrewjiang** • `hardware` `display` `fun`

用一个廉价全息立方体作为智能体在桌面上的实体面孔。
</Card>

<Card title="GoHome 自动化" icon="house-signal" href="https://github.com/joshp123/gohome">
  **@joshp123** • `home` `nix` `grafana`

以 OpenClaw 作为界面的 Nix 原生家庭自动化系统，并配有 Grafana 仪表板。

  <img src="/assets/showcase/gohome-grafana.png" alt="GoHome Grafana 仪表板" />
</Card>

<Card title="Roborock 扫地机器人" icon="robot" href="https://github.com/joshp123/gohome/tree/main/plugins/roborock">
  **@joshp123** • `vacuum` `iot` `plugin`

通过自然对话控制你的 Roborock 扫地机器人。

  <img src="/assets/showcase/roborock-screenshot.jpg" alt="Roborock 状态" />
</Card>

</CardGroup>

## 社区项目

从单一工作流发展为更广泛产品或生态系统的项目。

<CardGroup cols={2}>

<Card title="StarSwap 市场" icon="star" href="https://star-swap.com/">
  **社区** • `marketplace` `astronomy` `webapp`

完整的天文设备市场。基于 OpenClaw 生态系统构建，并围绕该生态运行。
</Card>

<Card title="Clinch 智能体协商协议" icon="handshake" href="https://clawhub.ai/publicstringapps/clinch">
  **@publicstringapps** • `protocol` `p2p` `skill`

开放的智能体间协商：你的智能体会与其他节点就交易、日程和服务协议讨价还价，并以加密方式签署结果——你只需批准或拒绝。
</Card>

</CardGroup>

## 提交你的项目

<Steps>
  <Step title="分享项目">
    发布到 [Discord 上的 #self-promotion](https://discord.gg/clawd)，或[发推文并提及 @openclaw](https://x.com/openclaw)。
  </Step>
  <Step title="提供详细信息">
    告诉我们它的功能，附上代码仓库或演示链接；如果有截图，也请一并分享。
  </Step>
  <Step title="获得推荐">
    我们会将出色的项目添加到此页面。
  </Step>
</Steps>

## 相关内容

- [入门指南](/zh-CN/start/getting-started)
- [OpenClaw](/zh-CN/start/openclaw)
- [openclaw.ai 上的完整 X 项目展示](https://openclaw.ai/showcase/)
