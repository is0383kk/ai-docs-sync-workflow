---
read_when:
    - テストの実行または修正
summary: ローカルでテスト（vitest）を実行する方法と、force/coverage モードを使用するタイミング
title: テスト
x-i18n:
    generated_at: "2026-07-26T10:30:35Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 391185703e853bb523e1396eb22da4693d10d47b1644d3b2a51707d329f67dae
    source_path: reference/test.md
    workflow: 16
---

- 完全なテストキット（スイート、ライブ、Docker）：[テスト](/ja-JP/help/testing)
- アップデートと Plugin パッケージの検証：[アップデートと Plugin のテスト](/ja-JP/help/testing-updates-plugins)

## エージェントのデフォルト

エージェントセッションでは、信頼できるソースであり、既存の依存関係がインストール済みの場合に限り、1つまたは少数の対象を絞ったテストと低コストの静的チェックをローカルで実行します。信頼できないリポジトリのツールは、ローカルでは決して実行しません。大規模なスイート、型チェックや lint のファンアウトを伴う変更ゲート、ビルド、Docker、パッケージレーン、E2E、ライブ証明、クロスプラットフォーム検証は、Crabbox を介してリモートで実行します。信頼できるメンテナーによる負荷の高い証明では、デフォルトで Blacksmith Testbox を使用します。構成された Testbox ワークフローは認証情報を投入するため、信頼できないコントリビューターまたはフォークのコードでは、代わりにシークレットなしのフォーク CI またはサニタイズ済みの直接 AWS Crabbox を使用する必要があります。

予想される作業のために事前ウォームアップしないでください。最初の負荷の高いコマンドを実行する準備ができた時点でバックエンドを遅延取得し、返された `tbx_...` ID を以降の負荷の高いコマンドで再利用し、実行のたびに現在のチェックアウトを同期して、引き渡し前に停止します。

最初に正常に再利用された後、ラッパーはリースのベース、依存関係、および Testbox ワークフローのフィンガープリントを `.crabbox/testbox-leases/` に記録します。ソースのみの編集では、ウォームアップ済みボックスを引き続き再利用します。マージベース、ロックファイル、パッケージマネージャー入力、ラッパー、または Testbox ワークフローが変更されるとフェイルクローズし、新しいリースが必要になります。各実行では引き続き現在のチェックアウトを同期します。
`OPENCLAW_TESTBOX_ALLOW_STALE=1` は意図的な診断専用であり、リリース証明には使用しません。

以下のローカルテストコマンドは、人間のワークフローと範囲を限定したエージェント証明用です。リモートプロバイダーが利用できない場合は報告する必要があります。これは、大規模なローカルゲートを暗黙に実行してよいという許可ではありません。

信頼できないソースの負荷の高い証明では、`--provider aws` を使用して遅延ウォームアップします。各実行では `CRABBOX_ENV_ALLOW=CI` を設定し、`--provider aws --no-hydrate` を渡し、依存関係のインストールまたはテストの実行前に新しい一時リモート `HOME` を使用する必要があります。その信頼できないソース専用に新しくウォームアップしたリースを使用し、信頼できるリースや以前に認証情報を投入したリースは決して再利用しないでください。クリーンで信頼できる `main` チェックアウトから、インストール済みの信頼できる Crabbox バイナリを起動し、`--fresh-pr` を使用してリモート PR のみを取得します。信頼できないチェックアウトのラッパーや構成をローカルで実行してはなりません。
`CRABBOX_AWS_INSTANCE_PROFILE` を設定解除し、解決された `aws.instanceProfile` が空でない限りフェイルクローズします。インストールやテストの前に、信頼できる絶対パスのツールを使用して IMDSv2 トークンを必須とし、IAM 認証情報エンドポイントが 404 を返すことを証明し、リモートの `git rev-parse HEAD` がレビュー済み PR ヘッドの完全な SHA と等しいことを検証します。リースをその SHA に関連付け、ヘッドが変更された場合は停止して再ウォームアップします。クリーンな `main` から信頼できる `scripts/crabbox-untrusted-bootstrap.sh` を `--fresh-pr` とともにアップロードします。これは固定バージョンの Node/pnpm をインストールし、SHA とパッケージマネージャーの固定値を検証し、`HOME` を分離し、依存関係をインストールしてから、要求されたテストを実行します。ブローカーがロールなしを証明できない場合、またはリモート PR が存在しない場合は、シークレットなしのフォーク CI を使用します。`hydrate-github`、`--no-sync`、または認証情報を投入する Testbox ワークフローは使用しないでください。
すべての `CRABBOX_TAILSCALE*` オーバーライドを設定解除し、`--network public
--tailscale=false` を強制し、出口ノード/LAN フラグをクリアして、スクリプトをアップロードする前に `crabbox inspect` が Tailscale 状態のないパブリックネットワークを報告することを必須とします。

