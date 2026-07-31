---
read_when:
    - Codex モードの OpenClaw エージェントで Codex Computer Use を使用する場合
    - Codex Computer Use、PeekabooBridge、cua-driver MCP の直接利用のどれを選ぶか検討しています
    - バンドルされた Codex Plugin 向けに computerUse を設定しています
    - /codex のコンピューター操作のステータスまたはインストールに関するトラブルシューティングを行っています
summary: Codex モードの OpenClaw エージェント向けに Codex Computer Use をセットアップする
title: Codex のコンピューター操作
x-i18n:
    generated_at: "2026-07-26T10:09:28Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 6b11d00c74bc2990a4e33b6ffe23209ed76a1e10180ce5950dbb5073ea57ad05
    source_path: plugins/codex-computer-use.md
    workflow: 16
---

Computer Use は、ローカルデスクトップ制御用の Codex ネイティブ MCP plugin です。OpenClaw
はデスクトップアプリを同梱せず、デスクトップ操作を自ら実行することも、
Codex の権限を回避することもありません。同梱の `codex` plugin は Codex app-server の準備のみを行います。
Codex plugin のサポートを有効にし、設定された Computer Use
plugin を検索またはインストールし、`computer-use` MCP server が利用可能であることを確認した後、
Codex モードのターン中は Codex にネイティブ MCP ツール呼び出しを管理させます。

OpenClaw がすでにネイティブ Codex ハーネスを使用している場合は、このページを使用してください。
ランタイム自体のセットアップについては、[Codex ハーネス](/ja-JP/plugins/codex-harness)を参照してください。

これは、OpenClaw に組み込まれた [Node ベースのコンピューターツール](/ja-JP/nodes/computer-use)とは異なります。エージェントが Gateway と別の Node のどちらで実行される場合でも、同じエージェント契約でペアリング済みの Mac を制御する必要がある場合は、組み込みツールを使用してください。Codex app-server がローカル MCP のインストール、権限、ネイティブツール呼び出しを管理する必要がある場合は、Codex Computer Use を使用してください。

## OpenClaw.app と Peekaboo

OpenClaw.app の Peekaboo 統合は、Codex Computer Use とは別のものです。
macOS アプリは PeekabooBridge ソケットをホストできるため、`peekaboo` CLI は
Peekaboo 独自の自動化ツール用に、アプリのローカルのアクセシビリティと画面収録の許可を再利用できます。
このブリッジは Codex Computer Use のインストールやプロキシを行わず、
Codex Computer Use も PeekabooBridge ソケットを介して呼び出しません。

OpenClaw.app を Peekaboo CLI 自動化の権限対応ホストとして使用する場合は、
[Peekaboo ブリッジ](/ja-JP/platforms/mac/peekaboo)を使用してください。Codex モードの OpenClaw エージェントが
ターン開始前に Codex ネイティブの `computer-use` MCP plugin を
利用できるようにする場合は、このページを使用してください。

## iOS アプリ

iOS アプリは Codex Computer Use とは別のものです。Codex の
`computer-use` MCP server をインストールまたはプロキシせず、デスクトップ制御のバックエンドでもありません。
代わりに、iOS アプリは OpenClaw Node として接続し、
`canvas.*`、`camera.*`、`screen.*`、
`location.*`、`talk.*` などの Node コマンドを通じてモバイル機能を公開します。

Gateway を通じてエージェントに iPhone Node を操作させる場合は、
[iOS](/ja-JP/platforms/ios)を使用してください。Codex モードのエージェントが Codex ネイティブの Computer Use plugin を介して
ローカルの macOS デスクトップを制御する必要がある場合は、このページを使用してください。

## cua-driver MCP への直接接続

Codex Computer Use は、デスクトップ制御を公開する唯一の方法ではありません。
OpenClaw が管理するランタイムから TryCua のドライバーを直接呼び出す場合は、
Codex 固有のマーケットプレイスフローではなく、OpenClaw の MCP レジストリを介してアップストリームの
`cua-driver mcp` server を使用してください。

`cua-driver` をインストールした後、OpenClaw 用コマンドを出力させるには、次を実行します。

```bash
cua-driver mcp-config --client openclaw
```

または、stdio server を直接登録します。

```bash
openclaw mcp set cua-driver '{"command":"cua-driver","args":["mcp"]}'
```

