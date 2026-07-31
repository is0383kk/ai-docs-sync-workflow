---
read_when:
    - Plugin をコア npm パッケージに同梱するか、個別にインストールするかを決定しています
    - バンドル済み Plugin のパッケージメタデータまたはリリース自動化を更新している場合
    - 正規の内部 Plugin と外部 Plugin の一覧が必要です
summary: コアに同梱、外部公開、またはソースのみで保持されている OpenClaw plugins の生成済み一覧
title: Plugin インベントリ
x-i18n:
    generated_at: "2026-07-26T09:11:33Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 2d835087afbe9d75f883c3db9739f914bedab5ac87a9c20b69c248304b61c594
    source_path: plugins/plugin-inventory.md
    workflow: 16
---

# Plugin インベントリ

このページは `extensions/*/package.json`、`openclaw.plugin.json`、
およびルート npm パッケージの `files` 除外設定から生成されています。次のコマンドで再生成します。

```bash
pnpm plugins:inventory:gen
```

## 定義

- **コア npm パッケージ:** `openclaw` npm パッケージに組み込まれており、Plugin を個別にインストールせずに利用できます。
- **公式外部パッケージ:** OpenClaw が保守する Plugin で、コア npm パッケージには含まれませんが、この公式インベントリに掲載され、必要に応じて ClawHub や npm を通じてインストールされます。
- **ソースチェックアウト限定:** 公開 npm アーティファクトには含まれず、インストール可能なパッケージとして案内されないリポジトリローカルの Plugin です。

ソースチェックアウトは npm インストールとは異なります。`pnpm install` の後、
バンドルされた Plugin は `extensions/<id>` から読み込まれるため、ローカルでの編集とパッケージローカルなワークスペース
依存関係を利用できます。

## Plugin のインストール

各項目のインストール経路を参照して、インストールが必要かどうかを判断します。
`included in OpenClaw` と記載された Plugin は、すでにコアパッケージに含まれています。
公式外部パッケージは一度インストールした後、Gateway を再起動する必要があります。

たとえば、Discord は公式外部パッケージです。

```bash
openclaw plugins install @openclaw/discord
openclaw gateway restart
openclaw plugins inspect discord --runtime --json
```

リリース移行期間中は、通常のベアパッケージ指定でも引き続き npm からインストールされます。
ソースを明示する必要がある場合は、`clawhub:@openclaw/discord` または `npm:@openclaw/discord` を使用します。
インストール後、認証情報とチャンネル設定を追加するには、[Discord](/ja-JP/channels/discord) などの
Plugin のセットアップドキュメントに従ってください。更新、アンインストール、公開の
コマンドについては、[Plugin の管理](/ja-JP/plugins/manage-plugins)を参照してください。

各項目には、パッケージ、配布経路、説明が記載されています。

## コア npm パッケージ

70 個の Plugin

- **[admin-http-rpc](/ja-JP/plugins/reference/admin-http-rpc)** (`@openclaw/admin-http-rpc`) - OpenClaw に含まれています。OpenClaw 管理用 HTTP RPC エンドポイントです。

- **[alibaba](/ja-JP/plugins/reference/alibaba)** (`@openclaw/alibaba-provider`) - OpenClaw に含まれています。動画生成プロバイダーのサポートを追加します。

- **[anthropic](/ja-JP/plugins/reference/anthropic)** (`@openclaw/anthropic-provider`) - OpenClaw に含まれています。Anthropic モデル、Claude CLI、ネイティブ Claude セッションカタログを提供します。

- **[azure-speech](/ja-JP/plugins/reference/azure-speech)** (`@openclaw/azure-speech`) - OpenClaw に含まれています。Azure AI Speech によるテキスト読み上げ（MP3、ネイティブ Ogg/Opus ボイスメモ、PCM テレフォニー）を提供します。

- **[bonjour](/ja-JP/plugins/reference/bonjour)** (`@openclaw/bonjour`) - OpenClaw に含まれています。ローカルの OpenClaw Gateway を Bonjour/mDNS 経由で公開します。

- **[browser](/ja-JP/plugins/reference/browser)** (`@openclaw/browser-plugin`) - OpenClaw に含まれています。エージェントから呼び出し可能なツールを追加します。

- **[byteplus](/ja-JP/plugins/reference/byteplus)** (`@openclaw/byteplus-provider`) - OpenClaw に含まれています。BytePlus および BytePlus Plan モデルプロバイダーのサポートを OpenClaw に追加します。

- **[canvas](/ja-JP/plugins/reference/canvas)** (`@openclaw/canvas-plugin`) - OpenClaw に含まれています。ペアリングされた Node 向けの実験的な Canvas 制御と A2UI レンダリングサーフェスを提供します。

