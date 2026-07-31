---
read_when:
    - メモリの昇格を自動的に実行したい場合
    - 各 Dreaming フェーズの動作を理解したい場合
    - MEMORY.md を汚さずに統合を調整したい場合
sidebarTitle: Dreaming
summary: 軽度、深度、REM の各フェーズと Dream Diary を備えたバックグラウンドメモリ統合
title: Dreaming
x-i18n:
    generated_at: "2026-07-26T09:00:57Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 501ab42cfdfa0216c308896aa8c1719b06b49d64a62afdb004e097102a376eac
    source_path: concepts/dreaming.md
    workflow: 16
---

Dreaming は、`memory-core` のバックグラウンドメモリ統合システムです。強い短期シグナルを永続的なメモリへ移行しながら、そのプロセスを説明可能かつレビュー可能に保ちます。

<Note>
Dreaming は**オプトイン**であり、デフォルトでは無効です。
</Note>

## Dreaming が書き込む内容

- `memory/.dreams/` の**マシン状態**（リコールストア、フェーズシグナル、取り込みチェックポイント、ロック）。
- `DREAMS.md`（または既存の `dreams.md`）の**人間が読める出力**と、必要に応じて `memory/dreaming/<phase>/YYYY-MM-DD.md` 配下のフェーズレポートファイル。

長期昇格で引き続き書き込まれるのは `MEMORY.md` のみです。

## フェーズモデル

Dreaming は、スイープごとに light -> REM -> deep の3つの協調フェーズを順番に実行します。これらは内部実装フェーズであり、ユーザーが個別に設定するモードではありません。

| フェーズ | 目的                                      | 永続的な書き込み  |
| -------- | ----------------------------------------- | ----------------- |
| Light    | 最近の短期素材を分類してステージングする | なし              |
| REM      | テーマや繰り返し現れるアイデアを振り返る | なし              |
| Deep     | 永続化候補をスコアリングして昇格する     | あり（`MEMORY.md`） |

<AccordionGroup>
  <Accordion title="Light フェーズ">
    - 最近の短期リコール状態、日次メモリファイル、および利用可能な場合は編集済みのセッショントランスクリプトを読み取ります。
    - シグナルを重複排除し、候補行をステージングします。
    - ストレージにインライン出力が含まれる場合、管理対象の `## Light Sleep` ブロックを書き込みます。
    - 後続の deep ランキングで使用する強化シグナルを記録します。
    - `MEMORY.md` には書き込みません。

  </Accordion>
  <Accordion title="REM フェーズ">
    - 最近の短期トレースからテーマと振り返りの要約を作成します。
    - ストレージにインライン出力が含まれる場合、管理対象の `## REM Sleep` ブロックを書き込みます。
    - deep ランキングで使用する REM 強化シグナルを記録します。
    - `MEMORY.md` には書き込みません。

  </Accordion>
  <Accordion title="Deep フェーズ">
    - 重み付きスコアリングとしきい値ゲートで候補をランク付けします（`minScore`、`minRecallCount`、`minUniqueQueries` のすべてを通過する必要があります）。
    - 書き込み前に現在の日次ファイルからスニペットを再取得するため、古くなったスニペットや削除済みのスニペットはスキップされます。
    - 昇格したエントリを `MEMORY.md` に追記します。
    - `## Deep Sleep` の要約を `DREAMS.md` に書き込み、必要に応じて `memory/dreaming/deep/YYYY-MM-DD.md` にも書き込みます。

  </Accordion>
</AccordionGroup>

## セッショントランスクリプトの取り込み

Dreaming は、編集済みのセッショントランスクリプトを Dreaming コーパスに取り込めます。利用可能な場合、トランスクリプトは日次メモリシグナルやリコールトレースとともに light フェーズへ入力されます。個人的な内容や機密性の高い内容は、取り込み前に編集されます。

## Dream Diary

Dreaming は、`DREAMS.md` に物語形式の **Dream Diary** を保持します。各フェーズに十分な素材が集まると、`memory-core` はベストエフォート方式のバックグラウンドサブエージェントターンを実行し、短い日記エントリを追記します。`dreaming.model` が設定されていない場合は、デフォルトのランタイムモデルを使用します。設定されたモデルが利用できない場合、日記の実行はセッションのデフォルトモデルで1回再試行されます。信頼または許可リストのエラーは再試行されず、汎用的な日記エントリへ暗黙的にフォールバックする代わりに、ログ上で確認できる状態のままになります。

