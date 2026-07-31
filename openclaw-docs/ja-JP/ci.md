---
read_when:
    - CI ジョブが実行された、または実行されなかった理由を把握する必要がある
    - 失敗している GitHub Actions チェックをデバッグしています
    - リリース検証の実行または再実行を調整している場合
    - ClawSweeper のディスパッチまたは GitHub アクティビティの転送を変更する場合
summary: CI ジョブグラフ、スコープゲート、リリースアンブレラ、ローカルコマンドの対応関係
title: CI パイプライン
x-i18n:
    generated_at: "2026-07-26T08:53:37Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: f9de5b527354f3cc9eed3813e961116f3834c61bd72b29c92f762c46722815df
    source_path: ci.md
    workflow: 16
---

OpenClaw の CI は、`main` へのプッシュ（トリガーでは Markdown および `docs/**` パスは無視されます）、すべてのドラフトではないプルリクエスト、および手動ディスパッチで実行されます。
正規の `main` へのプッシュは単一実行です。`CI` 同時実行グループにより、1 回の完全な統合サイクルを実行しつつ、GitHub は保留中のプッシュを最新の 1 件だけ保持します。
新しいマージが発生すると、すでに Blacksmith マトリックスを登録した処理をキャンセルするのではなく、その保留中の実行が置き換えられます。
プルリクエストでは引き続き、古くなったヘッドがキャンセルされ、手動ディスパッチには独立したグループが使用されます。`preflight` は差分を分類し、無関係な領域だけが変更された場合は負荷の高いレーンを無効にします。
手動の `workflow_dispatch` 実行では、意図的にスマートスコープをバイパスし、リリース候補と広範な検証のためにグラフ全体へファンアウトします。Android レーンは `include_android`（または `release_gate` 入力）によるオプトインのままです。リリース専用の Plugin カバレッジは、独立した
[`Plugin Prerelease`](#plugin-prerelease) ワークフローにあり、
[`Full Release Validation`](#full-release-validation) または明示的な手動
ディスパッチからのみ実行されます。

## CI パイプラインの概要

| ジョブ                                | 目的                                                                                                                                                                                                               | 実行条件                                   |
| ---------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------- |
| `preflight`                        | 変更されたスコープを検出して CI マニフェストを構築します。Node に関連する正規の `main` では、ファンアウト前に依存関係スナップショットを更新および保守します                                                                        | ドラフトではないプッシュと PR では常に実行             |
| `security-fast`                    | 秘密鍵の検出、`zizmor` による変更済みワークフローの監査、および本番用ロックファイルの監査                                                                                                                             | ドラフトではないプッシュと PR では常に実行             |
| `pnpm-store-warmup`                | Linux Node シャードをブロックせずに、プルリクエストと手動実行用のロックファイルで固定された Actions キャッシュをウォームアップします                                                                                                           | main 以外で Node またはドキュメントチェックのレーンが選択された場合 |
| `build-artifacts`                  | `dist/` と Control UI のビルド、ビルド済み CLI のスモークチェック、起動時メモリ、および埋め込みビルド成果物のチェック                                                                                                                 | Node 関連の変更                          |
| `control-ui-i18n`                  | 生成された Control UI のロケールバンドル、メタデータ、および翻訳メモリを検証します。自動実行では勧告のみ、手動のリリース CI ではブロッキングになります                                                                               | Control UI の i18n 関連の変更および手動 CI |
| `checks-fast-core`                 | 高速な Linux 正当性レーン：抑制ベースラインの最大行数ラチェット、バンドル済み + プロトコル、Bun ランチャー、および CI ルーティングの高速タスク                                                                                  | Node 関連の変更                          |
| `qa-smoke-ci-profile`              | 制限された自動 QA スモーク代表セットを、自己完結した均衡の取れた 2 つのパートに分割します。完全な分類体系のカバレッジは、明示的な QA プロファイルから引き続き利用できます                                                         | Node 関連の変更                          |
| `checks-fast-contracts-plugins-*`  | 重み付けされた 2 つの Plugin コントラクトシャード                                                                                                                                                                                   | Node 関連の変更                          |
| `checks-fast-contracts-channels-*` | 重み付けされた 2 つのチャネルコントラクトシャード                                                                                                                                                                                  | Node 関連の変更                          |
| `checks-node-*`                    | プルリクエストでは変更対象の Node テスト、`main`、手動、リリース、および広範なフォールバック実行では完全なコアシャード                                                                                                      | Node 関連の変更                          |
| `check-*`                          | シャード化された main のローカルゲート相当：ガード、shrinkwrap、バンドル済みチャネルの設定メタデータ、本番型、lint、依存関係、テスト型                                                                                   | Node 関連の変更                          |
| `check-additional-*`               | 境界チェックのストライプ（プロンプトスナップショットのドリフトを含む）、セッションアクセサー／トランスクリプトリーダー／SQLite トランザクションの境界、拡張機能の lint グループ、パッケージ境界のコンパイル／カナリア、およびランタイムトポロジーのアーキテクチャ | Node 関連の変更                          |
| `checks-node-compat-node22`        | Node 22 互換性ビルドおよびスモークレーン                                                                                                                                                                            | リリース向けの手動 CI ディスパッチ                |
| `check-docs`                       | ドキュメントのフォーマット、lint、およびリンク切れのチェック                                                                                                                                                                         | ドキュメント変更時（PR および手動ディスパッチ）         |
| `native-i18n`                      | ソース PR ではネイティブソースの抽出とローカライズの安全性を検証し、生成 PR と手動 CI では翻訳済み／プラットフォーム生成済み成果物の完全な一致を強制します                                                               | ネイティブ i18n 関連の変更                   |
| `skills-python`                    | Python ベースの Skills に対する Ruff + pytest                                                                                                                                                                                | Python Skills 関連の変更                  |
| `checks-windows`                   | Windows 固有のプロセス／パステスト、および共有ランタイムのインポート指定子に関するリグレッションテスト                                                                                                                                  | Windows 関連の変更                       |
| `macos-node`                       | macOS に焦点を当てた TypeScript テスト：launchd、Homebrew、ランタイムパス、パッケージングスクリプト、プロセスグループラッパー                                                                                                            | macOS 関連の変更                         |
| `macos-swift`                      | macOS アプリの Swift lint とビルド、およびアプリと共有 OpenClawKit パッケージのテスト                                                                                                                         | macOS 関連の変更                         |
| `ios-build`                        | Xcode プロジェクトの生成および iOS アプリのシミュレータービルド                                                                                                                                                             | iOS アプリ、共有アプリキット、または Swabble の変更    |
| `android`                          | 両方のフレーバーに対する Android 単体テスト、および 1 回のデバッグ APK ビルド                                                                                                                                                          | Android 関連の変更                       |
| `openclaw/ci-gate`                 | 最終集約：事前チェックとセキュリティの成功を必須とし、マニフェストによって無効化されたダウンストリームレーンについてのみスキップを許可します                                                                                                           | ドラフトではないすべての CI 実行                         |
| `test-performance-agent`           | 独立したワークフロー：信頼済みアクティビティ後に Codex の低速テストを毎日最適化します                                                                                                                                          | main CI の成功時または手動ディスパッチ             |
| `openclaw-performance`             | 独立したワークフロー：モックプロバイダー、詳細プロファイル、GPT 5.6 ライブレーンを使用した Kova ランタイムのパフォーマンスレポートを毎日またはオンデマンドで生成します                                                                                          | スケジュール実行および手動ディスパッチ                  |

独立した Periphery ワークフローにより、iOS および macOS アプリのデッドコード検出件数がゼロであることを強制します。共有 OpenClawKit ワークフローは両方の利用側を並列にスキャンし、両方のビルドから同じ Swift USR が Periphery によって出力された場合にのみ宣言を報告します。生成された `OpenClawProtocol/GatewayModels.swift` スキーマコントラクトは、アプリローカルのデッドコードとして扱われるのではなく、ジェネレーターが所有するコードとして保持されます。

## フェイルファストの順序

1. `preflight` は、どのレーンが存在するかを決定します。`docs-scope` と `changed-scope` のロジックは、このジョブ内のステップであり、独立したジョブではありません。正規の `main` は直ちに開始されますが、その同時実行グループは完全な実行を 1 つだけ許可し、それ以降のプッシュを最新の保留中の 1 実行にまとめます。Node に関連する main へのプッシュでは、ダウンストリームジョブがキーをマウントできるようになる前に、唯一の依存関係ディスク書き込み処理とそのサイズ保守もここで直列化されます。Blacksmith は新しいコミットを後続のワークフロー実行にしか公開しない場合があるため、同じ実行内のコンシューマーはマーカー検証済みのローカルフォールバックを維持します。
2. `security-fast`、`check-*`、`check-additional-*`、`check-docs`、および `skills-python` は、より負荷の高い成果物ジョブやプラットフォームマトリックスジョブを待たずに迅速に失敗します。
3. `build-artifacts` とロケールチェックは、高速な Linux レーンと並行して実行されます。Control UI およびネイティブアプリのソース PR では、生成されたロケールスナップショット／リソースを除外します。それらの直列化された更新ワークフローは、独立した生成 PR をバックグラウンドで修復し、自動マージします。ソース CI は引き続き、古くなったソースインベントリと安全でないローカライズ呼び出しをブロックします。生成 PR、手動 CI、およびリリース準備では、翻訳済み／プラットフォーム生成済み成果物の完全な一致を強制します。正規の `release/YYYY.M.PATCH` ブランチには、他の生成済みリリース出力とともに、リリース準備用のロケール修復が含まれる場合があります。
4. その後、より負荷の高いプラットフォームおよびランタイムレーンがファンアウトします：`checks-fast-core`、`checks-fast-contracts-plugins-*`、`checks-fast-contracts-channels-*`、`checks-node-*`、`checks-windows`、`macos-node`、`macos-swift`、`ios-build`、および `android`。
5. `openclaw/ci-gate` は、選択されたすべてのレーンを待機します。事前チェックとセキュリティは成功しなければなりません。ダウンストリームジョブをスキップできるのは、マニフェストで選択されなかった場合のみです。選択されたレーンが失敗またはキャンセルされると、集約も失敗します。

マージコーディネーターは、同じプルリクエストのヘッドについて、認証済みで成功した `openclaw/ci-gate`
を最大 24 時間再利用できます。これにより、無関係な `main` の変更後に
コントリビューターのブランチを書き換えずに済みます。再利用可能な結果は、現在の `main` に対する
独立した厳格な、App 所有のテストマージチェックを置き換えるものではありません。
その後の保留中または失敗した再実行によって、鮮度期間中に変更されていない
ヘッドに対する以前の成功結果が消去されることはありません。

デフォルトブランチのルールセットでは、GitHub Actions が所有する `openclaw/ci-gate` チェックが必須です。リポジトリのメンテナーと管理者には、署名付きの直接 fast-forward ランディングのみを目的とした、監査対象の緊急時バイパスがあります。ただし、組織のルールセットは削除と非 fast-forward 更新を引き続きブロックします。通常のプルリクエストのマージでは、失敗した CI をバイパスせず、引き続きゲートを使用する必要があります。これとは別の厳格な App 所有の test-merge チェックは、引き続き head を現在の `main` に結び付けます。

新しい head がランディングすると、GitHub は置き換えられたプルリクエストジョブを `cancelled` としてマークすることがあります。同じ PR の最新の実行も失敗している場合を除き、これは CI ノイズとして扱います。正規の `main` 実行は、受け入れ後にはキャンセルされません。マージトラフィックが到着すると、GitHub は古い保留中の実行だけを最新の tip に置き換えます。マトリックスジョブは `fail-fast: false` を使用し、`build-artifacts` は小さな検証ジョブをキューに入れる代わりに、組み込みチャンネル、コアサポート境界、および Gateway 監視の失敗を直接報告します。自動 CI の同時実行キーにはバージョンが付けられているため（`CI-v7-*`）、古いキューグループにある GitHub 側のゾンビが新しい main の実行を無期限にブロックすることはありません。手動のフルスイート実行は `CI-manual-v1-*` を使用し、進行中の実行をキャンセルしません。プラグインリストの起動時メモリガードは、セルフホストの Blacksmith Linux では上限を 350 MiB に維持し、同じビルド済み CLI でも RSS ベースラインが高い GitHub ホストの Linux では 425 MiB を許容します。

GitHub Actions から実時間、キュー時間、最も遅いジョブ、失敗、および `pnpm-store-warmup` ファンアウトバリアを要約するには、`pnpm ci:timings`、`pnpm ci:timings:recent`、または `node scripts/ci-run-timings.mjs <run-id>` を使用します。ワークフロー内の `ci-timings-summary` ジョブは `ci.yml` に存在しますが、現在は無効です（`if: false`）。代わりに、タイミングヘルパーをローカルで実行してください。ビルド時間については、`build-artifacts` ジョブの `Build dist` ステップを確認してください。`pnpm build:ci-artifacts` は `[build-all] phase timings:` を出力し、`ui:build` を含みます。また、このジョブは `startup-memory` アーティファクトもアップロードします。

## PR のコンテキストと証拠

外部コントリビューターの PR では、
`.github/workflows/real-behavior-proof.yml` から PR のコンテキストと証拠のゲートが実行されます。
このワークフローは、信頼されたワークフローのリビジョン
（`github.workflow_sha`）をチェックアウトし、PR 本文のみを評価します。
コントリビューターのブランチのコードは実行しません。

このゲートは、リポジトリの所有者、メンバー、
コラボレーター、ボットのいずれでもない PR 作成者に適用されます。
PR 本文に作成者自身が記述した `What Problem This Solves` および
`Evidence` セクションが含まれていれば合格します。証拠には、対象を絞った
テスト、CI 結果、スクリーンショット、録画、ターミナル出力、ライブ観察、
編集済みログ、またはアーティファクトへのリンクを使用できます。本文によって意図と有用な検証情報が示され、
レビュー担当者はコード、テスト、CI を確認して正しさを評価します。

チェックが失敗した場合は、別のコードコミットをプッシュするのではなく、PR 本文を更新してください。

## スコープとルーティング

スコープロジックは `scripts/ci-changed-scope.mjs` にあり、`src/scripts/ci-changed-scope.test.ts` の単体テストでカバーされています。手動ディスパッチでは変更スコープの検出をスキップし、プリフライトマニフェストはすべての対象領域が変更されたものとして動作します。

iOS と macOS の個別の Periphery ワークフローでは、検出ゼロのデッドコードポリシーを適用します。各ワークフローは、ドラフトではないプルリクエストが対応するネイティブスキャンスコープに触れた場合、または手動でディスパッチされた場合にのみ実行されます。

- **CI ワークフローの編集**では、Node CI グラフ、ワークフローの lint、および Windows レーン（`ci.yml` が実行）を検証しますが、それだけで iOS、Android、または macOS のネイティブビルドを強制することはありません。これらのプラットフォームレーンは、引き続きプラットフォームのソース変更に限定されます。
- **ワークフローの健全性チェック**では、すべてのワークフロー YAML ファイルに対して `actionlint` と `zizmor`、複合アクションの補間ガード、および競合マーカーガードを実行します。PR スコープの `security-fast` ジョブでも、変更されたワークフローファイルに対して `zizmor` を実行し、ワークフローのセキュリティ上の問題がメインの CI グラフで早期に失敗するようにします。
- `main` へのプッシュ時の **ドキュメント**は、CI と同じ ClawHub ドキュメントミラーを使用するスタンドアロンの `Docs` ワークフローでチェックされるため、コードとドキュメントが混在するプッシュで CI の `check-docs` シャードまでキューに追加されることはありません。プルリクエストと手動 CI では、ドキュメントが変更された場合、引き続き CI から `check-docs` を実行します。
- **TUI PTY** は、TUI の変更時に `checks-node-core-runtime-tui-pty` Linux Node シャードで実行されます。このシャードは `OPENCLAW_TUI_PTY_INCLUDE_LOCAL=1` を指定して `test/vitest/vitest.tui-pty.config.ts` を実行するため、決定論的な `TuiBackend` フィクスチャレーンと、外部モデルエンドポイントのみをモックする低速な `tui --local` スモークの両方をカバーします。
- **CI ルーティングのみの編集、高速タスクが直接実行する少数のコアテストフィクスチャ、および限定的なプラグイン契約ヘルパーの編集**では、高速な Node 専用のマニフェストパスを使用します。対象は `preflight`、`security-fast`、および変更が影響する高速レーンのみです。具体的には、単一の `checks-fast-core` CI ルーティングタスク、2 つのプラグイン契約シャード、またはその両方です。このパスでは、ビルドアーティファクト、Node 22 互換性、チャンネル契約、完全なコアシャード、バンドル済みプラグインシャード、および追加のガードマトリックスをスキップします。
- **Windows Node チェック**は、Windows 固有のプロセス／パスラッパー、npm／pnpm／UI ランナーヘルパー、パッケージマネージャー設定、およびそのレーンを実行する CI ワークフローのサーフェスに限定されます。無関係なソース、プラグイン、インストールスモーク、およびテストのみの変更は、Linux Node レーンに留まります。

最も遅い Node テストファミリーは、各ジョブを小規模に保ち、ランナーを過剰に予約しないように分割または均衡化されています。

- Plugin コントラクトとチャネルコントラクトは、それぞれ標準の GitHub ランナーフォールバックを備えた、重み付きの Blacksmith バックエンドシャード 2 つとして実行されます。
- コアユニットの高速レーンとサポートレーンは別々に実行されます。コアランタイムインフラは、プロセス、共有、フック、シークレット、および 3 つの Cron ドメインシャードに分割されます。
- 自動返信は負荷を均等化したワーカーとして実行され、返信サブツリーは agent-runner、commands、dispatch、session、state-routing の各シャードに分割されます。
- エージェント型 Gateway/サーバー（コントロールプレーン）の設定は、ビルド済み成果物を待つのではなく、チャット、認証、モデル、HTTP/Plugin、ランタイム、起動の各レーンに分割されます。
- 通常の CI は、分離されたインフラの include-pattern シャードのみを最大 64 個のテストファイルからなる決定論的なバンドルにまとめ、分離されていない command/cron、ステートフルな agents-core、または gateway/server の各スイートを統合することなく Node マトリックスを削減します。負荷の高い固定スイートは引き続き 8 vCPU を使用し、バンドルされたレーンと低負荷のレーンは 4 vCPU を使用します。
- 正規リポジトリのプルリクエストでは、合成されたマージツリーの差分に対して変更テストリゾルバーを再利用します。変更範囲を正確に特定できる場合は、対象を限定した Node ジョブを 1 つ実行します。ステートフルなスイートの分離を維持するため、選択された各テストファイルには独自のプロセスが割り当てられます。プランナーは兄弟テストとインポートグラフ上の依存側を組み合わせ、ワークスペースパッケージ、パッケージ/ロックファイル、共有ハーネス、分割設定、名前変更、削除、公開拡張コントラクトの変更、特別なシャード設定を持つテスト、部分的にしか解決できない対象または空の対象、過大なパスまたは対象プラン、およびプランナーエラーの場合は、既存の 14 ジョブからなるコンパクトなフルスイートプランへフォールバックします。対象限定プランでも、リポジトリスキャナーをインポートから導出できないため、ビルド済み成果物の境界ゲート全体を常に維持します。`main` のプッシュでも同じコンパクトなフルスイートを実行します。保留中の中間プッシュイベントはまとめられる可能性があるため、最後に残った最新の実行では、最終的な単一プッシュの差分だけでなく、統合ツリー全体を検証する必要があります。手動ディスパッチとリリースゲートでは、名前付きの完全なシャード別マトリックスを維持します。
- 完全な Node マトリックスでは、常に低速な直列ツール処理、自動返信コマンドシャード、広範な core-fast キャッシュライターを最初に投入します。これにより、28 ジョブの上限を維持しながら、クリティカルパスの処理と次回実行用の変換シードが後続のウェーブへずれ込むのを防ぎます。
- 広範なブラウザー、QA、メディア、その他の Plugin テストでは、共有の Plugin 一括設定ではなく、それぞれ専用の Vitest 設定を使用します。include-pattern シャードは CI シャード名を使用してタイミングエントリを記録するため、`.artifacts/vitest-shard-timings.json` は設定全体とフィルタリングされたシャードを区別できます。
- Linux Node シャードジョブでは、Vitest の実験的なファイルシステムモジュールキャッシュをアップストリームの Actions キャッシュ API 経由で永続化します。Blacksmith はそのランナー上でこれを透過的に高速化します。すべての CI シャードは復元専用であり、保護されたシードをそれぞれ独自のランナーローカルルートに展開します。その後、シャードラッパーは並行する Vitest プロセスに個別のライブサブディレクトリを割り当てます。キャンセルされない日次ウォーマーまたは明示的にディスパッチされたウォーマーのみが新しい不変アーカイブを保存するため、プルリクエストが変換結果を公開したり、PR ごとのキャッシュファミリーを生成したりすることはできません。変換入力のフィンガープリントにより、互換性のないロックファイル、パッケージ、tsconfig、Vitest 設定の世代が消去されます。保護されたライターは復元済みキャッシュを走査し、2 GiB を超えると 75% まで削減します。Vitest はモジュール ID、ソース内容、環境、解決済みの変換設定をハッシュするため、通常の部分的なソース変更では未変更のエントリをウォーム状態に保ちつつ、変更されたモジュールは安全にキャッシュミスとなります。大まかな復元プレフィックスがワークフロー実行間を橋渡しし、通常の Actions キャッシュの LRU と非アクティブ時の削除によって古い不変アーカイブが制限されます。
- 信頼済みの Linux Node ジョブでは、サポート対象の Node 系統ごとに 1 つの保護された依存関係ディスクから pnpm ストアと `node_modules` もバインドします。パッケージマニフェスト、インストール設定、ランナープラットフォーム、正確な Node パッチバージョンはディスクキーに含めません。正確なランタイムとインストール入力のフィンガープリントによって、ジョブがツリーを再利用するか、再インストールして同じディスクを更新するかを決定します。マニフェストはハッシュ化の前に正規化されます。監査済みの直接的なルートフックでは pnpm のインストールライフサイクルスクリプトのみを維持するため、フォーマット処理や通常のテスト/ビルドスクリプトの編集ではウォーム状態の依存関係ツリーを維持できます。監査されていないライフサイクルフックのドリフトは、そのソース入力がフィンガープリントコントラクトに加わるまでフェイルクローズとなります。依存関係、パッケージマネージャー、フックソース、ロックファイルの変更は常にスナップショットを無効化します。フィンガープリントの一致は必要条件ですが十分条件ではありません。セットアップではインポーターアーカイブとマニフェストのチェックサムも確認し、その後、postinstall によって保持されたレジストリ由来のロックファイル依存関係を、Node が各インポーターから解決するパッケージマニフェストに照らして検証します。インポーターの内容が欠落しているか古い場合は、ルートのホイスト結果を提供せず、新規インストールへフォールバックします。読み取り専用スナップショットを利用できないプルリクエストでは、ワークスペースのバインドを解除してランナーローカルストレージへインストールし、公開できないクローンへの低速な書き込みを回避します。スティッキーなコールドインストールでは pnpm 内部のフェッチ再試行を無効にし、段階的にウォーム化されるストアから、上限付きの完全インストールを最大 3 回試行します。タイムアウトは引き続き失敗として扱われます。内容検証済みの復元または frozen-lockfile インストールの後、セットアップは pnpm による冗長な実行前依存関係チェックを無効にします。このリポジトリでは意図的に Plugin ローカルの `node_modules` を削減しており、無効にしない場合、pnpm はそれらを古いものと判断し、シャードのファンアウト中に安全でない暗黙的な並行インストールによって修復します。正規 main のプリフライトだけが書き込みを行い、更新のたびにストアを測定し、廃止されたパッケージバージョンによって 8 GiB を超えた場合にのみ `pnpm store prune` を実行します。Blacksmith のスナップショット公開はライタージョブの完了後も非同期であるため、新しいキーまたはフィンガープリントの最初の実行はコールドなままの場合があります。その後の内容検証済みの完全一致マーカー復元がロールアウトの証明となります。必須の CI ジョブとプルリクエストには使い捨てクローンが割り当てられるため、依存関係の変更によって新しいディスク、競合するスナップショット、またはビルドをキャンセルし得るキャッシュロックが作成されることはありません。
- Node シャードジョブとビルド成果物ジョブでは、Node のポータブルなオンディスクコンパイルキャッシュも、不変の Actions キャッシュ経由で復元します。独立した `test` と `build` の名前空間により、それぞれのライターが互いのアーカイブを置き換えることを防ぎます。スケジュールされたテストウォーマーは保護されたテストシードを所有し、`build-artifacts` は信頼済みの `main` プッシュから、UTC で 1 日あたり最大 1 つの保護されたビルドアーカイブを公開できます。PR ジョブと通常のテストジョブは保護されたスナップショットを読み取るだけなので、機能ブランチのバイトコードが共有シードへ入ることはなく、PR トラフィックによってキャッシュアーカイブが作成されることもありません。これにより、ソースグラフの一部のみが変更された場合も含め、異なるチェックアウトパス間で、Node が読み込むオーケストレーション、ビルドツール、外部依存関係の V8 バイトコードを再利用できます。動的な設定内でカバレッジが有効になる可能性があり、スクリプトがバイトコードからデシリアライズされた場合に V8 カバレッジがソース位置の精度を失う可能性があるため、Vitest の子プロセスは継承されたコンパイルキャッシュを無効にします。
- ビルド成果物ジョブでは、内容フィンガープリント付きの `build-all` ステップ出力も永続化します。CI が自身でビルドする Plugin SDK 宣言では、リポジトリ所有の TypeScript/JSON ソースグラフ全体をハッシュし、インストール済みディレクトリと生成済みディレクトリを除外します。また、`tsdown` が `dist` を消去した後、フラットな宣言とパッケージブリッジの両方を復元します。そのグラフ外のドキュメント、ワークフロー、Plugin、その他の変更では宣言スナップショットを再利用できます。ソース変更がある場合は、エクスポートゲートの実行前に再ビルドされます。
- 完全な宣言ビルドでは、`tsdown` を AI、ワークスペースパッケージ、統合の各グループに分割します。各グループは宣言のみをキャッシュし、それらの宣言を復元する前にランタイム JavaScript を毎回再ビルドします。そのため、コアまたは Plugin の変更では大規模な統合グラフのみが無効化され、ワークスペースパッケージの変更では依存するすべての宣言グループが保守的に無効化されます。公開の完全ビルドでは通常、不変の Actions キャッシュを使用します。大まかな復元キーが部分的な変更のシードとなり、グループごとの内容フィンガープリントが古いデータを拒否し、GitHub のキャッシュクォータが古い世代を削除します。一方、週次の Node 22 レーンでは、`main` の実行成功後に 14 日間保持される成果物を公開し、その不変の生成元 ID が `main` 上の当該ワークフローに解決される成果物のみを復元します。これにより、PR コードに共有キャッシュへの書き込みを許可せずに、クォータの入れ替わりを回避します。キャッシュの名前空間は機密性の境界ではないため、Private-QA の宣言が Actions キャッシュへ永続化されることはありません。
- `check-additional-*` は、補助的な境界ガードリスト（`scripts/run-additional-boundary-checks.mjs`）を、プロンプト負荷の高い 1 つのシャード（Codex プロンプトスナップショットのドリフトチェックを含む `check-additional-boundaries-a`）と、残りのストライプをまとめた 1 つのシャード（`check-additional-boundaries-bcd`）に分散します。各シャードは独立したガードを並行実行し、チェックごとの所要時間を出力します。パッケージ境界のコンパイル/カナリア処理はまとめたままにし、ランタイムトポロジーのアーキテクチャチェックは `build-artifacts` に組み込まれた Gateway watch カバレッジとは別に実行します。
- 32 vCPU のセルフホスト型ビルドランナーでは、`dist/` と `dist-runtime/` のビルド完了後、Gateway watch、チャネルテスト、コアのサポート境界シャードが `build-artifacts` 内で同時に開始されます。GitHub ホスト型のフォールバック実行では、コア数の少なさによる競合で準備完了の期限を超過しないよう、Gateway watch を直列のまま実行します。

実行が許可されると、正規の Linux CI では Node テストジョブを最大 28 個、
小規模な高速/チェックレーンを最大 12 個まで同時実行できます。Windows と Android は
ランナープールがより限定されているため、2 個のままです。設定全体をまとめたコンパクトなバッチには
120 分のバッチタイムアウトが適用され、include-pattern グループは同じ上限付き
ジョブ予算を共有します。

Android CI は `testPlayDebugUnitTest` と `testThirdPartyDebugUnitTest` の両方を実行した後、Play デバッグ APK をビルドします。サードパーティフレーバーには個別のソースセットやマニフェストがありません。そのユニットテストレーンでは SMS/通話履歴の BuildConfig フラグを使用して引き続きフレーバーをコンパイルしますが、Android に関連するプッシュのたびに重複するデバッグ APK パッケージングジョブを実行することは避けます。現在の各 Gradle タスクには保護されたスティッキーディスクが 1 つあります。PR ジョブは使い捨てクローンを使用し、保護された実行は内容アドレス付きの Gradle エントリをその場で更新します。

Blacksmith のスティッキーディスクキーは、サポート対象のランタイムまたはタスクの次元によって意図的に制限され、PR 番号、コミット、実行、ブランチ、依存関係ハッシュは決して使用しません。ランタイム変換キャッシュとコンパイルキャッシュでは、スティッキーディスクではなく Actions キャッシュを使用します。不変アーカイブでは検証可能な復元/保存結果が得られ、可変スナップショットの昇格失敗を回避できるためです。スティッキーキーのバージョン移行後は、廃止された正確なキー、アーキテクチャ、リージョンの ID のみを `.github/retired-sticky-disks.json` に追加し、同じ次元と確認指定で `main` から `Sticky Disk Cleanup` をディスパッチして、削除を検証した後、それらのエントリを削除します。このワークフローは ARM の ID を ARM ランナーへルーティングし、ランナーとリージョンの不一致を拒否し、Blacksmith の完全一致キ―削除アクションを使用します。Docker ビルダーキャッシュやワイルドカードプレフィックスは決して削除しません。Actions キャッシュのアーカイブでは通常の LRU と非アクティブ時の削除を使用します。

`check-dependencies` シャードは、本番環境の Knip による依存関係、未使用ファイル、未使用エクスポートのチェックを実行します。未使用ファイルガードは、PR が新しい未レビューの未使用ファイルを追加した場合、または古い許可リストエントリを残した場合に失敗します。一方で、Knip が静的に解決できない、意図的な動的 Plugin、生成物、ビルド、ライブテスト、パッケージブリッジの各サーフェスは維持します。未使用エクスポートガードはテストサポートファイルを除外し、本番環境の未使用エクスポートが 1 つでもあれば失敗します。意図的な動的コンシューマーは `config/knip.config.ts` でモデル化する必要があります。履歴上の対象は、エクスポートガードを備えている場合はそれを実行し、それ以外の場合は従来のデッドコードフォールバックを維持します。

## ClawSweeper アクティビティの転送

`.github/workflows/clawsweeper-dispatch.yml` は、OpenClaw リポジトリのアクティビティを ClawSweeper に接続するターゲット側のブリッジです。信頼されていないプルリクエストのコードをチェックアウトしたり実行したりすることはありません。このワークフローは `CLAWSWEEPER_APP_PRIVATE_KEY` から GitHub App トークンを作成し、コンパクトな `repository_dispatch` ペイロードを `openclaw/clawsweeper` にディスパッチします。

このワークフローには 4 つのレーンがあります。

- `clawsweeper_item`：特定の Issue およびプルリクエストのレビューリクエスト用。
- `clawsweeper_comment`：Issue コメント内の明示的な ClawSweeper コマンド用。
- `clawsweeper_commit_review`：`main` へのプッシュに対するコミット単位のレビューリクエスト用。
- `github_activity`：ClawSweeper エージェントが調査できる一般的な GitHub アクティビティ用。

`github_activity` レーンは、正規化されたメタデータのみを転送します。イベントタイプ、アクション、アクター、リポジトリ、項目番号、URL、タイトル、状態、および存在する場合はコメントまたはレビューの短い抜粋です。Webhook 本文全体は意図的に転送しません。`openclaw/clawsweeper` 内の受信ワークフローは `.github/workflows/github-activity.yml` であり、正規化されたイベントを ClawSweeper エージェント用の OpenClaw Gateway フックに投稿します。

一般的なアクティビティは監視対象であり、デフォルトで配信されるものではありません。ClawSweeper エージェントはプロンプト内で Discord の送信先を受け取り、イベントが予想外、対処可能、危険、または運用上有用な場合にのみ `#clawsweeper` に投稿する必要があります。通常の作成、編集、bot による変動、重複した Webhook ノイズ、通常のレビュートラフィックの場合は、`NO_REPLY` とする必要があります。

この経路全体を通じて、GitHub のタイトル、コメント、本文、レビューテキスト、ブランチ名、コミットメッセージは信頼されていないデータとして扱ってください。これらは要約とトリアージのための入力であり、ワークフローやエージェントランタイムへの指示ではありません。

## 手動ディスパッチ

手動 CI ディスパッチは通常の CI と同じジョブグラフを実行しますが、Android 以外のスコープ対象レーンをすべて強制的に有効化します。対象は Linux Node シャード、バンドル Plugin シャード、Plugin およびチャネル契約シャード、Node 22 互換性、`check-*`、`check-additional-*`、ビルド済み成果物のスモークチェック、ドキュメントチェック、Python Skills、Windows、macOS、iOS ビルド、Control UI／ネイティブアプリの i18n です。自動実行されるソース PR では、同じ PR に翻訳済み出力やプラットフォーム生成出力を含めることなく、ネイティブ抽出インベントリと Android／Apple ローカライズの安全性を検証します。直列化された Native App Locale Refresh ワークフローは、これらの成果物を 1 つの分離された PR で再ビルドし、必須チェックの合格後に正確な head に対する自動マージを有効にします。生成成果物の PR、手動 CI、Full Release Validation、リリース準備では、ネイティブの完全な同等性が引き続きブロッキング要件です。Control UI のロケール同等性は、自動 PR および `main` の実行では勧告扱い、手動／リリース CI ではブロッキング要件です。単独の手動 CI ディスパッチでは、`include_android=true` を指定した場合のみ Android を実行します（`release_gate` 入力でも Android が強制されます）。完全なリリース包括ワークフローでは、`include_android=true` を渡して Android を有効化します。Plugin プレリリースの静的チェック、リリース専用の `agentic-plugins` シャード、すべての拡張機能を対象とする一括スイープ、および Plugin プレリリースの Docker レーンは CI から除外されます。Docker プレリリーススイートは、`Full Release Validation` がリリース検証ゲートを有効にして別個の `Plugin Prerelease` ワークフローをディスパッチした場合にのみ実行されます。

PR の最大行数チェックでは、チェックアウトされた合成マージツリーからベースラインを導出し、その head の親がイベントの head と一致することを検証します。手動実行では一意の同時実行グループを使用するため、リリース候補のフルスイートが、同じ ref に対する別のプッシュや PR 実行によってキャンセルされることはありません。任意の `target_ref` 入力を使用すると、信頼された呼び出し元は、選択したディスパッチ ref のワークフローファイルを使用しながら、ブランチ、タグ、または完全なコミット SHA に対してそのグラフを実行できます。最大行数のベースラインは、その実行で解決されたデフォルトブランチの head とターゲットとのマージベースに対して比較されます。`release_gate` 入力は、キャパシティ不足で停滞した PR CI に対するメンテナー向けの正確な SHA フォールバックです。`target_ref` が、ディスパッチされたブランチの head と一致する完全なコミット SHA であり、`pull_request_number` がマージツリーの検証対象となるオープン PR を識別する必要があります。

```bash
gh workflow run ci.yml --ref release/YYYY.M.PATCH
gh workflow run ci.yml --ref main -f target_ref=<branch-or-sha> -f include_android=true
gh workflow run full-release-validation.yml --ref main -f ref=<branch-or-sha>
```

Gateway extended-stable は、`extended-stable/YYYY.M.33` から npm プリフライト、Full Release Validation、および Plugin
npm リリースを実行します。コアの公開処理は、これら 3 つの
実行 ID と検証試行番号を使用します。公開処理ではすべての実行が正規ブランチとリリース SHA に紐付けられるため、
`release-ci/*` の証拠は無効です。タグは
Gateway イメージと `extended-stable*` エイリアスのみを公開します。この経路では
通常のオーケストレーターと、その ClawHub、ネイティブアプリ、GitHub Release、Web サイト、
および非公開 dist-tag の各サーフェスをスキップします。コマンドと復旧方法については、[Gateway の月次 extended-stable
公開](/ja-JP/reference/RELEASING#monthly-gateway-extended-stable-publication)
を参照してください。

## Runner

| Runner                          | ジョブ                                                                                                                                                                                                                                                                              |
| ------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `ubuntu-24.04`                  | `security-fast`、手動 CI ディスパッチおよび非正規リポジトリのフォールバック、QA Smoke 集約、CodeQL のセキュリティおよび品質スキャン、ワークフロー健全性チェック、ラベラー、自動応答、単独の Docs ワークフロー、Install Smoke ワークフロー全体                                |
| `blacksmith-4vcpu-ubuntu-2404`  | `preflight`、`pnpm-store-warmup`、`native-i18n`、QA Smoke CI を除く `checks-fast-core`、Plugin／チャネル契約シャード、バンドルされた低負荷の Linux Node シャードの大部分、`check-lint` を除く `check-*` レーン、選択された `check-additional-*` シャード、`check-docs`、および `skills-python` |
| `blacksmith-8vcpu-ubuntu-2404`  | 維持されている高負荷 Linux Node スイート、境界／拡張機能の負荷が高い `check-additional-*` シャード、および `android`                                                                                                                                                                             |
| `blacksmith-16vcpu-ubuntu-2404` | 自動 QA Smoke CI シャード、CI および Testbox の `build-artifacts`、ならびに `check-lint`（CPU 依存度が高く、8 vCPU では節約額を上回るコストが発生）                                                                                                                                  |
| `blacksmith-8vcpu-windows-2025` | `checks-windows`                                                                                                                                                                                                                                                                  |
| `blacksmith-6vcpu-macos-15`     | `openclaw/openclaw` 上の `macos-node`。フォークでは `macos-15` にフォールバック                                                                                                                                                                                                                |
| `blacksmith-12vcpu-macos-26`    | `openclaw/openclaw` 上の `macos-swift` および `ios-build`。フォークでは `macos-26` にフォールバック                                                                                                                                                                                               |

## Runner 登録予算

OpenClaw の現在の GitHub Runner 登録バケットでは、`ghx api rate_limit` において 5 分あたり 10,000 件のセルフホスト
Runner 登録が報告されています。GitHub が
このバケットを変更する可能性があるため、調整作業のたびに `actions_runner_registration` を再確認してください。
この上限は `openclaw` 組織内のすべての Blacksmith Runner 登録で共有されるため、
Blacksmith のインストールを追加しても新しいバケットは追加されません。

バースト制御では、Blacksmith ラベルを希少なリソースとして扱ってください。
ルーティング、通知、要約、シャード選択、または短時間の CodeQL スキャンのみを行うジョブは、
Blacksmith 固有の必要性が測定によって示されていない限り、GitHub ホスト Runner に
残す必要があります。新しい Blacksmith マトリクス、より大きな `max-parallel`、または高頻度の
ワークフローでは、最悪時の登録数を示し、組織レベルの
目標を実際のバケットの約 60% 未満に維持する必要があります。現在の 10,000 件の登録
バケットでは、これは運用目標を 6,000 件とし、同時実行される
リポジトリ、再試行、バーストの重複に備えて余裕を残すことを意味します。

変更対象に基づく PR プランにより、一般的な Node テストのバーストは Blacksmith 登録 14 件から 1 件に削減されます。広範なリスクを伴う PR では 14 件登録のコンパクトなフォールバックを維持するため、最悪時の件数は増加しません。

正規リポジトリの CI では、通常のプッシュおよびプルリクエスト実行に対するデフォルトの Runner 経路として Blacksmith を維持します。`workflow_dispatch` および非正規リポジトリの実行では GitHub ホスト Runner を使用しますが、通常の正規実行では、現在 Blacksmith のキュー状態を確認せず、Blacksmith が利用できない場合に GitHub ホストのラベルへ自動的にフォールバックすることもありません。

## サーフェスのラチェット

縮小のみを許可する 2 つの予算が、設定サーフェスを保護します。どちらも、同じ PR で
予算ファイルが意図的に更新されるまで、増加時に CI を失敗させます。また、クリーンアップによって実際の数が減少した場合は、
ラチェットの引き下げが必要です。

- `config/env-var-count-budget.txt` は、`src/`、`packages/`、および `extensions/`
  配下の本番ソースに存在する個別の `OPENCLAW_*` 名の数を制限します
  （テストと QA Lab は除外）。`node scripts/check-env-var-count.mjs` でチェックされます。
  環境変数を削除する場合は、同じ PR で数値を引き下げてください。追加する場合は
  設定サーフェスに関する決定となるため、PR 本文で正当化してください。
- `docs/.generated/config-baseline.counts.json` は、種類ごとの
  （コア／チャネル／Plugin）`openclaw.json` スキーマエントリ数を制限します。`pnpm config:docs:check` で
  チェックされます。スキーマを変更した後は、`pnpm config:docs:gen` で再生成してください。

## ローカルでの同等手順

```bash
pnpm changed:lanes                            # origin/main...HEAD に対するローカルの変更レーン分類を確認
pnpm check:changed                            # スマートなローカルチェックゲート：境界レーン別に変更されたフォーマット／型チェック／lint／ガードを検査
pnpm check                                    # 高速なローカルゲート：本番 tsgo + 分割 lint + 並列高速ガード
pnpm check:test-types
pnpm check:timed                              # ステージごとの所要時間を含む同じゲート
pnpm build:strict-smoke
pnpm check:architecture
pnpm test:gateway:watch-regression
OPENCLAW_TUI_PTY_INCLUDE_LOCAL=1 node scripts/run-vitest.mjs run --config test/vitest/vitest.tui-pty.config.ts
pnpm test                                     # vitest テスト
pnpm test:changed                             # 低コストでスマートな変更対象 Vitest
pnpm test:ui                                  # Control UI のユニット／ブラウザテストスイート
pnpm ui:i18n:check                            # 生成された Control UI ロケールの整合性（リリースゲート）
pnpm native:i18n:baseline                     # ソース管理のネイティブ抽出インベントリを更新
pnpm native:i18n:verify                       # ソースインベントリ + Android／Apple ローカライズの安全性
pnpm native:i18n:check                        # 翻訳済み／プラットフォーム生成物の厳密な整合性（リリースゲート）
pnpm test:channels
pnpm test:contracts:channels
pnpm check:docs                               # ドキュメントのフォーマット + lint + リンク切れ
pnpm build                                    # CI アーティファクト／スモークチェックが重要な場合に dist をビルド
pnpm ios:build                                # iOS アプリプロジェクトを生成してビルド
pnpm ci:timings                               # 最新の origin/main push CI 実行を要約
pnpm ci:timings:recent                        # 最近成功した main CI 実行を比較
node scripts/ci-run-timings.mjs <run-id>      # 経過時間、キュー時間、最も遅いジョブを要約
node scripts/ci-run-timings.mjs --latest-main # Issue／コメントのノイズを無視し、origin/main push CI を選択
node scripts/ci-run-timings.mjs --recent 10   # 最近成功した main CI 実行を比較
pnpm test:perf:groups --full-suite --allow-failures --output .artifacts/test-perf/baseline-before.json
pnpm test:perf:groups:compare .artifacts/test-perf/baseline-before.json .artifacts/test-perf/after-agent.json
pnpm test:startup:memory
pnpm test:extensions:memory -- --json .artifacts/openclaw-performance/source/mock-provider/extension-memory.json
pnpm perf:kova:summary --report .artifacts/kova/reports/mock-provider/report.json --output .artifacts/kova/summary.md
```

## OpenClaw のパフォーマンス

`OpenClaw Performance` は製品／ランタイムのパフォーマンスワークフローです。`main` で毎日実行され、手動でディスパッチすることもできます。

```bash
gh workflow run openclaw-performance.yml --ref main -f profile=diagnostic -f repeat=3
gh workflow run openclaw-performance.yml --ref main -f profile=smoke -f repeat=1 -f deep_profile=true -f live_openai_candidate=true
gh workflow run openclaw-performance.yml --ref main -f target_ref=v2026.5.2 -f profile=diagnostic -f repeat=3
```

通常、手動ディスパッチではワークフローの ref をベンチマークします。現在のワークフロー実装を使用してリリースタグまたは別のブランチをベンチマークするには、`target_ref` を設定します。公開されるレポートパスと最新ポインターはテスト対象の ref をキーとし、各 `index.md` には、テスト対象の ref／SHA、ワークフローの ref／SHA、Kova の ref、プロファイル、レーン認証モード、モデル、反復回数、シナリオフィルターが記録されます。

このワークフローは、固定されたリリースから OCM を、`openclaw/Kova` から固定された `kova_ref` 入力の Kova をインストールし、次の 3 つのレーンを実行します。

- `mock-provider`：決定論的な偽の OpenAI 互換認証を使用し、ローカルビルドのランタイムに対して Kova 診断シナリオを実行します。
- `mock-deep-profile`：起動、Gateway、エージェントターンのホットスポットに対する CPU／ヒープ／トレースのプロファイリングです。スケジュール時、または `deep_profile=true` を指定したディスパッチ時に実行されます。
- `live-openai-candidate`：実際の OpenAI `openai/gpt-5.6-luna` エージェントターンです。`OPENAI_API_KEY` が利用できない場合はスキップされます。スケジュール時、または `live_openai_candidate=true` を指定したディスパッチ時に実行されます。

モックプロバイダーレーンでは、Kova の実行後に OpenClaw ネイティブのソースプローブも実行します。対象は、デフォルト、チャンネルスキップ、内部フック、50 Plugin の起動ケースにおける Gateway の起動時間とメモリ、バンドルされた Plugin のインポート RSS、モック OpenAI `channel-chat-baseline` の反復 hello ループ、起動済み Gateway に対する CLI 起動コマンド、SQLite 状態スモークパフォーマンスプローブです。テスト対象の ref について以前に公開されたモックプロバイダーのソースレポートが利用可能な場合、ソースサマリーは現在の RSS 値とヒープ値をそのベースラインと比較し、RSS の大幅な増加を `watch` としてマークします。ソースプローブの Markdown サマリーはレポートバンドル内の `source/index.md` にあり、その隣に生の JSON があります。

各レーンは、CPU、ヒープ、トレース、圧縮された診断バンドルを含む完全な GitHub アーティファクトをアップロードします。別のパブリッシャージョブがそれらのアーティファクトをダウンロードして検証し、`openclaw/clawgrit-reports` のコンテンツのみにスコープを限定した短期有効の ClawSweeper GitHub App トークンを発行して、Git push ステップにのみ渡します。このジョブは、`report.json`、`report.md`、`index.md`、ソースプローブのアーティファクト、バンドルのメタデータ／チェックサムを `openclaw-performance/<tested-ref>/<run-id>-<attempt>/<lane>/` 配下にコミットします。完全な診断アーカイブは、リンクされた Actions アーティファクトに残ります。パブリッシャーは、push を試みる前に 50 MB を超えるレポートファイルを拒否します。現在のテスト対象 ref のポインターは `openclaw-performance/<tested-ref>/latest-<lane>.json` です。スケジュール実行と `profile=release` ディスパッチは、App トークンの作成またはレポートの公開に失敗すると失敗します。リリース以外の手動ディスパッチでは、公開は勧告扱いのままとなり、認証または公開に失敗した場合も GitHub アーティファクトを保持します。以前のソースベースラインは公開レポートリポジトリから匿名で取得されるため、ベースラインの取得成功はパブリッシャー認証の成功を証明しません。

## 完全なリリース検証

`Full Release Validation` は、「リリース前にすべてを実行する」ための手動統括ワークフローです。ブランチ、タグ、または完全なコミット SHA を受け取り、そのターゲットで手動の `CI` ワークフロー（Android を含む）をディスパッチし、リリース専用の Plugin／パッケージ／静的／Docker 検証のために `Plugin Prerelease` をディスパッチし、ターゲット SHA に対して `OpenClaw Performance` をディスパッチし、さらにインストールスモーク、パッケージ受け入れ、クロス OS パッケージチェック、QA Lab の整合性、Matrix、Telegram、およびゲート付きの Discord、WhatsApp、Slack レーンのために `OpenClaw Release Checks` をディスパッチします（勧告扱いの成熟度スコアカードのレンダリングは、`run_maturity_scorecard` によりオプトインできます）。stable および full プロファイルには、網羅的なライブ／E2E と Docker リリースパスの長時間検証が常に含まれます。beta プロファイルでは、`run_release_soak=true` を使用してオプトインできます。標準のパッケージ Telegram E2E は Package Acceptance 内で実行されるため、完全な候補では重複するライブポーラーを開始しません。公開後は、`release_package_spec` を渡すことで、リビルドせずにリリースチェック、Package Acceptance、Docker、クロス OS、Telegram で出荷済み npm パッケージを再利用できます。公開済みパッケージに対する Telegram の限定的な再実行にのみ `npm_telegram_package_spec` を使用してください。Codex Plugin のライブパッケージレーンも、デフォルトで同じ選択状態を使用します。公開済みの `release_package_spec=openclaw@<tag>` からは `codex_plugin_spec=npm:@openclaw/codex@<tag>` が導出され、SHA／アーティファクト実行では選択された ref から `extensions/codex` がパックされます。`npm:`、`npm-pack:`、`git:` の仕様など、カスタム Plugin ソースには `codex_plugin_spec` を明示的に設定します。そのライブエージェント検証は、目に見える進捗を送信し、ランダム化されたワークスペース読み取りと正確なアーティファクト書き込みを最後まで続行してから、完了を送信します。

ステージマトリクス、正確なワークフロージョブ名、プロファイルの違い、アーティファクト、限定的な再実行用ハンドルについては、[完全なリリース検証](/ja-JP/reference/full-release-validation)を参照してください。

`OpenClaw Release Publish` は、変更を伴う手動リリースワークフローです。リリースタグが存在し、OpenClaw npm の事前チェックが成功した後に、信頼済みの `main` から通常の beta および stable 公開をディスパッチします（事前チェックでは、チェック項目の一つとして `pnpm plugins:sync:check` が実行されます）。タグは引き続き、`release/YYYY.M.PATCH` 上のコミットを含む正確なリリースコミットを選択します。Tideclaw alpha の公開では、対応する alpha ブランチを引き続き使用します。このワークフローには、保存済みの `preflight_run_id`、成功した `full_release_validation_run_id`、およびその正確な `full_release_validation_run_attempt` が必要です。公開可能なすべての Plugin パッケージについて `Plugin NPM Release` をディスパッチし、同じリリース SHA について `Plugin ClawHub Release` をディスパッチした後にのみ、`OpenClaw NPM Release` をディスパッチします。stable 公開には、正確な `windows_node_tag` も必要です。ワークフローは、公開用の子ワークフローを開始する前に Windows ソースリリースを検証し、その x64／ARM64 インストーラーを候補として承認済みの `windows_node_installer_digests` 入力と比較します。その後、GitHub リリースドラフトを公開する前に、固定された同一のインストーラーダイジェストに加え、正確な付随アセットおよびチェックサムの契約を昇格して検証します。Plugin のみに限定した修復では、空でないパッケージリストを指定して `plugin_publish_scope=selected` を使用します。Plugin のみの `all-publishable` 実行には、コア公開と同じ不変の npm 事前チェックおよび完全なリリース検証の証拠が必要です。

```bash
gh workflow run openclaw-release-publish.yml \
  --ref main \
  -f tag=vYYYY.M.PATCH-beta.N \
  -f preflight_run_id=<successful-openclaw-npm-preflight-run-id> \
  -f full_release_validation_run_id=<successful-full-release-validation-run-id> \
  -f full_release_validation_run_attempt=<successful-full-release-validation-run-attempt> \
  -f npm_dist_tag=beta
```

更新の速いブランチ上で固定コミットを検証するには、`gh workflow run ... --ref main -f ref=<sha>` の代わりにヘルパーを使用します。

```bash
pnpm ci:full-release --sha <full-sha>
```

GitHub ワークフローのディスパッチ ref は、未加工のコミット SHA ではなく、ブランチまたはタグでなければなりません。ヘルパーは、信頼済みの `main` ワークフロー SHA に一時的な `release-ci/<sha>-...` ブランチを push し、要求されたターゲット SHA をワークフローの `ref` 入力経由で渡し、利用可能な場合は厳密な完全一致ターゲットの証拠を再利用し、すべての子ワークフローの `headSha` が信頼済みワークフロー SHA と一致することを検証し、実行完了時に一時ブランチを削除します。検証を新規に強制するには、`-f reuse_evidence=false` を渡します。統括ベリファイアーは、いずれかの子ワークフローが異なるワークフロー SHA で実行された場合にも失敗します。

`release_profile` は、リリースチェックに渡されるライブ／プロバイダーの対象範囲を制御します。手動リリースワークフローのデフォルトは `stable` です。広範な勧告扱いのプロバイダー／メディアマトリクスを意図的に実行する場合にのみ、`full` を使用してください。stable および full リリースチェックでは、網羅的なライブ／E2E と Docker リリースパスの長時間検証が常に実行されます。beta プロファイルでは、`run_release_soak=true` を使用してオプトインできます。

- `beta` は、最も高速な OpenAI／コアのリリース必須レーンを維持します。
- `stable` は、安定したプロバイダー／バックエンドのセットを追加します。
- `full` は、広範な勧告扱いのプロバイダー／メディアマトリクスを実行します。

統括ワークフローは、ディスパッチした子実行 ID を記録します。最後の `Verify full validation` ジョブは、子実行の現在の結果を再確認し、各子実行について最も遅いジョブの表を追記します。子ワークフローが再実行されて成功した場合は、親のベリファイアージョブのみを再実行して、統括結果と所要時間サマリーを更新します。

復旧時には、`Full Release Validation` と `OpenClaw Release Checks` の両方が `rerun_group` を受け付けます。リリース候補には `all`、通常の完全な CI 子ワークフローだけには `ci`、Plugin プレリリース子ワークフローだけには `plugin-prerelease`、OpenClaw Performance 子ワークフローだけには `performance`、すべてのリリース子ワークフローには `release-checks` を使用します。より限定されたグループとして、アンブレラで `install-smoke`、`cross-os`、`live-e2e`、`package`、`qa`、`qa-parity`、`qa-live`、または `npm-telegram` を使用することもできます。これにより、対象を絞った修正後に、失敗したリリースボックスの再実行範囲を限定できます。失敗したクロス OS レーンが 1 つだけの場合は、`rerun_group=cross-os` と `cross_os_suite_filter` を組み合わせます（例: `windows/packaged-upgrade`）。長時間実行されるクロス OS コマンドは Heartbeat 行を出力し、パッケージ化アップグレードのサマリーにはフェーズごとの所要時間が含まれます。選択された Matrix および Telegram QA レーンは、コアのランタイムペア用ツールカバレッジゲートと同様に、通常のリリース検証をブロックします。QA パリティ、ランタイムパリティ、およびゲート対象の Discord、WhatsApp、Slack ライブレーンは参考情報です。

`OpenClaw Release Checks` は、信頼済みのワークフロー参照を使用して、選択された参照を一度だけ `release-package-under-test` tarball に解決し、そのアーティファクトをクロス OS チェックと Package Acceptance に渡します。ソークカバレッジの実行時には、ライブ/E2E リリースパス Docker ワークフローにも渡します。これにより、リリースボックス間でパッケージのバイト列が一貫し、複数の子ジョブで同じ候補を再パッケージ化することを回避できます。Codex npm Plugin ライブレーンでは、リリースチェックは `release_package_spec` から導出された一致する公開済み Plugin 仕様を渡すか、オペレーター指定の `codex_plugin_spec` を渡すか、入力を空のままにして Docker スクリプトに選択されたチェックアウトの Codex Plugin をパッケージ化させます。

`ref=main` および `rerun_group=all` に対する重複した `Full Release Validation` 実行は、
古いアンブレラより優先されます。親モニターは、親がキャンセルされた時点で
すでにディスパッチ済みの子ワークフローをすべてキャンセルするため、新しい main の検証が
古い 2 時間のリリースチェック実行の後ろで待機することはありません。リリースブランチ/タグの
検証および対象を絞った再実行グループでは、`cancel-in-progress: false` が維持されます。

## ライブおよび E2E シャード

リリースのライブ/E2E 子ワークフローは広範なネイティブ `pnpm test:live` カバレッジを維持しますが、単一の直列ジョブではなく、`scripts/test-live-shard.mjs` を通じて名前付きシャードとして実行します。

- `native-live-src-agents` および `native-live-src-agents-zai-coding`
- `native-live-src-gateway-core`
- プロバイダーでフィルタリングされた `native-live-src-gateway-profiles` ジョブ
- `native-live-src-gateway-backends`
- `native-live-src-infra`
- `native-live-test`
- `native-live-extensions-a-k`
- `native-live-extensions-l-n`
- `native-live-extensions-moonshot`
- `native-live-extensions-openai`
- `native-live-extensions-o-z-other`
- `native-live-extensions-xai`
- 分割されたメディア音声/動画シャード、およびプロバイダーでフィルタリングされた音楽シャード

これにより、同じファイルカバレッジを維持しながら、低速なライブプロバイダーの失敗を再実行および診断しやすくなります。集約された `native-live-src-gateway`、`native-live-extensions-o-z`、`native-live-extensions-media`、および `native-live-extensions-media-music` シャード名は、手動の単発再実行でも引き続き有効です。

ネイティブのライブメディアシャードは、`Live Media Runner Image` ワークフローによってビルドされた `ghcr.io/openclaw/openclaw-live-media-runner:ubuntu-24.04` 内で実行されます。このイメージには `ffmpeg` と `ffprobe` がプリインストールされています。メディアジョブはセットアップ前にバイナリを確認するだけです。Docker ベースのライブスイートは通常の Blacksmith ランナー上で実行してください。コンテナジョブは、ネストされた Docker テストを起動する場所として不適切です。

Docker ベースのライブモデル/バックエンドシャードは、選択されたコミットごとに別個の共有 `ghcr.io/openclaw/openclaw-live-test:<sha>-<extensions>` イメージを使用します。ライブリリースワークフローはこのイメージを一度だけビルドしてプッシュし、その後、Docker ライブモデル、プロバイダー別にシャード化された Gateway、CLI バックエンド、ACP バインド、および Codex ハーネスの各シャードが `OPENCLAW_SKIP_DOCKER_BUILD=1` を使用して実行されます。Gateway Docker シャードでは、ワークフローのジョブタイムアウトより短い明示的なスクリプトレベルの `timeout` 上限を設定します。これにより、停止したコンテナやクリーンアップパスがリリースチェックの予算全体を消費せず、速やかに失敗します。これらのシャードが完全なソース Docker ターゲットを個別に再ビルドする場合、リリース実行の構成が誤っており、重複したイメージビルドに実時間を浪費します。

## Package Acceptance

「このインストール可能な OpenClaw パッケージは製品として動作するか」を確認する場合は、`Package Acceptance` を使用します。これは通常の CI とは異なります。通常の CI はソースツリーを検証しますが、Package Acceptance は、ユーザーがインストールまたは更新後に使用するものと同じ Docker E2E ハーネスを通じて、単一の tarball を検証します。

### ジョブ

1. `resolve_package` は `workflow_ref` をチェックアウトし、1 つのパッケージ候補を解決し、`.artifacts/docker-e2e-package/openclaw-current.tgz` と `.artifacts/docker-e2e-package/package-candidate.json` を書き込み、両方を `package-under-test` アーティファクトとしてアップロードし、ソース、ワークフロー参照、パッケージ参照、バージョン、SHA-256、およびプロファイルを GitHub ステップサマリーに出力します。
2. `package_integrity` は `package-under-test` アーティファクトをダウンロードし、`scripts/check-openclaw-package-tarball.mjs` を使用して公開パッケージ tarball のコントラクトを適用します。
3. `docker_acceptance` は、解決されたパッケージソース SHA（`workflow_ref` にフォールバック）と `package_artifact_name=package-under-test` を指定して `openclaw-live-and-e2e-checks-reusable.yml` を呼び出します。再利用可能なワークフローは、そのアーティファクトをダウンロードし、tarball のインベントリを検証し、必要に応じてパッケージダイジェスト Docker イメージを準備し、ワークフローのチェックアウトをパッケージ化する代わりに、そのパッケージに対して選択された Docker レーンを実行します。プロファイルが複数の対象 `docker_lanes` を選択した場合、再利用可能なワークフローはパッケージと共有イメージを一度だけ準備し、それらのレーンを一意のアーティファクトを持つ並列の対象別 Docker ジョブとして展開します。
4. `package_telegram` は、必要に応じて `NPM Telegram Beta E2E` を呼び出します。これは `telegram_mode` が `none` でない場合に実行され、Package Acceptance がパッケージを解決した場合は同じ `package-under-test` アーティファクトをインストールします。Telegram のスタンドアロンディスパッチでは、公開済み npm 仕様を引き続きインストールできます。
5. `summary` は、パッケージ解決、整合性、Docker Acceptance、または任意の Telegram レーンが失敗した場合にワークフローを失敗させます。`advisory` 入力は、参考用途の呼び出し元に対して Acceptance の失敗を警告に格下げします。

### 候補ソース

- `source=npm` は、`openclaw@extended-stable`、`openclaw@beta`、`openclaw@latest`、または `openclaw@2026.4.27-beta.2` のような OpenClaw の正確なリリースバージョンのみを受け付けます。公開済みの extended-stable、プレリリース、または stable の Acceptance に使用します。
- `source=ref` は、信頼済みの `package_ref` ブランチ、タグ、または完全なコミット SHA をパッケージ化します。リゾルバーは OpenClaw のブランチ/タグをフェッチし、選択されたコミットがリポジトリのブランチ履歴またはリリースタグから到達可能であることを検証し、切り離されたワークツリーに依存関係をインストールし、`scripts/package-openclaw-for-docker.mjs` を使用してパッケージ化します。
- `source=url` は、公開 HTTPS `.tgz` をダウンロードします。`package_sha256` は必須です。このパスでは、URL の認証情報、デフォルト以外の HTTPS ポート、プライベート/内部/特殊用途のホスト名または解決済み IP、および同じ公開安全ポリシーの範囲外へのリダイレクトを拒否します。
- `source=trusted-url` は、`.github/package-trusted-sources.json` 内の名前付き信頼済みソースポリシーから HTTPS `.tgz` をダウンロードします。`package_sha256` と `trusted_source_id` は必須です。設定済みのホスト、ポート、パスプレフィックス、リダイレクト先ホスト、またはプライベートネットワーク解決が必要な、メンテナー所有のエンタープライズミラーやプライベートパッケージリポジトリにのみ使用してください。ポリシーが Bearer 認証を宣言している場合、ワークフローは固定の `OPENCLAW_TRUSTED_PACKAGE_TOKEN` シークレットを使用します。URL に埋め込まれた認証情報は引き続き拒否されます。
- `source=artifact` は、`artifact_run_id` および `artifact_name` から 1 つの `.tgz` をダウンロードします。`package_sha256` は任意ですが、外部共有されるアーティファクトには指定する必要があります。

`workflow_ref` と `package_ref` は分けて扱ってください。`workflow_ref` はテストを実行する信頼済みのワークフロー/ハーネスコードです。`package_ref` は、`source=ref` の場合にパッケージ化されるソースコミットです。これにより、古いワークフローロジックを実行せずに、現在のテストハーネスで以前の信頼済みソースコミットを検証できます。

### スイートプロファイル

- `smoke` — `npm-onboard-channel-agent`、`gateway-network`、`config-reload`
- `package` — `npm-onboard-channel-agent`、`doctor-switch`、`update-channel-switch`、`skill-install`、`update-corrupt-plugin`、`upgrade-survivor`、`published-upgrade-survivor`、`root-managed-vps-upgrade`、`update-restart-auth`、`plugins-offline`、`plugin-update`
- `product` — `plugins-offline` の代わりにライブ `plugins` カバレッジを使用する `package` セットに、`mcp-channels`、`cron-mcp-cleanup`、`openai-web-search-minimal`、`openwebui` を追加
- `full` — OpenWebUI を含む完全な Docker リリースパスチャンク
- `custom` — 正確な `docker_lanes`。`suite_profile=custom` の場合は必須

`package` プロファイルはオフライン Plugin カバレッジを使用するため、公開パッケージの検証はライブ ClawHub の可用性によってゲートされません。任意の Telegram レーンは `NPM Telegram Beta E2E` 内の `package-under-test` アーティファクトを再利用し、スタンドアロンディスパッチでは公開済み npm 仕様のパスを維持します。

ローカルコマンド、Docker レーン、Package Acceptance の入力、リリースのデフォルト、および障害トリアージを含む専用の更新および Plugin テストポリシーについては、
[更新と Plugin のテスト](/ja-JP/help/testing-updates-plugins)を参照してください。

リリースチェックは、`source=artifact`、準備済みリリースパッケージアーティファクト、`suite_profile=custom`、`docker_lanes='doctor-switch update-channel-switch skill-install update-corrupt-plugin upgrade-survivor published-upgrade-survivor root-managed-vps-upgrade update-restart-auth plugins-offline plugin-update plugin-binding-command-escape'`、および `telegram_mode=mock-openai` を指定して Package Acceptance を呼び出します。これにより、パッケージ移行、更新、ライブ ClawHub Skill のインストール、古い Plugin 依存関係のクリーンアップ、設定済み Plugin のインストール修復、オフライン Plugin、Plugin 更新、および Telegram の検証が、同じ解決済みパッケージ tarball に対して行われます。ベータ公開後に Full Release Validation または OpenClaw Release Checks で `release_package_spec` を設定すると、再ビルドせずに出荷済み npm パッケージに対して同じマトリックスが実行されます。Package Acceptance でリリース検証の他の部分とは異なるパッケージが必要な場合にのみ、`package_acceptance_package_spec` を設定してください。クロス OS リリースチェックは、OS 固有のオンボーディング、インストーラー、およびプラットフォーム動作を引き続きカバーします。パッケージ/更新の製品検証は Package Acceptance から開始する必要があります。

`published-upgrade-survivor` Docker レーンは、ブロッキングリリースパスで実行ごとに 1 つの公開済みパッケージベースラインを検証します。Package Acceptance では、解決された `package-under-test` tarball が常に候補となり、`published_upgrade_survivor_baseline` がフォールバックの公開済みベースラインを選択します。デフォルトは `openclaw@latest` です。失敗したレーンの再実行コマンドでは、そのベースラインが維持されます。`run_release_soak=true` または `release_profile=full` を指定した Full Release Validation は、`published_upgrade_survivor_baselines='last-stable-4 2026.4.23 2026.5.2 2026.4.15'` と `published_upgrade_survivor_scenarios=reported-issues` を設定し、最新 4 つの stable npm リリースに加え、Plugin 互換性境界の固定リリース、および Feishu 設定、保持された bootstrap/persona ファイル、設定済み OpenClaw Plugin のインストール、チルダ付きログパス、古いレガシー Plugin 依存関係ルート向けの問題形状フィクスチャまで対象を拡張します。複数ベースラインの公開済みアップグレード生存者選択は、ベースラインごとにシャード化され、個別の対象別 Docker ランナージョブになります。別個の `Update Migration` ワークフローは、通常の Full Release CI の範囲ではなく、公開済み更新のクリーンアップを網羅的に確認する場合に、`all-since-2026.4.23` ベースラインと `plugin-deps-cleanup` シナリオを指定して `update-migration` Docker レーンを使用します。ローカルの集約実行では、`OPENCLAW_UPGRADE_SURVIVOR_BASELINE_SPECS` で正確なパッケージ仕様を渡すか、`openclaw@2026.4.15` のような `OPENCLAW_UPGRADE_SURVIVOR_BASELINE_SPEC` で単一レーンを維持するか、シナリオマトリックス用に `OPENCLAW_UPGRADE_SURVIVOR_SCENARIOS` を設定できます。公開済みレーンは組み込みの `openclaw config set` コマンドレシピを使用してベースラインを設定し、レシピのステップを `summary.json` に記録し、Gateway 起動後に `/healthz`、`/readyz`、および RPC ステータスをプローブします。Windows のパッケージ化済みレーンおよびインストーラーの新規インストールレーンでは、インストール済みパッケージが未加工の絶対 Windows パスからブラウザー制御オーバーライドをインポートできることも検証します。OpenAI クロス OS エージェントターンのスモークテストでは、`OPENCLAW_CROSS_OS_OPENAI_MODEL` が設定されている場合はそれを使用し、それ以外の場合は `openai/gpt-5.6-luna` をデフォルトとします。これにより、インストールと Gateway の検証で、より低コストの GPT-5.6 テストティアが使用されます。

### レガシー互換性期間

Package Acceptance には、すでに公開済みのパッケージに対する期間限定のレガシー互換性ウィンドウがあります。`2026.4.25-beta.*` を含む `2026.4.25` までのパッケージでは、互換性パスを使用できます。

- `dist/postinstall-inventory.json` 内の既知の非公開 QA エントリは、tarball から省略されたファイルを参照できます。
- パッケージがそのフラグを公開していない場合、`doctor-switch` は `gateway install --wrapper` 永続化サブケースをスキップできます。
- `update-channel-switch` は、tarball から派生した偽の git fixture から欠落している pnpm `patchedDependencies` を除外でき、欠落している永続化済み `update.channel` をログに記録できます。
- Plugin のスモークテストは、レガシーのインストールレコードの場所を読み取るか、マーケットプレイスのインストールレコードが永続化されていない状態を許容できます。
- `plugin-update` は、インストールレコードと再インストールなしの動作を変更しないことを引き続き必須としながら、設定メタデータの移行を許容できます。

公開済みの `2026.4.26` パッケージでは、すでに出荷されたローカルビルドメタデータのスタンプファイルについて警告することもでき、`2026.5.20` までのパッケージでは、`npm-shrinkwrap.json` が欠落している場合に失敗ではなく警告とすることができます。それ以降のパッケージは最新の契約を満たす必要があり、同じ条件では警告やスキップではなく失敗します。

### 例

```bash
# 現在のベータパッケージを製品レベルのカバレッジで検証します。
gh workflow run package-acceptance.yml \
  --ref main \
  -f workflow_ref=main \
  -f source=npm \
  -f package_spec=openclaw@beta \
  -f suite_profile=product \
  -f telegram_mode=mock-openai

# 公開済みの延長安定版パッケージをパッケージカバレッジで検証します。
gh workflow run package-acceptance.yml \
  --ref main \
  -f workflow_ref=main \
  -f source=npm \
  -f package_spec=openclaw@extended-stable \
  -f suite_profile=package \
  -f telegram_mode=mock-openai

# 現在のハーネスを使用してリリースブランチをパックし、検証します。
gh workflow run package-acceptance.yml \
  --ref main \
  -f workflow_ref=main \
  -f source=ref \
  -f package_ref=release/YYYY.M.PATCH \
  -f suite_profile=package \
  -f telegram_mode=mock-openai

# tarball URL を検証します。source=url では SHA-256 が必須です。
gh workflow run package-acceptance.yml \
  --ref main \
  -f workflow_ref=main \
  -f source=url \
  -f package_url=https://example.com/openclaw-current.tgz \
  -f package_sha256=<64-char-sha256> \
  -f suite_profile=smoke

# 名前付きの信頼済み非公開ミラーポリシーから tarball を検証します。
gh workflow run package-acceptance.yml \
  --ref main \
  -f workflow_ref=main \
  -f source=trusted-url \
  -f trusted_source_id=enterprise-artifactory \
  -f package_url=https://packages.example.internal:8443/artifactory/openclaw/openclaw-current.tgz \
  -f package_sha256=<64-char-sha256> \
  -f suite_profile=smoke

# 別の Actions 実行によってアップロードされた tarball を再利用します。
gh workflow run package-acceptance.yml \
  --ref main \
  -f workflow_ref=main \
  -f source=artifact \
  -f artifact_run_id=<run-id> \
  -f artifact_name=package-under-test \
  -f suite_profile=custom \
  -f docker_lanes='install-e2e plugin-update'
```

失敗した Package Acceptance 実行をデバッグする場合は、まず `resolve_package` の概要でパッケージのソース、バージョン、SHA-256 を確認します。次に、`docker_acceptance` の子実行と、その Docker アーティファクト（`.artifacts/docker-tests/**/summary.json`、`failures.json`、レーンログ、フェーズ時間、再実行コマンド）を調査します。完全なリリース検証を再実行するのではなく、失敗したパッケージプロファイルまたは正確な Docker レーンを再実行することを推奨します。

## インストールスモークテスト

`Install Smoke` ワークフローは、プルリクエストまたは `main` へのプッシュでは実行されなくなりました。夜間／手動ラッパーとリリース検証は、どちらも読み取り専用の `install-smoke-reusable.yml` コアを呼び出し、すべての実行で GitHub ホストランナー上の完全なインストールスモークテストパスを使用します。

- ルート Dockerfile のスモークイメージはターゲット SHA ごとに一度だけビルドされ、変更不能なアーティファクト内でワークフローリビジョンおよび生成元の試行に紐付けられた後、CLI スモークテスト、エージェントの共有ワークスペース削除 CLI スモークテスト、コンテナ Gateway ネットワーク E2E、同梱 `matrix` Plugin のビルド引数スモークテストによって読み込まれます。Plugin スモークテストは、ランタイム依存関係のインストールミラーリングと、エントリエスケープ診断なしで Plugin が読み込まれることを検証します。
- QR パッケージのインストールおよびインストーラー／更新 Docker スモークテスト（Rocky Linux インストーラーレーンと、設定可能な `update_baseline_version` npm ベースラインに対する更新レーンを含む）は別々のジョブとして実行されるため、インストーラー処理がルートイメージのスモークテストを待つことはありません。

低速な Bun グローバルインストールのイメージプロバイダースモークテストは、`run_bun_global_install_smoke` によって個別に制御されます。これは夜間スケジュールで実行され、リリースチェックからのワークフロー呼び出しではデフォルトで有効になり、手動の `Install Smoke` ディスパッチではオプトインできます。通常の PR CI では、Node に関連する変更に対して高速な Bun ランチャー回帰レーンが引き続き実行されます。QR およびインストーラーの Docker テストは、独自のインストール専用 Dockerfile を維持します。

## ローカル Docker E2E

`pnpm test:docker:all` は、共有ライブテストイメージを 1 つ事前ビルドし、OpenClaw を npm tarball として一度だけパックして、2 つの共有 `scripts/e2e/Dockerfile` イメージをビルドします。

- インストーラー／更新／Plugin 依存関係レーン用の最小構成の Node/Git ランナー。
- 通常の機能レーン向けに、同じ tarball を `/app` にインストールする機能イメージ。

Docker レーン定義は `scripts/lib/docker-e2e-scenarios.mjs` に、プランナーのロジックは `scripts/lib/docker-e2e-plan.mjs` にあり、ランナーは選択されたプランのみを実行します。スケジューラーは `OPENCLAW_DOCKER_E2E_BARE_IMAGE` および `OPENCLAW_DOCKER_E2E_FUNCTIONAL_IMAGE` を使用してレーンごとにイメージを選択し、`OPENCLAW_SKIP_DOCKER_BUILD=1` でレーンを実行します。

### 調整可能項目

| 変数                               | デフォルト | 目的                                                                                       |
| -------------------------------------- | ------- | --------------------------------------------------------------------------------------------- |
| `OPENCLAW_DOCKER_ALL_PARALLELISM`      | 10      | 通常レーン用のメインプールのスロット数。                                                        |
| `OPENCLAW_DOCKER_ALL_TAIL_PARALLELISM` | 10      | プロバイダー依存のテールプールのスロット数。                                                      |
| `OPENCLAW_DOCKER_ALL_LIVE_LIMIT`       | 9       | プロバイダーによるスロットリングを防ぐための同時ライブレーン上限。                                        |
| `OPENCLAW_DOCKER_ALL_NPM_LIMIT`        | 5       | 同時 npm インストールレーン上限。                                                              |
| `OPENCLAW_DOCKER_ALL_SERVICE_LIMIT`    | 7       | 同時マルチサービスレーン上限。                                                            |
| `OPENCLAW_DOCKER_ALL_START_STAGGER_MS` | 2000    | Docker デーモンでの作成集中を避けるためのレーン開始間隔。間隔をなくすには `0` を設定します。     |
| `OPENCLAW_DOCKER_ALL_LANE_TIMEOUT_MS`  | 7200000 | レーンごとのフォールバックタイムアウト（120 分）。選択されたライブ／テールレーンではより厳しい上限が使用されます。           |
| `OPENCLAW_DOCKER_ALL_DRY_RUN`          | 未設定   | `1` はレーンを実行せずにスケジューラープランを出力します。                                          |
| `OPENCLAW_DOCKER_ALL_LANES`            | 未設定   | カンマ区切りの正確なレーン一覧。エージェントが失敗した 1 つのレーンを再現できるよう、クリーンアップスモークテストをスキップします。 |

実効上限より重いレーンでも、空のプールから開始でき、その後は容量を解放するまで単独で実行されます。ローカル集約処理は Docker を事前検査し、古い OpenClaw E2E コンテナを削除し、アクティブなレーンの状態を出力し、最長優先の順序付けのためにレーン時間を永続化します。また、デフォルトでは最初の失敗後に新しいプールレーンのスケジューリングを停止します。

### 再利用可能なライブ／E2E ワークフロー

再利用可能なライブ／E2E ワークフローは、必要なパッケージ、イメージ種別、ライブイメージ、レーン、および認証情報のカバレッジを `scripts/test-docker-all.mjs --plan-json` に問い合わせます。その後、`scripts/docker-e2e.mjs` がそのプランを GitHub の出力と概要に変換します。これは、`scripts/package-openclaw-for-docker.mjs` を介して OpenClaw をパックするか、現在の実行のパッケージアーティファクトをダウンロードするか、`package_artifact_run_id` からパッケージアーティファクトをダウンロードしてから、tarball の内容一覧を検証します。デフォルトの `no-push-artifact` パスは、Blacksmith の Docker レイヤーキャッシュを使用してパッケージダイジェストでタグ付けされた最小構成／機能イメージをビルドし、正確なイメージバイトを変更不能なワークフローアーティファクトにパックして、各コンシューマーにそのアーティファクトを検証させて読み込ませます。代わりに `existing-only` を使用する場合、明示的な `docker_e2e_bare_image`/`docker_e2e_functional_image` GHCR 参照が必須となり、ビルドやプッシュは一切行われません。これらのレジストリプルでは、試行ごとに 180 秒の制限付きタイムアウトを使用するため、ストリームが停止しても CI のクリティカルパスの大部分を消費せず、迅速に再試行されます。スケジュールされた検証が成功すると、`openclaw-scheduled-live-checks.yml` は変更不能なテスト済みイメージマニフェストを別のパッケージ書き込みパブリッシャーへ渡します。読み取り専用のリリースおよびプレリリース呼び出し元は、そのライターを経由しません。

### リリースパスのチャンク

リリース Docker カバレッジは、`OPENCLAW_SKIP_DOCKER_BUILD=1` を使用して小さく分割されたジョブを実行します。これにより、各チャンクは必要なアーティファクトベースのイメージ種別のみを検証して読み込むか、明示的な `existing-only` 再利用の場合はプルし、同じ重み付きスケジューラーを介して複数のレーンを実行します。

- `OPENCLAW_DOCKER_ALL_PROFILE=release-path`
- `OPENCLAW_DOCKER_ALL_CHUNK=core | package-update-openai | package-update-anthropic | package-update-core | plugins-runtime-plugins | plugins-runtime-services | plugins-runtime-install-a..h | openwebui`

現在のリリース Docker チャンクは、`core`、`package-update-openai`、`package-update-anthropic`、`package-update-core`、`plugins-runtime-plugins`、`plugins-runtime-services`、`plugins-runtime-install-a` から `plugins-runtime-install-h`、および `openwebui` です。`package-update-openai` にはライブ Codex Plugin パッケージレーンが含まれます。このレーンは、候補の OpenClaw パッケージをインストールし、Codex CLI のインストールを明示的に承認したうえで `codex_plugin_spec` または同じ参照の tarball から Codex Plugin をインストールし、Codex CLI の事前検査と同一セッション内のエージェントターンを実行します。その後、再試行なしの中程度の思考ターンを実行し、進行状況を送信し、ランダム化されたワークスペース入力を読み取り、それらと完全に一致するアーティファクトを書き込み、完了を送信します。`plugins-runtime-core`、`plugins-runtime`、および `plugins-integrations` は、引き続き Plugin／ランタイムの集約エイリアスです。`install-e2e` レーンエイリアスは、両方のプロバイダーインストーラーレーンを対象とする集約手動再実行エイリアスとして維持されます。

OpenWebUI は、安定版または完全なリリースパスカバレッジで要求された場合、再利用可能なワークフローが対応ジョブを GitHub ホストランナーに振り分ける場合でも、専用の大容量ディスク Blacksmith ランナー上で独立した `openwebui` チャンクとして実行されます。外部イメージのプルを分離することで、大容量イメージが `plugins-runtime-services` 内の共有パッケージおよび Plugin イメージと競合するのを防ぎます。レガシーの集約 Plugin／ランタイムチャンクには、互換性のある手動再実行のために OpenWebUI が引き続き含まれます。同梱チャンネルの更新レーンは、一時的な npm ネットワーク障害に対して 1 回再試行します。

各チャンクは、レーンログ、時間、`summary.json`、`failures.json`、フェーズ時間、スケジューラープラン JSON、低速レーンの表、レーンごとの再実行コマンドを含む `.artifacts/docker-tests/` をアップロードします。ワークフローの `docker_lanes` 入力は、チャンクジョブの代わりに、その実行用に準備されたイメージに対して選択されたレーンを実行します。これにより、失敗したレーンのデバッグが 1 つの対象 Docker ジョブに限定されます。選択されたレーンがライブ Docker レーンの場合、対象ジョブはその再実行用のライブテストイメージをローカルでビルドします。内部の再利用可能ワークフローのパッケージタプルは `workflow_dispatch` スキーマの一部ではないため、再実行ヘルパーは失敗アーティファクトで正確に選択されたターゲット SHA を検証し、手動ディスパッチはその参照を再パックします。生成されるコマンドには、準備済みイメージの入力と、それらの入力が GHCR ベースの場合に限り `shared_image_policy=existing-only` が含まれます。ランナーローカルのアーティファクトタグは省略されるため、新しいランナーでは再ビルドされます。明示的なターゲットの上書きでは、アーティファクトが上書き対象との一致を証明しない限り、復元された GHCR イメージ参照を破棄します。完全リリース用の一時ブランチは削除されるため、アーティファクトから生成されるワークフロー定義の参照も省略されます。オペレーターが明示的に上書きしない限り、ディスパッチにはリポジトリのデフォルトブランチが使用されます。

```bash
pnpm test:docker:rerun <run-id>      # Docker アーティファクトをダウンロードし、統合された／レーンごとの対象再実行コマンドを出力します
pnpm test:docker:timings <summary>   # 低速レーンとフェーズのクリティカルパスの概要
```

スケジュールされたライブ／E2E ワークフローは、完全なリリースパス Docker スイートを毎日実行し、成功後に、正確なテスト済みイメージアーティファクト用の明示的なパブリッシャーを呼び出します。

## Plugin プレリリース

`Plugin Prerelease` は、より高コストな製品/パッケージのカバレッジであるため、`Full Release Validation` または明示的なオペレーターによってディスパッチされる別個のワークフローです。通常のプルリクエスト、`main` へのプッシュ、単独の手動 CI ディスパッチでは、このスイートは実行されません。バンドルされた Plugin のテストを 8 つの拡張ワーカーに分散します。これらの拡張シャードジョブは、一度に最大 2 つの Plugin 構成グループを、グループごとに 1 つの Vitest ワーカーと、より大きな Node ヒープを使用して実行するため、インポート負荷の高い Plugin バッチによって余分な CI ジョブが作成されることはありません。リリース専用の Docker プレリリースパス（`full_release_validation` 入力で有効化）は、対象の Docker レーンを 4 つずつのグループにまとめ、1～3 分のジョブのために数十台のランナーを確保することを避けます。また、このワークフローは `@openclaw/plugin-inspector` から情報提供用の `plugin-inspector-advisory` アーティファクトをアップロードします。インスペクターの検出結果はトリアージの入力であり、ブロッキング対象の Plugin プレリリースゲートには影響しません。

## QA Lab

QA Lab には、メインのスマートスコープワークフローとは別に専用の CI レーンがあります。エージェント型パリティは、単独の PR ワークフローではなく、広範な QA およびリリースハーネス内に組み込まれています。広範な検証実行にパリティを含める場合は、`rerun_group=qa-parity` とともに `Full Release Validation` を使用します。

- `QA-Lab - All Lanes` ワークフローは、`main` で毎晩および手動ディスパッチ時に実行され、モックパリティに加えて、実環境の Matrix、Telegram、Discord、WhatsApp、Slack ジョブへファンアウトします。実環境ジョブは `qa-live-shared` 環境を使用します。Telegram、Discord、WhatsApp、Slack は Convex リースを使用し、Matrix は使い捨てのローカル認証情報をプロビジョニングします。

リリースチェックでは、決定論的なモックプロバイダーとモック用モデル（`mock-openai/gpt-5.6-luna` および `mock-openai/gpt-5.6-luna-alt`）を使用して、Matrix と Telegram の実環境トランスポートレーンを実行します。これにより、チャンネル契約が実環境モデルのレイテンシや通常のプロバイダー Plugin 起動から分離されます。QA パリティではメモリ動作を別途カバーするため、実環境トランスポートの Gateway ではメモリ検索が無効化されています。プロバイダー接続性は、個別の実環境モデル、ネイティブプロバイダー、Docker プロバイダーの各スイートでカバーされます。

スケジュール済みおよびリリース用の Matrix ゲートは、共有 QA Lab スイートホストと実環境アダプターをリリースシナリオとともに使用します。CLI のデフォルトおよび手動ワークフロー入力は引き続き `all` です。手動の `all` ディスパッチは、`transport`、`media`、`e2ee-smoke`、`e2ee-deep`、`e2ee-cli` の各プロファイルにファンアウトし、93 シナリオの検証がジョブごとのタイムアウト内に収まるようにします。対象を絞った手動ディスパッチでは、1 つのジョブで `fast`、`release`、`transport` のいずれかを選択します。

`OpenClaw Release Checks` もリリース承認前にリリースクリティカルな QA Lab レーンを実行します。その QA パリティゲートは、候補パックとベースラインパックを並列レーンジョブとして実行し、その後、最終的なパリティ比較のために両方のアーティファクトを小規模なレポートジョブへダウンロードします。

通常の PR では、パリティを必須ステータスとして扱うのではなく、スコープされた CI/チェックのエビデンスに従ってください。

## CodeQL

`CodeQL` ワークフローは、リポジトリ全体のスキャンではなく、意図的に範囲を絞った初回セキュリティスキャナーです。日次、手動、`main` へのプッシュ、およびドラフトではないプルリクエストのガード実行では、Actions ワークフローコードに加え、最もリスクの高い JavaScript/TypeScript サーフェスを、高/重大の `security-severity` に絞り込んだ高信頼度セキュリティクエリでスキャンします。

プルリクエストガードは軽量に保たれています。`.github/actions`、`.github/codeql`、`.github/workflows`、`packages`、`scripts`、`src`、またはプロセスを所有するバンドル済み Plugin のランタイムパス配下に変更がある場合にのみ開始し、スケジュール済みワークフローと同じ高信頼度セキュリティマトリクスを実行します。Android と macOS の CodeQL は PR のデフォルト対象外です。

### セキュリティカテゴリ

| カテゴリ                                          | サーフェス                                                                                                                             |
| ------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------- |
| `/codeql-security-high/core-auth-secrets`         | 認証、シークレット、サンドボックス、Cron、Gateway のベースライン                                                                                  |
| `/codeql-security-high/channel-runtime-boundary`  | コアチャンネル実装の契約に加え、チャンネル Plugin ランタイム、Gateway、Plugin SDK、シークレット、監査の接点              |
| `/codeql-security-high/network-ssrf-boundary`     | コアの SSRF、IP 解析、ネットワークガード、Web フェッチ、Plugin SDK の SSRF ポリシーサーフェス                                                |
| `/codeql-security-high/mcp-process-tool-boundary` | MCP サーバー、プロセス実行ヘルパー、アウトバウンド配信、エージェントツール実行ゲート                                           |
| `/codeql-security-high/process-exec-boundary`     | ローカルシェル、プロセス生成ヘルパー、サブプロセスを所有するバンドル済み Plugin ランタイム、ワークフロースクリプトの連携部分                             |
| `/codeql-security-high/plugin-trust-boundary`     | Plugin のインストール、ローダー、マニフェスト、レジストリ、パッケージマネージャーによるインストール、ソース読み込み、Plugin SDK パッケージ契約の信頼境界 |

### プラットフォーム固有のセキュリティシャード

- `CodeQL Android Critical Security` — スケジュール実行される Android セキュリティシャード。ワークフロー健全性チェックで許容される最小の Blacksmith Linux ランナー上で、CodeQL 用に Android アプリを手動ビルドします。`/codeql-critical-security/android` としてアップロードします。
- `CodeQL macOS Critical Security` — 週次/手動の macOS セキュリティシャード。Blacksmith macOS 上で CodeQL 用に macOS アプリを手動ビルドし、アップロードする SARIF から依存関係のビルド結果を除外して、`/codeql-critical-security/macos` としてアップロードします。問題がない場合でも macOS ビルドが実行時間の大部分を占めるため、日次のデフォルト対象外としています。

### 重大な品質カテゴリ

`CodeQL Critical Quality` は、対応する非セキュリティシャードです。GitHub ホステッド Linux ランナー上の範囲を絞った高価値サーフェスに対して、エラー重大度かつ非セキュリティの JavaScript/TypeScript 品質クエリのみを実行するため、品質スキャンが Blacksmith のランナー登録予算を消費することはありません。プルリクエストガードは、意図的にスケジュール済みプロファイルより小さく設定されています。ドラフトではない PR では、変更されたサーフェスに対応するシャードのみを、PR でルーティング可能な 13 個のシャード（`agent-runtime-boundary`、`channel-runtime-boundary`、`config-boundary`、`core-auth-secrets`、`gateway-runtime-boundary`、`mcp-process-runtime-boundary`、`memory-runtime-boundary`、`network-runtime-boundary`、`plugin-boundary`、`plugin-sdk-package-contract`、`plugin-sdk-reply-runtime`、`provider-runtime-boundary`、`session-diagnostics-boundary`）から実行します。`ui-control-plane` と `web-media-runtime-boundary` は PR 実行の対象外です。CodeQL 構成および品質ワークフローの変更では、PR シャードセット全体を実行します（ネットワークランタイムシャードは、それ自身の CodeQL 構成ファイルとネットワークを所有するソースパスに基づいて作動します）。

手動ディスパッチでは、次を指定できます。

```text
profile=all|agent-runtime-boundary|config-boundary|core-auth-secrets|channel-runtime-boundary|gateway-runtime-boundary|memory-runtime-boundary|mcp-process-runtime-boundary|network-runtime-boundary|plugin-boundary|plugin-sdk-package-contract|plugin-sdk-reply-runtime|provider-runtime-boundary|session-diagnostics-boundary
```

範囲を絞ったプロファイルは、1 つの品質シャードを分離して実行するための学習/反復用フックです。

| カテゴリ                                                | サーフェス                                                                                                                                                           |
| ------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `/codeql-critical-quality/core-auth-secrets`            | 認証、シークレット、サンドボックス、Cron、Gateway のセキュリティ境界コード                                                                                                  |
| `/codeql-critical-quality/config-boundary`              | 構成スキーマ、移行、正規化、IO 契約                                                                                                         |
| `/codeql-critical-quality/gateway-runtime-boundary`     | Gateway プロトコルスキーマとサーバーメソッド契約                                                                                                              |
| `/codeql-critical-quality/channel-runtime-boundary`     | コアチャンネルおよびバンドル済みチャンネル Plugin の実装契約                                                                                                  |
| `/codeql-critical-quality/agent-runtime-boundary`       | コマンド実行、モデル/プロバイダーのディスパッチ、自動返信のディスパッチとキュー、ACP コントロールプレーンのランタイム契約                                               |
| `/codeql-critical-quality/mcp-process-runtime-boundary` | MCP サーバーとツールブリッジ、プロセス監視ヘルパー、アウトバウンド配信契約                                                                        |
| `/codeql-critical-quality/memory-runtime-boundary`      | メモリホスト SDK、メモリランタイムファサード、メモリ Plugin SDK エイリアス、メモリランタイム有効化の連携部分、メモリ doctor コマンド                                    |
| `/codeql-critical-quality/network-runtime-boundary`     | ネットワークポリシーパッケージ、raw ソケットおよびプロキシキャプチャのランタイム、SSH トンネル、Gateway ロック、JSONL ソケット、プッシュトランスポートのサーフェス                                 |
| `/codeql-critical-quality/session-diagnostics-boundary` | 返信キューの内部実装、セッション配信キュー、アウトバウンドセッションのバインディング/配信ヘルパー、診断イベント/ログバンドルのサーフェス、セッション doctor CLI 契約 |
| `/codeql-critical-quality/plugin-sdk-reply-runtime`     | Plugin SDK のインバウンド返信ディスパッチ、返信ペイロード/チャンク化/ランタイムヘルパー、チャンネル返信オプション、配信キュー、セッション/スレッドのバインディングヘルパー             |
| `/codeql-critical-quality/provider-runtime-boundary`    | モデルカタログの正規化、プロバイダー認証と検出、プロバイダーランタイム登録、プロバイダーのデフォルト/カタログ、Web/検索/フェッチ/埋め込みレジストリ    |
| `/codeql-critical-quality/ui-control-plane`             | Control UI のブートストラップ、ローカル永続化、Gateway 制御フロー、タスクコントロールプレーンのランタイム契約                                                          |
| `/codeql-critical-quality/web-media-runtime-boundary`   | コアの Web フェッチ/検索、メディア IO、メディア理解、画像生成、メディア生成のランタイム契約                                                    |
| `/codeql-critical-quality/plugin-boundary`              | ローダー、レジストリ、公開サーフェス、Plugin SDK エントリポイントの契約                                                                                             |
| `/codeql-critical-quality/plugin-sdk-package-contract`  | 公開パッケージ側の Plugin SDK ソースと Plugin パッケージ契約ヘルパー                                                                                      |

品質をセキュリティから分離することで、セキュリティシグナルを不明瞭にすることなく、品質の検出結果をスケジュール、測定、無効化、拡張できます。Swift、Python、バンドル済み Plugin の CodeQL 拡張は、範囲を絞ったプロファイルの実行時間とシグナルが安定した後にのみ、スコープされた、またはシャード化されたフォローアップ作業として追加し直す必要があります。

## メンテナンスワークフロー

### Docs Agent

`Docs Agent` ワークフローは、最近マージされた変更に既存ドキュメントを整合させ続けるための、イベント駆動型 Codex メンテナンスレーンです。純粋なスケジュール実行はありません。`main` 上でボット以外によるプッシュの CI 実行が成功するとトリガーでき、手動ディスパッチでも直接実行できます。ワークフロー実行による呼び出しは、`main` が先へ進んでいる場合、またはスキップされていない別の Docs Agent 実行が過去 1 時間以内に作成されている場合にスキップされます。実行時には、前回スキップされなかった Docs Agent のソース SHA から現在の `main` までのコミット範囲をレビューするため、1 時間ごとの 1 回の実行で、前回のドキュメント確認以降に蓄積された main のすべての変更をカバーできます。

### テストパフォーマンスエージェント

`Test Performance Agent` ワークフローは、遅いテスト向けのイベント駆動型 Codex メンテナンスレーンです。純粋なスケジュール実行はありません。`main` で bot 以外による push の CI 実行が成功するとトリガーされる場合がありますが、その UTC 日付内に別の workflow-run 呼び出しがすでに実行済みまたは実行中の場合はスキップされます。手動ディスパッチでは、この日次アクティビティゲートを迂回します。このレーンは、フルスイートをグループ化した Vitest パフォーマンスレポートを作成し、Codex が広範なリファクタリングではなく、カバレッジを維持する小規模なテストパフォーマンス修正のみを行えるようにします。その後、フルスイートレポートを再実行し、合格したベースラインテスト数を減らす変更を拒否します。グループ化されたレポートには、Linux と macOS での config ごとの実経過時間と最大 RSS が記録されるため、変更前後の比較では、所要時間の差分と並んでテストメモリの差分も表示されます。ベースラインに失敗するテストがある場合、Codex が修正できるのは明白な失敗のみであり、何かをコミットする前に、エージェント実行後のフルスイートレポートが合格しなければなりません。bot の push が反映される前に `main` が進んだ場合、このレーンは検証済みパッチをリベースし、`pnpm check:changed` を再実行して、push を再試行します。競合する古いパッチはスキップされます。GitHub ホストの Ubuntu を使用するため、Codex action は docs agent と同じ sudo 削除の安全方針を維持できます。

### マージ後の重複 PR

`Duplicate PRs After Merge` ワークフローは、ランディング後に重複を整理するためのメンテナー向け手動ワークフローです。デフォルトではドライランとなり、`apply=true` の場合にのみ、明示的に列挙された PR をクローズします。GitHub を変更する前に、ランディング済み PR がマージされていること、および各重複 PR に参照先として共通する issue または重複する変更ハンクがあることを検証します。

```bash
gh workflow run duplicate-after-merge.yml \
  -f landed_pr=70532 \
  -f duplicate_prs='70530,70592' \
  -f apply=true
```

## ローカルチェックゲートと変更ルーティング

### Config ベースライン数のラチェット

`pnpm config:docs:check` は、文書化されていない config サーフェスの増加、および破損または古くなった件数スナップショットを拒否します。レビュー済みのプロダクト変更によって意図的にスキーマパスを追加する場合は、`pnpm config:docs:gen` を実行し、core/channel/plugin の件数差分と生成された SHA-256 ファイルを確認してから、スキーマ、ヘルプ、ラベル、マイグレーション、テストとともに、意図したベースラインの引き上げをコミットします。ラチェットを迂回するために件数ファイルを手動編集しないでください。

Config 作成者は、新しいリーフを Settings 用に階層分けする必要もあります。リーフに `advanced: false` または
`advanced: true` を追加するか、すべての子孫に継承させる階層を持つ祖先の下にキーを配置します。分類されていないルートがあると、コピー＆ペースト可能なスタブを伴ってスキーマ品質
テストが失敗します。祖先のないパスはデフォルトで advanced になります。
選定された共通リーフのスナップショットにより、意図的な階層変更がレビューで可視化されます。

ローカルの変更レーンロジックは `scripts/changed-lanes.mjs` にあり、`scripts/check-changed.mjs` によって実行されます。このローカルチェックゲートでは、広範な CI プラットフォームスコープよりもアーキテクチャ境界が厳格に扱われます。

- core 本番コードの変更では、core prod と core test の型チェックに加え、core lint/guards を実行します。
- core のテストのみの変更では、core test の型チェックと core lint のみを実行します。
- extension 本番コードの変更では、extension prod と extension test の型チェックに加え、extension lint を実行します。
- extension のテストのみの変更では、extension test の型チェックと extension lint を実行します。
- 公開 Plugin SDK または plugin 契約の変更では、extensions がこれらの core 契約に依存するため、extension の型チェックまで拡張します（Vitest の extension スイープは、引き続き明示的なテスト作業です）。
- リリースメタデータのみのバージョン更新では、対象を絞ったバージョン/config/ルート依存関係チェックを実行します。
- 不明なルート/config 変更は、安全側に倒してすべてのチェックレーンを実行します。

ローカルの変更テストルーティングは `scripts/test-projects.test-support.mjs` にあり、意図的に `check:changed` より低コストになっています。テストの直接編集ではそのテスト自体を実行し、ソース編集では明示的なマッピングを優先した後、同階層のテストとインポートグラフ上の依存先を実行します。共有グループルーム配信 config は、明示的なマッピングの一つです。グループの可視返信 config、ソース返信の配信モード、またはメッセージツールのシステムプロンプトに対する変更は、core の返信テストに加え、Discord と Slack の配信回帰テストへルーティングされるため、共有デフォルトの変更は最初の PR push 前に失敗します。変更がハーネス全体に及び、低コストのマッピング済みセットを信頼できる代替指標として扱えない場合にのみ、`OPENCLAW_TEST_CHANGED_BROAD=1 pnpm test:changed` を使用してください。

## Testbox による検証

Crabbox は、メンテナーによる Linux 検証用のリポジトリ所有リモートボックスラッパーです。エージェント
セッションでは、既存の依存関係インストールが利用可能な場合に限り、信頼済みソースに対する少数の対象を絞ったテストと低コストな静的チェックのみをローカルで実行します。より大規模なスイートと、
ビルド、型チェック、lint のファンアウト、Docker、パッケージレーン、E2E、ライブ検証、CI 同等性などの
計算負荷が高い作業には Crabbox を使用します。信頼済みメンテナーの高負荷な
検証にはデフォルトで `blacksmith-testbox` を使用し、`.crabbox.yaml` も現在はこれをデフォルトとします。設定済みの
ワークフローはプロバイダーとエージェントの認証情報を注入するため、信頼されていないコントリビューターまたは
fork のコードでは、代わりに secretless fork CI またはサニタイズ済みの直接 AWS Crabbox を使用する必要があります。
サニタイズ済み AWS 実行では `CRABBOX_ENV_ALLOW=CI` を設定し、
`--no-hydrate` を渡し、新しい一時的なリモート `HOME` を使用します。これにより、リポジトリの
`OPENCLAW_*` 許可リストと既存の認証プロファイルが信頼されていないコードに届くのを防ぎます。
その信頼されていないソース専用に新しくウォームアップしたリースを使用し、
信頼済みまたは以前に認証情報が注入されたリースは決して使用しません。クリーンで信頼済みの `main` checkout から、インストール済みの信頼済み Crabbox
バイナリを起動し、`--fresh-pr` を使用してリモート PR のみを取得します。信頼されていない checkout のラッパーや config は、ローカルで決して実行しないでください。
`CRABBOX_AWS_INSTANCE_PROFILE` を未設定にし、解決済みの
`aws.instanceProfile` が空でない限り、安全側に倒して失敗させます。インストールやテストを行う前に、信頼済みの
絶対パスツールを使用して IMDSv2 トークンを必須とし、IAM 認証情報
エンドポイントが 404 を返すことを証明し、リモートの `git rev-parse HEAD` をレビュー済み PR の完全な
head SHA と比較します。リースをその SHA に紐付け、head が変わったら停止して再ウォームアップします。
クリーンな `main` から信頼済みの `scripts/crabbox-untrusted-bootstrap.sh` を
`--fresh-pr` とともにアップロードします。これは固定バージョンの Node/pnpm をインストールし、SHA と
パッケージマネージャーの固定値を検証し、`HOME` を分離し、依存関係をインストールしてから、要求された
テストを実行します。
すべての `CRABBOX_TAILSCALE*` オーバーライドを未設定にし、`--network public
--tailscale=false` を強制し、exit-node/LAN フラグをクリアして、スクリプトをアップロードする前に `crabbox inspect` が
Tailscale 状態のないパブリックネットワークを報告することを必須とします。
所有する AWS/Hetzner キャパシティも、Blacksmith の障害、
クォータ問題、または所有キャパシティでの明示的なテストに対するフォールバックとして引き続き使用できます。

エージェントは、予定されている作業のために事前ウォームアップしません。最初の
高負荷コマンドの準備ができた時点で Testbox を遅延取得し、返された `tbx_...` id を後続の高負荷
コマンドで再利用し、実行ごとに現在の checkout を同期し、引き渡し前に停止します。

Crabbox を使用する Blacksmith 実行では、ワンショット Testbox のウォームアップ、claim、同期、実行、レポート、クリーンアップを行います。
組み込みの同期サニティチェックは、同期済みボックス上の
`git status --short` で追跡対象の削除が 200 件以上表示された場合、即座に失敗します。これにより、
`pnpm-lock.yaml` のようなルートファイルの消失を検出します。意図的に
大量削除を行う PR では、リモートコマンドに `CRABBOX_ALLOW_MASS_DELETIONS=1` を設定します。

Crabbox は、同期後の出力がないまま
同期フェーズに 5 分を超えて留まったローカル Blacksmith CLI 呼び出しも終了します。このガードを無効にするには
`CRABBOX_BLACKSMITH_SYNC_TIMEOUT_MS=0` を設定し、異常に大きなローカル差分の場合は、より大きい
ミリ秒値を使用します。

初回実行前に、リポジトリルートからラッパーを確認します。

```bash
pnpm crabbox:run -- --help | sed -n '1,120p'
```

リポジトリラッパーは、選択したプロバイダーを公開していない古い Crabbox バイナリを拒否します。また、Blacksmith を使用する実行では、現在の Testbox の同期、キュー、クリーンアップ動作をラッパーが利用できるよう、Crabbox 0.22.0 以降が必要です。Codex worktree または linked/sparse checkout では、Crabbox の起動前に pnpm が依存関係を調整する可能性があるため、ローカルの `pnpm crabbox:run` スクリプトを避け、代わりに node ラッパーを直接呼び出します。

```bash
node scripts/crabbox-wrapper.mjs run --provider blacksmith-testbox --timing-json --shell -- "pnpm test <path-or-filter>"
```

同階層の checkout を使用する場合は、計測または検証作業の前に、無視対象のローカルバイナリを再ビルドします。

```bash
version="$(git -C ../crabbox describe --tags --always --dirty | sed 's/^v//')" \
  && go build -C ../crabbox -trimpath -ldflags "-s -w -X github.com/openclaw/crabbox/internal/cli.version=${version}" -o bin/crabbox ./cmd/crabbox
```

`.crabbox.yaml` の `blacksmith:` ブロックでは、組織、ワークフロー、ジョブ、ref のデフォルトがすでに固定されているため、以下の明示的なフラグは省略できます。変更ゲート:

```bash
pnpm crabbox:run -- --provider blacksmith-testbox \
  --blacksmith-org openclaw \
  --blacksmith-workflow .github/workflows/ci-check-testbox.yml \
  --blacksmith-job check \
  --blacksmith-ref main \
  --idle-timeout 90m \
  --ttl 240m \
  --timing-json \
  --shell -- \
  "corepack pnpm check:changed"
```

ローカル依存関係が利用できない場合、または対象がファンアウトする場合に Testbox で対象テストを再実行します。

```bash
pnpm crabbox:run -- --provider blacksmith-testbox \
  --idle-timeout 90m \
  --ttl 240m \
  --timing-json \
  --shell -- \
  "corepack pnpm test <path-or-filter>"
```

フルスイート:

```bash
pnpm crabbox:run -- --provider blacksmith-testbox \
  --idle-timeout 90m \
  --ttl 240m \
  --timing-json \
  --shell -- \
  "corepack pnpm test"
```

最後の JSON サマリーを確認します。有用なフィールドは `provider`、`leaseId`、
`syncDelegated`、`exitCode`、`commandMs`、`totalMs` です。委任された
Blacksmith Testbox 実行では、Crabbox ラッパーの終了コードと JSON サマリーが
コマンド結果です。リンクされた GitHub Actions 実行が認証情報の注入と keepalive を所有します。SSH
コマンドがすでに返った後に Testbox が外部から停止された場合、実行は
`cancelled` として終了することがあります。ラッパーの `exitCode` が 0 以外である場合、またはコマンド出力に失敗したテストが示されている場合を除き、
これはクリーンアップ/ステータス上のアーティファクトとして扱います。
Blacksmith を使用するワンショット Crabbox 実行では、Testbox が自動的に停止される必要があります。
実行が中断された場合、またはクリーンアップが不明確な場合は、稼働中のボックスを確認し、自分が作成した
ボックスのみを停止します。

```bash
blacksmith testbox list --all
blacksmith testbox status --id <tbx_id>
blacksmith testbox stop --id <tbx_id>
```

同じ認証情報注入済みボックス上で複数のコマンドを意図的に実行する必要がある場合にのみ、再利用を使用します。

```bash
node scripts/crabbox-wrapper.mjs run --provider blacksmith-testbox --id <tbx_id> --timing-json --shell -- "corepack pnpm test <path-or-filter>"
pnpm crabbox:stop -- <tbx_id>
```

古いソースではなく、リースを再利用します。実行ごとに現在の checkout がアップロードされるよう、`--no-sync` は省略します。
変更されていない同期済みツリーを意図的に再実行する場合にのみ使用します。
信頼されていないコントリビューター/fork のコードでは、各コマンドで
`CRABBOX_ENV_ALLOW=CI`、`--provider aws --no-hydrate`、および新しい
一時的なリモート `HOME` を使用する必要があります。テスト前に、そのサニタイズ済みコマンド内で依存関係をインストールします。
再利用できるのは、同じ信頼されていないソース専用に新しくウォームアップしたリースのみです。
信頼済みまたは以前に認証情報が注入されたリースは決して使用しません。信頼されていない checkout のラッパーや config は、ローカルで
決して実行しないでください。クリーンで信頼済みの `main` からインストール済みの
信頼済み Crabbox バイナリを起動し、実行ごとに `--fresh-pr` を渡します。
`CRABBOX_AWS_INSTANCE_PROFILE` を未設定のままにし、解決済みの
インスタンスプロファイルが空でない場合は拒否し、信頼済みリモート IMDS のロール不在証明を必須とし、インストール/テスト前に
レビュー済み head SHA を検証します。リースをその SHA に紐付け、head が変更されたら
停止して再ウォームアップします。リモート PR が存在しない場合は、secretless fork CI を使用します。
信頼されていないソースでは、`hydrate-github` または認証情報が注入される Blacksmith ワークフローを
決して選択しないでください。

Crabbox レイヤーが壊れていて Blacksmith 自体は動作する場合、直接
Blacksmith を使用するのは、`list`、`status`、クリーンアップなどの診断のみに限定します。直接の Blacksmith 実行をメンテナー検証として扱う前に、
Crabbox の経路を修正してください。

`blacksmith testbox list --all` と `blacksmith testbox status` は動作するものの、新しい
ウォームアップが数分経っても IP や Actions 実行 URL のない `queued` のままになる場合は、
Blacksmith プロバイダー、キュー、請求、または組織の上限による負荷として扱います。作成した
キュー内の ID を停止し、それ以上 Testbox を起動せず、誰かが Blacksmith ダッシュボード、
請求、組織の上限を確認している間に、検証を以下の所有 Crabbox キャパシティパスへ移します。

Blacksmith が停止している、クォータで制限されている、必要な環境がない、または所有キャパシティ自体が明示的な目的である場合にのみ、所有 Crabbox キャパシティへエスカレーションします。

```bash
CRABBOX_CAPACITY_REGIONS=eu-west-1,eu-west-2,eu-central-1,us-east-1,us-west-2 \
  pnpm crabbox:warmup -- --provider aws --class standard --market on-demand --idle-timeout 90m
pnpm crabbox:hydrate -- --provider aws --id <cbx_id-or-slug>
pnpm crabbox:run -- --provider aws --id <cbx_id-or-slug> --timing-json --shell -- "pnpm check:changed"
pnpm crabbox:stop -- --provider aws <cbx_id-or-slug>
```

AWS に負荷がかかっている状況では、タスクが本当に 48xlarge クラスの CPU を必要とする場合を除き、`class=beast` を避けてください。`beast` リクエストは 192 vCPU から始まり、リージョンの EC2 Spot または On-Demand Standard クォータに抵触する最も簡単な方法です。リポジトリ所有の `.crabbox.yaml` は、デフォルトで `class: standard`、on-demand マーケット、`capacity.hints: true` を使用するため、仲介された AWS リースには、選択されたリージョン／マーケット、クォータの逼迫、Spot フォールバック、高負荷クラスの警告が表示されます。より負荷の高い広範なチェックには `fast` を使用し、standard/fast では不十分な場合にのみ `large` を使用してください。`beast` は、フルスイートや全 Plugin の Docker マトリクス、明示的なリリース／ブロッカー検証、高コア数でのパフォーマンスプロファイリングなど、例外的な CPU バウンドレーンにのみ使用してください。`pnpm check:changed`、対象を絞ったテスト、ドキュメントのみの作業、通常の lint／型チェック、小規模な E2E 再現、Blacksmith 障害のトリアージには `beast` を使用しないでください。Spot マーケットの変動がシグナルに混入しないよう、キャパシティ診断には `--market on-demand` を使用してください。

`.crabbox.yaml` は、プロバイダー、同期、GitHub Actions のハイドレーションのデフォルトを管理します。Crabbox の同期では `.git` が転送されないため、ハイドレーションされた Actions チェックアウトは、メンテナーのローカルリモートやオブジェクトストアを同期せず、独自のリモート Git メタデータを保持します。また、リポジトリ設定では、転送すべきでないローカルのランタイム／ビルド成果物（`.artifacts` やテストレポートなど）も除外されます。`.github/workflows/crabbox-hydrate.yml` は、チェックアウト、Node/pnpm のセットアップ、`origin/main` の取得、および所有クラウドの `crabbox run --id <cbx_id>` コマンドへの非シークレット環境の引き渡しを管理します。

## 関連項目

- [インストールの概要](/ja-JP/install)
- [開発チャンネル](/ja-JP/install/development-channels)
