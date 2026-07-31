---
read_when:
    - 全新安装、新手引导卡住或首次运行错误
    - 选择身份验证方式和提供商订阅方案
    - 无法访问 docs.openclaw.ai，无法打开仪表板，安装卡住
sidebarTitle: First-run FAQ
summary: 常见问题：快速开始和首次运行设置 — 安装、新手引导、身份验证、订阅和初始故障
title: 常见问题：首次运行设置
x-i18n:
    generated_at: "2026-07-26T06:44:13Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: e1c93b89da625ae5f092db854c9b74adc005be75dd913af4bf89ed1a4f35396a
    source_path: help/faq-first-run.md
    workflow: 16
---

快速开始和首次运行问答。有关日常操作、模型、身份验证、会话和故障排查，请参阅主[常见问题](/zh-CN/help/faq)。

## 快速开始和首次运行设置

<AccordionGroup>
  <Accordion title="遇到问题时，最快的解决方法">
    使用能够**查看你的机器**的本地 AI 智能体。大多数“卡住了”的情况都是远程协助者无法检查的**本地配置或环境问题**，因此这种方式比在 Discord 中提问更有效。

    - **Claude Code**：[https://www.anthropic.com/claude-code/](https://www.anthropic.com/claude-code/)
    - **OpenAI Codex**：[https://openai.com/codex/](https://openai.com/codex/)

    通过可修改的（git）安装方式向智能体提供完整的源代码检出，以便它读取代码和文档，并根据你运行的确切版本进行分析：

    ```bash
    curl -fsSL --proto '=https' --tlsv1.2 https://openclaw.ai/install.sh | bash -s -- --install-method git
    ```

    让智能体逐步规划并监督修复过程，然后只执行必要的命令——差异越小，越容易审计。

    寻求帮助时（在 Discord 或 GitHub issue 中）请分享以下输出：

    | 命令 | 显示内容 |
    | --- | --- |
    | `openclaw status` | Gateway 网关/智能体健康状态和基本配置快照 |
    | `openclaw status --all` | 完整的只读诊断，可直接粘贴 |
    | `openclaw models status` | 提供商身份验证和模型可用性 |
    | `openclaw doctor` | 验证并修复常见的配置/状态问题 |
    | `openclaw logs --follow` | 实时日志追踪 |
    | `openclaw gateway status --deep` | 深度 Gateway 网关/配置/插件健康检查 |
    | `openclaw health --verbose` | 详细健康报告 |

    发现了实际 bug 或修复方案？请提交 issue 或 PR：
    [Issues](https://github.com/openclaw/openclaw/issues) /
    [Pull requests](https://github.com/openclaw/openclaw/pulls)。

    快速调试流程：[出现故障时的最初六十秒](/zh-CN/help/faq#first-60-seconds-if-something-is-broken)。
    安装文档：[安装](/zh-CN/install)、[安装程序标志](/zh-CN/install/installer)、[更新](/zh-CN/install/updating)。

  </Accordion>

  <Accordion title="Heartbeat 一直跳过。跳过原因是什么意思？">
    | 跳过原因 | 含义 |
    | --- | --- |
    | `quiet-hours` | 不在配置的活动时段范围内 |
    | `empty-heartbeat-file` | Heartbeat 监视器暂存内容存在，但其中只有空白、注释、标题、围栏或空检查清单框架 |
    | `alerts-disabled` | 所有 Heartbeat 可见性均已关闭（`showOk`、`showAlerts` 和 `useIndicator` 均已禁用） |

    较旧的 Heartbeat `tasks:` 块会通过 `openclaw doctor --fix` 迁移为独立调度的定时任务。

    文档：[Heartbeat](/zh-CN/gateway/heartbeat)、[自动化](/zh-CN/automation)。

  </Accordion>

  <Accordion title="安装和设置 OpenClaw 的推荐方式">
    ```bash
    curl -fsSL --proto '=https' --tlsv1.2 https://openclaw.ai/install.sh | bash
    openclaw onboard --install-daemon
    ```

    从源码安装（贡献者/开发者）：

    ```bash
    git clone https://github.com/openclaw/openclaw.git
    cd openclaw
    pnpm install
    pnpm build
    pnpm ui:build
    openclaw onboard
    ```

    还没有全局安装？请改为运行 `pnpm openclaw onboard`。如果缺少 Control UI 资源，新手引导会尝试自行构建；若失败，则回退到 `pnpm ui:build`。

  </Accordion>

  <Accordion title="完成新手引导后，如何打开仪表板？">
    设置完成后，新手引导会立即在浏览器中打开一个干净的（不含令牌的）仪表板 URL，并在摘要中输出该链接。请保持该标签页打开；如果浏览器没有启动，请在同一台机器上复制并粘贴输出的 URL。
  </Accordion>

  <Accordion title="如何在 localhost 和远程环境中对仪表板进行身份验证？">
    **Localhost（同一台机器）：**

    - 打开 `http://127.0.0.1:18789/`。
    - 如果系统要求共享密钥身份验证，请将配置的令牌或密码粘贴到 Control UI 设置中。
    - 令牌来源：`gateway.auth.token`（或 `OPENCLAW_GATEWAY_TOKEN`）。
    - 密码来源：`gateway.auth.password`（或 `OPENCLAW_GATEWAY_PASSWORD`）。
    - 尚未配置共享密钥？运行 `openclaw doctor --generate-gateway-token`（或 `openclaw doctor --fix --generate-gateway-token`）。

    **不在 localhost 上：**

    - **Tailscale Serve**（推荐）：保持绑定到 loopback，运行 `openclaw gateway --tailscale serve`，然后打开 `https://<magicdns>/`。启用 `gateway.auth.allowTailscale: true` 后，身份标头可满足 Control UI/WebSocket 身份验证要求（无需粘贴共享密钥，前提是 Gateway 网关主机受信任）；HTTP API 仍需要共享密钥身份验证，除非你有意使用专用入口 `none` 或受信任代理 HTTP 身份验证。
      来自同一客户端的并发错误身份验证 Serve 尝试会在失败身份验证限制器记录它们之前串行执行，因此第二次错误重试可能已经显示 `retry later`。
    - **Tailnet 绑定**：运行 `openclaw gateway --bind tailnet --token "<token>"`（或配置密码身份验证），打开 `http://<tailscale-ip>:18789/`，然后在仪表板设置中粘贴匹配的共享密钥。
    - **身份感知反向代理**：将 Gateway 网关置于受信任代理之后，设置 `gateway.auth.mode: "trusted-proxy"`，然后打开代理 URL。同主机 loopback 代理需要显式设置 `gateway.auth.trustedProxy.allowLoopback: true`。
    - **SSH 隧道**：运行 `ssh -N -L 18789:127.0.0.1:18789 user@gateway-host`，然后打开 `http://127.0.0.1:18789/`。通过隧道时仍适用共享密钥身份验证；如果出现提示，请粘贴配置的令牌或密码。

    有关绑定模式和身份验证的详细信息，请参阅[仪表板](/zh-CN/web/dashboard)和 [Web 界面](/zh-CN/web)。

  </Accordion>

  <Accordion title="为什么聊天审批有两种 Exec 审批配置？">
    它们控制不同的层：

    - `approvals.exec`——将审批提示转发到聊天目标。
    - `channels.<channel>.execApprovals`——使该渠道成为 Exec 审批的原生审批客户端。

    主机 Exec 策略仍是真正的审批关卡；聊天配置只控制提示出现的位置以及用户如何作答。

    通常不需要同时使用两者：

    - 如果聊天已经支持命令和回复，则同一聊天中的 `/approve` 可通过共享路径工作。
    - 当受支持的原生渠道可以安全推断审批人时，如果 `channels.<channel>.execApprovals.enabled` 未设置或为 `"auto"`，OpenClaw 会自动启用优先私信的原生审批。
    - 当原生审批卡片/按钮可用时，该 UI 是主要方式；只有当工具结果表明聊天审批不可用时，才提及手动 `/approve` 命令。
    - 仅当提示还必须发送到其他聊天或明确的运维房间时，才使用 `approvals.exec`。
    - 仅当你希望将审批提示发回原始房间/主题时，才使用 `channels.<channel>.execApprovals.target: "channel"` 或 `"both"`。
    - 插件审批是独立的：默认在同一聊天中使用 `/approve`，可选择通过 `approvals.plugin` 转发，并且只有部分原生渠道也会为其保留原生处理方式。

    简而言之：转发用于路由，原生客户端配置用于提供更丰富的渠道专属用户体验。
    请参阅 [Exec 审批](/zh-CN/tools/exec-approvals)。

  </Accordion>

  <Accordion title="需要什么运行时？">
    需要 Node **22.22.3+**、**24.15+** 或 **25.9+**（推荐 Node 24）。`pnpm` 是仓库的包管理器。
    Bun 可以安装依赖项并运行包脚本，但无法运行 OpenClaw CLI 或 Gateway 网关，因为它缺少 `node:sqlite`。
  </Accordion>

  <Accordion title="可以在 Raspberry Pi 上运行吗？">
    可以，但请先检查内存：Pi 5 和 Pi 4（2 GB+）最合适；Pi 3B+（1 GB）可以运行，但速度较慢；不推荐 Pi Zero 2 W（512 MB）。

    | 型号 | 内存 | 适用程度 |
    | --- | --- | --- |
    | Pi 5 | 4/8 GB | 最佳 |
    | Pi 4 | 4 GB | 良好 |
    | Pi 4 | 2 GB | 尚可，需添加交换空间 |
    | Pi 4 | 1 GB | 紧张 |
    | Pi 3B+ | 1 GB | 较慢 |
    | Pi Zero 2 W | 512 MB | 不推荐 |

    绝对最低要求：1 GB 内存、1 个核心、500 MB 可用磁盘空间、64 位操作系统。由于 Pi 仅运行 Gateway 网关（模型通过云 API 调用），即使配置一般的 Pi 也能承受负载。

    小型 Pi/VPS 也可以只托管 Gateway 网关，同时将笔记本电脑/手机作为**节点**配对，用于本地屏幕/摄像头/canvas 或命令执行。请参阅[节点](/zh-CN/nodes)。

    完整设置演练：[Raspberry Pi](/zh-CN/install/raspberry-pi)。

  </Accordion>

  <Accordion title="安装到 Raspberry Pi 时有什么建议？">
    - 使用 **64 位**操作系统；不要使用 32 位 Raspberry Pi OS。
    - 在 2 GB 或更小内存的开发板上添加交换空间。
    - 为提高性能和使用寿命，优先使用 **USB SSD**，而不是 SD 卡。
    - 优先使用可修改的（git）安装方式，以便查看日志并快速更新。
    - 开始时不要启用渠道/Skills，之后逐一添加。
    - 异常的二进制文件错误（“exec format error”）通常是因为某个可选 Skills 工具缺少 ARM64 构建版本。

    完整指南：[Raspberry Pi](/zh-CN/install/raspberry-pi)。另请参阅 [Linux](/zh-CN/platforms/linux)。

  </Accordion>

  <Accordion title="卡在 wake up my friend / 新手引导无法孵化。怎么办？">
    该屏幕依赖于 Gateway 网关可访问且身份验证成功。配置模型提供商后，TUI 还会在首次孵化时自动发送“Wake up, my friend!”。如果跳过了模型/身份验证设置，新手引导会显示“Model auth missing”提示，并打开 TUI，但不会发送任何内容——请使用 `openclaw configure --section model` 添加提供商。
    如果你看到唤醒消息但**没有回复**，并且令牌数一直为 0，则说明智能体从未运行。

    1. 重启 Gateway 网关：

    ```bash
    openclaw gateway restart
    ```

    2. 检查状态和身份验证：

    ```bash
    openclaw status
    openclaw models status
    openclaw logs --follow
    ```

    3. 仍然卡住？运行：

    ```bash
    openclaw doctor
    ```

    如果 Gateway 网关位于远程主机，请确认隧道/Tailscale 连接已建立，并且 UI 指向正确的 Gateway 网关。请参阅[远程访问](/zh-CN/gateway/remote)。

  </Accordion>

  <Accordion title="无需重新进行新手引导，能否将设置迁移到新机器？">
    可以。复制**状态目录**和**工作区**，然后运行一次 Doctor：

    1. 在新机器上安装 OpenClaw。
    2. 从旧机器复制 `$OPENCLAW_STATE_DIR`（默认值：`~/.openclaw`）。
    3. 复制你的工作区（默认值：`~/.openclaw/workspace`）。
    4. 运行 `openclaw doctor` 并重启 Gateway 网关服务。

    只要复制了**两个**位置，就会保留配置、身份验证配置文件、WhatsApp 凭据、会话和记忆，让你的机器人保持完全相同。在远程模式下，Gateway 网关主机拥有会话存储和工作区。

    **重要提示：**如果只将工作区提交/推送到 GitHub，你备份的只是**记忆和引导文件**，不包括会话历史记录或身份验证信息。后两者位于 `~/.openclaw/` 下（例如 `~/.openclaw/agents/<agentId>/agent/openclaw-agent.sqlite`）。

    相关内容：[迁移](/zh-CN/install/migrating)、[文件在磁盘上的存储位置](/zh-CN/help/faq#where-things-live-on-disk)、
    [Agent 工作区](/zh-CN/concepts/agent-workspace)、[Doctor](/zh-CN/gateway/doctor)、
    [远程模式](/zh-CN/gateway/remote)。

  </Accordion>

  <Accordion title="在哪里查看最新版本的新功能？">
    查看 GitHub 更新日志：
    [https://github.com/openclaw/openclaw/blob/main/CHANGELOG.md](https://github.com/openclaw/openclaw/blob/main/CHANGELOG.md)

    最新条目位于顶部。如果顶部章节是 **Unreleased**，则下一个带日期的章节是最新发布的版本。条目分组在**亮点**、**变更**和**修复**下（需要时还会有文档/其他章节）。

  </Accordion>

  <Accordion title="无法访问 docs.openclaw.ai（SSL 错误）">
    某些 Comcast/Xfinity 连接会被 Xfinity Advanced Security 错误地阻止访问 `docs.openclaw.ai`。请将其禁用或将 `docs.openclaw.ai` 加入允许列表，然后重试。请帮助我们解除阻止：[https://spa.xfinity.com/check_url_status](https://spa.xfinity.com/check_url_status)。

    仍然受阻？文档已镜像到 GitHub：
    [https://github.com/openclaw/openclaw/tree/main/docs](https://github.com/openclaw/openclaw/tree/main/docs)

  </Accordion>

  <Accordion title="稳定版和测试版有什么区别">
    **稳定版**和**测试版**是 **npm dist-tag**，并非不同的代码分支：

    - `latest` = 稳定版
    - `beta` = 用于测试的早期构建版本（当测试版不存在或比当前稳定版本更旧时，回退到 `latest`）

    稳定版本通常会先发布到**测试版**，然后通过明确的提升步骤，
    在不更改版本号的情况下将同一版本移至 `latest`。维护者
    也可以直接发布到 `latest`。因此，提升后测试版和稳定版可能指向
    **同一版本**。

    查看变更内容：[CHANGELOG.md](https://github.com/openclaw/openclaw/blob/main/CHANGELOG.md)。

    有关一行命令安装方式以及测试版与开发版之间的区别，请参阅下一个折叠面板。

  </Accordion>

  <Accordion title="如何安装测试版？测试版和开发版有什么区别？">
    **测试版**对应 npm dist-tag `beta`（提升后可能与 `latest` 相同）。
    **开发版**是 `main`（git）不断变化的最新版本；发布到 npm 时使用 dist-tag `dev`。

    一行命令（macOS/Linux）：

    ```bash
    curl -fsSL --proto '=https' --tlsv1.2 https://openclaw.ai/install.sh | bash -s -- --beta
    ```

    ```bash
    curl -fsSL --proto '=https' --tlsv1.2 https://openclaw.ai/install.sh | bash -s -- --install-method git
    ```

    Windows 安装程序（PowerShell）：`iwr -useb https://openclaw.ai/install.ps1 | iex`

    更多详情：[开发频道](/zh-CN/install/development-channels)和[安装程序标志](/zh-CN/install/installer)。

  </Accordion>

  <Accordion title="如何试用最新版本？">
    有两种方式：

    1. **开发频道（已有安装）：**

    ```bash
    openclaw update --channel dev
    ```

    这会切换到 `main` 的 git 检出，在上游分支上进行变基和构建，并从该检出
    安装 CLI。

    2. **可修改的（git）安装（全新机器）：**

    ```bash
    curl -fsSL --proto '=https' --tlsv1.2 https://openclaw.ai/install.sh | bash -s -- --install-method git
    ```

    建议手动克隆：

    ```bash
    git clone https://github.com/openclaw/openclaw.git
    cd openclaw
    pnpm install
    pnpm build
    ```

    文档：[更新](/zh-CN/cli/update)、[开发频道](/zh-CN/install/development-channels)、[安装](/zh-CN/install)。

  </Accordion>

  <Accordion title="安装和新手引导通常需要多长时间？">
    大致参考：

    - **安装：**2-5 分钟。
    - **快速开始新手引导：**几分钟（local loopback Gateway 网关、自动令牌、默认工作区）。
    - **高级/完整新手引导：**如果提供商登录、渠道配对、守护进程安装、网络下载或 Skills 需要额外设置，则耗时更长。

    向导会预先显示这一时间安排。可以跳过可选步骤，稍后通过
    `openclaw configure` 返回继续设置。

    卡住了？请参阅上面的[我卡住了](#quick-start-and-first-run-setup)。

  </Accordion>

  <Accordion title="安装程序卡住了？如何获取更多反馈？">
    使用 `--verbose` 重新运行：

    ```bash
    curl -fsSL --proto '=https' --tlsv1.2 https://openclaw.ai/install.sh | bash -s -- --verbose
    curl -fsSL --proto '=https' --tlsv1.2 https://openclaw.ai/install.sh | bash -s -- --beta --verbose
    curl -fsSL --proto '=https' --tlsv1.2 https://openclaw.ai/install.sh | bash -s -- --install-method git --verbose
    ```

    `install.ps1` 没有专用的详细输出开关；请改为使用 `Set-PSDebug -Trace 1` /
    `-Trace 0` 包装它。完整标志参考：[安装程序标志](/zh-CN/install/installer)。

  </Accordion>

  <Accordion title="Windows 安装提示找不到 git 或无法识别 openclaw">
    Windows 上常见的两个问题：

    **1) npm 错误 spawn git / 找不到 git**

    - 安装 **Git for Windows**，确保 `git` 位于 PATH 中。
    - 关闭并重新打开 PowerShell，然后再次运行安装程序。

    **2) 安装后无法识别 openclaw**

    - npm 全局二进制文件夹不在 PATH 中。
    - 检查路径：`npm config get prefix`。
    - 将该目录添加到用户 PATH（无需 `\bin` 后缀；在大多数系统上，该目录是 `%AppData%\npm`）。
    - 关闭并重新打开 PowerShell。

    更喜欢桌面应用？请使用 **Windows Hub**。对于仅使用终端的设置方式，PowerShell
    安装程序和 WSL2 Gateway 网关路径均受支持。文档：[Windows](/zh-CN/platforms/windows)。

  </Accordion>

  <Accordion title="Windows exec 输出显示乱码中文，该怎么办？">
    这通常是原生 Windows shell 中的控制台代码页不匹配所致。

    症状：`system.run`/`exec` 输出中的中文显示为乱码；同一命令
    在另一个终端配置文件中显示正常。

    PowerShell 中的解决方法：

    ```powershell
    chcp 65001
    [Console]::InputEncoding = [System.Text.UTF8Encoding]::new($false)
    [Console]::OutputEncoding = [System.Text.UTF8Encoding]::new($false)
    $OutputEncoding = [System.Text.UTF8Encoding]::new($false)
    ```

    然后重启 Gateway 网关并重试：

    ```powershell
    openclaw gateway restart
    ```

    在最新版本的 OpenClaw 上仍能复现？请跟踪/报告：[Issue #30640](https://github.com/openclaw/openclaw/issues/30640)。

  </Accordion>

  <Accordion title="文档没有解答我的问题，如何获得更好的答案？">
    使用可修改的（git）安装，以便在本地获得完整源代码和文档，然后
    **在该文件夹中**询问你的机器人（或 Claude/Codex），使其能够读取仓库并给出准确答案。

    ```bash
    curl -fsSL --proto '=https' --tlsv1.2 https://openclaw.ai/install.sh | bash -s -- --install-method git
    ```

    更多详情：[安装](/zh-CN/install)和[安装程序标志](/zh-CN/install/installer)。

  </Accordion>

  <Accordion title="如何在 Linux 上安装 OpenClaw？">
    - Linux 快速路径 + 服务安装：[Linux](/zh-CN/platforms/linux)。
    - 完整演练：[入门指南](/zh-CN/start/getting-started)。
    - 安装程序 + 更新：[安装和更新](/zh-CN/install/updating)。

  </Accordion>

  <Accordion title="如何在 VPS 上安装 OpenClaw？">
    任何 Linux VPS 均可使用。在服务器上安装，然后通过 SSH/Tailscale 访问 Gateway 网关。

    指南：[exe.dev](/zh-CN/install/exe-dev)、[Hetzner](/zh-CN/install/hetzner)、[Fly.io](/zh-CN/install/fly)。
    远程访问：[Gateway 远程访问](/zh-CN/gateway/remote)。

  </Accordion>

  <Accordion title="云端/VPS 安装指南在哪里？">
    包含常见提供商的托管中心：

    - [VPS 托管](/zh-CN/vps)（在一处查看所有提供商）
    - [Fly.io](/zh-CN/install/fly)
    - [Hetzner](/zh-CN/install/hetzner)
    - [exe.dev](/zh-CN/install/exe-dev)

    在云端，**Gateway 网关在服务器上运行**，你可以从笔记本电脑/手机
    通过 Control UI（或 Tailscale/SSH）访问。状态和工作区存储在服务器上，因此
    应将该主机视为事实来源并进行备份。

    将**节点**（Mac/iOS/Android/无头设备）与该云端 Gateway 网关配对，这样 Gateway 网关保持在
    云端运行时，仍可使用笔记本电脑上的本地屏幕/摄像头/canvas 或执行命令。

    中心：[平台](/zh-CN/platforms)。远程访问：[Gateway 远程访问](/zh-CN/gateway/remote)。
    节点：[节点](/zh-CN/nodes)、[节点 CLI](/zh-CN/cli/nodes)。

  </Accordion>

  <Accordion title="可以让 OpenClaw 自行更新吗？">
    可以，但不建议这样做。更新流程可能会重启 Gateway 网关（导致
    活跃会话断开），可能需要干净的 git 检出，并且可能提示确认。
    由操作员在 shell 中运行更新更安全。

    ```bash
    openclaw update
    openclaw update status
    openclaw update --channel stable|extended-stable|beta|dev
    openclaw update --tag <dist-tag|version>
    openclaw update --no-restart
    ```

    通过智能体实现自动化：

    ```bash
    openclaw update --yes --no-restart
    openclaw gateway restart
    ```

    文档：[更新](/zh-CN/cli/update)、[更新 OpenClaw](/zh-CN/install/updating)。

  </Accordion>

  <Accordion title="新手引导实际会执行哪些操作？">
    `openclaw onboard` 是推荐的设置路径。在**本地模式**下，它会引导完成：

    1. **模型/身份验证**——提供商 OAuth、API 密钥或手动身份验证（包括 LM Studio 等本地选项）；选择默认模型。
    2. **工作区**——位置 + 引导文件。
    3. **Gateway 网关**——端口、绑定地址、身份验证模式、Tailscale 暴露方式。
    4. **渠道**——内置及官方插件聊天渠道：iMessage、Discord、Feishu、Google Chat、Mattermost、Microsoft Teams、QQ Bot、Signal、Slack、Telegram、WhatsApp 等。
    5. **守护进程**——LaunchAgent（macOS）、systemd 用户单元（Linux/WSL2）或原生 Windows 计划任务。
    6. **健康检查**——启动 Gateway 网关并验证其正在运行。
    7. **Skills**——安装推荐的 Skills 和可选依赖项。

    它会预先说明预计耗时，并在配置的模型未知或
    缺少身份验证时发出警告。完整说明：[新手引导（CLI）](/zh-CN/start/wizard)。

  </Accordion>

  <Accordion title="运行 OpenClaw 是否需要 Claude 或 OpenAI 订阅？">
    不需要。可以使用 **API 密钥**（Anthropic/OpenAI/其他提供商）或**纯本地模型**
    运行 OpenClaw，让数据保留在你的设备上。订阅（Claude Pro/Max、ChatGPT/Codex）只是
    对这些提供商进行身份验证的可选方式。

    对于 Anthropic：使用 **API 密钥**时按标准的用量计费；**Claude CLI**
    会复用同一主机上现有的 Claude Code 登录。Anthropic 目前将
    Claude CLI 的非交互式 `claude -p` 路径视为 Agent SDK/程序化用法，
    仍会占用订阅方案的限额——依赖订阅行为之前，请查阅 Anthropic 当前的计费
    文档。对于长期运行的 Gateway 网关主机和共享自动化，Anthropic API 密钥是更可预测的选择。

    完全支持使用 OpenAI Codex OAuth（ChatGPT/Codex 订阅）验证智能体模型。
    OpenClaw 还支持托管式订阅选项，包括 **Qwen Cloud
    Coding Plan**、**MiniMax Coding Plan** 和 **Z.AI / GLM Coding Plan**。

    文档：[Anthropic](/zh-CN/providers/anthropic)、[OpenAI](/zh-CN/providers/openai)、
    [Qwen Cloud](/zh-CN/providers/qwen)、[MiniMax](/zh-CN/providers/minimax)、[Z.AI (GLM)](/zh-CN/providers/zai)、
    [本地模型](/zh-CN/gateway/local-models)、[Models](/zh-CN/concepts/models)。

  </Accordion>

  <Accordion title="没有 API 密钥，可以使用 Claude Max 订阅吗？">
    可以。OpenClaw 支持为 Pro/Max/Team/Enterprise 方案复用 Claude CLI。Anthropic
    目前将 OpenClaw 使用的 `claude -p` 路径视为受订阅方案限额约束的使用，
    而不是单独的免费额度——有关当前计费详情以及 Anthropic 自有支持文章的链接，请参阅
    [Anthropic](/zh-CN/providers/anthropic)。要获得最可预测的服务器端设置，请改用
    Anthropic API 密钥。
  </Accordion>

  <Accordion title="是否支持 Claude 订阅身份验证（Claude Pro 或 Max）？">
    支持，通过复用 Claude CLI 实现。Anthropic 对 `claude -p`/Agent SDK 用法的计费方式
    曾随时间变化；在依赖特定计费行为之前，请参阅 [Anthropic](/zh-CN/providers/anthropic)
    了解当前状态以及指向 Anthropic 支持文章的带日期链接。

    Anthropic setup-token 身份验证仍是受支持的令牌路径，但 OpenClaw 会在可用时优先
    复用 Claude CLI 和 `claude -p`。对于生产或多用户
    工作负载，Anthropic API key 仍是更安全、更可预测的选择。其他
    订阅式托管选项：[OpenAI](/zh-CN/providers/openai)、[Qwen Cloud](/zh-CN/providers/qwen)、
    [MiniMax](/zh-CN/providers/minimax)、[Z.AI (GLM)](/zh-CN/providers/zai)。

  </Accordion>

</AccordionGroup>

<a id="why-am-i-seeing-http-429-ratelimiterror-from-anthropic"></a>

<AccordionGroup>
  <Accordion title="为什么会看到来自 Anthropic 的 HTTP 429 rate_limit_error？">
    当前时间窗口的 **Anthropic 配额/速率限制** 已耗尽。使用 **Claude
    CLI** 时，请等待窗口重置或升级套餐。使用 **Anthropic API key** 时，
    请在 Anthropic Console 中检查用量/账单，并按需提高限制。

    如果消息明确为 `Extra usage is required for long context requests`，
    则请求正在尝试使用 Anthropic 的 1M 上下文窗口（支持 GA 的 1M Claude 4.x
    模型，或旧版 `params.context1m: true` 配置），而你当前的凭据不符合
    长上下文计费资格。

    设置一个**回退模型**，使 OpenClaw 在提供商受到速率限制时仍能继续回复。
    请参阅[模型](/zh-CN/cli/models)、[OAuth](/zh-CN/concepts/oauth)和
    [Anthropic 429：长上下文需要额外用量](/zh-CN/gateway/troubleshooting#anthropic-429-extra-usage-required-for-long-context)。

  </Accordion>

  <Accordion title="是否支持 AWS Bedrock？">
    是。OpenClaw 内置了 **Amazon Bedrock (Converse)** 提供商。如果存在 AWS 环境
    标记（`AWS_ACCESS_KEY_ID`、`AWS_PROFILE`、`AWS_BEARER_TOKEN_BEDROCK`），
    OpenClaw 会自动启用隐式 Bedrock 提供商以发现模型；否则，
    请设置 `plugins.entries.amazon-bedrock.config.discovery.enabled: true` 或手动添加
    提供商条目。请参阅 [Amazon Bedrock](/zh-CN/providers/bedrock)和[模型提供商](/zh-CN/providers/models)。
    如果你偏好托管密钥流程，也仍可选择在 Bedrock 前使用兼容 OpenAI 的代理。
  </Accordion>

  <Accordion title="Codex 身份验证如何工作？">
    OpenClaw 通过 OAuth（登录 ChatGPT）支持 **OpenAI Codex**。在没有主模型的
    全新设置中，会使用确切的 `openai/gpt-5.6-sol` 进行
    ChatGPT/Codex 订阅身份验证，并通过原生 Codex app-server 执行。
    重新进行身份验证时会保留现有的显式模型，包括
    `openai/gpt-5.5`。如果 Codex 工作区未提供 GPT-5.6，请显式选择
    `openai/gpt-5.5`；OpenClaw 不会静默降级。旧版
    Codex 前缀模型引用属于旧版配置，由 `openclaw doctor
    --fix` 修复。直接使用 OpenAI API key 仍适用于非 Agent 的 OpenAI
    API 界面；通过有序的 `openai` API key 配置文件，也适用于 Agent
    模型。请参阅[模型提供商](/zh-CN/concepts/model-providers)和
    [新手引导（CLI）](/zh-CN/start/wizard)。
  </Accordion>

  <Accordion title="为什么 OpenClaw 仍会提及旧版 OpenAI Codex 前缀？">
    `openai` 是 OpenAI API key 和
    ChatGPT/Codex OAuth 当前共用的提供商与身份验证配置文件 ID——OpenAI Codex 已合并到其中。你仍可能在旧版配置
    和迁移警告中看到旧版 `openai-codex` 前缀：

    - `openai/gpt-5.6-sol` = 使用原生 Codex 运行时处理 Agent 轮次的全新 ChatGPT/Codex 订阅设置。
    - `openai/gpt-5.5` = 为现有配置或无法访问 GPT-5.6 的账户提供的显式受支持选项。
    - 旧版 `openai-codex/*` 模型引用 = 由 `openclaw doctor --fix` 修复的旧版路由。
    - `openai/gpt-5.5` 加上有序的 `openai` API key 配置文件 = OpenAI Agent 模型的 API key 身份验证。
    - 旧版 `openai-codex` 身份验证配置文件 ID = 由 `openclaw doctor --fix` 迁移的旧版 ID。

    想使用 OpenAI Platform 直接计费？请设置 `OPENAI_API_KEY`。想使用 ChatGPT/Codex
    订阅身份验证？请运行 `openclaw models auth login --provider openai`。请将
    模型引用保留在规范的 `openai/*` 提供商下。全新订阅
    设置会使用确切的 `openai/gpt-5.6-sol`；Doctor 会修复带旧版 Codex 前缀的
    引用，但不会升级显式选择的 `openai/gpt-5.5`。

  </Accordion>

  <Accordion title="为什么 Codex OAuth 限制可能与 ChatGPT 网页版不同？">
    Codex OAuth 使用由 OpenAI 管理且取决于套餐的配额窗口，即使是同一个账户，也可能与
    ChatGPT 网站/应用的体验不同。

    `openclaw models status` 会显示当前可见的提供商用量/配额窗口，但
    不会凭空生成权限，也不会将 ChatGPT 网页版权益转换为直接 API 访问。若要使用
    OpenAI Platform 的直接计费/限制路径，请配合 API key 使用 `openai/*`。

  </Accordion>

  <Accordion title="是否支持 OpenAI 订阅身份验证（Codex OAuth）？">
    是，完全支持。OpenAI 明确允许在 OpenClaw 之类的外部
    工具/工作流中使用订阅 OAuth。新手引导可以为你运行 OAuth 流程。

    请参阅 [OAuth](/zh-CN/concepts/oauth)、[模型提供商](/zh-CN/concepts/model-providers)和[新手引导（CLI）](/zh-CN/start/wizard)。

  </Accordion>

  <Accordion title="如何设置 Gemini CLI OAuth？">
    Gemini CLI 使用**插件身份验证流程**，而不是在 `openclaw.json` 中设置客户端 ID 或密钥。

    1. 在本地安装 Gemini CLI，确保 `gemini` 位于 `PATH` 中：
       - Homebrew：`brew install gemini-cli`
       - npm：`npm install -g @google/gemini-cli`
    2. 启用插件：`openclaw plugins enable google`
    3. 登录：`openclaw models auth login --provider google-gemini-cli --set-default`
    4. 登录后的默认模型：`google/gemini-3.1-pro-preview`（运行时为 `google-gemini-cli`）
    5. 登录后请求失败？请在 Gateway 网关主机上设置 `GOOGLE_CLOUD_PROJECT` 或 `GOOGLE_CLOUD_PROJECT_ID`，然后重试。

    OAuth 令牌存储在 Gateway 网关主机上的身份验证配置文件中。详情请参阅：[Google](/zh-CN/providers/google)、[模型提供商](/zh-CN/concepts/model-providers)。

  </Accordion>

  <Accordion title="本地模型适合随意聊天吗？">
    通常不适合。OpenClaw 需要大上下文和强大的安全能力；小显存卡会截断上下文，
    并绕过提供商侧的安全过滤器。如果必须使用，请在本地运行你能运行的**最大**
    模型构建版本（LM Studio）——请参阅[本地模型](/zh-CN/gateway/local-models)。较小或量化的
    模型会增加提示注入风险——请参阅[安全](/zh-CN/gateway/security)。
  </Accordion>

  <Accordion title="如何让托管模型流量保留在特定区域？">
    请选择固定区域的端点。OpenRouter 为 MiniMax、Kimi
    和 GLM 提供美国托管选项；选择美国托管变体即可将数据保留在该区域。你仍可以使用
    `models.mode: "merge"` 同时列出 Anthropic/OpenAI，这样既可保留回退选项，
    又能遵守所选提供商的区域限制。
  </Accordion>

  <Accordion title="安装 OpenClaw 必须购买 Mac Mini 吗？">
    不需要。OpenClaw 可在 macOS 或 Linux 上运行（Windows 通过 WSL2）。Mac mini 是常见的
    常开主机选择，但小型 VPS、家庭服务器或 Raspberry Pi 级别的设备也可以。

    只有使用 **macOS 专属工具**时才需要 Mac。对于 iMessage，请在任何已登录“信息”应用的 Mac 上
    使用 [iMessage](/zh-CN/channels/imessage) 和 `imsg`——如果 Gateway 网关运行在 Linux 或其他位置，
    请将 `channels.imessage.cliPath` 设置为一个 SSH 包装器，由它在该 Mac 上运行 `imsg`。对于其他
    macOS 专属工具，请在 Mac 上运行 Gateway 网关，或配对一个 macOS 节点。

    文档：[iMessage](/zh-CN/channels/imessage)、[节点](/zh-CN/nodes)、[Mac 远程模式](/zh-CN/platforms/mac/remote)。

  </Accordion>

  <Accordion title="支持 iMessage 是否需要 Mac mini？">
    你需要**某台 macOS 设备**登录“信息”应用——不一定是 Mac mini，任何
    Mac 都可以。请使用 [iMessage](/zh-CN/channels/imessage) 和 `imsg`；Gateway 网关可以运行在该
    Mac 上，也可以运行在其他位置并使用 SSH 包装器 `cliPath`。

    常见设置：

    - Gateway 网关运行在 Linux/VPS 上，将 `channels.imessage.cliPath` 设置为一个 SSH 包装器，由它在已登录“信息”应用的 Mac 上运行 `imsg`。
    - 所有组件都运行在同一台 Mac 上，这是最简单的单机设置。

    文档：[iMessage](/zh-CN/channels/imessage)、[节点](/zh-CN/nodes)、[Mac 远程模式](/zh-CN/platforms/mac/remote)。

  </Accordion>

  <Accordion title="如果购买 Mac mini 运行 OpenClaw，可以将它连接到 MacBook Pro 吗？">
    可以。**Mac mini 可以运行 Gateway 网关**，而 MacBook Pro 作为**节点**
    （配套设备）连接。节点不运行 Gateway 网关，而是提供该设备上的
    屏幕、摄像头、画布和 `system.run` 等能力。

    常见模式：Gateway 网关运行在常开的 Mac mini 上；MacBook Pro 运行 macOS 应用或
    节点主机并与 Gateway 网关配对。使用 `openclaw nodes status` / `openclaw nodes list` 检查。

    文档：[节点](/zh-CN/nodes)、[节点 CLI](/zh-CN/cli/nodes)。

  </Accordion>

  <Accordion title="可以使用 Bun 吗？">
    你可以使用 Bun 安装依赖项或运行软件包脚本。OpenClaw CLI 和
    Gateway 网关需要 **Node**，因为规范状态存储使用 `node:sqlite`；Bun
    不提供该 API。
  </Accordion>

  <Accordion title="Telegram：allowFrom 中应填写什么？">
    `channels.telegram.allowFrom` 是**真人发送者的 Telegram 用户 ID**（数字），
    不是 Bot 用户名。设置流程仅要求数字用户 ID；`openclaw doctor --fix`
    可以尝试解析旧版 `@username` 条目。

    更安全的方式（不使用第三方 Bot）：向你的 Bot 发送私信，运行 `openclaw logs --follow`，读取 `from.id`。

    官方 Bot API：向你的 Bot 发送私信，调用 `https://api.telegram.org/bot<bot_token>/getUpdates`，读取 `message.from.id`。

    第三方方式（隐私性较低）：向 `@userinfobot` 或 `@getidsbot` 发送私信。

    请参阅 [Telegram 访问控制](/zh-CN/channels/telegram#access-control-and-activation)。

  </Accordion>

  <Accordion title="多个人能否使用同一个 WhatsApp 号码连接不同的 OpenClaw 实例？">
    可以，通过**多 Agent 路由**实现。将每个发送者的 WhatsApp 私信（`peer: { kind: "direct", id: "+15551234567" }`）绑定到不同的 `agentId`，让每个人拥有自己的工作区和会话存储。回复仍来自**同一个 WhatsApp 账户**；私信访问控制（`channels.whatsapp.dmPolicy` / `channels.whatsapp.allowFrom`）对每个账户全局生效。请参阅[多 Agent 路由](/zh-CN/concepts/multi-agent)和 [WhatsApp](/zh-CN/channels/whatsapp)。
  </Accordion>

  <Accordion title='能否同时运行一个“快速聊天”Agent 和一个“使用 Opus 编程”的 Agent？'>
    可以。使用多 Agent 路由：为每个 Agent 设置各自的默认模型，然后将入站
    路由（提供商账户或特定对端）绑定到相应 Agent。配置示例：
    [多 Agent 路由](/zh-CN/concepts/multi-agent)。另请参阅[模型](/zh-CN/concepts/models)和
    [配置](/zh-CN/gateway/configuration)。
  </Accordion>

  <Accordion title="Homebrew 能在 Linux 上运行吗？">
    可以，通过 Linuxbrew：

    ```bash
    /bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
    echo 'eval "$(/home/linuxbrew/.linuxbrew/bin/brew shellenv)"' >> ~/.profile
    eval "$(/home/linuxbrew/.linuxbrew/bin/brew shellenv)"
    brew install <formula>
    ```

    通过 systemd 运行 OpenClaw 时：请确保服务 PATH 包含
    `/home/linuxbrew/.linuxbrew/bin`（或你的 brew 前缀），以便通过 `brew` 安装的工具
    能在非登录 Shell 中解析。近期构建还会在 Linux
    systemd 服务中预置常见的用户二进制目录（例如 `~/.local/bin`、`~/.npm-global/bin`、
    `~/.local/share/pnpm`、`~/.bun/bin`），并在设置后采用 `PNPM_HOME`、`NPM_CONFIG_PREFIX`、
    `BUN_INSTALL`、`VOLTA_HOME`、`ASDF_DATA_DIR`、`NVM_DIR` 和 `FNM_DIR`。

  </Accordion>

  <Accordion title="可修改的 git 安装与 npm 安装之间的区别">
    - **可修改（git）安装：**完整的源代码检出，可编辑，最适合贡献者。你可以在本地构建并修改代码/文档。
    - **npm 安装：**全局 CLI 安装，不含代码仓库，最适合“安装后直接运行”。更新来自 npm dist-tag。

    文档：[入门指南](/zh-CN/start/getting-started)、[更新](/zh-CN/install/updating)。

  </Accordion>

  <Accordion title="以后可以在 npm 和 git 安装方式之间切换吗？">
    可以，在现有安装中使用 `openclaw update --channel ...` 即可。此操作**不会
    删除你的数据**——只会更改 OpenClaw 代码的安装方式。状态数据（`~/.openclaw`）和
    工作区（`~/.openclaw/workspace`）均保持不变。

    从 npm 切换到 git：

    ```bash
    openclaw update --channel dev
    ```

    从 git 切换到 npm：

    ```bash
    openclaw update --channel stable
    ```

    添加 `--dry-run` 可先预览计划中的模式切换。更新程序会运行 Doctor
    后续操作、刷新目标频道的插件源，并重启 Gateway 网关，
    除非传入 `--no-restart`。

    安装程序也可以强制使用任一模式：

    ```bash
    curl -fsSL --proto '=https' --tlsv1.2 https://openclaw.ai/install.sh | bash -s -- --install-method git
    curl -fsSL --proto '=https' --tlsv1.2 https://openclaw.ai/install.sh | bash -s -- --install-method npm
    ```

    备份提示：[磁盘上的内容存储位置](/zh-CN/help/faq#where-things-live-on-disk)。

  </Accordion>

  <Accordion title="应该在笔记本电脑还是 VPS 上运行 Gateway 网关？">
    想要 24/7 可靠性？请使用 **VPS**。想要最省事，并且可以接受
    休眠和重启？请在本地运行。

    **笔记本电脑（本地 Gateway 网关）**

    - **优点：**无需服务器费用，可直接访问本地文件，拥有可见的浏览器窗口。
    - **缺点：**休眠或网络中断会导致连接断开，操作系统更新或重启会造成中断，必须保持唤醒状态。

    **VPS / 云端**

    - **优点：**全天候运行，网络稳定，不受笔记本电脑休眠影响，更易于持续运行。
    - **缺点：**通常无图形界面（请使用截图），只能远程访问文件，更新需要使用 SSH。

    WhatsApp、Telegram、Slack、Mattermost 和 Discord 均可在 VPS 上正常运行——真正的
    取舍在于使用无头浏览器还是可见窗口。请参阅[浏览器](/zh-CN/tools/browser)。

    默认建议：如果以前遇到过 Gateway 网关断开连接，请使用 VPS；如果正在主动使用 Mac，
    并且需要访问本地文件或使用可见浏览器界面进行自动化操作，则本地运行很合适。

  </Accordion>

  <Accordion title="在专用计算机上运行 OpenClaw 有多重要？">
    这不是必需的，但为了提高可靠性和隔离性，建议这样做。

    - **专用主机（VPS/Mac mini/Raspberry Pi）：**全天候运行，较少因休眠或重启而中断，权限更清晰，更易于持续运行。
    - **共用笔记本电脑/台式机：**适合测试和主动使用，但计算机休眠或更新时可能会暂停运行。

    兼得两者优势的方法：将 Gateway 网关保留在专用主机上，并将笔记本电脑配对为
    **节点**，以使用本地屏幕、摄像头和 Exec 工具。请参阅[节点](/zh-CN/nodes)和[安全性](/zh-CN/gateway/security)。

  </Accordion>

  <Accordion title="VPS 的最低配置要求和推荐操作系统是什么？">
    - **绝对最低配置：**1 个 vCPU、1 GB 内存、约 500 MB 磁盘空间。
    - **推荐配置：**1-2 个 vCPU、2 GB 以上内存，以留出余量（用于日志、媒体和多个渠道）。节点工具和浏览器自动化可能会消耗大量资源。

    操作系统：**Ubuntu LTS**（或任何现代 Debian/Ubuntu）——这是经过最充分测试的 Linux 安装路径。

    文档：[Linux](/zh-CN/platforms/linux)、[VPS 托管](/zh-CN/vps)。

  </Accordion>

  <Accordion title="可以在虚拟机中运行 OpenClaw 吗？有哪些要求？">
    可以。将虚拟机视同 VPS：它需要始终保持运行、可访问，并且拥有足够的内存
    供 Gateway 网关和启用的所有渠道使用。

    - **绝对最低配置：**1 个 vCPU、1 GB 内存。
    - **推荐配置：**如果使用多个渠道、浏览器自动化或媒体工具，建议配备 2 GB 以上内存。
    - **操作系统：**Ubuntu LTS 或其他现代 Debian/Ubuntu。

    在 Windows 上，使用 **Windows Hub** 进行桌面设置；也可以使用 WSL2 运行 Linux 风格的 Gateway 网关虚拟机，
    以获得广泛的工具兼容性。请参阅 [Windows](/zh-CN/platforms/windows)、[VPS 托管](/zh-CN/vps)。
    在虚拟机中运行 macOS：请参阅 [macOS 虚拟机](/zh-CN/install/macos-vm)。

  </Accordion>
</AccordionGroup>

## 相关内容

- [常见问题](/zh-CN/help/faq)——主要常见问题（模型、会话、Gateway 网关、安全性等）
- [安装概览](/zh-CN/install)
- [入门指南](/zh-CN/start/getting-started)
- [故障排查](/zh-CN/help/troubleshooting)
