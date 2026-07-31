---
read_when:
    - 运行实时模型矩阵 / CLI 后端 / ACP / 媒体提供商冒烟测试
    - 调试实时测试凭据解析
    - 添加新的提供商专用实时测试
sidebarTitle: Live tests
summary: 实时（涉及网络访问）测试：模型矩阵、CLI 后端、ACP、媒体提供商、凭据
title: 测试：实时测试套件
x-i18n:
    generated_at: "2026-07-26T06:18:16Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: ea8279e734e3aa09dd1fa184806c925e0404edfa9acf0f682f73a4955ed90b8b
    source_path: help/testing-live.md
    workflow: 16
---

有关快速开始、QA 运行器、单元/集成测试套件和 Docker 流程，请参阅
[测试](/zh-CN/help/testing)。本页介绍**实时**（会访问网络的）测试：
模型矩阵、CLI 后端、ACP、媒体提供商和凭据处理。

## 实时测试与你的实际 Gateway 网关

实时测试套件和临时冒烟测试绝不能干扰已经在
处理实际流量的 Gateway 网关（无论是你的还是其他操作员的）：

- 使用你自己的 Gateway 网关：使用进程内 Gateway 网关（见下方第 2 层），或使用隔离的状态目录（`OPENCLAW_STATE_DIR=<scratch>`）和
  空闲端口启动开发实例。当实际 Gateway 网关正在使用默认 Gateway 网关端口（18789）时，
  不要绑定该端口。
- 不要对并非由你在本次会话中启动的服务执行 `openclaw gateway stop`/`restart`（或 `launchctl`/`systemctl`/tmux
  等效操作）——那是操作员的实时实例。必须先获得明确批准。
- 需要真实数据？将实时状态/数据库复制到开发状态目录中，并针对副本进行测试。
  对实时 Gateway 网关状态执行原地迁移也需要
  明确批准。

## 实时：本地冒烟测试命令

在执行临时实时检查前，先在进程环境中导出所需的提供商密钥。

安全的媒体冒烟测试：

```bash
pnpm openclaw infer tts convert --local --json \
  --text "OpenClaw 实时冒烟测试。" \
  --output /tmp/openclaw-live-smoke.mp3
```

安全的语音通话就绪性冒烟测试：

```bash
pnpm openclaw voicecall setup --json
pnpm openclaw voicecall smoke --to "+15555550123"
```

除非同时提供 `--yes`，否则 `voicecall smoke` 是试运行；仅当你确实打算拨打真实电话时
才使用 `--yes`。对于 Twilio、Telnyx 和 Plivo，
成功的就绪性检查需要公共 webhook URL——本地/私有
回环 URL 会被拒绝，因为这些提供商无法访问它们。

## 实时：Android 节点能力全面测试

- 测试：`src/gateway/android-node.capabilities.live.test.ts`
- 脚本：`pnpm android:test:integration`
- 目标：调用已连接 Android 节点**当前公布的每条命令**，并断言命令契约行为。
- 范围：
  - 需要预先准备/手动设置（该测试套件不会安装、运行或配对应用）。
  - 针对所选 Android 节点逐条验证 Gateway 网关命令 `node.invoke`。
- 所需的预先设置：
  - Android 应用已连接并配对到 Gateway 网关。
  - 应用保持在前台。
  - 已为你预期通过的能力授予权限/采集许可。
- 可选目标覆盖：
  - `OPENCLAW_ANDROID_NODE_ID` 或 `OPENCLAW_ANDROID_NODE_NAME`。
  - `OPENCLAW_ANDROID_GATEWAY_URL` / `OPENCLAW_ANDROID_GATEWAY_TOKEN` / `OPENCLAW_ANDROID_GATEWAY_PASSWORD`。
- 完整 Android 设置详情：[Android 应用](/zh-CN/platforms/android)

## 实时：模型冒烟测试（配置文件密钥）

实时模型测试分为两层，以便隔离故障：

- “直接模型”用于判断提供商/模型能否使用给定密钥正常回答。
- “Gateway 网关冒烟测试”用于判断该模型的完整 Gateway 网关 + 智能体流水线是否正常工作（会话、历史记录、工具、沙箱策略等）。

下方精选模型列表位于 `src/agents/live-model-filter.ts` 中，
并且会随时间变化；请以其中的数组为准，而不是以本页为准。

MiniMax M3 使用 `minimax/MiniMax-M3` 作为默认提供商/模型引用。

### 第 1 层：直接模型补全（无 Gateway 网关）

- 测试：`src/agents/models.profiles.live.test.ts`
- 目标：
  - 枚举发现的模型
  - 使用 `getApiKeyForModel` 选择你拥有凭据的模型
  - 为每个模型运行一次小型补全（并在需要时运行针对性的回归测试）
- 启用方式：
  - `pnpm test:live`（如果直接调用 Vitest，则使用 `OPENCLAW_LIVE_TEST=1`）
  - 设置 `OPENCLAW_LIVE_MODELS=modern`、`small` 或 `all`（`modern` 的别名）以实际运行此测试套件；否则它会跳过，因此单独使用 `pnpm test:live` 时仍只关注 Gateway 网关冒烟测试。
