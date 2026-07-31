---
read_when:
    - OpenClaw のバグに対するライブビジュアル QA の構築または実行
    - プルリクエストに変更前後の検証を追加する
    - Discord、Slack、WhatsApp、その他のライブトランスポートシナリオの追加
    - 候補 ref に対して対象を絞った Control UI ブラウザ検証を実行する
    - スクリーンショット、ブラウザ自動化、または VNC アクセスが必要な QA 実行のデバッグ
summary: Mantis は、ライブトランスポートの比較と候補のみを対象としたブラウザでの重点的な検証のために、視覚的なエンドツーエンドの証跡を取得し、その成果物を PR に添付します。
title: Mantis
x-i18n:
    generated_at: "2026-07-26T10:11:04Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 48a1b306e37aba7e8c67139df61f3680a9aec066361aa196d88c81270337bc1b
    source_path: concepts/mantis.md
    workflow: 16
---

Mantis は、OpenClaw の動作に関する視覚的な CI エビデンスと PR コメントを公開します。
ライブトランスポートシナリオでは、既知の不良ベースラインと候補 ref を比較します。
一方、対象を絞ったブラウザレーンでは、決定論的なモックトランスポートに対して
1 つの候補のみを検証する場合があります。Discord は、実際の bot 認証、guild チャンネル、
リアクション、スレッド、ブラウザによる確認を備えて最初にリリースされました。Slack、Telegram、および対象を絞った Control
UI チャットレーンも存在します。WhatsApp と Matrix は未実装です。

## 所有範囲

- OpenClaw（`extensions/qa-lab/src/mantis/*`）：シナリオランタイム、`pnpm openclaw qa mantis <command>` CLI、エビデンススキーマ。
- QA Lab（`extensions/qa-lab/src/live-transports/*`）：ライブトランスポートハーネス、ドライバー/SUT bot、レポート/エビデンスライター。
- Crabbox（`openclaw/crabbox`）：ウォームアップ済みの Linux マシン、リース、VNC、`crabbox media preview`。
- GitHub Actions（`.github/workflows/mantis-*.yml`）：リモートエントリポイント、アーティファクト保持。
- ClawSweeper：メンテナーの PR コマンドを解析し、ワークフローをディスパッチして、最終的な PR コメントを投稿します。

## CLI コマンド

すべてのコマンドは `pnpm openclaw qa mantis <command>` であり、
`extensions/qa-lab/src/mantis/cli.ts` で定義されています。ビルド時および実行時に `OPENCLAW_ENABLE_PRIVATE_QA_CLI=1` が必要です
（同梱のワークフローは、ビルド前に `OPENCLAW_BUILD_PRIVATE_QA=1` と
`OPENCLAW_ENABLE_PRIVATE_QA_CLI=1` を設定します）。

| コマンド                         | 目的                                                                                                                                                   |
| ------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `discord-smoke`                 | Mantis Discord bot が guild/チャンネルを認識し、投稿とリアクションを実行できることを検証します。                                                                                 |
| `run`                           | ベースラインと候補の ref に対して前後比較シナリオを実行します（Discord のみ）。                                                                           |
| `desktop-browser-smoke`         | Crabbox デスクトップをリースまたは再利用し、表示可能なブラウザを開いて、スクリーンショットと動画をキャプチャします。                                                                        |
| `slack-desktop-smoke`           | Crabbox デスクトップをリースまたは再利用し、その中で Slack QA を実行して Slack Web を開き、エビデンスをキャプチャします。                                                                  |
| `telegram-desktop-builder`      | Crabbox デスクトップをリースまたは再利用し、Telegram Desktop をインストールして、必要に応じて OpenClaw Gateway を設定します。                                                        |
| `visual-task` / `visual-driver` | オプションの画像理解アサーションを備えた汎用 Crabbox デスクトップキャプチャです。`visual-driver` は、`crabbox record --while` 配下で起動されるドライバー側です。 |

