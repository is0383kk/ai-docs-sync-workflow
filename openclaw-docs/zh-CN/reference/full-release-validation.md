---
doc-schema-version: 1
read_when:
    - 运行或重新运行完整发布验证
    - 稳定版与完整发布验证配置对比
    - 调试发布验证阶段失败问题
summary: 完整发布验证阶段、子工作流、发布配置、重新运行句柄和证据
title: 完整发布验证
x-i18n:
    generated_at: "2026-07-26T06:32:12Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: ddf165d5515f4b9bb11d239382649d332d20bb8a32bd4492ae99092fb5ee2216
    source_path: reference/full-release-validation.md
    workflow: 16
---

`Full Release Validation` 是发布产品验证的总括流程。大多数工作
在子工作流中进行，因此某个执行环境失败后可以重新运行，而无需重启
整个发布流程。在冻结 Code SHA 之前运行发布准备；如果后台机器人尚未合入
Control UI 本地化输出，该步骤会刷新相关输出，然后执行与发布 CI 所用相同的
严格零回退检查。

将产品完整、尚未更新变更日志的提交冻结为 **Code SHA**，然后运行：

```bash
pnpm ci:full-release \
  --sha <code-sha> \
  --target-ref release/YYYY.M.PATCH
```

`provider` 还接受 `anthropic` 或 `minimax`，用于跨操作系统新手引导和
端到端智能体轮次。对于 alpha/beta 软件包版本，该辅助工具会推断使用 `beta`
配置文件；否则使用 `stable`。通过 `-f key=value` 传入其他工作流输入；
仅在执行广泛的建议性扫描时使用 `-f release_profile=full`。

该辅助工具会创建一个临时 `release-ci/*` 引用，将其固定到一个受信任的
`origin/main` 工作流 SHA，仅将目标 SHA 作为候选 `ref` 传入，
并在验证后删除临时引用。每个被调度的子流程都必须
报告相同的工作流 SHA。传入
`-f reuse_evidence=false` 可强制执行全新运行，或传入
`--workflow-sha <trusted-main-sha>` 以选择仍可从当前 `origin/main`
访问的较旧工作流提交。工作流本身绝不会创建或更新
仓库引用。

## 扩展稳定版例外情况

扩展稳定版发布要求使用工作流和目标均为
规范分支的运行：

```bash
gh workflow run full-release-validation.yml \
  --ref extended-stable/YYYY.M.33 \
  -f ref=extended-stable/YYYY.M.33 \
  -f release_profile=stable
```

不要使用 `pnpm ci:full-release` 或 `release-ci/*`。发布会将该次运行的
分支、头部/目标 SHA、清单 `workflowRef`、ID 和尝试次数绑定到规范
分支和发布提交。

对于产品故障，应执行向后移植；对于已冻结目标的工具问题，应进行保持行为不变的最小修复；
对于提供商、审批或运行器故障，应在不更改源代码的情况下重试。任何分支变更都需要
完整的新运行。不得因目标较旧而省略必需的
软件包、安装程序、更新、渠道或实时行为验证。

对于常规发布，当 Code SHA 全部通过后，仅生成并提交
`CHANGELOG.md`。这个新提交即为 **Release SHA**。对
Release SHA 运行相同的辅助工具。仅当 GitHub 证明 Release
SHA 派生自 Code SHA，且完整的变更路径集合恰好为
`CHANGELOG.md` 时，才会复用产品证据；npm 预检和软件包/安装验收仍会针对
Release SHA 运行。

`release_profile=stable` 和 `release_profile=full` 始终运行全面的
实时/Docker 浸泡测试。传入 `run_release_soak=true` 可在
`beta` 配置文件中包含相同的浸泡测试通道。如果验证清单
缺少此浸泡测试和阻塞式产品性能证据，稳定版发布将被拒绝。

