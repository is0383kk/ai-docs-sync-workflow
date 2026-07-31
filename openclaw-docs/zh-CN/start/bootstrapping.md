---
read_when:
    - 了解智能体首次运行时会发生什么
    - 说明引导初始化文件的存放位置
    - 调试新手引导身份设置
sidebarTitle: Bootstrapping
summary: 用于初始化工作区和身份文件的智能体引导流程
title: 智能体引导启动
x-i18n:
    generated_at: "2026-07-26T07:01:09Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: efb47e1a6a86d68aef1aa1662fe9c5def9a4e5b45649b84aeb9060bfcba21a5d
    source_path: start/bootstrapping.md
    workflow: 16
---

引导初始化是首次运行时执行的流程，用于为新的 Agent 工作区填充初始内容，并
引导智能体选择身份。它只运行一次，即新手引导完成后智能体的
第一次真实轮次。

## 会发生什么

首次针对全新工作区（默认 `~/.openclaw/workspace`）运行时，
OpenClaw 会：

- 填充 `AGENTS.md`、`SOUL.md`、`TOOLS.md`、`IDENTITY.md`、`USER.md`、`HEARTBEAT.md` 和 `BOOTSTRAP.md`。
- 让智能体遵循最多三个步骤的诞生流程：询问你想如何
  称呼它，分享一句简短的灵魂/氛围描述，并询问你希望使用
  推荐的最小插件集还是获得最大便利性。
- 将约定的身份保存两次：写入 `IDENTITY.md` 和 `SOUL.md`（智能体
  读取的自身信息），并通过 `openclaw agents set-identity` 保存（渠道
  和 UI 显示的信息）。
- 读取新手引导期间已存储的应用推荐，而不重新扫描。
  官方插件使用 `openclaw plugins install <id>`；第三方 ClawHub
  Skills 仍需明确选择加入。处理选择后，智能体会确认已存储的推荐，
  因此不会再次询问。
- 工作区看起来已配置后删除 `BOOTSTRAP.md`，因此该流程只运行一次。

当 `SOUL.md`、`IDENTITY.md` 或 `USER.md` 中任一项
已不同于其初始模板，或存在 `memory/` 文件夹时，工作区即视为已配置。

<Note>
`BOOTSTRAP.md` 涵盖完整的身份对话。其内容请参阅
[BOOTSTRAP.md 模板](/zh-CN/reference/templates/BOOTSTRAP)。
</Note>

## 嵌入式和本地模型运行

对于嵌入式或本地模型运行，OpenClaw 不会将 `BOOTSTRAP.md` 放入
特权系统上下文。在主要交互式首次运行时，它仍会通过用户提示传递
文件内容，因此无法可靠调用 `read` 工具的模型仍可完成该流程。
如果当前运行无法安全访问工作区，智能体会收到一条简短的受限引导初始化说明，
而不是通用问候语。

## 跳过引导初始化

要在预先填充的工作区中跳过此流程，请运行：

```bash
openclaw onboard --skip-bootstrap
```

## 运行位置

引导初始化始终在 Gateway 网关主机上运行。如果 macOS 应用连接到
远程 Gateway 网关，则工作区及其引导初始化文件位于该远程
计算机上，而不在 Mac 上。

<Note>
当 Gateway 网关在另一台计算机上运行时，请在 Gateway 网关
主机上编辑工作区文件（例如 `user@gateway-host:~/.openclaw/workspace`）。
</Note>

## 相关文档

- macOS 应用新手引导：[新手引导](/zh-CN/start/onboarding)
- 工作区布局：[Agent 工作区](/zh-CN/concepts/agent-workspace)
- 模板内容：[BOOTSTRAP.md 模板](/zh-CN/reference/templates/BOOTSTRAP)
