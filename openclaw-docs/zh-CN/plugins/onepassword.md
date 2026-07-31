---
read_when:
    - 你希望智能体请求经过筛选的 1Password 机密信息
    - 你需要按每个机密分别配置审批策略并保留审计历史记录
    - 你正在为 OpenClaw 配置 1Password 服务账户
summary: 使用可选的 1Password 插件作为经审计的智能体机密信息代理服务
title: 1Password 密钥代理服务
x-i18n:
    generated_at: "2026-07-26T06:56:35Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 255ab4fd2c63754fef29d3ea87dcedc9ca2bd2f34bec1f81139e2ce5b6acdba2
    source_path: plugins/onepassword.md
    workflow: 16
---

# 1Password 密钥代理

内置的 `onepassword` 插件为智能体提供一个受策略控制的工具，用于
读取一组精选的 1Password 字段。该插件默认禁用，并且在
`plugins.entries.onepassword.config` 存在之前不会执行任何操作。

这是一个智能体工具，而不是 SecretRef 提供商。它不会注入环境变量，
也不会解析 OpenClaw 配置密钥。

## 安全模型

- 仅支持服务账户身份验证。令牌保留在本地凭据
  文件中，且绝不会在 `openclaw.json` 中被接受。
- 仅限精选注册表。智能体可以列出已配置的 slug，但该插件绝不会
  枚举 1Password 保险库。
- 每个 slug 均采用 `auto`、`approve` 或 `deny` 策略。
- 批准授权会过期。缓存值绝不会绕过当前策略。
- 每次访问尝试都会记录在 OpenClaw 的共享 SQLite 状态中。审计
  行包括提供的原因；原因中不得包含敏感信息。代理绝不会
  将获取的值或服务令牌复制到审计行中。
- 当前工具执行结束后，OpenClaw 所拥有的会话记录持久化机制
  会将成功的 `get` 值替换为已遮盖的元数据。
- 该值在此次执行期间对模型可见。如果模型将其复制到
  后续工具调用或回复中，则该单独记录不受此插件的
  持久化钩子管辖。请严格限制策略范围，且不要要求模型复述
  该值。
- 每次缓存未命中时，插件会调用一次 `op`。它不会重试速率限制或
  其他失败。
- 每次 `op` 调用都在最小化环境中运行，并禁用 1Password
  桌面应用集成（`OP_LOAD_DESKTOP_APP_SETTINGS=false`、
  `OP_BIOMETRIC_UNLOCK_ENABLED=false`），因此 Gateway 网关主机上安装的 1Password 应用绝不会
  触发生物识别或 macOS 权限对话框。

仅向服务账户授予对插件配置中所注册保险库和项目的读取权限。

## 开始之前

你需要：

- 在 Gateway 网关主机上安装 1Password CLI（`op`）
- 一个有权访问所选项目的 1Password 服务账户
- 一个专用的服务账户令牌文件

启用内置插件：

```bash
openclaw plugins enable onepassword
```

在 OpenClaw 状态目录下创建令牌目录和文件：

```bash
mkdir -p ~/.openclaw/credentials/onepassword
chmod 700 ~/.openclaw/credentials/onepassword
printf '%s' "$OP_SERVICE_ACCOUNT_TOKEN" > \
  ~/.openclaw/credentials/onepassword/service-account-token
chmod 600 ~/.openclaw/credentials/onepassword/service-account-token
unset OP_SERVICE_ACCOUNT_TOKEN
```

设置 `OPENCLAW_STATE_DIR` 后，请将 `~/.openclaw` 替换为该目录。
当令牌文件可被组用户或其他用户读取或写入时，插件会警告一次。

## 配置已注册的密钥

将插件配置添加到 `openclaw.json`：

```jsonc
{
  "plugins": {
    "entries": {
      "onepassword": {
        "enabled": true,
        "config": {
          "vault": "Automation",
          "defaultPolicy": "approve",
          "cacheTtlSeconds": 300,
          "grantTtlHours": 720,
          "opTimeoutMs": 15000,
          "items": {
            "repository-token": {
              "item": "Repository automation token",
              "field": "credential",
              "policy": "approve",
              "description": "Token for repository automation",
            },
            "model-key": {
              "item": "Model provider key",
              "vault": "Agent credentials",
              "policy": "auto",
            },
          },
        },
      },
    },
  },
}
```

slug 使用小写字母、数字和连字符，以字母或
数字开头，且最多包含 64 个字符。一个注册表最多可包含 32 个
slug；描述最多可包含 200 个字符。`field` 接受一个字段
标签或 ID，不得包含逗号，默认值为 `credential`。
项目级 `vault` 会覆盖默认保险库。`opBin` 可设置
`op` 可执行文件的绝对路径；否则插件会从 `PATH` 解析 `op`。
项目标题不得以连字符开头。

## 使用智能体工具

工具名称为 `onepassword`。

列出已注册的 slug：

```json
{ "action": "list" }
```

