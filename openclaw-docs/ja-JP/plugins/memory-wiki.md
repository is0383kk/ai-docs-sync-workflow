---
read_when:
    - 単なる MEMORY.md のメモを超える永続的な知識が必要な場合
    - バンドルされている memory-wiki Plugin を設定しています
    - 1 つの Gateway 内のエージェントごとに個別の Wiki 保管庫が必要です
    - wiki_search、wiki_get、またはブリッジモードについて理解したい場合
summary: memory-wiki：出典、主張、ダッシュボード、ブリッジモードを備えたコンパイル済みナレッジ保管庫
title: メモリ Wiki
x-i18n:
    generated_at: "2026-07-26T09:42:15Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: fda3c801ae39b529a3f1fcaf8791b6dcb1d8116ba2e73e99cca62dca6c64140a
    source_path: plugins/memory-wiki.md
    workflow: 16
---

`memory-wiki` は、永続的な知識をナビゲーション可能な wiki にコンパイルする同梱 Plugin です。決定論的なページ、根拠を伴う構造化された主張、来歴、ダッシュボード、機械可読なダイジェストを提供します。

Active Memory Plugin を置き換えるものではありません。想起、昇格、インデックス作成、Dreaming は、設定されたメモリバックエンド
（`memory-core`、QMD、Honcho など）が引き続き所有します。`memory-wiki` はその隣で動作し、知識を保守される wiki レイヤーへコンパイルします。

CLI、ツール、ランタイム統合を使用する前に Plugin を有効にします。

```bash
openclaw plugins enable memory-wiki
openclaw gateway restart
```

| レイヤー                | 所有するもの                                                                              |
| -------------------- | --------------------------------------------------------------------------------- |
| Active Memory Plugin | 想起、セマンティック検索、昇格、Dreaming、メモリランタイム                      |
| `memory-wiki`        | コンパイル済み wiki ページ、来歴情報が豊富な統合結果、ダッシュボード、wiki の検索／取得／適用 |

実用上のルール：

- 設定されているすべてのコーパスを横断して、広範な想起を 1 回実行するには `memory_search`
- wiki 固有のランキング、来歴、またはページレベルの信念構造が必要な場合は `wiki_search` / `wiki_get`
- Active Memory Plugin がコーパス選択をサポートしている場合、1 回の呼び出しで両方のレイヤーを横断するには `memory_search corpus=all`

