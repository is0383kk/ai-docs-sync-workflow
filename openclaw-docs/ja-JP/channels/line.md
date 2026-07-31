---
read_when:
    - OpenClaw を LINE に接続する場合
    - LINE Webhook と認証情報の設定が必要です
    - LINE 固有のメッセージオプションを使用したい場合
summary: LINE Messaging API Plugin のセットアップ、設定、使用方法
title: LINE
x-i18n:
    generated_at: "2026-07-26T10:05:06Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: aa160970278e0899637307136139f7d2fc83bf57defc30771d77649060f77274
    source_path: channels/line.md
    workflow: 16
---

LINE は LINE Messaging API を介して OpenClaw に接続します。Plugin は Gateway 上で Webhook
レシーバーとして動作し、チャネルアクセストークンとチャネルシークレットを
認証に使用します。

ステータス: 公式 Plugin、個別にインストール。ダイレクトメッセージ、グループチャット、メディア、
位置情報、Flex メッセージ、テンプレートメッセージ、クイックリプライに対応しています。
リアクションとスレッドには対応していません。

## インストール

チャネルを設定する前に LINE をインストールします。

```bash
openclaw plugins install @openclaw/line
```

ローカルチェックアウト（git リポジトリから実行する場合）:

```bash
openclaw plugins install ./path/to/local/line-plugin
```

## セットアップ

