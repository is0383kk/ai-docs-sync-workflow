---
read_when:
    - 你想管理智能体钩子
    - 你想检查 Hooks 是否可用，或启用工作区 Hooks
summary: '`openclaw hooks`（智能体钩子）的 CLI 参考'
title: Hooks
x-i18n:
    generated_at: "2026-07-26T06:40:44Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: d4d58ea2270cf5122018f7be2943401229929f48f448b15fdd126d1cc99e1e56
    source_path: cli/hooks.md
    workflow: 16
---

# `openclaw hooks`

管理智能体钩子（由 `/new`、`/reset` 等命令及 Gateway 网关启动事件驱动的自动化）。单独使用 `openclaw hooks` 等同于 `openclaw hooks list`。

相关：[Hooks](/zh-CN/automation/hooks) - [插件钩子](/zh-CN/plugins/hooks)

## 列出钩子

```bash
openclaw hooks list [--eligible] [--json] [-v|--verbose]
```

列出从工作区、托管、额外和内置目录中发现的钩子。

- `--eligible`：仅显示满足要求的钩子。
- `--json`：结构化输出。
- `-v, --verbose`：包含一列 Missing，显示未满足的要求。

```
钩子（4/5 个就绪）

就绪：
  🚀 boot-md ✓ - Gateway 网关启动时运行 BOOT.md
  📎 bootstrap-extra-files ✓ - 智能体引导期间注入其他工作区引导文件
  📝 command-logger ✓ - 将所有命令事件记录到集中式审计文件
  💾 session-memory ✓ - 发出 /new 或 /reset 命令时将会话上下文保存到记忆中
```

## 获取钩子信息

```bash
openclaw hooks info <name> [--json]
```

`<name>` 是钩子名称或钩子键（例如 `session-memory`）。显示来源、文件/处理程序路径、主页、事件以及各项要求的状态（二进制文件、环境变量、配置、操作系统）。

## 检查可用性

```bash
openclaw hooks check [--json]
```

输出就绪/未就绪数量摘要；存在未就绪的钩子时，会逐一列出并说明阻塞原因。

## 启用钩子

```bash
openclaw hooks enable <name>
```

在配置中添加/更新 `hooks.internal.entries.<name>.enabled = true`，并同时打开 `hooks.internal.enabled` 总开关（在至少配置一个钩子之前，Gateway 网关不会加载任何内部钩子处理程序）。如果钩子不存在、由插件管理或不符合可用条件（缺少必要要求），则操作失败。

由插件管理的钩子会在 `hooks list` 中显示 `plugin:<id>`，无法在此处启用或禁用；请改为启用或禁用其所属插件。

启用后重启 Gateway 网关（重启 macOS 菜单栏应用，或在开发环境中重启 Gateway 网关进程），以便重新加载钩子。

## 禁用钩子

```bash
openclaw hooks disable <name>
```

设置 `hooks.internal.entries.<name>.enabled = false`。之后重启 Gateway 网关。

## 安装和更新钩子包

```bash
openclaw plugins install <package>        # 默认使用 npm
openclaw plugins install npm:<package>    # 仅使用 npm
openclaw plugins install <package> --pin  # 固定解析后的版本
openclaw plugins install <path>           # 本地目录或归档文件
openclaw plugins install -l <path>        # 链接本地目录而非复制

openclaw plugins update <id>
openclaw plugins update --all
openclaw plugins update --dry-run
```

钩子包通过统一的插件安装器/更新器进行安装；`openclaw hooks install` / `openclaw hooks update` 仍作为已弃用的别名可用，它们会输出警告并转发到 `plugins` 命令。

- Npm 规格仅支持注册表：软件包名称加可选的精确版本或 dist-tag。Git/URL/文件规格和 semver 范围会被拒绝。依赖项通过 `--ignore-scripts` 在项目本地安装。
- 不带前缀的规格和 `@latest` 使用稳定版本通道；如果 npm 解析到预发布版本，OpenClaw 会停止并要求你明确选择加入（`@beta`、`@rc` 或精确的预发布版本）。
- 支持的归档格式：`.zip`、`.tgz`、`.tar.gz`、`.tar`。
- `-l, --link` 会链接本地目录而非复制（将其添加到 `hooks.internal.load.extraDirs`）；链接的钩子包是来自操作员所配置目录的托管钩子，而非工作区钩子。
- `--pin` 将 npm 安装以精确解析的 `name@version` 形式记录在共享 SQLite 状态中。
- 安装操作会将钩子包复制到 `~/.openclaw/hooks/<id>`，在 `hooks.internal.entries.*` 下启用其中的钩子，并在共享 SQLite 状态中记录安装来源。
- 如果已存储的完整性哈希不再与获取的工件匹配，OpenClaw 会发出警告并在继续前提示确认；传入全局 `--yes` 可绕过提示（例如在 CI 中）。

## 内置钩子

| 钩子                  | 事件                                            | 功能                                                                                       |
| --------------------- | ------------------------------------------------- | -------------------------------------------------------------------------------------------------- |
| boot-md               | `gateway:startup`                                 | Gateway 网关启动时，针对每个已配置的智能体作用域运行 `BOOT.md`                                  |
| bootstrap-extra-files | `agent:bootstrap`                                 | 智能体引导期间注入额外的引导文件（例如 monorepo 的 `AGENTS.md`/`TOOLS.md`） |
| command-logger        | `command`                                         | 将命令事件记录到 `~/.openclaw/logs/commands.log`                                             |
| compaction-notifier   | `session:compact:before`, `session:compact:after` | 会话压缩开始和结束时发送可见的聊天通知                             |
| session-memory        | `command:new`, `command:reset`                    | 在执行 `/new` 或 `/reset` 时将会话上下文保存到记忆中                                              |

使用 `openclaw hooks enable <hook-name>` 可启用任意内置钩子。完整详情、配置键和默认值：[内置钩子](/zh-CN/automation/hooks#bundled-hooks)。

### command-logger 日志文件

```bash
tail -n 20 ~/.openclaw/logs/commands.log        # 最近的命令
cat ~/.openclaw/logs/commands.log | jq .          # 美化输出
grep '"action":"new"' ~/.openclaw/logs/commands.log | jq .   # 按操作筛选
```

## 注意事项

- `hooks list --json`、`info --json` 和 `check --json` 会将结构化 JSON 直接写入标准输出。

## 相关内容

- [CLI 参考](/zh-CN/cli)
- [自动化钩子](/zh-CN/automation/hooks)
