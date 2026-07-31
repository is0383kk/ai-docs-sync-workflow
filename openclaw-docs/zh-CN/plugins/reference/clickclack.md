---
read_when:
    - 你正在安装、配置或审计 ClickClack 插件
summary: 新增 ClickClack 渠道界面，用于发送和接收 OpenClaw 消息。
title: ClickClack 插件
x-i18n:
    generated_at: "2026-07-26T06:56:49Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: fcb39341009946dc38a12cc24496e65fd704ed3f2f9aff44bb2dd29fdedaef26
    source_path: plugins/reference/clickclack.md
    workflow: 16
---

# ClickClack 插件

添加 ClickClack 渠道界面，用于发送和接收 OpenClaw 消息。

## 分发

- 软件包：`@openclaw/clickclack`
- 安装方式：npm；ClawHub：`clawhub:@openclaw/clickclack`

## 界面

渠道：`clickclack`；契约：`tools`

<!-- openclaw-plugin-reference:manual-start -->

该插件可以选择为每个 OpenClaw 会话创建一个与生命周期同步的 ClickClack 渠道。托管讨论渠道使用同一智能体的侧会话进行观察和中继，而所附加的主会话会收到一个仅限拉取的 `discussion` 工具。有关配置和会话工具可见性要求，请参阅 [ClickClack 会话讨论](/zh-CN/channels/clickclack#session-discussions)。

<!-- openclaw-plugin-reference:manual-end -->

## 相关文档

- [clickclack](/zh-CN/channels/clickclack)
