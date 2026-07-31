---
read_when:
    - memory-wiki CLI を使用する場合
    - '`openclaw wiki` を文書化または変更しています'
summary: '`openclaw wiki` の CLI リファレンス（memory-wiki vault のステータス、検索、コンパイル、lint、適用、ブリッジ、ChatGPT インポート、Obsidian ヘルパー）'
title: ウィキ
x-i18n:
    generated_at: "2026-07-26T10:10:00Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 1f793d52de270068cf3a06b13f52242bb66738235718639486e090a2de213e73
    source_path: cli/wiki.md
    workflow: 16
---

# `openclaw wiki`

`memory-wiki` vault を検査・保守します。バンドルされているオプションの `memory-wiki` plugin によって提供されます。初回使用前に有効化してください。

```bash
openclaw plugins enable memory-wiki
openclaw gateway restart
```

関連項目: [Memory Wiki plugin](/ja-JP/plugins/memory-wiki)、[メモリの概要](/ja-JP/concepts/memory)、[CLI: memory](/ja-JP/cli/memory)

## よく使うコマンド

```bash
openclaw wiki status
openclaw wiki doctor
openclaw wiki init
openclaw wiki ingest ./notes/alpha.md
openclaw wiki okf import ./knowledge-catalog/okf/bundles/ga4
openclaw wiki compile
openclaw wiki lint
openclaw wiki search "alpha"
openclaw wiki search "Teams について誰に尋ねるべきですか？" --mode route-question
openclaw wiki get entity.alpha --from 1 --lines 80

openclaw wiki apply synthesis "Alpha の概要" \
  --body "短い統合本文" \
  --source-id source.alpha

openclaw wiki apply metadata entity.alpha \
  --source-id source.alpha \
  --status review \
  --question "まだ有効ですか？"

openclaw wiki bridge import
openclaw wiki unsafe-local import
openclaw wiki chatgpt import --export ./chatgpt-export --dry-run
openclaw wiki chatgpt rollback <run-id>

openclaw wiki obsidian status
openclaw wiki obsidian search "alpha"
openclaw wiki obsidian open syntheses/alpha-summary.md
openclaw wiki obsidian command workspace:quick-switcher
openclaw wiki obsidian daily
```

## エージェントの選択

`plugins.entries.memory-wiki.config.vault.scope` が `agent` の場合、トップレベルの `--agent <id>` オプションで
vault を選択します。

```bash
openclaw wiki --agent support status
openclaw wiki --agent support search "返金ポリシー"
openclaw wiki --agent marketing ingest ./campaign-notes.md
```

複数のエージェントが設定されている構成では、コマンドが任意のデフォルト vault を読み書きできないようにするため、CLI
操作に `--agent` が必須です。設定されているエージェントが
1 つだけの場合は、そのエージェントが引き続きデフォルトになります。不明なエージェント ID は、
vault 操作が開始される前にエラーになります。`vault.scope` が `global` の場合、このオプションによって選択された
パスは変更されません。

Gateway クライアントにも同じ規則が適用されます。エージェントスコープのマルチエージェント構成では、vault を使用する `wiki.*`
リクエストに `agentId` を渡します。ID がない場合や不明な場合は
エラーになります。エージェントのターン、wiki ツール、メモリコーパスの補足、およびコンパイル済みプロンプトの
ダイジェストには、アクティブなランタイムエージェントのコンテキストがすでに含まれています。

## コマンド

### `wiki status`

vault のモードとスコープ、解決されたエージェント、健全性、および Obsidian CLI の利用可否を表示します。対象の vault が初期化されているか、ブリッジモードが正常か、または Obsidian 連携が利用可能かを確認するには、まずこれを使用します。

ブリッジモードが有効で、メモリアーティファクトを読み取るよう設定されている場合、このコマンドは実行中の Gateway に問い合わせるため、エージェント／ランタイムメモリと同じアクティブなメモリ plugin コンテキストを参照します。

### `wiki doctor`

wiki の健全性チェックを実行し、実行可能な修正方法を報告します。異常がある場合はゼロ以外で終了します。

ブリッジモードが有効で、メモリアーティファクトを読み取るよう設定されている場合、このコマンドはレポートを作成する前に実行中の Gateway に問い合わせます。ブリッジインポートが無効な場合や、メモリアーティファクトを読み取らないブリッジ設定の場合は、ローカル／オフラインのままです。

一般的な問題:

- 公開メモリアーティファクトなしでブリッジモードが有効になっている
- vault レイアウトが無効または存在しない
- Obsidian モードが想定されている場合に外部 Obsidian CLI がない

### `wiki init`

トップレベルのインデックスとキャッシュディレクトリを含む、wiki vault のレイアウトとスターターページを作成します。

