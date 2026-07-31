---
read_when:
    - 推論環境を構築してから、OpenClaw でセットアップを完了する場合
summary: '`openclaw onboard`（対話型オンボーディング）の CLI リファレンス'
title: オンボーディング
x-i18n:
    generated_at: "2026-07-26T09:16:11Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 8ec5cfc564aa14041d1aa67a978a4661e6105b7119a942940f71197c695e788b
    source_path: cli/onboard.md
    workflow: 16
---

# `openclaw onboard`

まず推論を確立するガイド付きセットアップです。既存の AI アクセスを検出し、
実際の補完を必須とし、動作するルートのみを永続化してから、
残りを構成するために OpenClaw を起動します。`openclaw setup` は新規
システム、またはオンボーディングオプションが指定されている場合にこのフローへ進みます。構成済みシステムでは、
システムエージェントとのチャットに引数なしの `openclaw setup` を使用します。`openclaw setup --baseline` は
ベースラインの構成とワークスペースのみを書き込みます。

<CardGroup cols={2}>
  <Card title="CLI オンボーディングハブ" href="/ja-JP/start/wizard" icon="rocket">
    対話型 CLI フローの手順ガイド。
  </Card>
  <Card title="オンボーディングの概要" href="/ja-JP/start/onboarding-overview" icon="map">
    OpenClaw のオンボーディング全体の仕組み。
  </Card>
  <Card title="CLI セットアップリファレンス" href="/ja-JP/start/wizard-cli-reference" icon="book">
    出力、内部動作、ステップごとの挙動。
  </Card>
  <Card title="CLI 自動化" href="/ja-JP/start/wizard-cli-automation" icon="terminal">
    非対話型フラグとスクリプトによるセットアップ。
  </Card>
  <Card title="macOS アプリのオンボーディング" href="/ja-JP/start/onboarding" icon="apple">
    macOS メニューバーアプリのオンボーディングフロー。
  </Card>
</CardGroup>

## 例

```bash
openclaw onboard
openclaw onboard --tui
openclaw onboard --classic
openclaw onboard --modern
openclaw onboard --flow quickstart
openclaw onboard --flow manual
openclaw onboard --flow import
openclaw onboard --import-from hermes --import-source ~/.hermes
openclaw onboard --skip-bootstrap
openclaw onboard recommendations --json
openclaw onboard recommendations acknowledge
openclaw onboard recommendations acknowledge --retry "<failed-id>"
openclaw onboard recommendations refresh
openclaw onboard --mode remote --remote-url wss://gateway-host:18789
```

`openclaw onboard recommendations` は、オンボーディング中に保存された保留中の
アプリ推薦の一致結果を読み取ります。初回実行時のブートストラップで使用する機械可読リストには
`--json` を追加します。このコマンドは、インストール済みアプリを再スキャンしたり、
モデルを呼び出したりしません。出力に含まれるのは、検証済みのインストール ID、ソース、階層のみです。
信頼されていないマーケットプレイスの説明文、モデルの理由、ローカルアプリの
ラベルは意図的に省略されます。推薦の提示に回答すると、このコマンドは
空のリストを返し、以降のオンボーディング実行ではこのステップが完全にスキップされます。
`openclaw onboard recommendations refresh` は保存された提示をクリアし、次回の
オンボーディング実行時にインストール済みアプリを再スキャンして新しい提示を作成します。

新規ワークスペースでは、推薦の選択をブートストラップ会話まで延期します。
その会話でユーザーの選択を処理した後、
`openclaw onboard recommendations acknowledge` が保存された提示を回答済みとしてマークします。
確認処理は冪等です。選択したインストールが失敗した場合は、失敗した
不透明 ID をそれぞれ `--retry <id...>` で渡します。成功した一致と辞退した一致は消費され、
失敗した一致は後続のオンボーディング実行のために保留状態のまま残ります。不明な ID の場合は、
保存された提示を変更せずに失敗します。中断された ClawHub スキルの
インストール後、既存のターゲットが成功とみなされるのは、
同じパブリッシャー修飾付き推薦 ID に対して `openclaw skills verify "@owner/slug"` が成功し、
その JSON 出力で `openclaw.resolution.source: "installed"` が報告された場合のみです。
レジストリの検証だけでは、ローカルインストールの
証明にはなりません。それ以外の場合は、その ID を `--retry` で保留状態のままにし、
既存のスキルを上書きしないでください。

