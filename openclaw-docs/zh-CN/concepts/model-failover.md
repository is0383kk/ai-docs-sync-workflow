---
read_when:
    - 诊断身份验证配置文件轮换、冷却或模型回退行为
    - 更新身份验证配置文件或模型的故障转移规则
    - 了解会话模型覆盖如何与回退重试交互
sidebarTitle: Model failover
summary: OpenClaw 如何轮换身份验证配置文件并在模型之间进行回退
title: 模型故障转移
x-i18n:
    generated_at: "2026-07-26T06:07:40Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 3dfedbc85038eebb5be056a7b3ffa3275b4329a0b0d791e1a2b4701cbaa4b595
    source_path: concepts/model-failover.md
    workflow: 16
---

OpenClaw 分两个阶段处理故障：

1. 当前提供商内的**身份验证配置文件轮换**。
2. **模型回退**到 `agents.defaults.model.fallbacks` 中的下一个模型。

## 运行时流程

<Steps>
  <Step title="解析会话状态">
    解析当前会话模型和身份验证配置文件偏好。
  </Step>
  <Step title="构建候选链">
    根据当前模型选择及该选择来源的回退策略构建模型候选链。已配置的默认模型、定时任务主模型和自动选择的回退模型可以使用已配置的回退；明确的用户会话选择则严格执行。
  </Step>
  <Step title="尝试当前提供商">
    按照身份验证配置文件轮换/冷却规则尝试当前提供商。
  </Step>
  <Step title="遇到应故障转移的错误时继续">
    如果该提供商因应故障转移的错误而耗尽可用选项，则转到下一个模型候选项。
  </Step>
  <Step title="当前轮次使用回退">
    运行成功的回退候选项，但不更改会话所选的提供商/模型。
  </Step>
  <Step title="安全重试纯过载耗尽">
    如果每个候选项都仅因提供商过载而失败，并且尚未开始执行工具或输出助手内容，则采用指数退避，最多重试整个轮次本地候选链 10 次。30 秒后发送一次状态通知，避免用户一直无提示地等待。
  </Step>
  <Step title="耗尽时抛出 FallbackSummaryError">
    如果所有候选项均失败，则抛出包含每次尝试详情的 `FallbackSummaryError`；如果已知，还会包含最早的冷却到期时间。
  </Step>
</Steps>

回退执行仅限当前轮次。回复运行器仅持久化回退通知状态，以便 `/status` 和转换通知区分所选模型与实际作答的模型；它不会将回退持久化为下一轮的模型选择。

## 选择来源策略

选择来源决定是否允许使用回退链：

- **已配置的默认模型**：`agents.defaults.model.primary` 使用 `agents.defaults.model.fallbacks`。
- **智能体主模型**：`agents.entries.*.model` 严格执行，除非该智能体的模型对象包含其自己的 `fallbacks`。使用 `fallbacks: []` 可明确指定严格行为，或使用非空列表为该智能体启用模型回退。
- **运行时回退**：回退候选项仅适用于当前轮次。下一轮会再次从所选主模型开始。OpenClaw 仍会识别之前存储的 `modelOverrideSource: "auto"` 条目，每 5 分钟探测其配置的来源，并在来源恢复后清除这些条目。`/new`、`/reset` 和 `sessions.reset` 也会清除这些条目。
- **用户会话覆盖**：`/model`、模型选择器、`session_status(model=...)` 和 `sessions.patch` 会写入 `modelOverrideSource: "user"`。这是精确的会话选择。如果所选提供商/模型在生成回复前失败，OpenClaw 会报告该故障，而不会使用无关的已配置回退模型作答。
- **旧版会话覆盖**：较旧的会话条目可能包含 `modelOverride`，但不包含 `modelOverrideSource`。OpenClaw 将其视为用户覆盖，避免明确的旧选择被静默转换为回退行为。
- **定时任务负载模型**：定时任务的 `payload.model` / `--model` 是任务主模型，而不是用户会话覆盖。除非任务提供 `payload.fallbacks`，否则它会使用已配置的回退；`payload.fallbacks: []` 会使该定时任务运行严格执行。

当某轮切换到回退模型时，OpenClaw 会发送可见通知；当后续轮次在所选主模型上成功时，还会发送另一条通知。持久化的通知状态可防止连续轮次使用相同的所选/活动模型组合时重复发送通知，同时模型选择本身保持不变。

