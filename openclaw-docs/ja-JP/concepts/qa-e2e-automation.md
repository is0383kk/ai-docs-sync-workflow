---
doc-schema-version: 1
read_when:
    - QA スタックの全体的な連携を理解する
    - qa-lab、qa-channel、またはトランスポートアダプターの拡張
    - リポジトリに基づく QA シナリオの追加
    - Gateway ダッシュボードを対象とした、より現実に近い QA 自動化の構築
summary: QA スタックの概要：qa-lab、qa-channel、リポジトリベースのシナリオ、ライブトランスポートレーン、トランスポートアダプター、レポート。
title: QA の概要
x-i18n:
    generated_at: "2026-07-26T09:39:04Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 91c34a50e6197195d57228d92b19caff1785ceaa5d82d7c88a1ec0ed76abd635
    source_path: concepts/qa-e2e-automation.md
    workflow: 16
---

プライベート QA スタックは、単体テストでは不可能な、実際のチャネルを模した現実的な方法で OpenClaw を検証します。

構成要素:

- `extensions/qa-channel`: DM、チャネル、スレッド、リアクション、編集、削除の各サーフェスを備えた合成メッセージチャネル。
- `extensions/qa-lab`: トランスクリプトの観察、受信メッセージの注入、Markdown レポートのエクスポートに使用するデバッガー UI、QA バス、シナリオプロファイル、ライブトランスポートアダプター。
- `qa/`: キックオフタスクとベースライン QA シナリオ用のリポジトリに基づくシードアセット。
- [Mantis](/ja-JP/concepts/mantis): 実際のトランスポート、ブラウザーのスクリーンショット、VM の状態、PR の証拠を必要とするバグの修正前後のライブ検証。

## コマンドサーフェス

すべての QA フローは `pnpm openclaw qa <subcommand>` で実行されます。多くには `pnpm qa:*` のスクリプトエイリアスがあり、どちらの形式でも動作します。

