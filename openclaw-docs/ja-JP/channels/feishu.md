---
read_when:
    - Feishu/Lark ボットに接続する場合
    - Feishu チャンネルを設定しています
summary: Feishu ボットの概要、機能、設定
title: Feishu
x-i18n:
    generated_at: "2026-07-26T09:52:34Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 5e7c4cbb704ce266b7c7b0f6e160c36c873050fee8d5808965e15b56ad637f28
    source_path: channels/feishu.md
    workflow: 16
---

OpenClaw は、公式 `@openclaw/feishu` Plugin を通じて Feishu/Lark（オールインワンのコラボレーションプラットフォーム）に接続します。ボットとの DM、グループチャット、ストリーミングカード応答、Feishu のドキュメント/wiki/ドライブ/Bitable ツールを利用できます。

**ステータス:** ボットとの DM とグループチャットにおいて本番環境で使用可能です。デフォルトのイベント転送方式は WebSocket（公開 URL は不要）で、Webhook モードは任意です。

## クイックスタート

<Note>
OpenClaw 2026.5.29 以降が必要です。`openclaw --version` を実行して確認してください。`openclaw update` でアップグレードできます。
</Note>

<Steps>
  <Step title="チャネル設定ウィザードを実行する">
  ```bash
  openclaw channels login --channel feishu
  ```
  `@openclaw/feishu` Plugin がない場合はインストールされ、その後、設定が順に案内されます。

- **手動設定**: Feishu Open Platform（`https://open.feishu.cn`）または Lark Developer（`https://open.larksuite.com`）から取得した App ID と App Secret を貼り付けます。
- **QR 設定**: Feishu アプリで QR コードをスキャンし、ボットを自動作成します。このフローでは、DM が自分のアカウント（自分の `open_id` を指定した `dmPolicy: "allowlist"`）に限定されます。

ウィザードでは、API ドメイン（Feishu または Lark）とグループポリシーについても確認されます。中国国内版 Feishu モバイルアプリで QR コードに反応しない場合は、設定を再実行して手動設定を選択してください。
</Step>

  <Step title="設定完了後、変更を適用するために Gateway を再起動する">
  ```bash
  openclaw gateway restart
  ```
  </Step>
</Steps>

## 受信イベントの耐久性

OpenClaw は、認証済みの `im.message.receive_v1` および `drive.notice.comment_add_v1` エンベロープを、エージェントへのディスパッチ前に永続キューへ登録します。保留中または再試行可能なイベントは Gateway の再起動後も維持され、チャットまたはドキュメント単位で直列化されたまま処理されます。また、アクティブな完了記録または保持されている完了記録が存在する間は、Feishu のイベント ID を使用してキューへの重複登録を抑止します。

制限付き再試行後も WebSocket イベントを永続化できない場合、OpenClaw は未コミットのターンを飛ばして処理を続行せず、そのソケットを閉じて新しい認証済み接続を強制します。リアクションや VC 会議への招待を含むその他の Feishu イベントタイプは通常のイベント経路を使用するため、この永続キューの保証は適用されません。

## アクセス制御

### ダイレクトメッセージ

ボットに DM を送信できるユーザーを制御するには、`channels.feishu.dmPolicy`（デフォルト: `pairing`）を設定します。

| 値         | 動作                                                                                                      |
| ------------- | ------------------------------------------------------------------------------------------------------------- |
| `"pairing"`   | 未知のユーザーにはペアリングコードが送信され、CLI で承認します                                                         |
| `"allowlist"` | `allowFrom` に記載されているユーザーのみチャットできます                                                                     |
| `"open"`      | DM を公開します。設定検証では、`allowFrom` に `"*"` が含まれている必要があります。ワイルドカード以外のエントリによって、アクセスは引き続き制限されます |

**ペアリング要求を承認する:**

```bash
openclaw pairing list feishu
openclaw pairing approve feishu <CODE>
```

### グループチャット

**グループポリシー**（`channels.feishu.groupPolicy`、デフォルト: `allowlist`）:

| 値         | 動作                                                                                     |
| ------------- | -------------------------------------------------------------------------------------------- |
| `"open"`      | グループ内のすべてのメッセージに応答します                                                            |
| `"allowlist"` | `groupAllowFrom` に含まれるグループ、または `groups.<chat_id>` で明示的に設定されたグループにのみ応答します |
| `"disabled"`  | すべてのグループメッセージを無効にします。明示的な `groups.<chat_id>` エントリでも上書きできません         |

**メンション要件**（`channels.feishu.requireMention`）:

