---
read_when:
    - 运行或调试 Gateway 网关进程
summary: Gateway 网关服务、生命周期和运维运行手册
title: Gateway 网关运行手册
x-i18n:
    generated_at: "2026-07-26T06:44:19Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: d8b50b6041905c321887ea0f579f8d4c3b74552b2b72c37ec655e43a53dfc130
    source_path: gateway/index.md
    workflow: 16
---

使用此页面完成 Gateway 网关服务的首日启动和后续日常运维。

<CardGroup cols={2}>
  <Card title="深入故障排查" icon="siren" href="/zh-CN/gateway/troubleshooting">
    按症状开展诊断，提供确切的命令步骤和日志特征。
  </Card>
  <Card title="配置" icon="sliders" href="/zh-CN/gateway/configuration">
    面向任务的设置指南 + 完整配置参考。
  </Card>
  <Card title="密钥管理" icon="key-round" href="/zh-CN/gateway/secrets">
    SecretRef 契约、运行时快照行为，以及迁移/重新加载操作。
  </Card>
  <Card title="密钥计划契约" icon="shield-check" href="/zh-CN/gateway/secrets-plan-contract">
    确切的 `secrets apply` 目标/路径规则和仅引用的身份验证配置文件行为。
  </Card>
</CardGroup>

## 5 分钟本地启动

<Steps>
  <Step title="启动 Gateway 网关">

```bash
openclaw gateway --port 18789
# 调试/跟踪信息同步输出到 stdio
openclaw gateway --port 18789 --verbose
# 强制终止所选端口上的监听进程，然后启动
openclaw gateway --force
```

  </Step>

  <Step title="验证服务健康状况">

```bash
openclaw gateway status
openclaw status
openclaw logs --follow
```

健康基线：`Runtime: running`、`Connectivity probe: ok`，以及符合预期的 `Capability` 行。使用 `openclaw gateway status --require-rpc` 证明读取范围 RPC 正常，而不只是证明可达。

  </Step>

  <Step title="验证渠道就绪状态">

```bash
openclaw channels status --probe
```

当 Gateway 网关可达时，此命令会实时运行每个账户的渠道探测和可选审计。如果 Gateway 网关不可达，CLI 会回退到仅基于配置的渠道摘要。

  </Step>
</Steps>

<Note>
Gateway 网关配置重新加载会监视活动配置文件路径（根据配置文件/状态默认值解析，或在设置 `OPENCLAW_CONFIG_PATH` 时使用该值）。默认模式为 `gateway.reload.mode="hybrid"`。首次成功加载后，运行中的进程会使用活动的内存配置快照提供服务；重新加载成功时会以原子方式替换该快照。
</Note>

## 运行时模型

- 一个始终运行的进程，负责路由、控制平面和渠道连接。
- 一个多路复用端口，用于：
  - WebSocket 控制/RPC
  - HTTP API（`/v1/models`、`/v1/embeddings`、`/v1/chat/completions`、`/v1/responses`、`/tools/invoke`）
  - 插件 HTTP 路由，例如可选的 `/api/v1/admin/rpc`
  - Control UI 和 Hooks
- 默认绑定模式：`loopback`。在检测到的容器环境中，有效默认值为 `auto`（解析为 `0.0.0.0` 以支持端口转发）；但当 Tailscale serve/funnel 处于活动状态时例外，此时始终强制使用 `loopback`。
- 默认要求身份验证。共享密钥设置使用 `gateway.auth.token` / `gateway.auth.password`（或 `OPENCLAW_GATEWAY_TOKEN` / `OPENCLAW_GATEWAY_PASSWORD`），非环回反向代理设置可以使用 `gateway.auth.mode: "trusted-proxy"`。

## OpenAI 兼容端点

OpenClaw 最具影响力的兼容性接口：

- `GET /v1/models`
- `GET /v1/models/{id}`
- `POST /v1/embeddings`
- `POST /v1/chat/completions`
- `POST /v1/responses`

此端点集的重要性：

- 大多数 Open WebUI、LobeChat 和 LibreChat 集成会首先探测 `/v1/models`。
- 许多 RAG 和记忆流水线需要 `/v1/embeddings`。
- 原生面向智能体的客户端越来越倾向于使用 `/v1/responses`。

