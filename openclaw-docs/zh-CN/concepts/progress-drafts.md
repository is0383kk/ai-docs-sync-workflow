---
read_when:
    - 为长时间运行的聊天轮次配置可见的进度更新
    - 在部分、分块和进度流式传输模式之间进行选择
    - 说明 OpenClaw 如何在工作进行期间更新一条渠道消息
    - 进度草稿、独立进度消息或最终回复回退的故障排查
summary: 进度草稿：一条可见的进行中消息，会在智能体运行时持续更新
title: 进度草稿
x-i18n:
    generated_at: "2026-07-26T06:13:59Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 4ef66dd4d7a31c753f5faa0b88b83ec3760beecf3118cf8aae84f5e57652e809
    source_path: concepts/progress-drafts.md
    workflow: 16
---

进度草稿会将一条渠道消息变成实时状态行，在智能体工作期间持续更新，而不是堆积一系列临时的“仍在处理”回复。设置
`channels.<channel>.streaming.mode: "progress"` 后，OpenClaw 会在实际工作开始时创建消息，并在智能体读取、规划、调用工具或等待审批时编辑该消息，最后将其转换为最终答案。

```text
处理中...
📖 来自 docs/concepts/progress-drafts.md
🔎 Web Search：搜索 "discord edit message"
🛠️ Bash：运行测试
```

