---
read_when:
    - ターミナルから Workboard カードを確認または作成したい場合
    - CLI から Workboard ワーカーの実行をディスパッチしたい場合
    - Workboard CLI またはスラッシュコマンドの動作をデバッグしている場合
summary: '`openclaw workboard` カード、ディスパッチ、ワーカー実行の CLI リファレンス'
title: ワークボード CLI
x-i18n:
    generated_at: "2026-07-26T09:00:21Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 640260ea6f5959b3aee1cdce76f2501097bff79e9bf1741bdd9ff7a8b43e1a7f
    source_path: cli/workboard.md
    workflow: 16
---

`openclaw workboard` は、同梱の [Workboard Plugin](/ja-JP/plugins/workboard) 用ターミナルインターフェースです。オペレーターは、カードの一覧表示、カードの作成、個別カードの確認、および実行中の Gateway に対する準備済み作業のサブエージェントワーカー実行へのディスパッチ要求を行えます。

コマンドを使用する前に Plugin を有効にします。

```bash
openclaw plugins enable workboard
openclaw gateway restart
```

## 使用方法

```bash
openclaw workboard list [--board <id>] [--status <status>] [--include-archived] [--json]
openclaw workboard create <title...> [--notes <text>] [--status <status>] [--priority <priority>] [--agent <id>] [--board <id>] [--labels <items>] [--json]
openclaw workboard show <id> [--json]
openclaw workboard move <id> --status <status> [--json]
openclaw workboard dispatch [--board <id>] [--max-starts <count>] [--admin] [--url <url>] [--token <token>] [--timeout <ms>] [--json]
```

このコマンドは、ダッシュボードおよび Workboard エージェントツールが使用するものと同じ、Plugin が所有する SQLite データベースを読み書きします。カード ID は UUID です。カード ID を受け取るコマンドでは、一意に特定できる ID のプレフィックスも使用できます（コンパクトなテキスト出力には先頭 8 文字が表示されます）。

有効な `status` の値: `triage`、`backlog`、`todo`、`scheduled`、`ready`、`running`、`review`、`blocked`、`done`。有効な `priority` の値: `low`、`normal`、`high`、`urgent`。

## `list`

```bash
openclaw workboard list
openclaw workboard list --board default --status ready
openclaw workboard list --json
```

テキスト出力はコンパクトです。

```text
7f4a2c10  ready     high    default agent-a  古いワーカーの Heartbeat を修正
```

列は、ID プレフィックス、ステータス、優先度、ボード ID、省略可能なエージェント ID、タイトルの順です。

| フラグ                 | 用途                                       |
| -------------------- | --------------------------------------------- |
| `--board <id>`       | 結果を 1 つのボード名前空間に限定する          |
| `--status <status>`  | 結果を 1 つの Workboard ステータスに限定する         |
| `--include-archived` | コンパクトなテキスト出力にアーカイブ済みカードを含める |
| `--json`             | カード一覧全体をマシン処理用 JSON として出力する      |

CLI を `/workboard list` と一致させるため、コンパクトなテキスト出力ではデフォルトでアーカイブ済みカードが非表示になります。表示するには `--include-archived` を渡します。既存の自動化との互換性を保つため、JSON 出力ではアーカイブ済みカードを含むカード一覧全体が常に維持されます。

## `create`

```bash
openclaw workboard create "古いワーカーの Heartbeat を修正" --priority high --labels bug,workboard
openclaw workboard create "Workboard のドキュメントを作成" --status ready --agent docs-agent --board docs --notes "CLI、スラッシュコマンド、ディスパッチ、SQLite の状態について説明する。"
```

| フラグ                    | 用途                                 |
| ----------------------- | --------------------------------------- |
| `--notes <text>`        | カードの初期メモ                      |
| `--status <status>`     | 初期ステータス。デフォルトは `todo`          |
| `--priority <priority>` | 優先度。デフォルトは `normal`              |
| `--agent <id>`          | カードをエージェントまたは所有者 ID に割り当てる |
| `--board <id>`          | カードをボード名前空間に保存する     |
| `--labels <items>`      | カンマ区切りのラベル                  |
| `--json`                | 作成したカードをマシン処理用 JSON として出力する  |

`create` は Workboard の SQLite 状態に直接書き込みます。カードは Control UI の Workboard タブおよび Workboard ツールにすぐ表示されます。

## `show`

