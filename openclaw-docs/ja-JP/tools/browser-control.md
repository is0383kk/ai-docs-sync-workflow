---
read_when:
    - ローカル制御 API を介したエージェントブラウザのスクリプト操作またはデバッグ
    - '`openclaw browser` CLI リファレンスをお探しですか'
    - スナップショットと参照を使用したカスタムブラウザ自動化の追加
summary: OpenClaw ブラウザ制御 API、CLI リファレンス、スクリプトアクション
title: ブラウザ制御 API
x-i18n:
    generated_at: "2026-07-26T10:21:51Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 812358a5ad366e419413b78507d3620ea9f3981224bc8cc62fb512b87eaadd9b
    source_path: tools/browser-control.md
    workflow: 16
---

セットアップ、設定、トラブルシューティングについては、[ブラウザー](/ja-JP/tools/browser)を参照してください。
このページは、ローカル制御 HTTP API、`openclaw browser`
CLI、およびスクリプト作成パターン（スナップショット、参照、待機、デバッグフロー）のリファレンスです。

## 制御 API（任意）

ローカル統合専用として、Gateway は小規模な loopback HTTP API を公開します。
このスタンドアロンサーバーはオプトインです。Gateway サービス環境で環境変数
`OPENCLAW_EAGER_BROWSER_CONTROL_SERVER=1` を設定し、
HTTP エンドポイントが使用可能になる前に Gateway を再起動してください。この変数がなくても、
ブラウザー制御ランタイムは CLI とエージェントツールを通じて引き続き動作しますが、
loopback 制御ポートでは何もリッスンしません。

- ステータス/開始/停止: `GET /`, `GET /doctor`, `POST /start`, `POST /stop`, `POST /reset-profile`
- プロファイル: `GET /profiles`, `POST /profiles/create`, `DELETE /profiles/:name`
- タブ: `GET /tabs`, `POST /tabs/open`, `POST /tabs/focus`, `DELETE /tabs/:targetId`, `POST /tabs/action`
- スナップショット/スクリーンショット: `GET /snapshot`, `POST /screenshot`
- アクション: `POST /navigate`, `POST /act`
- フック: `POST /hooks/file-chooser`, `POST /hooks/dialog`
- ダウンロード: `POST /download`, `POST /wait/download`
- 権限: `POST /permissions/grant`
- デバッグ: `GET /console`, `POST /pdf`
- デバッグ: `GET /errors`, `GET /requests`, `GET /dialogs`, `POST /trace/start`, `POST /trace/stop`, `POST /highlight`
- ネットワーク: `POST /response/body`
- 状態: `GET /cookies`, `POST /cookies/set`, `POST /cookies/clear`
- 状態: `GET /storage/:kind`, `POST /storage/:kind/set`, `POST /storage/:kind/clear`
- 設定: `POST /set/offline`, `POST /set/headers`, `POST /set/credentials`, `POST /set/geolocation`, `POST /set/media`, `POST /set/timezone`, `POST /set/locale`, `POST /set/device`

`POST /tabs/action` は、CLI が
`browser tab` サブコマンド（`{"action":"new"|"label"|"select"|"close"|"list", ...}`）のために内部的に使用するバッチ形式です。
直接スクリプトを作成する場合は、上記の単一目的のタブルートを使用してください。

すべてのエンドポイントは `?profile=<name>` を受け入れます。`POST /start?headless=true` は、
永続化されたブラウザー設定を変更せずに、ローカルの管理対象プロファイルを一度だけヘッドレスで起動するよう要求します。
OpenClaw がそれらのブラウザープロセスを起動しないため、接続専用、リモート CDP、および既存セッションのプロファイルでは
このオーバーライドは拒否されます。

タブエンドポイントでは、`targetId` が互換性用のフィールド名です。
`GET /tabs` または `POST /tabs/open` から得た
`suggestedTargetId` を渡すことを推奨します。ラベルと、`t1` などの
`tabId` ハンドルも使用できます。生の CDP ターゲット ID と一意な生の
ターゲット ID プレフィックスも引き続き機能しますが、これらは揮発性の診断用ハンドルです。

