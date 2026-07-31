---
read_when:
    - 更改 OpenClaw 更新、Doctor、软件包验收或插件安装行为
    - 准备或批准候选版本
    - 调试软件包更新、插件依赖清理或插件安装回归问题
sidebarTitle: Update and plugin tests
summary: OpenClaw 如何验证更新路径、软件包迁移以及插件安装/更新行为
title: 更新和插件测试
x-i18n:
    generated_at: "2026-07-26T06:17:18Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 96a11fe42472f758d4fd1cc568486e301f7460982fdb547cab8b39de04a8dabe
    source_path: help/testing-updates-plugins.md
    workflow: 16
---

更新和插件验证清单：证明可安装软件包可以
更新真实用户状态，通过 `doctor` 修复过时的旧版状态，并且仍能
从每个受支持的来源安装、加载、更新和卸载插件。

关于更广泛的测试运行器地图，请参阅[测试](/zh-CN/help/testing)。关于实时提供商
密钥和需要访问网络的测试套件，请参阅[实时测试](/zh-CN/help/testing-live)。

## 我们保护的内容

- 软件包 tarball 内容完整，具有有效的 `dist/postinstall-inventory.json`，
  且不依赖未打包的仓库文件。
- 用户可以从较旧的已发布软件包迁移到候选软件包，
  而不会丢失配置、智能体、会话、工作区、插件允许列表或
  渠道配置。
- `openclaw doctor --fix --non-interactive` 负责旧版清理和修复
  路径。启动过程不应针对过时的插件状态增加隐藏的兼容性迁移。
- 支持从本地目录、git 仓库、npm 软件包和
  ClawHub 注册表路径安装插件。
- 插件 npm 依赖项在每个插件各自的托管 npm 项目中安装，
  在信任前接受扫描，并在卸载插件时通过 `npm uninstall`
  移除，以免提升安装的依赖项残留。
- 当没有任何变化时，插件更新为空操作：安装记录、解析后的
  来源、已安装依赖项布局和启用状态均保持不变。

## 开发期间的本地验证

从小范围开始：

```bash
pnpm changed:lanes --json
pnpm check:changed
pnpm test:changed
```

对于插件安装、卸载、依赖项或软件包清单变更，还应
运行覆盖已编辑接缝的针对性测试：

```bash
pnpm test src/plugins/uninstall.test.ts src/infra/package-dist-inventory.test.ts test/scripts/package-acceptance-workflow.test.ts
```

在任何软件包 Docker 通道使用 tarball 之前，验证软件包工件：

```bash
pnpm release:check
```

`release:check` 运行配置/文档/API 漂移检查（配置模式、配置文档
基线、插件 SDK API 契约清单和导出、插件版本/清单），
写入软件包发布清单，运行 `npm pack --dry-run`，拒绝禁止
打包的文件，将 tarball 安装到临时前缀中，运行安装后脚本，并对
内置渠道入口点执行冒烟测试。

## Docker 通道

Docker 通道是产品级验证。它们在 Linux 容器内安装或更新真实的
软件包，并通过 CLI 命令、Gateway 网关启动、HTTP 探测、RPC 状态和
文件系统状态断言行为。

迭代时使用针对性通道：

```bash
pnpm test:docker:plugins
pnpm test:docker:plugin-lifecycle-matrix
pnpm test:docker:plugin-update
pnpm test:docker:upgrade-survivor
pnpm test:docker:published-upgrade-survivor
pnpm test:docker:update-restart-auth
pnpm test:docker:update-migration
```

重要通道：

- `test:docker:plugins` 涵盖插件安装冒烟测试、本地文件夹安装、
  本地文件夹更新跳过行为、具有预安装依赖项的本地文件夹、
  `file:` 软件包安装、带 CLI 执行的 git 安装、git
  移动引用更新、带提升安装的传递依赖项的 npm 注册表安装、npm 更新
  空操作、拒绝格式错误的 npm 软件包元数据、本地 ClawHub 固件安装和更新
  空操作、市场更新行为，以及 Claude 捆绑包启用/检查。设置
  `OPENCLAW_PLUGINS_E2E_CLAWHUB=0` 以使 ClawHub 测试块保持封闭/离线。
- `test:docker:plugin-lifecycle-matrix` 在裸容器中安装候选软件包，
  让一个 npm 插件依次经历安装、检查、禁用、启用、
  显式升级、显式降级，以及删除插件代码后的卸载。它按阶段记录
  RSS 和 CPU 指标。
- `test:docker:plugin-update` 验证未发生变化的已安装插件在
  `openclaw plugins update` 期间不会重新安装或丢失安装元数据。
- `test:docker:upgrade-survivor` 在脏的旧用户固件上覆盖安装候选
  tarball，运行软件包更新和非交互式 Doctor，然后启动
  loopback Gateway 网关并检查状态是否保留。
