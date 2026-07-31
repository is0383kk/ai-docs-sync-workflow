---
read_when:
    - 你想从终端读取或写入工作区文件中的叶节点
    - 你正在编写针对工作区状态的脚本，并希望使用一种稳定且与类型无关的寻址方案
    - 你正在调试 `oc://` 路径（验证语法，并查看它解析到什么位置）
summary: '`openclaw path` 的 CLI 参考（通过 `oc://` 寻址方案检查和编辑工作区文件）'
title: 路径
x-i18n:
    generated_at: "2026-07-26T06:40:57Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 7afe5bd1c3a5fca8dd22c7d807e390e751ae7e895c54bf0e10e2734f3889436c
    source_path: cli/path.md
    workflow: 16
---

# `openclaw path`

通过 Shell 访问 `oc://` 寻址方案：一种按类型分派的路径语法，用于检查和编辑可寻址的工作区文件（markdown、jsonc、jsonl、yaml/yml/lobster）。自行托管者、插件作者和编辑器扩展可使用它读取、查找或更新一处精确位置，而无需为每种文件手工编写解析器。

`path` 由内置的可选 `oc-path` 插件提供。首次使用前请启用它：

```bash
openclaw plugins enable oc-path
```

CLI 动词与寻址模型对应：

- `resolve` 是具体的，仅匹配单个目标。
- `find` 是用于通配符、联合、谓词和位置展开的多匹配动词。
- `set` 仅接受具体路径或插入标记；通配符模式会在写入前被拒绝。
- `validate` 在不访问文件系统的情况下解析路径。
- `emit` 通过解析 + 输出对文件进行往返处理（字节保真度诊断）。

## 使用理由

OpenClaw 状态分散在人为编辑的 Markdown、带注释的 JSONC 配置、仅追加的 JSONL 日志以及 YAML 工作流/规范文件中。脚本、钩子和智能体经常只需从这些文件中获取一个小值：frontmatter 键、插件设置、日志记录字段、YAML 步骤，或指定章节下的项目符号项。

`openclaw path` 为这些调用方提供稳定地址，无需为每种文件类型临时编写 grep、正则表达式或解析器。同一个 `oc://` 路径可以在终端中进行验证、解析、搜索、试运行和写入，使精确的自动化操作可审查、可重放。它会保留文件的其他部分，因此写入一个叶节点不会扰乱其中的注释、行尾格式或附近的排版。

当目标具有逻辑地址，但文件结构各不相同时，请使用它：

- 钩子从带注释的 JSONC 中读取一项设置，并在写回该值时保留注释。
- 维护脚本查找 JSONL 日志中每个匹配的事件字段，而无需将整个日志载入自定义解析器。
- 编辑器按 slug 跳转到 Markdown 章节或项目符号项，然后渲染解析到的确切行。
- 智能体在应用小型工作区编辑前先进行试运行，并在审查中显示发生变化的字节。

对于普通的整文件编辑、复杂的配置迁移或记忆专用写入，请勿使用 `openclaw path`；这些操作应使用其所有者命令或插件。`path` 适用于小型、可寻址的文件操作，在这类场景中，可重复执行的终端命令优于另一个专用解析器。

## 使用方式

从人为编辑的配置文件中读取一个值：

```bash
openclaw path resolve 'oc://config.jsonc/plugins/github/enabled'
```

在不触碰磁盘的情况下预览写入：

```bash
openclaw path set 'oc://config.jsonc/plugins/github/enabled' 'true' --dry-run
```

在仅追加的 JSONL 日志中查找匹配的记录：

```bash
openclaw path find 'oc://session.jsonl/[event=tool_call]/name'
```

按章节和项目而不是行号寻址 Markdown 中的指令：

```bash
openclaw path resolve 'oc://AGENTS.md/runtime-safety/openclaw-gateway'
```

在 CI 或预检脚本读取或写入之前验证路径：

