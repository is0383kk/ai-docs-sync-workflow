---
read_when:
    - Yuanbao ボットに接続したい場合
    - Yuanbao チャンネルを設定しています
summary: Yuanbao bot の概要、機能、設定
title: Yuanbao
x-i18n:
    generated_at: "2026-07-26T09:27:32Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 43488834f588530206b290cb0fb185fd1fe2e1f214ab4a4ccccc49b9b549b6ac
    source_path: channels/yuanbao.md
    workflow: 16
---

Tencent Yuanbao は Tencent の AI アシスタントプラットフォームです。コミュニティによってメンテナンスされている `openclaw-plugin-yuanbao` Plugin は、Yuanbao ボットを WebSocket 経由で OpenClaw に接続し、ダイレクトメッセージとグループチャットを利用できるようにします。

**ステータス:** ボットの DM とグループチャットで本番利用可能です。サポートされる接続モードは WebSocket のみです。この Plugin は OpenClaw コアではなく Tencent Yuanbao チームによって外部カタログエントリとしてメンテナンスされています。以下の設定と動作の詳細（インストールと汎用 CLI サーフェスを除く）は Plugin 独自のドキュメントに基づいており、OpenClaw コアのソースに対する検証は行われていません。

## クイックスタート

OpenClaw 2026.4.10 以降が必要です。`openclaw --version` で確認し、`openclaw update` でアップグレードしてください。

<Steps>
  <Step title="認証情報を使用して Yuanbao チャンネルを追加する">
  ```bash
  openclaw channels add --channel yuanbao --token "appKey:appSecret"
  ```
  `--token` には、コロンで区切った `appKey:appSecret` を使用します。アプリケーション設定でボットを作成し、Yuanbao アプリからこれらを取得してください。
  </Step>

  <Step title="変更を適用するために Gateway を再起動する">
  ```bash
  openclaw gateway restart
  ```
  </Step>
</Steps>

### 対話形式のセットアップ（代替方法）

```bash
openclaw channels login --channel yuanbao
```

プロンプトに従って App ID と App Secret を入力します。

## アクセス制御

### ダイレクトメッセージ

`channels.yuanbao.dm.policy`:

| 値               | 動作                                              |
| ---------------- | ------------------------------------------------- |
| `open`（デフォルト） | すべてのユーザーを許可                            |
| `pairing`        | 不明なユーザーにペアリングコードを発行し、CLI で承認 |
| `allowlist`      | `allowFrom` のユーザーのみチャット可能            |
| `disabled`       | すべての DM を無効化                              |

ペアリング要求を承認します。

```bash
openclaw pairing list yuanbao
openclaw pairing approve yuanbao <CODE>
```

### グループチャット

`channels.yuanbao.requireMention`（デフォルト `true`）: グループ内でボットが応答する前に @メンションを必須にします。ボット自身のメッセージへの返信は、暗黙のメンションとして扱われます。

## 設定例

基本セットアップ、オープンな DM ポリシー:

```json5
{
  channels: {
    yuanbao: {
      appKey: "your_app_key",
      appSecret: "your_app_secret",
      dm: {
        policy: "open",
      },
    },
  },
}
```

DM を特定のユーザーに制限します。

```json5
{
  channels: {
    yuanbao: {
      appKey: "your_app_key",
      appSecret: "your_app_secret",
      dm: {
        policy: "allowlist",
        allowFrom: ["user_id_1", "user_id_2"],
      },
    },
  },
}
```

グループでの @メンション要件を無効にします。

```json5
{
  channels: {
    yuanbao: {
      requireMention: false,
    },
  },
}
```

送信配信の調整:

```json5
{
  channels: {
    yuanbao: {
      outboundQueueStrategy: "merge-text",
      minChars: 2800, // この文字数までバッファリング
      maxChars: 3000, // この上限を超えた場合は強制的に分割
      idleMs: 5000, // アイドルタイムアウト後に自動フラッシュ（ms）
    },
  },
}
```

バッファリングせず各チャンクを送信するには、`outboundQueueStrategy: "immediate"` を設定します。

## 一般的なコマンド

| コマンド   | 説明                         |
| ---------- | ---------------------------- |
| `/help`    | 利用可能なコマンドを表示     |
| `/status`  | ボットのステータスを表示     |
| `/new`     | 新しいセッションを開始       |
| `/stop`    | 現在の実行を停止             |
| `/restart` | OpenClaw を再起動             |
| `/compact` | セッションコンテキストを圧縮 |

