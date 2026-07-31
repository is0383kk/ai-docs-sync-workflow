---
read_when:
    - 你正在接入提供商用量/配额界面
    - 你需要解释用量跟踪行为或身份验证要求
summary: 用量跟踪界面和凭据要求
title: 用量跟踪
x-i18n:
    generated_at: "2026-07-26T06:43:17Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 5a1bc9aeb95cd80a48ab57a18fcd24894fdd6fb71e10e8bea8bae67a8688b78e
    source_path: concepts/usage-tracking.md
    workflow: 16
---

## 它是什么

- 直接从每个提供商的用量端点提取提供商用量/配额。不估算提供商账单；仅提供提供商报告的套餐名称、配额窗口、余额、支出、预算、每日成本历史、Token/模型归因或账户状态摘要。
- 人类可读的配额窗口输出统一规范化为 `X% left`，即使提供商报告的是已消耗配额、剩余配额或仅原始计数。不提供可重置配额窗口的提供商将改为显示提供商摘要文本（例如余额）。
- 当实时会话快照缺少 Token/模型数据时，会话级 `/status` 和 `session_status` 工具会回退到会话的转录日志。此回退会补全缺失的 Token/缓存计数器，可以恢复当前运行时模型标签，并且当会话元数据缺失或较小时（`totalTokensFresh !== true`、零或低于从转录记录推导的值），优先采用面向提示词的较大总数。非零实时值始终优先于回退值。

## 显示位置

- 聊天中的 `/status`：显示会话 Token 和估算成本的状态卡（仅限 API key 模型）。如果可用，将显示**当前模型提供商**的提供商用量，形式为规范化的 `X% left` 窗口或提供商摘要文本。
- 聊天中的 `/usage off|tokens|full`：每次响应的用量页脚。
- 聊天中的 `/usage cost`：根据 OpenClaw 会话日志汇总的本地成本摘要。
- CLI：`openclaw status --usage` 输出完整的各提供商用量/配额明细。
- CLI：`openclaw models status` 列出 OAuth/Token 身份验证配置文件，并在每个具有用量窗口的提供商旁显示用量窗口摘要。
- Control UI：**用量**会在 OpenClaw 根据会话推导的 Token 和估算成本分析上方显示提供商套餐及账单卡片。Anthropic 和 OpenAI Admin API 凭据还会添加提供商报告的今日、7 天及 30 天支出、每日趋势、Token 总数、热门模型和成本类别。
- Control UI：聊天输入框的上下文环形弹出框会显示订阅提供商的**套餐用量**——各窗口进度条（5 小时、每周、模型范围），包括重置时间、已知的提供商套餐（例如 `Max (20x)`）以及额外用量额度。通过套餐计费的会话会隐藏按 Token 计算的美元估值；通过 API 计费的会话则保留 `Est. cost` 和按类型划分的成本明细。Claude Code CLI（`claude-cli`）设置会复用相同的 Anthropic 订阅用量。
- macOS 菜单栏：当提供商用量快照可用时，根级“用量”部分会显示在“上下文”下方。请参阅[菜单栏](/zh-CN/platforms/mac/menu-bar)。

`openclaw channels list` 不再输出提供商用量；它会改为引导用户使用 `openclaw status` 或 `openclaw models list`。

## Anthropic 和 OpenAI 成本历史

订阅配额与 API 账单是不同的提供商界面：

- Anthropic 订阅/设置凭据继续显示 Claude 配额窗口和可选的额外用量预算。设置 `ANTHROPIC_ADMIN_KEY` 或 `ANTHROPIC_ADMIN_API_KEY` 可改为显示组织 Usage and Cost API 历史记录。以 `sk-ant-admin` 开头的 Anthropic 提供商凭据会被自动检测。
- OpenAI ChatGPT/Codex OAuth 继续显示套餐、配额窗口和额度余额。设置 `OPENAI_ADMIN_KEY` 可改为显示组织成本和补全用量历史记录；还可选择设置 `OPENAI_PROJECT_ID`，将范围限定到单个项目。OpenClaw 绝不会将 `OPENAI_API_KEY`、提供商配置或身份验证配置文件中的推理凭据发送到组织 API，因为这些密钥可能属于自定义端点。

Admin 凭据具有优先权，因为它们提供实际的组织账单数据。OpenClaw 不会将这些提供商报告的总数与本地会话估算值合并；这两个部分有意回答不同的问题。