```bash
openclaw path validate 'oc://AGENTS.md/tools/$last/risk'
```

这些命令旨在直接复制到 Shell 脚本中。当调用方需要结构化输出时使用 `--json`，当用户检查结果时使用 `--human`。

## 工作原理

1. 将 `oc://` 地址解析为槽位：文件、章节、项目、字段和可选的会话查询。
2. 根据目标扩展名选择文件类型适配器（`.md`、`.jsonc`、`.json`、`.jsonl`、`.ndjson`、`.yaml`、`.yml`、`.lobster`）。
3. 根据相应文件类型的结构解析槽位：Markdown 标题/项目、JSONC 对象键/数组索引、JSONL 行记录或 YAML 映射/序列节点。
4. 对于 `set`，通过同一适配器输出编辑后的字节，使文件中未触及的部分在相应类型支持的情况下保留其注释、行尾格式和附近的排版。

`resolve` 和 `set` 要求一个具体目标。`find` 是探索性动词：它会将通配符、联合、谓词和序数展开为具体匹配项，供你在选择一个目标进行写入前检查。

## 子命令

| 子命令                  | 用途                                                                        |
| ----------------------- | --------------------------------------------------------------------------- |
| `resolve <oc-path>`     | 输出路径处的具体匹配项（或“未找到”）。                                      |
| `find <pattern>`        | 枚举通配符/联合/谓词路径的匹配项。                                           |
| `set <oc-path> <value>` | 在具体路径写入叶节点或插入目标。支持 `--dry-run`。                    |
| `validate <oc-path>`    | 仅解析；输出结构分解（文件/章节/项目/字段）。                                |
| `emit <file>`           | 通过解析 + 输出对文件进行往返处理（字节保真度诊断）。                        |

## 全局标志

| 标志            | 适用于                           | 用途                                                                       |
| --------------- | -------------------------------- | -------------------------------------------------------------------------- |
| `--cwd <dir>`   | `resolve`、`find`、`set`、`emit` | 相对于此目录解析文件槽位（默认值：`process.cwd()`）。                   |
| `--file <path>` | `resolve`、`find`、`set`、`emit` | 覆盖文件槽位解析出的路径（绝对路径访问）。                                 |
| `--json`        | 全部                             | 强制输出 JSON（stdout 不是 TTY 时的默认行为）。                            |
| `--human`       | 全部                             | 强制输出人类可读格式（stdout 是 TTY 时的默认行为）。                       |
| `--value-json`  | `set`                            | 将 `<value>` 解析为 JSON，用于替换 JSON/JSONC/JSONL 叶节点。       |
| `--dry-run`     | `set`                            | 输出将要写入的字节，但不实际写入。                                         |
| `--diff`        | `set`（需要 `--dry-run`）     | 输出统一差异，而不是完整字节。                                             |

`validate` 仅接受 `--json` / `--human`；它不访问文件系统，因此 `--cwd` 和 `--file` 不适用。

## `oc://` 语法

```text
oc://FILE/SECTION/ITEM/FIELD?session=SCOPE
```

槽位规则：`field` 需要 `item`，而 `item` 需要 `section`。所有四个槽位均遵循以下规则：

- **带引号的段** — `"a/b.c"` 可跨越 `/` 和 `.` 分隔符。内容按字节字面量处理；引号内不允许使用 `"` 和 `\`。文件槽位同样识别引号：`oc://"skills/email-drafter"/Tools/$last` 会将 `skills/email-drafter` 视为单个文件路径。
- **谓词** — `[k=v]`、`[k!=v]`、`[k<v]`、`[k<=v]`、`[k>v]`、`[k>=v]`。数值运算符要求两侧都能转换为有限数值。
- **联合** — `{a,b,c}` 匹配任一备选项。
- **通配符** — `*`（单个子段）和 `**`（零个或多个，递归）。`find` 接受这些通配符；`resolve` 和 `set` 会因存在歧义而拒绝它们。
- **位置** — `$first` / `$last` 解析为第一个/最后一个索引或已声明的键。
- **序数** — `#N` 表示按文档顺序排列的第 N 个匹配项。
- **插入标记** — `+`、`+key`、`+nnn` 用于按键/索引插入（与 `set` 配合使用）。
- **会话作用域** — `?session=cron-daily` 等。它与槽位嵌套相互独立。会话值为原始值，不进行百分号解码；其中不得包含控制字符或保留的查询分隔符（`?`、`&`、`%`）。