Yuanbao はネイティブのスラッシュコマンドメニューをサポートしています。Gateway の起動時にコマンドがプラットフォームへ自動的に同期されます。

## トラブルシューティング

**ボットがグループチャットで応答しない場合:**

1. ボットがグループに追加されていることを確認します
2. ボットを @メンションしていることを確認します（デフォルトで必須）
3. ログを確認します: `openclaw logs --follow`

**ボットがメッセージを受信しない場合:**

1. Yuanbao アプリでボットが作成され、承認されていることを確認します
2. `appKey` と `appSecret` が正しく設定されていることを確認します
3. Gateway が実行中であることを確認します: `openclaw gateway status`
4. ログを確認します: `openclaw logs --follow`

**ボットが空の応答またはフォールバック応答を送信する場合:**

1. AI モデルが有効なコンテンツを返しているか確認します
2. デフォルトのフォールバック応答: 「暂时无法解答，你可以换个问题问问我哦」
3. `channels.yuanbao.fallbackReply` でカスタマイズします

**App Secret が漏洩した場合:**

1. Yuanbao アプリで App Secret をリセットします
2. 設定内の値を更新します
3. Gateway を再起動します: `openclaw gateway restart`

## 高度な設定

### 複数アカウント

```json5
{
  channels: {
    yuanbao: {
      defaultAccount: "main",
      accounts: {
        main: {
          appKey: "key_xxx",
          appSecret: "secret_xxx",
          name: "Primary bot",
        },
        backup: {
          appKey: "key_yyy",
          appSecret: "secret_yyy",
          name: "Backup bot",
          enabled: false,
        },
      },
    },
  },
}
```

送信 API で `accountId` が指定されていない場合、`defaultAccount` によって使用するアカウントが決まります。

### メッセージ制限

- `maxChars`: 1 件のメッセージの最大文字数（デフォルト `3000`）
- `mediaMaxMb`: メディアのアップロード／ダウンロード制限（デフォルト `20` MB）
- `overflowPolicy`: メッセージが上限を超えた場合の動作。`"split"`（デフォルト）または `"stop"`

### ストリーミング

Yuanbao はブロック単位のストリーミング出力をサポートしています。ボットは生成しながらテキストをチャンク単位で送信します。

```json5
{
  channels: {
    yuanbao: {
      disableBlockStreaming: false, // ブロックストリーミングが有効（デフォルト）
    },
  },
}
```

完全な応答を 1 件のメッセージで送信するには、`disableBlockStreaming: true` を設定します。

### グループチャットの履歴コンテキスト

```json5
{
  channels: {
    yuanbao: {
      historyLimit: 100, // デフォルト: 100、無効にするには 0 を設定
    },
  },
}
```

グループチャットの AI コンテキストに含める過去のメッセージ数を制御します。

### 返信先モード

```json5
{
  channels: {
    yuanbao: {
      replyToMode: "first", // "off" | "first" | "all"（デフォルト: "first"）
    },
  },
}
```

| 値      | 動作                                                   |
| ------- | ------------------------------------------------------ |
| `off`   | 引用返信なし                                           |
| `first` | 受信メッセージごとに最初の返信のみ引用（デフォルト）   |
| `all`   | すべての返信を引用                                     |

### Markdown ヒントの挿入

デフォルトでは、モデルが応答全体を Markdown コードブロックで囲まないようにするため、ボットはシステムプロンプトへ指示を挿入します。

```json5
{
  channels: {
    yuanbao: {
      markdownHintEnabled: true, // デフォルト: true
    },
  },
}
```

### デバッグモード

```json5
{
  channels: {
    yuanbao: {
      debugBotIds: ["bot_user_id_1", "bot_user_id_2"],
    },
  },
}
```

一覧に含まれるボット ID に対して、サニタイズされていないログ出力を有効にします。

### マルチエージェントルーティング

Yuanbao の DM またはグループを異なるエージェントへルーティングするには、`bindings` を使用します。

