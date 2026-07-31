---
read_when:
    - 为提供商凭据和 `auth-profiles.json` 引用配置 SecretRefs
    - 在生产环境中安全地执行密钥重载、审计、配置和应用操作
    - 理解启动快速失败、非活动界面过滤和最近一次已知正常行为
sidebarTitle: Secrets management
summary: 密钥管理：SecretRef 契约、运行时快照行为和安全的单向清除
title: 密钥管理
x-i18n:
    generated_at: "2026-07-26T06:46:15Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: d10989ebbce367c68d28768244d4e3649028af5ab63c9523974352c270a3c55e
    source_path: gateway/secrets.md
    workflow: 16
---

OpenClaw 支持增量采用 SecretRef，因此受支持的凭据无需以明文形式存储在配置中。

<Note>
仍可使用明文。SecretRef 按凭据选择启用。
</Note>

<Warning>
如果明文凭据位于智能体可检查的文件中，包括 `openclaw.json`、`auth-profiles.json`、`.env` 或生成的 `agents/*/agent/models.json` 文件，智能体仍可读取这些凭据。只有在迁移所有受支持的凭据，并且 `openclaw secrets audit --check` 报告没有明文残留后，SecretRef 才能缩小此类本地影响范围。
</Warning>

## 运行时模型

- 密钥会在激活期间预先解析到内存中的运行时快照，而不是在请求路径上延迟解析。
- Gateway 网关冷启动时，如果已知的非 Gateway 网关所有者支持隔离，则可将可重试的 SecretRef 故障隔离到该所有者。已映射的所有者类别包括模型提供商和 Skills、媒体/TTS/定时任务提供商、符合条件的身份验证配置文件、按智能体配置的记忆、沙箱 SSH、渠道账号，以及清单中声明的插件路由。Gateway 网关会启动，将该所有者记录为“已配置但不可用”，并发出经过脱敏的降级警告。Gateway 网关入口身份验证、结构无效的引用或解析值、故障时关闭的所有者，以及运行时所有者未映射的引用仍会导致启动失败。
- 重新加载会独立验证每个已映射的所有者，然后以原子方式发布单个快照。健康的所有者会刷新。只有当引用标识、提供商定义和完整的非密钥所有者契约均未改变时，符合条件但验证失败的所有者才会保留其最后一个已知良好值并变为陈旧状态；发生变化或新增但验证失败的所有者会进入冷状态。严格故障会拒绝重新加载，并保留活动快照。
- 策略违规（例如，将 OAuth 模式的身份验证配置文件与 SecretRef 输入结合使用）会在运行时切换前导致激活失败。
- 运行时请求只读取活动的内存快照。模型提供商的 SecretRef 凭据会以进程本地哨兵值的形式经过身份验证存储和流选项，直到出站时才注入。出站传递路径（Discord 回复/线程传递、Telegram 操作发送）也会读取该快照，不会在每次发送时重新解析引用。

这样可以避免密钥提供商中断影响高频请求路径。

Gateway 网关入口保护、结构无效的配置或解析值、策略违规以及未知所有权仍会按故障时关闭方式处理。被隔离的所有者绝不会回退到优先级更低的凭据来源。

## 出站时注入（哨兵值）

对于由 SecretRef 支持的模型提供商凭据，OpenClaw 会在解析模型身份验证时生成一个不透明的进程本地哨兵值。因此，身份验证存储、流选项、SDK 配置、日志、错误对象和大多数运行时内省所看到的是类似 `oc-sent-v1-...` 的值，而不是提供商凭据。受保护的模型 fetch 和托管本地提供商健康探测会在每个请求离开进程之前，立即替换 URL 和标头值中的已知哨兵值。

未知的哨兵格式值会在发生网络活动前按故障时关闭方式处理。OpenClaw 会拒绝发送请求，而不是将未解析的哨兵值转发给提供商。作为纵深防御措施，解析后的密钥值也会注册用于精确值日志脱敏。

提供商适配器使用其 SDK 所支持的最靠后的注入点：

- 支持自定义 fetch 选项的 SDK 会接收 OpenClaw 的受保护 fetch，因此 SDK 会保留哨兵值。
- 不支持自定义 fetch 选项的 SDK 会在构造客户端之前立即解包哨兵值。插件所有的提供商流和 agent harness 会在核心所有的最终交接点解包，因为这些传输机制不共享 OpenClaw 的受保护 fetch。

哨兵值可以减少模型调用链中的明文暴露，但并不提供进程隔离。真实值仍存在于同一进程的内存中，并会出现在最终适配器边界。未通过 SecretRef 配置的明文环境凭据仍是明文，不属于此机制的保护范围。

