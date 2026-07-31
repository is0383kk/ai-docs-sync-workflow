---
read_when:
    - ネイティブ OpenClaw Plugin のビルドまたはデバッグ
    - Plugin の機能モデルまたは所有権の境界を理解する
    - Plugin の読み込みパイプラインまたはレジストリに取り組む
    - プロバイダーのランタイムフックまたはチャネル Plugin の実装
sidebarTitle: Internals
summary: Plugin の内部構造：ケイパビリティモデル、所有権、契約、読み込みパイプライン、ランタイムヘルパー
title: Plugin の内部構造
x-i18n:
    generated_at: "2026-07-26T09:09:32Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: d47551b1bc2f71ce2ade3dfdd14bff8ee187616c3807f8101c1a3236e1443cc1
    source_path: plugins/architecture.md
    workflow: 16
---

これは OpenClaw Plugin システムの**詳細なアーキテクチャリファレンス**です。実践的なガイドについては、以下の目的別ページのいずれかから始めてください。

<CardGroup cols={2}>
  <Card title="Plugin のインストールと使用" icon="plug" href="/ja-JP/tools/plugin">
    Plugin の追加、有効化、トラブルシューティングを行うエンドユーザー向けガイドです。
  </Card>
  <Card title="Plugin の構築" icon="rocket" href="/ja-JP/plugins/building-plugins">
    最小限の動作するマニフェストを使用した、最初の Plugin のチュートリアルです。
  </Card>
  <Card title="チャネル Plugin" icon="comments" href="/ja-JP/plugins/sdk-channel-plugins">
    メッセージングチャネル Plugin を構築します。
  </Card>
  <Card title="プロバイダー Plugin" icon="microchip" href="/ja-JP/plugins/sdk-provider-plugins">
    モデルプロバイダー Plugin を構築します。
  </Card>
  <Card title="SDK の概要" icon="book" href="/ja-JP/plugins/sdk-overview">
    インポートマップと登録 API のリファレンスです。
  </Card>
</CardGroup>

## 公開ケイパビリティモデル

ケイパビリティは、OpenClaw 内の公開**ネイティブ Plugin**モデルです。すべてのネイティブ OpenClaw Plugin は、1 つ以上のケイパビリティタイプに対して登録されます。

| ケイパビリティ             | 登録メソッド                              | Plugin の例                                             |
| ---------------------- | ------------------------------------------------ | ----------------------------------------------------------- |
| テキスト推論         | `api.registerProvider(...)`                      | `anthropic`, `openai`                                       |
| CLI 推論バックエンド  | `api.registerCliBackend(...)`                    | `anthropic`, `openai`                                       |
| 埋め込み             | `api.registerEmbeddingProvider(...)`             | プロバイダー所有のベクトル Plugin                               |
| 音声                 | `api.registerSpeechProvider(...)`                | `elevenlabs`, `microsoft`                                   |
| リアルタイム文字起こし | `api.registerRealtimeTranscriptionProvider(...)` | `openai`                                                    |
| リアルタイム音声         | `api.registerRealtimeVoiceProvider(...)`         | `google`, `openai`                                          |
| メディア理解    | `api.registerMediaUnderstandingProvider(...)`    | `google`, `openai`                                          |
| トランスクリプトソース     | `api.registerTranscriptSourceProvider(...)`      | `discord`, `google-meet`, `teams-meetings`, `zoom-meetings` |
| 画像生成       | `api.registerImageGenerationProvider(...)`       | `fal`, `google`, `openai`                                   |
| 音楽生成       | `api.registerMusicGenerationProvider(...)`       | `fal`, `google`, `minimax`                                  |
| 動画生成       | `api.registerVideoGenerationProvider(...)`       | `fal`, `google`, `qwen`                                     |
| Web 取得              | `api.registerWebFetchProvider(...)`              | `firecrawl`                                                 |
| Web 検索             | `api.registerWebSearchProvider(...)`             | `brave`, `firecrawl`, `google`                              |
| チャネル / メッセージング    | `api.registerChannel(...)`                       | `matrix`, `msteams`                                         |
| Gateway 検出      | `api.registerGatewayDiscoveryService(...)`       | `bonjour`                                                   |

