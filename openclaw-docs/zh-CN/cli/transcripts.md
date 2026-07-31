---
read_when:
    - 你想从终端读取已存储的转录摘要
    - 你需要获取转录内容的 Markdown 摘要路径
    - 你正在调试核心对话记录的存储布局
summary: '`openclaw transcripts` 的 CLI 参考（列出、显示和导出已存储的对话记录）'
title: 会话记录 CLI
x-i18n:
    generated_at: "2026-07-26T05:45:21Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: c04ba637fb46ec271383b2f0d17655e18018e07f489c34dc3fd14ad926f27aa4
    source_path: cli/transcripts.md
    workflow: 16
---

# `openclaw transcripts`

用于检查和导出持久化会议转录的命令。Google Meet、Microsoft Teams 和 Zoom 浏览器参与者会自动捕获笔记；`transcripts` 智能体工具还支持通过提供商捕获和手动导入。

规范转录状态存储在位于 `$OPENCLAW_STATE_DIR/state/openclaw.sqlite` 的共享 SQLite 数据库中。`show` 和 `path` 会显式地将面向用户的工件具体化到状态目录下：

```text
$OPENCLAW_STATE_DIR/transcripts/YYYY-MM-DD/<session>/
  metadata.json
  transcript.jsonl
  summary.json
  summary.md
```

这些文件是导出内容，而不是第二个运行时存储。OpenClaw 在捕获、摘要或列出期间不会将它们读回。默认状态目录为 `~/.openclaw`；可使用 `OPENCLAW_STATE_DIR` 覆盖。日期目录取自会话开始时间；会话目录是从会话 ID 派生的文件系统安全 slug。

## 命令

```bash
openclaw transcripts list
openclaw transcripts show <session>
openclaw transcripts show YYYY-MM-DD/<session>
openclaw transcripts path <session>
openclaw transcripts path YYYY-MM-DD/<session>
openclaw transcripts path <session> --dir
openclaw transcripts path <session> --metadata
openclaw transcripts path <session> --transcript
openclaw transcripts list --json
openclaw transcripts show <session> --json
openclaw transcripts path <session> --json
```

| 命令                          | 说明                                               |
| ----------------------------- | -------------------------------------------------- |
| `list`                        | 列出已存储的会话。                                 |
| `show <session>`              | 打印并具体化 `summary.md`。                  |
| `path <session>`              | 具体化并打印 `summary.md` 路径。             |
| `path <session> --dir`        | 具体化所有工件并打印其目录。                       |
| `path <session> --metadata`   | 具体化并打印 `metadata.json`。                  |
| `path <session> --transcript` | 具体化并打印 `transcript.jsonl`。                  |
| `--json`                      | 打印机器可读的输出（适用于任何子命令）。           |

`<session>` 接受裸会话 ID 或包含日期的选择器（`YYYY-MM-DD/<session>`）。当同一会话 ID 出现在多个日期时，请使用限定形式，例如 `openclaw transcripts show
2026-05-22/standup`。默认会话 ID 包含时间戳和随机后缀；仅当固定 ID 在当天唯一时，才为会话指定该 ID。

## 输出

`list` 为每个会话打印一行以制表符分隔的内容：选择器、开始时间、标题、摘要路径。

```text
2026-05-22/standup  2026-05-22T09:00:00.000Z  每周站会  /Users/user/.openclaw/transcripts/2026-05-22/standup/summary.md
```

选择器是传回 `show` 或 `path` 时最安全的值。

`list --json` 返回包含 `sessionId`、`selector`、`date`、`title`、`startedAt`、`stoppedAt`、`source`、`path`、`summaryPath`、`hasSummary` 的对象。存储的会议源 URL 仅包含源和路径；查询字符串、片段和嵌入式凭据会在持久化之前移除。

`show --json` 返回已存储的会话元数据、选择器、会话目录、摘要路径和摘要 Markdown 文本。

`path --json` 返回所选路径，以及该工件能否具体化。已存储会话的元数据和转录导出始终存在；在会话生成摘要之前，摘要路径会报告 `exists: false`。

## 每天多个会话

会话先按日期分组，再按会话 ID 分组。一天内的十场会议会形成十个同级文件夹：

```text
~/.openclaw/transcripts/2026-05-22/
  transcript-2026-05-22T09-00-00-000Z-a1b2c3d4/
  transcript-2026-05-22T10-30-00-000Z-b2c3d4e5/
  standup/
```

自动化应使用默认生成的 ID。仅当固定 ID（例如 `standup`）不会在同一日期重复时，才使用该 ID。

## 缺少摘要

实时会话会在会话停止时存储并具体化 `summary.md`；导入的转录则会在导入后立即执行此操作。在捕获仍处于活动状态、提供商在停止期间失败，或元数据在任何话语到达之前已存储时，会话可能会出现在 `list` 中但没有摘要。

使用 `path <session> --transcript` 检查原始的仅追加转录，或运行 `transcripts` 工具的 `summarize` 操作以重新生成 Markdown 摘要。

## 升级旧版文件存储

早于 SQLite 存储的 OpenClaw 版本会将规范运行时状态直接写入 `$OPENCLAW_STATE_DIR/transcripts/` 下。运行：

```bash
openclaw doctor --fix
```

Doctor 会将完整的旧版目录树导入 SQLite，验证行数和顺序，记录迁移回执，并将经过验证的源目录树移动到带时间戳的 `transcripts.migrated-*` 归档中。运行时命令不会回退到旧版文件。在确认导入的会话以及依赖的所有导出内容无误之前，请保留该归档。

## 配置

默认启用会议转录捕获。要在全局范围内选择退出：

```json
{
  "transcripts": {
    "enabled": false
  }
}
```

- `enabled`（默认值为 `true`）：启用自动会议笔记、转录工具和已配置的自动启动源。当会议笔记不应持久化到主机时，将其设置为 `false`。显式请求的会议 `transcribe` 模式会保留其现有的有界实时字幕尾部，但此设置为 false 时不会写入持久化行。
  使用 `transcripts.autoStart` 配置自动启动源。每个条目只要存在即为启用；省略条目即可禁用相应源。`discord-voice` 是内置的支持自动启动的源，需要 `guildId` 和 `channelId`：

```json
{
  "transcripts": {
    "enabled": true,
    "autoStart": [
      {
        "providerId": "discord-voice",
        "guildId": "1234567890",
        "channelId": "2345678901"
      }
    ]
  }
}
```

会议提供商 ID 为 `google-meet`、`teams` 和 `zoom`。它们的别名分别为 `googlemeet`/`meet`、`teams-meetings`/`microsoft-teams`/`msteams` 和 `zoom-meetings`。会议提供商会附加到已处于活动状态的会议机器人会话；正常加入会议不需要 `autoStart` 条目。
