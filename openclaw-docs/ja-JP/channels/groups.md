---
read_when:
    - グループチャットの動作またはメンション制限の変更
    - mentionPatterns の適用範囲を特定のグループ会話に限定する
sidebarTitle: Groups
summary: 各サーフェス（Discord/iMessage/Matrix/Microsoft Teams/QQBot/Signal/Slack/Telegram/WhatsApp/Zalo）におけるグループチャットの動作
title: グループ
x-i18n:
    generated_at: "2026-07-26T08:54:10Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 146378f0fc31e129b6504df6778ab8633c048cd4d02af02a5e6da1bfef640d3f
    source_path: channels/groups.md
    workflow: 16
---

OpenClaw は、Discord、iMessage、Matrix、Microsoft Teams、QQBot、Signal、Slack、Telegram、WhatsApp、Zalo など、グループに対応するすべてのチャンネルに同じグループルールを適用します。

エージェントが明示的に表示メッセージを送信しない限り、静かなコンテキストを提供する常時稼働ルームについては、[アンビエントルームイベント](/ja-JP/channels/ambient-room-events)を参照してください。

## 初心者向け概要（2 分）

OpenClaw は、自分のメッセージングアカウント上に「存在」します。独立した WhatsApp ボットユーザーはありません。**自分**がグループに参加していれば、OpenClaw はそのグループを認識し、そこで応答できます。

デフォルトの動作：

- グループは制限されます（`groupPolicy: "allowlist"`）。グループの送信者は許可リストに追加されるまでブロックされます。
- グループのメンションゲートを無効にしない限り、応答にはメンションが必要です。
- 最終応答テキストはルームに自動的に投稿されます（`visibleReplies: "automatic"`）。

つまり、許可リストに登録された送信者は、OpenClaw をメンションすることで起動できます。

<Note>
**要約**

- **DM アクセス**は `*.allowFrom` で制御されます。
- **グループアクセス**は `*.groupPolicy` と許可リスト（`*.groups`、`*.groupAllowFrom`）で制御されます。
- **応答のトリガー**はメンションゲート（`requireMention`、`/activation`）で制御されます。

</Note>

簡単な処理フロー（グループメッセージの処理）：

```text
groupPolicy? disabled -> 破棄
groupPolicy? allowlist -> グループが許可済み？ いいえ -> 破棄
requireMention? yes -> メンションされた？ いいえ -> コンテキスト専用として保存
メンション／返信／コマンド／DM -> ユーザーリクエスト
常時稼働グループの会話 -> ユーザーリクエスト、または設定されている場合はルームイベント
```

## 表示される応答

通常のグループ／チャンネルリクエストでは、OpenClaw のデフォルトは `messages.groupChat.visibleReplies: "automatic"` です。アシスタントの最終テキストが、表示される応答としてルームに投稿されます。

共有ルームで、エージェントが `message(action=send)` を呼び出して発言するタイミングを判断できるようにするには、`messages.groupChat.visibleReplies: "message_tool"` を使用します。これは、ツールを確実に使用できるモデル（たとえば GPT-5.6 Sol）で最も効果的です。モデルがツールを使用せず、実質的な最終テキストを返した場合、OpenClaw はそのテキストをルームに投稿せず非公開のまま保持します。

ツールのみの配信を確実に遵守できないモデルやランタイムには、`"automatic"` を使用します。通常の最終テキストはルームに直接投稿されます。また、最終テキストと一緒に送信できないファイル、画像、その他の添付ファイルについては、エージェントが引き続き `message(action=send)` を呼び出せます。

有効なツールポリシーでメッセージツールが利用できない場合、OpenClaw は応答を暗黙に抑制せず、表示される応答の自動送信にフォールバックします。`openclaw doctor` はこの不一致について警告します。

ダイレクトチャットやその他のソースイベントでは、`messages.visibleReplies: "message_tool"` により同じツールのみの動作がグローバルに適用されます。`messages.groupChat.visibleReplies` は、引き続きグループ／チャンネルルーム向けのより具体的なオーバーライドです。内部 WebChat のダイレクトターンでは、Pi と Codex が同じ表示応答の契約を受け取るように、最終応答の自動配信がデフォルトになります。