软件包验收通常根据解析后的
`ref` 构建候选 tarball，包括通过 `pnpm ci:full-release` 调度的完整 SHA 运行。发布
beta 版本后，传入 `release_package_spec=openclaw@YYYY.M.PATCH-beta.N` 可在发布检查、软件包验收、跨操作系统、
发布路径 Docker 和软件包 Telegram 测试中复用
已发布的 npm 软件包。仅当软件包验收需要有意验证不同的软件包时，才使用
`package_acceptance_package_spec`。Codex 插件实时软件包通道遵循相同状态：已发布的
`release_package_spec` 值派生出 `codex_plugin_spec=npm:@openclaw/codex@<version>`；
SHA/工件运行从选定引用打包 `extensions/codex`；操作员
可为 `npm:`、`npm-pack:` 或 `git:` 插件源直接设置
`codex_plugin_spec`。该通道会授予此插件所需的显式 Codex CLI 安装审批，
然后运行 Codex CLI 预检和同一会话中的 OpenAI 智能体轮次。
其最终的零重试、中等思考轮次会在省略 Codex `final` 的情况下发送可见进度，
读取随机化工作区输入，写入与其完全一致的工件，
并发送明确的完成消息。此验证可捕获 v2026.7.1 中普通进度发送
导致轮次终止的回归问题。

## 顶层阶段

对于 `rerun_group=all`，会首先运行 `Check for reusable validation evidence`
作业。它会查找具有相同发布配置文件、实际浸泡测试设置和验证输入的
最新一次先前通过的完整验证。完全相同目标的重新运行使用
`exact-target-full-validation-v1`。如果某个派生提交的完整差异恰好为
`CHANGELOG.md`，则使用 `changelog-only-release-v1`；所有产品通道都会被跳过，
验证器会独立重新检查 GitHub 提交比较、不可变
父工件、子流程运行和调度日志。任何其他目标变更都需要
全新的 Code SHA 验证。传入 `reuse_evidence=false` 可强制执行全新的完整
运行。仅当工作流来自 `main`，或来自规范的 SHA 固定
`release-ci/*` 引用且其工作流提交仍位于受信任的 `main` 谱系上时，
才会复用证据；其他工作流引用会重新运行选定的通道。

全新的面向软件包的验证会先准备一个不可变 tarball 和一个 Docker
镜像工件，然后再调度插件预发布和 OpenClaw 发布检查。
两个子流程都会在使用前验证相同的软件包 SHA、工件 ID、服务摘要、
生成方运行尝试次数和 Docker 归档摘要。与软件包无关的
裸 Docker 层使用内容寻址的 GHCR 缓存；候选版本专属镜像
仍为不可变 GitHub 工件。显式指定已发布软件包规范的聚焦运行
则保留现有软件包路径。

同样对于 `rerun_group=all`，`Verify Docker runtime image assets` 作业会使用
`OPENCLAW_EXTENSIONS=diagnostics-otel,codex` 构建
`runtime-assets` Docker 目标。它与
其他阶段并行运行，并由总括验证器强制检查；各通道不再等待
它完成后才调度。范围更窄的 `rerun_group` 会跳过此预检。

