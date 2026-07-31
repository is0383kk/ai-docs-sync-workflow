---
read_when:
    - エージェントまたはツールを誰が実行したのか、いつ実行したのか、そしてどのように終了したのかを回答する必要があります
    - コンテンツを含まない受信または送信メッセージのライフサイクルメタデータが必要です
    - 範囲が限定され、機密情報を安全に秘匿できるアクティビティのエクスポートが必要です
summary: メタデータのみの実行、ツール、メッセージのライフサイクル監査レコードに関する CLI リファレンス
title: 監査記録
x-i18n:
    generated_at: "2026-07-26T09:28:53Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: da9df6f388b0a24c3b79d755fa59d047cce99262bc6d9c890be7a83da75693a8
    source_path: cli/audit.md
    workflow: 16
---

# `openclaw audit`

エージェント実行、ツールアクション、オプトインのメッセージライフサイクル記録について、Gateway のメタデータ専用監査台帳を照会します。

台帳は、実行イベントとツールイベントについてデフォルトで有効です。すべての新規イベント記録を停止するには、
[`audit.enabled: false`](/ja-JP/gateway/configuration-reference#audit) を設定して Gateway を再起動します。
メッセージ記録は別途デフォルトで無効になっています。記録するには、`audit.messages` を
`direct` または `all` に設定して Gateway を再起動します。
既存の記録は、有効期限（30 日）を迎えるまで照会できます。

台帳は会話トランスクリプトとは別のものです。ID、順序、来歴、アクション、ステータス、正規化された結果コードを記録しますが、コンテンツは一切保存しません。また、メッセージ識別子はインストール環境内でのみ有効な鍵付き仮名としてのみ現れます。[監査履歴](/ja-JP/gateway/audit)では、完全なデータモデル、プライバシーの意味論、ストレージと保持期間の上限、カバレッジの制限を扱います。このページではコマンドの操作面を説明します。

```bash
openclaw audit
openclaw audit --agent main --status failed
openclaw audit --session "agent:main:main" --after 2026-07-01T00:00:00Z
openclaw audit --run 8c69f72e-8b11-4c54-98d5-1a3dd67450c3
openclaw audit --kind tool_action --limit 50 --json
openclaw audit --kind message --direction outbound --channel telegram --json
```

## フィルター

- `--agent <id>`: 完全一致するエージェント ID
- `--session <key>`: 完全一致するセッションキー
- `--run <id>`: 完全一致する実行 ID
- `--kind <kind>`: `agent_run`、`tool_action`、または `message`
- `--status <status>`: `started`、`succeeded`、`failed`、`cancelled`、
  `timed_out`、`blocked`、または `unknown`
- `--direction <direction>`: メッセージの方向（`inbound` または `outbound`）
- `--channel <channel>`: 完全一致するメッセージチャネル
- `--after <timestamp>` / `--before <timestamp>`: 境界を含む ISO タイムスタンプまたは
  Unix ミリ秒
- `--limit <count>`: 1～500 のページサイズ。デフォルトは `100`
- `--cursor <sequence>`: 以前の新しい順のクエリを続行
- `--json`: 上限付きページを JSON として出力

CLI はバージョン管理されたアクティビティ RPC を照会するため、1 つのコマンドで設定済み台帳全体を表示できます。テキスト出力には、時刻、種類、方向、チャネル、ステータス、エージェント、実行、アクションが表示されます。メッセージの来歴がない場合は `-` と表示されます。OpenClaw がエージェント ID や実行 ID を作り出すことはありません。ツールアクションにはツール名も表示されます。別のページが存在する場合、JSON 出力には `nextCursor` が含まれます。ページング中に到着した記録の順序を変えずに続行するには、その値を `--cursor` に渡します。

これらのエクスポートにはメッセージ本文や生のメッセージ ID フィールドが含まれませんが、依然として機密性のある運用メタデータです。エージェント ID、セッション ID、実行 ID、時刻、チャネル、結果、安定した HMAC 参照から、アクティビティを関連付けられる可能性があります。他の運用担当者向け記録と同じアクセス制御および保持方法で保護してください。

## 記録されるイベント

Gateway は、信頼されたライフサイクルストリームを 6 つのアクションに投影します。

- `agent.run.started`
- `agent.run.finished`
- `tool.action.started`
- `tool.action.finished`
- `message.inbound.processed`
- `message.outbound.finished`

返されるすべての記録には、安定したイベント ID、単調増加する台帳シーケンス、ライフサイクルのタイムスタンプ、アクター、アクション、ステータス、`schemaVersion: 1` マーカー、ソースシーケンス、`redaction: "metadata_only"` が含まれます。エージェント、セッション、実行の来歴とイベント固有のフィールドは、信頼されたソースから提供される場合にのみ存在します。メッセージ記録では意図的に `sessionKey` と `sessionId` が省略されるため、`--session` フィルターは実行記録とツール記録だけに適用されます。

終端状態の実行記録とツール記録では、成功、失敗、キャンセル、タイムアウト、ポリシーによるブロックを、クローズ済みステータスとエラーコードによって区別します。上流のランタイムが信頼できる終端結果を公開しない場合、`unknown` は明示的な非成功結果になります。ツール呼び出し ID は、安定したフィンガープリントとしてのみエクスポートされます。ツール名は、モデル向けの簡潔な名前の規約に一致する必要があります。それ以外の値は `unknown` になります。

メッセージ記録には、方向、チャネル、会話の種類、結果のほか、任意で配信の種類、失敗段階、所要時間、結果数、正規化された理由コード、鍵付きのアカウント、会話、メッセージ、送信先の仮名が追加されます。現在の受信境界は、コアディスパッチに到達した受理済みメッセージを対象としており、コアによる重複判定と終端処理の結果も含みます。送信境界は、共有の永続的な配信処理に到達した元の論理返信ペイロードごとに 1 つの終端行を書き込みます。チャンク分割とアダプターのファンアウトは `resultCount` に集約されます。再試行可能な送信または結果が曖昧な送信がキューに入った場合、確認応答、デッドレター、または照合によって結果が終端状態になった後にのみ記録されます。これらの共有境界を迂回する Plugin 内部の経路と直接送信経路は、まだ対象外です。行が存在しないことは、メッセージが存在しなかったことの証明にはなりません。

監査台帳は、トランスクリプト、タスク履歴、Cron 実行履歴、ログの代替ではありません。会話コンテンツを別のストアに複製することなく、運用担当者からの問い合わせに対応するための小規模な実行横断インデックスを提供します。

受信行では、`durationMs` がコアディスパッチを測定し、`resultCount` が確定したキュー内のツール、ブロック、返信ペイロードを数えます。送信行では、`durationMs` に配信の所有開始から終端まで（したがってキューでの待機時間も）が含まれ、`resultCount` は識別された物理プラットフォームへの送信数を数えます。`deliveryKind` が存在する場合、フック適用後かつレンダリング後の実効ペイロードを表します。抑制された行とクラッシュにより結果が曖昧な行では省略されます。

## Gateway RPC

`audit.activity.list` には `operator.read` が必要で、同じフィルターを受け付けます。実行、ツール、受信メッセージ、送信メッセージの記録を含む、名前付き V1 アクティビティイベントユニオンを返します。

```bash
openclaw gateway call audit.activity.list --params '{"channel":"telegram","limit":50}'
```

結果は `{ "events": AuditActivityEventV1[], "nextCursor"?: string }` です。
結果は新しい順に並び、リクエストごとに最大 500 件に制限されます。

出荷済みの `audit.list` RPC は、従来の実行クライアントおよびツールクライアント向けに変更されていません。古い Gateway で `audit.activity.list` が利用できない場合、CLI は要求されたすべてのフィルターが旧式メソッドでサポートされている場合にのみ `audit.list` を再試行します。`--kind message`、`--direction`、`--channel` は、古い Gateway では暗黙に破棄されず、アップグレードを促すメッセージとともに失敗します。

## 関連項目

- [監査履歴](/ja-JP/gateway/audit)
- [Gateway プロトコル](/ja-JP/gateway/protocol#audit-ledger-rpc)
- [セッション](/ja-JP/cli/sessions)
- [タスク](/ja-JP/cli/tasks)
- [Cron ジョブ](/ja-JP/automation/cron-jobs)