<Note>
ケイパビリティを 1 つも登録せず、フック、ツール、検出サービス、またはバックグラウンドサービスを提供する Plugin は、**従来のフック専用** Plugin です。このパターンは現在も完全にサポートされています。
</Note>

### 外部互換性に対する方針

ケイパビリティモデルはコアに導入され、現在バンドル済み / ネイティブ Plugin で使用されていますが、外部 Plugin の互換性には「エクスポートされているため固定されている」よりも厳格な基準が必要です。

| Plugin の状況                                  | ガイダンス                                                                                         |
| ------------------------------------------------- | ------------------------------------------------------------------------------------------------ |
| 既存の外部 Plugin                         | フックベースの統合が引き続き動作するようにします。これが互換性の基準です。                        |
| 新しいバンドル済み / ネイティブ Plugin                        | ベンダー固有の内部アクセスや新しいフック専用設計より、明示的なケイパビリティ登録を優先します。 |
| ケイパビリティ登録を採用する外部 Plugin | 許可されていますが、ドキュメントで安定版と明記されていない限り、ケイパビリティ固有のヘルパーサーフェスは発展途上として扱ってください。 |

ケイパビリティ登録が今後の方向性です。移行期間中、従来のフックは外部 Plugin にとって最も安全で破壊的変更のない方法です。エクスポートされたヘルパーのサブパスはすべて同等ではありません。付随的なヘルパーエクスポートよりも、範囲が狭く文書化された契約を優先してください。

### Plugin の形態

OpenClaw は、読み込まれた各 Plugin を、静的メタデータだけでなく実際の登録動作に基づいて形態に分類します。

<AccordionGroup>
  <Accordion title="plain-capability">
    ケイパビリティタイプを 1 つだけ登録します（たとえば、`arcee` や `chutes` のようなプロバイダー専用 Plugin）。
  </Accordion>
  <Accordion title="hybrid-capability">
    複数のケイパビリティタイプを登録します（たとえば `openai` は、テキスト推論、音声、メディア理解、画像生成を所有します）。
  </Accordion>
  <Accordion title="hook-only">
    フック（型付きまたはカスタム）のみを登録し、ケイパビリティ、ツール、コマンド、サービスは登録しません。
  </Accordion>
  <Accordion title="non-capability">
    ツール、コマンド、サービス、ルートを登録しますが、ケイパビリティは登録しません。
  </Accordion>
</AccordionGroup>

