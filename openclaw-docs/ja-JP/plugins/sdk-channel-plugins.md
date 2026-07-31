---
read_when:
    - 新しいメッセージングチャネル Plugin を構築しています
    - OpenClaw をメッセージングプラットフォームに接続する場合
    - ChannelPlugin アダプターのインターフェースを理解する必要があります
sidebarTitle: Channel Plugins
summary: OpenClaw向けメッセージングチャネルPluginの構築手順ガイド
title: チャネル Plugin の構築
x-i18n:
    generated_at: "2026-07-26T09:54:51Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 0ff8ad04346babf3eece7e10bd38946ee290947b2e504b6b5ca438865531bf38
    source_path: plugins/sdk-channel-plugins.md
    workflow: 16
---

このガイドでは、OpenClaw をメッセージングプラットフォームに接続するチャンネル Plugin を構築します。DM のセキュリティ、ペアリング、返信のスレッド化、送信メッセージについて説明します。

<Info>
  OpenClaw Plugin を初めて使用する場合は、まずパッケージ構造とマニフェストの設定について[はじめに](/ja-JP/plugins/building-plugins)をお読みください。
</Info>

## Plugin が担うもの

チャンネル Plugin は送信、編集、リアクションのツールを実装しません。コアが共有の
`message` ツールを 1 つ提供します。Plugin が担うものは次のとおりです。

- **設定** - アカウントの解決とセットアップウィザード
- **セキュリティ** - DM ポリシーと許可リスト
- **ペアリング** - DM 承認フロー
- **セッション文法** - プロバイダー固有の会話 ID をベースチャット、
  スレッド ID、親フォールバックにマッピングする方法
- **送信** - プラットフォームへのテキスト、メディア、投票の送信
- **スレッド化** - 返信をスレッド化する方法
- **Heartbeat の入力中表示** - Heartbeat の配信先に対する任意の入力中／処理中シグナル

コアは、共有メッセージツール、プロンプトの接続、外側のセッションキー形式、
汎用の `:thread:` 管理、ディスパッチを担います。

## メッセージアダプター

`openclaw/plugin-sdk/channel-outbound` の `defineChannelMessageAdapter` を使用して
`message` アダプターを公開します。ネイティブトランスポートが実際にサポートする永続的な最終送信機能のみを宣言し、ネイティブ側の副作用と返される受領情報を証明するコントラクトテストで裏付けます。テキスト／メディア送信は、従来の `outbound` アダプターが使用するものと同じトランスポート関数に接続します。完全な API コントラクト、機能マトリクス、受領情報の規則、ライブプレビューの確定、受信確認ポリシー、テスト、移行表については、[チャンネル送信 API](/ja-JP/plugins/sdk-channel-outbound)を参照してください。

既存の `outbound` アダプターに適切な送信メソッドと機能メタデータがすでにある場合は、別のブリッジを手書きせず、`createChannelMessageAdapterFromOutbound(...)` を使用して
`message` アダプターを派生させます。アダプターによる送信は `MessageReceipt` 値を返します。従来の ID については、`messageIds` フィールドを並行して維持するのではなく、`listMessageReceiptPlatformIds(...)` または
`resolveMessageReceiptPrimaryId(...)` で派生させます。

ライブ機能とファイナライザー機能を正確に宣言してください。コアはこれらを使用してチャンネルが実行できることを判断し、宣言された動作と実際の動作の不一致はコントラクトテストの失敗となります。

| サーフェス                            | 値                                                                                               |
| ------------------------------------- | ------------------------------------------------------------------------------------------------ |
| `message.live.capabilities`           | `draftPreview`、`previewFinalization`、`progressUpdates`、`nativeStreaming`、`quietFinalization` |
| `message.live.finalizer.capabilities` | `finalEdit`、`normalFallback`、`discardPending`、`previewReceipt`、`retainOnAmbiguousFailure`    |

下書きプレビューをその場で確定するチャンネルは、ランタイムロジックを
`defineFinalizableLivePreviewAdapter(...)` と
`deliverWithFinalizableLivePreviewAdapter(...)` を通じて処理し、宣言された機能を
`verifyChannelMessageLiveCapabilityAdapterProofs(...)` および
`verifyChannelMessageLiveFinalizerProofs(...)` のテストで裏付ける必要があります。これにより、ネイティブのプレビュー、進捗、編集、フォールバック／保持、クリーンアップ、受領情報の動作が暗黙のうちにずれることを防ぎます。

プラットフォームへの確認応答を遅延させる受信処理は、確認応答のタイミングをモニター内のローカル状態に隠すのではなく、
`message.receive.defaultAckPolicy` と `supportedAckPolicies` を宣言する必要があります。宣言したすべてのポリシーを
`verifyChannelMessageReceiveAckPolicyAdapterProofs(...)` で網羅してください。

`dispatchInboundReplyWithBase` や
`recordInboundSessionAndDispatchReply` などの従来の返信ヘルパーは、互換性ディスパッチャー向けに引き続き利用できます。新しいチャンネルコードでは使用しないでください。代わりに、`message` アダプター、受領情報、および
`openclaw/plugin-sdk/channel-outbound` の受信／送信ライフサイクルヘルパーから始めてください。

### 受信イングレス（試験的）

受信認可を移行するチャンネルは、ランタイムの受信パスから試験的な
`openclaw/plugin-sdk/channel-ingress-runtime` サブパスを使用できます。これは、プラットフォーム情報、生の許可リスト、ルート記述子、コマンド情報、アクセスグループ設定を受け取り、送信者／ルート／コマンド／アクティベーションの投影と順序付けされたイングレスグラフを返します。一方、プラットフォームの検索と副作用は Plugin 内に残ります。Plugin の ID 正規化はリゾルバーに渡す記述子内に保持し、解決済みの状態や決定から生の一致値をシリアライズしないでください。API 設計、所有権の境界、テスト要件については、[チャンネルイングレス API](/ja-JP/plugins/sdk-channel-ingress)を参照してください。

### 永続的なイングレスとリプレイ重複排除