ツールのみモードは、ほとんどの傍観モードのターンでモデルに `NO_REPLY` と応答させる従来のパターンに代わるものです。ツールのみモードでは、プロンプトは `NO_REPLY` の契約を定義しません。表示上何もしないとは、単にメッセージツールを呼び出さないことを意味します。

Plugin が所有する会話バインディングは例外です。Plugin がスレッドをバインドして受信ターンを引き受けると、Plugin が返す応答が表示されるバインディング応答となり、`message(action=send)` は必要ありません。この応答は Plugin ランタイムの出力であり、モデルの非公開な最終テキストではありません。

ダイレクトなグループリクエストでは、入力中インジケーターも引き続き送信されます。有効化されたアンビエント常時稼働ルームイベントは、エージェントがメッセージツールを呼び出さない限り、厳格かつ静かな状態を維持します。

セッションでは、詳細なツール／進行状況の要約がデフォルトで抑制されます。デバッグ中に現在のセッションで表示するには `/verbose on`（または `/verbose full`）を使用し、最終応答のみの動作に戻すには `/verbose off` を使用します。詳細表示の状態はセッション単位であり、ダイレクトチャット、グループ、チャンネル、フォーラムトピックで同じように機能します。

メンションされていない常時稼働グループの会話を、ユーザーリクエストではなく静かなルームコンテキストとして送信するには、[アンビエントルームイベント](/ja-JP/channels/ambient-room-events)を使用します。

```json5
{
  messages: {
    groupChat: {
      unmentionedInbound: "room_event",
    },
  },
}
```

デフォルトは `unmentionedInbound: "user_request"` です。メンションされたメッセージ、コマンド、中止リクエスト、DM はユーザーリクエストのままです。

グループ／チャンネルリクエストで、表示出力をメッセージツール経由にするには：

```json5
{
  messages: {
    groupChat: {
      visibleReplies: "message_tool",
    },
  },
}
```

すべてのソースチャットで必須にするには：

```json5
{
  messages: {
    visibleReplies: "message_tool",
  },
}
```

Gateway は、ファイルの保存後、再起動せずに `messages` の設定変更を反映します。設定の再読み込みが無効な場合（`gateway.reload.mode: "off"`）にのみ再起動してください。

コマンドターンは `visibleReplies: "message_tool"` をバイパスし、常に表示される応答を返します。ネイティブスラッシュコマンド（Discord、Telegram、およびネイティブコマンドに対応するその他のサーフェス）と、認可されたテキスト `/...` コマンドは、どちらも応答をソースチャットに投稿します。グループ内の認可されていないテキスト `/...` ターンはメッセージツールのみのままです。通常のチャットターンは設定されたデフォルトに従います。

## コンテキストの可視性と許可リスト

グループの安全性には、2 つの異なる制御が関係します。

- **トリガーの認可**：エージェントを起動できるユーザー（`groupPolicy`、`groups`、`groupAllowFrom`、チャンネル固有の許可リスト）。
- **コンテキストの可視性**：モデルに注入される補足コンテキスト（返信／引用テキスト、スレッド履歴、転送メタデータ）。

OpenClaw はデフォルトで、受信したコンテキストをそのまま維持します。許可リストは、アクションを起動できるユーザーを決定するものであり、モデルに表示される引用や履歴のスニペットを決定するものではありません。補足コンテキストもフィルタリングするには、`contextVisibility` を設定します。

| モード                | 動作                                                                         |
| ------------------- | -------------------------------------------------------------------------------- |
| `"all"`（デフォルト）   | 補足コンテキストを受信したまま維持します。                                           |
| `"allowlist"`       | 許可リストに登録された送信者の履歴／スレッド／引用／転送コンテキストのみを注入します。     |
| `"allowlist_quote"` | `allowlist` に加え、任意の送信者から明示的に引用された、または返信先となったメッセージを維持します。 |

