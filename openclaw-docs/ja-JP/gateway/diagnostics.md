---
read_when:
    - バグ報告またはサポートリクエストの準備
    - Gateway のクラッシュ、再起動、メモリ負荷、または過大なペイロードのデバッグ
    - 記録または編集される診断データの確認
summary: バグ報告用に共有可能な Gateway 診断バンドルを作成する
title: 診断情報のエクスポート
x-i18n:
    generated_at: "2026-07-26T09:41:39Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 97a805fed8d51de2e63e5c6a12ce03e91701d69654882cca7795c9f3553b1c55
    source_path: gateway/diagnostics.md
    workflow: 16
---

OpenClaw は、バグ報告用のローカル診断 `.zip` を作成できます。これには、サニタイズ済みの Gateway
ステータス、ヘルス情報、ログ、設定構造、およびペイロードを含まない最近の安定性イベントが含まれます。

診断バンドルは、レビューするまではシークレットとして扱ってください。ペイロードと認証情報は
設計上編集されますが、バンドルにはローカル Gateway ログと
ホストレベルのランタイム状態の概要が含まれます。

## クイックスタート

```bash
openclaw gateway diagnostics export
```

書き込まれた zip のパスを出力します。出力パスを指定するには、次のようにします。

```bash
openclaw gateway diagnostics export --output openclaw-diagnostics.zip
```

自動化する場合は、次のようにします。

```bash
openclaw gateway diagnostics export --json
```

## チャットコマンド

オーナーは任意の会話で `/diagnostics [note]` を実行し、コピー＆ペースト可能な単一のサポートレポートとしてローカル
Gateway エクスポートを要求できます。

1. `/diagnostics` を送信します。必要に応じて短いメモを付けられます（`/diagnostics bad tool choice`）。
2. OpenClaw は前置きメッセージを送信し、1 回の明示的な exec 承認を求めます。承認すると
   `openclaw gateway diagnostics export --json` が実行されます。診断を
   すべて許可するルールで承認しないでください。
3. 承認後、OpenClaw はローカルバンドルのパス、マニフェストの
   概要、プライバシーに関する注記、および関連するセッション ID を返信します。

グループチャットでもオーナーは `/diagnostics` を実行できますが、OpenClaw は
エクスポート結果、承認プロンプト、Codex のセッション／スレッド内訳を
オーナーに非公開で送信します。グループには、診断が非公開で送信されたことを示す
短い通知だけが表示されます。オーナーへの非公開ルートが存在しない場合、コマンドは安全側に倒して失敗し、
DM から実行するようオーナーに求めます。

アクティブなセッションでネイティブ OpenAI Codex ハーネスを使用している場合、同じ exec
承認によって、OpenClaw が認識している Codex スレッドの OpenAI フィードバックアップロードも
承認されます。このアップロードはローカル Gateway zip とは別であり、Codex ハーネスセッションでのみ
行われます。承認プロンプトには、承認すると
Codex フィードバックも送信されることが記載されますが、Codex のセッション ID やスレッド ID は表示されません。承認後、
返信には、OpenAI に送信されたスレッドのチャンネル、OpenClaw セッション ID、Codex スレッド ID、および
ローカル再開コマンドが一覧表示されます。承認を拒否または無視すると、
エクスポート、Codex フィードバックのアップロード、Codex ID リストはすべてスキップされます。

