---
read_when:
    - 你看到一个 `.experimental` 配置键，并想知道它是否稳定
    - 你想尝试预览版运行时功能，同时避免将它们与常规默认值混淆
    - 你希望有一个地方可以查找当前已记录的实验性标志位
summary: OpenClaw 中实验性标志的含义及当前已记录的标志
title: 实验性功能
x-i18n:
    generated_at: "2026-07-26T06:07:17Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 6c14b74bbafce77c0d1e1358ad94053675c4aad9e26be78719f58e78f455c3a2
    source_path: concepts/experimental-features.md
    workflow: 16
---

实验性功能是需要通过显式标志启用的预览功能。在获得稳定的默认设置或长期有效的契约之前，它们还需要更多实际使用检验。

- 除非文档描述了范围明确的自动设置规则，否则默认关闭。
- 其结构和行为的变化速度可能快于稳定配置。
- 如果已有稳定路径，请优先使用。
- 仅在较小环境中先行测试后，才进行广泛部署。

## 当前已记录的标志

| 功能界面                  | 键                                                                                           | 适用场景                                                                                                                       | 更多信息                                                                                   |
| ------------------------ | --------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------- |
| 本地模型运行时      | `agents.defaults.experimental.localModelLean`, `agents.entries.*.experimental.localModelLean` | 较小或限制更严格的本地后端无法处理 OpenClaw 完整的默认工具界面                                                | [本地模型](/zh-CN/gateway/local-models)                                                  |
| Codex harness            | `plugins.entries.codex.config.appServer.experimental.sandboxExecServer`                       | 你希望原生 Codex app-server 0.132.0 或更高版本以 OpenClaw 沙箱支持的 exec-server 为目标，而不是禁用代码模式 | [Codex harness reference](/zh-CN/plugins/codex-harness-reference#sandboxed-native-execution) |
| 结构化规划工具 | `tools.experimental.planTool`                                                                 | 你希望在兼容的运行时和 UI 中公开结构化 `update_plan` 工具，用于跟踪多步骤工作                    | [Gateway 配置参考](/zh-CN/gateway/config-tools#toolsexperimental)             |
| 代码模式                | `tools.codeMode.enabled`                                                                      | 你希望通过紧凑的代码编排方式访问隐藏的 OpenClaw 工具目录                                                       | [代码模式](/zh-CN/tools/code-mode)                                                          |
| Swarm                    | `tools.swarm.enabled`                                                                         | 你希望代码模式脚本并行编排有界的子智能体组                                                | [Swarm](/tools/swarm)                                                                  |

## Control UI 实验室

打开 **Settings → Agents & Tools → Labs**，管理带有
Control UI 开关的实验。启用或禁用实验室功能会立即修补规范的 Gateway 网关
配置；仅当某项功能需要重启时，页面才会显示重启提示。

代码模式和 Swarm 是目前已发布的实验室条目。两个开关均会
写入现有的、已验证的配置键，并且通常无需重启 Gateway 网关，
即可对后续智能体运行生效。

## 本地模型精简模式

`agents.defaults.experimental.localModelLean: true` 会在每轮中从智能体的直接界面移除重量级可选工具：`browser`、`cron`、`message`、`image_generate`、`music_generate`、`video_generate`、`tts` 和 `pdf`。显式允许或交付所需的工具仍然可用，但工具搜索可能会将其收录到目录中，而不是直接公开。未设置 `tools.toolSearch` 时，精简模式还会默认将插件/MCP/客户端目录设为结构化工具搜索（`tool_search`、`tool_describe`、`tool_call`）。使用 `agents.entries.*.experimental.localModelLean` 可将此设置限定到一个智能体。

在新手引导期间，经过验证的 `ollama` 或 `lmstudio` 推理路由会在缺少该值时自动设置 `agents.defaults.experimental.localModelLean: true`。OpenClaw 会记录该设置来自新手引导，因此之后经过验证的非本地路由只会撤销自动设置。显式配置的 `true` 或 `false` 会得到保留。不会根据模型名称或 URL 推断其他自托管提供商和 OpenAI 兼容提供商。

如果你已全局调整工具搜索，OpenClaw 会保留该配置不变。设置 `tools.toolSearch: false` 可选择不使用精简模式的默认工具搜索设置。

在结构化 `tools` 模式下，精简运行会使 `exec` 与工具搜索控件并列且保持直接可见，以便针对编码调优的本地模型仍可选择其熟悉的 shell 路径。这仅改变架构可见性：常规工具策略、沙箱隔离和 Exec 审批仍然适用。显式的 `code` 和 `directory` 模式会保留其常规压缩行为。

### 为什么选择这些工具

这些工具的描述最长、参数结构最广，或最有可能使小型模型偏离正常的编码和对话路径。对于上下文较小或限制更严格的 OpenAI 兼容后端，这决定了以下差异：

- 工具架构能够放入提示词，而不是挤占对话历史的空间。
- 模型能够选择正确的工具，而不是因为存在过多相似架构而发出格式错误的工具调用。
- Chat Completions 适配器能够保持在结构化输出限制以内，而不是因工具调用负载过大而返回 400。

移除这些工具只会缩短直接工具列表。模型仍可使用 `read`、`write`、`edit`、`exec`、`apply_patch`、图像理解、Web 搜索/获取（配置后）、记忆以及会话/智能体工具。除非设置 `tools.toolSearch: false`，否则仍可通过工具搜索访问其他目录；显式允许工具可以使精简智能体重新使用已被精简的工作流。

### 何时启用

确认模型能够与 Gateway 网关通信，但完整智能体轮次行为异常后，再启用精简模式：

1. `openclaw infer model run --gateway --model <ref> --prompt "Reply with exactly: pong"` 成功。
2. 常规智能体轮次因工具调用格式错误、提示词过大或模型忽略工具而失败。
3. 切换 `localModelLean: true` 后故障消失。

### 何时保持关闭

如果后端可以顺利处理完整的默认运行时，请保持关闭。它是针对需要更小工具界面的本地技术栈的变通方案，并非托管模型或资源充足的本地设备的默认设置。

精简模式不能替代 `tools.profile`、`tools.allow`/`tools.deny` 或模型的 `compat.supportsTools: false` 备用机制。若要为特定智能体永久缩小工具界面，请优先使用这些稳定的配置项。

### 启用

```json5
{
  agents: {
    defaults: {
      experimental: {
        localModelLean: true,
      },
    },
  },
}
```

仅用于一个智能体：

```json5
{
  agents: {
    list: [
      {
        id: "local",
        model: "lmstudio/gemma-4-e4b-it",
        experimental: {
          localModelLean: true,
        },
      },
    ],
  },
}
```

更改标志后重启 Gateway 网关。除非通过 `tools.allow` 或 `tools.alsoAllow` 显式保留，否则精简筛选会移除 `browser`、`cron`、`message`、`image_generate`、`music_generate`、`video_generate`、`tts` 和 `pdf`；工具搜索仍可能将保留的工具收录到目录中，而不是直接公开。

## 实验性并不意味着隐藏

实验性功能应在文档及配置路径本身中明确说明其性质，而不应隐藏在看似稳定的默认配置项后面。

## 相关内容

- [功能](/zh-CN/concepts/features)
- [发布渠道](/zh-CN/install/development-channels)