すべてのコマンドは `--repo-root <path>` と `--output-dir <path>` を受け付けます。Crabbox
コマンドはさらに、`--crabbox-bin`、`--provider`、`--machine-class`/`--class`、
`--lease-id`、`--idle-timeout`、`--ttl`、`--keep-lease` も受け付けます。特記がない限り、provider/class のローカル CLI デフォルトは
`hetzner`/`beast` です。CI ワークフローでは通常、両方を上書きします。

### `discord-smoke`

```bash
pnpm openclaw qa mantis discord-smoke \
  --output-dir .artifacts/qa-e2e/mantis/discord-smoke
```

Discord REST API（`https://discord.com/api/v10`）を呼び出して bot
ユーザー、guild、guild のチャンネル、および対象チャンネルを取得し、そのチャンネルが
guild に属することをアサートします。その後、`--skip-post` でない限り、メッセージを投稿して
`👀` リアクションを追加します。`mantis-discord-smoke-summary.json` と
`mantis-discord-smoke-report.md` を書き込みます。

トークンの解決順序は、`--token-file` の値、次に `OPENCLAW_QA_DISCORD_MANTIS_BOT_TOKEN`
（`--token-env` で上書き）、その次に `OPENCLAW_QA_DISCORD_MANTIS_BOT_TOKEN_FILE` で指定されたファイル
（`--token-file-env` で上書き）です。guild/チャンネル ID は
`OPENCLAW_QA_DISCORD_GUILD_ID` / `OPENCLAW_QA_DISCORD_CHANNEL_ID`（
`--guild-id` / `--channel-id` で上書き）から取得し、17～20 桁の Discord snowflake でなければなりません。
公開される概要とレポート内で bot/guild/チャンネル/メッセージの ID
および名前を `<redacted>` に置き換えるには、`OPENCLAW_QA_REDACT_PUBLIC_METADATA=1` を設定します。

### `run`

```bash
pnpm openclaw qa mantis run \
  --transport discord \
  --scenario discord-status-reactions-tool-only \
  --baseline origin/main \
  --candidate HEAD \
  --output-dir .artifacts/qa-e2e/mantis/local-discord-status-reactions
```

`--transport` は現在、`discord` のみを受け付けます。`--scenario` は 2 つある
組み込み ID のいずれかで、それぞれ独自のデフォルトベースライン ref と、想定される前後の
ラベル（`extensions/qa-lab/src/mantis/run.runtime.ts`）があります。

| シナリオ                                   | デフォルトベースライン                           | ベースラインの想定                         | 候補の想定            |
| ------------------------------------------ | ------------------------------------------ | ---------------------------------------- | ---------------------------- |
| `discord-status-reactions-tool-only`       | `0bf06e953fdda290799fc9fb9244a8f67fdae593` | `queued-only`                            | `queued -> thinking -> done` |
| `discord-thread-reply-filepath-attachment` | `81349cdc2a9d5143fd0991ed858b739e7d96e05c` | スレッド返信に `filePath` 添付ファイルが含まれない | スレッド返信に含まれる     |

`--candidate` のデフォルトは `HEAD` です。その他のフラグは、`--credential-source`
（デフォルト `convex`）、`--credential-role`（デフォルト `ci`）、`--provider-mode`
（デフォルト `live-frontier`）、`--fast`（デフォルトで有効）、`--skip-install`、`--skip-build` です。

ランナーは、ベースラインと候補について分離された `git worktree` チェックアウトを
`<output-dir>/worktrees/` 配下に作成し、それぞれで `pnpm install`/`pnpm build` を
実行し（スキップされない場合）、その後、各 worktree に対して
`pnpm openclaw qa discord --scenario <id> --model openai/gpt-5.4 --alt-model openai/gpt-5.4 --allow-failures`
を実行します。各レーンは `discord-qa-reaction-timelines.json`
と `<scenario-id>-timeline.html`/`.png` のペアを書き込みます。ランナーはこの
エビデンスを `baseline/`/`candidate/` 配下へコピーし、出力ディレクトリに `comparison.json`、
`mantis-report.md`、`mantis-evidence.json` を書き込み、
比較が合格しなかった場合（ベースラインが `fail`、候補が
`pass`）は非ゼロで終了します。

