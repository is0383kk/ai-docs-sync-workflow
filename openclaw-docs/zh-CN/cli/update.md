---
read_when:
    - 你希望安全地更新源代码检出目录
    - 你正在调试 `openclaw update` 输出或选项
    - 你需要了解 `--update` 的简写行为
summary: '`openclaw update` 的 CLI 参考（相对安全的源代码更新 + Gateway 网关自动重启）'
title: 更新
x-i18n:
    generated_at: "2026-07-26T06:12:51Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: b46696f6b9cba5c318f870bcb6c5ea8e0652940968da2ad85e86709fe4c11146
    source_path: cli/update.md
    workflow: 16
---

# `openclaw update`

更新 OpenClaw，并在 stable/extended-stable/beta/dev 渠道之间切换。

如果通过 **npm/pnpm/bun** 安装（全局安装，无 git 元数据），
更新将遵循
[更新](/zh-CN/install/updating)中所述的包管理器流程。

## 用法

```bash
openclaw update
openclaw update status
openclaw update repair
openclaw update wizard
openclaw update --channel extended-stable
openclaw update --channel beta
openclaw update --channel dev
openclaw update --tag beta
openclaw update --tag main
openclaw update --dry-run
openclaw update --no-restart
openclaw update --yes
openclaw update --acknowledge-clawhub-risk
openclaw update --json
openclaw --update
```

`openclaw --update` 会重写为 `openclaw update`（适用于 shell 和
启动器脚本）。

## 选项

| 标志                                             | 说明                                                                                                                                                                                                                                                                                                                                  |
| ------------------------------------------------ | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `--no-restart`                                   | 成功更新后不重启 Gateway 网关服务。会执行重启的包管理器更新将在命令成功前，验证重启后的服务是否报告预期版本。                                                                                                                                                |
| `--channel <stable\|extended-stable\|beta\|dev>` | 设置更新渠道，并在核心更新成功后将其持久化。Extended-stable 仅适用于包安装。                                                                                                                                                                                                                                            |
| `--tag <dist-tag\|version\|spec>`                | 仅为本次更新覆盖包目标。它不能与有效的 `extended-stable` 渠道结合使用，因为该渠道必须使用经过验证的确切目标。对于其他包安装，`main` 映射到 `github:openclaw/openclaw#main`；GitHub/git 源规范会先打包到临时 tarball 中，再执行分阶段的全局 npm 安装。 |
| `--dry-run`                                      | 预览计划执行的操作（渠道/标签/目标/重启流程），而不写入配置、执行安装、同步插件或重启。                                                                                                                                                                                                                |
| `--json`                                         | 输出机器可读的 `UpdateRunResult` JSON。其中包括托管插件需要修复时的 `postUpdate.plugins.warnings`、beta 渠道插件回退详情，以及在更新后同步期间检测到 npm 插件工件漂移时的 `postUpdate.plugins.integrityDrifts`。                                                                 |
| `--timeout <seconds>`                            | 每个步骤的超时时间。默认值为 `1800`。                                                                                                                                                                                                                                                                                                            |
| `--yes`                                          | 跳过确认提示（例如降级确认）。                                                                                                                                                                                                                                                                              |
| `--acknowledge-clawhub-risk`                     | 允许更新后插件同步在出现社区 ClawHub 信任警告时继续，而不显示交互式提示。如果没有此选项，当 OpenClaw 无法提示时，将跳过有风险的社区版本并保持其不变。官方 ClawHub 包和内置插件源不受此提示限制。                                                     |

不存在 `--verbose` 标志。使用 `--dry-run` 预览计划执行的操作，
使用 `--json` 获取机器可读结果，使用 `openclaw update status --json`
仅获取渠道/可用性信息。Gateway 网关控制台详细程度（`--verbose`）和
文件日志级别（`logging.level: "debug"`/`"trace"`）是相互独立的设置；请参阅
[Gateway 网关日志](/zh-CN/gateway/logging)。

