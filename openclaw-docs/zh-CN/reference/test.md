---
read_when:
    - 运行或修复测试
summary: 如何在本地运行测试（vitest），以及何时使用强制/覆盖率模式
title: 测试
x-i18n:
    generated_at: "2026-07-26T07:01:40Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 391185703e853bb523e1396eb22da4693d10d47b1644d3b2a51707d329f67dae
    source_path: reference/test.md
    workflow: 16
---

- 完整测试工具包（测试套件、实时测试、Docker）：[测试](/zh-CN/help/testing)
- 更新和插件包验证：[更新和插件测试](/zh-CN/help/testing-updates-plugins)

## Agent 默认行为

仅当源代码可信且现有依赖项已安装完毕时，Agent 会话才会在本地运行一个或少量聚焦测试及低成本静态检查。绝不在本地执行不受信任的仓库工具。较大型测试套件、包含类型检查/代码检查并行任务的变更门禁、构建、Docker、软件包测试通道、E2E、实时验证和跨平台验证均通过 Crabbox 远程运行。可信维护者的重型验证默认使用 Blacksmith Testbox。配置的 Testbox 工作流会注入凭据，因此不受信任的贡献者代码或 fork 代码必须改用无密钥的 fork CI 或经过清理的 AWS 直连 Crabbox。

不要为预计开展的工作预热。在第一个重型命令准备就绪时再按需获取后端，后续重型命令复用返回的 `tbx_...` ID，每次运行时同步当前检出内容，并在交接前停止后端。

首次成功复用后，包装器会在 `.crabbox/testbox-leases/` 下记录租约的基础版本、依赖项和 Testbox 工作流指纹。仅修改源代码时会继续复用已预热的机器。合并基础、锁文件、包管理器输入、包装器或 Testbox 工作流发生变化时，会以失败关闭方式拒绝运行并要求新租约。每次运行仍会同步当前检出内容。
`OPENCLAW_TESTBOX_ALLOW_STALE=1` 仅用于有意开展的诊断，不用于发布验证。

下面的本地测试命令适用于人工工作流和有界的 Agent 验证。
远程提供商不可用时必须报告；这并不意味着可以静默运行大范围本地门禁。

对于不受信任的重型验证，使用 `--provider aws` 按需预热。每次运行都必须设置
`CRABBOX_ENV_ALLOW=CI`、传入 `--provider aws --no-hydrate`，并在安装依赖项或运行测试之前使用一个全新的临时远程 `HOME`。使用专门用于该不受信任源代码的新预热租约；绝不复用可信租约或之前注入过凭据的租约。从干净且可信的 `main` 检出中启动已安装的可信 Crabbox 二进制文件，并仅使用 `--fresh-pr` 获取远程 PR；绝不在本地执行不受信任检出中的包装器或配置。
取消设置 `CRABBOX_AWS_INSTANCE_PROFILE`，并且除非解析后的
`aws.instanceProfile` 为空，否则以失败关闭方式拒绝运行。在执行任何安装/测试之前，使用可信的绝对路径工具要求提供 IMDSv2 令牌，证明 IAM 凭据端点返回 404，并验证远程 `git rev-parse HEAD` 等于经过审查的完整 PR 头部 SHA。将租约绑定到该 SHA，并在头部发生变化时停止并重新预热。将干净
`main` 中可信的 `scripts/crabbox-untrusted-bootstrap.sh` 与 `--fresh-pr` 一同上传；它会安装固定版本的 Node/pnpm、验证 SHA 和包管理器版本固定要求、隔离 `HOME`、安装依赖项，然后执行所请求的测试。如果代理无法证明不存在角色，或远程 PR 不存在，请使用无密钥的 fork CI。不要使用 `hydrate-github`、`--no-sync` 或注入过凭据的 Testbox 工作流。
取消设置所有 `CRABBOX_TAILSCALE*` 覆盖项，强制使用 `--network public
--tailscale=false`，清除出口节点/LAN 标志，并要求 `crabbox inspect` 在上传任何脚本之前报告公共网络且不存在 Tailscale 状态。

## 常规本地顺序

1. `pnpm test:changed` 用于变更范围内的 Vitest 验证。
2. `pnpm test <path-or-filter>` 用于单个文件、目录或显式目标。
3. 仅当确实需要完整的本地 Vitest 测试套件时，才使用 `pnpm test`。

