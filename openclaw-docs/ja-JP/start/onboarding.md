---
read_when:
    - macOS オンボーディングアシスタントの設計
    - 認証またはアイデンティティ設定の実装
sidebarTitle: 'Onboarding: macOS App'
summary: OpenClaw（macOSアプリ）の初回セットアップフロー
title: オンボーディング（macOS アプリ）
x-i18n:
    generated_at: "2026-07-26T10:21:11Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 55154774886c530de92b2110d367af24e2142fac48b901f288582d8552a6ca10
    source_path: start/onboarding.md
    workflow: 16
---

macOS アプリの初回実行フローでは、Gateway の実行場所を選択し、検証済みの AI バックエンドに接続し、権限を付与して、エージェント独自のブートストラップ手順に引き継ぎます。
CLI オンボーディングと両方の経路の比較については、[オンボーディングの概要](/ja-JP/start/onboarding-overview)を参照してください。

<Steps>
<Step title="macOS の警告を承認">
<Frame>
<img src="/assets/macos-onboarding/01-macos-warning.jpeg" alt="" />
</Frame>
</Step>
<Step title="ローカルネットワークの検索を承認">
<Frame>
<img src="/assets/macos-onboarding/02-local-networks.jpeg" alt="" />
</Frame>
</Step>
<Step title="ようこそ画面とセキュリティに関する注意事項">
<Frame caption="表示されたセキュリティに関する注意事項を読み、それに応じて判断してください">
<img src="/assets/macos-onboarding/03-security-notice.png" alt="" />
</Frame>

セキュリティの信頼モデル：

- デフォルトでは、OpenClaw は個人用エージェントであり、信頼されるオペレーターの境界は 1 つです。
- 共有またはマルチユーザー構成では、厳格な制限が必要です。信頼境界を分離し、ツールへのアクセスを最小限に抑え、[セキュリティ](/ja-JP/gateway/security)の指針に従ってください。
- ローカルのオンボーディングでは、新しい設定のデフォルトを `tools.profile: "coding"` にするため、新規構成でも、制限のない `full` プロファイルを使用せずにファイルシステムおよびランタイムツールを維持できます。
- フック、Webhook、またはその他の信頼されていないコンテンツフィードを有効にする場合は、性能の高い最新のモデル層を使用し、厳格なツールポリシーとサンドボックス化を維持してください。

</Step>
<Step title="ローカルとリモート">
<Frame>
<img src="/assets/macos-onboarding/04-choose-gateway.png" alt="" />
</Frame>

**Gateway** はどこで実行しますか？

- **この Mac（ローカルのみ）：** オンボーディングが認証を設定し、認証情報をローカルに書き込みます。
- **リモート（SSH/Tailnet 経由）：** オンボーディングはローカル認証を設定しません。
  認証情報は Gateway ホストにすでに存在している必要があります。リモート Gateway のトークン
  フィールドには、macOS アプリがその Gateway への接続に使用するトークンが保存されます。
  既存の `gateway.remote.token` SecretRef 値は、置き換えるまで
  維持されます。
- **後で設定：** セットアップをスキップし、アプリを未設定のままにします。

<Tip>
**Gateway 認証のヒント：**

- Gateway の認証モードは、ループバックへのバインドでもデフォルトで `token` になるため、ローカルの WS クライアントも認証する必要があります。
- `gateway.auth.mode: "none"` を設定すると、任意のローカルプロセスが接続できるようになります。完全に信頼できるマシンでのみ使用してください。
- 複数マシンからのアクセスまたは非ループバックへのバインドには、トークンを使用してください。

</Tip>
</Step>
<Step title="CLI">
  ローカルセットアップでは、npm、pnpm、または bun を使用してグローバルの `openclaw` CLI をインストールし、
  npm を最優先します。Gateway 自体の推奨ランタイムは引き続き Node です。
  既存の互換性のあるインストールは再利用されます。
</Step>
<Step title="AI に接続">
  エージェントモデルがすでに設定されている Gateway に接続した場合、この
  ページは完全にスキップされ、通常のエージェント UI が開きます。OpenClaw とプロバイダーのセットアップは、
  新規または未設定項目がある Gateway に対してのみ実行されます。

