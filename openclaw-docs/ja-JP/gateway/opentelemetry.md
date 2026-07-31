---
read_when:
    - OpenClaw のモデル使用状況、メッセージフロー、またはセッションメトリクスを OpenTelemetry コレクターに送信する場合
    - Grafana、Datadog、Honeycomb、New Relic、Tempo、またはその他の OTLP バックエンドにトレース、メトリクス、ログを連携する場合
    - ダッシュボードやアラートを構築するには、正確なメトリクス名、スパン名、または属性の形式が必要です
summary: diagnostics-otel Plugin を介して、OpenClaw の診断情報を OpenTelemetry コレクターまたは stdout JSONL にエクスポートする
title: OpenTelemetry エクスポート
x-i18n:
    generated_at: "2026-07-26T09:02:48Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 6ed37f094c6c151379d8e0aaa2633b3ebebdb08b7dcbc9403c4bdeb6e5b8cf76
    source_path: gateway/opentelemetry.md
    workflow: 16
---

OpenClaw は、公式の `diagnostics-otel` Plugin を通じて
**OTLP/HTTP (protobuf)** を使用して診断情報をエクスポートします。ログは、コンテナおよび
サンドボックスのログパイプライン向けに stdout JSONL として書き込むこともできます。
OTLP/HTTP を受け付ける任意のコレクターまたはバックエンドを、コードを変更せずに使用できます。
ローカルファイルのログについては、[ロギング](/ja-JP/logging)を参照してください。

- **診断イベント**は、モデル実行、メッセージフロー、セッション、キュー、
  exec について Gateway およびバンドルされたプラグインが生成する、構造化されたプロセス内レコードです。
- **`diagnostics-otel`** はこれらのイベントをサブスクライブし、OTLP/HTTP 経由で
  OpenTelemetry の**メトリクス**、**トレース**、**ログ**としてエクスポートします。また、
  ログレコードを stdout JSONL にミラーリングできます。
- **プロバイダー呼び出し**では、プロバイダーのトランスポートがカスタムヘッダーを受け付ける場合、
  OpenClaw の信頼されたモデル呼び出しスパンコンテキストから W3C `traceparent` ヘッダーを受け取ります。
  Plugin が生成したトレースコンテキストは伝播されません。
- エクスポーターは、診断サーフェスと Plugin の両方が有効な場合にのみ接続されるため、
  デフォルトではプロセス内コストがほぼゼロに保たれます。

## クイックスタート

```bash
openclaw plugins install clawhub:@openclaw/diagnostics-otel
```

```json5
{
  plugins: {
    allow: ["diagnostics-otel"],
    entries: {
      "diagnostics-otel": { enabled: true },
    },
  },
  diagnostics: {
    enabled: true,
    otel: {
      enabled: true,
      endpoint: "http://otel-collector:4318",
      protocol: "http/protobuf",
      serviceName: "openclaw-gateway",
      traces: true,
      metrics: true,
      logs: true,
      sampleRate: 0.2,
      flushIntervalMs: 60000,
    },
  },
}
```

または CLI から Plugin を有効にします: `openclaw plugins enable diagnostics-otel`。

<Note>
`protocol` は `http/protobuf` のみをサポートします。`traces` と `metrics` はデフォルトで有効になっているため、他の値（`grpc` を含む）を指定すると、`unsupported protocol` 警告により diagnostics-otel サブスクリプション全体が中止されます。これにより、stdout へのログエクスポートも停止します。非 OTLP プロトコル値で `logsExporter: "stdout"` のみを使用する場合は、`traces: false` と `metrics: false` を明示的に設定してください。
</Note>

## エクスポートされるシグナル

| シグナル      | 含まれる内容                                                                                                                                                                                              |
| ----------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| **メトリクス** | トークン使用量、コスト、実行時間、フェイルオーバー、Skills の使用状況、メッセージフロー、Talk イベント、キューレーン、セッションの状態と復旧、ツール実行、exec、メモリ、稼働状態、エクスポーターの健全性に関するカウンターとヒストグラム。 |
| **トレース**  | モデルの使用、モデル呼び出し、ハーネスのライフサイクル、Skills の使用状況、ツール実行、exec、Webhook とメッセージの処理、コンテキストの組み立て、ツールループに関するスパン。                                                      |
| **ログ**    | `diagnostics.otel.logs` が有効な場合に OTLP または stdout JSONL 経由でエクスポートされる、構造化された `logging.file` レコード。コンテンツキャプチャが明示的に有効化されていない限り、ログ本文は出力されません。                          |

`traces`、`metrics`、`logs` は個別に切り替えられます。`diagnostics.otel.enabled` が true の場合、トレースとメトリクスは
デフォルトでオンになります。ログはデフォルトでオフになり、
`diagnostics.otel.logs` が明示的に `true` の場合にのみエクスポートされます。ログのエクスポート先は
デフォルトで OTLP です。stdout に JSONL を出力するには `diagnostics.otel.logsExporter` を `stdout` に、
両方に出力するには `both` に設定します。

## 設定リファレンス

