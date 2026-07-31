---
read_when:
    - Gateway WS クライアントの実装または更新
    - プロトコルの不一致や接続失敗のデバッグ
    - プロトコルスキーマ/モデルの再生成
summary: Gateway WebSocket プロトコル：ハンドシェイク、フレーム、バージョニング
title: Gateway プロトコル
x-i18n:
    generated_at: "2026-07-26T10:02:14Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 89d637a9070bc6512a182fea0fd890b56287e0080515ba4fba9b2591c6247e0d
    source_path: gateway/protocol.md
    workflow: 16
---

Gateway WS プロトコルは、OpenClaw の単一のコントロールプレーン兼 Node トランスポートです。
オペレーターおよび Node クライアント（CLI、Web UI、macOS アプリ、iOS/Android Node、
ヘッドレス Node）は WebSocket 経由で接続し、ハンドシェイク時に **ロール** と **スコープ** を
宣言します。

## npm パッケージ

これらのパッケージは OpenClaw のリリース系列に含まれます。初期ロールアウト中は、
パッケージを含む最初のリリースが公開されるまで、npm が `E404` を返す場合があります。

- [`@openclaw/gateway-protocol`](https://www.npmjs.com/package/@openclaw/gateway-protocol)
  は、スキーマ、バリデーター、TypeScript 型、軽量なフレームおよびエラー
  ヘルパー、バージョン定数を公開します。その tarball には、生成された機械可読な
  [`protocol.schema.json`](https://unpkg.com/@openclaw/gateway-protocol/protocol.schema.json)
  コントラクトが含まれます。
- [`@openclaw/gateway-client`](https://www.npmjs.com/package/@openclaw/gateway-client)
  は、リファレンス Node クライアントと、`@openclaw/gateway-client/browser` にある
  ブラウザーセーフなエントリを公開します。

アプリケーションのライフサイクルに関するガイダンスについては、
[Gateway クライアントの構築](https://docs.openclaw.ai/gateway/clients)を参照してください。Gateway を
子プロセスとして監視するアプリについては、
[OpenClaw の埋め込み](https://docs.openclaw.ai/gateway/embedding)を参照してください。

## トランスポートとフレーミング

- WebSocket、テキストフレーム、JSON ペイロード。
- 最初のフレームは `connect` リクエストである**必要があります**。
- 接続前のフレームは 64 KiB（`MAX_PREAUTH_PAYLOAD_BYTES`）に制限されます。ハンドシェイク後は
  `hello-ok.policy.maxPayload` と
  `hello-ok.policy.maxBufferedBytes` に従います。診断が有効な場合、Gateway がフレームを閉じるか破棄する前に、
  サイズ超過の受信フレームと低速な送信バッファによって `payload.large` イベントが発行されます。
  これらのイベントには `surface`、バイトサイズ、上限、安全な理由コードが含まれますが、
  メッセージ本文、添付ファイルの内容、生のフレームバイト、トークン、Cookie、シークレットは
  決して含まれません。

フレーム形式：

- リクエスト：`{type:"req", id, method, params}`
- レスポンス：`{type:"res", id, ok, payload|error}`
- イベント：`{type:"event", event, payload, seq?, stateVersion?}`

レスポンスエラーでは `{ code, message, details?, retryable?, retryAfterMs? }` を使用します。
クライアントは `code` と `details.code` に基づいて分岐する必要があります。`message` は人間が読める形式のままであり、
互換性に関する注記で別途規定されている場合を除き、変更される可能性があります。メソッドレベルの
認可エラーでは、構造化された不足スコープの詳細とともに、トップレベルの `code: "FORBIDDEN"` を使用します。

- 不足スコープ：`{ code: "MISSING_SCOPE", missingScope, requiredScopes }`。
  `requiredScopes` は、要求された操作について既知のスコープをすべて含む完全なセットです。
  従来の `missing scope: <scope>` メッセージは古いクライアント向けに維持されています。

クライアントは最初に `details` を読み、従来のメッセージは互換性のための
フォールバックとしてのみ使用する必要があります。`readMissingScopeError` と `readMissingScopeErrorDetails` は
`@openclaw/gateway-protocol/gateway-error-details` からエクスポートされます。ブラウザーセーフな Gateway クライアントは、
これらを `@openclaw/gateway-client/browser` から再エクスポートします。

スキーマは `@openclaw/gateway-protocol/schema` から `GatewayErrorDetailsSchema`、
`MissingScopeErrorDetailsSchema` としてエクスポートされます。
HTTP スコープエラーでは、`error.details` の下に `MISSING_SCOPE` オブジェクトを反映し、
HTTP ステータス `403` を使用します。

副作用を伴うメソッドには冪等性キーが必要です（スキーマを参照）。

## ハンドシェイク

Gateway は接続前チャレンジを送信します。

```json
{
  "type": "event",
  "event": "connect.challenge",
  "payload": { "nonce": "…", "ts": 1737264000000 }
}
```

クライアントは `connect` で応答します。

```json
{
  "type": "req",
  "id": "…",
  "method": "connect",
  "params": {
    "minProtocol": 4,
    "maxProtocol": 4,
    "client": {
      "id": "cli",
      "version": "1.2.3",
      "platform": "macos",
      "mode": "operator"
    },
    "role": "operator",
    "scopes": ["operator.read", "operator.write"],
    "caps": [],
    "commands": [],
    "permissions": {},
    "auth": { "token": "…" },
    "locale": "en-US",
    "userAgent": "openclaw-cli/1.2.3",
    "device": {
      "id": "device_fingerprint",
      "publicKey": "…",
      "signature": "…",
      "signedAt": 1737264000000,
      "nonce": "…"
    }
  }
}
```

Gateway は `hello-ok` で応答します。

```json
{
  "type": "res",
  "id": "…",
  "ok": true,
  "payload": {
    "type": "hello-ok",
    "protocol": 4,
    "server": { "version": "…", "connId": "…" },
    "features": { "methods": ["…"], "events": ["…"] },
    "snapshot": { "…": "…" },
    "auth": {
      "role": "operator",
      "scopes": ["operator.read", "operator.write"]
    },
    "policy": {
      "maxPayload": 26214400,
      "maxBufferedBytes": 52428800,
      "tickIntervalMs": 15000
    }
  }
}
```

`server`、`features`、`snapshot`、`policy`、`auth` はすべて
`HelloOkSchema`（`packages/gateway-protocol/src/schema/frames.ts`）で必須です。`auth` は、
デバイストークンが発行されない場合でも、ネゴシエーションされたロールとスコープを報告します（上記の形式）。
`pluginSurfaceUrls` は省略可能で、Plugin サーフェス名（例：
`canvas`）をスコープ付きのホスト URL にマッピングします。有効期限が切れる場合があるため、
Node は新しいエントリを取得するために `{ "surface": "canvas" }` を指定して
`node.pluginSurface.refresh` を呼び出します。
非推奨の `canvasHostUrl` / `canvasCapability` / `node.canvas.capability.refresh`
パスはサポートされていません。Plugin サーフェスを使用してください。
スナップショットの省略可能な `appliedConfigHash` は、アクティブな Gateway ランタイムが受け入れた、
解決済みのソース設定リビジョンです。クライアントはこれを
`config.get.configRevisionHash` と比較し、より新しく保存された設定で再起動が引き続き
必要かどうかを判断できます。`config.get.hash` は、設定書き込みの競合ガードで使用される
生のルートファイルリビジョンのままです。

Gateway が起動時のサイドカーの処理をまだ完了していない間、`connect` は
`details.reason: "startup-sidecars"` と `retryAfterMs` を伴う、再試行可能な
`UNAVAILABLE` エラーを返す場合があります。これを最終的なハンドシェイク失敗として扱わず、
接続予算内で再試行してください。

デバイストークンが発行されると、`hello-ok.auth` に追加されます。

```json
{
  "auth": {
    "deviceToken": "…",
    "role": "operator",
    "scopes": ["operator.read", "operator.write"]
  }
}
```

組み込みの QR/セットアップコードによるブートストラップは、モバイルへの引き継ぎ経路です。
ベースラインのセットアップコード接続に成功すると、プライマリ Node トークンと、範囲が限定された
オペレータートークンが 1 つ返されます。

```json
{
  "auth": {
    "deviceToken": "…",
    "role": "node",
    "scopes": [],
    "deviceTokens": [
      {
        "deviceToken": "…",
        "role": "operator",
        "scopes": ["operator.approvals", "operator.read", "operator.talk.secrets", "operator.write"]
      }
    ]
  }
}
```

このオペレーターへの引き継ぎは意図的に範囲が限定されています。Talk の設定読み取り用の
`operator.talk.secrets` を含め、モバイルオペレーターのループとネイティブセットアップを
開始するには十分ですが、ペアリング変更スコープと `operator.admin` は含まれません。
より広範なペアリング/管理者アクセスには、別途承認されたペアリングまたはトークンフローが必要です。
ブートストラップ認証が信頼されたトランスポート（`wss://` または
loopback/ローカルペアリング）を介して実行された場合にのみ、`hello-ok.auth.deviceTokens` を永続化してください。

信頼された同一プロセスのバックエンドクライアント（`client.id: "gateway-client"`、
`client.mode: "backend"`）は、共有 Gateway トークン/パスワードで認証する場合、
直接 loopback 接続で `device` を省略できます。この経路は
内部コントロールプレーン RPC（例：サブエージェントのセッション更新）専用であり、
古い CLI/デバイスのペアリングベースラインがローカルバックエンド作業を妨げることを回避します。
リモート、ブラウザーオリジン、Node、および明示的なデバイストークン/デバイス ID クライアントは、
引き続き通常のペアリングとスコープアップグレードのチェックを通過します。

### ワーカーロールと閉じたプロトコル

クラウドワーカーは、Gateway が所有し、ホストキーが固定された SSH トンネルを介して、
専用の loopback イングレスを使用します。このイングレスはワーカー ID のみを受け入れ、
一般認証、Node イベント、オペレーター RPC、Plugin メソッドを決してディスパッチしません。
厳格な `connect` は、保存時にハッシュ化された、環境、バンドルハッシュ、
所有者エポック、RPC セットバージョン、有効期限、および null 許容の 1 セッションに
紐付けられた短期クレデンシャルを検証します。また、現在のバージョンと機能セットを個別に確認します。
成功時には最小限の `worker-hello-ok` が返されます。機能ネゴシエーションは、
一般プロトコルのバージョンとは独立しています。フレームは 64 KiB 未満に保たれますが、
ネゴシエーションされた `worker.inference.start` フレームのみ最大 25 MiB まで許可されます。
閉じた許可リストには `worker.heartbeat`、`worker.transcript.commit`、
`worker.live-event`、`worker.inference.start`、`worker.inference.cancel` が含まれます。

トランスクリプトのコミットでは、所有者エポックによるフェンシング、Gateway が所有するセッションバインディング、
ベースリーフの compare-and-swap、永続的なシーケンス再生を使用します。Gateway は通常のセッションライターを介して、
トランスクリプトエントリ ID と親 ID を生成します。所有権と有効期限は RPC ごとに再確認されます。

### クライアント機能

オペレータークライアントは、`connect.params.caps` で省略可能な機能を通知できます。

- `tool-events`：構造化されたツールライフサイクルイベントを受け入れます。
- `inline-widgets`：ホストされたインラインウィジェットのツール結果をレンダリングできます。

クライアント機能は、認可ではなく、接続されているクライアントについて記述します。エージェントツールは必要な機能を宣言できます。起点となるクライアントの `caps` にすべての要件が含まれていない場合、Gateway はそれらのツールを除外します。チャネル起点の実行には Gateway クライアント機能がないため、ツールポリシーで明示的に許可されている場合でも、機能によって制限されたツールは利用できません。

### Node 接続の例

```json
{
  "type": "req",
  "id": "…",
  "method": "connect",
  "params": {
    "minProtocol": 4,
    "maxProtocol": 4,
    "client": {
      "id": "ios-node",
      "version": "1.2.3",
      "platform": "ios",
      "mode": "node"
    },
    "role": "node",
    "scopes": [],
    "caps": ["camera", "canvas", "screen", "location", "voice"],
    "commands": ["camera.snap", "canvas.navigate", "screen.record", "location.get"],
    "permissions": { "camera.capture": true, "screen.record": false },
    "auth": { "token": "…" },
    "locale": "en-US",
    "userAgent": "openclaw-ios/1.2.3",
    "device": {
      "id": "device_fingerprint",
      "publicKey": "…",
      "signature": "…",
      "signedAt": 1737264000000,
      "nonce": "…"
    }
  }
}
```

Node は接続時に機能要求を宣言します。

- `caps`：`camera`、`canvas`、`screen`、
  `location`、`voice`、`talk` などの上位カテゴリ。
- `commands`：呼び出し用のコマンド許可リスト。
- `permissions`：細粒度の切り替え（例：`screen.record`、`camera.capture`）。

Gateway はこれらを要求として扱い、サーバー側の許可リストを適用します。

## ロールとスコープ

オペレータースコープの完全なモデル、承認時のチェック、共有シークレットの
セマンティクスについては、[オペレータースコープ](/ja-JP/gateway/operator-scopes)を参照してください。

ロール：

- `operator`：コントロールプレーンクライアント（CLI/UI/自動化）。
- `node`：機能ホスト（camera/screen/canvas/system.run）。
- `worker`：専用の閉じたワーカープロトコル上のクラウド実行ホスト。

オペレータースコープ（`src/gateway/operator-scopes.ts`）の完全な閉集合：

- `operator.read`
- `operator.write`
- `operator.admin`
- `operator.approvals`
- `operator.pairing`
- `operator.talk.secrets`

`includeSecrets: true` を指定した `talk.config` には、`operator.talk.secrets`（または
`operator.admin`）が必要です。シークレットが含まれる場合は、アクティブな Talk プロバイダーの
クレデンシャルを `talk.resolved.config.apiKey` から読み取ります。`talk.providers.<id>.apiKey` は
ソースの形式を維持し、SecretRef オブジェクトまたは墨消しされた文字列である場合があります。

Plugin が登録した Gateway RPC メソッドは独自のオペレータースコープを要求できますが、
次の予約済みコアプレフィックスは常に `operator.admin`
（`src/shared/gateway-method-policy.ts`）に解決されます：`config.*`、`exec.approvals.*`、
`wizard.*`、`update.*`。

メソッドスコープは最初のゲートにすぎません。`chat.send` 経由で到達する一部の
スラッシュコマンドには、より厳格なコマンドレベルのチェックが適用されます。すでに低位の
オペレータースコープを保持している Gateway クライアントであっても、永続的な `/config set` と
`/config unset` の書き込みには `operator.admin` が必要です。

`node.pair.approve` には、基本メソッドスコープ（`operator.pairing`）に加えて、
保留中のリクエストで宣言された `commands`（`src/infra/node-pairing-authz.ts`）に基づく、
承認時の追加スコープチェックがあります。

| 宣言されたコマンド                                                                                                             | 必要なスコープ                       |
| ----------------------------------------------------------------------------------------------------------------------------- | ------------------------------------- |
| なし                                                                                                                          | `operator.pairing`                    |
| 通常のコマンド                                                                                                             | `operator.pairing` + `operator.write` |
| `system.run`、`system.run.prepare`、`system.which`、`browser.proxy`、`fs.listDir`、または `system.execApprovals.get/set` を含む | `operator.pairing` + `operator.admin` |

### 機能／コマンド／権限（Node）

Node は接続時に機能の宣言を行います。

- `caps`: `camera`、`canvas`、`screen`、
  `location`、`voice`、`talk` などの上位レベルの機能カテゴリ。
- `commands`: 呼び出し用のコマンド許可リスト。
- `permissions`: 詳細な切り替え（例: `screen.record`、`camera.capture`）。

Gateway はこれらを**宣言**として扱い、サーバー側の許可リストを適用します。
接続済みの Node は、接続または再接続が成功した後に、`node.pluginTools.update` を使用して、任意のエージェント可視 Plugin または MCP ツールの
記述子を公開できます。ヘッドレス Node ホストでは、宣言的な MCP インベントリの
変更を適用するために再起動します。この更新メソッドが唯一の公開経路であり、Plugin ツール記述子は
`connect` パラメーターでは受け付けられません。各記述子は、プロバイダーで安全に使用できるツール `name` を使用し、Node の現在のコマンド許可リストにある
`command` を指定する必要があります。Gateway はペアリングされた Node からの記述子
メタデータを信頼し、承認済みのコマンド範囲外にある記述子をフィルタリングし、
Node の切断時に削除し、別の Node のカタログを変更しようとする
オペレーターの操作を拒否します。Node が公開した記述子を無視するには、`gateway.nodes.pluginTools.enabled: false`
を設定します。

接続済みの Node ホストは、`node.skills.update` を使用して、完全な Skills 置換カタログを公開します。
この Node ロールのメソッドが Node の Skills を公開する唯一の経路であり、
Skills は `connect` パラメーターでは受け付けられません。各記述子には、
安全な名前、説明、およびサイズが制限された `SKILL.md` コンテンツが含まれます。Gateway はその
コンテンツを通常の Skills ローダーで解析し、Node の接続中はエージェントの Skills スナップショットに
含め、切断時に削除します。Node が公開した Skills を無視するには、
`gateway.nodes.allowSkills: false` を設定します。

## プレゼンス

- `system-presence` はデバイス ID をキーとするエントリを返します。これには
  `deviceId`、`roles`、`scopes` が含まれるため、オペレーターと Node の両方として
  接続している場合でも、UI はデバイスごとに 1 行を表示できます。
- `node.list` には任意の `lastSeenAtMs` と `lastSeenReason` が含まれます。接続済みの
  Node は、理由 `connect` とともに現在の接続時刻を報告します。ペアリングされた Node は、
  信頼済み Node イベントを介して、永続的なバックグラウンドプレゼンスも報告できます。

ネイティブ macOS Node は、入力アイドル時間を制限した、認証済みの `node.presence.activity` イベントも送信できます。
Gateway は自身のクロックでアクティビティのタイムスタンプを生成し、
`node.list` と `node.describe` を通じて、接続中で最も新しい Mac を公開し、
`node.presence` の更新を読み取りスコープを持つクライアントにブロードキャストします。
ユーザーがアクティビティ共有を無効にすると、アプリは `{ "action": "clear" }` を送信します。
Gateway は、その認証済み Node 接続に完全に一致するタイムスタンプのみを消去します。
この確認済みアクションより前の Gateway は未処理として返すため、Mac
Node は一度再接続し、切断時のクリーンアップによって古い接続状態を削除します。
選択、プライバシー、モデルコンテキスト、および通知ルーティングの動作については、
[アクティブなコンピューターのプレゼンス](/ja-JP/nodes/presence)を参照してください。

### Node のバックグラウンド生存イベント

Node は、ペアリングされた Node を接続済みとしてマークせずに、バックグラウンド復帰中に
稼働していたことを記録するため、`event: "node.presence.alive"` を指定して `node.event` を呼び出します。

```json
{
  "event": "node.presence.alive",
  "payloadJSON": "{\"trigger\":\"silent_push\",\"sentAtMs\":1737264000000,\"displayName\":\"Peter's iPhone\",\"version\":\"2026.4.28\",\"platform\":\"iOS 18.4.0\",\"deviceFamily\":\"iPhone\",\"modelIdentifier\":\"iPhone17,1\",\"pushTransport\":\"relay\"}"
}
```

`trigger` は、`background`、`silent_push`、`bg_app_refresh`、
`significant_location`、`manual`、`connect` からなる閉じた列挙型です。不明な値は
`background`（`src/shared/node-presence.ts`）に正規化されます。このイベントは、
認証済みの Node デバイスセッションに対してのみ永続化されます。デバイスのないセッションまたはペアリングされていないセッションは、
`handled: false` を返します。

成功した Gateway は構造化された結果を返します。

```json
{
  "ok": true,
  "event": "node.presence.alive",
  "handled": true,
  "reason": "persisted"
}
```

古い Gateway では、`node.event` に対して `{ "ok": true }` のみが返される場合があります。これは永続的なプレゼンスの保存ではなく、確認済みの RPC として扱ってください。

## ブロードキャストイベントのスコープ

サーバーからプッシュされるブロードキャストイベントはスコープによって制限されるため、ペアリングスコープまたは Node 専用のセッションがセッション内容を受動的に受信することはありません
（`src/gateway/server-broadcast.ts`）：

- チャット、エージェント、ツール結果のフレーム（ストリーミングされる `agent` イベント、ツール結果イベント）には、少なくとも `operator.read` が必要です。これを持たないセッションでは、これらのフレームが完全にスキップされます。
- Plugin が定義する `plugin.*` ブロードキャストは、デフォルトで `operator.write` または `operator.admin` に制限されます。`plugin.approval.requested` / `plugin.approval.resolved` などの明示的なエントリでは、代わりに `operator.approvals` が使用されます。
- ステータス／トランスポートイベント（`heartbeat`、`presence`、`tick`、接続／切断ライフサイクル）は制限されないため、認証済みのすべてのセッションからトランスポートの健全性を監視できます。
- 未知のブロードキャストイベントファミリーは、登録済みのハンドラーが明示的に制限を緩和しない限り、デフォルトでスコープによって制限されます（フェイルクローズ）。

各クライアント接続はクライアントごとに独自のシーケンス番号を保持するため、クライアントごとにイベントストリームの異なるスコープフィルタ済みサブセットが表示される場合でも、そのソケット上のブロードキャストは単調増加の順序を維持します。

## RPC メソッドファミリー

`hello-ok.features.methods` は、`src/gateway/server-methods-list.ts` と、読み込まれた Plugin／チャンネルのメソッドエクスポートから構築される控えめな検出リストです。これはすべてのメソッドを生成して列挙したものではなく、一部のメソッド（たとえば `push.test`、`web.login.start`、`web.login.wait`、`sessions.usage`）は実在して呼び出し可能であっても、意図的に検出対象から除外されています。これは `src/gateway/server-methods/*.ts` の完全な列挙ではなく、機能検出として扱ってください。

<AccordionGroup>
  <Accordion title="システムとアイデンティティ">
    - `health` は、キャッシュ済みまたは新たにプローブされた Gateway の健全性スナップショットを返します。
    - `diagnostics.stability` は、最近の診断安定性レコーダーを上限付きで返します。内容は、イベント名、件数、バイトサイズ、メモリ測定値、キュー／セッション状態、チャンネル／Plugin 名、セッション ID です。チャットテキスト、Webhook 本文、ツール出力、生のリクエスト／レスポンス本文、トークン、Cookie、シークレットは含まれません。`operator.read` が必要です。
    - `status` は、`/status` 形式の Gateway サマリーを返します。機密フィールドは、管理者スコープを持つオペレータークライアントにのみ返されます。
    - `gateway.identity.get` は、リレーおよびペアリングフローで使用される Gateway のデバイスアイデンティティを返します。
    - `system-presence` は、接続中のオペレーター／Node デバイスの現在のプレゼンススナップショットを返します。
    - `system-event` は、システムイベントを追加し、プレゼンスコンテキストを更新／ブロードキャストできます。
    - `last-heartbeat` は、永続化された最新の Heartbeat イベントを返します。
    - `set-heartbeats` は、Gateway での Heartbeat 処理を切り替えます。
    - `gateway.suspend.prepare` は、追跡対象の Gateway 処理がアイドル状態の場合にのみ、短時間の協調的な一時停止リースを作成します。`gateway.suspend.status` はそのリースを確認し、`gateway.suspend.resume` は再開後またはホスト操作の中止後にリースを解放します。

  </Accordion>

  <Accordion title="モデルと使用量">
    - `models.list` は、ランタイムで許可されているモデルカタログを返します。後述の「`models.list` ビュー」を参照してください。
    - `usage.status` は、プロバイダーの使用期間／残りクォータのサマリーを返します。
    - `usage.cost` は、指定した日付範囲のコスト使用量の集計サマリーを返します。1 つのエージェントには `agentId` を渡し、設定済みのエージェントを集計するには `agentScope: "all"` を渡します。
    - `doctor.memory.status` は、アクティブなデフォルトエージェントのワークスペースについて、ベクトルメモリ／キャッシュ済み埋め込みの準備状況を返します。明示的に稼働中の埋め込みプロバイダーへ ping する場合にのみ、`{ "probe": true }` または `{ "deep": true }` を渡します。Dreaming ストアの統計を 1 つのエージェントワークスペースに限定するには `{ "agentId": "agent-id" }` を渡します。省略すると、設定済みの Dreaming ワークスペースが集計されます。
    - `doctor.memory.dreamDiary`、`doctor.memory.backfillDreamDiary`、`doctor.memory.resetDreamDiary`、`doctor.memory.resetGroundedShortTerm`、`doctor.memory.repairDreamingArtifacts`、`doctor.memory.dedupeDreamDiary` は、オプションの `{ "agentId": "agent-id" }` を受け付けます。省略した場合、設定済みのデフォルトエージェントワークスペースを対象に動作します。
    - `doctor.memory.remHarness` は、リモートのコントロールプレーンクライアント向けに、上限付きの読み取り専用 REM ハーネスプレビューを返します。これには、ワークスペースパス、メモリの抜粋、根拠に基づいてレンダリングされた Markdown、詳細な昇格候補が含まれます。`operator.read` が必要です。
    - `sessions.usage` は、セッションごとの使用量サマリーを返します。1 つのエージェントには `agentId` を渡し、設定済みのエージェントをまとめて一覧表示するには `agentScope: "all"` を渡します。
      どちらの使用量メソッドも、夏時間を考慮した暦日の境界とバケットのために、IANA `timeZone` を含む `mode: "specific"` を受け付けます。`utcOffset` は、古いクライアント向け、および Gateway ランタイムが要求されたゾーンを認識しない場合のフォールバックとして引き続きサポートされます。
    - `sessions.usage.timeseries` は、1 つのセッションの時系列使用量を返します。
    - `sessions.usage.logs` は、1 つのセッションの使用量ログエントリを返します。

  </Accordion>

  <Accordion title="チャンネルとログインヘルパー">
    - `channels.status` は、組み込みおよびバンドル済みチャンネル／Plugin のステータスサマリーを返します。
    - `channels.logout` は、チャンネルが対応している場合に、指定したチャンネル／アカウントからログアウトします。
    - `web.login.start` は、現在の QR 対応 Web チャンネルプロバイダーの QR／Web ログインフローを開始します。
    - `web.login.wait` は、そのフローが完了するまで待機し、成功するとチャンネルを開始します。
    - `push.test` は、登録済みの iOS Node にテスト用 APNs プッシュを送信します。
    - `voicewake.get` は、保存済みのウェイクワードトリガーを返します。
    - `voicewake.set` は、ウェイクワードトリガーを更新し、変更をブロードキャストします。

  </Accordion>

  <Accordion title="Plugin 管理">
    - `plugins.list`（`operator.read`）は、インストール済み Plugin の一覧に加え、ローカルで選定された公式推奨項目、診断情報、および現在のインストールモードで変更が許可されているかどうかを返します。
    - `plugins.search`（`operator.read`）は、インストール可能な ClawHub のコード Plugin およびバンドル Plugin ファミリーを検索します。空でない `query` と、任意で 1～100 の `limit` を渡します。
    - `plugins.install`（`operator.admin`）は、`{ source: "official", pluginId }` を使用して公式カタログエントリを、または `{ source: "clawhub", packageName, version?, acknowledgeClawHubRisk? }` を使用して ClawHub パッケージをインストールします。ClawHub のインストールでは、Gateway の信頼性、整合性、およびインストールポリシーのチェックが維持されます。インストールが成功した場合は、Gateway の再起動が必要です。
    - `plugins.setEnabled`（`operator.admin`）は、`{ pluginId, enabled }` を使用して、インストール済みの 1 つの Plugin の有効化ポリシーを変更します。応答には、更新されたカタログエントリ、再起動メタデータ、およびスロット選択に関する警告が含まれます。
    - `plugins.uninstall`（`operator.admin`）は、`{ pluginId }` を使用して、外部からインストールされた 1 つの Plugin を削除します。削除対象には、設定参照、インストール記録、および管理対象ファイルが含まれます。バンドル済み Plugin はアンインストールできず、無効化のみ可能です。応答には削除アクションが一覧表示され、常に Gateway の再起動が必要です。

  </Accordion>

  <Accordion title="メッセージングとログ">
    - `send` は、チャットランナー外でチャネル、アカウント、スレッドを対象に送信するための、直接のアウトバウンド配信 RPC です。
    - `logs.tail` は、カーソル、上限、および最大バイト数の制御を使用して、設定済みの Gateway ファイルログの末尾を返します。

  </Accordion>

  <Accordion title="オペレーター端末">
    - `terminal.open` は、明示的な `agentId` またはデフォルトエージェント用のホスト PTY を開始し、解決されたエージェント、作業ディレクトリ、シェル、および隔離状態を返します。
    - `terminal.input`、`terminal.resize`、および `terminal.close` は、呼び出し元の接続が所有するセッションのみを操作します。
    - `terminal.upload` は、最大 16 MiB の base64 ファイルを 1 つ受け取り、セッションの Gateway またはペアリング済み Node のホスト上にある非公開の 24 時間一時ディレクトリに配置し、絶対パスを返します。呼び出し元は、そのパスを貼り付けるなどして使用する必要があります。RPC が端末入力を書き込んだり、コマンドを実行したりすることはありません。
    - `terminal.data` および `terminal.exit` イベントは、セッションを所有する接続にのみストリーミングされます。
    - 接続が切断されたセッションは終了されず、デタッチされます。セッションは `gateway.terminal.detachedSessionTimeoutSeconds`（デフォルトは 300。`0` を指定すると切断時終了に戻ります）の間、再アタッチ可能な状態を保ち、その間、直近の出力は容量制限付きのサーバー側バッファーに蓄積されます。
    - `terminal.list` はアタッチ可能なセッションを返します。`terminal.attach` は、実行中またはデタッチ済みのセッションを呼び出し元の接続に再バインドし、再生バッファーを返します（tmux 形式の引き継ぎです。以前の実行中の所有者は、理由 `detached` とともに `terminal.exit` を受信します）。`terminal.text` は、アタッチせずにバッファーをプレーンテキストとして読み取ります。
    - すべての端末メソッドには `operator.admin` が必要です。`gateway.terminal.enabled` は明示的に true にする必要があります。完全にサンドボックス化されたエージェントは拒否され、エージェントポリシーが変更されると、既存および処理中の PTY が、デタッチ済みのものも含めて閉じられます。

  </Accordion>

  <Accordion title="通話と TTS">
    - `talk.catalog` は、音声、ストリーミング文字起こし、およびリアルタイム音声用の読み取り専用 Talk プロバイダーカタログを返します。これには、正規プロバイダー ID、レジストリエイリアス、ラベル、設定状態、任意のグループレベルの `ready` の結果、公開されたモデル／音声 ID、正規モード、トランスポート、ブレイン戦略、およびリアルタイム音声／機能フラグが含まれますが、プロバイダーのシークレットを返したり、グローバル設定を変更したりすることはありません。現在の Gateway は、ランタイムのプロバイダー選択を適用した後に `ready` を設定します。古い Gateway でこれが存在しない場合は、未検証として扱ってください。
    - `talk.config` は有効な Talk 設定ペイロードを返します。`includeSecrets` には `operator.talk.secrets`（または `operator.admin`）が必要です。
    - `talk.session.create` は、`realtime/gateway-relay`、`transcription/gateway-relay`、または `stt-tts/managed-room` 用に、Gateway が所有する Talk セッションを作成します。`stt-tts/managed-room` では、`sessionKey` を渡す `operator.write` 呼び出し元は、スコープ付きセッションキーの可視性のために `spawnedBy` も渡す必要があります。スコープなしの `sessionKey` の作成と `brain: "direct-tools"` には `operator.admin` が必要です。
    - `talk.session.join` は、管理対象ルームのセッショントークンを検証し、必要に応じて `session.ready` または `session.replaced` を発行し、ルーム／セッションのメタデータと最近の Talk イベントを返します。平文のトークンやそのハッシュを返すことはありません。
    - `talk.session.appendAudio` は、base64 PCM 入力音声を、Gateway が所有するリアルタイムリレーおよび文字起こしセッションに追加します。
    - `talk.session.startTurn`、`talk.session.endTurn`、および `talk.session.cancelTurn` は、状態をクリアする前に古いターンを拒否しつつ、管理対象ルームのターンライフサイクルを制御します。
    - `talk.session.cancelOutput` は、主に Gateway リレーセッションで VAD により制御される割り込みのために、アシスタントの音声出力を停止します。
    - `talk.session.submitToolResult` は、Gateway が所有するリアルタイムリレーセッションによって発行されたプロバイダーツール呼び出しを完了します。リクエストは、プロバイダーブリッジが公開する非同期完了シグナルがある場合、そのシグナルを待機します。送信に失敗した場合、リンクされた実行はアクティブなままとなり、成功したツール結果イベントは発行されません。途中経過のツール出力には `options: { willContinue: true }` を渡します。プロバイダーブリッジが抑制をサポートしており、結果によって別の応答を開始すべきでない場合は、`options: { suppressResponse: true }` を渡します。
    - `talk.session.steer` は、Gateway が所有するエージェントベースの Talk セッションに、アクティブな実行の音声制御を送信します。形式は `{ sessionId, text, mode? }` で、`mode` は `status`、`steer`、`cancel`、または `followup` です。モードを省略すると、発話テキストから分類されます。
    - `talk.session.close` は、Gateway が所有するリレー、文字起こし、または管理対象ルームのセッションを閉じ、終端 Talk イベントを発行します。
    - `talk.mode` は、WebChat／Control UI クライアント向けに現在の Talk モード状態を設定し、ブロードキャストします。
    - `talk.client.create` は、`webrtc` または `provider-websocket` を使用して、クライアントが所有するリアルタイムプロバイダーセッションを作成または再開します。この間、Gateway は認証情報、指示、ツールポリシー、および返される `voiceSessionId` を管理します。クライアントは `sessionKey` を渡し、1 回の通話中にプロバイダートランスポートを置き換える際は `voiceSessionId` を再利用します。
    - `talk.client.transcript` は、確定済みの `{ role, text }` 項目を 1 つ、通常のエージェントセッションに追加します。必須の `entryId` は `voiceSessionId` 内でべき等であり、再試行してもトランスクリプトメッセージは重複しません。
    - `talk.client.close` は、保留中のトランスクリプト書き込み後に論理音声セッションを閉じます。クローズ処理はべき等であり、変更のみを含む通話ダイジェストをセッションの最後の非 WebChat チャネルに配信する場合があります。
    - `talk.client.toolCall` により、クライアントが所有するリアルタイムトランスポートは、プロバイダーツール呼び出しを Gateway ポリシーに転送できます。最初にサポートされるツールは `openclaw_agent_consult` です。クライアントは実行 ID を受け取り、通常のチャットライフサイクルイベントを待ってから、プロバイダー固有のツール結果を送信します。音声に紐付いた影響の大きいアクションは、その正確なアクションを後続の確定済みユーザー発話が明示的に確認し、次の問い合わせで `confirmationId` が指定されるまで、`VOICE_CONFIRMATION_REQUIRED:<id>` を返します。
    - `talk.client.steer` は、クライアントが所有するリアルタイムトランスポートに、アクティブな実行の音声制御を送信します。Gateway は `sessionKey` からアクティブな埋め込み実行を解決し、ステアリングを暗黙に破棄するのではなく、承認／拒否を示す構造化された結果を返します。
    - `talk.event` は、リアルタイム、文字起こし、STT／TTS、管理対象ルーム、テレフォニー、および会議アダプター向けの単一の Talk イベントチャネルです。
    - `talk.speak` は、アクティブな Talk 音声プロバイダーを通じて音声を合成します。
    - `tts.status` は、TTS の有効状態、アクティブなプロバイダー、フォールバックプロバイダー、およびプロバイダー設定状態を返します。
    - `tts.providers` は、表示可能な TTS プロバイダー一覧を返します。
    - `tts.enable` および `tts.disable` は、TTS 設定状態を切り替えます。
    - `tts.setProvider` は、優先する TTS プロバイダーを更新します。
    - `tts.convert` は、1 回限りのテキスト読み上げ変換を実行します。
    - `tts.speak`（`operator.write`）は、設定済みの汎用 TTS プロバイダーチェーンを使用して、空でない `text` をレンダリングし、クリップ全体を `audioBase64` としてインラインで返します。さらに、`provider` と、任意の `outputFormat`、`mimeType`、および `fileExtension` メタデータも返します。`tts.convert` とは異なり、Gateway ローカルのパスは返しません。`talk.speak` とは異なり、Talk プロバイダーは必要ありません。`tts.maxTextLength` を超えるテキストでは `INVALID_REQUEST` が返され、音声合成に失敗した場合は `UNAVAILABLE` が返されます。

  </Accordion>

  <Accordion title="シークレット、設定、更新、ウィザード">
    - `secrets.reload` は、アクティブな SecretRef を再解決し、所有者を考慮したランタイム状態をアトミックに公開します。対象となる所有者の失敗は、`warningCount` とともにコールドまたはステイルの縮退状態として公開できます。厳格な失敗またはマッピングされていない失敗の場合、再読み込みを拒否し、アクティブなスナップショットを維持します。
    - `secrets.resolve` は、特定のコマンド／ターゲットセットに対するコマンドターゲットのシークレット割り当てを解決します。
    - `config.get` は、現在のディスク上の設定スナップショット、未加工のルートファイル `hash`、解決済みの `configRevisionHash`、およびアクティブな Gateway ランタイムが受け入れた解決済みリビジョンのオプションの `appliedConfigHash` を返します。
    - `config.set` は、検証済みの設定ペイロードを書き込みます。
    - `config.patch` は、部分的な設定更新をマージします。破壊的な配列置換では、影響を受けるパスを `replacePaths` に指定する必要があります。配列エントリ内のネストされた配列には、`agents.entries.*.skills` などの `[]` パスを使用します。
    - `config.apply` は、設定ペイロード全体を検証して置き換えます。
    - `config.schema` は、Control UI と CLI ツールで使用されるライブ設定スキーマのペイロードを返します。これには、スキーマ、`uiHints`、バージョン、生成メタデータ、および読み込み可能な場合は Plugin とチャンネルのスキーマメタデータが含まれます。また、該当するフィールドドキュメントが存在する場合、UI と同じラベル／ヘルプテキストに基づく `title`／`description` メタデータが含まれ、ネストされたオブジェクト、ワイルドカード、配列項目、および `anyOf`／`oneOf`／`allOf` の合成分岐にも対応します。
    - `config.schema.lookup` は、1 つの設定パスに対するパススコープの検索ペイロードを返します。これには、正規化されたパス、浅いスキーマノード、一致したヒントと `hintPath`、オプションの `reloadKind`、および UI／CLI のドリルダウン用の直下の子要素の概要が含まれます。`reloadKind` は `restart`、`hot`、または `none`（`src/config/schema.ts`）のいずれかで、要求されたパスに対する Gateway 設定再読み込みプランナーを反映します。検索スキーマノードは、ユーザー向けドキュメントと一般的な検証フィールド（`title`、`description`、`type`、`enum`、`const`、`format`、`pattern`、数値／文字列／配列／オブジェクトの境界、`additionalProperties`、`deprecated`、`readOnly`、`writeOnly`）を保持します。子要素の概要では、`key`、正規化された `path`、`type`、`required`、`hasChildren`、オプションの `reloadKind`、および一致した `hint`／`hintPath` を公開します。
    - `update.run` は Gateway の更新フローを実行し、更新が成功した場合にのみ再起動をスケジュールします。セッションを持つ呼び出し元は `continuationMessage` を含めることで、起動時に再起動継続キューを介して後続のエージェントターンを 1 回再開できます。コントロールプレーンからのパッケージマネージャー更新および監視対象の git チェックアウト更新では、稼働中の Gateway 内でパッケージツリーを置き換えたり、チェックアウト／ビルド出力を変更したりする代わりに、切り離されたマネージドサービスへのハンドオフを使用します。開始されたハンドオフは、`result.reason: "managed-service-handoff-started"` および `handoff.status: "started"` とともに `ok: true` を返します。同じ Gateway プロセスによって処理される 2 つ目の同時実行 `update.run` は、`result.reason: "managed-service-handoff-already-running"` および `handoff.status: "already-running"` とともに `ok: false` を返します。その継続は受け入れられないため、呼び出し元はアクティブな更新の完了後に再試行できます。スタンドアロンの CLI アップデーターと置換後の Gateway プロセスは、このプロセスローカルなガードの対象外です。利用不能または失敗したハンドオフは、`managed-service-handoff-unavailable` または `managed-service-handoff-failed` とともに `ok: false` を返し、手動のシェル更新が必要な場合はさらに `handoff.command` を返します。「利用不能」とは、systemd の `OPENCLAW_SYSTEMD_UNIT` など、OpenClaw に安全なスーパーバイザー境界または永続的なサービス ID がないことを意味します。開始されたハンドオフ中、再起動センチネルが一時的に `stats.reason: "restart-health-pending"` を報告する場合があります。CLI が再起動後の Gateway を検証し、最終的な `ok` センチネルを書き込むまで、継続は遅延されます。
    - `update.status` は、更新用の再起動センチネルを更新して返します。利用可能な場合は、再起動後に稼働しているバージョンも含まれます。
    - `wizard.start`、`wizard.next`、`wizard.status`、および `wizard.cancel` は、WS RPC 経由でオンボーディングウィザードを公開します。

  </Accordion>

  <Accordion title="エージェントとワークスペースのヘルパー">
    - `agents.list` は、Gateway から可視のエージェントエントリを返します。これには、有効なモデル／ランタイムメタデータと、オプションのセマンティック `kind`（`agent` または `system`）が含まれます。クライアントは `agent-kind` ハンドシェイク機能を通知することで、完全な型付き名簿を受け取ります。この機能を持たないクライアントには、システム行を含まない従来のセレクター安全な名簿が維持されます。種別を認識するクライアントは、`system` 行を診断ビューには保持しつつ、通常のセレクターから除外します。古い v4 Gateway は、`kind` のない行を返す場合があります。
    - `agents.create`、`agents.update`、および `agents.delete` は、エージェントレコードとワークスペースの接続を管理します。
    - `agents.files.list`、`agents.files.get`、および `agents.files.set` は、エージェントに公開されるブートストラップ用ワークスペースファイルを管理します。
    - `audit.activity.list` は、バージョン管理されたメタデータ専用のアクティビティ台帳を返します。`audit.list` は、引き続き互換性を保つ run/tool RPC です。
    - `agents.workspace.list` および `agents.workspace.get`（`operator.read`）は、[オペレータースコープ](/ja-JP/gateway/operator-scopes)で説明されている信頼済みオペレータードメイン内のクライアント向けに、エージェントのワークスペースディレクトリを読み取り専用かつページ分割して閲覧できるようにします。リクエストではワークスペース相対パスのみを受け付けます。読み取りは実パス解決済みのワークスペースルート内に限定され（シンボリックリンクおよびハードリンクによる脱出は拒否）、サイズ上限が適用され、UTF-8 テキストと一般的な画像形式（base64）のみに制限されます。レスポンスでは、ホスト上のワークスペースパスを公開しません。この名前空間には書き込み操作はありません。
    - `tasks.list`、`tasks.get`、および `tasks.cancel` は、Gateway のタスク台帳を SDK とオペレータークライアントに公開します。後述の[タスク台帳 RPC](#task-ledger-rpcs)を参照してください。
    - `artifacts.list`、`artifacts.get`、および `artifacts.download` は、明示的な `sessionKey`、`runId`、または `taskId` スコープに対して、トランスクリプトから派生したアーティファクトの概要とダウンロードを公開します。実行およびタスクのクエリでは、所有するセッションをサーバー側で解決し、出所が一致するトランスクリプトメディアのみを返します。安全でない URL ソースまたはローカル URL ソースについては、サーバー側で取得せず、未対応のダウンロードとして返します。
    - `environments.list` および `environments.status` は、Gateway ローカル環境と Node 環境の検出を維持します。設定済みのクラウドワーカーと以前のプロファイルが残した永続レコードには、`providerId`、オプションの `leaseId`、`state`、`ageMs`、オプションの `idleMs`、および `attachedSessionIds` を含む `worker` メタデータが追加されます。ワーカーのライフサイクル状態は、`requested`、`provisioning`、`bootstrapping`、`ready`、`attached`、`idle`、`draining`、`destroying`、`destroyed`、`failed`、および `orphaned` です。
    - `environments.create`（`{ profileId, idempotencyKey }`）は、設定済みの Plugin プロバイダープロファイルからワーカーをプロビジョニングします。同じキーで再試行すると、永続化された操作が再利用されます。`environments.destroy`（`{ environmentId }`）は、永続的なワーカー環境のべき等な破棄を要求します。どちらも `operator.admin` が必須で、コントロールプレーンへの書き込みであり、ステータスレスポンスで使用されるものと同じ形式の環境概要を返します。
    - `agent.identity.get` は、エージェントまたはセッションに対する有効なアシスタント ID を返します。
    - `agent.wait` は、実行が完了するまで待機し、利用可能な場合は終了時のスナップショットを返します。

  </Accordion>

  <Accordion title="セッション制御">
    - `sessions.list` は、エージェントランタイムバックエンドが構成されている場合の行ごとの `agentRuntime` メタデータを含む、現在のセッションインデックスを返します。クラウドワーカー配置が有効な場合、または永続的な復旧状態が存在する場合、セッション行には、クローズ済みの `placement` 状態（`local`、`requested`、`provisioning`、`syncing`、`starting`、`active`、`draining`、`reconciling`、`reclaimed`、または `failed`）に加えて、状態固有の環境、所有者エポック、ワークスペース、バンドル、ACK カーソル、または復旧フィールドも含まれます。
    - `sessions.subscribe` と `sessions.unsubscribe` は、現在の WS クライアントに対するセッション変更イベントのサブスクリプションを切り替えます。
    - `sessions.messages.subscribe` と `sessions.messages.unsubscribe` は、1 つのセッションに対するトランスクリプト／メッセージイベントのサブスクリプションを切り替えます。永続化された対象者にそのセッションが厳密に含まれ、かつレビュー担当者のバインディングによって購読クライアントが承認されている承認について、サニタイズ済みの `session.approval` ライフサイクルイベントも受信するには、`includeApprovals: true` を渡します。その場合、購読応答には上限付きの保留中 `approvalReplay` が含まれます。これは `truncated` が false の場合に正式な情報となります。このオプトインは購読呼び出しごとに適用され、固定されません。同じセッションを `includeApprovals: true` なしで再購読すると、既存の承認サブスクリプションが削除されます。通常のセッション読み取り権限に加えて、このオプトインには `operator.admin`、またはペアリング済みデバイス上の `operator.approvals` が必要です。
    - `sessions.preview` は、指定されたセッションキーの上限付きトランスクリプトプレビューを返します。
    - `sessions.describe` は、厳密に一致するセッションキーについて 1 つの Gateway セッション行を返します。
    - `sessions.resolve` は、セッションターゲットを解決または正規化します。
    - `sessions.create` は、新しいセッションエントリを作成します。省略可能な `model` と `thinkingLevel` の値は、初期モデルと推論のオーバーライドをアトミックに永続化します。`worktree: true` は管理対象ワークツリーをプロビジョニングします。省略可能な `worktreeBaseRef`/`worktreeName` はベース ref とブランチ名を選択し、`execNode`（`operator.admin`）はセッションの実行を Node ホストにバインドします。作成されたワークツリーは結果に反映され、セッション行（`worktree: { id, branch, repoRoot }`）に永続化されます。エントリは作成されたものの、ネストされた初期 `chat.send` が拒否された場合、成功結果には `runStarted: false` と `runError` が含まれます。クライアントはプロンプトを保持し、返されたセッションキーに対して再試行できます。`parentSessionKey` を `emitCommandHooks: true` とともに渡す呼び出し元は、別個の子のライフサイクル上の処置も宣言する必要があります。`succeedsParent: true` は親を `session_end` で終了し、`false` は親をアクティブなまま維持し、子の `session_start` のみを発行します。`succeedsParent` を省略すると、既存クライアント向けの従来の親ロールオーバー動作が維持されます。この処置には親とのリンクとコマンドフックの両方が必要です。フォークはその親を成功状態にできません。別個の子は作成されないため、メインセッションのその場でのリセット動作は変わりません。新しい行には、信頼された作成境界から書き込み一回限りの作成来歴（`createdVia`、`createdActor`、`createdAt`）が付与されます。既存キーを採用しても、再度付与されることはありません。人間のプロファイルアクターの場合、行の投影時に `createdActor.label` が現在のユーザープロファイルから解決され、セッションエントリには一切保存されないため、プロファイル名を変更しても不整合は生じません。セッション行には、`parentSessionKey`（ナビゲーション上の親、永続化）、`controlOwnerSessionKey`（稼働中のランタイムコントローラー）、`forkSource`（フォーク元の厳密なキーとトランスクリプト世代）、および `previousSessionId`（同じキーの以前のトランスクリプト世代）も含まれます。
    - `sessions.dispatch`（`operator.admin`）は、セッション所有の管理対象ワークツリーを持つ既存のローカル OpenClaw セッションを、構成済みのクラウドワーカープロファイルへ移動します。`{ key, profileId, agentId? }` を渡します。ワーカープロファイルが構成されていない場合、このメソッドは存在しません。アクティブな作業をドレインする前にローカルでのターン受け付けを閉じ、配置が `active` のワーカー所有権に達した後にのみ返ります。ディスパッチは一方向です。ワーカーからローカルへの引き戻しは、この RPC の対象外です。
    - `sessions.groups.list`、`sessions.groups.put`、`sessions.groups.rename`、および `sessions.groups.delete` は、Gateway が所有するカスタムセッショングループカタログ（名前と表示順）を管理します。メンバーシップは各セッションの `category` フィールドに保持され、名前変更と削除では、メンバーセッションがサーバー側で更新されます。
    - `sessions.send` は、既存のセッションにメッセージを送信します。
    - `sessions.steer` は、アクティブなセッション向けの中断および誘導バリアントです。
    - `sessions.abort` は、セッションのアクティブな作業を中止します。`key` と省略可能な `runId` を渡すか、Gateway がセッションを解決できるアクティブな実行については `runId` のみを渡します。`runId` を指定すると、キャンセルの範囲がその実行に限定されます。キーのみを指定した非グローバルリクエストで `clearQueued: true` を設定すると、そのセッションが所有するフォローアップキューとレーンキューも破棄されます。`clearQueued` を省略する既存の呼び出し元では、これらのキューが維持されます。リテラルの `global` キーでは、既存のエージェント修飾付き `chat.abort` 所有権ルールが維持され、非グローバルなフォローアップまたはレーンのクリーンアップは実行されません。
    - `sessions.patch` は、セッションのメタデータ／オーバーライドを更新し、解決された正規モデルと有効な `agentRuntime` を報告します。生成系統（`spawnedBy`、`spawnedWorkspaceDir`、`spawnedCwd`、`spawnDepth`、`subagentRole`、`subagentControlScope`）は、公開 API からパッチできなくなりました。これらの情報は信頼された作成パスによって一度だけ書き込まれ、引き続き送信するリクエストは拒否されます。
    - `sessions.reset`、`sessions.delete`、および `sessions.compact` は、セッションのメンテナンスを実行します。
    - `sessions.get` は、保存されている完全なセッション行を返します。
    - チャット実行では、引き続き `chat.history`、`chat.send`、`chat.abort`、および `chat.inject` を使用します。`chat.history` は UI クライアント向けに表示正規化されます。インラインディレクティブタグは表示テキストから除去され、プレーンテキストのツール呼び出し XML ペイロード（`<tool_call>...</tool_call>`、`<function_call>...</function_call>`、`<tool_calls>...</tool_calls>`、`<function_calls>...</function_calls>`、および途中で切れたツール呼び出しブロック）と漏洩した ASCII／全角のモデル制御トークンも除去されます。サイレントトークンのみのアシスタント行（厳密に `NO_REPLY` / `no_reply`）は省略され、サイズ超過の行はプレースホルダーに置き換えられることがあります。
    - `chat.message.get` は、単一の表示可能なトランスクリプトエントリ向けに追加された、上限付きの完全メッセージ読み取り機能です。`sessionKey`、セッション選択がエージェントスコープの場合は省略可能な `agentId`、および `chat.history` を通じて以前に公開されたトランスクリプトの `messageId` を渡します。保存済みエントリが引き続き利用可能でサイズ超過でない場合、Gateway は軽量履歴の切り詰め上限なしで、同じ表示正規化済み投影を返します。
    - `chat.toolTitles` は、Control UI に表示されるツール呼び出しの短い目的タイトルを返します（バッチ処理、入力サイズに上限がある最大 24 項目）。この機能は `gateway.controlUi.toolTitles` によるオプトイン方式です（デフォルトは無効）。無効な Gateway はモデルを呼び出さずに `{ titles: {}, disabled: true }` と応答するため、クライアントは問い合わせを停止します。有効な場合、タイトルには標準のユーティリティモデルルーティングが使用されます。明示的に構成された `utilityModel`（すべてのユーティリティタスクと同様に、選択したプロバイダーへ上限付きのタスク内容を送信する可能性がある運用者の判断）があればそれを使用し、なければセッションプロバイダーが宣言した小規模モデルのデフォルトを使用するため、新しい送信先が暗黙的に追加されることはありません。空の `utilityModel` を指定すると完全に無効になります。タイトルがプライマリモデルへフォールバックすることはありません。結果は、ツール名と入力をキーとしてエージェントごとの状態データベースにキャッシュされるため、同じ呼び出しを繰り返し表示しても再課金されることはありません。
    - `chat.send` は、1 ターン限りの `fastMode: "auto"` を受け付け、自動カットオフより前に開始されたモデル呼び出しでは高速モードを使用し、それ以降に開始される再試行、フォールバック、ツール結果、または継続の呼び出しでは高速モードを使用しません。カットオフのデフォルトは 60 秒（`DEFAULT_FAST_MODE_AUTO_ON_SECONDS`）で、`agents.defaults.models["<provider>/<model>"].params.fastAutoOnSeconds` を使用してモデルごとに構成できます。`chat.send` の呼び出し元は、1 ターン限りの `fastAutoOnSeconds` を渡して、そのリクエストのカットオフをオーバーライドできます。このリクエストに限り保存済みのキューモードをオーバーライドするには、`queueMode`（`steer`、`followup`、`collect`、または `interrupt`）を渡します。明示的な Control UI の誘導アクションでは `queueMode: "steer"` を使用します。対話型クライアントは、表示中のアクティブなトランスクリプトブランチのリーフを `expectedLeafEntryId` で渡すか、正式に空のトランスクリプトを `null` で渡すことができます。別のクライアントが先にブランチを切り替えた場合、Gateway は送信を `details.reason: "active-leaf-changed"` で拒否します。

  </Accordion>

  <Accordion title="デバイスのペアリングとデバイストークン">
    - `device.pair.list` は、保留中および承認済みのペアリング済みデバイスを返します。
    - `device.pair.setupCode` は、モバイルセットアップコードと、デフォルトでは PNG QR データ URL を作成します。これには `operator.admin` が必要で、公開されるディスカバリーからは意図的に除外されています。結果には、`setupCode`、省略可能な `qrDataUrl`、`gatewayUrl`、秘密ではない `auth` ラベル、および `urlSource` が含まれます。
    - `device.pair.approve`、`device.pair.reject`、および `device.pair.remove` は、デバイスペアリングレコードを管理します。
    - `device.pair.rename` は、クライアントから報告された表示名より優先され、デバイスの修復または再承認後も維持される運用者ラベル（`{ deviceId, label }`）を割り当てます。
    - `device.token.rotate` は、承認済みロールと呼び出し元のスコープ境界内で、ペアリング済みデバイスのトークンをローテーションします。
    - `device.token.revoke` は、承認済みロールと呼び出し元のスコープ境界内で、ペアリング済みデバイスのトークンを失効させます。

    セットアップコードには、有効期間の短いブートストラップ認証情報が埋め込まれています。クライアントは、
    ペアリングフローの範囲を超えてこれをログに記録したり、永続化したりしてはなりません。

  </Accordion>

  <Accordion title="Node のペアリング、呼び出し、および保留中の作業">
    - `node.pair.list`、`node.pair.approve`、`node.pair.reject`、および `node.pair.remove` は、Node の機能承認を扱います。`node.pair.request` と `node.pair.verify` は、スタンドアロンの Node ペアリングストアとともに 2026.7 で削除されました。保留中のリクエストは、Node の接続中に Gateway によって作成されます。
    - `node.list` と `node.describe` は、既知／接続済みの Node の状態を返します。
    - `node.rename` は、ペアリング済み Node のラベルを更新します。
    - `node.invoke` は、接続済み Node にコマンドを転送します。
    - `node.invoke.result` は、呼び出しリクエストの結果を返します。
    - `mcp.tools.call.v1` は、構成済みの Node ローカル MCP ツールを呼び出すための、ヘッドレス Node ホスト用コマンドです。これは `node.invoke` を通じて伝送され、Node がそのコマンドを宣言している必要があり、引き続きペアリング承認と `gateway.nodes.commands.deny` の対象となります。
    - `node.event` は、Node で発生したイベントを Gateway に返します。
    - `node.pluginTools.update` は、接続済み Node のエージェントから見える Plugin／MCP ツール記述子を置き換える唯一の公開パスです。`connect` パラメーターにはこれらは含まれません。
    - `node.pending.pull` と `node.pending.ack` は、接続済み Node のキュー API です。
    - `node.pending.enqueue` と `node.pending.drain` は、オフライン／切断済み Node の永続的な保留中作業を管理します。

  </Accordion>

  <Accordion title="承認ファミリー">
    - `approval.history` は、exec、Plugin、system-agent リクエストについて、30 日間保持される終了済み承認を新しい順で返します（スコープ `operator.approvals`）。カーソルページネーションとオプションの種別フィルターをサポートします。保留中の承認は履歴行には含まれません。
    - `approval.get` と `approval.resolve` は、種別に依存しない永続的な承認メソッドです（スコープ `operator.approvals`）。`approval.get` は、安定した `urlPath` を持つ、サニタイズ済みの保留中または保持された終了済みプロジェクションを返します。`approval.resolve` は、正規の承認 ID、明示的な `kind`、および決定を受け取り、最初の回答を優先する解決を適用し、記録された正規の結果を常に返します。
    - `exec.approval.request`、`exec.approval.get`、`exec.approval.list`、および `exec.approval.resolve` は、単発の exec 承認リクエストと、保留中の承認の検索／再生を扱います。これらは、同じ永続的承認レジストリ上のプロトコル境界アダプターです。
    - `exec.approval.waitDecision` は、1 件の保留中の exec 承認を待機し、最終決定（またはタイムアウト時は `null`）を返します。
    - `exec.approvals.get` と `exec.approvals.set` は、Gateway の exec 承認ポリシースナップショットを管理します。
    - `exec.approvals.node.get` と `exec.approvals.node.set` は、Node リレーコマンドを介して Node ローカルの exec 承認ポリシーを管理します。
    - `plugin.approval.request`、`plugin.approval.list`、`plugin.approval.waitDecision`、および `plugin.approval.resolve` は、Plugin 定義の承認フローを扱います。

  </Accordion>

  <Accordion title="Control UI コマンド">
    - `ui.command` を使用すると、`operator.write` 呼び出し元は、`ui-commands` 機能を通知する接続済みの Control UI クライアントに、型付きのレイアウトおよびナビゲーションコマンドを送信できます。
    - コマンドは、ペインの分割／閉じる／フォーカス、サイドバーの表示状態、ターミナル／ブラウザパネルの表示状態とドッキング、およびセッションナビゲーションを扱います。
    - プロトコル v1 は、意図的に、接続されている対応可能なすべての Control UI にファンアウトします。接続されているものがない場合、レイアウトが変更されたように装うのではなく、リクエストは `UNAVAILABLE` で失敗します。

  </Accordion>

  <Accordion title="自動化、Skills、ツール">
    - 自動化：`wake` は、即時または次回の Heartbeat でウェイクテキストを注入するようスケジュールします。`cron.get`、`cron.list`、`cron.status`、`cron.add`、`cron.update`、`cron.remove`、`cron.run`、`cron.runs` は、スケジュールされた作業を管理します。
    - `cron.run` は、手動実行用のエンキュー形式 RPC として維持されています。完了セマンティクスが必要なクライアントは、返された `runId` を読み取り、`cron.runs` をポーリングする必要があります。
    - `cron.runs` は、オプションの空でない `runId` フィルターを受け付けます。これにより、クライアントは、同じジョブの他の履歴エントリと競合せずに、キューに入れられた 1 件の手動実行を追跡できます。
    - Skills とツール：`commands.list`、`skills.*`、`tools.catalog`、`tools.effective`、`tools.invoke`。下記の[オペレーター向けヘルパーメソッド](#operator-helper-methods)を参照してください。

  </Accordion>
</AccordionGroup>

### 共通イベントファミリー

- `chat`：`chat.inject` などの UI チャット更新、およびトランスクリプトのみに関係するその他のチャット
  イベント。プロトコル v4 では、差分ペイロードに `deltaText` が含まれます。`message` は引き続き、
  アシスタントの累積スナップショットです。接頭辞ではない置換では
  `replace=true` が設定され、置換テキストとして `deltaText` が使用されます。
- `session.message`、`session.operation`、`session.tool`：購読中のセッションに関するトランスクリプト、実行中の
  セッション操作、およびイベントストリームの更新。
- `session.approval`：明示的にオプトインした完全一致セッション購読者向けの、サニタイズ済みの保留中および終了済み承認の正確な状態。
  子承認は永続化された祖先のオーディエンスを使用します。イベントがトランスクリプトを変更したり、エージェントをウェイクしたりすることはありません。
- `sessions.changed`：セッションインデックスまたはメタデータが変更されました。
- `presence`：システムプレゼンススナップショットの更新。
- `tick`：定期的なキープアライブ／稼働確認イベント。
- `health`：Gateway ヘルススナップショットの更新。
- `heartbeat`：Heartbeat イベントストリームの更新。
- `cron`：Cron 実行／ジョブ変更イベント。
- `shutdown`：Gateway シャットダウン通知。
- `node.pair.requested` / `node.pair.resolved`：Node ペアリングのライフサイクル。
- `node.invoke.request`：Node 呼び出しリクエストのブロードキャスト。
- `device.pair.requested` / `device.pair.resolved`：ペアリング済みデバイスのライフサイクル。
- `voicewake.changed`：ウェイクワードトリガー設定が変更されました。
- `config.changed`：設定の書き込みが永続化されました（ペイロードには設定パス、
  新しいスナップショットハッシュ、およびタイムスタンプが含まれます。設定内容は決して含まれません）。オペレーター読み取り
  スコープです。クライアントは `config.get` を介して更新します。
- `exec.approval.requested` / `exec.approval.resolved`：exec 承認の
  ライフサイクル。
- `plugin.approval.requested` / `plugin.approval.resolved`：Plugin 承認の
  ライフサイクル。

### Node ヘルパーメソッド

Node は、自動許可チェック用の Skills 実行可能ファイルの現在の一覧を取得するために、`skills.bins` を呼び出せます。

## 監査台帳 RPC

`audit.activity.list` は、エージェント実行、ツールアクション、およびオプトインされたメッセージのライフサイクルメタデータを、オペレータークライアント向けに安定した新しい順のビューで提供します。
`operator.read` が必要です。クエリは 30 日より古いレコードを除外し、共有
SQLite 台帳は 100,000 レコードに制限されます。期限切れの行は、
Gateway の起動時、1 時間ごとのメンテナンス時、および以降の書き込み時に削除されます。データモデルとプライバシーのセマンティクスについては、
[監査履歴](/ja-JP/gateway/audit)を参照してください。

- パラメーター：オプションの完全一致 `agentId`、`sessionKey`、または `runId`。オプションの `kind`
  （`"agent_run"`、`"tool_action"`、または `"message"`）。オプションの `status`
  （`"started"`、`"succeeded"`、`"failed"`、`"cancelled"`、`"timed_out"`、
  `"blocked"`、または `"unknown"`）。オプションのメッセージ `direction`（`"inbound"` または
  `"outbound"`）と完全一致 `channel`。オプションの包含的な `after` / `before`
  Unix ミリ秒境界。`1` から `500` までのオプションの `limit`。および前のページから取得したオプションの
  文字列 `cursor`。
- 結果：`{ "events": AuditActivityEventV1[], "nextCursor"?: string }`。

名前付き V1 結果ユニオンには、エージェント実行、ツールアクション、受信メッセージ、
および送信メッセージ用の個別のスキーマがあります。`eventType` 判別子はそれぞれ
`agent_run`、`tool_action`、`inbound_message`、または `outbound_message` です。`kind` と
メッセージ `direction` は、引き続きフィルタリングと表示に使用できます。すべてのイベントには
整数の `schemaVersion: 1` があります。メッセージ ID 参照は正確な
`hmac-sha256:v1:<32 hex key id>:<64 hex digest>` 形式を使用します。チャンネル送信者のアクター
ID も同じ形式を使用します。

すべてのバリアントには、`eventType`、`schemaVersion`、`eventId`、`sequence`、
`sourceSequence`、`occurredAt`、`kind`、`action`、`status`、`actor`、および
`redaction` が必要です。バリアントフィールドは次のとおりです。

| `eventType`        | 必須フィールド                                                   | オプションフィールド                                                                                                                 |
| ------------------ | ----------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------- |
| `agent_run`        | `agentId`、`runId`、`kind: "agent_run"`                           | `sessionKey`、`sessionId`、`errorCode`                                                                                          |
| `tool_action`      | `agentId`、`runId`、`kind: "tool_action"`                         | `sessionKey`、`sessionId`、`toolCallId`、`toolName`、`errorCode`                                                                |
| `inbound_message`  | `direction: "inbound"`、`channel`、`conversationKind`、`outcome`  | `agentId`、`runId`、`durationMs`、`resultCount`、ID 参照、`reasonCode`、`errorCode`                                 |
| `outbound_message` | `direction: "outbound"`、`channel`、`conversationKind`、`outcome` | `agentId`、`runId`、`durationMs`、`resultCount`、ID 参照、`reasonCode`、`deliveryKind`、`failureStage`、`errorCode` |

閉じたメッセージ列挙型は次のとおりです。

- `conversationKind`：`direct`、`group`、`channel`、または `unknown`。
- 受信 `outcome`：`completed`、`skipped`、または `failed`。オプションの
  `reasonCode`：`duplicate`、`reply_operation_active`、
  `reply_operation_aborted`、`fast_abort`、`plugin_bound_handled`、
  `plugin_bound_unavailable`、`plugin_bound_declined`、`plugin_bound_error`、
  `before_dispatch_handled`、`acp_dispatch_completed`、`acp_dispatch_failed`、
  `acp_dispatch_empty`、または `acp_dispatch_aborted`。
- 送信 `outcome`：`sent`、`suppressed`、`failed`、または `unknown`。オプションの
  `reasonCode`：`cancelled_by_message_sending_hook`、
  `cancelled_by_reply_payload_sending_hook`、
  `empty_after_message_sending_hook`、`empty_after_reply_payload_sending_hook`、
  または `no_visible_payload`。プラットフォーム ID を返さないアダプターは
  `unknown` です。外部への副作用がなかったことを証明できないためです。
- `deliveryKind`：`text`、`media`、または `other`。`failureStage`：
  `platform_send`、`queue`、または `unknown`。

終了フィールドは相互に関連しており、個別にオプションではありません。

| バリアント          | 終了時の対応関係                                                                                                                                                   |
| ---------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| エージェント実行        | `started` には `errorCode` がありません。成功以外の各完了ステータスには、対応する `run_*` コードが必要です。                                                                 |
| ツールアクション      | `started` と成功には `errorCode` がありません。その他の各完了ステータスには、対応する `tool_*` コードが必要です。                                                       |
| 受信メッセージ  | 成功 = `completed`、ブロック = `skipped`、失敗 = `failed` と `message_processing_failed`。`reasonCode` が存在する場合、その終了ファミリーに属している必要があります。 |
| 送信メッセージ | 成功 = `sent`、ブロック = `suppressed` と `reasonCode`、失敗 = `failed` と `errorCode` および `failureStage`、不明 = `unknown` と `failureStage`。      |

各アクティビティイベントには、安定したイベント ID、単調増加する台帳シーケンス、
ソースイベントシーケンス、タイムスタンプ、アクター、アクション、ステータス、整数の
`schemaVersion: 1`、および `redaction: "metadata_only"` が含まれます。実行レコードとツールレコードには
エージェントと実行の来歴が必要であり、セッションの来歴が含まれる場合もあります。メッセージ
レコードにはエージェント ID と実行 ID が含まれる場合がありますが、意図的に
`sessionKey` または `sessionId` は含まれません。そのため、`sessionKey` クエリフィルターは
実行行とツール行にのみ適用されます。ツールイベントには、ツール呼び出し ID とツール名が含まれる場合があります。

メッセージレコードは `message.inbound.processed` または
`message.outbound.finished` を使用し、方向、チャンネル、会話の種類、
正規化された結果に加えて、オプションの配信種類、失敗ステージ、所要時間、
結果件数、理由コード、およびインストール環境固有のキーで生成された
アカウント／会話／メッセージ／ターゲットの仮名を追加します。これらの仮名は
相関分析に役立ちますが、匿名化ではありません。状態データベースにはそのキーが含まれますが、
RPC および CLI のエクスポートには含まれません。台帳には、プロンプト、メッセージ本文、
ツール引数、ツール結果、コマンド出力、または生のエラーテキストは保存されません。
実行／ツールの `sessionKey` 値は生の相関メタデータのままであり、
プラットフォームのアカウント ID またはピア ID を含む場合があります。メッセージレコードではセッションキーを省略します。

受信行では、`durationMs` はコアディスパッチからその終了までを測定し、
`resultCount` は確定済みのキュー投入ツール、ブロック、および返信ペイロードを数えます。
送信行では、`durationMs` は配信所有権の取得から確認応答、
デッドレター、または照合まで（キューでの待機時間を含む）を対象とし、`resultCount` は
識別された物理的なプラットフォーム送信を数えます。`deliveryKind` が存在する場合は、
フックとレンダリング後の実効ペイロードを示します。抑制された行、または
クラッシュにより結果が曖昧な行では省略されます。

現在のメッセージ対象範囲には、コアディスパッチに到達した受理済みの受信メッセージが含まれ、
コアでの重複／終了結果も含まれます。送信側では、共有の永続的な
配信に到達した元の論理返信ペイロードごとに、終了行を 1 行書き込みます。チャンク化と
アダプターのファンアウトは `resultCount` に集約されます。キューに投入された
再試行可能または結果が曖昧な送信は、確認応答、デッドレター、
または照合の後にのみ記録されます。これらの共有境界を迂回する Plugin ローカルおよび
直接送信の経路は、まだ対象外です。上限付きワーカーキューはベストエフォートであり、
障害や飽和時にレコードが欠落する可能性があるため、この機能は
損失のないコンプライアンスアーカイブではありません。

記録はデフォルトで有効であり、
[`audit.enabled`](/ja-JP/gateway/configuration-reference#audit) で制御します。メッセージ記録は
`audit.messages` で個別に制御され、デフォルトは `"off"` です。
記録を無効にしても、`audit.activity.list` は以前に書き込まれたレコードを
有効期限が切れるまで引き続き提供します。

出荷済みの `audit.list` リクエスト、結果、および `AuditEvent` スキーマは
変更されておらず、エージェント実行およびツールアクションのレコードのみを返します。新しい運用クライアントは、
Gateway が `audit.activity.list` を公開している場合にそれを呼び出す必要があります。古い
Gateway は、`unknown method: audit.activity.list`、または出荷済みバージョンでは
メソッド検索より前に認可が行われていたため、読み取りスコープのリクエストに対して
`missing scope:
operator.admin` を報告する場合があります。後者をメソッド不在として扱うのは、
そのメソッドが公開されていなかった場合に限ります。その後、クライアントはフィルターが
メッセージ種類、方向、またはチャンネルのサポートを必要としない場合に限り、
`audit.list` を再試行できます。

テキストクエリと上限付き JSON エクスポートには、[`openclaw audit`](/ja-JP/cli/audit) を使用します。

## タスク台帳 RPC

運用クライアントは、タスク台帳 RPC（`packages/gateway-protocol/src/schema/tasks.ts`）を介して
Gateway のバックグラウンドタスクレコードを検査およびキャンセルします。これらは
生のランタイム状態ではなく、サニタイズされたタスク概要を返します。

- `tasks.list` には `operator.read` が必要です。
  - パラメーター: オプションの `status`（`"queued"`、`"running"`、`"completed"`、
    `"failed"`、`"cancelled"`、または `"timed_out"`）、もしくはこれらのステータスの配列、
    オプションの `agentId`、オプションの `sessionKey`、`1` から
    `500` までのオプションの `limit`、およびオプションの文字列 `cursor`。
  - 結果: `{ "tasks": TaskSummary[], "nextCursor"?: string }`。
- `tasks.get` には `operator.read` が必要です。
  - パラメーター: `{ "taskId": string }`。
  - 結果: `{ "task": TaskSummary }`。
  - 存在しないタスク ID に対しては、Gateway の not-found エラー形式が返されます。
- `tasks.cancel` には `operator.write` が必要です。
  - パラメーター: `{ "taskId": string, "reason"?: string }`。
  - 結果: `{ "found": boolean, "cancelled": boolean, "reason"?: string, "task"?: TaskSummary }`。
  - `found` は、台帳に一致するタスクがあったかどうかを示します。`cancelled` は、
    ランタイムがキャンセルを受理または記録したかどうかを示します。

`TaskSummary` には、`id`、`status`、およびオプションのメタデータとして `kind`、
`runtime`、`title`、`agentId`、`sessionKey`、`childSessionKey`、`ownerKey`、
`runId`、`taskId`、`flowId`、`parentTaskId`、`sourceId`、タイムスタンプ、進捗、
終了概要、およびサニタイズされたエラーテキストが含まれます。`agentId` はタスクを実行する
エージェントを識別し、`sessionKey` と `ownerKey` はリクエスターおよび制御の
コンテキストを保持します。

## 運用ヘルパーメソッド

- `commands.list`（`operator.read`）は、エージェントのランタイムコマンド一覧を
  取得します。
  - `agentId` はオプションです。デフォルトのエージェントワークスペースを読み取るには省略します。
  - `scope` は、プライマリ `name` が対象とするサーフェスを制御します。`text` は、
    先頭の `/` を除いたプライマリテキストコマンドトークンを返します。`native` と
    デフォルトの `both` 経路は、利用可能な場合にプロバイダー対応のネイティブ名を返します。
  - `textAliases` は、`/model` や `/m` などの正確なスラッシュエイリアスを保持します。
  - `nativeName` は、存在する場合にプロバイダー対応のネイティブコマンド名を保持します。
  - `provider` はオプションであり、ネイティブ名とネイティブ Plugin コマンドの
    可用性にのみ影響します。
  - `includeArgs=false` は、シリアル化された引数メタデータをレスポンスから省略します。
- `tools.catalog`（`operator.read`）は、エージェントのランタイムツールカタログを
  取得します。レスポンスには、グループ化されたツールと来歴メタデータが含まれます。
  - `source`: `core` または `plugin`
  - `pluginId`: `source="plugin"` の場合の Plugin 所有者
  - `optional`: Plugin ツールがオプションかどうか
- `tools.effective`（`operator.read`）は、セッションのランタイム実効ツール
  一覧を取得します。
  - `sessionKey` は必須です。
  - Gateway は、呼び出し元が指定した認証または配信コンテキストを受け入れるのではなく、
    サーバー側のセッションから信頼済みランタイムコンテキストを導出します。
  - レスポンスは、有効な一覧をセッションスコープでサーバーが導出した投影であり、
    コア、Plugin、チャンネル、および検出済みの MCP
    サーバーツールを含みます。
  - `tools.effective` は MCP に対して読み取り専用です。ウォームセッションの MCP
    カタログを最終ツールポリシーを介して投影できますが、MCP ランタイムの作成、
    トランスポートへの接続、または `tools/list` の発行は行いません。一致するウォームカタログが
    存在しない場合、レスポンスには `mcp-not-yet-connected`、
    `mcp-not-yet-listed`、または `mcp-stale-catalog` などの通知が含まれる場合があります。
  - 実効ツールエントリは、`source="core"`、`source="plugin"`、
    `source="channel"`、または `source="mcp"` を使用します。
- `tools.invoke`（`operator.write`）は、`/tools/invoke` と同じ
  Gateway ポリシー経路を介して、利用可能なツールを 1 つ呼び出します。
  - `name` は必須です。`args`、`sessionKey`、`agentId`、`confirm`、および
    `idempotencyKey` はオプションです。
  - `sessionKey` と `agentId` の両方が存在する場合、解決されたセッションエージェントは
    `agentId` と一致する必要があります。
  - `cron`、`gateway`、`nodes` などの所有者専用コアラッパーでは、
    `tools.invoke` 自体が `operator.write` であっても、
    所有者／管理者のアイデンティティ（`operator.admin`）が必要です。
  - レスポンスは、`ok`、`toolName`、オプションの
    `output`、および型付き `error` フィールドを持つ SDK 向けエンベロープです。承認またはポリシーによる拒否は、
    Gateway のツールポリシーパイプラインを迂回せず、ペイロード内で
    `ok:false` を返します。
- `skills.status`（`operator.read`）は、エージェントに表示される Skills 一覧を
  取得します。
  - `agentId` はオプションです。デフォルトのエージェントワークスペースを読み取るには省略します。
  - レスポンスには、生のシークレット値を公開することなく、適格性、不足している要件、設定チェック、
    およびサニタイズされたインストールオプションが含まれます。
- `skills.search` と `skills.detail`（`operator.read`）は、ClawHub
  の検出メタデータを返します。
- `skills.upload.begin`、`skills.upload.chunk`、および `skills.upload.commit`
  （`operator.admin`）は、プライベートな Skills アーカイブをインストール前にステージングします。これは、
  信頼済みクライアント用の独立した管理者アップロード経路であり、通常の ClawHub
  Skills インストールフローではありません。`skills.install.allowUploadedArchives` が有効でない限り、
  デフォルトでは無効です。
  - `skills.upload.begin({ kind: "skill-archive", slug, sizeBytes, sha256?, force?, idempotencyKey? })` は、
    そのスラッグと force 値にバインドされたアップロードを作成します。
  - `skills.upload.chunk({ uploadId, offset, dataBase64 })` は、
    正確なデコード済みオフセットにバイトを追加します。
  - `skills.upload.commit({ uploadId, sha256? })` は、最終サイズと
    SHA-256 を検証します。コミットはアップロードを確定するだけで、Skills をインストールしません。
  - アップロードされる Skills アーカイブは、`SKILL.md` ルートを含む zip アーカイブです。
    アーカイブ内部のディレクトリ名がインストール先を選択することはありません。
- `skills.install`（`operator.admin`）には 3 つのモードがあります。
  - ClawHub モード: `{ source: "clawhub", slug, version?, force? }` は、
    Skills フォルダーをデフォルトのエージェントワークスペースの `skills/` ディレクトリにインストールします。
  - アップロードモード: `{ source: "upload", uploadId, slug, force?, sha256?, timeoutMs? }` は、
    コミット済みアップロードをデフォルトのエージェントワークスペースの
    `skills/<slug>` ディレクトリにインストールします。スラッグと force 値は、
    元の `skills.upload.begin` リクエストと一致する必要があります。
    `skills.install.allowUploadedArchives` が有効でない限り拒否されます。この設定は
    ClawHub のインストールには影響しません。
  - Gateway インストーラーモード: `{ name, installId, timeoutMs? }` は、Gateway ホスト上で宣言済みの
    `metadata.openclaw.install` アクションを実行します。古いクライアントは
    引き続き `dangerouslyForceUnsafeInstall` を送信する場合があります。このフィールドは非推奨であり、
    プロトコル互換性のためにのみ受理され、無視されます。運用者が所有するインストール判断には
    `security.installPolicy` を使用します。
- `skills.update`（`operator.admin`）には 2 つのモードがあります。
  - ClawHub モードは、デフォルトのエージェントワークスペースにある追跡対象スラッグ 1 件、
    または追跡対象のすべての ClawHub インストールを更新します。
  - 設定モードは、`enabled`、
    `apiKey`、`env` などの `skills.entries.<skillKey>` 値にパッチを適用します。

### `models.list` ビュー

`models.list` は、オプションの `view` パラメーター
（`src/agents/model-catalog-visibility.ts`）を受け入れます。

- 省略または `"default"`: `agents.defaults.modelPolicy.allow` が設定されている場合、
  レスポンスは許可されたカタログとなり、`provider/*` エントリの
  動的に検出されたモデルも含まれます。それ以外の場合、レスポンスは Gateway の
  完全なカタログです。
- `"configured"`: ピッカー向けの動作です。`agents.defaults.modelPolicy.allow` が
  設定されている場合は、`provider/*` エントリのプロバイダースコープ検出を含め、
  引き続きそれが優先されます。許可リストがない場合、レスポンスは明示的な
  `models.providers.<provider>.models` エントリを使用し、設定済みのモデル行が存在しない場合に限り
  完全なカタログへフォールバックします。
- `"provider-config"`: ソースで定義された `models.providers.*.models` 一覧であり、
  ピッカーの許可リストから独立しています。行には公開モデル機能と
  ルートを考慮した可用性が含まれますが、プロバイダーのエンドポイント、認証情報、および
  ランタイムリクエスト設定は省略されます。
- `"all"`: `agents.defaults.modelPolicy.allow` を迂回する Gateway の完全なカタログです。
  通常のモデルピッカーではなく、診断／検出 UI に使用します。

## 実行の承認

- exec リクエストに承認が必要な場合、Gateway は
  `exec.approval.requested` をブロードキャストします。
- オペレータークライアントは `exec.approval.resolve` を呼び出して解決します（
  `operator.approvals` が必要です）。
- `host=node` では、`exec.approval.request` に `systemRunPlan`
  （正規の `argv`/`cwd`/`rawCommand`/セッションメタデータ）を含める必要があります。
  `systemRunPlan` がないリクエストは拒否されます。
- 承認後、転送される `node.invoke system.run` 呼び出しは、その正規の
  `systemRunPlan` を信頼できるコマンド/cwd/セッションコンテキストとして再利用します。
- 呼び出し元が準備から最終的に承認された `system.run` の転送までの間に、
  `command`、`rawCommand`、`cwd`、`agentId`、または
  `sessionKey` を変更した場合、Gateway は変更されたペイロードを信頼せず、実行を拒否します。

## エージェント配信のフォールバック

- `agent` リクエストには、外部への配信を要求するために `deliver=true` を含めることができます。
- `bestEffortDeliver=false`（デフォルト）は厳密な動作を維持します。解決できない、または
  内部専用の配信先に対しては `INVALID_REQUEST` を返します。
- `bestEffortDeliver=true` を指定すると、外部配信可能なルートを解決できない場合（たとえば、内部/webchat
  セッションや複数チャネルの設定が曖昧な場合）に、セッション内のみの実行へフォールバックできます。
- 配信が要求された場合、最終的な `agent` の結果には `result.deliveryStatus` が含まれることがあります。
  これには、[`openclaw agent --json --deliver`](/ja-JP/cli/agent#json-delivery-status)で説明されているものと同じ
  `sent`、`suppressed`、`partial_failed`、および
  `failed` のステータスが使用されます。

## バージョニング

- `PROTOCOL_VERSION`、`MIN_CLIENT_PROTOCOL_VERSION`、
  `MIN_NODE_PROTOCOL_VERSION`、および `MIN_PROBE_PROTOCOL_VERSION` は
  `packages/gateway-protocol/src/version.ts` にあります。
- クライアントは `minProtocol` + `maxProtocol` を送信します。オペレータークライアントと UI クライアントは、
  その範囲に現在のプロトコルを含める必要があります。現在のクライアントとサーバーは
  プロトコル v4 で動作します。
- `role: "node"` と `client.mode: "node"` の両方を持つ認証済みクライアントは、
  N-1 Node プロトコル（現在は v3）を使用できます。軽量な再起動プローブも
  同じ N-1 ウィンドウを使用します。この互換性ウィンドウによって、デバイス認証、ペアリング、スコープ、コマンドポリシー、および exec
  の承認は変更されません。Plugin が所有する Node の
  ケイパビリティとコマンドは、それらをホストするサーフェスが N-1 契約に含まれないため、Node が現在の
  プロトコルへアップグレードするまで提供されません。
- スキーマとモデルは TypeBox 定義から生成されます。
  - `pnpm protocol:gen`
  - `pnpm protocol:gen:swift`
  - `pnpm protocol:check`

### クライアント定数

リファレンスクライアントの実装は `packages/gateway-client/src/` にあります
（OpenClaw は薄い `src/gateway/client.ts` ファサードを介してこれをラップします）。これらの
デフォルト値はプロトコル v4 全体で安定しており、サードパーティークライアントに期待される基準値です。

| 定数                                      | デフォルト                                            | ソース                                                                                                                    |
| ----------------------------------------- | ----------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------- |
| `PROTOCOL_VERSION`                        | `4`                                                   | `packages/gateway-protocol/src/version.ts`                                                                                |
| `MIN_CLIENT_PROTOCOL_VERSION`             | `4`                                                   | `packages/gateway-protocol/src/version.ts`                                                                                |
| `MIN_NODE_PROTOCOL_VERSION`               | `3`                                                   | `packages/gateway-protocol/src/version.ts`                                                                                |
| `MIN_PROBE_PROTOCOL_VERSION`              | `3`                                                   | `packages/gateway-protocol/src/version.ts`                                                                                |
| リクエストタイムアウト（RPC ごと）        | `30_000` ms                                           | `packages/gateway-client/src/client.ts`（`requestTimeoutMs`）                                                              |
| 事前認証/接続チャレンジのタイムアウト     | `15_000` ms                                           | `packages/gateway-client/src/timeouts.ts`（`OPENCLAW_HANDSHAKE_TIMEOUT_MS` env により、ペアとなるサーバー/クライアントの時間枠を延長可能） |
| 初回再接続バックオフ                      | `1_000` ms                                            | `packages/gateway-client/src/client.ts`（`GATEWAY_RECONNECT_POLICY`）                                                      |
| 最大再接続バックオフ                      | `30_000` ms                                           | `packages/gateway-client/src/client.ts`（`GATEWAY_RECONNECT_POLICY`）                                                      |
| デバイストークンによる切断後の高速再試行上限 | `250` ms                                              | `packages/gateway-client/src/client.ts`                                                                                   |
| `terminate()` 前の強制停止猶予期間   | `250` ms                                              | `FORCE_STOP_TERMINATE_GRACE_MS`                                                                                           |
| `stopAndWait()` のデフォルトタイムアウト | `1_000` ms                                            | `STOP_AND_WAIT_TIMEOUT_MS`                                                                                                |
| デフォルトの tick 間隔（`hello-ok` 前） | `30_000` ms                                           | `packages/gateway-client/src/client.ts`                                                                                   |
| tick タイムアウトによる切断               | 無通信状態が `tickIntervalMs * 2` を超えた場合はコード `4000` | `packages/gateway-client/src/client.ts`                                                                                   |
| `MAX_PAYLOAD_BYTES`                       | `25 * 1024 * 1024`（25 MB）                            | `src/gateway/server-constants.ts`                                                                                         |

サーバーは、有効な `policy.tickIntervalMs`、
`policy.maxPayload`、および `policy.maxBufferedBytes` を `hello-ok` で通知します。クライアントは、
ハンドシェイク前のデフォルト値ではなく、これらの値に従う必要があります。

保留中のすべてのリクエストに期限が設定されている場合、リファレンスクライアントでは、有限のリクエストが
設定された期限を管理します。有限の
`timeoutMs` がない `expectFinal` リクエスト、`timeoutMs: null` を持つリクエスト、または有限リクエストと
無期限リクエストの混在がある場合、tick ウォッチドッグは有効なままになります。受信イベントと
レスポンスが tick タイムアウトのしきい値を超えてもない場合、クライアントは
コード `4000` でソケットを閉じ、保留中のすべてのリクエストを拒否して再接続します。
再接続後に拒否されたリクエストを再実行することはありません。

## 認証

- 共有シークレットによる Gateway 認証では、設定された
  `gateway.auth.mode`（`"none" | "token" | "password" | "trusted-proxy"`）に応じて、`connect.params.auth.token` または
  `connect.params.auth.password` を使用します。
- Tailscale Serve（`gateway.auth.allowTailscale: true`）や非ループバックの `gateway.auth.mode: "trusted-proxy"`
  など、アイデンティティを伴うモードでは、`connect.params.auth.*` の代わりにリクエストヘッダーから
  接続認証チェックを満たします。
- プライベートイングレスの `gateway.auth.mode: "none"` では、共有シークレットによる接続認証を
  完全にスキップします。このモードを公開または信頼されていないイングレスに公開しないでください。
- ペアリング後、Gateway は接続のロールとスコープに限定されたデバイストークンを発行し、
  `hello-ok.auth.deviceToken` で返します。クライアントは、接続に成功するたびに
  それを永続化する必要があります。
- 保存したデバイストークンで再接続する場合は、そのトークンについて保存済みの
  承認済みスコープセットも再利用する必要があります。これにより、すでに付与された読み取り、プローブ、ステータスへのアクセスが
  維持され、再接続時に暗黙の管理者専用スコープへ気付かないまま狭められることを
  防止できます。
- クライアント側の接続認証の組み立て（`packages/gateway-client/src/client.ts` の
  `selectConnectAuth`）：
  - `auth.password` は独立しており、設定されている場合は常に転送されます。
  - `auth.token` は、最初に明示的な共有トークン、
    次に明示的な `deviceToken`、その後に保存済みのデバイスごとのトークン
    （`deviceId` + `role` をキーとする）の優先順で設定されます。
  - `auth.bootstrapToken` は、上記のいずれでも
    `auth.token` が解決されなかった場合にのみ送信されます。共有トークンまたは解決済みのデバイストークンがある場合は送信されません。
  - 1 回限りの `AUTH_TOKEN_MISMATCH` 再試行時に保存済みデバイストークンを
    自動昇格できるのは、信頼済みエンドポイントのみです。具体的には、ループバック、
    または `tlsFingerprint` がピン留めされた `wss://` です。ピン留めされていない公開 `wss://` は
    対象になりません。
- 組み込みのセットアップコードによるブートストラップは、信頼済みモバイルへの引き渡し用に、
  プライマリ Node の `hello-ok.auth.deviceToken` と、制限されたオペレータートークンを
  `hello-ok.auth.deviceTokens` で返します。このオペレータートークンには、
  ネイティブ Talk 設定の読み取り用の `operator.talk.secrets` が含まれますが、
  ペアリング変更スコープと `operator.admin` は含まれません。
- 非ベースラインのセットアップコードによるブートストラップが承認を待機している間、
  `PAIRING_REQUIRED` の詳細には `recommendedNextStep: "wait_then_retry"`、
  `retryable: true`、および `pauseReconnect: false` が含まれます。リクエストが承認されるか、トークンが
  無効になるまで、同じブートストラップトークンで再接続を続けてください。
- `hello-ok.auth.deviceTokens` を永続化するのは、接続が `wss://` やループバック／ローカルペアリングなどの
  信頼済みトランスポート上でブートストラップ認証を使用した場合に限ります。
- クライアントが明示的な `deviceToken` または明示的な `scopes` を指定した場合、
  呼び出し元が要求したそのスコープセットが引き続き優先されます。キャッシュ済みスコープが
  再利用されるのは、クライアントが保存済みのデバイスごとのトークンを再利用する場合のみです。
- デバイストークンは、`device.token.rotate` および
  `device.token.revoke`（`operator.pairing` が必要）を使用してローテーションまたは失効できます。
  Node またはその他の非オペレーターロールをローテーションまたは失効する場合は、`operator.admin` も必要です。
- `device.token.rotate` はローテーションのメタデータを返します。置換後の
  Bearer トークンを返すのは、そのデバイストークンですでに認証されている
  同一デバイスからの呼び出しの場合のみです。これにより、トークンのみを使用するクライアントは、再接続前に置換トークンを永続化できます。
  共有／管理者によるローテーションでは、Bearer トークンは返されません。
- トークンの発行、ローテーション、失効は、そのデバイスのペアリングエントリに記録された
  承認済みロールセットの範囲内に制限されます。トークンの変更によって、ペアリング承認で一度も付与されていない
  デバイスロールへ拡張したり、そのロールを対象にしたりすることはできません。
- ペアリング済みデバイスのトークンセッションでは、呼び出し元が
  `operator.admin` も持っていない限り、デバイス管理は自身に限定されます。管理者以外の呼び出し元が管理できるのは、
  自身のデバイスエントリのオペレータートークンのみです。Node およびその他の非オペレータートークンの
  管理は、呼び出し元自身のデバイスであっても管理者専用です。
- `device.token.rotate` と `device.token.revoke` は、対象の
  オペレータートークンのスコープセットも、呼び出し元の現在のセッションスコープと照合します。
  管理者以外の呼び出し元は、自身がすでに保持しているものより広いオペレータートークンを
  ローテーションまたは失効できません。
- 認証エラーには、`error.details.code` と復旧のヒントが含まれます：
  - `error.details.canRetryWithDeviceToken`（真偽値）
  - `error.details.recommendedNextStep`：`retry_with_device_token`、
    `update_auth_configuration`、`update_auth_credentials`、
    `wait_then_retry`、`review_auth_configuration`
    （`packages/gateway-protocol/src/connect-error-details.ts`）のいずれか。
- `AUTH_TOKEN_MISMATCH` に対するクライアントの動作：
  - 信頼済みクライアントは、キャッシュ済みのデバイスごとのトークンを使用して、
    制限された再試行を 1 回だけ試みることができます。
  - その再試行が失敗した場合は、自動再接続ループを停止し、オペレーターが行うべき
    対応のガイダンスを表示します。
- `AUTH_SCOPE_MISMATCH` は、デバイストークンが認識されたものの、要求された
  ロール／スコープを満たしていないことを意味します。これを不正なトークンとして表示せず、
  再ペアリングするか、より狭い／広いスコープ契約を承認するようオペレーターに促してください。

## デバイスアイデンティティとペアリング

- Node は、キーペアのフィンガープリントから導出された安定したデバイスアイデンティティ
  （`device.id`）を含める必要があります。
- Gateway はデバイスとロールの組み合わせごとにトークンを発行します。
- ローカルでの自動承認が有効でない限り、新しいデバイス ID には
  ペアリングの承認が必要です。
- ペアリングの自動承認は、直接の local loopback 接続を中心に適用されます。
- OpenClaw には、信頼済みの共有シークレットヘルパーフロー向けに、
  バックエンド／コンテナローカルで自己接続する限定的な経路もあります。
- 同一ホストの tailnet または LAN 接続もペアリング上はリモートとして扱われ、
  承認が必要です。
- WS クライアントは通常、`connect` の際に `device` アイデンティティを含めます
  （オペレーター + Node）。デバイスなしのオペレーターに対する唯一の例外は、次の明示的な信頼経路です：
  - `gateway.auth.mode: "trusted-proxy"` によるオペレーター向け Control UI 認証の成功。
  - 予約済みの内部ヘルパー経路での、直接ループバック `gateway-client`
    バックエンド RPC。
- デバイスアイデンティティを省略すると、スコープに影響します。デバイスなしの
  オペレーター接続が明示的な信頼経路によって許可されている場合でも、その経路に
  名前付きのスコープ保持例外がない限り、OpenClaw は自己申告されたスコープを空のセットに
  クリアします。その後、スコープで制限されたメソッドは
  `missing scope` で失敗します。
- 予約済みの直接ループバック `gateway-client` バックエンドヘルパー経路が
  スコープを保持するのは、内部のローカルコントロールプレーン RPC に限られます。カスタムバックエンド ID には
  この例外は適用されません。
- すべての接続は、サーバーから提供された `connect.challenge` nonce に署名する必要があります。

### デバイス認証移行の診断

チャレンジ以前の署名動作を引き続き使用しているレガシークライアントに対して、`connect` は
安定した `error.details.reason` とともに、`error.details.code` の下で `DEVICE_AUTH_*` 詳細コードを
返します。

一般的な移行エラー：

| メッセージ                     | details.code                     | details.reason           | 意味                                            |
| --------------------------- | -------------------------------- | ------------------------ | -------------------------------------------------- |
| `device nonce required`     | `DEVICE_AUTH_NONCE_REQUIRED`     | `device-nonce-missing`   | クライアントが `device.nonce` を省略した（または空で送信した）。     |
| `device nonce mismatch`     | `DEVICE_AUTH_NONCE_MISMATCH`     | `device-nonce-mismatch`  | クライアントが古い、または誤った nonce で署名した。            |
| `device signature invalid`  | `DEVICE_AUTH_SIGNATURE_INVALID`  | `device-signature`       | 署名ペイロードが v2 ペイロードと一致しない。       |
| `device signature expired`  | `DEVICE_AUTH_SIGNATURE_EXPIRED`  | `device-signature-stale` | 署名されたタイムスタンプが許容される時刻ずれの範囲外にある。          |
| `device identity mismatch`  | `DEVICE_AUTH_DEVICE_ID_MISMATCH` | `device-id-mismatch`     | `device.id` が公開鍵のフィンガープリントと一致しない。 |
| `device public key invalid` | `DEVICE_AUTH_PUBLIC_KEY_INVALID` | `device-public-key`      | 公開鍵の形式／正規化に失敗した。         |

移行先：

- 常に `connect.challenge` を待機します。
- サーバーの nonce を含む v2 ペイロードに署名します。
- `connect.params.device.nonce` で同じ nonce を送信します。
- 推奨される署名ペイロードは `v3`
  （`packages/gateway-client/src/device-auth.ts` の `buildDeviceAuthPayloadV3`）です。
  これは、デバイス／クライアント／ロール／スコープ／トークン／nonce フィールドに加えて、
  `platform` と `deviceFamily` をバインドします。
- 互換性のため、レガシーの `v2` 署名も引き続き受け入れられますが、
  再接続時のコマンドポリシーは、引き続きペアリング済みデバイスのメタデータピン留めによって制御されます。

## TLS とピン留め

- WS 接続では TLS がサポートされています（`gateway.tls` の設定）。
- クライアントは、`gateway.remote.tlsFingerprint` または CLI の `--tls-fingerprint` を使用して、
  Gateway 証明書のフィンガープリントを任意でピン留めできます。

## スコープ

このプロトコルは、ステータス、チャンネル、モデル、チャット、
エージェント、セッション、Node、承認など、Gateway API の全機能を公開します。正確な対象範囲は、
`packages/gateway-protocol/src/schema.ts` から再エクスポートされる TypeBox スキーマによって定義されます。

## 関連項目

- [Gateway クライアントの構築](https://docs.openclaw.ai/gateway/clients)
- [OpenClaw の埋め込み](https://docs.openclaw.ai/gateway/embedding)
- [ブリッジプロトコル](/ja-JP/gateway/bridge-protocol)
- [Gateway ランブック](/ja-JP/gateway)
