---
read_when:
    - 在 iOS 节点上启用 HealthKit 摘要
    - 调用 health.summary 或排查缺失的健康指标
    - 审查哪些健康数据可以离开 iOS 设备
summary: 在 iOS 节点上启用并调用受隐私权限控制的 HealthKit 摘要
title: HealthKit 摘要
x-i18n:
    generated_at: "2026-07-26T05:52:55Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: b8ac13d2870c55e2083a5e3a14c3d04238c2780a9e83d091f31923eb738476af
    source_path: platforms/ios-healthkit.md
    workflow: 16
---

# HealthKit 摘要

OpenClaw 可以从已连接的 iPhone 或 iPad 节点请求当前日历日的只读摘要。设备在本地计算汇总数据，并且仅返回步数、睡眠时长、平均静息心率以及健身记录次数和时长。不支持单条 HealthKit 样本、来源、元数据、临床记录、后台摄取和写入。

此功能默认关闭。它需要在 iOS 设备上单独同意，并在 Gateway 网关上授权。

## 要求

- 一台运行 OpenClaw iOS 应用，并且 HealthKit 报告健康数据可用的 iPhone 或 iPad。
- 一个已连接并获批准的 iOS 节点。请参阅 [iOS 应用设置](/zh-CN/platforms/ios)。
- 一个能够访问 iOS 节点的当前版本 Gateway 网关。
- 你希望查看的所有指标都具有可读的健康数据。Apple Watch 可以向 Apple 健康数据存储贡献数据，但 HealthKit 摘要不需要 OpenClaw watchOS 应用。

## 启用访问权限

### 1. 授权 Gateway 网关命令

将 `health.summary` 添加到 `openclaw.json` 中现有的 `gateway.nodes.commands.allow` 数组。保留已有的所有命令：

```json5
{
  gateway: {
    nodes: {
      commands: { allow: ["health.summary"] },
    },
  },
}
```

`health.summary` 被归类为高度涉及隐私的命令，iOS 平台默认绝不允许该命令。`gateway.nodes.commands.deny` 中的条目会覆盖允许条目。请参阅[节点命令策略](/zh-CN/nodes#command-policy)。

### 2. 在 iOS 设备上启用共享

在 iOS 应用中：

1. 打开 **Settings -> Permissions**，然后在始终可见的 **Apple Health** 部分中找到 **Apple Health Summaries**。
2. 轻触 **Enable Apple Health Summaries**。
3. 阅读披露说明，然后在 Apple 的权限表单中选择 OpenClaw 可以读取的健康类别。

该开关会记录你明确作出的 OpenClaw 共享选择。它并不表示 Apple 已授予所有请求类别的权限。

启用健康摘要会将 `health.summary` 添加到节点声明的命令表面。批准由此产生的节点配对更新：

```bash
openclaw nodes pending
openclaw nodes approve <requestId>
```

然后验证已连接的 iOS 设备是否公开有效的 `health.summary` 命令：

```bash
openclaw nodes describe --node "<iOS device name>"
```

## 请求今日摘要

仅支持 `today`。它涵盖从本地午夜到请求时刻的时间段，并使用 iOS 设备当前的日历和时区。

```bash
openclaw nodes invoke \
  --node "<iOS device name>" \
  --command health.summary \
  --params '{"period":"today"}' \
  --json
```

智能体可以使用 `nodes` 工具调用同一命令：

```json
{
  "action": "invoke",
  "node": "<iOS device name>",
  "invokeCommand": "health.summary",
  "invokeParamsJson": "{\"period\":\"today\"}"
}
```

摘要载荷包含：

| 字段                     | 含义                                      |
| ------------------------ | ----------------------------------------- |
| `period`                 | 始终为 `today`                           |
| `startISO`               | 当地一天的开始时间，编码为 ISO 时刻        |
| `endISO`                 | 请求时间，编码为 ISO 时刻                  |
| `timeZoneIdentifier`     | iOS 设备的时区标识符                       |
| `stepCount`              | 四舍五入后的累计步数                       |
| `sleepDurationMinutes`   | 去重后的睡眠时间，范围截取为今日            |
| `restingHeartRateBpm`    | 平均静息心率                               |
| `workoutCount`           | 今日开始的健身记录                         |
| `workoutDurationMinutes` | 这些健身记录的总时长                       |

指标字段是可选的；当 HealthKit 未返回可读值时会省略这些字段。计算时长之前会合并睡眠阶段和重叠来源，因此同一分钟不会被重复计算。

## 隐私行为

- 汇总在 iOS 设备上进行。原始样本不会离开设备。
- 请求的汇总数据会通过你的 Gateway 网关离开设备。当智能体请求该数据时，汇总数据会到达已配置的 AI 提供商，并且可能保留在聊天历史记录中。直接通过 CLI 调用会将其返回给 CLI 操作员。
- OpenClaw 仅请求读取权限。它无法添加或修改健康数据。
- OpenClaw 仅在调用 `health.summary` 时读取 HealthKit。不会在后台摄取健康数据。
- HealthKit 特意不透露读取权限是否遭到拒绝。指标缺失可能意味着访问被拒绝、没有匹配的样本，或者数据类型不可用。OpenClaw 无法区分这些情况。
- 此摘要用于提供个人健康和健身背景信息，而非诊断或医疗建议。

要停止共享，请返回 **Apple Health Summaries** 并轻触 **Turn Off Summaries**。随后，iOS 设备会从其节点表面移除健康功能和 `health.summary` 命令。你也可以从 `gateway.nodes.commands.allow` 中移除 `health.summary`，以关闭 Gateway 网关侧的权限门控。

## 故障排查

### 节点未声明该命令

确认已在 iOS 应用中启用 Apple 健康摘要，并且设备已连接。运行 `openclaw nodes pending` 并批准所有功能更新，然后再次检查 `openclaw nodes describe --node "<iOS device name>"`。

### 命令需要明确选择启用

将 `health.summary` 添加到 `gateway.nodes.commands.allow`。还要检查 `gateway.nodes.commands.deny` 是否包含该命令；拒绝列表优先。

### `HEALTH_ACCESS_DISABLED`

应用侧的共享开关已关闭。在 iOS 设备上的 **Settings -> Permissions -> Apple Health** 下启用 **Apple Health Summaries**。

### 摘要请求成功，但指标缺失

打开 Apple 的健康应用并确认其中存在今日数据。在 Apple 的健康设置中检查 OpenClaw 的访问权限，但不要将空结果视为访问被拒绝的证据：HealthKit 会有意隐藏这种区别。

### 较早的时间范围失败

该命令仅接受 `{"period":"today"}`。不支持多日和历史摘要。

## 相关内容

- [iOS 应用](/zh-CN/platforms/ios)
- [节点](/zh-CN/nodes)
- [Gateway 配置参考](/zh-CN/gateway/configuration-reference#gateway)
- [安全审计](/zh-CN/gateway/security)