```json5
{
  diagnostics: {
    enabled: true,
    otel: {
      enabled: true,
      endpoint: "http://otel-collector:4318",
      tracesEndpoint: "http://otel-collector:4318/v1/traces",
      metricsEndpoint: "http://otel-collector:4318/v1/metrics",
      logsEndpoint: "http://otel-collector:4318/v1/logs",
      protocol: "http/protobuf", // grpc は OTLP エクスポートを無効にする
      serviceName: "openclaw-gateway", // 未設定の場合は OTEL_SERVICE_NAME、次に "openclaw" にフォールバックする
      headers: { "x-collector-token": "..." },
      traces: true,
      metrics: true,
      logs: true,
      logsExporter: "otlp", // otlp | stdout | both
      sampleRate: 0.2, // ルートスパンのサンプラー、0.0..1.0
      flushIntervalMs: 60000, // メトリクスのエクスポート間隔（最小 1000ms）
      captureContent: {
        enabled: false,
        inputMessages: false,
        outputMessages: false,
        toolInputs: false,
        toolOutputs: false,
        systemPrompt: false,
        toolDefinitions: false,
      },
    },
  },
}
```

### 環境変数

| 変数                                                                                                          | 用途                                                                                                                                                                                                                                                                                                        |
| ----------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `OTEL_EXPORTER_OTLP_ENDPOINT`                                                                                     | 設定キーが未設定の場合の `diagnostics.otel.endpoint` のフォールバック。                                                                                                                                                                                                                                         |
| `OTEL_EXPORTER_OTLP_TRACES_ENDPOINT` / `OTEL_EXPORTER_OTLP_METRICS_ENDPOINT` / `OTEL_EXPORTER_OTLP_LOGS_ENDPOINT` | 対応する `diagnostics.otel.*Endpoint` 設定キーが未設定の場合に使用される、シグナル固有のエンドポイントのフォールバック。シグナル固有の設定はシグナル固有の環境変数より優先され、シグナル固有の環境変数は共有エンドポイントより優先されます。                                                                                                         |
| `OTEL_SERVICE_NAME`                                                                                               | 設定キーが未設定の場合の `diagnostics.otel.serviceName` のフォールバック。デフォルトのサービス名は `openclaw` です。                                                                                                                                                                                                  |
| `OTEL_EXPORTER_OTLP_PROTOCOL`                                                                                     | `diagnostics.otel.protocol` が未設定の場合のワイヤープロトコルのフォールバック。`http/protobuf` のみがエクスポートを有効にします。                                                                                                                                                                                                 |
| `OTEL_SEMCONV_STABILITY_OPT_IN`                                                                                   | 最新の GenAI 推論スパン形式（`{gen_ai.operation.name} {gen_ai.request.model}` スパン名、`CLIENT` スパン種別、従来の `gen_ai.system` の代わりとなる `gen_ai.provider.name`）を生成するには、`gen_ai_latest_experimental` に設定します。GenAI メトリクスでは、この設定に関係なく、範囲が限定された低カーディナリティ属性が常に使用されます。 |
| `OPENCLAW_OTEL_PRELOADED`                                                                                         | 別のプリロードまたはホストプロセスがグローバル OpenTelemetry SDK をすでに登録している場合は、`1` に設定します。Plugin は自身の NodeSDK ライフサイクルをスキップしますが、診断リスナーの接続は継続し、`traces`/`metrics`/`logs` に従います。                                                                                    |

## プライバシーとコンテンツキャプチャ

モデルやツールの生コンテンツは、デフォルトでは**エクスポートされません**。スパンには、
範囲が限定された識別子（チャンネル、プロバイダー、モデル、エラーカテゴリ、ハッシュのみのリクエスト ID、
ツールのソース、ツールの所有者、Skills 名とソース）が含まれますが、プロンプトテキスト、
応答テキスト、ツール入力、ツール出力、Skills ファイルパス、セッションキーは一切含まれません。
スコープ付きエージェントセッションキーのように見える値（たとえば
`agent:` で始まる値）は、低カーディナリティ属性では `unknown` に置き換えられます。OTLP ログ
レコードは、デフォルトで重大度、ロガー、コード位置、信頼されたトレースコンテキスト、
サニタイズ済み属性を保持します。生のログメッセージ本文は、
`diagnostics.otel.captureContent` がブール値 `true` の場合にのみエクスポートされます。個別の
`captureContent.*` サブキーによってログ本文が有効になることはありません。Talk メトリクスは、
範囲が限定されたイベントメタデータ（モード、トランスポート、プロバイダー、イベント種別）のみをエクスポートし、
トランスクリプト、音声ペイロード、セッション ID、ターン ID、通話 ID、ルーム ID、
ハンドオフトークンはエクスポートしません。

送信モデルリクエストには、アクティブなモデル呼び出しについて
OpenClaw が所有する診断トレースコンテキストのみから生成された W3C `traceparent` ヘッダーが含まれる場合があります。
呼び出し元が指定した既存の `traceparent` ヘッダーは置き換えられるため、Plugin や
カスタムプロバイダーオプションによってサービス間のトレース祖先関係を偽装することはできません。

コレクターおよび保持ポリシーが、プロンプト、応答、ツール、または
システムプロンプトのテキストについて承認されている場合にのみ、`diagnostics.otel.captureContent.*` を `true` に設定してください。
各サブキーは独立しています。

- `inputMessages` - ユーザープロンプトの内容。
- `outputMessages` - モデル応答の内容。
- `toolInputs` - ツール引数のペイロード。
- `toolOutputs` - ツール結果のペイロード。
- `systemPrompt` - 組み立てられたシステム/開発者プロンプト。
- `toolDefinitions` - モデルツールの名前、説明、スキーマ。

いずれかのサブキーが有効な場合、モデルスパンとツールスパンには、
そのクラスについてのみ、範囲が限定され編集された `openclaw.content.*` 属性が付与されます。

