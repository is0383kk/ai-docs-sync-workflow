---
read_when:
    - OpenClaw と通信する外部アプリ、スクリプト、ダッシュボード、CI ジョブ、または IDE 拡張機能を構築している場合
    - Gateway RPC と Plugin SDK のどちらを使用するか選択しています
    - Gateway のエージェント実行、セッション、イベント、承認、モデル、またはツールと統合する場合
    - ホスティングコントローラーを外部のウェイクスケジューラーとペアリングしています
sidebarTitle: External apps
summary: 外部アプリ、スクリプト、ダッシュボード、CI ジョブ、IDE 拡張機能向けの現在の統合パス
title: 外部アプリ向け Gateway 統合
x-i18n:
    generated_at: "2026-07-26T09:02:17Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 276c6f4173197683a60770327e131e6ab2fa4d33f416ba96c170539df7246f83
    source_path: gateway/external-apps.md
    workflow: 16
---

外部アプリは Gateway プロトコル（WebSocket トランスポートと RPC メソッド）を介して OpenClaw と通信します。スクリプト、ダッシュボード、CI ジョブ、IDE
拡張機能、または別のプロセスからエージェント実行の開始、イベントのストリーミング、結果の待機、
処理のキャンセル、Gateway リソースの検査を行う場合に使用します。