## 身份验证失败跳过缓存

默认情况下，每个新轮次都会保留现有的回退重试行为：OpenClaw 会再次尝试每个已配置的回退候选项，包括最近因 `auth` 或 `auth_permanent` 而失败的非主候选项。

使用以下设置选择启用重复身份验证失败抑制：

```bash
OPENCLAW_FALLBACK_SKIP_TTL_MS=60000
```

启用后，如果非主回退候选项发生身份验证类故障，OpenClaw 会为其记录一个内存中、会话范围的跳过标记，并以会话 ID、提供商和模型作为键。主候选项永远不会被跳过，因此明确的用户模型选择仍会显示真实的身份验证错误。该缓存仅存在于当前进程中，并会在 Gateway 网关重启时清除。

该值是以毫秒为单位的 TTL。`0` 或未设置会禁用缓存。正值会限制在 1 秒至 10 分钟之间。

## 用户可见的回退通知

当会话切换到自动选择的回退模型时，OpenClaw 会在同一回复界面中发送状态通知：

```text
↪️ 模型回退：<fallback>（已选择 <primary>；<reason>）
```

当后续探测成功且会话返回所选主模型时，OpenClaw 会发送：

```text
↪️ 模型回退已清除：<primary>（之前为 <fallback>）
```

这些通知是操作消息，而非助手内容。它们会在每次状态变更时发送一次，在可行的情况下也包括仅产生副作用的轮次，但重复的轮次本地回退转换不会重复发送。通知传递会绕过常规的来源回复抑制，不占用线程式渠道中的第一个助手回复槽位，并且不会参与文本转语音和跟进承诺提取。

## 身份验证存储（密钥 + OAuth）

OpenClaw 对 API 密钥和 OAuth 令牌都使用**身份验证配置文件**。

- 机密信息和运行时身份验证路由状态存储在 `~/.openclaw/agents/<agentId>/agent/openclaw-agent.sqlite` 中。
- 配置 `auth.profiles` / `auth.order` **仅包含元数据和路由信息**（不含机密信息）。
- 仅用于旧版导入的 OAuth 文件：`~/.openclaw/credentials/oauth.json`（首次使用时导入每个智能体的身份验证存储）。
- 旧版 `auth-profiles.json`、`auth-state.json` 和每个智能体的 `auth.json` 文件由 `openclaw doctor --fix` 导入。

更多详情：[OAuth](/zh-CN/concepts/oauth)

凭据类型：

- `type: "api_key"` → `{ provider, key }`
- `type: "oauth"` → `{ provider, access, refresh, expires, email? }`（某些提供商还使用 `projectId`/`enterpriseUrl`）
- `type: "token"` → 静态 Bearer 式令牌，可以选择设置过期时间；OpenClaw 不会刷新它（用于 `aws-sdk` 和其他凭据链身份验证模式）

## 配置文件 ID

OAuth 登录会创建不同的配置文件，使多个账号能够共存。

- 默认值：没有可用电子邮件地址时使用 `provider:default`。
- 包含电子邮件地址的 OAuth：`provider:<email>`（例如 `google-antigravity:user@gmail.com`）。

配置文件存储在每个智能体的 `openclaw-agent.sqlite` 身份验证配置文件存储中。

## 轮换顺序

当一个提供商有多个配置文件时，OpenClaw 按以下顺序选择：

<Steps>
  <Step title="显式配置">
    `auth.order[provider]`（如果已设置）。
  </Step>
  <Step title="已配置的配置文件">
    按提供商筛选的 `auth.profiles`。
  </Step>
  <Step title="已存储的配置文件">
    该提供商在每个智能体的 SQLite 中存储的身份验证配置文件条目。
  </Step>
</Steps>

如果未配置显式顺序，OpenClaw 会使用轮询顺序：

- **主键：**配置文件类型（**依次为 OAuth、静态令牌、API 密钥**）。
- **OAuth 的次级键：**当前访问令牌可用的配置文件优先于
  访问令牌已过期的配置文件。已过期的 OAuth 配置文件仍具有候选资格，以便
  在没有可用的同级配置文件时由运行时刷新。