- **[clawrouter](/ja-JP/plugins/reference/clawrouter)** (`@openclaw/clawrouter`) - OpenClaw に含まれています。ClawRouter モデルプロバイダーのサポートを OpenClaw に追加します。

- **[cohere](/ja-JP/plugins/reference/cohere)** (`@openclaw/cohere-provider`) - OpenClaw に含まれています。npm、ClawHub: `clawhub:@openclaw/cohere-provider`。OpenClaw の Cohere プロバイダー Plugin です。

- **[comfy](/ja-JP/plugins/reference/comfy)** (`@openclaw/comfy-provider`) - OpenClaw に含まれています。ComfyUI モデルプロバイダーのサポートを OpenClaw に追加します。

- **[copilot-proxy](/ja-JP/plugins/reference/copilot-proxy)** (`@openclaw/copilot-proxy`) - OpenClaw に含まれています。Copilot Proxy モデルプロバイダーのサポートを OpenClaw に追加します。

- **[crabbox](/ja-JP/plugins/reference/crabbox)** (`@openclaw/crabbox-provider`) - OpenClaw に含まれています。Crabbox CLI を基盤とするクラウドワーカープロバイダーです。

- **[cua-computer](/ja-JP/plugins/reference/cua-computer)** (`@openclaw/cua-computer`) - OpenClaw に含まれています。Windows および Linux Node ホスト向けの実験的な cua-driver コンピューター制御を提供します。

- **[deepgram](/ja-JP/plugins/reference/deepgram)** (`@openclaw/deepgram-provider`) - OpenClaw に含まれています。メディア理解プロバイダーのサポートを追加します。リアルタイム文字起こしプロバイダーのサポートを追加します。

- **[document-extract](/ja-JP/plugins/reference/document-extract)** (`@openclaw/document-extract-plugin`) - OpenClaw に含まれています。ローカルの文書添付ファイルからテキストと、フォールバック用のページ画像を抽出します。

- **[duckduckgo](/ja-JP/plugins/reference/duckduckgo)** (`@openclaw/duckduckgo-plugin`) - OpenClaw に含まれています。ウェブ検索プロバイダーのサポートを追加します。

- **[elevenlabs](/ja-JP/plugins/reference/elevenlabs)** (`@openclaw/elevenlabs-speech`) - OpenClaw に含まれています。メディア理解プロバイダーのサポートを追加します。リアルタイム文字起こしプロバイダーのサポートを追加します。テキスト読み上げプロバイダーのサポートを追加します。

- **[fal](/ja-JP/plugins/reference/fal)** (`@openclaw/fal-provider`) - OpenClaw に含まれています。fal モデルプロバイダーのサポートを OpenClaw に追加します。

- **[file-transfer](/ja-JP/plugins/reference/file-transfer)** (`@openclaw/file-transfer`) - OpenClaw に含まれています。専用の Node コマンドを介して、ペアリングされた Node 上のファイルを取得、一覧表示、書き込みします。最大 16 MB のバイナリに対して node.invoke 経由で base64 を使用することで、bash の標準出力切り捨てを回避します。

- **[github-copilot](/ja-JP/plugins/reference/github-copilot)** (`@openclaw/github-copilot-provider`) - OpenClaw に含まれています。GitHub Copilot モデルプロバイダーのサポートを OpenClaw に追加します。

- **[google](/ja-JP/plugins/reference/google)** (`@openclaw/google-plugin`) - OpenClaw に含まれています。Google、Google Gemini CLI、Google Vertex モデルプロバイダーのサポートを OpenClaw に追加します。

- **[huggingface](/ja-JP/plugins/reference/huggingface)** (`@openclaw/huggingface-provider`) - OpenClaw に含まれています。Hugging Face モデルプロバイダーのサポートを OpenClaw に追加します。

- **[imessage](/ja-JP/plugins/reference/imessage)** (`@openclaw/imessage`) - OpenClaw に含まれています。OpenClaw メッセージを送受信するための iMessage チャンネルサーフェスを追加します。

- **[linux-canvas](/ja-JP/plugins/reference/linux-canvas)** (`@openclaw/linux-canvas`) - OpenClaw に含まれています。OpenClaw Linux デスクトップアプリ向けの Canvas レンダリングブリッジです。

- **[linux-node](/ja-JP/plugins/reference/linux-node)** (`@openclaw/linux-node`) - OpenClaw に含まれています。Linux Node ホスト向けにデスクトップ通知、カメラ撮影、位置情報を提供します。

- **[litellm](/ja-JP/plugins/reference/litellm)** (`@openclaw/litellm-provider`) - OpenClaw に含まれています。LiteLLM モデルプロバイダーのサポートを OpenClaw に追加します。

- **[llm-task](/ja-JP/plugins/reference/llm-task)** (`@openclaw/llm-task`) - OpenClaw に含まれています。ワークフローから呼び出し可能な構造化タスク向けの、JSON のみを扱う汎用 LLM ツールです。

