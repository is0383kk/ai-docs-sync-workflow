---
read_when:
    - ライブモデルマトリクス / CLI バックエンド / ACP / メディアプロバイダーのスモークテストを実行する
    - ライブテストの認証情報解決をデバッグする
    - プロバイダー固有の新しいライブテストの追加
sidebarTitle: Live tests
summary: ライブ（ネットワークにアクセスする）テスト：モデルマトリックス、CLI バックエンド、ACP、メディアプロバイダー、認証情報
title: テスト：ライブスイート
x-i18n:
    generated_at: "2026-07-26T09:25:48Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: ea8279e734e3aa09dd1fa184806c925e0404edfa9acf0f682f73a4955ed90b8b
    source_path: help/testing-live.md
    workflow: 16
---

クイックスタート、QA ランナー、ユニット/統合スイート、Docker フローについては、
[テスト](/ja-JP/help/testing)を参照してください。このページでは、**ライブ**（ネットワークにアクセスする）テスト、
すなわちモデルマトリクス、CLI バックエンド、ACP、メディアプロバイダー、認証情報の取り扱いについて説明します。

## ライブテストと実際の Gateway の違い

ライブスイートおよびアドホックなスモークテストが、すでに実際のトラフィックを
処理している Gateway（自分または別のオペレーターのもの）を妨害することは絶対に避けてください。

- 専用の Gateway を使用する: プロセス内 Gateway（以下のレイヤー 2）を使用するか、
  分離された状態ディレクトリ（`OPENCLAW_STATE_DIR=<scratch>`）と空いているポートで
  開発インスタンスを起動します。実際の Gateway がデフォルトの Gateway ポート（18789）で
  稼働している間は、そのポートにバインドしないでください。
- このセッションで自分が起動していないサービスを `openclaw gateway stop`/`restart`（または `launchctl`/`systemctl`/tmux
  の同等操作）しないでください。それはオペレーターのライブインスタンスです。先に明示的な承認を得てください。
- 現実的なデータが必要な場合: ライブの状態/DB を開発用の状態ディレクトリにコピーし、
  そのコピーに対してテストしてください。ライブ Gateway の状態をその場で移行する場合も、
  明示的な承認が必要です。

## ライブ: ローカルスモークコマンド

アドホックなライブチェックを行う前に、必要なプロバイダーキーをプロセス環境に
エクスポートしてください。

安全なメディアスモークテスト:

```bash
pnpm openclaw infer tts convert --local --json \
  --text "OpenClaw ライブスモークテスト。" \
  --output /tmp/openclaw-live-smoke.mp3
```

安全な音声通話準備状況スモークテスト:

```bash
pnpm openclaw voicecall setup --json
pnpm openclaw voicecall smoke --to "+15555550123"
```

`--yes` も指定されていない限り、`voicecall smoke` はドライランです。実際に
通話する場合にのみ `--yes` を使用してください。Twilio、Telnyx、Plivo では、
準備状況チェックを成功させるには公開 Webhook URL が必要です。これらのプロバイダーから
到達できないため、ローカル/プライベートのループバック URL は拒否されます。

## ライブ: Android Node 機能スイープ

- テスト: `src/gateway/android-node.capabilities.live.test.ts`
- スクリプト: `pnpm android:test:integration`
- 目的: 接続済みの Android Node が現在公開している**すべてのコマンド**を呼び出し、コマンド契約の動作を検証します。
- 範囲:
  - 事前条件を満たすための手動セットアップ（このスイートはアプリのインストール、実行、ペアリングを行いません）。
  - 選択した Android Node に対する、コマンドごとの Gateway `node.invoke` 検証。
- 必要な事前セットアップ:
  - Android アプリがすでに Gateway に接続され、ペアリングされていること。
  - アプリをフォアグラウンドに維持すること。
  - 成功を期待する機能について、権限/キャプチャの同意が付与されていること。
- 任意のターゲットオーバーライド:
  - `OPENCLAW_ANDROID_NODE_ID` または `OPENCLAW_ANDROID_NODE_NAME`。
  - `OPENCLAW_ANDROID_GATEWAY_URL` / `OPENCLAW_ANDROID_GATEWAY_TOKEN` / `OPENCLAW_ANDROID_GATEWAY_PASSWORD`。
- Android の完全なセットアップ詳細: [Android アプリ](/ja-JP/platforms/android)

## ライブ: モデルスモークテスト（プロファイルキー）

ライブモデルテストは、障害を分離できるように 2 つのレイヤーに分かれています。

- 「モデルへの直接接続」では、指定したキーでプロバイダー/モデルが応答できるかどうかを確認します。
- 「Gateway スモークテスト」では、そのモデルで Gateway とエージェントのパイプライン全体（セッション、履歴、ツール、サンドボックスポリシーなど）が動作するかどうかを確認します。

以下の厳選されたモデルリストは `src/agents/live-model-filter.ts` にあり、
時間の経過とともに変更されます。このページではなく、そこにある配列を信頼できる唯一の情報源として扱ってください。

MiniMax M3 は、デフォルトのプロバイダー/モデル参照として `minimax/MiniMax-M3` を使用します。

### レイヤー 1: モデル補完への直接接続（Gateway なし）

- テスト: `src/agents/models.profiles.live.test.ts`
- 目的:
  - 検出されたモデルを列挙する
  - `getApiKeyForModel` を使用して、認証情報を持つモデルを選択する
  - モデルごとに小規模な補完を実行する（必要に応じて対象を絞ったリグレッションテストも実行）
- 有効化方法:
  - `pnpm test:live`（Vitest を直接呼び出す場合は `OPENCLAW_LIVE_TEST=1`）
  - このスイートを実際に実行するには、`OPENCLAW_LIVE_MODELS=modern`、`small`、または `all`（`modern` のエイリアス）を設定します。設定しない場合はスキップされるため、`pnpm test:live` だけを指定した場合は Gateway スモークテストのみに集中できます。
