---
read_when:
    - GitHub またはローカルから Mantis Slack デスクトップ QA を実行する
    - Mantis Slack デスクトップ実行の遅延をデバッグする
    - ソース、事前ハイドレーション済み、またはウォームリースモードの選択
    - PR にスクリーンショットと動画の証拠を投稿する
summary: Mantis Slack デスクトップ QA の運用手順書：GitHub ディスパッチ、ローカル CLI、ウォーム VNC リース、ハイドレートモード、所要時間の解釈、成果物、障害対応。
title: Mantis Slack デスクトップ運用手順書
x-i18n:
    generated_at: "2026-07-26T09:58:41Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: b3e956d99fc43a7b6fe65e2e820812b0e0e8b9e32badd25be27c74d302ab30dc
    source_path: concepts/mantis-slack-desktop-runbook.md
    workflow: 16
---

Mantis Slack デスクトップ QA は、Linux デスクトップ、VNC による復旧、Slack Web、実際の OpenClaw Gateway、スクリーンショット、動画、PR の証拠コメントが必要な Slack 系のバグ向け実 UI レーンです。単体テストやヘッドレスの Slack ライブレーンではバグを証明できない場合に使用します。

## ストレージモデル

Mantis は 3 つのストレージ層を使用します。

- **プロバイダーイメージ** - Crabbox が所有し、クラウドプロバイダーのアカウントに保存されます。
  マシンの機能（Chrome/Chromium、ffmpeg、scrot、
  Node/corepack/pnpm、ネイティブビルドツール）と空のキャッシュディレクトリを保持します。
- **ウォームリース状態** - 現在のオペレーターセッションが所有します。リースの存続中は、
  ログイン済みのブラウザプロファイル、`/var/cache/crabbox/pnpm`、準備済みのソース
  チェックアウトを保持できます。
- **Mantis アーティファクト** - OpenClaw の実行が所有します。
  `.artifacts/qa-e2e/mantis/...` 配下に存在し、GitHub Actions がアップロードし、Mantis
  GitHub App が PR にインラインの証拠をコメントします。

シークレット、ブラウザ Cookie、Slack のログイン状態、リポジトリのチェックアウト、
`node_modules`、`dist/` をプロバイダーイメージに組み込んではなりません。

## GitHub ディスパッチ

`main` からワークフローを実行します。

```bash
gh workflow run mantis-slack-desktop-smoke.yml \
  --ref main \
  -f candidate_ref=<trusted-ref-or-sha> \
  -f pr_number=<pr-number> \
  -f scenario_id=slack-canary \
  -f crabbox_provider=aws \
  -f keep_vm=false \
  -f hydrate_mode=source
```

ワークフローはライブ認証情報を使用するため、`candidate_ref` は制限されています。現在の
`main` の祖先、リリースタグ、または `openclaw/openclaw` にあるオープンな PR の head に
解決される必要があります。

ワークフローは以下を生成します。

- アップロードされたアーティファクト `mantis-slack-desktop-smoke-<run-id>-<attempt>`
- Mantis GitHub App からのインライン PR コメント
- `slack-desktop-smoke.png`、`slack-desktop-smoke.mp4`
- `slack-desktop-smoke-preview.gif`、`slack-desktop-smoke-change.mp4`
- `mantis-slack-desktop-smoke-summary.json`、`mantis-slack-desktop-smoke-report.md`
- リモートログ: `slack-desktop-command.log`、`openclaw-gateway.log`、`chrome.log`、`ffmpeg.log`

PR コメントは非表示の `<!-- mantis-slack-desktop-smoke -->` マーカーを介して同じ場所で更新されます。

## ローカル CLI

コールドなソース証明:

```bash
pnpm openclaw qa mantis slack-desktop-smoke \
  --provider aws \
  --class standard \
  --gateway-setup \
  --credential-source convex \
  --credential-role maintainer \
  --provider-mode live-frontier \
  --model openai/gpt-5.4 \
  --alt-model openai/gpt-5.4 \
  --scenario slack-canary \
  --hydrate-mode source
```

VNC による復旧のために VM を保持します。

```bash
pnpm openclaw qa mantis slack-desktop-smoke \
  --provider aws \
  --class standard \
  --gateway-setup \
  --scenario slack-canary \
  --keep-lease
```

VNC を開きます。

```bash
crabbox vnc --provider aws --id <cbx_id> --open
```

ウォームリースを再利用します。

```bash
pnpm openclaw qa mantis slack-desktop-smoke \
  --provider aws \
  --lease-id <cbx_id-or-slug> \
  --gateway-setup \
  --scenario slack-canary \
  --hydrate-mode source
```

再利用するリモートワークスペースに `node_modules` とビルド済みの
`dist/` がすでにある場合にのみ `--hydrate-mode prehydrated` を使用してください。それ以外の場合、Mantis は安全側に倒して失敗します。

ネイティブの Slack 承認 UI を証明します。

```bash
pnpm openclaw qa mantis slack-desktop-smoke \
  --provider aws \
  --class standard \
  --approval-checkpoints \
  --credential-source convex \
  --credential-role maintainer \
  --hydrate-mode source
```

