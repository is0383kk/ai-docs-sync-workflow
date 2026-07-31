---
doc-schema-version: 1
read_when:
    - 公開リリースチャンネルの定義を探しています
    - リリース検証またはパッケージ受け入れテストの実行
    - バージョンの命名規則とリリース周期を確認する
summary: リリースレーン、運用担当者向けチェックリスト、検証ボックス、バージョン命名規則、リリース頻度
title: リリースポリシー
x-i18n:
    generated_at: "2026-07-26T10:00:19Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: de2429f039bb42deabdcfe280b7d91afac3bae3dc24714203ab7a67672dcc10c
    source_path: reference/RELEASING.md
    workflow: 16
---

OpenClaw は、ユーザー向けに 4 つの更新チャネルを提供します。

- stable: npm 上で昇格された通常リリース `latest`
- extended-stable: npm 上の、完了した直前月の `.33+` メンテナンスライン
  `extended-stable`
- beta: npm 上のプレリリースタグ `beta`
- dev: `main` の更新され続ける最新ヘッド

Extended-stable では、通常の `latest` または `main` セレクターを変更せずに、直前月の Gateway、公式 npm plugins、Docker イメージを提供します。

Tideclaw alpha ビルドは独立した内部プレリリーストラック（npm dist-tag `alpha`）であり、[NPM ワークフロー入力](#npm-workflow-inputs)および[リリーステストボックス](#release-test-boxes)で説明しています。

## バージョン命名規則

- 月次 Gateway extended-stable リリースバージョン: `YYYY.M.PATCH`、`PATCH >= 33` 付き、git タグ `vYYYY.M.PATCH`
- 日次／通常の正式リリースバージョン: `YYYY.M.PATCH`、`PATCH < 33` 付き、git タグ `vYYYY.M.PATCH`
- 通常のフォールバック修正リリースバージョン: `YYYY.M.PATCH-N`、git タグ `vYYYY.M.PATCH-N`
- Beta プレリリースバージョン: `YYYY.M.PATCH-beta.N`、git タグ `vYYYY.M.PATCH-beta.N`
- Alpha プレリリースバージョン: `YYYY.M.PATCH-alpha.N`、git タグ `vYYYY.M.PATCH-alpha.N`
- 月またはパッチをゼロ埋めしない
- `PATCH` は暦日ではなく、連続した月次リリーストレイン番号です。通常の正式リリースと beta リリースは現在のトレインを進めます。alpha のみのタグが beta／通常リリースのパッチ番号を消費または進行させることはないため、beta または通常リリースのトレインを選択するときは、より大きなパッチ番号を持つ旧式の alpha のみのタグを無視してください。
- Alpha／nightly ビルドは、次の未リリースのパッチトレインを使用し、反復ビルドでは `alpha.N` のみを増加させます。そのパッチに beta が作成されると、新しい alpha ビルドは次のパッチへ移ります。
- npm バージョンは不変です。公開済みタグを削除、再公開、または再利用してはなりません。代わりに次のプレリリース番号または次の月次パッチを作成してください。
- `latest` は引き続き現在の通常／日次 npm ラインに従います。`beta` は現在の beta インストールターゲットです
- `extended-stable` は、パッチ `33` から始まる、サポート対象の直前月 Gateway ディストリビューションを意味します。パッチ `34` 以降は、その月次ラインのメンテナンスリリースです
- 通常の正式リリースと通常の修正リリースは、デフォルトで npm `beta` に公開されます。リリース担当者は `latest` を明示的に対象にすることも、検証済みの beta ビルドを後から昇格させることもできます
- Gateway extended-stable は、コア、npm に公開可能なすべての公式 Plugin、
  およびその Docker イメージを 1 つの完全に同一なバージョンで公開します。以下の専用ワークフローを参照してください。
- 通常の正式リリースでは毎回、npm パッケージ、macOS アプリ、署名済みのスタンドアロン Android APK、署名済みの Windows Hub インストーラーをまとめて提供します。Beta リリースでは通常、最初に npm／パッケージ経路を検証して公開し、ネイティブアプリのビルド／署名／公証／昇格は、明示的に要求されない限り通常の正式リリース時に行います。

## リリース頻度

- リリースは beta 優先で進みます。stable は最新の beta が検証された後にのみ続きます
- メンテナーは通常、現在の `main` から作成した `release/YYYY.M.PATCH` ブランチからリリースを作成します。これにより、リリースの検証と修正が `main` 上の新規開発を妨げません
- Beta タグが push または公開された後に修正が必要になった場合、メンテナーは古いタグを削除または再作成せず、次の `-beta.N` タグを作成します
- 詳細なリリース手順、承認、認証情報、復旧メモはメンテナー専用です

## 月次 Gateway extended-stable の公開

完了した月 `YYYY.M` について、`extended-stable/YYYY.M.33` を作成し、そのブランチから
`.33+` を公開します。タグ、ブランチ、チェックアウト、パッケージバージョン、事前チェック、検証は、
1 つのコミットを特定する必要があります。`.33` より前に、保護された `main` には
パッチ `33` 未満の翌月以降の正式バージョンが含まれている必要があります。それ以降のメンテナンスパッチも
引き続き対象です。

### 候補を準備して安定化する

未監査の mainline 範囲を監査し、非公開のセキュリティ作業を調整し、
範囲を限定したバックポートセットを承認して、連携された 1 つの PR を取り込みます。正規
ブランチへ直接 push してはなりません。

正規ブランチで `YYYY.M.P` を設定し、`pnpm release:prep` を実行して、
公開可能なすべての公式 Plugin でそのバージョンを必須にします。承認済み台帳から、
`### Highlights`、`### Changes`、`### Fixes` を含む完全な `## YYYY.M.P` セクションを生成してコミットし、
同等のバックポートについて元のマージ済み `main` PR を引用します。セクションがない場合や空の場合、事前チェックは拒否します。

現在の main にある Docker リリースチャネルの一式（ワークフロー、昇格処理、
ポリシー、共有分類器、テスト、ワークフロー検証）をすべて移植します。GitHub はタグ付けされたコミットからタグ用
ワークフローを読み込みます。不完全なコピーは、ビルド後に失敗したり、
通常のエイリアスを移動したりする可能性があります。対象を絞ったチェックを実行してください。

ブランチ先端全体の SHA を固定します。タグ付けの前に、その正確な npm バイトを事前チェックし、
その SHA に対して Full Release Validation を実行します。

```bash
RELEASE_SHA="$(git rev-parse HEAD)"

gh workflow run openclaw-npm-release.yml \
  --ref extended-stable/YYYY.M.33 \
  -f tag="$RELEASE_SHA" \
  -f preflight_only=true \
  -f npm_dist_tag=extended-stable

gh workflow run full-release-validation.yml \
  --ref extended-stable/YYYY.M.33 \
  -f ref=extended-stable/YYYY.M.33 \
  -f release_profile=stable
```

SHA 形式は事前チェック専用です。正規ブランチで検証を実行します。公開処理では、
そのワークフローの ref、head／target SHA、実行 ID、試行回数を紐付けます。両方の ID と
成功した `run_attempt` を保存し、`release-ci/*` の証拠は拒否してください。

編集前に失敗を分類します。

- 製品: 承認済みの別のバックポート PR を取り込む。
- 固定ターゲットのツール: 古い製品を変更せずにテストする、最小限の互換性修正のみをバックポートする。
- プロバイダー、承認、ランナー、またはサービス: 候補を変更せず、範囲を限定した再試行経路を使用する。

ブランチを変更すると、両方のゲートが無効になります。両方に合格したら、先端が引き続き
`RELEASE_SHA` と等しいことを確認し、署名済み `vYYYY.M.P` を push します。それ以降の変更には次の
パッチが必要です。タグを移動または削除してはなりません。その push によって `Docker Release` が開始されます。

### npm パッケージを公開する

同じ SHA から npm に公開可能なすべての公式 Plugin を公開し、
成功した実行 ID を保存します。

```bash
RELEASE_SHA="$(git rev-parse HEAD)"
gh workflow run plugin-npm-release.yml \
  --ref extended-stable/YYYY.M.33 \
  -f publish_scope=all-publishable \
  -f ref="$RELEASE_SHA" \
  -f npm_dist_tag=extended-stable
```

ワークフローは未変更のものを含むすべての `all-publishable` パッケージを対象とし、
各パッケージの正確なバージョンとセレクターを検証します。再実行では公開済みバージョンを再利用します。

次に、保存した 3 つすべての実行識別情報を使用して、準備済みのコア tarball を公開します。

```bash
gh workflow run openclaw-npm-release.yml \
  --ref extended-stable/YYYY.M.33 \
  -f tag=vYYYY.M.P \
  -f preflight_only=false \
  -f npm_dist_tag=extended-stable \
  -f preflight_run_id=<npm-preflight-run-id> \
  -f full_release_validation_run_id=<full-validation-run-id> \
  -f full_release_validation_run_attempt=<full-validation-run-attempt> \
  -f plugin_npm_run_id=<plugin-npm-run-id>
```

本番以外のリハーサルの場合に限り、事前チェックと公開に
`-f bypass_extended_stable_guard=true` を追加します。これは月のガードのみを迂回し、
正規 ref、SHA／タグ／バージョンの一致、来歴、承認、または読み戻しチェックを迂回することはありません。本番には絶対に使用しないでください。

### 検証と復旧

固定ブランチではなく、現在の `main` の別のクリーンなチェックアウトから、次を実行します。

```bash
node --import tsx scripts/openclaw-npm-postpublish-verify.ts YYYY.M.P
npm view openclaw@YYYY.M.P version --userconfig "$(mktemp)"
npm view openclaw@extended-stable version --userconfig "$(mktemp)"
```

正規ブランチの署名と npm provenance に加え、公開処理、
事前チェック、tarball ダイジェストがリリース SHA に紐付いていることを必須とします。両方のコマンドは
`YYYY.M.P` を返す必要があります。準備済みの各コアパッケージと `all-publishable`
個の公式 Plugin について、正確なバージョンとセレクターを検証します。

ルートセレクターのみが失敗した場合は、ワークフローの概要に出力された
生成済みの `npm dist-tag add openclaw@YYYY.M.P extended-stable` 修復コマンドを使用します。
既存の Plugin またはその他の準備済みコアのセレクターは、承認済みの認証情報分離ツールを通じて修復します。OIDC ソースでは変更できません。
不変のバージョンを再公開してはなりません。

`Docker Release` では、GHCR と Docker Hub にあるデフォルト、slim、ブラウザー、アーキテクチャ別の
正確なイメージ（証明およびプラットフォームバージョンを含む）を検証する必要があります。ダイジェストによって進める対象は
`extended-stable`、`extended-stable-slim`、`extended-stable-browser` のみに
限定する必要があります。通常のエイリアスは変更せず、自動ロールバックは拒否されます。

エイリアスを修復するには、現在の
`main` からタグを指定して、承認ゲート付きの `Docker Channel Promotion` を実行します。ダイジェスト、証明、プラットフォームのチェックを繰り返し、
明示的なロールバックを許可しますが、イメージを再ビルドすることはありません。

Slack、Discord、Codex は最初に文書化されたサポート対象であり、
リリース許可リストではありません。npm に公開可能なすべての公式 Plugin が提供されます。beta／`latest`、GitHub Releases、ClawHub、ネイティブアプリ、モバイル、
Web サイト、非公開 dist-tag は通常リリースのチェックリストのみが管理します。この Gateway 経路では、それらの手順を実行しないでください。

## 通常リリース担当者向けチェックリスト

このチェックリストは、リリースフローの公開形態を示します。非公開の認証情報、署名、公証、dist-tag の復旧、緊急ロールバックの詳細は、メンテナー専用のリリース運用手順書に残します。

1. 現在の `main` から開始します。最新を pull し、対象コミットが push 済みであること、および `main` の CI がブランチ作成に十分な状態で green であることを確認します。
2. そのコミットから `release/YYYY.M.PATCH` を作成します。バックポートは任意です。担当者が選択したセットのみを適用します。必要なすべてのバージョン箇所を更新し、`pnpm release:prep` を実行し、リリース修正と必要な forward-port を完了して、`src/plugins/compat/registry.ts` と `src/commands/doctor/shared/deprecation-compat.ts` を確認します。
3. 製品として完成した変更履歴作成前のコミットを **Code SHA** として固定します。決定論的なソース事前チェックを実行し、次に `node scripts/full-release-validation-at-sha.mjs --sha <code-sha> --target-ref release/YYYY.M.PATCH` を使用します。これにより、信頼済みのワークフローツールを固定しつつ、Vitest、Docker、QA、パッケージ、パフォーマンスの全マトリクスが正確な Code SHA を対象にします。
4. 編集前に失敗を分類します。製品／コードの失敗では新しい Code SHA が作成され、その SHA に対する Full Release Validation の合格が必要です。ワークフロー、ハーネス、認証情報、承認、またはインフラストラクチャの失敗は、その所有範囲で修復し、同じ Code SHA に対して再実行します。
5. Code SHA が green になった後に限り、到達可能な最後の出荷済みタグ以降のマージ済み PR と直接コミットから、先頭の `CHANGELOG.md` セクションを生成します。項目はユーザー向けにし、重複を排除します。分岐した出荷済みタグまたは後続の forward-port により、すでにリリース済みの PR が再関連付けされる場合は、`--shipped-ref` として明示的に渡します。
6. `CHANGELOG.md` のみをコミットします。このコミットが **Release SHA** です。Code SHA から Release SHA までの完全な差分は、正確に `CHANGELOG.md` でなければなりません。それ以外のパスが変更されている場合、リリースは手順 2 に戻ります。
7. 証拠の再利用を有効にして、Release SHA に対する SHA 固定の Full Release Validation を実行します。軽量の親は `changelog-only-release-v1` を記録し、green の Code SHA を参照し、製品の子レーンを一切 dispatch しないことが必須です。これは製品の証拠を再利用しますが、パッケージのバイト列は再利用しません。
8. Release SHA／タグに対して、`preflight_only=true` を指定して `OpenClaw NPM Release` を実行します。成功した `preflight_run_id` を保存します。これにより、最終的な変更履歴を含む正確なパッケージのバイト列をビルドしてチェックします。
9. Release SHA にタグを付け、次に成功した Release-SHA 検証の親と npm 事前チェックを指定して候補ヘルパーを実行します。どちらも再度 dispatch しないでください。

   ```bash
   pnpm release:candidate -- \
     --tag vYYYY.M.PATCH-beta.N \
     --full-release-run <release-sha-validation-run-id> \
     --npm-preflight-run <preflight-run-id> \
     --skip-dispatch
   ```

   安定版の場合は、`--windows-node-tag vX.Y.Z` も渡します。このヘルパーは、リリースノートの出所、npm プリフライトのバイト列、Parallels のインストール／更新証明、Telegram パッケージ証明、Plugin 公開計画を検証してから、公開コマンドを出力します。

   `OpenClaw Release Publish` は、選択された、または公開可能なすべての Plugin パッケージを npm に、同じ一式を ClawHub に並列でディスパッチし、Plugin の npm 公開が成功すると、準備済みの OpenClaw npm プリフライトアーティファクトを対応する dist-tag で昇格させます。リリースチェックアウトはプロダクト／データのルートとして維持されますが、計画と最終検証は、正確な信頼済みワークフローソースのチェックアウトから実行されるため、古いリリースコミットが廃止済みのリリースツールを暗黙に使用することはありません。公開子処理が開始される前に、正確な GitHub リリース本文をレンダリングしてキャッシュします。完全に一致する `CHANGELOG.md` セクションが GitHub の 125,000 文字制限とレンダラーの対応する 125,000 バイトの安全上限に収まる場合、ページには見出しを含むその正確な `## YYYY.M.PATCH` セクションが記載されます。ソースセクションが収まらない場合、ページでは正確にグループ化された編集済みノートを維持し、サイズ超過のコントリビューション記録を、タグに固定された `CHANGELOG.md` 内の完全な記録への安定したリンクに置き換えます。部分的な記録や途中で切れた箇条書きは公開されません。ワークフローは `### Release verification` を追加する前に完全版またはコンパクト版の本文を選択します。証明の末尾を追加すると制限を超える場合は、正規の本文を維持し、代わりに変更不能な添付証拠を使用します。npm `latest` に公開された安定版リリースは GitHub の最新リリースになり、npm `beta` に維持された安定版メンテナンスリリースは GitHub `latest=false` を指定して作成されます。また、リリース後のインシデント対応用に、プリフライト依存関係証拠、完全検証マニフェスト、公開後のレジストリ検証証拠を GitHub リリースへアップロードします。子実行 ID を直ちに出力し、ワークフロートークンによる承認が許可されたリリース環境ゲートを自動承認し、失敗した子ジョブをログ末尾とともに要約します。さらに、ドラフト GitHub リリースページを最初に作成し、OpenClaw の npm 公開と並行して Windows および Android アセットを昇格させ、それらのステージが成功するとリリースページと依存関係証拠を完成させます。OpenClaw npm が公開される場合は ClawHub を待機してから、信頼済み main のベータ検証ツールを実行し、GitHub リリース、npm パッケージ、選択された Plugin npm パッケージ、選択された ClawHub パッケージ、子ワークフロー実行 ID、任意の NPM Telegram 実行 ID に関する公開後証拠をアップロードします。ClawHub ブートストラップ検証ツールには、正確な信頼済み main のワークフローパスと SHA、生成側および最終実行の試行、リリース SHA、要求されたパッケージ一式、変更不能なパッケージアーティファクトのタプル、最終的なレジストリ読み戻しアーティファクトが必要です。成功した従来のリリース参照実行は受け入れられません。

   次に、公開済みの `openclaw@YYYY.M.PATCH-beta.N` または `openclaw@beta` パッケージに対して、公開後のパッケージ受け入れ検証を実行します。プッシュ済みまたは公開済みのプレリリースに修正が必要な場合は、対応する次のプレリリース番号を作成します。古いものを削除または書き換えてはいけません。

10. 公開試行が失敗した場合、失敗によってプロダクトまたは変更履歴の不具合が証明されない限り、Release SHA は変更しません。成功済みの変更不能な子処理とアーティファクトを再開し、すでに成功したパッケージバージョンを再ビルドまたは再公開してはいけません。
11. 安定版では、精査済みのベータ版またはリリース候補に必要な検証証拠が揃っている場合にのみ続行します。安定版の npm 公開も `OpenClaw Release Publish` を経由し、`preflight_run_id` により成功済みのプリフライトアーティファクトを再利用します。安定版 macOS リリースの準備完了には、パッケージ化された `.zip`、`.dmg`、`.dSYM.zip`、および `main` 上の更新済み `appcast.xml` も必要です。macOS 公開ワークフローは、リリースアセットの検証後、署名済み appcast を公開 `main` に自動公開します。ブランチ保護によって直接プッシュが阻止された場合は、appcast PR を作成または更新します。安定版 Windows Hub の準備完了には、OpenClaw GitHub リリース上の署名済み `OpenClawCompanion-Setup-x64.exe`、`OpenClawCompanion-Setup-arm64.exe`、`OpenClawCompanion-SHA256SUMS.txt` アセットが必要です。正確な署名済み `openclaw/openclaw-windows-node` リリースタグを `windows_node_tag` として、その候補承認済みインストーラーダイジェストマップを `windows_node_installer_digests` として渡します。`OpenClaw Release Publish` はリリースをドラフトのまま維持し、`Windows Node Release` をディスパッチして、公開前に 3 つすべてのアセットを検証します。
12. 公開後、npm 公開後検証ツールを実行し、公開後のチャンネル証明が必要な場合は任意のスタンドアロン公開済み npm Telegram E2E を実行します。必要に応じて dist-tag を昇格させ、生成された GitHub リリースページを検証し、リリース告知手順を実行してから、安定版リリースの完了を宣言する前に[安定版 main のクローズアウト](#stable-main-closeout)を完了します。

## 安定版 main のクローズアウト

`main` に実際に出荷されたリリース状態が反映されるまで、安定版の公開は完了していません。

1. 最新の `main` から開始します。それに対して `release/YYYY.M.PATCH` を監査し、`main` に存在しない実際の修正をフォワードポートします。リリース専用の互換性、テスト、検証アダプターを、より新しい `main` に無条件でマージしてはいけません。
2. 通常の手順では、`main` を出荷済みの安定版バージョンに設定します。遅れて行うクローズアウトでは、`main` が後の安定版 OpenClaw CalVer に進んだ後であれば、それを使用できます。前のリリースをクローズするためだけに、すでに開始済みのリリース系列をダウングレードしてはいけません。検証ツールは引き続き、正確な出荷済み変更履歴セクションと appcast エントリを要求し、実際の `main` バージョンと SHA を記録します。ルートバージョンを変更した場合は `pnpm release:prep` を実行し、その後 `pnpm deps:shrinkwrap:generate` を実行します。
3. `main` 上にある `CHANGELOG.md` の `## YYYY.M.PATCH` セクションを、タグ付けされたリリースブランチと正確に一致させます。Mac リリースで公開された場合は、安定版 `appcast.xml` の更新を含めます。
4. オペレーターがそのリリース系列を明示的に開始するまで、`main` に `YYYY.M.PATCH+1`、ベータバージョン、または空の将来の変更履歴セクションを追加してはいけません。
5. `pnpm release:generated:check`、`pnpm deps:shrinkwrap:check`、`OPENCLAW_TESTBOX=1 pnpm check:changed` を実行します。プッシュしてから、安定版リリースの完了を宣言する前に、`origin/main` に出荷済みバージョンと変更履歴が含まれていることを確認します。
6. 非公開ロールバック訓練のたびに、リポジトリ変数 `RELEASE_ROLLBACK_DRILL_ID` と `RELEASE_ROLLBACK_DRILL_DATE` を最新に維持します。

`OpenClaw Stable Main Closeout` は、安定版公開後に出荷済みバージョン、変更履歴、appcast を含む `main` のプッシュから開始されます。変更不能な公開後証拠を読み取り、出荷済みタグをその Full Release Validation および Publish 実行に関連付けてから、安定版 main の状態、リリース、必須の安定版ソーク、ブロッキング対象のパフォーマンス証拠を検証します。変更不能なクローズアウトマニフェストとチェックサムを GitHub リリースに添付します。自動プッシュトリガーは、変更不能な公開後証拠より前の従来のリリースをスキップし、そのスキップを完了済みクローズアウトとして扱うことはありません。

完全なクローズアウトには、両方のアセットと一致するチェックサムが必要です。部分的なマニフェストは、記録済みの `main` SHA とロールバック訓練を再実行して同一のバイト列を再生成し、不足しているチェックサムを添付します。無効な組み合わせ、またはマニフェストのないチェックサムは、引き続きブロッキング対象です。ロールバック訓練のリポジトリ変数がないプッシュトリガー実行は、クローズアウトを完了せずにスキップされます。訓練記録がない、または 90 日を超えて古い場合も、証拠に基づく手動クローズアウトは引き続きブロックされます。非公開の復旧コマンドは、メンテナー専用ランブックに残します。手動ディスパッチは、証拠に基づく安定版クローズアウトの修復または再実行にのみ使用します。

Release Publish の親処理が、変更不能な npm／Plugin 証拠の添付後にのみ失敗した場合は、まずすべての安定版プラットフォームアセットを修復して公開します。その後、メンテナーは `allow_failed_publish_recovery=true` を指定してクローズアウトを手動ディスパッチできます。このモードでは、完了済みで失敗状態の親処理のみを受け入れます。さらに、通常の macOS／appcast 検証に加えて、正確な Android および Windows アセット契約、GitHub SHA-256 ダイジェスト、チェックサム検証、Android の出所、および親処理からディスパッチされ正常に完了した Windows 昇格が必要です。その Authenticode 検証と候補承認済みダイジェストは、公開済みインストーラーと一致している必要があります。自動プッシュによるクローズアウトでは、この復旧モードは決して有効になりません。

従来のフォールバック修正タグがベースパッケージの証拠を再利用できるのは、修正タグがベース安定版タグと同じソースコミットに解決される場合のみです。その Android リリースは、ベースタグの検証済み APK を再利用し、修正タグの出所を追加します。異なるソースを使用する修正では、独自のパッケージ証拠を公開して検証し、より大きい Android `versionCode` を使用する必要があります。

## リリースプリフライト

- リリースプリフライトの前に `pnpm check:test-types` を実行し、より高速なローカル `pnpm check` ゲートの外でもテスト用 TypeScript が対象となるようにします。
- リリースプリフライトの前に `pnpm check:architecture` を実行し、より高速なローカルゲートの外でも、より広範なインポートサイクルとアーキテクチャ境界のチェックが成功するようにします。
- `pnpm release:check` の前に `pnpm build && pnpm ui:build` を実行し、想定される `dist/*` リリースアーティファクトと Control UI バンドルがパック検証手順用に存在するようにします。
- ルートバージョンの更新後、タグ付け前に `pnpm release:prep` を実行します。これは、バージョン／設定／API の変更後に差異が発生しやすい決定論的なリリースジェネレーターをすべて実行します。対象は、Plugin バージョン、npm shrinkwrap、Plugin インベントリ、基本設定スキーマ、同梱チャンネル設定メタデータ、設定ドキュメントのベースライン、Plugin SDK エクスポート、Plugin SDK API 契約マニフェスト、Control UI ロケールバンドルです。また、ネイティブアプリの翻訳とプラットフォーム生成のロケールリソースがソースインベントリと一致するまで処理をブロックします。遅延している場合は、Code SHA を確定する前に `Native App Locale Refresh` を待機するかディスパッチします。`pnpm release:check` は、それらのガードをチェックモードで再実行し、パッケージのリリースチェックを実行する前に、厳格なロケールゲートと Plugin SDK サーフェス予算を含む生成物の差異によるすべての失敗を 1 回の実行で報告します。
- Plugin バージョン同期は、デフォルトで、公開可能な `@openclaw/ai` ランタイムパッケージ、公式 Plugin パッケージのバージョン、既存の `openclaw.compat.pluginApi` 下限を OpenClaw リリースバージョンに更新します。このフィールドは単なるパッケージバージョンのコピーではなく、Plugin SDK／ランタイム API の下限として扱います。古い OpenClaw ホストとの互換性を意図的に維持する Plugin 専用リリースでは、サポート対象の最も古いホスト API に下限を維持し、その選択を Plugin リリース証明に記載します。
- リリース承認前に、手動の `Full Release Validation` ワークフローを実行し、1 つのエントリポイントからすべてのプレリリーステストボックスを開始します。これはブランチ、タグ、または完全なコミット SHA を受け取り、手動の `CI` をディスパッチし、インストールスモーク、パッケージ受け入れ検証、クロス OS パッケージチェック、QA Lab の同等性、Matrix、Telegram の各レーン用に `OpenClaw Release Checks` をディスパッチします。安定版と完全実行では、包括的なライブ／E2E と Docker リリースパスのソークが常に含まれます。`run_release_soak=true` は、明示的なベータ版ソーク用に維持されています。Package Acceptance は候補検証中に正規のパッケージ Telegram E2E を提供し、同時に 2 つ目のライブポーラーが動作することを回避します。

  ベータ版の公開後に `release_package_spec` を指定すると、リリース tarball を再ビルドせずに、出荷済み npm パッケージをリリースチェック、Package Acceptance、パッケージ Telegram E2E で再利用できます。Telegram だけで、リリース検証の残りとは異なる公開済みパッケージを使用する必要がある場合にのみ、`npm_telegram_package_spec` を指定します。Package Acceptance でリリースパッケージ指定とは異なる公開済みパッケージを使用する必要がある場合は、`package_acceptance_package_spec` を指定します。Telegram E2E を強制せずに、検証が公開済み npm パッケージと一致することをリリース証拠レポートで証明する場合は、`evidence_package_spec` を指定します。

  ```bash
  node scripts/full-release-validation-at-sha.mjs \
    --sha <code-sha> \
    --target-ref release/YYYY.M.PATCH
  ```

- リリース作業を継続しながらパッケージ候補のサイドチャネル証明が必要な場合は、手動の `Package Acceptance` ワークフローを実行します。`openclaw@beta`、`openclaw@latest`、または正確なリリースバージョンには `source=npm` を使用します。現在の `workflow_ref` ハーネスで信頼済みの `package_ref` ブランチ、タグ、または SHA をパックするには `source=ref` を使用します。必須の SHA-256 と厳格な公開 URL ポリシーを備えた公開 HTTPS tarball には `source=url` を使用します。必須の `trusted_source_id` と SHA-256 を使用する名前付きの信頼済みソースポリシーには `source=trusted-url` を使用します。または、別の GitHub Actions 実行によってアップロードされた tarball には `source=artifact` を使用します。

  このワークフローは候補を `package-under-test` に解決し、その tarball に対して Docker E2E リリーススケジューラを再利用します。また、`telegram_mode=mock-openai` または `telegram_mode=live-frontier` を使用して、同じ tarball に対する Telegram QA を実行できます。選択した Docker レーンに `published-upgrade-survivor` が含まれる場合、パッケージアーティファクトが候補となり、`published_upgrade_survivor_baseline` が公開済みベースラインを選択します。`update-restart-auth` は候補パッケージをインストール済み CLI とテスト対象パッケージの両方として使用するため、候補の更新コマンドにおける管理対象の再起動パスを検証します。

  例:

  ```bash
  gh workflow run package-acceptance.yml --ref main -f workflow_ref=main -f source=npm -f package_spec=openclaw@beta -f suite_profile=product -f published_upgrade_survivor_baseline=openclaw@2026.4.26 -f telegram_mode=mock-openai
  ```

  一般的なプロファイル:
  - `smoke`: インストール、チャネル、エージェント、Gateway ネットワーク、および設定再読み込みのレーン
  - `package`: OpenWebUI またはライブ ClawHub を使用しない、アーティファクトネイティブのパッケージ、更新、再起動、および Plugin のレーン
  - `product`: パッケージプロファイルに加え、MCP チャネル、cron/サブエージェントのクリーンアップ、OpenAI ウェブ検索、および OpenWebUI
  - `full`: OpenWebUI を使用する Docker リリースパスのチャンク
  - `custom`: 集中的な再実行のための正確な `docker_lanes` 選択

- リリース候補に対して決定論的な通常の CI カバレッジのみが必要な場合は、手動の `CI` ワークフローを直接実行します。手動 CI ディスパッチは変更スコープをバイパスし、Linux Node シャード、バンドル済み Plugin シャード、Plugin およびチャネル契約シャード、Node 22 互換性、`check-*`、`check-additional-*`、ビルド済みアーティファクトのスモークチェック、ドキュメントチェック、Python Skills、Windows、macOS、および Control UI i18n レーンを強制実行します。単独の手動 CI 実行では、`include_android=true` を指定してディスパッチした場合にのみ Android を実行します。`Full Release Validation` はその入力を子 CI に渡します。

  ```bash
  gh workflow run ci.yml --ref release/YYYY.M.PATCH -f include_android=true
  ```

- リリーステレメトリを検証する場合は、`pnpm qa:otel:smoke` を実行します。これはローカル OTLP/HTTP レシーバーを介して QA-lab を動作させ、Opik、Langfuse、その他の外部コレクターを必要とせずに、トレース、メトリクス、ログのエクスポートに加え、制限されたトレース属性とコンテンツ/識別子の秘匿化を検証します。
- コレクターの互換性を検証する場合は、`pnpm qa:otel:collector-smoke` を実行します。これは、ローカルレシーバーのアサーションの前に、同じ QA-lab OTLP エクスポートを実際の OpenTelemetry Collector Docker コンテナ経由でルーティングします。
- 保護された Prometheus スクレイピングを検証する場合は、`pnpm qa:prometheus:smoke` を実行します。これは QA-lab を動作させ、未認証のスクレイピングを拒否し、リリースに不可欠なメトリクスファミリーにプロンプト内容、生の識別子、認証トークン、およびローカルパスが含まれないことを検証します。
- ソースチェックアウトの OpenTelemetry および Prometheus スモークレーンを連続して実行するには、`pnpm qa:observability:smoke` を実行します。
- タグ付きリリースのたびに、その前に `pnpm release:check` を実行します。
- `OpenClaw NPM Release` プリフライトは、npm tarball をパックする前に依存関係のリリース証拠を生成します。npm アドバイザリの脆弱性ゲートはリリースをブロックします。推移的マニフェストのリスク、依存関係の所有権/インストール面、および依存関係変更レポートは、リリース証拠としてのみ使用されます。依存関係変更レポートは、リリース候補を以前の到達可能なリリースタグと比較します。プリフライトは依存関係の証拠を `openclaw-release-dependency-evidence-<tag>` としてアップロードし、準備済み npm プリフライトアーティファクト内の `dependency-evidence/` にも埋め込みます。実際の公開パスはそのプリフライトアーティファクトを再利用し、同じ証拠を `openclaw-<version>-dependency-evidence.zip` として GitHub リリースに添付します。
- タグが存在した後、状態を変更する公開シーケンスには `OpenClaw Release Publish` を実行します。通常のベータ版および安定版の公開は、信頼済みの `main` からディスパッチします。リリースタグは引き続き正確なターゲットコミットを選択し、`release/YYYY.M.PATCH` 内を指すこともできます。Tideclaw アルファ版の公開は、対応するアルファブランチ上に留めます。成功した OpenClaw npm `preflight_run_id`、成功した `full_release_validation_run_id`、および正確な `full_release_validation_run_attempt` を渡し、集中的な修復を意図的に実行する場合を除き、デフォルトの Plugin 公開スコープ `all-publishable` を維持します。このワークフローは Plugin の npm 公開、Plugin の ClawHub 公開、および OpenClaw の npm 公開を直列化し、外部化された Plugin より先にコアパッケージが公開されないようにします。Windows および Android のプロモーションは、ドラフトリリースページに対するコア npm 公開と並行して実行されます。公開の再実行は再開可能です。すでに公開済みのコア npm バージョンについては、レジストリの tarball がタグのプリフライトアーティファクトと一致することをワークフローが証明した後、コアのディスパッチをスキップします。また、リリースに検証済みのアセット契約がすでに含まれている場合、Windows/Android のプロモーションをスキップするため、再試行では失敗したステージのみを再実行します。Plugin のみに絞った修復には、`plugin_publish_scope=selected` と空でない Plugin リストが必要です。Plugin のみの `all-publishable` 実行には、完全かつ不変のプリフライト証拠と Full Release Validation 証拠が必要です。不完全な証拠は拒否されます。
- 安定版の `OpenClaw Release Publish` には、対応するプレリリースではない `openclaw/openclaw-windows-node` リリースが存在した後の正確な `windows_node_tag` と、候補で承認された `windows_node_installer_digests` マップが必要です。公開子ワークフローをディスパッチする前に、そのソースリリースが公開済みかつプレリリースではなく、必要な x64/ARM64 インストーラーを含み、承認済みマップと引き続き一致していることを検証します。その後、OpenClaw リリースがまだドラフトである間に、固定されたインストーラーダイジェストマップを変更せずに引き継いで `Windows Node Release` をディスパッチします。子ワークフローは、その正確なタグから署名済み Windows Hub インストーラーをダウンロードし、固定されたダイジェストと照合し、Windows ランナー上で Authenticode 署名が期待される OpenClaw Foundation 署名者を使用していることを検証し、SHA-256 マニフェストを書き込み、インストーラーとマニフェストを正規の OpenClaw GitHub リリースにアップロードします。その後、プロモートされたアセットを再ダウンロードし、マニフェストへの登録とハッシュを検証します。親ワークフローは公開前に、現在の x64、ARM64、およびチェックサムのアセット契約を検証します。直接リカバリーでは、想定される契約アセットを固定されたソースバイトで置換する前に、予期しない `OpenClawCompanion-*` アセット名を拒否します。

  `Windows Node Release` はリカバリーの場合にのみ手動でディスパッチし、`latest` ではなく必ず正確なタグと、承認済みソースリリースの明示的な `expected_installer_digests` JSON マップを渡します。ウェブサイトのダウンロードリンクは、現在の安定版リリースの正確な OpenClaw リリースアセット URL を指すようにします。または、GitHub の latest リダイレクトが同じリリースを指していることを確認した後に限り、`releases/latest/download/...` を使用します。コンパニオンリポジトリのリリースページだけにリンクしないでください。

- リリースチェックは、独立した手動ワークフロー `OpenClaw Release Checks` で実行されるようになりました。また、リリース承認前に QA Lab モックパリティレーン、Matrix リリースプロファイル、Telegram QA レーンも実行されます。ライブレーンは `qa-live-shared` 環境を使用し、Telegram は Convex CI 認証情報リースも使用します。メンテナンス対象のすべての Matrix シナリオを実行する場合は、`matrix_profile=all` を指定して手動の `QA-Lab - All Lanes` ワークフローを実行してください。このワークフローは、ジョブごとのタイムアウト内で完全な証明を維持するため、その選択内容をトランスポート、メディア、E2EE の各プロファイルに分散します。
- クロス OS のインストールおよびアップグレードのランタイム検証は、公開 `OpenClaw Release Checks` と `Full Release Validation` の一部であり、これらは再利用可能なワークフロー `.github/workflows/openclaw-cross-os-release-checks-reusable.yml` を直接呼び出します。この分離は意図的なものです。実際の npm リリースパスを短く、決定論的で、アーティファクトに集中した状態に保つ一方、時間のかかるライブチェックは独自のレーンに置き、公開処理を停滞させたりブロックしたりしないようにします。
- シークレットを伴うリリースチェックは、ワークフローロジックとシークレットが管理された状態を維持できるよう、`Full Release Validation` または `main`/release ワークフロー ref からディスパッチする必要があります。
- `OpenClaw Release Checks` は、解決されたコミットが OpenClaw のブランチまたはリリースタグから到達可能である限り、ブランチ、タグ、完全なコミット SHA を受け入れます。
- `OpenClaw NPM Release` の検証専用プリフライトは、プッシュ済みタグを必要とせず、現在の完全な 40 文字のワークフローブランチコミット SHA も受け入れます。この SHA パスは検証専用であり、実際の公開に昇格させることはできません。SHA モードでは、ワークフローはパッケージメタデータチェックのためだけに `v<package.json version>` を合成します。実際の公開には、引き続き実在するリリースタグが必要です。
- どちらのワークフローも、実際の公開および昇格パスには GitHub ホストランナーを使用し続けますが、変更を伴わない検証パスでは、より大規模な Blacksmith Linux ランナーを使用できます。
- このワークフローは、ワークフローシークレットの `OPENAI_API_KEY` と `ANTHROPIC_API_KEY` の両方を使用して `OPENCLAW_LIVE_TEST=1 OPENCLAW_LIVE_CACHE_TEST=1 pnpm test:live:cache` を実行します。
- npm リリースプリフライトは、独立したリリースチェックレーンを待機しなくなりました。
- ローカルでリリース候補にタグを付ける前に、`RELEASE_TAG=vYYYY.M.PATCH-beta.N pnpm release:fast-pretag-check` を実行してください。このヘルパーは、高速なリリースガードレール、Plugin の npm/ClawHub リリースチェック、ビルド、UI ビルド、`release:openclaw:npm:check` を、GitHub 公開ワークフローの開始前に承認を妨げる一般的なミスを検出できる順序で実行します。
- 承認前に `RELEASE_TAG=vYYYY.M.PATCH node --import tsx scripts/openclaw-npm-release-check.ts`（または対応するプレリリース／修正版タグ）を実行してください。
- npm 公開後、`node --import tsx scripts/openclaw-npm-postpublish-verify.ts YYYY.M.PATCH`（または対応するベータ／修正版）を実行し、新しい一時プレフィックスで公開済みレジストリからのインストールパスを検証してください。
- ベータ公開後、`OPENCLAW_NPM_TELEGRAM_PACKAGE_SPEC=openclaw@YYYY.M.PATCH-beta.N OPENCLAW_NPM_TELEGRAM_CREDENTIAL_SOURCE=convex OPENCLAW_NPM_TELEGRAM_CREDENTIAL_ROLE=ci pnpm test:docker:npm-telegram-live` を実行し、共有のリース済み Telegram 認証情報プールを使用して、公開済み npm パッケージに対するインストール済みパッケージのオンボーディング、Telegram セットアップ、実際の Telegram E2E を検証してください。メンテナーがローカルで単発実行する場合は、Convex 変数を省略し、3 つの `OPENCLAW_QA_TELEGRAM_*` 環境認証情報を直接渡すことができます。
- メンテナーのマシンから公開後の完全なベータスモークを実行するには、`pnpm release:beta-smoke -- --beta betaN` を使用します。このヘルパーは、Parallels の npm 更新／新規ターゲット検証を実行し、`NPM Telegram Beta E2E` をディスパッチし、該当するワークフロー実行をポーリングし、アーティファクトをダウンロードして、Telegram レポートを出力します。
- メンテナーは、手動の `NPM Telegram Beta E2E` ワークフローを使用して、GitHub Actions から同じ公開後チェックを実行できます。これは意図的に手動専用であり、マージのたびには実行されません。
- メンテナー向けリリース自動化では、プリフライト後に昇格する方式を使用します。
  - 実際の npm 公開には、npm `preflight_run_id` の成功が必要です。
  - 通常のベータおよび安定版の公開オーケストレーションとプリフライトでは、対象タグそのものに対して信頼済みの `main` を使用します。Tideclaw アルファの公開とプリフライトでは、対応するアルファブランチを使用します。
  - 安定版 npm リリースでは、デフォルトで `beta` を使用します。安定版 npm 公開では、ワークフロー入力を介して `latest` を明示的に指定できます。
  - トークンベースの npm dist-tag 変更は `openclaw/releases/.github/workflows/openclaw-npm-dist-tags.yml` に配置されています。これは、ソースリポジトリが OIDC 専用公開を維持する一方で、`npm dist-tag add` には引き続き `NPM_TOKEN` が必要なためです。
  - 公開 `macOS Release` は検証専用です。タグがリリースブランチにのみ存在し、ワークフローが `main` からディスパッチされる場合は、`public_release_branch=release/YYYY.M.PATCH` を設定してください。
  - 実際の macOS 公開には、macOS `preflight_run_id` と `validate_run_id` の成功が必要です。
  - 実際の公開パスでは、再度ビルドするのではなく、準備済みのアーティファクトを昇格させます。
- `YYYY.M.PATCH-N` のような安定版修正リリースでは、公開後検証ツールは `YYYY.M.PATCH` から `YYYY.M.PATCH-N` への同じ一時プレフィックスによるアップグレードパスも確認します。これにより、リリース修正によって古いグローバルインストールが基本安定版のペイロードのまま暗黙に取り残されることを防ぎます。
- tarball に `dist/control-ui/index.html` と空でない `dist/control-ui/assets/` ペイロードの両方が含まれていない場合、npm リリースプリフライトは安全側に倒して失敗します。これにより、空のブラウザーダッシュボードを再び配布することを防ぎます。
- 公開後検証では、公開済み Plugin のエントリポイントとパッケージメタデータが、インストールされたレジストリレイアウト内に存在することも確認します。Plugin のランタイムペイロードが欠落したリリースは公開後検証に失敗し、`latest` に昇格できません。
- `pnpm test:install:smoke` は、候補更新 tarball に対する npm pack の `unpackedSize` 予算も適用します。これにより、インストーラー E2E は、リリース公開パスに入る前に意図しないパッケージ肥大化を検出できます。
- リリース作業で CI 計画、拡張機能のタイミングマニフェスト、または拡張機能のテストマトリクスを変更した場合は、承認前に `.github/workflows/plugin-prerelease.yml` からプランナーが所有する `plugin-prerelease-extension-shard` マトリクス出力を再生成してレビューし、リリースノートに古い CI レイアウトが記載されないようにしてください。
- 安定版 macOS リリースの準備状況には、アップデーターの各サーフェスも含まれます。GitHub リリースには最終的にパッケージ化された `.zip`、`.dmg`、`.dSYM.zip` が必要です。公開後、`main` 上の `appcast.xml` は新しい安定版 zip を指している必要があります（macOS 公開ワークフローが自動的にコミットするか、直接プッシュがブロックされている場合は appcast PR を作成します）。また、パッケージ化されたアプリは、デバッグ用ではないバンドル ID、空でない Sparkle フィード URL、そのリリースバージョンに対応する正規の Sparkle ビルド下限以上の `CFBundleVersion` を維持する必要があります。

## リリーステストボックス

`Full Release Validation` は、オペレーターが 1 つのエントリポイントから製品マトリクス全体を開始するための手段です。このヘルパーを使用すると、要求されたコミットをテスト対象候補として維持しながら、すべての子ワークフローが 1 つの信頼済み `main` ワークフロー SHA に固定された一時ブランチから実行されます。

```bash
pnpm ci:full-release \
  --sha <code-sha> \
  --target-ref release/YYYY.M.PATCH
```

このヘルパーは、現在の `origin/main` をフェッチし、その信頼済みワークフローコミットに `release-ci/<workflow-sha>-...` をプッシュし、アルファ／ベータのパッケージバージョンでは `beta` を、それ以外では `stable` を推論し、`ref=<target-sha>` を指定して一時ブランチから `Full Release Validation` をディスパッチし、すべての子ワークフローの `headSha` が固定された親ワークフロー SHA と一致することを検証してから、一時ブランチを削除します。新規実行を強制するには `-f reuse_evidence=false`、広範なアドバイザリスイープには `-f release_profile=full`、現在の `origin/main` から引き続き到達可能な古いコミットを固定するには `--workflow-sha <trusted-main-sha>` を渡します。ワークフロー自体がリポジトリの ref を書き換えることはありません。これにより、候補にツール用コミットを追加せずに main 専用のリリースツールを利用でき、誤ってより新しい `main` の子実行を証明してしまうことも防げます。

Code SHA がグリーンになった後、`CHANGELOG.md` のみをコミットし、Release SHA を指定して同じヘルパーを実行します。

```bash
pnpm ci:full-release \
  --sha <release-sha> \
  --target-ref release/YYYY.M.PATCH
```

2 回目の親実行が製品エビデンスを再利用するのは、Release SHA が Code SHA の子孫であり、変更されたパスの完全な集合が厳密に `CHANGELOG.md` であることを GitHub が証明した場合のみです。この実行は `changelog-only-release-v1` を記録し、製品の子ワークフローをディスパッチしません。tarball のバイト列が変更されているため、npm プリフライトとパッケージ／インストール受け入れテストは Release SHA に対して引き続き実行されます。

新しい Code SHA の場合、ワークフローはターゲットを解決し、手動の `CI` をディスパッチしてから、`OpenClaw Release Checks` をディスパッチします。`OpenClaw Release Checks` は、インストールスモーク、クロス OS リリースチェック、soak が有効な場合のライブ／E2E Docker リリースパスカバレッジ、正規の Telegram パッケージ E2E を含む Package Acceptance、QA Lab パリティ、ライブ Matrix、ライブ Telegram に分散して実行します。完全／全対象の実行が許容されるのは、個別の `Plugin Prerelease` 子ワークフローを意図的にスキップした限定的な再実行を除き、`Full Release Validation` のサマリーで `normal_ci`、`plugin_prerelease`、`release_checks` が成功と表示されている場合のみです。公開済みパッケージに対する限定的な再実行では、`release_package_spec` または `npm_telegram_package_spec` を指定したスタンドアロンの `npm-telegram` 子ワークフローのみを使用してください。最終検証サマリーには各子実行の最遅ジョブ表が含まれるため、リリースマネージャーはログをダウンロードせずに現在のクリティカルパスを確認できます。

このリリースパスでは、製品パフォーマンスの子ワークフローはアーティファクト専用です。
アンブレラワークフローは `publish_reports=false` を指定してこれをディスパッチし、アーティファクト専用ガードによって Clawgrit レポート公開処理がスキップされたままであることが証明されない限り、検証は拒否されます。

ステージマトリクス全体、正確なワークフロージョブ名、安定版と完全版のプロファイル差異、アーティファクト、限定的な再実行用ハンドルについては、[完全なリリース検証](/ja-JP/reference/full-release-validation)を参照してください。

子ワークフローは、`Full Release Validation` を実行する SHA 固定済みの信頼済み ref からディスパッチされます。すべての子実行は、親ワークフローと完全に同じ SHA を使用する必要があります。リリース証明に生の `--ref main -f ref=<sha>` ディスパッチを使用せず、`pnpm ci:full-release --sha <target-sha> --target-ref release/YYYY.M.PATCH` を使用してください。

ライブ／プロバイダーの範囲を選択するには、`release_profile` を使用します。

- `beta`: 最速のリリース必須 OpenAI／コアライブおよび Docker パス
- `stable`: リリース承認向けのベータおよび安定版プロバイダー／バックエンドカバレッジ
- `full`: 安定版に加えた広範なアドバイザリプロバイダー／メディアカバレッジ

安定版および完全版の検証では、昇格前に必ず網羅的なライブ／E2E、Docker リリースパス、範囲を限定した公開済みアップグレード生存確認スイープを実行します。ベータに対して同じスイープを要求するには、`run_release_soak=true` を使用します。このスイープは、最新の 4 つの安定版パッケージに加え、固定された `2026.4.23` および `2026.5.2` のベースライン、さらに古い `2026.4.15` のカバレッジを対象とします。重複するベースラインは削除され、各ベースラインは個別の Docker ランナージョブにシャーディングされます。

`OpenClaw Release Checks` は、信頼済みワークフロー ref を使用してターゲット ref を一度だけ `release-package-under-test` として解決し、soak 実行時のクロス OS、Package Acceptance、リリースパス Docker チェックでそのアーティファクトを再利用します。これにより、パッケージ関連のすべてのボックスが同じバイト列を使用し、パッケージの繰り返しビルドを回避できます。ベータがすでに npm に公開されている場合は、`release_package_spec=openclaw@YYYY.M.PATCH-beta.N` を設定します。これにより、リリースチェックは公開済みパッケージを一度だけダウンロードし、`dist/build-info.json` からビルド元 SHA を抽出して、そのアーティファクトをクロス OS、Package Acceptance、リリースパス Docker、パッケージ Telegram の各レーンで再利用します。

クロス OS の OpenAI インストールスモークでは、リポジトリ／組織変数が設定されている場合は `OPENCLAW_CROSS_OS_OPENAI_MODEL` を、設定されていない場合は `openai/gpt-5.6-luna` を使用します。このレーンの目的は、最も高性能なモデルのベンチマークではなく、パッケージのインストール、オンボーディング、Gateway の起動、1 回のライブエージェントターンを証明することだからです。より広範なライブプロバイダーマトリクスが、引き続きモデル固有のカバレッジを担います。

リリース段階に応じて、次のバリエーションを使用します。

```bash
# 製品完成時点の Code SHA を検証します。
pnpm ci:full-release \
  --sha <code-sha> \
  --target-ref release/YYYY.M.PATCH

# Code SHA の製品エビデンスを再利用して、変更履歴のみの Release SHA を検証します。
pnpm ci:full-release \
  --sha <release-sha> \
  --target-ref release/YYYY.M.PATCH

# ベータ版の公開後、公開済みパッケージの Telegram E2E を追加します。
pnpm ci:full-release \
  --sha <release-sha> \
  --target-ref release/YYYY.M.PATCH \
  -f release_package_spec=openclaw@YYYY.M.PATCH-beta.N \
  -f evidence_package_spec=openclaw@YYYY.M.PATCH-beta.N \
  -f npm_telegram_provider_mode=mock-openai
```

対象を絞った修正後の最初の再実行として、包括的な全体ワークフローを使用しないでください。1 つのボックスが失敗した場合、次の検証には失敗した子ワークフロー、ジョブ、Docker レーン、パッケージプロファイル、モデルプロバイダー、または QA レーンを使用します。修正によって共有リリースオーケストレーションが変更された場合、または以前の全ボックスのエビデンスが古くなった場合にのみ、包括的な全体ワークフローを再実行してください。全体ワークフローの最終検証処理は、記録された子ワークフローの実行 ID を再確認するため、子ワークフローの再実行が成功した後は、失敗した `Verify full validation` 親ジョブのみを再実行します。

`rerun_group=all` は、リリースプロファイル、実効的なソーク設定、検証入力が一致し、かつ対象 SHA が同一であるか、新しい対象がその子孫であり、変更されたパスの完全な集合が正確に `CHANGELOG.md` である場合、以前に成功した全体ワークフローの実行を再利用できます。対象が完全に一致する場合の再利用では `exact-target-full-validation-v1` が記録され、検証後の Release SHA では `changelog-only-release-v1` が記録されます。後者で再利用されるのは製品検証のみです。Npm の事前チェック、パッケージのバイト列、リリースノートの来歴、インストール／更新の受け入れ検証は、引き続き Release SHA に対して実行する必要があります。バージョン、ソース、生成物、依存関係、パッケージ、またはワークフローが所有する対象に変更がある場合は、新しい Code SHA と新規の完全検証が必要です。同じ `release/*` ref と再実行グループに対する新しい全体ワークフローの実行は、進行中の実行を自動的に置き換えます。新規の完全実行を強制するには `reuse_evidence=false` を渡します。

範囲を限定した復旧では、全体ワークフローに `rerun_group` を渡します。`all` は実際のリリース候補実行、`ci` は通常の CI 子ワークフローのみ、`plugin-prerelease` はリリース専用 Plugin 子ワークフローのみ、`release-checks` はすべてのリリースボックスを実行します。より範囲の狭いリリースグループは `install-smoke`、`cross-os`、`live-e2e`、`package`、`qa`、`qa-parity`、`qa-live`、および `npm-telegram` です。対象を絞った `npm-telegram` の再実行には `release_package_spec` または `npm_telegram_package_spec` が必要です。完全実行／全実行では、Package Acceptance 内の標準パッケージ Telegram E2E を使用します。対象を絞ったクロス OS 再実行には、`cross_os_suite_filter=windows/packaged-upgrade` または別の OS／スイートフィルターを追加できます。QA リリースチェックの失敗は、コアのランタイムペアレーンにおける OpenClaw 動的ツールのドリフトを含め、通常のリリース検証をブロックします。Tideclaw アルファ実行では、パッケージ安全性に関係しないリリースチェックレーンを引き続き参考情報として扱うことができます。`release_profile=beta` では、`Run repo/live E2E validation` ライブプロバイダースイートは参考情報扱いです（警告のみで、ブロックしません）。安定版プロファイルと完全プロファイルでは、引き続きブロック対象です。`live_suite_filter` が Discord、WhatsApp、Slack などのゲート付き QA ライブレーンを明示的に要求する場合、対応する `OPENCLAW_RELEASE_QA_*_LIVE_CI_ENABLED` リポジトリ変数を有効にする必要があります。有効でない場合、レーンを暗黙的にスキップするのではなく、入力の取得が失敗します。

### Vitest

Vitest ボックスは、手動の `CI` 子ワークフローです。手動 CI は意図的に変更範囲による絞り込みを回避し、リリース候補に対して通常のテストグラフを強制します。対象には Linux Node シャード、バンドル済み Plugin シャード、Plugin およびチャネル契約シャード、Node 22 互換性、`check-*`、`check-additional-*`、ビルド済み成果物のスモークチェック、ドキュメントチェック、Python Skills、Windows、macOS、Control UI の i18n が含まれます。`Full Release Validation` がこのボックスを実行する場合は、全体ワークフローが `include_android=true` を渡すため Android も含まれます。単独の手動 CI で Android を対象にするには、`include_android=true` が必要です。

このボックスは「ソースツリーが通常の完全テストスイートに合格したか」という問いに答えるために使用します。リリース経路の製品検証とは異なります。保持するエビデンスは次のとおりです。

- `Full Release Validation` のサマリー。ディスパッチされた `CI` 実行 URL を表示するもの
- 正確な対象 SHA に対して成功した `CI` の実行
- リグレッション調査時に CI ジョブから取得した、失敗した、または遅いシャード名
- 実行のパフォーマンス分析が必要な場合の、`.artifacts/vitest-shard-timings.json` などの Vitest タイミング成果物

リリースで決定論的な通常 CI が必要だが、Docker、QA Lab、ライブ、クロス OS、またはパッケージの各ボックスが不要な場合にのみ、手動 CI を直接実行します。Android を含まない直接 CI には最初のコマンドを使用します。直接実行するリリース候補 CI で Android も対象にする必要がある場合は、`include_android=true` を追加します。

```bash
gh workflow run ci.yml --ref main -f target_ref=release/YYYY.M.PATCH
gh workflow run ci.yml --ref main -f target_ref=release/YYYY.M.PATCH -f include_android=true
```

### Docker

Docker ボックスは、`OpenClaw Release Checks` から `openclaw-live-and-e2e-checks-reusable.yml`、およびリリースモードの `install-smoke` ワークフローにあります。ソースレベルのテストだけでなく、パッケージ化された Docker 環境を通じてリリース候補を検証します。

リリース向け Docker の対象範囲は次のとおりです。

- 低速な Bun グローバルインストールのスモークを有効にした完全インストールスモーク
- 対象 SHA ごとのルート Dockerfile スモークイメージの準備／再利用。QR、root/Gateway、installer/Bun の各スモークジョブを個別のインストールスモークシャードとして実行
- リポジトリの E2E レーン
- リリース経路の Docker チャンク：`core`、`package-update-openai`、`package-update-anthropic`、`package-update-core`、`plugins-runtime-plugins`、`plugins-runtime-services`、`plugins-runtime-install-a` から `plugins-runtime-install-h`、および `openwebui`
- 要求時に専用の大容量ディスクランナーで実行する OpenWebUI の検証
- `bundled-plugin-install-uninstall-0` から `bundled-plugin-install-uninstall-23` までの、分割されたバンドル済み Plugin のインストール／アンインストールレーン
- リリースチェックにライブスイートが含まれる場合の、ライブ／E2E プロバイダースイートおよび Docker ライブモデルの検証

再実行の前に Docker 成果物を使用してください。リリース経路のスケジューラーは、レーンログ、`summary.json`、`failures.json`、フェーズのタイミング、スケジューラー計画 JSON、再実行コマンドを含む `.artifacts/docker-tests/` をアップロードします。対象を絞った復旧では、すべてのリリースチャンクを再実行する代わりに、再利用可能なライブ／E2E ワークフローで `docker_lanes=<lane[,lane]>` を使用します。生成される再実行コマンドには、利用可能な場合、以前の `package_artifact_run_id` と準備済み Docker イメージの入力が含まれるため、失敗したレーンで同じ tarball と GHCR イメージを再利用できます。

### QA Lab

QA Lab ボックスも `OpenClaw Release Checks` の一部です。これはエージェント動作とチャネルレベルのリリースゲートであり、Vitest や Docker のパッケージ処理とは別です。

リリース向け QA Lab の対象範囲は次のとおりです。

- エージェント型パリティパックを使用して、OpenAI 候補レーンを `anthropic/claude-opus-4-8` ベースラインと比較するモックパリティレーン
- `qa-live-shared` 環境を使用する Matrix ライブアダプターのリリースプロファイル
- Convex CI の認証情報リースを使用するライブ Telegram QA レーン
- リリーステレメトリに明示的なローカル検証が必要な場合の `pnpm qa:otel:smoke`、`pnpm qa:otel:collector-smoke`、`pnpm qa:prometheus:smoke`、または `pnpm qa:observability:smoke`

このボックスは「リリースが QA シナリオとライブチャネルフローで正しく動作するか」という問いに答えるために使用します。リリースを承認する際は、パリティ、Matrix、Telegram の各レーンの成果物 URL を保持してください。Matrix の完全な検証は、デフォルトのリリースクリティカルなレーンではなく、手動のシャード化された QA Lab 実行として引き続き利用できます。

### パッケージ

Package ボックスは、インストール可能な製品のゲートです。`Package Acceptance` とリゾルバー `scripts/resolve-openclaw-package-candidate.mjs` によって支えられています。リゾルバーは候補を Docker E2E が使用する `package-under-test` tarball に正規化し、パッケージインベントリを検証し、パッケージのバージョンと SHA-256 を記録し、ワークフローハーネスの ref とパッケージソースの ref を分離して保持します。

サポートされる候補ソースは次のとおりです。

- `source=npm`：`openclaw@beta`、`openclaw@latest`、または OpenClaw の正確なリリースバージョン
- `source=ref`：選択した `workflow_ref` ハーネスを使用して、信頼済みの `package_ref` ブランチ、タグ、または完全なコミット SHA をパック
- `source=url`：必須の `package_sha256` を指定して公開 HTTPS `.tgz` をダウンロード。URL の認証情報、デフォルト以外の HTTPS ポート、プライベート／内部／特殊用途のホスト名または解決済みアドレス、安全でないリダイレクトは拒否される
- `source=trusted-url`：`.github/package-trusted-sources.json` 内の名前付きポリシーから、必須の `package_sha256` と `trusted_source_id` を指定して HTTPS `.tgz` をダウンロード。`source=url` に入力レベルのプライベートネットワーク回避策を追加する代わりに、メンテナー所有のエンタープライズミラーまたはプライベートパッケージリポジトリで使用する
- `source=artifact`：別の GitHub Actions 実行によってアップロードされた `.tgz` を再利用

`OpenClaw Release Checks` は、`source=artifact`、準備済みリリースパッケージ成果物、`suite_profile=custom`、`docker_lanes=doctor-switch update-channel-switch skill-install update-corrupt-plugin upgrade-survivor published-upgrade-survivor root-managed-vps-upgrade update-restart-auth plugins-offline plugin-update plugin-binding-command-escape`、`telegram_mode=mock-openai` を使用して Package Acceptance を実行します。Package Acceptance では、同じ解決済み tarball に対して、移行、更新、root 管理 VPS のアップグレード、設定済み認証の更新後再起動、ライブ ClawHub skill のインストール、古い Plugin 依存関係のクリーンアップ、オフライン Plugin フィクスチャ、Plugin の更新、Plugin コマンドバインディングのエスケープ強化、Telegram パッケージ QA を実行します。ブロック対象のリリースチェックでは、デフォルトで最新の公開済みパッケージをベースラインとして使用します。`run_release_soak=true`、`release_profile=stable`、または `release_profile=full` を指定したベータプロファイルでは、公開済みアップグレード存続確認を `last-stable-4` に加え、固定された `2026.4.23`、`2026.5.2`、`2026.4.15` の各ベースラインまで拡張し、`reported-issues` シナリオを使用します。すでに公開済みの候補には `source=npm`、公開前の SHA ベースのローカル npm tarball には `source=ref`、メンテナー所有のエンタープライズ／プライベートミラーには `source=trusted-url`、別の GitHub Actions 実行によってアップロードされた準備済み tarball には `source=artifact` を指定して Package Acceptance を使用します。

これは、以前 Parallels が必要だったパッケージ／更新検証の大部分を置き換える GitHub ネイティブの仕組みです。OS 固有のオンボーディング、インストーラー、プラットフォーム動作についてはクロス OS リリースチェックも引き続き重要ですが、パッケージ／更新の製品検証では Package Acceptance を優先してください。

更新および Plugin 検証の標準チェックリストは、[更新と Plugin のテスト](/ja-JP/help/testing-updates-plugins)です。Plugin のインストール／更新、doctor によるクリーンアップ、公開済みパッケージの移行変更を、ローカル、Docker、Package Acceptance、またはリリースチェックのどのレーンで検証するか判断する際に使用してください。すべての安定版 `2026.4.23+` パッケージからの網羅的な公開済み更新移行は、Full Release CI の一部ではなく、独立した手動 `Update Migration` ワークフローです。

従来の Package Acceptance の緩和措置には、意図的に期限が設定されています。`2026.4.25` までのパッケージでは、npm にすでに公開されたメタデータ欠落に対する互換性経路を使用できます。対象には、tarball に存在しないプライベート QA インベントリエントリ、欠落した `gateway install --wrapper`、tarball から生成された git フィクスチャ内の欠落したパッチファイル、永続化されていない `update.channel`、従来の Plugin インストール記録の場所、マーケットプレイスのインストール記録の永続化欠落、`plugins update` 中の設定メタデータ移行が含まれます。公開済みの `2026.4.26` パッケージでは、すでに出荷されたローカルビルドメタデータのスタンプファイルについて警告となる場合があります。それ以降のパッケージは、最新のパッケージ契約を満たす必要があります。同じ欠落があるとリリース検証は失敗します。

リリースに関する確認事項が実際にインストール可能なパッケージについてである場合は、より広範な Package Acceptance プロファイルを使用します。

```bash
gh workflow run package-acceptance.yml \
  --ref main \
  -f workflow_ref=main \
  -f source=npm \
  -f package_spec=openclaw@beta \
  -f suite_profile=product \
  -f published_upgrade_survivor_baseline=openclaw@2026.4.26
```

一般的なパッケージプロファイル：

- `smoke`: パッケージの迅速なインストール／チャネル／エージェント、Gateway ネットワーク、設定再読み込みの各レーン
- `package`: インストール／更新／再起動／Plugin パッケージの契約に加え、ClawHub からの Skills のライブインストール証明。これはリリースチェックのデフォルトです
- `product`: `package` に加え、MCP チャネル、Cron／サブエージェントのクリーンアップ、OpenAI ウェブ検索、OpenWebUI
- `full`: OpenWebUI を含む Docker リリースパスのチャンク
- `custom`: 対象を絞った再実行用の正確な `docker_lanes` リスト

パッケージ候補の Telegram 証明では、Package Acceptance で `telegram_mode=mock-openai` または `telegram_mode=live-frontier` を有効にします。このワークフローは、解決された `package-under-test` tarball を Telegram レーンに渡します。スタンドアロンの Telegram ワークフローでは、公開後チェック用に、公開済みの npm 仕様も引き続き受け付けます。

## 通常リリースの公開自動化

ベータ、`latest`、Plugin、GitHub Release、プラットフォームへの公開では、
`OpenClaw Release Publish` が通常の変更実行エントリポイントです。毎月の
`.33+` Gateway 延長安定版パスでは、このオーケストレーターを使用しません。
通常のワークフローは、リリースに必要な順序で信頼済みパブリッシャーの
ワークフローをオーケストレーションします。

1. リリースタグをチェックアウトし、そのコミット SHA を解決します。
2. タグが `main` または `release/*`（アルファプレリリースの場合は Tideclaw アルファブランチ）から到達可能であることを確認します。
3. `pnpm plugins:sync:check` を実行します。
4. `publish_scope=all-publishable` および `ref=<release-sha>` を指定して `Plugin NPM Release` をディスパッチします。
5. 同じスコープと SHA で `Plugin ClawHub Release` をディスパッチします。
6. 保存済みの `full_release_validation_run_id` と正確な実行試行回数を検証した後、リリースタグ、npm dist-tag、保存済みの `preflight_run_id` を指定して `OpenClaw NPM Release` をディスパッチします。
7. 安定版リリースでは、GitHub リリースをドラフトとして作成または更新し、明示的な `windows_node_tag` と候補承認済みの `windows_node_installer_digests` を指定して `Windows Node Release` をディスパッチし、正規の Windows インストーラー／チェックサムアセットを検証します。また、正確なタグの署名済み APK、チェックサム、プロベナンスをビルドするために `Android Release` もディスパッチします。ドラフトを公開する前に、両方のネイティブアセット契約を検証します。

ベータ公開の例：

```bash
gh workflow run openclaw-release-publish.yml \
  --ref main \
  -f tag=vYYYY.M.PATCH-beta.N \
  -f preflight_run_id=<successful-openclaw-npm-preflight-run-id> \
  -f full_release_validation_run_id=<successful-full-release-validation-run-id> \
  -f full_release_validation_run_attempt=<successful-full-release-validation-run-attempt> \
  -f npm_dist_tag=beta
```

デフォルトのベータ dist-tag への安定版公開：

```bash
gh workflow run openclaw-release-publish.yml \
  --ref main \
  -f tag=vYYYY.M.PATCH \
  -f windows_node_tag=vX.Y.Z \
  -f windows_node_installer_digests='{"OpenClawCompanion-Setup-x64.exe":"sha256:<approved-x64-sha256>","OpenClawCompanion-Setup-arm64.exe":"sha256:<approved-arm64-sha256>"}' \
  -f preflight_run_id=<successful-openclaw-npm-preflight-run-id> \
  -f full_release_validation_run_id=<successful-full-release-validation-run-id> \
  -f full_release_validation_run_attempt=<successful-full-release-validation-run-attempt> \
  -f npm_dist_tag=beta
```

`latest` への安定版の直接昇格は明示的に行います：

```bash
gh workflow run openclaw-release-publish.yml \
  --ref main \
  -f tag=vYYYY.M.PATCH \
  -f windows_node_tag=vX.Y.Z \
  -f windows_node_installer_digests='{"OpenClawCompanion-Setup-x64.exe":"sha256:<approved-x64-sha256>","OpenClawCompanion-Setup-arm64.exe":"sha256:<approved-arm64-sha256>"}' \
  -f preflight_run_id=<successful-openclaw-npm-preflight-run-id> \
  -f full_release_validation_run_id=<successful-full-release-validation-run-id> \
  -f full_release_validation_run_attempt=<successful-full-release-validation-run-attempt> \
  -f npm_dist_tag=latest
```

下位レベルの `Plugin NPM Release` および `Plugin ClawHub Release` ワークフローは、対象を絞った修復または再公開作業にのみ使用します。`publish_openclaw_npm=true` の場合、`OpenClaw Release Publish` は `plugin_publish_scope=selected` を拒否するため、`@openclaw/diffs-language-pack` を含む公開可能なすべての公式 Plugin なしでコアパッケージが出荷されることはありません。選択した Plugin の修復では、`plugin_publish_scope=selected` および `plugins=@openclaw/name` とともに `publish_openclaw_npm=false` を設定するか、子ワークフローを直接ディスパッチします。

初回公開時の ClawHub ブートストラップは例外です。信頼済みの `main`
から `Plugin ClawHub New` をディスパッチし、`ref` を通じて対象リリースの完全な SHA を渡します。
ブートストラップワークフロー自体をリリースタグまたはブランチから実行してはなりません。

```bash
gh workflow run plugin-clawhub-new.yml \
  --ref main \
  -f plugins=@openclaw/name \
  -f ref=<full-40-character-release-sha> \
  -f pretag_validation=true \
  -f dry_run=true
```

タグ付け前の検証には `dry_run=true` が必要であり、リリースタグおよび親実行の
入力を拒否し、`main` または `release/*` から到達可能な正確な対象のみを受け付けます。
ClawHub の認証情報を読み込んだり、パッケージバイトを公開したり、信頼済み
パブリッシャーの設定を変更したりすることはありません。ワークフローは引き続きライブレジストリ計画を解決し、
シークレットなしのジョブでのみ対象をチェックアウトしてパックし、ロックされた
ClawHub ツールチェーンを実体化し、リリースタグが存在する前に不変アーティファクトとパッケージの
スラッグ／ID を検証します。シークレットなしのパックジョブが
完了した後にのみ `clawhub-plugin-bootstrap` 環境を承認してください。この保護された検証ジョブには、認証情報も変更コマンドもありません。

タグ付け後に承認されたドライランまたは実際のブートストラップを実行するには、正確な
リリースタグに加え、親 `OpenClaw Release Publish` の実行 ID、試行回数、
ブランチを含める必要があります。親は自身のワークフロー SHA と、`Plugin ClawHub New` 用の別個の正確な信頼済み
`main` SHA を証明します。子実行および保護された
環境のすべての承認は、その承認済み子 SHA と一致する必要があります。リリースタグは、
公開の各試行および信頼済みパブリッシャーの変更前に再確認されます。

パックジョブは、
名前、Actions アーティファクト ID／ダイジェスト、
生成元の実行／試行、対象 SHA、パッケージごとの tarball SHA-256／サイズが
検証ジョブおよび保護ジョブに引き継がれる、単一の不変アーティファクトをアップロードします。保護ジョブは、信頼済みの `main`
ツールのみをチェックアウトし、GitHub API を通じてアーティファクトの組を検証し、正確なアーティファクト ID
でダウンロードし、すべての tarball を再ハッシュし、固定された CLI の USTAR 正規化ルールに従ってローカル TAR パスと
パッケージ ID を検証します。その後、すべての候補は固定された CLI の公開ドライランを通過します。このドライランは、
レジストリ検索または認証の前に戻ります。認証情報ジョブの事前フィルターは、圧縮済み ClawPack を
120 MiB、ファイルペイロード合計を 50 MiB、展開後の TAR データを 64 MiB、
TAR エントリ数を 10,000 に制限します。既存パッケージの信頼済みパブリッシャー修復は
引き続き設定のみですが、対象のパックも行い、信頼済みパブリッシャーの
設定を変更する前に、要求されたタグに加えて、レジストリの正確なバイトおよびメタデータの一致を必須とします。
公開後の検証では ClawHub アーティファクトをダウンロードし、
同じ SHA-256 とサイズを必須とします。失敗した実行の再実行による復旧では、正確な生成元ジョブが
正常に完了した場合に限り、以前の試行のパッケージアーティファクトを再利用できます。
最終証拠は、ロックされた ClawHub バージョン、ロックの
SHA-256、npm integrity にも紐付けられます。不一致の場合は、新しいパッケージバージョンが必要です。

## NPM ワークフローの入力

`OpenClaw NPM Release` は、オペレーターが制御する以下の入力を受け付けます。

- `tag`: `v2026.4.2`、`v2026.4.2-1`、`v2026.4.2-beta.1`、`v2026.4.2-alpha.1` などの必須リリースタグ。`preflight_only=true` の場合、検証専用の事前チェック用として、現在の完全な 40 文字のワークフローブランチコミット SHA も指定できます
- `preflight_only`: 検証／ビルド／パッケージ化のみの場合は `true`、実際の公開パスの場合は `false`
- `preflight_run_id`: 既存の正常終了した事前チェック実行 ID。実際の公開パスでは必須であり、ワークフローは tarball を再ビルドせず、準備済みのものを再利用します
- `full_release_validation_run_id`: このタグ／SHA に対する正常終了した `Full Release Validation` 実行 ID。実際の公開では必須です。ベータ公開は警告付きで事前チェックのみでも続行できますが、安定版／`latest` への昇格では引き続き必須です。
- `full_release_validation_run_attempt`: `full_release_validation_run_id` と組み合わせる正確な正の実行試行回数。実行 ID を指定する場合は必須であり、再実行によって公開中の承認証拠を変更できないようにします。
- `release_publish_run_id`: 承認済みの `OpenClaw Release Publish` 実行 ID。このワークフローがその親からディスパッチされる場合（ボットアクターによる実公開呼び出し）に必須です
- `plugin_npm_run_id`: 正常終了した正確な HEAD の `Plugin NPM Release` 実行 ID。`extended-stable` コアを実際に公開する場合に必須です
- `npm_dist_tag`: 公開パス用の npm 対象タグ。`alpha`、`beta`、`latest`、`extended-stable` を受け付け、デフォルトは `beta` です。最終パッチ `33` 以降では `extended-stable` を使用する必要があります。デフォルトでは、`extended-stable` はそれより前のパッチを拒否し、最終版ではないタグは常に拒否します。
- `bypass_extended_stable_guard`: テスト専用のブール値。デフォルトは `false` です。`npm_dist_tag=extended-stable` の場合、リリース ID、アーティファクト、承認、読み戻しの各チェックを維持しながら、毎月の延長安定版の適格性をバイパスします。

`Plugin NPM Release` は、既存のリリース動作には `npm_dist_tag=default`、
ガード付きの毎月のパスには `npm_dist_tag=extended-stable` を受け付けます。
延長安定版オプションには `publish_scope=all-publishable`、空の
`plugins` 入力、`33` 以上の最終パッチ、および正確な先端にある正規の
`extended-stable/YYYY.M.33` ブランチが必要です。Plugin の
`latest` または `beta` を移動することはありません。新しいパッケージバージョンには、OIDC による信頼済み公開（`npm publish --tag extended-stable`）を通じて
`extended-stable` がアトミックに付与されます。この
ソースワークフローは、トークン認証された `npm dist-tag add` を使用しません。再試行では、
npm にすでに存在する正確なバージョンをスキップし、その後、すべての正確なパッケージと `extended-stable` タグが収束したことを完全な
読み戻しで確認できない限り、安全側に失敗します。

`OpenClaw Release Publish` は、オペレーターが制御する以下の入力を受け付けます。

- `tag`: 必須のリリースタグ。すでに存在している必要があります
- `preflight_run_id`: 正常終了した `OpenClaw NPM Release` 事前チェック実行 ID。`publish_openclaw_npm=true` または `plugin_publish_scope=all-publishable` の場合に必須です
- `full_release_validation_run_id`: 正常終了した `Full Release Validation` 実行 ID。`publish_openclaw_npm=true` または `plugin_publish_scope=all-publishable` の場合に必須です
- `full_release_validation_run_attempt`: `full_release_validation_run_id` と組み合わせる正確な正の試行回数。実行 ID を指定する場合は必須です
- `windows_node_tag`: プレリリースではない正確な `openclaw/openclaw-windows-node` リリースタグ。OpenClaw 安定版の公開に必須です
- `windows_node_installer_digests`: 現在の Windows インストーラー名から固定された `sha256:` ダイジェストへの、候補承認済みのコンパクトな JSON マップ。OpenClaw 安定版の公開に必須です
- `npm_telegram_run_id`: 最終リリース証拠に含める、正常終了した任意の `NPM Telegram Beta E2E` 実行 ID
- `npm_dist_tag`: OpenClaw パッケージ用の npm 対象タグ。`alpha`、`beta`、`latest` のいずれか
- `plugin_publish_scope`: デフォルトは `all-publishable` です。`publish_openclaw_npm=false` を使用する、対象を絞った Plugin のみの修復作業に限り `selected` を使用します
- `plugins`: `plugin_publish_scope=selected` の場合に指定する、カンマ区切りの `@openclaw/*` パッケージ名
- `publish_openclaw_npm`: デフォルトは `true` です。ワークフローを Plugin のみの修復オーケストレーターとして使用する場合に限り、`false` を設定します
- `release_profile`: リリース証拠の要約に使用するリリースカバレッジプロファイル。デフォルトは `from-validation` で、検証マニフェストから読み取ります。または `beta`、`stable`、`full` で上書きします
- `wait_for_clawhub`: ClawHub サイドカーによって npm の可用性が妨げられないよう、デフォルトは `false` です。ワークフローの完了に ClawHub の完了も含める必要がある場合に限り、`true` を設定します

`OpenClaw Release Checks` は、オペレーターが制御する以下の入力を受け付けます。

- `ref`: 検証するブランチ、タグ、または完全なコミット SHA。シークレットを使用するチェックでは、解決されたコミットが OpenClaw のブランチまたはリリースタグから到達可能である必要があります。
- `run_release_soak`: ベータリリースチェックで、網羅的なライブ/E2E、Docker リリースパス、および全期間のアップグレード耐久ソークテストを有効にします。`release_profile=stable` および `release_profile=full` によって強制的に有効になります。

ルール:

- パッチ `33` 未満の通常の最終版および修正版は、`beta` または `latest` のいずれにも公開できます。パッチ `33` 以上の最終版は `extended-stable` に公開する必要があり、その境界で修正サフィックスが付いたバージョンは拒否されます。
- ベータのプレリリースタグは `beta` にのみ公開でき、アルファのプレリリースタグは `alpha` にのみ公開できます
- `OpenClaw NPM Release` では、`preflight_only=true` の場合にのみ完全なコミット SHA を入力できます
- `OpenClaw Release Checks` および `Full Release Validation` は常に検証専用です
- 実際の公開パスでは、プリフライト時に使用したものと同じ `npm_dist_tag` を使用する必要があります。公開を続行する前に、ワークフローがそのメタデータを検証します

## 通常のベータ/最新安定版リリース手順

この従来の手順は、plugins、GitHub Release、Windows、およびその他のプラットフォーム作業も担う、通常のオーケストレーションされたリリース向けです。このページの冒頭に記載されている毎月の `.33+` Gateway 延長安定版パスではありません。

通常のオーケストレーションされた安定版リリースを作成する場合:

1. `preflight_only=true` を指定して `OpenClaw NPM Release` を実行します。タグが存在する前は、プリフライトワークフローの検証専用ドライランに、現在の完全なワークフローブランチのコミット SHA を使用できます。
2. 通常のベータ優先フローには `npm_dist_tag=beta` を選択し、意図的に安定版を直接公開する場合にのみ `latest` を選択します。
3. 1 つの手動ワークフローで通常の CI に加え、ライブプロンプトキャッシュ、Docker、QA Lab、Matrix、および Telegram のカバレッジが必要な場合は、リリースブランチ、リリースタグ、または完全なコミット SHA に対して `Full Release Validation` を実行します。意図的に決定論的な通常のテストグラフのみが必要な場合は、代わりにリリース ref に対して手動の `CI` ワークフローを実行します。
4. 署名済みの x64 および ARM64 インストーラーを出荷する、プレリリースではない正確な `openclaw/openclaw-windows-node` リリースタグを選択します。それを `windows_node_tag` として保存し、検証済みのダイジェストマップを `windows_node_installer_digests` として保存します。リリース候補ヘルパーは両方を記録し、生成する公開コマンドに含めます。
5. 成功した `preflight_run_id`、`full_release_validation_run_id`、および正確な `full_release_validation_run_attempt` を保存します。
6. 信頼できる `main` から、同じ `tag`、同じ `npm_dist_tag`、選択した `windows_node_tag`、保存したその `windows_node_installer_digests`、保存した `preflight_run_id`、`full_release_validation_run_id`、および `full_release_validation_run_attempt` を指定して `OpenClaw Release Publish` を実行します。OpenClaw npm パッケージを昇格する前に、外部化された plugins を npm および ClawHub に公開します。
7. リリースが `beta` に公開された場合は、`openclaw/releases/.github/workflows/openclaw-npm-dist-tags.yml` ワークフローを使用して、その安定版を `beta` から `latest` に昇格します。
8. リリースを意図的に `latest` に直接公開し、`beta` でも同じ安定版ビルドを直ちに使用する必要がある場合は、同じリリースワークフローを使用して両方の dist-tag をその安定版に向けるか、スケジュールされた自己修復同期によって後で `beta` を移動させます。

dist-tag の変更は引き続き `NPM_TOKEN` を必要とするためリリース台帳リポジトリで行い、ソースリポジトリでは OIDC のみを使用した公開を維持します。これにより、直接公開パスとベータ優先の昇格パスの両方が文書化され、オペレーターから確認可能な状態に保たれます。

メンテナーがローカルの npm 認証にフォールバックする必要がある場合は、1Password CLI（`op`）コマンドを専用の tmux セッション内でのみ実行します。メインのエージェントシェルから `op` を直接呼び出さないでください。tmux 内で実行することで、プロンプト、アラート、および OTP の処理を確認可能にし、ホストでアラートが繰り返し発生するのを防ぎます。

## 公開リファレンス

- [`.github/workflows/full-release-validation.yml`](https://github.com/openclaw/openclaw/blob/main/.github/workflows/full-release-validation.yml)
- [`.github/workflows/package-acceptance.yml`](https://github.com/openclaw/openclaw/blob/main/.github/workflows/package-acceptance.yml)
- [`.github/workflows/openclaw-npm-release.yml`](https://github.com/openclaw/openclaw/blob/main/.github/workflows/openclaw-npm-release.yml)
- [`.github/workflows/openclaw-release-checks.yml`](https://github.com/openclaw/openclaw/blob/main/.github/workflows/openclaw-release-checks.yml)
- [`.github/workflows/openclaw-cross-os-release-checks-reusable.yml`](https://github.com/openclaw/openclaw/blob/main/.github/workflows/openclaw-cross-os-release-checks-reusable.yml)
- [`.github/workflows/docker-release.yml`](https://github.com/openclaw/openclaw/blob/main/.github/workflows/docker-release.yml)
- [`scripts/resolve-openclaw-package-candidate.mjs`](https://github.com/openclaw/openclaw/blob/main/scripts/resolve-openclaw-package-candidate.mjs)
- [`scripts/openclaw-npm-release-check.ts`](https://github.com/openclaw/openclaw/blob/main/scripts/openclaw-npm-release-check.ts)
- [`scripts/package-mac-dist.sh`](https://github.com/openclaw/openclaw/blob/main/scripts/package-mac-dist.sh)
- [`scripts/make_appcast.sh`](https://github.com/openclaw/openclaw/blob/main/scripts/make_appcast.sh)

メンテナーは、実際のランブックとして [`openclaw/maintainers/release/README.md`](https://github.com/openclaw/maintainers/blob/main/release/README.md) の非公開リリースドキュメントを使用します。

## 関連項目

- [リリースチャンネル](/ja-JP/install/development-channels)