```bash
openclaw workboard show 7f4a2c10
openclaw workboard show 7f4a2c10 --json
```

テキスト出力では、コンパクトなカード行とメモが表示されます。JSON 出力では、実行メタデータ、試行、コメント、リンク、証明、成果物、ワーカーログ、プロトコル状態、診断、自動化メタデータを含む、カードレコード全体が返されます。

JSON 内の証明ステータスは、ワーカーから報告された結果です。`passed` は、添付されたコマンドまたはチェックに対する
ワーカー自身の評価を記録するものであり、独立した検証
結果ではありません。

## `move`

```bash
openclaw workboard move 7f4a2c10 --status review
openclaw workboard move 7f4a2c10 --status done --json
```

`move` は、ダッシュボードでカードをドラッグする場合と同じ手動オペレーター経路を使用して、カードのステータスを変更します。完全なカード ID または一意に特定できるプレフィックスを受け取ります。アクティブな依存関係およびスケジュールによる保留は引き続き適用されます。オペレーターは、エージェントのクレームトークンがなくてもクレーム済みカードを移動できます。クレームトークンは引き続きエージェントツールによる変更のみに限定され、JSON 出力では秘匿されます。

## `dispatch`

```bash
openclaw workboard dispatch
openclaw workboard dispatch --json
openclaw workboard dispatch --max-starts 10
openclaw workboard dispatch --admin
openclaw workboard dispatch --url http://127.0.0.1:18789 --token "$OPENCLAW_GATEWAY_TOKEN"
```

`dispatch` はまず、実行中の Gateway の RPC メソッド `workboard.cards.dispatch` を呼び出します。このメソッドはダッシュボードのディスパッチ操作と同じサブエージェントランタイムを使用するため、準備済みカードはリンクされたセッションキーを持つ、タスク追跡対象のワーカー実行になります。`--max-starts` は追加された `workboard.cards.dispatchWithOptions` メソッドを使用するため、古い Gateway はワーカーを開始する前にこのオプションを拒否します。このフラグを使用する前に、アップグレード後の Gateway を再起動してください。エージェントが割り当てられたカードでは、エージェントスコープのサブエージェントセッションキーが使用されます。未割り当てのカードではスコープなしのサブエージェントキーが維持されるため、Gateway に設定されたデフォルトエージェントが保持されます。

ディスパッチループ:

1. 依存関係の準備が整った子を `ready` に昇格します。
2. 期限切れのクレームまたはタイムアウトしたワーカー実行をブロックします。
3. 準備済みカードにディスパッチメタデータを記録します。
4. クレームされていない準備済みカードから少数のバッチを選択します。
5. 選択した各カードをディスパッチャーまたは割り当て済みエージェントにクレームさせます。
6. 制限されたカードコンテキストとカードのクレームトークンを使用して、サブエージェントワーカー実行を開始します。
7. ワーカー実行 ID、セッションキー、Gateway のタスク台帳から報告された場合はタスクの関連付け、実行ステータス、ワーカーログをカードに保存します。

選択は保守的に行われます。1 回のディスパッチでデフォルトでは最大 3 つのワーカーを開始し、アーカイブ済みまたはすでにクレーム済みのカードをスキップし、1 回の処理で所有者またはエージェントごとに 1 枚のカードのみを開始します。実行中またはレビュー中のアクティブな作業と同じ所有者にすでに割り当てられているカードは、後続のディスパッチに残されます。1 回の処理あたりの上限を変更するには、正の整数を指定して `--max-starts <count>` を渡します。所有者ごとに 1 枚という規則は引き続き適用されるため、実際の開始数はこれより少なくなる場合があります。

カードがクレームされた後にワーカーの開始に失敗した場合、Workboard はそのカードをブロックしてクレームを解除し、カードの実行メタデータとワーカーログメタデータに失敗を記録します。これにより、開始失敗をカードを暗黙的にキューへ戻すことなく表示できます。

明示的な Gateway ターゲットが指定されておらず、ローカル Gateway が利用できないか、Workboard のディスパッチメソッドをまだ公開していない場合、CLI はローカル Workboard 状態に対するデータのみのディスパッチへフォールバックします。データのみのディスパッチでも、依存関係の昇格、古いクレームのクリーンアップ、タイムアウトした実行のブロックは可能ですが、ワーカーは開始されません。認証、権限、検証の失敗、および明示的な `--url` または `--token` ターゲットに対する失敗は、フォールバックを発生させず直接報告されます。