<Note>
ブール値 `captureContent: true` は、`inputMessages`、`outputMessages`、`toolInputs`、`toolOutputs`、`toolDefinitions`、および OTLP ログ本文をまとめて有効にしますが、`systemPrompt` は有効に**しません**。組み立てられたシステムプロンプトも必要な場合は、`captureContent.systemPrompt: true` を明示的に設定してください。
</Note>

`toolInputs`/`toolOutputs` の内容は、組み込みエージェント
ランタイムのツール実行についてキャプチャされます（完了/エラーのスパンでは
`openclaw.content.tool_input` と `gen_ai.tool.call.arguments`、
完了スパンでは `openclaw.content.tool_output` と
`gen_ai.tool.call.result`）。`openclaw.content.*` の名前は、安定した OpenClaw 属性名として
維持されます。`gen_ai.tool.call.*` のコピーは、semconv ネイティブのビューアー向けにそれらをミラーリングします。
外部ハーネスのツール呼び出し（Codex、Claude CLI）は、
コンテンツペイロードを含まない `tool.execution.*` スパンを生成します。キャプチャされたコンテンツは、
信頼されたリスナー専用チャンネルを通じて伝送され、公開診断イベントバスには
一切配置されません。

## サンプリングとフラッシュ

- **トレース:** `diagnostics.otel.sampleRate` はルートスパンのみに `TraceIdRatioBasedSampler`
  を設定します（`0.0` はすべて破棄し、`1.0` はすべて保持します）。未設定の場合は
  OpenTelemetry SDK のデフォルト（常時オン）を使用します。
- **メトリクス:** `diagnostics.otel.flushIntervalMs`（最小値
  `1000` に制限）。未設定の場合は SDK の定期エクスポートのデフォルトを使用します。
- **ログ:** OTLP ログは `logging.level`（ファイルログレベル）に従い、
  コンソールの書式設定ではなく、診断ログレコードの秘匿化パスを使用します。大量のログが発生する
  インストール環境では、ローカルサンプリングよりも OTLP コレクターのサンプリングやフィルタリングを
  優先してください。プラットフォームがすでに stdout/stderr をログプロセッサーへ
  送信しており、OTLP ログコレクターがない場合は `diagnostics.otel.logsExporter: "stdout"` を設定します。
  stdout レコードは、`ts`、`signal`、
  `service.name`、重大度、本文、秘匿化された属性、および利用可能な場合は信頼済みのトレース
  フィールドを含む、1 行につき 1 つの JSON オブジェクトです。
- **ファイルログの相関:** JSONL ファイルログには、ログ呼び出しに有効な
  診断トレースコンテキストが含まれる場合、トップレベルの `traceId`、
  `spanId`、`parentSpanId`、および `traceFlags` が含まれます。これにより、ログプロセッサーはローカルのログ行を
  エクスポートされたスパンと結合できます。
- **リクエストの相関:** Gateway の HTTP リクエストと WebSocket フレームは、
  内部リクエストトレーススコープを作成します。そのスコープ内のログと診断イベントは
  デフォルトでリクエストトレースを継承し、エージェント実行とモデル呼び出しの
  スパンは子として作成されるため、プロバイダーの `traceparent` ヘッダーは同じ
  トレース上に維持されます。
- **モデル呼び出しの相関:** `openclaw.model.call` スパンには、デフォルトで安全なプロンプト
  コンポーネントサイズが含まれ、プロバイダーの結果で使用量が公開される場合は呼び出しごとのトークン属性も含まれます。
  `openclaw.model.usage` は、コスト、コンテキスト、およびチャンネルのダッシュボードを集計するための実行レベルの
  会計スパンとして引き続き使用され、イベントを発行するランタイムに信頼済みの
  トレースコンテキストがある場合は同じ診断トレース上に維持されます。

### モデル呼び出しの観測単位

各 `openclaw.model.call` スパンは、そのライフサイクルが測定する対象を
`openclaw.model_call.observation_unit` で識別します。

- `request` - 観測可能な 1 回のモデル／プロバイダーリクエスト。ネイティブの組み込みモデル
  呼び出しはこの単位を使用し、古い発行元または外部の発行元との互換性のため、
  値がない場合、エクスポーターは `request` として扱います。
- `turn` - 非公開のモデルリクエスト、再試行、ツール処理、またはバックグラウンド処理を含む可能性がある、不透明な 1 回のエージェント CLI ターン。
  Claude Code CLI と Codex app-server の呼び出しはこの単位を使用します。

どちらの単位もモデル呼び出しスパンのままであるため、トレースバックエンドはモデルの入力、
出力、使用量、および階層をレンダリングできます。リクエストスパンは API から導出された GenAI オペレーション
（`chat`、`generate_content`、または `text_completion`）を使用し、ターンスパンは
`gen_ai.operation.name = invoke_agent` を使用します。どちらも
`gen_ai.client.operation.duration` に寄与し、オペレーション名によって直接の
リクエストレイテンシーとターン全体のレイテンシーを区別します。OpenClaw の OTEL モデル呼び出し
メトリクスには `openclaw.model_call.observation_unit` も含まれ、Prometheus の
モデル呼び出しメトリクスでは同等の `observation_unit` ラベルが公開されます。

### Claude Code CLI のモデル呼び出し忠実度

Claude Code CLI の各ターンは、ターンレベルの合成 `openclaw.model.call`
スパンを 1 つ発行します。これは Anthropic HTTP リクエストスパンではありません。`openclaw.api =
claude-code`、`openclaw.model_call.observation_unit = turn` を使用し、
オペレーションを `gen_ai.operation.name = invoke_agent` として識別します。
OpenClaw の CLI 境界は
`openclaw.transport` によって識別します。