- `test:docker:published-upgrade-survivor` 首先安装已发布的基线，
  通过内置的 `openclaw config set` 配方进行配置，将其更新到
  候选 tarball，运行 Doctor，检查旧版清理，启动 Gateway 网关，并
  探测 `/healthz`、`/readyz` 和 RPC 状态。
- `test:docker:update-restart-auth` 安装候选软件包，启动
  使用托管令牌身份验证的 Gateway 网关，为
  `openclaw update --yes --json` 取消设置调用方 Gateway 网关身份验证环境变量，
  并要求候选更新命令在常规探测前重启 Gateway 网关。
- `test:docker:update-migration` 是以清理为主的已发布版本更新通道。它
  从已配置的 Discord/Telegram 风格用户状态开始，运行基线
  Doctor，使已配置的插件依赖项有机会生成；为已配置的打包插件植入
  旧版插件依赖项残留；更新到候选 tarball；并要求更新后的 Doctor
  移除旧版依赖项根目录。

实用的已发布升级存续测试变体：

```bash
OPENCLAW_UPGRADE_SURVIVOR_BASELINE_SPEC=openclaw@2026.4.23 \
OPENCLAW_UPGRADE_SURVIVOR_SCENARIO=versioned-runtime-deps \
pnpm test:docker:published-upgrade-survivor

OPENCLAW_UPGRADE_SURVIVOR_BASELINE_SPEC=openclaw@latest \
OPENCLAW_UPGRADE_SURVIVOR_SCENARIO=bootstrap-persona \
pnpm test:docker:published-upgrade-survivor
```

可用场景：`base`、`acpx-openclaw-tools-bridge`、`feishu-channel`、
`bootstrap-persona`、`channel-post-core-restore`、`plugin-deps-cleanup`、
`configured-plugin-installs`、`stale-source-plugin-shadow`、`tilde-log-path`
和 `versioned-runtime-deps`。在聚合运行中，`OPENCLAW_UPGRADE_SURVIVOR_SCENARIOS=reported-issues`
（别名 `far-reaching`）会展开为所有场景，包括
已配置插件的安装迁移。

完整更新迁移有意与完整发布 CI 分开。当发布问题是“从 2026.4.23
开始的每个已发布稳定版本能否更新到此候选版本，并清理插件依赖项
残留？”时，请使用手动 `Update Migration` 工作流：

```bash
gh workflow run update-migration.yml \
  --ref main \
  -f workflow_ref=main \
  -f package_ref=main \
  -f baselines=all-since-2026.4.23 \
  -f scenarios=plugin-deps-cleanup
```

## 软件包验收

软件包验收是 GitHub 原生的软件包门禁。它将一个候选
软件包解析为 `package-under-test` tarball，记录版本和 SHA-256，然后
针对该确切 tarball 运行可复用的 Docker E2E 通道。工作流测试框架
引用与软件包来源引用相互独立，因此当前测试逻辑可以验证
较旧的可信版本。

候选来源：

- `source=npm`：验证 `openclaw@extended-stable`、`openclaw@beta`、
  `openclaw@latest` 或确切的已发布版本。
- `source=ref`：使用选定的当前测试框架打包可信分支、标签或提交。
- `source=url`：使用必需的 `package_sha256` 验证公共 HTTPS tarball。
  此路径拒绝 URL 凭据、非默认 HTTPS 端口、私有/内部
  主机名或 DNS/IP 结果、特殊用途 IP 空间以及不安全的重定向。
- `source=trusted-url`：使用必需的
  `package_sha256` 和 `trusted_source_id`，依据
  `.github/package-trusted-sources.json` 中由维护者拥有的策略验证 HTTPS tarball。对于企业/私有
  镜像，请使用此方式，而不是通过输入级允许私有资源开关来削弱
  `source=url`。由策略配置的不记名身份验证使用固定的
  `OPENCLAW_TRUSTED_PACKAGE_TOKEN` 密钥。
- `source=artifact`：复用由另一个 Actions 运行上传的 tarball。

完整发布验证默认使用 `source=artifact`，它由
解析后的发布 SHA 构建。对于发布后验证，请传入
`package_acceptance_package_spec=openclaw@YYYY.M.PATCH`，使同一升级矩阵改为面向已发布的 npm 软件包。

发布检查使用软件包/更新/重启/插件测试集调用软件包验收：

```text
doctor-switch update-channel-switch skill-install update-corrupt-plugin upgrade-survivor published-upgrade-survivor root-managed-vps-upgrade update-restart-auth plugins-offline plugin-update plugin-binding-command-escape
```

启用发布浸泡测试时（对 `release_profile=stable` 和
`full` 强制启用），它们还会传入：

```text
published_upgrade_survivor_baselines=last-stable-4 2026.4.23 2026.5.2 2026.4.15
published_upgrade_survivor_scenarios=reported-issues
telegram_mode=mock-openai
```

这样可以让软件包迁移、更新渠道切换、损坏的托管插件
容错、过时的插件依赖项清理、离线插件覆盖、插件更新行为和
Telegram 软件包 QA 使用同一个解析后的工件，同时避免让默认的
发布软件包门禁遍历每个已发布版本。

