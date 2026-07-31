---
read_when:
    - TUI の初心者向けチュートリアルを求めている場合
    - TUI の機能、コマンド、ショートカットの完全な一覧が必要です
summary: ターミナル UI（TUI）：Gateway に接続するか、組み込みモードでローカル実行する
title: TUI
x-i18n:
    generated_at: "2026-07-26T09:57:23Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: dc4dc5e2a408b5097b3615283b5a4590e8b55bccb15c26d8e38ab2c84b902f4a
    source_path: web/tui.md
    workflow: 16
---

## クイックスタート

### Gateway モード

1. Gateway を起動します。

```bash
openclaw gateway
```

2. TUI を開きます。

```bash
openclaw tui
```

3. メッセージを入力し、Enter キーを押します。

リモート Gateway：

```bash
openclaw tui --url ws://<host>:<port> --token <gateway-token>
```

Gateway がパスワード認証を使用している場合は、`--password` を使用します。

### ローカルモード

Gateway なしで TUI を実行します。

```bash
openclaw chat
# または
openclaw tui --local
```

- `openclaw chat` と `openclaw terminal` は `openclaw tui --local` のエイリアスです。
- `--local` は `--url`、`--token`、`--password` と組み合わせて使用できません。
- ローカルモードでは、組み込みのエージェントランタイムを直接使用します。ほとんどのローカルツールは動作しますが、Gateway 専用機能は利用できません。
- サブコマンドなしの `openclaw` は、ターゲットを自動的に選択します。未設定のインストールでは推論のオンボーディングを実行し、無効な設定では従来の Doctor ガイダンスを開き、到達可能な設定済み Gateway がある場合はこの TUI シェルを Gateway モードで開き、それ以外でローカルモデルが設定済みの場合はローカルモードで開きます。

## 表示内容

- ヘッダー：接続 URL、現在のエージェント、現在のセッション。
- チャットログ：ユーザーメッセージ、アシスタントの応答、システム通知、ツールカード。
- ステータス行：接続／実行状態（接続中、実行中、ストリーミング中、アイドル、エラー）。
- フッター：エージェント + セッション + モデル + ゴール状態 + think/fast/verbose/trace/reasoning + トークン数 + 配信。
- 入力：オートコンプリート付きテキストエディター。

## メンタルモデル：エージェント + セッション

- エージェントは一意のスラッグです（例：`main`、`research`）。Gateway がその一覧を公開します。
- セッションは現在のエージェントに属します。
- セッションキーは `agent:<agentId>:<sessionKey>` として保存されます。
  - `/session main` と入力すると、TUI はそれを `agent:<currentAgent>:main` に展開します。
  - `/session agent:other:main` と入力すると、そのエージェントセッションへ明示的に切り替わります。
- セッションスコープ：
  - `per-sender`（デフォルト）：各エージェントに複数のセッションがあります。
  - `global`：TUI は常に `global` セッションを使用します（ピッカーが空になる場合があります）。
- 現在のエージェントとセッションは、常にフッターに表示されます。
- セッションに[ゴール](/ja-JP/tools/goal)がある場合、フッターにはその簡潔な状態として、
  `Pursuing goal`、`Goal paused (/goal resume)`、`Goal blocked (/goal resume)`、または `Goal achieved` が表示されます。
- `--session` なしで起動した場合、Gateway モードの TUI は、同じ Gateway、エージェント、セッションスコープについて最後に選択されたセッションがまだ存在すれば、そのセッションを再開します。`--session`、`/session`、`/new`、または `/reset` を渡した場合は、引き続き明示的な指定として扱われます。

## 送信 + 配信

- メッセージは常に Gateway（ローカルモードでは組み込みランタイム）へ送られます。アシスタントの応答をチャットプロバイダーへ配信する処理は、それとは別であり、デフォルトでは無効です。
- TUI は WebChat と同様の内部ソース画面であり、汎用の送信チャネルではありません。表示可能な応答に `tools.message` を必要とするハーネスでは、ターゲットなしの `message.send` によってアクティブな TUI ターンを満たせます。明示的なプロバイダー配信では、引き続き通常の設定済みチャネルを使用し、`lastChannel` にフォールバックすることはありません。
- 配信設定は起動時に TUI セッション全体に対して固定されます。有効にするには `openclaw tui --deliver` を指定して起動します。セッションの途中で切り替えるための `/deliver` スラッシュコマンドや Settings トグルはありません。変更するには TUI を再起動します。

## ピッカー + オーバーレイ

- モデルピッカー：利用可能なモデルを一覧表示し、セッションのオーバーライドを設定します。
- エージェントピッカー：別のエージェントを選択します。
- セッションピッカー：過去 7 日以内に更新された現在のエージェントのセッションを最大 50 件表示します。既知の古いセッションへ移動するには、`/session <key>` を使用します。
- Settings（`/settings`）：ツール出力の展開と思考内容の表示を切り替えます。このパネルでは配信を制御しません。

## キーボードショートカット

