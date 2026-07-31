---
read_when:
    - 保存されているセッションの一覧と最近のアクティビティを確認する場合
summary: '`openclaw sessions` の CLI リファレンス（保存済みセッションと使用量の一覧）'
title: セッション
x-i18n:
    generated_at: "2026-07-26T08:58:52Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: e00d846229dfad1ada1a8c9a548e26f26247d3f7e5a35106903f6cd4818878b5
    source_path: cli/sessions.md
    workflow: 16
---

# `openclaw sessions`

保存されている会話セッションを一覧表示します。

セッション一覧は、チャンネルやプロバイダーの稼働状況を確認するものではありません。セッションストアに永続化された
会話行を表示します。Discord、Slack、Telegram、または
その他のチャンネルが静かな状態でも、メッセージが処理されるまでは新しいセッション行を
作成せずに正常に再接続できます。チャンネルのライブ
接続状況を確認する必要がある場合は、`openclaw channels status --probe`、
`openclaw status --deep`、または `openclaw health --verbose` を使用してください。

```bash
openclaw sessions
openclaw sessions --agent work
openclaw sessions --all-agents
openclaw sessions --active 120
openclaw sessions --limit 25
openclaw sessions --store ./tmp/sessions.json
openclaw sessions --json
```

フラグ:

| フラグ                 | 説明                                                            |
| -------------------- | ---------------------------------------------------------------------- |
| `--agent <id>`       | 設定済みのエージェントストアを 1 つ指定します（デフォルト: 設定済みのデフォルトエージェント）。        |
| `--all-agents`       | 設定済みのすべてのエージェントストアを集約します。                                 |
| `--store <path>`     | ストアパスを明示的に指定します（`--agent` または `--all-agents` とは併用できません）。 |
| `--active <minutes>` | 過去 N 分以内に更新されたセッションのみを表示します。                  |
| `--limit <n\|all>`   | 出力する最大行数（デフォルトは `100`。`all` で全件出力に戻します）。        |
| `--json`             | 機械可読形式で出力します。                                               |
| `--verbose`          | 詳細ログを出力します。                                                       |

`openclaw sessions` と Gateway の `sessions.list` RPC にはデフォルトで上限があり、
大規模で長期間使用されるストアが CLI プロセスや Gateway のイベント
ループを占有しないようになっています。CLI はデフォルトで最新の 100 セッションを返します。より小さい、または大きい範囲を指定するには `--limit <n>` を、
ストア全体が意図的に必要な場合は `--limit all` を渡します。JSON レスポンスには、呼び出し元がさらに行が存在することを示す必要がある場合に、
`totalCount`、`limitApplied`、および `hasMore` が含まれます。

RPC クライアントは `configuredAgentsOnly: true` を渡すことで、広範な統合
検出ソースを維持しながら、現在設定に存在するエージェントの行のみを返せます。
Control UI はデフォルトでこのモードを使用するため、削除済みまたはディスク上にのみ存在するエージェントストアが
セッションビューに再表示されることはありません。

`--all-agents` は設定済みのエージェントストアを読み取ります。Gateway と ACP のセッション
検出はより広範であり、設定済みのエージェントルートまたはテンプレート化された `session.store` ルートから
解決された SQLite ストアも含まれます。従来のセレクターパスは
エージェントルート内に解決される必要があります。シンボリックリンクおよびルート外のパスは
スキップされます。

`openclaw sessions --all-agents --json`:

```json
{
  "path": null,
  "stores": [
    { "agentId": "main", "path": "/home/user/.openclaw/agents/main/sessions/sessions.json" },
    { "agentId": "work", "path": "/home/user/.openclaw/agents/work/sessions/sessions.json" }
  ],
  "allAgents": true,
  "count": 2,
  "totalCount": 2,
  "limitApplied": 100,
  "hasMore": false,
  "activeMinutes": null,
  "sessions": [
    { "agentId": "main", "key": "agent:main:main", "model": "openai/gpt-5.6-sol" },
    { "agentId": "work", "key": "agent:work:main", "model": "anthropic/claude-sonnet-4-6" }
  ]
}
```

## 末尾の軌跡の進行状況

```bash
openclaw sessions tail
openclaw sessions tail --follow
openclaw sessions tail --session-key "agent:main:telegram:direct:123" --tail 25
openclaw sessions --agent work tail --follow
openclaw sessions --all-agents tail --follow
```