## 通常のローカル実行順序

1. 変更範囲の Vitest 証明には `pnpm test:changed`。
2. 1つのファイル、ディレクトリ、または明示的なターゲットには `pnpm test <path-or-filter>`。
3. 完全なローカル Vitest スイートが意図的に必要な場合に限り `pnpm test`。

Codex ワークツリーまたはリンク済み/スパースチェックアウトでは、エージェントはローカルでの `pnpm test*` / `pnpm check*` / `pnpm crabbox:run` の直接実行を避けます。

- 依存関係の準備ができている場合の範囲を限定した対象テスト：
  `node scripts/run-vitest.mjs <path-or-filter>`。
- 分類優先の変更チェック：`node scripts/check-changed.mjs`。ドキュメントのみ、変更なし、および小規模なメタデータのプランは、依存関係の準備ができている場合はローカルに留め、負荷の高いプランまたは依存関係が不足しているプランは Testbox に委任します。
- 保持したリースによる明示的な大規模証明：`node scripts/crabbox-wrapper.mjs run --provider blacksmith-testbox ... -- env OPENCLAW_CHECK_CHANGED_REMOTE_CHILD=1 OPENCLAW_CHANGED_LANES_RAW_SYNC=1 corepack pnpm check:changed`。これにより pnpm は Testbox 内で実行されます。
- ラッパーの最後の `exitCode` とタイミング JSON がコマンド結果です。委任された Blacksmith GitHub Actions 実行では、SSH コマンドが成功した後でも、キープアライブアクションの外部から Testbox が停止されるため `cancelled` が表示される場合があります。失敗と判断する前に、ラッパーの概要とコマンド出力を確認してください。
- `OPENCLAW_HEAVY_CHECK_LOCK_SCOPE=worktree <local-heavy-check command>`：`pnpm check:changed` や対象を絞った `pnpm test ...` などのコマンドで、負荷の高いチェックの直列化を Git 共通ディレクトリではなく現在のワークツリー内に保持します。リンクされた複数のワークツリーで独立したチェックを意図的に実行する場合に限り、高性能なローカルホストで使用してください。

## コアコマンド

テストラッパーの実行は、短い `[test] passed|failed|skipped ... in ...` の概要で終了します。Vitest 自体の所要時間行は、シャードごとの詳細として残ります。

| コマンド                                          | 実行内容                                                                                                                                                                                                                                                                                                                                                    |
| ------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `pnpm test`                                       | 明示的なファイル/ディレクトリターゲットは、範囲指定された Vitest レーンを介して処理されます。ターゲットなしの実行は完全なスイート証明です。固定シャードグループはローカル並列実行用の末端構成に展開され、開始前に想定されるシャードのファンアウトが出力されます。拡張機能グループは、1つの巨大なルートプロジェクトプロセスではなく、常に拡張機能ごとのシャード構成に展開されます。           |
| `pnpm test:changed`                               | 低コストでスマートな変更テスト実行：テストへの直接編集、兄弟の `*.test.ts` ファイル、明示的なソースマッピング、ローカルインポートグラフから正確なターゲットを特定します。大規模な変更、構成変更、パッケージ変更は、正確なテストにマッピングされない限りスキップされます。                                                                                                                               |
| `OPENCLAW_TEST_CHANGED_BROAD=1 pnpm test:changed` | 明示的な大規模変更テスト実行。テストハーネス、構成、またはパッケージの編集で、Vitest のより広範な変更テスト動作にフォールバックする必要がある場合に使用します。                                                                                                                                                                                                                        |
| `pnpm test:force`                                 | 構成された OpenClaw Gateway ポート（デフォルトは `18789`）を解放してから、分離された Gateway ポートで完全なスイートを実行し、サーバーテストが実行中のインスタンスと競合しないようにします。                                                                                                                                                                                    |
| `pnpm test:coverage`                              | デフォルトのユニットレーン（`vitest.unit.config.ts`）について、情報提供用の V8 カバレッジレポートを出力します。カバレッジしきい値は適用されません。                                                                                                                                                                                                                             |
| `pnpm test:coverage:changed`                      | `origin/main` 以降に変更されたファイルのみを対象とするユニットカバレッジ。                                                                                                                                                                                                                                                                                                       |
| `pnpm changed:lanes`                              | `origin/main` との差分によってトリガーされるアーキテクチャレーンを表示します。                                                                                                                                                                                                                                                                                      |
| `pnpm check:changed`                              | 実行方法を選択する前に、変更されたレーンを分類します。ドキュメントのみ、変更なし、および小規模なメタデータのプランは、依存関係の準備ができている場合はローカルに留めます。型チェック/lint のファンアウト、その他の負荷の高いレーン、またはローカル依存関係の不足を含むプランは、CI 外では Crabbox/Testbox に委任します。Vitest は実行しません。テスト証明には `pnpm test:changed` または `pnpm test <target>` を使用してください。 |

