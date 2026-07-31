---
read_when:
    - 你需要了解会加载哪些环境变量，以及它们的加载顺序
    - 你正在调试 Gateway 网关中缺失的 API 密钥
    - 你正在编写提供商身份验证或部署环境的文档
summary: OpenClaw 加载环境变量的位置及其优先级顺序
title: 环境变量
x-i18n:
    generated_at: "2026-07-26T06:17:12Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: db9990dea5df7731e54c8d442f4704bd4d6e0caf6f2c2fdea32d2583cd41128c
    source_path: help/environment.md
    workflow: 16
---

OpenClaw 从多个来源读取环境变量。规则是**绝不覆盖现有值**。
工作区 `.env` 文件是可信度较低的来源：OpenClaw 在应用优先级之前，会忽略工作区 `.env` 中的提供商凭据和受保护的运行时控制项。

## 优先级（从高到低）

1. **进程环境**（Gateway 网关进程已从父 shell/守护进程继承的环境）。
2. **当前工作目录中的 `.env`**（dotenv 默认来源；不覆盖现有值；忽略提供商凭据和受保护的运行时控制项）。
3. 位于 `~/.openclaw/.env` 的**全局 `.env`**（也称为 `$OPENCLAW_STATE_DIR/.env`；建议用于提供商 API 密钥；不覆盖现有值）。
4. `~/.openclaw/openclaw.json` 中的**配置 `env` 块**（仅在缺失时应用）。
5. **可选的登录 shell 导入**（`env.shellEnv.enabled` 或 `OPENCLAW_LOAD_SHELL_ENV=1`），仅应用于缺失的预期键名。

在使用默认状态目录的全新 Ubuntu 安装中，OpenClaw 还会在全局 `.env` 之后将 `~/.config/openclaw/gateway.env` 作为兼容性回退来源。如果两个文件都存在且内容不一致，OpenClaw 会保留 `~/.openclaw/.env` 并输出警告。

如果配置文件完全不存在，则跳过第 4 步；如果已启用 shell 导入，它仍会运行。

## 支持的面向操作员的变量

以下变量构成面向操作员的受支持环境契约。未记录的 `OPENCLAW_*` 变量属于内部实现细节，可能会在不另行通知的情况下消失。

### 路径和实例

| 变量                     | 用途                                                              |
| ------------------------ | ----------------------------------------------------------------- |
| `OPENCLAW_HOME`          | 覆盖 OpenClaw 路径默认值所使用的主目录。                           |
| `OPENCLAW_STATE_DIR`     | 覆盖可变状态目录。                                                |
| `OPENCLAW_CONFIG_PATH`   | 覆盖当前使用的配置文件路径。                                      |
| `OPENCLAW_WORKSPACE_DIR` | 覆盖默认 Agent 工作区。                                           |
| `OPENCLAW_PROFILE`       | 选择具备独立默认值的命名配置文件。                                |
| `OPENCLAW_GIT_DIR`       | 覆盖开发渠道更新所使用的源代码检出目录。                          |
| `OPENCLAW_INCLUDE_ROOTS` | 允许从其他根目录解析 `$include`。                          |

### Gateway 网关和身份验证

| 变量                        | 用途                                                            |
| --------------------------- | --------------------------------------------------------------- |
| `OPENCLAW_GATEWAY_URL`      | 覆盖客户端使用的远程 Gateway 网关 URL。                         |
| `OPENCLAW_GATEWAY_PORT`     | 覆盖本地 Gateway 网关端口。                                     |
| `OPENCLAW_GATEWAY_TOKEN`    | 为 Gateway 网关服务器和客户端提供令牌身份验证。                 |
| `OPENCLAW_GATEWAY_PASSWORD` | 为 Gateway 网关服务器和客户端提供密码身份验证。                 |

### 提供商凭据

核心和内置提供商插件可识别以下凭据及提供商选择变量。如果需要限定范围的凭据，而不是整个进程共用的单一值，请优先使用各提供商的配置字段或 SecretRef 字段。

