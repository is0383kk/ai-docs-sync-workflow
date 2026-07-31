---
read_when:
    - プロバイダーのランタイムフック、チャンネルのライフサイクル、またはパッケージパックの実装
    - Plugin の読み込み順序またはレジストリ状態のデバッグ
    - 新しい Plugin 機能またはコンテキストエンジン Plugin の追加
summary: Plugin アーキテクチャの内部構造：読み込みパイプライン、レジストリ、ランタイムフック、HTTP ルート、リファレンステーブル
title: Plugin アーキテクチャの内部構造
x-i18n:
    generated_at: "2026-07-26T10:21:25Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 278ac23a9454ab69407c59fa197e75756fa0dc5880fcae6c3eecc15bd4733a09
    source_path: plugins/architecture-internals.md
    workflow: 16
---

公開 capability モデル、plugin の形状、所有権/実行契約については、[Plugin アーキテクチャ](/ja-JP/plugins/architecture)を参照してください。このページでは、ロードパイプライン、レジストリ、ランタイムフック、Gateway HTTP ルート、インポートパス、スキーマテーブルという内部メカニズムを扱います。

## ロードパイプライン

起動時、OpenClaw はおおよそ次の処理を行います。

1. plugin ルートの候補を検出する
2. ネイティブまたは互換性のあるバンドルマニフェストとパッケージメタデータを読み取る
3. 安全でない候補を拒否する
4. plugin 設定を正規化する（`plugins.enabled`、`allow`、`deny`、`entries`、
   `slots`、`load.paths`）
5. 各候補を有効にするか決定する
6. 有効なネイティブモジュールをロードする：ビルド済みの同梱モジュールにはネイティブローダーを使用し、
   サードパーティ製のローカルソース TypeScript には緊急用の Jiti フォールバックを使用する
7. ネイティブの `register(api)` フックを呼び出し、登録内容を plugin レジストリへ収集する
8. コマンドやランタイムサーフェスにレジストリを公開する

安全性ゲートは、ランタイム実行の**前**に動作します。次の場合、検出処理は候補をブロックします。

- 解決されたエントリが plugin ルート外へ出ている
- パス（またはそのルートディレクトリ）が全ユーザーから書き込み可能である
- 同梱されていない plugin について、パスの所有者が現在の uid（または root）と一致しない

全ユーザーから書き込み可能な同梱ディレクトリに対しては、ゲートによる再確認の前に、まずインプレースの `chmod` 修復を試みます（npm/グローバルインストールではパッケージディレクトリが `0777` となる場合があります）。同梱元については、所有権チェックを完全に省略します。

ブロックされた候補についても、判明している場合は出力される診断に plugin ID が含まれます（それ以外の理由で拒否されたディレクトリ内のマニフェストから解決された ID も含みます）。そのため、その ID を参照する設定には、無関係な「不明な plugin」エラーではなく、パス安全性の警告に関連付けられたブロック済み plugin が表示されます。

### マニフェスト優先の動作

マニフェストは、コントロールプレーンにおける信頼できる情報源です。OpenClaw はこれを使用して次の処理を行います。

- plugin を識別する
- 宣言されたチャンネル、Skills、設定スキーマ、またはバンドル capability を検出する
- `plugins.entries.<id>.config` を検証する
- Control UI のラベルやプレースホルダーを補完する
- インストールやカタログのメタデータを表示する
- plugin ランタイムをロードせずに、低コストの有効化およびセットアップ記述子を保持する

ネイティブ plugin では、ランタイムモジュールがデータプレーン部分です。フック、ツール、コマンド、プロバイダーフローなどの実際の動作を登録します。

オプションのマニフェスト `activation` および `setup` ブロックは、コントロールプレーンにとどまります。これらは有効化計画とセットアップ検出のためのメタデータ専用記述子であり、ランタイム登録、`register(...)`、または `setupEntry` の代わりにはなりません。ライブ有効化のコンシューマーは、より広範なレジストリの実体化に先立ち、マニフェストのコマンド、チャンネル、プロバイダーのヒントを使用して plugin のロード対象を絞り込みます。

- CLI のロードでは、要求されたプライマリコマンドを所有する plugin に絞り込む
- チャンネルのセットアップ/plugin 解決では、要求された
  チャンネル ID を所有する plugin に絞り込む
- 明示的なプロバイダーのセットアップ/ランタイム解決では、要求された
  プロバイダー ID を所有する plugin に絞り込む
- Gateway の起動計画では、明示的な起動時
  インポートに `activation.onStartup` を使用する。起動メタデータがない plugin は、より限定的な
  有効化トリガーによってのみロードされる

有効化プランナーは、既存の呼び出し元向けの ID 専用 API と、診断向けの計画 API の両方を公開します。計画エントリは plugin が選択された理由を報告し、明示的な `activation.*` ヒントと、マニフェスト所有権によるフォールバックを区別します。

| 理由（`activation.*` ヒント由来）   | 理由（マニフェスト所有権由来）                                                             |
| ------------------------------------ | -------------------------------------------------------------------------------------------- |
| `activation-agent-harness-hint`      | —                                                                                            |
| `activation-capability-hint`         | —                                                                                            |
| `activation-channel-hint`            | `manifest-channel-owner`（`channels`）                                                        |
| `activation-command-hint`            | `manifest-command-alias`（`commandAliases`）                                                  |
| `activation-provider-hint`           | `manifest-provider-owner`（`providers`）、`manifest-setup-provider-owner`（`setup.providers`） |
| `activation-route-hint`              | —                                                                                            |
| —（フックトリガーにはヒントのバリアントがない） | `manifest-hook-owner`（`hooks`）、`manifest-tool-contract`（`contracts.tools`）                |

この理由の区別が互換性の境界です。既存の plugin メタデータは引き続き機能し、新しいコードはランタイムのロードセマンティクスを変更せずに、広範なヒントやフォールバック動作を検出できます。

広範な `all` スコープを要求するリクエスト時のランタイムプリロードでも、設定、起動計画、設定済みチャンネル、スロット、自動有効化ルール
（`src/plugins/effective-plugin-ids.ts` の `resolveEffectivePluginIds`）から、明示的で実効的な plugin ID セットを導出します。その導出されたセットが空の場合、OpenClaw はスコープを検出可能なすべての plugin に拡大せず、空のままにします。

セットアップ検出では、`setup.providers` や `setup.cliBackends` などの記述子所有 ID を優先して候補 plugin を絞り込み、それができない場合は、セットアップ時のランタイムフックが依然として必要な plugin に対して `setup-api` を使用します。プロバイダーのセットアップ一覧では、プロバイダーランタイムをロードせずに、マニフェストの `providerAuthChoices`、記述子から導出されたセットアップ選択肢、インストールカタログのメタデータを使用します。明示的な `setup.requiresRuntime: false` は記述子のみを使用するカットオフです。`requiresRuntime` を省略すると、互換性のために従来の setup-api フォールバックが維持されます。検出された複数の plugin が、正規化された同じセットアッププロバイダーまたは CLI バックエンド ID を要求する場合、セットアップ検索は検出順序に依存せず、曖昧な所有者を拒否します。セットアップランタイムが実際に実行されるとき、レジストリ診断は、従来の plugin をブロックすることなく、`setup.providers` / `setup.cliBackends` と、setup-api によって実際に登録されたプロバイダーまたは CLI バックエンドとの不一致を報告します。

### Plugin キャッシュの境界

OpenClaw は、plugin の検出結果や直接のマニフェストレジストリデータを、実時間ベースの期間を設けてキャッシュしません。インストール、マニフェスト編集、ロードパスの変更は、次回の明示的なメタデータ読み取りまたはスナップショット再構築時に反映されなければなりません。マニフェストファイルパーサーは、開いたマニフェストのパスに加え、デバイス/inode、サイズ、mtime/ctime をキーとする、上限付きのファイルシグネチャキャッシュを保持します。このキャッシュは、変更されていないバイト列の再解析を避けるだけであり、検出、レジストリ、所有者、ポリシーの回答をキャッシュしてはなりません。

安全なメタデータの高速パスは、隠れたキャッシュではなく、明示的なオブジェクト所有権です。Gateway 起動時のホットパスでは、現在の `PluginMetadataSnapshot`、導出された `PluginLookUpTable`、または明示的なマニフェストレジストリをコールチェーン経由で渡す必要があります。設定検証、起動時の自動有効化、plugin のブートストラップ、プロバイダーの選択では、それらのオブジェクトが現在の設定と plugin インベントリを表している間、再利用できます。セットアップ検索では、特定のセットアップパスが明示的なマニフェストレジストリを受け取らない限り、引き続き必要に応じてマニフェストメタデータを再構築します。隠れた検索キャッシュを追加するのではなく、コールドパスのフォールバックとして維持してください。入力が変化した場合は、スナップショットを変更したり履歴コピーを保持したりせず、再構築して置き換えます。有効な plugin レジストリのビューと、同梱チャンネルのブートストラップヘルパーは、現在のレジストリ/ルートから再計算する必要があります。短命なマップは、1 回の呼び出し内で処理の重複排除や再入の防止に使用して構いませんが、プロセスのメタデータキャッシュにしてはなりません。

plugin のロードでは、永続的なキャッシュ層はランタイムロードです。コードやインストール済みアーティファクトが実際にロードされる場合は、次のようなローダー状態を再利用できます。