设置 `OPENCLAW_SECRET_SENTINELS=off`（也接受 `0` 或 `false`，不区分大小写）可在事件响应或兼容性故障排除期间禁用哨兵值生成。此终止开关不会禁用精确值脱敏注册。

## 智能体访问边界

SecretRef 可防止凭据持久化到配置和生成的模型文件中，但它并不是进程隔离边界。如果明文凭据仍存储在智能体可读取的磁盘路径中，则仍可通过文件或 shell 工具读取，从而绕过 API 层脱敏。

对于将智能体可访问文件纳入保护范围的生产部署，只有满足以下所有条件时，才应视为迁移完成：

- 受支持的凭据使用 SecretRef，而不是明文值。
- 已从 `openclaw.json`、`auth-profiles.json`、`.env` 和生成的 `models.json` 文件中清除旧版明文残留。
- 迁移后，`openclaw secrets audit --check` 的检查结果无异常。
- 任何剩余的不受支持或轮换凭据均受操作系统隔离、容器隔离或外部凭据代理保护。

因此，审计/配置/应用工作流是安全迁移的门槛，而不只是一个便捷辅助工具。

<Warning>
SecretRef 无法使任意可读文件变得安全。备份、复制的配置、旧的生成模型目录以及不受支持的凭据类别，在被删除、移出智能体信任边界或单独隔离之前，仍属于生产密钥。
</Warning>

## 活动表面筛选

仅对实际处于活动状态的表面验证 SecretRef：

- **已启用的表面**：对于已映射且可隔离的所有者，可重试故障会进入冷降级或陈旧降级状态。严格、故障时关闭、Gateway 网关必需或未映射的故障会阻止启动/重新加载。
- **非活动表面**：未解析的引用不会阻止启动/重新加载；它们会发出非致命的 `SECRETS_REF_IGNORED_INACTIVE_SURFACE` 诊断。

<Accordion title="非活动表面示例">
- 已禁用的渠道/账号条目。
- 未被任何已启用账号继承的顶层渠道凭据。
- 已禁用的工具/功能表面。
- 未由 `tools.web.search.provider` 选中的 Web 搜索提供商专用密钥。在自动模式（未设置提供商）下，会按优先级依次查询密钥以进行自动检测，直到某个密钥成功解析；完成选择后，未选中提供商的密钥处于非活动状态。
- 只有当有效的沙箱后端为 `ssh`、沙箱模式不是 `off`，并且对象是默认智能体或已启用的智能体时，沙箱 SSH 身份验证材料（`agents.defaults.sandbox.ssh.identityData`、`certificateData`、`knownHostsData` 以及按智能体配置的覆盖项）才处于活动状态。
- 满足以下任一条件时，`gateway.remote.token` / `gateway.remote.password` SecretRef 处于活动状态：
  - `gateway.mode=remote`
  - 已配置 `gateway.remote.url`
  - `gateway.tailscale.mode` 为 `serve` 或 `funnel`
  - 在没有这些远程表面的本地模式下：当令牌身份验证可能胜出且未配置环境/身份验证令牌时，`gateway.remote.token` 处于活动状态；只有当密码身份验证可能胜出且未配置环境/身份验证密码时，`gateway.remote.password` 才处于活动状态。
- 设置 `OPENCLAW_GATEWAY_TOKEN` 后，`gateway.auth.token` SecretRef 对启动身份验证解析处于非活动状态，因为该运行时会优先采用环境令牌输入。

</Accordion>

## Gateway 网关身份验证表面诊断

在 `gateway.auth.token`、`gateway.auth.password`、`gateway.remote.token` 或 `gateway.remote.password` 上设置 SecretRef 后，Gateway 网关启动/重新加载会使用代码 `SECRETS_GATEWAY_AUTH_SURFACE` 记录表面状态：

- `active`：SecretRef 是有效身份验证表面的一部分，必须成功解析。
- `inactive`：另一个身份验证表面胜出，或者远程身份验证已禁用/未处于活动状态。

日志条目包含活动表面策略所采用的原因。

## 新手引导引用预检

在交互式新手引导中，选择 SecretRef 存储后，会在保存前运行预检验证：

- 环境引用：验证环境变量名称，并确认设置期间可见的值非空。
- 提供商引用（`file` 或 `exec`）：验证提供商选择、解析 `id`，并检查解析值的类型。
- 快速开始流程：当 `gateway.auth.token` 已是 SecretRef 时，新手引导会在探测/仪表板引导启动前，使用相同的快速失败门槛解析该引用（适用于 `env`、`file` 和 `exec` 引用）。