- 选择模型的方式：
  - `OPENCLAW_LIVE_MODELS=modern` 运行精选的高信号优先列表（参见[实时：模型矩阵](#live-model-matrix-what-we-cover)）
  - `OPENCLAW_LIVE_MODELS=small` 运行精选的小模型优先列表
  - `OPENCLAW_LIVE_MODELS=all` 是 `modern` 的别名
  - 或者使用 `OPENCLAW_LIVE_MODELS="openai/gpt-5.6-luna,anthropic/claude-opus-4-6,..."`（逗号分隔的允许列表）
  - 本地 Ollama 小模型运行默认使用 `http://127.0.0.1:11434`；仅对局域网、自定义或 Ollama Cloud 端点设置 `OPENCLAW_LIVE_OLLAMA_BASE_URL`。
  - 现代/全部模型和小模型的全面测试默认以其精选列表长度作为上限；设置 `OPENCLAW_LIVE_MAX_MODELS=0` 可对所选配置文件执行无遗漏的全面测试，也可设置一个正数以使用更小的上限。
  - 无遗漏的全面测试使用 `OPENCLAW_LIVE_TEST_TIMEOUT_MS` 设置整个直接模型测试的超时时间。默认值：60 分钟。
  - 直接模型探测默认以 20 路并行运行；设置 `OPENCLAW_LIVE_MODEL_CONCURRENCY` 可覆盖该值。
- 选择提供商的方式：
  - `OPENCLAW_LIVE_PROVIDERS="google,google-antigravity,google-gemini-cli"`（逗号分隔的允许列表）
- 密钥来源：
  - 默认：配置文件存储和环境变量回退
  - 设置 `OPENCLAW_LIVE_REQUIRE_PROFILE_KEYS=1` 可强制**仅使用配置文件存储**
- 为何需要此测试：
  - 将“提供商 API 已损坏/密钥无效”与“Gateway 网关智能体流水线已损坏”区分开来
  - 包含小型、隔离的回归测试（例如：OpenAI Responses/Codex Responses 推理重放 + 工具调用流程）

### 第 2 层：Gateway 网关 + 开发智能体冒烟测试（“@openclaw”的实际行为）

- 测试：`src/gateway/gateway-models.profiles.live.test.ts`
- 目标：
  - 启动一个进程内 Gateway 网关
  - 创建/修补一个 `agent:dev:*` 会话（每次运行覆盖模型）
  - 遍历具有密钥的模型并断言：
    - “有意义的”响应（不使用工具）
    - 实际工具调用可以正常工作（读取探测）
    - 可选的额外工具探测（执行 + 读取探测）
    - OpenAI 回归路径（仅工具调用 -> 后续响应）持续正常工作
- 探测详情（便于你快速解释故障）：
  - `read` 探测：测试在工作区中写入一个随机数文件，并要求智能体使用 `read` 读取它，然后回显该随机数。
  - `exec+read` 探测：测试要求智能体使用 `exec` 将随机数写入临时文件，然后使用 `read` 将其读回。
  - 图像探测：测试附加一个生成的 PNG（猫 + 随机代码），并预期模型返回 `cat <CODE>`。
  - 实现参考：`src/gateway/gateway-models.profiles.live.test.ts` 和 `test/helpers/live-image-probe.ts`。
- 启用方式：
  - `pnpm test:live`（如果直接调用 Vitest，则使用 `OPENCLAW_LIVE_TEST=1`）
- 选择模型的方式：
  - 默认：精选的高信号（`modern`）优先列表
  - `OPENCLAW_LIVE_GATEWAY_MODELS=small` 通过完整的 Gateway 网关 + 智能体流水线运行精选的小模型列表
  - `OPENCLAW_LIVE_GATEWAY_MODELS=all` 是 `modern` 的别名
  - 或者设置 `OPENCLAW_LIVE_GATEWAY_MODELS="provider/model"`（或逗号分隔的列表）以缩小范围
  - 现代/全部模型和小模型的 Gateway 网关全面测试默认以其精选列表长度作为上限；设置 `OPENCLAW_LIVE_GATEWAY_MAX_MODELS=0` 可对所选模型执行无遗漏的全面测试，也可设置一个正数以使用更小的上限。
- 选择提供商的方式（避免“所有内容都通过 OpenRouter”）：
  - `OPENCLAW_LIVE_GATEWAY_PROVIDERS="google,google-antigravity,google-gemini-cli,openai,anthropic,zai,minimax"`（逗号分隔的允许列表）
- 此实时测试始终启用工具和图像探测：
  - `read` 探测 + `exec+read` 探测（工具压力测试）
  - 当模型声明支持图像输入时运行图像探测
  - 流程（概览）：
    - 测试生成一个包含“CAT”+ 随机代码的微型 PNG（`test/helpers/live-image-probe.ts`）
    - 通过 `agent` `attachments: [{ mimeType: "image/png", content: "<base64>" }]` 发送它
    - Gateway 网关将附件解析为 `images[]`（`src/gateway/server-methods/agent.ts` + `src/gateway/chat-attachments.ts`）
    - 嵌入式智能体将多模态用户消息转发给模型
    - 断言：回复包含 `cat` + 该代码（OCR 容错：允许轻微错误）

<Tip>
要查看你的计算机上可以测试哪些内容（以及确切的 `provider/model` ID），请运行：

```bash
openclaw models list
openclaw models list --json
```

</Tip>

## 实时：CLI 后端冒烟测试（Claude、Gemini 或其他本地 CLI）

- 测试：`src/gateway/gateway-cli-backend.live.test.ts`
- 目标：使用本地 CLI 后端验证 Gateway 网关 + 智能体流水线，而不修改默认配置。
- 特定于后端的冒烟测试默认值与所属插件的 `cli-backend.ts` 定义存放在一起。
- 启用：
  - `pnpm test:live`（如果直接调用 Vitest，则使用 `OPENCLAW_LIVE_TEST=1`）
  - `OPENCLAW_LIVE_CLI_BACKEND=1`
- 默认值：
  - 默认提供商/模型：`claude-cli/claude-sonnet-4-6`
  - 命令/参数/图像行为来自所属 CLI 后端插件的元数据。
- 覆盖项（可选）：
  - `OPENCLAW_LIVE_CLI_BACKEND_MODEL="claude-cli/claude-sonnet-4-6"`
  - `OPENCLAW_LIVE_CLI_BACKEND_COMMAND="/full/path/to/claude"`
  - `OPENCLAW_LIVE_CLI_BACKEND_ARGS='["-p","--output-format","json"]'`
  - `OPENCLAW_LIVE_CLI_BACKEND_IMAGE_PROBE=1` 用于发送真实图像附件（路径会注入提示词中）。在 Docker 配方中默认关闭。
  - `OPENCLAW_LIVE_CLI_BACKEND_IMAGE_ARG="--image"` 用于将图像文件路径作为 CLI 参数传递，而不是注入提示词。
  - `OPENCLAW_LIVE_CLI_BACKEND_IMAGE_MODE="repeat"`（或 `"list"`）用于控制设置 `IMAGE_ARG` 时图像参数的传递方式。
  - `OPENCLAW_LIVE_CLI_BACKEND_RESUME_PROBE=1` 用于发送第二轮消息并验证恢复流程。
  - `OPENCLAW_LIVE_CLI_BACKEND_MODEL_SWITCH_PROBE=1` 用于选择启用 Claude Sonnet -> Opus 同一会话连续性探测，前提是所选模型支持切换目标。默认关闭，Docker 配方中也同样关闭。
  - `OPENCLAW_LIVE_CLI_BACKEND_MCP_PROBE=1` 用于选择启用 MCP/工具回环探测。在 Docker 配方中默认关闭。

示例：

```bash
  OPENCLAW_LIVE_CLI_BACKEND=1 \
  OPENCLAW_LIVE_CLI_BACKEND_MODEL="claude-cli/claude-sonnet-4-6" \
  pnpm test:live src/gateway/gateway-cli-backend.live.test.ts
```

低成本 Gemini MCP 配置冒烟测试：

```bash
OPENCLAW_LIVE_TEST=1 \
  pnpm test:live src/agents/cli-runner/bundle-mcp.gemini.live.test.ts
```

此测试不会要求 Gemini 生成响应。它会写入 OpenClaw 提供给 Gemini 的相同系统
设置，然后运行 `gemini --debug mcp list`，以证明已保存的 `transport: "streamable-http"` 服务器会被规范化为 Gemini 的 HTTP MCP
结构，并且能够连接到本地可流式传输的 HTTP MCP 服务器。

Docker 配方：

```bash
pnpm test:docker:live-cli-backend
```

单提供商 Docker 配方：

```bash
pnpm test:docker:live-cli-backend:claude
pnpm test:docker:live-cli-backend:claude-subscription
pnpm test:docker:live-cli-backend:gemini
```

注意事项：

- Docker 运行器位于 `scripts/test-live-cli-backend-docker.sh`。
- 它在仓库 Docker 镜像中以非 root 的 `node` 用户身份运行实时 CLI 后端冒烟测试。
- 它从所属插件解析 CLI 冒烟测试元数据，然后将匹配的 Linux CLI 软件包（`@anthropic-ai/claude-code` 或 `@google/gemini-cli`）安装到 `OPENCLAW_DOCKER_CLI_TOOLS_DIR` 中缓存的可写前缀（默认值：`~/.cache/openclaw/docker-cli-tools`）。
- `codex-cli` 不再是内置 CLI 后端；请改用 `openai/*` 和 Codex app-server 运行时（参见[实时：Codex app-server harness 冒烟测试](#live-codex-app-server-harness-smoke)）。
- `pnpm test:docker:live-cli-backend:claude-subscription` 需要可移植的 Claude Code 订阅 OAuth，可通过带有 `claudeAiOauth.subscriptionType` 的 `~/.claude/.credentials.json`，或来自 `claude setup-token` 的 `CLAUDE_CODE_OAUTH_TOKEN` 提供。它首先在 Docker 中验证直接 `claude -p`，然后运行两个 Gateway 网关 CLI 后端轮次，且不保留 Anthropic API key 环境变量。此订阅通道默认禁用 Claude MCP/工具和图像探测，因为它会消耗已登录订阅的使用限额，并且 Anthropic 可能在 OpenClaw 未发布新版本的情况下更改 Claude Agent SDK / `claude -p` 的计费和速率限制行为。
- Claude 和 Gemini 通过上述标志支持同一组探测（文本轮次、图像分类、MCP `cron` 工具调用、模型切换连续性），但默认不会运行其中任何探测——根据需要通过各自的标志选择启用。

## 实时：APNs HTTP/2 代理可达性

- 测试：`src/infra/push-apns-http2.live.test.ts`
- 目标：通过本地 HTTP CONNECT 代理建立到 Apple 沙箱 APNs 端点的隧道，发送 APNs HTTP/2 验证请求，并断言 Apple 的真实 `403 InvalidProviderToken` 响应会通过代理路径返回。
- 启用：
  - `OPENCLAW_LIVE_TEST=1 OPENCLAW_LIVE_APNS_REACHABILITY=1 pnpm test:live src/infra/push-apns-http2.live.test.ts`
- 可选超时：
  - `OPENCLAW_LIVE_APNS_TIMEOUT_MS=30000`

## 实时：ACP 绑定冒烟测试（`/acp spawn ... --bind here`）

- 测试：`src/gateway/gateway-acp-bind.live.test.ts`
- 目标：使用实时 ACP 智能体验证真实的 ACP 对话绑定流程：
  - 发送 `/acp spawn <agent> --bind here`
  - 原地绑定合成的消息渠道对话
  - 在同一对话中发送正常的后续消息
  - 验证后续消息已写入绑定的 ACP 会话记录
- 启用：
  - `pnpm test:live src/gateway/gateway-acp-bind.live.test.ts`
  - `OPENCLAW_LIVE_ACP_BIND=1`
- 默认值：
  - Docker 中的 ACP 智能体：`claude,codex,gemini`
  - 用于直接 `pnpm test:live ...` 的 ACP 智能体：`claude`
  - 合成渠道：Slack 私信式对话上下文
  - ACP 后端：`acpx`
- 覆盖项：
  - `OPENCLAW_LIVE_ACP_BIND_AGENT=claude`
  - `OPENCLAW_LIVE_ACP_BIND_AGENT=codex`
  - `OPENCLAW_LIVE_ACP_BIND_AGENT=droid`
  - `OPENCLAW_LIVE_ACP_BIND_AGENT=gemini`
  - `OPENCLAW_LIVE_ACP_BIND_AGENT=opencode`
  - `OPENCLAW_LIVE_ACP_BIND_AGENTS=claude,codex,gemini`
  - `OPENCLAW_LIVE_ACP_BIND_AGENT_COMMAND='npx -y @agentclientprotocol/claude-agent-acp@<version>'`
  - `OPENCLAW_LIVE_ACP_BIND_CODEX_MODEL=gpt-5.6-luna`
  - `OPENCLAW_LIVE_ACP_BIND_OPENCODE_MODEL=opencode/kimi-k2.6`
  - 使用 `OPENCLAW_LIVE_ACP_BIND_IMAGE_PROBE=1`（或 `on`/`true`/`yes`）强制启用图像探测；任何其他值都会强制禁用。除 `opencode` 外，默认对每个智能体运行。
  - `OPENCLAW_LIVE_ACP_BIND_REQUIRE_CRON=1`
  - `OPENCLAW_LIVE_ACP_BIND_PARENT_MODEL=openai/gpt-5.6-luna`
- 说明：
  - 此通道使用 Gateway 网关的 `chat.send` 表面以及仅限管理员使用的合成来源路由字段，使测试能够附加消息渠道上下文，而无需假装向外部投递。
  - 未设置 `OPENCLAW_LIVE_ACP_BIND_AGENT_COMMAND` 时，测试会为选定的 ACP harness 智能体使用嵌入式 `acpx` 插件的内置智能体注册表。
  - 默认情况下，绑定会话的 cron MCP 创建采用尽力而为方式，因为外部 ACP harness 可能会在绑定/图像验证通过后取消 MCP 调用；设置 `OPENCLAW_LIVE_ACP_BIND_REQUIRE_CRON=1` 可使该绑定后 cron 探测采用严格模式。

示例：

```bash
OPENCLAW_LIVE_ACP_BIND=1 \
  OPENCLAW_LIVE_ACP_BIND_AGENT=claude \
  pnpm test:live src/gateway/gateway-acp-bind.live.test.ts
```

Docker 运行方法：

```bash
pnpm test:docker:live-acp-bind
```

单智能体 Docker 运行方法：

```bash
pnpm test:docker:live-acp-bind:claude
pnpm test:docker:live-acp-bind:codex
pnpm test:docker:live-acp-bind:droid
pnpm test:docker:live-acp-bind:gemini
pnpm test:docker:live-acp-bind:opencode
```

Docker 说明：

- Docker 运行器位于 `scripts/test-live-acp-bind-docker.sh`。
- 默认情况下，它会依次针对聚合的实时 CLI 智能体运行 ACP 绑定冒烟测试：`claude`、`codex`，然后是 `gemini`。
- 使用 `OPENCLAW_LIVE_ACP_BIND_AGENTS=claude`、`OPENCLAW_LIVE_ACP_BIND_AGENTS=codex`、`OPENCLAW_LIVE_ACP_BIND_AGENTS=droid`、`OPENCLAW_LIVE_ACP_BIND_AGENTS=gemini` 或 `OPENCLAW_LIVE_ACP_BIND_AGENTS=opencode` 缩小矩阵范围。
- 它会将匹配的 CLI 身份验证材料暂存到容器中，然后在缺失时安装所请求的实时 CLI（`@anthropic-ai/claude-code`、`@openai/codex`、通过 `https://app.factory.ai/cli` 安装的 Factory Droid、`@google/gemini-cli` 或 `opencode-ai`）。ACP 后端本身是来自官方 `acpx` 插件的嵌入式 `acpx/runtime` 软件包。
- Droid Docker 变体会暂存 `~/.factory` 以提供设置、转发 `FACTORY_API_KEY`，并要求提供该 API key，因为本地 Factory OAuth/密钥环身份验证无法移植到容器中。它使用 ACPX 的内置 `droid exec --output-format acp` 注册表条目。
- OpenCode Docker 变体是严格的单智能体回归通道。它会根据 `OPENCLAW_LIVE_ACP_BIND_OPENCODE_MODEL`（默认值为 `opencode/kimi-k2.6`）写入临时的 `OPENCODE_CONFIG_CONTENT` 默认模型。
- 直接 `acpx` CLI 调用仅作为在 Gateway 网关之外比较行为的手动/变通路径。Docker ACP 绑定冒烟测试会执行 OpenClaw 的嵌入式 `acpx` 运行时后端。

## 实时：Codex app-server harness 冒烟测试

- 目标：通过常规 Gateway 网关
  `agent` 方法验证插件所属的 Codex harness：
  - 加载内置 `codex` 插件
  - 通过 `/model <ref> --runtime codex` 选择 OpenAI 模型
  - 使用请求的思考级别发送第一个 Gateway 网关智能体轮次
  - 向同一 OpenClaw 会话发送第二个轮次，并验证 app-server
    线程可以恢复
  - 通过同一 Gateway 网关命令
    路径运行 `/codex status` 和 `/codex models`
  - 可选择运行两个经过 Guardian 审查的提升权限 shell 探测：一个应获批准的良性
    命令，以及一个应被拒绝的虚假机密上传，使智能体发起
    询问
- 测试：`src/gateway/gateway-codex-harness.live.test.ts`
- 启用：`OPENCLAW_LIVE_CODEX_HARNESS=1`
- Harness 基线模型：`openai/gpt-5.6-luna`
- 全新 OpenAI API key 选择默认值：`openai/gpt-5.6`
- 默认思考级别：`low`
- 模型覆盖：`OPENCLAW_LIVE_CODEX_HARNESS_MODEL=openai/<model>`
- 思考级别覆盖：`OPENCLAW_LIVE_CODEX_HARNESS_THINKING=<level>`
- 非默认模型工作量断言：
  `OPENCLAW_LIVE_CODEX_HARNESS_EXPECTED_EFFORT=<level>`
- 矩阵覆盖：`OPENCLAW_LIVE_CODEX_HARNESS_TARGETS=<model>=<thinking>,...`
- 身份验证模式：`OPENCLAW_LIVE_CODEX_HARNESS_AUTH=codex-auth`（默认）使用
  复制的 Codex 登录信息；`api-key` 通过 Codex app-server 使用 `OPENAI_API_KEY`。
- 可选图像探测：`OPENCLAW_LIVE_CODEX_HARNESS_IMAGE_PROBE=1`
- 可选 MCP/工具探测：`OPENCLAW_LIVE_CODEX_HARNESS_MCP_PROBE=1`
- 可选 Guardian 探测：`OPENCLAW_LIVE_CODEX_HARNESS_GUARDIAN_PROBE=1`
- 可选恢复压力测试：`OPENCLAW_LIVE_CODEX_HARNESS_RESUME_STRESS=1` 会添加
  四个历史轮次，然后关闭并重启 Gateway 网关和 Codex app-server
  三次，同时要求保持相同的原生线程 ID 和对话
  历史记录。可使用
  `OPENCLAW_LIVE_CODEX_HARNESS_RESUME_STRESS_HISTORY_TURNS`（1-20）和
  `OPENCLAW_LIVE_CODEX_HARNESS_RESUME_STRESS_RESTARTS`（1-10）覆盖这些有界次数。
- 可选扇出压力测试：设置 `OPENCLAW_LIVE_CODEX_HARNESS_SUBAGENT_PROBE=1`
  和 `OPENCLAW_LIVE_CODEX_HARNESS_SUBAGENT_COUNT`（1-12）。Harness 会并发启动
  每个子项，等待每个运行进入终止状态，并验证各个
  唯一子项回复和原生线程身份。
- 可选压缩压力测试：`OPENCLAW_LIVE_CODEX_HARNESS_COMPACTION_STRESS=1`
  会生成有界的原生工具输出，要求触发自动压缩事件，
  验证持久化的压缩计数和隐藏标记回忆，重启
  Gateway 网关和实际的 Codex app-server，然后重复输出和
  压缩波次。可使用
  `OPENCLAW_LIVE_CODEX_HARNESS_COMPACTION_STRESS_TURNS`（1-8）和
  `OPENCLAW_LIVE_CODEX_HARNESS_LARGE_OUTPUT_BYTES`（100000-800000）调整有界工作量。
- 完整直接 API 上下文：`OPENCLAW_LIVE_CODEX_HARNESS_FULL_CONTEXT=1` 会应用
  `922000` 上下文限制和 `700000` 总压缩限制，发送密集的有界
  用户轮次，在每个波次运行两个显式原生压缩检查点，并在
  每个检查点后继续后续轮次。它要求
  `OPENCLAW_LIVE_CODEX_HARNESS_AUTH=api-key` 以及绝对
  `OPENCLAW_LIVE_CODEX_HARNESS_MODEL_CATALOG` 路径。目录必须通过 `max_context_window: 922000` 公开
  所选模型，以免 Codex 将覆盖值限制回其常规目录窗口。
  上述普通的降低阈值压力测试仍会保留更严格的自动压缩和隐藏标记
  留存断言。
- 可选循环中继退出探测：
  `OPENCLAW_LIVE_CODEX_HARNESS_DISABLE_LOOP_RELAY=1`
- 请求的思考偏好可能会映射到 Codex 针对该模型公布的最接近工作量。
  例如，Luna 会将 `minimal` 映射为 `low`。
- 已知的 Codex 目录模型会自动推导出该精确的原生工作量。
  未知模型覆盖必须声明预期的映射工作量。
- 该冒烟测试会强制使用提供商/模型 `agentRuntime.id: "codex"`，以防损坏的 Codex
  harness 通过静默回退到 OpenClaw 而通过测试。
- 身份验证：使用本地 Codex 订阅登录的 Codex app-server 身份验证，或在
  `OPENCLAW_LIVE_CODEX_HARNESS_AUTH=api-key` 时使用 `OPENAI_API_KEY`。Docker 可以
  复制 `~/.codex/auth.json` 和 `~/.codex/config.toml` 以运行订阅测试。

本地运行方法：

```bash
OPENCLAW_LIVE_CODEX_HARNESS=1 \
  OPENCLAW_LIVE_CODEX_HARNESS_IMAGE_PROBE=1 \
  OPENCLAW_LIVE_CODEX_HARNESS_MCP_PROBE=1 \
  OPENCLAW_LIVE_CODEX_HARNESS_GUARDIAN_PROBE=1 \
  OPENCLAW_LIVE_CODEX_HARNESS_MODEL=openai/gpt-5.6-luna \
  pnpm test:live -- src/gateway/gateway-codex-harness.live.test.ts
```

Docker 运行方法：

```bash
pnpm test:docker:live-codex-harness
```

重启和历史记录压力测试：

```bash
OPENCLAW_LIVE_CODEX_HARNESS_RESUME_STRESS=1 \
pnpm test:docker:live-codex-harness
```

扇出、大输出、压缩和重启压力测试：

```bash
OPENCLAW_LIVE_CODEX_HARNESS_AUTH=api-key \
  OPENCLAW_LIVE_CODEX_HARNESS_SUBAGENT_PROBE=1 \
  OPENCLAW_LIVE_CODEX_HARNESS_SUBAGENT_COUNT=8 \
  OPENCLAW_LIVE_CODEX_HARNESS_RESUME_STRESS=1 \
  OPENCLAW_LIVE_CODEX_HARNESS_COMPACTION_STRESS=1 \
  pnpm test:docker:live-codex-harness
```

完整原生 Codex `922000` 输入预算压缩压力测试：

```bash
OPENCLAW_LIVE_CODEX_HARNESS=1 \
  OPENCLAW_LIVE_CODEX_HARNESS_AUTH=api-key \
  OPENCLAW_LIVE_CODEX_HARNESS_FULL_CONTEXT=1 \
  OPENCLAW_LIVE_CODEX_HARNESS_MODEL_CATALOG=/absolute/path/to/models-api-1m.json \
  OPENCLAW_LIVE_CODEX_HARNESS_MODEL=openai/gpt-5.6-terra \
  OPENCLAW_LIVE_CODEX_HARNESS_THINKING=medium \
  OPENCLAW_LIVE_CODEX_HARNESS_COMPACTION_STRESS_TURNS=8 \
  OPENCLAW_LIVE_CODEX_HARNESS_LARGE_OUTPUT_BYTES=800000 \
  pnpm test:live -- src/gateway/gateway-codex-harness.live.test.ts
```

GPT-5.6 原生 Codex 矩阵：

```bash
OPENCLAW_LIVE_CODEX_HARNESS_AUTH=api-key \
  OPENCLAW_LIVE_CODEX_HARNESS_TARGETS='openai/gpt-5.6-sol=ultra,openai/gpt-5.6-terra=ultra,openai/gpt-5.6-luna=max' \
  pnpm test:docker:live-codex-harness
```

## 实时：OpenAI 重复压缩

- 目标：通过至少两次真实的自动压缩来运行嵌入式 OpenClaw `openai-responses` Agent loop，然后验证持久化标记仍然存在。
- 测试：`src/agents/sessions/agent-session.openai-compaction.live.test.ts`
- 启用：`OPENCLAW_LIVE_OPENAI_COMPACTION=1`
- 默认模型：`gpt-5.6-luna`
- 模型覆盖：`OPENCLAW_LIVE_OPENAI_COMPACTION_MODEL=<model>`
- 常规压力模式使用较小的客户端上下文预算，以有限的 API 支出进入相同的真实压缩路径。
- 全上下文模式将客户端预算设为 `922000`，将压缩预留设为 `222000`，因此自动压缩从 `700000` 开始。它还要求观察到的提供商输入计数高于 `272000` 长上下文定价边界。

有界实时测试方案：

```bash
OPENCLAW_LIVE_TEST=1 \
  OPENCLAW_LIVE_OPENAI_COMPACTION=1 \
  pnpm test:live -- src/agents/sessions/agent-session.openai-compaction.live.test.ts
```

完整 `922000` 输入预算测试方案：

```bash
OPENCLAW_LIVE_TEST=1 \
  OPENCLAW_LIVE_OPENAI_COMPACTION=1 \
  OPENCLAW_LIVE_OPENAI_COMPACTION_FULL=1 \
  OPENCLAW_LIVE_OPENAI_COMPACTION_MODEL=gpt-5.6-terra \
  pnpm test:live -- src/agents/sessions/agent-session.openai-compaction.live.test.ts
```

<Warning>
完整模式会有意跨越 OpenAI 的长上下文定价边界，并且可能发起多次大型 API 调用。仅在明确批准支出后使用。
</Warning>

新 OpenAI API key 默认值：

```bash
OPENCLAW_LIVE_GATEWAY_OPENAI_API_DEFAULT=1 \
  OPENCLAW_LIVE_GATEWAY_PROVIDERS=openai \
  OPENCLAW_LIVE_GATEWAY_THINKING=off \
  pnpm test:live -- src/gateway/gateway-models.profiles.live.test.ts
```

此验证让 `OPENCLAW_LIVE_GATEWAY_MODELS` 保持未设置状态，通过新的新手引导推理选择接缝解析模型，断言 `openai/gpt-5.6`，然后使用解析出的模型运行一次真实的 Gateway 网关轮次。

GPT-5.6 嵌入式 OpenClaw 矩阵：

```bash
OPENCLAW_LIVE_GATEWAY_THINKING=ultra \
  OPENCLAW_LIVE_GATEWAY_PROVIDERS=openai \
  OPENCLAW_LIVE_GATEWAY_MODELS='openai/gpt-5.6-sol,openai/gpt-5.6-terra,openai/gpt-5.6-luna' \
  pnpm test:live -- src/gateway/gateway-models.profiles.live.test.ts
```

Docker 说明：

- Docker 运行器位于 `scripts/test-live-codex-harness-docker.sh`。
- 它会传递 `OPENAI_API_KEY`，在 Codex CLI 身份验证文件存在时复制这些文件，将 `@openai/codex` 安装到可写的已挂载 npm 前缀中，暂存源代码树，然后仅运行 Codex harness 实时测试。
- Docker 默认启用镜像、MCP/工具和 Guardian 探测。需要更窄范围的调试运行时，请设置 `OPENCLAW_LIVE_CODEX_HARNESS_IMAGE_PROBE=0`、`OPENCLAW_LIVE_CODEX_HARNESS_MCP_PROBE=0` 或 `OPENCLAW_LIVE_CODEX_HARNESS_GUARDIAN_PROBE=0`。
- Docker 使用相同的显式 Codex 运行时配置，因此旧版别名或 OpenClaw 回退无法掩盖 Codex harness 回归。
- 矩阵目标在一个容器中依次运行。Docker 脚本会根据目标数量调整其默认 35 分钟超时；任何外层 shell 或 CI 超时都必须允许相同的总时长。规范 CI 会将每个 GPT-5.6 目标放在单独的分片中。

### 推荐的实时测试方案

范围窄且明确的允许列表速度最快，也最不容易不稳定：

- 单个模型，直接运行（不使用 Gateway 网关）：
  - `OPENCLAW_LIVE_MODELS="openai/gpt-5.6-luna" pnpm test:live src/agents/models.profiles.live.test.ts`

- 小模型直接配置：
  - `OPENCLAW_LIVE_MODELS=small pnpm test:live src/agents/models.profiles.live.test.ts`

- 小模型 Gateway 网关配置：
  - `OPENCLAW_LIVE_GATEWAY_MODELS=small pnpm test:live src/gateway/gateway-models.profiles.live.test.ts`

- Ollama Cloud API 冒烟测试：
  - `OPENCLAW_LIVE_TEST=1 OPENCLAW_LIVE_OLLAMA=1 OPENCLAW_LIVE_OLLAMA_BASE_URL=https://ollama.com OPENCLAW_LIVE_OLLAMA_MODEL=glm-5.1:cloud OPENCLAW_LIVE_OLLAMA_WEB_SEARCH=0 pnpm test:live -- extensions/ollama/ollama.live.test.ts`

- 单个模型，Gateway 网关冒烟测试：
  - `OPENCLAW_LIVE_GATEWAY_MODELS="openai/gpt-5.6-luna" pnpm test:live src/gateway/gateway-models.profiles.live.test.ts`

- 跨多个提供商的工具调用：
  - `OPENCLAW_LIVE_GATEWAY_MODELS="openai/gpt-5.6-luna,anthropic/claude-opus-4-6,google/gemini-3.5-flash,deepseek/deepseek-v4-flash,zai/glm-5.1,minimax/MiniMax-M3" pnpm test:live src/gateway/gateway-models.profiles.live.test.ts`

- Z.AI Coding Plan GLM-5.2 直接冒烟测试：
  - `ZAI_CODING_LIVE_TEST=1 pnpm test:live src/agents/zai.live.test.ts`

- Google 重点测试（Gemini API key + Antigravity）：
  - Gemini（API key）：`OPENCLAW_LIVE_GATEWAY_MODELS="google/gemini-3.5-flash" pnpm test:live src/gateway/gateway-models.profiles.live.test.ts`
  - Antigravity（OAuth）：`OPENCLAW_LIVE_GATEWAY_MODELS="google-antigravity/claude-opus-4-6-thinking,google-antigravity/gemini-3-pro-high" pnpm test:live src/gateway/gateway-models.profiles.live.test.ts`

- Google 自适应思考冒烟测试（来自私有 QA CLI 的 `qa manual`——需要 `OPENCLAW_ENABLE_PRIVATE_QA_CLI=1` 和源代码检出；请参阅 [QA overview](/zh-CN/concepts/qa-e2e-automation)）：
  - Gemini 3 动态默认值：`OPENCLAW_ENABLE_PRIVATE_QA_CLI=1 pnpm openclaw qa manual --provider-mode live-frontier --model google/gemini-3.1-pro-preview --alt-model google/gemini-3.1-pro-preview --message '/think adaptive Reply exactly: GEMINI_ADAPTIVE_OK' --timeout-ms 180000`
  - Gemini 2.5 动态预算：`OPENCLAW_ENABLE_PRIVATE_QA_CLI=1 pnpm openclaw qa manual --provider-mode live-frontier --model google/gemini-2.5-flash --alt-model google/gemini-2.5-flash --message '/think adaptive Reply exactly: GEMINI25_ADAPTIVE_OK' --timeout-ms 180000`

说明：

- `google/...` 使用 Gemini API（API key）。
- `google-antigravity/...` 使用 Antigravity OAuth 桥接（Cloud Code Assist 风格的 Agent 端点）。
- `google-gemini-cli/...` 使用你计算机上的本地 Gemini CLI（具有独立的身份验证和工具差异）。
- Gemini API 与 Gemini CLI：
  - API：OpenClaw 通过 HTTP 调用 Google 托管的 Gemini API（API key/配置文件身份验证）；大多数用户所说的“Gemini”指的就是它。
  - CLI：OpenClaw 通过 shell 调用本地 `gemini` 二进制文件；它具有自己的身份验证，并且行为可能有所不同（流式传输/工具支持/版本偏差）。

## 实时测试：模型矩阵（覆盖范围）

实时测试需要选择启用，因此不存在固定的“CI 模型列表”。`OPENCLAW_LIVE_MODELS=modern` / `OPENCLAW_LIVE_GATEWAY_MODELS=modern`（及其 `all` 别名）按以下优先级顺序，运行 `src/agents/live-model-filter.ts` 中来自 `HIGH_SIGNAL_LIVE_MODEL_PRIORITY` 的精选优先级列表：

| 提供商/模型                                | 说明      |
| --------------------------------------------- | ---------- |
| `anthropic/claude-opus-5`                     |            |
| `anthropic/claude-opus-4-8`                   |            |
| `anthropic/claude-sonnet-5`                   |            |
| `anthropic/claude-sonnet-4-6`                 |            |
| `anthropic/claude-opus-4-7`                   |            |
| `google/gemini-3.1-pro-preview`               | Gemini API |
| `google/gemini-3.5-flash`                     | Gemini API |
| `cohere/command-a-plus-05-2026`               |            |
| `moonshot/kimi-k3`                            |            |
| `anthropic/claude-opus-4-6`                   |            |
| `deepseek/deepseek-v4-flash`                  |            |
| `deepseek/deepseek-v4-pro`                    |            |
| `minimax/MiniMax-M3`                          |            |
| `openai/gpt-5.5`                              |            |
| `openrouter/openai/gpt-5.2-chat`              |            |
| `openrouter/minimax/minimax-m2.7`             |            |
| `opencode-go/glm-5`                           |            |
| `openrouter/ai21/jamba-large-1.7`             |            |
| `xai/grok-4.5`                                |            |
| `xai/grok-4.20-0309-reasoning`                |            |
| `zai/glm-5.1`                                 |            |
| `fireworks/accounts/fireworks/models/glm-5p1` |            |
| `minimax-portal/minimax-m3`                   |            |

精选的**小模型**列表（`OPENCLAW_LIVE_MODELS=small` / `OPENCLAW_LIVE_GATEWAY_MODELS=small`）来自 `SMALL_LIVE_MODEL_PRIORITY`：

| 提供商/模型               |
| ---------------------------- |
| `lmstudio/qwen/qwen3.5-9b`   |
| `vllm/qwen/qwen3-8b`         |
| `sglang/qwen/qwen3-8b`       |
| `ollama/gemma3:4b`           |
| `openrouter/qwen/qwen3.5-9b` |
| `openrouter/z-ai/glm-5.1`    |
| `openrouter/z-ai/glm-5`      |
| `zai/glm-5.1`                |

现代列表说明：

- `codex` 和 `codex-cli` 提供商不包含在默认现代扫描中（它们覆盖 CLI 后端/ACP 行为，已在上文单独测试）。`openai/gpt-5.5` 本身默认通过 Codex app-server harness 路由；请参阅[实时测试：Codex app-server harness 冒烟测试](#live-codex-app-server-harness-smoke)。
- `fireworks`、`google`、`openrouter` 和 `xai` 在现代扫描中仅运行其明确精选的模型 ID（不会自动扩展为“此提供商的所有模型”）。
- 在 `OPENCLAW_LIVE_GATEWAY_MODELS` 中至少包含一个支持图像的模型（Claude/Gemini/OpenAI 系列视觉变体等），以运行图像探测。

在手工选择的跨提供商集合中，使用工具和图像运行 Gateway 网关冒烟测试：

```bash
OPENCLAW_LIVE_GATEWAY_MODELS="openai/gpt-5.6-luna,anthropic/claude-opus-4-6,google/gemini-3.1-pro-preview,google/gemini-3.5-flash,google-antigravity/claude-opus-4-6-thinking,deepseek/deepseek-v4-flash,zai/glm-5.1,minimax/MiniMax-M3" pnpm test:live src/gateway/gateway-models.profiles.live.test.ts
```

精选列表之外的可选额外覆盖（最好测试；选择一个已启用且支持“工具”的模型）：

- Mistral：`mistral/...`
- Cerebras：`cerebras/...`（如果你有访问权限）
- LM Studio：`lmstudio/...`（本地；工具调用取决于 API 模式）

### 聚合器/备用 Gateway 网关

如果你已启用相应密钥，也可以通过以下方式测试：

- OpenRouter：`openrouter/...`（数百个模型；使用 `openclaw models scan` 查找支持工具和图像的候选模型）
- OpenCode：Zen 使用 `opencode/...`，Go 使用 `opencode-go/...`（通过 `OPENCODE_API_KEY` / `OPENCODE_ZEN_API_KEY` 进行身份验证）

可以包含在实时矩阵中的更多提供商（如果你有凭据/配置）：

- 内置：`anthropic`、`cerebras`、`github-copilot`、`google`、`google-antigravity`、`google-gemini-cli`、`google-vertex`、`groq`、`mistral`、`openai`、`openrouter`、`opencode`、`opencode-go`、`xai`、`zai`
- 通过 `models.providers`（自定义端点）：`minimax`（云端/API），以及任何兼容 OpenAI/Anthropic 的代理（LM Studio、vLLM、LiteLLM 等）

<Tip>
不要在文档中硬编码“所有模型”。权威列表是你的计算机上 `discoverModels(...)` 返回的内容，加上所有可用密钥对应的模型。
</Tip>

## 凭据（切勿提交）

实时测试发现凭据的方式与 CLI 相同。实际含义如下：

- 如果 CLI 可以工作，实时测试应能找到相同的密钥。
- 如果实时测试提示“无凭据”，请采用调试 `openclaw models list` / 模型选择时的相同方式进行调试。

- 每 Agent 身份验证配置文件：`~/.openclaw/agents/<agentId>/agent/auth-profiles.json`（这就是实时测试中“配置文件密钥”的含义）
- 配置：`~/.openclaw/openclaw.json`（或 `OPENCLAW_CONFIG_PATH`）
- 旧版 OAuth 目录：`~/.openclaw/credentials/`（存在时会复制到暂存的实时测试主目录中，但它不是主要的配置文件密钥存储区）
- 本地实时测试运行会将当前配置（去除 `agents.*.workspace` / `agentDir` 覆盖）和每个 Agent 的 `auth-profiles.json` 复制到临时测试主目录中，而不会复制该 Agent 目录的其余内容，因此 `workspace/` 和 `sandboxes/` 数据绝不会进入暂存主目录；此外还会复制旧版 `credentials/` 目录和支持的外部 CLI 身份验证文件/目录（`.claude.json`、`.claude/.credentials.json`、`.claude/settings*.json`、`.claude/backups`、`.codex/auth.json`、`.codex/config.toml`、`.gemini`、`.minimax`）。

如果想依赖环境变量中的密钥，请在本地测试之前导出它们，或者使用下方的 Docker 运行器并显式设置 `OPENCLAW_PROFILE_FILE`。

## Deepgram 实时测试（音频转录）

- 测试：`extensions/deepgram/audio.live.test.ts`
- 启用：`DEEPGRAM_API_KEY=... DEEPGRAM_LIVE_TEST=1 pnpm test:live extensions/deepgram/audio.live.test.ts`

## BytePlus 编码套餐实时测试

- 测试：`extensions/byteplus/live.test.ts`
- 启用：`BYTEPLUS_API_KEY=... BYTEPLUS_LIVE_TEST=1 pnpm test:live extensions/byteplus/live.test.ts`
- 可选模型覆盖：`BYTEPLUS_CODING_MODEL=ark-code-latest`

## ComfyUI 工作流媒体实时测试

- 测试：`extensions/comfy/comfy.live.test.ts`
- 启用：`OPENCLAW_LIVE_TEST=1 COMFY_LIVE_TEST=1 pnpm test:live -- extensions/comfy/comfy.live.test.ts`
- 范围：
  - 运行内置的 comfy 图像、视频和 `music_generate` 路径
  - 除非已配置 `plugins.entries.comfy.config.<capability>`，否则跳过各项能力
  - 适用于更改 comfy 工作流提交、轮询、下载或插件注册之后的验证

## 图像生成实时测试

- 测试：`test/image-generation.runtime.live.test.ts`
- 命令：`pnpm test:live test/image-generation.runtime.live.test.ts`
- 测试框架：`pnpm test:live:media image`
- 范围：
  - 枚举所有已注册的图像生成提供商插件
  - 探测前使用已导出的提供商环境变量
  - 默认优先使用实时/环境变量 API key，而非存储的身份验证配置文件，因此 `auth-profiles.json` 中的过期测试 key 不会掩盖真实的 shell 凭据
  - 跳过没有可用身份验证、配置文件或模型的提供商
  - 通过共享图像生成运行时运行每个已配置的提供商：
    - `<provider>:generate`
    - 当提供商声明支持编辑时运行 `<provider>:edit`
- 当前覆盖的内置提供商：
  - `deepinfra`
  - `fal`
  - `google`
  - `minimax`
  - `openai`
  - `openrouter`
  - `vydra`
  - `xai`
- 可选的范围缩小选项：
  - `OPENCLAW_LIVE_IMAGE_GENERATION_PROVIDERS="openai,google,openrouter,xai"`
  - `OPENCLAW_LIVE_IMAGE_GENERATION_PROVIDERS="deepinfra"`
  - `OPENCLAW_LIVE_IMAGE_GENERATION_MODELS="openai/gpt-image-2,google/gemini-3.1-flash-image,openrouter/google/gemini-3.1-flash-image-preview,xai/grok-imagine-image"`
  - `OPENCLAW_LIVE_IMAGE_GENERATION_CASES="google:flash-generate,google:pro-edit,openrouter:generate,xai:default-generate,xai:default-edit"`
- 可选的身份验证行为：
  - `OPENCLAW_LIVE_REQUIRE_PROFILE_KEYS=1`，用于强制使用配置文件存储中的身份验证并忽略仅限环境变量的覆盖

对于已发布的 CLI 路径，在提供商/运行时实时测试通过后添加一次 `infer` 冒烟测试：

```bash
OPENCLAW_LIVE_TEST=1 OPENCLAW_LIVE_INFER_CLI_TEST=1 pnpm test:live -- test/image-generation.infer-cli.live.test.ts
openclaw infer image providers --json
openclaw infer image generate \
  --model google/gemini-3.1-flash-image \
  --prompt "最简扁平化测试图像：白色背景上有一个蓝色正方形，无文字。" \
  --output ./openclaw-infer-image-smoke.png \
  --json
```

此测试覆盖 CLI 参数解析、配置/默认智能体解析、内置插件激活、共享图像生成运行时和实时提供商请求。运行时加载前应已安装插件依赖项。

## 音乐生成实时测试

- 测试：`extensions/music-generation-providers.live.test.ts`
- 启用：`OPENCLAW_LIVE_TEST=1 pnpm test:live -- extensions/music-generation-providers.live.test.ts`
- 测试框架：`pnpm test:live:media music`
- 范围：
  - 测试共享的内置音乐生成提供商路径
  - 当前覆盖 `fal`、`google`、`minimax` 和 `openrouter`
  - 探测前使用已导出的提供商环境变量
  - 默认优先使用实时/环境变量 API key，而非存储的身份验证配置文件，因此 `auth-profiles.json` 中的过期测试 key 不会掩盖真实的 shell 凭据
  - 跳过没有可用身份验证、配置文件或模型的提供商
  - 可用时运行声明的两种运行时模式：
    - `generate`，仅提供提示词输入
    - 当提供商声明 `capabilities.edit.enabled` 时运行 `edit`
  - `comfy` 有自己的独立实时测试文件，不包含在此共享遍历测试中
- 可选的范围缩小选项：
  - `OPENCLAW_LIVE_MUSIC_GENERATION_PROVIDERS="google,minimax"`
  - `OPENCLAW_LIVE_MUSIC_GENERATION_MODELS="google/lyria-3-clip-preview,minimax/music-2.6"`
- 可选的身份验证行为：
  - `OPENCLAW_LIVE_REQUIRE_PROFILE_KEYS=1`，用于强制使用配置文件存储中的身份验证并忽略仅限环境变量的覆盖

## 视频生成实时测试

- 测试：`extensions/video-generation-providers.live.test.ts`
- 启用：`OPENCLAW_LIVE_TEST=1 pnpm test:live -- extensions/video-generation-providers.live.test.ts`
- 测试框架：`pnpm test:live:media video`
- 范围：
  - 跨 `alibaba`、`byteplus`、`deepinfra`、`fal`、`google`、`minimax`、`openai`、`openrouter`、`pixverse`、`qwen`、`runway`、`together`、`vydra`、`xai` 测试共享的内置视频生成提供商路径
  - 默认使用发布安全的冒烟测试路径：每个提供商发送一次文生视频请求、使用一秒钟的龙虾提示词，并采用 `OPENCLAW_LIVE_VIDEO_GENERATION_TIMEOUT_MS` 中设置的每提供商操作上限（默认为 `180000`）
  - 默认跳过 FAL，因为提供商侧的队列延迟可能占据大部分发布时间；传入 `OPENCLAW_LIVE_VIDEO_GENERATION_PROVIDERS="fal"`（或清空跳过列表）可显式运行它
  - 探测前使用已导出的提供商环境变量
  - 默认优先使用实时/环境变量 API key，而非存储的身份验证配置文件，因此 `auth-profiles.json` 中的过期测试 key 不会掩盖真实的 shell 凭据
  - 跳过没有可用身份验证、配置文件或模型的提供商
  - 默认仅运行 `generate`
  - 设置 `OPENCLAW_LIVE_VIDEO_GENERATION_FULL_MODES=1`，还可在可用时运行声明的转换模式：
    - 当提供商声明 `capabilities.imageToVideo.enabled`，且所选提供商/模型在共享遍历测试中接受由缓冲区支持的本地图像输入时，运行 `imageToVideo`
    - 当提供商声明 `capabilities.videoToVideo.enabled`，且所选提供商/模型在共享遍历测试中接受由缓冲区支持的本地视频输入时，运行 `videoToVideo`
  - 共享遍历测试中当前已声明但跳过的 `imageToVideo` 提供商：
    - `vydra`（此测试通道不支持由缓冲区支持的本地图像输入）
  - Vydra 提供商专用测试覆盖：
    - `OPENCLAW_LIVE_TEST=1 OPENCLAW_LIVE_VYDRA_VIDEO=1 pnpm test:live -- extensions/vydra/vydra.live.test.ts`
    - 该文件运行 `veo3` 文生视频测试，以及一个 `kling` 图生视频测试通道；后者默认使用远程图像 URL 固件（可通过 `OPENCLAW_LIVE_VYDRA_KLING_IMAGE_URL` 覆盖）。
  - xAI 提供商专用测试覆盖：
    - `OPENCLAW_LIVE_TEST=1 OPENCLAW_LIVE_XAI_VIDEO=1 pnpm test:live -- extensions/xai/xai.live.test.ts -t "classic Grok Imagine"`
    - 经典用例首先生成一张正方形本地 PNG 作为首帧，不指定几何尺寸，请求一秒钟的图生视频片段，轮询直至完成，并验证下载的缓冲区。
    - `OPENCLAW_LIVE_TEST=1 OPENCLAW_LIVE_XAI_VIDEO=1 pnpm test:live -- extensions/xai/xai.live.test.ts -t "Grok Imagine Video 1.5"`
    - 1.5 用例生成一张本地 PNG 作为首帧，请求一秒钟的 1080P 图生视频片段，轮询直至完成，并验证下载的缓冲区。
  - 当前 `videoToVideo` 实时测试覆盖：
    - 仅当所选模型解析为 `gen4_aleph` 时运行 `runway`
  - 共享遍历测试中当前已声明但跳过的 `videoToVideo` 提供商：
    - `alibaba`、`google`、`openai`、`qwen`、`xai`，因为这些路径目前需要远程 `http(s)` 引用 URL，而不是由缓冲区支持的本地输入
- 可选的范围缩小选项：
  - `OPENCLAW_LIVE_VIDEO_GENERATION_PROVIDERS="deepinfra,google,openai,runway"`
  - `OPENCLAW_LIVE_VIDEO_GENERATION_MODELS="google/veo-3.1-fast-generate-preview,openai/sora-2,runway/gen4_aleph"`
  - `OPENCLAW_LIVE_VIDEO_GENERATION_SKIP_PROVIDERS=""`，用于在默认遍历测试中包含所有提供商，包括 FAL
  - `OPENCLAW_LIVE_VIDEO_GENERATION_TIMEOUT_MS=60000`，用于降低每个提供商的操作上限，以执行更激进的冒烟测试
- 可选的身份验证行为：
  - `OPENCLAW_LIVE_REQUIRE_PROFILE_KEYS=1`，用于强制使用配置文件存储中的身份验证并忽略仅限环境变量的覆盖

## 媒体实时测试框架

- 命令：`pnpm test:live:media`
- 入口点：`test/e2e/qa-lab/media/hosted-media-provider-live.ts`，它会针对每个所选测试套件运行 `pnpm test:live -- <suite-test-file>`，因此 Heartbeat 和静默模式行为与其他 `pnpm test:live` 运行保持一致。
- 用途：
  - 通过一个仓库原生入口点运行共享的图像、音乐和视频实时测试套件
  - 从 `~/.profile` 自动加载缺失的提供商环境变量
  - 默认自动将每个测试套件缩小至当前具有可用身份验证的提供商
- 标志：
  - `--providers <csv>` 是全局提供商筛选器；`--image-providers` / `--music-providers` / `--video-providers` 将筛选器限定到单个测试套件
  - `--all-providers` 跳过基于身份验证的自动筛选
  - 当筛选后没有可运行的提供商时，`--allow-empty` 以 `0` 退出
  - `--quiet` / `--no-quiet` 会原样传递给 `test:live`
- 示例：
  - `pnpm test:live:media`
  - `pnpm test:live:media image video --providers openai,google,minimax`
  - `pnpm test:live:media video --video-providers openai,runway --all-providers`
  - `pnpm test:live:media music --quiet`

## 相关内容

- [测试](/zh-CN/help/testing) - 单元测试、集成测试、QA 和 Docker 测试套件