## 共有テスト状態とプロセスヘルパー

- `src/test-utils/openclaw-test-state.ts`：テストで分離された `HOME`、`OPENCLAW_STATE_DIR`、`OPENCLAW_CONFIG_PATH`、構成フィクスチャ、ワークスペース、エージェントディレクトリ、または認証プロファイルストアが必要な場合に、Vitest から使用します。
- `pnpm test:env-mutations:report`：`HOME`、`OPENCLAW_STATE_DIR`、`OPENCLAW_CONFIG_PATH`、`OPENCLAW_WORKSPACE_DIR`、または関連する環境キーを直接変更するテスト/ハーネスについてのノンブロッキングレポートです。共有テスト状態ヘルパーへの移行候補を見つけるために使用します。
- `test/helpers/openclaw-test-instance.ts`：実行中の Gateway、CLI 環境、ログ取得、クリーンアップを1か所で必要とするプロセスレベルの E2E テスト。
- `scripts/lib/docker-e2e-image.sh` を読み込む Docker/Bash E2E レーンは、`docker_e2e_test_state_shell_b64 <label> <scenario>` をコンテナに渡し、`scripts/lib/openclaw-e2e-instance.sh` でデコードできます。複数ホームのスクリプトは `docker_e2e_test_state_function_b64` を渡し、各フローで `openclaw_test_state_create <label> <scenario>` を呼び出せます。`node scripts/lib/openclaw-test-state.mjs -- create --label <name> --scenario <name> --env-file <path> --json` は読み込み可能なホスト環境ファイルを書き込みます（`create` の前の `--` により、新しい Node ランタイムが `--env-file` を Node フラグとして扱うことを防ぎます）。Gateway を起動するレーンは、エントリポイントの解決、モック OpenAI の起動、フォアグラウンド/バックグラウンド起動、準備完了プローブ、状態環境のエクスポート、ログダンプ、プロセスのクリーンアップのために `scripts/lib/openclaw-e2e-instance.sh` を読み込めます。

## Control UI、TUI、拡張機能レーン

- **モック化された Control UI E2E:** `pnpm test:ui:e2e` は、Vite Control UI を起動し、モック化された Gateway WebSocket に対して実際の Chromium ページを操作する Vitest + Playwright レーンを実行します。テストは `ui/src/**/*.e2e.test.ts` にあり、共有モックと制御機能は `ui/src/test-helpers/control-ui-e2e.ts` にあります。`pnpm test:e2e` にはこのレーンが含まれます。エージェント実行では、対象を絞った検証も含めて、デフォルトで Testbox/Crabbox を使用します。明示的にローカルへフォールバックする場合にのみ `node scripts/run-vitest.mjs run --config test/vitest/vitest.ui-e2e.config.ts --configLoader runner ui/src/ui/e2e/chat-flow.e2e.test.ts` を使用してください。
- **TUI PTY テスト:** `node scripts/run-vitest.mjs run --config test/vitest/vitest.tui-pty.config.ts` は、高速な偽バックエンド PTY レーンを実行します。`OPENCLAW_TUI_PTY_INCLUDE_LOCAL=1` または `pnpm tui:pty:test:watch --mode local` は、外部モデルエンドポイントのみをモック化する、より低速な `tui --local` スモークテストを実行します。生の ANSI スナップショットではなく、安定した表示テキストまたはフィクスチャ呼び出しをアサートしてください。
- `pnpm test:extensions` と `pnpm test extensions` は、すべての拡張機能/Plugin シャードを実行します。負荷の高いチャンネル Plugin、ブラウザー Plugin、OpenAI は専用シャードとして実行され、その他の Plugin グループはバッチ処理のままです。`pnpm test extensions/<id>` は、バンドルされた Plugin のレーンを 1 つ実行します。
- 同階層にテストがあるソースファイルは、より広いディレクトリ glob にフォールバックする前に、その同階層のテストへマッピングされます。`src/channels/plugins/contracts/test-helpers`、`src/plugin-sdk/test-helpers`、`src/plugins/contracts` 配下のヘルパー編集では、依存関係パスを正確に特定できる場合、すべてのシャードを広範に実行する代わりに、ローカルインポートグラフを使用してインポート元のテストを実行します。
- コントラクトディレクトリのターゲットは、それぞれのコントラクトレーンへ分岐します。`pnpm test src/channels/plugins/contracts` は 4 つのチャンネルコントラクト構成を実行し、`pnpm test src/plugins/contracts` は Plugin コントラクト構成を実行します。これは、汎用の `channels`/`plugins` プロジェクトが `contracts/**` を除外するためです。
- `auto-reply` は 3 つの専用構成（`core`、`top-level`、`reply`）に分割されているため、返信ハーネスが、より軽量なトップレベルのステータス/トークン/ヘルパーテストを圧迫しません。
- 選択された `plugin-sdk` および `commands` のテストファイルは、`test/setup.ts` のみを保持する専用の軽量レーンを経由し、ランタイム負荷の高いケースは既存のレーンに残します。
- 基本 Vitest 構成のデフォルトは `pool: "threads"` と `isolate: false` で、共有の非分離ランナーがリポジトリ全体の構成で有効になっています。
- `pnpm test:channels` は `vitest.channels.config.ts` を実行します。

