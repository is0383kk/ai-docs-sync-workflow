---
read_when:
    - 你正在安装、配置或审计 opencode 插件
summary: 为 OpenClaw 添加 OpenCode 模型提供商支持。
title: OpenCode 插件
x-i18n:
    generated_at: "2026-07-26T06:52:41Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: aecf396cfc645e4a036b8130ed7f33db9081dffda120c6d06ebe863dd3be3730
    source_path: plugins/reference/opencode.md
    workflow: 16
---

# OpenCode 插件

为 OpenClaw 添加 OpenCode 模型提供商支持。

## 分发

- 软件包：`@openclaw/opencode-provider`
- 安装方式：OpenClaw 已内置

## 接口

提供商：`opencode`；契约：`mediaUnderstandingProviders`

<!-- openclaw-plugin-reference:manual-start -->

## 原生会话

OpenClaw 会自动检测 Gateway 网关和已配对节点上的 `opencode` CLI。随后，已存储的
会话会显示在会话侧边栏的 **OpenCode** 分组中，并可通过官方
`opencode --pure db ... --format json` 和 `opencode --pure export` 命令以只读方式
浏览转录记录。受限环境和 `--pure` 模式可防止目录浏览加载
项目插件或继承无关的 Gateway 网关凭据。

在 **Config > Plugins > OpenCode** 下关闭 **OpenCode Session Catalog**，
即可禁用发现功能。该功能默认启用。

<!-- openclaw-plugin-reference:manual-end -->

## 相关文档

- [opencode](/zh-CN/providers/opencode)