- **[lmstudio](/ja-JP/plugins/reference/lmstudio)** (`@openclaw/lmstudio-provider`) - OpenClaw に含まれています。LM Studio モデルプロバイダーのサポートを OpenClaw に追加します。

- **[logbook](/ja-JP/plugins/reference/logbook)** (`@openclaw/logbook`) - OpenClaw に含まれています。自動作業日誌です。ペアリングされた Node から定期的に画面のスナップショットを取得し、1 日の活動を確認可能なタイムラインに変換します。

- **[memory-core](/ja-JP/plugins/reference/memory-core)** (`@openclaw/memory-core`) - OpenClaw に含まれています。エージェントから呼び出し可能なツールを追加します。

- **[memory-wiki](/ja-JP/plugins/reference/memory-wiki)** (`@openclaw/memory-wiki`) - OpenClaw に含まれています。OpenClaw 向けの永続 Wiki コンパイラーおよび Obsidian 対応ナレッジ保管庫です。

- **[meta](/ja-JP/plugins/reference/meta)** (`@openclaw/meta-provider`) - OpenClaw に含まれています。npm、ClawHub: `clawhub:@openclaw/meta-provider`。Meta モデルプロバイダーのサポートを OpenClaw に追加します。

- **[microsoft](/ja-JP/plugins/reference/microsoft)** (`@openclaw/microsoft-speech`) - OpenClaw に含まれています。テキスト読み上げプロバイダーのサポートを追加します。

- **[microsoft-foundry](/ja-JP/plugins/reference/microsoft-foundry)** (`@openclaw/microsoft-foundry`) - OpenClaw に含まれています。Microsoft Foundry モデルプロバイダーのサポートを OpenClaw に追加します。

- **[migrate-claude](/ja-JP/plugins/reference/migrate-claude)** (`@openclaw/migrate-claude`) - OpenClaw に含まれています。Claude Code および Claude Desktop の指示、MCP サーバー、Skills、安全な設定を OpenClaw にインポートします。

- **[migrate-hermes](/ja-JP/plugins/reference/migrate-hermes)** (`@openclaw/migrate-hermes`) - OpenClaw に含まれています。Hermes の設定、メモリ、Skills、サポートされている認証情報を OpenClaw にインポートします。

- **[minimax](/ja-JP/plugins/reference/minimax)** (`@openclaw/minimax-provider`) - OpenClaw に含まれています。MiniMax および MiniMax Portal モデルプロバイダーのサポートを OpenClaw に追加します。

- **[mistral](/ja-JP/plugins/reference/mistral)** (`@openclaw/mistral-provider`) - OpenClaw に含まれています。Mistral モデルプロバイダーのサポートを OpenClaw に追加します。

- **[novita](/ja-JP/plugins/reference/novita)** (`@openclaw/novita-provider`) - OpenClaw に含まれています。Novita、Novita AI、Novitaai モデルプロバイダーのサポートを OpenClaw に追加します。

- **[nvidia](/ja-JP/plugins/reference/nvidia)** (`@openclaw/nvidia-provider`) - OpenClaw に含まれています。NVIDIA モデルプロバイダーのサポートを OpenClaw に追加します。

- **[oc-path](/ja-JP/plugins/reference/oc-path)** (`@openclaw/oc-path`) - OpenClaw に含まれています。oc:// ワークスペースファイルアドレス指定用の openclaw path CLI を追加します。

- **[ollama](/ja-JP/plugins/reference/ollama)** (`@openclaw/ollama-provider`) - OpenClaw に含まれています。Ollama および Ollama Cloud モデルプロバイダーのサポートを OpenClaw に追加します。

- **[onepassword](/ja-JP/plugins/reference/onepassword)** (`@openclaw/onepassword`) - OpenClaw に含まれています。承認ポリシーと SQLite 監査履歴を備えた、厳選された 1Password シークレットブローカーです。

- **[open-prose](/ja-JP/plugins/reference/open-prose)** (`@openclaw/open-prose`) - OpenClaw に含まれています。/prose スラッシュコマンドを備えた OpenProse VM スキルパックです。

- **[openai](/ja-JP/plugins/reference/openai)** (`@openclaw/openai-provider`) - OpenClaw に含まれています。OpenAI モデルプロバイダーのサポートを OpenClaw に追加します。

- **[opencode](/ja-JP/plugins/reference/opencode)** (`@openclaw/opencode-provider`) - OpenClaw に含まれています。OpenCode モデルプロバイダーのサポートを OpenClaw に追加します。

- **[opencode-go](/ja-JP/plugins/reference/opencode-go)** (`@openclaw/opencode-go-provider`) - OpenClaw に含まれています。OpenCode Go モデルプロバイダーのサポートを OpenClaw に追加します。

