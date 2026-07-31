---
read_when:
    - workboard Plugin をインストール、設定、または監査しています
summary: エージェントが担当する課題とセッションのためのダッシュボード型ワークボード。
title: Workboard Plugin
x-i18n:
    generated_at: "2026-07-26T10:13:33Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 4be96893d46c009a127ed3dca5047f8ee4c33fe3c243f8e6867d64976b50b783
    source_path: plugins/reference/workboard.md
    workflow: 16
---

# Workboard Plugin

エージェントが所有する課題とセッションのためのダッシュボード Workboard。

## 配布

- パッケージ: `@openclaw/workboard`
- インストール経路: OpenClaw に同梱

## サーフェス

コントラクト: `tools`; ダッシュボードのデータバインディング: `workboard.cards.list`, `workboard.stats`, `workboard.boards.list`; ダッシュボードのアクション動詞: `workboard.dispatch`

## 関連ドキュメント

- [Workboard](/ja-JP/plugins/workboard)
