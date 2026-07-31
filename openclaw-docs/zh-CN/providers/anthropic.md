---
read_when:
    - 你想在 OpenClaw 中使用 Anthropic 模型
    - 你想浏览已配对计算机上的 Claude CLI 或 Claude Desktop 会话
summary: 在 OpenClaw 中通过 API 密钥或 Claude CLI 使用 Anthropic Claude
title: Anthropic
x-i18n:
    generated_at: "2026-07-26T06:23:43Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 08b34794352a559d549f7cf0cb88aca9cb537984049367f55be371bd8e0c10f0
    source_path: providers/anthropic.md
    workflow: 16
---

Anthropic 构建了 **Claude** 模型系列。OpenClaw 支持两种身份验证方式：

- **API 密钥** - 通过按用量计费直接访问 Anthropic API（`anthropic/*` 模型）
- **Claude CLI** - 复用同一主机上现有的 Claude Code 登录

## 用量和成本跟踪

OpenClaw 会检测可用的 Anthropic 凭据，并选择相应的用量界面：

- Claude 订阅/设置凭据会显示配额周期和可选的额外用量预算。
- `ANTHROPIC_ADMIN_KEY` 或 `ANTHROPIC_ADMIN_API_KEY` 会在 Control UI 的 **用量** 中显示提供商报告的过去 30 天组织成本和 Messages API 用量，包括每日支出、令牌/缓存总量、热门模型和成本类别。
- 存储在 Anthropic 提供商配置文件中的 `sk-ant-admin...` 凭据会被自动检测为 Admin API 密钥。