## Gateway と E2E

- Gateway 統合はオプトインです: `OPENCLAW_TEST_INCLUDE_GATEWAY=1 pnpm test` または `pnpm test:gateway`。
- `pnpm test:e2e`: リポジトリ E2E 集約 = `pnpm test:e2e:gateway && pnpm test:ui:e2e`。
- `pnpm test:e2e:gateway`: Gateway のエンドツーエンドスモークテスト（複数インスタンスの WS/HTTP/Node ペアリング）。デフォルトは `threads` + `isolate: false` で、`vitest.e2e.config.ts` では適応型ワーカーを使用します。`OPENCLAW_E2E_WORKERS=<n>` で調整し、`OPENCLAW_E2E_VERBOSE=1` で詳細ログを有効にできます。
- `pnpm test:live`: プロバイダーのライブテスト（Claude/Minimax/DeepSeek/z.ai など、`*.live.test.ts` でゲート）。スキップを解除するには API キーと `LIVE=1`（または `OPENCLAW_LIVE_TEST=1`）が必要です。`OPENCLAW_LIVE_TEST_QUIET=0` で詳細出力を有効にできます。

## 完全な Docker スイート（`pnpm test:docker:all`）

共有ライブテストイメージをビルドし、OpenClaw を npm tarball として一度パックし、ベア Node/Git ランナーイメージと、その tarball を `/app` にインストールする機能イメージをビルドまたは再利用した後、重み付きスケジューラーを介して Docker スモークレーンを実行します。`scripts/package-openclaw-for-docker.mjs` は、ローカル/CI で単一のパッケージパッカーとして機能し、Docker が tarball を使用する前に、その tarball と `dist/postinstall-inventory.json` を検証します。

- ベアイメージ（`OPENCLAW_DOCKER_E2E_BARE_IMAGE`）: インストーラー/更新/Plugin 依存関係レーン。コピーされたリポジトリソースではなく、事前ビルド済みの tarball をマウントします。
- 機能イメージ（`OPENCLAW_DOCKER_E2E_FUNCTIONAL_IMAGE`）: 通常のビルド済みアプリ機能レーン。
- レーン定義: `scripts/lib/docker-e2e-scenarios.mjs`。プランナー: `scripts/lib/docker-e2e-plan.mjs`。エグゼキューター: `scripts/test-docker-all.mjs`。
- `node scripts/test-docker-all.mjs --plan-json` は、Docker をビルドまたは実行せずに、スケジューラーが所有する CI プラン（レーン、イメージ種別、パッケージ/ライブイメージの要否、状態シナリオ、認証情報チェック）を出力します。

スケジューリング調整項目（環境変数、括弧内はデフォルト）:

