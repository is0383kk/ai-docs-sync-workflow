---
read_when:
    - ターミナルから保存済みのトランスクリプト要約を読みたい場合
    - トランスクリプトの Markdown サマリーへのパスが必要です
    - コアのトランスクリプトストレージ構成をデバッグしています
summary: '`openclaw transcripts` の CLI リファレンス（保存されたトランスクリプトの一覧表示、詳細表示、エクスポート）'
title: トランスクリプト CLI
x-i18n:
    generated_at: "2026-07-26T08:59:46Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: c04ba637fb46ec271383b2f0d17655e18018e07f489c34dc3fd14ad926f27aa4
    source_path: cli/transcripts.md
    workflow: 16
---

# `openclaw transcripts`

永続的な会議トランスクリプトを検査およびエクスポートするためのコマンドです。Google Meet、
Microsoft Teams、Zoom のブラウザ参加者はメモを自動的に取得します。
`transcripts` エージェントツールは、プロバイダーによる取得と手動インポートにも対応しています。

トランスクリプトの正規状態は、`$OPENCLAW_STATE_DIR/state/openclaw.sqlite` にある共有 SQLite データベースに
保存されます。`show` と `path` は、状態ディレクトリの配下に
ユーザー向けの成果物を明示的に生成します。

```text
$OPENCLAW_STATE_DIR/transcripts/YYYY-MM-DD/<session>/
  metadata.json
  transcript.jsonl
  summary.json
  summary.md
```

これらのファイルはエクスポートであり、第 2 のランタイムストアではありません。OpenClaw は、
取得、要約、一覧表示の際にこれらを読み戻しません。デフォルトの状態ディレクトリは
`~/.openclaw` です。`OPENCLAW_STATE_DIR` で上書きできます。日付ディレクトリは
セッションの開始時刻に基づきます。セッションディレクトリは、セッション ID から生成された、
ファイルシステムで安全に使用できるスラッグです。

## コマンド

```bash
openclaw transcripts list
openclaw transcripts show <session>
openclaw transcripts show YYYY-MM-DD/<session>
openclaw transcripts path <session>
openclaw transcripts path YYYY-MM-DD/<session>
openclaw transcripts path <session> --dir
openclaw transcripts path <session> --metadata
openclaw transcripts path <session> --transcript
openclaw transcripts list --json
openclaw transcripts show <session> --json
openclaw transcripts path <session> --json
```

| コマンド                       | 説明                                          |
| ----------------------------- | ---------------------------------------------------- |
| `list`                        | 保存済みセッションを一覧表示します。                                |
| `show <session>`              | `summary.md` を出力して生成します。                  |
| `path <session>`              | `summary.md` のパスを生成して出力します。         |
| `path <session> --dir`        | すべての成果物を生成し、そのディレクトリを出力します。 |
| `path <session> --metadata`   | `metadata.json` を生成して出力します。               |
| `path <session> --transcript` | `transcript.jsonl` を生成して出力します。            |
| `--json`                      | 機械可読形式の出力を表示します（すべてのサブコマンド）。      |

`<session>` には、セッション ID のみ、または日付付きセレクター
（`YYYY-MM-DD/<session>`）を指定できます。同じセッション ID が複数の日に
存在する場合は、`openclaw transcripts show
2026-05-22/standup` のように日付付き形式を使用してください。デフォルトのセッション ID にはタイムスタンプとランダムな
サフィックスが含まれます。固定 ID をセッションに指定するのは、その ID が当日中に一意である場合のみにしてください。

## 出力

`list` は、セッションごとにセレクター、開始時刻、タイトル、
要約パスをタブ区切りの 1 行で出力します。

```text
2026-05-22/standup  2026-05-22T09:00:00.000Z  週次スタンドアップ  /Users/user/.openclaw/transcripts/2026-05-22/standup/summary.md
```

セレクターは、`show` または `path` に再度渡す値として最も安全です。

