---
read_when:
    - OpenClaw のログについて初心者向けの概要が必要です
    - ログレベル、形式、またはマスキングを設定したい場合
    - トラブルシューティング中で、ログをすばやく見つける必要がある場合
summary: ファイルログ、コンソール出力、CLI での追跡、および Control UI の Logs タブ
title: ロギング
x-i18n:
    generated_at: "2026-07-26T09:39:09Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 2c9303c4bc9c0797ca9c5775a281dce95229661b61d710425b2f7bec182b2e75
    source_path: logging.md
    workflow: 16
---

OpenClaw には、主に 2 つのログ出力先があります。

- Gateway によって書き込まれる **ファイルログ**（JSON Lines）。
- Gateway を実行しているターミナルの **コンソール出力**。

Control UI の **ログ** タブは、Gateway のファイルログを追尾します。このページでは、ログの保存場所、読み方、ログレベルと形式の設定方法について説明します。

## ログの保存場所

デフォルトでは、Gateway は日ごとにローテーションするログファイルを書き込みます。デフォルトプロファイルでは、従来のパスが維持されます。

`/tmp/openclaw/openclaw-YYYY-MM-DD.log`

名前付きプロファイルでは、同じディレクトリ内でプロファイル名を含むファイル名を使用します。

`/tmp/openclaw/openclaw-<profile>-YYYY-MM-DD.log`

ファイル名のプロファイル部分は小文字で、英字、数字、ダッシュのみに制限されます。単純な小文字の名前は読みやすいまま維持されるため、`--dev` という短縮指定では `openclaw-dev-YYYY-MM-DD.log` に書き込まれます。大文字と小文字、アンダースコア、リテラルのダッシュには可逆的なダッシュエスケープが使用されるため、異なるプロファイル名が同じログファイルを共有することはありません。環境を通じて直接設定された過大な値には、ファイルシステムのファイル名制限内に収めるため、長さを制限したハッシュサフィックスが使用されます。明示的な `logging.file` は、これらのデフォルトを上書きします。

日付には、Gateway ホストのローカルタイムゾーンが使用されます。`/tmp/openclaw` が安全でない、または利用できない場合（Windows では常に）、OpenClaw は代わりに OS の一時ディレクトリ配下にあるユーザー単位の `openclaw-<uid>` ディレクトリを使用します。日付付きログファイルは 24 時間後に削除されます。

各ファイルは、次の書き込みによって `logging.maxFileBytes`（デフォルト: 100 MB）を超える場合にローテーションされます。OpenClaw は、`openclaw-YYYY-MM-DD.1.log` や `openclaw-dev-YYYY-MM-DD.1.log` など、番号付きのアーカイブをアクティブファイルの隣に最大 5 個保持し、診断情報の出力を抑制する代わりに、新しいアクティブログへの書き込みを継続します。

`~/.openclaw/openclaw.json` でパスを上書きできます。

```json
{
  "logging": {
    "file": "/path/to/openclaw.log"
  }
}
```

## ログの読み方

### CLI: ライブ追尾（推奨）

RPC 経由で Gateway のログファイルを追尾します。

```bash
openclaw logs --follow
openclaw --dev logs --follow
openclaw --profile work logs --follow
```

ルートのプロファイルセレクターは、Gateway が使用するものと同じプロファイル固有のファイルを解決します。これには、ローカル RPC が利用できない場合の CLI によるフォールバック読み取りも含まれます。

オプション:

