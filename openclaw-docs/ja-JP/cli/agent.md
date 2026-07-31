---
read_when:
    - スクリプトからエージェントのターンを1回実行する（必要に応じて返信を配信する）
summary: '`openclaw agent` の CLI リファレンス（Gateway 経由でエージェントの 1 ターンを送信）'
title: エージェント
x-i18n:
    generated_at: "2026-07-26T10:07:51Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 1a4c139a3b235d6a56ba63063737b80f93448c2dbb7a92c6d0756fb19a9f95e4
    source_path: cli/agent.md
    workflow: 16
---

# `openclaw agent`

Gateway を介してエージェントのターンを 1 回実行します。明示的な `--local` フラグが、唯一の組み込み実行パスです。

セッションセレクターとして、`--to`、`--session-key`、`--session-id`、または `--agent` のうち少なくとも 1 つを渡します。

関連項目：[エージェント送信ツール](/ja-JP/tools/agent-send)

## オプション

- `-m, --message <text>`：メッセージ本文
- `--message-file <path>`：UTF-8 ファイルからメッセージ本文を読み取る
- `-t, --to <dest>`：セッションキーの導出に使用する受信者
- `--session-key <key>`：ルーティングに使用する明示的なセッションキー
- `--session-id <id>`：明示的なセッション ID
- `--agent <id>`：エージェント ID。ルーティングバインディングを上書きする
- `--model <id>`：この実行でのモデルの上書き（`provider/model` またはモデル ID）
- `--thinking <level>`：エージェントの思考レベル（`off`、`minimal`、`low`、`medium`、`high`、および `xhigh`、`adaptive`、`max` など、プロバイダーがサポートするカスタムレベル）
- `--verbose <on|off>`：セッションの詳細出力レベルを永続化する
- `--channel <channel>`：配信チャンネル。メインセッションのチャンネルを使用する場合は省略する
- `--reply-to <target>`：配信先の上書き
- `--reply-channel <channel>`：配信チャンネルの上書き
- `--reply-account <id>`：配信アカウントの上書き
- `--local`：組み込みエージェントを直接実行する（Plugin レジストリのプリロード後）
- `--deliver`：選択したチャンネル／配信先へ応答を送り返す
- `--timeout <seconds>`：このコマンドのエージェントターンの期限を上書きする（デフォルトは 600、または `agents.defaults.timeoutSeconds`）。`0` を指定すると全体の期限が無効になる。600 秒のフォールバックはこの CLI コマンドに属するもので、通常の Gateway ターンには適用されない。通常の Gateway ターンのデフォルトは 48 時間。
- `--json`：JSON を出力する

## 例

```bash
openclaw agent --to +15555550123 --message "ステータス更新" --deliver
openclaw agent --agent ops --message "ログを要約"
openclaw agent --agent ops --message-file ./task.md
openclaw agent --agent ops --model openai/gpt-5.4 --message "ログを要約"
openclaw agent --session-key agent:ops:incident-42 --message "ステータスを要約"
openclaw agent --agent ops --session-key incident-42 --message "ステータスを要約"
openclaw agent --session-id 1234 --message "受信トレイを要約" --thinking medium
openclaw agent --to +15555550123 --message "ログを追跡" --verbose on --json
openclaw agent --agent ops --message "レポートを生成" --deliver --reply-channel slack --reply-to "#reports"
openclaw agent --agent ops --message "ローカルで実行" --local
```

## 注記