この方法では、ドライバースキーマや構造化された MCP レスポンスを含む、
アップストリームの MCP ツールサーフェスがそのまま維持されます。CUA ドライバーを
通常の OpenClaw MCP server として利用する場合に使用してください。Codex app-server が
Codex モードのターン内で plugin のインストール、MCP の再読み込み、
ネイティブツール呼び出しを管理する必要がある場合は、このページの Codex Computer Use セットアップを使用してください。

CUA のドライバーは、macOS、Windows（x64 および ARM64）、
Linux（x64 および ARM64、プレビュー段階）向けのプレリリースビルドを提供しています。引き続き、
macOS のアクセシビリティや画面収録など、アプリが要求するローカル OS の権限が必要です。
OpenClaw は `cua-driver` のインストール、それらの権限の付与、
アップストリームドライバーの安全モデルの回避を行いません。

## クイックセットアップ

スレッド開始前に Codex モードのターンで Computer Use を利用可能にする必要がある場合は、
`plugins.entries.codex.config.computerUse` を設定します。`autoInstall: true` は
Computer Use を有効化し、ターン前に OpenClaw がインストールまたは再有効化できるようにします。

```json5
{
  plugins: {
    entries: {
      codex: {
        enabled: true,
        config: {
          computerUse: {
            autoInstall: true,
          },
        },
      },
    },
  },
  agents: {
    defaults: {
      model: "openai/gpt-5.6-sol",
    },
  },
}
```

この設定では、OpenClaw は Codex モードの各ターン前に Codex app-server を確認します。
Computer Use が見つからなくても、Codex app-server がインストール可能なマーケットプレイスをすでに検出している場合、
OpenClaw は Codex app-server に plugin のインストールまたは再有効化と、
MCP server の再読み込みを要求します。macOS で隔離された Codex app-server を起動する前に、
自動インストールは、ネイティブクライアントが見つからない場合、選択したデスクトップアプリバンドルから
公式に署名された Computer Use サービスアプリを、その Codex ホームの
`computer-use` ディレクトリにもコピーします。
macOS では、一致するマーケットプレイスが登録されておらず、標準のデスクトップアプリバンドルが存在する場合、
OpenClaw は `/Applications/ChatGPT.app/Contents/Resources/plugins/openai-bundled` から
同梱の Codex マーケットプレイスを登録することも試み、
従来のスタンドアロンインストール用のフォールバックとして
`/Applications/Codex.app/Contents/Resources/plugins/openai-bundled` を維持します。それでもセットアップで
MCP server を利用可能にできない場合、スレッド開始前にターンは失敗します。
厳密な準備状態の失敗はハーネスの事前チェック失敗として扱われるため、モデルのフォールバックで
モデル候補ごとに同じローカル準備シーケンスが繰り返されることはありません。

既存の Codex スレッドがすでに開始されている場合、Computer Use の設定変更後にテストする前に、
対象のチャットで `/new` または `/reset` を使用してください。

macOS では、Computer Use の管理対象起動は
`/Applications/ChatGPT.app/Contents/Resources/codex` にあるデスクトップアプリのバイナリを優先し、その後、
従来のスタンドアロンインストール用の `/Applications/Codex.app/Contents/Resources/codex` に
フォールバックします。これは、独自のクライアントを起動する単発の Computer Use ステータスおよび
インストールコマンドにも適用されます。これにより、デスクトップ制御は
ローカルの macOS 権限を所有するアプリバンドルの配下に維持されます。デスクトップアプリが
インストールされていない場合、OpenClaw は plugin の横にインストールされた管理対象 Codex バイナリに
フォールバックします。デフォルトの隔離されたエージェントホームを使用する通常の管理対象 Codex ターンでは、
古いデスクトップアプリが現在のモデルサポートを隠さないよう、その固定パッケージが最初に優先されます。
ユーザースコープのホームはネイティブの Computer Use 状態を読み込めるため、引き続きデスクトップ優先です。
有効な Codex 設定で Computer Use が有効になっている隔離されたエージェントホームも、
デスクトップ優先のままです。明示的な
`appServer.command` 設定または `OPENCLAW_CODEX_APP_SERVER_BIN` は、引き続き
この管理対象の選択を上書きします。