| 環境変数                                                                                                        | デフォルト          | 用途                                                                                                                                                                                                                                                                                       |
| --------------------------------------------------------------------------------------------------------------- | ------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `OPENCLAW_DOCKER_ALL_PARALLELISM`                                                                               | 10                  | プロセススロット。                                                                                                                                                                                                                                                                         |
| `OPENCLAW_DOCKER_ALL_TAIL_PARALLELISM`                                                                          | 10                  | プロバイダー依存のテールプール。                                                                                                                                                                                                                                                           |
| `OPENCLAW_DOCKER_ALL_LIVE_LIMIT`                                                                                | 9                   | 負荷の高いライブプロバイダーレーンの上限。                                                                                                                                                                                                                                                 |
| `OPENCLAW_DOCKER_ALL_NPM_LIMIT`                                                                                 | 5                   | npm リソースレーンの上限。                                                                                                                                                                                                                                                                 |
| `OPENCLAW_DOCKER_ALL_SERVICE_LIMIT`                                                                             | 7                   | サービスリソースレーンの上限。                                                                                                                                                                                                                                                             |
| `OPENCLAW_DOCKER_ALL_LIVE_CLAUDE_LIMIT` / `_CODEX_LIMIT` / `_GEMINI_LIMIT` / `_DROID_LIMIT` / `_OPENCODE_LIMIT` | 4                   | プロバイダーごとの高負荷レーンの上限。                                                                                                                                                                                                                                                     |
| `OPENCLAW_DOCKER_ALL_LIVE_OPENAI_LIMIT` / `_TELEGRAM_LIMIT`                                                     | 1                   | プロバイダーごとの、より狭い上限。                                                                                                                                                                                                                                                         |
| `OPENCLAW_DOCKER_ALL_WEIGHT_LIMIT` / `OPENCLAW_DOCKER_ALL_DOCKER_LIMIT`                                         | -                   | より大規模なホスト向けのオーバーライド。                                                                                                                                                                                                                                                   |
| `OPENCLAW_DOCKER_ALL_START_STAGGER_MS`                                                                          | 2000                | レーン開始間の遅延。ローカル Docker デーモンで作成処理が集中することを防ぎます。                                                                                                                                                                                                            |
| `OPENCLAW_DOCKER_ALL_LANE_TIMEOUT_MS`                                                                           | 7,200,000 (120 min) | レーンごとのフォールバックタイムアウト。選択されたライブ/テールレーンでは、より厳しい上限を使用します。                                                                                                                                                                                     |
| `OPENCLAW_DOCKER_ALL_LIVE_RETRIES`                                                                              | 1                   | 一時的なライブプロバイダー障害に対する再試行回数。                                                                                                                                                                                                                                         |
| `OPENCLAW_DOCKER_ALL_DRY_RUN`                                                                                   | off                 | Docker を実行せずにレーンマニフェストを出力します。                                                                                                                                                                                                                                        |
| `OPENCLAW_DOCKER_ALL_STATUS_INTERVAL_MS`                                                                        | 30000               | アクティブレーンのステータス出力間隔。                                                                                                                                                                                                                                                     |
| `OPENCLAW_DOCKER_ALL_TIMINGS`                                                                                   | on                  | 最長優先の順序付けに `.artifacts/docker-tests/lane-timings.json` を再利用します。無効にするには `0` に設定します。                                                                                                                                                                        |
| `OPENCLAW_DOCKER_ALL_LIVE_MODE`                                                                                 | -                   | 決定的/ローカルレーンのみの場合は `skip`、ライブプロバイダーレーンのみの場合は `only`。エイリアス: `pnpm test:docker:local:all`、`pnpm test:docker:live:all`。ライブのみのモードでは、メインとテールのライブレーンを 1 つの最長優先プールに統合し、プロバイダーバケットが Claude/Codex/Gemini の処理をまとめて配置できるようにします。 |
| `OPENCLAW_LIVE_CLI_BACKEND_SETUP_TIMEOUT_SECONDS`                                                               | 180                 | CLI バックエンドの Docker セットアップタイムアウト。                                                                                                                                                                                                                                       |

リソース上限の環境変数パターンは `OPENCLAW_DOCKER_ALL_<RESOURCE>_LIMIT` です（リソース名を大文字にし、英数字以外を `_` にまとめます）。

その他の動作: ランナーはデフォルトで Docker の事前チェックを行い、古い OpenClaw E2E コンテナをクリーンアップし、互換性のあるレーン間でプロバイダー CLI ツールのキャッシュを共有します。また、`OPENCLAW_DOCKER_ALL_FAIL_FAST=0` が設定されていない限り、最初の失敗後は新しいプールレーンのスケジュールを停止します。並列度の低いホストで、1 つのレーンが有効な重み/リソース上限を超える場合でも、空のプールから開始し、容量を解放するまで単独で実行できます。レーンごとのログ、`summary.json`、`failures.json`、およびフェーズのタイミングは `.artifacts/docker-tests/<run-id>/` 配下に書き込まれます。遅いレーンの調査には `pnpm test:docker:timings <summary.json>` を使用し、低コストの対象限定再実行コマンドを出力するには `pnpm test:docker:rerun <run-id|summary.json|failures.json>` を使用します。

### 主な Docker レーン