带引号、谓词或联合的段以外不允许出现保留字符（`?`、`&`、`%`）。任何位置均不允许出现控制字符（U+0000-U+001F、U+007F），包括 `session` 查询值。

规范路径保证支持 `formatOcPath(parseOcPath(path)) === path`。除第一个非空的 `session=` 值外，非规范查询参数将被忽略。

硬性限制：路径上限为 4096 字节，最多包含 4 个槽位（文件/章节/项目/字段），每个槽位最多包含 64 个以点分隔的子段，深层 JSON 路径最多包含 256 层嵌套遍历。此外，对于任何会加载 JSONC/JSON 文件的动词，超过 16 MiB 的输入都会被拒绝并返回解析诊断，而不会进行解析。

## 按文件类型寻址

| 类型          | 文件扩展名                  | 寻址模型                                                                                            |
| ------------- | --------------------------- | --------------------------------------------------------------------------------------------------- |
| Markdown      | `.md`                       | 按 slug 寻址 H2 章节，按 slug 或 `#N` 寻址项目符号项，通过 `[frontmatter]` 寻址 frontmatter。 |
| JSONC/JSON    | `.jsonc`、`.json`           | 对象键和数组索引；除非带引号，否则点号会拆分嵌套子段。                                             |
| JSONL         | `.jsonl`、`.ndjson`         | 顶层行地址（`L1`、`L2`、`$first`、`$last`），然后在行内按 JSONC 风格向下寻址。 |
| YAML/.lobster | `.yaml`、`.yml`、`.lobster` | 映射键和序列索引；注释和流式样式由 YAML 文档 API 处理。                                            |

`resolve` 返回结构化匹配项：`root`、`node`、`leaf` 或 `insertion-point`，并附带从 1 开始的行号。叶节点值以文本和 `leafType` 的形式提供，使插件作者无需依赖各文件类型的 AST 结构即可渲染预览。

## 变更约定

`set` 写入一个具体目标：

- Markdown frontmatter 值和 `- key: value` 项字段都是字符串
  叶节点。Markdown 插入操作会追加章节、frontmatter 键或章节
  项，并为更改后的文件渲染规范的 Markdown 结构。无法通过 `set`
  整体写入章节正文。
- JSONC 叶节点写入会将字符串值强制转换为现有叶节点类型
  （`string`、有限 `number`、`true`/`false` 或 `null`）。当 JSONC/JSON/JSONL 叶节点替换应将 `<value>` 解析为 JSON
  且可能改变结构时，请使用 `--value-json`，例如将字符串形式的 secret-ref 简写替换为
  对象。JSONC 对象和数组插入会将 `<value>` 解析为 JSON，并对普通叶节点写入使用
  `jsonc-parser` 编辑路径，同时保留注释
  和附近的格式。
- JSONL 叶节点写入会在行内像 JSONC 一样进行强制转换。整行替换
  和追加会将 `<value>` 解析为 JSON。渲染后的 JSONL 会保留文件中
  占主导地位的 LF/CRLF 行尾约定（根据整个文件的
  换行符进行多数表决，因此主要使用 CRLF 的文件即使夹杂少量 LF，也仍会保持 CRLF）。
- YAML 叶节点写入会强制转换为现有标量类型（`string`、有限
  `number`、`true`/`false` 或 `null`）。YAML 插入使用内置
  `yaml` 包的文档 API 更新映射/序列。存在解析器错误的格式错误 YAML
  文档会在修改前被拒绝，并返回
  `parse-error`。

