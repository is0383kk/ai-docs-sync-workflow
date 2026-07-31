---
read_when:
    - 使用 ClawHub CLI
    - 调试安装、更新或发布问题
summary: CLI 参考：命令、标志、配置和锁文件行为。
x-i18n:
    generated_at: "2026-07-26T06:36:24Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: eba91a83c5542c4b570bd22a526911633e43d0b4e921c013e6fd29451193f2a7
    source_path: clawhub/cli.md
    workflow: 16
---

# CLI

CLI 软件包：`clawhub`，二进制文件：`clawhub`。

使用 npm 或 pnpm 全局安装：

```bash
npm i -g clawhub
# 或
pnpm add -g clawhub
```

然后验证安装：

```bash
clawhub --help
clawhub login
clawhub whoami
```

## 全局标志

- `--workdir <dir>`：工作目录（默认：当前工作目录；如果已配置，则回退到 Clawdbot 工作区）
- `--dir <dir>`：工作目录下的安装目录（默认：`skills`）
- `--site <url>`：浏览器登录的基础 URL（默认：`https://clawhub.ai`）
- `--registry <url>`：API 基础 URL（默认：自动发现，否则为 `https://clawhub.ai`）
- `--no-input`：禁用提示

对应的环境变量：

- `CLAWHUB_SITE`（旧版 `CLAWDHUB_SITE`）
- `CLAWHUB_REGISTRY`（旧版 `CLAWDHUB_REGISTRY`）
- `CLAWHUB_WORKDIR`（旧版 `CLAWDHUB_WORKDIR`）

### HTTP 代理

对于位于企业代理或受限网络之后的系统，CLI 支持标准 HTTP 代理环境变量：

- `HTTPS_PROXY` / `https_proxy`
- `HTTP_PROXY` / `http_proxy`
- `NO_PROXY` / `no_proxy`

设置其中任一变量后，CLI 会通过指定的代理路由出站请求。HTTPS 请求使用 `HTTPS_PROXY`，纯 HTTP 请求使用 `HTTP_PROXY`。系统会遵循 `NO_PROXY` / `no_proxy`，对特定主机或域绕过代理。

在直接出站连接被阻止的系统上，这是必需的（例如 Docker 容器、只能通过代理访问互联网的 Hetzner VPS、企业防火墙）。

示例：

```bash
export HTTPS_PROXY=http://proxy.example.com:3128
export NO_PROXY=localhost,127.0.0.1
clawhub search "我的查询"
```

未设置代理变量时，行为保持不变（直接连接）。

## 配置文件

存储你的 API 令牌和缓存的注册表 URL。

- macOS：`~/Library/Application Support/clawhub/config.json`
- Linux/XDG：`$XDG_CONFIG_HOME/clawhub/config.json` 或 `~/.config/clawhub/config.json`
- Windows：`%APPDATA%\\clawhub\\config.json`
- 旧版回退：如果 `clawhub/config.json` 尚不存在，但 `clawdhub/config.json` 存在，CLI 将复用旧版路径
- 覆盖：`CLAWHUB_CONFIG_PATH`（旧版 `CLAWDHUB_CONFIG_PATH`）

## 命令

### `login` / `auth login`

- 默认：打开浏览器访问 `<site>/cli/auth`，并通过回环回调完成登录。
- 无头模式：`clawhub login --token clh_...`
- 远程/无头交互模式：`clawhub login --device` 会打印一个代码，并在你前往 `<site>/cli/device` 授权期间等待。

### `whoami`

- 通过 `/api/v1/whoami` 验证已存储的令牌。

### `token`

- 将已存储的 API 令牌打印到标准输出。
- 适用于通过管道将本地登录令牌传入 CI 密钥设置命令。

### `star <skill>` / `unstar <skill>`

- 在你的书签中添加/移除一项技能。为保持兼容，命令名称仍为 `star` 和
  `unstar`。
- 调用 `POST /api/v1/stars/<slug>` 和 `DELETE /api/v1/stars/<slug>`。
- `--yes` 跳过确认。

### `search <query...>`