永続的なイングレスを採用するチャンネルは、実質的に異なる受け入れまたはポンプのコントラクトが必要でない限り、`openclaw/plugin-sdk/channel-outbound` の `createChannelIngressMonitor` を使用する必要があります。単一の受信チョークポイントで生のトランスポートエンベロープをキューに追加し（受信時には正規化しません）、Webhook トランスポートでは永続的な追加の成功を条件としてトランスポートの確認応答を行い、会話ごとに 1 つの直列化レーンを派生させ、ディスパッチによる引き受け時にイベントを完了としてマークします。キューの主キーは `(queue_name, event_id)` です。完了時には行を削除せずトゥームストーン化するため、同じ `event_id` がプラットフォームから遅れて再配信されても、トゥームストーンの保持期間中は永続的に拒否されます。モニター API とシャットダウンコントラクトについては、[チャンネル送信 API](/ja-JP/plugins/sdk-channel-outbound#durable-ingress-monitors)を参照してください。

このトゥームストーンが、リプレイガード
（`openclaw/plugin-sdk/persistent-dedupe`）のレイヤリング規則になります。ドレイン済みのチャンネルが別個のリプレイガードを維持するのは、そのガードの ID または保持期間がキューを上回る場合に限ります。つまり、トランスポート配信 ID とは異なる論理メッセージキー（Telegram では、デバウンスによるマージで新しい `update_id` の下にメッセージが再出現する可能性があるため、`chat_id:message_id` の重複を排除します）、またはチャンネルのトゥームストーン保持期間より長いウィンドウです。ガードキーがドレインの `event_id` と等しくなる場合は、ドレインの採用時にガードを削除し、代わりに `completedTtlMs`/`completedMaxEntries` が以前のガードウィンドウをカバーするようにサイズを設定します。経過時間フェンスなど、重複排除以外の保護はこの規則とは無関係です。安定した送信メッセージ ID には、チャンネルローカルの TTL キャッシュではなく、`openclaw/plugin-sdk/channel-outbound` の共有送信エコーレジストリを使用します。

#### トランスポートクラスと保持

受信境界での復旧保証に基づいてトランスポートを分類します。

- **確認応答で制御される Webhook またはイベント配信：** 永続的な追加の後にのみ確認応答または成功を返します。追加に失敗した場合は、配信を再試行可能な状態に維持するか、受信境界を失敗させる必要があります。このクラスには Slack、SMS、Zalo、Microsoft Teams、Google Chat、LINE、Synology Chat が含まれます。
- **待機されるポーリングまたはストリーム配信：** 追加の後にのみリモートカーソルを進めるか、トランスポートの確認応答を送信します。明示的なカーソルがない場合は、受信コールバックを直列化して待機し、追加の失敗時に受信ループが先へ進めないようにします。Telegram のポーリング、Signal、Tlon はこのクラスを使用します。Telegram の Webhook 配信には、前述の確認応答で制御される規則が適用されます。
- **リプレイ不可のソケット：** IRC、Mattermost、Twitch、Zalo Personal は、受け入れ済みイベントの再配信をプラットフォームに要求できません。これらの永続キューはプロセスクラッシュの時間枠を保護し、ローカル再起動時の復旧を支援します。完了トゥームストーンは、プラットフォームのリプレイに対してほぼ効果を持ちません。

フリートのトゥームストーン TTL の慣例として 30 日を使用します。SDK のデフォルトではありません。大量の再配信が発生するウィンドウでは通常、完了済みの上限を 20,000 エントリとします。少量の待機型およびリプレイ不可トランスポートでは通常、1,000～2,000 とします。現在の例外には、LINE の上限 4,096 エントリ、SMS の完了済み TTL 24 時間、Tlon の上限のみによる完了済み保持があります。失敗行の上限も、完了済みの上限より低く設定される場合があります。TTL と上限はいずれも行を削除するため、有効な保持期間は先に到達した境界で終了します。文書化されたプラットフォームの再試行期間、維持すべきリリース済みリプレイガードのウィンドウ、予想される量またはディスク予算、あるいはリプレイ不可のトランスポートを理由とする場合に限り逸脱し、保持コントラクトをテストで網羅してください。

#### 少なくとも 1 回の副作用

ドレインのディスパッチは、イングレス行が完了トゥームストーンに到達する前にコマンドの副作用を実行します。この 2 つのステップの間にプロセスがクラッシュすると、行がリプレイされ、副作用が再度実行される可能性があります。この少なくとも 1 回実行されるクラッシュウィンドウがデフォルトのコントラクトです。設定の書き込み、ストレージの消去、返信レーン外での可視確認応答など、非冪等な処理には
`openclaw/plugin-sdk/ingress-effect-once` の
`createIngressEffectOnce(...)` を使用します。各呼び出しには、安定したイングレス
`eventId` とエフェクト名を指定します。イングレスのキュー／アカウントごとに 1 つのヘルパーを作成し、そのスコープには安定かつ一意の `namespacePrefix` を使用します。トランスポートイベント ID はキュー内でのみ一意な場合があるためです。ヘルパーはエフェクトが成功した後にのみ永続的なクレームをコミットします。エフェクトが例外をスローした場合はクレームを解放し、ドレインの再試行で再実行できるようにします。同時呼び出し元はアクティブなクレームを待機します。永続状態のエラーは、指定されている場合に `onDiskError` を呼び出し、プロセスメモリへフォールバックせずに拒否します。

ヘルパーの `ttlMs` は、少なくともチャンネルのイングレストゥームストーン保持期間に、エフェクトのコミットから行の完了までの最大遅延（上限付きのダウンタイムとドレインの再試行を含む）を加えた長さに設定します。エフェクトレコードの TTL はコミット時に開始しますが、トゥームストーンの保持はそれより後の完了時に開始します。保留行の存続期間に上限がない場合、有限の TTL では任意のダウンタイムをカバーできません。トゥームストーンが行をリプレイできなくなった後は、それより古いエフェクトレコードは不要になります。`stateMaxEntries` は、キューの完了済みエントリ上限とイベントごとの最大エフェクト数を考慮し、その保持ウィンドウ内に存在し得る個別のイベント／エフェクトキーをすべて収容できるサイズに設定します。上限が低いと、最も古いレコードが TTL より先に削除され、そのエフェクトが再実行される可能性があります。エフェクトの成功後、クレームのコミット前にプロセスが停止または永続化に失敗した場合、あるいはイングレス行がまだ保留中にレコードが期限切れになった場合は、少なくとも 1 回実行されるウィンドウが残ります。

#### アカウント単位の再起動コントラクト

チャンネル設定の変更では、デフォルトでチャンネル全体を再起動します。複数アカウント対応チャンネルが `reload.accountScopedRestart: true` を設定できるのは、設定の解決でチャンネル全体の共有フィールドと選択されたアカウントのみを読み取り、兄弟アカウントを決して読み取らず、かつ Gateway が兄弟ランタイムを置き換えることなく 1 つの `(channel, accountId)` ランタイムを停止および起動できる場合に限ります。

スコープ付きパスは、
`channels.<channel>.accounts.<non-default-id>.*` 配下の変更にのみ適用されます。共有チャンネルフィールド、`accounts.default`、削除済みまたは解決不能なアカウント、継承に影響を与え得る混在した変更は、チャンネル全体の再起動に昇格します。オプトインしない Plugin は常にチャンネル全体のパスを使用します。

永続的なイングレスドレインを使用するチャンネルでは、アカウントモニターの停止パスが、受け入れ済みのすべてのトランスポート受け入れ処理をまず完了させ、その後ドレインを破棄して完了を待機する必要があります。アカウントを起動すると、同じアカウントキーのキューが開かれ、その初期ドレインが未ディスパッチの永続行を復旧します。再読み込み固有の 2 回目のリプレイ処理を追加しないでください。キューの復旧が正規の再起動パスです。

このフラグは、パフォーマンス上の好みではなく機能の宣言として扱ってください。コントラクトテストでは、名前付きアカウントを 1 つ追加または編集しても兄弟アカウントの解決済み設定が変化しないこと、1 つのアカウントを停止してもそのアカウントのモニターとドレインのみが完了すること、新しいモニターがそのアカウントの行を正確に 1 回復旧することを証明する必要があります。いずれかの保証を証明できない場合は、このフラグを省略してください。

### 入力中インジケーター

チャンネルが受信返信以外でも入力中インジケーターをサポートする場合は、チャンネル Plugin で
`heartbeat.sendTyping(...)` を公開します。コアは Heartbeat のモデル実行開始前に、解決済みの Heartbeat 配信先を指定してこれを呼び出し、共有の入力中状態キープアライブ／クリーンアップライフサイクルを使用します。プラットフォームで明示的な停止シグナルが必要な場合は、
`heartbeat.clearTyping(...)` を追加します。

### メディアソースパラメーター

チャンネルがメディアソースを保持するメッセージツールパラメーターを追加する場合は、そのパラメーター名を
`plugin.actions.describeMessageTool(...).mediaSourceParams` を通じて公開します。コアはその明示的なリストをサンドボックスパスの正規化と送信メディアのアクセスポリシーに使用するため、Plugin はプロバイダー固有のアバター、添付ファイル、カバー画像のパラメーターについて共有コアに特別処理を追加する必要がありません。

アクションをキーとするマップ（`{ "set-profile": ["avatarUrl", "avatarPath"] }` など）を推奨します。
これにより、無関係なアクションが別のアクションのメディア引数を継承しません。フラット配列も、
公開されているすべてのアクションで意図的に共有するパラメーターには引き続き使用できます。

プラットフォーム側でメディアを取得するための一時的な公開 URL を公開する必要がある
チャンネルでは、Plugin 状態ストアとともに `openclaw/plugin-sdk/outbound-media` の
`createHostedOutboundMediaStore(...)` を使用できます。プラットフォームの
ルート解析とトークンの適用はチャンネル Plugin に保持してください。共有ヘルパーが
担当するのは、メディアの読み込み、有効期限メタデータ、チャンク行、クリーンアップのみです。

受信添付ファイルには、並列の `Media*` フィールドではなく、順序付きのファクトを使用します。チャンネルレコードを
`openclaw/plugin-sdk/channel-inbound` の `toInboundMediaFacts(...)` で正規化し、
受信コンテキストを構築するときに `media` として渡します。
Plugin がローカルメディアの読み取りを認可する必要がある場合は、対象を絞った
`openclaw/plugin-sdk/media-local-roots` サブパスから
`getAgentScopedMediaLocalRoots(...)` または
`getAgentScopedMediaLocalRootsForSources(...)` をインポートします。古い
`agent-media-payload` ビルダー／ルートファサードは、非推奨の互換機能です。

### ネイティブペイロードの整形

チャンネルで `message(action="send")` にプロバイダー固有の整形が必要な場合は、
`actions.prepareSendPayload(...)` を推奨します。ネイティブカード、ブロック、埋め込み、その他の
永続的なデータを `payload.channelData.<channel>` の下に配置し、コアが
送信／メッセージアダプターを介して送信するようにします。シリアライズおよび
再試行できないペイロードに限り、互換フォールバックとして送信時に
`actions.handleAction(...)` を使用します。

### セッション会話文法

プラットフォームが会話 ID 内に追加のスコープを保存する場合、その解析は
`messaging.resolveSessionConversation(...)` を使用して Plugin 内に保持します。これは、
`rawId` を基本会話 ID、任意のスレッド ID、明示的な
`baseConversationId`、および任意の
`parentConversationCandidates` にマッピングするための正規フックです。`parentConversationCandidates` を返す場合は、
最も狭い親から最も広い親／基本会話の順に並べます。

`messaging.resolveParentConversationCandidates(...)` は、
汎用／生の ID に親フォールバックを追加するだけでよい Plugin 向けの、非推奨の
互換フォールバックです。両方のフックが存在する場合、コアは最初に
`resolveSessionConversation(...).parentConversationCandidates` を使用し、正規フックで省略されている場合にのみ
`resolveParentConversationCandidates(...)` にフォールバックします。

チャンネルレジストリの起動前に同じ解析が必要なバンドル Plugin は、
対応する `resolveSessionConversation(...)` エクスポートを持つトップレベルの
`session-key-api.ts` ファイルを公開できます（Feishu および Telegram
Plugin を参照）。コアは、ランタイム Plugin レジストリがまだ利用できない場合に限り、
そのブートストラップセーフなサーフェスを使用します。

Plugin コードでルートのようなフィールドを正規化したり、子スレッドとその親ルートを比較したり、
`{ channel, to, accountId, threadId }` から安定した重複排除キーを構築したりする必要がある場合は、
`openclaw/plugin-sdk/channel-route` を使用します。このヘルパーは、
コアと同じ方法で数値スレッド ID を正規化するため、場当たり的な
`String(threadId)` 比較よりも優先してください。プロバイダー固有のターゲット文法を持つ
Plugin は `messaging.resolveOutboundSessionRoute(...)` を公開し、パーサーシムを使わずに
プロバイダーネイティブのセッションおよびスレッド ID をコアへ提供する必要があります。

### アカウントスコープの会話バインディング対応

チャンネルが汎用の現在会話バインディングに対応する場合は、
`conversationBindings.supportsCurrentConversationBinding` を設定します。`createChatChannelPlugin(...)` は、
デフォルトでこの静的ケイパビリティを `true` に設定します。

設定済みアカウントによって対応状況が異なる場合は、
`conversationBindings.isCurrentConversationBindingSupported({ accountId })` も実装します。
コアは、静的ケイパビリティが有効になった後に限り、この同期フックを評価します。
`false` を返すと、そのアカウントでは汎用の現在会話ケイパビリティ、
バインド、検索、一覧表示、タッチ、バインド解除操作が利用できなくなります。
このフックを省略すると、静的ケイパビリティがすべてのアカウントに適用されます。

回答は、読み込み済みのアカウント設定またはランタイム状態から解決します。この
フックが制御するのは汎用の現在会話バインディングのみであり、
設定済みのバインディングルールや Plugin 所有のセッションルーティングを置き換えるものではありません。契約テストでは、
`openclaw/plugin-sdk/channel-core` がエクスポートする
`ChannelPlugin["conversationBindings"]` 契約を介して、対応アカウントと非対応アカウントを
少なくとも 1 つずつ網羅する必要があります。

## 承認とチャンネルケイパビリティ

ほとんどのチャンネル Plugin では、承認固有のコードは不要です。コアが同一チャットの
`/approve`、共有承認ボタンのペイロード、汎用フォールバック配信を担当します。
`ChannelPlugin.approvals` は削除されました。代わりに、承認の配信／ネイティブ／レンダリング／認証に関する
ファクトを 1 つの `approvalCapability` オブジェクトに配置してください。`plugin.auth` は
ログイン／ログアウト専用です。コアはこのオブジェクトから承認認証フックを読み取らなくなりました。

`approvalCapability.delivery` はネイティブ承認のルーティングまたはフォールバック抑制にのみ使用し、
`approvalCapability.render` は、共有レンダラーではなくカスタム承認ペイロードが
本当に必要なチャンネルにのみ使用します。

### 承認認証

- `approvalCapability.authorizeActorAction` と
  `approvalCapability.getActionAvailabilityState` が、正規の
  承認認証シームです。
- 同一チャットの承認認証が利用可能かどうかには `getActionAvailabilityState` を使用します。
  ネイティブ配信が無効な場合でも、設定済み承認者を `/approve` で利用可能に
  保ってください。配信／設定ガイダンスには、代わりにネイティブの開始元サーフェス状態を
  使用します。
- チャンネルがネイティブ exec 承認を公開し、その開始元サーフェス／ネイティブクライアントの
  状態が同一チャットの承認認証と異なる場合は、
  `approvalCapability.getExecInitiatingSurfaceState` を使用します。コアはこの exec 固有フックを使用して
  `enabled` と `disabled` を区別し、開始元チャンネルがネイティブ exec
  承認に対応しているかどうかを判断し、そのチャンネルをネイティブクライアントの
  フォールバックガイダンスに含めます。一般的なケースでは
  `createApproverRestrictedNativeApprovalCapability(...)` がこれを補完します。
- チャンネルが既存の設定から安定した所有者相当の DM ID を推論できる場合は、
  `openclaw/plugin-sdk/approval-runtime` の `createResolvedApproverActionAuthAdapter` を使用して、
  承認固有のコアロジックを追加せずに同一チャットの `/approve` を制限します。
- カスタム承認認証で同一チャットのフォールバックのみを意図的に許可する場合は、
  `openclaw/plugin-sdk/approval-auth-runtime` から `markImplicitSameChatApprovalAuthorization({ authorized: true })` を返します。
  それ以外の場合、コアはその結果を明示的な承認者の認可として扱います。
- チャンネル所有のネイティブコールバックが承認を直接解決する場合は、解決前に
  `isImplicitSameChatApprovalAuthorization(...)` を使用し、暗黙の
  フォールバックもチャンネルの通常のアクター認可を通るようにします。

### ペイロードのライフサイクルと設定ガイダンス

- 重複するローカル承認プロンプトを非表示にしたり、配信前に入力中インジケーターを
  送信したりするなど、チャンネル固有のペイロードライフサイクル動作には、
  `outbound.shouldSuppressLocalPayloadPrompt` または
  `outbound.beforeDeliverPayload` を使用します。
- 無効時の応答で、ネイティブ exec 承認を有効にするために必要な正確な設定項目を
  説明する場合は、`approvalCapability.describeExecApprovalSetup` を使用します。
  このフックは `{ channel, channelLabel, accountId }` を受け取ります。
  名前付きアカウントを持つチャンネルでは、トップレベルのデフォルトではなく
  `channels.<channel>.accounts.<id>.execApprovals.*` のようなアカウントスコープのパスを
  レンダリングする必要があります。
- Plugin 承認のルートなしエラーおよびタイムアウトエラーで、Plugin 承認の
  失敗ガイダンスを安全に表示できる場合は、`approvalCapability.describePluginApprovalSetup` を使用します。
  `createApproverRestrictedNativeApprovalCapability(...)` は
  `describeExecApprovalSetup` からこれを推論しません。Plugin 承認と exec 承認が本当に同じ
  ネイティブ設定を使用する場合に限り、同じヘルパーを明示的に渡します。

### ネイティブ承認の配信

チャンネルにネイティブ承認の配信が必要な場合は、チャンネルコードを
ターゲットの正規化とトランスポート／プレゼンテーションのファクトに集中させます。
`openclaw/plugin-sdk/approval-runtime` の
`createChannelExecApprovalProfile`、`createChannelNativeOriginTargetResolver`、
`createChannelApproverDmTargetResolver`、および
`createApproverRestrictedNativeApprovalCapability` を使用します。チャンネル固有のファクトは
`approvalCapability.nativeRuntime` の背後に配置し、理想的には
`createChannelApprovalNativeRuntimeAdapter(...)` または
`createLazyChannelApprovalNativeRuntimeAdapter(...)` を使用します。これにより、コアがハンドラーを組み立て、
リクエストのフィルタリング、ルーティング、重複排除、有効期限、Gateway
サブスクリプション、および別経路へルーティングされたことの通知を担当できます。

`nativeRuntime` は、いくつかの小さなシームに分割されています。

- `availability` - アカウントが設定済みか、およびリクエストを
  処理すべきかどうか
- `presentation` - 共有承認ビューモデルを、
  保留中／解決済み／期限切れのネイティブペイロードまたは最終アクションにマッピング
- `transport` - ターゲットを準備し、ネイティブ承認
  メッセージを送信／更新／削除
- `interactions` - ネイティブボタンまたはリアクション向けの、任意のバインド／バインド解除／アクション消去フックと、
  任意の `cancelDelivered` フック。`deliverPending` がプロセス内または永続的な
  状態（リアクションターゲットストアなど）を登録する場合は、`cancelDelivered` を実装します。
  これにより、ハンドラーの停止によって `bindPending` の実行前に配信がキャンセルされた場合、
  または `bindPending` がハンドルを返さない場合に、その状態を解放できます。
- `observe` - 任意の配信診断フック

その他の承認ヘルパー：

- チャンネルがセッション起点のネイティブ配信と明示的な承認転送ターゲットの両方に
  対応する場合は、`openclaw/plugin-sdk/approval-native-runtime` の
  `createNativeApprovalChannelRouteGates` を使用します。このヘルパーは、
  承認設定の選択、`mode` の処理、エージェント／セッションフィルター、
  アカウントバインディング、セッションターゲットの照合、およびターゲットリストの照合を
  一元化します。一方で呼び出し側は引き続き、チャンネル ID、デフォルト転送モード、
  アカウント検索、トランスポート有効化チェック、ターゲット正規化、およびターンソースの
  ターゲット解決を担当します。コア所有のチャンネルポリシーのデフォルトを作成するために
  使用しないでください。チャンネルで文書化されたデフォルトモードを明示的に渡します。
- `createChannelNativeOriginTargetResolver` は、`{ to, accountId, threadId }` ターゲットに対して、デフォルトで共有チャンネルルート
  マッチャーを使用します。Slack のタイムスタンプ接頭辞マッチングなど、チャンネルに
  プロバイダー固有の同値ルールがある場合に限り、`targetsMatch` を渡します。
  配信用の元のターゲットを維持しながら、デフォルトのルートマッチャーまたはカスタムの
  `targetsMatch` コールバックが実行される前にプロバイダー ID を正規化する必要がある場合は、
  `normalizeTargetForMatch` を渡します。解決された配信ターゲット自体を正規化する必要がある場合に限り、
  `normalizeTarget` を使用します。
- チャンネルでクライアント、トークン、Bolt
  アプリ、Webhook レシーバーなどのランタイム所有オブジェクトが必要な場合は、
  `openclaw/plugin-sdk/channel-runtime-context` を介して登録します。汎用ランタイムコンテキスト
  レジストリにより、承認固有のラッパーグルーを追加せずに、チャンネルの
  起動状態からケイパビリティ駆動のハンドラーをコアがブートストラップできます。
- ケイパビリティ駆動のシームの表現力がまだ十分でない場合に限り、低レベルの
  `createChannelApprovalHandler` または
  `createChannelNativeApprovalRuntime` を使用します。
- ネイティブ承認チャンネルは、`accountId` と `approvalKind` の両方を
  これらのヘルパー経由でルーティングする必要があります。`accountId` は複数アカウントの承認ポリシーを
  適切なボットアカウントに限定し、`approvalKind` はコアにハードコードされた分岐を
  追加せずに、exec 承認と Plugin 承認の動作をチャンネルで利用可能に保ちます。
- 承認の再ルーティング通知もコアが担当します。チャンネル Plugin は、
  `createChannelNativeApprovalRuntime` から独自の「承認は DM／別のチャンネルに送信されました」という
  フォローアップメッセージを送信しないでください。代わりに、共有承認ケイパビリティ
  ヘルパーを介して正確な起点＋承認者 DM ルーティングを公開し、コアが実際の配信を
  集約した後、開始元チャットへの通知を投稿するようにします。
- 配信された承認 ID の種別をエンドツーエンドで維持します。ネイティブクライアントは、
  チャンネルローカルの状態から exec 承認と Plugin 承認のルーティングを推測したり、
  書き換えたりしてはなりません。
- その明示的な `approvalKind` を `resolveApprovalOverGateway` に渡します。これは、
  正規の `approval.resolve` サービスを使用し、別のサーフェスが先に応答した場合は
  記録済みの勝者を返します。古い明示的な `resolveMethod` 入力は、
  コマンドベースのコントロール向けに残されています。新しいネイティブアクションでは、
  これを使用したり、ID から種別を推論したりしてはなりません。
- 承認の種別ごとに、意図的に異なるネイティブサーフェスを公開できます。現在のバンドル例では、
  Matrix は exec 承認と Plugin 承認で同じネイティブ DM／チャンネルルーティングと
  リアクション UX を維持しつつ、承認種別によって認証を変えられます。Slack は
  exec ID と Plugin ID の両方でネイティブ承認ルーティングを利用可能に保ちます。
- `createApproverRestrictedNativeApprovalAdapter` は互換ラッパーとして
  引き続き存在しますが、新しいコードではケイパビリティビルダーを優先し、
  Plugin 上で `approvalCapability` を公開する必要があります。

### より限定的な承認ランタイムサブパス

高頻度で使用されるチャンネルエントリーポイントでは、このファミリーの一部のみが必要な場合、
より広範な `approval-runtime` バレルよりも、次の限定的なサブパスを優先してください。

- `openclaw/plugin-sdk/approval-auth-runtime`
- `openclaw/plugin-sdk/approval-client-runtime`
- `openclaw/plugin-sdk/approval-delivery-runtime`
- `openclaw/plugin-sdk/approval-gateway-runtime`
- `openclaw/plugin-sdk/approval-reference-runtime`
- `openclaw/plugin-sdk/approval-handler-adapter-runtime`
- `openclaw/plugin-sdk/approval-handler-runtime`
- `openclaw/plugin-sdk/approval-native-runtime`
- `openclaw/plugin-sdk/approval-reply-runtime`
- `openclaw/plugin-sdk/channel-runtime-context`

同様に、すべてが必要でない場合は、より広範な包括的サーフェスよりも
`openclaw/plugin-sdk/reply-runtime`、
`openclaw/plugin-sdk/reply-dispatch-runtime`、
`openclaw/plugin-sdk/reply-reference`、および
`openclaw/plugin-sdk/reply-chunking`を優先してください。

### セットアップのサブパス

- `openclaw/plugin-sdk/setup-runtime`は、ランタイムで安全なセットアップヘルパーを対象とします。
  `createSetupTranslator`、インポートしても安全なセットアップパッチアダプター
  （`createPatchedAccountSetupAdapter`、`createEnvPatchedAccountSetupAdapter`、
  `createSetupInputPresenceValidator`）、検索メモの出力、
  `promptResolvedAllowFrom`、`splitSetupEntries`、および委譲された
  セットアッププロキシビルダーです。
- `openclaw/plugin-sdk/channel-setup`は、オプションインストール用セットアップ
  ビルダーと、セットアップで安全ないくつかのプリミティブを対象とします。`createOptionalChannelSetupSurface`、
  `createOptionalChannelSetupAdapter`、`createOptionalChannelSetupWizard`、
  `DEFAULT_ACCOUNT_ID`、`createTopLevelChannelDmPolicy`、
  `setSetupChannelEnabled`、および`splitSetupEntries`です。
- `moveSingleAccountChannelSectionToDefaultAccount(...)`のような、より重量のある共有セットアップ／設定ヘルパーも
  必要な場合にのみ、より広範な`openclaw/plugin-sdk/setup`シームを使用してください。

チャンネルがセットアップサーフェスで「最初にこのPluginをインストールしてください」と
案内するだけの場合は、`createOptionalChannelSetupSurface(...)`を優先してください。生成される
アダプター／ウィザードは、設定の書き込みと完了処理をフェイルクローズし、
検証、完了処理、ドキュメントリンクの文面で同じインストール必須メッセージを
再利用します。

チャンネルが環境変数によるセットアップまたは認証をサポートする場合は、
チャンネル設定スキーマとセットアップ記述子を通じて公開してください。チャンネルランタイムの
`envVars`またはローカル定数は、運用者向けの文面にのみ使用してください。

Pluginランタイムの起動前にチャンネルが`status`、`channels list`、`channels status`、または
SecretRefスキャンに現れる可能性がある場合は、`package.json`に
`openclaw.setupEntry`を追加してください。そのエントリポイントは、読み取り専用コマンドの
パスで安全にインポートでき、それらの概要に必要なチャンネルメタデータ、
セットアップで安全な設定アダプター、ステータスアダプター、およびチャンネルのシークレット対象メタデータを
返す必要があります。セットアップエントリからクライアント、リスナー、トランスポートランタイムを
起動しないでください。

メインチャンネルエントリのインポートパスも狭く保ってください。ディスカバリーは、
チャンネルを有効化せずにエントリとチャンネルPluginモジュールを評価し、
機能を登録できます。`channel-plugin-api.ts`のようなファイルは、
セットアップウィザード、トランスポートクライアント、ソケットリスナー、
サブプロセスランチャー、サービス起動モジュールをインポートせずに、
チャンネルPluginオブジェクトをエクスポートする必要があります。
これらのランタイム要素は、`registerFull(...)`から読み込まれるモジュール、
ランタイムセッター、または遅延機能アダプターに配置してください。

### その他の狭いチャンネルサブパス

その他の高頻度なチャンネルパスでは、より広範なレガシーサーフェスよりも
狭いヘルパーを優先してください。

- 複数アカウントの設定とデフォルトアカウントへの
  フォールバックには、`openclaw/plugin-sdk/account-core`、`openclaw/plugin-sdk/account-id`、
  `openclaw/plugin-sdk/account-resolution`、および
  `openclaw/plugin-sdk/account-helpers`
- 受信ルート／エンベロープ、および記録してディスパッチする
  結線には、`openclaw/plugin-sdk/inbound-envelope`と
  `openclaw/plugin-sdk/channel-inbound`
- 対象解析ヘルパーには、`openclaw/plugin-sdk/channel-targets`
- 送信ID／送信デリゲートと型付きペイロード計画には、
  `openclaw/plugin-sdk/channel-outbound`
- 送信ルートが明示的な`replyToId`/`threadId`を保持する必要がある場合、または
  ベースセッションキーが引き続き一致した後に現在の`:thread:`セッションを復元する必要がある場合は、
  `openclaw/plugin-sdk/channel-core`の`buildThreadAwareOutboundSessionRoute(...)`。
  プラットフォームにネイティブのスレッド配信セマンティクスがある場合、
  プロバイダーPluginは優先順位、サフィックスの動作、スレッドIDの正規化を
  オーバーライドできます。
- スレッドバインディングのライフサイクルと
  アダプター登録には、`openclaw/plugin-sdk/thread-bindings-runtime`

認証のみのチャンネルは、通常デフォルトパスで十分です。コアが
承認を処理し、Pluginは送信／認証機能を公開するだけです。Matrix、Slack、
Telegram、カスタムチャットトランスポートなど、ネイティブ承認を備えたチャンネルは、
独自の承認ライフサイクルを実装する代わりに、共有ネイティブヘルパーを
使用してください。

## 受信メンションポリシー

受信メンション処理は、次の2層に分けてください。

- Plugin所有の根拠収集
- 共有ポリシー評価

メンションポリシーの判断には`openclaw/plugin-sdk/channel-mention-gating`を使用してください。
より広範な受信ヘルパーバレルが必要な場合にのみ、
`openclaw/plugin-sdk/channel-inbound`を使用してください。

Pluginローカルのロジックに適しているもの：

- Botへの返信の検出
- 引用されたBotの検出
- スレッド参加の確認
- サービス／システムメッセージの除外
- Botの参加を立証するために必要なプラットフォームネイティブのキャッシュ

共有ヘルパーに適しているもの：

- `requireMention`
- 明示的メンションの結果
- 暗黙的メンションの許可リスト
- コマンドのバイパス
- 最終的なスキップ判定

推奨フロー：

1. ローカルのメンション情報を計算します。
2. その情報を`resolveInboundMentionDecision({ facts, policy })`に渡します。
3. 受信ゲートで`decision.effectiveWasMentioned`、`decision.shouldBypassMention`、および
   `decision.shouldSkip`を使用します。

```typescript
import {
  implicitMentionKindWhen,
  matchesMentionWithExplicit,
  resolveInboundMentionDecision,
} from "openclaw/plugin-sdk/channel-inbound";
import { resolveChannelImplicitMentions } from "openclaw/plugin-sdk/channel-ingress-runtime";

const wasMentioned = matchesMentionWithExplicit({
  text,
  mentionRegexes,
  explicit: {
    hasAnyMention,
    isExplicitlyMentioned,
    canResolveExplicit,
  },
});

const facts = {
  canDetectMention: true,
  wasMentioned,
  hasAnyMention,
  implicitMentionKinds: [
    ...implicitMentionKindWhen("reply_to_bot", isReplyToBot),
    ...implicitMentionKindWhen("quoted_bot", isQuoteOfBot),
  ],
};

const implicitMentions = resolveChannelImplicitMentions({
  cfg,
  channel: channelId,
  accountId,
});

const decision = resolveInboundMentionDecision({
  facts,
  policy: {
    isGroup,
    requireMention,
    implicitMentions,
    allowTextCommands,
    hasControlCommand,
    commandAuthorized,
  },
});

if (decision.shouldSkip) return;
```

`matchesMentionWithExplicit(...)`は真偽値を返します。`hasAnyMention`、
`isExplicitlyMentioned`、および`canResolveExplicit`は、チャンネル独自の
ネイティブメンションメタデータ（メッセージエンティティ、Botへの返信フラグなど）から取得します。
プラットフォームでそれらを検出できない場合は、`false`/`undefined`の値を
指定してください。

`api.runtime.channel.mentions`は、ランタイム注入にすでに依存している
同梱チャンネルPlugin向けに、同じ共有メンションヘルパーを公開します。
`buildMentionRegexes`、`matchesMentionPatterns`、`matchesMentionWithExplicit`、
`implicitMentionKindWhen`、`resolveInboundMentionDecision`です。

`implicitMentionKindWhen`と`resolveInboundMentionDecision`だけが必要な場合は、
無関係な受信ランタイムヘルパーの読み込みを避けるため、
`openclaw/plugin-sdk/channel-mention-gating`からインポートしてください。

## 手順

<Steps>
  <a id="step-1-package-and-manifest"></a>
  <Step title="パッケージとマニフェスト">
    標準のPluginファイルを作成します。`openclaw.plugin.json`の
    `channels`フィールド（`kind`フィールドではありません）が、
    マニフェストがチャンネルを所有していることを示します。パッケージメタデータの
    全サーフェスについては、[Pluginのセットアップと設定](/ja-JP/plugins/sdk-setup#openclaw-channel)を参照してください。

    <CodeGroup>
    ```json package.json
    {
      "name": "@myorg/openclaw-acme-chat",
      "version": "1.0.0",
      "type": "module",
      "openclaw": {
        "extensions": ["./index.ts"],
        "setupEntry": "./setup-entry.ts",
        "channel": {
          "id": "acme-chat",
          "label": "Acme Chat",
          "blurb": "OpenClawをAcme Chatに接続します。"
        }
      }
    }
    ```

    ```json openclaw.plugin.json
    {
      "id": "acme-chat",
      "channels": ["acme-chat"],
      "name": "Acme Chat",
      "description": "Acme ChatチャンネルPlugin",
      "configSchema": {
        "type": "object",
        "additionalProperties": false,
        "properties": {}
      },
      "channelConfigs": {
        "acme-chat": {
          "schema": {
            "type": "object",
            "additionalProperties": false,
            "properties": {
              "token": { "type": "string" },
              "allowFrom": {
                "type": "array",
                "items": { "type": "string" }
              }
            }
          },
          "uiHints": {
            "token": {
              "label": "Botトークン",
              "sensitive": true
            }
          }
        }
      }
    }
    ```
    </CodeGroup>

    `configSchema`は`plugins.entries.acme-chat.config`を検証します。チャンネルアカウント設定ではない、
    Plugin所有の設定に使用してください。
    `channelConfigs.acme-chat.schema`は`channels.acme-chat`を検証し、
    Pluginランタイムが読み込まれる前に、設定スキーマ、セットアップ、UIサーフェスで
    使用されるコールドパスのソースです。トップレベルフィールドの完全なリファレンスについては、
    [Pluginマニフェスト](/ja-JP/plugins/manifest)を参照してください。

  </Step>

  <Step title="チャンネルPluginオブジェクトを構築する">
    `ChannelPlugin`インターフェースには、多数のオプションのアダプターサーフェスがあります。
    まず最小構成の`id`、`config`、`setup`から始め、
    必要に応じてアダプターを追加してください。

    `src/channel.ts`を作成します。

    ```typescript src/channel.ts
    import {
      createChatChannelPlugin,
      createChannelPluginBase,
    } from "openclaw/plugin-sdk/channel-core";
    import type { OpenClawConfig } from "openclaw/plugin-sdk/channel-core";
    import { acmeChatApi } from "./client.js"; // your platform API client

    type ResolvedAccount = {
      accountId: string | null;
      token: string;
      allowFrom: string[];
      dmPolicy: string | undefined;
    };

    function resolveAccount(
      cfg: OpenClawConfig,
      accountId?: string | null,
    ): ResolvedAccount {
      const section = (cfg.channels as Record<string, any>)?.["acme-chat"];
      const token = section?.token;
      if (!token) throw new Error("acme-chat: token is required");
      return {
        accountId: accountId ?? null,
        token,
        allowFrom: section?.allowFrom ?? [],
        dmPolicy: section?.dmSecurity,
      };
    }

    export const acmeChatPlugin = createChatChannelPlugin<ResolvedAccount>({
      base: createChannelPluginBase({
        id: "acme-chat",
        // Account resolution/inspection belongs on `config`, not `setup`.
        // `setup` covers onboarding writes (applyAccountConfig, validateInput).
        config: {
          listAccountIds: () => ["default"],
          resolveAccount,
          inspectAccount(cfg, accountId) {
            const section =
              (cfg.channels as Record<string, any>)?.["acme-chat"];
            return {
              enabled: Boolean(section?.token),
              configured: Boolean(section?.token),
              tokenStatus: section?.token ? "available" : "missing",
            };
          },
        },
        setup: {
          applyAccountConfig: ({ cfg, input }) => ({
            ...cfg,
            channels: {
              ...cfg.channels,
              "acme-chat": { ...(cfg.channels as any)?.["acme-chat"], ...input },
            },
          }),
        },
      }),

      // DM security: who can message the bot
      security: {
        dm: {
          channelKey: "acme-chat",
          resolvePolicy: (account) => account.dmPolicy,
          resolveAllowFrom: (account) => account.allowFrom,
          defaultPolicy: "allowlist",
        },
      },

      // Pairing: approval flow for new DM contacts
      pairing: {
        text: {
          idLabel: "Acme Chat username",
          message: "Send this code to verify your identity:",
          notify: async ({ target, code }) => {
            await acmeChatApi.sendDm(target, `Pairing code: ${code}`);
          },
        },
      },

      // Threading: how replies are delivered
      threading: { topLevelReplyToMode: "reply" },

      // Outbound: send messages to the platform
      outbound: {
        attachedResults: {
          channel: "acme-chat",
          sendText: async (params) => {
            const result = await acmeChatApi.sendMessage(
              params.to,
              params.text,
            );
            return { messageId: result.id };
          },
        },
        base: {
          sendMedia: async (params) => {
            await acmeChatApi.sendFile(params.to, params.filePath);
          },
        },
      },
    });
    ```

    正規のトップレベル DM キーと従来のネストされたキーの両方を受け付けるチャネルでは、`plugin-sdk/channel-config-helpers` のヘルパーを使用します。`resolveChannelDmAccess`、`resolveChannelDmPolicy`、`resolveChannelDmAllowFrom`、および `normalizeChannelDmPolicy` は、継承されたルート値よりもアカウントローカルの値を優先します。ランタイムと移行が同じコントラクトを読み取るように、`normalizeLegacyDmAliases` を介した doctor 修復にも同じリゾルバーを組み合わせてください。

    <Accordion title="createChatChannelPlugin が行うこと">
      低レベルのアダプターインターフェースを手動で実装する代わりに、
      宣言的なオプションを渡すと、ビルダーがそれらを構成します。

      | オプション | 接続される機能 |
      | --- | --- |
      | `security.dm` | 設定フィールドからスコープ付き DM セキュリティリゾルバーを構成 |
      | `pairing.text` | コード交換を使用するテキストベースの DM ペアリングフロー |
      | `threading` | 返信先モードのリゾルバー（固定、アカウントスコープ、またはカスタム） |
      | `outbound.attachedResults` | 結果メタデータ（メッセージ ID）を返す送信関数。コアが返された配信結果にチャネルを記録できるように、同階層の `channel` ID が必要 |

      完全な制御が必要な場合は、宣言的なオプションの代わりに
      生のアダプターオブジェクトを渡すこともできます。

      生のアウトバウンドアダプターでは、`chunker(text, limit, ctx)` 関数を定義できます。
      オプションの `ctx.formatting` は、
      `maxLinesPerMessage` など、配信時のフォーマットに関する決定を保持します。送信前に適用することで、
      返信スレッドとチャンク境界が共有アウトバウンド配信によって一度だけ解決されます。
      ネイティブの返信先が解決された場合、送信コンテキストには
      `replyToIdSource`（`implicit` または `explicit`）も含まれるため、
      ペイロードヘルパーは暗黙的な一回限りの返信スロットを消費せずに、
      明示的な返信タグを維持できます。
    </Accordion>

    ### グループツールポリシーアダプター

    `group.resolveToolPolicy` を実装し、
    `toolsBySender` をサポートするチャネルは、完全な `ChannelGroupContext` を
    共有ポリシーリゾルバーに転送する必要があります。特に、基本の
    `tools` ポリシーは引き続き適用しながら、マッチしたグループとワイルドカードの
    両方のスコープで送信者固有のオーバーレイをスキップすることにより、
    `senderPolicyMode: "never"` を尊重してください。

    OpenClaw がこのモードを設定するのは、明示的に上限が設定されたスケジュール実行など、
    送信者の権限がサーバー所有のエンベロープにすでに記録されている、
    信頼された非イングレス実行の場合のみです。Plugin は、受信メタデータから
    このモードを導出したり、チャネル状態として永続化したり、設定として公開したりしてはなりません。
    一致する基本の `tools` 制限を削除せずに、ワイルドカードの
    `toolsBySender` エントリがこのモードによってスキップされることを証明する
    アダプターテストを追加してください。

  </Step>

  <Step title="エントリポイントを接続する">
    `index.ts` を作成します。

    ```typescript index.ts
    import { defineChannelPluginEntry } from "openclaw/plugin-sdk/channel-core";
    import { acmeChatPlugin } from "./src/channel.js";

    export default defineChannelPluginEntry({
      id: "acme-chat",
      name: "Acme Chat",
      description: "Acme Chat channel plugin",
      plugin: acmeChatPlugin,
      registerCliMetadata(api) {
        api.registerCli(
          ({ program }) => {
            program
              .command("acme-chat")
              .description("Acme Chat management");
          },
          {
            descriptors: [
              {
                name: "acme-chat",
                description: "Acme Chat management",
                hasSubcommands: false,
              },
            ],
          },
        );
      },
      registerFull(api) {
        api.registerGatewayMethod(/* ... */);
      },
    });
    ```

    チャネル所有の CLI ディスクリプターは `registerCliMetadata(...)` に配置してください。これにより OpenClaw は、
    完全なチャネルランタイムを有効化せずにルートヘルプへ表示できる一方、
    通常の完全ロードでも実際のコマンド登録に同じディスクリプターを使用できます。
    `registerFull(...)` はランタイム専用処理に使用してください。
    `defineChannelPluginEntry` は登録モードの分岐を自動的に処理します。
    `registerFull(...)` で Gateway RPC メソッドを登録する場合は、
    Plugin 固有のプレフィックスを使用してください。コア管理名前空間（`config.*`、
    `exec.approvals.*`、`wizard.*`、`update.*`）は予約されたままであり、常に
    `operator.admin` に解決されます。すべてのオプションについては、
    [エントリポイント](/ja-JP/plugins/sdk-entrypoints#definechannelpluginentry)を参照してください。

  </Step>

  <Step title="セットアップエントリを追加する">
    オンボーディング中の軽量ロード用に `setup-entry.ts` を作成します。

    ```typescript setup-entry.ts
    import { defineSetupPluginEntry } from "openclaw/plugin-sdk/channel-core";
    import { acmeChatPlugin } from "./src/channel.js";

    export default defineSetupPluginEntry(acmeChatPlugin);
    ```

    チャネルが無効または未設定の場合、OpenClaw は完全なエントリの代わりにこれを読み込みます。
    これにより、セットアップフロー中に重いランタイムコードが読み込まれるのを回避できます。
    詳細については、[セットアップと設定](/ja-JP/plugins/sdk-setup#setup-entry)を参照してください。

    セットアップに安全なエクスポートをサイドカーモジュールに分割する
    バンドル済みワークスペースチャネルでは、セットアップ時の明示的な
    ランタイムセッターも必要な場合、`openclaw/plugin-sdk/channel-entry-contract` の
    `defineBundledChannelSetupEntry(...)` を使用できます。

  </Step>

  <Step title="受信メッセージを処理する">
    Plugin はプラットフォームからメッセージを受信し、OpenClaw に転送する必要があります。
    一般的なパターンは、リクエストを検証し、チャネルの受信ハンドラーを介して
    ディスパッチする Webhook です。

    ```typescript
    registerFull(api) {
      api.registerHttpRoute({
        path: "/acme-chat/webhook",
        auth: "plugin", // Plugin が管理する認証（署名は自身で検証）
        handler: async (req, res) => {
          const event = parseWebhookPayload(req);

          // 受信ハンドラーがメッセージを OpenClaw にディスパッチします。
          // 正確な接続方法はプラットフォーム SDK によって異なります。
          // バンドル済みの Microsoft Teams または Google Chat Plugin パッケージにある実例を参照してください。
          await handleAcmeChatInbound(api, event);

          res.statusCode = 200;
          res.end("ok");
          return true;
        },
      });
    }
    ```

    <Note>
      受信メッセージの処理はチャネル固有です。各チャネル Plugin が
      独自の受信パイプラインを所有します。実際のパターンについては、
      バンドル済みチャネル Plugin（Microsoft Teams または Google Chat の
      Plugin パッケージなど）を参照してください。
    </Note>

  </Step>

<a id="step-6-test"></a>
<Step title="テスト">
`src/channel.test.ts` に同じ場所へ配置するテストを作成します。

    ```typescript src/channel.test.ts
    import { describe, it, expect } from "vitest";
    import { acmeChatPlugin } from "./channel.js";

    describe("acme-chat Plugin", () => {
      it("設定からアカウントを解決する", () => {
        const cfg = {
          channels: {
            "acme-chat": { token: "test-token", allowFrom: ["user1"] },
          },
        } as any;
        const account = acmeChatPlugin.config.resolveAccount(cfg, undefined);
        expect(account.token).toBe("test-token");
      });

      it("シークレットを実体化せずにアカウントを検査する", () => {
        const cfg = {
          channels: { "acme-chat": { token: "test-token" } },
        } as any;
        const result = acmeChatPlugin.config.inspectAccount!(cfg, undefined);
        expect(result.configured).toBe(true);
        expect(result.tokenStatus).toBe("available");
      });

      it("設定がないことを報告する", () => {
        const cfg = { channels: {} } as any;
        const result = acmeChatPlugin.config.inspectAccount!(cfg, undefined);
        expect(result.configured).toBe(false);
      });
    });
    ```

    ```bash
    pnpm test <bundled-plugin-root>/acme-chat/
    ```

    共有テストヘルパーについては、[テスト](/ja-JP/plugins/sdk-testing)を参照してください。

</Step>
</Steps>

## ファイル構成

```text
<bundled-plugin-root>/acme-chat/
├── package.json              # openclaw.channel メタデータ
├── openclaw.plugin.json      # 設定スキーマを含むマニフェスト
├── index.ts                  # defineChannelPluginEntry
├── setup-entry.ts            # defineSetupPluginEntry
├── api.ts                    # 公開エクスポート（任意）
├── runtime-api.ts            # 内部ランタイムエクスポート（任意）
└── src/
    ├── channel.ts            # createChatChannelPlugin による ChannelPlugin
    ├── channel.test.ts       # テスト
    ├── client.ts             # プラットフォーム API クライアント
    └── runtime.ts            # ランタイムストア（必要な場合）
```

## 高度なトピック

<CardGroup cols={2}>
  <Card title="スレッド化オプション" icon="git-branch" href="/ja-JP/plugins/sdk-entrypoints#registration-mode">
    固定、アカウントスコープ、またはカスタムの返信モード
  </Card>
  <Card title="メッセージツールの統合" icon="puzzle" href="/ja-JP/plugins/architecture#channel-plugins-and-the-shared-message-tool">
    describeMessageTool とアクション検出
  </Card>
  <Card title="ターゲット解決" icon="crosshair" href="/ja-JP/plugins/architecture-internals#channel-target-resolution">
    inferTargetChatType、looksLikeId、reservedLiterals、resolveTarget
  </Card>
  <Card title="ランタイムヘルパー" icon="settings" href="/ja-JP/plugins/sdk-runtime">
    api.runtime 経由の TTS、STT、メディア、サブエージェント
  </Card>
  <Card title="チャネル受信 API" icon="bolt" href="/ja-JP/plugins/sdk-channel-inbound">
    共有受信イベントのライフサイクル：取り込み、解決、記録、ディスパッチ、完了処理
  </Card>
</CardGroup>

<Note>
バンドル済み Plugin の保守と互換性のために、バンドル済みヘルパーの接続面が一部残されています。
これらは新しいチャネル Plugin に推奨されるパターンではありません。
そのバンドル済み Plugin ファミリーを直接保守している場合を除き、共通 SDK
サーフェスの汎用チャネル、セットアップ、返信、ランタイムの各サブパスを使用してください。
</Note>

## 次のステップ

- [プロバイダー Plugin](/ja-JP/plugins/sdk-provider-plugins) - Plugin でモデルも提供する場合
- [SDK の概要](/ja-JP/plugins/sdk-overview) - サブパスインポートの完全なリファレンス
- [SDK のテスト](/ja-JP/plugins/sdk-testing) - テストユーティリティとコントラクトテスト
- [Plugin マニフェスト](/ja-JP/plugins/manifest) - 完全なマニフェストスキーマ

## 関連項目

- [Plugin SDK のセットアップ](/ja-JP/plugins/sdk-setup)
- [Plugin の構築](/ja-JP/plugins/building-plugins)
- [エージェントハーネス Plugin](/ja-JP/plugins/sdk-agent-harness)