- `stdio` - 1 回限りのローカル Claude Code プロセス。
- `stdio-live` - 管理対象の永続 Claude stdio セッション上の 1 ターン。
- `paired-node-cli` - ペアリング済みの
  Node に委任された 1 回限りの Claude Code 実行。

Claude CLI の診断は、プロセス診断ディスパッチャーが有効であり、内部または信頼済みのイベントリスナーが接続されている間のみ
インスタンス化されます。可観測性 Plugin またはその他のリスナーがアクティブでない場合、Claude CLI のターンでは
合成トレース階層、コンテンツバッファー、および診断ストリームのバイト数集計を
省略します。コンテンツキャプチャが有効な場合、プロンプトとシステムプロンプトのフィールドは
それぞれ 128 KiB に制限されます。アシスタント出力は最大 200 個のエンベロープ全体で 128 KiB に制限され、
最終的に表示されるフォールバック応答用として 16 KiB と 1 項目が予約されます。
上限に達すると、切り捨てを示すマーカーが記録されます。

OpenClaw は Claude CLI のターンに、他のエージェントランタイムで使用されるものと同じ
所有権階層を適用します。`openclaw.harness.run`（`openclaw.harness.id = claude-cli`）
には `openclaw.run` が含まれ、その中に Claude の `openclaw.model.call`
スパンが含まれます。ハーネスと実行スパンは OpenClaw の合成ターン境界であり、
Claude Code の内部フェーズではありません。1 回限りのターンと管理対象 stdio のターンは同じ
階層を使用します。実際に新規セッションで再試行すると、同じ OpenClaw 実行内に別のモデル呼び出しの子が作成されます。

スパンは、OpenClaw が準備済みの CLI ターンを受け入れた時点で開始し、
そのターンが成功または失敗した後にのみ終了します。管理対象セッションでは、Claude が結果を保持するバックグラウンドエージェントまたは
ワークフローを報告している間は、途中の成功結果でスパンは終了せず、
ドレイン後の最終結果で終了します。中止、タイムアウト、プロセス障害、
出力／解析の失敗、およびその他のターン失敗では、同じスパンがエラーとして終了します。

Claude Code はアシスタントメッセージごとの使用量を報告し、終了結果で累積
使用量も報告する場合があります。OpenClaw の応答集計では、既存のコストセマンティクスを変更しないよう
引き続き最後のアシスタントメッセージを使用します。ターンレベルのモデル呼び出しスパンでは、
利用可能な場合、キャッシュ読み取りトークンとキャッシュ作成トークンを含む終了時の累積使用量を使用します。

これらの CLI スパンでは、バイト数とタイミングのフィールドは観測可能な OpenClaw
CLI 境界を表します。

- `openclaw.model_call.request_bytes` は、1 回限りの stdin/argv を介して送信されたプロンプト値、
  または管理対象 stdio の JSONL ユーザーエンベロープの UTF-8 サイズです。
  Claude Code の非公開モデルリクエストのサイズではありません。
- `openclaw.model_call.response_bytes` は、ターン中に観測された Claude CLI stdout の
  UTF-8 サイズです。Anthropic HTTP 応答のサイズではありません。
- `openclaw.model_call.time_to_first_byte_ms` は、観測可能な最初の
  Claude CLI stdout または stderr 出力までの時間です。ネットワーク TTFB ではありません。

対応する粒度の細かい `captureContent` フィールドを有効にすると、スパンは
OpenClaw が Claude Code に送信する実効プロンプト、OpenClaw が追加したシステム
プロンプト、および表示可能なアシスタントのテキスト／推論／ツール呼び出し識別情報を
`gen_ai.input.messages`、`gen_ai.output.messages`、および
`gen_ai.system_instructions` を通じてエクスポートします。ツール引数、不透明な思考署名、および
ツール結果は Claude アシスタントエンベロープから省略されます。OpenClaw は、
Claude Code の非公開システムプロンプト、非公開の再開済みまたは
圧縮済みリクエストペイロード、ネイティブの内部ツールスキーマ、生の Anthropic HTTP
リクエスト、内部再試行、アップストリームリクエスト ID、または真のネットワーク TTFB にアクセスできるとは主張しません。
Claude Code は実効的なネイティブツール定義を正確に公開しないため、
これらのスパンには `gen_ai.tool.definitions` を設定しません。

外部 Claude ハーネスのツールスパンは、ツールコンテンツの
キャプチャが有効な場合でもメタデータのみです。すべてのモデルスパンと同様に、キャプチャされた Claude CLI コンテンツは
信頼済みリスナー専用のパスと、エクスポーターの既存の秘匿化およびサイズ
制限を使用します。コンテンツはデフォルトでは無効です。

## エクスポートされるメトリクス

### モデル使用量