これにより Codex のデバッグループが短くなります。チャンネルで不適切な動作に気づいたら、
`/diagnostics` を実行し、一度承認してレポートを共有します。その後、自分でスレッドを確認する場合は、
出力された `codex resume <thread-id>` コマンドをローカルで実行します。
[Codex ハーネス](/ja-JP/plugins/codex-harness#inspect-codex-threads-locally)を参照してください。

## エクスポートの内容

- `summary.md`: サポート向けの人間が読める概要。
- `diagnostics.json`: 設定、ログ、ステータス、ヘルス情報、
  安定性データの機械可読な概要。
- `manifest.json`: エクスポートのメタデータとファイル一覧。
- サニタイズ済みの設定構造と、シークレットではない設定の詳細。
- サニタイズ済みのログ概要と、編集済みの最近のログ行。
- ベストエフォートの Gateway ステータスおよびヘルススナップショット。
- `stability/latest.json`: 利用可能な場合、永続化された最新の安定性バンドル。

Gateway が異常な状態でも、エクスポートは役立ちます。ステータス／ヘルス
リクエストが失敗しても、ローカルログ、設定構造、および最新の安定性バンドルは、
利用可能な場合に引き続き収集されます。

## プライバシーモデル

保持されるもの: サブシステム名、Plugin ID、プロバイダー ID、チャンネル ID、設定済みの
モード、ステータスコード、継続時間、バイト数、キュー状態、メモリ測定値、
サニタイズ済みのログメタデータ、編集済みの運用メッセージ、設定構造、および
シークレットではない機能設定。

省略または編集されるもの: チャットテキスト、プロンプト、指示、Webhook 本文、ツール
出力、認証情報、API キー、トークン、Cookie、シークレット値、生の
リクエスト／レスポンス本文、アカウント ID、メッセージ ID、生のセッション ID、
ホスト名、およびローカルユーザー名。

ログメッセージがユーザー、チャット、プロンプト、またはツールのペイロードテキストに見える場合、
エクスポートには、メッセージが省略されたという情報とそのバイト数のみが保持されます。

## 安定性レコーダー

診断が有効な場合、Gateway はデフォルトで、サイズが制限されたペイロードなしの安定性ストリームを
記録します。記録するのは運用上の事実であり、コンテンツではありません。

同じ Heartbeat は、イベントループまたは CPU が飽和しているように見える場合にもライブネスを
サンプリングし、イベントループ遅延、イベントループ使用率、CPU コア比率、
アクティブ／待機中／キュー待ちのセッション数、現在の起動／ランタイムフェーズ（判明している場合）、
最近のフェーズスパン、およびサイズが制限された作業ラベルを含む `diagnostic.liveness.warning` イベントを
生成します。これらは、作業が待機中またはキュー待ちの場合、あるいはアクティブな作業が
持続的なイベントループ遅延と重なっている場合にのみ、Gateway の `warn` レベルのログ行になります。
それ以外の場合は `debug` で記録されます。アイドル時のライブネスサンプルも診断イベントとして記録されますが、
それ自体で警告に昇格することはありません。

起動フェーズでは、実時間と
CPU タイミングを含む `diagnostic.phase.completed` イベントが生成されます。停止している埋め込み実行の診断では、
最後のブリッジ進行状況が終了状態に見える場合（たとえば、生のレスポンス
項目やレスポンス完了イベント）でも、Gateway が埋め込み実行を
アクティブと見なしているときに `terminalProgressStale=true` を記録します。

ライブレコーダーを確認するには、次のようにします。

```bash
openclaw gateway stability
openclaw gateway stability --type payload.large
openclaw gateway stability --json
```

致命的終了、シャットダウンタイムアウト、または
再起動時の起動失敗後に、永続化された最新のバンドルを確認するには、次のようにします。

```bash
openclaw gateway stability --bundle latest
```

永続化された最新のバンドルから診断 zip を作成するには、次のようにします。

```bash
openclaw gateway stability --bundle latest --export
```

イベントが存在する場合、永続化されたバンドルは `~/.openclaw/logs/stability/` 配下に保存されます。

## 便利なオプション

```bash
openclaw gateway diagnostics export \
  --output openclaw-diagnostics.zip \
  --log-lines 5000 \
  --log-bytes 1000000
```

| フラグ                    | デフォルト                                                                       | 説明                                        |
| ----------------------- | ----------------------------------------------------------------------------- | -------------------------------------------------- |
| `--output <path>`       | `$OPENCLAW_STATE_DIR/logs/support/openclaw-diagnostics-<timestamp>-<pid>.zip` | 特定の zip パス（またはディレクトリ）に書き込みます。       |
| `--log-lines <count>`   | `5000`                                                                        | 含めるサニタイズ済みログ行の最大数。            |
| `--log-bytes <bytes>`   | `1000000`                                                                     | 検査するログの最大バイト数。                      |
| `--url <url>`           | -                                                                             | ステータス／ヘルススナップショット用の Gateway WebSocket URL。 |
| `--token <token>`       | -                                                                             | ステータス／ヘルススナップショット用の Gateway トークン。         |
| `--password <password>` | -                                                                             | ステータス／ヘルススナップショット用の Gateway パスワード。      |
| `--timeout <ms>`        | `3000`                                                                        | ステータス／ヘルススナップショットのタイムアウト。                    |
| `--no-stability-bundle` | オフ                                                                           | 永続化された安定性バンドルの検索をスキップします。            |
| `--json`                | オフ                                                                           | 機械可読なエクスポートメタデータを出力します。            |

## 診断を無効にする

診断はデフォルトで有効です。安定性レコーダーと
診断イベントの収集を無効にするには、次のようにします。

```json5
{
  diagnostics: {
    enabled: false,
  },
}
```

診断を無効にするとバグ報告の詳細が減りますが、通常の
Gateway ロギングには影響しません。

メモリ負荷イベントは、ファイルシステムのスキャンや OOM 前スナップショットの書き込みを行わずに、
RSS、ヒープ、しきい値、および増加に関する情報
（`rss_threshold`、`heap_threshold`、`rss_growth`）を記録します。

## 関連項目

- [ヘルスチェック](/ja-JP/gateway/health)
- [Gateway CLI](/ja-JP/cli/gateway#gateway-diagnostics-export)
- [Gateway プロトコル](/ja-JP/gateway/protocol#rpc-method-families)
- [ロギング](/ja-JP/logging)
- [OpenTelemetry エクスポート](/ja-JP/gateway/opentelemetry) - 診断をコレクターへストリーミングするための別のフロー
