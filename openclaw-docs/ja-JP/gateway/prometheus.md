---
read_when:
    - Prometheus、Grafana、VictoriaMetrics、または別のスクレイパーで OpenClaw Gateway のメトリクスを収集する場合
    - ダッシュボードやアラート向けに、Prometheus のメトリクス名とラベルポリシーが必要な場合
    - OpenTelemetry コレクターを実行せずにメトリクスを取得したい場合
sidebarTitle: Prometheus
summary: diagnostics-prometheus Plugin を通じて、OpenClaw の診断情報を Prometheus テキストメトリクスとして公開する
title: Prometheus メトリクス
x-i18n:
    generated_at: "2026-07-26T09:42:33Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 9d04a46bdb401df3cdd2571b973f2a60f264862cf74da02c5a9cfa1de6ea9ffe
    source_path: gateway/prometheus.md
    workflow: 16
---

OpenClaw は、公式の
`diagnostics-prometheus` Plugin を通じて診断メトリクスを公開できます。この Plugin は、信頼済みの診断情報に加え、
内部でタグ付けされたディスパッチャー所有の診断イベント（キュー、メモリ、
セッション復旧シグナル）をリッスンし、次の場所で Prometheus テキストエンドポイントを提供します。

```text
GET /api/diagnostics/prometheus
```

コンテンツタイプは、標準の
Prometheus 公開形式である `text/plain; version=0.0.4; charset=utf-8` です。

<Warning>
このルートは Gateway 認証（オペレータースコープ、信頼済みオペレーター向けサーフェス）を使用します。認証のない公開 `/metrics` エンドポイントとして公開しないでください。他のオペレーター API に使用するものと同じ認証パスを通じてスクレイピングしてください。
</Warning>

トレース、ログ、OTLP プッシュ、OpenTelemetry GenAI セマンティック属性については、[OpenTelemetry エクスポート](/ja-JP/gateway/opentelemetry)を参照してください。

## クイックスタート

<Steps>
  <Step title="Plugin をインストール">
    ```bash
    openclaw plugins install clawhub:@openclaw/diagnostics-prometheus
    ```
  </Step>
  <Step title="Plugin を有効化">
    <Tabs>
      <Tab title="設定">
        ```json5
        {
          plugins: {
            allow: ["diagnostics-prometheus"],
            entries: {
              "diagnostics-prometheus": { enabled: true },
            },
          },
          diagnostics: {
            enabled: true,
          },
        }
        ```
      </Tab>
      <Tab title="CLI">
        ```bash
        openclaw plugins enable diagnostics-prometheus
        ```
      </Tab>
    </Tabs>
  </Step>
  <Step title="Gateway を再起動">
    HTTP ルートは Plugin の起動時に登録されるため、有効化後に再読み込みしてください。
  </Step>
  <Step title="保護されたルートをスクレイピング">
    オペレータークライアントが使用するものと同じ Gateway 認証を送信します。

    ```bash
    curl -H "Authorization: Bearer $OPENCLAW_GATEWAY_TOKEN" \
      http://127.0.0.1:18789/api/diagnostics/prometheus
    ```

  </Step>
  <Step title="Prometheus を接続">
    ```yaml
    # prometheus.yml
    scrape_configs:
      - job_name: openclaw
        scrape_interval: 30s
        metrics_path: /api/diagnostics/prometheus
        authorization:
          credentials_file: /etc/prometheus/openclaw-gateway-token
        static_configs:
          - targets: ["openclaw-gateway:18789"]
    ```
  </Step>
</Steps>

<Note>
`diagnostics.enabled` のデフォルトは `true` です。厳密に制約された環境でのみ `false` に設定してください。`false` の場合でも、Plugin は HTTP ルートを登録しますが、診断イベントはエクスポーターに流れないため、レスポンスは空になります。
</Note>

## エクスポートされるメトリクス

