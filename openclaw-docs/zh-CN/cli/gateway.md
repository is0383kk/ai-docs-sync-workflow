---
read_when:
    - 从 CLI 运行 Gateway 网关（开发环境或服务器）
    - 调试 Gateway 网关身份验证、绑定模式和连接性
    - 通过 Bonjour 发现 Gateway 网关（本地 + 广域 DNS-SD）
    - 集成外部 Gateway 网关进程监督器
sidebarTitle: Gateway
summary: OpenClaw Gateway CLI（`openclaw gateway`）— 运行、查询和发现 Gateway 网关
title: Gateway 网关
x-i18n:
    generated_at: "2026-07-26T06:11:00Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 0188d7c79571ebf8f350295775625533a83cb2eb909bcc8763e8ce81806d2214
    source_path: cli/gateway.md
    workflow: 16
---

Gateway 网关是 OpenClaw 的 WebSocket 服务器（渠道、节点、会话、钩子）。以下所有子命令均位于 `openclaw gateway ...` 下。

<CardGroup cols={3}>
  <Card title="Bonjour 设备发现" href="/zh-CN/gateway/bonjour">
    本地 mDNS + 广域 DNS-SD 设置。
  </Card>
  <Card title="设备发现概览" href="/zh-CN/gateway/discovery">
    OpenClaw 如何通告和查找 Gateway 网关。
  </Card>
  <Card title="配置" href="/zh-CN/gateway/configuration">
    顶层 Gateway 网关配置键。
  </Card>
</CardGroup>

## 运行 Gateway 网关

```bash
openclaw gateway
openclaw gateway run   # 等效的显式形式
```

<AccordionGroup>
  <Accordion title="启动行为">
    - 除非在 `~/.openclaw/openclaw.json` 中设置了 `gateway.mode=local`，否则拒绝启动。临时/开发运行请使用 `--allow-unconfigured`；它会绕过此防护，但不会写入或修复配置。
    - 如果启动时发现可修复的无效配置，交互式终端会询问是否运行 `openclaw doctor --fix`，并在获得同意后重试启动一次。非交互式运行绝不会自动修复；它们只会输出该命令。如果修复后的配置仍然无效，启动仍会停止。
    - `openclaw onboard --mode local` 和 `openclaw setup` 会写入 `gateway.mode=local`。如果配置文件存在但缺少 `gateway.mode`，系统会将其视为配置损坏或被覆盖，Gateway 网关会拒绝替你猜测 `local`——请重新运行新手引导、手动设置该键，或传入 `--allow-unconfigured`。
    - 禁止在没有身份验证的情况下绑定到 loopback 以外的地址。
    - `--bind` 的值 `lan`、`tailnet` 和 `custom` 目前通过仅 IPv4 的路径解析；仅 IPv6 的自带主机设置需要在 Gateway 网关前放置 IPv4 sidecar 或代理。
    - `SIGUSR1` 在获得授权时会触发进程内重启。`commands.restart`（默认：启用）控制外部发送的 `SIGUSR1`；将其设置为 `false` 可阻止手动操作系统信号重启。面向智能体的 `gateway` 工具为只读；智能体通过需人工批准的 `openclaw` 委派工具请求重启。
    - `SIGINT`/`SIGTERM` 会停止进程，但不会恢复自定义终端状态——如果你将 CLI 包装在 TUI 或原始输入模式中，请在退出前自行恢复终端。

  </Accordion>
</AccordionGroup>

### 选项

<ParamField path="--port <port>" type="number">
  WebSocket 端口（默认值来自配置/环境；通常为 `18789`）。
</ParamField>
<ParamField path="--bind <mode>" type="string">
  绑定模式：`loopback`（默认）、`lan`、`tailnet`、`auto`、`custom`。
</ParamField>
<ParamField path="--token <token>" type="string">
  用于 `connect.params.auth.token` 的共享令牌。设置 `OPENCLAW_GATEWAY_TOKEN` 后默认使用其值。
</ParamField>
<ParamField path="--auth <mode>" type="string">
  身份验证模式：`none`、`token`、`password`、`trusted-proxy`。
</ParamField>
<ParamField path="--password <password>" type="string">
  用于 `--auth password` 的密码。
</ParamField>
<ParamField path="--password-file <path>" type="string">
  从文件读取 Gateway 网关密码。
</ParamField>
<ParamField path="--tailscale <mode>" type="string">
  Tailscale 暴露方式：`off`、`serve`、`funnel`。
</ParamField>
<ParamField path="--tailscale-reset-on-exit" type="boolean">
  关闭时重置 Tailscale serve/funnel 配置。