テキスト出力では、ワーカーの開始数が報告されます。

```text
ディスパッチ完了: 開始=2 失敗=0
```

フォールバック出力では明示的に示されます。

```text
Gateway を利用できません。データディスパッチのみ: 昇格=1 ブロック=0
```

JSON 出力にはディスパッチ結果が含まれます。Gateway を使用するディスパッチには `started` と `startFailures` が含まれる場合があります。データのみのフォールバックには `gatewayUnavailable: true` が含まれます。クレームトークンはカードの JSON 出力で秘匿されます。

ダッシュボードでは、同じディスパッチ結果が短い概要として表示されるため、オペレーターはカードの詳細を開かなくても、開始、昇格、ブロック、再クレーム、失敗したカード数を確認できます。

## スラッシュコマンドとの同等性

コマンド対応チャンネルでは、対応するスラッシュコマンドを使用できます。

```text
/workboard list
/workboard show 7f4a2c10
/workboard create 古いワーカーの Heartbeat を修正
/workboard move 7f4a2c10 --status review
/workboard dispatch
```

スラッシュコマンドのディスパッチでも Gateway のサブエージェントランタイムが使用されるため、ダッシュボードおよび CLI の Gateway 経路と同じクレーム、ワーカー開始、失敗時の動作に従います。

`/workboard list` と `/workboard show` は、承認済みのコマンド送信者が使用できる読み取りコマンドです。`/workboard create`、`/workboard move`、`/workboard dispatch` はボードの状態を変更するため、チャットインターフェースでは所有者ステータスが必要であり、Gateway クライアントでは `operator.write` または `operator.admin` が必要です。

## 権限

CLI のディスパッチ経路は通常、Gateway の `operator.write` および `operator.read` スコープを要求します。ワークスペースに紐付けられたカードは、正確に設定されたエージェントワークスペースで直接実行されます。ワークツリー要求は、ホストにリポジトリ管理下のコードを生成させるのではなく、そのディレクトリに限定されます。選択されたワーカーには、その正確なワークスペースに対する書き込み可能かつ非共有の Docker サンドボックスアクセス、要求されたマウントおよびポリシーと一致する稼働中のコンテナハッシュが必要であり、ホストへの脱出機能があってはなりません。`operator.admin` を明示的に要求し、別のホストチェックアウトを許可して、通常の管理対象ワークツリー設定を使用するには `--admin` を渡します。そのスコープがクライアントに対して承認されていない場合、接続は失敗します。読み取り専用の Gateway トークンは読み取りメソッドを通じて Workboard データを確認できますが、カードの作成やワーカーのディスパッチはできません。Workboard の変更権限を持つ呼び出し元による手動のカード移動については、ワークスペースの制限によるその他の変更はありません。

ローカルの `list`、`create`、`show`、`move` コマンドは、現在のプロファイルが使用するローカル OpenClaw 状態ディレクトリを操作します。別の状態ルートが必要な場合は、最上位の `openclaw` コマンドで `--dev` または `--profile <name>` を使用します。

## トラブルシューティング

### カードが表示されない

同じプロファイルおよび状態ルートで Plugin が有効になっていることを確認します。

```bash
openclaw plugins inspect workboard --runtime --json
```

ダッシュボードにはカードが表示されるのに CLI には表示されない場合、両方のコマンドが同じ `--dev` または `--profile` 設定を使用していることを確認してください。

### データのみとディスパッチに表示される

Gateway を起動または再起動します。

```bash
openclaw gateway restart
openclaw gateway status --deep
```

その後、`openclaw workboard dispatch` を再試行します。データのみのフォールバックはローカル状態のクリーンアップに便利ですが、ワーカー実行には稼働中の Gateway が必要です。

### ディスパッチで何も開始されない

アクティブなクレームがない `ready` カードが少なくとも 1 枚あることを確認します。

```bash
openclaw workboard list --status ready
```

同じ所有者に実行中またはレビュー中の作業がすでにある場合も、カードがスキップされることがあります。完了した作業を `done` に移動するか、Workboard ツールで古いクレームを解除するか、アクティブなワーカーの終了後にディスパッチを再実行します。

## 関連項目

- [Workboard Plugin](/ja-JP/plugins/workboard)
- [CLI リファレンス](/ja-JP/cli)
- [スラッシュコマンド](/ja-JP/tools/slash-commands)
- [Control UI](/ja-JP/web/control-ui)