- **[openrouter](/ja-JP/plugins/reference/openrouter)** (`@openclaw/openrouter-provider`) - OpenClaw に含まれています。OpenRouter モデルプロバイダーのサポートを OpenClaw に追加します。

- **[policy](/ja-JP/plugins/reference/policy)** (`@openclaw/policy`) - OpenClaw に含まれています。ワークスペースの適合性を検証する、ポリシーに基づく doctor チェックを追加します。

- **[reef](/ja-JP/plugins/reference/reef)** (`@openclaw/reef`) - OpenClaw に含まれています。保護されたエンドツーエンド暗号化 claw チャンネルです。

- **[runway](/ja-JP/plugins/reference/runway)** (`@openclaw/runway-provider`) - OpenClaw に含まれています。動画生成プロバイダーのサポートを追加します。

- **[senseaudio](/ja-JP/plugins/reference/senseaudio)** (`@openclaw/senseaudio-provider`) - OpenClaw に含まれています。メディア理解プロバイダーのサポートを追加します。

- **[sglang](/ja-JP/plugins/reference/sglang)** (`@openclaw/sglang-provider`) - OpenClaw に含まれています。SGLang モデルプロバイダーのサポートを OpenClaw に追加します。

- **[synthetic](/ja-JP/plugins/reference/synthetic)** (`@openclaw/synthetic-provider`) - OpenClaw に含まれています。Synthetic モデルプロバイダーのサポートを OpenClaw に追加します。

- **[teams-meetings](/ja-JP/plugins/reference/teams-meetings)** (`@openclaw/teams-meetings`) - OpenClaw に含まれています。Chrome ブラウザーのゲストとして Microsoft Teams 会議に参加します。

- **[telegram](/ja-JP/plugins/reference/telegram)** (`@openclaw/telegram`) - OpenClaw に含まれています。OpenClaw メッセージを送受信するための Telegram チャンネルサーフェスを追加します。

- **[together](/ja-JP/plugins/reference/together)** (`@openclaw/together-provider`) - OpenClaw に含まれています。Together モデルプロバイダーのサポートを OpenClaw に追加します。

- **[tts-local-cli](/ja-JP/plugins/reference/tts-local-cli)** (`@openclaw/tts-local-cli`) - OpenClaw に含まれています。テキスト読み上げプロバイダーのサポートを追加します。

- **[vault](/ja-JP/plugins/reference/vault)** (`@openclaw/vault`) - OpenClaw に同梱。HashiCorp Vault SecretRef プロバイダー統合。

- **[vllm](/ja-JP/plugins/reference/vllm)** (`@openclaw/vllm-provider`) - OpenClaw に同梱。OpenClaw に vLLM モデルプロバイダーのサポートを追加します。

- **[volcengine](/ja-JP/plugins/reference/volcengine)** (`@openclaw/volcengine-provider`) - OpenClaw に同梱。OpenClaw に Volcengine、Volcengine Plan モデルプロバイダーのサポートを追加します。

- **[voyage](/ja-JP/plugins/reference/voyage)** (`@openclaw/voyage-provider`) - OpenClaw に同梱。メモリ埋め込みプロバイダーのサポートを追加します。

- **[vydra](/ja-JP/plugins/reference/vydra)** (`@openclaw/vydra-provider`) - OpenClaw に同梱。OpenClaw に Vydra モデルプロバイダーのサポートを追加します。

- **[web-readability](/ja-JP/plugins/reference/web-readability)** (`@openclaw/web-readability-plugin`) - OpenClaw に同梱。ローカルの HTML ウェブ取得レスポンスから読みやすい記事コンテンツを抽出します。

- **[webhooks](/ja-JP/plugins/reference/webhooks)** (`@openclaw/webhooks`) - OpenClaw に同梱。外部オートメーションを OpenClaw TaskFlow に関連付ける、認証済みの受信 Webhook。

- **[workboard](/ja-JP/plugins/reference/workboard)** (`@openclaw/workboard`) - OpenClaw に同梱。エージェントが所有する Issue とセッション向けのダッシュボード型ワークボード。

- **[xai](/ja-JP/plugins/reference/xai)** (`@openclaw/xai-plugin`) - OpenClaw に同梱。OpenClaw に xAI モデルプロバイダーのサポートを追加します。

- **[xiaomi](/ja-JP/plugins/reference/xiaomi)** (`@openclaw/xiaomi-provider`) - OpenClaw に同梱。OpenClaw に Xiaomi、Xiaomi Token Plan モデルプロバイダーのサポートを追加します。

- **[zoom-meetings](/plugins/reference/zoom-meetings)** (`@openclaw/zoom-meetings`) - OpenClaw に同梱。Chrome ブラウザーのゲストとして Zoom ミーティングに参加します。

## 公式外部パッケージ

