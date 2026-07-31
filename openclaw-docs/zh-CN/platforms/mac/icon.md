---
read_when:
    - 更改菜单栏图标行为
summary: OpenClaw 在 macOS 上的菜单栏图标状态和动画
title: 菜单栏图标
x-i18n:
    generated_at: "2026-07-26T06:14:23Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 8a38f1253f0c376ef2ce6c0ae339b67084c472c764964bcc7ad21e10133e2b47
    source_path: platforms/mac/icon.md
    workflow: 16
---

# 菜单栏图标状态

范围：macOS 应用（`apps/macos`）。渲染：`CritterIconRenderer.makeIcon(...)`。动画/状态连接：`CritterStatusLabel` + `CritterStatusLabel+Behavior.swift`。

## 状态

| 状态                  | 触发条件                                  | 视觉效果                                                                                              |
| --------------------- | ----------------------------------------- | --------------------------------------------------------------------------------------------------- |
| 空闲                  | 默认                                      | 正常的眨眼/摆动动画；睁开的眼睛保持光泽亮点                                        |
| 已暂停                | `isPaused=true`                        | 触角下垂（“休班”）且眼睛睁开；无动作                                               |
| 睡眠                  | Gateway 网关断开连接/未配置               | 触角下垂，眼睛闭合成 `⌣ ⌣` 眼睑；无动作                                            |
| 庆祝                  | 消息已发送（`sendCelebrationTick`）          | 眼睛闪现欢快的 `∩ ∩` 弧线约 0.9 秒，同时踢腿                                               |
| 语音唤醒（大耳朵）    | 听到唤醒词                                | 触角竖直挺起并变得更高（`earScale=1.9`）；静默后恢复                          |
| 工作中                | `isWorking=true` 或活动的 `IconState` | 腿部摆动得更快（`legWiggle` 最高至 `1.0`），并有轻微的水平偏移；叠加在空闲摆动之上 |

当会话中有活动的任务或工具时，同一个小动物图标上可叠加显示工具活动徽章（SF Symbol 圆形徽标，例如用于 exec 的 `chevron.left.slash.chevron.right`）。该徽章来自 `IconState`/`ActivityKind`；完整状态模型请参阅[菜单栏](/zh-CN/platforms/mac/menu-bar)。

## 语音唤醒耳朵

- 触发：`AppStateStore.shared.triggerVoiceEars(ttl: nil)`，由语音唤醒捕获流水线（`VoiceWakeRuntime`）以及语音唤醒调试/测试工具（`VoiceWakeTester`、`VoiceWakeOverlayController`）调用。
- 停止：`stopVoiceEars()`，在捕获结束时调用。
- 结束前的静默窗口：通常为 `2.0s`；如果只听到触发词，之后没有其他语音（`VoiceWakeRuntime.silenceWindow` / `triggerOnlySilenceWindow`），则为 `5.0s`。
- 增强期间，空闲状态的眨眼/摆动/腿部/耳朵计时器会暂停（`earBoostActive` 控制 `CritterStatusLabel+Behavior` 中的动画任务）。

## 形状和尺寸

- 画布：18x18pt 模板图像，渲染到 36x36px 位图后备存储（2x），使图标在 Retina 屏幕上保持清晰。
- 耳朵缩放比例默认为 `1.0`；语音增强将其设为 `earScale=1.9`，但不改变整体边框。
- `antennaDroop`（0-1）会将触角向下折叠，形成暂停和睡眠姿态。
- 腿部快速移动使用 `legWiggle`，最高至 `1.0`，并伴随轻微的水平抖动。

## 行为说明

- 没有用于控制耳朵或工作状态的外部 CLI/broker 开关；两者均由应用信号（`AppState.setWorking`、`AppState.triggerVoiceEars`）在内部驱动，以避免意外反复切换。
- 任何新的 TTL 都应保持较短（远低于 10 秒），以便任务挂起时图标能快速恢复到基准状态。

## 相关内容

- [菜单栏](/zh-CN/platforms/mac/menu-bar)
- [macOS 应用](/zh-CN/platforms/macos)
