---
read_when:
    - Mac アプリのヘルスインジケーターのデバッグ
summary: macOS アプリが Gateway／チャンネルの正常性状態を報告する仕組み
title: ヘルスチェック（macOS）
x-i18n:
    generated_at: "2026-07-26T09:08:56Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 095abdbefa7db7c0d14435e2c5db7d1ebc03afa0c539555a7abdd9170d015fb8
    source_path: platforms/mac/health.md
    workflow: 16
---

# macOS でのヘルスチェック

メニューバーアプリからリンク済みチャンネルのヘルス状態を確認する方法です。

## メニューバー

ステータスドット：

- 緑：リンク済みでプローブは正常です。
- オレンジ：リンク済みですが、チャンネルプローブから機能低下または未接続が報告されています。
- 赤：まだリンクされていません。

2 行目には「リンク済み · 認証 12 分前」と表示されるか、失敗理由が表示されます。
メニューの「Run Health Check Now」を選択すると、オンデマンドのプローブが実行されます。

## 設定

- General タブにはヘルスカードが表示されます。ステータスドット、概要行（リンク状態 +
  認証からの経過時間）、任意の失敗詳細行があり、**Retry now** ボタンと
  **Open logs** ボタンも表示されます。
- **Channels タブ**には、WhatsApp と Telegram のチャンネルごとの状態と操作（ログイン用 QR、
  ログアウト、プローブ、最後の切断またはエラー）が表示されます。

## プローブの仕組み

アプリは、既存の WebSocket 接続（CLI の外部プロセス呼び出しではありません）を介して Gateway の `health` RPC を
約 60 秒ごと、およびオンデマンドで呼び出します。この RPC は
認証情報を読み込み、メッセージを送信せずに状態を報告します。アプリは最後に正常だった
スナップショットと最後のエラーを個別にキャッシュするため、UI は即座に読み込まれ、
オフライン中もちらつきません。

## 判断に迷う場合

[Gateway のヘルス](/ja-JP/gateway/health) に記載された CLI の手順（`openclaw status`、
`openclaw status --deep`、`openclaw health --json`）を使用し、
`openclaw logs --follow` を実行して、`web-heartbeat` / `web-reconnect` で絞り込みます。

## 関連項目

- [Gateway のヘルス](/ja-JP/gateway/health)
- [macOS アプリ](/ja-JP/platforms/macos)