`openclaw sessions tail` は、最近のランタイム軌跡イベントを簡潔な
進行状況行として表示します。`--session-key` を指定しない場合、まず実行中のセッションを追跡し、その後
保存されている最新のセッションを追跡します。`--tail <count>` は、追跡モードの開始前に出力する既存イベント数を
制御します。デフォルトは `80` で、`0` を指定すると現在の末尾から開始します。
`--follow` は、選択した SQLite ベースのセッションまたは明示的に指定した
従来の軌跡ファイルの監視を継続します。

進行状況ビューは意図的に控えめです。プロンプトテキスト、ツール引数、
およびツール結果の本文は出力されません。ツール呼び出しではツール名と
`{...redacted...}` が表示され、ツール結果では `ok`、`error`、`done` などのステータスが表示されます。
モデル完了行には、プロバイダー、モデル、および終了ステータスが表示されます。

## 軌跡バンドルのエクスポート

```bash
openclaw sessions export-trajectory --session-key "agent:main:telegram:direct:123" --workspace .
openclaw sessions export-trajectory --session-key "agent:main:telegram:direct:123" --output bug-123 --json
```

これは、所有者が exec リクエストを承認した後に
`/export-trajectory` スラッシュコマンドが使用するコマンドパスです。出力ディレクトリは常に、
選択したワークスペース配下の `.openclaw/trajectory-exports/` 内に解決されます。

## クリーンアップ保守

次の書き込みサイクルを待たずに、今すぐ保守処理を実行します。

```bash
openclaw sessions cleanup --dry-run
openclaw sessions cleanup --agent work --dry-run
openclaw sessions cleanup --all-agents --dry-run
openclaw sessions cleanup --enforce
openclaw sessions cleanup --enforce --active-key "agent:main:telegram:direct:123"
openclaw sessions cleanup --dry-run --fix-dm-scope
openclaw sessions cleanup --json
```

