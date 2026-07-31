---
read_when:
    - 你正在编写或验证 CLAW.md 清单文件
    - 你想从一个 Claw 中预览或添加一个智能体
    - 你需要检查 Claw 的所有权、漂移或清理行为
summary: 创建、添加、更新和移除实验性 Claw 智能体包
title: Claws
x-i18n:
    generated_at: "2026-07-26T06:04:27Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: da4b52bdee2b4cf4898677aadeeabb2c0cf98e7c3c53cec6f0b4c6d0b8ab3ae5
    source_path: cli/claws.md
    workflow: 16
---

# `openclaw claws`

Claw 是用于一个新 OpenClaw 智能体的版本化设置。它可以描述智能体的可移植身份、工作区文件、Skills、插件、MCP 服务器和定时任务。特定于 harness 的智能体设置可以包含在引用的包配置文件中。Claw 不会替换或修改现有智能体。

Claws 是实验性功能。其架构、命令输出和生命周期可能会发生变化。
请显式启用命令界面：

```bash
export OPENCLAW_EXPERIMENTAL_CLAWS=1
```

当前 CLI 会读取本地包目录、`CLAW.md` 或分组的 JSON 清单。
通过 ClawHub 发布、搜索和安装完整 Claws 属于单独的注册表路线，尚未纳入此命令界面。

## 创建 Claw 包

包包含 `package.json`、一个 `CLAW.md` 清单，以及该清单引用的任何配置文件或工作区附属文件：

```json
{
  "name": "@acme/incident-triage-claw",
  "version": "1.0.0",
  "type": "module",
  "openclaw": { "claw": "CLAW.md" }
}
```

`CLAW.md` 以 YAML frontmatter 开头。其 Markdown 正文面向用户描述 Claw，不属于智能体配置：

```md
---
schemaVersion: 1
agent:
  id: incident-triage
  name: 事件分诊
metadata:
  openclaw.config: profiles/openclaw.yml
workspace:
  bootstrapFiles: {}
packages: []
mcpServers: {}
cronJobs: []
---

# 事件分诊

创建一个用于审查和路由事件的智能体。
```

`metadata` 是用于可移植使用方提示的字符串到字符串映射。OpenClaw 的
`openclaw.config` 键指向可选的包相对 YAML 配置文件。导出的默认值为 `profiles/openclaw.yml`；该指针具有规范性，因此包可以选择另一个安全的相对 `.yml` 或 `.yaml` 路径。

```yaml
schemaVersion: 1
agent:
  tools:
    profile: coding
    alsoAllow: [cron]
    deny: [exec]
    fs:
      workspaceOnly: true
  memory:
    search:
      enabled: true
      rememberAcrossConversations: true
      sources: [memory, sessions]
```

此配置文件仅存在于 Claw 包内。OpenClaw 在检查、添加、更新和导出该 Claw 时验证并使用它；它不会被复制到用户的常规 OpenClaw 配置路径。其他 harness 可以忽略带命名空间的元数据键，并使用可移植清单字段。

相同的严格版本 1 架构继续接受分组的 JSON 清单。分组 JSON 使用相同的 `metadata.openclaw.config` 指针，而不是嵌入 OpenClaw 配置文件的第二份副本。本页其余架构片段使用 JSON，`CLAW.md` frontmatter 中提供等效键。

OpenClaw 包配置文件可以选择运行中 OpenClaw 版本注册的任何内置工具配置文件，然后使用 `alsoAllow`、`deny` 和 `tools.fs.workspaceOnly: true` 进一步调整。Claw 不能将该字段设置为 `false` 并削弱主机文件系统限制。`tools.allow` 仍可用作显式允许列表，但不能与 `alsoAllow` 组合使用。Claw 还可以设置 `memory.search.enabled`，选择可移植的 `memory` 和 `sessions` 来源，并通过 `rememberAcrossConversations` 选择启用跨对话记忆。
声明 `sessions` 来源需要启用该选项。
主机策略仍会约束这些设置，Claws 不携带自定义配置文件定义、提供商、凭据、绑定或本地记忆路径。
引用的配置文件大小限制为 256 KiB，必须是与 JSON 兼容的 YAML，不得使用别名、锚点、标签或合并键，并且必须是包内的常规文件，不能是符号链接或硬链接。

包路径和工作区路径必须保持在包根目录内。清单大小限制为 1 MiB，包元数据限制为 256 KiB，工作区来源则分别实施单文件限制和总量限制。工作区来源还会拒绝包含符号链接的父目录。