在 Codex 工作树或链接式/稀疏检出中，Agent 应避免直接在本地运行
`pnpm test*` / `pnpm check*` / `pnpm crabbox:run`：

- 依赖项已就绪时的有界聚焦验证：
  `node scripts/run-vitest.mjs <path-or-filter>`。
- 优先分类的变更检查：`node scripts/check-changed.mjs`；当依赖项已就绪时，仅文档、无变更和小型元数据计划保留在本地执行，而重型计划或缺少依赖项的计划则委托给 Testbox。
- 显式保留租约的大范围验证：`node scripts/crabbox-wrapper.mjs run --provider blacksmith-testbox ... -- env OPENCLAW_CHECK_CHANGED_REMOTE_CHILD=1 OPENCLAW_CHANGED_LANES_RAW_SYNC=1 corepack pnpm check:changed`，以便 pnpm 在 Testbox 内运行。
- 包装器最终输出的 `exitCode` 和计时 JSON 即为命令结果。委托的 Blacksmith GitHub Actions 运行可能会在 SSH 命令成功后显示 `cancelled`，因为 Testbox 是从保活操作外部停止的；将其视为失败之前，请检查包装器摘要和命令输出。
- `OPENCLAW_HEAVY_CHECK_LOCK_SCOPE=worktree <local-heavy-check command>`：对于 `pnpm check:changed` 和定向 `pnpm test ...` 等命令，使重型检查的串行化状态保留在当前工作树中，而不是 Git 公共目录中。仅当有意在高容量本地主机上跨链接工作树运行相互独立的检查时才使用它。

## 核心命令

测试包装器运行结束时会输出简短的 `[test] passed|failed|skipped ... in ...` 摘要；Vitest 自身的持续时间行仍提供每个分片的详细信息。

| 命令                                              | 作用                                                                                                                                                                                                                                                                                                                                                            |
| ------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `pnpm test`                                       | 显式文件/目录目标通过限定范围的 Vitest 通道运行。无目标的运行属于完整测试套件验证：固定分片组会展开为叶级配置以在本地并行执行，并在启动前打印预期的分片并行数。扩展组始终展开为逐扩展分片配置，而不是使用一个巨大的根项目进程。           |
| `pnpm test:changed`                               | 低成本的智能变更测试运行：根据直接测试编辑、同级 `*.test.ts` 文件、显式源代码映射和本地导入图确定精确目标。除非大范围变更、配置变更或软件包变更能够映射到精确测试，否则会跳过这些变更。                                                                                                                               |
| `OPENCLAW_TEST_CHANGED_BROAD=1 pnpm test:changed` | 显式的大范围变更测试运行；当测试工具、配置或软件包编辑应回退到 Vitest 范围更广的变更测试行为时使用。                                                                                                                                                                                                                        |
| `pnpm test:force`                                 | 释放配置的 OpenClaw Gateway 网关端口（默认值为 `18789`），然后使用隔离的 Gateway 网关端口运行完整测试套件，使服务器测试不会与正在运行的实例发生冲突。                                                                                                                                                                                    |
| `pnpm test:coverage`                              | 为默认单元测试通道（`vitest.unit.config.ts`）输出供参考的 V8 覆盖率报告；不强制实施覆盖率阈值。                                                                                                                                                                                                                             |
| `pnpm test:coverage:changed`                      | 仅统计自 `origin/main` 以来发生变更的文件的单元测试覆盖率。                                                                                                                                                                                                                                                                                                       |
| `pnpm changed:lanes`                              | 显示相对于 `origin/main` 的差异触发了哪些架构通道。                                                                                                                                                                                                                                                                                      |
| `pnpm check:changed`                              | 在选择执行方式之前对变更通道进行分类。当依赖项已就绪时，仅文档、无变更和小型元数据计划保留在本地执行；包含类型检查/代码检查并行任务、其他重型通道或缺少本地依赖项的计划，会在 CI 外委托给 Crabbox/Testbox。此命令不运行 Vitest；测试验证请使用 `pnpm test:changed` 或 `pnpm test <target>`。 |

## 共享测试状态和进程辅助工具

