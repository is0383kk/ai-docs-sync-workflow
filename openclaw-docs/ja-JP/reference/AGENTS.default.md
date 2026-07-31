---
read_when:
    - 新しい OpenClaw エージェントセッションの開始
    - デフォルトの Skills の有効化または監査
summary: パーソナルアシスタント設定向けのデフォルトの OpenClaw エージェント指示と Skills 一覧
title: デフォルトの AGENTS.md
x-i18n:
    generated_at: "2026-07-26T09:49:54Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 645342f8c6e2805135817cf4bbc2c8bd1d57066054ed671eda93876b2762ffb1
    source_path: reference/AGENTS.default.md
    workflow: 16
---

## 初回実行（推奨）

OpenClaw エージェントはワークスペースディレクトリを使用します。デフォルト: `~/.openclaw/workspace`（`agents.defaults.workspace` で設定可能、`~` をサポート）。

1. ワークスペースを作成します。

```bash
mkdir -p ~/.openclaw/workspace
```

2. デフォルトのワークスペーステンプレートをそこにコピーします。

```bash
cp docs/reference/templates/AGENTS.md ~/.openclaw/workspace/AGENTS.md
cp docs/reference/templates/SOUL.md ~/.openclaw/workspace/SOUL.md
cp docs/reference/templates/TOOLS.md ~/.openclaw/workspace/TOOLS.md
```

3. 任意: 汎用テンプレートの代わりに、このファイルのパーソナルアシスタント向けスキル一覧を使用します。

```bash
cp docs/reference/AGENTS.default.md ~/.openclaw/workspace/AGENTS.md
```

4. 任意: 別のワークスペースを指定します。

```json5
{
  agents: { defaults: { workspace: "~/.openclaw/workspace" } },
}
```

## 安全性のデフォルト

- ディレクトリの内容やシークレットをチャットに出力しないでください。
- 明示的に依頼されない限り、破壊的なコマンドを実行しないでください。
- 設定やスケジューラー（crontab、systemd ユニット、nginx 設定、シェルの rc ファイル）を変更する前に、まず既存の状態を確認し、デフォルトでは内容を保持またはマージしてください。
- 外部メッセージングサーフェスには、部分的な応答やストリーミング応答を送信しないでください（最終応答のみ）。

## 既存ソリューションの事前確認

カスタムのシステム、機能、ワークフロー、ツール、統合、または自動化を提案または構築する前に、十分に要件を満たすオープンソースプロジェクト、保守されているライブラリ、既存の OpenClaw plugins、または無料プラットフォームがないか確認してください。適切なものがあれば優先して使用してください。既存の選択肢が不適切、高額、保守されていない、安全でない、非準拠である場合、またはユーザーが明示的にカスタム実装を求めた場合にのみ、カスタムで構築してください。ユーザーが支出を明示的に承認しない限り、有料サービスの推奨は避けてください。これは調査タスクではなく、軽量な事前確認ゲートとして行ってください。

## セッション開始（必須）

- 応答する前に、`SOUL.md`、`USER.md`、および `memory/` 内の今日と昨日の内容を読んでください。
- `MEMORY.md` が存在する場合は読んでください。

## Soul（必須）

- `SOUL.md` は、アイデンティティ、トーン、境界を定義します。常に最新の状態に保ってください。
- `SOUL.md` を変更した場合は、ユーザーに伝えてください。
- 各セッションでは新しいインスタンスとして動作し、継続性はこれらのファイルに保持されます。

## 共有スペース（推奨）

- ユーザー本人として発言するものではありません。グループチャットや公開チャンネルでは慎重に対応してください。
- 個人データ、連絡先情報、内部メモを共有しないでください。

## メモリシステム（推奨）

- 日次ログ: `memory/YYYY-MM-DD.md`（必要に応じて `memory/` を作成）。
- 長期メモリ: 永続的な事実、設定、決定事項を保存する `MEMORY.md`。
- 小文字の `memory.md` は、レガシー修復用の入力としてのみ使用します。意図的に両方のルートファイルを保持しないでください。
- セッション開始時に、今日、昨日、および存在する場合は `MEMORY.md` を読んでください。
- メモリファイルに書き込む前に、まずその内容を読んでください。具体的な更新のみを書き込み、空のプレースホルダーは決して書き込まないでください。
- 記録対象: 決定事項、設定、制約、未完了事項。
- 明示的に要求されない限り、シークレットは記録しないでください。

## ツールと Skills

- ツールは Skills 内にあります。必要な場合は、各 Skills の `SKILL.md` に従ってください。
- 環境固有のメモは `TOOLS.md`（Skills 用メモ）に保存してください。

## バックアップのヒント（推奨）

このワークスペースをアシスタントのメモリとして扱い、`AGENTS.md` とメモリファイルがバックアップされるように git リポジトリ（理想的には非公開）にしてください。