验证失败时会显示错误，并允许你重试。

## SecretRef 契约

所有位置均使用同一种对象结构：

```json5
{ source: "env" | "file" | "exec", provider: "default", id: "..." }
```

<Tabs>
  <Tab title="env">
    ```json5
    { source: "env", provider: "default", id: "OPENAI_API_KEY" }
    ```

    SecretInput 字段也接受简写字符串：

    ```json5
    "${OPENAI_API_KEY}"
    "$OPENAI_API_KEY"
    ```

    验证：

    - `provider` 必须匹配 `^[a-z][a-z0-9_-]{0,63}$`
    - `id` 必须匹配 `^[A-Z][A-Z0-9_]{0,127}$`

  </Tab>
  <Tab title="file">
    ```json5
    { source: "file", provider: "filemain", id: "/providers/openai/apiKey" }
    ```

    验证：

    - `provider` 必须匹配 `^[a-z][a-z0-9_-]{0,63}$`
    - `id` 必须是绝对 JSON 指针（`/...`）；对于 `singleValue` 提供商，也可以是字面值 `value`
    - 分段中的 RFC 6901 转义：`~` 转换为 `~0`，`/` 转换为 `~1`

  </Tab>
  <Tab title="exec">
    ```json5
    { source: "exec", provider: "vault", id: "providers/openai/apiKey#value" }
    ```

    验证：

    - `provider` 必须匹配 `^[a-z][a-z0-9_-]{0,63}$`
    - `id` 必须匹配 `^[A-Za-z0-9][A-Za-z0-9._:/#-]{0,255}$`（支持 `secret#json_key` 等选择器）
    - `id` 不得包含作为斜杠分隔路径段的 `.` 或 `..`（例如，`a/../b` 会被拒绝）

  </Tab>
</Tabs>

## 提供商配置

在 `secrets.providers` 下定义提供商：

```json5
{
  secrets: {
    providers: {
      default: { source: "env" },
      filemain: {
        source: "file",
        path: "~/.openclaw/secrets.json",
        mode: "json", // 或 "singleValue"
      },
      vault: {
        source: "exec",
        command: "/usr/local/bin/openclaw-vault-resolver",
        args: ["--profile", "prod"],
        passEnv: ["PATH", "VAULT_ADDR"],
        jsonOnly: true,
      },
      "team-secrets": {
        source: "exec",
        pluginIntegration: {
          pluginId: "acme-secrets",
          integrationId: "secret-store",
        },
      },
    },
    defaults: {
      env: "default",
      file: "filemain",
      exec: "vault",
    },
  },
}
```

<Accordion title="环境提供商">
- 可通过 `allowlist` 配置可选的精确名称允许列表。
- 环境值缺失或为空会导致解析失败。

</Accordion>

<Accordion title="文件提供商">
- 读取 `path` 处的本地文件。
- `mode: "json"`（默认值）要求 JSON 对象负载，并将 `id` 解析为 JSON 指针。
- `mode: "singleValue"` 要求引用 ID 为 `"value"`，并返回原始文件内容（移除末尾换行符）。
- 路径必须通过所有权/权限检查；`timeoutMs`（默认值为 5000）和 `maxBytes`（默认值为 1 MiB）会限制读取操作。
- Windows 按故障时关闭方式处理：如果无法验证该路径的 ACL，则解析失败。仅对于受信任路径，可在该提供商上设置 `allowInsecurePath: true` 以绕过检查。

</Accordion>

<Accordion title="Exec 提供商">
- 直接运行配置的绝对二进制路径，不使用 shell。
- 默认情况下，`command` 必须是常规文件，而不能是符号链接。设置 `allowSymlinkCommand: true` 可允许符号链接命令路径（例如 Homebrew shim），并将其与 `trustedDirs`（例如 `["/opt/homebrew"]`）配合使用，以便只有软件包管理器路径符合条件。
- 支持 `timeoutMs`（默认值为 5000）、`noOutputTimeoutMs`（默认值等于 `timeoutMs`）、`maxOutputBytes`（默认值为 1 MiB）、`env`/`passEnv` 允许列表以及 `trustedDirs`。
- `jsonOnly` 默认为 `true`。使用 `jsonOnly: false` 且仅请求一个 id 时，普通的非 JSON stdout 可作为该 id 的值接受。
- Windows 采用失败时关闭策略：如果无法验证命令路径的 ACL，解析将失败。仅对于受信任的路径，可在该提供商上设置 `allowInsecurePath: true` 以绕过此检查。
- 由插件管理的 Exec 提供商可以使用 `pluginIntegration`，而无需复制 `command`/`args`。OpenClaw 会在启动/重新加载期间，从已安装的插件清单中解析当前命令详情；如果插件被禁用、移除、不受信任或不再声明该集成，则该提供商上处于活动状态的 SecretRef 将以失败时关闭方式处理。

