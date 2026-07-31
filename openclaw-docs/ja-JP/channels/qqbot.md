---
read_when:
    - OpenClaw を QQ に接続する場合
    - QQ Bot の認証情報を設定する必要があります
    - QQ Botのグループチャットまたはプライベートチャットへの対応が必要な場合
summary: QQ Bot のセットアップ、設定、使用方法
title: QQ ボット
x-i18n:
    generated_at: "2026-07-26T09:29:44Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: b185a2b1182471bbec3688b40fb72b671bdf3a2e8351aa6e2f7918f4f5936825
    source_path: channels/qqbot.md
    workflow: 16
---

QQ Bot は、公式 QQ Bot API（WebSocket Gateway）を介して OpenClaw に接続します。
C2C プライベートチャットとグループでの `@` メンションが主なチャット形式であり、画像、音声、動画、ファイルなどのリッチメディアに対応しています。ギルドチャンネルメッセージでは、テキストとリモート URL の画像のみがサポートされます。音声、動画、ファイルのアップロード、およびローカル画像や Base64 画像は、ギルドチャンネルでは利用できません。リアクションとスレッドは、どの場所でもサポートされていません。

ステータス：公式のダウンロード可能な Plugin。

## インストール

```bash
openclaw plugins install @openclaw/qqbot
```

## セットアップ

