---
read_when:
    - 将 Codex、Claude Code 或其他 MCP 客户端连接到由 OpenClaw 支持的渠道
    - 正在运行 `openclaw mcp serve`
    - 管理 OpenClaw 保存的 MCP 服务器定义
sidebarTitle: MCP
summary: 通过 MCP 公开 OpenClaw 渠道对话并管理已保存的 MCP 服务器定义
title: MCP
x-i18n:
    generated_at: "2026-07-26T06:09:49Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: ee6146bbc0181d10997336094d1bd693d0afb0985f1febef8e8c6b0d6e656cf9
    source_path: cli/mcp.md
    workflow: 16
---

`openclaw mcp` 有两个用途：

- 使用 `openclaw mcp serve` 将 OpenClaw 作为 MCP 服务器运行
- 使用 `list`、`show`、`status`、`doctor`、`probe`、`add`、`set`、`configure`、`tools`、`login`、`logout`、`reload` 和 `unset` 管理由 OpenClaw 管理的出站 MCP 服务器定义

`serve` 是作为 MCP 服务器运行的 OpenClaw。其他子命令则让 OpenClaw 充当 MCP 客户端侧注册表，供其自身运行时稍后使用其中的服务器。

<Note>
  `list`、`show`、`set` 和 `unset` 只会读写 OpenClaw 配置中由 OpenClaw 管理的 `mcp.servers` 条目。它们不包括 `config/mcporter.json` 中的 mcporter 服务器；请使用 `mcporter list` 管理该注册表。
</Note>

当 OpenClaw 应自行托管编码工具框架会话，并通过 ACP 路由该运行时时，请使用 [`openclaw acp`](/zh-CN/cli/acp)。

## 选择正确的 MCP 路径

| 目标                                                                | 使用                                                                  | 原因                                                                                                             |
| ------------------------------------------------------------------- | -------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------- |
| 让外部 MCP 客户端读取或发送 OpenClaw 渠道会话 | `openclaw mcp serve`                                                 | OpenClaw 是 MCP 服务器，通过 stdio 公开由 Gateway 网关支持的会话。                                 |
| 保存第三方 MCP 服务器，供 OpenClaw 管理的智能体运行使用        | `openclaw mcp add`、`set`、`configure`、`tools`、`login`             | OpenClaw 是 MCP 客户端侧注册表，稍后会将这些服务器映射到符合条件的运行时中。               |
| 在不运行智能体轮次的情况下检查已保存的服务器                  | `openclaw mcp status`、`doctor`、`probe`                             | `status` 和 `doctor` 检查配置；`probe` 会建立实时 MCP 连接并列出能力。               |
| 从浏览器编辑 MCP 配置                                      | Control UI `/settings/mcp`（别名 `/mcp`）                            | 该页面显示清单、启用状态、OAuth/过滤器摘要、命令提示和限定范围的 `mcp` 编辑器。         |
| 为 Codex app-server 提供限定范围的原生 MCP 服务器                    | `mcp.servers.<name>.codex`                                           | `codex` 块只影响 Codex app-server 线程映射，并会在原生配置交接前被移除。 |
| 运行由 ACP 托管的工具框架会话                                     | [`openclaw acp`](/zh-CN/cli/acp) 和 [ACP 智能体](/zh-CN/tools/acp-agents-setup) | ACP 桥接模式不接受按会话注入 MCP 服务器；请改为配置 Gateway 网关/插件桥接。     |

<Tip>
如果不确定需要哪条路径，请从 `openclaw mcp status --verbose` 开始。它会显示 OpenClaw 已保存的内容，而不会启动任何 MCP 服务器。
</Tip>

## 将 OpenClaw 作为 MCP 服务器

这是 `openclaw mcp serve` 路径。

### 何时使用 serve

在以下情况下使用 `openclaw mcp serve`：

- Codex、Claude Code 或其他 MCP 客户端应直接与由 OpenClaw 支持的渠道会话通信
- 你已经有一个具备已路由会话的本地或远程 OpenClaw Gateway 网关
- 你希望使用一个适用于 OpenClaw 各渠道后端的 MCP 服务器，而不是为每个渠道分别运行桥接

当 OpenClaw 应自行托管编码运行时，并将智能体会话保留在 OpenClaw 内部时，请改用 [`openclaw acp`](/zh-CN/cli/acp)。

### 工作原理

`openclaw mcp serve` 会启动一个 stdio MCP 服务器。该进程由 MCP 客户端所有。当客户端保持 stdio 会话打开时，桥接器会通过 WebSocket 连接到本地或远程 OpenClaw Gateway 网关，并通过 MCP 公开已路由的渠道会话。

<Steps>
  <Step title="客户端生成桥接器">
    MCP 客户端会生成 `openclaw mcp serve`。
  </Step>
  <Step title="桥接器连接到 Gateway 网关">
    桥接器通过 WebSocket 连接到 OpenClaw Gateway 网关。
  </Step>
  <Step title="会话成为 MCP 会话">
    已路由的会话会成为 MCP 会话以及转录记录/历史记录工具。
  </Step>
  <Step title="实时事件进入队列">
    桥接器连接期间，实时事件会在内存中排队。
  </Step>
  <Step title="可选的 Claude 推送">
    如果启用了 Claude 渠道模式，同一会话还可以接收 Claude 专用的推送通知。
  </Step>
</Steps>

