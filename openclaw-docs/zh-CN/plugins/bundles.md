---
read_when:
    - 你想安装兼容 Codex、Claude 或 Cursor 的捆绑包
    - 你需要了解 OpenClaw 如何将包内容映射为原生功能
    - 你正在调试内置包检测或能力缺失问题
summary: 将 Codex、Claude 和 Cursor 工具包作为 OpenClaw 插件安装和使用
title: 插件包
x-i18n:
    generated_at: "2026-07-26T06:20:08Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: d44006866238f53ee2e3e8126cc4f7ed6f7413534257775f7904c9b877778c59
    source_path: plugins/bundles.md
    workflow: 16
---

OpenClaw 可以安装来自三个外部生态系统的插件：**Codex**、**Claude**
和 **Cursor**。它们称为 **bundle**——一种内容和元数据包，
OpenClaw 会将其映射为 Skills、Hooks 和 MCP 工具等原生功能。

<Info>
  bundle 与原生 OpenClaw 插件**不同**。原生插件在进程内运行，
  可以注册任意能力。bundle 是内容包，仅选择性映射功能，
  信任边界也更窄。
</Info>

## 为什么需要 bundle

许多实用插件以 Codex、Claude 或 Cursor 格式发布。OpenClaw
无需作者将其重写为原生 OpenClaw 插件，而是会检测这些格式，
并将其中支持的内容映射到原生功能集。你可以安装 Claude 命令包或
Codex Skills bundle，并立即使用。

## 安装 bundle

<Steps>
  <Step title="从目录、归档文件或市场安装">
    ```bash
    # 本地目录
    openclaw plugins install ./my-bundle

    # 归档文件
    openclaw plugins install ./my-bundle.tgz

    # Claude 市场
    openclaw plugins marketplace list <source>
    openclaw plugins install <plugin> --marketplace <source>
    ```

    `<source>` 是本地市场路径/仓库或 git/GitHub 来源。

  </Step>

  <Step title="验证检测结果">
    ```bash
    openclaw plugins list
    openclaw plugins inspect <id>
    ```

    bundle 会显示 `Format: bundle`，以及值为 `codex`、
    `claude` 或 `cursor` 的 `Bundle format:`。

  </Step>

  <Step title="重启并使用">
    ```bash
    openclaw gateway restart
    ```

    映射后的功能（Skills、Hooks、MCP 工具、LSP 默认值）将在下一个会话中可用。

  </Step>
</Steps>

## OpenClaw 从 bundle 映射的内容

目前并非所有 bundle 功能都能在 OpenClaw 中运行。以下列出了
已经可用的功能，以及已检测但尚未接通的功能。

### 当前支持

| 功能          | 映射方式                                                                                         | 适用格式       |
| ------------- | ------------------------------------------------------------------------------------------------ | -------------- |
| Skills 内容   | bundle Skills 根目录作为普通 OpenClaw Skills 加载                                                | 所有格式       |
| 命令          | `commands/` 和 `.cursor/commands/` 被视为 Skills 根目录                                    | Claude、Cursor |
| Hook 包       | OpenClaw 风格的 `HOOK.md` + `handler.ts` 布局                                    | Codex          |
| MCP 工具      | bundle MCP 配置合并到嵌入式 OpenClaw 设置中；加载受支持的 stdio 和 HTTP 服务器                   | 所有格式       |
| LSP 服务器    | Claude `.lsp.json` 和清单声明的 `lspServers` 合并到嵌入式 OpenClaw LSP 默认值中     | Claude         |
| 设置          | Claude `settings.json` 作为嵌入式 OpenClaw 默认值导入                                        | Claude         |

#### Skills 内容

- bundle Skills 根目录作为普通 OpenClaw Skills 根目录加载。
- Claude `commands/` 根目录被视为额外的 Skills 根目录。
- Cursor `.cursor/commands/` 根目录被视为额外的 Skills 根目录。

Claude Markdown 命令文件和 Cursor 命令 Markdown 都通过
常规 OpenClaw Skills 加载器工作。

#### Hook 包

只有使用常规 OpenClaw Hook 包布局时，bundle Hook 根目录才有效：
`HOOK.md` 加 `handler.ts` 或 `handler.js`。目前这主要适用于
与 Codex 兼容的情况。

#### 嵌入式 OpenClaw 的 MCP

- 已启用的 bundle 可以提供 MCP 服务器配置。
- OpenClaw 将 bundle MCP 配置作为 `mcpServers` 合并到
  有效的嵌入式 OpenClaw 设置中。