`openclaw sessions cleanup` は、設定の `session.maintenance` 設定を使用します
（[設定リファレンス](/ja-JP/gateway/config-agents#session)）。

- スコープに関する注記: `openclaw sessions cleanup` はセッションストア、
  トランスクリプト、軌跡行、および従来の軌跡サイドカーを保守します。Cron の実行履歴は
  削除しません。Cron の実行履歴では、ジョブごとに最新の 2000 行が自動的に保持されます
  （[Cron の設定](/ja-JP/automation/cron-jobs#configuration)）。
- クリーンアップでは、参照されていない従来のトランスクリプトアーティファクトやアーカイブ済みトランスクリプトアーティファクト、
  Compaction チェックポイント、および `session.maintenance.pruneAfter` より古い軌跡サイドカーも削除されます。
  SQLite のセッション行から引き続き参照されているアーティファクトは
  保持されます。
- クリーンアップでは、短期間のみ存在する Gateway モデル実行プローブのクリーンアップが
  `modelRunPruned` として個別に報告されます。これは、
  `agent:*:explicit:model-run-<uuid>` の形式を持つ厳密で明示的なキーにのみ一致します。保持期間は固定の `24h` であり、
  負荷に応じて実行されます。つまり、セッションエントリの
  保守または上限の負荷に達した場合にのみ、古いプローブ行を削除します。実行時には、モデル実行のクリーンアップが
  グローバルな古いデータのクリーンアップおよび上限制御より先に行われます。

フラグ:

| フラグ                 | 説明                                                                                                                                                                                                                                                                                                |
| -------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `--dry-run`          | 書き込みを行わずに、削除または上限制御されるエントリ数をプレビューします。テキストモードでは、セッションごとのアクション表（`Action`、`Key`、`Age`、`Model`、`Flags`）と、セッションラベル別にグループ化された概要を出力します。                                                                                                       |
| `--enforce`          | `session.maintenance.mode` が `warn` の場合でも保守処理を適用します。                                                                                                                                                                                                                                          |
| `--fix-missing`      | アーカイブ済みトランスクリプトアーティファクトが存在しない、またはヘッダーのみ／空である従来のエントリを、通常の経過時間や件数の対象外であっても削除します。                                                                                                                                                             |
| `--fix-dm-scope`     | `session.dmScope` が `main` の場合、以前の `per-peer`、`per-channel-peer`、または `per-account-channel-peer` ルーティングによって残された、古いピアキー形式のダイレクト DM 行を廃止します。最初に `--dry-run` を使用してください。適用すると SQLite からそれらの行が削除され、従来のトランスクリプトアーティファクトは削除済みアーカイブとして保持されます。 |
| `--active-key <key>` | 特定のアクティブキーをディスク容量制限による削除から保護します。グループセッションやスレッドスコープのチャットセッションなど、永続的な外部会話ポインターも、経過時間／件数／ディスク容量制限の保守処理によって保持されます。                                                                                               |
| `--agent <id>`       | 設定済みのエージェントストア 1 つに対してクリーンアップを実行します。                                                                                                                                                                                                                                                                |
| `--all-agents`       | 設定済みのすべてのエージェントストアに対してクリーンアップを実行します。                                                                                                                                                                                                                                                               |
| `--store <path>`     | 特定の従来のストアセレクターパスに対して実行します。                                                                                                                                                                                                                                                         |
| `--json`             | JSON 形式の概要を出力します。`--all-agents` を指定した場合、出力にはストアごとの概要が含まれます。                                                                                                                                                                                                                          |

Gateway に到達できる場合、設定済みのエージェントストアに対するドライラン以外のクリーンアップは
Gateway 経由で送信され、ランタイムトラフィックと同じセッションストアライターを
共有します。従来のストアセレクターを明示的にオフライン修復するには、
`--store <path>` を使用してください。

`openclaw sessions cleanup --all-agents --dry-run --json`:

```json
{
  "allAgents": true,
  "mode": "warn",
  "dryRun": true,
  "stores": [
    {
      "agentId": "main",
      "storePath": "/home/user/.openclaw/agents/main/sessions/sessions.json",
      "beforeCount": 120,
      "afterCount": 80,
      "missing": 0,
      "dmScopeRetired": 0,
      "pruned": 40,
      "capped": 0
    },
    {
      "agentId": "work",
      "storePath": "/home/user/.openclaw/agents/work/sessions/sessions.json",
      "beforeCount": 18,
      "afterCount": 18,
      "missing": 0,
      "dmScopeRetired": 0,
      "pruned": 0,
      "capped": 0
    }
  ]
}
```

## セッションの Compaction

停止状態またはサイズ超過のセッションからコンテキスト予算を回収します。`openclaw sessions
compact <key>` は、`sessions.compact`
Gateway RPC の第一級ラッパーであり、実行中の Gateway が必要です。

```bash
openclaw sessions compact "agent:main:main"
openclaw sessions compact "agent:main:main" --max-lines 200
openclaw sessions compact "agent:work:main" --agent work --json
```

- `--max-lines` を指定しない場合、Gateway は LLM を使用してトランスクリプトを要約します。CLI は
  デフォルトではクライアント側の期限を設けません。設定された
  Compaction ライフサイクルは Gateway が管理します。
- `--max-lines <n>` を指定した場合、トランスクリプトを最後の `n` 行までに切り詰め、
  以前のトランスクリプトを `.bak` サイドカーとしてアーカイブします。
- `--agent <id>`: セッションを所有するエージェント。`global` キーでは必須です。
- `--url` / `--token` / `--password`: Gateway 接続のオーバーライド。
- `--timeout <ms>`: クライアント側の任意の RPC タイムアウト（ミリ秒）。
- `--json`: 生の RPC ペイロードを出力します。

Gateway が Compaction の失敗を報告した場合、または到達不能な場合、コマンドは 0 以外で終了するため、Cron やスクリプトが何も行わないサイレントな処理を成功と誤認することはありません。

<Note>
`openclaw agent --message '/compact ...'` は Compaction のパスでは**ありません**。CLI からのスラッシュコマンドは、承認済み送信者チェックによって拒否されます。その呼び出しは何も行わずに終了するのではなく、ここを案内するガイダンスとともに 0 以外で終了します。
</Note>

### sessions.compact RPC

`openclaw gateway call sessions.compact --params '<json>'` は以下を受け付けます。

| フィールド      | 型        | 必須 | 説明                                                |
| ---------- | ----------- | -------- | ---------------------------------------------------------- |
| `key`      | string      | はい      | Compaction するセッションキー（例: `agent:main:main`）。    |
| `agentId`  | string      | いいえ       | セッションを所有するエージェント ID（`global` キー用）。        |
| `maxLines` | integer ≥ 1 | いいえ       | LLM による要約の代わりに、末尾の N 行までに切り詰めます。 |

LLM による要約レスポンスの例:

```json
{
  "ok": true,
  "key": "agent:main:main",
  "compacted": true,
  "result": { "tokensBefore": 243868, "tokensAfter": 34941 }
}
```

切り詰めレスポンスの例（`--max-lines 200`）:

```json
{
  "ok": true,
  "key": "agent:main:main",
  "compacted": true,
  "archived": "/home/user/.openclaw/agents/main/sessions/transcripts/<id>.jsonl.bak",
  "kept": 200
}
```

## 関連項目

- [セッション設定](/ja-JP/gateway/config-agents#session)
- [セッション管理](/ja-JP/concepts/session)
- [Compaction](/ja-JP/concepts/compaction)
- [CLI リファレンス](/ja-JP/cli)
