---
read_when:
    - 你需要说明 Agent 工作区或其文件布局
    - 你想备份或迁移 Agent 工作区
sidebarTitle: Agent workspace
summary: Agent 工作区：位置、布局和备份策略
title: Agent 工作区
x-i18n:
    generated_at: "2026-07-26T06:41:49Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: b58ead9079c3dda4bcaec3253f8d55e67e7e554d5c5b87ccfec6b08ec4ba038f
    source_path: concepts/agent-workspace.md
    workflow: 16
---

工作区是智能体的主目录：文件工具和工作区上下文所使用的工作目录。请将其保密，并把它视为记忆。

它与 `~/.openclaw/` 相互独立，后者存储配置、凭据和会话。

<Warning>
工作区是**默认 cwd**，而不是严格的沙箱。工具会基于工作区解析相对路径，但除非启用沙箱隔离，否则绝对路径仍可访问主机上的其他位置。如果需要隔离，请使用 [`agents.defaults.sandbox`](/zh-CN/gateway/sandboxing)（和/或按 Agent 配置的沙箱配置）。

启用沙箱隔离且 `workspaceAccess` 不是 `"rw"` 时，工具会在 `~/.openclaw/sandboxes` 下的沙箱工作区内运行，而不是在你的主机工作区内运行。
</Warning>

## 默认位置

- 默认值：`~/.openclaw/workspace`
- 如果已设置 `OPENCLAW_PROFILE` 且其值不是 `"default"`，默认值将变为 `~/.openclaw/workspace-<profile>`。
- 设置后，`OPENCLAW_WORKSPACE_DIR` 会覆盖上述两项。
- 没有显式工作区的非默认智能体（`agents.entries.*`）会解析到 `<state-dir>/workspace-<agentId>`，而不是共享的默认工作区。

在 `~/.openclaw/openclaw.json` 中覆盖：

```json5
{
  agents: {
    defaults: {
      workspace: "~/.openclaw/workspace",
    },
  },
}
```

按 Agent 覆盖：`agents.entries.*.workspace`。

如果工作区或引导文件不存在，`openclaw onboard`、`openclaw configure` 或 `openclaw setup` 会创建工作区并填充引导文件。

<Note>
沙箱种子复制只接受工作区内的常规文件；解析到源工作区之外的符号链接或硬链接别名会被忽略。
</Note>

如果你已经自行管理工作区文件，请禁用引导文件创建：

```json5
{ agents: { defaults: { skipBootstrap: true } } }
```

## 额外工作区文件夹

较旧的安装可能已创建 `~/openclaw`。保留多个工作区目录可能导致令人困惑的身份验证或状态漂移，因为同一时间只有一个工作区处于活动状态。

<Note>
**建议：**只保留一个活动工作区。如果你不再使用额外的文件夹，请将其归档或移至废纸篓（例如 `trash ~/openclaw`）。如果你有意保留多个工作区，请确保 `agents.defaults.workspace`（或按 Agent 配置的 `workspace` 键）指向活动工作区。
</Note>

## 工作区文件映射

OpenClaw 预期工作区内包含的标准文件：

