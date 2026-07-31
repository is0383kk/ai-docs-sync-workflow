---
read_when:
    - チャンネルの送信または受信動作のリファクタリング
    - チャネルの受信、返信ディスパッチ、送信キュー、プレビューストリーミング、または Plugin SDK メッセージ API の変更
    - 永続的な送信、受信確認、プレビュー、編集、再試行を必要とする新しいチャンネル Plugin の設計
summary: 永続的なメッセージ受信／送信ライフサイクルの状況：リリース済みの内容、当初の設計からの変更点、未解決の課題
title: メッセージライフサイクルのリファクタリング
x-i18n:
    generated_at: "2026-07-26T10:11:36Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: d21eda70b8be0de78677f4ff6d7547317112731d9e86a5bef58eac0268899818
    source_path: concepts/message-lifecycle-refactor.md
    workflow: 16
---

<Note>
このページは、将来を見据えた設計提案として作成されました。その設計の中核はその後、`src/channels/message/*` および公開
`openclaw/plugin-sdk/channel-outbound` / `channel-inbound` サブパスに実装されました。現在の API については、[チャンネル送信 API](/ja-JP/plugins/sdk-channel-outbound) と
[チャンネル受信 API](/ja-JP/plugins/sdk-channel-inbound) を参照してください。このページでは、実装済みの内容、実装が当初の草案と異なる点、および未解決の事項を記録します。
</Note>

## このリファクタリングが行われた理由

チャンネルスタックは、複数の局所的な修正から発展しました。成熟度ごとの個別の受信ヘルパー（単純なアダプター向けの
`runtime.channel.inbound.run`、高機能なアダプター向けの
`runtime.channel.inbound.runPreparedReply`）、レガシーな返信ディスパッチヘルパー
（`dispatchInboundReplyWithBase`、`recordInboundSessionAndDispatchReply`）、
チャンネル固有のプレビューストリーミング、既存の返信ペイロード経路に後付けされた最終配信の耐久性などです。この構造により、公開される概念が多すぎるうえ、配信セマンティクスが乖離し得る箇所も多すぎました。

再設計を余儀なくした信頼性上の欠陥は次のとおりです。

```text
Telegram のポーリング更新が ack される
  -> アシスタントの最終テキストが存在する
  -> sendMessage が成功する前にプロセスが再起動する
  -> 最終応答が失われる
```

目標とする不変条件は、可視の送信メッセージが存在すべきだとコアが決定した時点で、プラットフォーム呼び出しを試行する前に送信意図が永続化され、成功後にプラットフォームの受領情報がコミットされることです。これにより、デフォルトで少なくとも1回の復旧が可能になります。厳密に1回の動作が成立するのは、アダプターがネイティブの冪等性を証明する場合、または送信後の成否不明な試行を再実行前にプラットフォームの状態と照合する場合に限られます。

## 実装済みの内容

内部ドメインは `src/channels/message/*` にあります。

| ファイル                        | 担当                                                                                                               |
| --------------------------- | ------------------------------------------------------------------------------------------------------------------ |
| `types.ts`                  | アダプター、送信コンテキスト、受領情報、永続的な意図の型コントラクト                                                  |
| `send.ts`                   | `withDurableMessageSendContext` / `sendDurableMessageBatch` — 永続的な送信コンテキスト                             |
| `receive.ts`                | `createMessageReceiveContext` — 受信 ack ポリシーのステートマシン                                                   |
| `live.ts`                   | ライブプレビューの状態と、その場で確定するかフォールバックするロジック                                                        |
| `state.ts`                  | `classifyDurableSendRecoveryState` — 中断後の復旧分類                                    |
| `receipt.ts`                | プラットフォームの送信結果を `MessageReceipt` に正規化                                                             |
| `capabilities.ts`           | ペイロードから必要な永続的最終配信機能を導出                                                         |
| `contracts.ts`              | 宣言されたアダプター機能のコントラクト証明を検証                                                      |
| `adapter.ts`                | `defineChannelMessageAdapter`                                                                                      |
| `outbound-bridge.ts`        | `createChannelMessageAdapterFromOutbound` — レガシーな `sendText`/`sendMedia`/`sendPayload`/`sendPoll` 関数をラップ |
| `ingress-queue.ts`          | `createChannelIngressQueue` — 永続的な受信イベントキュー                                                          |
| `durable-receive.ts`        | `createDurableInboundReceiveJournal` — 受信重複排除用の受け入れ/保留/完了/解放ジャーナル                  |
| `inbound-reply-dispatch.ts` | `dispatchChannelInboundReply` とレガシー名のラッパー                                                            |
| `reply-pipeline.ts`         | `createChannelReplyPipeline`、返信プレフィックス、入力中コールバックのヘルパー                                             |