- 调用 `/api/v1/search?q=...`。
- 输出包含技能短名、所有者用户名、显示名称和相关性分数。
- 搜索会优先考虑短名/名称词元的精确匹配，然后才考虑下载热度。像 `map` 这样的独立短名词元，与 `personal-map` 的匹配强度高于它作为 `amap` 中子字符串时的匹配强度。
- 热度只是权重较小的排序先验因素，并不保证排在首位。
- 如果某项技能本应出现却未出现，请在登录状态下运行 `clawhub inspect @owner/slug`，先检查所有者可见的审核诊断信息，再考虑重命名元数据。

### `explore`

- 通过 `/api/v1/skills?limit=...&sort=createdAt` 列出最新技能（按 `createdAt` 降序排列）。
- 标志：
  - `--limit <n>`（1-200，默认：25）
  - `--sort newest|updated|rating|downloads|trending`（默认：最新）。为保持兼容，旧版安装排序别名仍然有效。
  - `--json`（机器可读输出）
- 输出：`<slug>  v<version>  <age>  <summary>`（摘要截断为 50 个字符）。

### `inspect @owner/slug`

- 获取技能元数据和版本文件，但不安装。
- `--version <version>`：检查特定版本（默认：最新版本）。
- `--tag <tag>`：检查带标签的版本（例如 `latest`）。
- `--versions`：列出版本历史记录（第一页）。
- `--limit <n>`：要列出的最大版本数（1-200）。
- `--files`：列出所选版本的文件。
- `--file <path>`：获取原始文件字节（限制为 10MB）。
- `--json`：机器可读输出；`--file` 包含以 base64 编码的精确字节，并在可用时包含 UTF-8 文本。

### `install @owner/slug`

- 解析指定所有者和技能的最新版本。
- 通过 `/api/v1/download` 下载 zip 文件。
- 解压到 `<workdir>/<dir>/<slug>`。
- 拒绝覆盖已固定的技能；请先运行 `clawhub unpin <skill>`。
- 写入：
  - `<workdir>/.clawhub/lock.json`（旧版 `.clawdhub`）
  - `<skill>/.clawhub/origin.json`（旧版 `.clawdhub`）

### `uninstall <skill>`

- 移除 `<workdir>/<dir>/<slug>`，并删除锁文件条目。
- 登录时会尽力发送遥测数据，以便停用当前安装计数。
- 交互模式：请求确认。
- 非交互模式（`--no-input`）：需要 `--yes`。

### `list`

- 读取 `<workdir>/.clawhub/lock.json`（旧版 `.clawdhub`）。
- 在使用 `clawhub pin` 冻结的技能旁显示 `pinned`，包括可选原因。

### `pin <skill>`

- 在锁文件中将已安装的技能标记为已固定。
- `--reason <text>` 记录冻结该技能的原因。
- `update --all` 会跳过已固定的技能，直接执行 `update <skill>` 也会被拒绝。
- 已固定的技能也会拒绝 `install --force`，以防意外替换本地字节。

### `unpin <skill>`

- 从已安装技能中移除锁文件固定项，使后续更新可以修改它。

### `update [@owner/slug]` / `update --all`

- 根据本地文件计算指纹。
- 如果指纹与已知版本匹配：不显示提示。
- 如果指纹不匹配：
  - 默认拒绝
  - 使用 `--force` 覆盖（如果是交互模式，则显示提示）
- `--force` 永远不会更新已固定的技能。
- `update <skill>` 遇到已固定的技能时会立即失败，并提示你先运行 `clawhub unpin <skill>`。
- `update --all` 会跳过已固定的短名，并打印仍保持冻结状态的项目摘要。

### `skill publish <path>`

- 将本地软件包指纹与 ClawHub 进行比较；如果内容已经发布，则成功退出。
- 新技能默认使用 `1.0.0`；已更改的技能默认使用下一个补丁版本。
- `--version <version>` 显式选择版本，即使内容与现有版本匹配也会发布。
- `--dry-run` 在不上传的情况下解析发布；`--json` 打印机器可读的结果。
- `--owner <handle>` 在执行者拥有发布者访问权限时，以组织/用户发布者用户名发布。
- `--migrate-owner` 在发布新版本的同时，将现有技能移动到 `--owner`。需要对两个发布者都拥有管理员/所有者访问权限。
- 所有者和审核行为详见 `docs/publishing.md`。
- 发布技能意味着该技能在 ClawHub 上以 `MIT-0` 许可发布。
- 已发布的技能可免费使用、修改和再分发，无需署名。
- ClawHub 不支持付费技能或按技能定价。
- 旧版别名：`publish <path>`。

