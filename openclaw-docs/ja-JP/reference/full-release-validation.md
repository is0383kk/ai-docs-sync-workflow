---
doc-schema-version: 1
read_when:
    - 完全リリース検証の実行または再実行
    - 安定版リリースと完全リリースの検証プロファイルの比較
    - リリース検証ステージの失敗のデバッグ
summary: 完全リリース検証のステージ、子ワークフロー、リリースプロファイル、再実行ハンドル、エビデンス
title: 完全なリリース検証
x-i18n:
    generated_at: "2026-07-26T09:43:42Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: ddf165d5515f4b9bb11d239382649d332d20bb8a32bd4492ae99092fb5ee2216
    source_path: reference/full-release-validation.md
    workflow: 16
---

`Full Release Validation` は、リリースの製品検証を統括します。作業の大半は子ワークフローで行われるため、失敗したボックスはリリース全体を再起動せずに再実行できます。Code SHA を固定する前にリリース準備を実行してください。バックグラウンドボットが Control UI のロケール出力をまだ反映していない場合はこれを更新し、その後、リリース CI と同じ厳格なフォールバックゼロチェックを適用します。

製品が完成した changelog 前のコミットを **Code SHA** として固定し、次を実行します。

```bash
pnpm ci:full-release \
  --sha <code-sha> \
  --target-ref release/YYYY.M.PATCH
```

`provider` は、OS 横断オンボーディングとエンドツーエンドのエージェントターンに対して、`anthropic` または `minimax` も受け付けます。このヘルパーは、alpha/beta パッケージバージョンから `beta` プロファイルを推測し、それ以外の場合は `stable` を推測します。代替のワークフロー入力は `-f key=value` で渡し、広範なアドバイザリスイープに限って `-f release_profile=full` を使用します。

このヘルパーは、信頼された 1 つの `origin/main` ワークフロー SHA に固定された一時的な `release-ci/*` ref を作成し、ターゲット SHA を候補 `ref` としてのみ渡し、検証後に一時 ref を削除します。ディスパッチされたすべての子ワークフローは、同じワークフロー SHA を報告する必要があります。新規実行を強制するには `-f reuse_evidence=false` を渡し、現在の `origin/main` から引き続き到達可能な古いワークフローコミットを選択するには `--workflow-sha <trusted-main-sha>` を渡します。ワークフロー自体がリポジトリの ref を作成または更新することはありません。

## Extended-stable の例外

Extended-stable の公開には、ワークフローとターゲットの両方が正規ブランチである実行が必要です。

```bash
gh workflow run full-release-validation.yml \
  --ref extended-stable/YYYY.M.33 \
  -f ref=extended-stable/YYYY.M.33 \
  -f release_profile=stable
```

`pnpm ci:full-release` または `release-ci/*` は使用しないでください。公開では、実行のブランチ、head/target SHA、マニフェスト `workflowRef`、ID、試行回数を正規ブランチとリリースコミットに結び付けます。

製品の失敗はバックポートし、固定されたターゲットのツールには動作を維持する最小限の修正を行い、プロバイダー、承認、またはランナーの失敗はソースを変更せずに再試行します。ブランチを変更した場合は、完全な新規実行が必要です。ターゲットが古いという理由で、必須のパッケージ、インストーラー、更新、チャンネル、またはライブ動作を省略しないでください。

通常のリリースでは、Code SHA が green になったら、`CHANGELOG.md` のみを生成してコミットします。この新しいコミットが **Release SHA** です。Release SHA に対して同じヘルパーを実行します。製品エビデンスが再利用されるのは、Release SHA が Code SHA の子孫であり、変更されたパスの完全な集合が厳密に `CHANGELOG.md` であることを GitHub が証明した場合に限られます。npm preflight とパッケージ／インストール受け入れは、引き続き Release SHA に対して実行されます。

`release_profile=stable` と `release_profile=full` は、常に網羅的なライブ／Docker soak を実行します。`beta` プロファイルでも同じ soak レーンを含めるには、`run_release_soak=true` を渡します。Stable 公開では、この soak とブロッキング対象の製品パフォーマンスエビデンスがない検証マニフェストは拒否されます。