当精确字节很重要时，请在用户可见的写入操作之前使用 `--dry-run`。JSONC
和 YAML 编辑会修补现有文档（通过 `jsonc-parser` 或 `yaml`
文档 API），因此未触及的字节通常会保留；Markdown 在每次编辑时都会根据
解析后的结构重建文件，这可能会规范化已更改叶节点之外的附带
格式。如果希望将预览显示为聚焦的前后差异补丁，而不是完整渲染的文件，
请添加 `--diff`。

## 示例

```bash
# 验证路径（不访问文件系统）
openclaw path validate 'oc://AGENTS.md/Tools/$last/risk'

# 读取叶节点
openclaw path resolve 'oc://gateway.jsonc/version'

# 通配符搜索
openclaw path find 'oc://session.jsonl/*/event' --file ./logs/session.jsonl

# 试运行写入
openclaw path set 'oc://gateway.jsonc/version' '2.0' --dry-run

# 以统一 diff 格式试运行写入
openclaw path set 'oc://gateway.jsonc/version' '2.0' --dry-run --diff

# 应用写入
openclaw path set 'oc://gateway.jsonc/version' '2.0'

# 字节保真往返转换（诊断）
openclaw path emit ./AGENTS.md
```

更多语法示例：

```bash
# 为包含 / 或 . 的键加引号
openclaw path resolve 'oc://config.jsonc/agents.defaults.models/"anthropic/claude-opus-4-7"/alias'

# 深层 JSON/JSONC 路径可以使用斜杠分段；它们会规范化为点分子段
openclaw path set 'oc://openclaw.json/agents/list/0/tools/exec/security' 'allowlist' --dry-run

# 用解析后的对象替换 JSONC 叶节点
openclaw path set 'oc://openclaw.json/gateway/auth/token' '{"source":"file","provider":"secrets","id":"/test"}' --value-json --dry-run

# 对 JSONC 子节点进行谓词搜索
openclaw path find 'oc://config.jsonc/plugins/[enabled=true]/id'

# 插入 JSONC 数组
openclaw path set 'oc://config.jsonc/items/+1' '{"id":"new","enabled":true}' --dry-run

# 插入 JSONC 对象键
openclaw path set 'oc://config.jsonc/plugins/+github' '{"enabled":true}' --dry-run

# 追加 JSONL 事件
openclaw path set 'oc://session.jsonl/+' '{"event":"checkpoint","ok":true}' --file ./logs/session.jsonl

# 解析最后一个 JSONL 值行
openclaw path resolve 'oc://session.jsonl/$last/event' --file ./logs/session.jsonl

# 解析 YAML 工作流步骤
openclaw path resolve 'oc://workflow.yaml/steps/0/id'

# 更新 YAML 标量
openclaw path set 'oc://workflow.yaml/steps/$last/id' 'classify-renamed' --dry-run

# 寻址 Markdown frontmatter
openclaw path resolve 'oc://AGENTS.md/[frontmatter]/name'

# 插入 Markdown frontmatter
openclaw path set 'oc://AGENTS.md/[frontmatter]/+description' 'Agent instructions' --dry-run

# 查找 Markdown 项字段
openclaw path find 'oc://SKILL.md/Tools/*/send_email'

# 验证会话范围内的路径
openclaw path validate 'oc://AGENTS.md/Tools/$last/risk?session=cron-daily'
```

## 按文件类型分类的用法

相同的五个动词适用于所有类型；寻址方案根据
文件扩展名进行分派。

### Markdown

```text
<!-- frontmatter.md -->
---
name: drafter
description: 邮件起草智能体
tier: core
---
## 工具
- gh: GitHub CLI
- curl: HTTP 客户端
- send_email: 已启用
```

