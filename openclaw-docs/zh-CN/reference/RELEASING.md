---
doc-schema-version: 1
read_when:
    - 查找公开发布渠道的定义
    - 运行发布验证或软件包验收
    - 查找版本命名和发布节奏
summary: 发布通道、操作员检查清单、验证环境、版本命名和发布节奏
title: 发布策略
x-i18n:
    generated_at: "2026-07-26T06:59:01Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: de2429f039bb42deabdcfe280b7d91afac3bae3dc24714203ab7a67672dcc10c
    source_path: reference/RELEASING.md
    workflow: 16
---

OpenClaw 提供四个面向用户的更新渠道：

- stable：npm 上已晋升的常规版本 `latest`
- extended-stable：npm 上前一个已结束月份的 `.33+` 维护线
  `extended-stable`
- beta：npm 上的预发布标签 `beta`
- dev：`main` 的动态最新提交

扩展稳定版发布前一个月份的 Gateway 网关、官方 npm 插件和
Docker 镜像，但不会移动常规的 `latest` 或 `main` 选择器。

Tideclaw alpha 构建是独立的内部预发布轨道（npm dist-tag `alpha`），详见 [NPM 工作流输入](#npm-workflow-inputs)和[发布测试箱](#release-test-boxes)。

## 版本命名

- 每月 Gateway 网关扩展稳定版版本：`YYYY.M.PATCH`，包含 `PATCH >= 33`，git 标签为 `vYYYY.M.PATCH`
- 每日/常规正式版本：`YYYY.M.PATCH`，包含 `PATCH < 33`，git 标签为 `vYYYY.M.PATCH`
- 常规回退修正版版本：`YYYY.M.PATCH-N`，git 标签为 `vYYYY.M.PATCH-N`
- Beta 预发布版本：`YYYY.M.PATCH-beta.N`，git 标签为 `vYYYY.M.PATCH-beta.N`
- Alpha 预发布版本：`YYYY.M.PATCH-alpha.N`，git 标签为 `vYYYY.M.PATCH-alpha.N`
- 月份或补丁号绝不补零
- `PATCH` 是按顺序递增的每月发布列车编号，而不是日历日期。常规正式版和 beta 版本会推进当前发布列车；仅 alpha 标签绝不会占用或推进 beta/常规补丁号，因此选择 beta 或常规发布列车时，应忽略补丁号更高的旧版仅 alpha 标签。
- Alpha/夜间构建使用下一个尚未发布的补丁列车，重复构建时仅递增 `alpha.N`。该补丁发布 beta 后，新的 alpha 构建将移至后续补丁。
- npm 版本不可变：绝不删除、重新发布或重复使用已发布的标签。应改为发布下一个预发布编号或下一个月度补丁。
- `latest` 继续跟随当前常规/每日 npm 发布线；`beta` 是当前 beta 安装目标
- `extended-stable` 表示受支持的前一个月份 Gateway 网关发行版，从补丁 `33` 开始；补丁 `34` 及更高版本是该月度发布线的维护版本
- 常规正式版和常规修正版默认发布到 npm `beta`；发布操作员可以明确指定 `latest`，也可以稍后晋升经过审查的 beta 构建
- Gateway 网关扩展稳定版以同一个精确版本发布核心、所有可发布到 npm 的官方插件
  及其 Docker 镜像；请参阅下方专用工作流。
- 每个常规正式版都会同时发布 npm 软件包、macOS 应用、已签名的独立 Android APK 和已签名的 Windows Hub 安装程序。Beta 版本通常先验证并发布 npm/软件包路径；除非明确要求，否则原生应用的构建、签名、公证和晋升仅用于常规正式版。

## 发布节奏

- 发布流程先发布 beta；只有最新 beta 通过验证后，才会发布 stable
- 维护者通常从当前 `main` 创建的 `release/YYYY.M.PATCH` 分支发布版本，以免发布验证和修复阻塞 `main` 上的新开发
- 如果 beta 标签已推送或发布但需要修复，维护者会发布下一个 `-beta.N` 标签，而不是删除或重新创建旧标签
- 详细发布过程、审批、凭据和恢复说明仅供维护者使用

## 每月 Gateway 网关扩展稳定版发布

对于已结束的月份 `YYYY.M`，创建 `extended-stable/YYYY.M.33`，并从该分支发布
`.33+`。标签、分支、检出、软件包版本、预检和
验证必须指向同一个提交。在 `.33` 之前，受保护的 `main` 必须包含
后续月份中补丁号低于 `33` 的正式版本；之后的维护补丁仍然
符合条件。

### 准备并稳定候选版本

审计尚未审计的主线范围，协调私有安全工作，批准
有界的向后移植集合，并通过一个协调一致的 PR 合入。不要直接推送到规范
分支。

在规范分支上设置 `YYYY.M.P`，运行 `pnpm release:prep`，并要求
所有可发布的官方插件均使用该版本。根据已批准的台账，
生成并提交完整的 `## YYYY.M.P` 章节，其中包含 `### Highlights`、
`### Changes` 和 `### Fixes`；对于等效的向后移植，应引用原始已合并的 `main` PR。
预检会拒绝缺失或为空的章节。

完整移植当前 main 的 Docker 发布渠道单元：工作流、晋升器、
策略、共享分类器、测试和工作流验证。GitHub 会从带标签的提交中加载标签
工作流；不完整的副本可能会在构建后失败，或移动常规别名。运行针对性检查。

冻结完整的分支末端 SHA。添加标签前，对其确切的 npm 字节执行预检，
并针对该 SHA 运行完整发布验证：

```bash
RELEASE_SHA="$(git rev-parse HEAD)"

gh workflow run openclaw-npm-release.yml \
  --ref extended-stable/YYYY.M.33 \
  -f tag="$RELEASE_SHA" \
  -f preflight_only=true \
  -f npm_dist_tag=extended-stable

gh workflow run full-release-validation.yml \
  --ref extended-stable/YYYY.M.33 \
  -f ref=extended-stable/YYYY.M.33 \
  -f release_profile=stable
```

SHA 形式仅用于预检。在规范分支上运行验证；发布操作会绑定其工作流引用、
头部/目标 SHA、运行 ID 和尝试次数。保存两个 ID 和成功的 `run_attempt`；
拒绝 `release-ci/*` 证据。

编辑前先对失败分类：

- 产品：合入另一个已批准的向后移植 PR。
- 冻结目标工具：仅向后移植最小的兼容性修复，
  并在保持旧产品不变的情况下进行测试。
- 提供商、审批、运行器或服务：保持候选版本不变，并使用
  有界重试路径。

任何分支变更都会使两项门禁失效。两者通过后，要求分支末端仍
等于 `RELEASE_SHA`，然后推送已签名的 `vYYYY.M.P`。后续变更需要使用下一个
补丁；绝不移动或删除标签。推送该标签会启动 `Docker Release`。

### 发布 npm 软件包

从同一个 SHA 发布所有可发布到 npm 的官方插件，并保存
成功的运行 ID：

```bash
RELEASE_SHA="$(git rev-parse HEAD)"
gh workflow run plugin-npm-release.yml \
  --ref extended-stable/YYYY.M.33 \
  -f publish_scope=all-publishable \
  -f ref="$RELEASE_SHA" \
  -f npm_dist_tag=extended-stable
```

该工作流涵盖所有 `all-publishable` 软件包，包括未变更的软件包，
并验证每个确切版本和选择器。重新运行会复用已发布的版本。

然后使用全部三个已保存的运行标识发布准备好的核心 tarball：

```bash
gh workflow run openclaw-npm-release.yml \
  --ref extended-stable/YYYY.M.33 \
  -f tag=vYYYY.M.P \
  -f preflight_only=false \
  -f npm_dist_tag=extended-stable \
  -f preflight_run_id=<npm-preflight-run-id> \
  -f full_release_validation_run_id=<full-validation-run-id> \
  -f full_release_validation_run_attempt=<full-validation-run-attempt> \
  -f plugin_npm_run_id=<plugin-npm-run-id>
```

仅在非生产演练中，向预检和发布添加
`-f bypass_extended_stable_guard=true`。它只会绕过
月份门禁，绝不会绕过规范引用、SHA/标签/版本相等性、来源证明、
审批或回读检查。绝不要在生产环境中使用它。

### 验证和恢复

在单独、干净的当前 `main` 检出中运行以下命令，而不是在冻结分支中运行：

```bash
node --import tsx scripts/openclaw-npm-postpublish-verify.ts YYYY.M.P
npm view openclaw@YYYY.M.P version --userconfig "$(mktemp)"
npm view openclaw@extended-stable version --userconfig "$(mktemp)"
```

要求规范分支具有签名和 npm 来源证明，并且发布、
预检和 tarball 摘要均绑定到发布 SHA。两个命令都必须
返回 `YYYY.M.P`。验证每个准备好的核心软件包和 `all-publishable`
官方插件的确切版本及选择器。

如果只有根选择器失败，请使用工作流摘要中生成并打印的
`npm dist-tag add openclaw@YYYY.M.P extended-stable` 修复命令。
通过已批准且凭据隔离的工具修复现有插件或其他准备好的核心选择器；
OIDC 来源无法修改这些选择器。绝不重新发布不可变版本。

要求 `Docker Release` 验证 GHCR 和 Docker Hub 中确切的默认、slim、browser 和架构
镜像，包括证明和平台版本。它必须仅按
摘要推进 `extended-stable`、`extended-stable-slim` 和 `extended-stable-browser`；
常规别名保持不变，并拒绝自动回滚。

如需修复别名，请从当前 `main` 使用该标签运行需审批的 `Docker Channel Promotion`。
它会重复摘要、证明和平台检查，允许明确回滚，
且绝不会重新构建镜像。

Slack、Discord 和 Codex 是最初记录的支持界面，而不是
发布允许列表：所有可发布到 npm 的官方插件都会发布。只有常规
检查清单负责 beta/`latest`、GitHub Releases、ClawHub、原生应用、移动端、
网站和私有 dist-tag；不要为此 Gateway 网关路径运行这些步骤。

## 常规发布操作员检查清单

此检查清单是发布流程的公开形式。私有凭据、签名、公证、dist-tag 恢复和紧急回滚细节保留在仅供维护者使用的发布运行手册中。

1. 从当前 `main` 开始：拉取最新内容，确认目标提交已推送，并确认 `main` CI 状态足够良好，可以从中创建分支。
2. 从该提交创建 `release/YYYY.M.PATCH`。向后移植是可选的；仅应用操作员选择的集合。更新所有必需位置的版本，运行 `pnpm release:prep`，完成发布修复和必需的向前移植，并审查 `src/plugins/compat/registry.ts` 和 `src/commands/doctor/shared/deprecation-compat.ts`。
3. 将产品完整、尚未更新变更日志的提交冻结为 **Code SHA**。运行确定性的源代码预检，然后使用 `node scripts/full-release-validation-at-sha.mjs --sha <code-sha> --target-ref release/YYYY.M.PATCH`。这样会固定受信任的工作流工具，同时让完整的 Vitest、Docker、QA、软件包和性能矩阵以确切的 Code SHA 为目标。
4. 编辑前先对失败分类。产品/代码失败会产生新的 Code SHA，并要求该 SHA 通过完整验证。工作流、测试框架、凭据、审批或基础设施失败应在其所属界面中修复，并针对同一个 Code SHA 重新运行。
5. 只有 Code SHA 通过后，才能根据自上一个可达的已发布标签以来合并的 PR 和直接提交，生成顶部的 `CHANGELOG.md` 章节。条目应面向用户且去重。如果存在分叉的已发布标签，或后续向前移植重新关联了已发布的 PR，请将其明确传入 `--shipped-ref`。
6. 仅提交 `CHANGELOG.md`。此提交即为 **Release SHA**。从 Code SHA 到 Release SHA 的完整差异必须恰好为 `CHANGELOG.md`；如有任何其他路径发生变更，发布流程将返回步骤 2。
7. 为 Release SHA 运行固定到 SHA 的完整发布验证，并启用证据复用。轻量级父任务必须记录 `changelog-only-release-v1`，指向已通过验证的 Code SHA，并且不分派任何产品子任务通道。此过程会复用产品证据，但不会复用软件包字节。
8. 针对 Release SHA/标签运行 `OpenClaw NPM Release`，并设置 `preflight_only=true`。保存成功的 `preflight_run_id`。此操作会构建并检查包含最终变更日志的确切软件包字节。
9. 为 Release SHA 添加标签，然后使用成功的 Release-SHA 验证父任务和 npm 预检运行候选版本辅助工具，而不是再次分派任一任务：

   ```bash
   pnpm release:candidate -- \
     --tag vYYYY.M.PATCH-beta.N \
     --full-release-run <release-sha-validation-run-id> \
     --npm-preflight-run <preflight-run-id> \
     --skip-dispatch
   ```

   对于稳定版，还要传入 `--windows-node-tag vX.Y.Z`。该辅助程序会验证发布说明来源、npm 预检字节、Parallels 安装/更新证明、Telegram 包证明和插件发布计划，然后输出发布命令。

   `OpenClaw Release Publish` 会将选定或所有可发布的插件包并行分发到 npm，并将同一组包分发到 ClawHub；插件 npm 发布成功后，再使用匹配的 dist-tag 推送准备好的 OpenClaw npm 预检工件。发布检出仍作为产品/数据根目录，而规划和最终验证则从完全一致且可信的工作流源检出中执行，确保较旧的发布提交无法悄然使用过时的发布工具。在启动任何发布子任务之前，它会渲染并缓存完全一致的 GitHub 发布正文。当完整且匹配的 `CHANGELOG.md` 章节同时符合 GitHub 的 125,000 字符限制和渲染器匹配的 125,000 字节安全上限时，页面会包含该完全一致的 `## YYYY.M.PATCH` 章节及其标题。当源章节无法容纳时，页面会保留完全一致的分组编辑说明，并将过大的贡献记录替换为指向标签固定的 `CHANGELOG.md` 中完整记录的稳定链接；绝不会发布不完整的记录或截断的项目符号。工作流会在添加 `### Release verification` 之前选择完整或精简正文；如果证明尾部会超出限制，它会保留规范正文，改为依赖附加的不可变证据。发布到 npm `latest` 的稳定版本会成为 GitHub 最新发布，而保留在 npm `beta` 上的稳定维护版本则会使用 GitHub `latest=false` 创建。工作流还会将预检依赖证据、完整验证清单和发布后注册表验证证据上传到 GitHub 发布，以供发布后事件响应使用。它会立即输出子任务运行 ID，自动批准工作流令牌有权批准的发布环境门禁，使用日志尾部汇总失败的子任务作业，预先创建 GitHub 发布草稿页面，并在发布 OpenClaw npm 包的同时并发推送 Windows 和 Android 资产；这些阶段成功后，它会完成发布页面和依赖证据，发布 OpenClaw npm 包时还会等待 ClawHub 完成；随后运行可信 main 上的 beta 验证器，并为 GitHub 发布、npm 包、选定的插件 npm 包、选定的 ClawHub 包、子工作流运行 ID 和可选的 NPM Telegram 运行 ID 上传发布后证据。ClawHub 引导验证器要求完全一致的可信 main 工作流路径和 SHA、生产者和终止运行尝试、发布 SHA、请求的包集合、不可变包工件元组以及终止注册表回读工件；不接受成功的旧版发布引用运行。

   然后针对已发布的 `openclaw@YYYY.M.PATCH-beta.N` 或 `openclaw@beta` 包运行发布后包验收。如果已推送或已发布的预发布版本需要修复，请发布下一个匹配的预发布编号；绝不要删除或重写旧版本。

10. 发布尝试失败时，除非失败证明产品或变更日志存在缺陷，否则保持 Release SHA 不变。继续使用已成功的不可变子任务和工件；绝不要重新构建或重新发布已经成功的包版本。
11. 对于稳定版，只有经过审核的 beta 或候选发布版本具备所需验证证据后才能继续。稳定版 npm 发布也通过 `OpenClaw Release Publish` 进行，并通过 `preflight_run_id` 复用成功的预检工件。稳定版 macOS 发布就绪还要求 `main` 上存在已打包的 `.zip`、`.dmg`、`.dSYM.zip` 和已更新的 `appcast.xml`；macOS 发布工作流会在验证发布资产后，自动将已签名的 appcast 发布到公共 `main`，如果分支保护阻止直接推送，则会创建或更新 appcast PR。稳定版 Windows Hub 就绪要求 OpenClaw GitHub 发布中存在已签名的 `OpenClawCompanion-Setup-x64.exe`、`OpenClawCompanion-Setup-arm64.exe` 和 `OpenClawCompanion-SHA256SUMS.txt` 资产。将完全一致的已签名 `openclaw/openclaw-windows-node` 发布标签作为 `windows_node_tag` 传入，并将其经候选版本批准的安装程序摘要映射作为 `windows_node_installer_digests` 传入；`OpenClaw Release Publish` 会保留发布草稿、分发 `Windows Node Release`，并在发布前验证全部三个资产。
12. 发布后，运行 npm 发布后验证器；需要发布后渠道证明时，运行可选的独立已发布 npm Telegram E2E；必要时推送 dist-tag；验证生成的 GitHub 发布页面；执行发布公告步骤；然后完成[稳定版 main 收尾](#stable-main-closeout)，之后才能将稳定版发布视为完成。

## 稳定版 main 收尾

在 `main` 包含实际已发布的发布状态之前，稳定版发布尚未完成。

1. 从最新的 `main` 开始。以它为基准审计 `release/YYYY.M.PATCH`，并前向移植 `main` 中缺失的实际修复。不要盲目地将仅适用于发布的兼容性、测试或验证适配器合并到更新的 `main` 中。
2. 对于正常路径，将 `main` 设置为已发布的稳定版本。如果 `main` 已推进到更晚的 OpenClaw 稳定 CalVer，延迟的收尾可以使用它；不要仅为完成上一个发布而降级已经启动的发布序列。验证器仍要求完全一致的已发布变更日志章节和 appcast 条目，并记录实际的 `main` 版本和 SHA。根版本发生任何更改后运行 `pnpm release:prep`，然后运行 `pnpm deps:shrinkwrap:generate`。
3. 使 `CHANGELOG.md` 在 `main` 上的 `## YYYY.M.PATCH` 章节与已打标签的发布分支完全一致。如果 Mac 发布包含稳定版 `appcast.xml` 更新，也要将其纳入。
4. 在操作员明确启动该发布序列之前，不要向 `main` 添加 `YYYY.M.PATCH+1`、beta 版本或空的未来变更日志章节。
5. 运行 `pnpm release:generated:check`、`pnpm deps:shrinkwrap:check` 和 `OPENCLAW_TESTBOX=1 pnpm check:changed`。推送后，验证 `origin/main` 包含已发布版本和变更日志，然后才能将稳定版发布视为完成。
6. 每次进行私有回滚演练后，保持仓库变量 `RELEASE_ROLLBACK_DRILL_ID` 和 `RELEASE_ROLLBACK_DRILL_DATE` 为最新状态。

`OpenClaw Stable Main Closeout` 从稳定版发布后包含已发布版本、变更日志和 appcast 的 `main` 推送开始。它读取不可变的发布后证据，将已发布标签绑定到其完整发布验证和发布运行，然后验证稳定版 main 状态、发布、强制稳定版浸泡测试和阻塞性性能证据。它会将不可变的收尾清单及校验和附加到 GitHub 发布。自动推送触发器会跳过早于不可变发布后证据的旧版发布，并且绝不会将这种跳过视为已完成收尾。

完整收尾要求同时存在两个资产及匹配的校验和。部分清单会重放其中记录的 `main` SHA 和回滚演练，以重新生成完全相同的字节，然后附加缺失的校验和；无效的组合或只有校验和而没有清单的情况仍会阻塞。缺少回滚演练仓库变量的推送触发运行会跳过执行，但不会完成收尾；缺失或超过 90 天的演练记录仍会阻塞手动的证据支持收尾。私有恢复命令保留在仅限维护者使用的运行手册中。仅使用手动分发来修复或重放有证据支持的稳定版收尾。

如果 Release Publish 父任务仅在附加不可变 npm/插件证据后失败，请先修复并发布所有稳定版平台资产。然后维护者可以使用 `allow_failed_publish_recovery=true` 手动分发收尾；该模式仅接受已完成但失败的父任务，并且除常规 macOS/appcast 检查外，还要求完全一致的 Android 和 Windows 资产契约、GitHub SHA-256 摘要、校验和验证、Android 来源证明，以及由父任务分发且成功的 Windows 推送，其中 Authenticode 检查和候选版本批准的摘要必须与已发布的安装程序匹配。自动推送收尾绝不会启用此恢复模式。

只有当纠正标签与基础稳定版标签解析到同一源提交时，旧版回退纠正标签才能复用基础包证据。其 Android 发布会复用基础标签已验证的 APK，并添加纠正标签的来源证明。源不同的纠正版本必须发布并验证自己的包证据，并使用更高的 Android `versionCode`。

## 发布预检

- 在发布预检之前运行 `pnpm check:test-types`，以确保测试 TypeScript 在更快的本地 `pnpm check` 门禁之外仍有覆盖。
- 在发布预检之前运行 `pnpm check:architecture`，以确保更广泛的导入循环和架构边界检查在更快的本地门禁之外保持通过。
- 在 `pnpm release:check` 之前运行 `pnpm build && pnpm ui:build`，以确保预期的 `dist/*` 发布工件和 Control UI 包已存在，可供打包验证步骤使用。
- 在根版本升级之后、打标签之前运行 `pnpm release:prep`。它会运行所有在版本、配置或 API 更改后经常产生漂移的确定性发布生成器：插件版本、npm shrinkwrap、插件清单、基础配置模式、内置渠道配置元数据、配置文档基线、插件 SDK 导出、插件 SDK API 契约清单以及 Control UI 语言区域包。它还会阻塞，直到原生应用翻译和平台生成的语言区域资源与源清单一致；如果存在滞后，请在冻结 Code SHA 之前等待或分发 `Native App Locale Refresh`。`pnpm release:check` 会以检查模式重新运行这些防护措施（包括严格语言区域门禁和插件 SDK 表面预算），并在运行包发布检查之前，一次性报告所有生成内容漂移失败。
- 默认情况下，插件版本同步会将可发布的 `@openclaw/ai` 运行时包、官方插件包版本和现有的 `openclaw.compat.pluginApi` 下限更新为 OpenClaw 发布版本。应将该字段视为插件 SDK/运行时 API 下限，而不只是包版本的副本：对于有意保持与较旧 OpenClaw 主机兼容的仅插件发布，请将下限保持为受支持的最旧主机 API，并在插件发布证明中记录该选择。
- 在发布批准之前运行手动 `Full Release Validation` 工作流，以从单一入口启动所有预发布测试环境。它接受分支、标签或完整提交 SHA，分发手动 `CI`，并分发 `OpenClaw Release Checks`，用于安装冒烟测试、包验收、跨操作系统包检查、QA Lab 一致性、Matrix 和 Telegram 测试通道。稳定版和完整运行始终包含详尽的实时/E2E 测试和 Docker 发布路径浸泡测试；`run_release_soak=true` 保留用于显式 beta 浸泡测试。在候选版本验证期间，包验收会提供规范的包 Telegram E2E，从而避免第二个并发实时轮询器。

  发布 beta 后提供 `release_package_spec`，以便在各项发布检查、包验收和包 Telegram E2E 中复用已发布的 npm 包，而无需重新构建发布 tarball。仅当 Telegram 应使用与其余发布验证不同的已发布包时，才提供 `npm_telegram_package_spec`。当包验收应使用与发布包规范不同的已发布包时，提供 `package_acceptance_package_spec`。当发布证据报告应证明验证与已发布的 npm 包一致，但不强制执行 Telegram E2E 时，提供 `evidence_package_spec`。

  ```bash
  node scripts/full-release-validation-at-sha.mjs \
    --sha <code-sha> \
    --target-ref release/YYYY.M.PATCH
  ```

- 如果希望在发布工作继续进行的同时，为候选包获取旁路证明，请运行手动 `Package Acceptance` 工作流。使用 `source=npm` 指定 `openclaw@beta`、`openclaw@latest` 或确切的发布版本；使用 `source=ref`，通过当前 `workflow_ref` 测试框架打包受信任的 `package_ref` 分支/标签/SHA；使用 `source=url` 指定具有必需 SHA-256 和严格公共 URL 策略的公共 HTTPS tarball；使用 `source=trusted-url` 指定采用必需 `trusted_source_id` 和 SHA-256 的具名可信来源策略；或使用 `source=artifact` 指定由另一次 GitHub Actions 运行上传的 tarball。

  该工作流将候选版本解析为 `package-under-test`，针对该 tarball 复用 Docker E2E 发布调度器，并可使用 `telegram_mode=mock-openai` 或 `telegram_mode=live-frontier` 针对同一 tarball 运行 Telegram QA。当所选 Docker 通道包含 `published-upgrade-survivor` 时，包工件即为候选版本，而 `published_upgrade_survivor_baseline` 用于选择已发布的基线。`update-restart-auth` 将候选包同时用作已安装的 CLI 和待测试包，以便测试候选更新命令的托管重启路径。

  示例：

  ```bash
  gh workflow run package-acceptance.yml --ref main -f workflow_ref=main -f source=npm -f package_spec=openclaw@beta -f suite_profile=product -f published_upgrade_survivor_baseline=openclaw@2026.4.26 -f telegram_mode=mock-openai
  ```

  常用配置：
  - `smoke`：安装/渠道/智能体、Gateway 网关网络和配置重新加载通道
  - `package`：不含 OpenWebUI 或实时 ClawHub 的工件原生包/更新/重启/插件通道
  - `product`：包配置，以及 MCP 渠道、定时任务/子智能体清理、OpenAI Web 搜索和 OpenWebUI
  - `full`：包含 OpenWebUI 的 Docker 发布路径分块
  - `custom`：用于聚焦重新运行的精确 `docker_lanes` 选择

- 如果只需要为候选发布版本提供确定性的常规 CI 覆盖，请直接运行手动 `CI` 工作流。手动 CI 调度会绕过变更范围限定，并强制运行 Linux Node 分片、内置插件分片、插件和渠道契约分片、Node 22 兼容性、`check-*`、`check-additional-*`、构建工件冒烟检查、文档检查、Python Skills、Windows、macOS 和 Control UI 国际化通道。独立手动 CI 仅在使用 `include_android=true` 调度时运行 Android；`Full Release Validation` 会将该输入传递给其 CI 子工作流。

  ```bash
  gh workflow run ci.yml --ref release/YYYY.M.PATCH -f include_android=true
  ```

- 验证发布遥测时，运行 `pnpm qa:otel:smoke`。它通过本地 OTLP/HTTP 接收器运行 QA-lab，并验证跟踪、指标和日志导出，以及受限的跟踪属性和内容/标识符脱敏，无需 Opik、Langfuse 或其他外部收集器。
- 验证收集器兼容性时，运行 `pnpm qa:otel:collector-smoke`。它先将同一 QA-lab OTLP 导出路由到真实的 OpenTelemetry Collector Docker 容器，再执行本地接收器断言。
- 验证受保护的 Prometheus 抓取时，运行 `pnpm qa:prometheus:smoke`。它会运行 QA-lab，拒绝未经身份验证的抓取，并验证发布关键指标系列中不包含提示词内容、原始标识符、身份验证令牌和本地路径。
- 运行 `pnpm qa:observability:smoke`，依次执行源代码检出的 OpenTelemetry 和 Prometheus 冒烟通道。
- 在每次带标签发布之前运行 `pnpm release:check`。
- `OpenClaw NPM Release` 预检会在打包 npm tarball 之前生成依赖项发布证据。npm 公告漏洞门禁会阻止发布。传递性清单风险、依赖项所有权/安装范围和依赖项变更报告仅作为发布证据。依赖项变更报告会将候选发布版本与此前可访问的发布标签进行比较。预检会将依赖项证据上传为 `openclaw-release-dependency-evidence-<tag>`，并将其嵌入准备好的 npm 预检工件内的 `dependency-evidence/` 下。实际发布路径会复用该预检工件，然后将同一证据作为 `openclaw-<version>-dependency-evidence.zip` 附加到 GitHub 发布中。
- 标签存在后，运行 `OpenClaw Release Publish` 以执行会产生变更的发布流程。从受信任的 `main` 调度常规 beta 和稳定版发布；发布标签仍会选择确切的目标提交，并且可能指向 `release/YYYY.M.PATCH`。Tideclaw alpha 发布仍在其对应的 alpha 分支上进行。传入成功的 OpenClaw npm `preflight_run_id`、成功的 `full_release_validation_run_id` 和确切的 `full_release_validation_run_attempt`，并保留默认插件发布范围 `all-publishable`，除非有意执行聚焦修复。该工作流会依次执行插件 npm 发布、插件 ClawHub 发布和 OpenClaw npm 发布，确保核心包不会在其外置插件之前发布；Windows 和 Android 推广会与针对草稿发布页面的核心 npm 发布并行运行。发布重新运行可从中断处继续：如果核心 npm 版本已发布，则工作流在证明注册表 tarball 与标签的预检工件一致后，会跳过核心调度；当发布已包含经过验证的工件契约时，会跳过 Windows/Android 推广，因此重试只会重新执行失败的阶段。仅针对插件的聚焦修复需要 `plugin_publish_scope=selected` 和非空插件列表。仅插件的 `all-publishable` 运行需要完整且不可变的预检和 Full Release Validation 证据；不接受部分证据。
- 稳定版 `OpenClaw Release Publish` 要求在对应的非预发布 `openclaw/openclaw-windows-node` 发布存在后提供确切的 `windows_node_tag`，以及候选版本已获批准的 `windows_node_installer_digests` 映射。在调度任何发布子工作流之前，它会验证源发布已发布、不是预发布版本、包含所需的 x64/ARM64 安装程序，并且仍与该已批准映射一致。随后，它会在 OpenClaw 发布仍为草稿时调度 `Windows Node Release`，并原样携带固定的安装程序摘要映射。子工作流会从该确切标签下载已签名的 Windows Hub 安装程序，将其与固定摘要进行匹配，在 Windows 运行器上验证其 Authenticode 签名使用预期的 OpenClaw Foundation 签名者，写入 SHA-256 清单，并将安装程序和清单上传到规范的 OpenClaw GitHub 发布；然后重新下载已推广的工件，并验证清单成员关系和哈希值。父工作流会在发布前验证当前的 x64、ARM64 和校验和工件契约。直接恢复会先拒绝意外的 `OpenClawCompanion-*` 工件名称，再使用固定的源字节替换预期的契约工件。

  仅在恢复时手动调度 `Windows Node Release`，并且始终传入确切标签，绝不要传入 `latest`，同时传入来自已批准源发布的显式 `expected_installer_digests` JSON 映射。网站下载链接应指向当前稳定版的确切 OpenClaw 发布工件 URL；只有在验证 GitHub 的 latest 重定向指向同一发布后，才可使用 `releases/latest/download/...`；不要仅链接到配套仓库的发布页面。

- 发布检查现在在一个单独的手动工作流中运行：`OpenClaw Release Checks`。它还会在批准发布前运行 QA Lab 模拟一致性通道、Matrix 发布配置以及 Telegram QA 通道。实时通道使用 `qa-live-shared` 环境；Telegram 还使用 Convex CI 凭据租约。当需要运行所有维护中的 Matrix 场景时，请使用 `matrix_profile=all` 运行手动 `QA-Lab - All Lanes` 工作流；该工作流会将此选择分派到传输、媒体和 E2EE 配置中，以确保完整验证不会超出各作业的超时时间。
- 跨操作系统安装和升级运行时验证属于公共 `OpenClaw Release Checks` 和 `Full Release Validation` 的一部分，它们会直接调用可复用工作流 `.github/workflows/openclaw-cross-os-release-checks-reusable.yml`。这种拆分是有意为之：保持真实 npm 发布路径简短、确定且专注于工件，同时让较慢的实时检查保留在自己的通道中，避免它们拖延或阻止发布。
- 包含密钥的发布检查应通过 `Full Release Validation` 调度，或从 `main`/release 工作流引用调度，以确保工作流逻辑和密钥始终受控。
- `OpenClaw Release Checks` 接受分支、标签或完整提交 SHA，前提是解析出的提交可从 OpenClaw 分支或发布标签访问。
- `OpenClaw NPM Release` 的仅验证预检也接受当前完整的 40 字符工作流分支提交 SHA，无需已推送的标签。该 SHA 路径仅用于验证，不能提升为真实发布。在 SHA 模式下，工作流仅为软件包元数据检查合成 `v<package.json version>`；真实发布仍需要真实的发布标签。
- 两个工作流都将真实发布和提升路径保留在 GitHub 托管的运行器上，而非变更性验证路径可以使用更大的 Blacksmith Linux 运行器。
- 该工作流使用 `OPENAI_API_KEY` 和 `ANTHROPIC_API_KEY` 这两个工作流密钥运行 `OPENCLAW_LIVE_TEST=1 OPENCLAW_LIVE_CACHE_TEST=1 pnpm test:live:cache`。
- npm 发布预检不再等待单独的发布检查通道。
- 在本地为候选版本创建标签前，运行 `RELEASE_TAG=vYYYY.M.PATCH-beta.N pnpm release:fast-pretag-check`。该辅助程序会依次运行快速发布防护检查、插件 npm/ClawHub 发布检查、构建、UI 构建以及 `release:openclaw:npm:check`，以便在 GitHub 发布工作流启动前发现常见的审批阻塞错误。
- 在批准前运行 `RELEASE_TAG=vYYYY.M.PATCH node --import tsx scripts/openclaw-npm-release-check.ts`（或匹配的预发布/修正版标签）。
- npm 发布后，运行 `node --import tsx scripts/openclaw-npm-postpublish-verify.ts YYYY.M.PATCH`（或匹配的 beta/修正版本），以在全新的临时前缀中验证已发布的注册表安装路径。
- beta 发布后，运行 `OPENCLAW_NPM_TELEGRAM_PACKAGE_SPEC=openclaw@YYYY.M.PATCH-beta.N OPENCLAW_NPM_TELEGRAM_CREDENTIAL_SOURCE=convex OPENCLAW_NPM_TELEGRAM_CREDENTIAL_ROLE=ci pnpm test:docker:npm-telegram-live`，使用共享的租赁 Telegram 凭据池，针对已发布的 npm 软件包验证已安装软件包的新手引导、Telegram 设置和真实 Telegram E2E。本地维护者的一次性运行可以省略 Convex 变量，并直接传入三个 `OPENCLAW_QA_TELEGRAM_*` 环境凭据。
- 要从维护者计算机运行完整的发布后 beta 冒烟测试，请使用 `pnpm release:beta-smoke -- --beta betaN`。该辅助程序会运行 Parallels npm 更新/全新目标验证、调度 `NPM Telegram Beta E2E`、轮询精确的工作流运行、下载工件，并输出 Telegram 报告。
- 维护者可以通过手动 `NPM Telegram Beta E2E` 工作流，在 GitHub Actions 中运行相同的发布后检查。它有意设计为只能手动运行，不会在每次合并时运行。
- 维护者发布自动化采用先预检后提升的方式：
  - 真实 npm 发布必须通过成功的 npm `preflight_run_id`。
  - 常规 beta 和稳定版的发布编排及预检使用受信任的 `main`，并针对精确的目标标签运行。Tideclaw alpha 发布和预检使用匹配的 alpha 分支。
  - 稳定版 npm 发布默认为 `beta`；稳定版 npm 发布可以通过工作流输入显式指定 `latest`。
  - 基于令牌的 npm dist-tag 变更位于 `openclaw/releases/.github/workflows/openclaw-npm-dist-tags.yml`，因为 `npm dist-tag add` 仍需要 `NPM_TOKEN`，而源代码仓库仅保留 OIDC 发布。
  - 公共 `macOS Release` 仅用于验证；当标签仅存在于发布分支上，但工作流从 `main` 调度时，请设置 `public_release_branch=release/YYYY.M.PATCH`。
  - 真实 macOS 发布必须通过成功的 macOS `preflight_run_id` 和 `validate_run_id`。
  - 真实发布路径会提升已准备好的工件，而不是再次重新构建。
- 对于 `YYYY.M.PATCH-N` 这样的稳定版修正发布，发布后验证程序还会检查从 `YYYY.M.PATCH` 到 `YYYY.M.PATCH-N` 的相同临时前缀升级路径，以防发布修正悄无声息地使较旧的全局安装继续使用基础稳定版载荷。
- 除非 tarball 同时包含 `dist/control-ui/index.html` 和非空的 `dist/control-ui/assets/` 载荷，否则 npm 发布预检会以失败关闭，从而避免再次发布空的浏览器仪表盘。
- 发布后验证还会检查已发布插件的入口点和软件包元数据是否存在于已安装的注册表布局中。如果发布缺少插件运行时载荷，发布后验证程序将失败，且该版本无法提升到 `latest`。
- `pnpm test:install:smoke` 还会对候选更新 tarball 强制执行 npm pack `unpackedSize` 预算，从而使安装程序 E2E 能在发布路径开始前发现意外的软件包膨胀。
- 如果发布工作涉及 CI 规划、扩展计时清单或扩展测试矩阵，请在批准前从 `.github/workflows/plugin-prerelease.yml` 重新生成并审查由规划器管理的 `plugin-prerelease-extension-shard` 矩阵输出，避免发布说明描述过时的 CI 布局。
- 稳定版 macOS 的发布就绪性还包括更新程序表面：GitHub 发布最终必须包含已打包的 `.zip`、`.dmg` 和 `.dSYM.zip`；发布后，`main` 上的 `appcast.xml` 必须指向新的稳定版 zip（macOS 发布工作流会自动提交它，或在直接推送受阻时创建 appcast PR）；已打包应用必须保留非调试 bundle id、非空 Sparkle feed URL，并且其 `CFBundleVersion` 必须不低于该发布版本的规范 Sparkle 构建下限。

## 发布测试机

`Full Release Validation` 是操作员从单一入口启动完整产品矩阵的方式。请使用该辅助程序，使每个子工作流都从固定在一个受信任 `main` 工作流 SHA 上的临时分支运行，同时将请求的提交作为待测候选版本：

```bash
pnpm ci:full-release \
  --sha <code-sha> \
  --target-ref release/YYYY.M.PATCH
```

该辅助程序会获取当前 `origin/main`，将 `release-ci/<workflow-sha>-...` 推送到该受信任工作流提交，针对 alpha/beta 软件包版本推断 `beta`，其他版本则推断 `stable`，使用 `ref=<target-sha>` 从临时分支调度 `Full Release Validation`，验证每个子工作流的 `headSha` 均与固定的父工作流 SHA 匹配，然后删除临时分支。传入 `-f reuse_evidence=false` 可强制全新运行，传入 `-f release_profile=full` 可运行广泛的建议性扫描，传入 `--workflow-sha <trusted-main-sha>` 可固定仍能从当前 `origin/main` 访问的较旧提交。工作流本身绝不会写入仓库引用。这样既能使用仅存在于 main 的发布工具，又无需向候选版本添加工具提交，还可避免意外使用更新的 `main` 子运行进行验证。

Code SHA 通过所有检查后，仅提交 `CHANGELOG.md`，然后使用 Release SHA 运行同一个辅助程序：

```bash
pnpm ci:full-release \
  --sha <release-sha> \
  --target-ref release/YYYY.M.PATCH
```

仅当 GitHub 证明 Release SHA 是 Code SHA 的后代，且完整的变更路径集恰好为 `CHANGELOG.md` 时，第二个父工作流才会复用产品证据。它会记录 `changelog-only-release-v1`，且不调度任何产品子工作流。npm 预检和软件包/安装验收仍会在 Release SHA 上运行，因为其 tarball 字节已发生变化。

对于全新的 Code SHA，该工作流会解析目标，调度手动 `CI`，然后调度 `OpenClaw Release Checks`。启用 soak 时，`OpenClaw Release Checks` 会分派安装冒烟测试、跨操作系统发布检查、实时/E2E Docker 发布路径覆盖、包含规范 Telegram 软件包 E2E 的软件包验收、QA Lab 一致性、实时 Matrix 和实时 Telegram。只有当 `Full Release Validation` 摘要显示 `normal_ci`、`plugin_prerelease` 和 `release_checks` 均成功时，full/all 运行才可接受；除非某次聚焦重运行有意跳过单独的 `Plugin Prerelease` 子工作流。仅在使用 `release_package_spec` 或 `npm_telegram_package_spec` 对已发布软件包进行聚焦重运行时，才使用独立的 `npm-telegram` 子工作流。最终验证程序摘要包含每个子运行的最慢作业表，因此发布经理无需下载日志即可了解当前关键路径。

在此发布路径中，产品性能子工作流仅生成工件。
总工作流使用 `publish_reports=false` 调度它；除非其仅工件防护证明
Clawgrit 报告发布程序始终处于跳过状态，否则验证将被拒绝。

有关完整阶段矩阵、精确的工作流作业名称、稳定版与完整配置的差异、工件以及聚焦重运行句柄，请参阅[完整发布验证](/zh-CN/reference/full-release-validation)。

子工作流从运行 `Full Release Validation` 的 SHA 固定受信任引用中调度。每个子运行都必须使用与父工作流完全相同的 SHA。不要使用原始 `--ref main -f ref=<sha>` 调度作为发布验证；请使用 `pnpm ci:full-release --sha <target-sha> --target-ref release/YYYY.M.PATCH`。

使用 `release_profile` 选择实时/提供商覆盖范围：

- `beta`：最快的发布关键 OpenAI/核心实时和 Docker 路径
- `stable`：用于发布批准的 beta 加稳定版提供商/后端覆盖
- `full`：稳定版加广泛的建议性提供商/媒体覆盖

稳定版和完整验证始终会在提升前运行详尽的实时/E2E、Docker 发布路径以及有界的已发布版本升级存活扫描。使用 `run_release_soak=true` 可为 beta 请求相同的扫描。该扫描涵盖最新四个稳定版软件包、固定的 `2026.4.23` 和 `2026.5.2` 基线，以及较旧的 `2026.4.15` 覆盖；它会移除重复基线，并将每个基线分别分片到独立的 Docker 运行器作业中。

`OpenClaw Release Checks` 使用受信任的工作流引用将目标引用一次解析为 `release-package-under-test`，并在运行 soak 时在跨操作系统、软件包验收和发布路径 Docker 检查中复用该工件。这样可使所有面向软件包的测试机使用相同的字节，并避免重复构建软件包。beta 已发布到 npm 后，设置 `release_package_spec=openclaw@YYYY.M.PATCH-beta.N`，发布检查就会下载一次已发布的软件包，从 `dist/build-info.json` 提取其构建源 SHA，并在跨操作系统、软件包验收、发布路径 Docker 和软件包 Telegram 通道中复用该工件。

跨操作系统 OpenAI 安装冒烟测试会在设置仓库/组织变量时使用 `OPENCLAW_CROSS_OS_OPENAI_MODEL`，否则使用 `openai/gpt-5.6-luna`，因为此通道验证的是软件包安装、新手引导、Gateway 网关启动和一次实时智能体轮次，而不是对最强大的模型进行基准测试。更广泛的实时提供商矩阵仍是执行特定模型覆盖的场所。

请根据发布阶段使用以下变体：

```bash
# 验证产品完整的 Code SHA。
pnpm ci:full-release \
  --sha <code-sha> \
  --target-ref release/YYYY.M.PATCH

# 通过复用 Code SHA 产品证据，验证仅含变更日志的 Release SHA。
pnpm ci:full-release \
  --sha <release-sha> \
  --target-ref release/YYYY.M.PATCH

# 发布 beta 后，添加基于已发布软件包的 Telegram E2E。
pnpm ci:full-release \
  --sha <release-sha> \
  --target-ref release/YYYY.M.PATCH \
  -f release_package_spec=openclaw@YYYY.M.PATCH-beta.N \
  -f evidence_package_spec=openclaw@YYYY.M.PATCH-beta.N \
  -f npm_telegram_provider_mode=mock-openai
```

不要在针对性修复后的首次重跑中使用完整总流程。如果某个检查项失败，请在下一次验证中使用失败的子工作流、作业、Docker 通道、软件包配置、模型提供商或 QA 通道。仅当修复更改了共享发布编排，或导致先前所有检查项的证据失效时，才再次运行完整总流程。总流程的最终验证器会重新检查记录的子工作流运行 ID，因此成功重跑子工作流后，只需重跑失败的 `Verify full validation` 父作业。

当发布配置、实际浸泡测试设置和验证输入相匹配，并且目标 SHA 相同，或新目标是其后代且完整的变更路径集合恰好为 `CHANGELOG.md` 时，`rerun_group=all` 可以复用先前已通过的总流程运行。完全相同的目标复用会记录 `exact-target-full-validation-v1`；验证后的 Release SHA 会记录 `changelog-only-release-v1`。后者仅复用产品验证。npm 预检、软件包字节、发布说明来源以及安装/更新验收仍必须针对 Release SHA 运行。任何版本、源代码、生成内容、依赖项、软件包或工作流所有的目标发生变化，都需要新的 Code SHA 和全新的完整验证。同一 `release/*` 引用和重跑组中较新的总流程运行会自动取代正在进行的运行。传入 `reuse_evidence=false` 可强制执行全新的完整运行。

对于有限范围的恢复，请将 `rerun_group` 传给总流程。`all` 是真正的候选发布运行，`ci` 仅运行常规 CI 子流程，`plugin-prerelease` 仅运行发布专用插件子流程，`release-checks` 运行所有发布检查项，而范围更窄的发布组是 `install-smoke`、`cross-os`、`live-e2e`、`package`、`qa`、`qa-parity`、`qa-live` 和 `npm-telegram`。针对性的 `npm-telegram` 重跑需要 `release_package_spec` 或 `npm_telegram_package_spec`；完整/全部运行会使用 Package Acceptance 中规范的软件包 Telegram E2E。针对性的跨操作系统重跑可以添加 `cross_os_suite_filter=windows/packaged-upgrade` 或其他操作系统/测试套件筛选器。QA 发布检查失败会阻止常规发布验证，包括核心运行时配对通道中的 OpenClaw 动态工具漂移。Tideclaw alpha 运行仍可将不涉及软件包安全的发布检查通道视为建议性检查。使用 `release_profile=beta` 时，`Run repo/live E2E validation` 实时提供商测试套件是建议性的（仅警告，不阻止）；stable 和 full 配置仍会将其作为阻止项。当 `live_suite_filter` 明确请求受门控的 QA 实时通道（如 Discord、WhatsApp 或 Slack）时，必须启用相应的 `OPENCLAW_RELEASE_QA_*_LIVE_CI_ENABLED` 仓库变量；否则输入捕获会失败，而不是静默跳过该通道。

### Vitest

Vitest 检查项是手动 `CI` 子工作流。手动 CI 会有意绕过变更范围限定，并为候选版本强制执行常规测试图：Linux Node 分片、内置插件分片、插件和渠道契约分片、Node 22 兼容性、`check-*`、`check-additional-*`、构建产物冒烟检查、文档检查、Python Skills、Windows、macOS 和 Control UI 国际化。当 `Full Release Validation` 运行此检查项时，由于总流程会传入 `include_android=true`，因此包含 Android；独立手动 CI 需要 `include_android=true` 才能覆盖 Android。

使用此检查项回答“源代码树是否通过了完整的常规测试套件？”它与发布路径产品验证并不相同。需要保留的证据：

- `Full Release Validation` 摘要，其中显示已调度的 `CI` 运行 URL
- `CI` 在确切目标 SHA 上运行通过
- 调查回归问题时，从 CI 作业中获取失败或缓慢的分片名称
- 运行需要性能分析时，保留 `.artifacts/vitest-shard-timings.json` 等 Vitest 计时工件

仅当发布需要确定性的常规 CI，但不需要 Docker、QA Lab、实时、跨操作系统或软件包检查项时，才直接运行手动 CI。对于不包含 Android 的直接 CI，请使用第一条命令。当直接候选版本 CI 必须覆盖 Android 时，请添加 `include_android=true`：

```bash
gh workflow run ci.yml --ref main -f target_ref=release/YYYY.M.PATCH
gh workflow run ci.yml --ref main -f target_ref=release/YYYY.M.PATCH -f include_android=true
```

### Docker

Docker 检查项位于 `OpenClaw Release Checks` 至 `openclaw-live-and-e2e-checks-reusable.yml`，以及发布模式的 `install-smoke` 工作流中。它通过已打包的 Docker 环境验证候选版本，而不仅仅进行源代码级测试。

发布 Docker 覆盖范围包括：

- 完整安装冒烟测试，并启用耗时较长的 Bun 全局安装冒烟测试
- 按目标 SHA 准备/复用根 Dockerfile 冒烟镜像，QR、根/Gateway 网关以及安装程序/Bun 冒烟作业作为独立的安装冒烟分片运行
- 仓库 E2E 通道
- 发布路径 Docker 分块：`core`、`package-update-openai`、`package-update-anthropic`、`package-update-core`、`plugins-runtime-plugins`、`plugins-runtime-services`、`plugins-runtime-install-a` 至 `plugins-runtime-install-h`，以及 `openwebui`
- 按需在专用大磁盘运行器上运行 OpenWebUI 覆盖测试
- 拆分的内置插件安装/卸载通道 `bundled-plugin-install-uninstall-0` 至 `bundled-plugin-install-uninstall-23`
- 当发布检查包含实时测试套件时，运行实时/E2E 提供商测试套件和 Docker 实时模型覆盖测试

重跑之前请先使用 Docker 工件。发布路径调度器会上传 `.artifacts/docker-tests/`，其中包含通道日志、`summary.json`、`failures.json`、阶段计时、调度器计划 JSON 和重跑命令。对于针对性恢复，请在可复用的实时/E2E 工作流上使用 `docker_lanes=<lane[,lane]>`，而不是重跑所有发布分块。生成的重跑命令会包含先前的 `package_artifact_run_id` 和已准备的 Docker 镜像输入（如可用），因此失败的通道可以复用相同的 tarball 和 GHCR 镜像。

### QA Lab

QA Lab 检查项也是 `OpenClaw Release Checks` 的一部分。它是智能体行为和渠道级发布门禁，与 Vitest 和 Docker 软件包机制分开。

发布 QA Lab 覆盖范围包括：

- 模拟一致性通道，使用智能体一致性测试包将 OpenAI 候选通道与 `anthropic/claude-opus-4-8` 基线进行比较
- 使用 `qa-live-shared` 环境的 Matrix 实时适配器发布配置
- 使用 Convex CI 凭据租约的实时 Telegram QA 通道
- 当发布遥测需要明确的本地验证时，使用 `pnpm qa:otel:smoke`、`pnpm qa:otel:collector-smoke`、`pnpm qa:prometheus:smoke` 或 `pnpm qa:observability:smoke`

使用此检查项回答“发布版本在 QA 场景和实时渠道流程中的行为是否正确？”批准发布时，请保留一致性、Matrix 和 Telegram 通道的工件 URL。完整的 Matrix 覆盖测试仍可通过手动分片 QA Lab 运行执行，而不是作为默认的发布关键通道。

### 软件包

软件包检查项是可安装产品的门禁。它由 `Package Acceptance` 和解析器 `scripts/resolve-openclaw-package-candidate.mjs` 提供支持。解析器会将候选版本规范化为供 Docker E2E 使用的 `package-under-test` tarball、验证软件包清单、记录软件包版本和 SHA-256，并将工作流测试框架引用与软件包源引用分开保存。

支持的候选来源：

- `source=npm`：`openclaw@beta`、`openclaw@latest` 或确切的 OpenClaw 发布版本
- `source=ref`：使用选定的 `workflow_ref` 测试框架打包受信任的 `package_ref` 分支、标签或完整提交 SHA
- `source=url`：下载具有必需 `package_sha256` 的公共 HTTPS `.tgz`；包含凭据的 URL、非默认 HTTPS 端口、私有/内部/特殊用途主机名或解析地址以及不安全的重定向都会被拒绝
- `source=trusted-url`：从 `.github/package-trusted-sources.json` 中的命名策略下载具有必需 `package_sha256` 和 `trusted_source_id` 的 HTTPS `.tgz`；对于维护者所有的企业镜像或私有软件包仓库，请使用此方式，而不要为 `source=url` 添加输入级私有网络绕过机制
- `source=artifact`：复用由另一个 GitHub Actions 运行上传的 `.tgz`

`OpenClaw Release Checks` 使用 `source=artifact`、已准备的发布软件包工件、`suite_profile=custom`、`docker_lanes=doctor-switch update-channel-switch skill-install update-corrupt-plugin upgrade-survivor published-upgrade-survivor root-managed-vps-upgrade update-restart-auth plugins-offline plugin-update plugin-binding-command-escape`、`telegram_mode=mock-openai` 运行 Package Acceptance。Package Acceptance 针对同一个已解析 tarball 执行迁移、更新、根管理的 VPS 升级、已配置身份验证的更新重启、实时 ClawHub 技能安装、过期插件依赖项清理、离线插件夹具、插件更新、插件命令绑定转义加固和 Telegram 软件包 QA。阻止发布的检查使用默认的最新已发布软件包基线；使用 `run_release_soak=true`、`release_profile=stable` 或 `release_profile=full` 的 beta 配置会将已发布升级存续扫描扩展到 `last-stable-4`，以及固定的 `2026.4.23`、`2026.5.2` 和 `2026.4.15` 基线，并使用 `reported-issues` 场景。对于已经发布的候选版本，请使用带 `source=npm` 的 Package Acceptance；对于发布前由 SHA 支持的本地 npm tarball，请使用 `source=ref`；对于维护者所有的企业/私有镜像，请使用 `source=trusted-url`；对于由另一个 GitHub Actions 运行上传的已准备 tarball，请使用 `source=artifact`。

它是过去大多数需要 Parallels 才能完成的软件包/更新覆盖测试的 GitHub 原生替代方案。对于特定操作系统的新手引导、安装程序和平台行为，跨操作系统发布检查仍然很重要，但软件包/更新产品验证应优先使用 Package Acceptance。

更新和插件验证的规范检查清单是[更新和插件测试](/zh-CN/help/testing-updates-plugins)。在决定使用哪个本地、Docker、Package Acceptance 或发布检查通道来验证插件安装/更新、Doctor 清理或已发布软件包迁移变更时，请使用该清单。从每个 stable `2026.4.23+` 软件包执行的详尽已发布版本更新迁移属于独立的手动 `Update Migration` 工作流，不属于完整发布 CI。

旧版软件包验收宽松策略有意设置了时限。`2026.4.25` 及之前的软件包可以针对已经发布到 npm 的元数据缺失使用兼容路径：tarball 中缺少私有 QA 清单条目、缺少 `gateway install --wrapper`、从 tarball 派生的 git 夹具中缺少补丁文件、缺少持久化的 `update.channel`、旧版插件安装记录位置、缺少市场安装记录持久化，以及 `plugins update` 期间的配置元数据迁移。已发布的 `2026.4.26` 软件包可以针对已经发布的本地构建元数据戳文件发出警告。后续软件包必须满足现代软件包契约；相同的缺失会导致发布验证失败。

当发布问题涉及实际可安装的软件包时，请使用更广泛的 Package Acceptance 配置：

```bash
gh workflow run package-acceptance.yml \
  --ref main \
  -f workflow_ref=main \
  -f source=npm \
  -f package_spec=openclaw@beta \
  -f suite_profile=product \
  -f published_upgrade_survivor_baseline=openclaw@2026.4.26
```

常用软件包配置：

- `smoke`：快速软件包安装/渠道/智能体、Gateway 网关网络和配置重新加载通道
- `package`：安装/更新/重启/插件软件包契约，以及实时 ClawHub Skills 安装验证；这是发布检查的默认选项
- `product`：`package`，以及 MCP 渠道、定时任务/子智能体清理、OpenAI Web 搜索和 OpenWebUI
- `full`：包含 OpenWebUI 的 Docker 发布路径分块
- `custom`：用于针对性重新运行的确切 `docker_lanes` 列表

对于软件包候选版本的 Telegram 验证，请在软件包验收中启用 `telegram_mode=mock-openai` 或 `telegram_mode=live-frontier`。该工作流会将解析出的 `package-under-test` tarball 传入 Telegram 通道；独立的 Telegram 工作流仍接受已发布的 npm 规范，用于发布后检查。

## 常规发布自动化

对于 beta、`latest`、插件、GitHub Release 和平台发布，
`OpenClaw Release Publish` 是常规的变更入口点。每月
`.33+` Gateway 网关扩展稳定版路径不使用此编排器。
常规工作流会按照发布所需的顺序编排受信任发布者工作流：

1. 检出发布标签并解析其提交 SHA。
2. 验证该标签可从 `main` 或 `release/*` 到达（对于 alpha 预发布版本，也可以从 Tideclaw alpha 分支到达）。
3. 运行 `pnpm plugins:sync:check`。
4. 使用 `publish_scope=all-publishable` 和 `ref=<release-sha>` 调度 `Plugin NPM Release`。
5. 使用相同的范围和 SHA 调度 `Plugin ClawHub Release`。
6. 在验证已保存的 `full_release_validation_run_id` 和确切运行尝试次数后，使用发布标签、npm dist-tag 和已保存的 `preflight_run_id` 调度 `OpenClaw NPM Release`。
7. 对于稳定版本，将 GitHub release 创建或更新为草稿，使用明确的 `windows_node_tag` 和候选版本已批准的 `windows_node_installer_digests` 调度 `Windows Node Release`，并验证规范的 Windows 安装程序/校验和资产。还要调度 `Android Release`，以构建与确切标签对应的签名 APK、校验和及来源证明。在发布草稿之前验证这两项原生资产契约。

Beta 发布示例：

```bash
gh workflow run openclaw-release-publish.yml \
  --ref main \
  -f tag=vYYYY.M.PATCH-beta.N \
  -f preflight_run_id=<successful-openclaw-npm-preflight-run-id> \
  -f full_release_validation_run_id=<successful-full-release-validation-run-id> \
  -f full_release_validation_run_attempt=<successful-full-release-validation-run-attempt> \
  -f npm_dist_tag=beta
```

将稳定版本发布到默认的 beta dist-tag：

```bash
gh workflow run openclaw-release-publish.yml \
  --ref main \
  -f tag=vYYYY.M.PATCH \
  -f windows_node_tag=vX.Y.Z \
  -f windows_node_installer_digests='{"OpenClawCompanion-Setup-x64.exe":"sha256:<approved-x64-sha256>","OpenClawCompanion-Setup-arm64.exe":"sha256:<approved-arm64-sha256>"}' \
  -f preflight_run_id=<successful-openclaw-npm-preflight-run-id> \
  -f full_release_validation_run_id=<successful-full-release-validation-run-id> \
  -f full_release_validation_run_attempt=<successful-full-release-validation-run-attempt> \
  -f npm_dist_tag=beta
```

直接将稳定版本提升到 `latest` 需要明确指定：

```bash
gh workflow run openclaw-release-publish.yml \
  --ref main \
  -f tag=vYYYY.M.PATCH \
  -f windows_node_tag=vX.Y.Z \
  -f windows_node_installer_digests='{"OpenClawCompanion-Setup-x64.exe":"sha256:<approved-x64-sha256>","OpenClawCompanion-Setup-arm64.exe":"sha256:<approved-arm64-sha256>"}' \
  -f preflight_run_id=<successful-openclaw-npm-preflight-run-id> \
  -f full_release_validation_run_id=<successful-full-release-validation-run-id> \
  -f full_release_validation_run_attempt=<successful-full-release-validation-run-attempt> \
  -f npm_dist_tag=latest
```

仅将较底层的 `Plugin NPM Release` 和 `Plugin ClawHub Release` 工作流用于针对性修复或重新发布工作。当 `publish_openclaw_npm=true` 时，`OpenClaw Release Publish` 会拒绝 `plugin_publish_scope=selected`，以防核心软件包在未包含所有可发布的官方插件（包括 `@openclaw/diffs-language-pack`）时发布。对于选定插件的修复，请设置 `publish_openclaw_npm=false`，并同时设置 `plugin_publish_scope=selected` 和 `plugins=@openclaw/name`，或直接调度子工作流。

首次发布时的 ClawHub 引导是例外：从受信任的 `main`
调度 `Plugin ClawHub New`，并通过 `ref` 传入完整的目标发布 SHA。
切勿从发布标签或分支运行引导工作流本身：

```bash
gh workflow run plugin-clawhub-new.yml \
  --ref main \
  -f plugins=@openclaw/name \
  -f ref=<full-40-character-release-sha> \
  -f pretag_validation=true \
  -f dry_run=true
```

标签前验证要求 `dry_run=true`，会拒绝发布标签和父运行
输入，并且仅接受可从 `main` 或 `release/*` 到达的确切目标。
它不会加载 ClawHub 凭据、发布软件包字节，也不会更改受信任
发布者配置。该工作流仍会解析实时注册表计划，
仅在无密钥作业中检出目标并打包，准备
锁定的 ClawHub 工具链，并在发布标签存在之前验证不可变工件及软件包
slug/身份。仅在无密钥打包作业
完成后批准 `clawhub-plugin-bootstrap` 环境；此受保护的验证作业没有凭据或变更命令。

标签创建后，已批准的试运行或实际引导必须包含确切的
发布标签，以及父 `OpenClaw Release Publish` 的运行 ID、尝试次数和
分支。父运行会证明其自身的工作流 SHA，以及 `Plugin ClawHub New` 所使用的另一个确切且受信任的
`main` SHA；子运行和每个受保护
环境的批准都必须匹配该已批准的子 SHA。每次发布尝试和受信任发布者变更之前，
都会重新检查发布标签。

打包作业会
上传一个不可变工件，其名称、Actions 工件 ID/摘要、
生成方运行/尝试次数、目标 SHA，以及各软件包 tarball 的 SHA-256/大小
都会传递给验证作业和受保护作业。受保护作业仅检出受信任的 `main`
工具，通过 GitHub API 验证工件元组，按确切工件 ID 下载，
重新计算每个 tarball 的哈希，并使用固定版本 CLI 的 USTAR 规范化规则验证本地 TAR 路径和
软件包身份。随后，每个候选版本都会通过固定版本 CLI 的发布试运行；该试运行会在
查询注册表或进行身份验证之前返回。凭据作业的预筛选将压缩后的 ClawPack
限制为 120 MiB，将文件载荷总量限制为 50 MiB，将展开后的 TAR 数据限制为 64 MiB，并将
TAR 条目数限制为 10,000。现有软件包的受信任发布者修复仍然
仅执行配置，但仍会打包目标，并要求所请求的标签与注册表中的确切字节和元数据
完全一致，才能更改受信任发布者
配置。发布后验证会下载 ClawHub 工件，并
要求 SHA-256 和大小保持一致。仅当确切的生成方作业已成功
完成时，重新运行失败项的恢复操作才可复用较早尝试中的软件包工件。
最终证据还会绑定锁定的 ClawHub 版本、锁文件
SHA-256 和 npm 完整性值。若不匹配，则必须使用新的软件包版本。

## NPM 工作流输入

`OpenClaw NPM Release` 接受以下由操作员控制的输入：

- `tag`：必需的发布标签，例如 `v2026.4.2`、`v2026.4.2-1`、`v2026.4.2-beta.1` 或 `v2026.4.2-alpha.1`；当 `preflight_only=true` 时，它也可以是当前工作流分支的完整 40 字符提交 SHA，仅用于验证预检
- `preflight_only`：`true` 表示仅验证/构建/打包，`false` 表示实际发布路径
- `preflight_run_id`：现有的成功预检运行 ID；实际发布路径必须提供，以便工作流复用已准备的 tarball，而不是重新构建
- `full_release_validation_run_id`：此标签/SHA 对应的成功 `Full Release Validation` 运行 ID，实际发布时必需。Beta 发布可以在仅完成预检的情况下继续，但会发出警告；稳定版/`latest` 提升仍然必须提供。
- `full_release_validation_run_attempt`：与 `full_release_validation_run_id` 配对的确切正整数运行尝试次数；只要提供运行 ID，就必须提供此值，以防重新运行在发布期间更改授权证据。
- `release_publish_run_id`：已批准的 `OpenClaw Release Publish` 运行 ID；当此工作流由该父工作流调度时必须提供（机器人执行者的实际发布调用）
- `plugin_npm_run_id`：成功且与确切 HEAD 对应的 `Plugin NPM Release` 运行 ID；实际发布 `extended-stable` 核心软件包时必须提供
- `npm_dist_tag`：发布路径的 npm 目标标签；接受 `alpha`、`beta`、`latest` 或 `extended-stable`，默认为 `beta`。最终补丁 `33` 及更高版本必须使用 `extended-stable`；默认情况下，`extended-stable` 会拒绝更早的补丁，并且始终拒绝非最终标签。
- `bypass_extended_stable_guard`：仅用于测试的布尔值，默认为 `false`；设为 `npm_dist_tag=extended-stable` 时，会绕过每月扩展稳定版资格检查，同时保留发布身份、工件、批准和回读检查。

`Plugin NPM Release` 接受 `npm_dist_tag=default` 以使用现有发布
行为，或接受 `npm_dist_tag=extended-stable` 以使用受保护的每月发布路径。
扩展稳定版选项要求 `publish_scope=all-publishable`、空的
`plugins` 输入、不低于 `33` 的最终补丁，以及位于确切分支尖端的规范
`extended-stable/YYYY.M.33` 分支。它绝不会移动插件的
`latest` 或 `beta`。新软件包版本会通过 OIDC 受信任发布
（`npm publish --tag extended-stable`）以原子方式获得 `extended-stable`；此
源工作流不使用基于令牌身份验证的 `npm dist-tag add`。重试会
跳过 npm 中已存在的确切版本，然后以失败关闭方式终止，除非完整
回读确认每个确切软件包和 `extended-stable` 标签都已收敛。

`OpenClaw Release Publish` 接受以下由操作员控制的输入：

- `tag`：必需的发布标签；必须已存在
- `preflight_run_id`：成功的 `OpenClaw NPM Release` 预检运行 ID；当 `publish_openclaw_npm=true` 或 `plugin_publish_scope=all-publishable` 时必须提供
- `full_release_validation_run_id`：成功的 `Full Release Validation` 运行 ID；当 `publish_openclaw_npm=true` 或 `plugin_publish_scope=all-publishable` 时必须提供
- `full_release_validation_run_attempt`：与 `full_release_validation_run_id` 配对的确切正整数尝试次数；只要提供运行 ID，就必须提供
- `windows_node_tag`：确切的非预发布 `openclaw/openclaw-windows-node` 发布标签；发布稳定版 OpenClaw 时必须提供
- `windows_node_installer_digests`：经候选版本批准的紧凑 JSON 映射，将当前 Windows 安装程序名称映射到其固定的 `sha256:` 摘要；发布稳定版 OpenClaw 时必须提供
- `npm_telegram_run_id`：可选的成功 `NPM Telegram Beta E2E` 运行 ID，用于纳入最终发布证据
- `npm_dist_tag`：OpenClaw 软件包的 npm 目标标签，可为 `alpha`、`beta` 或 `latest` 之一
- `plugin_publish_scope`：默认为 `all-publishable`；仅在使用 `publish_openclaw_npm=false` 进行针对性的纯插件修复工作时使用 `selected`
- `plugins`：当 `plugin_publish_scope=selected` 时，以逗号分隔的 `@openclaw/*` 软件包名称
- `publish_openclaw_npm`：默认为 `true`；仅当将该工作流用作纯插件修复编排器时设置为 `false`
- `release_profile`：用于发布证据摘要的发布覆盖配置；默认为 `from-validation`，即从验证清单中读取；也可覆盖为 `beta`、`stable` 或 `full`
- `wait_for_clawhub`：默认为 `false`，因此 npm 可用性不会被 ClawHub 辅助流程阻塞；仅当工作流完成必须包含 ClawHub 完成时才设置为 `true`

`OpenClaw Release Checks` 接受以下由操作员控制的输入：

- `ref`：要验证的分支、标签或完整提交 SHA。包含密钥的检查要求解析后的提交可从 OpenClaw 分支或发布标签访问。
- `run_release_soak`：为 beta 发布检查启用详尽的实时/E2E、Docker 发布路径及涵盖所有历史版本的升级存活浸泡测试。`release_profile=stable` 和 `release_profile=full` 会强制启用此项。

规则：

- 补丁版本低于 `33` 的常规正式版本和修正版可以发布到 `beta` 或 `latest`。补丁版本达到或高于 `33` 的正式版本必须发布到 `extended-stable`，并且会拒绝处于该边界的带修正后缀版本。
- Beta 预发布标签只能发布到 `beta`；alpha 预发布标签只能发布到 `alpha`
- 对于 `OpenClaw NPM Release`，仅当 `preflight_only=true` 时才允许输入完整提交 SHA
- `OpenClaw Release Checks` 和 `Full Release Validation` 始终仅用于验证
- 实际发布路径必须使用预检期间所用的同一 `npm_dist_tag`；工作流会在继续发布前验证该元数据

## 常规 beta/最新稳定版发布顺序

此旧版顺序适用于同时负责插件、GitHub Release、Windows 及其他平台工作的常规编排式发布。它并非本页顶部所述的每月 `.33+` Gateway 网关扩展稳定版路径。

创建常规编排式稳定版时：

1. 使用 `preflight_only=true` 运行 `OpenClaw NPM Release`。标签尚不存在时，可以使用当前工作流分支的完整提交 SHA，对预检工作流执行仅验证的试运行。
2. 对于常规的 beta 优先流程，选择 `npm_dist_tag=beta`；仅当你有意直接发布稳定版时才选择 `latest`。
3. 如果希望通过一个手动工作流获得常规 CI 以及实时提示词缓存、Docker、QA Lab、Matrix 和 Telegram 覆盖，请在发布分支、发布标签或完整提交 SHA 上运行 `Full Release Validation`。如果你明确只需要确定性的常规测试图，请改为在发布引用上运行手动 `CI` 工作流。
4. 选择其已签名 x64 和 ARM64 安装程序应随版本发布的确切非预发布 `openclaw/openclaw-windows-node` 发布标签。将其保存为 `windows_node_tag`，并将它们已验证的摘要映射保存为 `windows_node_installer_digests`。候选发布版辅助工具会记录两者，并将其包含在生成的发布命令中。
5. 保存成功的 `preflight_run_id`、`full_release_validation_run_id` 和确切的 `full_release_validation_run_attempt`。
6. 从受信任的 `main` 运行 `OpenClaw Release Publish`，并使用相同的 `tag`、相同的 `npm_dist_tag`、所选的 `windows_node_tag`、其已保存的 `windows_node_installer_digests`、已保存的 `preflight_run_id`、`full_release_validation_run_id` 和 `full_release_validation_run_attempt`。它会先将外部化插件发布到 npm 和 ClawHub，再提升 OpenClaw npm 软件包。
7. 如果版本发布到了 `beta`，请使用 `openclaw/releases/.github/workflows/openclaw-npm-dist-tags.yml` 工作流，将该稳定版本从 `beta` 提升到 `latest`。
8. 如果版本有意直接发布到 `latest`，并且 `beta` 应立即跟随同一稳定构建，请使用同一发布工作流将两个 dist-tag 都指向该稳定版本，或者让其定时自愈同步稍后移动 `beta`。

dist-tag 变更位于发布账本仓库中，因为它仍需要 `NPM_TOKEN`，而源代码仓库仅保留基于 OIDC 的发布。这样可使直接发布路径和 beta 优先提升路径都具有文档记录，并对操作员可见。

如果维护者必须回退到本地 npm 身份验证，请仅在专用 tmux 会话中运行任何 1Password CLI（`op`）命令。不要从主智能体 shell 直接调用 `op`；将其保留在 tmux 中可让提示、警报和 OTP 处理保持可观察，并防止主机重复发出警报。

## 公开参考资料

- [`.github/workflows/full-release-validation.yml`](https://github.com/openclaw/openclaw/blob/main/.github/workflows/full-release-validation.yml)
- [`.github/workflows/package-acceptance.yml`](https://github.com/openclaw/openclaw/blob/main/.github/workflows/package-acceptance.yml)
- [`.github/workflows/openclaw-npm-release.yml`](https://github.com/openclaw/openclaw/blob/main/.github/workflows/openclaw-npm-release.yml)
- [`.github/workflows/openclaw-release-checks.yml`](https://github.com/openclaw/openclaw/blob/main/.github/workflows/openclaw-release-checks.yml)
- [`.github/workflows/openclaw-cross-os-release-checks-reusable.yml`](https://github.com/openclaw/openclaw/blob/main/.github/workflows/openclaw-cross-os-release-checks-reusable.yml)
- [`.github/workflows/docker-release.yml`](https://github.com/openclaw/openclaw/blob/main/.github/workflows/docker-release.yml)
- [`scripts/resolve-openclaw-package-candidate.mjs`](https://github.com/openclaw/openclaw/blob/main/scripts/resolve-openclaw-package-candidate.mjs)
- [`scripts/openclaw-npm-release-check.ts`](https://github.com/openclaw/openclaw/blob/main/scripts/openclaw-npm-release-check.ts)
- [`scripts/package-mac-dist.sh`](https://github.com/openclaw/openclaw/blob/main/scripts/package-mac-dist.sh)
- [`scripts/make_appcast.sh`](https://github.com/openclaw/openclaw/blob/main/scripts/make_appcast.sh)

维护者使用 [`openclaw/maintainers/release/README.md`](https://github.com/openclaw/maintainers/blob/main/release/README.md) 中的私有发布文档作为实际运行手册。

## 相关内容

- [发布渠道](/zh-CN/install/development-channels)