- デフォルトでは @メンションが必要です。ただし、有効なグループポリシーが `"open"` の場合は、メンションを付けられないメッセージ（画像など）もエージェントに届くよう、デフォルトが `false` になります。
- 上書きするには、`true` または `false` を明示的に設定します。グループ単位で上書きするには、`channels.feishu.groups.<chat_id>.requireMention` を使用します。
- ブロードキャスト専用の `@all` および `@_all` は、ボットへのメンションとして扱われません。`@all` とボットの両方を直接メンションしたメッセージは、引き続きボットへのメンションとして扱われます。

## グループ設定の例

### すべてのグループを許可し、@メンションを不要にする

```json5
{
  channels: {
    feishu: {
      groupPolicy: "open", // "open" では requireMention のデフォルトは false
    },
  },
}
```

### すべてのグループを許可し、引き続き @メンションを必須にする

```json5
{
  channels: {
    feishu: {
      groupPolicy: "open",
      requireMention: true,
    },
  },
}
```

### 特定のグループのみを許可する

```json5
{
  channels: {
    feishu: {
      groupPolicy: "allowlist",
      // グループ ID の形式: oc_xxx
      groupAllowFrom: ["oc_xxx", "oc_yyy"],
    },
  },
}
```

`allowlist` モードでは、明示的な `groups.<chat_id>` エントリを追加してグループを許可することもできます。明示的なエントリでも `groupPolicy: "disabled"` は上書きされません。`groups.*` のワイルドカードデフォルトは一致するグループを設定しますが、それだけでグループが許可されるわけではありません。

```json5
{
  channels: {
    feishu: {
      groupPolicy: "allowlist",
      groups: {
        oc_xxx: {
          requireMention: false,
        },
      },
    },
  },
}
```

### グループ内の送信者を制限する

```json5
{
  channels: {
    feishu: {
      groupPolicy: "allowlist",
      groupAllowFrom: ["oc_xxx"],
      groups: {
        oc_xxx: {
          // ユーザーの open_id の形式: ou_xxx
          allowFrom: ["ou_user1", "ou_user2"],
        },
      },
    },
  },
}
```

`channels.feishu.groupSenderAllowFrom` は、すべてのグループに同じ送信者許可リストを設定します。グループ単位の `allowFrom` が優先されます。

### ボットが作成したメッセージ

Feishu は、デフォルトで他のボットが作成したメッセージを無視します。ボット間のグループ会話を許可するには、アプリに `im:message.group_at_msg.include_bot:readonly` および `im:message:readonly` スコープを付与してから、`allowBots` を設定します。

```json5
{
  channels: {
    feishu: {
      allowBots: true,
    },
  },
}
```

Feishu がボット作成のグループイベントを配信するのは、別のボットがこのボットをメンションした場合のみです。既存のグループポリシー、送信者許可リスト、メンション要件は引き続き適用されます。OpenClaw は自身が作成したメッセージを破棄し、すべてのテキスト応答またはカード応答で相手のボットをメンションし、共通の [`channels.defaults.botLoopProtection`](/ja-JP/channels/bot-loop-protection) ガードを適用します。

<a id="get-groupuser-ids"></a>

## グループ ID/ユーザー ID を取得する

### グループ ID（`chat_id`、形式: `oc_xxx`）

Feishu/Lark でグループを開き、右上のメニューアイコンをクリックして、**Settings** に移動します。グループ ID（`chat_id`）は設定ページに表示されます。

![グループ ID を取得する](/images/feishu-get-group-id.png)

### ユーザー ID（`open_id`、形式: `ou_xxx`）

Gateway を起動してボットに DM を送信し、ログを確認します。

```bash
openclaw logs --follow
```

ログ出力で `open_id` を探します。保留中のペアリング要求を確認することもできます。

```bash
openclaw pairing list feishu
```

## よく使用するコマンド

| コマンド   | 説明                 |
| --------- | --------------------------- |
| `/status` | ボットのステータスを表示します             |
| `/reset`  | 現在のセッションをリセットします   |
| `/model`  | AI モデルを表示または切り替えます |

<Note>
Feishu/Lark はネイティブのスラッシュコマンドメニューをサポートしていないため、これらをプレーンテキストメッセージとして送信してください。
</Note>

## トラブルシューティング

### ボットがグループチャットで応答しない

1. ボットがグループに追加されていることを確認します
2. ボットを @メンションしていることを確認します（デフォルトで必須）
3. `groupPolicy` が `"disabled"` ではないことを確認します
4. ログを確認します: `openclaw logs --follow`

### ボットがメッセージを受信しない