- `PluginLoaderCacheState` および互換性のある有効なランタイムレジストリ
- 同じランタイムサーフェスを繰り返しインポートしないために使用される jiti/module キャッシュおよび公開サーフェスローダーキャッシュ
- インストール済み plugin アーティファクト用のファイルシステムキャッシュ
- パスの正規化や重複解決のための、呼び出しごとの短命なマップ

これらのキャッシュはデータプレーンの実装詳細です。呼び出し元が意図的にランタイムロードを要求した場合を除き、「このプロバイダーを所有する plugin はどれか」のようなコントロールプレーンの問いに答えるために使用してはなりません。

次の対象には、永続的なキャッシュや実時間ベースのキャッシュを追加しないでください。

- 検出結果
- 直接のマニフェストレジストリ
- インストール済み plugin インデックスから再構築されたマニフェストレジストリ
- プロバイダー所有者の検索、モデルの抑制、プロバイダーポリシー、または公開アーティファクトの
  メタデータ
- 変更されたマニフェスト、インストール済みインデックス、
  またはロードパスが次回のメタデータ読み取り時に反映されるべき、その他のマニフェスト由来の回答

永続化されたインストール済み plugin インデックスからマニフェストメタデータを再構築する呼び出し元は、そのレジストリを必要に応じて再構築します。インストール済みインデックスは永続的なソースプレーン状態であり、隠れたインプロセスのメタデータキャッシュではありません。

## レジストリモデル

ロードされた plugin は、無関係なコアのグローバル状態を直接変更しません。中央の plugin レジストリ（`src/plugins/registry-types.ts` の `PluginRegistry`）へ登録します。このレジストリは、plugin レコード（ID、ソース、配布元、状態、診断）に加え、すべての capability に対応する配列を追跡します。対象には、ツール、従来のフックと型付きフック、チャンネル、プロバイダー、Gateway RPC ハンドラー、HTTP ルート、CLI レジストラー、バックグラウンドサービス、plugin 所有コマンドのほか、音声、埋め込み、画像/動画/音楽生成、Web フェッチ/検索、エージェントハーネス、セッションアクションなど、数十種類の型付きプロバイダーファミリーが含まれます。

その後、コア機能は plugin モジュールと直接やり取りせず、このレジストリから情報を読み取ります。これにより、ロードは一方向に保たれます。

- plugin モジュール -> レジストリへの登録
- コアランタイム -> レジストリの利用

この分離は保守性の面で重要です。ほとんどのコアサーフェスに必要な統合ポイントは、「各 plugin モジュールを個別に特別扱いする」ことではなく、「レジストリを読み取る」ことだけになります。

## 会話バインディングのコールバック

会話をバインドする plugin は、承認が解決されたときに処理を実行できます。

バインド要求が承認または拒否された後にコールバックを受け取るには、`api.onConversationBindingResolved(...)` を使用します。

```ts
export default {
  id: "my-plugin",
  register(api) {
    api.onConversationBindingResolved(async (event) => {
      if (event.status === "approved") {
        // この plugin と会話のバインディングが作成されました。
        console.log(event.binding?.conversationId);
        return;
      }

      // 要求が拒否されたため、ローカルの保留状態をすべてクリアします。
      console.log(event.request.conversation.conversationId);
    });
  },
};
```

コールバックペイロードのフィールド：

- `status`: `"approved"` または `"denied"`
- `decision`: `"allow-once"`、`"allow-always"`、または `"deny"`
- `binding`: 承認された要求に対して解決されたバインディング
- `request`: 元の要求の概要、デタッチのヒント、送信者 ID、および
  会話メタデータ

このコールバックは通知専用です。会話のバインドを許可される主体は変更されず、コアの承認処理が完了した後に実行されます。

## プロバイダーのランタイムフック

プロバイダー plugin には、3 つの層があります。

- 実行前の低コストな検索に使用する**マニフェストメタデータ**：
  `setup.providers[].envVars`、`providerAuthAliases`、`providerAuthChoices`、
  および `channelConfigs`。