2 番目の Discord シナリオ（`discord-thread-reply-filepath-attachment`）は、
ドライバー bot で親メッセージを投稿し、実際のスレッドを作成して、リポジトリローカルの `filePath` を使用して SUT の
`message.thread-reply` アクションを呼び出し、その後、返信と添付ファイル名を確認するために
スレッドをポーリングします。`mantis-thread-report.md` という名前の添付ファイルを想定します。

### `desktop-browser-smoke`

```bash
pnpm openclaw qa mantis desktop-browser-smoke \
  --output-dir .artifacts/qa-e2e/mantis/desktop-browser
```

Crabbox デスクトップをリースまたは再利用し、VNC セッション内でブラウザを起動して
`--browser-url`（デフォルト `https://openclaw.ai`）またはレンダリングされた
`--html-file` を表示し、待機して、`scrot` でスクリーンショットを取得します。必要に応じて
`ffmpeg` で MP4 を録画し、`desktop-browser-smoke.png` / `.mp4` / `remote-metadata.json`
を `--output-dir` へ rsync で戻します。

フラグ：

- `--lease-id <cbx_...>` は、新規作成せずにウォームアップ済みのデスクトップを再利用します。
- `--browser-profile-dir <remote-path>` は、リモートの Chrome user-data-dir を再利用し、永続デスクトップが実行間でログイン状態を維持できるようにします（長期間使用する Discord Web ビューアープロファイルに使用）。
- `--browser-profile-archive-env <name>` は、起動前にその環境変数から base64 の `.tgz` Chrome プロファイルアーカイブを復元します（デフォルト `OPENCLAW_MANTIS_BROWSER_PROFILE_TGZ_B64`）。Discord Web のようなログイン済み確認環境に使用します。
- `--video-duration <seconds>` は MP4 のキャプチャ時間を制御します（デフォルト 10 秒）。
- `--keep-lease`（または `OPENCLAW_MANTIS_KEEP_VM=1`）は、この実行で作成したリースを VNC 検査用に開いたままにします。リースを作成した実行が失敗した場合も、デフォルトでリースを保持します。

Discord Web のエビデンスには、Mantis は bot
トークンではなく専用のビューアーアカウントを使用します。Discord REST オラクル（`qa discord` 経由）は引き続き信頼できる基準です。`OPENCLAW_QA_DISCORD_CAPTURE_UI_METADATA=1` が設定されている場合、シナリオは
Discord Web URL アーティファクトも書き込み、`OPENCLAW_QA_DISCORD_KEEP_THREADS=1` は
ブラウザがスレッドを開けるだけの時間、スレッドを開いたままにします。

GitHub ワークフローでは、
`MANTIS_DISCORD_VIEWER_CHROME_PROFILE_DIR` による永続ビューアープロファイルを優先します（完全なプロファイルアーカイブは
GitHub のシークレットサイズ上限を超える可能性があります）。小規模なプロファイルやブートストラップ用プロファイルの場合は、代わりに
`MANTIS_DISCORD_VIEWER_CHROME_PROFILE_TGZ_B64` から base64 の `.tgz` を復元できます。どちらのソースも
設定されていない場合でも、ワークフローは決定論的な
ベースライン/候補のスクリーンショットを公開し、ログイン済みの確認が
スキップされたことをログに記録します。

### `slack-desktop-smoke`

```bash
pnpm openclaw qa mantis slack-desktop-smoke \
  --output-dir .artifacts/qa-e2e/mantis/slack-desktop \
  --gateway-setup \
  --scenario slack-canary \
  --keep-lease
```