<Note>
日記は Dreams UI で人が読むためのものであり、昇格元ではありません。日記およびレポートの成果物は短期昇格の対象外です。`MEMORY.md` に昇格できるのは、根拠のあるメモリスニペットのみです。
</Note>

レビューおよび復旧作業向けに、根拠に基づく履歴バックフィルレーンもあります。

<AccordionGroup>
  <Accordion title="バックフィルコマンド">
    - `memory rem-harness --path ... --grounded` は、過去の `YYYY-MM-DD.md` ノートから根拠に基づく日記出力をプレビューします。
    - `memory rem-backfill --path ...` は、元に戻せる根拠に基づく日記エントリを `DREAMS.md` に書き込みます。
    - `memory rem-backfill --path ... --stage-short-term` は、根拠に基づく永続化候補を、通常の deep フェーズが使用するものと同じ短期エビデンスストアにステージングします。
    - `memory rem-backfill --rollback` と `--rollback-short-term` は、通常の日記エントリや現在の短期リコールに触れることなく、ステージングされたバックフィル成果物を削除します。

  </Accordion>
</AccordionGroup>

Control UI では、エージェントの Memory タブ（Agents ページ）に同じ日記のバックフィル／リセットフローが用意されています。これにより、根拠に基づく候補を昇格させる価値があるか判断する前に、ドリームシーンで結果を確認できます。独立した根拠付き Scene レーンには、過去のリプレイに由来するステージング済み短期エントリと、根拠に基づく情報が主導した昇格項目が表示されます。また、現在の短期状態に触れることなく、根拠に基づくものだけのステージング済みエントリを消去できます。

## Deep ランキングシグナル

Deep ランキングでは、6つの重み付き基本シグナルとフェーズ強化を使用します。

| シグナル       | 重み | 説明                                               |
| -------------- | ---- | -------------------------------------------------- |
| 関連性         | 0.30 | エントリの平均取得品質                             |
| 頻度           | 0.24 | エントリに蓄積された短期シグナルの数               |
| クエリの多様性 | 0.15 | エントリが現れた個別のクエリ／日コンテキスト       |
| 新しさ         | 0.15 | 時間減衰を適用した鮮度スコア                       |
| 統合度         | 0.10 | 複数日にわたる再出現の強度                         |
| 概念的な豊かさ | 0.06 | スニペット／パスに含まれる概念タグの密度           |

Light および REM フェーズでのヒットは、`memory/.dreams/phase-signals.json` から新しさに応じて減衰する小さなブーストを追加します。

シャドウトライアルの結果は、永続的な書き込みを行う前のレビューシグナルとして基本スコアに重ねられます。有用なトライアルは候補に小さな上限付きブーストを与え、中立的なトライアルは候補を保留のままにし、有害なトライアルはそのスコアリング処理で候補を却下済みとしてマークします。このシグナルはレポート専用です。候補の順序やレビューメタデータを変更することはありますが、`MEMORY.md` への書き込みや、単独での候補昇格は行いません。

### QA シャドウトライアルレポートのカバレッジ

QA Lab には、将来の Dreaming シャドウトライアルが昇格前に候補メモリをレビューする方法を検討するための、レポート専用シナリオが含まれています。エージェントはベースラインの回答と候補メモリを使用可能な回答を比較し、判定、理由、リスクフラグを含むローカルレポートを書き込みます。このカバレッジは QA の範囲に限定されます。レポート成果物が `MEMORY.md` から分離されたままであることと、エージェントが候補を昇格済みだと主張しないことを検証します。本番環境のシャドウトライアル動作を追加したり、deep フェーズの昇格エンジンを変更したりするものではありません。

`memory-core` シャドウトライアルランナーは、安定した成果物を必要とするコードパス向けに、同じレポート専用契約を維持します。候補、トライアルプロンプト、ベースライン結果、候補使用時の結果、判定、理由、リスクフラグ、エビデンス参照を受け取り、`promotion action: report-only` を使用してレポートを書き込みます。有用という判定は `promote` の推奨に、中立という判定は `defer` に、有害という判定は `reject` にマッピングされます。いずれも `MEMORY.md` への書き込みや deep フェーズの昇格適用は行いません。

## スケジュール