- OpenClaw 会在嵌入式 OpenClaw 智能体轮次期间，通过启动 stdio
  服务器或连接 HTTP 服务器来公开受支持的 bundle MCP 工具。
- `coding` 和 `messaging` 工具配置默认包含 bundle MCP 工具；
  使用 `tools.deny: ["bundle-mcp"]` 可针对智能体或 Gateway 网关选择停用。
- 项目本地的嵌入式智能体设置仍在 bundle 默认值之后应用，因此
  工作区设置可在需要时覆盖 bundle MCP 条目。
- bundle MCP 工具目录会在注册前进行确定性排序，因此
  上游 `listTools()` 顺序变化不会导致提示缓存的工具块频繁变动。

##### 传输协议

MCP 服务器可以使用 stdio 或 HTTP 传输协议。

**Stdio** 会启动子进程：

```json
{
  "mcp": {
    "servers": {
      "my-server": {
        "command": "node",
        "args": ["server.js"],
        "env": { "PORT": "3000" }
      }
    }
  }
}
```

**HTTP** 会连接正在运行的 MCP 服务器，除非请求
`streamable-http`，否则默认为 `sse`：

```json
{
  "mcp": {
    "servers": {
      "my-server": {
        "url": "http://localhost:3100/mcp",
        "transport": "streamable-http",
        "headers": {
          "Authorization": "Bearer ${MY_SECRET_TOKEN}"
        },
        "connectionTimeoutMs": 30000
      }
    }
  }
}
```

- `transport` 接受 `"streamable-http"` 或 `"sse"`；省略时默认为 `sse`。
- `type: "http"` 是 CLI 原生下游结构；在 OpenClaw 配置中使用 `transport: "streamable-http"`。`openclaw mcp set` 和 `openclaw doctor --fix` 会规范化该常用别名。
- 仅允许 `http:` 和 `https:` URL 方案。
- `headers` 值支持 `${ENV_VAR}` 插值。
- 同时包含 `command` 和 `url` 的服务器条目将被拒绝。
- 工具描述和日志中的 URL 凭据（用户信息和查询参数）
  会被隐去。
- `connectionTimeoutMs` 会覆盖 stdio 和 HTTP 传输协议默认的
  30 秒连接超时。请求超时默认为 60 秒，
  可使用 `requestTimeoutMs` 覆盖。

##### 工具命名

OpenClaw 使用 `serverName__toolName` 形式的提供商安全名称注册
bundle MCP 工具。例如，键为 `"vigil-harbor"` 的服务器公开
`memory_search` 工具时，会注册为 `vigil-harbor__memory_search`。

- `A-Za-z0-9_-` 之外的字符会替换为 `-`。
- 以非字母开头的片段会添加字母前缀，因此
  `12306` 等数字服务器键会变成提供商安全的工具前缀。
- 服务器前缀最长为 30 个字符。
- 完整工具名称最长为 64 个字符。
- 空服务器名称会回退为 `mcp`。
- 清理后发生冲突的名称会使用数字后缀消除歧义。
- 最终公开的工具按安全名称进行确定性排序，以保持
  重复的嵌入式智能体轮次缓存稳定。
- 配置文件筛选会将来自同一个 bundle MCP 服务器的每个工具
  视为由 `bundle-mcp` 插件拥有，因此配置文件允许/拒绝列表可以引用
  单个公开工具名称或 `bundle-mcp` 插件键。

#### 嵌入式 OpenClaw 设置

启用 bundle 后，Claude `settings.json` 会作为默认嵌入式
OpenClaw 设置导入。OpenClaw 会在应用前清理 shell 覆盖键：

- `shellPath`
- `shellCommandPrefix`

#### 嵌入式 OpenClaw LSP

- 已启用的 Claude bundle 可以提供 LSP 服务器配置。
- OpenClaw 会加载 `.lsp.json` 以及清单声明的所有 `lspServers` 路径。
- bundle LSP 配置会合并到有效的嵌入式 OpenClaw LSP
  默认值中。
- 目前只有受支持的 stdio 后端 LSP 服务器可以运行；不受支持的
  传输协议仍会显示在 `openclaw plugins inspect <id>` 中。

### 已检测但未执行

以下内容可以识别并显示在诊断信息中，但 OpenClaw 不会运行它们：

- Claude `agents`、`hooks/hooks.json` 自动化、`outputStyles`
- Cursor `.cursor/agents`、`.cursor/hooks.json`、`.cursor/rules`
- Codex 中能力报告以外的 `.app.json` 元数据

