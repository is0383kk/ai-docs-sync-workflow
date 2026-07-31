---
read_when:
    - 你希望将 QMD 设置为记忆后端
    - 你需要重新排序或额外索引路径等高级记忆功能
summary: 本地优先的搜索边车，支持 BM25、向量、重排序和查询扩展
title: QMD 记忆引擎
x-i18n:
    generated_at: "2026-07-26T06:13:36Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: c0e54dc9a18d834036e4c79d6b7bdecb268a29976d9f30ea6e82a56ca5d71fda
    source_path: concepts/memory-qmd.md
    workflow: 16
---

[QMD](https://github.com/tobi/qmd) 是一个本地优先的搜索边车，与 OpenClaw
并行运行。它在单个二进制文件中结合了 BM25、向量搜索和重排序，
并且可以为工作区记忆文件之外的内容建立索引。

## 相较于内置引擎的增强功能

- **重排序和查询扩展**，提高召回率。
- **为额外目录建立索引** — 项目文档、团队笔记以及磁盘上的任何内容。
- **为会话记录建立索引** — 回忆较早的对话。
- **完全本地运行** — 使用官方 llama.cpp provider 插件运行，并
  自动下载 GGUF 模型。
- **自动回退** — 如果 QMD 不可用，OpenClaw 会无缝回退到
  内置引擎。

## 入门指南

### 前置条件

- 安装 QMD：`npm install -g @tobilu/qmd` 或 `bun install -g @tobilu/qmd`
- 允许扩展的 SQLite 构建版本（macOS 上为 `brew install sqlite`）。
- QMD 必须位于 Gateway 网关的 `PATH` 中。
- macOS 和 Linux 开箱即用。Windows 最适合通过 WSL2 使用。

### 启用

```json5
{
  memory: {
    backend: "qmd",
  },
}
```

OpenClaw 会在
`~/.openclaw/agents/<agentId>/qmd/` 下创建一个自包含的 QMD 主目录，并自动管理边车生命周期
— 集合、更新和嵌入运行都由系统处理。
它优先使用当前的 QMD 集合和 MCP 查询形式，但会在需要时回退到
替代的集合模式标志和较旧的 MCP 工具名称。
启动协调还会在仍存在同名旧 QMD 集合时，重新创建过时的托管集合，
使其恢复为规范模式。

## 边车的工作原理

- OpenClaw 根据工作区记忆文件和配置的
  `memory.qmd.paths` 创建集合。QMD 适配器负责更新、嵌入、防抖和
  超时启发式策略；这些不是用户配置项。
- QMD 继续负责其 `index.sqlite`、YAML 集合配置以及每个 Agent 的
  QMD 主目录下的模型下载；这些是外部工具工件，
  并非 OpenClaw 状态表。OpenClaw 所有的协调仅存在于 SQLite 中：
  一个共享租约限制跨 Agent 的嵌入工作，而每个
  Agent 数据库中的一个租约会串行化该 Agent 的集合、更新和嵌入写入。
  运行时不再创建 QMD 文件锁边车。`openclaw doctor --fix`
  仅在确认其旧进程所有者已失效后，才会移除退役的边车。
  升级采用完全切换方式：使用新版本前，停止并重新启动共享该状态目录的
  所有 OpenClaw 进程。不支持新旧 QMD 写入器混用；运行时有意不对
  已退役的边车执行双重加锁。
- 默认工作区集合会跟踪 `MEMORY.md` 以及 `memory/`
  目录树。小写的 `memory.md` 不会作为根记忆文件建立索引。
- QMD 自身的扫描器会忽略隐藏路径，以及常见的依赖项/构建
  目录，例如 `.git`、`.cache`、`node_modules`、`vendor`、`dist` 和
  `build`。Gateway 网关启动时会保持 QMD 延迟加载；管理器会在首次使用记忆时
  初始化。
- 搜索使用配置的 `searchMode`（默认值：`search`；还支持
  `vsearch` 和 `query`）。`search` 仅使用 BM25，因此 OpenClaw 在该模式下会跳过语义
  向量就绪性探测和嵌入维护。如果某个模式
  失败，OpenClaw 会使用 `qmd query` 重试。
- 当 `searchMode` 为 `query` 时，将 `memory.qmd.rerank` 设置为 `false`，即可使用
  QMD 不含重排序器的混合查询路径（需要 QMD 2.1 或更高版本）。
  OpenClaw 会向直接 QMD CLI 路径传递 `--no-rerank`，
  并向 QMD 的 MCP 查询工具传递 `rerank: false`。
- 对于声明支持多集合过滤器的 QMD 版本，OpenClaw 会将
  相同来源的集合分组到一次 QMD 搜索调用中。较旧的 QMD 版本
  会保留兼容的逐集合回退路径。
- 如果 QMD 完全失败，OpenClaw 会回退到内置 SQLite 引擎。
  打开失败后，重复的聊天轮次尝试会短暂退避，避免
  二进制文件缺失或边车依赖项损坏造成重试风暴；
  `openclaw memory status` 和一次性 CLI 探测仍会直接重新检查 QMD。

<Info>
首次搜索可能较慢 — QMD 会在第一次运行 `qmd query` 时自动下载用于
重排序和查询扩展的 GGUF 模型（约 2 GB）。
</Info>

## 搜索性能和兼容性

OpenClaw 会确保 QMD 搜索路径同时兼容当前和较旧版本的 QMD
安装。

启动时，OpenClaw 会为每个管理器检查一次已安装 QMD 的帮助文本。如果
二进制文件声明支持多个集合过滤器，OpenClaw 会使用一条命令
搜索所有相同来源的集合：

```bash
qmd search "router notes" --json -n 10 -c memory-root-main -c memory-dir-main
```

这样可以避免为每个持久记忆集合启动一个 QMD 子进程。
会话记录集合会保留在自己的来源组中，因此混合
`memory` + `sessions` 搜索仍会向结果多样化器提供来自
两个来源的输入。

较旧的 QMD 构建版本仅接受一个集合过滤器。当 OpenClaw 检测到
此类构建版本时，会保留兼容路径并分别搜索每个集合，
然后合并结果并去重。

要手动检查已安装版本的契约，请运行：

```bash
qmd --help | grep -i collection
```

当前 QMD 帮助会提到以一个或多个集合为目标。较旧的帮助
通常只描述单个集合。

## 模型覆盖设置

QMD 模型环境变量会从 Gateway 网关进程原样传递，
因此无需添加新的 OpenClaw 配置即可全局调整 QMD：

```bash
export QMD_EMBED_MODEL="hf:Qwen/Qwen3-Embedding-0.6B-GGUF/Qwen3-Embedding-0.6B-Q8_0.gguf"
export QMD_RERANK_MODEL="/absolute/path/to/reranker.gguf"
export QMD_GENERATE_MODEL="/absolute/path/to/generator.gguf"
```

更改嵌入模型后，请重新运行嵌入，使索引与新的
向量空间匹配。

## 为额外路径建立索引

将 QMD 指向其他目录，使其内容可供搜索：

```json5
{
  memory: {
    backend: "qmd",
    qmd: {
      paths: [{ name: "docs", path: "~/notes", pattern: "**/*.md" }],
    },
  },
}
```

额外路径中的片段会在搜索结果中显示为
`qmd/<collection>/<relative-path>`。`memory_get` 能识别此前缀，并从
正确的集合根目录读取内容。

## 为会话记录建立索引

启用会话索引以回忆较早的对话。QMD 同时需要
通用的 `memory.search` 会话来源和 QMD 会话记录导出器：

```json5
{
  memory: {
    backend: "qmd",
    search: {
      experimental: { sessionMemory: true },
      sources: ["memory", "sessions"],
    },
    qmd: {
      sessions: { enabled: true },
    },
  },
}
```

会话记录会以经过清理的用户/助手轮次形式，导出到
`~/.openclaw/agents/<id>/qmd/sessions/` 下的专用 QMD 集合中。仅设置
`sources: ["sessions"]` 不会将会话记录导出到 QMD；还需启用
`rememberAcrossConversations` 或显式的 QMD 会话导出。

会话命中仍会根据
[`tools.sessions.visibility`](/zh-CN/gateway/config-tools#toolssessions) 进行过滤。默认的
`tree` 可见性包括当前会话、它派生的会话，
以及通过环境群组感知监视的同 Agent 群组会话。使用
`session.dmScope: "main"` 时，多用户私信设置中的用户会共享主
会话，并能回忆其受监视群组中的内容。使用按对端划分的
`dmScope` 实现私信隔离，或将可见性设置为 `"self"`，以停用环境
监视会话读取。其他不相关的同 Agent 会话仍需要
`"agent"` 可见性。

## 搜索范围

默认情况下，QMD 搜索结果仅在直接会话中显示（不会在
群组或频道聊天中显示）。配置 `memory.qmd.scope` 可更改此行为：

```json5
{
  memory: {
    qmd: {
      scope: {
        default: "deny",
        rules: [{ action: "allow", match: { chatType: "direct" } }],
      },
    },
  },
}
```

上面的片段就是实际的默认规则。当搜索被范围规则拒绝时，
OpenClaw 会记录一条包含派生渠道和聊天类型的警告，以便更轻松地调试
空结果。

## 引用

当 `memory.citations` 为 `auto` 或 `on` 时，搜索片段会附加
`Source: <path>#L<line>`（或 `#L<start>-L<end>`）页脚。在 `auto`
模式下，仅会为直接聊天会话添加该页脚。设置
`memory.citations = "off"` 可省略页脚，同时仍在内部将路径传递给
智能体。

## 适用场景

在有以下需求时选择 QMD：

- 通过重排序获得更高质量的结果。
- 搜索工作区之外的项目文档或笔记。
- 回忆过去的会话对话。
- 无需 API 密钥的完全本地搜索。

对于更简单的设置，[内置引擎](/zh-CN/concepts/memory-builtin) 无需额外依赖项即可良好运行。

## 故障排查

**找不到 QMD？** 请确保二进制文件位于 Gateway 网关的 `PATH` 中。如果 OpenClaw
作为服务运行，请创建符号链接：
`sudo ln -s ~/.bun/bin/qmd /usr/local/bin/qmd`。

如果 `qmd --version` 能在 shell 中运行，但 OpenClaw 仍报告
`spawn qmd ENOENT`，则 Gateway 网关进程的 `PATH` 很可能与你的
交互式 shell 不同。请显式固定二进制文件路径：

```json5
{
  memory: {
    backend: "qmd",
    qmd: {
      command: "/absolute/path/to/qmd",
    },
  },
}
```

在安装 QMD 的环境中使用 `command -v qmd`，然后通过
`openclaw memory status --deep` 重新检查。

**首次搜索非常慢？** QMD 会在首次使用时下载 GGUF 模型。请使用
`qmd query "test"` 预热，并使用与 OpenClaw 相同的 XDG 目录。

**搜索期间出现许多 QMD 子进程？** 如果可以，请更新 QMD。只有当
已安装的 QMD 声明支持多个 `-c` 过滤器时，OpenClaw
才会为相同来源的多集合搜索使用单个进程；否则，为确保正确性，
它会保留较旧的逐集合回退路径。

**仅使用 BM25 的 QMD 仍尝试构建 llama.cpp？** 请设置
`memory.qmd.searchMode = "search"`。OpenClaw 会将该模式视为
仅词法模式，跳过 QMD 向量状态探测和嵌入维护，并将
语义就绪性检查留给 `vsearch` 或 `query` 设置。

**搜索超时？** 增加 `memory.qmd.limits.timeoutMs`（默认值：4000ms）。
对于较慢的硬件，请设置更高的值，例如 `120000`。此限制适用于
Agent `memory_search` 调用期间 QMD 自身的搜索命令；设置、同步、
内置回退和补充语料库工作会保留各自更短的截止时间。

**群组或频道聊天中的结果为空？** 对于默认
`memory.qmd.scope`，这是预期行为，因为它仅允许直接会话。如果希望在这些位置显示 QMD 结果，
请为 `group` 或 `channel` 聊天类型添加
`allow` 规则。

**根记忆搜索突然变得过于宽泛？** 请重启 Gateway 网关，或等待
下一次启动协调。OpenClaw 检测到同名冲突时，会将过时的托管
集合重新创建为规范的 `MEMORY.md` 和 `memory/` 模式。

**工作区可见的临时仓库导致 `ENAMETOOLONG` 或索引损坏？**
QMD 遍历遵循底层 QMD 扫描器，而不是 OpenClaw 的
内置符号链接规则。在 QMD 提供循环安全遍历或显式排除控制之前，
请将临时 monorepo 检出目录放在 `.tmp/` 等隐藏
目录下，或放在已索引的 QMD 根目录之外。

## 配置

有关完整配置范围（`memory.qmd.*`）、搜索模式、更新间隔、
范围规则以及所有其他选项，请参阅
[记忆配置参考](/zh-CN/reference/memory-config)。

## 相关内容

- [记忆概览](/zh-CN/concepts/memory)
- [内置记忆引擎](/zh-CN/concepts/memory-builtin)
- [Honcho 记忆](/zh-CN/concepts/memory-honcho)