## 默认用量页脚模式

`/usage off|tokens|full` 设置会话的页脚，并会为该
会话记住此设置。对于尚未选择模式的会话，`messages.responseUsage` 会设定初始模式，
因此无需每次输入 `/usage`，页脚也可以默认启用。

可以为所有渠道设置一个模式，也可以设置按渠道划分的映射，并提供 `default` 回退：

```jsonc
{
  "messages": {
    "responseUsage": "tokens",
    // 或：{ "default": "off", "discord": "full" }
  },
}
```

接受的值：`"off"`、`"tokens"`、`"full"`，以及旧版别名 `"on"`（按 `"tokens"` 处理）。

### 三种不同的会话状态

会话的 `responseUsage` 字段有三种可表示的状态，每种状态具有
不同的语义：

| 状态               | 存储值                    | 生效模式                                                        |
| ------------------- | ------------------------------- | --------------------------------------------------------------------- |
| **未设置/继承** | `undefined`（不存在）            | 依次回退到 `messages.responseUsage` 配置默认值，然后是 `off`。 |
| **显式关闭**    | `"off"`（已存储）                | 始终关闭；非关闭的配置默认值无法重新启用页脚。     |
| **显式开启**     | `"tokens"` 或 `"full"`（已存储） | 无论配置默认值如何，均使用该模式。                              |

### 优先级

生效模式 = 会话覆盖 → 渠道配置条目 → `default` → `off`。

显式的 `/usage off` 会以字面值 `"off"` **持久化**到
会话中，它与“未设置”不同。一旦用户显式禁用页脚，非关闭的 `messages.responseUsage`
默认值无法将其重新开启。

### 重置与关闭

- `/usage off` 强制关闭页脚并持久化该选择。已配置的
  非关闭默认值无法覆盖它。
- `/usage reset`（别名：`default`、`inherit`、`inherited`、`clear`、`unpin`）会清除会话
  覆盖。随后，会话会**继承**生效的配置默认值
  （`messages.responseUsage`）。如果未配置默认值，页脚将保持关闭。
- 完整会话重置（`/reset` 或 `/new`）或会话滚动会**保留**
  显式的用量模式偏好，因此用户的显示选择可在
  会话滚动后继续保留。只有 `/usage reset`（及其别名）会清除覆盖。

### 切换行为

不带参数的 `/usage` 会按以下顺序循环：关闭 → Token → 完整 → 关闭。循环的起点
是当前的**生效**模式（未设置会话覆盖时会回退到
配置默认值），因此循环始终与用户当前在页脚中
看到的内容一致。

### 配置

未进行配置时，保留之前的行为（页脚关闭，直到使用 `/usage`）。使用
`/usage reset` 可清除会话覆盖，并重新继承已配置的默认值。

## 自定义 `/usage full` 页脚

`/usage tokens` 始终呈现一行纯 `Usage: X in / Y out`（如果可用，还会附加缓存和
估算成本后缀）。只有 `/usage full` 会呈现下述更丰富的
页脚。

`/usage full` 会显示内置的紧凑页脚；当相应字段可用时，其中包含模型、推理、快速/慢速、
上下文窗口和成本。内置页脚
不需要模板文件。

`messages.usageTemplate` 仅用于高级自定义布局。该值可以是
JSON 文件路径（支持 `~`）或内联对象；有效时会替换内置
页脚。文件路径会被监视，并在发生变化时实时重新加载。

```json
{
  "messages": {
    "usageTemplate": "~/.openclaw/usage-footer.json"
  }
}
```

缺失或空模板会静默回退到内置页脚。无法读取
或无效的已配置模板（JSON 错误，或其结构中没有可呈现的输出
片段）也会回退到内置页脚，并发出操作员警告。

以该内置结构为起点创建自定义模板，然后编辑想要
更改的部分：

