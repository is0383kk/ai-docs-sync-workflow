---
read_when:
    - 你正在安装、配置或审计 acpx 插件
summary: 由插件管理会话和传输的 OpenClaw ACP 运行时后端。
title: ACPx 插件
x-i18n:
    generated_at: "2026-07-26T06:25:26Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 9816ca3ada81eb44883b641f3d761b76f894bd83c8aa978c516125c77842f664
    source_path: plugins/reference/acpx.md
    workflow: 16
---

# ACPx 插件

OpenClaw ACP 运行时后端，由插件负责会话和传输管理。

## 分发

- 软件包：`@openclaw/acpx`
- 安装途径：npm；ClawHub

## 接口

Skills

<!-- openclaw-plugin-reference:manual-start -->

## Pi 原生会话

内置运行时会自动检测 Gateway 网关和已配对节点上的 Pi 会话存储。已存储的会话会显示在会话侧边栏的 **Pi** 分组中，并支持以只读方式浏览采用 Pi 所记录 JSONL 会话格式的对话记录。目录会识别项目和全局 `settings.json` 会话目录，以及 `PI_CODING_AGENT_DIR` 和 `PI_CODING_AGENT_SESSION_DIR`。相对路径从包含其 `settings.json` 文件的目录解析。

在 **Config > Plugins > ACPX Runtime** 下关闭 **Pi Session Catalog** 可禁用发现功能。此功能默认启用。

<!-- openclaw-plugin-reference:manual-end -->

## 相关文档

- [acpx](/zh-CN/tools/acp-agents-setup)