Package Acceptance は通常、`pnpm ci:full-release` でディスパッチされた完全 SHA の実行を含め、解決された `ref` から候補 tarball をビルドします。beta 公開後は、`release_package_spec=openclaw@YYYY.M.PATCH-beta.N` を渡すことで、リリースチェック、Package Acceptance、OS 横断、リリースパス Docker、パッケージ Telegram の全体で公開済み npm パッケージを再利用できます。Package Acceptance で意図的に別のパッケージを検証する場合に限って、`package_acceptance_package_spec` を使用します。Codex Plugin のライブパッケージレーンも同じ状態に従います。公開済みの `release_package_spec` 値から `codex_plugin_spec=npm:@openclaw/codex@<version>` が導出され、SHA／アーティファクト実行では選択した ref から `extensions/codex` がパックされ、オペレーターは `npm:`、`npm-pack:`、または `git:` の Plugin ソースに対して `codex_plugin_spec` を直接設定できます。このレーンは、その Plugin に必要な明示的な Codex CLI インストール承認を付与した後、Codex CLI preflight と同一セッション内の OpenAI エージェントターンを実行します。最後の再試行ゼロ・中程度思考ターンでは、Codex の `final` を省略した可視の進捗を送信し、ランダム化されたワークスペース入力を読み取り、その正確なアーティファクトを書き込み、明示的な完了を送信します。これにより、通常の進捗送信によってターンが終了していた v2026.7.1 のリグレッションを検出します。

## 最上位ステージ

`rerun_group=all` の場合、最初に `Check for reusable validation evidence` ジョブが実行されます。このジョブは、同じリリースプロファイル、有効な soak 設定、検証入力を持つ、過去の最新の green な完全検証を探します。同一ターゲットの再実行では `exact-target-full-validation-v1` を使用します。完全な差分が厳密に `CHANGELOG.md` である子孫では `changelog-only-release-v1` を使用し、すべての製品レーンをスキップして、検証ツールが GitHub のコミット比較、不変の親アーティファクト、子実行、ディスパッチログを独立して再確認します。それ以外のターゲット変更には、新しい Code SHA 検証が必要です。新しい完全実行を強制するには、`reuse_evidence=false` を渡します。エビデンスの再利用は、`main`、またはワークフローコミットが信頼された `main` 系統に残っている正規の SHA 固定 `release-ci/*` ref からのみ実行されます。その他のワークフロー ref では、選択されたレーンを新規実行します。

新規のパッケージ向け検証では、Plugin Prerelease と OpenClaw Release Checks をディスパッチする前に、1 つの不変 tarball と 1 つの Docker イメージアーティファクトを準備します。両方の子ワークフローは、使用前に同一のパッケージ SHA、アーティファクト ID、サービスダイジェスト、生成元の実行試行、Docker アーカイブダイジェストを検証します。パッケージに依存しないベア Docker レイヤーではコンテンツアドレス指定された GHCR キャッシュを使用し、候補固有のイメージは不変の GitHub アーティファクトとして維持されます。公開済みパッケージ仕様を明示した限定的な実行では、代わりに既存のパッケージパスを維持します。

また、`rerun_group=all` の場合、`Verify Docker runtime image assets` ジョブが `OPENCLAW_EXTENSIONS=diagnostics-otel,codex` を使用して `runtime-assets` Docker ターゲットをビルドします。このジョブは他のステージと並列で実行され、統括検証ツールによって適用されます。各レーンはディスパッチ前にこのジョブを待たなくなりました。より限定的な `rerun_group` では、この preflight をスキップします。