- モデルの選択方法:
  - `OPENCLAW_LIVE_MODELS=modern` は、厳選されたシグナル価値の高い優先リストを実行します（[ライブ: モデルマトリクス](#live-model-matrix-what-we-cover)を参照）
  - `OPENCLAW_LIVE_MODELS=small` は、厳選された小規模モデルの優先リストを実行します
  - `OPENCLAW_LIVE_MODELS=all` は `modern` のエイリアスです
  - または `OPENCLAW_LIVE_MODELS="openai/gpt-5.6-luna,anthropic/claude-opus-4-6,..."`（カンマ区切りの許可リスト）
  - ローカル Ollama の小規模モデル実行では、デフォルトで `http://127.0.0.1:11434` が使用されます。LAN、カスタム、または Ollama Cloud エンドポイントの場合にのみ `OPENCLAW_LIVE_OLLAMA_BASE_URL` を設定してください。
  - 最新/全モデルおよび小規模モデルのスイープでは、デフォルトで各厳選リストの長さが上限になります。選択したプロファイルを網羅的にスイープするには `OPENCLAW_LIVE_MAX_MODELS=0` を設定し、上限を小さくするには正の数を設定します。
  - 網羅的なスイープでは、モデル直接接続テスト全体のタイムアウトとして `OPENCLAW_LIVE_TEST_TIMEOUT_MS` を使用します。デフォルト: 60 分。
  - モデル直接接続プローブは、デフォルトで 20 並列で実行されます。変更するには `OPENCLAW_LIVE_MODEL_CONCURRENCY` を設定します。
- プロバイダーの選択方法:
  - `OPENCLAW_LIVE_PROVIDERS="google,google-antigravity,google-gemini-cli"`（カンマ区切りの許可リスト）
- キーの取得元:
  - デフォルト: プロファイルストアと環境変数のフォールバック
  - **プロファイルストア**のみに限定するには `OPENCLAW_LIVE_REQUIRE_PROFILE_KEYS=1` を設定します
- このテストが存在する理由:
  - 「プロバイダー API が壊れている/キーが無効」と「Gateway エージェントパイプラインが壊れている」を切り分けます
  - 小規模で分離されたリグレッションテストを含みます（例: OpenAI Responses/Codex Responses の推論リプレイとツール呼び出しフロー）

### レイヤー 2: Gateway + 開発エージェントのスモークテスト（「@openclaw」が実際に行う処理）

- テスト: `src/gateway/gateway-models.profiles.live.test.ts`
- 目的:
  - プロセス内 Gateway を起動する
  - `agent:dev:*` セッションを作成/更新する（実行ごとにモデルをオーバーライド）
  - キーを持つモデルを順に処理し、以下を検証する:
    - 「意味のある」応答（ツールなし）
    - 実際のツール呼び出しが動作すること（読み取りプローブ）
    - 任意の追加ツールプローブ（実行+読み取りプローブ）
    - OpenAI のリグレッションパス（ツール呼び出しのみ -> フォローアップ）が引き続き動作すること
- プローブの詳細（障害を迅速に説明できるようにするため）:
  - `read` プローブ: テストはワークスペースに nonce ファイルを書き込み、エージェントにそのファイルを `read` して nonce をそのまま返すよう要求します。
  - `exec+read` プローブ: テストはエージェントに nonce を一時ファイルへ `exec` で書き込み、その後 `read` して読み返すよう要求します。
  - 画像プローブ: テストは生成した PNG（猫 + ランダム化されたコード）を添付し、モデルが `cat <CODE>` を返すことを期待します。
  - 実装リファレンス: `src/gateway/gateway-models.profiles.live.test.ts` および `test/helpers/live-image-probe.ts`。
- 有効化方法:
  - `pnpm test:live`（Vitest を直接呼び出す場合は `OPENCLAW_LIVE_TEST=1`）
- モデルの選択方法:
  - デフォルト: 厳選されたシグナル価値の高い（`modern`）優先リスト
  - `OPENCLAW_LIVE_GATEWAY_MODELS=small` は、厳選された小規模モデルのリストを Gateway とエージェントのパイプライン全体で実行します
  - `OPENCLAW_LIVE_GATEWAY_MODELS=all` は `modern` のエイリアスです
  - または、対象を絞るために `OPENCLAW_LIVE_GATEWAY_MODELS="provider/model"`（またはカンマ区切りのリスト）を設定します
  - 最新/全モデルおよび小規模モデルの Gateway スイープでは、デフォルトで各厳選リストの長さが上限になります。選択対象を網羅的にスイープするには `OPENCLAW_LIVE_GATEWAY_MAX_MODELS=0` を設定し、上限を小さくするには正の数を設定します。
- プロバイダーの選択方法（「OpenRouter の全モデル」を避ける）:
  - `OPENCLAW_LIVE_GATEWAY_PROVIDERS="google,google-antigravity,google-gemini-cli,openai,anthropic,zai,minimax"`（カンマ区切りの許可リスト）
- このライブテストでは、ツールと画像のプローブが常に有効です:
  - `read` プローブ + `exec+read` プローブ（ツール負荷テスト）
  - モデルが画像入力対応を公開している場合、画像プローブを実行します
  - フロー（概要）:
    - テストが「CAT」+ ランダムコードを含む小さな PNG を生成します（`test/helpers/live-image-probe.ts`）
    - `agent` `attachments: [{ mimeType: "image/png", content: "<base64>" }]` 経由で送信します
    - Gateway が添付ファイルを `images[]` に解析します（`src/gateway/server-methods/agent.ts` + `src/gateway/chat-attachments.ts`）
    - 組み込みエージェントがマルチモーダルなユーザーメッセージをモデルに転送します
    - 検証: 応答に `cat` + コードが含まれること（OCR の許容範囲: 軽微な誤りは許容）

<Tip>
使用しているマシンでテスト可能な対象（および正確な `provider/model` ID）を確認するには、次を実行します。

```bash
openclaw models list
openclaw models list --json
```

</Tip>

## ライブ: CLI バックエンドのスモークテスト（Claude、Gemini、またはその他のローカル CLI）

- テスト: `src/gateway/gateway-cli-backend.live.test.ts`
- 目的: デフォルト設定に触れずに、ローカル CLI バックエンドを使用して Gateway とエージェントのパイプラインを検証します。
- バックエンド固有のスモークテストのデフォルトは、所有する Plugin の `cli-backend.ts` 定義にあります。
- 有効化:
  - `pnpm test:live`（Vitest を直接呼び出す場合は `OPENCLAW_LIVE_TEST=1`）
  - `OPENCLAW_LIVE_CLI_BACKEND=1`
- デフォルト:
  - デフォルトのプロバイダー/モデル: `claude-cli/claude-sonnet-4-6`
  - コマンド/引数/画像の動作は、所有する CLI バックエンド Plugin のメタデータから取得されます。
- オーバーライド（任意）:
  - `OPENCLAW_LIVE_CLI_BACKEND_MODEL="claude-cli/claude-sonnet-4-6"`
  - `OPENCLAW_LIVE_CLI_BACKEND_COMMAND="/full/path/to/claude"`
  - `OPENCLAW_LIVE_CLI_BACKEND_ARGS='["-p","--output-format","json"]'`
  - 実際の画像添付を送信するには `OPENCLAW_LIVE_CLI_BACKEND_IMAGE_PROBE=1` を使用します（パスはプロンプトに挿入されます）。Docker レシピではデフォルトで無効です。
  - プロンプトへの挿入ではなく、画像ファイルのパスを CLI 引数として渡すには `OPENCLAW_LIVE_CLI_BACKEND_IMAGE_ARG="--image"` を使用します。
  - `IMAGE_ARG` が設定されている場合に画像引数を渡す方法を制御するには、`OPENCLAW_LIVE_CLI_BACKEND_IMAGE_MODE="repeat"`（または `"list"`）を使用します。
  - 2 ターン目を送信して再開フローを検証するには `OPENCLAW_LIVE_CLI_BACKEND_RESUME_PROBE=1` を使用します。
  - 選択したモデルが切り替え先をサポートしている場合に、Claude Sonnet -> Opus の同一セッション継続性プローブを有効にするには `OPENCLAW_LIVE_CLI_BACKEND_MODEL_SWITCH_PROBE=1` を使用します。Docker レシピを含め、デフォルトでは無効です。
  - MCP/ツールのループバックプローブを有効にするには `OPENCLAW_LIVE_CLI_BACKEND_MCP_PROBE=1` を使用します。Docker レシピではデフォルトで無効です。

例:

```bash
  OPENCLAW_LIVE_CLI_BACKEND=1 \
  OPENCLAW_LIVE_CLI_BACKEND_MODEL="claude-cli/claude-sonnet-4-6" \
  pnpm test:live src/gateway/gateway-cli-backend.live.test.ts
```

低コストの Gemini MCP 設定スモークテスト:

```bash
OPENCLAW_LIVE_TEST=1 \
  pnpm test:live src/agents/cli-runner/bundle-mcp.gemini.live.test.ts
```

これは Gemini に応答の生成を要求しません。OpenClaw が Gemini に渡すものと同じシステム
設定を書き込み、次に `gemini --debug mcp list` を実行して、保存済みの `transport: "streamable-http"`
サーバーが Gemini の HTTP MCP 形式に正規化され、ローカルのストリーム対応 HTTP MCP
サーバーに接続できることを検証します。

Docker レシピ:

```bash
pnpm test:docker:live-cli-backend
```

単一プロバイダー用 Docker レシピ:

```bash
pnpm test:docker:live-cli-backend:claude
pnpm test:docker:live-cli-backend:claude-subscription
pnpm test:docker:live-cli-backend:gemini
```

注:

- Docker ランナーは `scripts/test-live-cli-backend-docker.sh` にあります。
- リポジトリの Docker イメージ内で、非 root の `node` ユーザーとしてライブ CLI バックエンドスモークを実行します。
- 所有する Plugin から CLI スモークのメタデータを解決し、対応する Linux CLI パッケージ（`@anthropic-ai/claude-code` または `@google/gemini-cli`）を `OPENCLAW_DOCKER_CLI_TOOLS_DIR`（デフォルト: `~/.cache/openclaw/docker-cli-tools`）のキャッシュされた書き込み可能なプレフィックスにインストールします。
- `codex-cli` はバンドルされた CLI バックエンドではなくなりました。代わりに Codex app-server ランタイムで `openai/*` を使用してください（[ライブ: Codex app-server ハーネススモーク](#live-codex-app-server-harness-smoke)を参照）。
- `pnpm test:docker:live-cli-backend:claude-subscription` には、`claudeAiOauth.subscriptionType` を指定した `~/.claude/.credentials.json`、または `claude setup-token` の `CLAUDE_CODE_OAUTH_TOKEN` のいずれかを介した、ポータブルな Claude Code サブスクリプション OAuth が必要です。まず Docker 内で直接 `claude -p` を検証し、その後 Anthropic API キーの環境変数を保持せずに、Gateway CLI バックエンドのターンを 2 回実行します。このサブスクリプションレーンでは、サインイン済みサブスクリプションの使用量制限を消費し、Anthropic が OpenClaw のリリースなしに Claude Agent SDK / `claude -p` の請求およびレート制限の動作を変更できるため、Claude MCP/ツールおよび画像プローブはデフォルトで無効です。
- Claude と Gemini は、上記のフラグを介して同じプローブセット（テキストターン、画像分類、MCP `cron` ツール呼び出し、モデル切り替えの継続性）をサポートしますが、これらのプローブはいずれもデフォルトでは実行されません。必要に応じてフラグごとに有効化してください。

## ライブ: APNs HTTP/2 プロキシ到達性

- テスト: `src/infra/push-apns-http2.live.test.ts`
- 目的: ローカル HTTP CONNECT プロキシを介して Apple のサンドボックス APNs エンドポイントにトンネル接続し、APNs HTTP/2 検証リクエストを送信して、Apple の実際の `403 InvalidProviderToken` レスポンスがプロキシ経路を通って返されることを確認します。
- 有効化:
  - `OPENCLAW_LIVE_TEST=1 OPENCLAW_LIVE_APNS_REACHABILITY=1 pnpm test:live src/infra/push-apns-http2.live.test.ts`
- オプションのタイムアウト:
  - `OPENCLAW_LIVE_APNS_TIMEOUT_MS=30000`

## ライブ: ACP バインドスモーク（`/acp spawn ... --bind here`）

- テスト: `src/gateway/gateway-acp-bind.live.test.ts`
- 目的: ライブ ACP エージェントを使用して、実際の ACP 会話バインドフローを検証します。
  - `/acp spawn <agent> --bind here` を送信する
  - 合成メッセージチャネル会話をその場でバインドする
  - 同じ会話で通常のフォローアップを送信する
  - フォローアップがバインドされた ACP セッションのトランスクリプトに記録されることを確認する
- 有効化:
  - `pnpm test:live src/gateway/gateway-acp-bind.live.test.ts`
  - `OPENCLAW_LIVE_ACP_BIND=1`
- デフォルト:
  - Docker 内の ACP エージェント: `claude,codex,gemini`
  - 直接 `pnpm test:live ...` 用の ACP エージェント: `claude`
  - 合成チャネル: Slack DM 形式の会話コンテキスト
  - ACP バックエンド: `acpx`
- オーバーライド:
  - `OPENCLAW_LIVE_ACP_BIND_AGENT=claude`
  - `OPENCLAW_LIVE_ACP_BIND_AGENT=codex`
  - `OPENCLAW_LIVE_ACP_BIND_AGENT=droid`
  - `OPENCLAW_LIVE_ACP_BIND_AGENT=gemini`
  - `OPENCLAW_LIVE_ACP_BIND_AGENT=opencode`
  - `OPENCLAW_LIVE_ACP_BIND_AGENTS=claude,codex,gemini`
  - `OPENCLAW_LIVE_ACP_BIND_AGENT_COMMAND='npx -y @agentclientprotocol/claude-agent-acp@<version>'`
  - `OPENCLAW_LIVE_ACP_BIND_CODEX_MODEL=gpt-5.6-luna`
  - `OPENCLAW_LIVE_ACP_BIND_OPENCODE_MODEL=opencode/kimi-k2.6`
  - 画像プローブを強制的に有効化するには `OPENCLAW_LIVE_ACP_BIND_IMAGE_PROBE=1`（または `on`/`true`/`yes`）を使用します。それ以外の値では強制的に無効になります。`opencode` を除くすべてのエージェントでデフォルト実行されます。
  - `OPENCLAW_LIVE_ACP_BIND_REQUIRE_CRON=1`
  - `OPENCLAW_LIVE_ACP_BIND_PARENT_MODEL=openai/gpt-5.6-luna`
- 注記:
  - このレーンでは、管理者専用の合成送信元ルートフィールドを備えた Gateway `chat.send` サーフェスを使用するため、テストは外部に配信したように装うことなくメッセージチャネルのコンテキストを添付できます。
  - `OPENCLAW_LIVE_ACP_BIND_AGENT_COMMAND` が未設定の場合、テストは選択された ACP ハーネスエージェントについて、組み込みの `acpx` Plugin に内蔵されたエージェントレジストリを使用します。
  - 外部 ACP ハーネスではバインド/画像検証の成功後に MCP 呼び出しがキャンセルされる場合があるため、バインド済みセッションの Cron MCP 作成はデフォルトでベストエフォートです。バインド後の Cron プローブを厳格にするには `OPENCLAW_LIVE_ACP_BIND_REQUIRE_CRON=1` を設定します。

例:

```bash
OPENCLAW_LIVE_ACP_BIND=1 \
  OPENCLAW_LIVE_ACP_BIND_AGENT=claude \
  pnpm test:live src/gateway/gateway-acp-bind.live.test.ts
```

Docker 手順:

```bash
pnpm test:docker:live-acp-bind
```

単一エージェント用 Docker 手順:

```bash
pnpm test:docker:live-acp-bind:claude
pnpm test:docker:live-acp-bind:codex
pnpm test:docker:live-acp-bind:droid
pnpm test:docker:live-acp-bind:gemini
pnpm test:docker:live-acp-bind:opencode
```

Docker の注記:

- Docker ランナーは `scripts/test-live-acp-bind-docker.sh` にあります。
- デフォルトでは、集約されたライブ CLI エージェントに対して ACP バインドスモークを `claude`、`codex`、`gemini` の順に実行します。
- マトリックスを絞り込むには、`OPENCLAW_LIVE_ACP_BIND_AGENTS=claude`、`OPENCLAW_LIVE_ACP_BIND_AGENTS=codex`、`OPENCLAW_LIVE_ACP_BIND_AGENTS=droid`、`OPENCLAW_LIVE_ACP_BIND_AGENTS=gemini`、または `OPENCLAW_LIVE_ACP_BIND_AGENTS=opencode` を使用します。
- 対応する CLI 認証情報をコンテナにステージングし、要求されたライブ CLI（`@anthropic-ai/claude-code`、`@openai/codex`、`https://app.factory.ai/cli` を介した Factory Droid、`@google/gemini-cli`、または `opencode-ai`）が存在しない場合はインストールします。ACP バックエンド自体は、公式 `acpx` Plugin の組み込み `acpx/runtime` パッケージです。
- Droid Docker バリアントは設定用の `~/.factory` をステージングし、`FACTORY_API_KEY` を転送します。また、ローカルの Factory OAuth/キーリング認証はコンテナに移植できないため、その API キーが必要です。ACPX に内蔵された `droid exec --output-format acp` レジストリエントリを使用します。
- OpenCode Docker バリアントは、厳格な単一エージェント回帰レーンです。`OPENCLAW_LIVE_ACP_BIND_OPENCODE_MODEL`（デフォルト `opencode/kimi-k2.6`）から一時的な `OPENCODE_CONFIG_CONTENT` デフォルトモデルを書き込みます。
- 直接の `acpx` CLI 呼び出しは、Gateway 外部の動作を比較するための手動/回避策の経路にすぎません。Docker ACP バインドスモークでは、OpenClaw の組み込み `acpx` ランタイムバックエンドを実行します。

## ライブ: Codex app-server ハーネススモーク

- 目的: 通常の Gateway
  `agent` メソッドを介して、Plugin が所有する Codex ハーネスを検証します。
  - バンドルされた `codex` Plugin を読み込む
  - `/model <ref> --runtime codex` を介して OpenAI モデルを選択する
  - 要求された思考レベルで最初の Gateway エージェントターンを送信する
  - 同じ OpenClaw セッションに 2 回目のターンを送信し、app-server
    スレッドを再開できることを確認する
  - 同じ Gateway コマンド経路を介して `/codex status` と
    `/codex models` を実行する
  - オプションで、Guardian がレビューする昇格済みシェルプローブを 2 つ実行する。
    承認されるべき無害なコマンドと、拒否されてエージェントがユーザーに確認を求めるべき
    偽シークレットのアップロード
- テスト: `src/gateway/gateway-codex-harness.live.test.ts`
- 有効化: `OPENCLAW_LIVE_CODEX_HARNESS=1`
- ハーネスのベースラインモデル: `openai/gpt-5.6-luna`
- 新規 OpenAI API キー選択のデフォルト: `openai/gpt-5.6`
- デフォルトの思考レベル: `low`
- モデルのオーバーライド: `OPENCLAW_LIVE_CODEX_HARNESS_MODEL=openai/<model>`
- 思考レベルのオーバーライド: `OPENCLAW_LIVE_CODEX_HARNESS_THINKING=<level>`
- 非デフォルトモデルのエフォートアサーション:
  `OPENCLAW_LIVE_CODEX_HARNESS_EXPECTED_EFFORT=<level>`
- マトリックスのオーバーライド: `OPENCLAW_LIVE_CODEX_HARNESS_TARGETS=<model>=<thinking>,...`
- 認証モード: `OPENCLAW_LIVE_CODEX_HARNESS_AUTH=codex-auth`（デフォルト）は
  コピーされた Codex ログインを使用します。`api-key` は Codex app-server を介して `OPENAI_API_KEY` を使用します。
- オプションの画像プローブ: `OPENCLAW_LIVE_CODEX_HARNESS_IMAGE_PROBE=1`
- オプションの MCP/ツールプローブ: `OPENCLAW_LIVE_CODEX_HARNESS_MCP_PROBE=1`
- オプションの Guardian プローブ: `OPENCLAW_LIVE_CODEX_HARNESS_GUARDIAN_PROBE=1`
- オプションの再開ストレス: `OPENCLAW_LIVE_CODEX_HARNESS_RESUME_STRESS=1` は
  履歴ターンを 4 回追加した後、同じネイティブスレッド ID と会話履歴を
  必須としながら、Gateway と Codex app-server を 3 回終了して再起動します。
  制限付きの回数は、`OPENCLAW_LIVE_CODEX_HARNESS_RESUME_STRESS_HISTORY_TURNS`（1-20）と
  `OPENCLAW_LIVE_CODEX_HARNESS_RESUME_STRESS_RESTARTS`（1-10）でオーバーライドします。
- オプションのファンアウトストレス: `OPENCLAW_LIVE_CODEX_HARNESS_SUBAGENT_PROBE=1`
  と `OPENCLAW_LIVE_CODEX_HARNESS_SUBAGENT_COUNT`（1-12）を設定します。ハーネスは
  すべての子を同時に開始し、すべての実行が終了するまで待機して、各子の
  一意な応答とネイティブスレッド ID を検証します。
- オプションの Compaction ストレス: `OPENCLAW_LIVE_CODEX_HARNESS_COMPACTION_STRESS=1` は
  制限付きのネイティブツール出力を生成し、自動 Compaction イベントを必須とし、
  永続化された Compaction 回数と隠しマーカーの想起を検証し、
  Gateway と物理 Codex app-server を再起動してから、出力と
  Compaction の波を繰り返します。制限付きの処理量は、
  `OPENCLAW_LIVE_CODEX_HARNESS_COMPACTION_STRESS_TURNS`（1-8）と
  `OPENCLAW_LIVE_CODEX_HARNESS_LARGE_OUTPUT_BYTES`（100000-800000）で調整します。
- 完全な直接 API コンテキスト: `OPENCLAW_LIVE_CODEX_HARNESS_FULL_CONTEXT=1` は
  `922000` コンテキストと `700000` の合計 Compaction 制限を適用し、密度の高い制限付き
  ユーザーターンを送信し、波ごとに 2 つの明示的なネイティブ Compaction チェックポイントを実行して、
  各チェックポイント後も後続のターンを続行します。これには
  `OPENCLAW_LIVE_CODEX_HARNESS_AUTH=api-key` と絶対
  `OPENCLAW_LIVE_CODEX_HARNESS_MODEL_CATALOG` パスが必要です。Codex がオーバーライドを
  通常のカタログウィンドウに制限しないように、カタログは選択したモデルを
  `max_context_window: 922000` で公開する必要があります。上記の通常のしきい値削減
  ストレスでは、より厳格な自動 Compaction および隠しマーカー
  保持のアサーションを維持します。
- オプションのループリレーオプトアウトプローブ:
  `OPENCLAW_LIVE_CODEX_HARNESS_DISABLE_LOOP_RELAY=1`
- 要求された思考設定は、Codex がそのモデルについて提示する最も近いエフォートに
  マッピングされる場合があります。たとえば、Luna は `minimal` を `low` にマッピングします。
- 既知の Codex カタログモデルでは、その正確なネイティブエフォートが自動的に導出されます。
  不明なモデルのオーバーライドでは、想定されるマッピング済みエフォートを指定する必要があります。
- スモークはプロバイダー/モデル `agentRuntime.id: "codex"` を強制するため、壊れた Codex
  ハーネスが暗黙的に OpenClaw にフォールバックして成功することはありません。
- 認証: ローカルの Codex サブスクリプションログインによる Codex app-server 認証、または
  `OPENCLAW_LIVE_CODEX_HARNESS_AUTH=api-key` の場合は `OPENAI_API_KEY`。Docker は
  サブスクリプション実行用に `~/.codex/auth.json` と `~/.codex/config.toml` をコピーできます。

ローカル手順:

```bash
OPENCLAW_LIVE_CODEX_HARNESS=1 \
  OPENCLAW_LIVE_CODEX_HARNESS_IMAGE_PROBE=1 \
  OPENCLAW_LIVE_CODEX_HARNESS_MCP_PROBE=1 \
  OPENCLAW_LIVE_CODEX_HARNESS_GUARDIAN_PROBE=1 \
  OPENCLAW_LIVE_CODEX_HARNESS_MODEL=openai/gpt-5.6-luna \
  pnpm test:live -- src/gateway/gateway-codex-harness.live.test.ts
```

Docker 手順:

```bash
pnpm test:docker:live-codex-harness
```

再起動および履歴ストレス:

```bash
OPENCLAW_LIVE_CODEX_HARNESS_RESUME_STRESS=1 \
pnpm test:docker:live-codex-harness
```

ファンアウト、大規模出力、Compaction、および再起動ストレス:

```bash
OPENCLAW_LIVE_CODEX_HARNESS_AUTH=api-key \
  OPENCLAW_LIVE_CODEX_HARNESS_SUBAGENT_PROBE=1 \
  OPENCLAW_LIVE_CODEX_HARNESS_SUBAGENT_COUNT=8 \
  OPENCLAW_LIVE_CODEX_HARNESS_RESUME_STRESS=1 \
  OPENCLAW_LIVE_CODEX_HARNESS_COMPACTION_STRESS=1 \
  pnpm test:docker:live-codex-harness
```

完全なネイティブ Codex `922000` 入力バジェット Compaction ストレス:

```bash
OPENCLAW_LIVE_CODEX_HARNESS=1 \
  OPENCLAW_LIVE_CODEX_HARNESS_AUTH=api-key \
  OPENCLAW_LIVE_CODEX_HARNESS_FULL_CONTEXT=1 \
  OPENCLAW_LIVE_CODEX_HARNESS_MODEL_CATALOG=/absolute/path/to/models-api-1m.json \
  OPENCLAW_LIVE_CODEX_HARNESS_MODEL=openai/gpt-5.6-terra \
  OPENCLAW_LIVE_CODEX_HARNESS_THINKING=medium \
  OPENCLAW_LIVE_CODEX_HARNESS_COMPACTION_STRESS_TURNS=8 \
  OPENCLAW_LIVE_CODEX_HARNESS_LARGE_OUTPUT_BYTES=800000 \
  pnpm test:live -- src/gateway/gateway-codex-harness.live.test.ts
```

GPT-5.6 ネイティブ Codex マトリックス:

```bash
OPENCLAW_LIVE_CODEX_HARNESS_AUTH=api-key \
  OPENCLAW_LIVE_CODEX_HARNESS_TARGETS='openai/gpt-5.6-sol=ultra,openai/gpt-5.6-terra=ultra,openai/gpt-5.6-luna=max' \
  pnpm test:docker:live-codex-harness
```

## ライブ: OpenAI の反復 Compaction

- 目標: 埋め込み OpenClaw `openai-responses` エージェントループで、実際の自動 Compaction を
  少なくとも 2 回実行し、その後、永続的なマーカーが維持されることを検証します。
- テスト: `src/agents/sessions/agent-session.openai-compaction.live.test.ts`
- 有効化: `OPENCLAW_LIVE_OPENAI_COMPACTION=1`
- デフォルトモデル: `gpt-5.6-luna`
- モデルのオーバーライド: `OPENCLAW_LIVE_OPENAI_COMPACTION_MODEL=<model>`
- 通常のストレスモードでは、クライアントのコンテキスト予算を減らすことで、API コストを
  制限しながら、同じ実際の Compaction パスに到達します。
- フルコンテキストモードでは、クライアント予算を `922000`、Compaction 予約量を
  `222000` に設定するため、自動 Compaction は `700000` で開始されます。また、
  観測されたプロバイダー入力数が `272000` のロングコンテキスト料金境界を超えている必要があります。

制限付きライブ実行手順:

```bash
OPENCLAW_LIVE_TEST=1 \
  OPENCLAW_LIVE_OPENAI_COMPACTION=1 \
  pnpm test:live -- src/agents/sessions/agent-session.openai-compaction.live.test.ts
```

フル `922000` 入力予算の実行手順:

```bash
OPENCLAW_LIVE_TEST=1 \
  OPENCLAW_LIVE_OPENAI_COMPACTION=1 \
  OPENCLAW_LIVE_OPENAI_COMPACTION_FULL=1 \
  OPENCLAW_LIVE_OPENAI_COMPACTION_MODEL=gpt-5.6-terra \
  pnpm test:live -- src/agents/sessions/agent-session.openai-compaction.live.test.ts
```

<Warning>
フルモードでは意図的に OpenAI のロングコンテキスト料金境界を超えるため、
大規模な API 呼び出しが複数回行われる可能性があります。明示的に支出が承認されている場合にのみ使用してください。
</Warning>

新規 OpenAI API キーのデフォルト:

```bash
OPENCLAW_LIVE_GATEWAY_OPENAI_API_DEFAULT=1 \
  OPENCLAW_LIVE_GATEWAY_PROVIDERS=openai \
  OPENCLAW_LIVE_GATEWAY_THINKING=off \
  pnpm test:live -- src/gateway/gateway-models.profiles.live.test.ts
```

この検証では `OPENCLAW_LIVE_GATEWAY_MODELS` を未設定のままにし、
新規オンボーディングの推論選択シームを通じてモデルを解決し、`openai/gpt-5.6` をアサートした後、
解決されたモデルで実際の Gateway ターンを実行します。

GPT-5.6 埋め込み OpenClaw マトリクス:

```bash
OPENCLAW_LIVE_GATEWAY_THINKING=ultra \
  OPENCLAW_LIVE_GATEWAY_PROVIDERS=openai \
  OPENCLAW_LIVE_GATEWAY_MODELS='openai/gpt-5.6-sol,openai/gpt-5.6-terra,openai/gpt-5.6-luna' \
  pnpm test:live -- src/gateway/gateway-models.profiles.live.test.ts
```

Docker に関する注意事項:

- Docker ランナーは `scripts/test-live-codex-harness-docker.sh` にあります。
- `OPENAI_API_KEY` を渡し、存在する場合は Codex CLI 認証ファイルをコピーし、
  書き込み可能なマウント済み npm プレフィックスに
  `@openai/codex` をインストールし、ソースツリーをステージングした後、Codex ハーネスのライブテストのみを実行します。
- Docker では、イメージ、MCP/ツール、Guardian のプローブがデフォルトで有効です。より限定的なデバッグ実行が
  必要な場合は、`OPENCLAW_LIVE_CODEX_HARNESS_IMAGE_PROBE=0`、
  `OPENCLAW_LIVE_CODEX_HARNESS_MCP_PROBE=0`、または
  `OPENCLAW_LIVE_CODEX_HARNESS_GUARDIAN_PROBE=0` を設定してください。
- Docker は同じ明示的な Codex ランタイム設定を使用するため、レガシーエイリアスや OpenClaw の
  フォールバックによって Codex ハーネスのリグレッションが隠れることはありません。
- マトリクスのターゲットは、1 つのコンテナ内で順番に実行されます。Docker スクリプトは、
  デフォルトの 35 分のタイムアウトをターゲット数に応じて拡大します。外側のシェルまたは CI のタイムアウトでも、
  同じ合計時間を許容する必要があります。正規の CI では、各 GPT-5.6 ターゲットを個別のシャードに配置します。

### 推奨されるライブ実行手順

対象を限定した明示的な許可リストが、最も高速で不安定になりにくい方法です:

- 単一モデル、直接実行（Gateway なし）:
  - `OPENCLAW_LIVE_MODELS="openai/gpt-5.6-luna" pnpm test:live src/agents/models.profiles.live.test.ts`

- 小規模モデルの直接プロファイル:
  - `OPENCLAW_LIVE_MODELS=small pnpm test:live src/agents/models.profiles.live.test.ts`

- 小規模モデルの Gateway プロファイル:
  - `OPENCLAW_LIVE_GATEWAY_MODELS=small pnpm test:live src/gateway/gateway-models.profiles.live.test.ts`

- Ollama Cloud API スモークテスト:
  - `OPENCLAW_LIVE_TEST=1 OPENCLAW_LIVE_OLLAMA=1 OPENCLAW_LIVE_OLLAMA_BASE_URL=https://ollama.com OPENCLAW_LIVE_OLLAMA_MODEL=glm-5.1:cloud OPENCLAW_LIVE_OLLAMA_WEB_SEARCH=0 pnpm test:live -- extensions/ollama/ollama.live.test.ts`

- 単一モデルの Gateway スモークテスト:
  - `OPENCLAW_LIVE_GATEWAY_MODELS="openai/gpt-5.6-luna" pnpm test:live src/gateway/gateway-models.profiles.live.test.ts`

- 複数のプロバイダーにわたるツール呼び出し:
  - `OPENCLAW_LIVE_GATEWAY_MODELS="openai/gpt-5.6-luna,anthropic/claude-opus-4-6,google/gemini-3.5-flash,deepseek/deepseek-v4-flash,zai/glm-5.1,minimax/MiniMax-M3" pnpm test:live src/gateway/gateway-models.profiles.live.test.ts`

- Z.AI Coding Plan GLM-5.2 の直接スモークテスト:
  - `ZAI_CODING_LIVE_TEST=1 pnpm test:live src/agents/zai.live.test.ts`

- Google に重点を置いたテスト（Gemini API キー + Antigravity）:
  - Gemini（API キー）: `OPENCLAW_LIVE_GATEWAY_MODELS="google/gemini-3.5-flash" pnpm test:live src/gateway/gateway-models.profiles.live.test.ts`
  - Antigravity（OAuth）: `OPENCLAW_LIVE_GATEWAY_MODELS="google-antigravity/claude-opus-4-6-thinking,google-antigravity/gemini-3-pro-high" pnpm test:live src/gateway/gateway-models.profiles.live.test.ts`

- Google 適応的思考スモークテスト（プライベート QA CLI の `qa manual` — `OPENCLAW_ENABLE_PRIVATE_QA_CLI=1` とソースチェックアウトが必要です。[QA の概要](/ja-JP/concepts/qa-e2e-automation)を参照してください）:
  - Gemini 3 の動的デフォルト: `OPENCLAW_ENABLE_PRIVATE_QA_CLI=1 pnpm openclaw qa manual --provider-mode live-frontier --model google/gemini-3.1-pro-preview --alt-model google/gemini-3.1-pro-preview --message '/think adaptive Reply exactly: GEMINI_ADAPTIVE_OK' --timeout-ms 180000`
  - Gemini 2.5 の動的予算: `OPENCLAW_ENABLE_PRIVATE_QA_CLI=1 pnpm openclaw qa manual --provider-mode live-frontier --model google/gemini-2.5-flash --alt-model google/gemini-2.5-flash --message '/think adaptive Reply exactly: GEMINI25_ADAPTIVE_OK' --timeout-ms 180000`

注意事項:

- `google/...` は Gemini API（API キー）を使用します。
- `google-antigravity/...` は Antigravity OAuth ブリッジ（Cloud Code Assist 形式のエージェントエンドポイント）を使用します。
- `google-gemini-cli/...` はマシン上のローカル Gemini CLI を使用します（認証が別で、ツールにも固有の癖があります）。
- Gemini API と Gemini CLI の比較:
  - API: OpenClaw は HTTP 経由で Google がホストする Gemini API を呼び出します（API キー / プロファイル認証）。ほとんどのユーザーが「Gemini」と呼ぶのはこれです。
  - CLI: OpenClaw はローカルの `gemini` バイナリをシェルから実行します。独自の認証を持ち、動作が異なる場合があります（ストリーミング / ツール対応 / バージョン差異）。

## ライブ: モデルマトリクス（対象範囲）

ライブテストはオプトインであるため、固定の「CI モデルリスト」はありません。`OPENCLAW_LIVE_MODELS=modern` / `OPENCLAW_LIVE_GATEWAY_MODELS=modern`（およびそれらの `all` エイリアス）は、`src/agents/live-model-filter.ts` 内の `HIGH_SIGNAL_LIVE_MODEL_PRIORITY` から、次の優先順位で選定済みの優先リストを実行します:

| プロバイダー/モデル                            | 備考       |
| --------------------------------------------- | ---------- |
| `anthropic/claude-opus-5`                     |            |
| `anthropic/claude-opus-4-8`                   |            |
| `anthropic/claude-sonnet-5`                   |            |
| `anthropic/claude-sonnet-4-6`                 |            |
| `anthropic/claude-opus-4-7`                   |            |
| `google/gemini-3.1-pro-preview`               | Gemini API |
| `google/gemini-3.5-flash`                     | Gemini API |
| `cohere/command-a-plus-05-2026`               |            |
| `moonshot/kimi-k3`                            |            |
| `anthropic/claude-opus-4-6`                   |            |
| `deepseek/deepseek-v4-flash`                  |            |
| `deepseek/deepseek-v4-pro`                    |            |
| `minimax/MiniMax-M3`                          |            |
| `openai/gpt-5.5`                              |            |
| `openrouter/openai/gpt-5.2-chat`              |            |
| `openrouter/minimax/minimax-m2.7`             |            |
| `opencode-go/glm-5`                           |            |
| `openrouter/ai21/jamba-large-1.7`             |            |
| `xai/grok-4.5`                                |            |
| `xai/grok-4.20-0309-reasoning`                |            |
| `zai/glm-5.1`                                 |            |
| `fireworks/accounts/fireworks/models/glm-5p1` |            |
| `minimax-portal/minimax-m3`                   |            |

`SMALL_LIVE_MODEL_PRIORITY` の、選定済みの**小規模モデル**リスト（`OPENCLAW_LIVE_MODELS=small` / `OPENCLAW_LIVE_GATEWAY_MODELS=small`）:

| プロバイダー/モデル           |
| ---------------------------- |
| `lmstudio/qwen/qwen3.5-9b`   |
| `vllm/qwen/qwen3-8b`         |
| `sglang/qwen/qwen3-8b`       |
| `ollama/gemma3:4b`           |
| `openrouter/qwen/qwen3.5-9b` |
| `openrouter/z-ai/glm-5.1`    |
| `openrouter/z-ai/glm-5`      |
| `zai/glm-5.1`                |

モダンリストに関する注意事項:

- `codex` および `codex-cli` プロバイダーは、デフォルトのモダンスイープから除外されています（CLI バックエンド / ACP の動作を対象とし、上記で個別にテストされています）。`openai/gpt-5.5` 自体はデフォルトで Codex app-server ハーネスを経由します。[ライブ: Codex app-server ハーネスのスモークテスト](#live-codex-app-server-harness-smoke)を参照してください。
- `fireworks`、`google`、`openrouter`、および `xai` は、モダンスイープで明示的に選定されたモデル ID のみを実行します（「このプロバイダーのすべてのモデル」への自動展開はありません）。
- イメージプローブを実行するには、`OPENCLAW_LIVE_GATEWAY_MODELS` に画像対応モデル（Claude/Gemini/OpenAI ファミリーのビジョンバリアントなど）を少なくとも 1 つ含めてください。

選定した複数プロバイダーのセットで、ツールと画像を使用する Gateway スモークテストを実行します:

```bash
OPENCLAW_LIVE_GATEWAY_MODELS="openai/gpt-5.6-luna,anthropic/claude-opus-4-6,google/gemini-3.1-pro-preview,google/gemini-3.5-flash,google-antigravity/claude-opus-4-6-thinking,deepseek/deepseek-v4-flash,zai/glm-5.1,minimax/MiniMax-M3" pnpm test:live src/gateway/gateway-models.profiles.live.test.ts
```

選定済みリスト以外で任意に追加できる対象（追加できれば望ましいもの。利用可能な「ツール」対応モデルを選択してください）:

- Mistral: `mistral/...`
- Cerebras: `cerebras/...`（アクセス権がある場合）
- LM Studio: `lmstudio/...`（ローカル。ツール呼び出しは API モードに依存します）

### アグリゲーター / 代替 Gateway

キーを有効にしている場合は、次の経路でもテストできます:

- OpenRouter: `openrouter/...`（数百のモデル。ツールと画像に対応する候補を探すには `openclaw models scan` を使用してください）
- OpenCode: Zen には `opencode/...`、Go には `opencode-go/...`（`OPENCODE_API_KEY` / `OPENCODE_ZEN_API_KEY` による認証）

認証情報 / 設定がある場合、ライブマトリクスに追加できるその他のプロバイダー:

- 組み込み: `anthropic`、`cerebras`、`github-copilot`、`google`、`google-antigravity`、`google-gemini-cli`、`google-vertex`、`groq`、`mistral`、`openai`、`openrouter`、`opencode`、`opencode-go`、`xai`、`zai`
- `models.providers` 経由（カスタムエンドポイント）: `minimax`（クラウド / API）に加え、OpenAI/Anthropic 互換の任意のプロキシ（LM Studio、vLLM、LiteLLM など）

<Tip>
ドキュメントに「すべてのモデル」をハードコードしないでください。正式なリストは、マシン上で `discoverModels(...)` が返す内容と、利用可能なキーによって決まります。
</Tip>

## 認証情報（絶対にコミットしないでください）

ライブテストは CLI と同じ方法で認証情報を検出します。実際には、次のことを意味します:

- CLI が動作する場合、ライブテストも同じキーを検出するはずです。
- ライブテストで「認証情報なし」と表示された場合は、`openclaw models list` / モデル選択をデバッグする場合と同じ方法でデバッグしてください。

- エージェントごとの認証プロファイル: `~/.openclaw/agents/<agentId>/agent/auth-profiles.json`（ライブテストでいう「プロファイルキー」とはこれを指します）
- 設定: `~/.openclaw/openclaw.json`（または `OPENCLAW_CONFIG_PATH`）
- レガシー OAuth ディレクトリ: `~/.openclaw/credentials/`（存在する場合はステージング済みライブホームにコピーされますが、メインのプロファイルキーストアではありません）
- ローカルのライブ実行では、アクティブな設定（`agents.*.workspace` / `agentDir` のオーバーライドを除去済み）と、各エージェントの `auth-profiles.json` のみが一時テストホームにコピーされます。エージェントのディレクトリ内のそれ以外の内容はコピーされないため、`workspace/` および `sandboxes/` のデータがステージング済みホームに渡ることはありません。さらに、レガシーの `credentials/` ディレクトリと、対応する外部 CLI 認証ファイル / ディレクトリ（`.claude.json`、`.claude/.credentials.json`、`.claude/settings*.json`、`.claude/backups`、`.codex/auth.json`、`.codex/config.toml`、`.gemini`、`.minimax`）もコピーされます。

環境変数のキーを使用する場合は、ローカルテストの前にエクスポートするか、
以下の Docker ランナーで明示的な `OPENCLAW_PROFILE_FILE` を使用してください。

## Deepgram ライブ（音声文字起こし）

- テスト: `extensions/deepgram/audio.live.test.ts`
- 有効化: `DEEPGRAM_API_KEY=... DEEPGRAM_LIVE_TEST=1 pnpm test:live extensions/deepgram/audio.live.test.ts`

## BytePlus コーディングプランのライブテスト

- テスト: `extensions/byteplus/live.test.ts`
- 有効化: `BYTEPLUS_API_KEY=... BYTEPLUS_LIVE_TEST=1 pnpm test:live extensions/byteplus/live.test.ts`
- 任意のモデルオーバーライド: `BYTEPLUS_CODING_MODEL=ark-code-latest`

## ComfyUI ワークフローメディアのライブテスト

- テスト: `extensions/comfy/comfy.live.test.ts`
- 有効化: `OPENCLAW_LIVE_TEST=1 COMFY_LIVE_TEST=1 pnpm test:live -- extensions/comfy/comfy.live.test.ts`
- 対象範囲:
  - バンドルされた comfy の画像、動画、および `music_generate` のパスを実行します
  - `plugins.entries.comfy.config.<capability>` が設定されていない場合は、各機能をスキップします
  - comfy のワークフロー送信、ポーリング、ダウンロード、または Plugin 登録を変更した後に役立ちます

## 画像生成のライブテスト

- テスト: `test/image-generation.runtime.live.test.ts`
- コマンド: `pnpm test:live test/image-generation.runtime.live.test.ts`
- ハーネス: `pnpm test:live:media image`
- 範囲:
  - 登録済みのすべての画像生成プロバイダー Plugin を列挙
  - プローブの前に、すでにエクスポートされているプロバイダー環境変数を使用
  - デフォルトでは保存済み認証プロファイルよりもライブ／環境変数の API キーを優先するため、`auth-profiles.json` 内の古いテストキーが実際のシェル認証情報を隠すことはない
  - 使用可能な認証／プロファイル／モデルがないプロバイダーをスキップ
  - 設定済みの各プロバイダーを共有画像生成ランタイムで実行:
    - `<provider>:generate`
    - プロバイダーが編集対応を宣言している場合は `<provider>:edit`
- 現在対象となる同梱プロバイダー:
  - `deepinfra`
  - `fal`
  - `google`
  - `minimax`
  - `openai`
  - `openrouter`
  - `vydra`
  - `xai`
- 任意の絞り込み:
  - `OPENCLAW_LIVE_IMAGE_GENERATION_PROVIDERS="openai,google,openrouter,xai"`
  - `OPENCLAW_LIVE_IMAGE_GENERATION_PROVIDERS="deepinfra"`
  - `OPENCLAW_LIVE_IMAGE_GENERATION_MODELS="openai/gpt-image-2,google/gemini-3.1-flash-image,openrouter/google/gemini-3.1-flash-image-preview,xai/grok-imagine-image"`
  - `OPENCLAW_LIVE_IMAGE_GENERATION_CASES="google:flash-generate,google:pro-edit,openrouter:generate,xai:default-generate,xai:default-edit"`
- 任意の認証動作:
  - プロファイルストア認証を強制し、環境変数のみのオーバーライドを無視するには `OPENCLAW_LIVE_REQUIRE_PROFILE_KEYS=1`

出荷される CLI パスについては、プロバイダー／ランタイムのライブテストが
成功した後に `infer` スモークテストを追加します:

```bash
OPENCLAW_LIVE_TEST=1 OPENCLAW_LIVE_INFER_CLI_TEST=1 pnpm test:live -- test/image-generation.infer-cli.live.test.ts
openclaw infer image providers --json
openclaw infer image generate \
  --model google/gemini-3.1-flash-image \
  --prompt "テキストなし、白い背景に青い正方形が 1 つある、最小限のフラットなテスト画像。" \
  --output ./openclaw-infer-image-smoke.png \
  --json
```

これは、CLI 引数の解析、設定／デフォルトエージェントの解決、同梱
Plugin の有効化、共有画像生成ランタイム、およびライブプロバイダーへの
リクエストを対象とします。Plugin の依存関係は、ランタイムの読み込み前に存在している必要があります。

## 音楽生成ライブ

- テスト: `extensions/music-generation-providers.live.test.ts`
- 有効化: `OPENCLAW_LIVE_TEST=1 pnpm test:live -- extensions/music-generation-providers.live.test.ts`
- ハーネス: `pnpm test:live:media music`
- 範囲:
  - 共有の同梱音楽生成プロバイダーパスを実行
  - 現在は `fal`、`google`、`minimax`、および `openrouter` が対象
  - プローブの前に、すでにエクスポートされているプロバイダー環境変数を使用
  - デフォルトでは保存済み認証プロファイルよりもライブ／環境変数の API キーを優先するため、`auth-profiles.json` 内の古いテストキーが実際のシェル認証情報を隠すことはない
  - 使用可能な認証／プロファイル／モデルがないプロバイダーをスキップ
  - 利用可能な場合、宣言された両方のランタイムモードを実行:
    - プロンプトのみの入力で `generate`
    - プロバイダーが `capabilities.edit.enabled` を宣言している場合は `edit`
  - `comfy` には独立した専用ライブファイルがあり、この共有スイープには含まれない
- 任意の絞り込み:
  - `OPENCLAW_LIVE_MUSIC_GENERATION_PROVIDERS="google,minimax"`
  - `OPENCLAW_LIVE_MUSIC_GENERATION_MODELS="google/lyria-3-clip-preview,minimax/music-2.6"`
- 任意の認証動作:
  - プロファイルストア認証を強制し、環境変数のみのオーバーライドを無視するには `OPENCLAW_LIVE_REQUIRE_PROFILE_KEYS=1`

## 動画生成ライブ

- テスト: `extensions/video-generation-providers.live.test.ts`
- 有効化: `OPENCLAW_LIVE_TEST=1 pnpm test:live -- extensions/video-generation-providers.live.test.ts`
- ハーネス: `pnpm test:live:media video`
- 範囲:
  - `alibaba`、`byteplus`、`deepinfra`、`fal`、`google`、`minimax`、`openai`、`openrouter`、`pixverse`、`qwen`、`runway`、`together`、`vydra`、`xai` にわたって共有の同梱動画生成プロバイダーパスを実行
  - デフォルトではリリースに安全なスモークパスを使用: プロバイダーごとに 1 回のテキストから動画へのリクエスト、1 秒間のロブスタープロンプト、および `OPENCLAW_LIVE_VIDEO_GENERATION_TIMEOUT_MS` で指定されるプロバイダーごとの操作上限（デフォルトは `180000`）
  - プロバイダー側のキュー待ち時間がリリース時間の大部分を占める可能性があるため、デフォルトでは FAL をスキップ。明示的に実行するには `OPENCLAW_LIVE_VIDEO_GENERATION_PROVIDERS="fal"` を渡す（またはスキップリストを空にする）
  - プローブの前に、すでにエクスポートされているプロバイダー環境変数を使用
  - デフォルトでは保存済み認証プロファイルよりもライブ／環境変数の API キーを優先するため、`auth-profiles.json` 内の古いテストキーが実際のシェル認証情報を隠すことはない
  - 使用可能な認証／プロファイル／モデルがないプロバイダーをスキップ
  - デフォルトでは `generate` のみを実行
  - 利用可能な場合に宣言済みの変換モードも実行するには、`OPENCLAW_LIVE_VIDEO_GENERATION_FULL_MODES=1` を設定:
    - プロバイダーが `capabilities.imageToVideo.enabled` を宣言し、選択したプロバイダー／モデルが共有スイープでバッファを基にしたローカル画像入力を受け付ける場合は `imageToVideo`
    - プロバイダーが `capabilities.videoToVideo.enabled` を宣言し、選択したプロバイダー／モデルが共有スイープでバッファを基にしたローカル動画入力を受け付ける場合は `videoToVideo`
  - 共有スイープで現在宣言済みだがスキップされる `imageToVideo` プロバイダー:
    - `vydra`（このレーンではバッファを基にしたローカル画像入力はサポートされない）
  - Vydra プロバイダー固有の対象範囲:
    - `OPENCLAW_LIVE_TEST=1 OPENCLAW_LIVE_VYDRA_VIDEO=1 pnpm test:live -- extensions/vydra/vydra.live.test.ts`
    - このファイルは、`veo3` のテキストから動画への処理に加え、デフォルトでリモート画像 URL フィクスチャを使用する `kling` の画像から動画へのレーンも実行する（上書きするには `OPENCLAW_LIVE_VYDRA_KLING_IMAGE_URL`）。
  - xAI プロバイダー固有の対象範囲:
    - `OPENCLAW_LIVE_TEST=1 OPENCLAW_LIVE_XAI_VIDEO=1 pnpm test:live -- extensions/xai/xai.live.test.ts -t "classic Grok Imagine"`
    - classic ケースでは、最初に正方形のローカル PNG の先頭フレームを生成し、ジオメトリを省略して 1 秒間の画像から動画へのクリップをリクエストし、完了までポーリングして、ダウンロードしたバッファを検証する。
    - `OPENCLAW_LIVE_TEST=1 OPENCLAW_LIVE_XAI_VIDEO=1 pnpm test:live -- extensions/xai/xai.live.test.ts -t "Grok Imagine Video 1.5"`
    - 1.5 ケースでは、ローカル PNG の先頭フレームを生成し、1 秒間の 1080P 画像から動画へのクリップをリクエストし、完了までポーリングして、ダウンロードしたバッファを検証する。
  - 現在の `videoToVideo` ライブ対象範囲:
    - 選択したモデルが `gen4_aleph` に解決される場合のみ `runway`
  - 共有スイープで現在宣言済みだがスキップされる `videoToVideo` プロバイダー:
    - `alibaba`、`google`、`openai`、`qwen`、`xai`。これらのパスでは現在、バッファを基にしたローカル入力ではなく、リモートの `http(s)` 参照 URL が必要なため
- 任意の絞り込み:
  - `OPENCLAW_LIVE_VIDEO_GENERATION_PROVIDERS="deepinfra,google,openai,runway"`
  - `OPENCLAW_LIVE_VIDEO_GENERATION_MODELS="google/veo-3.1-fast-generate-preview,openai/sora-2,runway/gen4_aleph"`
  - FAL を含むすべてのプロバイダーをデフォルトのスイープに含めるには `OPENCLAW_LIVE_VIDEO_GENERATION_SKIP_PROVIDERS=""`
  - 積極的なスモーク実行のために各プロバイダーの操作上限を減らすには `OPENCLAW_LIVE_VIDEO_GENERATION_TIMEOUT_MS=60000`
- 任意の認証動作:
  - プロファイルストア認証を強制し、環境変数のみのオーバーライドを無視するには `OPENCLAW_LIVE_REQUIRE_PROFILE_KEYS=1`

## メディアライブハーネス

- コマンド: `pnpm test:live:media`
- エントリポイント: `test/e2e/qa-lab/media/hosted-media-provider-live.ts`。選択したスイートごとに `pnpm test:live -- <suite-test-file>` を実行するため、Heartbeat と静音モードの動作は他の `pnpm test:live` 実行と一貫性を保つ。
- 目的:
  - 1 つのリポジトリネイティブなエントリポイントを通じて、共有の画像、音楽、動画ライブスイートを実行
  - `~/.profile` から不足しているプロバイダー環境変数を自動読み込み
  - デフォルトでは、現在使用可能な認証を持つプロバイダーに各スイートを自動的に絞り込み
- フラグ:
  - `--providers <csv>` はグローバルプロバイダーフィルター。`--image-providers`／`--music-providers`／`--video-providers` はフィルターの適用範囲を 1 つのスイートに限定
  - `--all-providers` は認証に基づく自動フィルターをスキップ
  - フィルタリング後に実行可能なプロバイダーが残っていない場合、`--allow-empty` は `0` で終了
  - `--quiet`／`--no-quiet` は `test:live` にそのまま渡される
- 例:
  - `pnpm test:live:media`
  - `pnpm test:live:media image video --providers openai,google,minimax`
  - `pnpm test:live:media video --video-providers openai,runway --all-providers`
  - `pnpm test:live:media music --quiet`

## 関連項目

- [テスト](/ja-JP/help/testing) - 単体、統合、QA、Docker スイート