OpenClaw は、実行中の 1 つの Gateway 内で、ネイティブ Codex 設定の読み取りと Computer Use のインストールを
直列化します。別の Codex プロセスや別の Gateway は、その排他制御の対象外です。
Gateway の外部でネイティブ Codex plugin の設定を変更した後は、新しい選択に依存する前に
Gateway を再起動し、新しいチャットを開始してください。

## コマンド

`codex` plugin のコマンドサーフェスを利用できる任意のチャット画面から、
`/codex computer-use` コマンドを使用します。これらは OpenClaw のチャット／ランタイムコマンドであり、
`openclaw codex ...` CLI のサブコマンドではありません。

```text
/codex computer-use status
/codex computer-use install
/codex computer-use install --source <marketplace-source>
/codex computer-use install --marketplace-path <path>
/codex computer-use install --marketplace <name>
```

`status` はデフォルトのアクションであり、読み取り専用です。マーケットプレイスの
ソース追加、plugin のインストール、Codex plugin サポートの有効化は行いません。Computer Use を
有効化する設定がない場合、単発のインストールコマンドを実行した後でも
`status` が無効と報告することがあります。

`install` は Codex app-server の plugin サポートを有効にし、必要に応じて
設定済みのマーケットプレイスソースを追加し、Codex app-server を介して設定済み plugin を
インストールまたは再有効化し、MCP server を再読み込みし、MCP
server がツールを公開していることを検証します。インストールでは信頼済みホストのリソースが変更されるため、
`install` を実行できるのは、所有者または `operator.admin` Gateway クライアントのみです。
その他の許可された送信者も、オーバーライドを含め、読み取り専用の
`status` コマンドを引き続き使用できます。

以前のリリースでは、単発の `--plugin`、`--server`、`--mcp-server`
ID オーバーライドが受け入れられていました。代わりに `computerUse.pluginName` と
`computerUse.mcpServerName` を永続的に設定してください。従来の ID フラグを
使用すると、コマンドは永続化すべき設定を正確に示し、要求されたアクションと
サポートされているマーケットプレイスフラグを移行案内内で繰り返します。

## マーケットプレイスの選択肢

OpenClaw は Codex 自体が公開しているものと同じ app-server API を使用します。
マーケットプレイスフィールドでは、Codex が `computer-use` を検索する場所を選択します。

| フィールド                | 使用する場合                                                        | インストール対応                                          |
| -------------------- | --------------------------------------------------------------- | -------------------------------------------------------- |
| マーケットプレイスフィールドなし | Codex app-server に既知のマーケットプレイスを使用させる場合。 | app-server がローカルマーケットプレイスを返す場合は対応。        |
| `marketplaceSource`  | app-server が追加できる Codex マーケットプレイスソースがある場合。         | 明示的な `/codex computer-use install` に対応。         |
| `marketplacePath`    | ホスト上のローカルマーケットプレイスファイルパスがすでに判明している場合。   | 明示的なインストールとターン開始時の自動インストールに対応。   |
| `marketplaceName`    | 登録済みのマーケットプレイスを名前で 1 つ選択する場合。  | 選択したマーケットプレイスにローカルパスがある場合のみ対応。 |

新しい Codex ホームでは、公式マーケットプレイスの初期登録に少し時間が必要な場合があります。
インストール中、OpenClaw は最大 `marketplaceDiscoveryTimeoutMs` ミリ秒（デフォルト 60 秒）、
`plugin/list` をポーリングします。

既知の複数のマーケットプレイスに Computer Use が含まれる場合、OpenClaw は
`openai-bundled`、次に `openai-curated`、次に `local` の順で優先します。
不明で曖昧な一致の場合は安全側に倒して失敗し、`marketplaceName` または
`marketplacePath` を設定するよう求めます。

## 同梱の macOS マーケットプレイス

現在の ChatGPT デスクトップビルドでは、Computer Use は次の場所に同梱されています。従来のスタンドアロン
Codex デスクトップビルドでは、`Codex.app` 配下の同じレイアウトを使用します。

```text
/Applications/ChatGPT.app/Contents/Resources/plugins/openai-bundled/plugins/computer-use
/Applications/Codex.app/Contents/Resources/plugins/openai-bundled/plugins/computer-use
```

`computerUse.autoInstall` が true で、`computer-use` を含むマーケットプレイスが
登録されていない場合、OpenClaw は存在する最初の標準的な同梱マーケットプレイスルートを
追加しようとします。

