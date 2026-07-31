---
read_when:
    - llama-cpp Plugin をインストール、設定、または監査している場合
summary: node-llama-cpp を介したローカル GGUF テキスト推論および埋め込み。
title: Llama Cpp Plugin
x-i18n:
    generated_at: "2026-07-26T09:12:04Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 2756d4b3e00bbe37b4dedec1d54d28bfe6662e8105504317a402293254ce0240
    source_path: plugins/reference/llama-cpp.md
    workflow: 16
---

# Llama Cpp Plugin

node-llama-cpp を介したローカル GGUF テキスト推論および埋め込み。

## 配布

- パッケージ: `@openclaw/llama-cpp-provider`
- インストール経路: npm、ClawHub

## サーフェス

プロバイダー: `llama-cpp`、コントラクト: `embeddingProviders`

<!-- openclaw-plugin-reference:manual-start -->

## デフォルトのテキストモデル

対話形式のセットアップ中、OpenClaw は約 5.0 GB のバンドルダウンロードとして Gemma 4 E4B IT Q4_K_M を提示します。この提示には、合計 RAM が 16 GiB 以上必要です。より小規模なマシンでも、既存のキャッシュ済みモデルは引き続き検出されます。

別のモデルを使用するには、`params.modelPath` を任意のカスタム GGUF に設定します。カスタムモデルには、バンドルダウンロードの RAM 要件は適用されません。要件を下回るマシンでは、Ollama または LM Studio を介してより小さいモデルを実行するか、クラウドプロバイダーを選択することもできます。

<!-- openclaw-plugin-reference:manual-end -->

## 関連ドキュメント

- [llama-cpp](/ja-JP/plugins/llama-cpp)
