---
read_when:
    - 你使用语音通话插件，并且希望了解所有 CLI 入口点
    - 你需要 setup、smoke、call、continue、speak、dtmf、end、status、tail、latency、expose 和 start 的标志表及默认值
summary: '`openclaw voicecall` 的 CLI 参考（语音通话插件命令界面）'
title: 语音通话
x-i18n:
    generated_at: "2026-07-26T06:39:01Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: aec445886cccb79c9212dd9f1f448ff9634274deb380632be786478c9bb29670
    source_path: cli/voicecall.md
    workflow: 16
---

# `openclaw voicecall`

`voicecall` 是由插件提供的命令。它仅在语音通话插件已安装并启用时出现。

Gateway 网关运行时，操作命令（`call`、`start`、
`continue`、`speak`、`dtmf`、`end`、`status`）会路由到该 Gateway 网关的
语音通话运行时。如果无法连接任何 Gateway 网关，则会回退到独立的
CLI 运行时。

## 子命令

```bash
openclaw voicecall setup    [--json]
openclaw voicecall smoke    [-t <phone>] [--message <text>] [--mode <m>] [--yes] [--json]
openclaw voicecall call     -m <text> [-t <phone>] [--mode <m>]
openclaw voicecall start    --to <phone> [--message <text>] [--mode <m>]
openclaw voicecall continue --call-id <id> --message <text>
openclaw voicecall speak    --call-id <id> --message <text>
openclaw voicecall dtmf     --call-id <id> --digits <digits>
openclaw voicecall end      --call-id <id>
openclaw voicecall status   [--call-id <id>] [--json]
openclaw voicecall tail     [--file <path>] [--since <n>] [--poll <ms>]
openclaw voicecall latency  [--file <path>] [--last <n>]
openclaw voicecall expose   [--mode <m>] [--path <p>] [--port <port>] [--serve-path <p>]
```

| 子命令 | 说明                                                     |
| ---------- | --------------------------------------------------------------- |
| `setup`    | 显示提供商和 Webhook 就绪状态检查。                     |
| `smoke`    | 运行就绪状态检查；仅在使用 `--yes` 时拨打实时测试电话。 |
| `call`     | 发起出站语音通话。                                |
| `start`    | `call` 的别名，其中 `--to` 为必填项，`--message` 为可选项。 |
| `continue` | 播报消息并等待下一条响应。                 |
| `speak`    | 播报消息，但不等待响应。                 |
| `dtmf`     | 向活动通话发送 DTMF 数字。                             |
| `end`      | 挂断活动通话。                                         |
| `status`   | 检查活动通话（或通过 `--call-id` 检查指定通话）。                   |
| `tail`     | 跟踪 `calls.jsonl`（在提供商测试期间很有用）。              |
| `latency`  | 汇总 `calls.jsonl` 中的轮次延迟指标。              |
| `expose`   | 为 Webhook 端点切换 Tailscale Serve/Funnel。         |

## 设置和冒烟测试

### `setup`

默认输出人类可读的就绪状态检查结果。脚本请传入 `--json`。

```bash
openclaw voicecall setup
openclaw voicecall setup --json
```

### `smoke`

运行相同的就绪状态检查。仅当 `--to` 和
`--yes` 均存在时，才会拨打真实电话。

| 标志               | 默认值                           | 说明                             |
| ------------------ | --------------------------------- | --------------------------------------- |
| `-t, --to <phone>` | （无）                            | 实时冒烟测试要拨打的电话号码。  |
| `--message <text>` | `OpenClaw voice call smoke test.` | 冒烟测试通话期间要播报的消息。 |
| `--mode <mode>`    | `notify`                          | 通话模式：`notify` 或 `conversation`。  |
| `--yes`            | `false`                           | 实际拨打实时出站电话。  |
| `--json`           | `false`                           | 输出机器可读的 JSON。            |

```bash
openclaw voicecall smoke
openclaw voicecall smoke --to "+15555550123"        # 试运行
openclaw voicecall smoke --to "+15555550123" --yes  # 实时通知通话
```

<Note>
对于外部提供商（`plivo`、`telnyx`、`twilio`），`setup` 和 `smoke` 需要来自 `publicUrl`、隧道或 Tailscale 暴露的公共 Webhook URL。系统会拒绝 local loopback 或私有 Serve 回退，因为运营商无法访问它。
</Note>

## 通话生命周期

### `call`

发起出站语音通话。

