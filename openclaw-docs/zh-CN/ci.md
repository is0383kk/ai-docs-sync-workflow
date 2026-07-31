---
read_when:
    - 你需要了解 CI 任务为何运行或未运行
    - 你正在调试一项失败的 GitHub Actions 检查
    - 你正在协调一次发布验证运行或重新运行
    - 你正在更改 ClawSweeper 调度或 GitHub 活动转发
summary: CI 任务图、范围门禁、发布汇总任务和对应的本地命令
title: CI 流水线
x-i18n:
    generated_at: "2026-07-26T05:40:07Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: f9de5b527354f3cc9eed3813e961116f3834c61bd72b29c92f762c46722815df
    source_path: ci.md
    workflow: 16
---

OpenClaw CI 在推送到 `main` 时（触发器会忽略 Markdown 和 `docs/**` 路径）、每个非草稿拉取请求以及手动触发时运行。
规范的 `main` 推送采用单航班机制：`CI` 并发组允许一个完整的集成周期运行，同时 GitHub 仅保留最新的待处理推送。
新的合并会替换该待处理运行，而不会取消已经注册 Blacksmith 矩阵的工作。
拉取请求仍会取消已被取代的头提交，手动触发则使用隔离的组。`preflight` 对差异进行分类，并在仅有无关区域发生变化时关闭高开销通道。手动
`workflow_dispatch` 运行会有意绕过智能范围界定，并展开完整依赖图，用于发布候选版本和广泛验证。Android 通道通过 `include_android`（或 `release_gate` 输入）保持选择性启用。仅限发布的插件覆盖位于单独的
[`Plugin Prerelease`](#plugin-prerelease) 工作流中，并且仅从
[`Full Release Validation`](#full-release-validation) 或显式手动触发运行。

## 流水线概览

| 作业                                | 用途                                                                                                                                                                                                               | 运行时机                                   |
| ---------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------- |
| `preflight`                        | 检测变更范围并构建 CI 清单；在与 Node 相关的规范 `main` 上，在展开前刷新并维护依赖快照                                                                        | 始终在非草稿推送和拉取请求上运行             |
| `security-fast`                    | 私钥检测、通过 `zizmor` 审计已更改的工作流，以及生产锁文件审计                                                                                                                             | 始终在非草稿推送和拉取请求上运行             |
| `pnpm-store-warmup`                | 为拉取请求和手动运行预热由锁文件固定的 Actions 缓存，同时不阻塞 Linux Node 分片                                                                                                           | 在 main 之外选择 Node 或文档检查通道时 |
| `build-artifacts`                  | 构建 `dist/`、Control UI、已构建 CLI 冒烟检查、启动内存检查和嵌入式构建工件检查                                                                                                                 | 与 Node 相关的变更                          |
| `control-ui-i18n`                  | 验证生成的 Control UI 语言区域包、元数据和翻译记忆库；自动运行时仅提供建议，手动发布 CI 时阻塞                                                                               | 与 Control UI 国际化相关的变更及手动 CI |
| `checks-fast-core`                 | 快速 Linux 正确性通道：抑制基线最大行数棘轮、内置组件 + 协议、Bun 启动器和 CI 路由快速任务                                                                                  | 与 Node 相关的变更                          |
| `qa-smoke-ci-profile`              | 有界自动 QA 冒烟代表集的两个自包含均衡部分；完整分类法覆盖仍可通过显式 QA 配置文件使用                                                         | 与 Node 相关的变更                          |
| `checks-fast-contracts-plugins-*`  | 两个加权插件契约分片                                                                                                                                                                                   | 与 Node 相关的变更                          |
| `checks-fast-contracts-channels-*` | 两个加权渠道契约分片                                                                                                                                                                                  | 与 Node 相关的变更                          |
| `checks-node-*`                    | 拉取请求上针对已更改目标的 Node 测试；在 `main`、手动、发布和广泛回退运行上执行完整核心分片                                                                                                      | 与 Node 相关的变更                          |
| `check-*`                          | 分片化的主要本地门禁等效项：防护检查、shrinkwrap、内置渠道配置元数据、生产类型、lint、依赖项、测试类型                                                                                   | 与 Node 相关的变更                          |
| `check-additional-*`               | 边界检查条带（包括提示词快照漂移）、会话访问器/转录读取器/SQLite 事务边界、扩展 lint 组、软件包边界编译/金丝雀检查和运行时拓扑架构 | 与 Node 相关的变更                          |
| `checks-node-compat-node22`        | Node 22 兼容性构建和冒烟通道                                                                                                                                                                            | 用于发布的手动 CI 触发                |
| `check-docs`                       | 文档格式、lint 和失效链接检查                                                                                                                                                                         | 文档发生变更（拉取请求和手动触发）         |
| `native-i18n`                      | 在源代码拉取请求上验证原生源提取和本地化安全性；在生成的拉取请求和手动 CI 上强制要求完整的翻译/平台生成内容一致                                                               | 与原生国际化相关的变更                   |
| `skills-python`                    | 对 Python 支持的 Skills 运行 Ruff + pytest                                                                                                                                                                                | 与 Python Skills 相关的变更                  |
| `checks-windows`                   | Windows 特有的进程/路径测试，以及共享运行时导入说明符回归测试                                                                                                                                  | 与 Windows 相关的变更                       |
| `macos-node`                       | 聚焦的 macOS TypeScript 测试：launchd、Homebrew、运行时路径、打包脚本、进程组包装器                                                                                                            | 与 macOS 相关的变更                         |
| `macos-swift`                      | macOS 应用的 Swift lint 和构建，以及该应用和共享 OpenClawKit 软件包的测试                                                                                                                         | 与 macOS 相关的变更                         |
| `ios-build`                        | 生成 Xcode 项目并构建 iOS 应用模拟器版本                                                                                                                                                             | iOS 应用、共享应用套件或 Swabble 变更    |
| `android`                          | 两个变体的 Android 单元测试，以及一次调试 APK 构建                                                                                                                                                          | 与 Android 相关的变更                       |
| `openclaw/ci-gate`                 | 最终汇总：要求预检和安全检查；仅接受清单禁用的下游通道被跳过                                                                                                           | 每次非草稿 CI 运行                         |
| `test-performance-agent`           | 单独的工作流：可信活动后的每日 Codex 慢速测试优化                                                                                                                                          | Main CI 成功或手动触发             |
| `openclaw-performance`             | 单独的工作流：按日/按需生成 Kova 运行时性能报告，包含模拟提供商、深度分析和 GPT 5.6 实时通道                                                                                          | 定时和手动触发                  |

独立的 Periphery 工作流强制要求 iOS 和 macOS 应用不存在死代码发现项。共享 OpenClawKit 工作流会并行扫描两个使用方，并且仅当 Periphery 从两个构建中发出相同的 Swift USR 时才报告声明。其生成的 `OpenClawProtocol/GatewayModels.swift` 架构契约会作为生成器所有的代码保留，而不会被视为应用本地的死代码。

## 快速失败顺序

1. `preflight` 决定究竟存在哪些通道。`docs-scope` 和 `changed-scope` 逻辑是此作业内的步骤，而不是独立作业。规范的 `main` 会立即启动，但其并发组只允许一个完整运行进入，并将后续推送合并为一个最新的待处理运行。与 Node 相关的 main 推送还会在此处串行执行唯一的依赖磁盘写入器及其大小维护，之后下游作业才能挂载该键；Blacksmith 可能只会向后续工作流运行提供新提交，因此同次运行的使用方会保留经过标记检查的本地回退。
2. `security-fast`、`check-*`、`check-additional-*`、`check-docs` 和 `skills-python` 会快速失败，无需等待更重的工件和平台矩阵作业。
3. `build-artifacts` 和语言区域检查会与快速 Linux 通道重叠运行。Control UI 和原生应用源代码拉取请求会排除生成的语言区域快照/资源；其串行刷新工作流会在后台修复并自动合并隔离的生成拉取请求。源代码 CI 仍会阻止过期的源清单和不安全的本地化调用。生成的拉取请求、手动 CI 和发布准备会强制要求完整的翻译/平台生成内容一致。规范的 `release/YYYY.M.PATCH` 分支可以将发布准备语言区域修复与其他生成的发布输出一并包含。
4. 之后会展开开销更大的平台和运行时通道：`checks-fast-core`、`checks-fast-contracts-plugins-*`、`checks-fast-contracts-channels-*`、`checks-node-*`、`checks-windows`、`macos-node`、`macos-swift`、`ios-build` 和 `android`。
5. `openclaw/ci-gate` 会等待每个已选择的通道。预检和安全检查必须成功；只有在清单未选择下游作业时，才允许跳过它们。已选择的通道失败或被取消会导致汇总失败。

合并协调器可以为同一拉取请求头提交重复使用在 24 小时内通过身份验证且成功的 `openclaw/ci-gate`。
这样可避免在发生无关的 `main` 变更后重写贡献者分支。可重复使用的结果不会取代针对当前 `main`、由 App 所有的独立严格测试合并检查。
后续待处理或失败的重新运行不会在有效期窗口内抹除该未变更头提交此前的成功结果。

默认分支规则集要求由 GitHub Actions 所有的 `openclaw/ci-gate` 棹查。仓库维护者和管理员拥有经审计的紧急绕过权限，仅用于已签名的直接快进落地；组织规则集仍会阻止删除和非快进更新。正常的拉取请求合并应继续使用该门禁，而不是绕过失败的 CI。单独的、由严格 App 所有的测试合并检查仍会将头提交绑定到当前 `main`。

当较新的头提交落地时，GitHub 可能会将已被取代的拉取请求任务标记为 `cancelled`。除非同一 PR 的最新运行也失败，否则应将其视为 CI 噪声。规范的 `main` 运行在准入后不会取消；当合并流量到来时，GitHub 只会用最新提交替换较旧的待处理运行。矩阵任务使用 `fail-fast: false`，而 `build-artifacts` 会直接报告嵌入式渠道、核心支持边界和 Gateway 网关监视失败，而不是将小型验证器任务加入队列。自动 CI 并发键带有版本号（`CI-v7-*`），因此 GitHub 端旧队列组中的僵尸任务无法无限期阻塞较新的 main 运行。手动完整套件运行使用 `CI-manual-v1-*`，并且不会取消正在进行的运行。插件列表启动内存防护在自托管 Blacksmith Linux 上维持 350 MiB 上限，在 GitHub 托管的 Linux 上则允许 425 MiB；对于同一个已构建 CLI，后者的 RSS 基线更高。

使用 `pnpm ci:timings`、`pnpm ci:timings:recent` 或 `node scripts/ci-run-timings.mjs <run-id>` 汇总 GitHub Actions 的总耗时、排队时间、最慢任务、失败情况以及 `pnpm-store-warmup` 扇出屏障。工作流内的 `ci-timings-summary` 任务存在于 `ci.yml` 中，但当前已禁用（`if: false`）；请改为在本地运行计时辅助工具。若要查看构建计时，请检查 `build-artifacts` 任务的 `Build dist` 步骤：`pnpm build:ci-artifacts` 会输出 `[build-all] phase timings:`，并包含 `ui:build`；该任务还会上传 `startup-memory` 工件。

## PR 上下文和证据

外部贡献者的 PR 会从
`.github/workflows/real-behavior-proof.yml` 运行 PR 上下文和证据门禁。该工作流检出
受信任的工作流修订版（`github.workflow_sha`），并且仅评估 PR 正文；
它不会执行贡献者分支中的代码。

该门禁适用于既不是仓库所有者、成员、协作者，也不是机器人的 PR 作者。当 PR 正文包含作者撰写的
`What Problem This Solves` 和 `Evidence` 章节时，门禁通过。证据可以是聚焦测试、CI 结果、屏幕截图、录制内容、终端输出、实时观察、已脱敏日志或工件链接。正文用于说明意图并提供有用的验证信息；
审查者会检查代码、测试和 CI，以评估正确性。

当检查失败时，请更新 PR 正文，而不是再推送一个代码提交。

## 范围和路由

范围逻辑位于 `scripts/ci-changed-scope.mjs`，其单元测试位于 `src/scripts/ci-changed-scope.test.ts`。手动触发会跳过变更范围检测，并使预检清单表现得像所有限定范围都发生了变更。

独立的 iOS 和 macOS Periphery 工作流会强制执行零发现项的死代码策略。每个工作流仅在非草稿拉取请求触及其原生扫描范围时运行，或在手动触发时运行。

- **CI 工作流编辑**会验证 Node CI 图、工作流 lint 和 Windows 通道（由 `ci.yml` 执行），但其本身不会强制执行 iOS、Android 或 macOS 原生构建；这些平台通道仍仅限于平台源代码变更。
- **工作流完整性检查**会对所有工作流 YAML 文件运行 `actionlint` 和 `zizmor`，并运行复合操作插值防护和冲突标记防护。PR 范围内的 `security-fast` 任务还会对已变更的工作流文件运行 `zizmor`，使工作流安全发现项能在主 CI 图中尽早触发失败。
- **推送到 `main` 时的文档**由独立的 `Docs` 工作流检查，该工作流使用与 CI 相同的 ClawHub 文档镜像，因此同时包含代码和文档的推送不会额外将 CI 的 `check-docs` 分片加入队列。当文档发生变更时，拉取请求和手动 CI 仍会从 CI 运行 `check-docs`。
- **TUI PTY**会在 TUI 变更时于 `checks-node-core-runtime-tui-pty` Linux Node 分片中运行。该分片使用 `OPENCLAW_TUI_PTY_INCLUDE_LOCAL=1` 运行 `test/vitest/vitest.tui-pty.config.ts`，因此既覆盖确定性的 `TuiBackend` 固定数据通道，也覆盖仅模拟外部模型端点、速度较慢的 `tui --local` 冒烟测试。
- **仅修改 CI 路由、快速任务直接运行的少量核心测试固定数据，以及范围有限的插件契约辅助工具编辑**会使用仅含 Node 的快速清单路径：`preflight`、`security-fast`，以及变更所触及的快速通道——单个 `checks-fast-core` CI 路由任务、两个插件契约分片，或两者兼有。该路径会跳过构建工件、Node 22 兼容性、渠道契约、完整核心分片、内置插件分片和其他防护矩阵。
- **Windows Node 检查**仅限于 Windows 特定的进程/路径包装器、npm/pnpm/UI 运行器辅助工具、包管理器配置，以及执行该通道的 CI 工作流表面；不相关的源代码、插件、安装冒烟测试和纯测试变更仍留在 Linux Node 通道上。

最慢的 Node 测试系列会进行拆分或均衡，以便每个任务保持较小规模，同时避免过度预留运行器：

- 插件契约和渠道契约各自作为两个由 Blacksmith 支持的加权分片运行，并配有标准 GitHub 运行器回退。
- 核心单元快速/支持通道独立运行；核心运行时基础设施拆分为进程、共享、钩子、密钥以及三个定时任务领域分片。
- 自动回复以均衡工作进程运行，回复子树拆分为 agent-runner、命令、分派、会话和状态路由分片。
- 智能体式 Gateway 网关/服务器（控制平面）配置拆分到聊天、身份验证、模型、HTTP/插件、运行时和启动通道中，而不是等待构建产物。
- 常规 CI 仅将隔离的基础设施 include-pattern 分片打包为确定性的捆绑包，每个最多包含 64 个测试文件，从而减少 Node 矩阵，同时不合并非隔离的命令/定时任务、带状态的 agents-core 或 Gateway 网关/服务器套件。繁重的固定套件继续使用 8 vCPU，而捆绑通道和较低权重通道使用 4 vCPU。
- 规范仓库上的 PR 会针对合成合并树差异复用变更测试解析器。精确变更运行一个定向 Node 作业；每个选中的测试文件都使用自己的进程，因此带状态套件的隔离保持不变。规划器将同级测试与导入图依赖项组合起来；对于工作区包、包/锁文件、共享测试框架、拆分配置、重命名或删除的变更、公共扩展契约变更、具有特殊分片设置的测试、部分解析或空目标、过大的路径或目标计划以及规划器错误，则回退到现有的 14 作业紧凑完整套件计划。定向计划始终保留完整的构建产物边界门禁，因为其仓库扫描器无法从导入关系推导得出。`main` 推送运行相同的完整紧凑套件：待处理的中间推送事件可能会被合并，因此最新保留下来的运行必须验证完整的集成树，而不能仅验证其最终单次推送的差异。手动分派和发布门禁保留完整的具名逐分片矩阵。
- 完整 Node 矩阵会优先接纳持续较慢的串行工具、自动回复命令分片和广泛的 core-fast 缓存写入器。这样既能维持 28 作业上限，又能防止关键路径工作和下一次运行的转换种子滑入后续批次。
- 广泛的浏览器、QA、媒体和杂项插件测试使用各自专用的 Vitest 配置，而不是共享的插件兜底配置。include-pattern 分片使用 CI 分片名称记录计时条目，因此 `.artifacts/vitest-shard-timings.json` 可以区分完整配置和经过筛选的分片。
- Linux Node 分片作业通过上游 Actions 缓存 API 持久化 Vitest 的实验性文件系统模块缓存，Blacksmith 会在其运行器上透明地加速该缓存。每个 CI 分片都仅执行恢复，并将受保护种子解压到各自的运行器本地根目录中；随后，分片包装器为并发 Vitest 进程分配独立的实时子目录。只有不会被取消的每日预热器或显式分派的预热器才会保存新的不可变归档，因此 PR 无法发布转换结果或创建每 PR 缓存系列。转换输入指纹会清除不兼容的锁文件、包、tsconfig 和 Vitest 配置代际。受保护写入器会扫描其恢复的缓存，并在缓存超过 2 GiB 后将其裁剪至 75%。Vitest 会对模块 ID、源内容、环境和解析后的转换配置进行哈希，因此普通的部分源代码变更可使未变更的条目保持预热状态，而变更的模块会安全地缓存未命中。粗粒度恢复前缀可跨工作流运行衔接；常规 Actions 缓存的 LRU 和非活动淘汰机制会限制旧的不可变归档。
- 受信任的 Linux Node 作业还会从每个受支持 Node 版本线的一块受保护依赖磁盘绑定 pnpm 存储和 `node_modules`。包清单、安装设置、运行器平台和确切的 Node 补丁版本不纳入磁盘键；由确切的运行时和安装输入指纹决定作业是复用依赖树，还是重新安装并刷新同一磁盘。清单会在哈希前进行规范化。经审计的直接根钩子仅保留 pnpm 的安装生命周期脚本，因此格式化以及普通测试/构建脚本的编辑可继续使用预热的依赖树；未经审计的生命周期钩子漂移会采用故障关闭方式，直到其源输入加入指纹契约。依赖项、包管理器、钩子源和锁文件的变更始终使快照失效。匹配的指纹是必要条件，但并非充分条件：设置过程还会检查导入方归档和清单校验和，然后对注册表支持、由 postinstall 保留在锁文件中的依赖项，按照 Node 从其导入方解析出的包清单进行验证。导入方内容缺失或过期时，会回退到全新安装，而不是提供根提升内容。只读快照不可用的 PR 会解除工作区绑定，并安装到运行器本地存储中，以避免缓慢写入其无法发布的克隆。粘性冷安装会禁用 pnpm 内部的获取重试，并从逐步预热的存储最多执行三次有界完整安装尝试；超时仍视为失败。在完成内容验证的恢复或 frozen-lockfile 安装后，设置过程会禁用 pnpm 冗余的运行前依赖检查：仓库会有意裁剪插件本地的 `node_modules`，否则 pnpm 会将其视为过期，并在分片扇出期间通过不安全的并发隐式安装进行修复。规范 main 预检是唯一的写入器，并在每次刷新时测量存储大小，仅当已淘汰的包版本使其超过 8 GiB 后才运行 `pnpm store prune`。即使写入器作业已经完成，Blacksmith 快照发布仍是异步的，因此使用新键或指纹后的第一次运行可能仍是冷运行；后续通过内容验证的精确标记恢复才是部署生效的证明。必需的 CI 作业和 PR 使用一次性克隆，因此依赖项变更不会创建新磁盘、竞争快照或产生可能取消构建的缓存锁。
- Node 分片和构建产物作业还会通过不可变 Actions 缓存恢复 Node 的可移植磁盘编译缓存。独立的 `test` 和 `build` 命名空间可防止各自的写入器替换对方的归档：计划测试预热器拥有受保护的测试种子，而 `build-artifacts` 每个 UTC 日最多可以从受信任的 `main` 推送发布一个受保护的构建归档。PR 和普通测试作业只读取受保护快照，因此功能分支字节码永远不会进入共享种子，并且 PR 流量不会创建缓存归档。这会在不同检出路径之间复用由 Node 加载的编排代码、构建工具和外部依赖项的 V8 字节码，包括仅有部分源代码图发生变更的情况。Vitest 子进程会禁用继承的编译缓存，因为动态配置中可能启用覆盖率，而当脚本从字节码反序列化时，V8 覆盖率可能丢失源位置精度。
- 构建产物作业还会持久化带内容指纹的 `build-all` 步骤输出。CI 自行构建的插件 SDK 声明会对仓库所有的完整 TypeScript/JSON 源代码图进行哈希，排除已安装目录和生成目录，并在 `tsdown` 清除 `dist` 后恢复扁平声明和包桥接。该图之外的文档、工作流、插件及其他变更可以复用声明快照；源代码变更会在导出门禁运行前重新构建声明。
- 完整声明构建将 `tsdown` 拆分为 AI、工作区包和统一组。每个组仅缓存声明，然后仍会在恢复这些声明之前重新构建运行时 JavaScript。因此，核心或插件变更只会使大型统一图失效，而工作区包变更会保守地使每个依赖声明组失效。公共完整构建通常使用不可变 Actions 缓存；粗粒度恢复键为部分变更提供种子，各组内容指纹会拒绝过期数据，而 GitHub 的缓存配额会淘汰旧代际。每周 Node 22 通道则会在 `main` 成功运行后发布一个保留 14 天的工件，并且仅恢复其不可变生产者身份在 `main` 上解析为该工作流的工件，从而避免配额频繁变动，同时不允许 PR 代码写入共享缓存。Private-QA 声明绝不会持久化到 Actions 缓存中，因为缓存命名空间不是保密边界。
- `check-additional-*` 将补充边界守卫列表（`scripts/run-additional-boundary-checks.mjs`）条带化为一个提示词密集型分片（`check-additional-boundaries-a`，其中包括 Codex 提示词快照漂移检查）以及一个用于其余条带的组合分片（`check-additional-boundaries-bcd`），每个分片都会并发运行独立守卫并打印各项检查的计时。包边界编译/金丝雀工作保持在一起，而运行时拓扑架构则与嵌入 `build-artifacts` 的 Gateway 网关 watch 覆盖率分开运行。
- 在 32-vCPU 自托管构建运行器上，Gateway 网关 watch、渠道测试和核心支持边界分片会在 `dist/` 和 `dist-runtime/` 已构建完成后，一起在 `build-artifacts` 内启动。GitHub 托管的回退运行会保持 Gateway 网关 watch 串行执行，以免低核心数资源争用耗尽其就绪期限。

获准运行后，规范 Linux CI 最多允许 28 个 Node 测试作业以及
12 个较小的快速/检查通道并发运行；Windows 和 Android 保持为两个，因为
这些运行器池规模较小。紧凑的全配置批次使用
120 分钟的批次超时，而 include-pattern 组共享相同的有界
作业预算。

Android CI 会同时运行 `testPlayDebugUnitTest` 和 `testThirdPartyDebugUnitTest`，然后构建 Play 调试 APK。第三方变体没有独立的源集或清单；其单元测试通道仍会使用 SMS/通话记录 BuildConfig 标志编译该变体，同时避免在每次与 Android 相关的推送中执行重复的调试 APK 打包作业。每个当前 Gradle 任务都有一块受保护的粘性磁盘；PR 作业使用一次性克隆，而受保护运行会就地刷新内容寻址的 Gradle 条目。

Blacksmith 粘性磁盘键会刻意限制在受支持的运行时或任务维度内，绝不会包含 PR 编号、提交、运行、分支或依赖项哈希。运行时转换和编译缓存使用 Actions 缓存而非粘性磁盘，因为不可变归档可提供可验证的恢复/保存结果，并避免可变快照提升失败。完成粘性键版本迁移后，仅将确切的废弃键、架构和区域身份添加到 `.github/retired-sticky-disks.json`，使用相同维度和确认信息从 `main` 分派 `Sticky Disk Cleanup`，验证删除结果，然后移除这些条目。该工作流会将 ARM 身份路由到 ARM 运行器，拒绝运行器区域不匹配，使用 Blacksmith 的精确键删除操作，并且绝不会删除 Docker 构建器缓存或通配符前缀。Actions 缓存归档使用常规 LRU 和非活动淘汰机制。

`check-dependencies` 分片运行生产环境 Knip 依赖项、未使用文件和未使用导出检查。当 PR 添加新的未经审查的未使用文件，或留下过期的允许列表条目时，未使用文件守卫会失败，同时保留 Knip 无法静态解析的有意动态插件、生成、构建、实时测试和包桥接表面。未使用导出守卫会排除测试支持文件，并对每个未使用的生产环境导出报错；有意的动态使用方必须在 `config/knip.config.ts` 中建模。历史目标在提供导出守卫时运行该守卫，否则继续使用其旧版死代码回退检查。

## ClawSweeper 活动转发

`.github/workflows/clawsweeper-dispatch.yml` 是从 OpenClaw 仓库活动到 ClawSweeper 的目标端桥接。它不会检出或执行不受信任的拉取请求代码。该工作流使用 `CLAWSWEEPER_APP_PRIVATE_KEY` 创建 GitHub App 令牌，然后将紧凑的 `repository_dispatch` 负载分派到 `openclaw/clawsweeper`。

该工作流有四条通道：

- `clawsweeper_item`，用于精确的议题和拉取请求审查请求；
- `clawsweeper_comment`，用于议题评论中的显式 ClawSweeper 命令；
- `clawsweeper_commit_review`，用于 `main` 推送上的提交级审查请求；
- `github_activity`，用于 ClawSweeper 智能体可检查的一般 GitHub 活动。

`github_activity` 通道仅转发规范化元数据：事件类型、操作、执行者、仓库、条目编号、URL、标题、状态，以及评论或审查存在时的简短摘录。它有意避免转发完整的 webhook 正文。`openclaw/clawsweeper` 中的接收工作流是 `.github/workflows/github-activity.yml`，它会将规范化事件发布到供 ClawSweeper 智能体使用的 OpenClaw Gateway 网关钩子。

一般活动用于观察，默认不投递。ClawSweeper 智能体会在提示词中收到 Discord 目标，并且仅当事件出乎意料、可采取行动、存在风险或对运维有用时，才应发布到 `#clawsweeper`。常规打开、编辑、机器人活动、重复的 webhook 噪声和正常审查流量应产生 `NO_REPLY`。

在整个路径中，将 GitHub 标题、评论、正文、审查文本、分支名称和提交消息视为不受信任的数据。它们是摘要和分诊的输入，而不是工作流或智能体运行时的指令。

## 手动分派

手动 CI 分派运行与正常 CI 相同的作业图，但会强制启用所有非 Android 范围的通道：Linux Node 分片、内置插件分片、插件和渠道契约分片、Node 22 兼容性、`check-*`、`check-additional-*`、构建工件冒烟检查、文档检查、Python Skills、Windows、macOS、iOS 构建以及 Control UI/原生应用 i18n。自动源代码 PR 会验证原生提取清单以及 Android/Apple 本地化安全性，而不要求在同一 PR 中包含翻译或平台生成的输出。串行化的 Native App Locale Refresh 工作流会在一个隔离 PR 中重新构建这些工件，并在所需检查通过后启用精确 HEAD 自动合并。生成工件 PR、手动 CI、Full Release Validation 和发布准备仍要求完整的原生一致性，并将其作为阻塞条件。Control UI 区域设置一致性在自动 PR 和 `main` 运行中仍为建议性检查，在手动/发布 CI 中则为阻塞检查。独立的手动 CI 分派仅在设置 `include_android=true` 时运行 Android（`release_gate` 输入也会强制启用 Android）；完整发布总流程通过传递 `include_android=true` 来启用 Android。插件预发布静态检查、仅发布使用的 `agentic-plugins` 分片、完整扩展批量扫描和插件预发布 Docker 通道不包含在 CI 中。仅当 `Full Release Validation` 在启用发布验证门禁的情况下分派单独的 `Plugin Prerelease` 工作流时，才会运行 Docker 预发布套件。

PR 最大行数检查从已检出的合成合并树派生基线，并对照事件 HEAD 验证其 HEAD 父提交。手动运行使用唯一的并发组，因此候选发布版本的完整套件不会被同一引用上的其他推送或 PR 运行取消。可选的 `target_ref` 输入允许受信任的调用方针对分支、标签或完整提交 SHA 运行该作业图，同时使用所选分派引用中的工作流文件；最大行数基线会与该目标相对于此次运行所解析的默认分支 HEAD 的合并基点进行比较。`release_gate` 输入是在 PR CI 因容量而停滞时供维护者使用的精确 SHA 后备方案：它要求 `target_ref` 是与所分派分支 HEAD 匹配的完整提交 SHA，并要求 `pull_request_number` 标识其合并树接受验证的开放 PR。

```bash
gh workflow run ci.yml --ref release/YYYY.M.PATCH
gh workflow run ci.yml --ref main -f target_ref=<branch-or-sha> -f include_android=true
gh workflow run full-release-validation.yml --ref main -f ref=<branch-or-sha>
```

Gateway 网关扩展稳定版从 `extended-stable/YYYY.M.33` 运行 npm 预检、Full Release Validation 和插件
npm 发布；核心发布使用这三个
运行 ID 及验证尝试次数。`release-ci/*` 证据无效，因为
发布会将每次运行绑定到规范分支和发布 SHA。该标签
发布 Gateway 网关镜像，并且仅发布 `extended-stable*` 别名；此路径会跳过
常规编排器及其 ClawHub、原生应用、GitHub Release、网站
和私有 dist-tag 表面。有关命令和恢复操作，请参阅[每月 Gateway 网关扩展稳定版
发布](/zh-CN/reference/RELEASING#monthly-gateway-extended-stable-publication)。

## Runner

| Runner                          | 作业                                                                                                                                                                                                                                                                              |
| ------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `ubuntu-24.04`                  | `security-fast`、手动 CI 分派和非规范仓库后备方案、QA Smoke 聚合作业、CodeQL 安全和质量扫描、工作流完整性检查、标签器、自动响应、独立文档工作流以及整个 Install Smoke 工作流                                |
| `blacksmith-4vcpu-ubuntu-2404`  | `preflight`、`pnpm-store-warmup`、`native-i18n`、除 QA Smoke CI 外的 `checks-fast-core`、插件/渠道契约分片、大多数内置/低负载 Linux Node 分片、除 `check-lint` 外的 `check-*` 通道、选定的 `check-additional-*` 分片、`check-docs` 和 `skills-python` |
| `blacksmith-8vcpu-ubuntu-2404`  | 保留的高负载 Linux Node 套件、边界/扩展密集型 `check-additional-*` 分片以及 `android`                                                                                                                                                                             |
| `blacksmith-16vcpu-ubuntu-2404` | 自动 QA Smoke CI 分片、CI 和 Testbox 中的 `build-artifacts`，以及 `check-lint`（对 CPU 足够敏感，使用 8 vCPU 的成本高于节省的成本）                                                                                                                                  |
| `blacksmith-8vcpu-windows-2025` | `checks-windows`                                                                                                                                                                                                                                                                  |
| `blacksmith-6vcpu-macos-15`     | `openclaw/openclaw` 上的 `macos-node`；分叉仓库回退到 `macos-15`                                                                                                                                                                                                                |
| `blacksmith-12vcpu-macos-26`    | `openclaw/openclaw` 上的 `macos-swift` 和 `ios-build`；分叉仓库回退到 `macos-26`                                                                                                                                                                                               |

## Runner 注册预算

OpenClaw 当前的 GitHub Runner 注册桶在 `ghx api rate_limit` 中报告每 5 分钟 10,000 次自托管
Runner 注册。每次调优前重新检查
`actions_runner_registration`，因为 GitHub 可能会更改
此桶。该限制由
`openclaw` 组织中的所有 Blacksmith Runner 注册共享，因此添加另一个 Blacksmith 安装不会增加
新的桶。

将 Blacksmith 标签视为突发控制的稀缺资源。仅执行
路由、通知、摘要、选择分片或短时 CodeQL 扫描的作业应
保留在 GitHub 托管的 Runner 上，除非它们具有经测量证实的 Blacksmith 特定
需求。任何新的 Blacksmith 矩阵、更大的 `max-parallel` 或高频
工作流都必须说明其最坏情况下的注册数，并将组织级
目标保持在实时桶的约 60% 以下。对于当前 10,000 次注册的
桶，这意味着 6,000 次注册的运行目标，为
并发仓库、重试和突发重叠留出余量。

变更目标 PR 计划将常见 Node 测试突发从 14 次 Blacksmith 注册减少到一次。高风险范围 PR 保留 14 次注册的紧凑后备方案，因此最坏情况不会增加。

规范仓库 CI 保持 Blacksmith 作为正常推送和拉取请求运行的默认 Runner 路径。`workflow_dispatch` 和非规范仓库运行使用 GitHub 托管的 Runner，但正常规范运行目前不会探测 Blacksmith 队列健康状况，也不会在 Blacksmith 不可用时自动回退到 GitHub 托管标签。

## 表面棘轮

两个只减不增的预算用于约束配置表面。只要出现增长，两者都会使 CI 失败，
直到在同一 PR 中有意识地更新预算文件；当清理使实际数量降低时，两者都要求
同步下调棘轮值。

- `config/env-var-count-budget.txt` 限制 `src/`、`packages/` 和 `extensions/` 下生产源代码中不同 `OPENCLAW_*`
  名称的数量
  （不包括测试和 QA Lab）。由 `node scripts/check-env-var-count.mjs` 检查。
  移除环境变量时：在同一 PR 中降低该数字。添加环境变量属于
  配置表面决策——请在 PR 正文中说明理由。
- `docs/.generated/config-baseline.counts.json` 限制每种类型
  （核心/渠道/插件）的 `openclaw.json` 架构条目数量。由
  `pnpm config:docs:check` 检查；任何
  架构变更后，使用 `pnpm config:docs:gen` 重新生成。

## 本地等效命令

```bash
pnpm changed:lanes                            # 检查针对 origin/main...HEAD 的本地变更通道分类器
pnpm check:changed                            # 智能本地检查门禁：按边界通道检查变更涉及的格式、类型检查、lint 和防护项
pnpm check                                    # 快速本地门禁：生产代码 tsgo + 分片 lint + 并行快速防护项
pnpm check:test-types
pnpm check:timed                              # 相同门禁，并提供各阶段耗时
pnpm build:strict-smoke
pnpm check:architecture
pnpm test:gateway:watch-regression
OPENCLAW_TUI_PTY_INCLUDE_LOCAL=1 node scripts/run-vitest.mjs run --config test/vitest/vitest.tui-pty.config.ts
pnpm test                                     # vitest 测试
pnpm test:changed                             # 低成本智能选择发生变更的 Vitest 目标
pnpm test:ui                                  # Control UI 单元测试/浏览器测试套件
pnpm ui:i18n:check                            # 生成的 Control UI 语言区域一致性检查（发布门禁）
pnpm native:i18n:baseline                     # 更新由源代码维护的原生提取清单
pnpm native:i18n:verify                       # 源清单 + Android/Apple 本地化安全检查
pnpm native:i18n:check                        # 严格检查翻译与平台生成内容的一致性（发布门禁）
pnpm test:channels
pnpm test:contracts:channels
pnpm check:docs                               # 文档格式 + lint + 失效链接
pnpm build                                    # 当 CI 工件/冒烟检查很重要时构建 dist
pnpm ios:build                                # 生成并构建 iOS 应用项目
pnpm ci:timings                               # 汇总最新的 origin/main 推送 CI 运行
pnpm ci:timings:recent                        # 比较近期成功的 main CI 运行
node scripts/ci-run-timings.mjs <run-id>      # 汇总实际耗时、排队时间和最慢的任务
node scripts/ci-run-timings.mjs --latest-main # 忽略 issue/comment 噪声并选择 origin/main 推送 CI
node scripts/ci-run-timings.mjs --recent 10   # 比较近期成功的 main CI 运行
pnpm test:perf:groups --full-suite --allow-failures --output .artifacts/test-perf/baseline-before.json
pnpm test:perf:groups:compare .artifacts/test-perf/baseline-before.json .artifacts/test-perf/after-agent.json
pnpm test:startup:memory
pnpm test:extensions:memory -- --json .artifacts/openclaw-performance/source/mock-provider/extension-memory.json
pnpm perf:kova:summary --report .artifacts/kova/reports/mock-provider/report.json --output .artifacts/kova/summary.md
```

## OpenClaw 性能

`OpenClaw Performance` 是产品/运行时性能工作流。它每天在 `main` 上运行，也可以手动调度：

```bash
gh workflow run openclaw-performance.yml --ref main -f profile=diagnostic -f repeat=3
gh workflow run openclaw-performance.yml --ref main -f profile=smoke -f repeat=1 -f deep_profile=true -f live_openai_candidate=true
gh workflow run openclaw-performance.yml --ref main -f target_ref=v2026.5.2 -f profile=diagnostic -f repeat=3
```

手动调度通常会对工作流引用进行基准测试。设置 `target_ref` 可使用当前工作流实现对发布标签或其他分支进行基准测试。已发布的报告路径和 latest 指针按被测引用区分，每个 `index.md` 都会记录被测引用/SHA、工作流引用/SHA、Kova 引用、配置文件、通道身份验证模式、模型、重复次数和场景筛选条件。

该工作流从固定版本安装 OCM，并从 `openclaw/Kova` 按固定的 `kova_ref` 输入安装 Kova，然后运行三个通道：

- `mock-provider`：针对本地构建运行时运行 Kova 诊断场景，并使用确定性的模拟 OpenAI 兼容身份验证。
- `mock-deep-profile`：针对启动、Gateway 网关和智能体轮次热点进行 CPU/堆/跟踪性能分析。按计划运行，或在使用 `deep_profile=true` 调度时运行。
- `live-openai-candidate`：执行一次真实的 OpenAI `openai/gpt-5.6-luna` 智能体轮次；当 `OPENAI_API_KEY` 不可用时跳过。按计划运行，或在使用 `live_openai_candidate=true` 调度时运行。

模拟提供商通道还会在 Kova 测试通过后运行 OpenClaw 原生源代码探针：测量默认、跳过渠道、内部钩子和五十插件启动场景下的 Gateway 网关启动耗时与内存；测试内置插件导入 RSS、重复的模拟 OpenAI `channel-chat-baseline` 问候循环、针对已启动 Gateway 网关执行的 CLI 启动命令，以及 SQLite 状态冒烟性能探针。当被测引用存在此前发布的模拟提供商源报告时，源摘要会将当前 RSS 和堆值与该基线进行比较，并将 RSS 的大幅增长标记为 `watch`。源探针 Markdown 摘要位于报告包中的 `source/index.md`，原始 JSON 位于其旁边。

每个通道都会上传完整的 GitHub 工件，包括 CPU、堆、跟踪和压缩诊断包。单独的发布任务会下载并验证这些工件，然后生成一个短期有效的 ClawSweeper GitHub App 令牌，其权限范围仅限 `openclaw/clawgrit-reports` 内容，并且只将其传递给 Git 推送步骤。该任务会在 `openclaw-performance/<tested-ref>/<run-id>-<attempt>/<lane>/` 下提交 `report.json`、`report.md`、`index.md`、源探针工件以及包元数据/校验和；完整诊断归档保留在所链接的 Actions 工件中。发布任务会在尝试推送前拒绝任何超过 50 MB 的报告文件。当前被测引用指针是 `openclaw-performance/<tested-ref>/latest-<lane>.json`。如果 App 令牌创建或报告发布失败，计划运行和 `profile=release` 调度将失败。对于非发布版本的手动调度，发布仅作为建议项；身份验证或发布失败时仍会保留 GitHub 工件。此前的源基线会从公共报告仓库匿名获取，因此成功获取基线并不能证明发布者身份验证成功。

## 全面发布验证

`Full Release Validation` 是用于“发布前运行所有内容”的手动总控工作流。它接受分支、标签或完整提交 SHA，使用该目标调度手动 `CI` 工作流（包括 Android），调度 `Plugin Prerelease` 以执行仅限发布的插件/软件包/静态/Docker 验证，针对目标 SHA 调度 `OpenClaw Performance`，并调度 `OpenClaw Release Checks` 以执行安装冒烟测试、软件包验收、跨操作系统软件包检查、QA Lab 一致性检查、Matrix、Telegram，以及设有门禁的 Discord、WhatsApp 和 Slack 通道（可通过 `run_maturity_scorecard` 选择启用建议性的成熟度评分卡渲染）。稳定版和完整配置始终包含详尽的实时/E2E 和 Docker 发布路径浸泡测试覆盖；beta 配置可通过 `run_release_soak=true` 选择启用。标准软件包 Telegram E2E 在软件包验收中运行，因此完整候选版本不会启动重复的实时轮询器。发布后，传入 `release_package_spec` 可在发布检查、软件包验收、Docker、跨操作系统和 Telegram 中复用已发布的 npm 软件包，而无需重新构建。仅在针对已发布软件包进行重点 Telegram 重跑时使用 `npm_telegram_package_spec`。Codex 插件实时软件包通道默认使用同一选定状态：已发布的 `release_package_spec=openclaw@<tag>` 会派生出 `codex_plugin_spec=npm:@openclaw/codex@<tag>`，而 SHA/工件运行会从选定引用打包 `extensions/codex`。对于 `npm:`、`npm-pack:` 或 `git:` 规范等自定义插件源，请显式设置 `codex_plugin_spec`。其实时智能体验证会发送可见进度，继续执行随机化的工作区读取和精确工件写入，然后发送完成消息。

有关阶段矩阵、准确的工作流任务名称、配置差异、工件和
重点重跑入口，请参阅[全面发布验证](/zh-CN/reference/full-release-validation)。

`OpenClaw Release Publish` 是会执行变更的手动发布工作流。在发布标签
已存在且 OpenClaw npm 预检成功后，从可信的 `main` 调度
常规 beta 和稳定版发布（预检会运行
`pnpm plugins:sync:check` 等检查）。标签仍会选择确切的
发布提交，包括 `release/YYYY.M.PATCH` 上的提交；Tideclaw alpha
发布继续使用相匹配的 alpha 分支。该工作流要求已保存
`preflight_run_id`，并且要求
`full_release_validation_run_id` 及其确切的
`full_release_validation_run_attempt` 成功；它会为所有
可发布插件软件包调度 `Plugin NPM Release`，为同一
发布 SHA 调度 `Plugin ClawHub Release`，之后才会调度 `OpenClaw NPM Release`。稳定版发布还
要求确切的 `windows_node_tag`；该工作流会验证 Windows 源
发布，并在任何发布子工作流之前，将其 x64/ARM64 安装程序与候选版本已批准的
`windows_node_installer_digests` 输入进行比较，然后提升
并验证相同的固定安装程序摘要以及确切的配套资产
和校验和契约，最后发布 GitHub 发布草稿。
仅修复插件的重点操作使用 `plugin_publish_scope=selected`，并提供非空的
软件包列表。仅插件的 `all-publishable` 运行要求与核心发布相同的不可变 npm
预检和全面发布验证证据。

```bash
gh workflow run openclaw-release-publish.yml \
  --ref main \
  -f tag=vYYYY.M.PATCH-beta.N \
  -f preflight_run_id=<successful-openclaw-npm-preflight-run-id> \
  -f full_release_validation_run_id=<successful-full-release-validation-run-id> \
  -f full_release_validation_run_attempt=<successful-full-release-validation-run-attempt> \
  -f npm_dist_tag=beta
```

若要在快速变化的分支上验证固定提交，请使用辅助工具，而不是
`gh workflow run ... --ref main -f ref=<sha>`：

```bash
pnpm ci:full-release --sha <full-sha>
```

GitHub 工作流调度引用必须是分支或标签，不能是原始提交 SHA。该
辅助工具会在可信的 `main`
工作流 SHA 上推送临时 `release-ci/<sha>-...` 分支，通过工作流的 `ref` 输入传递请求的目标 SHA，
在可用时复用严格的精确目标证据，验证每个子
工作流的 `headSha` 都与可信工作流 SHA 匹配，并在运行完成后删除临时
分支。传入 `-f reuse_evidence=false` 可强制执行全新的
验证。如果任何子工作流使用了不同的工作流 SHA，总控验证器也会失败。

`release_profile` 控制传入发布检查的实时/提供商覆盖范围。
手动发布工作流默认为 `stable`；仅在你
有意需要广泛的建议性提供商/媒体矩阵时使用 `full`。稳定版和完整
发布检查始终运行详尽的实时/E2E 和 Docker 发布路径浸泡测试；
beta 配置可通过 `run_release_soak=true` 选择启用。

- `beta` 保留最快的 OpenAI/核心发布关键通道。
- `stable` 添加稳定的提供商/后端集合。
- `full` 运行广泛的建议性提供商/媒体矩阵。

总控工作流会记录所调度的子运行 ID，最终的 `Verify full validation` 任务会重新检查当前子运行的结论，并为每个子运行附加最慢任务表。如果子工作流重跑后变为绿色，只需重跑父级验证器任务，即可刷新总控结果和耗时摘要。

为进行恢复，`Full Release Validation` 和 `OpenClaw Release Checks` 都接受 `rerun_group`。对于候选发布版本，使用 `all`；仅运行常规完整 CI 子工作流，使用 `ci`；仅运行插件预发布子工作流，使用 `plugin-prerelease`；仅运行 OpenClaw Performance 子工作流，使用 `performance`；运行所有发布子工作流，使用 `release-checks`；或者在总工作流上使用更窄的组：`install-smoke`、`cross-os`、`live-e2e`、`package`、`qa`、`qa-parity`、`qa-live` 或 `npm-telegram`。这样，在针对性修复后，失败的发布执行环境重跑仍能限定在一定范围内。对于单个失败的跨操作系统通道，将 `rerun_group=cross-os` 与 `cross_os_suite_filter` 组合使用，例如 `windows/packaged-upgrade`；耗时较长的跨操作系统命令会输出心跳行，打包升级摘要则包含各阶段耗时。选定的 Matrix 和 Telegram QA 通道会阻止常规发布验证，核心运行时配对工具覆盖率门禁也是如此。QA 一致性、运行时一致性以及设有门禁的 Discord、WhatsApp 和 Slack 实时通道仅供参考。

`OpenClaw Release Checks` 使用受信任的工作流引用，将所选引用一次性解析为 `release-package-under-test` tarball，然后将该工件传递给跨操作系统检查和 Package Acceptance；运行浸泡测试覆盖时，还会传递给实时/E2E 发布路径 Docker 工作流。这样可确保各发布执行环境使用一致的软件包字节，并避免在多个子任务中重复打包同一个候选版本。对于 Codex npm 插件实时通道，发布检查会传入由 `release_package_spec` 派生的匹配已发布插件规范、传入操作员提供的 `codex_plugin_spec`，或者将输入留空，让 Docker 脚本打包所选检出的 Codex 插件。

针对 `ref=main` 和 `rerun_group=all` 的重复 `Full Release Validation` 运行
会取代较旧的总工作流。父监控器会在父工作流被取消时，取消它
已分派的所有子工作流，因此较新的主分支验证
不会排在过时的两小时发布检查运行之后。发布分支/标签
验证和针对性重跑组会保留 `cancel-in-progress: false`。

## 实时和 E2E 分片

发布实时/E2E 子工作流保留广泛的原生 `pnpm test:live` 覆盖，但通过 `scripts/test-live-shard.mjs` 将其作为具名分片运行，而不是作为单个串行任务运行：

- `native-live-src-agents` 和 `native-live-src-agents-zai-coding`
- `native-live-src-gateway-core`
- 按提供商筛选的 `native-live-src-gateway-profiles` 任务
- `native-live-src-gateway-backends`
- `native-live-src-infra`
- `native-live-test`
- `native-live-extensions-a-k`
- `native-live-extensions-l-n`
- `native-live-extensions-moonshot`
- `native-live-extensions-openai`
- `native-live-extensions-o-z-other`
- `native-live-extensions-xai`
- 拆分后的媒体音频/视频分片和按提供商筛选的音乐分片

这样既能保持相同的文件覆盖范围，又能更轻松地重跑和诊断缓慢的实时提供商故障。聚合的 `native-live-src-gateway`、`native-live-extensions-o-z`、`native-live-extensions-media` 和 `native-live-extensions-media-music` 分片名称仍可用于手动单次重跑。

原生实时媒体分片在由 `Live Media Runner Image` 工作流构建的 `ghcr.io/openclaw/openclaw-live-media-runner:ubuntu-24.04` 中运行。该镜像预装了 `ffmpeg` 和 `ffprobe`；媒体任务在设置前只验证这些二进制文件。请将 Docker 支持的实时套件保留在常规 Blacksmith 运行器上——容器任务不适合启动嵌套 Docker 测试。

Docker 支持的实时模型/后端分片会为每个所选提交使用单独的共享 `ghcr.io/openclaw/openclaw-live-test:<sha>-<extensions>` 镜像。实时发布工作流只构建并推送该镜像一次，随后 Docker 实时模型、按提供商分片的 Gateway 网关、CLI 后端、ACP 绑定和 Codex harness 分片会使用 `OPENCLAW_SKIP_DOCKER_BUILD=1` 运行。Gateway 网关 Docker 分片在工作流任务超时之前设置了明确的脚本级 `timeout` 上限，因此卡住的容器或清理路径会快速失败，而不会耗尽整个发布检查时间预算。如果这些分片各自重新构建完整的源代码 Docker 目标，则说明发布运行配置有误，并会因重复构建镜像而浪费实际时间。

## Package Acceptance

当问题是“这个可安装的 OpenClaw 软件包能否作为产品正常工作？”时，使用 `Package Acceptance`。它与常规 CI 不同：常规 CI 验证源代码树，而 Package Acceptance 则通过用户安装或更新后所使用的同一 Docker E2E 测试框架来验证单个 tarball。

### 任务

1. `resolve_package` 检出 `workflow_ref`，解析一个软件包候选版本，写入 `.artifacts/docker-e2e-package/openclaw-current.tgz`，写入 `.artifacts/docker-e2e-package/package-candidate.json`，将二者作为 `package-under-test` 工件上传，并在 GitHub 步骤摘要中输出来源、工作流引用、软件包引用、版本、SHA-256 和配置方案。
2. `package_integrity` 下载 `package-under-test` 工件，并使用 `scripts/check-openclaw-package-tarball.mjs` 强制执行公共软件包 tarball 契约。
3. `docker_acceptance` 使用解析后的软件包源 SHA（回退为 `workflow_ref`）和 `package_artifact_name=package-under-test` 调用 `openclaw-live-and-e2e-checks-reusable.yml`。可复用工作流会下载该工件、验证 tarball 清单、在需要时准备基于软件包摘要的 Docker 镜像，并针对该软件包运行所选 Docker 通道，而不是打包工作流检出内容。当一个配置方案选择多个目标 `docker_lanes` 时，可复用工作流会一次性准备软件包和共享镜像，然后将这些通道扇出为具有唯一工件的并行目标 Docker 任务。
4. `package_telegram` 可选择调用 `NPM Telegram Beta E2E`。当 `telegram_mode` 不是 `none` 时，它会运行；如果 Package Acceptance 已解析软件包，则会安装同一个 `package-under-test` 工件；独立 Telegram 分派仍可安装已发布的 npm 规范。
5. `summary` 会在软件包解析、完整性检查、Docker 验收或可选 Telegram 通道失败时使工作流失败。对于仅供参考的调用方，`advisory` 输入会将验收失败降级为警告。

### 候选版本来源

- `source=npm` 仅接受 `openclaw@extended-stable`、`openclaw@beta`、`openclaw@latest` 或确切的 OpenClaw 发布版本，例如 `openclaw@2026.4.27-beta.2`。将其用于已发布的延长稳定版、预发布版或稳定版验收。
- `source=ref` 打包受信任的 `package_ref` 分支、标签或完整提交 SHA。解析器会获取 OpenClaw 分支/标签，验证所选提交可从仓库分支历史记录或发布标签访问，在分离的工作树中安装依赖项，并使用 `scripts/package-openclaw-for-docker.mjs` 将其打包。
- `source=url` 下载公共 HTTPS `.tgz`；必须提供 `package_sha256`。此路径会拒绝 URL 凭据、非默认 HTTPS 端口、私有/内部/特殊用途主机名或解析后的 IP，以及不符合相同公共安全策略的重定向。
- `source=trusted-url` 从 `.github/package-trusted-sources.json` 中具名的受信任来源策略下载 HTTPS `.tgz`；必须提供 `package_sha256` 和 `trusted_source_id`。仅将其用于维护者拥有的企业镜像或需要配置主机、端口、路径前缀、重定向主机或私有网络解析的私有软件包仓库。如果策略声明了持有者身份验证，工作流会使用固定的 `OPENCLAW_TRUSTED_PACKAGE_TOKEN` 密钥；仍会拒绝嵌入 URL 的凭据。
- `source=artifact` 从 `artifact_run_id` 和 `artifact_name` 下载一个 `.tgz`；`package_sha256` 是可选的，但对于向外部共享的工件应予以提供。

将 `workflow_ref` 和 `package_ref` 分开处理。`workflow_ref` 是运行测试的受信任工作流/测试框架代码。`package_ref` 是在 `source=ref` 时被打包的源提交。这样，当前测试框架便能验证较旧的受信任源提交，而无需运行旧工作流逻辑。

### 套件配置方案

- `smoke` — `npm-onboard-channel-agent`、`gateway-network`、`config-reload`
- `package` — `npm-onboard-channel-agent`、`doctor-switch`、`update-channel-switch`、`skill-install`、`update-corrupt-plugin`、`upgrade-survivor`、`published-upgrade-survivor`、`root-managed-vps-upgrade`、`update-restart-auth`、`plugins-offline`、`plugin-update`
- `product` — `package` 集合，其中使用实时 `plugins` 覆盖代替 `plugins-offline`，另加 `mcp-channels`、`cron-mcp-cleanup`、`openai-web-search-minimal`、`openwebui`
- `full` — 包含 OpenWebUI 的完整 Docker 发布路径分块
- `custom` — 确切的 `docker_lanes`；当 `suite_profile=custom` 时必须提供

`package` 配置方案使用离线插件覆盖，因此已发布软件包的验证不会受实时 ClawHub 可用性限制。可选 Telegram 通道会在 `NPM Telegram Beta E2E` 中复用 `package-under-test` 工件，同时保留已发布 npm 规范路径供独立分派使用。

有关专门的更新和插件测试策略，包括本地命令、
Docker 通道、Package Acceptance 输入、发布默认值和失败分类排查，
请参阅[更新和插件测试](/zh-CN/help/testing-updates-plugins)。

发布检查使用 `source=artifact`、准备好的发布软件包工件、`suite_profile=custom`、`docker_lanes='doctor-switch update-channel-switch skill-install update-corrupt-plugin upgrade-survivor published-upgrade-survivor root-managed-vps-upgrade update-restart-auth plugins-offline plugin-update plugin-binding-command-escape'` 和 `telegram_mode=mock-openai` 调用 Package Acceptance。这样可确保软件包迁移、更新、实时 ClawHub 技能安装、过时插件依赖项清理、已配置插件安装修复、离线插件、插件更新和 Telegram 验证均使用同一个解析后的软件包 tarball。在发布 Beta 版后，在 Full Release Validation 或 OpenClaw Release Checks 上设置 `release_package_spec`，即可针对已发布的 npm 软件包运行相同矩阵而无需重新构建；仅当 Package Acceptance 需要使用与发布验证其余部分不同的软件包时，才设置 `package_acceptance_package_spec`。跨操作系统发布检查仍覆盖特定于操作系统的新手引导、安装程序和平台行为；软件包/更新产品验证应从 Package Acceptance 开始。

`published-upgrade-survivor` Docker 通道会在阻塞式发布路径中，每次运行验证一个已发布软件包基线。在 Package Acceptance 中，解析后的 `package-under-test` tarball 始终是候选版本，`published_upgrade_survivor_baseline` 用于选择回退的已发布基线，默认为 `openclaw@latest`；失败通道的重跑命令会保留该基线。使用 `run_release_soak=true` 或 `release_profile=full` 的 Full Release Validation 会设置 `published_upgrade_survivor_baselines='last-stable-4 2026.4.23 2026.5.2 2026.4.15'` 和 `published_upgrade_survivor_scenarios=reported-issues`，将范围扩展到最新四个稳定版 npm 发布版本，以及固定的插件兼容性边界版本和针对具体问题设计的测试夹具，涵盖 Feishu 配置、保留的引导启动/角色设定文件、已配置的 OpenClaw 插件安装、波浪号日志路径和过时的旧版插件依赖根目录。多基线已发布升级存活测试会按基线分片到单独的目标 Docker 运行器任务中。当问题是全面验证已发布版本的更新清理，而不是常规 Full Release CI 的覆盖广度时，单独的 `Update Migration` 工作流会使用具有 `all-since-2026.4.23` 基线和 `plugin-deps-cleanup` 场景的 `update-migration` Docker 通道。本地聚合运行可以通过 `OPENCLAW_UPGRADE_SURVIVOR_BASELINE_SPECS` 传入确切的软件包规范，通过 `OPENCLAW_UPGRADE_SURVIVOR_BASELINE_SPEC`（例如 `openclaw@2026.4.15`）只保留单个通道，或者设置 `OPENCLAW_UPGRADE_SURVIVOR_SCENARIOS` 以使用场景矩阵。已发布版本通道使用内置的 `openclaw config set` 命令步骤配置基线，在 `summary.json` 中记录步骤，并在 Gateway 网关启动后探测 `/healthz`、`/readyz` 以及 RPC 状态。Windows 打包版和安装程序全新安装通道还会验证，已安装的软件包能否从原始的 Windows 绝对路径导入浏览器控制替代实现。OpenAI 跨操作系统 Agent 轮次冒烟测试会在已设置 `OPENCLAW_CROSS_OS_OPENAI_MODEL` 时默认使用它，否则使用 `openai/gpt-5.6-luna`，以便安装和 Gateway 网关验证使用成本较低的 GPT-5.6 测试层级。

### 旧版兼容窗口

Package Acceptance 对已发布的软件包设有有限的旧版兼容窗口。截至 `2026.4.25` 的软件包（包括 `2026.4.25-beta.*`）可以使用兼容路径：

- `dist/postinstall-inventory.json` 中已知的私有 QA 条目可以指向 tarball 中省略的文件；
- 当软件包未公开该标志时，`doctor-switch` 可以跳过 `gateway install --wrapper` 持久化子用例；
- `update-channel-switch` 可以从基于 tarball 派生的伪 git 固件中移除缺失的 pnpm `patchedDependencies`，并可以记录缺失的已持久化 `update.channel`；
- 插件冒烟测试可以读取旧版安装记录位置，或接受缺失的市场安装记录持久化；
- `plugin-update` 可以允许配置元数据迁移，但仍要求安装记录和不重新安装行为保持不变。

已发布的 `2026.4.26` 软件包还可以针对已随包发布的本地构建元数据戳文件发出警告；截至 `2026.5.20` 的软件包在缺少 `npm-shrinkwrap.json` 时可以发出警告而非失败。后续软件包必须满足现代契约；遇到相同情况时将失败，而非警告或跳过。

### 示例

```bash
# 使用产品级覆盖验证当前 beta 软件包。
gh workflow run package-acceptance.yml \
  --ref main \
  -f workflow_ref=main \
  -f source=npm \
  -f package_spec=openclaw@beta \
  -f suite_profile=product \
  -f telegram_mode=mock-openai

# 使用软件包覆盖验证已发布的 extended-stable 软件包。
gh workflow run package-acceptance.yml \
  --ref main \
  -f workflow_ref=main \
  -f source=npm \
  -f package_spec=openclaw@extended-stable \
  -f suite_profile=package \
  -f telegram_mode=mock-openai

# 使用当前测试框架打包并验证发布分支。
gh workflow run package-acceptance.yml \
  --ref main \
  -f workflow_ref=main \
  -f source=ref \
  -f package_ref=release/YYYY.M.PATCH \
  -f suite_profile=package \
  -f telegram_mode=mock-openai

# 验证 tarball URL。source=url 时必须提供 SHA-256。
gh workflow run package-acceptance.yml \
  --ref main \
  -f workflow_ref=main \
  -f source=url \
  -f package_url=https://example.com/openclaw-current.tgz \
  -f package_sha256=<64-char-sha256> \
  -f suite_profile=smoke

# 验证来自指定可信私有镜像策略的 tarball。
gh workflow run package-acceptance.yml \
  --ref main \
  -f workflow_ref=main \
  -f source=trusted-url \
  -f trusted_source_id=enterprise-artifactory \
  -f package_url=https://packages.example.internal:8443/artifactory/openclaw/openclaw-current.tgz \
  -f package_sha256=<64-char-sha256> \
  -f suite_profile=smoke

# 复用另一次 Actions 运行上传的 tarball。
gh workflow run package-acceptance.yml \
  --ref main \
  -f workflow_ref=main \
  -f source=artifact \
  -f artifact_run_id=<run-id> \
  -f artifact_name=package-under-test \
  -f suite_profile=custom \
  -f docker_lanes='install-e2e plugin-update'
```

调试失败的软件包验收运行时，先查看 `resolve_package` 摘要，确认软件包来源、版本和 SHA-256。然后检查 `docker_acceptance` 子运行及其 Docker 工件：`.artifacts/docker-tests/**/summary.json`、`failures.json`、通道日志、阶段计时和重新运行命令。应优先重新运行失败的软件包配置或具体 Docker 通道，而非重新运行完整发布验证。

## 安装冒烟测试

`Install Smoke` 工作流不再针对拉取请求或 `main` 推送运行。其每夜/手动包装器和发布验证均调用只读的 `install-smoke-reusable.yml` 核心，并且每次运行都会在 GitHub 托管的运行器上执行完整的安装冒烟测试路径：

- 每个目标 SHA 只构建一次根 Dockerfile 冒烟测试镜像，并通过不可变工件绑定到工作流修订版本和生成方尝试次数；随后由 CLI 冒烟测试、智能体删除共享工作区 CLI 冒烟测试、容器 Gateway 网关网络 E2E，以及内置 `matrix` 插件 build-arg 冒烟测试加载该镜像。插件冒烟测试会验证运行时依赖安装镜像同步，并确认插件加载时不会出现入口逃逸诊断。
- QR 软件包安装和安装程序/更新 Docker 冒烟测试（包括 Rocky Linux 安装程序通道，以及针对可配置 `update_baseline_version` npm 基线的更新通道）作为单独的作业运行，因此安装程序工作无需排在根镜像冒烟测试之后等待。

较慢的 Bun 全局安装镜像提供商冒烟测试由 `run_bun_global_install_smoke` 单独控制。它按每夜计划运行，对于来自发布检查的工作流调用默认启用，手动 `Install Smoke` 调度也可选择启用。常规 PR CI 仍会针对与 Node 相关的变更运行快速 Bun 启动器回归通道。QR 和安装程序 Docker 测试继续使用各自专注于安装的 Dockerfile。

## 本地 Docker E2E

`pnpm test:docker:all` 会预构建一个共享的实时测试镜像，将 OpenClaw 一次性打包为 npm tarball，并构建两个共享的 `scripts/e2e/Dockerfile` 镜像：

- 用于安装程序/更新/插件依赖通道的纯 Node/Git 运行器；
- 将同一个 tarball 安装到 `/app` 中、用于常规功能通道的功能镜像。

Docker 通道定义位于 `scripts/lib/docker-e2e-scenarios.mjs`，规划器逻辑位于 `scripts/lib/docker-e2e-plan.mjs`，运行器只执行选定的计划。调度器通过 `OPENCLAW_DOCKER_E2E_BARE_IMAGE` 和 `OPENCLAW_DOCKER_E2E_FUNCTIONAL_IMAGE` 为每个通道选择镜像，然后使用 `OPENCLAW_SKIP_DOCKER_BUILD=1` 运行通道。

### 可调参数

| 变量                               | 默认值 | 用途                                                                                       |
| -------------------------------------- | ------- | --------------------------------------------------------------------------------------------- |
| `OPENCLAW_DOCKER_ALL_PARALLELISM`      | 10      | 常规通道的主池槽位数。                                                        |
| `OPENCLAW_DOCKER_ALL_TAIL_PARALLELISM` | 10      | 对提供商敏感的尾池槽位数。                                                      |
| `OPENCLAW_DOCKER_ALL_LIVE_LIMIT`       | 9       | 实时通道并发上限，避免提供商限流。                                        |
| `OPENCLAW_DOCKER_ALL_NPM_LIMIT`        | 5       | npm 安装通道并发上限。                                                              |
| `OPENCLAW_DOCKER_ALL_SERVICE_LIMIT`    | 7       | 多服务通道并发上限。                                                            |
| `OPENCLAW_DOCKER_ALL_START_STAGGER_MS` | 2000    | 通道启动之间的错峰时间，用于避免 Docker 守护进程创建风暴；设置 `0` 可取消错峰。     |
| `OPENCLAW_DOCKER_ALL_LANE_TIMEOUT_MS`  | 7200000 | 每通道后备超时时间（120 分钟）；选定的实时/尾部通道使用更严格的上限。           |
| `OPENCLAW_DOCKER_ALL_DRY_RUN`          | 未设置   | `1` 输出调度器计划而不运行通道。                                          |
| `OPENCLAW_DOCKER_ALL_LANES`            | 未设置   | 以逗号分隔的精确通道列表；跳过清理冒烟测试，使智能体能够复现单个失败通道。 |

即使某个通道所需的容量高于其实际上限，它仍可从空池启动，随后独占运行，直到释放容量。本地聚合流程会预检 Docker、移除过时的 OpenClaw E2E 容器、输出活动通道状态、持久化通道耗时以便按最长优先排序，并且默认在首次失败后停止调度新的池化通道。

### 可复用的实时/E2E 工作流

可复用的实时/E2E 工作流会询问 `scripts/test-docker-all.mjs --plan-json` 需要哪种软件包、镜像类型、实时镜像、通道和凭据覆盖。随后，`scripts/docker-e2e.mjs` 将该计划转换为 GitHub 输出和摘要。它会通过 `scripts/package-openclaw-for-docker.mjs` 打包 OpenClaw、下载当前运行的软件包工件，或从 `package_artifact_run_id` 下载软件包工件，然后验证 tarball 清单。默认的 `no-push-artifact` 路径通过 Blacksmith 的 Docker 层缓存构建带软件包摘要标签的纯净/功能镜像，将镜像的精确字节打包到不可变工作流工件中，并让每个使用方验证并加载该工件。`existing-only` 则要求明确指定 `docker_e2e_bare_image`/`docker_e2e_functional_image` GHCR 引用，且从不构建或推送。这些注册表拉取操作对每次尝试设置了 180 秒的有限超时，使卡住的数据流能够快速重试，而不是占用 CI 关键路径的大部分时间。计划验证成功后，`openclaw-scheduled-live-checks.yml` 会将不可变的已测试镜像清单传递给独立的软件包写入发布器；只读的发布版和预发布版调用方绝不会经过该写入器。

### 发布路径分块

发布 Docker 覆盖通过 `OPENCLAW_SKIP_DOCKER_BUILD=1` 运行较小的分块作业，使每个分块仅验证并加载其所需的工件支持镜像类型（或在明确的 `existing-only` 复用模式下拉取镜像），并通过同一个加权调度器执行多个通道：

- `OPENCLAW_DOCKER_ALL_PROFILE=release-path`
- `OPENCLAW_DOCKER_ALL_CHUNK=core | package-update-openai | package-update-anthropic | package-update-core | plugins-runtime-plugins | plugins-runtime-services | plugins-runtime-install-a..h | openwebui`

当前发布 Docker 分块为 `core`、`package-update-openai`、`package-update-anthropic`、`package-update-core`、`plugins-runtime-plugins`、`plugins-runtime-services`、`plugins-runtime-install-a` 至 `plugins-runtime-install-h`，以及 `openwebui`。`package-update-openai` 包含实时 Codex 插件软件包通道，该通道会安装候选 OpenClaw 软件包，从 `codex_plugin_spec` 或同一引用的 tarball 安装 Codex 插件并明确批准 Codex CLI 安装，运行 Codex CLI 预检和同一会话内的智能体轮次，然后运行一次零重试、中等思考级别的轮次：发送进度、读取随机化工作区输入、写入其精确工件，并发送完成消息。`plugins-runtime-core`、`plugins-runtime` 和 `plugins-integrations` 仍是聚合插件/运行时别名。`install-e2e` 通道别名仍是两个提供商安装程序通道的聚合手动重新运行别名。

每当稳定版或完整发布路径覆盖请求 OpenWebUI 时，它都会作为独立的 `openwebui` 分块在专用的大磁盘 Blacksmith 运行器上运行，即使可复用工作流将支持的作业路由到 GitHub 托管的运行器也是如此。将外部镜像拉取保持独立，可防止大型镜像在 `plugins-runtime-services` 中与共享软件包和插件镜像争用资源；旧版聚合插件/运行时分块仍包含 OpenWebUI，以便进行兼容的手动重新运行。内置渠道更新通道会针对临时 npm 网络故障重试一次。

每个分块都会上传 `.artifacts/docker-tests/`，其中包含通道日志、计时、`summary.json`、`failures.json`、阶段计时、调度器计划 JSON、慢通道表格和逐通道重新运行命令。工作流的 `docker_lanes` 输入会使用为该次运行准备的镜像执行选定通道，而不是使用分块作业，从而将失败通道调试限制在单个有针对性的 Docker 作业内；如果选定通道是实时 Docker 通道，目标作业会为该次重新运行在本地构建实时测试镜像。重新运行辅助工具会验证失败工件中精确选定的目标 SHA，并且手动调度会重新打包该引用，因为内部可复用工作流的软件包元组不属于 `workflow_dispatch` 架构。生成的命令仅在这些输入由 GHCR 支持时包含已准备的镜像输入和 `shared_image_policy=existing-only`；运行器本地工件标签会被省略，以便新运行器重新构建它们。除非工件能够证明恢复的 GHCR 镜像引用与显式目标覆盖匹配，否则显式目标覆盖会丢弃这些引用。工件生成的工作流定义引用也会被省略，因为完整发布的临时分支会被删除；除非操作员明确覆盖，否则调度将使用仓库默认分支。

```bash
pnpm test:docker:rerun <run-id>      # 下载 Docker 工件并输出组合式/逐通道的定向重新运行命令
pnpm test:docker:timings <summary>   # 慢通道和阶段关键路径摘要
```

计划运行的实时/E2E 工作流每天执行完整的发布路径 Docker 套件，并在成功后针对已精确测试的镜像工件调用显式发布器。

## 插件预发布版

`Plugin Prerelease` 的产品/软件包覆盖成本更高，因此它是一个由 `Full Release Validation` 或显式操作员调度的独立工作流。普通拉取请求、`main` 推送和独立的手动 CI 调度均不会启用该套件。它将内置插件测试分配到八个扩展工作节点上；这些扩展分片作业一次最多运行两个插件配置组，每组使用一个 Vitest 工作线程，并采用更大的 Node 堆，以免导入密集型插件批次产生额外的 CI 作业。仅发布时运行的 Docker 预发布路径（通过 `full_release_validation` 输入启用）以每组四个的方式批量运行目标 Docker 通道，避免为仅耗时一到三分钟的作业预留数十个运行器。该工作流还会从 `@openclaw/plugin-inspector` 上传一个信息性 `plugin-inspector-advisory` 工件；检查器发现是分类处置的输入，不会改变具有阻塞作用的插件预发布门禁。

## QA Lab

QA Lab 在主要智能作用域工作流之外拥有专用的 CI 通道。智能体一致性检查嵌套在广泛的 QA 和发布测试框架中，而不是独立的 PR 工作流。当一致性检查应随广泛验证运行时，使用 `Full Release Validation` 和 `rerun_group=qa-parity`。

- `QA-Lab - All Lanes` 工作流每晚在 `main` 上运行，也可通过手动调度运行；它会扇出模拟一致性检查以及 Matrix、Telegram、Discord、WhatsApp 和 Slack 的实时作业。实时作业使用 `qa-live-shared` 环境；Telegram、Discord、WhatsApp 和 Slack 使用 Convex 租约，而 Matrix 则配置一次性的本地凭据。

发布检查使用确定性模拟提供商和具备模拟资格的模型（`mock-openai/gpt-5.6-luna` 和 `mock-openai/gpt-5.6-luna-alt`）运行 Matrix 和 Telegram 实时传输通道，使渠道契约不受实时模型延迟和常规提供商插件启动的影响。实时传输 Gateway 网关会禁用记忆搜索，因为 QA 一致性检查会单独覆盖记忆行为；提供商连接性由独立的实时模型、原生提供商和 Docker 提供商套件覆盖。

定时和发布 Matrix 门禁使用共享的 QA Lab 套件主机和实时适配器运行发布场景。CLI 默认值和手动工作流输入仍为 `all`；手动 `all` 调度会扇出 `transport`、`media`、`e2ee-smoke`、`e2ee-deep` 和 `e2ee-cli` 配置文件，使 93 个场景的验证保持在各作业超时限制内。聚焦式手动调度会在单个作业中选择 `fast`、`release` 或 `transport`。

`OpenClaw Release Checks` 还会在发布审批前运行发布关键型 QA Lab 通道；其 QA 一致性门禁将候选包和基线包作为并行通道作业运行，然后将两个工件下载到一个小型报告作业中，以执行最终的一致性比较。

对于普通 PR，应遵循作用域化的 CI/检查证据，而不要将一致性检查视为必需状态。

## CodeQL

`CodeQL` 工作流特意设计为窄范围的第一轮安全扫描器，而非完整的仓库扫描。每日、手动、`main` 推送和非草稿拉取请求守卫运行会扫描 Actions 工作流代码以及风险最高的 JavaScript/TypeScript 表面，使用高置信度安全查询，并筛选为高危/严重 `security-severity`。

拉取请求守卫保持轻量：它仅在 `.github/actions`、`.github/codeql`、`.github/workflows`、`packages`、`scripts`、`src` 或拥有进程的内置插件运行时路径发生更改时启动，并运行与定时工作流相同的高置信度安全矩阵。Android 和 macOS CodeQL 不纳入 PR 默认检查。

### 安全类别

| 类别                                              | 表面                                                                                                                                |
| ------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------- |
| `/codeql-security-high/core-auth-secrets`         | 身份验证、机密、沙箱、cron 和 Gateway 网关基线                                                                                     |
| `/codeql-security-high/channel-runtime-boundary`  | 核心渠道实现契约，以及渠道插件运行时、Gateway 网关、插件 SDK、机密和审计接触点                                                     |
| `/codeql-security-high/network-ssrf-boundary`     | 核心 SSRF、IP 解析、网络守卫、Web 获取和插件 SDK SSRF 策略表面                                                                     |
| `/codeql-security-high/mcp-process-tool-boundary` | MCP 服务器、进程执行辅助程序、出站交付和智能体工具执行门禁                                                                          |
| `/codeql-security-high/process-exec-boundary`     | 本地 shell、进程派生辅助程序、拥有子进程的内置插件运行时和工作流脚本粘合代码                                                       |
| `/codeql-security-high/plugin-trust-boundary`     | 插件安装、加载器、清单、注册表、软件包管理器安装、源代码加载和插件 SDK 软件包契约信任表面                                          |

### 平台特定安全分片

- `CodeQL Android Critical Security` — 定时 Android 安全分片。在工作流完整性检查所接受的最小 Blacksmith Linux 运行器上手动构建 Android 应用以供 CodeQL 分析。以 `/codeql-critical-security/android` 名义上传。
- `CodeQL macOS Critical Security` — 每周/手动 macOS 安全分片。在 Blacksmith macOS 上手动构建 macOS 应用以供 CodeQL 分析，从上传的 SARIF 中过滤掉依赖项构建结果，并以 `/codeql-critical-security/macos` 名义上传。由于即使构建无问题，macOS 构建仍占据大部分运行时间，因此不纳入每日默认检查。

### 关键质量类别

`CodeQL Critical Quality` 是对应的非安全分片。它仅在 GitHub 托管的 Linux 运行器上，对窄范围的高价值表面运行错误严重级别的非安全 JavaScript/TypeScript 质量查询，以避免质量扫描消耗 Blacksmith 运行器注册预算。其拉取请求守卫特意比定时配置文件更小：非草稿 PR 仅运行与所触及表面匹配的分片，这些分片来自十三个可由 PR 路由的分片——`agent-runtime-boundary`、`channel-runtime-boundary`、`config-boundary`、`core-auth-secrets`、`gateway-runtime-boundary`、`mcp-process-runtime-boundary`、`memory-runtime-boundary`、`network-runtime-boundary`、`plugin-boundary`、`plugin-sdk-package-contract`、`plugin-sdk-reply-runtime`、`provider-runtime-boundary` 和 `session-diagnostics-boundary`。`ui-control-plane` 和 `web-media-runtime-boundary` 不参与 PR 运行。CodeQL 配置和质量工作流更改会运行完整的 PR 分片集（网络运行时分片由其自身的 CodeQL 配置文件和拥有网络功能的源代码路径触发）。

手动调度接受：

```text
profile=all|agent-runtime-boundary|config-boundary|core-auth-secrets|channel-runtime-boundary|gateway-runtime-boundary|memory-runtime-boundary|mcp-process-runtime-boundary|network-runtime-boundary|plugin-boundary|plugin-sdk-package-contract|plugin-sdk-reply-runtime|provider-runtime-boundary|session-diagnostics-boundary
```

这些窄范围配置文件是用于单独运行一个质量分片的教学/迭代入口。

| 类别                                                    | 表面                                                                                                                                                              |
| ------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `/codeql-critical-quality/core-auth-secrets`            | 身份验证、机密、沙箱、cron 和 Gateway 网关安全边界代码                                                                                                            |
| `/codeql-critical-quality/config-boundary`              | 配置架构、迁移、规范化和 IO 契约                                                                                                                                  |
| `/codeql-critical-quality/gateway-runtime-boundary`     | Gateway 网关协议架构和服务器方法契约                                                                                                                              |
| `/codeql-critical-quality/channel-runtime-boundary`     | 核心渠道和内置渠道插件实现契约                                                                                                                                    |
| `/codeql-critical-quality/agent-runtime-boundary`       | 命令执行、模型/提供商调度、自动回复调度和队列，以及 ACP 控制平面运行时契约                                                                                       |
| `/codeql-critical-quality/mcp-process-runtime-boundary` | MCP 服务器和工具桥接、进程监管辅助程序以及出站交付契约                                                                                                            |
| `/codeql-critical-quality/memory-runtime-boundary`      | 记忆主机 SDK、记忆运行时外观、记忆插件 SDK 别名、记忆运行时激活粘合代码和记忆 Doctor 命令                                                                        |
| `/codeql-critical-quality/network-runtime-boundary`     | 网络策略软件包、原始套接字和代理捕获运行时、SSH 隧道、Gateway 网关锁、JSONL 套接字和推送传输表面                                                                 |
| `/codeql-critical-quality/session-diagnostics-boundary` | 回复队列内部机制、会话交付队列、出站会话绑定/交付辅助程序、诊断事件/日志包表面和会话 Doctor CLI 契约                                                              |
| `/codeql-critical-quality/plugin-sdk-reply-runtime`     | 插件 SDK 入站回复调度、回复载荷/分块/运行时辅助程序、渠道回复选项、交付队列和会话/线程绑定辅助程序                                                                |
| `/codeql-critical-quality/provider-runtime-boundary`    | 模型目录规范化、提供商身份验证和设备发现、提供商运行时注册、提供商默认值/目录，以及 Web/搜索/获取/嵌入注册表                                                      |
| `/codeql-critical-quality/ui-control-plane`             | Control UI 引导启动、本地持久化、Gateway 网关控制流和任务控制平面运行时契约                                                                                       |
| `/codeql-critical-quality/web-media-runtime-boundary`   | 核心 Web 获取/搜索、媒体 IO、媒体理解、图像生成和媒体生成运行时契约                                                                                               |
| `/codeql-critical-quality/plugin-boundary`              | 加载器、注册表、公共表面和插件 SDK 入口点契约                                                                                                                     |
| `/codeql-critical-quality/plugin-sdk-package-contract`  | 已发布软件包侧的插件 SDK 源代码和插件软件包契约辅助程序                                                                                                           |

质量检查与安全检查保持分离，以便在不模糊安全信号的情况下定时运行、度量、禁用或扩展质量发现。只有在窄范围配置文件的运行时间和信号稳定后，才应将 Swift、Python 和内置插件 CodeQL 扩展作为作用域化或分片化的后续工作重新加入。

## 维护工作流

### 文档 Agent

`Docs Agent` 工作流是一个事件驱动的 Codex 维护通道，用于使现有文档与最近合入的更改保持一致。它没有纯定时计划：`main` 上成功的非机器人推送 CI 运行可以触发它，也可以通过手动调度直接运行。当 `main` 已继续推进，或过去一小时内已创建另一个未跳过的文档 Agent 运行时，工作流运行调用会跳过。运行时，它会审查从上一个未跳过的文档 Agent 源 SHA 到当前 `main` 的提交范围，因此每小时的一次运行可以覆盖自上次文档检查以来累积的所有主分支更改。

### 测试性能 Agent

`Test Performance Agent` 工作流是一个事件驱动的 Codex 慢速测试维护通道。它没有纯定时计划：`main` 上一次成功的非机器人推送 CI 运行可以触发它，但如果同一 UTC 日期内另一个 workflow-run 调用已经运行或正在运行，它就会跳过。手动调度会绕过该每日活动门控。该通道会生成全套测试的分组 Vitest 性能报告，仅允许 Codex 进行保持覆盖率的小型测试性能修复，而不是大范围重构，然后重新运行全套测试报告，并拒绝任何导致通过基线测试数量减少的更改。分组报告会记录 Linux 和 macOS 上每项配置的实际耗时和最大 RSS，因此前后对比会在持续时间变化旁显示测试内存变化。如果基线中存在失败的测试，Codex 只能修复明显的故障，并且智能体处理后的全套测试报告必须通过，才能提交任何内容。当 `main` 在机器人推送落地前推进时，该通道会对已验证的补丁执行变基，重新运行 `pnpm check:changed`，然后重试推送；存在冲突的过期补丁会被跳过。它使用 GitHub 托管的 Ubuntu，因此 Codex action 可以保持与文档智能体相同的 drop-sudo 安全策略。

### 合并后的重复 PR

`Duplicate PRs After Merge` 工作流是用于落地后清理重复项的手动维护者工作流。它默认执行试运行，仅当 `apply=true` 时才会关闭明确列出的 PR。在修改 GitHub 之前，它会验证已落地的 PR 已合并，并且每个重复 PR 要么引用了同一个问题，要么修改的代码块存在重叠。

```bash
gh workflow run duplicate-after-merge.yml \
  -f landed_pr=70532 \
  -f duplicate_prs='70530,70592' \
  -f apply=true
```

## 本地检查门控和变更路由

### 配置基线计数棘轮

`pnpm config:docs:check` 会拒绝未记录的配置表面增长以及损坏或过期的计数快照。当经过审查的产品变更有意添加 schema 路径时，运行 `pnpm config:docs:gen`，检查核心/渠道/插件计数变化和生成的 SHA-256 文件，并将有意识的基线提升与 schema、帮助信息、标签、迁移和测试一起提交。不要手动编辑计数文件来绕过棘轮。

配置作者还必须为 Settings 中的新叶节点划分层级。在叶节点添加 `advanced: false` 或
`advanced: true`，或者将键放在所有后代都应继承其层级的祖先节点下。
未分类的根节点会导致 schema 质量测试失败，并提供可复制粘贴的存根；没有祖先节点的路径默认归为高级层级。
精选的常用叶节点快照可让有意的层级变更在审查中清晰可见。

本地变更通道路由逻辑位于 `scripts/changed-lanes.mjs`，并由 `scripts/check-changed.mjs` 执行。该本地检查门控对架构边界的要求比宽泛的 CI 平台范围更严格：

- 核心生产代码变更会运行核心生产代码和核心测试类型检查，以及核心 lint/守卫检查；
- 仅核心测试变更只运行核心测试类型检查和核心 lint；
- 插件生产代码变更会运行插件生产代码和插件测试类型检查，以及插件 lint；
- 仅插件测试变更会运行插件测试类型检查和插件 lint；
- 公共插件 SDK 或插件契约变更会扩展到插件类型检查，因为插件依赖这些核心契约（Vitest 插件扫描仍属于显式测试工作）；
- 仅发布元数据的版本提升会运行针对性的版本/配置/根依赖检查；
- 未知的根目录/配置变更会安全地回退到所有检查通道。

本地变更测试路由逻辑位于 `scripts/test-projects.test-support.mjs`，并且特意设计得比 `check:changed` 更轻量：直接修改的测试会运行自身；源代码修改会优先使用显式映射，然后运行同级测试和导入图中的依赖项。共享群组房间交付配置是其中一项显式映射：对群组可见回复配置、源回复交付模式或消息工具系统提示词的更改，会路由至核心回复测试以及 Discord 和 Slack 交付回归测试，从而使共享默认值的变更在首次推送 PR 之前就失败。仅当变更对测试框架的影响足够广泛，导致轻量映射集合无法作为可靠代理时，才使用 `OPENCLAW_TEST_CHANGED_BROAD=1 pnpm test:changed`。

## Testbox 验证

Crabbox 是仓库自有的远程环境包装器，用于维护者的 Linux 验证。仅当源代码可信且现有依赖安装已就绪时，智能体
会话才会在本地运行少量重点测试和轻量静态检查。对于更大的测试套件和
计算密集型工作，包括构建、类型检查、lint 扇出、
Docker、软件包通道、E2E、实时验证和 CI 一致性验证，它们会使用 Crabbox。可信维护者的重型
验证默认使用 `blacksmith-testbox`，而 `.crabbox.yaml` 现在也默认使用它。其配置的
工作流会注入提供商和智能体凭据，因此不可信的贡献者或
复刻仓库代码必须改用无密钥的复刻仓库 CI 或经过净化的直接 AWS Crabbox。
经过净化的 AWS 运行会设置 `CRABBOX_ENV_ALLOW=CI`，传递
`--no-hydrate`，并使用一个全新的临时远程 `HOME`；这可防止仓库的
`OPENCLAW_*` 允许列表和现有身份验证配置文件接触不可信代码。
它们使用专门为该不可信源新预热的租约，绝不使用
可信或此前注入过凭据的租约。从干净、可信的 `main` 检出中启动已安装的可信 Crabbox
二进制文件，并仅通过 `--fresh-pr` 获取远程 PR；绝不在本地执行不可信检出的包装器或配置。
取消设置 `CRABBOX_AWS_INSTANCE_PROFILE`，并且除非解析后的
`aws.instanceProfile` 为空，否则以关闭方式失败。在任何安装/测试之前，使用可信的
绝对路径工具要求提供 IMDSv2 令牌，证明 IAM 凭据
端点返回 404，并将远程 `git rev-parse HEAD` 与完整的
已审查 PR 头部 SHA 进行比较。将租约绑定到该 SHA，并在头部发生变化时停止并重新预热。
将干净 `main` 中可信的 `scripts/crabbox-untrusted-bootstrap.sh`
与 `--fresh-pr` 一起上传；它会安装固定版本的 Node/pnpm，验证 SHA 和
软件包管理器固定版本，隔离 `HOME`，安装依赖项，然后执行
请求的测试。
取消设置所有 `CRABBOX_TAILSCALE*` 覆盖，强制使用 `--network public
--tailscale=false`，清除出口节点/LAN 标志，并要求 `crabbox inspect`
在上传任何脚本之前报告公共网络且无 Tailscale 状态。
自有 AWS/Hetzner 容量也仍可作为 Blacksmith 服务中断、
配额问题或明确要求使用自有容量测试时的后备方案。

智能体不会为预期工作提前预热。当第一个重型命令准备就绪时，才按需获取 Testbox，
复用返回的 `tbx_...` ID 执行后续重型
命令，每次运行时同步当前检出，并在交接前将其停止。

由 Crabbox 支持的 Blacksmith 运行会对一次性 Testbox 执行预热、认领、同步、运行、报告和清理。
内置同步完整性检查会在同步后环境中的
`git status --short` 显示至少 200 个已跟踪文件被删除时快速失败，
从而捕获 `pnpm-lock.yaml` 等根文件消失的情况。对于有意大量删除文件的
PR，请为远程命令设置 `CRABBOX_ALLOW_MASS_DELETIONS=1`。

如果本地 Blacksmith CLI 调用停留在
同步阶段超过五分钟，且同步后没有任何输出，Crabbox 也会将其终止。设置
`CRABBOX_BLACKSMITH_SYNC_TIMEOUT_MS=0` 可禁用该守卫；对于异常庞大的本地差异，也可以使用更大的
毫秒值。

首次运行前，请从仓库根目录检查包装器：

```bash
pnpm crabbox:run -- --help | sed -n '1,120p'
```

仓库包装器会拒绝未声明所选提供商的过期 Crabbox 二进制文件；由 Blacksmith 支持的运行要求 Crabbox 0.22.0 或更高版本，以便包装器获得当前的 Testbox 同步、队列和清理行为。在 Codex 工作树或链接/稀疏检出中，请避免使用本地 `pnpm crabbox:run` 脚本，因为 pnpm 可能会在 Crabbox 启动前协调依赖项；应改为直接调用 node 包装器：

```bash
node scripts/crabbox-wrapper.mjs run --provider blacksmith-testbox --timing-json --shell -- "pnpm test <path-or-filter>"
```

使用同级检出时，请在计时或验证工作之前重新构建被忽略的本地二进制文件：

```bash
version="$(git -C ../crabbox describe --tags --always --dirty | sed 's/^v//')" \
  && go build -C ../crabbox -trimpath -ldflags "-s -w -X github.com/openclaw/crabbox/internal/cli.version=${version}" -o bin/crabbox ./cmd/crabbox
```

`.crabbox.yaml` 中的 `blacksmith:` 块已经固定了组织、工作流、作业和引用的默认值，因此下面的显式标志是可选的。变更门控：

```bash
pnpm crabbox:run -- --provider blacksmith-testbox \
  --blacksmith-org openclaw \
  --blacksmith-workflow .github/workflows/ci-check-testbox.yml \
  --blacksmith-job check \
  --blacksmith-ref main \
  --idle-timeout 90m \
  --ttl 240m \
  --timing-json \
  --shell -- \
  "corepack pnpm check:changed"
```

当本地依赖不可用或目标会产生扇出时，在 Testbox 上重新运行重点测试：

```bash
pnpm crabbox:run -- --provider blacksmith-testbox \
  --idle-timeout 90m \
  --ttl 240m \
  --timing-json \
  --shell -- \
  "corepack pnpm test <path-or-filter>"
```

完整测试套件：

```bash
pnpm crabbox:run -- --provider blacksmith-testbox \
  --idle-timeout 90m \
  --ttl 240m \
  --timing-json \
  --shell -- \
  "corepack pnpm test"
```

阅读最终的 JSON 摘要。有用的字段是 `provider`、`leaseId`、
`syncDelegated`、`exitCode`、`commandMs` 和 `totalMs`。对于委派的
Blacksmith Testbox 运行，Crabbox 包装器的退出代码和 JSON 摘要就是
命令结果。关联的 GitHub Actions 运行负责凭据注入和保活；当 SSH
命令已经返回后 Testbox 又从外部停止时，它可能以 `cancelled` 状态结束。除非
包装器的 `exitCode` 非零，或命令输出显示测试失败，否则应将其视为清理/状态工件。
由 Blacksmith 支持的一次性 Crabbox 运行应自动停止 Testbox；
如果运行被中断或清理状态不明确，请检查实时环境，并仅停止
你创建的环境：

```bash
blacksmith testbox list --all
blacksmith testbox status --id <tbx_id>
blacksmith testbox stop --id <tbx_id>
```

仅当你有意需要在同一个已注入凭据的环境中执行多个命令时才使用复用：

```bash
node scripts/crabbox-wrapper.mjs run --provider blacksmith-testbox --id <tbx_id> --timing-json --shell -- "corepack pnpm test <path-or-filter>"
pnpm crabbox:stop -- <tbx_id>
```

复用租约，而不是过期的源代码。省略 `--no-sync`，以便每次运行都上传
当前检出；仅当你有意重新运行未更改且已同步的工作树时才使用它。
不可信的贡献者/复刻仓库代码必须为每个命令使用
`CRABBOX_ENV_ALLOW=CI`、`--provider aws --no-hydrate` 和一个全新的
临时远程 `HOME`；在该净化命令中安装依赖项后再进行测试。仅复用专门为
同一不可信源新预热的租约；绝不使用可信或此前注入过凭据的租约。绝不
在本地执行不可信检出的包装器或配置：从干净、可信的 `main` 启动已安装的
可信 Crabbox 二进制文件，并在每次运行时传递 `--fresh-pr`。
保持 `CRABBOX_AWS_INSTANCE_PROFILE` 未设置，拒绝非空的已解析
实例配置文件，要求可信远程环境提供 IMDS 无角色证明，并在安装/测试前验证
已审查的头部 SHA。将租约绑定到该 SHA；头部发生任何变化后都要停止并
重新预热。如果不存在远程 PR，请使用无密钥的复刻仓库 CI。
绝不为不可信源选择 `hydrate-github` 或注入凭据的 Blacksmith 工作流。

如果故障层是 Crabbox 而 Blacksmith 本身正常，仅将直接
Blacksmith 用于 `list`、`status` 和清理等诊断。在将直接 Blacksmith 运行视为维护者验证之前，
先修复 Crabbox 路径。

如果 `blacksmith testbox list --all` 和 `blacksmith testbox status` 可以工作，但新的
预热在几分钟后仍停留于 `queued`，既没有 IP，也没有 Actions 运行 URL，
应将其视为 Blacksmith 提供商、队列、计费或组织限制压力。停止你创建的
排队 ID，避免启动更多 Testbox，并将验证移至下方
自有 Crabbox 容量路径，同时由其他人检查 Blacksmith 仪表板、
计费和组织限制。

仅当 Blacksmith 宕机、受到配额限制、缺少所需环境，或明确以自有容量为目标时，才升级到自有 Crabbox 容量：

```bash
CRABBOX_CAPACITY_REGIONS=eu-west-1,eu-west-2,eu-central-1,us-east-1,us-west-2 \
  pnpm crabbox:warmup -- --provider aws --class standard --market on-demand --idle-timeout 90m
pnpm crabbox:hydrate -- --provider aws --id <cbx_id-or-slug>
pnpm crabbox:run -- --provider aws --id <cbx_id-or-slug> --timing-json --shell -- "pnpm check:changed"
pnpm crabbox:stop -- --provider aws <cbx_id-or-slug>
```

在 AWS 容量紧张时，除非任务确实需要 48xlarge 级别的 CPU，否则避免使用 `class=beast`。一个 `beast` 请求从 192 个 vCPU 起步，最容易触发区域性 EC2 Spot 或 On-Demand Standard 配额。仓库自有的 `.crabbox.yaml` 默认使用 `class: standard`、按需市场和 `capacity.hints: true`，因此经代理分配的 AWS 租约会输出所选区域/市场、配额压力、Spot 回退和高压力实例规格警告。较重的广泛检查使用 `fast`；仅在 standard/fast 不够用后使用 `large`；仅对异常消耗 CPU 的通道使用 `beast`，例如完整测试套件或全插件 Docker 矩阵、明确的发布/阻塞问题验证，或高核心数性能分析。不要将 `beast` 用于 `pnpm check:changed`、聚焦测试、仅文档工作、普通 lint/类型检查、小型 E2E 复现或 Blacksmith 故障排查。容量诊断应使用 `--market on-demand`，以免 Spot 市场波动混入诊断信号。

`.crabbox.yaml` 负责提供商、同步和 GitHub Actions 水合默认设置。Crabbox 同步绝不会传输 `.git`，因此水合后的 Actions 检出会保留其自身的远程 Git 元数据，而不会同步维护者本地的远程仓库配置和对象存储；此外，仓库配置还会排除绝不应传输的本地运行时/构建工件（例如 `.artifacts` 和测试报告）。`.github/workflows/crabbox-hydrate.yml` 负责检出、Node/pnpm 设置、`origin/main` 获取，以及自有云 `crabbox run --id <cbx_id>` 命令的非机密环境交接。

## 相关内容

- [安装概览](/zh-CN/install)
- [开发渠道](/zh-CN/install/development-channels)