```text
/Applications/ChatGPT.app/Contents/Resources/plugins/openai-bundled
/Applications/Codex.app/Contents/Resources/plugins/openai-bundled
```

Codex を使用してシェルから明示的に登録することもできます。

```bash
codex plugin marketplace add /Applications/ChatGPT.app/Contents/Resources/plugins/openai-bundled
```

標準以外の Codex アプリパスを使用する場合は、`/codex computer-use install
--source <marketplace-root>` を 1 回実行するか、
`computerUse.marketplacePath` をローカルマーケットプレイスのファイルパスに設定します。
`--marketplace-path` は、同梱のマーケットプレイスルートではなく、
マーケットプレイス JSON ファイルのパスがある場合にのみ使用してください。

### 共有 plugin キャッシュ

デフォルトの `pluginCacheMode: "independent"` では、各 Codex ホームとその
plugin キャッシュは管理されません。app-server の起動前に、同梱の Computer Use plugin を
アクティブな Codex ホームで検出可能な plugin キャッシュにコピーするには、
`pluginCacheMode: "shared"` を設定します。共有モードでは、実行中の Codex クライアントが
バージョン付き plugin ディレクトリを引き続き参照できるため、古いキャッシュバージョンが保持されます。
置換コピーに失敗した場合も、アクティブなキャッシュが保持されます。明示的な
`marketplaceName` または `marketplacePath` 設定は、この
調整処理を無効にするため、OpenClaw はその選択を上書きしません。

## リモートカタログの制限

Codex app-server はリモート専用のカタログエントリを一覧表示して読み取れますが、
現在はリモートの `plugin/install` をサポートしていません。つまり、
`marketplaceName` はステータス確認用にリモート専用マーケットプレイスを選択できますが、
インストールと再有効化には、引き続き `marketplaceSource` または
`marketplacePath` を介したローカルマーケットプレイスが必要です。

ステータスで、plugin はリモートの Codex マーケットプレイスで利用可能だが
リモートインストールがサポートされていないと表示された場合は、ローカルソースまたはパスを指定して
インストールを実行します。

```text
/codex computer-use install --source <marketplace-source>
/codex computer-use install --marketplace-path <path>
```

## 設定リファレンス

| フィールド                           | デフォルト        | 意味                                                                        |
| ------------------------------- | -------------- | ------------------------------------------------------------------------------ |
| `enabled`                       | 推論値       | Computer Use を必須にします。別の Computer Use フィールドが設定されている場合、デフォルトは true です。 |
| `autoInstall`                   | false          | ターン開始時にネイティブクライアントをプロビジョニングし、Plugin をインストールまたは再有効化します。 |
| `marketplaceDiscoveryTimeoutMs` | 60000          | Codex app-server のマーケットプレイス検出をインストールが待機する時間です。             |
| `liveTestTimeoutMs`             | 60000          | 一時的な準備完了確認スレッドとそのクリーンアップリクエストのタイムアウトです。           |
| `toolCallTimeoutMs`             | 60000          | Computer Use の `list_apps` 準備完了確認ツール呼び出しのタイムアウトです。                  |
| `healthCheckEnabled`            | false          | 所有元の app-server クライアントがアクティブな間、準備完了プローブを定期的に実行します。    |
| `healthCheckIntervalMinutes`    | 60             | プローブ間隔です。指定できる値は 30、60、120、または 240 分です。                |
| `pluginCacheMode`               | `independent`  | `shared` を使用して、バンドルされたデスクトップ Plugin から Codex ホームのキャッシュを更新します。  |
| `strictReadiness`               | false          | ライブプローブに失敗した場合、警告を表示して続行せず、起動を停止します。      |
| `autoRepair`                    | false          | スコープ内の古い Computer Use MCP 子プロセスを終了し、失敗したプローブを 1 回再試行します。     |
| `marketplaceSource`             | 未設定          | Codex app-server の `marketplace/add` に渡すソース文字列です。                    |
| `marketplacePath`               | 未設定          | Plugin を含むローカル Codex マーケットプレイスのファイルパスです。                       |
| `marketplaceName`               | 未設定          | 選択する登録済み Codex マーケットプレイス名です。                                   |
| `pluginName`                    | `computer-use` | Codex マーケットプレイスの Plugin 名です。                                                 |
| `mcpServerName`                 | `computer-use` | インストール済み Plugin が公開する MCP サーバー名です。                               |