### `wiki ingest <path>`

ローカルの Markdown またはテキストファイルを、ソースページとして wiki の `sources/` フォルダーにインポートします。`<path>` はローカルファイルパスである必要があります。現在、URL からの取り込みには対応していません。バイナリファイルは拒否されます。

インポートされたソースページには、出所を示す frontmatter（`sourceType: local-file`、`sourcePath`、`ingestedAt`）が含まれます。取り込み後は常に vault が再コンパイルされます。

フラグ: `--title <title>` はソースタイトルを上書きします（デフォルト: ファイル名から生成）。

### `wiki okf import <path>`

展開済みの Open Knowledge Format バンドルを wiki のコンセプトページにインポートします。

インポーターは、OKF ディレクトリツリー内にある予約済みでないすべての `.md` コンセプトドキュメントを読み取り、空でない `type` フィールドを必須とし、不明な OKF `type` 値を汎用コンセプトとして扱います。予約済みの OKF `index.md` および `log.md` ファイルは、コンセプトとしてインポートされません。

インポートされたページは `concepts/` 配下にフラット化されるため、既存の wiki のコンパイル、検索、取得、ダイジェスト、およびダッシュボードのフローですぐに参照できます。元の OKF コンセプト ID、`type`、`resource`、`tags`、タイムスタンプ、ソースパス、および完全な frontmatter はページの frontmatter に保持されます。OKF 内部の Markdown リンクは生成された wiki ページを指すよう書き換えられ、リンク切れまたは外部リンクは変更されません。インポート後は常に vault が再コンパイルされます。

例:

```bash
openclaw wiki okf import ./bundles/ga4
openclaw wiki okf import ./bundles/ga4 --json
openclaw wiki search "BigQuery テーブル" --mode source-evidence --json
openclaw wiki get <path-from-json-result>
```

### `wiki compile`

インデックス、関連ブロック、ダッシュボード、およびコンパイル済みのクエリ／プロンプトスナップショットを再構築します。スナップショットは OpenClaw の共有 SQLite plugin 状態に永続化され、同期的なプロンプト投影のためにメモリ内に保持されます。vault 内にキャッシュファイルは作成されません。

`render.createDashboards` が有効な場合、コンパイルによってレポートページも更新されます。

### `wiki lint`

vault を lint し、次の内容を含むレポートを書き込みます。

- 構造上の問題（リンク切れ、ID の欠落／重複、ページタイプまたはタイトルの欠落、無効な frontmatter）
- 出所情報の欠落（ソース ID の欠落、インポート元情報の欠落）
- 矛盾（フラグが付いた矛盾、競合する主張）
- 未解決の質問
- 信頼度の低いページと主張
- 古くなったページと主張

wiki に重要な更新を加えた後に実行してください。

### `wiki search <query>`

wiki のコンテンツを検索します。動作は設定によって異なります。

- `search.backend`: `shared` または `local`
- `search.corpus`: `wiki`、`memory`、または `all`
- `--mode`: `auto`、`find-person`、`route-question`、`source-evidence`、または `raw-claim`

wiki 固有のランキングと出所情報には `wiki search` を使用します。広範な共有検索を 1 回行う場合は、アクティブなメモリ plugin が共有検索を公開しているときに `openclaw memory search` を優先します。

検索モード:

- `find-person`: 別名、ハンドル、ソーシャル情報、正規 ID、および人物ページ
- `route-question`: 問い合わせ先／最適な用途のヒントと関係性のコンテキスト
- `source-evidence`: ソースページと構造化された証拠フィールド
- `raw-claim`: 主張／証拠メタデータを含む構造化された主張テキスト

例:

```bash
openclaw wiki search "bgroux" --mode find-person
openclaw wiki search "Teams の展開に詳しいのは誰ですか？" --mode route-question
openclaw wiki search "maintainer-whois" --mode source-evidence
openclaw wiki search "Teams への強いルート" --mode raw-claim --json
```

結果が構造化された主張と一致する場合、テキスト出力には `Claim:` 行と `Evidence:` 行が含まれます。JSON 出力では、エージェント側で詳細を掘り下げるために `matchedClaimId`、`matchedClaimStatus`、`matchedClaimConfidence`、`evidenceKinds`、および `evidenceSourceIds` も公開されます。

### `wiki get <lookup>`

ID または相対パスを指定して wiki ページを読み取ります。

```bash
openclaw wiki get entity.alpha
openclaw wiki get syntheses/alpha-summary.md --from 1 --lines 80
```

### `wiki apply`

ページを自由形式で直接編集せず、限定的な変更を適用します。

- `apply synthesis <title>`: 管理された要約本文を含む統合ページを作成または更新する
- `apply metadata <lookup>`: 既存ページのメタデータを更新する