<AccordionGroup>
  <Accordion title="重要行为">
    - 实时队列状态会在桥接器连接时开始
    - 使用 `messages_read` 读取较早的转录历史记录
    - Claude 推送通知仅在 MCP 会话存续期间存在
    - 客户端断开连接时，桥接器会退出，实时队列也会消失
    - `openclaw agent` 和 `openclaw infer model run` 等一次性智能体入口点会在回复完成时停用其打开的所有内置 MCP 运行时，因此重复执行脚本不会不断累积 stdio MCP 子进程
    - OpenClaw 启动的 stdio MCP 服务器（内置或用户配置）会在关闭时以进程树为单位终止，因此服务器启动的子进程不会在父 stdio 客户端退出后继续存活
    - 删除或重置会话时，会通过共享运行时清理路径释放该会话的 MCP 客户端，因此不会留下与已移除会话关联的 stdio 连接

  </Accordion>
</AccordionGroup>

### 选择客户端模式

<Tabs>
  <Tab title="通用 MCP 客户端">
    仅使用标准 MCP 工具。使用 `conversations_list`、`messages_read`、`events_poll`、`events_wait`、`messages_send` 和审批工具。
  </Tab>
  <Tab title="Claude Code">
    使用标准 MCP 工具以及 Claude 专用的渠道适配器。启用 `--claude-channel-mode on`，或保留默认的 `auto`。
  </Tab>
</Tabs>

<Note>
目前，`auto` 的行为与 `on` 相同。尚未实现客户端能力检测。
</Note>

### serve 公开的内容

桥接器使用现有的 Gateway 网关会话路由元数据，公开由渠道支持的会话。当 OpenClaw 已有包含已知路由的会话状态时，会显示相应会话，例如：

- `channel`
- 接收方或目标元数据
- 可选的 `accountId`
- 可选的 `threadId`

这为 MCP 客户端提供了一个统一位置，用于：

- 列出最近已路由的会话
- 读取最近的转录历史记录
- 等待新的入站事件
- 通过同一路由发回回复
- 查看桥接器连接期间收到的审批请求

### 用法

<Tabs>
  <Tab title="本地 Gateway 网关">
    ```bash
    openclaw mcp serve
    ```
  </Tab>
  <Tab title="远程 Gateway 网关（令牌）">
    ```bash
    openclaw mcp serve --url wss://gateway-host:18789 --token-file ~/.openclaw/gateway.token
    ```
  </Tab>
  <Tab title="远程 Gateway 网关（密码）">
    ```bash
    openclaw mcp serve --url wss://gateway-host:18789 --password-file ~/.openclaw/gateway.password
    ```
  </Tab>
  <Tab title="详细输出/关闭 Claude">
    ```bash
    openclaw mcp serve --verbose
    openclaw mcp serve --claude-channel-mode off
    ```
  </Tab>
</Tabs>

### 桥接工具

<AccordionGroup>
  <Accordion title="conversations_list">
    列出 Gateway 网关会话状态中已有路由元数据的近期会话支持型会话。

    过滤器：`limit`（最大 500）、`search`、`channel`、`includeDerivedTitles`、`includeLastMessage`。

  </Accordion>
  <Accordion title="conversation_get">
    通过直接查询 Gateway 网关会话，使用 `session_key` 返回一个会话。
  </Accordion>
  <Accordion title="messages_read">
    读取一个会话支持型会话的近期转录消息。`limit` 默认为 20，最大为 200。
  </Accordion>
  <Accordion title="attachments_fetch">
    从一条转录消息中提取非文本消息内容块。这是转录内容的元数据视图，而不是独立的持久化附件二进制对象存储。
  </Accordion>
  <Accordion title="events_poll">
    读取从数字游标开始的已排队实时事件。`limit` 最大为 200。
  </Accordion>
  <Accordion title="events_wait">
    长轮询，直到下一个匹配的队列事件到达或超时（默认 30s，最大 300s）。

    当通用 MCP 客户端需要接近实时的传递，但不使用 Claude 专用推送协议时，请使用此工具。

  </Accordion>
  <Accordion title="messages_send">
    通过会话中已记录的同一路由发回文本。

    当前行为：

    - 需要已有会话路由
    - 使用会话的渠道、接收方、账户 ID 和线程 ID
    - 仅发送文本

  </Accordion>
  <Accordion title="permissions_list_open">
    列出桥接器自连接到 Gateway 网关以来观察到的待处理 Exec/插件审批请求。
  </Accordion>
  <Accordion title="permissions_respond">
    使用以下值之一处理待处理的 Exec/插件审批请求：

    - `allow-once`
    - `allow-always`
    - `deny`

  </Accordion>
</AccordionGroup>

### 事件模型

桥接器在连接期间会维护一个内存事件队列。

当前事件类型：

- `message`
- `exec_approval_requested`
- `exec_approval_resolved`
- `plugin_approval_requested`
- `plugin_approval_resolved`
- `claude_permission_request`

<Warning>
- 该队列仅包含实时数据；它会在 MCP 桥接器启动时开始
- `events_poll` 和 `events_wait` 本身不会重放较早的 Gateway 网关历史记录
- 应使用 `messages_read` 读取持久化积压记录

</Warning>

### Claude 渠道通知

桥接器还可以公开 Claude 专用的渠道通知。这相当于 OpenClaw 中的 Claude Code 渠道适配器：标准 MCP 工具仍然可用，但实时入站消息也可以作为 Claude 专用 MCP 通知到达。

<Tabs>
  <Tab title="off">
    `--claude-channel-mode off`：仅使用标准 MCP 工具。
  </Tab>
  <Tab title="on">
    `--claude-channel-mode on`：启用 Claude 渠道通知。
  </Tab>
  <Tab title="auto（默认）">
    `--claude-channel-mode auto`：当前默认值；桥接器行为与 `on` 相同。
  </Tab>
</Tabs>

启用 Claude 渠道模式后，服务器会声明 Claude 实验性能力，并可以发出：

- `notifications/claude/channel`
- `notifications/claude/channel/permission`

当前桥接器行为：