Crabbox デスクトップをリースまたは再利用し、チェックアウトを VM に同期して、その中で
`pnpm openclaw qa slack` を実行し、VNC ブラウザで Slack Web を開き、
デスクトップをキャプチャして、Slack QA アーティファクト（`slack-qa/`）と
VNC のスクリーンショット/動画の両方をローカルへコピーします。これは、
SUT Gateway とブラウザの両方が同じ VM 内で実行される唯一の Mantis 形式です。

`--gateway-setup` を指定すると、コマンドは VM 内の
`$HOME/.openclaw-mantis/slack-openclaw` に永続的で破棄可能な OpenClaw
ホームを作成し、対象チャンネル向けの Slack
Socket Mode 設定をパッチして、
`openclaw gateway run --dev --allow-unconfigured --port 38973` を起動し、
Chrome を VNC セッション内で実行したままにします。`--gateway-setup` を省略すると、代わりに通常の
bot 間 Slack QA レーンを実行します。

`--credential-source env` に必要な環境変数（ローカルのデフォルトは `env`、role
のデフォルトは `maintainer`）：

- `OPENCLAW_QA_SLACK_CHANNEL_ID`
- `OPENCLAW_QA_SLACK_DRIVER_BOT_TOKEN`
- `OPENCLAW_QA_SLACK_SUT_BOT_TOKEN`
- `OPENCLAW_QA_SLACK_SUT_APP_TOKEN`
- `OPENCLAW_LIVE_OPENAI_KEY` はリモートモデルレーン用です（ローカルで `OPENAI_API_KEY`
のみが設定されている場合、Mantis は Crabbox を
呼び出す前にそれを `OPENCLAW_LIVE_OPENAI_KEY` へコピーします）

`--credential-source convex` を指定すると、Mantis は VM を作成する前に
共有プールから Slack SUT 認証情報をリースし、チャンネル ID、app token、bot token を
`OPENCLAW_MANTIS_SLACK_*` 環境変数として VM に転送します。そのため、GitHub
ワークフローに必要なのは Convex broker secret のみで、生の Slack token は不要です。

その他のフラグ：`--slack-url <url>` は特定の URL を開きます（指定しない場合、Mantis は
`auth.test` から `https://app.slack.com/client/<team>/<channel>` を導出します）。
`--slack-channel-id <id>` は Gateway の許可リストチャンネルを設定します。
`OPENCLAW_MANTIS_SLACK_BROWSER_PROFILE_DIR` は VM 内の永続 Chrome
プロファイルを制御します（デフォルト `$HOME/.config/openclaw-mantis/slack-chrome-profile`）。
`--approval-checkpoints` はネイティブ Slack 承認シナリオ
（`slack-approval-exec-native`、`slack-approval-plugin-native`）を実行し、
Gateway セットアップの代わりに保留中/解決済みのチェックポイントスクリーンショットをレンダリングします
（`--gateway-setup` とは同時に指定できません）。`--hydrate-mode source|prehydrated`、
`--provider-mode`、`--model`、`--alt-model`、`--fast` は
Slack ライブレーンにそのまま渡されます。

承認チェックポイントのスクリーンショットは、ライブの Slack UI ではなく、
シナリオが観測した Slack API メッセージからレンダリングされます。`slack-desktop-smoke.png` が
Slack Web 自体の証拠となるのは、リースのブラウザプロファイルがすでにログイン済みだった場合のみです。

### `telegram-desktop-builder`

```bash
pnpm openclaw qa mantis telegram-desktop-builder \
  --credential-source convex \
  --credential-role maintainer \
  --keep-lease
```

Crabbox デスクトップをリースまたは再利用し、ネイティブ Linux Telegram Desktop をインストールして、
必要に応じてユーザーセッションアーカイブを復元し、リースした Telegram SUT bot token で OpenClaw を設定して、
`openclaw gateway run --dev --allow-unconfigured --port 38974` を起動し、
ドライバー bot の準備完了メッセージをリースしたプライベートグループに投稿してから、
スクリーンショットと MP4 をキャプチャします。bot token は OpenClaw の設定にのみ使用され、
Telegram Desktop へのログインには決して使用されません。デスクトップビューアーは別の Telegram ユーザーセッションであり、
`--telegram-profile-archive-env <name>` から復元するか、
VNC 経由で手動ログインし、`--keep-lease` によって稼働状態を維持します。

