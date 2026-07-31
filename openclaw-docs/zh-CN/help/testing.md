---
read_when:
    - 在本地或 CI 中运行测试
    - 为模型/提供商错误添加回归测试
    - 调试 Gateway 网关和智能体行为
summary: 测试工具包：单元/E2E/实时测试套件、Docker 运行器，以及各项测试的覆盖范围
title: 测试
x-i18n:
    generated_at: "2026-07-26T06:44:20Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 20e0aa22bf16561334f83342abffabb387ed0b41b901773939123ecfbc0ae330
    source_path: help/testing.md
    workflow: 16
---

OpenClaw 有三套 Vitest 测试套件（单元/集成、端到端、实时），以及 Docker
运行器。本页介绍每套测试的覆盖范围、针对特定工作流应运行的命令、
实时测试如何发现凭据，以及如何为真实场景中的提供商/模型错误添加
回归测试。

<Note>
**QA 技术栈（qa-lab、qa-channel、实时传输通道）**单独提供文档：

- [QA overview](/zh-CN/concepts/qa-e2e-automation) - 架构、命令界面、场景编写和 Matrix 配置文件。
- [成熟度评分卡](/zh-CN/maturity/scorecard) - 发布 QA 证据如何支持稳定性和 LTS 决策。
- [QA channel](/zh-CN/channels/qa-channel) - 仓库支持的场景所使用的合成传输插件。