```bash
$ openclaw path resolve 'oc://x.md/[frontmatter]/tier' --file frontmatter.md --human
叶节点 @ L4: "core"（字符串）

$ openclaw path resolve 'oc://x.md/tools/gh/gh' --file frontmatter.md --human
叶节点 @ L9: "GitHub CLI"（字符串）

$ openclaw path find 'oc://x.md/tools/*' --file frontmatter.md --human
oc://x.md/tools/* 有 3 个匹配项：
  oc://x.md/tools/gh           →  节点 @ L9 [md-item]
  oc://x.md/tools/curl         →  节点 @ L10 [md-item]
  oc://x.md/tools/send-email   →  节点 @ L11 [md-item]
```

`[frontmatter]` 谓词寻址 YAML frontmatter 块；`tools`
通过 slug 匹配 `## Tools` 标题，即使源内容使用下划线，项叶节点也会保留其 slug 形式
（`send_email` 会变为 `send-email`）。

### JSONC

```text
// config.jsonc
{
  "plugins": {
    "github": {"enabled": true, "role": "vcs"},
    "slack":  {"enabled": false, "role": "chat"}
  }
}
```

```bash
$ openclaw path resolve 'oc://config.jsonc/plugins/github/enabled' --file config.jsonc --human
叶节点 @ L4: "true"（布尔值）

$ openclaw path set 'oc://config.jsonc/plugins/slack/enabled' 'true' --file config.jsonc --dry-run
--dry-run：将向 /…/config.jsonc 写入 142 字节
{
  "plugins": {
    "github": {"enabled": true, "role": "vcs"},
    "slack":  {"enabled": true, "role": "chat"}
  }
}
```

JSONC 编辑通过 `jsonc-parser` 进行，因此注释和空白在
`set` 后仍会保留。请先使用 `--dry-run` 运行，以便在提交前检查字节。
`.json` 文件使用与 `.jsonc` 相同的适配器和编辑路径。

### JSONL

```text
{"event":"start","userId":"u1","ts":1}
{"event":"action","userId":"u1","ts":2}
{"event":"end","userId":"u1","ts":3}
```

```bash
$ openclaw path find 'oc://session.jsonl/[event=action]/userId' --file session.jsonl --human
oc://session.jsonl/[event=action]/userId 有 1 个匹配项：
  oc://session.jsonl/L2/userId  →  叶节点 @ L2: "u1"（字符串）

$ openclaw path resolve 'oc://session.jsonl/L2/ts' --file session.jsonl --human
叶节点 @ L2: "2"（数字）
```

每一行都是一条记录。不知道行号时按谓词（`[event=action]`）寻址，
知道行号时则按规范的 `LN` 段寻址。
`.ndjson` 文件使用与 `.jsonl` 相同的适配器。

### YAML

```text
# workflow.yaml
name: inbox-triage
steps:
  - id: fetch
    command: gmail.search
  - id: classify
    command: openclaw.invoke
```

```bash
$ openclaw path resolve 'oc://workflow.yaml/steps/0/id' --file workflow.yaml --human
叶节点 @ L3: "fetch"（字符串）

$ openclaw path set 'oc://workflow.yaml/steps/$last/id' 'classify-renamed' --file workflow.yaml --dry-run
--dry-run：将向 /…/workflow.yaml 写入 99 字节
name: inbox-triage
steps:
  - id: fetch
    command: gmail.search
  - id: classify-renamed
    command: openclaw.invoke
```

YAML 使用 `yaml` 包的 `Document` API，而不是手写
解析器，因此普通的解析/输出往返转换会保留注释和编写
格式，同时解析后的路径使用与 JSONC 相同的映射键/序列索引模型。
同一适配器会处理 `.yaml`、`.yml` 和 `.lobster` 文件。

## 子命令参考

### `resolve <oc-path>`

读取单个叶节点或节点。拒绝通配符——请改用 `find`。
匹配时以 `0` 退出，无匹配且无错误时以 `1` 退出，遇到解析错误或被拒绝的
模式时以 `2` 退出。

