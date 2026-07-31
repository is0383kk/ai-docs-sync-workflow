---
read_when:
    - ローカルまたは CI でテストを実行する
    - モデル／プロバイダーのバグに対する回帰テストの追加
    - Gateway とエージェントの動作のデバッグ
summary: テストキット：ユニット／E2E／ライブスイート、Docker ランナー、および各テストの対象範囲
title: テスト
x-i18n:
    generated_at: "2026-07-26T09:44:08Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 20e0aa22bf16561334f83342abffabb387ed0b41b901773939123ecfbc0ae330
    source_path: help/testing.md
    workflow: 16
---

OpenClaw には、3 つの Vitest スイート（unit/integration、e2e、live）と Docker
ランナーがあります。このページでは、各スイートの対象範囲、各ワークフローで実行する
コマンド、live テストによる認証情報の検出方法、および実際の provider/model のバグに対する
回帰テストの追加方法について説明します。

<Note>
**QA スタック（qa-lab、qa-channel、live transport レーン）**については、別途ドキュメント化されています。

- [QA の概要](/ja-JP/concepts/qa-e2e-automation) - アーキテクチャ、コマンドサーフェス、シナリオ作成、および Matrix プロファイル。
- [成熟度スコアカード](/ja-JP/maturity/scorecard) - リリース QA のエビデンスが安定性と LTS の判断をどのように支えるか。
- [QA チャネル](/ja-JP/channels/qa-channel) - リポジトリに基づくシナリオで使用される合成 transport plugin。