本页介绍常规测试套件以及 Docker/Parallels 运行器。下文的 [QA 专用运行器](#qa-specific-runners)列出了具体的 `qa` 调用，并链接回上述参考资料。
</Note>

## 快速开始

大多数情况下：

- 完整门禁（推送前应运行）：`pnpm build && pnpm check && pnpm check:test-types && pnpm test`
- 在资源充足的机器上更快地运行本地完整套件：`pnpm test:max`
- 直接运行 Vitest 监视循环：`pnpm test:watch`
- 直接指定文件也能路由插件/渠道路径：`pnpm test extensions/discord/src/monitor/message-handler.preflight.test.ts`
- 迭代处理单个失败时，优先运行针对性测试。
- Docker 支持的 QA 站点：`pnpm qa:lab:up`
- Linux 虚拟机支持的 QA 通道：`pnpm openclaw qa suite --runner multipass --scenario channel-chat-baseline`

修改测试或希望获得更多信心时：

- 仅供参考的 V8 覆盖率报告：`pnpm test:coverage`
- 端到端测试套件：`pnpm test:e2e`

## 测试临时目录

对测试拥有的临时目录使用 `test/helpers/temp-dir.ts` 中的共享辅助程序，
以便明确所有权，并确保清理操作保留在测试生命周期内：

```ts
import { afterEach } from "vitest";
import { useAutoCleanupTempDirTracker } from "../helpers/temp-dir.js";

const tempDirs = useAutoCleanupTempDirTracker(afterEach);

it("使用临时工作区", () => {
  const workspace = tempDirs.make("openclaw-example-");
  // 使用工作区
});
```

`useAutoCleanupTempDirTracker(afterEach)` 特意不提供手动清理方法——每次测试后由 Vitest
负责清理。较旧的底层辅助程序（`makeTempDir`、`cleanupTempDirs`、`createTempDirTracker`）
仍然存在，以供尚未迁移的测试使用；应避免新增对它们的使用，也应避免新增裸
`fs.mkdtemp*` 调用，除非测试明确验证原始临时目录行为。
确实需要裸临时目录时，请添加可审计的允许注释并说明原因：

```ts
// openclaw-temp-dir: allow 验证原始文件系统清理行为
const workspace = fs.mkdtempSync(prefix);
```

`node scripts/report-test-temp-creations.mjs` 会报告新增差异行中的新裸临时目录创建和新增的
共享辅助程序手动用法，但不会阻止现有清理方式。它采用与
`scripts/changed-lanes.mjs` 相同的测试路径分类，并跳过共享辅助程序实现本身。
`check:changed` 会针对已更改的测试路径运行此报告，作为仅警告的
CI 信号（GitHub 警告注解，而非失败）。

## 实时和 Docker/Parallels 工作流

调试真实提供商/模型时（需要真实凭据）：

- 实时测试套件（模型 + Gateway 网关工具/图像探测）：`pnpm test:live`
- 安静地指定一个实时测试文件：`pnpm test:live -- src/agents/models.profiles.live.test.ts`
- 运行时性能报告：分派 `OpenClaw Performance`，并使用
  `live_openai_candidate=true` 执行真实的 `openai/gpt-5.6-luna` 智能体轮次，或使用
  `deep_profile=true` 生成 Kova CPU/堆/跟踪工件。每日计划运行会通过单独的
  工件消费发布作业，将模拟提供商、深度分析和 GPT-5.6 Luna 通道报告发布到
  `openclaw/clawgrit-reports`；发布者身份验证缺失或无效会导致计划运行和
  `profile=release` 运行失败。非发布的手动分派会保留 GitHub 工件，
  并将报告发布视为建议项。模拟提供商报告还包括源码级 Gateway 网关启动、
  内存、插件压力、重复的虚假模型 hello 循环和 CLI 启动数据。
- Docker 实时模型扫描：`pnpm test:docker:live-models`
  - 每个选定模型都会运行一个文本轮次和一个小型文件读取式探测。
    元数据声明支持 `image` 输入的模型还会运行一个微型图像轮次。
    隔离提供商失败时，可使用 `OPENCLAW_LIVE_MODEL_FILE_PROBE=0` 或
    `OPENCLAW_LIVE_MODEL_IMAGE_PROBE=0` 禁用额外探测。
  - CI 覆盖：每日 `OpenClaw Scheduled Live And E2E Checks` 和手动
    `OpenClaw Release Checks` 都会使用 `include_live_suites: true` 调用可复用的实时/端到端工作流，
    其中包括按提供商分片的 Docker 实时模型矩阵作业。
  - 如需针对性地重新运行 CI，请分派 `OpenClaw Live And E2E Checks (Reusable)`，
    并使用 `include_live_suites: true` 和 `live_models_only: true`。
  - 将新的高信号提供商密钥添加到 `scripts/ci-hydrate-live-auth.sh`、
    `.github/workflows/openclaw-live-and-e2e-checks-reusable.yml` 及其计划/发布调用方中。
- Native Codex 绑定聊天冒烟测试：`pnpm test:docker:live-codex-bind`
  - 针对 Codex app-server 路径运行 Docker 实时通道，使用
    `/codex bind` 绑定一个合成 Slack 私信，执行 `/codex fast` 和
    `/codex permissions`，然后验证纯文本回复和图像附件是否通过原生插件绑定路由，
    而不是通过 ACP。
- Codex app-server harness 冒烟测试：`pnpm test:docker:live-codex-harness`
  - 通过插件所有的 Codex app-server harness 运行 Gateway 网关智能体轮次，
    验证 `/codex status` 和 `/codex models`，默认还会执行图像、cron MCP、
    子智能体和 Guardian 探测。隔离其他失败时，可使用
    `OPENCLAW_LIVE_CODEX_HARNESS_SUBAGENT_PROBE=0` 禁用子智能体探测。如需针对性检查子智能体，
    请禁用其他探测：
    `OPENCLAW_LIVE_CODEX_HARNESS_IMAGE_PROBE=0 OPENCLAW_LIVE_CODEX_HARNESS_MCP_PROBE=0 OPENCLAW_LIVE_CODEX_HARNESS_GUARDIAN_PROBE=0 OPENCLAW_LIVE_CODEX_HARNESS_SUBAGENT_PROBE=1 pnpm test:docker:live-codex-harness`。
    除非设置了 `OPENCLAW_LIVE_CODEX_HARNESS_SUBAGENT_ONLY=0`，否则该测试会在子智能体探测后退出。
- Codex 按需安装冒烟测试：`pnpm test:docker:codex-on-demand`
  - 在 Docker 中安装打包的 OpenClaw tarball，运行 OpenAI API 密钥
    新手引导，并验证 Codex 插件及 `@openai/codex` 依赖项是否已按需下载到
    受管理的 npm 项目根目录中。
- Codex npm 插件实时包冒烟测试：`pnpm test:docker:live-codex-npm-plugin`
  - 将候选 OpenClaw 包和精确版本的 Codex 插件安装到 Docker 中，
    然后使用真实的 OpenAI 密钥执行 CLI 预检和同一会话中的轮次。
  - 其零重试、中等思考强度的后续轮次必须发送进度，在随机工作区读取和
    精确工件写入过程中持续工作，然后发送完成消息。仅发送进度便终止的轮次会使该通道失败。
- 实时插件工具依赖项冒烟测试：`pnpm test:docker:live-plugin-tool`
  - 打包一个带有真实 `slugify` 依赖项的夹具插件，通过
    `npm-pack:` 安装它，验证受管理的 npm 项目根目录下的依赖项，
    然后请求一个实时 OpenAI 模型调用插件工具并返回隐藏 slug。
- OpenClaw 救援命令冒烟测试：`pnpm test:live:system-agent-rescue-channel`
  - 针对消息渠道救援命令界面的可选双重保障检查。
    执行 `/openclaw status`，将持久模型更改加入队列，回复
    `/openclaw yes`，并验证审计/配置写入路径。
- OpenClaw 首次运行 Docker 冒烟测试：`pnpm test:docker:system-agent-first-run`
  - 从空的 OpenClaw 状态目录开始，首先证明打包的
    `openclaw setup` CLI 在不进行推断的情况下会以安全方式失败。随后通过打包的
    激活模块测试并激活虚假 Claude。只有在此之后，模糊的打包 CLI 请求才会到达规划器，
    解析为类型化设置，然后执行一次性的模型、智能体、Discord 配置和 SecretRef 操作。
    它会验证配置和审计条目。这是支持门禁/操作的证据，而不是交互式新手引导，
    也不是 OpenClaw 智能体/工具/审批的证据。QA Lab 中也通过
    `pnpm openclaw qa suite --scenario system-agent-ring-zero-setup` 提供同一通道。
- Moonshot/Kimi 成本冒烟测试：设置 `MOONSHOT_API_KEY` 后，运行
  `openclaw models list --provider moonshot --json`，然后针对 `moonshot/kimi-k2.6` 运行隔离的
  `openclaw agent --local --session-id live-kimi-cost --message 'Reply exactly: KIMI_LIVE_OK' --thinking off --json`。
  验证 JSON 报告 Moonshot/K2.6，并且助手转录存储规范化的 `usage.cost`。

<Tip>
仅需处理一个失败用例时，优先通过下文所述的允许列表环境变量缩小实时测试范围。
</Tip>

## QA 专用运行器

需要 QA Lab 真实性时，这些命令与主测试套件配合使用。

CI 在专用工作流中运行 QA Lab。智能体一致性测试嵌套在
`QA-Lab - All Lanes` 和发布验证中，而不是独立的 PR 工作流。
广泛验证应使用 `Full Release Validation` 搭配
`rerun_group=qa-parity`，或使用发布检查 QA 组。稳定版/默认发布检查会将穷尽式实时/Docker
浸泡测试置于 `run_release_soak=true` 之后；`full` 配置文件会强制启用浸泡测试。
`QA-Lab - All Lanes` 每晚在 `main` 上运行，也可通过手动分派运行，
其中模拟一致性通道、实时 Matrix 通道、由 Convex 管理的实时 Telegram 通道，
以及由 Convex 管理的实时 Discord 通道会作为并行作业执行。计划 QA 和发布检查
通过共享实时适配器运行 Matrix 发布配置文件。Matrix CLI 和手动工作流输入的默认值
仍为 `all`；手动 `all` 分派会并行展开传输、媒体和
E2EE 配置文件，而针对性分派可选择 `fast`、`release` 或
`transport`。`OpenClaw Release Checks` 会在发布审批前运行一致性测试、
可复用的 Matrix 实时适配器配置文件和 Telegram 通道。发布传输检查使用
`mock-openai/gpt-5.6-luna`，从而保持确定性并避免正常的提供商插件启动。这些实时传输
Gateway 网关会禁用记忆搜索；记忆行为仍由 QA 一致性测试套件覆盖。

完整发布实时媒体分片使用
`ghcr.io/openclaw/openclaw-live-media-runner:ubuntu-24.04`，其中已包含
`ffmpeg` 和 `ffprobe`。Docker 实时模型/后端分片使用针对每个
选定提交仅构建一次的共享 `ghcr.io/openclaw/openclaw-live-test:<sha>` 镜像，然后通过
`OPENCLAW_SKIP_DOCKER_BUILD=1` 拉取该镜像，而不是在每个分片内部重新构建。

- `pnpm openclaw qa suite`
  - 直接在主机上运行由仓库支持的 QA 场景。
  - 为所选场景集写入顶层 `qa-evidence.json`、`qa-suite-summary.json` 和
    `qa-suite-report.md` 工件，包括混合流程、Vitest 和 Playwright 场景选择。
  - 由 `pnpm openclaw qa run --qa-profile <profile>` 调度时，会将
    所选分类法配置文件的评分卡嵌入同一个 `qa-evidence.json`。
    `smoke-ci` 写入精简证据（`evidenceMode: "slim"`，不含逐条
    `execution`）。`release` 覆盖精选的发布就绪范围；`all`
    选择每个活跃的成熟度类别，并在需要完整评分卡工件时，以显式的 QA Profile
    Evidence 工作流调度为目标。
  - 默认使用相互隔离的 Gateway 网关工作进程并行运行多个所选场景。
    `qa-channel` 默认并发数为 4（受所选场景数量限制）。使用 `--concurrency <count>`
    调整工作进程数量，或使用 `--concurrency 1` 运行旧版串行通道。
  - 任何场景失败时以非零状态退出。使用 `--allow-failures` 可生成
    工件而不返回失败退出码。
  - 支持提供商模式 `live-frontier`、`mock-openai` 和 `aimock`。
    `aimock` 启动由本地 AIMock 支持的提供商服务器，为实验性
    固件和协议模拟提供覆盖，而不会取代场景感知的
    `mock-openai` 通道。
- `pnpm openclaw qa coverage --match <query>`
  - 搜索场景 ID、标题、表面、覆盖 ID、文档引用、代码
    引用、插件和提供商要求，然后输出匹配的测试套件
    目标。
  - 如果知道受影响的行为或文件路径，但不知道最小适用场景，请在运行 QA Lab 前使用此功能。
    这仅供参考——仍需根据变更的行为选择模拟、
    实时、Multipass、Matrix 或传输证明。
- `pnpm test:plugins:kitchen-sink-live`
  - 通过 QA Lab 运行实时 OpenAI Kitchen Sink 插件全套考验。
    安装外部 Kitchen Sink 包，验证插件 SDK
    表面清单，探测 `/healthz` 和 `/readyz`，记录 Gateway 网关
    CPU/RSS 证据，运行一次实时 OpenAI 轮次，并检查对抗性
    诊断。需要实时 OpenAI 身份验证，例如 `OPENAI_API_KEY`。在
    已注入凭据的 Testbox 会话中，如果存在 `openclaw-testbox-env` 辅助程序，
    则会自动加载 Testbox 实时身份验证配置文件。
- `pnpm test:gateway:cpu-scenarios`
  - 运行 Gateway 网关启动基准测试和一组小型模拟 QA Lab 场景包
    （`channel-chat-baseline`、`memory-failure-fallback`、
    `gateway-restart-inflight-run`），并在 `.artifacts/gateway-cpu-scenarios/` 下写入合并的 CPU 观测
    摘要。
  - 默认仅标记持续的 CPU 高占用观测（`--cpu-core-warn`，
    默认为 `0.9`；`--hot-wall-warn-ms`，默认为 `30000`），因此短暂的启动
    峰值会被记录为指标，而不会看起来像持续数分钟的
    Gateway 网关占满回归。
  - 针对已构建的 `dist` 工件运行；如果检出中尚无最新的运行时输出，
    请先运行构建。
- `pnpm openclaw qa suite --runner multipass`
  - 在一次性 Multipass Linux 虚拟机中运行相同的 QA 测试套件，并保留
    与 `qa suite` 相同的场景选择和提供商/模型标志。
  - 实时运行会转发适用于来宾系统的 QA 身份验证输入：
    基于环境变量的提供商密钥、QA 实时提供商配置路径，以及存在时的
    `CODEX_HOME`。
  - 输出目录必须位于仓库根目录下，以便来宾系统通过已挂载的工作区写回。
  - 写入常规 QA 报告和摘要，以及 `.artifacts/qa-e2e/...` 下的 Multipass 日志。
- `pnpm qa:lab:up`
  - 启动由 Docker 支持的 QA 站点，用于操作员式 QA 工作。
- `pnpm test:docker:npm-onboard-channel-agent`
  - 从当前检出构建 npm tarball，在 Docker 中全局安装，运行非交互式 OpenAI API 密钥新手引导，
    默认配置 Telegram，验证打包后的插件运行时无需启动依赖修复即可加载，
    运行 Doctor，并针对模拟的 OpenAI 端点运行一次本地智能体轮次。
  - 使用 `OPENCLAW_NPM_ONBOARD_CHANNEL=discord` 可通过 Discord 运行相同的打包安装
    通道。
- `pnpm test:docker:session-runtime-context`
  - 为嵌入式运行时上下文记录运行确定性的已构建应用 Docker 冒烟测试。
    验证隐藏的 OpenClaw 运行时上下文会作为非显示自定义消息持久保存，而不会泄漏到可见的用户
    轮次中；随后植入一个受影响的损坏会话 JSONL，并验证
    `openclaw doctor --fix` 会将其重写到活动分支并创建备份。
- `pnpm test:docker:npm-telegram-live`
  - 在 Docker 中安装 OpenClaw 候选包，运行已安装包的
    新手引导，通过已安装的 CLI 配置 Telegram，然后复用
    实时 Telegram QA 通道，并将该已安装包用作被测系统的
    Gateway 网关。
  - 包装器仅从检出中挂载 `qa-lab` 测试框架源代码；
    已安装包拥有 `dist`、`openclaw/plugin-sdk` 和内置
    插件运行时，因此该通道不会将当前检出的插件混入
    被测包。
  - 默认为 `OPENCLAW_NPM_TELEGRAM_PACKAGE_SPEC=openclaw@beta`；设置
    `OPENCLAW_NPM_TELEGRAM_PACKAGE_TGZ=/path/to/openclaw-current.tgz` 或
    `OPENCLAW_CURRENT_PACKAGE_TGZ`，可测试已解析的本地 tarball，而不是
    从注册表安装。
  - 默认使用 `OPENCLAW_NPM_TELEGRAM_RTT_SAMPLES=20`，在 `qa-evidence.json` 中输出重复的 RTT 计时。
    可覆盖
    `OPENCLAW_NPM_TELEGRAM_RTT_SAMPLES`、
    `OPENCLAW_NPM_TELEGRAM_RTT_TIMEOUT_MS` 或
    `OPENCLAW_NPM_TELEGRAM_RTT_MAX_FAILURES` 来调整运行。
    `OPENCLAW_NPM_TELEGRAM_RTT_CHECKS` 选择要采样的 Telegram QA 场景；
    支持的 RTT 目标是 `channel-canary`。
  - 使用与 `pnpm openclaw qa telegram` 相同的 Telegram 环境变量凭据或 Convex 凭据源。
    对于 CI/发布自动化，请设置
    `OPENCLAW_NPM_TELEGRAM_CREDENTIAL_SOURCE=convex`、
    `OPENCLAW_QA_CONVEX_SITE_URL` 和角色密钥。如果
    CI 中存在 `OPENCLAW_QA_CONVEX_SITE_URL` 和 Convex 角色密钥，
    Docker 包装器会自动选择 Convex。
  - 包装器会在执行 Docker 构建/安装工作前，在主机上验证 Telegram 或 Convex 凭据环境变量。
    仅在有意调试凭据设置前的流程时设置
    `OPENCLAW_NPM_TELEGRAM_SKIP_CREDENTIAL_PREFLIGHT=1`。
  - `OPENCLAW_NPM_TELEGRAM_CREDENTIAL_ROLE=ci|maintainer` 仅为此通道覆盖
    共享的 `OPENCLAW_QA_CREDENTIAL_ROLE`。选择 Convex
    凭据但未设置角色时，包装器在 CI 中使用 `ci`，
    在 CI 外使用 `maintainer`。
  - GitHub Actions 将此通道公开为手动维护者工作流
    `NPM Telegram Beta E2E`。它不会在合并时运行。该工作流使用
    `qa-live-shared` 环境和 Convex CI 凭据租约。
- GitHub Actions 还公开 `Package Acceptance`，用于针对单个候选包进行旁路产品证明。
  它接受 Git 引用、已发布的 npm 规范、
  HTTPS tarball URL 及 SHA-256、可信 URL 策略，或来自另一次运行的 tarball 工件
  （`source=ref|npm|url|trusted-url|artifact`），将规范化后的
  `openclaw-current.tgz` 作为 `package-under-test` 上传，然后使用 `smoke`、`package`、`product`、`full`
  或 `custom` 通道配置文件运行现有 Docker E2E 调度器。设置 `telegram_mode=mock-openai` 或
  `live-frontier`，可针对同一个
  `package-under-test` 工件运行 Telegram QA 工作流。
  - 最新 beta 产品证明：

```bash
gh workflow run package-acceptance.yml --ref main \
  -f source=npm \
  -f package_spec=openclaw@beta \
  -f suite_profile=product \
  -f telegram_mode=mock-openai
```

- 精确 tarball URL 证明需要摘要，并使用公共 URL 安全策略：

```bash
gh workflow run package-acceptance.yml --ref main \
  -f source=url \
  -f package_url=https://registry.npmjs.org/openclaw/-/openclaw-VERSION.tgz \
  -f package_sha256=<sha256> \
  -f suite_profile=package
```

- 企业/私有 tarball 镜像使用显式的可信源策略：

```bash
gh workflow run package-acceptance.yml --ref main \
  -f source=trusted-url \
  -f trusted_source_id=enterprise-artifactory \
  -f package_url=https://packages.example.internal:8443/artifactory/openclaw/openclaw-VERSION.tgz \
  -f package_sha256=<sha256> \
  -f suite_profile=package
```

`source=trusted-url` 从可信工作流引用读取 `.github/package-trusted-sources.json`，且不接受 URL 凭据或通过工作流输入绕过私有网络限制。如果指定策略声明了 bearer 身份验证，请配置固定的 `OPENCLAW_TRUSTED_PACKAGE_TOKEN` 密钥。

- 工件证明从另一次 Actions 运行中下载 tarball 工件：

```bash
gh workflow run package-acceptance.yml --ref main \
  -f source=artifact \
  -f artifact_run_id=<run-id> \
  -f artifact_name=<artifact-name> \
  -f suite_profile=smoke
```

- `pnpm test:docker:plugins`
  - 在 Docker 中打包并安装当前 OpenClaw 构建，启动已配置 OpenAI 的
    Gateway 网关，然后通过编辑配置启用内置渠道/插件。
  - 验证设置发现过程会让未配置的可下载插件保持缺失，首次配置后的 Doctor 修复会显式安装每个缺失的
    可下载插件，并且第二次重启不会运行
    隐式依赖修复。
  - 还会安装一个已知的旧版 npm 基线，在运行 `openclaw update --tag <candidate>` 前启用 Telegram，
    并验证候选版本更新后的 Doctor 会清理旧版插件依赖残留，
    而无需测试框架侧的 postinstall 修复。
- `pnpm test:parallels:npm-update`
  - 在 Parallels 来宾系统上运行原生打包安装更新冒烟测试。
    每个所选平台先安装请求的基线包，
    然后在同一来宾系统中运行已安装的 `openclaw update` 命令，并
    验证已安装版本、更新状态、Gateway 网关就绪状态和
    一次本地智能体轮次。
  - 迭代单个来宾系统时使用 `--platform macos`、`--platform windows` 或 `--platform linux`。
    使用 `--json` 指定摘要工件
    路径和各通道状态。
  - OpenAI 通道默认使用 `openai/gpt-5.6-luna` 进行实时智能体轮次证明。
    传递 `--model <provider/model>` 或设置
    `OPENCLAW_PARALLELS_OPENAI_MODEL`，可验证其他 OpenAI 模型。
  - 使用主机超时包装长时间的本地运行，以免 Parallels 传输停滞
    耗尽剩余测试时间：

    ```bash
    timeout --foreground 150m pnpm test:parallels:npm-update -- --json
    timeout --foreground 90m pnpm test:parallels:npm-update -- --platform windows --json
    ```

  - 该脚本在
    `/tmp/openclaw-parallels-npm-update.*` 下写入嵌套通道日志。在认定外层
    包装器已挂起前，请检查 `windows-update.log`、
    `macos-update.log` 或 `linux-update.log`。
  - 在冷启动的来宾系统上，Windows 更新可能会在更新后的 Doctor 和
    包更新工作中耗时 10 到 15 分钟；只要嵌套的 npm 调试日志仍在推进，
    就仍属正常。
  - 不要将此聚合包装器与单独的 Parallels
    macOS、Windows 或 Linux 冒烟通道并行运行。它们共享虚拟机状态，可能在
    快照恢复、包服务或来宾 Gateway 网关状态方面发生冲突。
  - 更新后的证明会运行常规内置插件表面，因为
    语音、图像生成和媒体理解等能力门面通过内置运行时 API
    加载，即使智能体轮次本身只检查简单的文本响应。

- `pnpm openclaw qa aimock`
  - 仅启动本地 AIMock 提供商服务器，用于直接进行协议冒烟
    测试。
- `pnpm openclaw qa matrix`
  - 针对由 Docker 支持的临时 Tuwunel
    主服务器运行 Matrix 实时 QA 通道。仅限源码检出版本——打包安装不会包含
    `qa-lab`。
  - 完整的 CLI、配置文件/场景目录、环境变量和工件布局：
    [Matrix 冒烟通道](/zh-CN/concepts/qa-e2e-automation#matrix-smoke-lanes)。
- `pnpm openclaw qa telegram`
  - 使用环境变量中的驱动程序和 SUT Bot 令牌，针对真实私有群组运行
    Telegram 实时 QA 通道。
  - 需要 `OPENCLAW_QA_TELEGRAM_GROUP_ID`、
    `OPENCLAW_QA_TELEGRAM_DRIVER_BOT_TOKEN` 和
    `OPENCLAW_QA_TELEGRAM_SUT_BOT_TOKEN`。群组 ID 必须是数字形式的
    Telegram 聊天 ID。
  - 支持通过 `--credential-source convex` 使用共享池化凭据。
    默认使用环境变量模式，或设置 `OPENCLAW_QA_CREDENTIAL_SOURCE=convex`
    以选择使用池化租约。
  - 默认覆盖金丝雀测试、提及门控、命令寻址、`/status`、
    Bot 间提及回复和核心原生命令回复。
    `mock-openai` 默认还覆盖确定性回复链和
    Telegram 最终消息流式传输回归。使用 `--list-scenarios`
    运行可选探测，例如 `session_status`。
  - 任何场景失败时都会以非零状态退出。使用 `--allow-failures`
    可在生成工件的同时避免失败退出码。
  - 需要同一私有群组中的两个不同 Bot，且 SUT Bot
    必须公开 Telegram 用户名。
  - 为稳定观察 Bot 间通信，请在 `@BotFather` 中为两个 Bot
    启用 Bot-to-Bot Communication Mode，并确保驱动 Bot 能观察
    群组中的 Bot 流量。
  - 在 `.artifacts/qa-e2e/...` 下写入 Telegram QA 报告、摘要和
    `qa-evidence.json`。回复场景包含从驱动程序发送
    请求到观察到 SUT 回复的 RTT。

`Mantis Telegram Live` 是此通道的 PR 证据包装器。它使用从 Convex 租用的
Telegram 凭据运行候选引用，在 Crabbox 桌面浏览器中呈现
已脱敏的 QA 报告/证据包，录制 MP4 证据，生成经运动画面裁剪的 GIF，
上传工件包，并在设置 `pr_number` 时通过 Mantis GitHub App
发布内联 PR 证据。维护者可通过 `Mantis Scenario`
（`scenario_id: telegram-live`）从 Actions UI 启动它，也可以直接通过拉取请求评论启动：

```text
@openclaw-mantis telegram
@openclaw-mantis telegram scenario=telegram-status-command
@openclaw-mantis telegram scenarios=telegram-status-command,channel-canary
```

`Mantis Telegram Desktop Proof` 是用于 PR 可视化证明的智能体式原生 Telegram Desktop
前后对比包装器。可通过自由格式的 `instructions` 从 Actions UI 启动，
通过 `Mantis Scenario`（`scenario_id:
telegram-desktop-proof`）启动，或通过 PR 评论启动：

```text
@openclaw-mantis telegram desktop proof
```

Mantis 智能体读取 PR，确定哪些 Telegram 可见行为能够证明
该变更，在基线和候选引用上运行真实用户 Crabbox Telegram Desktop
证明通道，反复调整直至原生 GIF 有效，写入配对的 `motionPreview`
清单，并在设置 `pr_number` 时通过 Mantis GitHub App
发布相同的两列 GIF 表格。

- `pnpm openclaw qa mantis telegram-desktop-builder`
  - 租用或复用 Crabbox Linux 桌面，安装原生 Telegram
    Desktop，使用租用的 Telegram SUT Bot 令牌配置 OpenClaw，
    启动 Gateway 网关，并从可见的 VNC 桌面录制屏幕截图/MP4 证据。
  - 默认为 `--credential-source convex`，因此工作流只需要
    Convex 代理密钥。使用 `--credential-source env` 时，所需的
    `OPENCLAW_QA_TELEGRAM_*` 变量与 `pnpm openclaw qa telegram` 相同。
  - Telegram Desktop 仍需要用户登录/配置文件。Bot 令牌
    仅用于配置 OpenClaw。可使用 `--telegram-profile-archive-env <name>`
    提供 base64 `.tgz` 配置文件归档，或使用 `--keep-lease`，
    并通过 VNC 手动登录一次。
  - 在输出目录下写入 `mantis-telegram-desktop-builder-report.md`、
    `mantis-telegram-desktop-builder-summary.json`、
    `telegram-desktop-builder.png` 和 `telegram-desktop-builder.mp4`。

实时传输通道共享一套标准契约，避免新增传输出现偏差；各通道的覆盖矩阵位于
[QA overview - 实时传输覆盖范围](/zh-CN/concepts/qa-e2e-automation#live-transport-coverage)。
`qa-channel` 是广泛的合成测试套件，不属于该矩阵。

### 通过 Convex 共享 Telegram 凭据（v1）

为实时传输 QA 启用 `--credential-source convex`（或 `OPENCLAW_QA_CREDENTIAL_SOURCE=convex`）
后，QA Lab 会从基于 Convex 的池中获取独占租约，在通道运行期间为该租约
发送 Heartbeat，并在关闭时释放租约。该节名称早于 Discord、Slack 和
WhatsApp 支持；这些类型共享同一租约契约。

参考 Convex 项目脚手架：`qa/convex-credential-broker/`

必需的环境变量：

- `OPENCLAW_QA_CONVEX_SITE_URL`（例如 `https://your-deployment.convex.site`）
- 所选角色对应的一个密钥：
  - `OPENCLAW_QA_CONVEX_SECRET_MAINTAINER`，用于 `maintainer`
  - `OPENCLAW_QA_CONVEX_SECRET_CI`，用于 `ci`
- 凭据角色选择：
  - CLI：`--credential-role maintainer|ci`
  - 环境变量默认值：`OPENCLAW_QA_CREDENTIAL_ROLE`（在 CI 中默认为 `ci`，其他情况下默认为 `maintainer`）

可选环境变量：

- `OPENCLAW_QA_CREDENTIAL_LEASE_TTL_MS`（默认值为 `1200000`）
- `OPENCLAW_QA_CREDENTIAL_HEARTBEAT_INTERVAL_MS`（默认值为 `30000`）
- `OPENCLAW_QA_CREDENTIAL_ACQUIRE_TIMEOUT_MS`（默认值为 `90000`）
- `OPENCLAW_QA_CREDENTIAL_HTTP_TIMEOUT_MS`（默认值为 `15000`）
- `OPENCLAW_QA_CONVEX_ENDPOINT_PREFIX`（默认值为 `/qa-credentials/v1`）
- `OPENCLAW_QA_CREDENTIAL_OWNER_ID`（可选跟踪 ID）
- `OPENCLAW_QA_ALLOW_INSECURE_HTTP=1` 允许将 local loopback `http://` Convex URL 用于仅限本地的开发。

正常运行时，`OPENCLAW_QA_CONVEX_SITE_URL` 应使用 `https://`。

维护者管理命令（池的添加/移除/列出）明确要求使用
`OPENCLAW_QA_CONVEX_SECRET_MAINTAINER`。

维护者 CLI 辅助命令：

```bash
pnpm openclaw qa credentials doctor
pnpm openclaw qa credentials add --kind telegram --payload-file qa/telegram-credential.json
pnpm openclaw qa credentials list --kind telegram
pnpm openclaw qa credentials remove --credential-id <credential-id>
```

在实时运行前使用 `doctor`，可在不打印密钥值的情况下检查 Convex 站点 URL、
代理密钥、端点前缀、HTTP 超时以及管理/列表可访问性。在脚本和 CI
实用程序中使用 `--json` 获取机器可读输出。

默认端点契约（`OPENCLAW_QA_CONVEX_SITE_URL` + `/qa-credentials/v1`）。
请求使用 `Authorization: Bearer <role secret>` 标头进行身份验证；
以下请求正文省略该标头：

- `POST /acquire`
  - 请求：`{ kind, ownerId, actorRole, leaseTtlMs, heartbeatIntervalMs }`
  - 成功：`{ status: "ok", credentialId, leaseToken, payload, leaseTtlMs?, heartbeatIntervalMs? }`
  - 池耗尽/可重试：`{ status: "error", code: "POOL_EXHAUSTED" | "NO_CREDENTIAL_AVAILABLE", ... }`
- `POST /payload-chunk`
  - 请求：`{ kind, ownerId, actorRole, credentialId, leaseToken, index }`
  - 成功：`{ status: "ok", index, data }`
- `POST /heartbeat`
  - 请求：`{ kind, ownerId, actorRole, credentialId, leaseToken, leaseTtlMs }`
  - 成功：`{ status: "ok" }`（或空的 `2xx`）
- `POST /release`
  - 请求：`{ kind, ownerId, actorRole, credentialId, leaseToken }`
  - 成功：`{ status: "ok" }`（或空的 `2xx`）
- `POST /admin/add`（仅限维护者密钥）
  - 请求：`{ kind, actorId, payload, note?, status? }`
  - 成功：`{ status: "ok", credential }`
- `POST /admin/remove`（仅限维护者密钥）
  - 请求：`{ credentialId, actorId }`
  - 成功：`{ status: "ok", changed, credential }`
  - 活跃租约防护：`{ status: "error", code: "LEASE_ACTIVE", ... }`
- `POST /admin/list`（仅限维护者密钥）
  - 请求：`{ kind?, status?, includePayload?, limit? }`
  - 成功：`{ status: "ok", credentials, count }`

Telegram 类型的载荷结构：

- `{ groupId: string, driverToken: string, sutToken: string }`
- `groupId` 必须是数字形式的 Telegram 聊天 ID 字符串。
- `admin/add` 会验证 `kind: "telegram"` 的此结构，并拒绝格式错误的载荷。

Telegram 真实用户类型的载荷结构：

- `{ groupId: string, sutToken: string, testerUserId: string, testerUsername: string, telegramApiId: string, telegramApiHash: string, tdlibDatabaseEncryptionKey: string, tdlibArchiveBase64: string, tdlibArchiveSha256: string, desktopTdataArchiveBase64: string, desktopTdataArchiveSha256: string }`
- `groupId`、`testerUserId` 和 `telegramApiId` 必须是数字字符串。
- `tdlibArchiveSha256` 和 `desktopTdataArchiveSha256` 必须是 SHA-256 十六进制字符串。
- `kind: "telegram-user"` 专用于 Mantis Telegram Desktop 证明工作流。通用 QA Lab 通道不得获取它。

由代理验证的多渠道载荷：

- Discord：`{ guildId: string, channelId: string, driverBotToken: string, sutBotToken: string, sutApplicationId: string, voiceChannelId?: string }`
- WhatsApp：`{ driverPhoneE164: string, sutPhoneE164: string, driverAuthArchiveBase64: string, sutAuthArchiveBase64: string, groupJid?: string }`

Slack 通道也可以从池中租用凭据，但 Slack 载荷验证
目前位于 Slack QA 运行器中，而非代理中。对 Slack 记录使用
`{ channelId: string, driverBotToken: string, sutBotToken: string, sutAppToken: string }`。

### 向 QA 添加渠道

新渠道适配器的架构和场景辅助函数名称位于
[QA overview - 添加渠道](/zh-CN/concepts/qa-e2e-automation#adding-a-channel)。
最低要求：在共享 `qa-lab` 主机接缝上实现传输运行器，
为共享场景添加 `adapterFactory`，在插件清单中声明 `qaRunners`，
挂载为 `openclaw qa <runner>`，并在 `qa/scenarios/` 下编写场景。

## 测试套件（在哪里运行哪些测试）

可以将这些套件视为“真实性逐渐提高”（不稳定性/成本也随之增加）。

### 单元测试/集成测试（默认）

- 命令：`pnpm test`
- 配置：无目标运行使用 `vitest.full-*.config.ts` 分片集，并可能
  将多项目分片展开为各项目配置，以便并行
  调度
- 文件：`src/**/*.test.ts`、
  `packages/**/*.test.ts` 和 `test/**/*.test.ts` 下的核心/单元测试清单；UI 单元测试在
  专用的 `unit-ui` 分片中运行
- 范围：
  - 纯单元测试
  - 进程内集成测试（Gateway 网关身份验证、路由、工具、解析、配置）
  - 已知错误的确定性回归测试
- 预期：
  - 在 CI 中运行
  - 无需真实密钥
  - 应快速且稳定
  - 解析器和公共表面加载器测试必须使用生成的微型插件夹具证明广泛的 `api.js`
    和 `runtime-api.js` 回退行为，而非真实的内置插件源 API。真实插件 API
    加载应由插件自身的契约/集成测试套件负责。

原生依赖策略：

- 默认测试安装会跳过可选的 Discord 原生 opus 构建。Discord
  语音使用内置的 `libopus-wasm`，并且 `@discordjs/opus` 在
  `allowBuilds` 中保持禁用，因此本地测试和 Testbox 通道不会编译原生
  插件。
- 请在 `libopus-wasm` 基准测试仓库中比较原生 opus 性能，而不是
  在默认 OpenClaw 安装/测试循环中比较。不要在默认 `allowBuilds` 中将
  `@discordjs/opus` 设置为 `true`；这会导致无关的安装/测试
  循环编译原生代码。

<AccordionGroup>
  <Accordion title="项目、分片和限定范围的通道">

    - 未指定目标的 `pnpm test` 会运行十三个较小的分片配置（`core-unit-fast`、`core-unit-src`、`core-unit-security`、`core-unit-ui`、`core-unit-support`、`core-support-boundary`、`core-tooling`、`core-contracts`、`core-bundled`、`core-runtime`、`agentic`、`auto-reply`、`extensions`），而不是一个庞大的原生根项目进程。这可以降低高负载机器上的 RSS 峰值，并避免自动回复/插件工作使无关测试套件资源不足。
    - `pnpm test --watch` 仍使用原生根 `vitest.config.ts` 项目图，因为多分片监视循环并不实用。
    - `pnpm test`、`pnpm test:watch` 和 `pnpm test:perf:imports` 会先通过限定范围的通道来路由明确的文件/目录目标，因此 `pnpm test extensions/discord/src/monitor/message-handler.preflight.test.ts` 无需承担完整根项目的启动开销。
    - `pnpm test:changed` 默认会将 Git 中已更改的路径扩展到低开销的限定范围通道：直接测试编辑、同级 `*.test.ts` 文件、显式源映射以及本地导入图中的依赖方。配置/设置/软件包编辑不会广泛运行测试，除非你明确使用 `OPENCLAW_TEST_CHANGED_BROAD=1 pnpm test:changed`。
    - `pnpm check:changed` 是窄范围工作的常规智能本地检查门禁。它将差异分类为核心、核心测试、插件、插件测试、应用、文档、发布元数据、实时 Docker 工具和工具链，然后运行对应的类型检查、lint 和防护命令。它不会运行 Vitest 测试；请调用 `pnpm test:changed` 或显式的 `pnpm test <target>` 来提供测试证明。仅涉及发布元数据的版本升级会运行针对性的版本/配置/根依赖检查，并通过防护机制拒绝顶层版本字段以外的软件包更改。
    - 实时 Docker ACP harness 编辑会运行聚焦检查：检查实时 Docker 身份验证脚本的 shell 语法，并对实时 Docker 调度器执行试运行。仅当差异局限于 `scripts["test:docker:live-*"]` 时，才会包含 `package.json` 更改；依赖、导出、版本和其他软件包表面编辑仍使用更广泛的防护。
    - 来自智能体、命令、插件、自动回复辅助程序、`plugin-sdk` 和类似纯工具区域的轻量导入单元测试会通过 `unit-fast` 通道路由，该通道会跳过 `test/setup-openclaw-runtime.ts`；有状态/运行时密集型文件仍使用现有通道。
    - 选定的 `plugin-sdk` 和 `commands` 辅助源文件还会将更改模式运行映射到这些轻量通道中的显式同级测试，因此辅助程序编辑无需重新运行该目录的完整重型测试套件。
    - `auto-reply` 为顶层核心辅助程序、顶层 `reply.*` 集成测试和 `src/auto-reply/reply/**` 子树设置了专用分组。CI 还会进一步将回复子树拆分为 agent-runner、分发和命令/状态路由分片，使单个导入密集型分组不必独占完整的 Node 尾部耗时。
    - 常规 PR/main CI 会有意跳过内置插件批量扫描和仅用于发布的 `agentic-plugins` 分片。完整发布验证会针对候选版本分派单独的 `Plugin Prerelease` 子工作流，以运行这些插件密集型测试套件。

  </Accordion>

  <Accordion title="嵌入式运行器覆盖范围">

    - 更改消息工具发现输入或压缩运行时
      上下文时，请保留两个层级的覆盖范围。
    - 为纯路由和规范化
      边界添加聚焦的辅助程序回归测试。
    - 保持嵌入式运行器集成测试套件正常运行：
      `src/agents/embedded-agent-runner/compact.hooks.test.ts`、
      `src/agents/embedded-agent-runner/run.overflow-compaction.test.ts` 和
      `src/agents/embedded-agent-runner/run.overflow-compaction.loop.test.ts`。
    - 这些测试套件验证限定范围的 ID 和压缩行为仍会流经
      真实的 `run.ts` / `compact.ts` 路径；仅使用辅助程序测试
      不足以替代这些集成路径。

  </Accordion>

  <Accordion title="Vitest 池和隔离默认值">

    - 基础 Vitest 配置默认为 `threads`。
    - 共享 Vitest 配置固定了 `isolate: false`，并在
      根项目、端到端配置和实时配置中使用非隔离运行器。
    - 根 UI 通道保留其 `jsdom` 设置和优化器，但也在
      共享的非隔离运行器上运行。
    - 每个 `pnpm test` 分片都从共享 Vitest 配置继承相同的 `threads` + `isolate: false`
      默认值。
    - `scripts/run-vitest.mjs` 默认会为 Vitest 子 Node
      进程添加 `--no-maglev`，以减少大型本地运行期间的 V8 编译抖动。
      设置 `OPENCLAW_VITEST_ENABLE_MAGLEV=1` 可与原始 V8
      行为进行比较。
    - `scripts/run-vitest.mjs` 会终止连续 5 分钟
      没有 stdout 或 stderr 输出的显式非监视 Vitest 运行。对于
      有意保持静默的调查，请设置 `OPENCLAW_VITEST_NO_OUTPUT_TIMEOUT_MS=0`
      以禁用看门狗。

  </Accordion>

  <Accordion title="快速本地迭代">

    - `pnpm changed:lanes` 会显示差异触发了哪些架构通道。
    - 预提交钩子仅执行格式化。它会重新暂存已格式化的文件，
      不会运行 lint、类型检查或测试。
    - 在交接或推送前需要智能本地检查门禁时，请显式运行
      `pnpm check:changed`。
    - `pnpm test:changed` 默认通过低开销的限定范围通道路由。仅当
      智能体确定 harness、配置、软件包或契约编辑确实需要
      更广泛的 Vitest 覆盖范围时，才使用 `OPENCLAW_TEST_CHANGED_BROAD=1 pnpm test:changed`。
    - `pnpm test:max` 和 `pnpm test:changed:max` 保持相同的路由
      行为，只是工作进程上限更高。
    - 本地工作进程自动扩缩策略有意保持保守，并会在
      主机平均负载已经很高时缩减，因此默认情况下多个并发
      Vitest 运行造成的影响更小。
    - 基础 Vitest 配置将项目/配置文件标记为
      `forceRerunTriggers`，以便测试
      接线更改时，更改模式的重新运行仍然正确。
    - 该配置在支持的主机上保持启用
      `OPENCLAW_VITEST_FS_MODULE_CACHE`；设置 `OPENCLAW_VITEST_FS_MODULE_CACHE_PATH=/abs/path`
      可为直接性能分析指定一个明确的缓存位置。

  </Accordion>

  <Accordion title="性能调试">

    - `pnpm test:perf:imports` 会启用 Vitest 导入时长报告以及
      导入细分输出。
    - `pnpm test:perf:imports:changed` 会将相同的性能分析视图限定到
      自 `origin/main` 以来更改的文件。
    - 分片计时数据会写入 `.artifacts/vitest-shard-timings.json`。
      整体配置运行使用配置路径作为键；包含模式 CI
      分片会附加分片名称，以便单独跟踪经过筛选的分片。
    - 当某个热点测试的大部分时间仍消耗在启动导入上时，
      请将重型依赖置于一个狭窄的本地 `*.runtime.ts` 接缝之后，并
      直接模拟该接缝，而不要深度导入运行时辅助程序，
      仅仅为了通过 `vi.mock(...)` 传递它们。
    - `pnpm test:perf:changed:bench -- --ref <git-ref>` 会针对该已提交差异，将路由后的
      `test:changed` 与原生根项目路径进行比较，并输出实际耗时以及 macOS 最大 RSS。
    - `pnpm test:perf:changed:bench -- --worktree` 通过将已更改文件列表路由至
      `scripts/test-projects.mjs` 和根 Vitest 配置，对当前
      脏工作树进行基准测试。
    - `pnpm test:perf:profile:main` 会为
      Vitest/Vite 启动和转换开销写入主线程 CPU 性能分析。
    - `pnpm test:perf:profile:runner` 会在禁用文件并行的情况下，
      为单元测试套件写入运行器 CPU+堆性能分析。

  </Accordion>
</AccordionGroup>

### 稳定性（Gateway 网关）

- 命令：`pnpm test:stability:gateway`
- 配置：`test/vitest/vitest.gateway.config.ts`、`test/vitest/vitest.logging.config.ts` 和 `test/vitest/vitest.infra.config.ts`，每个都强制使用一个工作进程
- 范围：
  - 启动一个真实的 loopback Gateway 网关，并默认启用诊断
  - 通过诊断事件路径驱动合成的 Gateway 网关消息、记忆和大型负载抖动
  - 通过 Gateway 网关 WS RPC 查询 `diagnostics.stability`
  - 覆盖诊断稳定性包的持久化辅助程序
  - 断言记录器保持有界、合成 RSS 样本保持在压力预算以下，并且每个会话的队列深度都回落至零
- 预期：
  - 可安全用于 CI，且无需密钥
  - 用于跟进稳定性回归的窄范围通道，不能替代完整的 Gateway 网关测试套件

### 端到端测试（仓库聚合）

- 命令：`pnpm test:e2e`
- 范围：
  - 运行 Gateway 网关冒烟端到端测试通道
  - 运行模拟的 Control UI 浏览器端到端测试通道
- 预期：
  - 可安全用于 CI，且无需密钥
  - 需要安装 Playwright Chromium

### 端到端测试（Gateway 网关冒烟测试）

- 命令：`pnpm test:e2e:gateway`
- 配置：`test/vitest/vitest.e2e.config.ts`
- 文件：`src/**/*.e2e.test.ts`、`test/**/*.e2e.test.ts`，以及 `extensions/` 下的内置插件端到端测试
- 运行时默认值：
  - 使用带有 `isolate: false` 的 Vitest `threads`，与仓库其余部分保持一致。
  - 使用自适应工作进程（CI：最多 2 个；本地：默认为 1 个）。
  - 默认以静默模式运行，以减少控制台 I/O 开销。
- 实用覆盖选项：
  - `OPENCLAW_E2E_WORKERS=<n>` 用于强制指定工作进程数（上限为 16）。
  - `OPENCLAW_E2E_VERBOSE=1` 用于重新启用详细控制台输出。
- 范围：
  - 多实例 Gateway 网关端到端行为
  - WebSocket/HTTP 表面、节点配对和较重的网络功能
- 预期：
  - 在 CI 中运行（在流水线中启用时）
  - 无需真实密钥
  - 比单元测试涉及更多活动部件（可能更慢）

### 端到端测试（Control UI 模拟浏览器）

- 命令：`pnpm test:ui:e2e`
- 配置：`test/vitest/vitest.ui-e2e.config.ts`
- 文件：`ui/src/**/*.e2e.test.ts`
- 范围：
  - 启动 Vite Control UI
  - 通过 Playwright 驱动真实 Chromium 页面
  - 使用确定性的浏览器内模拟替换 Gateway 网关 WebSocket
- 预期：
  - 作为 `pnpm test:e2e` 的一部分在 CI 中运行
  - 无需真实的 Gateway 网关、智能体或提供商密钥
  - 必须存在浏览器依赖（`pnpm --dir ui exec playwright install chromium`）

### 端到端测试：OpenShell 后端冒烟测试

- 命令：`pnpm test:e2e:openshell`
- 文件：`extensions/openshell/src/backend.e2e.test.ts`
- 范围：
  - 复用活跃的本地 OpenShell Gateway 网关
  - 从临时本地 Dockerfile 创建沙箱
  - 通过真实的 `sandbox ssh-config` + SSH exec 对 OpenClaw 的 OpenShell 后端执行测试
  - 通过沙箱文件系统桥验证远程规范文件系统行为
- 预期：
  - 仅可选择性启用；不属于默认 `pnpm test:e2e` 运行
  - 需要本地 `openshell` CLI 和正常工作的 Docker 守护进程
  - 需要活跃的本地 OpenShell Gateway 网关及其配置源
  - 使用隔离的 `HOME` / `XDG_CONFIG_HOME`，随后销毁测试沙箱
- 实用覆盖选项：
  - `OPENCLAW_E2E_OPENSHELL=1` 用于在手动运行更广泛的端到端测试套件时启用该测试
  - `OPENCLAW_E2E_OPENSHELL_COMMAND=/path/to/openshell` 用于指向非默认 CLI 二进制文件或包装脚本
  - `OPENCLAW_E2E_OPENSHELL_CONFIG_HOME=/path/to/config` 用于向隔离测试公开已注册的 Gateway 网关配置
  - `OPENCLAW_E2E_OPENSHELL_HOST_IP=172.18.0.1` 用于覆盖主机策略夹具所使用的 Docker Gateway 网关 IP

### 实时测试（真实提供商 + 真实模型）

- 命令：`pnpm test:live`
- 配置：`test/vitest/vitest.live.config.ts`
- 文件：`src/**/*.live.test.ts`、`test/**/*.live.test.ts`，以及 `extensions/` 下的内置插件实时测试
- 默认值：由 `pnpm test:live` 设置为**启用**（设置 `OPENCLAW_LIVE_TEST=1`）
- 范围：
  - “此提供商/模型使用真实凭据在_今天_是否确实可用？”
  - 捕获提供商格式变更、工具调用特殊行为、身份验证问题和速率限制行为
- 预期：
  - 按设计不保证在 CI 中稳定（真实网络、真实提供商策略、配额和服务中断）
  - 会产生费用/消耗速率限制配额
  - 优先运行范围缩小的子集，而不是“全部”
- 实时运行使用已导出的 API 密钥和暂存的身份验证配置文件。
- 默认情况下，实时运行仍会隔离 `HOME`，并将配置/身份验证材料复制到临时测试主目录中，因此单元测试夹具无法修改你真实的 `~/.openclaw`。
- 仅当你有意让实时测试使用真实主目录时，才设置 `OPENCLAW_LIVE_USE_REAL_HOME=1`。
- `pnpm test:live` 默认采用更安静的模式：保留 `[live] ...` 进度输出，并静默 Gateway 网关启动日志/Bonjour 杂讯。如果要恢复完整的启动日志，请设置 `OPENCLAW_LIVE_TEST_QUIET=0`。
- API 密钥轮换（特定于提供商）：使用逗号/分号格式设置 `*_API_KEYS`，或设置 `*_API_KEY_1`、`*_API_KEY_2`（例如 `OPENAI_API_KEYS`、`ANTHROPIC_API_KEYS`、`GEMINI_API_KEYS`），也可通过 `OPENCLAW_LIVE_*_KEY` 针对每次实时运行进行覆盖；测试遇到速率限制响应时会重试。
- 进度/心跳输出：
  - 实时测试套件会向 stderr 输出进度行，因此即使 Vitest 控制台捕获没有输出，也能直观看到耗时较长的提供商调用仍处于活动状态。
  - `test/vitest/vitest.live.config.ts` 会禁用 Vitest 控制台拦截，使提供商/Gateway 网关进度行在实时运行期间立即流式输出。
  - 使用 `OPENCLAW_LIVE_HEARTBEAT_MS` 调整直接模型的心跳。
  - 使用 `OPENCLAW_LIVE_GATEWAY_HEARTBEAT_MS` 调整 Gateway 网关/探测的心跳。

## 应该运行哪个测试套件？

使用此决策表：

- 编辑逻辑/测试：运行 `pnpm test`（如果改动较多，也运行 `pnpm test:coverage`）
- 涉及 Gateway 网关网络/WS 协议/配对：添加 `pnpm test:e2e`
- 调试“我的机器人宕机了”/特定提供商故障/工具调用：运行范围缩小的 `pnpm test:live`

## 实时（访问网络的）测试

有关实时模型矩阵、CLI 后端冒烟测试、ACP 冒烟测试、Codex app-server
harness，以及所有媒体提供商实时测试（Deepgram、BytePlus、ComfyUI、
图像、音乐、视频、媒体 harness）和实时运行的凭据处理，

- 请参阅[测试实时套件](/zh-CN/help/testing-live)。有关专门的更新和
  插件验证清单，请参阅
  [更新和插件测试](/zh-CN/help/testing-updates-plugins)。

## Docker 运行器（可选的“在 Linux 中可用”检查）

这些 Docker 运行器分为两类：

- 实时模型运行器：`test:docker:live-models` 和 `test:docker:live-gateway` 仅在仓库 Docker 镜像（`src/agents/models.profiles.live.test.ts` 和 `src/gateway/gateway-models.profiles.live.test.ts`）内运行各自匹配的配置文件密钥实时文件，并挂载本地配置目录、工作区和可选的配置文件环境变量文件。对应的本地入口点为 `test:live:models-profiles` 和 `test:live:gateway-profiles`。
- Docker 实时运行器会在必要时保留各自的实用上限：
  `test:docker:live-models` 默认为精选的受支持高信号集合，而
  `test:docker:live-gateway` 默认为 `OPENCLAW_LIVE_GATEWAY_SMOKE=1`、
  `OPENCLAW_LIVE_GATEWAY_MAX_MODELS=8`、
  `OPENCLAW_LIVE_GATEWAY_STEP_TIMEOUT_MS=45000` 和
  `OPENCLAW_LIVE_GATEWAY_MODEL_TIMEOUT_MS=90000`。仅当明确需要更小的上限或更大范围的扫描时，才设置 `OPENCLAW_LIVE_MAX_MODELS`
  或 Gateway 网关环境变量。
- `test:docker:all` 通过 `test:docker:live-build` 构建一次实时 Docker 镜像，通过 `scripts/package-openclaw-for-docker.mjs` 将 OpenClaw 打包一次为 npm tarball，然后构建/复用两个 `scripts/e2e/Dockerfile` 镜像。基础镜像仅作为 Node/Git 运行器，用于安装/更新/插件依赖通道；这些通道会挂载预构建的 tarball。功能镜像将同一个 tarball 安装到 `/app`，用于已构建应用的功能通道。Docker 通道定义位于 `scripts/lib/docker-e2e-scenarios.mjs`；规划器逻辑位于 `scripts/lib/docker-e2e-plan.mjs`；`scripts/test-docker-all.mjs` 执行选定的计划。聚合运行使用加权本地调度器：`OPENCLAW_DOCKER_ALL_PARALLELISM` 控制进程槽位，而资源上限可防止高负载实时通道、npm 安装通道和多服务通道同时全部启动。如果某个通道的负载超过当前上限，调度器仍可在池为空时启动它，并让其独占运行，直到再次有可用容量。默认值为 10 个槽位、`OPENCLAW_DOCKER_ALL_LIVE_LIMIT=9`、`OPENCLAW_DOCKER_ALL_NPM_LIMIT=5` 和 `OPENCLAW_DOCKER_ALL_SERVICE_LIMIT=7`；仅当 Docker 主机有更多余量时，才调整 `OPENCLAW_DOCKER_ALL_WEIGHT_LIMIT` 或 `OPENCLAW_DOCKER_ALL_DOCKER_LIMIT`（以及其他 `OPENCLAW_DOCKER_ALL_<RESOURCE>_LIMIT` 覆盖项）。运行器默认执行 Docker 预检，移除陈旧的 OpenClaw E2E 容器，每 30 秒输出一次状态，将成功通道的耗时存储在 `.artifacts/docker-tests/lane-timings.json` 中，并在后续运行中利用这些耗时优先启动较长的通道。使用 `OPENCLAW_DOCKER_ALL_DRY_RUN=1` 可输出加权通道清单而不构建或运行 Docker；使用 `node scripts/test-docker-all.mjs --plan-json` 可输出选定通道、软件包/镜像需求和凭据的 CI 计划。
- `Package Acceptance` 是 GitHub 原生软件包门禁，用于验证“这个可安装的 tarball 能否作为产品正常工作？”它从 `source=npm`、`source=ref`、`source=url`、`source=trusted-url` 或 `source=artifact` 中解析一个候选软件包，将其作为 `package-under-test` 上传，然后针对该确切 tarball 运行可复用的 Docker E2E 通道，而不是重新打包选定的 ref。配置文件按覆盖范围排序：`smoke`、`package`、`product` 和 `full`（另有 `custom` 用于显式通道列表）。有关软件包/更新/插件契约、已发布升级存续矩阵、发布默认值和故障分类，请参阅[更新和插件测试](/zh-CN/help/testing-updates-plugins)。
- 构建和发布检查会在 tsdown 后运行 `scripts/check-cli-bootstrap-imports.mjs`。该防护会从 `dist/entry.js` 和 `dist/cli/run-main.js` 遍历静态构建图；如果这个调度前启动图在命令调度前静态导入任何外部软件包（Commander、提示 UI、undici、日志及类似的启动高负载依赖项均计入），检查将失败；它还会将内置 Gateway 网关运行分块限制为 70 KB，并拒绝该分块静态导入已知的冷 Gateway 网关路径（`control-ui-assets`、`diagnostic-stability-bundle`、`onboard-helpers`、`process-respawn`、`restart-sentinel`、`server-close`、`server-reload-handlers`）。`scripts/release-check.ts` 会另外使用 `--help`、`onboard --help`、`doctor --help`、`status --json --timeout 1`、`config schema` 和 `models list --provider openai` 对已打包的 CLI 进行冒烟测试。
- Package Acceptance 的旧版兼容性截止到 `2026.4.25`（包括 `2026.4.25-beta.*`）。在此截止版本及之前，harness 仅容忍已发布软件包的元数据缺口：省略私有 QA 清单条目、缺少 `gateway install --wrapper`、tarball 派生的 git 夹具中缺少补丁文件、缺少持久化的 `update.channel`、旧版插件安装记录位置、缺少市场安装记录持久化，以及在 `plugins update` 期间迁移配置元数据。对于 `2026.4.25` 之后的软件包，这些情况会导致严格失败。
- 容器冒烟测试运行器：`test:docker:openwebui`、`test:docker:onboard`、`test:docker:npm-onboard-channel-agent`、`test:docker:release-user-journey`、`test:docker:release-typed-onboarding`、`test:docker:release-media-memory`、`test:docker:release-upgrade-user-journey`、`test:docker:release-plugin-marketplace`、`test:docker:skill-install`、`test:docker:update-channel-switch`、`test:docker:upgrade-survivor`、`test:docker:published-upgrade-survivor`、`test:docker:session-runtime-context`、`test:docker:agents-delete-shared-workspace`、`test:docker:gateway-network`、`test:docker:browser-cdp-snapshot`、`test:docker:mcp-channels`、`test:docker:agent-bundle-mcp-tools`、`test:docker:cron-mcp-cleanup`、`test:docker:plugins`、`test:docker:plugin-update`、`test:docker:plugin-lifecycle-matrix` 和 `test:docker:config-reload` 会启动一个或多个真实容器，并验证更高层级的集成路径。
- 通过 `scripts/lib/openclaw-e2e-instance.sh` 安装已打包 OpenClaw tarball 的 Docker/Bash E2E 通道，会将 `npm install` 限制为 `OPENCLAW_E2E_NPM_INSTALL_TIMEOUT`（默认值为 `600s`；设置 `0` 可禁用包装器以便调试）。

实时模型 Docker 运行器还只会绑定挂载所需的 CLI 身份验证主目录
（如果运行范围未缩小，则挂载所有受支持的主目录），随后在运行前将其复制到
容器主目录中，使外部 CLI OAuth 能够刷新令牌，
而不会修改主机身份验证存储：

- 直接模型：`pnpm test:docker:live-models`（脚本：`scripts/test-live-models-docker.sh`）
- ACP 绑定冒烟测试：`pnpm test:docker:live-acp-bind`（脚本：`scripts/test-live-acp-bind-docker.sh`；默认涵盖 Claude、Codex 和 Gemini，并通过 `pnpm test:docker:live-acp-bind:droid` 和 `pnpm test:docker:live-acp-bind:opencode` 严格涵盖 Droid/OpenCode）
- CLI 后端冒烟测试：`pnpm test:docker:live-cli-backend`（脚本：`scripts/test-live-cli-backend-docker.sh`）
- Codex app-server harness 冒烟测试：`pnpm test:docker:live-codex-harness`（脚本：`scripts/test-live-codex-harness-docker.sh`）
- Gateway 网关 + 开发智能体：`pnpm test:docker:live-gateway`（脚本：`scripts/test-live-gateway-models-docker.sh`）
- 可观测性冒烟测试：`pnpm qa:otel:smoke`、`pnpm qa:prometheus:smoke` 和 `pnpm qa:observability:smoke` 是私有 QA 源码检出通道。它们有意不属于软件包 Docker 发布通道，因为 npm tarball 不包含 QA Lab。
- Open WebUI 实时冒烟测试：`pnpm test:docker:openwebui`（脚本：`scripts/e2e/openwebui-docker.sh`）
- 新手引导向导（TTY，完整脚手架）：`pnpm test:docker:onboard`（脚本：`scripts/e2e/onboard-docker.sh`）
- Npm tarball 新手引导/渠道/智能体冒烟测试：`pnpm test:docker:npm-onboard-channel-agent` 会在 Docker 中全局安装已打包的 OpenClaw tarball，默认通过环境变量引用新手引导配置 OpenAI 和 Telegram，运行 Doctor，并运行一次模拟的 OpenAI 智能体轮次。使用 `OPENCLAW_CURRENT_PACKAGE_TGZ=/path/to/openclaw-*.tgz` 可复用预构建的 tarball，使用 `OPENCLAW_NPM_ONBOARD_HOST_BUILD=0` 可跳过主机重新构建，或使用 `OPENCLAW_NPM_ONBOARD_CHANNEL=discord` 或 `OPENCLAW_NPM_ONBOARD_CHANNEL=slack` 切换渠道。

- 发布版用户旅程冒烟测试：`pnpm test:docker:release-user-journey` 在干净的 Docker 主目录中全局安装打包后的 OpenClaw tarball，运行新手引导，配置模拟的 OpenAI provider，运行一次智能体轮次，安装/卸载外部插件，针对本地 fixture 配置 ClickClack，验证出站/入站消息，重启 Gateway 网关，并运行 Doctor。
- 发布版类型化新手引导冒烟测试：`pnpm test:docker:release-typed-onboarding` 安装打包后的 tarball，通过真实 TTY 驱动 `openclaw onboard`，将 OpenAI 配置为 env-ref provider，验证不会持久化原始密钥，并运行一次模拟的智能体轮次。
- 发布版媒体/记忆冒烟测试：`pnpm test:docker:release-media-memory` 安装打包后的 tarball，验证对 PNG 附件的图像理解、OpenAI 兼容的图像生成输出、记忆搜索召回，以及 Gateway 网关重启后召回能力仍然保留。
- 发布版升级用户旅程冒烟测试：`pnpm test:docker:release-upgrade-user-journey` 默认安装已发布且早于候选 tarball 的最新基线，在已发布的软件包上配置提供商/插件/ClickClack 状态，升级到候选 tarball，然后重新运行核心智能体/插件/渠道旅程。如果不存在更早的已发布基线，则复用候选版本。使用 `OPENCLAW_RELEASE_UPGRADE_BASELINE_SPEC=openclaw@<version>` 覆盖基线。
- 发布版插件市场冒烟测试：`pnpm test:docker:release-plugin-marketplace` 从本地 fixture 市场安装，更新已安装的插件，将其卸载，并验证插件 CLI 随着安装元数据被清理而消失。
- 技能安装冒烟测试：`pnpm test:docker:skill-install` 在 Docker 中全局安装打包后的 OpenClaw tarball，在配置中禁用上传归档安装，通过搜索解析当前线上 ClawHub 技能 slug，使用 `openclaw skills install` 安装该技能，并验证已安装的技能以及 `.clawhub` 来源/锁定元数据。
- 更新渠道切换冒烟测试：`pnpm test:docker:update-channel-switch` 在 Docker 中全局安装打包后的 OpenClaw tarball，从软件包 `stable` 切换到 git `dev`，验证持久化的渠道和插件更新后工作正常，然后切换回软件包 `stable` 并检查更新状态。
- 升级存续冒烟测试：`pnpm test:docker:upgrade-survivor` 在一个含智能体、渠道配置、插件允许列表、过时插件依赖状态和现有工作区/会话文件的脏旧用户 fixture 上安装打包后的 OpenClaw tarball。它在没有线上提供商或渠道密钥的情况下运行软件包更新和非交互式 Doctor，然后启动 local loopback Gateway 网关，并检查配置/状态保留以及启动/状态时间预算。
- 已发布版本升级存续冒烟测试：`pnpm test:docker:published-upgrade-survivor` 默认安装 `openclaw@latest`，植入真实的现有用户文件，使用内置命令方案配置该基线，验证生成的配置，将已发布的安装更新到候选 tarball，运行非交互式 Doctor，写入 `.artifacts/upgrade-survivor/summary.json`，然后启动 local loopback Gateway 网关，并检查已配置的意图、状态保留、启动、`/healthz`、`/readyz` 和 RPC 状态时间预算。使用 `OPENCLAW_UPGRADE_SURVIVOR_BASELINE_SPEC` 覆盖一个基线；通过 `OPENCLAW_UPGRADE_SURVIVOR_BASELINE_SPECS`（例如 `openclaw@2026.5.2 openclaw@2026.4.23 openclaw@2026.4.15`）要求聚合调度器展开精确的本地基线；通过 `OPENCLAW_UPGRADE_SURVIVOR_SCENARIOS`（例如 `reported-issues`）展开问题形态的 fixture；已报告问题集合包含 `configured-plugin-installs`，用于自动修复外部 OpenClaw 插件安装。Package Acceptance 将这些公开为 `published_upgrade_survivor_baseline`、`published_upgrade_survivor_baselines` 和 `published_upgrade_survivor_scenarios`，解析 `last-stable-4` 或 `all-since-2026.4.23` 等元基线令牌，而 Full Release Validation 将发布浸泡软件包门禁展开为 `last-stable-4 2026.4.23 2026.5.2 2026.4.15` 加 `reported-issues`。
- 会话运行时上下文冒烟测试：`pnpm test:docker:session-runtime-context` 验证隐藏运行时上下文的转录持久化，以及 Doctor 对受影响的重复提示词重写分支的修复。
- Bun 全局安装冒烟测试：`bash scripts/e2e/bun-global-install-smoke.sh` 打包当前代码树，在隔离的主目录中使用 `bun install -g` 安装，并验证 `openclaw infer image providers --json` 会返回内置图像提供商而不是挂起。使用 `OPENCLAW_BUN_GLOBAL_SMOKE_PACKAGE_TGZ=/path/to/openclaw-*.tgz` 复用预构建的 tarball，使用 `OPENCLAW_BUN_GLOBAL_SMOKE_HOST_BUILD=0` 跳过主机构建，或使用 `OPENCLAW_BUN_GLOBAL_SMOKE_DIST_IMAGE=openclaw-dockerfile-smoke:local` 从已构建的 Docker 镜像复制 `dist/`。
- 安装程序 Docker 冒烟测试：`bash scripts/test-install-sh-docker.sh` 在其 root、更新和直接 npm 容器之间共享一个 npm 缓存。更新冒烟测试默认以 npm `latest` 作为稳定基线，然后再升级到候选 tarball。在本地使用 `OPENCLAW_INSTALL_SMOKE_UPDATE_BASELINE=2026.4.22` 覆盖，或在 GitHub 上使用 Install Smoke 工作流的 `update_baseline_version` 输入覆盖。非 root 安装程序检查会保留隔离的 npm 缓存，避免 root 所有的缓存条目掩盖用户本地安装行为。设置 `OPENCLAW_INSTALL_SMOKE_NPM_CACHE_DIR=/path/to/cache`，可在本地重新运行时复用 root/更新/直接 npm 缓存。
- Install Smoke CI 使用 `OPENCLAW_INSTALL_SMOKE_SKIP_NPM_GLOBAL=1` 跳过重复的直接 npm 全局更新；需要覆盖直接 `npm install -g` 时，请在本地运行脚本且不设置该环境变量。
- 智能体删除共享工作区 CLI 冒烟测试：`pnpm test:docker:agents-delete-shared-workspace`（脚本：`scripts/e2e/agents-delete-shared-workspace-docker.sh`）默认构建根 Dockerfile 镜像，在隔离的容器主目录中植入共享一个工作区的两个智能体，运行 `agents delete --json`，并验证 JSON 有效且工作区保留行为正确。使用 `OPENCLAW_AGENTS_DELETE_SHARED_WORKSPACE_E2E_IMAGE=openclaw-dockerfile-smoke:local OPENCLAW_AGENTS_DELETE_SHARED_WORKSPACE_E2E_SKIP_BUILD=1` 复用 install-smoke 镜像。
- Gateway 网关网络和主机生命周期：`pnpm test:docker:gateway-network`（脚本：`scripts/e2e/gateway-network-docker.sh`）保留双容器 LAN WebSocket 身份验证/健康冒烟测试，然后使用 local loopback Admin HTTP 证明准备阶段隔离、保留的控制访问、恢复后的复原能力，以及已准备的同容器停止/启动。重启检查必须在原始租约到期前完成，验证挂起状态仅存在于进程本地，而持久化的 Gateway 网关配置和容器身份仍然保留，并输出机器可读的阶段耗时 JSON。
- 浏览器 CDP 快照冒烟测试：`pnpm test:docker:browser-cdp-snapshot`（脚本：`scripts/e2e/browser-cdp-snapshot-docker.sh`）构建源代码 E2E 镜像及 Chromium 层，使用原始 CDP 启动 Chromium，运行 `browser doctor --deep`，并验证 CDP 角色快照覆盖链接 URL、由光标提升的可点击元素、iframe 引用和 frame 元数据。
- OpenAI Responses web_search 最小推理回归测试：`pnpm test:docker:openai-web-search-minimal`（脚本：`scripts/e2e/openai-web-search-minimal-docker.sh`）通过 Gateway 网关运行模拟的 OpenAI 服务器，验证 `web_search` 将 `reasoning.effort` 从 `minimal` 提高到 `low`，然后强制提供商 schema 拒绝请求，并检查原始详情是否出现在 Gateway 网关日志中。
- MCP 渠道桥接（已植入数据的 Gateway 网关 + stdio 桥接 + 原始 Claude 通知帧冒烟测试）：`pnpm test:docker:mcp-channels`（脚本：`scripts/e2e/mcp-channels-docker.sh`）
- OpenClaw bundle MCP 工具（真实 stdio MCP 服务器 + 嵌入式 OpenClaw 配置文件允许/拒绝冒烟测试）：`pnpm test:docker:agent-bundle-mcp-tools`（脚本：`scripts/e2e/agent-bundle-mcp-tools-docker.sh`）
- 定时任务/子智能体 MCP 清理（真实 Gateway 网关 + 隔离的定时任务和一次性子智能体运行后拆除 stdio MCP 子进程）：`pnpm test:docker:cron-mcp-cleanup`（脚本：`scripts/e2e/cron-mcp-cleanup-docker.sh`）
- 插件（针对本地路径、`file:`、具有提升依赖项的 npm 注册表、格式错误的 npm 软件包元数据、git 可移动引用、ClawHub kitchen-sink、市场更新以及 Claude bundle 启用/检查的安装/更新冒烟测试）：`pnpm test:docker:plugins`（脚本：`scripts/e2e/plugins-docker.sh`）
  设置 `OPENCLAW_PLUGINS_E2E_CLAWHUB=0` 可跳过 ClawHub 部分，或使用 `OPENCLAW_PLUGINS_E2E_CLAWHUB_SPEC` 和 `OPENCLAW_PLUGINS_E2E_CLAWHUB_ID` 覆盖默认的 kitchen-sink 软件包/运行时组合。如果未设置 `OPENCLAW_CLAWHUB_URL`/`CLAWHUB_URL`，测试将使用密闭的本地 ClawHub fixture 服务器。
- 插件更新无变化冒烟测试：`pnpm test:docker:plugin-update`（脚本：`scripts/e2e/plugin-update-unchanged-docker.sh`）
- 插件生命周期矩阵冒烟测试：`pnpm test:docker:plugin-lifecycle-matrix` 在空白容器中安装打包后的 OpenClaw tarball，安装一个 npm 插件，切换启用/禁用状态，通过本地 npm 注册表升级和降级该插件，删除已安装的代码，然后验证卸载仍会移除过时状态，同时记录每个生命周期阶段的 RSS/CPU 指标。
- 配置重载元数据冒烟测试：`pnpm test:docker:config-reload`（脚本：`scripts/e2e/config-reload-source-docker.sh`）
- 插件：`pnpm test:docker:plugins` 覆盖针对本地路径、`file:`、具有提升依赖项的 npm 注册表、git 可移动引用、ClawHub fixture、市场更新以及 Claude bundle 启用/检查的安装/更新冒烟测试。`pnpm test:docker:plugin-update` 覆盖已安装插件的无变化更新行为。`pnpm test:docker:plugin-lifecycle-matrix` 覆盖带资源跟踪的 npm 插件安装、启用、禁用、升级、降级和代码缺失时的卸载。

要手动预构建并复用共享功能镜像：

```bash
OPENCLAW_DOCKER_E2E_IMAGE=openclaw-docker-e2e-functional:local pnpm test:docker:e2e-build
OPENCLAW_DOCKER_E2E_IMAGE=openclaw-docker-e2e-functional:local OPENCLAW_SKIP_DOCKER_BUILD=1 pnpm test:docker:mcp-channels
```

设置后，`OPENCLAW_GATEWAY_NETWORK_E2E_IMAGE` 等测试套件专用镜像覆盖项仍然优先。当 `OPENCLAW_SKIP_DOCKER_BUILD=1` 指向远程共享镜像时，如果本地尚无该镜像，脚本会将其拉取下来。QR 和安装程序 Docker 测试保留各自的 Dockerfile，因为它们验证的是软件包/安装行为，而不是共享的已构建应用运行时。

实时模型 Docker 运行器还会以只读方式绑定挂载当前检出，
并将其暂存到容器内的临时工作目录中。这样既能保持
运行时镜像精简，又能让 Vitest 针对你的确切本地
源代码/配置运行。暂存步骤会跳过大型的仅限本地缓存和应用构建
输出，例如 `.pnpm-store`、`.worktrees`、`__openclaw_vitest__`，以及
应用本地的 `.build` 或 Gradle 输出目录，避免 Docker 实时运行
花费数分钟复制机器特定的工件。它们还会设置
`OPENCLAW_SKIP_CHANNELS=1`，防止 Gateway 网关实时探测在容器内启动真实的
Telegram/Discord 等渠道工作进程。
`test:docker:live-models` 仍会运行 `pnpm test:live`，因此当你需要缩小或排除该 Docker 通道中的 Gateway 网关
实时覆盖范围时，也请传入
`OPENCLAW_LIVE_GATEWAY_*`。

`test:docker:openwebui` 是更高层级的兼容性冒烟测试：它启动一个
启用了 OpenAI 兼容 HTTP 端点的 OpenClaw Gateway 网关容器，
再启动一个针对该 Gateway 网关的固定版本 Open WebUI 容器，通过
Open WebUI 登录，验证 `/api/models` 会公开 `openclaw/default`，然后通过 Open WebUI 的
`/api/chat/completions` 代理发送真实聊天请求。对于仅需完成
Open WebUI 登录和模型发现，而无需等待实时模型
完成响应的发布路径 CI 检查，可设置
`OPENWEBUI_SMOKE_MODE=models`。首次运行可能明显较慢，因为 Docker 可能需要
拉取 Open WebUI 镜像，并且 Open WebUI 可能需要完成自身的
冷启动设置。此通道需要可用的实时模型密钥，该密钥可通过
进程环境、暂存的身份验证配置文件或显式的
`OPENCLAW_PROFILE_FILE` 提供。成功运行会输出类似
`{ "ok": true, "model": "openclaw/default", ... }` 的小型 JSON 载荷。

`test:docker:mcp-channels` 特意设计为确定性测试，不需要真实的
Telegram、Discord 或 iMessage 账户。它会启动已植入数据的 Gateway 网关
容器，再启动第二个容器以生成 `openclaw mcp serve`，然后
验证路由会话发现、转录读取、附件
元数据、实时事件队列行为、出站发送路由，以及通过真实 stdio MCP 桥接传输的 Claude 风格
渠道和权限通知。通知检查会直接检查原始 stdio MCP 帧，因此该冒烟测试
验证的是桥接实际发出的内容，而不只是某个特定客户端 SDK
恰好公开的内容。

`test:docker:agent-bundle-mcp-tools` 是确定性的，不需要实时模型密钥。它会构建仓库的 Docker 镜像，在容器内启动真实的 stdio MCP 探测服务器，通过嵌入式 OpenClaw bundle MCP 运行时将该服务器具体化，执行工具，然后验证 `coding` 和 `messaging` 会保留 `bundle-mcp` 工具，而 `minimal` 和 `tools.deny: ["bundle-mcp"]` 会将其过滤掉。

`test:docker:cron-mcp-cleanup` 是确定性的，不需要实时模型密钥。它会启动带有真实 stdio MCP 探测服务器的预置 Gateway 网关，运行一个隔离的定时任务轮次和一个 `sessions_spawn` 一次性子轮次，然后验证 MCP 子进程在每次运行后都会退出。

手动 ACP 自然语言线程冒烟测试（非 CI）：

- `bun scripts/dev/discord-acp-plain-language-smoke.ts --channel <discord-channel-id> ...`
- 保留此脚本以用于回归/调试工作流。ACP 线程路由验证可能还会再次需要它，因此请勿删除。

实用环境变量：

- `OPENCLAW_CONFIG_DIR=...`（默认值：`~/.openclaw`）挂载到 `/home/node/.openclaw`
- `OPENCLAW_WORKSPACE_DIR=...`（默认值：`~/.openclaw/workspace`）挂载到 `/home/node/.openclaw/workspace`
- `OPENCLAW_PROFILE_FILE=...` 会在运行测试前挂载并作为脚本加载
- `OPENCLAW_DOCKER_PROFILE_ENV_ONLY=1` 用于验证仅使用从 `OPENCLAW_PROFILE_FILE` 加载的环境变量，并使用临时配置/工作区目录，且不挂载外部 CLI 身份验证目录
- `OPENCLAW_DOCKER_CLI_TOOLS_DIR=...`（默认值：`~/.cache/openclaw/docker-cli-tools`，除非运行已使用 CI/托管绑定目录）挂载到 `/home/node/.npm-global`，用于缓存在 Docker 内安装的 CLI
- `$HOME` 下的外部 CLI 身份验证目录/文件会以只读方式挂载到 `/host-auth...` 下，然后在测试开始前复制到 `/home/node/...`
  - 默认目录（运行未限定到特定提供商时使用）：`.factory`、`.gemini`、`.minimax`
  - 默认文件：`~/.codex/auth.json`、`~/.codex/config.toml`、`.claude.json`、`~/.claude/.credentials.json`、`~/.claude/settings.json`、`~/.claude/settings.local.json`
  - 限定提供商的运行仅挂载根据 `OPENCLAW_LIVE_PROVIDERS` / `OPENCLAW_LIVE_GATEWAY_PROVIDERS` 推断出的必要目录/文件
  - 使用 `OPENCLAW_DOCKER_AUTH_DIRS=all`、`OPENCLAW_DOCKER_AUTH_DIRS=none` 或 `OPENCLAW_DOCKER_AUTH_DIRS=.claude,.codex` 这类逗号分隔列表进行手动覆盖
- 使用 `OPENCLAW_LIVE_GATEWAY_MODELS=...` / `OPENCLAW_LIVE_MODELS=...` 缩小运行范围
- 使用 `OPENCLAW_LIVE_GATEWAY_PROVIDERS=...` / `OPENCLAW_LIVE_PROVIDERS=...` 在容器内筛选提供商
- 使用 `OPENCLAW_SKIP_DOCKER_BUILD=1` 为无需重新构建的重复运行复用现有 `openclaw:local-live` 镜像
- 使用 `OPENCLAW_LIVE_REQUIRE_PROFILE_KEYS=1` 确保凭据来自配置文件存储（而非环境变量）
- 使用 `OPENCLAW_OPENWEBUI_MODEL=...` 选择 Gateway 网关为 Open WebUI 冒烟测试公开的模型
- 使用 `OPENCLAW_OPENWEBUI_PROMPT=...` 覆盖 Open WebUI 冒烟测试使用的随机数检查提示词
- 使用 `OPENWEBUI_IMAGE=...` 覆盖固定的 Open WebUI 镜像标签

## 文档完整性检查

编辑文档后运行文档检查：`pnpm check:docs`。
如果还需要检查页内标题，请运行完整的 Mintlify 锚点验证：`pnpm docs:check-links:anchors`。

## 离线回归（CI 安全）

以下是不使用真实提供商的“真实流水线”回归：

- Gateway 网关工具调用（模拟 OpenAI，真实 Gateway 网关 + Agent loop）：`src/gateway/gateway.test.ts`（用例：“通过 Gateway 网关 Agent loop 端到端运行模拟 OpenAI 工具调用”）
- Gateway 网关向导（WS `wizard.start`/`wizard.next`，写入配置并强制执行身份验证）：`src/gateway/gateway.test.ts`（用例：“通过 ws 运行向导并写入身份验证令牌配置”）

## 智能体可靠性评估（Skills）

我们已有一些行为类似“智能体可靠性评估”的 CI 安全测试：

- 通过真实 Gateway 网关 + Agent loop 执行模拟工具调用（`src/gateway/gateway.test.ts`）。
- 验证会话连接和配置效果的端到端向导流程（`src/gateway/gateway.test.ts`）。

Skills 方面仍缺少以下内容（参见 [Skills](/zh-CN/tools/skills)）：

- **决策：**提示词中列出 Skills 时，智能体是否会选择正确的 Skill（或避开无关的 Skill）？
- **合规性：**智能体是否会在使用前读取 `SKILL.md`，并遵循所需步骤/参数？
- **工作流契约：**用于断言工具顺序、会话历史延续和沙箱边界的多轮场景。

未来的评估应优先保持确定性：

- 使用模拟提供商的场景运行器，用于断言工具调用及其顺序、Skill 文件读取和会话连接。
- 一套小型的 Skill 专项场景（使用与规避、门控、提示词注入）。
- 仅在 CI 安全套件就绪后，才添加可选的实时评估（选择启用、通过环境变量控制）。

## 契约测试（插件和渠道结构）

契约测试验证每个已注册的插件和渠道是否符合其接口契约。它们会遍历所有发现的插件，并运行一套结构和行为断言。默认的 `pnpm test` 单元测试通道会有意跳过这些共享接缝和冒烟测试文件；触及共享渠道或提供商表面时，请显式运行契约命令。

### 命令

- 所有契约：`pnpm test:contracts`
- 仅渠道契约：`pnpm test:contracts:channels`
- 仅提供商契约：`pnpm test:contracts:plugins`

### 渠道契约

位于 `src/channels/plugins/contracts/*.contract.test.ts`。当前顶层类别：

- **channel-catalog** - 内置/注册表渠道目录条目元数据
- **plugin**（由注册表支持、已分片）- 基本插件注册结构
- **surfaces-only**（由注册表支持、已分片）- 对 `actions`、`setup`、`status`、`outbound`、`messaging`、`threading`、`directory` 和 `gateway` 执行逐表面结构检查
- **session-binding**（由注册表支持）- 会话绑定行为
- **outbound-payload** - 消息负载结构和规范化
- **group-policy**（回退）- 每个渠道的默认群组策略执行
- **threading**（由注册表支持、已分片）- 线程 ID 处理
- **directory**（由注册表支持、已分片）- 目录/成员名单 API
- **registry** 和 **plugins-core.\*** - 渠道插件注册表、加载器和配置写入授权内部机制

这些套件使用的入站分派捕获和出站负载测试框架辅助程序通过 `src/plugin-sdk/channel-contract-testing.ts` 在内部公开（已从 npm 排除，不是公共 SDK 子路径）；此目录中不存在独立的 `inbound.contract.test.ts` 文件。

### 提供商契约

位于 `src/plugins/contracts/*.contract.test.ts`。当前类别包括：

- **shape** - 插件清单、API 和运行时导出结构
- **plugin-registration**（+ 并行）- 清单注册用例
- **package-manifest** - 软件包清单要求
- **loader** - 插件加载器设置/拆卸行为
- **registry** - 插件契约注册表内容和查找
- **providers** - 内置提供商之间的共享提供商行为，以及 Web 搜索提供商
- **auth-choice** - 身份验证选项元数据和设置行为
- **provider-catalog-deprecation** - 已弃用的提供商目录元数据
- **wizard.choice-resolution**、**wizard.model-picker**、**wizard.setup-options** - 提供商设置向导契约
- **embedding-provider**、**memory-embedding-provider**、**web-fetch-provider**、**tts** - 特定能力的提供商契约
- **session-actions**、**session-attachments**、**session-entry-projection** - 插件自有的会话状态契约
- **scheduled-turns** - 插件定时轮次元数据和时间戳边界
- **host-hooks**、**run-context-lifecycle**、**runtime-import-side-effects**、**runtime-seams** - 插件宿主/运行时生命周期和导入边界契约
- **extension-runtime-dependencies** - 扩展的运行时依赖放置

### 何时运行

- 更改插件 SDK 导出或子路径后
- 添加或修改渠道或提供商插件后
- 重构插件注册或设备发现后

契约测试在 CI 中运行，不需要真实 API 密钥。

## 添加回归测试（指南）

修复实时测试中发现的提供商/模型问题时：

- 尽可能添加 CI 安全的回归测试（模拟/存根提供商，或捕获准确的请求结构转换）
- 如果问题本质上只能实时测试（速率限制、身份验证策略），请保持实时测试范围精简，并通过环境变量选择启用
- 优先针对能够捕获该错误的最小层：
  - 提供商请求转换/重放错误 -> 直接模型测试
  - Gateway 网关会话/历史记录/工具流水线错误 -> Gateway 网关实时冒烟测试或 CI 安全的 Gateway 网关模拟测试
- SecretRef 遍历防护：
  - `src/secrets/exec-secret-ref-id-parity.test.ts` 根据注册表元数据（`listSecretTargetRegistryEntries()`）为每个 SecretRef 类派生一个抽样目标，然后断言包含遍历路径段的 Exec ID 会被拒绝。
  - 如果在 `src/secrets/target-registry-data.ts` 中添加新的 `includeInPlan` SecretRef 目标族，请更新该测试中的 `classifyTargetClass`。该测试会有意在遇到未分类的目标 ID 时失败，以确保新类别无法被静默跳过。

## 相关内容

- [实时测试](/zh-CN/help/testing-live)
- [更新和插件测试](/zh-CN/help/testing-updates-plugins)
- [CI](/zh-CN/ci)
