---
read_when:
    - 在运行时重新解析密钥引用
    - 审计明文残留和未解析的引用
    - 配置 SecretRefs 并应用单向清理更改
summary: '`openclaw secrets` 的 CLI 参考（重新加载、审计、配置、应用）'
title: 密钥
x-i18n:
    generated_at: "2026-07-26T06:44:44Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 61f6f81e358ca2e6a97ac9498186b32f7a74d16052d226c398dad0030d47211e
    source_path: cli/secrets.md
    workflow: 16
---

# `openclaw secrets`

管理 SecretRef，并保持活动运行时快照健康。

| 命令     | 作用                                                                                                                                                                                         |
| ----------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `reload`    | Gateway 网关 RPC（`secrets.reload`）：重新解析引用，并以原子方式发布感知所有者的运行时快照（不写入配置）；符合条件的所有者故障可能以冷警告或陈旧警告的形式发布 |
| `audit`     | 对配置/身份验证/生成模型存储和旧版残留进行只读扫描，检查明文、未解析的引用和优先级漂移（除非指定 `--allow-exec`，否则跳过 exec 引用）                      |
| `configure` | 用于提供商设置、目标映射和预检的交互式规划器（需要 TTY）                                                                                                       |
| `apply`     | 执行已保存的计划（`--dry-run` 仅执行验证，默认跳过 exec 检查；除非指定 `--allow-exec`，否则写入模式会拒绝包含 exec 的计划），然后清除目标明文残留 |

建议的操作员流程：

```bash
openclaw secrets audit --check
openclaw secrets configure
openclaw secrets apply --from /tmp/openclaw-secrets-plan.json --dry-run
openclaw secrets apply --from /tmp/openclaw-secrets-plan.json
openclaw secrets audit --check
openclaw secrets reload
```

如果计划包含 `exec` SecretRef/提供商，请在试运行和写入 `apply` 命令中都传入 `--allow-exec`。

CI/门禁的退出代码：

- `audit --check` 在发现问题时返回 `1`。
- 未解析的引用返回 `2`（无论是否指定 `--check`）。

相关内容：[密钥管理](/zh-CN/gateway/secrets) · [SecretRef 凭据界面](/zh-CN/reference/secretref-credential-surface) · [安全](/zh-CN/gateway/security)

## 重新加载运行时快照

```bash
openclaw secrets reload
openclaw secrets reload --json
openclaw secrets reload --url ws://127.0.0.1:18789 --token <token>
```

使用 Gateway 网关 RPC 方法 `secrets.reload`。健康的所有者会独立刷新。仅当引用标识、提供商定义和完整的非密钥所有者契约保持不变时，符合条件的故障所有者才会变为陈旧状态；新增或发生变化的故障会变为冷状态。这种降级激活会成功，并报告 `warningCount`。严格模式故障或未映射故障会返回错误，并保留之前的活动快照。

选项：`--url <url>`、`--token <token>`、`--timeout <ms>`、`--json`。

## 审计

扫描 OpenClaw 状态以查找：

- 明文密钥存储
- 未解析的引用
- 优先级漂移（`auth-profiles.json` 凭据遮蔽 `openclaw.json` 引用）
- 生成的 `agents/*/agent/models.json` 残留（提供商 `apiKey` 值和敏感的提供商标头）
- 旧版残留（旧版身份验证存储条目、OAuth 提醒）

`.env` 扫描覆盖有效状态目录和包含活动配置的目录。当两个路径指向同一文件时，只扫描一次。

敏感提供商标头的检测基于名称启发式规则：它会标记名称与常见身份验证/凭据片段（`authorization`、`x-api-key`、`token`、`secret`、`password`、`credential`）匹配的标头。

```bash
openclaw secrets audit
openclaw secrets audit --check
openclaw secrets audit --json
openclaw secrets audit --allow-exec
```

报告结构：

- `status`：`clean | findings | unresolved`
- `resolution`：`refsChecked`、`skippedExecRefs`、`resolvabilityComplete`
- `summary`：`plaintextCount`、`unresolvedRefCount`、`shadowedRefCount`、`legacyResidueCount`
- 发现代码：`PLAINTEXT_FOUND`、`REF_UNRESOLVED`、`REF_SHADOWED`、`LEGACY_RESIDUE`

## 配置（交互式辅助工具）

以交互方式构建提供商和 SecretRef 变更，运行预检，并可选择应用：