| メトリクス                                           | タイプ      | ラベル                                                                                    |
| ------------------------------------------------ | --------- | ----------------------------------------------------------------------------------------- |
| `openclaw_run_completed_total`                   | カウンター   | `channel`, `model`, `outcome`, `provider`, `trigger`                                      |
| `openclaw_run_duration_seconds`                  | ヒストグラム | `channel`, `model`, `outcome`, `provider`, `trigger`                                      |
| `openclaw_model_call_total`                      | カウンター   | `api`, `error_category`, `model`, `observation_unit`, `outcome`, `provider`, `transport`  |
| `openclaw_model_call_duration_seconds`           | ヒストグラム | `api`, `error_category`, `model`, `observation_unit`, `outcome`, `provider`, `transport`  |
| `openclaw_model_failover_total`                  | カウンター   | `from_model`, `from_provider`, `lane`, `reason`, `suspended`, `to_model`, `to_provider`   |
| `openclaw_model_tokens_total`                    | カウンター   | `agent`, `channel`, `model`, `provider`, `token_type`                                     |
| `openclaw_gen_ai_client_token_usage`             | ヒストグラム | `model`, `provider`, `token_type`                                                         |
| `openclaw_model_cost_usd_total`                  | カウンター   | `agent`, `channel`, `model`, `provider`                                                   |
| `openclaw_model_usage_duration_seconds`          | ヒストグラム | `agent`, `channel`, `model`, `provider`                                                   |
| `openclaw_skill_used_total`                      | カウンター   | `activation`, `agent`, `skill`, `source`                                                  |
| `openclaw_tool_execution_total`                  | カウンター   | `error_category`, `outcome`, `params_kind`, `tool`, `tool_owner`, `tool_source`           |
| `openclaw_tool_execution_duration_seconds`       | ヒストグラム | `error_category`, `outcome`, `params_kind`, `tool`, `tool_owner`, `tool_source`           |
| `openclaw_tool_execution_blocked_total`          | カウンター   | `denied_reason`, `params_kind`, `tool`, `tool_owner`, `tool_source`                       |
| `openclaw_harness_run_total`                     | カウンター   | `channel`, `error_category`, `harness`, `model`, `outcome`, `phase`, `plugin`, `provider` |
| `openclaw_harness_run_duration_seconds`          | ヒストグラム | `channel`, `error_category`, `harness`, `model`, `outcome`, `phase`, `plugin`, `provider` |
| `openclaw_webhook_received_total`                | カウンター   | `channel`, `webhook`                                                                      |
| `openclaw_webhook_error_total`                   | カウンター   | `channel`, `webhook`                                                                      |
| `openclaw_webhook_duration_seconds`              | ヒストグラム | `channel`, `webhook`                                                                      |
| `openclaw_message_received_total`                | カウンター   | `channel`, `source`                                                                       |
| `openclaw_message_dispatch_started_total`        | カウンター   | `channel`, `source`                                                                       |
| `openclaw_message_dispatch_completed_total`      | カウンター   | `channel`, `outcome`, `reason`, `source`                                                  |
| `openclaw_message_dispatch_duration_seconds`     | ヒストグラム | `channel`, `outcome`, `reason`, `source`                                                  |
| `openclaw_message_processed_total`               | カウンター   | `channel`, `outcome`, `reason`                                                            |
| `openclaw_message_processed_duration_seconds`    | ヒストグラム | `channel`, `outcome`, `reason`                                                            |
| `openclaw_message_delivery_started_total`        | カウンター   | `channel`, `delivery_kind`                                                                |
| `openclaw_message_delivery_total`                | カウンター   | `channel`, `delivery_kind`, `error_category`, `outcome`                                   |
| `openclaw_message_delivery_duration_seconds`     | ヒストグラム | `channel`, `delivery_kind`, `error_category`, `outcome`                                   |
| `openclaw_talk_event_total`                      | カウンター   | `brain`, `event_type`, `mode`, `provider`, `transport`                                    |
| `openclaw_talk_event_duration_seconds`           | ヒストグラム | `brain`, `event_type`, `mode`, `provider`, `transport`                                    |
| `openclaw_talk_audio_bytes`                      | ヒストグラム | `brain`, `event_type`, `mode`, `provider`, `transport`                                    |
| `openclaw_queue_lane_size`                       | ゲージ     | `lane`                                                                                    |
| `openclaw_queue_lane_wait_seconds`               | ヒストグラム | `lane`                                                                                    |
| `openclaw_session_state_total`                   | カウンター   | `reason`, `state`                                                                         |
| `openclaw_session_queue_depth`                   | ゲージ     | `state`                                                                                   |
| `openclaw_session_turn_created_total`            | カウンター   | `agent`, `channel`, `trigger`                                                             |
| `openclaw_session_stuck_total`                   | カウンター   | `reason`, `state`                                                                         |
| `openclaw_session_stuck_age_seconds`             | ヒストグラム | `reason`, `state`                                                                         |
| `openclaw_session_recovery_total`                | カウンター   | `action`, `active_work_kind`, `state`, `status`                                           |
| `openclaw_session_recovery_age_seconds`          | ヒストグラム | `action`, `active_work_kind`, `state`, `status`                                           |
| `openclaw_liveness_warning_total`                | カウンター   | `reason`                                                                                  |
| `openclaw_liveness_sessions`                     | ゲージ     | `state`                                                                                   |
| `openclaw_liveness_event_loop_delay_p99_seconds` | ヒストグラム | `reason`                                                                                  |
| `openclaw_liveness_event_loop_delay_max_seconds` | ヒストグラム | `reason`                                                                                  |
| `openclaw_liveness_event_loop_utilization_ratio` | ヒストグラム | `reason`                                                                                  |
| `openclaw_liveness_cpu_core_ratio`               | ヒストグラム | `reason`                                                                                  |
| `openclaw_payload_large_total`                   | カウンター   | `action`, `channel`, `plugin`, `reason`, `surface`                                        |
| `openclaw_payload_large_bytes`                   | ヒストグラム | `action`, `channel`, `plugin`, `reason`, `surface`                                        |
| `openclaw_memory_bytes`                          | ゲージ     | `kind`                                                                                    |
| `openclaw_memory_rss_bytes`                      | ヒストグラム | なし                                                                                      |
| `openclaw_memory_pressure_total`                 | カウンター   | `level`, `reason`                                                                         |
| `openclaw_telemetry_exporter_total`              | カウンター   | `exporter`, `reason`, `signal`, `status`                                                  |
| `openclaw_prometheus_series_dropped_total`       | カウンター   | なし                                                                                      |
| `openclaw_diagnostic_async_queue_dropped_total`  | カウンター   | `drop_class`                                                                              |
| `openclaw_diagnostic_async_queue_length`         | ゲージ     | なし                                                                                      |