```json5
{
  agents: {
    list: [
      { id: "main" },
      { id: "agent-a", workspace: "/home/user/agent-a" },
      { id: "agent-b", workspace: "/home/user/agent-b" },
    ],
  },
  bindings: [
    {
      agentId: "agent-a",
      match: {
        channel: "yuanbao",
        peer: { kind: "direct", id: "user_xxx" },
      },
    },
    {
      agentId: "agent-b",
      match: {
        channel: "yuanbao",
        peer: { kind: "group", id: "group_zzz" },
      },
    },
  ],
}
```

- `match.channel`: `"yuanbao"`
- `match.peer.kind`: `"direct"`（DM）または `"group"`（グループチャット）
- `match.peer.id`: ユーザー ID またはグループコード

## 設定リファレンス

完全な設定: [Gateway の設定](/ja-JP/gateway/configuration)

| 設定                                       | 説明                                              | デフォルト                             |
| ------------------------------------------ | ------------------------------------------------- | -------------------------------------- |
| `channels.yuanbao.enabled`                 | チャンネルを有効化／無効化                        | `true`                                 |
| `channels.yuanbao.defaultAccount`          | 送信ルーティングのデフォルトアカウント            | `default`                              |
| `channels.yuanbao.accounts.<id>.appKey`    | App Key（署名とチケット生成）                     | -                                      |
| `channels.yuanbao.accounts.<id>.appSecret` | App Secret（署名）                                | -                                      |
| `channels.yuanbao.accounts.<id>.token`     | 署名済みトークン（自動チケット署名を省略）        | -                                      |
| `channels.yuanbao.accounts.<id>.name`      | アカウント表示名                                  | -                                      |
| `channels.yuanbao.accounts.<id>.enabled`   | 特定のアカウントを有効化／無効化                  | `true`                                 |
| `channels.yuanbao.dm.policy`               | DM ポリシー                                       | `open`                                 |
| `channels.yuanbao.dm.allowFrom`            | DM 許可リスト（ユーザー ID の一覧）               | -                                      |
| `channels.yuanbao.requireMention`          | グループで @メンションを必須にする                | `true`                                 |
| `channels.yuanbao.overflowPolicy`          | 長いメッセージの処理（`split` または `stop`） | `split`                       |
| `channels.yuanbao.replyToMode`             | グループの返信先戦略（`off`、`first`、`all`） | `first`              |
| `channels.yuanbao.outboundQueueStrategy`   | 送信戦略（`merge-text` または `immediate`） | `merge-text`                           |
| `channels.yuanbao.minChars`                | テキスト結合: 送信を開始する最小文字数            | `2800`                                 |
| `channels.yuanbao.maxChars`                | テキスト結合: メッセージごとの最大文字数          | `3000`                                 |
| `channels.yuanbao.idleMs`                  | テキスト結合: 自動フラッシュまでのアイドルタイムアウト（ms） | `5000`                    |
| `channels.yuanbao.mediaMaxMb`              | メディアサイズの上限（MB）                        | `20`                                   |
| `channels.yuanbao.historyLimit`            | グループチャット履歴のコンテキスト件数            | `100`                                  |
| `channels.yuanbao.disableBlockStreaming`   | ブロック単位のストリーミング出力を無効化          | `false`                                |
| `channels.yuanbao.fallbackReply`           | モデルがコンテンツを返さない場合のフォールバック応答 | `暂时无法解答，你可以换个问题问问我哦` |
| `channels.yuanbao.markdownHintEnabled`     | Markdown の全体囲みを防ぐ指示を挿入               | `true`                                 |
| `channels.yuanbao.debugBotIds`             | デバッグ許可リストのボット ID（サニタイズされていないログ） | `[]`                         |

## サポートされるメッセージタイプ

**受信:** テキスト、画像、ファイル、音声／ボイス、動画、ステッカー／カスタム絵文字、カスタム要素（リンクカード）。

**送信:** テキスト（Markdown）、画像、ファイル、音声、動画、ステッカー。

**スレッドと返信:** 引用返信（`replyToMode` で設定可能）。スレッド返信はプラットフォームでサポートされていません。

## 関連項目

- [チャンネル概要](/ja-JP/channels) - サポートされているすべてのチャンネル
- [ペアリング](/ja-JP/channels/pairing) - DM 認証とペアリングの流れ
- [グループ](/ja-JP/channels/groups) - グループチャットの動作とメンションによる制御
- [チャンネルルーティング](/ja-JP/channels/channel-routing) - メッセージのセッションルーティング
- [セキュリティ](/ja-JP/gateway/security) - アクセスモデルと堅牢化