| コマンド                                            | 目的                                                                                                                                                                                                                                                             |
| --------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `qa run`                                            | `--qa-profile` を使用しないバンドル済み QA セルフチェック。`--qa-profile smoke-ci`、`--qa-profile release`、または `--qa-profile all` を使用する、分類体系に基づく成熟度プロファイルランナー。                                                                                                  |
| `qa suite`                                          | QA Gateway レーンに対して、リポジトリに基づくシナリオを実行します。`--runner multipass` はホストの代わりに使い捨ての Linux VM を使用します。                                                                                                                                         |
| `qa coverage`                                       | YAML シナリオカバレッジインベントリを出力します（マシン出力には `--json`、変更した動作に対応するシナリオの検索には `--match <query>`、ランタイムツールのフィクスチャカバレッジには `--tools`）。                                                                                  |
| `qa parity-report`                                  | モデル軸のパリティゲート向けに 2 つの `qa-suite-summary.json` ファイルを比較します。または、`--runtime-axis --token-efficiency` を使用して Codex と OpenClaw のランタイムパリティおよびトークン効率レポートを書き出します。                                                                          |
| `qa confidence-report`                              | マニフェストに照らして QA 証拠アーティファクトを分類し、不明項目ゼロの信頼度レポートを生成します。                                                                                                                                                                               |
| `qa confidence-self-test`                           | 信頼度ゲートがドリフトを検出することを証明する、シード済みネガティブコントロールカナリアを書き出します。                                                                                                                                                                                   |
| `qa jsonl-replay`                                   | 厳選された JSONL トランスクリプトをランタイムパリティ再生ハーネスで再生します。                                                                                                                                                                                         |
| `qa character-eval`                                 | 複数のライブモデルでキャラクター QA シナリオを実行し、評価済みレポートを生成します。[レポート](#reporting)を参照してください。                                                                                                                                                        |
| `qa manual`                                         | 選択したプロバイダー／モデルレーンに対して単発のプロンプトを実行します。                                                                                                                                                                                                      |
| `qa ui`                                             | QA デバッガー UI とローカル QA バスを起動します（エイリアス: `pnpm qa:lab:ui`）。                                                                                                                                                                                                |
| `qa docker-build-image`                             | 事前構築済み QA Docker イメージをビルドします。                                                                                                                                                                                                                                 |
| `qa docker-scaffold`                                | QA ダッシュボードと Gateway レーン用の docker-compose スキャフォールドを書き出します。                                                                                                                                                                                                |
| `qa up`                                             | QA サイトをビルドし、Docker ベースのスタックを起動して URL を出力します（エイリアス: `pnpm qa:lab:up`。`:fast` バリアントは `--use-prebuilt-image --bind-ui-dist --skip-ui-build` を追加します）。                                                                                              |
| `qa aimock`                                         | AIMock プロバイダーサーバーのみを起動します。                                                                                                                                                                                                                              |
| `qa mock-openai`                                    | シナリオ対応の `mock-openai` プロバイダーサーバーのみを起動します。                                                                                                                                                                                                        |
| `qa credentials doctor` / `add` / `list` / `remove` | 共有 Convex 認証情報プールを管理します。                                                                                                                                                                                                                           |
| `qa discord`                                        | 実際の非公開 Discord ギルドチャネルに対するライブトランスポートレーン。                                                                                                                                                                                                   |
| `qa matrix`                                         | 使い捨ての Tuwunel ホームサーバーに対する QA Lab Matrix プロファイル。[Matrix スモークレーン](#matrix-smoke-lanes)を参照してください。                                                                                                                                                      |
| `qa slack`                                          | 実際の非公開 Slack チャネルに対するライブトランスポートレーン。                                                                                                                                                                                                           |
| `qa telegram`                                       | 実際の非公開 Telegram グループに対するライブトランスポートレーン。                                                                                                                                                                                                          |
| `qa whatsapp`                                       | 実際の WhatsApp Web アカウントに対するライブトランスポートレーン。                                                                                                                                                                                                             |
| `qa mantis`                                         | ライブトランスポートのバグ向けの修正前後検証ランナー。Discord のステータスリアクションによる証拠、Crabbox のデスクトップ／ブラウザースモーク、Slack-in-VNC スモークを備えています。[Mantis](/ja-JP/concepts/mantis) および [Mantis Slack Desktop ランブック](/ja-JP/concepts/mantis-slack-desktop-runbook)を参照してください。 |

### プロファイルに基づく `qa run`

プロファイルに基づく `qa run` は `taxonomy.yaml` からメンバーシップを読み取り、解決されたシナリオを `qa suite` 経由でディスパッチします。`--surface` と `--category` は、個別のレーンを定義するのではなく、選択したプロファイルを絞り込みます。生成される `qa-evidence.json` には、選択したカテゴリの件数と不足しているカバレッジ ID を含むプロファイルスコアカードの概要が含まれます。個々の証拠エントリは、テスト、カバレッジの役割、結果に関する信頼できる唯一の情報源のままです。分類体系の機能カバレッジ ID は別名ではなく、厳密な証明対象です。プライマリシナリオのカバレッジは一致する ID を満たしますが、セカンダリカバレッジは参考情報にとどまります。各カバレッジ ID は、`taxonomy.yaml` の短いサーフェス ID を使用した正確な `taxonomy-surface.feature` です。シナリオの独立した `surface` フィールドは、実行／レポート用のラベル（たとえば `channel` や `runtime-tool`）であり、分類体系の所有権を定義するものではありません。

スリム証拠ではエントリごとの `execution` が省略され、`evidenceMode: "slim"` が設定されます。`smoke-ci` のデフォルトはスリムで、`--evidence-mode full` を指定すると完全なエントリに戻ります。

```bash
pnpm openclaw qa run \
  --qa-profile smoke-ci \
  --category channels.conversation-routing-and-delivery \
  --provider-mode mock-openai \
  --output-dir .artifacts/qa-e2e/smoke-ci-profile-dispatch
```

モックモデルプロバイダーと Crabline ローカルプロバイダーサーバーによる決定論的なプロファイル証明には `smoke-ci` を使用します。ライブチャネルに対する Stable/LTS 証明には `release` を使用します。`all` は、完全な分類体系の証拠を明示的に実行する場合にのみ使用してください。これは有効な成熟度カテゴリをすべて選択し、`qa_profile=all` を使用して `QA
Profile Evidence` GitHub Actions ワークフローからディスパッチできます。コマンドで OpenClaw のルートプロファイルも必要な場合は、ルートプロファイルを QA コマンドの前に指定します。

```bash
pnpm openclaw --profile work qa run --qa-profile smoke-ci
```

## オペレーターフロー

現在の QA オペレーターフローは、2 ペイン構成の QA サイトです。

- 左: エージェントを備えた Gateway ダッシュボード（Control UI）。
- 右: Slack 風のトランスクリプトとシナリオ計画を表示する QA Lab。

次のコマンドで実行します。

```bash
pnpm qa:lab:up
```

これにより QA サイトがビルドされ、Docker ベースの Gateway レーンが起動し、QA Lab ページが公開されます。オペレーターまたは自動化ループは、このページでエージェントに QA ミッションを与え、実際のチャネル動作を観察し、成功したこと、失敗したこと、ブロックされたままのことを記録できます。

毎回 Docker イメージを再ビルドせずに QA Lab UI をすばやく反復開発するには、QA Lab バンドルをバインドマウントしてスタックを起動します。

```bash
pnpm openclaw qa docker-build-image
pnpm qa:lab:build
pnpm qa:lab:up:fast
pnpm qa:lab:watch
```

`qa:lab:up:fast` は Docker サービスで事前構築済みイメージを使用し続け、`extensions/qa-lab/web/dist` を `qa-lab` コンテナにバインドマウントします。`qa:lab:watch` は変更時にそのバンドルを再ビルドし、QA Lab のアセットハッシュが変更されるとブラウザーが自動的に再読み込みされます。

### オブザーバビリティのスモークテスト

<Note>
オブザーバビリティ QA はソースチェックアウト専用のままです。npm tarball では意図的に QA Lab（および `qa-channel`）を省略しているため、パッケージ Docker リリースレーンでは `qa` コマンドを実行しません。診断インストルメンテーションを変更する場合は、ビルド済みのソースチェックアウトからこれらを実行してください。
</Note>

| エイリアス                                   | 実行内容                                                                                                                            |
| --------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------- |
| `pnpm qa:otel:smoke`                    | ローカルの OpenTelemetry レシーバーに加え、`diagnostics-otel` を有効にした `otel-trace-smoke` シナリオ。                                      |
| `pnpm qa:otel:collector-smoke`          | 実際の OpenTelemetry Collector Docker コンテナを介した同じレーン。エンドポイントの配線または Collector/OTLP の互換性を変更する場合に使用します。 |
| `pnpm qa:prometheus:smoke`              | `diagnostics-prometheus` を有効にした `docker-prometheus-smoke` シナリオ。                                                           |
| `pnpm qa:observability:smoke`           | `qa:otel:smoke`、続いて `qa:prometheus:smoke`。                                                                                      |
| `pnpm qa:observability:collector-smoke` | `qa:otel:collector-smoke`、続いて `qa:prometheus:smoke`。                                                                            |

`qa:otel:smoke` はローカルの OTLP/HTTP レシーバーを起動し、最小限の QA-channel
エージェントターンを実行してから、トレース、メトリクス、ログがエクスポートされたことを検証します。エクスポートされた
protobuf トレーススパンをデコードし、リリースに不可欠な構造として、
`openclaw.run`、`openclaw.harness.run`、最新の GenAI セマンティック規約に準拠した
モデル呼び出しスパン、`openclaw.context.assembled`、`openclaw.message.delivery`
がすべて存在することを確認します。このスモークは
`OTEL_SEMCONV_STABILITY_OPT_IN=gen_ai_latest_experimental` を強制するため、モデル呼び出し
スパンは `{gen_ai.operation.name} {gen_ai.request.model}` という名前を使用する必要があります。成功したターンでは、モデル
呼び出しが `StreamAbandoned` をエクスポートしてはなりません。また、生の診断
ID と `openclaw.content.*` 属性をトレースに含めてはなりません。シナリオの
プロンプトは、固定マーカーで応答し、固定の秘密文字列を出力しないようモデルに要求します。生の OTLP
ペイロードには、そのどちらも、シナリオ ID から導出された QA
セッションキーも含まれていてはなりません。QA スイートのアーティファクトの隣に `otel-smoke-summary.json`
を書き込みます。

`qa:prometheus:smoke` は認証されていないスクレイプが拒否されることを検証してから、
認証済みスクレイプに、プロンプト内容、応答内容、生の診断識別子、認証
トークン、ローカルパスを含まずに、リリースに不可欠なメトリクスファミリーが含まれることを
確認します。

### Matrix スモークレーン

モデルプロバイダーの認証情報を必要としない、実際のトランスポートを使用する Matrix スモークレーンでは、
決定論的なモック OpenAI プロバイダーを指定してリリースプロファイルを実行します。

```bash
pnpm openclaw qa matrix --provider-mode mock-openai --profile release
```

ライブのフロンティアプロバイダーレーンでは、OpenAI 互換の認証情報を
明示的に指定します。

```bash
OPENCLAW_LIVE_OPENAI_KEY="${OPENAI_API_KEY}" \
  pnpm openclaw qa matrix --provider-mode live-frontier --profile release
```

引数なしの `pnpm openclaw qa matrix` は完全な `all` プロファイルを実行し、
シナリオが失敗しても続行します。短いフィードバックループには `--fail-fast` を使用するか、
`--scenario <id>` を繰り返して個別のシナリオを選択します。明示的なシナリオ ID は
`--profile` より優先されます。

| プロファイル      | シナリオ | 目的                                                                                                                                  |
| ------------ | --------- | ---------------------------------------------------------------------------------------------------------------------------------------- |
| `all`        | 93        | 完全なカタログ（デフォルト）。                                                                                                              |
| `release`    | 2         | リリースに不可欠なチャンネルベースラインと allowlist のライブ再読み込み。                                                                             |
| `fast`       | 12        | スレッド、リアクション、承認、ポリシー、ボットゲーティング、暗号化返信に焦点を当てたカバレッジ。                                               |
| `transport`  | 50        | スレッド、DM/ルームのルーティング、自動参加、承認、リアクション、再起動、メンション/allowlist ポリシー、編集、複数アクターの順序付け。         |
| `media`      | 7         | 画像、生成画像、音声、添付ファイル、未対応メディア、暗号化メディアのカバレッジ。                                              |
| `e2ee-smoke` | 8         | 暗号化された返信、スレッド、ブートストラップ、復旧、再起動、墨消し、失敗に関する最小限のカバレッジ。                                       |
| `e2ee-deep`  | 18        | 状態喪失、バックアップ、キー復旧、デバイス衛生、SAS/QR/DM 検証。                                                            |
| `e2ee-cli`   | 9         | ハーネスを通じた `openclaw matrix encryption setup`、復旧キー、複数アカウント、Gateway ラウンドトリップ、自己検証コマンド。 |

プロファイルの所属とチャンネル要件は、`qa/scenarios/channels/` 配下の宣言的な Matrix
シナリオとともに定義されています。実行時にチャンネルドライバーが選択されます。
それらのライブ実装は
`extensions/qa-lab/src/live-transports/matrix/scenarios/` 配下にあります。

アダプターは Docker 内に使い捨ての Tuwunel ホームサーバー（デフォルト
イメージ `ghcr.io/matrix-construct/tuwunel:v1.5.1`、サーバー名 `matrix-qa.test`、
ポート `28008`）をプロビジョニングし、一時的なドライバー、SUT、オブザーバーユーザーを登録し、必要な
ルームを準備して、秘匿化されたリクエスト/レスポンス境界を記録します。次に、
そのトランスポートに限定された子 QA Gateway 内で実際の Matrix Plugin を実行し
（`qa-channel` は使用しません）、環境を破棄します。

共通オプション：

| フラグ                     | デフォルト           | 目的                                                                              |
| ------------------------ | ----------------- | ------------------------------------------------------------------------------------ |
| `--profile <profile>`    | `all`             | 上記のプロファイルから 1 つ選択します。                                                    |
| `--scenario <id>`        | -                 | シナリオを 1 つ選択します。繰り返し指定できます。                                                     |
| `--fail-fast`            | オフ               | 最初に失敗したチェックまたはシナリオの後で停止します。                                       |
| `--allow-failures`       | オフ               | シナリオが失敗しても失敗終了コードを返さずにアーティファクトを書き込みます。         |
| `--provider-mode <mode>` | `live-frontier`   | 決定論的ディスパッチには `mock-openai`、ライブプロバイダーには `live-frontier` を使用します。 |
| `--model <ref>`          | プロバイダーのデフォルト  | プライマリ `provider/model` 参照を設定します。                                          |
| `--alt-model <ref>`      | プロバイダーのデフォルト  | モデルを切り替えるシナリオで使用する代替モデルを設定します。                        |
| `--fast`                 | オフ               | 対応している場合、プロバイダーの高速モードを有効にします。                                           |
| `--output-dir <path>`    | 自動生成         | レポートディレクトリを選択します。相対パスは `--repo-root` を基準に解決されます。           |
| `--repo-root <path>`     | 現在のディレクトリ | 中立的な作業ディレクトリから実行します。                                                |
| `--sut-account <id>`     | `sut`             | 子 Gateway 設定内の Matrix アカウント ID を選択します。                            |

Matrix QA は共有 Matrix 認証情報をリースしません。アダプターが
使い捨てユーザーをローカルで作成するため、`--credential-source` または
`--credential-role` は受け付けません。ホームサーバーイメージは
`OPENCLAW_QA_MATRIX_TUWUNEL_IMAGE` で上書きし、否定的な無応答アサーションは
`OPENCLAW_QA_MATRIX_NO_REPLY_WINDOW_MS`（デフォルト `8000`、アクティブな
シナリオのタイムアウト以内に制限）で調整します。Matrix 暗号化のネイティブハンドルはクリーンアップ後も存続する可能性があるため、
単発コマンドは通常、アーティファクトのフラッシュ後にクリーンな終了を強制します。代わりに
コマンドが戻る必要がある直接テストハーネスでのみ、
`OPENCLAW_QA_MATRIX_DISABLE_FORCE_EXIT=1` を設定します。

各実行では、選択した出力ディレクトリに通常の QA Lab アーティファクト、
`qa-suite-report.md`、`qa-suite-summary.json`、
`qa-evidence.json` が書き込まれます。クリーンアップが失敗した場合は、出力された
`docker compose ... down --remove-orphans` 復旧コマンドを実行します。低速なランナーでは
無応答ウィンドウを増やします。高速な CI では、ウィンドウを小さくすると否定的
アサーションを短縮できます。

シナリオは、単体テストではエンドツーエンドで証明できないトランスポート動作を対象とします。
メンションゲーティング、ボット許可ポリシー、allowlist、トップレベルおよびスレッド内の
返信、DM ルーティング、リアクション処理、受信編集の抑制、再起動時のリプレイ重複排除、
ホームサーバー中断からの復旧、承認メタデータの配信、メディア処理、Matrix E2EE の
ブートストラップ/復旧/検証フローが含まれます。E2EE CLI プロファイルは、
Gateway の返信を確認する前に、同じ使い捨てホームサーバーを通じて `openclaw matrix encryption setup` と
検証コマンドも実行します。

`matrix-room-block-streaming` と `subagent-thread-spawn` は、
`--scenario` で明示的に選択すれば引き続き利用できますが、
デフォルトの `all` プロファイルには含まれません。

CI は
`.github/workflows/qa-live-transports-convex.yml` で同じコマンドサーフェスを使用します。スケジュール実行とリリース実行では、
リリースシナリオを実行します。手動の `matrix_profile=all` ディスパッチでは、
`transport`、`media`、`e2ee-smoke`、`e2ee-deep`、`e2ee-cli` の各プロファイルを
並列展開します。対象を絞ったディスパッチでは、1 つのジョブで `fast`、`release`、`transport` のいずれかを選択します。

### Discord Mantis シナリオ

Discord には、バグ再現用の Mantis 専用オプトインシナリオもあります。明示的なステータス
リアクションのタイムラインには `--scenario discord-status-reactions-tool-only` を使用し、
実際の Discord スレッドを作成して `message.thread-reply` が
`filePath` 添付ファイルを保持することを検証するには `--scenario discord-thread-reply-filepath-attachment`
を使用します。これらのシナリオは広範なスモークカバレッジではなく変更前後の再現プローブであるため、デフォルトの
ライブ Discord レーンには含まれません。スレッド添付ファイルの Mantis ワークフローでは、
QA 環境に
`MANTIS_DISCORD_VIEWER_CHROME_PROFILE_DIR` または
`MANTIS_DISCORD_VIEWER_CHROME_PROFILE_TGZ_B64` が設定されている場合、ログイン済みの Discord Web 証跡動画を追加することもできます。
このビューワープロファイルは視覚的なキャプチャ専用です。合否判定は
引き続き Discord REST オラクルに基づきます。

その他の実際のトランスポートを使用するスモークレーン：

```bash
pnpm openclaw qa discord
pnpm openclaw qa slack
pnpm openclaw qa telegram
pnpm openclaw qa whatsapp
```

これらは、2 つのボットまたはアカウント（ドライバー +
SUT）が存在する既存の実チャンネルを対象とします。これら 4 つのトランスポートに必要な環境変数、
シナリオ一覧、出力アーティファクト、Convex
認証情報プールについては、以下の
[Discord、Slack、Telegram、WhatsApp の QA リファレンス](#discord-slack-telegram-and-whatsapp-qa-reference)
で説明しています。

### Mantis Slack デスクトップおよびビジュアルタスクランナー

VNC レスキューを備えた完全な Slack デスクトップ VM 実行では、次を実行します。

```bash
pnpm openclaw qa mantis slack-desktop-smoke \
  --gateway-setup \
  --scenario slack-canary \
  --keep-lease
```

このコマンドは Crabbox のデスクトップ/ブラウザマシンをリースし、VM 内で Slack ライブ
レーンを実行し、VNC ブラウザで Slack Web を開き、デスクトップをキャプチャして、
`slack-qa/`、`slack-desktop-smoke.png`、および
`slack-desktop-smoke.mp4`（動画キャプチャが利用可能な場合）を
Mantis アーティファクトディレクトリへコピーします。Crabbox のデスクトップ/ブラウザリースには、
キャプチャツールとブラウザ/ネイティブビルド用ヘルパーパッケージがあらかじめ用意されているため、
このシナリオでフォールバックをインストールするのは古いリースの場合だけです。Mantis は
`mantis-slack-desktop-smoke-report.md` に合計およびフェーズごとの所要時間を報告するため、
実行が遅い場合、リースのウォームアップ、認証情報の取得、リモートセットアップ、
アーティファクトのコピーのどこに時間がかかったかを確認できます。VNC 経由で Slack Web に
手動ログインした後は、`--lease-id <cbx_...>` を再利用してください。再利用したリースでは、
Crabbox の pnpm ストアキャッシュもウォーム状態に保たれます。デフォルトの
`--hydrate-mode source` はソースチェックアウトから検証し、VM 内でインストール/ビルドを
実行します。再利用するリモートワークスペースに `node_modules` とビルド済みの
`dist/` がすでに存在する場合にのみ、`--hydrate-mode prehydrated` を使用してください。
このモードではコストの高いインストール/ビルド手順を省略し、ワークスペースの準備が
できていない場合はフェイルクローズします。`--gateway-setup` を指定すると、Mantis は
ポート `38973` で永続的な OpenClaw Slack Gateway を VM 内に残して実行します。
指定しない場合、このコマンドは通常のボット間 Slack QA レーンを実行し、
アーティファクトのキャプチャ後に終了します。

デスクトップの証拠を使用してネイティブ Slack 承認 UI を実証するには、Mantis の
承認チェックポイントモードを実行します。

```bash
pnpm openclaw qa mantis slack-desktop-smoke \
  --approval-checkpoints \
  --credential-source convex \
  --credential-role maintainer
```

このモードは `--gateway-setup` と同時には使用できません。Slack の承認シナリオを
実行し、承認以外のシナリオ ID を拒否し、保留中および解決済みの各承認状態で待機し、
観測された Slack API メッセージを `approval-checkpoints/<scenario>-pending.png` と
`approval-checkpoints/<scenario>-resolved.png` にレンダリングします。その後、チェックポイント、
メッセージの証拠、確認応答、またはレンダリング済みスクリーンショットのいずれかが
欠落しているか空の場合は失敗します。コールド状態の CI リースでは、
`slack-desktop-smoke.png` に Slack のサインイン画面が表示される場合があります。このレーンの
視覚的証拠には、承認チェックポイント画像を使用します。

デフォルトのチェックポイント実行では、2 つの標準 Slack 承認シナリオが維持されます。
オプトインの Codex 承認ルートのいずれかをキャプチャするには、
`--scenario slack-codex-approval-exec-native` または
`--scenario slack-codex-approval-plugin-native` で明示的に選択します。Mantis は両方を受け付け、
同じ保留中/解決済みスクリーンショットのペアを出力します。ランナーは、選択された各
Codex ルートについてチェックポイントとリモートコマンドの期限を延長し、
承認、エージェントの完了、解決済み更新の一連の処理全体が完了できるようにします。

オペレーターチェックリスト、GitHub ワークフローディスパッチコマンド、証拠コメントの
契約、ハイドレートモードの判断表、所要時間の解釈、および障害処理手順については、
[Mantis Slack デスクトップランブック](/ja-JP/concepts/mantis-slack-desktop-runbook)を参照してください。

エージェント/CV 形式のデスクトップタスクを実行するには、次を使用します。

```bash
pnpm openclaw qa mantis visual-task \
  --browser-url https://example.net \
  --expect-text "Example Domain" \
  --vision-model openai/gpt-5.6-luna
```

`visual-task` は Crabbox のデスクトップ/ブラウザマシンをリースまたは再利用し、
`crabbox record --while` を起動し、ネストされた `visual-driver` を介して
表示中のブラウザを操作し、`visual-task.png` をキャプチャします。
`--vision-mode image-describe` が選択されている場合は、スクリーンショットに対して
`openclaw infer image
describe` を実行し、`visual-task.mp4`、`mantis-visual-task-summary.json`、
`mantis-visual-task-driver-result.json`、および
`mantis-visual-task-report.md` を書き込みます。`--expect-text` が設定されている場合、
ビジョンプロンプトは構造化 JSON 判定（`visible`、`evidence`、
`reason`）を要求し、期待されるテキストを引用する証拠とともにモデルが
`visible: true` を報告した場合にのみ成功します。対象テキストを単に引用するだけの
`visible: false` 応答では、引き続きアサーションに失敗します。
画像理解プロバイダーを呼び出さずに、デスクトップ、ブラウザ、スクリーンショット、
動画の一連の処理を実証するモデルなしのスモークテストには、
`--vision-mode metadata` を使用します。録画は `visual-task` の必須アーティファクトです。
Crabbox が空でない `visual-task.mp4` を録画しなかった場合、視覚ドライバーが
成功していてもタスクは失敗します。失敗時、タスクがすでに成功しており
`--keep-lease` が設定されていない場合を除き、Mantis は VNC 用にリースを維持します。

### 認証情報プールの健全性チェック

プールされたライブ認証情報を使用する前に、次を実行します。

```bash
pnpm openclaw qa credentials doctor
```

doctor は Convex ブローカー環境変数（`OPENCLAW_QA_CONVEX_SITE_URL`、
`OPENCLAW_QA_CONVEX_ENDPOINT_PREFIX`）を確認し、エンドポイント設定を検証し、
`OPENCLAW_QA_CONVEX_SECRET_CI` と `OPENCLAW_QA_CONVEX_SECRET_MAINTAINER` については
設定済み/未設定の状態だけを報告します。また、メンテナーシークレットが存在する場合は、
管理および一覧取得が可能かを検証します。

## 正規シナリオカバレッジ

ルートの `taxonomy.yaml` は意味的カバレッジ ID を定義します。
`qa/scenarios/` 配下のシナリオ YAML ファイルは、各シナリオをそれらの ID に
マッピングし、実行メタデータを所有します。`channel` が唯一のチャネル要件であり、
`profiles` は名前付き実行への所属を宣言します。チャネルドライバーは、
実行レベルで交換可能な実装上の選択肢です。TypeScript ランナーはそのカタログを
クエリし、並行するシナリオまたはカバレッジのインベントリを維持しません。

静的な `qa coverage` 出力は、分類体系からシナリオへのマッピングを報告します。
実際の証明は `qa-evidence.json` から得られます。ここには、実行されたシナリオ、
カバレッジ ID、チャネル、実際に使用されたドライバー、および結果が記録されます。
チャネルとドライバーは報告のディメンションであり、追加のカバレッジ ID 語彙や
シナリオ適格性の軸ではありません。

QA パスに Docker を持ち込まず、使い捨ての Linux VM レーンを実行するには、
次を使用します。

```bash
pnpm openclaw qa suite --runner multipass --scenario channel-chat-baseline
```

これは新しい Multipass ゲストを起動し、依存関係をインストールし、ゲスト内で
OpenClaw をビルドし、`qa suite` を実行してから、通常の QA レポートと
サマリーをホスト上の `.artifacts/qa-e2e/...` にコピーします。ホスト上の
`qa suite` と同じシナリオ選択動作を再利用します。

ホストおよび Multipass のスイート実行では、デフォルトで、分離された Gateway
ワーカーを使用して複数の選択済みシナリオを並列実行します。`qa-channel` の
デフォルトの並行数は 4 で、選択されたシナリオ数を上限とします。ワーカー数を調整するには
`--concurrency
<count>`、直列実行には `--concurrency 1` を使用します。
パーソナルアシスタントのベンチマークパック（10 シナリオ）を実行するには、
`--pack personal-agent` を使用します。パックセレクターは、繰り返し指定される
`--scenario` フラグに対して加算的に動作します。明示的なシナリオを先に実行し、
その後、パック内の順序でパックシナリオを実行し、重複は削除されます。カスタム QA
ランナーが OpenTelemetry コレクターのセットアップをすでに提供している場合に、
`otel-trace-smoke` と `docker-prometheus-smoke` のシナリオをまとめて選択するには、
`--pack observability` を使用します。

いずれかのシナリオが失敗すると、コマンドは非ゼロで終了します。終了コードを失敗にせず
アーティファクトを取得する場合は、`--allow-failures` を使用します。

ライブ実行では、ゲストで実用的な、サポート対象の QA 認証入力を転送します。
これには、環境変数ベースのプロバイダーキー、QA ライブプロバイダー設定パス、および
存在する場合の `CODEX_HOME` が含まれます。ゲストがマウントされたワークスペースを
介して書き戻せるよう、`--output-dir` はリポジトリルート配下に置いてください。

## Discord、Slack、Telegram、WhatsApp の QA リファレンス

Matrix アダプターは、前述の使い捨て Docker ベースレーンを使用します。
Discord、Slack、Telegram、WhatsApp は既存の実トランスポートに対して実行されるため、
それらのリファレンスをここに記載します。

### 共通 CLI フラグ

これらのレーンは
`extensions/qa-lab/src/live-transports/shared/live-transport-cli.ts` を介して登録され、
同じフラグを受け付けます。

| フラグ                                  | デフォルト                                            | 説明                                                                                                                                     |
| ------------------------------------- | -------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------- |
| `--scenario <id>`                     | -                                                  | このシナリオだけを実行します。繰り返し指定できます。                                                                                                             |
| `--output-dir <path>`                 | `<repo>/.artifacts/qa-e2e/<transport>-<timestamp>` | レポート、サマリー、証拠、トランスポート固有のアーティファクト、および出力ログの書き込み先です。相対パスは `--repo-root` を基準に解決されます。 |
| `--repo-root <path>`                  | `process.cwd()`                                    | 中立的な cwd から呼び出す場合のリポジトリルートです。                                                                                               |
| `--sut-account <id>`                  | `sut`                                              | QA Gateway 設定内の一時アカウント ID です。                                                                                              |
| `--provider-mode <mode>`              | `live-frontier`                                    | `mock-openai`、`aimock`、または `live-frontier`。                                                                                                    |
| `--model <ref>` / `--alt-model <ref>` | プロバイダーのデフォルト                                   | プライマリ/代替モデル参照です。                                                                                                                   |
| `--fast`                              | オフ                                                | サポートされている場合のプロバイダー高速モードです。                                                                                                             |
| `--credential-source <env\|convex>`   | `env`                                              | [Convex 認証情報プール](#convex-credential-pool)を参照してください。                                                                                          |
| `--credential-role <maintainer\|ci>`  | CI では `ci`、それ以外では `maintainer`                 | `--credential-source convex` の場合に使用するロールです。                                                                                                    |
| `--allow-failures`                    | オフ                                                | シナリオが失敗しても失敗終了コードを返さずにアーティファクトを書き込みます。                                                                      |

各レーンは、いずれかのシナリオが失敗すると非ゼロで終了します。
`--allow-failures` は、失敗終了コードを設定せずにアーティファクトを書き込みます。
Telegram は、利用可能なシナリオ ID を出力して終了するための
`--list-scenarios` も受け付けます。他のレーンではこのフラグは公開されていません。

### Telegram QA

```bash
pnpm openclaw qa telegram
```

2 つの異なるボット（ドライバー + SUT）が参加する、実在する 1 つの非公開 Telegram
グループを対象とします。SUT ボットには Telegram ユーザー名が必要です。両方のボットで
`@BotFather` の **Bot-to-Bot Communication Mode** が有効になっていると、
ボット間の観測が最も安定します。

`--credential-source env` の場合に必要な環境変数：

- `OPENCLAW_QA_TELEGRAM_GROUP_ID` - 数値形式のチャット ID（文字列）。
- `OPENCLAW_QA_TELEGRAM_DRIVER_BOT_TOKEN`
- `OPENCLAW_QA_TELEGRAM_SUT_BOT_TOKEN`

`release` プロファイルは、メンテナンス対象の Telegram YAML シナリオを
選択します。`all` は、オプトインのセッション、使用量、返信チェーン、
ストリーミングのストレスチェックを追加します。明示的な `--scenario` の値は、
プロファイルより優先されます。

- `channel-canary`
- `channel-mention-gating`
- `telegram-help-command`
- `telegram-commands-command`
- `telegram-tools-compact-command`
- `telegram-whoami-command`
- `telegram-status-command`
- `telegram-repeated-command-authorization`
- `telegram-other-bot-command-gating`
- `telegram-context-command`
- `telegram-current-session-status-tool`
- `telegram-tool-only-usage-footer`
- `telegram-reply-chain-exact-marker`
- `telegram-stream-final-single-message`
- `telegram-long-final-reuses-preview`
- `telegram-long-final-three-chunks`

`release` プロファイルは、canary、メンションゲーティング、ネイティブコマンドの
返信、コマンドの宛先指定、bot 間のグループ返信を常に対象とします。`mock-openai`
には、決定論的な長い最終プレビューのチェックも含まれます。
`telegram-current-session-status-tool` と
`telegram-tool-only-usage-footer` は引き続きオプトインです。前者が安定するのは、
canary の直後にスレッド化した場合のみです。後者は、ツールのみの返信に付く
`/usage` フッターを実際の Telegram で検証します。現在の
デフォルトとオプションの区分を回帰参照付きで出力するには、`pnpm openclaw qa telegram
--list-scenarios --provider-mode mock-openai` を使用します。すべての
Telegram ライブアダプターシナリオでは、`--profile all` を使用します。

出力アーティファクト:

- `qa-suite-report.md`
- `qa-suite-summary.json`
- `qa-evidence.json` - ライブトランスポートチェックのエビデンスエントリ。
  プロファイル、カバレッジ、プロバイダー、チャネル、アーティファクト、結果、RTT
  フィールドを含みます。

パッケージの Telegram 実行では、同じ Telegram 認証情報コントラクトを使用します。RTT の
反復測定は、通常のパッケージ Telegram ライブレーンの一部です。選択した RTT チェックの
RTT 分布は、`result.timing` の下にある `qa-evidence.json` に
統合されます。

```bash
OPENCLAW_QA_CREDENTIAL_SOURCE=convex \
pnpm test:docker:npm-telegram-live
```

`OPENCLAW_QA_CREDENTIAL_SOURCE=convex` が設定されている場合、パッケージライブラッパーは
`kind: "telegram"` 認証情報をリースし、リースされたグループ、ドライバー、SUT
bot の環境変数をインストール済みパッケージの実行へエクスポートし、リースに Heartbeat を送信して、
シャットダウン時に解放します。パッケージラッパーのデフォルトは、
`channel-canary` の RTT チェック 20 回、RTT タイムアウト 30s、Convex が選択されている場合の
CI 外での Convex ロール `maintainer` です。RTT 専用コマンドや
Telegram 固有のサマリー形式を作成せずに RTT 測定を調整するには、
`OPENCLAW_NPM_TELEGRAM_RTT_SAMPLES`、`OPENCLAW_NPM_TELEGRAM_RTT_TIMEOUT_MS`、
または `OPENCLAW_NPM_TELEGRAM_RTT_MAX_FAILURES` を上書きします。

### Discord QA

```bash
pnpm openclaw qa discord
```

実際の非公開 Discord ギルドチャネル 1 つを、2 つの bot で対象にします。1 つは
ハーネスが制御するドライバー bot、もう 1 つはバンドルされた Discord Plugin を介して
子 OpenClaw Gateway が起動する SUT bot です。チャネルでのメンション処理、
SUT bot がネイティブの `/help` コマンドを Discord に登録済みであること、
およびオプトインの Mantis エビデンスシナリオを検証します。

`--credential-source env` の場合に必要な環境変数:

- `OPENCLAW_QA_DISCORD_GUILD_ID`
- `OPENCLAW_QA_DISCORD_CHANNEL_ID`
- `OPENCLAW_QA_DISCORD_DRIVER_BOT_TOKEN`
- `OPENCLAW_QA_DISCORD_SUT_BOT_TOKEN`
- `OPENCLAW_QA_DISCORD_SUT_APPLICATION_ID` - Discord が返す SUT bot のユーザー ID と
  一致する必要があります（一致しない場合、レーンは即座に失敗します）。

オプション:

- `OPENCLAW_QA_DISCORD_VOICE_CHANNEL_ID` は、`discord-voice-autojoin` 用の
  ボイス/ステージチャネルを選択します。指定しない場合、シナリオは SUT bot から見える
  最初のボイス/ステージチャネルを選択します。

Discord YAML モジュールシナリオ（`qa/scenarios/channels/discord-*.yaml`）:

- `discord-canary`
- `discord-mention-gating`
- `discord-native-help-command-registration`
- `discord-voice-autojoin` - オプトインのボイスシナリオ。単独で実行され、
  `channels.discord.voice.autoJoin` を有効にして、SUT bot の現在の
  Discord ボイス状態が対象のボイス/ステージチャネルであることを検証します。Convex の Discord
  認証情報には、オプションの `voiceChannelId` を含められます。含まれない場合、ランナー
  アダプターがギルド内で最初に見えるボイス/ステージチャネルを検出します。
- `discord-status-reactions-tool-only` - オプトインの Mantis シナリオ。SUT を
  `messages.statusReactions.enabled=true` により常時有効なツールのみのギルド返信へ
  切り替えるため、単独で実行されます。その後、REST
  リアクションタイムラインと HTML/PNG ビジュアルアーティファクトを取得します。Mantis の前後
  レポートでは、シナリオが提供した MP4 アーティファクトも `baseline.mp4`
  および `candidate.mp4` として保持されます。
- `discord-thread-reply-filepath-attachment` - オプトインの Mantis シナリオ。詳細は
  [Discord Mantis シナリオ](#discord-mantis-scenarios)を参照してください。

Discord ボイス自動参加シナリオを明示的に実行します:

```bash
pnpm openclaw qa discord \
  --scenario discord-voice-autojoin \
  --provider-mode mock-openai
```

Mantis ステータスリアクションシナリオを明示的に実行します:

```bash
pnpm openclaw qa discord \
  --scenario discord-status-reactions-tool-only \
  --provider-mode live-frontier \
  --model openai/gpt-5.6-luna \
  --alt-model openai/gpt-5.6-luna \
  --fast
```

出力アーティファクト:

- `qa-suite-report.md`
- `qa-suite-summary.json`
- `qa-evidence.json` - ライブトランスポートチェックのエビデンスエントリ。
- `discord-qa-reaction-timelines.json` および
  ステータスリアクションシナリオの実行時に `discord-status-reactions-tool-only-timeline.png`。

### Slack QA

```bash
pnpm openclaw qa slack
```

実際の非公開 Slack チャネル 1 つを、異なる 2 つの bot で対象にします。1 つは
ハーネスが制御するドライバー bot、もう 1 つはバンドルされた Slack Plugin を介して
子 OpenClaw Gateway が起動する SUT bot です。

`--credential-source env` の場合に必要な環境変数:

- `OPENCLAW_QA_SLACK_CHANNEL_ID`
- `OPENCLAW_QA_SLACK_DRIVER_BOT_TOKEN`
- `OPENCLAW_QA_SLACK_SUT_BOT_TOKEN`
- `OPENCLAW_QA_SLACK_SUT_APP_TOKEN`

オプション:

- `OPENCLAW_QA_SLACK_APPROVAL_CHECKPOINT_DIR` は、Mantis のビジュアル承認
  チェックポイントを有効にします。アダプターは `<scenario>.pending.json` と
  `<scenario>.resolved.json` を書き込み、一致する `.ack.json` ファイルを待機します。
- `OPENCLAW_QA_SLACK_APPROVAL_CHECKPOINT_TIMEOUT_MS` は、チェックポイントの
  確認応答タイムアウトを上書きします。デフォルトは `120000` です。

Slack ライブアダプターを通じて公開される正規 YAML シナリオ:

- `thread-follow-up`
- `thread-isolation`

Slack YAML モジュールシナリオ（`qa/scenarios/channels/slack-*.yaml`）:

- `slack-canary`
- `slack-mention-gating`
- `slack-allowlist-block`
- `slack-channel-disabled-warning` - 設定上無効なチャネルが返信せずに
  構造化された警告を出力することを確認する、オプトインの実 Slack プローブ。
- `slack-top-level-reply-shape`
- `slack-restart-resume`
- `slack-progress-commentary-true`、`slack-progress-commentary-false`、
  `slack-progress-commentary-omitted`、および
  `slack-progress-commentary-verbose-dedupe` - 独立した解説/ツール進捗制御、
  キー省略時のレガシーデフォルト、および永続的な詳細進捗が有効な場合の単一配信動作を確認する、
  オプトインの実 Slack プローブ。
- `slack-reaction-glyph-native` - オプトインのライブメッセージツールリアクションシナリオ。
  エージェントに正確な `✅` グリフを渡すよう指示し、対象メッセージ上で
  Slack が SUT bot 用に `white_check_mark` を保存したことを確認します。
- `slack-chart-presentation-native` - ネイティブの `data_visualization` ブロックと
  正確なアクセシブルテキストを検証する、オプトインのポータブルチャートシナリオ。
- `slack-table-presentation-native` - ネイティブの `data_table` ブロック、
  正確な行、アクセシブルテキストを検証する、オプトインのポータブルテーブルシナリオ。
- `slack-table-invalid-blocks-fallback` - 101 データ行とヘッダーを含む、
  構造的に読み取り可能な制限超過の raw テーブルを本番の Slack 送信パス経由で送信する、
  オプトインのダイレクトトランスポートシナリオ。Slack 自体が `invalid_blocks` を返すことを証明し、
  保存された書式無効時のフォールバックが完全で、ネイティブデータブロックを含まないことを
  検証します。シナリオ詳細には、安全なエラーコード、件数、真偽値の
  エビデンスのみを保持します。
- `slack-approval-exec-native` - オプトインのネイティブ Slack exec 承認シナリオ。
  Gateway を介して exec 承認を要求し、Slack メッセージにネイティブの承認ボタンが
  あることを検証して承認を解決し、解決済みの Slack
  更新を検証します。
- `slack-approval-plugin-native` - オプトインのネイティブ Slack Plugin 承認
  シナリオ。exec と Plugin の承認転送を同時に有効化し、Plugin
  イベントが exec 承認ルーティングによって抑制されないようにしたうえで、同じ
  保留中/解決済みのネイティブ Slack UI パスを検証します。
- `slack-codex-approval-exec-native` - オプトインの Codex Guardian コマンド承認
  シナリオ。Codex Plugin を Guardian モードで有効化し、Slack を起点とする
  Gateway エージェントターンを Codex app-server ハーネス経由でルーティングし、
  `openclaw-codex-app-server` に対するネイティブ Slack Plugin 承認プロンプトを待って
  解決し、Codex ターンが想定されるコマンド出力マーカーとアシスタントマーカーで
  完了することを検証します。
- `slack-codex-approval-plugin-native` - オプトインの Codex Guardian ファイル承認
  シナリオ。ワークスペース外の `apply_patch` 命令を使用して Codex に
  app-server のファイル変更承認ルートを出力させた後、同じネイティブ Slack の
  保留中/解決済み承認パス、最終アシスタントマーカー、正確なファイル内容を検証してから
  クリーンアップします。

Codex 承認シナリオには、`openai/*` または `codex/*` の `--model`、
通常のライブモデル認証情報、および Codex Plugin が受け入れる Codex 認証または API キー認証が必要です。
シナリオ詳細には、秘匿化された Slack 承認メタデータに加えて、Codex app-server メソッド、
選択された Codex モデルキー、最終 Codex ターンステータス、操作マーカーの検証が含まれます。

出力アーティファクト:

- `qa-suite-report.md`
- `qa-suite-summary.json`
- `qa-evidence.json` - ライブトランスポートチェックのエビデンスエントリ。
- `approval-checkpoints/` - Mantis が
  `OPENCLAW_QA_SLACK_APPROVAL_CHECKPOINT_DIR` を設定した場合のみ。チェックポイント JSON、
  確認応答 JSON、保留中/解決済みのスクリーンショットを含みます。

#### Slack ワークスペースのセットアップ

このレーンには、1 つのワークスペース内に異なる 2 つの Slack アプリと、両方の
bot が参加しているチャネルが必要です:

- `channelId` - 両方の bot が招待されているチャネルの `Cxxxxxxxxxx` ID。
  専用チャネルを使用してください。このレーンは実行のたびに投稿します。
- `driverBotToken` - **ドライバー**アプリの bot トークン（`xoxb-...`）。
- `sutBotToken` - **SUT** アプリの bot トークン（`xoxb-...`）。bot ユーザー ID を
  区別できるよう、ドライバーとは別の Slack アプリである必要があります。
- `sutAppToken` - `connections:write` を持つ SUT アプリの
  アプリレベルトークン（`xapp-...`）。SUT アプリがイベントを受信できるよう、Socket Mode で使用します。

本番ワークスペースを再利用するよりも、QA 専用の Slack ワークスペースを
使用することを推奨します。

以下の SUT マニフェストは、バンドルされた Slack Plugin の
本番インストール（`extensions/slack/src/setup-shared.ts:12`）を、ライブ Slack QA スイートが対象とする
権限とイベントに意図的に限定しています。ユーザーから見た本番チャネルのセットアップについては、
[Slack チャネルのクイックセットアップ](/ja-JP/channels/slack#quick-setup)を参照してください。QA のドライバー/SUT
ペアが意図的に分離されているのは、このレーンが 1 つのワークスペース内で異なる 2 つの bot ユーザー
ID を必要とするためです。

**1. ドライバーアプリを作成する**

[api.slack.com/apps](https://api.slack.com/apps) → _Create New App_ →
_From a manifest_ に移動し、QA ワークスペースを選択して次のマニフェストを貼り付け、
_Install to Workspace_ を実行します:

```json
{
  "display_information": {
    "name": "OpenClaw QA ドライバー",
    "description": "OpenClaw QA Slack ライブレーン用のテストドライバー bot"
  },
  "features": {
    "bot_user": {
      "display_name": "OpenClaw QA ドライバー",
      "always_online": true
    }
  },
  "oauth_config": {
    "scopes": {
      "bot": ["chat:write", "channels:history", "groups:history", "users:read"]
    }
  },
  "settings": {
    "socket_mode_enabled": false
  }
}
```

_Bot User OAuth Token_（`xoxb-...`）をコピーします。これが
`driverBotToken` になります。ドライバーに必要なのは、メッセージの投稿と自身の識別のみです。
イベントも Socket Mode も不要です。

**2. SUT アプリを作成する**

同じワークスペースで _Create New App → From a manifest_ を繰り返します。この QA アプリは、
バンドルされた Slack Plugin の本番マニフェスト（`extensions/slack/src/setup-shared.ts:12`）を意図的に
縮小したものを使用します。ライブ Slack QA スイートではまだリアクション処理を対象としていないため、
リアクションのスコープとイベントは省略されています。

```json
{
  "display_information": {
    "name": "OpenClaw QA SUT",
    "description": "OpenClaw 用 OpenClaw QA SUT コネクタ"
  },
  "features": {
    "bot_user": {
      "display_name": "OpenClaw QA SUT",
      "always_online": true
    },
    "app_home": {
      "home_tab_enabled": true,
      "messages_tab_enabled": true,
      "messages_tab_read_only_enabled": false
    }
  },
  "oauth_config": {
    "scopes": {
      "bot": [
        "app_mentions:read",
        "assistant:write",
        "channels:history",
        "channels:read",
        "chat:write",
        "commands",
        "emoji:read",
        "files:read",
        "files:write",
        "groups:history",
        "groups:read",
        "im:history",
        "im:read",
        "im:write",
        "mpim:history",
        "mpim:read",
        "mpim:write",
        "pins:read",
        "pins:write",
        "usergroups:read",
        "users:read"
      ]
    }
  },
  "settings": {
    "socket_mode_enabled": true,
    "event_subscriptions": {
      "bot_events": [
        "app_home_opened",
        "app_mention",
        "channel_rename",
        "member_joined_channel",
        "member_left_channel",
        "message.channels",
        "message.groups",
        "message.im",
        "message.mpim",
        "pin_added",
        "pin_removed"
      ]
    }
  }
}
```

Slack がアプリを作成したら、その設定ページで次の 2 つを行います。

- _Install to Workspace_ → _Bot User OAuth Token_ をコピー → これが
  `sutBotToken` になります。
- _Basic Information → App-Level Tokens → Generate Token and Scopes_ → 
  スコープ `connections:write` を追加 → 保存 → `xapp-...` の値をコピー → これが
  `sutAppToken` になります。

各トークンで `auth.test` を呼び出し、2 つのボットのユーザー ID が異なることを確認します。ランタイムはユーザー ID によってドライバーと SUT を区別します。両方に同じアプリを再利用すると、メンションゲーティングが直ちに失敗します。

**3. チャンネルを作成する**

QA ワークスペースでチャンネル（例: `#openclaw-qa`）を作成し、チャンネル内から両方のボットを招待します。

```text
/invite @OpenClaw QA Driver
/invite @OpenClaw QA SUT
```

_channel info → About → Channel ID_ から `Cxxxxxxxxxx` ID をコピーします。これが
`channelId` になります。パブリックチャンネルを使用できます。プライベートチャンネルを使用する場合でも、両方のアプリにはすでに `groups:history` があるため、ハーネスによる履歴の読み取りは引き続き成功します。

**4. 認証情報を登録する**

方法は 2 つあります。単一マシンでのデバッグには環境変数を使用するか（4 つの
`OPENCLAW_QA_SLACK_*` 変数を設定して `--credential-source env` を渡します）、共有 Convex プールにシードして CI や他のメンテナーがリースできるようにします。

Convex プールの場合は、4 つのフィールドを JSON ファイルに書き込みます。

```json
{
  "channelId": "Cxxxxxxxxxx",
  "driverBotToken": "xoxb-...",
  "sutBotToken": "xoxb-...",
  "sutAppToken": "xapp-..."
}
```

シェルで `OPENCLAW_QA_CONVEX_SITE_URL` と `OPENCLAW_QA_CONVEX_SECRET_MAINTAINER` をエクスポートした状態で、登録して確認します。

```bash
pnpm openclaw qa credentials add \
  --kind slack \
  --payload-file slack-creds.json \
  --note "QA Slack pool seed"

pnpm openclaw qa credentials list --kind slack --status all --json
```

`count: 1`、`status: "active"` があり、`lease` フィールドがないことを確認します。

**5. エンドツーエンドで検証する**

両方のボットがブローカー経由で相互に通信できることを確認するため、ローカルでレーンを実行します。

```bash
pnpm openclaw qa slack \
  --credential-source convex \
  --credential-role maintainer \
  --output-dir .artifacts/qa-e2e/slack-local
```

正常な実行は 30 秒を大幅に下回る時間で完了し、`qa-suite-report.md` では
`slack-canary` と `slack-mention-gating` の両方のステータスが `pass` と表示されます。レーンが約 90 秒間停止して `Convex credential pool exhausted
for kind "slack"` で終了する場合、プールが空であるか、すべての行がリース中です。どちらであるかは `qa
credentials list --kind slack --status all --json` で確認できます。

### WhatsApp QA

```bash
pnpm openclaw qa whatsapp
```

2 つの専用 WhatsApp Web アカウントを対象とします。1 つはハーネスによって制御されるドライバーアカウントで、もう 1 つはバンドルされた WhatsApp Plugin を通じて子 OpenClaw Gateway が起動する SUT アカウントです。

`--credential-source env` の場合に必要な環境変数:

- `OPENCLAW_QA_WHATSAPP_DRIVER_PHONE_E164`
- `OPENCLAW_QA_WHATSAPP_SUT_PHONE_E164`
- `OPENCLAW_QA_WHATSAPP_DRIVER_AUTH_ARCHIVE_BASE64`
- `OPENCLAW_QA_WHATSAPP_SUT_AUTH_ARCHIVE_BASE64`

オプション:

- `OPENCLAW_QA_WHATSAPP_GROUP_JID` は、
  `whatsapp-mention-gating`、`whatsapp-group-pending-history-context`、
  `whatsapp-broadcast-group-fanout`、`whatsapp-group-activation-always`、
  `whatsapp-group-reply-to-bot-triggers`、グループアクション／メディア／投票シナリオ、
  および `whatsapp-group-allowlist-block` などのグループシナリオを有効にします。

WhatsApp YAML シナリオ（`qa/scenarios/channels/whatsapp-*.yaml`）:

- ベースラインとグループゲーティング: `whatsapp-canary`、`whatsapp-pairing-block`、
  `whatsapp-mention-gating`、`whatsapp-group-pending-history-context`、
  `whatsapp-group-activation-always`、`whatsapp-group-reply-to-bot-triggers`、
  `whatsapp-top-level-reply-shape`、`whatsapp-restart-resume`、
  `whatsapp-group-allowlist-block`。
- ネイティブコマンド: `whatsapp-help-command`、`whatsapp-status-command`、
  `whatsapp-commands-command`、`whatsapp-tools-compact-command`、
  `whatsapp-whoami-command`、`whatsapp-context-command`、
  `whatsapp-native-new-command`。
- 返信と最終出力の動作: `whatsapp-tool-only-usage-footer`、
  `whatsapp-reply-to-message`、`whatsapp-group-reply-to-message`、
  `whatsapp-reply-to-mode-batched`、`whatsapp-reply-context-isolation`、
  `whatsapp-reply-delivery-shape`、`whatsapp-stream-final-message-accounting`。
- ユーザーパスのメッセージアクション: `whatsapp-agent-message-action-react` は
  実際のドライバー DM から開始し、モデルに `message` ツールを呼び出させ、
  WhatsApp のネイティブリアクションを監視します。`whatsapp-agent-message-action-upload-file` は
  `message(action=upload-file)` に同じ実行姿勢を使用し、
  WhatsApp のネイティブメディアを監視します。`whatsapp-group-agent-message-action-react` と
  `whatsapp-group-agent-message-action-upload-file` は、実際の WhatsApp グループで
  同じユーザー可視アクションを実証します。
- グループファンアウト: `whatsapp-broadcast-group-fanout` は、メンションを含む
  1 件の WhatsApp グループメッセージから開始し、`main`
  と `qa-second` から個別の可視返信があることを検証します。
- グループのアクティベーション: `whatsapp-group-activation-always` は実際のグループ
  セッションを `/activation always` に変更し、メンションのないグループメッセージによって
  エージェントが起動することを実証した後、`/activation mention` に戻します。
  `whatsapp-group-reply-to-bot-triggers` はボットの返信をシードし、明示的なメンションなしで
  それに対するネイティブな引用返信を送信し、その返信コンテキストから
  エージェントが起動することを検証します。
- 受信メディアと構造化メッセージ: `whatsapp-inbound-image-caption`、
  `whatsapp-audio-preflight`、`whatsapp-inbound-structured-messages`、
  `whatsapp-group-audio-gating`、`whatsapp-inbound-reaction-no-trigger`。
  これらは、実際の WhatsApp の画像、音声、ドキュメント、位置情報、連絡先、
  ステッカー、リアクションイベントをドライバー経由で送信します。
- Gateway コントラクトの直接プローブ: `whatsapp-outbound-media-matrix`、
  `whatsapp-outbound-document-preserves-filename`、`whatsapp-outbound-poll`、
  `whatsapp-outbound-send-serialization`、
  `whatsapp-group-outbound-media`、`whatsapp-group-outbound-poll`、
  `whatsapp-message-actions`、`whatsapp-reply-context-isolation`、
  `whatsapp-reply-delivery-shape`。これらは意図的にモデルへのプロンプトを迂回し、
  決定論的な Gateway／チャネルの `send`、`poll`、
  および `message.action` コントラクトを実証します。
- アクセス制御のカバレッジ: `whatsapp-access-control-dm-open`、
  `whatsapp-access-control-dm-disabled`、`whatsapp-access-control-group-open`、
  `whatsapp-access-control-group-disabled`、`whatsapp-group-allowlist-block`。
- ネイティブ承認: `whatsapp-approval-exec-deny-native`、
  `whatsapp-approval-exec-native`、`whatsapp-approval-exec-reaction-native`、
  `whatsapp-approval-exec-group-reaction-native`、
  `whatsapp-approval-plugin-native`。
- ステータスリアクション: `whatsapp-status-reactions`、
  `whatsapp-status-reaction-lifecycle`。

カタログには現在 52 個のシナリオがあります。`live-frontier` のデフォルトレーンは、
高速なスモークカバレッジのため 8 個のシナリオに抑えられています。`mock-openai`
のデフォルトレーンは、モデル出力のみをモックしながら、実際の WhatsApp
トランスポートを通じて 39 個のシナリオを決定論的に実行します。承認シナリオと、
一部の高負荷／ブロッキングチェックは、引き続きシナリオ ID で明示的に指定します。

WhatsApp QA ドライバーは、構造化されたライブイベント（`text`、`media`、
`location`、`reaction`、および `poll`）を監視し、メディア、投票、
連絡先、位置情報、ステッカーを能動的に送信できます。QA Lab は、WhatsApp の非公開
ランタイムファイルへ直接アクセスするのではなく、`@openclaw/whatsapp/api.js` パッケージ
サーフェスを通じてそのドライバーをインポートします。グループ監視では、`fromJid` が
グループ JID であり、`participantJid` と `fromPhoneE164` が参加者である送信者を識別します。
メッセージ内容はデフォルトで秘匿化されます。Gateway による投票、ファイルアップロード、
メディア、グループ投票、グループメディア、返信形式の直接プローブは、トランスポート／API
コントラクトのチェックです。ユーザーのプロンプトによってエージェントが同じアクションを
選択したことの証明としては扱われません。ユーザーパスによるアクションの証明は、
`whatsapp-agent-message-action-react` や `whatsapp-group-agent-message-action-react` などのシナリオから得られます。
これらでは、ドライバーが通常の WhatsApp メッセージを送信し、
QA Lab が結果として生成される WhatsApp のネイティブ成果物を監視します。
WhatsApp シナリオの詳細には、各シナリオの実行姿勢（`user-path`、
`direct-gateway`、または `native-approval`）が含まれるため、エビデンスが
実際に実証するものより強いコントラクトだと誤解されることはありません。

出力成果物:

- `qa-suite-report.md`
- `qa-suite-summary.json`
- `qa-evidence.json` - ライブトランスポートチェックのエビデンスエントリ。

### Convex 認証情報プール

Discord、Slack、Telegram、WhatsApp の各レーンは、上記の環境変数を読み取る代わりに、
共有 Convex プールから認証情報をリースできます。
`--credential-source convex` を渡します（または `OPENCLAW_QA_CREDENTIAL_SOURCE=convex` を設定します）。
QA Lab は排他的リースを取得し、実行中は Heartbeat を送り、
シャットダウン時にリースを解放します。プールの種類は `"discord"`、`"slack"`、
`"telegram"`、および `"whatsapp"` です。

ブローカーが `admin/add` で検証するペイロード形式:

- Discord（`kind: "discord"`）: `{ guildId: string, channelId: string,
driverBotToken: string, sutBotToken: string, sutApplicationId: string }`。
- Telegram（`kind: "telegram"`）: `{ groupId: string, driverToken: string,
sutToken: string }` - `groupId` は数値のチャット ID 文字列でなければなりません。
- Telegram の実ユーザー（`kind: "telegram-user"`）: `{ groupId: string, sutToken:
string, testerUserId: string, testerUsername: string, telegramApiId:
string, telegramApiHash: string, tdlibDatabaseEncryptionKey: string,
tdlibArchiveBase64: string, tdlibArchiveSha256: string,
desktopTdataArchiveBase64: string, desktopTdataArchiveSha256: string }` -
  Mantis Telegram Desktop の証明専用です。汎用 QA Lab レーンは
  この種類を取得してはなりません。
- WhatsApp（`kind: "whatsapp"`）: `{ driverPhoneE164: string, sutPhoneE164:
string, driverAuthArchiveBase64: string, sutAuthArchiveBase64: string,
groupJid?: string }` - 電話番号は互いに異なる E.164 文字列でなければなりません。

Mantis Telegram Desktop の証明ワークフローは、TDLib CLI ドライバーと
Telegram Desktop の立会いの両方に対して 1 つの排他的な Convex
`telegram-user` リースを保持し、証明の公開後に解放します。

PR で決定論的なビジュアル差分が必要な場合、Telegram のフォーマッターまたは
配信レイヤーを変更しながら、Mantis は `main` と PR の head で同じ
モックモデル返信を使用できます。キャプチャのデフォルトは PR コメント向けに調整されています。
標準 Crabbox クラス、24fps のデスクトップ録画、24fps のモーション GIF、
1920px のプレビュー幅です。変更前／変更後のコメントでは、意図した GIF のみを含む
クリーンなバンドルを公開する必要があります。

Slack レーンもこのプールを使用できます。Slack のペイロード形式チェックは現在、
ブローカーではなく Slack QA ランナーにあります。`{ channelId: string,
driverBotToken: string, sutBotToken: string, sutAppToken: string }` を、
`Cxxxxxxxxxx` のような Slack チャネル ID とともに使用します。アプリと
スコープのプロビジョニングについては、
[Slack ワークスペースのセットアップ](#setting-up-the-slack-workspace)を参照してください。

運用環境変数と Convex ブローカーのエンドポイントコントラクトについては、
[テスト → Convex 経由の共有 Telegram 認証情報](/ja-JP/help/testing#shared-telegram-credentials-via-convex-v1)
を参照してください（このセクション名はマルチチャネルプールより前から存在します。
リースのセマンティクスは各種類で共有されています）。

## リポジトリに格納されたシード

シードアセットは `qa/` にあります:

- `qa/scenarios/index.yaml`
- `qa/scenarios/<theme>/*.yaml`

これらは意図的に git に含められており、人間とエージェントの両方が
QA 計画を確認できます。

`qa-lab` は汎用 YAML シナリオランナーのままです。各シナリオ YAML ファイルは、
1 回のテスト実行における信頼できる唯一の情報源であり、次を定義する必要があります:

- トップレベルの `title`
- `scenario` メタデータ
- `scenario` 内のオプションのカテゴリ、機能、レーン、リスクのメタデータ
- `scenario` 内のドキュメントおよびコード参照
- `scenario` 内のオプションの Plugin 要件
- `scenario` 内のオプションの Gateway 設定パッチ
- フローシナリオ用の実行可能なトップレベル `flow`、または
  Vitest および Playwright シナリオ用の `scenario.execution.kind`／`scenario.execution.path`

`flow` を支える再利用可能なランタイムサーフェスは、汎用かつ
横断的なままにします。たとえば、YAML シナリオでは、特別なケースのランナーを追加せずに、
Gateway の `browser.request` シームを介して埋め込みの Control UI を操作する
ブラウザ側ヘルパーと、トランスポート側ヘルパーを組み合わせられます。

シナリオファイルは、ソースツリーのフォルダーではなく、プロダクト機能ごとに
グループ化してください。ファイルを移動してもシナリオ ID は安定させ、実装の
トレーサビリティには `docsRefs` と `codeRefs` を使用します。

ベースラインの一覧は、少なくとも次をカバーできる十分な広さを維持してください。

- DM とチャンネルチャット
- スレッドの動作
- メッセージアクションのライフサイクル
- Cron コールバック
- メモリの想起
- モデルの切り替え
- サブエージェントへの引き継ぎ
- リポジトリとドキュメントの読み取り
- Lobster Invaders などの小規模なビルドタスク 1 件

## プロバイダーモックレーン

`qa suite` には、2 つのローカルプロバイダーモックレーンがあります。

- `mock-openai` は、シナリオ対応の OpenClaw モックです。リポジトリベースの QA と
  パリティゲート向けのデフォルトの決定論的モックレーンとして維持されます。
- `aimock` は、実験的なプロトコル、フィクスチャ、記録・再生、および
  カオスカバレッジ向けに AIMock ベースのプロバイダーサーバーを起動します。
  これは追加的なものであり、`mock-openai` シナリオディスパッチャーを置き換えません。

プロバイダーレーンの実装は `extensions/qa-lab/src/providers/` 配下にあります。
各プロバイダーは、独自のデフォルト、ローカルサーバーの起動、Gateway のモデル設定、
認証プロファイルのステージング要件、およびライブ／モック機能フラグを所有します。
共有スイートと Gateway のコードは、プロバイダー名による分岐ではなく、
プロバイダーレジストリを介してルーティングします。

## トランスポートアダプター

`qa-lab` は、YAML QA シナリオ向けの汎用トランスポートシームを所有します。`qa-channel` は
合成デフォルトです。`crabline` はローカルのプロバイダー形式サーバーを起動し、
OpenClaw の通常のチャンネル Plugin をそれらに対して実行します。`live` は、
実際のプロバイダー認証情報と外部チャンネル用に予約されています。

アーキテクチャレベルでは、責務の分割は次のとおりです。

- `qa-lab` は、汎用シナリオ実行、ワーカーの並行処理、アーティファクトの
  書き込み、およびレポート作成を所有します。
- トランスポートアダプターは、Gateway の設定、準備完了確認、受信と送信の
  監視、トランスポートアクション、および正規化されたトランスポート状態を所有します。
- `qa/scenarios/` 配下の YAML シナリオファイルがテスト実行を定義し、`qa-lab` が
  それらを実行する再利用可能なランタイムサーフェスを提供します。

### チャンネルの追加

YAML QA システムにチャンネルを追加するには、チャンネル実装に加えて、
チャンネル契約を検証するシナリオパックが必要です。スモーク CI の
カバレッジには、対応する Crabline ローカルプロバイダーサーバーを追加し、
`crabline` ドライバーを介して公開します。

共有の `qa-lab` ホストがフローを所有できる場合は、新しいトップレベルの
QA コマンドルートを追加しないでください。

`qa-lab` は、共有ホストの仕組みを所有します。

- `openclaw qa` コマンドルート
- スイートの起動と終了処理
- ワーカーの並行処理
- アーティファクトの書き込み
- レポートの生成
- シナリオの実行
- 以前の `qa-channel` シナリオ向けの互換性エイリアス

ランナー Plugin はトランスポート契約を所有します。

- `openclaw qa <runner>` を共有の `qa` ルート配下にマウントする方法
- そのトランスポート用に Gateway を設定する方法
- 準備完了を確認する方法
- 受信イベントを注入する方法
- 送信メッセージを監視する方法
- トランスクリプトと正規化されたトランスポート状態を公開する方法
- トランスポートを利用するアクションを実行する方法
- トランスポート固有のリセットまたはクリーンアップを処理する方法

新しいチャンネルを採用するための最低要件は次のとおりです。

1. 共有の `qa` ルートの所有者は `qa-lab` のままにします。
2. 共有の `qa-lab` ホストシーム上にトランスポートランナーを実装します。
3. トランスポート固有の仕組みは、ランナー Plugin またはチャンネル
   ハーネス内に保持します。
4. 競合するルートコマンドを登録するのではなく、ランナーを
   `openclaw qa <runner>` としてマウントします。ランナー Plugin は
   `openclaw.plugin.json` で `qaRunners` を宣言し、`runtime-api.ts` から対応する
   `qaRunnerCliRegistrations` 配列をエクスポートする必要があります。`runtime-api.ts` は
   軽量に保ち、遅延 CLI とランナー実行は別々のエントリポイントの背後に
   置いてください。任意の `adapterFactory` を使用すると、コマンドの既存の
   シナリオカタログを変更せずに、共有シナリオへトランスポートを公開できます。
   ファクトリが、すべてのインスタンスで分離された認証情報または使い捨てサーバー、
   Gateway 状態、アーティファクトパスを所有すると宣言しない限り、同一チャンネルの
   パーティションは直列に実行されます。
5. テーマ別の `qa/scenarios/` ディレクトリ配下で YAML シナリオを作成または
   適応します。
6. 新しいシナリオでは汎用シナリオヘルパーを使用します。
7. リポジトリが意図的な移行を行っている場合を除き、既存の互換性
   エイリアスを機能させ続けます。

判断規則は厳格です。

- 動作を `qa-lab` で一度だけ表現できる場合は、`qa-lab` に配置します。
- 動作が 1 つのチャンネルトランスポートに依存する場合は、そのランナー
  Plugin または Plugin ハーネス内に保持します。
- 複数のチャンネルで使用できる新しい機能をシナリオが必要とする場合は、
  `suite.ts` にチャンネル固有の分岐を追加するのではなく、汎用ヘルパーを追加します。
- 動作が 1 つのトランスポートでのみ意味を持つ場合は、シナリオを
  トランスポート固有のままにし、そのことをシナリオ契約で明示します。

### シナリオヘルパー名

新しいシナリオで推奨される汎用ヘルパーは次のとおりです。

- `waitForTransportReady`
- `waitForChannelReady`
- `injectInboundMessage`
- `injectOutboundMessage`
- `waitForTransportOutboundMessage`
- `waitForChannelOutboundMessage`
- `waitForNoTransportOutbound`
- `getTransportSnapshot`
- `readTransportMessage`
- `readTransportTranscript`
- `formatTransportTranscript`
- `resetTransport`

既存のシナリオでは、互換性エイリアスである
`waitForQaChannelReady`、`waitForOutboundMessage`、`waitForNoOutbound`、
`formatConversationTranscript`、`resetBus` を引き続き使用できますが、新しいシナリオの
作成には汎用名を使用してください。これらのエイリアスは一斉移行を避けるために
存在するものであり、今後のモデルではありません。

## レポート

`qa-lab` は、観測されたバスタイムラインから Markdown プロトコルレポートを
エクスポートします。レポートでは次の問いに答える必要があります。

- 機能したもの
- 失敗したもの
- ブロックされたままのもの
- 追加する価値があるフォローアップシナリオ

利用可能なシナリオの一覧を確認するには、`pnpm openclaw qa coverage` を実行します
（機械可読出力には `--json` を追加します）。これは、フォローアップ作業の
規模を見積もる場合や、新しいトランスポートを接続する場合に役立ちます。変更した
動作またはファイルパスに対する対象を絞った検証を選択する場合は、
`pnpm openclaw qa coverage --match <query>` を実行します。照合レポートは、シナリオメタデータ、
ドキュメント参照、コード参照、カバレッジ ID、Plugin、プロバイダー要件を検索し、
一致する `qa suite
--scenario ...` ターゲットを出力します。

各 `qa suite` 実行では、選択したシナリオセットに対してトップレベルの
`qa-evidence.json`、`qa-suite-summary.json`、`qa-suite-report.md` アーティファクトが
書き込まれます。`execution.kind: vitest` または `execution.kind: playwright` を宣言するシナリオは、
対応するテストパスを実行し、シナリオごとのログも書き込みます。
`execution.kind: script` を宣言するシナリオは、`node --import tsx` を介して
`execution.path` のエビデンス生成プログラムを実行します
（`${outputDir}` と `${scenarioId}` は `execution.args` 内で展開されます）。
生成プログラムは独自の `qa-evidence.json` を書き込み、そのエントリはスイート出力に
インポートされ、アーティファクトパスはその生成プログラムの `qa-evidence.json` を
基準に解決されます。`qa run
--qa-profile` を介して `qa suite` に到達した場合、
同じ `qa-evidence.json` に、選択した分類カテゴリのプロファイルスコアカード概要も
含まれます。

カバレッジ出力は探索の補助として扱い、ゲートの代わりにはしないでください。
選択したシナリオには、テスト対象の動作に応じた適切なプロバイダーモード、
ライブトランスポート、Multipass、Testbox、またはリリースレーンが依然として必要です。
スコアカードの背景情報については、[成熟度スコアカード](/ja-JP/maturity/scorecard)を参照してください。

キャラクターとスタイルを検査するには、複数のライブモデル参照に対して同じシナリオを
実行し、判定済みの Markdown レポートを作成します。

```bash
pnpm openclaw qa character-eval \
  --model openai/gpt-5.6-luna,thinking=medium,fast \
  --model openai/gpt-5.2,thinking=xhigh \
  --model openai/gpt-5,thinking=xhigh \
  --model anthropic/claude-opus-4-8,thinking=high \
  --model anthropic/claude-sonnet-4-6,thinking=high \
  --model zai/glm-5.1,thinking=high \
  --model moonshot/kimi-k2.5,thinking=high \
  --model google/gemini-3.1-pro-preview,thinking=high \
  --judge-model openai/gpt-5.6-sol,thinking=xhigh,fast \
  --judge-model anthropic/claude-opus-4-8,thinking=high \
  --blind-judge-models \
  --concurrency 16 \
  --judge-concurrency 16
```

このコマンドは Docker ではなく、ローカル QA Gateway の子プロセスを実行します。
キャラクター評価シナリオでは、`SOUL.md` を介してペルソナを設定し、その後、
チャット、ワークスペースのヘルプ、小規模なファイルタスクなどの通常のユーザーターンを
実行してください。候補モデルには、評価中であることを伝えないでください。このコマンドは、
各トランスクリプト全体を保持し、基本的な実行統計を記録した後、対応している場合は
`xhigh` 推論を使用した高速モードで判定モデルに問い合わせ、自然さ、雰囲気、
ユーモアによって実行結果を順位付けします。プロバイダーを比較する場合は
`--blind-judge-models` を使用します。判定プロンプトには引き続きすべてのトランスクリプトと
実行ステータスが渡されますが、候補参照は `candidate-01` などの中立的なラベルに
置き換えられます。レポートでは、解析後にランキングを実際の参照へ対応付け直します。

候補の実行では、デフォルトで `high` thinking を使用し、GPT-5.6 Luna では
`medium`、対応している以前の OpenAI 評価参照では `xhigh` を
使用します。特定の候補を上書きするには、`--model provider/model,thinking=<level>` をインラインで指定します。
インラインオプションでは、`fast`、`no-fast`、
`fast=<bool>` もサポートされます。`--thinking
<level>` は引き続きグローバルな
フォールバックを設定し、以前の `--model-thinking
<provider/model=level>` 形式も互換性のために維持されます。
OpenAI の候補参照では、プロバイダーが対応している場合に優先処理が使用されるよう、
デフォルトで高速モードになります。すべての候補モデルで高速モードを強制的に有効に
したい場合のみ、`--fast` を渡してください。候補と判定の所要時間は
ベンチマーク分析用にレポートへ記録されますが、判定プロンプトでは速度によって
順位付けしないよう明示されます。候補モデルと判定モデルの実行は、どちらも
デフォルトで並行数 16 です。プロバイダーの制限またはローカル Gateway の負荷によって
実行時のノイズが多すぎる場合は、`--concurrency` または `--judge-concurrency` を
引き下げてください。

候補の `--model` が渡されない場合、キャラクター評価ではデフォルトで
`openai/gpt-5.6-luna`、`openai/gpt-5.2`、`openai/gpt-5`、
`anthropic/claude-opus-4-8`、`anthropic/claude-sonnet-4-6`、`zai/glm-5.1`、
`moonshot/kimi-k2.5`、`google/gemini-3.1-pro-preview` を使用します。
`--judge-model` が渡されない場合、判定モデルのデフォルトは
`openai/gpt-5.6-sol,thinking=xhigh,fast` と
`anthropic/claude-opus-4-8,thinking=high` です。

## 関連ドキュメント

- [成熟度スコアカード](/ja-JP/maturity/scorecard)
- [パーソナルエージェントベンチマークパック](/ja-JP/concepts/personal-agent-benchmark-pack)
- [QA チャンネル](/ja-JP/channels/qa-channel)
- [テスト](/ja-JP/help/testing)
- [ダッシュボード](/ja-JP/web/dashboard)
