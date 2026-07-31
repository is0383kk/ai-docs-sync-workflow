---
read_when:
    - 生成或审查 `openclaw secrets apply` 计划
    - 调试 `Invalid plan target path` 错误
    - 了解目标类型和路径验证行为
summary: '`secrets apply` 计划契约：目标验证、路径匹配和 `auth-profiles.json` 目标范围'
title: 密钥应用计划契约
x-i18n:
    generated_at: "2026-07-26T06:43:44Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 71ee8afd958646930af4db3bbad08e033ff79da48890a989d72b361abcbda3bb
    source_path: gateway/secrets-plan-contract.md
    workflow: 16
---

本页定义了 `openclaw secrets apply` 强制执行的严格契约。如果目标不符合这些规则，应用操作会在修改任何文件之前失败。

## 计划文件要求

`openclaw secrets apply --from <plan.json>` 接受最大为 16 MiB（16,777,216 字节）的常规文件。此限制适用于完整的序列化文件，包括空白字符。目录、FIFO、设备文件以及超过此限制的文件都会在 JSON 解析或目标验证之前被拒绝。

`openclaw secrets configure --plan-out <plan.json>` 会在创建文件之前，对 UTF-8 序列化输出强制执行相同的限制。手写计划和外部计划生成器也必须确保序列化文件不超过此限制。

## 计划文件结构

`openclaw secrets apply --from <plan.json>` 需要一个由计划目标组成的 `targets` 数组：

```json5
{
  version: 1,
  protocolVersion: 1,
  targets: [
    {
      type: "models.providers.apiKey",
      path: "models.providers.openai.apiKey",
      pathSegments: ["models", "providers", "openai", "apiKey"],
      providerId: "openai",
      ref: { source: "env", provider: "default", id: "OPENAI_API_KEY" },
    },
    {
      type: "auth-profiles.api_key.key",
      path: "profiles.openai:default.key",
      pathSegments: ["profiles", "openai:default", "key"],
      agentId: "main",
      ref: { source: "env", provider: "default", id: "OPENAI_API_KEY" },
    },
  ],
}
```

`openclaw secrets configure` 会生成这种结构的计划。你也可以手写或编辑计划。

## 提供商更新插入和删除

计划还可以包含两个可选的顶层字段，用于在逐目标写入的同时修改 `secrets.providers` 映射：

- `providerUpserts` —— 以提供商别名为键的对象。每个值都是一个提供商定义（其结构与 `openclaw.json` 中 `secrets.providers.<alias>` 所接受的结构相同，例如 `exec` 或 `file` 提供商）。
- `providerDeletes` —— 要移除的提供商别名数组。

`providerUpserts` 在 `targets` 之前运行，因此 `target.ref.provider` 可以引用同一计划在 `providerUpserts` 中引入的提供商别名。如果没有这种执行顺序，引用 `openclaw.json` 中尚未配置的别名的计划会因 `provider "<alias>" is not configured` 而失败。

```json5
{
  version: 1,
  protocolVersion: 1,
  providerUpserts: {
    onepassword_anthropic: {
      source: "exec",
      command: "/usr/bin/op",
      args: ["read", "op://Vault/Anthropic/credential"],
    },
  },
  providerDeletes: ["legacy_unused_alias"],
  targets: [
    {
      type: "models.providers.apiKey",
      path: "models.providers.anthropic.apiKey",
      pathSegments: ["models", "providers", "anthropic", "apiKey"],
      providerId: "anthropic",
      ref: { source: "exec", provider: "onepassword_anthropic", id: "credential" },
    },
  ],
}
```

