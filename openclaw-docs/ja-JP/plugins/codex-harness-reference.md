---
read_when:
    - Codex ハーネスのすべての設定フィールドが必要です
    - app-server のトランスポート、認証、検出、またはタイムアウトの動作を変更する場合
    - Codex ハーネスの起動、モデル検出、または環境分離をデバッグしている場合
summary: Codex ハーネスの設定、認証、検出、アプリサーバーに関するリファレンス
title: Codex ハーネスリファレンス
x-i18n:
    generated_at: "2026-07-26T10:21:30Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 149f065f5bef18d0f491c97facc4b5991afc3f7e1077abdc7a4b49f506eac3e0
    source_path: plugins/codex-harness-reference.md
    workflow: 16
---

このリファレンスでは、公式 `codex` plugin の詳細な設定について説明します。
セットアップとルーティングの判断については、
[Codex ハーネス](/ja-JP/plugins/codex-harness)から始めてください。

## Plugin 設定サーフェス

Codex ハーネスのすべての設定は `plugins.entries.codex.config` 配下にあります。

```json5
{
  plugins: {
    entries: {
      codex: {
        enabled: true,
        config: {
          discovery: {
            enabled: true,
            timeoutMs: 2500,
          },
          appServer: {
            mode: "guardian",
          },
        },
      },
    },
  },
}
```

最上位フィールド：

| フィールド                 | デフォルト               | 意味                                                                                                                                           |
| -------------------------- | ------------------------ | ---------------------------------------------------------------------------------------------------------------------------------------------- |
| `discovery`                | 有効                     | Codex app-server `model/list` のモデル検出設定。                                                                                         |
| `appServer`                | 管理対象 stdio app-server | トランスポート、コマンド、認証、承認、サンドボックス、タイムアウトの設定。通常のハーネスは、デフォルトでエージェントスコープの状態を使用します。 |
| `codexDynamicToolsLoading` | `"searchable"`           | OpenClaw の動的ツールを初期 Codex ツールコンテキストに直接配置するには、`"direct"` を使用します。                                       |
| `codexDynamicToolsExclude` | `[]`                     | Codex app-server のターンから除外する追加の OpenClaw 動的ツール名。                                                                             |
| `codexPlugins`             | 無効                     | 接続済みアカウントのアプリへのオプトインアクセスを含む、ネイティブ Codex plugin/app のサポート。[ネイティブ Codex plugin](/ja-JP/plugins/codex-native-plugins)を参照してください。 |
| `computerUse`              | 無効                     | Codex Computer Use のセットアップ。[Codex Computer Use](/ja-JP/plugins/codex-computer-use)を参照してください。                                      |
| `sessionCatalog`           | 有効                     | サイドバー向けのネイティブ Codex セッション検出。プロバイダーやハーネスを無効にせず検出のみを無効にするには、`enabled: false` を設定します。   |
| `supervision`              | 無効                     | エージェント向けのネイティブセッショントランスクリプトおよび書き込み制御ポリシー。[Codex の監督](/ja-JP/plugins/codex-supervision)を参照してください。 |

## 監督

ネイティブセッション検出では、デフォルトで Gateway
コンピューターおよびオプトインしたペアリング済み Node から、アーカイブされていない Codex セッションを一覧表示します。このカタログのみを無効にするには、次のように設定します。

```json5
{
  plugins: {
    entries: {
      codex: {
        enabled: true,
        config: {
          sessionCatalog: {
            enabled: false,
          },
        },
      },
    },
  },
}
```

`supervision` は、エージェント向けツールを個別に制御します。

| フィールド            | デフォルト                | 意味                                                                                                                                                                                                                                      |
| --------------------- | ------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `enabled`             | `false`                 | エージェント向け Codex 監督ツールを有効にします。これは、認証済みオペレーターのセッションカタログを制御しません。                                                                                                                          |
| `endpoints`           | 組み込みローカルエンドポイント | 維持されている Codex 監督エージェントおよびスタンドアロン MCP ツール向けの、互換性および高度なエンドポイントターゲット。人間向けカタログとブランチフローはこれらのターゲットを無視し、`appServer` から解決された監督用 App Server を使用します。 |
| `allowRawTranscripts` | `false`                 | 監督が有効な場合、自律エージェントまたはスタンドアロン MCP によるトランスクリプトの読み取りと、トランスクリプトから派生する一覧フィールドを許可します。`codex_threads` のメタデータのみの読み取りは引き続き利用できます。認証済み Control UI の継続操作は制御しません。 |
| `allowWriteControls`  | `false`                 | 監督が有効な場合、自律的な `codex_threads` のフォーク、名前変更、アーカイブ、アーカイブ解除の変更に加え、スタンドアロン MCP の送信、誘導、中断操作を許可します。その他のバインディング、ホスト、ステータス、確認チェックを回避するものではありません。 |

エンドポイントエントリでは、次のフィールドを使用できます。

| フィールド     | 適用対象      | 意味                                                                    |
| -------------- | ------------- | ----------------------------------------------------------------------- |
| `id`           | すべて        | 安定したエンドポイント ID。                                             |
| `label`        | すべて        | 任意の表示ラベル。                                                       |
| `transport`    | すべて        | `"stdio-proxy"` または `"websocket"`。                          |
| `command`      | `stdio-proxy` | 任意の App Server コマンド。                                             |
| `args`         | `stdio-proxy` | 任意のコマンド引数。                                                     |
| `cwd`          | `stdio-proxy` | 任意の子プロセス作業ディレクトリ。                                       |
| `url`          | `websocket`   | 必須の WebSocket URL またはサポート対象のローカルソケット URL。          |
| `authTokenEnv` | `websocket`   | その値でエンドポイントを認証する任意の環境変数。                         |