`--approval-checkpoints` と `--gateway-setup` は同時に使用できません。明示的な承認チェックポイント
`--scenario` を渡さない限り、オプトインの `slack-approval-exec-native` と
`slack-approval-plugin-native` のシナリオを実行します。その他の Slack
シナリオは VM の起動前に拒否されます。Slack QA ランナーは観測した実際の Slack API
メッセージから各チェックポイントの JSON ファイルを書き込み、その後、リモートウォッチャーがそのメッセージを
`approval-checkpoints/<scenario>-pending.png` と
`approval-checkpoints/<scenario>-resolved.png` にレンダリングします。
チェックポイント JSON、メッセージの証拠、ack JSON、レンダリングされたスクリーンショットのいずれかが存在しないか空の場合、実行は失敗します。

コールドな GitHub Actions リースには Slack Web の Cookie がないため、ブラウザキャプチャが
Slack のサインイン画面になる場合があります。承認チェックポイントの証明では、
`slack-desktop-smoke.png` ではなく、レンダリングされたチェックポイント画像と Slack QA
アーティファクトを信頼してください。ブラウザのスクリーンショット自体に Slack Web
を表示する必要がある場合にのみ、Slack Web に手動でログインしたプロファイルを持つ、保持中のウォームリースを使用してください。

## ハイドレートモード

| モード          | 使用する場合                                  | リモートでの動作                                                                       | トレードオフ                                                 |
| ------------- | ----------------------------------------- | ------------------------------------------------------------------------------------- | -------------------------------------------------------- |
| `source`      | 通常の PR 証明、コールドマシン、CI        | VM 内で `pnpm install --frozen-lockfile --prefer-offline` と `pnpm build` を実行 | 最も低速ですが、ソースチェックアウトによる証明は最も強力です                 |
| `prehydrated` | 再利用するリースを意図的に準備した場合 | 既存の `node_modules` と `dist/` が必要で、インストールとビルドをスキップします                     | 高速ですが、オペレーターが管理するウォームリースでのみ有効です |

GitHub Actions は VM の実行前に候補のチェックアウトを常に準備します。その
pnpm ストアは OS、Node のバージョン、ロックファイル別にキャッシュされます。VM の `source` の実行も、
`/var/cache/crabbox/pnpm` が存在する場合は再利用します。

## タイミングの解釈

`mantis-slack-desktop-smoke-report.md` にはフェーズ別の所要時間が含まれます。

- `crabbox.warmup` - クラウドプロバイダーの起動、デスクトップとブラウザの準備、SSH。
- `crabbox.inspect` - リースメタデータの検索。
- `credentials.prepare` - Convex 認証情報リースの取得。
- `crabbox.remote_run` - 同期、ブラウザの起動、OpenClaw のインストールとビルドまたは
  ハイドレート検証、Gateway の起動、スクリーンショット、動画キャプチャ。
- `artifacts.copy` - VM からの rsync によるコピー。

Crabbox がゼロ以外のリモートステータスを返しても、OpenClaw Gateway
のセットアップが完了したか、Slack QA コマンド自体が正常終了したことを証明するメタデータを Mantis がコピーした場合、
`crabbox.remote_run` に `accepted` と表示されることがあります。
`accepted` はシナリオの失敗ではなく、説明付きの成功として扱ってください。

実行が遅い場合:

- ウォームアップが支配的: より適切な Crabbox プロバイダーイメージを事前作成または昇格してください。
- `source` で `remote_run` が支配的: ウォームリースを使用するか、pnpm ストアの
  再利用を改善するか、マシンの前提条件をプロバイダーイメージに移してください。
- `prehydrated` で `remote_run` が支配的: リモートワークスペースが
  実際には準備できていないか、Gateway、ブラウザ、Slack のセットアップが遅くなっています。
- アーティファクトのコピーが支配的: 動画のサイズとアーティファクトディレクトリの内容を確認してください。

## 証拠チェックリスト

適切な PR コメントには以下が表示されます。

- シナリオ ID と候補 SHA
- GitHub Actions の実行 URL とアーティファクト URL
- インラインの承認チェックポイントのスクリーンショット、またはログイン済みのウォームリースから取得した
  Slack Web のスクリーンショット
- 利用可能な場合はインラインのアニメーションプレビュー
- 完全版 MP4 とトリミング済み MP4 のリンク
- 成功または失敗のステータスとレポートのタイミング概要

スクリーンショットや動画をリポジトリにコミットしないでください。GitHub
Actions のアーティファクトまたは PR コメント内に保持してください。

## 障害処理

VM の実行前にワークフローが失敗した場合は、最初に Actions ジョブを確認してください。
一般的な原因は、信頼されていない `candidate_ref`、環境シークレットの欠落、候補の
インストールまたはビルドの失敗です。

VM の実行が失敗してもスクリーンショットがコピーされた場合は、以下を確認してください。

```bash
cat mantis-slack-desktop-smoke-report.md
cat mantis-slack-desktop-smoke-summary.json
cat slack-desktop-command.log
cat openclaw-gateway.log
cat chrome.log
cat ffmpeg.log
```

実行でリースを保持した場合は、レポートの `crabbox vnc ...`
コマンドで VNC を開き、完了したらリースを停止します。

```bash
crabbox stop --provider aws <cbx_id-or-slug>
```

Slack のログインが期限切れの場合は、保持中のリースで VNC を使用して修復し、
`--lease-id` で再実行してください。そのブラウザプロファイルをプロバイダーイメージに組み込まないでください。

## 関連項目

- [QA の概要](/ja-JP/concepts/qa-e2e-automation)
- [Slack チャンネル](/ja-JP/channels/slack)
- [テスト](/ja-JP/help/testing)