- 入站 `user` 转录消息会转发为 `notifications/claude/channel`
- 通过 MCP 收到的 Claude 权限请求会在内存中跟踪
- 如果关联会话中的命令所有者随后发送 `yes <id>` 或 `no <id>`（`<id>` 是 5 个字母的请求 ID，不包括 `l`），桥接器会将其转换为 `notifications/claude/channel/permission`
- 这些通知仅适用于实时会话；如果 MCP 客户端断开连接，就不会再有推送目标

这是特意针对特定客户端设计的。通用 MCP 客户端应使用标准轮询工具。

### MCP 客户端配置

stdio 客户端配置示例：

```json
{
  "mcpServers": {
    "openclaw": {
      "command": "openclaw",
      "args": [
        "mcp",
        "serve",
        "--url",
        "wss://gateway-host:18789",
        "--token-file",
        "/path/to/gateway.token"
      ]
    }
  }
}
```

对于大多数通用 MCP 客户端，请从标准工具界面开始，并忽略 Claude 模式。仅对确实理解 Claude 特有通知方法的客户端启用 Claude 模式。

### 选项

`openclaw mcp serve` 支持：

<ParamField path="--url" type="string">
  Gateway 网关 WebSocket URL。配置后默认为 `gateway.remote.url`。
</ParamField>
<ParamField path="--token" type="string">
  Gateway 网关令牌。
</ParamField>
<ParamField path="--token-file" type="string">
  从文件读取令牌。
</ParamField>
<ParamField path="--password" type="string">
  Gateway 网关密码。
</ParamField>
<ParamField path="--password-file" type="string">
  从文件读取密码。
</ParamField>
<ParamField path="--claude-channel-mode" type='"auto" | "on" | "off"'>
  Claude 通知模式。默认为 `auto`。
</ParamField>
<ParamField path="-v, --verbose" type="boolean">
  在 stderr 上输出详细日志。
</ParamField>

<Tip>
如果可以，请优先使用 `--token-file` 或 `--password-file`，而不是内联密钥。
</Tip>

### 安全和信任边界

该桥接器不会自行创建路由。它只会公开 Gateway 网关已经知道如何路由的对话。

这意味着：

- 发送者允许列表、配对和渠道级信任仍归底层 OpenClaw 频道配置所有
- `messages_send` 只能通过已有的已存储路由回复
- 审批状态仅在当前桥接会话期间实时保存在内存中
- 桥接身份验证应使用与任何其他远程 Gateway 网关客户端相同的、你信任的 Gateway 网关令牌或密码控制措施

如果 `conversations_list` 中缺少某个对话，通常原因并非 MCP 配置，而是底层 Gateway 网关会话中的路由元数据缺失或不完整。

### 测试

OpenClaw 为此桥接器提供了确定性的 Docker 冒烟测试：

```bash
pnpm test:docker:mcp-channels
```

该冒烟测试运行单个容器：它植入对话状态，启动 Gateway 网关，然后将 `openclaw mcp serve` 作为 stdio 子进程生成，并以 MCP 客户端的方式驱动它。它会通过真实的 stdio MCP 桥接验证对话发现、对话记录读取、附件元数据读取、实时事件队列行为，以及 Claude 风格的频道和权限通知。出站发送路由（`messages_send` 复用已存储的对话路由）由 `src/mcp/channel-server.test.ts` 中的单元测试单独覆盖。

这是在测试运行中无需接入真实 Telegram、Discord 或 iMessage 账户即可证明桥接器正常工作的最快方式。

有关更广泛的测试背景，请参阅[测试](/zh-CN/help/testing)。

### 故障排查

<AccordionGroup>
  <Accordion title="未返回任何对话">
    通常表示 Gateway 网关会话尚不可路由。确认底层会话已存储渠道/提供商、接收方以及可选的账户/线程路由元数据。
  </Accordion>
  <Accordion title="events_poll 或 events_wait 遗漏较早的消息">
    这是预期行为。实时队列会在桥接器连接时启动。使用 `messages_read` 读取更早的对话记录历史。
  </Accordion>
  <Accordion title="Claude 通知未显示">
    请检查以下所有项目：

    - 客户端保持 stdio MCP 会话处于打开状态
    - `--claude-channel-mode` 为 `on` 或 `auto`
    - 客户端确实理解 Claude 特有的通知方法
    - 入站消息发生在桥接器连接之后

  </Accordion>
  <Accordion title="缺少审批">
    `permissions_list_open` 仅显示桥接器连接期间观察到的审批请求。它不是持久化的审批历史 API。
  </Accordion>
</AccordionGroup>

## 将 OpenClaw 用作 MCP 客户端注册表

这是 `openclaw mcp list`、`show`、`status`、`doctor`、`probe`、`add`、`set`、
`configure`、`tools`、`login`、`logout`、`reload` 和 `unset` 路径。

这些命令不会通过 MCP 公开 OpenClaw。它们管理 OpenClaw 配置中 `mcp.servers` 下由 OpenClaw 管理的 MCP 服务器定义。它们不会从 `config/mcporter.json` 读取 mcporter 服务器。

这些已保存的定义供 OpenClaw 稍后启动或配置的运行时使用，例如嵌入式 OpenClaw 和其他运行时适配器。OpenClaw 集中存储这些定义，因此这些运行时无需各自维护重复的 MCP 服务器列表。

