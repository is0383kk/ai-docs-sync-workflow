---
read_when:
    - Codex モードの OpenClaw エージェントでネイティブ Codex プラグインを使用したい場合
    - ソースからインストールされた OpenAI 厳選の Codex Pluginを移行しています
    - 既存のワークスペースディレクトリにある Codex Plugin を設定しています
    - codexPlugins、アプリインベントリ、破壊的アクション、またはPluginアプリ診断のトラブルシューティングを行っている場合
summary: Codex モードの OpenClaw エージェント向けにネイティブ Codex Plugin を設定する
title: ネイティブ Codex プラグイン
x-i18n:
    generated_at: "2026-07-26T09:31:48Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 0b1cfa39838d4dbd1f33a1e5b7f52faec4b033f9fa98ef5c029003177c2e27e5
    source_path: plugins/codex-native-plugins.md
    workflow: 16
---

ネイティブ Codex Plugin 対応により、Codex モードの OpenClaw エージェントは、OpenClaw のターンを処理するものと同じ Codex スレッド内で、Codex app-server 自身のアプリおよび Plugin 機能を使用できます。Plugin 呼び出しはネイティブ Codex トランスクリプト内に保持され、アプリを利用する MCP の実行は Codex app-server が担います。OpenClaw は Codex Plugin を合成された `codex_plugin_*` OpenClaw 動的ツールに変換しません。

基本の [Codex ハーネス](/ja-JP/plugins/codex-harness)が動作した後に、このページを使用してください。

## 要件

- エージェントランタイムはネイティブ Codex ハーネスでなければなりません。
- `plugins.entries.codex.enabled` は `true` です。
- `plugins.entries.codex.config.codexPlugins.enabled` は `true` です。
- 対象の Codex app-server から、想定されるマーケットプレイス、Plugin、アプリのインベントリを確認できる必要があります。
- 移行でサポートされるのは、移行元の Codex ホームでソースからインストール済みであることが確認された `openai-curated` Plugin のみです。
- 手動で構成された `workspace-directory` Plugin には、`plugin/list` が `marketplaceKinds` を受け入れ、パスなしのワークスペース概要に `remotePluginId` が含まれる Codex app-server が必要です。Plugin はすでにインストールされ、有効になっていなければならず、その Plugin が所有するアプリには `app/list` でアクセスできる必要があります。

`codexPlugins` は OpenClaw プロバイダーの実行、ACP 会話バインディング、その他のハーネスには影響しません。これらのパスでは、ネイティブの `apps` 設定を持つ Codex app-server スレッドが作成されないためです。