```jsonc
{
  "schema": "openclaw.usageBar.v1",
  "scales": {
    "braille": "⠐⡀⡄⡆⡇⣇⣧⣷⣿",
    "block": "░▏▎▍▌▋▊▉█",
    "shade": "░▒▓█",
    "moon": "🌑🌘🌗🌖🌕",
    "level": "▁▂▃▄▅▆▇█",
    "weather": ["🥶", "☁️", "🌥", "⛅️", "🌤", "☀️"],
    "plants": ["🪾", "🍂", "🌱", "☘️", "🍀", "🌿"],
    "moons6": ["🌑", "🌚", "🌘", "🌗", "🌖", "🌝"],
  },
  "aliases": {
    "models": {
      "claude-opus-4-6": "opus46",
      "claude-opus-4-8": "opus48",
      "claude-sonnet-4-6": "sonnet46",
      "claude-haiku-4-5": "haiku45",
      "gpt-5.5": "gpt5.5",
    },
    "reasoning": {
      "off": "🌑",
      "minimal": "🌚",
      "low": "🌘",
      "medium": "🌗",
      "high": "🌕",
      "xhigh": "🌝",
    },
  },
  "output": {
    "sep": "",
    "default": [
      { "text": "{model.provider}{identity.emoji|🤖}{model.display_name|alias:models}" },
      { "map": "model.is_fallback", "cases": { "true": "🔄" } },
      { "map": "model.is_override", "cases": { "true": "📌" } },
      { "when": "model.reasoning", "text": "{model.reasoning|alias:reasoning}" },
      { "map": "state.fast_mode", "cases": { "true": "⚡️", "false": "🐌" } },
      {
        "when": "context.max_tokens",
        "text": " | 📚[{context.pct_used|meter:5:braille}]{context.max_tokens|num}",
      },
      { "when": "cost.turn_usd", "text": " 💰{cost.turn_usd|fixed:4}" },
    ],
    "surfaces": {
      "discord": [
        { "text": "-# -\n" },
        { "text": "-# {model.provider}{identity.emoji|🤖}{model.display_name|alias:models}" },
        { "map": "model.is_fallback", "cases": { "true": "🔄" } },
        { "map": "model.is_override", "cases": { "true": "📌" } },
        { "when": "model.reasoning", "text": "{model.reasoning|alias:reasoning}" },
        { "map": "state.fast_mode", "cases": { "true": "⚡️", "false": "🐌" } },
        {
          "when": "context.max_tokens",
          "text": " | 📚[{context.pct_used|meter:5:braille}]{context.max_tokens|num}",
        },
        { "when": "cost.turn_usd", "text": " 💰{cost.turn_usd|fixed:4}" },
      ],
    },
  },
}
```

### 结构

```jsonc
{
  "schema": "openclaw.usageBar.v1",
  "scales": { "<name>": "从低到高的字形" }, // 字符串（每个字符 1 个字形）或数组
  "aliases": { "<table>": { "<value>": "<label>" } },
  "output": {
    "sep": "", // 连接保留下来的片段
    "default": [/* pieces */], // 任意界面的回退
    "surfaces": {
      "discord": [/* pieces */],
      "telegram": [/* pieces */],
    },
  },
}
```

每个界面都是一个有序的**片段**列表；引擎会呈现每个片段，丢弃
空片段，并使用 `sep` 连接保留的片段。没有条目的界面使用
`output.default`。

### 契约路径

片段通过点路径读取每轮契约中的值。缺失的值视为空
（因此使用 `when` 条件或 `|fallback` 可保持片段简洁）。

| 路径                                                                                | 含义                                                                                              |
| ----------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------- |
| `surface`                                                                           | 渠道 ID（`discord`/`telegram`/等）                                                               |
| `agentId` / `chat_type`                                                             | 所属智能体 ID / 聊天界面类型                                                                  |
| `model.id` / `model.display_name` / `model.provider`                                | 模型 ID / 显示名称 / 提供商 ID                                                                |
| `model.actual`, `model.resolved_ref`                                                | 此轮实际使用的提供商/模型引用                                                        |
| `model.requested`                                                                   | 请求的提供商/模型引用（回退前）                                                       |
| `model.reasoning`                                                                   | 推理强度（`off` 到 `xhigh`）                                                                       |
| `model.is_fallback` / `model.is_override`                                           | 布尔值：是否使用回退 / 是否固定模型                                                                   |
| `model.override_source` / `model.auth_mode`                                         | 覆盖来源标签 / 凭据模式（`oauth`、`api-key`、`token`、`mixed`、`aws-sdk`、`unknown`） |
| `state.fast_mode`                                                                   | 布尔值：快速或慢速                                                                                   |
| `state.compactions`                                                                 | 会话的压缩次数                                                                     |
| `context.max_tokens` / `context.used_tokens` / `context.pct_used`                   | 窗口预算 / 已占用 token / 已使用百分比（0-100）                                                         |
| `usage.input_tokens` / `usage.output_tokens` / `usage.total_tokens`                 | 此轮汇总                                                                                       |
| `usage.cache_read_tokens` / `usage.cache_write_tokens`                              | 此轮的缓存读取和缓存写入 token                                                       |
| `usage.has_tokens` / `usage.has_split_tokens` / `usage.has_total_only_tokens`       | token 显示保护条件                                                                                 |
| `usage.cache_hit_pct`                                                               | 缓存读取量占提示词 token 总数的比例                                                              |
| `usage.last.input_tokens` / `usage.last.output_tokens` / `usage.last.cache_hit_pct` | 仅最终模型调用（另有 `cache_read_tokens`、`cache_write_tokens`、`total_tokens`）           |
| `cost.turn_usd` / `cost.available`                                                  | 此轮预估成本 / 是否解析到成本表                                                  |
| `timing.duration_ms`                                                                | 此轮的实际耗时                                                                             |
| `identity.name` / `identity.emoji` / `identity.avatar`                              | 智能体身份名称 / 表情符号 / 头像                                                                 |
| `session.id`                                                                        | 会话 ID                                                                                           |

