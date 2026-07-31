---
doc-schema-version: 1
read_when:
    - 了解 QA 技术栈如何协同工作
    - 扩展 qa-lab、qa-channel 或传输适配器
    - 添加由仓库支持的 QA 场景
    - 围绕 Gateway 网关仪表板构建更高真实度的 QA 自动化
summary: QA 栈概览：qa-lab、qa-channel、仓库支持的场景、实时传输通道、传输适配器和报告。
title: QA overview
x-i18n:
    generated_at: "2026-07-26T06:40:30Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 91c34a50e6197195d57228d92b19caff1785ceaa5d82d7c88a1ec0ed76abd635
    source_path: concepts/qa-e2e-automation.md
    workflow: 16
---

私有 QA 技术栈以贴近真实情况、符合渠道形态的方式测试 OpenClaw，这是单元测试无法做到的。

组成部分：

- `extensions/qa-channel`：合成消息渠道，包含私信、频道、话题串、表情回应、编辑和删除功能。
- `extensions/qa-lab`：调试器 UI、QA 总线、场景配置文件和实时传输适配器，用于观察对话记录、注入入站消息并导出 Markdown 报告。
- `qa/`：由仓库支持的初始任务种子资源和基准 QA 场景。
- [Mantis](/zh-CN/concepts/mantis)：针对需要真实传输协议、浏览器截图、虚拟机状态和 PR 证据的错误进行修复前后实时验证。

## 命令界面

所有 QA 流程都在 `pnpm openclaw qa <subcommand>` 下运行。许多流程都有 `pnpm qa:*` 脚本别名；两种形式均可使用。