```bash
cd ~/.openclaw/workspace
git init
git add AGENTS.md
git commit -m "Add workspace"
# 任意: 非公開リモートを追加してプッシュ
```

## OpenClaw の機能

- メッセージングチャンネル Gateway（WhatsApp、Telegram、Discord、Signal、iMessage、Slack など）と組み込みエージェントを実行し、アシスタントがチャットの読み書き、コンテキストの取得、ホストマシン経由での Skills の実行を行えるようにします。
- macOS アプリは権限（画面収録、通知、マイク）を管理し、同梱バイナリを通じて `openclaw` CLI を公開します。
- デフォルトでは、ダイレクトチャットはエージェントの `main` セッションに統合され、グループとチャンネル／ルームにはそれぞれ独自のセッションキーが割り当てられます。正確なキー形式については、[チャンネルルーティング](/ja-JP/channels/channel-routing)を参照してください。Heartbeat はバックグラウンドタスクを継続させます。

## コア Skills（Settings → Skills で有効化）

パーソナルアシスタント用ワークスペースの一覧例です。環境に合う Skills に置き換えてください。

- **mcporter** - 外部スキルバックエンドを管理するためのツールサーバーランタイム／CLI。
- **Peekaboo** - オプションの AI ビジョン分析を備えた高速な macOS スクリーンショット。
- **camsnap** - RTSP/ONVIF セキュリティカメラからフレーム、クリップ、またはモーションアラートを取得。
- **oracle** - セッション再生とブラウザ制御を備えた OpenAI 対応エージェント CLI。
- **eightctl** - ターミナルから睡眠を管理。
- **imsg** - iMessage と SMS の送信、読み取り、ストリーミング。
- **wacli** - WhatsApp CLI: 同期、検索、送信。
- **discord** - Discord の操作: リアクション、ステッカー、投票。`user:<id>` または `channel:<id>` ターゲットを使用してください（数字のみの ID は曖昧です）。
- **gog** - Google Suite CLI: Gmail、Calendar、Drive、Contacts。
- **spotify-player** - 再生の検索、キューへの追加、操作を行うターミナル版 Spotify クライアント。
- **sag** - macOS の say に似た UX を備えた ElevenLabs 音声。デフォルトではスピーカーにストリーミングします。
- **Sonos CLI** - スクリプトから Sonos スピーカー（検出／ステータス／再生／音量／グループ化）を操作。
- **blucli** - スクリプトから BluOS プレーヤーの再生、グループ化、自動化を実行。
- **OpenHue CLI** - シーンと自動化のための Philips Hue 照明制御。
- **OpenAI Whisper** - 素早い音声入力とボイスメール文字起こしのためのローカル音声テキスト変換。
- **Gemini CLI** - ターミナルから Google Gemini モデルを使用して迅速に質疑応答。
- **agent-tools** - 自動化とヘルパースクリプト向けのユーティリティツールキット。

## 使用上の注意

- スクリプト作成には `openclaw` CLI を優先してください。デスクトップアプリが権限を処理します。
- インストールは Skills タブから実行してください。必要なバイナリがすでに存在する場合、インストールボタンは非表示になります。
- アシスタントがリマインダーのスケジュール設定、受信トレイの監視、カメラ撮影のトリガーを実行できるように、Heartbeat を有効にしておいてください。
- Canvas UI はネイティブオーバーレイ付きの全画面表示で動作します。重要なコントロールを左上、右上、下端に配置しないでください。セーフエリアのインセットに依存せず、明示的なレイアウト余白を追加してください。
- ブラウザ駆動の検証には、OpenClaw が管理する Chrome/Brave/Edge/Chromium プロファイルで `openclaw browser` CLI（同梱の `browser` Plugin）を使用してください。
- 管理: `status`、`doctor [--deep]`、`start [--headless]`、`stop`、`tabs`、`tab [new|select|close]`、`open <url>`、`focus <id>`、`close <id>`。
- 検査: `screenshot [--full-page|--ref|--labels]`、`snapshot [--format ai|aria|--interactive|--efficient]`、`console`、`errors`、`requests`、`pdf`、`responsebody`。
- 操作: `navigate`、`click <ref>`、`type <ref> <text>`、`press`、`hover`、`drag`、`select`、`upload`、`download`、`fill`、`dialog`、`wait`、`evaluate --fn <js>`、`highlight`。操作には `snapshot` から取得した `ref` が必要です（操作では CSS セレクターを使用できません）。`document.querySelector` 形式のターゲット指定が必要な場合は、`evaluate` を使用してください。
- 任意の検査コマンドで機械可読出力を得るには、`--json` を追加してください。

## 関連項目

- [エージェントワークスペース](/ja-JP/concepts/agent-workspace)
- [エージェントランタイム](/ja-JP/concepts/agent)
- [チャンネルルーティング](/ja-JP/channels/channel-routing)