```bash
openclaw path resolve 'oc://AGENTS.md/tools/gh/risk' --human
openclaw path resolve 'oc://gateway.jsonc/server/port' --json
```

### `find <pattern>`

枚举通配符/谓词/并集模式的每个匹配项。存在至少一个匹配项时以 `0`
退出，零匹配时以 `1` 退出。文件槽位通配符会被拒绝并返回
`OC_PATH_FILE_WILDCARD_UNSUPPORTED`——请传入具体文件（多文件
glob 是后续功能）。

```bash
openclaw path find 'oc://AGENTS.md/tools/**/risk'
openclaw path find 'oc://session.jsonl/[event=action]/userId'
openclaw path find 'oc://config.jsonc/plugins/{github,slack}/enabled'
```

### `set <oc-path> <value>`

写入叶节点。与 `--dry-run` 配合使用，可在不修改文件的情况下预览将要
写入的字节。添加 `--diff` 可预览统一 diff。
写入成功时以 `0` 退出；底层拒绝操作时（例如命中
哨兵防护）以 `1` 退出；发生解析错误时以 `2` 退出。

```bash
openclaw path set 'oc://gateway.jsonc/version' '2.0' --dry-run
openclaw path set 'oc://gateway.jsonc/version' '2.0' --dry-run --diff
openclaw path set 'oc://gateway.jsonc/version' '2.0'
openclaw path set 'oc://AGENTS.md/Tools/+gh/risk' 'low'
```

如果指定名称的子节点尚不存在，`+key` 插入标记会创建它；
`+nnn` 和裸 `+` 分别用于按索引插入和追加插入。

### `validate <oc-path>`

仅解析检查。不访问文件系统。适用于在替换变量前确认
模板路径格式正确，或希望获取结构化分解信息以进行调试的情况：

```bash
$ openclaw path validate 'oc://AGENTS.md/tools/gh' --human
有效：oc://AGENTS.md/tools/gh
  文件：    AGENTS.md
  章节： tools
  项：    gh
```

有效时以 `0` 退出；无效时以 `1` 退出（包含结构化的 `code`
和 `message`）；发生参数错误时以 `2` 退出。

### `emit <file>`

通过各文件类型对应的解析器和输出器对文件进行往返转换。对于格式正确的文件，输出应
与输入逐字节相同；出现差异表示存在
解析器错误或命中了哨兵。适用于在
真实输入上调试底层行为。

```bash
openclaw path emit ./AGENTS.md
openclaw path emit ./gateway.jsonc --json
```

## 退出代码

| 代码 | 含义                                                                    |
| ---- | -------------------------------------------------------------------------- |
| `0`  | 成功。（`resolve` / `find`：至少有一个匹配项。`set`：写入成功。） |
| `1`  | 无匹配项，或 `set` 被底层拒绝（无系统级错误）。      |
| `2`  | 参数或解析错误。                                                   |

## 输出模式

`openclaw path` 可感知 TTY：在终端上输出人类可读内容，stdout
通过管道传输或重定向时输出 JSON。`--json` 和 `--human` 可覆盖
自动检测。

## 注意事项

- `set` 通过底层基质的 emit 路径写入字节，该路径会自动应用
  脱敏哨兵保护。携带
  `__OPENCLAW_REDACTED__`（原样或作为子字符串）的叶节点会在写入
  时被拒绝。
- JSONC 解析和叶节点编辑使用插件本地的 `jsonc-parser`
  依赖项，因此常规叶节点写入会保留注释和格式，
  而不会经过手工编写的解析器/重新渲染路径。
- `path` 不感知最后已知良好（LKG）配置的跟踪或恢复；
  该生命周期由其他位置负责。如果通过 `path` 编辑的文件
  也受 LKG 跟踪，则下一次读取配置时会决定是将其提升还是
  恢复；应将 `path` 编辑视为对
  该文件的任何其他直接写入。

## 相关内容

- [CLI 参考](/zh-CN/cli)