</ParamField>
<ParamField path="--allow-unconfigured" type="boolean">
  启动时不强制要求 `gateway.mode=local`。仅用于临时/开发引导；不会持久化或修复配置。
</ParamField>
<ParamField path="--dev" type="boolean">
  如果缺失，则创建开发配置和工作区（跳过 `BOOTSTRAP.md`）。
</ParamField>
<ParamField path="--dev-ambient-channels" type="boolean">
  允许开发 Gateway 网关根据环境中的环境变量自动配置渠道。需要 `--dev`。
</ParamField>
<ParamField path="--reset" type="boolean">
  重置开发配置、凭据、会话和工作区。需要 `--dev`。
</ParamField>
<ParamField path="--force" type="boolean">
  启动前终止目标端口上的所有现有监听器。在非交互式 shell 中，此选项会拒绝终止已验证的 Gateway 网关监听器；请改用 `--dev`，或使用具有空闲端口的隔离 `--profile`。
</ParamField>
<ParamField path="--verbose" type="boolean">
  将详细日志输出到 stdout/stderr。
</ParamField>
<ParamField path="--cli-backend-logs" type="boolean">
  控制台中仅显示 CLI 后端日志（同时启用 stdout/stderr）。
</ParamField>
<ParamField path="--ws-log <style>" type="string" default="auto">
  WebSocket 日志样式：`auto`、`full`、`compact`。
</ParamField>
<ParamField path="--compact" type="boolean">
  `--ws-log compact` 的别名。
</ParamField>
<ParamField path="--raw-stream" type="boolean">
  将原始模型流事件记录为 JSONL。
</ParamField>
<ParamField path="--raw-stream-path <path>" type="string">
  原始流 JSONL 路径。
</ParamField>

`--claude-cli-logs` 是 `--cli-backend-logs` 已弃用的别名。

对于 `--bind custom`，请将 `gateway.customBindHost` 设置为 IPv4 地址。除 `127.0.0.1` 或 `0.0.0.0` 以外的任何地址，还要求在同一端口提供 `127.0.0.1`，以供同一主机上的客户端使用；如果任一监听器无法绑定，启动将失败。通配符 `0.0.0.0` 不会新增一个单独的必需别名。仅 IPv6 的自带主机设置需要在 Gateway 网关前放置 IPv4 sidecar 或代理。

## 重启 Gateway 网关

```bash
openclaw gateway restart
openclaw gateway restart --safe
openclaw gateway restart --safe --skip-deferral
openclaw gateway restart --force
openclaw gateway restart --wait 30s
```

`--safe` 会请求正在运行的 Gateway 网关预检活动工作，并在这些工作排空后安排一次合并重启。等待时间上限为 5 分钟；预算到期后将强制重启。`--safe` 不能与 `--force` 或 `--wait` 组合使用。

`--skip-deferral` 会在安全重启时绕过活动工作延迟门控，因此即使报告了阻塞项，Gateway 网关也会立即重启。它需要 `--safe`——当延迟因失控任务而卡住时使用。

`--wait <duration>` 会覆盖普通（非安全）重启的排空预算。接受不带单位的毫秒值或单位后缀 `ms`、`s`、`m`、`h`、`d`（例如 `30s`、`5m`、`1h30m`）；`--wait 0` 表示无限期等待。不能与 `--force` 或 `--safe` 组合使用。

`--force` 会跳过活动工作排空并立即重启。普通的 `restart`（不带标志）会保留现有的服务管理器重启行为。

<Warning>
内联的 `--password` 可能会暴露在本地进程列表中。优先使用 `--password-file`、环境变量或由 SecretRef 支持的 `gateway.auth.password`。
</Warning>

### 外部监管器

仅当另一个进程管理器负责 Gateway 网关生命周期时，才设置 `OPENCLAW_SUPERVISOR_MODE=external`。在此模式下：

- `openclaw gateway restart` 会保留现有的安全、强制和有界等待行为，但目标改为已验证的正在运行的 Gateway 网关，而不是 launchd、systemd 或 Task Scheduler。
- 系统会拒绝原生服务安装、启动、停止和卸载操作，并提示使用外部监管器。
- 系统会拒绝 OpenClaw 自更新，以便监管器可以停止 Gateway 网关、替换并完成运行时更新，然后安全地重新启动。
- 新进程重启会在干净退出前写入一个有界的 SQLite 交接记录。如果持久化失败，Gateway 网关会回退到进程内重启，而不是在没有可用交接记录的情况下退出。

`OPENCLAW_SERVICE_REPAIR_POLICY=external` 仍是独立的 Doctor 修复策略。它不声明运行时所有权；同时需要这两种行为的监管器应设置这两个变量。

外部监管器可以通过以下隐藏的机器契约协商并使用重启交接：