公開サーフェスは、`openclaw/plugin-sdk/channel-outbound`（送信/受領情報/永続化/ライブ/返信パイプラインのヘルパー）と `openclaw/plugin-sdk/channel-inbound`（受信コンテキスト、`runChannelInboundEvent`、
`dispatchChannelInboundReply`）です。アダプターの例、現在の型名、移行に関する注意事項については、これらのページを参照してください。以下の草案ではなく、これらが API 構造の信頼できる唯一の情報源です。

### 送信コンテキスト

`withDurableMessageSendContext` は、1つの送信メッセージに関して、チャンネルコードに `render`、`previewUpdate`、
`send`、`edit`、`delete`、`commit`、`fail` の各ステップを提供します。`sendDurableMessageBatch` は一般的なケース向けのラッパーです。レンダリングして送信し、`sent`/`suppressed` の場合はコミットし、エラー時には失敗として処理します。

`sendDurableMessageBatch` は、次のいずれかの判別可能な結果を返します。

| ステータス           | 意味                                                                          |
| ---------------- | -------------------------------------------------------------------------------- |
| `sent`           | 可視のプラットフォームメッセージが少なくとも1件配信された                              |
| `suppressed`     | プラットフォームメッセージが欠落したものとして扱うべきではない（フックによるキャンセル、ドライランなど） |
| `partial_failed` | 後続のペイロードまたは副作用が失敗する前に、少なくとも1件のメッセージが配信された      |
| `failed`         | プラットフォームの受領情報が生成されなかった                                                 |

耐久性は、`required`、`best_effort`、`disabled` のいずれかです
（`src/channels/message/types.ts` 内の `MessageDurabilityPolicy`）。`required` は永続的な意図を書き込めない場合にフェイルクローズします。`best_effort` は永続化を利用できない場合に直接送信へ移行し、`disabled` はリファクタリング前の直接送信動作を維持します。レガシー互換ヘルパーはデフォルトで
`disabled` を使用し、チャンネルに汎用の送信アダプターがあるという理由だけで `required` を推論することはありません。

依然として危険なのは、プラットフォーム呼び出しが成功してから受領情報がコミットされるまでの境界です。その間にプロセスが停止した場合、アダプターが `reconcileUnknownSend` を宣言していない限り、プラットフォームメッセージが存在するかどうかをコアは判断できません。このフックは、中断された送信を `sent`、`not_sent`、`unresolved` のいずれかに分類します。再実行を許可するのは `not_sent` のみです。照合機能のないチャンネルは `unknown_after_send` 状態
（`src/channels/message/state.ts`、`src/infra/outbound/delivery-queue-recovery.ts`）にフォールバックします。可視メッセージの重複がそのチャンネルにとって許容可能で文書化されたトレードオフである場合に限り、少なくとも1回の再実行を選択できます。

### 受信コンテキスト

`createMessageReceiveContext` は、各受信イベントの ack/nack 状態を、冪等な `ack()` と明示的な `nack(error)` によって追跡します。ack ポリシー
（`ChannelMessageReceiveAckPolicy`）は次のいずれかです。

| ポリシー                 | ack するタイミング                                                                                     |
| ---------------------- | --------------------------------------------------------------------------------------------- |
| `after_receive_record` | 再配信を重複排除またはルーティングするために十分な受信メタデータをコアが永続化した時                           |
| `after_agent_dispatch` | エージェントの実行がディスパッチされた時                                                             |
| `after_durable_send`   | このターンの永続的な送信がコミットされた時                                             |
| `manual`               | 呼び出し元が ack のタイミングを明示的に制御する（ポリシーを宣言しないアダプターのデフォルト） |

Telegram のポーリングでは、これを使用して安全に完了した更新のウォーターマーク
（`extensions/telegram/src/bot-update-tracker.ts` 内の `safeCompletedUpdateId`）を永続化します。
grammY は引き続き、ミドルウェアチェーンに入るすべての更新を監視しますが、OpenClaw はディスパッチが完了した更新より先に永続化された再起動ウォーターマークを進めません。そのため、失敗した更新やまだ保留中の更新は、再起動後に再実行されます。Telegram のアップストリーム `getUpdates` オフセットは引き続き grammY が所有します。このウォーターマークを超えてプラットフォームレベルの再配信を制御する、完全に永続的なポーリングソースは構築されていません（「未解決の問題」を参照）。