- `src/test-utils/openclaw-test-state.ts`：当测试需要隔离的 `HOME`、`OPENCLAW_STATE_DIR`、`OPENCLAW_CONFIG_PATH`、配置夹具、工作区、Agent 目录或身份验证配置文件存储时，在 Vitest 中使用。
- `pnpm test:env-mutations:report`：以非阻塞方式报告直接修改 `HOME`、`OPENCLAW_STATE_DIR`、`OPENCLAW_CONFIG_PATH`、`OPENCLAW_WORKSPACE_DIR` 或相关环境变量键的测试/测试工具。使用它查找适合迁移到共享测试状态辅助工具的候选项。
- `test/helpers/openclaw-test-instance.ts`：供需要在一个位置统一处理运行中的 Gateway 网关、CLI 环境、日志捕获和清理的进程级 E2E 测试使用。
- 引用 `scripts/lib/docker-e2e-image.sh` 的 Docker/Bash E2E 通道可以将 `docker_e2e_test_state_shell_b64 <label> <scenario>` 传入容器，并使用 `scripts/lib/openclaw-e2e-instance.sh` 对其进行解码；多主目录脚本可以传入 `docker_e2e_test_state_function_b64`，并在每个流程中调用 `openclaw_test_state_create <label> <scenario>`。`node scripts/lib/openclaw-test-state.mjs -- create --label <name> --scenario <name> --env-file <path> --json` 会写入一个可通过 source 加载的主机环境文件（`create` 前的 `--` 可防止较新版本的 Node 运行时将 `--env-file` 视为 Node 标志）。启动 Gateway 网关的通道可以引用 `scripts/lib/openclaw-e2e-instance.sh`，以统一处理入口点解析、模拟 OpenAI 启动、前台/后台启动、就绪探测、状态环境变量导出、日志转储和进程清理。

## Control UI、TUI 和扩展通道

- **Control UI 模拟 E2E：** `pnpm test:ui:e2e` 运行 Vitest + Playwright 通道，启动 Vite Control UI，并驱动真实 Chromium 页面与模拟的 Gateway 网关 WebSocket 交互。测试位于 `ui/src/**/*.e2e.test.ts`；共享模拟和控制项位于 `ui/src/test-helpers/control-ui-e2e.ts`。`pnpm test:e2e` 包含此通道。智能体运行默认使用 Testbox/Crabbox，包括针对性验证；仅在明确需要本地回退时使用 `node scripts/run-vitest.mjs run --config test/vitest/vitest.ui-e2e.config.ts --configLoader runner ui/src/ui/e2e/chat-flow.e2e.test.ts`。
- **TUI PTY 测试：** `node scripts/run-vitest.mjs run --config test/vitest/vitest.tui-pty.config.ts` 运行快速的虚假后端 PTY 通道。`OPENCLAW_TUI_PTY_INCLUDE_LOCAL=1` 或 `pnpm tui:pty:test:watch --mode local` 运行较慢的 `tui --local` 冒烟测试，该测试仅模拟外部模型端点。应断言稳定的可见文本或固件调用，而不是原始 ANSI 快照。
- `pnpm test:extensions` 和 `pnpm test extensions` 运行所有扩展/插件分片。重量级渠道插件、浏览器插件和 OpenAI 作为专用分片运行；其他插件组保持批量运行。`pnpm test extensions/<id>` 运行一个内置插件通道。
- 具有同级测试的源文件会先映射到该同级测试，再回退到更广泛的目录 glob。对 `src/channels/plugins/contracts/test-helpers`、`src/plugin-sdk/test-helpers` 和 `src/plugins/contracts` 下辅助程序的修改会使用本地导入图运行导入它们的测试；当依赖路径明确时，不会广泛运行每个分片。
- 契约目录目标会分流到各自的契约通道：`pnpm test src/channels/plugins/contracts` 运行四个渠道契约配置，`pnpm test src/plugins/contracts` 运行插件契约配置，因为通用的 `channels`/`plugins` 项目会排除 `contracts/**`。
- `auto-reply` 拆分为三个专用配置（`core`、`top-level`、`reply`），以免回复测试工具拖慢较轻量的顶层状态/令牌/辅助程序测试。
- 选定的 `plugin-sdk` 和 `commands` 测试文件会通过专用轻量通道运行，这些通道仅保留 `test/setup.ts`，而运行时负载较高的用例仍留在其现有通道上。
- 基础 Vitest 配置默认为 `pool: "threads"` 和 `isolate: false`，并在整个仓库的配置中启用共享的非隔离运行器。
- `pnpm test:channels` 运行 `vitest.channels.config.ts`。

