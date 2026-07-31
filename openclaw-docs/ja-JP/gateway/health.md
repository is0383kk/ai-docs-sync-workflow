---
read_when:
    - チャネル接続または Gateway の正常性の診断
    - ヘルスチェック CLI コマンドとオプションを理解する
summary: ヘルスチェックコマンドと Gateway のヘルス監視
title: ヘルスチェック
x-i18n:
    generated_at: "2026-07-26T09:35:21Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 59a7fbfb7fb86be7dbd3a03f96c7328c2bc8cc851230c0bdd1b1b750b3014be4
    source_path: gateway/health.md
    workflow: 16
---

推測せずにチャネル接続を確認するための短いガイドです。

## クイックチェック

- `openclaw status` - ローカル概要：Gateway の到達可能性／モード、更新のヒント、リンク済みチャネル認証の経過時間、セッションと最近のアクティビティ。
- `openclaw status --all` - 完全なローカル診断（読み取り専用、カラー表示、デバッグ用に安全に貼り付け可能）。
- `openclaw status --deep` - 実行中の Gateway にライブプローブを要求します（`probe:true` を指定した `health`）。対応している場合は、アカウントごとのチャネルプローブも含まれます。
- `openclaw status --usage` - モデルプロバイダーの使用量／クォータのスナップショットを表示します。
- `openclaw health` - 実行中の Gateway にヘルススナップショットを要求します（WS のみ。CLI からチャネルソケットへ直接接続しません）。
- `openclaw health --verbose`（別名 `--debug`）- ライブヘルスプローブを強制し、Gateway の接続詳細を表示します。
- `openclaw health --json` - 機械可読なヘルススナップショットを出力します。
- エージェントを呼び出さずにステータス応答を取得するには、任意のチャネルで `/status` を単独のチャットコマンドとして送信します。
- ログ：`openclaw logs --follow`（または `openclaw --profile <profile> logs --follow`）を実行し、`web-heartbeat`、`web-reconnect`、`web-auto-reply`、`web-inbound` でフィルタリングします。

Discord などのチャットプロバイダーでは、セッション行はソケットの稼働状態を示すものではありません。
`openclaw sessions`、Gateway の `sessions.list`、およびエージェントの `sessions_list` ツールは、
保存された会話状態を読み取ります。プロバイダーは再接続後、新しいセッション行が作成される前でも、
チャネルステータスが正常と表示される場合があります。ライブ接続の確認には、上記のチャネルステータス
およびヘルスコマンドを使用してください。

## 詳細診断

- ディスク上の認証情報：`ls -l ~/.openclaw/credentials/whatsapp/<accountId>/creds.json`（mtime は最近の値である必要があります）。
- セッションストア：`ls -l ~/.openclaw/agents/<agentId>/agent/openclaw-agent.sqlite`。件数と最近の受信者は `status` で確認できます。
- 再リンク手順：ステータスコード 409-515 または `loggedOut` がログに表示された場合は、`openclaw channels logout && openclaw channels login --verbose` を実行します。QR ログインフローは、ペアリング後にステータス 515 が発生すると一度だけ自動再起動します。
- 診断はデフォルトで有効です（`diagnostics.enabled: false` で無効化）。メモリイベントには、RSS／ヒープのバイト数と、しきい値／増加による圧迫が記録されます。稼働警告には、プロセスが実行中でも飽和している場合に、イベントループの遅延／使用率、CPU コア比率、アクティブ／待機中／キュー内のセッション数が記録されます。過大なペイロードのイベントには、拒否／切り詰め／分割された内容の種類と、そのサイズおよび上限が記録されますが、メッセージ本文、添付ファイルの内容、Webhook 本文、生のリクエスト／レスポンス本文、トークン、Cookie、シークレット値は記録されません。
- 同じ Heartbeat が、上限付きの安定性レコーダーも駆動します：`openclaw gateway stability`（または `diagnostics.stability` Gateway RPC）。Gateway の致命的な終了、シャットダウンのタイムアウト、再起動時の起動失敗が発生すると、最新のスナップショットが `~/.openclaw/logs/stability/` に保存されます。最新のバンドルは `openclaw gateway stability --bundle latest` で確認します。
- バグ報告では、`openclaw gateway diagnostics export` を実行し、生成された zip を添付してください。zip には Markdown の概要、最新の安定性バンドル、サニタイズ済みのログメタデータ、サニタイズ済みの Gateway ステータス／ヘルススナップショット、設定の構造が含まれます。チャット本文、Webhook 本文、ツール出力、認証情報、Cookie、アカウント／メッセージ識別子、シークレット値は除外または編集されます。[診断のエクスポート](/ja-JP/gateway/diagnostics)を参照してください。

