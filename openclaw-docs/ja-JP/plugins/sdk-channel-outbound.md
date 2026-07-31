---
read_when:
    - メッセージングチャネル Plugin の送信パスを構築またはリファクタリングしている場合
    - 永続的な最終応答の配信、受領確認、ライブプレビューの確定、または受信確認ポリシーが必要な場合
    - チャンネルメッセージまたは従来の返信ディスパッチヘルパーから移行しています
summary: チャンネル Plugin 向け送信メッセージライフサイクル API：アダプター、受領確認、永続的な送信、ライブプレビュー、返信パイプラインヘルパー
title: チャンネル送信 API
x-i18n:
    generated_at: "2026-07-26T09:45:16Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 8edeca81d2e9261f33be1d538153caaea87caedb90dfccac33dd227c924501f1
    source_path: plugins/sdk-channel-outbound.md
    workflow: 16
---

チャネル Plugin は、
`openclaw/plugin-sdk/channel-outbound` から送信メッセージの動作を公開します。受信／コンテキスト／ディスパッチの
オーケストレーションには
`openclaw/plugin-sdk/channel-inbound` を使用します。

コアは、キューイング、耐久性、永続的な **イングレス監視とドレイン**
（`createChannelIngressMonitor`、`createChannelIngressDrain`、
`openChannelIngressDrain`）、汎用再試行ポリシー、ターン引き継ぎライフサイクル
（`turnAdoptionLifecycle` / `bindIngressLifecycleToReplyOptions`）、フック、
受領情報、共有 `message` ツールを所有します。Plugin は、ネイティブの
送信／編集／削除呼び出し、ターゲットの正規化、プラットフォームのスレッド処理、選択された
引用、通知フラグ、アカウント状態、イングレス検査とペイロードの
エンコード、レーンキー、再試行不可の述語、任意の置き換え
承認、プラットフォーム固有の副作用を所有します。

## 永続的なイングレスモニター

チャネルが受理したトランスポートイベントをディスパッチ前に永続化する必要がある場合は、`createChannelIngressMonitor(...)` を使用します。これは、チャネルのイングレスキューとドレインを、
共有の受け入れ、ポーリング、プルーニング、配信、シャットダウンのライフサイクルと
組み合わせます。トランスポートが実質的に異なる受け入れ契約またはポンプ契約を
所有する場合に限り、低レベルの `createChannelIngressDrain(...)` を使用します。

必須オプションは次のとおりです。

| オプション                           | 契約                                                                                                                                                                                                                                                                                                         |
| -------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `queue`                          | `ChannelIngressQueue`、またはアカウントスコープのキューを開く遅延ファクトリ。                                                                                                                                                                                                                                  |
| `inspect(raw, context)`          | 安定した `eventId` とシリアル化された `laneKey` を返します。無視されるイベントの場合は `null` を返します。クレーム時の情報は、永続化された ID およびレーンと一致する必要があります。                                                                                                                                                                    |
| `payload`                        | ペイロードのバージョンと、本文のシリアル化／逆シリアル化を提供します。標準の `{ version, rawEvent }` 文字列エンベロープには `storage: "raw-event"` を使用し、既存のチャネル固有形式にはカスタムのエンコード／デコードコールバックを指定します。`createClaimError` は、無効なバージョンまたは変更された ID を分類します。 |
| `deliver(raw, lifecycle, claim)` | デコードされたイベントを 1 件ディスパッチし、完全な引き継ぎライフサイクルを受け取ります。`completed`、`deferred`、`failed-retryable`、または何も返さない場合があります。                                                                                                                                                                |
| `pollIntervalMs`                 | モニターの実行中に、復旧／ドレインのポーリングをスケジュールします。                                                                                                                                                                                                                                                     |
| `retention`                      | プルーニング間隔と、完了／失敗の TTL およびエントリ上限を提供します。                                                                                                                                                                                                                                              |

モニターは受け入れを直列化するため、追加時のバックオフによってレーンの順序が
逆転することはありません。デフォルトの上限付き追加遅延は `0`、`100`、`300` ms です。これを使い切ると、
永続化されなかったイベントをディスパッチする代わりに、トランスポートのコールバックを
拒否します。クレーム時には、バージョン付きペイロードをデコードし、`inspect` を再実行して、
配信前に ID またはレーンの不一致を拒否します。