72 個の Plugin

- **[acpx](/ja-JP/plugins/reference/acpx)** (`@openclaw/acpx`) - npm、ClawHub。Plugin が所有するセッションおよびトランスポート管理を備えた OpenClaw ACP ランタイムバックエンド。

- **[amazon-bedrock](/ja-JP/plugins/reference/amazon-bedrock)** (`@openclaw/amazon-bedrock-provider`) - npm、ClawHub。モデル検出、埋め込み、ガードレールのサポートを備えた OpenClaw Amazon Bedrock プロバイダー Plugin。

- **[amazon-bedrock-mantle](/ja-JP/plugins/reference/amazon-bedrock-mantle)** (`@openclaw/amazon-bedrock-mantle-provider`) - npm、ClawHub。OpenAI 互換モデルルーティング向けの OpenClaw Amazon Bedrock Mantle プロバイダー Plugin。

- **[anthropic-vertex](/ja-JP/plugins/reference/anthropic-vertex)** (`@openclaw/anthropic-vertex-provider`) - npm、ClawHub。Google Vertex AI 上の Claude モデル向け OpenClaw Anthropic Vertex プロバイダー Plugin。

- **[arcee](/ja-JP/plugins/reference/arcee)** (`@openclaw/arcee-provider`) - npm、ClawHub: `clawhub:@openclaw/arcee-provider`。OpenClaw に Arcee モデルプロバイダーのサポートを追加します。

- **[baseten](/plugins/reference/baseten)** (`@openclaw/baseten-provider`) - npm、ClawHub: `clawhub:@openclaw/baseten-provider`。OpenClaw Baseten プロバイダー Plugin。

- **[brave](/ja-JP/plugins/reference/brave)** (`@openclaw/brave-plugin`) - npm、ClawHub。ウェブ検索向け OpenClaw Brave Search プロバイダー Plugin。

- **[cerebras](/ja-JP/plugins/reference/cerebras)** (`@openclaw/cerebras-provider`) - npm、ClawHub: `clawhub:@openclaw/cerebras-provider`。OpenClaw に Cerebras モデルプロバイダーのサポートを追加します。

- **[chutes](/ja-JP/plugins/reference/chutes)** (`@openclaw/chutes-provider`) - npm、ClawHub: `clawhub:@openclaw/chutes-provider`。OpenClaw に Chutes モデルプロバイダーのサポートを追加します。

- **[clickclack](/ja-JP/plugins/reference/clickclack)** (`@openclaw/clickclack`) - npm、ClawHub: `clawhub:@openclaw/clickclack`。OpenClaw メッセージを送受信するための Clickclack チャネルサーフェスを追加します。

- **[cloudflare-ai-gateway](/ja-JP/plugins/reference/cloudflare-ai-gateway)** (`@openclaw/cloudflare-ai-gateway-provider`) - npm、ClawHub: `clawhub:@openclaw/cloudflare-ai-gateway-provider`。OpenClaw に Cloudflare AI Gateway モデルプロバイダーのサポートを追加します。

- **[codex](/ja-JP/plugins/reference/codex)** (`@openclaw/codex`) - npm、ClawHub。Codex app-server ハーネスとネイティブセッションカタログ。

- **[copilot](/ja-JP/plugins/reference/copilot)** (`@openclaw/copilot`) - npm、ClawHub: `clawhub:@openclaw/copilot`。GitHub Copilot エージェントランタイムを登録します。

- **[deepinfra](/ja-JP/plugins/reference/deepinfra)** (`@openclaw/deepinfra-provider`) - npm、ClawHub: `clawhub:@openclaw/deepinfra-provider`。OpenClaw に DeepInfra モデルプロバイダーのサポートを追加します。

- **[deepseek](/ja-JP/plugins/reference/deepseek)** (`@openclaw/deepseek-provider`) - npm、ClawHub: `clawhub:@openclaw/deepseek-provider`。OpenClaw に DeepSeek モデルプロバイダーのサポートを追加します。

- **[diagnostics-otel](/ja-JP/plugins/reference/diagnostics-otel)** (`@openclaw/diagnostics-otel`) - npm、ClawHub: `clawhub:@openclaw/diagnostics-otel`。メトリクス、トレース、ログ向けの OpenClaw 診断用 OpenTelemetry エクスポーター。

- **[diagnostics-prometheus](/ja-JP/plugins/reference/diagnostics-prometheus)** (`@openclaw/diagnostics-prometheus`) - npm、ClawHub: `clawhub:@openclaw/diagnostics-prometheus`。ランタイムメトリクス向けの OpenClaw 診断用 Prometheus エクスポーター。

- **[diffs](/ja-JP/plugins/reference/diffs)** (`@openclaw/diffs`) - npm、ClawHub。エージェント向けの OpenClaw 読み取り専用差分ビューアー Plugin およびファイルレンダラー。