1. ボットが Feishu Open Platform / Lark Developer で公開および承認されていることを確認します
2. イベントサブスクリプションに `im.message.receive_v1` が含まれていることを確認します
3. 会議招待への自動参加を使用する場合は、`vc.bot.meeting_invited_v1` もサブスクライブします
4. **persistent connection**（WebSocket）が選択されていることを確認します
5. 必要なすべての権限スコープが付与されていることを確認します
6. Gateway が実行中であることを確認します: `openclaw gateway status`
7. ログを確認します: `openclaw logs --follow`

`vc.bot.meeting_invited_v1` をサブスクライブしても、イベントが配信されるだけです。自動参加は
デフォルトで無効です。すべてのアカウントで有効にするには、次のように設定します。

```json5
{
  channels: {
    feishu: {
      vcAutoJoin: true,
    },
  },
}
```

1 つのアカウントだけで有効にするには、トップレベルの切り替えを省略し、アカウントの上書きを設定します。

```json5
{
  channels: {
    feishu: {
      accounts: {
        meetings: { vcAutoJoin: true },
      },
    },
  },
}
```

エージェントが参加ターンを受信する前に、招待者には通常の Feishu DM ポリシー、許可リスト/ペアリング、セッション、応答
ルーティングが引き続き適用されます。参加には、アプリのアイデンティティ用に設定され、かつ
`vc:meeting.bot.join:write` スコープを持つ利用可能な Feishu VC 参加
ツールも必要です。たとえば、公式の
[`lark-cli` VC エージェント skill](https://github.com/larksuite/cli/tree/main/skills/lark-vc-agent)
は `vc +meeting-join` を提供します。

<Warning>
公式の `lark-cli` VC エージェント skill では、現在、会議ボットのアクションが限定ベータとして示されています。ツールが `ErrNotInGray` またはエラーコード `20017` を返す場合、そのアプリまたはテナントではベータが有効になっていません。通常のスコープ付与をトラブルシューティングする前に、リンク先の skill に記載された早期アクセスの案内を使用してください。
</Warning>

### Feishu モバイルアプリで QR 設定に反応しない

1. 設定を再実行します: `openclaw channels login --channel feishu`
2. 手動設定を選択します
3. Feishu Open Platform で自社開発アプリを作成し、その App ID と App Secret をコピーします
4. それらの認証情報を設定ウィザードに貼り付けます

### App Secret が漏えいした

1. Feishu Open Platform / Lark Developer で App Secret をリセットします
2. 設定内の値を更新します
3. Gateway を再起動します: `openclaw gateway restart`

## 高度な設定

### 複数のアカウント

```json5
{
  channels: {
    feishu: {
      defaultAccount: "main",
      accounts: {
        main: {
          appId: "cli_xxx",
          appSecret: "xxx",
          name: "プライマリボット",
          tts: {
            providers: {
              openai: { voice: "shimmer" },
            },
          },
        },
        backup: {
          appId: "cli_yyy",
          appSecret: "yyy",
          name: "バックアップボット",
          enabled: false,
        },
      },
    },
  },
}
```

`defaultAccount` は、送信 API で `accountId` が指定されていない場合に使用するアカウントを制御します。アカウントエントリはトップレベルの設定を継承し、ほとんどのトップレベルキーはアカウント単位で上書きできます。
`accounts.<id>.tts` は `tts` と同じ形式を使用し、グローバル TTS 設定に対してディープマージされます。そのため、複数ボットの Feishu 設定では、共有するプロバイダー認証情報をグローバルに保持しながら、音声、モデル、ペルソナ、または自動モードのみをアカウント単位で上書きできます。

### メッセージの制限

- `textChunkLimit` - 送信テキストのチャンクサイズ（デフォルト: `4000` 文字）
- `streaming.chunkMode` - `"length"`（デフォルト）は上限で分割し、`"newline"` は改行位置を優先します
- `mediaMaxMb` - メディアのアップロード/ダウンロード上限（デフォルト: `30` MB）

### ストリーミング

Feishu/Lark は、インタラクティブカード（Card Kit ストリーミング API）によるストリーミング応答をサポートしています。有効にすると、ボットはテキストの生成中にカードをリアルタイムで更新します。

```json5
{
  channels: {
    feishu: {
      streaming: {
        mode: "partial", // ストリーミングカード出力（デフォルト: "partial"）
        block: { enabled: true }, // 完了ブロックのストリーミングを有効にする
      },
    },
  },
}
```

`streaming.mode: "off"` を設定すると、返信全体が 1 つのメッセージで送信されます。`renderMode: "raw"`（カードではなくプレーンテキスト）でもストリーミングカードが無効になります。`streaming.block.enabled` はデフォルトで無効です。完了したアシスタントブロックを最終返信の前に送信したい場合にのみ有効にしてください。従来のブール値 `streaming` とフラットな `blockStreaming` / `blockStreamingCoalesce` / `chunkMode` キーは、`openclaw doctor --fix` によってこのネストされた形式へ移行されます。

### クォータの最適化

2 つのオプションフラグを使用して、Feishu/Lark API の呼び出し回数を減らせます。

- `typingIndicator`（デフォルト `true`）：`false` に設定すると、入力中リアクションの呼び出しを省略します
- `resolveSenderNames`（デフォルト `true`）：`false` に設定すると、送信者プロフィールの検索を省略します

```json5
{
  channels: {
    feishu: {
      typingIndicator: false,
      resolveSenderNames: false,
    },
  },
}
```

### グループセッションのスコープとトピックスレッド

`channels.feishu.groupSessionScope`（トップレベル、アカウント単位、またはグループ単位）は、グループメッセージをエージェントセッションにマッピングする方法を制御します。

| 値                  | セッション                                                          |
| ---------------------- | ---------------------------------------------------------------- |
| `"group"`（デフォルト）    | グループチャットごとに 1 つのセッション                                       |
| `"group_sender"`       | （グループ + 送信者）ごとに 1 つのセッション                                 |
| `"group_topic"`        | トピックスレッドごとに 1 つのセッション。グループセッションにフォールバックします    |
| `"group_topic_sender"` | （トピック + 送信者）ごとに 1 つのセッション。（グループ + 送信者）にフォールバックします |

トピックスコープでは、Feishu/Lark のネイティブトピックグループはイベント `thread_id`（`omt_*`）を正規のトピックセッションキーとして使用します。ネイティブのトピック開始イベントで `thread_id` が省略されている場合、OpenClaw はターンをルーティングする前に Feishu からその値を取得します。OpenClaw がスレッドに変換する通常のグループ返信では、引き続き返信ルートメッセージ ID（`om_*`）を使用するため、最初のターンと後続のターンが同じセッションに維持されます。

`replyInThread: "enabled"`（トップレベルまたはグループ単位）を設定すると、ボットの返信はインラインで返信する代わりに、Feishu のトピックスレッドを作成または継続します。`topicSessionMode` は `groupSessionScope` の非推奨の旧設定です。`groupSessionScope` を使用してください。

### Feishu ワークスペースツール

Plugin には、Feishu のドキュメント、チャット、ナレッジベース、クラウドストレージ、権限、Bitable 用のエージェントツールと、対応する Skills（`feishu-doc`、`feishu-drive`、`feishu-perm`、`feishu-wiki`）が含まれています。ツールファミリーは `channels.feishu.tools` によって制限されます。

| キー             | ツール                                         | デフォルト             |
| --------------- | --------------------------------------------- | ------------------- |
| `tools.doc`     | `feishu_doc` ドキュメント操作              | `true`              |
| `tools.chat`    | `feishu_chat` チャット情報 + メンバーのクエリ      | `true`              |
| `tools.wiki`    | `feishu_wiki` ナレッジベース（`doc` が必要） | `true`              |
| `tools.drive`   | `feishu_drive` クラウドストレージ                  | `true`              |
| `tools.perm`    | `feishu_perm` 権限管理           | `false`（機密性が高い） |
| `tools.scopes`  | `feishu_app_scopes` アプリスコープ診断     | `true`              |
| `tools.bitable` | `feishu_bitable_*` Bitable/Base 操作    | `true`              |

`tools.base` は `tools.bitable` のエイリアスです。両方が設定されている場合は、明示的な `bitable` の値が優先されます。アカウント単位の制限は `accounts.<id>.tools` に配置します。

ルートディレクトリ外で `feishu_drive info` を直接検索するには、アプリに完全な `drive:drive` スコープがすでに付与されている場合を除き、`drive:drive.metadata:readonly` を付与してください。どちらのスコープもない場合、`info` により、従来のルートディレクトリ検索を `drive:drive:readonly` 経由で引き続き利用できます。

### ACP セッション

Feishu/Lark は、DM とグループスレッドメッセージで ACP をサポートしています。Feishu/Lark の ACP はテキストコマンドで操作します。ネイティブのスラッシュコマンドメニューはないため、会話内で `/acp ...` メッセージを直接使用してください。

#### 永続的な ACP バインディング

```json5
{
  agents: {
    list: [
      {
        id: "codex",
        runtime: {
          type: "acp",
          acp: {
            agent: "codex",
            backend: "acpx",
            mode: "persistent",
            cwd: "/workspace/openclaw",
          },
        },
      },
    ],
  },
  bindings: [
    {
      type: "acp",
      agentId: "codex",
      match: {
        channel: "feishu",
        accountId: "default",
        peer: { kind: "direct", id: "ou_1234567890" },
      },
    },
    {
      type: "acp",
      agentId: "codex",
      match: {
        channel: "feishu",
        accountId: "default",
        peer: { kind: "group", id: "oc_group_chat:topic:om_topic_root" },
      },
      acp: { label: "codex-feishu-topic" },
    },
  ],
}
```

#### チャットから ACP を生成

Feishu/Lark の DM またはスレッドで次を実行します。

```text
/acp spawn codex --thread here
```

`--thread here` は、DM と Feishu/Lark のスレッドメッセージで機能します。バインドされた会話内の後続メッセージは、その ACP セッションへ直接ルーティングされます。

### マルチエージェントルーティング

`bindings` を使用して、Feishu/Lark の DM またはグループを異なるエージェントへルーティングします。

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
        channel: "feishu",
        peer: { kind: "direct", id: "ou_xxx" },
      },
    },
    {
      agentId: "agent-b",
      match: {
        channel: "feishu",
        peer: { kind: "group", id: "oc_zzz" },
      },
    },
  ],
}
```

ルーティングフィールド：

- `match.channel`：`"feishu"`
- `match.peer.kind`：`"direct"`（DM）または `"group"`（グループチャット）
- `match.peer.id`：ユーザー Open ID（`ou_xxx`）またはグループ ID（`oc_xxx`）

検索のヒントについては、[グループ ID／ユーザー ID を取得する](#get-groupuser-ids)を参照してください。

## ユーザー単位のエージェント分離（動的エージェント作成）

`dynamicAgentCreation` を有効にすると、DM ユーザーごとに**分離されたエージェントインスタンス**が自動的に作成されます。各ユーザーには、以下が個別に割り当てられます。

- 独立したワークスペースディレクトリ
- 個別の `USER.md` / `SOUL.md` / `MEMORY.md`
- 非公開の会話履歴
- 分離された Skills と状態

これは、各ユーザーに専用の非公開 AI アシスタント体験を提供する公開ボットに不可欠です。

<Note>
動的バインディングには正規化された Feishu `accountId` が含まれるため、デフォルトアカウントと名前付きアカウントのどちらでも、各送信者が正しい動的エージェントへルーティングされます。

名前付きアカウントが以前のリリースでスコープなしの動的エージェントを作成した場合、その従来のエージェントも引き続き `maxAgents` にカウントされます。削除する前にデフォルトアカウントで使用されていないことを確認するか、`maxAgents` を一時的に増やしてください。OpenClaw は、所有アカウントが曖昧な従来の状態を安全に推測できません。
</Note>

### クイックセットアップ

```json5
{
  channels: {
    feishu: {
      dmPolicy: "open",
      allowFrom: ["*"],
      dynamicAgentCreation: {
        enabled: true,
        workspaceTemplate: "~/.openclaw/workspace-{agentId}",
        agentDirTemplate: "~/.openclaw/agents/{agentId}/agent",
      },
    },
  },
  session: {
    // 重要：各ユーザーの DM をそのユーザーの「メインセッション」にします
    // USER.md / SOUL.md / MEMORY.md を自動的に読み込みます
    // より強力に分離するには、代わりに "per-channel-peer" を使用します
    dmScope: "main",
  },
}
```

### 動作の仕組み

新しいユーザーが初めて DM を送信すると、次の処理が行われます。

1. チャンネルが一意の `agentId` を生成します。デフォルトアカウントでは `feishu-{user_open_id}`、名前付きアカウントでは長さを制限したアカウント接頭辞付きの識別子ダイジェストになります
2. `workspaceTemplate` パスに新しいワークスペースを作成します
3. エージェントを登録し、このユーザー用のバインディングを作成します
4. ワークスペースヘルパーが、初回アクセス時にブートストラップファイル（`AGENTS.md`、`SOUL.md`、`USER.md` など）を確実に用意します
5. このユーザーからの今後のすべてのメッセージを専用エージェントへルーティングします

### 設定オプション

| 設定                                                  | 説明                                | デフォルト                              |
| -------------------------------------------------------- | ------------------------------------------ | ------------------------------------ |
| `channels.feishu.dynamicAgentCreation.enabled`           | ユーザー単位のエージェント自動作成を有効にする   | `false`                              |
| `channels.feishu.dynamicAgentCreation.workspaceTemplate` | 動的エージェントのワークスペース用パステンプレート | `~/.openclaw/workspace-{agentId}`    |
| `channels.feishu.dynamicAgentCreation.agentDirTemplate`  | エージェントディレクトリ名のテンプレート              | `~/.openclaw/agents/{agentId}/agent` |
| `channels.feishu.dynamicAgentCreation.maxAgents`         | 作成する動的エージェントの最大数 | 無制限                            |

テンプレート変数：

- `{agentId}` - 生成されたエージェント ID（例：`feishu-ou_xxxxxx` または `feishu-support-<identity_digest>`）
- `{userId}` - 送信者の Feishu open_id（例：`ou_xxxxxx`）

### セッションスコープ

`session.dmScope` は、ダイレクトメッセージをエージェントセッションにマッピングする方法を制御します。これは、すべてのチャンネルに影響する**グローバル設定**です。

| 値                        | 動作                                                            | 適した用途                                                           |
| ---------------------------- | ------------------------------------------------------------------- | ------------------------------------------------------------------ |
| `"main"`                     | 各ユーザーの DM をそのエージェントのメインセッションにマッピングします                   | `USER.md` / `SOUL.md` を自動読み込みしたいシングルユーザーボット |
| `"per-peer"`                 | 各ピアに個別のセッションを割り当てます（チャンネルは問いません）           | 送信者の識別情報のみに基づく分離                            |
| `"per-channel-peer"`         | （チャンネル + ユーザー）の組み合わせごとに個別のセッションを割り当てます           | より強力な分離が必要な公開マルチユーザーボット                  |
| `"per-account-channel-peer"` | （アカウント + チャンネル + ユーザー）の組み合わせごとに個別のセッションを割り当てます | アカウント単位のセッション分離が必要なマルチアカウントボット         |

**トレードオフ**：`"main"` を使用すると、ブートストラップファイル（`USER.md`、`SOUL.md`、`MEMORY.md`）の自動読み込みが有効になりますが、すべてのチャンネルの全 DM で同じセッションキーパターンが共有されます。ブートストラップの自動読み込みより分離を重視する公開マルチユーザーボットでは、`"per-channel-peer"` を検討し、ブートストラップファイルを手動で管理してください。

<Note>
名前付き Feishu アカウントで、同じ送信者に対して個別のセッションを維持する場合は、`"per-account-channel-peer"` を使用してください。動的バインディングではアカウントスコープが維持されます。
</Note>

### 一般的なマルチユーザー構成

```json5
{
  channels: {
    feishu: {
      appId: "cli_xxx",
      appSecret: "xxx",
      dmPolicy: "open",
      allowFrom: ["*"],
      groupPolicy: "open",
      requireMention: true,
      dynamicAgentCreation: {
        enabled: true,
        workspaceTemplate: "~/.openclaw/workspace-{agentId}",
        agentDirTemplate: "~/.openclaw/agents/{agentId}/agent",
      },
    },
  },
  session: {
    // 必要な分離レベルに基づいて dmScope を選択します：
    // ブートストラップの自動読み込みには "main"、より強力な分離には "per-channel-peer"
    dmScope: "main",
  },
  bindings: [], // 空 - 動的エージェントは自動的にバインドされます
}
```

### 検証

Gateway のログを確認し、動的作成が機能していることを確認します。

```text
feishu: ユーザー ou_xxxxxx 用の動的エージェント "feishu-ou_xxxxxx" を作成しています
  ワークスペース: /home/user/.openclaw/workspace-feishu-ou_xxxxxx
  エージェントディレクトリ: /home/user/.openclaw/agents/feishu-ou_xxxxxx/agent
