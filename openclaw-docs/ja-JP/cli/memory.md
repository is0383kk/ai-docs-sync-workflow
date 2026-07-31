---
read_when:
    - セマンティックメモリのインデックス作成または検索を行いたい場合
    - メモリの可用性やインデックス作成をデバッグしている場合
    - 呼び出された短期記憶を `MEMORY.md` に昇格させる場合。
summary: '`openclaw memory` の CLI リファレンス（status/index/search/promote/promote-explain/rem-harness/rem-backfill）'
title: メモリ
x-i18n:
    generated_at: "2026-07-26T09:30:11Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 6354745f8622ee80345325fa6f3e7d6c5f280cb63b9cdb100a766cf9e300af59
    source_path: cli/memory.md
    workflow: 16
---

# `openclaw memory`

セマンティックメモリのインデックス作成、検索、および `MEMORY.md` への昇格を管理します。
バンドルされている `memory-core` Plugin によって提供され、
`plugins.slots.memory` で `memory-core`（デフォルト）が選択されている場合に利用できます。その他のメモリ
Plugin は、それぞれ独自の CLI 名前空間を公開します。

関連項目: [メモリ](/ja-JP/concepts/memory)の概念、[Dreaming](/ja-JP/concepts/dreaming)、
[メモリ設定リファレンス](/ja-JP/reference/memory-config)、[メモリ Wiki](/ja-JP/plugins/memory-wiki)、
[Wiki](/ja-JP/cli/wiki)、[Plugin](/ja-JP/tools/plugin)。

## `memory status`

```bash
openclaw memory status [--agent <id>] [--deep] [--index] [--fix] [--json] [--verbose]
```

`--agent` を指定しない場合、`agents.entries` 内のすべてのエージェントに対して実行されます。エージェントリストが
設定されていない場合は、デフォルトのエージェントにフォールバックします。

| フラグ        | 効果                                                                                                                                                                                                                                                                                                    |
| ----------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `--deep`    | ベクトルストア、埋め込みプロバイダー、およびセマンティック検索の準備状況を検査します（追加のプロバイダー呼び出しが発生します）。通常の `memory status` は高速性を維持するため、この検査をスキップします。ベクトル／セマンティック状態が不明であることは、検査されていないことを意味します。QMD の字句 `searchMode: "search"` は、`--deep` を指定した場合でも、常にセマンティックベクトル検査をスキップします。 |
| `--index`   | ストアがダーティな場合に再インデックスします。`--deep` も有効になります。                                                                                                                                                                                                                                                          |
| `--fix`     | 古いリコールロックを修復し、昇格メタデータを正規化します。                                                                                                                                                                                                                                               |
| `--json`    | JSON を出力します。                                                                                                                                                                                                                                                                                               |
| `--verbose` | フェーズごとの詳細なログを出力します。                                                                                                                                                                                                                                                                             |

`dreaming.enabled: true` を指定しても `Dreaming` 行が `off` のままである場合、または
スケジュールされたスイープがまったく実行されていないように見える場合、管理対象の Dreaming Cron は、
調整をトリガーするデフォルトエージェントの Heartbeat の発火に依存しています。スケジュールの詳細については、
[Dreaming](/ja-JP/concepts/dreaming)を参照してください。

ステータスには、`memory.search.extraPaths` の追加検索パスも表示されます。

## `memory index`

```bash
openclaw memory index [--agent <id>] [--force] [--verbose]
```

エージェントごとのスコープは `status` と同じです。`--force` は、増分再インデックスではなく
完全な再インデックスを実行します。`--verbose` は、インデックス作成の進行状況を表示する前に、エージェントごとのプロバイダー、モデル、ソース、および
追加パスの詳細を出力します。

## `memory search`

```bash
openclaw memory search [query] [--query <text>] [--agent <id>] [--max-results <n>] [--min-score <n>] [--json]
```

- クエリ: 位置引数 `[query]` または `--query <text>`。両方が設定されている場合は、`--query`
  が優先されます。どちらも設定されていない場合、コマンドはエラーになります。
- `--agent <id>`: デフォルトでは、エージェントリスト全体ではなくデフォルトのエージェントを使用します。
- `--max-results <n>`: 結果数を制限します（正の整数）。
- `--min-score <n>`: このスコア未満の一致を除外します。

## `memory promote`

`memory/YYYY-MM-DD.md` の短期候補をランク付けし、必要に応じて上位のエントリを
`MEMORY.md` に追記します。

```bash
openclaw memory promote [--agent <id>] [--limit <n>] [--min-score <n>] \
  [--min-recall-count <n>] [--min-unique-queries <n>] [--apply] [--include-promoted] [--json]
```

| フラグ                       | デフォルト      | 効果                                                            |
| -------------------------- | ------------ | ----------------------------------------------------------------- |
| `--limit <n>`              |              | 返却／適用する候補の最大数。                                   |
| `--min-score <n>`          | `0.75`       | 重み付き昇格スコアの最小値。                                 |
| `--min-recall-count <n>`   | `3`          | 必要な最小リコール回数。                                    |
| `--min-unique-queries <n>` | `2`          | 必要な異なるクエリの最小数。                            |
| `--apply`                  | プレビューのみ | 選択した候補を `MEMORY.md` に追記し、昇格済みとしてマークします。 |
| `--include-promoted`       |              | 以前のサイクルですでに昇格された候補を含めます。           |
| `--json`                   |              | JSON を出力します。                                                       |