**Codex Sessions** ページでは、plugin の監督用 App Server を使用し、
アーカイブされていないセッションのみを表示します。明示的な `appServer` 接続設定がない場合、
この接続は管理対象のユーザーホーム stdio になります。保存済みまたはアイドル状態の行から、
最後に永続化されたソースターンまでの、範囲を限定したユーザーおよびアシスタント履歴を含む
モデル固定 Chat を作成できます。そのプライベートバインディングにより、スナップショットのフォーク、
正規の `appServer` ソースブランチ、履歴の注入、および後続ターンがその接続上に維持されます。
最初の正規開始では、フォークから返されたペアを使用します。後続の再開では、OpenClaw のモデルと
プロバイダーのオーバーライドを省略し、Codex が正規スレッドに永続化されたペアを復元できるようにします。
別のネイティブ変更によってそのペアを更新できますが、外側のモデルおよびフォールバックチェーンが
置き換えることはありません。保存済みおよびアイドル状態の行は、他にランナーが存在しないことを確認した後、
アーカイブできます。ただし、別のアクティブな OpenClaw バインディングが、対象そのもの、またはそこから生成された
アーカイブされていない子孫のいずれかを所有している場合を除きます。OpenClaw は Codex の子孫ページネーションに従い、
列挙エラー、循環、または安全上限の枯渇が発生した場合はフェイルクローズします。確認処理は引き続き、
不明なネイティブクライアントと、ステータス確認からアーカイブまでの競合を対象とします。
監督対象のモデル固定 Chat は、ネイティブバインディングを保護している間は削除できません。
アクティブなソースからブランチを作成したり、アーカイブしたりすることはできませんが、既存の監督対象
Chat は引き続き開くことができます。ペアリング済み Node のすべての行は読み取り専用のままです。
Node トランスポートは、ハーネスに必要なストリーミングライフサイクルをまだ提供していません。

`appServer.homeScope: "user"` だけが、管理対象ハーネスプロセスが使用する Codex ホームを変更します。
これはフリートカタログを公開しません。監督を有効にしても、ハーネスのデフォルトは変更されません。
代わりに、明示的な `appServer` 接続設定がない場合、個別の監督接続は
デフォルトで管理対象のユーザーホーム stdio を使用します。その接続では明示的な設定が尊重されます。
保留中およびコミット済みの監督対象バインディングは、すべてのターンでその接続を維持します。
監督が無効である場合や、接続またはライフサイクルにずれがある場合は、エージェントホームのハーネスへ
フォールバックせず、フェイルクローズします。デフォルト接続は、ネイティブ Codex クライアントと
保存済みセッションを共有しますが、各クライアントのプロセスローカルなアクティビティ状態は共有しません。

従来の `plugins.entries.codex-supervisor` 設定は廃止されました。古いエントリ、エンドポイント定義、ポリシーフラグ、
および plugin の許可/拒否参照をこのブロックへ移行するには、
`openclaw doctor --fix` を実行してください。競合する場合は、明示的な正規の
`codex.config.supervision` 値が優先されます。

## App-server トランスポート

通常のハーネスターンでは、OpenClaw は公式 plugin に同梱されている
管理対象 Codex バイナリ（現在は `@openai/codex` `0.145.0`）を起動します。

```bash
codex app-server --listen stdio://
```

これにより、app-server のバージョンは、ローカルに別途インストールされている Codex CLI ではなく、
公式 `codex` plugin に関連付けられます。意図的に別の実行可能ファイルを使用する場合にのみ、
`appServer.command` を設定してください。分離されたエージェントホームをデフォルトで使用する通常の管理対象ターンでは、
macOS デスクトップバンドルがインストールされている場合でも、この固定パッケージが優先されます。
[Computer Use](/ja-JP/plugins/codex-computer-use) が有効な場合、または `homeScope` が
`"user"` でネイティブ Computer Use の状態を読み込める場合、管理対象の起動では代わりに、
必要な macOS 権限を所有するデスクトップアプリのバイナリが優先されます。分離されたエージェントホームの有効な
Codex 設定でネイティブ Computer Use が有効な場合も、同じデスクトップ優先ルールが適用されます。
デスクトップアプリバンドルがインストールされていない場合、OpenClaw は固定パッケージのバイナリにフォールバックします。

実行可能ファイルの引き継ぎとネイティブ設定のフェンシングにより、1 つの実行中 Gateway プロセス内の
クライアントが協調します。別のプロセスがネイティブ Codex plugin 設定を変更した後は、Gateway を再起動してください。

監督では、個別の接続が解決されます。明示的な `appServer` 接続設定がない場合、
`homeScope: "user"` を使用する管理対象 stdio が使われます。通常のハーネスは、
`homeScope: "agent"` を使用する管理対象 stdio のままです。明示的な接続設定は両方のパスで尊重されます。
通常のハーネスでネイティブクライアントと `$CODEX_HOME`（または `~/.codex`）を共有する必要がある場合は、
`homeScope: "user"` を明示的に設定してください。プライベートな監督対象バインディングは、
通常のハーネスのデフォルトに関係なく監督接続を使用します。独立した App Server プロセスは、
それぞれ個別のライブステータスと承認状態を維持します。

すでに実行中の app-server を対象とした非本番テストでは、WebSocket
トランスポートを使用できます。

```json5
{
  plugins: {
    entries: {
      codex: {
        enabled: true,
        config: {
          appServer: {
            transport: "websocket",
            url: "ws://gateway-host:39175",
            authToken: "${CODEX_APP_SERVER_TOKEN}",
            requestTimeoutMs: 60000,
          },
        },
      },
    },
  },
}
```

Codex は WebSocket トランスポートを実験的かつサポート対象外として分類しています。
本番ワークロードでは、管理対象 stdio またはローカル Unix 制御ソケットを優先してください。

`appServer` のフィールド：

