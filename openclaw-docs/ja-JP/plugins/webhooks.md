---
read_when:
    - 外部システムから TaskFlow をトリガーまたは駆動したい場合
    - バンドルされている webhooks Plugin を設定しています
summary: Webhook Plugin：信頼できる外部自動化向けの認証済み TaskFlow 受信口
title: Webhooks Plugin
x-i18n:
    generated_at: "2026-07-26T09:56:37Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 77e455450d6183635c76a1e8002feeb287deb4ff242dbd555ef9d0f2b21ce5f6
    source_path: plugins/webhooks.md
    workflow: 16
---

Webhooks Plugin は認証済み HTTP ルートを追加し、信頼された外部
システム（Zapier、n8n、CI ジョブ、内部サービス）が、カスタム Plugin を作成せずに
HTTP 経由で管理対象の OpenClaw TaskFlow を作成および操作できるようにします。

Plugin は Gateway プロセス内で実行されます。リモート Gateway の場合は、そのホストに
インストールして設定し、Gateway を再起動します。初期状態ではルートが
設定されていないため、少なくとも 1 つのルートを追加するまでは何も実行しません。

## ルートを設定する

`plugins.entries.webhooks.config` 配下に設定します。

```json5
{
  plugins: {
    entries: {
      webhooks: {
        enabled: true,
        config: {
          routes: {
            zapier: {
              path: "/plugins/webhooks/zapier",
              sessionKey: "agent:main:main",
              secret: {
                source: "env",
                provider: "default",
                id: "OPENCLAW_WEBHOOK_SECRET",
              },
              controllerId: "webhooks/zapier",
              description: "Zapier TaskFlow ブリッジ",
            },
          },
        },
      },
    },
  },
}
```

ルートのフィールド：

| フィールド          | 必須 | デフォルト                       | 備考                                         |
| -------------- | -------- | ----------------------------- | --------------------------------------------- |
| `enabled`      | いいえ       | `true`                        |                                               |
| `path`         | いいえ       | `/plugins/webhooks/<routeId>` | ルート間で一意である必要があります。                 |
| `sessionKey`   | はい      | -                             | バインドされた TaskFlow を所有するセッション。        |
| `secret`       | はい      | -                             | プレーン文字列または SecretRef（下記）。          |
| `controllerId` | いいえ       | `webhooks/<routeId>`          | デフォルトの `create_flow` コントローラーとして使用されます。 |
| `description`  | いいえ       | -                             | オペレーター向けの注記のみ。                       |

`secret` は、プレーン文字列または SecretRef（`{ source: "env" | "file" | "exec", provider: "default", id: "..." }`）を受け付けます。

SecretRef は、Gateway の起動時の設定スナップショット内で解決されます。あるルートの
secret を解決できない場合も Gateway は実行を継続し、そのルート自体は登録されたまま
非稼働状態になります。リクエストには一般的な認証失敗（`401`）が返されます。
その他のルートは引き続き利用できます。SecretRef のソースを修正し、Gateway を再読み込みまたは
再起動して新しいスナップショットを有効にします。SecretRef の値が公開リクエストパスで
解決されることはありません。

## セキュリティモデル

各ルートは、設定された `sessionKey` の TaskFlow 権限で動作します。そのセッションが
所有する任意の TaskFlow を検査および変更できます。TaskFlow へのアクセスは
常に `api.runtime.tasks.managedFlows.bindSession(...)` を経由するため、
ルートがバインド先のセッション外で動作することはありません。影響範囲を抑えるには：

- ルートごとに強力で一意な secret を使用します。
- インラインの平文 secret より SecretRef を優先します。
- ワークフローに適合する最小範囲のセッションにルートをバインドします。
- 必要な Webhook パスだけを公開します。

各パスのリクエスト処理順序は、HTTP メソッド（`POST` のみ）と
`Content-Type: application/json` のチェック、固定ウィンドウ方式のレート制限（パスとクライアント IP の
キーごとに 60 秒間で 120 リクエスト、追跡するキーは最大 4,096 個）、
処理中リクエストの制限（キーごとに同時 8 リクエスト、追跡するキーは最大
4,096 個）、共有 secret による認証、256 KB／
15 秒の JSON ボディ読み取りです。前段のチェックに失敗したリクエストが
後段に到達することはありません。