### ライブプレビュー

`src/channels/message/live.ts` は、プレビュー/編集/確定を1つのライフサイクルとしてモデル化します。
`createLiveMessageState`、`markLiveMessagePreviewUpdated`、
`markLiveMessageFinalized`、`markLiveMessageCancelled`、および
`deliverFinalizableLivePreviewAdapter`（下書きから最終編集を構築して適用し、編集が不可能または失敗した場合は通常の送信にフォールバック）です。
`LiveMessageState.phase` は `idle | previewing | finalizing | finalized |
cancelled` です。`canFinalizeInPlace` は、プレビューを新規送信ではなく編集によって最終メッセージにできるかどうかを制御します。

### 永続的な受領情報

`MessageReceipt`（`src/channels/message/types.ts`）は、1回の論理送信から得られる1つ以上のプラットフォームメッセージ ID を、`platformMessageIds` と、各パートの
`parts`（種類、インデックス、スレッド ID、返信先 ID）へ正規化します。スレッド化と後続の編集のために、プライマリ ID が保持されます。これにより、複数パートの配信（テキストとメディア、分割されたテキスト、カードのフォールバック）を再起動後に再実行し、重複排除できます。

### 公開 SDK の縮小

このリファクタリングでは、`reply-runtime`、`reply-dispatch-runtime`、
`reply-reference`、`reply-chunking`、公開 API として公開されていた `reply-payload` ヘルパー、`inbound-reply-dispatch`、`channel-reply-pipeline`、および旧送信ファサードの公開用途の大部分が吸収または非推奨化されました。`src/plugin-sdk/channel-message.ts` は現在、`channel-outbound` /
`channel-inbound` を参照する `@deprecated` 再エクスポートバレルです。`channel.turn` ランタイムエイリアスは削除され、旧
`/plugins/sdk-channel-turn` ドキュメントページは
[チャンネル受信 API](/ja-JP/plugins/sdk-channel-inbound) にリダイレクトされます。新しい Plugin コードでは、`channel-outbound` と `channel-inbound` を直接対象にしてください。

## 実装が当初の設計と異なる点

以下の設計草案は、記載どおりの形では実装されませんでした。歴史的な正確性のために記録を残していますが、これらの型名を現在の API として扱わないでください。

- **`MessageOrigin` / `shouldDropOpenClawEcho` はありません。** 当初の計画では、Gateway 障害メッセージに `source: "openclaw"` オリジンタグを付け、`allowBots` の認可前に、共有ルーム内でタグ付けされたボット作成のエコーを除外する共通述語を用意することになっていました。その型と述語はコードベースに存在しません。`allowBots` 自体は実際のチャンネル単位の設定キー（Slack、Discord、Google Chat など）ですが、それを保護するためのオリジンタグ付け機構は構築されませんでした。ボットが有効なルームでの Gateway 障害エコーの抑制は、実装済みの保証ではなく、未解決の欠陥として残っています。
- **統一された `core.messages.receive/send/live/state` 名前空間はありません。** 実装された関数は、`core.messages.*` ファサードの背後ではなく、`src/channels/message/*` に直接配置されています
（`withDurableMessageSendContext`、`createMessageReceiveContext`、
`createLiveMessageState`、`classifyDurableSendRecoveryState`）。
- **汎用の `ChannelMessage` / `MessageTarget` / `MessageRelation` 正規化メッセージ型はありません。** コアは、`kind: "reply" |
"followup" | "broadcast" | "system"` 関係を持つ単一のプラットフォーム中立なメッセージ構造ではなく、具体的な返信ペイロード
（`ReplyPayload`）とチャンネル固有のコンテキストを送信アダプターに引き続き渡します。
- **ack ポリシー名は草案と異なります。** 実装済み:
  `after_receive_record | after_agent_dispatch | after_durable_send | manual`。
  当初の草案では、Webhook タイムアウトの理由フィールドを持つ `immediate | after-record | after-durable-send |
manual` を使用していましたが、その構造は構築されませんでした。
- **`DurableFinalDeliveryRequirementMap` 機能キーが、草案の
  `MessageCapabilities` オブジェクトを置き換えました。** 機能は、ネストされた `text.chunking` / `attachments.voice` 形式の構造ではなく、`verifyDurableFinalCapabilityProofs` によって検証されるフラットなブールフラグ