- `openclaw.tokens`（カウンター、属性: `openclaw.token`、`openclaw.channel`、`openclaw.provider`、`openclaw.model`、`openclaw.agent`）
- `openclaw.cost.usd`（カウンター、属性: `openclaw.channel`、`openclaw.provider`、`openclaw.model`）
- `openclaw.run.duration_ms`（ヒストグラム、属性: `openclaw.channel`、`openclaw.provider`、`openclaw.model`）
- `openclaw.context.tokens`（ヒストグラム、属性: `openclaw.context`、`openclaw.channel`、`openclaw.provider`、`openclaw.model`）
- `gen_ai.client.token.usage`（ヒストグラム、GenAI セマンティック規約メトリクス、属性: `gen_ai.token.type` = `input`/`output`、`gen_ai.provider.name`、`gen_ai.operation.name`、`gen_ai.request.model`）
- `gen_ai.client.operation.duration`（ヒストグラム、秒、モデルリクエストおよび合成エージェントターン用の GenAI セマンティック規約メトリクス。属性: `gen_ai.provider.name`、`gen_ai.operation.name`、`gen_ai.request.model`、任意の `error.type`。ターンの観測では `gen_ai.operation.name = invoke_agent` を使用）
- `openclaw.model_call.duration_ms`（ヒストグラム、属性: `openclaw.provider`、`openclaw.model`、`openclaw.api`、`openclaw.transport`、`openclaw.model_call.observation_unit`、さらに分類済みエラーでは `openclaw.errorCategory` および `openclaw.failureKind`）
- `openclaw.model_call.request_bytes`（ヒストグラム、最終モデルリクエストペイロードの UTF-8 バイトサイズ。Claude Code CLI の場合は前述の観測可能なプロンプト入力／エンベロープ。生のペイロードコンテンツは含まれません）
- `openclaw.model_call.response_bytes`（ヒストグラム、ストリーミング応答チャンクペイロードの UTF-8 バイトサイズ。高頻度のテキスト、思考、およびツール呼び出しの差分では、増分の `delta` バイトのみを計上。Claude Code CLI の場合は観測された stdout バイト。生の応答コンテンツは含まれません）
- `openclaw.model_call.time_to_first_byte_ms`（ヒストグラム、最初のストリーミング応答イベントまでの経過時間。Claude Code CLI の場合はネットワーク TTFB ではなく、観測可能な最初の CLI 出力）
- `openclaw.model.failover`（カウンター、属性: `openclaw.provider`、`openclaw.model`、`openclaw.failover.to_provider`、`openclaw.failover.to_model`、`openclaw.failover.reason`、`openclaw.failover.suspended`、`openclaw.lane`）
- `openclaw.skill.used`（カウンター、属性: `openclaw.skill.name`、`openclaw.skill.source`、`openclaw.skill.activation`、任意の `openclaw.agent`、任意の `openclaw.toolName`）

### メッセージフロー

- `openclaw.webhook.received`（カウンター、属性: `openclaw.channel`、`openclaw.webhook`）
- `openclaw.webhook.error`（カウンター、属性: `openclaw.channel`、`openclaw.webhook`）
- `openclaw.webhook.duration_ms`（ヒストグラム、属性: `openclaw.channel`、`openclaw.webhook`）
- `openclaw.message.queued`（カウンター、属性: `openclaw.channel`、`openclaw.source`）
- `openclaw.message.received`（カウンター、属性: `openclaw.channel`、`openclaw.source`）
- `openclaw.message.dispatch.started`（カウンター、属性: `openclaw.channel`、`openclaw.source`）
- `openclaw.message.dispatch.completed`（カウンター、属性: `openclaw.channel`、`openclaw.outcome`、`openclaw.reason`、`openclaw.source`）
- `openclaw.message.dispatch.duration_ms`（ヒストグラム、属性: `openclaw.channel`、`openclaw.outcome`、`openclaw.reason`、`openclaw.source`）
- `openclaw.message.processed`（カウンター、属性: `openclaw.channel`、`openclaw.outcome`）
- `openclaw.message.duration_ms`（ヒストグラム、属性: `openclaw.channel`、`openclaw.outcome`）
- `openclaw.message.delivery.started`（カウンター、属性: `openclaw.channel`、`openclaw.delivery.kind`）
- `openclaw.message.delivery.duration_ms`（ヒストグラム、属性: `openclaw.channel`、`openclaw.delivery.kind`、`openclaw.outcome`、`openclaw.errorCategory`）

### Talk

- `openclaw.talk.event`（カウンター、属性: `openclaw.talk.event_type`、`openclaw.talk.mode`、`openclaw.talk.transport`、`openclaw.talk.brain`、`openclaw.talk.provider`）
- `openclaw.talk.event.duration_ms`（ヒストグラム、属性: `openclaw.talk.event` と同じ。Talk イベントが継続時間を報告したときに発行）
- `openclaw.talk.audio.bytes`（ヒストグラム、属性: `openclaw.talk.event` と同じ。バイト長を報告する Talk 音声フレームイベントに対して発行）

### キューとセッション

- `openclaw.queue.lane.enqueue`（カウンター、属性: `openclaw.lane`）
- `openclaw.queue.lane.dequeue`（カウンター、属性: `openclaw.lane`）
- `openclaw.queue.depth`（ヒストグラム、属性: `openclaw.lane` または `openclaw.channel=heartbeat`）
- `openclaw.queue.wait_ms`（ヒストグラム、属性: `openclaw.lane`）
- `openclaw.session.state`（カウンター、属性: `openclaw.state`、`openclaw.reason`）
- `openclaw.session.stuck`（カウンター、属性: `openclaw.state`。回復可能な古いセッション管理情報に対して発行）
- `openclaw.session.stuck_age_ms`（ヒストグラム、属性: `openclaw.state`。回復可能な古いセッション管理情報に対して発行）
- `openclaw.session.turn.created`（カウンター、属性: `openclaw.agent`、`openclaw.channel`、`openclaw.trigger`）
- `openclaw.session.recovery.requested`（カウンター、属性: `openclaw.state`、`openclaw.action`、`openclaw.active_work_kind`、`openclaw.reason`）
- `openclaw.session.recovery.completed`（カウンター、属性: `openclaw.state`、`openclaw.action`、`openclaw.status`、`openclaw.active_work_kind`、`openclaw.reason`）
- `openclaw.session.recovery.age_ms`（ヒストグラム、属性: 対応する回復カウンターと同じ）
- `openclaw.run.attempt`（カウンター、属性: `openclaw.attempt`）