```bash
openclaw gateway restart-handoff capabilities --json
openclaw gateway restart-handoff consume --expected-pid <pid> --json
```

协议版本 `1` 支持 `consume` 操作。使用操作会在一个立即执行的 SQLite 事务中验证预期 PID 和有界交接字段。已接受的交接会在返回成功结果前被删除，因此并发或重放的使用者无法同时接受它。PID 不匹配的记录会为匹配的所有者保留；缺失、过期和无效的行不会授权重启。

有效的机器请求会返回退出代码为 `0` 的 JSON，包括无需重启的结果。无效参数返回 `reason: "invalid-expected-pid"`，退出代码为 `2`；状态存储失败返回 `reason: "store-unavailable"`，退出代码为 `1`。监管器应在将要使用的确切运行时或启动器上探测 `capabilities`，而不是根据 OpenClaw 版本字符串推断支持情况或直接读取私有 SQLite 架构。

### Gateway 网关性能分析

- `OPENCLAW_GATEWAY_STARTUP_TRACE=1` 会记录启动期间的各阶段耗时，包括每个阶段的 `eventLoopMax` 延迟和插件查找表耗时（已安装索引、清单注册表、启动规划、所有者映射工作）。
- `OPENCLAW_GATEWAY_RESTART_TRACE=1` 会记录重启范围内的 `restart trace:` 行：信号处理、活动工作排空、关闭阶段、下次启动、就绪耗时和内存指标。
- `OPENCLAW_DIAGNOSTICS=timeline` 与 `OPENCLAW_DIAGNOSTICS_TIMELINE_PATH=<path>` 配合使用时，会为外部 QA 工具框架尽力写入 JSONL 启动诊断时间线（等效于配置 `diagnostics.flags: ["timeline"]`；路径仍只能通过环境变量设置）。添加 `OPENCLAW_DIAGNOSTICS_EVENT_LOOP=1` 可包含事件循环采样。
- 先运行 `pnpm build`，再运行 `pnpm test:startup:gateway -- --runs 5 --warmup 1`，可针对已构建的 CLI 入口对 Gateway 网关启动进行基准测试：首次进程输出、`/healthz`、`/readyz`、启动跟踪耗时、事件循环延迟和插件查找表耗时。
- 先运行 `pnpm build`，再运行 `pnpm test:restart:gateway -- --case skipChannels --runs 1 --restarts 5`，可在 macOS 或 Linux 上对进程内重启进行基准测试（Windows 不支持；重启需要 `SIGUSR1`）。它使用 `SIGUSR1`，在子进程中启用两种跟踪，并记录下一个 `/healthz`、下一个 `/readyz`、停机时间、就绪耗时、CPU、RSS 和重启跟踪指标。
- `/healthz` 表示存活状态；`/readyz` 表示可用就绪状态。应将跟踪行和基准测试输出视为所有者归因信号，而不是基于单个时间跨度或样本得出的完整性能结论。

## 查询正在运行的 Gateway 网关

所有查询命令均使用 WebSocket RPC。

<Tabs>
  <Tab title="输出模式">
    - 默认：便于人类阅读（在 TTY 中带颜色）。
    - `--json`：机器可读的 JSON（无样式/加载动画）。
    - `--no-color`（或 `NO_COLOR=1`）：禁用 ANSI，同时保留人类可读布局。

  </Tab>
  <Tab title="共享选项">
    - `--url <url>`：Gateway 网关 WebSocket URL。
    - `--token <token>`：Gateway 网关令牌。
    - `--password <password>`：Gateway 网关密码。
    - `--timeout <ms>`：超时/预算（默认值因命令而异；请参阅下方各命令）。
    - `--expect-final`：等待“最终”响应（智能体调用）。

  </Tab>
</Tabs>

<Note>
设置 `--url` 后，CLI 不会回退使用配置或环境中的凭据。请显式传入 `--token` 或 `--password`。缺少显式凭据会导致错误。
</Note>

### `gateway health`

```bash
openclaw gateway health --url ws://127.0.0.1:18789
openclaw gateway health --port 18789
```

`/healthz` 是存活探针：只要服务器能够响应 HTTP，它就会立即返回。`/readyz` 更严格，在启动插件 sidecar、渠道或已配置的钩子仍处于稳定过程中时，它会保持红色。来自本地或经过身份验证的详细 `/readyz` 响应包含一个 `eventLoop` 诊断块（延迟、利用率、CPU 核心比率、`degraded` 标志）。

<ParamField path="--port <port>" type="number">
  以此端口上的 local loopback Gateway 网关为目标。为本次调用覆盖 `OPENCLAW_GATEWAY_URL` 和 `OPENCLAW_GATEWAY_PORT`。