| ステージ                | 詳細                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                     |
| ----------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| ターゲット解決          | **ジョブ:** `Resolve target ref`<br />**子ワークフロー:** なし<br />**検証内容:** リリースブランチ、タグ、または完全なコミット SHA を解決し、選択された入力を記録します。<br />**再実行:** これが失敗した場合は統括ワークフローを再実行します。                                                                                                                                                                                                                                                                                                            |
| 共有候補                | **ジョブ:** `Prepare shared release candidate`<br />**子ワークフロー:** `OpenClaw Live And E2E Checks (Reusable)`<br />**検証内容:** 1 つの正確な SHA のパッケージをパックして検証し、機能する Docker イメージを 1 つビルドし、パッケージ向けの両方の子ワークフロー用に、不変のパッケージおよびイメージアーティファクトのタプルを記録します。<br />**再実行:** 影響を受けるパッケージ、Plugin prerelease、OS 横断、またはライブ／E2E グループを再実行します。                                                                                                                 |
| Docker アセット preflight | **ジョブ:** `Verify Docker runtime image assets`<br />**子ワークフロー:** なし<br />**検証内容:** 他のステージがディスパッチされる前に、`runtime-assets` Docker ビルドターゲットが引き続き成功することを検証します。`rerun_group=all` の場合にのみ実行されます。<br />**再実行:** `rerun_group=all` を指定して統括ワークフローを再実行します。                                                                                                                                                                                                                                         |
| Vitest と通常の CI      | **ジョブ:** `Run normal full CI`<br />**子ワークフロー:** `CI`<br />**検証内容:** ターゲット ref に対する手動の完全 CI グラフ。Linux Node レーン、同梱 Plugin シャード、Plugin およびチャンネル契約シャード、Node 22 互換性、`check-*`、`check-additional-*`、ビルド済みアーティファクトのスモークチェック、ドキュメントチェック、Python Skills、Windows、macOS、Control UI i18n、および統括ワークフロー経由の Android を含みます。<br />**再実行:** `rerun_group=ci`。                                                                                          |
| Plugin prerelease       | **ジョブ:** `Run plugin prerelease validation`<br />**子ワークフロー:** `Plugin Prerelease`<br />**検証内容:** リリース専用の Plugin 静的チェック、エージェント型 Plugin のカバレッジ、完全な Plugin バッチシャード、Plugin prerelease Docker レーン、および互換性トリアージ用の非ブロッキング `plugin-inspector-advisory` アーティファクト。<br />**再実行:** `rerun_group=plugin-prerelease`。                                                                                                                                                          |
| リリースチェック        | **ジョブ:** `Run release/live/Docker/QA validation`<br />**子ワークフロー:** `OpenClaw Release Checks`<br />**検証内容:** インストールスモーク、OS 横断パッケージチェック、Package Acceptance、QA Lab の同等性、ライブ Matrix と Telegram、およびゲート付きのアドバイザリ Discord、WhatsApp、Slack レーン。Stable および full プロファイルでは、網羅的なライブ／E2E スイートと Docker リリースパスチャンクも実行されます。beta では `run_release_soak=true` でオプトインできます。<br />**再実行:** `rerun_group=release-checks` または、より限定的なリリースチェックハンドル。              |
| パッケージ Telegram     | **ジョブ:** `Run package Telegram E2E`<br />**子ワークフロー:** `NPM Telegram Beta E2E`<br />**検証内容:** `release_package_spec` または `npm_telegram_package_spec` が設定されている場合の、公開済みパッケージに限定した Telegram E2E。完全な候補検証では、代わりに正規の Package Acceptance Telegram E2E を使用します。<br />**再実行:** `release_package_spec` または `npm_telegram_package_spec` を指定した `rerun_group=npm-telegram`。                                                                                                              |
| 製品パフォーマンス      | **ジョブ:** `Run product performance evidence`<br />**子ワークフロー:** `OpenClaw Performance`<br />**検証内容:** ターゲット SHA に対するリリースプロファイルのパフォーマンス実行（`profile=release`、`repeat=3`、`fail_on_regression=true`、`publish_reports=false`）。Kova の出力はワークフローアーティファクトに保持され、子ワークフローはレポート公開ジョブがスキップされたことを証明する必要があります。`rerun_group=all` または `rerun_group=performance` の場合にのみ必須（ブロッキング）であり、より限定的な再実行グループでは必須ではありません。<br />**再実行:** `rerun_group=performance`。 |
| 統括検証ツール          | **ジョブ:** `Verify full validation`<br />**子ワークフロー:** なし<br />**検証内容:** 記録された子実行の結果を再確認し、子ワークフローから最も遅いジョブの表を追記します。<br />**再実行:** 失敗した子ワークフローを再実行して green にした後、このジョブのみを再実行します。                                                                                                                                                                                                                                                                 |

統括ワークフローは、常に製品パフォーマンスをアーティファクト専用モードでディスパッチします。`OpenClaw Performance` がレポート公開を許可するのは、スケジュール実行、または `publish_reports=true` を明示的に設定した手動ディスパッチの場合に限られます。アーティファクト専用ガードは正常に完了し、公開ジョブがスキップされたままであることを証明する必要があります。新規および再利用されたエビデンスには `controls.performanceReportPublication=artifact-only` が記録されます。検証ツールと再利用セレクターは、対応する正規化済みパフォーマンス子ワークフローの証明がないエビデンスを拒否します。