| コマンド                                                                    | 検証内容                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                              |
| --------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `pnpm test:docker:browser-cdp-snapshot`                                     | 生の CDP と分離された Gateway を使用する Chromium ベースのソース E2E コンテナ。`browser doctor --deep` の CDP ロールスナップショットには、リンク URL、カーソルによってクリック可能に昇格された要素、iframe 参照、フレームメタデータが含まれます。                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                      |
| `pnpm test:docker:skill-install`                                            | `skills.install.allowUploadedArchives: false` を使用してパック済み tarball を最小構成の Docker ランナーにインストールし、稼働中の ClawHub 検索から現在のスキルスラッグを解決し、`openclaw skills install` 経由でインストールして、`SKILL.md`、`.clawhub/origin.json`、`.clawhub/lock.json`、および `skills info --json` を検証します。                                                                                                                                                                                                                                                                                                                                                                                                                                              |
| `pnpm test:docker:live-cli-backend:claude`, `:claude:resume`, `:claude:mcp` | CLI バックエンドに対象を絞ったライブプローブ。Gemini には対応する `:resume` および `:mcp` エイリアスがあります。                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                    |
| `pnpm test:docker:openwebui`                                                | Docker 化された OpenClaw + Open WebUI: サインインし、`/api/models` を確認して、`/api/chat/completions` 経由で実際のプロキシチャットを実行します。使用可能なライブモデルキーが必要で、外部イメージをプルします。単体/E2E スイートのような CI 安定性は想定されていません。                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                         |
| `pnpm test:docker:mcp-channels`                                             | シード済みの Gateway コンテナと、`openclaw mcp serve` を起動するクライアントコンテナ: ルーティングされた会話の検出、トランスクリプトの読み取り、添付ファイルのメタデータ、ライブイベントキューの動作、送信ルーティング、および実際の stdio ブリッジを介した Claude 形式のチャンネル通知と権限通知を検証します（アサーションは生の stdio MCP フレームを直接読み取ります）。                                                                                                                                                                                                                                                                                                                                                                                                               |
| `pnpm test:docker:upgrade-survivor`                                         | 既存ユーザーの古く変更されたフィクスチャ上にパック済み tarball をインストールし、ライブのプロバイダー/チャンネルキーを使用せずにパッケージ更新と非対話型 doctor を実行して、loopback Gateway を起動します。エージェント/チャンネル設定、Plugin 許可リスト、ワークスペース/セッションファイル、古いレガシー Plugin の依存関係状態、起動、および RPC ステータスが維持されることを確認します。                                                                                                                                                                                                                                                                                                                                                                                                                                     |
| `pnpm test:docker:published-upgrade-survivor`                               | デフォルトで `openclaw@latest` をインストールし、現実的な既存ユーザーファイルをシードし、組み込みの `openclaw config set` レシピを使用して設定し、パック済み tarball に更新して、非対話型 doctor を実行し、`.artifacts/upgrade-survivor/summary.json` を書き込み、`/healthz`、`/readyz`、および RPC ステータスを確認します。`OPENCLAW_UPGRADE_SURVIVOR_BASELINE_SPEC` で上書きし、`OPENCLAW_UPGRADE_SURVIVOR_BASELINE_SPECS` でマトリックスを拡張するか、`OPENCLAW_UPGRADE_SURVIVOR_SCENARIOS=reported-issues` でシナリオフィクスチャを追加できます（`configured-plugin-installs` と `stale-source-plugin-shadow` を含む）。Package Acceptance では、これらを `published_upgrade_survivor_baseline(s)` / `_scenarios` として公開し、`last-stable-4` や `all-since-2026.4.23` などのメタトークンを解決します。 |
| `pnpm test:docker:update-migration`                                         | `plugin-deps-cleanup` シナリオの公開済みアップグレード存続ハーネス。デフォルトでは `openclaw@2026.4.23` から開始します。`Update Migration` ワークフローは、`baselines=all-since-2026.4.23` を使用してこれを拡張し、Full Release CI の外部で設定済み Plugin の依存関係クリーンアップを検証します。                                                                                                                                                                                                                                                                                                                                                                                                                                                                       |
| `pnpm test:docker:plugins`                                                  | ローカルパス、`file:`、依存関係がホイストされた npm レジストリパッケージ、移動する git 参照、ClawHub フィクスチャ、マーケットプレイス更新、および Claude バンドルの有効化/検査に対するインストール/更新スモークテスト。                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                              |

## ローカル PR ゲート

ローカルで PR のランディング/ゲートチェックを行うには、次を実行します:

- `pnpm check:changed`
- `pnpm check`
- `pnpm check:test-types`
- `pnpm build`
- `pnpm test`
- `pnpm check:docs`

負荷の高いホストで `pnpm test` が不安定になった場合は、リグレッションとして扱う前に一度再実行し、その後 `pnpm test <path/to/test>` で切り分けます。メモリに制約のあるホストでは:

- `OPENCLAW_VITEST_MAX_WORKERS=1 pnpm test`
- `OPENCLAW_VITEST_FS_MODULE_CACHE_PATH=/tmp/openclaw-vitest-cache pnpm test:changed`

## テストパフォーマンスツール