- `--classic`: 完全なステップ形式のウィザードを開きます。
  `--non-interactive` と組み合わせることはできません。自動セットアップでは `--classic` を省略します。
- `--flow quickstart`: 最小限のプロンプトでクラシックウィザードを開き、
  デフォルトでトークン認証を使用します。保存済みまたは明示的な
  認証情報が適用されない場合はトークンを生成します。
  `--gateway-port`、`--gateway-bind`、`--gateway-auth`、`--tailscale` などの明示的なローカル Gateway フラグは、
  対応する保存済みまたはデフォルトのクイックスタート値を上書きします。省略した
  オプションは現在の値を維持します。
- `--flow manual`（別名 `advanced`）: ポート、バインド、認証について
  すべてのプロンプトを表示するクラシックウィザードを開きます。
- `--flow import`: 新規セットアップに対して、検出された移行プロバイダー（たとえば `--import-from hermes` 経由の Hermes）を実行します。確認後、オンボーディングは構成、認証情報、ワークスペースファイル、メモリ、スキルを非公開の一時ターゲットにステージングします。インポートされた推論は、ワークスペースとエージェント状態が昇格され、構成がコミットされる前に、実際の補完に成功する必要があります。昇格前の失敗またはキャンセルでは、稼働中のターゲットは変更されません。Codex Plugin のインストールなど、ロールバックできない外部アクティベーション手順はその後に実行され、移行レポートから再試行できます。構成、認証情報、セッション、ワークスペース状態が存在する場合は、最初にリセットしてください。ドライラン計画、上書きモード、検証済みバックアップ、レポート、正確なマッピングについては、[`openclaw migrate`](/ja-JP/cli/migrate) を使用してください。
- `--remote-url` と `--remote-token`: クラシックのリモート Gateway ステップに値を事前入力し、この実行では保存済みのリモート値を上書きします。URL を変更しても、トークンも渡さない限り保存済みの認証情報は再利用されません。トークンはプロンプト内でマスクされたままになり、ウィザードの既存の平文または SecretRef ストレージの選択に従います。
- `--tailscale-reset-on-exit` と `--no-tailscale-reset-on-exit`: Gateway の終了時に Tailscale Serve または Funnel の構成をリセットするかどうかを明示的に制御します。両方を省略すると、非対話型の再実行時に現在の設定が維持されます。
- `--modern` は、OpenClaw の対話型セットアップ
  アシスタントの互換性別名です。`openclaw setup` と同じ実推論ゲートを使用し、
  `--workspace`、`--accept-risk`、
  `--non-interactive`、`--json` のみを受け付けます。その他のセットアップフラグは
  暗黙に無視されるのではなく拒否されます。

## ガイド付きフロー

引数なしの `openclaw onboard` はガイド付きフローを開始します。セキュリティ通知を表示してから、
最初に 1 つ質問します。**フルアクセス**（推奨 — セットアップが
AI アプリ、キー、ローカルランタイムを自動的に検索）または **先に確認**（セットアップが
周辺を検索する前に一度確認するか、手動構成を選択可能）です。
選択内容は `wizard.accessMode` として永続化されます。検出が許可されている場合、オンボーディングは、
構成済みモデル、API キーの環境変数、サポート対象のローカル CLI を通じて
すでに利用可能な AI アクセスを検出し、推薦候補を実際の補完で
テストします。候補が失敗すると、オンボーディングは使用可能な次の候補を静かに
試し、応答しなかった候補を 1 行にまとめます。動作するルートが通知され、
キーを 1 回押すだけで他のすべての候補を表示できます。

