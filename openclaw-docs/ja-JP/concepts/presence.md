---
read_when:
    - Control UI のデバイスページでライブステータスをデバッグする
    - 重複または古いインスタンス行の調査
    - Gateway の WebSocket 接続またはシステムイベントビーコンの変更
summary: OpenClaw のプレゼンスエントリが生成、マージ、表示される仕組み
title: プレゼンス
x-i18n:
    generated_at: "2026-07-26T09:18:55Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: ac5800eebddb82e69a7d0c06733e6a19addbc57be7776e7361411866af0c60f5
    source_path: concepts/presence.md
    workflow: 16
---

OpenClaw の「プレゼンス」は、以下を軽量かつベストエフォートで把握するためのものです。

- **Gateway** 自体、および
- **Gateway に接続されているユーザー表示対象のクライアント**（Mac アプリ、WebChat、Node など）

プレゼンスは、Control UI の **Devices** ページ
（**Settings → Devices** 内）と、macOS アプリの **Instances** タブにライブ接続メタデータを表示します。

このページでは、Gateway のクライアント一覧について説明します。最後に使用した Mac を検出し、
Node のアラートをその Mac にルーティングする方法については、
[アクティブなコンピューターのプレゼンス](/ja-JP/nodes/presence)を参照してください。

## プレゼンスフィールド（表示される内容）

プレゼンスエントリは、次のようなフィールドを持つ構造化オブジェクトです。

- `instanceId`（任意ですが、強く推奨）：安定したクライアント ID（通常は `connect.client.instanceId`）
- `host`：人が識別しやすいホスト名
- `ip`：ベストエフォートで取得した IP アドレス
- `version`：クライアントのバージョン文字列
- `deviceFamily` / `modelIdentifier`：ハードウェアに関するヒント
- `mode`：`ui`、`webchat`、`cli`、`backend`、`node`、`probe`、`test`
- `lastInputSeconds`：判明している場合、最後のユーザー入力からの経過秒数
- `reason`：クライアントが指定する自由形式の文字列。Gateway 自体が出力するのは `self`、`connect`、`disconnect` のみ
- `deviceId`、`roles`、`scopes`：接続ハンドシェイクから取得したデバイス ID、およびロール／スコープに関するヒント
- `ts`：最終更新タイムスタンプ（エポックからの経過ミリ秒）

## 生成元（プレゼンスの取得元）

プレゼンスエントリは複数のソースによって生成され、**マージ**されます。

### 1) Gateway 自身のエントリ

クライアントが接続する前でも UI に Gateway ホストが表示されるように、
Gateway は起動時に必ず「自身」のエントリを初期登録します。

### 2) WebSocket 接続

すべての WS クライアントは `connect` リクエストから開始します。ハンドシェイクが成功すると、
Gateway はその接続のプレゼンスエントリを upsert します。

#### 一時的なコントロールプレーン接続が表示されない理由

CLI コマンド、バックエンド RPC クライアント、プローブは、多くの場合短時間だけ接続します。
その変動をプレゼンスの TTL 全期間にわたって保持しないよう、`cli`、`backend`、
または `probe` モードのクライアントはプレゼンスエントリに**変換されません**。テストモードのクライアントは、
テストスイートで実際のクライアントの代わりとして使用されるため、追跡が継続されます。

### 3) `system-event` ビーコン

クライアントは、`system-event` メソッドを介して、より詳細な情報を含む定期ビーコンを送信できます。Mac
アプリはこれを使用して、ホスト名、IP、バージョン、稼働状況のメタデータを報告します。物理的な
入力アクティビティはこの汎用ビーコンには含まれません。[アクティブなコンピューターのプレゼンス](/ja-JP/nodes/presence)で説明している
専用のネイティブ Node イベントがそれを処理します。Mac はこれらのビーコンに
`system-presence-clear-last-input` タグを付けます。現在の Gateway は、その後方互換性のあるマーカーを使用して、
古いアプリから保持されている可能性のある入力の直近性情報を削除します。このビーコンには固定値の 30 日も含まれるため、
タグを無視する古い Gateway でも、正確な直近性情報を保持する代わりに上書きできます。この互換性維持用の値のために、
新しいアクティビティがサンプリングされることはありません。

