---
read_when:
    - OpenClaw リポジトリ外でのオペレーター、ダッシュボード、または WebChat クライアントの構築
    - Gateway の再接続、履歴、承認、またはデバイスペアリングの実装
    - 新しい Gateway ワイヤーバージョンに対応するためのサードパーティ製クライアントの更新
summary: Gateway WebSocket プロトコル向けのサードパーティ製オペレーターまたは WebChat クライアントを構築する
title: Gateway クライアントの構築
x-i18n:
    generated_at: "2026-07-26T09:34:45Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: fa24b196ff1fa28fb3b64d49ac25597f22cf1945aea56029e78e4375f1bdddb7
    source_path: gateway/clients.md
    workflow: 16
---

公開済みの Gateway パッケージを使用して、オペレーターダッシュボード、WebChat クライアント、
その他のサードパーティアプリケーションを構築します。このガイドでは、
通信プロトコルに関するクライアントのライフサイクル（認証、機能、再接続からの復旧、履歴、
サブスクリプション、バージョンアップ）について説明します。

フレーム形式、ハンドシェイク、エラー、完全なメソッド一覧については、
[Gateway プロトコル仕様](https://docs.openclaw.ai/gateway/protocol)を参照してください。

## パッケージのインストール

```bash
npm install @openclaw/gateway-client @openclaw/gateway-protocol
```

<Note>
これらのパッケージは OpenClaw のリリース系列に含まれます。初期展開中は、パッケージを含む最初の
OpenClaw リリースが公開されるまで、npm が `E404` を返す場合があります。
以下のレジストリページにアクセスできるようになってからインストールしてください。
</Note>

- [`@openclaw/gateway-protocol`](https://www.npmjs.com/package/@openclaw/gateway-protocol)
  は、スキーマ、ランタイムバリデーター、TypeScript 型、クライアント ID と
  機能のレジストリ、構造化エラーの読み取り機能、プロトコルバージョン定数を提供します。
  npm tarball には、生成済みの機械可読コントラクトである
  [`protocol.schema.json`](https://unpkg.com/@openclaw/gateway-protocol/protocol.schema.json)
  も含まれます。
- [`@openclaw/gateway-client`](https://www.npmjs.com/package/@openclaw/gateway-client)
  は、リファレンス接続実装です。Node クライアントにはパッケージルートを、
  ブラウザーで安全に使用できるプロトコル、デバイス認証、再接続ヘルパーには
  `@openclaw/gateway-client/browser` をインポートしてください。

Node エントリは自身の WebSocket トランスポートを管理します。ブラウザーホストは、
WebSocket アダプターに加えて、デバイス ID とデバイストークン用の永続ストレージおよび
署名コールバックを提供します。

## スコープの選択とデバイスのペアリング

承認プロンプトも表示する完全な対話型チャットクライアントでは、
次のスコープを指定して `role: "operator"` を要求する必要があります。

| スコープ             | 用途                                                                                      |
| -------------------- | ----------------------------------------------------------------------------------------- |
| `operator.read`      | `chat.history`、`sessions.list`、`sessions.subscribe`、モデルステータス、読み取り専用イベント |
| `operator.write`     | `chat.send` と通常のセッション変更                                                        |
| `operator.approvals` | exec または Plugin の承認の一覧表示、表示、解決                                             |

クライアントが対話型の質問を処理する場合にのみ `operator.questions` を、
ペアリング済みデバイスまたは Node を管理する場合にのみ `operator.pairing` を、
`config.patch` などの管理操作を行う場合にのみ `operator.admin` を追加してください。
[オペレータースコープのリファレンス](https://docs.openclaw.ai/gateway/operator-scopes)では、
メソッドと承認時のルールがすべて定義されています。

`openclaw.json` を手動編集してクライアントごとのベアラートークンを作成しないでください。
`openclaw configure --section
gateway` または `openclaw onboard --gateway-auth ...` オプションを使用して Gateway の共有ブートストラップ認証を設定し、
デバイスのペアリングによってクライアントトークンを発行させます。

1. Ed25519 デバイス ID をクライアントに永続化します。
2. `connect.challenge` を待ち、チャレンジに紐付けられたデバイスペイロードに署名して、
   要求するオペレーターのロールとスコープ、およびブートストラップ認証用の共有 Gateway トークン
   またはパスワードを指定して `connect` を送信します。
3. Gateway が構造化された `PAIRING_REQUIRED` の詳細を返した場合は、リクエスト
   ID を表示し、`error.details.recommendedNextStep` に従って一時停止または再試行します。
4. Gateway ホスト上で `openclaw devices list` を使用してリクエストを確認し、
   `openclaw devices approve <requestId>` を使用して、その時点の該当リクエストのみを承認します。
5. 再接続し、ネゴシエートされたロールとスコープとともに `hello-ok.auth.deviceToken` を
   永続化します。以降の接続では、そのデバイストークンを使用します。

スコープまたはロールをアップグレードすると、新しい保留中のペアリングリクエストが作成されます。
トークンをローテーションしても、承認済みのペアリングコントラクトを拡張することはできません。
承認、ローテーション、失効のコマンドについては、
[デバイス CLI](https://docs.openclaw.ai/cli/devices)を参照してください。

## クライアント機能の通知

`connect.params.caps` は、クライアントが利用できるオプション動作を記述します。
認可を付与するものではありません。文字列リテラルを重複して記述せず、
`GATEWAY_CLIENT_CAPS` から名前をインポートしてください。

```ts
import { GATEWAY_CLIENT_CAPS } from "@openclaw/gateway-protocol/client-info";

const caps = [GATEWAY_CLIENT_CAPS.TOOL_EVENTS];
```

現在のレジストリには、`approvals`、`exec-approvals`、`inline-widgets`、
`run-tool-bindings`、`session-scoped-events`、`plugin-approvals`、
`task-suggestions`、`terminal-offset-seq`、`tool-events`、`ui-commands` が含まれます。
クライアントが実際に実装している機能のみを通知してください。

<Warning>
`tool-events` は、ツール実行のライブストリーミングを制御します。Gateway は、
この機能を通知した接続のみを、実行の構造化ツールイベントの受信先として登録します。
この機能がない場合、接続はライブツールイベントを受信せず、
ハンドシェイクでもエラーは報告されません。
</Warning>

機能によって制限されるエージェントツールは、同じ宣言を別の目的で使用するものです。
エージェントツールがクライアント機能を必要とする場合、接続元クライアントが必要なすべての機能を
通知していなければ、Gateway はそのツールを省略します。

## 再接続後の状態復旧

成功した再接続はすべて、永続履歴と現在のメモリ内実行状態に基づく
新しいプロジェクションとして扱います。

1. `sessions.subscribe` と、選択したセッションの
   `sessions.messages.subscribe` サブスクリプションを再確立します。
2. 選択した `sessionKey` に対して `chat.history` を呼び出し、
   ローカルに永続化された行を、返された `messages` プロジェクションで置き換えます。
3. `inFlightRun` が存在する場合は、その `runId`、
   バッファー済みの `text`、および任意の `plan` を採用します。
   `text` が空の場合でも、その実行を採用します。
4. `sessionInfo.hasActiveRun` と `sessionInfo.activeRunIds` を読み取ります。保持された実行が
   引き続きストリーミング UI を所有しているかを判断する際は、`activeRunIds` への
   完全一致を優先してください。ID が列挙されていない状態で `hasActiveRun` が true の場合、
   別のアクティブなランタイムプロジェクションを表すことがあります。
5. 以降の `agent` イベントを、`payload.runId` と
   `payload.seq` に基づいて整合させます。実行ごとに、受け入れた最大シーケンスを
   個別に保持し、確認済みまたはそれ以下のシーケンスを無視し、前方の欠落がある場合は
   正規の履歴を再読み込みします。

外側のイベントフレームにも任意の `seq` があり、現在の WebSocket 接続上の
イベント順序を示します。新しい接続になるとリセットされます。`agent` イベントの
ペイロード内にある `seq` は実行ごとに割り当てられ、その実行のライフサイクル、
アシスタント、計画、ツール、その他のストリームイベントの順序を示します。

## 履歴メタデータと安定したアンカーの使用

`chat.history` が返す行には、`__openclaw` メタデータエンベロープが含まれる場合があります。

- `id` はトランスクリプトエントリの ID です。アンカー付き履歴リクエストに
  使用しますが、一意な表示行キーとしては使用しないでください。
- `seq` は正のトランスクリプトレコードシーケンスです。1 件の保存レコードが
  複数の表示行に投影される場合があるため、同じ `id` とシーケンスを持つ兄弟行を
  まとめて保持してください。
- `kind` は合成行を識別します。Compaction 境界では
  `kind: "compaction"` が使用され、一致するチェックポイントにそれらのメトリクスが記録されている場合は、
  `tokensBefore` と `tokensAfter` が含まれることがあります。

レスポンスの `hasMore` と `nextOffset` の値を使用して、過去方向にページングします。
数値オフセットは現在のトランスクリプトプロジェクションを表すため、リセットや Compaction をまたぐ
長期的なブックマークとして永続化しないでください。代わりに `__openclaw.id` を永続化します。
既知の行の周辺を復元するには、`messageId` と、それを返した `sessionId` を指定して
`chat.history` を呼び出します。Gateway はリセットアーカイブ履歴からそのアンカーを解決できます。
アンカー付きレスポンスでは、意図的に数値ページングメタデータが省略されます。

## ポーリングではなくサブスクリプションによる使用量の取得

`sessions.list` を使用して初期カタログを読み込み、接続ごとに `sessions.subscribe` を
1 回呼び出します。`sessions.changed` イベントを `sessionKey` に基づいてマージします。
セッション変更ペイロードには、ライブの `inputTokens`、`outputTokens`、
`totalTokens`、`totalTokensFresh`、`contextTokens`、`estimatedCostUsd`、
レスポンス使用量設定、アクティブな実行状態が含まれる場合があります。

一部の変更通知は無効化シグナルにすぎません。イベントでビューに必要な行フィールドが省略されている場合は、
`sessions.list` を更新してください。ライブセッションリストを最新に保つ目的で
`usage.cost` または `sessions.usage` をポーリングしないでください。
これらのメソッドは、オンデマンドの集計レポートまたは詳細レポートにのみ使用してください。

## exec 承認のバックフィル

`operator.approvals` を持つクライアントは、`hello-ok` が完了したら直ちに
イベントリスナーを登録し、その後 `exec.approval.list` を呼び出して、接続前から存在する
リクエストをバックフィルする必要があります。リストとライブの
`exec.approval.requested` / `exec.approval.resolved` イベントを承認 ID に基づいて整合させ、
リストリクエストと競合する遷移が失われたり復活したりしないようにします。

## プロトコルバージョンの追跡

現在の通信バージョンは `4` です。一般的なオペレータークライアントおよび
WebChat クライアントは、`minProtocol: 4` と `maxProtocol: 4` を使用して、
現在のバージョンと完全に一致するようネゴシエートする必要があります。
N-1 の許容範囲があるのは、認証済み Node クライアントと軽量プローブのみであり、
現在はプロトコル `3` から `4` までです。

プロトコルの変更は、まず追加的に行われます。`protocol.schema.json` には、
`since` のリリース時期メタデータとコアメソッドに必要なスコープメタデータが含まれますが、
通信バージョンの更新は、サードパーティクライアントにとって依然として明示的な破壊的変更です。
テストしたパッケージバージョンを固定し、通信バージョンが変更された場合はクライアントと Gateway を
同時にアップグレードし、アップグレードのたびに
[OpenClaw の変更履歴](https://github.com/openclaw/openclaw/blob/main/CHANGELOG.md)
を確認してください。

## 関連項目

- [Gateway プロトコル](https://docs.openclaw.ai/gateway/protocol)
- [OpenClaw の埋め込み](https://docs.openclaw.ai/gateway/embedding)
- [Gateway RPC リファレンス](https://docs.openclaw.ai/reference/rpc)
- [外部アプリ向け Gateway 連携](https://docs.openclaw.ai/gateway/external-apps)
