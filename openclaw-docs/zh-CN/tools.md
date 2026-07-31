---
doc-schema-version: 1
read_when:
    - 你想了解 OpenClaw 提供哪些工具
    - 你正在内置工具、Skills 和插件之间进行选择
    - 你需要找到适用于工具策略、自动化或智能体协调的正确文档入口点
summary: OpenClaw 工具、技能和插件概览：智能体可以调用什么以及如何扩展它们
title: 概览
x-i18n:
    generated_at: "2026-07-26T06:26:54Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 45745bd5f2008a84cb6c4c1c9840073bfa8a9c40a0ff65bfefc682c5d99af09b
    source_path: tools/index.md
    workflow: 16
---

使用本页选择合适的能力界面。**工具**是可调用的操作，**技能**用于指导智能体如何工作，而**插件**则添加运行时能力，例如工具、提供商、渠道、钩子和打包的技能。

这是一个概览和导航页面。有关完整的工具策略、默认值、组成员关系、提供商限制和配置字段，请参阅[工具和自定义提供商](/zh-CN/gateway/config-tools)。

## 从这里开始

对于大多数智能体，请先使用内置工具类别，然后仅在智能体应该看到更少的工具或需要明确的主机访问权限时调整策略。

| 如果需要……                                   | 请先使用                                       | 然后阅读                                                                                                                                               |
| -------------------------------------------- | ---------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------ |
| 让智能体使用现有能力执行操作                 | [内置工具](#built-in-tool-categories)          | [工具类别](#built-in-tool-categories)                                                                                                                  |
| 控制智能体可以调用的内容                     | [工具策略](#configure-access-and-approvals)    | [工具和自定义提供商](/zh-CN/gateway/config-tools)                                                                                                            |
| 向智能体传授工作流                           | [Skills](#choose-tools-skills-or-plugins)      | [Skills](/zh-CN/tools/skills)、[创建技能](/zh-CN/tools/creating-skills)、[技能工作坊](/zh-CN/tools/skill-workshop)和[自我学习](/zh-CN/tools/self-learning)                       |
| 添加新的集成或运行时界面                     | [插件](#extend-capabilities)                   | [插件](/zh-CN/tools/plugin)和[构建插件](/zh-CN/plugins/building-plugins)                                                                                           |
| 稍后或在后台运行工作                         | [自动化](/zh-CN/automation)                          | [自动化概览](/zh-CN/automation)                                                                                                                              |
| 协调多个智能体或 harness                     | [子智能体](/zh-CN/tools/subagents)                   | [ACP 智能体](/zh-CN/tools/acp-agents)和[智能体发送](/zh-CN/tools/agent-send)                                                                                       |
| 通过代码编排并发智能体                       | [Swarm](/tools/swarm)                          | [代码模式](/zh-CN/tools/code-mode)和[子智能体](/zh-CN/tools/subagents)                                                                                             |
| 搜索大型 OpenClaw 工具目录                   | [工具搜索](/zh-CN/tools/tool-search)                 | [工具搜索](/zh-CN/tools/tool-search)                                                                                                                         |
| 在一个紧凑程序中组合多个工具                 | [代码模式](/zh-CN/tools/code-mode)                   | [代码模式](/zh-CN/tools/code-mode)                                                                                                                           |

## 选择工具、Skills 或插件

<Steps>
  <Step title="当智能体需要执行操作时使用工具">
    工具是智能体可以调用的类型化函数，例如 `exec`、`browser`、
    `web_search`、`message` 或 `image_generate`。当智能体需要读取数据、更改文件、
    发送消息、调用提供商或操作其他系统时，请使用工具。可见工具会以结构化函数定义的形式发送给模型。

    模型只能看到经过当前配置文件、允许/拒绝策略、提供商限制、沙箱状态、渠道权限和插件可用性筛选后保留下来的工具。

  </Step>

  <Step title="当智能体需要说明时使用 Skills">
    Skill 是加载到智能体提示词中的 `SKILL.md` 指令包。当智能体已经拥有所需工具，但需要可重复的工作流、
    审查准则、命令序列或操作约束时，请使用 Skill。

    Skills 可以位于工作区、共享技能目录、托管的 OpenClaw 技能根目录或插件包中。

    [Skills](/zh-CN/tools/skills) | [技能工作坊](/zh-CN/tools/skill-workshop) | [自我学习](/zh-CN/tools/self-learning) | [创建技能](/zh-CN/tools/creating-skills) | [Skills 配置](/zh-CN/tools/skills-config)

  </Step>

  <Step title="当 OpenClaw 需要新能力时使用插件">
    插件可以添加工具、Skills、渠道、模型提供商、语音、实时语音、媒体生成、Web 搜索、Web 获取、钩子及其他运行时能力。
    当能力包含代码、凭据、生命周期钩子、清单元数据或可安装的软件包时，请使用插件。现有插件可以从 ClawHub、npm、git、
    本地目录或归档文件安装。

    [安装和配置插件](/zh-CN/tools/plugin) | [构建插件](/zh-CN/plugins/building-plugins) | [插件 SDK](/zh-CN/plugins/sdk-overview)

  </Step>
</Steps>

## 内置工具类别

下表列出了代表性工具，帮助你了解相应界面。它不是完整的策略参考。有关确切的组、默认值以及允许/拒绝语义，请参阅[工具和自定义提供商](/zh-CN/gateway/config-tools)。

| 类别                    | 在智能体需要执行以下操作时使用……                                                     | 代表性工具                                                                                                          | 接下来阅读                                                                                                             |
| ----------------------- | ------------------------------------------------------------------------------------ | ------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------- |
| 运行时                  | 运行命令、管理进程或使用由提供商支持的 Python 分析                                   | `exec`、`process`、`terminal`、`code_execution`                                     | [Exec](/zh-CN/tools/exec)、[Control UI 终端](/zh-CN/web/control-ui#operator-terminal)、[代码执行](/zh-CN/tools/code-execution)            |
| 文件                    | 读取和更改工作区文件                                                                 | `read`、`write`、`edit`、`apply_patch`                                     | [应用补丁](/zh-CN/tools/apply-patch)                                                                                         |
| 人工输入                | 暂停并等待由用户决定的结构化决策                                                     | `ask_user`                                                                                                  | [询问用户](/tools/ask-user)                                                                                            |
| Web                     | 搜索 Web、搜索 X 帖子或获取可读的页面内容                                            | `web_search`、`x_search`、`web_fetch`                                                         | [Web 工具](/zh-CN/tools/web)、[Web 获取](/zh-CN/tools/web-fetch)                                                                   |
| 浏览器                  | 操作浏览器会话                                                                       | `browser`                                                                                                  | [浏览器](/zh-CN/tools/browser)                                                                                               |
| 操作员界面              | 排列已连接的 Control UI 窗格、面板和导航                                             | `screen`                                                                                                  | [屏幕](/tools/screen)                                                                                                  |
| 消息和渠道              | 发送回复或执行渠道操作                                                               | `message`                                                                                                  | [智能体发送](/zh-CN/tools/agent-send)                                                                                        |
| 会话和智能体            | 检查会话、委派工作、编排收集器、引导另一次运行或报告状态                             | `sessions_*`、`agents_wait`、`subagents`、`agents_list`、`session_status`、`get_goal`、`create_goal`、`update_goal` | [目标](/zh-CN/tools/goal)、[Swarm](/tools/swarm)、[子智能体](/zh-CN/tools/subagents)、[会话工具](/zh-CN/concepts/session-tool)           |
| 自动化                  | 安排工作或响应后台事件                                                               | `cron`、`heartbeat_respond`                                                                              | [自动化](/zh-CN/automation)                                                                                                  |
| Gateway 网关和节点      | 检查 Gateway 网关状态或已配对的目标设备                                              | `gateway`、`nodes`                                                                              | [Gateway 配置](/zh-CN/gateway/configuration)、[节点](/zh-CN/nodes)                                                                 |
| 媒体                    | 分析、生成媒体或将其转换为语音                                                       | `image`、`image_generate`、`music_generate`、`video_generate`、`tts`                 | [媒体概览](/zh-CN/tools/media-overview)                                                                                      |
| 大型 OpenClaw 目录      | 搜索、调用并组合大量符合条件的工具，而无需将每个架构都发送给模型                     | `exec`、`wait`、`tool_search_code`、`tool_search`、`tool_describe`                 | [代码模式](/zh-CN/tools/code-mode)、[工具搜索](/zh-CN/tools/tool-search)                                                           |

<Note>
代码模式和工具搜索是实验性的 OpenClaw 智能体界面。Codex harness 运行使用 Codex 原生代码模式、原生工具搜索、延迟动态工具和嵌套工具调用，而不是 `tools.codeMode` 或 `tools.toolSearch`。
</Note>

## 插件提供的工具

插件可以注册其他工具。插件作者通过 `api.registerTool(...)` 和清单中的 `contracts.tools` 接入工具；有关契约详情，请参阅[插件 SDK](/zh-CN/plugins/sdk-overview)和[插件清单](/zh-CN/plugins/manifest)。

常见的插件提供工具包括：

- [Diffs](/zh-CN/tools/diffs)，用于呈现文件和 Markdown 差异
- [显示小组件](/zh-CN/tools/show-widget)，用于在支持的聊天客户端中呈现自包含的内联 SVG 和 HTML
- [屏幕](/tools/screen)，用于排列已连接的 Control UI
- [LLM 任务](/zh-CN/tools/llm-task)，用于仅使用 JSON 的工作流步骤
- [Lobster](/zh-CN/tools/lobster)，用于支持可恢复审批的类型化工作流
- [Tokenjuice](/zh-CN/tools/tokenjuice)，用于压缩冗杂的 `exec` 和 `bash` 工具
  输出
- [工具搜索](/zh-CN/tools/tool-search)，用于发现和调用大型工具
  目录，而无需将每个 schema 都放入提示词
- [Canvas](/zh-CN/plugins/reference/canvas)，用于节点 Canvas 控制和 A2UI
  渲染

## 配置访问权限和审批

工具策略在模型调用前执行。如果策略移除了某个工具，模型在该轮中
将不会收到该工具的 schema。一次运行可能因全局配置、按 Agent 配置、渠道策略、提供商
限制、沙箱规则、渠道/运行时策略或插件可用性而失去工具。

- [工具和自定义提供商](/zh-CN/gateway/config-tools)介绍工具配置文件、
  允许/拒绝列表、提供商特定限制、循环检测和
  提供商支持的工具设置。
- [Exec 审批](/zh-CN/tools/exec-approvals)介绍主机命令审批
  策略。
- [提升权限的 Exec](/zh-CN/tools/elevated)介绍在沙箱外进行的受控
  执行。
- [沙箱、工具策略和提升权限](/zh-CN/gateway/sandbox-vs-tool-policy-vs-elevated)
  说明哪个层级控制文件和进程访问权限。
- [按 Agent 配置的沙箱和工具限制](/zh-CN/tools/multi-agent-sandbox-tools)
  介绍委派运行中针对智能体的特定限制。

## 扩展能力

根据需要 OpenClaw 完成的任务选择扩展路径：

- 使用[插件](/zh-CN/tools/plugin)安装或管理现有插件。
- 使用[构建插件](/zh-CN/plugins/building-plugins)构建新的集成、提供商、渠道、工具或钩子。
- 使用 [Skills](/zh-CN/tools/skills) 和
  [创建技能](/zh-CN/tools/creating-skills)添加或调整可复用的智能体指令。
- 需要实现契约时，请使用[插件 SDK](/zh-CN/plugins/sdk-overview)和
  [插件清单](/zh-CN/plugins/manifest)。

## 排查工具缺失问题

如果模型无法看到或调用某个工具，请从当前轮次的有效策略
开始排查：

1. 在[工具和自定义提供商](/zh-CN/gateway/config-tools)中检查活动配置文件、`tools.allow` 和 `tools.deny`。
2. 在[工具和自定义提供商](/zh-CN/gateway/config-tools)中检查提供商特定限制，并确认所选
   [模型提供商](/zh-CN/concepts/model-providers)支持该工具
   结构。
3. 使用[沙箱、工具策略和提升权限](/zh-CN/gateway/sandbox-vs-tool-policy-vs-elevated)
   和[提升权限的 Exec](/zh-CN/tools/elevated)检查渠道权限、沙箱状态和提升权限访问。
4. 在[插件](/zh-CN/tools/plugin)中检查所属插件是否已安装并启用。
5. 对于委派运行，请在[按 Agent 配置的沙箱和工具限制](/zh-CN/tools/multi-agent-sandbox-tools)中检查按 Agent 配置的限制。
6. 对于大型 OpenClaw 目录，请确认运行使用的是直接工具
   暴露、[代码模式](/zh-CN/tools/code-mode)还是[工具搜索](/zh-CN/tools/tool-search)。

## 相关内容

- [自动化](/zh-CN/automation)，涵盖 cron、任务、Heartbeat、钩子、
  常设指令和 Task Flow
- [智能体](/zh-CN/concepts/agent)，介绍智能体模型、会话、记忆和
  多智能体协调
- [工具和自定义提供商](/zh-CN/gateway/config-tools)，提供权威工具
  策略参考
- [插件](/zh-CN/tools/plugin)，介绍插件安装和管理
- [插件 SDK](/zh-CN/plugins/sdk-overview)，提供插件作者参考
- [Skills](/zh-CN/tools/skills)，介绍技能加载顺序、门控和配置
- [技能工坊](/zh-CN/tools/skill-workshop)，用于生成和审查技能
  创建
- [工具搜索](/zh-CN/tools/tool-search)，用于精简的 OpenClaw 工具目录
  发现
- [代码模式](/zh-CN/tools/code-mode)，用于通过隐藏的 OpenClaw 工具目录运行精简的 JavaScript 或 TypeScript 工作流
- [Swarm](/tools/swarm)，用于从代码模式进行结构化扇出和结果收集