Admin API 成本历史记录来自 Anthropic 的[用量和成本 API](https://platform.claude.com/docs/en/manage-claude/usage-cost-api)。这是提供商的实际账单，与 OpenClaw 根据会话估算的成本相互独立。

<Warning>
OpenClaw 的 Claude CLI 后端以非交互式打印模式
（`claude -p`）运行已安装的 Claude Code CLI。Anthropic 当前的 Claude Code 文档
将该模式描述为 Agent SDK/编程式用法。Anthropic 于 2026 年 6 月 15 日发布的
支持更新暂停了已宣布的独立 Agent SDK 计费变更：Claude
Agent SDK、`claude -p` 和第三方应用用量仍会计入已登录
订阅的用量限制，而此前宣布的每月 Agent SDK
额度在 Anthropic 修订该计划期间不可用。

交互式 Claude Code 仍会计入已登录 Claude 套餐的限制。
API 密钥身份验证采用直接按用量付费的计费方式，不依赖该套餐。
对于长期运行的 Gateway 网关主机、共享自动化和需要可预测生产
支出的场景，请使用 Anthropic API 密钥。

Anthropic 当前的支持文章可能会在不发布
OpenClaw 新版本的情况下更改此行为：

- [Claude Code CLI 参考](https://code.claude.com/docs/en/cli-usage)
- [在 Claude 套餐中使用 Claude Agent SDK](https://support.claude.com/en/articles/15036540-use-the-claude-agent-sdk-with-your-claude-plan)
- [在 Pro 或 Max 套餐中使用 Claude Code](https://support.claude.com/en/articles/11145838-use-claude-code-with-your-pro-or-max-plan)
- [在 Team 或 Enterprise 套餐中使用 Claude Code](https://support.claude.com/en/articles/11845131-using-claude-code-with-your-team-or-enterprise-plan)
- [管理 Claude Code 成本](https://code.claude.com/docs/en/costs)

</Warning>

## 入门指南

<Tabs>
  <Tab title="API 密钥">
    **最适合：** 标准 API 访问和按用量计费。

    <Steps>
      <Step title="获取 API 密钥">
        在 [Anthropic Console](https://console.anthropic.com/) 中创建 API 密钥。
      </Step>
      <Step title="运行新手引导">
        ```bash
        openclaw onboard
        # 选择：Anthropic API 密钥
        ```

        或直接传入密钥：

        ```bash
        openclaw onboard --anthropic-api-key "$ANTHROPIC_API_KEY"
        ```
      </Step>
      <Step title="验证模型是否可用">
        ```bash
        openclaw models list --provider anthropic
        ```
      </Step>
    </Steps>

    ### 配置示例

    ```json5
    {
      env: { ANTHROPIC_API_KEY: "example-anthropic-key-not-real" },
      agents: { defaults: { model: { primary: "anthropic/claude-opus-5" } } },
    }
    ```

  </Tab>

  <Tab title="Claude CLI">
    **最适合：** 无需单独的 API 密钥即可复用现有 Claude CLI 登录。

    <Steps>
      <Step title="确保 Claude CLI 已安装且已登录">
        使用以下命令验证：

        ```bash
        claude --version
        ```
      </Step>
      <Step title="运行新手引导">
        ```bash
        openclaw onboard
        # 选择：Claude CLI
        ```

        OpenClaw 会检测并复用现有的 Claude CLI 凭据。
      </Step>
      <Step title="验证模型是否可用">
        ```bash
        openclaw models list --provider anthropic
        ```
      </Step>
    </Steps>

    <Note>
    Claude CLI 后端的设置和运行时详情请参阅 [CLI 后端](/zh-CN/gateway/cli-backends)。
    </Note>

    <Warning>
    复用 Claude CLI 要求 OpenClaw 进程与 Claude CLI 登录运行在同一主机上。
    Docker 安装可以持久化容器主目录，并在其中登录
    Claude Code；请参阅
    [Docker 中的 Claude CLI 后端](/zh-CN/install/docker#claude-cli-backend-in-docker)。
    [Podman](/zh-CN/install/podman) 等其他容器安装不会在设置或运行时挂载主机的
    `~/.claude`；请在其中使用 Anthropic API 密钥，或选择
    由 OpenClaw 管理 OAuth 的提供商，例如
    [OpenAI Codex](/zh-CN/providers/openai)。
    </Warning>

    ### 获取设置令牌

    在任何已安装 Claude Code 的计算机上运行 `claude setup-token`。它会输出
    一个以 `sk-ant-oat01-` 开头的长期令牌。

    在新手引导期间，在 macOS 应用中依次选择
    **Anthropic setup-token** 和 **Connect with an API key or token**，然后粘贴该令牌；也可以使用：

    ```bash
    openclaw models auth login --provider anthropic --method setup-token
    ```

    ### 配置示例

    建议使用规范的 Anthropic 模型引用，并添加 CLI 运行时覆盖：

    ```json5
    {
      agents: {
        defaults: {
          model: { primary: "anthropic/claude-opus-5" },
          models: {
            "anthropic/claude-opus-5": {
              agentRuntime: { id: "claude-cli" },
            },
          },
        },
      },
    }
    ```

    为保持兼容性，旧版 `claude-cli/claude-opus-4-7` 模型引用仍然有效，
    但新配置应将提供商/模型选择保持为
    `anthropic/*`，并将执行后端放入提供商/模型运行时策略中。

    ### 计费和 `claude -p`

    OpenClaw 使用 Claude Code 的非交互式 `claude -p` 路径运行 Claude CLI。
    Anthropic 当前将该路径视为 Agent SDK/编程式用法：

    - Anthropic 于 2026 年 6 月 15 日发布的支持更新暂停了此前宣布的
      独立 Agent SDK 额度计划。
    - 订阅套餐中的 Claude Agent SDK、`claude -p` 和第三方应用用量
      仍会计入已登录订阅的用量限制。
    - 在 Anthropic 修订该计划期间，此前宣布的每月 Agent SDK 额度
      不可用。
    - Console/API 密钥登录采用按用量付费的 API 计费方式，不会获得
      订阅套餐的 Agent SDK 额度。

    有关暂停通知，请参阅 Anthropic 的 [Agent SDK 套餐
    文章](https://support.claude.com/en/articles/15036540-use-the-claude-agent-sdk-with-your-claude-plan)；有关订阅行为，请参阅 Claude Code 套餐文章：
    [Pro/Max](https://support.claude.com/en/articles/11145838-use-claude-code-with-your-pro-or-max-plan)
    和
    [Team/Enterprise](https://support.claude.com/en/articles/11845131-use-claude-code-with-your-team-or-enterprise-plan)。

    Anthropic 可以在不发布 OpenClaw 新版本的情况下更改 Claude Code 的计费和速率限制行为。
    如果计费的可预测性很重要，请检查 `claude auth status`、`/status` 和
    Anthropic 的链接文档。

    <Tip>
    对于共享的生产自动化，请使用 Anthropic API 密钥，而不是
    Claude CLI。OpenClaw 还支持以下提供商提供的订阅式选项：
    [OpenAI Codex](/zh-CN/providers/openai)、[Qwen Cloud](/zh-CN/providers/qwen)、
    [MiniMax](/zh-CN/providers/minimax) 和 [Z.AI / GLM](/zh-CN/providers/zai)。
    </Tip>

  </Tab>
</Tabs>

## 跨计算机的 Claude 会话

内置的 Anthropic 插件会在常规会话侧边栏中添加一个 **Claude Code** 分组。
点击行后会在常规聊天窗格中打开。它会发现 Gateway 网关和已连接节点主机上
未归档的 Claude Code 会话：

- Claude CLI 会话来自有效的项目索引记录。对于未编入索引的
  对话记录，有限的元数据回退机制会识别 `~/.claude/projects/` 下并发的非 sidechain
  交互式（`cli`）会话和无头 Agent SDK CLI（`sdk-cli`）会话。
- 当 Claude Desktop 的元数据指向同一个 Claude Code 会话 ID 时，
  Claude Desktop 会话会使用 Desktop 标题、活动时间和归档状态。
- 仅限 CLI 的会话没有归档标志，因此只要其对话记录仍然存在，
  它就会保持可见。

设备发现无需额外的 OpenClaw 配置。Anthropic 插件已内置且默认启用；
当本地 `~/.claude/projects/` 目录存在时，原生 macOS 节点会公布只读的
Claude 会话命令。这些命令首次出现时，请批准节点配对升级。

侧边栏按 Gateway 网关或已配对节点主机对行进行分组，并在每台计算机响应后立即显示
该主机最新的有限页面。主机连接状态发生变化、页面重新获得焦点时，它会再次进行协调；
页面可见期间最多每 30 秒协调一次，因此在 OpenClaw 外部创建的 Claude 会话无需重新加载
即可显示。目录发生变化时，会更快地执行一次后续检查。使用目录分组下方的**加载更多
会话**，可为仍有更多历史记录的每台主机追加下一页；追加的行会保持可见，并在刷新时
重新获取到相同深度。目录客户端使用 `sessions.catalog.list`；打开行时使用
`sessions.catalog.read`。

终端接管会先从所属主机用户的登录 shell PATH 中解析 `claude`，
然后再从服务/守护进程 PATH 中解析。这样可确保由应用启动的会话与操作员在普通终端中
使用的 Claude CLI 保持一致。

选择一行后，会先读取最新的对话记录页面。**加载更早的对话记录
项目**会沿着不透明字节游标，从 JSONL 文件中读取另一个有限区段，而不是加载全部历史记录。
常规的用户、助手、推理、工具调用和工具结果内容都会保留。超出节点/Gateway 网关安全上限的
单个项目会被明确标记为已截断。

对于 Gateway 网关本地的 `claude-cli` 行，在常规编辑器中输入内容会调用
`sessions.catalog.continue`。OpenClaw 会重新解析本地目录记录，
创建或复用锁定模型的原生会话，导入最多 200 个可见项目或 512 KiB，
并初始化 Claude CLI 绑定。首轮会使用 `--fork-session` 恢复；
Claude 会为分叉分配新的会话 ID，因此后续轮次会使用该分叉，源会话保持不变。

无头节点主机也可以通过启用以下节点本地设置并重启节点主机，使其 Claude CLI 行可继续：

```json5
{
  nodeHost: {
    agentRuns: {
      claude: { enabled: true },
    },
  },
}
```

仅当该设置已启用且本地 `claude` 可执行文件可解析时，
节点才会公布 `agent.cli.claude.run.v1`。OpenClaw 会在该节点上重新解析目录
记录，导入相同的有限历史记录，并将接管的会话绑定到该节点以及目录报告的工作目录。
每一轮都会使用该节点的 Claude 文件和登录信息，运行该节点真实的
`claude -p` 进程。节点的 Exec 审批策略仍然适用；Gateway 网关无法强制启用该选项。

节点续接 v1 仅支持单次运行。它不包含 Gateway 网关 local loopback MCP 配置和
Gateway 网关 Skills 插件参数，不会从 Gateway 网关对话记录重新初始化，并且会拒绝附件和图像。
Claude Desktop 行仍然只能查看。原生 macOS 应用节点也会保持只能查看，直到应用公布运行命令。

<Note>
除非无头节点明确公布 `agent.cli.claude.run.v1`，否则已配对节点上的 Claude 会话仍然
是只读的。OpenClaw 永远不会修改 Claude Desktop 元数据或归档 Claude 会话。
该页面需要具有写入权限范围的操作员连接，因为它使用经过身份验证的
`node.invoke`；即使节点已启用续接，列出和读取操作也仍然是只读的。
</Note>

请参阅[节点：Claude 会话和转录记录](/zh-CN/nodes#claude-sessions-and-transcripts)，
了解节点命令和安全边界。

## 思考默认设置（Claude Opus 5、Sonnet 5、Mythos 5、Fable 5、4.8 和 4.6）

`anthropic/claude-opus-5` 默认使用 `high` 强度的自适应思考。
使用 `/think off` 可禁用思考，使用 `/think xhigh|max` 可启用模型原生的
更高强度级别。对于 Opus 5，OpenClaw 不会设置手动思考预算、自定义
采样参数、助手预填充和 Priority Tier，因为 Anthropic 不支持此模型使用
这些请求功能。目录中公布了其 1,000,000 token 上下文窗口、128,000 token
输出限制、图像输入以及 `$5/$25` 输入/输出定价。

`anthropic/claude-sonnet-5` 使用相同的自适应思考默认设置和请求
限制。目录在 2026 年 8 月 31 日之前采用 Anthropic 的入门 `$2/$10` 输入/输出
定价；标准 `$3/$15` 定价将于 2026 年 9 月 1 日开始。

`anthropic/claude-fable-5` 始终使用自适应思考，默认强度为 `high`。
Anthropic 不允许为此模型禁用思考，因此
`/think off` 和 `/think minimal` 会改为映射到 `low` 强度。OpenClaw 还会
忽略 Fable 5 请求中的自定义温度值，因为 Anthropic 会拒绝
任何启用思考的请求所携带的温度覆盖值。

`anthropic/claude-mythos-5` 是一种访问受限的模型，采用相同的始终启用
自适应思考约定。OpenClaw 默认使用 `high`，将 `/think off` 和
`/think minimal` 映射到 `low`，并忽略调用方选择的采样参数。
目录中公布了其 1,000,000 token 上下文窗口、128,000 token 输出
限制、图像输入以及 `$10/$50` 输入/输出定价。

在 OpenClaw 中，Claude Opus 4.8 默认关闭思考。当你使用
`/think high|xhigh|max` 显式启用自适应思考时，OpenClaw 会发送
Anthropic 的 Opus 4.8 强度值；Claude 4.6 模型（Opus 4.6 和 Sonnet 4.6）
默认使用 `adaptive`。

可使用 `/think:<level>` 按消息覆盖，或在模型参数中配置：

```json5
{
  agents: {
    defaults: {
      models: {
        "anthropic/claude-opus-5": {
          params: { thinking: "high" },
        },
      },
    },
  },
}
```

<Note>
Anthropic 相关文档：
- [自适应思考](https://platform.claude.com/docs/en/build-with-claude/adaptive-thinking)
- [扩展思考](https://platform.claude.com/docs/en/build-with-claude/extended-thinking)

</Note>

## 安全拒绝回退（Claude Fable 5）

<Warning>
使用 Claude Fable 5 也意味着会使用 Claude Opus 4.8。Fable 5 随附
可拒绝请求的安全分类器，而 Anthropic 认可的恢复方式是让
`claude-opus-4-8` 处理该轮次。对于直接使用 API key 的请求，OpenClaw 会自动
启用此机制，因此某些 Fable 轮次将由 Claude Opus 4.8 回答并按其计费。
如果你的策略或预算无法接受由 Opus 处理的轮次，请勿选择
`anthropic/claude-fable-5`。
</Warning>

### 此机制为何存在

Fable 5 分类器会对受限领域的请求返回 `stop_reason: "refusal"`，
也会对与这些领域相邻的无害工作产生误报（安全
工具、生命科学，甚至要求模型复现其原始
推理）。如果没有回退，即使另一个 Claude 模型愿意处理请求，
该轮次仍会以错误结束——Anthropic 自己的拒绝消息会要求
API 集成方配置回退模型。

### 工作原理

1. 对于向 `anthropic/claude-fable-5` 发出的每个直接 API key 请求，OpenClaw
   都会发送 Anthropic 的服务端回退选择加入设置：
   `server-side-fallback-2026-06-01` beta 标头以及
   `fallbacks: [{"model": "claude-opus-4-8"}]`。Claude Opus 4.8 是 Anthropic 唯一允许
   Fable 5 使用的回退目标。
2. 只有安全分类器拒绝才会触发回退。速率限制、
   过载和服务器错误的行为与此前完全相同，并通过
   OpenClaw 的常规[模型故障转移](/zh-CN/concepts/model-failover)机制处理。
3. 补救过程发生在同一次调用中。在生成任何输出前发生的拒绝，
   除延迟外不可见；整个回答都来自 Opus 4.8。如果在
   流式传输过程中拒绝，则保留部分文本作为回退模型继续生成的前缀，
   而被拒绝模型的推理和工具调用会按照 Anthropic 的重放规则
   丢弃（不得将其原样返回或执行）。
4. 如果 Claude Opus 4.8 也拒绝，该轮次会将拒绝作为
   错误返回，与此功能推出前完全相同。

回退发生在 Anthropic API 层，因此无需将 `claude-opus-4-8`
加入你配置的模型列表或回退链——能够使用 Fable 的
API key 始终可以调用 Opus。

### 可观测性和计费

- 由回退模型处理的轮次会在助手消息中记录 `provider_fallback` 诊断信息，
  其中会注明 `fromModel` 和 `toModel`，且该消息的
  `responseModel` 会报告 `claude-opus-4-8`。
- Anthropic 按尝试计费：在输出前拒绝不收费，补救请求
  按 Claude Opus 4.8 费率计费（目前是 Fable 5 费率的一半）。OpenClaw
  会按 Opus 费率估算由回退模型处理的轮次费用，以保持一致。
- 如果在流式传输过程中拒绝，Anthropic 还会对已流式输出的 Fable 部分
  计费；该部分会在 API 的各次尝试用量中报告，
  但不会计入 OpenClaw 的单轮费用估算。

### 适用范围

适用于使用 API key 身份验证并请求
`api.anthropic.com` 的 `anthropic/claude-fable-5`。OAuth（复用 Claude CLI 订阅）、
代理基础 URL、Bedrock、Vertex 和 Foundry 请求不受影响，
在这些路径中仍会将拒绝作为错误返回。

实时验证结果：在未设置回退时，向 Fable 5 发送一个要求其复现原始思维链的
无害提示词会被拒绝，并返回 `category: "reasoning_extraction"`；
而通过 OpenClaw 发送相同提示词时，会正常返回由 Opus 处理的回答，
并附带 `provider_fallback` 诊断信息。

有关底层行为，请参阅 Anthropic 的[拒绝和回退
指南](https://platform.claude.com/docs/en/build-with-claude/refusals-and-fallback)。

## 提示词缓存

OpenClaw 支持 Anthropic 面向 API key 身份验证的提示词缓存功能。

| 值               | 缓存时长 | 说明                            |
| ------------------- | -------------- | -------------------------------------- |
| `"short"`（默认） | 5 分钟      | 自动应用于 API key 身份验证 |
| `"long"`            | 1 小时         | 延长缓存                         |
| `"none"`            | 不缓存     | 禁用提示词缓存                 |

```json5
{
  agents: {
    defaults: {
      models: {
        "anthropic/claude-opus-4-6": {
          params: { cacheRetention: "long" },
        },
      },
    },
  },
}
```

<AccordionGroup>
  <Accordion title="按智能体覆盖缓存设置">
    以模型级参数作为基准，然后通过 `agents.entries.*.params` 覆盖特定智能体：

    ```json5
    {
      agents: {
        defaults: {
          model: { primary: "anthropic/claude-opus-4-6" },
          models: {
            "anthropic/claude-opus-4-6": {
              params: { cacheRetention: "long" },
            },
          },
        },
        list: [
          { id: "research", default: true },
          { id: "alerts", params: { cacheRetention: "none" } },
        ],
      },
    }
    ```

    配置合并顺序：

    1. `agents.defaults.models["provider/model"].params`
    2. `agents.entries.*.params`（匹配 `id`，按键覆盖）

    这样，一个智能体可以保留长期缓存，而同一模型上的另一个智能体可以针对突发性、低复用流量禁用缓存。

  </Accordion>

  <Accordion title="Bedrock Claude 注意事项">
    - Bedrock 上的 Anthropic Claude 模型（`amazon-bedrock/*anthropic.claude*`）在配置后接受 `cacheRetention` 透传。
    - 非 Anthropic Bedrock 模型会在运行时被强制设为 `cacheRetention: "none"`。
    - 如果未设置显式值，API key 智能默认值还会为 Bedrock 上的 Claude 引用填充 `cacheRetention: "short"`。

  </Accordion>
</AccordionGroup>

## 高级配置

<AccordionGroup>
  <Accordion title="快速模式">
    OpenClaw 的共享 `/fast` 开关会为直接使用 API key 访问 `api.anthropic.com` 的流量设置 Anthropic 的 `service_tier` 字段。

    | 命令 | 映射为 |
    |---------|---------|
    | `/fast on` | `service_tier: "auto"` |
    | `/fast off` | `service_tier: "standard_only"` |

    ```json5
    {
      agents: {
        defaults: {
          models: {
            "anthropic/claude-sonnet-4-6": {
              params: { fastMode: true },
            },
          },
        },
      },
    }
    ```

    <Note>
    - 仅适用于使用 API key 发出的直接 `api.anthropic.com` 请求。OAuth/订阅令牌请求和代理路由绝不会收到 `service_tier` 字段。
    - 同时设置时，显式的 `serviceTier` 或 `service_tier` 参数会覆盖 `/fast`。
    - Claude Opus 5 和 Sonnet 5 不支持 Priority Tier，因此 OpenClaw 会为这些模型忽略 `service_tier`。
    - 对于没有 Priority Tier 容量的账户，`service_tier: "auto"` 可能会解析为 `standard`。

    </Note>

  </Accordion>

  <Accordion title="媒体理解（图像和 PDF）">
    内置 Anthropic 插件会注册图像和 PDF 理解功能。OpenClaw
    会根据已配置的 Anthropic 身份验证自动解析媒体能力；
    无需额外配置。

    | 属性        | 值                 |
    | --------------- | --------------------- |
    | 默认模型   | `claude-opus-5`       |
    | 支持的输入 | 图像、PDF 文档 |

    当对话中附加图像或 PDF 时，OpenClaw 会自动
    通过 Anthropic 媒体理解提供商处理它。

  </Accordion>

  <Accordion title="1M 上下文窗口">
    Claude Opus 5、Sonnet 5、Mythos 5 和 Fable 5 具有精确的
    1,000,000 token 输入窗口，并支持最多 128,000 个输出 token。
    Anthropic 的 1M 上下文窗口也已在支持自适应思考的 Claude 4.x 模型上正式发布：
    Opus 4.8、
    Opus 4.7、Opus 4.6 和 Sonnet 4.6。OpenClaw 会自动为这些模型确定大小，
    无需 `params.context1m`：

    ```json5
    {
      agents: {
        defaults: {
          models: {
            "anthropic/claude-opus-5": {},
            "anthropic/claude-sonnet-5": {},
            "anthropic/claude-mythos-5": {},
            "anthropic/claude-opus-4-6": {},
          },
        },
      },
    }
    ```

    旧配置可以保留 `params.context1m: true`；对于
    这些模型，它是无害的空操作，而且无论如何，OpenClaw 都不再发送已停用的
    `context-1m-2025-08-07` beta 标头。请求标头解析期间会丢弃值为该内容的旧 `anthropicBeta`
    配置条目，而不受支持的旧 Claude 模型仍使用其常规上下文窗口。

    对于 Claude CLI 后端（`claude-cli/*`），`params.context1m: true`
    的行为相同：符合条件且支持正式版功能的 Opus 和 Sonnet 模型已经会自动获得
    1M 窗口，因此该参数在那里也是可选的。

    <Warning>
    需要你的 Anthropic 凭据具有长上下文访问权限。OAuth/订阅令牌身份验证会保留其必需的 Anthropic beta 标头，但如果旧配置中仍存在已停用的 1M beta 标头，OpenClaw 会将其移除。
    </Warning>

  </Accordion>

  <Accordion title="Claude Opus 5 1M 上下文">
    `anthropic/claude-opus-5` 及其 `claude-cli` 变体默认具有 1M 上下文
    窗口；无需 `params.context1m: true`。
  </Accordion>
</AccordionGroup>

## 故障排查

<AccordionGroup>
  <Accordion title="401 错误/令牌突然失效">
    Anthropic 令牌身份验证会过期，也可能被撤销。对于新设置，请改用 Anthropic API key。
  </Accordion>

  <Accordion title='未找到提供商 "anthropic" 的 API key'>
    Anthropic 身份验证**按智能体独立配置**；新智能体不会继承主智能体的密钥。为该智能体重新运行新手引导（或在 Gateway 网关主机上配置 API key），然后使用 `openclaw models status` 验证。
  </Accordion>

  <Accordion title='未找到配置文件 "anthropic:default" 的凭据'>
    运行 `openclaw models status` 查看当前使用的身份验证配置文件。重新运行新手引导，或为该配置文件路径配置 API key。
  </Accordion>

  <Accordion title="没有可用的身份验证配置文件（全部处于冷却期）">
    检查 `openclaw models status --json` 中的 `auth.unusableProfiles`。Anthropic 的速率限制冷却期可能仅适用于特定模型，因此同属 Anthropic 的其他模型可能仍可使用。添加另一个 Anthropic 配置文件，或等待冷却期结束。
  </Accordion>
</AccordionGroup>

<Note>
更多帮助：[故障排查](/zh-CN/help/troubleshooting)和[常见问题](/zh-CN/help/faq)。
</Note>

## 相关内容

<CardGroup cols={2}>
  <Card title="模型选择" href="/zh-CN/concepts/model-providers" icon="layers">
    选择提供商、模型引用和故障转移行为。
  </Card>
  <Card title="CLI 后端" href="/zh-CN/gateway/cli-backends" icon="terminal">
    Claude CLI 后端设置和运行时详情。
  </Card>
  <Card title="提示词缓存" href="/zh-CN/reference/prompt-caching" icon="database">
    提示词缓存如何在不同提供商之间工作。
  </Card>
  <Card title="OAuth 和身份验证" href="/zh-CN/gateway/authentication" icon="key">
    身份验证详情和凭据复用规则。
  </Card>
</CardGroup>