```bash
clawhub skill publish ./my-skill --dry-run
clawhub skill publish ./my-skill
clawhub skill publish ./my-skill --version 2.0.0
```

#### GitHub Actions

ClawHub 的可复用
[`skill-publish.yml`](https://github.com/openclaw/clawhub/blob/main/.github/workflows/skill-publish.yml)
工作流会针对一个 `skill_path` 调用 `skill publish`，或针对 `root`（默认：`skills`）下的每个直接技能文件夹调用它。该工作流会跳过未更改的技能，并采用相同的自动补丁版本行为。

将 `dry_run: true` 设置为无需令牌即可预览。实际发布需要 `clawhub_token` 密钥。

### `sync`

- 扫描当前工作目录、已配置的技能目录，以及任何 `--root <dir>` 文件夹，查找包含 `SKILL.md` 或 `skill.md` 的本地技能文件夹。
- 将每项本地技能的指纹与 ClawHub 进行比较，仅发布新增或已更改的技能。
- 新技能发布为 `1.0.0`；已更改的技能默认发布下一个补丁版本。对于需要提升更大语义版本步幅的批量更新，请使用 `--bump minor|major`。
- `--dry-run` 显示发布计划但不上传；`--json` 打印机器可读的计划。
- `--all` 发布每项新增或已更改的技能，且不显示提示。未使用 `--all` 时，可在交互式终端中选择要发布的技能。
- `--owner <handle>` 在执行者拥有发布者访问权限时，以组织/用户发布者用户名发布。
- `sync` 仅执行单向发布。它不会安装、更新、下载，也不会报告安装/下载遥测数据。

```bash
clawhub sync --all --dry-run
clawhub sync --all
clawhub sync --root ./skills --owner openclaw --bump minor
```

### `scan --slug <slug>`

- 需要 `clawhub login`。
- 通过 `POST /api/v1/skills/-/scan` 运行 ClawHub ClawScan，然后轮询至扫描进入终止状态。
- 扫描异步执行，可能需要一些时间才能完成。排队期间，终端旋转指示器会显示当前的优先扫描位置以及前方的扫描数量。
- 已发布版本的扫描需要所有权或发布者管理访问权限。版主/管理员可通过 `clawhub-admin` 使用相同的后端。
- `--update` 仅可与 `--slug` 一起使用；它会将成功的已发布扫描结果写回所选版本。
- `--output <file.zip>` 下载完整的报告归档，其中包含 `manifest.json`、`clawscan.json`、`skillspector.json`、`static-analysis.json`、`virustotal.json` 和 `README.md`。
- `--json` 打印完整的轮询响应，供自动化使用。
- 不再支持本地路径扫描。请上传新版本，然后使用 `scan download` 获取该已提交版本所存储的扫描结果。

```bash
clawhub scan --slug gifgrep
clawhub scan --slug gifgrep --version 1.2.3
clawhub scan --slug gifgrep --update --output report.zip
```

### `scan download <name>`

- 需要 `clawhub login`。
- 下载已提交技能或插件版本所存储的扫描报告 ZIP，包括被 ClawHub 安全检查阻止或隐藏的版本。
- 技能下载使用技能 slug，默认为 `--kind skill`。
- 插件下载使用包名，并且需要 `--kind plugin`。
- 必须提供 `--version`，以便作者检查被 ClawHub 阻止的确切提交版本。
- `--output <file.zip>` 用于选择目标路径。

```bash
clawhub scan download gifgrep --version 1.2.3
clawhub scan download @scope/demo --version 2.0.0 --kind plugin --output report.zip
```

#### GitHub Actions

ClawHub 为技能仓库和目录仓库提供了一个官方可复用工作流：
[`/.github/workflows/skill-publish.yml`](https://github.com/openclaw/clawhub/blob/62a697ef1e1b623afd71cf8813b545487a17354f/.github/workflows/skill-publish.yml)。

典型的目录设置：

```yaml
name: Skill Publish

on:
  pull_request:
  workflow_dispatch:

jobs:
  dry-run:
    if: github.event_name == 'pull_request'
    uses: openclaw/clawhub/.github/workflows/skill-publish.yml@v1
    with:
      owner: nvidia
      dry_run: true

  publish:
    if: github.event_name == 'workflow_dispatch'
    uses: openclaw/clawhub/.github/workflows/skill-publish.yml@v1
    with:
      owner: nvidia
      dry_run: false
    secrets:
      clawhub_token: ${{ secrets.CLAWHUB_TOKEN }}
```

注意：

- 对于目录仓库，`root` 默认为 `skills`。
- 传入 `skill_path: skills/review-helper` 可处理一个技能文件夹。
- `owner` 对应 CLI 的 `--owner` 标志；省略该标志则以已通过身份验证的用户身份发布。
- V1 技能发布使用 `clawhub_token`；GitHub OIDC 可信发布目前仅适用于包。

### `delete <skill>`

- 不使用 `--version` 时，软删除技能（所有者、版主或管理员）。
- 调用 `DELETE /api/v1/skills/{slug}`。
- 由所有者发起的软删除会保留 slug 30 天；该命令会输出到期时间。
- `--version <version>` 通过故障时关闭的版本专用路由撤回一个自有的非最新版本。该版本号仍会保留，不能使用不同内容重新发布。删除当前最新版本前，请先发布替代版本。平台工作人员无法在此仅限版本的流程中绕过所有权限制。
- `--reason <text>` 会在整个技能的软删除记录和审计日志中记录审核备注。
- `--note <text>` 是 `--reason` 的别名。
- `--yes` 跳过确认。

### `undelete <skill>`

- 恢复隐藏的技能（所有者、版主或管理员）。
- 调用 `POST /api/v1/skills/{slug}/undelete`。
- `--version <version>` 仅恢复先前由同一所有者操作者撤回并保留的确切工件。它不会将恢复的版本设为最新版本，也不会重新创建已移除的标签。
- 版本恢复调用 `POST /api/v1/skills/{slug}/versions/{version}/restore`。
- `--reason <text>` 会在技能和审计日志中记录审核备注。
- `--note <text>` 是 `--reason` 的别名。
- `--yes` 跳过确认。

### `hide <skill>`

- 隐藏技能（所有者、版主或管理员）。
- 是 `delete` 的别名。

### `unhide <skill>`

- 取消隐藏技能（所有者、版主或管理员）。
- 是 `undelete` 的别名。

### `skill rename <skill> <new-name>`

- 重命名自有技能，并将之前的 slug 保留为重定向别名。
- 调用 `POST /api/v1/skills/{slug}/rename`。
- `--yes` 跳过确认。

### `skill merge <source> <target>`

- 将一个自有技能合并到另一个自有技能中。
- 源 slug 将停止公开列出，并成为指向目标的重定向别名。
- 调用 `POST /api/v1/skills/{sourceSlug}/merge`。
- `--yes` 跳过确认。

### `transfer`

- 所有权转移工作流。
- 转移给用户账号名时会创建一个待处理请求，由接收者接受。
- 仅当操作者对当前所有者和目标发布者都拥有管理员访问权限时，转移给组织/发布者账号名才会立即生效。
- 子命令：
  - `transfer request <skill> <handle> [--message "..."] [--yes]`
  - `transfer list [--outgoing]`
  - `transfer accept <skill> [--yes]`
  - `transfer reject <skill> [--yes]`
  - `transfer cancel <skill> [--yes]`
- 端点：
  - `POST /api/v1/skills/{slug}/transfer`
  - `POST /api/v1/skills/{slug}/transfer/accept`
  - `POST /api/v1/skills/{slug}/transfer/reject`
  - `POST /api/v1/skills/{slug}/transfer/cancel`
  - `GET /api/v1/transfers/incoming`
  - `GET /api/v1/transfers/outgoing`

### `package explore [query...]`

- 通过 `GET /api/v1/packages` 和 `GET /api/v1/packages/search` 浏览或搜索统一包目录。
- 将其用于插件和其他包系列条目；顶层 `search` 仍是技能搜索界面。
- 标志：
  - `--family skill|code-plugin|bundle-plugin`
  - `--official`
  - `--executes-code`
  - `--target <target>`、`--os <os>`、`--arch <arch>`、`--libc <libc>`
  - `--requires-browser`、`--requires-desktop`、`--requires-native-deps`
  - `--requires-external-service`、`--external-service <name>`
  - `--binary <name>`、`--os-permission <name>`
  - `--artifact-kind legacy-zip|npm-pack`
  - `--npm-mirror`
  - `--limit <n>`（1-100，默认值：25）
  - `--json`

示例：

```bash
clawhub package explore --family code-plugin
clawhub package explore --family code-plugin --os darwin --requires-desktop
clawhub package explore --family code-plugin --artifact-kind npm-pack
clawhub package explore --npm-mirror
clawhub package explore episodic-claw --family code-plugin
```

### `package inspect <name>`

- 获取包元数据而不进行安装。
- 将其用于检查插件元数据、兼容性、验证信息、源代码以及版本/文件。
- `--version <version>`：检查特定版本（默认值：最新版本）。
- `--tag <tag>`：检查带标签的版本（例如 `latest`）。
- `--versions`：列出版本历史记录（第一页）。
- `--limit <n>`：要列出的最大版本数（1-100）。
- `--files`：列出所选版本的文件。
- `--file <path>`：获取有界的 UTF-8 文本预览（限制为 200KB）。
- `--json`：机器可读输出。

### `package download <name>`

- 通过 `GET /api/v1/packages/{name}/versions/{version}/artifact` 解析包版本。
- 从解析器的 `downloadUrl` 下载工件。
- 验证所有工件的 ClawHub SHA-256。
- 对于 ClawPack npm-pack 工件，还会验证 npm `sha512` 完整性、npm shasum，以及 tarball 的 `package.json` 名称/版本。
- 旧版 ZIP 版本通过旧版 ZIP 路由下载。
- 标志：
  - `--version <version>`：下载特定版本。
  - `--tag <tag>`：下载带标签的版本（默认值：`latest`）。
  - `-o, --output <path>`：输出文件或目录。
  - `--force`：覆盖现有输出文件。
  - `--json`：机器可读输出。

示例：

```bash
clawhub package download @openclaw/example-plugin --tag latest
clawhub package download @openclaw/example-plugin --version 1.2.3 -o artifacts/
```

### `package verify <file>`

- 计算本地工件的 ClawHub SHA-256、npm `sha512` 完整性和 npm shasum。
- 使用 `--package` 时，会从 ClawHub 解析预期元数据，并将本地文件与已发布的工件元数据进行比较。
- 使用直接摘要标志时，无需网络查询即可验证。
- 标志：
  - `--package <name>`：用于解析预期工件元数据的包名。
  - `--version <version>` 或 `--tag <tag>`：预期的包版本。
  - `--sha256 <hex>`：预期的 ClawHub SHA-256。
  - `--npm-integrity <sri>`：预期的 npm 完整性值。
  - `--npm-shasum <sha1>`：预期的 npm shasum。
  - `--json`：机器可读输出。

示例：

```bash
clawhub package verify ./example-plugin-1.2.3.tgz --package @openclaw/example-plugin --version 1.2.3
clawhub package verify ./example-plugin-1.2.3.tgz --sha256 <hex>
```

### `package validate <source>`

- 针对本地插件包文件夹运行 ClawHub CLI 内置的 Plugin Inspector。
- 默认执行离线/静态验证，不会定位或导入本地 OpenClaw 检出。
- 严重兼容性错误会以非零状态退出。只有警告的发现会被输出，但退出状态为零。
- 标志：
  - `--out <dir>`：将 Plugin Inspector 报告写入此目录。
  - `--openclaw <path>`：针对明确指定的本地 OpenClaw 检出进行检查。
  - `--runtime`：启用运行时捕获；会导入插件代码。
  - `--allow-execute`：允许在隔离的工作区中进行运行时捕获。
  - `--no-mock-sdk`：在运行时捕获期间禁用模拟的 OpenClaw SDK。
  - `--json`：机器可读输出。

示例：

```bash
clawhub package validate ./example-plugin
```

如果验证报告了包、清单、SDK 导入或工件问题，请参阅
[插件验证修复](/zh-CN/clawhub/plugin-validation-fixes)，然后重新运行该命令。

### `package delete <name>`

- 不使用 `--version` 时，软删除包及其所有发行版本。
- `--version <version>` 通过故障时关闭的版本专用路由撤回一个自有的非最新发行版本。该版本号仍会保留，不能使用不同内容重新发布。删除当前最新版本前，请先发布替代版本。此仅限版本的流程要求操作者是包所有者或组织发布者管理员；平台工作人员无法绕过包所有权限制。
- 整个包的软删除要求操作者是包所有者、组织发布者所有者/管理员、平台版主或平台管理员。
- 标志：
  - `--version <version>`：撤回一个非最新版本。
  - `--yes`：跳过确认。
  - `--json`：机器可读输出。

示例：

```bash
clawhub package delete @openclaw/example-plugin --yes
clawhub package delete @openclaw/example-plugin --version 1.2.3 --yes
```

### `package undelete <name>`

- 恢复软删除的包及其发行版本。
- 要求操作者是包所有者、组织发布者所有者/管理员、平台版主或平台管理员。
- 调用 `POST /api/v1/packages/{name}/undelete`。
- `--version <version>` 仅恢复先前由同一所有者操作者撤回并保留的确切发行版本。它不会将该发行版本设为最新版本，也不会重新创建已移除的包标签/dist-tags。
- 版本恢复调用 `POST /api/v1/packages/{name}/versions/{version}/restore`。
- 标志：
  - `--version <version>`：恢复一个由所有者撤回的发行版本。
  - `--yes`：跳过确认。
  - `--json`：机器可读输出。

示例：

```bash
clawhub package undelete @openclaw/example-plugin --yes
```

### `package transfer <name>`

- 将软件包转移给另一个发布者。
- 需要对当前软件包所有者和目标发布者都拥有管理员访问权限，除非由平台管理员执行。
- 带作用域的软件包名称必须转移给匹配的作用域所有者。
- 调用 `POST /api/v1/packages/{name}/transfer`。
- 标志：
  - `--to <owner>`：目标发布者标识。
  - `--reason <text>`：可选的审计原因。
  - `--json`：机器可读输出。

示例：

```bash
clawhub package transfer @openclaw/example-plugin --to openclaw
```

### `package report`

- 用于向审核员举报软件包的身份验证命令。
- 调用 `POST /api/v1/packages/{name}/report`。
- 举报以软件包为单位，可选择关联到某个版本，并会对审核员可见以供审查。
- 举报本身不会自动隐藏软件包或阻止下载。
- 标志：
  - `--version <version>`：要附加到举报的可选软件包版本。
  - `--reason <text>`：必填的举报原因。
  - `--json`：机器可读输出。

示例：

```bash
clawhub package report @openclaw/example-plugin --version 1.2.3 --reason "suspicious native payload"
```

### `package moderation-status`

- 用于检查软件包审核可见性的所有者命令。
- 调用 `GET /api/v1/packages/{name}/moderation`。
- 显示当前软件包扫描状态、未处理举报数量、最新发布版本的人工审核状态、下载阻止状态和审核原因。
- 标志：
  - `--json`：机器可读输出。

示例：

```bash
clawhub package moderation-status @openclaw/example-plugin
```

### `package readiness <name>`

- 检查软件包是否已准备好供 OpenClaw 将来使用。
- 调用 `GET /api/v1/packages/{name}/readiness`。
- 报告正式状态、ClawPack 可用性、工件摘要、来源出处、OpenClaw 兼容性、主机目标、环境元数据和扫描状态方面的阻碍因素。
- 标志：
  - `--json`：机器可读输出。

示例：

```bash
clawhub package readiness @openclaw/example-plugin
```

### `package migration-status <name>`

- 为可能替代 OpenClaw 内置插件的软件包显示面向操作员的迁移状态。
- 调用与 `package readiness` 相同的计算就绪状态端点，但输出以迁移为重点的状态、最新版本、正式软件包状态、检查结果和阻碍因素。
- 标志：
  - `--json`：机器可读输出。

示例：

```bash
clawhub package migration-status @openclaw/example-plugin
```

### `publisher create <handle>`

- 创建由已通过身份验证的用户拥有的组织发布者。
- 标识会规范化为小写，传入时可以带或不带 `@`。
- 新创建的组织发布者默认不受信任，也不是正式发布者。
- 如果标识已被现有发布者、用户或保留路由使用，则操作失败。

```bash
clawhub publisher create opik --display-name "Opik"
```

### `package publish <source>`

- 通过 `POST /api/v1/packages` 发布代码插件或插件包。
- `<source>` 接受：
  - 本地文件夹路径：`./my-plugin`
  - 本地 ClawPack npm-pack tarball：`./my-plugin-1.2.3.tgz`
  - GitHub 仓库：`owner/repo` 或 `owner/repo@ref`
  - GitHub URL：`https://github.com/owner/repo`
- 元数据会从 `package.json`、`openclaw.plugin.json` 以及真实的 OpenClaw 插件包标记（例如 `.codex-plugin/plugin.json`、`.claude-plugin/plugin.json` 和 `.cursor-plugin/plugin.json`）中自动检测。
- `.tgz` 来源会被视为 ClawPack。CLI 上传 npm-pack 的原始字节，仅将提取出的 `package/` 内容用于验证和预填充元数据。
- 代码插件文件夹会在上传前打包为 ClawPack npm tarball，以便 OpenClaw 安装时能够验证确切工件。插件包文件夹仍使用提取文件发布路径。
- 对于 GitHub 来源，来源归属会根据仓库、解析出的提交、引用和子路径自动填充。
- 对于本地文件夹，如果源远程地址指向 GitHub，则会从本地 git 自动检测来源归属。
- 外部代码插件必须显式声明 `openclaw.compat.pluginApi` 和 `openclaw.build.openclawVersion`。
  顶层 `package.json.version` 不会用作发布验证的回退值。
- `--dry-run` 会预览解析后的发布载荷，但不上传。
- `--json` 会为 CI 输出机器可读结果。
- `--owner <handle>` 会在操作方拥有发布者访问权限时，以用户或组织发布者标识的名义发布。
- 带作用域的软件包名称必须与所选所有者匹配。请参阅 `docs/publishing.md`。
- 现有标志（`--family`、`--name`、`--version`、`--source-repo`、`--source-commit`、`--source-ref`、`--source-path`）仍可作为覆盖项使用。
- 私有 GitHub 仓库需要 `GITHUB_TOKEN`。

```bash
clawhub package publish ./plugin.tgz --owner openclaw
```

#### 推荐的本地流程

首先使用 `--dry-run`，以便在创建实际发布版本之前确认解析出的软件包元数据和来源归属：

```bash
npm pack
clawhub package publish ./my-plugin-1.2.3.tgz --family code-plugin --dry-run
clawhub package publish ./my-plugin-1.2.3.tgz --family code-plugin
```

#### 本地文件夹流程

对于代码插件，文件夹发布会从软件包文件夹构建并上传 ClawPack 工件：

```bash
clawhub package publish ./my-plugin --family code-plugin --dry-run
clawhub package publish ./my-plugin --family code-plugin
```

#### 用于 `--family code-plugin` 的最小 `package.json`

外部代码插件需要在 `package.json` 中包含少量 OpenClaw 元数据。以下最小清单足以成功发布：

```json
{
  "name": "@myorg/openclaw-my-plugin",
  "version": "1.0.0",
  "type": "module",
  "openclaw": {
    "extensions": ["./index.ts"],
    "compat": {
      "pluginApi": ">=2026.3.24-beta.2"
    },
    "build": {
      "openclawVersion": "2026.3.24-beta.2"
    }
  }
}
```

必填字段：

- `openclaw.compat.pluginApi`
- `openclaw.build.openclawVersion`

说明：

- `package.json.version` 是你的软件包发布版本，但不会用作 OpenClaw 兼容性/构建验证的回退值。
- `openclaw.hostTargets` 和 `openclaw.environment` 是可选元数据。
  ClawHub 可能会在它们存在时显示这些信息，但发布并不要求提供。
- 如果希望发布更详细的兼容性元数据，可以选择额外提供 `openclaw.compat.minGatewayVersion` 和 `openclaw.build.pluginSdkVersion`。
- 如果你使用的是较旧的 `clawhub` CLI 版本，请在发布前升级，以便在上传前运行本地预检。
- 如果验证报告了修复代码，请参阅[插件验证修复](/zh-CN/clawhub/plugin-validation-fixes)。

#### GitHub Actions

ClawHub 还为插件仓库提供了正式的可复用工作流：[`/.github/workflows/package-publish.yml`](https://github.com/openclaw/clawhub/blob/62a697ef1e1b623afd71cf8813b545487a17354f/.github/workflows/package-publish.yml)。

典型的调用方设置：

```yaml
name: Package Publish

on:
  pull_request:
  workflow_dispatch:
  push:
    tags:
      - "v*"

jobs:
  dry-run:
    if: github.event_name == 'pull_request'
    uses: openclaw/clawhub/.github/workflows/package-publish.yml@v0.12.0
    with:
      dry_run: true

  publish:
    if: github.event_name == 'workflow_dispatch' || startsWith(github.ref, 'refs/tags/')
    permissions:
      contents: read
      id-token: write
    uses: openclaw/clawhub/.github/workflows/package-publish.yml@v0.12.0
    with:
      dry_run: false
    secrets:
      clawhub_token: ${{ secrets.CLAWHUB_TOKEN }}
```

说明：

- 可复用工作流默认将 `source` 设为调用方仓库。
- 对于 monorepo，请传入 `source_path`，使工作流发布插件软件包文件夹，例如 `source_path: extensions/codex`。
- 请将可复用工作流固定到稳定标签或完整提交 SHA。不要从 `@main` 运行发布。
- `pull_request` 应使用 `dry_run: true`，确保 CI 不产生污染。
- 实际发布应仅限于 `workflow_dispatch` 或标签推送等可信事件。
- 不使用密钥的可信发布仅适用于 `workflow_dispatch`；标签推送仍需要 `clawhub_token`。
- 对于首次发布、不受信任的软件包或紧急发布，请保留 `clawhub_token`。
- 工作流会将 JSON 结果作为工件上传，并将其公开为工作流输出。

### `package trusted-publisher get <name>`

- 显示软件包的 GitHub Actions 可信发布者配置。
- 设置配置后，可使用此命令确认仓库、工作流文件名和可选的环境固定项。
- 标志：
  - `--json`：机器可读输出。

示例：

```bash
clawhub package trusted-publisher get @openclaw/example-plugin
```

### `package trusted-publisher set <name>`

- 为现有软件包附加或替换 GitHub Actions 可信发布者配置。
- 必须先通过常规手动或令牌身份验证的 `clawhub package publish` 创建软件包。
- 设置配置后，未来受支持的 GitHub Actions 发布可以使用 OIDC/可信发布，而不需要长期有效的 ClawHub 令牌。
- `--repository <repo>` 必须是 `owner/repo`。
- `--workflow-filename <file>` 必须与 `.github/workflows/` 中的工作流文件名匹配。
- `--environment <name>` 是可选的。配置后，OIDC 声明中的 GitHub Actions 环境必须完全匹配。
- 运行此命令时，ClawHub 会验证所配置的 GitHub 仓库。
  公共仓库可以通过公共 GitHub 元数据进行验证。私有仓库要求 ClawHub 拥有该仓库的 GitHub 访问权限，例如通过未来安装的 ClawHub GitHub App 或其他已获授权的 GitHub 集成。
- 标志：
  - `--repository <repo>`：GitHub 仓库，例如 `openclaw/example-plugin`。
  - `--workflow-filename <file>`：工作流文件名，例如 `package-publish.yml`。
  - `--environment <name>`：可选的精确匹配 GitHub Actions 环境。
  - `--json`：机器可读输出。

示例：

```bash
clawhub package trusted-publisher set @openclaw/example-plugin \
  --repository openclaw/example-plugin \
  --workflow-filename package-publish.yml \
  --environment release
```

### `package trusted-publisher delete <name>`

- 从软件包中移除可信发布者配置。
- 如果需要禁用或重新创建工作流、仓库或环境固定项，请使用此命令回滚。
- 在再次设置配置之前，未来的实际发布必须使用常规的身份验证发布方式。
- 标志：
  - `--json`：机器可读输出。

示例：

```bash
clawhub package trusted-publisher delete @openclaw/example-plugin
```

### 安装遥测

- 登录后会在 `clawhub install <slug>` 之后发送，除非设置了 `CLAWHUB_DISABLE_TELEMETRY=1`。
- 上报采用尽力而为的方式。如果遥测不可用，安装命令不会失败。
- 详情：`docs/telemetry.md`。
