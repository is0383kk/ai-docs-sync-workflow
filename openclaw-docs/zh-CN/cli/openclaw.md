---
read_when:
    - 你已完成推理设置，并希望 OpenClaw 配置其余部分
    - 你需要使用本地设置智能体检查或修复 OpenClaw
    - 你正在设计或启用消息渠道救援模式
summary: 推理支持的 OpenClaw 设置和修复助手的 CLI 参考与安全模型
title: OpenClaw 设置智能体
x-i18n:
    generated_at: "2026-07-26T06:11:48Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 9578d1493ff514ea6dd07dae995bf83443e9e17f2c2134bc801faa45254615bf
    source_path: cli/openclaw.md
    workflow: 16
---

# `openclaw setup`

OpenClaw 内置了一个系统智能体——它以“OpenClaw”的身份交流——用于本地设置、修复和配置（以前称为 Crestodian）。它仅在实际生效的默认模型完成一次真实轮次后启动。
全新安装会先建立推理能力；配置格式错误时仍使用经典 Doctor 流程。

## 启动时机

运行不带子命令的 `openclaw` 时，会根据配置状态进行路由：

- 配置缺失，或配置存在但不包含用户编写的设置（为空，或仅含 `$schema`/`meta` 键）：启动带实时 AI 验证的引导式新手引导。
- 配置存在但验证失败：启动经典新手引导，它会报告问题并引导你运行 `openclaw doctor`。
- 配置存在且有效：打开正常的智能体 TUI。如果已配置的 Gateway 网关可访问，且其默认智能体具有模型，则会直接进入该 UI，而不经过新手引导或 OpenClaw。之后可在 TUI 中使用 `/openclaw`，或直接运行 `openclaw setup`，进入 OpenClaw。

运行 `openclaw setup` 会先对已配置的默认模型执行实时测试。轮次通过后启动 OpenClaw。交互模式下测试失败时，会打开引导式推理设置，并在候选项通过测试后转交给 OpenClaw。当推理不可用时，单次、JSON 和其他非交互请求会失败，并提示运行 `openclaw onboard`。`openclaw --help` 和 `openclaw --version` 仍使用其正常的快速路径。

非交互式运行不带参数的 `openclaw`（无 TTY）时，会输出一条简短消息并退出，而不是打印根帮助：对于全新安装或无效安装，它会指向非交互式新手引导；配置有效时，则指向 `openclaw agent --local ...`。

`openclaw onboard --modern` 仍是 OpenClaw 的兼容别名，但使用相同的推理门控：推理正常时打开聊天；交互模式下失败时启动引导式推理设置；非交互模式下失败时退出并提供新手引导说明。`openclaw onboard --classic` 会打开完整的分步向导。

## OpenClaw 显示的内容

交互式 OpenClaw 会打开与 `openclaw tui` 相同的 TUI 外壳，并使用 OpenClaw 聊天后端。启动问候信息包括：

- 配置有效性和默认智能体
- OpenClaw 正在使用的已验证模型
- 首次启动探测得到的 Gateway 网关可访问性
- 下一项建议的调试操作

它不会为了启动而转储密钥或加载插件 CLI 命令。

使用 `status` 可查看详细清单：配置路径、文档/源代码路径、本地 CLI 探测、密钥/令牌是否存在、智能体、模型和 Gateway 网关详细信息。