フラグ：`--lease-id <cbx_...>` は、Telegram Desktop にすでにログイン済みの VM に対して再実行します。
`--telegram-profile-archive-env <name>` は、起動前に base64 の
`.tgz` プロファイルアーカイブを復元します。`--telegram-profile-dir <remote-path>`
はリモートプロファイルディレクトリを設定します（デフォルト `$HOME/.local/share/TelegramDesktop`）。
`--no-gateway-setup` は Telegram Desktop のインストールと起動のみを行います。
`--credential-source`/`--credential-role` のデフォルトは `convex`/`maintainer` です。

## エビデンスマニフェスト

PR に公開するすべてのシナリオは、レポートの隣に `mantis-evidence.json` を書き込みます。

```json
{
  "schemaVersion": 1,
  "id": "discord-status-reactions",
  "title": "Mantis Discord ステータスリアクション QA",
  "summary": "PR コメント用の人間が読める冒頭サマリー。",
  "scenario": "discord-status-reactions-tool-only",
  "comparison": {
    "baseline": { "sha": "...", "status": "fail", "expected": "queued-only" },
    "candidate": { "sha": "...", "status": "pass", "expected": "queued -> thinking -> done" },
    "pass": true
  },
  "artifacts": [
    {
      "kind": "timeline",
      "lane": "baseline",
      "label": "ベースラインは queued-only",
      "path": "baseline/timeline.png",
      "targetPath": "baseline.png",
      "alt": "ベースラインの Discord タイムライン",
      "width": 420
    }
  ]
}
```

アーティファクトの `path` はマニフェストのディレクトリを基準とし、`targetPath` は設定された R2/S3 アーティファクトプレフィックスを基準とします。`scripts/mantis/publish-pr-evidence.mjs` はパストラバーサルを拒否し、ファイルが見つからない場合は `"required": false` のエントリをスキップします。

アーティファクトの種類：`timeline`（決定論的な変更前／変更後のスクリーンショット）、`desktopScreenshot`（VNC／ブラウザーのスクリーンショット）、`motionPreview`（録画から生成したインラインアニメーション GIF）、`motionClip`（動きのない部分をトリミングした MP4）、`fullVideo`（完全な録画）、`metadata`（JSON／ログサイドカー）、`report`（Markdown レポート）。

実行時のディスク上のアーティファクトレイアウト：

```text
.artifacts/qa-e2e/mantis/<run-id>/
  mantis-report.md
  mantis-evidence.json
  baseline/
  candidate/
  comparison.json
```

スクリーンショットは証拠でありシークレットではありませんが、それでも墨消しを徹底する必要があります。非公開チャンネル名、ユーザー名、またはメッセージ内容が含まれる可能性があります。公開アーティファクトのアップロードでは `OPENCLAW_QA_REDACT_PUBLIC_METADATA=1` を設定してください。Discord／Slack／Telegram の GitHub ワークフローではデフォルトで有効です。

## GitHub 自動化

`scripts/mantis/publish-pr-evidence.mjs` は再利用可能なパブリッシャーです。ワークフローは、マニフェスト、対象 PR、アーティファクトの対象ルート、コメントマーカー、アーティファクト URL、実行 URL、リクエスト元を指定してこれを呼び出します。宣言されたアーティファクトを Mantis R2 バケットにアップロードし、冒頭にサマリーを置いた PR コメントをインライン画像／プレビューおよびリンク付き動画とともに作成してから、既存のマーカーコメントを更新するか、新しいコメントを作成します。必須の環境変数：