| フラグ                | デフォルト  | 動作                                                                              |
| ------------------- | -------- | ------------------------------------------------------------------------------------- |
| `--follow`          | オフ      | 追尾を継続し、切断時はバックオフ付きで再接続する                                   |
| `--limit <n>`       | `200`    | 1 回の取得あたりの最大行数                                                                   |
| `--max-bytes <n>`   | `250000` | 1 回の取得あたりに読み取る最大バイト数                                                           |
| `--interval <ms>`   | `1000`   | 追尾中のポーリング間隔                                                         |
| `--json`            | オフ      | 行区切り JSON（1 行につき 1 イベント）                                              |
| `--plain`           | オフ      | TTY セッションでプレーンテキストを強制する                                                      |
| `--no-color`        | —        | ANSI カラーを無効にする                                                                   |
| `--utc`             | オフ      | タイムスタンプを UTC で表示する（デフォルトはローカル時刻）                                      |
| `--local-time`      | オフ      | ローカル時刻をデフォルトとする指定との互換性のために受け付ける表記。それ以外の効果はない       |
| `--url` / `--token` | —        | 標準の Gateway RPC フラグ                                                            |
| `--timeout <ms>`    | `30000`  | Gateway RPC のタイムアウト                                                                   |
| `--expect-final`    | オフ      | エージェントを利用する RPC の最終応答待機フラグ（共有クライアントレイヤーを通じてここでも受け付ける） |

出力モード:

- **TTY セッション**: 見やすく色分けされた構造化ログ行。
- **非 TTY セッション**: プレーンテキスト。

明示的な `--url` を渡した場合、CLI は設定や環境の認証情報を自動適用しません。`--token` を明示的に含めないと、呼び出しは `gateway url override requires explicit credentials` で失敗します。

JSON モードでは、CLI は `type` タグ付きオブジェクトを出力します。

- `meta`: ストリームのメタデータ（file、source、sourceKind、service、cursor、size）
- `log`: 解析済みログエントリ
- `notice`: 切り詰め／ローテーションのヒント
- `raw`: 未解析のログ行
- `error`: Gateway 接続エラー（stderr に書き込まれる）

暗黙的な local loopback Gateway がペアリングを要求した場合、接続中に閉じた場合、または `logs.tail` が応答する前にタイムアウトした場合、`openclaw logs` は設定済みの Gateway ファイルログへ自動的にフォールバックします。明示的な `--url` ターゲットでは、このフォールバックを使用しません。`openclaw logs --follow` はより厳格です。Linux では、利用可能な場合は PID に基づいてアクティブなユーザー systemd Gateway ジャーナルを使用し、それ以外の場合は、古くなっている可能性がある併置ファイルを追尾せず、バックオフ付きで稼働中の Gateway への接続を再試行します。

Gateway に接続できない場合、CLI は次のコマンドを実行するよう短いヒントを表示します。

```bash
openclaw doctor
```

### Control UI（Web）

Control UI の **ログ** タブは、`logs.tail` を使用して同じファイルを追尾します。開き方については、[Control UI](/ja-JP/web/control-ui) を参照してください。

### チャンネル専用ログ

チャンネルのアクティビティ（WhatsApp、Telegram など）を絞り込むには、次を使用します。

```bash
openclaw channels logs --channel whatsapp
```

`--channel` のデフォルトは `all` です。`--lines <n>`（デフォルト 200）と `--json` も利用できます。

## ログ形式

### ファイルログ（JSONL）

ログファイルの各行は JSON オブジェクトです。CLI と Control UI はこれらのエントリを解析し、構造化された出力（時刻、レベル、サブシステム、メッセージ）として表示します。

ファイルログの JSONL レコードには、利用可能な場合、機械的なフィルタリングが可能なトップレベルフィールドも含まれます。

- `hostname`: Gateway のホスト名。
- `message`: 全文検索用に平坦化されたログメッセージテキスト。
- `agent_id`: ログ呼び出しにエージェントコンテキストが含まれる場合のアクティブなエージェント ID。
- `session_id`: ログ呼び出しにセッションコンテキストが含まれる場合のアクティブなセッション ID／キー。
- `channel`: ログ呼び出しにチャンネルコンテキストが含まれる場合のアクティブなチャンネル。

OpenClaw は、これらのフィールドとともに元の構造化ログ引数も保持するため、番号付きの tslog 引数キーを読み取る既存のパーサーは引き続き動作します。

