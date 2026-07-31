---
read_when:
    - CLI オンボーディングの実行または設定
    - 新しいマシンのセットアップ
sidebarTitle: 'Onboarding: CLI'
summary: CLI オンボーディング：推論を検証してから、残りのセットアップを OpenClaw に引き継ぐ
title: オンボーディング（CLI）
x-i18n:
    generated_at: "2026-07-26T10:21:36Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 150adfac1424b42d66fa3035339082574cc631ce0dc3db09ad32376ef139bf1c
    source_path: start/wizard.md
    workflow: 16
---

```bash
openclaw onboard
```

CLI オンボーディングは、macOS、Linux、および Windows（ネイティブまたは WSL2）で推奨されるターミナルセットアップ手順です。デフォルトでは、マシンですでに利用可能な AI アクセスを検出し、実際の補完で検証してから OpenClaw を起動し、ワークスペース、Gateway、およびオプション機能を設定します。`openclaw setup` は同じフローを実行します（[セットアップ](/ja-JP/cli/setup)では、設定のみを行う `--baseline` バリアントについて説明しています）。Windows デスクトップユーザーは [Windows Hub](/ja-JP/platforms/windows) から開始することもできます。

ガイド付きオンボーディングでは、最初に推論を確立します。利用可能な AI アクセスを検出し、実際の補完を必須とし、その後にのみ [OpenClaw](/ja-JP/cli/openclaw) を起動して OpenClaw の残りの部分を設定します。**Skip for now** を選択すると、OpenClaw を起動せずにオンボーディングを終了します。

カスタムプロバイダー、リモート Gateway のセットアップ、チャンネルのペアリング、デーモン制御、Skills、およびインポートには、従来のウィザードを引き続き利用できます。`openclaw onboard --classic` で明示的に実行してください。ガイド付き推論選択機能からこのウィザードに処理が委譲されることはありません。推論に合格すると、OpenClaw は `open channel wizard for
<channel>` を使用して、シークレットを必要とするチャンネル設定を、入力をマスクするターミナルウィザードに引き渡せます。
モデルプロバイダーまたはその認証を変更するには、OpenClaw を終了して `openclaw onboard` を実行してください。OpenClaw からガイド付きまたは従来のプロバイダーフローが開かれることはありません。

<Info>
最速で最初のチャットを始めるには、ガイド付きセットアップを完了し、`openclaw dashboard` を実行して、Control UI を介してブラウザーでチャットします。ドキュメント：[ダッシュボード](/ja-JP/web/dashboard)。
</Info>

## ロケール

ウィザードは、固定されたオンボーディング文言をローカライズします。`OPENCLAW_LOCALE`、`LC_ALL`、`LC_MESSAGES`、`LANG` の順に最初の空でない値を使用し、その後
英語にフォールバックします。サポートされるロケール：`en`、`zh-CN`、`zh-TW`。

```bash
OPENCLAW_LOCALE=zh-CN openclaw onboard
OPENCLAW_LOCALE=en openclaw onboard # 明示的に英語へ上書き
```

製品名、コマンド、設定キー、URL、プロバイダー ID、モデル ID、および
プラグイン／チャンネルのラベルは、ロケールに関係なく英語のままです。

推論以外の設定を後から再構成するには：

```bash
openclaw configure
openclaw agents add <name>
```

<Note>
`--json` は非対話モードを意味しません。スクリプトでは `--non-interactive` を使用してください（[CLI 自動化](/ja-JP/start/wizard-cli-automation)を参照）。
</Note>

<Tip>
従来のウィザードには、プロバイダーを選択できる Web 検索手順があります：Brave、
DuckDuckGo、Exa、Firecrawl、Gemini、Grok、Kimi、MiniMax Search、Ollama Web
Search、Perplexity、SearXNG、または Tavily。一部には API キーが必要ですが、キーなしで利用できるものもあります。後から `openclaw configure --section web` で設定できます。ドキュメント：
[Web ツール](/ja-JP/tools/web)。
</Tip>

## ガイド付きのデフォルト

通常の `openclaw onboard` は次の手順に従います：

1. セキュリティ通知に同意します。
2. 設定済みモデル、API キーの環境変数、サポートされているローカル AI
   CLI、および Gateway ホストから到達可能な Ollama または LM
   Studio サーバーにすでにインストールされている、ツール対応モデルを検出します。この読み取り専用の処理ではモデルをダウンロードしません。Gemini CLI、Antigravity、Pi、および OpenCode のインストールについても、ガイド付きセットアップで再利用可能な推論経路として使用できない場合に報告されます。
   Gemini と Antigravity ではツールなしのプローブを強制できません。Pi と OpenCode はセットアップ用の推論経路ではなく、エージェント全体のハーネスです。