OpenClaw 使用与普通智能体相同的参考资料发现机制：在 Git 检出中，它会指向本地 `docs/` 和源代码树；在 npm 安装中，它使用内置文档并链接到 [https://github.com/openclaw/openclaw](https://github.com/openclaw/openclaw)，同时建议在文档不足时查看源代码。

## 示例

```bash
openclaw
openclaw setup
openclaw setup --json
openclaw setup --message "models"
openclaw setup --message "validate config"
openclaw setup --message "setup workspace ~/Projects/work" --yes
openclaw setup --message "set default model openai/gpt-5.6" --yes
openclaw onboard --modern
```

在 OpenClaw TUI 中：

```text
状态
健康
Doctor
验证配置
设置
设置工作区 ~/Projects/work
配置设置 gateway.port 19001
配置设置引用 gateway.auth.token 环境 OPENCLAW_GATEWAY_TOKEN
Gateway 网关状态
重启 Gateway 网关
智能体
创建智能体 work 工作区 ~/Projects/work
模型
配置模型提供商
设置默认模型 openai/gpt-5.6
渠道
渠道信息 slack
连接 slack
打开 slack 的渠道向导
插件列表
插件搜索 slack
插件安装 clawhub:openclaw-codex-app-server
与 work 智能体对话
与 ~/Projects/work 的智能体对话
审计
退出
```

## 操作和审批

OpenClaw 使用类型化操作，而不是临时编辑配置。

只读操作会立即运行：显示概览、列出智能体、列出已安装插件、搜索 ClawHub 插件、显示模型/后端状态、运行状态/健康检查、检查 Gateway 网关可访问性、运行不含交互式修复的 Doctor、验证配置以及显示审计日志路径。

启动引导式渠道设置（`connect telegram`）也会立即运行。其向导会收集明确的回答，并负责执行由此产生的写入。

持久化操作需要通过对话审批（直接命令也可使用 `--yes`）：写入配置、`config set`、`config set-ref`、设置/新手引导引导初始化、更改默认模型、启动/停止/重启 Gateway 网关、创建智能体以及安装插件。

OpenClaw 内部不提供 Doctor 修复，因为这些修复可能会重写为当前会话提供支持的提供商、身份验证或默认智能体推理路由。请退出 OpenClaw，并在终端中运行 `openclaw doctor --fix`。OpenClaw 内部仍可使用只读的 `doctor`。

新智能体会继承经过实时验证的默认推理路由。智能体 ID `openclaw` 和 `crestodian` 为系统智能体保留，不能作为普通智能体创建。已停用的 ID 仍会被阻止，以免旧配置占用该 ID。

`config set` 和 `config set-ref` 可以更改用户可更改的任何设置，
但有一份简短且仅供人工判断的拒绝列表：`$include`、`auth.*`、`env.*`、`models.*`
和 `secrets.*` 仍会被拒绝，因为它们包含凭据材料、
备用配置包含项，或用于推理路由的提供商/目录定义。
推理路由本身也受到保护：默认模型路由
（`agents.defaults` 的模型/参数/运行时字段）以及支持当前生效默认路由的智能体的路由字段
会被拒绝，智能体身份/拓扑字段（`id`、`agentDir`、`default`）也是如此。其他智能体的路由字段
经审批后仍可写入。Gateway 网关和渠道身份验证仍属于
普通配置表面。对于已配置的路由，请使用 `set default model <provider/model>`；
它会在保存前实时测试该路由。若要配置或修复提供商/身份验证访问权限，请退出 OpenClaw 并运行
`openclaw onboard`。

允许执行 `plugins.entries.<id>.*` 写入（启用/禁用/配置已安装的插件），
除非该插件支持当前生效的推理路由。插件安装来源和加载策略
仍在类型化插件安装工作流中保持其信任边界。出于同样原因，
也会拒绝卸载支持该路由的插件；请退出 OpenClaw，并在终端中运行
`openclaw plugins uninstall <id>`。

审批使用你自己的措辞给出：明确的回复（“是”“可以”“继续”“暂时不要”）会通过一个封闭的确定性列表进行解析。当已配置的路由支持单独的补全调用时，其他回复可以仅根据你的消息和待处理提案进行分类——绝不会由对话模型本身分类，因为它不能自行批准。无法分类或含糊的回复会使提案保持待处理状态，并由对话再次询问。

### 更改历史

Ask OpenClaw 页面可以显示最近应用的系统智能体操作、Doctor
迁移、设置和 CLI 配置写入，以及对
`openclaw.json` 的手动编辑。当 Gateway 网关正在监视时、OpenClaw
执行自有写入期间，或离线编辑后的下次启动时，配置日志会检测外部编辑。

历史记录存储在共享
`~/.openclaw/state/openclaw.sqlite` 数据库的 `diagnostic_events` 表中，位于 `system-agent-audit`
和 `config-audit` 作用域下。每个作用域保留最新的 50,000 条记录。
设备发现和只读操作不包含在内。密钥绝不会出现在更改历史中；
配置日志记录包含更改的路径而非配置值，并且值比较使用受保护的指纹。

渠道设置可以作为托管对话运行，直到需要输入密钥。
本地 OpenClaw TUI 不接受敏感的向导回答，因为终端聊天输入可见。
它会立即提供 `open channel wizard`，将所选渠道带入
使用掩码输入的终端向导；你也可以稍后运行
`openclaw channels add --channel <channel>`。

### 切换到掩码渠道设置

本地聊天可以将控制权交给使用掩码输入的渠道向导：

```text
打开 slack 的渠道向导
渠道信息 slack
```

`open channel wizard for <channel>` 会在聊天 TUI 关闭后打开使用掩码输入的渠道设置。
请先使用 `channel info <channel>` 查看渠道标签、设置状态、
先决条件摘要和文档链接。

OpenClaw 绝不会在自身会话中更改提供商/身份验证访问权限：
该会话已经依赖此推理路由。对于模型提供商设置或修复，
`configure model provider` 会返回退出/新手引导说明，而不会
启动向导或写入配置。退出 OpenClaw 并运行 `openclaw
onboard`；
新手引导会暂存凭据，并且只保存能够完成一次真实实时轮次的路由。
新手引导成功后，再次启动 OpenClaw。

## 设置引导初始化

在引导式新手引导已经建立推理能力后，`setup` 会配置其余工作区和 Gateway 网关状态。它只通过类型化配置操作进行写入，并会先请求审批。

```text
设置
设置工作区 ~/Projects/work
```

`setup` 会保留已验证的实际生效模型。它不会配置或
替换推理设置。

如果缺少推理能力或实时检查失败，请退出 OpenClaw 并运行 `openclaw onboard`。引导式新手引导会依次尝试已配置的模型、已通过身份验证的订阅 CLI、API 密钥和其余受支持的 CLI；它会要求每个候选项给出真实回复，并且只持久化通过测试的路由。越过这一边界后，OpenClaw 会立即启动，随后可以配置工作区、Gateway 网关、渠道、智能体、插件和其他可选功能。

当 macOS 应用连接到已配置的 Gateway 网关，且其默认智能体已经配置模型时，
会完全跳过这一阶梯，直接打开正常的智能体
UI。
对于全新或不完整的 Gateway 网关，应用会通过
`openclaw.setup.detect` 和 `openclaw.setup.activate` Gateway 网关方法驱动推理阶梯：
detect 会列出找到的每个候选后端，activate 会实时测试一个
候选项（一次真实的“回复 OK”补全），并且仅在测试通过后持久化
该路由所需的模型、凭据和提供商/运行时状态。工作区和 Gateway 网关默认设置仍由 OpenClaw 处理。失败的候选项
绝不会更改配置；应用会自动沿阶梯向下尝试，最后
提供一个手动密钥/令牌步骤，其中的选项来自 Gateway 网关当前生效的
文本推理提供商插件。所选提供商负责其起始模型
和配置，凭据也会通过相同方式验证后再保存。

Codex 监管和其他可选插件功能不属于此
推理激活事务。仅在推理正常工作且 OpenClaw
已启动后配置这些功能；推理设置期间不会改动现有插件策略和明确的
监管退出设置。

## AI 对话

交互式 OpenClaw 的自由形式对话通过与普通 OpenClaw 智能体相同的智能体循环运行，但仅限使用一个具有最高权限的 OpenClaw 权限工具 `openclaw`，该工具封装了类型化操作。读取操作可自由运行；变更操作需要你针对该项具体操作进行对话审批（参见“操作和审批”）；每项已应用的写入都会被审计并重新验证。智能体会话会持久化，因此 OpenClaw 具有真正的多轮记忆。如果已验证的推理路由之后停止工作，请返回 `openclaw onboard` 修复它，然后再继续。

宿主不会将自然语言请求解析为操作。自由形式
消息——包括看似命令的文本和“为什么我的
Gateway 网关停止了？”之类的问题——会发送给 AI，后者可以通过
`openclaw` 工具将请求映射为类型化操作。

当变更操作处于待处理状态时，仅会直接解析封闭列表中含义明确的批准或拒绝短语，而不进行推断。含糊的同意会交由单独配置的补全调用处理，否则将以失败关闭。结构化向导字段和精确的主机导航属于 UI 控件，而不是自然语言操作解析。有一项机密信息卫生例外尤为重要：敏感路径（令牌、密钥、密码）上的精确 `config set` 永远不会发送给模型。主机会创建经过脱敏的提案，并在 AI 可见的历史记录中遮蔽该值。对于机密信息，优先使用 `config set-ref <path> env <ENV_VAR>`。

消息渠道救援模式绝不会使用模型辅助规划器。远程救援保持确定性，因此无法利用已损坏或遭入侵的常规智能体路径将其用作配置编辑器。

### CLI harness 信任模型

嵌入式运行时和 Codex app-server harness 会直接强制执行零环限制：该运行携带一个 OpenClaw 工具允许列表，其中仅包含 `openclaw` 工具。对于 Codex，OpenClaw 还会为该运行禁用环境、原生执行、多智能体、目标、应用/插件、技能/MCP、Web 搜索和 `request_user_input` 表面。Codex 仍会注入其无操作能力的原生 `update_plan` 实用工具；它可以更新模型的临时检查清单，但无法写入文件或 OpenClaw 配置。CLI harness 不使用 OpenClaw 的允许列表，因此 OpenClaw 仅允许自身工具选择契约能够证明同等限制的后端：

- 可选择的后端（包括 Claude Code）启动时采用空的原生工具选择，并仅启用一个 MCP 工具 `openclaw`。Claude 生成的 MCP 配置通过 `--strict-mcp-config` 应用，因此不会加载其他 MCP 服务器。
- 声明不提供原生工具的后端会获得同一个专用 OpenClaw MCP 服务器。
- 始终启用原生工具或原生工具情况未知的后端会在推断前以失败关闭；它们无法托管 OpenClaw 会话。

只有 OpenClaw 会话会获得 openclaw MCP 服务器；常规智能体运行永远不会看到此工具。因此，可选择/无原生工具的 CLI 后端和 API 密钥模型会强制执行严格的单工具循环。Codex app-server 模型会强制仅使用一个 OpenClaw 权限工具，外加无操作能力的原生规划实用工具。在这三种情况下，设置写入都仍被限制在 OpenClaw 经审计的审批契约内。

Gemini CLI 仍可供常规智能体使用，但它无法强制执行推断门控所需的无工具探测，因此无法托管 OpenClaw。

## 切换到智能体

使用自然语言选择器离开 OpenClaw 并打开常规 TUI：

```text
与智能体交谈
与工作智能体交谈
切换到主智能体
```

`openclaw tui`、`openclaw chat` 和 `openclaw terminal` 会直接打开常规智能体 TUI；它们不会启动 OpenClaw。切换到常规 TUI 后，`/openclaw` 会返回 OpenClaw，并可选择附带后续请求：

```text
/openclaw
/openclaw restart gateway
```

## 消息救援模式

消息救援模式是 OpenClaw 的消息渠道入口点：当常规智能体已停止工作，但受信任的渠道（例如 WhatsApp）仍能接收命令时，可使用此模式。

这是一个确定性的紧急命令处理程序，而不是对话式 OpenClaw 智能体。它不会引导全新设置，也不会放宽 OpenClaw 聊天的推断门控。

支持的命令：`/openclaw <request>`。救援模式仅接受精确键入的命令语法——自然语言会被拒绝并返回提示，绝不会将其猜测为某项操作，也绝不会调用模型。

```text
你，在受信任的所有者私信中：/openclaw status
OpenClaw：OpenClaw 救援模式。Gateway 网关可达：否。配置有效：否。
你：/openclaw restart gateway
OpenClaw：计划：重启 Gateway 网关。回复 /openclaw yes 以应用。
你：/openclaw yes
OpenClaw：已应用。已写入审计条目。
```

还可以在本地或通过救援模式将智能体创建操作加入队列：

```text
create agent work workspace ~/Projects/work model openai/gpt-5.6-sol
/openclaw create agent work workspace ~/Projects/work
```

创建智能体时只能指定当前已实时验证的默认模型。省略模型即可继承该路由。

远程救援属于管理表面，必须将其视为远程配置修复，而不是常规聊天。

远程救援的安全契约：

- 当智能体/会话启用沙箱隔离时禁用；OpenClaw 会拒绝远程救援，并引导使用本地 CLI 修复。
- 默认有效状态为 `auto`：仅允许在受信任的 YOLO 操作中进行远程救援，此时运行时已拥有未沙箱隔离的本地权限（`tools.exec.security` 解析为 `full`，`tools.exec.ask` 解析为 `off`，沙箱模式为 `off`）。
- 需要明确的所有者身份；不允许通配符发送者规则、开放群组策略、未经身份验证的 Webhooks 或匿名渠道。
- 救援仅限所有者私信。
- 插件搜索和列表操作为只读。插件安装始终仅限本地（即使其他情况下已启用，在救援模式中也会被阻止），因为它会下载可执行代码。本地 OpenClaw 和救援模式都会拒绝卸载插件；请从终端运行 `openclaw plugins uninstall <id>`。
- 远程救援无法打开本地 TUI，也无法切换到交互式智能体会话；智能体交接请使用本地 `openclaw`。
- 即使在救援模式下，持久化写入仍需审批。
- 待处理的审批仅可使用一次。对于同一账户、渠道和发送者，任何更新的救援命令都会撤销旧计划；执行失败也会消耗审批，因此如需重试，请重新发送该命令。
- 每项已应用的救援操作都会被审计。消息渠道救援会记录渠道、账户、发送者和源地址元数据；修改配置的操作还会记录修改前后的配置哈希值。
- 绝不会回显机密信息。SecretRef 检查仅报告是否可用，不报告具体值。
- 如果 Gateway 网关处于运行状态，救援模式会优先使用 Gateway 网关的类型化操作；如果 Gateway 网关已停止工作，救援模式仅使用不依赖常规智能体循环的最小本地修复表面。

救援策略为内置策略：仅当有效运行时为 YOLO、沙箱隔离已关闭且请求来自所有者私信时才可使用。待处理的写入审批会在 15 分钟后过期。`openclaw doctor --fix` 会移除已停用的 `systemAgent` 和 `crestodian` 配置块。

Docker 通道涵盖远程救援：

```bash
pnpm test:docker:system-agent-rescue
```

一项可选启用的实时渠道命令表面冒烟测试会检查 `/openclaw status`，以及通过救援处理程序完成的一次持久化审批往返：

```bash
pnpm test:live:system-agent-rescue-channel
```

受推断门控保护的打包版一次性设置由以下测试涵盖：

```bash
pnpm test:docker:system-agent-first-run
```

该打包版 CLI 通道从空状态目录开始，并证明 OpenClaw 在无法推断时会以失败关闭。随后，它会通过打包的激活模块测试并激活模拟 Claude。只有在此之后，模糊请求才会到达规划器并解析为类型化设置，接着执行一次性命令来创建额外智能体、通过启用插件并设置令牌 SecretRef 来配置 Discord、验证配置并检查审计日志。此通道提供门控/操作的辅助证据；它不测试交互式新手引导，也不测试 OpenClaw 智能体/工具/审批对话。下方的 QA Lab 场景会重定向到同一个 Docker 通道：

```bash
pnpm openclaw qa suite --scenario system-agent-ring-zero-setup
```

## 相关内容

- [CLI 参考](/zh-CN/cli)
- [Doctor](/zh-CN/cli/doctor)
- [TUI](/zh-CN/cli/tui)
- [沙箱](/zh-CN/cli/sandbox)
- [安全性](/zh-CN/cli/security)