<AccordionGroup>
  <Accordion title="重要行为">
    - 这些命令仅会读取或写入 OpenClaw 配置
    - `status`、`list`、`show`、不带 `--probe` 的 `doctor`、`set`、`configure`、`tools`、`logout`、`reload` 和 `unset` 不会连接目标 MCP 服务器
    - `login` 对已配置的 HTTP 服务器执行 MCP OAuth 网络流程，并保存生成的本地凭据
    - `status --verbose` 会输出解析后的传输、身份验证、超时、过滤器和并行工具调用提示，但不会连接
    - `doctor` 会检查已保存的定义是否存在本地设置问题，例如缺少 stdio 命令、工作目录无效、TLS 文件缺失、服务器被禁用、敏感标头/环境变量使用字面值，以及 OAuth 授权不完整
    - 静态检查通过后，`doctor --probe` 会添加与 `probe` 相同的实时连接证明
    - `probe` 会连接所选服务器或所有已配置的服务器，列出工具，并报告能力/诊断信息
    - `add` 会根据标志构建定义并在保存前进行探测，除非设置了 `--no-probe` 或需要先完成 OAuth 授权
    - 运行时适配器会在执行时决定它们实际支持哪些传输结构
    - `enabled: false` 会保留已保存的服务器，但将其排除在嵌入式运行时发现之外
    - `requestTimeoutMs` 和 `connectionTimeoutMs` 以毫秒为单位设置每台服务器的请求和连接超时
    - `supportsParallelToolCalls: true` 标记适配器可并发调用的服务器
    - HTTP 服务器可以使用静态标头、OAuth 登录、TLS 验证控制以及 mTLS 证书/密钥路径
    - 嵌入式 OpenClaw 会在常规 `coding` 和 `messaging` 工具配置文件中公开已配置的 MCP 工具；`minimal` 仍会隐藏它们，而 `tools.deny: ["bundle-mcp"]` 会明确禁用它们
    - 每台服务器的 `toolFilter.include` 和 `toolFilter.exclude` 会在发现的 MCP 工具成为 OpenClaw 工具之前对其进行过滤
    - 声明资源或提示词的服务器还会公开用于列出/读取资源以及列出/获取提示词的实用工具；这些生成的实用工具名称（`resources_list`、`resources_read`、`prompts_list`、`prompts_get`）使用相同的包含/排除过滤器
    - MCP 工具列表的动态更改会使该会话的缓存目录失效；下次发现/使用时会从服务器刷新
    - 重复的 MCP 工具请求/协议失败会使该服务器短暂暂停，避免单台故障服务器占用整个轮次
    - 会话范围的内置 MCP 运行时会在空闲 10 分钟后被回收，一次性嵌入式运行则会在运行结束时将其清理

  </Accordion>
</AccordionGroup>

运行时适配器可以将此共享注册表规范化为其下游客户端所期望的结构。例如，嵌入式 OpenClaw 直接使用 OpenClaw `transport` 值，而 Claude Code 和 Gemini 接收 CLI 原生的 `type` 值，例如 `http`、`sse` 或 `stdio`。

Codex app-server 还支持每台服务器上的可选 `codex` 块。这是
仅用于 Codex app-server 线程的 OpenClaw 投影元数据；它不会
更改 ACP 会话、通用 Codex harness 配置或其他运行时适配器。
使用非空的 `codex.agents`，可仅将服务器投影到特定 OpenClaw
智能体 ID。空白或无效的智能体列表会被配置
验证拒绝，并由运行时投影路径省略，而不会变为
全局设置。使用 `codex.defaultToolsApprovalMode`（`auto`、`prompt` 或 `approve`）
为可信服务器发出 Codex 原生的 `default_tools_approval_mode`。
OpenClaw 会先移除 `codex` 元数据，再将原生 `mcp_servers`
配置交给 Codex。

### 已保存的 MCP 服务器定义

命令：

- `openclaw mcp list`
- `openclaw mcp show [name]`
- `openclaw mcp status [--verbose]`
- `openclaw mcp doctor [name] [--probe]`
- `openclaw mcp probe [name]`
- `openclaw mcp add <name> [flags]`
- `openclaw mcp set <name> <json>`
- `openclaw mcp configure <name> [flags]`
- `openclaw mcp tools <name> [--include csv] [--exclude csv] [--clear]`
- `openclaw mcp login <name> [--code code]`
- `openclaw mcp logout <name>`
- `openclaw mcp reload`
- `openclaw mcp unset <name>`

说明：

- `list` 会对服务器名称进行排序。
- 不带名称的 `show` 会输出完整的已配置 MCP 服务器对象。
- `status` 会在不连接的情况下对已配置的传输进行分类。`--verbose` 包含解析后的启动、超时、OAuth、过滤器和并行调用详情，包括已存储的 OAuth 令牌何时需要额外授权。文本和 JSON 输出中的含凭据 stdio 参数会被遮盖。
- `doctor` 会在不连接的情况下执行静态检查。如果命令还应验证已启用服务器能否连接，请添加 `--probe`。
- `probe` 会连接并报告工具数量、资源/提示词支持、列表更改支持和诊断信息。
- `add` 接受 stdio 标志，例如 `--command`、`--arg`、`--env` 和 `--cwd`，或 HTTP 标志，例如 `--url`、`--transport`、`--header`、`--auth oauth`、TLS、超时和工具选择标志。
- `set` 要求在命令行中提供一个 JSON 对象值。
- `configure` 会更新启用状态、工具过滤器、超时、OAuth、TLS 和并行工具调用提示，而不会替换整个服务器定义。添加 `--probe` 可在保存前验证更新后的服务器。
- `tools` 会更新每台服务器的工具过滤器。包含/排除条目是 MCP 工具名称和简单的 `*` glob。
- `login` 会为配置了 `auth: "oauth"` 的 HTTP 服务器运行 OAuth 流程。首次运行会输出授权 URL；批准后使用 `--code` 重新运行。
- `logout` 会清除指定服务器已存储的 OAuth 凭据，但不会移除已保存的服务器定义。
- `reload` 仅会释放当前 CLI 进程中缓存的进程内 MCP 运行时。其他进程中的 Gateway 网关或智能体进程仍需执行各自的重新加载或重启流程。
- 对于 Streamable HTTP MCP 服务器，请使用 `transport: "streamable-http"`。为实现兼容性，`openclaw mcp set` 还会将 CLI 原生的 `type: "http"` 规范化为相同的标准配置结构。
- 如果指定的服务器不存在，`unset` 将失败。