`/v1/models` 以智能体为中心：它会为每个已配置的智能体返回 `openclaw`、`openclaw/default` 和 `openclaw/<agentId>`。`openclaw/default` 是稳定别名，始终映射到已配置的默认智能体。如需覆盖后端提供商/模型，请发送 `x-openclaw-model`；否则继续由所选智能体的常规模型和嵌入设置进行控制。

所有这些端点都在主 Gateway 网关端口上运行，并与 Gateway 网关 HTTP API 的其余部分共用同一可信操作员身份验证边界。

管理 HTTP RPC（`POST /api/v1/admin/rpc`）是一个独立且默认关闭的插件路由，供无法使用 WebSocket RPC 的主机工具使用。请参阅[管理 HTTP RPC](/zh-CN/plugins/admin-http-rpc)。

### 端口和绑定优先级

| 设置         | 解析顺序                                                             |
| ------------ | -------------------------------------------------------------------- |
| Gateway 网关端口 | `--port` → `OPENCLAW_GATEWAY_PORT` → `gateway.port` → `18789`        |
| 绑定模式     | CLI/覆盖值 → `gateway.bind` → `loopback`（容器中为 `auto`） |

已安装的 Gateway 网关服务会在监管程序元数据中记录解析后的 `--port`。更改 `gateway.port` 后，运行 `openclaw doctor --fix` 或 `openclaw gateway install --force`，以便 launchd/systemd/schtasks 在新端口上启动进程。

Gateway 网关启动时，会使用相同的有效端口和绑定，为非环回绑定预填充本地 Control UI 来源。例如，`--bind lan --port 3000` 会在运行时验证开始前预填充 `http://localhost:3000` 和 `http://127.0.0.1:3000`。请将所有远程浏览器来源（例如 HTTPS 代理 URL）显式添加到 `gateway.controlUi.allowedOrigins`。

### 热重新加载模式

| `gateway.reload.mode` | 行为                                       |
| --------------------- | ------------------------------------------ |
| `off`                 | 不重新加载配置                             |
| `hot`                 | 仅应用可安全热更新的更改                   |
| `restart`             | 遇到需要重新加载的更改时重启               |
| `hybrid`（默认）      | 安全时热应用，需要时重启                   |

## 操作员命令集

```bash
openclaw gateway status
openclaw gateway status --deep   # 添加系统级服务扫描
openclaw gateway status --json
openclaw gateway install
openclaw gateway restart
openclaw gateway stop
openclaw secrets reload
openclaw logs --follow
openclaw doctor
```

`gateway status --deep` 用于额外的服务发现（LaunchDaemons/systemd 系统单元/schtasks），而不是更深入的 RPC 健康探测。

## 多个 Gateway 网关（同一主机）

大多数安装应在每台机器上运行一个 Gateway 网关。单个 Gateway 网关可以托管多个智能体和渠道。仅当有意实现隔离或需要救援机器人时，才需要多个 Gateway 网关。

实用检查：

```bash
openclaw gateway status --deep
openclaw gateway probe
```

预期结果：

- `gateway status --deep` 可以报告 `Other gateway-like services detected (best effort)`，并在仍存在过期的 launchd/systemd/schtasks 安装时输出清理提示。
- 当不同的 Gateway 网关作出响应，或 OpenClaw 无法证明可达目标属于同一个 Gateway 网关时，`gateway probe` 可以针对 `multiple reachable gateway identities` 发出警告。即使传输端口不同，指向同一个 Gateway 网关的 SSH 隧道、代理 URL 或已配置远程 URL，仍然是一个具有多种传输方式的 Gateway 网关。
- 如果这是有意的，请为每个 Gateway 网关隔离端口、配置/状态和工作区根目录。

每个实例的检查清单：

- 唯一的 `gateway.port`
- 唯一的 `OPENCLAW_CONFIG_PATH`
- 唯一的 `OPENCLAW_STATE_DIR`
- 唯一的 `agents.defaults.workspace`

示例：

