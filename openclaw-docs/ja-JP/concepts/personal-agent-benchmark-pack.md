---
read_when:
    - ローカルのパーソナルエージェントの信頼性チェックを実行する
    - リポジトリに基づく QA シナリオカタログの拡張
    - リマインダー、返信、メモリ、墨消し、安全なツール実行の完遂、タスクステータス、安全に共有できる診断情報、証拠に裏付けられた完了報告、障害復旧の検証
summary: プライバシーを保護するパーソナルアシスタントのワークフローチェック用ローカル qa-channel シナリオ。
title: パーソナルエージェントベンチマークパック
x-i18n:
    generated_at: "2026-07-26T09:00:37Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 35da45e4b22b1044a777fa8d6bce87f9ace377950dd0af3f2419b40cfe4d9be6
    source_path: concepts/personal-agent-benchmark-pack.md
    workflow: 16
---

Personal Agent Benchmark Pack は、ローカルのパーソナルアシスタントワークフロー向けの、
リポジトリに基づく小規模な QA シナリオパックです。汎用的なモデルベンチマークではなく、
新しいランナーも必要ありません。プライベート QA スタック（[QA の概要](/ja-JP/concepts/qa-e2e-automation)）、
合成 [QA チャンネル](/ja-JP/channels/qa-channel)、既存の
`qa/scenarios` YAML カタログを再利用します。

## シナリオ

`qa/scenarios/personal/*.yaml` で定義されている 10 個のシナリオ：

| シナリオ ID                                | チェック内容                                                                                       |
| ------------------------------------------ | -------------------------------------------------------------------------------------------- |
| `personal-reminder-roundtrip`              | ローカルの Cron 配信を通じた架空の個人向けリマインダー                                          |
| `personal-channel-thread-reply`            | `qa-channel` を通じた架空の DM とスレッド返信のルーティング                                        |
| `personal-memory-preference-recall`        | 一時的な QA ワークスペースのメモリファイルからの架空の設定の呼び出し                          |
| `personal-redaction-no-secret-leak`        | 架空のシークレットがエコーバックされないことのチェック                                                                   |
| `personal-tool-safety-followthrough`       | 短い承認形式のやり取り後に、安全な読み取りベースのツール操作が最後まで実行されること                        |
| `personal-approval-denial-stop`            | 機密性の高いローカル読み取りリクエストに対する承認拒否時の停止動作                             |
| `personal-task-followthrough-status`       | 保留中、ブロック中、完了を区別する、証拠に基づくタスクステータスの報告            |
| `personal-share-safe-diagnostics-artifact` | 生の個人コンテンツを省略しながら有用なステータスを維持する、安全に共有可能な診断アーティファクト |
| `personal-no-fake-progress`                | ローカルの証拠が存在する前に偽の進捗を示さない、証拠に基づく完了報告         |
| `personal-failure-recovery`                | 部分的なステータスを報告し、再試行の境界を明確に保つ障害復旧                |

機械可読なパックメタデータ（ID リスト、タイトル、説明）は、
`extensions/qa-lab/src/scenario-packs.ts` に `QA_PERSONAL_AGENT_SCENARIO_IDS` として格納されています。
`--pack personal-agent` を使用してパックを実行します：

```bash
OPENCLAW_ENABLE_PRIVATE_QA_CLI=1 pnpm openclaw qa suite \
  --provider-mode mock-openai \
  --pack personal-agent \
  --concurrency 1
```

`--pack` は、繰り返し指定された `--scenario` フラグに追加されます。明示的なシナリオが
最初に実行され、その後、パックのシナリオが `QA_PERSONAL_AGENT_SCENARIO_IDS` の順序で
重複を除いて実行されます。

このパックは、`mock-openai` または別のローカル QA プロバイダー
レーンを使用する `qa-channel` を対象としています。ライブチャットサービスや実際の個人アカウントに接続しないでください。

## プライバシーモデル

シナリオでは、架空のユーザー、架空の設定、架空のシークレット、および
スイートによって作成された一時的な QA Gateway ワークスペースのみを使用します。実際の
OpenClaw ユーザーのメモリ、セッション、認証情報、起動エージェント、グローバル
設定、ライブ Gateway の状態を読み書きしてはなりません。

アーティファクトは既存の QA スイートのアーティファクトディレクトリ内に保持され、
テスト出力として扱われます。秘匿化チェックでは架空のマーカーを使用するため、失敗時も
安全に調査し、Issue に記録できます。

## パックの拡張

`qa/scenarios/personal/` の下に新しい `.yaml` ケースを追加し、その後シナリオ ID を
`QA_PERSONAL_AGENT_SCENARIO_IDS` に追加します。各ケースは小規模かつローカルで、`mock-openai` において決定的にし、
1 つのパーソナルアシスタント動作に焦点を当ててください。

有望な次の候補：秘匿化された軌跡のエクスポートチェック、ローカル限定の
Plugin ワークフローチェック。

シナリオカタログにそのサーフェスを正当化できるだけの安定したケースが蓄積されるまでは、
新しいランナー、Plugin、依存関係、ライブトランスポート、モデル判定機能を追加しないでください。