Talk、リアルタイム音声、管理対象ルームのアクティビティは、同じファイルログパイプラインを通じて、サイズを制限したライフサイクルログレコードを出力します。これらのレコードには、利用可能な場合、イベントタイプ、モード、トランスポート、プロバイダー、サイズ／タイミング測定値が含まれますが、文字起こしテキスト、音声ペイロード、ターン ID、通話 ID、プロバイダー項目 ID は含まれません。

### コンソール出力

コンソールログは **TTY を認識**し、読みやすい形式で出力されます。

- サブシステムのプレフィックス（例: `gateway/channels/whatsapp`）
- レベルごとの色分け（info／warn／error）
- オプションのコンパクトモードまたは JSON モード

コンソールの形式は `logging.consoleStyle` で制御します。

### Gateway WebSocket ログ

`openclaw gateway` には、RPC トラフィック用の WebSocket プロトコルログもあります。

- 通常モード: 注目すべき結果のみ（エラー、解析エラー、低速な呼び出し）
- `--verbose`: すべてのリクエスト／レスポンストラフィック
- `--ws-log auto|compact|full`: 詳細表示の形式を選択する
- `--compact`: `--ws-log compact` のエイリアス

例:

```bash
openclaw gateway
openclaw gateway --verbose --ws-log compact
openclaw gateway --verbose --ws-log full
```

## ログの設定

すべてのログ設定は、`~/.openclaw/openclaw.json` の `logging` 配下にあります。

```json
{
  "logging": {
    "level": "info",
    "file": "/path/to/openclaw.log",
    "consoleLevel": "info",
    "consoleStyle": "pretty",
    "redactSensitive": "tools",
    "redactPatterns": ["sk-.*"]
  }
}
```

### ログレベル

レベル: `silent`、`fatal`、`error`、`warn`、`info`、`debug`、`trace`。

- `logging.level`: **ファイルログ**（JSONL）のレベル（デフォルト: `info`）。
- `logging.consoleLevel`: **コンソール**の詳細度レベル。

どちらも **`OPENCLAW_LOG_LEVEL`** 環境変数（例: `OPENCLAW_LOG_LEVEL=debug`）で上書きできます。この環境変数は設定ファイルより優先されるため、`openclaw.json` を編集せず、1 回の実行だけ詳細度を上げることができます。グローバル CLI オプション **`--log-level <level>`**（例: `openclaw --log-level debug gateway run`）を渡すこともでき、そのコマンドでは環境変数より優先されます。

`--verbose` はコンソール出力と WS ログの詳細度にのみ影響し、ファイルログのレベルは変更しません。

### モデルトランスポートを対象とした診断

プロバイダー呼び出しをデバッグする場合は、すべてのログを `debug` に引き上げる代わりに、対象を限定した環境フラグを使用します。

```bash
OPENCLAW_DEBUG_MODEL_TRANSPORT=1 openclaw gateway
OPENCLAW_DEBUG_MODEL_PAYLOAD=tools OPENCLAW_DEBUG_SSE=events openclaw gateway
```

利用可能なフラグ:

- `OPENCLAW_DEBUG_MODEL_TRANSPORT=1`: リクエスト開始、fetch レスポンス、SDK
  ヘッダー、最初のストリーミングイベント、ストリーム完了、トランスポートエラーを
  `info` レベルで出力する。
- `OPENCLAW_DEBUG_MODEL_PAYLOAD=summary`: モデルリクエストログに、サイズを制限したリクエストペイロードの
  要約を含める。
- `OPENCLAW_DEBUG_MODEL_PAYLOAD=tools`: ペイロードの要約に、モデル向けのすべてのツール名を含める。
- `OPENCLAW_DEBUG_MODEL_PAYLOAD=full-redacted`: 編集済みでサイズ上限のある JSON
  ペイロードのスナップショットを含める。デバッグ時にのみ使用してください。シークレットは編集されますが、プロンプトや
  メッセージテキストが含まれる場合があります。