```bash
OPENCLAW_CONFIG_PATH=~/.openclaw/a.json OPENCLAW_STATE_DIR=~/.openclaw-a openclaw gateway --port 19001
OPENCLAW_CONFIG_PATH=~/.openclaw/b.json OPENCLAW_STATE_DIR=~/.openclaw-b openclaw gateway --port 19002
```

详细设置：[/gateway/multiple-gateways](/zh-CN/gateway/multiple-gateways)。

## 远程访问

首选：Tailscale/VPN。
备用：SSH 隧道。

```bash
ssh -N -L 18789:127.0.0.1:18789 user@gateway-host
```

然后将客户端连接到本地的 `ws://127.0.0.1:18789`。

<Warning>
SSH 隧道不会绕过 Gateway 网关身份验证。对于共享密钥身份验证，即使通过隧道，客户端仍
必须发送 `token`/`password`。对于携带身份信息的模式，
请求仍须满足对应的身份验证路径。
</Warning>

请参阅：[远程 Gateway 网关](/zh-CN/gateway/remote)、[身份验证](/zh-CN/gateway/authentication)、[Tailscale](/zh-CN/gateway/tailscale)。

## 监管和服务生命周期

使用受监管的运行方式，以获得生产环境级别的可靠性。

<Tabs>
  <Tab title="macOS (launchd)">

```bash
openclaw gateway install
openclaw gateway status
openclaw gateway restart
openclaw gateway stop
```

使用 `openclaw gateway restart` 进行重启。不要串联 `openclaw gateway stop` 和 `openclaw gateway start` 来代替重启。

在 macOS 上，`gateway stop` 默认使用 `launchctl bootout`。这会从当前启动会话中移除 LaunchAgent，但不会永久禁用它，因此意外崩溃后 KeepAlive 自动恢复仍然有效，且 `gateway start` 可以正常重新启用。若要在重启后仍持续禁止自动重新生成，请传递 `--disable`：`openclaw gateway stop --disable`。

LaunchAgent 标签为 `ai.openclaw.gateway`（默认）或 `ai.openclaw.<profile>`（命名配置文件）。`openclaw doctor` 会审计并修复服务配置漂移。

  </Tab>

  <Tab title="Linux (systemd 用户服务)">

```bash
openclaw gateway install
systemctl --user enable --now openclaw-gateway[-<profile>].service
openclaw gateway status
```

若要在注销后继续运行，请启用 lingering：

```bash
sudo loginctl enable-linger $(whoami)
```

在没有桌面会话的无头服务器上，重试 `systemctl --user` 命令前，还要确保已设置 `XDG_RUNTIME_DIR`（`export XDG_RUNTIME_DIR=/run/user/$(id -u)`）。

需要自定义安装路径时，可使用以下手动用户单元示例：

```ini
[Unit]
Description=OpenClaw Gateway
After=network-online.target
Wants=network-online.target
StartLimitBurst=5
StartLimitIntervalSec=60

[Service]
ExecStart=/usr/local/bin/openclaw gateway --port 18789
Restart=always
RestartSec=5
RestartPreventExitStatus=78
TimeoutStopSec=30
TimeoutStartSec=30
SuccessExitStatus=0 143
OOMPolicy=continue
KillMode=control-group

[Install]
WantedBy=default.target
```

  </Tab>

  <Tab title="Windows（原生）">

```powershell
openclaw gateway install
openclaw gateway status --json
openclaw gateway restart
openclaw gateway stop
```

原生 Windows 托管启动使用名为 `OpenClaw Gateway`
（命名配置文件使用 `OpenClaw Gateway (<profile>)`）的计划任务。如果计划任务
创建遭到拒绝，OpenClaw 会回退到每用户的“启动”文件夹启动器，
该启动器指向状态目录中的 `gateway.cmd`。

  </Tab>

  <Tab title="Linux（系统服务）">