</ParamField>

### `gateway usage-cost`

从会话日志中获取用量成本摘要。

```bash
openclaw gateway usage-cost
openclaw gateway usage-cost --days 7
openclaw gateway usage-cost --agent work --json
openclaw gateway usage-cost --all-agents
openclaw gateway usage-cost --json
```

<ParamField path="--days <days>" type="number" default="30">
  要包含的天数。
</ParamField>
<ParamField path="--agent <id>" type="string">
  将摘要范围限定为一个已配置的智能体 ID。
</ParamField>
<ParamField path="--all-agents" type="boolean">
  汇总所有已配置的智能体。不能与 `--agent` 组合使用。
</ParamField>

### `gateway stability`

从正在运行的 Gateway 网关获取近期的诊断稳定性记录。

```bash
openclaw gateway stability
openclaw gateway stability --type payload.large
openclaw gateway stability --bundle latest
openclaw gateway stability --bundle latest --export
openclaw gateway stability --json
```

<ParamField path="--limit <limit>" type="number" default="25">
  要包含的最大近期事件数（最大值为 `1000`）。
</ParamField>
<ParamField path="--type <type>" type="string">
  按诊断事件类型筛选，例如 `payload.large` 或 `diagnostic.memory.pressure`。
</ParamField>
<ParamField path="--since-seq <seq>" type="number">
  仅包含诊断序列号之后的事件。
</ParamField>
<ParamField path="--bundle [path]" type="string">
  读取持久化的稳定性包，而不是调用正在运行的 Gateway 网关。`--bundle latest`（或仅使用 `--bundle`）会选择状态目录下最新的包；也可以直接传入包的 JSON 路径。
</ParamField>
<ParamField path="--export" type="boolean">
  写入可共享的支持诊断 zip，而不是打印稳定性详情。
</ParamField>
<ParamField path="--output <path>" type="string">
  `--export` 的输出路径。
</ParamField>

<AccordionGroup>
  <Accordion title="隐私和包行为">
    - 记录会保留操作元数据：事件名称、计数、字节大小、内存读数、队列/会话状态、审批 ID、渠道/插件名称和经过脱敏的会话摘要。它们不包含聊天文本、webhook 正文、工具输出、原始请求/响应正文、令牌、Cookie、密钥值、主机名和原始会话 ID。设置 `diagnostics.enabled: false` 可完全禁用记录器。
    - 当记录器中有事件时，Gateway 网关致命退出、关闭超时和重启启动失败会将同一诊断快照写入 `~/.openclaw/logs/stability/openclaw-stability-*.json`。使用 `openclaw gateway stability --bundle latest` 检查最新的包；`--limit`、`--type` 和 `--since-seq` 也适用于包输出。

  </Accordion>
</AccordionGroup>

### `gateway diagnostics export`

写入专为错误报告设计的本地诊断 zip。有关隐私模型和包内容，请参阅[诊断导出](/zh-CN/gateway/diagnostics)。

```bash
openclaw gateway diagnostics export
openclaw gateway diagnostics export --output openclaw-diagnostics.zip
openclaw gateway diagnostics export --json
```

<ParamField path="--output <path>" type="string">
  输出 zip 路径。默认为状态目录下的支持导出文件。
</ParamField>
<ParamField path="--log-lines <count>" type="number" default="5000">
  要包含的已清理日志的最大行数。
</ParamField>
<ParamField path="--log-bytes <bytes>" type="number" default="1000000">
  要检查的最大日志字节数。
</ParamField>
<ParamField path="--url <url>" type="string">
  用于健康快照的 Gateway 网关 WebSocket URL。
</ParamField>
<ParamField path="--token <token>" type="string">
  用于健康快照的 Gateway 网关令牌。
</ParamField>
<ParamField path="--password <password>" type="string">
  用于健康快照的 Gateway 网关密码。
</ParamField>
<ParamField path="--timeout <ms>" type="number" default="3000">
  状态/健康快照超时时间。
</ParamField>
<ParamField path="--no-stability-bundle" type="boolean">
  跳过持久化稳定性包查找。
</ParamField>
<ParamField path="--json" type="boolean">
  以 JSON 格式打印写入的路径、大小和清单。
</ParamField>

导出包包括：`manifest.json`（文件清单）、`summary.md`（Markdown 摘要）、`diagnostics.json`（顶层配置/日志/设备发现/稳定性/状态/健康摘要）、`config/sanitized.json`、`status/gateway-status.json`、`health/gateway-health.json`、`logs/openclaw-sanitized.jsonl`，以及在包存在时的 `stability/latest.json`。