- Enter：メッセージを送信
- Esc：アクティブな実行を中止
- Ctrl+C：入力をクリア（2 回押すと終了）
- Ctrl+D：終了
- Ctrl+L：モデルピッカー
- Ctrl+G：エージェントピッカー
- Ctrl+P：セッションピッカー
- Ctrl+O：ツール出力の展開を切り替え
- Ctrl+T：思考内容の表示を切り替え（履歴を再読み込み）

## スラッシュコマンド

コア：

- `/help`
- `/status`（Gateway に転送され、セッション／モデルの概要を表示）
- `/gateway-status`（エイリアス：`/gwstatus`。Gateway の接続状態を直接表示）
- `/agent <id>`（または `/agents`）
- `/session <key>`（または `/sessions`）
- `/model <provider/model>`（または `/models`）

セッション制御：

- `/think <off|minimal|low|medium|high>`（モデルによっては、上位ティアで `xhigh`／`max` のようなレベルが追加される場合があります）
- `/fast <status|auto|on|off>`
- `/verbose <on|full|off>`
- `/trace <on|off>`
- `/reasoning <on|off|stream>`
- `/usage <off|tokens|full|reset>`（`reset`／`inherit`／`clear`／`default` を指定すると、セッションのオーバーライドを解除）
- `/goal [status] | /goal start <objective> | /goal edit <objective> | /goal pause|resume|complete|block|clear`
- `/elevated <on|off|ask|full>`（エイリアス：`/elev`）
- `/activation <mention|always>`
- `/queue <steer|followup|collect|interrupt> [debounce:<duration>] [cap:<n>] [drop:<summarize|old|new>]`
- `/queue default`（または `/queue reset`）はセッションのオーバーライドを解除します

セッションのライフサイクル：

- `/new`（新しいキーで、独立した新規セッションを作成。古いセッションを使用中の他の TUI クライアントには影響しません）
- `/reset`（現在のセッションキーをその場でリセット）
- `/abort`（アクティブな実行を中止）
- `/settings`
- `/exit`（または `/quit`）

ローカルモードのみ：

- `/auth [provider]` は、TUI 内でプロバイダーの認証／ログインフローを開きます。

ローカルモードでは、組み込みランタイム内に同じキューモードが実装されています。実行途中の
プロンプトは、セッションの `/queue` ポリシーに従います。`steer` は、
ランタイムが受け入れ可能な場合に挿入し、`followup` は別のターンまで待機し、`collect` は
保留中のプロンプトを結合し、`interrupt` は新しい実行を開始する前に現在の
実行を停止します。明示的な `/steer <message>` は Gateway 専用です。ローカルモードでは、
`/queue steer` と通常のメッセージを使用します。

OpenClaw：

