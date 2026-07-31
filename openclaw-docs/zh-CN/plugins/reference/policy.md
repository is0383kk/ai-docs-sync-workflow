---
read_when:
    - 你正在安装、配置或审计策略插件
summary: 新增由策略支持的 Doctor 工作区合规性检查。
title: 策略插件
x-i18n:
    generated_at: "2026-07-26T06:27:14Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 440f2f46e4149fdd5e65bf0140d4981c6d840e8e8c8a85d05eeb23a0839a61ac
    source_path: plugins/reference/policy.md
    workflow: 16
---

# Policy 插件

添加基于策略的 Doctor 检查，以验证工作区合规性。

## 分发

- 软件包：`@openclaw/policy`
- 安装方式：OpenClaw 内置

## 接口

插件

<!-- openclaw-plugin-reference:manual-start -->

## 行为

Policy 插件为策略管理的 OpenClaw 设置和受管控的工作区声明提供 Doctor 健康检查。Policy 目前涵盖渠道合规性、受管控的工具元数据、MCP 服务器安全状况、模型提供商安全状况、专用网络访问安全状况、Gateway 网关暴露安全状况、智能体工作区/工具安全状况、已配置的全局/每智能体工具安全状况、已配置的沙箱运行时安全状况、入口/渠道访问安全状况、数据处理安全状况，以及 OpenClaw 配置密钥提供商/身份验证配置文件的安全状况。

Policy 将编写的要求存储在 `policy.jsonc` 中，将现有 OpenClaw 设置和工作区声明作为证据进行观察，并通过 `openclaw policy check` 和 `openclaw doctor --lint` 报告偏差。无异常的策略检查会生成策略、证据、发现和认证哈希值，操作员可将其记录下来用于审计。

`openclaw policy compare --baseline <file>` 将一个策略文件与另一个策略文件进行比较。它只检查配置级合规性：使用策略规则元数据验证被检查的策略是否存在缺失项或弱于编写的基准，并且不会检查运行时状态、凭据或密钥值。

工具安全状况规则可以要求使用获批的配置文件、仅限工作区的文件系统工具、受限的 Exec 安全性/询问/主机设置、禁用提升权限模式、完全匹配的 `alsoAllow` 条目，以及必需的工具拒绝条目。证据记录会添加增量 `alsoAllow` 条目，因为这些条目可能扩大有效工具权限范围。这些检查仅观察配置合规性；不会读取运行时审批状态或添加运行时强制措施。

沙箱安全状况规则可以要求使用获批的沙箱模式/后端、禁止主机容器联网、禁止加入容器命名空间、要求容器挂载为只读、禁止挂载容器运行时套接字和使用不受限的容器配置文件，以及要求配置沙箱浏览器 CDP 来源范围。
这些检查仅观察配置合规性；不会读取运行时审批状态、检查正在运行的容器或添加运行时强制措施。

数据处理规则可以要求对敏感日志进行脱敏、禁止遥测内容捕获、要求执行会话保留维护，以及禁止对会话转录建立记忆索引。这些检查仅观察配置合规性；不会检查原始日志、遥测导出、转录、记忆文件、密钥或个人数据。

`scopes.<scopeName>` 下的命名策略作用域可以为其列出的选择器添加更严格的常规策略部分。`agentIds` 支持 `tools`、`agents.workspace`、`sandbox` 和 `dataHandling.memory`；`channelIds` 支持
`ingress.channels`。
未在 `agents.entries.*` 中明确列出的运行时智能体 ID 会根据继承的全局/默认安全状况进行检查，而不会在没有证据的情况下静默通过。`policy.jsonc` 中存在的每个作用域都必须对其选择器有效且可强制执行。叠加规则属于附加声明，因此不会削弱顶层策略；当同一项观察到的配置同时违反两个作用域时，这些规则可以分别生成自己的发现。

<!-- openclaw-plugin-reference:manual-end -->

## 相关文档

- [策略](/zh-CN/cli/policy)
