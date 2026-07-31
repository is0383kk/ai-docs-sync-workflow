---
read_when:
    - 你需要先建立推理连接，然后使用 OpenClaw 完成设置
summary: '`openclaw onboard`（交互式新手引导）的 CLI 参考'
title: 引导设置
x-i18n:
    generated_at: "2026-07-26T06:09:44Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 8ec5cfc564aa14041d1aa67a978a4661e6105b7119a942940f71197c695e788b
    source_path: cli/onboard.md
    workflow: 16
---

# `openclaw onboard`

以推理设置为先的引导式设置：检测现有的 AI 访问方式，要求完成一次实时补全，仅持久化可用的路由，然后启动 OpenClaw 以配置其余内容。全新系统或存在新手引导选项时，`openclaw setup` 会进入此流程；已配置的系统使用不带参数的 `openclaw setup` 进行系统智能体聊天。`openclaw setup --baseline` 仅写入基线配置和工作区。

<CardGroup cols={2}>
  <Card title="CLI 新手引导中心" href="/zh-CN/start/wizard" icon="rocket">
    交互式 CLI 流程的分步指南。
  </Card>
  <Card title="新手引导概览" href="/zh-CN/start/onboarding-overview" icon="map">
    OpenClaw 新手引导各部分如何协同工作。
  </Card>
  <Card title="CLI 设置参考" href="/zh-CN/start/wizard-cli-reference" icon="book">
    输出、内部机制和各步骤的行为。
  </Card>
  <Card title="CLI 自动化" href="/zh-CN/start/wizard-cli-automation" icon="terminal">
    非交互式标志和脚本化设置。
  </Card>
  <Card title="macOS 应用新手引导" href="/zh-CN/start/onboarding" icon="apple">
    macOS 菜单栏应用的新手引导流程。
  </Card>
</CardGroup>

## 示例

```bash
openclaw onboard
openclaw onboard --tui
openclaw onboard --classic
openclaw onboard --modern
openclaw onboard --flow quickstart
openclaw onboard --flow manual
openclaw onboard --flow import
openclaw onboard --import-from hermes --import-source ~/.hermes
openclaw onboard --skip-bootstrap
openclaw onboard recommendations --json
openclaw onboard recommendations acknowledge
openclaw onboard recommendations acknowledge --retry "<failed-id>"
openclaw onboard recommendations refresh
openclaw onboard --mode remote --remote-url wss://gateway-host:18789
```

`openclaw onboard recommendations` 读取新手引导期间存储的待处理应用推荐匹配项。添加 `--json` 可获取首次运行引导所用的机器可读列表。该命令不会重新扫描已安装的应用，也不会调用模型。其输出仅包含经过验证的安装 ID、来源和层级；它会有意省略不受信任的市场文案、模型理由和本地应用标签。应用推荐提议得到答复后，该命令会返回空列表，后续的新手引导运行将完全跳过此步骤。
`openclaw onboard recommendations refresh` 会清除已存储的提议，使下次新手引导运行重新扫描已安装的应用并创建新提议。

全新工作区会将推荐选择推迟到引导对话中。该对话处理完用户的选择后，`openclaw onboard recommendations acknowledge` 会将已存储的提议标记为已答复。
确认操作具有幂等性。如果某个选定的安装失败，请使用 `--retry <id...>` 传入每个失败的不透明 ID；成功和已拒绝的匹配项会被消耗，而失败的匹配项仍保持待处理状态，供后续新手引导运行使用。未知 ID 会导致失败，但不会更改已存储的提议。ClawHub skill 安装中断后，只有当 `openclaw skills verify "@owner/slug"` 针对同一发布者限定的推荐 ID 成功执行，且其 JSON 输出报告 `openclaw.resolution.source: "installed"` 时，现有目标才算安装成功。仅验证注册表并不能证明已在本地安装。否则，请使用 `--retry` 将该 ID 保持为待处理状态，并且不要覆盖现有 skill。

