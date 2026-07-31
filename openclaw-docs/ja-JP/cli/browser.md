---
read_when:
    - '`openclaw browser` を使用しており、一般的なタスクの例を確認したい場合'
    - Node ホスト経由で別のマシン上で動作しているブラウザを制御する場合
    - Chrome MCP を介して、ローカルでログイン済みの Chrome に接続する場合
summary: '`openclaw browser` の CLI リファレンス（ライフサイクル、プロファイル、タブ、アクション、状態、デバッグ）'
title: ブラウザ
x-i18n:
    generated_at: "2026-07-26T09:55:41Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 62eb41248cda87cef96be7b0dfe3e0d36a9d3e1ee55c165bd8e3efd68d1e9a5e
    source_path: cli/browser.md
    workflow: 16
---

# `openclaw browser`

OpenClaw のブラウザ制御サーフェスを管理し、ライフサイクル、プロファイル、タブ、スナップショット、スクリーンショット、ナビゲーション、入力、状態エミュレーション、デバッグなどのブラウザ操作を実行します。

関連項目: [ブラウザツール](/ja-JP/tools/browser)

## 共通フラグ

- `--url <gatewayWsUrl>`: Gateway WebSocket URL（デフォルトは設定値）。
- `--token <token>`: Gateway トークン（必要な場合）。
- `--timeout <ms>`: リクエストのタイムアウト（ミリ秒、デフォルト: `30000`）。
- `--expect-final`: Gateway の最終応答を待機します。
- `--browser-profile <name>`: ブラウザプロファイルを選択します（デフォルト: `openclaw`、または `browser.defaultProfile`）。
- `--json`: 機械可読な出力（サポートされている場合）。これはブラウザレベルのオプションであるため、
  `openclaw browser --json status` のように、曖昧さのない形式にするにはサブコマンドの前に配置します。
  選択した子コマンドが独自の `--json` を定義していない場合は、
  `openclaw browser status --json` のように末尾へ配置しても機能します。

## クイックスタート（ローカル）

```bash
openclaw browser profiles
openclaw browser --browser-profile openclaw start
openclaw browser --browser-profile openclaw open https://example.com
openclaw browser --browser-profile openclaw snapshot
```

エージェントは `browser({ action: "doctor" })` を使用して同じ準備状況チェックを実行できます。

## クイックトラブルシューティング

`start` が `not reachable after start` で失敗する場合は、まず CDP の準備状況をトラブルシューティングしてください。`start` と `tabs` が成功しても `open` または `navigate` が失敗する場合、ブラウザ制御プレーンは正常であり、通常はナビゲーションの SSRF ポリシーによるブロックが原因です。

最小限の手順:

```bash
openclaw browser --browser-profile openclaw doctor
openclaw browser --browser-profile openclaw start
openclaw browser --browser-profile openclaw tabs
openclaw browser --browser-profile openclaw open https://example.com
```