3. 最初に検出された候補を実際の補完でテストします。失敗した場合は理由を表示し、次の使用可能な候補に進みます。
4. 検出候補をすべて試しても見つからない場合は、OpenAI、Anthropic、xAI（Grok）、Google、または
   OpenRouter を選択するか、残りのプロバイダーについては **More…** を選択します。各プロバイダーの
   リージョン、プラン、およびサポートされているブラウザー、デバイス、API キー、またはトークン方式が
   2 番目のメニューに表示され、同じ実際の補完でテストされます。
   OpenClaw を起動せずに終了するには **Skip for now** を選択します。
5. 検証済みのモデル経路と、それに必要な認証情報／プラグインの状態のみを永続化します。ワークスペースと Gateway の設定は変更されません。
6. 検証済みモデルで OpenClaw を起動し、ワークスペース、
   Gateway、チャンネル、エージェント、プラグイン、および残りのオプション設定を構成できるようにします。

設定済みのインストール環境でコマンドを再実行すると、現在のデフォルトモデルが最初にテストされるため、ガイド付きフローは検証と修復の処理として機能します。チェックに失敗しても、設定済みモデルが自動的に置き換えられることはありません。オンボーディングは停止し、続行方法を確認します。推論以外の項目を後から追加するには `openclaw channels add` または `openclaw configure` を実行し、プロバイダーまたは認証経路の変更には `openclaw onboard` を使用します。

## 従来のウィザード：QuickStart と Advanced

完全なウィザードを開くには `openclaw onboard --classic` を実行します。最初に
**QuickStart**（デフォルト）と **Advanced**（完全な制御）のいずれかを選択します。`--flow quickstart` または `--flow advanced`（別名 `manual`）を渡すと、従来のフローを選択してこのプロンプトをスキップできます。