工作区文件按路径声明，并从包附属文件中读取。`SOUL.md` 等引导文件使用命名条目；其他文件使用包相对来源和工作区相对目标：

```json
{
  "workspace": {
    "bootstrapFiles": {
      "SOUL.md": { "source": "workspace/SOUL.md" }
    },
    "files": [
      {
        "source": "workspace/reference/policy.md",
        "path": "reference/policy.md"
      }
    ]
  }
}
```

Skills 和插件使用精确的 ClawHub 版本：

```json
{
  "packages": [
    {
      "kind": "skill",
      "source": "clawhub",
      "ref": "incident-triage",
      "version": "1.0.0"
    },
    {
      "kind": "plugin",
      "source": "clawhub",
      "ref": "@acme/audit-plugin",
      "version": "2.0.0"
    }
  ]
}
```

试运行使用现有的技能和插件预检路径，在同意前解析确切工件、完整性信息和任何 ClawHub 信任警告。警告会继续显示在受完整性约束的计划中。应用操作会安装缺失的工件或复用匹配的工件，并记录 Claw 是引入还是引用了每项资源。插件仍是进程范围的 OpenClaw 能力，而不是按智能体安装。

定时任务声明新智能体要执行的计划工作：

```json
{
  "cronJobs": [
    {
      "id": "daily-summary",
      "name": "每日事件摘要",
      "schedule": { "cron": "0 9 * * *", "timezone": "UTC" },
      "session": "isolated",
      "message": "汇总活动事件。"
    }
  ]
}
```

Claws 使用现有的 Gateway 网关调度器，并将创建的任务绑定到新智能体。预览、来源记录、状态和移除操作涵盖这些任务，而不会改变普通定时任务命令的行为。移除操作会通过 Gateway 网关重新读取实时任务；如果其拥有的定义在规划后发生变化，则会保留该任务。

MCP 声明使用现有的 `mcp.servers` 配置模型：

```json
{
  "mcpServers": {
    "statuspage": {
      "command": "npx",
      "args": ["--yes", "@acme/statuspage-mcp@1.0.0"],
      "env": { "STATUSPAGE_TOKEN": "${STATUSPAGE_TOKEN}" }
    }
  }
}
```

环境引用仍保持为引用；Claws 不嵌入解析后的密钥值。无冲突的声明会变为托管状态，而完全相同的现有声明或共享声明会被引用。预览、来源记录、状态、导出和移除遵循与其他 Claw 资源相同的所有权策略。

## 检查和预览

验证来源而不规划本地更改：

```bash
openclaw claws inspect ./incident-triage.claw.json
```

预览所有拟议的生命周期操作：

```bash
openclaw claws add ./incident-triage.claw.json --dry-run --json
```

计划会报告派生的智能体和工作区、每项拟议操作、前提条件、阻碍因素、各项不同的能力提升，以及一个 `planIntegrity` 摘要。能力记录会显示确切的包、MCP、计划工作、沙箱、工具或 Heartbeat 影响。创建智能体前请审查计划：

```bash
openclaw claws add ./incident-triage.claw.json \
  --yes \
  --plan-integrity <SHA256_FROM_DRY_RUN>
```

仅使用 `--yes` 并不足够。如果预览后来源、目标或实时配置发生变化，OpenClaw 会重新构建计划并拒绝同意。当包默认值与本地状态冲突时，请在预览和应用期间都使用 `--agent-id` 或 `--workspace`。对于一次性配置文件和并行验证，请传递显式的 `--workspace`；`OPENCLAW_STATE_DIR` 会重新定位运行时状态，但不会更改默认工作区位置。

添加 Claw 会创建新的智能体和工作区配置，写入声明的工作区文件，安装或复用声明的技能和插件工件，并记录包、MCP 和定时任务的来源信息。现有文件不会被覆盖；如果拥有的内容发生漂移，重试会以关闭方式失败。

## 检查已安装状态

```bash
openclaw claws status
openclaw claws status incident-triage --json
openclaw doctor
```

`status` 会将已安装的智能体及其记录的工作区、包、MCP 和定时任务来源信息与当前状态进行比较。它会报告未完成的安装、缺失资源和漂移，而不会更改本地状态。`openclaw doctor` 会添加 Claw 专用诊断，用于检测不完整的所有权记录、不安全的托管文件，以及无法通过实时 Gateway 网关清单核实的定时任务。

Claw 来源记录区分两种关系：

- **托管：**Claw 引入并且当前管理该资源。当资源未更改且不存在冲突的所有者时，它是清理候选项。
- **引用：**该资源原本独立存在或由多方共享。移除操作会释放此 Claw 的引用，并默认保留该资源。

