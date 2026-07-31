---
description: Real-world OpenClaw projects from the community
read_when:
    - OpenClaw の実際の使用例を探す
    - コミュニティプロジェクトのハイライトを更新する
summary: OpenClaw を活用したコミュニティ製のプロジェクトと連携機能
title: ショーケース
x-i18n:
    generated_at: "2026-07-26T09:46:10Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 64af6f1da52ebdccff82fe2cdb0f7a5f0cd57627b08ee796369e2933f47fbae4
    source_path: start/showcase.md
    workflow: 16
---

コミュニティが構築した OpenClaw プロジェクト：PR レビューループ、モバイルアプリ、ホームオートメーション、音声システム、開発ツール、メモリワークフローを、Telegram、WhatsApp、Discord、ターミナル上でチャットネイティブに構築。

<Info>
**掲載を希望しますか？** [Discord の #self-promotion](https://discord.gg/clawd) でプロジェクトを共有するか、[X で @openclaw をタグ付け](https://x.com/openclaw)してください。
</Info>

## Discord から届いた新着

コーディング、開発ツール、モバイル、チャットネイティブなプロダクト開発における最近の注目作。

<CardGroup cols={2}>

<Card title="Dropage の即時 HTML デプロイ" icon="cloud-arrow-up" href="https://clawhub.ai/jiantoucn/skills/dropage-deploy">
  **@jiantoucn** • `deploy` `hosting` `skill`

エージェントに「この HTML をデプロイして」と伝えると、約 1 秒で公開 URL が返されます。ページは 1 時間後に自動で期限切れになります。サーバー、設定、登録は不要です。
</Card>

<Card title="詐欺対策 URL チェッカー" icon="shield-halved" href="https://clawhub.ai/phishguard-niki/anti-scam-guard">
  **@phishguard-niki** • `security` `phishing` `skill`

任意の URL を貼り付けるだけで、判定結果を得られます。38 件のフィード（PhishTank、OpenPhish、CERT.PL など）から収集した 250 万以上の詐欺ドメインとローカルで照合するため、閲覧履歴がマシン外に送信されることはありません。
</Card>

<Card title="プロダクト設計の推論 Skills" icon="pen-ruler" href="https://clawhub.ai/monikazapisekstudio/skills/socratic-dialog">
  **@monikazapisekstudio** • `product` `reasoning` `skills`

プロダクト開発向けの 3 点セットです。[ソクラテス式対話](https://clawhub.ai/monikazapisekstudio/skills/socratic-dialog)は回答前に問いをさまざまな角度から検討し、[狩野モデルストラテジスト](https://clawhub.ai/monikazapisekstudio/skills/kano-model-strategist)は機能を採用する価値に応じて分類し、[読みやすいエージェント出力](https://clawhub.ai/monikazapisekstudio/skills/legible-agent-output)はエージェントの出力を平易な言葉に書き換えます。
</Card>

<Card title="サブエージェント用メールボックスブローカー" icon="inbox" href="https://clawhub.ai/albzhu/skills/miab-broker">
  **@albzhu** • `multi-agent` `async` `skill`

サブエージェントの作業中にオーケストレーターが待機状態になるのを防ぎます。親エージェントをブロックせず、結果をメールボックスに届ける非同期コールバック機構です。
</Card>

<Card title="メモリの少ないマシン向け lite-mode" icon="feather" href="https://clawhub.ai/skills/lite-mode">
  **@mirajmahmudul** • `performance` `skill`

2～4 GB のマシンでも OpenClaw を実用的に使えるようにします。空きメモリを確認し、マシンがスワップを始める前に負荷の高い機能を削減します。[GitHub のソース](https://github.com/mirajmahmudul/openclaw-lite-mode)。
</Card>

<Card title="tokenomics コストトラッカー" icon="coins" href="https://github.com/ncz-os/tokenomics">
  **@ncz-os** • `devtools` `costs` `tokens`

NVIDIA のエンジニアが開発した、OpenClaw を第一級でサポートするトークンコストトラッカーです。エージェントの支出先をモデル別、セッション別に正確に確認できます。
</Card>

<Card title="Excalidraw 図解ジェネレーター" icon="shapes" href="https://x.com/swiftlysingh/status/2009684853827281070">
  **@swiftlysingh** • `diagrams` `excalidraw` `devtools`

チャットで図を説明すると、プログラムで生成された Excalidraw のスケッチが返されます。
</Card>

<Card title="GA4 分析 Skill" icon="chart-column" href="https://x.com/jdrhyne/status/2012028725710192741">
  **@jdrhyne** • `analytics` `ga4` `skill`

OpenClaw に独自の Google Analytics クエリツールを構築させた後、パッケージ化して ClawHub に公開しました。
</Card>

<Card title="ClawEval モデルランキング" icon="ranking-star" href="https://github.com/AIgenteur/ClawEval">
  **@AIgenteur** • `evals` `models` `devtools`

59 種類のエージェントの役割にわたってモデルをベンチマークし、「自分の GPU にはどの LLM が適しているか？」という問いに答えます。ローカルモデル選びでコミュニティに人気のツールです。
</Card>

<Card title="Music Craft" icon="music" href="https://clawhub.ai/luischarro/music-craft">
  **@luischarro** • `music` `generation` `skill`

プロバイダーに依存しない楽曲生成です。一度きりのプロンプトではなく、楽曲を計画し、歌詞を構成して、不十分な結果を修正します。BPM、キー、構成、マッシュアップを制御できる [MiniMax バリアント](https://clawhub.ai/luischarro/music-craft-minimax)も含まれます。
</Card>

<Card title="PR レビューから Telegram へのフィードバック" icon="code-pull-request" href="https://x.com/i/status/2010878524543131691">
  **@bangnokia** • `review` `github` `telegram`

OpenCode が変更を完了して PR を開き、OpenClaw が差分をレビューして、提案と明確なマージ判定を Telegram に返信します。

  <img src="/assets/showcase/pr-review-telegram.jpg" alt="Telegram に送信された OpenClaw の PR レビューフィードバック" />
</Card>

<Card title="数分で作るワインセラー Skill" icon="wine-glass" href="https://x.com/i/status/2010916352454791216">
  **@prades_maxime** • `skills` `local` `csv`

「Robby」（@openclaw）にローカルのワインセラー Skill を依頼しました。サンプルの CSV エクスポートと保存先パスを要求し、Skill を構築してテストします（例では 962 本）。
 
  <img src="/assets/showcase/wine-cellar-skill.jpg" alt="CSV からローカルのワインセラー Skill を構築する OpenClaw" />
</Card>

<Card title="Tesco の買い物オートパイロット" icon="cart-shopping" href="https://x.com/i/status/2009724862470689131">
  **@marchattonhere** • `automation` `browser` `shopping`

週間献立を作成し、定番商品を選び、配達時間枠を予約して、注文を確定します。API は使用せず、ブラウザー操作だけで実行します。

  <img src="/assets/showcase/tesco-shop.jpg" alt="チャットによる Tesco の買い物自動化" />
</Card>

<Card title="SNAG のスクリーンショットから Markdown への変換" icon="scissors" href="https://github.com/am-will/snag">
  **@am-will** • `devtools` `screenshots` `markdown`

ホットキーで画面領域を選択し、Gemini の画像認識を使って、即座に Markdown をクリップボードへ出力します。

  <img src="/assets/showcase/snag.png" alt="SNAG のスクリーンショットから Markdown への変換ツール" />
</Card>

<Card title="Agents UI" icon="window-maximize" href="https://releaseflow.net/kitze/agents-ui">
  **@kitze** • `ui` `skills` `sync`

Agents、Claude、Codex、OpenClaw の Skills とコマンドを一元管理するデスクトップアプリです。

  <img src="/assets/showcase/agents-ui.jpg" alt="Agents UI アプリ" />
</Card>

<Card title="Telegram 音声メモ（papla.media）" icon="microphone" href="https://papla.media/docs">
  **コミュニティ** • `voice` `tts` `telegram`

papla.media の TTS をラップし、結果を Telegram の音声メモとして送信します（煩わしい自動再生はありません）。

  <img src="/assets/showcase/papla-tts.jpg" alt="TTS から出力された Telegram 音声メモ" />
</Card>

<Card title="CodexMonitor" icon="eye" href="https://clawhub.ai/odrobnik/skills/codexmonitor">
  **@odrobnik** • `devtools` `codex` `brew`

ローカルの OpenAI Codex セッションを一覧表示、調査、監視するための、Homebrew でインストールできるヘルパーです（CLI + VS Code）。

  <img src="/assets/showcase/codexmonitor.png" alt="ClawHub 上の CodexMonitor" />
</Card>

<Card title="Bambu 3D プリンター制御" icon="print" href="https://clawhub.ai/tobiasbischoff/skills/bambu-cli">
  **@tobiasbischoff** • `hardware` `3d-printing` `skill`

BambuLab プリンターの制御とトラブルシューティングを行います。ステータス、ジョブ、カメラ、AMS、キャリブレーションなどに対応します。

  <img src="/assets/showcase/bambu-cli.png" alt="ClawHub 上の Bambu CLI Skill" />
</Card>

<Card title="ウィーンの交通機関（Wiener Linien）" icon="train" href="https://clawhub.ai/hjanuschka/skills/wienerlinien">
  **@hjanuschka** • `travel` `transport` `skill`

ウィーンの公共交通機関について、リアルタイムの出発時刻、運行障害、エレベーターの状況、経路案内を提供します。

  <img src="/assets/showcase/wienerlinien.png" alt="ClawHub 上の Wiener Linien Skill" />
</Card>

<Card title="ParentPay の学校給食" icon="utensils">
  **@George5562** • `automation` `browser` `parenting`

ParentPay を介して英国の学校給食予約を自動化します。表のセルを確実にクリックするため、マウス座標を使用します。
</Card>

<Card title="R2 アップロード（Send Me My Files）" icon="cloud-arrow-up" href="https://clawhub.ai/julianengel/skills/r2-upload">
  **@julianengel** • `files` `r2` `presigned-urls`

Cloudflare R2/S3 にアップロードし、安全な署名済みダウンロードリンクを生成します。リモートの OpenClaw インスタンスに便利です。

  <img src="/assets/showcase/r2-upload.png" alt="ClawHub 上の R2 アップロード Skill" />
</Card>

<Card title="Telegram 経由の iOS アプリ" icon="mobile">
  **@coard** • `ios` `xcode` `app-store`

地図と音声録音を備え、App Store で配布できる状態の完全な iOS アプリを、すべて Telegram チャット経由で構築しました。
</Card>

<Card title="Oura Ring 健康アシスタント" icon="heart-pulse">
  **@AS** • `health` `oura` `calendar`

Oura Ring のデータをカレンダー、予定、ジムのスケジュールと統合する、個人向け AI 健康アシスタントです。

  <img src="/assets/showcase/oura-health.png" alt="Oura Ring 健康アシスタント" />
</Card>

<Card title="Kev's Dream Team（14 以上のエージェント）" icon="robot" href="https://github.com/adam91holt/orchestrated-ai-articles">
  **@adam91holt** • `multi-agent` `orchestration`

1 つの Gateway 配下に 14 以上のエージェントを配置し、Opus 4.5 オーケストレーターが Codex ワーカーに委任します。[技術解説](https://github.com/adam91holt/orchestrated-ai-articles)と、エージェントのサンドボックス化に使用する [Clawdspace](https://github.com/adam91holt/clawdspace)を参照してください。
</Card>

<Card title="Linear CLI" icon="terminal" href="https://github.com/Finesssee/linear-cli">
  **@NessZerra** • `devtools` `linear` `cli`

エージェント型ワークフロー（Claude Code、OpenClaw）と統合できる Linear 用 CLI です。ターミナルから課題、プロジェクト、ワークフローを管理できます。
</Card>

<Card title="Beeper CLI" icon="message" href="https://github.com/blqke/beepcli">
  **@jules** • `messaging` `beeper` `cli`

Beeper Desktop を介してメッセージの閲覧、送信、アーカイブを行います。Beeper のローカル MCP API を使用するため、エージェントはすべてのチャット（iMessage、WhatsApp など）を一元管理できます。
</Card>

</CardGroup>

## 自動化とワークフロー

スケジューリング、ブラウザー制御、サポートループ、そしてプロダクトの「タスクを代わりに実行して」機能。

<CardGroup cols={2}>

<Card title="Winix 空気清浄機の制御" icon="wind" href="https://x.com/antonplex/status/2010518442471006253">
  **@antonplex** • `automation` `hardware` `air-quality`

Claude Code が空気清浄機の制御方法を発見して確認した後、OpenClaw が引き継いで室内の空気品質を管理します。

  <img src="/assets/showcase/winix-air-purifier.jpg" alt="OpenClaw による Winix 空気清浄機の制御" />
</Card>

<Card title="美しい空のカメラ写真" icon="camera" href="https://x.com/signalgaining/status/2010523120604746151">
  **@signalgaining** • `automation` `camera` `skill`

屋根のカメラをトリガーとして、空が美しく見えるたびに写真を撮るよう OpenClaw に依頼します。OpenClaw が Skill を設計し、撮影しました。

  <img src="/assets/showcase/roof-camera-sky.jpg" alt="OpenClaw が撮影した屋根カメラの空のスナップショット" />
</Card>

<Card title="視覚的な朝のブリーフィングシーン" icon="robot" href="https://x.com/buddyhadry/status/2010005331925954739">
  **@buddyhadry** • `automation` `briefing` `telegram`

スケジュールされたプロンプトが、OpenClaw のペルソナを介して毎朝 1 枚のシーン画像（天気、タスク、日付、お気に入りの投稿または引用）を生成します。
</Card>

<Card title="パデルコートの予約" icon="calendar-check" href="https://github.com/joshp123/padel-cli">
  **@joshp123** • `automation` `booking` `cli`

Playtomic の空き状況チェッカーと予約 CLI です。空いているコートをもう見逃しません。

  <img src="/assets/showcase/padel-screenshot.jpg" alt="padel-cli のスクリーンショット" />
</Card>

<Card title="会計書類の取り込み" icon="file-invoice-dollar">
  **コミュニティ** • `automation` `email` `pdf`

メールから PDF を収集し、税理士向けに書類を準備します。毎月の経理処理を自動化します。
</Card>

<Card title="寝ながら開発モード" icon="couch" href="https://davekiss.com">
  **@davekiss** • `telegram` `migration` `astro`

Netflix を見ながら、Telegram 経由で個人サイト全体を再構築。Notion から Astro へ移行し、18 件の投稿を移行、DNS を Cloudflare に変更しました。ノート PC は一度も開いていません。
</Card>

<Card title="求人検索エージェント" icon="briefcase">
  **@attol8** • `automation` `api` `skill`

求人情報を検索し、履歴書のキーワードと照合して、関連する求人をリンク付きで返します。JSearch API を使用して 30 分で構築されました。
</Card>

<Card title="Jira skill ビルダー" icon="diagram-project" href="https://x.com/jdrhyne/status/2008336434827002232">
  **@jdrhyne** • `jira` `skill` `devtools`

OpenClaw を Jira に接続し、その場で新しい skill を生成しました（ClawHub に登場する前のことです）。
</Card>

<Card title="Telegram 経由の Todoist skill" icon="list-check" href="https://x.com/iamsubhrajyoti/status/2009949389884920153">
  **@iamsubhrajyoti** • `todoist` `skill` `telegram`

Todoist のタスクを自動化し、OpenClaw に Telegram チャット内で直接 skill を生成させました。
</Card>

<Card title="TradingView 分析" icon="chart-line">
  **@bheem1798** • `finance` `browser` `automation`

ブラウザ自動化で TradingView にログインし、チャートのスクリーンショットを撮影して、要求に応じてテクニカル分析を実行します。API は不要で、ブラウザ操作だけで動作します。
</Card>

<Card title="自動車価格交渉（$4,200 節約）" icon="car-side" href="https://x.com/astuyve/status/2014147784098681217">
  **@astuyve** • `negotiation` `email` `automation`

OpenClaw に自動車販売店との交渉を任せたところ、やり取りをすべて処理し、価格を $4,200 引き下げました。
</Card>

<Card title="フライトチェックインの自動化" icon="plane-departure" href="https://x.com/armanddp/status/2008767951340794245">
  **@armanddp** • `travel` `email` `automation`

メールから次のフライトを見つけ、オンラインチェックインを行い、窓側の席を選択します。航空会社のアプリは必要ありません。
</Card>

<Card title="保険請求の申請" icon="file-signature" href="https://x.com/avi_press/status/2013066316467560521">
  **@avi_press** • `automation` `insurance` `browser`

保険請求を申請し、その後の予約を自律的に手配しました。
</Card>

<Card title="Idealista 不動産 skill" icon="building" href="https://x.com/quifago/status/2012458753786859872">
  **@quifago** • `real-estate` `api` `skill`

物件検索と査定のための Idealista API CLI を skill としてラップし、エージェントがチャット内で物件を探せるようにしました。
</Card>

<Card title="造園事業のバックオフィス" icon="seedling" href="https://news.ycombinator.com/item?id=47783940">
  **@mjsweet** • `automation` `email` `invoicing`

Gmail で作業指示を監視し、Telegram で送られた物件写真を分析して、複数ページの LaTeX 見積書 PDF を作成し、Xero で請求書を発行します。
</Card>

<Card title="Slack 自動サポート" icon="slack">
  **@henrymascot** • `slack` `automation` `support`

会社の Slack チャンネルを監視して役立つ回答を行い、通知を Telegram に転送します。依頼されることなく、デプロイ済みアプリの本番環境のバグを自律的に修正しました。
</Card>

</CardGroup>

## ナレッジとメモリ

個人やチームのナレッジをインデックス化、検索、記憶し、それに基づいて推論するシステムです。

<CardGroup cols={2}>

<Card title="xuezh 中国語学習" icon="language" href="https://github.com/joshp123/xuezh">
  **@joshp123** • `learning` `voice` `skill`

OpenClaw を通じて発音フィードバックと学習フローを提供する中国語学習エンジンです。

  <img src="/assets/showcase/xuezh-pronunciation.jpeg" alt="xuezh の発音フィードバック" />
</Card>

<Card title="X 投稿分析パイプライン" icon="hashtag" href="https://x.com/andrewjiang/status/2008388427180630155">
  **@andrewjiang** • `analysis` `x` `pipeline`

X の上位 100 アカウントから 400 万件の投稿を取得し、クエリ可能な分析パイプラインに変換しました。
</Card>

<Card title="検査結果を Notion へ" icon="flask" href="https://x.com/danpeguine/status/2013388700479058068">
  **@danpeguine** • `health` `notion` `organization`

数年分の血液検査結果を、構造化された Notion データベースに整理しました。
</Card>

<Card title="Obsidian セカンドブレイン" icon="book" href="https://notesbylex.com/openclaw-the-missing-piece-for-obsidians-second-brain">
  **@lexandstuff** • `obsidian` `whatsapp` `memory`

すべてのメモリをバージョン管理された Obsidian 保管庫内の Markdown として保存する、WhatsApp 上の日常利用アシスタントです。カロリーやワークアウトの記録、ToDo リスト、生活上の事務管理に対応します。
</Card>

<Card title="家族史ボット" icon="people-roof" href="https://news.ycombinator.com/item?id=47783940">
  **@brtkwr** • `telegram` `memory` `family`

家族の Telegram グループチャットに参加し、50 人以上の親族にまつわる話を記録して、内容を踏まえた追加質問を行います。ネパール語の母語話者にはネパール語で応答します。
</Card>

<Card title="WhatsApp メモリ保管庫" icon="vault">
  **コミュニティ** • `memory` `transcription` `indexing`

WhatsApp の完全なエクスポートを取り込み、1,000 件以上の音声メモを書き起こし、git ログと照合して、リンク付きの Markdown レポートを出力します。
</Card>

<Card title="Karakeep セマンティック検索" icon="magnifying-glass" href="https://github.com/jamesbrooksco/karakeep-semantic-search">
  **@jamesbrooksco** • `search` `vector` `bookmarks`

Qdrant と OpenAI または Ollama の埋め込みを使用して、Karakeep のブックマークにベクトル検索を追加します。
</Card>

<Card title="インサイド・ヘッド2 メモリ" icon="brain">
  **コミュニティ** • `memory` `beliefs` `self-model`

セッションファイルを記憶に変換し、次に信念へ、そして進化し続ける自己モデルへと変換する独立したメモリマネージャーです。
</Card>

</CardGroup>

## 音声と電話

音声を中心としたエントリーポイント、電話ブリッジ、文字起こしを多用するワークフローです。

<CardGroup cols={2}>

<Card title="Pebble Ring のワンタップ音声操作" icon="ring" href="https://x.com/thekitze/status/2014765279650189578">
  **@thekitze** • `voice` `wearable` `hardware`

Pebble Ring を 1 回タップすると OpenClaw との音声会話が始まり、ウェアラブルデバイスからエージェントにアクセスできます。
</Card>

<Card title="クリエイター向けメディアスタジオ" icon="clapperboard" href="https://x.com/cedric_chee/status/2014608153393168425">
  **@cedric_chee** • `media` `tts` `transcription`

チャット内で利用できる完全なメディアスタジオです。TTS、文字起こし、ブラウザ自動化を Codex 5.2 と MiniMax に接続しています。
</Card>

<Card title="Action Button トランシーバー" icon="walkie-talkie" href="https://x.com/i/status/2072766510053888497">
  **@buddyhadry** • `voice` `ios` `mobile`

iPhone の Action Button を OpenClaw に接続しています。押して話すと、エージェントがトランシーバーのように音声で応答します。
</Card>

<Card title="Clawdia 電話ブリッジ" icon="phone" href="https://github.com/alejandroOPI/clawdia-bridge">
  **@alejandroOPI** • `voice` `vapi` `bridge`

Vapi 音声アシスタントと OpenClaw をつなぐ HTTP ブリッジです。エージェントとほぼリアルタイムで通話できます。
</Card>

<Card title="OpenRouter 文字起こし" icon="microphone" href="https://clawhub.ai/obviyus/skills/openrouter-transcribe">
  **@obviyus** • `transcription` `multilingual` `skill`

OpenRouter（Gemini など）を使用した多言語音声の文字起こしです。ClawHub で利用できます。

  <img src="/assets/showcase/openrouter-transcribe.png" alt="ClawHub の OpenRouter 文字起こし skill" />
</Card>

</CardGroup>

## インフラストラクチャとデプロイ

OpenClaw の実行や拡張を容易にするパッケージング、デプロイ、インテグレーションです。

<CardGroup cols={2}>

<Card title="Home Assistant アドオン" icon="home" href="https://github.com/ngutman/openclaw-ha-addon">
  **@ngutman** • `homeassistant` `docker` `raspberry-pi`

SSH トンネルのサポートと永続的な状態を備え、Home Assistant OS 上で動作する OpenClaw Gateway です。
</Card>

<Card title="Home Assistant skill" icon="toggle-on" href="https://clawhub.ai/homeofe/skills/openclaw-homeassistant">
  **@homeofe** • `homeassistant` `skill` `automation`

自然言語を使用して Home Assistant デバイスを制御、自動化します。

  <img src="/assets/showcase/homeassistant.png" alt="ClawHub の Home Assistant skill" />
</Card>

<Card title="macOS メニューバーマネージャー" icon="desktop" href="https://x.com/MagiMetal/status/2009424267801485362">
  **@MagiMetal** • `macos` `swift` `ui`

エージェントの状態とクイック操作を表示する、ネイティブ Swift メニューバーアプリです。
</Card>

<Card title="Nix パッケージング" icon="snowflake" href="https://github.com/openclaw/nix-openclaw">
  **@openclaw** • `nix` `packaging` `deployment`

再現可能なデプロイに必要なものを一式備えた、Nix 化済みの OpenClaw 設定です。
</Card>

<Card title="CalDAV カレンダー" icon="calendar" href="https://clawhub.ai/asleep123/skills/caldav-calendar">
  **@asleep123** • `calendar` `caldav` `skill`

khal と vdirsyncer を使用するカレンダー skill です。セルフホスト型カレンダーとのインテグレーションを提供します。

  <img src="/assets/showcase/caldav-calendar.png" alt="ClawHub の CalDAV カレンダー skill" />
</Card>

</CardGroup>

## ホームとハードウェア

住宅、センサー、カメラ、掃除機など、OpenClaw が物理世界と関わる領域です。

<CardGroup cols={2}>

<Card title="自作 HomePod skill" icon="volume-high" href="https://x.com/localghost/status/2014763987683225685">
  **@localghost** • `homepod` `discovery` `skill`

OpenClaw がローカルネットワーク上の HomePod を見つけ、それらを制御する skill を自ら作成しました。
</Card>

<Card title="$35 のホロキューブインターフェース" icon="cube" href="https://x.com/andrewjiang/status/2013140793649734032">
  **@andrewjiang** • `hardware` `display` `fun`

机上でエージェントの物理的な顔として機能する、安価なホログラフィックキューブです。
</Card>

<Card title="GoHome オートメーション" icon="house-signal" href="https://github.com/joshp123/gohome">
  **@joshp123** • `home` `nix` `grafana`

OpenClaw をインターフェースとして使用する Nix ネイティブのホームオートメーションで、Grafana ダッシュボードも備えています。

  <img src="/assets/showcase/gohome-grafana.png" alt="GoHome Grafana ダッシュボード" />
</Card>

<Card title="Roborock 掃除機" icon="robot" href="https://github.com/joshp123/gohome/tree/main/plugins/roborock">
  **@joshp123** • `vacuum` `iot` `plugin`

自然な会話を通じて Roborock ロボット掃除機を制御します。

  <img src="/assets/showcase/roborock-screenshot.jpg" alt="Roborock の状態" />
</Card>

</CardGroup>

## コミュニティプロジェクト

単一のワークフローを超え、より幅広い製品やエコシステムへと発展したものです。

<CardGroup cols={2}>

<Card title="StarSwap マーケットプレイス" icon="star" href="https://star-swap.com/">
  **コミュニティ** • `marketplace` `astronomy` `webapp`

天文機材を網羅するマーケットプレイスです。OpenClaw エコシステムを活用し、その周辺に構築されています。
</Card>

<Card title="Clinch エージェント交渉プロトコル" icon="handshake" href="https://clawhub.ai/publicstringapps/clinch">
  **@publicstringapps** • `protocol` `p2p` `skill`

オープンなエージェント間交渉です。エージェントが他の Node と取引条件、スケジュール、サービス契約について交渉し、結果に暗号学的署名を行います。利用者は承認または拒否するだけです。
</Card>

</CardGroup>

## プロジェクトを投稿する

<Steps>
  <Step title="共有する">
    [Discord の #self-promotion](https://discord.gg/clawd) に投稿するか、[@openclaw にポスト](https://x.com/openclaw)してください。
  </Step>
  <Step title="詳細を記載する">
    何ができるのかを説明し、リポジトリまたはデモへのリンクを添え、スクリーンショットがあれば共有してください。
  </Step>
  <Step title="掲載される">
    特に優れたプロジェクトをこのページに追加します。
  </Step>
</Steps>

## 関連情報

- [はじめに](/ja-JP/start/getting-started)
- [OpenClaw](/ja-JP/start/openclaw)
- [openclaw.ai の X ショーケース一覧](https://openclaw.ai/showcase/)