- `MANTIS_ARTIFACT_R2_ACCESS_KEY_ID`
- `MANTIS_ARTIFACT_R2_SECRET_ACCESS_KEY`
- `MANTIS_ARTIFACT_R2_BUCKET`（ワークフローは `openclaw-crabbox-artifacts` を設定）
- `MANTIS_ARTIFACT_R2_ENDPOINT`
- `MANTIS_ARTIFACT_R2_REGION`（ワークフローは `auto` を設定）
- `MANTIS_ARTIFACT_R2_PUBLIC_BASE_URL`（ワークフローは `https://artifacts.openclaw.ai` を設定）

コメントは `github-actions[bot]` ではなく、Mantis GitHub App（`MANTIS_GITHUB_APP_ID`／`MANTIS_GITHUB_APP_PRIVATE_KEY`）を通じて投稿され、非表示のマーカーコメントを upsert キーとして使用します。

| ワークフロー                          | トリガー                                                                                    | 実行内容                                                                                                                                                                                                                                                                                                     |
| --------------------------------- | ------------------------------------------------------------------------------------------ | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `Mantis Discord Smoke`            | 手動ディスパッチ                                                                            | 選択した ref に対して `discord-smoke` を実行します。                                                                                                                                                                                                                                                                       |
| `Mantis Discord Status Reactions` | PR コメントまたは手動ディスパッチ                                                              | ベースラインと候補のワークツリーを個別に構築し、それぞれで `discord-status-reactions-tool-only` を実行し、各レーンのタイムラインを Crabbox デスクトップブラウザーでレンダリングし、`crabbox media preview` で動きのない部分をトリミングした GIF／MP4 プレビューを生成し、アーティファクトをアップロードして、インラインの PR 証拠を投稿します。                                 |
| `Mantis Scenario`                 | 手動ディスパッチ                                                                            | 汎用ディスパッチャー：`scenario_id`（`discord-status-reactions-tool-only`、`discord-thread-reply-filepath-attachment`、`slack-desktop-smoke`、`telegram-live`、`telegram-desktop-proof`、`web-ui-chat-proof`）、`baseline_ref`、`candidate_ref`、`pr_number` を受け取り、対応するシナリオワークフローに転送します。 |
| `Mantis Slack Desktop Smoke`      | 手動ディスパッチ                                                                            | Crabbox Linux デスクトップをリースし（デフォルトは `aws`、`hetzner` から選択可能）、候補に対して `slack-desktop-smoke --gateway-setup` を実行し、デスクトップを録画し、モーションプレビューを生成して、アーティファクトをアップロードします。PR 番号が指定されている場合は PR 証拠を投稿します。                                                      |
| `Mantis Telegram Live`            | PR コメントまたは手動ディスパッチ                                                              | bot API の Telegram ライブ QA レーン（`openclaw qa telegram`）を実行し、QA サマリーから `mantis-evidence.json` を書き込み、墨消し済みの証拠 HTML を Crabbox デスクトップブラウザーでレンダリングし、モーション GIF を生成して、PR 証拠を投稿します。このレーンでは Telegram Web へのログインは不要です。                               |
| `Mantis Telegram Desktop Proof`   | メンテナーの PR ラベル（`mantis: telegram-visible-proof`）と PR コメント、または手動ディスパッチ | エージェントによるネイティブ Telegram Desktop の変更前／変更後の証明。PR、ベースライン／候補の ref、メンテナーの指示を Codex に渡します。Codex は両方の ref に対して実ユーザーの Crabbox Telegram Desktop 証明レーンを実行し、2 列の PR 証拠表を投稿します。                                                              |
| `Mantis Web UI Chat Proof`        | PR コメントまたは手動ディスパッチ                                                              | 候補に対して対象を絞った OpenClaw Control UI チャットの Playwright 証明を実行し、ブラウザーがモック化された Gateway を通じて送信することを検証し、スクリーンショット／動画アーティファクトを取得して、PR 証拠を投稿します。このレーンは Web チャットのみを証明するものであり、WinUI／ネイティブアプリや任意のビジュアル証明を対象としません。                           |

