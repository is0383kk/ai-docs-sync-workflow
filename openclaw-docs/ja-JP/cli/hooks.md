---
read_when:
    - エージェントフックを管理したい場合
    - フックが利用可能か確認する、またはワークスペースのフックを有効にする場合
summary: '`openclaw hooks`（エージェントフック）の CLI リファレンス'
title: フック
x-i18n:
    generated_at: "2026-07-26T09:56:19Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: d4d58ea2270cf5122018f7be2943401229929f48f448b15fdd126d1cc99e1e56
    source_path: cli/hooks.md
    workflow: 16
---

# `openclaw hooks`

エージェントフック（`/new`、`/reset`、Gateway の起動などのコマンドに対するイベント駆動型オートメーション）を管理します。単独の `openclaw hooks` は `openclaw hooks list` と同等です。

関連項目: [フック](/ja-JP/automation/hooks) - [Plugin フック](/ja-JP/plugins/hooks)

## フックの一覧表示

```bash
openclaw hooks list [--eligible] [--json] [-v|--verbose]
```

ワークスペース、管理対象、追加、およびバンドル済みの各ディレクトリから検出されたフックを一覧表示します。

- `--eligible`: 要件を満たすフックのみ。
- `--json`: 構造化された出力。
- `-v, --verbose`: 満たされていない要件を示す Missing 列を含めます。

```
フック（準備完了: 4/5）

準備完了:
  🚀 boot-md ✓ - Gateway の起動時に BOOT.md を実行
  📎 bootstrap-extra-files ✓ - エージェントのブートストラップ中に追加のワークスペースブートストラップファイルを挿入
  📝 command-logger ✓ - すべてのコマンドイベントを一元管理された監査ファイルに記録
  💾 session-memory ✓ - /new または /reset コマンドの発行時にセッションコンテキストをメモリに保存
```

## フック情報の取得

```bash
openclaw hooks info <name> [--json]
```

`<name>` はフック名またはフックキーです（例: `session-memory`）。ソース、ファイル/ハンドラーのパス、ホームページ、イベント、および要件ごとの状態（バイナリ、環境変数、設定、OS）を表示します。

## 適格性の確認

```bash
openclaw hooks check [--json]
```

準備完了/未完了の件数概要を出力します。準備が完了していないフックがある場合は、それぞれをブロックしている理由とともに一覧表示します。

## フックの有効化

```bash
openclaw hooks enable <name>
```

設定内の `hooks.internal.entries.<name>.enabled = true` を追加または更新し、`hooks.internal.enabled` マスタースイッチもオンにします（少なくとも 1 つ設定されるまで、Gateway は内部フックハンドラーを読み込みません）。フックが存在しない場合、Plugin によって管理されている場合、または適格でない場合（要件不足）は失敗します。

Plugin によって管理されるフックは `hooks list` に `plugin:<id>` と表示され、ここでは有効化または無効化できません。代わりに、そのフックを所有する Plugin を有効化または無効化してください。

有効化後は、フックを再読み込みするために Gateway を再起動してください（macOS メニューバーアプリの再起動、または開発環境で Gateway プロセスを再起動）。

## フックの無効化

```bash
openclaw hooks disable <name>
```

`hooks.internal.entries.<name>.enabled = false` を設定します。その後、Gateway を再起動してください。

## フックパックのインストールと更新

```bash
openclaw plugins install <package>        # デフォルトでは npm
openclaw plugins install npm:<package>    # npm のみ
openclaw plugins install <package> --pin  # 解決されたバージョンを固定
openclaw plugins install <path>           # ローカルディレクトリまたはアーカイブ
openclaw plugins install -l <path>        # コピーせずローカルディレクトリをリンク

openclaw plugins update <id>
openclaw plugins update --all
openclaw plugins update --dry-run
```

フックパックは統合された Plugin インストーラー/アップデーターを介してインストールされます。`openclaw hooks install` / `openclaw hooks update` は、警告を表示して `plugins` コマンドに転送する非推奨のエイリアスとして引き続き機能します。

- npm の指定はレジストリのみに対応しています。パッケージ名に、オプションで完全一致バージョンまたは dist-tag を加えます。Git/URL/ファイルの指定および semver 範囲は拒否されます。依存関係のインストールは `--ignore-scripts` を使用してプロジェクトローカルで実行されます。
- 単独の指定と `@latest` は安定版トラックを維持します。npm がプレリリース版を解決した場合、OpenClaw は停止し、明示的なオプトイン（`@beta`、`@rc`、またはプレリリース版の完全一致バージョン）を求めます。
- 対応アーカイブ: `.zip`、`.tgz`、`.tar.gz`、`.tar`。
- `-l, --link` は、ローカルディレクトリをコピーせずにリンクします（`hooks.internal.load.extraDirs` に追加）。リンクされたフックパックは、ワークスペースフックではなく、オペレーターが設定したディレクトリから読み込まれる管理対象フックです。
- `--pin` は、npm インストールを正確に解決された `name@version` として共有 SQLite 状態に記録します。
- インストール時にパックを `~/.openclaw/hooks/<id>` にコピーし、そのフックを `hooks.internal.entries.*` で有効化して、インストール元情報を共有 SQLite 状態に記録します。
- 保存された整合性ハッシュが取得したアーティファクトと一致しなくなった場合、OpenClaw は警告を表示し、続行前に確認を求めます。確認を省略するには、グローバルな `--yes` を渡してください（CI で使用する場合など）。

## バンドル済みフック

| フック                  | イベント                                            | 動作                                                                                       |
| --------------------- | ------------------------------------------------- | -------------------------------------------------------------------------------------------------- |
| boot-md               | `gateway:startup`                                 | 設定された各エージェントスコープについて、Gateway の起動時に `BOOT.md` を実行します                                  |
| bootstrap-extra-files | `agent:bootstrap`                                 | エージェントのブートストラップ中に追加のブートストラップファイル（モノレポの `AGENTS.md`/`TOOLS.md` など）を挿入します |
| command-logger        | `command`                                         | コマンドイベントを `~/.openclaw/logs/commands.log` に記録します                                             |
| compaction-notifier   | `session:compact:before`, `session:compact:after` | セッションの Compaction の開始時と完了時に、表示可能なチャット通知を送信します                             |
| session-memory        | `command:new`, `command:reset`                    | `/new` または `/reset` の実行時に、セッションコンテキストをメモリに保存します                                              |

バンドル済みフックは `openclaw hooks enable <hook-name>` で有効化できます。詳細、設定キー、デフォルトについては、[バンドル済みフック](/ja-JP/automation/hooks#bundled-hooks)を参照してください。

### command-logger のログファイル

```bash
tail -n 20 ~/.openclaw/logs/commands.log        # 最近のコマンド
cat ~/.openclaw/logs/commands.log | jq .          # 読みやすく整形して出力
grep '"action":"new"' ~/.openclaw/logs/commands.log | jq .   # アクションでフィルタリング
```

## 注記

- `hooks list --json`、`info --json`、および `check --json` は、構造化された JSON を標準出力に直接書き込みます。

## 関連項目

- [CLI リファレンス](/ja-JP/cli)
- [オートメーションフック](/ja-JP/automation/hooks)