- **設定時フック**：`catalog` と `applyConfigDefaults`。
- **ランタイムフック**：認証、モデル解決、
  ストリームのラップ、思考レベル、リプレイポリシー、使用量エンドポイントを扱う 40 個以上のオプションフック。[フックの順序と使用方法](#hook-order-and-usage)を参照してください。

OpenClaw は、汎用エージェントループ、フェイルオーバー、トランスクリプト処理、および
ツールポリシーを引き続き担います。これらのフックは、完全に独自の推論トランスポートを
必要とせずにプロバイダー固有の動作を実装するための拡張面です。

汎用の認証、ステータス、モデル選択の各パスが Plugin ランタイムを
読み込まずに参照すべき環境変数ベースの認証情報をプロバイダーが持つ場合は、マニフェストの `setup.providers[].envVars` を使用します。
あるプロバイダー ID で別のプロバイダー ID の環境変数、認証プロファイル、
設定ベースの認証、および API キーのオンボーディング選択肢を再利用する場合は、マニフェストの `providerAuthAliases`
を使用します。オンボーディングまたは認証方法選択の CLI 画面が、プロバイダーランタイムを
読み込まずにプロバイダーの選択肢 ID、グループラベル、および単一フラグによる簡易な認証設定を
認識する必要がある場合は、マニフェストの `providerAuthChoices` を使用します。
オンボーディングのラベルや OAuth のクライアント ID／クライアントシークレット設定変数など、
運用者向けのヒントには、プロバイダーランタイムの
`envVars` を使用します。

環境変数によるチャネル設定と認証は、所有元の
`channelConfigs.<id>.schema` および設定記述子を通じて定義します。

### フックの順序と使用方法

モデル／プロバイダー Plugin の場合、OpenClaw はおおむね次の順序でフックを呼び出します。
「使用する場合」列は、判断のためのクイックガイドです。
OpenClaw が呼び出さなくなった互換性専用のプロバイダーフィールド（
`ProviderPlugin.capabilities` や `suppressBuiltInModel` など）は、意図的に
ここには記載していません。

| フック                              | 機能                                                                                                   | 使用する場合                                                                                                                                   |
| --------------------------------- | -------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------- |
| `catalog`                         | `models.json` の生成中にプロバイダー設定を `models.providers` へ公開                                | プロバイダーがカタログまたはベース URL のデフォルトを所有する場合                                                                                                  |
| `applyConfigDefaults`             | 設定の具体化時にプロバイダー所有のグローバル設定デフォルトを適用                                      | デフォルトが認証モード、環境変数、またはプロバイダーのモデルファミリーのセマンティクスに依存する場合                                                                         |
| _(組み込みモデル検索)_         | OpenClaw は最初に通常のレジストリ／カタログ経路を試行する                                                          | _(Plugin フックではない)_                                                                                                                         |
| `normalizeModelId`                | 検索前にレガシーまたはプレビュー版のモデル ID エイリアスを正規化                                                     | 正規モデルの解決前にプロバイダーがエイリアスのクリーンアップを所有する場合                                                                                 |
| `normalizeTransport`              | 汎用モデルの組み立て前にプロバイダーファミリーの `api` / `baseUrl` を正規化                                      | 同じトランスポートファミリー内のカスタムプロバイダー ID について、プロバイダーがトランスポートのクリーンアップを所有する場合                                                          |
| `normalizeConfig`                 | ランタイム／プロバイダーの解決前に `models.providers.<id>` を正規化                                           | Plugin 側に置くべき設定のクリーンアップがプロバイダーに必要な場合。バンドル済みの Google ファミリーヘルパーも、サポート対象の Google 設定エントリを補完する   |
| `applyNativeStreamingUsageCompat` | ネイティブストリーミング使用量の互換性書き換えを設定プロバイダーに適用                                               | エンドポイントに応じたネイティブストリーミング使用量メタデータの修正がプロバイダーに必要な場合                                                                          |
| `resolveConfigApiKey`             | ランタイム認証の読み込み前に、設定プロバイダーの環境変数マーカー認証を解決                                       | プロバイダーが独自の環境変数マーカー API キー解決フックを公開する場合                                                                                |
| `resolveSyntheticAuth`            | 平文を永続化せずに、ローカル／セルフホスト型または設定ベースの認証を公開                                   | プロバイダーが合成／ローカル認証情報マーカーで動作できる場合                                                                                 |
| `resolveExternalAuthProfiles`     | プロバイダー所有の外部認証プロファイルをオーバーレイする。CLI／アプリ所有の認証情報では、デフォルトの `persistence` は `runtime-only` | コピーした更新トークンを永続化せずに、プロバイダーが外部認証情報を再利用する場合。マニフェストで `contracts.externalAuthProviders` を宣言する |
| `shouldDeferSyntheticProfileAuth` | 環境変数／設定ベースの認証よりも、保存済みの合成プロファイルプレースホルダーの優先度を下げる                                      | 優先されるべきでない合成プレースホルダープロファイルをプロバイダーが保存する場合                                                                 |
| `resolveDynamicModel`             | ローカルレジストリにまだ存在しない、プロバイダー所有のモデル ID に対する同期フォールバック                                       | プロバイダーが任意のアップストリームモデル ID を受け入れる場合                                                                                                 |
| `prepareDynamicModel`             | 非同期ウォームアップ後、`resolveDynamicModel` を再実行                                                           | 不明な ID を解決する前にプロバイダーがネットワークメタデータを必要とする場合                                                                                  |
| `normalizeResolvedModel`          | 組み込みランナーが解決済みモデルを使用する前の最終書き換え                                               | プロバイダーがトランスポートの書き換えを必要とするが、引き続きコアトランスポートを使用する場合                                                                             |
| `normalizeToolSchemas`            | 組み込みランナーに渡される前にツールスキーマを正規化                                                    | プロバイダーがトランスポートファミリーのスキーマをクリーンアップする必要がある場合                                                                                                |
| `inspectToolSchemas`              | 正規化後にプロバイダー所有のスキーマ診断を公開                                                  | コアにプロバイダー固有のルールを持たせず、プロバイダーがキーワード警告を提供する場合                                                                 |
| `resolveReasoningOutputMode`      | ネイティブまたはタグ付きの推論出力契約を選択                                                              | ネイティブフィールドではなく、タグ付きの推論／最終出力がプロバイダーに必要な場合                                                                         |
| `prepareExtraParams`              | 汎用ストリームオプションラッパーの前にリクエストパラメーターを正規化                                              | デフォルトのリクエストパラメーターまたはプロバイダーごとのパラメータークリーンアップが必要な場合                                                                           |
| `createStreamFn`                  | 通常のストリーム経路をカスタムトランスポートで完全に置き換える                                                   | 単なるラッパーではなく、カスタムワイヤープロトコルがプロバイダーに必要な場合                                                                                     |
| `wrapStreamFn`                    | 汎用ラッパーの適用後にストリームをラップ                                                              | カスタムトランスポートを使わずに、リクエストヘッダー／本文／モデルの互換性ラッパーがプロバイダーに必要な場合                                                          |
| `resolveTransportTurnState`       | ターンごとのネイティブトランスポートヘッダーまたはメタデータを付加                                                           | 汎用トランスポートからプロバイダー固有のターン識別情報を送信する場合                                                                       |
| `resolveWebSocketSessionPolicy`   | ネイティブ WebSocket ヘッダーまたはセッションのクールダウンポリシーを付加                                                    | 汎用 WS トランスポートのセッションヘッダーまたはフォールバックポリシーをプロバイダーが調整する場合                                                               |
| `formatApiKey`                    | 認証プロファイルのフォーマッター：保存済みプロファイルをランタイムの `apiKey` 文字列へ変換                                     | プロバイダーが追加の認証メタデータを保存し、カスタムのランタイムトークン形式を必要とする場合                                                                    |
| `refreshOAuth`                    | カスタム更新エンドポイントまたは更新失敗ポリシー向けの OAuth 更新オーバーライド                                  | プロバイダーが OpenClaw の共有更新処理に適合しない場合                                                                                          |
| `buildAuthDoctorHint`             | OAuth 更新失敗時に修復ヒントを追加                                                                  | 更新失敗後にプロバイダー所有の認証修復ガイダンスが必要な場合                                                                      |
| `matchesContextOverflowError`     | プロバイダー所有のコンテキストウィンドウ超過マッチャー                                                                 | 汎用ヒューリスティクスでは検出できない生の超過エラーがプロバイダーにある場合                                                                                |
| `classifyFailoverReason`          | プロバイダー所有のフェイルオーバー理由分類                                                                  | プロバイダーが生の API／トランスポートエラーをレート制限／過負荷などに対応付けられる場合                                                                          |
| `isCacheTtlEligible`              | プロキシ／バックホールプロバイダー向けのプロンプトキャッシュポリシー                                                               | プロバイダーがプロキシ固有のキャッシュ TTL 制御を必要とする場合                                                                                                |
| `buildMissingAuthMessage`         | 汎用の認証情報不足リカバリーメッセージを置き換える                                                      | プロバイダー固有の認証情報不足リカバリーヒントが必要な場合                                                                                 |
| `augmentModelCatalog`             | 検出後に合成／最終カタログ行を追加（非推奨。下記参照）                                  | `models list` とピッカーに、合成された将来互換行がプロバイダーに必要な場合                                                                     |
| `resolveThinkingProfile`          | モデル固有の `/think` レベルセット、表示ラベル、デフォルト                                                 | 選択したモデル向けに、プロバイダーがカスタム思考段階または二値ラベルを公開する場合                                                                 |
| `isBinaryThinking`                | 推論のオン／オフ切り替え互換性フック                                                                     | プロバイダーが二値の思考オン／オフのみを公開する場合                                                                                                  |
| `supportsXHighThinking`           | `xhigh` 推論サポート互換性フック                                                                   | 一部のモデルだけで `xhigh` を有効にする場合                                                                                             |
| `resolveDefaultThinkingLevel`     | デフォルトの `/think` レベル互換性フック                                                                      | モデルファミリー向けのデフォルト `/think` ポリシーをプロバイダーが所有する場合                                                                                      |
| `isModernModelRef`                | ライブプロファイルフィルターとスモーク選択向けの最新モデルマッチャー                                              | ライブ／スモークで優先するモデルのマッチングをプロバイダーが所有する場合                                                                                             |
| `prepareRuntimeAuth`              | 推論の直前に、設定済み認証情報を実際のランタイムトークン／キーへ交換                       | トークン交換または短期間有効なリクエスト認証情報がプロバイダーに必要な場合                                                                             |
| `resolveUsageAuth`                | `/usage` および関連するステータス画面向けに、使用量／請求認証情報を解決                                     | カスタムの使用量／クォータトークン解析または別の使用量認証情報がプロバイダーに必要な場合                                                               |
| `fetchUsageSnapshot`              | 認証解決後に、プロバイダー固有の使用量／クォータスナップショットを取得して正規化                             | プロバイダー固有の使用量エンドポイントまたはペイロードパーサーが必要な場合                                                                           |
| `createEmbeddingProvider`         | メモリ/検索用のプロバイダー所有の埋め込みアダプターを構築する                                                     | メモリの埋め込み動作はプロバイダー Plugin に属する                                                                                    |
| `buildReplayPolicy`               | プロバイダーのトランスクリプト処理を制御するリプレイポリシーを返す                                        | プロバイダーにはカスタムトランスクリプトポリシー（たとえば、思考ブロックの除去）が必要                                                               |
| `sanitizeReplayHistory`           | 汎用トランスクリプトのクリーンアップ後にリプレイ履歴を書き換える                                                        | プロバイダーには共有 Compaction ヘルパーを超える、プロバイダー固有のリプレイ書き換えが必要                                                             |
| `validateReplayTurns`             | 組み込みランナーの実行前に、リプレイターンの最終検証または再形成を行う                                           | プロバイダーのトランスポートには、汎用サニタイズ後のより厳格なターン検証が必要                                                                    |
| `onModelSelected`                 | プロバイダー所有の選択後副作用を実行する                                                                 | モデルがアクティブになったとき、プロバイダーにはテレメトリまたはプロバイダー所有の状態が必要                                                                  |

`normalizeModelId`、`normalizeTransport`、および `normalizeConfig` は、まず一致したプロバイダー Plugin を確認し、その後、いずれかが実際にモデル ID またはトランスポート／設定を変更するまで、フック対応の他のプロバイダー Plugin にフォールスルーします。これにより、呼び出し元がどのバンドル Plugin が書き換えを所有しているかを把握していなくても、エイリアス／互換プロバイダーシムが機能し続けます。サポート対象の Google 系設定エントリをプロバイダーフックが書き換えない場合も、バンドルされた Google 設定ノーマライザーがその互換性クリーンアップを適用します。

プロバイダーに完全にカスタムなワイヤープロトコルまたはカスタムリクエストエグゼキューターが必要な場合、それは別種の拡張です。これらのフックは、OpenClaw の通常の推論ループ上で引き続き実行されるプロバイダー動作のためのものです。

`resolveUsageAuth` は、OpenClaw が `fetchUsageSnapshot` を呼び出すか、使用量／ステータス表示用の汎用認証情報解決にフォールバックするかを決定します。プロバイダーに使用量取得用の認証情報がある場合は `{ token, accountId?, subscriptionType?, rateLimitTier? }` を返します（オプションのプランメタデータは `fetchUsageSnapshot` に渡されます）。プロバイダー所有の使用量認証がリクエストを処理し、汎用 API キー／OAuth フォールバックを抑止する必要がある場合は `{ handled: true }` を返します。プロバイダーが使用量認証を処理しなかった場合は `null` または `undefined` を返します。

組織または請求用の認証情報は、マニフェストの `providerUsageAuthEnvVars` で宣言します。これにより、それらを推論認証の候補にすることなく、汎用の検出およびシークレット除去サーフェスが認識できます。

### プロバイダーの例

```ts
api.registerProvider({
  id: "example-proxy",
  label: "Example Proxy",
  auth: [],
  catalog: {
    order: "simple",
    run: async (ctx) => {
      const apiKey = ctx.resolveProviderApiKey("example-proxy").apiKey;
      if (!apiKey) {
        return null;
      }
      return {
        provider: {
          baseUrl: "https://proxy.example.com/v1",
          apiKey,
          api: "openai-completions",
          models: [{ id: "auto", name: "Auto" }],
        },
      };
    },
  },
  resolveDynamicModel: (ctx) => ({
    id: ctx.modelId,
    name: ctx.modelId,
    provider: "example-proxy",
    api: "openai-completions",
    baseUrl: "https://proxy.example.com/v1",
    reasoning: false,
    input: ["text"],
    cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 },
    contextWindow: 128000,
    maxTokens: 8192,
  }),
  prepareRuntimeAuth: async (ctx) => {
    const exchanged = await exchangeToken(ctx.apiKey);
    return {
      apiKey: exchanged.token,
      baseUrl: exchanged.baseUrl,
      expiresAt: exchanged.expiresAt,
    };
  },
  resolveUsageAuth: async (ctx) => {
    const auth = await ctx.resolveOAuthToken();
    return auth ? { token: auth.token } : null;
  },
  fetchUsageSnapshot: async (ctx) => {
    return await fetchExampleProxyUsage(ctx.token, ctx.timeoutMs, ctx.fetchFn);
  },
});
```

### 組み込みの例

バンドルされたプロバイダー Plugin は、各ベンダーのカタログ、認証、思考、リプレイ、および使用量の要件に合わせて、上記のフックを組み合わせます。正式なフックセットは `extensions/` 配下の各 Plugin にあります。このページでは、その一覧を複製するのではなく、構成例を示します。

<AccordionGroup>
  <Accordion title="パススルーカタログプロバイダー">
    OpenRouter、Kilocode、Z.AI、xAI は、`catalog` に加えて
    `resolveDynamicModel`／`prepareDynamicModel` を登録し、OpenClaw の静的カタログより先に
    アップストリームのモデル ID を提示できるようにします。
  </Accordion>
  <Accordion title="OAuth および使用量エンドポイントのプロバイダー">
    GitHub Copilot、Gemini CLI、ChatGPT Codex、MiniMax、Xiaomi、z.ai は、
    `prepareRuntimeAuth` または `formatApiKey` と `resolveUsageAuth` +
    `fetchUsageSnapshot` を組み合わせ、トークン交換と `/usage` 統合を所有します。
  </Accordion>
  <Accordion title="リプレイおよびトランスクリプトクリーンアップ系">
    名前付きの共有ファミリー（`google-gemini`、`passthrough-gemini`、
    `anthropic-by-model`、`hybrid-anthropic-openai`）により、各 Plugin がクリーンアップを
    再実装する代わりに、`buildReplayPolicy` を介してトランスクリプトポリシーを
    選択できます。
  </Accordion>
  <Accordion title="カタログ専用プロバイダー">
    `byteplus`、`cloudflare-ai-gateway`、`huggingface`、`kimi-coding`、`nvidia`、
    `qianfan`、`synthetic`、`together`、`venice`、`vercel-ai-gateway`、および
    `volcengine` は、`catalog` のみを登録し、共有推論ループを利用します。
  </Accordion>
  <Accordion title="Anthropic 固有のストリームヘルパー">
    ベータヘッダー、`/fast`／`serviceTier`、および `context1m` は、
    汎用 SDK ではなく、Anthropic Plugin の公開 `api.ts`／`contract-api.ts` シーム
    （`wrapAnthropicProviderStream`、`resolveAnthropicBetas`、
    `resolveAnthropicFastMode`、`resolveAnthropicServiceTier`）内にあります。
  </Accordion>
</AccordionGroup>

## ランタイムヘルパー

Plugin は、`api.runtime` を介して選択されたコアヘルパーにアクセスできます。TTS の場合：

```ts
const clip = await api.runtime.tts.textToSpeech({
  text: "Hello from OpenClaw",
  cfg: api.config,
});

const result = await api.runtime.tts.textToSpeechTelephony({
  text: "Hello from OpenClaw",
  cfg: api.config,
});

const voices = await api.runtime.tts.listVoices({
  provider: "elevenlabs",
  cfg: api.config,
});
```

注：

- `textToSpeech` は、ファイル／ボイスノートサーフェス用の通常のコア TTS 出力ペイロードを返します。
- コアの `tts` 設定およびプロバイダー選択を使用します。
- PCM オーディオバッファーとサンプルレートを返します。Plugin はプロバイダー向けにリサンプリング／エンコードする必要があります。
- `listVoices` はプロバイダーごとにオプションです。ベンダー所有の音声選択画面またはセットアップフローに使用します。
- コアは解決済みのリクエスト期限をプロバイダーの `listVoices` フックに渡します。プロバイダー固有のタイムアウト設定で上書きできます。
- 音声一覧には、ロケール、性別、パーソナリティタグなど、プロバイダー対応の選択画面向けの詳細なメタデータを含めることができます。
- 現在、OpenAI と ElevenLabs はテレフォニーをサポートしています。Microsoft はサポートしていません。

Plugin は、`api.registerSpeechProvider(...)` を介して音声プロバイダーを登録することもできます。

```ts
api.registerSpeechProvider({
  id: "acme-speech",
  label: "Acme Speech",
  isConfigured: ({ config }) => Boolean(config.messages?.tts),
  synthesize: async (req) => {
    return {
      audioBuffer: Buffer.from([]),
      outputFormat: "mp3",
      fileExtension: ".mp3",
      voiceCompatible: false,
    };
  },
});
```

注：

- TTS ポリシー、フォールバック、および返信配信はコアに保持します。
- ベンダー所有の音声合成動作には、音声プロバイダーを使用します。
- 従来の Microsoft `edge` 入力は、`microsoft` プロバイダー ID に正規化されます。
- 推奨される所有モデルは企業単位です。OpenClaw がこれらの
  機能コントラクトを追加するにつれて、1 つのベンダー Plugin がテキスト、音声、
  画像、および将来のメディアプロバイダーを所有できます。

画像／音声／動画の理解については、Plugin は汎用のキー／値バッグではなく、型付けされた
メディア理解プロバイダーを 1 つ登録します。

```ts
api.registerMediaUnderstandingProvider({
  id: "google",
  capabilities: ["image", "audio", "video"],
  describeImage: async (req) => ({ text: "..." }),
  transcribeAudio: async (req) => ({ text: "..." }),
  describeVideo: async (req) => ({ text: "..." }),
});
```

注：

- オーケストレーション、フォールバック、設定、およびチャネル配線はコアに保持します。
- ベンダーの動作はプロバイダー Plugin に保持します。
- 追加的な拡張は型付けを維持する必要があります。新しいオプションメソッド、新しいオプションの
  結果フィールド、新しいオプション機能を使用します。
- 動画生成はすでに同じパターンに従っています。
  - コアが機能コントラクトとランタイムヘルパーを所有します
  - ベンダー Plugin が `api.registerVideoGenerationProvider(...)` を登録します
  - 機能／チャネル Plugin が `api.runtime.videoGeneration.*` を使用します

メディア理解ランタイムヘルパーでは、Plugin は次のように呼び出せます。

```ts
const image = await api.runtime.mediaUnderstanding.describeImageFile({
  filePath: "/tmp/inbound-photo.jpg",
  cfg: api.config,
  agentDir: "/tmp/agent",
});

const video = await api.runtime.mediaUnderstanding.describeVideoFile({
  filePath: "/tmp/inbound-video.mp4",
  cfg: api.config,
});

const extraction = await api.runtime.mediaUnderstanding.extractStructuredWithModel({
  provider: "codex",
  model: "gpt-5.6-sol",
  input: [
    {
      type: "image",
      buffer: receiptImageBuffer,
      fileName: "receipt.png",
      mime: "image/png",
    },
    { type: "text", text: "Use the printed fields as the source of truth." },
  ],
  instructions: "Return entities and searchable tags.",
  schemaName: "example.evidence",
  jsonSchema: {
    type: "object",
    properties: {
      entities: { type: "array", items: { type: "string" } },
      tags: { type: "array", items: { type: "string" } },
    },
  },
  cfg: api.config,
});
```

音声文字起こしでは、Plugin はメディア理解ランタイムまたは従来の STT エイリアスのいずれかを使用できます。

```ts
const { text } = await api.runtime.mediaUnderstanding.transcribeAudioFile({
  filePath: "/tmp/inbound-audio.ogg",
  cfg: api.config,
  // Optional when MIME cannot be inferred reliably:
  mime: "audio/ogg",
});
```

注：

- `api.runtime.mediaUnderstanding.*` は、画像／音声／動画理解の推奨共有サーフェスです。
- `extractStructuredWithModel(...)` は、範囲が限定されたプロバイダー所有の画像優先抽出向けの
  Plugin-facing シームです。少なくとも 1 つの画像入力を含めてください。
  テキスト入力は補足的なコンテキストです。製品 Plugin がルートとスキーマを所有し、
  OpenClaw がプロバイダー／ランタイム境界を所有します。
- コアのメディア理解音声設定（`tools.media.audio`）およびプロバイダーのフォールバック順序を使用します。
- 文字起こし出力が生成されない場合（たとえば、スキップされた入力／サポートされていない入力）は、`{ text: undefined }` を返します。

Plugin は、`api.runtime.subagent` を介してバックグラウンドのサブエージェント実行を開始することもできます。

```ts
const result = await api.runtime.subagent.run({
  sessionKey: "agent:main:subagent:search-helper",
  message: "Expand this query into focused follow-up searches.",
  toolsAlsoAllow: ["my_plugin_progress"],
  provider: "openai",
  model: "gpt-4.1-mini",
  deliver: false,
});
```

注：

- `provider` および `model` は、永続的なセッション変更ではなく、実行ごとのオプションの上書きです。
- `toolsAlsoAllow` は、呼び出し元 Plugin によって登録された、所有者が一意に特定できる正確なツール名を受け付けます。コアの名前および曖昧な名前は拒否されます。これは通常のプロファイルに追加されますが、オペレーターの許可リストと拒否設定が引き続き優先されます。
- OpenClaw は、信頼された呼び出し元に対してのみ、これらの上書きフィールドを受け入れます。
- Plugin 所有のフォールバック実行では、オペレーターが `plugins.entries.<id>.subagent.allowModelOverride: true` を使用して明示的に有効化する必要があります。
- 信頼された Plugin を特定の正規 `provider/model` ターゲットに制限するには `plugins.entries.<id>.subagent.allowedModels` を使用し、任意のターゲットを明示的に許可するには `"*"` を使用します。
- 信頼されていない Plugin のサブエージェント実行も引き続き動作しますが、上書きリクエストは暗黙にフォールバックするのではなく拒否されます。
- Plugin が作成したサブエージェントセッションには、作成元の Plugin ID がタグ付けされます。フォールバックの `api.runtime.subagent.deleteSession(...)` が削除できるのは、それらの所有セッションのみです。任意のセッションを削除するには、引き続き管理者スコープの Gateway リクエストが必要です。

Web 検索では、Plugin はエージェントツールの配線に直接アクセスする代わりに、
共有ランタイムヘルパーを使用できます。

```ts
const providers = api.runtime.webSearch.listProviders({
  config: api.config,
});

const result = await api.runtime.webSearch.search({
  config: api.config,
  args: {
    query: "OpenClaw plugin runtime helpers",
    count: 5,
  },
});
```

Plugin は、`api.registerWebSearchProvider(...)` を介して Web 検索プロバイダーを登録することもできます。

注：

- プロバイダー選択、認証情報の解決、および共有リクエストのセマンティクスはコアに保持します。
- ベンダー固有の検索トランスポートには、Web 検索プロバイダーを使用します。
- `api.runtime.webSearch.*` は、エージェントツールラッパーに依存せずに検索動作を必要とする機能／チャネル Plugin 向けの推奨共有サーフェスです。

### `api.runtime.imageGeneration`

```ts
const result = await api.runtime.imageGeneration.generate({
  config: api.config,
  args: { prompt: "親しみやすいロブスターのマスコット", size: "1024x1024" },
});

const providers = api.runtime.imageGeneration.listProviders({
  config: api.config,
});
```

- `generate(...)`: 設定された画像生成プロバイダーチェーンを使用して画像を生成します。
- `listProviders(...)`: 利用可能な画像生成プロバイダーとその機能を一覧表示します。

## Gateway HTTP ルート

Plugin は `api.registerHttpRoute(...)` を使用して HTTP エンドポイントを公開できます。

```ts
api.registerHttpRoute({
  path: "/acme/webhook",
  auth: "plugin",
  match: "exact",
  handler: async (_req, res) => {
    res.statusCode = 200;
    res.end("ok");
    return true;
  },
});
```

ルートのフィールド:

- `path`: Gateway HTTP サーバー配下のルートパス。
- `auth`: 必須。`"gateway"` または `"plugin"`。通常の Gateway 認証を要求するには `"gateway"` を、Plugin が管理する認証/Webhook 検証には `"plugin"` を使用します。
- `match`: 省略可能。`"exact"`（デフォルト）または `"prefix"`。
- `handleUpgrade`: 同じルートでの WebSocket アップグレードリクエスト用の、省略可能なハンドラー。
- `replaceExisting`: 省略可能。同じ Plugin が自身の既存のルート登録を置き換えられるようにします。
- `handler`: ルートがリクエストを処理した場合は `true` を返します。

注意:

- `api.registerHttpHandler(...)` は削除されており、使用すると Plugin の読み込みエラーが発生します。代わりに `api.registerHttpRoute(...)` を使用してください。
- Plugin のルートでは `auth` を明示的に宣言する必要があります。
- 完全一致する `path + match` の競合は、`replaceExisting: true` でない限り拒否されます。また、ある Plugin が別の Plugin のルートを置き換えることはできません。
- `auth` レベルが異なる重複ルートは拒否されます。`exact`/`prefix` のフォールスルーチェーンは、同じ認証レベル内にのみ維持してください。
- `auth: "plugin"` ルートには、オペレーターのランタイムスコープが自動的に付与されることは**ありません**。これは Plugin が管理する Webhook/署名検証用であり、特権 Gateway ヘルパー呼び出し用ではありません。
- `auth: "gateway"` ルートは Gateway リクエストのランタイムスコープ内で実行されます。デフォルトのサーフェス（`gatewayRuntimeScopeSurface: "write-default"`）は意図的に保守的です。
  - 共有シークレットによる Bearer 認証（`gateway.auth.mode = "token"` / `"password"`）および信頼済みプロキシ以外の認証方式には、呼び出し元が `x-openclaw-scopes` を送信した場合でも、単一の `operator.write` スコープが付与されます
  - 明示的な `x-openclaw-scopes` ヘッダーのない `trusted-proxy` 呼び出し元も、従来の `operator.write` のみのサーフェスを維持します
  - `x-openclaw-scopes` を送信する `trusted-proxy` 呼び出し元には、代わりに宣言されたスコープが付与されます
  - ルートは `gatewayRuntimeScopeSurface: "trusted-operator"` を有効にすることで、アイデンティティを伴う認証モードでは常に `x-openclaw-scopes` を尊重できます（ヘッダーがない場合は CLI のデフォルトスコープ一式にフォールバックします）
- `auth: "gateway"` ルートを使用するサンドボックス化された外部 Control UI タブでは、認証済みのブートストラップのみが発行する短期間有効な署名付き Cookie グラントを使用します。Plugin 認証のタブは、直接 iframe パスを維持します。マウント前に、親は同じ不透明なサンドボックス内でルート所有のプローブを実行し、ブラウザーのプライバシーポリシーによって Cookie がブロックされる場合はフェイルクローズします。グラントは、所有する Plugin、一致するルートのルートパス、現在の認証世代に紐付けられます。プロセスごとにランダムな Cookie 名により、同じホスト上の信頼済み Gateway 同士が互いに上書きすることを防ぎますが、Cookie が TCP ポートを分離することはありません。したがって、Gateway のホスト名は単一のクレデンシャル境界です。他のポートを含め、そのホスト名上で相互に信頼されていないサービスを同居させないでください。ルートディスパッチは、別の Plugin が所有するネストされたルートに対する再利用を拒否します。サンドボックスの子孫は Cookie の観点ではクロスサイトとなるため、グラントは `operator.read` を伴う `GET` と `HEAD` のみを受け入れます。変更操作と WebSocket アップグレードは、明示的に Gateway 認証されたサーフェスに留まります。この Cookie は意図的に CHIPS を使用できません。現在のブラウザーではパーティションキーにクロスサイト祖先ビットが含まれるため、ネストされた不透明なサンドボックスフレームは同一ルートのアセットにアクセスできなくなるからです。この Cookie にはセキュアコンテキストとクロスサイト Cookie に対するブラウザーの許可が必要です。そのため、Gateway 認証を使用する外部タブは、平文 HTTP の LAN オリジンや、サードパーティー Cookie が完全にブロックされている環境では利用できません。HTTPS/Tailscale Serve、または互換性のある Cookie ポリシーを備えたブラウザー信頼済みのループバックを使用してください。
- このグラントは、Gateway Bearer トークンの漏えいと、意図しないルート/スコープの再利用を防止します。ネイティブ Plugin 間にセキュリティ境界を作成するものではありません。ネイティブ Plugin のコードと、それが提供する UI コンテンツは、同じ信頼済みインプロセス Plugin 境界の一部であり続けます。
- 実用上のルール: Gateway 認証された Plugin ルートを暗黙の管理者サーフェスだと想定しないでください。ルートに管理者専用の動作が必要な場合は、`trusted-operator` スコープサーフェスを有効にし、アイデンティティを伴う認証モードを要求し、明示的な `x-openclaw-scopes` ヘッダーの契約を文書化してください。
- ルートの照合と認証後、通常のハンドラーは Gateway のルートワーク受付に参加します。準備中または再起動中の Gateway は、ハンドラーを呼び出す前に `503` を返します。限定的な例外は、マニフェストで権限を付与された `auth: "gateway"` ルートであり、さらにルート固有の `trusted-operator` サーフェスを有効にしている場合です。このルートは、停止制御のディスパッチが到達不能にならないようアクセス可能なままとなりますが、同じ Plugin の通常の兄弟ルートは受付境界の背後に留まります。WebSocket の `handleUpgrade` 所有権にも、同じアトミックな受付境界が適用されます。ハンドラーがソケットを受け入れた後のソケットの存続期間は Plugin が所有し、この境界では追跡されません。

## Plugin SDK のインポートパス

新しい Plugin を作成する際は、モノリシックな `openclaw/plugin-sdk` ルートバレルではなく、用途を限定した SDK サブパスを使用してください。コアのサブパス:

| サブパス                           | 用途                                         |
| ---------------------------------- | -------------------------------------------- |
| `openclaw/plugin-sdk/plugin-entry` | Plugin 登録のプリミティブ                    |
| `openclaw/plugin-sdk/channel-core` | チャンネルのエントリ/ビルドヘルパー         |
| `openclaw/plugin-sdk/core`         | 汎用共有ヘルパーと包括的な契約               |

チャンネル Plugin は、用途を限定した一連のシーム（`channel-setup`、
`setup-runtime`、`setup-tools`、`channel-pairing`、
`channel-contract`、`channel-feedback`、`channel-inbound`、`channel-outbound`、
`command-auth`、`secret-input`、`webhook-ingress`、
`channel-targets`、`channel-actions`）から選択します。承認動作は、関連のない
Plugin フィールド間で混在させるのではなく、1 つの `approvalCapability` 契約に
集約する必要があります。[チャンネル Plugin](/ja-JP/plugins/sdk-channel-plugins)を参照してください。

ランタイムおよび設定ヘルパーは、対応する用途限定の `*-runtime` サブパス
（`approval-runtime`、`agent-runtime`、`lazy-runtime`、`directory-runtime`、
`text-runtime`、`runtime-store`、`system-event-runtime`、`heartbeat-runtime`、
`channel-activity-runtime` など）にあります。広範な `config-runtime` 互換バレルではなく、
`config-contracts`、`plugin-config-runtime`、`runtime-config-snapshot`、
`config-mutation` を優先してください。

<Info>
`openclaw/plugin-sdk/channel-lifecycle`、小規模なチャンネルヘルパーファサード、
`openclaw/plugin-sdk/config-runtime`、`openclaw/plugin-sdk/infra-runtime`
は、古い Plugin 向けの非推奨の互換シムです。新しいコードでは、代わりに用途を限定した
汎用プリミティブをインポートしてください。
</Info>

リポジトリ内部のエントリポイント（バンドルされた各 Plugin パッケージのルートごと）:

- `index.js` — バンドルされた Plugin のエントリ
- `api.js` — ヘルパー/型のバレル
- `runtime-api.js` — ランタイム専用バレル
- `setup-entry.js` — セットアップ Plugin のエントリ

外部 Plugin は `openclaw/plugin-sdk/*` サブパスのみをインポートしてください。コアまたは別の Plugin から、
他の Plugin パッケージの `src/*` をインポートしないでください。
ファサードから読み込まれるエントリポイントは、アクティブなランタイム設定のスナップショットが
存在する場合はそれを優先し、その後ディスク上の解決済み設定ファイルにフォールバックします。

`image-generation`、`media-understanding`、
`speech` などの機能固有サブパスが存在するのは、現在バンドルされた Plugin が使用しているためです。これらは
外部向けの長期固定契約として自動的に保証されるものではありません。依存する場合は、関連する SDK
リファレンスページを確認してください。

## メッセージツールのスキーマ

Plugin は、リアクション、既読、投票など、メッセージ以外のプリミティブに対する
チャンネル固有の `describeMessageTool(...)` スキーマへの追加を所有する必要があります。
共有の送信プレゼンテーションでは、プロバイダー固有のボタン、コンポーネント、ブロック、カードの各フィールドではなく、
汎用の `MessagePresentation` 契約を使用してください。
契約、フォールバックルール、プロバイダーのマッピング、Plugin 作成者向けチェックリストについては、
[メッセージプレゼンテーション](/ja-JP/plugins/message-presentation)を参照してください。

送信機能を持つ Plugin は、メッセージ機能を通じてレンダリング可能な内容を宣言します。

- セマンティックなプレゼンテーションブロック（`text`、`context`、
  `divider`、`chart`、`table`、`buttons`、`select`）には `presentation`
- 固定配信リクエストには `delivery-pin`

コアは、プレゼンテーションをネイティブにレンダリングするか、テキストにフォールバックするかを決定します。
汎用メッセージツールからプロバイダー固有の UI エスケープハッチを公開しないでください。
従来のネイティブスキーマ向けの非推奨 SDK ヘルパーは、既存の
サードパーティー Plugin 向けに引き続きエクスポートされますが、新しい Plugin では使用しないでください。

## チャンネルターゲットの解決

チャンネル Plugin は、チャンネル固有のターゲットセマンティクスを所有する必要があります。共有の
送信ホストは汎用に保ち、プロバイダーのルールにはメッセージングアダプターのサーフェスを使用してください。

- `messaging.inferTargetChatType({ to })` は、正規化されたターゲットをディレクトリ検索の前に
  `direct`、`group`、`channel` のどれとして扱うべきかを決定します。
- `messaging.targetResolver.looksLikeId(raw, normalized)` は、入力がディレクトリ検索ではなく
  ID のような値としての解決に直接進むべきかどうかをコアに通知します。
- `messaging.targetResolver.reservedLiterals` は、そのプロバイダーにおいて
  チャンネル/セッション参照となる単独の語を一覧化します。解決処理は、予約済みリテラルを拒否する前に
  設定済みのディレクトリエントリを維持し、その後ディレクトリ検索に失敗した場合はフェイルクローズします。
- `messaging.targetResolver.resolveTarget(...)` は、正規化後または
  ディレクトリ検索失敗後にコアが最終的なプロバイダー所有の解決を必要とする場合の Plugin フォールバックです。
- `messaging.resolveOutboundSessionRoute(...)` は、ターゲットの解決後に
  プロバイダー固有のセッションルート構築を所有します。

推奨される分割:

- ピア/グループを検索する前に行うべきカテゴリ判定には `inferTargetChatType` を使用します。
- 「これを明示的な/ネイティブのターゲット ID として扱う」かどうかの確認には `looksLikeId` を使用します。
- 広範なディレクトリ検索ではなく、プロバイダー固有の正規化フォールバックに `resolveTarget` を使用します。
- チャット ID、スレッド ID、JID、ハンドル、ルーム ID などのプロバイダー固有の ID は、
  汎用 SDK フィールドではなく、`target` の値またはプロバイダー固有のパラメーター内に保持してください。

## 設定に基づくディレクトリ

設定からディレクトリエントリを導出する Plugin は、そのロジックを
Plugin 内に保持し、`openclaw/plugin-sdk/directory-runtime` の共有ヘルパーを
再利用してください。

チャンネルが次のような設定に基づくピア/グループを必要とする場合に使用します。

- 許可リストに基づく DM ピア
- 設定済みのチャンネル/グループマップ
- アカウントスコープの静的ディレクトリフォールバック

`directory-runtime` の共有ヘルパーは、次の汎用操作のみを処理します。

- クエリのフィルタリング
- 上限の適用
- 重複排除/正規化ヘルパー
- `ChannelDirectoryEntry[]` の構築

チャンネル固有のアカウント検査と ID 正規化は、
Plugin 実装内に保持してください。

## プロバイダーカタログ

プロバイダー Plugin は、`registerProvider({ catalog: { run(...) { ... } } })` を使用して
推論用のモデルカタログを定義できます。

`catalog.run(...)` は、OpenClaw が
`models.providers` に書き込むものと同じ形式を返します。

- `{ provider }`（単一のプロバイダーエントリ用）
- `{ providers }`（複数のプロバイダーエントリ用）

Plugin がプロバイダー固有のモデル ID、ベース URL のデフォルト、
または認証によって制限されるモデルメタデータを所有する場合は、`catalog` を使用します。

`catalog.order` は、Plugin のカタログを OpenClaw の
組み込み暗黙的プロバイダーに対していつマージするかを制御します。

- `simple`：通常の API キーまたは環境変数駆動のプロバイダー
- `profile`：認証プロファイルが存在するときに表示されるプロバイダー
- `paired`：関連する複数のプロバイダーエントリを合成するプロバイダー
- `late`：他の暗黙的プロバイダーの後に行う最終パス

キーが衝突した場合は後のプロバイダーが優先されるため、Plugin は同じプロバイダー ID を持つ
組み込みプロバイダーエントリを意図的に上書きできます。

Plugin は `api.registerModelCatalogProvider({ provider, kinds, staticCatalog, liveCatalog
})` を通じて読み取り専用のモデル行を
公開することもできます。これは一覧、ヘルプ、選択画面の将来的な経路であり、
`text`、`voice`、`image_generation`、`video_generation`、`music_generation`
の各行をサポートします。プロバイダー Plugin は引き続き、実際のエンドポイント呼び出し、トークン交換、
ベンダーレスポンスのマッピングを所有します。コアは共通の行形式、ソースラベル、
メディアツールのヘルプ書式を所有します。メディア生成プロバイダーの登録では、
`defaultModel`、`models`、`capabilities` から
静的カタログ行が自動的に合成されます。

互換性：

- `discovery` は従来のエイリアスとして引き続き動作しますが、非推奨警告が出力されます
- `catalog` と `discovery` の両方が登録されている場合、OpenClaw は `catalog` を使用し、
  警告を出力します
- `augmentModelCatalog` は非推奨です。バンドルされたプロバイダーは
  `registerModelCatalogProvider` を通じて補足行を公開する必要があります

## 読み取り専用のチャネル検査

Plugin がチャネルを登録する場合は、`resolveAccount(...)` と併せて
`plugin.config.inspectAccount(cfg, accountId)` を実装することを推奨します。

理由：

- `resolveAccount(...)` はランタイム経路です。認証情報が完全に実体化されていることを前提としてよく、
  必須のシークレットがない場合は即座に失敗できます。
- `openclaw status`、`openclaw status --all`、
  `openclaw channels status`、`openclaw channels resolve`、doctor/config
  修復フローなどの読み取り専用コマンド経路では、設定を説明するだけのために
  ランタイム認証情報を実体化する必要はありません。

推奨される `inspectAccount(...)` の動作：

- 説明用のアカウント状態のみを返します。
- `enabled` と `configured` を保持します。
- 必要に応じて、次のような認証情報のソース／状態フィールドを含めます。
  - `tokenSource`、`tokenStatus`
  - `botTokenSource`、`botTokenStatus`
  - `appTokenSource`、`appTokenStatus`
  - `signingSecretSource`、`signingSecretStatus`
- 読み取り専用の可用性を報告するだけなら、生のトークン値を返す必要はありません。
  状態表示用コマンドには、`tokenStatus: "available"`（および対応するソース
  フィールド）を返せば十分です。
- 認証情報が SecretRef 経由で設定されているものの、
  現在のコマンド経路では利用できない場合は、`configured_unavailable` を使用します。

これにより、読み取り専用コマンドはクラッシュしたりアカウントが未設定であると誤って報告したりせず、
「設定済みだが、このコマンド経路では利用不可」と報告できます。

## パッケージパック

Plugin ディレクトリには、`openclaw.extensions` を含む `package.json` を配置できます。

```json
{
  "name": "my-pack",
  "openclaw": {
    "extensions": ["./src/safety.ts", "./src/tools.ts"],
    "setupEntry": "./src/setup-entry.ts"
  }
}
```

各エントリが 1 つの Plugin になります。パックに複数の拡張機能が列挙されている場合、
Plugin ID は `<manifestOrPackageName>/<fileBase>` になります（マニフェスト ID が
存在する場合はそれが優先され、存在しない場合はスコープなしの `package.json` 名が使用されます）。

Plugin が npm の依存関係をインポートする場合は、`node_modules` を使用できるように、
そのディレクトリに依存関係をインストールします（`npm install` / `pnpm install`）。

セキュリティ上のガードレール：すべての `openclaw.extensions` エントリは、シンボリックリンクの解決後も
Plugin ディレクトリ内に収まる必要があります。パッケージディレクトリの外に出るエントリは
拒否されます。

セキュリティ上の注意：`openclaw plugins install` は、プロジェクトローカルの
`npm install --omit=dev --ignore-scripts` を使用して Plugin の依存関係をインストールします
（ライフサイクルスクリプトなし、ランタイムの開発依存関係なし）。継承されたグローバル npm インストール設定は無視されます。
Plugin の依存関係ツリーは「純粋な JS/TS」に保ち、
`postinstall` ビルドを必要とするパッケージは避けてください。

任意：`openclaw.setupEntry` で、軽量なセットアップ専用モジュールを指定できます。
OpenClaw が無効化されたチャネル Plugin のセットアップ画面を必要とする場合、または
チャネル Plugin が有効でもまだ未設定の場合、完全な Plugin エントリではなく
`setupEntry` を読み込みます。これにより、メインの Plugin エントリがツール、フック、
その他のランタイム専用コードも接続している場合でも、起動とセットアップを軽量に保てます。

任意：`openclaw.startup.deferConfiguredChannelFullLoadUntilAfterListen`
を使用すると、チャネルがすでに設定済みであっても、Gateway のリッスン前の起動フェーズで、
チャネル Plugin を同じ `setupEntry` 経路にオプトインできます。

Gateway がリッスンを開始する前に存在する必要がある起動画面を
`setupEntry` が完全にカバーしている場合にのみ、これを使用してください。実際には、セットアップエントリが、
次のような起動時に依存するチャネル所有のすべての機能を登録する必要があります。

- チャネル登録自体
- Gateway がリッスンを開始する前に利用可能である必要があるすべての HTTP ルート
- 同じ期間内に存在する必要があるすべての Gateway メソッド、ツール、サービス

完全なエントリが必須の起動機能を 1 つでも引き続き所有している場合は、
このフラグを有効にしないでください。Plugin はデフォルトの動作のままにし、OpenClaw が
起動時に完全なエントリを読み込むようにします。

バンドルされたチャネルは、完全なチャネルランタイムが読み込まれる前にコアが参照できる、
セットアップ専用のコントラクト画面ヘルパーも公開できます。現在のセットアップ
昇格画面は次のとおりです。

- `singleAccountKeysToMove`
- `namedAccountPromotionKeys`
- `resolveSingleAccountPromotionTarget(...)`

コアは、完全な Plugin エントリを読み込まずに従来の単一アカウント用チャネル設定を
`channels.<id>.accounts.*` に昇格する必要がある場合、この画面を使用します。
現在のバンドル例は Matrix です。名前付きアカウントがすでに存在する場合、
認証／ブートストラップキーのみを名前付きの昇格先アカウントへ移動します。また、常に
`accounts.default` を作成するのではなく、設定済みの非正規デフォルトアカウントキーを保持できます。

これらのセットアップパッチアダプターにより、バンドルされたコントラクト画面の検出は遅延したままになります。
インポート時は軽量に保たれ、昇格画面はモジュールのインポート時に
バンドルされたチャネルの起動処理へ再進入するのではなく、初回使用時にのみ読み込まれます。

これらの起動画面に Gateway RPC メソッドが含まれる場合は、
Plugin 固有のプレフィックスを付けてください。コア管理名前空間（`config.*`、
`exec.approvals.*`、`wizard.*`、`update.*`）は予約されたままであり、Plugin がより狭いスコープを要求しても、
常に `operator.admin` に解決されます。

例：

```json
{
  "name": "@scope/my-channel",
  "openclaw": {
    "extensions": ["./index.ts"],
    "setupEntry": "./setup-entry.ts",
    "startup": {
      "deferConfiguredChannelFullLoadUntilAfterListen": true
    }
  }
}
```

### チャネルカタログのメタデータ

チャネル Plugin は、`openclaw.channel` を通じてセットアップ／検出メタデータを、
`openclaw.install` を通じてインストールヒントを公開できます。これにより、コアカタログにデータを持たせずに済みます。

例：

```json
{
  "name": "@openclaw/nextcloud-talk",
  "openclaw": {
    "extensions": ["./index.ts"],
    "channel": {
      "id": "nextcloud-talk",
      "label": "Nextcloud Talk",
      "selectionLabel": "Nextcloud Talk (self-hosted)",
      "docsPath": "/channels/nextcloud-talk",
      "docsLabel": "nextcloud-talk",
      "blurb": "Nextcloud Talk の Webhook ボットを介したセルフホスト型チャット。",
      "order": 65,
      "aliases": ["nc-talk", "nc"]
    },
    "install": {
      "npmSpec": "@openclaw/nextcloud-talk",
      "localPath": "<bundled-plugin-local-path>",
      "defaultChoice": "npm"
    }
  }
}
```

最小限の例以外で有用な `openclaw.channel` フィールド：

- `detailLabel`：より充実したカタログ／状態画面向けの副ラベル
- `docsLabel`：ドキュメントリンクのリンクテキストを上書き
- `preferOver`：このカタログエントリが優先される、優先度の低い Plugin／チャネル ID
- `selectionDocsPrefix`、`selectionDocsOmitLabel`、`selectionExtras`：選択画面の文言制御
- `markdownCapable`：送信時の書式設定判断のため、チャネルが Markdown 対応であることを示す
- `exposure.configured`：`false` に設定した場合、設定済みチャネル一覧画面からチャネルを非表示にする
- `exposure.setup`：`false` に設定した場合、対話型のセットアップ／設定選択画面からチャネルを非表示にする
- `exposure.docs`：ドキュメントのナビゲーション画面で、チャネルを内部用／非公開として示す
- `quickstartAllowFrom`：チャネルを標準のクイックスタート `allowFrom` フローにオプトインする
- `forceAccountBinding`：アカウントが 1 つしか存在しない場合でも、明示的なアカウントのバインドを必須にする
- `preferSessionLookupForAnnounceTarget`：通知先の解決時にセッション検索を優先する

OpenClaw は、**外部チャネルカタログ**（たとえば MPM
レジストリエクスポート）もマージできます。JSON ファイルを次のいずれかに配置します。

- `~/.openclaw/mpm/plugins.json`
- `~/.openclaw/mpm/catalog.json`
- `~/.openclaw/plugins/catalog.json`

または、`OPENCLAW_PLUGIN_CATALOG_PATHS`（または `OPENCLAW_MPM_CATALOG_PATHS`）で
1 つ以上の JSON ファイルを指定します（カンマ／セミコロン／`PATH` 区切り）。
各ファイルには `{ "entries": [ { "name": "@scope/pkg", "openclaw": { "channel": {...}, "install": {...} } } ] }` を含める必要があります。パーサーは、`"entries"` キーの従来のエイリアスとして
`"packages"` または `"plugins"` も受け入れます。

生成されたチャネルカタログエントリとプロバイダーインストールカタログエントリは、
生の `openclaw.install` ブロックとともに正規化されたインストール元の情報を公開します。
正規化された情報は、npm 指定が正確なバージョンか変動するセレクターか、
想定される整合性メタデータが存在するか、ローカルソースパスも利用可能かを示します。
カタログ／パッケージの ID が既知の場合、解析された npm パッケージ名がその ID からずれていると、
正規化された情報に警告が表示されます。また、`defaultChoice` が無効である場合、
利用できないソースを指している場合、または有効な npm ソースなしで npm 整合性メタデータが存在する場合も警告されます。
手作りのエントリやカタログシムでこのフィールドを合成する必要がないように、
利用側は `installSource` を追加可能な任意フィールドとして扱う必要があります。
これにより、オンボーディングと診断は Plugin ランタイムをインポートせずに、
ソースプレーンの状態を説明できます。

公式の外部 npm エントリでは、正確な `npmSpec` と
`expectedIntegrity` を使用することを推奨します。パッケージ名だけの指定や dist-tag も
互換性のため引き続き動作しますが、ソースプレーンの警告が表示されます。これにより、既存の Plugin を壊さず、
カタログをバージョン固定かつ整合性確認済みのインストールへ移行できます。
オンボーディングがローカルカタログパスからインストールする場合、`source: "path"` と、
可能であればワークスペース相対の `sourcePath` を含む管理対象 Plugin の
Plugin インデックスエントリを記録します。実際の絶対ロードパスは `plugins.load.paths` に保持されます。
インストールレコードでは、ローカルワークステーションのパスを長期保存される設定へ重複して記録しません。
これにより、ローカル開発用インストールをソースプレーン診断から確認可能にしつつ、
生のファイルシステムパスを公開する 2 つ目の画面を追加せずに済みます。
永続化された `installed_plugin_index` SQLite テーブルがインストール元の
信頼できる唯一の情報源であり、Plugin ランタイムモジュールを読み込まずに更新できます。
その `installRecords` マップは、Plugin マニフェストが存在しないか無効な場合でも永続的に保持されます。
`plugins` ペイロードは再構築可能なマニフェストビューです。

## コンテキストエンジン Plugin

コンテキストエンジン Plugin は、取り込み、組み立て、
Compaction におけるセッションコンテキストのオーケストレーションを所有します。Plugin から
`api.registerContextEngine(id, factory)` を使用して登録し、`plugins.slots.contextEngine` で
アクティブなエンジンを選択します。

Plugin でメモリ検索やフックを追加するだけでなく、デフォルトのコンテキスト
パイプラインを置換または拡張する必要がある場合に使用します。

```ts
import { buildMemorySystemPromptAddition } from "openclaw/plugin-sdk/core";

export default function (api) {
  api.registerContextEngine("lossless-claw", (ctx) => ({
    info: { id: "lossless-claw", name: "Lossless Claw", ownsCompaction: true },
    async ingest() {
      return { ingested: true };
    },
    async assemble({ messages, sessionKey, availableTools, citationsMode }) {
      return {
        messages,
        estimatedTokens: 0,
        systemPromptAddition: buildMemorySystemPromptAddition({
          availableTools: availableTools ?? new Set(),
          citationsMode,
          agentSessionKey: sessionKey,
        }),
      };
    },
    async compact() {
      return { ok: true, compacted: false };
    },
  }));
}
```

ファクトリ `ctx` は、構築時の初期化に使用できる任意の `config`、`agentDir`、`workspaceDir`
値を公開します。

ホストは、非レガシーエンジンの `assemble()` を呼び出す前に、登録済みの非同期メモリプロンプト準備を完了します。`buildMemorySystemPromptAddition(...)` は
同期のままであり、`assemble()` がアクティブな間、その不変の実行スナップショットを読み取ります。
指定されたツールと引用のコンテキストは変更せずに渡し、スナップショットが
実行境界を越えないようにしてください。

アクティブなハーネスに永続的なバックエンドスレッドがある場合、`assemble()` は `contextProjection` を返すことができます。
レガシーのターンごとの投影では省略してください。組み立てられたコンテキストを
バックエンドスレッドへ一度だけ注入し、エポックが変更されるまで再利用する場合は、
`{ mode: "thread_bootstrap", epoch }` を返してください。エンジン所有の Compaction パスの後など、
エンジンのセマンティックコンテキストが変更された後にエポックを変更してください。
ホストは、スレッドのブートストラップ投影内にツール呼び出しメタデータ、入力形式、
および秘匿化されたツール結果を保持できます。これにより、新しいバックエンドスレッドは、
シークレットを含む未加工のペイロードをコピーせずにツールの連続性を維持できます。

エンジンが Compaction アルゴリズムを所有して**いない**場合は、`compact()` の実装を維持し、
明示的に委譲してください。

```ts
import {
  buildMemorySystemPromptAddition,
  delegateCompactionToRuntime,
} from "openclaw/plugin-sdk/core";

export default function (api) {
  api.registerContextEngine("my-memory-engine", (ctx) => ({
    info: {
      id: "my-memory-engine",
      name: "My Memory Engine",
      ownsCompaction: false,
    },
    async ingest() {
      return { ingested: true };
    },
    async assemble({ messages, sessionKey, availableTools, citationsMode }) {
      return {
        messages,
        estimatedTokens: 0,
        systemPromptAddition: buildMemorySystemPromptAddition({
          availableTools: availableTools ?? new Set(),
          citationsMode,
          agentSessionKey: sessionKey,
        }),
      };
    },
    async compact(params) {
      return await delegateCompactionToRuntime(params);
    },
  }));
}
```

## 新しいケイパビリティの追加

Plugin が現在の API に収まらない動作を必要とする場合、非公開の内部アクセスによって
Plugin システムを迂回しないでください。不足しているケイパビリティを追加してください。

推奨手順：

1. **コアコントラクトを定義します。** コアが所有すべき共有動作を決定します。
   ポリシー、フォールバック、設定のマージ、ライフサイクル、チャンネル向けのセマンティクス、
   およびランタイムヘルパーの形式が含まれます。
2. **型付きの Plugin 登録／ランタイムサーフェスを追加します。** 
   `OpenClawPluginApi` や `api.runtime` を、実用上必要最小限の型付き
   ケイパビリティサーフェスで拡張します。
3. **コアとチャンネル／機能のコンシューマーを接続します。** チャンネルおよび機能 Plugin は、
   ベンダー実装を直接インポートせず、コアを介して新しいケイパビリティを
   使用する必要があります。
4. **ベンダー実装を登録します。** その後、ベンダー Plugin が
   各バックエンドをケイパビリティに登録します。
5. **コントラクトのカバレッジを追加します。** 所有権と登録形式が
   長期にわたって明示的に保たれるよう、テストを追加します。

この仕組みにより、OpenClaw は特定のプロバイダーの世界観にハードコードされることなく、
明確な方針を維持できます。具体的なファイルチェックリストと実例については、
[ケイパビリティクックブック](/ja-JP/plugins/adding-capabilities)を参照してください。

### ケイパビリティのチェックリスト

新しいケイパビリティを追加する場合、通常は次のサーフェスをまとめて
変更する必要があります。

- `src/<capability>/types.ts` 内のコアコントラクト型
- `src/<capability>/runtime.ts` 内のコアランナー／ランタイムヘルパー
- `src/plugins/types.ts` 内の Plugin API 登録サーフェス
- `src/plugins/registry.ts` 内の Plugin レジストリ接続
- 機能／チャンネル Plugin が使用する必要がある場合は、`src/plugins/runtime/*` 内の
  Plugin ランタイム公開
- `src/test-utils/plugin-registration.ts` 内のキャプチャ／テストヘルパー
- `src/plugins/contracts/registry.ts` 内の所有権／コントラクトアサーション
- `docs/` 内のオペレーター／Plugin ドキュメント

これらのサーフェスのいずれかが欠けている場合、通常はケイパビリティが
まだ完全に統合されていないことを示します。

### ケイパビリティのテンプレート

最小パターン：

```ts
// core contract
export type VideoGenerationProviderPlugin = {
  id: string;
  label: string;
  generateVideo: (req: VideoGenerationRequest) => Promise<VideoGenerationResult>;
};

// plugin API
api.registerVideoGenerationProvider({
  id: "openai",
  label: "OpenAI",
  async generateVideo(req) {
    return await generateOpenAiVideo(req);
  },
});

// shared runtime helper for feature/channel plugins
const clip = await api.runtime.videoGeneration.generate({
  prompt: "Show the robot walking through the lab.",
  cfg,
});
```

コントラクトテストのパターン（`src/plugins/contracts/registry.ts` は `providerContractPluginIds` などの所有権検索を公開し、
テストでは Plugin の `contracts.videoGenerationProviders` リストが実際の登録内容と一致することを検証します）：

```ts
expect(pluginManifest.contracts?.videoGenerationProviders).toEqual(["openai"]);
```

これにより、ルールはシンプルに保たれます。

- コアはケイパビリティのコントラクトとオーケストレーションを所有する
- ベンダー Plugin はベンダー実装を所有する
- 機能／チャンネル Plugin はランタイムヘルパーを使用する
- コントラクトテストは所有権を明示的に保つ

## 関連項目

- [Plugin アーキテクチャ](/ja-JP/plugins/architecture) — 公開ケイパビリティモデルと形式
- [Plugin SDK のサブパス](/ja-JP/plugins/sdk-subpaths)
- [Plugin SDK のセットアップ](/ja-JP/plugins/sdk-setup)
- [Plugin の構築](/ja-JP/plugins/building-plugins)
