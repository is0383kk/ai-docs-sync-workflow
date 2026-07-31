---
read_when:
    - OpenClaw を IRC チャンネルまたは DM に接続する場合
    - IRC の許可リスト、グループポリシー、またはメンション制御を設定する場合
summary: IRC Plugin のセットアップ、アクセス制御、トラブルシューティング
title: IRC
x-i18n:
    generated_at: "2026-07-26T09:53:08Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 85c3da80b45d6611872ddbd10b3be4a5742b46e355e8bb554353a478f2a1702f
    source_path: channels/irc.md
    workflow: 16
---

IRC は、従来型のチャンネル（`#room`）やダイレクトメッセージで OpenClaw を使用したい場合に利用します。
公式 IRC Plugin をインストールし、`channels.irc` 配下で設定します。

## クイックスタート

1. Plugin をインストールします。

```bash
openclaw plugins install @openclaw/irc
```

2. `~/.openclaw/openclaw.json` で、少なくともホスト、ニックネーム、参加するチャンネルを設定します。

```json5
{
  channels: {
    irc: {
      enabled: true,
      host: "irc.example.com",
      port: 6697,
      tls: true,
      nick: "openclaw-bot",
      channels: ["#openclaw"],
    },
  },
}
```

3. Gateway を起動または再起動します。

```bash
openclaw gateway run
```

ボットの連携にはプライベート IRC サーバーを推奨します。意図的にパブリック IRC ネットワークを使用する場合、一般的な選択肢には Libera.Chat、OFTC、Snoonet があります。ボットやスウォームのバックチャンネルトラフィックには、推測されやすいパブリックチャンネルを使用しないでください。

## 受信の耐久性

OpenClaw は、受理した各 IRC `PRIVMSG` を、通常のポリシーチェックとエージェントへのディスパッチより前に、永続的な受信キューへ書き込みます。保留中または再試行可能なメッセージは Gateway の再起動後も保持され、チャンネルまたはダイレクトメッセージの相手ごとに直列化された状態を維持します。

IRC は、再生可能な配信 ID を提供せず、切断中のクライアントが受信できなかったメッセージを再送しません。そのため OpenClaw は、現在の TCP 接続内でのみ安定するローカル ID を割り当てます。キューが保護するのは、ローカルでの受理からディスパッチまでの区間です。OpenClaw に到達しなかったメッセージを復元したり、接続をまたいでサーバーによる再送を重複排除したりすることはできません。

## 接続設定

| キー                           | デフォルト                       | 注記                                                       |
| ----------------------------- | ----------------------------- | ----------------------------------------------------------- |
| `host`                        | なし（必須）               | IRC サーバーのホスト名                                         |
| `port`                        | TLS では `6697`、平文では `6667` | 1-65535                                                     |
| `tls`                         | `true`                        | 意図的に平文を使用する場合のみ `false` を設定                  |
| `nick`                        | なし（必須）               | ボットのニックネーム                                                    |
| `username`                    | ニックネーム、未設定時は `openclaw`         | IRC ユーザー名                                                |
| `realname`                    | `OpenClaw`                    | Realname/GECOS フィールド                                        |
| `password` / `passwordFile`   | なし                          | サーバーパスワード。ファイルは通常ファイルである必要があります                |
| `channels`                    | なし                          | 参加するチャンネル（`["#openclaw"]`）                          |
| `accounts` / `defaultAccount` | なし                          | マルチアカウント設定。環境変数はデフォルトアカウントにのみ適用されます |

## セキュリティのデフォルト

- IRC は、OpenClaw のオペレーターが管理するフォワードプロキシのルーティング外で、生の TCP/TLS ソケットを使用します。すべての外向き通信をそのフォワードプロキシ経由にする必要があるデプロイでは、IRC への直接の外向き通信が明示的に承認されていない限り、`channels.irc.enabled=false` を設定してください。
- `channels.irc.dmPolicy` のデフォルトは `"pairing"` です。不明な DM 送信者にはペアリングコードが送られ、`openclaw pairing approve irc <code>` で承認します。
- `channels.irc.groupPolicy` のデフォルトは `"allowlist"` です。
- `groupPolicy="allowlist"` を使用する場合、許可するチャンネルを定義するために `channels.irc.groups` を設定します。
- 平文転送を意図的に受け入れる場合を除き、TLS（`channels.irc.tls=true`）を使用してください。

