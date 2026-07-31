---
read_when:
    - 解答常见的设置、安装、新手引导或运行时支持问题
    - 在深入调试前对用户报告的问题进行分类排查
summary: 关于 OpenClaw 设置、配置和使用的常见问题
title: 常见问题
x-i18n:
    generated_at: "2026-07-26T06:50:17Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 7bddbf851a0e25323aa7e7cfc3882b33cc0d33a2aa223cccf00328af477ab4c4
    source_path: help/faq.md
    workflow: 16
---

快速解答以及针对实际设置（本地开发、VPS、多智能体、OAuth/API 密钥、模型故障转移）的深入故障排查。有关运行时诊断，请参阅[故障排查](/zh-CN/gateway/troubleshooting)。有关完整的配置参考，请参阅[配置](/zh-CN/gateway/configuration)。

## 出现故障时的最初六十秒

<Steps>
  <Step title="快速状态">
    ```bash
    openclaw status
    ```
    快速本地摘要：操作系统 + 更新、Gateway 网关/服务可达性、智能体/会话、提供商配置 + 运行时问题（当 Gateway 网关可达时）。
  </Step>
  <Step title="可粘贴的报告（可安全分享）">
    ```bash
    openclaw status --all
    ```
    只读诊断，包含日志末尾内容（令牌已脱敏）。
  </Step>
  <Step title="守护进程 + 端口状态">
    ```bash
    openclaw gateway status
    ```
    显示监督程序运行时与 RPC 可达性、探测目标 URL，以及服务可能使用的配置。
  </Step>
  <Step title="深度探测">
    ```bash
    openclaw status --deep
    ```
    实时 Gateway 健康探测，包括支持时的渠道探测（需要可达的 Gateway 网关）。请参阅[健康状态](/zh-CN/gateway/health)。
  </Step>
  <Step title="跟踪最新日志">
    ```bash
    openclaw logs --follow
    ```
    如果 RPC 不可用，请改用：
    ```bash
    tail -f "/tmp/openclaw/openclaw-$(date +%F).log"
    # 命名配置文件示例：
    tail -f "/tmp/openclaw/openclaw-dev-$(date +%F).log"
    ```
    文件日志与服务日志相互独立；请参阅[日志](/zh-CN/logging)和[故障排查](/zh-CN/gateway/troubleshooting)。
  </Step>
  <Step title="运行 Doctor（修复）">
    ```bash
    openclaw doctor
    ```
    修复/迁移配置和状态，然后运行健康检查。请参阅 [Doctor](/zh-CN/gateway/doctor)。
  </Step>
  <Step title="Gateway 快照（仅限 WS）">
    ```bash
    openclaw health --json
    openclaw health --verbose   # 出错时显示目标 URL + 配置路径
    ```
    向正在运行的 Gateway 网关请求完整快照。请参阅[健康状态](/zh-CN/gateway/health)。
  </Step>
</Steps>

## 快速开始和首次运行设置

首次运行问答——安装、新手引导、身份验证路由、订阅、初始故障——请参阅[首次运行常见问题](/zh-CN/help/faq-first-run)。

## OpenClaw 是什么？