| フィールド                                         | デフォルト                                                | 意味                                                                                                                                                                                                                                                                                                                                                                                         |
| --------------------------------------------- | ------------------------------------------------------ | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `transport`                                   | `"stdio"`                                              | `"stdio"` は Codex を起動します。明示的な `"unix"` はローカル制御ソケットに接続し、`"websocket"` は `url` に接続します。                                                                                                                                                                                                                                                                                |
| `homeScope`                                   | `"agent"`                                              | `"agent"` は通常のハーネス状態を OpenClaw エージェントごとに分離します。`"user"` は、ネイティブの `$CODEX_HOME` または `~/.codex` を共有し、ネイティブ認証を使用して、所有者専用のスレッド管理を有効にする明示的なオプトインです。ユーザースコープでは、ローカル stdio または Unix トランスポートをサポートします。別個の監視接続では、値が未設定の場合、stdio または Unix では `"user"`、WebSocket では `"agent"` に解決されます。     |
| `command`                                     | 管理対象の Codex バイナリ                                   | stdio トランスポート用の実行可能ファイルです。管理対象のバイナリを使用する場合は未設定のままにします。                                                                                                                                                                                                                                                                                                                          |
| `args`                                        | `["app-server", "--listen", "stdio://"]`               | stdio トランスポート用の引数です。                                                                                                                                                                                                                                                                                                                                                                  |
| `url`                                         | 未設定                                                  | WebSocket App Server URL または `unix://` URL です。明示的に空の Unix パスを指定すると、標準のユーザーホーム制御ソケットが選択されます。                                                                                                                                                                                                                                                                          |
| `authToken`                                   | 未設定                                                  | WebSocket トランスポート用の Bearer トークンです。リテラル文字列、または `${CODEX_APP_SERVER_TOKEN}` などの SecretInput を受け入れます。                                                                                                                                                                                                                                                                              |
| `headers`                                     | `{}`                                                   | 追加の WebSocket ヘッダーです。ヘッダー値には、リテラル文字列または SecretInput 値（例: `x-codex-client-session-token: "${CODEX_CLIENT_SESSION_TOKEN}"`）を指定できます。                                                                                                                                                                                                                               |
| `clearEnv`                                    | `[]`                                                   | OpenClaw が継承環境を構築した後、起動した stdio app-server プロセスから削除される追加の環境変数名です。                                                                                                                                                                                                                                                             |
| `remoteWorkspaceRoot`                         | 未設定                                                  | リモート Codex app-server のワークスペースルートです。設定すると、OpenClaw は解決済みの OpenClaw ワークスペースからローカルワークスペースルートを推定し、このリモートルート配下で現在の cwd のサフィックスを維持して、最終的な app-server の cwd のみを Codex に送信します。cwd が解決済みの OpenClaw ワークスペースルート外にある場合、OpenClaw は Gateway ローカルのパスをリモート app-server に送信せず、フェイルクローズします。 |
| `loopDetectionPreToolUseRelay`                | `true`                                                 | OpenClaw のループ検出と、その明示的なポリシーなしマーカーにのみ使用される Codex `PreToolUse` サブプロセスをインストールします。ツールごとのプロセス分岐を減らすには、`false` を設定します。ツール実行前の Plugin フックと信頼済みツールポリシーでは、引き続き必要なリレーがインストールされます。                                                                                                                                         |
| `requestTimeoutMs`                            | `60000`                                                | app-server のコントロールプレーン呼び出しのタイムアウトです。                                                                                                                                                                                                                                                                                                                                                     |
| `turnCompletionIdleTimeoutMs`                 | `60000`                                                | OpenClaw が `turn/completed` を待機している間に、Codex がターンを受け入れた後、またはターンスコープの app-server リクエスト後に適用される静止時間枠です。                                                                                                                                                                                                                                                                    |
| `turnAssistantCompletionIdleTimeoutMs`        | `10000`                                                | OpenClaw が引き続き `turn/completed` を待機している間に、最終または非 commentary のアシスタント項目、あるいはツール実行前の未加工アシスタント完了によってアシスタント出力の解放が準備された後に適用される静止時間枠です。値を大きくすると、OpenClaw が中断してセッションレーンを解放する前に、Codex が `turn/completed` を出力するための時間を長く確保できます。                                                                                            |
| `postToolRawAssistantCompletionIdleTimeoutMs` | `300000`                                               | OpenClaw が `turn/completed` を待機している間に、ツールへの引き渡し、ネイティブツールの完了、ツール実行後の未加工アシスタント進行、未加工の推論完了、または推論進行の後に使用される、完了時のアイドル時間と進行状況のガードです。ツール実行後の統合処理が、最終アシスタント解放の時間枠よりも正当に長く無出力になり得る、信頼済みまたは負荷の高いワークロードに使用します。                                |
| `mode`                                        | ローカルの Codex 要件で YOLO が許可されない場合を除き `"yolo"` | YOLO またはガーディアンレビュー済み実行用のプリセットです。                                                                                                                                                                                                                                                                                                                                                 |
| `approvalPolicy`                              | `"never"` または許可されたガーディアン承認ポリシー       | スレッドの開始、再開、ターン時に送信されるネイティブ Codex 承認ポリシーです。                                                                                                                                                                                                                                                                                                                            |
| `sandbox`                                     | `"danger-full-access"` または許可されたガーディアンサンドボックス  | スレッドの開始および再開時に送信されるネイティブ Codex サンドボックスモードです。有効な OpenClaw サンドボックスは、`danger-full-access` ターンを Codex `workspace-write` に制限します。ターンのネットワークフラグは、OpenClaw サンドボックスの外向き通信設定に従います。                                                                                                                                                                                       |
| `approvalsReviewer`                           | `"user"` または許可されたガーディアンレビュアー               | 許可されている場合に Codex がネイティブ承認プロンプトをレビューできるようにするには、`"auto_review"` を使用します。                                                                                                                                                                                                                                                                                                                   |
| `defaultWorkspaceDir`                         | 現在のプロセスディレクトリ                              | `--cwd` が省略された場合に `/codex bind` が使用するワークスペースです。                                                                                                                                                                                                                                                                                                                                        |
| `serviceTier`                                 | 未設定                                                  | オプションの Codex app-server サービス階層です。`"priority"` は高速モードのルーティングを有効にし、`"flex"` は flex 処理を要求し、`null` はオーバーライドを解除します。従来の `"fast"` は `"priority"` として受け入れられます。                                                                                                                                                                                                 |
| `networkProxy`                                | 無効                                               | app-server コマンドで Codex 権限プロファイルのネットワーク機能を使用するようオプトインします。OpenClaw は選択された `permissions.<profile>.network` 設定を定義し、`sandbox` を送信する代わりに `default_permissions` で選択します。                                                                                                                                                                             |
| `experimental.sandboxExecServer`              | `false`                                                | サポート対象の Codex app-server に OpenClaw サンドボックスを基盤とする Codex 環境を登録し、ネイティブ Codex 実行をアクティブな OpenClaw サンドボックス内で実行できるようにする、プレビューへのオプトイン。                                                                                                                                                                                                            |

