---
read_when:
    - 你正在安装、配置或审计 llama-cpp 插件
summary: 通过 node-llama-cpp 在本地进行 GGUF 文本推理和嵌入。
title: Llama Cpp 插件
x-i18n:
    generated_at: "2026-07-26T06:18:03Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 2756d4b3e00bbe37b4dedec1d54d28bfe6662e8105504317a402293254ce0240
    source_path: plugins/reference/llama-cpp.md
    workflow: 16
---

# Llama Cpp 插件

通过 node-llama-cpp 在本地进行 GGUF 文本推理和嵌入。

## 分发

- 软件包：`@openclaw/llama-cpp-provider`
- 安装方式：npm；ClawHub

## 接口

提供商：`llama-cpp`；契约：`embeddingProviders`

<!-- openclaw-plugin-reference:manual-start -->

## 默认文本模型

在交互式设置期间，OpenClaw 会提供 Gemma 4 E4B IT Q4_K_M，作为一个约 5.0 GB 的内置下载项。此下载选项要求至少有 16 GiB 的总内存。现有缓存模型在内存较小的机器上仍会被检测到。

要使用其他模型，请将 `params.modelPath` 设置为任意自定义 GGUF。自定义模型不受内置下载项的内存要求限制。在低于该要求的机器上，也可以通过 Ollama 或 LM Studio 运行更小的模型，或者选择云提供商。

<!-- openclaw-plugin-reference:manual-end -->

## 相关文档

- [llama-cpp](/zh-CN/plugins/llama-cpp)