结果仅包含 slug、描述、策略，以及长期
授权是否有效。结果绝不会包含密钥值，也不会查询 1Password。

请求一个密钥：

```json
{
  "action": "get",
  "slug": "repository-token",
  "reason": "Authenticate the requested repository operation"
}
```

`reason` 为必填项，不能为空，且最多包含 300 个字符。
成功的 `get` 会返回该值以及已配置的 slug、项目标题和
字段标签。

工具架构还声明了一个内部 `authorizationNonce` 参数。
策略层评估请求后会注入该参数，以便将授权
传递给正在执行的工具调用。切勿手动设置：策略钩子会覆盖
任何提供的值，而未知值会导致请求失败。

## 策略层级和批准

- `auto`：立即获取并审计请求。
- `deny`：阻止并审计请求。
- `approve`：使用未过期的长期授权，或请求人工选择仅允许一次、
  始终允许或拒绝。

仅允许一次只授权当前工具调用。始终允许会将该智能体和 slug 的长期
授权写入 SQLite；其他智能体必须单独获得
批准。仅当调用方具有具体智能体身份时，OpenClaw 才会提供始终允许
选项。授权将在 `grantTtlHours` 后过期，默认值为 720 小时。
未解决或超时的批准会拒绝请求；批准等待时间最长为
600 秒。插件最多保留 1,024 个长期授权；达到该
上限时，最早的授权会被逐出，其智能体必须为下一次访问重新获得批准。

每个经过评估的授权只能使用一次，并通过共享 SQLite 状态传递给正在执行的工具
调用，因此当 Gateway 网关进程中有多个
插件实例处于活动状态时，该交接机制仍然有效。未使用的授权会在
600 秒批准窗口结束后过期。

内存缓存默认为 300 秒，其容量受已配置的
slug 注册表限制。将 `cacheTtlSeconds` 设置为 `0` 可将其禁用。每次缓存查找前都会评估策略，
缓存命中也会被审计。运行时配置重新加载会在每个策略和执行边界生效；禁用插件，或
删除、拒绝或重新指定 slug，都会使待处理的授权和
缓存值失效。

## 检查状态和审计历史记录

显示就绪状态和注册表计数：

```bash
openclaw onepassword status
```

此命令会报告令牌文件是否存在、`op` 是否已解析及其路径、
已注册项目数，以及按策略划分的数量。它绝不会读取或输出
令牌或密钥值。

显示最近的 50 条审计行：

```bash
openclaw onepassword audit
openclaw onepassword audit --limit 100
```

审计行按从新到旧的顺序排列，并显示时间戳、智能体、slug、结果、尝试失败时的
`errorCode`，以及截断后的原因。原因会按
原样存储；代理绝不会将获取的值添加到审计日志中。

## 1Password CLI 行为

每次缓存未命中都会使用已配置的项目、保险库、精确
字段选择器、JSON 输出、有限的超时时间和 `--cache=false` 运行 `op item get`。子进程
仅接收该字段，而不是完整项目。子进程环境中仅存在
`OP_SERVICE_ACCOUNT_TOKEN` 和 `HOME`。

插件只尝试一次。对于 `RATE_LIMITED` 错误，应先等待，
再由智能体发起后续请求；插件不会创建自动重试
循环。

## 错误代码

失败的尝试会在工具结果和审计
行中携带一个封闭式错误代码。

1Password 访问错误：

| 代码              | 含义                                                          |
| ----------------- | ---------------------------------------------------------------- |
| `TOKEN_MISSING`   | 令牌文件缺失或为空                                   |
| `OP_NOT_FOUND`    | 无法解析 `op` 二进制文件                                |
| `ITEM_NOT_FOUND`  | 配置的项目不在保险库中                              |
| `FIELD_NOT_FOUND` | 配置的字段不在项目中；会列出可用标签 |
| `RATE_LIMITED`    | 已达到 1Password 服务账户速率限制                     |
| `AUTH_FAILED`     | 服务账户身份验证失败                            |
| `TIMEOUT`         | `op` 超过 `opTimeoutMs`                                      |
| `OP_ERROR`        | 任何其他 `op` 失败或无效输出                         |

策略和验证错误：

| 代码                                               | 含义                                                                      |
| -------------------------------------------------- | ---------------------------------------------------------------------------- |
| `INVALID_ACTION`、`INVALID_REASON`、`INVALID_SLUG` | 请求未通过输入验证                                              |
| `UNKNOWN_SLUG`                                     | slug 不在已配置的注册表中                                       |
| `TOOL_CALL_ID_MISSING`                             | 调用到达时没有工具调用 ID                                          |
| `POLICY_NOT_EVALUATED`                             | 此调用没有匹配的授权；请求未获策略批准 |
| `POLICY_CHANGED`                                   | 配置在批准与执行之间发生变化                                |
| `GRANT_EXPIRED`                                    | 长期授权在执行前失效                                       |
| `APPROVAL_CANCELLED`                               | 批准待处理期间运行被中止                           |