`appServer.networkProxy` は、Codex サンドボックスの契約を変更するため明示的に指定します。有効にすると、OpenClaw は Codex スレッド設定に `features.network_proxy.enabled` と
`default_permissions` も設定し、生成された権限プロファイルが Codex 管理のネットワーク機能を開始できるようにします。OpenClaw はデフォルトで、プロファイル本体から衝突しにくい
`openclaw-network-<fingerprint>` プロファイル名を生成します。安定したローカル名が必要な場合にのみ
`profileName` を使用してください。

```js
export default {
  plugins: {
    entries: {
      codex: {
        config: {
          appServer: {
            sandbox: "workspace-write",
            networkProxy: {
              enabled: true,
              domains: {
                "api.openai.com": "allow",
                "blocked.example.com": "deny",
              },
              allowUpstreamProxy: true,
              proxyUrl: "http://127.0.0.1:3128",
            },
          },
        },
      },
    },
  },
};
```

通常の app-server ランタイムが `danger-full-access` になる場合、`networkProxy` を有効にすると、生成される権限プロファイルでは代わりにワークスペース形式のファイルシステムアクセスが使用されます。Codex 管理のネットワーク適用はサンドボックス化されたネットワーク機能であるため、フルアクセスプロファイルでは送信トラフィックを保護できません。

Plugin は、古い、未検証の新しい、プレリリース、ビルド接尾辞付き、またはバージョンなしの app-server ハンドシェイクをブロックします。Codex app-server は、`0.143.0` から同梱の `0.145.0` までの安定版バージョンを報告する必要があります。

OpenClaw は、非ループバックの WebSocket app-server URL をリモートとして扱い、`appServer.authToken` または
`Authorization` ヘッダーによる、ID 情報を含む WebSocket 認証を要求します。`appServer.authToken` と各 `appServer.headers.*`
値には SecretInput を使用できます。シークレットランタイムは OpenClaw が app-server の起動オプションを構築する前に SecretRef と env
短縮記法を解決し、未解決の構造化 SecretRef がある場合は、トークンやヘッダーが送信される前に失敗します。ネイティブ Codex Plugin が設定されている場合、OpenClaw は接続先 app-server の Plugin
コントロールプレーンを使用してそれらの Plugin をインストールまたは更新し、その後アプリのインベントリを更新して、Plugin が所有するアプリを Codex スレッドから参照できるようにします。`app/list` は引き続き正式なインベントリおよびメタデータソースですが、一覧にあるアクセス可能なアプリについて Codex が現在無効とマークしている場合でも、`thread/start` が `config.apps[appId].enabled = true` を送信するかどうかは OpenClaw のポリシーが決定します。不明または欠落しているアプリ ID は引き続きフェイルクローズになります。このパスでは `plugin/install` を介してマーケットプレイスの Plugin を有効化し、インベントリを更新するだけです。OpenClaw 管理の Plugin インストールとアプリインベントリの更新を受け入れることが信頼できるリモート app-server にのみ、OpenClaw を接続してください。

## 承認モードとサンドボックスモード

ローカル stdio app-server セッションのデフォルトは YOLO モードです：
`approvalPolicy: "never"`、`approvalsReviewer: "user"`、および
`sandbox: "danger-full-access"`。この信頼されたローカルオペレーター向けの設定により、応答する人がいないネイティブ承認プロンプトに妨げられることなく、無人の OpenClaw ターンと Heartbeat を進行できます。

Codex のローカルシステム要件ファイルで暗黙的な YOLO の承認、レビュアー、またはサンドボックス値が許可されていない場合、OpenClaw は暗黙のデフォルトを代わりに guardian として扱い、許可された guardian 権限を選択します。`tools.exec.mode: "auto"`
も guardian によるレビュー付き Codex 承認を強制し、安全でない従来の
`approvalPolicy: "never"` または `sandbox: "danger-full-access"` のオーバーライドは維持しません。意図的に承認なしの設定にするには `tools.exec.mode: "full"` を設定してください。同じ要件ファイル内でホスト名が一致する `[[remote_sandbox_config]]` エントリも、サンドボックスのデフォルト決定に使用されます。

Codex の guardian によるレビュー付き承認を使用するには、`appServer.mode: "guardian"` を設定します。

```json5
{
  plugins: {
    entries: {
      codex: {
        enabled: true,
        config: {
          appServer: {
            mode: "guardian",
            serviceTier: "priority",
          },
        },
      },
    },
  },
}
```