`deliver` は、`onAdopted`、`onDeferred`、`onAdoptionFinalizing`、
`onAbandoned`、`abortSignal` を受け取ります。明示的に引き渡さずに戻ると、
ディスパッチされない終端イベントが引き継がれたものとしてマークされます。`admission` は常に `exclusive` です。
遅延された引き渡しではクレームが保持されますが、シャットダウンまたは中止の場合、引き継がれていない
作業は再試行可能なままです。引き継ぎによって、チャネルの配信 Promise が
戻る前に行がトゥームストーン化される場合があるため、モニターは配信をクレームの決着とは
独立して追跡します。

任意設定には、カスタムの追加遅延、高度なドレイン順序／並行処理／再試行ポリシー向けの `drain` オプションブロック、外部の `abortSignal`、
クロック、ポンプエラーの報告、停止エラーのファクトリ、受け入れポリシーが含まれます。
返されるモニターは、`admit`、`start`、`pause`、`stop`、`waitForIdle`、
`isRunning`、`isStopped` を公開します。`stop` は、まず受理済みの受け入れを決着させ、次に
ドレインを中止して破棄し、ポンプとアクティブな配信を待機してから、
遅延作成の競合を解消するために再度破棄します。

トランスポート固有の秘匿化、未加工エンベロープの検証、再試行不可の
分類、永続化するペイロード形式は Plugin に保持します。Webhook トランスポートは、
`admit` が解決した後にのみ応答を返す必要があります。再生できないトランスポートは、
黙ってディスパッチするのではなく、永続的な追加の試行上限到達を表面化する必要があります。

## アダプター

ほとんどの Plugin は、`message` アダプターを 1 つ定義します。

```ts
import {
  defineChannelMessageAdapter,
  createMessageReceiptFromOutboundResults,
} from "openclaw/plugin-sdk/channel-outbound";

export const demoMessageAdapter = defineChannelMessageAdapter({
  id: "demo",
  durableFinal: {
    capabilities: {
      text: true,
      replyTo: true,
      thread: true,
      messageSendingHooks: true,
    },
  },
  send: {
    text: async ({ cfg, to, text, accountId, replyToId, threadId, signal }) => {
      const sent = await sendDemoMessage({
        cfg,
        to,
        text,
        accountId: accountId ?? undefined,
        replyToId: replyToId ?? undefined,
        threadId: threadId == null ? undefined : String(threadId),
        signal,
      });

      return {
        receipt: createMessageReceiptFromOutboundResults({
          results: [{ channel: "demo", messageId: sent.id, conversationId: to }],
          kind: "text",
          threadId: threadId == null ? undefined : String(threadId),
          replyToId: replyToId ?? undefined,
        }),
      };
    },
  },
});
```

ネイティブトランスポートが実際に保持する機能だけを宣言してください。宣言した
各送信、受領情報、ライブプレビュー、受信確認の機能を、
このサブパスからエクスポートされる契約ヘルパーで網羅してください。

## 送信メッセージのエコー抑制

プラットフォームが Plugin 自身の送信メッセージを受信メッセージとして再配信する可能性がある場合は、チャネル、アカウント、会話、および安定したプラットフォームメッセージまたは送信元 ID を指定して `recordOutboundMessageIdentity(...)` を呼び出します。共有の受信ターンパスは、セッション記録またはエージェントへのディスパッチ前に、一致する ID を上限付きの 30 秒間ウィンドウで破棄します。配信の競合を解消するために、送信元 ID を送信前に予約したり、チャネルルートが削除されたときに更新したりできます。`isRecentOutboundMessageIdentity(...)` は、チャネル診断およびテスト向けに同じクエリを公開します。同じ安定した ID のために、並行するチャネルローカルの TTL キャッシュを維持しないでください。

## プレーンテキストのサニタイズ

送信アダプターで、サポート対象の HTML 書式タグを軽量な
テキストマークアップに変換する必要がある場合は、`sanitizeForPlainText(...)` を使用します。デフォルトでは、
既存のチャット形式の太字および取り消し線マーカーが維持されます。チャネルが結果を Markdown として
再解析する場合に限り、`{ style: "markdown" }` を渡します。

```ts
import { sanitizeForPlainText } from "openclaw/plugin-sdk/channel-outbound";

const chatText = sanitizeForPlainText(text);
const markdownText = sanitizeForPlainText(text, { style: "markdown" });
```

Markdown スタイルでは `**bold**` と `~~strikethrough~~` を使用します。斜体とインライン
コードでは、どちらのスタイルでも `_italic_` とバッククォートマーカーが維持されます。サニタイズ後に
マーカーテキストを書き換えるのではなく、チャネル境界でスタイルを選択してください。

## 配信証拠