これらの CLI のデフォルトは、スケジュールされた Dreaming スイープの深層フェーズの
しきい値とは異なります（下記の[Dreaming](#dreaming)を参照）。一度限りの手動実行で
スイープの動作に合わせるには、明示的なフラグを渡してください。

ランキングシグナルには、リコール頻度、取得関連性、クエリの多様性、
時間的な新しさ、日をまたぐ統合、および派生概念の豊富さが含まれ、これらは
メモリのリコールと日次取り込みパスの両方から取得されます。さらに、Dreaming で繰り返し再訪された項目には、
ライト／REM フェーズによる強化ブーストがわずかに加わります。書き込み前に、昇格処理は
現在の日次ノートを再読み込みするため、ランキング後に短期スニペットが編集または削除されていても、
古いスナップショットから昇格することなく、その変更が反映されます。

## `memory promote-explain`

1 件の昇格候補について、スコアの内訳を説明します。

```bash
openclaw memory promote-explain <selector> [--agent <id>] [--include-promoted] [--json]
```

`<selector>` は、候補のキー（完全一致または部分文字列）、パス、またはスニペット
テキストと照合します。

## `memory rem-harness`

何も書き込まずに、REM の振り返り、真実の候補、および深層フェーズの昇格出力を
プレビューします。

```bash
openclaw memory rem-harness [--agent <id>] [--path <file-or-dir>] [--grounded] [--include-promoted] [--json]
```

- `--path <file-or-dir>`: 現在のワークスペースではなく、過去の `YYYY-MM-DD.md`
  日次ファイルからハーネスを初期化します。
- `--grounded`: 過去のノートから、根拠に基づく `What Happened`／`Reflections`／
  `Possible Lasting Updates` のプレビューもレンダリングします。

## `memory rem-backfill`

UI で確認できるよう、根拠に基づく過去の REM 要約を `DREAMS.md` に書き込みます。
元に戻せます。

```bash
openclaw memory rem-backfill --path <file-or-dir> [--agent <id>] [--stage-short-term] [--json]
openclaw memory rem-backfill --rollback [--rollback-short-term] [--json]
```

- `--path <file-or-dir>`: `--rollback`／`--rollback-short-term`
  が設定されていない限り必須です。バックフィル元となる過去の日次メモリファイル、またはディレクトリです。
- `--stage-short-term`: 根拠に基づく永続的な候補を現在の
  短期昇格ストアにも初期投入し、通常の深層フェーズでランク付けできるようにします。
- `--rollback`: 以前に書き込まれた、根拠に基づく日記エントリを
  `DREAMS.md` から削除します。
- `--rollback-short-term`: 以前にステージングされた、根拠に基づく短期
  候補を削除します。

## Dreaming

Dreaming は、1 つのスケジュール上で順番に実行される 3 つの協調
フェーズを備えたバックグラウンドのメモリ統合システムです。**ライト**（短期
素材の整理／ステージング）、**REM**（振り返りとテーマの抽出）、**深層**（永続的な
事実を `MEMORY.md` に昇格）。`MEMORY.md` に書き込むのは深層フェーズだけです。

- `plugins.entries.memory-core.config.dreaming.enabled: true` で有効にします
  （デフォルトは `false`）。`memory-core` はスイープの Cron ジョブを自動管理するため、手動で
  `openclaw cron add` を設定する必要はありません。
- チャットから `/dreaming on|off` で切り替え、`/dreaming status`
  （または `/dreaming`／`/dreaming help`）で確認します。`on`／`off` には、チャンネル所有者のステータス
  または Gateway の `operator.admin` が必要です。`status` とヘルプは、コマンドを
  実行できるすべてのユーザーが引き続き利用できます。
- 人が読める形式のフェーズ出力は、`DREAMS.md`（または既存の `dreams.md`）に送られます。
  デフォルト（`dreaming.storage.mode: "separate"`）では、各フェーズは
  個別のレポートも `memory/dreaming/<phase>/YYYY-MM-DD.md` に書き込みます。レポートを日次メモリファイルに統合するには `mode:
"inline"` を、両方に出力するには `"both"`
  を設定します。
- スケジュール実行と手動の `memory promote` 実行では、同じ深層フェーズの
  ランキングシグナルを共有します。異なるのはデフォルトのしきい値だけです（上記の表と
  下記のスケジュール時のデフォルトを参照）。
- スケジュール実行は、設定されたすべてのエージェントのメモリワークスペースに分散して実行されます。

スケジュール時のデフォルト（`plugins.entries.memory-core.config.dreaming`）:

| キー                                    | デフォルト     |
| -------------------------------------- | ----------- |
| `frequency`                            | `0 3 * * *` |
| `phases.deep.minScore`                 | `0.8`       |
| `phases.deep.minRecallCount`           | `3`         |
| `phases.deep.minUniqueQueries`         | `3`         |
| `phases.deep.recencyHalfLifeDays`      | `14`        |
| `phases.deep.maxAgeDays`               | `30`        |
| `phases.deep.maxPromotedSnippetTokens` | `160`       |

```json
{
  "plugins": {
    "entries": {
      "memory-core": {
        "config": {
          "dreaming": {
            "enabled": true
          }
        }
      }
    }
  }
}
```

キーの完全な一覧とフェーズの詳細: [Dreaming](/ja-JP/concepts/dreaming)、
[メモリ設定リファレンス](/ja-JP/reference/memory-config#dreaming)。

## SecretRef の Gateway 依存関係

Active Memory のリモート API キーフィールドが SecretRef として設定されている場合、`memory`
コマンドはアクティブな Gateway スナップショットからそれらを解決します。Gateway が
利用できない場合、コマンドは即座に失敗します。これには、`secrets.resolve`
メソッドをサポートする Gateway が必要です。古い Gateway は不明なメソッドのエラーを返します。

## 関連項目

- [CLI リファレンス](/ja-JP/cli)
- [メモリの概要](/ja-JP/concepts/memory)