- `pnpm test:perf:imports`: Vitest のインポート所要時間とインポート内訳のレポートを有効にします。明示的なファイル/ディレクトリ対象には、引き続きスコープ付きレーンルーティングを使用します。`pnpm test:perf:imports:changed` は、同じプロファイリングの対象を `origin/main` 以降に変更されたファイルに限定します。
- `pnpm test:perf:changed:bench -- --ref <git-ref>` は、同じコミット済み git 差分について、ルーティングされた変更モードのパスをネイティブのルートプロジェクト実行と比較してベンチマークします。`pnpm test:perf:changed:bench -- --worktree` は、先にコミットせずに現在のワークツリーの変更セットをベンチマークします。
- `pnpm test:perf:profile:main` は、Vitest のメインスレッド用 CPU プロファイル（`.artifacts/vitest-main-profile`）を書き込みます。`pnpm test:perf:profile:runner` は、単体テストランナー用の CPU + ヒーププロファイル（`.artifacts/vitest-runner-profile`）を書き込みます。
- `pnpm test:perf:groups --full-suite --allow-failures --output .artifacts/test-perf/baseline-before.json`: フルスイートの各 Vitest リーフ設定を直列に実行し、グループ化された所要時間データと設定ごとの JSON/ログアーティファクトを書き込みます。フルスイートレポートはデフォルトでファイルを分離するため、以前のファイルで保持されたモジュールグラフや GC の一時停止が、後続のアサーションに計上されません。共有ワーカーでの蓄積を意図的にプロファイリングする場合にのみ、`-- --no-isolate` を渡してください。Test Performance Agent は、低速テストの修正を試みる前に、これをベースラインとして使用します。`pnpm test:perf:groups:compare .artifacts/test-perf/baseline-before.json .artifacts/test-perf/after-agent.json` は、パフォーマンス重視の変更後にグループ化されたレポートを比較します。
- フル、拡張機能、および include パターンのシャード実行は、`.artifacts/vitest-shard-timings.json` のローカルタイミングデータを更新します。後続の設定全体の実行では、それらのタイミングを使用して低速シャードと高速シャードのバランスを取ります。include パターンの CI シャードはタイミングキーにシャード名を追加するため、設定全体のタイミングデータを置き換えることなく、フィルタリングされたシャードのタイミングを確認できます。ローカルタイミングアーティファクトを無視するには、`OPENCLAW_TEST_PROJECTS_TIMINGS=0` を設定します。

## ベンチマーク

<Accordion title="モデルのレイテンシ（scripts/bench-model.ts）">

```bash
pnpm tsx scripts/bench-model.ts --runs 10
```

任意の環境変数: `MINIMAX_API_KEY`、`MINIMAX_BASE_URL`、`MINIMAX_MODEL`、`ANTHROPIC_API_KEY`。デフォルトのプロンプト: 「1 語だけで返信してください: ok。句読点や余分なテキストは不要です。」

</Accordion>

<Accordion title="CLI の起動（scripts/bench-cli-startup.ts）">

```bash
pnpm test:startup:bench
pnpm test:startup:bench:smoke
pnpm test:startup:bench:save
pnpm test:startup:bench:update
pnpm test:startup:bench:check
pnpm tsx scripts/bench-cli-startup.ts --runs 12
pnpm tsx scripts/bench-cli-startup.ts --preset real --case status --case gatewayStatus --runs 3
pnpm tsx scripts/bench-cli-startup.ts --entry openclaw.mjs --entry-secondary dist/entry.js --preset all
```

プリセット:

- `startup`: `--version`, `--help`, `health`, `health --json`, `status --json`, `status`
- `real`: `health`, `status`, `status --json`, `sessions`, `sessions --json`, `tasks --json`, `tasks list --json`, `tasks audit --json`, `agents list --json`, `gateway status`, `gateway status --json`, `gateway health --json`, `config get gateway.port`
- `all`: 両方のプリセットを組み合わせたもの

出力には、コマンドごとの `sampleCount`、平均、p50、p95、最小値/最大値、終了コード/シグナルの分布、最大 RSS が含まれます。`--cpu-prof-dir` / `--heap-prof-dir` は実行ごとに V8 プロファイルを書き込みます。

保存される出力: `pnpm test:startup:bench:smoke` は `.artifacts/cli-startup-bench-smoke.json` を書き込み、`pnpm test:startup:bench:save` は `.artifacts/cli-startup-bench-all.json`（`runs=5 warmup=1`）を書き込みます。チェックイン済みフィクスチャ: `test/fixtures/cli-startup-bench.json`。`pnpm test:startup:bench:update` で更新され、`pnpm test:startup:bench:check` で比較されます。

</Accordion>

<Accordion title="Gateway の起動（scripts/bench-gateway-startup.ts）">

デフォルトでは `dist/entry.js` にあるビルド済み CLI エントリを使用します。最初に `pnpm build` を実行してください。代わりにソースランナーを測定するには `--entry scripts/run-node.mjs` を渡し、その結果はビルド済みエントリのベースラインと分けて管理してください。