| 标志                   | 必填 | 默认值           | 说明                                                                |
| ---------------------- | -------- | ----------------- | -------------------------------------------------------------------------- |
| `-m, --message <text>` | 是      | （无）            | 通话接通时要播报的消息。                                   |
| `-t, --to <phone>`     | 否       | 配置 `toNumber` | 要拨打的 E.164 电话号码。                                                |
| `--mode <mode>`        | 否       | `conversation`    | 通话模式：`notify`（消息播报后挂断）或 `conversation`（保持接通）。 |

```bash
openclaw voicecall call --to "+15555550123" --message "你好"
openclaw voicecall call -m "提醒一下" --mode notify
```

### `start`

`call` 的别名，采用不同的默认标志形式。

| 标志               | 必填 | 默认值        | 说明                              |
| ------------------ | -------- | -------------- | ---------------------------------------- |
| `--to <phone>`     | 是      | （无）         | 要拨打的电话号码。                    |
| `--message <text>` | 否       | （无）         | 通话接通时要播报的消息。 |
| `--mode <mode>`    | 否       | `conversation` | 通话模式：`notify` 或 `conversation`。   |

### `continue`

播报消息并等待响应。

| 标志               | 必填 | 说明       |
| ------------------ | -------- | ----------------- |
| `--call-id <id>`   | 是      | 通话 ID。          |
| `--message <text>` | 是      | 要播报的消息。 |

### `speak`

播报消息，但不等待响应。

| 标志               | 必填 | 说明       |
| ------------------ | -------- | ----------------- |
| `--call-id <id>`   | 是      | 通话 ID。          |
| `--message <text>` | 是      | 要播报的消息。 |

### `dtmf`

向活动通话发送 DTMF 数字。

| 标志                | 必填 | 说明                                      |
| ------------------- | -------- | ------------------------------------------------ |
| `--call-id <id>`    | 是      | 通话 ID。                                         |
| `--digits <digits>` | 是      | DTMF 数字（例如，使用 `ww123456#` 表示等待）。 |

### `end`

挂断活动通话。

| 标志             | 必填 | 说明 |
| ---------------- | -------- | ----------- |
| `--call-id <id>` | 是      | 通话 ID。    |

### `status`

检查活动通话。

| 标志             | 默认值 | 说明                  |
| ---------------- | ------- | ---------------------------- |
| `--call-id <id>` | （无）  | 将输出限制为一个通话。 |
| `--json`         | `false` | 输出机器可读的 JSON。 |

```bash
openclaw voicecall status
openclaw voicecall status --json
openclaw voicecall status --call-id <id>
```

## 日志和指标

### `tail`

跟踪语音通话 JSONL 日志。启动时输出最后 `--since` 行，然后
在写入新行时对其进行流式输出。

| 标志            | 默认值                    | 说明                    |
| --------------- | -------------------------- | ------------------------------ |
| `--file <path>` | 从插件存储解析 | `calls.jsonl` 的路径。         |
| `--since <n>`   | `25`                       | 开始跟踪前要输出的行数。 |
| `--poll <ms>`   | `250`（最小值为 50）         | 轮询间隔（毫秒）。 |

### `latency`

汇总 `calls.jsonl` 中的轮次延迟和监听等待指标。输出为
JSON，其中包含 `recordsScanned`、`turnLatency` 和 `listenWait` 摘要。

| 标志            | 默认值                    | 说明                          |
| --------------- | -------------------------- | ------------------------------------ |
| `--file <path>` | 从插件存储解析 | `calls.jsonl` 的路径。               |
| `--last <n>`    | `200`（最小值为 1）          | 要分析的最近记录数。 |

## 暴露 Webhook

### `expose`

为语音 Webhook 启用、禁用或更改 Tailscale Serve/Funnel 配置。

| 标志                  | 默认值                                   | 说明                                     |
| --------------------- | ----------------------------------------- | ----------------------------------------------- |
| `--mode <mode>`       | `funnel`                                  | `off`、`serve`（tailnet）或 `funnel`（公共）。 |
| `--path <path>`       | 配置 `tailscale.path` 或 `--serve-path` | 要暴露的 Tailscale 路径。                       |
| `--port <port>`       | 配置 `serve.port` 或 `3334`             | 本地 Webhook 端口。                             |
| `--serve-path <path>` | 配置 `serve.path` 或 `/voice/webhook`   | 本地 Webhook 路径。                             |

```bash
openclaw voicecall expose --mode serve
openclaw voicecall expose --mode funnel
openclaw voicecall expose --mode off
```

<Warning>
仅将 Webhook 端点暴露给你信任的网络。在可行的情况下，优先使用 Tailscale Serve，而不是 Funnel。
</Warning>

## 相关内容

- [CLI 参考](/zh-CN/cli)
- [语音通话插件](/zh-CN/plugins/voice-call)