（提供商的速率限制窗口**不**属于此契约；目前没有值为数组的路径，因此 `each` 片段没有可迭代的内容。）

### 动词

将值从左到右依次传入动词；非动词段作为回退值。

| 动词            | 效果                                | 示例                           |
| --------------- | ------------------------------------- | --------------------------------- |
| `num`           | 紧凑计数                         | `272000 -> 272k`                  |
| `fixed:N`       | N 位小数（`0..100`，默认 2）      | `0.0377`                          |
| `dur`           | 将秒数转换为时长                   | `14820 -> 4h07m`                  |
| `pct`           | 追加 `%`                            | `96 -> 96%`                       |
| `inv`           | `100 - x`                             | 用于将已用量转换为剩余量             |
| `alias:TABLE`   | 在 `aliases` 中查找，未列出时原样返回 | `medium -> 🌗`                    |
| `meter:W:SCALE` | 在 0-100 的值上绘制宽度为 W 个单元格的字形条   | `[⣿⣿⠐⠐⠐]`（`meter:1` = 一个字形） |

`fixed:N` 仅接受 0 到 100 之间的完整十进制整数。无效的
精度参数会使该插值为空。

`meter:W:SCALE` 仅接受 1 到 100 之间的完整十进制整数宽度。将宽度留空可使用默认值 5（`meter::braille`）；无效的
宽度会使该插值为空。

### 片段形式

- `{ "text": "📚 {context.max_tokens|num}" }`：字面量 + 插值。
- `{ "when": "<path>", "text": "..." }`：仅当路径值为真时渲染。
- `{ "map": "<path>", "cases": { "true": "⚡", "false": "🐌" } }`：将值映射为字形（`_default` 分支涵盖未匹配的值）。
- `{ "each": "<array-path>", "item": "{label}" }`：迭代值为数组的路径（当前契约中没有值为数组的路径）。

### 示例

```jsonc
{
  "schema": "openclaw.usageBar.v1",
  "scales": { "braille": "⠐⡀⡄⡆⡇⣇⣧⣷⣿" },
  "aliases": { "reasoning": { "medium": "🌗", "high": "🌕" } },
  "output": {
    "surfaces": {
      "discord": [
        { "text": "{model.display_name}" },
        { "when": "model.reasoning", "text": " {model.reasoning|alias:reasoning}" },
        { "map": "state.fast_mode", "cases": { "true": " ⚡", "false": " 🐌" } },
        {
          "when": "context.max_tokens",
          "text": " | 📚 [{context.pct_used|meter:5:braille}]{context.max_tokens|num}",
        },
      ],
    },
  },
}
```

例如会渲染为 `claude-sonnet-4-6 🌗 🐌 | 📚 [⣿⣿⣿⣿⣧]272k`。

## 提供商 + 凭据

无法解析出可用的提供商用量身份验证信息时，将隐藏用量。OpenClaw
会自动发现已启用且声明
`contracts.usageProviders`，并同时实现 `resolveUsageAuth` 和
`fetchUsageSnapshot` 的提供商插件；核心中没有单独的提供商允许列表。静态
契约可在不导入每个提供商插件的情况下限制发现范围。每个
插件负责自己的上游端点和响应映射。共享快照
以提供商无关的形式保存套餐名称、配额窗口、余额、支出和预算，
供 CLI、应用和 Control UI 使用。