示例：

```bash
openclaw mcp list
openclaw mcp show context7 --json
openclaw mcp status --verbose
openclaw mcp doctor --probe
openclaw mcp probe context7 --json
openclaw mcp add memory --command npx --arg -y --arg @modelcontextprotocol/server-memory
openclaw mcp set context7 '{"command":"uvx","args":["context7-mcp"]}'
openclaw mcp tools context7 --include 'resolve-library-id,get-library-docs'
openclaw mcp set docs '{"url":"https://mcp.example.com","transport":"streamable-http"}'
openclaw mcp configure docs --timeout 20 --connect-timeout 5 --include 'search,read_*'
openclaw mcp configure docs --auth oauth --oauth-scope 'docs.read'
openclaw mcp login docs
openclaw mcp logout docs
openclaw mcp unset context7
```

### 常用服务器配置方案

这些示例仅保存服务器定义。随后运行 `openclaw mcp doctor --probe`，以验证服务器能够启动并公开工具。

<Tabs>
  <Tab title="文件系统">
    ```bash
    openclaw mcp add files \
      --command npx \
      --arg -y \
      --arg @modelcontextprotocol/server-filesystem \
      --arg "$HOME/Documents" \
      --include 'read_file,list_directory,search_files'
    openclaw mcp doctor files --probe
    ```

    将文件系统服务器的作用域限制为智能体应读取或编辑的最小目录树。

  </Tab>
  <Tab title="内存">
    ```bash
    openclaw mcp add memory \
      --command npx \
      --arg -y \
      --arg @modelcontextprotocol/server-memory
    openclaw mcp probe memory --json
    ```

    如果服务器公开了不应供普通智能体使用的写入工具，请使用工具筛选器。

  </Tab>
  <Tab title="本地脚本">
    ```bash
    openclaw mcp add local-tools \
      --command node \
      --arg ./dist/mcp-server.js \
      --cwd /srv/openclaw-tools \
      --env API_BASE=https://internal.example
    openclaw mcp status --verbose
    ```

    `doctor` 会检查 `cwd` 是否存在，以及是否能从已配置的环境中解析该命令。

  </Tab>
  <Tab title="远程 HTTP">
    ```bash
    openclaw mcp add docs \
      --url https://mcp.example.com/mcp \
      --transport streamable-http \
      --auth oauth \
      --oauth-scope docs.read \
      --timeout 20 \
      --connect-timeout 5 \
      --include 'search,read_*'
    openclaw mcp doctor docs --probe
    ```

    当远程服务器支持 OAuth 时，请使用 OAuth。如果服务器需要静态标头，请避免提交明文承载令牌。

  </Tab>
  <Tab title="桌面/CUA">
    ```bash
    openclaw mcp set cua-driver '{"command":"cua-driver","args":["mcp"]}'
    openclaw mcp tools cua-driver --include 'list_apps,get_window_state,click,type_text'
    openclaw mcp doctor cua-driver --probe
    ```

    直接控制桌面的服务器会继承其所启动进程的权限。请使用严格的工具筛选器和操作系统级权限提示。

  </Tab>
</Tabs>

### JSON 输出结构

在脚本和仪表板中使用 `--json`。字段集可能会随时间增加，因此使用方应忽略未知键。

<AccordionGroup>
  <Accordion title="status --json">
    ```json
    {
      "path": "/home/user/.openclaw/openclaw.json",
      "servers": [
        {
          "name": "docs",
          "configured": true,
          "enabled": true,
          "ok": true,
          "transport": "streamable-http",
          "launch": "streamable-http https://mcp.example.com/mcp",
          "auth": "oauth",
          "authStatus": {
            "hasTokens": true,
            "requiresAuthorization": false,
            "hasClientInformation": true,
            "hasCodeVerifier": false,
            "hasDiscoveryState": true,
            "hasLastAuthorizationUrl": false
          },
          "requestTimeoutMs": 20000,
          "connectionTimeoutMs": 5000,
          "toolFilter": {
            "include": ["search", "read_*"],
            "exclude": []
          },
          "supportsParallelToolCalls": true
        }
      ]
    }
    ```
  </Accordion>
  <Accordion title="doctor --json">
    ```json
    {
      "ok": true,
      "path": "/home/user/.openclaw/openclaw.json",
      "servers": [
        {
          "name": "docs",
          "ok": true,
          "issues": [
            {
              "level": "warning",
              "message": "OAuth 凭据尚未授权；请运行 openclaw mcp login docs"
            }
          ]
        }
      ]
    }
    ```

    当任何已启用并接受检查的服务器存在 `error` 级别的问题时，`doctor --json` 会以非零状态退出。系统会报告 `warning` 和 `info` 问题，但它们本身不会导致命令失败。

  </Accordion>
  <Accordion title="probe --json">
    ```json
    {
      "generatedAt": "2026-05-31T09:00:00.000Z",
      "servers": {
        "docs": {
          "launch": "streamable-http https://mcp.example.com/mcp",
          "tools": 2,
          "resources": true,
          "listChanged": {
            "tools": true,
            "resources": false,
            "prompts": false
          }
        }
      },
      "tools": ["docs__read_page", "docs__search"],
      "diagnostics": []
    }
    ```

    `probe --json` 会打开实时 MCP 客户端会话并直接输出结果；与 `status`/`doctor` 不同，其输出没有顶层 `path` 字段。仅当服务器确实声明相应能力时，才会出现 `resources` 和 `prompts` 键（没有提示能力的服务器会省略 `prompts` 键，而不是报告 `false`）。使用 `probe` 验证可达性和能力，而不要将其用于静态配置审计。

  </Accordion>
