---
read_when:
    - clickclack Plugin のインストール、設定、または監査を行っている場合
summary: OpenClawメッセージを送受信するためのClickclackチャネルサーフェスを追加します。
title: Clickclack Plugin
x-i18n:
    generated_at: "2026-07-26T10:24:21Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: fcb39341009946dc38a12cc24496e65fd704ed3f2f9aff44bb2dd29fdedaef26
    source_path: plugins/reference/clickclack.md
    workflow: 16
---

# Clickclack Plugin

OpenClaw メッセージを送受信するための Clickclack チャネルサーフェスを追加します。

## 配布

- パッケージ: `@openclaw/clickclack`
- インストール経路: npm、ClawHub: `clawhub:@openclaw/clickclack`

## サーフェス

チャネル: `clickclack`、コントラクト: `tools`

<!-- openclaw-plugin-reference:manual-start -->

Plugin はオプションで、OpenClaw の各セッションに対して、ライフサイクルと同期された ClickClack チャネルを作成できます。管理対象のディスカッションチャネルは、観察とリレーに同一エージェントのサイドセッションを使用し、接続されたメインセッションにはプル専用の `discussion` ツールが提供されます。設定およびセッションツールの可視性要件については、[ClickClack セッションディスカッション](/ja-JP/channels/clickclack#session-discussions)を参照してください。

<!-- openclaw-plugin-reference:manual-end -->

## 関連ドキュメント

- [clickclack](/ja-JP/channels/clickclack)