`guardian` プリセットは、それらの値が許可されている場合、`approvalPolicy: "on-request"`、
`approvalsReviewer: "auto_review"`、および `sandbox: "workspace-write"` に展開されます。個別のポリシーフィールドは `mode` をオーバーライドします。以前の
`guardian_subagent` レビュアー値は互換性エイリアスとして引き続き受け入れられますが、新しい設定では `auto_review` を使用してください。

OpenClaw サンドボックスが有効な場合でも、ローカル Codex app-server プロセスは Gateway ホスト上で実行されます。そのため OpenClaw は、Codex ホスト側のサンドボックス化を OpenClaw サンドボックスバックエンドと同等とはみなさず、そのターンでは Codex ネイティブの Code Mode、ユーザー MCP サーバー、およびアプリを基盤とする Plugin 実行を無効にします。通常の exec/process ツールが利用できる場合、シェルアクセスは `sandbox_exec` や `sandbox_process` など、OpenClaw サンドボックスを基盤とする動的ツールを通じて公開されます。

<Note>
Docker を基盤とする OpenClaw サンドボックスホスト（`agents.defaults.sandbox.mode` が Docker バックエンドに設定されている場合）では、`openclaw doctor` が、サンドボックスコンテナ内での `workspace-write` シェル実行にネストされた Codex `bwrap` が必要とする、非特権ユーザー名前空間と、Docker サンドボックスのネットワーク送信が無効な場合はネットワーク名前空間を、ホストが許可しているかどうか調査します。調査に失敗すると、通常 Ubuntu/AppArmor ホストでは
`bwrap: setting up uid map: Permission denied` または
`bwrap: loopback: Failed RTM_NEWADDR: Operation not permitted` として表面化します。OpenClaw サービスユーザーについて、報告されたホストの名前空間ポリシーを修正し、Gateway を再起動してください。ホスト全体に適用される
`kernel.apparmor_restrict_unprivileged_userns=0` の代替策よりも、サービスプロセスに限定された AppArmor プロファイルを優先し、ネストされた `bwrap` を満たすためだけに Docker コンテナへより広範な権限を付与しないでください。
</Note>

## サンドボックス化されたネイティブ実行

安定版のデフォルトはフェイルクローズです。有効な OpenClaw サンドボックスは、それ以外の場合に Codex app-server ホストから実行されるネイティブ Codex 実行サーフェスを無効にします。OpenClaw のサンドボックスバックエンドで Codex のリモート環境サポートを試す場合にのみ、`appServer.experimental.sandboxExecServer: true` を使用してください。このプレビューパスは、サポートされるすべての Codex app-server バージョンで動作します。

```json5
{
  plugins: {
    entries: {
      codex: {
        enabled: true,
        config: {
          appServer: {
            experimental: {
              sandboxExecServer: true,
            },
          },
        },
      },
    },
  },
}
```

このフラグが有効で、現在の OpenClaw セッションがサンドボックス化されている場合、OpenClaw は有効なサンドボックスを基盤とする local loopback exec-server を起動し、Codex app-server に登録して、その OpenClaw 所有の環境で Codex スレッドとターンを開始します。app-server が環境を登録できない場合、ホスト実行へ暗黙的にフォールバックせず、実行はフェイルクローズで失敗します。

このプレビューパスはローカル専用です。リモート WebSocket app-server は同じホスト上で実行されていない限りループバック exec-server に到達できないため、OpenClaw はこの組み合わせを拒否します。

## 認証と環境の分離

デフォルトのエージェント単位のホームでは、認証は次の順序で選択されます。

1. エージェント用の明示的な OpenClaw Codex 認証プロファイル。
2. そのエージェントの Codex ホームに存在する app-server の既存アカウント。
3. ローカル stdio app-server の起動時に限り、app-server アカウントが存在せず、OpenAI 認証が引き続き必要な場合は、`CODEX_API_KEY`、次に
   `OPENAI_API_KEY`。

OpenClaw が ChatGPT サブスクリプション形式の Codex 認証プロファイル（OAuth またはトークン認証情報タイプ）を検出すると、起動する Codex 子プロセスから `CODEX_API_KEY` と `OPENAI_API_KEY` を削除します。これにより、Gateway レベルの API キーを埋め込みや OpenAI モデルの直接利用に使用できる状態を維持しながら、ネイティブ Codex app-server のターンが誤って API 経由で課金されるのを防ぎます。

明示的な Codex API キープロファイルとローカル stdio の環境キーによるフォールバックでは、子プロセスに継承された環境変数ではなく app-server ログインを使用します。WebSocket app-server 接続には Gateway の環境変数による API キーのフォールバックは渡されません。明示的な認証プロファイルまたはリモート app-server 自身のアカウントを使用してください。

stdio app-server の起動では、デフォルトで OpenClaw のプロセス環境を継承します。OpenClaw は Codex app-server のアカウントブリッジを所有し、`CODEX_HOME` を、そのエージェントの OpenClaw 状態内にあるエージェント単位のディレクトリへ設定します。これにより Codex の設定、アカウント、Plugin のキャッシュとデータ、およびスレッド状態は、オペレーター個人の `~/.codex` ホームから混入せず、OpenClaw エージェントのスコープ内に維持されます。

Codex Desktop および CLI とネイティブ Codex 状態を共有するには、`appServer.homeScope: "user"` を設定します。このローカルユーザーホームモードは、管理対象 stdio と明示的な Unix トランスポートをサポートします。設定されている場合は `$CODEX_HOME` を、それ以外の場合は `~/.codex` を使用し、ネイティブ認証、設定、Plugin、およびスレッドを含みます。
OpenClaw は app-server に対する認証プロファイルブリッジを省略します。確認済みの所有者ターンでは、`codex_threads` を使用して、それらのスレッドの一覧表示（任意の `search` フィルター付き）、読み取り、フォーク、名前変更、アーカイブ、アーカイブ解除を実行できます。OpenClaw でスレッドを継続する前にフォークしてください。独立した Codex プロセス同士は、同じスレッドへの同時書き込みを調整しません。

