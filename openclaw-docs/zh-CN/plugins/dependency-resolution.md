---
read_when:
    - 你正在调试插件软件包安装问题
    - 你正在更改插件启动、Doctor 或包管理器安装行为
    - 你正在维护打包的 OpenClaw 安装或内置插件清单
sidebarTitle: Dependencies
summary: OpenClaw 如何安装插件包并解析插件依赖项
title: 插件依赖解析
x-i18n:
    generated_at: "2026-07-26T05:53:24Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: ae24a82568e275399cb7b68729d2805956792852612f84d6918850305f0eb243
    source_path: plugins/dependency-resolution.md
    workflow: 16
---

OpenClaw 仅在安装/更新时间处理插件依赖项。运行时加载绝不会运行包管理器、修复依赖关系树或修改 OpenClaw 包目录。

## 职责划分

插件包拥有自己的依赖关系图：

- 运行时依赖项位于插件包的 `dependencies` 或
  `optionalDependencies` 中。
- SDK/核心导入是对等依赖或由 OpenClaw 提供的导入。
- 本地开发插件自带已安装的依赖项。
- npm 和 git 插件安装到 OpenClaw 所有的包根目录中。

OpenClaw 仅负责插件生命周期：

- 发现插件来源。
- 仅在明确请求时安装或更新包。
- 记录安装元数据。
- 加载插件入口点。
- 缺少依赖项时，以可指导操作的错误终止。

## 安装根目录

OpenClaw 为每个来源使用稳定的根目录：

- npm 包安装到 `~/.openclaw/npm/projects/<encoded-package>` 下各插件独立的项目中。
- git 包克隆到 `~/.openclaw/git` 下。
- 本地/路径/归档安装通过复制或引用完成，不修复依赖项。

npm 安装会在对应插件的项目根目录中运行：

```bash
cd ~/.openclaw/npm/projects/<encoded-package>
npm install --omit=dev --omit=peer --legacy-peer-deps --ignore-scripts --no-audit --no-fund
```

`openclaw plugins install npm-pack:<path.tgz>` 对本地 npm-pack tarball 使用相同的插件专属 npm
项目根目录：OpenClaw 读取 tarball 的 npm 元数据，将其作为复制的
`file:` 依赖项添加到托管项目中，执行上述常规 npm 安装，然后在信任插件之前验证已安装的锁文件元数据。此路径用于包验收和候选版本验证，其中本地打包工件应当像它所模拟的注册表工件一样运行。

在发布前测试官方或外部插件包时，请使用 `npm-pack:`。原始归档或路径安装适用于本地调试，但无法证明其依赖路径与已安装的 npm 或 ClawHub 包相同。`npm-pack:` 可证明托管包的安装形态；但其本身并不能证明插件是与目录关联的官方内容。

当行为取决于内置插件或受信任官方插件的状态时，应将本地包验证与目录支持的官方安装或记录了官方信任状态的已发布包路径配合使用。特权辅助程序访问和受信任官方作用域处理应在该受信任安装路径上验证，而不能根据本地 tarball 安装推断。

如果插件在运行时因缺少导入而失败，应修复包清单，而不是手动修复托管项目。运行时导入应位于插件包的 `dependencies` 或 `optionalDependencies` 中；托管运行时项目不会安装 `devDependencies`。在
`~/.openclaw/npm/projects/<encoded-package>` 内添加本地 `npm install` 可以临时解除诊断阻碍，但这不能作为包验收证明，因为下次安装或更新会根据包元数据重新创建项目。

npm 可能会将传递依赖项提升到插件包旁边、该插件专属项目的
`node_modules` 中。OpenClaw 会在信任安装之前扫描托管项目根目录，并在卸载时删除该项目，因此提升的运行时依赖项仍处于该插件的清理边界内。

已发布的 npm 插件包可以附带 `npm-shrinkwrap.json`；npm 会在安装期间使用该可发布锁文件，而 OpenClaw 的托管 npm 项目根目录通过常规安装路径支持它。OpenClaw 所有的可发布插件包必须包含根据该包已发布依赖关系图生成的包本地 shrinkwrap：

```bash
pnpm deps:shrinkwrap:generate
pnpm deps:shrinkwrap:check
```

生成器会剥离插件 `devDependencies`，应用工作区覆盖策略，并为每个具有
`openclaw.release.publishToNpm: true` 的插件写入 `extensions/<id>/npm-shrinkwrap.json`。第三方插件包也可以附带 shrinkwrap；OpenClaw 不要求社区包提供此文件，但如果存在，npm 会遵循它。

在将本地包视为候选版本证明之前，请检查即将安装的 tarball：

```bash
npm pack --pack-destination /tmp
tar -xOf /tmp/<plugin-package>.tgz package/package.json
tar -tf /tmp/<plugin-package>.tgz | grep '^package/dist/'
```

对于依赖项变更，还需验证生产安装可以在没有开发依赖项的情况下解析运行时包：