| 阶段                    | 详情                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                      |
| ----------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 目标解析                | **作业：** `Resolve target ref`<br />**子工作流：** 无<br />**验证内容：** 解析发布分支、标签或完整提交 SHA，并记录选定的输入。<br />**重新运行：** 如果此步骤失败，请重新运行总括流程。                                                                                                                                                                                                                                                                                                          |
| 共享候选版本            | **作业：** `Prepare shared release candidate`<br />**子工作流：** `OpenClaw Live And E2E Checks (Reusable)`<br />**验证内容：** 打包并验证一个精确 SHA 软件包，构建一个可正常工作的 Docker 镜像，并为两个面向软件包的子工作流记录不可变的软件包和镜像工件元组。<br />**重新运行：** 重新运行受影响的软件包、插件预发布、跨操作系统或实时/E2E 组。                                                                                                                           |
| Docker 资产预检         | **作业：** `Verify Docker runtime image assets`<br />**子工作流：** 无<br />**验证内容：** 在调度任何其他阶段之前，确认 `runtime-assets` Docker 构建目标仍可成功构建。仅针对 `rerun_group=all` 运行。<br />**重新运行：** 使用 `rerun_group=all` 重新运行总括流程。                                                                                                                                                                                                                 |
| Vitest 和常规 CI        | **作业：** `Run normal full CI`<br />**子工作流：** `CI`<br />**验证内容：** 针对目标引用运行手动完整 CI 图，包括 Linux Node 通道、内置插件分片、插件和渠道契约分片、Node 22 兼容性、`check-*`、`check-additional-*`、已构建工件冒烟检查、文档检查、Python Skills、Windows、macOS、Control UI 国际化，以及通过总括流程运行的 Android。<br />**重新运行：** `rerun_group=ci`。                                                                 |
| 插件预发布              | **作业：** `Run plugin prerelease validation`<br />**子工作流：** `Plugin Prerelease`<br />**验证内容：** 仅发布时执行的插件静态检查、智能体式插件覆盖、完整插件批次分片、插件预发布 Docker 通道，以及用于兼容性分诊的非阻塞式 `plugin-inspector-advisory` 工件。<br />**重新运行：** `rerun_group=plugin-prerelease`。                                                                                                                                                      |
| 发布检查                | **作业：** `Run release/live/Docker/QA validation`<br />**子工作流：** `OpenClaw Release Checks`<br />**验证内容：** 安装冒烟测试、跨操作系统软件包检查、软件包验收、QA Lab 一致性、实时 Matrix 和 Telegram，以及受门控的建议性 Discord、WhatsApp 和 Slack 通道。stable 和 full 配置文件还会运行全面的实时/E2E 套件和 Docker 发布路径分块；beta 可通过 `run_release_soak=true` 选择启用。<br />**重新运行：** `rerun_group=release-checks` 或范围更窄的发布检查句柄。 |
| 软件包 Telegram         | **作业：** `Run package Telegram E2E`<br />**子工作流：** `NPM Telegram Beta E2E`<br />**验证内容：** 设置 `release_package_spec` 或 `npm_telegram_package_spec` 时，针对已发布软件包运行聚焦式 Telegram E2E。完整候选版本验证改用规范的软件包验收 Telegram E2E。<br />**重新运行：** 使用 `release_package_spec` 或 `npm_telegram_package_spec` 运行 `rerun_group=npm-telegram`。                                                                                                           |
| 产品性能                | **作业：** `Run product performance evidence`<br />**子工作流：** `OpenClaw Performance`<br />**验证内容：** 针对目标 SHA 运行发布配置文件性能测试（`profile=release`、`repeat=3`、`fail_on_regression=true`、`publish_reports=false`）。Kova 输出保留在工作流工件中，子流程必须证明其报告发布器已被跳过。仅对 `rerun_group=all` 或 `rerun_group=performance` 为必需（阻塞）项；范围更窄的重新运行组不需要。<br />**重新运行：** `rerun_group=performance`。 |
| 总括验证器              | **作业：** `Verify full validation`<br />**子工作流：** 无<br />**验证内容：** 重新检查已记录的子流程运行结论，并附加来自子工作流的最慢作业表。<br />**重新运行：** 将失败的子流程重新运行至通过后，仅重新运行此作业。                                                                                                                                                                                                                                                             |

总括流程始终以仅生成工件的模式调度产品性能测试。
`OpenClaw Performance` 仅允许定时运行或显式设置
`publish_reports=true` 的手动调度发布报告。仅生成工件的
防护步骤必须成功完成，以证明发布器作业保持跳过状态。
全新和复用的证据都会记录
`controls.performanceReportPublication=artifact-only`；验证器和复用
选择器会拒绝缺少匹配的规范化性能子流程证明的证据。