モデル呼び出しメトリクスでは、`observation_unit="request"` は観測可能な
プロバイダーリクエスト 1 件を測定します。`observation_unit="turn"` は、複数の非表示のプロバイダーリクエストを含む可能性がある、合成された Claude Code
または Codex CLI エージェントターンを測定します。
レイテンシを比較する際は、これらの系列を分けてください。

## ラベルポリシー

<AccordionGroup>
  <Accordion title="制限された低カーディナリティのラベル">
    Prometheus ラベルは、制限された低カーディナリティに保たれます。エクスポーターは、`runId`、`sessionKey`、`sessionId`、`callId`、`toolCallId`、メッセージ ID、チャット ID、プロバイダーリクエスト ID などの生の診断識別子を出力しません。

    ラベル値は秘匿化され、OpenClaw の低カーディナリティ文字ポリシーに一致する必要があります。ポリシーに違反する値は、メトリクスに応じて `unknown`、`other`、または `none` に置き換えられます。スコープ付きエージェントセッションキーに見えるラベルも、`unknown` に置き換えられます。

  </Accordion>
  <Accordion title="系列上限と超過数の記録">
    エクスポーターがメモリ内に保持する時系列は、カウンター、ゲージ、ヒストグラムの合計で **2048** 系列に制限されます。この上限を超える新しい系列は破棄され、そのたびに `openclaw_prometheus_series_dropped_total` が 1 増加します。

    このカウンターは、上流の属性から高カーディナリティ値が漏れていることを示す明確なシグナルとして監視してください。エクスポーターが上限を自動的に引き上げることはありません。値が増加した場合は、上限を無効にするのではなく、発生源を修正してください。

  </Accordion>
  <Accordion title="Prometheus 出力に決して含まれないもの">
    - プロンプトテキスト、応答テキスト、ツール入力、ツール出力、システムプロンプト
    - 通話の文字起こし、音声ペイロード、通話 ID、ルーム ID、ハンドオフトークン、ターン ID、生のセッション ID
    - 生のプロバイダーリクエスト ID（該当する場合でも、スパンには制限されたハッシュのみを使用し、メトリクスには決して使用しません）
    - セッションキーとセッション ID
    - ホスト名、ファイルパス、シークレット値

  </Accordion>
</AccordionGroup>

## PromQL レシピ