## アクセス制御

IRC チャンネルには、次の 2 つの独立した「ゲート」があります。

1. **チャンネルアクセス**（`groupPolicy` + `groups`）：ボットがそのチャンネルからのメッセージを受け付けるかどうか。
2. **送信者アクセス**（`groupAllowFrom` / チャンネルごとの `groups["#channel"].allowFrom`）：そのチャンネル内でボットを起動できるユーザー。

設定キー：

- DM 許可リスト（DM 送信者アクセス）：`channels.irc.allowFrom`
- グループ送信者許可リスト（チャンネル送信者アクセス）：`channels.irc.groupAllowFrom`
- チャンネルごとの制御（チャンネル、送信者、メンションルール）：`channels.irc.groups["#channel"]`。設定項目は `requireMention`、`allowFrom`、`enabled`、`tools`、`toolsBySender`、`skills`、`systemPrompt`
- `channels.irc.groupPolicy="open"` は未設定のチャンネルを許可します（**デフォルトでは引き続きメンションが必要です**）

許可リストのエントリには、安定した送信者 ID（`nick!user@host`）を使用してください。
ニックネームのみの照合は変更され得るため、`channels.irc.dangerouslyAllowNameMatching: true` の場合にのみ有効になります。

### よくある落とし穴：`allowFrom` は DM 用であり、チャンネル用ではありません

次のようなログが表示される場合：

- `irc: drop group sender alice!ident@host (policy=allowlist)`

これは、送信者が **グループ/チャンネル** メッセージで許可されていなかったことを意味します。次のいずれかで修正します。

- `channels.irc.groupAllowFrom` を設定する（すべてのチャンネルに適用されるグローバル設定）、または
- チャンネルごとの送信者許可リスト `channels.irc.groups["#channel"].allowFrom` を設定する

例（`#openclaw` 内の全員がボットと会話できるようにする）：

```json5
{
  channels: {
    irc: {
      groupPolicy: "allowlist",
      groups: {
        "#openclaw": { allowFrom: ["*"] },
      },
    },
  },
}
```

## 返信のトリガー（メンション）

チャンネルが（`groupPolicy` + `groups` により）許可され、送信者も許可されていても、OpenClaw はグループコンテキストではデフォルトで **メンションゲート** を適用します。接続中のボットのニックネームがメッセージに含まれる場合、または設定済みのメンションパターンに一致する場合、ボットへのメンションとして扱われます。

そのため、メッセージにボットと一致するメンションパターンが含まれていないと、`drop channel … (missing-mention)` のようなログが表示されることがあります。

IRC チャンネルでボットが **メンションなしで** 返信するようにするには、そのチャンネルのメンションゲートを無効にします。

```json5
{
  channels: {
    irc: {
      groupPolicy: "allowlist",
      groups: {
        "#openclaw": {
          requireMention: false,
          allowFrom: ["*"],
        },
      },
    },
  },
}
```

または、（チャンネルごとの許可リストを使用せずに）**すべての** IRC チャンネルを許可し、引き続きメンションなしで返信するには、次のように設定します。

```json5
{
  channels: {
    irc: {
      groupPolicy: "open",
      groups: {
        "*": { requireMention: false, allowFrom: ["*"] },
      },
    },
  },
}
```

## セキュリティ上の注意（パブリックチャンネルで推奨）

パブリックチャンネルで `allowFrom: ["*"]` を許可すると、誰でもボットにプロンプトを送信できます。
リスクを軽減するため、そのチャンネルで使用できるツールを制限してください。

### チャンネル内の全員に同じツールを適用する