ターン開始時の自動インストールは、設定された `marketplaceSource`
の値を意図的に拒否します。新しいソースの追加は明示的なセットアップ操作であるため、
`/codex computer-use install --source <marketplace-source>` を一度使用し、その後は
`autoInstall` に、検出されたローカルマーケットプレイスからの再有効化を処理させます。
設定済みの `marketplacePath` はホスト上のローカルパスであるため、
ターン開始時の自動インストールで使用できます。

各フィールドでは、対応する設定キーが未設定の場合に確認される
環境変数による上書きも使用できます。

| フィールド                           | 環境変数                                                        |
| ------------------------------- | -------------------------------------------------------------- |
| `enabled`                       | `OPENCLAW_CODEX_COMPUTER_USE`                                  |
| `autoInstall`                   | `OPENCLAW_CODEX_COMPUTER_USE_AUTO_INSTALL`                     |
| `marketplaceDiscoveryTimeoutMs` | `OPENCLAW_CODEX_COMPUTER_USE_MARKETPLACE_DISCOVERY_TIMEOUT_MS` |
| `liveTestTimeoutMs`             | `OPENCLAW_CODEX_COMPUTER_USE_LIVE_TEST_TIMEOUT_MS`             |
| `toolCallTimeoutMs`             | `OPENCLAW_CODEX_COMPUTER_USE_TOOL_CALL_TIMEOUT_MS`             |
| `healthCheckEnabled`            | `OPENCLAW_CODEX_COMPUTER_USE_HEALTH_CHECK_ENABLED`             |
| `healthCheckIntervalMinutes`    | `OPENCLAW_CODEX_COMPUTER_USE_HEALTH_CHECK_INTERVAL_MINUTES`    |
| `pluginCacheMode`               | `OPENCLAW_CODEX_COMPUTER_USE_PLUGIN_CACHE_MODE`                |
| `strictReadiness`               | `OPENCLAW_CODEX_COMPUTER_USE_STRICT_READINESS`                 |
| `autoRepair`                    | `OPENCLAW_CODEX_COMPUTER_USE_AUTO_REPAIR`                      |
| `marketplaceSource`             | `OPENCLAW_CODEX_COMPUTER_USE_MARKETPLACE_SOURCE`               |
| `marketplacePath`               | `OPENCLAW_CODEX_COMPUTER_USE_MARKETPLACE_PATH`                 |
| `marketplaceName`               | `OPENCLAW_CODEX_COMPUTER_USE_MARKETPLACE_NAME`                 |
| `pluginName`                    | `OPENCLAW_CODEX_COMPUTER_USE_PLUGIN_NAME`                      |
| `mcpServerName`                 | `OPENCLAW_CODEX_COMPUTER_USE_MCP_SERVER_NAME`                  |

## OpenClaw が確認する内容

OpenClaw は内部で安定したセットアップ理由を報告し、
ユーザー向けのステータスをチャット用に整形します。

| 理由                       | 意味                                                | 次の手順                                     |
| ---------------------------- | ------------------------------------------------------ | --------------------------------------------- |
| `disabled`                   | `computerUse.enabled` が false と評価されました。               | `enabled` または別の Computer Use フィールドを設定します。  |
| `marketplace_missing`        | 一致するマーケットプレイスがありませんでした。                 | ソース、パス、またはマーケットプレイス名を設定します。  |
| `plugin_not_installed`       | マーケットプレイスは存在しますが、Plugin がインストールされていません。   | インストールを実行するか、`autoInstall` を有効にします。          |
| `plugin_disabled`            | Plugin はインストールされていますが、Codex 設定で無効になっています。      | インストールを実行して再有効化します。                  |
| `remote_install_unsupported` | 選択したマーケットプレイスはリモート専用です。                   | `marketplaceSource` または `marketplacePath` を使用します。 |
| `mcp_missing`                | Plugin は有効ですが、MCP サーバーを利用できません。  | Codex Computer Use と OS の権限を確認します。  |
| `ready`                      | Plugin と MCP ツールを利用できます。                    | Codex モードのターンを開始します。                    |
| `check_failed`               | ステータス確認中に Codex app-server リクエストが失敗しました。 | app-server の接続とログを確認します。       |
| `auto_install_blocked`       | ターン開始時のセットアップでは、新しいソースを追加する必要があります。       | 先に明示的なインストールを実行します。                   |