这并不是引用计数。普通的插件、技能和智能体命令维持现有行为；Claws 在此基础上增加来源记录和受保护的生命周期操作。

## 更新已安装的 Claw

默认情况下，更新操作使用添加 Claw 时记录的来源。当该来源已移动或测试其他包目录时，请使用 `--from`：

```bash
openclaw claws update incident-triage --dry-run --json
openclaw claws update incident-triage \
  --from ./incident-triage-next \
  --dry-run --json
```

计划会将当前来源记录和实时状态与目标清单进行比较。它会报告智能体、工作区、包、MCP、定时任务和所有权更改，包括能力提升和阻碍因素。能力提升具有单独的机器可读记录；在人类可读输出中，还会显示包含确切脱敏影响的 `!` 行。其中包括解析后的包完整性、安装身份和任何信任警告。移除包声明会释放此 Claw 的依赖边，但更新期间不会卸载工件。最终的精确 `planIntegrity` 确认会同时约束该已披露集合和普通内容更改。主机可以使用相同的记录显示单独的对话框或进行聚合的多智能体审查。使用显式同意应用经过准确审查的计划：

```bash
openclaw claws update incident-triage \
  --yes \
  --plan-integrity <SHA256_FROM_DRY_RUN>
```

OpenClaw 会重新构建计划，并在每次变更前对拥有的状态执行比较并交换。移除包声明会释放依赖边，而不会卸载工件。定时任务更改会重新读取实时调度器定义，并在操作员更改导致漂移时停止。包安装程序、来源配置写入器和 Gateway 网关调度器并非一个事务。如果外部变更后无法证明补偿成功，OpenClaw 会报告错误代码 `update_partial` 和结构化的 `status: partial`，保留不确定的来源记录，并停止执行。检查 `claws status`、受影响的资源和 `openclaw doctor`；然后再次预览，之后再重试或移除任何内容。

## 移除已安装的 Claw

选择清理操作前先预览移除计划：

```bash
openclaw claws remove incident-triage --dry-run --json
openclaw claws remove incident-triage \
  --yes \
  --plan-integrity <SHA256_FROM_DRY_RUN>
```

默认操作会移除符合条件的托管状态并释放引用状态。已修改的文件以及存在其他当前所有者的资源会被保留或阻止移除。清理选项属于计划摘要的一部分；`--yes` 绝不会扩大其范围。全局安装的插件会被保留，同时释放此 Claw 的引用；如果确实要卸载进程范围的插件，请另行使用普通的插件生命周期。

要移除由 Claw 引入、未更改且不存在其他当前所有者的引用，请在预览和应用中都包含 `--remove-unused`。要改为选择确切的被引用资源，请重复使用 `--remove-referenced`：

```bash
openclaw claws remove incident-triage \
  --dry-run \
  --remove-referenced 'plugin:@acme/audit-plugin@2.0.0'
```

仅在审查显示的依赖方、独立所有者和既有来源后使用 `--force-referenced`。它允许在存在这些冲突的情况下清理所选内容，但不会跳过计划完整性同意。

## 导出已安装的智能体

导出会创建一个新的包目录；如果目标已存在或托管状态已发生漂移，则操作失败：

```bash
openclaw claws export incident-triage --out ./incident-triage-export --json
```

结果包含 `package.json`、规范的 `CLAW.md` 和托管工作区
辅助文件。它是一个可移植的 Claw 包，而不是整个实例的备份：不相关的
智能体、凭据、会话和不归其所有的本地状态均不包含在内。

## 命令参考

| 命令                                | 用途                                                   |
| ----------------------------------- | ------------------------------------------------------ |
| `claws inspect <source>`                  | 验证包目录或分组清单。                                 |
| `claws add <source>`                  | 预览或创建一个新的智能体和工作区。                     |
| `claws status [claw-or-agent]`                  | 报告已安装状态、所有权和漂移。                         |
| `claws update <claw-or-agent>`                  | 预览或应用所选来源中的更改。                           |
| `claws remove <claw-or-agent>`                  | 预览或移除智能体及符合条件的资源。                     |
| `claws export <agent> --out <path>`                  | 从已安装的智能体创建可移植包。                         |

使用 `--json` 获取实验性的机器可读输出。

## 另请参阅

- [智能体](/zh-CN/cli/agents)
- [Skills](/zh-CN/tools/skills)
- [插件](/zh-CN/tools/plugin)
- [定时任务](/zh-CN/automation/cron-jobs)
- [MCP 配置](/zh-CN/gateway/configuration-reference#mcp)