验证器将规范清单上传为
`full-release-validation-<run-id>-<run-attempt>`。证据工具在下载该确切工件 ID 之前，会验证
其工件 ID、摘要、生成方运行和尝试次数。它会限制所下载 ZIP 的大小，依据 REST
`sha256:` 摘要验证其字节，并以流式方式读取唯一允许且大小受限的清单条目，而不
解压归档。为兼容较旧的发布消费者，会暂时保留一个稳定名称别名。验证器始终优先使用带尝试次数限定的工件；
在过渡期间，仅当生成方为第 1 次尝试的清单 v2 时，才接受稳定名称。
对于后续尝试和清单 v3，它会拒绝该旧名称。

对于带 `rerun_group=all` 的 `ref=main`、`release/*` 引用以及 Tideclaw
alpha 引用，具有相同引用和重新运行组的较新总括运行会取代较旧运行。
父运行被取消时，其监控器会取消它已分派的所有子
工作流。标签验证运行和固定 SHA 验证运行不会
相互取消。

## 发布检查阶段

`OpenClaw Release Checks` 是最大的子工作流。它只解析目标
一次，并在总括工作流的共享软件包工件可用时对其进行验证。
直接或聚焦分派会在面向软件包或 Docker 的阶段需要时，自行准备
`release-package-under-test` 工件。