`MessageReceipt` は、チャネルアダプターが返した結果を記録します。具体的な
プラットフォームメッセージ ID は、プラットフォームの送信パスが
メッセージを受理したことを示しますが、受信者のデバイスに表示された、または既読になったことを
証明するものではありません。プラットフォームメッセージ ID のない受領情報は、
ローカルの受領メタデータにすぎません。既読通知またはデバイス配信状態を備えるチャネルは、
それらの情報をチャネル固有の別個のパスで追跡する必要があります。

失敗を再試行しても受信者に表示される送信が重複せず、
かつ確定処理が可能な呼び出しが開始されていないことをチャネルアダプターが証明できる場合は、
`openclaw/plugin-sdk/error-runtime` から
`new PlatformMessageNotDispatchedError("...", { cause: error })` をスローします。これによりコアは、古い送信試行の
証拠を消去し、キューに入ったインテントを安全に再試行できます。この表明を行えるのは、
最終ディスパッチ境界を所有するアダプターのみです。確定処理／送信呼び出しの開始後、または
曖昧な結果が返された後には、このマーカーを決して使用しないでください。誤ってマークすると、
メッセージが重複する可能性があります。

## 既存の送信アダプター

チャネルに互換性のある `outbound` アダプターがすでにある場合は、送信コードを
重複させるのではなく、そこからメッセージアダプターを派生させます。

```ts
import { createChannelMessageAdapterFromOutbound } from "openclaw/plugin-sdk/channel-outbound";

export const messageAdapter = createChannelMessageAdapterFromOutbound({
  id: "demo",
  outbound,
  durableFinal: {
    capabilities: {
      text: true,
      media: true,
    },
  },
});
```

## 永続的な送信

ランタイム送信ヘルパーも `channel-outbound` にあります。

- `sendDurableMessageBatch(...)`
- `withDurableMessageSendContext(...)`
- `deliverInboundReplyWithMessageSendContext(...)`
- `resolveChannelDraftStreamingChunking(...)` などのドラフトストリーミング／進捗ヘルパー

`sendDurableMessageBatch(...)` は、次の明示的な結果のいずれかを返します。

| 結果          | 意味                                                                                 |
| ---------------- | --------------------------------------------------------------------------------------- |
| `sent`           | 表示可能なプラットフォームメッセージが少なくとも 1 件、プラットフォームの送信パスによって受理された            |
| `suppressed`     | プラットフォームメッセージが欠落したものとして扱われるべきではない                                        |
| `partial_failed` | 後続のペイロードまたは副作用が失敗する前に、少なくとも 1 件のプラットフォームメッセージが受理された |
| `failed`         | プラットフォームの受領情報が生成されなかった                                                        |

バッチ内に送信済み、抑制済み、失敗した
ペイロードが混在する場合は、`payloadOutcomes` を使用します。空のレガシーな
直接配信結果からフックのキャンセルを推測しないでください。

## 遅延配信の受け入れ

解決済みのアカウントが、コア管理の送信または遅延配信を
安全に受け入れられない場合は、`message.durableFinal.admitDeferredDelivery(...)` を使用します。コアは、
キューへの永続化を省略するパスを含むライブ送信処理の前に、このフックを同期的に呼び出し、
復旧したインテントを再生する前にも再度呼び出します。コンテキストには、
`cfg`、`channel`、`to`、`accountId`、および
`live` または `recovery` の `phase` が含まれます。

続行するには `{ status: "allowed" }` を返します。配信を
永続化、直接送信、または再生してはならない場合は、
`{ status: "permanent_rejection", reason }` を返します。ライブでの拒否は、キュー作成、
メッセージフック、プラットフォーム処理の前に失敗します。復旧時の拒否は、
キュー内のレコードを失敗としてマークし、調整と再生を省略します。フックを省略した場合は、
許可されたものとみなされます。

このフックは同期的な受け入れ判定であり、送信経路ではありません。すでに
読み込まれている設定またはランタイム状態のみを読み取り、ネットワーク、ファイルシステム、
その他の非同期 I/O を実行しないでください。契約テストでは、
`openclaw/plugin-sdk/channel-outbound` の `ChannelMessageDurableFinalAdapter` を通じて、両方のフェーズと
両方の結果バリアントを検証する必要があります。

## 互換性ディスパッチ

`channel-inbound` の `dispatchChannelInboundReply(...)` を通じて、
受信返信のディスパッチを構成します。プラットフォームへの配信は配信アダプター内に維持し、
メッセージアダプター、永続的な送信、受領確認、ライブプレビュー、
返信パイプラインのオプションには `channel-outbound` を使用します。
