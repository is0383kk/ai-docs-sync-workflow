---
read_when:
    - OpenClaw エージェントを Microsoft Teams 会議に参加させたい場合
    - Teams 会議へのトークバック用に Chrome、BlackHole、または SoX を設定しています
summary: Microsoft Teams 会議 Plugin：Chrome ブラウザのゲストとして職場または個人向けの会議に参加する
title: Microsoft Teams 会議 Plugin
x-i18n:
    generated_at: "2026-07-26T09:14:59Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 6f84e58d478185d026dd79a02a8500af48f51689ef6865d56badb0e27c6d2814
    source_path: plugins/teams-meetings.md
    workflow: 16
---

`teams-meetings` Plugin は、OpenClaw Chrome プロファイルでゲストとして Microsoft Teams のリンクに参加します。`teams.microsoft.com/l/meetup-join/...` 配下の職場向けリンクと、`teams.live.com/meet/...` 配下の個人向けリンクを受け付けます。会議の作成、電話での参加、Microsoft Graph の呼び出し、音声または動画の録画は行いません。

## セットアップ

音声応答では、[Google Meet Plugin](/ja-JP/plugins/google-meet)と同じローカル音声の前提条件（macOS、`BlackHole 2ch` 仮想オーディオデバイス、SoX）を使用します。

```bash
brew install blackhole-2ch sox
sudo reboot
system_profiler SPAudioDataType | grep -i BlackHole
command -v sox
```

Plugin は同梱され、デフォルトで有効になっています。カスタマイズする場合のみエントリを追加し、その後セットアップを確認します。

```json5
{
  plugins: {
    entries: {
      "teams-meetings": {
        config: {
          defaultMode: "agent",
          chrome: { guestName: "OpenClaw Agent" },
        },
      },
    },
  },
}
```

Plugin を有効にしない場合は、`openclaw plugins disable teams-meetings` を実行します。

```bash
openclaw teamsmeetings setup
openclaw teamsmeetings join 'https://teams.microsoft.com/l/meetup-join/...'
```

ペアリング済みの macOS Node で Chrome、BlackHole、SoX を実行するには、`chromeNode.node` を使用します。Node では `teamsmeetings.chrome` と `browser.proxy` を許可する必要があります。

## モード

| モード         | 動作                                                                    |
| ------------ | --------------------------------------------------------------------------- |
| `agent`      | リアルタイム文字起こしが設定済みの OpenClaw エージェントに問い合わせ、TTS で応答します。 |
| `bidi`       | リアルタイム音声モデルが直接聞き取り、応答します。                        |
| `transcribe` | ライブキャプションの文字起こしスナップショットを取得する、監視専用の参加です。                   |

OpenClaw が話者情報付きのメモを永続化できるよう、すべてのモードで参加承認後に Teams のライブキャプションが有効になります。`transcript` アクションは、`transcribe` セッションに限り、引き続き上限付きライブバッファのみを返します。退出時に、OpenClaw は永続的な文字起こしと生成された要約を共有状態データベースに保存します。[`openclaw transcripts`](/ja-JP/cli/transcripts)を使用して一覧表示またはエクスポートできます。

自動メモはデフォルトで有効です。永続的なメモをグローバルに無効にするには、`transcripts.enabled: false` を設定します。明示的な `transcribe` モードでは、引き続き上限付きのライブ末尾部分のみが公開されます。

## ゲスト参加の制限

ブラウザアダプターは、アプリへの誘導画面を閉じ、ゲスト名を入力し、カメラをオフにして、選択したモード用にマイクを設定し、参加ボタンをクリックします。通話中の状態では通話終了コントロールを使用します。ロビー、テナントへのサインイン、デバイス権限の状態では、必要な手動操作の理由を明示的に返します。個人向け会議ランチャーのリダイレクトと、Chrome に表示される `BlackHole 2ch (Virtual)` ラベルに対応しています。

Teams のテナントポリシーにより、サインイン、メール認証、または開催者による参加承認が必要になる場合があります。OpenClaw Chrome プロファイルでその手順を完了してから、ステータスまたは音声操作を再試行してください。Plugin はテナントポリシーを回避しません。

個人向け Teams Web クライアントでは、アプリへの誘導画面、ゲスト名の入力、参加前のマイクとカメラの切り替え、参加、ロビーでの承認、メディア権限、通話中の検出、ライブキャプション、BlackHole の入出力ルーティング、退出、通話後の検出について実環境で検証済みです。職場のテナントでは、サインイン、メール認証、参加承認、退出確認に異なるポリシーが適用される場合があります。報告された手動操作は、OpenClaw Chrome プロファイルで完了してください。

## ツールと Gateway のインターフェース

`teams_meetings` エージェントツールは、`join`、`leave`、`status`、`transcript`、`speak` をサポートします。Gateway メソッドでは `teamsmeetings.*` プレフィックスを使用します。Node コマンドは `teamsmeetings.chrome` です。

## 関連項目

- [会議 Plugin の概要](/plugins/meeting-plugins)
- [Microsoft Teams チャネル](/ja-JP/channels/msteams)
