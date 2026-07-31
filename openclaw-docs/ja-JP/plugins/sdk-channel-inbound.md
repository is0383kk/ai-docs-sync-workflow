---
read_when:
    - メッセージングチャネル Plugin の受信パスを構築またはリファクタリングしている場合
    - 共有の受信コンテキスト構築、セッション記録、または準備済み返信のディスパッチが必要です
    - 古いチャンネルターンヘルパーをインバウンド/メッセージ API に移行しています
summary: チャンネル Plugin 向け受信イベントヘルパー：コンテキスト構築、共有ランナーのオーケストレーション、セッション記録、準備済み返信のディスパッチ
title: チャンネル受信 API
x-i18n:
    generated_at: "2026-07-26T09:14:23Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 854408ca42cfe1e1b48e4fd223b176f438e1db28deb9a5aa33eea8238127d9df
    source_path: plugins/sdk-channel-inbound.md
    workflow: 16
---

チャネルの受信パスは、1つのフローに従います。

```text
プラットフォームイベント -> 受信ファクト/コンテキスト -> エージェントの応答 -> メッセージ配信
```

受信イベントの正規化、フォーマット、ルート、オーケストレーションには `openclaw/plugin-sdk/channel-inbound` を使用します。
ネイティブ送信、受領情報、永続的な配信、ライブプレビューの動作には
`openclaw/plugin-sdk/channel-outbound` を使用します。

## コアヘルパー

```ts
import {
  buildChannelInboundEventContext,
  runChannelInboundEvent,
  dispatchChannelInboundReply,
} from "openclaw/plugin-sdk/channel-inbound";
```

- `buildChannelInboundEventContext(...)`: 正規化されたチャネルファクトを
  プロンプト/セッションコンテキストに投影します。チャネルが所有する送信者/チャットのメタデータは
  `channelContext` を通じて渡します。Plugin フックからは `ctx.channelContext` として参照されます。
  チャネル固有フィールドについては、このサブパスから `PluginHookChannelSenderContext` または
  `PluginHookChannelChatContext` を拡張します。
- `runChannelInboundEvent(...)`: 1つの受信プラットフォームイベントについて、取り込み、分類、事前チェック、解決、
  記録、ディスパッチ、終了処理を実行します。
- `dispatchChannelInboundReply(...)`: すでに組み立てられた受信応答を、
  配信アダプターを使用して記録し、ディスパッチします。

メディアのみの受信イベントでは、メッセージ本文とコマンドテキストを空にし、
ネイティブ添付ファイルごとに1つの `ChannelInboundMediaInput` ファクトを渡します。周辺の
履歴行や別のテキストのみのキャリアでそれらのファクトを記述する必要がある場合は、
`formatMediaPlaceholderText(media)` を使用します。各ファクトは `kind`、MIME
タイプ、パスまたは URL の拡張子の順に分類されます。未ダウンロードのネイティブ添付ファイルも、
それぞれタイプのみのファクトを1つ提供する必要があります。フォーマッターを使用して
主要な受信本文を合成しないでください。

Plugin が所有する添付ファイルレコードを `toInboundMediaFacts(...)` で正規化し、
結果の順序付き配列をコンテキストの `media` フィールドを通じて渡します。

```ts
const media = toInboundMediaFacts([
  { path: saved.path, url: nativeUrl, contentType: saved.contentType, messageId },
]);

const ctx = finalizeInboundContext({ Body: caption, media });
```

配列内の位置が添付ファイルの識別子です。ファクトごとの `transcribed`、`messageId`、
`workspaceDir` が、従来の並列インデックス/ワークスペースフィールドを置き換えます。
`MediaPath`、`MediaPaths`、`MediaUrl`、`MediaUrls`、`MediaType`、`MediaTypes`、
`MediaTranscribedIndexes`、`MediaWorkspaceDir`、`MediaStaged` のコンテキストフィールドと
`buildChannelInboundMediaPayload(...)` は、非推奨の互換性機能としてのみ引き続き利用できます。
新しい Plugin では、これらを構築したり読み取ったりしないでください。

注入された Plugin ランタイムオブジェクトをすでに受け取っている同梱/ネイティブチャネルは、
このサブパスを直接インポートする代わりに、`runtime.channel.inbound.*` 配下の
同じヘルパーを呼び出せます。

```ts
await runtime.channel.inbound.run({
  channel: "demo",
  accountId,
  raw: platformEvent,
  adapter: {
    ingest: normalizePlatformEvent,
    resolveTurn: resolveInboundReply,
  },
});
```

プラットフォーム配信を配信アダプター内に保持する互換性ディスパッチャー向けに、
`dispatchChannelInboundReply(...)` の入力を組み立てます。新しい送信パスでは、代わりに
`channel-outbound` のメッセージアダプターと永続メッセージヘルパーを使用してください。

## 配信確定コントラクト