- **下一个键：**`usageStats.lastUsed`（在每种类型/状态层级内，最早的优先）。
- **处于冷却期/已禁用的配置文件**会移到末尾，并按最早到期时间排序。

### 会话粘性（有利于缓存）

OpenClaw 会**为每个会话固定选定的身份验证配置文件**，以保持提供商缓存处于热状态。它**不会**在每次请求时轮换。固定的配置文件会重复使用，直到：

- 会话被重置（`/new` / `/reset`）
- 压缩完成（压缩计数递增）
- 配置文件处于冷却期/已禁用

通过 `/model …@<profileId>` 手动选择会为该会话设置**用户覆盖**，并且在新会话开始前不会自动轮换。

<Note>
自动固定的配置文件（由会话路由器选择）会被视为一种**偏好**：优先尝试它们，但遇到速率限制/超时时，OpenClaw 可能轮换到其他配置文件。当原始配置文件再次可用时，新的运行可以再次优先选择它，而无需更改所选模型或运行时。用户固定的配置文件会继续锁定到该配置文件；如果它失败且已配置模型回退，OpenClaw 会转到下一个模型，而不是切换配置文件。
</Note>

### OpenAI Codex 订阅及 API 密钥备用方案

对于 OpenAI 智能体模型，身份验证与运行时彼此独立。`openai/gpt-*` 保持使用 Codex harness，而身份验证可以在 Codex 订阅配置文件和 OpenAI API 密钥备用配置文件之间轮换。

使用 `auth.order.openai` 设置面向用户的顺序：

```json5
{
  auth: {
    order: {
      openai: ["openai:user@example.com", "openai:api-key-backup"],
    },
  },
}
```

ChatGPT/Codex OAuth 配置文件和 OpenAI API 密钥配置文件都使用 `openai:*`。当订阅达到 Codex 使用限制时，如果 Codex 提供了准确的重置时间，OpenClaw 会记录该时间、尝试下一个按顺序排列的身份验证配置文件，并让运行继续留在 Codex harness 内。重置时间过后，订阅配置文件会再次具备候选资格，下一次自动选择可以返回该配置文件。

仅当你希望在该会话中强制使用某个账号/密钥时，才使用用户固定的配置文件。用户固定的配置文件有意采用严格行为，不会静默跳转到其他配置文件。

## 冷却期

当配置文件因身份验证/速率限制错误（或类似速率限制的超时）而失败时，OpenClaw 会将其标记为处于冷却期，并转到下一个配置文件。

<AccordionGroup>
  <Accordion title="哪些情况会归入速率限制/超时类别">
    该速率限制类别比单纯的 `429` 更广：它还包括 `Too many concurrent requests`、`ThrottlingException`、`concurrency limit reached`、`workers_ai ... quota limit exceeded`、`throttled`、`resource exhausted` 等提供商消息，以及 `weekly limit reached` 或 `monthly limit exhausted` 等周期性用量窗口限制。

    格式/无效请求错误通常是终止性错误，因为使用相同负载重试仍会以相同方式失败，所以 OpenClaw 会直接显示这些错误，而不是轮换身份验证配置文件。已知的重试修复路径可以明确选择启用：例如，Cloud Code Assist 工具调用 ID 验证失败会被清理，并通过 `allowFormatRetry` 策略重试一次。

    OpenAI 兼容的**提供商已完成**停止/结束原因（例如 `Unhandled stop reason: error`、`stop reason: error`、`reason: error` 和 `Provider finish_reason: error`）会被分类为 **`server_error`**（类似 HTTP 的状态码 500），而不是超时。它们仍符合模型/配置文件轮换的故障转移条件，但诊断信息会保留提供商的结束原因文本，而不会将面向用户的文字改写为“LLM 请求超时”。`Provider finish_reason: abort`、`network_error` 和 `malformed_response` 等传输类结束原因仍归入超时/故障转移类别（状态码 408）。

    当来源符合已知的瞬态模式时，通用服务器文本也可能归入该超时类别。例如，单独出现的模型运行时流封装器消息 `An unknown error occurred` 对所有提供商都被视为应故障转移，因为共享模型运行时会在提供商流以 `stopReason: "aborted"` 或 `stopReason: "error"` 结束且没有具体详情时发出该消息。包含 `internal server error`、`unknown error, 520`、`upstream error` 或 `backend error` 等瞬态服务器文本的 JSON `api_error` 负载也会被视为应故障转移的超时。

    OpenRouter 特有的通用上游文本（例如单独出现的 `Provider returned error`）仅在提供商上下文确实为 OpenRouter 时才被视为超时。通用内部回退文本（例如 `LLM request failed with an unknown error.`）仍采用保守处理，其本身不会触发故障转移。

  </Accordion>
  <Accordion title="SDK retry-after 上限">
    否则，某些提供商 SDK 可能会在一个较长的 `Retry-After` 时间窗口内休眠，然后才将控制权返回给 OpenClaw。对于 Anthropic 和 OpenAI 等基于 Stainless 的 SDK，OpenClaw 默认将 SDK 内部的 `retry-after-ms` / `retry-after` 等待时间限制为 60 秒，并立即暴露需要等待更长时间但可重试的响应，以便运行此故障转移路径。可使用 `OPENCLAW_SDK_RETRY_MAX_WAIT_SECONDS` 调整或禁用该上限；请参阅[重试行为](/zh-CN/concepts/retry)。
  </Accordion>
  <Accordion title="模型级冷却">
    速率限制冷却也可以限定到模型：

    - 当已知失败模型的 ID 时，OpenClaw 会为速率限制失败记录 `cooldownModel`。
    - 如果冷却限定到其他模型，仍可尝试同一提供商的同级模型。
    - 计费/禁用时间窗口仍会跨模型阻止整个配置文件。

  </Accordion>