<AccordionGroup>
  <Accordion title="AGENTS.md - 操作说明">
    智能体的操作说明以及它应如何使用记忆。每个会话开始时加载。适合存放规则、优先级和“行为方式”详情。
  </Accordion>
  <Accordion title="SOUL.md - 人格和语气">
    人格、语气和边界。每个会话都会加载。指南：[SOUL.md 人格指南](/zh-CN/concepts/soul)。
  </Accordion>
  <Accordion title="USER.md - 用户身份">
    用户是谁以及应如何称呼他们。每个会话都会加载。
  </Accordion>
  <Accordion title="IDENTITY.md - 名称、风格和表情符号">
    智能体的名称、风格和表情符号。在引导仪式期间创建或更新。
  </Accordion>
  <Accordion title="TOOLS.md - 本地工具约定">
    关于本地工具和约定的说明。它不控制工具可用性；仅用于提供指导。
  </Accordion>
  <Accordion title="HEARTBEAT.md - Heartbeat 检查清单">
    用于 Heartbeat 运行的可选精简检查清单。请保持简短，以避免消耗过多 token。
  </Accordion>
  <Accordion title="BOOT.md - 启动检查清单">
    Gateway 网关重启时自动运行的可选启动检查清单（启用[内部钩子](/zh-CN/automation/hooks)时）。请保持简短；使用消息工具发送出站消息。
  </Accordion>
  <Accordion title="BOOTSTRAP.md - 首次运行仪式">
    仅执行一次的首次运行仪式。只会为全新的工作区创建。仪式完成后将其删除。
  </Accordion>
  <Accordion title="memory/YYYY-MM-DD.md - 每日记忆日志">
    每日记忆日志（每天一个文件）。建议在会话开始时阅读今天和昨天的日志。
  </Accordion>
  <Accordion title="MEMORY.md - 精选长期记忆（可选）">
    精选长期记忆：持久事实、偏好、决策和简短摘要。将详细日志保存在 `memory/YYYY-MM-DD.md` 中，以便记忆工具按需检索，而不必将其注入每个提示词。仅在私密的主会话中加载 `MEMORY.md`（不在共享或群组上下文中加载）。有关工作流程和自动记忆刷新的信息，请参阅[记忆](/zh-CN/concepts/memory)。
  </Accordion>
  <Accordion title="skills/ - 工作区 Skills（可选）">
    工作区专用 Skills。当名称冲突时，这是该工作区中优先级最高的 Skills 位置，优先于项目智能体 Skills、个人智能体 Skills、托管 Skills、内置 Skills 和 `skills.load.extraDirs`。
  </Accordion>
  <Accordion title="canvas/ - Canvas UI 文件（可选）">
    用于节点显示的 Canvas UI 文件（例如 `canvas/index.html`）。
  </Accordion>
</AccordionGroup>

<Note>
如果缺少引导文件，OpenClaw 会向会话中注入“文件缺失”标记并继续运行。注入时会截断过大的引导文件；可通过 `agents.defaults.bootstrapMaxChars`（默认值：`20000`）和 `agents.defaults.bootstrapTotalMaxChars`（默认值：`60000`）调整限制。`openclaw setup` 可以重新创建缺失的默认文件，而不会覆盖现有文件。
</Note>

## 工作区中不包含的内容

以下内容位于 `~/.openclaw/` 下，不应提交到工作区仓库：

- `~/.openclaw/openclaw.json`（配置）
- `~/.openclaw/state/openclaw.sqlite`（共享工作区设置状态和证明）
- `~/.openclaw/agents/<agentId>/agent/auth-profiles.json`（模型身份验证配置文件：OAuth + API 密钥）
- `~/.openclaw/agents/<agentId>/agent/openclaw-agent.sqlite`（会话行、对话记录和按 Agent 配置的运行时状态）
- `~/.openclaw/agents/<agentId>/agent/codex-home/`（按 Agent 配置的 Codex 运行时账户、配置、Skills、插件和原生线程状态）
- `~/.openclaw/credentials/`（渠道/提供商状态以及旧版 OAuth 导入数据）
- `~/.openclaw/agents/<agentId>/sessions/`（旧版迁移源和归档/支持工件）
- `~/.openclaw/skills/`（托管 Skills）

如果需要迁移会话或配置，请单独复制它们，并将其排除在版本控制之外。

较旧的 OpenClaw 版本会写入工作区附属文件 `openclaw-workspace-state.json`、
`.openclaw/workspace-state.json` 和 `.attested`。当前
运行时仅使用共享 SQLite 数据库保存该状态。如果 Doctor 报告
其中一个文件，请运行 `openclaw doctor --fix`；Doctor 会导入有效的旧版
状态，并且仅在验证数据库行后才删除源文件。

## Git 备份（建议使用私有仓库）