## Gateway 网关和 E2E

- Gateway 网关集成需要选择启用：`OPENCLAW_TEST_INCLUDE_GATEWAY=1 pnpm test` 或 `pnpm test:gateway`。
- `pnpm test:e2e`：仓库 E2E 聚合 = `pnpm test:e2e:gateway && pnpm test:ui:e2e`。
- `pnpm test:e2e:gateway`：Gateway 网关端到端冒烟测试（多实例 WS/HTTP/节点配对）。默认在 `vitest.e2e.config.ts` 中使用 `threads` + `isolate: false` 和自适应工作进程；使用 `OPENCLAW_E2E_WORKERS=<n>` 调整，使用 `OPENCLAW_E2E_VERBOSE=1` 启用详细日志。
- `pnpm test:live`：提供商实时测试（Claude/Minimax/DeepSeek/z.ai/等，由 `*.live.test.ts` 控制）。需要 API 密钥和 `LIVE=1`（或 `OPENCLAW_LIVE_TEST=1`）才能取消跳过；使用 `OPENCLAW_LIVE_TEST_QUIET=0` 启用详细输出。

## 完整 Docker 套件（`pnpm test:docker:all`）

构建共享实时测试镜像，将 OpenClaw 一次性打包为 npm tarball，构建/复用一个基础 Node/Git 运行器镜像和一个将该 tarball 安装到 `/app` 的功能镜像，然后通过加权调度器运行 Docker 冒烟测试通道。`scripts/package-openclaw-for-docker.mjs` 是本地/CI 唯一的软件包打包器，并在 Docker 使用 tarball 前验证它以及 `dist/postinstall-inventory.json`。

- 基础镜像（`OPENCLAW_DOCKER_E2E_BARE_IMAGE`）：安装程序/更新/插件依赖通道；挂载预构建的 tarball，而不是复制的仓库源文件。
- 功能镜像（`OPENCLAW_DOCKER_E2E_FUNCTIONAL_IMAGE`）：常规已构建应用功能通道。
- 通道定义：`scripts/lib/docker-e2e-scenarios.mjs`。规划器：`scripts/lib/docker-e2e-plan.mjs`。执行器：`scripts/test-docker-all.mjs`。
- `node scripts/test-docker-all.mjs --plan-json` 输出由调度器管理的 CI 计划（通道、镜像类型、软件包/实时镜像需求、状态场景、凭据检查），但不构建或运行 Docker。

调度选项（环境变量，括号中为默认值）：