詳細なガイダンス: [ブラウザのトラブルシューティング](/ja-JP/tools/browser#cdp-startup-failure-vs-navigation-ssrf-block)

## ライフサイクル

```bash
openclaw browser status
openclaw browser doctor
openclaw browser doctor --deep
openclaw browser start
openclaw browser start --headless
openclaw browser stop
openclaw browser --browser-profile openclaw reset-profile
```

- `doctor --deep` はライブスナップショットプローブを追加します。基本的な CDP の準備状況は正常でも、現在のタブを検査できることを確認したい場合に役立ちます。
- 実行中のローカル管理対象プロファイルでは、`status` と `doctor` が Chrome からキャッシュされた
  グラフィックス診断を報告します。これには、ハードウェア／ソフトウェアの分類、レンダラー、
  バックエンド、デバイス／ドライバー、機能と無効化状態の詳細、アクセラレーション対応の
  動画機能が含まれます。`openclaw browser --json status` は完全な構造化ペイロードを返します。
  パッシブなステータス確認では、これらの情報を収集するためだけに Chrome を起動することはありません。
- `stop` は、OpenClaw 自体がブラウザプロセスを起動していない `attachOnly` およびリモート CDP プロファイルでも、アクティブな制御セッションを閉じ、一時的なエミュレーションのオーバーライドを消去します。ローカル管理対象プロファイルでは、`stop` は起動したブラウザプロセスも停止します。
- `start --headless` はその起動リクエストにのみ適用され、OpenClaw がローカル管理対象ブラウザを起動する場合に限られます。`browser.headless` やプロファイル設定を書き換えることはなく、すでに実行中のブラウザに対しては何も行いません。
- `DISPLAY` または `WAYLAND_DISPLAY` がない Linux ホストでは、`OPENCLAW_BROWSER_HEADLESS=0`、`browser.headless=false`、または `browser.profiles.<name>.headless=false` が表示ブラウザを明示的に要求しない限り、ローカル管理対象プロファイルは自動的にヘッドレスで実行されます。

## コマンドが見つからない場合

`openclaw browser` が不明なコマンドである場合は、`~/.openclaw/openclaw.json` の `plugins.allow` を確認してください。`plugins.allow` が存在する場合、設定にルートの `browser` ブロックがすでにない限り、バンドルされたブラウザ Plugin を明示的に一覧へ追加します。

```json5
{
  plugins: {
    allow: ["telegram", "browser"],
  },
}
```

ルートに明示的な `browser` ブロック（たとえば `browser.enabled=true` または `browser.profiles.<name>`）がある場合も、制限付き Plugin 許可リストのもとでバンドルされたブラウザ Plugin が有効になります。

関連項目: [ブラウザツール](/ja-JP/tools/browser#missing-browser-command-or-tool)

## プロファイル

プロファイルは、名前付きのブラウザルーティング設定です。

- `openclaw`（デフォルト）: OpenClaw が管理する専用の Chrome インスタンスを起動するか接続します（ユーザーデータディレクトリは分離されます）。
- `user`: Chrome DevTools MCP を介して、ログイン済みの既存の Chrome セッションを制御します。
- カスタム CDP プロファイル: ローカルまたはリモートの CDP エンドポイントを指定します。

```bash
openclaw browser profiles
openclaw browser system-profiles
openclaw browser system-profiles --browser brave
openclaw browser import-profile --browser chrome --system Default --into imported
openclaw browser import-profile --system "Profile 1" --into work --domains google.com,youtube.com
openclaw browser create-profile --name work --color "#FF5A36"
openclaw browser create-profile --name chrome-live --driver existing-session
openclaw browser create-profile --name remote --cdp-url https://browser-host.example.com
openclaw browser delete-profile --name work
```

任意のサブコマンドで `--browser-profile <name>` を使用して特定のプロファイルを指定できます。例: `openclaw browser --browser-profile work tabs`。

macOS では、`system-profiles` はホスト上で利用可能な実際の Chrome、Brave、Edge、または Chromium のプロファイルを一覧表示します。`import-profile` は、macOS キーチェーン／Touch ID の同意プロンプトを一度表示した後に Cookie を復号し、新しい OpenClaw 管理対象プロファイルへ注入します。インポートされるのは Cookie のみで、ローカルストレージと IndexedDB は変更されません。一部の Google セッションではデバイスにバインドされたセッション認証情報（DBSC）が使用されるため、インポート後も再認証が必要になる場合があります。

macOS アプリがローカル Gateway を使用している場合、このインポートを一度提示し、分離されたインポート済みプロファイルをエージェントのブラウジング用デフォルトに設定できます。インポートには必ず明示的なクリックが必要です。インポートの成功またはダイアログの終了後は、それ以降の自動プロンプトが抑制され、再インポートには **Settings → General → Browser login** を引き続き使用できます。

システムプロファイルのインポートはデフォルトで有効です。CLI とエージェントが開始するインポートの両方を無効にするには、`browser.allowSystemProfileImport=false` を設定します。インポートはホストローカルであり、ブラウザ Node プロキシを介して実行することはできません。

## タブ

```bash
openclaw browser tabs
openclaw browser tab new --label docs
openclaw browser tab label t1 docs
openclaw browser tab select 2
openclaw browser tab close 2
openclaw browser open https://docs.openclaw.ai --label docs
openclaw browser focus docs
openclaw browser close t1
```

`tabs` は最初に `suggestedTargetId`、続いて安定した `tabId`（`t1` など）、任意のラベル、生の `targetId` を返します。`suggestedTargetId` を `focus`、`close`、スナップショット、およびアクションへ渡してください。`open --label`、`tab new --label`、または `tab label` でラベルを割り当てます。ラベル、タブ ID、生のターゲット ID、一意なターゲット ID プレフィックスのいずれも使用できます。互換性のため、リクエストフィールドの名前は引き続き `targetId` ですが、これらのタブ参照をすべて受け付けます。

生のターゲット ID は揮発性の診断用ハンドルであり、永続的なエージェントメモリではありません。ナビゲーションまたはフォーム送信中に Chromium が基盤となる生のターゲットを置き換えた場合、OpenClaw は一致を確認できれば、安定した `tabId`／ラベルを置換後のタブに維持します。`suggestedTargetId` を優先してください。

## スナップショット／スクリーンショット／アクション

スナップショット:

```bash
openclaw browser snapshot
openclaw browser snapshot --urls
```

スクリーンショット:

```bash
openclaw browser screenshot
openclaw browser screenshot --full-page
openclaw browser screenshot --ref e12
openclaw browser screenshot --labels
```

- `--full-page` はページキャプチャ専用です。`--ref` または `--element` と組み合わせることはできません。
- `existing-session`／`user` プロファイルでは、ページのスクリーンショットと、スナップショット出力の `--ref` のスクリーンショットをサポートしますが、CSS `--element` のスクリーンショットはサポートしません。
- `--labels` は、現在のスナップショット参照をスクリーンショット上にオーバーレイします。Playwright ベースのプロファイルでは、`--full-page`（ページ全体のオーバーレイ）、`--ref`（ARIA 参照による要素クリップのオーバーレイ）、`--element`（CSS セレクターによる要素クリップのオーバーレイ）で機能します。要素クリップモードでは、ラベルは要素を基準に投影されます。応答には、各参照の境界ボックスを含む `annotations` 配列（空の場合は省略）も含まれます。キャプチャ画像の座標空間（ビューポート／ページ全体／要素相対）における `ref`、`number`、`role`、任意の `name`、および `box: {x, y, width, height}` が格納されます。
  `existing-session` プロファイルは、ページのスクリーンショット上に chrome-mcp オーバーレイをレンダリングしますが、Playwright の投影ヘルパーは使用せず、`annotations` も含みません。CSS `--element` のスクリーンショットはサポートされません。Playwright または chrome-mcp がない場合、ラベル付きスクリーンショットは利用できません。
- `snapshot --urls` は、検出されたリンク先を AI スナップショットに追加します。これにより、エージェントはリンクテキストだけから推測せず、直接のナビゲーション先を選択できます。

移動／クリック／入力（参照ベースの UI 自動化）:

```bash
openclaw browser navigate https://example.com
openclaw browser click <ref>
openclaw browser click-coords 120 340
openclaw browser type <ref> "hello"
openclaw browser press Enter
openclaw browser hover <ref>
openclaw browser scrollintoview <ref>
openclaw browser drag <startRef> <endRef>
openclaw browser select <ref> OptionA OptionB
openclaw browser fill --fields '[{"ref":"1","value":"Ada"}]'
openclaw browser wait --text "Done"
openclaw browser evaluate --fn '(el) => el.textContent' --ref <ref>
openclaw browser evaluate --fn 'const title = document.title; return title;'
openclaw browser evaluate --timeout-ms 30000 --fn 'async () => { await window.ready; return true; }'
```

`evaluate --fn` は、関数ソース、式、または文の本体を受け付けます。文の本体は非同期関数としてラップされるため、返したい値には `return` を使用します。ページ側の関数がデフォルトの評価タイムアウトより長い時間を必要とする可能性がある場合は、`--timeout-ms` を使用します。`browser.evaluateEnabled=false`（デフォルト: `true`）は、`evaluate` と `wait --fn` の両方を無効にします。

アクションの応答は、OpenClaw が置換後のタブを確認できる場合、アクションによるページ置換後の現在の生の `targetId` を返します。長時間実行するワークフローでは、スクリプトは引き続き `suggestedTargetId`／ラベルを保存して渡す必要があります。

ファイルとダイアログのヘルパー:

```bash
openclaw browser upload /tmp/openclaw/uploads/file.pdf --ref <ref>
openclaw browser upload media://inbound/file.pdf --ref <ref>
openclaw browser waitfordownload
openclaw browser download <ref> report.pdf
openclaw browser dialog --accept
openclaw browser dialog --dismiss --dialog-id d1
```

管理対象の Chrome プロファイルでは、通常のクリックで開始されたダウンロードを OpenClaw のダウンロードディレクトリ（デフォルトでは `/tmp/openclaw/downloads`、または設定された一時ルート）に保存します。エージェントが特定のファイルを待機してそのパスを返す必要がある場合は、`waitfordownload` または `download` を使用します。これらの明示的な待機処理は、次のダウンロードを占有します。アップロードでは、OpenClaw の一時アップロードルートおよび OpenClaw が管理する受信メディアにあるファイルを使用できます。これには、`media://inbound/<id>` とサンドボックス相対の `media/inbound/<id>` 参照が含まれます。ネストされたメディア参照、パストラバーサル、任意のローカルパスは拒否されます。

アクションによってモーダルダイアログが開かれた場合、アクションの応答は `browserState.dialogs.pending` を持つ `blockedByDialog` を返します。直接応答するには `--dialog-id` を渡します。OpenClaw の外部で処理されたダイアログは `browserState.dialogs.recent` に表示されます。

一括アクション:

```bash
openclaw browser batch --actions '[{"kind":"wait","timeMs":500},{"kind":"click","ref":"12"},{"kind":"type","ref":"23","text":"hello"}]'
openclaw browser batch --actions-file plan.json
openclaw browser batch --actions-file - --continue
```

`openclaw browser batch` は、ネストされた `BrowserActRequest` アクション（`wait`、`click`、`type`、`evaluate`、...）を含む `kind="batch"` `/act` リクエストを送信します。`open`/`navigate`/`snapshot`/`screenshot` は `/act` の種類ではなく CLI サブコマンドであるため、これらは送信しません。`--continue` は `stopOnError=false` を設定します（デフォルトでは最初のエラーで停止します）。`--target-id` はバッチ全体のスコープを 1 つのタブに限定します。ネストされたアクションが失敗すると、コマンドは 0 以外で終了します。順序付けられた `results` レスポンスを保持するには、`--json` を使用します。完全な契約（ref のライフサイクル、ターゲット ID の競合、エラーの概要）については、[ブラウザバッチ CLI](/ja-JP/tools/browser-control#browser-batch-cli)を参照してください。`batch` は `profile="user"` / 既存セッションプロファイルではサポートされていません。

## 状態とストレージ

ビューポート + エミュレーション:

```bash
openclaw browser resize 1280 720
openclaw browser set viewport 1280 720
openclaw browser set offline on
openclaw browser set media dark
openclaw browser set timezone Europe/London
openclaw browser set locale en-GB
openclaw browser set geo 51.5074 -0.1278 --accuracy 25
openclaw browser set device "iPhone 14"
openclaw browser set headers '{"x-test":"1"}'
openclaw browser set credentials myuser mypass
```

Cookie + ストレージ:

```bash
openclaw browser cookies
openclaw browser cookies set session abc123 --url https://example.com
openclaw browser cookies clear
openclaw browser storage local get
openclaw browser storage local set token abc123
openclaw browser storage session clear
```

## デバッグ

```bash
openclaw browser console --level error
openclaw browser pdf
openclaw browser responsebody "**/api"
openclaw browser highlight <ref>
openclaw browser errors --clear
openclaw browser requests --filter api
openclaw browser trace start
openclaw browser trace stop --out trace.zip
```

## MCP 経由の既存の Chrome

組み込みの `user` プロファイルを使用するか、独自の `existing-session` プロファイルを作成します。

```bash
openclaw browser --browser-profile user tabs
openclaw browser create-profile --name chrome-live --driver existing-session
openclaw browser create-profile --name brave-live --driver existing-session --user-data-dir "~/Library/Application Support/BraveSoftware/Brave-Browser"
openclaw browser create-profile --name chrome-port --driver existing-session --cdp-url http://127.0.0.1:9222
openclaw browser --browser-profile chrome-live tabs
```

デフォルトの既存セッションパスは、ホスト専用の Chrome MCP 自動接続です。ブラウザが DevTools エンドポイントですでに実行されている場合は、`--cdp-url` を渡すと、Chrome MCP は代わりにそのエンドポイントへ接続します。Chrome MCP のセマンティクスが不要な Docker、Browserless、またはその他のリモート構成では、代わりに CDP プロファイルを使用します。

現在の既存セッションの制限:

- スナップショット駆動のアクションでは、CSS セレクターではなく ref を使用します。
- サポートされている `act` リクエストでは、呼び出し元が `timeoutMs` を省略した場合、組み込みのデフォルト値 60000 ms が使用されます。呼び出しごとの `timeoutMs` が指定されている場合は、引き続きそちらが優先されます。
- `click` は左クリックのみです。
- `type` は `slowly=true` をサポートしていません。
- `press` は `delayMs` をサポートしていません。
- `hover`、`scrollintoview`、`drag`、`select`、および `fill` は、呼び出しごとのタイムアウト上書きを拒否します。`evaluate` は `--timeout-ms` を受け付けます。
- `select` は 1 つの値のみをサポートします。
- `wait --load networkidle` はサポートされていません（管理対象および raw/リモート CDP プロファイルでは動作します）。
- ファイルのアップロードには `--ref` / `--input-ref` が必要です。CSS `--element` はサポートされず、一度に 1 つのファイルのみをサポートします。
- ダイアログフックは `--timeout` をサポートしていません。
- スクリーンショットはページキャプチャと `--ref` をサポートしますが、CSS `--element` はサポートしません。
- `responsebody`、ダウンロードのインターセプト、PDF エクスポート、およびバッチアクションには、引き続き管理対象ブラウザまたは raw CDP プロファイルが必要です。

## リモートブラウザ制御（Node ホストプロキシ）

Gateway がブラウザとは異なるマシンで実行されている場合は、Chrome/Brave/Edge/Chromium があるマシン上で **Node ホスト**を実行します。Gateway はブラウザアクションをその Node にプロキシします。別個のブラウザ制御サーバーは不要です。

自動ルーティングを制御するには `gateway.nodes.browser.mode` を使用し、複数の Node が接続されている場合に特定の Node を固定するには `gateway.nodes.browser.node` を使用します。

セキュリティ + リモート設定: [ブラウザツール](/ja-JP/tools/browser)、[リモートアクセス](/ja-JP/gateway/remote)、[Tailscale](/ja-JP/gateway/tailscale)、[セキュリティ](/ja-JP/gateway/security)

## 関連項目

- [CLI リファレンス](/ja-JP/cli)
- [ブラウザ](/ja-JP/tools/browser)