OpenAI 側の Codex アカウント、アプリの利用可否、ワークスペースのアプリ／Plugin 制御は、サインイン中の Codex アカウントに基づきます。OpenAI アカウントと管理モデルについては、[ChatGPT プランで Codex を使用する](https://help.openai.com/en/articles/11369540-using-codex-with-your-chatgpt-plan)を参照してください。

## クイックスタート

移行元の Codex ホームからの移行をプレビューします。

```bash
openclaw migrate codex --dry-run
```

`--verify-plugin-apps` を追加すると、移行時に移行元の `app/list` が呼び出され、ネイティブ有効化を計画する前に、その Plugin が所有するすべてのアプリが存在し、有効で、アクセス可能であることが必須になります。

```bash
openclaw migrate codex --dry-run --verify-plugin-apps
```

計画に問題がなければ、移行を適用します。

```bash
openclaw migrate apply codex --yes
```

移行では、対象となる Plugin 用の明示的な `codexPlugins` エントリが書き込まれ、選択した Plugin に対して Codex app-server の `plugin/install` が呼び出されます。移行後の設定は次のようになります。

```json5
{
  plugins: {
    entries: {
      codex: {
        enabled: true,
        config: {
          codexPlugins: {
            enabled: true,
            allow_destructive_actions: true,
            plugins: {
              "google-calendar": {
                enabled: true,
                marketplaceName: "openai-curated",
                pluginName: "google-calendar",
              },
            },
          },
        },
      },
    },
  },
}
```

移行の対象は引き続き `openai-curated` に限定されます。既存の `workspace-directory` Plugin を使用するには、`plugin/list` が返す、マーケットプレイスで完全修飾された正確な `summary.id` を使用して手動で追加してください。たとえば、Codex が `example-plugin@workspace-directory` を返す場合は、表示名ではなく、その完全な値を設定します。

```json5
{
  plugins: {
    entries: {
      codex: {
        enabled: true,
        config: {
          codexPlugins: {
            enabled: true,
            plugins: {
              "example-plugin": {
                enabled: true,
                marketplaceName: "workspace-directory",
                pluginName: "example-plugin@workspace-directory",
              },
            },
          },
        },
      },
    },
  },
}
```

OpenClaw は `workspace-directory` Plugin に対して `plugin/install` を呼び出したり、認証を開始したりしません。OpenClaw ポリシーを追加または有効化する前に、Codex でインストール、有効化、認証を行ってください。レスポンスに正確なマーケットプレイス、Plugin ID、詳細 ID、またはアプリ準備状況の証拠が含まれていない場合、OpenClaw はアプリを非表示のままにします。Codex が明示的なワークスペースの `plugin/list` リクエストを拒否した場合、OpenClaw は有効な各ワークスペース Plugin について `marketplace_missing` を報告し、独立して検出された curated Plugin は引き続き利用可能にします。

`codexPlugins` を変更すると、新しい Codex 会話には更新されたアプリセットが自動的に適用されます。現在の会話を更新するには、`/new` または `/reset` を実行してください。Plugin の有効化／無効化の変更に Gateway の再起動は必要ありません。

## チャットから Plugin を管理する

`/codex plugins` を使用すると、Codex ハーネスを操作している同じチャットから、設定済みのネイティブ Codex Plugin を確認または変更できます。

```text
/codex plugins
/codex plugins list
/codex plugins disable google-calendar
/codex plugins enable google-calendar
```

`/codex plugins` は `/codex plugins list` の別名です。リストには、`plugins.entries.codex.config.codexPlugins.plugins` にある各設定済み Plugin のキー、オン／オフ状態、Codex Plugin 名、マーケットプレイスが表示されます。

`enable`/`disable` が書き込むのは `~/.openclaw/openclaw.json` のみです。`~/.codex/config.toml` を編集したり、新しい Codex Plugin をインストールしたりすることはありません。これらを実行できるのは、所有者または `operator.admin` スコープを持つ Gateway クライアントのみです。

設定済み Plugin を有効にすると、グローバルな `codexPlugins.enabled` スイッチもオンになります。移行で `auth_required` が返されたため curated Plugin が無効として書き込まれた場合は、OpenClaw で有効にする前に Codex でアプリを再認可してください。`workspace-directory` エントリの場合、ここで有効にしても変更されるのは OpenClaw ポリシーのみです。Plugin とアプリは Codex ですでに有効になっている必要があります。

## ネイティブ Plugin のセットアップの仕組み

この統合では、次の 3 つの状態を追跡します。

| 状態       | 意味                                                                                                                                 |
| ---------- | ---------------------------------------------------------------------------------------------------------------------------------- |
| インストール済み | Codex の対象 app-server ランタイムに Plugin バンドルが存在します。                                                                      |
| 有効       | Codex が Plugin を有効と報告し、OpenClaw 設定が Codex ハーネスのターンでその使用を許可しています。                                           |
| アクセス可能 | Codex app-server が、Plugin のアプリエントリをアクティブなアカウントで利用でき、設定済みの Plugin ID に対応していることを確認しています。 |

`openai-curated` Plugin の場合、移行が永続的なインストール／対象判定のステップになります。

- 計画時に、OpenClaw は移行元 Codex の `plugin/read` の詳細を読み取り、移行元 Codex app-server のアカウントが ChatGPT サブスクリプションアカウントであることを確認します。ChatGPT 以外のアカウント、またはアカウントのレスポンスがない場合、アプリを利用する Plugin は `codex_subscription_required` としてスキップされます。
- デフォルトでは、移行時に移行元の `app/list` 呼び出しはスキップされます。アカウント条件を満たす、アプリを利用する移行元 Plugin は、移行元アプリへのアクセス可否を検証せずに計画され、アカウント検索の通信エラーが発生した場合は `codex_account_unavailable` としてスキップされます。
- `--verify-plugin-apps` を使用すると、移行時に移行元の新しい `app/list` スナップショットが取得され、ネイティブ有効化を計画する前に、その Plugin が所有するすべてのアプリが存在し、有効で、アクセス可能であることが必須になります。この場合、アカウント検索の通信エラーは即座にスキップされず、移行元アプリのインベントリ条件による判定に進みます。

`workspace-directory` Plugin の場合、セットアップは OpenClaw の外部で行われます。OpenClaw は、有効なワークスペースエントリが少なくとも 1 つ設定されている場合にのみ、そのマーケットプレイスを照会し、正確な `summary.id` によって各 Plugin を解決し、既存の `plugin/read` 所有権チェックと `app/list` 準備状況チェックを再利用します。インストールされていない、無効、アクセス不能、または未認証の Plugin からアプリが公開されることはありません。OpenClaw はインストールや認証を試行しません。

ランタイムのアプリインベントリは、移行された curated Plugin と手動設定されたワークスペース Plugin の両方に対する、対象セッションでのアクセス可否チェックです。Codex ハーネスのセッションセットアップでは、有効かつアクセス可能な Plugin アプリから制限的なスレッドアプリ設定を計算します。この設定はターンごとには再計算されないため、`/codex plugins enable`/`disable` が影響するのは新しい Codex 会話のみです。現在の会話に変更を適用するには、`/new` または `/reset` を使用してください。

## V1 のサポート範囲

- 移行対象となるのは、移行元 Codex app-server のインベントリにすでにインストールされている `openai-curated` Plugin のみです。
- ランタイムでは、`plugin/list` が `marketplaceKinds` を実装し、パスなしのワークスペース概要に対して `remotePluginId` を返す app-server ビルド上の明示的な `workspace-directory` エントリもサポートされます。これらのエントリでは、マーケットプレイスで完全修飾された正確な `summary.id` を使用する必要があり、すでにインストールされ、有効で、アプリにアクセス可能でなければなりません。ワークスペース一覧リクエストが拒否されると、Plugin ごとに既存の `marketplace_missing` 診断が生成されます。マーケットプレイス、Plugin、詳細、またはアプリの証拠がない場合、ワークスペースアプリは公開されません。デフォルトの一覧リクエストから得られる curated インベントリは引き続き使用できます。
- アプリを利用する移行元 Plugin は、移行時のサブスクリプション条件を満たす必要があります。`--verify-plugin-apps` を追加すると、移行元アプリのインベントリ条件も適用されます。サブスクリプション条件を満たさないアカウント、および検証モードでアクセス不能、無効、または欠落している移行元アプリや、アプリインベントリの更新エラーは、有効な設定エントリではなく、スキップされた手動項目として報告されます。読み取れない Plugin の詳細は、アプリインベントリ条件の判定前にスキップされます。
- 移行では明示的な Plugin ID（`marketplaceName` および `pluginName`）が書き込まれます。ローカルの `marketplacePath` キャッシュパスは書き込まれません。
- `codexPlugins.enabled` が唯一のグローバル有効化スイッチです。任意のインストール権限を付与する `plugins["*"]` ワイルドカードや設定キーはありません。
- curated 以外のマーケットプレイス、キャッシュ済み Plugin バンドル、フック、Codex 設定ファイルは、自動的に有効化されず、手動確認用として移行レポートに保持されます。ランタイムは手動設定された `workspace-directory` エントリを受け入れます。その他のマーケットプレイスはサポートされません。

## アプリインベントリと所有権

OpenClaw は app-server の `app/list` を通じて Codex のアプリインベントリを読み取り、メモリ内に 1 時間キャッシュし、古いまたは存在しないエントリを非同期で更新します。キャッシュはプロセスローカルです。CLI または Gateway を再起動すると破棄され、次回の `app/list` 読み取り時に OpenClaw が再構築します。

移行とランタイムでは、別々のキャッシュキーを使用します。

- 移行元の移行検証では、移行元の Codex ホームと起動オプションが使用されます。これは `--verify-plugin-apps` を指定した場合にのみ実行され、その計画実行に対して移行元の `app/list` の新しい走査を強制します。
- 対象ランタイムのセットアップでは、スレッドアプリ設定を構築するときに、対象エージェントの Codex app-server ID が使用されます。curated Plugin を有効化すると、その対象キャッシュキーが無効化され、`plugin/install` の後に強制更新されます。`workspace-directory` のセットアップでは、この有効化パスは実行されません。

Plugin アプリが公開されるのは、安定した所有権を介して OpenClaw が設定済み Plugin に対応付けられる場合のみです。対応付けには、Plugin の詳細にある正確なアプリ ID、既知の MCP サーバー名、または一意で安定したメタデータが使用されます。表示名だけに基づく所有権や曖昧な所有権は、次回のインベントリ更新で所有権が確認されるまで除外されます。

## 接続済みアカウントのアプリ

所有者が運用するエージェントは、一致する Plugin パッケージを必要とせず、Codex アカウントにすでに接続されているすべてのアプリの使用を選択できます。

```json5
{
  plugins: {
    entries: {
      codex: {
        enabled: true,
        config: {
          codexPlugins: {
            enabled: true,
            allow_all_plugins: true,
            allow_destructive_actions: "auto",
          },
        },
      },
    },
  },
}
```

`allow_all_plugins: true` は、新しいネイティブ Codex スレッドが確立されたときに完全な `app/list` スナップショットを取得し、そのアカウントでアクセス可能と記録されたアプリのみを許可します。アプリをグローバルにインストール、認証、または有効化することはありません。既存のスレッドでは、永続化されたアプリセットが維持されます。新たに接続または取り消されたアプリを反映するには、`/new`、`/reset` を使用するか、Gateway を再起動してください。

アカウントアプリはグローバルな `codexPlugins.allow_destructive_actions` 値を継承します。
この値には `true`、`false`、`"auto"`、または `"ask"` を指定できます。Plugin ごとの明示的なポリシーは、
重複するアプリ ID に対してグローバルポリシーを上書きします。インベントリの取得に失敗した場合は、
無制限のデフォルトへフォールバックせず、閉じた状態で失敗します。

## スレッドアプリの設定

OpenClaw は Codex スレッドに制限的な `config.apps` パッチを注入します。
`_default` は無効になり、有効化された設定済み Plugin が所有するアプリ、または
`allow_all_plugins` によって許可されたアクセス可能なアカウントアプリのみが有効になります。

各アプリの `destructive_enabled` は、有効なグローバルまたは
Plugin ごとの `allow_destructive_actions` ポリシーから取得されます。`true`、`"auto"`、`"ask"` は
いずれも `destructive_enabled: true` を設定し、`false` はこれを `false` に設定します。Codex は引き続き、
ネイティブアプリツールのアノテーションに含まれる破壊的ツールのメタデータを適用します。
`_default` は `open_world_enabled: false` によって無効になり、有効な Plugin アプリには
`open_world_enabled: true` が設定されます。OpenClaw は、Plugin レベルの独立した
オープンワールドポリシー設定を公開せず、Plugin ごとの
破壊的なツール名の拒否リストも維持しません。

許可されたアプリでは、ツール承認モードのデフォルトは自動であるため、非破壊的な
読み取りツールは同じスレッドで承認を求めずに実行されます。破壊的ツールは引き続き、
各アプリの `destructive_enabled` ポリシーによって制御されます。

## 破壊的アクションのポリシー

設定済みの Codex Plugin では、破壊的な Plugin elicitation はデフォルトで許可されますが、
安全でないスキーマや所有者が曖昧な場合は閉じた状態で失敗します。

- グローバルな `allow_destructive_actions` のデフォルトは `true` です。
- Plugin ごとの `allow_destructive_actions` は、その Plugin に対してグローバルポリシーを
  上書きします。
- `false`: OpenClaw は決定論的な拒否応答を返します。
- `true`: OpenClaw は、ブール値の承認フィールドなど、承認応答にマッピングできる
  安全なスキーマのみを自動承認します。
- `"auto"`: OpenClaw は破壊的な Plugin アクションを Codex に公開し、その後、
  所有権が証明された MCP 承認 elicitation を OpenClaw の Plugin
  承認に変換してから、Codex の承認応答を返します。
- `"ask"`: OpenClaw は `"auto"` と同じ Codex の書き込み／破壊的操作ゲートを使用し、
  スレッドの開始前に、そのアプリに対する Codex の永続的なツール別承認オーバーライドを消去し、
  1 回限りの承認または拒否のみを提示します。これにより、
  永続的な承認によって以後の書き込みアクションのプロンプトが抑制されることを防ぎます。
  `"ask"` を使用する許可済みアプリごとに、OpenClaw はそのアプリに対して
  Codex の人間による承認レビュアーを選択し、Codex が承認 elicitation を
  OpenClaw に送信するようにします。その他のアプリおよびアプリ以外のスレッド承認では、
  設定済みのレビュアーとポリシーが維持されます。
- Plugin の識別情報がない場合、所有権が曖昧な場合、ターン ID がないか一致しない場合、
  または elicitation スキーマが安全でない場合は、プロンプトを表示せず拒否します。

## トラブルシューティング

| コード                                              | 意味                                                                                                                              | 修正方法                                                                                                                    |
| ------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------ | ---------------------------------------------------------------------------------------------------------------------- |
| `auth_required`                                   | 移行によって Plugin はインストールされましたが、そのアプリのいずれかで引き続き認証が必要です。再認可するまで、エントリは無効な状態で書き込まれます。 | Codex でアプリを再認可してから、OpenClaw で Plugin を有効にします。                                                      |
| `app_inaccessible`、`app_disabled`、`app_missing` | `--verify-plugin-apps` を使用した場合、移行元の Codex アプリインベントリで、所有するすべてのアプリが存在し、有効で、アクセス可能であることを確認できませんでした。         | Codex でアプリを再認可または有効化してから、`--verify-plugin-apps` を指定して移行を再実行します。                              |
| `app_inventory_unavailable`                       | 移行元アプリの厳格な検証が要求されましたが、移行元の Codex アプリインベントリの更新に失敗しました。                                      | 移行元の Codex app-server へのアクセスを修正するか、`--verify-plugin-apps` を指定せずに再試行して、より高速なアカウントゲート方式のプランを受け入れます。   |
| `codex_subscription_required`                     | 移行元の Codex app-server アカウントは ChatGPT サブスクリプションアカウントではありませんでした。                                                          | サブスクリプション認証を使用して Codex アプリにログインしてから、移行を再実行します。                                                  |
| `codex_account_unavailable`                       | 移行元の Codex app-server アカウントを読み取れませんでした。                                                                               | 移行元の Codex app-server 認証を修正するか、`--verify-plugin-apps` を指定して再実行し、移行元のアプリインベントリに適格性を判定させます。 |
| `marketplace_missing`、`plugin_missing`           | マーケットプレイスまたは指定された Plugin を利用できません。明示的なワークスペースカタログ要求が拒否された可能性があります。ワークスペースアプリは閉じた状態で失敗します。  | 以下に記載されている互換性のある app-server コントラクトと正確な ID を確認します。                                                |
| `plugin_detail_unavailable`                       | OpenClaw は Plugin の所有権の詳細を読み取れませんでした。                                                                                    | 対象 app-server の `plugin/list` および `plugin/read` 応答を確認します。                                             |
| `plugin_disabled`                                 | Codex は、Plugin がインストール済みだが無効であると報告しています。                                                                                     | キュレーション済みアクティベーションによって修復できる場合があります。再試行する前に、Codex でワークスペース Plugin を有効にします。                                  |
| `plugin_activation_failed`                        | Plugin のアクティベーションが完了しませんでした。                                                                                                  | 添付された診断情報を使用して、マーケットプレイス、認証、更新、またはワークスペース準備状態の失敗を切り分けます。                |
| `app_inventory_missing`、`app_inventory_stale`    | アプリの準備状態が、空または古いキャッシュから取得されました。                                                                                     | OpenClaw は非同期更新を自動的にスケジュールします。所有権と準備状態が判明するまで、Plugin アプリは除外されたままです。  |
| `app_ownership_ambiguous`                         | アプリインベントリは表示名のみで一致しました。                                                                                          | 後続の更新で所有権が証明されるまで、そのアプリは Codex スレッドで非表示のままです。                                     |

**ワークスペース Plugin はインストール済みだが表示されない場合:** ワークスペースの
`plugin/list` の結果で、設定された正確な ID がインストール済みかつ有効として報告されていることを確認し、
次に `app/list` で、同じ Codex アカウントから所有するすべてのアプリにアクセスできると報告されていることを確認します。
アカウントインベントリでそのアプリが現在無効と報告されている場合でも、OpenClaw は
アクセス可能なアプリをスレッドで有効にできます。Gateway がアプリインベントリをキャッシュした後に
その状態を変更した場合は、1 時間のキャッシュ更新を待つか Gateway を再起動してから、
`/new` または `/reset` を使用します。OpenClaw はワークスペース Plugin の修復や認証を行いません。
明示的なワークスペース一覧要求が拒否された場合、有効な各ワークスペース
エントリは `marketplace_missing` を報告します。無関係なキュレーション済みエントリは、
デフォルトの一覧応答から引き続き処理されます。

`plugin_detail_unavailable` の場合、パスのないワークスペース概要には
`remotePluginId` が含まれている必要があります。そのセレクターまたは後続の
`plugin/read` の結果を利用できない場合、OpenClaw は所有するアプリを非表示のままにします。
`plugin_activation_failed` の場合、キュレーション済み Plugin では、マーケットプレイス、認証、または
インストール後の更新の失敗が報告されることがあります。ワークスペース Plugin が
まだアクティブでない場合、このコードが報告されます。OpenClaw の外部でインストール、有効化、認証を行ってください。

**設定を変更したがエージェントから Plugin が見えない場合:** `/codex plugins
list` を実行して設定状態を確認し、次に `/new` または `/reset` を実行します。既存の
Codex スレッドバインディングは、OpenClaw が新しいハーネスセッションを確立するか、
古いバインディングを置き換えるまで、開始時のアプリ設定を維持します。

**破壊的アクションが拒否される場合:** グローバルおよび Plugin ごとの
`allow_destructive_actions` の値を確認します。`true`、`"auto"`、または `"ask"` を指定していても、
安全でない elicitation スキーマや曖昧な Plugin 識別情報は引き続き閉じた状態で失敗します。

## 関連項目

- [Codex ハーネス](/ja-JP/plugins/codex-harness)
- [Codex ハーネスリファレンス](/ja-JP/plugins/codex-harness-reference)
- [Codex ハーネスランタイム](/ja-JP/plugins/codex-harness-runtime)
- [設定リファレンス](/ja-JP/gateway/configuration-reference#codex-harness-plugin-config)
- [移行 CLI](/ja-JP/cli/migrate)
