---
read_when:
    - OpenClaw エージェントを Zoom ミーティングに参加させたい場合
    - Zoom ミーティングでのトークバック用に Chrome、BlackHole、または SoX を設定している場合
summary: Zoom ミーティング Plugin：Chrome ブラウザのゲストとしてミーティングに参加する
title: Zoom ミーティング Plugin
x-i18n:
    generated_at: "2026-07-26T09:15:23Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: d91e57cccb163f634c6eaee71dd3832fc7b9e783fc5cd02601572b302d0d25e8
    source_path: plugins/zoom-meetings.md
    workflow: 16
---

`zoom-meetings` Plugin は、OpenClaw Chrome プロファイルの Zoom Web App を介して、ゲストとして Zoom ミーティングリンクに参加します。`zoom.us/j/...` 配下のミーティングリンクと、`example.zoom.us/j/...` などのアカウントサブドメインを受け付けます。ミーティングの作成、電話での参加、Zoom Meeting SDK の使用、音声・映像録画のキャプチャは行いません。

## セットアップ

音声応答には、[Google Meet Plugin](/ja-JP/plugins/google-meet) と同じローカル音声の前提条件（macOS、`BlackHole 2ch` 仮想オーディオデバイス、SoX）が必要です。

```bash
brew install blackhole-2ch sox
sudo reboot
system_profiler SPAudioDataType | grep -i BlackHole
command -v sox
```

Plugin は同梱されており、デフォルトで有効です。カスタマイズする場合にのみエントリを追加し、その後セットアップを確認します。

```json5
{
  plugins: {
    entries: {
      "zoom-meetings": {
        config: {
          defaultMode: "agent",
          chrome: { guestName: "OpenClaw Agent" },
        },
      },
    },
  },
}
```

Plugin を有効にしたくない場合は、`openclaw plugins disable zoom-meetings` を実行します。

```bash
openclaw zoommeetings setup
openclaw zoommeetings join 'https://zoom.us/j/1234567890'
```

ペアリング済みの macOS Node で Chrome、BlackHole、SoX を実行するには、`chromeNode.node` を使用します。Node では `zoommeetings.chrome` と `browser.proxy` を許可する必要があります。

## モード

| モード         | 動作                                                                    |
| ------------ | --------------------------------------------------------------------------- |
| `agent`      | リアルタイム文字起こしが設定済みの OpenClaw エージェントに問い合わせ、TTS で応答します。 |
| `bidi`       | リアルタイム音声モデルが直接聞き取り、応答します。                        |
| `transcribe` | ライブキャプションの文字起こしスナップショットを取得する、観察専用の参加です。                   |

OpenClaw がミーティングノートを永続化できるように、すべてのモードで入室許可後に Zoom のライブキャプションが有効になります。`transcript` アクションが返すのは、引き続き `transcribe` セッションの上限付きライブバッファのみです。退出時に、OpenClaw は永続的な文字起こしと、それから生成した要約を共有状態データベースに保存します。[`openclaw transcripts`](/ja-JP/cli/transcripts) を使用して一覧表示またはエクスポートできます。

自動ノートはデフォルトで有効です。永続的なノートをグローバルに無効にするには、`transcripts.enabled: false` を設定します。明示的な `transcribe` モードでは、引き続き上限付きのライブ末尾部分のみが公開されます。

## ゲスト参加の制限

ブラウザアダプターは **Join from browser** を選択し、ゲスト名を入力し、カメラをオフにし、選択したモードに合わせてマイクを設定して、**Join** をクリックします。Zoom Web App は `app.zoom.us` で動作します。Plugin はナビゲーションの前に、そのオリジンへマイクとスピーカー選択の権限を付与します。通話中の状態では、Zoom の Leave コントロールを使用します。ロビー、サインイン、パスコード、CAPTCHA、デバイス権限の各状態では、手動操作が必要な理由を明示的に返します。

Zoom のホストおよびアカウントポリシーにより、ブラウザからの参加が無効になっている場合、認証やメール確認が必要な場合、CAPTCHA が表示される場合、またはホストによる入室許可が必要な場合があります。OpenClaw Chrome プロファイルでその手順を完了してから、ステータス確認または音声送信を再試行してください。Plugin は Zoom のポリシーを回避しません。

Zoom Web App については、公式の Zoom テストミーティングを使用し、アプリの中間画面、iframe でのゲスト名入力、参加前のマイクとカメラのコントロール、参加、ブラウザと macOS のメディア権限、通話中の検出、ライブキャプションの有効化、ホストによる終了の検出を実環境で検証済みです。ロビーと認証の状態はホストのポリシーに依存し、安定した DOM 識別子が利用できない場合はテキストによるフォールバックを維持します。

## ツールと Gateway のサーフェス

`zoom_meetings` エージェントツールは、`join`、`leave`、`status`、`transcript`、`speak` をサポートします。Gateway メソッドは `zoommeetings.*` プレフィックスを使用します。Node コマンドは `zoommeetings.chrome` です。

## 関連項目

- [ミーティング Plugin の概要](/plugins/meeting-plugins)
