---
read_when:
    - 特定の `openclaw onboard` ステップの詳細な動作が必要です
    - オンボーディング結果をデバッグする、またはオンボーディングクライアントを統合する場合
sidebarTitle: CLI reference
summary: openclaw onboard のステップごとの動作：各ステップの処理内容、書き込まれる設定、内部動作
title: CLI セットアップリファレンス
x-i18n:
    generated_at: "2026-07-26T10:04:42Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 41bb9243ac7276b383274f4c27e3782b29e8ecf9d883229a44e3ab59aca5a34f
    source_path: start/wizard-cli-reference.md
    workflow: 16
---

このページでは、オンボーディングの動作、出力、内部処理を手順ごとに説明します。
詳しい手順については、[オンボーディング（CLI）](/ja-JP/start/wizard)を参照してください。CLI フラグの完全な
リファレンス（すべての `--flag`、非対話型の例、プロバイダー固有の
コマンド）については、[`openclaw onboard`](/ja-JP/cli/onboard)を参照してください。

## ウィザードの動作

ローカルモード（デフォルト）では、次の手順を案内します。

- モデルと認証のセットアップ（Anthropic、OpenAI Code サブスクリプション OAuth、xAI、OpenCode、カスタムエンドポイント、その他のプロバイダー所有の認証フロー）
- ワークスペースの場所とブートストラップファイル
- Gateway の設定（ポート、バインド、認証、Tailscale）
- チャンネルとプロバイダー（Discord、Feishu、Google Chat、iMessage、Mattermost、Microsoft Teams、QQ Bot、Signal、Slack、Telegram、WhatsApp、その他の同梱または Plugin チャンネル）
- ウェブ検索プロバイダー（任意）
- デーモンのインストール（LaunchAgent、systemd ユーザーユニット、または Startup フォルダーへのフォールバックを備えたネイティブ Windows の Scheduled Task）
- ヘルスチェック
- Skills のセットアップ

リモートモードでは、このマシンから別の場所にある Gateway へ接続するよう設定します。リモートホストには
何もインストールせず、変更も加えません。

## ローカルフローの詳細