</AccordionGroup>

配置结构示例：

```json
{
  "mcp": {
    "servers": {
      "context7": {
        "command": "uvx",
        "args": ["context7-mcp"]
      },
      "docs": {
        "url": "https://mcp.example.com",
        "transport": "streamable-http",
        "requestTimeoutMs": 20000,
        "connectionTimeoutMs": 5000,
        "supportsParallelToolCalls": true,
        "auth": "oauth",
        "oauth": {
          "scope": "docs.read"
        },
        "sslVerify": true,
        "clientCert": "/path/to/client.crt",
        "clientKey": "/path/to/client.key",
        "toolFilter": {
          "include": ["search_*"],
          "exclude": ["admin_*"]
        }
      }
    }
  }
}
```

### Stdio 传输

启动本地子进程，并通过 stdin/stdout 进行通信。

| 字段                       | 说明                         |
| -------------------------- | ---------------------------- |
| `command`                  | 要生成的可执行文件（必需）   |
| `args`                     | 命令行参数数组               |
| `env`                      | 额外的环境变量               |
| `cwd` / `workingDirectory` | 进程的工作目录               |

<Warning>
**Stdio 环境变量安全筛选器**

OpenClaw 会在生成 stdio MCP 服务器之前拒绝解释器启动、加载器劫持和 shell 初始化环境变量键，即使这些键出现在服务器的 `env` 块中也是如此。此机制采用与其他由 OpenClaw 生成的进程相同的宿主环境安全策略：它会阻止已知的解释器启动钩子（例如 `NODE_OPTIONS`、`PYTHONSTARTUP`、`PERL5OPT`、`RUBYOPT`、`BASHOPTS`、`KSH_ENV`）、共享库和函数注入前缀（`DYLD_*`、`LD_*`、`BASH_FUNC_*`）以及类似的运行时控制变量。启动时会静默丢弃这些变量并记录警告，使它们无法注入隐式前置代码、替换解释器、启用调试器或针对 stdio 进程劫持动态链接器。显式允许列表可确保常规 MCP 凭据环境变量仍然可用（`GITHUB_TOKEN`、`GH_TOKEN`、`GITLAB_TOKEN`、`NPM_TOKEN`、`NODE_AUTH_TOKEN`、`DATABASE_URL`、`MONGODB_URI`、`REDIS_URL`、`AMQP_URL`、`AWS_ACCESS_KEY_ID`、`AWS_SECRET_ACCESS_KEY`、`AWS_SESSION_TOKEN`、`AZURE_CLIENT_ID`、`AZURE_CLIENT_SECRET`），普通代理和服务器特定的环境变量也同样可用（`HTTP_PROXY`、自定义 `*_API_KEY` 等）。其他 `AWS_*` 键（例如 `AWS_CONFIG_FILE` 和 `AWS_SHARED_CREDENTIALS_FILE`）仍会被阻止，因为它们指向凭据文件，而不是直接携带凭据值。

如果你的 MCP 服务器确实需要其中某个被阻止的变量，请在 Gateway 网关宿主进程上进行设置，而不要在 stdio 服务器的 `env` 下设置。
</Warning>

### SSE / HTTP 传输

通过 HTTP 服务器发送事件连接到远程 MCP 服务器。

| 字段                        | 说明                                                        |
| --------------------------- | ----------------------------------------------------------- |
| `url`                       | 远程服务器的 HTTP 或 HTTPS URL（必需）                      |
| `headers`                   | 可选的 HTTP 标头键值映射（例如身份验证令牌）                |
| `connectionTimeoutMs`       | 每台服务器的连接超时时间，以毫秒为单位（可选）              |
| `requestTimeoutMs`          | 每台服务器的 MCP 请求超时时间，以毫秒为单位                 |
| `auth: "oauth"`             | 使用由 `openclaw mcp login` 保存的 MCP OAuth 凭据           |
| `sslVerify`                 | 仅对明确受信任的私有 HTTPS 端点设为 false                   |
| `clientCert` / `clientKey`  | mTLS 客户端证书和密钥路径                                  |
| `supportsParallelToolCalls` | 表明此服务器可安全进行并发调用的提示                        |

示例：

```json
{
  "mcp": {
    "servers": {
      "remote-tools": {
        "url": "https://mcp.example.com",
        "auth": "oauth",
        "requestTimeoutMs": 20000,
        "headers": {
          "Authorization": "Bearer <token>"
        }
      }
    }
  }
}
```

`url`（用户信息）和 `headers` 中的敏感值会在日志和状态输出中被隐去。当看起来敏感的 `headers` 或 `env` 条目包含明文值时，`openclaw mcp doctor` 会发出警告，以便操作员将这些值移出已提交的配置。

### OAuth 工作流

OAuth 适用于声明支持 MCP OAuth 流程的 HTTP MCP 服务器。启用 `auth: "oauth"` 时，服务器的静态 `Authorization` 标头会被忽略。由 `openclaw mcp login` 保存的凭据可用于嵌入式 MCP、CLI 运行器和本地 Codex app-server。

原生 MCP OAuth 会话存储在仅所有者可访问的共享 SQLite 数据库 `<state-dir>/state/openclaw.sqlite`（`mcp_oauth_stores`）中。该行可包含访问令牌和刷新令牌、动态客户端注册密钥、发现元数据以及临时 PKCE 验证器。刷新、登录和注销使用同一个 SQLite 租约，因此并行的 OpenClaw 进程无法消耗同一个刷新令牌或恢复已注销的会话。

从已停用的 `<state-dir>/mcp-oauth/*.json` 存储升级只能由 `openclaw doctor --fix` 处理。运行时代码绝不会读取、写入这些文件，也不会回退到这些文件。