自動検出ですべての候補を試し終えると、プロバイダーピッカーには OpenAI、
Anthropic、xAI (Grok)、Google、OpenRouter が最初に表示されます。その他すべての
サポート対象プロバイダーをプロバイダー別にグループ化して表示するには、**More…** を選択します。
その後、リージョン、プラン、認証方式が 2 番目のメニューに表示されます。サポート対象の
ブラウザーまたはデバイスによるサインインと、マスクされた API キーまたはトークン方式は、
同じ実補完パスを使用します。OpenClaw はテスト成功後に、検証済みの
モデルルートとその認証情報のみを永続化します。失敗した候補によって構成済みモデルが
置き換えられたり、試行した認証情報が保存されたりすることはありません。OpenClaw を起動せずに
終了するには **Skip for now** を選択し、準備ができたら
`openclaw onboard` を再実行します。OpenClaw が起動するまで、ワークスペースと Gateway のセットアップは
変更されません。

ガイド付きモードでは、`--workspace <dir>` が OpenClaw の提案するワークスペースと、
分離された推論コンテキストを指定します。OpenClaw のセットアップ提案を承認するまでは
永続化されません。クラシックおよび非対話型オンボーディングでは、通常の
セットアップフローを通じてワークスペースが永続化されます。既存のエージェント一覧がある状態で
再実行すると、オンボーディングは構成済みフリートのワークスペースを維持します。クラシック
ウィザードでは両方のパスが表示され、移動前に明示的な確認が必要です。一方、
非対話型セットアップでは警告を表示して現在の値を維持します。

推論が成功すると、オンボーディングはサポート対象のローカル AI
ツールにあるメモリを確認します。Claude Code の自動メモリ、Codex の統合メモリ、Hermes のメモリ
ファイルが対象です。いずれかが見つかると、インデックス付き検索のために
`memory/imports/` 配下のエージェントワークスペースへコピーするかを 1 ページで提示します。
確認なしにインポートされるものはなく、以前にインポート済みのファイルはスキップされます。また、
同じメモリ限定スコープを提供する Control UI の[メモリインポートページ](/ja-JP/web/control-ui)から、
いつでも後でインポートできます。（完全な [`openclaw migrate`](/ja-JP/cli/migrate) の実行は
より広範で、構成、スキル、認証情報もインポートできます。）クラシック
ウィザードでは、ワークスペースの準備後に同じページが表示されます。