它专为共享而设计。它会保留对调试有用的操作详情——安全日志字段、子系统名称、状态码、持续时间、已配置模式、端口、插件/提供商 ID、非密钥功能设置和经过脱敏的操作日志消息——并省略或脱敏聊天文本、webhook 正文、工具输出、凭据、Cookie、账号/消息标识符、提示词/指令文本、主机名和密钥值。当日志消息看起来像用户/聊天/工具载荷文本（例如“用户说”“聊天文本”“工具输出”“webhook 正文”）时，导出只保留消息已被省略这一事实及其字节数。

### `gateway status`

显示 Gateway 网关服务（launchd/systemd/schtasks）以及可选的连接性/身份验证探针。

```bash
openclaw gateway status
openclaw gateway status --json
openclaw gateway status --require-rpc
```

<ParamField path="--url <url>" type="string">
  添加显式探针目标。仍会探测已配置的远程目标和 localhost。
</ParamField>
<ParamField path="--token <token>" type="string">
  探针使用的令牌身份验证。
</ParamField>
<ParamField path="--password <password>" type="string">
  探针使用的密码身份验证。
</ParamField>
<ParamField path="--timeout <ms>" type="number" default="10000">
  探针超时时间。
</ParamField>
<ParamField path="--no-probe" type="boolean">
  跳过连接性探针（仅查看服务）。
</ParamField>
<ParamField path="--deep" type="boolean">
  同时扫描系统级服务。
</ParamField>
<ParamField path="--require-rpc" type="boolean">
  将连接性探针升级为读取探针，并在失败时以非零状态退出。不能与 `--no-probe` 组合使用。
</ParamField>

<AccordionGroup>
  <Accordion title="状态语义">
    - 即使本地 CLI 配置缺失或无效，也仍可用于诊断。
    - 默认输出会验证服务状态、WebSocket 连接和握手时可见的身份验证能力，而不是读取/写入/管理操作。
    - 对于首次设备身份验证，探针不会产生修改：存在已缓存的设备令牌时会复用它，但绝不会仅为检查状态而创建新的 CLI 设备身份或只读配对记录。
    - 探针身份验证会尽可能解析已配置的身份验证 SecretRef。如果所需的 SecretRef 未解析，当探针连接/身份验证失败时，`--json` 会报告 `rpc.authWarning`；请显式传入 `--token`/`--password` 或修复密钥来源。一旦探针成功，就会抑制未解析身份验证警告。
    - 当正在运行的 Gateway 网关报告 `gateway.version` 时，JSON 输出会包含它；如果握手探针无法提供版本元数据，`--require-rpc` 可以回退到 `status.runtimeVersion` RPC 载荷。
    - 当监听中的服务还不够，并且还需要读取权限范围的 RPC 保持健康时，请在脚本/自动化中使用 `--require-rpc`。
    - `--deep` 会扫描额外的 launchd/systemd/schtasks 安装；发现多个类似 Gateway 网关的服务时，人类可读输出会打印清理提示（通常每台机器运行一个 Gateway 网关），并在相关情况下报告近期的监督程序重启交接。
    - `--deep` 还会以插件感知模式（`pluginValidation: "full"`）运行配置验证，并显示插件清单警告（例如缺少渠道配置元数据）。默认的 `gateway status` 会保留跳过插件验证的快速只读路径。
    - 人类可读输出包括解析后的文件日志路径，以及 CLI 与服务的配置路径/有效性，以帮助诊断配置文件或状态目录偏移。
    - 人类可读输出包括 `Gateway heap:`，其中含有已应用的限制及其自适应推导。JSON 输出通过 `service.gatewayHeap` 提供同一报告。

  </Accordion>
  <Accordion title="Linux systemd 身份验证偏移检查">
    - 服务身份验证偏移检查会从单元中读取 `Environment=` 和 `EnvironmentFile=`（包括 `%h`、带引号的路径、多个文件和可选的 `-` 文件）。
    - 使用合并后的运行时环境解析 `gateway.auth.token` SecretRef（优先使用服务命令环境，然后回退到进程环境）。
    - 当令牌身份验证实际上未启用时，令牌偏移检查会跳过配置令牌解析（`gateway.auth.mode` 显式为 `password`/`none`/`trusted-proxy`，或者模式未设置、密码可以胜出且没有令牌候选项可以胜出）。

  </Accordion>
</AccordionGroup>

### `gateway probe`

“调试所有内容”命令。它始终会探测：

- 已配置的远程 Gateway 网关（如果已设置），以及
- localhost（环回地址），**即使已配置远程目标**。

传入 `--url` 会将该显式目标添加到两者之前。人类可读输出会将目标标记为 `URL (explicit)`、`Remote (configured)` / `Remote (configured, inactive)` 和 `Local loopback`。