| 命令                                                | 用途                                                                                                                                                                                                                                                                |
| --------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `qa run`                                            | 不使用 `--qa-profile` 的内置 QA 自检；由分类法支持的成熟度配置文件运行器，可配合 `--qa-profile smoke-ci`、`--qa-profile release` 或 `--qa-profile all` 使用。                                                                                                  |
| `qa suite`                                          | 针对 QA Gateway 网关通道运行由仓库支持的场景。`--runner multipass` 使用一次性 Linux 虚拟机而非主机。                                                                                                                                         |
| `qa coverage`                                       | 输出 YAML 场景覆盖清单（使用 `--json` 输出机器可读格式；使用 `--match <query>` 查找与所改行为相关的场景；使用 `--tools` 查看运行时工具固件覆盖情况）。                                                                                  |
| `qa parity-report`                                  | 比较两个 `qa-suite-summary.json` 文件以进行模型维度一致性门控，或使用 `--runtime-axis --token-efficiency` 写入 Codex 与 OpenClaw 的运行时一致性和令牌效率报告。                                                                          |
| `qa confidence-report`                              | 根据清单对 QA 证明工件进行分类，生成未知项为零的置信度报告。                                                                                                                                                                               |
| `qa confidence-self-test`                           | 写入带种子的负面对照金丝雀，用于证明置信度门控能够检测漂移。                                                                                                                                                                                   |
| `qa jsonl-replay`                                   | 通过运行时一致性重放工具重放精选的 JSONL 对话记录。                                                                                                                                                                                         |
| `qa character-eval`                                 | 跨多个实时模型运行角色 QA 场景并生成评判报告。请参阅[报告](#reporting)。                                                                                                                                                        |
| `qa manual`                                         | 针对所选提供商/模型通道运行一次性提示词。                                                                                                                                                                                                      |
| `qa ui`                                             | 启动 QA 调试器 UI 和本地 QA 总线（别名：`pnpm qa:lab:ui`）。                                                                                                                                                                                                |
| `qa docker-build-image`                             | 构建预制 QA Docker 镜像。                                                                                                                                                                                                                                 |
| `qa docker-scaffold`                                | 为 QA 仪表板和 Gateway 网关通道写入 docker-compose 脚手架。                                                                                                                                                                                                |
| `qa up`                                             | 构建 QA 站点，启动由 Docker 支持的技术栈并输出 URL（别名：`pnpm qa:lab:up`；`:fast` 变体会添加 `--use-prebuilt-image --bind-ui-dist --skip-ui-build`）。                                                                                              |
| `qa aimock`                                         | 仅启动 AIMock 提供商服务器。                                                                                                                                                                                                                              |
| `qa mock-openai`                                    | 仅启动场景感知的 `mock-openai` 提供商服务器。                                                                                                                                                                                                        |
| `qa credentials doctor` / `add` / `list` / `remove` | 管理共享的 Convex 凭据池。                                                                                                                                                                                                                           |
| `qa discord`                                        | 针对真实私有 Discord 服务器频道的实时传输通道。                                                                                                                                                                                                   |
| `qa matrix`                                         | 针对一次性 Tuwunel 主服务器运行 QA Lab Matrix 配置文件。请参阅 [Matrix 冒烟通道](#matrix-smoke-lanes)。                                                                                                                                                      |
| `qa slack`                                          | 针对真实私有 Slack 频道的实时传输通道。                                                                                                                                                                                                           |
| `qa telegram`                                       | 针对真实私有 Telegram 群组的实时传输通道。                                                                                                                                                                                                          |
| `qa whatsapp`                                       | 针对真实 WhatsApp Web 账户的实时传输通道。                                                                                                                                                                                                             |
| `qa mantis`                                         | 实时传输错误的修复前后验证运行器，包含 Discord 状态表情回应证据、Crabbox 桌面端/浏览器冒烟测试和 VNC 中的 Slack 冒烟测试。请参阅 [Mantis](/zh-CN/concepts/mantis) 和 [Mantis Slack 桌面端运行手册](/zh-CN/concepts/mantis-slack-desktop-runbook)。 |

### 基于配置文件的 `qa run`

基于配置文件的 `qa run` 从 `taxonomy.yaml` 读取成员关系，然后通过 `qa suite` 分派解析后的场景。`--surface` 和 `--category` 用于筛选所选配置文件，而不是定义单独的通道。生成的 `qa-evidence.json` 包含配置文件评分卡摘要，其中列出所选类别的数量和缺失的覆盖 ID；各个证据条目仍是测试、覆盖角色和结果的事实依据。分类法功能覆盖 ID 是精确的证明目标，而非别名：主要场景覆盖满足匹配的 ID，而次要覆盖仅作为参考。每个覆盖 ID 都严格采用 `taxonomy-surface.feature` 格式，其中使用来自 `taxonomy.yaml` 的简短功能面 ID。场景中单独的 `surface` 字段是执行/报告标签（例如 `channel` 或 `runtime-tool`）；它不定义分类法归属关系。

精简证据会省略每个条目的 `execution`，并设置 `evidenceMode: "slim"`；`smoke-ci` 默认使用精简模式，而 `--evidence-mode full` 会恢复完整条目：

```bash
pnpm openclaw qa run \
  --qa-profile smoke-ci \
  --category channels.conversation-routing-and-delivery \
  --provider-mode mock-openai \
  --output-dir .artifacts/qa-e2e/smoke-ci-profile-dispatch
```

使用 `smoke-ci`，通过模拟模型提供商和 Crabline 本地提供商服务器生成确定性的配置文件证明。使用 `release`，针对实时渠道进行 Stable/LTS 证明。仅在明确进行完整分类法证据运行时使用 `all`；它会选择所有活跃的成熟度类别，并可通过 `QA
Profile Evidence` GitHub Actions 工作流配合 `qa_profile=all` 进行分派。当命令还需要 OpenClaw 根配置文件时，请将根配置文件置于 QA 命令之前：

```bash
pnpm openclaw --profile work qa run --qa-profile smoke-ci
```

## 操作员流程

当前的 QA 操作员流程采用双窗格 QA 站点：

- 左侧：带有智能体的 Gateway 网关仪表板（Control UI）。
- 右侧：QA Lab，显示类似 Slack 的对话记录和场景计划。

运行方式：

```bash
pnpm qa:lab:up
```

该命令会构建 QA 站点、启动由 Docker 支持的 Gateway 网关通道，并公开 QA Lab 页面，操作员或自动化循环可在其中向智能体分配 QA 任务、观察真实的渠道行为，并记录哪些内容正常工作、失败或仍处于阻塞状态。

如需更快地迭代 QA Lab UI，且不希望每次都重新构建 Docker 镜像，请使用绑定挂载的 QA Lab 包启动技术栈：

```bash
pnpm openclaw qa docker-build-image
pnpm qa:lab:build
pnpm qa:lab:up:fast
pnpm qa:lab:watch
```

`qa:lab:up:fast` 让 Docker 服务继续使用预构建镜像，并将 `extensions/qa-lab/web/dist` 绑定挂载到 `qa-lab` 容器中。`qa:lab:watch` 会在发生更改时重新构建该包，而当 QA Lab 资源哈希值变化时，浏览器会自动重新加载。

### 可观测性冒烟测试

<Note>
可观测性 QA 仅保留在源代码检出中。npm tarball 会有意省略 QA Lab（以及 `qa-channel`），因此软件包 Docker 发布通道不会运行 `qa` 命令。更改诊断插桩时，请从已构建的源代码检出中运行这些命令。
</Note>

| 别名                                    | 运行内容                                                                                                                                |
| --------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------- |
| `pnpm qa:otel:smoke`                    | 本地 OpenTelemetry 接收器，以及启用了 `diagnostics-otel` 的 `otel-trace-smoke` 场景。                                      |
| `pnpm qa:otel:collector-smoke`          | 位于真实 OpenTelemetry Collector Docker 容器之后的同一通道。更改端点连接或收集器/OTLP 兼容性时使用。 |
| `pnpm qa:prometheus:smoke`              | 启用了 `diagnostics-prometheus` 的 `docker-prometheus-smoke` 场景。                                                           |
| `pnpm qa:observability:smoke`           | 先运行 `qa:otel:smoke`，然后运行 `qa:prometheus:smoke`。                                                                                      |
| `pnpm qa:observability:collector-smoke` | 先运行 `qa:otel:collector-smoke`，然后运行 `qa:prometheus:smoke`。                                                                            |

`qa:otel:smoke` 会启动本地 OTLP/HTTP 接收器，运行最小化的 QA channel
智能体轮次，然后断言已导出跟踪、指标和日志。它会解码
导出的 protobuf 跟踪 span，并检查发布关键结构：
必须包含 `openclaw.run`、`openclaw.harness.run`、采用最新 GenAI 语义约定的
模型调用 span、`openclaw.context.assembled` 和 `openclaw.message.delivery`。
冒烟测试会强制使用
`OTEL_SEMCONV_STABILITY_OPT_IN=gen_ai_latest_experimental`，因此模型调用
span 必须使用 `{gen_ai.operation.name} {gen_ai.request.model}` 名称；模型
调用在轮次成功时不得导出 `StreamAbandoned`；原始诊断
ID 和 `openclaw.content.*` 属性不得出现在跟踪中。该场景的
提示词要求模型使用固定标记回复，并禁止输出固定的
秘密字符串；原始 OTLP 负载不得包含两者，也不得包含从场景 ID
派生的 QA 会话键。它会在 QA 套件工件旁写入 `otel-smoke-summary.json`。

`qa:prometheus:smoke` 会验证未经身份验证的抓取请求遭到拒绝，然后
检查经过身份验证的抓取结果是否包含发布关键指标族，且不包含
提示词内容、响应内容、原始诊断标识符、身份验证
令牌或本地路径。

### Matrix 冒烟测试通道

要运行不需要模型提供商凭据、使用真实传输的 Matrix 冒烟测试通道，
请使用确定性的模拟 OpenAI provider 运行发布配置：

```bash
pnpm openclaw qa matrix --provider-mode mock-openai --profile release
```

对于实时前沿提供商通道，请显式提供 OpenAI 兼容凭据：

```bash
OPENCLAW_LIVE_OPENAI_KEY="${OPENAI_API_KEY}" \
  pnpm openclaw qa matrix --provider-mode live-frontier --profile release
```

直接运行 `pnpm openclaw qa matrix` 会执行完整的 `all` 配置，并在
场景失败后继续。使用 `--fail-fast` 可缩短反馈周期，或重复
`--scenario <id>` 以选择单个场景；显式场景 ID 的优先级
高于 `--profile`。

| 配置         | 场景数    | 用途                                                                                                                                     |
| ------------ | --------- | ---------------------------------------------------------------------------------------------------------------------------------------- |
| `all`        | 93        | 完整目录（默认）。                                                                                                              |
| `release`    | 2         | 发布关键渠道基线和实时允许列表重新加载。                                                                             |
| `fast`       | 12        | 针对线程、表情回应、审批、策略、Bot 门控和加密回复的重点覆盖。                                               |
| `transport`  | 50        | 覆盖线程、私信/房间路由、自动加入、审批、表情回应、重启、提及/允许列表策略、编辑和多参与者顺序。         |
| `media`      | 7         | 覆盖图像、生成的图像、语音、附件、不支持的媒体和加密媒体。                                              |
| `e2ee-smoke` | 8         | 最小化覆盖加密回复、线程、引导启动、恢复、重启、修订和失败。                                       |
| `e2ee-deep`  | 18        | 覆盖状态丢失、备份、密钥恢复、设备卫生和 SAS/QR/私信验证。                                                            |
| `e2ee-cli`   | 9         | 通过测试框架覆盖 `openclaw matrix encryption setup`、恢复密钥、多账户、Gateway 网关往返和自验证命令。 |

配置成员和渠道要求与
`qa/scenarios/channels/` 下的声明式 Matrix 场景放在一起。运行时会选择渠道驱动程序。
其实时实现位于
`extensions/qa-lab/src/live-transports/matrix/scenarios/` 下。

适配器会在 Docker 中部署一次性 Tuwunel homeserver（默认
镜像为 `ghcr.io/matrix-construct/tuwunel:v1.5.1`，服务器名称为 `matrix-qa.test`，
端口为 `28008`），注册临时驱动程序、被测系统和观察者用户，初始化
所需房间，并记录经过修订的请求/响应边界。然后，它会
在限定于该传输的子 QA Gateway 网关中运行真实 Matrix 插件
（无 `qa-channel`），最后拆除环境。

常用选项：

| 标志                     | 默认值            | 用途                                                                                 |
| ------------------------ | ----------------- | ------------------------------------------------------------------------------------ |
| `--profile <profile>`    | `all`             | 选择上述配置之一。                                                    |
| `--scenario <id>`        | -                 | 选择一个场景；可重复指定。                                                     |
| `--fail-fast`            | 关闭              | 在首次检查或场景失败后停止。                                       |
| `--allow-failures`       | 关闭              | 写入工件，但不因场景失败返回失败退出代码。         |
| `--provider-mode <mode>` | `live-frontier`   | 使用 `mock-openai` 进行确定性分派，或使用 `live-frontier` 连接实时提供商。 |
| `--model <ref>`          | 提供商默认值      | 设置主要 `provider/model` 引用。                                          |
| `--alt-model <ref>`      | 提供商默认值      | 设置切换模型的场景所使用的备用模型。                        |
| `--fast`                 | 关闭              | 在支持的情况下启用提供商快速模式。                                           |
| `--output-dir <path>`    | 自动生成          | 选择报告目录；相对路径基于 `--repo-root` 解析。           |
| `--repo-root <path>`     | 当前目录          | 从中立工作目录运行。                                                |
| `--sut-account <id>`     | `sut`             | 在子 Gateway 网关配置中选择 Matrix 账户 ID。                            |

Matrix QA 不会租用共享 Matrix 凭据：适配器会在本地创建
一次性用户，因此不接受 `--credential-source` 或
`--credential-role`。使用
`OPENCLAW_QA_MATRIX_TUWUNEL_IMAGE` 覆盖 homeserver 镜像；使用
`OPENCLAW_QA_MATRIX_NO_REPLY_WINDOW_MS` 调整否定性无回复断言（默认值为 `8000`，
并限制在当前场景超时时间以内）。单次命令通常会在
工件刷新后强制干净退出，因为 Matrix 加密原生句柄可能在清理后仍然存活；
仅在需要命令返回而非退出的直接测试框架中设置
`OPENCLAW_QA_MATRIX_DISABLE_FORCE_EXIT=1`。

每次运行都会在所选输出目录下写入常规 QA Lab 工件：
`qa-suite-report.md`、`qa-suite-summary.json` 和
`qa-evidence.json`。如果清理失败，请运行输出的
`docker compose ... down --remove-orphans` 恢复命令。在较慢的运行器上，
请增加无回复窗口；在快速 CI 中，较小的窗口可以缩短否定性
断言的耗时。

这些场景覆盖单元测试无法进行端到端证明的传输行为：
提及门控、允许 Bot 策略、允许列表、顶层和线程式
回复、私信路由、表情回应处理、入站编辑抑制、重启后重放去重、
homeserver 中断恢复、审批元数据交付、
媒体处理，以及 Matrix E2EE 引导启动/恢复/验证流程。
E2EE CLI 配置还会通过同一个一次性 homeserver 驱动 `openclaw matrix encryption setup`
和验证命令，然后再检查
Gateway 网关回复。

`matrix-room-block-streaming` 和 `subagent-thread-spawn` 仍可通过
显式选择 `--scenario` 使用，但不包含在默认 `all` 配置中。

CI 在
`.github/workflows/qa-live-transports-convex.yml` 中使用相同的命令界面。定时和发布运行
会执行发布场景。手动 `matrix_profile=all` 分派会扇出运行
`transport`、`media`、`e2ee-smoke`、`e2ee-deep` 和 `e2ee-cli` 配置；
重点分派会在一个作业中选择 `fast`、`release` 或 `transport`。

### Discord Mantis 场景

Discord 还提供仅限 Mantis 的可选场景，用于复现错误。使用
`--scenario discord-status-reactions-tool-only` 可测试显式状态
表情回应时间线，或使用 `--scenario discord-thread-reply-filepath-attachment`
创建真实 Discord 线程，并验证 `message.thread-reply`
是否保留 `filePath` 附件。这些场景不包含在默认
实时 Discord 通道中，因为它们是修复前后的复现探针，而不是
广泛的冒烟测试覆盖。配置了
`MANTIS_DISCORD_VIEWER_CHROME_PROFILE_DIR` 或
`MANTIS_DISCORD_VIEWER_CHROME_PROFILE_TGZ_B64` 时，线程附件 Mantis 工作流还可以在 QA
环境中添加已登录 Discord Web 的见证视频。
该查看器配置仅用于视觉捕获；通过/失败
判定仍由 Discord REST 预言机作出。

对于其他使用真实传输的冒烟测试通道：

```bash
pnpm openclaw qa discord
pnpm openclaw qa slack
pnpm openclaw qa telegram
pnpm openclaw qa whatsapp
```

它们以包含两个 Bot 或账户（驱动程序 + 被测系统）的预先存在的真实渠道为目标。
这四种传输所需的环境变量、场景列表、输出工件和 Convex
凭据池记录在下方的
[Discord、Slack、Telegram 和 WhatsApp QA 参考](#discord-slack-telegram-and-whatsapp-qa-reference)
中。

### Mantis Slack 桌面和视觉任务运行器

要运行带 VNC 救援的完整 Slack 桌面虚拟机，请运行：

```bash
pnpm openclaw qa mantis slack-desktop-smoke \
  --gateway-setup \
  --scenario slack-canary \
  --keep-lease
```

该命令会租用一台 Crabbox 桌面/浏览器机器，在虚拟机中运行 Slack 实时
通道，在 VNC 浏览器中打开 Slack Web，捕获桌面画面，并将
`slack-qa/`、`slack-desktop-smoke.png` 和
`slack-desktop-smoke.mp4`（视频捕获可用时）复制回
Mantis 工件目录。Crabbox 桌面/浏览器租约会预先提供捕获
工具和浏览器/原生构建辅助软件包，因此该场景应仅在较旧的租约上
安装后备项。Mantis 会在 `mantis-slack-desktop-smoke-report.md` 中报告总耗时和
各阶段耗时，以便在运行缓慢时显示时间是花在租约预热、凭据获取、远程设置还是
工件复制上。通过 VNC 手动登录 Slack Web 后，复用
`--lease-id <cbx_...>`；复用的租约还会使 Crabbox 的 pnpm 存储缓存
保持预热状态。默认的 `--hydrate-mode source` 从源代码检出中进行验证，并
在虚拟机内运行安装/构建。仅当复用的远程工作区已有
`node_modules` 和已构建的 `dist/` 时，才使用
`--hydrate-mode prehydrated`；该模式会跳过耗时的安装/构建步骤，并在
工作区未就绪时以失败关闭。使用 `--gateway-setup` 时，Mantis 会在虚拟机内的
端口 `38973` 上保留一个持续运行的 OpenClaw Slack Gateway 网关；不使用该选项时，该
命令会运行常规的 Bot 对 Bot Slack QA 通道，并在捕获工件后
退出。

要通过桌面证据验证原生 Slack 审批 UI，请运行 Mantis
审批检查点模式：

```bash
pnpm openclaw qa mantis slack-desktop-smoke \
  --approval-checkpoints \
  --credential-source convex \
  --credential-role maintainer
```

此模式与 `--gateway-setup` 互斥。它会运行 Slack
审批场景，拒绝非审批场景 ID，在每个待处理和已解决的审批状态等待，将观察到的 Slack API 消息渲染到
`approval-checkpoints/<scenario>-pending.png` 和
`approval-checkpoints/<scenario>-resolved.png`，然后在任何检查点、
消息证据、确认或渲染的屏幕截图缺失或为空时失败。冷启动的 CI 租约可能仍会在
`slack-desktop-smoke.png` 中显示 Slack 登录界面；审批检查点图像是此
通道的视觉证据。

默认检查点运行会保留两个标准 Slack 审批场景。
要捕获任一选择启用的 Codex 审批路由，请使用
`--scenario slack-codex-approval-exec-native` 或
`--scenario slack-codex-approval-plugin-native` 显式选择；Mantis 接受两者，并生成
相同的待处理/已解决屏幕截图对。运行器会针对每个选定的 Codex 路由延长其检查点
和远程命令截止时间，使完整的
审批、智能体完成和已解决更新序列能够完成。

操作员检查清单、GitHub 工作流分派命令、证据评论
约定、hydrate 模式决策表、耗时解读和故障
处理步骤位于
[Mantis Slack 桌面运行手册](/zh-CN/concepts/mantis-slack-desktop-runbook)。

对于智能体/CV 风格的桌面任务，请运行：

```bash
pnpm openclaw qa mantis visual-task \
  --browser-url https://example.net \
  --expect-text "Example Domain" \
  --vision-model openai/gpt-5.6-luna
```

`visual-task` 会租用或复用一台 Crabbox 桌面/浏览器机器，启动
`crabbox record --while`，通过嵌套的
`visual-driver` 驱动可见浏览器，捕获 `visual-task.png`，在选择 `--vision-mode image-describe` 时针对屏幕截图运行 `openclaw infer image
describe`，并写入
`visual-task.mp4`、`mantis-visual-task-summary.json`、
`mantis-visual-task-driver-result.json` 和
`mantis-visual-task-report.md`。设置 `--expect-text` 后，视觉
提示词会要求结构化 JSON 判定（`visible`、`evidence`、`reason`），并且仅当模型报告 `visible: true`，且证据
引用了预期文本时才会通过；仅引用
目标文本的 `visible: false` 响应仍无法通过断言。使用 `--vision-mode metadata`
可执行不调用模型的冒烟测试，在不调用图像理解提供商的情况下验证桌面、浏览器、屏幕截图和视频
管线。录制是 `visual-task` 的
必需工件；如果 Crabbox 未录制非空的
`visual-task.mp4`，即使视觉驱动器通过，任务也会失败。发生
失败时，Mantis 会保留租约以供 VNC 使用，除非任务此前已通过
且未设置 `--keep-lease`。

### 凭据池健康检查

使用池化实时凭据前，请运行：

```bash
pnpm openclaw qa credentials doctor
```

Doctor 会检查 Convex 代理环境变量（`OPENCLAW_QA_CONVEX_SITE_URL`、
`OPENCLAW_QA_CONVEX_ENDPOINT_PREFIX`），验证端点设置，仅报告
`OPENCLAW_QA_CONVEX_SECRET_CI` 和
`OPENCLAW_QA_CONVEX_SECRET_MAINTAINER` 的已设置/缺失状态，并在存在维护者密钥时验证管理/列表可达性。

## 规范场景覆盖范围

根目录的 `taxonomy.yaml` 定义语义覆盖 ID。位于
`qa/scenarios/` 下的场景 YAML 文件会将每个场景映射到这些 ID，并拥有执行
元数据：`channel` 是唯一的渠道要求，而 `profiles` 声明
具名运行成员资格。渠道驱动程序是可互换的运行级
实现选择。TypeScript
运行器会查询该目录；它们不会维护并行的场景或覆盖范围
清单。

静态 `qa coverage` 输出会报告分类法到场景的映射。实际
证据来自 `qa-evidence.json`，其中记录已执行的场景、
覆盖 ID、渠道、实际使用的驱动程序和结果。渠道和驱动程序是
报告维度，而不是额外的覆盖 ID 词汇表或场景
资格维度。

要运行不将 Docker 引入 QA 路径的一次性 Linux 虚拟机通道，请执行：

```bash
pnpm openclaw qa suite --runner multipass --scenario channel-chat-baseline
```

这会启动一个全新的 Multipass 客户机，安装依赖项，在客户机内构建 OpenClaw，
运行 `qa suite`，然后将常规 QA 报告和
摘要复制回主机上的 `.artifacts/qa-e2e/...`。它复用与主机上的
`qa suite` 相同的场景选择行为。

默认情况下，主机和 Multipass 套件运行会通过相互隔离的 Gateway 网关工作进程并行
执行多个选定场景。`qa-channel` 默认
并发数为 4，上限为选定的场景数量。使用 `--concurrency
<count>` 调整工作进程数，或使用 `--concurrency 1` 串行执行。
使用 `--pack personal-agent` 运行个人助理基准包（10 个
场景）。包选择器可与重复的 `--scenario` 标志叠加：
先运行显式场景，然后按包内顺序运行包场景，并
移除重复项。当自定义 QA 运行器已提供 OpenTelemetry 收集器设置时，使用
`--pack observability` 同时选择
`otel-trace-smoke` 和 `docker-prometheus-smoke` 场景。

任何场景失败时，该命令都会以非零状态退出。需要工件但不希望返回失败退出代码时，请使用 `--allow-failures`。

实时运行会转发适合客户机使用的受支持 QA 身份验证输入：
基于环境变量的提供商密钥、QA 实时提供商配置路径，以及存在时的
`CODEX_HOME`。将 `--output-dir` 保留在仓库根目录下，以便
客户机可通过挂载的工作区写回。

## Discord、Slack、Telegram 和 WhatsApp QA 参考

Matrix 适配器使用上文记录的、由 Docker 支持的一次性通道。
Discord、Slack、Telegram 和 WhatsApp 针对预先存在的真实
传输运行，因此其参考信息位于此处。

### 共享 CLI 标志

这些通道通过
`extensions/qa-lab/src/live-transports/shared/live-transport-cli.ts` 注册，并
接受相同的标志：

| 标志                                  | 默认值                                            | 描述                                                                                                                                     |
| ------------------------------------- | -------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------- |
| `--scenario <id>`                     | -                                                  | 仅运行此场景。可重复使用。                                                                                                             |
| `--output-dir <path>`                 | `<repo>/.artifacts/qa-e2e/<transport>-<timestamp>` | 写入报告、摘要、证据、传输专用工件和输出日志的位置。相对路径以 `--repo-root` 为基准解析。 |
| `--repo-root <path>`                  | `process.cwd()`                                    | 从中立的当前工作目录调用时使用的仓库根目录。                                                                                               |
| `--sut-account <id>`                  | `sut`                                              | QA Gateway 网关配置中的临时账户 ID。                                                                                              |
| `--provider-mode <mode>`              | `live-frontier`                                    | `mock-openai`、`aimock` 或 `live-frontier`。                                                                                                    |
| `--model <ref>` / `--alt-model <ref>` | 提供商默认值                                   | 主要/备用模型引用。                                                                                                                   |
| `--fast`                              | 关闭                                                | 支持时使用提供商快速模式。                                                                                                             |
| `--credential-source <env\|convex>`   | `env`                                              | 请参阅 [Convex 凭据池](#convex-credential-pool)。                                                                                          |
| `--credential-role <maintainer\|ci>`  | CI 中为 `ci`，否则为 `maintainer`                 | `--credential-source convex` 时使用的角色。                                                                                                    |
| `--allow-failures`                    | 关闭                                                | 场景失败时写入工件，但不返回失败退出代码。                                                                      |

任何场景失败时，各通道都会以非零状态退出。`--allow-failures` 会写入
工件，但不设置失败退出代码。Telegram 还接受
`--list-scenarios`，用于打印可用场景 ID 并退出；其他通道
不提供该标志。

### Telegram QA

```bash
pnpm openclaw qa telegram
```

目标是一个真实的私有 Telegram 群组，其中包含两个不同的 Bot（驱动程序 +
被测系统）。被测系统 Bot 必须拥有 Telegram 用户名；当两个 Bot 都在
`@BotFather` 中启用 **Bot-to-Bot Communication Mode** 时，Bot 对 Bot 观察
效果最佳。

使用 `--credential-source env` 时所需的环境变量：

- `OPENCLAW_QA_TELEGRAM_GROUP_ID` - 数字聊天 ID（字符串）。
- `OPENCLAW_QA_TELEGRAM_DRIVER_BOT_TOKEN`
- `OPENCLAW_QA_TELEGRAM_SUT_BOT_TOKEN`

`release` 配置文件会选择维护中的 Telegram YAML 场景；`all`
会添加选择启用的会话、用量、回复链和流式传输压力检查。显式的
`--scenario` 值会覆盖该配置文件。

- `channel-canary`
- `channel-mention-gating`
- `telegram-help-command`
- `telegram-commands-command`
- `telegram-tools-compact-command`
- `telegram-whoami-command`
- `telegram-status-command`
- `telegram-repeated-command-authorization`
- `telegram-other-bot-command-gating`
- `telegram-context-command`
- `telegram-current-session-status-tool`
- `telegram-tool-only-usage-footer`
- `telegram-reply-chain-exact-marker`
- `telegram-stream-final-single-message`
- `telegram-long-final-reuses-preview`
- `telegram-long-final-three-chunks`

`release` 配置文件始终涵盖金丝雀测试、提及门控、原生命令
回复、命令寻址，以及 Bot 间的群组回复。`mock-openai`
还包括确定性的长篇最终预览检查。
`telegram-current-session-status-tool` 和
`telegram-tool-only-usage-footer` 仍需选择启用：前者仅在紧接金丝雀测试之后按线程运行时
才稳定，后者则通过真实 Telegram 验证仅含工具调用的回复中
`/usage` 页脚。使用 `pnpm openclaw qa telegram
--list-scenarios --provider-mode mock-openai` 可打印当前
默认/可选划分及回归引用。每个
Telegram 实时适配器场景都使用 `--profile all`。

输出工件：

- `qa-suite-report.md`
- `qa-suite-summary.json`
- `qa-evidence.json` - 实时传输检查的证据条目，
  包括配置文件、覆盖范围、提供商、渠道、工件、结果和 RTT
  字段。

软件包 Telegram 运行使用相同的 Telegram 凭据约定。重复 RTT
测量是常规软件包 Telegram 实时通道的一部分；RTT
分布会针对所选 RTT 检查归入 `qa-evidence.json` 的 `result.timing` 下。

```bash
OPENCLAW_QA_CREDENTIAL_SOURCE=convex \
pnpm test:docker:npm-telegram-live
```

设置 `OPENCLAW_QA_CREDENTIAL_SOURCE=convex` 后，软件包实时包装器会
租用 `kind: "telegram"` 凭据，将租用的群组/驱动端/SUT
Bot 环境变量导出到已安装软件包的运行中，为租约发送 Heartbeat，并在
关闭时释放租约。软件包包装器默认执行 20 次
`channel-canary` RTT 检查，RTT 超时为 30s；选择 Convex 时，在 CI
之外默认使用 Convex 角色 `maintainer`。可覆盖
`OPENCLAW_NPM_TELEGRAM_RTT_SAMPLES`、`OPENCLAW_NPM_TELEGRAM_RTT_TIMEOUT_MS`
或 `OPENCLAW_NPM_TELEGRAM_RTT_MAX_FAILURES` 来调整 RTT 测量，而无需
创建单独的 RTT 命令或 Telegram 专用摘要格式。

### Discord QA

```bash
pnpm openclaw qa discord
```

目标是一个真实的私有 Discord 服务器频道，其中包含两个 Bot：一个由
测试框架控制的驱动 Bot，以及一个由子 OpenClaw Gateway 网关通过内置
Discord 插件启动的 SUT Bot。验证频道提及处理、SUT Bot
是否已向 Discord 注册原生 `/help` 命令，以及
选择启用的 Mantis 证据场景。

`--credential-source env` 时必需的环境变量：

- `OPENCLAW_QA_DISCORD_GUILD_ID`
- `OPENCLAW_QA_DISCORD_CHANNEL_ID`
- `OPENCLAW_QA_DISCORD_DRIVER_BOT_TOKEN`
- `OPENCLAW_QA_DISCORD_SUT_BOT_TOKEN`
- `OPENCLAW_QA_DISCORD_SUT_APPLICATION_ID` - 必须与 Discord 返回的 SUT Bot 用户 ID
  匹配（否则该通道会快速失败）。

可选：

- `OPENCLAW_QA_DISCORD_VOICE_CHANNEL_ID` 为
  `discord-voice-autojoin` 选择语音/舞台频道；未设置时，该场景会为 SUT Bot
  选择第一个可见的语音/舞台频道。

Discord YAML 模块场景（`qa/scenarios/channels/discord-*.yaml`）：

- `discord-canary`
- `discord-mention-gating`
- `discord-native-help-command-registration`
- `discord-voice-autojoin` - 选择启用的语音场景。单独运行，启用
  `channels.discord.voice.autoJoin`，并验证 SUT Bot 当前的
  Discord 语音状态是否为目标语音/舞台频道。Convex Discord
  凭据可以包含可选的 `voiceChannelId`；否则，运行器
  适配器会发现服务器中第一个可见的语音/舞台频道。
- `discord-status-reactions-tool-only` - 选择启用的 Mantis 场景。该场景
  单独运行，因为它使用 `messages.statusReactions.enabled=true` 将 SUT 切换为始终开启、仅含工具调用的服务器回复，
  随后捕获 REST
  表情回应时间线以及 HTML/PNG 可视化工件。Mantis 前后
  报告还会将场景提供的 MP4 工件保留为 `baseline.mp4`
  和 `candidate.mp4`。
- `discord-thread-reply-filepath-attachment` - 选择启用的 Mantis 场景；请参阅
  [Discord Mantis 场景](#discord-mantis-scenarios)。

显式运行 Discord 语音自动加入场景：

```bash
pnpm openclaw qa discord \
  --scenario discord-voice-autojoin \
  --provider-mode mock-openai
```

显式运行 Mantis 状态表情回应场景：

```bash
pnpm openclaw qa discord \
  --scenario discord-status-reactions-tool-only \
  --provider-mode live-frontier \
  --model openai/gpt-5.6-luna \
  --alt-model openai/gpt-5.6-luna \
  --fast
```

输出工件：

- `qa-suite-report.md`
- `qa-suite-summary.json`
- `qa-evidence.json` - 实时传输检查的证据条目。
- `discord-qa-reaction-timelines.json` 和
  `discord-status-reactions-tool-only-timeline.png`，仅在状态表情回应
  场景运行时生成。

### Slack QA

```bash
pnpm openclaw qa slack
```

目标是一个真实的私有 Slack 频道，其中包含两个不同的 Bot：一个由测试框架
控制的驱动 Bot，以及一个由子 OpenClaw Gateway 网关通过内置
Slack 插件启动的 SUT Bot。

`--credential-source env` 时必需的环境变量：

- `OPENCLAW_QA_SLACK_CHANNEL_ID`
- `OPENCLAW_QA_SLACK_DRIVER_BOT_TOKEN`
- `OPENCLAW_QA_SLACK_SUT_BOT_TOKEN`
- `OPENCLAW_QA_SLACK_SUT_APP_TOKEN`

可选：

- `OPENCLAW_QA_SLACK_APPROVAL_CHECKPOINT_DIR` 为 Mantis 启用可视化审批
  检查点。适配器写入 `<scenario>.pending.json` 和
  `<scenario>.resolved.json`，随后等待匹配的 `.ack.json` 文件。
- `OPENCLAW_QA_SLACK_APPROVAL_CHECKPOINT_TIMEOUT_MS` 覆盖检查点
  确认超时。默认值为 `120000`。

通过 Slack 实时适配器公开的规范 YAML 场景：

- `thread-follow-up`
- `thread-isolation`

Slack YAML 模块场景（`qa/scenarios/channels/slack-*.yaml`）：

- `slack-canary`
- `slack-mention-gating`
- `slack-allowlist-block`
- `slack-channel-disabled-warning` - 选择启用的真实 Slack 探测，确认
  配置为禁用的频道会发出结构化警告，而不会回复。
- `slack-top-level-reply-shape`
- `slack-restart-resume`
- `slack-progress-commentary-true`、`slack-progress-commentary-false`、
  `slack-progress-commentary-omitted` 和
  `slack-progress-commentary-verbose-dedupe` - 选择启用的真实 Slack 探测，用于验证
  相互独立的评论/工具进度控制、缺省键的旧版默认行为，以及启用持久详细进度时的
  单次投递行为。
- `slack-reaction-glyph-native` - 选择启用的实时消息工具表情回应场景。
  指示智能体传递确切的 `✅` 字形，并确认 Slack 在目标消息上
  为 SUT Bot 存储了 `white_check_mark`。
- `slack-chart-presentation-native` - 选择启用的可移植图表场景，
  验证原生 `data_visualization` 块和完全一致的无障碍文本。
- `slack-table-presentation-native` - 选择启用的可移植表格场景，
  验证原生 `data_table` 块、完全一致的行和无障碍文本。
- `slack-table-invalid-blocks-fallback` - 选择启用的直接传输场景，
  通过生产 Slack 发送路径发送一张结构上可读取、超出限制的原始表格，其中包含
  101 个数据行及其表头；证明 Slack 本身返回 `invalid_blocks`，
  并验证已存储的禁用格式回退内容完整，且不含
  原生数据块。场景详情仅保留安全的错误代码、计数和
  布尔值证据。
- `slack-approval-exec-native` - 选择启用的原生 Slack Exec 审批场景。
  通过 Gateway 网关请求 Exec 审批，验证 Slack 消息
  包含原生审批按钮，完成审批，并验证审批完成后的 Slack
  更新。
- `slack-approval-plugin-native` - 选择启用的原生 Slack 插件审批
  场景。同时启用 Exec 和插件审批转发，避免插件
  事件被 Exec 审批路由抑制，随后验证相同的
  待处理/已完成原生 Slack UI 路径。
- `slack-codex-approval-exec-native` - 选择启用的 Codex Guardian 命令审批
  场景。以 Guardian 模式启用 Codex 插件，将源自
  Slack 的 Gateway 网关智能体轮次通过 Codex app-server 测试框架进行路由，
  等待 `openclaw-codex-app-server` 的原生 Slack 插件审批提示，
  完成审批，并验证 Codex 轮次
  以预期的命令输出和助手标记结束。
- `slack-codex-approval-plugin-native` - 选择启用的 Codex Guardian 文件审批
  场景。使用工作区外部的 `apply_patch` 指令，使 Codex 发出
  app-server 文件更改审批路由，随后验证相同的原生
  Slack 待处理/已完成审批路径、最终助手标记，以及清理前完全一致的文件
  内容。

Codex 审批场景需要 `openai/*` 或 `codex/*` `--model`、
常规实时模型凭据，以及 Codex 插件接受的 Codex 身份验证或 API 密钥身份验证。
场景详情包括 Codex app-server 方法、所选 Codex 模型
键、最终 Codex 轮次状态和操作标记验证，以及
经过删减的 Slack 审批元数据。

输出工件：

- `qa-suite-report.md`
- `qa-suite-summary.json`
- `qa-evidence.json` - 实时传输检查的证据条目。
- `approval-checkpoints/` - 仅在 Mantis 设置
  `OPENCLAW_QA_SLACK_APPROVAL_CHECKPOINT_DIR` 时生成；包含检查点 JSON、
  确认 JSON，以及待处理/已完成截图。

#### 设置 Slack 工作区

该通道需要同一工作区中的两个不同 Slack 应用，以及一个两个
Bot 都已加入的频道：

- `channelId` - 两个 Bot 均已受邀加入的频道的 `Cxxxxxxxxxx` ID。
  请使用专用频道；该通道每次运行都会发布消息。
- `driverBotToken` - **驱动端**应用的 Bot 令牌（`xoxb-...`）。
- `sutBotToken` - **SUT** 应用的 Bot 令牌（`xoxb-...`）；该应用必须是
  与驱动端不同的 Slack 应用，以确保其 Bot 用户 ID 不同。
- `sutAppToken` - SUT 应用具有
  `connections:write` 的应用级令牌（`xapp-...`），Socket Mode 使用该令牌让 SUT 应用接收事件。

建议使用专用于 QA 的 Slack 工作区，而不是复用生产
工作区。

下面的 SUT 清单有意将内置 Slack 插件的
生产安装（`extensions/slack/src/setup-shared.ts:12`）缩小到
实时 Slack QA 套件覆盖的权限和事件。有关用户所见的
生产频道设置，请参阅
[Slack 频道快速设置](/zh-CN/channels/slack#quick-setup)；QA 驱动端/SUT
组合有意分开，因为该通道需要同一工作区中的两个不同 Bot 用户
ID。

**1. 创建驱动端应用**

前往 [api.slack.com/apps](https://api.slack.com/apps) → _Create New App_ →
_From a manifest_ → 选择 QA 工作区，粘贴以下清单，
然后选择 _Install to Workspace_：

```json
{
  "display_information": {
    "name": "OpenClaw QA Driver",
    "description": "Test driver bot for OpenClaw QA Slack live lane"
  },
  "features": {
    "bot_user": {
      "display_name": "OpenClaw QA Driver",
      "always_online": true
    }
  },
  "oauth_config": {
    "scopes": {
      "bot": ["chat:write", "channels:history", "groups:history", "users:read"]
    }
  },
  "settings": {
    "socket_mode_enabled": false
  }
}
```

复制 _Bot User OAuth Token_（`xoxb-...`）——它将成为
`driverBotToken`。驱动端只需发布消息并标识
自身；不需要事件，也不需要 Socket Mode。

**2. 创建 SUT 应用**

在同一工作区中重复执行 _Create New App → From a manifest_。此 QA 应用
有意使用内置 Slack 插件生产清单（`extensions/slack/src/setup-shared.ts:12`）的较精简版本：由于实时 Slack QA 套件尚未覆盖
表情回应处理，因此省略了表情回应
权限范围和事件。

```json
{
  "display_information": {
    "name": "OpenClaw QA SUT",
    "description": "用于 OpenClaw 的 OpenClaw QA SUT 连接器"
  },
  "features": {
    "bot_user": {
      "display_name": "OpenClaw QA SUT",
      "always_online": true
    },
    "app_home": {
      "home_tab_enabled": true,
      "messages_tab_enabled": true,
      "messages_tab_read_only_enabled": false
    }
  },
  "oauth_config": {
    "scopes": {
      "bot": [
        "app_mentions:read",
        "assistant:write",
        "channels:history",
        "channels:read",
        "chat:write",
        "commands",
        "emoji:read",
        "files:read",
        "files:write",
        "groups:history",
        "groups:read",
        "im:history",
        "im:read",
        "im:write",
        "mpim:history",
        "mpim:read",
        "mpim:write",
        "pins:read",
        "pins:write",
        "usergroups:read",
        "users:read"
      ]
    }
  },
  "settings": {
    "socket_mode_enabled": true,
    "event_subscriptions": {
      "bot_events": [
        "app_home_opened",
        "app_mention",
        "channel_rename",
        "member_joined_channel",
        "member_left_channel",
        "message.channels",
        "message.groups",
        "message.im",
        "message.mpim",
        "pin_added",
        "pin_removed"
      ]
    }
  }
}
```

Slack 创建应用后，在其设置页面执行两项操作：

- _Install to Workspace_ → 复制 _Bot User OAuth Token_ → 它将成为
  `sutBotToken`。
- _Basic Information → App-Level Tokens → Generate Token and Scopes_ → 添加
  权限范围 `connections:write` → 保存 → 复制 `xapp-...` 的值 → 它将
  成为 `sutAppToken`。

分别使用两个令牌调用 `auth.test`，验证两个 Bot 的用户 ID 不同。
运行时通过用户 ID 区分驱动端和 SUT；如果二者复用同一个应用，
提及门控将立即失败。

**3. 创建频道**

在 QA 工作区中创建一个频道（例如 `#openclaw-qa`），然后在频道内邀请两个
Bot：

```text
/invite @OpenClaw QA Driver
/invite @OpenClaw QA SUT
```

从 _channel info → About → Channel ID_ 复制 `Cxxxxxxxxxx` ID——它将
成为 `channelId`。公共频道可以正常使用；如果使用私有频道，
两个应用已拥有 `groups:history`，因此测试框架仍能成功读取历史记录。

**4. 注册凭据**

有两种方式。单机调试时使用环境变量（设置四个
`OPENCLAW_QA_SLACK_*` 变量并传入 `--credential-source env`），或者为
共享 Convex 池填充数据，以便 CI 和其他维护者租用这些凭据。

对于 Convex 池，将以下四个字段写入 JSON 文件：

```json
{
  "channelId": "Cxxxxxxxxxx",
  "driverBotToken": "xoxb-...",
  "sutBotToken": "xoxb-...",
  "sutAppToken": "xapp-..."
}
```

在 shell 中导出 `OPENCLAW_QA_CONVEX_SITE_URL` 和 `OPENCLAW_QA_CONVEX_SECRET_MAINTAINER`
后，执行注册和验证：

```bash
pnpm openclaw qa credentials add \
  --kind slack \
  --payload-file slack-creds.json \
  --note "QA Slack pool seed"

pnpm openclaw qa credentials list --kind slack --status all --json
```

预期看到 `count: 1`、`status: "active"`，且没有 `lease` 字段。

**5. 验证端到端流程**

在本地运行该测试通道，确认两个 Bot 可以通过代理相互通信：

```bash
pnpm openclaw qa slack \
  --credential-source convex \
  --credential-role maintainer \
  --output-dir .artifacts/qa-e2e/slack-local
```

成功运行会在远低于 30 秒内完成，并且 `qa-suite-report.md`
会显示 `slack-canary` 和 `slack-mention-gating` 的状态均为 `pass`。如果该
测试通道挂起约 90 秒后以 `Convex credential pool exhausted
for kind "slack"` 退出，则表示池为空或所有记录均已出租——
`qa
credentials list --kind slack --status all --json` 会说明具体是哪种情况。

### WhatsApp QA

```bash
pnpm openclaw qa whatsapp
```

以两个专用 WhatsApp Web 账号为目标：一个由测试框架控制的驱动账号，
以及一个由子 OpenClaw Gateway 网关通过内置 WhatsApp 插件启动的 SUT 账号。

使用 `--credential-source env` 时所需的环境变量：

- `OPENCLAW_QA_WHATSAPP_DRIVER_PHONE_E164`
- `OPENCLAW_QA_WHATSAPP_SUT_PHONE_E164`
- `OPENCLAW_QA_WHATSAPP_DRIVER_AUTH_ARCHIVE_BASE64`
- `OPENCLAW_QA_WHATSAPP_SUT_AUTH_ARCHIVE_BASE64`

可选：

- `OPENCLAW_QA_WHATSAPP_GROUP_JID` 可启用群组场景，例如
  `whatsapp-mention-gating`、`whatsapp-group-pending-history-context`、
  `whatsapp-broadcast-group-fanout`、`whatsapp-group-activation-always`、
  `whatsapp-group-reply-to-bot-triggers`、群组操作/媒体/投票场景
  以及 `whatsapp-group-allowlist-block`。

WhatsApp YAML 场景（`qa/scenarios/channels/whatsapp-*.yaml`）：

- 基线与群组门控：`whatsapp-canary`、`whatsapp-pairing-block`、
  `whatsapp-mention-gating`、`whatsapp-group-pending-history-context`、
  `whatsapp-group-activation-always`、`whatsapp-group-reply-to-bot-triggers`、
  `whatsapp-top-level-reply-shape`、`whatsapp-restart-resume`、
  `whatsapp-group-allowlist-block`。
- 原生命令：`whatsapp-help-command`、`whatsapp-status-command`、
  `whatsapp-commands-command`、`whatsapp-tools-compact-command`、
  `whatsapp-whoami-command`、`whatsapp-context-command`、
  `whatsapp-native-new-command`。
- 回复和最终输出行为：`whatsapp-tool-only-usage-footer`、
  `whatsapp-reply-to-message`、`whatsapp-group-reply-to-message`、
  `whatsapp-reply-to-mode-batched`、`whatsapp-reply-context-isolation`、
  `whatsapp-reply-delivery-shape`、`whatsapp-stream-final-message-accounting`。
- 用户路径消息操作：`whatsapp-agent-message-action-react` 从真实的驱动端私信开始，
  让模型调用 `message` 工具，并观察 WhatsApp 原生表情回应。
  `whatsapp-agent-message-action-upload-file` 对 `message(action=upload-file)` 采用相同方式，并观察
  WhatsApp 原生媒体。`whatsapp-group-agent-message-action-react` 和
  `whatsapp-group-agent-message-action-upload-file` 在真实 WhatsApp 群组中验证相同的
  用户可见操作。
- 群组扇出：`whatsapp-broadcast-group-fanout` 从一条提及机器人的
  WhatsApp 群组消息开始，并验证来自 `main`
  和 `qa-second` 的不同可见回复。
- 群组激活：`whatsapp-group-activation-always` 将真实群组
  会话更改为 `/activation always`，验证未提及机器人的群组消息会唤醒
  智能体，然后恢复为 `/activation mention`。
  `whatsapp-group-reply-to-bot-triggers` 先生成一条 Bot 回复，再在没有显式提及的情况下
  发送对它的原生引用回复，并验证智能体会从该回复上下文中
  被唤醒。
- 入站媒体和结构化消息：`whatsapp-inbound-image-caption`、
  `whatsapp-audio-preflight`、`whatsapp-inbound-structured-messages`、
  `whatsapp-group-audio-gating`、`whatsapp-inbound-reaction-no-trigger`。
  这些场景通过驱动端发送真实的 WhatsApp 图像、音频、文档、位置、联系人、
  贴纸和表情回应事件。
- 直接 Gateway 网关契约探测：`whatsapp-outbound-media-matrix`、
  `whatsapp-outbound-document-preserves-filename`、`whatsapp-outbound-poll`、
  `whatsapp-outbound-send-serialization`、
  `whatsapp-group-outbound-media`、`whatsapp-group-outbound-poll`、
  `whatsapp-message-actions`、`whatsapp-reply-context-isolation`、
  `whatsapp-reply-delivery-shape`。这些场景有意绕过模型提示，
  并验证确定性的 Gateway 网关/渠道 `send`、`poll` 和
  `message.action` 契约。
- 访问控制覆盖：`whatsapp-access-control-dm-open`、
  `whatsapp-access-control-dm-disabled`、`whatsapp-access-control-group-open`、
  `whatsapp-access-control-group-disabled`、`whatsapp-group-allowlist-block`。
- 原生审批：`whatsapp-approval-exec-deny-native`、
  `whatsapp-approval-exec-native`、`whatsapp-approval-exec-reaction-native`、
  `whatsapp-approval-exec-group-reaction-native`、
  `whatsapp-approval-plugin-native`。
- 状态表情回应：`whatsapp-status-reactions`、
  `whatsapp-status-reaction-lifecycle`。

目录目前包含 52 个场景。`live-frontier` 默认测试通道保持精简，
仅运行 8 个场景，以提供快速冒烟覆盖。`mock-openai`
默认测试通道通过真实 WhatsApp 传输确定性地运行 39 个场景，
仅模拟模型输出；审批场景以及少数更重或会阻塞的检查仍需通过场景 ID 显式运行。

WhatsApp QA 驱动端会观察结构化实时事件（`text`、`media`、
`location`、`reaction` 和 `poll`），并可主动发送媒体、投票、
联系人、位置和贴纸。QA Lab 通过
`@openclaw/whatsapp/api.js` 包公开接口导入该驱动端，而不访问私有的
WhatsApp 运行时文件。观察群组事件时，`fromJid` 是群组 JID，
而 `participantJid` 和 `fromPhoneE164` 用于标识参与者发送者。
消息内容默认经过遮盖。直接 Gateway 网关投票、文件上传、
媒体、群组投票、群组媒体和回复形态探测属于传输/API
契约检查；它们不能证明用户提示会使
智能体选择相同操作。用户路径操作的证据来自
`whatsapp-agent-message-action-react` 和
`whatsapp-group-agent-message-action-react` 等场景，其中驱动端发送普通的
WhatsApp 消息，QA Lab 则观察由此产生的 WhatsApp 原生工件。
WhatsApp 场景详情包含每个场景的验证方式（`user-path`、
`direct-gateway` 或 `native-approval`），防止将证据误认为
它实际未能证明的更强契约。

输出工件：

- `qa-suite-report.md`
- `qa-suite-summary.json`
- `qa-evidence.json`——实时传输检查的证据条目。

### Convex 凭据池

Discord、Slack、Telegram 和 WhatsApp 测试通道可以从
共享 Convex 池租用凭据，而不读取上述环境变量。传入
`--credential-source convex`（或设置 `OPENCLAW_QA_CREDENTIAL_SOURCE=convex`）；
QA Lab 会获取独占租约，在运行期间持续发送心跳，并在关闭时释放租约。
池类型为 `"discord"`、`"slack"`、
`"telegram"` 和 `"whatsapp"`。

代理在 `admin/add` 上验证的负载形态：

- Discord（`kind: "discord"`）：`{ guildId: string, channelId: string,
driverBotToken: string, sutBotToken: string, sutApplicationId: string }`。
- Telegram（`kind: "telegram"`）：`{ groupId: string, driverToken: string,
sutToken: string }`——`groupId` 必须是数字聊天 ID 字符串。
- Telegram 真实用户（`kind: "telegram-user"`）：`{ groupId: string, sutToken:
string, testerUserId: string, testerUsername: string, telegramApiId:
string, telegramApiHash: string, tdlibDatabaseEncryptionKey: string,
tdlibArchiveBase64: string, tdlibArchiveSha256: string,
desktopTdataArchiveBase64: string, desktopTdataArchiveSha256: string }`——
  仅用于 Mantis Telegram Desktop 验证。通用 QA Lab 测试通道不得获取
  此类型。
- WhatsApp（`kind: "whatsapp"`）：`{ driverPhoneE164: string, sutPhoneE164:
string, driverAuthArchiveBase64: string, sutAuthArchiveBase64: string,
groupJid?: string }`——电话号码必须是不同的 E.164 字符串。

Mantis Telegram Desktop 验证工作流会为 TDLib CLI 驱动端和 Telegram Desktop
见证端持有同一个独占 Convex `telegram-user` 租约，
然后在发布证据后释放它。

当 PR 需要确定性的视觉差异时，Mantis 可以在 `main` 和 PR
头部提交上使用相同的模拟模型回复，同时更改 Telegram 格式化器或
交付层。捕获默认值针对 PR 评论进行了调整：标准
Crabbox 规格、24fps 桌面录制、24fps 动态 GIF，以及 1920px 预览
宽度。前后对比评论应发布一个仅包含
预期 GIF 的整洁工件包。

Slack 测试通道也可以使用该池。Slack 负载形态检查目前位于
Slack QA 运行器中，而不是代理中；请使用 `{ channelId: string,
driverBotToken: string, sutBotToken: string, sutAppToken: string }`，并指定类似
`Cxxxxxxxxxx` 的 Slack 频道 ID。有关应用
和权限范围配置，请参阅[设置 Slack 工作区](#setting-up-the-slack-workspace)。

运行环境变量和 Convex 代理端点契约位于
[测试 → 通过 Convex 共享 Telegram 凭据](/zh-CN/help/testing#shared-telegram-credentials-via-convex-v1)
（该章节名称早于多渠道池；各类型共享相同的租约语义）。

## 仓库支持的种子数据

种子资源位于 `qa/`：

- `qa/scenarios/index.yaml`
- `qa/scenarios/<theme>/*.yaml`

这些内容有意存储在 Git 中，以便人和
智能体都能看到 QA 计划。

`qa-lab` 保持为通用 YAML 场景运行器。每个场景 YAML 文件都是
单次测试运行的事实来源，并应定义：

- 顶层 `title`
- `scenario` 元数据
- `scenario` 中可选的类别、能力、测试通道和风险元数据
- `scenario` 中的文档和代码引用
- `scenario` 中可选的插件要求
- `scenario` 中可选的 Gateway 网关配置补丁
- 流程场景的可执行顶层 `flow`，或者
  Vitest 和 Playwright 场景的 `scenario.execution.kind` / `scenario.execution.path`

支持 `flow` 的可复用运行时表面保持通用且
横跨多个领域。例如，YAML 场景可以将传输端
辅助工具与浏览器端辅助工具结合起来，后者通过
Gateway 网关 `browser.request` 接缝驱动嵌入式 Control UI，而无需添加特殊情况运行器。

场景文件应按产品能力分组，而不是按源代码
树文件夹分组。移动文件时应保持场景 ID 稳定；使用 `docsRefs` 和
`codeRefs` 实现可追溯性。

基线列表应足够广泛，至少涵盖：

- 私信和渠道聊天
- 话题行为
- 消息操作生命周期
- 定时任务回调
- 记忆召回
- 模型切换
- 子智能体交接
- 仓库阅读和文档阅读
- 一个小型构建任务，例如 Lobster Invaders

## 提供商模拟通道

`qa suite` 有两个本地提供商模拟通道：

- `mock-openai` 是场景感知型 OpenClaw 模拟。它仍是
基于仓库的 QA 和一致性门禁所用的默认确定性模拟通道。
- `aimock` 启动一个由 AIMock 支持的提供商服务器，用于实验性
协议、夹具、录制/重放和混沌测试覆盖。它是增量补充，
不会取代 `mock-openai` 场景分派器。

提供商通道实现位于 `extensions/qa-lab/src/providers/` 下。
每个提供商负责自身的默认值、本地服务器启动、Gateway 网关模型配置、
身份验证配置文件暂存需求，以及实时/模拟能力标志。共享套件和
Gateway 网关代码通过提供商注册表进行路由，而不是根据
提供商名称进行分支。

## 传输适配器

`qa-lab` 为 YAML QA 场景提供通用传输接缝。`qa-channel` 是
合成默认项。`crabline` 启动具有本地提供商形式的服务器，并
针对它们运行 OpenClaw 的常规渠道插件。`live` 保留用于
真实提供商凭据和外部渠道。

在架构层面，职责划分如下：

- `qa-lab` 负责通用场景执行、工作进程并发、工件
  写入和报告。
- 传输适配器负责 Gateway 网关配置、就绪状态、入站和出站
  观测、传输操作及规范化传输状态。
- `qa/scenarios/` 下的 YAML 场景文件定义测试运行；`qa-lab`
  提供执行这些场景的可复用运行时表面。

### 添加渠道

将渠道添加到 YAML QA 系统时，需要提供渠道实现，
并添加用于检验渠道契约的场景包。对于冒烟 CI
覆盖，请添加匹配的 Crabline 本地提供商服务器，并通过
`crabline` 驱动程序公开该服务器。

当共享 `qa-lab` 宿主能够负责该流程时，不要添加新的顶层 QA 命令根。

`qa-lab` 负责共享宿主机制：

- `openclaw qa` 命令根
- 套件启动和拆卸
- 工作进程并发
- 工件写入
- 报告生成
- 场景执行
- 旧版 `qa-channel` 场景的兼容性别名

运行器插件负责传输契约：

- 如何在共享 `qa` 根下挂载 `openclaw qa <runner>`
- 如何为该传输配置 Gateway 网关
- 如何检查就绪状态
- 如何注入入站事件
- 如何观测出站消息
- 如何公开转录记录和规范化传输状态
- 如何执行由传输支持的操作
- 如何处理传输专用的重置或清理

新渠道的最低采用标准：

1. 继续由 `qa-lab` 负责共享 `qa` 根。
2. 在共享 `qa-lab` 宿主接缝上实现传输运行器。
3. 将传输专用机制保留在运行器插件或渠道
   测试框架中。
4. 将运行器挂载为 `openclaw qa <runner>`，而不是注册一个
   与之竞争的根命令。运行器插件应在
   `openclaw.plugin.json` 中声明 `qaRunners`，并从 `runtime-api.ts` 导出匹配的
   `qaRunnerCliRegistrations` 数组。保持 `runtime-api.ts` 轻量；延迟加载 CLI 和
   运行器执行应位于不同入口点之后。可选的
   `adapterFactory` 可将传输公开给共享场景，而不更改
   命令现有的场景目录。除非工厂声明每个实例都拥有隔离的凭据或
   一次性服务器、Gateway 网关状态和工件路径，否则同一渠道的分区应串行运行。
5. 在按主题组织的 `qa/scenarios/`
   目录下编写或调整 YAML 场景。
6. 新场景应使用通用场景辅助工具。
7. 除非仓库正在进行有意迁移，否则应保持现有
   兼容性别名正常工作。

决策规则非常严格：

- 如果某项行为可以在 `qa-lab` 中统一表达一次，请将其放入 `qa-lab`。
- 如果某项行为依赖于一个渠道传输，请将其保留在对应的运行器
  插件或插件测试框架中。
- 如果某个场景需要一种可供多个渠道使用的新能力，
  应添加通用辅助工具，而不是在 `suite.ts` 中添加渠道专用分支。
- 如果某项行为只对一种传输有意义，请保持该场景
  特定于传输，并在场景契约中明确说明。

### 场景辅助工具名称

新场景首选使用以下通用辅助工具：

- `waitForTransportReady`
- `waitForChannelReady`
- `injectInboundMessage`
- `injectOutboundMessage`
- `waitForTransportOutboundMessage`
- `waitForChannelOutboundMessage`
- `waitForNoTransportOutbound`
- `getTransportSnapshot`
- `readTransportMessage`
- `readTransportTranscript`
- `formatTransportTranscript`
- `resetTransport`

兼容性别名仍可供现有场景使用：
`waitForQaChannelReady`、`waitForOutboundMessage`、`waitForNoOutbound`、
`formatConversationTranscript`、`resetBus`，但编写新场景时
应使用通用名称。保留这些别名是为了避免一次性
迁移，而不是作为未来采用的模型。

## 报告

`qa-lab` 根据观测到的总线时间线导出 Markdown 协议报告。
报告应回答：

- 哪些功能正常
- 哪些功能失败
- 哪些功能仍被阻塞
- 哪些后续场景值得添加

要查看可用场景清单（适用于评估后续工作规模
或接入新传输），请运行 `pnpm openclaw qa coverage`（添加 `--json`
可获取机器可读输出）。为已触及的
行为或文件路径选择针对性验证时，请运行 `pnpm openclaw qa coverage --match <query>`。
匹配报告会搜索场景元数据、文档引用、代码引用、覆盖范围 ID、
插件和提供商要求，然后输出匹配的 `qa suite
--scenario ...` 目标。

每次 `qa suite` 运行都会为所选
场景集写入顶层 `qa-evidence.json`、
`qa-suite-summary.json` 和 `qa-suite-report.md` 工件。声明了 `execution.kind: vitest` 或
`execution.kind: playwright` 的场景会运行匹配的测试路径，并写入
每个场景的日志。声明了 `execution.kind: script` 的场景会通过
`node --import tsx` 运行位于 `execution.path` 的证据生成器（其中
`${outputDir}` 和 `${scenarioId}` 会在 `execution.args` 中展开）；
生成器会写入自己的 `qa-evidence.json`，其中的条目将导入
套件输出，其工件路径相对于该生成器
`qa-evidence.json` 进行解析。当通过 `qa run
--qa-profile` 到达 `qa suite` 时，同一 `qa-evidence.json` 还会包含所选分类类别的配置文件
评分卡摘要。

应将覆盖范围输出视为发现辅助信息，而不是门禁替代品；
所选场景仍需针对被测行为采用正确的提供商模式、实时传输、
Multipass、Testbox 或发布通道。有关
评分卡的背景信息，请参阅[成熟度评分卡](/zh-CN/maturity/scorecard)。

对于角色和风格检查，请使用多个实时
模型引用运行同一场景，并编写经评审的 Markdown 报告：

```bash
pnpm openclaw qa character-eval \
  --model openai/gpt-5.6-luna,thinking=medium,fast \
  --model openai/gpt-5.2,thinking=xhigh \
  --model openai/gpt-5,thinking=xhigh \
  --model anthropic/claude-opus-4-8,thinking=high \
  --model anthropic/claude-sonnet-4-6,thinking=high \
  --model zai/glm-5.1,thinking=high \
  --model moonshot/kimi-k2.5,thinking=high \
  --model google/gemini-3.1-pro-preview,thinking=high \
  --judge-model openai/gpt-5.6-sol,thinking=xhigh,fast \
  --judge-model anthropic/claude-opus-4-8,thinking=high \
  --blind-judge-models \
  --concurrency 16 \
  --judge-concurrency 16
```

该命令运行本地 QA Gateway 网关子进程，而不是 Docker。角色
评估场景应通过 `SOUL.md` 设置角色设定，然后运行常规
用户轮次，例如聊天、工作区帮助和小型文件任务。不应告知候选
模型它正在接受评估。该命令会保留
每份完整转录记录，记录基本运行统计数据，然后要求评审模型以
快速模式运行，并在支持的情况下使用 `xhigh` 推理，根据
自然度、氛围和幽默感对运行结果进行排名。比较
提供商时使用 `--blind-judge-models`：评审提示仍会获得每份转录记录和运行状态，但
候选引用会替换为 `candidate-01` 等中性标签；解析后，
报告会将排名映射回真实引用。

候选运行默认使用 `high` 思考级别；GPT-5.6 Luna 使用 `medium`，
而支持该设置的旧版 OpenAI 评估引用使用 `xhigh`。可通过 `--model provider/model,thinking=<level>` 内联覆盖特定
候选项；内联
选项还支持 `fast`、`no-fast` 和 `fast=<bool>`。`--thinking
<level>` 仍用于设置全局后备值，旧版 `--model-thinking
<provider/model=level>` 形式则为兼容性而保留。OpenAI 候选
引用默认使用快速模式，以便在提供商支持时使用优先处理。
只有在需要为所有候选模型强制开启快速模式时，才传入 `--fast`。
候选项和评审项的持续时间都会记录在
报告中以供基准分析，但评审提示会明确要求不要根据
速度进行排名。候选模型和评审模型运行的默认并发数均为 16。
当提供商限制或本地
Gateway 网关压力导致运行噪声过大时，降低 `--concurrency` 或 `--judge-concurrency`。

未传入候选 `--model` 时，角色评估默认使用
`openai/gpt-5.6-luna`、`openai/gpt-5.2`、`openai/gpt-5`、
`anthropic/claude-opus-4-8`、`anthropic/claude-sonnet-4-6`、`zai/glm-5.1`、
`moonshot/kimi-k2.5` 和 `google/gemini-3.1-pro-preview`。未传入
`--judge-model` 时，评审模型默认使用
`openai/gpt-5.6-sol,thinking=xhigh,fast` 和
`anthropic/claude-opus-4-8,thinking=high`。

## 相关文档

- [成熟度评分卡](/zh-CN/maturity/scorecard)
- [个人智能体基准包](/zh-CN/concepts/personal-agent-benchmark-pack)
- [QA 渠道](/zh-CN/channels/qa-channel)
- [测试](/zh-CN/help/testing)
- [仪表板](/zh-CN/web/dashboard)