| 阶段                     | 详情                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                             |
| ------------------------ | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 发布目标                 | **作业：** `Resolve target ref`<br />**后备工作流：** 无<br />**测试：** 所选引用、可选的预期 SHA、配置文件、重新运行组和聚焦实时套件筛选器。<br />**重新运行：** `rerun_group=release-checks`。                                                                                                                                                                                                                                                                                                                                                             |
| 软件包工件               | **作业：** `Prepare release package artifact`<br />**后备工作流：** 无<br />**测试：** 验证总括工作流的不可变软件包元组；对于直接/聚焦的发布检查分派，则打包一个候选 tarball，随后将其提供给下游面向软件包的检查。<br />**重新运行：** 受影响的软件包、跨操作系统或实时/E2E 组。                                                                                                                                                                                                                                |
| 安装冒烟测试             | **作业：** `Run install smoke`<br />**后备工作流：** `Install Smoke`<br />**测试：** 完整安装路径，包括复用根 Dockerfile 冒烟镜像、QR 软件包安装、根级和 Gateway 网关 Docker 冒烟测试、安装程序 Docker 测试，以及 Bun 全局安装镜像提供商冒烟测试。<br />**重新运行：** `rerun_group=install-smoke`。                                                                                                                                                                                                                                                           |
| 跨操作系统               | **作业：** `cross_os_release_checks`<br />**后备工作流：** `OpenClaw Cross-OS Release Checks (Reusable)`<br />**测试：** 针对所选提供商和模式，在 Linux、Windows 和 macOS 上运行全新安装和升级通道，使用候选 tarball 及基准软件包。<br />**重新运行：** `rerun_group=cross-os`。                                                                                                                                                                                                                                                                 |
| 仓库和实时 E2E           | **作业：** `Run repo/live E2E validation`<br />**后备工作流：** `OpenClaw Live And E2E Checks (Reusable)`<br />**测试：** 仓库 E2E、实时缓存、OpenAI WebSocket 流式传输、原生实时提供商和插件分片，以及由 `release_profile` 选择、以 Docker 为后端的实时模型/后端/Gateway 网关测试框架。<br />**运行：** `run_release_soak=true`、`release_profile=full` 或聚焦的 `rerun_group=live-e2e`。<br />**重新运行：** `rerun_group=live-e2e`，可选择搭配 `live_suite_filter`。                                                                                |
| Docker 发布路径          | **作业：** `Run Docker release-path validation`<br />**后备工作流：** `OpenClaw Live And E2E Checks (Reusable)`<br />**测试：** 针对共享软件包工件运行发布路径 Docker 分块。<br />**运行：** `run_release_soak=true`、`release_profile=full` 或聚焦的 `rerun_group=live-e2e`。<br />**重新运行：** `rerun_group=live-e2e`。                                                                                                                                                                                                                                     |
| 软件包验收               | **作业：** `Run package acceptance`<br />**后备工作流：** `Package Acceptance`<br />**测试：** 离线插件软件包固件、插件更新、规范的模拟 OpenAI Telegram 软件包 E2E，以及针对同一 tarball 的已发布版本升级存续检查。阻塞式发布检查使用默认的最新已发布基准；浸泡检查（`run_release_soak=true`）扩展至最近 4 个稳定 npm 版本以及 3 个固定的历史版本（`2026.4.23`、`2026.5.2`、`2026.4.15`），并针对已报告问题的升级固件运行。<br />**重新运行：** `rerun_group=package`。 |
| 成熟度评分卡             | **作业：** `Render maturity scorecard release docs`<br />**后备工作流：** `maturity-scorecard.yml`<br />**测试：** 针对目标引用渲染建议性的成熟度评分卡文档。仅在传入 `run_maturity_scorecard=true` 时运行。<br />**重新运行：** 搭配 `run_maturity_scorecard=true` 运行 `rerun_group=qa`。                                                                                                                                                                                                                                                           |
| QA 一致性                | **作业：** `Run QA Lab parity lane` 和 `Run QA Lab parity report`<br />**后备工作流：** 直接作业<br />**测试：** 候选和基准智能体一致性包，随后生成一致性报告。<br />**重新运行：** `rerun_group=qa-parity` 或 `rerun_group=qa`。                                                                                                                                                                                                                                                                                                                         |
| QA 运行时一致性          | **作业：** `Verify QA Lab runtime-pair lanes`<br />**后备工作流：** 直接作业<br />**测试：** 规范核心 `openclaw`/`codex` 通道（`pnpm openclaw qa suite --runtime-pair openclaw,codex --runtime-pair-lane core`），以及搭配 `run_release_soak=true` 时运行的浸泡通道。建议性检查：各个通道作业不会阻塞发布检查验证器。<br />**重新运行：** `rerun_group=qa-parity` 或 `rerun_group=qa`。                                                                                                                                                             |
| QA 运行时工具覆盖率      | **作业：** `Enforce QA Lab runtime tool coverage`<br />**后备工作流：** 直接作业<br />**测试：** 使用规范核心运行时配对通道（`pnpm openclaw qa coverage --tools`）的输出，检查该通道中 `openclaw` 与 `codex` 之间的动态工具漂移。阻塞性：此作业无法通过建议性覆盖机制绕过。<br />**重新运行：** `rerun_group=qa-parity` 或 `rerun_group=qa`。                                                                                                                                                                                                     |
| QA 实时 Matrix           | **作业：** `Run QA Live Matrix profile`<br />**后备工作流：** `QA-Lab - All Lanes` 可复用工作流<br />**测试：** 在 `qa-live-shared` 环境中，通过共享 Matrix 实时适配器运行已通过一致性验证的 YAML 场景。<br />**重新运行：** `rerun_group=qa-live` 或 `rerun_group=qa`；使用 `live_suite_filter=qa-live-matrix` 进行聚焦 Matrix 重新运行。                                                                                                                                                                                                                    |
| QA 实时 Telegram         | **作业：** `Run QA Lab live Telegram lane`<br />**后备工作流：** 受信任的 `OpenClaw Release Telegram QA` 分派<br />**测试：** 使用 Convex CI 凭据租约进行实时 Telegram QA。<br />**重新运行：** `rerun_group=qa-live` 或 `rerun_group=qa`。                                                                                                                                                                                                                                                                                                                                 |
| QA 实时 Discord          | **作业：** `Run QA Lab live Discord lane`<br />**后备工作流：** 直接建议性作业<br />**测试：** 启用 `OPENCLAW_RELEASE_QA_DISCORD_LIVE_CI_ENABLED` 时，使用 Convex CI 凭据租约进行实时 Discord QA。<br />**重新运行：** 搭配 `live_suite_filter=qa-live-discord` 运行 `rerun_group=qa-live`。                                                                                                                                                                                                                                                                            |
| QA 实时 WhatsApp         | **作业：** `Run QA Lab live WhatsApp lane`<br />**后备工作流：** 直接建议性作业<br />**测试：** 启用 `OPENCLAW_RELEASE_QA_WHATSAPP_LIVE_CI_ENABLED` 时，使用 Convex CI 凭据租约进行实时 WhatsApp QA。<br />**重新运行：** 搭配 `live_suite_filter=qa-live-whatsapp` 运行 `rerun_group=qa-live`。                                                                                                                                                                                                                                                                        |
| QA 实时 Slack            | **作业：** `Run QA Lab live Slack lane`<br />**后备工作流：** 直接建议性作业<br />**测试：** 启用 `OPENCLAW_RELEASE_QA_SLACK_LIVE_CI_ENABLED` 时，使用 Convex CI 凭据租约进行实时 Slack QA。<br />**重新运行：** 搭配 `live_suite_filter=qa-live-slack` 运行 `rerun_group=qa-live`。                                                                                                                                                                                                                                                                                    |
| 发布验证器               | **作业：** `Verify release checks`<br />**后备工作流：** 无<br />**测试：** 所选重新运行组所需的发布检查作业。<br />**重新运行：** 聚焦子作业通过后重新运行。                                                                                                                                                                                                                                                                                                                                                                                   |