検証ツールは正規マニフェストを
`full-release-validation-<run-id>-<run-attempt>` としてアップロードします。エビデンスツールは、
その正確なアーティファクト ID をダウンロードする前に、アーティファクト ID、ダイジェスト、生成元の実行、および試行を検証します。ダウンロードする ZIP のサイズを制限し、そのバイト列を REST
`sha256:` ダイジェストと照合して検証し、アーカイブを展開せずに、許可された唯一のサイズ制限付きマニフェストエントリをストリーミングします。古い公開処理の利用側のために、安定名のエイリアスが一時的に残されています。検証ツールは常に試行修飾付きアーティファクトを優先します。
移行措置として、試行 1 のマニフェスト v2
生成元に限り、安定名を受け入れます。それ以降の試行およびマニフェスト v3 では、そのレガシー名を拒否します。

`rerun_group=all` を指定した `ref=main`、`release/*` ref、および Tideclaw
alpha ref では、同じ ref と再実行グループを持つ古いアンブレラ実行は、より新しいアンブレラ実行によって置き換えられます。親がキャンセルされると、そのモニターは、すでにディスパッチしたすべての子
ワークフローをキャンセルします。タグ検証実行と固定 SHA 検証実行が互いをキャンセルすることはありません。

## リリースチェックのステージ

`OpenClaw Release Checks` は最大の子ワークフローです。ターゲットを
一度だけ解決し、利用可能な場合はアンブレラの共有パッケージアーティファクトを検証します。
直接またはフォーカス指定のディスパッチでは、パッケージまたは Docker 関連のステージで必要な場合に、独自の `release-package-under-test`
アーティファクトを準備します。