## bundle 格式

<AccordionGroup>
  <Accordion title="Codex bundle">
    标记：`.codex-plugin/plugin.json`

    可选内容：`skills/`、`hooks/`、`.mcp.json`、`.app.json`

    当 Codex bundle 使用 Skills 根目录和 OpenClaw 风格的
    Hook 包目录（`HOOK.md` + `handler.ts`）时，与 OpenClaw 的适配效果最佳。

  </Accordion>

  <Accordion title="Claude bundle">
    两种检测模式：

    - **基于清单：** `.claude-plugin/plugin.json`
    - **无清单：** 默认 Claude 布局（`skills/`、`commands/`、`agents/`、`hooks/`、`.mcp.json`、`.lsp.json`、`settings.json`）

    Claude 特有行为：

    - `commands/` 被视为 Skills 内容
    - `settings.json` 会导入嵌入式 OpenClaw 设置（shell 覆盖键会被清理）
    - `.mcp.json` 会向嵌入式 OpenClaw 公开受支持的 stdio 工具
    - `.lsp.json` 以及清单声明的 `lspServers` 路径会加载到嵌入式 OpenClaw LSP 默认值中
    - `hooks/hooks.json` 会被检测但不执行
    - 清单中的自定义组件路径是增量添加的；它们会扩展默认值，而非替换默认值

  </Accordion>

  <Accordion title="Cursor bundle">
    标记：`.cursor-plugin/plugin.json`

    可选内容：`skills/`、`.cursor/commands/`、`.cursor/agents/`、`.cursor/rules/`、`.cursor/hooks.json`、`.mcp.json`

    - `.cursor/commands/` 被视为 Skills 内容
    - `.cursor/rules/`、`.cursor/agents/` 和 `.cursor/hooks.json` 仅检测

  </Accordion>
</AccordionGroup>

## 检测优先级

OpenClaw 首先检查原生插件格式：

1. `openclaw.plugin.json` 或带有 `openclaw.extensions` 的有效 `package.json`——视为**原生插件**
2. bundle 标记（`.codex-plugin/`、`.claude-plugin/` 或默认 Claude/Cursor 布局）——视为 **bundle**

如果目录同时包含两种格式，OpenClaw 会使用原生路径。这可以防止
双格式包以 bundle 形式被部分安装。

## 运行时依赖项和清理

- 第三方兼容 bundle 不会在启动时进行 `npm install` 修复。
  它们应通过 `openclaw plugins install` 安装，并在已安装的插件目录中
  附带所需的一切内容。
- OpenClaw 自有的内置插件要么以轻量形式随核心一起提供，要么
  可通过插件安装器下载。Gateway 网关启动时绝不会为它们运行
  包管理器。
- `openclaw doctor --fix` 会移除过时的本地内置插件安装记录；
  当配置仍引用可下载插件，但本地插件索引中缺少该插件时，
  还可以恢复该插件。

## 安全性

bundle 的信任边界比原生插件更窄：

- OpenClaw **不会**在进程内加载任意 bundle 运行时模块。
- Skills 和 Hook 包路径必须位于插件根目录内（经过边界检查）。
- 读取设置文件时使用相同的边界检查。
- 受支持的 stdio MCP 服务器可能会作为子进程启动。

这使 bundle 默认更安全，但对于第三方 bundle 实际公开的功能，
仍应将其视为可信内容。

## 故障排查

<AccordionGroup>
  <Accordion title="检测到软件包，但功能无法运行">
    运行 `openclaw plugins inspect <id>`。如果某项功能已列出但标记为
    未接入，则这是产品限制，而不是安装损坏。
  </Accordion>

  <Accordion title="Claude 命令文件未显示">
    确保已启用该软件包，并且 Markdown 文件位于检测到的
    `commands/` 或 `skills/` 根目录中。
  </Accordion>

  <Accordion title="Claude 设置未生效">
    仅支持来自 `settings.json` 的嵌入式 OpenClaw 设置。OpenClaw
    不会将软件包设置视为原始配置补丁。
  </Accordion>

  <Accordion title="Claude 钩子未执行">
    `hooks/hooks.json` 仅用于检测。如果需要可运行的钩子，请使用
    OpenClaw 钩子包布局或发布原生插件。
  </Accordion>
</AccordionGroup>

## 相关内容

- [安装和配置插件](/zh-CN/tools/plugin)
- [Building Plugins](/zh-CN/plugins/building-plugins) - 创建原生插件
- [Plugin Manifest](/zh-CN/plugins/manifest) - 原生清单架构