共有シークレットによる Gateway 認証が設定されている場合、ブラウザー HTTP ルートにも認証が必要です。

- `Authorization: Bearer <gateway token>`
- `x-openclaw-password: <gateway password>`、またはそのパスワードを使用した HTTP Basic 認証

注意:

- このスタンドアロンの loopback ブラウザー API は、信頼済みプロキシまたは
  Tailscale Serve の ID ヘッダーを使用**しません**。
- `gateway.auth.mode` が `none` または `trusted-proxy` の場合、これらの loopback ブラウザー
  ルートはそのような ID を伴うモードを継承しません。loopback 専用のままにしてください。

### `/act` エラー契約

`POST /act` は、ルートレベルの検証および
ポリシー違反に対して構造化されたエラーレスポンスを使用します。

```json
{ "error": "<message>", "code": "ACT_*" }
```

現在の `code` 値:

- `ACT_KIND_REQUIRED`（HTTP 400）: `kind` がないか、認識されません。
- `ACT_INVALID_REQUEST`（HTTP 400）: アクションのペイロードの正規化または検証に失敗しました。
- `ACT_SELECTOR_UNSUPPORTED`（HTTP 400）: サポートされていないアクション種別で `selector` が使用されました。
- `ACT_EVALUATE_DISABLED`（HTTP 403）: `evaluate`（または `wait --fn`）が設定により無効化されています。
- `ACT_TARGET_ID_MISMATCH`（HTTP 403）: トップレベルまたはバッチ処理された `targetId` がリクエストターゲットと競合しています。
- `ACT_EXISTING_SESSION_UNSUPPORTED`（HTTP 501）: このアクションは既存セッションのプロファイルではサポートされていません。

その他のランタイム障害では、`code` フィールドなしで
`{ "error": "<message>" }` が返される場合があります。

### Playwright の要件

一部の機能（移動/操作/AI スナップショット/ロールスナップショット、要素のスクリーンショット、
PDF）には Playwright が必要です。Playwright がインストールされていない場合、これらのエンドポイントは
明確な 501 エラーを返します。

Playwright なしでも機能するもの:

- ARIA スナップショット
- タブごとの CDP WebSocket が利用可能な場合の、ロール形式のアクセシビリティスナップショット（`--interactive`、`--compact`、
  `--depth`、`--efficient`）。これは調査と参照検出のための
  フォールバックです。主要なアクションエンジンは引き続き Playwright です。
- タブごとの CDP
  WebSocket が利用可能な場合の、管理対象 `openclaw` ブラウザーのページスクリーンショット
- `existing-session` / Chrome MCP プロファイルのページスクリーンショット
- スナップショット出力からの `existing-session` 参照ベースのスクリーンショット（`--ref`）

引き続き Playwright が必要なもの:

- `navigate`
- `act`
- Playwright ネイティブの AI スナップショット形式に依存する AI スナップショット
- CSS セレクターによる要素のスクリーンショット（`--element`）
- ブラウザー全体の PDF エクスポート

要素のスクリーンショットでは `--full-page` も拒否され、ルートは `fullPage is
not supported for element screenshots` を返します。

`Playwright is not available in this gateway build` が表示される場合、パッケージ化された
Gateway にコアブラウザーランタイムの依存関係がありません。OpenClaw を再インストールまたは更新し、
Gateway を再起動してください。Docker の場合は、以下に示すように Chromium
ブラウザーバイナリもインストールしてください。

#### Docker への Playwright のインストール

Gateway を Docker で実行している場合は、`npx playwright`（npm オーバーライドの競合）を避けてください。
カスタムイメージでは、Chromium をイメージに組み込んでください。

```bash
OPENCLAW_INSTALL_BROWSER=1 ./scripts/docker/setup.sh
```

既存のイメージでは、代わりに同梱の CLI を通じてインストールしてください。

```bash
docker compose run --rm openclaw-cli \
  node /app/node_modules/playwright-core/cli.js install chromium
```

ブラウザーのダウンロードを永続化するには、`PLAYWRIGHT_BROWSERS_PATH`（例:
`/home/node/.cache/ms-playwright`）を設定し、`/home/node` が
`OPENCLAW_HOME_VOLUME` またはバインドマウントを通じて永続化されていることを確認してください。OpenClaw は Linux 上で、
永続化された Chromium を自動検出します。[Docker](/ja-JP/install/docker)を参照してください。