### 4) Node の接続（ロール：Node）

Node が `role: node` を使用して Gateway WebSocket 経由で接続すると、Gateway は
その Node のプレゼンスエントリを upsert します（他の WS クライアントと同じフローです）。

## マージと重複排除のルール（`instanceId` が重要な理由）

プレゼンスエントリは、単一のインメモリマップに保存されます。キーには、大文字と小文字を区別せず、
次の順序で最初に利用可能なものが使用されます：ペアリング済みデバイス ID、`connect.client.instanceId`、
最後の手段として接続ごとの ID。

一時的なコントロールプレーンクライアントは追跡対象から完全に除外されるため（前述）、
その接続 ID がキーになることはありません。その他のすべてのクライアントでは、接続 ID へのフォールバックにより、
安定した `instanceId` を持たずに再接続したクライアントは**重複した**行として表示されます。

## TTL とサイズ上限

プレゼンスは意図的に一時的なものとして扱われます。

- **TTL：** 5 分より古いエントリは削除
- **最大エントリ数：** 200（古いものから先に削除）

これにより一覧を最新に保ち、メモリ使用量が際限なく増えるのを防ぎます。

## リモート／トンネルに関する注意事項（ループバック IP）

クライアントが SSH トンネル／ローカルポートフォワーディング経由で接続すると、Gateway は
リモートアドレスを `127.0.0.1` として認識する場合があります。そのトンネルアドレスが
クライアントの IP として記録されるのを避けるため、接続処理ではローカル（ループバック）として検出されたクライアントについて、
ループバックアドレスをエントリに書き込むのではなく、`ip` を完全に省略します。

## 利用側

### Control UI の Devices ページ

**Devices** ページは、`system-presence` を永続的なペアリングレコードおよび Node
レコードと結合します。Gateway 自身のビーコンを先頭に固定し、一致するデバイス ID または
インスタンス ID を使用して、プラットフォーム、バージョン、モデル、入力の直近性に関するライブメタデータを表示します。

### macOS の Instances タブ

macOS アプリは `system-presence` の出力をレンダリングし、最終更新からの経過時間に基づいて、
小さなステータスインジケーター（Active/Idle/Stale）を適用します。

## デバッグのヒント

- 未加工の一覧を確認するには、Gateway に対して `system-presence` を呼び出します。
- 重複が表示される場合：
  - クライアントがハンドシェイクで安定した `client.instanceId` を送信していることを確認する
  - 定期ビーコンで同じ `instanceId` が使用されていることを確認する
  - 接続から生成されたエントリに `instanceId` がないか確認する（その場合、重複は想定どおりです）

## 関連項目

<CardGroup cols={2}>
  <Card title="アクティブなコンピューターのプレゼンス" href="/ja-JP/nodes/presence" icon="computer-mouse">
    Mac での物理入力によってアクティブな Node を選択し、接続アラートをルーティングする仕組み。
  </Card>
  <Card title="入力中インジケーター" href="/ja-JP/concepts/typing-indicators" icon="ellipsis">
    入力中インジケーターが送信されるタイミングと、その調整方法。
  </Card>
  <Card title="ストリーミングとチャンク分割" href="/ja-JP/concepts/streaming" icon="bars-staggered">
    送信ストリーミング、チャンク分割、チャンネルごとの書式設定。
  </Card>
  <Card title="Gateway のアーキテクチャ" href="/ja-JP/concepts/architecture" icon="diagram-project">
    Gateway のコンポーネントと、プレゼンス更新を駆動する WebSocket プロトコル。
  </Card>
  <Card title="Gateway プロトコル" href="/ja-JP/gateway/protocol" icon="plug">
    `connect`、`system-event`、`system-presence` のワイヤープロトコル。
  </Card>
</CardGroup>