## Docker 发布路径分块

当 `live_suite_filter` 为空时，Docker 发布路径阶段会运行以下分块：

| 分块                                                           | 覆盖范围                                                                                                                                     |
| --------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------- |
| `core`                                                          | 核心 Docker 发布路径冒烟测试通道。                                                                                                        |
| `package-update-openai`                                         | OpenAI 软件包安装/更新行为、Codex 按需安装、Codex 插件实时进度跟进，以及 Chat Completions 工具调用。 |
| `package-update-anthropic`                                      | Anthropic 软件包安装和更新行为。                                                                                               |
| `package-update-core`                                           | 提供商无关的软件包和更新行为。                                                                                                |
| `plugins-runtime-plugins`                                       | 验证插件行为的插件运行时通道。                                                                                          |
| `plugins-runtime-services`                                      | 由服务支持的插件运行时通道和实时插件运行时通道。                                                                                                |
| `plugins-runtime-install-a` 到 `plugins-runtime-install-h` | 为并行发布验证而拆分的插件安装/运行时批次。                                                                        |
| `openwebui`                                                     | 按需在专用大磁盘运行器上隔离运行的 OpenWebUI 兼容性冒烟测试。                                                      |

当只有一个 Docker 通道失败时，请在可复用的实时/E2E 工作流中使用定向的 `docker_lanes=<lane[,lane]>`。如果可用，发布工件会包含每个通道的重新运行命令，以及复用软件包工件和镜像的输入参数。

## 发布配置

`release_profile` 主要控制发布检查中的实时测试/提供商覆盖广度。它不会移除常规完整 CI、插件预发布、安装冒烟测试、软件包验收或 QA Lab。稳定版和完整配置始终运行详尽的仓库/实时 E2E 和 Docker 发布路径浸泡测试。Beta 配置可通过 `run_release_soak=true` 选择加入。软件包验收为每个完整候选版本提供规范的软件包 Telegram E2E，因此总括流程不会重复运行该实时轮询器。