<Note>
  npm パッケージ、デバイスのペアリング、再接続からの復旧、履歴、サブスクリプション、
  承認については、まず
  [Gateway クライアントの構築](https://docs.openclaw.ai/gateway/clients)を参照してください。アプリが
  Gateway を子プロセスとして監視する場合は、
  [OpenClaw の埋め込み](https://docs.openclaw.ai/gateway/embedding)も参照してください。最初の
  パッケージ展開中は、パッケージを含む最初の OpenClaw リリースが公開されるまで、npm が `E404` を返す場合があります。
</Note>

<Note>
  このページは OpenClaw プロセス外のコードを対象としています。OpenClaw 内で実行される
  Plugin コードでは、代わりにドキュメント化された `openclaw/plugin-sdk/*` サブパスを使用してください。
</Note>

## 現在利用可能なもの

| サーフェス                                                        | ステータス      | 用途                                                                                          |
| ---------------------------------------------------------------- | ------------- | --------------------------------------------------------------------------------------------- |
| [Gateway クライアントガイド](https://docs.openclaw.ai/gateway/clients) | リリース系列    | npm パッケージ、認証、再接続、履歴、イベント、承認、バージョンポリシー。                     |
| [埋め込みガイド](https://docs.openclaw.ai/gateway/embedding)      | リリース系列    | 子プロセスの環境、準備状態、ライフサイクル、復旧、RPC の所有権、パッケージング。             |
| [Gateway プロトコル](/ja-JP/gateway/protocol)                          | 利用可能        | WebSocket トランスポート、接続ハンドシェイク、認証スコープ、プロトコルのバージョニング、イベント。 |
| [Gateway RPC リファレンス](/ja-JP/reference/rpc)                       | 利用可能        | エージェント、セッション、タスク、モデル、ツール、成果物、承認に関する現在の Gateway メソッド。 |
| [`openclaw agent`](/ja-JP/cli/agent)                                 | 利用可能        | CLI の呼び出しで十分な場合の、単発のスクリプト統合。                                        |
| [`openclaw message`](/ja-JP/cli/message)                               | 利用可能        | スクリプトからのメッセージまたはチャンネルアクションの送信。                                |

## 推奨手順

1. Gateway を実行または検出します。
2. [Gateway プロトコル](/ja-JP/gateway/protocol)経由で接続します。
3. [Gateway RPC リファレンス](/ja-JP/reference/rpc)に記載された RPC メソッドを呼び出します。
4. テスト対象の OpenClaw バージョンを固定します。
5. OpenClaw のアップグレード時に RPC リファレンスを再確認します。

エージェント実行では、`agent` RPC から始め、ターミナル結果を得るために `agent.wait` と組み合わせます。
永続的な会話状態には、`sessions.*` メソッドを使用します。
UI 統合では、Gateway イベントをサブスクライブし、アプリが理解できるイベント
ファミリーだけをレンダリングします。

## 協調的なホスト停止

実行中のプロセスをフリーズまたはスナップショットするホスティングコントローラーでは、次の
ホスト中立な停止ハンドシェイクを使用できます。

1. ホストが制御する外部イングレスの受け入れを停止します。
2. 安定した一意の `requestId` を指定して `gateway.suspend.prepare` を呼び出します。
3. レスポンスが `busy` の場合は、プロセスを実行したままにし、後で再試行します。
4. `ready` の場合は、返された `suspensionId` を保存し、`expiresAtMs` より前に
   プロセスをフリーズまたはスナップショットします。
5. 再開後、または停止を中止した場合は、既存の WebSocket または Admin HTTP 制御
   パスを介して、その `suspensionId` を指定して `gateway.suspend.resume` を呼び出します。

準備済みの Gateway は新しい WebSocket ハンドシェイクを拒否します。WebSocket コントローラーは、
ホスト操作中も認証済み接続を開いたままにする必要があります。それを
保証できない場合は、準備を行う前に
[Admin HTTP RPC Plugin](/ja-JP/plugins/admin-http-rpc)を有効にして使用してください。
制御パスが失われた場合は、再接続する前に 2 分間のリースが期限切れになるのを
待ってください。期限切れになると受け入れが自動的に再開されます。

RPC コントラクトは次のとおりです。

- `gateway.suspend.prepare` — `operator.admin`、パラメーター:
  `{ "requestId": "stable-host-operation-id" }`
- `gateway.suspend.status` — `operator.read`、パラメーター:
  `{ "suspensionId": "id-from-prepare" }`
- `gateway.suspend.resume` — `operator.admin`、パラメーター:
  `{ "suspensionId": "id-from-prepare" }`

ID は前後の空白が除去され、空白以外の文字を含む必要があり、128 文字に
制限されます。ビジー状態の prepare 結果には `status: "busy"`、`reason`、
`retryAfterMs`、`activeCount`、`blockers` が含まれます。ready 結果の形式は次のとおりです。

```json
{
  "status": "ready",
  "suspensionId": "2c3f...",
  "expiresAtMs": 1770000000000,
  "activeCount": 0,
  "blockers": []
}
```

status は `{"status":"running"}`、または `expiresAtMs` を含む ready 結果を返します。
resume は `{"ok":true,"status":"running","resumed":true}` を返し、正常に再開した後に
繰り返すと `resumed: false` を返します。

競合するリクエスト ID または一時的なスケジューラー再開失敗では、`retryAfterMs` を含む
再試行可能な `UNAVAILABLE` が返されます。スケジューラーの復旧中は、prepare、status、
resume のすべてがそのエラーを返し、Gateway は準備未完了かつ
フェイルクローズ状態を維持するため、ホストはフリーズまたはスナップショットしてはなりません。OpenClaw は
スケジューラーを自動的に再試行し、復旧が成功した後にのみ受け入れを再開します。
一致しない resume ID では `INVALID_REQUEST` が返されます。prepare は Gateway の
1 分あたり 3 回というコントロールプレーン書き込み予算を共有します。返された
再試行遅延に従ってください。WebSocket クライアントはデバイスと IP ごとにバケット化されます。Admin HTTP
コントローラーは解決済みクライアント IP ごとにバケット化されるため、1 つの
プロキシの背後にあるコントローラーは予算を共有する場合があります。

準備は拒否のみを行います。OpenClaw は新しいルート、セッション、コマンドの受け入れを閉じ、
自動 Cron ティックを一時停止し、処理を同期的に検査します。何かが
アクティブな場合は、`busy` を返す前にスケジューラーを再開して受け入れを
再開します。その処理を中断したり、完了まで待ったりはしません。ready リースは 2
分間持続します。同じ `requestId` で `prepare` を繰り返すと更新され、期限切れになると
受け入れを再開する前にスケジューラーが再開されます。
ready リース中に期限を迎えた再起動の発行は、リースが再開されるまで待機します。
進行中の再起動がある場合、準備は `busy` を返します。

ready の間も `/healthz` は稼働し続け、`/readyz` は `503` を返します。ローカルまたは
認証済みの準備状態レスポンスには `gateway-draining` が含まれます。未認証の
リモートプローブには `{ "ready": false }` のみが返されます。HTTP ヘルスプローブ、
既存の WebSocket 接続上の停止メソッド、およびすでに有効な
Admin HTTP RPC ルートは引き続き利用できます。その他の RPC は再試行可能な
`UNAVAILABLE` を返します。OpenAI 互換 API、ツール／セッション操作、Node 監視、
設定済みフックを含む、組み込み HTTP ユーザー処理ルートおよび通常の Plugin HTTP ルートは、
`error.code: "gateway_unavailable"` を含む `503` を返します。Plugin が所有する新しい
WebSocket アップグレードも `503` を返します。これはアップグレードの
所有権を対象とし、確立済みの Plugin ソケット上で後から実行される処理は対象としません。

このハンドシェイクは、受信メッセージの永続化、サードパーティ製チャンネル
トランスポートの停止、またはホスティングプラットフォームの制御を行いません。ホストは準備前にイングレスを
遮断する必要があり、ウェイク、スナップショット／フリーズ、停止について引き続き責任を負います。
`activeCount` は追跡対象処理の集計数で、`blockers` には
ゼロ以外のカテゴリ数と上限付きのタスク詳細が含まれます。これは一般的な
プロセス静止障壁ではありません。`background-exec` ブロッカーは集計情報のみです。
コマンドテキスト、プロセス ID、出力、セッションまたはスコープ識別子が
プロトコルを越えることはありません。チャンネルのヘルス、メンテナンス、キャッシュ更新、確立済みの
Plugin WebSocket セッション、未登録で Plugin が所有するバックグラウンド処理は
アクティブなままになる可能性があります。
ホスティングプラットフォームは、プロセスツリー全体とその
ファイルシステムを一貫してフリーズまたはスナップショットする必要があります。この最初の
コントラクトでは、未登録の処理がアイドル状態であることを証明できません。

<Tip>
  ホストのウェイクスケジューリングでは、OpenClaw 向けの部分をインプロセス
  Plugin に保持し、べき等な完全スナップショットを外部ホストアダプターへ投影してください。
  ホスティングコントローラーは Plugin SDK をインポートしたり、イベント差分から Cron
  状態を再構築したりしないでください。[安全な外部 Cron
  投影](/ja-JP/plugins/hooks#safe-external-cron-projection)を参照してください。
</Tip>

## アプリコードと Plugin コード

コードが OpenClaw の外部にある場合は Gateway RPC を使用します。

- エージェント実行を開始または監視する Node スクリプト
- Gateway を呼び出す CI ジョブ
- ダッシュボードと管理パネル
- IDE 拡張機能
- チャンネル Plugin になる必要のない外部ブリッジ
- 偽または実際の Gateway トランスポートを使用する統合テスト

コードが OpenClaw 内で実行される場合は Plugin SDK を使用します。

- プロバイダー Plugin
- チャンネル Plugin
- ツールまたはライフサイクルフック
- エージェントハーネス Plugin
- 信頼済みランタイムヘルパー

外部アプリは `openclaw/plugin-sdk/*` をインポートしないでください。これらのサブパスは、
OpenClaw によってロードされる Plugin 用です。

## 関連項目

- [Gateway クライアントの構築](https://docs.openclaw.ai/gateway/clients)
- [OpenClaw の埋め込み](https://docs.openclaw.ai/gateway/embedding)
- [Gateway プロトコル](/ja-JP/gateway/protocol)
- [Gateway RPC リファレンス](/ja-JP/reference/rpc)
- [CLI エージェントコマンド](/ja-JP/cli/agent)
- [CLI メッセージコマンド](/ja-JP/cli/message)
- [エージェントループ](/ja-JP/concepts/agent-loop)
- [エージェントランタイム](/ja-JP/concepts/agent-runtimes)
- [セッション](/ja-JP/concepts/session)
- [バックグラウンドタスク](/ja-JP/automation/tasks)
- [ACP エージェント](/ja-JP/tools/acp-agents)
- [Plugin SDK の概要](/ja-JP/plugins/sdk-overview)
