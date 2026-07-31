---
read_when:
    - 你想了解 npm shrinkwrap 在 OpenClaw 发布中的含义
    - 你正在审查软件包锁定文件、依赖项变更或供应链风险
    - 你正在发布前验证根 npm 包或插件 npm 包
summary: OpenClaw 发布中 npm shrinkwrap 的通俗与技术说明
title: npm shrinkwrap
x-i18n:
    generated_at: "2026-07-26T06:44:04Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: d1e6c0d4541da9220d50cde0b9db064e5a91b81d6562cb16ac697de7d4017098
    source_path: gateway/security/shrinkwrap.md
    workflow: 16
---

OpenClaw 源代码检出使用 `pnpm-lock.yaml`。已发布的 OpenClaw npm 包使用 `npm-shrinkwrap.json`（npm 的可发布依赖锁文件），因此包安装会使用发布期间已审查的依赖关系图。

## 重要性

Shrinkwrap 是随 npm 包发布的依赖树凭据：它会告知 npm 要安装哪些确切的传递依赖版本。

| 文件                  | 适用场景                 | 含义                              |
| --------------------- | ------------------------ | --------------------------------- |
| `pnpm-lock.yaml`      | OpenClaw 源代码检出      | 维护者依赖关系图                  |
| `npm-shrinkwrap.json` | 已发布的 npm 包          | 面向用户的 npm 安装关系图         |
| `package-lock.json`   | 本地 npm 应用            | 不属于 OpenClaw 的发布契约        |

对于 OpenClaw 版本发布，这意味着：

- 已发布的包不会要求 npm 在安装时临时生成全新的依赖关系图；
- 依赖项变更可以接受审查，因为它们会体现在锁文件差异中；
- 发布验证测试的是用户将要安装的同一依赖关系图；
- 包大小或原生依赖项方面的意外问题会在发布前暴露。

Shrinkwrap 不是沙箱。它本身不能使依赖项变得安全，也不能取代主机隔离、`openclaw security audit`、包来源证明或安装冒烟测试。

OpenClaw 是 Gateway 网关、插件宿主、模型路由器和智能体运行时，因此默认安装会影响启动时间、磁盘占用、原生包下载和供应链暴露面。Shrinkwrap 为发布审查提供了稳定边界：审查者可以看到传递依赖项的变化，验证程序会拒绝意外的锁文件漂移，而插件包会携带各自锁定的依赖关系图，而不是依赖根包。

## 生成和检查

根 `openclaw` npm 包、OpenClaw 所有的 npm 插件包（例如 `@openclaw/discord`）以及 [`@openclaw/ai`](/zh-CN/reference/openclaw-ai) 等可发布的工作区包在发布时会包含 `npm-shrinkwrap.json`。工作区依赖项会从根 shrinkwrap 中省略，因为它们会与根包一同发布；每个可发布的工作区包会分别锁定自己的传递依赖树。适合的插件包也可以使用显式 `bundledDependencies` 进行发布，将其运行时依赖文件包含在插件 tarball 中，而不是仅依靠安装时解析。

```bash
# 所有由 shrinkwrap 管理的包（根包 + 可发布插件）
pnpm deps:shrinkwrap:generate
pnpm deps:shrinkwrap:check

# 仅根包
pnpm deps:shrinkwrap:root:generate
pnpm deps:shrinkwrap:root:check

# 仅受当前变更集影响的包
pnpm deps:shrinkwrap:changed:generate
pnpm deps:shrinkwrap:changed:check
```

生成器会解析 npm 的可发布锁文件格式，但会拒绝生成 `pnpm-lock.yaml` 中尚不存在的包版本。这样可以保持 pnpm 依赖项新旧程度、覆盖规则和补丁审查边界不变。

以下内容应作为安全敏感项进行审查：

- `pnpm-lock.yaml`
- `npm-shrinkwrap.json`
- 内置插件依赖项载荷
- 任何 `package-lock.json` 差异

OpenClaw 包验证程序要求新的根包 tarball 包含 shrinkwrap，并拒绝已发布包中的 `package-lock.json`。插件 npm 发布流程会检查插件本地的 shrinkwrap，安装包本地的内置依赖项，然后打包或发布。

## 检查已发布的包

根包：

```bash
npm pack openclaw@<version> --json --pack-destination /tmp/openclaw-pack
tar -tf /tmp/openclaw-pack/openclaw-<version>.tgz | grep '^package/npm-shrinkwrap.json$'
```

插件包：

```bash
npm pack @openclaw/discord@<version> --json --pack-destination /tmp/openclaw-plugin-pack
tar -tf /tmp/openclaw-plugin-pack/openclaw-discord-<version>.tgz | grep '^package/npm-shrinkwrap.json$'
tar -tf /tmp/openclaw-plugin-pack/openclaw-discord-<version>.tgz | grep '^package/node_modules/'
```

背景资料：[npm-shrinkwrap.json](https://docs.npmjs.com/cli/v11/configuring-npm/npm-shrinkwrap-json)。