## 仕組み（内部）

小規模な loopback 制御サーバーが HTTP リクエストを受け入れ、CDP を介して Chromium ベースのブラウザーに接続します。高度なアクション（クリック/入力/スナップショット/PDF）は、CDP 上の Playwright を通じて実行されます。Playwright がない場合は、Playwright を使用しない操作のみ利用できます。ローカル/リモートブラウザーやプロファイルが内部で自由に切り替わっても、エージェントからは単一の安定したインターフェースとして見えます。

## CLI クイックリファレンス

すべてのコマンドは、特定のプロファイルを対象にするための `--browser-profile <name>` と、機械可読な出力のための `--json` を受け入れます。

<AccordionGroup>

<Accordion title="基本: ステータス、タブ、開く/フォーカス/閉じる">

```bash
openclaw browser status
openclaw browser doctor
openclaw browser doctor --deep    # ライブスナップショットプローブを追加
openclaw browser start
openclaw browser start --headless # ローカルの管理対象ブラウザーを一度だけヘッドレスで起動
openclaw browser stop            # 接続専用/リモート CDP でのエミュレーションもクリア
openclaw browser reset-profile   # プロファイルのブラウザーデータをゴミ箱に移動
openclaw browser tabs
openclaw browser tab             # 現在のタブへのショートカット
openclaw browser tab new
openclaw browser tab new --label research
openclaw browser tab label abcd1234 research
openclaw browser tab select 2
openclaw browser tab close 2
openclaw browser open https://example.com
openclaw browser focus abcd1234
openclaw browser close abcd1234
```

</Accordion>

<Accordion title="プロファイル: 一覧、作成、削除">

```bash
openclaw browser profiles
openclaw browser create-profile --name research --color "#0066CC"
openclaw browser create-profile --name attach --driver existing-session --cdp-url http://127.0.0.1:9222
openclaw browser delete-profile --name research
```

</Accordion>

<Accordion title="調査: スクリーンショット、スナップショット、コンソール、エラー、リクエスト">

```bash
openclaw browser screenshot
openclaw browser screenshot --full-page
openclaw browser screenshot --ref 12        # またはロール参照の場合は --ref e12
openclaw browser screenshot --labels
openclaw browser snapshot
openclaw browser snapshot --format aria --limit 200
openclaw browser snapshot --interactive --compact --depth 6
openclaw browser snapshot --efficient
openclaw browser snapshot --labels
openclaw browser snapshot --urls
openclaw browser snapshot --selector "#main" --interactive
openclaw browser snapshot --frame "iframe#main" --interactive
openclaw browser snapshot --out snapshot.txt
openclaw browser console --level error
openclaw browser errors --clear
openclaw browser requests --filter api --clear
openclaw browser pdf
openclaw browser responsebody "**/api" --max-chars 5000
```

</Accordion>

<Accordion title="アクション: 移動、クリック、入力、ドラッグ、待機、評価">

```bash
openclaw browser navigate https://example.com
openclaw browser resize 1280 720
openclaw browser click 12 --double           # ロール参照の場合は e12 も使用可能
openclaw browser click-coords 120 340        # ビューポート座標
openclaw browser type 23 "hello" --submit
openclaw browser press Enter
openclaw browser hover 44
openclaw browser scrollintoview e12
openclaw browser drag 10 11
openclaw browser select 9 OptionA OptionB
openclaw browser download e12 report.pdf
openclaw browser waitfordownload report.pdf
openclaw browser upload /tmp/openclaw/uploads/file.pdf
openclaw browser upload /tmp/openclaw/uploads/file.pdf --ref e12
openclaw browser upload media://inbound/file.pdf
openclaw browser fill --fields '[{"ref":"1","type":"text","value":"Ada"}]'
openclaw browser dialog --accept
openclaw browser dialog --dismiss --dialog-id d1
openclaw browser wait --text "Done"
openclaw browser wait "#main" --url "**/dash" --load networkidle --fn "window.ready===true"
openclaw browser evaluate --fn '(el) => el.textContent' --ref 7
openclaw browser evaluate --fn 'const title = document.title; return title;'
openclaw browser evaluate --timeout-ms 30000 --fn 'async () => { await window.ready; return true; }'
openclaw browser highlight e12
openclaw browser trace start
openclaw browser trace stop
```