</AccordionGroup>

常规（非计费、非永久身份验证）冷却会根据配置文件近期的错误次数递增：

- 第 1 次失败：30 秒
- 第 2 次失败：1 分钟
- 第 3 次及以后失败：5 分钟（上限）

配置文件内置的失败时间窗口结束后，计数器会重置。

状态存储在每个 Agent 的 SQLite 身份验证状态中的 `usageStats` 下：

```json
{
  "usageStats": {
    "provider:profile": {
      "lastUsed": 1736160000000,
      "cooldownUntil": 1736160600000,
      "errorCount": 2
    }
  }
}
```

## 因计费问题禁用

计费/额度失败（例如“额度不足”/“额度余额过低”）会被视为应触发故障转移，但通常不是暂时性问题。OpenClaw 不会采用短暂冷却，而是将配置文件标记为**已禁用**（具有更长的退避时间），并轮换到下一个配置文件/提供商。

<Note>
并非每个具有计费特征的响应都是 `402`，也并非每个 HTTP `402` 都归入此处。即使提供商返回的是 `401` 或 `403`，OpenClaw 仍会将明确的计费文本归入计费路径，但提供商特有的匹配器仍仅限于拥有它们的提供商（例如 OpenRouter `403 Key limit exceeded`）。

同时，当消息看起来可重试时（例如 `weekly usage limit exhausted`、`daily limit reached, resets tomorrow` 或 `organization spending limit exceeded`），暂时性的 `402` 使用时间窗口以及组织/工作区支出限制错误会被归类为 `rate_limit`。这些错误会继续使用短暂冷却/故障转移路径，而不是长期的计费禁用路径。
</Note>

高置信度的永久身份验证失败（密钥已撤销/停用、工作区已停用）会进入类似的禁用路径，但其恢复时间比计费问题短得多，因为某些提供商在事故期间可能会暂时返回看似身份验证失败的负载。

状态存储在每个 Agent 的 SQLite 身份验证状态中：

```json
{
  "usageStats": {
    "provider:profile": {
      "disabledUntil": 1736178000000,
      "disabledReason": "billing"
    }
  }
}
```

过载和速率限制错误的处理比计费冷却更激进：默认情况下，OpenClaw 允许在同一提供商内重试一个身份验证配置文件，然后无需等待便切换到下一个已配置的后备模型。

## 模型后备

如果某个提供商的所有配置文件均失败，OpenClaw 会转到 `agents.defaults.model.fallbacks` 中的下一个模型。这适用于已耗尽配置文件轮换的身份验证失败、速率限制和超时（其他错误不会推进后备流程）。对于未暴露足够详细信息的提供商错误，后备状态仍会使用精确标签：`empty_response` 表示提供商未返回可用的消息或状态；`no_error_details` 表示提供商明确返回了 `Unknown error (no error details in response)`；`unclassified` 表示 OpenClaw 保留了原始预览，但尚无分类器与之匹配。