### セッションの生存性テレメトリ

OpenClaw が応答、ツール、ステータス、ブロック、または ACP ランタイムの進行を観測している間、`processing` セッションは組み込みの生存性しきい値に向かって経過時間が加算されません。入力中を示すキープアライブは進行として数えられないため、応答のないモデルやハーネスも引き続き検出できます。

OpenClaw は、引き続き観測可能な処理に基づいてセッションを分類します。

- `session.long_running`: アクティブな組み込み処理、モデル呼び出し、またはツール呼び出しが
  引き続き進行しています。所有者のいる無応答のモデル呼び出しも、組み込みの中止しきい値に達するまでは長時間実行中として報告されるため、中止を観測できる間は、低速または非ストリーミングのモデルプロバイダーが停止した Gateway セッションと見なされることはありません。
- `session.stalled`: アクティブな処理は存在しますが、アクティブな実行から
  最近の進行が報告されていません。所有者のいるモデル呼び出しは、組み込みの中止しきい値に達するか超えると `session.long_running` から
  `session.stalled` に切り替わります。所有者がなく
  古くなったモデルまたはツールのアクティビティは、無害な長時間実行処理とは見なされません。
  停止した組み込み実行は、最初は観測のみに留まり、その後、
  進行がないまま中止しきい値を超えると中止後の排出処理を行い、レーンの後ろで待機しているターンを再開できるようにします。
- `session.stuck`: アクティブな処理がない古いセッション管理情報、または
  所有者のない古いモデルまたはツールのアクティビティを伴う、アイドル状態のキュー済みセッションです。これにより、
  回復ゲートを通過した直後に、影響を受けたセッションレーンが解放されます。

回復では、構造化された `session.recovery.requested` イベントと
`session.recovery.completed` イベントが発行されます。診断用セッション状態がアイドルとマークされるのは、
状態を変更する回復結果（`aborted` または `released`）の後であり、かつ
同じ処理世代が引き続き現行である場合に限られます。