```promql
# プロバイダー別の 1 分あたりのトークン数
sum by (provider) (rate(openclaw_model_tokens_total[1m]))

# 過去 1 時間のモデル別コスト（USD）
sum by (model) (increase(openclaw_model_cost_usd_total[1h]))

# モデル実行時間の 95 パーセンタイル
histogram_quantile(
  0.95,
  sum by (le, provider, model)
    (rate(openclaw_run_duration_seconds_bucket[5m]))
)

# キュー待機時間の SLO（95 パーセンタイルが 2 秒未満）
histogram_quantile(
  0.95,
  sum by (le, lane) (rate(openclaw_queue_lane_wait_seconds_bucket[5m]))
) < 2

# 制限されたソース別の Skill 使用状況
sum by (skill, source) (increase(openclaw_skill_used_total[24h]))

# 破棄された Prometheus 系列（カーディナリティアラーム）
increase(openclaw_prometheus_series_dropped_total[15m]) > 0
```

<Tip>
プロバイダー横断ダッシュボードには `gen_ai_client_token_usage` を推奨します。これは OpenTelemetry GenAI セマンティック規約に準拠し、OpenClaw 以外の GenAI サービスのメトリクスとも一貫しています。
</Tip>

## Prometheus と OpenTelemetry エクスポートの選択

OpenClaw は両方のサーフェスを独立してサポートしています。どちらか一方、両方、またはどちらも使用しない構成で実行できます。

<Tabs>
  <Tab title="diagnostics-prometheus">
    - **プル**モデル：Prometheus が `/api/diagnostics/prometheus` をスクレイプします。
    - 外部コレクターは不要です。
    - 通常の Gateway 認証によって認証されます。
    - サーフェスはメトリクスのみです（トレースやログは含まれません）。
    - すでに Prometheus + Grafana で標準化されているスタックに最適です。

  </Tab>
  <Tab title="diagnostics-otel">
    - **プッシュ**モデル：OpenClaw が OTLP/HTTP を使用してコレクターまたは OTLP 互換バックエンドに送信します。
    - サーフェスにはメトリクス、トレース、ログが含まれます。
    - 両方が必要な場合は、OpenTelemetry Collector（`prometheus` または `prometheusremotewrite` エクスポーター）を介して Prometheus に接続します。
    - 完全なカタログについては、[OpenTelemetry エクスポート](/ja-JP/gateway/opentelemetry)を参照してください。

  </Tab>
</Tabs>

## トラブルシューティング

<AccordionGroup>
  <Accordion title="応答本文が空">
    - 設定で `diagnostics.enabled` が `false` に設定されていないことを確認してください（デフォルトは `true` です）。
    - `openclaw plugins list --enabled` を使用して、Plugin が有効化され、読み込まれていることを確認してください。
    - トラフィックを発生させてください。カウンターとヒストグラムは、少なくとも 1 件のイベントが発生するまで行を出力しません。

  </Accordion>
  <Accordion title="401 / 未認証">
    エンドポイントには Gateway オペレータースコープ（`gatewayRuntimeScopeSurface: "trusted-operator"` を指定した `auth: "gateway"`）が必要です。Prometheus が他の Gateway オペレータールートで使用するものと同じトークンまたはパスワードを使用してください。公開された未認証モードはありません。
  </Accordion>
  <Accordion title="`openclaw_prometheus_series_dropped_total` が増加している">
    新しい属性が **2048** 系列の上限を超えています。最近のメトリクスを調べ、予期せずカーディナリティが高くなっているラベルを特定して、発生源で修正してください。エクスポーターはラベルを暗黙的に書き換えるのではなく、意図的に新しい系列を破棄します。
  </Accordion>
  <Accordion title="再起動後も Prometheus に古い系列が表示される">
    Plugin は状態をメモリ内にのみ保持します。Gateway の再起動後、カウンターはゼロにリセットされ、ゲージは次に報告された値から再開します。リセットを適切に処理するには、PromQL の `rate()` と `increase()` を使用してください。
  </Accordion>
</AccordionGroup>

## 関連項目

- [診断エクスポート](/ja-JP/gateway/diagnostics) — サポートバンドル用のローカル診断 zip
- [ヘルスチェックと準備状況](/ja-JP/gateway/health) — `/healthz` および `/readyz` プローブ
- [ログ記録](/ja-JP/logging) — ファイルベースのログ記録
- [OpenTelemetry エクスポート](/ja-JP/gateway/opentelemetry) — トレース、メトリクス、ログの OTLP プッシュ