```bash
openclaw secrets configure
openclaw secrets configure --plan-out /tmp/openclaw-secrets-plan.json
openclaw secrets configure --apply --yes
openclaw secrets configure --providers-only
openclaw secrets configure --skip-provider-setup
openclaw secrets configure --agent ops
openclaw secrets configure --json
```

流程：先设置提供商（添加/编辑/删除 `secrets.providers` 别名），然后映射凭据（选择字段并分配 `{source, provider, id}` 引用），最后执行预检并可选择应用。

标志：

- `--providers-only`：仅配置 `secrets.providers`，跳过凭据映射
- `--skip-provider-setup`：跳过提供商设置，将凭据映射到现有提供商
- `--agent <id>`：将 `auth-profiles.json` 目标发现和写入范围限定为一个智能体存储
- `--allow-exec`：允许在预检/应用期间执行 exec SecretRef 检查（可能会执行提供商命令）

`--providers-only` 和 `--skip-provider-setup` 不能组合使用。

注意：

- 需要交互式 TTY。
- 以 `openclaw.json` 中包含密钥的字段以及所选智能体范围的 `auth-profiles.json` 为目标；规范支持界面：[SecretRef 凭据界面](/zh-CN/reference/secretref-credential-surface)。
- 支持直接在选择器流程中创建新的 `auth-profiles.json` 映射。
- 在应用前运行预检解析。
- 生成的计划默认启用清除选项（`scrubEnv`、`scrubAuthProfilesForProviderTargets`、`scrubLegacyAuthJson`）。对于已清除的明文值，应用操作不可逆。
- `--plan-out` 拒绝创建 UTF-8 序列化形式超过 16 MiB（16,777,216 字节）的计划，这与 `apply --from` 输入限制一致。
- 如果未指定 `--apply`，CLI 仍会在预检后提示 `Apply this plan now?`。
- 指定 `--apply`（且未指定 `--yes`）时，CLI 会额外提示确认不可逆迁移。
- `--json` 会输出计划和预检报告，但仍需要交互式 TTY。

### Exec 提供商安全性

Homebrew 安装通常会在 `/opt/homebrew/bin/*` 下提供符号链接二进制文件。仅在受信任的软件包管理器路径确有需要时设置 `allowSymlinkCommand: true`，并与 `trustedDirs` 配合使用（例如 `["/opt/homebrew"]`）。在 Windows 上，如果无法验证提供商路径的 ACL，OpenClaw 会采用失败关闭策略；仅对于受信任的路径，可在该提供商上设置 `allowInsecurePath: true` 以绕过路径安全检查。

## 应用已保存的计划

```bash
openclaw secrets apply --from /tmp/openclaw-secrets-plan.json
openclaw secrets apply --from /tmp/openclaw-secrets-plan.json --allow-exec
openclaw secrets apply --from /tmp/openclaw-secrets-plan.json --dry-run
openclaw secrets apply --from /tmp/openclaw-secrets-plan.json --dry-run --allow-exec
openclaw secrets apply --from /tmp/openclaw-secrets-plan.json --json
```

`--dry-run` 会在不写入文件的情况下验证预检；试运行默认跳过 exec SecretRef 检查。除非指定 `--allow-exec`，否则写入模式会拒绝包含 exec SecretRef/提供商的计划。在任一模式下，使用 `--allow-exec` 可选择启用 exec 提供商检查/执行。

`--from` 必须指向不大于 16 MiB（16,777,216 字节）的常规文件。字节限制适用于包括空白字符在内的完整序列化文件。

`apply` 可能更新的内容：

- `openclaw.json`（SecretRef 目标 + 提供商更新插入/删除）
- `auth-profiles.json`（提供商目标清除）
- 旧版 `auth.json` 残留
- 有效状态目录和活动配置目录中的 `.env` 文件，针对值已迁移的已知密钥键

计划契约详情（允许的目标路径、验证规则、失败语义）：[密钥应用计划契约](/zh-CN/gateway/secrets-plan-contract)。

### 为什么没有回滚备份

`secrets apply` 有意不写入包含旧明文值的回滚备份。安全性来自严格预检和近似原子的应用操作，并在失败时尽力在内存中恢复。

## 示例

```bash
openclaw secrets audit --check
openclaw secrets configure
openclaw secrets audit --check
```

如果 `audit --check` 仍报告明文发现，请更新其余报告的目标路径，然后重新运行审计。

## 相关内容

- [CLI 参考](/zh-CN/cli)
- [密钥管理](/zh-CN/gateway/secrets)
- [Vault SecretRef](/zh-CN/plugins/vault)