Plugin の形態とケイパビリティの内訳を確認するには、`openclaw plugins inspect <id>` を使用します。詳細については、[CLI リファレンス](/ja-JP/cli/plugins#inspect)を参照してください。

### 互換性シグナル

`openclaw doctor`、`openclaw plugins inspect <id>`、`openclaw status --all`、`openclaw plugins doctor` は、次の互換性通知を表示します。

| シグナル                                     | 意味                                                                                                       |
| ------------------------------------------ | ------------------------------------------------------------------------------------------------------------- |
| **設定が有効**                           | 設定の解析に成功し、Plugin が解決されます                                                                        |
| **フック専用**（情報）                       | Plugin はフックのみを登録します。サポートされている方法ですが、まだケイパビリティ登録へ移行されていません                |
| **非推奨のメモリ埋め込み API**（警告） | バンドルされていない Plugin が、`registerEmbeddingProvider` ではなく古いメモリ固有の埋め込みプロバイダー API を使用しています |
| **重大なエラー**                             | 設定が無効であるか、Plugin の読み込みに失敗しました                                                                    |

現時点では、助言 / 警告シグナルによって Plugin が動作しなくなることはありません。これらのシグナルは `openclaw status --all` と `openclaw plugins doctor` にも表示されます。

## アーキテクチャの概要

OpenClaw の Plugin システムには 4 つのレイヤーがあります。

<Steps>
  <Step title="マニフェスト + 検出">
    OpenClaw は、設定されたパス、ワークスペースルート、グローバル Plugin ルート、バンドル済み Plugin から候補 Plugin を検出します。検出では、最初にネイティブの `openclaw.plugin.json` マニフェストと、サポートされているバンドルマニフェストを読み取ります。
  </Step>
  <Step title="有効化 + 検証">
    コアは、検出された Plugin を有効化、無効化、ブロックするか、またはメモリなどの排他的スロットに選択するかを決定します。
  </Step>
  <Step title="ランタイム読み込み">
    ネイティブ OpenClaw Plugin はプロセス内で読み込まれ、ケイパビリティを中央レジストリに登録します。パッケージ化された JavaScript はネイティブの `require` を通じて読み込まれます。サードパーティのローカルソース TypeScript には、緊急時の Jiti フォールバックが使用されます。互換性のあるバンドルは、ランタイムコードをインポートせずにレジストリレコードへ正規化されます。
  </Step>
  <Step title="サーフェスによる利用">
    OpenClaw のその他の部分はレジストリを読み取り、ツール、チャネル、プロバイダー設定、フック、HTTP ルート、CLI コマンド、サービスを公開します。
  </Step>
</Steps>

Plugin CLI に限っては、ルートコマンドの検出は 2 つのフェーズに分かれます。

- 解析時のメタデータは `registerCli(..., { descriptors: [...] })` から取得される
- 実際の Plugin CLI モジュールは遅延読み込みのままにし、最初の呼び出し時に登録できる

これにより、Plugin が所有する CLI コードを Plugin 内に保持しながら、OpenClaw が解析前にルートコマンド名を予約できます。

重要な設計境界は次のとおりです。

- マニフェスト / 設定の検証は、Plugin コードを実行せずに**マニフェスト / スキーマメタデータ**から機能する必要がある
- ネイティブケイパビリティの検出では、信頼された Plugin エントリコードを読み込み、非アクティブ化レジストリのスナップショットを構築できる
- ネイティブランタイムの動作は、`api.registrationMode === "full"` を使用する Plugin モジュールの `register(api)` パスから提供される

この分離により、完全なランタイムがアクティブになる前に、OpenClaw は設定を検証し、欠落または無効化された Plugin を説明し、UI / スキーマのヒントを構築できます。

### Plugin メタデータのスナップショットとルックアップテーブル

Gateway の起動時に、現在の設定スナップショット用の `PluginMetadataSnapshot` が 1 つ構築されます。このスナップショットにはメタデータのみが含まれます。インストール済み Plugin のインデックス、マニフェストレジストリ、マニフェスト診断、所有者マップ、Plugin ID ノーマライザー、マニフェストレコードが保存されます。読み込まれた Plugin モジュール、プロバイダー SDK、パッケージ内容、ランタイムエクスポートは保持しません。

Plugin 対応の設定検証、起動時の自動有効化、Gateway Plugin のブートストラップは、マニフェスト / インデックスのメタデータを個別に再構築する代わりに、このスナップショットを利用します。`PluginLookUpTable` は同じスナップショットから派生し、現在のランタイム設定に対応する起動時の Plugin プランを追加します。

起動後、Gateway は現在のメタデータスナップショットを置換可能なランタイム成果物として保持します。ランタイムで繰り返されるプロバイダー検出では、プロバイダーカタログを処理するたびにインストール済みインデックスとマニフェストレジストリを再構築する代わりに、このスナップショットを借用できます。Gateway のシャットダウン時、設定 / Plugin インベントリの変更時、インストール済みインデックスへの書き込み時に、スナップショットはクリアまたは置換されます。互換性のある最新スナップショットが存在しない場合、呼び出し元はコールドなマニフェスト / インデックスパスへフォールバックします。ワークスペース Plugin はメタデータスコープの一部であるため、互換性チェックには `plugins.load.paths` やデフォルトのエージェントワークスペースなどの Plugin 検出ルートを含める必要があります。

スナップショットとルックアップテーブルにより、起動時に繰り返される判断が高速パスに維持されます。

- チャネルの所有権
- 遅延チャネル起動
- 起動時の Plugin ID
- プロバイダーと CLI バックエンドの所有権
- セットアッププロバイダー、コマンドエイリアス、モデルカタログプロバイダー、マニフェスト契約の所有権
- Plugin 設定スキーマとチャネル設定スキーマの検証
- 起動時の自動有効化の判断

安全性の境界は、スナップショットの変更ではなく置換です。設定、Plugin インベントリ、インストールレコード、永続化されたインデックスポリシーが変更された場合は、スナップショットを再構築してください。これを広範で変更可能なグローバルレジストリとして扱ったり、履歴スナップショットを無制限に保持したりしないでください。ランタイム Plugin の読み込みはメタデータスナップショットから分離されたままなので、古いランタイム状態がメタデータキャッシュの背後に隠されることはありません。

キャッシュ規則については、[Plugin アーキテクチャの内部構造](/ja-JP/plugins/architecture-internals#plugin-cache-boundary)で説明されています。呼び出し元が現在のフローに対する明示的なスナップショット、ルックアップテーブル、またはマニフェストレジストリを保持していない限り、マニフェストと検出のメタデータは最新です。非公開のメタデータキャッシュと実時間ベースの TTL は、Plugin 読み込みの一部ではありません。コードまたはインストール済み成果物が実際に読み込まれた後も保持できるのは、ランタイムローダー、モジュール、依存関係成果物のキャッシュのみです。

一部のコールドパス呼び出し元では、Gateway `PluginLookUpTable` を受け取る代わりに、永続化されたインストール済み Plugin インデックスからマニフェストレジストリを直接再構築しています。このパスでは、必要に応じてレジストリを再構築するようになりました。呼び出し元が現在のルックアップテーブルまたは明示的なマニフェストレジストリをすでに保持している場合は、それをランタイムフローに渡すことを推奨します。

### アクティベーション計画

アクティベーション計画はコントロールプレーンの一部です。呼び出し元は、より広範なランタイムレジストリを読み込む前に、具体的なコマンド、プロバイダー、チャネル、ルート、エージェントハーネス、またはケイパビリティに関連する Plugin を問い合わせることができます。

プランナーは、現在のマニフェスト動作との互換性を維持します。

- `activation.*` フィールドは明示的なプランナーヒント
- `providers`、`channels`、`commandAliases`、`setup.providers`、`contracts.tools`、およびフックは引き続きマニフェスト所有権のフォールバック
- ID のみを返すプランナー API は既存の呼び出し元向けに引き続き利用可能
- 計画 API は理由ラベルを報告するため、診断で明示的なヒントと所有権フォールバックを区別可能

<Warning>
`activation` をライフサイクルフックや `register(...)` の代替として扱わないでください。これは読み込み範囲を絞り込むために使用されるメタデータです。所有権フィールドですでに関係が記述されている場合はそちらを優先し、`activation` は追加のプランナーヒントにのみ使用してください。
</Warning>

### チャネル Plugin と共有メッセージツール

チャネル Plugin は、通常のチャットアクション用に個別の送信、編集、リアクションツールを登録する必要はありません。OpenClaw はコアに単一の共有 `message` ツールを保持し、その背後にあるチャネル固有の検出と実行はチャネル Plugin が所有します。

現在の境界は次のとおりです。

- コアは共有 `message` ツールホスト、プロンプト配線、セッションおよびスレッドの記録管理、実行ディスパッチを所有
- チャネル Plugin はスコープ付きアクション検出、ケイパビリティ検出、およびチャネル固有のスキーマフラグメントを所有
- チャネル Plugin は、会話 ID にスレッド ID をエンコードする方法や親会話から継承する方法など、プロバイダー固有のセッション会話文法を所有
- チャネル Plugin はアクションアダプターを通じて最終アクションを実行

チャネル Plugin 向けの SDK サーフェスは `ChannelMessageActionAdapter.describeMessageTool(...)` です。この統合された検出呼び出しにより、Plugin は表示可能なアクション、ケイパビリティ、スキーマへの追加をまとめて返せるため、それらの間にずれが生じません。

メッセージアクション名には、すべてのトランスポートがすべてのアクションをレンダリングできるように、意図的に閉じたコア所有の語彙を使用します。Plugin にアクション名を追加するにはコアへの PR が必要です。ランタイム登録は意図的にサポートされていません。

チャネル固有のメッセージツールパラメーターにローカルパスやリモートメディア URL などのメディアソースが含まれる場合、Plugin は `describeMessageTool(...)` から `mediaSourceParams` も返す必要があります。コアはこの明示的なリストを使用して、Plugin 所有のパラメーター名をハードコードせずに、サンドボックスパスの正規化と送信メディアアクセスのヒントを適用します。プロファイル専用のメディアパラメーターが `send` のような無関係なアクションで正規化されないように、チャネル全体で共通のフラットなリストではなく、アクション単位のマップを使用してください。

コアは、その検出ステップにランタイムスコープを渡します。重要なフィールドには次が含まれます。

- `accountId`
- `currentChannelId`
- `currentThreadTs`
- `currentMessageId`
- `sessionKey`
- `sessionId`
- `agentId`
- 信頼された受信 `requesterSenderId`

これは、コンテキスト依存の Plugin にとって重要です。チャネルは、コアの `message` ツールにチャネル固有の分岐をハードコードせずに、アクティブなアカウント、現在のルーム、スレッド、メッセージ、または信頼されたリクエスターの識別情報に基づいてメッセージアクションを非表示または表示できます。

このため、埋め込みランナーのルーティング変更も引き続き Plugin 側の作業です。共有 `message` ツールが現在のターンに適したチャネル所有のサーフェスを公開できるように、ランナーは現在のチャットおよびセッションの識別情報を Plugin 検出境界へ転送する責任を負います。

チャネル所有の実行ヘルパーについては、チャネル Plugin が実行ランタイムを自身の Plugin モジュール内に保持する必要があります。コアは `src/agents/tools` 配下の Discord、Slack、Telegram、WhatsApp のメッセージアクションランタイムを所有しなくなりました。個別の `plugin-sdk/*-action-runtime` サブパスは公開しません。これらの Plugin は、自身が所有する Plugin モジュールからローカルランタイムコードを直接インポートする必要があります。

同じ境界は、一般的なプロバイダー名付き SDK シームにも適用されます。コアは、Discord、Signal、Slack、WhatsApp、または類似する Plugin のチャネル固有の便利なバレルをインポートしてはなりません。コアで特定の動作が必要な場合は、バンドルされた Plugin 自身の `api.ts` / `runtime-api.ts` バレルを使用するか、その要件を共有 SDK の限定的で汎用的なケイパビリティへ昇格させます。

バンドルされた Plugin にも同じルールが適用されます。バンドルされた Plugin の `runtime-api.ts` から、その Plugin 固有のブランド名付き `openclaw/plugin-sdk/<plugin-id>` ファサードを再エクスポートしてはなりません。これらのブランド名付きファサードは、外部 Plugin と古い利用者向けの互換性シムとして残りますが、バンドルされた Plugin はローカルエクスポートと、`openclaw/plugin-sdk/channel-policy`、`openclaw/plugin-sdk/runtime-store`、`openclaw/plugin-sdk/webhook-ingress` などの限定的で汎用的な SDK サブパスを使用する必要があります。既存の外部エコシステムとの互換性境界で必要な場合を除き、新しいコードで Plugin ID 固有の SDK ファサードを追加してはなりません。

投票には、具体的に次の 2 つの実行パスがあります。

- `outbound.sendPoll` は共通の投票モデルに適合するチャネル向けの共有ベースライン
- `actions.handleAction("poll")` はチャネル固有の投票セマンティクスや追加の投票パラメーター向けの推奨パス

コアは、Plugin の投票ディスパッチがアクションを処理しないと判断するまで共有投票の解析を遅延するようになりました。これにより、Plugin 所有の投票ハンドラーは、汎用投票パーサーに先に阻止されることなく、チャネル固有の投票フィールドを受け入れられます。

起動シーケンス全体については、[Plugin アーキテクチャの内部構造](/ja-JP/plugins/architecture-internals)を参照してください。

## ケイパビリティ所有権モデル

OpenClaw は、ネイティブ Plugin を、無関係な統合を寄せ集めたものではなく、**企業**または**機能**の所有権境界として扱います。

つまり、次のようになります。

- 企業 Plugin は通常、その企業に関する OpenClaw 向けサーフェスをすべて所有
- 機能 Plugin は通常、自身が導入する機能サーフェス全体を所有
- チャネルはプロバイダー動作を場当たり的に再実装せず、共有コアケイパビリティを使用

<AccordionGroup>
  <Accordion title="ベンダーの複数ケイパビリティ">
    `google` はテキスト推論、CLI バックエンド、埋め込み、音声、リアルタイム音声、メディア理解、画像・音楽・動画生成、Web 検索を所有します。`openai` はテキスト推論、埋め込み、音声、リアルタイム文字起こし、リアルタイム音声、メディア理解、画像・動画生成を所有します。`minimax` はテキスト推論に加えて、メディア理解、音声、画像・音楽・動画生成、Web 検索を所有します。
  </Accordion>
  <Accordion title="ベンダーの単一ケイパビリティ">
    `arcee` と `chutes` はテキスト推論のみを所有し、`microsoft` は音声のみを所有します。ベンダー Plugin は、そのベンダーのより広範なサーフェスを扱う必要が生じるまで、この限定的な範囲を維持できます。
  </Accordion>
  <Accordion title="機能 Plugin">
    `voice-call` は通話トランスポート、ツール、CLI、ルート、Twilio メディアストリームブリッジを所有しますが、ベンダー Plugin を直接インポートせず、共有の音声、リアルタイム文字起こし、リアルタイム音声ケイパビリティを使用します。
  </Accordion>
</AccordionGroup>

目指す最終状態は次のとおりです。

- ベンダーの OpenClaw 向けサーフェスは、テキストモデル、音声、画像、動画にまたがる場合でも単一の Plugin に配置
- 他のベンダーも自身のサーフェス領域について同様に対応可能
- チャネルはどのベンダー Plugin がプロバイダーを所有しているかを意識せず、コアが公開する共有ケイパビリティ契約を使用

重要な違いは次のとおりです。

- **Plugin** = 所有権境界
- **ケイパビリティ** = 複数の Plugin が実装または使用できるコア契約

したがって、OpenClaw に動画などの新しいドメインを追加する場合、最初に問うべきことは「どのプロバイダーに動画処理をハードコードするか」ではありません。最初に問うべきことは「コアの動画ケイパビリティ契約をどのように定義するか」です。その契約が存在すれば、ベンダー Plugin はそれに対して登録でき、チャネルおよび機能 Plugin はそれを使用できます。

ケイパビリティがまだ存在しない場合、通常は次の対応が適切です。

<Steps>
  <Step title="ケイパビリティを定義">
    不足しているケイパビリティをコアで定義します。
  </Step>
  <Step title="SDK を通じて公開">
    型付けされた形で Plugin API およびランタイムを通じて公開します。
  </Step>
  <Step title="利用側を接続">
    チャネルおよび機能をそのケイパビリティに接続します。
  </Step>
  <Step title="ベンダー実装">
    ベンダー Plugin が実装を登録できるようにします。
  </Step>
</Steps>

これにより、単一のベンダーや一度限りの Plugin 固有コードパスに依存するコア動作を避けながら、所有権を明確に保てます。

### ケイパビリティの階層化

コードの配置先を判断する際は、次のメンタルモデルを使用してください。

<Tabs>
  <Tab title="コアケイパビリティ層">
    共有オーケストレーション、ポリシー、フォールバック、設定のマージルール、配信セマンティクス、型付き契約。
  </Tab>
  <Tab title="ベンダー Plugin 層">
    ベンダー固有の API、認証、モデルカタログ、音声合成、画像生成、動画バックエンド、使用量エンドポイント。
  </Tab>
  <Tab title="チャネルおよび機能 Plugin 層">
    コアケイパビリティを使用し、サーフェス上に提示する Discord、Slack、音声通話などの統合。
  </Tab>
</Tabs>

たとえば、TTS は次の構造に従います。

- コアは返信時の TTS ポリシー、フォールバック順序、設定、チャネル配信を所有
- `elevenlabs`、`google`、`microsoft`、`openai` は合成実装を所有
- `voice-call` は電話通話用 TTS ランタイムヘルパーを使用

今後のケイパビリティでも、同じパターンを優先してください。

### 複数ケイパビリティを持つ企業 Plugin の例

企業 Plugin は、外部から見て一貫したまとまりを持つ必要があります。OpenClaw にモデル、音声、リアルタイム文字起こし、リアルタイム音声、メディア理解、画像生成、動画生成、Web 取得、Web 検索の共有契約がある場合、ベンダーはすべてのサーフェスを 1 か所で所有できます。

```ts
import { definePluginEntry } from "openclaw/plugin-sdk/plugin-entry";
import { exampleAiMedia } from "./exampleai-media.js";

export default definePluginEntry({
  id: "exampleai",
  name: "ExampleAI",
  description: "ExampleAI のモデルとメディアケイパビリティ。",
  register(api) {
    api.registerProvider({
      id: "exampleai",
      // 認証、モデルカタログ、ランタイムフック
    });

    api.registerSpeechProvider({
      id: "exampleai",
      // ベンダーの音声設定 — SpeechProviderPlugin インターフェースを直接実装
    });

    api.registerMediaUnderstandingProvider({
      id: "exampleai",
      capabilities: ["image", "audio", "video"],
      describeImage: (req) => exampleAiMedia.describeImage(req),
      transcribeAudio: (req) => exampleAiMedia.transcribeAudio(req),
      describeVideo: (req) => exampleAiMedia.describeVideo(req),
    });

    api.registerWebSearchProvider({
      id: "exampleai-search",
      createTool() {
        // ベンダー所有の Web 検索ツールを返す。
      },
    });
  },
});
```

重要なのは、正確なヘルパー名ではありません。重要なのは構造です。

- 単一の Plugin がベンダーサーフェスを所有
- コアは引き続きケイパビリティ契約を所有
- プロバイダーリクエストの変換と HTTP ヘルパーはベンダー Plugin 内に保持
- チャネルおよび機能 Plugin はベンダーコードではなく `api.runtime.*` ヘルパーを使用
- 契約テストで、Plugin が所有すると宣言したケイパビリティを登録したことを検証可能

### ケイパビリティの例: 動画理解

OpenClaw はすでに画像、音声、動画の理解を 1 つの共有ケイパビリティとして扱っています。同じ所有権モデルがここにも適用されます。

<Steps>
  <Step title="コアが契約を定義する">
    コアがメディア理解の契約を定義します。
  </Step>
  <Step title="ベンダー Plugin が登録する">
    ベンダー Plugin は、該当する `describeImage`、`transcribeAudio`、`describeVideo` を登録します。
  </Step>
  <Step title="コンシューマーが共有動作を使用する">
    チャネルと機能 Plugin は、ベンダーコードへ直接接続するのではなく、共有されたコアの動作を利用します。
  </Step>
</Steps>

これにより、あるプロバイダーの動画に関する前提をコアに組み込むことを避けられます。Plugin はベンダー側のサーフェスを所有し、コアは機能契約とフォールバック動作を所有します。

動画生成でも、すでに同じ手順を使用しています。コアが型付き機能契約とランタイムヘルパーを所有し、ベンダー Plugin がその契約に対して `api.registerVideoGenerationProvider(...)` の実装を登録します。

具体的な展開チェックリストが必要ですか？[機能クックブック](/ja-JP/plugins/adding-capabilities)を参照してください。

## 契約と適用

Plugin API サーフェスは、意図的に `OpenClawPluginApi` に型付けして集約されています。この契約は、サポートされる登録ポイントと、Plugin が利用できるランタイムヘルパーを定義します。

これが重要な理由：

- Plugin 作者が安定した単一の内部標準を利用できる
- 同じプロバイダー ID を 2 つの Plugin が登録する場合など、所有権の重複をコアが拒否できる
- 不正な登録に対して、起動時に対処可能な診断情報を提示できる
- 契約テストで同梱 Plugin の所有権を適用し、気付かないまま乖離することを防止できる

適用には 2 つの層があります：

<AccordionGroup>
  <Accordion title="ランタイム登録の適用">
    Plugin レジストリは、Plugin の読み込み時に登録を検証します。たとえば、プロバイダー ID の重複、音声プロバイダー ID の重複、不正な登録に対しては、未定義の動作ではなく Plugin の診断情報が生成されます。
  </Accordion>
  <Accordion title="契約テスト">
    テスト実行時に同梱 Plugin が契約レジストリへ記録されるため、OpenClaw は所有権を明示的に検証できます。現在、これはモデルプロバイダー、音声プロバイダー、Web 検索プロバイダー、同梱登録の所有権に使用されています。
  </Accordion>
</AccordionGroup>

実際には、どの Plugin がどのサーフェスを所有するかを OpenClaw が事前に把握できます。所有権が暗黙的ではなく、宣言され、型付けされ、テスト可能であるため、コアとチャネルをシームレスに構成できます。

### 契約に含めるべきもの

<Tabs>
  <Tab title="良い契約">
    - 型付けされている
    - 小さい
    - 機能に特化している
    - コアが所有する
    - 複数の Plugin で再利用できる
    - ベンダーに関する知識がなくてもチャネルや機能から利用できる

  </Tab>
  <Tab title="悪い契約">
    - コアに隠されたベンダー固有のポリシー
    - レジストリを迂回する、単発の Plugin 用エスケープハッチ
    - ベンダー実装へ直接アクセスするチャネルコード
    - `OpenClawPluginApi` または `api.runtime` の一部ではないアドホックなランタイムオブジェクト

  </Tab>
</Tabs>

判断に迷う場合は、抽象化レベルを引き上げてください。まず機能を定義し、その後で Plugin がそこへ組み込めるようにします。

## 実行モデル

ネイティブ OpenClaw Plugin は Gateway と同じ**プロセス内**で実行されます。サンドボックス化されていません。読み込まれたネイティブ Plugin は、コアコードと同じプロセスレベルの信頼境界を持ちます。

<Warning>
ネイティブ Plugin に伴う影響：Plugin はツール、ネットワークハンドラー、フック、サービスを登録できます。Plugin のバグによって Gateway がクラッシュしたり不安定になったりする可能性があります。また、悪意のあるネイティブ Plugin は、OpenClaw プロセス内で任意のコードを実行することと同等です。
</Warning>

OpenClaw は現在、互換バンドルをメタデータ／コンテンツパックとして扱うため、デフォルトではより安全です。現在のリリースでは、主に Skills のバンドルを指します。

同梱されていない Plugin には、許可リストと明示的なインストール／読み込みパスを使用してください。ワークスペース Plugin は本番環境のデフォルトではなく、開発時のコードとして扱ってください。

同梱ワークスペースパッケージ名では、Plugin ID を npm 名に固定してください。デフォルトは `@openclaw/<id>` です。パッケージが意図的により限定された Plugin の役割を公開する場合は、`-provider`、`-plugin`、`-speech`、`-sandbox`、`-media-understanding` など、承認済みの型付きサフィックスを使用します。

<Note>
**信頼に関する注意：** `plugins.allow` が信頼するのは、ソースの来歴ではなく **Plugin ID** です。同梱 Plugin と同じ ID を持つワークスペース Plugin は、そのワークスペース Plugin が有効化されているか許可リストに登録されている場合、意図的に同梱版を上書きします。これは正常な動作であり、ローカル開発、パッチテスト、ホットフィックスに有用です。同梱 Plugin の信頼は、インストールメタデータではなく、読み込み時にディスク上に存在するマニフェストとコードというソーススナップショットから解決されます。インストール記録が破損または置換されても、実際のソースが宣言する範囲を超えて、同梱 Plugin の信頼サーフェスが暗黙的に拡大することはありません。
</Note>

## エクスポート境界

OpenClaw がエクスポートするのは、実装上の利便性ではなく機能です。

機能登録は公開したままにします。契約に含まれないヘルパーのエクスポートは削減してください：

- 同梱 Plugin 固有のヘルパーサブパス
- 公開 API を意図していないランタイム配管用サブパス
- ベンダー固有の便利なヘルパー
- 実装の詳細であるセットアップ／オンボーディング用ヘルパー

予約済みの同梱 Plugin 用ヘルパーサブパスは、生成される SDK エクスポートマップから廃止されました。所有者固有のヘルパーは、それを所有する Plugin パッケージ内に保持してください。再利用可能なホスト動作のみを、`plugin-sdk/gateway-runtime`、`plugin-sdk/security-runtime`、注入される Plugin API 機能などの汎用 SDK 契約へ昇格させてください。

## 内部構造とリファレンス

読み込みパイプライン、レジストリモデル、プロバイダーのランタイムフック、Gateway HTTP ルート、メッセージツールスキーマ、チャネルターゲットの解決、プロバイダーカタログ、コンテキストエンジン Plugin、新しい機能を追加するためのガイドについては、[Plugin アーキテクチャの内部構造](/ja-JP/plugins/architecture-internals)を参照してください。

## 関連項目

- [Plugin の構築](/ja-JP/plugins/building-plugins)
- [Plugin マニフェスト](/ja-JP/plugins/manifest)
- [Plugin SDK のセットアップ](/ja-JP/plugins/sdk-setup)
