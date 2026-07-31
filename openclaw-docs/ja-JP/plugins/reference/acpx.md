---
read_when:
    - acpx Plugin のインストール、設定、または監査を行っている場合
summary: Plugin が所有するセッションおよびトランスポート管理を備えた OpenClaw ACP ランタイムバックエンド。
title: ACPx Plugin
x-i18n:
    generated_at: "2026-07-26T09:34:28Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 9816ca3ada81eb44883b641f3d761b76f894bd83c8aa978c516125c77842f664
    source_path: plugins/reference/acpx.md
    workflow: 16
---

# ACPx plugin

Plugin が所有するセッションおよびトランスポート管理を備えた OpenClaw ACP ランタイムバックエンド。

## 配布

- パッケージ: `@openclaw/acpx`
- インストール経路: npm、ClawHub

## 提供機能

Skills

<!-- openclaw-plugin-reference:manual-start -->

## Pi ネイティブセッション

バンドルされたランタイムは、Gateway およびペアリング済みの Node 上にある Pi のセッションストアを自動検出します。保存されたセッションは、セッションサイドバーの **Pi** グループに表示され、Pi で文書化されている JSONL セッション形式のトランスクリプトを読み取り専用で閲覧できます。カタログは、プロジェクトおよびグローバルの `settings.json` セッションディレクトリに加え、`PI_CODING_AGENT_DIR` と `PI_CODING_AGENT_SESSION_DIR` に対応します。相対パスは、その `settings.json` ファイルが格納されているディレクトリを基準に解決されます。

検出を無効にするには、**Config > Plugins > ACPX Runtime** にある **Pi Session Catalog** をオフにします。デフォルトでは有効です。

<!-- openclaw-plugin-reference:manual-end -->

## 関連ドキュメント

- [acpx](/ja-JP/tools/acp-agents-setup)