| 环境变量                                                                                                         | 默认值             | 用途                                                                                                                                                                                                                                                                                    |
| --------------------------------------------------------------------------------------------------------------- | ------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `OPENCLAW_DOCKER_ALL_PARALLELISM`                                                                               | 10                  | 进程槽位。                                                                                                                                                                                                                                                                             |
| `OPENCLAW_DOCKER_ALL_TAIL_PARALLELISM`                                                                          | 10                  | 对提供商敏感的尾部池。                                                                                                                                                                                                                                                              |
| `OPENCLAW_DOCKER_ALL_LIVE_LIMIT`                                                                                | 9                   | 重量级实时提供商通道上限。                                                                                                                                                                                                                                                              |
| `OPENCLAW_DOCKER_ALL_NPM_LIMIT`                                                                                 | 5                   | npm 资源通道上限。                                                                                                                                                                                                                                                                     |
| `OPENCLAW_DOCKER_ALL_SERVICE_LIMIT`                                                                             | 7                   | 服务资源通道上限。                                                                                                                                                                                                                                                                 |
| `OPENCLAW_DOCKER_ALL_LIVE_CLAUDE_LIMIT` / `_CODEX_LIMIT` / `_GEMINI_LIMIT` / `_DROID_LIMIT` / `_OPENCODE_LIMIT` | 4                   | 每个提供商的重量级通道上限。                                                                                                                                                                                                                                                              |
| `OPENCLAW_DOCKER_ALL_LIVE_OPENAI_LIMIT` / `_TELEGRAM_LIMIT`                                                     | 1                   | 更严格的每提供商上限。                                                                                                                                                                                                                                                                |
| `OPENCLAW_DOCKER_ALL_WEIGHT_LIMIT` / `OPENCLAW_DOCKER_ALL_DOCKER_LIMIT`                                         | -                   | 用于较大主机的覆盖值。                                                                                                                                                                                                                                                                 |
| `OPENCLAW_DOCKER_ALL_START_STAGGER_MS`                                                                          | 2000                | 通道启动之间的延迟，避免本地 Docker 守护进程出现集中创建风暴。                                                                                                                                                                                                                       |
| `OPENCLAW_DOCKER_ALL_LANE_TIMEOUT_MS`                                                                           | 7,200,000 (120 min) | 每通道回退超时；选定的实时/尾部通道使用更严格的上限。                                                                                                                                                                                                                      |
| `OPENCLAW_DOCKER_ALL_LIVE_RETRIES`                                                                              | 1                   | 针对暂时性实时提供商故障的重试次数。                                                                                                                                                                                                                                              |
| `OPENCLAW_DOCKER_ALL_DRY_RUN`                                                                                   | off                 | 打印通道清单而不运行 Docker。                                                                                                                                                                                                                                            |
| `OPENCLAW_DOCKER_ALL_STATUS_INTERVAL_MS`                                                                        | 30000               | 活动通道状态打印间隔。                                                                                                                                                                                                                                                         |
| `OPENCLAW_DOCKER_ALL_TIMINGS`                                                                                   | on                  | 复用 `.artifacts/docker-tests/lane-timings.json` 以按最长优先排序；设为 `0` 可禁用。                                                                                                                                                                                       |
| `OPENCLAW_DOCKER_ALL_LIVE_MODE`                                                                                 | -                   | `skip` 表示仅运行确定性/本地通道，`only` 表示仅运行实时提供商通道。别名：`pnpm test:docker:local:all`、`pnpm test:docker:live:all`。仅实时模式会将主实时通道和尾部实时通道合并为一个最长优先池，使提供商桶能够将 Claude/Codex/Gemini 工作打包在一起。 |
| `OPENCLAW_LIVE_CLI_BACKEND_SETUP_TIMEOUT_SECONDS`                                                               | 180                 | CLI 后端 Docker 设置超时。                                                                                                                                                                                                                                                          |

资源上限的环境变量模式为 `OPENCLAW_DOCKER_ALL_<RESOURCE>_LIMIT`（资源名称转换为大写，非字母数字字符折叠为 `_`）。

其他行为：运行器默认会预检 Docker、清理过期的 OpenClaw E2E 容器、在兼容的通道之间共享提供商 CLI 工具缓存，并在首次失败后停止调度新的池化通道，除非设置了 `OPENCLAW_DOCKER_ALL_FAIL_FAST=0`。如果某个通道在低并行度主机上超过有效权重/资源上限，它仍可从空池启动并独占运行，直到释放容量。每通道日志、`summary.json`、`failures.json` 和阶段耗时会写入 `.artifacts/docker-tests/<run-id>/`；使用 `pnpm test:docker:timings <summary.json>` 检查缓慢通道，使用 `pnpm test:docker:rerun <run-id|summary.json|failures.json>` 输出开销较低的针对性重跑命令。

### 重要 Docker 通道