请求负载（stdin）：

```json
{ "protocolVersion": 1, "provider": "vault", "ids": ["providers/openai/apiKey"] }
```

响应负载（stdout）：

```jsonc
{ "protocolVersion": 1, "values": { "providers/openai/apiKey": "<openai-api-key>" } } // pragma: allowlist secret
```

可选的逐 id 错误：

```json
{
  "protocolVersion": 1,
  "values": {},
  "errors": { "providers/openai/apiKey": { "code": "NOT_FOUND" } }
}
```

`code` 是可选的机器可读诊断信息。OpenClaw 会将可识别的
代码 `NOT_FOUND` 和 `AMBIGUOUS_DUPLICATE_KEY` 与提供商及引用 id 一并显示。其他
代码和自由格式字段（例如 `message`）会出于 protocol-v1 兼容性而被接受，
但不会显示，因为解析器输出可能包含凭据材料。

</Accordion>

## 基于文件的 API 密钥

不要在配置的 `env` 块中放置 `file:...` 字符串。该块按字面值处理且不可覆盖，因此永远不会在其中解析 `file:...`。

请改为在受支持的凭据字段上使用文件 SecretRef：

```json5
{
  secrets: {
    providers: {
      xai_key_file: {
        source: "file",
        path: "~/.openclaw/secrets/xai-api-key.txt",
        mode: "singleValue",
      },
    },
  },
  models: {
    providers: {
      xai: {
        apiKey: { source: "file", provider: "xai_key_file", id: "value" },
      },
    },
  },
}
```

对于 `mode: "singleValue"`，SecretRef `id` 为 `"value"`。对于 `mode: "json"`，请使用绝对 JSON 指针，例如 `"/providers/xai/apiKey"`。

有关接受 SecretRef 的字段，请参阅 [SecretRef 凭据适用范围](/zh-CN/reference/secretref-credential-surface)。

## Exec 集成示例

有关服务账户、内置 Agent Skills 和故障排除的 1Password 专用指南，请参阅 [1Password](/zh-CN/gateway/1password)。