| 配置  | 预期用途                      | 包含的实时测试/提供商覆盖范围                                                                                                                                                                            |
| -------- | --------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `beta`   | 最快的发布关键冒烟测试。   | OpenAI/核心实时路径、OpenAI 的 Docker 实时模型、原生 Gateway 网关核心、原生 OpenAI Gateway 网关配置、原生 OpenAI 插件，以及 Docker 实时 Gateway 网关 OpenAI。                                            |
| `stable` | 默认发布审批配置。 | `beta` 加上 Anthropic 冒烟测试、Google、MiniMax、后端、原生实时测试框架、Docker 实时 CLI 后端、Docker ACP 绑定、Docker Codex harness、Docker 子智能体公告，以及一个 OpenCode Go 冒烟测试分片。 |
| `full`   | 广泛的建议性扫描。             | `stable` 加上建议性提供商、插件实时分片和媒体实时分片。                                                                                                                               |

## 仅完整配置包含的附加项

以下测试套件会被 `stable` 跳过，并由 `full` 包含：

| 区域                             | 仅完整配置包含的覆盖范围                                                                                                          |
| -------------------------------- | --------------------------------------------------------------------------------------------------------------------------- |
| Docker 实时模型               | OpenCode Go、OpenRouter、xAI、Z.ai 和 Fireworks。                                                                          |
| Docker 实时 Gateway 网关              | 建议性提供商拆分为 DeepSeek/Fireworks、OpenCode Go/OpenRouter 和 xAI/Z.ai 分片。                              |
| 原生 Gateway 网关提供商配置 | 完整 Anthropic Opus 和 Sonnet/Haiku 分片、Fireworks、DeepSeek、完整 OpenCode Go 模型分片、OpenRouter、xAI 和 Z.ai。 |
| 原生插件实时分片        | 插件 A-K、L-N、O-Z 其他插件、Moonshot 和 xAI。                                                                             |
| 原生媒体实时分片         | 音频、Google 音乐、MiniMax 音乐和视频组 A-D。                                                                   |

`stable` 包含 `native-live-src-gateway-profiles-anthropic-smoke` 和 `native-live-src-gateway-profiles-opencode-go-smoke`；`full` 则使用覆盖范围更广的 Anthropic 和 OpenCode Go 模型分片。定向重新运行仍可使用聚合句柄 `native-live-src-gateway-profiles-anthropic` 或 `native-live-src-gateway-profiles-opencode-go`。

## 定向重新运行

使用 `rerun_group` 可避免重复运行无关的发布执行环境：

| 句柄              | 范围                                                                                           |
| ------------------- | ----------------------------------------------------------------------------------------------- |
| `all`               | 所有完整发布验证阶段。                                                             |
| `ci`                | 仅手动完整 CI 子工作流。                                                                      |
| `plugin-prerelease` | 仅插件预发布子工作流。                                                                   |
| `release-checks`    | 所有 OpenClaw 发布检查阶段。                                                             |
| `install-smoke`     | 从安装冒烟测试到发布检查。                                                           |
| `cross-os`          | 跨操作系统发布检查。                                                                        |
| `live-e2e`          | 仓库/实时 E2E 和 Docker 发布路径验证。                                               |
| `package`           | 软件包验收。                                                                             |
| `qa`                | QA 一致性以及 QA 实时通道。                                                                   |
| `qa-parity`         | 仅 QA 一致性通道和报告。                                                                |
| `qa-live`           | QA 实时 Matrix/Telegram，以及启用时受门控的 Discord、WhatsApp 和 Slack 通道。             |
| `npm-telegram`      | 已发布软件包的 Telegram E2E；需要 `release_package_spec` 或 `npm_telegram_package_spec`。 |
| `performance`       | 仅产品性能证据。                                                              |

当一个实时测试套件失败时，将 `live_suite_filter` 与 `rerun_group=live-e2e` 配合使用。有效的筛选器 ID 在可复用的实时/E2E 工作流中定义，包括 `docker-live-models`、`live-gateway-docker`、`live-gateway-anthropic-docker`、`live-gateway-google-docker`、`live-gateway-minimax-docker`、`live-gateway-advisory-docker`、`live-cli-backend-docker`、`live-acp-bind-docker` 和 `live-codex-harness-docker`。