- **[diffs-language-pack](/ja-JP/plugins/reference/diffs-language-pack)** (`@openclaw/diffs-language-pack`) - npm、ClawHub: `clawhub:@openclaw/diffs-language-pack`。デフォルトの差分ビューアーセットに含まれない言語の構文強調表示を追加します。

- **[discord](/ja-JP/plugins/reference/discord)** (`@openclaw/discord`) - npm、ClawHub。チャネル、DM、コマンド、アプリイベントに対応する OpenClaw Discord チャネル Plugin。

- **[exa](/ja-JP/plugins/reference/exa)** (`@openclaw/exa-plugin`) - npm、ClawHub: `clawhub:@openclaw/exa-plugin`。ウェブ検索プロバイダーのサポートを追加します。

- **[featherless](/ja-JP/plugins/reference/featherless)** (`@openclaw/featherless-provider`) - npm、ClawHub: `clawhub:@openclaw/featherless-provider`。OpenClaw Featherless AI プロバイダー Plugin。

- **[feishu](/ja-JP/plugins/reference/feishu)** (`@openclaw/feishu`) - npm、ClawHub。チャットとワークプレイスツール向けの OpenClaw Feishu/Lark チャネル Plugin（@m1heng によりコミュニティメンテナンス）。

- **[firecrawl](/ja-JP/plugins/reference/firecrawl)** (`@openclaw/firecrawl-plugin`) - npm、ClawHub: `clawhub:@openclaw/firecrawl-plugin`。エージェントから呼び出し可能なツールを追加します。ウェブ取得プロバイダーのサポートを追加します。ウェブ検索プロバイダーのサポートを追加します。

- **[fireworks](/ja-JP/plugins/reference/fireworks)** (`@openclaw/fireworks-provider`) - npm、ClawHub: `clawhub:@openclaw/fireworks-provider`。OpenClaw に Fireworks モデルプロバイダーのサポートを追加します。

- **[gmi](/ja-JP/plugins/reference/gmi)** (`@openclaw/gmi-provider`) - npm、ClawHub: `clawhub:@openclaw/gmi-provider`。OpenClaw GMI Cloud プロバイダー Plugin。

- **[google-meet](/ja-JP/plugins/reference/google-meet)** (`@openclaw/google-meet`) - npm、ClawHub。Chrome または Twilio トランスポート経由で通話に参加するための OpenClaw Google Meet 参加者 Plugin。

- **[googlechat](/ja-JP/plugins/reference/googlechat)** (`@openclaw/googlechat`) - npm、ClawHub。スペースとダイレクトメッセージ向けの OpenClaw Google Chat チャネル Plugin。

- **[gradium](/ja-JP/plugins/reference/gradium)** (`@openclaw/gradium-speech`) - npm、ClawHub: `clawhub:@openclaw/gradium-speech`。テキスト読み上げプロバイダーのサポートを追加します。

- **[groq](/ja-JP/plugins/reference/groq)** (`@openclaw/groq-provider`) - npm、ClawHub: `clawhub:@openclaw/groq-provider`。OpenClaw に Groq モデルプロバイダーのサポートを追加します。

- **[inworld](/ja-JP/plugins/reference/inworld)** (`@openclaw/inworld-speech`) - npm、ClawHub: `clawhub:@openclaw/inworld-speech`。Inworld ストリーミングテキスト読み上げ（MP3、OGG_OPUS、PCM テレフォニー）。

- **[irc](/ja-JP/plugins/reference/irc)** (`@openclaw/irc`) - npm、ClawHub: `clawhub:@openclaw/irc`。OpenClaw メッセージを送受信するための IRC チャネルサーフェスを追加します。

- **[kilocode](/ja-JP/plugins/reference/kilocode)** (`@openclaw/kilocode-provider`) - npm、ClawHub: `clawhub:@openclaw/kilocode-provider`。OpenClaw に Kilocode モデルプロバイダーのサポートを追加します。

- **[kimi](/ja-JP/plugins/reference/kimi)** (`@openclaw/kimi-provider`) - npm、ClawHub: `clawhub:@openclaw/kimi-provider`。OpenClaw に Kimi、Kimi Coding モデルプロバイダーのサポートを追加します。

- **[line](/ja-JP/plugins/reference/line)** (`@openclaw/line`) - npm、ClawHub。LINE Bot API チャット向けの OpenClaw LINE チャネル Plugin。

- **[llama-cpp](/ja-JP/plugins/reference/llama-cpp)** (`@openclaw/llama-cpp-provider`) - npm、ClawHub。node-llama-cpp を介したローカル GGUF テキスト推論と埋め込み。

- **[lobster](/ja-JP/plugins/reference/lobster)** (`@openclaw/lobster`) - npm、ClawHub。型付きパイプラインと再開可能な承認向けの Lobster ワークフローツール Plugin。