`Mantis Discord Status Reactions` と `Mantis Telegram Live` はどちらも `baseline_ref`／`candidate_ref`（または PR コメント内の `baseline=`／`candidate=`）を受け取り、シークレットを含む認証情報を使用して実行する前に、解決された SHA が `origin/main` の祖先、リリースタグ（`v*`）、またはオープンな PR の head のいずれかであることを検証します。

write／maintain／admin アクセスを持つユーザーが PR から使用できるコメントトリガー：

```text
@openclaw-mantis discord status reactions
@openclaw-mantis discord status reactions baseline=origin/main candidate=HEAD
@openclaw-mantis telegram
@openclaw-mantis telegram scenario=telegram-status-command
@openclaw-mantis telegram scenarios=telegram-status-command,channel-canary
@openclaw-mantis web ui chat
@openclaw-mantis web-ui-chat candidate=HEAD
```

Telegram のコメントトリガーでは、デフォルトで PR の head SHA を候補、`telegram-status-command` をシナリオとして使用します。特定の Crabbox プロバイダーまたは事前にウォームアップされたデスクトップを対象にするため、`provider=aws|hetzner` と `lease=<cbx_...>` を指定できます。`Mantis Telegram Desktop Proof` は、PR に `mantis: telegram-visible-proof` ラベルがすでに付いている場合にのみ PR コメントへ応答します。

Web UI チャットのコメントトリガーでは、デフォルトで PR の head SHA を候補として使用します。Control UI のモック Gateway チャット証明を実行し、ブラウザーアーティファクトを公開します。ほかの Web ページやネイティブアプリのサーフェスでは、通常の Playwright／ブラウザー証明、メンテナーのスクリーンショット、Crabbox、またはローカルアーティファクトを使用してください。

ClawSweeper からシナリオを直接ディスパッチすることもできます。

```text
@clawsweeper mantis discord discord-status-reactions-tool-only
```

## マシンとシークレット

ローカル CLI の Crabbox のデフォルトは `--provider hetzner --class beast` です。`--provider`、`--class`／`--machine-class`、または `OPENCLAW_MANTIS_CRABBOX_PROVIDER`／`OPENCLAW_MANTIS_CRABBOX_CLASS` で上書きできます。GitHub ワークフローでは通常、両方を上書きします（例：`--class standard`、および Slack ワークフローの `aws`／`hetzner` プロバイダー選択入力）。プロバイダーが遅すぎるか利用できない場合は、フォールバックをハードコードするのではなく、同じ Crabbox インターフェースの背後に追加してください。

VM のベースライン：デスクトップ対応の Chrome／Chromium、CDP アクセス、VNC／noVNC、Node 22.22.3+、24.15+、または 25.9+ と pnpm、OpenClaw のチェックアウトを備え、対象トランスポート、GitHub、モデルプロバイダー、認証情報ブローカーへのアウトバウンドアクセスが可能な Linux。

Mantis のコマンドとワークフロー全体で使用される認証情報および環境変数名：

- `OPENCLAW_QA_DISCORD_MANTIS_BOT_TOKEN`
- `OPENCLAW_QA_DISCORD_GUILD_ID`
- `OPENCLAW_QA_DISCORD_CHANNEL_ID`
- ローカルの `qa mantis run --credential-source env` には、
  `OPENCLAW_QA_DISCORD_DRIVER_BOT_TOKEN`、`OPENCLAW_QA_DISCORD_SUT_BOT_TOKEN`、
  `OPENCLAW_QA_DISCORD_SUT_APPLICATION_ID` も必要です。GitHub ワークフローでは通常、生の
  Discord bot トークンの代わりに `--credential-source convex` と以下のブローカー認証情報を使用します。