<AccordionGroup>
  <Accordion title="用一段话说明 OpenClaw 是什么？">
    OpenClaw 是一款在你自己的设备上运行的个人 AI 助手。它可以在你已使用的消息平台（Discord、Google Chat、iMessage、Mattermost、Signal、Slack、Telegram、WebChat、WhatsApp，以及 QQ Bot 等内置渠道插件）中回复，也可以在支持的平台上提供语音功能和实时 Canvas。**Gateway 网关**是始终在线的控制平面；助手本身才是产品。
  </Accordion>

  <Accordion title="价值主张">
    OpenClaw 不“只是一个 Claude 包装器”。它是一个**本地优先的控制平面**，可在**你自己的硬件**上运行功能强大的助手，并能从你已使用的聊天应用访问；它提供有状态会话、记忆和工具，而无需将你的工作流交给托管式 SaaS。

    - **你的设备，你的数据**：可在任何所需位置（Mac、Linux、VPS）运行 Gateway 网关，并将工作区和会话历史记录保留在本地。
    - **真实渠道，而非 Web 沙箱**：支持 Discord/iMessage/Signal/Slack/Telegram/WhatsApp 等，还可在支持的平台上使用移动端语音和 Canvas。
    - **不受模型限制**：使用 Anthropic、MiniMax、OpenAI、OpenRouter 等，并支持按智能体路由和故障转移。
    - **仅本地选项**：运行本地模型，使所有数据都能留在你的设备上。
    - **多智能体路由**：可按渠道、账户或任务划分不同智能体，每个智能体都有自己的工作区和默认设置。
    - **开源且可定制**：无需受制于供应商，即可检查、扩展和自行托管。

    文档：[Gateway 网关](/zh-CN/gateway)、[渠道](/zh-CN/channels)、[多智能体](/zh-CN/concepts/multi-agent)、[记忆](/zh-CN/concepts/memory)。

  </Accordion>

  <Accordion title="我刚完成设置，首先应该做什么？">
    适合入门的项目：构建网站（WordPress、Shopify 或静态网站）；制作移动应用原型（大纲、界面、API 计划）；整理文件和文件夹；连接 Gmail，并自动生成摘要或跟进事项。

    它可以处理大型任务，但将任务拆分为多个阶段并使用子智能体并行处理时效果最佳。

  </Accordion>

  <Accordion title="OpenClaw 最常见的五种日常用例是什么？">
    - **个人简报**：汇总收件箱、日历以及你关注的新闻。
    - **研究和起草**：快速研究、生成摘要，以及撰写电子邮件或文档初稿。
    - **提醒和跟进**：由定时任务或 Heartbeat 驱动的提醒和检查清单。
    - **浏览器自动化**：填写表单、收集数据、重复执行 Web 任务。
    - **跨设备协调**：从手机发送任务，让 Gateway 网关在服务器上运行任务，然后在聊天中接收结果。

  </Accordion>

  <Accordion title="OpenClaw 能否帮助 SaaS 进行潜在客户开发、推广、广告和博客创作？">
    可以，用于**研究、筛选和起草**：扫描网站、创建候选名单、汇总潜在客户信息，以及撰写推广内容或广告文案草稿。

    对于**推广或广告投放**，应确保有人参与审核。避免发送垃圾信息，遵守当地法律和平台政策，并在发送前审核所有内容。让 OpenClaw 起草，由你批准。

    文档：[安全](/zh-CN/gateway/security)。

  </Accordion>

  <Accordion title="与 Claude Code 相比，OpenClaw 在 Web 开发方面有哪些优势？">
    OpenClaw 是**个人助手**和协调层，而不是 IDE 的替代品。要在代码仓库中获得最快的直接编码循环，请使用 Claude Code 或 Codex。要获得持久记忆、跨设备访问和工具编排能力，请使用 OpenClaw。

    - 跨会话持久保留记忆和工作区。
    - 多平台访问（Telegram、WhatsApp、TUI、WebChat）。
    - 工具编排（浏览器、文件、调度、Hooks）。
    - 始终在线的 Gateway 网关（在 VPS 上运行，可从任何位置交互）。
    - 用于本地浏览器/屏幕/摄像头/Exec 的节点。

    案例展示：[https://openclaw.ai/showcase](https://openclaw.ai/showcase)。

  </Accordion>
</AccordionGroup>

## Skills 和自动化

<AccordionGroup>
  <Accordion title="如何在不弄脏代码仓库的情况下自定义 Skills？">
    使用托管覆盖，而不要编辑代码仓库中的副本。将更改放入 `~/.openclaw/skills/<name>/SKILL.md`（或通过 `~/.openclaw/openclaw.json` 中的 `skills.load.extraDirs` 添加文件夹）。优先级：`<workspace>/skills` -> `<workspace>/.agents/skills` -> `~/.agents/skills` -> `~/.openclaw/skills` -> 内置 -> `skills.load.extraDirs`，因此托管覆盖可以优先于内置 Skills，而无需修改 git。若要进行全局安装但只让部分智能体可见，请将共享副本保留在 `~/.openclaw/skills` 中，并通过 `agents.defaults.skills` / `agents.entries.*.skills` 控制可见性。只有值得提交到上游的修改才应针对代码仓库副本发起 PR。
  </Accordion>

  <Accordion title="可以从自定义文件夹加载 Skills 吗？">
    可以：通过 `~/.openclaw/openclaw.json` 中的 `skills.load.extraDirs` 添加目录（在上述顺序中优先级最低）。`clawhub` 默认安装到 `./skills`，OpenClaw 会在下一个会话中将其视为 `<workspace>/skills`。若要将可见性限制为特定智能体，请配合使用 `agents.defaults.skills` 或 `agents.entries.*.skills`。
  </Accordion>

  <Accordion title="如何针对不同任务使用不同的模型或设置？">
    支持的模式：

    - **定时任务**：隔离任务可以为每个任务设置 `model` 覆盖。
    - **智能体**：将任务路由到使用不同默认模型、思考级别和流式参数的独立智能体。
    - **按需切换**：`/model` 可随时切换当前会话的模型。

    示例——相同模型，不同的智能体设置：

    ```json5
    {
      agents: {
        list: [
          {
            id: "coder",
            model: "xiaomi/mimo-v2.5-pro",
            thinkingDefault: "high",
            params: { temperature: 0.1 },
          },
          {
            id: "chat",
            model: "xiaomi/mimo-v2.5-pro",
            thinkingDefault: "off",
            params: { temperature: 0.8 },
          },
        ],
      },
    }
    ```

    将共享的模型默认值放入 `agents.defaults.models["provider/model"].params`，然后将智能体专属覆盖放入扁平的 `agents.entries.*.params`。不要在嵌套的 `agents.entries.*.models["provider/model"].params` 下重复添加同一模型；该路径用于按智能体配置模型目录和运行时覆盖。

    请参阅[定时任务](/zh-CN/automation/cron-jobs)、[多智能体路由](/zh-CN/concepts/multi-agent)、[配置](/zh-CN/gateway/config-agents)、[斜杠命令](/zh-CN/tools/slash-commands)。

  </Accordion>

  <Accordion title="Bot 在执行繁重工作时卡住。如何卸载这类任务？">
    对耗时或并行任务使用**子智能体**：它们在自己的会话中运行、返回摘要，并让主聊天保持响应。让 Bot“为此任务创建一个子智能体”，或使用 `/subagents`。使用 `/status` 查看 Gateway 网关当前是否繁忙。

    长任务和子智能体都会消耗令牌；如果需要考虑成本，请通过 `agents.defaults.subagents.model` 为子智能体设置更便宜的模型。

    文档：[子智能体](/zh-CN/tools/subagents)、[后台任务](/zh-CN/automation/tasks)。

  </Accordion>

  <Accordion title="Discord 上绑定到线程的子智能体会话如何工作？">
    将 Discord 线程绑定到子智能体或会话目标，使该线程中的后续消息继续进入绑定的会话。

    - 使用 `sessions_spawn` 创建，并设置 `thread: true`（也可设置 `mode: "session"`，以支持持久后续交互）。
    - 或者使用 `/focus <target>` 手动绑定。
    - `/agents` 用于检查绑定状态。
    - `/session idle <duration|off>` 和 `/session max-age <duration|off>` 用于控制自动取消聚焦。
    - `/unfocus` 用于解除线程绑定。

    配置：`session.threadBindings.enabled`（全局开关）、`session.threadBindings.idleHours`（默认值为 `24`，`0` 表示禁用）、`session.threadBindings.maxAgeHours`（默认值为 `0`，即无硬性上限），以及用于创建时自动绑定的 `session.threadBindings.spawnSessions`（默认值为 `true`）。

    文档：[子智能体](/zh-CN/tools/subagents)、[Discord](/zh-CN/channels/discord)、[配置参考](/zh-CN/gateway/configuration-reference)、[斜杠命令](/zh-CN/tools/slash-commands)。

  </Accordion>

  <Accordion title="子智能体已完成，但完成更新发送到了错误的位置或从未发布。我应该检查什么？">
    检查解析后的请求方路由：

    - 在存在绑定线程或对话路由时，完成模式的子智能体交付会优先使用该路由。
    - 如果完成来源只携带渠道信息，OpenClaw 会回退到请求方会话中存储的路由（`lastChannel` / `lastTo` / `lastAccountId`），因此仍可能成功直接交付。
    - 既没有绑定路由，也没有可用的已存储路由：直接交付可能失败，结果会回退到排队的会话交付，而不是立即发布。
    - 无效或过时的目标也可能导致回退到队列或最终交付失败。
    - 如果子智能体最后一条可见的助手回复恰好是 `NO_REPLY` / `no_reply` 或 `ANNOUNCE_SKIP`，OpenClaw 会有意抑制通知，以免发布此前已过时的进度。

    调试：`openclaw tasks show <lookup>`，其中 `<lookup>` 是任务 ID、运行 ID 或会话键。

    文档：[子智能体](/zh-CN/tools/subagents)、[后台任务](/zh-CN/automation/tasks)、[会话工具](/zh-CN/concepts/session-tool)。

  </Accordion>

  <Accordion title="定时任务或提醒未触发。我应该检查什么？">
    定时任务在 Gateway 网关进程内运行；如果 Gateway 网关未持续运行，则不会触发。

    - 确认已启用定时任务（`cron.enabled`），且未设置 `OPENCLAW_SKIP_CRON`。
    - 确认 Gateway 网关正在 24/7 运行（未休眠/重启）。
    - 验证任务时区（`--tz` 与主机时区）。

    调试：
    ```bash
    openclaw cron run <jobId>
    openclaw cron runs --id <jobId> --limit 50
    ```

    文档：[定时任务](/zh-CN/automation/cron-jobs)、[自动化](/zh-CN/automation)。

  </Accordion>

  <Accordion title="定时任务已触发，但未向渠道发送任何内容。为什么？">
    检查投递模式：

    - `--no-deliver` / `delivery.mode: "none"`：预期不会由运行器进行后备发送。
    - 通知目标缺失或无效（`channel` / `to`）：运行器跳过了出站投递。
    - 渠道身份验证失败（`unauthorized`、`Forbidden`）：运行器尝试了投递，但凭据阻止了投递。
    - 静默的隔离结果（仅含 `NO_REPLY` / `no_reply`）会被视为有意不投递，因此排队的后备投递也会被抑制。

    对于隔离的定时任务，当存在聊天路由时，智能体仍可使用 `message` 工具直接发送。`--announce` 仅控制运行器对智能体尚未自行发送的最终文本进行后备投递。

    调试：
    ```bash
    openclaw cron runs --id <jobId> --limit 50
    openclaw tasks show <lookup>
    ```

    文档：[定时任务](/zh-CN/automation/cron-jobs)、[后台任务](/zh-CN/automation/tasks)。

  </Accordion>

  <Accordion title="为什么隔离的定时任务运行会切换模型或重试一次？">
    这是实时模型切换路径，并非重复调度。当活动运行抛出 `LiveSessionModelSwitchError` 时，隔离的定时任务会持久化运行时模型交接并重试，在重试前保留切换后的提供商/模型（以及任何切换后的身份验证配置文件覆盖）。

    模型选择优先级：首先是 Gmail 钩子的模型覆盖（`hooks.gmail.model`），然后是每个任务的 `model`，接着是已存储的定时任务会话模型覆盖，最后是常规的智能体/默认模型选择。

    重试循环以初始尝试加 2 次切换重试为上限；此后定时任务会中止，而不是无限循环。

    调试：
    ```bash
    openclaw cron runs --id <jobId> --limit 50
    ```

    文档：[定时任务](/zh-CN/automation/cron-jobs)、[cron CLI](/zh-CN/cli/cron)。

  </Accordion>

  <Accordion title="如何在 Linux 上安装 Skills？">
    使用原生 `openclaw skills` 命令，或将 Skills 放入工作区；macOS Skills UI 在 Linux 上不可用。可在 [https://clawhub.ai](https://clawhub.ai) 浏览 Skills。

    ```bash
    openclaw skills search "calendar"
    openclaw skills search --limit 20
    openclaw skills install @owner/<skill-slug>
    openclaw skills install @owner/<skill-slug> --version <version>
    openclaw skills install @owner/<skill-slug> --force
    openclaw skills install @owner/<skill-slug> --global
    openclaw skills update --all
    openclaw skills update --all --global
    openclaw skills list --eligible
    openclaw skills check
    ```

    默认情况下，原生 `openclaw skills install` 会写入活动工作区的 `skills/` 目录。添加 `--global` 可将其安装到共享的托管 Skills 目录，供所有本地智能体使用。仅在发布或同步你自己的 Skills 时，才安装单独的 `clawhub` CLI。使用 `agents.defaults.skills` 或 `agents.entries.*.skills` 可限定哪些智能体能看到共享 Skills。

  </Accordion>

  <Accordion title="OpenClaw 能否按计划运行任务，或持续在后台运行任务？">
    可以，通过 Gateway 网关调度器实现：

    - **定时任务**：用于计划任务或重复任务（重启后仍保留）。
    - **Heartbeat**：用于主会话的定期检查。
    - **隔离任务**：用于发布摘要或向聊天投递内容的自主智能体。

    文档：[定时任务](/zh-CN/automation/cron-jobs)、[自动化](/zh-CN/automation)、[Heartbeat](/zh-CN/gateway/heartbeat)。

  </Accordion>

  <Accordion title="能否从 Linux 运行仅限 Apple macOS 的 Skills？">
    不能直接运行。macOS Skills 受 `metadata.openclaw.os` 和所需二进制文件限制，并且只有在 **Gateway 网关主机**上符合条件时才会加载。在 Linux 上，除非覆盖此限制，否则仅限 `darwin` 的 Skills（`apple-notes`、`apple-reminders`、`things-mac`）不会加载。

    支持以下三种模式：

    **选项 A - 在 Mac 上运行 Gateway 网关（最简单）**。在存在 macOS 二进制文件的主机上运行 Gateway 网关，然后从 Linux 以[远程模式](#gateway-ports-already-running-and-remote-mode)或通过 Tailscale 连接。由于 Gateway 网关主机是 macOS，Skills 会正常加载。

    **选项 B - 使用 macOS 节点（无需 SSH）**。在 Linux 上运行 Gateway 网关，配对一个 macOS 节点（菜单栏应用），并在 Mac 上将 **Node Run Commands** 设置为 "Always Ask" 或 "Always Allow"。当节点上存在所需二进制文件时，OpenClaw 会将仅限 macOS 的 Skills 视为符合条件；智能体通过 `nodes` 工具运行它们。使用 "Always Ask" 时，在提示中批准 "Always Allow" 会将该命令添加到允许列表。

    **选项 C - 通过 SSH 代理 macOS 二进制文件（高级）**。继续在 Linux 上运行 Gateway 网关，但让所需 CLI 二进制文件解析为在 Mac 上运行的 SSH 包装脚本，然后覆盖 Skill 以允许 Linux，使其保持符合条件。

    1. 为二进制文件创建 SSH 包装脚本（示例：用于 Apple Notes 的 `memo`）：
       ```bash
       #!/usr/bin/env bash
       set -euo pipefail
       exec ssh -T user@mac-host /opt/homebrew/bin/memo "$@"
       ```
    2. 将包装脚本放入 Linux 主机上的 `PATH`（例如 `~/bin/memo`）。
    3. 覆盖 Skill 元数据（在工作区或 `~/.openclaw/skills` 中）以允许 Linux：
       ```markdown
       ---
       name: apple-notes
       description: 通过 macOS 上的 memo CLI 管理 Apple Notes。
       metadata: { "openclaw": { "os": ["darwin", "linux"], "requires": { "bins": ["memo"] } } }
       ---
       ```
    4. 启动新会话，以刷新 Skills 快照。

  </Accordion>

  <Accordion title="你们是否提供 Notion 或 HeyGen 集成？">
    目前未内置。可选方案：

    - **自定义 Skill / 插件**：最适合可靠的 API 访问（两者均提供 API）。
    - **浏览器自动化**：无需编写代码即可工作，但速度较慢且更脆弱。

    对于代理机构式的每客户上下文：为每位客户保留一个 Notion 页面（上下文 + 偏好设置 + 当前工作），并要求智能体在会话开始时获取该页面。

    如需原生集成，请提交功能请求，或针对这些 API 构建 Skill。

    ```bash
    openclaw skills install @owner/<skill-slug>
    openclaw skills update --all
    ```

    原生安装会放入活动工作区的 `skills/` 目录；使用 `--global` 可供所有本地智能体使用，或配置 `agents.defaults.skills` / `agents.entries.*.skills` 以限制可见性。某些 Skills 需要通过 Homebrew 安装的二进制文件；在 Linux 上，这意味着 Linuxbrew。

    请参阅 [Skills](/zh-CN/tools/skills)、[Skills 配置](/zh-CN/tools/skills-config)、[ClawHub](/tools/clawhub)。

  </Accordion>

  <Accordion title="如何让 OpenClaw 使用我现有的已登录 Chrome？">
    使用内置的 `user` 浏览器配置文件，它通过 Chrome DevTools MCP 附加：

    ```bash
    openclaw browser --browser-profile user tabs
    openclaw browser --browser-profile user snapshot
    ```

    如需自定义名称，请创建显式 MCP 配置文件：

    ```bash
    openclaw browser create-profile --name chrome-live --driver existing-session
    openclaw browser --browser-profile chrome-live tabs
    ```

    这可以使用本地主机浏览器或已连接的浏览器节点。如果 Gateway 网关在其他位置运行，请在浏览器所在计算机上运行节点主机，或改用远程 CDP。

    与托管的 `openclaw` 配置文件相比，`existing-session` / `user` 配置文件当前存在以下限制：

    - `click`、`type`、`hover`、`scrollIntoView`、`drag` 和 `select` 需要快照引用，而不是 CSS 选择器。
    - 上传钩子需要 `ref` 或 `inputRef`，每次一个文件，不支持 CSS `element`。
    - `responsebody`、PDF 导出、下载拦截和批量操作仍需要托管浏览器路径。

    完整比较请参阅[浏览器](/zh-CN/tools/browser#existing-session-via-chrome-devtools-mcp)。

  </Accordion>
</AccordionGroup>

## 沙箱隔离和记忆

<AccordionGroup>
  <Accordion title="是否有专门的沙箱隔离文档？">
    有：[沙箱隔离](/zh-CN/gateway/sandboxing)。有关 Docker 特定设置（在 Docker 中运行完整 Gateway 网关或使用沙箱镜像），请参阅 [Docker](/zh-CN/install/docker)。
  </Accordion>

  <Accordion title="Docker 功能似乎受限——如何启用完整功能？">
    默认镜像以安全性为先，并以 `node` 用户身份运行，因此不包含系统软件包、Homebrew 和内置浏览器。如需更完整的设置：

    - 使用 `OPENCLAW_HOME_VOLUME` 持久化 `/home/node`，使缓存得以保留。
    - 使用 `OPENCLAW_IMAGE_APT_PACKAGES` 将系统依赖项构建到镜像中。
    - 通过内置 CLI 安装 Playwright 浏览器：`node /app/node_modules/playwright-core/cli.js install chromium`。
    - 设置 `PLAYWRIGHT_BROWSERS_PATH` 并持久化该路径。

    文档：[Docker](/zh-CN/install/docker)、[浏览器](/zh-CN/tools/browser)。

  </Accordion>

  <Accordion title="能否使用一个智能体让私信保持私密，同时让群组公开并进行沙箱隔离？">
    可以，前提是私密流量为**私信**，公开流量为**群组**。设置 `agents.defaults.sandbox.mode: "non-main"`，使群组/渠道会话（非主键）在配置的沙箱后端中运行，而主私信会话仍在主机上运行。启用沙箱隔离后，Docker 是默认后端。通过 `tools.sandbox.tools` 限制沙箱隔离会话中可用的工具。

    设置演练：[群组：个人私信 + 公开群组](/zh-CN/channels/groups#pattern-personal-dms-public-groups-single-agent)。关键参考：[Gateway 配置](/zh-CN/gateway/config-agents#agentsdefaultssandbox)。

  </Accordion>

  <Accordion title="如何将主机文件夹绑定到沙箱中？">
    将 `agents.defaults.sandbox.docker.binds` 设置为 `["host:container:mode"]`（例如 `"/home/user/src:/src:ro"`）。全局绑定和每智能体绑定会合并；当 `scope: "shared"` 时，会忽略每智能体绑定。任何敏感内容都应使用 `:ro`；绑定会绕过沙箱文件系统边界。

    OpenClaw 会同时根据规范化路径以及通过最深层现有祖先解析出的规范路径验证绑定源，因此即使最终路径段尚不存在，通过符号链接父目录逃逸的尝试也会以关闭方式失败。

    请参阅[沙箱隔离](/zh-CN/gateway/sandboxing#custom-bind-mounts)和[沙箱、工具策略和提升权限](/zh-CN/gateway/sandbox-vs-tool-policy-vs-elevated#bind-mounts-security-quick-check)。

  </Accordion>

  <Accordion title="记忆如何工作？">
    OpenClaw 的记忆是智能体工作区中的 Markdown 文件：每日笔记位于 `memory/YYYY-MM-DD.md`，整理后的长期笔记位于 `MEMORY.md`（仅限主会话/私密会话）。

    OpenClaw 还会在压缩对话摘要之前静默执行**压缩前记忆刷新**，提醒模型先写入持久笔记。仅当工作区可写时才会运行（只读沙箱会跳过）；可使用 `agents.defaults.compaction.memoryFlush.enabled: false` 禁用。请参阅[记忆](/zh-CN/concepts/memory)。

  </Accordion>

  <Accordion title="记忆总是忘记内容。如何让它记牢？">
    要求 Bot **将事实写入记忆**：长期笔记写入 `MEMORY.md`，短期上下文写入 `memory/YYYY-MM-DD.md`。提醒模型存储记忆通常可以解决此问题。如果仍然遗忘，请验证 Gateway 网关每次运行时使用的都是同一工作区。

    文档：[记忆](/zh-CN/concepts/memory)、[Agent 工作区](/zh-CN/concepts/agent-workspace)。

  </Accordion>

  <Accordion title="记忆会永久保留吗？有哪些限制？">
    记忆文件存储在磁盘上，在删除前会一直保留；限制来自你的存储空间，而不是模型。**会话上下文**仍受模型上下文窗口限制，因此长对话可能会被压缩或截断——这正是记忆搜索存在的原因，它只将相关部分重新拉取到上下文中。

    文档：[记忆](/zh-CN/concepts/memory)、[上下文](/zh-CN/concepts/context)。

  </Accordion>

  <Accordion title="语义记忆搜索需要 OpenAI API key 吗？">
    仅当你使用默认提供商 **OpenAI embeddings** 时才需要。Codex OAuth 仅涵盖聊天/补全，**不**授予 embeddings 访问权限，因此使用 Codex 登录（OAuth 或 Codex CLI 登录）不会启用语义记忆搜索。OpenAI embeddings 仍需要真实的 API key（`OPENAI_API_KEY` 或 `models.providers.openai.apiKey`）。

    若要保持本地运行，请设置 `memory.search.provider: "local"`（GGUF/llama.cpp）。其他受支持的提供商包括：Bedrock、DeepInfra、Gemini（`GEMINI_API_KEY` 或 `memory.search.remote.apiKey`）、GitHub Copilot、LM Studio、Mistral、Ollama、OpenAI-compatible 和 Voyage。设置详情请参阅[记忆](/zh-CN/concepts/memory)和[记忆搜索](/zh-CN/concepts/memory-search)。

  </Accordion>
</AccordionGroup>

## 数据在磁盘上的存储位置

<AccordionGroup>
  <Accordion title="与 OpenClaw 一起使用的所有数据都会保存在本地吗？">
    不会：**OpenClaw 自身的状态存储在本地**，但**外部服务仍能看到你发送给它们的内容**。

    - **默认存储在本地**：会话、记忆文件、配置和工作区位于 Gateway 网关主机上（`~/.openclaw` 以及你的工作区目录）。
    - **必然会传输到远程**：发送给模型提供商（Anthropic/OpenAI 等）的消息会进入其 API，聊天平台（Slack/Telegram/WhatsApp 等）也会将消息数据存储在其服务器上。
    - **你可以控制数据足迹**：本地模型会将提示词保留在你的机器上，但渠道流量仍会通过该渠道的服务器。

    相关内容：[Agent 工作区](/zh-CN/concepts/agent-workspace)、[记忆](/zh-CN/concepts/memory)。

  </Accordion>

  <Accordion title="OpenClaw 将数据存储在哪里？">
    所有内容都位于 `$OPENCLAW_STATE_DIR` 下（默认：`~/.openclaw`）：

    | 路径                                                               | 用途                                                            |
    | ------------------------------------------------------------------ | ------------------------------------------------------------------ |
    | `$OPENCLAW_STATE_DIR/openclaw.json`                                 | 主配置（JSON5）                                                 |
    | `$OPENCLAW_STATE_DIR/credentials/oauth.json`                        | 旧版 OAuth 导入（首次使用时复制到身份验证配置文件中）        |
    | `$OPENCLAW_STATE_DIR/agents/<agentId>/agent/auth-profiles.json`     | 身份验证配置文件（OAuth、API key、可选的 `keyRef`/`tokenRef`）        |
    | `$OPENCLAW_STATE_DIR/secrets.json`                                  | `file` SecretRef 提供商可选的文件支持型机密载荷   |
    | `$OPENCLAW_STATE_DIR/agents/<agentId>/agent/auth.json`              | 旧版兼容文件（已清除静态 `api_key` 条目）        |
    | `$OPENCLAW_STATE_DIR/credentials/`                                  | 提供商状态（例如 `whatsapp/<accountId>/creds.json`）      |
    | `$OPENCLAW_STATE_DIR/agents/`                                       | 每个智能体的状态（agentDir + 旧版/归档会话工件）        |
    | `$OPENCLAW_STATE_DIR/agents/<agentId>/agent/openclaw-agent.sqlite`  | 每个智能体的 SQLite 状态，包括会话行和转录记录      |
    | `$OPENCLAW_STATE_DIR/agents/<agentId>/sessions/`                    | 旧版会话迁移源和归档/支持工件      |

    旧版单智能体路径 `~/.openclaw/agent/*` 由 `openclaw doctor` 迁移。

    你的**工作区**（AGENTS.md、记忆文件、Skills 等）单独存放，通过 `agents.defaults.workspace` 配置（默认：`~/.openclaw/workspace`）。

  </Accordion>

  <Accordion title="AGENTS.md / SOUL.md / USER.md / MEMORY.md 应该存放在哪里？">
    这些文件位于 **Agent 工作区**中，而不是 `~/.openclaw`。

    - **工作区（每个智能体）**：`AGENTS.md`、`SOUL.md`、`IDENTITY.md`、`USER.md`、`MEMORY.md`、`memory/YYYY-MM-DD.md`，以及可选的 `HEARTBEAT.md`。根目录下的小写 `memory.md` 仅作为旧版修复输入；当两者都存在时，`openclaw doctor --fix` 可以将其合并到 `MEMORY.md` 中。
    - **状态目录（`~/.openclaw`）**：配置、渠道/提供商状态、身份验证配置文件、会话、日志、共享 Skills（`~/.openclaw/skills`）。

    默认工作区为 `~/.openclaw/workspace`，可进行配置：

    ```json5
    {
      agents: { defaults: { workspace: "~/.openclaw/workspace" } },
    }
    ```

    如果机器人重启后“忘记”了内容，请确认 Gateway 网关每次启动时都使用同一个工作区（远程模式使用 **Gateway 网关主机上的**工作区，而不是你本地笔记本电脑上的工作区）。

    提示：对于需要持久保留的行为或偏好，应让机器人**将其写入 AGENTS.md 或 MEMORY.md**，而不是依赖聊天历史记录。

    请参阅 [Agent 工作区](/zh-CN/concepts/agent-workspace)和[记忆](/zh-CN/concepts/memory)。

  </Accordion>

  <Accordion title="可以增大 SOUL.md 吗？">
    可以。`SOUL.md` 是注入智能体上下文的工作区引导文件之一。默认的单文件注入限制为 `20000` 个字符；所有文件的引导总预算为 `60000` 个字符。

    更改共享默认值：

    ```json5
    {
      agents: {
        defaults: {
          bootstrapMaxChars: 50000,
          bootstrapTotalMaxChars: 300000,
        },
      },
    }
    ```

    或在 `agents.entries.*.bootstrapMaxChars` / `bootstrapTotalMaxChars` 下覆盖某个智能体的设置。

    使用 `/context` 检查原始大小与注入大小，并确认是否发生了截断。让 `SOUL.md` 专注于语气、立场和个性；将操作规则放入 `AGENTS.md`，将持久事实放入记忆。

    请参阅[上下文](/zh-CN/concepts/context)和[智能体配置](/zh-CN/gateway/config-agents)。

  </Accordion>

  <Accordion title="推荐的备份策略">
    将你的 **Agent 工作区**放入**私有** Git 仓库，并备份到私有位置（例如 GitHub 私有仓库）。这样会捕获记忆以及 AGENTS/SOUL/USER 文件，让你以后能够恢复助手的“思维”。

    **不要**提交 `~/.openclaw` 下的任何内容（凭据、会话、令牌、加密的机密载荷）。如需完整恢复，请分别备份工作区和状态目录。

    文档：[Agent 工作区](/zh-CN/concepts/agent-workspace)。

  </Accordion>

  <Accordion title="如何彻底卸载 OpenClaw？">
    请参阅[卸载](/zh-CN/install/uninstall)。
  </Accordion>

  <Accordion title="智能体可以在工作区之外工作吗？">
    可以。工作区是**默认 cwd** 和记忆锚点，而不是硬性沙箱。相对路径在工作区内解析；除非启用了沙箱隔离，否则绝对路径可以访问主机上的其他位置。若要实现隔离，请使用 [`agents.defaults.sandbox`](/zh-CN/gateway/sandboxing) 或按智能体设置沙箱。若要将某个仓库设为默认工作目录，请将该智能体的 `workspace` 指向仓库根目录——OpenClaw 仓库本身只是源代码，因此除非你有意让智能体在其中工作，否则请将工作区与其分开。

    ```json5
    {
      agents: {
        defaults: {
          workspace: "~/Projects/my-repo",
        },
      },
    }
    ```

  </Accordion>

  <Accordion title="远程模式：会话存储在哪里？">
    会话状态由 **Gateway 网关主机**拥有。在远程模式下，你关心的会话存储位于远程机器上，而不是你的本地笔记本电脑上。请参阅[会话管理](/zh-CN/concepts/session)。
  </Accordion>
</AccordionGroup>

## 配置基础知识

<AccordionGroup>
  <Accordion title="配置采用什么格式？位于哪里？">
    OpenClaw 从 `$OPENCLAW_CONFIG_PATH`（默认：`~/.openclaw/openclaw.json`）读取可选的 **JSON5** 配置。如果文件不存在，它会使用较为安全的默认值，其中默认工作区为 `~/.openclaw/workspace`。
  </Accordion>

  <Accordion title='我设置了 gateway.bind: "lan"（或 "tailnet"），但现在没有任何监听 / UI 显示未经授权'>
    非回环绑定**需要有效的 Gateway 网关身份验证路径**：共享机密身份验证（令牌或密码），或者在正确配置的身份感知反向代理后使用 `gateway.auth.mode: "trusted-proxy"`。

    ```json5
    {
      gateway: {
        bind: "lan",
        auth: {
          mode: "token",
          token: "replace-me",
        },
      },
    }
    ```

    - `gateway.remote.token` / `.password` 本身**不会**启用本地 Gateway 网关身份验证；只有当 `gateway.auth.*` 未设置时，本地调用路径才能使用 `gateway.remote.*` 作为回退。
    - 对于密码身份验证，请设置 `gateway.auth.mode: "password"` 以及 `gateway.auth.password`（或 `OPENCLAW_GATEWAY_PASSWORD`）。
    - 如果通过 SecretRef 显式配置的 `gateway.auth.token` / `.password` 无法解析，则解析会以关闭方式失败（不会用远程回退掩盖问题）。
    - 使用共享机密的 Control UI 设置通过 `connect.params.auth.token` 或 `connect.params.auth.password` 进行身份验证（存储在应用/UI 设置中）。Tailscale Serve 或 `trusted-proxy` 等携带身份的模式改用请求标头——避免将共享机密放入 URL。
    - 使用 `gateway.auth.mode: "trusted-proxy"` 时，同主机回环反向代理需要显式设置 `gateway.auth.trustedProxy.allowLoopback = true`，并在 `gateway.trustedProxies` 中添加回环条目。

  </Accordion>

  <Accordion title="为什么现在 localhost 上也需要令牌？">
    OpenClaw 默认强制执行 Gateway 网关身份验证，包括回环地址。如果未配置显式的身份验证路径，启动时会解析为令牌模式，并为该次启动生成仅限运行时使用的令牌，因此本地 WS 客户端必须进行身份验证。这可以阻止其他本地进程调用 Gateway 网关。

    当客户端需要在重启后继续使用稳定的机密时，请显式配置 `gateway.auth.token`、`gateway.auth.password`、`OPENCLAW_GATEWAY_TOKEN` 或 `OPENCLAW_GATEWAY_PASSWORD`。你也可以选择密码模式，或为身份感知反向代理选择 `trusted-proxy`。若要开放回环访问，请显式设置 `gateway.auth.mode: "none"`。`openclaw doctor --generate-gateway-token` 可随时生成令牌。

  </Accordion>

  <Accordion title="更改配置后必须重启吗？">
    Gateway 网关会监视配置并支持热重载：`gateway.reload.mode: "hybrid"`（默认）会热应用安全更改，并在遇到关键更改时重启。也支持 `hot`、`restart` 和 `off`。大多数 `tools.*`、`agents.*` 策略、`session.*` 和 `messages.*` 更改会立即生效，完全不需要执行重载操作；`gateway.*` 绑定/端口更改需要重启。
  </Accordion>

  <Accordion title="如何启用 Web 搜索（和 Web 获取）？">
    `web_fetch` 无需 API key 即可工作。`web_search` 取决于你选择的提供商：

    | 提供商 | 无需密钥 | 环境变量 |
    | --- | --- | --- |
    | Brave | 否 | `BRAVE_API_KEY` |
    | DuckDuckGo | 是（非官方，基于 HTML） | - |
    | Exa | 否 | `EXA_API_KEY` |
    | Firecrawl | 否 | `FIRECRAWL_API_KEY` |
    | Gemini | 否 | `GEMINI_API_KEY` |
    | Grok | 否（xAI OAuth 或密钥） | `XAI_API_KEY` |
    | Kimi | 否 | `KIMI_API_KEY` 或 `MOONSHOT_API_KEY` |
    | MiniMax Search | 否 | `MINIMAX_CODE_PLAN_KEY`、`MINIMAX_CODING_API_KEY` 或 `MINIMAX_API_KEY` |
    | Ollama Web 搜索 | 是（需要 `ollama signin`） | - |
    | Perplexity | 否 | `PERPLEXITY_API_KEY` 或 `OPENROUTER_API_KEY` |
    | SearXNG | 是（自托管） | `SEARXNG_BASE_URL` |
    | Tavily | 否 | `TAVILY_API_KEY` |

    Grok 还可以复用模型身份验证中的 xAI OAuth（`openclaw onboard --auth-choice xai-oauth`）。

    **推荐**：`openclaw configure --section web`，然后选择一个提供商。

    ```json5
    {
      plugins: {
        entries: {
          brave: {
            config: {
              webSearch: {
                apiKey: "BRAVE_API_KEY_HERE",
              },
            },
          },
        },
      },
      tools: {
        web: {
          search: {
            enabled: true,
            provider: "brave",
            maxResults: 5,
          },
          fetch: {
            enabled: true,
            provider: "firecrawl", // 可选；省略以自动检测
          },
        },
      },
    }
    ```

    特定提供商的 Web 搜索配置位于 `plugins.entries.<plugin>.config.webSearch.*` 下。旧版 `tools.web.search.*` 提供商路径仍会加载以保持兼容，但不应在新配置中使用。Firecrawl Web 获取回退配置位于 `plugins.entries.firecrawl.config.webFetch.*` 下。

    - 允许列表：添加 `web_search`/`web_fetch`/`x_search`，或使用 `group:web` 同时添加三者。
    - `web_fetch` 默认启用。
    - 如果省略 `tools.web.fetch.provider`，OpenClaw 会根据可用凭据自动检测第一个就绪的获取回退提供商；官方 Firecrawl 插件提供该回退。
    - 守护进程从 `~/.openclaw/.env`（或服务环境）读取环境变量。

    文档：[Web 工具](/zh-CN/tools/web)。

  </Accordion>

  <Accordion title="config.apply 清空了我的配置。如何恢复并避免再次发生？">
    `config.apply` 会替换**整个配置**；使用部分对象会删除其他所有内容。

    当前版本的 OpenClaw 可防止大多数意外覆盖：

    - OpenClaw 自身执行的配置写入会在写入前验证变更后的完整配置。
    - 无效或具有破坏性的 OpenClaw 自身写入会被拒绝，并保存为 `openclaw.json.rejected.*`。
    - 如果直接编辑导致启动或热重载失败，Gateway 网关会以关闭方式失败或跳过重载；它不会重写 `openclaw.json`。
    - `openclaw doctor --fix` 负责修复，可恢复上次已知正常的配置，并将被拒绝的文件保存为 `openclaw.json.clobbered.*`。

    恢复方法：

    - 检查 `openclaw logs --follow` 中是否有 `Invalid config at`、`Config write rejected:` 或 `config reload skipped (invalid config)`。
    - 检查活动配置旁最新的 `openclaw.json.clobbered.*` 或 `openclaw.json.rejected.*`。
    - 运行 `openclaw config validate` 和 `openclaw doctor --fix`。
    - 使用 `openclaw config set` 或 `config.patch`，仅复制回需要的键。
    - 如果没有上次已知正常的配置或被拒绝的载荷：从备份恢复，或重新运行 `openclaw doctor` 并重新配置渠道/模型。
    - 如果发生意外丢失：使用上次已知的配置或备份提交错误报告。本地编码智能体通常可以根据日志或历史记录重建可用配置。

    避免方法：小幅变更使用 `openclaw config set`，交互式编辑使用 `openclaw configure`，检查不熟悉的路径时使用 `config.schema.lookup`（返回浅层 schema 节点和直接子项摘要），部分 RPC 编辑使用 `config.patch`；仅将 `config.apply` 用于完整配置替换。面向智能体的 `gateway` 运行时工具即使通过旧版 `tools.bash.*` 别名，也会拒绝重写 `tools.exec.ask` / `tools.exec.security`。

    文档：[配置](/zh-CN/cli/config)、[配置设置](/zh-CN/cli/configure)、[Gateway 网关故障排查](/zh-CN/gateway/troubleshooting#gateway-rejected-invalid-config)、[Doctor](/zh-CN/gateway/doctor)。

  </Accordion>

  <Accordion title="如何运行一个中央 Gateway 网关，并在多台设备上使用专用工作节点？">
    常见模式：**一个 Gateway 网关**（例如 Raspberry Pi）加上**节点**和**智能体**。

    - **Gateway 网关（中央）**：负责渠道（Signal/WhatsApp）、路由和会话。
    - **节点（设备）**：Mac/iOS/Android 作为外围设备连接，并公开本地工具（`system.run`、`canvas`、`camera`）。
    - **智能体（工作节点）**：为特殊角色提供独立的智能核心/工作区（例如运维数据与个人数据）。
    - **子智能体**：从主智能体生成后台任务，以实现并行处理。
    - **TUI**：连接到 Gateway 网关并切换智能体/会话。

    文档：[节点](/zh-CN/nodes)、[远程访问](/zh-CN/gateway/remote)、[多智能体路由](/zh-CN/concepts/multi-agent)、[子智能体](/zh-CN/tools/subagents)、[TUI](/zh-CN/web/tui)。

  </Accordion>

  <Accordion title="OpenClaw 浏览器能以无头模式运行吗？">
    可以：

    ```json5
    {
      browser: { headless: true },
      agents: {
        defaults: {
          sandbox: { browser: { headless: true } },
        },
      },
    }
    ```

    默认值为 `false`（有界面模式）。在某些网站上，无头模式更容易触发反机器人检查（X/Twitter 经常阻止无头会话）。它使用相同的 Chromium 引擎，适用于大多数自动化任务；主要区别是没有可见的浏览器窗口（使用截图查看视觉内容）。请参阅[浏览器](/zh-CN/tools/browser)。

  </Accordion>

  <Accordion title="如何使用 Brave 进行浏览器控制？">
    将 `browser.executablePath` 设置为 Brave 二进制文件（或任何基于 Chromium 的浏览器）的路径，然后重启 Gateway 网关。请参阅[浏览器](/zh-CN/tools/browser#use-brave-or-another-chromium-based-browser)。
  </Accordion>
</AccordionGroup>

## 远程 Gateway 网关和节点

<AccordionGroup>
  <Accordion title="命令如何在 Telegram、Gateway 网关和节点之间传递？">
    Telegram 消息由 **Gateway 网关**处理，它运行智能体，并且仅在需要节点工具时才通过 **Gateway WebSocket** 调用节点：

    Telegram -> Gateway 网关 -> 智能体 -> `node.*` -> 节点 -> Gateway 网关 -> Telegram

    节点看不到入站提供商流量；它们只接收节点 RPC 调用。

  </Accordion>

  <Accordion title="如果 Gateway 网关托管在远程位置，我的智能体如何访问我的计算机？">
    将计算机配对为**节点**。Gateway 网关在其他位置运行，但可以通过 Gateway WebSocket 调用本地计算机上的 `node.*` 工具（屏幕、摄像头、系统）。

    1. 在始终在线的主机（VPS/家庭服务器）上运行 Gateway 网关。
    2. 将 Gateway 网关主机和你的计算机加入同一个 tailnet。
    3. 确保 Gateway WS 可访问（绑定到 tailnet 或使用 SSH 隧道）。
    4. 在本地打开 macOS 应用，并使用 **Remote over SSH** 模式（或直接使用 tailnet）连接，使其注册为节点。
    5. 批准节点：
       ```bash
       openclaw devices list
       openclaw devices approve <requestId>
       ```

    无需单独的 TCP 网桥；节点通过 Gateway WebSocket 连接。

    安全提醒：配对 macOS 节点后，将允许在该计算机上使用 `system.run`。仅配对你信任的设备；请查看[安全](/zh-CN/gateway/security)。

    文档：[节点](/zh-CN/nodes)、[Gateway 网关协议](/zh-CN/gateway/protocol)、[macOS 远程模式](/zh-CN/platforms/mac/remote)、[安全](/zh-CN/gateway/security)。

  </Accordion>

  <Accordion title="Tailscale 已连接，但我收不到回复。该怎么办？">
    检查基本状态：

    ```bash
    openclaw gateway status
    openclaw status
    openclaw channels status
    ```

    然后验证身份验证和路由：如果使用 Tailscale Serve，请确认 `gateway.auth.allowTailscale` 设置正确；如果通过 SSH 隧道连接，请确认隧道已启动并指向正确端口；确认你的私信/群组允许列表包含你的账号。

    文档：[Tailscale](/zh-CN/gateway/tailscale)、[远程访问](/zh-CN/gateway/remote)、[渠道](/zh-CN/channels)。

  </Accordion>

  <Accordion title="两个 OpenClaw 实例可以互相通信吗（本地 + VPS）？">
    可以，但没有内置的 Bot 间网桥。

    **最简单的方法**：使用两个 Bot 都能访问的普通聊天渠道（Slack/Telegram/WhatsApp）。让 Bot A 向 Bot B 发送消息，然后让 Bot B 正常回复。

    **CLI 网桥（通用）**：运行一个脚本，通过 `openclaw agent --message ... --deliver` 调用另一个 Gateway 网关，并将消息发送到另一个 Bot 正在监听的聊天。如果其中一个 Bot 位于远程 VPS 上，请通过 SSH/Tailscale 将 CLI 指向该远程 Gateway 网关（请参阅[远程访问](/zh-CN/gateway/remote)）：

    ```bash
    openclaw agent --message "来自本地 Bot 的问候" --deliver --channel telegram --reply-to <chat-id>
    ```

    添加防护措施，防止两个 Bot 无限循环（仅在被提及时响应、使用渠道允许列表，或设置“不回复 Bot 消息”规则）。

    文档：[远程访问](/zh-CN/gateway/remote)、[智能体 CLI](/zh-CN/cli/agent)、[智能体发送](/zh-CN/tools/agent-send)。

  </Accordion>

  <Accordion title="多个智能体需要使用独立的 VPS 吗？">
    不需要。一个 Gateway 网关可以托管多个智能体，每个智能体都有自己的工作区、默认模型和路由；这是常规设置，而且比每个智能体使用一个 VPS 更便宜、更简单。仅在需要严格隔离（安全边界），或存在不希望共享的差异很大的配置时，才使用独立 VPS。
  </Accordion>

  <Accordion title="在个人笔记本电脑上使用节点，而不是从 VPS 通过 SSH 访问，有什么优势？">
    有：节点是从远程 Gateway 网关访问笔记本电脑的首选方式，提供的能力不止 shell 访问。Gateway 网关可运行在 macOS/Linux 上（Windows 通过 WSL2），而且资源占用很低（小型 VPS 或 Raspberry Pi 级设备即可；4 GB RAM 已经足够），因此常见设置是使用一台始终在线的主机，并将笔记本电脑作为节点。

    - **无需入站 SSH**——节点通过设备配对主动连接到 Gateway WebSocket。
    - **更安全的执行控制**——`system.run` 受该笔记本电脑上的节点允许列表/审批限制。
    - **更多设备工具**——除 `system.run` 外，节点还会公开 `canvas`、`camera` 和 `screen`。
    - **本地浏览器自动化**——将 Gateway 网关保留在 VPS 上，但通过节点主机在本地运行 Chrome，或通过 Chrome MCP 连接本地 Chrome。

    SSH 适合临时 shell 访问；对于持续的智能体工作流和设备自动化，节点更加简单。

    文档：[节点](/zh-CN/nodes)、[节点 CLI](/zh-CN/cli/nodes)、[浏览器](/zh-CN/tools/browser)。

  </Accordion>

  <Accordion title="节点会运行 Gateway 网关服务吗？">
    不会。除非有意运行隔离的配置文件，否则每台主机只应运行**一个 Gateway 网关**（请参阅[多个 Gateway 网关](/zh-CN/gateway/multiple-gateways)）。节点是连接到 Gateway 网关的外围设备（iOS/Android 节点，或菜单栏应用中的 macOS“节点模式”）。关于无头节点主机和 CLI 控制，请参阅[节点主机 CLI](/zh-CN/cli/node)。

    更改 `gateway`、`discovery` 和托管插件表面后，需要完全重启。

  </Accordion>

  <Accordion title="是否可以通过 API / RPC 应用配置？">
    可以：

    - `config.schema.lookup`：在写入前检查一个配置子树及其浅层 schema 节点、匹配的 UI 提示和直接子项摘要。
    - `config.get`：获取当前快照及哈希值。
    - `config.patch`：安全的部分更新（大多数 RPC 编辑的首选方式）；可行时热重载，必要时重启。
    - `config.apply`：验证并替换完整配置；可行时热重载，必要时重启。
    - 面向智能体的 `gateway` 运行时工具仍会拒绝重写 `tools.exec.ask` / `tools.exec.security`；旧版 `tools.bash.*` 别名会规范化为相同的受保护路径。

  </Accordion>

  <Accordion title="首次安装所需的最小合理配置">
    ```json5
    {
      agents: { defaults: { workspace: "~/.openclaw/workspace" } },
      channels: { whatsapp: { allowFrom: ["+15555550123"] } },
    }
    ```

    设置你的工作区，并限制可以触发 Bot 的用户。

  </Accordion>

  <Accordion title="如何在 VPS 上设置 Tailscale 并从我的 Mac 连接？">
    1. **在 VPS 上安装并登录**：
       ```bash
       curl -fsSL https://tailscale.com/install.sh | sh
       sudo tailscale up
       ```
    2. 使用 Tailscale 应用在你的 Mac 上**安装并登录**，加入同一 tailnet。
    3. 在 Tailscale 管理控制台中**启用 MagicDNS**，以便 VPS 拥有稳定的名称。
    4. **使用 tailnet 主机名**：SSH `ssh user@your-vps.tailnet-xxxx.ts.net`；Gateway 网关 WS `ws://your-vps.tailnet-xxxx.ts.net:18789`。

    若要在不使用 SSH 的情况下访问 Control UI，请在 VPS 上使用 Tailscale Serve：

    ```bash
    openclaw gateway --tailscale serve
    ```

    这会让 Gateway 网关继续绑定到环回地址，并通过 Tailscale 暴露 HTTPS。请参阅 [Tailscale](/zh-CN/gateway/tailscale)。

  </Accordion>

  <Accordion title="如何将 Mac 节点连接到远程 Gateway 网关（Tailscale Serve）？">
    Serve 会暴露 **Gateway 网关 Control UI + WS**；节点通过同一个 Gateway 网关 WS 端点连接。

    1. 确保 VPS 和 Mac 位于同一个 tailnet 中。
    2. 以远程模式使用 macOS 应用（SSH 目标可以是 tailnet 主机名）——它会通过隧道转发 Gateway 网关端口，并作为节点连接。
    3. 批准节点：
       ```bash
       openclaw devices list
       openclaw devices approve <requestId>
       ```

    文档：[Gateway 网关协议](/zh-CN/gateway/protocol)、[设备发现](/zh-CN/gateway/discovery)、[macOS 远程模式](/zh-CN/platforms/mac/remote)。

  </Accordion>

  <Accordion title="我应该在第二台笔记本电脑上安装，还是只添加一个节点？">
    如果只需在第二台笔记本电脑上使用**本地工具**（屏幕/摄像头/exec），请将其添加为**节点**——使用一个 Gateway 网关，无需重复配置。本地节点工具目前仅支持 macOS。只有在需要**强隔离**或两个完全独立的机器人时，才安装第二个 Gateway 网关。

    文档：[节点](/zh-CN/nodes)、[节点 CLI](/zh-CN/cli/nodes)、[多个 Gateway 网关](/zh-CN/gateway/multiple-gateways)。

  </Accordion>
</AccordionGroup>

## 环境变量和 .env 加载

<AccordionGroup>
  <Accordion title="OpenClaw 如何加载环境变量？">
    OpenClaw 会从父进程（shell、launchd/systemd、CI 等）读取环境变量，并额外加载：

    - 当前工作目录中的 `.env`。
    - 来自 `~/.openclaw/.env` 的全局回退文件 `.env`（`$OPENCLAW_STATE_DIR/.env`）。

    两个 `.env` 文件都不会覆盖现有环境变量。对于工作区 `.env`，提供商凭据和端点路由键属于例外：诸如 `GEMINI_API_KEY`、`XAI_API_KEY`、`MISTRAL_API_KEY`、任何以 `_ENDPOINT` 结尾的键（以及其他内置提供商的身份验证或端点环境变量）都会在工作区 `.env` 中被忽略，应放在进程环境、`~/.openclaw/.env` 或配置 `env` 中。

    配置中的内联环境变量仅在进程环境中缺失时应用：

    ```json5
    {
      env: {
        OPENROUTER_API_KEY: "sk-or-...",
        vars: { GROQ_API_KEY: "gsk-..." },
      },
    }
    ```

    有关完整的优先级和来源，请参阅 [/environment](/zh-CN/help/environment)。

  </Accordion>

  <Accordion title="我通过服务启动了 Gateway 网关，但环境变量消失了。现在该怎么办？">
    有两种解决方法：

    1. 将缺失的键放入 `~/.openclaw/.env`，这样即使服务未继承你的 shell 环境，也能加载这些键。
    2. 启用 shell 导入（可选的便利功能）：
       ```json5
       {
         env: {
           shellEnv: {
             enabled: true,
             timeoutMs: 15000,
           },
         },
       }
       ```
       这会运行你的登录 shell，并仅导入缺失的预期键（绝不覆盖）。对应的环境变量：`OPENCLAW_LOAD_SHELL_ENV=1`、`OPENCLAW_SHELL_ENV_TIMEOUT_MS=15000`。

  </Accordion>

  <Accordion title='我设置了 COPILOT_GITHUB_TOKEN，但模型状态显示“Shell env: off.”。为什么？'>
    `openclaw models status` 报告 **shell 环境导入**是否已启用。“Shell env: off”并不意味着你的环境变量缺失——它只表示 OpenClaw 不会自动加载你的登录 shell。

    如果 Gateway 网关作为服务（launchd/systemd）运行，它不会继承你的 shell 环境。解决方法是将令牌放入 `~/.openclaw/.env`、启用 `env.shellEnv.enabled: true`，或将其添加到配置 `env` 中（仅在缺失时应用），然后重启 Gateway 网关并重新检查：

    ```bash
    openclaw models status
    ```

    Copilot 令牌按以下顺序解析：`OPENCLAW_GITHUB_TOKEN`，然后是 `COPILOT_GITHUB_TOKEN`，再然后是 `GH_TOKEN`，最后是 `GITHUB_TOKEN`。

    请参阅 [/concepts/model-providers](/zh-CN/concepts/model-providers) 和 [/environment](/zh-CN/help/environment)。

  </Accordion>
</AccordionGroup>

## 会话和多个聊天

<AccordionGroup>
  <Accordion title="如何开始全新的对话？">
    将 `/new` 或 `/reset` 作为独立消息发送。请参阅[会话管理](/zh-CN/concepts/session)。
  </Accordion>

  <Accordion title="如果我从不发送 /new，会话会自动重置吗？">
    默认不会。会话会保持相同的 `sessionId`，并且随着对话增长，压缩会限制活跃模型上下文的大小。`/new` 和 `/reset` 仍然可用，或者你可以使用 `mode: "daily"` 或 `mode: "idle"` 选择启用自动重置。每日模式会在 Gateway 网关主机上的 `session.reset.atHour`（默认 `4`，0-23）切换；空闲模式使用自上次实际交互以来的 `session.reset.idleMinutes`，不包括 heartbeat/cron/exec 系统事件。

    ```json5
    {
      session: {
        reset: { mode: "daily", atHour: 4 },
        resetByType: {
          group: { mode: "idle", idleMinutes: 120 },
          thread: { mode: "daily", atHour: 6 },
        },
        resetByChannel: {
          discord: { mode: "idle", idleMinutes: 10080 },
        },
      },
    }
    ```

    `resetByType` 支持 `direct`、`group` 和 `thread`。Doctor 会将旧版 `dm` 条目迁移到 `direct`；该架构会拒绝 `dm`。当未设置 `session.reset`/`resetByType` 块时，旧版顶层 `session.idleMinutes` 仍可作为空闲模式默认值的兼容别名使用。有关完整生命周期，请参阅[会话管理](/zh-CN/concepts/session)。

  </Accordion>

  <Accordion title="有没有办法组建一支 OpenClaw 实例团队（一个 CEO 和多个智能体）？">
    可以，通过**多智能体路由**和**子智能体**实现：一个协调智能体加上多个拥有各自工作区和模型的工作智能体。

    最好将其视为一个有趣的实验——它会消耗大量令牌，而且通常不如使用具有独立会话的单个机器人高效。典型模式是与一个机器人交互，使用不同会话并行处理工作，并在需要时生成子智能体。

    文档：[多智能体路由](/zh-CN/concepts/multi-agent)、[子智能体](/zh-CN/tools/subagents)、[智能体 CLI](/zh-CN/cli/agents)。

  </Accordion>

  <Accordion title="为什么任务执行到一半时上下文被截断？如何防止这种情况？">
    会话上下文受模型窗口限制。长时间聊天、大量工具输出或大量文件都可能触发压缩或截断。

    - 让机器人汇总当前状态并将其写入文件。
    - 在长任务开始前使用 `/compact`，切换主题时使用 `/new`。
    - 将重要上下文保存在工作区中，并让机器人重新读取。
    - 对于耗时较长或并行进行的工作，使用子智能体，以减小主聊天的上下文。
    - 如果经常发生这种情况，请选择上下文窗口更大的模型。

  </Accordion>

  <Accordion title="如何完全重置 OpenClaw 但保留其安装？">
    ```bash
    openclaw reset
    ```

    非交互式完全重置：

    ```bash
    openclaw reset --scope full --yes --non-interactive
    ```

    然后重新运行设置：

    ```bash
    openclaw onboard --install-daemon
    ```

    如果新手引导检测到现有配置，也会提供 **重置** 选项；请参阅[新手引导（CLI）](/zh-CN/start/wizard)。如果使用了配置文件（`--profile` / `OPENCLAW_PROFILE`），请重置每个状态目录（默认值为 `~/.openclaw-<profile>`）。仅限开发环境的重置：`openclaw gateway --dev --reset` 会清除开发配置、凭据、会话和工作区。

  </Accordion>

  <Accordion title='出现“context too large”错误时，如何重置或压缩？'>
    - **压缩**（保留对话并汇总较早的轮次）：使用 `/compact`，或使用 `/compact <instructions>` 指导摘要生成。
    - **重置**（为同一个聊天键创建全新的会话 ID）：使用 `/new` 或 `/reset`。

    如果问题持续发生，请调整**会话修剪**（`agents.defaults.contextPruning`）以清理较早的工具输出，或使用上下文窗口更大的模型。

    文档：[压缩](/zh-CN/concepts/compaction)、[会话修剪](/zh-CN/concepts/session-pruning)、[会话管理](/zh-CN/concepts/session)。

  </Accordion>

  <Accordion title='为什么会看到“LLM request rejected: messages.content.tool_use.input field required”？'>
    提供商验证错误：模型生成了一个 `tool_use` 块，但缺少必需的 `input`。这通常意味着会话历史记录已过期或损坏（常见于长对话之后，或工具/架构发生更改之后）。

    修复方法：使用 `/new` 开始一个新会话（作为独立消息发送）。

  </Accordion>

  <Accordion title="为什么每隔 30 分钟就会收到 Heartbeat 消息？">
    默认情况下，Heartbeat 每 **30m** 运行一次；如果解析出的身份验证模式为 Anthropic OAuth/token 身份验证（包括复用 Claude CLI），且未设置 `heartbeat.every`，则每 **1h** 运行一次。可以调整或禁用：

    ```json5
    {
      agents: {
        defaults: {
          heartbeat: {
            every: "2h", // 或使用 "0m" 禁用
          },
        },
      },
    }
    ```

    如果 `HEARTBEAT.md` 存在但实际上为空（仅包含空行、Markdown/HTML 注释、ATX 标题、代码围栏标记或空列表项占位符），OpenClaw 会跳过本次 Heartbeat 运行，以节省 API 调用。如果文件不存在，Heartbeat 仍会运行，由模型决定如何处理。

    每个智能体的覆盖配置使用 `agents.entries.*.heartbeat`。文档：[Heartbeat](/zh-CN/gateway/heartbeat)。

  </Accordion>

  <Accordion title='是否需要向 WhatsApp 群组添加“机器人账号”？'>
    不需要。OpenClaw 使用**你自己的账号**运行——只要你在群组中，OpenClaw 就能看到该群组。默认情况下，在你允许发送者（`groupPolicy: "allowlist"`）之前，群组回复会被阻止。

    要将群组回复限制为仅你本人：

    ```json5
    {
      channels: {
        whatsapp: {
          groupPolicy: "allowlist",
          groupAllowFrom: ["+15551234567"],
        },
      },
    }
    ```

  </Accordion>

  <Accordion title="如何获取 WhatsApp 群组的 JID？">
    最快的方法：持续查看日志，并在群组中发送一条测试消息。

    ```bash
    openclaw logs --follow --json
    ```

    查找以 `@g.us` 结尾的 `chatId`（或 `from`），例如 `1234567890-1234567890@g.us`。

    如果已完成配置或加入允许列表，请从配置中列出群组：

    ```bash
    openclaw directory groups list --channel whatsapp
    ```

    文档：[WhatsApp](/zh-CN/channels/whatsapp)、[目录](/zh-CN/cli/directory)、[日志](/zh-CN/cli/logs)。

  </Accordion>

  <Accordion title="为什么 OpenClaw 不在群组中回复？">
    两种常见原因：默认启用了提及门控（必须 @提及机器人，或匹配 `mentionPatterns`）；或者你配置了 `channels.whatsapp.groups`，但未配置 `"*"`，并且该群组不在允许列表中。

    请参阅[群组](/zh-CN/channels/groups)和[群组消息](/zh-CN/channels/group-messages)。

  </Accordion>

  <Accordion title="群组/话题是否与私信共享上下文？">
    默认情况下，直接聊天会归入主会话。群组/渠道有各自的会话键，Telegram 话题和 Discord 线程也是独立会话。请参阅[群组](/zh-CN/channels/groups)和[群组消息](/zh-CN/channels/group-messages)。
  </Accordion>

  <Accordion title="可以创建多少个工作区和智能体？">
    没有硬性限制——创建几十个甚至数百个都没问题，但请注意：

    - **磁盘增长**：活跃会话和转录记录存储在每个 Agent 的 SQLite 数据库中；旧版/归档工件仍可能在 `~/.openclaw/agents/<agentId>/sessions/` 下不断累积。
    - **Token 成本**：Agent 越多，并发模型使用量越大。
    - **运维开销**：每个 Agent 都有各自的身份验证配置文件、工作区和频道路由。

    每个 Agent 保留一个**活跃**工作区（`agents.defaults.workspace`）；如果磁盘占用增长，请使用 `openclaw sessions cleanup` 清理旧会话（不要手动编辑活跃的 SQLite 状态）；使用 `openclaw doctor` 查找遗留工作区和配置文件不匹配问题。

  </Accordion>

  <Accordion title="我能否同时运行多个 Bot 或聊天（Slack），应该如何设置？">
    可以，通过**多 Agent 路由**实现：运行多个相互隔离的 Agent，并按频道/账号/对等方路由入站消息。Slack 支持作为频道，并可绑定到特定 Agent。

    浏览器访问能力很强，但并非“人类能做什么就能做什么”——反 Bot 机制、CAPTCHA 和 MFA 仍可能阻止自动化。要获得最可靠的控制，请使用主机上的本地 Chrome MCP，或使用实际运行浏览器的计算机上的 CDP。

    最佳实践设置：使用始终在线的 Gateway 网关主机（VPS/Mac mini），每个角色使用一个 Agent（绑定），将 Slack 频道绑定到这些 Agent，并在需要时通过 Chrome MCP 或节点使用本地浏览器。

    文档：[多 Agent 路由](/zh-CN/concepts/multi-agent)、[Slack](/zh-CN/channels/slack)、[浏览器](/zh-CN/tools/browser)、[节点](/zh-CN/nodes)。

  </Accordion>
</AccordionGroup>

## 模型、故障转移和身份验证配置文件

有关模型的问答（默认值、选择、别名、切换、故障转移和身份验证配置文件）请参阅[模型常见问题](/zh-CN/help/faq-models)。

## Gateway 网关：端口、“已在运行”和远程模式

<AccordionGroup>
  <Accordion title="Gateway 网关使用哪个端口？">
    `gateway.port` 控制 WebSocket + HTTP（Control UI、Hooks 等）共用的单个多路复用端口。优先级：

    ```text
    --port > OPENCLAW_GATEWAY_PORT > gateway.port > 默认值 18789
    ```

  </Accordion>

  <Accordion title='为什么 openclaw gateway status 显示“Runtime: running”，但“Connectivity probe: failed”？'>
    “Running”是**进程监督器**的视角（launchd/systemd/schtasks）；连接探测则是 CLI 实际连接 Gateway 网关 WebSocket。请以 `openclaw gateway status` 中的以下几行为准：`Probe target:`（探测使用的 URL）、`Listening:`（端口上实际绑定的内容）、`Last gateway error:`（进程仍在运行但端口未监听时的常见根本原因）。
  </Accordion>

  <Accordion title='为什么 openclaw gateway status 显示的“Config (cli)”和“Config (service)”不同？'>
    你正在编辑一个配置文件，而服务运行时使用的是另一个配置文件（通常是 `--profile` / `OPENCLAW_STATE_DIR` 不匹配）。

    修复方法：从你希望服务使用的同一 `--profile` / 环境中运行：

    ```bash
    openclaw gateway install --force
    ```

  </Accordion>

  <Accordion title='“another gateway instance is already listening”是什么意思？'>
    OpenClaw 在启动时立即绑定 WebSocket 监听器（默认 `ws://127.0.0.1:18789`），以此实施运行时锁。如果绑定因 `EADDRINUSE` 失败，就会抛出 `GatewayLockError`（“another gateway instance is already listening”）。

    修复方法：停止另一个实例、释放端口，或使用 `openclaw gateway --port <port>` 运行。

  </Accordion>

  <Accordion title="如何以远程模式运行 OpenClaw（客户端连接到其他位置的 Gateway 网关）？">
    设置 `gateway.mode: "remote"` 并指向远程 WebSocket URL，也可以选择配置共享密钥远程凭据：

    ```json5
    {
      gateway: {
        mode: "remote",
        remote: {
          url: "ws://gateway.tailnet:18789",
          token: "your-token",
          password: "your-password",
        },
      },
    }
    ```

    - `openclaw gateway` 仅在 `gateway.mode` 为 `local` 时启动（或者你传入覆盖标志）。
    - macOS 应用会监视配置文件，并在这些值发生变化时实时切换模式。
    - `gateway.remote.token` / `.password` 仅是客户端远程凭据；它们本身不会启用本地 Gateway 网关身份验证。

  </Accordion>

  <Accordion title='Control UI 显示“unauthorized”（或不断重新连接）。现在怎么办？'>
    你的 Gateway 网关身份验证路径与 UI 的身份验证方式不匹配。

    事实（来自代码）：

    - Control UI 将 Token 保存在 `sessionStorage` 中，其作用域仅限当前浏览器标签页和所选 Gateway 网关 URL，因此同一标签页中的刷新仍可正常工作，而无需将 Token 长期持久化到 localStorage。
    - 在 `AUTH_TOKEN_MISMATCH` 上，当 Gateway 网关返回重试提示（`canRetryWithDeviceToken=true`、`recommendedNextStep=retry_with_device_token`）时，受信任的客户端可以使用缓存的设备 Token 尝试一次有界重试。
    - 该缓存 Token 重试会复用与设备 Token 一同存储的已批准权限范围；显式 `deviceToken` / 显式 `scopes` 调用方会保留其请求的权限范围集，而不会继承缓存的权限范围。
    - 在该重试路径之外，连接身份验证的优先级依次为：显式共享 Token/密码、显式 `deviceToken`、已存储的设备 Token，最后是引导 Token。
    - 内置设置代码引导会返回一个具有 `scopes: []` 的节点设备 Token，以及一个用于受信任移动端新手引导的有界操作员交接 Token。操作员交接可以读取设置期间的原生配置，但不会授予配对变更权限范围或 `operator.admin`。

    修复方法：

    - 最快的方法：`openclaw dashboard`（输出并复制仪表板 URL，并尝试打开；如果是无头环境，则显示 SSH 提示）。
    - 还没有 Token：`openclaw doctor --generate-gateway-token`。
    - 远程连接：先使用 `ssh -N -L 18789:127.0.0.1:18789 user@host` 建立隧道，然后打开 `http://127.0.0.1:18789/`。
    - 共享密钥模式：设置 `gateway.auth.token` / `OPENCLAW_GATEWAY_TOKEN` 或 `gateway.auth.password` / `OPENCLAW_GATEWAY_PASSWORD`，然后在 Control UI 设置中粘贴对应的密钥。
    - Tailscale Serve 模式：确认 `gateway.auth.allowTailscale` 已启用，并且你打开的是 Serve URL，而不是绕过 Tailscale 身份标头的原始环回/tailnet URL。
    - 受信任代理模式：确认你通过已配置的身份感知代理访问。同一主机上的环回代理还需要 `gateway.auth.trustedProxy.allowLoopback = true`。
    - 一次重试后仍不匹配：轮换/重新批准已配对的设备 Token：
      ```bash
      openclaw devices list
      openclaw devices rotate --device <id> --role operator
      ```
    - 轮换被拒绝：已配对设备的会话只能轮换其**自身**设备，除非它们还具有 `operator.admin`；显式 `--scope` 值不能超出调用方当前的操作员权限范围。
    - 仍无法解决：`openclaw status --all`，以及参阅[故障排查](/zh-CN/gateway/troubleshooting)。有关身份验证的详细信息，请参阅[仪表板](/zh-CN/web/dashboard)。

  </Accordion>

  <Accordion title="我设置了 gateway.bind tailnet，但它只监听环回接口">
    `tailnet` 绑定会从你的网络接口中选择一个 Tailscale IP（100.64.0.0/10）。如果计算机未连接到 Tailscale（或接口已关闭），Gateway 网关会回退到环回接口，而不会暴露其他网络接口。

    修复方法：在该主机上启动 Tailscale 并重启 Gateway 网关，或显式切换到 `gateway.bind: "loopback"` / `"lan"`。

    `tailnet` 是显式设置；`auto` 优先使用环回接口。使用 `gateway.bind: "tailnet"` 可将非环回暴露限制在 Tailnet 内，同时保留必需的同一主机 `127.0.0.1` 监听器。

  </Accordion>

  <Accordion title="能否在同一主机上运行多个 Gateway 网关？">
    通常不能——一个 Gateway 网关可以运行多个消息频道和 Agent。仅在需要冗余（例如救援 Bot）或严格隔离时使用多个 Gateway 网关，并为每个实例分别设置独立的 `OPENCLAW_CONFIG_PATH`、`OPENCLAW_STATE_DIR`、`agents.defaults.workspace` 和唯一的 `gateway.port`。

    建议：每个实例使用 `openclaw --profile <name> ...`（自动创建 `~/.openclaw-<name>`）；每个配置文件的配置使用唯一的 `gateway.port`（手动运行时可使用 `--port`）；并通过 `openclaw --profile <name> gateway install` 为每个配置文件创建服务。

    配置文件还会为服务名称添加后缀：launchd `ai.openclaw.<profile>`、systemd `openclaw-gateway-<profile>.service`、Windows `OpenClaw Gateway (<profile>)`。不带限定符的 `openclaw-gateway` systemd 单元仅用于默认配置文件；重命名前的旧版 systemd 单元名称 `clawdbot-gateway` 会自动迁移。

    完整指南：[多个 Gateway 网关](/zh-CN/gateway/multiple-gateways)。

  </Accordion>

  <Accordion title='“invalid handshake”/代码 1008 是什么意思？'>
    Gateway 网关是一个 **WebSocket 服务器**，要求第一条消息是 `connect` 帧。任何其他消息都会导致连接以**代码 1008**（违反策略）关闭。

    常见原因：你在浏览器中打开了 **HTTP** URL，而不是使用 WS 客户端；使用了错误的端口/路径；或者代理/隧道移除了身份验证标头或发送了非 Gateway 网关请求。

    修复方法：使用 WS URL（`ws://<host>:18789`，或通过 HTTPS 使用 `wss://...`）；不要在普通浏览器标签页中打开 WS 端口；启用身份验证时，在 `connect` 帧中包含 Token/密码。CLI/TUI 示例：

    ```bash
    openclaw tui --url ws://<host>:18789 --token <token>
    ```

    协议详情：[Gateway 网关协议](/zh-CN/gateway/protocol)。

  </Accordion>
</AccordionGroup>

## 日志和调试

<AccordionGroup>
  <Accordion title="日志在哪里？">
    文件日志（结构化）：默认配置文件使用 `/tmp/openclaw/openclaw-YYYY-MM-DD.log`，命名配置文件使用 `/tmp/openclaw/openclaw-<profile>-YYYY-MM-DD.log`。通过 `logging.file` 设置稳定路径；通过 `logging.level` 设置文件日志级别；通过 `--verbose` 和 `logging.consoleLevel` 设置控制台详细程度。

    最快的实时查看方式：

    ```bash
    openclaw logs --follow
    ```

    服务/进程监督器日志（Gateway 网关通过 launchd/systemd 运行时）：

    - macOS launchd 标准输出：`~/Library/Logs/openclaw/gateway.log`（配置文件使用 `gateway-<profile>.log`；标准错误输出会被抑制）。
    - Linux：`journalctl --user -u openclaw-gateway[-<profile>].service -n 200 --no-pager`。
    - Windows：`schtasks /Query /TN "OpenClaw Gateway (<profile>)" /V /FO LIST`。

    详见[故障排查](/zh-CN/gateway/troubleshooting)。

  </Accordion>

  <Accordion title="如何启动/停止/重启 Gateway 网关服务？">
    ```bash
    openclaw gateway status
    openclaw gateway restart
    ```

    如果你手动运行 Gateway 网关，`openclaw gateway --force` 可以重新占用该端口。参阅 [Gateway 网关](/zh-CN/gateway)。

  </Accordion>

  <Accordion title="我在 Windows 上关闭了终端——如何重启 OpenClaw？">
    Windows 有三种安装模式：

    **1) Windows Hub 本地设置**：原生应用管理应用自有的本地 WSL Gateway 网关。从开始菜单或系统托盘打开 **OpenClaw Companion**，然后使用 **Gateway Setup** 或 Connections 标签页。

    **2) 手动设置 WSL2 Gateway 网关**：Gateway 网关在 Linux 内运行。
    ```powershell
    wsl
    openclaw gateway status
    openclaw gateway restart
    ```
    如果你从未安装服务，请在前台启动：`openclaw gateway run`。

    **3) 原生 Windows CLI/Gateway 网关**：直接在 Windows 中运行。
    ```powershell
    openclaw gateway status
    openclaw gateway restart
    ```
    如果你手动运行（没有服务）：`openclaw gateway run`。

    文档：[Windows](/zh-CN/platforms/windows)、[Gateway 网关服务运行手册](/zh-CN/gateway)。

  </Accordion>

  <Accordion title="Gateway 网关已启动，但始终收不到回复。应该检查什么？">
    快速健康检查：

    ```bash
    openclaw status
    openclaw models status
    openclaw channels status
    openclaw logs --follow
    ```

    常见原因：**Gateway 网关主机**未加载模型身份验证信息（检查 `models status`）；频道配对/允许列表阻止了回复（检查频道配置和日志）；或者 WebChat/仪表板打开时未使用正确的 Token。如果是远程连接，请确认隧道/Tailscale 连接已建立，并且 Gateway 网关 WebSocket 可访问。

    文档：[渠道](/zh-CN/channels)、[故障排查](/zh-CN/gateway/troubleshooting)、[远程访问](/zh-CN/gateway/remote)。

  </Accordion>

  <Accordion title='"已断开与 Gateway 网关的连接：无原因"——现在该怎么办？'>
    这通常意味着 UI 失去了 WebSocket 连接。请检查：Gateway 网关是否正在运行（`openclaw gateway status`）？是否健康（`openclaw status`）？UI 使用的 token 是否正确（`openclaw dashboard`）？如果是远程连接，隧道/Tailscale 链接是否正常？

    然后持续查看日志：

    ```bash
    openclaw logs --follow
    ```

    文档：[仪表板](/zh-CN/web/dashboard)、[远程访问](/zh-CN/gateway/remote)、[故障排查](/zh-CN/gateway/troubleshooting)。

  </Accordion>

  <Accordion title="Telegram setMyCommands 失败。我应该检查什么？">
    ```bash
    openclaw channels status
    openclaw channels logs --channel telegram
    ```

    然后根据错误进行排查：

    - `BOT_COMMANDS_TOO_MUCH`：Telegram 菜单中的条目过多。OpenClaw 已经会将条目裁剪到 Telegram 的限制以内，并使用更少的命令重试，但仍可能丢弃一些菜单条目。请减少插件/技能/自定义命令，或者在不需要菜单时禁用 `channels.telegram.commands.native`。
    - `TypeError: fetch failed`、`Network request for 'setMyCommands' failed!` 或类似网络错误：如果在 VPS 上或代理之后，请确认允许出站 HTTPS，并且 `api.telegram.org` 的 DNS 解析正常。

    如果 Gateway 网关位于远程主机上，请在 Gateway 网关主机上检查日志。

    文档：[Telegram](/zh-CN/channels/telegram)、[渠道故障排查](/zh-CN/channels/troubleshooting)。

  </Accordion>

  <Accordion title="TUI 没有显示输出。我应该检查什么？">
    ```bash
    openclaw status
    openclaw models status
    openclaw logs --follow
    ```

    在 TUI 中，使用 `/status` 查看当前状态。如果你期望在聊天渠道中收到回复，请确认已启用投递（`/deliver on`）。

    文档：[TUI](/zh-CN/web/tui)、[斜杠命令](/zh-CN/tools/slash-commands)。

  </Accordion>

  <Accordion title="如何彻底停止再启动 Gateway 网关？">
    如果你安装了服务（macOS 上的 launchd、Linux 上的 systemd）：

    ```bash
    openclaw gateway stop
    openclaw gateway start
    ```

    在前台运行时，使用 Ctrl-C 停止，然后执行 `openclaw gateway run`。

    文档：[Gateway 网关服务运行手册](/zh-CN/gateway)。

  </Accordion>

  <Accordion title="简单解释：openclaw gateway restart 与 openclaw gateway 的区别">
    `openclaw gateway restart` 重启**后台服务**（launchd/systemd）。`openclaw gateway` 在当前终端会话中**以前台方式**运行 Gateway 网关。如果你安装了服务，请使用 gateway 子命令；如果只需临时运行一次，请直接以前台方式运行。
  </Accordion>

  <Accordion title="出现故障时获取更多详细信息的最快方法">
    使用 `--verbose` 启动 Gateway 网关，以便在控制台中查看更多详细信息，然后检查日志文件中的渠道身份验证、模型路由和 RPC 错误。
  </Accordion>
</AccordionGroup>

## 媒体和附件

<AccordionGroup>
  <Accordion title="我的技能生成了图像/PDF，但没有发送任何内容">
    智能体发送出站附件时，必须使用 `media`、`mediaUrl`、`path` 或 `filePath` 等结构化媒体字段。请参阅 [OpenClaw 助手设置](/zh-CN/start/openclaw)和[智能体发送](/zh-CN/tools/agent-send)。

    ```bash
    openclaw message send --target +15555550123 --message "给你" --media /path/to/file.png
    ```

    还要检查：目标渠道支持出站媒体，且未被允许列表阻止；文件未超过提供商的大小限制（图像会缩放至最长边不超过 2048px）；`tools.fs.workspaceOnly=true` 将本地路径发送限制为工作区、临时/媒体存储区和经沙箱验证的文件；`tools.fs.workspaceOnly=false`（默认）允许结构化本地媒体发送使用智能体已经能够读取的主机本地文件，适用于媒体和安全的文档类型（图像、音频、视频、PDF、Office 文档，以及经过验证的文本文件，如 Markdown/MD、TXT、JSON、YAML/YML）。这并不是秘密扫描器——只要扩展名和内容验证匹配，就可以附加智能体可读取的 `secret.txt` 或 `config.json`。请将敏感文件放在智能体可读路径之外，或者保留 `tools.fs.workspaceOnly=true`，以便对本地路径发送进行更严格的限制。

    请参阅[图像](/zh-CN/nodes/images)。

  </Accordion>
</AccordionGroup>

## 安全和访问控制

<AccordionGroup>
  <Accordion title="允许 OpenClaw 接收入站私信是否安全？">
    请将入站私信视为不可信输入。默认设置会降低风险：

    - 支持私信的渠道默认采用**配对**行为：未知发送者会收到配对码，其消息不会被处理。使用 `openclaw pairing approve --channel <channel> [--account <id>] <code>` 批准。待处理请求上限为**每个渠道 3 个**；如果未收到配对码，请检查 `openclaw pairing list --channel <channel> [--account <id>]`。
    - 公开开放私信需要明确选择启用（`dmPolicy: "open"` 和允许列表 `"*"`）。

    运行 `openclaw doctor` 以发现有风险的私信策略。

  </Accordion>

  <Accordion title="提示词注入只是公共机器人需要担心的问题吗？">
    不是。提示词注入涉及的是**不可信内容**，而不只是哪些人可以向机器人发送私信。如果你的助手会读取外部内容（Web 搜索/抓取、浏览器页面、电子邮件、文档、附件、粘贴的日志），这些内容就可能携带试图劫持模型的指令——即使只有你一个发送者也是如此。

    启用工具时风险最大：模型可能受骗而泄露上下文，或代表你调用工具。请缩小影响范围：

    - 使用只读或禁用工具的“阅读器”智能体来总结不可信内容
    - 对于启用了工具的智能体，保持关闭 `web_search` / `web_fetch` / `browser`
    - 也要将解码后的文件/文档文本视为不可信内容：OpenResponses `input_file` 和媒体附件提取都会使用明确的外部内容边界标记包裹提取的文本，而不是直接传递原始文件文本
    - 启用沙箱隔离，并使用严格的工具允许列表

    详情：[安全](/zh-CN/gateway/security)。

  </Accordion>

  <Accordion title="OpenClaw 使用 TypeScript/Node 而非 Rust/WASM，是否因此更不安全？">
    语言和运行时很重要，但它们并不是个人智能体面临的主要风险。实际风险包括 Gateway 网关暴露、谁能向机器人发送消息、提示词注入、工具权限范围、凭据处理、浏览器访问、Exec 访问，以及对第三方技能/插件的信任。

    Rust 和 WASM 可以为某些代码类别提供更强的隔离，但无法解决提示词注入、不当的允许列表、Gateway 网关公开暴露、权限范围过大的工具，或已登录敏感账户的浏览器配置文件。请将以下措施视为主要控制手段：保持 Gateway 网关私有或要求身份验证；对私信/群组使用配对和允许列表；对于不可信输入，拒绝使用高风险工具或将其置于沙箱中；仅安装可信的插件和技能；并在更改配置后运行 `openclaw security audit --deep`。

    详情：[安全](/zh-CN/gateway/security)、[沙箱隔离](/zh-CN/gateway/sandboxing)。

  </Accordion>

  <Accordion title="我看到有关 OpenClaw 实例暴露的报告。应该检查什么？">
    ```bash
    openclaw security audit --deep
    openclaw gateway status
    ```

    更安全的基线：Gateway 网关绑定到 `loopback`，或仅通过经过身份验证的私有访问方式暴露（tailnet、SSH 隧道、token/密码身份验证，或正确配置的可信代理）；私信采用 `pairing` 或 `allowlist` 模式；群组使用允许列表，并要求提及后才响应，除非每位成员都可信；对于会读取不可信内容的智能体，拒绝使用高风险工具（`exec`、`browser`、`gateway`、`cron`）或严格限制其权限范围；在执行工具时如需缩小影响范围，则启用沙箱隔离。

    应优先修复的问题包括：未经身份验证的公开绑定、启用了工具的开放私信/群组，以及暴露的浏览器控制。详情：[openclaw security audit](/zh-CN/gateway/security#openclaw-security-audit)。

  </Accordion>

  <Accordion title="安装 ClawHub 技能和第三方插件安全吗？">
    请将第三方技能和插件视为你选择信任的代码。ClawHub 技能页面会在安装前显示扫描状态，但扫描并不是完整的安全边界。OpenClaw 在安装或更新插件/技能时，不会运行内置的本地危险代码阻止机制；请使用由操作员管理的 `security.installPolicy` 在本地作出允许/阻止决策。

    更安全的做法：优先选择可信作者和固定版本；启用技能/插件前先阅读其内容；严格限制插件/技能允许列表；在仅配备最少工具的沙箱中运行涉及不可信输入的工作流；并避免向第三方代码授予广泛的文件系统、Exec、浏览器或秘密访问权限。

    详情：[Skills](/zh-CN/tools/skills)、[插件](/zh-CN/tools/plugin)、[安全](/zh-CN/gateway/security)。

  </Accordion>

  <Accordion title="我的机器人是否应该使用独立的电子邮件、GitHub 账户或电话号码？">
    对大多数设置而言，是的。使用独立账户和电话号码隔离机器人，可以在出现问题时缩小影响范围，也便于轮换凭据或撤销访问权限，而不会影响你的个人账户。

    从小范围开始：仅授予对实际所需工具和账户的访问权限，之后可按需扩展。

    文档：[安全](/zh-CN/gateway/security)、[配对](/zh-CN/channels/pairing)。

  </Accordion>

  <Accordion title="我可以让它自主处理我的短信吗？这样安全吗？">
    我们**不**建议让它完全自主处理你的个人消息。最安全的方式是：将私信保持在**配对模式**或使用严格的允许列表；如果它需要代表你发送消息，请使用**独立的号码或账户**；让它起草消息，并由你在**发送前批准**。

    如需试验，请使用专用的隔离账户。请参阅[安全](/zh-CN/gateway/security)。

  </Accordion>

  <Accordion title="我可以使用更便宜的模型来执行个人助手任务吗？">
    可以，**前提是**智能体仅用于聊天且输入可信。较小层级的模型更容易受到指令劫持，因此请避免将其用于启用了工具的智能体，或用于读取不可信内容。如果必须使用较小的模型，请严格限制工具并在沙箱中运行。请参阅[安全](/zh-CN/gateway/security)。
  </Accordion>

  <Accordion title="我在 Telegram 中运行了 /start，但没有收到配对码">
    仅当未知发送者向机器人发送消息且启用了 `dmPolicy: "pairing"` 时，才会发送配对码；仅执行 `/start` 不会生成配对码。

    检查待处理请求：

    ```bash
    openclaw pairing list telegram
    ```

    如需立即访问，请将你的发送者 ID 加入允许列表，或为该账户设置 `dmPolicy: "open"`。

  </Accordion>

  <Accordion title="WhatsApp：它会向我的联系人发送消息吗？配对如何运作？">
    不会。WhatsApp 的默认私信策略是**配对**。未知发送者只会收到配对码；其消息**不会被处理**。OpenClaw 只会回复它收到消息的聊天，或执行你明确触发的发送操作。

    ```bash
    openclaw pairing approve whatsapp <code>
    openclaw pairing list whatsapp
    ```

    向导中的电话号码提示用于设置你的**允许列表/所有者**，以允许你自己的私信——不会用于自动发送。在你的个人 WhatsApp 号码上，请使用该号码并启用 `channels.whatsapp.selfChatMode`。

  </Accordion>
</AccordionGroup>

## 聊天命令、中止任务和“它停不下来”

<AccordionGroup>
  <Accordion title="如何停止在聊天中显示内部系统消息？">
    大多数内部/工具消息仅在为该会话启用**详细输出**、**跟踪**或**推理**时显示。

    在出现这些消息的聊天中执行以下命令：

    ```text
    /verbose off
    /trace off
    /reasoning off
    ```

    如果仍然很嘈杂：请检查 Control UI 中的会话设置，并将详细输出设为 **inherit**；确认你使用的机器人配置文件未在配置中设置 `verboseDefault: "on"`。

    文档：[思考和详细输出](/zh-CN/tools/thinking)、[安全](/zh-CN/gateway/security/index#reasoning-and-verbose-output-in-groups)。

  </Accordion>

  <Accordion title="如何停止/取消正在运行的任务？">
    将以下任一内容**作为独立消息**（不带斜杠）发送，即可触发中止：`stop`、`stop action`、`stop current action`、`stop run`、`stop current run`、`stop agent`、`stop the agent`、`stop openclaw`、`openclaw stop`、`stop don't do anything`、`stop do not do anything`、`stop doing anything`、`do not do that`、`please stop`、`stop please`、`abort`、`esc`、`exit`、`interrupt`、`halt`。常见的非英语触发词（法语、德语、西班牙语、中文、日语、印地语、阿拉伯语、俄语）也有效。

    对于由 Exec 工具启动的后台进程，让智能体运行：

    ```text
    process action:kill sessionId:XXX
    ```

    大多数斜杠命令必须作为以 `/` 开头的**独立**消息发送，但少数快捷方式（如 `/status`）也可由允许列表中的发送者在消息内使用。请参阅[斜杠命令](/zh-CN/tools/slash-commands)。

  </Accordion>

  <Accordion title='如何从 Telegram 发送 Discord 消息？（“Cross-context messaging denied”）'>
    OpenClaw 默认阻止**跨提供商**消息传递。如果工具调用绑定到 Telegram，则不会向 Discord 发送消息，除非你明确允许；此设置会立即生效，无需重启 Gateway 网关：

    ```json5
    {
      tools: {
        message: {
          crossContext: {
            allowAcrossProviders: true,
            marker: { enabled: true, prefix: "[from {channel}] " },
          },
        },
      },
    }
    ```

  </Accordion>

  <Accordion title='为什么感觉 Bot 会“忽略”快速连续发送的消息？'>
    默认情况下，运行期间收到的提示会被引导至当前活动运行。使用 `/queue` 选择活动运行的行为：

    - `steer`（默认）- 在下一个模型边界引导活动运行。
    - `followup` - 将消息加入队列，并在当前运行结束后逐条运行。
    - `collect` - 将兼容的消息加入队列，并在当前运行结束后统一回复一次。
    - `interrupt` - 中止当前运行并重新开始。

    可以为队列模式添加选项，例如 `debounce:0.5s cap:25 drop:summarize`。请参阅[命令队列](/zh-CN/concepts/queue)和[Steering queue](/zh-CN/concepts/queue-steering)。

  </Accordion>
</AccordionGroup>

## 其他

<AccordionGroup>
  <Accordion title='使用 API 密钥时，Anthropic 的默认模型是什么？'>
    凭据和模型选择是相互独立的。设置 `ANTHROPIC_API_KEY`（或在身份验证配置文件中存储 Anthropic API 密钥）会启用身份验证，但实际的默认模型取决于你在 `agents.defaults.model.primary` 中的配置（例如 `anthropic/claude-sonnet-4-6` 或 `anthropic/claude-opus-4-6`）。`No credentials found for profile "anthropic:default"` 表示 Gateway 网关无法在运行中智能体的预期 `auth-profiles.json` 中找到 Anthropic 凭据。
  </Accordion>
</AccordionGroup>

---

仍未解决？请在 [Discord](https://discord.com/invite/clawd) 中提问，或发起 [GitHub 讨论](https://github.com/openclaw/openclaw/discussions)。

## 相关内容

- [首次运行常见问题](/zh-CN/help/faq-first-run) - 安装、新手引导、身份验证、订阅、早期故障
- [模型常见问题](/zh-CN/help/faq-models) - 模型选择、故障转移、身份验证配置文件
- [故障排除](/zh-CN/help/troubleshooting) - 按症状优先进行问题分类