<Steps>
  <Step title="既存設定の検出">
    - `~/.openclaw/openclaw.json` が存在する場合は、**現在の値を維持**、**確認して更新**、または**セットアップ前にリセット**を選択します。
    - 明示的にリセットを選択する（または `--reset` を渡す）場合を除き、ウィザードを再実行しても何も消去されません。
    - CLI の `--reset` はデフォルトで `config+creds+sessions` になります。ワークスペースも削除するには `--reset-scope full` を使用します。
    - 設定が無効であるか、レガシーキーが含まれている場合、ウィザードは停止し、続行する前に `openclaw doctor` を実行するよう求めます。
    - リセットでは状態をゴミ箱へ移動し（直接削除することはありません）、次の範囲を選択できます。
      - 設定のみ
      - 設定 + 認証情報 + セッション
      - 完全リセット（ワークスペースも削除）

  </Step>
  <Step title="モデルと認証">
    - すべての選択肢については、[認証とモデルのオプション](#auth-and-model-options)を参照してください。

  </Step>
  <Step title="ワークスペース">
    - デフォルトは `~/.openclaw/workspace`（設定可能）。
    - 初回実行時のブートストラップに必要なワークスペースファイルを作成します。
    - 再実行時に既存のエージェント一覧がある場合、移動を明示的に確認しない限り、
      フリート全体のワークスペースが維持されます。非対話型の再実行では警告を表示し、
      現在の値を維持します。
    - ワークスペースのレイアウト：[エージェントワークスペース](/ja-JP/concepts/agent-workspace)。

  </Step>
  <Step title="Gateway">
    - ポート、バインド、認証モード、Tailscale への公開について入力を求めます。
    - 推奨：local loopback でもトークン認証を有効にし、ローカルの WS クライアントにも認証を必須にします。
    - トークンモードの対話型セットアップでは、次の選択肢があります。
      - **平文トークンを生成して保存**（デフォルト）
      - **SecretRef を使用**（オプトイン）
    - パスワードモードの対話型セットアップでも、平文または SecretRef での保存がサポートされます。
    - 非対話型でトークンを SecretRef に保存するパス：`--gateway-token-ref-env <ENV_VAR>`。
      - オンボーディングプロセスの環境に、空でない環境変数が必要です。
      - `--gateway-token` と併用できません。
    - すべてのローカルプロセスを完全に信頼できる場合にのみ、認証を無効にしてください。
    - local loopback 以外へのバインドでは、引き続き認証が必要です。

  </Step>
  <Step title="チャンネル">
    - [WhatsApp](/ja-JP/channels/whatsapp)：任意の QR ログイン
    - [Telegram](/ja-JP/channels/telegram)：ボットトークン
    - [Discord](/ja-JP/channels/discord)：ボットトークン
    - [Google Chat](/ja-JP/channels/googlechat)：サービスアカウント JSON + Webhook オーディエンス
    - [Mattermost](/ja-JP/channels/mattermost)：ボットトークン + ベース URL
    - [Signal](/ja-JP/channels/signal)：任意の `signal-cli` インストール + アカウント設定
    - [iMessage](/ja-JP/channels/imessage)：`imsg` CLI パス + Messages DB へのアクセス。Gateway が Mac 以外で動作する場合は SSH ラッパーを使用します
    - DM のセキュリティ：デフォルトはペアリングです。最初の DM でコードが送信されます。
      `openclaw pairing approve <channel> <code>` で承認するか、許可リストを使用します。
  </Step>
  <Step title="ウェブ検索">
    - プロバイダー（Brave、DuckDuckGo、Exa、Firecrawl、Gemini、Grok、Kimi、MiniMax Search、Ollama Web Search、Perplexity、SearXNG、Tavily）を選択するか、スキップします。
    - `--skip-search` でこの手順をスキップできます。後から `openclaw configure --section web` で再設定できます。

  </Step>
  <Step title="デーモンのインストール">
    - macOS：LaunchAgent
      - ログイン済みのユーザーセッションが必要です。ヘッドレス環境では、カスタム LaunchDaemon（同梱されていません）を使用します。
    - Linux および WSL2 経由の Windows：systemd ユーザーユニット
      - ログアウト後も Gateway が動作し続けるよう、ウィザードは `loginctl enable-linger <user>` を試行します。
      - sudo を求める場合があります（`/var/lib/systemd/linger` に書き込みます）。最初に sudo なしで試行します。
    - ネイティブ Windows：最初に Scheduled Task
      - タスクの作成が拒否された場合、OpenClaw はユーザー単位の Startup フォルダーのログイン項目へフォールバックし、Gateway を直ちに起動します。
      - より適切なスーパーバイザー状態を提供するため、Scheduled Tasks が引き続き推奨されます。
    - ランタイムの選択：OpenClaw の標準ランタイム状態ストアでは `node:sqlite` を使用するため、Node が必要です。

  </Step>
  <Step title="ヘルスチェック">
    - 必要に応じて Gateway を起動し、`openclaw health` を実行します。
    - `openclaw status --deep` を指定すると、対応している場合はチャンネルプローブも含め、稼働中の Gateway のヘルスプローブがステータス出力に追加されます。

  </Step>
  <Step title="Skills">
    - 利用可能な Skills を読み込み、要件を確認します。
    - Node マネージャーとして npm、pnpm、bun のいずれかを選択できます。
    - 必要なインストーラーが利用可能な場合、信頼済みの同梱 Skills に必要な任意の
      依存関係をインストールします。
    - 利用できない Homebrew、uv、Go のインストーラーをスキップし、影響を受ける
      Skills を手動セットアップの案内とともにグループ化します。不足している前提条件を
      インストールした後、`openclaw doctor` を実行してください。

  </Step>
  <Step title="完了">
    - iOS、Android、macOS アプリのオプションを含む、概要と次の手順を表示します。

  </Step>
</Steps>

<Note>
GUI が検出されない場合、ウィザードはブラウザーを開く代わりに、Control UI 用の SSH ポートフォワーディング手順を表示します。
Control UI のアセットがない場合、ウィザードはビルドを試行します。フォールバックは `pnpm ui:build` です（UI の依存関係を自動インストールします）。
</Note>

## リモートモードの詳細

リモートモードでは、このマシンから別の場所にある Gateway へ接続するよう設定します。リモートホストには
何もインストールせず、変更も加えません。

設定する項目：

- リモート Gateway の URL（`ws://...` または `wss://...`）
- リモート Gateway の設定と一致する、トークン、パスワード、または認証なし

<Steps>
  <Step title="検出（任意）">
    `dns-sd`（macOS）または `avahi-browse`（Linux）が利用可能な場合、オンボーディングでは
    URL の手動入力に切り替える前に、Bonjour/mDNS の Gateway ビーコンを検索するか選択できます。
    設定されている場合は、広域 DNS-SD 検出も試行されます。ドキュメント：[Gateway の検出](/ja-JP/gateway/discovery)、[Bonjour](/ja-JP/gateway/bonjour)。
  </Step>
  <Step title="接続方法">
    ビーコンを選択した場合は、直接 WebSocket または SSH トンネルを選択します。
    - **直接**：`wss://` 経由で接続し、検出された
      TLS フィンガープリントを信頼するか確認します（初回使用時の信頼によるピン留め。承認した場合のみピン留めされます）。
    - **SSH トンネル**：最初に実行する `ssh -N -L 18789:127.0.0.1:18789 <user>@<host>`
      コマンドを表示し、その後 local loopback のトンネルエンドポイントへ接続します。
  </Step>
  <Step title="認証">
    トークン（推奨）、パスワード、または認証なしを選択し、必要に応じて平文ではなく
    SecretRef として保存します。
  </Step>
</Steps>

<Note>
Gateway が local loopback 専用で検出できない場合は、SSH トンネリングまたは tailnet を手動で使用します。
平文の `ws://` は local loopback、プライベート IP リテラル、`.local`、Tailnet の `*.ts.net` URL で使用できます。その他のプライベート DNS 名には `OPENCLAW_ALLOW_INSECURE_PRIVATE_WS=1` が必要です。
</Note>

## 認証とモデルのオプション

対話型オンボーディングでプロバイダーのセットアップ手順が失敗した場合（たとえば、ローカルでサインインしていない状態で CLI 再利用オプションを選択した場合）、
ウィザードは終了せず、エラーを表示してプロバイダー選択画面に戻ります。
自動化用に明示的に `--auth-choice` を指定した実行では、引き続き即座に失敗します。

<AccordionGroup>
  <Accordion title="Anthropic API キー">
    `ANTHROPIC_API_KEY` が存在する場合はそれを使用し、存在しない場合はキーの入力を求めて、デーモンで使用できるよう保存します。
  </Accordion>
  <Accordion title="Anthropic Claude CLI">
    対話型のオンボーディングおよび設定で推奨されるローカルパスです。利用可能な場合は、既存の Claude CLI サインインを再利用します。
  </Accordion>
  <Accordion title="OpenAI Code サブスクリプション（OAuth）">
    ブラウザーフローで `code#state` を貼り付けます。

    プライマリモデルがない新規セットアップでは、Codex ランタイムを通じて
    `agents.defaults.model` を `openai/gpt-5.6-sol` に設定します。

  </Accordion>
  <Accordion title="OpenAI Code サブスクリプション（デバイスペアリング）">
    有効期間の短いデバイスコードを使用するブラウザーペアリングフローです。

    プライマリモデルがない新規セットアップでは、Codex ランタイムを通じて
    `agents.defaults.model` を `openai/gpt-5.6-sol` に設定します。

  </Accordion>
  <Accordion title="OpenAI API キー">
    `OPENAI_API_KEY` が存在する場合はそれを使用し、存在しない場合はキーの入力を求めて、認証プロファイルに認証情報を保存します。

    プライマリモデルがない新規セットアップでは、`agents.defaults.model` を
    `openai/gpt-5.6` に設定します。修飾子のない直接 API モデル ID は Sol ティアとして解決されます。

    OpenAI を追加または再認証しても、`openai/gpt-5.5` を含む、明示的に指定された既存のプライマリ
    モデルは維持されます。アカウントで GPT-5.6 が提供されていない場合は、
    `openai/gpt-5.5` を明示的に選択してください。OpenClaw が暗黙的にダウングレードすることはありません。

  </Accordion>
  <Accordion title="xAI (Grok) OAuth">
    対象となる SuperGrok または X Premium アカウントでブラウザからサインインします。これは
    ほとんどのユーザーに推奨される xAI の方法です。OpenClaw は、Grok モデル、Grok `web_search`、
    `x_search`、および `code_execution` 用に、生成された認証プロファイルを保存します。
  </Accordion>
  <Accordion title="xAI (Grok) デバイスコード">
    localhost コールバックの代わりに短いコードを使用する、リモート環境に適したブラウザサインインです。
    SSH、Docker、または VPS ホストから使用してください。
  </Accordion>
  <Accordion title="xAI (Grok) API キー">
    `XAI_API_KEY` の入力を求め、xAI をモデルプロバイダーとして設定します。サブスクリプション OAuth ではなく
    xAI Console API キーを使用する場合に選択してください。
  </Accordion>
  <Accordion title="OpenCode">
    `OPENCODE_API_KEY`（または `OPENCODE_ZEN_API_KEY`）の入力を求め、Zen または Go カタログを選択できます（1 つの API キーで両方を利用できます）。
    セットアップ URL: [opencode.ai/auth](https://opencode.ai/auth)。
  </Accordion>
  <Accordion title="API キー（汎用）">
    キーを保存します。
  </Accordion>
  <Accordion title="Vercel AI Gateway">
    `AI_GATEWAY_API_KEY` の入力を求めます。
    詳細: [Vercel AI Gateway](/ja-JP/providers/vercel-ai-gateway)。
  </Accordion>
  <Accordion title="Cloudflare AI Gateway">
    アカウント ID、Gateway ID、および `CLOUDFLARE_AI_GATEWAY_API_KEY` の入力を求めます。
    詳細: [Cloudflare AI Gateway](/ja-JP/providers/cloudflare-ai-gateway)。
  </Accordion>
  <Accordion title="MiniMax">
    設定は自動的に書き込まれます。ホステッド環境のデフォルトは `MiniMax-M3` です。API キーのセットアップでは
    `minimax/...`、OAuth のセットアップでは `minimax-portal/...` を使用します。
    詳細: [MiniMax](/ja-JP/providers/minimax)。
  </Accordion>
  <Accordion title="StepFun">
    中国またはグローバルのエンドポイントに対して、StepFun Standard または Step Plan の設定が自動的に書き込まれます。
    Standard には現在 `step-3.5-flash` が含まれ、Step Plan には `step-3.5-flash-2603` も含まれます。
    詳細: [StepFun](/ja-JP/providers/stepfun)。
  </Accordion>
  <Accordion title="Synthetic（Anthropic 互換）">
    `SYNTHETIC_API_KEY` の入力を求めます。
    詳細: [Synthetic](/ja-JP/providers/synthetic)。
  </Accordion>
  <Accordion title="Ollama（クラウドおよびローカルのオープンモデル）">
    最初に `Cloud + Local`、`Cloud only`、または `Local only` の入力を求めます。
    `Cloud only` は、`https://ollama.com` とともに `OLLAMA_API_KEY` を使用します。
    ホストを使用するモードではベース URL（デフォルトは `http://127.0.0.1:11434`）の入力を求め、利用可能なモデルを検出して、デフォルト候補を提案します。
    `Cloud + Local` では、その Ollama ホストがクラウドアクセス用にサインイン済みかどうかも確認します。
    詳細: [Ollama](/ja-JP/providers/ollama)。
  </Accordion>
  <Accordion title="Moonshot と Kimi Coding">
    Moonshot（Kimi K2）と Kimi Coding の設定は自動的に書き込まれます。
    詳細: [Moonshot AI（Kimi + Kimi Coding）](/ja-JP/providers/moonshot)。
  </Accordion>
  <Accordion title="カスタムプロバイダー">
    OpenAI 互換、OpenAI Responses 互換、および Anthropic 互換のエンドポイントで動作します。

    対話型オンボーディングでは、他のプロバイダーの API キーフローと同じ API キー保存方法を利用できます。
    - **API キーを今すぐ貼り付ける**（平文）
    - **シークレット参照を使用する**（環境変数参照または設定済みプロバイダー参照。事前検証あり）

    オンボーディングは、一般的なビジョンモデル ID（GPT-4o/4.1/5.x、Claude 3/4、Gemini、Qwen-VL、LLaVA、Pixtral など）から画像対応を推測し、モデル名が不明な場合にのみ確認します。

    非対話型フラグ:
    - `--auth-choice custom-api-key`
    - `--custom-base-url`
    - `--custom-model-id`
    - `--custom-api-key`（任意。指定しない場合は `CUSTOM_API_KEY` にフォールバック）
    - `--custom-provider-id`（任意）
    - `--custom-compatibility <openai|openai-responses|anthropic>`（任意。デフォルトは `openai`）
    - `--custom-image-input` / `--custom-text-input`（任意。推測されたモデル入力機能を上書き）

  </Accordion>
  <Accordion title="スキップ">
    認証を未設定のままにします。
  </Accordion>
</AccordionGroup>

モデルの動作:

- 検出された選択肢からデフォルトモデルを選ぶか、プロバイダーとモデルを手動で入力します。
- プロバイダーの認証方法を選択してオンボーディングを開始した場合、モデル選択画面では
  そのプロバイダーが自動的に優先されます。Volcengine と BytePlus では、同じ優先設定が
  それぞれのコーディングプランのバリエーション（`volcengine-plan/*`、
  `byteplus-plan/*`）にも一致します。
- 優先プロバイダーで絞り込んだ結果が空になる場合、モデルを何も表示しないのではなく、
  完全なカタログにフォールバックします。
- ウィザードはモデルチェックを実行し、設定されたモデルが不明な場合や認証がない場合に警告します。

認証情報とプロファイルのパス:

- 認証プロファイル（API キー + OAuth）: `~/.openclaw/agents/<agentId>/agent/auth-profiles.json`
- レガシー OAuth のインポート: `~/.openclaw/credentials/oauth.json`

認証情報の保存モード:

- デフォルトのオンボーディングでは、API キーを認証プロファイルに平文の値として永続化します。
- `--secret-input-mode ref` を使用すると、平文でキーを保存する代わりに参照モードが有効になります。
  対話型セットアップでは、次のいずれかを選択できます。
  - 環境変数参照（例: `keyRef: { source: "env", provider: "default", id: "OPENAI_API_KEY" }`）
  - 設定済みプロバイダー参照（`file` または `exec`）。プロバイダーのエイリアス + ID を使用
- 対話型の参照モードでは、保存前に高速な事前検証を実行します。
  - 環境変数参照: 現在のオンボーディング環境で、変数名と値が空でないことを検証します。
  - プロバイダー参照: プロバイダー設定を検証し、要求された ID を解決します。
  - 事前検証に失敗すると、オンボーディングでエラーが表示され、再試行できます。
- 非対話型モードでは、`--secret-input-mode ref` は環境変数のみを参照します。
  - オンボーディングプロセスの環境にプロバイダーの環境変数を設定してください。
  - インラインキーフラグ（例: `--openai-api-key`）を使用するには、その環境変数を設定する必要があります。設定されていない場合、オンボーディングは即座に失敗します。
  - カスタムプロバイダーでは、非対話型の `ref` モードにより、`models.providers.<id>.apiKey` が `{ source: "env", provider: "default", id: "CUSTOM_API_KEY" }` として保存されます。
  - このカスタムプロバイダーの場合、`--custom-api-key` を使用するには `CUSTOM_API_KEY` を設定する必要があります。設定されていない場合、オンボーディングは即座に失敗します。
- Gateway の認証情報では、対話型セットアップで平文と SecretRef を選択できます。
  - トークンモード: **平文トークンを生成して保存**（デフォルト）または **SecretRef を使用**。
  - パスワードモード: 平文または SecretRef。
- 非対話型トークンの SecretRef パス: `--gateway-token-ref-env <ENV_VAR>`。
- 既存の平文セットアップは変更なく引き続き動作します。

<Note>
ヘッドレス環境とサーバー向けのヒント: ブラウザがあるマシンで OAuth を完了してから、そのエージェントの
`auth-profiles.json`（例:
`~/.openclaw/agents/<agentId>/agent/auth-profiles.json`、または対応する
`$OPENCLAW_STATE_DIR/...` のパス）を Gateway ホストへコピーします。`credentials/oauth.json`
はレガシーインポート元としてのみ使用されます。
</Note>

## 出力と内部動作

`~/.openclaw/openclaw.json` の一般的なフィールド:

- `agents.defaults.workspace`
- `--skip-bootstrap` が渡された場合は `agents.defaults.skipBootstrap`
- `agents.defaults.model` / `models.providers`（Minimax を選択した場合）
- `tools.profile`（未設定の場合、ローカルオンボーディングではデフォルトで `"coding"` になります。既存の明示的な値は保持されます）
- `gateway.*`（モード、バインド、認証、Tailscale）
- `session.dmScope`（オンボーディングでは明示的な値を保持し、それ以外の場合は未設定のままにします。そのため、`main` のデフォルトにより、すべてのチャネルのダイレクトメッセージがエージェントのローリングメインセッションに保持されます。これは個人用エージェントのデフォルトです。共有またはマルチユーザーの受信トレイでは `per-channel-peer` を使用してください。`openclaw security audit` は、マルチユーザーの DM トラフィックを検出すると分離を推奨します）
- `channels.telegram.botToken`、`channels.discord.token`、`channels.matrix.*`、`channels.signal.*`、`channels.imessage.*`
- プロンプト中にオプトインした場合のチャネル許可リスト（Discord、iMessage、Signal、Slack、Telegram、WhatsApp）。Discord と Slack では入力した名前も ID に解決されます
- `skills.install.nodeManager`
  - `setup --node-manager` フラグは `npm`、`pnpm`、または `bun` を受け付けます。
  - 後から手動設定で `skills.install.nodeManager: "yarn"` を指定することもできます。
- `wizard.lastRunAt`
- `wizard.lastRunVersion`
- `wizard.lastRunCommit`
- `wizard.lastRunCommand`
- `wizard.lastRunMode`
- `wizard.securityAcknowledgedAt`

`openclaw agents add` は `agents.entries.*` と、任意の `bindings` を書き込みます。

WhatsApp の認証情報は `~/.openclaw/credentials/whatsapp/<accountId>/` に保存されます。
アクティブなセッションとトランスクリプトは
`~/.openclaw/agents/<agentId>/agent/openclaw-agent.sqlite` に保存されます。
`~/.openclaw/agents/<agentId>/sessions/` ディレクトリは、レガシー移行の
入力およびアーカイブやサポート用の成果物に使用されます。

<Note>
一部のチャネルは Plugin として提供されます。セットアップ中に選択すると、ウィザードは
チャネルを設定する前に Plugin（npm またはローカルパス）のインストールを求めます。
</Note>

### インストール済みアプリの推奨

モデルアクセスのチェックに成功すると、macOS の従来の対話型オンボーディングは、macOS のプライバシー権限を要求せずにアプリケーション名とバンドル ID をスキャンします。公式 Plugin カタログと ClawHub を検索し、設定済みモデルに、名前が誤って一致した候補を除外して関連する Plugin または Skills を推奨するよう求めます。推奨された一致候補はデフォルトで選択されます。任意の一致候補は明示的に選択する必要があります。

結果画面には検出されたアプリケーションが一覧表示され、「アプリ名は、設定済みモデルと ClawHub 検索を使用して照合されました。」と表示されます。`wizard.appRecommendations` を `false` に設定すると、このオンボーディング手順と、Gateway による Node アプリインベントリへのアクセスの両方が無効になります。このスキャンは、クイックスタートまたは macOS 以外のオンボーディングでは使用されません。

## 非対話型セットアップ

`--non-interactive` には `--accept-risk` が必要です（エージェントは強力であり、システム全体へのアクセスにはリスクがあることを確認します）。

```bash
openclaw onboard --non-interactive --accept-risk \
  --auth-choice apiKey \
  --anthropic-api-key "$ANTHROPIC_API_KEY"
```

フラグの完全なリファレンスとプロバイダー固有の例: [`openclaw onboard`](/ja-JP/cli/onboard)、[CLI 自動化](/ja-JP/start/wizard-cli-automation)。

## Gateway ウィザード RPC

- `wizard.start`
- `wizard.next`
- `wizard.cancel`
- `wizard.status`

クライアント（macOS アプリと Control UI）は、オンボーディングのロジックを再実装せずに手順を表示できます。

## Signal のセットアップ動作

- 公式の `signal-cli` GitHub リリースから適切なリリースアセットをダウンロードします（ネイティブビルド、Linux x86-64 のみ）
- その他のプラットフォーム（macOS、x64 以外の Linux）では、代わりに Homebrew を使用してインストールします
- リリースアセットによるインストールを `~/.openclaw/tools/signal-cli/<version>/` に保存します
- 設定に `kind: "managed-native"` を含む `channels.signal.transport.cliPath` を書き込みます
- ネイティブ Windows はまだサポートされていません。Linux のインストールパスを使用するには WSL2 内でオンボーディングを実行してください

## 関連ドキュメント

- オンボーディングハブ: [オンボーディング（CLI）](/ja-JP/start/wizard)
- 自動化とスクリプト: [CLI 自動化](/ja-JP/start/wizard-cli-automation)
- コマンドリファレンス: [`openclaw onboard`](/ja-JP/cli/onboard)