チャット出力には、Plugin の状態、MCP サーバーの状態、マーケットプレイス、
利用可能な場合はツール、および失敗したセットアップ手順に固有のメッセージが含まれます。

## macOS の権限

この Codex 所有の Computer Use パスは macOS 上で実行されます。MCP サーバーが
アプリを検査または操作するには、事前にローカル OS の権限が必要になる場合があります。
（Windows および Linux の Node ホストでのクロスプラットフォームなデスクトップ操作については、
[cua-computer フルフィラー](/ja-JP/nodes/computer-use#windows-and-linux-experimental-via-cua-driver)を参照してください。）
Computer Use はインストール済みだが MCP サーバーを利用できないと OpenClaw が報告する場合は、
まず Codex 側の Computer Use セットアップを確認します。

- Codex app-server が、デスクトップ操作を実行するホストと同じホスト上で動作している。
- Computer Use Plugin が Codex 設定で有効になっている。
- Codex app-server の MCP ステータスに `computer-use` MCP サーバーが表示される。
- macOS がデスクトップ操作アプリに必要な権限を付与している。
- 現在のホストセッションから、操作対象のデスクトップにアクセスできる。

`computerUse.enabled` が true の場合、OpenClaw は意図的にフェイルクローズします。
設定で必須とされたネイティブデスクトップツールなしに、
Codex モードのターンが暗黙的に続行されることはありません。

## トラブルシューティング

**ステータスに未インストールと表示される。** `/codex computer-use install` を実行します。
マーケットプレイスが検出されない場合は、`--source` または `--marketplace-path` を渡します。

**ステータスにインストール済みだが無効と表示される。** `/codex computer-use install` を
再度実行します。Codex app-server のインストールにより、Plugin 設定が有効な状態で書き戻されます。

**ステータスにリモートインストールはサポートされていないと表示される。** ローカルマーケットプレイスの
ソースまたはパスを使用します。リモート専用のカタログエントリは確認できますが、
現在の app-server API を介してインストールすることはできません。

**ステータスに MCP サーバーを利用できないと表示される。** MCP サーバーを
再読み込みするため、インストールを一度再実行します。引き続き利用できない場合は、
Codex Computer Use アプリ、Codex app-server の MCP ステータス、または macOS の権限を修正します。

**ステータスまたはプローブが `computer-use.list_apps` でタイムアウトする。** Plugin と
MCP サーバーは存在しますが、ローカル Computer Use ブリッジが応答しませんでした。
Codex Computer Use を終了または再起動し、必要に応じて Codex Desktop を再起動してから、
新しい OpenClaw セッションで再試行します。ホストが以前、古い管理対象 Codex app-server を介して
Computer Use を実行していた場合は、デスクトップにバンドルされたマーケットプレイスから
インストール済み Plugin を更新します（スタンドアロンの Codex デスクトップインストールでは
`Codex.app` パスを使用します）。

```text
/codex computer-use install --source /Applications/ChatGPT.app/Contents/Resources/plugins/openai-bundled
```

**Computer Use ツールに `Native hook relay unavailable` と表示される。**
Codex ネイティブのツールフックは、ローカルブリッジまたは Gateway フォールバックを介して
アクティブな OpenClaw リレーに到達できませんでした。`/new` または
`/reset` を使用して新しい OpenClaw セッションを開始します。一度は動作しても、
その後のツール呼び出しで再び失敗する場合、`/new` は現在の試行のみを
クリアしています。古いスレッドとフック登録が削除されるよう Codex app-server または
OpenClaw Gateway を再起動してから、新しいセッションで再試行します。

**ターン開始時の自動インストールがソースを拒否する。** これは意図された動作です。
まず明示的な `/codex computer-use install --source
<marketplace-source>` でソースを追加すると、それ以降のターン開始時の自動インストールで
検出済みのローカルマーケットプレイスを使用できるようになります。

## 関連項目

- [Codex ハーネス](/ja-JP/plugins/codex-harness)
- [Peekaboo ブリッジ](/ja-JP/platforms/mac/peekaboo)
- [iOS アプリ](/ja-JP/platforms/ios)