1. [QQ Open Platform](https://q.qq.com/) にアクセスし、スマートフォンの QQ で QR コードをスキャンして登録またはログインします。
2. **Create Bot** をクリックして、新しい QQ Bot を作成します。
3. Bot の設定ページで **AppID** と **AppSecret** を見つけてコピーします。

<Note>
AppSecret は平文では保存されません。保存せずにページを離れた場合は、新しいものを再生成する必要があります。
</Note>

4. チャンネルを追加します。

```bash
openclaw channels add --channel qqbot --token "AppID:AppSecret"
```

5. Gateway を再起動します。

## 受信イベントの永続性

QQ Gateway のターンイベントでは、OpenClaw は保存済みの Gateway 再開シーケンスを進める前に、生のイベントを永続化します。保留中または再試行可能なターンは Gateway の再起動後も保持され、会話ごとに直列化された状態を維持します。また、処理中または保持中の完了レコードが存在する間は、プロバイダーイベント ID を使用してキューへの重複登録を抑止します。

永続キューへの登録に失敗した場合、OpenClaw はシーケンスを進めずに現在の Gateway ソケットを切断します。これにより、再接続および再開処理で未コミットのイベントを再度要求できます。キューからエージェントへの受け渡し境界では、引き続き少なくとも 1 回の配信となるため、受け渡し中にクラッシュするとターンが再実行される可能性があります。

対話形式のセットアップ：

```bash
openclaw channels add
```

ウィザードでは、AppID/AppSecret を手動で入力する代わりに、QR コードによるバインドも選択できます。対象の QQ Bot に関連付けられたスマートフォンアプリでコードをスキャンすると、バインドが完了します。OpenClaw は返された認証情報をアカウントの設定スコープに永続化します。

## 設定

最小構成：

```json5
{
  channels: {
    qqbot: {
      enabled: true,
      appId: "YOUR_APP_ID",
      clientSecret: "YOUR_APP_SECRET",
    },
  },
}
```

デフォルトアカウントの環境変数（トップレベルアカウントのみ）：

- `QQBOT_APP_ID`
- `QQBOT_CLIENT_SECRET`

ファイル参照型 AppSecret：

```json5
{
  channels: {
    qqbot: {
      enabled: true,
      appId: "YOUR_APP_ID",
      clientSecretFile: "/path/to/qqbot-secret.txt",
    },
  },
}
```

環境変数 SecretRef 型 AppSecret：

```json5
{
  channels: {
    qqbot: {
      enabled: true,
      appId: "YOUR_APP_ID",
      clientSecret: { source: "env", provider: "default", id: "QQBOT_CLIENT_SECRET" },
    },
  },
}
```

注意事項：

- `openclaw channels add --channel qqbot --token-file ...` は AppSecret のみを設定します。`appId` は、設定または `QQBOT_APP_ID` ですでに設定されている必要があります。
- `clientSecret` には、平文文字列、ファイルパス（`clientSecretFile`）、または構造化された SecretRef オブジェクトを指定できます。
- 従来の `secretref:...` / `secretref-env:...` マーカー文字列は、`clientSecret` では拒否されます。代わりに、構造化された SecretRef オブジェクトを使用してください。

### ストリーミング

```json5
{
  channels: {
    qqbot: {
      streaming: {
        mode: "partial", // ブロックストリーミング："partial"（デフォルト）または "off"
        nativeTransport: true, // DM で QQ 公式の C2C stream_messages API を使用
      },
    },
  },
}
```

- `streaming.mode: "off"` は、アカウントのブロックストリーミングを無効にします。
- `streaming.nativeTransport: true` は、QQ 公式の `stream_messages` API を介して C2C（DM）の返信をストリーミングします。グループおよびチャンネルのターゲットには影響しません。
- 従来の `streaming: true|false` スカラー値と `streaming.c2cStreamApi` キーは、`openclaw doctor --fix` によってこの形式へ移行されます。
- `/bot-streaming on|off` を使用すると、DM から同じ設定を切り替えられます。

### アクセスポリシー

- `allowFrom` / `groupAllowFrom` は、C2C / グループの各コンテキストで Bot とチャットできるユーザーを制限します。`dmPolicy` / `groupPolicy`（`open` | `allowlist` | `disabled`）は適用モードを制御します。`allowFrom` に具体的なエントリ（ワイルドカード以外）がある場合、`dmPolicy` のデフォルトは `allowlist` になり、それ以外の場合は `open` になります。`groupAllowFrom` または `allowFrom` のいずれかに具体的なエントリがある場合、`groupPolicy` のデフォルトは `allowlist` になり、それ以外の場合は `open` になります。
- 「Auth: allowlist」のスラッシュコマンドには、`dmPolicy` / `groupPolicy` の値にかかわらず、`allowFrom`（グループからの実行の場合は `groupAllowFrom`）に明示的なワイルドカード以外のエントリが必要です。詳細は[スラッシュコマンド](#slash-commands)を参照してください。

### 複数アカウントのセットアップ

1 つの OpenClaw インスタンスで複数の QQ Bot を実行します。

```json5
{
  channels: {
    qqbot: {
      enabled: true,
      appId: "111111111",
      clientSecret: "secret-of-bot-1",
      accounts: {
        bot2: {
          enabled: true,
          appId: "222222222",
          clientSecret: "secret-of-bot-2",
        },
      },
    },
  },
}
```

各アカウントは、`appId` をキーとして、独立した WebSocket 接続、API クライアント、トークンキャッシュを保持します。ログ行には所有元のアカウント ID が付与されるため、1 つの Gateway で複数の Bot を実行する場合でも、診断情報を個別に識別できます。

CLI で 2 つ目の Bot を追加します。

```bash
openclaw channels add --channel qqbot --account bot2 --token "222222222:secret-of-bot-2"
```

### グループチャット

グループ対応では、表示名ではなく QQ グループの OpenID を使用します。Bot をグループに追加してからメンションするか、メンションなしで動作するようにグループを設定します。

```json5
{
  channels: {
    qqbot: {
      groupPolicy: "allowlist",
      groupAllowFrom: ["member_openid"],
      groups: {
        "*": {
          requireMention: true,
          commandLevel: "all",
          historyLimit: 50,
          tools: { deny: ["exec", "read", "write"] },
        },
        GROUP_OPENID: {
          name: "Release room",
          requireMention: false,
          ignoreOtherMentions: true,
          commandLevel: "safety",
          historyLimit: 20,
          prompt: "Keep replies short and operational.",
        },
      },
    },
  },
}
```

`groups["*"]` はすべてのグループのデフォルトを設定します。具体的な `groups.GROUP_OPENID` エントリは、1 つのグループに対してそのデフォルトを上書きします。グループ設定：

| フィールド                 | デフォルト          | 説明                                                                                        |
| --------------------- | ---------------- | -------------------------------------------------------------------------------------------------- |
| `requireMention`      | `true`           | Bot が返信する前に `@` メンションを必須にします。                                                     |
| `commandLevel`        | `all`            | グループで実行できる組み込みスラッシュコマンドを指定します（以下を参照）。                                    |
| `ignoreOtherMentions` | `false`          | Bot ではなく他のユーザーをメンションしたメッセージを破棄します。                                           |
| `historyLimit`        | `50`             | 次にメンションされたターンのコンテキストとして保持する、直近の非メンションメッセージ数です。`0` にすると履歴を無効化します。     |
| `tools`               | —                | グループ全体でツールを許可または拒否します。                                                              |
| `toolsBySender`       | —                | 送信者ごとのツール上書き設定です。[グループ](/ja-JP/channels/groups#groupchannel-tool-restrictions-optional)を参照してください。 |
| `name`                | OpenID の接頭辞    | ログおよびグループコンテキストで使用するわかりやすいラベルです。                                                     |
| `prompt`              | 組み込みのデフォルト | エージェントコンテキストに追加されるグループごとの動作プロンプトです。                                           |

`commandLevel` に指定できる値：

| レベル    | 動作                                                                                                                                      |
| -------- | --------------------------------------------------------------------------------------------------------------------------------------------- |
| `all`    | 既存の組み込みコマンドを引き続き使用できます。一部はメニューに表示されませんが、認可されたユーザーはグループ内で引き続き実行できます。                  |
| `safety` | `/help`、`/btw`、`/stop` はグループ内に表示されます。機密性の高いコマンド（`/config`、`/tools`、`/bash` など）は、プライベートチャットで実行する必要があります。      |
| `strict` | 厳格な運用に必要なグループセッション制御のみを許可します。`/stop` は引き続き使用できるため、認可された送信者は実行中の処理を中断できます。 |

旧 QQBot の `toolPolicy` エントリは廃止されています。`openclaw doctor --fix` を実行して、`tools` に移行してください。

有効化モードは `mention` と `always` です。`requireMention: true` は `mention` に、`requireMention: false` は `always` に対応します。セッションレベルの有効化上書きが存在する場合は、設定よりも優先されます。

受信キューはピアごとに分かれています。グループピアのキュー上限はダイレクトピアより大きく（50 対 20）、満杯になった場合は人間が作成したメッセージより先に Bot が作成したメッセージを削除します。また、通常のグループメッセージが短時間に集中した場合は、送信者情報付きの 1 つのターンに統合します。スラッシュコマンドは、統合バッチとは独立して 1 つずつ実行されます。

### 音声（STT / TTS）

STT と TTS は、優先順位に基づくフォールバックを備えた 2 段階の設定に対応しています。

| 設定 | Plugin 固有                                          | フレームワークのフォールバック                               |
| ------- | -------------------------------------------------------- | ------------------------------------------------ |
| STT     | `channels.qqbot.stt`                                     | 最初の音声対応 `tools.media.models[]` エントリ |
| TTS     | `channels.qqbot.tts`、`channels.qqbot.accounts.<id>.tts` | `tts`                                            |

```json5
{
  channels: {
    qqbot: {
      stt: {
        provider: "your-provider",
        model: "your-stt-model",
      },
      tts: {
        provider: "your-provider",
        model: "your-tts-model",
        voice: "your-voice",
      },
      accounts: {
        "qq-main": {
          tts: {
            providers: {
              openai: { voice: "shimmer" },
            },
          },
        },
      },
    },
  },
}
```

いずれかに `enabled: false` を設定すると無効になります。アカウントレベルの TTS 上書きは `tts` と同じ形式を使用し、チャンネルまたはグローバルの TTS 設定に対してディープマージされます。

STT リクエストはデフォルトで 60 秒後にタイムアウトします。Plugin 固有の STT は、選択された `models.providers.<id>.timeoutSeconds` 上書きを使用します。フレームワークの音声 STT は、選択された音声対応 `tools.media.models[]` エントリの `timeoutSeconds` を使用し、その後に選択されたプロバイダーの上書きを適用します。

受信した QQ 音声添付ファイルは、生の音声ファイルを汎用の `MediaPaths` に含めず、音声メディアのメタデータとしてエージェントに公開されます。TTS が設定されている場合、プレーンテキストの返信に含まれる `[[audio_as_voice]]` によって TTS が合成され、QQ ネイティブ音声メッセージとして送信されます。

送信音声のアップロードおよびトランスコード動作は、`channels.qqbot.audioFormatPolicy` でも調整できます。

- `sttDirectFormats`
- `uploadDirectFormats`
- `transcodeEnabled`

## ターゲット形式

| 形式                     | 説明        |
| -------------------------- | ------------------ |
| `qqbot:c2c:OPENID`         | プライベートチャット（C2C） |
| `qqbot:group:GROUP_OPENID` | グループチャット         |
| `qqbot:channel:CHANNEL_ID` | ギルドチャンネル      |

<Note>
各 Bot は、それぞれ固有のユーザー OpenID セットを持ちます。Bot A が受信した OpenID を使用して、Bot B 経由でメッセージを送信することは**できません**。
</Note>

## スラッシュコマンド

AI キューに入る前にインターセプトされる組み込みコマンド：

| コマンド              | 認証      | スコープ        | 説明                                                                    |
| -------------------- | --------- | ------------ | ------------------------------------------------------------------------------ |
| `/bot-ping`          | —         | 任意          | レイテンシテスト                                                                   |
| `/bot-help`          | —         | 任意          | すべてのコマンドを一覧表示                                                              |
| `/bot-me`            | —         | プライベートのみ | `allowFrom` / `groupAllowFrom` の設定に使用する送信者の QQ ユーザー ID（openid）を表示 |
| `/bot-version`       | —         | プライベートのみ | OpenClaw フレームワークと Plugin のバージョンを表示                         |
| `/bot-upgrade`       | —         | プライベートのみ | QQBot アップグレードガイドへのリンクを表示                                              |
| `/bot-approve`       | 許可リスト | プライベートのみ | コマンド実行承認設定を管理（on / off / always / reset / status）  |
| `/bot-logs`          | 許可リスト | プライベートのみ | 最近の Gateway ログをファイルとしてエクスポート                                           |
| `/bot-clear-storage` | 許可リスト | プライベートのみ | QQBot メディアディレクトリ内のキャッシュ済みダウンロードを削除                        |
| `/bot-streaming`     | 許可リスト | プライベートのみ | C2C ストリーミング返信を切り替え                                                   |
| `/bot-group-allways` | 許可リスト | プライベートのみ | デフォルトのグループ有効化モード（メンション必須 / 常時有効）を切り替え      |

使用方法のヘルプを表示するには、任意のコマンドに `?` を追加します（例: `/bot-upgrade ?`）。

「認証: 許可リスト」のコマンドでは、送信者の openid が明示的な非ワイルドカードの
`allowFrom` リストに含まれている必要もあります（グループから発行されたコマンドでは
`groupAllowFrom` が優先され、なければ `allowFrom` にフォールバックします）。ワイルドカード
`allowFrom: ["*"]` はチャットを許可しますが、これらのコマンドは許可しません。プライベートチャット以外で実行した場合、
または認証されていない場合は、メッセージを暗黙に破棄せず、
ヒントを返します。

`/bot-me`、`/bot-version`、`/bot-upgrade` はプライベートチャット専用ですが、
許可リストは必要ありません。任意の C2C 送信者が実行できます。

QQ Bot の実行承認でデフォルトの同一チャットへのフォールバックを使用する場合、ネイティブの承認
ボタンのクリックにも、同じ明示的な非ワイルドカードのコマンド許可リストが適用されます。より広範なコマンドアクセスを許可せず、
承認のみのアクセス権を付与するには、
`channels.qqbot.execApprovals.approvers` を設定します。ネイティブの実行承認はデフォルトで
有効です。

## メディアとストレージ

- 受信、送信、および Gateway ブリッジのメディアは、`~/.openclaw/media/qqbot` 配下の単一のペイロードルートを共有します
  （`OPENCLAW_HOME` が設定されている場合はそれに従います）。そのため、アップロード、
  ダウンロード、トランスコードキャッシュは、保護された単一のディレクトリ内に保持されます。
- C2C およびグループターゲットへのリッチメディア配信は、単一の `sendMedia`
  パスを通ります。5&nbsp;MiB 以上のローカルファイルとメモリ内バッファには QQ の
  チャンクアップロードエンドポイントを使用し、それより小さいペイロードとリモート URL/Base64 ソースには
  1 回限りのアップロード API を使用します。
- ホットアップグレードによって、`openclaw.json` の書き込み完了前に Gateway が中断された場合、
  Plugin は次回起動時に、内部スナップショットからそのアカウントの直近の `appId` / `clientSecret`
  を復元します（意図的な設定変更を上書きすることはありません）。そのため、QR コードを再スキャンする
  必要はありません。

## トラブルシューティング

- **Gateway が起動しない / 受信メッセージがない:** `appId` と
  `clientSecret` が正しいこと、および QQ Open Platform でボットが有効になっていることを確認してください。
  認証情報がない場合は「QQBot not configured (missing appId or
  clientSecret)」と表示されます。
- **`--token-file` で設定しても未設定と表示される:** `--token-file` は
  AppSecret のみを設定します。`appId` は引き続き設定または `QQBOT_APP_ID` で指定する必要があります。
- **突発的なグループ返信が衝突する:** ピアのキューがいっぱいになると、受信キューは人間が作成した
  メッセージより先にボットが作成したメッセージを除去し、通常の（コマンドではない）グループメッセージの
  バーストを送信者情報付きの 1 ターンに統合します。そのため、大量のボットチャットによって
  人間のメッセージが処理されなくなることはありません。
- **能動的なメッセージが届かない:** ユーザーが最近やり取りしていない場合、
  QQ がボットから開始されたメッセージをブロックすることがあります。
- **音声が文字起こしされない:** STT が設定され、プロバイダーに
  到達できることを確認してください。

## 関連項目

- [ペアリング](/ja-JP/channels/pairing)
- [グループ](/ja-JP/channels/groups)
- [チャンネルのトラブルシューティング](/ja-JP/channels/troubleshooting)