在凭据可用之前，OpenClaw 只会从智能体运行时中省略该 MCP 服务器，而不会导致智能体轮次失败。操作员或具有 shell 访问权限的智能体随后可以运行 `openclaw mcp login <name>`，并在后续轮次中使用该服务器。

如果服务器以 `insufficient_scope` 拒绝令牌，OpenClaw 会保留请求的作用域并要求执行 `openclaw mcp login <name>`，而不是重复无法授予新作用域的刷新操作。该登录操作会启动新的授权请求，同时保留旧令牌，直至保存替代凭据。

当远程 MCP 服务已由单独的、支持刷新的 OpenClaw 身份验证配置文件提供支持时，可以选择设置 `oauth.authProfileId`。OpenClaw 会在运行时投影之前刷新任一凭据来源，并且仅将当前访问令牌传递给下游 MCP 客户端。

<Steps>
  <Step title="保存服务器">
    使用 `auth: "oauth"` 以及任何可选的 OAuth 元数据添加或更新服务器。

    ```bash
    openclaw mcp set docs '{"url":"https://mcp.example.com/mcp","transport":"streamable-http","auth":"oauth","oauth":{"scope":"docs.read"}}'
    ```

    对于由身份验证配置文件支持的 bearer，请保存配置文件绑定：

    ```bash
    openclaw mcp set docs '{"url":"https://mcp.example.com/mcp","transport":"streamable-http","auth":"oauth","oauth":{"authProfileId":"docs:mcp"}}'
    ```

  </Step>
  <Step title="开始登录">
    运行登录命令以创建授权请求。

    ```bash
    openclaw mcp login docs
    ```

    OpenClaw 会输出授权 URL，并将临时 OAuth 验证器状态存储在共享 SQLite 中。

  </Step>
  <Step title="使用代码完成登录">
    在浏览器中批准后，将返回的代码传回 OpenClaw。

    ```bash
    openclaw mcp login docs --code abc123
    ```

  </Step>
  <Step title="检查授权">
    使用状态或 Doctor 确认令牌存在且不需要额外授权。如果状态报告 `authorization-required`，或 Doctor 要求额外授权，请再次运行 `openclaw mcp login <name>`。

    ```bash
    openclaw mcp status --verbose
    openclaw mcp doctor docs --probe
    ```

  </Step>
  <Step title="清除凭据">
    注销会删除已存储的 OAuth 凭据，但保留已保存的服务器定义。

    ```bash
    openclaw mcp logout docs
    ```

  </Step>
</Steps>

如果提供商轮换令牌或授权状态卡住，请运行 `openclaw mcp logout <name>`，然后重复 `login`。即使 `auth: "oauth"` 已从配置中删除，只要服务器名称和 URL 仍可标识凭据存储条目，`logout` 仍可清除已保存 HTTP 服务器的凭据。

### 可流式传输的 HTTP 传输协议

`streamable-http` 是除 `sse` 和 `stdio` 之外的另一种传输协议选项。它使用 HTTP 流式传输与远程 MCP 服务器进行双向通信。

| 字段                       | 说明                                                                            |
| --------------------------- | -------------------------------------------------------------------------------------- |
| `url`                       | 远程服务器的 HTTP 或 HTTPS URL（必填）                                      |
| `transport`                 | 设置为 `"streamable-http"` 以选择此传输协议；省略时，OpenClaw 使用 `sse` |
| `headers`                   | 可选的 HTTP 标头键值映射（例如身份验证令牌）                       |
| `connectionTimeoutMs`       | 每台服务器的连接超时时间，单位为 ms（可选）                                         |
| `requestTimeoutMs`          | 每台服务器的 MCP 请求超时时间，单位为毫秒                                         |
| `auth: "oauth"`             | 使用由 `openclaw mcp login` 保存的 MCP OAuth 凭据                                |
| `sslVerify`                 | 仅对明确受信任的私有 HTTPS 端点设置为 false                          |
| `clientCert` / `clientKey`  | mTLS 客户端证书和密钥路径                                                  |
| `supportsParallelToolCalls` | 指示此服务器可安全处理并发调用                                    |

OpenClaw 配置将 `transport: "streamable-http"` 用作规范拼写。通过 `openclaw mcp set` 保存时，会接受 CLI 原生 MCP `type: "http"` 值，并由 `openclaw doctor --fix` 修复现有配置，但嵌入式 OpenClaw 直接使用的是 `transport`。

示例：

```json
{
  "mcp": {
    "servers": {
      "streaming-tools": {
        "url": "https://mcp.example.com/stream",
        "transport": "streamable-http",
        "connectionTimeoutMs": 10000,
        "requestTimeoutMs": 30000,
        "headers": {
          "Authorization": "Bearer <token>"
        }
      }
    }
  }
}
```

<Note>
注册表命令不会启动渠道桥接。只有 `probe` 和 `doctor --probe` 会打开实时 MCP 客户端会话，以验证目标服务器是否可访问。
</Note>

## Control UI

浏览器 Control UI 在 `/settings/mcp` 提供专用的 MCP 设置页面；之前的 `/mcp` 路径仍作为别名保留。该页面显示已配置服务器数量、启用/OAuth/筛选摘要、各服务器的传输协议行、启用/禁用控件、常用 CLI 命令，以及用于编辑 `mcp` 配置部分的限定范围编辑器。

使用该页面执行操作员编辑和快速清点。需要实时服务器验证时，请使用 `openclaw mcp doctor --probe` 或 `openclaw mcp probe`。

操作员工作流：