`ModelNotReadyException` 等提供商繁忙信号会归入过载类别，并遵循与速率限制相同的“轮换一次后进入后备”策略（请参阅上方的默认值表）。

如果整个候选链仅因过载失败而耗尽，回复运行器会在同一轮次中重试该候选链，最多 10 次。仅在工具执行或助手输出开始之前才允许重试整个轮次，以避免在已完成可观察操作后发生过载时造成重复修改或消息。退避从 2.5 秒开始，并按倍数增长，最高为 30 秒。当该轮次已等待 30 秒后，OpenClaw 会发送一次临时状态通知：`The AI service is temporarily overloaded. I’m still retrying; this may take a few minutes.`。重试以及最终胜出的任何后备候选都仅限当前轮次；普通的暂时性服务器错误仍采用其单独的一次重试策略。

当运行从已配置的默认主模型、定时任务主模型、具有显式后备模型的 Agent 主模型，或自动选择的后备覆盖项开始时，OpenClaw 可以遍历匹配的已配置后备链。没有显式后备模型的 Agent 主模型和显式用户选择（例如 `/model ollama/qwen3.5:27b`、模型选择器、`sessions.patch` 或一次性 CLI 提供商/模型覆盖项）采用严格策略：如果该提供商/模型不可访问或在生成回复前失败，OpenClaw 会报告失败，而不是使用无关的后备模型作答。

### 候选链规则

OpenClaw 根据当前请求的 `provider/model` 和已配置的后备模型构建候选列表。

<AccordionGroup>
  <Accordion title="规则">
    - 请求的模型始终排在首位。
    - 显式配置的后备模型会去重，但不会按模型允许列表进行筛选。它们被视为操作员的明确意图。
    - 如果当前运行已使用同一提供商系列中的已配置后备模型，OpenClaw 会继续使用完整的已配置候选链。
    - 如果未提供显式后备覆盖项，即使请求的模型使用其他提供商，也会先尝试已配置的后备模型，然后再尝试已配置的主模型。
    - 如果未向后备运行器提供显式后备覆盖项，则会将已配置的主模型追加到末尾，以便在更早的候选模型耗尽后，候选链可以重新落回正常的默认模型。
    - 当调用方提供 `fallbacksOverride` 时，运行器将仅使用请求的模型和该覆盖列表。空列表会禁用模型后备，并阻止将已配置的主模型作为隐藏的重试目标追加进来。

  </Accordion>
</AccordionGroup>

### 哪些错误会推进后备流程