<AccordionGroup>
  <Accordion title="1Password CLI">
    ```json5
    {
      secrets: {
        providers: {
          onepassword_openai: {
            source: "exec",
            command: "/opt/homebrew/bin/op",
            allowSymlinkCommand: true, // Homebrew 符号链接二进制文件需要此项
            trustedDirs: ["/opt/homebrew"],
            args: ["read", "op://Personal/OpenClaw QA API Key/password"],
            passEnv: ["HOME"],
            jsonOnly: false,
          },
        },
      },
      models: {
        providers: {
          openai: {
            baseUrl: "https://api.openai.com/v1",
            models: [{ id: "gpt-5", name: "gpt-5" }],
            apiKey: { source: "exec", provider: "onepassword_openai", id: "value" },
          },
        },
      },
    }
    ```
  </Accordion>
  <Accordion title="Bitwarden Secrets Manager (`bws`)">
    使用解析器包装器，将 SecretRef id 映射到 Bitwarden Secrets Manager 项目键。仓库中包含 `scripts/secrets/openclaw-bws-resolver.mjs`；请将其安装或复制到运行 Gateway 网关的主机上的受信任绝对路径。

    要求：

    - Gateway 网关主机上已安装 Bitwarden Secrets Manager CLI（`bws`）。
    - Gateway 网关服务可以使用 `BWS_ACCESS_TOKEN`。
    - 将 `PATH` 传递给解析器，或将 `BWS_BIN` 设置为 `bws` 二进制文件的绝对路径。
    - 使用自行托管的 Bitwarden 实例时，在环境中设置 `BWS_SERVER_URL`。

    ```json5
    {
      secrets: {
        providers: {
          bws: {
            source: "exec",
            command: "/usr/local/bin/openclaw-bws-resolver.mjs",
            passEnv: ["BWS_ACCESS_TOKEN", "BWS_SERVER_URL", "PATH", "BWS_BIN"],
            jsonOnly: true,
          },
        },
      },
      models: {
        providers: {
          openai: {
            baseUrl: "https://api.openai.com/v1",
            models: [{ id: "gpt-5", name: "gpt-5" }],
            apiKey: {
              source: "exec",
              provider: "bws",
              id: "openclaw/providers/openai/apiKey",
            },
          },
        },
      },
    }
    ```

    解析器会批量处理请求的 id、运行 `bws secret list`，并返回匹配密钥的 `key` 字段值。请使用符合 Exec SecretRef id 约定的键，例如 `openclaw/providers/openai/apiKey`；使用下划线的环境变量样式键会在解析器运行前被拒绝。如果多个可见的 Bitwarden 密钥具有请求的同一键，解析器会将该 id 标记为存在歧义并使其失败，而不会进行猜测。更新配置后，请验证解析器路径：

    ```bash
    openclaw secrets audit --allow-exec
    ```

  </Accordion>
  <Accordion title="HashiCorp Vault CLI">
    ```json5
    {
      secrets: {
        providers: {
          vault_openai: {
            source: "exec",
            command: "/opt/homebrew/bin/vault",
            allowSymlinkCommand: true, // Homebrew 符号链接二进制文件需要此项
            trustedDirs: ["/opt/homebrew"],
            args: ["kv", "get", "-field=OPENAI_API_KEY", "secret/openclaw"],
            passEnv: ["VAULT_ADDR", "VAULT_TOKEN"],
            jsonOnly: false,
          },
        },
      },
      models: {
        providers: {
          openai: {
            baseUrl: "https://api.openai.com/v1",
            models: [{ id: "gpt-5", name: "gpt-5" }],
            apiKey: { source: "exec", provider: "vault_openai", id: "value" },
          },
        },
      },
    }
    ```
  </Accordion>
  <Accordion title="password-store (`pass`)">
    使用一个小型解析器包装器，将 SecretRef id 直接映射到 `pass` 条目。将其保存为绝对路径下的可执行文件，该路径必须通过 Exec 提供商的路径检查，例如 `/usr/local/bin/openclaw-pass-resolver`。`#!/usr/bin/env node` shebang 会从解析器进程的 `PATH` 中解析 `node`，因此请在 `passEnv` 中包含 `PATH`。如果 `pass` 不在该 `PATH` 中，请在父环境中设置 `PASS_BIN`，并同样将其包含在 `passEnv` 中：

    ```js
    #!/usr/bin/env node
    const { spawnSync } = require("node:child_process");

    let stdin = "";
    process.stdin.setEncoding("utf8");
    process.stdin.on("data", (chunk) => {
      stdin += chunk;
    });
    process.stdin.on("error", (err) => {
      process.stderr.write(`${err.message}\n`);
      process.exit(1);
    });
    process.stdin.on("end", () => {
      let request;
      try {
        request = JSON.parse(stdin || "{}");
      } catch (err) {
        process.stderr.write(`Failed to parse request: ${err.message}\n`);
        process.exit(1);
      }

      const passBin = process.env.PASS_BIN || "pass";
      const values = {};
      const errors = {};

      for (const id of request.ids ?? []) {
        const result = spawnSync(passBin, ["show", id], { encoding: "utf8" });
        if (result.status === 0) {
          values[id] = result.stdout.split(/\r?\n/, 1)[0] ?? "";
        } else {
          errors[id] = { message: (result.stderr || `pass exited ${result.status}`).trim() };
        }
      }

      process.stdout.write(JSON.stringify({ protocolVersion: 1, values, errors }));
    });
    ```

    然后配置 Exec 提供商，并将 `apiKey` 指向 `pass` 条目路径：

    ```json5
    {
      secrets: {
        providers: {
          pass_store: {
            source: "exec",
            command: "/usr/local/bin/openclaw-pass-resolver",
            passEnv: ["PATH", "HOME", "GNUPGHOME", "GPG_TTY", "PASSWORD_STORE_DIR", "PASS_BIN"],
            jsonOnly: true,
          },
        },
      },
      models: {
        providers: {
          openai: {
            baseUrl: "https://api.openai.com/v1",
            models: [{ id: "gpt-5", name: "gpt-5" }],
            apiKey: {
              source: "exec",
              provider: "pass_store",
              id: "openclaw/providers/openai/apiKey",
            },
          },
        },
      },
    }
    ```

    将密钥保留在 `pass` 条目的第一行，或者自定义包装器，使其改为返回完整的 `pass show` 输出。更新配置后，请同时验证静态审计和 Exec 解析器路径：

    ```bash
    openclaw secrets audit --check
    openclaw secrets audit --allow-exec
    ```

  </Accordion>
  <Accordion title="sops">
    ```json5
    {
      secrets: {
        providers: {
          sops_openai: {
            source: "exec",
            command: "/opt/homebrew/bin/sops",
            allowSymlinkCommand: true, // Homebrew 符号链接二进制文件需要此项
            trustedDirs: ["/opt/homebrew"],
            args: ["-d", "--extract", '["providers"]["openai"]["apiKey"]', "/path/to/secrets.enc.json"],
            passEnv: ["SOPS_AGE_KEY_FILE"],
            jsonOnly: false,
          },
        },
      },
      models: {
        providers: {
          openai: {
            baseUrl: "https://api.openai.com/v1",
            models: [{ id: "gpt-5", name: "gpt-5" }],
            apiKey: { source: "exec", provider: "sops_openai", id: "value" },
          },
        },
      },
    }
    ```
  </Accordion>
