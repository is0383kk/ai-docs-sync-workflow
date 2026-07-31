---
read_when:
    - ファイル転送 Plugin のインストール、設定、または監査を行っている
summary: 専用の Node コマンドを使用して、ペアリングされた Node 上のファイルを取得、一覧表示、書き込みします。最大 16 MB のバイナリには `node.invoke` 経由で base64 を使用することで、bash の標準出力の切り捨てを回避します。
title: ファイル転送 Plugin
x-i18n:
    generated_at: "2026-07-26T09:35:45Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: f76e92a821be53e988011e2fd9dd53b107b43a8191bf4cdf41baaf918a9c5412
    source_path: plugins/reference/file-transfer.md
    workflow: 16
---

# ファイル転送 Plugin

専用の Node コマンドを介して、ペアリング済み Node 上のファイルを取得、一覧表示、書き込みできます。最大 16 MB のバイナリに対して node.invoke 経由で base64 を使用することで、bash の標準出力の切り捨てを回避します。

## 配布

- パッケージ: `@openclaw/file-transfer`
- インストール経路: OpenClaw に同梱

## サーフェス

コントラクト: `tools`