<Tabs>
  <Tab title="以下情况继续">
    - 身份验证失败
    - 速率限制和冷却耗尽
    - 过载/提供商繁忙错误
    - 具有超时特征的故障转移错误
    - 因计费问题禁用
    - `LiveSessionModelSwitchError`，它会被规范化为故障转移路径，以免过时的持久化模型造成外层重试循环
    - 仍有剩余候选项时出现的其他无法识别的错误

  </Tab>
  <Tab title="以下情况不继续">
    - 不具有超时/故障转移特征的显式中止
    - 应留在压缩/重试逻辑内处理的上下文溢出错误（例如 `request_too_large`、`input token count exceeds the maximum number of input tokens`、`input exceeds the maximum number of tokens`、`input too long for the model` 或 `ollama error: context length exceeded`）
    - 没有剩余候选项时的最终未知错误
    - Claude Fable 5 安全拒绝；直接使用 API 密钥的请求会改由提供商层处理，通过 Anthropic 的服务端后备切换到 `claude-opus-4-8`（请参阅 [Anthropic](/zh-CN/providers/anthropic#safety-refusal-fallback-claude-fable-5)）

  </Tab>
</Tabs>

### 跳过冷却与探测行为

当某个提供商的每个身份验证配置文件都已进入冷却状态时，OpenClaw 不会自动永远跳过该提供商，而是针对每个候选项进行判断：

<AccordionGroup>
  <Accordion title="针对每个候选项的判断">
    - 永久身份验证失败会立即跳过整个提供商。
    - 因计费问题禁用时通常会跳过，但仍可以按节流策略探测主候选项，以便无需重启即可恢复。
    - 在接近冷却到期时，可以按照每个提供商的节流策略探测主候选项。
    - 如果失败看起来是暂时性的（`rate_limit`、`overloaded` 或未知），即使处于冷却状态，也可以尝试同一提供商的同级后备模型。当速率限制限定到模型，并且同级模型可能立即恢复时，这一点尤其重要。
    - 每次后备运行中，每个提供商最多进行一次暂时性冷却探测，以免单个提供商阻碍跨提供商后备流程。

  </Accordion>
</AccordionGroup>

## 会话覆盖项和实时模型切换

会话模型变更属于共享状态。活动运行器、`/model` 命令、压缩/会话更新以及实时会话协调，都会读取或写入同一会话条目的不同部分。执行后备流程不会写入模型选择字段，因此它在重试时无法替换较新的手动选择。

实时模型切换遵循以下规则：

- 只有由用户明确发起的模型变更才会标记待处理的实时切换，包括 `/model`、`session_status(model=...)` 和 `sessions.patch`。
- 后备轮换、Heartbeat 覆盖或压缩等由系统驱动的模型变更绝不会自行标记待处理的实时切换。
- 对于后备策略，用户驱动的模型覆盖项会被视为精确选择，因此无法访问所选提供商时会暴露为失败，而不会被 `agents.defaults.model.fallbacks` 掩盖。
- 运行时后备候选项仅限当前轮次。下一轮次会从当前选择的模型开始，包括在上一轮运行期间到达的手动选择。
- 继续支持先前存储的自动后备覆盖项：OpenClaw 会定期探测其已配置的来源，并在恢复时清除覆盖项；`/new`、`/reset` 和 `sessions.reset` 会立即清除自动来源的覆盖项。
- 每次状态变更时，用户回复会各通知一次后备转换以及后备清除后的恢复。选择模型/活动模型组合相同的连续轮次不会重复通知。
- `/status` 会显示选择的模型；当后备状态不同时，还会显示当前活动的后备模型及其原因。
- 实时会话协调会优先采用持久化的会话覆盖项，而不是过时的运行时模型字段。
- 如果实时切换错误指向活动后备链中较后的候选项，OpenClaw 会直接跳转到该选择的模型，而不是先遍历无关的候选项。

活动运行会直接携带其选择的候选项。仅当存在用户明确发起的待处理切换时，实时协调才会更改该候选项，因此不需要临时后备覆盖项或回滚。

## 可观测性和失败摘要

`runWithModelFallback(...)` 会记录每次尝试的详细信息，用于日志和面向用户的冷却消息：

- 尝试的提供商/模型
- 原因（`rate_limit`、`overloaded`、`billing`、`auth`、`model_not_found` 及类似的故障转移原因）
- 可选的状态/代码
- 易于理解的错误摘要

当候选项失败、被跳过或后续后备项成功时，结构化的 `model_fallback_decision` 日志还会包含扁平化的 `fallbackStep*` 字段。这些字段会明确记录尝试的转换（`fallbackStepFromModel`、`fallbackStepToModel`、`fallbackStepFromFailureReason`、`fallbackStepFromFailureDetail`、`fallbackStepFinalOutcome`），以便日志和诊断导出器能够重建主候选项的失败情况，即使最终的后备项也失败。

当所有候选项都失败时，OpenClaw 会抛出 `FallbackSummaryError`。外层回复运行器可以据此构建更具体的消息，例如“所有模型暂时均受到速率限制”，并在已知时包含最早的冷却到期时间。

该冷却摘要会区分模型：

- 与尝试的提供商/模型链无关的模型级速率限制将被忽略
- 如果剩余阻止项是匹配的模型级速率限制，OpenClaw 会报告仍在阻止该模型的最后一个匹配到期时间

## 相关配置

有关以下内容，请参阅 [Gateway 配置](/zh-CN/gateway/configuration)：

- `auth.profiles` / `auth.order`
- `agents.defaults.model.primary` / `agents.defaults.model.fallbacks`
- `agents.defaults.imageModel` 路由

有关更全面的模型选择和回退概览，请参阅[模型](/zh-CN/concepts/models)。