`openclaw.session.stuck` カウンター、`openclaw.session.stuck_age_ms`
ヒストグラム、および `openclaw.session.stuck` スパンを発行するのは、`session.stuck` だけです。
セッションが変化しない間、繰り返される `session.stuck` 診断はバックオフするため、
ダッシュボードでは Heartbeat の各ティックではなく、増加が継続している場合にアラートを出す必要があります。設定項目とデフォルトについては、
[設定リファレンス](/ja-JP/gateway/configuration-reference#diagnostics)を参照してください。

生存性の警告では、次の項目も発行されます。

- `openclaw.liveness.warning`（カウンター、属性: `openclaw.liveness.reason`）
- `openclaw.liveness.event_loop_delay_p99_ms`（ヒストグラム、属性: `openclaw.liveness.reason`）
- `openclaw.liveness.event_loop_delay_max_ms`（ヒストグラム、属性: `openclaw.liveness.reason`）
- `openclaw.liveness.event_loop_utilization`（ヒストグラム、属性: `openclaw.liveness.reason`）
- `openclaw.liveness.cpu_core_ratio`（ヒストグラム、属性: `openclaw.liveness.reason`）

### ハーネスのライフサイクル

- `openclaw.harness.duration_ms`（ヒストグラム、属性: `openclaw.harness.id`、`openclaw.harness.plugin`、`openclaw.outcome`、エラー時は `openclaw.harness.phase`）

### ツールの実行とループ検出

- `openclaw.tool.execution.duration_ms`（ヒストグラム、属性: `gen_ai.tool.name`、`openclaw.toolName`、`openclaw.tool.source`、`openclaw.tool.owner`、`openclaw.tool.params.kind`、およびエラー時は `openclaw.errorCategory`）
- `openclaw.tool.execution.blocked`（カウンター、属性: `gen_ai.tool.name`、`openclaw.toolName`、`openclaw.tool.source`、`openclaw.tool.owner`、`openclaw.tool.params.kind`、`openclaw.deniedReason`）
- `openclaw.tool.loop`（カウンター、属性: `openclaw.toolName`、`openclaw.loop.level`、`openclaw.loop.action`、`openclaw.loop.detector`、`openclaw.loop.count`、任意の `openclaw.loop.paired_tool`。反復的なツール呼び出しループが検出されたときに発行）

### Exec

- `openclaw.exec.duration_ms`（ヒストグラム、属性: `openclaw.exec.target`、`openclaw.exec.mode`、`openclaw.outcome`、`openclaw.failureKind`）

### 診断の内部情報（メモリ、ペイロード、エクスポーターの状態）

- `openclaw.payload.large`（カウンター、属性: `openclaw.payload.surface`、`openclaw.payload.action`、`openclaw.channel`、`openclaw.plugin`、`openclaw.reason`）
- `openclaw.payload.large_bytes`（ヒストグラム、属性: `openclaw.payload.large` と同じ）
- `openclaw.memory.rss_bytes` / `openclaw.memory.heap_used_bytes` / `openclaw.memory.heap_total_bytes` / `openclaw.memory.external_bytes` / `openclaw.memory.array_buffers_bytes`（ヒストグラム、属性なし。プロセスメモリのサンプル）
- `openclaw.memory.pressure`（カウンター、属性: `openclaw.memory.level`、`openclaw.memory.reason`）
- `openclaw.diagnostic.async_queue.dropped`（カウンター、属性: `openclaw.diagnostic.async_queue.drop_class`。内部診断キューのバックプレッシャーによる破棄）
- `openclaw.telemetry.exporter.events`（カウンター、属性: `openclaw.exporter`、`openclaw.signal`、`openclaw.status`、任意の `openclaw.reason`、任意の `openclaw.errorCategory`。エクスポーターのライフサイクルおよび障害に関する自己テレメトリ）

## エクスポートされるスパン

- `openclaw.model.usage`
  - `openclaw.channel`、`openclaw.provider`、`openclaw.model`
  - `openclaw.tokens.*`（input/output/cache_read/cache_write/total）
  - デフォルトでは `gen_ai.system`、最新の GenAI セマンティック規約をオプトインした場合は `gen_ai.provider.name`
  - `gen_ai.request.model`、`gen_ai.operation.name`、`gen_ai.usage.*`
- `openclaw.run`
  - `openclaw.outcome`、`openclaw.channel`、`openclaw.provider`、`openclaw.model`、`openclaw.errorCategory`
- `openclaw.model.call`
  - デフォルトでは `gen_ai.system`、最新の GenAI セマンティック規約をオプトインした場合は `gen_ai.provider.name`
  - `gen_ai.request.model`、`gen_ai.operation.name`、`openclaw.provider`、`openclaw.model`、`openclaw.api`、`openclaw.transport`、`openclaw.model_call.observation_unit`（`request` または `turn`）
  - `openclaw.errorCategory`、`error.type`、およびエラー時の任意の `openclaw.failureKind`
  - `openclaw.model_call.request_bytes`、`openclaw.model_call.response_bytes`、`openclaw.model_call.time_to_first_byte_ms`
  - `openclaw.model_call.prompt.input_messages_count`、`openclaw.model_call.prompt.input_messages_chars`、`openclaw.model_call.prompt.system_prompt_chars`、`openclaw.model_call.prompt.tool_definitions_count`、`openclaw.model_call.prompt.tool_definitions_chars`、`openclaw.model_call.prompt.total_chars`（安全なコンポーネントサイズのみ。プロンプトテキストは含まない）
  - 結果に該当リクエストまたは集約ターンの使用量が含まれる場合は、`openclaw.model_call.usage.*` および `gen_ai.usage.*`
  - アップストリームプロバイダーの結果でリクエスト ID が公開されている場合、属性 `openclaw.upstreamRequestIdHash`（上限付き、ハッシュベース）を持つスパンイベント `openclaw.provider.request`。生の ID は決してエクスポートされません
  - `OTEL_SEMCONV_STABILITY_OPT_IN=gen_ai_latest_experimental` を使用すると、リクエストスパンでは最新の GenAI 推論スパン名 `{gen_ai.operation.name} {gen_ai.request.model}` が使用されます。OpenClaw は不透明な CLI 境界からネイティブのエージェント名を推定しないため、ターンスパンでは `invoke_agent` が使用されます。どちらも `openclaw.model.call` ではなく、スパン種別 `CLIENT` を使用します。
- `openclaw.harness.run`
  - `openclaw.harness.id`、`openclaw.harness.plugin`、`openclaw.outcome`、`openclaw.provider`、`openclaw.model`、`openclaw.channel`
  - 完了時: `openclaw.harness.result_classification`、`openclaw.harness.yield_detected`、`openclaw.harness.items.started`、`openclaw.harness.items.completed`、`openclaw.harness.items.active`
  - エラー時: `openclaw.harness.phase`、`openclaw.errorCategory`、任意の `openclaw.harness.cleanup_failed`
- `openclaw.tool.execution`
  - `gen_ai.tool.name`、`gen_ai.operation.name`（`execute_tool`）、`openclaw.toolName`、`openclaw.tool.source`、任意の `gen_ai.tool.call.id`、`openclaw.tool.owner`、`openclaw.tool.params.*`
  - エラー時の任意の `openclaw.errorCategory`/`openclaw.errorCode`、ポリシーまたはサンドボックスにより拒否された場合の `openclaw.deniedReason` および `openclaw.outcome=blocked`
- `openclaw.exec`
  - `openclaw.exec.target`、`openclaw.exec.mode`、`openclaw.outcome`、`openclaw.failureKind`、`openclaw.exec.command_length`、`openclaw.exec.exit_code`、`openclaw.exec.exit_signal`、`openclaw.exec.timed_out`
- `openclaw.webhook.processed`
  - `openclaw.channel`、`openclaw.webhook`
- `openclaw.webhook.error`
  - `openclaw.channel`、`openclaw.webhook`、`openclaw.error`
- `openclaw.message.processed`
  - `openclaw.channel`、`openclaw.outcome`、`openclaw.reason`
- `openclaw.message.delivery`
  - `openclaw.channel`、`openclaw.delivery.kind`、`openclaw.outcome`、`openclaw.errorCategory`、`openclaw.delivery.result_count`
- `openclaw.session.stuck`
  - `openclaw.state`、`openclaw.ageMs`、`openclaw.queueDepth`
- `openclaw.context.assembled`
  - `openclaw.prompt.size`、`openclaw.history.size`、`openclaw.context.tokens`、`openclaw.errorCategory`（プロンプト、履歴、応答、セッションキーの内容は含まない）
- `openclaw.tool.loop`
  - `openclaw.toolName`、`openclaw.loop.level`、`openclaw.loop.action`、`openclaw.loop.detector`、`openclaw.loop.count`、任意の `openclaw.loop.paired_tool`（ループメッセージ、パラメーター、ツール出力は含まない）
- `openclaw.memory.pressure`
  - `openclaw.memory.level`、`openclaw.memory.reason`、`openclaw.memory.rss_bytes`、`openclaw.memory.heap_used_bytes`、`openclaw.memory.heap_total_bytes`、`openclaw.memory.external_bytes`、`openclaw.memory.array_buffers_bytes`、任意の `openclaw.memory.threshold_bytes`/`openclaw.memory.rss_growth_bytes`/`openclaw.memory.window_ms`

コンテンツキャプチャを明示的に有効にすると、モデルスパンとツールスパンには、
オプトインした特定のコンテンツクラスについて、上限付きで秘匿処理された
`openclaw.content.*` 属性も含めることができます。

## 診断イベントカタログ

以下のイベントは、前述のメトリクスとスパンの基盤となるか、Plugin から直接サブスクライブできます。
`run.progress` と `run.execution_phase` は直接利用専用の
ライフサイクルシグナルです。diagnostics-otel Plugin はこれらを
独立した OTLP シグナルとしてエクスポートしません。イベント種別と `run.execution_phase.phase` の値は
追加される可能性があります。TypeScript の利用側では、いずれのユニオンも永続的に網羅的であると仮定せず、
default 分岐を維持する必要があります。

**モデル使用量**

- `model.usage` - トークン、コスト、所要時間、コンテキスト、プロバイダー/モデル/チャンネル、
  セッション ID。`usage` はコストとテレメトリに使用するプロバイダー/ターンのアカウンティングです。
  `context.used` は現在のプロンプト/コンテキストのスナップショットであり、
  キャッシュ済み入力またはツールループ呼び出しが関係する場合、プロバイダーの `usage.total` より小さくなることがあります。

**メッセージフロー**

- `webhook.received` / `webhook.processed` / `webhook.error`
- `message.queued` / `message.processed`
- `message.delivery.started` / `message.delivery.completed` / `message.delivery.error`

**キューとセッション**

- `queue.lane.enqueue` / `queue.lane.dequeue`
- `session.state` / `session.long_running` / `session.stalled` / `session.stuck`
- `run.attempt` / `run.progress`
- `run.execution_phase`（公開される、セッションに関連付けられた組み込みランナーの起動マイルストーン）
- `diagnostic.heartbeat`（集約カウンター: Webhook/キュー/セッション）

**ハーネスのライフサイクル**

- `harness.run.started` / `harness.run.completed` / `harness.run.error` -
  エージェントハーネスの実行ごとのライフサイクル。`harnessId`、任意の
  `pluginId`、プロバイダー/モデル/チャンネル、および実行 ID を含みます。完了時には
  `durationMs`、`outcome`、任意の `resultClassification`、`yieldDetected`、
  および `itemLifecycle` のカウントが追加されます。エラー時には `phase`
  （`prepare`/`start`/`send`/`resolve`/`cleanup`）、`errorCategory`、および
  任意の `cleanupFailed` が追加されます。

**Exec**

- `exec.process.completed` - ターミナルの結果、所要時間、ターゲット、モード、終了
  コード、失敗の種類。コマンドテキストと作業ディレクトリは
  含まれません。
- `exec.approval.followup_suppressed` - セッションの再バインド後に破棄された
  古い承認のフォローアップ。`approvalId`、`reason`
  （`session_rebound`）、`phase`（`direct_delivery` または `gateway_preflight`）、
  およびディスパッチャーのタイムスタンプが含まれます。セッションキー、ルート、コマンドテキストは
  含まれません。

## エクスポーターを使用しない場合

`diagnostics-otel` を実行せずに、診断イベントを Plugin またはカスタムシンクで
利用できるようにします。

```json5
{
  diagnostics: { enabled: true },
}
```

`logging.level` を引き上げずに対象を絞ったデバッグ出力を行うには、診断
フラグを使用します。フラグでは大文字と小文字が区別されず、ワイルドカード
（`telegram.*` または `*`）がサポートされます。

```json5
{
  diagnostics: { flags: ["telegram.http"] },
}
```

または、1 回限りの環境変数オーバーライドとして指定します。

```bash
OPENCLAW_DIAGNOSTICS=telegram.http,telegram.payload openclaw gateway
```

フラグの出力先は標準ログファイル（`logging.file`）であり、引き続き
`logging.redactSensitive` によって秘匿化されます。完全なガイド：
[診断フラグ](/ja-JP/diagnostics/flags)。

## 無効化

```json5
{
  diagnostics: { otel: { enabled: false } },
}
```

または、`plugins.allow` から `diagnostics-otel` を省略するか、
`openclaw plugins disable diagnostics-otel` を実行します。

## 関連項目

- [ロギング](/ja-JP/logging) - ファイルログ、コンソール出力、CLI の追尾、Control UI の Logs タブ
- [Gateway ロギングの内部構造](/ja-JP/gateway/logging) - WS ログのスタイル、サブシステムのプレフィックス、コンソールキャプチャ
- [診断フラグ](/ja-JP/diagnostics/flags) - 対象を絞ったデバッグログ用フラグ
- [診断エクスポート](/ja-JP/gateway/diagnostics) - オペレーター向けサポートバンドルツール（OTEL エクスポートとは別）
- [設定リファレンス](/ja-JP/gateway/configuration-reference#diagnostics) - `diagnostics.*` フィールドの完全なリファレンス