- `--classic`：打开完整的分步向导。它不能与 `--non-interactive` 组合使用；自动化设置时请省略 `--classic`。
- `--flow quickstart`：打开提示最少的经典向导，默认使用令牌身份验证；当没有适用的已存储或显式凭据时，会生成令牌。`--gateway-port`、`--gateway-bind`、`--gateway-auth` 和 `--tailscale` 等显式本地 Gateway 网关标志会覆盖相应的已存储或默认快速开始值；省略的选项会保留其当前值。
- `--flow manual`（别名 `advanced`）：打开经典向导，并完整提示输入端口、绑定和身份验证信息。
- `--flow import`：针对全新设置运行检测到的迁移提供商（例如通过 `--import-from hermes` 使用 Hermes）。确认后，新手引导会在私有临时目标下暂存配置、凭据、工作区文件、记忆和 Skills；导入的推理必须通过实时补全，之后才会提升工作区和智能体状态并提交配置。在提升前失败或取消，不会改动实时目标。无法回滚的外部激活步骤（例如安装 Codex 插件）会在之后运行，并可根据迁移报告重试。如果已存在任何配置、凭据、会话或工作区状态，请先将其重置。有关试运行计划、覆盖模式、已验证备份、报告和精确映射，请参阅 [`openclaw migrate`](/zh-CN/cli/migrate)。
- `--remote-url` 和 `--remote-token`：预填经典远程 Gateway 网关步骤，并在本次运行中覆盖已存储的远程值。更改 URL 不会复用已存储的凭据，除非同时传入令牌。令牌在提示中保持遮掩状态，并遵循向导现有的明文或 SecretRef 存储选择。
- `--tailscale-reset-on-exit` 和 `--no-tailscale-reset-on-exit`：显式控制 Gateway 网关退出时是否重置 Tailscale Serve 或 Funnel 配置。省略两者可在非交互式重新运行期间保留当前设置。
- `--modern` 是 OpenClaw 对话式设置助手的兼容性别名。它使用与 `openclaw setup` 相同的实时推理门控，并且仅接受 `--workspace`、`--accept-risk`、`--non-interactive` 和 `--json`。其他设置标志会被拒绝，而不是被静默忽略。

## 引导式流程

不带参数的 `openclaw onboard` 会启动引导式流程。它会显示安全通知，然后一开始只询问一个问题：**完全访问**（推荐——设置会自动查找 AI 应用、密钥和本地运行时）或**先询问**（设置会在查找前询问一次，或允许手动配置）。选择会持久化为 `wizard.accessMode`。允许发现后，新手引导会检测已通过配置的模型、API 密钥环境变量和受支持的本地 CLI 提供的 AI 访问方式，然后使用实际补全测试推荐候选项。如果候选项失败，新手引导会静默尝试下一个可用候选项，并在一行内汇总所有未响应的项目；可用路由会被公布，同时提供单键选项以改为查看其他所有选项。

如果自动检测已穷尽，提供商选择器会优先显示 OpenAI、Anthropic、xAI（Grok）、Google 和 OpenRouter。选择 **更多…** 可查看按提供商分组的所有其他受支持提供商；随后会在第二个菜单中显示区域、套餐和身份验证方式。受支持的浏览器或设备登录方式，以及遮掩的 API 密钥或令牌方式，都使用相同的实时补全路径。只有测试成功后，OpenClaw 才会持久化已验证的模型路由及其凭据；失败的候选项不会替换已配置的模型，也不会保存尝试使用的凭据。选择**暂时跳过**可退出而不启动 OpenClaw，并在准备就绪后重新运行 `openclaw onboard`。在 OpenClaw 启动之前，工作区和 Gateway 网关设置保持不变。

在引导模式下，`--workspace <dir>` 会提供 OpenClaw 建议的工作区和隔离的推理上下文。在批准 OpenClaw 设置提议之前，它不会被持久化。经典和非交互式新手引导会通过其常规设置流程持久化工作区。在已有智能体名册的情况下重新运行时，新手引导会保留已配置的智能体群工作区：经典向导会显示两个路径，并要求在移动前显式确认；非交互式设置则会发出警告并保留当前值。