- `/openclaw [request]` は、通常のエージェント TUI から [OpenClaw](#openclaw-setup-and-repair-helper) セットアップ／修復チャットに戻り、必要に応じて 1 件のリクエストを転送します。

その他の Gateway スラッシュコマンド（例：`/context`）は Gateway に転送され、システム出力として表示されます。[スラッシュコマンド](/ja-JP/tools/slash-commands)を参照してください。

## ローカルシェルコマンド

- 行の先頭に `!` を付けると、TUI ホスト上でローカルシェルコマンドを実行します。
- TUI は、ローカル実行を許可するかどうかをセッションごとに一度確認します。拒否した場合、そのセッションでは `!` が無効のままになります。
- コマンドは TUI の作業ディレクトリにある新しい非対話型シェルで実行されます（永続的な `cd`／環境変数はありません）。
- ローカルシェルコマンドの環境には `OPENCLAW_SHELL=tui-local` が渡されます。
- `!` だけの行は通常のメッセージとして送信されます。先頭に空白があってもローカル実行はトリガーされません。

## OpenClaw セットアップ／修復ヘルパー

OpenClaw は最優先レベルのセットアップ／修復アシスタントです。設定されたデフォルトモデルがライブ推論チェックに合格すると、`openclaw setup` として利用できます。推論を利用できない場合、対話型の呼び出しは推論のオンボーディングに戻り、自動化は修復ガイダンスとともに失敗します。これは `openclaw tui --local` と同じローカル TUI シェル内で実行され、OpenClaw の型付きかつ承認制の操作に制限された AI エージェントによって支えられています。

```bash
openclaw setup                       # 対話形式で開始
openclaw setup -m "status"           # 1 件のリクエストを実行して終了
openclaw setup -m "set default model openai/gpt-5.2" --yes   # 設定への書き込みを適用
```

- 永続的な設定への書き込みには承認が必要です。対話形式で確認するか、`--yes` を渡します。
- `--json` は、チャットを開始する代わりに起動時の概要を JSON として出力します。
- OpenClaw 内から `open-tui` リクエスト（たとえば通常のエージェントとの対話を求めるもの）を行うと、OpenClaw を終了して通常のエージェント TUI を開きます。戻るには、そこで `/openclaw` を使用します。

現在の設定がすでに検証に合格しており、実行中の Gateway に依存せず、同じマシン上で組み込みエージェントに設定を調査させ、ドキュメントと比較し、ずれの修復を支援させたい場合は、ローカルモードを使用します。

`openclaw config validate` がすでに失敗している場合は、まず `openclaw configure` または `openclaw doctor --fix` から始めます。`openclaw chat` の起動にも、読み込み可能な設定が必要です。

一般的な手順：

1. ローカルモードを起動します。

```bash
openclaw chat
```

2. 確認してほしい内容をエージェントに依頼します。例：

```text
Gateway の認証設定をドキュメントと比較し、最小限の修正を提案してください。
```

3. 正確な根拠の取得と検証には、ローカルシェルコマンドを使用します。

```text
!openclaw config file
!openclaw docs gateway auth token secretref
!openclaw config validate
!openclaw doctor
```

4. `openclaw config set` または `openclaw configure` で限定的な変更を適用し、`!openclaw config validate` を再実行します。
5. Doctor が自動移行または修復を推奨した場合は、内容を確認して `!openclaw doctor --fix` を実行します。

ヒント：

- `openclaw.json` を手作業で編集するより、`openclaw config set` または `openclaw configure` を優先してください。
- `openclaw docs "<query>"` は、同じマシンから最新のドキュメントインデックスを検索します。
- 構造化されたスキーマおよび SecretRef／解決可能性のエラーを確認したい場合は、`openclaw config validate --json` が便利です。

## ツール出力

- ツール呼び出しは、引数と結果を含むカードとして表示されます。
- Ctrl+O で折りたたみ表示と展開表示を切り替えます。
- ツールの実行中は、部分的な更新が同じカードにストリーミングされます。

## ターミナルの色

- TUI はアシスタントの本文をターミナルのデフォルトの前景色で表示するため、暗い背景でも明るい背景でも読みやすさが保たれます。
- ターミナルの背景が明るく、自動検出が正しくない場合は、`openclaw tui` を起動する前に `OPENCLAW_THEME=light` を設定します。
- 代わりに元のダークパレットを強制するには、`OPENCLAW_THEME=dark` を設定します。

## 履歴 + ストリーミング

- 接続時に、TUI は最新の履歴を読み込みます（デフォルトは 200 メッセージ）。
- ストリーミング応答は、確定するまでその場で更新されます。
- TUI は、より詳細なツールカードを表示するために、エージェントのツールイベントも監視します。

## 接続の詳細

- TUI は、大まかな `ui` クライアントモードでクライアント ID `openclaw-tui` を使用して接続します（Gateway ポリシーについて Control UI と WebChat が使用するものと同じモードです）。
- 再接続はシステムメッセージで示され、イベントの欠落はログに表示されます。

## オプション

- `--local`: ローカルの組み込みエージェントランタイムに対して実行
- `--url <url>`: Gateway WebSocket URL（デフォルトは設定の `gateway.remote.url`、または loopback の `ws://127.0.0.1:<port>`）
- `--token <token>`: Gateway トークン（必要な場合）
- `--password <password>`: Gateway パスワード（必要な場合）
- `--tls-fingerprint <sha256>`: 証明書をピン留めした `wss://` Gateway に期待される TLS 証明書フィンガープリント
- `--session <key>`: セッションキー（デフォルト: `main`、スコープがグローバルの場合は `global`）
- `--deliver`: アシスタントの応答をプロバイダーに配信（デフォルトはオフ）
- `--thinking <level>`: 送信時の思考レベルを上書き
- `--message <text>`: 接続後に最初のメッセージを送信
- `--timeout-ms <ms>`: エージェントのタイムアウト（ミリ秒、デフォルトは `agents.defaults.timeoutSeconds`）
- `--history-limit <n>`: 読み込む履歴エントリ数（デフォルトは `200`）

<Warning>
`--url` を設定すると、TUI は設定または環境の認証情報にフォールバックしません。`--token` または `--password` を明示的に渡し、ターゲットがピン留めされた証明書を使用する場合は `--tls-fingerprint` も渡してください。明示的な認証情報がない場合はエラーになります。ローカルモードでは、`--url`、`--token`、`--password`、`--tls-fingerprint` を渡さないでください。
</Warning>

## トラブルシューティング

メッセージの送信後に出力がない場合:

- TUI で `/status` を実行し、Gateway が接続され、アイドル状態またはビジー状態であることを確認します。
- Gateway のログを確認します: `openclaw logs --follow`。
- エージェントを実行できることを確認します: `openclaw status` および `openclaw models status`。
- チャットチャンネルにメッセージが表示されることを期待する場合は、TUI が `--deliver` を指定して起動されたことを確認します（再起動せずに後から有効にすることはできません）。

## 接続のトラブルシューティング

- `disconnected`: Gateway が実行中で、`--url/--token/--password` が正しいことを確認します。
- 選択画面にエージェントがない場合: `openclaw agents list` とルーティング設定を確認します。
- セッション選択画面が空の場合: グローバルスコープになっているか、まだセッションがない可能性があります。

## 関連項目

- [コントロール UI](/ja-JP/web/control-ui) — Web ベースの制御インターフェース
- [設定](/ja-JP/cli/config) — `openclaw.json` の確認、検証、編集
- [Doctor](/ja-JP/cli/doctor) — ガイド付きの修復および移行チェック
- [CLI リファレンス](/ja-JP/cli) — CLI コマンドの完全なリファレンス