| ステージ                    | 詳細                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                           |
| ------------------------ | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| リリースターゲット           | **ジョブ:** `Resolve target ref`<br />**基盤ワークフロー:** なし<br />**テスト:** 選択した ref、任意の期待 SHA、プロファイル、再実行グループ、およびフォーカス指定のライブスイートフィルター。<br />**再実行:** `rerun_group=release-checks`。                                                                                                                                                                                                                                                                                                                                                             |
| パッケージアーティファクト         | **ジョブ:** `Prepare release package artifact`<br />**基盤ワークフロー:** なし<br />**テスト:** アンブレラの不変パッケージタプルを検証するか、Release Checks の直接／フォーカス指定ディスパッチ用に候補 tarball を 1 つパックし、後続のパッケージ関連チェックに公開します。<br />**再実行:** 影響を受けるパッケージ、クロス OS、またはライブ／E2E グループ。                                                                                                                                                                                                                                |
| インストールスモーク            | **ジョブ:** `Run install smoke`<br />**基盤ワークフロー:** `Install Smoke`<br />**テスト:** ルート Dockerfile のスモークイメージ再利用、QR パッケージのインストール、ルートおよび Gateway の Docker スモーク、インストーラーの Docker テスト、Bun グローバルインストールのイメージプロバイダースモークを含む完全なインストールパス。<br />**再実行:** `rerun_group=install-smoke`。                                                                                                                                                                                                                                                           |
| クロス OS                 | **ジョブ:** `cross_os_release_checks`<br />**基盤ワークフロー:** `OpenClaw Cross-OS Release Checks (Reusable)`<br />**テスト:** 候補 tarball とベースラインパッケージを使用し、選択したプロバイダーおよびモードについて Linux、Windows、macOS 上で新規インストールレーンとアップグレードレーンを実行します。<br />**再実行:** `rerun_group=cross-os`。                                                                                                                                                                                                                                                                 |
| リポジトリおよびライブ E2E        | **ジョブ:** `Run repo/live E2E validation`<br />**基盤ワークフロー:** `OpenClaw Live And E2E Checks (Reusable)`<br />**テスト:** リポジトリ E2E、ライブキャッシュ、OpenAI WebSocket ストリーミング、ネイティブのライブプロバイダーおよび Plugin シャード、さらに `release_profile` で選択された Docker ベースのライブモデル／バックエンド／Gateway ハーネス。<br />**実行:** `run_release_soak=true`、`release_profile=full`、またはフォーカス指定の `rerun_group=live-e2e`。<br />**再実行:** `rerun_group=live-e2e`。任意で `live_suite_filter` も指定できます。                                                                                |
| Docker リリースパス      | **ジョブ:** `Run Docker release-path validation`<br />**基盤ワークフロー:** `OpenClaw Live And E2E Checks (Reusable)`<br />**テスト:** 共有パッケージアーティファクトに対するリリースパスの Docker チャンク。<br />**実行:** `run_release_soak=true`、`release_profile=full`、またはフォーカス指定の `rerun_group=live-e2e`。<br />**再実行:** `rerun_group=live-e2e`。                                                                                                                                                                                                                                     |
| パッケージ受け入れテスト       | **ジョブ:** `Run package acceptance`<br />**基盤ワークフロー:** `Package Acceptance`<br />**テスト:** オフライン Plugin パッケージフィクスチャ、Plugin の更新、正規のモック OpenAI Telegram パッケージ E2E、および同じ tarball に対する公開済みバージョンからのアップグレード継続性チェック。ブロッキングリリースチェックでは、デフォルトで公開済みの最新ベースラインを使用します。ソークチェック（`run_release_soak=true`）では、直近 4 つの安定版 npm リリースと、固定された過去の 3 バージョン（`2026.4.23`、`2026.5.2`、`2026.4.15`）まで対象を拡大し、報告済み問題のアップグレードフィクスチャに対して実行します。<br />**再実行:** `rerun_group=package`。 |
| 成熟度スコアカード       | **ジョブ:** `Render maturity scorecard release docs`<br />**基盤ワークフロー:** `maturity-scorecard.yml`<br />**テスト:** ターゲット ref に対して参考情報としての成熟度スコアカードドキュメントをレンダリングします。`run_maturity_scorecard=true` が渡された場合にのみ実行されます。<br />**再実行:** `run_maturity_scorecard=true` を指定した `rerun_group=qa`。                                                                                                                                                                                                                                                           |
| QA パリティ                | **ジョブ:** `Run QA Lab parity lane` および `Run QA Lab parity report`<br />**基盤ワークフロー:** 直接ジョブ<br />**テスト:** 候補とベースラインのエージェント型パリティパック、その後にパリティレポート。<br />**再実行:** `rerun_group=qa-parity` または `rerun_group=qa`。                                                                                                                                                                                                                                                                                                                         |
| QA ランタイムパリティ        | **ジョブ:** `Verify QA Lab runtime-pair lanes`<br />**基盤ワークフロー:** 直接ジョブ<br />**テスト:** 正規のコア `openclaw`/`codex` レーン（`pnpm openclaw qa suite --runtime-pair openclaw,codex --runtime-pair-lane core`）、および `run_release_soak=true` を指定した場合のソークレーン。参考情報: 個々のレーンジョブは、リリースチェック検証ツールをブロックしません。<br />**再実行:** `rerun_group=qa-parity` または `rerun_group=qa`。                                                                                                                                                             |
| QA ランタイムツールカバレッジ | **ジョブ:** `Enforce QA Lab runtime tool coverage`<br />**基盤ワークフロー:** 直接ジョブ<br />**テスト:** 正規のコアランタイムペアレーン（`pnpm openclaw qa coverage --tools`）で、そのレーンの出力を使用した `openclaw` と `codex` 間の動的ツールドリフト。ブロッキング: このジョブは参考情報扱いに上書きできません。<br />**再実行:** `rerun_group=qa-parity` または `rerun_group=qa`。                                                                                                                                                                                                     |
| QA ライブ Matrix           | **ジョブ:** `Run QA Live Matrix profile`<br />**基盤ワークフロー:** `QA-Lab - All Lanes` 再利用可能ワークフロー<br />**テスト:** `qa-live-shared` 環境内の共有 Matrix ライブアダプターを通じた、パリティが証明済みの YAML シナリオ。<br />**再実行:** `rerun_group=qa-live` または `rerun_group=qa`。Matrix にフォーカスした再実行には `live_suite_filter=qa-live-matrix` を使用します。                                                                                                                                                                                                                    |
| QA ライブ Telegram         | **ジョブ:** `Run QA Lab live Telegram lane`<br />**基盤ワークフロー:** 信頼済みの `OpenClaw Release Telegram QA` ディスパッチ<br />**テスト:** Convex CI の認証情報リースを使用するライブ Telegram QA。<br />**再実行:** `rerun_group=qa-live` または `rerun_group=qa`。                                                                                                                                                                                                                                                                                                                                 |
| QA ライブ Discord          | **ジョブ:** `Run QA Lab live Discord lane`<br />**基盤ワークフロー:** 直接の参考情報ジョブ<br />**テスト:** `OPENCLAW_RELEASE_QA_DISCORD_LIVE_CI_ENABLED` が有効な場合に、Convex CI の認証情報リースを使用するライブ Discord QA。<br />**再実行:** `live_suite_filter=qa-live-discord` を指定した `rerun_group=qa-live`。                                                                                                                                                                                                                                                                            |
| QA ライブ WhatsApp         | **ジョブ:** `Run QA Lab live WhatsApp lane`<br />**基盤ワークフロー:** 直接の参考情報ジョブ<br />**テスト:** `OPENCLAW_RELEASE_QA_WHATSAPP_LIVE_CI_ENABLED` が有効な場合に、Convex CI の認証情報リースを使用するライブ WhatsApp QA。<br />**再実行:** `live_suite_filter=qa-live-whatsapp` を指定した `rerun_group=qa-live`。                                                                                                                                                                                                                                                                        |
| QA ライブ Slack            | **ジョブ:** `Run QA Lab live Slack lane`<br />**基盤ワークフロー:** 直接の参考情報ジョブ<br />**テスト:** `OPENCLAW_RELEASE_QA_SLACK_LIVE_CI_ENABLED` が有効な場合に、Convex CI の認証情報リースを使用するライブ Slack QA。<br />**再実行:** `live_suite_filter=qa-live-slack` を指定した `rerun_group=qa-live`。                                                                                                                                                                                                                                                                                    |
| リリース検証ツール         | **ジョブ:** `Verify release checks`<br />**基盤ワークフロー:** なし<br />**テスト:** 選択した再実行グループに必要なリリースチェックジョブ。<br />**再実行:** フォーカス指定の子ジョブが成功した後に再実行します。                                                                                                                                                                                                                                                                                                                                                                                   |