```

作成されたすべてのワークスペースを一覧表示します。

```bash
ls -la ~/.openclaw/workspace-*
```

### 注記

- **ワークスペースの分離**: 各ユーザーには専用のワークスペースディレクトリとエージェントインスタンスが割り当てられます。通常のメッセージングフローでは、ユーザーは互いの会話履歴やファイルを参照できません。
- **セキュリティ境界**: これはメッセージングコンテキストを分離する仕組みであり、敵対的な共同テナントに対するセキュリティ境界ではありません。エージェントプロセスとホスト環境は共有されます。
- **設定の書き込みを有効にしておく必要があります**: 動的エージェントの作成では、エージェントとバインディングが設定に書き込まれます。`channels.feishu.configWrites` が `false` の場合はスキップされます（デフォルト: 有効）。
- **`bindings` は空にする必要があります**: 動的エージェントは自身のバインディングを自動登録します
- **アップグレードパス**: 既存の手動バインディングは動的エージェントと併用できます
- **`session.dmScope` はグローバルです**: これは Feishu だけでなく、すべてのチャンネルに影響します

## 設定リファレンス

完全な設定: [Gateway の設定](/ja-JP/gateway/configuration)

| 設定                                                     | 説明                                                                                 | デフォルト                           |
| -------------------------------------------------------- | ------------------------------------------------------------------------------------ | ------------------------------------ |
| `channels.feishu.enabled`                                | チャンネルを有効化または無効化                                                       | `true`                               |
| `channels.feishu.domain`                                 | API ドメイン（`feishu`、`lark`、または `https://` ベース URL）                             | `feishu`                             |
| `channels.feishu.connectionMode`                         | イベント転送方式（`websocket` または `webhook`）                                           | `websocket`                          |
| `channels.feishu.defaultAccount`                         | 送信ルーティングのデフォルトアカウント                                               | `default`                            |
| `channels.feishu.verificationToken`                      | Webhook モードでは必須                                                               | -                                    |
| `channels.feishu.encryptKey`                             | Webhook モードでは必須                                                               | -                                    |
| `channels.feishu.webhookPath`                            | Webhook のルートパス                                                                 | `/feishu/events`                     |
| `channels.feishu.webhookHost`                            | Webhook のバインドホスト                                                             | `127.0.0.1`                          |
| `channels.feishu.webhookPort`                            | Webhook のバインドポート                                                             | `3000`                               |
| `channels.feishu.accounts.<id>.appId`                    | アプリ ID                                                                            | -                                    |
| `channels.feishu.accounts.<id>.appSecret`                | アプリシークレット                                                                   | -                                    |
| `channels.feishu.accounts.<id>.domain`                   | アカウントごとのドメイン上書き                                                       | `feishu`                             |
| `channels.feishu.accounts.<id>.tts`                      | アカウントごとの TTS 上書き                                                          | `tts`                                |
| `channels.feishu.dmPolicy`                               | DM ポリシー（`pairing`、`allowlist`、`open`）                                           | `pairing`                            |
| `channels.feishu.allowFrom`                              | DM 許可リスト（open_id のリスト）                                                     | -                                    |
| `channels.feishu.groupPolicy`                            | グループポリシー（`open`、`allowlist`、`disabled`）                                       | `allowlist`                          |
| `channels.feishu.groupAllowFrom`                         | グループ許可リスト                                                                   | -                                    |
| `channels.feishu.groupSenderAllowFrom`                   | すべてのグループに適用される送信者許可リスト                                         | -                                    |
| `channels.feishu.requireMention`                         | グループ内で @メンションを必須にする                                                 | `true`（ポリシーが `open` の場合は `false`）  |
| `channels.feishu.allowBots`                              | ボットループ保護を適用し、このボットをメンションする他のボットを受け入れる           | `false`                              |
| `channels.feishu.groups.<chat_id>.requireMention`        | グループごとの @メンション上書き。明示的な ID は許可リストモードでもそのグループを許可します     | 継承                                 |
| `channels.feishu.groups.<chat_id>.enabled`               | 特定のグループを有効化または無効化                                                   | `true`                               |
| `channels.feishu.groups.<chat_id>.allowFrom`             | グループごとの送信者許可リスト（`groupSenderAllowFrom` を上書き）                        | -                                    |
| `channels.feishu.groupSessionScope`                      | グループセッションのマッピング（`group`、`group_sender`、`group_topic`、`group_topic_sender`） | `group`                              |
| `channels.feishu.replyInThread`                          | ボットの返信でトピックスレッドを作成または継続（`disabled`、`enabled`）                    | `disabled`                           |
| `channels.feishu.reactionNotifications`                  | 受信リアクションイベント（`off`、`own`、`all`）                                        | `own`                                |
| `channels.feishu.vcAutoJoin`                             | 通常の DM 認可後、招待された VC 会議に参加                                           | `false`                              |
| `channels.feishu.dynamicAgentCreation.enabled`           | ユーザーごとのエージェントの自動作成を有効化                                         | `false`                              |
| `channels.feishu.dynamicAgentCreation.workspaceTemplate` | 動的エージェントのワークスペース用パステンプレート                                   | `~/.openclaw/workspace-{agentId}`    |
| `channels.feishu.dynamicAgentCreation.agentDirTemplate`  | エージェントディレクトリ名のテンプレート                                             | `~/.openclaw/agents/{agentId}/agent` |
| `channels.feishu.dynamicAgentCreation.maxAgents`         | 作成する動的エージェントの最大数                                                     | 無制限                               |
| `channels.feishu.textChunkLimit`                         | メッセージのチャンクサイズ                                                           | `4000`                               |
| `channels.feishu.streaming.chunkMode`                    | チャンクの分割（`length` または `newline`）                                              | `length`                             |
| `channels.feishu.mediaMaxMb`                             | メディアのサイズ制限                                                                 | `30`                                 |
| `channels.feishu.renderMode`                             | 返信のレンダリング（`auto`、`raw`、`card`）                                              | `auto`                               |
| `channels.feishu.streaming.mode`                         | ストリーミングカード出力（`partial` または `off`）                                           | `partial`                            |
| `channels.feishu.streaming.block.enabled`                | 完了したブロック単位での返信ストリーミング                                           | `false`                              |
| `channels.feishu.typingIndicator`                        | 入力中リアクションを送信                                                             | `true`                               |
| `channels.feishu.resolveSenderNames`                     | 送信者の表示名を解決                                                                 | `true`                               |
| `channels.feishu.configWrites`                           | チャンネルから開始される設定の書き込みを許可（動的エージェントに必要）               | `true`                               |
| `channels.feishu.tools.doc`                              | ドキュメントツールを有効化                                                           | `true`                               |
| `channels.feishu.tools.chat`                             | チャット情報ツールを有効化                                                           | `true`                               |
| `channels.feishu.tools.wiki`                             | ナレッジベースツールを有効化（`doc` が必要）                                         | `true`                               |
| `channels.feishu.tools.drive`                            | クラウドストレージツールを有効化                                                     | `true`                               |
| `channels.feishu.tools.perm`                             | 権限管理ツールを有効化                                                               | `false`                              |
| `channels.feishu.tools.scopes`                           | アプリスコープ診断ツールを有効化                                                     | `true`                               |
| `channels.feishu.tools.bitable`                          | Bitable/Base ツールを有効化                                                          | `true`                               |
| `channels.feishu.tools.base`                             | `channels.feishu.tools.bitable` のエイリアス。両方が設定されている場合は明示的な `bitable` が優先     | `true`                               |
| `channels.feishu.accounts.<id>.tools.bitable`            | アカウントごとの Bitable/Base ツールゲート                                           | 継承                                 |
| `channels.feishu.accounts.<id>.tools.base`               | アカウントごとの `tools.bitable` のエイリアス                                                | 継承                                 |