通过 `providerUpserts` 引入的 Exec 提供商仍受 [Exec 提供商同意行为](#exec-provider-consent-behavior)中的 Exec 同意规则约束：包含 Exec 提供商的计划在写入模式下需要 `--allow-exec`。

## 支持的目标范围

对于 [SecretRef 凭据表面](/zh-CN/reference/secretref-credential-surface)中支持的凭据路径，计划目标会被接受。

## 目标类型行为

`target.type` 必须是可识别的目标类型，并且规范化后的 `target.path` 必须与该类型注册的路径结构匹配。

除了规范类型名称外，某些目标类型还接受 `target.type` 作为现有计划的兼容性别名：

| 规范类型                             | 接受的别名                                      |
| ------------------------------------ | ----------------------------------------------- |
| `models.providers.apiKey`            | `models.providers.*.apiKey`                     |
| `skills.entries.apiKey`              | `skills.entries.*.apiKey`                       |
| `channels.googlechat.serviceAccount` | `channels.googlechat.accounts.*.serviceAccount` |

## 路径验证规则

每个目标都会按照以下所有规则进行验证：

- `type` 必须是可识别的目标类型。
- `path` 必须是非空的点分路径。
- `pathSegments` 可以省略。如果提供，它规范化后必须与 `path` 的路径完全相同。
- 禁止使用以下段：`__proto__`、`prototype`、`constructor`。
- 规范化后的路径必须与目标类型注册的路径结构匹配。
- 如果设置了 `providerId` 或 `accountId`，它必须与路径中编码的 ID 匹配。
- `auth-profiles.json` 目标需要 `agentId`。
- 创建新的 `auth-profiles.json` 映射时，请包含 `authProfileProvider`。

## 失败行为

如果目标验证失败，应用操作会退出并显示类似以下错误：

```text
models.providers.apiKey 的计划目标路径无效：models.providers.openai.baseUrl
```

无效计划不会提交任何写入：目标解析和路径验证会在接触任何文件之前运行。另外，有效计划开始写入后，应用操作会先为每个涉及的文件创建快照；如果同一次运行中的后续写入失败，则会恢复这些快照，因此部分写入绝不会导致配置、身份验证配置文件或环境变量状态不同步。

## Exec 提供商同意行为

- `--dry-run` 默认跳过 Exec SecretRef 检查。
- 除非设置了 `--allow-exec`，否则包含 Exec SecretRef/提供商的计划在写入模式下会被拒绝。
- 验证或应用包含 Exec 的计划时，请在试运行和写入命令中都传入 `--allow-exec`。

## 运行时和审计范围说明

- 仅含引用的 `auth-profiles.json` 条目（`keyRef`/`tokenRef`）包含在运行时凭据解析和审计覆盖范围内。
- `secrets apply` 会写入受支持的 `openclaw.json` 目标和受支持的 `auth-profiles.json` 目标，并执行三个默认启用的可选清理过程：`scrubEnv`（从有效状态目录和活动配置目录中的 `.env` 文件移除已迁移的明文值）、`scrubAuthProfilesForProviderTargets`（清除计划刚迁移的提供商在 `auth-profiles.json` 中残留的明文/未使用引用）以及 `scrubLegacyAuthJson`（从旧版 `auth.json` 存储中删除已迁移的 `api_key` 条目）。在计划中将 `options.scrubEnv`、`options.scrubAuthProfilesForProviderTargets` 或 `options.scrubLegacyAuthJson` 中的任意一项设置为 `false`，即可跳过对应过程。

## 操作员检查

```bash
# 验证计划但不写入
openclaw secrets apply --from /tmp/openclaw-secrets-plan.json --dry-run

# 然后实际应用
openclaw secrets apply --from /tmp/openclaw-secrets-plan.json

# 对于包含 Exec 的计划，请在两种模式下都显式选择启用
openclaw secrets apply --from /tmp/openclaw-secrets-plan.json --dry-run --allow-exec
openclaw secrets apply --from /tmp/openclaw-secrets-plan.json --allow-exec
```

如果应用操作失败并显示目标路径无效消息，请使用 `openclaw secrets configure` 重新生成计划，或将目标路径修正为上面支持的结构。

## 相关文档

- [密钥管理](/zh-CN/gateway/secrets)
- [CLI `secrets`](/zh-CN/cli/secrets)
- [SecretRef 凭据表面](/zh-CN/reference/secretref-credential-surface)
- [配置参考](/zh-CN/gateway/configuration-reference)