## Docker リリースパスのチャンク

Docker リリースパスステージでは、`live_suite_filter` が空の場合に次のチャンクを実行します。

| チャンク                                                           | カバレッジ                                                                                                                                     |
| --------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------- |
| `core`                                                          | コア Docker リリースパスのスモークレーン。                                                                                                        |
| `package-update-openai`                                         | OpenAI パッケージのインストール／更新動作、Codex のオンデマンドインストール、Codex Plugin のライブ進捗の完了確認、および Chat Completions ツール呼び出し。 |
| `package-update-anthropic`                                      | Anthropic パッケージのインストールおよび更新動作。                                                                                               |
| `package-update-core`                                           | プロバイダーに依存しないパッケージおよび更新動作。                                                                                                |
| `plugins-runtime-plugins`                                       | Plugin の動作を検証する Plugin ランタイムレーン。                                                                                          |
| `plugins-runtime-services`                                      | サービスを使用するライブ Plugin ランタイムレーン。                                                                                                |
| `plugins-runtime-install-a` から `plugins-runtime-install-h` | 並列リリース検証用に分割された Plugin のインストール／ランタイムバッチ。                                                                        |
| `openwebui`                                                     | 要求された場合に専用の大容量ディスクランナーで分離して実行する OpenWebUI 互換性スモーク。                                                      |

Docker レーンが 1 つだけ失敗した場合は、再利用可能なライブ／E2E ワークフローで対象を絞った `docker_lanes=<lane[,lane]>` を使用します。リリース成果物には、利用可能な場合、パッケージ成果物とイメージを再利用する入力を指定したレーンごとの再実行コマンドが含まれます。

## リリースプロファイル

`release_profile` は主にリリースチェック内のライブ／プロバイダーの対象範囲を制御します。
通常のフル CI、Plugin Prerelease、インストールスモーク、パッケージ受け入れ、QA Lab は除外されません。stable プロファイルと full プロファイルでは、常にリポジトリ／ライブ E2E と Docker リリースパスの網羅的な長時間カバレッジを実行します。beta プロファイルでは、`run_release_soak=true` を使用してオプトインできます。パッケージ受け入れでは、すべての完全な候補に対して標準のパッケージ Telegram E2E が提供されるため、包括ワークフローではそのライブポーラーを重複して実行しません。