将工作区视为私密记忆。请将其放入**私有** Git 仓库，以便备份和恢复。

在运行 Gateway 网关的计算机上执行以下步骤（工作区就在该计算机上）。

<Steps>
  <Step title="初始化仓库">
    如果已安装 Git，全新的工作区会自动初始化。如果此工作区还不是仓库，请运行：

    ```bash
    cd ~/.openclaw/workspace
    git init
    git add AGENTS.md SOUL.md TOOLS.md IDENTITY.md USER.md HEARTBEAT.md memory/
    git commit -m "Add agent workspace"
    ```

  </Step>
  <Step title="添加私有远程仓库">
    <Tabs>
      <Tab title="GitHub Web UI">
        1. 在 GitHub 上创建一个新的**私有**仓库。
        2. 不要使用 README 初始化（避免合并冲突）。
        3. 复制 HTTPS 远程 URL。
        4. 添加远程仓库并推送：

        ```bash
        git branch -M main
        git remote add origin <https-url>
        git push -u origin main
        ```
      </Tab>
      <Tab title="GitHub CLI (gh)">
        ```bash
        gh auth login
        gh repo create openclaw-workspace --private --source . --remote origin --push
        ```
      </Tab>
      <Tab title="GitLab Web UI">
        1. 在 GitLab 上创建一个新的**私有**仓库。
        2. 不要使用 README 初始化（避免合并冲突）。
        3. 复制 HTTPS 远程 URL。
        4. 添加远程仓库并推送：

        ```bash
        git branch -M main
        git remote add origin <https-url>
        git push -u origin main
        ```
      </Tab>
    </Tabs>

  </Step>
  <Step title="持续更新">
    ```bash
    git status
    git add .
    git commit -m "Update memory"
    git push
    ```
  </Step>
</Steps>

## 不要提交密钥

<Warning>
即使是私有仓库，也应避免在工作区中存储密钥：

- API 密钥、OAuth token、密码或私密凭据。
- `~/.openclaw/` 下的任何内容。
- 聊天或敏感附件的原始转储。

如果必须存储敏感引用，请使用占位符，并将真正的密钥保存在其他位置（密码管理器、环境变量或 `~/.openclaw/`）。
</Warning>

建议的 `.gitignore` 初始内容：

```gitignore
.DS_Store
.env
**/*.key
**/*.pem
**/secrets*
```

## 将工作区迁移到新计算机

<Steps>
  <Step title="克隆仓库">
    将仓库克隆到所需路径（默认路径为 `~/.openclaw/workspace`）。
  </Step>
  <Step title="更新配置">
    在 `~/.openclaw/openclaw.json` 中将 `agents.defaults.workspace` 设置为该路径。
  </Step>
  <Step title="填充缺失文件">
    运行 `openclaw setup --workspace <path>` 以填充所有缺失文件。
  </Step>
  <Step title="复制会话（可选）">
    如果需要会话，请单独从旧计算机复制 `~/.openclaw/agents/<agentId>/agent/openclaw-agent.sqlite`。
    仅当还需要旧版迁移输入或归档/支持工件时，才复制 `~/.openclaw/agents/<agentId>/sessions/`。
  </Step>
</Steps>

## 高级说明

- 多智能体路由可通过 `agents.entries.*.workspace` 为每个智能体使用不同的工作区。有关路由配置，请参阅[频道路由](/zh-CN/channels/channel-routing)。
- 如果启用了 `agents.defaults.sandbox`，非主会话可以使用 `agents.defaults.sandbox.workspaceRoot` 下的按会话沙箱工作区。

## 相关内容

- [Heartbeat](/zh-CN/gateway/heartbeat) - HEARTBEAT.md 工作区文件
- [沙箱隔离](/zh-CN/gateway/sandboxing) - 沙箱隔离环境中的工作区访问
- [会话](/zh-CN/concepts/session) - 会话存储路径
- [常设指令](/zh-CN/automation/standing-orders) - 工作区文件中的持久指令