推論が成功すると（およびメモリインポートの提示後）、ガイド付きオンボーディングは
標準セットアップ（ワークスペース、Gateway、セッション）を自動的に適用します。
これは対話型の `openclaw setup` チャットが「yes」で適用するものと同じ計画です。
その後、インストール済みアプリから Plugin とスキルの推薦を提示します。アプリ名は、
構成済みモデルと ClawHub 検索を通じて照合されます。このステップは
[`wizard.appRecommendations`](/ja-JP/gateway/configuration-reference#wizard) で無効にできます。
macOS、Linux、または Windows のデスクトップセッションでは、その後、認証済みの
Control UI ダッシュボードを開き、ブラウザークライアントの接続を最大 60 秒間
待機します。ヘッドレス Linux または SSH 経由では、コピー＆ペースト可能な
目立つダッシュボード URL を表示します。local loopback の Gateway 用の SSH ポート転送コマンドも含まれ、
最大 5 分間待機します。接続に成功するとブラウザーで続行します。
Gateway に到達できない場合やタイムアウトした場合は、以前と同じターミナル経路に
フォールバックします。ブラウザーへの引き継ぎをスキップして、そのターミナル経路を強制するには
`--tui` を渡します。セットアップの適用に失敗した場合、オンボーディングは
対話型で完了するために OpenClaw の対話チャットへフォールバックします。チャンネル、エージェント、
Plugin、その他のオプション機能は引き続き OpenClaw チャットで扱います。
`openclaw` を実行し、`open channel wizard for <channel>` を使用して、チャンネルの
認証情報収集をマスク付きターミナルウィザードに引き渡します。モデル
プロバイダーまたはその認証を変更するには、OpenClaw を終了して `openclaw onboard` を実行します。
OpenClaw はガイド付きまたはクラシックのプロバイダーフローを開きません。

構成済みのインストールで `openclaw onboard` を再度実行すると、最初に現在の
デフォルトモデルを検証するため、同じフローが検証および修復処理として機能します。
セットアップの再適用、再インストール、Gateway サービスの再起動は行いません。
このチェックが失敗しても、構成済みモデルが自動的に置き換えられることはありません。
オンボーディングは停止し、続行方法を確認します。このチェックは
ワークスペース外で実行されるため、ワークスペースの Plugin が提供するモデルは、エージェントでは
動作していてもここでは失敗することがあります。
プロバイダー固有の認証、チャンネル、スキル、
リモート Gateway のセットアップ、インポート、Gateway の完全な制御には `openclaw onboard --classic` を使用します。
推論を伴わない対話型のセットアップと修復には `openclaw setup` を実行します。`openclaw onboard
--modern` は同じ推論ゲートを経由する互換性別名です。クラシック
ウィザードでは、実際の補完を使用してデフォルトモデルを任意で検証できますが、
OpenClaw 自体の実推論チェックが成功するまで OpenClaw は起動しません。

対話型ターミナルでは、引数なしの `openclaw`（サブコマンドなし）は、構成
状態に応じてルーティングされます。

- アクティブな構成ファイルが存在しない場合、または作成済みの設定がない場合（空または
  メタデータのみ）、ガイド付きオンボーディングを開始します。
- 構成ファイルは存在するものの検証に失敗する場合、
  `openclaw doctor` の案内付きでクラシックオンボーディングパスを開始します。OpenClaw には動作する
  推論が必要なため、この推論前の状態の修復には使用されません。
- 構成ファイルが有効な場合、通常のエージェント TUI を開きます。到達可能な
  構成済み Gateway にエージェントとモデルがある場合は、オンボーディングや OpenClaw を経由せず、
  その UI に直接進みます。構成済みのインストールでは、TUI 内の
  `/openclaw` または `openclaw setup` で OpenClaw にアクセスします。

平文の `ws://` は、local loopback、プライベート IP リテラル、`.local`、Tailnet の `*.ts.net` Gateway URL で受け入れられます。その他の信頼済みプライベート DNS 名については、オンボーディングプロセスの環境で `OPENCLAW_ALLOW_INSECURE_PRIVATE_WS=1` を設定してください。

## リセット

```bash
openclaw onboard --reset
openclaw onboard --reset --reset-scope full
```

`--reset` はセットアップの実行前に状態を消去します。`--reset-scope` は消去範囲を制御します：`config`（設定のみ）、`config+creds+sessions`（`--reset` がスコープなしで渡された場合のデフォルト）、または `full`（ワークスペースもリセット）。ワークスペースのリセットは `--reset-scope full` の場合にのみ行われます。

## ロケール

対話型オンボーディングでは、固定のセットアップ文言に CLI ウィザードのロケールを使用します。次の順序で最初の空でない値を使用します：

1. `OPENCLAW_LOCALE`
2. `LC_ALL`
3. `LC_MESSAGES`
4. `LANG`
5. 英語へのフォールバック

サポートされているウィザードのロケールは `en`、`zh-CN`、`zh-TW` です。ロケール値には、`zh_CN.UTF-8` のようなアンダースコア形式または POSIX サフィックス形式も使用できます。製品名、コマンド名、設定キー、URL、プロバイダー ID、モデル ID、Plugin／チャンネルのラベルはそのまま維持されます。

```bash
OPENCLAW_LOCALE=zh-CN openclaw onboard
OPENCLAW_LOCALE=en openclaw onboard # 明示的に英語へ上書き
```

## 非対話型セットアップ

`--non-interactive` には `--accept-risk` が必要です（エージェントが強力であり、システム全体へのアクセスにはリスクがあることを了承します）。`--mode` のデフォルトは `local` です。

```bash
openclaw onboard --non-interactive \
  --auth-choice custom-api-key \
  --custom-base-url "https://llm.example.com/v1" \
  --custom-model-id "foo-large" \
  --custom-api-key "$CUSTOM_API_KEY" \
  --secret-input-mode plaintext \
  --custom-compatibility openai \
  --custom-image-input
```

`--custom-api-key` は省略可能です。省略した場合、オンボーディングは環境内の `CUSTOM_API_KEY` を確認します。OpenClaw は一般的なビジョンモデル ID（GPT-4o/4.1/5.x、Claude 3/4、Gemini、Qwen-VL、LLaVA、Pixtral、および類似モデル）を画像対応として自動的にマークします。不明なカスタムビジョン ID には `--custom-image-input` を渡し、テキスト専用メタデータを強制するには `--custom-text-input` を渡します。`/v1/responses` をサポートするが `/v1/chat/completions` はサポートしない OpenAI 互換エンドポイントには `--custom-compatibility openai-responses` を使用します。有効な値は `openai`（デフォルト）、`openai-responses`、`anthropic` です。

LM Studio にはプロバイダー固有のキーフラグもあります：

```bash
openclaw onboard --non-interactive \
  --auth-choice lmstudio \
  --custom-base-url "http://localhost:1234/v1" \
  --custom-model-id "qwen/qwen3.5-9b" \
  --lmstudio-api-key "$LM_API_TOKEN" \
  --accept-risk
```

非対話型 Ollama：

```bash
openclaw onboard --non-interactive \
  --auth-choice ollama \
  --custom-base-url "http://ollama-host:11434" \
  --custom-model-id "qwen3.5:27b" \
  --accept-risk
```

`--custom-base-url` のデフォルトは `http://127.0.0.1:11434` です。`--custom-model-id` は省略可能です。省略した場合、オンボーディングは Ollama の推奨デフォルトを使用します。`kimi-k2.5:cloud` のようなクラウドモデル ID もここで使用できます。

プロバイダーキーをプレーンテキストではなく参照として保存します：

```bash
openclaw onboard --non-interactive \
  --auth-choice openai-api-key \
  --secret-input-mode ref \
  --accept-risk
```

`--secret-input-mode ref` を使用すると、オンボーディングはプレーンテキストのキー値ではなく、環境変数に基づく参照を書き込みます。認証プロファイルに基づくプロバイダーでは `keyRef: { source: "env", provider: "default", id: <envVar> }` を書き込み、カスタムプロバイダーでは同じ方法で `models.providers.<id>.apiKey`（例：`{ source: "env", provider: "default", id: "CUSTOM_API_KEY" }`）を書き込みます。契約：オンボーディングプロセスの環境でプロバイダーの環境変数（例：`OPENAI_API_KEY`）を設定してください。その環境変数が設定されていない限り、インラインキーフラグも同時に渡さないでください。対応する環境変数なしでフラグ値を指定すると、ガイダンスを表示して即座に失敗します。

### Gateway 認証（非対話型）

- `--gateway-auth token --gateway-token <token>` はプレーンテキストのトークンを保存します。`token` がデフォルトの認証モードです。
- `--gateway-auth token --gateway-token-ref-env <name>` は `gateway.auth.token` を環境変数の SecretRef として保存します。オンボーディングプロセスの環境に、その名前の空でない環境変数が必要です。
- `--gateway-token` と `--gateway-token-ref-env` は相互排他的です。
- `--install-daemon` を使用する場合：SecretRef で管理される `gateway.auth.token` は検証されますが、解決済みのプレーンテキストとしてスーパーバイザーサービスの環境メタデータには永続化されません。参照を解決できない場合、インストールは修復ガイダンスを表示して安全側に失敗します。`gateway.auth.token` と `gateway.auth.password` の両方が設定され、`gateway.auth.mode` が未設定の場合、モードが明示的に設定されるまでインストールはブロックされます。
- ローカルオンボーディングは設定に `gateway.mode="local"` を書き込みます。後から設定ファイルに `gateway.mode` がない場合、それは有効なローカルモードのショートカットではなく、設定の破損または不完全な手動編集を示します。
- ローカルオンボーディングは、選択したセットアップパスに必要なダウンロード可能な Plugin をインストールします（たとえば、それらの認証方式用の Codex または Copilot ランタイム Plugin）。リモートオンボーディングはリモート Gateway の接続情報のみを書き込み、ローカルの Plugin パッケージをインストールすることはありません。
- `--allow-unconfigured` は独立した `openclaw gateway run` の緊急回避手段であり、オンボーディングが `gateway.mode` を省略できるようにするものではありません。

```bash
export OPENAI_API_KEY="your-provider-key"
export OPENCLAW_GATEWAY_TOKEN="your-token"
openclaw onboard --non-interactive \
  --mode local \
  --auth-choice openai-api-key \
  --secret-input-mode ref \
  --gateway-auth token \
  --gateway-token-ref-env OPENCLAW_GATEWAY_TOKEN \
  --accept-risk
```

### ローカル Gateway の健全性

- `--skip-health` を渡さない限り、オンボーディングは到達可能なローカル Gateway を待ってから正常終了します。
- `--install-daemon` は、最初に管理対象 Gateway のインストールパスを開始します。これを指定しない場合、ローカル Gateway がすでに実行中である必要があります（例：`openclaw gateway run`）。
- 自動化で設定／ワークスペース／ブートストラップの書き込みだけを行う場合、`--skip-health` は待機を省略します。
- `--skip-bootstrap` は `agents.defaults.skipBootstrap: true` を設定し、`AGENTS.md`、`SOUL.md`、`TOOLS.md`、`IDENTITY.md`、`USER.md`、`HEARTBEAT.md`、`BOOTSTRAP.md` の作成を省略します。
- ネイティブ Windows では、`--install-daemon` は最初にタスク スケジューラを試し、タスクの作成が拒否された場合はユーザー単位のスタートアップフォルダーのログイン項目にフォールバックします。

### 対話型参照モード

- プロンプトが表示されたら **シークレット参照を使用** を選択し、続いて **環境変数** または設定済みのシークレットプロバイダー（`file` または `exec`）を選択します。
- オンボーディングは参照を保存する前に高速な事前検証を実行し、失敗した場合は再試行できます。

### Z.AI エンドポイントの選択肢

<Note>
`--auth-choice zai-api-key` は、キーに最適な Z.AI エンドポイントとモデルを自動検出します。Coding Plan エンドポイントでは `zai/glm-5.2` が優先され（利用できない場合は `glm-5.1` にフォールバック）、一般 API エンドポイントではデフォルトで `zai/glm-5.1` が使用されます。Coding Plan エンドポイントを強制するには、`zai-coding-global` または `zai-coding-cn` を直接選択します。
</Note>

```bash
# プロンプトなしでエンドポイントを選択
openclaw onboard --non-interactive \
  --auth-choice zai-coding-global \
  --zai-api-key "$ZAI_API_KEY"

# その他の Z.AI エンドポイントの選択肢：zai-coding-cn、zai-global、zai-cn
```

Mistral：

```bash
openclaw onboard --non-interactive \
  --auth-choice mistral-api-key \
  --mistral-api-key "$MISTRAL_API_KEY"
```

## その他の非対話型フラグ

トークンベースのモデル認証（`--auth-choice token` と併用）：

| フラグ                            | 説明                                                                                                                 |
| ------------------------------- | --------------------------------------------------------------------------------------------------------------------------- |
| `--token-provider <id>`         | トークンを発行するトークンプロバイダー ID                                                                                         |
| `--token <token>`               | モデル認証用のトークン値                                                                                        |
| `--token-profile-id <id>`       | 認証プロファイル ID（デフォルトは `<provider>:manual`。プロバイダー所有の一部フローでは、`anthropic:default` など独自のデフォルトを使用） |
| `--token-expires-in <duration>` | 省略可能なトークン有効期限（例：`365d`、`12h`）                                                                         |

Cloudflare AI Gateway：`--cloudflare-ai-gateway-account-id <id>`、`--cloudflare-ai-gateway-gateway-id <id>`。

デーモンインストールの制御：`--no-install-daemon`／`--skip-daemon`（エイリアス。Gateway サービスのインストールを省略）、`--daemon-runtime <node>`。

Skills：`--node-manager <npm|pnpm|bun>`（デフォルトは `npm`）、`--skip-skills`。

UI とフックのセットアップ：`--skip-ui`（Control UI／TUI のプロンプトを省略）、`--skip-hooks`（Webhook／フックのセットアップを省略）、`--skip-channels`、`--skip-search`。

出力：`--suppress-gateway-token-output` は、トークンを含む Gateway／UI 出力（トークンのヒント、トークンが埋め込まれた自動ログイン URL、Control UI の自動起動）を抑制します。共有端末や CI で役立ちます。

<Note>
`--json` は、ガイド付きまたはクラシックオンボーディングで非対話型モードを意味するものではありません。
`--modern` を使用すると、JSON は OpenClaw の概要を一度だけ出力し、その単一の結果の後に終了します。ほかのスクリプトには `--non-interactive` を使用してください。
</Note>

## プロバイダーの事前フィルタリング

認証方式が優先プロバイダーを示す場合、オンボーディングはデフォルトモデルと許可リストの選択画面を、そのプロバイダーのモデルに事前フィルタリングします。このフィルターは、同じ Plugin が所有するほかのプロバイダーにも一致します。これにより、`volcengine`／`volcengine-plan` や `byteplus`／`byteplus-plan` などの Coding Plan バリアントも対象になります。優先プロバイダーのフィルターで読み込まれたモデルが 1 つも得られない場合、選択画面を空のままにせず、フィルターされていないカタログにフォールバックします。

## Web 検索の追加プロンプト

一部の Web 検索プロバイダーは、オンボーディング中にプロバイダー固有の追加プロンプトを表示します：

- **Grok** は、同じ xAI 認証と `x_search` モデルの選択を使用する、省略可能な `x_search` セットアップを提示できます。
- **Kimi** は、Moonshot API のリージョン（`api.moonshot.ai` または `api.moonshot.cn`）と、デフォルトの Kimi Web 検索モデルの入力を求めることがあります。

## その他の動作

- ローカルオンボーディングの DM スコープの動作：[CLI セットアップリファレンス](/ja-JP/start/wizard-cli-reference#outputs-and-internals)。
- 最速で最初のチャットを開始する方法：`openclaw dashboard`（Control UI、チャンネル設定なし）。
- カスタムプロバイダー：一覧にないホステッドプロバイダーを含め、OpenAI または Anthropic 互換の任意のエンドポイントに接続できます。ライブプローブで自動検出するには、互換性として **不明** を使用します。
- Hermes の状態が検出された場合、オンボーディングは移行フローを提示します（上記の `--flow import` を参照）。

## よく使用する後続コマンド

対象を限定した非推論設定の変更には、後から `openclaw configure` を使用し、チャンネルのみのセットアップには `openclaw
channels add` を使用します。モデルプロバイダーまたは認証ルートを変更する場合は、代わりに `openclaw onboard` を実行します。

```bash
openclaw channels add
openclaw configure
openclaw agents add <name>
```