- 公開アーティファクトのアップロード用の `OPENCLAW_QA_REDACT_PUBLIC_METADATA=1`
- `OPENCLAW_QA_CONVEX_SITE_URL`、`OPENCLAW_QA_CONVEX_SECRET_CI`
- `OPENAI_API_KEY`（または Telegram Desktop 証明専用の
  `OPENCLAW_MANTIS_AGENT_OPENAI_API_KEY`）
- `CRABBOX_COORDINATOR`／`CRABBOX_COORDINATOR_TOKEN`（ワークフローでは
  フォールバックとして `OPENCLAW_QA_MANTIS_CRABBOX_COORDINATOR`／`_TOKEN` も受け取り、
  Crabbox を呼び出す前にプレーンな名前へマッピングします）
- `CRABBOX_ACCESS_CLIENT_ID`、`CRABBOX_ACCESS_CLIENT_SECRET`
- `MANTIS_GITHUB_APP_ID`、`MANTIS_GITHUB_APP_PRIVATE_KEY`

Mantis ランナーは、Discord／Slack／Telegram の bot トークン、プロバイダー API キー、ブラウザー Cookie、認証プロファイルの内容、VNC パスワード、または生の認証情報ペイロードを決して出力してはなりません。トークンが issue、PR、チャット、またはログに漏洩した場合は、代替シークレットを保存した後にローテーションしてください。

## 実行結果

変更前／変更後のトランスポートシナリオでは、不安定な環境が製品のリグレッションと解釈されないよう、次の結果を区別します。

- **バグを再現**：ベースラインがシナリオの想定どおりに失敗しました。
- **ハーネス障害**：オラクルが意味を持つ前に、環境のセットアップ、認証情報、トランスポート API、ブラウザー、
  またはプロバイダーが失敗しました。

候補のみのブラウザー証明は、候補がモック化された Gateway と可視 UI アサーションに合格したかどうかを報告します。ベースラインを再現したとは主張しません。

## シナリオの追加

ライブトランスポートシナリオは、独立した宣言型ファイル形式ではなく、トランスポートごとに TypeScript で定義します（Discord の変更前／変更後の形式については、`extensions/qa-lab/src/mantis/run.runtime.ts` の `MANTIS_SCENARIO_CONFIGS` を参照）。各シナリオには、ID とタイトル、トランスポート、必須の認証情報、ベースラインの ref ポリシー、候補の ref ポリシー、OpenClaw 設定パッチ、セットアップ／刺激ステップ、想定されるベースラインと候補のオラクル、ビジュアル取得対象、タイムアウト枠、クリーンアップ手順が必要です。

対象を絞った候補のみのブラウザー証明では、専用の決定論的 E2E テストとワークフローを使用できます。スコープを明示し、実行前に候補の ref を検証し、シークレットを使用する公開処理を分離して、同じ証拠マニフェスト契約を出力してください。

ビジョンチェックよりも、小さく型付けされたオラクルを優先してください。たとえば、Discord のリアクション状態またはメッセージ参照、Slack スレッドの `ts`／リアクション API 状態、メールのメッセージ ID とヘッダーです。UI が唯一の信頼できる観測対象である場合はブラウザーのスクリーンショットを使用し、プラットフォーム API のオラクルが存在する場合は、ビジョンチェックをそれに追加する形にしてください。

Discord、Slack、Telegram に続き、同じランナー形式を WhatsApp（QR ログイン、再識別、配信、メディア、リアクション）および Matrix（暗号化ルーム、スレッド／返信関係、再起動後の再開）へ拡張できますが、どちらもまだ実装されていません。

## 未解決の課題

- 既存の Mantis ボットを再利用する場合、どの Discord ボットをドライバーとし、どのボットを SUT とすべきですか？
- GitHub は PR の Mantis アーティファクトをどのくらいの期間保持すべきですか？
- ClawSweeper は、メンテナーのコマンドを待たずに、どのような場合に Mantis シナリオを自動的に推奨すべきですか？
- 公開 PR にアップロードする前に、スクリーンショットを墨消しまたは切り抜きすべきですか？
