---
read_when:
    - 调试 Mac 应用健康指示器
summary: macOS 应用如何报告 Gateway 网关/渠道健康状态
title: 健康检查（macOS）
x-i18n:
    generated_at: "2026-07-26T05:52:59Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 095abdbefa7db7c0d14435e2c5db7d1ebc03afa0c539555a7abdd9170d015fb8
    source_path: platforms/mac/health.md
    workflow: 16
---

# macOS 上的健康检查

如何从菜单栏应用中读取已链接渠道的健康状态。

## 菜单栏

状态圆点：

- 绿色：已链接且探测正常。
- 橙色：已链接，但渠道探测报告状态降级/未连接。
- 红色：尚未链接。

第二行显示“已链接 · 身份验证 12 分钟”，或显示失败原因。
菜单中的“Run Health Check Now”会触发按需探测。

## 设置

- General 选项卡显示健康卡片：状态圆点、摘要行（链接状态 +
  身份验证时长）以及可选的失败详情行，并带有 **Retry now** 和
  **Open logs** 按钮。
- **Channels 选项卡**显示 WhatsApp 和 Telegram 的各渠道状态及控制项（登录二维码、
  注销、探测、上次断开连接/错误）。

## 探测的工作原理

应用通过现有的 WebSocket 连接调用 Gateway 网关的 `health` RPC
（而非调用 CLI shell），约每 60 秒调用一次，也可按需调用。RPC 会加载
凭据并报告状态，而不会发送消息。应用分别缓存最后一次
正常快照和最后一次错误，因此 UI 能够立即加载，
并且离线时不会闪烁。

## 如果不确定

使用 [Gateway 健康](/zh-CN/gateway/health) 中的 CLI 流程（`openclaw status`、
`openclaw status --deep`、`openclaw health --json`），并运行
`openclaw logs --follow`，筛选 `web-heartbeat` / `web-reconnect`。

## 相关内容

- [Gateway 健康](/zh-CN/gateway/health)
- [macOS 应用](/zh-CN/platforms/macos)