<Note>
  当未设置 `channels.discord.streaming` 时，Discord 已默认使用 `streaming.mode: "progress"`，因此无需任何配置即可在其中显示进度草稿。其他所有渠道均默认为 `partial`
  或 `off`；有关各渠道默认值的完整表格，请参阅[流式传输和分块](/zh-CN/concepts/streaming#channel-mapping)。
</Note>

## 快速开始

```json5
{
  channels: {
    discord: {
      streaming: {
        mode: "progress",
      },
    },
  },
}
```

此处的默认设置为：启动延迟 5 秒，在进行有用工作时显示紧凑的进度行，并抑制该轮次中旧版的独立进度消息。原始工具行草稿使用自动生成的单词标签；状态标题会省略这个冗余标题，除非你明确配置一个标题。

本页介绍进度草稿体验及其配置选项。有关完整的流式传输模式矩阵、各渠道运行时说明和旧版键迁移，请参阅[流式传输和分块](/zh-CN/concepts/streaming)。

## 用户看到的内容

| 部分            | 用途                                                                           |
| --------------- | --------------------------------------------------------------------------------- |
| 状态标题 | 在 Discord 和 Telegram 上显示模型前言；Discord 还会添加实用的补充说明。       |
| 标签           | 可选的起始/状态行，例如 `Working`。                                   |
| 进度行  | 使用与 `/verbose` 相同的工具图标和详情格式化程序显示紧凑的运行更新。 |

对于原始工具进度，当智能体开始有意义的工作并在初始延迟期间持续忙碌时，标签便会出现。
它位于滚动进度行列表的顶部，因此当出现足够多的具体工作行后，它会滚出显示区域。除非明确配置标签，否则状态标题只显示智能体的自然语言状态。纯文本回复绝不会显示进度草稿；只有实际工作更新才会显示进度行，例如 `🛠️ Bash: run tests`、`🔎 Web Search: for "discord edit message"`
或 `✍️ Write: to /tmp/file`。

当渠道能够安全地执行此操作时，最终答案会原位替换草稿；否则 OpenClaw 会通过正常投递发送最终答案，并清理或停止更新草稿（请参阅[最终处理](#finalization)）。

## 选择模式

`channels.<channel>.streaming.mode` 控制可见的进行中行为：

| 模式       | 最适合                         | 聊天中显示的内容                              |
| ---------- | -------------------------------- | ------------------------------------------------- |
| `off`      | 安静的渠道                   | 仅显示最终答案。                            |
| `partial`  | 观察答案文本逐步出现      | 编辑一条草稿以显示最新答案文本。     |
| `block`    | 较大的答案预览分块     | 以较大分块更新或追加一条预览。 |
| `progress` | 大量使用工具或长时间运行的轮次 | 一条状态草稿，然后显示最终答案。          |

如果用户更关心“正在发生什么”，而不是逐 token 观察答案文本流式显示，请选择 `progress`；如果答案文本本身就是进度信号，请选择 `partial`；如需较大的预览分块，请选择 `block`。在 Discord 和
Telegram 上，`streaming.mode: "block"` 仍是预览流式传输，而不是普通的分块回复投递——后者请使用 `streaming.block.enabled`。

## 配置标签

进度标签位于 `channels.<channel>.streaming.progress` 下。默认的原始工具行标签是 `"auto"`，它使用内置的普通 `Working`
标签。状态标题会隐藏这个隐式标签；如果你还希望在其上方显示标签，请明确设置
`label: "auto"`：

```text
处理中
```

使用固定标签：

```json5
{
  channels: {
    discord: {
      streaming: {
        mode: "progress",
        progress: {
          label: "正在调查",
        },
      },
    },
  },
}
```

使用你自己的标签池（当值为 `label: "auto"` 时，仍按随机方式/种子选取）：

```json5
{
  channels: {
    discord: {
      streaming: {
        mode: "progress",
        progress: {
          label: "auto",
          labels: ["正在检查", "正在读取", "正在测试", "即将完成"],
        },
      },
    },
  },
}
```

隐藏标签，仅显示进度行：

```json5
{
  channels: {
    discord: {
      streaming: {
        mode: "progress",
        progress: {
          label: false,
        },
      },
    },
  },
}
```

## 控制进度行

进度行来自实际运行事件：工具启动、项目更新、任务计划、审批、命令输出、补丁摘要以及类似的智能体活动。
它们默认启用（`progress.toolProgress`，默认值为 `true`）。

工具还可以在单次调用仍在运行时发出类型化进度。这样，缓慢的获取或搜索操作就能在工具返回最终结果之前更新可见草稿。进度更新是一个部分工具结果，模型内容为空，并包含明确的公共渠道元数据：

```json
{
  "content": [],
  "progress": {
    "text": "正在获取页面内容...",
    "visibility": "channel",
    "privacy": "public",
    "id": "web_fetch:fetching"
  }
}
```

OpenClaw 仅在渠道进度 UI 中渲染 `progress.text`。普通工具结果稍后仍会以 `content`/`details` 的形式到达，并且只有这部分会返回给模型。

为工具添加进度时，应发出简短、通用的消息，并等到操作挂起足够长时间、显示进度确有帮助时再发送。`web_fetch`
通过 5 秒延迟准确实现了这一点：

```typescript
const clearProgressTimer = scheduleToolProgress(
  onUpdate,
  { text: "正在获取页面内容...", id: "web_fetch:fetching" },
  5_000,
  { signal },
);

try {
  return await runToolWork();
} finally {
  clearProgressTimer();
}
```

快速调用不会显示进度行；长时间调用会在仍处于挂起状态时显示一行；已取消的调用会在过时进度出现之前清除计时器。进度文本是公共 UI 辅助渠道，因此绝不能包含密钥、原始参数、获取的内容、命令输出或页面文本。

### 详情模式

OpenClaw 对进度草稿和 `/verbose` 使用相同的格式化程序：

```json5
{
  agents: {
    defaults: {
      toolProgressDetail: "explain", // explain | raw
    },
  },
}
```

`"explain"` 是默认值，它通过简洁标签使草稿保持稳定。
`"raw"` 会在可用时追加底层命令，这在调试时很有用，但会使聊天内容更嘈杂。例如，`node --check /tmp/app.js` 调用在不同模式下的渲染方式如下：

| 模式      | 进度行                                                   |
| --------- | --------------------------------------------------------------- |
| `explain` | `🛠️ check js syntax for /tmp/app.js`                            |
| `raw`     | `🛠️ check js syntax for /tmp/app.js · node --check /tmp/app.js` |

### 命令/exec 文本

`streaming.progress.commandText`（默认值为 `"raw"`）控制 exec/bash 进度行旁显示多少命令详情，与上述详情模式无关。将其设置为 `"status"`，可在隐藏全部命令文本的同时保留可见的工具进度行：

```json5
{
  channels: {
    discord: {
      streaming: {
        mode: "progress",
        progress: {
          commandText: "status",
        },
      },
    },
  },
}
```

### 解说通道

`streaming.progress.commentary`（默认值为 `false`）会将模型在工具调用前的解说/前言叙述（💬，例如“我会先检查……然后……”）与工具行交错显示在草稿中。有关各渠道共用的配置结构，请参阅[流式传输和分块](/zh-CN/concepts/streaming#commentary-progress-lane)。

启用解说通道后，前言仅渲染为这些交错显示的 💬 行；下方的状态标题不会显示，以便该通道保持其文档所述的结构。

### 状态标题

在 Discord 和 Telegram 的进度模式下，只要模型发出了类型化的工具调用前前言，它就会成为草稿的状态标题。其他使用进度模式的渠道则保持其现有状态行为。标题默认启用，且不会绕过短轮次的常规活动门槛；启用 `streaming.progress.commentary` 后，前言会改由交错显示的解说通道处理。

在 Discord 上，当智能体能够解析到实用模型时——即明确设置的
[`utilityModel`](/zh-CN/gateway/config-agents#utilitymodel)，或主提供商声明的小模型默认值（OpenAI → `gpt-5.6-luna`，
Anthropic → `claude-haiku-4-5`）——如果模型没有发出前言，或已沉默约 20 秒，该模型会提供简短的自然语言补充说明
（目前 Telegram 的标题仅使用前言）：

```text
正在更新配置中的默认模型，然后重启 Gateway 网关以应用该配置。
一次智能体列表调用失败，正在重试。
```

实用叙述默认启用（`streaming.progress.narration`，默认值为
`true`），并且绝不会回退到主模型：只有为智能体的主提供商明确配置了 `utilityModel` 或存在提供商声明的默认值时，它才会运行。
设置 `utilityModel: ""` 可完全禁用实用路由。工具行会继续在下方累积，并在两个状态来源均停止后重新显示。草稿编辑仍会等待常规活动门槛和实际文本变化，从而避免快速轮次中出现闪烁，并减少繁忙渠道中的编辑抖动。设置 `narration: false` 可仅禁用实用模型补充说明；模型前言标题仍保持启用：

```json5
{
  channels: {
    discord: {
      streaming: {
        mode: "progress",
        progress: {
          narration: false,
        },
      },
    },
  },
}
```

叙述输入受到限制并经过脱敏：实用模型接收传入请求文本，以及草稿会渲染的同一组紧凑、脱敏的工具摘要——绝不会接收原始命令输出或工具结果。使用
`commandText: "status"` 时，叙述输入还会省略 exec/bash 命令文本，与草稿显示的内容一致。

### 行数限制

限制保持可见的行数（默认值为 8）：

```json5
{
  channels: {
    discord: {
      streaming: {
        mode: "progress",
        progress: {
          maxLines: 4,
        },
      },
    },
  },
}
```

进度行会自动压缩，以减少草稿编辑时聊天气泡的重新排版；OpenClaw 还会截断过长的行，以免反复编辑草稿时，每次更新都产生不同的换行。默认的单行预算为 120 个字符；普通文本会在单词边界处截断，而路径或原始命令等较长详情会使用中间省略号缩短，以便保留可见的后缀。

调整单行预算：

```json5
{
  channels: {
    discord: {
      streaming: {
        mode: "progress",
        progress: {
          maxLineChars: 160,
        },
      },
    },
  },
}
```

### 富文本渲染（Slack）

Slack 可以将进度行渲染为结构化 Block Kit 字段，而不是纯文本：

```json5
{
  channels: {
    slack: {
      streaming: {
        mode: "progress",
        progress: {
          render: "rich",
        },
      },
    },
  },
}
```

富文本渲染始终会在 Block Kit 字段旁发送相同的纯文本正文，因此无法渲染这种更丰富结构的客户端仍会显示紧凑的进度文本。

### 隐藏工具/任务行

保留单条进度草稿，但隐藏工具行和任务行：

```json5
{
  channels: {
    discord: {
      streaming: {
        mode: "progress",
        progress: {
          toolProgress: false,
        },
      },
    },
  },
}
```

使用 `toolProgress: false` 时，OpenClaw 仍会抑制该轮次中较旧的独立
工具进度消息——渠道在视觉上保持安静，直到最终答案出现；
如果配置了标签，则标签除外。

## 渠道行为

| 渠道            | 进度传输方式                           | 说明                                                                                                                                                      |
| --------------- | -------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Discord         | 发送一条消息，然后编辑它。             | 默认使用 `progress` 模式；最终答案会附带 `-#` 活动回执，并在答案送达后删除状态草稿。                                               |
| Matrix          | 发送一个事件，然后编辑它。             | 账户级流式传输配置控制账户级草稿。                                                                                                                        |
| Microsoft Teams | 在个人聊天中使用 Teams 原生流。        | `streaming.mode: "block"` 会改为映射到 Teams 分块投递。                                                                                                          |
| Slack           | 原生流或可编辑的草稿帖子。             | 需要回复线程目标；没有该目标的顶层私信仍会获得草稿预览帖子及其编辑。                                                                                      |
| Telegram        | 发送一条消息，然后编辑它。             | 如果有消息在进度草稿和答案之间送达，草稿会重新发布到该消息下方（先发布新草稿，再删除旧草稿），而不是让客户端滚动位置发生跳跃。                            |
| Mattermost      | 可编辑的草稿帖子。                     | `block` 模式会在已完成文本帖子和工具活动帖子之间轮换；其他模式会将工具活动合并到同一个草稿式帖子中。                                            |

不支持安全编辑的渠道会回退到输入状态指示器或仅投递
最终答案。有关每个渠道完整的运行时行为细分，请参阅[流式传输和分块](/zh-CN/concepts/streaming)。

## 最终处理

最终答案准备就绪后，OpenClaw 会尽量保持聊天整洁：

- 在 Discord 的 `progress` 模式下，最终答案会作为新消息发送，
  并附带一条简短的 `-#` 活动回执（例如
  `-# 🧠 2 thoughts · 🛠️ 5 tool calls · ⏱️ 12s`）；答案投递后，
  状态草稿会被删除。即使渠道消息繁忙，回复上方也不会留下孤立的工具
  日志；如果最终结果是错误，草稿会保留，作为该失败轮次的可见记录。
- 如果草稿可以安全地变为最终答案（`partial`/`block` 模式），
  OpenClaw 会就地编辑草稿。
- 如果渠道使用原生进度流式传输，当原生传输接受最终文本时，
  OpenClaw 会完成该流。
- 否则（媒体、审批提示、显式回复目标、分块过多，
  或编辑/发送失败），OpenClaw 会通过常规渠道投递路径发送最终答案，
  而不是覆盖草稿。

这种回退是有意设计的：发送一条新的最终答案，胜过丢失文本、
将回复发到错误的线程，或使用渠道无法安全表示的载荷覆盖草稿。

## 故障排查

**我只能看到最终答案。**

检查处理该消息的账户或渠道中，`channels.<channel>.streaming.mode` 是否为
`progress`。当渠道无法安全编辑正确的消息时，某些群组或引用回复路径
会针对该轮次禁用草稿预览。

**我能看到标签，但看不到工具行。**

检查 `streaming.progress.toolProgress`。如果它是 `false`，OpenClaw 会保留
单草稿行为，但隐藏工具和任务进度行。

**我看到的是一条新的最终消息，而不是经过编辑的草稿。**

这是[最终处理](#finalization)中所述的安全回退。媒体回复、较长答案、
显式回复目标、旧 Telegram 草稿、缺少 Slack 线程目标、已删除的预览消息，
或原生流最终处理失败，都可能触发这种情况。

**我仍能看到独立的进度消息。**

只要草稿处于活动状态，进度模式就会抑制默认的独立工具进度消息。
如果仍出现独立消息，请确认该轮次实际使用的是
`progress` 模式，而不是 `streaming.mode: "off"`，并且未使用无法为该消息
创建草稿的渠道路径。

**Teams 的行为与 Discord 或 Telegram 不同。**

Microsoft Teams 在个人聊天中使用原生流，而不是通用的
发送并编辑预览传输方式；它会将 `streaming.mode: "block"` 映射到 Teams
分块投递，因为它没有 Discord 和 Telegram 那样的草稿预览分块模式。

## 相关内容

- [流式传输和分块](/zh-CN/concepts/streaming)
- [消息](/zh-CN/concepts/messages)
- [频道配置](/zh-CN/gateway/config-channels)
- [Discord](/zh-CN/channels/discord)
- [Matrix](/zh-CN/channels/matrix)
- [Microsoft Teams](/zh-CN/channels/msteams)
- [Slack](/zh-CN/channels/slack)
- [Telegram](/zh-CN/channels/telegram)
- [Mattermost](/zh-CN/channels/mattermost)