`ChannelInboundTurnPlan.delivery` は、論理応答ペイロードごとのネイティブ送信を所有します。
コアは送信フックの順序と、アダプターがオプトインした場合の
終端 `message_sent` 監視を所有します。1つのペイロードから終端イベントが重複して
生成されないように、これらの責務を分離してください。

配信結果フィールドには次の意味があります。

| フィールド                    | コントラクト                                                                                                                                                                                                                     |
| ------------------------ | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `content`                | ネイティブのフォーマットまたは終了処理後にプロバイダーが受理した、論理ペイロードの表示テキスト。省略した場合、終端監視には準備済みのペイロードテキストが使用されます。メディアのみの送信では省略できます。                             |
| `messageIds` / `receipt` | 表示送信に対する実際のプロバイダー識別情報。`MessageReceipt` を優先してください。コアは、そのプライマリプロバイダー ID を `message_sent` に使用します。                                                                                            |
| `visibleReplySent`       | プロバイダーが表示可能なプレビューも最終メッセージも生成しなかった場合にのみ、`false` に設定します。コアは、その結果に対して成功した `message_sent` を発行しません。                                                                          |
| `finalization`           | インプレースのストリーミングカードを閉じたり編集したりする場合など、同じ論理ペイロードの遅延ネイティブ確定を表す Promise。解決されたフィールドは、終端監視と `onDelivered` の前に即時結果を上書きします。 |

コアがこのアダプターの非永続送信について、標準の Plugin および内部
`message_sent` イベントを発行する必要がある場合は、配信アダプターの
`observeMessageSent` オプションを `true` に設定します。このオプションを
`deliver` から返さず、Plugin 側でもそれらのイベントを発行しないでください。
永続送信は共有送信オーナーを通じてすでに発行されるため、重複しません。

論理ペイロードごとに1つの結果を返します。`finalization` は2回目の送信ではないため、
`reply_payload_sending` または `message_sending` を再実行してはなりません。
`deliver` が戻るとすぐに、コアは終了処理 Promise の reject を監視するため、
未処理になることはありません。コアは引き続き、応答ディスパッチの確定後に元の Promise を
await します。その後、確定済みのコンテンツとプロバイダー ID を使用して、
ペイロードごとに最大1回の終端監視を発行します。`onDelivered` が存在する場合は、
その監視後に確定済みの結果を受け取ります。

ネイティブ配信が失敗した場合は、`deliver` または `finalization` を reject します。
プロバイダーへの送信が試行されなかった場合は、`openclaw/plugin-sdk/error-runtime` から
`PlatformMessageNotDispatchedError` を throw します。コアは誤った `message_sent`
イベントを抑制します。後続の操作が失敗する前にネイティブ送信が表示された場合は、
表示された部分をエラーに保持します。

```ts
import { createChannelPartialDeliveryError } from "openclaw/plugin-sdk/channel-inbound";

throw createChannelPartialDeliveryError(cause, {
  visibleReplySent: true,
  content: finalizedVisibleText,
  receipt,
});
```

コアは、そのプロバイダーに表示されたコンテンツと識別情報を使用して、失敗した終端監視を発行し、
配信自体は失敗した状態に保ちます。これにより、呼び出し元が部分的な成功を正常な送信と
誤認することを防ぎます。プレビュー、下書き、添付ファイル、最終メッセージのいずれかが
表示された後は、`visibleReplySent: false` を報告しないでください。

`reply_payload_sending` または `message_sending` が登録されている場合、
どちらのフックも論理ペイロードを書き換えたりキャンセルしたりできるため、
プロバイダーに表示されるものを作成する前に、それらのフックを確定させる必要があります。
ネイティブプレビューを先行して作成すると、書き換え前のコンテンツが漏洩したり、
キャンセル済みの下書きが残ったりします。受理されたペイロードが
`deliver` に到達するまで、プレビューコンテンツをバッファリングしてください。
どちらかのフックが登録されている間は、プレビューをより早く開始する互換性ディスパッチャーで、
その先行プレビューを抑制する必要があります。新しいプレビューパスには、
[チャネル送信 API](/ja-JP/plugins/sdk-channel-outbound) の終了処理可能なライブプレビューヘルパーを使用してください。

## 移行

`runtime.channel.turn.*` ランタイムエイリアスは削除されました。次を使用してください。

- `runtime.channel.inbound.run(...)`: 未加工の受信イベント用。
- `runtime.channel.inbound.dispatchReply(...)`: 組み立て済みの応答コンテキスト用。
- `runtime.channel.inbound.buildContext(...)`: 受信コンテキストペイロード用。
- `runtime.channel.inbound.runPreparedReply(...)`: 非推奨。独自のディスパッチクロージャをすでに
  組み立てている、チャネル所有の準備済みディスパッチパスでのみ使用します。

新しい Plugin コードでは、`turn` という名前のチャネル API を導入しないでください。
モデルまたはエージェントのターンに関する語彙はエージェント/プロバイダーコード内に留め、
チャネル Plugin では受信、メッセージ、配信、応答という用語を使用してください。