1. 打开 Control UI 并选择 **MCP**。
2. 查看摘要卡片中的服务器总数、已启用、OAuth 和已筛选数量。
3. 通过各服务器行查看传输协议、身份验证、筛选器、超时和命令提示。
4. 如果要保留定义但将其排除在运行时发现之外，请切换其启用状态。
5. 编辑限定范围的 `mcp` 配置部分，以进行新增服务器、标头、TLS、OAuth 元数据或工具筛选器等结构性更改。
6. 选择 **Save** 仅持久化配置，或选择 **Save & Publish** 通过 Gateway 网关配置路径应用配置。
7. 需要实时验证已编辑的服务器能够启动并列出工具时，请运行 `openclaw mcp doctor --probe`。

注意：

- 命令片段会引用服务器名称，以便名称特殊时仍可在 shell 中复制使用
- 显示的类 URL 值如果包含嵌入式凭据，会在渲染前进行脱敏
- 该页面本身不会启动 MCP 传输协议
- 根据 MCP 客户端所属的进程，活动运行时可能需要 `openclaw mcp reload`、发布 Gateway 网关配置或重启进程

## MCP Apps

OpenClaw 可以渲染实现稳定版 [MCP Apps 扩展](https://modelcontextprotocol.io/extensions/apps)的工具。Apps 需要主动启用，因为其 HTML 来自已配置的 MCP 服务器，并且可以从同一服务器请求对 App 可见的工具或资源。

启用主机桥接：

```bash
openclaw config set mcp.apps.enabled true --strict-json
```

更改此设置后，请重启 Gateway 网关。启用后，OpenClaw 会在 Gateway 网关端口加一的端口上启动一个仅用于沙箱的 HTTP(S) 监听器（对于默认 Gateway 网关，即 `18790`）。Control UI 从该独立来源加载 Apps；该监听器绝不会提供 Control UI、需要身份验证的 Gateway 网关路由或用户数据。

直接连接 Gateway 网关时需要能够访问这两个端口。如果反向代理或 TLS 终结器公开 Control UI，请为 Apps 提供专用的公共来源，并且仅将该来源代理到沙箱监听器：

```json5
{
  mcp: {
    apps: {
      enabled: true,
      sandboxOrigin: "https://mcp-apps.example.com",
      sandboxPort: 18790,
    },
  },
}
```

沙箱来源必须与 Control UI 来源不同。不要在其上托管其他需要身份验证或敏感的内容。

例如，可以按如下方式配置官方基础 React 演示：

```json5
{
  mcp: {
    apps: { enabled: true },
    servers: {
      "basic-react": {
        command: "npx",
        args: ["-y", "@modelcontextprotocol/server-basic-react", "--stdio"],
      },
    },
  },
}
```

行为和安全边界：

- 仅当 Apps 已启用时，OpenClaw 才会公布 `io.modelcontextprotocol/ui` 扩展。
- 仅渲染 MIME 类型与 `text/html;profile=mcp-app` 完全匹配的 `ui://` 资源。
- UI 资源上限为 2 MiB，置于专用外层来源上的双 iframe 代理之后，加载到不透明的内层 App 来源中，并受根据资源元数据派生的 CSP 约束。
- 仅限 App 使用的工具（`_meta.ui.visibility: ["app"]`）不会出现在模型工具列表中。Apps 只能调用其所属服务器上对 App 可见，并且也通过创建该视图的运行所采用的有效 OpenClaw 工具策略的工具。
- 当内层 App 文档使用不透明来源实现跨 App 隔离时，不会授予与来源绑定的 App 权限，例如摄像头、麦克风和地理位置权限。
- App HTML、完整工具参数和原始结果存放在有界的十分钟内存视图租约中，不会写入磁盘，也不会复制到对话记录预览元数据中。对话记录仅存储与原始工具调用 ID 绑定的有界服务器/工具/资源描述符。Gateway 网关重启后，Control UI 可以对照已通过身份验证的会话对话记录验证该描述符，并重新获取 `ui://` 资源；在新的运行建立当前工具权限之前，重建的视图为只读状态。
- 在渠道对话中，一个轮次内最新成功的 App 视图会向最终智能体回复添加一个 **打开 App** 样式的操作。Telegram 私信使用原生 Mini App 按钮；Slack 和 Discord 将同一可移植操作渲染为链接。其他渠道保留原始回复文本，并附加易于理解的 HTTPS 链接。
- 仅当 Gateway 网关的 Tailscale 暴露已准备好发布的 HTTPS 来源时，渠道启动链接才可用。`gateway.tailscale.mode: "serve"` 仅可从 tailnet 访问；`"funnel"` 可从公共互联网访问。由 `gateway.tailscale.preserveFunnel` 保留的外部托管 Funnel 也被视为可从互联网访问。请参阅 [Tailscale](/zh-CN/gateway/tailscale)。
- 启动票据是不透明的，仅在生成最终渠道回复时签发，并在最多两分钟后或底层视图租约到期时失效，以较早者为准。URL 不包含 Gateway 网关 bearer 凭据、会话密钥、视图元数据、App HTML、工具输入或工具结果。
- 如果没有可用的已发布来源或票据容量、视图或票据已过期，或者传输协议无法渲染原生控件，原始智能体文本仍然可用。Control UI 会保留其现有的内嵌 App 画布，并且不会收到重复的启动操作。
- 启用桥接时，`openclaw security audit` 会发出警告。不需要时，请使用 `openclaw config set mcp.apps.enabled false --strict-json` 将其禁用。

## 当前限制

本页记录当前已发布的桥接实现。

当前限制：

- 对话发现依赖现有 Gateway 网关会话路由元数据
- 除 Claude 专用适配器外，尚无通用推送协议
- 目前尚无消息编辑或表情回应工具
- HTTP/SSE/streamable-http 传输协议连接到单个远程服务器；尚不支持多路复用上游
- `permissions_list_open` 仅包含桥接连接期间观察到的审批

## 相关内容

- [CLI 参考](/zh-CN/cli)
- [插件](/zh-CN/cli/plugins)