## 対応しているメッセージタイプ

### 受信

- ✅ テキスト
- ✅ リッチテキスト（投稿）
- ✅ 画像
- ✅ ファイル
- ✅ 音声
- ✅ 動画／メディア
- ✅ ステッカー

受信した Feishu/Lark の音声メッセージは、生の `file_key` JSON ではなく、
メディアプレースホルダーとして正規化されます。`tools.media.audio` が設定されている場合、
OpenClaw は音声メモのリソースをダウンロードし、エージェントターンの前に共有音声文字起こしを
実行するため、エージェントは発話内容の文字起こしを受け取ります。Feishu が音声ペイロードに
文字起こしテキストを直接含めている場合、そのテキストが使用され、追加の ASR 呼び出しは
行われません。音声文字起こしプロバイダーがない場合でも、エージェントは生の Feishu
リソースペイロードではなく、`<media:audio>` プレースホルダーと保存された添付ファイルを
受け取ります。

### 送信

- ✅ テキスト
- ✅ 画像
- ✅ ファイル
- ✅ 音声
- ✅ 動画／メディア
- ✅ インタラクティブカード（ストリーミング更新を含む）
- ⚠️ リッチテキスト（投稿形式の書式設定。Feishu/Lark のすべての作成機能には対応していません）

Feishu/Lark ネイティブの音声バブルは、Feishu の `audio` メッセージタイプを使用し、
Ogg/Opus アップロードメディア（`file_type: "opus"`）を必要とします。既存の `.opus` および `.ogg` メディアは、
ネイティブ音声として直接送信されます。MP3/WAV/M4A およびその他の音声形式と推定されるファイルは、
返信で音声配信が要求された場合（`audioAsVoice` / メッセージツール `asVoice`、TTS 音声メモの
返信を含む）に限り、`ffmpeg` を使用して 48kHz Ogg/Opus にトランスコードされます。
通常の MP3 添付ファイルは通常のファイルのままです。`ffmpeg` が存在しない場合、または
変換に失敗した場合、OpenClaw はファイル添付にフォールバックし、その理由をログに記録します。

### スレッドと返信

- ✅ インライン返信
- ✅ スレッド返信
- ✅ スレッドメッセージに返信する場合、メディア返信でもスレッド情報が維持されます

トピックグループのセッションルーティングについては、
[グループセッションのスコープとトピックスレッド](#group-session-scope-and-topic-threads)を参照してください。

## 関連項目

- [チャンネルの概要](/ja-JP/channels) - サポートされているすべてのチャンネル
- [ペアリング](/ja-JP/channels/pairing) - DM 認証とペアリングの流れ
- [グループ](/ja-JP/channels/groups) - グループチャットの動作とメンションゲート
- [チャンネルルーティング](/ja-JP/channels/channel-routing) - メッセージのセッションルーティング
- [セキュリティ](/ja-JP/gateway/security) - アクセスモデルと堅牢化
