---
read_when:
    - 複数のメッセージチャネルで同じ許可リストを設定する
    - DM とグループの送信者アクセスルールの共有
    - メッセージチャネルのアクセス制御のレビュー
summary: メッセージチャネル向けの再利用可能な送信者許可リスト
title: アクセスグループ
x-i18n:
    generated_at: "2026-07-26T09:28:42Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 099abc95e90d9a7b7006d19062c46b4ffdb2aecb1e8e714454a3182131a786d0
    source_path: channels/access-groups.md
    workflow: 16
---

アクセスグループは、`accessGroups` の下で一度定義し、チャンネルの許可リストから `accessGroup:<name>` で参照する、名前付きの送信者リストです。

同じ利用者を複数のメッセージチャンネルで許可する場合や、1 つの信頼済みセットを DM とグループ送信者の認可の両方に適用する場合に使用します。

グループ自体は何も付与しません。許可リストフィールドから参照された場合にのみ意味を持ちます。

## 静的メッセージ送信者グループ

静的送信者グループは `type: "message.senders"` を使用します。`members` はメッセージチャンネル ID をキーとし、さらに全チャンネルで共有するエントリには `"*"` を使用します。

```json5
{
  accessGroups: {
    operators: {
      type: "message.senders",
      members: {
        "*": ["global-owner-id"],
        discord: ["discord:123456789012345678"],
        telegram: ["987654321"],
        whatsapp: ["+15551234567"],
      },
    },
  },
}
```

| キー                        | 意味                                                                     |
| -------------------------- | --------------------------------------------------------------------------- |
| `"*"`                      | このグループを参照するすべてのメッセージチャンネルで照合される共有エントリ。 |
| `discord`、`telegram`、... | そのチャンネルの許可リスト照合でのみ確認されるエントリ。                 |

エントリは、宛先チャンネルの通常の `allowFrom` ルールで照合されます。OpenClaw はチャンネル間で送信者 ID を変換しません。Alice が Telegram ID と Discord ID を持つ場合は、両方の ID を対応するチャンネルキーの下に記載してください。

## 許可リストからグループを参照する

メッセージチャンネルのパスが送信者許可リストをサポートする任意の場所で、`accessGroup:<name>` を使用してグループを参照します。

DM 許可リストの例：

```json5
{
  accessGroups: {
    operators: {
      type: "message.senders",
      members: {
        discord: ["discord:123456789012345678"],
        telegram: ["987654321"],
      },
    },
  },
  channels: {
    discord: {
      dmPolicy: "allowlist",
      allowFrom: ["accessGroup:operators"],
    },
    telegram: {
      dmPolicy: "allowlist",
      allowFrom: ["accessGroup:operators"],
    },
  },
}
```

グループ送信者許可リストの例：

```json5
{
  accessGroups: {
    oncall: {
      type: "message.senders",
      members: {
        whatsapp: ["+15551234567"],
        googlechat: ["users/1234567890"],
      },
    },
  },
  channels: {
    whatsapp: {
      groupPolicy: "allowlist",
      groupAllowFrom: ["accessGroup:oncall"],
    },
    googlechat: {
      groups: {
        "spaces/AAA": {
          users: ["accessGroup:oncall"],
        },
      },
    },
  },
}
```

グループと直接エントリを組み合わせることもできます。

```json5
{
  channels: {
    discord: {
      dmPolicy: "allowlist",
      allowFrom: ["accessGroup:operators", "discord:123456789012345678"],
    },
  },
}
```

## サポートされるメッセージチャンネルのパス

アクセスグループは、共有メッセージチャンネル認可パスで機能します。

- `channels.<channel>.allowFrom` などの DM 送信者許可リスト
- `channels.<channel>.groupAllowFrom` などのグループ送信者許可リスト
- 同じ送信者照合ルールを使用する、チャンネル固有のルーム単位送信者許可リスト（例：Google Chat の `groups.<space>.users`）
- メッセージチャンネルの送信者許可リストを再利用するコマンド認可パス