对于多用户/始终在线的主机，请使用系统单元。

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now openclaw-gateway[-<profile>].service
```

使用与用户单元相同的服务主体，但将其安装在
`/etc/systemd/system/openclaw-gateway[-<profile>].service` 下；如果 `openclaw` 二进制文件位于其他位置，
请调整 `ExecStart=`。

不要同时让 `openclaw doctor --fix` 为同一配置文件/端口安装用户级 Gateway 网关服务。当 Doctor 发现系统级 OpenClaw Gateway 网关服务时，会拒绝该自动安装；当系统单元负责生命周期时，请使用 `OPENCLAW_SERVICE_REPAIR_POLICY=external`。

  </Tab>
</Tabs>

无效配置错误会以代码 `78` 退出。Linux systemd 单元使用 `RestartPreventExitStatus=78`，在配置修复前停止重新启动。launchd 和 Windows 任务计划程序没有等效的按退出代码停止规则，因此 Gateway 网关还会持久保存短时间内的异常启动历史，并在启动反复失败后禁止渠道/提供商账户自动启动。在此安全模式下，控制平面仍会启动，以供检查和修复；配置热重新加载和 `secrets.reload` 会拒绝自动重启渠道，而操作员显式发出的 `channels.start` 请求可以覆盖此限制。

## 开发配置文件快速路径

```bash
openclaw --dev setup
openclaw --dev gateway --allow-unconfigured
openclaw --dev status
```

默认设置包括隔离的状态/配置，以及 Gateway 网关基础端口 `19001`。

## 协议快速参考（操作员视角）

- 第一个客户端帧必须是 `connect`。
- Gateway 网关返回一个 `hello-ok` 帧，其中包含 `snapshot`（`presence`、`health`、`stateVersion`、`uptimeMs`）以及 `policy` 限制（`maxPayload`、`maxBufferedBytes`、`tickIntervalMs`）。
- `hello-ok.features.methods` / `events` 是一份保守的设备发现列表，并非
  每个可调用辅助路由的自动生成转储。
- 请求：`req(method, params)` → `res(ok/payload|error)`。
- 常见事件包括 `connect.challenge`、`agent`、`chat`、
  `session.message`、`session.operation`、`session.tool`、选择启用的
  `session.approval`、`sessions.changed`、`presence`、`tick`、`health`、
  `heartbeat`、配对/审批生命周期事件以及 `shutdown`。

智能体运行分为两个阶段：

1. 立即返回已接受确认（`status:"accepted"`）
2. 最终完成响应（`status:"ok"|"error"`），期间会流式传输 `agent` 事件。

完整协议文档请参阅：[Gateway 协议](/zh-CN/gateway/protocol)。

## 操作检查

### 存活性

- 打开 WS 并发送 `connect`。
- 预期收到包含快照的 `hello-ok` 响应。

### 就绪性

```bash
openclaw gateway status
openclaw channels status --probe
openclaw health
```

### 间隙恢复

事件不会重放。出现序列间隙时，请先刷新状态（`health`、`system-presence`），然后再继续。

## 常见故障特征

| 特征                                                      | 可能的问题                                                                  |
| -------------------------------------------------------------- | ----------------------------------------------------------------------------- |
| `refusing to bind gateway ... without auth`                    | 在没有有效 Gateway 网关身份验证路径的情况下绑定到非 loopback 地址                           |
| `another gateway instance is already listening` / `EADDRINUSE` | 端口冲突                                                                 |
| `Gateway start blocked: set gateway.mode=local`                | 配置设为远程模式，或受损配置中缺少 `gateway.mode` |
| 连接期间出现 `unauthorized`                                  | 客户端与 Gateway 网关之间的身份验证不匹配                                      |

有关完整的诊断步骤，请参阅 [Gateway 故障排查](/zh-CN/gateway/troubleshooting)。

## 安全保证

- Gateway 网关不可用时，Gateway 网关协议客户端会快速失败（不会隐式回退到直接渠道）。
- 无效的第一个帧或非连接帧会被拒绝，并关闭连接。
- 正常关闭会在套接字关闭前发出 `shutdown` 事件。

## 相关内容

- [配置](/zh-CN/gateway/configuration)
- [Gateway 故障排查](/zh-CN/gateway/troubleshooting)
- [后台进程](/zh-CN/gateway/background-process)
- [健康状态](/zh-CN/gateway/health)
- [Doctor](/zh-CN/gateway/doctor)
- [身份验证](/zh-CN/gateway/authentication)
- [远程访问](/zh-CN/gateway/remote)
- [密钥管理](/zh-CN/gateway/secrets)
