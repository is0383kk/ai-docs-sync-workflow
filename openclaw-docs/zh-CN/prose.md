---
read_when:
    - 你想要运行或编写 `.prose` 工作流文件
    - 你想要启用 OpenProse 插件
    - 你需要了解 OpenProse 如何映射到 OpenClaw 原语
sidebarTitle: OpenProse
summary: OpenProse 是一种面向多智能体 AI 会话、以 Markdown 为先的工作流格式。在 OpenClaw 中，它以插件形式提供，包含 `/prose` 斜杠命令和 Skills 包。
title: OpenProse
x-i18n:
    generated_at: "2026-07-26T06:19:29Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 8b04eb23bf827fbec6db11c1e95993e7f6c617451c5f4fda771ad078674c12bc
    source_path: prose.md
    workflow: 16
---

OpenProse 是一种可移植、Markdown 优先的工作流格式，用于编排 AI
会话。在 OpenClaw 中，它作为插件提供，会安装 OpenProse 技能
包和 `/prose` 斜杠命令。程序存放在 `.prose` 文件中，并可
通过显式控制流生成多个子智能体。

<CardGroup cols={3}>
  <Card title="安装" icon="download" href="#install">
    启用 OpenProse 插件并重启 Gateway 网关。
  </Card>
  <Card title="运行程序" icon="play" href="#slash-command">
    使用 `/prose run` 执行 `.prose` 文件或远程程序。
  </Card>
  <Card title="编写程序" icon="pencil" href="#example-parallel-research-and-synthesis">
    使用并行和顺序步骤编写多智能体工作流。
  </Card>
</CardGroup>

## 安装

<Steps>
  <Step title="启用插件">
    OpenProse 已内置，但默认禁用。启用它：

    ```bash
    openclaw plugins enable open-prose
    ```

  </Step>
  <Step title="重启 Gateway 网关">
    ```bash
    openclaw gateway restart
    ```
  </Step>
  <Step title="验证">
    ```bash
    openclaw plugins list | grep prose
    ```

    你应该会看到 `open-prose` 已启用。现在可以在聊天中使用
    `/prose` 技能命令。

  </Step>
</Steps>

在仓库检出目录中，可以直接安装该插件：
`openclaw plugins install ./extensions/open-prose`

## 斜杠命令

OpenProse 将 `/prose` 注册为用户可调用的技能命令：

```text
/prose help
/prose run <file.prose>
/prose run <handle/slug>
/prose run <https://example.com/file.prose>
/prose compile <file.prose>
/prose examples
/prose update
```

`/prose run <handle/slug>` 会解析为 `https://p.prose.md/<handle>/<slug>`。
直接 URL 会使用 `web_fetch` 工具按原样获取。

顶层远程运行是显式的。`.prose` 程序中的远程导入属于
传递性代码依赖：在 OpenProse 获取任何远程 `use` 目标之前，
它会显示解析后的导入列表，并要求操作员针对该次运行准确回复
`approve remote prose imports`。

## 功能

- 通过显式并行机制开展多智能体研究和综合。
- 可重复且审批安全的工作流（代码审查、事件分流、内容流水线）。
- 可在受支持的 Agent Runtimes 中运行的可复用 `.prose` 程序。

## 示例：并行研究和综合

```prose
# 由两个智能体并行执行研究和综合。

input topic: "我们应该研究什么？"

agent researcher:
  model: sonnet
  prompt: "你需要进行全面研究并引用来源。"

agent writer:
  model: opus
  prompt: "你需要撰写简明摘要。"

parallel:
  findings = session: researcher
    prompt: "研究 {topic}。"
  draft = session: writer
    prompt: "总结 {topic}。"

session "将研究结果和草稿合并为最终答案。"
  context: { findings, draft }
```

## OpenClaw 运行时映射

OpenProse 程序映射到 OpenClaw 原语：

| OpenProse 概念            | OpenClaw 工具                                    |
| ------------------------- | ----------------------------------------------- |
| 生成会话 / Task 工具      | `sessions_spawn`                                |
| 文件读取 / 写入           | `read` / `write`                                |
| Web 获取                  | `web_fetch`（需要 POST 时使用 `exec` + curl） |

<Warning>
  如果你的工具允许列表阻止 `sessions_spawn`、`read`、`write` 或
  `web_fetch`，OpenProse 程序将会失败。请检查
  [工具允许列表配置](/zh-CN/gateway/config-tools)。
</Warning>

## 文件位置

OpenProse 将状态保存在工作区的 `.prose/` 下：

```text
.prose/
├── .env                      # 配置（key=value），例如 OPENPROSE_POSTGRES_URL
├── runs/
│   └── {YYYYMMDD}-{HHMMSS}-{random}/
│       ├── program.prose     # 正在运行的程序副本
│       ├── state.md          # 执行状态
│       ├── bindings/
│       ├── imports/          # 嵌套的远程程序运行
│       └── agents/
└── agents/                   # 项目范围的持久化智能体
```

用户级持久化智能体（跨项目共享）位于：

```text
~/.prose/agents/
```

## 状态后端

<AccordionGroup>
  <Accordion title="文件系统（默认）">
    状态写入工作区中的 `.prose/runs/...`。无需额外
    依赖。
  </Accordion>
  <Accordion title="上下文内">
    临时状态保存在上下文窗口中；使用 `--in-context` 选择。
    适用于小型、短期运行的程序。
  </Accordion>
  <Accordion title="sqlite（实验性）">
    使用 `--state=sqlite` 选择。要求 `PATH` 中存在 `sqlite3` 二进制文件
    （缺失时回退到文件系统）；状态存储在
    `.prose/runs/{id}/state.db` 中。
  </Accordion>
  <Accordion title="postgres（实验性）">
    使用 `--state=postgres` 选择。要求 `psql`，并在
    `OPENPROSE_POSTGRES_URL` 中提供连接字符串（在 `.prose/.env` 中设置）。

    <Warning>
      Postgres 凭据会进入子智能体日志。请使用专用的
      最小权限数据库。
    </Warning>

  </Accordion>
</AccordionGroup>

## 安全

将 `.prose` 文件视为代码。运行前请审查这些文件，包括远程
`use` 导入。顶层 `/prose run https://...` 请求是显式的，但
传递性远程导入在获取或执行前需要逐次运行审批。使用 OpenClaw 工具允许列表和审批门槛来控制
副作用。对于确定性且受审批门槛控制的工作流，可与
[Lobster](/zh-CN/tools/lobster) 比较。

## 相关内容

<CardGroup cols={2}>
  <Card title="Skills 参考" href="/zh-CN/tools/skills" icon="puzzle-piece">
    OpenProse 技能包的加载方式以及适用的门槛。
  </Card>
  <Card title="子智能体" href="/zh-CN/tools/subagents" icon="users">
    OpenClaw 的原生多智能体协调层。
  </Card>
  <Card title="文本转语音" href="/zh-CN/tools/tts" icon="volume-high">
    为工作流添加音频输出。
  </Card>
  <Card title="斜杠命令" href="/zh-CN/tools/slash-commands" icon="terminal">
    所有可用的聊天命令，包括 /prose。
  </Card>
</CardGroup>

官方网站：[https://www.prose.md](https://www.prose.md)
