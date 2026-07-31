---
read_when:
    - 你想在 stable/extended-stable/beta/dev 之间切换
    - 你想固定到特定版本、标签或 SHA
    - 你正在标记或发布预发布版本
sidebarTitle: Release Channels
summary: 稳定版、长期稳定版、测试版和开发版渠道：语义、切换、版本锁定和标签管理
title: 发布渠道
x-i18n:
    generated_at: "2026-07-26T06:44:29Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: a99e31f5121c0ab8696e638cb10a7ce16e8f32c81e4b2bef1f703eef71191494
    source_path: install/development-channels.md
    workflow: 16
---

OpenClaw 提供四个更新渠道：

- **stable**：npm dist-tag `latest`。推荐大多数用户使用。
- **extended-stable**：npm dist-tag `extended-stable`。这是一个全新的、滞后发布的
  受支持月份软件包渠道。它仅提供软件包，并且只能在前台安装。启用
  `update.checkOnStart` 后，已存储的选择会收到只读更新提示，但绝不会自动应用更新。
- **beta**：npm dist-tag `beta`。当 `beta` 缺失
  或早于当前稳定版时，回退到 `latest`。
- **dev**：`main` 的滚动最新版本（git）。发布时使用 npm dist-tag `dev`。`main`
  用于实验和积极开发；它可能包含未完成的功能或破坏性变更。不要将其用于生产环境的 Gateway 网关。

稳定版通常会先发布到 **beta**，在那里经过验证，然后在不提升版本号的情况下
升级为 **latest**。维护者也可以直接发布到 `latest`。
dist-tag 是 npm 安装的唯一事实来源。

## 切换渠道

```bash
openclaw update --channel stable
openclaw update --channel extended-stable
openclaw update --channel beta
openclaw update --channel dev
```

`--channel` 会将选择持久化到配置中的 `update.channel`，并驱动以下两种
安装路径：

| 渠道              | npm/软件包安装                                                                                                                                                                          | git 安装                                                                                                                                                            |
| ----------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `stable`          | dist-tag `latest`                                                                                                                                                                      | 最新稳定版 git 标签（排除 `-alpha.N`、`-beta.N`、`-rc.N`、`-dev.N`、`-next.N`、`-preview.N`、`-canary.N`、`-nightly.N` 和其他命名的预发布后缀） |
| `extended-stable` | 解析公开的 npm `extended-stable` 选择器，验证选中的确切软件包，并安装该确切版本。失败时直接关闭，不回退到 `latest`、`beta` 或 `dev`。 | 不支持：OpenClaw 保持检出目录不变，并提示你使用软件包安装                                                                                                           |
| `beta`            | dist-tag `beta`；当 `beta` 缺失或版本更旧时，回退到 `latest`                                                                                                      | 最新 beta git 标签；当 beta 缺失或版本更旧时，回退到最新稳定版 git 标签                                                                                             |
| `dev`             | dist-tag `dev`（很少使用；大多数 dev 用户使用 git 安装）                                                                                                                               | 获取更新、将检出目录变基到上游 `main` 分支、构建并重新安装全局 CLI                                                                                       |

对于 `dev` git 安装，默认检出目录为 `~/openclaw`（设置
`OPENCLAW_HOME` 时为 `$OPENCLAW_HOME/openclaw`）；可使用
`OPENCLAW_GIT_DIR` 覆盖。

<Tip>
若要并行保留 stable 和 dev，请使用两个独立的检出目录，并将每个 Gateway 网关分别指向各自的目录。
</Tip>

## 一次性指定版本或标签

使用 `--tag` 为单次更新指定特定的 dist-tag、版本或软件包规范，
且**不会**更改持久化的渠道：

```bash
# 安装特定版本
openclaw update --tag 2026.4.1-beta.1

# 从 beta dist-tag 安装（一次性，不持久化）
openclaw update --tag beta

# 切换到滚动更新的 GitHub main 检出目录（持久化）
openclaw update --channel dev

# 安装特定的 npm 软件包规范
openclaw update --tag openclaw@2026.4.1-beta.1

# 从 GitHub main 安装一次，但不持久化渠道
openclaw update --tag main
```

