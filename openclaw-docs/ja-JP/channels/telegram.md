---
read_when:
    - Telegram の機能または Webhook に取り組む
summary: Telegram ボットのサポート状況、機能、設定
title: Telegram
x-i18n:
    generated_at: "2026-07-26T09:31:12Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: f34067478f4a5a71ed8f18503234b4cfcf573ac740aa887b65d13d0e1f09ba54
    source_path: channels/telegram.md
    workflow: 16
---

grammY を介したボットの DM とグループに対応し、本番環境で利用できます。デフォルトのトランスポートはロングポーリングで、Webhook モードは任意です。

<CardGroup cols={3}>
  <Card title="ペアリング" icon="link" href="/ja-JP/channels/pairing">
    Telegram のデフォルトの DM ポリシーはペアリングです。
  </Card>
  <Card title="チャンネルのトラブルシューティング" icon="wrench" href="/ja-JP/channels/troubleshooting">
    チャンネル横断の診断と修復手順。
  </Card>
  <Card title="Gateway の設定" icon="settings" href="/ja-JP/gateway/configuration">
    チャンネル設定の完全なパターンと例。
  </Card>
</CardGroup>

## クイックセットアップ

<Steps>
  <Step title="BotFather でボットトークンを作成する">
    どちらの手順でも、最後に OpenClaw に貼り付けるトークンを取得できます。いずれかを選択してください。

    - **チャットでの手順**：Telegram を開き、**@BotFather** とチャットし（ハンドルが正確に `@BotFather` であることを確認）、`/newbot` を実行して指示に従い、トークンを保存します。
    - **Web での手順**：[BotFather の Web アプリ](https://t.me/BotFather?startapp)を開きます。これは [web.telegram.org](https://web.telegram.org) を含むすべての Telegram クライアントで動作します。UI でボットを作成し、そのトークンをコピーします。

  </Step>

  <Step title="トークンと DM ポリシーを設定する">

```json5
{
  channels: {
    telegram: {
      enabled: true,
      botToken: "123:abc",
      dmPolicy: "pairing",
      groups: { "*": { requireMention: true } },
    },
  },
}
```

    環境変数によるフォールバック：`TELEGRAM_BOT_TOKEN`（デフォルトアカウントのみ。名前付きアカウントでは `botToken` または `tokenFile` を使用する必要があります）。
    Telegram は `openclaw channels login telegram` を使用しません。設定または環境変数にトークンを指定してから、Gateway を起動してください。

  </Step>

  <Step title="Gateway を起動して最初の DM を承認する">

```bash
openclaw gateway
openclaw pairing list telegram
openclaw pairing approve telegram <CODE>
```

    ペアリングコードは 1 時間後に期限切れになります。

  </Step>

  <Step title="ボットをグループに追加する">
    ボットをグループに追加し、グループアクセスに必要な次の 2 つの ID を取得します。

    - `allowFrom` / `groupAllowFrom` に使用する Telegram ユーザー ID
    - `channels.telegram.groups` 配下のキーとして使用する Telegram グループチャット ID

    グループチャット ID は、`openclaw logs --follow`、転送されたメッセージの ID を取得するボット、または Bot API の `getUpdates` から取得します。グループを許可した後は、`/whoami@<bot_username>` でユーザー ID とグループ ID を確認できます。

    `-100` で始まる負のスーパーグループ ID はグループチャット ID です。`groupAllowFrom` ではなく `channels.telegram.groups` 配下に指定します。

  </Step>
</Steps>

<Note>
トークンの解決ではアカウントが考慮されます。`tokenFile`、`botToken`、環境変数の順に優先され、設定は常に `TELEGRAM_BOT_TOKEN` より優先されます（後者はデフォルトアカウントでのみ解決されます）。正常に起動した後、OpenClaw はボットの ID 情報を最大 24 時間キャッシュするため、再起動時には追加の `getMe` 呼び出しが省略されます。トークンを変更または削除すると、このキャッシュは消去されます。
</Note>

## Telegram 側の設定

<AccordionGroup>
  <Accordion title="プライバシーモードとグループでの可視性">
    Telegram ボットではデフォルトで **Privacy Mode** が有効になっており、受信できるグループメッセージが制限されます。

    すべてのグループメッセージを受信するには、次のいずれかを行います。

    - `/setprivacy` でプライバシーモードを無効にする
    - ボットをグループ管理者にする

    プライバシーモードを切り替えた後は、Telegram に変更を適用させるため、各グループからボットを削除して再度追加してください。

  </Accordion>

  <Accordion title="グループ権限">
    管理者ステータスは Telegram のグループ設定で制御します。管理者ボットはすべてのグループメッセージを受信するため、常時動作するグループ機能に役立ちます。
  </Accordion>

  <Accordion title="便利な BotFather の切り替え項目">

    - `/setjoingroups` — グループへの追加を許可または拒否
    - `/setprivacy` — グループでの可視性の動作

    チャットコマンドより UI を使いたい場合は、[BotFather の Web アプリ](https://t.me/BotFather?startapp)でも同じ設定を利用できます。

  </Accordion>
</AccordionGroup>

## ダッシュボード Mini App

ボットとの DM で `/dashboard` を実行すると、Telegram 内で OpenClaw ダッシュボードが開きます。

要件：

- 公開 HTTPS Mini App URL 用の `gateway.tailscale.mode: "serve"` または `"funnel"`
- 数値形式の Telegram ユーザー ID が、選択したアカウントの有効な `allowFrom` または `commands.ownerAllowFrom` に含まれている必要があります。
- DM を使用してください。グループでは、`/dashboard` は `open this in a DM with the bot` と返信し、ボタンを送信しません。
- Docker インストール：Serve/Funnel モードでは、Gateway が `tailscaled` と同じ場所のループバックにバインドされる必要がありますが、公開ポートを使用するブリッジネットワークではこの要件を満たせません。Gateway コンテナを `network_mode: host` で実行し、ホストの `tailscaled` ソケット（`/var/run/tailscale`）と `tailscale` CLI をコンテナにマウントしてください。

Mini App は Tailscale 専用の v1 パスであり、Telegram Web iframe には対応していません。

## アクセス制御と有効化

### グループでのボット ID

グループとフォーラムトピックでは、設定されたボットハンドルへの明示的なメンション（たとえば `@my_bot`）が、エージェントのペルソナ名と Telegram ユーザー名が異なる場合でも、選択された OpenClaw エージェント宛てとして扱われます。無関係なメッセージにはグループの無応答ポリシーが引き続き適用されますが、ボットハンドル自体が「別の誰か」として扱われることはありません。

<Tabs>
  <Tab title="DM ポリシー">
    `channels.telegram.dmPolicy` はダイレクトメッセージへのアクセスを制御します。

    - `pairing`（デフォルト）
    - `allowlist`（`allowFrom` に少なくとも 1 つの送信者 ID が必要）
    - `open`（`allowFrom` に `"*"` が含まれている必要があります）
    - `disabled`

    `allowFrom: ["*"]` を指定した `dmPolicy: "open"` では、ボットのユーザー名を見つけるか推測した任意の Telegram アカウントがボットに命令できます。厳しく制限されたツールを備え、意図的に公開するボットにのみ使用してください。所有者が 1 人のボットでは、数値形式のユーザー ID とともに `allowlist` を使用してください。

    `channels.telegram.allowFrom` は数値形式の Telegram ユーザー ID を受け付けます。`telegram:` / `tg:` プレフィックスも受け付けられ、正規化されます。
    複数アカウントの設定では、制限的なトップレベルの `channels.telegram.allowFrom` が安全境界となります。アカウントレベルの `allowFrom: ["*"]` を指定しても、マージ後の有効な許可リストに明示的なワイルドカードが引き続き含まれていない限り、そのアカウントは公開されません。
    `allowFrom` が空の `dmPolicy: "allowlist"` はすべての DM をブロックし、設定検証で拒否されます。
    セットアップでは数値形式のユーザー ID のみを要求します。古いセットアップで作成された `@username` 許可リストエントリが設定に含まれている場合は、`openclaw doctor --fix` を実行して数値 ID に解決してください（ベストエフォート。Telegram ボットトークンが必要です）。
    以前ペアリングストアの許可リストファイルを利用していた場合、`openclaw doctor --fix` を使用すると、許可リストを使うフロー向けにエントリを `channels.telegram.allowFrom` へ復元できます（たとえば、`dmPolicy: "allowlist"` に明示的な ID がまだない場合）。

    所有者が 1 人のボットでは、以前のペアリング承認に依存せず、明示的な数値形式の `allowFrom` ID とともに `dmPolicy: "allowlist"` を使用することを推奨します。

    よくある誤解：DM のペアリング承認は、「この送信者があらゆる場所で認可されている」という意味ではありません。ペアリングで付与されるのは DM へのアクセスのみです。コマンド所有者がまだ存在しない場合、最初に承認されたペアリングによって `commands.ownerAllowFrom` も設定され、所有者専用コマンドと exec 承認に明示的なオペレーターアカウントが割り当てられます。グループ送信者の認可は、引き続き設定内の明示的な許可リストによって決まります。
    1 つの ID で DM とグループコマンドの両方に認可されるには、数値形式の Telegram ユーザー ID を `channels.telegram.allowFrom` に指定し、所有者専用コマンドを使用する場合は `commands.ownerAllowFrom` に `telegram:<your user id>` が含まれていることを確認してください。

    ### Telegram ユーザー ID を確認する

    より安全な方法（サードパーティ製ボット不要）：ボットに DM を送り、`openclaw logs --follow` を実行して、`from.id` を確認します。

    公式 Bot API を使用する方法：

```bash
curl "https://api.telegram.org/bot<bot_token>/getUpdates"
```

    サードパーティを使用する方法（プライバシーは低下）：`@userinfobot` または `@getidsbot`。

  </Tab>

  <Tab title="グループポリシーと許可リスト">
    次の 2 つの制御が同時に適用されます。

    1. **許可するグループ**（`channels.telegram.groups`）
       - `groups` の設定なし、`groupPolicy: "open"`：すべてのグループがグループ ID チェックを通過
       - `groups` の設定なし、`groupPolicy: "allowlist"`（デフォルト）：`groups` エントリ（または `"*"`）を追加するまで、すべてのグループをブロック
       - `groups` を設定済み：許可リストとして機能（明示的な ID または `"*"`）

    2. **グループ内で許可する送信者**（`channels.telegram.groupPolicy`）
       - `open` / `allowlist`（デフォルト）/ `disabled`

    `groupAllowFrom` はグループの送信者をフィルタリングします。未設定の場合、Telegram は `allowFrom` にフォールバックします（ペアリングストアではありません。グループ送信者の認証が DM のペアリングストア承認を継承することはなく、これは `2026.2.25` 以降の安全境界です）。
    `groupAllowFrom` のエントリには数値形式の Telegram ユーザー ID を指定してください（`telegram:` / `tg:` プレフィックスは正規化されます）。数値以外のエントリは無視されます。グループまたはスーパーグループのチャット ID はここに指定しないでください。負のチャット ID は `channels.telegram.groups` 配下に指定します。
    所有者が 1 人のボットでの実用的なパターン：ユーザー ID を `channels.telegram.allowFrom` に設定し、`groupAllowFrom` は未設定のままにして、対象グループを `channels.telegram.groups` 配下で許可します。
    設定に `channels.telegram` がまったく存在しない場合、`channels.defaults.groupPolicy` が明示的に設定されていない限り、ランタイムはフェイルクローズの `groupPolicy="allowlist"` をデフォルトで使用します。

    所有者専用のグループ設定：

```json5
{
  channels: {
    telegram: {
      enabled: true,
      dmPolicy: "pairing",
      allowFrom: ["<YOUR_TELEGRAM_USER_ID>"],
      groupPolicy: "allowlist",
      groups: {
        "<GROUP_CHAT_ID>": {
          requireMention: true,
        },
      },
    },
  },
}
```

    グループから `@<bot_username> ping` を使用してテストします。`requireMention: true` の間は、通常のグループメッセージでボットが起動することはありません。

    特定の 1 つのグループですべてのメンバーを許可する：

```json5
{
  channels: {
    telegram: {
      groups: {
        "-1001234567890": {
          groupPolicy: "open",
          requireMention: false,
        },
      },
    },
  },
}
```

    特定の 1 つのグループで特定のユーザーのみを許可する：

```json5
{
  channels: {
    telegram: {
      groups: {
        "-1001234567890": {
          requireMention: true,
          allowFrom: ["8734062810", "745123456"],
        },
      },
    },
  },
}
```

    <Warning>
      よくある間違い：`groupAllowFrom` はグループの許可リストではありません。

      - 負の Telegram グループ／スーパーグループチャット ID（`-1001234567890`）は `channels.telegram.groups` 配下に指定します。
      - Telegram ユーザー ID（`8734062810`）は、許可されたグループ内でボットを起動できるユーザーを制限するため、`groupAllowFrom` 配下に指定します。
      - 許可されたグループの任意のメンバーがボットとやり取りできるようにする場合にのみ、`groupAllowFrom: ["*"]` を使用します。

    </Warning>

  </Tab>

  <Tab title="メンションの動作">
    デフォルトでは、グループで返信させるにはメンションが必要です。メンションには次のいずれかを使用できます。

    - ネイティブの `@botusername` メンション
    - `agents.entries.*.groupChat.mentionPatterns` または `messages.groupChat.mentionPatterns` のメンションパターン

    セッションレベルの切り替え（状態のみで、永続化されません）：`/activation always`、`/activation mention`。永続化するには設定を使用します。

```json5
{
  channels: {
    telegram: {
      groups: {
        "*": { requireMention: false },
      },
    },
  },
}
```

    グループ履歴のコンテキストは常に有効で、`historyLimit` によって上限が設定されます。グループ履歴ウィンドウを無効にするには、`channels.telegram.historyLimit: 0` を設定します。`openclaw doctor --fix` は廃止された `includeGroupHistoryContext` キーを削除します。

    グループチャット ID の取得方法：グループメッセージを `@userinfobot` / `@getidsbot` に転送する、`openclaw logs --follow` から `chat.id` を確認する、Bot API の `getUpdates` を調べる、または（グループを許可した後に）`/whoami@<bot_username>` を実行します。

  </Tab>
</Tabs>

## ランタイムの動作

- Telegram は Gateway プロセス内で動作します。
- ルーティングは決定論的です。Telegram から受信したメッセージへの返信は Telegram に返されます（モデルがチャンネルを選択することはありません）。
- 受信メッセージは、返信メタデータ、メディアプレースホルダー、Gateway が確認した返信に対する永続化された返信チェーンコンテキストを含む、共有チャンネルエンベロープへ正規化されます。
- グループセッションはグループ ID ごとに分離されます。フォーラムトピックには `:topic:<threadId>` が追加されます。
- DM メッセージには `message_thread_id` を含めることができ、OpenClaw は返信用にこれを保持します。DM トピックセッションが分割されるのは、Telegram `getMe` がボットについて `has_topics_enabled: true` を報告した場合のみです。それ以外の DM はフラットセッションのままです。
- ロングポーリングには、チャット単位およびスレッド単位の順序制御を備えた grammY runner を使用します。Runner のシンク並行処理には `agents.defaults.maxConcurrent` を使用します。
- 複数アカウントの起動では、並行する `getMe` プローブ数を制限し、大規模なボット群ですべてのアカウントプローブが一斉に実行されないようにします。
- 各 Gateway プロセスはロングポーリングを保護し、一度に 1 つのアクティブなポーラーだけがボットトークンを使用できるようにします。`getUpdates` の 409 競合が継続する場合、同じトークンを使用している別の OpenClaw Gateway、スクリプト、または外部ポーラーが存在します。
- ポーリングウォッチドッグは、`getUpdates` の生存確認が完了しない状態が 120 秒続くと再起動します。
- Telegram Bot API は既読通知をサポートしていません（`sendReadReceipts` は適用されません）。

<Note>
  `channels.telegram.dm.threadReplies` と `channels.telegram.direct.<chatId>.threadReplies` は削除されました。アップグレード後も設定にこれらのキーが残っている場合は、`openclaw doctor --fix` を実行してください。DM トピックのルーティングは、Telegram `getMe.has_topics_enabled`（BotFather のスレッドモードで制御）に従うようになりました。トピックが有効なボットでは、Telegram が `message_thread_id` を送信するとスレッド単位の DM セッションを使用し、それ以外の DM はフラットセッションのままです。
</Note>

## 機能リファレンス

<AccordionGroup>
  <Accordion title="ライブストリームプレビュー（メッセージ編集）">
    OpenClaw は、ダイレクトチャット、グループ、トピックで部分的な返信をリアルタイムにストリーミングします。プレビューメッセージを送信してから `editMessageText` を繰り返し実行し、同じ場所で最終化します。

    - `channels.telegram.streaming` は `off | partial | block | progress` です（デフォルト: `partial`）
    - 短い初期回答のプレビューはデバウンスされ、実行がまだアクティブな場合は上限付きの遅延後に実体化されます
    - `progress` はツールの進行状況用に編集可能なステータス下書きを 1 つ維持し、ツールの進行状況より先に回答アクティビティが届いた場合は安定したステータスラベルを表示し、完了時にそれを消去して、最終回答を通常のメッセージとして送信します
    - `streaming.preview.toolProgress` は、ツールや進行状況の更新で同じ編集済みプレビューメッセージを再利用するかどうかを制御します（デフォルト: プレビューストリーミングが有効な場合は `true`）
    - `streaming.preview.commandText` は、これらの行に含めるコマンドや実行の詳細を制御します。`raw`（デフォルト）または `status`（ツールラベルのみ）
    - `streaming.progress.commentary`（デフォルト: `false`）を使用すると、一時的な進行状況下書きにアシスタントの解説や前置きテキストを含められます
    - 従来の `channels.telegram.streamMode`、ブール値の `streaming`、廃止されたネイティブ下書きプレビューキーは検出されます。移行するには `openclaw doctor --fix` を実行してください

    ツール進行状況行とは、ツールの実行中に表示される短いステータス更新です（コマンド実行、ファイル読み取り、計画更新、パッチ概要、app-server モードでの Codex の前置きや解説）。Telegram ではデフォルトで有効です（`v2026.4.22` 以降のリリース済み動作と一致します）。

    回答プレビューの編集を維持しつつ、ツール進行状況行を非表示にする場合:

    ```json
    {
      "channels": {
        "telegram": {
          "streaming": {
            "mode": "partial",
            "preview": { "toolProgress": false }
          }
        }
      }
    }
    ```

    ツール進行状況を表示したまま、コマンドや実行のテキストを非表示にする場合:

    ```json
    {
      "channels": {
        "telegram": {
          "streaming": {
            "mode": "partial",
            "preview": { "commandText": "status" }
          }
        }
      }
    }
    ```

    `progress` モードでは、最終回答をそのメッセージへ編集して入れることなく、ツールの進行状況を表示します。コマンドテキストのポリシーは `streaming.progress` の下に配置します。

    ```json
    {
      "channels": {
        "telegram": {
          "streaming": {
            "mode": "progress",
            "progress": {
              "toolProgress": true,
              "commandText": "status"
            }
          }
        }
      }
    }
    ```

    `streaming.mode: "off"` はプレビュー編集を無効にし、一般的なツールや進行状況のメッセージを独立したステータスメッセージとして送信せず抑制します。承認プロンプト、メディア、エラーは引き続き通常の最終配信経路を使用します。`streaming.preview.toolProgress: false` は回答プレビューの編集のみを維持します。

    <Note>
      選択範囲を引用した返信は例外です。`replyToMode` が `first`、`all`、または `batched` で、受信メッセージに選択された引用テキストが含まれている場合、OpenClaw は回答プレビューを編集する代わりに、Telegram のネイティブ引用返信経路で最終回答を送信するため、そのターンでは `streaming.preview.toolProgress` にステータス行を表示できません。選択された引用テキストのない現在のメッセージへの返信は、引き続きストリーミングされます。ネイティブ引用返信よりツール進行状況の表示を優先する場合は `replyToMode: "off"` を設定し、このトレードオフを許容する場合は `streaming.preview.toolProgress: false` を設定してください。
    </Note>

    テキストのみの返信の場合、短いプレビューは同じ場所で最終編集されます。複数メッセージに分割される長い最終回答では、プレビューを最初のチャンクとして再利用し、残りだけを送信します。進行状況モードの最終回答ではステータス下書きを消去し、通常の最終配信を使用します。完了が確認される前に最終編集が失敗した場合、OpenClaw は通常の最終配信へフォールバックし、古いプレビューをクリーンアップします。複雑な返信（メディアペイロード）の場合、OpenClaw は常に通常の最終配信へフォールバックし、プレビューをクリーンアップします。

    プレビューストリーミングとブロックストリーミングは相互排他的です。ブロックストリーミングが明示的に有効な場合、OpenClaw は二重ストリーミングを避けるためプレビューストリームをスキップします。

    推論: `/reasoning stream` は生成中の推論をライブプレビューへストリーミングし、最終配信後に推論プレビューを削除します（表示したままにするには `/reasoning on` を使用します）。最終回答は推論テキストを含めずに送信されます。

  </Accordion>

  <Accordion title="リッチメッセージの書式設定">
    送信テキストでは、現在の各クライアントで読み取れる標準の Telegram HTML メッセージをデフォルトで使用します。太字、斜体、リンク、コード、スポイラー、引用に対応しますが、Bot API 10.2 のリッチ専用ブロック（ネイティブテーブル、詳細、リッチメディア、数式）は使用しません。

    Bot API 10.2 のリッチメッセージを有効にする場合:

```json5
{
  channels: {
    telegram: {
      richMessages: true,
    },
  },
}
```

    有効にすると、このボットまたはアカウントでリッチメッセージが利用可能であることがエージェントに通知されます（サポートされている Markdown と HTML アイランドのオーサリング規約を含む）。Markdown テキストは OpenClaw の Markdown IR を介して、型付きの Bot API 10.2 リッチブロック（見出し、テーブル、詳細、チェックリスト、リッチメディア、数式、地図、コラージュ）としてレンダリングされます。メディアのキャプションには引き続き Telegram HTML キャプションを使用します（リッチメッセージはキャプションを置き換えず、キャプションの上限は 1024 文字です）。

    これにより、モデルのテキストが Telegram のリッチ Markdown 記号から分離されるため、`$400-600K` のような通貨が数式として解析されることはありません。長いリッチテキストは Telegram の制限に合わせて自動的に分割されます。20 列の上限を超えるテーブルはコードブロックへフォールバックします。

    デフォルトはオフです。これはクライアント互換性のためです。現在の一部の Desktop、Web、Android、およびサードパーティ製クライアントでは、受け入れられたリッチメッセージが未対応としてレンダリングされます。ボットで使用するすべてのクライアントがレンダリングできる場合を除き、オフのままにしてください。`/status` は、現在のセッションでリッチメッセージがオンかオフかを示します。

    リンクプレビューはデフォルトで有効です。`channels.telegram.linkPreview: false` はリッチテキストの自動エンティティ検出を無効にします。

  </Accordion>

  <Accordion title="ネイティブコマンドとカスタムコマンド">
    Telegram のコマンドメニューは起動時に `setMyCommands` で登録されます。`commands.native: "auto"` は Telegram のネイティブコマンドを有効にします。

    カスタムコマンドのメニュー項目を追加する場合:

```json5
{
  channels: {
    telegram: {
      customCommands: [
        { command: "backup", description: "Git バックアップ" },
        { command: "generate", description: "画像を作成" },
      ],
    },
  },
}
```

    ルール: 名前は正規化されます（先頭の `/` を削除し、小文字化）。有効なパターンは `a-z`、`0-9`、`_`、長さは 1～32 です。カスタムコマンドはネイティブコマンドを上書きできません。競合や重複はスキップされ、ログに記録されます。

    カスタムコマンドはメニュー項目にすぎず、動作は自動実装されません。Plugin または Skills のコマンドは、Telegram メニューに表示されていなくても入力すれば引き続き動作します。ネイティブコマンドが無効な場合、組み込みコマンドは削除されます。カスタムコマンドや Plugin コマンドは、設定されていれば引き続き登録される場合があります。

    よくあるセットアップエラー:

    - トリムの再試行後に `BOT_COMMANDS_TOO_MUCH` を伴う `setMyCommands failed` が発生する場合、メニューがまだ上限を超えています。Plugin、Skills、カスタムコマンドを減らすか、`channels.telegram.commands.native` を無効にしてください。
    - Bot API への直接の curl コマンドは動作する一方で、`deleteWebhook`、`deleteMyCommands`、または `setMyCommands` が `404: Not Found` で失敗する場合、通常は `channels.telegram.apiRoot` に完全な `/bot<TOKEN>` エンドポイントが設定されています。`apiRoot` には Bot API のルートのみを指定する必要があります。`openclaw doctor --fix` は、誤って末尾に付いた `/bot<TOKEN>` を削除します。
    - `getMe returned 401` は、設定されたボットトークンを Telegram が拒否したことを意味します。`botToken`、`tokenFile`、または `TELEGRAM_BOT_TOKEN`（デフォルトアカウント）を現在の BotFather トークンで更新してください。OpenClaw はポーリング前に停止するため、Webhook のクリーンアップ失敗としては報告されません。
    - ネットワークやフェッチのエラーを伴う `setMyCommands failed` は、通常、`api.telegram.org` への送信 DNS/HTTPS がブロックされていることを意味します。

    ### デバイスペアリングコマンド（`device-pair` Plugin）

    インストールされている場合:

    1. `/pair` はセットアップコードを生成します
    2. コードを iOS アプリに貼り付けます
    3. `/pair pending` は保留中のリクエスト（ロールやスコープを含む）を一覧表示します
    4. 承認: `/pair approve <requestId>`、`/pair approve`（保留中のリクエストが 1 件のみの場合）、または `/pair approve latest`

    デバイスが変更された認証情報（ロール、スコープ、公開鍵）で再試行すると、以前の保留中リクエストは新しい `requestId` に置き換えられます。承認前に `/pair pending` を再実行してください。

    詳細: [ペアリング](/ja-JP/channels/pairing#pair-via-telegram)。

  </Accordion>

  <Accordion title="インラインボタン">
    インラインキーボードのスコープを設定する場合:

```json5
{
  channels: {
    telegram: {
      capabilities: {
        inlineButtons: "allowlist",
      },
    },
  },
}
```

    アカウント単位で上書きする場合:

```json5
{
  channels: {
    telegram: {
      accounts: {
        main: {
          capabilities: {
            inlineButtons: "allowlist",
          },
        },
      },
    },
  },
}
```

    スコープ: `off`、`dm`、`group`、`all`、`allowlist`（デフォルト）。従来の `capabilities: ["inlineButtons"]` は `"all"` にマッピングされます。

    メッセージアクションの例:

```json5
{
  action: "send",
  channel: "telegram",
  to: "123456789",
  message: "オプションを選択してください:",
  buttons: [
    [
      { text: "はい", callback_data: "yes" },
      { text: "いいえ", callback_data: "no" },
    ],
    [{ text: "キャンセル", callback_data: "cancel" }],
  ],
}
```

    Mini App ボタンの例:

```json5
{
  action: "send",
  channel: "telegram",
  to: "123456789",
  message: "アプリを開く:",
  presentation: {
    blocks: [
      {
        type: "buttons",
        buttons: [{ label: "起動", web_app: { url: "https://example.com/app" } }],
      },
    ],
  },
}
```

    `web_app` ボタンは、ユーザーとボット間のプライベートチャットでのみ機能します。

    登録済みの Plugin インタラクティブハンドラーによって処理されなかったコールバッククリックは、テキストとしてエージェントに渡されます: `callback_data: <value>`。

  </Accordion>

  <Accordion title="エージェントと自動化向けの Telegram メッセージアクション">
    アクション:

    - `sendMessage`（`to`、`content`、オプションの `mediaUrl`、`replyToMessageId`、`messageThreadId`）
    - `react`（`chatId`、`messageId`、`emoji`）
    - `deleteMessage`（`chatId`、`messageId`）
    - `editMessage`（`chatId`、`messageId`、`content` または `caption`、オプションの `presentation` インラインボタン。ボタンのみの編集では返信マークアップが更新されます）
    - `createForumTopic`（`chatId`、`name`、オプションの `iconColor`、`iconCustomEmojiId`）

    操作しやすいエイリアス：`send`、`react`、`delete`、`edit`、`sticker`、`sticker-search`、`topic-create`。

    ゲート制御：`channels.telegram.actions.sendMessage`、`deleteMessage`、`reactions`、`sticker`（デフォルト：無効）。`edit`、`createForumTopic`、`editForumTopic` は専用の切り替え設定なしでデフォルトで有効です。
    ランタイム送信では、起動時または再読み込み時のアクティブな設定／シークレットのスナップショットを使用するため、アクションパスは送信ごとに `SecretRef` の値を再解決しません。

    リアクション削除のセマンティクス：[/tools/reactions](/ja-JP/tools/reactions)。

  </Accordion>

  <Accordion title="返信スレッドタグ">
    生成された出力で明示的に返信スレッドを指定するタグ：

    - `[[reply_to_current]]` — トリガーとなったメッセージに返信します
    - `[[reply_to:<id>]]` — 特定のメッセージ ID に返信します

    `channels.telegram.replyToMode`：`off`（デフォルト）、`first`、`all`。

    返信スレッドが有効で、元のテキスト／キャプションを利用できる場合、OpenClaw はネイティブ引用の抜粋を自動的に追加します。Telegram のネイティブ引用テキストは 1024 UTF-16 コード単位に制限されます。長いメッセージは先頭から引用され、Telegram が引用を拒否した場合は通常の返信にフォールバックします。

    `off` は暗黙的な返信スレッドのみを無効にします。明示的な `[[reply_to_*]]` タグは引き続き適用されます。

  </Accordion>

  <Accordion title="フォーラムトピックとスレッドの動作">
    フォーラムスーパーグループ：トピックのセッションキーには `:topic:<threadId>` が付加されます。返信と入力中表示は対象トピックのスレッドに送られます。トピック設定パスは `channels.telegram.groups.<chatId>.topics.<threadId>` です。

    一般トピック（`threadId=1`）は特殊なケースです。メッセージ送信では `message_thread_id` を省略します（Telegram は `sendMessage(...thread_id=1)` を「スレッドが見つかりません」として拒否します）が、入力中アクションには引き続き `message_thread_id` を含めます（入力中インジケーターを表示するために経験上必要です）。

    トピックのエントリは、上書きされない限りグループ設定を継承します（`requireMention`、`allowFrom`、`skills`、`systemPrompt`、`enabled`、`groupPolicy`）。`agentId` はトピック専用で、グループのデフォルト設定を継承しません。`topics."*"` はそのグループ内のすべてのトピックにデフォルト値を設定しますが、正確なトピック ID は引き続き `"*"` より優先されます。

    **トピックごとのエージェントルーティング**：各トピックは、トピック設定内の `agentId` を使用して異なるエージェントにルーティングでき、個別のワークスペース、メモリ、セッションを持たせることができます。

    ```json5
    {
      channels: {
        telegram: {
          groups: {
            "-1001234567890": {
              topics: {
                "1": { agentId: "main" },      // 一般トピック -> main エージェント
                "3": { agentId: "zu" },        // 開発トピック -> zu エージェント
                "5": { agentId: "coder" }      // コードレビュー -> coder エージェント
              }
            }
          }
        }
      }
    }
    ```

    各トピックはそれぞれ独自のセッションキーを持ちます。例：`agent:zu:telegram:group:-1001234567890:topic:3`。

    **永続的な ACP トピックバインディング**：フォーラムトピックでは、トップレベルの型付きバインディング（`bindings[]`、`type: "acp"`、`match.channel: "telegram"`、`peer.kind: "group"`、および `-1001234567890:topic:42` のようなトピック修飾 ID）を通じて ACP ハーネスセッションを固定できます。現在はグループ／スーパーグループ内のフォーラムトピックに限定されています。[ACP エージェント](/ja-JP/tools/acp-agents)を参照してください。

    **チャットからのスレッドバインド ACP 起動**：`/acp spawn <agent> --thread here|auto` は現在のトピックを新しい ACP セッションにバインドします。以降のメッセージはそこへ直接ルーティングされ、OpenClaw は起動確認をトピック内に固定します。`session.threadBindings.spawnSessions`（デフォルト：`true`）で制御します。

    テンプレートコンテキストでは `MessageThreadId` と `IsForum` が公開されます。`message_thread_id` を持つ DM チャットでは返信メタデータを保持しますが、Telegram の `getMe` が `has_topics_enabled: true` を報告する場合にのみ、スレッド対応セッションキーを使用します。
    廃止された `dm.threadReplies` と `direct.*.threadReplies` の上書き設定は削除されました。BotFather のスレッドモードが唯一の信頼できる情報源です。古い設定キーを削除するには `openclaw doctor --fix` を実行してください。

  </Accordion>

  <Accordion title="音声、動画、ステッカー">
    ### 音声メッセージ

    Telegram はボイスメモと音声ファイルを区別します。デフォルトは音声ファイルとしての動作です。エージェントの返信に `[[audio_as_voice]]` を付けると、ボイスメモとして強制的に送信します。受信したボイスメモの文字起こしは、エージェントコンテキスト内で機械生成された信頼されないテキストとして扱われますが、メンション検出には引き続き生の文字起こしが使用されるため、メンション必須の音声メッセージも動作します。

```json5
{
  action: "send",
  channel: "telegram",
  to: "123456789",
  media: "https://example.com/voice.ogg",
  asVoice: true,
}
```

    ### 動画メッセージ

    Telegram は動画ファイルとビデオメモを区別します。ビデオメモはキャプションに対応していないため、指定されたメッセージテキストは別途送信されます。

```json5
{
  action: "send",
  channel: "telegram",
  to: "123456789",
  media: "https://example.com/video.mp4",
  asVideoNote: true,
}
```

    ### 位置情報と会場

    既存の `send` アクションに、単独の `location` オブジェクトを 1 つ指定します。座標を指定するとネイティブのピンを送信し、`name` と `address` の両方を追加するとネイティブの会場カードを送信します。位置情報の送信をメッセージテキストやメディアと組み合わせることはできません。

```json5
{
  action: "send",
  channel: "telegram",
  to: "123456789",
  location: {
    latitude: 48.858844,
    longitude: 2.294351,
    accuracy: 12,
    name: "エッフェル塔",
    address: "シャン・ド・マルス、パリ",
  },
}
```

    ### ステッカー

    受信時：静止画 WEBP はダウンロードして処理されます（プレースホルダー `<media:sticker>`）。アニメーション TGS と動画 WEBM はスキップされます。

    ステッカーのコンテキストフィールド：`Sticker.emoji`、`Sticker.setName`、`Sticker.fileId`、`Sticker.fileUniqueId`、`Sticker.cachedDescription`。繰り返しのビジョン呼び出しを減らすため、説明は OpenClaw SQLite Plugin 状態にキャッシュされます。

    ステッカーアクションを有効にする：

```json5
{
  channels: {
    telegram: {
      actions: {
        sticker: true,
      },
    },
  },
}
```

    送信：

```json5
{
  action: "sticker",
  channel: "telegram",
  to: "123456789",
  fileId: "CAACAgIAAxkBAAI...",
}
```

    キャッシュされたステッカーを検索：

```json5
{
  action: "sticker-search",
  channel: "telegram",
  query: "手を振る猫",
  limit: 5,
}
```

  </Accordion>

  <Accordion title="リアクション通知">
    Telegram のリアクションは、メッセージペイロードとは別の `message_reaction` 更新として届きます。有効にすると、OpenClaw は `Telegram reaction added: 👍 by Alice (@alice) on msg 42` のようなシステムイベントをキューに追加します。

    - `channels.telegram.reactionNotifications`：`off | own | all`（デフォルト：`own`）
    - `channels.telegram.reactionLevel`：`off | ack | minimal | extensive`（デフォルト：`minimal`）

    `own` は、ボットが送信したメッセージに対するユーザーのリアクションのみを意味します（送信済みメッセージのキャッシュを使用するベストエフォート方式）。リアクションイベントにも Telegram のアクセス制御（`dmPolicy`、`allowFrom`、`groupPolicy`、`groupAllowFrom`）が適用され、許可されていない送信者は破棄されます。

    Telegram はリアクション更新でスレッド ID を提供しません。フォーラムではないグループはグループチャットのセッションにルーティングされ、フォーラムグループは発生元の正確なトピックではなく一般トピックのセッション（`:topic:1`）にルーティングされます。

    ポーリング／Webhook の `allowed_updates` には `message_reaction` が自動的に含まれます。

  </Accordion>

  <Accordion title="確認リアクション">
    `ackReaction` は、OpenClaw が受信メッセージを処理している間に確認用の絵文字を送信します。`messages.ackReactionScope` は送信する*タイミング*を決定します。

    **絵文字の解決順序：**

    - `channels.telegram.accounts.<accountId>.ackReaction`
    - `channels.telegram.ackReaction`
    - `messages.ackReaction`
    - エージェント ID の絵文字にフォールバック（`agents.entries.*.identity.emoji`、なければ「👀」）

    Telegram では Unicode 絵文字（例：「👀」）が必要です。チャンネルまたはアカウントでリアクションを無効にするには `""` を使用します。

    **スコープ（`messages.ackReactionScope`、デフォルトは `"group-mentions"`。現時点では Telegram アカウントまたは Telegram チャンネル単位の上書きはありません）：**

    `all`（DM とグループ。アンビエントルームイベントを含む）、`direct`（DM のみ）、`group-all`（アンビエントルームイベントを除くすべてのグループメッセージ。DM は除外）、`group-mentions`（ボットがメンションされたグループ。**DM は除外** — デフォルト）、`off`／`none`（無効）。

    <Note>
    デフォルトのスコープ（`group-mentions`）では、DM またはアンビエントルームイベントで確認リアクションは送信されません。DM では `direct` または `all` を使用してください。アンビエントルームイベントを確認するのは `all` のみです。この値は Telegram プロバイダーの起動時に読み込まれるため、変更を反映するには Gateway の再起動が必要です。
    </Note>

  </Accordion>

  <Accordion title="Telegram のイベントとコマンドによる設定の書き込み">
    チャンネル設定の書き込みはデフォルトで有効です（`configWrites !== false`）。Telegram をトリガーとする書き込みには、グループ移行イベント（`migrate_to_chat_id`、`channels.telegram.groups` を更新）と `/config set`／`/config unset`（コマンドの有効化が必要）が含まれます。

    無効にする：

```json5
{
  channels: {
    telegram: {
      configWrites: false,
    },
  },
}
```

  </Accordion>

  <Accordion title="ロングポーリングと Webhook">
    デフォルトはロングポーリングです。Webhook モードでは `channels.telegram.webhookUrl` と `channels.telegram.webhookSecret` を設定します。オプションとして `webhookPath`（デフォルト `/telegram-webhook`）、`webhookHost`（デフォルト `127.0.0.1`）、`webhookPort`（デフォルト `8787`）、`webhookCertPath`（IP 直接接続またはドメインなしの構成向けの自己署名証明書 PEM）を指定できます。

    ロングポーリングモードでは、OpenClaw は更新のディスパッチが正常に完了した後にのみ、再起動ウォーターマークを永続化します。ハンドラーが失敗した場合、その更新は完了済みとしてマークされず、同じプロセス内で再試行可能な状態のままになります。

    ローカルリスナーはデフォルトで `127.0.0.1:8787` にバインドします。公開受信にはローカルポートの前段にリバースプロキシを配置するか、意図的に `webhookHost: "0.0.0.0"` を設定してください。

    Webhook モードでは、リクエストガード、Telegram のシークレットトークン、JSON 本文を検証してから、空の `200` を返す前に更新を永続的な受信キューへコミットします。永続的な取り込みが成功した場合は `x-openclaw-delivery-accepted: durable` が含まれます。ヘルス、ルーティング、認証、検証、ストレージエラーのレスポンスでは、このヘッダーは省略されます。リバースプロキシとホストコントローラーはこのヘッダーを必須とすることで、レスポンスのタイミングから受理を推測せずに、OpenClaw による取り込みと一般的な空の `200` を区別できます。

    永続的な書き込み後、OpenClaw はコアのチャンネル受信ドレインを通じて更新を取得して処理します（チャットごと／トピックごとのレーン、ターンの取り込み時に完了、取り込み前の停滞タイムアウト）。エージェントのターンが遅くても、Telegram の配信 ACK は保留されません。

  </Accordion>

  <Accordion title="制限と CLI ターゲット">
    - `channels.telegram.textChunkLimit` のデフォルトは 4000 です。`streaming.chunkMode="newline"` は、長さで分割する前に段落境界（空行）を優先します。
    - `channels.telegram.mediaMaxMb`（デフォルト 100）は、受信および送信メディアのサイズを制限します。
    - グループコンテキスト履歴では `channels.telegram.historyLimit` または `messages.groupChat.historyLimit`（デフォルト 50）を使用します。`0` で無効化します。
    - 返信、引用、転送の補足コンテキストは、Gateway が親メッセージを観測済みの場合、選択された単一の会話コンテキストウィンドウに正規化されます。観測済みメッセージのキャッシュは OpenClaw SQLite Plugin 状態に保存され、`openclaw doctor --fix` が従来のサイドカーをインポートします。Telegram は更新ごとに浅い `reply_to_message` を 1 つだけ含めるため、キャッシュより古いチェーンはそのペイロードに制限されます。
    - Telegram の許可リストは主に、エージェントをトリガーできるユーザーを制限するものであり、補足コンテキスト全体の秘匿化境界ではありません。
    - DM 履歴: `channels.telegram.dmHistoryLimit`、`channels.telegram.dms["<user_id>"].historyLimit`。

    CLI およびメッセージツールの送信ターゲットには、数値のチャット ID、ユーザー名、またはフォーラムトピックのターゲットを指定できます。

```bash
openclaw message send --channel telegram --target 123456789 --message "hi"
openclaw message send --channel telegram --target @name --message "hi"
openclaw message send --channel telegram --target -1001234567890:topic:42 --message "hi topic"
```

    投票では `openclaw message poll` を使用し、フォーラムトピックにも対応しています。

```bash
openclaw message poll --channel telegram --target 123456789 \
  --poll-question "Ship it?" --poll-option "Yes" --poll-option "No"
openclaw message poll --channel telegram --target -1001234567890:topic:42 \
  --poll-question "Pick a time" --poll-option "10am" --poll-option "2pm" \
  --poll-duration-seconds 300 --poll-public
```

    Telegram 専用の投票フラグ: `--poll-duration-seconds`（5-600）、`--poll-anonymous`、`--poll-public`、`--thread-id`（または `:topic:` ターゲット）。`--poll-option` は 2-12 回繰り返します（Telegram の選択肢上限）。

    Telegram の送信では、インラインキーボード用の `buttons` ブロックを含む `--presentation`（`channels.telegram.capabilities.inlineButtons` で許可されている場合）、ボットがそのチャットでピン留めできる場合にピン留め配信を要求する `--pin` または `--delivery '{"pin":true}'`、および送信画像、GIF、動画を圧縮画像、アニメーション、動画としてアップロードする代わりにドキュメントとして送信する `--force-document` にも対応しています。

    アクション制御: `channels.telegram.actions.sendMessage=false` は投票を含むすべての送信メッセージを無効にします。`channels.telegram.actions.poll=false` は通常の送信を有効にしたまま、投票の作成を無効にします。

  </Accordion>

  <Accordion title="Telegram での exec 承認">
    Telegram は承認者の DM で exec 承認に対応し、必要に応じて発信元のチャットまたはトピックにプロンプトを投稿できます。承認者には数値の Telegram ユーザー ID を指定する必要があります。

    - `channels.telegram.execApprovals.enabled`（少なくとも 1 人の承認者を解決できる場合、`"auto"` で有効化）
    - `channels.telegram.execApprovals.approvers`（`commands.ownerAllowFrom` の数値オーナー ID にフォールバック）
    - `channels.telegram.execApprovals.target`: `dm`（デフォルト）| `channel` | `both`
    - `agentFilter`、`sessionFilter`

    `channels.telegram.allowFrom`、`groupAllowFrom`、`defaultTo` は、ボットと会話できるユーザーと通常の返信の送信先を制御しますが、ユーザーを exec 承認者にするものではありません。コマンドオーナーがまだ存在しない場合、最初に承認された DM ペアリングによって `commands.ownerAllowFrom` が初期設定されるため、オーナーが 1 人の構成では `execApprovals.approvers` に ID を重複して指定する必要はありません。

    チャネル配信では、コマンドテキストがチャットに表示されます。信頼できるグループまたはトピックでのみ `channel` または `both` を有効にしてください。プロンプトがフォーラムトピックに届いた場合、OpenClaw は承認プロンプトとその後のメッセージでもトピックを維持します。exec 承認はデフォルトで 30 分後に期限切れになります。

    インライン承認ボタンを使用するには、ターゲットのサーフェス（`dm`、`group`、または `all`）を `channels.telegram.capabilities.inlineButtons` で許可する必要もあります。`plugin:` で始まる承認 ID は Plugin 承認によって解決され、それ以外はまず exec 承認によって解決されます。

    [exec 承認](/ja-JP/tools/exec-approvals)を参照してください。

  </Accordion>
</AccordionGroup>

## エラー返信の制御

エージェントで配信エラーまたはプロバイダーエラーが発生した場合、エラーポリシーによってエラーメッセージを Telegram チャットに送信するかどうかが制御されます。

| キー                             | 値                     | デフォルト  | 説明                                                                                                                                                                |
| ------------------------------- | -------------------------- | -------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `channels.telegram.errorPolicy` | `always`、`once`、`silent` | `always` | `always` はすべてのエラーメッセージをチャットに送信します。`once` は一意の各エラーメッセージを組み込みのクールダウン期間ごとに 1 回送信します。`silent` はエラーメッセージをチャットに送信しません。 |

アカウント単位、グループ単位、トピック単位のオーバーライドに対応しています（他の Telegram 設定キーと同じ継承方式）。

```json5
{
  channels: {
    telegram: {
      errorPolicy: "always",
      groups: {
        "-1001234567890": {
          errorPolicy: "silent", // このグループではエラーを抑制
        },
      },
    },
  },
}
```

## トラブルシューティング

<AccordionGroup>
  <Accordion title="メンションのないグループメッセージにボットが応答しない">

    - `requireMention=false` の場合、Telegram のプライバシーモードで完全な可視性を許可する必要があります。BotFather の `/setprivacy` -> Disable を選択してから、ボットをグループから削除し、再度追加してください。
    - 設定でメンションのないグループメッセージを想定している場合、`openclaw channels status` が警告します。
    - `openclaw channels status --probe` は明示的な数値グループ ID を確認します。ワイルドカード `"*"` ではメンバーシップを検査できません。
    - 簡易セッションテスト: `/activation always`。

  </Accordion>

  <Accordion title="ボットがグループメッセージをまったく認識しない">

    - `channels.telegram.groups` が存在する場合、グループをリストに含める必要があります（または `"*"` を含めます）。
    - ボットがグループのメンバーであることを確認してください。
    - スキップ理由については `openclaw logs --follow` を確認してください。

  </Accordion>

  <Accordion title="コマンドが一部しか動作しない、またはまったく動作しない">

    - 送信者 ID を承認してください（ペアリングおよび／または数値の `allowFrom`）。グループポリシーが `open` の場合でも、コマンドの承認は引き続き適用されます。
    - `BOT_COMMANDS_TOO_MUCH` を伴う `setMyCommands failed` は、ネイティブメニューの項目が多すぎることを意味します。Plugin、Skills、カスタムコマンドを減らすか、ネイティブメニューを無効にしてください。
    - 起動時の `deleteMyCommands` / `setMyCommands` 呼び出しと `sendChatAction` の入力中通知呼び出しには制限があり、リクエストがタイムアウトした場合は Telegram のトランスポートフォールバック経由で 1 回再試行されます。ネットワーク／fetch エラーが継続する場合、通常は `api.telegram.org` への DNS/HTTPS 接続が利用できないことを意味します。

  </Accordion>

  <Accordion title="起動時に未承認トークンが報告される">

    - `getMe returned 401` は、設定されたボットトークンに対する Telegram 認証エラーです。BotFather でトークンを再コピーまたは再生成し、`channels.telegram.botToken`、`tokenFile`、`accounts.<id>.botToken`、または `TELEGRAM_BOT_TOKEN`（デフォルトアカウント）を更新してください。
    - 起動時の `deleteWebhook 401 Unauthorized` も認証エラーです。これを「Webhook が存在しない」として扱っても、同じ不正なトークンによるエラーが後続の API 呼び出しまで延期されるだけです。

  </Accordion>

  <Accordion title="ポーリングまたはネットワークが不安定">

    - カスタム fetch／プロキシを使用する Node 22+ では、`AbortSignal` の型が一致しない場合、即時中断が発生することがあります。
    - 一部のホストでは `api.telegram.org` が最初に IPv6 に解決されます。IPv6 の外向き通信に問題があると、API 障害が断続的に発生します。
    - `TypeError: fetch failed` または `Network request for 'getUpdates' failed!` を含むログは、回復可能なネットワークエラーとして再試行されます。
    - ポーリング起動時、OpenClaw は起動時に成功した `getMe` プローブを grammY 用に再利用するため、ランナーは最初の `getUpdates` の前に 2 回目の `getMe` を行う必要がありません。
    - ポーリング起動中に `deleteWebhook` が一時的なネットワークエラーで失敗した場合、OpenClaw はポーリング前の制御プレーン呼び出しをもう一度行わず、ロングポーリングへ移行します。Webhook がまだ有効な場合は `getUpdates` の競合として表面化し、OpenClaw はトランスポートを再構築して Webhook のクリーンアップを再試行します。
    - ログの `Polling stall detected` は、デフォルトではロングポーリングの稼働確認が 120 秒間完了しなかったため、OpenClaw がポーリングを再起動し、トランスポートを再構築したことを意味します。
    - `openclaw channels status --probe` と `openclaw doctor` は、実行中のポーリングアカウントが起動猶予期間後も `getUpdates` を完了していない場合、実行中の Webhook アカウントが起動猶予期間後も `setWebhook` を完了していない場合、または最後に成功したポーリングトランスポートのアクティビティが古くなっている場合に警告します。
    - Telegram は Bot API トランスポートでプロセスのプロキシ環境変数 `HTTP_PROXY`、`HTTPS_PROXY`、`ALL_PROXY` およびそれらの小文字表記に従います。`NO_PROXY` / `no_proxy` では引き続き `api.telegram.org` をバイパスできます。
    - サービス環境に `OPENCLAW_PROXY_URL` が設定され、標準のプロキシ環境変数が存在しない場合、Telegram はその URL を Bot API トランスポートにも使用します。
    - 直接の外向き通信／TLS が不安定な VPS ホストでは、Telegram API 呼び出しをプロキシ経由でルーティングしてください。

```yaml
channels:
  telegram:
    proxy: socks5://<user>:<password>@proxy-host:1080
```

    - Node 22+ のデフォルトは `autoSelectFamily=true` です（WSL2 を除く）。Telegram の DNS 結果順序は `OPENCLAW_TELEGRAM_DNS_RESULT_ORDER`、次に `channels.telegram.network.dnsResultOrder`、次にプロセスのデフォルト（例: `NODE_OPTIONS=--dns-result-order=ipv4first`）に従い、いずれも適用されない場合は Node 22+ で `ipv4first` にフォールバックします。
    - WSL2、または IPv4 のみの動作がより適切な場合は、アドレスファミリーの選択を強制してください。

```yaml
channels:
  telegram:
    network:
      autoSelectFamily: false
```

    - RFC 2544 ベンチマーク範囲の応答（`198.18.0.0/15`）は、Telegram メディアのダウンロードでデフォルトですでに許可されています。信頼できる fake-IP または透過プロキシが、メディアのダウンロード中に `api.telegram.org` を別のプライベート／内部／特殊用途アドレスへ書き換える場合は、Telegram 専用のバイパスを明示的に有効にしてください。

```yaml
channels:
  telegram:
    network:
      dangerouslyAllowPrivateNetwork: true
```

    - 同じオプトインは、`channels.telegram.accounts.<accountId>.network.dangerouslyAllowPrivateNetwork` でアカウントごとにも利用できます。
    - プロキシが Telegram のメディアホストを `198.18.x.x` に解決する場合は、まず危険なフラグをオフのままにしてください。この範囲はデフォルトですでに許可されています。

    <Warning>
      `channels.telegram.network.dangerouslyAllowPrivateNetwork` は Telegram メディアの SSRF 保護を弱めます。RFC 2544 ベンチマーク範囲外のプライベートまたは特殊用途の応答を生成する、信頼できる運用者管理のプロキシ環境（Clash、Mihomo、Surge の fake-IP ルーティング）でのみ使用してください。通常の公開インターネット経由の Telegram アクセスではオフのままにしてください。
    </Warning>

    - 一時的な環境オーバーライド: `OPENCLAW_TELEGRAM_DISABLE_AUTO_SELECT_FAMILY=1`、`OPENCLAW_TELEGRAM_ENABLE_AUTO_SELECT_FAMILY=1`、`OPENCLAW_TELEGRAM_DNS_RESULT_ORDER=ipv4first`。
    - DNS 応答を検証します。

```bash
dig +short api.telegram.org A
dig +short api.telegram.org AAAA
```

  </Accordion>
</AccordionGroup>

詳細については、[チャネルのトラブルシューティング](/ja-JP/channels/troubleshooting)を参照してください。

## 設定リファレンス

主なリファレンス: [設定リファレンス - Telegram](/ja-JP/gateway/config-channels#telegram)。

<Accordion title="重要な Telegram フィールド">

- 起動/認証: `enabled`、`botToken`、`tokenFile`（通常ファイルである必要があります。シンボリックリンクは拒否されます）、`accounts.*`
- アクセス制御: `dmPolicy`、`allowFrom`、`groupPolicy`、`groupAllowFrom`、`groups`、`groups.*.topics.*`、トップレベルの `bindings[]`（`type: "acp"`）
- トピックのデフォルト: `groups.<chatId>.topics."*"` は一致しないフォーラムトピックに適用されます。正確なトピック ID がこれを上書きします
- 実行承認: `execApprovals`、`accounts.*.execApprovals`
- コマンド/メニュー: `commands.native`、`commands.nativeSkills`、`customCommands`
- スレッド/返信: `replyToMode`、`threadBindings`
- ストリーミング: `streaming`（モード `off | partial | block | progress`）、`streaming.preview.toolProgress`
- 書式設定/配信: `textChunkLimit`、`streaming.chunkMode`、`richMessages`、`markdown.tables`（`off | bullets | code | block`）、`linkPreview`、`responsePrefix`
- メディア/ネットワーク: `mediaMaxMb`、`network.autoSelectFamily`、`network.dangerouslyAllowPrivateNetwork`、`proxy`
- カスタム API ルート: `apiRoot`（Bot API ルートのみ。`/bot<TOKEN>` は含めないでください）、`trustedLocalFileRoots`（セルフホスト Bot API の絶対 `file_path` ルート）
- Webhook: `webhookUrl`、`webhookSecret`、`webhookPath`、`webhookHost`、`webhookPort`、`webhookCertPath`
- アクション/機能: `capabilities.inlineButtons`、`actions.sendMessage|editMessage|deleteMessage|reactions|sticker|createForumTopic|editForumTopic`
- リアクション: `reactionNotifications`、`reactionLevel`
- エラー: `errorPolicy`、`silentErrorReplies`
- 書き込み/履歴: `configWrites`、`historyLimit`、`dmHistoryLimit`、`dms.*.historyLimit`

</Accordion>

<Note>
複数アカウントの優先順位: 2 つ以上のアカウント ID を設定している場合は、デフォルトのルーティングを明示するために `channels.telegram.defaultAccount` を設定（または `channels.telegram.accounts.default` を含める）してください。それ以外の場合、OpenClaw は正規化された最初のアカウント ID にフォールバックし、`openclaw doctor` が警告を出します。名前付きアカウントは `channels.telegram.allowFrom` / `groupAllowFrom` を継承しますが、`accounts.default.*` の値は継承しません。
</Note>

## 関連項目

<CardGroup cols={2}>
  <Card title="ペアリング" icon="link" href="/ja-JP/channels/pairing">
    Telegram ユーザーを Gateway とペアリングします。
  </Card>
  <Card title="グループ" icon="users" href="/ja-JP/channels/groups">
    グループおよびトピックの許可リストの動作。
  </Card>
  <Card title="チャネルルーティング" icon="route" href="/ja-JP/channels/channel-routing">
    受信メッセージをエージェントにルーティングします。
  </Card>
  <Card title="セキュリティ" icon="shield" href="/ja-JP/gateway/security">
    脅威モデルと堅牢化。
  </Card>
  <Card title="マルチエージェントルーティング" icon="sitemap" href="/ja-JP/concepts/multi-agent">
    グループとトピックをエージェントにマッピングします。
  </Card>
  <Card title="トラブルシューティング" icon="wrench" href="/ja-JP/channels/troubleshooting">
    チャネル横断の診断。
  </Card>
</CardGroup>