この `homeScope` のオプトインは、通常のハーネスセッションに適用されます。Codex Sessions を通じて作成された Chat は、代わりに専用の監督接続を使用します。これにより、正規ブランチおよび今後の再開時に、ネイティブ接続の認証とプロバイダー設定が維持されます。

モデルがロックされた監督対象 Chat では、`codex_threads` は別のフォークをアタッチしたり、Chat にバインドされたネイティブスレッドをアーカイブしたりできません。一覧表示とメタデータのみの読み取りは引き続き利用できます。未加工のトランスクリプトを読み取るには `allowRawTranscripts` が必要です。これが無効な場合、ネイティブ検索がトランスクリプトのプレビューに一致する可能性があるため、一覧検索も拒否されます。別の OpenClaw Chat が所有していない無関係なスレッドの名前変更、アーカイブ解除、切り離されたフォーク、およびアーカイブには
`allowWriteControls` が必要です。どちらのオプションもロックされたバインドを迂回しません。

OpenClaw は通常のローカル app-server 起動時に `HOME` を書き換えません。`openclaw`、`gh`、`git`、クラウド CLI、シェルコマンドなど、Codex が実行するサブプロセスは通常のプロセスホームを参照し、ユーザーホームの設定とトークンを検出できます。Codex は `$HOME/.agents/skills` と
`$HOME/.agents/plugins/marketplace.json` も検出する場合があります。この `.agents` の検出は意図的にオペレーターホームと共有され、分離された
`~/.codex` 状態とは別です。

デフォルトのエージェントスコープでは、OpenClaw の Plugin と OpenClaw の Skills スナップショットは引き続き OpenClaw 独自の Plugin レジストリと Skills ローダーを経由しますが、個人用 Codex `~/.codex` アセットは経由しません。分離された OpenClaw エージェントの一部にすべき、有用な Codex CLI の Skills や Plugin が Codex ホームにある場合は、それらのインベントリを明示的に作成します。

```bash
openclaw migrate codex --dry-run
openclaw migrate apply codex --yes
```

デプロイで環境をさらに分離する必要がある場合は、それらの変数を `appServer.clearEnv` に追加します。

```json5
{
  plugins: {
    entries: {
      codex: {
        enabled: true,
        config: {
          appServer: {
            clearEnv: ["CODEX_API_KEY", "OPENAI_API_KEY"],
          },
        },
      },
    },
  },
}
```

`appServer.clearEnv` が影響するのは、起動された Codex app-server 子プロセスのみです。OpenClaw はローカル起動の正規化中に、このリストから `CODEX_HOME` と `HOME` を削除します。`CODEX_HOME` は選択されたエージェントまたはユーザースコープを引き続き参照し、`HOME` は継承されたままになるため、サブプロセスは通常のユーザーホーム状態を使用できます。

## 動的ツール

Codex 動的ツールのデフォルトは `searchable` 読み込みで、`openclaw` 名前空間の下に `deferLoading: true` とともに公開されます。通常、OpenClaw は Codex ネイティブのワークスペース操作や Codex 独自のツール検索サーフェスと重複する動的ツールを公開しません。

- `read`
- `write`
- `edit`
- `apply_patch`
- `exec`
- `process`
- `update_plan`
- `tool_call`
- `tool_describe`
- `tool_search`
- `tool_search_code`

有限のランタイム許可リストによってネイティブ Code Mode が無効化される場合、OpenClaw は空の実行環境選択を送信します。その直接的でサンドボックス化されていない場合、OpenClaw はポリシーでフィルタリングされた `exec` と `process` ツールをシェルのフォールバックとして維持します。ランタイム許可リストと `codexDynamicToolsExclude` は引き続き適用されます。

メッセージング、メディア、Cron、ブラウザー、ノード、Gateway、`heartbeat_respond`、`web_search` など、残りの OpenClaw 統合ツールの大半は、その名前空間の Codex ツール検索から利用できます。これにより、初期モデルコンテキストを小さく保てます。Codex ツール検索が利用できない場合や、コネクターのみのツール群として解決される場合があるため、`codexDynamicToolsLoading` にかかわらず、少数のツールは引き続き直接呼び出せます（`agents_list`、`sessions_spawn`、`sessions_yield`）。開発者指示は、Codex ネイティブのサブエージェント作業では通常の Codex サブエージェントを引き続きネイティブの `spawn_agent` へ誘導します。一方、明示的な OpenClaw または ACP の委任には `sessions_spawn` を引き続き利用できます。メッセージツールのみを使用するソース返信も、ターン制御の契約であるため、引き続き直接処理されます。

Codex Code Mode は、汎用的な OpenClaw 動的ツールの結果をテキストとして投影します。フィールドを読み取る前に JSON の結果を解析してください。ネストされた動的呼び出しは Codex ランタイムによって直列化されるため、`Promise.all` はそれらを並行送信しません。コレクターの子を開始するときは、上限付きの逐次起動ループを使用してください。

OpenClaw の `computer` ツールを含む、`catalogMode: "direct-only"` とマークされたツールは、`openclaw_direct` の下にグループ化されます。OpenClaw は、オペレーターが指定したエントリを置き換えることなく、その名前空間を Codex の `code_mode.direct_only_tool_namespaces` リストに追加します。そのため Codex は、通常のスレッドおよび Code Mode 専用スレッドで、それらのツールをネストされた Code Mode の `tools.*` 呼び出しを介してルーティングするのではなく、`DirectModelOnly` として公開します。この境界は、画像を含む結果に必要です。ネストされた Code Mode のシリアライズでは画像出力がテキストに平坦化され、次のコンピューター操作に必要なスクリーンショットが失われるためです。

