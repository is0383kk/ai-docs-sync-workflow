---
read_when:
    - 你希望以交互方式调整凭据、设备或智能体默认设置
summary: '`openclaw configure` 的 CLI 参考（交互式配置提示）'
title: 配置
x-i18n:
    generated_at: "2026-07-26T06:40:04Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 5980d06e75a5df9e5269d0ef78431f730d6f5fd050dca74784ef3426fb0433d8
    source_path: cli/configure.md
    workflow: 16
---

# `openclaw configure`

通过交互式提示对现有设置进行针对性更改：凭据、设备、智能体默认值、Gateway 网关、渠道、插件、Skills 和健康检查。

使用 `openclaw onboard` 或 `openclaw setup` 完成完整的首次运行引导流程；仅设置基础配置/工作区时使用 `openclaw setup --baseline`；仅需设置渠道账户时使用 `openclaw channels add`。

<Tip>
不带子命令的 `openclaw config` 会打开同一个向导。使用 `openclaw config get|set|unset` 进行非交互式编辑。
</Tip>

## 选项

`--section <section>`：可重复使用的分区筛选器。可用分区：

`workspace`、`model`、`web`、`gateway`、`daemon`、`channels`、`plugins`、`skills`、`health`

```bash
openclaw configure
openclaw configure --section web
openclaw configure --section model --section channels
openclaw configure --section gateway --section daemon
```

选择 `gateway`、`daemon` 或 `health`（或者在没有 `--section` 的情况下运行完整向导）时，系统会询问 Gateway 网关的运行位置并更新 `gateway.mode`。跳过这三个分区的分区筛选器会直接进入请求的设置流程，不显示网关模式提示。选择远程网关模式会写入远程配置并立即退出；不会执行安装插件等仅限本地的步骤。

<Note>
`openclaw configure` 需要交互式终端（stdin 和 stdout 都必须是 TTY）。如果没有交互式终端，它会输出等效的非交互式 `openclaw config get|set|patch|validate` 命令并报错退出，而不是仅执行部分流程。
</Note>

## 模型分区

<Note>
**模型**包含一个多选项，用于设置显式的 `agents.defaults.modelPolicy.allow` 列表（即 `/model` 和模型选择器中显示的内容）。限定提供商范围的设置选项会将所选模型合并到现有列表中，而不会替换配置中已有的其他无关提供商。各模型的别名和参数仍位于 `agents.defaults.models` 下；这些条目本身不会限制模型覆盖。

从 configure 重新运行提供商身份验证时，会保留现有的 `agents.defaults.model.primary`，即使提供商的身份验证步骤返回的配置补丁包含其自行推荐的默认模型。添加提供商或对其重新进行身份验证只会使其模型可用，不会取代当前的主要模型。使用 `openclaw models auth login --provider <id> --set-default` 或 `openclaw models set <model>` 可主动更改默认模型。
</Note>

当 configure 从某个提供商的身份验证选项启动时，默认模型和模型策略选择器会自动优先选择该提供商。对于火山引擎和 BytePlus 等配对提供商，同一偏好也会匹配其编程套餐变体（`volcengine-plan/*`、`byteplus-plan/*`）。如果首选提供商筛选结果为空，configure 会回退到未筛选的目录，而不是显示空白选择器。

## Web 分区

`openclaw configure --section web` 用于选择 Web 搜索提供商并配置其凭据。某些提供商会显示特定于提供商的后续设置：

- **Grok** 可以提供可选的 `x_search` 设置，该设置使用相同的 xAI OAuth 配置文件或 API key，并允许选择一个 `x_search` 模型。
- **Kimi** 可以询问 Moonshot API 区域（`api.moonshot.ai` 或 `api.moonshot.cn`）以及默认的 Kimi Web 搜索模型。

## 其他说明

- 写入本地配置后，如果所选设置路径需要可下载插件，configure 会安装选中的插件。远程网关配置不会安装本地插件包。
- 面向渠道的服务（Slack/Discord/Matrix/Microsoft Teams）会在设置期间提示配置渠道/房间允许列表。可以输入名称或 ID；向导会尽可能将名称解析为 ID。
- 如果运行后台进程安装步骤，令牌身份验证需要令牌。如果 `gateway.auth.token` 由 SecretRef 管理，configure 会验证 SecretRef，但不会将解析后的明文令牌值持久化到监管服务的环境元数据中；如果 SecretRef 未解析，configure 会阻止后台进程安装，并提供可操作的修复指导。
- 如果同时配置了 `gateway.auth.token` 和 `gateway.auth.password`，但未设置 `gateway.auth.mode`，configure 会阻止后台进程安装，直到明确设置模式。

## 相关内容

- [CLI 参考](/zh-CN/cli)
- [配置](/zh-CN/gateway/configuration)
- 配置 CLI：[配置](/zh-CN/cli/config)