<Note>
在 Nix 模式（`OPENCLAW_NIX_MODE=1`）下，会禁用会产生变更的 `openclaw update` 运行。请改为更新此安装的 Nix 源或 flake 输入；对于 nix-openclaw，请使用以智能体为先的[快速开始](https://github.com/openclaw/nix-openclaw#quick-start)。`openclaw update status` 和 `openclaw update --dry-run` 仍为只读操作。
</Note>

<Warning>
降级需要确认，因为旧版本可能会破坏配置。
如果安装已将会话迁移到 SQLite，请先恢复已归档的旧版
转录工件，再启动基于文件存储的旧版本。请参阅
[Doctor：会话迁移到 SQLite 后进行降级](/zh-CN/cli/doctor#downgrading-after-session-sqlite-migration)。
</Warning>

## `update status`

显示当前更新渠道、git 标签/分支/SHA（仅限源代码检出），
以及更新可用性。

```bash
openclaw update status
openclaw update status --json
openclaw update status --timeout 10
```

| 标志                  | 默认值 | 说明                         |
| --------------------- | ------- | ----------------------------------- |
| `--json`              | `false` | 输出机器可读的状态 JSON。 |
| `--timeout <seconds>` | `3`     | 检查超时时间。                 |

对于 extended-stable 包安装，状态检查会执行与前台更新相同的公共选择器解析
和确切包验证。当已安装版本较新时，它可能会报告
`ahead of extended-stable`。JSON 失败结果
包含 `registry.reason`（`selector_missing`、`selector_query_failed`、
`exact_package_mismatch` 或 `unsupported_git_channel`）。

## `update repair`

当核心包已发生更改，但后续修复工作未能顺利完成时，
重新运行更新收尾流程。如果 `openclaw update` 已安装新的核心包，
但核心更新后的插件同步、托管 npm 插件元数据、注册表刷新或 Doctor 修复
未能收敛，这是受支持的恢复路径。

```bash
openclaw update repair
openclaw update repair --channel beta
openclaw update repair --acknowledge-clawhub-risk
openclaw update repair --json
```

| 标志                                             | 说明                                                                                                                                                                                                                                                         |
| ------------------------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `--channel <stable\|extended-stable\|beta\|dev>` | 在修复前持久化核心更新渠道。对于 extended-stable，遵循裸/default 或 `latest` 意图的符合条件的官方 npm 插件，会以确切的已安装核心版本为目标。Git 检出中的 extended-stable 修复会被拒绝，且不会更改配置。 |
| `--json`                                         | 输出机器可读的收尾 JSON。                                                                                                                                                                                                                           |
| `--timeout <seconds>`                            | 修复步骤的超时时间。默认值为 `1800`。                                                                                                                                                                                                                           |
| `--yes`                                          | 跳过确认提示。                                                                                                                                                                                                                                          |
| `--acknowledge-clawhub-risk`                     | 行为与 `openclaw update` 相同。                                                                                                                                                                                                                              |
| `--no-restart`                                   | 为保持一致性而接受此选项；修复流程从不重启 Gateway 网关。                                                                                                                                                                                                             |

`update repair` 会运行 `openclaw doctor --fix`，重新加载修复后的配置和
安装记录，为当前更新渠道同步受跟踪的插件，更新
托管的 npm 插件安装，修复缺失的已配置插件载荷，
刷新插件注册表，并写入已收敛的安装记录元数据。
它不会安装新的核心包，也不会重启 Gateway 网关。

## `update wizard`

通过交互式流程选择更新渠道，并确认随后是否重启
Gateway 网关（默认重启）。如果没有 git 检出，选择 `dev`
时会提供创建检出的选项。

| 标志                  | 默认值 | 说明                   |
| --------------------- | ------- | ----------------------------- |
| `--timeout <seconds>` | `1800`  | 每个更新步骤的超时时间。 |

## 工作原理

显式切换渠道（`--channel ...`）还会确保安装方式
保持一致：

- `dev` -> 确保存在 git 检出（默认为 `~/openclaw`，或
  在设置 `OPENCLAW_HOME` 时为 `$OPENCLAW_HOME/openclaw`；可使用
  `OPENCLAW_GIT_DIR` 覆盖），更新该检出，并从该
  检出安装全局 CLI。
- `stable` -> 使用 `latest` 从 npm 安装。
- `extended-stable` -> 解析公共 npm `extended-stable` 选择器，
  验证选中的确切包，并安装该确切版本。它
  不会回退到其他选择器，并且不允许用于 Git 检出。
- `beta` -> 优先使用 npm dist-tag `beta`；当 beta
  不存在或早于当前稳定版本时，回退到 `latest`。

### 重启交接

Gateway 网关核心自动更新程序（通过配置启用时）会在实时 Gateway 网关请求处理程序之外
启动 CLI 更新路径。控制平面的
`update.run` 包管理器更新和受监管的 git 检出更新使用
相同的托管服务交接机制，而不会在实时 Gateway 网关进程中替换包树或
重新构建 `dist/`：Gateway 网关会启动一个
分离的辅助进程并退出，该辅助进程随后从 Gateway 网关进程树之外运行 `openclaw update --yes --json`。
如果交接不可用，
`update.run` 会返回结构化响应，其中包含可安全手动运行的 shell 命令。

启用 `update.checkOnStart` 后，已存储的扩展稳定版选择会在启动时获得只读提示，并每 24 小时获得一次更新提示。这些检查绝不会应用更新、启动交接、重启 Gateway 网关、使用稳定版延迟/抖动或采用测试版轮询频率。仍支持显式前台更新、带有已存储 `update.channel: "extended-stable"` 的无参数前台更新、按需状态查询及其托管式 Gateway 网关交接。

安装本地托管式 Gateway 网关服务并启用重启后，包管理器和 Git 检出更新会先停止正在运行的服务，再替换软件包目录树或修改检出目录/构建输出。然后，更新程序会刷新服务元数据、重启服务并验证重启后的 Gateway 网关，之后才报告 `Gateway: restarted and verified.`。
包管理器更新还会验证重启后的 Gateway 网关报告了预期的软件包版本；Git 检出更新则会在重新构建后验证 Gateway 健康和服务就绪状态。

包管理器更新通常会继续使用托管服务中记录的 Node 二进制文件。如果该 Node 无法运行目标版本，但当前 CLI 使用的 Node 可以运行，并且已证实该服务属于正在更新的软件包，则启用了重启的更新会使用当前 Node 完成收尾，并将服务元数据改写为使用该运行时。`--no-restart` 无法修复服务元数据，因此遇到相同的运行时不匹配时，会在修改软件包之前停止。

在 macOS 上，更新后检查还会验证 LaunchAgent 已针对活动配置文件加载并正在运行，且配置的环回端口处于健康状态。如果 plist 已安装，但 launchd 未对其进行监管，OpenClaw 会自动重新引导 LaunchAgent，并重新运行健康状况/版本/渠道就绪检查（全新引导会直接加载 `RunAtLoad` 作业，因此恢复过程不会立即 `kickstart -k` 新生成的 Gateway 网关）。如果 Gateway 网关仍未恢复健康，命令将以非零状态退出，并输出重启日志路径以及重启、重新安装和软件包回滚说明。

如果无法执行重启，命令会输出 `Gateway: restart skipped (...)` 或 `Gateway: restart failed: ...`，并附带手动执行 `openclaw gateway restart` 的提示。
使用 `--no-restart` 时，软件包替换或 Git 重新构建仍会执行，但托管服务不会停止或重启，因此正在运行的 Gateway 网关会继续使用旧代码，直到你手动重启它。

### 控制平面响应结构

当 `update.run` 通过 Gateway 网关控制平面在包管理器安装或受监管的 Git 检出中运行时，处理程序会将交接启动与 Gateway 网关退出后继续进行的 CLI 更新分别报告：

- `ok: true`、`result.status: "skipped"`、
  `result.reason: "managed-service-handoff-started"` 和
  `handoff.status: "started"`：Gateway 网关已创建托管服务交接并安排自身重启，以便分离的辅助程序能够在实时服务进程之外运行
  `openclaw update --yes --json`。
- `ok: false`、`result.reason: "managed-service-handoff-unavailable"` 和
  `handoff.status: "unavailable"`：OpenClaw 无法找到用于安全交接的监管服务边界和持久服务标识（例如，systemd 交接需要 `OPENCLAW_SYSTEMD_UNIT` 单元标识，而不能只依赖环境中的 systemd 进程标记）。响应包含
  `handoff.command`，即需要从 Gateway 网关外部运行的 shell 命令。
- `ok: false`、`result.reason: "managed-service-handoff-failed"`：Gateway 网关尝试创建交接，但无法生成分离的辅助程序。

`sentinel` 载荷会在 Gateway 网关退出前写入，CLI 交接会在托管服务重启健康检查完成后更新同一个重启哨兵。交接期间，该哨兵可能包含 `stats.reason: "restart-health-pending"`，且没有成功续接；重启后的 Gateway 网关会轮询该哨兵，并且仅在 CLI 验证服务健康状况并用最终的 `ok` 结果改写哨兵后，才触发续接。
当该哨兵处于待处理或失败状态时，`openclaw status` 和 `openclaw status --all` 会显示一行 `Update restart`，而 `update.status` 会刷新并返回最新哨兵。

## Git 检出流程

### 渠道选择

- `stable`：检出最新的非测试版标签，然后执行构建和 Doctor。
- `beta`：优先选择最新的 `-beta` 标签；如果测试版不存在或版本更旧，则回退到最新的稳定版标签。
- `dev`：检出 `main`，然后获取并变基。
- `extended-stable`：Git 检出不支持此项；不会修改检出目录。

### 更新步骤

<Steps>
  <Step title="验证工作树是否干净">
    要求不存在未提交的更改。
  </Step>
  <Step title="切换渠道">
    切换到所选渠道（标签或分支）。
  </Step>
  <Step title="获取上游">
    仅限开发版。
  </Step>
  <Step title="预检构建（仅限开发版）">
    在临时工作树中运行 TypeScript 构建。如果分支尖端构建失败，则最多回退检查 10 个提交，以查找最新的可构建提交。设置 `OPENCLAW_UPDATE_PREFLIGHT_LINT=1` 后，还会在此预检期间运行 lint；lint 会以资源受限的串行模式运行，因为用户的更新主机通常比 CI 运行器配置更低。
  </Step>
  <Step title="变基">
    变基到所选提交（仅限开发版）。
  </Step>
  <Step title="安装依赖项">
    使用仓库的包管理器。对于 pnpm 检出，更新程序会按需引导 `pnpm`（先通过 `corepack`，然后使用临时 `npm install pnpm@11` 作为回退），而不是在 pnpm 工作区内运行 `npm run build`。如果 pnpm 引导仍然失败，更新程序会提前停止并报告包管理器专属错误，而不会尝试在检出目录中运行 `npm run build`。
  </Step>
  <Step title="构建 Control UI">
    构建 Gateway 网关和 Control UI。
  </Step>
  <Step title="运行 Doctor">
    `openclaw doctor` 作为最终的安全更新检查运行。
  </Step>
  <Step title="同步插件">
    将插件同步到活动渠道。开发版使用内置插件；稳定版和测试版使用 npm。更新受跟踪的插件安装。
  </Step>
</Steps>

### 插件同步详情

在测试版渠道中，遵循默认/最新版本线的受跟踪 npm 和 ClawHub 插件安装会先尝试插件 `@beta` 版本。如果插件没有测试版，OpenClaw 会回退到已记录的默认/最新规范并报告警告。对于 npm 插件，如果测试版软件包存在但未通过安装验证，OpenClaw 也会回退。这些回退警告不会导致核心更新失败。绝不会改写精确版本和显式标签。

<Warning>
如果精确固定的 npm 插件更新解析出的工件完整性与存储的安装记录不同，`openclaw update` 会中止该插件工件更新，而不会安装它。只有在确认信任新工件后，才能显式重新安装或更新该插件。
</Warning>

<Note>
如果更新后的插件同步失败仅限于某个托管插件，并且同步路径能够绕过该故障（例如，非必要插件的 npm 注册表无法访问），则核心更新成功后会将其报告为警告。JSON 结果会保留顶层更新 `status: "ok"`，并报告 `postUpdate.plugins.status: "warning"`，其中包含 `openclaw update repair` 和 `openclaw plugins inspect <id> --runtime --json` 指引。意外的更新程序或同步异常仍会使更新结果失败。修复插件安装或更新错误，然后重新运行 `openclaw update repair`。如果失败的更新导致托管插件不可用，OpenClaw 会禁用其运行时条目并重置活动槽位，但不会更改操作员编写的 `plugins.allow` 或 `plugins.deny` 策略。

完成逐插件同步步骤后，`openclaw update` 会在 Gateway 网关重启前运行强制性的**核心更新后收敛**过程：修复缺失的已配置插件载荷，验证磁盘上每条_活动_受跟踪安装记录，并静态验证其 `package.json` 可解析（以及任何显式声明的 `main` 均存在）。此过程中的失败以及无效的配置快照会返回 `postUpdate.plugins.status: "error"`，并将顶层更新 `status` 更改为 `"error"`，因此 `openclaw update` 会以非零状态退出，并且_不会_使用未经验证的插件集重启 Gateway 网关。错误包含结构化的 `postUpdate.plugins.warnings[].guidance` 行，指向 `openclaw update repair` 和 `openclaw plugins inspect <id> --runtime --json`。此处会跳过已禁用的插件条目，以及并非受信任来源关联的官方同步目标的记录（与缺失载荷检查使用的 `skipDisabledPlugins` 策略一致），因此陈旧的已禁用插件记录不会阻止原本有效的更新。

更新后的 Gateway 网关启动时，插件加载仅执行验证：启动过程不会运行包管理器或修改依赖项树。包管理器 `update.run` 重启会交由 CLI 托管服务路径处理，因此软件包交换会在旧 Gateway 网关进程之外进行，而服务健康检查决定是否可以将更新报告为已完成。
</Note>

扩展稳定版核心更新成功后，核心更新后的插件完整性和收敛会以符合条件的官方 npm 插件为目标，并使用已安装核心的精确版本。对于默认/`latest` 意图，OpenClaw 不会查询插件 `@extended-stable`，也不会回退到 npm `latest`；它会根据已安装的核心推导软件包版本。显式版本固定、显式的非 `latest` 标签、第三方软件包和非 npm 来源会保留其现有意图。

对于包管理器安装，`openclaw update` 会在调用包管理器之前解析目标软件包版本。npm 全局安装采用暂存式安装：OpenClaw 将新软件包安装到临时 npm 前缀中，让候选软件包在 `preinstall` 期间验证主机 Node 版本，并在那里验证打包的 `dist` 清单。在 `preinstall` 成功前，打包完成守卫始终位于该清单之外，因此跳过生命周期脚本的包管理器也会在激活前停止。在 npm 12 及更高版本中，更新程序仅批准候选 OpenClaw 的生命周期脚本；传递依赖项脚本仍会被阻止。然后，OpenClaw 会将干净的软件包目录树交换到实际的全局前缀中。如果验证失败，则不会从可疑目录树运行更新后 Doctor、插件同步和重启操作。即使已安装版本已与目标匹配，该命令仍会刷新全局软件包安装，然后运行插件同步、核心命令补全刷新和重启操作。这会使打包的附属组件和渠道所有的插件记录与已安装的 OpenClaw 构建保持一致，同时将完整的插件命令补全重新构建留给显式的 `openclaw completion --write-state` 运行。

## 相关内容

- `openclaw doctor`（在 Git 检出中会先提议运行更新）
- [开发渠道](/zh-CN/install/development-channels)
- [更新](/zh-CN/install/updating)
- [CLI 参考](/zh-CN/cli)