| 命令                                                                     | 验证内容                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                              |
| --------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `pnpm test:docker:browser-cdp-snapshot`                                     | 基于 Chromium 的源码 E2E 容器，使用原始 CDP + 隔离的 Gateway 网关；`browser doctor --deep` CDP 角色快照包含链接 URL、由光标提升的可点击元素、iframe 引用和帧元数据。                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                      |
| `pnpm test:docker:skill-install`                                            | 使用 `skills.install.allowUploadedArchives: false` 在纯净 Docker 运行器中安装打包的 tarball，通过实时 ClawHub 搜索解析当前技能 slug，使用 `openclaw skills install` 安装，并验证 `SKILL.md`、`.clawhub/origin.json`、`.clawhub/lock.json` 和 `skills info --json`。                                                                                                                                                                                                                                                                                                                                                                                                                                              |
| `pnpm test:docker:live-cli-backend:claude`, `:claude:resume`, `:claude:mcp` | 针对 CLI 后端的实时探测；Gemini 提供对应的 `:resume` 和 `:mcp` 别名。                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                    |
| `pnpm test:docker:openwebui`                                                | Docker 化的 OpenClaw + Open WebUI：登录、检查 `/api/models`，并通过 `/api/chat/completions` 运行真实的代理聊天。需要可用的实时模型密钥，并会拉取外部镜像；其 CI 稳定性预计无法达到单元/E2E 套件的水平。                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                         |
| `pnpm test:docker:mcp-channels`                                             | 预置数据的 Gateway 网关容器，以及一个生成 `openclaw mcp serve` 的客户端容器：路由式会话发现、转录记录读取、附件元数据、实时事件队列行为、出站发送路由，以及通过真实 stdio 桥接传输的 Claude 风格渠道 + 权限通知（断言直接读取原始 stdio MCP 帧）。                                                                                                                                                                                                                                                                                                                                                                                                               |
| `pnpm test:docker:upgrade-survivor`                                         | 在包含脏旧用户数据的夹具上安装打包的 tarball，在没有实时提供商/渠道密钥的情况下运行软件包更新和非交互式 Doctor，启动 loopback Gateway 网关，并检查智能体/渠道配置/插件允许列表/工作区/会话文件/过期旧版插件依赖状态/启动/RPC 状态能否保留。                                                                                                                                                                                                                                                                                                                                                                                                                                     |
| `pnpm test:docker:published-upgrade-survivor`                               | 默认安装 `openclaw@latest`，预置真实的现有用户文件，通过内置的 `openclaw config set` 配方进行配置，更新到打包的 tarball，运行非交互式 Doctor，写入 `.artifacts/upgrade-survivor/summary.json`，检查 `/healthz`、`/readyz` 和 RPC 状态。可使用 `OPENCLAW_UPGRADE_SURVIVOR_BASELINE_SPEC` 覆盖，使用 `OPENCLAW_UPGRADE_SURVIVOR_BASELINE_SPECS` 扩展矩阵，或使用 `OPENCLAW_UPGRADE_SURVIVOR_SCENARIOS=reported-issues` 添加场景夹具（包含 `configured-plugin-installs` 和 `stale-source-plugin-shadow`）。软件包验收将其公开为 `published_upgrade_survivor_baseline(s)` / `_scenarios`，并解析 `last-stable-4` 或 `all-since-2026.4.23` 等元标记。 |
| `pnpm test:docker:update-migration`                                         | `plugin-deps-cleanup` 场景中的已发布版本升级存续测试框架，默认从 `openclaw@2026.4.23` 开始。`Update Migration` 工作流使用 `baselines=all-since-2026.4.23` 对此进行扩展，以验证 Full Release CI 之外已配置插件的依赖项清理。                                                                                                                                                                                                                                                                                                                                                                                                                                                                       |
| `pnpm test:docker:plugins`                                                  | 针对本地路径、`file:`、具有提升依赖项的 npm 注册表软件包、git 移动引用、ClawHub 夹具、市场更新以及 Claude 捆绑包启用/检查的安装/更新冒烟测试。                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                              |

## 本地 PR 门禁

对于本地 PR 落地/门禁检查，请运行：

- `pnpm check:changed`
- `pnpm check`
- `pnpm check:test-types`
- `pnpm build`
- `pnpm test`
- `pnpm check:docs`

如果 `pnpm test` 在高负载主机上出现偶发失败，请重跑一次再将其视为回归，然后使用 `pnpm test <path/to/test>` 隔离问题。对于内存受限的主机：

- `OPENCLAW_VITEST_MAX_WORKERS=1 pnpm test`
- `OPENCLAW_VITEST_FS_MODULE_CACHE_PATH=/tmp/openclaw-vitest-cache pnpm test:changed`

## 测试性能工具