</AccordionGroup>

## MCP 服务器环境变量

通过 `plugins.entries.acpx.config.mcpServers` 配置的 MCP 服务器环境变量接受 SecretInput，从而避免在明文配置中存放 API 密钥和令牌：

```json5
{
  plugins: {
    entries: {
      acpx: {
        enabled: true,
        config: {
          mcpServers: {
            github: {
              command: "npx",
              args: ["-y", "@modelcontextprotocol/server-github"],
              env: {
                GITHUB_PERSONAL_ACCESS_TOKEN: {
                  source: "env",
                  provider: "default",
                  id: "MCP_GITHUB_PAT",
                },
              },
            },
          },
        },
      },
    },
  },
}
```

明文字符串值仍然有效。类似 `${MCP_SERVER_API_KEY}` 的环境变量模板引用和 SecretRef 对象会在 Gateway 网关激活期间、MCP 服务器进程生成之前解析。与其他 SecretRef 适用范围一样，仅当 `acpx` 插件实际处于活动状态时，未解析的引用才会阻止激活。

## 沙箱 SSH 身份验证材料

核心 `ssh` 沙箱后端还支持将 SecretRef 用于 SSH 身份验证材料：

```json5
{
  agents: {
    defaults: {
      sandbox: {
        mode: "all",
        backend: "ssh",
        ssh: {
          target: "user@gateway-host:22",
          identityData: { source: "env", provider: "default", id: "SSH_IDENTITY" },
          certificateData: { source: "env", provider: "default", id: "SSH_CERTIFICATE" },
          knownHostsData: { source: "env", provider: "default", id: "SSH_KNOWN_HOSTS" },
        },
      },
    },
  },
}
```

运行时行为：

- OpenClaw 在沙箱激活期间解析这些引用，而不是在每次 SSH 调用时延迟解析。
- 解析后的值会以严格的文件权限（`0o600`）写入临时目录，并用于生成的 SSH 配置。
- 如果生效的沙箱后端不是 `ssh`（或沙箱模式为 `off`），这些引用将保持非活动状态，并且不会阻止启动。

## 支持的凭据范围

[SecretRef 凭据范围](/zh-CN/reference/secretref-credential-surface)中列出了规范支持和不支持的凭据。

<Note>
有意将运行时签发或轮换的凭据以及 OAuth 刷新材料排除在只读 SecretRef 解析之外。
</Note>

## 必需行为和优先级

- 没有引用的字段：保持不变。
- 带有引用的字段：激活期间在活动范围上为必需项。
- 如果明文和引用同时存在，在支持优先级的路径上引用优先。
- 脱敏哨兵值 `__OPENCLAW_REDACTED__` 保留用于内部配置脱敏/还原；如果将其作为字面配置数据提交，则会被拒绝。

警告和审计信号：

- `SECRETS_REF_OVERRIDES_PLAINTEXT`（运行时警告）
- `REF_SHADOWED`（当 `auth-profiles.json` 凭据的优先级高于 `openclaw.json` 引用时产生的审计发现）

Google Chat `serviceAccount` 接受内联 JSON 或 SecretRef。如果此规范字段未设置，Doctor 会将已弃用的同级字段 `serviceAccountRef` 移入其中。

## 激活触发条件

Secret 激活会在以下情况运行：

- 启动（预检加最终激活）
- 配置重载热应用路径
- 配置重载重启检查路径
- 通过 `secrets.reload` 手动重载
- Gateway 配置写入 RPC 预检（`config.set` / `config.apply` / `config.patch`），在持久化编辑内容之前，验证所提交配置载荷中活动范围的 SecretRef

激活契约：