若要定向重新运行 QA 传输测试，请设置 `rerun_group=qa-live`，并使用规范选择器 `qa-live-matrix`、`qa-live-telegram`、`qa-live-discord`、`qa-live-whatsapp` 或 `qa-live-slack`。

`live-gateway-advisory-docker` 句柄是其三个提供商分片的聚合重新运行句柄，因此仍会扇出到所有建议性 Docker Gateway 网关任务。

当一个跨操作系统通道失败时，将 `cross_os_suite_filter` 与 `rerun_group=cross-os` 配合使用。该筛选器接受操作系统 ID、测试套件 ID 或操作系统/测试套件组合，例如 `windows/packaged-upgrade`、`windows` 或 `packaged-fresh`。跨操作系统摘要包含软件包升级通道各阶段的耗时，长时间运行的命令还会输出 Heartbeat 行，以便在任务超时之前发现卡住的更新。

只有选定的 Matrix、Telegram 和 QA 运行时工具覆盖通道发生 QA 发布检查失败时，才会阻止常规发布验证。QA 一致性、运行时一致性以及受门控的 Discord、WhatsApp 和 Slack 实时通道属于建议性检查，会发布状态工件，但不会阻止发布验证器。Tideclaw Alpha 运行仍可将非软件包安全相关的发布检查通道视为建议性检查。使用 `release_profile=beta` 时，`Run repo/live E2E validation` 实时提供商测试套件属于建议性检查：第三方模型部署会在发布期间发生变化，因此 Beta 会将其失败显示为警告，而稳定版和完整配置仍会让这些失败阻止发布。当 `live_suite_filter` 明确请求受门控的 QA 实时通道（如 Discord、WhatsApp 或 Slack）时，必须启用匹配的 `OPENCLAW_RELEASE_QA_*_LIVE_CI_ENABLED` 仓库变量；否则输入捕获会失败，而不是静默跳过该通道。需要最新 QA 证据时，请重新运行 `rerun_group=qa`、`qa-parity` 或 `qa-live`。

## 要保留的证据

保留 `Full Release Validation` 摘要作为发布级索引。它链接子运行 ID，并包含最慢任务表。发生失败时，先检查子工作流，然后重新运行上述范围最小的匹配句柄。

对于常规发布，请记录 Code SHA 和 Release SHA、复用策略和变更路径集、绿色的 Code SHA 父运行，以及轻量级 Release SHA 父运行。对于扩展稳定版，请记录规范分支、确切的发布 SHA、新的父运行 ID 和尝试次数、工作流引用、每个子运行，以及任何冻结目标兼容性修复或有意省略的内容。

实用工件：

- `release-package-under-test`，来自 `OpenClaw Release Checks`
- `.artifacts/docker-tests/` 下的 Docker 发布路径工件
- 软件包验收 `package-under-test` 和 Docker 验收工件
- 每个操作系统和测试套件的跨操作系统发布检查工件
- QA 一致性、运行时一致性，以及选定的 Matrix、Telegram、Discord、WhatsApp 或 Slack 工件

## 工作流文件

- `.github/workflows/full-release-validation.yml`
- `.github/workflows/openclaw-release-checks.yml`
- `.github/workflows/openclaw-live-and-e2e-checks-reusable.yml`
- `.github/workflows/plugin-prerelease.yml`
- `.github/workflows/install-smoke.yml`
- `.github/workflows/install-smoke-reusable.yml`
- `.github/workflows/openclaw-cross-os-release-checks-reusable.yml`
- `.github/workflows/package-acceptance.yml`
- `.github/workflows/openclaw-performance.yml`
- `.github/workflows/npm-telegram-beta-e2e.yml`