- `OPENCLAW_DEBUG_SSE=events`: 最初のイベントとストリーム完了のタイミングを出力する。
- `OPENCLAW_DEBUG_SSE=peek`: 編集済みの SSE イベントペイロードを最初の 5 件についても出力し、
  各イベントのサイズを制限する。
- `OPENCLAW_DEBUG_CODE_MODE=1`: コードモードがツールサーフェスを所有するために
  ネイティブプロバイダーツールが非表示になる場合など、コードモードのモデルサーフェス診断を出力する。

これらのフラグは通常の OpenClaw ログを通じて記録されるため、`openclaw logs --follow` と Control UI のログタブに表示されます。フラグを指定しない場合でも、同じ診断は `debug` レベルで引き続き利用できます。

`[model-fetch]` の開始およびレスポンスのメタデータ（プロバイダー、API、モデル、ステータス、レイテンシー、およびメソッド、URL、タイムアウト、プロキシ、ポリシーなどのリクエストフィールド）は、`OPENCLAW_DEBUG_MODEL_TRANSPORT` に関係なく、常に `info` レベルで出力されます。そのため、デバッグフラグなしでも基本的なモデルトランスポートの健全性を確認できます。

### トレースの関連付け

ファイルログは JSONL 形式です。ログ呼び出しに有効な診断トレースコンテキストが含まれる場合、OpenClaw はトレースフィールドをトップレベルの JSON キー（`traceId`、`spanId`、`parentSpanId`、`traceFlags`）として書き込みます。これにより、外部ログプロセッサーは、その行を OTEL スパンおよびプロバイダーの `traceparent` 伝播と関連付けられます。

Gateway HTTP リクエストと Gateway WebSocket フレームは、内部リクエストトレーススコープを確立します。その非同期スコープ内で出力されるログと診断イベントは、明示的なトレースコンテキストを渡さない場合、リクエストトレースを継承します。エージェント実行とモデル呼び出しのトレースは、アクティブなリクエストトレースの子になります。そのため、ローカルログ、診断スナップショット、OTEL スパン、信頼されたプロバイダーの `traceparent` ヘッダーを、未加工のリクエスト内容やモデル内容をログに記録することなく、`traceId` で結合できます。

OpenTelemetry のログエクスポートが有効な場合、Talk のライフサイクルログレコードも diagnostics-otel のログエクスポートに送られ、ファイルログと同じサイズ制限付き属性が使用されます。`diagnostics.otel.logsExporter` を設定して、OTLP、stdout JSONL、または両方の出力先を選択します。

### モデル呼び出しのサイズとタイミング

モデル呼び出しの診断では、未加工のプロンプトやレスポンス内容を取得することなく、サイズを制限したリクエスト／レスポンスの測定値を記録します。

- `requestPayloadBytes`: 最終的なモデルリクエストペイロードの UTF-8 バイトサイズ
- `responseStreamBytes`: ストリーミングされたモデルレスポンスチャンクの UTF-8 バイトサイズ
  ペイロード。高頻度のテキスト、思考、ツール呼び出しの差分イベントでは、
  完全な `partial` スナップショットではなく、増分の `delta` バイトのみがカウントされます。
- `timeToFirstByteMs`: 最初のストリーミングレスポンスイベントまでの経過時間
- `durationMs`: モデル呼び出しの合計所要時間

これらのフィールドは、診断エクスポートが有効な場合、診断スナップショット、モデル呼び出しの Plugin フック、
および OTEL モデル呼び出しのスパン／メトリクスで利用できます。

### コンソールスタイル

`logging.consoleStyle`:

- `pretty`: タイムスタンプ付きで、人が読みやすいカラー表示。
- `compact`: より簡潔な出力（長時間のセッションに最適）。
- `json`: 1 行ごとの JSON（ログプロセッサ向け）。

### マスキング

OpenClaw は、機密トークンがコンソール出力、ファイルログ、
OTLP ログレコード、永続化されたセッショントランスクリプトのテキスト、または Control UI のツール
イベントペイロード（ツール開始引数、部分的／最終的な結果ペイロード、派生した
実行出力、パッチの概要）に到達する前にマスキングできます。