`AI_GATEWAY_API_KEY`, `ANTHROPIC_ADMIN_API_KEY`, `ANTHROPIC_ADMIN_KEY`, `ANTHROPIC_API_KEY`, `ANTHROPIC_OAUTH_TOKEN`, `ARCEEAI_API_KEY`, `AZURE_OPENAI_API_KEY`, `AZURE_SPEECH_API_KEY`, `AZURE_SPEECH_KEY`, `AZURE_SPEECH_REGION`, `BASETEN_API_KEY`, `BRAVE_API_KEY`, `BYTEPLUS_API_KEY`, `BYTEPLUS_SEED_SPEECH_API_KEY`, `CEREBRAS_API_KEY`, `CHUTES_API_KEY`, `CHUTES_OAUTH_TOKEN`, `CLAWROUTER_API_KEY`, `CLOUDFLARE_AI_GATEWAY_API_KEY`, `CODEX_API_KEY`, `COHERE_API_KEY`, `COMFY_API_KEY`, `COMFY_CLOUD_API_KEY`, `COPILOT_GITHUB_TOKEN`, `DASHSCOPE_API_KEY`, `DEEPGRAM_API_KEY`, `DEEPINFRA_API_KEY`, `DEEPSEEK_API_KEY`, `ELEVENLABS_API_KEY`, `EXA_API_KEY`, `FAL_API_KEY`, `FAL_KEY`, `FEATHERLESS_API_KEY`, `FIRECRAWL_API_KEY`, `FIREWORKS_API_KEY`, `GCLOUD_PROJECT`, `GEMINI_API_KEY`, `GH_TOKEN`, `GITHUB_TOKEN`, `GMI_API_KEY`, `GOOGLE_API_KEY`, `GOOGLE_APPLICATION_CREDENTIALS`, `GOOGLE_CLOUD_API_KEY`, `GOOGLE_CLOUD_LOCATION`, `GOOGLE_CLOUD_PROJECT`, `GRADIUM_API_KEY`, `GROQ_API_KEY`, `HF_TOKEN`, `HUGGINGFACE_HUB_TOKEN`, `INWORLD_API_KEY`, `KILOCODE_API_KEY`, `KIMICODE_API_KEY`, `KIMI_API_KEY`, `LITELLM_API_KEY`, `LM_API_TOKEN`, `LONGCAT_API_KEY`, `MINIMAX_API_KEY`, `MINIMAX_CODE_PLAN_KEY`, `MINIMAX_CODING_API_KEY`, `MINIMAX_OAUTH_TOKEN`, `MISTRAL_API_KEY`, `MODELSTUDIO_API_KEY`, `MODEL_API_KEY`, `MOONSHOT_API_KEY`, `NOVITA_API_KEY`, `NVIDIA_API_KEY`, `OLLAMA_API_KEY`, `OPENAI_ADMIN_KEY`, `OPENAI_API_KEY`, `OPENCODE_API_KEY`, `OPENCODE_ZEN_API_KEY`, `OPENROUTER_API_KEY`, `PARALLEL_API_KEY`, `PERPLEXITY_API_KEY`, `PIXVERSE_API_KEY`, `QIANFAN_API_KEY`, `QWEN_API_KEY`, `QWEN_TOKEN_PLAN_API_KEY`, `RUNWAYML_API_SECRET`, `RUNWAY_API_KEY`, `SENSEAUDIO_API_KEY`, `SGLANG_API_KEY`, `SPEECH_KEY`, `SPEECH_REGION`, `STEPFUN_API_KEY`, `SYNTHETIC_API_KEY`, `TAVILY_API_KEY`, `TOGETHER_API_KEY`, `TOKENHUB_API_KEY`, `TOKENPLAN_API_KEY`, `VENICE_API_KEY`, `VLLM_API_KEY`, `VOLCANO_ENGINE_API_KEY`, `VOLCENGINE_TTS_API_KEY`, `VOLCENGINE_TTS_APPID`, `VOLCENGINE_TTS_TOKEN`, `VOYAGE_API_KEY`, `VYDRA_API_KEY`, `XAI_API_KEY`, `XIAOMI_API_KEY`, `XIAOMI_TOKEN_PLAN_API_KEY`, `XI_API_KEY`, `ZAI_API_KEY` 和 `Z_AI_API_KEY`。

已安装的第三方插件可以在其插件清单中声明其他凭据变量；这些变量是声明它们的插件所提供的契约，并非 OpenClaw 核心变量。

### 日志和诊断

| 变量                                 | 用途                                                          |
| ------------------------------------ | ------------------------------------------------------------- |
| `OPENCLAW_LOG_LEVEL`                 | 覆盖文件和控制台日志级别。                                    |
| `OPENCLAW_DEBUG_MODEL_TRANSPORT`     | 启用模型传输时序诊断。                                        |
| `OPENCLAW_DEBUG_MODEL_PAYLOAD`       | 选择经过脱敏的模型载荷诊断。                                  |
| `OPENCLAW_DEBUG_SSE`                 | 选择 SSE 时序或事件速览诊断。                                 |
| `OPENCLAW_DEBUG_CODE_MODE`           | 启用代码模式界面诊断。                                        |
| `OPENCLAW_DIAGNOSTICS`               | 启用指定的诊断标志，或使用 `0` 禁用所有标志。   |
| `OPENCLAW_DIAGNOSTICS_TIMELINE_PATH` | 选择时间线诊断所使用的 JSONL 路径。                            |
| `OPENCLAW_DIAGNOSTICS_EVENT_LOOP`    | 将事件循环采样添加到时间线诊断中。                            |