## リクエスト形式

`Content-Type: application/json` を指定し、`Authorization: Bearer <secret>` または `x-openclaw-webhook-secret: <secret>` のいずれかを使用して
`POST` リクエストを送信します。

```bash
curl -X POST https://gateway.example.com/plugins/webhooks/zapier \
  -H 'Content-Type: application/json' \
  -H 'Authorization: Bearer YOUR_SHARED_SECRET' \
  -d '{"action":"create_flow","goal":"受信キューを確認する"}'
```

## サポートされるアクション

| アクション             | 目的                                                            |
| ------------------ | ------------------------------------------------------------------ |
| `create_flow`      | ルートのセッション用の管理対象 TaskFlow を作成します。                 |
| `get_flow`         | ID で 1 つの TaskFlow を取得します。                                          |
| `list_flows`       | ルートのセッションの TaskFlow を一覧表示します。                            |
| `find_latest_flow` | 最後に更新された TaskFlow を取得します。                          |
| `resolve_flow`     | 不透明トークンで TaskFlow を解決します。                                |
| `get_task_summary` | TaskFlow のタスク概要を取得します。                             |
| `set_waiting`      | 任意の状態／待機データを指定して、TaskFlow を待機中としてマークします。            |
| `resume_flow`      | 待機中／ブロック中の TaskFlow を再開します。                                 |
| `finish_flow`      | TaskFlow を完了としてマークします。                                          |
| `fail_flow`        | TaskFlow を失敗としてマークします。                                            |
| `request_cancel`   | 協調的なキャンセルを要求します。                                  |
| `cancel_flow`      | TaskFlow をキャンセルします（子がまだアクティブな場合は `202` を返すことがあります）。 |
| `run_task`         | 既存の TaskFlow 内に管理対象の子タスクを作成します。           |

変更アクション（`set_waiting`、`resume_flow`、`finish_flow`、`fail_flow`、
`request_cancel`）では、楽観的並行性制御のために `flowId` と `expectedRevision` が
必要です。古いリビジョンには `409 revision_conflict` が返されます。

### `create_flow`

```json
{
  "action": "create_flow",
  "goal": "受信キューを確認する",
  "status": "queued",
  "notifyPolicy": "done_only"
}
```

### `run_task`

許可される `runtime` の値：`subagent`、`acp`。`startedAt`、`lastEventAt`、
`progressSummary` は、`status` が `"running"` の場合にのみ有効です。それ以外の
ステータスとともに送信すると `400 invalid_request` が返されます。

```json
{
  "action": "run_task",
  "flowId": "flow_123",
  "runtime": "acp",
  "childSessionKey": "agent:main:acp:worker",
  "task": "次のメッセージバッチを検査する"
}
```

## レスポンスの形式

```json
{
  "ok": true,
  "routeId": "zapier",
  "result": {}
}
```

```json
{
  "ok": false,
  "routeId": "zapier",
  "code": "not_found",
  "error": "TaskFlow が見つかりません。",
  "result": {}
}
```

フローとタスクのビューには所有者／セッションのメタデータが含まれないため、レスポンスから
ルートにバインドされた `sessionKey` が漏洩することはありません。`code` の値には `not_found`、
`not_managed`、`revision_conflict`、`persist_failed`、`cancel_requested`、
`cancel_pending`、`terminal`、`invalid_request`、`request_rejected`、および
変更が上記の名前付きコードで扱われていない理由により拒否された場合の
アクション固有のフォールバックコード（`mutation_rejected`、`create_rejected`、
`task_not_created`、`cancel_rejected`）が含まれます。

## 関連項目

- [Hooks](/ja-JP/automation/hooks) - 内部のイベント駆動型フックと、この HTTP ベースの TaskFlow ブリッジとの比較
- [Gateway Webhook（`hooks.*` 設定）](/ja-JP/automation/cron-jobs#webhooks) - 独立した汎用 Gateway HTTP エンドポイント機能。この Plugin のルートとは異なります
- [Plugin ランタイム SDK](/ja-JP/plugins/sdk-runtime)
- [CLI Webhook](/ja-JP/cli/webhooks)