`list --json` は、`sessionId`、`selector`、`date`、`title`、
`startedAt`、`stoppedAt`、`source`、`path`、`summaryPath`、`hasSummary` を含むオブジェクトを返します。
保存される会議ソース URL にはオリジンとパスのみが含まれます。クエリ文字列、
フラグメント、埋め込まれた認証情報は、永続化前に削除されます。

`show --json` は、保存されたセッションメタデータ、セレクター、セッション
ディレクトリ、要約パス、要約の Markdown テキストを返します。

`path --json` は、選択されたパスと、その成果物を
生成できたかどうかを返します。保存済みセッションには、メタデータとトランスクリプトのエクスポートが常に
存在します。セッションに要約が作成されるまで、要約パスは `exists: false` を報告します。

## 1 日あたり複数のセッション

セッションは日付、次にセッション ID の順でグループ化されます。1 日に 10 件の会議がある場合、
10 個の同階層フォルダーになります。

```text
~/.openclaw/transcripts/2026-05-22/
  transcript-2026-05-22T09-00-00-000Z-a1b2c3d4/
  transcript-2026-05-22T10-30-00-000Z-b2c3d4e5/
  standup/
```

自動化にはデフォルトで生成される ID を使用してください。`standup` のような固定 ID は、
同じ日付に重複しない場合にのみ使用してください。

## 要約がない場合

ライブセッションでは、セッションの停止時に `summary.md` を保存して生成します。
インポートされたトランスクリプトでは、インポート直後に実行します。取得がまだ進行中の場合、プロバイダーが
停止処理中に失敗した場合、または発話が到着する前にメタデータが保存された場合は、要約のないセッションが
`list` に表示されることがあります。

追記専用の未加工トランスクリプトを確認するには `path <session> --transcript` を使用します。
または、`transcripts` ツールの `summarize` アクションを実行して Markdown
要約を再生成します。

## 従来のファイルストアからのアップグレード

SQLite ストアより前の OpenClaw リリースでは、正規のランタイム状態を
`$OPENCLAW_STATE_DIR/transcripts/` の直下に書き込んでいました。次を実行します。

```bash
openclaw doctor --fix
```

Doctor は、従来のツリー全体を SQLite にインポートし、行数と
順序を検証して移行記録を保存し、検証済みのソースツリーをタイムスタンプ付きの
`transcripts.migrated-*` アーカイブに移動します。ランタイムコマンドが
従来のファイルへフォールバックすることはありません。インポートされた
セッションと、依存しているすべてのエクスポートを検証するまで、アーカイブを保持してください。

## 設定

会議トランスクリプトの取得はデフォルトで有効です。グローバルに無効化するには、次のようにします。

```json
{
  "transcripts": {
    "enabled": false
  }
}
```

- `enabled`（デフォルトは `true`）：会議メモ、トランスクリプト
  ツール、および設定済みの自動開始ソースを有効にします。会議
  メモをホストに永続化しない場合は、`false` に設定します。明示的に要求された会議の
  `transcribe` モードでは、既存の上限付きライブキャプション末尾を維持しますが、この設定が false の間は
  永続的な行を書き込みません。
  自動開始ソースは `transcripts.autoStart` で設定します。各エントリは
  存在することで有効になります。そのソースを無効にするには、エントリを省略してください。`discord-voice`
  は同梱の自動開始対応ソースで、`guildId` と
  `channelId` が必要です。

```json
{
  "transcripts": {
    "enabled": true,
    "autoStart": [
      {
        "providerId": "discord-voice",
        "guildId": "1234567890",
        "channelId": "2345678901"
      }
    ]
  }
}
```

会議プロバイダー ID は `google-meet`、`teams`、`zoom` です。それぞれのエイリアスは
`googlemeet`/`meet`、`teams-meetings`/`microsoft-teams`/`msteams`、
`zoom-meetings` です。会議プロバイダーは、すでにアクティブな
会議ボットセッションに接続します。通常の会議参加では、`autoStart` エントリは不要です。