### 功能和运行时开关

| 变量                                 | 用途                                                                         |
| ------------------------------------ | ---------------------------------------------------------------------------- |
| `OPENCLAW_LOAD_SHELL_ENV`            | 从登录 shell 导入缺失的预期变量。                                             |
| `OPENCLAW_SHELL_ENV_TIMEOUT_MS`      | 设置登录 shell 导入超时时间。                                                 |
| `OPENCLAW_EXEC_SHELL_SNAPSHOT`       | 使用 `0` 禁用 Exec shell 快照。                                |
| `OPENCLAW_OFFLINE`                   | 阻止下载固定版本的 Agent 辅助二进制文件。                                    |
| `OPENCLAW_BROWSER_HEADLESS`          | 强制托管浏览器以有界面模式（`0`）或无头模式（`1`）启动。 |
| `OPENCLAW_DISABLE_BONJOUR`           | 强制开启（`0`）或关闭（`1`）Bonjour 广播。      |
| `OPENCLAW_NO_AUTO_UPDATE`            | 禁用自动应用更新。                                                           |
| `OPENCLAW_ALLOW_INSECURE_PRIVATE_WS` | 作为紧急覆盖选项，允许可信的私有 DNS `ws://` 连接。                 |
| `OPENCLAW_ALLOW_MULTI_GATEWAY`       | 允许多个 Gateway 网关进程，同时保留每个状态目录的所有权锁。                    |
| `OPENCLAW_SKIP_CHANNELS`             | 启动 Gateway 网关但不启用渠道传输，以便进行故障排除。                          |
| `OPENCLAW_THEME`                     | 强制 TUI 调色板使用 `light` 或 `dark`。                 |

## 提供商凭据和工作区 `.env`

不要只将提供商 API 密钥保存在工作区 `.env` 中。OpenClaw 会阻止从工作区 `.env` 文件读取大量提供商凭据键和端点重定向键，其中包括所有已知的提供商身份验证环境变量（例如 `GEMINI_API_KEY`、`GOOGLE_API_KEY`、`XAI_API_KEY`、`MISTRAL_API_KEY`、`GROQ_API_KEY`、`DEEPSEEK_API_KEY`、`PERPLEXITY_API_KEY`、`BRAVE_API_KEY`、`TAVILY_API_KEY`、`EXA_API_KEY`、`FIRECRAWL_API_KEY`），以及任何以 `_API_HOST`、`_BASE_URL`、`_ENDPOINT` 或 `_HOMESERVER` 结尾的键，并包括整个 `OPENCLAW_*`、`CLAWHUB_*`、`ANTHROPIC_API_KEY_*` 和 `OPENAI_API_KEY_*` 命名空间。

请改用以下任一可信来源存储提供商凭据：

- Gateway 网关进程环境，例如 shell、launchd/systemd 单元、容器 Secret 或 CI Secret。
- 位于 `~/.openclaw/.env` 或 `$OPENCLAW_STATE_DIR/.env` 的全局运行时 dotenv 文件。
- `~/.openclaw/openclaw.json` 中的配置 `env` 块。
- 启用 `env.shellEnv.enabled` 或 `OPENCLAW_LOAD_SHELL_ENV=1` 后进行可选的登录 shell 导入。

如果之前仅将提供商密钥或端点路由值存储在工作区 `.env` 中，请将它们移至上述某个可信来源。工作区 `.env` 仍可提供不属于凭据、端点重定向、主机覆盖或 `OPENCLAW_*` 运行时控制项的普通项目变量。

