---
doc-schema-version: 1
read_when:
    - サードパーティ製の OpenClaw Plugin を探したい場合
    - ClawHub で独自の Plugin を公開または掲載したい場合
summary: コミュニティによってメンテナンスされている OpenClaw Plugin を探して公開する
title: コミュニティ Plugin
x-i18n:
    generated_at: "2026-07-26T09:50:54Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 6a9eb477f20da8171a35c22ea6b112d77ff4afe0878f60314c052746aef4e0ac
    source_path: plugins/community.md
    workflow: 16
---

コミュニティ Plugin は、チャンネル、ツール、プロバイダー、フック、その他の機能によって OpenClaw を拡張するサードパーティパッケージです。公開コミュニティ Plugin を見つけるための主要な場所として、[ClawHub](/clawhub) を使用してください。

## Plugin を探す

CLI から ClawHub を検索します。

```bash
openclaw plugins search "calendar"
```

明示的なソースプレフィックスを指定して ClawHub Plugin をインストールします。

```bash
openclaw plugins install clawhub:<package-name>
```

リリース移行期間中も、npm からの直接インストールはサポートされています。

```bash
openclaw plugins install npm:<package-name>
```

一般的なインストール、更新、調査、アンインストールの例については、[Plugin を管理する](/ja-JP/plugins/manage-plugins)を参照してください。コマンドの完全なリファレンスとソース選択ルールについては、[`openclaw plugins`](/ja-JP/cli/plugins)を参照してください。

## Plugin を公開する

OpenClaw ユーザーが見つけてインストールできるように、公開コミュニティ Plugin を ClawHub で公開します。ClawHub は現在のパッケージ一覧、リリース履歴、スキャンステータス、インストールのヒントを管理します。ドキュメントでは、サードパーティ Plugin の静的カタログを管理しません。

```bash
clawhub package publish your-org/your-plugin --dry-run
clawhub package publish your-org/your-plugin
```

公開前に、Plugin にパッケージメタデータ、Plugin マニフェスト、セットアップドキュメント、明確なメンテナンス担当者が含まれていることを確認してください。ClawHub はリリースを作成する前に、所有者のスコープ、パッケージ名、バージョン、ファイル制限、ソースメタデータを検証します。その後、レビューと検証が完了するまで、新しいリリースを通常のインストールおよびダウンロード画面に表示しません。

公開前のチェックリスト：

| 要件                 | 理由                                                |
| -------------------- | --------------------------------------------------- |
| ClawHub で公開済み   | `openclaw plugins install` のヒントを機能させるために必要です |
| 公開 GitHub リポジトリ | ソースレビュー、Issue の追跡、透明性                |
| セットアップと使用方法のドキュメント | ユーザーが設定方法を理解するために必要です |
| 継続的なメンテナンス | 最近の更新または迅速な Issue 対応                   |

完全な公開契約：

- [ClawHub での公開](/ja-JP/clawhub/publishing) - 所有者、スコープ、リリース、
  レビュー、パッケージ検証、パッケージ移管
- [Plugin の構築](/ja-JP/plugins/building-plugins) - Plugin パッケージの構成と
  初回公開のワークフロー
- [Plugin マニフェスト](/ja-JP/plugins/manifest) - ネイティブ Plugin マニフェストのフィールド

## 関連項目

- [Plugin](/ja-JP/tools/plugin) - インストール、設定、再起動、トラブルシューティング
- [Plugin を管理する](/ja-JP/plugins/manage-plugins) - コマンド例
- [ClawHub での公開](/ja-JP/clawhub/publishing) - 公開とリリースのルール