推理通过后，新手引导会检查受支持的本地 AI 工具中的记忆：Claude Code 自动记忆、Codex 整合记忆和 Hermes 记忆文件。发现任何此类内容时，会在一个页面中提供将其复制到智能体工作区 `memory/imports/` 下以供索引检索的选项。未经确认不会导入任何内容，之前导入的文件会被跳过，并且随时可以稍后从 Control UI 的[记忆导入页面](/zh-CN/web/control-ui)导入，该页面提供相同的仅记忆范围。（完整运行 [`openclaw migrate`](/zh-CN/cli/migrate) 的范围更广：它还可以导入配置、Skills 和凭据。）经典向导会在准备好工作区后显示相同页面。

推理通过（并完成记忆导入提议）后，引导式新手引导会自动应用标准设置——工作区、Gateway 网关和会话，即对话式 `openclaw setup` 聊天在收到“是”时所应用的同一计划——然后根据已安装的应用推荐插件和 skill；应用名称会通过已配置的模型和 ClawHub 搜索进行匹配，并且可以使用 [`wizard.appRecommendations`](/zh-CN/gateway/configuration-reference#wizard) 禁用此步骤。
在 macOS、Linux 或 Windows 桌面会话中，它随后会打开经过身份验证的 Control UI 仪表板，并等待浏览器客户端连接，最长等待 60 秒。在无头 Linux 或通过 SSH 运行时，它会醒目地输出一个可复制粘贴的仪表板 URL；对于回环 Gateway 网关，还会包含 SSH 端口转发命令，并等待最长五分钟。连接成功后会在浏览器中继续；无法访问 Gateway 网关或超时，则会回退到与之前相同的终端出口。传入 `--tui` 可跳过浏览器移交并强制使用该终端出口。
如果应用设置失败，新手引导会回退到对话式 OpenClaw 聊天，以交互方式完成设置。渠道、智能体、插件和其他可选功能仍由 OpenClaw 聊天负责：运行 `openclaw`，并使用 `open channel wizard for <channel>` 将渠道凭据收集交给遮掩输入的终端向导。要更改模型提供商或其身份验证，请退出 OpenClaw 并运行 `openclaw onboard`；OpenClaw 不会打开引导式或经典提供商流程。

在已配置的安装上，再次运行 `openclaw onboard` 会先验证当前默认模型，因此同一流程可用作验证和修复过程——它不会重新应用设置、重新安装或重启 Gateway 网关服务。
如果该检查失败，已配置的模型绝不会被自动替换——新手引导会停止并询问如何继续。检查在工作区外运行，因此由工作区插件提供的模型可能会在此处失败，但仍可在智能体中正常工作。
对于特定提供商的身份验证、渠道、Skills、远程 Gateway 网关设置、导入或完整 Gateway 网关控制，请使用 `openclaw onboard --classic`。对于不涉及推理的对话式设置和修复，请运行 `openclaw setup`；`openclaw onboard
--modern` 是通过同一推理门控的兼容性别名。经典向导可选择使用实时补全验证默认模型，但在 OpenClaw 自身的实时推理检查通过之前，OpenClaw 不会启动。

在交互式终端中，不带子命令的 `openclaw` 会根据配置状态进行路由：

- 如果活动配置文件缺失或不含人工编写的设置（为空或仅包含元数据），则启动引导式新手引导。
- 如果配置文件存在但未通过验证，则启动经典新手引导路径，并提供 `openclaw doctor` 指引。OpenClaw 需要可用的推理能力，不能用于修复这种推理前状态。
- 如果配置文件有效，则打开正常的智能体 TUI。如果可访问的已配置 Gateway 网关包含智能体和模型，则会直接进入该 UI，而不经过新手引导或 OpenClaw。在已配置的安装上，可在 TUI 中使用 `/openclaw`，或使用 `openclaw setup` 进入 OpenClaw。

对于 local loopback、私有 IP 字面量、`.local` 和 Tailnet `*.ts.net` Gateway 网关 URL，可以使用明文 `ws://`。对于其他受信任的私有 DNS 名称，请在新手引导进程环境中设置 `OPENCLAW_ALLOW_INSECURE_PRIVATE_WS=1`。

## 重置

```bash
openclaw onboard --reset
openclaw onboard --reset --reset-scope full
```

`--reset` 会在运行设置前清除状态。`--reset-scope` 控制清除范围：`config`（仅配置）、`config+creds+sessions`（传入 `--reset` 但未指定范围时的默认值）或 `full`（同时重置工作区）。仅使用 `--reset-scope full` 时才会重置工作区。

## 区域设置

交互式新手引导对固定的设置文案使用 CLI 向导的区域设置。它按以下顺序采用第一个非空值：

1. `OPENCLAW_LOCALE`
2. `LC_ALL`
3. `LC_MESSAGES`
4. `LANG`
5. 英语回退

向导支持的区域设置为 `en`、`zh-CN` 和 `zh-TW`。区域设置值可以使用下划线或 POSIX 后缀形式，例如 `zh_CN.UTF-8`。产品名称、命令名称、配置键、URL、提供商 ID、模型 ID 以及插件/渠道标签保持原样。

```bash
OPENCLAW_LOCALE=zh-CN openclaw onboard
OPENCLAW_LOCALE=en openclaw onboard # 显式覆盖为英语
```

## 非交互式设置

`--non-interactive` 要求提供 `--accept-risk`（确认智能体功能强大，而完整的系统访问权限存在风险）。`--mode` 默认为 `local`。

```bash
openclaw onboard --non-interactive \
  --auth-choice custom-api-key \
  --custom-base-url "https://llm.example.com/v1" \
  --custom-model-id "foo-large" \
  --custom-api-key "$CUSTOM_API_KEY" \
  --secret-input-mode plaintext \
  --custom-compatibility openai \
  --custom-image-input
```

`--custom-api-key` 是可选的；如果省略，新手引导会检查环境中的 `CUSTOM_API_KEY`。OpenClaw 会自动将常见的视觉模型 ID（GPT-4o/4.1/5.x、Claude 3/4、Gemini、Qwen-VL、LLaVA、Pixtral 及类似模型）标记为支持图像。对于未知的自定义视觉模型 ID，请传入 `--custom-image-input`；要强制使用仅文本元数据，请传入 `--custom-text-input`。对于支持 `/v1/responses` 但不支持 `/v1/chat/completions` 的 OpenAI 兼容端点，请使用 `--custom-compatibility openai-responses`；有效值为 `openai`（默认值）、`openai-responses`、`anthropic`。

LM Studio 还有一个提供商专用的密钥标志：

```bash
openclaw onboard --non-interactive \
  --auth-choice lmstudio \
  --custom-base-url "http://localhost:1234/v1" \
  --custom-model-id "qwen/qwen3.5-9b" \
  --lmstudio-api-key "$LM_API_TOKEN" \
  --accept-risk
```

非交互式 Ollama：

```bash
openclaw onboard --non-interactive \
  --auth-choice ollama \
  --custom-base-url "http://ollama-host:11434" \
  --custom-model-id "qwen3.5:27b" \
  --accept-risk
```

`--custom-base-url` 默认为 `http://127.0.0.1:11434`。`--custom-model-id` 是可选的；如果省略，新手引导会使用 Ollama 建议的默认值。此处也支持 `kimi-k2.5:cloud` 等云模型 ID。

将提供商密钥存储为引用，而不是明文：

```bash
openclaw onboard --non-interactive \
  --auth-choice openai-api-key \
  --secret-input-mode ref \
  --accept-risk
```

使用 `--secret-input-mode ref` 时，新手引导会写入由环境变量支持的引用，而不是明文密钥值：对于由身份验证配置文件支持的提供商，它会写入 `keyRef: { source: "env", provider: "default", id: <envVar> }`；对于自定义提供商，则以相同方式写入 `models.providers.<id>.apiKey`（例如 `{ source: "env", provider: "default", id: "CUSTOM_API_KEY" }`）。约定：在新手引导进程的环境中设置提供商环境变量（例如 `OPENAI_API_KEY`），并且除非已设置该环境变量，否则不要同时传入内联密钥标志——如果提供了标志值但缺少对应的环境变量，程序会快速失败并给出指引。

### Gateway 网关身份验证（非交互式）

- `--gateway-auth token --gateway-token <token>` 存储明文令牌。`token` 是默认身份验证模式。
- `--gateway-auth token --gateway-token-ref-env <name>` 将 `gateway.auth.token` 存储为环境变量 SecretRef。要求新手引导进程的环境中存在该名称的非空环境变量。
- `--gateway-token` 与 `--gateway-token-ref-env` 互斥。
- 使用 `--install-daemon` 时：由 SecretRef 管理的 `gateway.auth.token` 会经过验证，但不会以解析后的明文形式持久化到监管服务的环境元数据中；如果无法解析该引用，安装会以关闭方式失败并提供修复指引。如果同时配置了 `gateway.auth.token` 和 `gateway.auth.password`，且未设置 `gateway.auth.mode`，安装会阻塞，直到显式设置模式。
- 本地新手引导会将 `gateway.mode="local"` 写入配置。之后如果配置文件缺少 `gateway.mode`，表示配置损坏或手动编辑不完整，而不是有效的本地模式快捷方式。
- 本地新手引导会安装所选设置路径所需的可下载插件（例如，这些身份验证选项需要的 Codex 或 Copilot 运行时插件）。远程新手引导只会写入远程 Gateway 网关的连接信息，绝不会安装本地插件包。
- `--allow-unconfigured` 是一个独立的 `openclaw gateway run` 逃生舱；它不能让新手引导跳过 `gateway.mode`。

```bash
export OPENAI_API_KEY="your-provider-key"
export OPENCLAW_GATEWAY_TOKEN="your-token"
openclaw onboard --non-interactive \
  --mode local \
  --auth-choice openai-api-key \
  --secret-input-mode ref \
  --gateway-auth token \
  --gateway-token-ref-env OPENCLAW_GATEWAY_TOKEN \
  --accept-risk
```

### 本地 Gateway 网关健康状态

- 除非传入 `--skip-health`，否则新手引导会等待本地 Gateway 网关可访问后才成功退出。
- `--install-daemon` 会先启动托管的 Gateway 网关安装路径。如果不使用它，则本地 Gateway 网关必须已在运行（例如 `openclaw gateway run`）。
- 如果在自动化中只需要写入配置/工作区/引导文件，`--skip-health` 会跳过等待。
- `--skip-bootstrap` 会设置 `agents.defaults.skipBootstrap: true`，并跳过创建 `AGENTS.md`、`SOUL.md`、`TOOLS.md`、`IDENTITY.md`、`USER.md`、`HEARTBEAT.md` 和 `BOOTSTRAP.md`。
- 在原生 Windows 上，`--install-daemon` 会先尝试使用 Scheduled Tasks；如果创建任务被拒绝，则回退到每用户 Startup 文件夹中的登录项。

### 交互式引用模式

- 出现提示时，选择 **Use secret reference**，然后选择 **Environment variable** 或已配置的机密信息提供商（`file` 或 `exec`）。
- 新手引导会在保存引用前运行快速预检验证，并允许在失败后重试。

### Z.AI 端点选项

<Note>
`--auth-choice zai-api-key` 会自动检测最适合你的密钥的 Z.AI 端点和模型：Coding Plan 端点优先使用 `zai/glm-5.2`（如果不可用，则回退到 `glm-5.1`）；通用 API 端点默认使用 `zai/glm-5.1`。要强制使用 Coding Plan 端点，请直接选择 `zai-coding-global` 或 `zai-coding-cn`。
</Note>

```bash
# 无提示选择端点
openclaw onboard --non-interactive \
  --auth-choice zai-coding-global \
  --zai-api-key "$ZAI_API_KEY"

# 其他 Z.AI 端点选项：zai-coding-cn、zai-global、zai-cn
```

Mistral：

```bash
openclaw onboard --non-interactive \
  --auth-choice mistral-api-key \
  --mistral-api-key "$MISTRAL_API_KEY"
```

## 其他非交互式标志

基于令牌的模型身份验证（与 `--auth-choice token` 配合使用）：

| 标志                            | 说明                                                                                                                 |
| ------------------------------- | --------------------------------------------------------------------------------------------------------------------------- |
| `--token-provider <id>`         | 签发令牌的令牌提供商 ID                                                                                         |
| `--token <token>`               | 用于模型身份验证的令牌值                                                                                        |
| `--token-profile-id <id>`       | 身份验证配置文件 ID（默认为 `<provider>:manual`；某些由提供商所有的流程使用其自己的默认值，例如 `anthropic:default`） |
| `--token-expires-in <duration>` | 可选的令牌有效期（例如 `365d`、`12h`）                                                                         |

Cloudflare AI Gateway：`--cloudflare-ai-gateway-account-id <id>`、`--cloudflare-ai-gateway-gateway-id <id>`。

守护进程安装控制：`--no-install-daemon` / `--skip-daemon`（别名；跳过 Gateway 网关服务安装）、`--daemon-runtime <node>`。

Skills：`--node-manager <npm|pnpm|bun>`（默认为 `npm`）、`--skip-skills`。

UI 和钩子设置：`--skip-ui`（跳过 Control UI/TUI 提示）、`--skip-hooks`（跳过 webhook/钩子设置）、`--skip-channels`、`--skip-search`。

输出：`--suppress-gateway-token-output` 会隐藏包含令牌的 Gateway 网关/UI 输出（令牌提示、嵌入令牌的自动登录 URL，以及自动启动 Control UI），适用于共享终端和 CI。

<Note>
在引导式或经典新手引导中，`--json` 并不表示非交互式模式。
使用 `--modern` 时，JSON 是一次性的 OpenClaw 概览，并会在生成该
单个结果后退出。其他脚本请使用 `--non-interactive`。
</Note>

## 提供商预筛选

当身份验证选项隐含首选提供商时，新手引导会将默认模型和允许列表选择器预筛选为该提供商的模型。该筛选器还会匹配由同一插件所有的其他提供商，其中包括 `volcengine`/`volcengine-plan` 和 `byteplus`/`byteplus-plan` 等 Coding Plan 变体。如果首选提供商筛选器没有产生任何已加载模型，新手引导会回退到未筛选的目录，而不是让选择器保持为空。

## Web 搜索后续设置

某些 Web 搜索提供商会在新手引导期间触发提供商专用的后续提示：

- **Grok** 可以提供可选的 `x_search` 设置，使用相同的 xAI 身份验证和一个 `x_search` 模型选项。
- **Kimi** 可以询问 Moonshot API 区域（`api.moonshot.ai` 或 `api.moonshot.cn`）以及默认的 Kimi Web 搜索模型。

## 其他行为

- 本地新手引导的私信范围行为：[CLI 设置参考](/zh-CN/start/wizard-cli-reference#outputs-and-internals)。
- 最快开始首次聊天的方式：`openclaw dashboard`（Control UI，无需设置渠道）。
- 自定义提供商：连接任何 OpenAI 或 Anthropic 兼容端点，包括未列出的托管提供商。使用 **Unknown** 兼容性选项，通过实时探测进行自动检测。
- 如果检测到 Hermes 状态，新手引导会提供迁移流程（请参阅上面的 `--flow import`）。

## 常用后续命令

之后可使用 `openclaw configure` 进行不涉及推断的针对性更改，并使用 `openclaw
channels add` 仅设置渠道。对于模型提供商或身份验证路由更改，
请改为运行 `openclaw onboard`。

```bash
openclaw channels add
openclaw configure
openclaw agents add <name>
```