どちらも `--source-id`、`--contradiction`、`--question`（それぞれ複数回指定可能）、`--confidence <n>`（0～1）、および `--status <status>` を受け付けます。`apply metadata` は、保存済みの信頼度値を削除するための `--clear-confidence` も受け付けます。管理された生成ブロックを維持したまま wiki ページを発展させるには、この方法がサポートされています。

### `wiki bridge import`

アクティブなメモリ plugin から公開メモリアーティファクトを、ブリッジを使用するソースページにインポートします。`bridge` モードでこれを使用し、エクスポートされた最新のメモリアーティファクトを wiki vault に取り込みます。

アクティブなブリッジアーティファクトの読み取りでは、CLI は Gateway RPC 経由でインポートを処理し、ランタイムのメモリ plugin コンテキストを使用します。ブリッジインポートが無効な場合やアーティファクトの読み取りがオフの場合、コマンドはローカル／オフラインでインポート件数ゼロの動作を維持します。インポート後のインデックス更新は `ingest.autoCompile` によって制御されます。

### `wiki unsafe-local import`

`unsafe-local` モードで、明示的に設定されたローカルパス（`unsafeLocal.paths`）からインポートします。意図的に実験的な機能であり、同一マシン内でのみ使用できます。インポート後のインデックス更新は `ingest.autoCompile` によって制御されます。

### `wiki chatgpt import`

ChatGPT のエクスポートを wiki の下書きソースページにインポートします。

```bash
openclaw wiki chatgpt import --export ./chatgpt-export
openclaw wiki chatgpt import --export ./conversations.json --dry-run
```

| フラグ              | デフォルト    | 説明                                                   |
| ----------------- | ---------- | ------------------------------------------------------------- |
| `--export <path>` | （必須） | ChatGPT のエクスポートディレクトリまたは `conversations.json` パス。        |
| `--dry-run`       | `false`    | ページを書き込まずに、作成／更新／スキップ件数をプレビューします。 |

ドライランではないインポートでページが変更されると、ロールバックに必要なインポート実行 ID が記録され、概要に表示されます。

### `wiki chatgpt rollback <run-id>`

以前に適用した ChatGPT インポートの実行をロールバックし、作成されたページを削除して、上書きされたページを復元します。実行がすでにロールバックされている場合は何も行わず、`alreadyRolledBack` を報告します。

### `wiki obsidian ...`

Obsidian に適したモードで動作する vault 用の Obsidian ヘルパーコマンド: `status`、`search`、`open`、`command`、`daily`。`obsidian.useOfficialCli` が有効な場合、これらには `PATH` 上の公式 `obsidian` CLI が必要です。

`vault.scope` が `agent` の場合、
`obsidian.vaultName` はエージェントごとのマッピングではなく単一のグローバル設定であるため、設定検証によって `obsidian.useOfficialCli: true` は拒否されます。
Obsidian に適した Markdown レンダリングは引き続き
利用できます。

## 実用的な使用ガイド

- 出所情報とページ ID が重要な場合は、`wiki search` + `wiki get` を使用します。
- 管理された生成セクションを手動編集する代わりに、`wiki apply` を使用します。
- 矛盾するコンテンツや信頼度の低いコンテンツを信用する前に、`wiki lint` を使用します。
- 一括インポートまたはソース変更後に、最新のダッシュボードとコンパイル済みダイジェストをすぐに利用する場合は、`wiki compile` を使用します。
- データカタログ、ドキュメントのエクスポート、またはエージェント拡充パイプラインがすでに OKF Markdown バンドルを出力する場合は、`wiki okf import` を使用します。
- ブリッジモードが新しくエクスポートされたメモリアーティファクトに依存する場合は、`wiki bridge import` を使用します。

## 関連する設定

`openclaw wiki` の動作は、次の設定によって決まります。

- `plugins.entries.memory-wiki.config.vaultMode`
- `plugins.entries.memory-wiki.config.vault.scope`
- `plugins.entries.memory-wiki.config.vault.path`
- `plugins.entries.memory-wiki.config.search.backend`
- `plugins.entries.memory-wiki.config.search.corpus`
- `plugins.entries.memory-wiki.config.bridge.*`
- `plugins.entries.memory-wiki.config.obsidian.*`
- `plugins.entries.memory-wiki.config.ingest.autoCompile`
- `plugins.entries.memory-wiki.config.render.*`
- `plugins.entries.memory-wiki.config.context.includeCompiledDigestPrompt`

完全な設定モデルについては、[Memory Wiki plugin](/ja-JP/plugins/memory-wiki)を参照してください。

## 関連項目

- [CLI リファレンス](/ja-JP/cli)
- [Memory wiki](/ja-JP/plugins/memory-wiki)