```bash
pnpm test:startup:gateway -- --runs 5 --warmup 1
pnpm test:startup:gateway -- --case skipChannels --case fiftyPlugins --runs 5
node --import tsx scripts/bench-gateway-startup.ts --case default --runs 5 --output .artifacts/gateway-startup.json
```

ケース ID: `default`、`skipChannels`（チャンネルの起動をスキップ）、`oneInternalHook`、`allInternalHooks`、`fiftyPlugins`（50 個のマニフェスト Plugin）、`fiftyStartupLazyPlugins`（50 個の起動時遅延読み込みマニフェスト Plugin）。

出力には、最初のプロセス出力、`/healthz`、`/readyz`、HTTP リッスンログ時刻、Gateway 準備完了ログ時刻、CPU 時間、CPU コア比率、最大 RSS、ヒープ、起動トレースメトリクス、イベントループ遅延、Plugin ルックアップテーブルの詳細メトリクスが含まれます。スクリプトは子 Gateway 環境に `OPENCLAW_GATEWAY_STARTUP_TRACE=1` を設定します。

`/healthz` はライブネス（HTTP サーバーが応答可能であること）です。`/readyz` は利用可能な準備完了状態（起動時の Plugin サイドカー、チャンネル、および準備完了に不可欠なアタッチ後の処理が完了していること）です。起動フックは非同期にディスパッチされるため、準備完了の保証には含まれません。準備完了ログ時刻は Gateway の内部タイムスタンプであり、プロセス側の要因分析には役立ちますが、外部の `/readyz` プローブの代わりにはなりません。

変更を比較する場合は、JSON 出力または `--output` を使用してください。フェーズのタイミングだけでは説明できないインポート、コンパイル、または CPU バウンドの処理がトレース出力で示された場合に限り、`--cpu-prof-dir` を使用してください。

</Accordion>

<Accordion title="Gateway の再起動（scripts/bench-gateway-restart.ts）">

macOS と Linux のみ（プロセス内再起動に SIGUSR1 を使用し、Windows では即座に失敗します）。上記の Gateway 起動と同じく、デフォルトではビルド済みエントリを使用し、`--entry scripts/run-node.mjs` でオーバーライドできます。

```bash
pnpm test:restart:gateway -- --case skipChannels --runs 1 --restarts 5
pnpm test:restart:gateway -- --case default --runs 3 --restarts 3 --warmup 1
```

ケース ID: `skipChannels`、`skipChannelsAcpxProbe`（ACPX 起動プローブ有効）、`skipChannelsNoAcpxProbe`（プローブ無効）、`default`、`fiftyPlugins`。

出力には、次の `/healthz`、次の `/readyz`、ダウンタイム、再起動の準備完了タイミング、CPU、RSS、置換プロセスの起動トレースメトリクス、およびシグナル処理、アクティブな処理のドレイン、終了フェーズ、次回起動、準備完了タイミング、メモリスナップショットに関する再起動トレースメトリクスが含まれます。スクリプトは `OPENCLAW_GATEWAY_STARTUP_TRACE=1` と `OPENCLAW_GATEWAY_RESTART_TRACE=1` を設定します。

変更が再起動シグナリング、終了ハンドラー、再起動後の起動、サイドカーのシャットダウン、サービスの引き継ぎ、または再起動後の準備完了状態に影響する場合は、このベンチマークを使用してください。チャンネル起動から Gateway の仕組みを切り分けるには `skipChannels` から始め、狭いケースで再起動経路を説明できた後にのみ、`default` または Plugin を多用するケースを使用してください。トレースメトリクスは要因分析の手掛かりであり、判定そのものではありません。再起動の変更は、複数のサンプル、対応する所有者スパン、`/healthz`/`/readyz` の動作、およびユーザーから見える再起動の契約に基づいて評価してください。

</Accordion>

## オンボーディング E2E（Docker）

任意です。コンテナ化されたオンボーディングのスモークテストにのみ必要です。クリーンな Linux コンテナで完全なコールドスタートフローを実行します。

```bash
scripts/e2e/onboard-docker.sh
```

疑似 tty を介して対話型ウィザードを操作し、設定/ワークスペース/セッションファイルを検証した後、Gateway を起動して `openclaw health` を実行します。

## QR インポートのスモークテスト（Docker）

メンテナンス対象の QR ランタイムヘルパーが、サポート対象の Docker Node ランタイム（デフォルトは Node 24、Node 22 と互換）で読み込まれることを確認します。

```bash
pnpm test:docker:qr
```

## 関連項目

- [テスト](/ja-JP/help/testing)
- [ライブテスト](/ja-JP/help/testing-live)
- [アップデートと Plugin のテスト](/ja-JP/help/testing-updates-plugins)