## ヘルスモニターの設定

- `channels.<provider>.healthMonitor.enabled`：グローバル監視を有効のままにして、特定のチャネルに対するヘルスモニターの再起動を無効化します。
- `channels.<provider>.accounts.<accountId>.healthMonitor.enabled`：チャネルレベルの設定より優先される、複数アカウント用のオーバーライドです。
- これらのチャネル別オーバーライドは、現在この機能を公開している組み込みチャネル（Discord、Google Chat、iMessage、IRC、Microsoft Teams、Signal、Slack、Telegram、WhatsApp）に適用されます。

## 稼働時間監視

外部の稼働時間監視サービスでは、`/v1/chat/completions` ではなく専用の `/health` エンドポイントを使用してください。

- **使用するもの：** `GET /health` - 即時に応答し、セッションを作成せず、LLM を呼び出さず、`{"ok":true,"status":"live"}` を返します
- **使用しないもの：** ヘルスチェックに `/v1/chat/completions` を使用しないでください。リクエストごとに、Skills のスナップショット、コンテキストの組み立て、LLM 呼び出しを伴う完全なエージェントセッションが作成されます

`x-openclaw-session-key` ヘッダーも `user` フィールドも指定されていない場合、`/v1/chat/completions` はリクエストごとに新しいランダムなセッションを生成します。15 分ごとに ping する監視サービスでは、1 日あたり約 96 セッションが作成され、それぞれ 4-22KB を消費します。時間の経過とともにセッションストアが肥大化し、コンテキストウィンドウのオーバーフローにつながる可能性があります。

### 監視サービスの設定例

- **BetterStack：** ヘルスチェック URL を `https://<your-gateway-host>:<port>/health` に設定します
- **UptimeRobot：** URL に `https://<your-gateway-host>:<port>/health` を指定した新しい HTTP モニターを追加します
- **汎用：** Gateway が正常な場合、`/health` への任意の HTTP GET は `{"ok":true}` とともに 200 を返します

## 問題が発生した場合

- `logged out` またはステータス 409-515 -> `openclaw channels logout`、続いて `openclaw channels login` を実行して再リンクします。
- Gateway に到達できない -> `openclaw gateway --port 18789` で起動します（ポートが使用中の場合は `--force` を使用します）。
- 受信メッセージがない -> リンク済みの電話がオンラインで、送信者が許可されていることを確認します（`channels.whatsapp.allowFrom`）。グループチャットでは、許可リストとメンションのルールが一致していることを確認します（`channels.whatsapp.groups`、`agents.entries.*.groupChat.mentionPatterns`）。

## 専用の「health」コマンド

`openclaw health` は、実行中の Gateway にヘルススナップショットを要求します（CLI から
チャネルソケットへ直接接続しません）。デフォルトでは、キャッシュされた新しい Gateway スナップショットを返し、
Gateway はバックグラウンドでそのキャッシュを更新します。代わりに `--verbose` を指定すると、ライブプローブを強制します。
このコマンドは、利用可能な場合はリンク済みの認証情報／認証の経過時間、チャネルごとのプローブ概要、
セッションストアの概要、プローブ時間を報告します。Gateway に到達できない場合、またはプローブが失敗／タイムアウトした場合は、
ゼロ以外の終了コードで終了します。

オプション：

- `--json`：機械可読な JSON 出力
- `--timeout <ms>`：デフォルトの 10s プローブタイムアウトを上書き
- `--verbose`：ライブプローブを強制し、Gateway の接続詳細を表示
- `--debug`：`--verbose` の別名

ヘルススナップショットには、`ok`（真偽値）、`ts`（タイムスタンプ）、`durationMs`（プローブ時間）、チャネルごとのステータス、エージェントの可用性、セッションストアの概要が含まれます。

## 関連項目

- [Gateway 運用手順書](/ja-JP/gateway)
- [診断のエクスポート](/ja-JP/gateway/diagnostics)
- [Gateway のトラブルシューティング](/ja-JP/gateway/troubleshooting)