注意：

- `--tag` 仅适用于**软件包（npm）安装**；git 安装会忽略它。
- 标签不会持久化；下一次 `openclaw update` 会使用已配置的
  渠道。
- `--tag main` 会在该次运行中映射到与 npm 兼容的规范 `github:openclaw/openclaw#main`。
  对于持久化的滚动 `main` 安装，请使用
  `openclaw update --channel dev`（软件包安装会切换到 git 检出目录），
  或使用安装程序的 git 方法重新安装：
  `curl -fsSL https://openclaw.ai/install.sh | bash -s -- --install-method git --version main`。
  npm 安装路径会直接拒绝 GitHub/git 源目标，并提示你改用 git 方法。
- 降级保护：如果目标版本早于当前版本，OpenClaw 会提示确认
  （可使用 `--yes` 跳过）。
- extended-stable 始终使用其经过验证的确切软件包目标。它不是
  `--tag extended-stable` 的一次性别名，并且 `--tag` 不能与实际生效的
  extended-stable 渠道结合使用。
- `--channel beta` 与 `--tag beta` 不同：当 beta 缺失或版本更旧时，
  渠道流程可以回退到 stable/latest，而 `--tag beta` 在该次运行中始终
  以原始 `beta` dist-tag 为目标。

## 试运行

预览 `openclaw update` 将执行的操作，而不进行任何更改：

```bash
openclaw update --dry-run
openclaw update --channel beta --dry-run
openclaw update --tag 2026.4.1-beta.1 --dry-run
openclaw update --dry-run --json
```

试运行会报告实际生效的渠道、目标版本、计划执行的操作，
以及是否需要确认降级。

## 插件和渠道

使用 `openclaw update` 切换渠道时，也会同步插件来源：

- `dev` 会将有内置对应项的已安装插件切换回
  其内置（git 检出目录）来源。
- `stable` 和 `beta` 会恢复通过 npm 或 ClawHub 安装的插件
  软件包。
- `extended-stable` 会将符合条件且使用裸/default
  或 `latest` 意图的官方 npm 插件解析到与已安装核心完全相同的版本。它不会在运行时查询
  插件的 `@extended-stable` 标签。
- 通过 npm 安装的插件会在核心更新完成后更新。

## 检查当前状态

```bash
openclaw update status
```

显示活动渠道（以及决定该渠道的来源：配置、git 标签、
git 分支、已安装版本或默认值）、安装类型（git 或软件包）、
当前版本和可用更新。

## 标签最佳实践

- 为希望 git 检出目录采用的版本添加发布标签：稳定版使用 `vYYYY.M.PATCH`，
  beta 版使用 `vYYYY.M.PATCH-beta.N`。诸如
  `-alpha.N`、`-rc.N` 和 `-next.N` 等命名的预发布后缀不属于稳定版或 beta 目标。
- 为兼容性考虑，仍会将 `vYYYY.M.PATCH-1` 和 `v1.0.1-1` 等旧版纯数字稳定标签
  识别为稳定版 git 标签。
- 为兼容性考虑，也会识别 `vYYYY.M.PATCH.beta.N`（以点分隔）；
  建议使用 `-beta.N`。
- 保持标签不可变：绝不要移动或重复使用标签。
- npm dist-tag 仍是 npm 安装的唯一事实来源：
  - `latest` -> 稳定版
  - `extended-stable` -> 滞后发布的受支持月份软件包版本
  - `beta` -> 候选构建或先以 beta 形式发布的稳定版构建
  - `dev` -> main 快照（可选）

## macOS 应用可用性

beta 和 dev 构建可能**不**包含 macOS 应用版本。这没有问题：

- git 标签和 npm dist-tag 仍然可以独立发布。
- 在发布说明或变更日志中注明“此 beta 没有 macOS 构建”。

## 相关内容

- [更新](/zh-CN/install/updating)
- [安装程序内部机制](/zh-CN/install/installer)