</Accordion>

<Accordion title="状態: Cookie、ストレージ、オフライン、ヘッダー、位置情報、デバイス">

```bash
openclaw browser cookies
openclaw browser cookies set session abc123 --url "https://example.com"
openclaw browser cookies clear
openclaw browser storage local get
openclaw browser storage local set theme dark
openclaw browser storage session clear
openclaw browser set offline on
openclaw browser set headers --headers-json '{"X-Debug":"1"}'
openclaw browser set credentials user pass            # 削除するには --clear
openclaw browser set geo 37.7749 -122.4194 --origin "https://example.com"
openclaw browser set media dark
openclaw browser set timezone America/New_York
openclaw browser set locale en-US
openclaw browser set device "iPhone 14"
```

</Accordion>

</AccordionGroup>

注意:

- エージェント向けの `browser` ツールは、`action=download`（必須の `ref` と
  `path`）および `action=waitfordownload`（任意の `path`）を公開します。どちらも、保存された
  ダウンロード URL、推奨ファイル名、保護されたローカルパスを返します。明示的なダウンロード
  インターセプトは、管理対象の Playwright プロファイルで利用できます。既存セッションの
  プロファイルでは、未サポート操作エラーが返されます。
- アトミックなチューザーアップロードを推奨します。アップロードとともにトリガー `--ref` を渡すと、OpenClaw が 1 回のリクエストで準備とクリックを行います。後で意図的にトリガーする場合は、パスのみの `upload` も引き続きサポートされます。ファイル入力を直接設定するには、`--input-ref` または `--element` を使用します。`dialog` は準備呼び出しです。ダイアログをトリガーするクリック／キー押下の前に実行してください。アクションによってモーダルが開く場合、アクションレスポンスには `blockedByDialog` と `browserState.dialogs.pending` が含まれます。直接応答するには、その `dialogId` を渡します。OpenClaw の外部で処理されたダイアログは、`browserState.dialogs.recent` の下に表示されます。
- `click`/`type`/などには、`snapshot` から取得した `ref`（数値の `12`、ロール参照 `e12`、または操作可能な ARIA 参照 `ax12`）が必要です。アクションでは CSS セレクターを意図的にサポートしていません。表示中のビューポート位置だけが信頼できるターゲットである場合は、`click-coords` を使用します。
- ダウンロードパスとトレースパスは、OpenClaw の一時ルート `/tmp/openclaw{,/downloads}`（フォールバック：`${os.tmpdir()}/openclaw/...`）内に制限されます。
- `upload` は、OpenClaw の一時アップロードルートにあるファイルと、
  OpenClaw が管理する受信メディアを受け付けます。管理対象の受信メディアは、
  `media://inbound/<id>`、サンドボックス相対の `media/inbound/<id>`、または管理対象の
  受信メディアディレクトリ内で解決されたパスとして参照できます。ネストされたメディア参照、
  トラバーサル、シンボリックリンク、ハードリンク、任意のローカルパスは引き続き拒否されます。
- `upload` でも、`--input-ref` または `--element` を使用してファイル入力を直接設定できます。

OpenClaw が置換後のタブを特定できる場合、安定したタブ ID とラベルは Chromium の raw ターゲット置換後も維持されます。たとえば、同じ URL に対して旧タブと新タブの組み合わせが一意である場合や、フォーム送信後に単一の旧タブが単一の新タブになった場合です。URL が重複して置換先が曖昧な場合は、新しいハンドルが割り当てられます。raw ターゲット ID は引き続き変動するため、スクリプトでは `tabs` の `suggestedTargetId` を使用してください。

スナップショットフラグの概要：

