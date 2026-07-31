---
read_when:
    - 你想以非交互方式读取或编辑配置
sidebarTitle: Config
summary: '`openclaw config` 的 CLI 参考（get/set/patch/unset/file/schema/validate）'
title: 配置
x-i18n:
    generated_at: "2026-07-26T06:37:44Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 4c4f8edb19737070e421c9107f7da8886e5617d9a043d8647666505c7ac9638d
    source_path: cli/config.md
    workflow: 16
---

用于 `openclaw.json` 的非交互式辅助命令：按路径获取/设置/修补/取消设置值、输出 schema、验证或输出当前文件路径。不带子命令运行 `openclaw config`，即可打开与 `openclaw configure` 相同的引导式向导。

<Note>
当 `OPENCLAW_NIX_MODE=1` 时，OpenClaw 会将 `openclaw.json` 视为不可变。只读命令（`config get`、`config file`、`config schema`、`config validate`）仍可使用；配置写入命令会拒绝执行。请改为编辑该安装的 Nix 源；对于第一方 nix-openclaw 发行版，请参阅 [nix-openclaw 快速开始](https://github.com/openclaw/nix-openclaw#quick-start)，并在 `programs.openclaw.config` 或 `instances.<name>.config` 下设置值。
</Note>

## 根选项

<ParamField path="--section <section>" type="string">
  不带子命令运行 `openclaw config` 时，可重复指定的引导式设置章节筛选器。
</ParamField>

引导式章节：`workspace`、`model`、`web`、`gateway`、`daemon`、`channels`、`plugins`、`skills`、`health`。

## 示例

```bash
openclaw config file
openclaw config --section model
openclaw config --section gateway --section daemon
openclaw config schema
openclaw config get browser.executablePath
openclaw config set browser.executablePath "/usr/bin/google-chrome"
openclaw config set browser.profiles.work.executablePath "/Applications/Google Chrome.app/Contents/MacOS/Google Chrome"
openclaw config set agents.defaults.heartbeat.every "2h"
openclaw config set 'agents.entries.main.tools.exec.node' "node-id-or-name"
openclaw config set agents.defaults.models '{"openai/gpt-5.4":{}}' --strict-json --merge
openclaw config set channels.discord.token --ref-provider default --ref-source env --ref-id DISCORD_BOT_TOKEN
openclaw config set secrets.providers.vaultfile --provider-source file --provider-path /etc/openclaw/secrets.json --provider-mode json
openclaw config patch --file ./openclaw.patch.json5 --dry-run
openclaw config unset plugins.entries.brave.config.webSearch.apiKey
openclaw config set channels.discord.token --ref-provider default --ref-source env --ref-id DISCORD_BOT_TOKEN --dry-run
openclaw config validate
openclaw config validate --json
```

### 路径

支持点号或方括号表示法。在 shell 示例中，请为方括号路径加引号，以免 zsh 对 `[0]` 进行 glob 展开：

```bash
openclaw config get agents.defaults.workspace
openclaw config get agents.entries.main
openclaw config get agents.entries
openclaw config set 'agents.entries.work.tools.exec.node' "node-id-or-name"
```

### `config get`

从已脱敏的配置快照中读取值（绝不会输出密钥）。`--json` 以 JSON 格式输出原始值；否则，字符串/数字/布尔值直接输出，对象/数组以格式化的 JSON 输出。

路径不存在时，`--json` 会将 `{ "error": "Config path not found: <path>" }` 写入 stdout，并以状态码 1 退出。如果未使用 `--json`，诊断信息仍会输出到 stderr。

```bash
openclaw config get browser.executablePath
openclaw config get agents.defaults.model --json
```

### `config file`

输出当前配置文件路径，该路径从 `OPENCLAW_CONFIG_PATH` 或默认位置解析得出。该路径指向普通文件，而不是符号链接；请参阅[写入安全](#write-safety)。

### `config schema`

将为 `openclaw.json` 生成的 JSON schema 输出到 stdout。

<AccordionGroup>
  <Accordion title="包含内容">
    - 当前根配置 schema，以及供编辑器工具使用的根级 `$schema` 字符串字段。
    - Control UI 使用的字段 `title` / `description` 文档元数据。
    - 当存在匹配的字段文档时，嵌套对象、通配符（`*`）和数组项（`[]`）节点会继承相同的 `title` / `description` 元数据。
    - `anyOf` / `oneOf` / `allOf` 分支也会继承相同的文档元数据。
    - 在可以加载运行时清单时，提供尽力而为的实时插件 + 渠道 schema 元数据。
    - 即使当前配置无效，也会提供干净的回退 schema。

  </Accordion>
  <Accordion title="相关运行时 RPC">
    `config.schema.lookup` 返回一个规范化配置路径，其中包含浅层 schema 节点（`title`、`description`、`type`、`enum`、`const`、通用边界）、匹配的 UI 提示元数据和直接子项摘要。可用于在 Control UI 或自定义客户端中按路径逐层查看。
  </Accordion>
</AccordionGroup>

```bash
openclaw config schema
openclaw config schema > openclaw.schema.json
```

### `config validate`

在不启动 Gateway 网关的情况下，依据当前 schema 验证当前配置。

```bash
openclaw config validate
openclaw config validate --json
```

<Note>
如果验证已经失败，请先使用 `openclaw configure` 或 `openclaw doctor --fix`。`openclaw chat` 不会绕过无效配置保护。
</Note>

## 值

值会尽可能解析为 JSON5；否则将被视为原始字符串。使用 `--strict-json` 可要求使用不带字符串回退的标准 JSON（此时会拒绝注释、尾随逗号或未加引号的键等仅限 JSON5 的语法）。在 `config set` 上，`--json` 是 `--strict-json` 的旧版别名。

```bash
openclaw config set agents.defaults.heartbeat.every "0m"
openclaw config set gateway.port 19001 --strict-json
openclaw config set channels.whatsapp.groups '["*"]' --strict-json
```

`config get <path> --json` 以 JSON 格式输出原始值，而不是终端格式化文本。

当写入操作更改 `agents.defaults.model` 或每个 Agent 的 `agents.entries.*.model` 时，OpenClaw 会在写入前，通过已配置的提供商目录解析每个已更改的主模型或回退模型。未知的模型引用会被拒绝，且不会更改当前配置；运行 `openclaw models list` 可查看可用模型。

<Note>
默认情况下，对象赋值会替换目标路径。对于通常包含用户添加条目的受保护路径，如果替换会移除现有条目，则会被拒绝，除非传入 `--replace`：`agents.defaults.models`、`agents.entries`、`models.providers`、`models.providers.<id>`、`models.providers.<id>.models`、`plugins.entries` 和 `auth.profiles`。
</Note>

向这些映射添加条目时，请使用 `--merge`：

```bash
openclaw config set agents.defaults.models '{"openai/gpt-5.4":{}}' --strict-json --merge
openclaw config set models.providers.ollama.models '[{"id":"llama3.2","name":"Llama 3.2"}]' --strict-json --merge
```

仅当所提供的值应有意成为完整的目标值时，才使用 `--replace`。

## `config set` 模式

<Tabs>
  <Tab title="值模式">
    ```bash
    openclaw config set <path> <value>
    ```
  </Tab>
  <Tab title="SecretRef 构建器模式">
    ```bash
    openclaw config set channels.discord.token \
      --ref-provider default \
      --ref-source env \
      --ref-id DISCORD_BOT_TOKEN
    ```
  </Tab>
  <Tab title="提供商构建器模式">
    仅以 `secrets.providers.<alias>` 路径为目标：

    ```bash
    openclaw config set secrets.providers.vault \
      --provider-source exec \
      --provider-command /usr/local/bin/openclaw-vault \
      --provider-arg read \
      --provider-arg openai/api-key \
      --provider-timeout-ms 5000
    ```

  </Tab>
  <Tab title="批处理模式">
    ```bash
    openclaw config set --batch-json '[
      {
        "path": "secrets.providers.default",
        "provider": { "source": "env" }
      },
      {
        "path": "channels.discord.token",
        "ref": { "source": "env", "provider": "default", "id": "DISCORD_BOT_TOKEN" }
      }
    ]'
    ```

    ```bash
    openclaw config set --batch-file ./config-set.batch.json --dry-run
    ```

    批处理文件的大小上限为 8 MiB。

  </Tab>
</Tabs>

<Warning>
不支持运行时修改的表面会拒绝 SecretRef 赋值（例如 `hooks.token`、`commands.ownerDisplaySecret`、Discord 线程绑定 webhook 令牌以及 WhatsApp 凭据 JSON）。请参阅 [SecretRef 凭据表面](/zh-CN/reference/secretref-credential-surface)。
</Warning>

批处理解析始终以批处理载荷（`--batch-json`/`--batch-file`）为事实来源；`--strict-json` / `--json` 不会改变批处理解析行为。

JSON 路径/值模式也可直接用于 SecretRef 和提供商：

```bash
openclaw config set channels.discord.token \
  '{"source":"env","provider":"default","id":"DISCORD_BOT_TOKEN"}' \
  --strict-json

openclaw config set secrets.providers.vaultfile \
  '{"source":"file","path":"/etc/openclaw/secrets.json","mode":"json"}' \
  --strict-json
```

### 提供商构建器标志

提供商构建器目标必须使用 `secrets.providers.<alias>` 作为路径。

<AccordionGroup>
  <Accordion title="通用标志">
    - `--provider-source <env|file|exec>`
    - `--provider-timeout-ms <ms>`（`file`、`exec`）

  </Accordion>
  <Accordion title="环境变量提供商（--provider-source env）">
    - `--provider-allowlist <ENV_VAR>`（可重复）

  </Accordion>
  <Accordion title="文件提供商（--provider-source file）">
    - `--provider-path <path>`（必需）
    - `--provider-mode <singleValue|json>`
    - `--provider-max-bytes <bytes>`
    - `--provider-allow-insecure-path`

  </Accordion>
  <Accordion title="Exec 提供商（--provider-source exec）">
    - `--provider-command <path>`（必需）
    - `--provider-arg <arg>`（可重复）
    - `--provider-no-output-timeout-ms <ms>`
    - `--provider-max-output-bytes <bytes>`
    - `--provider-json-only`
    - `--provider-env <KEY=VALUE>`（可重复）
    - `--provider-pass-env <ENV_VAR>`（可重复）
    - `--provider-trusted-dir <path>`（可重复）
    - `--provider-allow-insecure-path`
    - `--provider-allow-symlink-command`

  </Accordion>
</AccordionGroup>

强化的 Exec 提供商示例：

```bash
openclaw config set secrets.providers.vault \
  --provider-source exec \
  --provider-command /usr/local/bin/openclaw-vault \
  --provider-arg read \
  --provider-arg openai/api-key \
  --provider-json-only \
  --provider-pass-env VAULT_TOKEN \
  --provider-trusted-dir /usr/local/bin \
  --provider-timeout-ms 5000
```

## `config patch`

粘贴或通过管道传入配置结构的 JSON5 修补内容，而不必运行多个基于路径的 `config set` 命令。对象会递归合并；数组和标量值会替换目标；`null` 会删除目标路径。

```bash
openclaw config patch --file ./openclaw.patch.json5 --dry-run
openclaw config patch --file ./openclaw.patch.json5
```

修补文件的大小上限为 8 MiB。通过管道传入的 `--stdin` 修补内容大小上限为 1 MiB。

对于远程设置脚本，可通过 stdin 传入修补内容：

```bash
ssh user@gateway-host 'openclaw config patch --stdin --dry-run' < ./openclaw.patch.json5
ssh user@gateway-host 'openclaw config patch --stdin' < ./openclaw.patch.json5
```

修补示例：

```json5
{
  channels: {
    slack: {
      enabled: true,
      mode: "socket",
      botToken: { source: "env", provider: "default", id: "SLACK_BOT_TOKEN" },
      appToken: { source: "env", provider: "default", id: "SLACK_APP_TOKEN" },
      groupPolicy: "open",
      requireMention: false,
    },
    discord: {
      enabled: true,
      token: { source: "env", provider: "default", id: "DISCORD_BOT_TOKEN" },
      dmPolicy: "disabled",
      dm: { enabled: false },
      groupPolicy: "allowlist",
    },
  },
  agents: {
    defaults: {
      model: { primary: "openai/gpt-5.6-sol" },
      models: {
        "openai/gpt-5.6-sol": { params: { fastMode: true } },
      },
    },
  },
}
```

当某个对象或数组必须完全变为所提供的值，而不是进行递归修补时，请使用 `--replace-path <path>`：

```bash
openclaw config patch --file ./discord.patch.json5 --replace-path 'channels.discord.guilds["123"].channels'
```

`--dry-run` 会执行架构和 SecretRef 可解析性检查，但不会写入。默认情况下，试运行期间会跳过由 Exec 支持的 SecretRef；如果确实希望试运行执行提供商命令，请添加 `--allow-exec`。

## 试运行

`--dry-run` 会验证更改，但不会写入 `openclaw.json`。可用于 `config set`、`config patch` 和 `config unset`。

```bash
openclaw config set channels.discord.token \
  --ref-provider default \
  --ref-source env \
  --ref-id DISCORD_BOT_TOKEN \
  --dry-run \
  --json

openclaw config set channels.discord.token \
  --ref-provider vault \
  --ref-source exec \
  --ref-id discord/token \
  --dry-run \
  --allow-exec
```

<AccordionGroup>
  <Accordion title="试运行行为">
    - 构建器模式：对已更改的引用/提供商执行 SecretRef 可解析性检查。
    - JSON 模式（`--strict-json`、`--json` 或批处理模式）：执行架构验证和 SecretRef 可解析性检查。
    - 策略验证会针对更改后的完整配置执行，因此父对象写入（例如将 `hooks` 设置为对象）无法绕过不受支持表面的验证。
    - 默认跳过 Exec SecretRef 检查，以避免命令产生副作用；传入 `--allow-exec` 可选择启用（这可能会执行提供商命令）。`--allow-exec` 仅用于试运行，若没有 `--dry-run` 则会报错。

  </Accordion>
  <Accordion title="--dry-run --json 字段">
    - `ok`：试运行是否通过
    - `operations`：已评估的赋值数量
    - `checks`：是否执行了架构/可解析性检查
    - `checks.resolvabilityComplete`：可解析性检查是否执行完成（跳过 Exec 引用时为 false）
    - `refsChecked`：试运行期间实际解析的引用数量
    - `skippedExecRefs`：因未设置 `--allow-exec` 而跳过的 Exec 引用数量
    - `errors`：当 `ok=false` 时，以结构化形式返回缺失路径、架构或可解析性失败

  </Accordion>
</AccordionGroup>

### JSON 输出结构

```json5
{
  ok: boolean,
  operations: number,
  configPath: string,
  inputModes: ["value" | "json" | "builder" | "unset", ...],
  checks: {
    schema: boolean,
    resolvability: boolean,
    resolvabilityComplete: boolean,
  },
  refsChecked: number,
  skippedExecRefs: number,
  errors?: [
    {
      kind: "missing-path" | "schema" | "resolvability" | "model",
      message: string,
      ref?: string, // 出现于可解析性错误
    },
  ],
}
```

<Tabs>
  <Tab title="成功示例">
    ```json
    {
      "ok": true,
      "operations": 1,
      "configPath": "~/.openclaw/openclaw.json",
      "inputModes": ["builder"],
      "checks": {
        "schema": false,
        "resolvability": true,
        "resolvabilityComplete": true
      },
      "refsChecked": 1,
      "skippedExecRefs": 0
    }
    ```
  </Tab>
  <Tab title="失败示例">
    ```json
    {
      "ok": false,
      "operations": 1,
      "configPath": "~/.openclaw/openclaw.json",
      "inputModes": ["builder"],
      "checks": {
        "schema": false,
        "resolvability": true,
        "resolvabilityComplete": true
      },
      "refsChecked": 1,
      "skippedExecRefs": 0,
      "errors": [
        {
          "kind": "resolvability",
          "message": "错误：未设置环境变量 \"MISSING_TEST_SECRET\"。",
          "ref": "env:default:MISSING_TEST_SECRET"
        }
      ]
    }
    ```
  </Tab>
</Tabs>

<AccordionGroup>
  <Accordion title="如果试运行失败">
    - `config schema validation failed`：更改后的配置结构无效；请修复路径/值或提供商/引用对象结构。
    - `Config policy validation failed: unsupported SecretRef usage`：将该凭据恢复为明文/字符串输入；SecretRef 只能用于受支持的表面。
    - `SecretRef assignment(s) could not be resolved`：当前无法解析引用的提供商/引用（环境变量缺失、文件指针无效、Exec 提供商失败或提供商/来源不匹配）。
    - `model reference validation failed`：已更改的文本模型主模型或回退模型未知；请运行 `openclaw models list` 并选择可用模型。
    - `Dry run note: skipped <n> exec SecretRef resolvability check(s)`：如果需要验证 Exec 可解析性，请使用 `--allow-exec` 重新运行。
    - 对于批处理模式，请修复失败的条目，并在写入前重新运行 `--dry-run`。

  </Accordion>
</AccordionGroup>

## 应用更改

每次成功执行 `config set` / `config patch` / `config unset` 后，CLI 都会输出以下三种提示之一，以便确定 Gateway 网关是否需要重启：

| 提示                                                | 含义                                |
| --------------------------------------------------- | -------------------------------------- |
| `Restart the gateway to apply.`                     | 更改的路径需要完整重启。 |
| `Change will apply without restarting the gateway.` | 热重载会自动应用更改。  |
| `No gateway restart needed.`                        | 未发生与运行时相关的更改。      |

写入 `plugins.entries`（或其任何子路径）始终需要重启，因为 CLI 无法确认是否已加载每个插件的重载元数据。

## 写入安全

`openclaw config set` 和其他 OpenClaw 自有的配置写入工具会在将更改提交到磁盘前验证更改后的完整配置。如果新负载未通过架构验证或疑似会造成破坏性覆盖，则活动配置保持不变，被拒绝的负载会以 `openclaw.json.rejected.*` 的形式保存在其旁边。

OpenClaw 自有的写入操作会将 JSON5 重新序列化为标准 JSON。当源文件包含注释时，写入工具会在删除注释前立即发出警告；如果需要保留注释，请使用编辑器直接修改。

<Warning>
活动配置路径必须是常规文件。写入操作不支持使用符号链接的 `openclaw.json` 布局；请改用 `OPENCLAW_CONFIG_PATH` 直接指向真实文件。
</Warning>

小范围编辑优先使用 CLI 写入：

```bash
openclaw config set gateway.reload.mode hybrid --dry-run
openclaw config set gateway.reload.mode hybrid
openclaw config validate
```

如果写入被拒绝，请检查已保存的负载并修复完整配置结构：

```bash
CONFIG="$(openclaw config file)"
ls -lt "$CONFIG".rejected.* 2>/dev/null | head
openclaw config validate
```

仍然允许直接使用编辑器写入，但在验证通过前，运行中的 Gateway 网关会将其视为不可信内容。无效的直接编辑会导致启动失败或被热重载跳过；Gateway 网关不会重写 `openclaw.json`。运行 `openclaw doctor --fix` 可修复带前缀/被覆盖的配置，或恢复上次已知正常的副本。请参阅 [Gateway 故障排除](/zh-CN/gateway/troubleshooting#gateway-rejected-invalid-config)。

仅 Doctor 修复可以执行全文件恢复。插件架构更改或 `minHostVersion` 偏差会保持明确报错，而不会回滚模型、提供商、身份验证配置文件、渠道、Gateway 网关暴露、工具、记忆、浏览器或定时任务配置等不相关的用户设置。

## 修复循环

`openclaw config validate` 通过后，使用本地 TUI，让嵌入式智能体对照文档比较活动配置，同时在同一终端中验证每项更改：

```bash
openclaw chat
```

在 TUI 中，以 `!` 开头的内容会运行字面意义上的本地 shell 命令（每个会话首次运行时需要确认）：

```text
!openclaw config file
!openclaw docs gateway auth token secretref
!openclaw config validate
!openclaw doctor
```

<Steps>
  <Step title="与文档比较">
    要求智能体将当前配置与相关文档页面进行比较，并建议最小修复方案。
  </Step>
  <Step title="应用针对性编辑">
    使用 `openclaw config set` 或 `openclaw configure` 应用针对性编辑。
  </Step>
  <Step title="重新验证">
    每次更改后重新运行 `openclaw config validate`。
  </Step>
  <Step title="使用 Doctor 解决运行时问题">
    如果验证通过但运行时仍不正常，请运行 `openclaw doctor` 或 `openclaw doctor --fix`，以获取迁移和修复帮助。
  </Step>
</Steps>

## 相关内容

- [CLI 参考](/zh-CN/cli)
- [配置](/zh-CN/gateway/configuration)