遅延された動的ツールを検索できないカスタム Codex app-server に接続する場合、またはツールの完全なペイロードをデバッグする場合にのみ、`codexDynamicToolsLoading: "direct"` を設定してください。

## タイムアウト

OpenClaw が所有する動的ツール呼び出しには、`appServer.requestTimeoutMs` とは独立した上限が設定されます。Codex の各 `item/tool/call` リクエストでは、次の順序で最初に利用可能なタイムアウトが使用されます。

- 呼び出しごとの正の `timeoutMs` 引数。
- `image_generate` の場合は、`agents.defaults.mediaModels.image.timeoutMs`。
- タイムアウトが設定されていない `image_generate` の場合は、画像生成のデフォルトである 120 秒。
- メディア理解用の `image` ツールの場合は、選択された画像対応 `tools.media.models[]` エントリの `timeoutSeconds` をミリ秒に変換した値、またはメディアのデフォルトである 60 秒。画像理解では、これはリクエスト自体に適用され、それ以前の準備作業によって短縮されません。
- `message` ツールの場合は、Gateway 配信と同一キーに対する上限付きの整合処理を含む、固定の 600 秒の外側予算。
- 動的ツールのデフォルトである 90 秒。

このウォッチドッグは、動的 `item/tool/call` の外側予算です。プロバイダー固有のリクエストタイムアウトはその呼び出し内で実行され、それぞれ独自のタイムアウト動作を維持します。動的ツールの予算は 600000 ms が上限です。`agents_wait` は外側の完了猶予として 30000 ms を追加し、app-server クライアントは 660000 ms を許容するため、構造化された待機結果を Codex に届けられます。タイムアウト時、OpenClaw は対応している場合にツールシグナルを中止し、失敗した動的ツール応答を Codex に返します。これにより、セッションを `processing` のまま残さずにターンを続行できます。

Codex がターンを受け入れた後、および OpenClaw がターンスコープの app-server リクエストに応答した後、ハーネスは Codex が現在のターンを進行させ、最終的に `turn/completed` でネイティブターンを完了することを期待します。app-server が `appServer.turnCompletionIdleTimeoutMs` の間何も返さない場合、OpenClaw はベストエフォートで Codex ターンを中断し、診断用タイムアウトを記録して OpenClaw セッションレーンを解放します。これにより、後続のチャットメッセージが古いネイティブターンの後ろでキューに滞留することを防ぎます。

同じターンに対する非終端通知の大半は、Codex がターンの存続を示したため、この短いウォッチドッグを解除します。ツールの引き継ぎでは、より長いツール後アイドル予算を使用します。具体的には、OpenClaw が `item/tool/call` 応答を返した後、`commandExecution` などのネイティブツール項目が完了した後、生の `custom_tool_call_output` が完了した後、およびツール後の生のアシスタント進行状況、生の推論完了、または推論進行状況の後です。このガードは、設定されている場合は `appServer.postToolRawAssistantCompletionIdleTimeoutMs` を使用し、それ以外の場合はデフォルトで 5 分です。同じツール後予算は、Codex が次の現在ターンイベントを発行する前の無音の統合処理時間に対しても、進行状況ウォッチドッグを延長します。推論完了、commentary の `agentMessage` 完了、およびツール前の生の推論またはアシスタント進行状況の後には自動的な最終返信が続く可能性があるため、セッションレーンを直ちに解放せず、進行後返信ガードを使用します。最終または非 commentary の完了済み `agentMessage` 項目と、ツール前の生のアシスタント完了だけが、アシスタント出力解放を作動させます。その後 Codex が `turn/completed` を発行せずに何も返さなくなった場合、OpenClaw はベストエフォートでネイティブターンを中断し、セッションレーンを解放します。アシスタント、ツール、アクティブ項目、または副作用の証拠がないターン完了アイドルタイムアウトを含む、リプレイしても安全な stdio app-server 障害は、新しい app-server 試行で 1 回再試行されます。安全でないタイムアウトでは、停止した app-server クライアントを破棄し、OpenClaw セッションレーンを解放します。また、自動的にリプレイする代わりに、古いネイティブスレッドのバインディングも消去します。完了監視タイムアウトでは、Codex 固有のタイムアウトテキストが表示されます。リプレイしても安全な場合は応答が不完全な可能性があることを示し、安全でない場合は再試行前に現在の状態を確認するようユーザーに伝えます。公開されるタイムアウト診断には、最後の app-server 通知メソッド、生のアシスタント応答項目の ID・タイプ・ロール、アクティブなリクエスト数と項目数、作動中の監視状態などの構造化フィールドが含まれます。最後の通知が生のアシスタント応答項目である場合は、長さを制限したアシスタントテキストのプレビューも含まれます。生のプロンプトやツール内容は含まれません。

## モデル検出

デフォルトでは、Codex Plugin は利用可能なモデルを app-server に問い合わせます。モデルの可用性は Codex app-server が所有するため、OpenClaw がバンドル済みの `@openai/codex` バージョンをアップグレードした場合や、デプロイ環境で `appServer.command` が別の Codex バイナリを参照するよう設定された場合に、リストが変わる可能性があります。可用性はアカウント単位の場合もあります。実行中の Gateway で `/codex models` を使用すると、そのハーネスとアカウントのライブカタログを確認できます。

検出に失敗するかタイムアウトした場合、OpenClaw はバンドル済みのフォールバックカタログを使用します。

| モデル ID       | 表示名 | 推論強度        |
| -------------- | ------------ | ------------------------ |
| `gpt-5.5`      | gpt-5.5      | low, medium, high, xhigh |
| `gpt-5.4-mini` | GPT-5.4-Mini | low, medium, high, xhigh |

