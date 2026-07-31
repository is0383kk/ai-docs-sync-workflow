---
read_when:
    - 你希望 OpenClaw 识别当前活跃的 Mac
    - 你正在调试最近输入活动或活跃节点选择问题
    - 你想了解节点连接通知路由机制
summary: 检测你最近使用的 Mac，并将节点提醒路由到那里
title: 活跃计算机在线状态
x-i18n:
    generated_at: "2026-07-26T05:52:31Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: c3f1d1d0e98b1f3b7478cf80696dc693677b57897b07260cce30938e9187c314
    source_path: nodes/presence.md
    workflow: 16
---

活动计算机在线状态会告知 Gateway 网关，哪个已连接的 macOS 节点最近接收了
物理鼠标或键盘输入。OpenClaw 使用该信号将某台 Mac 标记为
`active`，为智能体提供稳定的活动节点提示，并将节点连接提醒路由到
你最可能正在使用的计算机。

这与[系统在线状态](/zh-CN/concepts/presence)不同，后者是 Gateway 网关客户端的实时
名册；它也不同于持久化的 `node.presence.alive` 信标，后者记录移动节点上次唤醒
的时间，但不会将其视为已连接。

## 要求

- OpenClaw macOS 应用已配对，并以节点模式连接。
- 已启用 **Settings -> Permissions -> Active computer detection**。该选项默认关闭。
- 已向签名的 OpenClaw 应用授予 **Accessibility** 权限。
- 对于连接提醒，还需授予 **Notifications** 权限，并且
  Mac 节点需公开 `system.notify`。

活动报告目前由原生 macOS 节点实现。iOS、Android、watchOS 和无头节点主机
可以报告连接状态或后台最近在线状态，但它们不会参与活动计算机指定的竞争。

## 检查活动计算机

1. 在 macOS 应用中，打开 **Settings -> Permissions**，启用
   **Active computer detection**，并在 macOS 系统设置中授予 **Accessibility** 权限。
2. 确认 Mac 节点已连接：

   ```bash
   openclaw nodes status --connected
   ```

3. 在该 Mac 上移动鼠标或按下按键，然后运行：

   ```bash
   openclaw nodes status
   openclaw nodes describe --node <node-id-or-name>
   ```

最新的合格 Mac 会被标记为 `active`。状态输出会显示其距离上次输入
的时长；`describe` 会公开 `active`、`lastActiveAtMs` 和 `presenceUpdatedAtMs`。
活动报告会被有意合并，因此在最近一次报告后发生其他输入时，显示内容可能需要
最多约 15 秒才能更新。

## 活动如何转化为在线状态

macOS 报告器每两秒采样一次 HID 系统空闲时钟。节点连接就绪时，它会报告一次，
此后新发生的物理活动每 15 秒最多报告一次。空闲期间，它每三分钟发送一次
保活消息。空闲时长上限为 30 天，以免非常旧的样本随时间向前漂移并被错误地
认定为最新计算机。

禁用 **Active computer detection** 会停止采样，并通过当前节点连接发送经过身份验证
的清除事件。Gateway 网关会立即移除该 Mac 保留的活动时间戳，并重新计算活动计算机；
其他节点能力和进行中的工作仍保持连接。如果已连接的 Gateway 网关版本早于此清除操作
的引入时间，Mac 节点会重新连接一次，以便通过断开连接清理来移除保留的活动信息。

仅当以下条件全部满足时，Gateway 网关才会接受活动：

- 事件属于该节点 ID 当前经过身份验证的连接；
- 节点具有有效的 `accessibility: true` 权限；
- 载荷包含一个有界整数 `idleSeconds` 值。

Gateway 网关从自身的观测时间中减去 `idleSeconds`，从而得出
`lastActiveAtMs`。它绝不信任节点提供的挂钟时间戳。在已连接的合格 Mac 中，
最新的 `lastActiveAtMs` 胜出；若时间相同，则采用最近更新在线状态的 Mac。

在线状态仅存在于当前进程中，并绑定到连接。断开当前会话、使用相同节点 ID
的其他会话替换当前会话，或撤销 Accessibility 权限，都会清除该节点的活动状态
并重新计算活动 Mac。

## 隐私和模型上下文

活动共享默认关闭，并且与用于 UI 自动化的 Accessibility 授权相互独立。
OpenClaw 发送的是空闲时长，而非输入内容。它不会发送按键值、鼠标坐标、
应用程序名称、窗口标题或原始输入事件。macOS 报告器读取硬件 HID 状态，
因此合成的计算机控制事件不会让自动化 Mac 看起来像是你亲自使用过的计算机。

持续活动不会创建面向模型的系统事件。动态运行时行仅包含经过身份验证的节点 ID：

```text
active_node=<node-id>
```

确切时间戳和由节点控制的显示名称不会进入提示词，以避免提示词注入和缓存抖动。
当智能体需要当前详细信息时，`nodes` 工具可改为读取
`node.list` 或 `node.describe`。

## 连接提醒的路由方式

节点获批后首次成功完成 Gateway 网关握手时，OpenClaw 会等待 750 毫秒，
以便正在连接的 Mac 提交首个活动样本。随后，它会尝试向活动信息最新且支持通知
的已连接 Mac 发送提醒。

- 如果主要投递成功，其他 Mac 不会收到提醒。
- 如果没有可用的活动 Mac，或主要投递失败，OpenClaw 会等待五秒，
  然后尝试所有其他公开 `system.notify` 的已连接 Mac。
- 后续重新连接不会发出提醒。Gateway 网关会将成功连接记录在配对元数据中，
  因此 Gateway 网关重启后，不会为每个先前已连接的节点重新发送提醒。

提醒绑定到经过身份验证的节点身份。相同节点的替代会话会接管其待处理的首次连接
提醒；如果执行投递时该节点已不再连接，则取消提醒。

## 故障排查

| 症状                                      | 检查                                                                                                                                                                      |
| ----------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 没有任何行被标记为 `active`    | 确认已启用活动计算机检测、原生 macOS 节点已连接，并且 `openclaw nodes describe --node <id>` 显示 `permissions.accessibility: true`。                                                                         |
| 错误的 Mac 仍处于活动状态                 | 亲自使用该 Mac，等待合并时间窗口结束，然后重新运行 `openclaw nodes status`。合成的计算机控制操作不计入活动。                                                                    |
| 上次输入数据消失                          | 检查 Mac 是否已断开连接、其节点会话是否已被替换，或 Accessibility 权限是否已撤销。每种情况都会有意清除活动信息。                                                           |
| 提醒出现在多台 Mac 上                     | 主要投递不可用或失败，因此运行了延迟回退。确认活动 Mac 已连接、允许通知并公开 `system.notify`。                                                                         |
| 智能体未提及活动 Mac                      | 活动发生变化后开始新的轮次。运行时提示稳定且精简；使用 `nodes` 工具获取确切的当前元数据。                                                                        |

有关 TCC 恢复，请参阅 [macOS 权限](/zh-CN/platforms/mac/permissions)。有关节点
连接和命令失败，请参阅[节点故障排查](/zh-CN/nodes/troubleshooting)。

## 相关内容

- [节点](/zh-CN/nodes)
- [节点 CLI](/zh-CN/cli/nodes)
- [系统在线状态](/zh-CN/concepts/presence)
- [Gateway 网关协议](/zh-CN/gateway/protocol#presence)
- [macOS 应用](/zh-CN/platforms/macos)
