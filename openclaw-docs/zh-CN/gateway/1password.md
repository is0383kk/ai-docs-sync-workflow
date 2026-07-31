---
read_when:
    - 你希望将 API key 从 openclaw.json 中移出并存入 1Password
    - 你以无头模式运行 Gateway 网关，并且需要用于 op 的服务账号身份验证
    - 你希望智能体使用 `op` CLI 读取或注入密钥
summary: 使用 1Password CLI 解析 Gateway 网关密钥，并让智能体使用内置的 1password skill
title: 1Password
x-i18n:
    generated_at: "2026-07-26T06:15:00Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 5bb14944f0b3ce1ee3f90bf666a53e8673e7a9861e3e138a5fabe9c8e070cbd7
    source_path: gateway/1password.md
    workflow: 16
---

OpenClaw 以三种相互独立的方式与 **1Password** 配合使用：

- **配置密钥：** `openclaw.json` 中的任何 [SecretRef](/zh-CN/gateway/secrets) 字段都可以在运行时通过 `op` CLI 解析，因此 API 密钥绝不会存放在配置文件中。
- **智能体工作流：** 内置的 `1password` skill 会指导智能体登录，并使用 `op` 为自身任务读取或注入密钥。
- **浏览器登录：** `claude-cli` 后端可以通过 [1Password for Claude](https://support.1password.com/1password-claude/) 使用 Claude Code 的 Chrome 集成，让智能体登录网站，同时密码绝不会传递给模型或 OpenClaw。

## 要求

- Gateway 网关主机上已安装 [1Password CLI](https://developer.1password.com/docs/cli/get-started/)（`op`）（在 macOS 上为 `brew install 1password-cli`）。
- 为 `op` 配置一种身份验证模式：
  - **服务账户**（推荐用于无头 Gateway 网关）：在 Gateway 网关服务环境中导出 `OP_SERVICE_ACCOUNT_TOKEN`。无需桌面应用，也无需交互式登录。
  - **桌面应用集成**：1Password 应用在同一台计算机上运行，并已启用 CLI 集成。首次调用可能会触发 Touch ID 或系统身份验证。
  - **独立登录**：`op signin` 会在每个会话中提示登录。智能体可以通过 skill 使用这种方式，但它不适合在无头 Gateway 网关上解析配置密钥。

## 使用 op 解析配置密钥

声明一个运行 `op read` 并使用 `op://vault/item/field` 引用的 exec 密钥提供商，然后让任何支持 SecretRef 的字段指向它：

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

各部分的配合方式：

- `command` 必须是绝对路径；`trustedDirs` 将其目录标记为受信任，而 `allowSymlinkCommand` 是必需的，因为 Homebrew 会将 `op` 安装为符号链接。
- `args` 会原样传递 `op://vault/item/field` 引用。OpenClaw 本身不会解析 `op://` 方案；由 `op` 二进制文件负责解析。
- `passEnv` 会从 Gateway 网关环境中转发列出的变量。桌面应用集成需要 `HOME`；服务账户还需要 Gateway 网关服务环境中存在 `OP_SERVICE_ACCOUNT_TOKEN`（将其添加到 `passEnv`，或者仅在你接受令牌可从配置文件中读取时，才通过 `env` 设置）。
- 对于单值输出，请保留 `id: "value"`。使用 `jsonOnly: true` 和 JSON 载荷时，请改用 JSON 指针 ID 定位字段。
- 每个密钥使用一个提供商条目，可以让引用易于审计；请根据使用方命名提供商（`onepassword_openai`、`onepassword_telegram`）。

有关解析顺序、缓存和失败语义，请参阅 [Gateway 网关密钥](/zh-CN/gateway/secrets)；有关接受 SecretRef 的所有字段，请参阅 [SecretRef 凭据接口](/zh-CN/reference/secretref-credential-surface)。

## 无头 Gateway 网关的服务账户设置

1. 在你的 1Password 账户中创建服务账户，并仅授予其读取 Gateway 网关所需保管库项目的权限。
2. 将 `OP_SERVICE_ACCOUNT_TOKEN` 提供给 Gateway 网关服务（launchd plist、systemd 单元或容器环境）。
3. 将 `"OP_SERVICE_ACCOUNT_TOKEN"` 添加到提供商的 `passEnv` 列表。
4. 在 Gateway 网关主机环境中验证：`op whoami` 应直接输出服务账户，而不会提示登录。

服务账户读取要求在 `op://` 引用中明确指定保管库名称。应严格限制账户权限范围；它是一种持有者凭据。

## 面向智能体的 1password skill

OpenClaw 内置了一个 `1password` skill，可让智能体熟练操作 `op`：它会检测可用的身份验证模式（服务账户、桌面应用集成或独立登录），在读取任何内容前使用 `op whoami` 验证访问权限，并优先使用 `op run` / `op inject`，而不是将密钥值写入磁盘。该 skill 需要 `op` 二进制文件；缺失时会提供 Homebrew 安装选项。

智能体会将其用于自身工作流，例如在任务执行期间读取部署令牌，或将环境变量注入命令。它独立于配置密钥解析；Gateway 网关解析 SecretRef 时不涉及任何 skill。

## 使用 1Password for Claude 进行浏览器登录

[1Password for Claude](https://support.1password.com/1password-claude/) 允许 Claude 请求登录，并由 1Password 浏览器扩展通过加密通道将凭据直接填入页面。密钥绝不会进入模型上下文、对话记录或 OpenClaw。当 OpenClaw 在启用 Claude Code Chrome 集成的情况下运行 [`claude-cli` 后端](/zh-CN/gateway/cli-backends#claude-cli-specifics)时，智能体任务可以使用该流程访问需要真实登录会话的网站。

除了后端本身，此功能还需要：

- 一台安装了 Chrome 的 macOS Gateway 网关主机、已连接的 [Claude in Chrome extension](https://code.claude.com/docs/en/chrome)、1Password 桌面应用以及 1Password 浏览器扩展（后两者均须为 8.12.28 或更高版本）。
- Claude Code 已登录直接订阅的 Anthropic 方案（Pro、Max、Team 或 Enterprise）。通过 Amazon Bedrock、Google Cloud 或其他第三方提供商使用时，Chrome 集成不可用。
- 在 Anthropic 端完成一次性 1Password 连接：通过 Claude 桌面应用或 [1Password 指南](https://support.1password.com/1password-claude/)中所述的扩展流程设置 1Password for Claude；它目前是 macOS 测试版。在 1Password Business 中，管理员必须先在 Policies 下启用 "Allow AI agents to autofill for users"；Anthropic Team/Enterprise 方案默认也会关闭此集成，直到 Owner 将其启用。
- 一个将 `--chrome` 添加到 Claude 启动参数的 [CLI 后端插件](/zh-CN/plugins/cli-backend-plugins)；内置后端不会启用 Chrome。
- Gateway 网关主机旁需要有人操作：每次使用凭据时，1Password 都会显示确认提示，并须在该主机上确认（例如使用 Touch ID）。在限制严格的 exec 策略下，浏览器工具调用本身也会先作为 OpenClaw 审批转发到你的渠道。

在将此功能接入 OpenClaw 前，请在 Gateway 网关主机上的交互式会话中验证各个组件：运行 `claude --chrome`，确认扩展已连接，并检查 `claude-in-chrome` 工具中是否包含凭据工具。如果它们未在此处出现，也不会通过 OpenClaw 出现。

一次性密码由 1Password 在同一页面中填写；绝不要通过聊天转发验证码或密码。目前无头或远程 Gateway 网关无法使用此流程，因为审批操作和浏览器都位于 Gateway 网关主机上。

## 安全注意事项

- 通过 exec 提供商解析的密钥值会保留在 Gateway 网关内存中；配置快照和 `config.get` 响应会对 SecretRef 字段进行脱敏。
- 绝不要将密钥值放入 `openclaw.json`、日志或聊天中。配置中只保留项目名称，值则存放在 1Password 中。
- 1Password 审计记录会显示每次服务账户读取操作，便于执行密钥轮换和事件审查。

## 故障排查

- `command not found` 或生成进程错误：使用 `op` 的绝对路径，并将其目录加入 `trustedDirs`。
- `op` 可以解析，但读取因符号链接错误而失败：对于 Homebrew 安装，请设置 `allowSymlinkCommand: true`。
- `account is not signed in`：对于服务账户，请确认 `OP_SERVICE_ACCOUNT_TOKEN` 能传递至 Gateway 网关服务，并已列入 `passEnv`；对于桌面集成，请确认应用正在运行且已解锁。
- 首次读取缓慢：提高提供商上的 `timeoutMs`；在繁忙主机上，`op` 冷启动可能超过严格的超时时间。