- 成功时会以原子方式替换快照。
- 严格启动失败会中止 Gateway 网关启动。
- 在冷启动期间，如果某个已映射、可隔离的非 Gateway 网关所有者发生可重试的解析失败，可以发布快照，并将该确切所有者配置为不可用。对该所有者的请求会以 `SECRET_SURFACE_UNAVAILABLE` 失败；显式引用失败后，模型提供商所有者不会回退到环境或身份验证配置文件凭据。
- 重载和重启检查会隔离符合条件的已映射所有者。引用标识未变、提供商定义未变且完整的非 Secret 所有者契约未变时，会将其最后一次已知正常的确切值保留为陈旧状态；发生变更或新配置但无法解析的引用仅会使对应所有者以冷状态发布。严格重载失败会保留先前的活动快照。
- `config.set`、`config.apply` 和 `config.patch` 接受可隔离所有者中语法有效但尚未解析的引用，并返回脱敏的 `degradedSecretOwners` 报告。Gateway 网关入口身份验证、结构无效的配置或解析值、策略违规以及未知所有者仍会在修改磁盘之前被拒绝。
- 即使另一个所有者处于冷状态或陈旧状态，正常的同级所有者仍会正常解析并发布。
- 为出站辅助函数/工具调用提供显式的单次调用渠道令牌不会触发 SecretRef 激活；激活点仍为启动、重载和显式 `secrets.reload`。

## 降级和恢复信号

在正常状态之后，如果重载时激活失败，OpenClaw 会进入 Secret 降级状态，并发出一次性系统事件和日志代码：

- `SECRETS_RELOADER_DEGRADED`
- `SECRETS_RELOADER_RECOVERED`

行为：

- 降级：正常的所有者会刷新，陈旧的所有者保留最后一次已知正常值，冷状态所有者仍不可用。
- 已恢复：下一次激活成功后发出一次。
- 已经处于降级状态时，重复失败会记录警告，但不会再次发出事件。
- 严格启动失败绝不会发出降级事件，因为运行时从未进入活动状态。存在冷状态所有者但启动成功时，会记录所有者降级情况，但不会发出重载器事件。
- 引用范围的启动和重载失败会为每个受影响的所有者发出结构化的 `SECRETS_DEGRADED` 警告。提供商范围的故障会发出一条 `SECRETS_PROVIDER_DEGRADED` 警告，其中包含提供商及完整的受影响所有者列表，而不是针对每个所有者重复提供商故障。警告包含脱敏原因、`cold` 或 `stale` 所有者状态，以及 `openclaw secrets reload` 重试提示。警告绝不包含解析值或 SecretRef ID。
- `openclaw doctor` 会列出冷状态和陈旧状态所有者及其受影响的配置路径、脱敏原因和重试指导。

## 命令路径解析

命令路径可以通过 Gateway 网关快照 RPC 选择使用受支持的 SecretRef 解析。适用两类基本行为：

<Tabs>
  <Tab title="严格命令路径">
    例如 `openclaw memory` 远程记忆路径，以及需要远程共享 Secret 引用时的 `openclaw qr --remote`。它们从活动快照读取数据，并在必需的 SecretRef 不可用时快速失败。
  </Tab>
  <Tab title="只读命令路径">
    例如 `openclaw status`、`openclaw status --all`、`openclaw channels status`、`openclaw channels resolve`、`openclaw security audit`，以及只读 Doctor/配置修复流程。它们也优先使用活动快照，但在目标 SecretRef 不可用时会降级，而不是中止。

    只读行为：

    - Gateway 网关运行时，这些命令会首先从活动快照读取。
    - 如果 Gateway 网关解析不完整或 Gateway 网关不可用，它们会尝试针对该命令范围进行定向本地回退。
    - 如果目标 SecretRef 仍不可用，命令会继续生成降级的只读输出，并提供明确的诊断信息，说明已配置该引用，但此命令路径无法使用它。
    - 这种降级行为仅限当前命令；它不会削弱运行时启动、重载或发送/身份验证路径。

  </Tab>
</Tabs>

其他说明：

- 后端 Secret 轮换后的快照刷新由 `openclaw secrets reload` 处理。
- 这些命令路径使用的 Gateway 网关 RPC 方法：`secrets.resolve`。

## 审计和配置工作流

默认操作员流程：

<Steps>
  <Step title="审计当前状态">
    ```bash
    openclaw secrets audit --check
    ```
  </Step>
  <Step title="配置并应用 SecretRef">
    ```bash
    openclaw secrets configure --apply
    ```
  </Step>
  <Step title="重新审计">
    ```bash
    openclaw secrets audit --check
    ```
  </Step>
</Steps>

重新审计结果未清除所有问题之前，不要认为迁移已完成。如果审计仍报告静态存储中存在明文值，即使运行时 API 返回脱敏值，智能体访问风险仍然存在。