`last-stable-4` 解析为最新的四个已发布到 npm 的 OpenClaw
稳定版本。发布软件包验收将 `2026.4.23` 固定为首个插件更新
兼容性边界，将 `2026.5.2` 固定为插件架构变动边界，并将
`2026.4.15` 固定为较早的 2026.4.1x 已发布更新基线；解析器会
去除最新四个版本中已有的重复固定版本。如需详尽的已发布版本
更新迁移覆盖，请在单独的更新迁移工作流中使用 `all-since-2026.4.23`，
而不是使用完整发布 CI。如果还需要更广泛的手动抽样并包含旧版
日期前锚点，仍可使用 `release-history`。

选择多个已发布升级存续测试基线时，可复用的
Docker 工作流会将每个基线分片到各自的针对性运行器任务中。每个
基线分片仍会运行选定的场景集，但日志和工件按基线分别保留，
总耗时由最慢的分片决定，而不是由一个大型串行任务决定。

在发布前验证候选版本时，手动运行软件包配置：

```bash
gh workflow run package-acceptance.yml \
  --ref main \
  -f workflow_ref=main \
  -f source=npm \
  -f package_spec=openclaw@beta \
  -f suite_profile=package \
  -f published_upgrade_survivor_baselines="last-stable-4 2026.4.23 2026.5.2 2026.4.15" \
  -f published_upgrade_survivor_scenarios=reported-issues \
  -f telegram_mode=mock-openai
```

对于已发布的扩展稳定版金丝雀版本，设置
`package_spec=openclaw@extended-stable`。软件包验收会在 Docker 通道运行前将该
选择器解析为确切的 tarball。

当发布问题涉及 MCP 渠道、定时任务/子智能体清理、OpenAI Web 搜索或
OpenWebUI 时，使用 `suite_profile=product`。仅当需要完整的 Docker
发布路径覆盖时，才使用 `suite_profile=full`。

## 发布默认验证

对于候选发布版本，默认验证栈为：

1. `pnpm check:changed` 和 `pnpm test:changed`，用于源代码级回归。
2. `pnpm release:check`，用于软件包工件完整性。
3. 软件包验收 `package` 配置，或用于安装/更新/重启/插件
   契约的发布检查自定义软件包通道。
4. 跨操作系统发布检查，用于特定于操作系统的安装程序、新手引导和平台
   行为。
5. 仅当变更的表面涉及提供商或托管服务行为时，才运行实时测试套件。

在维护者机器上，除非明确执行本地验证，否则广泛的门禁和
Docker/软件包产品验证应在 Testbox 中运行。

## 旧版兼容性

兼容性宽容范围有限且有时间限制：

- 截至 `2026.4.25` 的软件包（包括 `2026.4.25-beta.*`）可以在
  软件包验收中容忍已发布软件包的元数据缺失。
- 已发布的 `2026.4.26` 软件包可以针对已发布的本地构建元数据戳
  文件发出警告。
- 后续软件包必须满足现代契约。同样的缺失会导致失败，而不是
  发出警告或跳过。

不要为这些旧形态添加新的启动迁移。添加或扩展 Doctor
修复，然后使用 `upgrade-survivor`、`published-upgrade-survivor` 或
`update-restart-auth` 进行验证；当更新命令负责重启时，使用后者。

## 添加覆盖范围

更改更新或插件行为时，应在能够因正确原因而失败的最低层添加测试覆盖：

- 纯路径或元数据逻辑：在源文件旁添加单元测试。
- 包清单或打包文件行为：添加 `package-dist-inventory` 或 tarball
  检查器测试。
- CLI 安装/更新行为：添加 Docker 通道断言或夹具。
- 已发布版本的迁移行为：添加 `published-upgrade-survivor` 场景。
- 由更新流程负责的重启行为：使用 `update-restart-auth`。
- 注册表/包来源行为：使用 `test:docker:plugins` 夹具或 ClawHub
  夹具服务器。
- 依赖项布局或清理行为：同时断言运行时执行和
  文件系统边界。npm 依赖项可能会提升到插件的
  托管 npm 项目中，因此测试应证明会扫描/清理该项目，
  而不是假定只有插件包本地的 `node_modules` 目录树。

默认情况下，新 Docker 夹具应保持封闭自足。除非测试目的在于验证实时注册表行为，
否则应使用本地夹具注册表和虚假包。

## 故障分类排查

从工件标识信息开始：

- 包验收 `resolve_package` 摘要：来源、版本、SHA-256 和
  工件名称。
- Docker 工件：`.artifacts/docker-tests/**/summary.json`、
  `failures.json`、通道日志和重新运行命令。
- 升级后保留项摘要：`.artifacts/upgrade-survivor/summary.json`，
  包括基线版本、候选版本、场景、各阶段耗时和
  配置方案覆盖情况。

与其重新运行整个发布总流程，不如使用同一包工件重新运行失败的确切通道。