- `pnpm test:perf:imports`：启用 Vitest 导入时长 + 导入细分报告，同时仍对显式文件/目录目标使用限定范围的通道路由。`pnpm test:perf:imports:changed` 将相同的性能分析限定到自 `origin/main` 以来发生更改的文件。
- `pnpm test:perf:changed:bench -- --ref <git-ref>` 针对同一已提交 git 差异，对比测试路由后的变更模式路径与原生根项目运行的性能；`pnpm test:perf:changed:bench -- --worktree` 无需先提交即可对当前工作树的变更集进行基准测试。
- `pnpm test:perf:profile:main` 为 Vitest 主线程写入 CPU 性能分析文件（`.artifacts/vitest-main-profile`）；`pnpm test:perf:profile:runner` 为单元测试运行器写入 CPU + 堆性能分析文件（`.artifacts/vitest-runner-profile`）。
- `pnpm test:perf:groups --full-suite --allow-failures --output .artifacts/test-perf/baseline-before.json`：串行运行完整套件的每个 Vitest 叶级配置，并写入分组时长数据以及各配置的 JSON/日志工件。完整套件报告默认隔离各文件，因此不会将先前文件保留的模块图和 GC 暂停计入后续断言；仅在有意分析共享工作进程的累积情况时传递 `-- --no-isolate`。测试性能 Agent 在尝试修复慢速测试前，将其用作基线。`pnpm test:perf:groups:compare .artifacts/test-perf/baseline-before.json .artifacts/test-perf/after-agent.json` 用于比较性能专项更改前后的分组报告。
- 完整套件、扩展和包含模式分片运行会更新 `.artifacts/vitest-shard-timings.json` 中的本地计时数据；后续完整配置运行会使用这些计时数据来平衡快慢分片。包含模式 CI 分片会将分片名称附加到计时键中，从而在不替换完整配置计时数据的情况下保留筛选后分片的可见计时数据。设置 `OPENCLAW_TEST_PROJECTS_TIMINGS=0` 可忽略本地计时工件。

## 基准测试

<Accordion title="模型延迟（scripts/bench-model.ts）">

```bash
pnpm tsx scripts/bench-model.ts --runs 10
```

可选环境变量：`MINIMAX_API_KEY`、`MINIMAX_BASE_URL`、`MINIMAX_MODEL`、`ANTHROPIC_API_KEY`。默认提示词：“只回复一个单词：ok。不要使用标点或添加额外文本。”

</Accordion>

<Accordion title="CLI 启动（scripts/bench-cli-startup.ts）">

```bash
pnpm test:startup:bench
pnpm test:startup:bench:smoke
pnpm test:startup:bench:save
pnpm test:startup:bench:update
pnpm test:startup:bench:check
pnpm tsx scripts/bench-cli-startup.ts --runs 12
pnpm tsx scripts/bench-cli-startup.ts --preset real --case status --case gatewayStatus --runs 3
pnpm tsx scripts/bench-cli-startup.ts --entry openclaw.mjs --entry-secondary dist/entry.js --preset all
```

预设：

- `startup`：`--version`、`--help`、`health`、`health --json`、`status --json`、`status`
- `real`：`health`、`status`、`status --json`、`sessions`、`sessions --json`、`tasks --json`、`tasks list --json`、`tasks audit --json`、`agents list --json`、`gateway status`、`gateway status --json`、`gateway health --json`、`config get gateway.port`
- `all`：合并两个预设

输出包括 `sampleCount`、平均值、p50、p95、最小值/最大值、退出代码/信号分布，以及每条命令的最大 RSS。`--cpu-prof-dir` / `--heap-prof-dir` 会为每次运行写入 V8 性能分析文件。

保存的输出：`pnpm test:startup:bench:smoke` 写入 `.artifacts/cli-startup-bench-smoke.json`；`pnpm test:startup:bench:save` 写入 `.artifacts/cli-startup-bench-all.json`（`runs=5 warmup=1`）。签入的夹具：`test/fixtures/cli-startup-bench.json`，由 `pnpm test:startup:bench:update` 刷新，由 `pnpm test:startup:bench:check` 比较。

</Accordion>

<Accordion title="Gateway 网关启动（scripts/bench-gateway-startup.ts）">

默认使用位于 `dist/entry.js` 的已构建 CLI 入口；请先运行 `pnpm build`。传入 `--entry scripts/run-node.mjs` 可改为测量源代码运行器，并将这些结果与已构建入口的基准结果分开保存。