1. LINE Developers アカウントを作成し、Console を開きます。
   [https://developers.line.biz/console/](https://developers.line.biz/console/)
2. Provider を作成（または選択）し、**Messaging API** チャネルを追加します。
3. チャネル設定から **Channel access token** と **Channel secret** をコピーします。
4. Messaging API 設定で **Use webhook** を有効にします。
5. Webhook URL を Gateway エンドポイントに設定します（HTTPS が必要です）。

```text
https://gateway-host/line/webhook
```

Gateway は LINE の Webhook 検証（GET）に応答します。署名付きの受信イベント
（POST）では、`200` を返す前に各イベントを永続的な受信キューへ書き込み、
エージェント処理は非同期で続行されます。配信に失敗した場合は、Gateway の再起動後も含めて
キューから再試行され、処理不能なイベントは回数制限付きの再試行後に失敗キューの
レコードになります。永続化に失敗した場合、失われる可能性のあるイベントを受領確認する代わりに
リクエストは `500` を返します。
キューからエージェントまでの境界では、少なくとも 1 回の配信が保証されます。配信中に Gateway が
シャットダウンまたはクラッシュすると、そのターンが再実行される場合があります。メッセージイベントは
LINE メッセージ ID で重複排除され、その他のイベント種別では `webhookEventId` が使用されます。保持された完了レコードにより
通常の重複 Webhook は抑制されますが、外部への副作用を伴うハンドラーは
引き続き冪等である必要があります。
カスタムパスが必要な場合は、`channels.line.webhookPath` または
`channels.line.accounts.<id>.webhookPath` を設定し、それに応じて URL を更新します。

セキュリティ上の注意:

- LINE の署名検証は本文に依存するため（生の本文に対する HMAC）、OpenClaw は検証前に厳格な認証前本文サイズ制限（64 KB）と読み取りタイムアウトを適用します。
- OpenClaw は検証済みの生リクエストバイトから Webhook イベントを処理します。署名の完全性を保護するため、上流のミドルウェアによって変換された `req.body` 値は無視されます。

## 設定

最小構成:

```json5
{
  channels: {
    line: {
      enabled: true,
      channelAccessToken: "LINE_CHANNEL_ACCESS_TOKEN",
      channelSecret: "LINE_CHANNEL_SECRET",
      dmPolicy: "pairing",
    },
  },
}
```

公開 DM 構成:

```json5
{
  channels: {
    line: {
      enabled: true,
      channelAccessToken: "LINE_CHANNEL_ACCESS_TOKEN",
      channelSecret: "LINE_CHANNEL_SECRET",
      dmPolicy: "open",
      allowFrom: ["*"],
    },
  },
}
```

環境変数（デフォルトアカウントのみ）:

- `LINE_CHANNEL_ACCESS_TOKEN`
- `LINE_CHANNEL_SECRET`

トークン／シークレットファイル:

```json5
{
  channels: {
    line: {
      tokenFile: "/path/to/line-token.txt",
      secretFile: "/path/to/line-secret.txt",
    },
  },
}
```

`tokenFile` と `secretFile` は通常ファイルを指す必要があります。シンボリックリンクは拒否されます。
インライン設定値はファイルより優先され、環境変数はデフォルトアカウントでの最後のフォールバックです。

複数アカウント:

```json5
{
  channels: {
    line: {
      accounts: {
        marketing: {
          channelAccessToken: "...",
          channelSecret: "...",
          webhookPath: "/line/marketing",
        },
      },
    },
  },
}
```

## アクセス制御

ダイレクトメッセージのデフォルトはペアリングです。不明な送信者にはペアリングコードが発行され、
承認されるまでそのメッセージは無視されます。

```bash
openclaw pairing list line
openclaw pairing approve line <CODE>
```

許可リストとポリシー:

- `channels.line.dmPolicy`: `pairing | allowlist | open | disabled`（デフォルトは `pairing`）
- `channels.line.allowFrom`: DM で許可する LINE ユーザー ID。`dmPolicy: "open"` には `["*"]` が必要です
- `channels.line.groupPolicy`: `allowlist | open | disabled`（デフォルトは `allowlist`）
- `channels.line.groupAllowFrom`: グループで許可する LINE ユーザー ID。DM の `allowFrom` エントリではグループ送信者は許可されません
- グループごとのオーバーライド: `channels.line.groups.<groupId>.allowFrom`（および `enabled`、`requireMention`、`systemPrompt`、`skills`）。`groupPolicy: "allowlist"` の場合は、
  `groupAllowFrom` またはグループごとの `allowFrom` を設定します。DM が公開されている場合でも、グループ許可リストが空ならグループメッセージはブロックされます。
- 静的な送信者アクセスグループは、`accessGroup:<name>` を使用して `allowFrom`、`groupAllowFrom`、およびグループごとの `allowFrom` から参照できます。[アクセスグループ](/ja-JP/channels/access-groups)を参照してください。
- ランタイム上の注意: `channels.line` が完全に存在しない場合、ランタイムはグループチェックで `groupPolicy="allowlist"` にフォールバックします（`channels.defaults.groupPolicy` が設定されている場合でも）。

LINE ID では大文字と小文字が区別されます。有効な ID は次の形式です。

- ユーザー: `U` + 32 桁の 16 進文字
- グループ: `C` + 32 桁の 16 進文字
- ルーム: `R` + 32 桁の 16 進文字

## メッセージの動作

- テキストは 5000 文字単位で分割されます。
- Markdown の書式は削除されます。可能な場合、コードブロックと表は Flex
  カードに変換されます。
- ストリーミング応答はバッファリングされます。エージェントの処理中は読み込み
  アニメーションが表示され、LINE は完全なチャンクを受信します。
- メディアのダウンロードは `channels.line.mediaMaxMb`（デフォルトは 10）によって制限されます。
- 受信メディアは、エージェントに渡される前に `~/.openclaw/media/inbound/` に保存されます。
  これは他のチャネル Plugin が使用する共有メディアストアと同じです。

## チャネルデータ（リッチメッセージ）

クイックリプライ、位置情報、Flex カード、またはテンプレート
メッセージを送信するには `channelData.line` を使用します。

```json5
{
  text: "Here you go",
  channelData: {
    line: {
      quickReplies: ["Status", "Help"],
      location: {
        title: "Office",
        address: "123 Main St",
        latitude: 35.681236,
        longitude: 139.767125,
      },
      flexMessage: {
        altText: "Status card",
        contents: {/* Flex payload */},
      },
      templateMessage: {
        type: "confirm",
        text: "Proceed?",
        confirmLabel: "Yes",
        confirmData: "yes",
        cancelLabel: "No",
        cancelData: "no",
      },
    },
  },
}
```

LINE Plugin には、Flex メッセージのプリセット用の `/card` コマンドも含まれています。

```text
/card info "ようこそ" "ご参加ありがとうございます！"
```

## ACP 対応

LINE は ACP（Agent Communication Protocol）の会話バインディングに対応しています。

- `/acp spawn <agent> --bind here` は、子スレッドを作成せずに現在の LINE チャットを ACP セッションにバインドします。
- 設定済みの ACP バインディングと、会話にバインドされたアクティブな ACP セッションは、他の会話チャネルと同様に LINE でも動作します。

詳細は [ACP エージェント](/ja-JP/tools/acp-agents)を参照してください。

## 送信メディア

LINE Plugin は、エージェントメッセージツールを介して画像、動画、音声を送信します。

- **画像**: LINE の画像メッセージとして送信されます。プレビュー画像のデフォルトはメディア URL です。
- **動画**: プレビュー画像が必要です。`channelData.line.previewImageUrl` に画像 URL を設定します。
- **音声**: LINE の音声メッセージとして送信されます。`channelData.line.durationMs` が設定されていない場合、再生時間のデフォルトは 60 秒です。

メディア種別は、`channelData.line.mediaKind` が設定されている場合はそこから取得され、それ以外の場合は
他の LINE オプションまたは URL のファイル拡張子から推測されます。フォールバックは画像です。

送信メディア URL は、2000 文字以内の公開 HTTPS URL である必要があります。OpenClaw は
URL を LINE に渡す前に宛先ホスト名を検証し、loopback、リンクローカル、および
プライベートネットワークの宛先を拒否します。

LINE 固有のオプションを指定しない汎用メディア送信では、画像ルートが使用されます。

## トラブルシューティング

- **Webhook の検証に失敗する:** Webhook URL が HTTPS であり、
  `channelSecret` が LINE Console と一致していることを確認します。
- **受信イベントがない:** Webhook パスが `channels.line.webhookPath` と一致し、
  LINE から Gateway に到達できることを確認します。
- **メディアのダウンロードエラー:** メディアがデフォルトの制限を超える場合は、
  `channels.line.mediaMaxMb` を増やします。

## 関連項目

- [チャネルの概要](/ja-JP/channels) — 対応しているすべてのチャネル
- [ペアリング](/ja-JP/channels/pairing) — DM の認証とペアリングの流れ
- [グループ](/ja-JP/channels/groups) — グループチャットの動作とメンションによる制御
- [チャネルルーティング](/ja-JP/channels/channel-routing) — メッセージのセッションルーティング
- [セキュリティ](/ja-JP/gateway/security) — アクセスモデルと堅牢化