- `--message` または `--message-file` のどちらか一方だけを渡してください。`--message-file` は先頭の UTF-8 BOM を除去し、複数行の内容を保持します。有効な UTF-8 ではないファイルは拒否されます。4 MiB を超えるファイルはディスパッチ前に拒否されます。
- スラッシュコマンド（例：`/compact`）は `--message` を介して実行できません。CLI はこれを拒否し、代わりにファーストクラスのコマンド（Compaction の場合は `openclaw sessions compact <key>`）を案内します。
- `--local` の実行はワンショットです。実行用に開かれたバンドル済み MCP loopback リソースとウォーム状態の Claude stdio セッションは応答後に破棄されるため、スクリプトからの呼び出しによってローカルの子プロセスが実行中のまま残ることはありません。一方、Gateway を介した実行では、Gateway が所有する MCP loopback リソースが実行中の Gateway プロセス内に保持されます。
- `--local` を使用したスタンドアロンの組み込み実行では、再起動からの復旧が保留中の場合、既存のメインセッションの再利用を拒否します。正常な Gateway を介してターンを実行するか、Gateway 上で `/new` または `/reset` を使用してリセットしてください。独立した組み込みプロセスでは、その復旧の所有者と Gateway スキャナーを安全に連携させることはできません。
- `--agent`、`--channel`、`--to` を併用すると、セッションルーティングはチャンネルの正規の受信者と `session.dmScope` に従います。安定した送信専用の受信者 ID を持つチャンネルでは、エージェントのメインセッションから分離された、プロバイダー所有のセッションが使用されます。`--reply-channel` と `--reply-account` は配信だけに影響します。
- `--session-key` は明示的なセッションキーを選択します。エージェント接頭辞付きのキーでは `agent:<agent-id>:<session-key>` を使用する必要があり、両方を指定した場合、`--agent` はキーのエージェント ID と一致しなければなりません。センチネルではない接頭辞なしのキーは、`--agent` が指定されている場合はそのスコープに入り、それ以外の場合は設定済みのデフォルトエージェントのスコープに入ります。たとえば、`--agent ops --session-key incident-42` は `agent:ops:incident-42` にルーティングされます。リテラルキー `global` と `unknown` がスコープなしのままになるのは、`--agent` が指定されていない場合だけです。
- `--json` は標準出力を JSON 応答専用にします。Gateway、Plugin、`--local` の診断は標準エラー出力に送られるため、スクリプトは標準出力を直接解析できます。
- 一時的なハンドシェイクリトライを使い切った後、Gateway のタイムアウトまたは接続切断が発生すると、コマンドは失敗します。CLI が組み込みでターンを暗黙的に再実行することはありません。トランスポートの切断は結果が不明確です。Gateway がリクエストを受け入れ、引き続きターンを完了する可能性があるため、ターンの二重実行を避ける目的で、標準エラー出力のヒントには、再試行または `--local` を使用した再実行の前に `openclaw gateway status` とセッショントランスクリプトを確認するよう表示されます。
- `SIGTERM`/`SIGINT` は、待機中の Gateway を介したリクエストを中断します。Gateway がすでに実行を受け入れている場合、CLI は終了前にその実行 ID に対して `chat.abort` も送信します。`--local` の実行も同じシグナルを受信しますが、`chat.abort` は送信しません。転送された最初の `SIGINT` または `SIGTERM` によって終了したランチャーの子プロセスは、それぞれステータス 130 または 143 で終了します。内部の実行重複排除キーに、このセッションのアクティブな実行がすでに存在する場合、応答では `status: "in_flight"` が報告され、非 JSON の CLI は空の応答ではなく標準エラー出力に診断を表示します。外部の cron/systemd ラッパーでは、シャットダウンを完了できない場合にスーパーバイザーがプロセスを回収できるよう、`timeout -k 60 600 openclaw agent ...` のような強制終了の予備手段を維持してください。
- このコマンドによって `models.json` の再生成がトリガーされた場合、SecretRef で管理されるプロバイダー認証情報は、解決された平文のシークレットではなく、非シークレットのマーカー（例：環境変数名、`secretref-env:ENV_VAR_NAME`、`secretref-managed`）として永続化されます。マーカーの書き込みには、解決されたランタイムシークレット値ではなく、アクティブなソース設定のスナップショットが使用されます。

## JSON 配信ステータス

`--json --deliver` を使用すると、CLI の JSON 応答にトップレベルの `deliveryStatus` が含まれるため、スクリプトは配信済み、抑止、部分的成功、失敗を区別できます。

```json
{
  "payloads": [{ "text": "レポートの準備ができました", "mediaUrl": null }],
  "meta": { "durationMs": 1200 },
  "deliveryStatus": {
    "requested": true,
    "attempted": true,
    "status": "sent",
    "succeeded": true,
    "resultCount": 1
  }
}
```

Gateway を介した CLI 応答では、`result.deliveryStatus` に Gateway の未加工の結果形式も保持されます。

`deliveryStatus.status` は次のいずれかです。

| ステータス           | 意味                                                                                                                                    |
| ---------------- | ------------------------------------------------------------------------------------------------------------------------------------------ |
| `sent`           | 配信が完了しました。                                                                                                                        |
| `suppressed`     | 配信は意図的に送信されませんでした（たとえば、メッセージ送信フックによってキャンセルされた場合や、表示可能な結果がなかった場合）。終端状態であり、再試行はありません。 |
| `partial_failed` | 後続のペイロードが失敗する前に、少なくとも 1 つのペイロードが送信されました。                                                                                   |
| `failed`         | 永続的な送信が完了しなかったか、配信の事前検証が失敗しました。                                                                                   |

共通フィールド：

- `requested`：このオブジェクトが存在する場合は常に `true`。
- `attempted`：永続的な送信パスが実行された場合は `true`。事前検証の失敗時または表示可能なペイロードがない場合は `false`。
- `succeeded`：`true`、`false`、または `"partial"`。`"partial"` は `status: "partial_failed"` と組み合わせて使用される。
- `reason`：永続的な配信または事前検証から得られる、小文字のスネークケース形式の理由。既知の値には `cancelled_by_message_sending_hook`、`no_visible_payload`、`no_visible_result`、`channel_resolved_to_internal`、`unknown_channel`、`invalid_delivery_target`、`no_delivery_target` が含まれます。永続的な送信の失敗では、失敗したステージも報告される場合があります。この値の集合は拡張される可能性があるため、未知の値は不透明な値として扱ってください。
- `resultCount`：利用可能な場合、チャンネル送信結果の数。
- `sentBeforeError`：部分的な失敗で、エラーになる前に少なくとも 1 つのペイロードが送信された場合は `true`。
- `error`：送信の失敗または部分的な失敗の場合は `true`。
- `errorMessage`：基盤となる配信エラーメッセージが取得された場合にのみ存在する。事前検証の失敗には `error`/`reason` が含まれるが、`errorMessage` は含まれない。
- `payloadOutcomes`：利用可能な場合、`index`、`status`、`reason`、`resultCount`、`error`、`stage`、`sentBeforeError`、またはフックのメタデータを含む、ペイロードごとの任意の結果。

## 関連項目

- [CLI リファレンス](/ja-JP/cli)
- [エージェントランタイム](/ja-JP/concepts/agent)
