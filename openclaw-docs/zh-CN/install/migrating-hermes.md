---
read_when:
    - 你正从 Hermes 迁移，并希望保留模型配置、提示词、记忆和 Skills
    - 你想了解 OpenClaw 会自动导入哪些内容，以及哪些内容仅归档保留
    - 你需要一条干净、可脚本化的迁移路径（CI、全新笔记本电脑、自动化）
summary: 通过可预览、可撤销的导入从 Hermes 迁移到 OpenClaw
title: 从 Hermes 迁移
x-i18n:
    generated_at: "2026-07-26T06:45:45Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: f8cdb7a77cfb8ecb0504ccc322b5600c6ed671a8bf9ac866d964fdf4b3494000
    source_path: install/migrating-hermes.md
    workflow: 16
---

内置的 Hermes 迁移提供商会遵循 `HERMES_HOME` 和当前生效的 Hermes 配置文件，并在 macOS/Linux 上回退到 `~/.hermes`，在 Windows 上回退到 `%LOCALAPPDATA%\hermes`。它会在应用前预览每项更改，并在计划和报告中隐去密钥。独立运行 `openclaw migrate` 会写入经过验证的备份；全新新手引导路径会暂存配置、凭据和文件，并仅在导入的推理配置通过验证后发布。显式指定的 `--from` 路径始终优先。

<Note>
导入要求使用全新的 OpenClaw 设置。如果已有本地 OpenClaw 状态，请先重置配置、凭据、会话和工作区，或者在查看计划后直接使用 `openclaw migrate apply hermes` 并配合 `--overwrite`。
</Note>

## 两种导入方式

<Tabs>
  <Tab title="新手引导向导">
    检测当前生效的 Hermes 主目录/配置文件，并在应用前显示预览。

    ```bash
    openclaw onboard --flow import
    ```

    或者指向特定来源：

    ```bash
    openclaw onboard --import-from hermes --import-source ~/.hermes
    ```

  </Tab>
  <Tab title="CLI">
    使用 `openclaw migrate` 进行脚本化或可重复运行。完整参考请参阅 [`openclaw migrate`](/zh-CN/cli/migrate)。

    ```bash
    openclaw migrate hermes --dry-run    # 仅预览
    openclaw migrate apply hermes --yes  # 跳过确认并应用
    ```

    添加 `--from <path>` 可覆盖 Hermes 主目录/配置文件发现结果。

  </Tab>
</Tabs>

## 导入的内容