一般的なローカルファースト構成では、想起用の Active Memory バックエンドとして QMD を使用し、永続的な統合ページ用に `memory-wiki` を `bridge` モードで使用します。[設定](#configuration)にある QMD + ブリッジモードの例を参照してください。

ブリッジモードでエクスポートされたアーティファクトが 0 件と報告される場合、Active Memory Plugin は現在、公開ブリッジ入力を公開していません。まず `openclaw wiki doctor` を実行し、Active Memory Plugin が公開アーティファクトをサポートしていることを確認してください。

## Vault モード

- `isolated`（デフォルト）：独自の Vault とソースを持ち、Active Memory Plugin に依存しません。自己完結型の精選された知識ストアに使用します。
- `bridge`：公開 Plugin SDK の境界を通じて、Active Memory Plugin から公開メモリアーティファクトとイベントログを読み取ります。Plugin の非公開内部へアクセスせずに、Memory Plugin がエクスポートしたアーティファクトをコンパイルする場合に使用します。
- `unsafe-local`：ローカルの非公開パスを使用するための、同一マシン上での明示的な非常手段です。意図的に実験的かつ移植不可能です。信頼境界を理解しており、ブリッジモードでは提供できないローカルファイルシステムへのアクセスが特に必要な場合にのみ使用してください。

Vault モードと Vault スコープは別々に選択します。

- `vaultMode` は、wiki の入力元を選択します。
- `vault.scope` は、すべてのエージェントが 1 つの Vault を使用するか、各エージェントが子 Vault を持つかを選択します。

`vault.scope: "global"` がデフォルトであり、既存の単一 Vault の動作を維持します。エージェント間で wiki ページ、コンパイル済みダイジェスト、検索結果、または書き込みを共有してはならない場合は、`isolated` または `bridge` モードで `vault.scope: "agent"` を使用します。設定された非公開パスはエージェント所有の入力ではないため、エージェントスコープを `unsafe-local` モードと組み合わせることはできません。設定検証では、この組み合わせが拒否されます。

ブリッジモードでは、`bridge.*` 設定トグルに応じて、次の項目をインデックス化できます。

- エクスポートされたメモリアーティファクト（`indexMemoryRoot`）
- 日次ノート（`indexDailyNotes`）
- Dreaming レポート（`indexDreamReports`）
- メモリイベントログ（`followMemoryEvents`）

ブリッジモードが有効で、`bridge.readMemoryArtifacts` が有効になっている場合、`openclaw wiki status`、`openclaw wiki doctor`、`openclaw wiki bridge
import` は実行中の Gateway を経由するため、エージェント／ランタイムメモリと同じ Active Memory Plugin コンテキストを参照します。ブリッジが無効な場合、またはアーティファクトの読み取りがオフの場合、これらのコマンドはローカル／オフライン動作を維持します。

## Vault のレイアウト

```text
<vault>/
  AGENTS.md
  WIKI.md
  index.md
  inbox.md
  entities/
  concepts/
  syntheses/
  sources/
  reports/
  _attachments/
  _views/
  .openclaw-wiki/
```

管理対象のコンテンツは生成ブロック内に保持され、人間が記述したノートブロックは再生成後も維持されます。

- `sources/`：インポートされた未加工素材、およびブリッジ／unsafe-local によって生成されたページ
- `entities/`：永続的な物事、人物、システム、プロジェクト、オブジェクト
- `concepts/`：アイデア、抽象概念、パターン、ポリシー（OKF インポートの格納先でもあります）
- `syntheses/`：コンパイル済みの要約と保守される集約
- `reports/`：生成されたダッシュボード

## Open Knowledge Format のインポート

```bash
openclaw wiki okf import ./bundles/ga4
```

展開済みの Open Knowledge Format バンドルを wiki の概念ページへインポートします。データカタログ、ドキュメントクローラー、またはエンリッチメントエージェントがすでに OKF を生成している場合に適しています。OKF を移植可能な交換アーティファクトとして維持し、`memory-wiki` によって OpenClaw ネイティブの概念ページとコンパイル済みダイジェストへ変換します。

- 予約されていない `.md` ファイルは概念ドキュメントです
- インポートする各概念には、空でない `type` frontmatter フィールドが必要です。`type` がない場合は `missing-type` 警告が生成され、そのファイルはスキップされます
- 不明な `type` 値は、汎用概念として受け入れられます
- `index.md` と `log.md` は予約済みであり、概念としてインポートされることはありません
- 壊れた Markdown リンクまたは外部 Markdown リンクは変更されません

インポートされたページは `concepts/` の直下にフラット化されるため、既存のコンパイル、検索、取得、ダッシュボードのフローから、2 つ目の wiki ツリーを使わずに参照できます。各ページには、元の OKF 概念 ID、ソースパス、`type`、`resource`、`tags`、タイムスタンプ、および生成元の完全な frontmatter が保持されます。内部 OKF リンクは生成された wiki 概念ページへのリンクに書き換えられ、`kind: okf-link` を伴う構造化された `relationships` エントリも生成されます。

## 構造化された主張と根拠

ページには自由形式のテキストだけでなく、構造化された `claims` frontmatter が含まれます。各主張には `id`、`text`、`status`、`confidence`、`evidence[]`、`updatedAt` を含めることができます。各根拠エントリには `kind`、`sourceId`、`path`、`lines`、`weight`、`confidence`、`privacyTier`、`note`、`updatedAt` を含めることができます。

これにより、wiki は受動的なノート置き場ではなく、信念レイヤーとして機能します。主張を追跡、評価、異議申し立てし、ソースまでさかのぼって解決できます。

## エージェント向けエンティティメタデータ

エンティティページには、人物、チーム、システム、プロジェクト、その他任意のエンティティタイプで利用できる汎用ルーティングメタデータが含まれます。

- `entityType`：例：`person`、`team`、`system`、`project`
- `canonicalId`：エイリアスやインポートをまたいで安定した識別キー
- `aliases`：同じページに解決される名前、ハンドル、またはラベル
- `privacyTier`：自由形式の文字列。`public` はレビュー不要として扱われ、それ以外の値（例：`local-private`、`sensitive`、`confirm-before-use`）は `reports/privacy-review.md` でフラグが付けられます
- `bestUsedFor` / `notEnoughFor`：簡潔なルーティングヒント
- `lastRefreshedAt`：ページの編集時刻とは別のソース更新タイムスタンプ
- `personCard`：任意の人物固有ルーティングカード（ハンドル、ソーシャル情報、メールアドレス、タイムゾーン、担当領域、依頼に適した事項、依頼を避けるべき事項、信頼度、プライバシー階層）
- `relationships`：関連ページへの型付きエッジ（対象、種類、重み、信頼度、根拠の種類、プライバシー階層、注記）

人物 wiki では、まず `reports/person-agent-directory.md` を使用し、連絡先情報や推測された事実を使用する前に、`wiki_get` で人物ページを開いてください。

<Accordion title="エンティティページの例">
```yaml
pageType: entity
entityType: person
id: entity.example-person
canonicalId: maintainer.example-person
aliases:
  - Alex
  - example-handle
privacyTier: local-private
bestUsedFor:
  - サンプルエコシステムのルーティング
notEnoughFor:
  - 法的承認
lastRefreshedAt: "2026-04-29T00:00:00.000Z"
personCard:
  handles:
    - "@example-handle"
  socials:
    - "https://x.example/example-handle"
  emails:
    - alex@example.com
  timezone: America/Chicago
  lane: サンプルエコシステム
  askFor:
    - サンプルのロールアウトに関する質問
  avoidAskingFor:
    - 無関係な請求判断
  confidence: 0.8
  privacyTier: confirm-before-use
relationships:
  - targetId: entity.other-person
    targetTitle: その他の人物
    kind: collaborates-with
    confidence: 0.7
    evidenceKind: discrawl-stat
claims:
  - id: claim.example.routing
    text: Alex はサンプルエコシステムのルーティングに役立ちます。
    status: supported
    confidence: 0.9
    evidence:
      - kind: maintainer-whois
        sourceId: source.maintainers
        privacyTier: local-private
```
</Accordion>

## コンパイルパイプライン

コンパイル処理は wiki ページを読み取り、要約を正規化し、機械向けスナップショットを OpenClaw の共有 SQLite Plugin 状態に永続化します。ランタイムコードは、ライフサイクルが所有するオーナースナップショットを使用し、非同期のプロンプト準備中に SQLite を読み込みます。同期的なプロンプト組み立てでは、Markdown のスクレイピングやキャッシュファイルの読み取りを行いません。コンパイル済み出力は、検索／取得用の wiki 初期インデックス作成、主張 ID から所有ページへの逆引き、簡潔なプロンプト補足、レポート生成にも使用されます。

ソースの編集と Vault の復元は、次回のコンパイル後にのみ機械向けの状態になります。Plugin のライフサイクルを再起動または更新すると、Vault の因果的に連鎖したコンパイル公開内容が SQLite と比較され、より新しい状態からロールバックされたスナップショットは拒否されます。ロールバック前に開始したコンパイラーは、復元された先行状態に対して公開できません。プロンプト準備では Vault のポーリングもファイルウォッチャーの導入も行いません。
ロールバック隔離後、実行中のプロセスでコンパイルすると、オーナーが即座にクリアされます。別のコンパイラープロセスを使用する場合は、デーモンが新しい永続的な公開内容を確認できるように、Plugin のライフサイクルを更新する必要があります。
コンパイル済みキャッシュは再構築可能です。公開エポックより前のキャッシュ行はミスとして扱われ、次回のコンパイルで置き換えられます。これらは移行されません。

## ダッシュボードと健全性レポート

`render.createDashboards` が有効な場合、コンパイル処理は `reports/` 配下のダッシュボードを保守します。

| レポート                              | 追跡対象                                             |
| ----------------------------------- | -------------------------------------------------- |
| `reports/open-questions.md`         | 未解決の質問があるページ                    |
| `reports/contradictions.md`         | 矛盾に関する注記のクラスター                        |
| `reports/low-confidence.md`         | 信頼度が低いページと主張                    |
| `reports/claim-health.md`           | 構造化された根拠がない主張                 |
| `reports/stale-pages.md`            | 古い、または不明な鮮度                         |
| `reports/person-agent-directory.md` | 人物／エンティティのルーティングカード                        |
| `reports/relationship-graph.md`     | 構造化された関係エッジ                      |
| `reports/provenance-coverage.md`    | 根拠クラスの網羅状況                            |
| `reports/privacy-review.md`         | 使用前にレビューが必要な非公開プライバシー階層 |

## 検索と取得

検索バックエンドは 2 つあります。

- `shared`：利用可能な場合は共有メモリ検索フローを使用します
- `local`：wiki をローカルで検索します

コーパスは 3 つです：`wiki`、`memory`、`all`。

- `wiki_search` / `wiki_get` は、可能な場合にコンパイル済みダイジェストを初回処理として使用します
- 主張 ID は所有ページに逆引きされます
- 異議のある主張、古い主張、新しい主張はランキングに影響します
- 来歴ラベルは結果にも保持されます

検索モード（`--mode` / ツールの `mode` パラメーター）：

| モード              | 強化対象                                                         |
| ----------------- | -------------------------------------------------------------- |
| `auto`            | バランスの取れたデフォルト                                               |
| `find-person`     | 人物に類するエンティティ、別名、ハンドル、ソーシャル情報、正規 ID |
| `route-question`  | エージェントカード、問い合わせ先／最適な用途のヒント、関係性のコンテキスト |
| `source-evidence` | ソースページと構造化されたエビデンスメタデータ                  |
| `raw-claim`       | 一致する構造化クレーム。クレーム／エビデンスメタデータを返す    |

結果が構造化クレームと一致すると、`wiki_search` は詳細ペイロードで
`matchedClaimId`、`matchedClaimStatus`、`matchedClaimConfidence`、
`evidenceKinds`、`evidenceSourceIds` を返します。利用可能な場合、テキスト出力には
コンパクトな `Claim:` 行と `Evidence:` 行が含まれます。

## エージェントツール

| ツール          | 目的                                                                                                                                                       |
| ------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `wiki_status` | 現在の保管庫モードとスコープ、解決済みエージェント、正常性、Obsidian CLI の利用可否                                                                               |
| `wiki_search` | Wiki ページを検索し、設定されている場合は共有メモリコーパスも検索する。人物検索、質問のルーティング、ソースエビデンス、生のクレームの詳細調査に `mode` を指定可能 |
| `wiki_get`    | ID／パスで Wiki ページを読み取る。共有検索が有効で検索対象が見つからない場合は、共有メモリコーパスにフォールバックする                                     |
| `wiki_apply`  | 自由形式のページ編集を行わず、限定的な統合／メタデータ変更を行う                                                                                             |
| `wiki_lint`   | 構造チェック、来歴の欠落、矛盾、未解決の質問                                                                                            |

また、この Plugin は非排他的なメモリコーパス補完を登録するため、アクティブなメモリ
Plugin がコーパス選択をサポートしている場合、共有の
`memory_search` と `memory_get` から Wiki にアクセスできます。

## プロンプトとコンテキストの動作

`context.includeCompiledDigestPrompt` が有効な場合、メモリプロンプトセクションに
Plugin の状態からコンパイルされたコンパクトなスナップショットが追加されます。対象は上位ページのみ、
上位クレームのみ、矛盾数、質問数、信頼度／鮮度の修飾情報です。
プロンプトの形状が変わるため、これはオプトインです。主に、メモリ補完を明示的に使用する
コンテキストエンジンやプロンプト組み立て処理に影響します。

## 設定

設定は `plugins.entries.memory-wiki.config` の下に配置します。

```json5
{
  plugins: {
    entries: {
      "memory-wiki": {
        enabled: true,
        config: {
          vaultMode: "isolated",
          vault: {
            scope: "global",
            path: "~/.openclaw/wiki/main",
            renderMode: "obsidian",
          },
          obsidian: {
            enabled: true,
            useOfficialCli: true,
            vaultName: "OpenClaw Wiki",
            openAfterWrites: false,
          },
          bridge: {
            enabled: false,
            readMemoryArtifacts: true,
            indexDreamReports: true,
            indexDailyNotes: true,
            indexMemoryRoot: true,
            followMemoryEvents: true,
          },
          unsafeLocal: {
            allowPrivateMemoryCoreAccess: false,
            paths: [],
          },
          ingest: {
            autoCompile: true,
            maxConcurrentJobs: 1,
            allowUrlIngest: true,
          },
          search: {
            backend: "shared",
            corpus: "wiki",
          },
          context: {
            includeCompiledDigestPrompt: false,
          },
          render: {
            preserveHumanBlocks: true,
            createBacklinks: true,
            createDashboards: true,
          },
        },
      },
    },
  },
}
```

主な切り替え項目：

| キー                                        | 値／デフォルト                               | 備考                                                                         |
| ------------------------------------------ | ---------------------------------------------- | ----------------------------------------------------------------------------- |
| `vaultMode`                                | `isolated`（デフォルト）、`bridge`、`unsafe-local` | 入力と統合の動作を選択する                                        |
| `vault.scope`                              | `global`（デフォルト）、`agent`                    | 1 つの共有保管庫、またはエージェントごとに 1 つの子保管庫                                 |
| `vault.path`                               | グローバルでのデフォルトは `~/.openclaw/wiki/main`         | グローバルでは保管庫の正確なパス。エージェントスコープの親はデフォルトで `~/.openclaw/wiki`       |
| `vault.renderMode`                         | `native`（デフォルト）、`obsidian`                 |                                                                               |
| `bridge.readMemoryArtifacts`               | デフォルトは `true`                                 | アクティブなメモリ Plugin の公開アーティファクトをインポートする                                  |
| `bridge.followMemoryEvents`                | デフォルトは `true`                                 | ブリッジモードでイベントログを含める                                             |
| `unsafeLocal.allowPrivateMemoryCoreAccess` | デフォルトは `false`                                | `unsafe-local` のインポートを実行するために必要                                        |
| `unsafeLocal.paths`                        | デフォルトは `[]`                                   | `unsafe-local` モードでインポートするローカルパスを明示的に指定する                         |
| `search.backend`                           | `shared`（デフォルト）、`local`                    |                                                                               |
| `search.corpus`                            | `wiki`（デフォルト）、`memory`、`all`              |                                                                               |
| `context.includeCompiledDigestPrompt`      | デフォルトは `false`                                | 選択したエージェントのコンパクトなダイジェストスナップショットをメモリプロンプトセクションに追加する |
| `render.createBacklinks`                   | デフォルトは `true`                                 | 決定論的な関連ブロックを生成する                                         |
| `render.createDashboards`                  | デフォルトは `true`                                 | ダッシュボードページを生成する                                                      |

### エージェントごとの保管庫

設定済みの各エージェントに個別の Wiki を割り当てるには、`vault.scope` を `agent` に設定します。
このスコープでは、`vault.path` は親ディレクトリとなり、OpenClaw が
正規化されたエージェント ID を追加します。

```json5
{
  agents: {
    list: [{ id: "support" }, { id: "marketing" }],
  },
  plugins: {
    entries: {
      "memory-wiki": {
        enabled: true,
        config: {
          vaultMode: "bridge",
          vault: {
            scope: "agent",
            path: "~/.openclaw/wiki",
          },
          bridge: {
            enabled: true,
            readMemoryArtifacts: true,
          },
        },
      },
    },
  },
}
```

これは `~/.openclaw/wiki/support` と
`~/.openclaw/wiki/marketing` に解決されます。エージェントスコープで `vault.path` を省略すると、
親のデフォルトは `~/.openclaw/wiki` になります。そのため、デフォルトの `main` エージェントは
既存の `~/.openclaw/wiki/main` パスを維持します。

エージェントツール、コンパイル済みプロンプトダイジェスト、および
`memory_search`／`memory_get` を通じて公開される Wiki 補完は、アクティブなエージェントコンテキストから保管庫を解決します。
複数の設定済みエージェントが存在する環境で CLI および Gateway を呼び出す場合は、
`openclaw wiki --agent <agentId> ...` または Gateway
リクエストの `agentId` を使用してエージェントを明示的に指定します。設定済みエージェントが 1 つだけの場合は、
ID が指定されていなくても引き続きそのエージェントがデフォルトになります。

ブリッジモードでは、エージェントスコープのインポートは、その
`agentIds` に選択したエージェントが含まれる場合に限り、公開メモリアーティファクトを受け入れます。別のエージェントが所有するアーティファクト、
所有権メタデータがないアーティファクト、または所有者が不明なアーティファクトはスキップされます。グローバルスコープでは、
既存の共有アーティファクトの動作が維持されます。

<Warning>
`vault.scope` を変更しても、既存の保管庫はコピーも分割もされません。エージェントスコープでは、
明示的に設定した `vault.path` が親ディレクトリになるため、本番環境のエージェントを切り替える前に、
既存のページを意図的に移動またはインポートしてください。最初に
保管庫をバックアップしてください。

エージェントごとの保管庫は、同一プロセス内の知識境界であり、オペレーティングシステムの
セキュリティ境界ではありません。ホストのファイルシステムにアクセスできる Plugin やサンドボックス化されていないツールは、
引き続き別のエージェントのディレクトリを読み取れます。エージェント同士が信頼し合わない場合は、
[サンドボックス化](/ja-JP/gateway/sandboxing)または
[個別の Gateway プロファイル](/ja-JP/gateway/multiple-gateways)を使用してください。
</Warning>

### 例：QMD + ブリッジモード

検索に QMD を使用し、管理された知識レイヤーとして `memory-wiki` を使用する場合に、この構成を使用します。
各レイヤーはそれぞれの役割に集中します。QMD は生のノート、セッションの
エクスポート、追加コレクションを検索可能な状態に保ち、`memory-wiki` は
安定したエンティティ、クレーム、ダッシュボード、ソースページをコンパイルします。

```json5
{
  memory: {
    backend: "qmd",
  },
  plugins: {
    entries: {
      "memory-wiki": {
        enabled: true,
        config: {
          vaultMode: "bridge",
          bridge: {
            enabled: true,
            readMemoryArtifacts: true,
            indexDreamReports: true,
            indexDailyNotes: true,
            indexMemoryRoot: true,
            followMemoryEvents: true,
          },
          search: {
            backend: "shared",
            corpus: "all",
          },
          context: {
            includeCompiledDigestPrompt: false,
          },
        },
      },
    },
  },
}
```

これにより、アクティブメモリの検索は引き続き QMD が担い、`memory-wiki` は
コンパイル済みページとダッシュボードに集中し、コンパイル済みダイジェストプロンプトを
意図的に有効化するまではプロンプトの形状も変わりません。

## CLI

```bash
openclaw wiki status
openclaw wiki doctor
openclaw wiki init
openclaw wiki ingest ./notes/alpha.md
openclaw wiki compile
openclaw wiki lint
openclaw wiki search "alpha"
openclaw wiki get entity.alpha
openclaw wiki apply synthesis "Alpha Summary" --body "..." --source-id source.alpha
openclaw wiki bridge import
openclaw wiki obsidian status
```

`wiki okf import`、`wiki apply metadata`、`wiki unsafe-local import`、
`wiki chatgpt import`／`wiki chatgpt rollback`、および `wiki obsidian` の
全サブコマンドを含む完全なコマンドリファレンスについては、[CLI：Wiki](/ja-JP/cli/wiki)を参照してください。

## Obsidian のサポート

`vault.renderMode` が `obsidian` の場合、Plugin は Obsidian 向けの
Markdown を書き込み、必要に応じて公式 `obsidian` CLI を使用して、ステータスの
確認、保管庫の検索、ページを開く操作、コマンドの呼び出し、デイリーノートへの移動を行えます。
これは任意です。Obsidian がなくても、Wiki はネイティブモードで引き続き動作します。

エージェントスコープの保管庫でも Obsidian 向け Markdown を使用できますが、
設定検証では `obsidian.useOfficialCli: true` と `vault.scope: "agent"` の組み合わせが拒否されます。
現在の `obsidian.vaultName` 設定はグローバルであり、エージェントごとに異なる
Obsidian 保管庫を選択することはできません。代わりに Wiki ツールと CLI 操作を使用するか、
Obsidian で操作する Wiki をグローバルスコープに維持してください。

## 推奨ワークフロー

<Steps>
<Step title="想起には Active Memory Plugin を維持する">
想起、昇格、Dreaming は、設定されたメモリバックエンドが引き続き所有します。
</Step>
<Step title="memory-wiki を有効にする">
ブリッジモードを明示的に使用する場合を除き、`isolated` モードから開始します。
</Step>
<Step title="出典が重要な場合は wiki_search / wiki_get を使用する">
Wiki 固有のランキングまたはページレベルの信念構造が必要な場合は、`memory_search` よりもこれらを優先します。
</Step>
<Step title="範囲を限定した統合またはメタデータの更新には wiki_apply を使用する">
管理対象の生成ブロックを手動で編集しないでください。
</Step>
<Step title="意味のある変更後に wiki_lint を実行する">
矛盾、未解決の疑問、出典の欠落を検出します。
</Step>
<Step title="古い情報や矛盾を可視化するためにダッシュボードを有効にする">
`render.createDashboards: true`（デフォルト）を設定します。
</Step>
</Steps>

## 関連ドキュメント

- [メモリの概要](/ja-JP/concepts/memory)
- [CLI：メモリ](/ja-JP/cli/memory)
- [CLI：Wiki](/ja-JP/cli/wiki)
- [Plugin SDK の概要](/ja-JP/plugins/sdk-overview)