チャンネル単位（`channels.<channel>.contextVisibility`）、アカウント単位（`channels.<channel>.accounts.<accountId>.contextVisibility`）、またはグローバル（`channels.defaults.contextVisibility`）に設定します。補足コンテキストを取得するチャンネル（Discord、Feishu、iMessage、Matrix、Microsoft Teams、Signal、Slack、Telegram、WhatsApp）は、受信コンテキストの構築時にこのポリシーを適用します。不明なポリシーの組み合わせはフェイルクローズとなり、コンテキストが省略されます。

これらのモードがフィルタリングするのは、チャンネルから提供される補足コンテキストのみです。ツールポリシーと所有者専用ツールの一覧は、プロンプト内に示されるすべての送信者ではなく、現在のターンを開始したリクエスト元に基づいて選択されます。[リクエスト元スコープの制御とプロンプトコンテキスト](/ja-JP/gateway/security#requester-scoped-controls-and-prompt-context)を参照してください。

![グループメッセージのフロー](/images/groups-flow.svg)

目的別の設定：

| 目的                                         | 設定                                                |
| -------------------------------------------- | ---------------------------------------------------------- |
| すべてのグループを許可し、@メンションにのみ応答する | `groups: { "*": { requireMention: true } }`                |
| すべてのグループ応答を無効にする                    | `groupPolicy: "disabled"`                                  |
| 特定のグループのみ                                 | `groups: { "<group-id>": { ... } }`（`"*"` キーなし）         |
| グループ内で自分だけが起動できる                    | `groupPolicy: "allowlist"`、`groupAllowFrom: ["+1555..."]` |
| 複数のチャンネルで信頼済み送信者セットを再利用する    | `groupAllowFrom: ["accessGroup:operators"]`                |

再利用可能な送信者許可リストについては、[アクセスグループ](/ja-JP/channels/access-groups)を参照してください。

## セッションキー

- グループセッションは `agent:<agentId>:<channel>:group:<id>` セッションキーを使用します（ルーム／チャンネルは `agent:<agentId>:<channel>:channel:<id>` を使用します）。
- Telegram フォーラムトピックでは、各トピックが独自のセッションを持つように、グループ ID に `:topic:<threadId>` が追加されます。
- ダイレクトチャットはメインセッションを使用します（`session.dmScope` が設定されている場合は送信者ごとのセッション）。
- Heartbeat は設定された Heartbeat セッション（デフォルト：エージェントのメインセッション）で実行されます。グループセッションでは独自の Heartbeat は実行されません。

<a id="pattern-personal-dms-public-groups-single-agent"></a>

## パターン：個人 DM + 公開グループ（単一エージェント）

はい。「個人」トラフィックが **DM**、「公開」トラフィックが**グループ**であれば、適切に機能します。

理由：単一エージェントモードでは、通常 DM は**メイン**セッションキー（`agent:main:main`）に入りますが、グループは常に**非メイン**セッションキー（`agent:main:<channel>:group:<id>`）を使用します。`mode: "non-main"` でサンドボックス化を有効にすると、これらのグループセッションは設定されたサンドボックスバックエンドで実行され、メインの DM セッションはホスト上に残ります。バックエンドを選択しない場合、Docker がデフォルトです。

これにより、1 つのエージェントの「頭脳」（共有ワークスペースとメモリ）を使用しながら、2 つの実行形態を利用できます。

- **DM**：すべてのツール（ホスト）
- **グループ**：サンドボックス + 制限されたツール

<Note>
完全に分離されたワークスペース／ペルソナが必要な場合（「個人」と「公開」を決して混在させない場合）は、2 つ目のエージェントとバインディングを使用してください。[マルチエージェントルーティング](/ja-JP/concepts/multi-agent)を参照してください。
</Note>

<Tabs>
  <Tab title="DM はホスト上、グループはサンドボックス内">
    ```json5
    {
      agents: {
        defaults: {
          sandbox: {
            mode: "non-main", // グループ／チャンネルは非メイン -> サンドボックス化
            scope: "session", // 最も強力な分離（グループ／チャンネルごとに 1 つのコンテナ）
            workspaceAccess: "none",
          },
        },
      },
      tools: {
        sandbox: {
          tools: {
            // allow が空でない場合、その他すべてがブロックされます（deny が常に優先されます）。
            allow: ["group:messaging", "group:sessions"],
            deny: ["group:runtime", "group:fs", "group:ui", "nodes", "cron", "gateway"],
          },
        },
      },
    }
    ```
  </Tab>
  <Tab title="グループには許可リストに登録されたフォルダーのみを表示">
    「ホストへのアクセスなし」ではなく「グループはフォルダー X のみを表示可能」にするには、`workspaceAccess: "none"` を維持し、許可リストに登録したパスのみをサンドボックスにマウントします。

    ```json5
    {
      agents: {
        defaults: {
          sandbox: {
            mode: "non-main",
            scope: "session",
            workspaceAccess: "none",
            docker: {
              binds: [
                // hostPath:containerPath:mode
                "/home/user/FriendsShared:/data:ro",
              ],
            },
          },
        },
      },
    }
    ```

  </Tab>
</Tabs>

関連項目：

- 設定キーとデフォルト：[Gateway の設定](/ja-JP/gateway/config-agents#agentsdefaultssandbox)
- ツールがブロックされる理由のデバッグ：[サンドボックス、ツールポリシー、昇格の比較](/ja-JP/gateway/sandbox-vs-tool-policy-vs-elevated)
- バインドマウントの詳細：[サンドボックス化](/ja-JP/gateway/sandboxing#custom-bind-mounts)

## 表示ラベル

- UI ラベルでは、利用可能な場合は `displayName` が使用され、`<channel>:<token>` として書式設定されます。
- `#room` はルーム／チャンネル用に予約されています。グループチャットでは `g-<slug>` を使用します（小文字、スペース -> `-`、`#@+._-` は維持）。非常に長い不透明な ID は、完全なルート ID が UI に漏洩しないよう、安定したトークンに短縮されます。

## グループポリシー

チャンネルごとにグループ／ルームメッセージの処理方法を制御します。

```json5
{
  channels: {
    whatsapp: {
      groupPolicy: "disabled", // "open" | "disabled" | "allowlist"
      groupAllowFrom: ["+15551234567"],
    },
    telegram: {
      groupPolicy: "disabled",
      groupAllowFrom: ["123456789"], // Telegram の数値ユーザー ID（セットアップ時に @username を解決）
    },
    signal: {
      groupPolicy: "disabled",
      groupAllowFrom: ["+15551234567"],
    },
    imessage: {
      groupPolicy: "disabled",
      groupAllowFrom: ["chat_id:123"],
    },
    msteams: {
      groupPolicy: "disabled",
      groupAllowFrom: ["user@org.com"],
    },
    discord: {
      groupPolicy: "allowlist",
      guilds: {
        GUILD_ID: { channels: { help: { enabled: true } } },
      },
    },
    slack: {
      groupPolicy: "allowlist",
      channels: { "#general": { enabled: true } },
    },
    matrix: {
      groupPolicy: "allowlist",
      groupAllowFrom: ["@owner:example.org"],
      groups: {
        "!roomId:example.org": { enabled: true },
        "#alias:example.org": { enabled: true },
      },
    },
  },
}
```

| ポリシー        | 動作                                                     |
| ------------- | ------------------------------------------------------------ |
| `"open"`      | グループは許可リストを迂回します。メンション制御は引き続き適用されます。      |
| `"disabled"`  | すべてのグループメッセージを完全にブロックします。                           |
| `"allowlist"` | 設定された許可リストに一致するグループ／ルームのみを許可します。 |

<AccordionGroup>
  <Accordion title="チャンネルごとの注意事項">
    - `groupPolicy` はメンション制御（@メンションを必要とする機能）とは別です。
    - WhatsApp／Telegram／Signal／iMessage／Microsoft Teams／Zalo：`groupAllowFrom` を使用します（フォールバック：明示的な `allowFrom`）。
    - Signal：`groupAllowFrom` は、受信した Signal グループ ID または送信者の電話番号／UUID のいずれかに一致できます。
    - DM ペアリングの承認（`*-allowFrom` ストアのエントリ）は DM アクセスにのみ適用されます。グループ送信者の認可は、引き続きグループ許可リストで明示的に行います。
    - Discord：許可リストには `channels.discord.guilds.<id>.channels` を使用します。
    - Slack：許可リストには `channels.slack.channels` を使用します。
    - Matrix：許可リストには `channels.matrix.groups` を使用します。ルーム ID（`!room:server`）またはエイリアス（`#alias:server`）を使用してください。ルーム名キーは `channels.matrix.dangerouslyAllowNameMatching: true` がある場合にのみ一致し、解決できないエントリは実行時に無視されます。送信者を制限するには `channels.matrix.groupAllowFrom` を使用します。ルームごとの `users` 許可リストにも対応しています。
    - グループ DM は個別に制御されます（`channels.discord.dm.*`、`channels.slack.dm.*`：`groupEnabled`、`groupChannels`）。
    - Telegram：送信者の許可リストには数値ユーザー ID のみを指定できます（`"123456789"`。`telegram:`／`tg:` プレフィックスは大文字と小文字を区別せずに削除されます）。`@username` エントリは実行時に一致せず、警告がログに記録されます。セットアップ時に `@username` が ID に解決されます。負のチャット ID は送信者の許可リストではなく、`channels.telegram.groups` に指定します。
    - デフォルトは `groupPolicy: "allowlist"` です。グループ許可リストが空の場合、グループメッセージはブロックされます。
    - 実行時の安全性：プロバイダーブロックが完全に欠落している場合（`channels.<provider>` がない場合）、グループポリシーは `channels.defaults.groupPolicy` を継承せず、フェイルクローズで `allowlist` になります。また、Gateway はアカウントごとにフォールバックを一度だけログに記録します。

  </Accordion>
</AccordionGroup>

簡単な概念モデル（グループメッセージの評価順序）：

<Steps>
  <Step title="groupPolicy">
    `groupPolicy`（open／disabled／allowlist）。
  </Step>
  <Step title="グループ許可リスト">
    グループ許可リスト（`*.groups`、`*.groupAllowFrom`、チャンネル固有の許可リスト）。
  </Step>
  <Step title="メンション制御">
    メンション制御（`requireMention`、`/activation`）。
  </Step>
</Steps>

## メンション制御（デフォルト）

グループごとに上書きされない限り、グループメッセージにはメンションが必要です。デフォルトはサブシステムごとに `*.groups."*"` で設定されます。

対応する暗黙的なメンション判定要素は、チャンネルごとに異なります。

| 判定要素                  | 現在の組み込み生成元                       |
| --------------------- | ------------------------------------------------ |
| ボットへの返信      | Discord、Microsoft Teams、QQBot、Slack、Telegram |
| ボットの引用      | WhatsApp、Zalo personal                          |
| ボットがスレッドに参加済み | Mattermost、Slack、Tlon                          |

各判定要素は、チャンネルがそれを生成する場合、デフォルトで有効です。その判定要素がメンション制御を迂回しないようにするには、対応する `implicitMentions` フラグを `false` に設定します。ネイティブの明示的メンションには影響しません。その判定要素を生成しないチャンネルでは、フラグは効果を持ちません。

```json5
{
  channels: {
    whatsapp: {
      groups: {
        "*": { requireMention: true },
        "123@g.us": { requireMention: false },
      },
    },
    telegram: {
      groups: {
        "*": { requireMention: true },
        "123456789": { requireMention: false },
      },
    },
    imessage: {
      groups: {
        "*": { requireMention: true },
        "123": { requireMention: false },
      },
    },
  },
  agents: {
    entries: {
      main: {
        groupChat: {
          mentionPatterns: ["@openclaw", "openclaw", "\\+15555550123"],
          historyLimit: 50,
        },
      },
    },
  },
}
```

## 設定済みメンションパターンのスコープ

設定済みの `mentionPatterns` は、正規表現によるフォールバックトリガーです。プラットフォームがネイティブのボットメンションを提供しない場合や、`openclaw:` のようなプレーンテキストをメンションとして扱いたい場合に使用します。ネイティブのプラットフォームメンションは別扱いです。Discord、Slack、Telegram、Matrix、Signal、またはその他のチャンネルが、メッセージ内でボットが明示的にメンションされたことを証明できる場合、設定済みの正規表現パターンが拒否されていても、そのネイティブメンションは引き続きトリガーになります。

デフォルトでは、設定済みのメンションパターンは、チャンネルがプロバイダーと会話の判定情報をメンション検出に渡すすべての場所で適用されます。広範なパターンによってすべてのグループでエージェントが起動することを防ぐには、`channels.<channel>.mentionPatterns` を使用してチャンネルごとにスコープを設定します。

正規表現のメンションパターンをチャンネルでデフォルト無効にし、`allowIn` を使用して特定のルームだけを有効にする場合は、`mode: "deny"` を使用します。

```json5
{
  messages: {
    groupChat: {
      mentionPatterns: ["\\bopenclaw\\b", "\\bops bot\\b"],
    },
  },
  channels: {
    slack: {
      mentionPatterns: {
        mode: "deny",
        allowIn: ["C0123OPS"],
      },
    },
  },
}
```

正規表現のメンションパターンを広範に適用し、`denyIn` を使用してノイズの多いルームで無効にする場合は、デフォルトの `mode: "allow"` を使用します（または `mode` を省略します）。

```json5
{
  messages: {
    groupChat: {
      mentionPatterns: ["\\bopenclaw\\b"],
    },
  },
  channels: {
    telegram: {
      mentionPatterns: {
        denyIn: ["-1001234567890", "-1001234567890:topic:42"],
      },
    },
  },
}
```

ポリシーの解決：

| フィールド           | 効果                                                                                                                |
| --------------- | --------------------------------------------------------------------------------------------------------------------- |
| `mode: "allow"` | 会話 ID が `denyIn` に含まれていない限り、正規表現のメンションパターンは有効です。これがデフォルトです。                    |
| `mode: "deny"`  | 会話 ID が `allowIn` に含まれていない限り、正規表現のメンションパターンは無効です。                                       |
| `allowIn`       | deny モードで正規表現のメンションパターンを有効にする会話 ID。                                               |
| `denyIn`        | 正規表現のメンションパターンを無効にする会話 ID。同じ ID が両方に含まれる場合、`denyIn` が `allowIn` より優先されます。 |

現在対応しているスコープ付き正規表現ポリシー：

| チャンネル  | `allowIn`／`denyIn` で使用する ID                             |
| -------- | ------------------------------------------------------------ |
| Discord  | Discord チャンネル ID。                                         |
| Matrix   | Matrix ルーム ID。                                             |
| Slack    | Slack チャンネル ID。                                           |
| Telegram | グループチャット ID、またはフォーラムトピックの場合は `chatId:topic:threadId`。 |
| WhatsApp | `123@g.us` などの WhatsApp 会話 ID。                |

複数のアカウントに対応するチャンネルでは、アカウントレベルのチャンネル設定で `channels.<channel>.accounts.<accountId>.mentionPatterns` に同じポリシーを設定できます。そのアカウントでは、アカウントポリシーがトップレベルのチャンネルポリシーより優先されます。

<AccordionGroup>
  <Accordion title="メンション制御の注意事項">
    - `mentionPatterns` は、大文字と小文字を区別しない安全な正規表現パターンです。無効なパターンや安全でない入れ子の反復形式は、警告とともに無視されます。
    - パターンの優先順位：`agents.entries.*.groupChat.mentionPatterns`（複数のエージェントがグループを共有する場合に便利）が `messages.groupChat.mentionPatterns` より優先されます。どちらも設定されていない場合、パターンはエージェントのアイデンティティ名／絵文字から生成されます。
    - メンション制御は、メンション検出が可能な場合（ネイティブメンション、または `mentionPatterns` が設定されている場合）にのみ適用されます。
    - グループまたは送信者を許可リストに登録しても、メンション制御は無効になりません。すべてのメッセージをトリガーにする場合は、そのグループの `requireMention` を `false` に設定します。
    - グループチャットの自動プロンプトコンテキストには、解決済みの無言返信指示がターンごとに含まれます。ワークスペースファイルで `NO_REPLY` の仕組みを重複させないでください。
    - 自動的な無言返信が許可されたグループでは、内容が空のモデルターンまたは推論のみのモデルターンを、`NO_REPLY` と同等の無言として扱います。ダイレクトチャットが `NO_REPLY` の指示を受け取ることはありません。また、メッセージツールのみを使用するグループ返信は、`message(action=send)` を呼び出さないことで無言を維持します。
    - 常時有効なグループ内の周辺会話には、デフォルトでユーザーリクエストのセマンティクスが使用されます。代わりに静かなコンテキストとして送信するには、`messages.groupChat.unmentionedInbound: "room_event"` を設定します。セットアップ例については、[アンビエントルームイベント](/ja-JP/channels/ambient-room-events)を参照してください。
    - ルームイベントは偽のユーザーリクエストとして保存されず、メッセージツールを使用しないルームイベントの非公開アシスタントテキストがチャット履歴として再生されることもありません。
    - Discord のデフォルトは `channels.discord.guilds."*"` にあります（ギルド／チャンネルごとに上書き可能）。
    - グループ履歴コンテキストは、すべてのチャンネルで統一的にラップされます。メンション制御されたグループでは、スキップされた保留中のメッセージが保持されます。常時有効なグループでも、チャンネルが対応している場合は、最近処理されたルームメッセージを保持できます。グローバルデフォルトには `messages.groupChat.historyLimit`、上書きには `channels.<channel>.historyLimit`（または `channels.<channel>.accounts.*.historyLimit`）を使用します。無効にするには `0` を設定します。

  </Accordion>
</AccordionGroup>

## グループ／チャンネルのツール制限（任意）

一部のチャンネル設定では、**特定のグループ／ルーム／チャンネル内**で利用できるツールを制限できます。

- `tools`：グループ全体でツールを許可／拒否します（`allow`、`alsoAllow`、`deny`。拒否が優先されます）。
- `toolsBySender`：グループ内の送信者ごとの上書き。明示的なキープレフィックス（`channel:<channelId>:<senderId>`、`id:<senderId>`、`e164:<phone>`、`username:<handle>`、`name:<displayName>`、および `"*"` ワイルドカード）を使用します。チャンネル ID には OpenClaw の正規チャンネル ID を使用します。`teams` などのエイリアスは `msteams` に正規化されます。プレフィックスのない従来のキーも引き続き受け付けられますが、`id:` としてのみ照合され、非推奨警告がログに記録されます。

解決順序（最も具体的なものが優先）：

<Steps>
  <Step title="グループの toolsBySender">
    グループ／チャンネルの `toolsBySender` の一致。
  </Step>
  <Step title="グループの tools">
    グループ／チャンネルの `tools`。
  </Step>
  <Step title="デフォルトの toolsBySender">
    デフォルト（`"*"`）の `toolsBySender` の一致。
  </Step>
  <Step title="デフォルトの tools">
    デフォルト（`"*"`）の `tools`。
  </Step>
</Steps>

例（Telegram）：

```json5
{
  channels: {
    telegram: {
      groups: {
        "*": { tools: { deny: ["exec"] } },
        "-1001234567890": {
          tools: { deny: ["exec", "read", "write"] },
          toolsBySender: {
            "id:123456789": { alsoAllow: ["exec"] },
          },
        },
      },
    },
  },
}
```

<Note>
グループ／チャンネルのツール制限は、グローバル／エージェントのツールポリシーに追加して適用されます（deny が常に優先されます）。一部のチャンネルでは、ルーム／チャンネルに異なるネスト構造を使用します（例: Discord `guilds.*.channels.*`、Slack `channels.*`、Microsoft Teams `teams.*.channels.*`）。
</Note>

## グループの許可リスト

`channels.whatsapp.groups`、`channels.telegram.groups`、または `channels.imessage.groups` が設定されている場合、そのキーがグループの許可リストとして機能します。デフォルトのメンション動作を設定しつつ、すべてのグループを許可するには `"*"` を使用します。

<Warning>
よくある混同: DM のペアリング承認は、グループの認可とは異なります。DM のペアリングをサポートするチャンネルでは、ペアリングストアが許可するのは DM のみです。グループコマンドには、`groupAllowFrom` などの設定上の許可リスト、またはそのチャンネルで文書化された設定フォールバックによる、グループ送信者の明示的な認可が引き続き必要です。
</Warning>

一般的な設定例（コピー＆ペースト用）:

<Tabs>
  <Tab title="グループへの返信をすべて無効化">
    ```json5
    {
      channels: { whatsapp: { groupPolicy: "disabled" } },
    }
    ```
  </Tab>
  <Tab title="特定のグループのみ許可（WhatsApp）">
    ```json5
    {
      channels: {
        whatsapp: {
          groups: {
            "123@g.us": { requireMention: true },
            "456@g.us": { requireMention: false },
          },
        },
      },
    }
    ```
  </Tab>
  <Tab title="すべてのグループを許可するがメンションを必須化">
    ```json5
    {
      channels: {
        whatsapp: {
          groups: { "*": { requireMention: true } },
        },
      },
    }
    ```
  </Tab>
  <Tab title="オーナーのみがトリガー可能（WhatsApp）">
    ```json5
    {
      channels: {
        whatsapp: {
          groupPolicy: "allowlist",
          groupAllowFrom: ["+15551234567"],
          groups: { "*": { requireMention: true } },
        },
      },
    }
    ```
  </Tab>
</Tabs>

## アクティベーション（オーナーのみ）

グループのオーナーは、単独のメッセージを使用してグループごとのアクティベーションを切り替えられます:

- `/activation mention`
- `/activation always`

`/activation` はコアのオーナー制限付きコマンドであり、グループチャットにのみ適用されます。オーナーとは、送信者が `commands.ownerAllowFrom` と一致することを意味します。チャンネルの `allowFrom` リストは、通常のチャンネルアクセスとコマンドアクセスのみを制御します。保存されたモードは、それを参照するチャンネル（Google Chat、QQBot、Telegram、WhatsApp）で、そのグループの `requireMention` より優先されます。また、グループのシステムプロンプトの導入部には、すべてのチャンネルで有効なモードが反映されます。

## コンテキストフィールド

グループの受信ペイロードでは、以下が設定されます:

- `ChatType=group`
- `GroupSubject`（判明している場合）
- `GroupMembers`（判明している場合）
- `WasMentioned`（メンション制御の結果）
- Telegram のフォーラムトピックには、`MessageThreadId` と `IsForum` も含まれます。

新しいグループセッションの最初のターン（および `/activation` の変更後）では、エージェントのシステムプロンプトにグループ向けの導入部が含まれます。これは、人間らしく応答すること、空行を最小限にして通常のチャットの間隔に従うこと、リテラルの `\n` シーケンスを入力しないことをモデルに促します。宣言されたテーブルモードでネイティブテーブルまたは未加工のテーブルが保持されないチャンネルでは、Markdown テーブルの使用も控えるよう促します。チャンネル由来のグループ名と参加者ラベルは、インラインのシステム指示ではなく、フェンスで囲まれた信頼されていないメタデータとしてレンダリングされます。

## iMessage 固有の事項

- ルーティングまたは許可リストへの登録には、`chat_id:<id>` を優先して使用します。
- チャットの一覧表示: `imsg chats --limit 20`。
- グループへの返信は、常に同じ `chat_id` に返されます。

## WhatsApp のシステムプロンプト

グループおよびダイレクトプロンプトの解決、ワイルドカードの動作、アカウントによる上書きのセマンティクスなど、WhatsApp の標準的なシステムプロンプト規則については、[WhatsApp](/ja-JP/channels/whatsapp#system-prompts) を参照してください。

## WhatsApp 固有の事項

WhatsApp 固有の動作（履歴の挿入、メンション処理の詳細）については、[グループメッセージ](/ja-JP/channels/group-messages)を参照してください。

## 関連項目

- [ブロードキャストグループ](/ja-JP/channels/broadcast-groups)
- [チャンネルルーティング](/ja-JP/channels/channel-routing)
- [グループメッセージ](/ja-JP/channels/group-messages)
- [ペアリング](/ja-JP/channels/pairing)