- 機密値のマスキングは常に有効です。
- `logging.redactPatterns`: ログ／トランスクリプト出力用のデフォルトセットを置き換える正規表現文字列のリスト。Control UI のツールペイロードでは、カスタムパターンが組み込みのデフォルトに追加して適用されるため、パターンを追加しても、デフォルトですでに検出される値のマスキングが弱まることはありません。

ファイルログとセッショントランスクリプトは JSONL のままですが、一致するシークレット値は、
行またはメッセージがディスクに書き込まれる前にマスクされます。マスキングはベストエフォートです。
テキストを含むメッセージコンテンツとログ文字列には適用されますが、すべての
識別子やバイナリペイロードフィールドに適用されるわけではありません。

組み込みのデフォルトは、一般的な API 認証情報、およびカード番号、CVC/CVV、
共有決済トークン、決済認証情報などの決済認証情報フィールド名が、
JSON フィールド、URL パラメータ、CLI フラグ、または代入として現れる場合に対応します。

OpenClaw は、UI クライアント、サポートバンドル、診断オブザーバー、承認プロンプト、
またはエージェントツールに表示される安全境界ペイロードもマスキングします。カスタム
`logging.redactPatterns` を使用すると、これらのサーフェスにプロジェクト固有のパターンを追加できます。

## 診断と OpenTelemetry

診断は、モデル実行およびメッセージフローのテレメトリ（Webhook、キューイング、セッション状態）のための、
構造化された機械可読イベントです。ログを**置き換えるものではなく**、
メトリクス、トレース、エクスポーターに供給されます。イベントはデフォルトでプロセス内に発行されます
（無効にするには `diagnostics.enabled: false` を設定します）。
イベントのエクスポートは別途行います。

隣接する 2 つのサーフェスがあります。

- **OpenTelemetry エクスポート** — OTLP/HTTP を介して、メトリクス、トレース、ログを
  OpenTelemetry 互換の任意のコレクターまたはバックエンド（Datadog、Grafana、
  Honeycomb、New Relic、Tempo など）に送信します。完全な設定、シグナルカタログ、
  メトリクス／スパン名、環境変数、プライバシーモデルについては専用ページを参照してください。
  [OpenTelemetry エクスポート](/ja-JP/gateway/opentelemetry)。
- **診断フラグ** — `logging.level` を引き上げることなく、追加のログを
  `logging.file` に送る、対象を絞ったデバッグログフラグです。フラグでは大文字と小文字が区別されず、
  ワイルドカード（`telegram.*`、`*`）がサポートされます。`diagnostics.flags` で設定するか、
  `OPENCLAW_DIAGNOSTICS=...` 環境変数による上書きを使用します。完全なガイド：
  [診断フラグ](/ja-JP/diagnostics/flags)。

コレクターへの OTLP エクスポートについては、[OpenTelemetry エクスポート](/ja-JP/gateway/opentelemetry)を参照してください。

## トラブルシューティングのヒント

- **Gateway に到達できない場合** まず `openclaw doctor` を実行してください。
- **ログが空の場合** Gateway が実行中で、`logging.file` にあるファイルパスへ
  書き込んでいることを確認してください。
- **さらに詳細が必要な場合** `logging.level` を `debug` または `trace` に設定して、再試行してください。

## 関連項目

- [OpenTelemetry エクスポート](/ja-JP/gateway/opentelemetry) — OTLP/HTTP エクスポート、メトリクス／スパンカタログ、プライバシーモデル
- [診断フラグ](/ja-JP/diagnostics/flags) — 対象を絞ったデバッグログフラグ
- [Gateway ロギングの内部構造](/ja-JP/gateway/logging) — WS ログスタイル、サブシステムのプレフィックス、コンソールキャプチャ
- [設定リファレンス](/ja-JP/gateway/configuration-reference#diagnostics) — `diagnostics.*` フィールドの完全なリファレンス