```bash
tmpdir=$(mktemp -d)
(
  cd "$tmpdir"
  npm init -y >/dev/null
  npm install --package-lock-only --omit=dev --omit=peer --legacy-peer-deps --ignore-scripts /tmp/<plugin-package>.tgz
)
rm -rf "$tmpdir"
```

OpenClaw 所有的 npm 插件包也可以使用显式
`bundledDependencies` 发布。npm 发布路径会叠加运行时依赖项名称列表，从已发布清单中剥离仅限开发的工作区元数据，为包本地运行时依赖项执行无脚本 npm 安装，然后打包或发布包含这些依赖项文件的插件 tarball。大量使用原生代码的包（Codex、ACPX、Copilot、llama.cpp、memory-lancedb、Tlon）通过
`openclaw.release.bundleRuntimeDependencies: false` 选择退出；它们仍会附带 shrinkwrap，但 npm 会在安装期间解析运行时依赖项，而不是将每个平台的二进制文件都嵌入插件 tarball。根 `openclaw`
包不会捆绑其完整依赖关系树。

导入 `openclaw/plugin-sdk/*` 的插件将 `openclaw` 声明为对等依赖项。OpenClaw 不允许 npm 将宿主包的独立注册表副本安装到托管项目中，因为过时的宿主包可能影响 npm 在该插件内的对等依赖项解析。托管 npm 安装会跳过 npm 对等依赖项的解析/实体化，并且在安装或更新后，OpenClaw 会为声明了宿主对等依赖项的已安装包重新确保插件本地
`node_modules/openclaw` 链接。

git 安装会克隆或刷新仓库，然后运行：

```bash
npm install --omit=dev --ignore-scripts --no-audit --no-fund
```

随后，已安装插件会从该包目录加载，因此包本地和父级
`node_modules` 解析方式与普通 Node 包相同。

## 本地插件

本地插件是由开发者控制的目录。OpenClaw 绝不会为其运行
`npm install`、`pnpm install` 或执行依赖项修复；如果本地插件具有依赖项，请在加载前将其安装到该插件中。

第三方 TypeScript 本地插件会通过 Jiti 加载，作为应急路径。已打包的 JavaScript 插件和内置内部插件则通过原生 import/require 加载。

## 启动和重新加载

Gateway 网关启动和配置重新加载绝不会安装插件依赖项。它们会读取插件安装记录、计算入口点并加载插件。

运行时缺少依赖项会导致插件加载失败，并给出引导操作员执行明确修复的错误：

```bash
openclaw plugins update <id>
openclaw plugins install <source>
openclaw doctor --fix
```

`doctor --fix` 会清理旧版 OpenClaw 生成的依赖项状态；当配置仍引用可下载插件，但本地安装记录中缺少这些插件时，它还可以恢复插件。Doctor 不会修复已安装本地插件的依赖项。

## 内置插件

轻量且对核心至关重要的内置插件作为 OpenClaw 的一部分交付。它们要么不应包含庞大的运行时依赖关系树，要么应迁移为 ClawHub/npm 上的可下载包。

有关随核心包交付、从外部安装或仅保留源代码的插件的当前生成列表，请参阅
[插件清单](/zh-CN/plugins/plugin-inventory)。

内置插件清单不得请求依赖项暂存。大型或可选的插件功能应打包为普通插件，并通过与第三方插件相同的 npm/git/ClawHub 路径安装。

在源代码检出中，OpenClaw 将仓库视为 pnpm monorepo。执行
`pnpm install` 后，内置插件从 `extensions/<id>` 加载，因此包本地工作区依赖项可用，并且编辑会被直接采用。源代码检出开发仅支持 pnpm；在仓库根目录运行普通的 `npm install` 不会准备内置插件依赖项。

| 安装形态                    | 内置插件位置               | 依赖项所有者                                                     |
| -------------------------------- | ------------------------------------- | -------------------------------------------------------------------- |
| `npm install -g openclaw`        | 包内的已构建运行时树 | OpenClaw 包及显式插件安装/更新/Doctor 流程     |
| Git 检出加 `pnpm install` | `extensions/<id>` 工作区包  | pnpm 工作区，包括每个插件包自身的依赖项 |
| `openclaw plugins install ...`   | 托管 npm 项目/git/ClawHub 根目录  | 插件安装/更新流程                                       |

## 旧版清理

旧版 OpenClaw 会在启动时或 Doctor 修复期间生成内置插件依赖项根目录。当前 Doctor 清理会使用 `--fix` 删除这些过时的目录和符号链接，包括旧的 `plugin-runtime-deps`
根目录、指向已清理
`plugin-runtime-deps` 目标的全局 Node 前缀包符号链接、`.openclaw-runtime-deps*` 清单、生成的插件 `node_modules`、安装暂存目录和包本地 pnpm 存储。打包后的 postinstall 还会在清理旧版目标根目录前删除这些全局符号链接，因此升级不会留下悬空的 ESM 包导入。

旧版 npm 安装还使用共享的 `~/.openclaw/npm/node_modules` 根目录。当前的安装、更新、卸载和 Doctor 流程仍会识别该旧版扁平根目录，但仅用于恢复和清理。新的 npm 安装会创建各插件独立的项目根目录。