| プロファイル  | 用途                      | 含まれるライブ／プロバイダーカバレッジ                                                                                                                                                                            |
| -------- | --------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `beta`   | 最速のリリース必須スモーク。   | OpenAI／コアのライブパス、OpenAI 用 Docker ライブモデル、ネイティブ Gateway コア、ネイティブ OpenAI Gateway プロファイル、ネイティブ OpenAI Plugin、および Docker ライブ Gateway OpenAI。                                            |
| `stable` | デフォルトのリリース承認プロファイル。 | `beta` に加えて、Anthropic スモーク、Google、MiniMax、バックエンド、ネイティブライブテストハーネス、Docker ライブ CLI バックエンド、Docker ACP バインド、Docker Codex ハーネス、Docker サブエージェント通知、および OpenCode Go スモークシャード。 |
| `full`   | 広範な勧告用スイープ。             | `stable` に加えて、勧告対象プロバイダー、Plugin ライブシャード、およびメディアライブシャード。                                                                                                                               |

## full のみの追加項目

これらのスイートは `stable` ではスキップされ、`full` では含まれます。

| 領域                             | full のみのカバレッジ                                                                                                          |
| -------------------------------- | --------------------------------------------------------------------------------------------------------------------------- |
| Docker ライブモデル               | OpenCode Go、OpenRouter、xAI、Z.ai、Fireworks。                                                                          |
| Docker ライブ Gateway              | 勧告対象プロバイダーを DeepSeek／Fireworks、OpenCode Go／OpenRouter、xAI／Z.ai の各シャードに分割。                              |
| ネイティブ Gateway プロバイダープロファイル | 完全な Anthropic Opus および Sonnet／Haiku シャード、Fireworks、DeepSeek、完全な OpenCode Go モデルシャード、OpenRouter、xAI、Z.ai。 |
| ネイティブ Plugin ライブシャード        | Plugin A～K、L～N、その他 O～Z、Moonshot、xAI。                                                                             |
| ネイティブメディアライブシャード         | 音声、Google 音楽、MiniMax 音楽、および動画グループ A～D。                                                                   |

`stable` には `native-live-src-gateway-profiles-anthropic-smoke` と
`native-live-src-gateway-profiles-opencode-go-smoke` が含まれます。代わりに `full` では、より広範な
Anthropic および OpenCode Go モデルシャードを使用します。対象を絞った再実行でも、集約された
`native-live-src-gateway-profiles-anthropic` または
`native-live-src-gateway-profiles-opencode-go` ハンドルを使用できます。

## 対象を絞った再実行

無関係なリリースボックスの再実行を避けるには、`rerun_group` を使用します。

| ハンドル              | 範囲                                                                                           |
| ------------------- | ----------------------------------------------------------------------------------------------- |
| `all`               | Full Release Validation の全ステージ。                                                             |
| `ci`                | 手動フル CI の子ワークフローのみ。                                                                      |
| `plugin-prerelease` | Plugin Prerelease の子ワークフローのみ。                                                                   |
| `release-checks`    | OpenClaw Release Checks の全ステージ。                                                             |
| `install-smoke`     | Install Smoke からリリースチェックまで。                                                           |
| `cross-os`          | クロス OS リリースチェック。                                                                        |
| `live-e2e`          | リポジトリ／ライブ E2E および Docker リリースパス検証。                                               |
| `package`           | パッケージ受け入れ。                                                                             |
| `qa`                | QA パリティと QA ライブレーン。                                                                   |
| `qa-parity`         | QA パリティレーンとレポートのみ。                                                                |
| `qa-live`           | QA ライブ Matrix／Telegram、および有効化されている場合はゲート付き Discord、WhatsApp、Slack レーン。             |
| `npm-telegram`      | 公開済みパッケージの Telegram E2E。`release_package_spec` または `npm_telegram_package_spec` が必要です。 |
| `performance`       | 製品パフォーマンスのエビデンスのみ。                                                              |

1 つのライブスイートが失敗した場合は、`live_suite_filter` と `rerun_group=live-e2e` を使用します。
有効なフィルター ID は再利用可能なライブ／E2E ワークフローで定義されており、
`docker-live-models`、`live-gateway-docker`、
`live-gateway-anthropic-docker`、`live-gateway-google-docker`、
`live-gateway-minimax-docker`、`live-gateway-advisory-docker`、
`live-cli-backend-docker`、`live-acp-bind-docker`、および
`live-codex-harness-docker` が含まれます。