<Note>
如果多个探针目标可访问，则会全部打印。SSH 隧道、TLS/代理 URL 和已配置的远程 URL 即使使用不同的传输端口，也可以指向同一个 Gateway 网关；`multiple_gateways` 保留用于不同或身份不明确的可访问 Gateway 网关。支持为隔离的配置文件运行多个 Gateway 网关（例如救援机器人），但大多数安装只运行一个 Gateway 网关。
</Note>

```bash
openclaw gateway probe
openclaw gateway probe --json
openclaw gateway probe --port 18789
```

<ParamField path="--port <port>" type="number">
  将此端口用于 local loopback 探针目标和 SSH 隧道远程端口。如果没有 `--url`，此选项只会选择 local loopback 目标，而不选择已配置的 Gateway 网关环境 URL、环境端口或远程目标。
</ParamField>

<AccordionGroup>
  <Accordion title="解释">
    - `Reachable: yes` 表示至少一个目标接受了 WebSocket 连接。
    - `Capability: read-only|write-capable|admin-capable|pairing-pending|connect-only` 报告探针能够确认的身份验证信息，该信息与可访问性分开。
    - `Read probe: ok` 表示读取权限范围的详细 RPC 调用（`health`/`status`/`system-presence`/`config.get`）也成功。
    - `Read probe: limited - missing scope: operator.read` 表示连接成功，但读取权限范围的 RPC 受限。报告为**降级**可访问性，而不是完全失败。
    - `Read probe: failed` 出现在 `Connect: ok` 之后，表示 WebSocket 已连接，但后续读取诊断超时或失败——同样是**降级**，而不是不可访问。
    - 与 `gateway status` 类似，探针会复用现有已缓存的设备身份验证，但不会创建首次设备身份或配对状态。
    - 仅当所有探测目标都不可访问时，退出代码才为非零。

  </Accordion>
  <Accordion title="JSON 输出">
    顶层：

    - `ok`：至少一个目标可访问。
    - `degraded`：至少一个目标接受了连接，但未完成完整的详细 RPC 诊断。
    - `capability`：在可访问目标中发现的最佳能力（`read_only`、`write_capable`、`admin_capable`、`pairing_pending`、`connected_no_operator_scope` 或 `unknown`）。
    - `primaryTargetId`：应视为当前生效目标的最佳目标，优先顺序为：显式 URL、SSH 隧道、已配置的远程目标、local loopback。
    - `warnings[]`：尽力而为的警告记录，包含 `code`、`message`，以及可选的 `targetIds`。
    - `network`：根据当前配置和主机网络派生的 local loopback/tailnet URL 提示。
    - `discovery.timeoutMs` / `discovery.count`：此轮探测实际使用的设备发现预算/结果数量。

    每个目标（`targets[].connect`）：`ok`（可访问性 + 降级分类）、`rpcOk`（完整详细 RPC 成功）、`scopeLimited`（由于缺少操作员权限范围，详细 RPC 失败）。

    每个目标（`targets[].auth`）：可用时在 `hello-ok` 中报告 `role` 和 `scopes`，以及显示的 `capability` 分类。

  </Accordion>
  <Accordion title="常见警告代码">
    - `ssh_tunnel_failed`：SSH 隧道设置失败；命令已回退到直接探测。
    - `multiple_gateways`：可访问到不同的 Gateway 网关身份，或者 OpenClaw 无法证明可访问的目标属于同一个 Gateway 网关。指向同一 Gateway 网关的 SSH 隧道、代理 URL 或已配置的远程 URL 不会触发此警告。
    - `auth_secretref_unresolved`：无法为失败的目标解析已配置的身份验证 SecretRef。
    - `probe_scope_limited`：WebSocket 连接成功，但由于缺少 `operator.read`，读取探测受到限制。
    - `local_tls_runtime_unavailable`：本地 Gateway 网关 TLS 已启用，但 OpenClaw 无法加载本地证书指纹。

  </Accordion>
</AccordionGroup>

#### 通过 SSH 远程连接（与 Mac 应用功能一致）

macOS 应用的“Remote over SSH”模式使用本地端口转发，使仅限环回访问的远程 Gateway 网关可以通过 `ws://127.0.0.1:<port>` 访问。

对应的 CLI 命令：

```bash
openclaw gateway probe --ssh user@gateway-host
```

<ParamField path="--ssh <target>" type="string">
  `user@host` 或 `user@host:port`（端口默认为 `22`）。
</ParamField>
<ParamField path="--ssh-identity <path>" type="string">
  身份文件。