<Note>
現在バンドルされているハーネスは `@openai/codex` `0.145.0` です。そのバンドル済み app-server に対する `model/list` プローブでは、次の公開モデル選択行が返されました。

| モデル ID        | 入力モダリティ | 推論強度                    |
| --------------- | ---------------- | ------------------------------------ |
| `gpt-5.6-sol`   | text, image      | low, medium, high, xhigh, max, ultra |
| `gpt-5.6-terra` | text, image      | low, medium, high, xhigh, max, ultra |
| `gpt-5.6-luna`  | text, image      | low, medium, high, xhigh, max        |
| `gpt-5.5`       | text, image      | low, medium, high, xhigh             |
| `gpt-5.2`       | text, image      | low, medium, high, xhigh             |

app-server カタログは `ultra` を報告できますが、OpenClaw の推論制御で現在公開されているレベルは `max` までです。

ライブのモデル選択行はアカウント単位であり、アカウント、Codex カタログ、またはバンドル済みバージョンによって変わる可能性があります。特定時点の表に依存せず、現在のリストを確認するには `/codex models` を実行してください。内部フローや特殊なフロー向けの非表示モデルが、通常のモデル選択肢ではない場合でも app-server カタログに表示されることがあります。
</Note>

検出は `plugins.entries.codex.config.discovery` で調整します。

```json5
{
  plugins: {
    entries: {
      codex: {
        enabled: true,
        config: {
          discovery: {
            enabled: true,
            timeoutMs: 2500,
          },
        },
      },
    },
  },
}
```

起動時に Codex のプローブを避け、フォールバックカタログのみを使用する場合は、検出を無効にします。

```json5
{
  plugins: {
    entries: {
      codex: {
        enabled: true,
        config: {
          discovery: {
            enabled: false,
          },
        },
      },
    },
  },
}
```

## ワークスペースのブートストラップファイル

Codex は、ネイティブのプロジェクトドキュメント検出を通じて `AGENTS.md` 自体を処理します。Codex のフォールバックは `AGENTS.md` がない場合にのみ適用されるため、OpenClaw は合成された Codex プロジェクトドキュメントファイルを書き込まず、ペルソナファイルに Codex のフォールバックファイル名を使用しません。

OpenClaw ワークスペースとの同等性を保つため、Codex ハーネスはその他のブートストラップファイルを開発者指示として転送しますが、すべてを同じ方法では転送しません。

- `TOOLS.md` は **継承される** Codex 開発者指示として転送されるため、ターン中に生成されたネイティブ Codex サブエージェントにも表示されます。
- `SOUL.md`、`IDENTITY.md`、`USER.md` は、**ターンスコープ** のコラボレーション指示として転送されます。ネイティブ Codex サブエージェントはこれらを継承しないため、サブエージェントのターンに親エージェントのペルソナやユーザープロファイルが引き継がれることはありません。
- 読み込まれた OpenClaw Skills の簡潔なリストも、ターンスコープのコラボレーション用開発者指示として転送されるため、ネイティブ Codex サブエージェントはこれも継承しません。
- `HEARTBEAT.md` の内容は注入されません。Heartbeat ターンでは、ファイルが存在し空でない場合にそのファイルを読むよう、コラボレーションモードのポインターが渡されます。
- 設定されたエージェントワークスペースの `MEMORY.md` の内容は、そのワークスペースでメモリツールを利用できる場合、ネイティブ Codex ターン入力に貼り付けられません。ファイルが存在する場合、ハーネスはターンスコープのコラボレーション用開発者指示に小さなワークスペースメモリポインターを追加し、永続メモリが関連する場合は Codex が `memory_search` または `memory_get` を使用する必要があります。ツールが無効である場合、メモリ検索が利用できない場合、またはアクティブなワークスペースがエージェントメモリのワークスペースと異なる場合、`MEMORY.md` は通常の上限付きターンコンテキスト経路を使用します。
- `BOOTSTRAP.md` が存在する場合、OpenClaw ターン入力の参照コンテキストとして転送されます。

## 環境オーバーライド

ローカルテストでは、引き続き環境オーバーライドを利用できます。

- `OPENCLAW_CODEX_APP_SERVER_BIN`
- `OPENCLAW_CODEX_APP_SERVER_ARGS`
- `OPENCLAW_CODEX_APP_SERVER_MODE=yolo|guardian`
- `OPENCLAW_CODEX_APP_SERVER_APPROVAL_POLICY`
- `OPENCLAW_CODEX_APP_SERVER_SANDBOX`

`appServer.command` が未設定の場合、`OPENCLAW_CODEX_APP_SERVER_BIN` は管理対象バイナリをバイパスします。

`OPENCLAW_CODEX_APP_SERVER_GUARDIAN=1` は削除されました。代わりに `plugins.entries.codex.config.appServer.mode: "guardian"` を使用するか、1 回限りのローカルテストには `OPENCLAW_CODEX_APP_SERVER_MODE=guardian` を使用してください。繰り返し可能なデプロイでは設定が推奨されます。設定を使用すると、Plugin の動作を Codex ハーネスのその他のセットアップと同じレビュー済みファイルに保持できるためです。

## 関連項目

- [Codex ハーネス](/ja-JP/plugins/codex-harness)
- [Codex ハーネスランタイム](/ja-JP/plugins/codex-harness-runtime)
- [Codex の監督](/ja-JP/plugins/codex-supervision)
- [ネイティブ Codex Plugin](/ja-JP/plugins/codex-native-plugins)
- [Codex Computer Use](/ja-JP/plugins/codex-computer-use)
- [OpenAI プロバイダー](/ja-JP/providers/openai)
- [設定リファレンス](/ja-JP/gateway/configuration-reference)
