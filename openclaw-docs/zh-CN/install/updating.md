---
read_when:
    - 更新 OpenClaw
    - 更新后出现故障
summary: 安全更新 OpenClaw（全局安装或源码安装）及回滚策略
title: 更新
x-i18n:
    generated_at: "2026-07-26T06:13:15Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 83444d56e0aa34f47830610538b0c3012903abb812bfe0fffb8163a5db9ac2db
    source_path: install/updating.md
    workflow: 16
---

让 OpenClaw 保持最新状态。

对于 Docker、Podman 和 Kubernetes 镜像替换，请参阅
[升级容器镜像](/zh-CN/install/docker#upgrading-container-images)。Gateway 网关会在就绪前运行可安全启动的升级工作；如果挂载的状态需要手动修复，则会退出。

## 推荐：`openclaw update`

检测你的安装类型（npm、pnpm、Bun 或 git），获取最新版本，运行 `openclaw doctor`，然后重启 Gateway 网关。

```bash
openclaw update
```

切换频道或指定特定版本：

```bash
openclaw update --channel beta
openclaw update --channel extended-stable
openclaw update --channel dev
openclaw update --dry-run   # 预览但不应用
```

`openclaw update` 没有 `--verbose` 标志（安装程序有）。若要进行诊断，请使用
`--dry-run` 预览计划执行的操作，使用 `--json` 获取结构化结果，或使用
`openclaw update status --json` 检查频道和可用性状态。

`--channel beta` 优先使用 beta npm dist-tag，但当 beta 标签不存在或其版本早于最新稳定版时，会回退到 stable/latest。若要执行一次性软件包更新并固定到原始 npm beta dist-tag，请改用 `--tag beta`。

`--channel extended-stable` 仅适用于软件包，且安装仍只能在前台进行。OpenClaw 会读取公共 npm `extended-stable` 选择器，验证所选的确切软件包，并安装该确切版本。注册表数据缺失或不一致时会以关闭方式失败；绝不会回退到 `latest`。
如果所选版本早于已安装版本，仍会应用常规的降级确认。核心更新成功后，CLI 会持久保存频道；直接执行 `npm install -g openclaw@extended-stable` 不会更新 `update.channel`。
核心替换后，符合条件且意图为 bare/default 或 `latest` 的官方 npm 插件会收敛到该确切核心版本。精确固定版本和显式非 `latest` 标签、第三方插件以及非 npm 来源均保持不变。由当前 OpenClaw 版本创建的目录安装会保留该默认意图。对于仅包含确切版本的旧记录，OpenClaw 无法安全区分旧的自动固定版本和用户固定版本，因此这些记录会保持固定；请在 extended-stable 频道上运行一次 `openclaw plugins update @openclaw/name`，让该插件重新选择跟踪确切核心版本。

`--channel dev` 提供一个持久跟随变动的 GitHub `main` 检出。对于一次性软件包更新，`--tag main` 会映射到 `github:openclaw/openclaw#main` 软件包规范，并通过目标软件包管理器（npm/pnpm/bun）直接安装。

对于托管插件，缺少 beta 版本只会产生警告，而不会导致失败：核心更新仍可成功，同时插件会回退到其记录的 default/latest 版本。

有关频道语义，请参阅[发布频道](/zh-CN/install/development-channels)。

## 在 npm 和 git 安装之间切换

使用频道更改安装类型。更新程序会保留 `~/.openclaw` 中的状态、配置、凭据和工作区；它只会更改 CLI 和 Gateway 网关使用的 OpenClaw 代码安装。

```bash
# npm 软件包安装 -> 可编辑的 git 检出
openclaw update --channel dev

# git 检出 -> npm 软件包安装
openclaw update --channel stable
```

先预览安装模式切换：

```bash
openclaw update --channel dev --dry-run
openclaw update --channel stable --dry-run
```

`dev` 会确保存在 git 检出、构建该检出，并从中安装全局 CLI。`stable`、`extended-stable` 和 `beta` 频道使用软件包安装。在 git 检出中选择 extended-stable 会被拒绝，且不会对其进行修改或转换。如果已安装 Gateway 网关，`openclaw update` 会刷新服务元数据并重启服务，除非传入 `--no-restart`。

对于使用托管 Gateway 网关服务的软件包安装，`openclaw update` 会以该服务使用的软件包根目录为目标。如果 shell 中的 `openclaw` 命令来自其他安装，更新程序会输出两个根目录和托管服务的 Node 路径，并在替换软件包前根据目标版本的 `engines.node` 要求检查该 Node 版本。

## 源码检出服务器（参考脚本）

在服务器上直接从 git 检出运行 Gateway 网关的团队，可以在该检出中使用 `scripts/update-gateway.sh` 进行更新。它是高效更新源码服务器的参考方案：恢复被 `pnpm build` 重写的已跟踪构建输出；如存在任何其他本地更改，则以关闭方式失败；快进 `main`（或将本地服务器分支变基到 `origin/main`）；安装依赖项；执行干净构建；并重启 Gateway 网关。

```bash
ssh you@server 'cd /path/to/openclaw && scripts/update-gateway.sh'
```

为自定义服务单元覆盖重启命令，或完全跳过重启：

```bash
OPENCLAW_UPDATE_RESTART_CMD='systemctl --user restart openclaw-gateway.service' scripts/update-gateway.sh
OPENCLAW_UPDATE_RESTART_CMD='' scripts/update-gateway.sh
```

对于普通的单用户源码安装，请改用 `openclaw update --channel dev`，它会为你管理检出、构建和 Gateway 网关重启。

## 替代方案：重新运行安装程序

```bash
curl -fsSL https://openclaw.ai/install.sh | bash
```

添加 `--no-onboard` 可跳过新手引导。若要强制使用特定安装类型，请传入 `--install-method git --no-onboard` 或 `--install-method npm --no-onboard`。

如果 `openclaw update` 在 npm 软件包安装阶段后失败，请重新运行安装程序。它不会调用更新程序，而是直接运行全局软件包安装，因此可以恢复部分更新的 npm 安装。

```bash
curl -fsSL https://openclaw.ai/install.sh | bash -s -- --install-method npm
```

使用 `--version` 将恢复过程固定到特定版本或 dist-tag：

```bash
curl -fsSL https://openclaw.ai/install.sh | bash -s -- --install-method npm --version <version-or-dist-tag>
```

## 替代方案：手动使用 npm、pnpm 或 bun

```bash
npm i -g openclaw@latest
```

对于受监管的安装，优先使用 `openclaw update`：它可以协调软件包替换与正在运行的 Gateway 网关服务。如果要手动更新受监管的安装，请先停止托管 Gateway 网关。软件包管理器会原地替换文件，否则正在运行的 Gateway 网关可能会尝试在替换过程中加载核心或插件文件。软件包管理器完成后，请重启 Gateway 网关，使其使用新安装。

对于由 root 拥有的 Linux 系统级全局安装，如果 `openclaw update` 因 `EACCES` 失败，请使用系统 npm 进行恢复，并在手动替换期间保持 Gateway 网关停止。使用平时为该 Gateway 网关使用的相同配置文件标志/环境。将 `/usr/bin/npm` 替换为主机上拥有 root 全局前缀的系统 npm：

```bash
openclaw gateway stop
sudo /usr/bin/npm i -g openclaw@latest
openclaw gateway install --force
openclaw gateway restart
```

然后验证：

```bash
openclaw --version
curl -fsS http://127.0.0.1:18789/readyz
openclaw plugins list --json
openclaw gateway status --deep --json
openclaw doctor --lint --json
```

当 `openclaw update` 管理全局 npm 安装时，它会先将目标安装到临时 npm 前缀中。候选软件包会在 `preinstall` 期间验证主机 Node 版本；只有验证通过后，OpenClaw 才会验证打包的 `dist` 清单，并将干净的软件包树替换到实际的全局前缀中。预期清单中会省略一个打包完成防护项，且只有在 `preinstall` 成功后才会将其移除，因此跳过生命周期脚本也会在替换前失败。在 npm 12 及更高版本中，更新程序只批准候选 OpenClaw 的生命周期；传递依赖项的脚本仍会被阻止。这可避免 npm 将新软件包覆盖到旧软件包遗留的过时文件上。如果安装命令失败，OpenClaw 会使用 `--omit=optional` 重试一次，这有助于处理无法编译原生可选依赖项的主机。

由 OpenClaw 管理的 npm 更新和插件更新命令还会为子 npm 进程清除 npm 的 `min-release-age` 供应链隔离设置（或较旧的 `before` 配置键）。该策略用于提供常规保护，但显式执行 OpenClaw 更新意味着“立即安装所选版本”。

```bash
pnpm add -g openclaw@latest
```

如果 pnpm 11 安装了 OpenClaw 2026.7.1，请手动运行一次该命令。该版本早于 pnpm 11 的隔离式全局软件包布局，因此其更新程序可能会将另一个 npm 安装误认为正在运行的 CLI。后续版本会保留 pnpm 所有权，并在更新期间跟随替换软件包的根目录。它们还会使用所属管理器报告的全局 bin 目录；当可用的 pnpm 命令报告另一个全局根目录或主版本，或调用方软件包已成为孤立软件包，或不是其中唯一有效的 OpenClaw 安装时，会在修改前停止。

如果 OpenClaw 与另一个软件包共享 pnpm 11 全局安装组，自动更新程序会在更改该组前停止。请手动更新原始的逗号分隔组，以保持其同组软件包和构建策略完整。

```bash
bun add -g openclaw@latest
```

### 高级 npm 安装主题

<AccordionGroup>
  <Accordion title="只读软件包树">
    OpenClaw 在运行时将打包的全局安装视为只读，即使当前用户对全局软件包目录拥有写入权限。插件软件包安装位于用户配置目录下由 OpenClaw 所有的 npm/git 根目录中，Gateway 网关启动时不会修改 OpenClaw 软件包树。

    一些 Linux npm 设置会将全局软件包安装在由 root 拥有的目录中，例如 `/usr/lib/node_modules/openclaw`。OpenClaw 支持这种布局，因为插件安装/更新命令会写入该全局软件包目录之外的位置。

  </Accordion>
  <Accordion title="加固的 systemd 单元">
    为 OpenClaw 提供其配置/状态根目录的写入权限，使显式插件安装、插件更新和 Doctor 清理能够持久保存更改：

    ```ini
    ReadWritePaths=/var/lib/openclaw /home/openclaw/.openclaw /tmp
    ```

  </Accordion>
  <Accordion title="磁盘空间预检">
    在软件包更新和显式插件安装前，OpenClaw 会尝试对目标卷执行尽力而为的磁盘空间检查。空间不足会产生包含检查路径的警告，但不会阻止更新，因为文件系统配额、快照和网络卷可能在检查后发生变化。实际的软件包管理器安装和安装后验证仍是最终依据。
  </Accordion>
</AccordionGroup>

## 自动更新程序

默认关闭。在 `~/.openclaw/openclaw.json` 中启用：

```json5
{
  update: {
    channel: "stable",
    auto: {
      enabled: true,
    },
  },
}
```

| 频道           | 行为                                                                                                                      |
| ----------------- | ----------------------------------------------------------------------------------------------------------------------------- |
| `stable`          | 在内置延迟后应用，并使用确定性抖动实现分散发布。                                                |
| `extended-stable` | 启用 `checkOnStart` 时，在启动时及每 24 小时检查一次只读更新提示。绝不会自动应用。 |
| `beta`            | 按内置时间间隔检查并立即应用。                                                                        |
| `dev`             | 不自动应用。请手动使用 `openclaw update`。                                                                           |

Gateway 网关还会在启动时记录更新提示（可通过
`update.checkOnStart: false` 禁用）。已存储的扩展稳定版选择会使用此
只读提示路径和现有的 24 小时提示间隔，但绝不会触发
自动安装、交接、重启、稳定版延迟/抖动或 Beta 版轮询。
进行降级或事件恢复时，请在 Gateway 网关环境中设置 `OPENCLAW_NO_AUTO_UPDATE=1`，以便即使已配置 `update.auto.enabled` 也阻止自动应用更新。除非同时禁用 `update.checkOnStart`，否则启动更新提示仍可运行。

通过实时 Gateway 网关控制平面
（`update.run`）请求的包管理器更新不会替换正在运行的 Gateway 网关
进程内的包树。在托管服务安装中，Gateway 网关会启动一个分离式交接流程，
然后退出，并让常规 `openclaw update --yes --json` CLI 路径停止
服务、替换包、刷新服务元数据、重启、验证
Gateway 网关版本和可访问性，并在可能时恢复已安装但未加载的 macOS
LaunchAgent。如果 Gateway 网关无法安全完成该交接，
`update.run` 会报告一条安全的 shell 命令，而不是在进程内运行包
管理器。

当 Control UI 侧边栏更新卡片将直接启动此
`update.run` 流程时，它会显示 **更新 Gateway 网关**。这适用于浏览器托管的 Control UI、远程
Gateway 网关和手动管理的本地 Gateway 网关。

在签名的 macOS 应用中，本地应用自有的 Gateway 网关会将该卡片改为
**更新 Mac 应用和 Gateway 网关**。Sparkle 会先更新应用；重新启动后，
应用会运行 `openclaw update --tag <app-version> --json`，重启其 Gateway 网关，
并在设置风格的进度窗口中验证健康状态。仅当该托管 Gateway 网关需要更新、修复或安装时，
此窗口才会出现；仅应用更新会在重新启动后直接进入应用。
失败详情会保持可见，并提供“重试”、[更新指南](/zh-CN/install/updating)和
[Discord](https://discord.gg/clawd) 操作。对于远程或外部管理的 Gateway 网关，应用绝不会使用此协调
路径；它绝不会降级较新的 Gateway 网关，也绝不会覆盖 `extended-stable` 渠道固定设置。

更新成功后，应用会为最近一个与真实用户/渠道发生过交互的
顶层直接会话排入一次性欢迎事件。定时任务运行、
Heartbeat 和仅后台会话更新不会改变该选择。在
远程模式下，应用只更新其本地 Mac 节点运行时，并且仅当连接的远程 Gateway 网关版本
不低于应用版本时才发送该事件。

## 更新后

<Steps>

### 运行 Doctor

```bash
openclaw doctor
```

迁移配置、审计私信策略并检查 Gateway 健康。详情：[Doctor](/zh-CN/gateway/doctor)

### 重启 Gateway 网关

```bash
openclaw gateway restart
```

### 验证

```bash
openclaw health
```

</Steps>

## 回滚

回滚分为两层：

1. 重新安装旧版 OpenClaw 代码，同时保留当前状态。
2. 仅当旧版代码无法使用已迁移的
   配置或数据库时，才恢复更新前的状态。

先从仅代码回滚开始。恢复状态会丢弃
备份后所做的更改。

### 更新前：创建经过验证的备份

`openclaw update` 会保留自动生成的更新前配置副本，但不会
创建完整的状态恢复点。在进行重大更新前，请显式创建一个：

```bash
mkdir -p ~/Backups/openclaw
openclaw backup create --output ~/Backups/openclaw --verify
```

归档清单会记录 OpenClaw 版本以及备份中包含的源路径。
归档可能包含凭据、身份验证配置文件和渠道
状态，因此请使用仅所有者可访问的权限存储它，并给予其与
实时状态目录相同的保护。有关包含和有意
省略的文件，请参阅[备份](/zh-CN/cli/backup)。

如需创建逐字节恢复点并包含可移植归档所省略的易失性工件，
请停止 Gateway 网关，并使用平台提供的文件系统、卷或虚拟机
快照。

### 回滚包安装

列出已发布的版本，然后预览并安装已知可用的版本：

```bash
npm view openclaw versions --json
openclaw update --tag <known-good-version> --dry-run
openclaw update --tag <known-good-version>
```

建议使用 `openclaw update --tag`，而不是直接通过包管理器安装。它会
检测降级并请求确认，针对已安装的目标运行托管插件收敛
和兼容性检查，刷新服务
元数据，重启 Gateway 网关，并验证正在运行的版本。如果已存储的
渠道为 `extended-stable`，请使用
`--channel stable --tag <known-good-version>`，因为精确的一次性标签不能
与 `extended-stable` 选择器组合使用。

包更新会在激活前暂存并验证候选版本。如果
文件系统交换或命令垫片替换失败，OpenClaw 会自动恢复旧
包。成功交换后，如果稍后 Gateway 健康检查失败，
系统会报告上一版本和手动回滚说明，而不是
再次自动替换包。

如果 CLI 更新路径不可用，请使用当前 Gateway 网关所属的同一包管理器和安装
范围：

```bash
openclaw gateway stop
npm i -g openclaw@<known-good-version>
openclaw gateway install --force
openclaw gateway restart
```

当安装由相应管理器负责时，请将 `npm` 替换为 `pnpm` 或 `bun`。在
事件恢复期间，请在 Gateway 网关环境中设置 `OPENCLAW_NO_AUTO_UPDATE=1`，防止已启用的自动更新程序立即应用
较新的版本。

### 回滚源代码检出

使用干净的检出并选择已知可用的标签或提交：

```bash
git fetch --all --tags
git checkout --detach <known-good-tag-or-commit>
pnpm install && pnpm build
openclaw gateway restart
```

要返回最新版，请运行：`git checkout main && git pull`。

Git 更新开始后，如果依赖项安装、构建、UI 构建或 Doctor 失败，
更新程序会自动将 Git 检出恢复到之前的分支和
SHA。当你有意选择较旧的提交时，仍需手动检出。

### 跨会话 SQLite 迁移进行降级

在启动较旧的基于文件存储的 OpenClaw 版本前，请使用当前 CLI
恢复已归档的旧版对话记录工件：

```bash
openclaw gateway stop
openclaw doctor --session-sqlite restore --session-sqlite-all-agents
```

此操作不会删除 SQLite 数据。在 SQLite 迁移后创建的会话
仅存在于 SQLite 中，不会出现在旧版运行时中。请参阅
[会话 SQLite 迁移后的降级](/zh-CN/cli/doctor#downgrading-after-session-sqlite-migration)。

### 仅在必要时恢复状态

如果旧版代码无法读取较新的配置或数据库架构，请停止
Gateway 网关，并恢复经过验证的更新前文件系统、卷或虚拟机快照。
恢复前请单独保留当前状态，因为此操作会移除
快照后所做的更改。

广泛的 `openclaw backup create` 归档支持创建和验证，但
不支持就地激活整个归档。请将广泛归档提取到暂存
目录，并使用其 `manifest.json` 源到归档映射进行离线
恢复。`openclaw backup sqlite restore` 同样会将经过验证的数据库
写入新的目标；激活该目标仍需由操作员显式执行离线
步骤。

### 验证回滚

```bash
openclaw --version
openclaw health
openclaw plugins list --json
openclaw gateway status --deep --json
openclaw doctor --lint --json
```

## 如果遇到问题

- 再次运行 `openclaw doctor`，并仔细阅读输出。
- 对于源代码检出上的 `openclaw update --channel dev`，更新程序会在需要时自动引导安装 `pnpm`。如果看到 pnpm/corepack 引导错误，请手动安装 `pnpm`（或重新启用 `corepack`），然后重新运行更新。
- 查看：[故障排查](/zh-CN/gateway/troubleshooting)
- 在 Discord 中提问：[https://discord.gg/clawd](https://discord.gg/clawd)

## 相关内容

- [安装概览](/zh-CN/install)：所有安装方法。
- [Doctor](/zh-CN/gateway/doctor)：更新后的健康检查。
- [迁移](/zh-CN/install/migrating)：主要版本迁移指南。