</ParamField>
<ParamField path="--ssh-auto" type="boolean">
  从已解析的设备发现端点（`local.`，以及已配置的广域网域名（如有））中选择第一个发现的 Gateway 网关主机作为 SSH 目标。忽略仅来自 TXT 的提示。
</ParamField>

配置默认值（可选）：`gateway.remote.sshTarget`、`gateway.remote.sshIdentity`。

### `gateway call <method>`

底层 RPC 辅助工具。

```bash
openclaw gateway call status
openclaw gateway call logs.tail --params '{"limit": 200}'
```

<ParamField path="--params <json>" type="string" default="{}">
  参数的 JSON 对象字符串。
</ParamField>
<ParamField path="--url <url>" type="string">
  Gateway 网关 WebSocket URL。
</ParamField>
<ParamField path="--token <token>" type="string">
  Gateway 网关令牌。
</ParamField>
<ParamField path="--password <password>" type="string">
  Gateway 网关密码。
</ParamField>
<ParamField path="--timeout <ms>" type="number" default="10000">
  超时预算。
</ParamField>
<ParamField path="--expect-final" type="boolean">
  主要用于在最终有效载荷之前流式传输中间事件的智能体式 RPC。
</ParamField>
<ParamField path="--json" type="boolean">
  机器可读的 JSON 输出。
</ParamField>

<Note>
`--params` 必须是有效的 JSON，并且每个方法都会验证自己的参数结构（多余或命名错误的字段会被拒绝）。
</Note>

## 管理 Gateway 网关服务

```bash
openclaw gateway install
openclaw gateway start
openclaw gateway stop
openclaw gateway restart
openclaw gateway uninstall
```

### 使用包装器安装

当托管服务必须通过另一个可执行文件启动时，请使用 `--wrapper`，例如密钥管理器适配程序或以指定用户身份运行的辅助工具。包装器会收到常规 Gateway 网关参数，并负责最终使用这些参数执行 `openclaw` 或 Node。

```bash
cat > ~/.local/bin/openclaw-doppler <<'EOF'
#!/usr/bin/env bash
set -euo pipefail
exec doppler run --project my-project --config production -- openclaw "$@"
EOF
chmod +x ~/.local/bin/openclaw-doppler

openclaw gateway install --wrapper ~/.local/bin/openclaw-doppler --force
openclaw gateway restart
```

也可以通过环境设置包装器。`gateway install` 会验证该路径是否为可执行文件，将包装器写入服务 `ProgramArguments`，并在服务环境中持久保存 `OPENCLAW_WRAPPER`，供以后强制重新安装、更新和 Doctor 修复使用。

```bash
OPENCLAW_WRAPPER="$HOME/.local/bin/openclaw-doppler" openclaw gateway install --force
openclaw doctor
```

要移除持久保存的包装器，请在重新安装时清除 `OPENCLAW_WRAPPER`：

```bash
OPENCLAW_WRAPPER= openclaw gateway install --force
openclaw gateway restart
```