有关安全原理，请参阅[工作区 `.env` 文件](/zh-CN/gateway/security#workspace-env-files)。

## 配置 `env` 块

可以使用以下两种等效方式设置内联环境变量（两者都不会覆盖现有值）：

```json5
{
  env: {
    OPENROUTER_API_KEY: "sk-or-...",
    vars: {
      GROQ_API_KEY: "gsk-...",
    },
  },
}
```

配置 `env` 块仅接受字符串字面值。它不会展开
`file:...` 值；例如，`XAI_API_KEY: "file:secrets/xai-api-key.txt"`
会作为该原始字符串原样传递给提供商。

对于基于文件的提供商密钥，请在支持 SecretRef 的凭据字段中使用
SecretRef：

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

有关支持的字段，请参阅 [Secret 管理](/zh-CN/gateway/secrets)和
[SecretRef 凭据界面](/zh-CN/reference/secretref-credential-surface)。

## Shell 环境导入

`env.shellEnv` 会运行你的登录 shell，并且仅导入**缺失的**预期键名：

```json5
{
  env: {
    shellEnv: {
      enabled: true,
      timeoutMs: 15000,
    },
  },
}
```

等效环境变量：

- `OPENCLAW_LOAD_SHELL_ENV=1`
- `OPENCLAW_SHELL_ENV_TIMEOUT_MS=15000`（默认值为 `15000`）

## Exec shell 快照

在非 Windows Gateway 网关主机上，bash 和 zsh 的 `exec` 命令默认使用启动快照。
在 Gateway 网关进程环境中设置 `OPENCLAW_EXEC_SHELL_SNAPSHOT=0` 可禁用此路径。
值 `false`、`no` 和 `off` 也会禁用此路径。每次调用的 `exec.env` 值无法切换
快照，也无法重定向快照缓存。

## 运行时注入的环境变量

OpenClaw 还会向生成的子进程中注入上下文标记：

- `OPENCLAW_SHELL=exec`：为通过 `exec` 工具运行的命令设置。
- `OPENCLAW_SHELL=acp-client`：当 `openclaw acp client` 生成 ACP 桥接进程时设置。
- `OPENCLAW_SHELL=tui-local`：为本地 TUI `!` shell 命令设置。
- `OPENCLAW_CLI=1`：为 CLI 入口点生成的子进程设置。

这些是运行时标记（不是必需的用户配置）。可以在 shell/profile 逻辑中使用它们，
以应用特定于上下文的规则。

## UI 环境变量

- `OPENCLAW_THEME=light`：当终端使用浅色背景时，强制使用浅色 TUI 调色板。
- `OPENCLAW_THEME=dark`：强制使用深色 TUI 调色板。
- `COLORFGBG`：如果终端导出了此变量，OpenClaw 会使用背景颜色提示自动选择 TUI 调色板。

## 配置中的环境变量替换

可以使用 `${VAR_NAME}` 语法，在配置字符串值中直接引用环境变量：

```json5
{
  models: {
    providers: {
      "vercel-gateway": {
        apiKey: "${VERCEL_GATEWAY_API_KEY}",
      },
    },
  },
}
```

了解详情，请参阅[配置：环境变量替换](/zh-CN/gateway/configuration-reference#env-var-substitution)。

## Secret refs 与 `${ENV}` 字符串

OpenClaw 支持两种由环境变量驱动的模式：

- 在配置值中进行 `${VAR}` 字符串替换。
- 对于支持密钥引用的字段，使用 SecretRef 对象（`{ source: "env", provider: "default", id: "VAR" }`）。

两者都在激活时从进程环境变量中解析。SecretRef 的详细信息记录在[密钥管理](/zh-CN/gateway/secrets)中。
配置的 `env` 块本身不会解析 SecretRef 或 `file:...`
简写值。

## 路径相关环境变量

| 变量                 | 用途                                                                                                                                                                                                                                 |
| ------------------------ | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `OPENCLAW_HOME`          | 覆盖用于 OpenClaw 内部路径默认值的主目录（`~/.openclaw/`、Agent 目录、会话、凭据、安装程序新手引导以及默认开发检出目录）。将 OpenClaw 作为专用服务用户运行时很有用。 |
| `OPENCLAW_STATE_DIR`     | 覆盖状态目录（默认值为 `~/.openclaw`）。                                                                                                                                                                                   |
| `OPENCLAW_CONFIG_PATH`   | 覆盖配置文件路径（默认值为 `~/.openclaw/openclaw.json`）。                                                                                                                                                                    |
| `OPENCLAW_INCLUDE_ROOTS` | 路径目录列表，`$include` 指令可从这些目录解析配置目录之外的文件（默认值：无——`$include` 仅限于配置目录）。支持波浪号展开。                                                         |

## Agent 辅助工具下载

设置 `OPENCLAW_OFFLINE=1`，可阻止 OpenClaw 下载其固定版本的 `fd`
和 `ripgrep` 辅助二进制文件。OpenClaw 工具目录中已有的辅助工具
以及可用的系统二进制文件仍可使用；缺失的辅助工具将保持不可用，
而不会触发网络请求。

## 日志

| 变量                         | 用途                                                                                                                                                                                      |
| -------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `OPENCLAW_LOG_LEVEL`             | 覆盖文件和控制台的日志级别（例如 `debug`、`trace`）。优先级高于配置中的 `logging.level` 和 `logging.consoleLevel`。无效值会被忽略并发出警告。 |
| `OPENCLAW_DEBUG_MODEL_TRANSPORT` | 在 `info` 级别输出针对性的模型请求/响应计时诊断，而无需启用全局调试日志。                                                                                  |
| `OPENCLAW_DEBUG_MODEL_PAYLOAD`   | 模型载荷诊断：`summary`、`tools` 或 `full-redacted`。`full-redacted` 会受到容量限制并经过脱敏，但可能包含提示词/消息文本。                                               |
| `OPENCLAW_DEBUG_SSE`             | 流式传输诊断：使用 `events` 记录首次/完成计时，使用 `peek` 包含前五个经过脱敏的 SSE 事件。                                                                                 |
| `OPENCLAW_DEBUG_CODE_MODE`       | 代码模式模型表面诊断，包括提供商工具隐藏以及紧凑控制/直接强制执行。                                                                                  |

### `OPENCLAW_HOME`

设置后，`OPENCLAW_HOME` 会替代系统主目录（`$HOME` / `os.homedir()`），用于 OpenClaw 内部路径默认值。其中包括默认状态目录、配置路径、Agent 目录、凭据、安装程序新手引导工作区，以及 `openclaw update --channel dev` 使用的默认开发检出目录。

**优先级：** `OPENCLAW_HOME` > `$HOME` > `USERPROFILE` > Android 上的 Termux `PREFIX` 主目录回退 > `os.homedir()`

**示例**（macOS LaunchDaemon）：

```xml
<key>EnvironmentVariables</key>
<dict>
  <key>OPENCLAW_HOME</key>
  <string>/Users/user</string>
</dict>
```

`OPENCLAW_HOME` 也可以设置为波浪号路径（例如 `~/svc`），使用前会按照同一套操作系统主目录回退链进行展开。

`OPENCLAW_STATE_DIR`、`OPENCLAW_CONFIG_PATH` 和 `OPENCLAW_GIT_DIR` 等显式路径变量仍具有更高优先级。操作系统账户相关任务（例如 shell 启动文件检测、包管理器设置和主机 `~` 展开）可能仍会使用真实的系统主目录。

## nvm 用户：web_fetch TLS 失败

如果 Node.js 是通过 **nvm**（而非系统包管理器）安装的，内置 `fetch()` 会使用
nvm 内置的 CA 存储，其中可能缺少现代根 CA（用于 Let's Encrypt 的 ISRG Root X1/X2、
DigiCert Global Root G2 等）。这会导致 `web_fetch` 在大多数 HTTPS 网站上以 `"fetch failed"` 失败。

在 Linux 上，OpenClaw 会自动检测 nvm，并在实际启动环境中应用修复：

- `openclaw gateway install` 将 `NODE_EXTRA_CA_CERTS` 写入 systemd 服务环境
- `openclaw` CLI 入口点会在 Node 启动前设置 `NODE_EXTRA_CA_CERTS`，然后重新执行自身

**手动修复（适用于旧版本或直接启动 `node ...`）：**

启动 OpenClaw 前导出该变量：

```bash
export NODE_EXTRA_CA_CERTS=/etc/ssl/certs/ca-certificates.crt
openclaw gateway run
```

不要依赖仅将此变量写入 `~/.openclaw/.env`；Node 会在进程启动时读取
`NODE_EXTRA_CA_CERTS`。

## 旧版环境变量

OpenClaw 仅会读取 `OPENCLAW_*` 环境变量。早期版本使用的旧版
`CLAWDBOT_*` 和 `MOLTBOT_*` 前缀会被静默
忽略。

如果 Gateway 网关进程启动时仍设置了其中任何变量，OpenClaw 会发出
一条 Node 弃用警告（`OPENCLAW_LEGACY_ENV_VARS`），列出检测到的
前缀及总数。请将旧版前缀替换为 `OPENCLAW_`，以重命名每个值（例如将 `CLAWDBOT_GATEWAY_TOKEN` 改为
`OPENCLAW_GATEWAY_TOKEN`）；旧名称不会生效。

## 相关内容

- [Gateway 配置](/zh-CN/gateway/configuration)
- [常见问题：环境变量和 .env 加载](/zh-CN/help/faq#env-vars-and-env-loading)
- [模型概览](/zh-CN/concepts/models)