```bash
pnpm test:startup:gateway -- --runs 5 --warmup 1
pnpm test:startup:gateway -- --case skipChannels --case fiftyPlugins --runs 5
node --import tsx scripts/bench-gateway-startup.ts --case default --runs 5 --output .artifacts/gateway-startup.json
```

用例 ID：`default`、`skipChannels`（跳过渠道启动）、`oneInternalHook`、`allInternalHooks`、`fiftyPlugins`（50 个清单插件）、`fiftyStartupLazyPlugins`（50 个启动时延迟加载的清单插件）。

输出包括首个进程输出、`/healthz`、`/readyz`、HTTP 监听日志时间、Gateway 网关就绪日志时间、CPU 时间、CPU 核心比率、最大 RSS、堆内存、启动跟踪指标、事件循环延迟，以及插件查找表详细指标。该脚本会在子 Gateway 网关环境中设置 `OPENCLAW_GATEWAY_STARTUP_TRACE=1`。

`/healthz` 表示存活状态（HTTP 服务器可以响应）。`/readyz` 表示可用就绪状态（启动插件 sidecar、渠道和挂接后对就绪至关重要的工作均已稳定）。启动钩子以异步方式分派，不属于就绪保证。就绪日志时间是 Gateway 网关的内部时间戳，可用于进程侧归因，但不能替代外部 `/readyz` 探测。

比较变更时，请使用 JSON 输出或 `--output`。仅当跟踪输出指向仅靠阶段计时无法解释的导入、编译或 CPU 密集型工作后，才使用 `--cpu-prof-dir`。

</Accordion>

<Accordion title="Gateway 网关重启（scripts/bench-gateway-restart.ts）">

仅支持 macOS 和 Linux（使用 SIGUSR1 进行进程内重启；在 Windows 上会立即失败）。默认使用与上述 Gateway 网关启动相同的已构建入口，也可通过 `--entry scripts/run-node.mjs` 覆盖。

```bash
pnpm test:restart:gateway -- --case skipChannels --runs 1 --restarts 5
pnpm test:restart:gateway -- --case default --runs 3 --restarts 3 --warmup 1
```

用例 ID：`skipChannels`、`skipChannelsAcpxProbe`（启用 ACPX 启动探测）、`skipChannelsNoAcpxProbe`（禁用探测）、`default`、`fiftyPlugins`。

输出包括下一个 `/healthz`、下一个 `/readyz`、停机时间、重启就绪计时、CPU、RSS、替换进程的启动跟踪指标，以及信号处理、活动工作排空、关闭阶段、下一次启动、就绪计时和内存快照的重启跟踪指标。该脚本会设置 `OPENCLAW_GATEWAY_STARTUP_TRACE=1` 和 `OPENCLAW_GATEWAY_RESTART_TRACE=1`。

当变更涉及重启信号、关闭处理程序、重启后启动、sidecar 关闭、服务交接或重启后的就绪状态时，请使用此基准测试。先从 `skipChannels` 开始，以便将 Gateway 网关机制与渠道启动隔离；只有在窄范围用例解释清楚重启路径后，才使用 `default` 或插件密集型用例。跟踪指标只是归因线索，不是结论——应结合多个样本、对应的所有者跨度、`/healthz`/`/readyz` 行为以及用户可见的重启契约来判断重启变更。

</Accordion>

## 新手引导 E2E（Docker）

可选；仅容器化新手引导冒烟测试需要。在干净的 Linux 容器中运行完整的冷启动流程：

```bash
scripts/e2e/onboard-docker.sh
```

通过伪终端驱动交互式向导，验证配置/工作区/会话文件，然后启动 Gateway 网关并运行 `openclaw health`。

## QR 导入冒烟测试（Docker）

确保维护中的 QR 运行时辅助程序可在受支持的 Docker Node 运行时下加载（默认使用 Node 24，并兼容 Node 22）：

```bash
pnpm test:docker:qr
```

## 相关内容

- [测试](/zh-CN/help/testing)
- [实时测试](/zh-CN/help/testing-live)
- [更新和插件测试](/zh-CN/help/testing-updates-plugins)