Gateway の準備ができると、オンボーディングは既存の AI アクセスを検索します。
対象は Claude Code または Codex のログイン、`OPENAI_API_KEY` / `ANTHROPIC_API_KEY`、あるいは
到達可能な Ollama または LM Studio サーバーにすでにインストールされている、
測定済みの有効コンテキストが 16K 以上のツール対応モデルです。検出は
Gateway ホスト上で実行されます。macOS アプリから Linux の Gateway に接続する場合も同様です。最適な
選択肢は実際の補完でテストされ、応答した場合にのみ
保存されます。テストが失敗すると、アプリは自動的に次の選択肢を試し、
前の選択肢が失敗した理由を表示します。複数の選択肢が見つかった場合は、
続行する前に切り替えられます。ローカルでの自動検出によってモデルが取得または
ダウンロードされることはありません。

Gateway ホストに Claude CLI のログインがない状態で Claude サブスクリプションを使用するには、
Claude Code がインストールされている任意のマシンで
`claude setup-token` を実行し、表示されたトークンを **Connect with an API key or
token** の **Anthropic setup-token** に貼り付けます。

インストール済みの Gemini CLI、Antigravity、Pi、OpenCode CLI は、
再利用可能なガイド付きセットアップの推論経路として選択できない場合でも、参考情報として表示されます。
Gemini と Antigravity では、ツールを使用しない推論プローブを強制できません。Pi と
OpenCode はセットアップ用の推論経路ではなく、エージェント全体のハーネスです。これらの
セッション統合には、ランタイムと Plugin を個別に設定する必要があります。

プロバイダー独自の OAuth またはデバイスペアリングフローを使用してサインインすることもできます。
組み込みの選択肢には、OpenAI/ChatGPT、OpenRouter、GitHub Copilot、Google
Gemini CLI、xAI、MiniMax Global および CN、Chutes が含まれます。この一覧は
固定されたアプリの一覧ではなく、Gateway で有効なテキスト推論プロバイダー Plugin から取得されるため、
プロバイダー固有の macOS コードを追加しなくても、別のプロバイダーが参加できます。

手動のキーまたはトークン選択でも、同じプロバイダーレジストリが使用されます。どの経路でも、
プロバイダーが初期モデルと設定を提供し、OpenClaw は
認証プロファイルを保存する前に、同じライブテストで認証情報を検証します。1 つのバックエンドが合格するまで
「次へ」はロックされたままになるため、推論が機能しなければ最初のエージェントチャットを
開始できません。そのライブチェックに合格すると、OpenClaw を使用して
残りのワークスペース、Gateway、チャンネル、その他の任意機能を設定できるようになります。OpenClaw が
短い選択肢一覧を提示した場合、アプリにはネイティブの選択肢カードが表示されます。いずれかを選ぶと
その選択内容が送信され、**Skip for
now** を選べば、その選択は常に任意のままです。OpenClaw は後から
Settings → OpenClaw でも利用できます。
</Step>
<Step title="メモリをインポート（検出時に表示）">
ローカル Gateway の場合、オンボーディングは対応する AI
ツールのメモリが Mac 上にあるか確認します。対象は Claude Code の自動メモリ、Codex の統合済みメモリ、Hermes のメモリ
ファイルです。いずれかが見つかると、このページには各ソースとそのメモリ数が表示され、
選択したソースをエージェントワークスペースの
`memory/imports/` にインポートし、インデックス化された検索に利用できます。インポート済みのファイルはスキップされ、
インポート対象がない場合、このページは表示されません。スキップしても問題ありません。
ダッシュボードのメモリインポートページから、後でファイル単位の
制御を使用して同じインポートを実行できます。
</Step>
<Step title="権限">

<Frame caption="OpenClaw に付与する権限を選択してください">
<img src="/assets/macos-onboarding/05-permissions.png" alt="" />
</Frame>

オンボーディングでは、オートメーション（AppleScript）、通知、アクセシビリティ、画面収録、マイク、音声認識、カメラ、位置情報に対する TCC 権限を要求します。

</Step>
<Step title="完了">
  推論がテストに合格すると、残りの任意セットアップは OpenClaw が担当し、
  通常のエージェントチャットに引き継ぐことができます。権限の手順を完了すると
  同じチャットが開きます。アプリが OpenClaw より前にワークスペースを作成したり、別の
  エージェントセットアップ会話を開始したりすることはありません。エージェントによる最初の実際のターン中に
  Gateway ホストで何が起こるかについては、
  [ブートストラップ](/ja-JP/start/bootstrapping)を参照してください。
</Step>
</Steps>

## 関連項目

- [オンボーディングの概要](/ja-JP/start/onboarding-overview)
- [はじめに](/ja-JP/start/getting-started)
