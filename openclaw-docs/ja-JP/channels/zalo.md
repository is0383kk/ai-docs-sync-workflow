---
read_when:
    - Zalo の機能または Webhook に取り組む
summary: Zalo bot のサポート状況、機能、設定
title: Zalo
x-i18n:
    generated_at: "2026-07-26T09:32:07Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: f3e0bfe6003d3b2f38411fcc5a4e82266733b042693c7853d0b3c8a3864273c5
    source_path: channels/zalo.md
    workflow: 16
---

ステータス: 実験的。ダイレクトメッセージとグループチャットはどちらも実装済みです。以下の[機能](#capabilities)表は、Zalo Bot Creator / Marketplace ボットで検証済みの動作を示しています。

## バンドル済み Plugin

現在の OpenClaw リリースでは、Zalo はバンドル済み Plugin として提供されるため、パッケージビルドで個別にインストールする必要はありません。

古いビルド、または Zalo を除外したカスタムインストールでは、npm パッケージを直接インストールします。

- インストール: `openclaw plugins install @openclaw/zalo`
- 固定バージョン: `openclaw plugins install @openclaw/zalo@2026.6.11`
- ローカルチェックアウトから: `openclaw plugins install ./path/to/local/zalo-plugin`
- 詳細: [Plugin](/ja-JP/tools/plugin)

## クイックセットアップ

1. [https://bot.zaloplatforms.com](https://bot.zaloplatforms.com) でボットトークンを作成します（サインインし、ボットを作成して設定を構成します）。トークンは `numeric_id:secret` です。Marketplace ボットでは、使用可能なランタイムトークンがボットのウェルカムメッセージに表示される場合があります。
2. デフォルトアカウントのみを対象とする環境変数 `ZALO_BOT_TOKEN=...`、または設定でトークンを指定します。
3. Gateway を再起動します。
4. 最初の DM 受信時にペアリングコードを承認します（デフォルトの DM ポリシーはペアリングです）。

最小設定:

```json5
{
  channels: {
    zalo: {
      enabled: true,
      accounts: {
        default: {
          botToken: "12345689:abc-xyz",
          dmPolicy: "pairing",
        },
      },
    },
  },
}
```

複数アカウント: `channels.zalo.accounts.<id>` の下にエントリを追加し、それぞれに固有の `botToken`/`name` を指定します。`channels.zalo.botToken`（フラット形式で `accounts` なし）は従来の単一アカウント用省略記法です。新しい設定には `accounts.<id>.*` を推奨します。

## 概要

Zalo はベトナム市場を中心とするメッセージングアプリです。その Bot API により、Gateway は 1:1 会話とグループチャットの両方でボットを実行でき、応答は決定論的に Zalo へルーティングされます（モデルがチャネルを選択することはありません）。

このページでは **Zalo Bot Creator / Marketplace ボット**を扱います。**Zalo Official Account (OA) ボット**は別の製品機能であり、動作が異なる場合があります。このページでは扱いません。

## 動作の仕組み

- 受信メッセージは、メディアプレースホルダーを含む共通チャネルエンベロープに正規化されます。
- 返信は常に同じ Zalo チャットへルーティングされます。引用返信は使用されません（`replyToMode` は常に無効です）。
- デフォルトではロングポーリング（`getUpdates`）を使用します。`channels.zalo.webhookUrl` で Webhook モードも利用できます。
- グループでボットを起動するには @メンションが必要です。チャネル単位では設定できません。

## 制限

| 制限                          | 値                                                                       |
| ----------------------------- | ------------------------------------------------------------------------ |
| 送信テキストのチャンクサイズ  | 2000 文字（Zalo API の制限）                                             |
| メディアサイズ（受信/送信）   | `channels.zalo.mediaMaxMb`、デフォルト `5` MB                     |
| Webhook リクエスト本文         | 1 MB、読み取りタイムアウト 30s                                           |
| Webhook レート制限             | パスとクライアント IP ごとに 60s あたり 120 リクエスト、その後 HTTP 429 |
| Webhook リプレイトゥームストーン | 30 日間、アカウントごとに完了イベント最大 20,000 件（メッセージ ID をキーとする） |

## アクセス制御

### ダイレクトメッセージ

- `channels.zalo.dmPolicy`: `pairing`（デフォルト）| `allowlist` | `open` | `disabled`。
- ペアリング: 未知の送信者にはペアリングコードが発行され、承認されるまでメッセージは無視されます。コードは 1 時間後に期限切れになります。
  - `openclaw pairing list zalo`
  - `openclaw pairing approve zalo <CODE>`
  - 詳細: [ペアリング](/ja-JP/channels/pairing)
- `channels.zalo.allowFrom` は数値の Zalo ユーザー ID を受け付けます（ユーザー名検索はありません）。`open` には `"*"` が必要です。

### グループ

グループチャットは Plugin（`chatTypes: ["direct", "group"]`）でサポートされ、メンションとグループポリシーによって制御されます。

- `channels.zalo.groupPolicy`: `open` | `allowlist` | `disabled`。
- `channels.zalo.groupAllowFrom` は、グループ内でボットを起動できる送信者 ID を制限します。未設定の場合は `allowFrom` にフォールバックします。
- デフォルトの解決: `channels.zalo` が設定されている場合、未設定の `groupPolicy` は `open` として解決されます。`channels.zalo` 自体が存在しない場合、ランタイムはフェイルクローズして `allowlist` になります。
- 実環境で報告されている注意点: 一部の Marketplace ボット設定では、ボットをグループにまったく追加できないことがあります。この問題が発生した場合は、使用しているボットの Zalo Bot Platform 設定を確認してください。これはプラットフォーム側の制約であり、OpenClaw のポリシーではありません。

## ロングポーリングと Webhook

- デフォルト: ロングポーリング（公開 URL は不要）。
- Webhook モード: `channels.zalo.webhookUrl` と `channels.zalo.webhookSecret` を設定します。
  - Webhook URL には HTTPS を使用する必要があります。
  - Webhook シークレットは 8-256 文字である必要があります。
  - Zalo は `X-Bot-Api-Secret-Token` ヘッダー付きでイベントを送信し、定数時間比較で検証されます。
  - Gateway HTTP は `channels.zalo.webhookPath` で Webhook リクエストを処理します（デフォルトは Webhook URL のパス）。
  - リクエストでは `Content-Type: application/json`（または `+json` メディアタイプ）を使用する必要があります。
  - 生のイベントが永続ストレージに保存された後にのみ HTTP 200 が返されます。保存に失敗した場合は HTTP 500 が返されます。
  - Zalo API ドキュメントでは、getUpdates ポーリングと Webhook はアカウントごとに排他的です。

## サポートされるメッセージタイプ

- テキスト: 完全にサポートされ、2000 文字単位に分割されます。
- メディア: 受信/送信に対応し、`mediaMaxMb` によって上限が設定されます。
- リアクション、スレッド、投票、ネイティブコマンド: Plugin ではサポートされません。
- ストリーミング: Plugin はブロックストリーミング機能を宣言していますが、Zalo には専用の送信キューやテキスト結合の調整オプションがありません（一部の他の地域向けチャネルとは異なります）。ユースケースで重要な場合は、使用環境で現在の動作を確認してください。

## 機能

| 機能                     | ステータス                        |
| ------------------------ | --------------------------------- |
| ダイレクトメッセージ     | サポート                          |
| グループ                 | サポート（メンション必須）        |
| メディア（受信/送信）    | サポート、上限は `mediaMaxMb` |
| リアクション             | 未サポート                        |
| スレッド                 | 未サポート                        |
| 投票                     | 未サポート                        |
| ネイティブコマンド       | 未サポート                        |
| 返信先 / 引用            | 未使用（常に無効）                |

## 配信先（CLI/Cron）

チャット ID を配信先として使用します。

```bash
openclaw message send --channel zalo --target 123456789 --message "hi"
```

## トラブルシューティング

**ボットが応答しない場合:**

- トークンを確認します: `openclaw channels status --probe`
- 送信者が承認済みであることを確認します（ペアリングまたは `allowFrom`）
- Gateway のログを確認します: `openclaw logs --follow`

**Webhook がイベントを受信しない場合:**

- Webhook URL が HTTPS を使用していることを確認します
- シークレットが 8-256 文字であることを確認します
- Gateway HTTP エンドポイントが設定されたパスで到達可能であることを確認します
- getUpdates ポーリングも同時に実行されていないことを確認します（両者は排他的です）
- リクエストが集中すると HTTP 429 が返される場合があります（パスと IP ごとに 60s あたり 120 リクエスト）。間隔を空けて再試行してください

## 設定リファレンス

完全な設定: [設定](/ja-JP/gateway/configuration)

| 設定                                         | 説明                                              | デフォルト            |
| -------------------------------------------- | ------------------------------------------------- | --------------------- |
| `channels.zalo.enabled`                           | チャネルの起動を有効化/無効化                     | `true`    |
| `channels.zalo.accounts.<id>.botToken`                           | Zalo Bot Platform のボットトークン                | -                     |
| `channels.zalo.accounts.<id>.tokenFile`                           | ファイルからトークンを読み取る（シンボリックリンクは拒否） | -                     |
| `channels.zalo.accounts.<id>.name`                           | 表示名                                            | -                     |
| `channels.zalo.accounts.<id>.enabled`                           | このアカウントを有効化/無効化                     | `true`    |
| `channels.zalo.accounts.<id>.dmPolicy`                           | アカウント単位の DM ポリシー                      | `pairing`    |
| `channels.zalo.accounts.<id>.allowFrom`                           | DM 許可リスト（ユーザー ID）                      | -                     |
| `channels.zalo.accounts.<id>.groupPolicy`                           | アカウント単位のグループポリシー                  | [グループ](#groups)を参照 |
| `channels.zalo.accounts.<id>.groupAllowFrom`                           | グループ送信者の許可リスト。`allowFrom` にフォールバック | -                     |
| `channels.zalo.accounts.<id>.mediaMaxMb`                           | 受信/送信メディアの上限（MB）                     | `5`    |
| `channels.zalo.accounts.<id>.webhookUrl`                           | Webhook モードを有効化（HTTPS 必須）              | -                     |
| `channels.zalo.accounts.<id>.webhookSecret`                           | Webhook シークレット（8-256 文字）                | -                     |
| `channels.zalo.accounts.<id>.webhookPath`                           | Gateway HTTP サーバー上の Webhook パス            | Webhook URL のパス    |
| `channels.zalo.accounts.<id>.proxy`                           | API リクエスト用のプロキシ URL                    | -                     |
| `channels.zalo.accounts.<id>.responsePrefix`                           | 送信応答プレフィックスの上書き                    | -                     |
| `channels.zalo.defaultAccount`                           | 複数設定時のデフォルトアカウント                  | `default`    |

`channels.zalo.botToken`、`channels.zalo.dmPolicy`、およびその他のフラットなトップレベルキーは、上記フィールドに対する従来の単一アカウント用省略記法です。どちらの形式もサポートされています。

環境変数オプション: `ZALO_BOT_TOKEN=...` はデフォルトアカウントのトークンにのみ解決されます。

## 関連項目

- [チャネル概要](/ja-JP/channels) - サポートされるすべてのチャネル
- [ペアリング](/ja-JP/channels/pairing) - DM の認証とペアリングの流れ
- [グループ](/ja-JP/channels/groups) - グループチャットの動作とメンションによる制御
- [チャネルルーティング](/ja-JP/channels/channel-routing) - メッセージのセッションルーティング
- [セキュリティ](/ja-JP/gateway/security) - アクセスモデルと強化
