---
read_when:
    - 你正在安装、配置或审计文件传输插件
summary: 通过专用节点命令在已配对的节点上获取、列出和写入文件。对不超过 16 MB 的二进制文件，通过 `node.invoke` 使用 base64，从而绕过 bash 标准输出截断限制。
title: 文件传输插件
x-i18n:
    generated_at: "2026-07-26T06:26:23Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: f76e92a821be53e988011e2fd9dd53b107b43a8191bf4cdf41baaf918a9c5412
    source_path: plugins/reference/file-transfer.md
    workflow: 16
---

# 文件传输插件

通过专用节点命令获取、列出文件以及向已配对的节点写入文件。对于最大 16 MB 的二进制文件，通过 node.invoke 使用 base64，从而绕过 bash stdout 截断限制。

## 分发

- 包：`@openclaw/file-transfer`
- 安装方式：OpenClaw 内置

## 接口

契约：`tools`