対象を絞った QA トランスポートの再実行では、`rerun_group=qa-live` を設定し、標準セレクター
`qa-live-matrix`、`qa-live-telegram`、`qa-live-discord`、
`qa-live-whatsapp`、または `qa-live-slack` を使用します。

`live-gateway-advisory-docker` ハンドルは 3 つのプロバイダーシャードをまとめた再実行ハンドルであるため、引き続きすべての勧告対象 Docker Gateway ジョブへファンアウトします。

1 つのクロス OS レーンが失敗した場合は、`cross_os_suite_filter` と `rerun_group=cross-os` を使用します。
フィルターには OS ID、スイート ID、または OS／スイートの組み合わせを指定できます。たとえば、`windows/packaged-upgrade`、`windows`、`packaged-fresh` です。クロス OS の概要には、パッケージ化されたアップグレードレーンのフェーズごとの所要時間が含まれます。また、長時間実行されるコマンドでは Heartbeat 行が出力されるため、停止した更新をジョブのタイムアウト前に確認できます。

QA リリースチェックの失敗によって通常のリリース検証がブロックされるのは、選択された Matrix、Telegram、および QA ランタイムツールのカバレッジレーンのみです。QA パリティ、ランタイムパリティ、およびゲート付きの Discord、WhatsApp、Slack ライブレーンは勧告扱いで、リリース検証をブロックせずにステータス成果物を公開します。Tideclaw の alpha 実行では、パッケージの安全性に関係しないリリースチェックレーンを引き続き勧告扱いにできます。`release_profile=beta` では、`Run repo/live E2E validation` ライブプロバイダースイートは勧告扱いです。サードパーティのモデルデプロイメントはリリースの進行中にも変化するため、beta ではその失敗を警告として表示し、stable プロファイルと full プロファイルでは引き続きブロッキング扱いにします。
`live_suite_filter` が Discord、WhatsApp、Slack などのゲート付き QA ライブレーンを明示的に要求する場合、対応する `OPENCLAW_RELEASE_QA_*_LIVE_CI_ENABLED` リポジトリ変数を有効にする必要があります。有効でない場合、レーンを暗黙にスキップするのではなく、入力取得が失敗します。
新しい QA エビデンスが必要な場合は、`rerun_group=qa`、`qa-parity`、または `qa-live` を再実行します。

## 保持するエビデンス

リリースレベルの索引として `Full Release Validation` の概要を保持します。これには子実行 ID へのリンクと、最も遅いジョブの表が含まれます。失敗時は、最初に子ワークフローを調査し、その後、上記の最小限の該当ハンドルを再実行します。

通常リリースでは、Code SHA と Release SHA の両方、再利用ポリシーと変更パスのセット、成功した Code SHA の親実行、および軽量な Release SHA の親実行を記録します。extended-stable では、標準ブランチ、正確なリリース SHA、新しい親実行 ID と試行回数、ワークフロー参照、すべての子実行、および固定対象の互換性修復または意図的な省略を記録します。

有用な成果物：

- `release-package-under-test`（`OpenClaw Release Checks` から）
- `.artifacts/docker-tests/` 配下の Docker リリースパス成果物
- パッケージ受け入れの `package-under-test` および Docker 受け入れ成果物
- 各 OS およびスイートのクロス OS リリースチェック成果物
- QA パリティ、ランタイムパリティ、および選択された Matrix、Telegram、Discord、WhatsApp、
  または Slack の成果物

## ワークフローファイル

- `.github/workflows/full-release-validation.yml`
- `.github/workflows/openclaw-release-checks.yml`
- `.github/workflows/openclaw-live-and-e2e-checks-reusable.yml`
- `.github/workflows/plugin-prerelease.yml`
- `.github/workflows/install-smoke.yml`
- `.github/workflows/install-smoke-reusable.yml`
- `.github/workflows/openclaw-cross-os-release-checks-reusable.yml`
- `.github/workflows/package-acceptance.yml`
- `.github/workflows/openclaw-performance.yml`
- `.github/workflows/npm-telegram-beta-e2e.yml`