有効にすると、`memory-core` は Dreaming の完全なスイープ用に1つの Cron ジョブを自動管理します。プライマリランタイムワークスペースと設定済みのすべてのエージェントワークスペース間で重複排除されるため、サブエージェントのワークスペース展開によってメインエージェントの `DREAMS.md` とメモリ状態が除外されることはありません。

| 設定                 | デフォルト             |
| -------------------- | ---------------------- |
| `dreaming.frequency` | `0 3 * * *`   |
| `dreaming.model`     | デフォルトモデル       |

## クイックスタート

<Tabs>
  <Tab title="Dreaming を有効にする">
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
  </Tab>
  <Tab title="カスタムスイープ間隔">
    ```json
    {
      "plugins": {
        "entries": {
          "memory-core": {
            "config": {
              "dreaming": {
                "enabled": true,
                "timezone": "America/Los_Angeles",
                "frequency": "0 */6 * * *"
              }
            }
          }
        }
      }
    }
    ```
  </Tab>
</Tabs>

## スラッシュコマンド

```text
/dreaming status
/dreaming on
/dreaming off
/dreaming help
```

`/dreaming on` と `/dreaming off` では、チャネルからの呼び出し元には所有者ステータスが、Gateway クライアントには `operator.admin` が必要です。`/dreaming status` と `/dreaming help` は読み取り専用です。

## CLI ワークフロー

<Tabs>
  <Tab title="昇格のプレビュー／適用">
    ```bash
    openclaw memory promote
    openclaw memory promote --apply
    openclaw memory promote --limit 5
    openclaw memory status --deep
    ```

    手動の `memory promote` では、CLI フラグで上書きしない限り、デフォルトで deep フェーズのしきい値が使用されます。

  </Tab>
  <Tab title="昇格の説明">
    特定の候補が昇格する、または昇格しない理由を説明します。

    ```bash
    openclaw memory promote-explain "router vlan"
    openclaw memory promote-explain "router vlan" --json
    ```

  </Tab>
  <Tab title="REM ハーネスのプレビュー">
    何も書き込まずに、REM の振り返り、候補となる事実、deep 昇格の出力をプレビューします。

    ```bash
    openclaw memory rem-harness
    openclaw memory rem-harness --json
    ```

  </Tab>
</Tabs>

## 主要なデフォルト

すべての設定は `plugins.entries.memory-core.config.dreaming` 配下にあります。

<ParamField path="enabled" type="boolean" default="false">
  Dreaming スイープを有効または無効にします。
</ParamField>
<ParamField path="frequency" type="string" default="0 3 * * *">
  Dreaming の完全なスイープを実行する Cron 間隔です。
</ParamField>
<ParamField path="model" type="string">
  Dream Diary サブエージェントモデルのオプションの上書きです。サブエージェントの `allowedModels` 許可リストも設定する場合は、正規の `provider/model` 値を使用します。
</ParamField>
<ParamField path="phases.deep.maxPromotedSnippetTokens" type="number" default="160">
  `MEMORY.md` に昇格する各短期リコールスニペットから保持される推定トークン数の上限です。ランキングの出所情報は引き続き確認できます。
</ParamField>

<Warning>
`dreaming.model` には `plugins.entries.memory-core.subagent.allowModelOverride: true` が必要です。制限するには、`plugins.entries.memory-core.subagent.allowedModels` も設定します。自動再試行の対象はモデルを利用できないエラーのみです。信頼または許可リストのエラーは、暗黙的にフォールバックする代わりにログ上で確認できる状態のままになります。
</Warning>

<Note>
フェーズポリシー、しきい値、ストレージ動作の大部分は内部実装の詳細です。すべてのキーの一覧については、[メモリ設定リファレンス](/ja-JP/reference/memory-config#dreaming)を参照してください。
</Note>

## Dreams UI

有効にすると、Gateway の **Dreams** タブには次の内容が表示されます。

- 現在の Dreaming 有効状態
- フェーズ単位のステータスと管理対象スイープの有無
- 短期、根拠付き、シグナル、本日昇格済みの各件数
- 次回のスケジュール実行時刻
- ステージングされた履歴リプレイエントリ用の独立した根拠付き Scene レーン
- `doctor.memory.dreamDiary` をデータソースとする展開可能な Dream Diary リーダー

## 関連項目

- [メモリ](/ja-JP/concepts/memory)
- [メモリ CLI](/ja-JP/cli/memory)
- [メモリ設定リファレンス](/ja-JP/reference/memory-config)
- [メモリ検索](/ja-JP/concepts/memory-search)