- **Anthropic（Claude）**：身份验证配置文件中的 OAuth token。如果 OAuth token 缺少
  `user:profile` 权限范围，则在已设置时回退到 `claude.ai` Web 会话（`CLAUDE_AI_SESSION_KEY`、
  `CLAUDE_WEB_SESSION_KEY`，或 `CLAUDE_WEB_COOKIE` 中的 `sessionKey=` Cookie）。
  当 Anthropic 报告按模型划分的限制以及已启用额外用量的每月支出/预算时，
  也会包含这些信息。显式的 Anthropic Admin API key，或
  自动检测到的 `sk-ant-admin...` 提供商配置文件，则会显示 30 天的
  组织成本和 Messages API 历史记录。
- **ClawRouter**：API key（`CLAWROUTER_API_KEY`）。配置后显示每月预算窗口
  和带类型的美元预算；否则显示总支出以及
  请求/token/成本摘要。
- **DeepSeek**：通过环境变量/配置/身份验证存储提供 API key（`DEEPSEEK_API_KEY`）。
  显示提供商报告的每种货币余额。
- **GitHub Copilot**：身份验证配置文件中的 OAuth token。
- **Gemini CLI**：身份验证配置文件中的 OAuth token。
- **MiniMax**：API key 或 MiniMax OAuth 身份验证配置文件。OpenClaw 将
  `minimax`、`minimax-cn` 和 `minimax-portal` 视为同一个 MiniMax 配额
  界面，存在已存储的 MiniMax OAuth 时优先使用，否则回退到
  `MINIMAX_CODE_PLAN_KEY`、`MINIMAX_CODING_API_KEY` 或 `MINIMAX_API_KEY`。
  配置后，用量轮询从 `models.providers.minimax-portal.baseUrl`
  或 `models.providers.minimax.baseUrl` 推导 Coding Plan 主机，否则使用
  MiniMax 中国区主机。
  MiniMax 的原始 `usage_percent` / `usagePercent` 字段表示**剩余**
  配额，因此 OpenClaw 会在显示前将其反转；存在基于计数的字段时，
  优先使用这些字段。
  - 如果存在提供商的小时/分钟字段，则窗口标签来自这些字段，否则
    回退到 `start_time` / `end_time` 时间跨度。
  - 如果 Coding Plan 端点返回 `model_remains`，OpenClaw 会优先选择
    聊天模型条目；当缺少显式的 `window_hours` / `window_minutes` 字段时，
    从时间戳推导窗口标签，并在套餐标签中包含模型名称。
- **OpenAI（Codex/ChatGPT 套餐）**：身份验证配置文件中的 OAuth token（存在账户 ID 时发送
  `ChatGPT-Account-Id` 标头）。显示 ChatGPT 套餐、可重置的
  Codex 窗口，以及提供商报告的积分余额。积分仍是提供商
  积分；OpenClaw 不会将其标记为美元。`OPENAI_ADMIN_KEY` 会在 key 具有 Usage
  Dashboard 访问权限时，添加 30 天的组织成本和 completions 用量历史记录。
  推理凭据绝不会转发给组织 API。
- **OpenRouter**：API key 或由 OAuth 支持的 API key（`OPENROUTER_API_KEY` 或身份验证
  配置文件）。将账户积分端点与 key 配额端点结合，
  因而当凭据可访问相应数据时，会显示账户余额/支出、key 预算以及每日/每周/每月用量。
  任一端点都可独立丰富快照。
- **Venice**：通过环境变量/配置/身份验证存储提供 API key（`VENICE_API_KEY`）。显示美元和
  DIEM 余额，以及提供商报告的 DIEM epoch 分配用量。
- **Xiaomi MiMo**：两个独立的用量界面。按量付费使用 API key
  （`XIAOMI_API_KEY`）；Token Plan 使用单独的 key（`XIAOMI_TOKEN_PLAN_API_KEY`）。
  两者目前都不报告配额窗口。
- **z.ai**：通过环境变量/配置/身份验证存储提供 API key（`ZAI_API_KEY` 或 `Z_AI_API_KEY`）。

## 相关内容

- [Token 用量和成本](/zh-CN/reference/token-use)
- [API 用量和成本](/zh-CN/reference/api-usage-costs)
- [提示词缓存](/zh-CN/reference/prompt-caching)
- [菜单栏](/zh-CN/platforms/mac/menu-bar)