```json5
{
  channels: {
    irc: {
      groups: {
        "#openclaw": {
          allowFrom: ["*"],
          tools: {
            deny: ["group:runtime", "group:fs", "gateway", "nodes", "cron", "browser"],
          },
        },
      },
    },
  },
}
```

### 送信者ごとに異なるツールを適用する（所有者にはより強い権限を付与）

`toolsBySender` を使用して、`"*"` にはより厳格なポリシーを、自分のニックネームにはより緩やかなポリシーを適用します。

```json5
{
  channels: {
    irc: {
      groups: {
        "#openclaw": {
          allowFrom: ["*"],
          toolsBySender: {
            "*": {
              deny: ["group:runtime", "group:fs", "gateway", "nodes", "cron", "browser"],
            },
            "id:alice": {
              deny: ["gateway", "nodes", "cron"],
            },
          },
        },
      },
    },
  },
}
```

注記：

- `toolsBySender` のキーには、明示的なプレフィックス（`channel:`、`id:`、`e164:`、`username:`、`name:`）を使用してください。IRC では、送信者 ID の値とともに `id:` を使用します。より強い照合には `id:alice` または `id:alice!~alice@203.0.113.7` を使用します。
- 従来のプレフィックスなしキーも引き続き受け付けられますが、`id:` としてのみ照合され、非推奨警告が出力されます。
- 最初に一致した送信者ポリシーが適用されます。`"*"` はワイルドカードのフォールバックです。

グループアクセスとメンションゲートの詳細、およびそれらの相互作用については、[/channels/groups](/ja-JP/channels/groups) を参照してください。

## NickServ

接続後に NickServ で識別するには、次のように設定します。

```json5
{
  channels: {
    irc: {
      nickserv: {
        enabled: true,
        service: "NickServ",
        password: "your-nickserv-password",
      },
    },
  },
}
```

パスワードが設定されている場合、NickServ の識別はデフォルトで常に実行されます（オプトアウトする場合にのみ `enabled` を `false` にする必要があります）。`service` のデフォルトは `NickServ` です。`passwordFile` は、インラインの `password` に代わる設定です。

接続時に任意で一度だけ登録する場合（`register: true` には `registerEmail` が必要です）：

```json5
{
  channels: {
    irc: {
      nickserv: {
        register: true,
        registerEmail: "bot@example.com",
      },
    },
  },
}
```

ニックネームの登録後は、REGISTER の試行が繰り返されないように `register` を無効にしてください。

## 環境変数

デフォルトアカウントでは、次の変数を使用できます。

- `IRC_HOST`
- `IRC_PORT`
- `IRC_TLS`
- `IRC_NICK`
- `IRC_USERNAME`
- `IRC_REALNAME`
- `IRC_PASSWORD`
- `IRC_CHANNELS`（カンマ区切り）
- `IRC_NICKSERV_PASSWORD`
- `IRC_NICKSERV_REGISTER_EMAIL`

`IRC_HOST` はワークスペースの `.env` から設定できません。[ワークスペースの `.env` ファイル](/ja-JP/gateway/security)を参照してください。

## トラブルシューティング

- ボットが接続してもチャンネルでまったく返信しない場合は、`channels.irc.groups` **と**、メンションゲートによってメッセージが破棄されていないか（`missing-mention`）を確認してください。メンションなしで返信させる場合は、そのチャンネルに `requireMention:false` を設定します。
- ログインに失敗する場合は、ニックネームが使用可能であることとサーバーパスワードを確認してください。
- カスタムネットワークで TLS に失敗する場合は、ホスト、ポート、証明書の設定を確認してください。

## 関連項目

- [チャンネルの概要](/ja-JP/channels) — サポートされているすべてのチャンネル
- [ペアリング](/ja-JP/channels/pairing) — DM の認証とペアリングの流れ
- [グループ](/ja-JP/channels/groups) — グループチャットの動作とメンションゲート
- [チャンネルルーティング](/ja-JP/channels/channel-routing) — メッセージのセッションルーティング
- [セキュリティ](/ja-JP/gateway/security) — アクセスモデルと強化策
