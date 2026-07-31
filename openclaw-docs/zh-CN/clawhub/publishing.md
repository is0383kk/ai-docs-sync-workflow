---
read_when:
    - 发布技能或插件
    - 调试所有者或软件包作用域错误
    - 添加发布 UI、CLI 或后端行为
summary: ClawHub 针对技能、插件、所有者、作用域、发布版本和审查的发布机制。
x-i18n:
    generated_at: "2026-07-26T06:39:40Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 582dffaf4429e9f24d7c38f2809cc7dc05f8471e4ae2f9c6be60153cc8604e3f
    source_path: clawhub/publishing.md
    workflow: 16
---

# 发布

发布会将 Skills 文件夹或插件包发送到 ClawHub，并归入你选择的所有者名下。ClawHub 会检查你的令牌是否有权为该所有者发布，验证元数据、名称、版本、文件和源信息，然后存储该版本并启动自动安全检查。

如果验证失败，则不会发布任何内容。新版本也可能暂不出现在常规安装和下载界面中，直到审核完成。

## Skills

最简单的发布方式是使用 CLI。登录后，发布本地 Skills 文件夹：

```bash
clawhub login
clawhub skill publish ./my-skill \
  --slug my-skill \
  --name "My Skill" \
  --owner <owner>
```

发布到组织所有者名下时，请使用 `--owner <handle>`。省略它则以已通过身份验证的用户身份发布。发布时会跳过未发生变化的内容。新 Skills 从 `1.0.0` 开始，后续更改会自动发布下一个补丁版本。仅在需要显式指定版本时传入 `--version`。

对于目录仓库，请使用 ClawHub 的可复用
[`skill-publish.yml` 工作流](https://github.com/openclaw/clawhub/blob/main/.github/workflows/skill-publish.yml)。
它会针对 `root` 下的每个直属 Skills 文件夹（默认值：
`skills`）调用 `skill publish`，或者仅处理通过 `skill_path` 提供的文件夹。

```yaml
jobs:
  publish:
    uses: openclaw/clawhub/.github/workflows/skill-publish.yml@main
    with:
      owner: <owner>
      dry_run: false
    secrets:
      clawhub_token: ${{ secrets.CLAWHUB_TOKEN }}
```

使用 `dry_run: true` 可以预览新增和已更改的 Skills，而不进行发布。

## 插件

插件使用 npm 风格的包名。带作用域的包名会在名称的第一部分包含所有者：

```text
@owner/package-name
```

作用域必须与选定的发布所有者匹配。如果你的包名为 `@openclaw/dronzer`，则只能以 `@openclaw` 的身份发布。如果以 `@vintageayu` 的身份发布，请将包重命名为 `@vintageayu/dronzer`。

这可以防止包冒用发布者无权控制的组织命名空间。

如果你是已在 ClawHub 上被占用或保留的组织、品牌、包作用域、所有者标识或命名空间的合法所有者，请提交
[组织/命名空间认领议题](https://github.com/openclaw/clawhub/issues/new?template=org-namespace-claim.yml)，
并提供公开的非敏感证明。有关需要包含和不得放入公开议题的内容，请参阅
[组织和命名空间认领](/clawhub/namespace-claims)。

### 发布插件前

- 选择与包作用域匹配的所有者。
- 包含 `openclaw.plugin.json`。代码插件还需要包含 `package.json`，其中应有
  `openclaw.compat.pluginApi` 和 `openclaw.build.openclawVersion`。
- 要在首页和插件列表页面显示自定义插件目录图标，
  请将 `icon` 添加到 `openclaw.plugin.json`，其值可以是任意 HTTPS 图片 URL。
- 包含源代码仓库和确切的提交元数据，或者在基于 GitHub 的检出中使用 CLI，以便 CLI 自动检测这些信息。
- 发布前运行 `clawhub package validate <source>`。有关包、清单、SDK 导入或工件问题，请参阅
  [插件验证修复](/clawhub/plugin-validation-fixes)。
- 创建版本前运行 `clawhub package publish <source> --dry-run`。
- 新版本在自动安全检查和验证完成前不会出现在公开安装界面中，这是预期行为。

### 包的可信发布

包的可信发布需要分两步设置：

1. 首先通过常规手动方式或使用令牌进行身份验证的
   `clawhub package publish` 发布一次包。这会创建包记录，并确定哪些包管理员可以更改其可信发布者配置。
2. 由包管理员设置 GitHub Actions 可信发布者配置：

```bash
clawhub package trusted-publisher set @owner/package-name \
  --repository owner/repo \
  --workflow-filename package-publish.yml
```

设置配置后，后续受支持的 GitHub Actions 发布可以使用 OIDC/可信发布，无需在仓库中存储长期有效的 ClawHub 令牌。配置的仓库和工作流文件名必须与 GitHub Actions OIDC 声明匹配。如果还传入 `--environment <name>`，则 GitHub Actions 环境声明必须与该名称完全匹配。

设置可信发布者配置时，ClawHub 会验证所配置的 GitHub 仓库。公开仓库可以通过公开的 GitHub 元数据进行验证。私有仓库要求 ClawHub 拥有该仓库的 GitHub 访问权限，例如通过未来安装的 ClawHub GitHub App 或其他已获授权的 GitHub 集成。

当前可复用的包发布工作流支持在 `id-token: write` 可用时，对 `workflow_dispatch` 发布执行无密钥的可信发布。通过推送标签进行实际发布时仍需要 `clawhub_token`，因此请确保 `CLAWHUB_TOKEN` 可用于标签版本发布、首次发布、不受信任的包或应急发布。

使用以下命令检查或删除配置：

```bash
clawhub package trusted-publisher get @owner/package-name
clawhub package trusted-publisher delete @owner/package-name
```

删除可信发布者配置是回滚方式。在包管理员重新设置配置前，它会禁止后续签发可信发布令牌。

## 常见问题

### 包作用域必须与选定的所有者匹配

如果包作用域与选定的所有者不匹配，ClawHub 会拒绝发布：

```text
包作用域“@openclaw”必须与选定的所有者“@vintageayu”匹配。
请以“@openclaw”的身份发布，或将此包重命名为“@vintageayu/dronzer”。
```

要修复此问题，可以选择包作用域指定的所有者，或者重命名包，使其作用域与你有权代表发布的所有者匹配。

如果包名已有正确的作用域，但包归错误的发布者所有，请改为转移所有权：

```sh
clawhub package transfer @opik/opik-openclaw --to opik
```

仅当你对当前所有者和目标发布者均拥有管理员访问权限时，才能转移包或 Skills。包转移并不会让你有权发布到自己无法管理的作用域。

如果你无权访问当前所有者，但认为你的组织、项目或品牌才是该命名空间的合法所有者，请提交
[组织/命名空间认领议题](https://github.com/openclaw/clawhub/issues/new?template=org-namespace-claim.yml)，
并提供公开的非敏感证明以供工作人员审核。提交前请参阅
[组织和命名空间认领](/clawhub/namespace-claims)。

这可以保护组织命名空间。名为 `@openclaw/dronzer` 的包会占用
`@openclaw` 命名空间，因此只有能够访问 `@openclaw` 所有者的发布者才能发布该包。