- **[longcat](/ja-JP/plugins/reference/longcat)** (`@openclaw/longcat-provider`) - npm、ClawHub: `clawhub:@openclaw/longcat-provider`。OpenClaw LongCat プロバイダー Plugin。

- **[matrix](/ja-JP/plugins/reference/matrix)** (`@openclaw/matrix`) - ClawHub: `clawhub:@openclaw/matrix`、npm。ルームとダイレクトメッセージ向けの OpenClaw Matrix チャネル Plugin。

- **[mattermost](/ja-JP/plugins/reference/mattermost)** (`@openclaw/mattermost`) - npm、ClawHub: `clawhub:@openclaw/mattermost`。OpenClaw メッセージを送受信するための Mattermost チャネルサーフェスを追加します。

- **[memory-lancedb](/ja-JP/plugins/reference/memory-lancedb)** (`@openclaw/memory-lancedb`) - npm、ClawHub。自動想起、自動取得、ベクトル検索を備えた、LanceDB を基盤とする OpenClaw 長期メモリ Plugin。

- **[moonshot](/ja-JP/plugins/reference/moonshot)** (`@openclaw/moonshot-provider`) - npm、ClawHub: `clawhub:@openclaw/moonshot-provider`。OpenClaw に Moonshot モデルプロバイダーのサポートを追加します。

- **[msteams](/ja-JP/plugins/reference/msteams)** (`@openclaw/msteams`) - npm、ClawHub。ボット会話向けの OpenClaw Microsoft Teams チャネル Plugin。

- **[mxc](/ja-JP/plugins/reference/mxc)** (`@openclaw/mxc-sandbox`) - npm、ClawHub。MXC を介した OS レベルのサンドボックス化ツール実行。設定済みの MXC ポリシーファイルを使用し、Windows ProcessContainer 内でコマンドを実行します。

- **[nextcloud-talk](/ja-JP/plugins/reference/nextcloud-talk)** (`@openclaw/nextcloud-talk`) - npm、ClawHub。会話向けの OpenClaw Nextcloud Talk チャネル Plugin。

- **[nostr](/ja-JP/plugins/reference/nostr)** (`@openclaw/nostr`) - npm、ClawHub。NIP-04 で暗号化されたダイレクトメッセージ向けの OpenClaw Nostr チャネル Plugin。

- **[openshell](/ja-JP/plugins/reference/openshell)** (`@openclaw/openshell-sandbox`) - npm、ClawHub。ミラーリングされたローカルワークスペースと SSH コマンド実行を備えた、NVIDIA OpenShell CLI 向け OpenClaw サンドボックスバックエンド。

- **[parallel](/ja-JP/tools/parallel-search)** (`@openclaw/parallel-plugin`) - npm、ClawHub: `clawhub:@openclaw/parallel-plugin`。ウェブ検索プロバイダーのサポートを追加します。

- **[perplexity](/ja-JP/plugins/reference/perplexity)** (`@openclaw/perplexity-plugin`) - npm、ClawHub: `clawhub:@openclaw/perplexity-plugin`。ウェブ検索プロバイダーのサポートを追加します。

- **[pixverse](/ja-JP/plugins/reference/pixverse)** (`@openclaw/pixverse-provider`) - npm、ClawHub: `clawhub:@openclaw/pixverse-provider`。OpenClaw PixVerse 動画生成プロバイダー Plugin。

- **[qianfan](/ja-JP/plugins/reference/qianfan)** (`@openclaw/qianfan-provider`) - npm、ClawHub: `clawhub:@openclaw/qianfan-provider`。OpenClaw に Qianfan モデルプロバイダーのサポートを追加します。

- **[qqbot](/ja-JP/plugins/reference/qqbot)** (`@openclaw/qqbot`) - npm、ClawHub。グループおよびダイレクトメッセージのワークフロー向け OpenClaw QQ Bot チャネル Plugin。

- **[qwen](/ja-JP/plugins/reference/qwen)** (`@openclaw/qwen-provider`) - npm、ClawHub: `clawhub:@openclaw/qwen-provider`。OpenClaw に Qwen、Qwen Cloud、Model Studio、DashScope、Qwen Token Plan、Bailian Token Plan モデルプロバイダーのサポートを追加します。

- **[raft](/ja-JP/plugins/reference/raft)** (`@openclaw/raft`) - npm、ClawHub。セキュアな CLI ウェイクブリッジ向けの OpenClaw Raft チャネル Plugin。

- **[searxng](/ja-JP/plugins/reference/searxng)** (`@openclaw/searxng-plugin`) - npm、ClawHub: `clawhub:@openclaw/searxng-plugin`。ウェブ検索プロバイダーのサポートを追加します。