このページでは、通常のテストスイートと Docker/Parallels ランナーについて説明します。以下の [QA 専用ランナー](#qa-specific-runners)では、具体的な `qa` の呼び出しを一覧にし、上記のリファレンスを参照します。
</Note>

## クイックスタート

通常は次のとおりです。

- フルゲート（push 前に実行することを想定）: `pnpm build && pnpm check && pnpm check:test-types && pnpm test`
- 十分なリソースがあるマシンで高速にローカルのフルスイートを実行: `pnpm test:max`
- Vitest の watch ループを直接実行: `pnpm test:watch`
- ファイルを直接指定すると、plugin/channel のパスにもルーティングされます: `pnpm test extensions/discord/src/monitor/message-handler.preflight.test.ts`
- 単一の失敗を反復修正する場合は、まず対象を絞った実行を優先してください。
- Docker ベースの QA サイト: `pnpm qa:lab:up`
- Linux VM ベースの QA レーン: `pnpm openclaw qa suite --runner multipass --scenario channel-chat-baseline`

テストを変更した場合や、確信をさらに高めたい場合:

- 参考情報としての V8 カバレッジレポート: `pnpm test:coverage`
- E2E スイート: `pnpm test:e2e`

## テスト用一時ディレクトリ

テストが所有する一時ディレクトリには、`test/helpers/temp-dir.ts` の共有ヘルパーを
使用してください。これにより所有権が明示され、クリーンアップがテストのライフサイクル内に維持されます。

```ts
import { afterEach } from "vitest";
import { useAutoCleanupTempDirTracker } from "../helpers/temp-dir.js";

const tempDirs = useAutoCleanupTempDirTracker(afterEach);

it("一時ワークスペースを使用する", () => {
  const workspace = tempDirs.make("openclaw-example-");
  // workspace を使用
});
```

`useAutoCleanupTempDirTracker(afterEach)` は意図的に手動の
クリーンアップメソッドを公開していません。各テスト後のクリーンアップは Vitest が所有します。移行されていない
テスト向けに、古い低レベルのヘルパー（`makeTempDir`、`cleanupTempDirs`、`createTempDirTracker`）も引き続き存在しますが、
新たな使用は避けてください。また、raw temp-dir の
動作をテストで明示的に検証する場合を除き、新たに `fs.mkdtemp*` を直接呼び出すことも避けてください。
一時ディレクトリを直接作成する必要が本当にある場合は、理由を付けて監査可能な許可
コメントを追加してください。

```ts
// openclaw-temp-dir: allow raw fs のクリーンアップ動作を検証
const workspace = fs.mkdtempSync(prefix);
```

`node scripts/report-test-temp-creations.mjs` は、追加された diff 行における新しい一時ディレクトリの直接作成と
共有ヘルパーの新しい手動使用を報告しますが、
既存のクリーンアップ形式はブロックしません。`scripts/changed-lanes.mjs` と同じテストパス分類に従い、
共有ヘルパーの実装自体はスキップします。`check:changed` は、変更されたテストパスに対してこのレポートを
警告のみの CI シグナル（失敗ではなく GitHub の警告アノテーション）として実行します。

## Live および Docker/Parallels ワークフロー

実際の provider/model をデバッグする場合（実際の認証情報が必要）:

- Live スイート（model + gateway tool/image プローブ）: `pnpm test:live`
- 単一の live ファイルを静かに対象指定: `pnpm test:live -- src/agents/models.profiles.live.test.ts`
- ランタイムパフォーマンスレポート: 実際の `openai/gpt-5.6-luna` agent ターンには
  `live_openai_candidate=true` を指定して `OpenClaw Performance` を dispatch するか、
  Kova の CPU/heap/trace アーティファクトには `deep_profile=true` を指定します。毎日のスケジュール実行では、
  mock-provider、deep-profile、および GPT-5.6 Luna レーンのレポートが、
  別のアーティファクト消費 publisher ジョブから `openclaw/clawgrit-reports` に公開されます。
  publisher 認証が欠落しているか無効な場合、スケジュール実行と
  `profile=release` の実行は失敗します。リリース以外の手動 dispatch では GitHub アーティファクトを
  保持し、レポート公開を参考情報として扱います。mock-provider レポートには、
  ソースレベルの gateway 起動、メモリ、plugin 負荷、反復される
  fake-model hello-loop、および CLI 起動の数値も含まれます。
- Docker live model スイープ: `pnpm test:docker:live-models`
  - 選択された各 model では、テキストターンに加えて、小さなファイル読み取り形式のプローブを実行します。
    メタデータで `image` 入力を示す model では、小さな画像ターンも実行します。
    provider の失敗を切り分ける場合は、`OPENCLAW_LIVE_MODEL_FILE_PROBE=0` または
    `OPENCLAW_LIVE_MODEL_IMAGE_PROBE=0` で追加プローブを無効にしてください。
  - CI カバレッジ: 毎日の `OpenClaw Scheduled Live And E2E Checks` と手動の
    `OpenClaw Release Checks` は、どちらも `include_live_suites: true` を指定して再利用可能な live/E2E ワークフローを呼び出します。
    これには provider ごとに shard 化された Docker live model matrix ジョブが含まれます。
  - 対象を絞った CI の再実行では、`include_live_suites: true` と
    `live_models_only: true` を指定して `OpenClaw Live And E2E Checks (Reusable)` を dispatch してください。
  - シグナルの強い新しい provider secret は `scripts/ci-hydrate-live-auth.sh` に加え、
    `.github/workflows/openclaw-live-and-e2e-checks-reusable.yml` とその
    schedule/release 呼び出し元にも追加してください。
- ネイティブ Codex bound-chat スモークテスト: `pnpm test:docker:live-codex-bind`
  - Codex app-server パスに対して Docker live レーンを実行し、`/codex bind` で
    合成 Slack DM を bind し、`/codex fast` と
    `/codex permissions` を実行した後、通常の返信と画像添付が
    ACP ではなくネイティブ plugin binding を経由することを検証します。
- Codex app-server harness スモークテスト: `pnpm test:docker:live-codex-harness`
  - plugin が所有する Codex app-server
    harness を通じて gateway agent ターンを実行し、`/codex status` と `/codex models` を検証します。デフォルトでは、
    image、cron MCP、sub-agent、および Guardian の各プローブも実行します。他の失敗を
    切り分ける場合は、`OPENCLAW_LIVE_CODEX_HARNESS_SUBAGENT_PROBE=0` で
    sub-agent プローブを無効にしてください。sub-agent の確認に対象を絞る場合は、
    他のプローブを無効にします:
    `OPENCLAW_LIVE_CODEX_HARNESS_IMAGE_PROBE=0 OPENCLAW_LIVE_CODEX_HARNESS_MCP_PROBE=0 OPENCLAW_LIVE_CODEX_HARNESS_GUARDIAN_PROBE=0 OPENCLAW_LIVE_CODEX_HARNESS_SUBAGENT_PROBE=1 pnpm test:docker:live-codex-harness`。
    `OPENCLAW_LIVE_CODEX_HARNESS_SUBAGENT_ONLY=0` が設定されていない限り、
    sub-agent プローブの後に終了します。
- Codex オンデマンドインストールのスモークテスト: `pnpm test:docker:codex-on-demand`
  - パッケージ化された OpenClaw tarball を Docker にインストールし、OpenAI API キーによる
    オンボーディングを実行して、Codex plugin と `@openai/codex` dependency が
    オンデマンドで管理対象 npm project root にダウンロードされたことを検証します。
- Codex npm-plugin live package スモークテスト: `pnpm test:docker:live-codex-npm-plugin`
  - 候補の OpenClaw package と正確な Codex plugin を Docker にインストールし、
    実際の OpenAI キーを使用して CLI preflight と同一セッションのターンを実行します。
  - retry なし、medium-thinking の後続ターンでは、進捗を送信し、ランダム化された
    workspace の読み取りと正確なアーティファクト書き込みを通じて作業を続行した後、
    完了を送信する必要があります。進捗のみで終了するターンはレーンを失敗させます。
- Live plugin tool dependency スモークテスト: `pnpm test:docker:live-plugin-tool`
  - 実際の `slugify` dependency を持つ fixture plugin を pack し、`npm-pack:` を通じて
    インストールし、管理対象 npm project root 配下の dependency を検証した後、
    live OpenAI model に plugin tool を呼び出して非公開の slug を返すよう要求します。
- OpenClaw rescue command スモークテスト: `pnpm test:live:system-agent-rescue-channel`
  - message-channel の rescue command
    サーフェスに対する、オプトインの念入りな確認です。`/openclaw status` を実行し、永続的な model
    変更をキューに入れ、`/openclaw yes` と返信して、audit/config の書き込み
    パスを検証します。
- OpenClaw 初回実行 Docker スモークテスト: `pnpm test:docker:system-agent-first-run`
  - 空の OpenClaw state dir から開始し、まずパッケージ化された
    `openclaw setup` CLI が推論なしで fail closed になることを確認します。その後、
    パッケージ化された activation module を通じて fake Claude をテストし、有効化します。
    その後にのみ、曖昧なパッケージ済み CLI リクエストが planner に到達して
    typed setup に解決され、続いて one-shot の model、agent、Discord config、
    および SecretRef の操作が行われます。config と audit entry を検証します。これは
    gate/operation を補足するエビデンスであり、対話的なオンボーディングや
    OpenClaw agent/tool/approval の証明ではありません。同じレーンは QA Lab でも
    `pnpm openclaw qa suite --scenario system-agent-ring-zero-setup` によって公開されています。
- Moonshot/Kimi コストスモークテスト: `MOONSHOT_API_KEY` を設定して
  `openclaw models list --provider moonshot --json` を実行し、その後、`moonshot/kimi-k2.6` に対して分離された
  `openclaw agent --local --session-id live-kimi-cost --message 'Reply exactly: KIMI_LIVE_OK' --thinking off --json`
  を実行します。JSON が Moonshot/K2.6 を報告し、assistant transcript に正規化された
  `usage.cost` が保存されることを検証します。

<Tip>
失敗しているケースが 1 つだけ必要な場合は、以下で説明する allowlist の環境変数を使って live テストの対象を絞ることを推奨します。
</Tip>

## QA 専用ランナー

これらのコマンドは、QA-lab の現実性が必要な場合に、メインのテストスイートと併用します。

CI は専用ワークフローで QA Lab を実行します。Agentic parity は、
単独の PR ワークフローではなく、`QA-Lab - All Lanes` とリリース検証の下にネストされています。
広範な検証では、`rerun_group=qa-parity` を指定した
`Full Release Validation`、または release-checks の QA グループを使用してください。stable/default のリリース
チェックでは、網羅的な live/Docker soak は `run_release_soak=true` の背後に置かれ、
`full` プロファイルでは soak が強制的に有効になります。`QA-Lab - All Lanes` は `main` で毎晩実行され、
手動 dispatch からは mock parity レーン、live Matrix レーン、
Convex 管理の live Telegram レーン、Convex 管理の live Discord レーンが
並列ジョブとして実行されます。スケジュールされた QA とリリースチェックは、
共有 live adapter を通じて Matrix release プロファイルを実行します。Matrix CLI と手動ワークフロー入力の
デフォルトは引き続き `all` です。手動の `all` dispatch では transport、media、
E2EE プロファイルに fan out し、対象を絞った dispatch では `fast`、`release`、または
`transport` を選択できます。`OpenClaw Release Checks` は、リリース承認前に parity に加えて、再利用可能な Matrix
live-adapter プロファイルと Telegram レーンを実行します。リリースの
transport チェックでは `mock-openai/gpt-5.6-luna` を使用するため、決定論的な動作を維持し、
通常の provider-plugin の起動を回避できます。これらの live transport gateway では
memory search が無効化されています。memory の動作は QA parity スイートで引き続きカバーされます。

フルリリースの live media shard では、
`ghcr.io/openclaw/openclaw-live-media-runner:ubuntu-24.04` を使用します。これには
`ffmpeg` と `ffprobe` がすでに含まれています。Docker live model/backend shard では、選択した
commit ごとに 1 回だけビルドされる共有 `ghcr.io/openclaw/openclaw-live-test:<sha>` image を使用し、
各 shard 内で再ビルドする代わりに `OPENCLAW_SKIP_DOCKER_BUILD=1` で pull します。

- `pnpm openclaw qa suite`
  - リポジトリに基づく QA シナリオをホスト上で直接実行します。
  - 選択したシナリオセットについて、混合フロー、Vitest、Playwright のシナリオ選択を含むトップレベルの `qa-evidence.json`、`qa-suite-summary.json`、および
    `qa-suite-report.md` アーティファクトを書き込みます。
  - `pnpm openclaw qa run --qa-profile <profile>` によってディスパッチされた場合、選択した分類プロファイルのスコアカードを同じ `qa-evidence.json` に埋め込みます。
    `smoke-ci` は簡略化されたエビデンス（`evidenceMode: "slim"`、エントリごとの
    `execution` なし）を書き込みます。`release` は厳選されたリリース準備状況の範囲を対象とし、`all`
    はすべてのアクティブな成熟度カテゴリを選択し、完全なスコアカードアーティファクトが必要な場合に明示的な QA Profile
    Evidence ワークフローディスパッチを対象とします。
  - デフォルトでは、分離された Gateway ワーカーを使用して、選択した複数のシナリオを並列実行します。
    `qa-channel` のデフォルト同時実行数は 4（選択したシナリオ数が上限）です。ワーカー数を調整するには `--concurrency <count>` を、
    従来の直列レーンには `--concurrency 1` を使用します。
  - いずれかのシナリオが失敗すると、ゼロ以外の終了コードで終了します。終了コードを失敗にせずアーティファクトを取得するには `--allow-failures` を使用します。
  - プロバイダーモード `live-frontier`、`mock-openai`、および `aimock` をサポートします。
    `aimock` は、シナリオ対応の
    `mock-openai` レーンを置き換えることなく、実験的なフィクスチャおよびプロトコルモックのカバレッジ用に、ローカルの AIMock ベースのプロバイダーサーバーを起動します。
- `pnpm openclaw qa coverage --match <query>`
  - シナリオ ID、タイトル、サーフェス、カバレッジ ID、ドキュメント参照、コード参照、Plugin、プロバイダー要件を検索し、一致するスイートターゲットを出力します。
  - 変更対象の動作やファイルパスは分かっていても、最小のシナリオが分からない場合は、QA Lab の実行前にこれを使用します。これは参考情報にすぎません。変更する動作に応じて、モック、ライブ、Multipass、Matrix、またはトランスポートの検証を引き続き選択してください。
- `pnpm test:plugins:kitchen-sink-live`
  - ライブの OpenAI Kitchen Sink Plugin 総合試験を QA Lab 経由で実行します。
    外部の Kitchen Sink パッケージをインストールし、Plugin SDK のサーフェスインベントリを検証し、`/healthz` と `/readyz` をプローブし、Gateway
    の CPU/RSS エビデンスを記録し、ライブの OpenAI ターンを実行して、敵対的診断を確認します。`OPENAI_API_KEY` などのライブ OpenAI 認証が必要です。
    ハイドレート済みの Testbox セッションでは、`openclaw-testbox-env` ヘルパーが存在する場合、Testbox のライブ認証プロファイルを自動的に読み込みます。
- `pnpm test:gateway:cpu-scenarios`
  - Gateway 起動ベンチと小規模なモック QA Lab シナリオパック
    （`channel-chat-baseline`、`memory-failure-fallback`、
    `gateway-restart-inflight-run`）を実行し、`.artifacts/gateway-cpu-scenarios/` 配下に CPU 観測結果をまとめたサマリーを書き込みます。
  - デフォルトでは、持続的な高 CPU 観測のみをフラグします（`--cpu-core-warn`、
    デフォルト `0.9`、`--hot-wall-warn-ms`、デフォルト `30000`）。そのため、短い起動時のバーストは、数分間続く
    Gateway の CPU 張り付きリグレッションのように見せることなく、メトリクスとして記録されます。
  - ビルド済みの `dist` アーティファクトに対して実行します。チェックアウトに新しいランタイム出力がまだない場合は、先にビルドを実行してください。
- `pnpm openclaw qa suite --runner multipass`
  - 同じ QA スイートを使い捨ての Multipass Linux VM 内で実行し、`qa suite` と同じシナリオ選択およびプロバイダー／モデルフラグを維持します。
  - ライブ実行では、ゲストで実用可能な QA 認証入力を転送します。
    これには、環境変数ベースのプロバイダーキー、QA ライブプロバイダー設定パス、および存在する場合の
    `CODEX_HOME` が含まれます。
  - ゲストがマウントされたワークスペースを通じて書き戻せるように、出力ディレクトリはリポジトリルート配下に置く必要があります。
  - 通常の QA レポートとサマリーに加えて、`.artifacts/qa-e2e/...` 配下に Multipass ログを書き込みます。
- `pnpm qa:lab:up`
  - オペレーター形式の QA 作業用に、Docker ベースの QA サイトを起動します。
- `pnpm test:docker:npm-onboard-channel-agent`
  - 現在のチェックアウトから npm tarball をビルドし、Docker にグローバルインストールして、非対話型の OpenAI API キーのオンボーディングを実行し、デフォルトで
    Telegram を設定します。さらに、パッケージ化された Plugin ランタイムが起動時の依存関係修復なしで読み込まれることを検証し、doctor を実行して、モックされた OpenAI エンドポイントに対してローカルエージェントのターンを 1 回実行します。
  - 同じパッケージインストールレーンを Discord で実行するには、`OPENCLAW_NPM_ONBOARD_CHANNEL=discord` を使用します。
- `pnpm test:docker:session-runtime-context`
  - 埋め込みランタイムコンテキストのトランスクリプトについて、決定論的なビルド済みアプリの Docker スモークテストを実行します。非表示の OpenClaw ランタイムコンテキストが、表示されるユーザーターンに漏れるのではなく、非表示のカスタムメッセージとして保持されることを検証します。次に、影響を受ける破損したセッション JSONL をシードし、
    `openclaw doctor --fix` がバックアップを作成してアクティブなブランチに書き換えることを検証します。
- `pnpm test:docker:npm-telegram-live`
  - OpenClaw パッケージ候補を Docker にインストールし、インストール済みパッケージのオンボーディングを実行し、インストール済み CLI を介して Telegram を設定します。その後、そのインストール済みパッケージを SUT
    Gateway として使用し、ライブ Telegram QA レーンを再利用します。
  - ラッパーはチェックアウトから `qa-lab` ハーネスのソースのみをマウントします。
    インストール済みパッケージが `dist`、`openclaw/plugin-sdk`、およびバンドルされた
    Plugin ランタイムを所有するため、このレーンでは現在のチェックアウトの Plugin がテスト対象パッケージに混在しません。
  - デフォルトは `OPENCLAW_NPM_TELEGRAM_PACKAGE_SPEC=openclaw@beta` です。レジストリからインストールする代わりに、解決済みのローカル tarball をテストするには、
    `OPENCLAW_NPM_TELEGRAM_PACKAGE_TGZ=/path/to/openclaw-current.tgz` または
    `OPENCLAW_CURRENT_PACKAGE_TGZ` を設定します。
  - デフォルトでは、`OPENCLAW_NPM_TELEGRAM_RTT_SAMPLES=20` を使用して `qa-evidence.json` に反復 RTT タイミングを出力します。
    実行を調整するには、
    `OPENCLAW_NPM_TELEGRAM_RTT_SAMPLES`、
    `OPENCLAW_NPM_TELEGRAM_RTT_TIMEOUT_MS`、または
    `OPENCLAW_NPM_TELEGRAM_RTT_MAX_FAILURES` を上書きします。
    `OPENCLAW_NPM_TELEGRAM_RTT_CHECKS` はサンプリングする Telegram QA シナリオを選択します。サポートされる RTT ターゲットは `channel-canary` です。
  - `pnpm openclaw qa telegram` と同じ Telegram 環境認証情報または Convex 認証情報ソースを使用します。CI／リリース自動化では、
    `OPENCLAW_NPM_TELEGRAM_CREDENTIAL_SOURCE=convex` に加えて
    `OPENCLAW_QA_CONVEX_SITE_URL` とロールシークレットを設定します。
    CI に `OPENCLAW_QA_CONVEX_SITE_URL` と Convex ロールシークレットが存在する場合、Docker ラッパーは Convex を自動的に選択します。
  - ラッパーは、Docker のビルド／インストール作業前に、ホスト上で Telegram または Convex の認証情報環境変数を検証します。
    認証情報設定前のデバッグを意図的に行う場合にのみ、
    `OPENCLAW_NPM_TELEGRAM_SKIP_CREDENTIAL_PREFLIGHT=1` を設定してください。
  - `OPENCLAW_NPM_TELEGRAM_CREDENTIAL_ROLE=ci|maintainer` は、このレーンに限り共有の
    `OPENCLAW_QA_CREDENTIAL_ROLE` を上書きします。Convex
    認証情報が選択され、ロールが設定されていない場合、ラッパーは CI 内では `ci`、CI 外では `maintainer` を使用します。
  - GitHub Actions では、このレーンを手動のメンテナーワークフロー
    `NPM Telegram Beta E2E` として提供します。マージ時には実行されません。このワークフローは
    `qa-live-shared` 環境と Convex CI 認証情報リースを使用します。
- GitHub Actions では、単一の候補パッケージに対するサイド実行の製品検証用に `Package Acceptance` も提供します。Git ref、公開済み npm 仕様、
  HTTPS tarball URL と SHA-256、信頼済み URL ポリシー、または別の実行からの tarball アーティファクト
  （`source=ref|npm|url|trusted-url|artifact`）を受け付け、正規化した
  `openclaw-current.tgz` を `package-under-test` としてアップロードします。その後、`smoke`、`package`、`product`、`full`、
  または `custom` のレーンプロファイルで既存の Docker E2E スケジューラーを実行します。同じ
  `package-under-test` アーティファクトに対して Telegram QA ワークフローを実行するには、`telegram_mode=mock-openai` または
  `live-frontier` を設定します。
  - 最新ベータ版の製品検証：

```bash
gh workflow run package-acceptance.yml --ref main \
  -f source=npm \
  -f package_spec=openclaw@beta \
  -f suite_profile=product \
  -f telegram_mode=mock-openai
```

- 正確な tarball URL の検証にはダイジェストが必要で、公開 URL の安全性ポリシーを使用します。

```bash
gh workflow run package-acceptance.yml --ref main \
  -f source=url \
  -f package_url=https://registry.npmjs.org/openclaw/-/openclaw-VERSION.tgz \
  -f package_sha256=<sha256> \
  -f suite_profile=package
```

- 企業／プライベート tarball ミラーでは、明示的な信頼済みソースポリシーを使用します。

```bash
gh workflow run package-acceptance.yml --ref main \
  -f source=trusted-url \
  -f trusted_source_id=enterprise-artifactory \
  -f package_url=https://packages.example.internal:8443/artifactory/openclaw/openclaw-VERSION.tgz \
  -f package_sha256=<sha256> \
  -f suite_profile=package
```

`source=trusted-url` は信頼済みワークフロー ref から `.github/package-trusted-sources.json` を読み取り、URL 認証情報やワークフロー入力によるプライベートネットワークのバイパスを受け付けません。指定されたポリシーで bearer 認証が宣言されている場合は、固定の `OPENCLAW_TRUSTED_PACKAGE_TOKEN` シークレットを設定します。

- アーティファクト検証では、別の Actions 実行から tarball アーティファクトをダウンロードします。

```bash
gh workflow run package-acceptance.yml --ref main \
  -f source=artifact \
  -f artifact_run_id=<run-id> \
  -f artifact_name=<artifact-name> \
  -f suite_profile=smoke
```

- `pnpm test:docker:plugins`
  - 現在の OpenClaw ビルドを Docker 内でパックしてインストールし、OpenAI を設定して
    Gateway を起動した後、設定の編集によってバンドルされたチャンネル／Plugin を有効にします。
  - セットアップ検出で、未設定のダウンロード可能な Plugin が存在しないままであること、最初の設定済み doctor 修復で欠落しているダウンロード可能な各 Plugin が明示的にインストールされること、および 2 回目の再起動では非表示の依存関係修復が実行されないことを検証します。
  - また、既知の古い npm ベースラインをインストールし、`openclaw update --tag <candidate>` を実行する前に Telegram を有効化し、候補版の更新後 doctor が、ハーネス側の postinstall 修復なしで従来の Plugin 依存関係の残骸を除去することを検証します。
- `pnpm test:parallels:npm-update`
  - Parallels ゲスト全体で、ネイティブのパッケージインストール更新スモークテストを実行します。
    選択した各プラットフォームでは、まず要求されたベースラインパッケージをインストールし、次に同じゲスト内でインストール済みの `openclaw update` コマンドを実行して、インストール済みバージョン、更新状態、Gateway の準備完了状態、およびローカルエージェントのターン 1 回を検証します。
  - 1 つのゲストで反復作業を行う場合は、`--platform macos`、`--platform windows`、または `--platform linux` を使用します。
    サマリーアーティファクトのパスとレーンごとの状態には `--json` を使用します。
  - OpenAI レーンでは、デフォルトでライブエージェントターンの検証に `openai/gpt-5.6-luna` を使用します。
    別の OpenAI モデルを検証するには、`--model <provider/model>` を渡すか、
    `OPENCLAW_PARALLELS_OPENAI_MODEL` を設定します。
  - Parallels のトランスポート停止によって残りのテスト時間が消費されないように、長時間のローカル実行はホスト側のタイムアウトで囲みます。

    ```bash
    timeout --foreground 150m pnpm test:parallels:npm-update -- --json
    timeout --foreground 90m pnpm test:parallels:npm-update -- --platform windows --json
    ```

  - スクリプトは、`/tmp/openclaw-parallels-npm-update.*` 配下にネストされたレーンログを書き込みます。
    外側のラッパーがハングしていると判断する前に、`windows-update.log`、
    `macos-update.log`、または `linux-update.log` を確認してください。
  - コールド状態のゲストでは、Windows の更新時に、更新後の doctor とパッケージ更新処理に 10～15 分かかることがあります。ネストされた npm デバッグログが進行していれば、正常な状態です。
  - この集約ラッパーを、個別の Parallels
    macOS、Windows、または Linux スモークレーンと並列実行しないでください。これらは VM 状態を共有するため、スナップショットの復元、パッケージの提供、またはゲストの Gateway 状態で競合する可能性があります。
  - 更新後の検証では、音声、画像生成、メディア理解などの機能ファサードが、エージェントターン自体では単純なテキスト応答のみを確認する場合でも、バンドルされたランタイム API を通じて読み込まれるため、通常のバンドル Plugin サーフェスを実行します。

- `pnpm openclaw qa aimock`
  - 直接プロトコルのスモークテスト用に、ローカルの AIMock プロバイダーサーバーのみを
    起動します。
- `pnpm openclaw qa matrix`
  - 使い捨ての Docker バックエンド Tuwunel ホームサーバーに対して、Matrix のライブ QA レーンを
    実行します。ソースチェックアウト専用です。パッケージ版インストールには
    `qa-lab` は含まれません。
  - CLI 全体、プロファイル／シナリオカタログ、環境変数、アーティファクトのレイアウト：
    [Matrix スモークレーン](/ja-JP/concepts/qa-e2e-automation#matrix-smoke-lanes)。
- `pnpm openclaw qa telegram`
  - 環境変数のドライバーおよび SUT ボットトークンを使用して、実際のプライベートグループに対して
    Telegram のライブ QA レーンを実行します。
  - `OPENCLAW_QA_TELEGRAM_GROUP_ID`、
    `OPENCLAW_QA_TELEGRAM_DRIVER_BOT_TOKEN`、および
    `OPENCLAW_QA_TELEGRAM_SUT_BOT_TOKEN` が必要です。グループ ID は数値の
    Telegram チャット ID でなければなりません。
  - 共有プール認証情報用の `--credential-source convex` をサポートします。
    デフォルトでは環境変数モードを使用するか、`OPENCLAW_QA_CREDENTIAL_SOURCE=convex` を設定して
    プールされたリースを明示的に有効にします。
  - デフォルトでは、カナリア、メンションゲーティング、コマンドアドレッシング、`/status`、
    ボット間のメンション付き返信、およびコアのネイティブコマンド返信を対象とします。
    `mock-openai` のデフォルトでは、決定論的な返信チェーンと
    Telegram の最終メッセージストリーミングのリグレッションも対象とします。`session_status` などの
    オプションプローブには `--list-scenarios` を使用します。
  - いずれかのシナリオが失敗すると、ゼロ以外で終了します。失敗を示す終了コードを返さずに
    アーティファクトを生成するには、`--allow-failures` を使用します。
  - 同じプライベートグループ内にある 2 つの異なるボットが必要で、SUT ボットには
    Telegram ユーザー名が公開されている必要があります。
  - ボット間の観測を安定させるには、両方のボットについて
    `@BotFather` で Bot-to-Bot Communication Mode を有効にし、ドライバーボットが
    グループ内のボットトラフィックを観測できることを確認します。
  - Telegram QA レポート、サマリー、および `qa-evidence.json` を
    `.artifacts/qa-e2e/...` の下に書き込みます。返信シナリオには、ドライバーの送信要求から
    観測された SUT の返信までの RTT が含まれます。

`Mantis Telegram Live` は、このレーンを囲む PR エビデンスラッパーです。Convex からリースした
Telegram 認証情報を使用して候補 ref を実行し、編集済みの QA レポート／エビデンスバンドルを
Crabbox デスクトップブラウザーでレンダリングし、MP4 エビデンスを記録し、
動きに合わせてトリミングした GIF を生成してアーティファクトバンドルをアップロードし、`pr_number` が
設定されている場合は Mantis GitHub App を通じてインラインの PR エビデンスを投稿します。メンテナーは Actions UI から `Mantis Scenario`
（`scenario_id: telegram-live`）を使用するか、プルリクエストのコメントから直接開始できます：

```text
@openclaw-mantis telegram
@openclaw-mantis telegram scenario=telegram-status-command
@openclaw-mantis telegram scenarios=telegram-status-command,channel-canary
```

`Mantis Telegram Desktop Proof` は、PR の視覚的証明用のエージェント型ネイティブ Telegram Desktop
変更前／変更後ラッパーです。Actions UI で自由形式の `instructions` を使用するか、
`Mantis Scenario`（`scenario_id:
telegram-desktop-proof`）を通じて、または PR コメントから開始します：

```text
@openclaw-mantis telegram desktop proof
```

Mantis エージェントは PR を読み、変更を証明する Telegram 上で可視の動作を判断し、
ベースラインおよび候補 ref に対して実ユーザーの Crabbox Telegram Desktop 証明レーンを実行し、
ネイティブ GIF が有用になるまで反復し、ペア構成の `motionPreview` マニフェストを書き込み、`pr_number` が設定されている場合は
Mantis GitHub App を通じて同じ 2 列の GIF テーブルを投稿します。

- `pnpm openclaw qa mantis telegram-desktop-builder`
  - Crabbox Linux デスクトップをリースまたは再利用し、ネイティブの Telegram
    Desktop をインストールし、リースされた Telegram SUT ボットトークンで OpenClaw を設定し、
    Gateway を起動して、表示中の VNC デスクトップからスクリーンショット／MP4 エビデンスを
    記録します。
  - デフォルトは `--credential-source convex` のため、ワークフローに必要なのは
    Convex ブローカーシークレットのみです。`pnpm openclaw qa telegram` と同じ
    `OPENCLAW_QA_TELEGRAM_*` 変数を使用する `--credential-source env` を使用します。
  - Telegram Desktop には引き続きユーザーログイン／プロファイルが必要です。ボットトークンで
    設定されるのは OpenClaw のみです。base64 の `.tgz` プロファイルアーカイブには
    `--telegram-profile-archive-env <name>` を使用するか、`--keep-lease` を使用して
    VNC から一度手動でログインします。
  - 出力ディレクトリの下に `mantis-telegram-desktop-builder-report.md`、
    `mantis-telegram-desktop-builder-summary.json`、
    `telegram-desktop-builder.png`、および `telegram-desktop-builder.mp4`
    を書き込みます。

新しいトランスポート間で差異が生じないように、ライブトランスポートレーンは 1 つの標準契約を共有します。
レーンごとのカバレッジマトリクスについては、
[QA の概要 - ライブトランスポートのカバレッジ](/ja-JP/concepts/qa-e2e-automation#live-transport-coverage)を参照してください。
`qa-channel` は広範な合成テストスイートであり、このマトリクスには含まれません。

### Convex 経由の共有 Telegram 認証情報（v1）

ライブトランスポート QA で `--credential-source convex`（または `OPENCLAW_QA_CREDENTIAL_SOURCE=convex`）
を有効にすると、QA ラボは Convex バックエンドのプールから排他的リースを取得し、
レーンの実行中はそのリースに Heartbeat を送信し、シャットダウン時にリースを
解放します。このセクション名は Discord、Slack、WhatsApp のサポートより前から存在しますが、
リース契約は種類間で共有されています。

参照用 Convex プロジェクトのスキャフォールド：`qa/convex-credential-broker/`

必須の環境変数：

- `OPENCLAW_QA_CONVEX_SITE_URL`（例：`https://your-deployment.convex.site`）
- 選択したロール用のシークレット 1 つ：
  - `maintainer` 用の `OPENCLAW_QA_CONVEX_SECRET_MAINTAINER`
  - `ci` 用の `OPENCLAW_QA_CONVEX_SECRET_CI`
- 認証情報ロールの選択：
  - CLI：`--credential-role maintainer|ci`
  - 環境変数のデフォルト：`OPENCLAW_QA_CREDENTIAL_ROLE`（CI ではデフォルトが `ci`、それ以外では `maintainer`）

オプションの環境変数：

- `OPENCLAW_QA_CREDENTIAL_LEASE_TTL_MS`（デフォルト：`1200000`）
- `OPENCLAW_QA_CREDENTIAL_HEARTBEAT_INTERVAL_MS`（デフォルト：`30000`）
- `OPENCLAW_QA_CREDENTIAL_ACQUIRE_TIMEOUT_MS`（デフォルト：`90000`）
- `OPENCLAW_QA_CREDENTIAL_HTTP_TIMEOUT_MS`（デフォルト：`15000`）
- `OPENCLAW_QA_CONVEX_ENDPOINT_PREFIX`（デフォルト：`/qa-credentials/v1`）
- `OPENCLAW_QA_CREDENTIAL_OWNER_ID`（オプションのトレース ID）
- `OPENCLAW_QA_ALLOW_INSECURE_HTTP=1` は、ローカル専用の開発で loopback `http://` Convex URL を許可します。

通常運用では、`OPENCLAW_QA_CONVEX_SITE_URL` に `https://` を使用する必要があります。

メンテナー向け管理コマンド（プールの追加／削除／一覧表示）には、
`OPENCLAW_QA_CONVEX_SECRET_MAINTAINER` が明示的に必要です。

メンテナー向け CLI ヘルパー：

```bash
pnpm openclaw qa credentials doctor
pnpm openclaw qa credentials add --kind telegram --payload-file qa/telegram-credential.json
pnpm openclaw qa credentials list --kind telegram
pnpm openclaw qa credentials remove --credential-id <credential-id>
```

ライブ実行前に `doctor` を使用して、シークレット値を出力せずに Convex サイト URL、ブローカーシークレット、
エンドポイントプレフィックス、HTTP タイムアウト、および管理／一覧表示への到達性を確認します。
スクリプトや CI ユーティリティで機械可読な出力を得るには、`--json` を使用します。

デフォルトのエンドポイント契約（`OPENCLAW_QA_CONVEX_SITE_URL` + `/qa-credentials/v1`）。
リクエストは `Authorization: Bearer <role secret>` ヘッダーで認証されます。
以下のボディではこのヘッダーを省略しています：

- `POST /acquire`
  - リクエスト：`{ kind, ownerId, actorRole, leaseTtlMs, heartbeatIntervalMs }`
  - 成功：`{ status: "ok", credentialId, leaseToken, payload, leaseTtlMs?, heartbeatIntervalMs? }`
  - 枯渇／再試行可能：`{ status: "error", code: "POOL_EXHAUSTED" | "NO_CREDENTIAL_AVAILABLE", ... }`
- `POST /payload-chunk`
  - リクエスト：`{ kind, ownerId, actorRole, credentialId, leaseToken, index }`
  - 成功：`{ status: "ok", index, data }`
- `POST /heartbeat`
  - リクエスト：`{ kind, ownerId, actorRole, credentialId, leaseToken, leaseTtlMs }`
  - 成功：`{ status: "ok" }`（または空の `2xx`）
- `POST /release`
  - リクエスト：`{ kind, ownerId, actorRole, credentialId, leaseToken }`
  - 成功：`{ status: "ok" }`（または空の `2xx`）
- `POST /admin/add`（メンテナーシークレットのみ）
  - リクエスト：`{ kind, actorId, payload, note?, status? }`
  - 成功：`{ status: "ok", credential }`
- `POST /admin/remove`（メンテナーシークレットのみ）
  - リクエスト：`{ credentialId, actorId }`
  - 成功：`{ status: "ok", changed, credential }`
  - アクティブリースのガード：`{ status: "error", code: "LEASE_ACTIVE", ... }`
- `POST /admin/list`（メンテナーシークレットのみ）
  - リクエスト：`{ kind?, status?, includePayload?, limit? }`
  - 成功：`{ status: "ok", credentials, count }`

Telegram 種類のペイロード形式：

- `{ groupId: string, driverToken: string, sutToken: string }`
- `groupId` は数値の Telegram チャット ID 文字列でなければなりません。
- `admin/add` は `kind: "telegram"` に対してこの形式を検証し、不正な形式のペイロードを拒否します。

Telegram 実ユーザー種類のペイロード形式：

- `{ groupId: string, sutToken: string, testerUserId: string, testerUsername: string, telegramApiId: string, telegramApiHash: string, tdlibDatabaseEncryptionKey: string, tdlibArchiveBase64: string, tdlibArchiveSha256: string, desktopTdataArchiveBase64: string, desktopTdataArchiveSha256: string }`
- `groupId`、`testerUserId`、および `telegramApiId` は数値文字列でなければなりません。
- `tdlibArchiveSha256` および `desktopTdataArchiveSha256` は SHA-256 16 進文字列でなければなりません。
- `kind: "telegram-user"` は Mantis Telegram Desktop 証明ワークフロー用に予約されています。汎用 QA ラボレーンがこれを取得してはなりません。

ブローカーで検証されるマルチチャネルペイロード：

- Discord：`{ guildId: string, channelId: string, driverBotToken: string, sutBotToken: string, sutApplicationId: string, voiceChannelId?: string }`
- WhatsApp：`{ driverPhoneE164: string, sutPhoneE164: string, driverAuthArchiveBase64: string, sutAuthArchiveBase64: string, groupJid?: string }`

Slack レーンもプールからリースできますが、Slack ペイロードの検証は現在、
ブローカーではなく Slack QA ランナーにあります。Slack の行には
`{ channelId: string, driverBotToken: string, sutBotToken: string, sutAppToken: string }`
を使用します。

### QA へのチャネル追加

新しいチャネルアダプターのアーキテクチャとシナリオヘルパー名については、
[QA の概要 - チャネルの追加](/ja-JP/concepts/qa-e2e-automation#adding-a-channel)を参照してください。
最低要件：共有の `qa-lab` ホストシーム上にトランスポートランナーを実装し、
共有シナリオ用の `adapterFactory` を追加し、Plugin マニフェストで `qaRunners` を宣言し、
`openclaw qa <runner>` としてマウントし、`qa/scenarios/` の下にシナリオを作成します。

## テストスイート（どこで何を実行するか）

各スイートは「現実性が増す」（同時に不安定性とコストも増す）ものと考えてください。

### ユニット／統合（デフォルト）

- コマンド：`pnpm test`
- 設定：対象を指定しない実行では `vitest.full-*.config.ts` シャードセットを使用し、
  並列スケジューリングのためにマルチプロジェクトシャードをプロジェクト単位の設定へ
  展開する場合があります
- ファイル：`src/**/*.test.ts`、
  `packages/**/*.test.ts`、および `test/**/*.test.ts` の下にあるコア／ユニットインベントリ。UI ユニットテストは
  専用の `unit-ui` シャードで実行されます
- スコープ：
  - 純粋なユニットテスト
  - プロセス内統合テスト（Gateway 認証、ルーティング、ツール、解析、設定）
  - 既知のバグに対する決定論的リグレッションテスト
- 要件：
  - CI で実行される
  - 実際のキーは不要
  - 高速かつ安定していること
  - リゾルバーおよび公開サーフェスのローダーテストでは、実際のバンドル済み Plugin ソース API ではなく、
    生成した小規模な Plugin フィクスチャを使用して、広範な `api.js` と
    `runtime-api.js` のフォールバック動作を証明する必要があります。実際の Plugin API のロードは、
    Plugin が所有する契約／統合スイートで行います。

ネイティブ依存関係のポリシー：

- デフォルトのテストインストールでは、オプションのネイティブ Discord opus ビルドをスキップします。Discord
  ボイスはバンドル済みの `libopus-wasm` を使用し、ローカルテストと Testbox レーンでネイティブ
  アドオンをコンパイルしないよう、`@discordjs/opus` は `allowBuilds` で無効のままにします。
- ネイティブ opus のパフォーマンス比較は、デフォルトの OpenClaw インストール／テストループではなく、
  `libopus-wasm` ベンチマークリポジトリで行います。デフォルトの `allowBuilds` で
  `@discordjs/opus` を `true` に設定しないでください。設定すると、無関係なインストール／テストループで
  ネイティブコードがコンパイルされます。

<AccordionGroup>
  <Accordion title="プロジェクト、シャード、スコープ付きレーン">

    - 対象指定なしの `pnpm test` は、1 つの巨大なネイティブルートプロジェクトプロセスの代わりに、13 個の小さなシャード設定（`core-unit-fast`、`core-unit-src`、`core-unit-security`、`core-unit-ui`、`core-unit-support`、`core-support-boundary`、`core-tooling`、`core-contracts`、`core-bundled`、`core-runtime`、`agentic`、`auto-reply`、`extensions`）を実行します。これにより、高負荷マシンでのピーク RSS が削減され、自動返信や Plugin の処理によって無関係なスイートがリソース不足になることを回避できます。
    - `pnpm test --watch` は引き続きネイティブルートの `vitest.config.ts` プロジェクトグラフを使用します。複数シャードの監視ループは実用的ではないためです。
    - `pnpm test`、`pnpm test:watch`、`pnpm test:perf:imports` は、明示的なファイルまたはディレクトリの対象を、まずスコープ付きレーンへ振り分けるため、`pnpm test extensions/discord/src/monitor/message-handler.preflight.test.ts` ではルートプロジェクト全体の起動コストを回避できます。
    - `pnpm test:changed` は、変更された git パスをデフォルトで低コストのスコープ付きレーンへ展開します。対象には、テストの直接編集、兄弟 `*.test.ts` ファイル、明示的なソースマッピング、ローカルインポートグラフの依存元が含まれます。設定、セットアップ、パッケージの編集では、`OPENCLAW_TEST_CHANGED_BROAD=1 pnpm test:changed` を明示的に使用しない限り、テストを広範囲に実行しません。
    - `pnpm check:changed` は、範囲の狭い作業向けの通常のスマートなローカルチェックゲートです。差分をコア、コアテスト、拡張機能、拡張機能テスト、アプリ、ドキュメント、リリースメタデータ、ライブ Docker ツール、ツールに分類し、対応する型チェック、lint、ガードコマンドを実行します。Vitest テストは実行しません。テストの証明には `pnpm test:changed` または明示的な `pnpm test <target>` を呼び出してください。リリースメタデータのみのバージョン更新では、バージョン、設定、ルート依存関係を対象としたチェックを実行し、最上位のバージョンフィールド以外のパッケージ変更をガードで拒否します。
    - ライブ Docker ACP ハーネスの編集では、対象を絞ったチェックを実行します。ライブ Docker 認証スクリプトのシェル構文と、ライブ Docker スケジューラのドライランです。`package.json` の変更は、差分が `scripts["test:docker:live-*"]` のみに限定される場合にだけ含まれます。依存関係、エクスポート、バージョン、その他のパッケージ公開面の編集には、引き続き広範なガードが使用されます。
    - エージェント、コマンド、Plugin、自動返信ヘルパー、`plugin-sdk`、および同様の純粋なユーティリティ領域にあるインポートの軽いユニットテストは、`test/setup-openclaw-runtime.ts` をスキップする `unit-fast` レーンへ振り分けられます。状態を持つファイルやランタイム負荷の高いファイルは、既存のレーンに残ります。
    - 選択された `plugin-sdk` および `commands` のヘルパーソースファイルも、変更モードの実行を、それらの軽量レーンにある明示的な兄弟テストへマッピングします。これにより、ヘルパーの編集時に、そのディレクトリの重量級スイート全体を再実行せずに済みます。
    - `auto-reply` には、最上位のコアヘルパー、最上位の `reply.*` 統合テスト、`src/auto-reply/reply/**` サブツリー専用のバケットがあります。CI ではさらに返信サブツリーを、エージェントランナー、ディスパッチ、コマンド／状態ルーティングの各シャードへ分割します。これにより、インポート負荷の高い 1 つのバケットが Node の処理末尾全体を占有することを防ぎます。
    - 通常の PR／main CI では、バンドルされた Plugin の一括スイープと、リリース時のみ使用する `agentic-plugins` シャードを意図的にスキップします。完全リリース検証では、リリース候補に対して、それらの Plugin 負荷が高いスイートを実行する別個の `Plugin Prerelease` 子ワークフローをディスパッチします。

  </Accordion>

  <Accordion title="組み込みランナーのカバレッジ">

    - メッセージツールの検出入力または Compaction ランタイムの
      コンテキストを変更する場合は、両方のレベルのカバレッジを維持してください。
    - 純粋なルーティングと正規化の境界には、対象を絞ったヘルパーの
      リグレッションテストを追加してください。
    - 組み込みランナーの統合スイートを正常な状態に保ってください：
      `src/agents/embedded-agent-runner/compact.hooks.test.ts`、
      `src/agents/embedded-agent-runner/run.overflow-compaction.test.ts`、および
      `src/agents/embedded-agent-runner/run.overflow-compaction.loop.test.ts`。
    - これらのスイートは、スコープ付き ID と Compaction の動作が引き続き
      実際の `run.ts`／`compact.ts` パスを通ることを検証します。ヘルパーのみのテストは、
      これらの統合パスを十分に代替するものではありません。

  </Accordion>

  <Accordion title="Vitest プールと分離のデフォルト">

    - 基本 Vitest 設定のデフォルトは `threads` です。
    - 共有 Vitest 設定は `isolate: false` を固定し、
      ルートプロジェクト、E2E、ライブ設定全体で非分離ランナーを使用します。
    - ルート UI レーンは `jsdom` のセットアップとオプティマイザーを維持しますが、
      共有の非分離ランナー上でも実行されます。
    - 各 `pnpm test` シャードは、共有 Vitest 設定から同じ `threads` と `isolate: false` の
      デフォルトを継承します。
    - `scripts/run-vitest.mjs` は、大規模なローカル実行時の V8 コンパイルの反復処理を削減するため、
      デフォルトで Vitest の子 Node プロセスに `--no-maglev` を追加します。
      標準の V8 動作と比較するには、`OPENCLAW_VITEST_ENABLE_MAGLEV=1` を設定してください。
    - `scripts/run-vitest.mjs` は、標準出力または標準エラー出力がない状態が
      5 分続いた場合、明示的な非監視 Vitest 実行を終了します。意図的に出力を抑えた調査で
      ウォッチドッグを無効にするには、`OPENCLAW_VITEST_NO_OUTPUT_TIMEOUT_MS=0` を設定してください。

  </Accordion>

  <Accordion title="高速なローカル反復">

    - `pnpm changed:lanes` は、差分によってどのアーキテクチャレーンがトリガーされるかを表示します。
    - pre-commit フックはフォーマットのみを行います。フォーマットされたファイルを再ステージしますが、
      lint、型チェック、テストは実行しません。
    - スマートなローカルチェックゲートが必要な場合は、引き渡しまたは push の前に
      `pnpm check:changed` を明示的に実行してください。
    - `pnpm test:changed` は、デフォルトで低コストのスコープ付きレーンへ振り分けられます。ハーネス、設定、
      パッケージ、または契約の編集に、より広範な Vitest カバレッジが本当に必要だとエージェントが判断した場合にのみ、
      `OPENCLAW_TEST_CHANGED_BROAD=1 pnpm test:changed` を使用してください。
    - `pnpm test:max` と `pnpm test:changed:max` は、ワーカー上限が高い点を除き、
      同じルーティング動作を維持します。
    - ローカルワーカーの自動スケーリングは意図的に控えめであり、ホストの負荷平均がすでに高い場合は
      処理量を抑えます。そのため、複数の Vitest を同時実行した際の影響がデフォルトで軽減されます。
    - 基本 Vitest 設定では、テストの配線が変更された場合でも変更モードの再実行が正しく保たれるように、
      プロジェクト／設定ファイルを `forceRerunTriggers` としてマークします。
    - この設定は、対応ホスト上で `OPENCLAW_VITEST_FS_MODULE_CACHE` を有効なまま維持します。
      直接プロファイリング用に明示的な単一のキャッシュ場所を指定するには、
      `OPENCLAW_VITEST_FS_MODULE_CACHE_PATH=/abs/path` を設定してください。

  </Accordion>

  <Accordion title="パフォーマンスのデバッグ">

    - `pnpm test:perf:imports` は、Vitest のインポート所要時間レポートと
      インポート内訳の出力を有効にします。
    - `pnpm test:perf:imports:changed` は、同じプロファイリングビューの対象を
      `origin/main` 以降に変更されたファイルに限定します。
    - シャードのタイミングデータは `.artifacts/vitest-shard-timings.json` に書き込まれます。
      設定全体の実行では設定パスをキーとして使用します。包含パターンを使用する CI
      シャードではシャード名を付加するため、フィルタリングされたシャードを
      個別に追跡できます。
    - 負荷の高い 1 つのテストが依然として起動時のインポートに大半の時間を費やしている場合は、
      重い依存関係を限定されたローカル `*.runtime.ts` 境界の背後に置き、
      ランタイムヘルパーを `vi.mock(...)` に渡すためだけに深いインポートを行うのではなく、
      その境界を直接モックしてください。
    - `pnpm test:perf:changed:bench -- --ref <git-ref>` は、コミット済みの差分について、ルーティングされた
      `test:changed` とネイティブルートプロジェクトのパスを比較し、
      経過時間と macOS の最大 RSS を出力します。
    - `pnpm test:perf:changed:bench -- --worktree` は、変更されたファイルの一覧を
      `scripts/test-projects.mjs` とルート Vitest 設定へ振り分けることで、
      現在の dirty tree をベンチマークします。
    - `pnpm test:perf:profile:main` は、Vitest／Vite の起動と変換のオーバーヘッドについて、
      メインスレッドの CPU プロファイルを書き出します。
    - `pnpm test:perf:profile:runner` は、ファイル並列処理を無効にしたユニットスイートについて、
      ランナーの CPU＋ヒーププロファイルを書き出します。

  </Accordion>
</AccordionGroup>

### 安定性（Gateway）

- コマンド：`pnpm test:stability:gateway`
- 設定：`test/vitest/vitest.gateway.config.ts`、`test/vitest/vitest.logging.config.ts`、`test/vitest/vitest.infra.config.ts`。それぞれ 1 ワーカーに固定
- 範囲：
  - 診断をデフォルトで有効にした実際のループバック Gateway を起動
  - 診断イベントパスを通じて、合成 Gateway メッセージ、メモリ、大容量ペイロードの反復処理を実行
  - Gateway WS RPC 経由で `diagnostics.stability` を照会
  - 診断安定性バンドルの永続化ヘルパーを対象に含める
  - レコーダーが上限内に収まり、合成 RSS サンプルが負荷予算を下回り、セッションごとのキュー深度が再びゼロになることを表明
- 期待事項：
  - CI で安全に実行でき、キー不要
  - 安定性リグレッションの追跡向けに範囲を絞ったレーンであり、Gateway スイート全体の代替ではない

### E2E（リポジトリ集約）

- コマンド：`pnpm test:e2e`
- 範囲：
  - Gateway スモーク E2E レーンを実行
  - モック化された Control UI ブラウザ E2E レーンを実行
- 期待事項：
  - CI で安全に実行でき、キー不要
  - Playwright Chromium がインストールされている必要がある

### E2E（Gateway スモーク）

- コマンド：`pnpm test:e2e:gateway`
- 設定：`test/vitest/vitest.e2e.config.ts`
- ファイル：`src/**/*.e2e.test.ts`、`test/**/*.e2e.test.ts`、および `extensions/` 配下のバンドルされた Plugin の E2E テスト
- ランタイムのデフォルト：
  - リポジトリの他の部分と同様に、Vitest `threads` と `isolate: false` を使用します。
  - 適応型ワーカーを使用します（CI：最大 2、ローカル：デフォルトで 1）。
  - コンソール I/O のオーバーヘッドを削減するため、デフォルトではサイレントモードで実行します。
- 便利なオーバーライド：
  - ワーカー数を強制するには `OPENCLAW_E2E_WORKERS=<n>`（上限 16）。
  - 詳細なコンソール出力を再度有効にするには `OPENCLAW_E2E_VERBOSE=1`。
- 範囲：
  - 複数インスタンスの Gateway のエンドツーエンド動作
  - WebSocket／HTTP 公開面、Node のペアリング、および負荷の高いネットワーク処理
- 期待事項：
  - CI で実行（パイプラインで有効な場合）
  - 実際のキーは不要
  - ユニットテストよりも可動部分が多い（時間がかかる場合がある）

### E2E（Control UI のモックブラウザ）

- コマンド：`pnpm test:ui:e2e`
- 設定：`test/vitest/vitest.ui-e2e.config.ts`
- ファイル：`ui/src/**/*.e2e.test.ts`
- 範囲：
  - Vite Control UI を起動
  - Playwright を通じて実際の Chromium ページを操作
  - Gateway WebSocket を、ブラウザ内の決定論的なモックに置き換える
- 期待事項：
  - `pnpm test:e2e` の一部として CI で実行
  - 実際の Gateway、エージェント、プロバイダーキーは不要
  - ブラウザ依存関係が存在する必要がある（`pnpm --dir ui exec playwright install chromium`）

### E2E：OpenShell バックエンドのスモーク

- コマンド：`pnpm test:e2e:openshell`
- ファイル：`extensions/openshell/src/backend.e2e.test.ts`
- 範囲：
  - 稼働中のローカル OpenShell Gateway を再利用
  - 一時的なローカル Dockerfile からサンドボックスを作成
  - 実際の `sandbox ssh-config`＋SSH exec を通じて OpenClaw の OpenShell バックエンドを実行
  - サンドボックスの fs ブリッジを通じて、リモートを正規とするファイルシステムの動作を検証
- 期待事項：
  - オプトインのみ。デフォルトの `pnpm test:e2e` 実行には含まれない
  - ローカルの `openshell` CLI と、動作中の Docker デーモンが必要
  - 稼働中のローカル OpenShell Gateway と、その設定ソースが必要
  - 分離された `HOME`／`XDG_CONFIG_HOME` を使用し、その後テスト用サンドボックスを破棄
- 便利なオーバーライド：
  - 広範な E2E スイートを手動で実行する際にテストを有効にするには `OPENCLAW_E2E_OPENSHELL=1`
  - デフォルト以外の CLI バイナリまたはラッパースクリプトを指定するには `OPENCLAW_E2E_OPENSHELL_COMMAND=/path/to/openshell`
  - 登録済みの Gateway 設定を分離テストに公開するには `OPENCLAW_E2E_OPENSHELL_CONFIG_HOME=/path/to/config`
  - ホストポリシーフィクスチャが使用する Docker Gateway IP を上書きするには `OPENCLAW_E2E_OPENSHELL_HOST_IP=172.18.0.1`

### ライブ（実際のプロバイダー＋実際のモデル）

- コマンド: `pnpm test:live`
- 設定: `test/vitest/vitest.live.config.ts`
- ファイル: `src/**/*.live.test.ts`、`test/**/*.live.test.ts`、および `extensions/` 配下の同梱 Plugin ライブテスト
- デフォルト: `pnpm test:live` により **有効**（`OPENCLAW_LIVE_TEST=1` を設定）
- 対象範囲:
  - 「このプロバイダー／モデルは、実際の認証情報を使って _現時点で_ 本当に動作するか？」
  - プロバイダーの形式変更、ツール呼び出しの特性、認証の問題、レート制限の動作を検出
- 想定事項:
  - 設計上、CI で安定しない（実際のネットワーク、実際のプロバイダーポリシー、クォータ、障害）
  - 費用が発生する／レート制限を消費する
  - 「すべて」ではなく、対象を絞ったサブセットの実行を推奨
- ライブ実行では、すでにエクスポートされている API キーとステージ済みの認証プロファイルを使用します。
- デフォルトでは、ライブ実行でも `HOME` を分離し、設定／認証情報を一時テストホームへコピーするため、ユニットテストのフィクスチャが実際の `~/.openclaw` を変更することはありません。
- ライブテストで実際のホームディレクトリを意図的に使用する必要がある場合にのみ、`OPENCLAW_LIVE_USE_REAL_HOME=1` を設定してください。
- `pnpm test:live` は、デフォルトでより静かなモードになります。`[live] ...` の進行状況出力は維持しつつ、Gateway のブートストラップログと Bonjour のメッセージを抑制します。起動ログをすべて再表示するには、`OPENCLAW_LIVE_TEST_QUIET=0` を設定してください。
- API キーのローテーション（プロバイダー固有）: カンマ／セミコロン形式の `*_API_KEYS`、または `*_API_KEY_1`、`*_API_KEY_2`（例: `OPENAI_API_KEYS`、`ANTHROPIC_API_KEYS`、`GEMINI_API_KEYS`）を設定するか、`OPENCLAW_LIVE_*_KEY` でライブ実行ごとにオーバーライドします。テストはレート制限レスポンスを受けると再試行します。
- 進行状況／Heartbeat 出力:
  - ライブスイートは進行状況行を stderr に出力するため、Vitest のコンソールキャプチャが静かな場合でも、時間のかかるプロバイダー呼び出しが動作中であることを確認できます。
  - `test/vitest/vitest.live.config.ts` は Vitest のコンソールインターセプトを無効にし、ライブ実行中にプロバイダー／Gateway の進行状況行を即座にストリーミングします。
  - 直接モデルの Heartbeat は `OPENCLAW_LIVE_HEARTBEAT_MS` で調整します。
  - Gateway／プローブの Heartbeat は `OPENCLAW_LIVE_GATEWAY_HEARTBEAT_MS` で調整します。

## どのスイートを実行すべきか？

次の判断表を使用してください:

- ロジック／テストの編集: `pnpm test` を実行（多数変更した場合は `pnpm test:coverage` も実行）
- Gateway のネットワーク処理／WS プロトコル／ペアリングの変更: `pnpm test:e2e` を追加
- 「ボットが停止している」問題／プロバイダー固有の障害／ツール呼び出しのデバッグ: 対象を絞った `pnpm test:live` を実行

## ライブ（ネットワーク接続を伴う）テスト

ライブモデルマトリクス、CLI バックエンドのスモークテスト、ACP スモークテスト、Codex app-server
ハーネス、およびすべてのメディアプロバイダーのライブテスト（Deepgram、BytePlus、ComfyUI、
画像、音楽、動画、メディアハーネス）に加え、ライブ実行の認証情報の処理については、

- [ライブスイートのテスト](/ja-JP/help/testing-live)を参照してください。更新と
  Plugin の専用検証チェックリストについては、
  [更新と Plugin のテスト](/ja-JP/help/testing-updates-plugins)を参照してください。

## Docker ランナー（任意の「Linux で動作する」ことの確認）

これらの Docker ランナーは、次の 2 つに分類されます:

- ライブモデルランナー: `test:docker:live-models` と `test:docker:live-gateway` は、リポジトリの Docker イメージ（`src/agents/models.profiles.live.test.ts` と `src/gateway/gateway-models.profiles.live.test.ts`）内で、それぞれに対応するプロファイルキーのライブファイルのみを実行し、ローカルの設定ディレクトリ、ワークスペース、および任意のプロファイル環境ファイルをマウントします。対応するローカルエントリーポイントは `test:live:models-profiles` と `test:live:gateway-profiles` です。
- Docker ライブランナーは、必要に応じて独自の実用的な上限を維持します:
  `test:docker:live-models` のデフォルトは、厳選された、サポート対象の高シグナルセットです。また、
  `test:docker:live-gateway` のデフォルトは `OPENCLAW_LIVE_GATEWAY_SMOKE=1`、
  `OPENCLAW_LIVE_GATEWAY_MAX_MODELS=8`、
  `OPENCLAW_LIVE_GATEWAY_STEP_TIMEOUT_MS=45000`、および
  `OPENCLAW_LIVE_GATEWAY_MODEL_TIMEOUT_MS=90000` です。明示的に上限を小さくする、またはスキャン範囲を広げる場合は、`OPENCLAW_LIVE_MAX_MODELS`
  または Gateway の環境変数を設定してください。
- `test:docker:all` は `test:docker:live-build` を介してライブ Docker イメージを一度だけビルドし、`scripts/package-openclaw-for-docker.mjs` を介して OpenClaw を npm tarball として一度だけパッケージ化してから、2 つの `scripts/e2e/Dockerfile` イメージをビルドまたは再利用します。ベアイメージは、インストール／更新／Plugin 依存関係レーン専用の Node／Git ランナーであり、これらのレーンは事前ビルド済み tarball をマウントします。機能イメージは、ビルド済みアプリの機能レーン向けに、同じ tarball を `/app` にインストールします。Docker レーンの定義は `scripts/lib/docker-e2e-scenarios.mjs`、プランナーのロジックは `scripts/lib/docker-e2e-plan.mjs` にあり、`scripts/test-docker-all.mjs` が選択されたプランを実行します。集約処理では重み付きローカルスケジューラーを使用します。`OPENCLAW_DOCKER_ALL_PARALLELISM` がプロセススロットを制御し、リソース上限により、負荷の高いライブ、npm インストール、複数サービスの各レーンが一斉に開始されるのを防ぎます。単一レーンの負荷が有効な上限を超えていても、プールが空であればスケジューラーはそのレーンを開始でき、再び容量が利用可能になるまで単独で実行し続けます。デフォルトは 10 スロット、`OPENCLAW_DOCKER_ALL_LIVE_LIMIT=9`、`OPENCLAW_DOCKER_ALL_NPM_LIMIT=5`、および `OPENCLAW_DOCKER_ALL_SERVICE_LIMIT=7` です。Docker ホストに十分な余裕がある場合にのみ、`OPENCLAW_DOCKER_ALL_WEIGHT_LIMIT` または `OPENCLAW_DOCKER_ALL_DOCKER_LIMIT`（およびその他の `OPENCLAW_DOCKER_ALL_<RESOURCE>_LIMIT` オーバーライド）を調整してください。ランナーはデフォルトで Docker の事前確認を実行し、古い OpenClaw E2E コンテナを削除し、30 秒ごとにステータスを出力し、成功したレーンの所要時間を `.artifacts/docker-tests/lane-timings.json` に保存し、後続の実行ではその所要時間を使用して時間のかかるレーンから先に開始します。Docker をビルドまたは実行せずに重み付きレーンマニフェストを出力するには `OPENCLAW_DOCKER_ALL_DRY_RUN=1`、選択したレーン、パッケージ／イメージの要件、認証情報に関する CI プランを出力するには `node scripts/test-docker-all.mjs --plan-json` を使用します。
- `Package Acceptance` は、「このインストール可能な tarball が製品として動作するか」を確認する GitHub ネイティブのパッケージゲートです。`source=npm`、`source=ref`、`source=url`、`source=trusted-url`、または `source=artifact` から候補パッケージを 1 つ解決し、それを `package-under-test` としてアップロードしてから、選択した ref を再パッケージ化する代わりに、その正確な tarball に対して再利用可能な Docker E2E レーンを実行します。プロファイルは対象範囲の広い順に `smoke`、`package`、`product`、`full`（および明示的なレーン一覧用の `custom`）です。パッケージ／更新／Plugin の契約、公開済みアップグレードのサバイバーマトリクス、リリースのデフォルト、障害のトリアージについては、[更新と Plugin のテスト](/ja-JP/help/testing-updates-plugins)を参照してください。
- ビルドとリリースのチェックでは、tsdown の後に `scripts/check-cli-bootstrap-imports.mjs` を実行します。このガードは `dist/entry.js` と `dist/cli/run-main.js` から静的なビルド済みグラフを走査し、コマンドのディスパッチ前に、そのブートストラップグラフが外部パッケージ（Commander、プロンプト UI、undici、ロギング、および同様に起動時の負荷が高い依存関係をすべて含む）を静的インポートしている場合に失敗します。また、バンドルされた Gateway 実行チャンクを 70 KB に制限し、そのチャンクから既知のコールド Gateway パス（`control-ui-assets`、`diagnostic-stability-bundle`、`onboard-helpers`、`process-respawn`、`restart-sentinel`、`server-close`、`server-reload-handlers`）を静的インポートすることを拒否します。`scripts/release-check.ts` は別途、パッケージ化された CLI を `--help`、`onboard --help`、`doctor --help`、`status --json --timeout 1`、`config schema`、および `models list --provider openai` でスモークテストします。
- Package Acceptance のレガシー互換性は `2026.4.25`（`2026.4.25-beta.*` を含む）までに制限されます。この期限までは、ハーネスが許容するのは、リリース済みパッケージのメタデータ欠落のみです。具体的には、非公開 QA インベントリエントリの省略、`gateway install --wrapper` の欠落、tarball 由来の git フィクスチャ内のパッチファイルの欠落、永続化された `update.channel` の欠落、レガシーな Plugin インストールレコードの保存場所、マーケットプレイスのインストールレコード永続化の欠落、および `plugins update` 中の設定メタデータ移行です。`2026.4.25` より後のパッケージでは、これらのパスは厳格な失敗として扱われます。
- コンテナスモークランナー: `test:docker:openwebui`、`test:docker:onboard`、`test:docker:npm-onboard-channel-agent`、`test:docker:release-user-journey`、`test:docker:release-typed-onboarding`、`test:docker:release-media-memory`、`test:docker:release-upgrade-user-journey`、`test:docker:release-plugin-marketplace`、`test:docker:skill-install`、`test:docker:update-channel-switch`、`test:docker:upgrade-survivor`、`test:docker:published-upgrade-survivor`、`test:docker:session-runtime-context`、`test:docker:agents-delete-shared-workspace`、`test:docker:gateway-network`、`test:docker:browser-cdp-snapshot`、`test:docker:mcp-channels`、`test:docker:agent-bundle-mcp-tools`、`test:docker:cron-mcp-cleanup`、`test:docker:plugins`、`test:docker:plugin-update`、`test:docker:plugin-lifecycle-matrix`、および `test:docker:config-reload` は、1 つ以上の実際のコンテナを起動し、より上位レベルの統合パスを検証します。
- `scripts/lib/openclaw-e2e-instance.sh` を介してパッケージ化された OpenClaw tarball をインストールする Docker／Bash E2E レーンは、`npm install` を `OPENCLAW_E2E_NPM_INSTALL_TIMEOUT` に制限します（デフォルトは `600s`。デバッグのためにラッパーを無効にするには `0` を設定）。

ライブモデルの Docker ランナーは、必要な CLI 認証ホームのみ
（実行対象が絞られていない場合はサポート対象のすべて）をバインドマウントし、実行前に
コンテナのホームへコピーします。これにより、外部 CLI の OAuth がホストの認証ストアを
変更することなくトークンを更新できます:

- 直接モデル: `pnpm test:docker:live-models`（スクリプト: `scripts/test-live-models-docker.sh`）
- ACP バインドスモークテスト: `pnpm test:docker:live-acp-bind`（スクリプト: `scripts/test-live-acp-bind-docker.sh`。デフォルトで Claude、Codex、Gemini を対象とし、`pnpm test:docker:live-acp-bind:droid` と `pnpm test:docker:live-acp-bind:opencode` により Droid／OpenCode を厳格に対象化）
- CLI バックエンドスモークテスト: `pnpm test:docker:live-cli-backend`（スクリプト: `scripts/test-live-cli-backend-docker.sh`）
- Codex app-server ハーネスのスモークテスト: `pnpm test:docker:live-codex-harness`（スクリプト: `scripts/test-live-codex-harness-docker.sh`）
- Gateway + 開発エージェント: `pnpm test:docker:live-gateway`（スクリプト: `scripts/test-live-gateway-models-docker.sh`）
- 可観測性スモークテスト: `pnpm qa:otel:smoke`、`pnpm qa:prometheus:smoke`、および `pnpm qa:observability:smoke` は、非公開 QA のソースチェックアウト用レーンです。npm tarball には QA Lab が含まれないため、意図的にパッケージ Docker リリースレーンの一部にはしていません。
- Open WebUI ライブスモークテスト: `pnpm test:docker:openwebui`（スクリプト: `scripts/e2e/openwebui-docker.sh`）
- オンボーディングウィザード（TTY、完全なスキャフォールディング）: `pnpm test:docker:onboard`（スクリプト: `scripts/e2e/onboard-docker.sh`）
- npm tarball のオンボーディング／チャンネル／エージェントのスモークテスト: `pnpm test:docker:npm-onboard-channel-agent` は、パッケージ化された OpenClaw tarball を Docker 内へグローバルインストールし、環境変数参照によるオンボーディングを介して OpenAI を設定するとともに、デフォルトで Telegram も設定し、doctor を実行して、モック化された OpenAI エージェントを 1 ターン実行します。事前ビルド済み tarball を再利用するには `OPENCLAW_CURRENT_PACKAGE_TGZ=/path/to/openclaw-*.tgz`、ホスト側の再ビルドをスキップするには `OPENCLAW_NPM_ONBOARD_HOST_BUILD=0`、チャンネルを切り替えるには `OPENCLAW_NPM_ONBOARD_CHANNEL=discord` または `OPENCLAW_NPM_ONBOARD_CHANNEL=slack` を使用します。

- リリース版ユーザージャーニースモーク: `pnpm test:docker:release-user-journey` は、クリーンな Docker ホームにパッケージ化された OpenClaw tarball をグローバルにインストールし、オンボーディングを実行し、モック OpenAI プロバイダーを設定し、エージェントターンを実行し、外部プラグインのインストールとアンインストールを行い、ローカルフィクスチャに対して ClickClack を設定し、送受信メッセージングを検証し、Gateway を再起動して doctor を実行します。
- リリース版の型付きオンボーディングスモーク: `pnpm test:docker:release-typed-onboarding` は、パッケージ化された tarball をインストールし、実際の TTY を介して `openclaw onboard` を操作し、OpenAI を環境変数参照プロバイダーとして設定し、生のキーが永続化されないことを検証して、モックされたエージェントターンを実行します。
- リリース版メディア／メモリスモーク: `pnpm test:docker:release-media-memory` は、パッケージ化された tarball をインストールし、PNG 添付ファイルからの画像理解、OpenAI 互換の画像生成出力、メモリ検索による想起、および Gateway 再起動後も想起が維持されることを検証します。
- リリース版アップグレードユーザージャーニースモーク: `pnpm test:docker:release-upgrade-user-journey` は、デフォルトで候補 tarball より古い公開済みベースラインのうち最新のものをインストールし、公開パッケージ上でプロバイダー／プラグイン／ClickClack の状態を設定し、候補 tarball にアップグレードしてから、エージェント／プラグイン／チャネルの中核ジャーニーを再実行します。古い公開済みベースラインが存在しない場合は、候補バージョンを再利用します。ベースラインは `OPENCLAW_RELEASE_UPGRADE_BASELINE_SPEC=openclaw@<version>` で上書きします。
- リリース版プラグインマーケットプレイススモーク: `pnpm test:docker:release-plugin-marketplace` は、ローカルフィクスチャのマーケットプレイスからインストールし、インストール済みプラグインを更新してアンインストールし、インストールメタデータの削除に伴ってプラグイン CLI が消えることを検証します。
- Skill インストールスモーク: `pnpm test:docker:skill-install` は、パッケージ化された OpenClaw tarball を Docker にグローバルインストールし、設定でアップロード済みアーカイブのインストールを無効化し、検索から現在公開中の ClawHub Skill スラッグを解決し、`openclaw skills install` でインストールして、インストール済み Skill と `.clawhub` のオリジン／ロックメタデータを検証します。
- 更新チャネル切り替えスモーク: `pnpm test:docker:update-channel-switch` は、パッケージ化された OpenClaw tarball を Docker にグローバルインストールし、パッケージ `stable` から git `dev` に切り替え、永続化されたチャネルと更新後のプラグイン動作を検証してから、パッケージ `stable` に戻し、更新ステータスを確認します。
- アップグレード生存性スモーク: `pnpm test:docker:upgrade-survivor` は、エージェント、チャネル設定、プラグイン許可リスト、古いプラグイン依存関係の状態、既存のワークスペース／セッションファイルを含む、変更済みの旧ユーザーフィクスチャ上にパッケージ化された OpenClaw tarball をインストールします。実際のプロバイダーキーやチャネルキーを使わずにパッケージ更新と非対話型 doctor を実行し、loopback Gateway を起動して、設定／状態の保持と起動／ステータスの時間枠を確認します。
- 公開済みアップグレード生存性スモーク: `pnpm test:docker:published-upgrade-survivor` は、デフォルトで `openclaw@latest` をインストールし、現実的な既存ユーザーファイルを用意し、組み込みのコマンドレシピでそのベースラインを設定し、生成された設定を検証し、その公開済みインストールを候補 tarball に更新し、非対話型 doctor を実行し、`.artifacts/upgrade-survivor/summary.json` を書き込んでから、loopback Gateway を起動し、設定された意図、状態の保持、起動、`/healthz`、`/readyz`、RPC ステータスの時間枠を確認します。1 つのベースラインを `OPENCLAW_UPGRADE_SURVIVOR_BASELINE_SPEC` で上書きし、`openclaw@2026.5.2 openclaw@2026.4.23 openclaw@2026.4.15` などの正確なローカルベースラインを `OPENCLAW_UPGRADE_SURVIVOR_BASELINE_SPECS` で展開するよう集約スケジューラーに指示し、`reported-issues` などの Issue 形式のフィクスチャを `OPENCLAW_UPGRADE_SURVIVOR_SCENARIOS` で展開します。報告済み Issue のセットには、外部 OpenClaw プラグインの自動インストール修復用の `configured-plugin-installs` が含まれます。Package Acceptance はこれらを `published_upgrade_survivor_baseline`、`published_upgrade_survivor_baselines`、`published_upgrade_survivor_scenarios` として公開し、`last-stable-4` や `all-since-2026.4.23` などのメタベースライントークンを解決します。また、Full Release Validation はリリースソークパッケージゲートを `last-stable-4 2026.4.23 2026.5.2 2026.4.15` と `reported-issues` に展開します。
- セッションランタイムコンテキストスモーク: `pnpm test:docker:session-runtime-context` は、非表示のランタイムコンテキストのトランスクリプト永続化と、影響を受けた重複プロンプト書き換え分岐に対する doctor 修復を検証します。
- Bun グローバルインストールスモーク: `bash scripts/e2e/bun-global-install-smoke.sh` は、現在のツリーをパッケージ化し、隔離されたホームに `bun install -g` でインストールし、`openclaw infer image providers --json` がハングせずにバンドル済み画像プロバイダーを返すことを検証します。事前ビルド済み tarball を `OPENCLAW_BUN_GLOBAL_SMOKE_PACKAGE_TGZ=/path/to/openclaw-*.tgz` で再利用し、`OPENCLAW_BUN_GLOBAL_SMOKE_HOST_BUILD=0` でホストビルドをスキップするか、`OPENCLAW_BUN_GLOBAL_SMOKE_DIST_IMAGE=openclaw-dockerfile-smoke:local` でビルド済み Docker イメージから `dist/` をコピーします。
- インストーラー Docker スモーク: `bash scripts/test-install-sh-docker.sh` は、root、更新、直接 npm の各コンテナ間で 1 つの npm キャッシュを共有します。更新スモークでは、候補 tarball へアップグレードする前の安定版ベースラインとして、デフォルトで npm `latest` を使用します。ローカルでは `OPENCLAW_INSTALL_SMOKE_UPDATE_BASELINE=2026.4.22`、GitHub では Install Smoke ワークフローの `update_baseline_version` 入力で上書きします。非 root インストーラーの確認では、root 所有のキャッシュエントリによってユーザーローカルのインストール動作が隠れないよう、隔離された npm キャッシュを維持します。ローカルでの再実行時に root／更新／直接 npm のキャッシュを再利用するには、`OPENCLAW_INSTALL_SMOKE_NPM_CACHE_DIR=/path/to/cache` を設定します。
- Install Smoke CI は、`OPENCLAW_INSTALL_SMOKE_SKIP_NPM_GLOBAL=1` によって重複する直接 npm グローバル更新をスキップします。直接 `npm install -g` のカバレッジが必要な場合は、その環境変数を指定せずにスクリプトをローカル実行します。
- エージェント共有ワークスペース削除 CLI スモーク: `pnpm test:docker:agents-delete-shared-workspace`（スクリプト: `scripts/e2e/agents-delete-shared-workspace-docker.sh`）は、デフォルトでルート Dockerfile イメージをビルドし、隔離されたコンテナホームに 1 つのワークスペースを共有する 2 つのエージェントを用意し、`agents delete --json` を実行して、有効な JSON とワークスペース保持動作を検証します。インストールスモークイメージは `OPENCLAW_AGENTS_DELETE_SHARED_WORKSPACE_E2E_IMAGE=openclaw-dockerfile-smoke:local OPENCLAW_AGENTS_DELETE_SHARED_WORKSPACE_E2E_SKIP_BUILD=1` で再利用します。
- Gateway のネットワークとホストライフサイクル: `pnpm test:docker:gateway-network`（スクリプト: `scripts/e2e/gateway-network-docker.sh`）は、2 コンテナ LAN WebSocket 認証／ヘルススモークを維持したうえで、loopback Admin HTTP を使用して、準備フェンシング、保持された制御アクセス、再開による復旧、準備済みの同一コンテナでの停止／起動を実証します。再起動確認は元のリースが期限切れになる前に完了する必要があり、永続化された Gateway 設定とコンテナ ID が維持される一方で、サスペンド状態はプロセスローカルであることを検証し、機械可読なフェーズ時間計測 JSON を出力します。
- ブラウザー CDP スナップショットスモーク: `pnpm test:docker:browser-cdp-snapshot`（スクリプト: `scripts/e2e/browser-cdp-snapshot-docker.sh`）は、ソース E2E イメージと Chromium レイヤーをビルドし、生の CDP で Chromium を起動し、`browser doctor --deep` を実行して、CDP ロールスナップショットがリンク URL、カーソルによってクリック可能に昇格した要素、iframe 参照、フレームメタデータを網羅することを検証します。
- OpenAI Responses web_search 最小推論リグレッション: `pnpm test:docker:openai-web-search-minimal`（スクリプト: `scripts/e2e/openai-web-search-minimal-docker.sh`）は、Gateway を介してモック OpenAI サーバーを実行し、`web_search` によって `reasoning.effort` が `minimal` から `low` に引き上げられることを検証してから、プロバイダースキーマの拒否を強制し、生の詳細が Gateway ログに表示されることを確認します。
- MCP チャネルブリッジ（シード済み Gateway + stdio ブリッジ + 生の Claude 通知フレームスモーク）: `pnpm test:docker:mcp-channels`（スクリプト: `scripts/e2e/mcp-channels-docker.sh`）
- OpenClaw バンドル MCP ツール（実際の stdio MCP サーバー + 埋め込み OpenClaw プロファイルの許可／拒否スモーク）: `pnpm test:docker:agent-bundle-mcp-tools`（スクリプト: `scripts/e2e/agent-bundle-mcp-tools-docker.sh`）
- Cron／サブエージェント MCP クリーンアップ（実際の Gateway + 隔離された Cron および 1 回限りのサブエージェント実行後の stdio MCP 子プロセス終了）: `pnpm test:docker:cron-mcp-cleanup`（スクリプト: `scripts/e2e/cron-mcp-cleanup-docker.sh`）
- プラグイン（ローカルパス、`file:`、依存関係をホイストした npm レジストリ、不正な npm パッケージメタデータ、移動する git 参照、ClawHub kitchen-sink、マーケットプレイス更新、Claude バンドルの有効化／検査に対するインストール／更新スモーク）: `pnpm test:docker:plugins`（スクリプト: `scripts/e2e/plugins-docker.sh`）
  ClawHub ブロックをスキップするには `OPENCLAW_PLUGINS_E2E_CLAWHUB=0` を設定し、デフォルトの kitchen-sink パッケージ／ランタイムの組み合わせを上書きするには `OPENCLAW_PLUGINS_E2E_CLAWHUB_SPEC` と `OPENCLAW_PLUGINS_E2E_CLAWHUB_ID` を設定します。`OPENCLAW_CLAWHUB_URL`/`CLAWHUB_URL` を指定しない場合、テストは自己完結型のローカル ClawHub フィクスチャサーバーを使用します。
- プラグイン更新変更なしスモーク: `pnpm test:docker:plugin-update`（スクリプト: `scripts/e2e/plugin-update-unchanged-docker.sh`）
- プラグインライフサイクルマトリクススモーク: `pnpm test:docker:plugin-lifecycle-matrix` は、最小構成のコンテナにパッケージ化された OpenClaw tarball をインストールし、npm プラグインをインストールし、有効／無効を切り替え、ローカル npm レジストリを介してアップグレードとダウングレードを行い、インストール済みコードを削除してから、各ライフサイクルフェーズの RSS／CPU メトリクスを記録しつつ、アンインストールによって古い状態が引き続き削除されることを検証します。
- 設定再読み込みメタデータスモーク: `pnpm test:docker:config-reload`（スクリプト: `scripts/e2e/config-reload-source-docker.sh`）
- プラグイン: `pnpm test:docker:plugins` は、ローカルパス、`file:`、依存関係をホイストした npm レジストリ、移動する git 参照、ClawHub フィクスチャ、マーケットプレイス更新、Claude バンドルの有効化／検査に対するインストール／更新スモークを網羅します。`pnpm test:docker:plugin-update` は、インストール済みプラグインの変更なし更新動作を網羅します。`pnpm test:docker:plugin-lifecycle-matrix` は、リソース追跡付き npm プラグインのインストール、有効化、無効化、アップグレード、ダウングレード、コード欠損時のアンインストールを網羅します。

共有機能イメージを手動で事前ビルドして再利用するには:

```bash
OPENCLAW_DOCKER_E2E_IMAGE=openclaw-docker-e2e-functional:local pnpm test:docker:e2e-build
OPENCLAW_DOCKER_E2E_IMAGE=openclaw-docker-e2e-functional:local OPENCLAW_SKIP_DOCKER_BUILD=1 pnpm test:docker:mcp-channels
```

`OPENCLAW_GATEWAY_NETWORK_E2E_IMAGE` などのスイート固有のイメージ上書きは、設定されている場合は引き続き優先されます。`OPENCLAW_SKIP_DOCKER_BUILD=1` がリモート共有イメージを指している場合、ローカルにまだ存在しなければスクリプトがそのイメージを pull します。QR およびインストーラーの Docker テストは、共有のビルド済みアプリランタイムではなくパッケージ／インストール動作を検証するため、独自の Dockerfile を維持します。

ライブモデル Docker ランナーも、現在のチェックアウトを読み取り専用でバインドマウントし、
コンテナ内の一時作業ディレクトリにステージングします。これにより、正確なローカルの
ソース／設定に対して Vitest を実行しながら、ランタイムイメージを軽量に保てます。
ステージング処理では、`.pnpm-store`、`.worktrees`、`__openclaw_vitest__`、
およびアプリローカルの `.build` や Gradle の出力ディレクトリなど、
容量の大きいローカル専用キャッシュとアプリのビルド出力をスキップするため、Docker のライブ実行で
マシン固有のアーティファクトのコピーに数分を費やすことはありません。また、
コンテナ内で Gateway のライブプローブが実際の
Telegram／Discord などのチャネルワーカーを起動しないように、
`OPENCLAW_SKIP_CHANNELS=1` も設定します。
`test:docker:live-models` は引き続き `pnpm test:live` を実行するため、その Docker レーンから
Gateway のライブカバレッジを絞り込んだり除外したりする必要がある場合は、
`OPENCLAW_LIVE_GATEWAY_*` も引き渡してください。

`test:docker:openwebui` は、より高レベルの互換性スモークです。OpenAI 互換 HTTP エンドポイントを
有効にした OpenClaw Gateway コンテナを起動し、その Gateway に対して固定バージョンの
Open WebUI コンテナを起動し、Open WebUI 経由でサインインし、
`/api/models` が `openclaw/default` を公開することを検証してから、
Open WebUI の `/api/chat/completions` プロキシを介して実際のチャットリクエストを送信します。
実際のモデル補完を待たず、Open WebUI へのサインインとモデル検出後に停止すべき
リリースパス CI チェックでは、`OPENWEBUI_SMOKE_MODE=models` を設定します。
初回実行は、Docker が Open WebUI イメージを pull する必要があり、Open WebUI 自体の
コールドスタート設定が完了するまで待つ場合があるため、明らかに遅くなることがあります。
このレーンでは、プロセス環境、ステージングされた認証プロファイル、または明示的な
`OPENCLAW_PROFILE_FILE` を通じて提供される、使用可能なライブモデルキーが必要です。
成功した実行では、`{ "ok": true, "model": "openclaw/default", ... }` のような小さな JSON ペイロードが出力されます。

`test:docker:mcp-channels` は意図的に決定論的であり、実際の Telegram、Discord、iMessage
アカウントを必要としません。シード済み Gateway コンテナを起動し、
`openclaw mcp serve` を生成する 2 つ目のコンテナを起動してから、
ルーティングされた会話の検出、トランスクリプトの読み取り、添付ファイルの
メタデータ、ライブイベントキューの動作、送信ルーティング、Claude 形式の
チャネル通知と権限通知を、実際の stdio MCP ブリッジ経由で検証します。
通知確認では、生の stdio MCP フレームを直接検査するため、このスモークは、
特定のクライアント SDK がたまたま公開する内容だけでなく、ブリッジが実際に
出力する内容を検証します。

`test:docker:agent-bundle-mcp-tools` は決定的であり、ライブモデルキーは不要です。
リポジトリの Docker イメージをビルドし、コンテナ内で実際の stdio MCP
プローブサーバーを起動し、組み込みの OpenClaw バンドル MCP ランタイムを通じて
そのサーバーを実体化してツールを実行した後、
`coding` と `messaging` では `bundle-mcp` ツールが維持される一方、`minimal` と
`tools.deny: ["bundle-mcp"]` ではフィルタリングされることを検証します。

`test:docker:cron-mcp-cleanup` は決定的であり、ライブ
モデルキーは不要です。実際の stdio MCP プローブサーバーを備えたシード済み Gateway を起動し、
分離された Cron ターンと `sessions_spawn` のワンショット子ターンを実行した後、
各実行後に MCP 子プロセスが終了することを検証します。

ACP の平易な言語によるスレッドの手動スモークテスト（CI ではありません）：

- `bun scripts/dev/discord-acp-plain-language-smoke.ts --channel <discord-channel-id> ...`
- このスクリプトは回帰／デバッグワークフロー用に保持してください。ACP スレッドルーティングの検証で再び必要になる可能性があるため、削除しないでください。

便利な環境変数：

- `OPENCLAW_CONFIG_DIR=...`（デフォルト：`~/.openclaw`）は `/home/node/.openclaw` にマウントされます
- `OPENCLAW_WORKSPACE_DIR=...`（デフォルト：`~/.openclaw/workspace`）は `/home/node/.openclaw/workspace` にマウントされます
- `OPENCLAW_PROFILE_FILE=...` はマウントされ、テスト実行前に読み込まれます
- `OPENCLAW_DOCKER_PROFILE_ENV_ONLY=1` は、一時的な設定／ワークスペースディレクトリを使用し、外部 CLI 認証マウントなしで、`OPENCLAW_PROFILE_FILE` から読み込まれた環境変数のみを検証します
- `OPENCLAW_DOCKER_CLI_TOOLS_DIR=...`（デフォルト：`~/.cache/openclaw/docker-cli-tools`。実行ですでに CI／管理対象のバインドディレクトリを使用している場合を除く）は、Docker 内のキャッシュ済み CLI インストール用に `/home/node/.npm-global` にマウントされます
- `$HOME` 配下の外部 CLI 認証ディレクトリ／ファイルは `/host-auth...` 配下に読み取り専用でマウントされ、テスト開始前に `/home/node/...` へコピーされます
  - デフォルトのディレクトリ（実行対象を特定のプロバイダーに限定しない場合に使用）：`.factory`、`.gemini`、`.minimax`
  - デフォルトのファイル：`~/.codex/auth.json`、`~/.codex/config.toml`、`.claude.json`、`~/.claude/.credentials.json`、`~/.claude/settings.json`、`~/.claude/settings.local.json`
  - プロバイダーを限定した実行では、`OPENCLAW_LIVE_PROVIDERS`／`OPENCLAW_LIVE_GATEWAY_PROVIDERS` から推定された必要なディレクトリ／ファイルのみをマウントします
  - `OPENCLAW_DOCKER_AUTH_DIRS=all`、`OPENCLAW_DOCKER_AUTH_DIRS=none`、または `OPENCLAW_DOCKER_AUTH_DIRS=.claude,.codex` のようなカンマ区切りリストで手動オーバーライドできます
- `OPENCLAW_LIVE_GATEWAY_MODELS=...`／`OPENCLAW_LIVE_MODELS=...` で実行対象を限定します
- `OPENCLAW_LIVE_GATEWAY_PROVIDERS=...`／`OPENCLAW_LIVE_PROVIDERS=...` でコンテナ内のプロバイダーをフィルタリングします
- `OPENCLAW_SKIP_DOCKER_BUILD=1` は、再ビルドが不要な再実行で既存の `openclaw:local-live` イメージを再利用します
- `OPENCLAW_LIVE_REQUIRE_PROFILE_KEYS=1` は、認証情報が環境変数ではなくプロファイルストアから取得されることを保証します
- `OPENCLAW_OPENWEBUI_MODEL=...` は、Open WebUI スモークテスト用に Gateway が公開するモデルを選択します
- `OPENCLAW_OPENWEBUI_PROMPT=...` は、Open WebUI スモークテストで使用する nonce チェックプロンプトをオーバーライドします
- `OPENWEBUI_IMAGE=...` は、固定された Open WebUI イメージタグをオーバーライドします

## ドキュメントの健全性チェック

ドキュメントの編集後にドキュメントチェックを実行します：`pnpm check:docs`。
ページ内見出しのチェックも必要な場合は、Mintlify の完全なアンカー検証を実行します：`pnpm docs:check-links:anchors`。

## オフライン回帰テスト（CI セーフ）

以下は、実際のプロバイダーを使用しない「実パイプライン」回帰テストです：

- Gateway のツール呼び出し（モック OpenAI、実際の Gateway＋エージェントループ）：`src/gateway/gateway.test.ts`（ケース：「Gateway エージェントループを介してモック OpenAI ツール呼び出しをエンドツーエンドで実行する」）
- Gateway ウィザード（WS `wizard.start`/`wizard.next`、設定を書き込み＋認証を強制）：`src/gateway/gateway.test.ts`（ケース：「WS 経由でウィザードを実行し、認証トークン設定を書き込む」）

## エージェント信頼性評価（Skills）

「エージェント信頼性評価」のように機能する CI セーフなテストが、すでにいくつかあります：

- 実際の Gateway＋エージェントループを介したモックツール呼び出し（`src/gateway/gateway.test.ts`）。
- セッション接続と設定の効果を検証するエンドツーエンドのウィザードフロー（`src/gateway/gateway.test.ts`）。

Skills についてまだ不足しているもの（[Skills](/ja-JP/tools/skills) を参照）：

- **判断：**プロンプトに Skills が列挙されている場合、エージェントは適切なスキルを選択するか（または無関係なスキルを回避するか）？
- **準拠：**エージェントは使用前に `SKILL.md` を読み、必須の手順／引数に従うか？
- **ワークフロー契約：**ツールの順序、セッション履歴の引き継ぎ、サンドボックス境界を検証するマルチターンシナリオ。

今後の評価では、まず決定性を維持する必要があります：

- モックプロバイダーを使用して、ツール呼び出しとその順序、スキルファイルの読み取り、セッション接続を検証するシナリオランナー。
- スキルに焦点を当てた小規模なシナリオスイート（使用と回避、ゲーティング、プロンプトインジェクション）。
- CI セーフなスイートが整備された後にのみ追加する、任意のライブ評価（オプトイン、環境変数による制御）。

## 契約テスト（Plugin とチャネルの形状）

契約テストは、登録されているすべての Plugin とチャネルが
それぞれのインターフェース契約に準拠していることを検証します。検出されたすべての Plugin を反復処理し、
形状と動作に関するアサーションスイートを実行します。デフォルトの `pnpm test` ユニットレーンでは、
これらの共有シームおよびスモークファイルを意図的にスキップします。共有チャネルまたはプロバイダーのサーフェスに変更を加えた場合は、
契約コマンドを明示的に実行してください。

### コマンド

- すべての契約：`pnpm test:contracts`
- チャネル契約のみ：`pnpm test:contracts:channels`
- プロバイダー契約のみ：`pnpm test:contracts:plugins`

### チャネル契約

`src/channels/plugins/contracts/*.contract.test.ts` にあります。現在の
最上位カテゴリ：

- **channel-catalog** - バンドル／レジストリのチャネルカタログエントリのメタデータ
- **plugin**（レジストリベース、シャーディング済み）- 基本的な Plugin 登録の形状
- **surfaces-only**（レジストリベース、シャーディング済み）- `actions`、`setup`、`status`、`outbound`、`messaging`、`threading`、`directory`、`gateway` の各サーフェスの形状チェック
- **session-binding**（レジストリベース）- セッションバインディングの動作
- **outbound-payload** - メッセージペイロードの構造と正規化
- **group-policy**（フォールバック）- チャネルごとのデフォルトグループポリシーの適用
- **threading**（レジストリベース、シャーディング済み）- スレッド ID の処理
- **directory**（レジストリベース、シャーディング済み）- ディレクトリ／名簿 API
- **registry** および **plugins-core.\*** - チャネル Plugin レジストリ、ローダー、設定書き込み認可の内部処理

これらのスイートで使用する受信ディスパッチキャプチャおよび送信ペイロードのハーネスヘルパーは、
`src/plugin-sdk/channel-contract-testing.ts` を通じて内部公開されています
（npm から除外されており、公開 SDK サブパスではありません）。このディレクトリには独立した
`inbound.contract.test.ts` ファイルはありません。

### プロバイダー契約

`src/plugins/contracts/*.contract.test.ts` にあります。現在のカテゴリには
以下が含まれます：

- **shape** - Plugin マニフェスト、API、ランタイムエクスポートの形状
- **plugin-registration**（＋並列）- マニフェスト登録のケース
- **package-manifest** - パッケージマニフェストの要件
- **loader** - Plugin ローダーのセットアップ／破棄の動作
- **registry** - Plugin 契約レジストリの内容と検索
- **providers** - バンドルされたプロバイダー全体で共有されるプロバイダー動作、およびウェブ検索プロバイダー
- **auth-choice** - 認証選択肢のメタデータとセットアップ動作
- **provider-catalog-deprecation** - 非推奨プロバイダーカタログのメタデータ
- **wizard.choice-resolution**、**wizard.model-picker**、**wizard.setup-options** - プロバイダーセットアップウィザードの契約
- **embedding-provider**、**memory-embedding-provider**、**web-fetch-provider**、**tts** - 機能固有のプロバイダー契約
- **session-actions**、**session-attachments**、**session-entry-projection** - Plugin 所有のセッション状態契約
- **scheduled-turns** - Plugin のスケジュール済みターンのメタデータとタイムスタンプ境界
- **host-hooks**、**run-context-lifecycle**、**runtime-import-side-effects**、**runtime-seams** - Plugin ホスト／ランタイムのライフサイクルとインポート境界の契約
- **extension-runtime-dependencies** - 拡張機能のランタイム依存関係の配置

### 実行するタイミング

- plugin-sdk のエクスポートまたはサブパスを変更した後
- チャネルまたはプロバイダー Plugin を追加または変更した後
- Plugin の登録または検出をリファクタリングした後

契約テストは CI で実行され、実際の API キーは不要です。

## 回帰テストの追加（ガイダンス）

ライブ環境で発見されたプロバイダー／モデルの問題を修正する場合：

- 可能であれば CI セーフな回帰テストを追加します（モック／スタブプロバイダー、または正確なリクエスト形状の変換をキャプチャ）
- 本質的にライブ限定の場合（レート制限、認証ポリシー）は、ライブテストを限定的に保ち、環境変数によるオプトイン方式にします
- バグを検出できる最小のレイヤーを対象にすることを優先します：
  - プロバイダーのリクエスト変換／リプレイのバグ -> models の直接テスト
  - Gateway のセッション／履歴／ツールパイプラインのバグ -> Gateway のライブスモークテストまたは CI セーフな Gateway モックテスト
- SecretRef トラバーサルのガードレール：
  - `src/secrets/exec-secret-ref-id-parity.test.ts` はレジストリメタデータ（`listSecretTargetRegistryEntries()`）から SecretRef クラスごとにサンプル対象を 1 つ導出し、トラバーサルセグメントを含む実行 ID が拒否されることを検証します。
  - `src/secrets/target-registry-data.ts` に新しい `includeInPlan` SecretRef ターゲットファミリーを追加する場合は、そのテストの `classifyTargetClass` を更新してください。新しいクラスが暗黙的にスキップされないよう、このテストは分類されていないターゲット ID に対して意図的に失敗します。

## 関連項目

- [ライブテスト](/ja-JP/help/testing-live)
- [更新と Plugin のテスト](/ja-JP/help/testing-updates-plugins)
- [CI](/ja-JP/ci)