<AccordionGroup>
  <Accordion title="模型配置">
    - 来自 Hermes `config.yaml` 的默认模型选择。
    - 来自 `model`、`providers` 和 `custom_providers` 的已配置模型提供商及自定义端点，包括当前的 Hermes Chat Completions、Codex Responses 和 Anthropic Messages 传输协议。

  </Accordion>
  <Accordion title="MCP 服务器">
    来自 `mcp_servers` 或 `mcp.servers` 的 MCP 服务器定义，包括禁用状态、超时、并行工具支持、OAuth 权限范围、兼容的 TLS 字段，以及原生/资源/提示词工具策略。字面量环境变量和标头需要获得凭据导入许可。Hermes 独有的生命周期、采样、信息征询、预检、保活、CA 包、受密码保护的客户端密钥以及预注册 OAuth 客户端设置会成为需要手动审查的项目，而不会生成无效的 OpenClaw 配置。
  </Accordion>
  <Accordion title="工作区文件">
    - `SOUL.md` 和 `AGENTS.md` 会复制到 OpenClaw Agent 工作区。
    - `memories/MEMORY.md` 和 `memories/USER.md` 会**追加**到对应的 OpenClaw 记忆文件，而不是覆盖它们。
    - 仅记忆界面的行为有所不同：新手引导的记忆页面和 Control UI 的 Memory 导入页面会将这两个文件复制到 `memory/imports/hermes/` 下以供索引检索，并保持现有工作区记忆不变。

  </Accordion>
  <Accordion title="记忆配置">
    OpenClaw 文件记忆的默认记忆配置。Honcho 等外部记忆提供商会记录为归档项目或需要手动审查的项目，以便你有计划地迁移它们。
  </Accordion>
  <Accordion title="Skills">
    系统会递归发现 `skills/` 下任何位置包含 `SKILL.md` 文件的 Skills，将它们扁平化后复制到 OpenClaw 工作区的技能目录，并同时复制其支持文件。来自 `skills.config` 的各技能配置值会予以保留。
  </Accordion>
  <Accordion title="身份验证凭据">
    交互式 `openclaw migrate` 会在导入身份验证凭据前询问，并默认选中“是”。可导入的内容包括当前 Hermes OpenAI Codex OAuth 条目、OpenCode OpenAI OAuth 和 GitHub Copilot 条目，以及[受支持的 Hermes `.env` 键](/zh-CN/cli/migrate#supported-env-keys)。使用 `--include-secrets` 进行非交互式导入，使用 `--no-auth-credentials` 跳过凭据，或使用新手引导的 `--import-secrets` 标志。导入 Hermes OAuth 后，不要让 Hermes 和 OpenClaw 继续使用同一刷新授权；同时运行二者之前，请在其中一方重新进行身份验证。
  </Accordion>
</AccordionGroup>

## 仅归档的内容

提供商会将以下内容复制到迁移报告目录供手动审查，但**不会**将其加载到正在使用的 OpenClaw 配置或凭据中：

- `plugins/`
- `sessions/`
- `logs/`
- `cron/`
- `mcp-tokens/`
- `plans/`、`workspace/`、`skins/` 和 `kanban/`
- `pairing/` 和 `platforms/` 存储，以及 Gateway 网关路由/进程状态
- `state.db`、`hermes_state.db`、`projects.db`、`response_store.db`、`memory_store.db`、`verification_evidence.db`、`kanban.db` 和 `retaindb_queue.db`

OpenClaw 拒绝自动执行或信任此状态，因为不同系统之间的格式和信任假设可能发生偏移。审查归档后，请手动迁移所需内容。

## 推荐流程

<Steps>
  <Step title="预览计划">
    ```bash
    openclaw migrate hermes --dry-run
    ```

    计划会列出所有将发生的更改，包括冲突、跳过的项目和敏感项目。输出中嵌套的疑似密钥键名会被隐去。

  </Step>
  <Step title="备份并应用">
    ```bash
    openclaw migrate apply hermes --yes
    ```

    OpenClaw 会在应用前创建并验证备份。此非交互式示例仅导入非敏感状态。运行时不添加 `--yes` 可交互回答凭据提示，或者添加 `--include-secrets`，在无人值守运行中包含受支持的凭据。

  </Step>
  <Step title="运行 Doctor">
    ```bash
    openclaw doctor
    ```

    [Doctor](/zh-CN/gateway/doctor) 会重新应用任何待处理的配置迁移，并检查导入期间引入的问题。

  </Step>
  <Step title="重启并验证">
    ```bash
    openclaw gateway restart
    openclaw status
    ```

    确认 Gateway 网关运行状况良好，并且已导入的模型、记忆和 Skills 均已加载。

  </Step>
</Steps>

## 冲突处理

当计划报告冲突（目标位置已有文件或配置值）时，应用操作会拒绝继续。

<Warning>
仅当确实希望替换现有目标时，才使用 `--overwrite` 重新运行。提供商仍可能在迁移报告目录中为被覆盖的文件写入项目级备份。
</Warning>

在全新安装中很少出现冲突。通常是在对已经包含用户编辑内容的设置重新运行导入时才会出现。

如果应用过程中出现冲突（例如配置文件发生意外的竞态条件），该项目会报告为冲突，而彼此独立的文件、Skills、凭据、归档和配置条目会继续处理。解决冲突项目后重新运行导入；相同的记忆导入具有幂等性。

## 密钥

交互式 `openclaw migrate` 会询问是否导入检测到的身份验证凭据，并默认选中“是”。

- 接受后会导入当前 Hermes OpenAI Codex OAuth 条目、OpenCode OpenAI OAuth 和 GitHub Copilot 条目，以及[受支持的 `.env` 键](/zh-CN/cli/migrate#supported-env-keys)。
- 使用 `--no-auth-credentials` 或在提示中回答“否”，可以仅导入非敏感状态。
- 使用 `--include-secrets` 可在无人值守的 `--yes` 运行中导入凭据。
- 使用新手引导向导的 `--import-secrets` 标志可从向导导入凭据。

## 用于自动化的 JSON 输出

```bash
openclaw migrate hermes --dry-run --json
openclaw migrate apply hermes --json --yes
```

使用 `--json` 且不使用 `--yes` 时，应用操作会输出计划但不修改状态——这是 CI 和共享脚本最安全的模式。

## 故障排查

<AccordionGroup>
  <Accordion title="应用因冲突而拒绝继续">
    检查计划输出。每个冲突都会标明源路径和现有目标。逐项决定是跳过、编辑目标，还是使用 `--overwrite` 重新运行。
  </Accordion>
  <Accordion title="Hermes 位于 ~/.hermes 之外">
    传入 `--from /actual/path`（CLI）或 `--import-source /actual/path`（新手引导）。
  </Accordion>
  <Accordion title="新手引导拒绝导入到现有设置">
    新手引导导入要求使用全新设置。可以重置状态并重新进行新手引导，也可以直接使用 `openclaw migrate apply hermes`，它支持 `--overwrite` 和显式备份控制。
  </Accordion>
  <Accordion title="API 密钥未导入">
    仅当你接受凭据提示时，交互式 `openclaw migrate` 才会导入 API 密钥。非交互式 `--yes` 运行需要 `--include-secrets`；新手引导导入需要 `--import-secrets`。系统仅识别[受支持的 `.env` 键](/zh-CN/cli/migrate#supported-env-keys)，其他 `.env` 变量会被忽略。
  </Accordion>
</AccordionGroup>

## 相关内容

- [`openclaw migrate`](/zh-CN/cli/migrate)：完整 CLI 参考、插件契约和 JSON 结构。
- [新手引导](/zh-CN/cli/onboard)：向导流程和非交互式标志。
- [迁移](/zh-CN/install/migrating)：在计算机之间迁移 OpenClaw 安装。
- [Doctor](/zh-CN/gateway/doctor)：迁移后的健康检查。
- [Agent 工作区](/zh-CN/concepts/agent-workspace)：`SOUL.md`、`AGENTS.md` 和记忆文件的存放位置。