（`text`、
  `media`、`poll`、`payload`、`silent`、`replyTo`、`thread`、`nativeQuote`、
  `messageSendingHooks`、`batch`、`reconcileUnknownSend`、`afterSendSuccess`、
  `afterCommit`）です。

## 具体的な移行上の危険性（現在も該当）

これらのチャンネル固有の副作用はリファクタリング以前から存在し、新しい送信経路でも引き続き動作する必要があります。これらは仮定上のものではなく、いずれも現在実装され、重要な役割を担っています。

- **iMessage**（`extensions/imessage/src/monitor/echo-cache.ts`、
  `persisted-echo-cache.ts`）：モニターは送信が成功した後、送信済みメッセージをエコー
  キャッシュに記録します。永続的な最終送信でもこのキャッシュを更新する必要が
  あります。そうしないと、OpenClaw が自身の返信を受信ユーザーメッセージとして再取り込みする可能性があります。
- **Tlon**（`extensions/tlon/src/monitor/index.ts`）：オプションのモデル
  署名を追加し、グループ返信後に参加したスレッドを記録します。永続的な
  配信でこれらの処理を迂回してはなりません。
- **Discord およびその他の準備済みディスパッチャー**は、すでに直接配信と
  プレビュー動作を担っています。準備済みディスパッチャーが最終メッセージを送信コンテキスト経由に
  明示的にルーティングするまでは、チャネルはエンドツーエンドで永続的ではありません。汎用
  アダプターだけで対応できると想定しないでください。
- **Telegram のサイレントフォールバック配信**では、チャンク化／フォールバック
  プロジェクション後に、最初のペイロードだけでなく、プロジェクションされた
  ペイロード配列全体を配信する必要があります。
- **LINE、Zalo、Nostr**、および同様のヘルパーパスには、返信トークンの
  処理、メディアのプロキシ、送信済みメッセージのキャッシュ、またはコールバック専用の宛先が
  存在する場合があります。これらのセマンティクスが送信アダプターで表現され、
  テストで網羅されるまでは、チャネル所有の配信を維持します。
- **ダイレクト DM ヘルパー**には、唯一の正しい
  トランスポート宛先となる返信コールバックが存在する場合があります。汎用の送信処理が生の
  プラットフォームフィールドから宛先を推測し、そのコールバックを迂回してはなりません。

## 障害分類

アダプターはトランスポート障害を `DeliveryFailureKind` 形式の閉じた
カテゴリ（一時的、レート制限、認証、権限、未検出、無効な
ペイロード、競合、キャンセル、不明）に分類します。コアポリシー：

- 一時的な障害とレート制限による障害は再試行します。
- レンダリングのフォールバックが存在しない限り、無効なペイロードによる障害は再試行しません。
- 設定が変更されるまで、認証または権限による障害は再試行しません。
- 未検出の場合、チャネルが安全であると宣言していれば、ライブ最終処理で編集から新規送信への
  フォールバックを許可します。
- 競合の場合、受領情報／冪等性の状態を使用して、メッセージが
  すでに存在するかどうかを判断します。
- プラットフォーム呼び出しは成功した可能性があるものの、受領情報の
  コミット前に発生したエラーは、アダプターがプラットフォーム操作が行われなかったことを
  証明しない限り、`unknown_after_send` になります。

## 未解決の問題

- Telegram が最終的に grammY（`1.43.0`）のポーリング
  ランナーを、OpenClaw の永続化された再起動ウォーターマーク
  （`safeCompletedUpdateId`）だけでなく、プラットフォームレベルの再配信を制御する完全に永続的な
  ポーリングソースに置き換えるべきか。
- ライブプレビュー状態を最終送信
  インテントと同じレコードに格納すべきか、同階層のライブ状態ストアに格納すべきか。
- 共有されたボット対応ルームでの Gateway 障害時のエコー抑制に、
  当初計画されていたオリジンタグ付け機構が必要か、より単純なチャネルごとの
  契約で十分か、または対象範囲外か。
- ボット間のエコー抑制について、ネイティブのオリジン／メタデータ対応を持つ
  チャネルと、永続化された送信レジストリを必要とするチャネルはどれか。

## 関連項目

- [メッセージ](/ja-JP/concepts/messages)
- [ストリーミングとチャンク化](/ja-JP/concepts/streaming)
- [進捗ドラフト](/ja-JP/concepts/progress-drafts)
- [再試行ポリシー](/ja-JP/concepts/retry)
- [チャネル送信 API](/ja-JP/plugins/sdk-channel-outbound)
- [チャネル受信 API](/ja-JP/plugins/sdk-channel-inbound)