- **[signal](/ja-JP/plugins/reference/signal)** (`@openclaw/signal`) - npm、ClawHub: `clawhub:@openclaw/signal`。OpenClaw メッセージを送受信するための Signal チャネルサーフェスを追加します。

- **[slack](/ja-JP/plugins/reference/slack)** (`@openclaw/slack`) - npm、ClawHub。チャネル、DM、コマンド、アプリイベントに対応する OpenClaw Slack チャネル Plugin。

- **[sms](/ja-JP/plugins/reference/sms)** (`@openclaw/sms`) - npm; ClawHub: `clawhub:@openclaw/sms`。OpenClawのテキストメッセージ向けTwilio SMSチャネルPlugin。

- **[stepfun](/ja-JP/plugins/reference/stepfun)** (`@openclaw/stepfun-provider`) - npm; ClawHub: `clawhub:@openclaw/stepfun-provider`。OpenClawにStepFun、StepFun Planモデルプロバイダーのサポートを追加します。

- **[synology-chat](/ja-JP/plugins/reference/synology-chat)** (`@openclaw/synology-chat`) - npm; ClawHub。OpenClawのチャネルとダイレクトメッセージ向けSynology ChatチャネルPlugin。

- **[tavily](/ja-JP/plugins/reference/tavily)** (`@openclaw/tavily-plugin`) - npm; ClawHub: `clawhub:@openclaw/tavily-plugin`。エージェントから呼び出せるツールを追加します。ウェブ検索プロバイダーのサポートを追加します。

- **[tencent](/ja-JP/plugins/reference/tencent)** (`@openclaw/tencent-provider`) - npm; ClawHub: `clawhub:@openclaw/tencent-provider`。OpenClawにTencent TokenHub、Tencent Tokenplanモデルプロバイダーのサポートを追加します。

- **[tlon](/ja-JP/plugins/reference/tlon)** (`@openclaw/tlon`) - npm; ClawHub。チャットワークフロー向けOpenClaw Tlon/UrbitチャネルPlugin。

- **[tokenjuice](/ja-JP/plugins/reference/tokenjuice)** (`@openclaw/tokenjuice`) - npm; ClawHub: `clawhub:@openclaw/tokenjuice`。Tokenjuiceリデューサーを使用してexecおよびbashツールの結果を圧縮します。

- **[twitch](/ja-JP/plugins/reference/twitch)** (`@openclaw/twitch`) - npm; ClawHub。チャットおよびモデレーションワークフロー向けOpenClaw TwitchチャネルPlugin。

- **[venice](/ja-JP/plugins/reference/venice)** (`@openclaw/venice-provider`) - npm; ClawHub: `clawhub:@openclaw/venice-provider`。OpenClawにVeniceモデルプロバイダーのサポートを追加します。

- **[vercel-ai-gateway](/ja-JP/plugins/reference/vercel-ai-gateway)** (`@openclaw/vercel-ai-gateway-provider`) - npm; ClawHub: `clawhub:@openclaw/vercel-ai-gateway-provider`。OpenClawにVercel AI Gatewayモデルプロバイダーのサポートを追加します。

- **[voice-call](/ja-JP/plugins/reference/voice-call)** (`@openclaw/voice-call`) - npm; ClawHub。Twilio、Telnyx、Plivoの電話通話向けOpenClaw音声通話Plugin。

- **[whatsapp](/ja-JP/plugins/reference/whatsapp)** (`@openclaw/whatsapp`) - ClawHub: `clawhub:@openclaw/whatsapp`; npm。WhatsApp Webチャット向けOpenClaw WhatsAppチャネルPlugin。

- **[zai](/ja-JP/plugins/reference/zai)** (`@openclaw/zai-provider`) - npm; ClawHub: `clawhub:@openclaw/zai-provider`。OpenClawにZ.AIモデルプロバイダーのサポートを追加します。

- **[zalo](/ja-JP/plugins/reference/zalo)** (`@openclaw/zalo`) - npm; ClawHub。ボットおよびWebhookチャット向けOpenClaw ZaloチャネルPlugin。

- **[zalouser](/ja-JP/plugins/reference/zalouser)** (`@openclaw/zalouser`) - npm; ClawHub。ネイティブのzca-js連携を使用するOpenClaw Zalo個人アカウントPlugin。

## ソースチェックアウト限定

2個のPlugin

- **[qa-channel](/ja-JP/plugins/reference/qa-channel)** (`@openclaw/qa-channel`) - ソースチェックアウト限定。OpenClawメッセージを送受信するためのQA Channelサーフェスを追加します。

- **[qa-lab](/ja-JP/plugins/reference/qa-lab)** (`@openclaw/qa-lab`) - ソースチェックアウト限定。非公開デバッガーUIとシナリオランナーを備えたOpenClaw QAラボPlugin。