<AccordionGroup>
  <Accordion title="命令选项">
    - `gateway status`：`--url`、`--token`、`--password`、`--timeout`、`--no-probe`、`--require-rpc`、`--deep`、`--json`
    - `gateway install`：`--port`、`--runtime <node>`（默认值：`node`）、`--token`、`--wrapper <path>`、`--force`、`--json`
    - `gateway restart`：`--safe`、`--skip-deferral`、`--force`、`--wait <duration>`、`--json`
    - `gateway uninstall|start`：`--json`
    - `gateway stop`：`--disable`、`--force`、`--json`

  </Accordion>
  <Accordion title="生命周期行为">
    - `gateway start` 是幂等的：当托管服务已在运行时，它会报告正在运行的进程并保持不变。已加载但已停止的服务仍会像以前一样启动。
    - 使用 `gateway restart` 重启托管服务。不要串联 `gateway stop` 和 `gateway start` 来代替重启。
    - 在非交互式 shell 中，`gateway stop` 需要 `--force`。交互式终端继续保留现有的无提示行为。对于自动化和测试，优先使用 `gateway run --dev`，或者使用带有空闲端口的隔离 `--profile`。
    - 在 macOS 上，`gateway stop` 默认使用 `launchctl bootout`，它会从当前启动会话中移除 LaunchAgent，而不会持久禁用它——KeepAlive 自动恢复仍会在未来崩溃时保持启用，并且 `gateway start` 无需手动执行 `launchctl enable` 即可正常重新启用。传入 `--disable` 可持久禁止 KeepAlive 和 RunAtLoad，使 Gateway 网关在下一次显式执行 `gateway start` 之前不会重新生成；当手动停止需要在重启后继续生效时，请使用此选项。
    - Gateway 网关生命周期变更会将尽力而为的键值审计记录追加到 `<state-dir>/logs/gateway-restart.log`，其中包括 CLI 启动、停止和重启操作、安全重启请求、监管器重启以及分离式移交。
    - 生命周期命令接受 `--json`，以便在脚本中使用。

  </Accordion>
  <Accordion title="托管 Gateway 网关堆大小">
    - `gateway install` 会为托管 Gateway 网关服务写入仅限堆的 `NODE_OPTIONS` 值。当 Node 报告容器或服务限制时，其目标为受限内存的 50%；否则为物理内存的 50%。
    - 标称目标范围为 2048–8192 MiB，并额外设有 75% 的原生内存余量上限。在小型主机上，该余量上限可能使实际应用的限制低于标称的 2048 MiB 下限。
    - 已安装服务中存储的有效显式 `--max-old-space-size` 会在强制重新安装和 Doctor 修复期间保留。其他 `NODE_OPTIONS` 标志不会被带入托管服务。
    - shell 环境中的 `NODE_OPTIONS` 不会覆盖此策略。使用 `gateway status` 或 `doctor` 检查已安装的值；运行 `openclaw gateway install --force` 可重新生成没有托管堆设置的旧版服务元数据。
    - 此策略仅适用于托管 Gateway 网关服务。前台 `gateway run`、节点服务和手写的监管器单元保留各自的运行时配置。

  </Accordion>
  <Accordion title="安装时的身份验证和 SecretRef">
    - 当令牌身份验证需要令牌且 `gateway.auth.token` 由 SecretRef 管理时，`gateway install` 会验证 SecretRef 是否可解析，但不会将解析后的令牌持久保存到服务环境元数据中。
    - 如果令牌身份验证需要令牌，而配置的令牌 SecretRef 无法解析，安装将以关闭方式失败，而不会持久保存回退明文。
    - 对于 `gateway run` 上的密码身份验证，优先使用 `OPENCLAW_GATEWAY_PASSWORD`、`--password-file` 或由 SecretRef 支持的 `gateway.auth.password`，而不是内联的 `--password`。
    - 在推断身份验证模式下，仅存在于 shell 中的 `OPENCLAW_GATEWAY_PASSWORD` 不会放宽安装令牌要求；安装托管服务时，请使用持久配置（`gateway.auth.password` 或配置 `env`）。
    - 如果同时配置了 `gateway.auth.token` 和 `gateway.auth.password`，但未设置 `gateway.auth.mode`，则安装会被阻止，直到显式设置模式为止。

  </Accordion>
</AccordionGroup>

## 发现 Gateway 网关（Bonjour）

`gateway discover` 扫描 Gateway 网关信标（`_openclaw-gw._tcp`）。

- 组播 DNS-SD：`local.`
- 单播 DNS-SD（广域 Bonjour）：选择一个域名（例如：`openclaw.internal.`），并设置分离式 DNS + DNS 服务器；请参阅 [Bonjour](/zh-CN/gateway/bonjour)。

只有启用了 Bonjour 设备发现（默认启用）的 Gateway 网关才会广播信标。

每个信标上的 TXT 提示：`role`（Gateway 网关角色提示）、`transport`（传输提示，例如 `gateway`）、`gatewayPort`（WebSocket 端口，通常为 `18789`）、`tailnetDns`（MagicDNS 主机名，如可用）、`gatewayTls` / `gatewayTlsSha256`（TLS 已启用 + 证书指纹）。`sshPort` 和 `cliPath` 仅在完整设备发现模式下发布（`discovery.mdns.mode: "full"`；默认为 `"minimal"`，会省略它们——此时客户端会默认使用端口 `22` 作为 SSH 目标）。

### `gateway discover`

```bash
openclaw gateway discover
```

<ParamField path="--timeout <ms>" type="number" default="2000">
  每条命令的超时时间（浏览/解析）。
</ParamField>
<ParamField path="--json" type="boolean">
  机器可读的输出（同时禁用样式和旋转指示器）。
</ParamField>

示例：

```bash
openclaw gateway discover --timeout 4000
openclaw gateway discover --json | jq '.beacons[].wsUrl'
```

<Note>
- 扫描 `local.`，并在启用广域网域名时同时扫描已配置的广域网域名。
- JSON 输出中的 `wsUrl` 派生自已解析的服务端点，而不是 `lanHost` 或 `tailnetDns` 等仅来自 TXT 的提示。
- `discovery.mdns.mode` 控制 `local.` mDNS 和广域 DNS-SD 上 `sshPort`/`cliPath` 的发布（见上文）。

</Note>

## 相关内容

- [CLI 参考](/zh-CN/cli)
- [Gateway 网关运行手册](/zh-CN/gateway)
