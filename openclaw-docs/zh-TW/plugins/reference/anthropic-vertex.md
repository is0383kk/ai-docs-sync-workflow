---
read_when:
    - 你正在安裝、設定或稽核 anthropic-vertex 外掛
summary: 適用於 Google Vertex AI 上 Claude 模型的 OpenClaw Anthropic Vertex 提供者外掛。
title: Anthropic Vertex 外掛
x-i18n:
    generated_at: "2026-07-26T07:26:58Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: bd73b80b4e49a85cd6b1d8e47df6bf8d2d791c36a677124112f299027bfd9af5
    source_path: plugins/reference/anthropic-vertex.md
    workflow: 16
---

# Anthropic Vertex 外掛

適用於 Google Vertex AI 上 Claude 模型的 OpenClaw Anthropic Vertex 供應商外掛。

## 發布

- 套件：`@openclaw/anthropic-vertex-provider`
- 安裝途徑：npm；ClawHub

## 介面

供應商：`anthropic-vertex`

<!-- openclaw-plugin-reference:manual-start -->

## Claude Fable 5

在你的 Google Cloud 區域可使用此模型時，請使用 `anthropic-vertex/claude-fable-5`。
Fable 5 一律使用自適應思考，且預設為 `high` 推理強度。`/think off` 和
`/think minimal` 使用 `low` 推理強度，因為此模型不支援停用思考。

## Claude Sonnet 5

搭配 Vertex 的 `global`、`us` 或 `eu`
端點使用 `anthropic-vertex/claude-sonnet-5`。Sonnet 5 預設以 `high` 推理強度進行自適應思考，並支援
`/think off` 或原生 `/think xhigh|max` 等級。OpenClaw 會自動發布其
1,000,000 個權杖的上下文視窗與 128,000 個權杖的輸出限制。

目錄定價依循 Vertex 的全球優惠費率，每百萬個輸入／輸出權杖為 `$2/$10`，
適用至 2026 年 8 月 31 日；自 9 月 1 日起則為 `$3/$15`。`us` 和 `eu` 多區域端點採用 Vertex 文件所述的
10% 加價。

<!-- openclaw-plugin-reference:manual-end -->