<Tabs>
  <Tab title="QuickStart（デフォルト）">
    - ローカル Gateway、ループバックバインド
    - デフォルトのワークスペース（または既存のワークスペース）
    - Gateway ポート **18789**
    - Gateway 認証 **Token**（ループバックでも自動生成）
    - ツールポリシー：新規セットアップでは `tools.profile: "coding"`（既存の明示的なプロファイルは維持されます）
    - DM セッション：オンボーディングでは明示的な `session.dmScope` を維持し、それ以外の場合は未設定のままにします。そのため、`"main"` のデフォルトにより、すべてのチャンネルからのダイレクトメッセージがエージェントの継続的なメインセッションに保持されます。これは個人用エージェントのデフォルトです。共有またはマルチユーザーの受信トレイでは `"per-channel-peer"` を使用してください。`openclaw security audit` は、マルチユーザーの DM トラフィックを検出すると分離を推奨します。詳細：[CLI セットアップリファレンス](/ja-JP/start/wizard-cli-reference#outputs-and-internals)
    - Tailscale 公開 **Off**
    - Telegram と WhatsApp の DM のデフォルトは **allowlist**：Telegram では数値の Telegram ユーザー ID、WhatsApp では電話番号の入力を求めます

  </Tab>
  <Tab title="Advanced（完全な制御）">
    - モード、ワークスペース、Gateway、チャンネル、デーモン、Skills のすべての手順を表示します

  </Tab>
</Tabs>

リモートモード（`--mode remote`）では常に高度なフローを使用します。このマシンを別の場所にある Gateway に接続するよう設定するだけで、リモートホストへのインストールや変更は一切行いません。

## 従来のオンボーディングで設定される内容

ローカルモード（デフォルト）では、次の手順を進めます：

1. **モデル／認証** - プロバイダーの認証フロー（API キー、OAuth、または
   プロバイダー固有の手動認証）を選択します。これにはカスタムプロバイダー
   （OpenAI 互換、OpenAI Responses 互換、Anthropic 互換、または
   不明な方式の自動検出）が含まれます。デフォルトモデルを選択します。
   新規の OpenAI API キーセットアップでは、デフォルトで `openai/gpt-5.6` が使用されます（修飾子のない直接 API
   ID は Sol に解決されます）。新規の ChatGPT/Codex セットアップでは、デフォルトで
   `openai/gpt-5.6-sol` が使用されます。セットアップを再実行すると、`openai/gpt-5.5` を含む既存の明示的なモデルが維持されます。アカウントで GPT-5.6 が提供されていない場合は、`openai/gpt-5.5` を明示的に選択してください。
   セキュリティ上の注意：このエージェントがツールを実行する、または Webhook／フックの内容を処理する場合は、利用可能な最新世代の最も強力なモデルを優先し、ツールポリシーを厳格に維持してください。性能の低い、または古い世代のモデルほど、プロンプトインジェクションを受けやすくなります。
   非対話実行では、`--secret-input-mode ref` はプレーンテキストの API キー値ではなく、環境変数に基づく参照を保存します。参照先の環境変数は事前に設定されている必要があり、設定されていない場合はオンボーディングが即座に失敗します。対話型のシークレット参照モードでは、環境変数または設定済みのプロバイダー参照（`file` または
   `exec`）を指定でき、保存前に簡易事前チェックが行われます。モデル／認証のセットアップ後、ウィザードではオプションのライブ補完テストが提示されます。失敗した場合はモデル／認証のセットアップに一度戻るか、従来のウィザードの残りを妨げずに無視できます。無視しても OpenClaw のロックは解除されません。対話型セットアップには、引き続き推論チェックへの合格が必要です。
2. **ワークスペース** - エージェントファイル用のディレクトリ（デフォルトは `~/.openclaw/workspace`）。ブートストラップファイルを生成します。
3. **Gateway** - ポート、バインドアドレス、認証モード、Tailscale 公開。対話型トークンモードでは、プレーンテキストでのトークン保存（デフォルト）を選択するか、SecretRef の使用を選択します。非対話型 SecretRef のパス：`--gateway-token-ref-env <ENV_VAR>`。
4. **チャンネル** - 組み込みおよび公式プラグインのチャットチャンネル。Discord、Feishu、Google Chat、iMessage、Mattermost、Microsoft Teams、
   QQ Bot、Signal、Slack、Telegram、WhatsApp などが含まれます。
5. **デーモン** - LaunchAgent（macOS）、systemd ユーザーユニット
   （Linux/WSL2）、またはユーザーごとの Startup フォルダーへのフォールバックを備えたネイティブ Windows Scheduled Task をインストールします。
   トークン認証が必要で、`gateway.auth.token` が SecretRef で管理されている場合、デーモンのインストールではそれを検証しますが、解決済みトークンをスーパーバイザーサービスの環境メタデータには永続化しません。未解決の SecretRef がある場合は、ガイダンスを表示してインストールをブロックします。`gateway.auth.mode` が未設定の状態で `gateway.auth.token` と
   `gateway.auth.password` の両方が設定されている場合は、モードを明示的に設定するまでインストールがブロックされます。
6. **ヘルスチェック** - Gateway を起動し、到達可能であることを検証します。
7. **Skills** - 推奨される Skills と、そのオプション依存関係をインストールします。

<Note>
オンボーディングを再実行しても、**Reset** を明示的に選択する（または `--reset` を渡す）場合を除き、何も消去されません。CLI の `--reset` は、デフォルトで設定、認証情報、およびセッションを対象とします。ワークスペースも削除するには `--reset-scope full` を使用してください。設定が無効であるか、従来のキーが含まれている場合、オンボーディングでは最初に `openclaw doctor` を実行するよう求められます。
</Note>

`--flow import` は、新規セットアップの代わりに、検出された移行フロー（Hermes など）を従来のウィザード内で実行します。[移行](/ja-JP/cli/migrate)および
[インストール](/ja-JP/install/migrating-hermes)配下の移行ガイドを参照してください。`openclaw onboard --modern` は
[OpenClaw](/ja-JP/cli/openclaw) の互換性エイリアスです。`openclaw setup` と同じ推論ゲートを使用します。推論が検証されるとアシスタントが起動し、対話型の検証に失敗するとガイド付き推論セットアップに戻ります。

## 別のエージェントを追加する

`openclaw agents add <name>` を使用すると、独自の
ワークスペース、セッション、および認証プロファイルを持つ別個のエージェントを作成できます。`--workspace` なしで実行すると、名前、ワークスペース、認証、チャンネル、およびバインディングの対話型フローが開始されます。これは完全な `openclaw onboard` ウィザードではありません。

設定される内容：

- `agents.entries.*.name`
- `agents.entries.*.workspace`
- `agents.entries.*.agentDir`

注意事項：

- デフォルトのワークスペース：`~/.openclaw/workspace-<agentId>`（`agents.defaults.workspace` が設定されている場合はその配下）。
- 受信メッセージをこのエージェントにルーティングするには `bindings` を追加します（オンボーディングで設定することもできます）。
- 非対話型フラグ：`--model`、`--agent-dir`、`--bind`、`--non-interactive`。

## 完全なリファレンス

手順ごとの詳細な動作と設定出力については、
[CLI セットアップリファレンス](/ja-JP/start/wizard-cli-reference)を参照してください。
非対話型の例については、[CLI 自動化](/ja-JP/start/wizard-cli-automation)を参照してください。
すべてのフラグのリファレンスについては、[`openclaw onboard`](/ja-JP/cli/onboard)を参照してください。

## 関連ドキュメント

- CLI コマンドリファレンス：[`openclaw onboard`](/ja-JP/cli/onboard)
- オンボーディングの概要：[オンボーディングの概要](/ja-JP/start/onboarding-overview)
- macOS アプリのオンボーディング：[オンボーディング](/ja-JP/start/onboarding)
- エージェントの初回実行手順：[エージェントのブートストラップ](/ja-JP/start/bootstrapping)