チャンネルのサポート状況は、そのチャンネルが OpenClaw の共有送信者認可ヘルパーを使用しているかどうかによって異なります。現在バンドルされているサポートには、ClickClack、Discord、Feishu、Google Chat、iMessage、IRC、LINE、Mattermost、Microsoft Teams、Nextcloud Talk、Nostr、QQ Bot、Signal、Slack、SMS、Telegram、WhatsApp、Zalo、Zalo Personal が含まれます。静的な `message.senders` グループはチャンネルに依存しないため、新しいメッセージチャンネルでも、独自の許可リスト展開ではなく共有 Plugin SDK の受信ヘルパーを使用することで利用できます。

## Discord チャンネルのオーディエンス

Discord は動的アクセスグループタイプにも対応しています。

```json5
{
  accessGroups: {
    maintainers: {
      type: "discord.channelAudience",
      guildId: "1456350064065904867",
      channelId: "1456744319972282449",
      membership: "canViewChannel",
    },
  },
  channels: {
    discord: {
      dmPolicy: "allowlist",
      allowFrom: ["accessGroup:maintainers"],
    },
  },
}
```

`discord.channelAudience` は「現在このギルドチャンネルを表示できる Discord DM 送信者を許可する」という意味です。OpenClaw は認可時に Discord を通じて送信者を解決し、Discord の `ViewChannel` 権限ルールを適用します。`membership` は省略可能で、デフォルトは `canViewChannel` です。

`#maintainers` や `#on-call` など、Discord チャンネルがすでにチームの信頼できる唯一の情報源である場合に使用します。

要件と失敗時の動作：

- ボットにはギルドとチャンネルへのアクセス権が必要です。
- ボットには Discord Developer Portal の **Server Members Intent** が必要です。
- Discord が `Missing Access` を返した場合、送信者をギルドメンバーとして解決できない場合、またはチャンネルが別のギルドに属する場合、アクセスグループは閉鎖的に失敗します。

Discord 固有のその他の例：[Discord のアクセス制御](/ja-JP/channels/discord#access-control-and-routing)

## Plugin の診断

Plugin の作成者は、構造化されたアクセスグループの状態をフラットな許可リストへ再展開せずに検査できます。

```typescript
import { resolveAccessGroupAllowFromState } from "openclaw/plugin-sdk/access-groups";

const state = await resolveAccessGroupAllowFromState({
  accessGroups: cfg.accessGroups,
  allowFrom: channelConfig.allowFrom,
  channel: "my-channel",
  accountId: "default",
  senderId,
  isSenderAllowed,
});
```

結果には、参照済み、一致、欠落、未対応、失敗の各グループが報告されます。診断や適合性テストに使用してください。フラットな `allowFrom` 配列を引き続き必要とする互換性パスでのみ `expandAllowFromWithAccessGroups(...)` を使用してください。

## セキュリティ上の注意

- アクセスグループは許可リストのエイリアスであり、ロールではありません。アクセスグループ自体が所有者を作成したり、ペアリング要求を承認したり、ツール権限を付与したりすることはありません。
- `dmPolicy: "open"` では、引き続き有効な DM 許可リストに `"*"` が必要です。アクセスグループの参照は、公開アクセスと同じではありません。
- 存在しないグループ名は閉鎖的に失敗します。`allowFrom` に `accessGroup:operators` が含まれ、`accessGroups.operators` が存在しない場合、そのエントリは誰も認可しません。
- チャンネル ID は安定させてください。チャンネルが両方をサポートしている場合、表示名よりも数値 ID またはユーザー ID を優先してください。

## トラブルシューティング

送信者が一致するはずなのにブロックされる場合：

1. 許可リストフィールドに正確な `accessGroup:<name>` 参照が含まれていることを確認します。
2. `accessGroups.<name>.type` が正しいことを確認します。
3. 送信者 ID が対応するチャンネルキーの下、または `"*"` の下に記載されていることを確認します。
4. エントリがそのチャンネルの通常の許可リスト構文を使用していることを確認します。
5. Discord チャンネルのオーディエンスでは、ボットがギルドチャンネルを表示でき、Server Members Intent が有効になっていることを確認します。

アクセス制御設定を編集した後は、`openclaw doctor` を実行してください。実行時に入る前に、多くの無効な許可リストとポリシーの組み合わせを検出できます。