如果在 `configure` 期间保存计划而不是应用，请在重新审计之前使用 `openclaw secrets apply --from <plan-path>` 应用该已保存计划。

<AccordionGroup>
  <Accordion title="secrets audit">
    发现项包括：

    - 静态存储中的明文值（`openclaw.json`、`auth-profiles.json`、`.env`，以及生成的 `agents/*/agent/models.json`）。
    - 生成的 `models.json` 条目中残留的明文敏感提供商标头。
    - 未解析的引用。
    - 优先级遮蔽（`auth-profiles.json` 的优先级高于 `openclaw.json` 引用）。
    - 旧版残留（`auth.json`、OAuth 提醒）。

    Exec 说明：默认情况下，审计会跳过 Exec SecretRef 可解析性检查，以避免命令产生副作用。使用 `openclaw secrets audit --allow-exec` 可在审计期间执行 Exec 提供商。

    标头残留说明：敏感提供商标头检测基于名称启发式规则（常见身份验证/凭据标头名称，以及 `authorization`、`x-api-key`、`token`、`secret`、`password` 和 `credential` 等片段）。

  </Accordion>
  <Accordion title="secrets configure">
    交互式辅助工具会：

    - 首先配置 `secrets.providers`（`env`/`file`/`exec`，添加/编辑/移除）。
    - 允许你为一个智能体范围选择 `openclaw.json` 中支持的 Secret 承载字段以及 `auth-profiles.json`。
    - 可以直接在目标选择器中创建新的 `auth-profiles.json` 映射。
    - 捕获 SecretRef 详细信息（`source`、`provider`、`id`）。
    - 运行预检解析，并可立即应用。

    Exec 说明：除非设置了 `--allow-exec`，否则预检会跳过 Exec SecretRef 检查。如果直接从 `configure --apply` 应用，并且计划包含 Exec 引用/提供商，请在应用步骤中也保持设置 `--allow-exec`。

    实用模式：

    - `openclaw secrets configure --providers-only`
    - `openclaw secrets configure --skip-provider-setup`
    - `openclaw secrets configure --agent <id>`

    `configure` 应用默认行为：

    - 从 `auth-profiles.json` 中清除目标提供商的匹配静态凭据。
    - 从 `auth.json` 中清除旧版静态 `api_key` 条目。
    - 从生效状态和活动配置的 `.env` 文件中清除匹配的已知 Secret 行（当两个路径匹配时会去重）。

  </Accordion>
  <Accordion title="secrets apply">
    应用已保存的计划：

    ```bash
    openclaw secrets apply --from /tmp/openclaw-secrets-plan.json
    openclaw secrets apply --from /tmp/openclaw-secrets-plan.json --allow-exec
    openclaw secrets apply --from /tmp/openclaw-secrets-plan.json --dry-run
    openclaw secrets apply --from /tmp/openclaw-secrets-plan.json --dry-run --allow-exec
    ```

    Exec 说明：除非设置了 `--allow-exec`，否则试运行会跳过 Exec 检查；除非设置了 `--allow-exec`，否则写入模式会拒绝包含 Exec SecretRef/提供商的计划。

    有关严格的目标/路径契约详情和确切拒绝规则，请参阅 [Secret 应用计划契约](/zh-CN/gateway/secrets-plan-contract)。

  </Accordion>
</AccordionGroup>

## 单向安全策略

<Warning>
OpenClaw 有意不写入包含历史明文 Secret 值的回滚备份。
</Warning>

安全模型：

- 进入写入模式之前，预检必须成功。
- 提交之前会验证运行时激活。
- 应用时通过原子文件替换更新文件，并在失败时尽力还原。

## 旧版身份验证兼容性说明

对于静态凭据，运行时不再依赖明文旧版身份验证存储。

- 运行时凭据来源是解析后的内存快照。
- 发现旧版静态 `api_key` 条目时会将其清除。
- OAuth 相关兼容行为保持独立。

## Web UI 说明

某些 SecretInput 联合类型在原始编辑器模式中比在表单模式中更容易配置。

## 相关内容

- [身份验证](/zh-CN/gateway/authentication) - 身份验证设置
- [CLI：密钥](/zh-CN/cli/secrets) - CLI 命令
- [Vault SecretRefs](/zh-CN/plugins/vault) - HashiCorp Vault 提供商设置
- [环境变量](/zh-CN/help/environment) - 环境变量优先级
- [SecretRef 凭据界面](/zh-CN/reference/secretref-credential-surface) - 凭据界面
- [密钥应用计划契约](/zh-CN/gateway/secrets-plan-contract) - 计划契约详情
- [安全性](/zh-CN/gateway/security) - 安全态势