- `--format ai`（Playwright 使用時のデフォルト）：数値参照（`aria-ref="<n>"`）を持つ AI スナップショット。
- `--format aria`：`axN` 参照を持つアクセシビリティツリー。Playwright が利用可能な場合、OpenClaw はバックエンド DOM ID を使用して参照をライブページに関連付けるため、後続のアクションでそれらを使用できます。利用できない場合、出力は調査専用として扱ってください。
- `--efficient`（または `--mode efficient`）：コンパクトなロールスナップショットのプリセット。これをデフォルトにするには `browser.snapshotDefaults.mode: "efficient"` を設定します（[Gateway の設定](/ja-JP/gateway/configuration-reference#browser)を参照）。
- `--interactive`、`--compact`、`--depth`、`--selector` は、`ref=e12` 参照を持つロールスナップショットを強制します。`--frame "<iframe>"` はロールスナップショットの範囲を iframe に限定します。
- Playwright では、`--labels` によって参照ラベルを重ねたスクリーンショット
  （`MEDIA:<path>` を出力）と、各参照の境界
  ボックスを含む `annotations` 配列が追加されます。`screenshot` では、Playwright ベースのラベルは `--full-page`、
  `--ref`、`--element` で機能します。`snapshot` では、付属するスクリーンショットは引き続き
  ビューポートのみに限定されます。既存セッション／chrome-mcp プロファイルは、ページの
  スクリーンショット上にオーバーレイラベルを描画しますが、`annotations` は返さず、Playwright の
  全ページ／参照／要素プロジェクションヘルパーも使用しません。Playwright または chrome-mcp がない場合、
  ラベル付きスクリーンショットは利用できません。
- `--urls` は、検出されたリンク先を AI スナップショットに追加します。

## スナップショットと参照

OpenClaw は 2 種類の「スナップショット」形式をサポートしています。

- **AI スナップショット（数値参照）**：`openclaw browser snapshot`（デフォルト、`--format ai`）
  - 出力：数値参照を含むテキストスナップショット。
  - アクション：`openclaw browser click 12`、`openclaw browser type 23 "hello"`。
  - 内部では、Playwright の `aria-ref` を使用して参照が解決されます。

- **ロールスナップショット（`e12` のようなロール参照）**：`openclaw browser snapshot --interactive`（または `--compact`、`--depth`、`--selector`、`--frame`）
  - 出力：`[ref=e12]`（および任意の `[nth=1]`）を持つロールベースのリスト／ツリー。
  - アクション：`openclaw browser click e12`、`openclaw browser highlight e12`。
  - 内部では、`getByRole(...)`（重複時は `nth()` も使用）によって参照が解決されます。
  - オーバーレイされた `e12` ラベル付きのスクリーンショットを含めるには、`--labels` を追加します。
    Playwright ベースのプロファイルでは、参照ごとの境界ボックスメタデータ
    （`annotations[]`）も返されます。
  - リンクテキストが曖昧で、エージェントが具体的な
    移動先を必要とする場合は、`--urls` を追加します。

- **ARIA スナップショット（`ax12` のような ARIA 参照）**：`openclaw browser snapshot --format aria`
  - 出力：構造化ノードとしてのアクセシビリティツリー。
  - アクション：スナップショットパスが Playwright と Chrome バックエンド DOM ID を介して
    参照を関連付けられる場合、`openclaw browser click ax12` が機能します。
- Playwright が利用できない場合でも、ARIA スナップショットは
  調査に役立ちますが、参照を操作できない可能性があります。アクション参照が必要な場合は、
  `--format ai` または `--interactive` でスナップショットを再取得してください。
- raw CDP フォールバックパスの Docker 検証：`pnpm test:docker:browser-cdp-snapshot` は、
  CDP を使用して Chromium を起動し、`browser doctor --deep` を実行して、ロール
  スナップショットにリンク URL、カーソルによって操作可能と判定されたクリック対象、iframe メタデータが含まれることを検証します。

参照の動作：

- 参照は**ナビゲーションをまたいで安定しません**。何かが失敗した場合は、`snapshot` を再実行して新しい参照を使用してください。
- `/act` は、アクションによって置換がトリガーされ、置換後のタブを特定できた場合に、現在の raw `targetId` を返します。
  後続のコマンドでは、安定したタブ ID／ラベルを引き続き使用してください。
- ロールスナップショットを `--frame` 付きで取得した場合、次のロールスナップショットまで、ロール参照の範囲はその iframe に限定されます。
- 不明または古い `axN` 参照は、Playwright の `aria-ref` セレクターに
  フォールスルーせず、即座に失敗します。その場合は、同じタブで新しいスナップショットを取得してください。

## ブラウザー一括 CLI

`openclaw browser batch` は、ネストされた `/act` アクションの配列を 1 回の `/act`
呼び出し（エージェントツールから到達するものと同じ `kind="batch"` ランタイム）で実行します。これにより CLI
ユーザーとスクリプトは、`wait`、`click`、`type`、
`evaluate` などのアクションを、アクションごとのラウンドトリップなしで、再実行可能な単一のプランにまとめられます。
`actions[]` の各エントリは `BrowserActRequest`、つまり `/act`
ルートが受け付ける閉じた共用体（`click`、`clickCoords`、`type`、`press`、`hover`、
`scrollIntoView`、`drag`、`select`、`fill`、`resize`、`wait`、`evaluate`、
`close`、`batch`）であり、任意の `openclaw browser` サブコマンドではありません。`batch` は、
`profile="user"` およびその他の既存セッション（chrome-mcp）
プロファイルではサポートされません。それらではアクションを個別に送信してください。

- CLI：`openclaw browser batch --actions '<json>'`、`openclaw browser batch
--actions-file plan.json`、または標準入力から
  JSON 配列を読み取る `openclaw browser batch --actions-file -`。`--continue` は `stopOnError=false` を設定します。
  デフォルトでは、最初のエラーで停止します。`--target-id` は、バッチ全体の範囲を
  1 つのタブに限定します。
- 参照のライフサイクル：参照はバッチ前の `snapshot` 実行から取得します（スナップショットは
  ネストされたアクションではありません）。ナビゲーションをトリガーする
  `click` や DOM を変更する `evaluate` など、ページ状態を変更するネストされたアクションは、
  バッチの残りの処理で以前の参照を無効にする可能性があります。状態を変更するアクションを
  最初に配置するか、スナップショットを再取得した後の後続バッチに分割してください。ナビゲーションと
  スナップショットの再取得はバッチの外部（`openclaw browser navigate` /
  `snapshot`）で行います。`open`、`navigate`、`snapshot` は `/act` の種類ではないためです。
- ターゲット ID の競合：ネストされたアクションでは `targetId` を省略するか、リクエストレベルの
  `targetId` を繰り返し指定できます。別のタブに解決される明示的なネスト済み `targetId` は、
  いずれかのアクションが実行される前に `ACT_TARGET_ID_MISMATCH` で拒否されます。
  一括アクションは設計上、リクエストのタブを共有します。
- エラー概要：レスポンスは `{ "results": [{ "ok": true }, { "ok": false,
"error": "<message>" }, ...] }` で、アクションごとに順番どおり 1 エントリが含まれます。
  `stopOnError` がデフォルトの場合、配列は最初の失敗で終了します。`--continue` では、
  すべてのアクションを対象とします。失敗したエントリが 1 つでもあると、CLI はゼロ以外で終了します。
  スクリプト用に順序どおりの完全なレスポンスを保持するには、`--json` を渡します。

## 待機機能の強化

時間やテキスト以外も待機対象にできます。

- URL を待機（Playwright による glob 対応）：
  - `openclaw browser wait --url "**/dash"`
- ロード状態を待機：
  - `openclaw browser wait --load networkidle`
  - 管理対象の `openclaw` および raw／リモート CDP プロファイルでサポートされます。`existing-session` ドライバーを使用するプロファイル（デフォルトの `user` プロファイルを含む）は `networkidle` を拒否します。その場合は、`--url`、`--text`、セレクター、または `--fn` 待機を使用してください。
- JS 述語を待機：
  - `openclaw browser wait --fn "window.ready===true"`
- セレクターが表示されるまで待機：
  - `openclaw browser wait "#main"`

これらは組み合わせて使用できます。

```bash
openclaw browser wait "#main" \
  --url "**/dash" \
  --load networkidle \
  --fn "window.ready===true" \
  --timeout-ms 15000
```

## デバッグワークフロー

アクションが失敗した場合（例：「表示されていない」、「strict mode 違反」、「覆われている」）：

1. `openclaw browser snapshot --interactive`
2. `click <ref>` / `type <ref>` を使用します（対話モードではロール参照を推奨）
3. それでも失敗する場合：Playwright が何をターゲットにしているか確認するには `openclaw browser highlight <ref>`
4. ページの動作がおかしい場合：
   - `openclaw browser errors --clear`
   - `openclaw browser requests --filter api --clear`
5. 詳細なデバッグでは、トレースを記録します：
   - `openclaw browser trace start`
   - 問題を再現します
   - `openclaw browser trace stop`（`TRACE:<path>` を出力）

## JSON 出力

`--json` は、スクリプトおよび構造化ツール向けです。

例：

```bash
openclaw browser --json status
openclaw browser --json snapshot --interactive
openclaw browser --json requests --filter api
openclaw browser --json cookies
```

JSON のロールスナップショットには、`refs` に加えて小さな `stats` ブロック（行数／文字数／参照数／対話可能数）が含まれるため、ツールはペイロードのサイズと密度を判断できます。

## 状態と環境の調整項目

これらは「サイトを X のように動作させる」ワークフローに役立ちます。

- Cookie：`cookies`、`cookies set`、`cookies clear`
- ストレージ：`storage local|session get|set|clear`
- オフライン：`set offline on|off`
- ヘッダー：`set headers --headers-json '{"X-Debug":"1"}'`（または位置指定形式の `set headers '{"X-Debug":"1"}'`）
- HTTP Basic 認証：`set credentials user pass`（または `--clear`）
- 位置情報：`set geo <lat> <lon> --origin "https://example.com"`（または `--clear`）
- メディア：`set media dark|light|no-preference|none`
- タイムゾーン／ロケール：`set timezone ...`、`set locale ...`
- デバイス／ビューポート：
  - `set device "iPhone 14"`（Playwright のデバイスプリセット）
  - `set viewport 1280 720`

## セキュリティとプライバシー

- openclaw ブラウザプロファイルにはログイン済みセッションが含まれる可能性があるため、機密情報として扱ってください。
- `browser act kind=evaluate` / `openclaw browser evaluate` および `wait --fn` は、
  ページコンテキスト内で任意の JavaScript を実行します。プロンプトインジェクションによって
  この動作が誘導される可能性があります。不要な場合は `browser.evaluateEnabled=false` で無効にしてください。
- `openclaw browser evaluate --fn` は、関数のソース、式、または
  ステートメント本体を受け付けます。ステートメント本体は非同期関数としてラップされるため、
  返してほしい値には `return` を使用してください。ページ側の関数が
  デフォルトの評価タイムアウトより長い時間を必要とする可能性がある場合は、`--timeout-ms <ms>` を使用してください。
- ログインおよびボット対策に関する注意事項（X/Twitter など）については、[ブラウザでのログインと X/Twitter への投稿](/ja-JP/tools/browser-login)を参照してください。
- Gateway/Node ホストは非公開（loopback または tailnet のみ）にしてください。
- リモート CDP エンドポイントは強力な機能を持つため、トンネルを使用して保護してください。

厳格モードの例（デフォルトでプライベート／内部宛先をブロック）：

```json5
{
  browser: {
    ssrfPolicy: {
      dangerouslyAllowPrivateNetwork: false,
      hostnameAllowlist: ["*.example.com", "example.com"],
      allowedHostnames: ["localhost"], // 任意の完全一致許可
    },
  },
}
```

## 関連項目

- [ブラウザ](/ja-JP/tools/browser) - 概要、設定、プロファイル、セキュリティ
- [ブラウザでのログイン](/ja-JP/tools/browser-login) - サイトへのログイン
- [Linux でのブラウザのトラブルシューティング](/ja-JP/tools/browser-linux-troubleshooting)
- [WSL2 でのブラウザのトラブルシューティング](/ja-JP/tools/browser-wsl2-windows-remote-cdp-troubleshooting)
