---
read_when:
    - OpenClaw がサポートする機能の完全な一覧を確認したい場合
summary: チャネル、ルーティング、メディア、UX 全般にわたる OpenClaw の機能。
title: 機能
x-i18n:
    generated_at: "2026-07-26T09:18:02Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 5bc3ebdd87a0f6ea0f3d75d029bf7cae469ecd9db84a165bd47c4896936fe303
    source_path: concepts/features.md
    workflow: 16
---

## ハイライト

<Columns>
  <Card title="チャンネル" icon="message-square" href="/ja-JP/channels">
    1 つの Gateway で Discord、iMessage、Signal、Slack、Telegram、WhatsApp、WebChat などを利用できます。
  </Card>
  <Card title="プラグイン" icon="plug" href="/ja-JP/tools/plugin">
    公式プラグインを 1 つのインストールコマンドで追加すると、Matrix、Nextcloud Talk、Nostr、Twitch、Zalo など数十種類を利用できます。
  </Card>
  <Card title="ルーティング" icon="route" href="/ja-JP/concepts/multi-agent">
    セッションを分離したマルチエージェントルーティング。
  </Card>
  <Card title="メディア" icon="image" href="/ja-JP/nodes/images">
    画像、音声、動画、ドキュメント、および画像・動画の生成。
  </Card>
  <Card title="アプリと UI" icon="monitor" href="/ja-JP/platforms">
    Windows Hub、ブラウザー版 Control UI、macOS メニューバーアプリ、モバイル Node。
  </Card>
  <Card title="モバイル Node" icon="smartphone" href="/ja-JP/nodes">
    ペアリング、音声・チャット、高度なデバイスコマンドに対応した iOS および Android Node。
  </Card>
</Columns>

## 全一覧

**チャンネル:**

- iMessage、Telegram、WebChat はコアインストールに同梱されています。その他のチャンネルはすべて、
  `openclaw plugins install @openclaw/<id>` でインストールする公式プラグインです（または
  `openclaw onboard` / `openclaw channels add` の実行中に必要に応じてインストールできます）
- 公式プラグインチャンネル: Discord、Feishu、Google Chat、IRC、LINE、Matrix、Mattermost、
  Microsoft Teams、Nextcloud Talk、Nostr、QQ Bot、Raft、Signal、Slack、SMS、Synology Chat、
  Tlon、Twitch、Voice Call、WhatsApp、Zalo、Zalo Personal
- OpenClaw リポジトリ外で保守されている外部プラグインチャンネル: WeChat、Yuanbao、Zalo ClawBot
- メンションによる起動に対応したグループチャット
- 許可リストとペアリングによる DM の安全性確保

**エージェント:**

- ツールストリーミングに対応した組み込みエージェントランタイム
- ワークスペースまたは送信者ごとにセッションを分離するマルチエージェントルーティング
- セッション: ダイレクトチャットは共有 `main` に統合され、グループは分離されます
- 長い応答のストリーミングと分割

**認証とプロバイダー:**

- 35 以上のモデルプロバイダー（Anthropic、OpenAI、Google など）
- OAuth によるサブスクリプション認証（例: OpenAI Codex）
- カスタムおよびセルフホスト型プロバイダーのサポート（vLLM、SGLang、Ollama、llama.cpp、LM Studio、および
  OpenAI 互換または Anthropic 互換の任意のエンドポイント）

**メディア:**

- 画像、音声、動画、ドキュメントの入出力
- 画像生成と動画生成で共有される機能サーフェス
- ボイスメモの文字起こし
- 複数のプロバイダーによるテキスト読み上げ

**アプリとインターフェース:**

- WebChat とブラウザー版 Control UI
- macOS メニューバーコンパニオンアプリ
- ペアリング、Canvas、カメラ、画面収録、位置情報、音声に対応した iOS Node
- ペアリング、チャット、音声、Canvas、カメラ、デバイスコマンドに対応した Android Node

**ツールと自動化:**

- ブラウザー自動化、コマンド実行、サンドボックス化
- Web 検索（Brave、DuckDuckGo、Exa、Firecrawl、Gemini、Grok、Kimi、MiniMax Search、Ollama Web Search、Perplexity、SearXNG、Tavily）
- Cron ジョブと Heartbeat のスケジュール設定
- Skills、プラグイン、ワークフローパイプライン（Lobster）

## 関連項目

<CardGroup cols={2}>
  <Card title="試験的な機能" href="/ja-JP/concepts/experimental-features" icon="flask">
    デフォルトのサーフェスにはまだ提供されていない、オプトイン形式の機能。
  </Card>
  <Card title="エージェントランタイム" href="/ja-JP/concepts/agent" icon="robot">
    エージェントのランタイムモデルと実行の振り分け方法。
  </Card>
  <Card title="チャンネル" href="/ja-JP/channels" icon="message-square">
    1 つの Gateway から Telegram、WhatsApp、Discord、Slack などに接続できます。
  </Card>
  <Card title="プラグイン" href="/ja-JP/tools/plugin" icon="plug">
    OpenClaw を拡張する公式および外部プラグイン。
  </Card>
</CardGroup>
