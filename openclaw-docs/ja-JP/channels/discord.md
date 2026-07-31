---
read_when:
    - Discord チャンネル機能の開発
summary: Discord ボットのセットアップ、設定キー、コンポーネント、音声、トラブルシューティング
title: Discord
x-i18n:
    generated_at: "2026-07-26T09:28:45Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 52a2926217f3a8dfb9398551ddacb0bc6aae6de0a164b215c55256eda9b6245e
    source_path: channels/discord.md
    workflow: 16
---

OpenClaw は公式の Discord gateway を介してボットとして Discord に接続します。DM とギルドチャンネルに対応しています。

<CardGroup cols={3}>
  <Card title="ペアリング" icon="link" href="/ja-JP/channels/pairing">
    Discord DM はデフォルトでペアリングモードになります。
  </Card>
  <Card title="スラッシュコマンド" icon="terminal" href="/ja-JP/tools/slash-commands">
    ネイティブコマンドの動作とコマンドカタログ。
  </Card>
  <Card title="チャンネルのトラブルシューティング" icon="wrench" href="/ja-JP/channels/troubleshooting">
    チャンネル横断の診断と修復フロー。
  </Card>
</CardGroup>

## クイックセットアップ

ボットを含む Discord アプリケーションを作成し、ボットをサーバーに追加して、OpenClaw とペアリングします。可能であればプライベートサーバーを使用してください。必要に応じて、先に[サーバーを作成](https://support.discord.com/hc/en-us/articles/204849977-How-do-I-create-a-server)します（**Create My Own > For me and my friends**）。

<Steps>
  <Step title="Discord アプリケーションとボットを作成する">
    [Discord Developer Portal](https://discord.com/developers/applications) で **New Application** をクリックし、名前を付けます（例:「OpenClaw」）。

    サイドバーで **Bot** を開き、**Username** をエージェントの名前に設定します。

  </Step>

  <Step title="特権インテントを有効にする">
    引き続き **Bot** ページの **Privileged Gateway Intents** で、以下を有効にします。

    - **Message Content Intent**（必須）
    - **Server Members Intent**（推奨。ロール許可リスト、名前から ID への照合、チャンネルオーディエンスアクセスグループには必須）
    - **Presence Intent**（任意。プレゼンス更新のみに使用）

  </Step>

  <Step title="ボットトークンをコピーする">
    **Bot** ページで **Reset Token** をクリックし、トークンをコピーします。

    <Note>
    この名前とは異なり、最初のトークンが生成されるだけで、何かが「リセット」されるわけではありません。
    </Note>

  </Step>

  <Step title="招待 URL を生成してボットをサーバーに追加する">
    サイドバーで **OAuth2** を開きます。**OAuth2 URL Generator** で、以下のスコープを有効にします。

    - `bot`
    - `applications.commands`

    表示される **Bot Permissions** セクションで、少なくとも以下を有効にします。

    **General Permissions**
      - View Channels

    **Text Permissions**
      - Send Messages
      - Read Message History
      - Embed Links
      - Attach Files
      - Add Reactions（任意）

    これが通常のテキストチャンネルに必要な基本設定です。フォーラムやメディアチャンネルでスレッドを作成または継続するワークフローなど、ボットがスレッドに投稿する場合は、**Send Messages in Threads** も有効にします。

    生成された URL をコピーしてブラウザで開き、サーバーを選択して **Continue** をクリックします。これでボットがサーバーに表示されます。

  </Step>

  <Step title="Developer Mode を有効にして ID を収集する">
    ID をコピーできるように、Discord アプリで Developer Mode を有効にします。

    1. **User Settings**（歯車アイコン）→ **Developer** → **Developer Mode** をオン
       *（モバイルの場合: **App Settings** → **Advanced**）*
    2. **サーバーアイコン**を右クリック → **Copy Server ID**
    3. **自分のアバター**を右クリック → **Copy User ID**

    Server ID と User ID をボットトークンと一緒に保管してください。次の手順では 3 つすべてが必要です。

  </Step>

  <Step title="サーバーメンバーからの DM を許可する">
    ペアリングを機能させるには、Discord でボットから DM を受信できるようにする必要があります。**サーバーアイコン**を右クリック → **Privacy Settings** → **Direct Messages** をオンにします。

    OpenClaw で Discord DM を使用する場合は、この設定をオンのままにしてください。ギルドチャンネルのみを使用する場合は、ペアリング後に無効にできます。

  </Step>

  <Step title="ボットトークンを安全に設定する（チャットには送信しない）">
    ボットトークンはシークレットです。エージェントにメッセージを送る前に、OpenClaw を実行しているマシンで設定します。

```bash
export DISCORD_BOT_TOKEN="YOUR_BOT_TOKEN"
cat > discord.patch.json5 <<'JSON5'
{
  channels: {
    discord: {
      enabled: true,
      token: { source: "env", provider: "default", id: "DISCORD_BOT_TOKEN" },
    },
  },
}
JSON5
openclaw config patch --file ./discord.patch.json5 --dry-run
openclaw config patch --file ./discord.patch.json5
openclaw gateway
```

    OpenClaw がすでにバックグラウンドサービスとして実行されている場合は、OpenClaw Mac アプリを使用するか、`openclaw gateway run` プロセスを停止して再起動してください。
    マネージドサービスとしてインストールしている場合は、`DISCORD_BOT_TOKEN` が設定されているシェルから `openclaw gateway install` を実行するか、再起動後にサービスが環境変数の SecretRef を解決できるよう、変数を `~/.openclaw/.env` に保存します。
    ホストが Discord の起動時アプリケーション検索によってブロックまたはレート制限される場合は、起動時にその REST 呼び出しを省略できるよう、Developer Portal のアプリケーション／クライアント ID を設定します。デフォルトアカウントには `channels.discord.applicationId`、ボットごとには `channels.discord.accounts.<accountId>.applicationId` を使用します。

  </Step>

  <Step title="OpenClaw を設定してペアリングする">

    <Tabs>
      <Tab title="エージェントに依頼">
        既存のチャンネル（Telegram など）で OpenClaw エージェントとチャットし、設定を依頼します。Discord が最初のチャンネルの場合は、代わりに CLI／設定タブを使用してください。

        > 「Discord ボットトークンは設定済みです。User ID `<user_id>` と Server ID `<server_id>` を使用して Discord のセットアップを完了してください。」
      </Tab>
      <Tab title="CLI／設定">
        ファイルベースの設定:

```json5
{
  channels: {
    discord: {
      enabled: true,
      token: {
        source: "env",
        provider: "default",
        id: "DISCORD_BOT_TOKEN",
      },
    },
  },
}
```

        デフォルトアカウント用の環境変数フォールバック:

```bash
DISCORD_BOT_TOKEN=...
```

        スクリプトまたはリモートでセットアップする場合は、`openclaw config patch --file ./discord.patch.json5 --dry-run` を使用して同じ JSON5 ブロックを書き込み、その後 `--dry-run` なしで再実行します。プレーンテキストの `token` 文字列も使用でき、env/file/exec プロバイダー全体で `channels.discord.token` に SecretRef 値を使用できます。[シークレット管理](/ja-JP/gateway/secrets)を参照してください。

        複数の Discord ボットを使用する場合は、各ボットのトークンとアプリケーション ID をそれぞれのアカウント内に保持します。トップレベルの `channels.discord.applicationId` は各アカウントに継承されるため、すべてのアカウントで同じアプリケーション ID を使用する場合にのみ、そこへ設定してください。

```json5
{
  channels: {
    discord: {
      enabled: true,
      accounts: {
        personal: {
          token: { source: "env", provider: "default", id: "DISCORD_PERSONAL_TOKEN" },
          applicationId: "111111111111111111",
        },
        work: {
          token: { source: "env", provider: "default", id: "DISCORD_WORK_TOKEN" },
          applicationId: "222222222222222222",
        },
      },
    },
  },
}
```

      </Tab>
    </Tabs>

  </Step>

  <Step title="最初の DM ペアリングを承認する">
    Gateway が起動したら、Discord でボットに DM を送信します。ボットからペアリングコードが返信されます。

    <Tabs>
      <Tab title="エージェントに依頼">
        既存のチャンネルでエージェントにペアリングコードを送信します。

        > 「この Discord ペアリングコードを承認してください: `<CODE>`」
      </Tab>
      <Tab title="CLI">

```bash
openclaw pairing list discord
openclaw pairing approve discord <CODE>
```

      </Tab>
    </Tabs>

    ペアリングコードは 1 時間後に期限切れになります。承認後、Discord DM でエージェントとチャットできます。

  </Step>
</Steps>

<Note>
トークン解決はアカウントを考慮します。設定内のトークン値は環境変数フォールバックより優先され、`DISCORD_BOT_TOKEN` はデフォルトアカウントにのみ使用されます。
有効な 2 つの Discord アカウントが同じボットトークンに解決される場合、OpenClaw はそのトークンに対して 1 つの Gateway モニターのみを起動します。設定由来のトークンは環境変数フォールバックより優先されます。それ以外の場合は、最初に有効化されたアカウントが優先され、重複するアカウントは理由 `duplicate bot token` とともに無効として報告されます。
高度な送信呼び出し（メッセージツール／チャンネルアクション）では、呼び出しごとに明示的な `token` が使用されます。これは送信および読み取り／プローブ形式のアクション（read/search/fetch/thread/pins/permissions）に適用されます。アカウントポリシー／再試行設定は、引き続きアクティブなランタイムスナップショットで選択されたアカウントから取得されます。
</Note>

## 推奨: ギルドワークスペースをセットアップする

DM が機能するようになったら、サーバーを、各チャンネルが独自のコンテキストを持つ個別のエージェントセッションとして動作する完全なワークスペースにできます。自分とボットだけがいるプライベートサーバーに推奨します。

<Steps>
  <Step title="サーバーをギルド許可リストに追加する">
    これにより、エージェントは DM だけでなく、サーバー上のすべてのチャンネルで応答できるようになります。

    <Tabs>
      <Tab title="エージェントに依頼">
        > 「Discord Server ID `<server_id>` をギルド許可リストに追加してください」
      </Tab>
      <Tab title="設定">

```json5
{
  channels: {
    discord: {
      groupPolicy: "allowlist",
      guilds: {
        YOUR_SERVER_ID: {
          requireMention: true,
          users: ["YOUR_USER_ID"],
        },
      },
    },
  },
}
```

      </Tab>
    </Tabs>

  </Step>

  <Step title="@メンションなしの応答を許可する">
    デフォルトでは、エージェントはギルドチャンネルで @メンションされた場合にのみ応答します。プライベートサーバーでは、すべてのメッセージに応答させることが一般的です。

    ギルドチャンネルでは、通常の返信はデフォルトで自動的に投稿されます。常時稼働する共有ルームでは、`messages.groupChat.visibleReplies: "message_tool"` を有効にすると、エージェントは待機し、チャンネルへの返信が有用だと判断した場合にのみ投稿できます。これは GPT-5.6 Sol のような、最新世代でツールの信頼性が高いモデルで最も効果的に動作します。ツールが送信しない限り、アンビエントルームイベントは投稿されません。待機モードの完全な設定については、[アンビエントルームイベント](/ja-JP/channels/ambient-room-events)を参照してください。

    Discord に入力中と表示され、ログにもトークン使用量が記録されているのにメッセージが投稿されない場合は、そのターンがアンビエントルームイベントとして設定されているか、メッセージツールによる可視返信が有効になっているかを確認してください。

    <Tabs>
      <Tab title="エージェントに依頼">
        > 「このサーバーで @メンションなしでもエージェントが応答できるようにしてください」
      </Tab>
      <Tab title="設定">
        ギルド設定で `requireMention: false` を設定します。

```json5
{
  channels: {
    discord: {
      guilds: {
        YOUR_SERVER_ID: {
          requireMention: false,
        },
      },
    },
  },
}
```

        表示されるグループ／チャンネル返信でメッセージツールからの送信を必須にするには、`messages.groupChat.visibleReplies: "message_tool"` を設定します。

      </Tab>
    </Tabs>

  </Step>

  <Step title="ギルドチャンネルでのメモリ利用を計画する">
    長期メモリ（MEMORY.md）は DM セッションでのみ自動的に読み込まれ、ギルドチャンネルでは読み込まれません。

    <Tabs>
      <Tab title="エージェントに依頼">
        > 「Discord チャンネルで質問した際に MEMORY.md の長期コンテキストが必要な場合は、memory_search または memory_get を使用してください。」
      </Tab>
      <Tab title="手動">
        すべてのチャンネルでコンテキストを共有するには、安定した指示を `AGENTS.md` または `USER.md`（各セッションに挿入されます）に記述します。長期的なメモは `MEMORY.md` に保存し、必要に応じてメモリツールからアクセスします。
      </Tab>
    </Tabs>

  </Step>
</Steps>

チャンネルを作成してチャットを開始します。エージェントはチャンネル名を認識し、各チャンネルは分離されたセッションになります。ワークフローに合わせて `#coding`、`#home`、`#research` などをセットアップしてください。

## ランタイムモデル

- Gateway が Discord 接続を管理します。
- 返信ルーティングは決定論的です。Discord から受信したメッセージへの返信は Discord に返されます。
- Discord のギルド／チャンネルメタデータは、ユーザーに表示される返信の接頭辞ではなく、信頼されていないコンテキストとしてモデルプロンプトに追加されます。モデルがそのエンベロープを返信にコピーした場合、OpenClaw は送信返信と今後の再生コンテキストからコピーされたメタデータを削除します。
- デフォルト（`session.dmScope=main`）では、ダイレクトチャットはエージェントのメインセッション（`agent:main:main`）を共有します。
- ギルドチャンネルには分離されたセッションキー（`agent:<agentId>:discord:channel:<channelId>`）が割り当てられます。
- グループ DM はデフォルトで無視されます（`channels.discord.dm.groupEnabled=false`）。
- ネイティブスラッシュコマンドは分離されたコマンドセッション（`agent:<agentId>:discord:slash:<userId>`）で実行されますが、ルーティング先の会話セッションへの `CommandTargetSessionKey` は引き続き保持されます。
- Discord へのテキストのみの Cron／Heartbeat アナウンス配信は、アシスタントに表示される最終回答に集約され、1 回だけ送信されます。エージェントが配信可能なペイロードを複数出力した場合、メディアと構造化コンポーネントのペイロードは引き続き複数メッセージとして送信されます。

## フォーラムチャンネル

Discord のフォーラムチャンネルとメディアチャンネルでは、スレッドへの投稿のみ受け付けます。OpenClaw では、次の 2 つの方法で作成できます。

- フォーラムの親 (`channel:<forumId>`) にメッセージを送信すると、スレッドが自動作成されます。スレッドタイトルには、メッセージの最初の空でない行が使用されます（Discord のスレッド名の上限である 100 文字に切り詰められます）。
- `openclaw message thread create` を使用してスレッドを直接作成します。フォーラムチャンネルには `--message-id` を渡さないでください。

フォーラムの親に送信してスレッドを作成する場合:

```bash
openclaw message send --channel discord --target channel:<forumId> \
  --message "トピックのタイトル\n投稿の本文"
```

フォーラムスレッドを明示的に作成する場合:

```bash
openclaw message thread create --channel discord --target channel:<forumId> \
  --thread-name "トピックのタイトル" --message "投稿の本文"
```

フォーラムの親は Discord コンポーネントを受け付けません。コンポーネントが必要な場合は、スレッド自体 (`channel:<threadId>`) に送信してください。

## インタラクティブコンポーネント

OpenClaw は、エージェントメッセージで Discord components v2 コンテナをサポートしています。`components` ペイロードを指定してメッセージツールを使用します。インタラクションの結果は通常の受信メッセージとしてエージェントに返され、既存の Discord `replyToMode` 設定に従います。

サポートされているブロック:

- `text`、`section`、`separator`、`actions`、`media-gallery`、`file`
- アクション行には、最大 5 個のボタンまたは 1 個の選択メニューを配置できます
- 選択タイプ: `string`、`user`、`role`、`mentionable`、`channel`

デフォルトでは、コンポーネントは 1 回だけ使用できます。ボタン、選択メニュー、フォームを期限切れになるまで複数回使用できるようにするには、`components.reusable=true` を設定します。

ボタンをクリックできるユーザーを制限するには、そのボタンに `allowedUsers` を設定します（Discord ユーザー ID、タグ、または `*`）。一致しないユーザーには、一時的な拒否メッセージが表示されます。

コンポーネントのコールバックは、デフォルトでは 30 分後に期限切れになります。デフォルトアカウントのコールバックレジストリの有効期間を変更するには `channels.discord.agentComponents.ttlMs` を、アカウントごとに変更するには `channels.discord.accounts.<accountId>.agentComponents.ttlMs` を設定します。値の単位はミリ秒で、正の整数である必要があり、上限は `86400000`（24 時間）です。長い TTL は、ボタンを使用可能な状態に保つ必要があるレビューや承認のワークフローに適していますが、古い Discord メッセージから引き続きアクションを実行できる期間も長くなります。要件を満たす最短の TTL を使用し、古いコールバックが予期しない動作を招く場合はデフォルトを維持してください。

`/model` および `/models` スラッシュコマンドは、プロバイダー、モデル、互換性のあるランタイムのドロップダウンと Submit ステップを備えたインタラクティブなモデル選択画面を開きます。`/models add` は非推奨であり、チャットからモデルを登録する代わりに非推奨メッセージを返します。選択画面の応答は一時的で、呼び出したユーザーだけが使用できます。Discord の選択メニューは 25 個のオプションに制限されているため、`openai` や `vllm` など、選択したプロバイダーについてのみ動的に検出されたモデルを選択画面に表示する場合は、`agents.defaults.modelPolicy.allow` に `provider/*` エントリを追加してください。

ファイル添付:

- `file` ブロックは添付ファイル参照 (`attachment://<filename>`) を指す必要があります
- 添付ファイルは `media`/`path`/`filePath`（単一ファイル）で指定します。複数のファイルには `media-gallery` を使用します
- アップロード名を添付ファイル参照と一致させる必要がある場合は、`filename` を使用して上書きします

モーダルフォーム:

- 最大 5 個のフィールドを含む `components.modal` を追加します
- フィールドタイプ: `text`、`checkbox`、`radio`、`select`、`role-select`、`user-select`
- OpenClaw がトリガーボタンを自動的に追加します

例:

```json5
{
  channel: "discord",
  action: "send",
  to: "channel:123456789012345678",
  message: "任意のフォールバックテキスト",
  components: {
    reusable: true,
    text: "パスを選択",
    blocks: [
      {
        type: "actions",
        buttons: [
          {
            label: "承認",
            style: "success",
            allowedUsers: ["123456789012345678"],
          },
          { label: "却下", style: "danger" },
        ],
      },
      {
        type: "actions",
        select: {
          type: "string",
          placeholder: "オプションを選択",
          options: [
            { label: "オプション A", value: "a" },
            { label: "オプション B", value: "b" },
          ],
        },
      },
    ],
    modal: {
      title: "詳細",
      triggerLabel: "フォームを開く",
      fields: [
        { type: "text", label: "申請者" },
        {
          type: "select",
          label: "優先度",
          options: [
            { label: "低", value: "low" },
            { label: "高", value: "high" },
          ],
        },
      ],
    },
  },
}
```

## アクセス制御とルーティング

<Tabs>
  <Tab title="DM ポリシー">
    `channels.discord.dmPolicy` は DM アクセスを制御します。`channels.discord.allowFrom` は正規の DM 許可リストです。

    - `pairing`（デフォルト）
    - `allowlist`（少なくとも 1 人の `allowFrom` 送信者が必要）
    - `open`（`channels.discord.allowFrom` に `"*"` を含める必要があります）
    - `disabled`

    DM ポリシーがオープンでない場合、不明なユーザーはブロックされます（または `pairing` モードではペアリングを求められます）。

    複数アカウントでの優先順位:

    - `channels.discord.accounts.default.allowFrom` は `default` アカウントにのみ適用されます。
    - 単一アカウントでは、`allowFrom` が従来の `dm.allowFrom` より優先されます。
    - 名前付きアカウントでは、独自の `allowFrom` と従来の `dm.allowFrom` が設定されていない場合、`channels.discord.allowFrom` を継承します。
    - 名前付きアカウントは `channels.discord.accounts.default.allowFrom` を継承しません。

    従来の `channels.discord.dm.policy` と `channels.discord.dm.allowFrom` は、互換性のために引き続き読み込まれます。アクセスを変更せずに実行できる場合、`openclaw doctor --fix` はこれらを `dmPolicy` と `allowFrom` に移行します。

    配信用の DM ターゲット形式:

    - `user:<id>`
    - `<@id>` メンション

    チャンネルのデフォルトが有効な場合、数字のみの ID は通常チャンネル ID として解決されます。ただし、アカウントで有効な DM `allowFrom` に記載されている ID は、互換性のためにユーザー DM ターゲットとして扱われます。

  </Tab>

  <Tab title="アクセスグループ">
    Discord の DM とテキストコマンドの認可では、`channels.discord.allowFrom` 内の動的な `accessGroup:<name>` エントリを使用できます。

    アクセスグループ名はメッセージチャンネル間で共有されます。メンバーを各チャンネルの通常の `allowFrom` 構文で表す静的グループには `type: "message.senders"` を使用し、Discord チャンネルの現在の `ViewChannel` 対象者によってメンバーシップを動的に定義する場合は `type: "discord.channelAudience"` を使用します。アクセスグループに共通する動作については、[アクセスグループ](/ja-JP/channels/access-groups)を参照してください。

```json5
{
  accessGroups: {
    operators: {
      type: "message.senders",
      members: {
        "*": ["global-owner-id"],
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
  },
}
```

    Discord のテキストチャンネルには、個別のメンバーリストはありません。`type: "discord.channelAudience"` では、メンバーシップを次のようにモデル化します。DM の送信者が設定されたギルドのメンバーであり、ロールとチャンネルの上書きを適用した後、設定されたチャンネルに対する有効な `ViewChannel` 権限を現在持っていることです。

    例: `#maintainers` を表示できるすべてのユーザーにボットへの DM を許可し、それ以外のユーザーからの DM は拒否します。

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

    動的エントリと静的エントリを組み合わせることもできます。

```json5
{
  accessGroups: {
    maintainers: {
      type: "discord.channelAudience",
      guildId: "1456350064065904867",
      channelId: "1456744319972282449",
    },
  },
  channels: {
    discord: {
      dmPolicy: "allowlist",
      allowFrom: ["accessGroup:maintainers", "discord:123456789012345678"],
    },
  },
}
```

    検索に失敗した場合はアクセスを拒否します。Discord が `Missing Access` を返した場合、メンバー検索に失敗した場合、またはチャンネルが別のギルドに属している場合、DM の送信者は認可されていないものとして扱われます。

    チャンネル対象者のアクセスグループを使用する場合は、Discord Developer Portal で **Server Members Intent** を有効にしてください。DM にはギルドメンバーの状態が含まれないため、OpenClaw は認可時に Discord REST を通じてメンバーを解決します。

  </Tab>

  <Tab title="ギルドポリシー">
    ギルドの処理は `channels.discord.groupPolicy` によって制御されます。

    - `open`
    - `allowlist`
    - `disabled`

    `channels.discord` が存在する場合の安全な基準値は `allowlist` です。

    `allowlist` の動作:

    - ギルドは `channels.discord.guilds` と一致する必要があります（`id` を推奨。スラッグも使用可能）
    - 任意の送信者許可リスト: `users`（安定した ID を推奨）および `roles`（ロール ID のみ）。いずれかが設定されている場合、送信者が `users` または `roles` に一致すれば許可されます
    - 名前やタグによる直接照合はデフォルトで無効です。緊急時の互換モードとしてのみ `channels.discord.dangerouslyAllowNameMatching: true` を有効にしてください
    - `users` では名前やタグを使用できますが、ID の方が安全です。名前やタグのエントリが使用されている場合、`openclaw security audit` が警告します
    - ギルドに `channels` が設定されている場合、リストにないチャンネルは拒否されます
    - ギルドに `channels` ブロックがない場合、許可リストに含まれるそのギルド内のすべてのチャンネルが許可されます

    例:

```json5
{
  channels: {
    discord: {
      groupPolicy: "allowlist",
      guilds: {
        "123456789012345678": {
          requireMention: true,
          ignoreOtherMentions: true,
          users: ["987654321098765432"],
          roles: ["123456789012345678"],
          channels: {
            general: { enabled: true },
            help: { enabled: true, requireMention: true },
          },
        },
      },
    },
  },
}
```

    従来のチャンネルごとの `allow` キーは、`openclaw doctor --fix` によって `enabled` に移行されます。

    `DISCORD_BOT_TOKEN` のみを設定し、`channels.discord` ブロックを作成しない場合、`channels.defaults.groupPolicy` が `open` であっても、ランタイムのフォールバックは `groupPolicy="allowlist"` になります（ログに警告が出力されます）。

  </Tab>

  <Tab title="メンションとグループ DM">
    ギルドメッセージは、デフォルトでメンションが必要です。

    メンションの検出対象:

    - ボットへの明示的なメンション
    - 設定されたメンションパターン（`agents.entries.*.groupChat.mentionPatterns`、フォールバックは `messages.groupChat.mentionPatterns`）
    - サポートされている場合の、ボットへの返信による暗黙的な動作

    Discord の送信メッセージを作成する場合は、正規のメンション構文を使用してください。ユーザーには `<@USER_ID>`、チャンネルには `<#CHANNEL_ID>`、ロールには `<@&ROLE_ID>` を使用します。従来の `<@!USER_ID>` ニックネームメンション形式は使用しないでください。

    `requireMention` はギルドまたはチャンネルごとに設定します（`channels.discord.guilds...`）。
    `ignoreOtherMentions` を指定すると、別のユーザーまたはロールにメンションしているものの、ボットにはメンションしていないメッセージを任意で破棄できます（@everyone/@here を除く）。

    グループ DM:

    - デフォルト: 無視（`dm.groupEnabled=false`）
    - `dm.groupChannels` による任意の許可リスト（チャンネル ID またはスラッグ）

  </Tab>
</Tabs>

### ロールベースのエージェントルーティング

Discord ギルドのメンバーをロール ID に基づいて異なるエージェントへルーティングするには、`bindings[].match.roles` を使用します。ロールベースのバインディングではロール ID のみを使用でき、ピアまたは親ピアのバインディングの後、ギルドのみのバインディングの前に評価されます。バインディングにほかの照合フィールド（たとえば `peer` + `guildId` + `roles`）も設定されている場合、設定されたすべてのフィールドが一致する必要があります。

```json5
{
  bindings: [
    {
      agentId: "opus",
      match: {
        channel: "discord",
        guildId: "123456789012345678",
        roles: ["111111111111111111"],
      },
    },
    {
      agentId: "sonnet",
      match: {
        channel: "discord",
        guildId: "123456789012345678",
      },
    },
  ],
}
```

## ネイティブコマンドとコマンド認証

- `commands.native` のデフォルトは `"auto"` で、Discord では有効です。
- チャンネルごとのオーバーライド: `channels.discord.commands.native`。
- `commands.native=false` は、起動時の Discord スラッシュコマンドの登録とクリーンアップをスキップします。以前に登録されたコマンドは、Discord アプリから削除するまで Discord に表示され続ける場合があります。
- ネイティブコマンド認証では、通常のメッセージ処理と同じ Discord の許可リスト/ポリシーが使用されます。
- 権限のないユーザーにも Discord UI でコマンドが表示される場合がありますが、実行時には OpenClaw の認証が適用され、「not authorized」と応答します。
- デフォルトのスラッシュコマンド設定: `ephemeral: true`（`channels.discord.slashCommand.ephemeral`）。

コマンドの一覧と動作については、[スラッシュコマンド](/ja-JP/tools/slash-commands)を参照してください。

## 機能の詳細

<AccordionGroup>
  <Accordion title="返信タグとネイティブ返信">
    Discord は、エージェントの出力内の返信タグをサポートします。

    - `[[reply_to_current]]`
    - `[[reply_to:<id>]]`

    `channels.discord.replyToMode` で制御します。

    - `off`（デフォルト）: 暗黙的な返信スレッド化は行いません。明示的な `[[reply_to_*]]` タグは引き続き適用されます
    - `first`: ターンの最初の送信 Discord メッセージに、暗黙的なネイティブ返信参照を付加します
    - `all`: すべての送信メッセージに付加します
    - `batched`: 受信イベントが複数メッセージをデバウンスしたバッチである場合にのみ付加します。すべての単一メッセージのターンではなく、主に曖昧なバースト状のチャットでネイティブ返信を使用したい場合に便利です

    メッセージ ID はコンテキスト/履歴に提示されるため、エージェントは特定のメッセージを対象にできます。

  </Accordion>

  <Accordion title="リンクプレビュー">
    Discord はデフォルトで URL のリッチリンク埋め込みを生成します。OpenClaw は送信する Discord メッセージで、生成される埋め込みをデフォルトで抑制します。そのため、オプトインしない限り、エージェントが送信した URL はプレーンリンクのままです。

```json5
{
  channels: {
    discord: {
      suppressEmbeds: false,
    },
  },
}
```

    1 つのアカウントをオーバーライドするには、`channels.discord.accounts.<id>.suppressEmbeds` を設定します。エージェントのメッセージツールによる送信では、単一メッセージに `suppressEmbeds: false` を渡すこともできます。明示的な Discord `embeds` ペイロードは、デフォルトのリンクプレビュー設定では抑制されません。

  </Accordion>

  <Accordion title="ライブストリームプレビュー">
    OpenClaw は、一時メッセージを送信し、テキストの到着に合わせて編集することで、返信の下書きをストリーミングできます。`channels.discord.streaming.mode` は `off` | `partial` | `block` | `progress` を取ります（`streaming`/従来の `streamMode` キーが設定されていない場合のデフォルト）。`streamMode` は従来のエイリアスです。`openclaw doctor --fix` を実行すると、永続化された設定が正規のネストされた `streaming` 形式に書き換えられます。

```json5
{
  channels: {
    discord: {
      streaming: {
        mode: "progress",
        progress: {
          maxLines: 8,
          maxLineChars: 120,
          toolProgress: false,
          commentary: false,
        },
      },
    },
  },
}
```

    - `off` は Discord のプレビュー編集を無効にします。
    - `partial` は、トークンの到着に合わせて 1 つのプレビューメッセージを編集します。
    - `block` は、下書きサイズのチャンクを生成します。サイズと区切り位置は `streaming.preview.chunk`（`minChars`、`maxChars`、`breakPreference`）で調整でき、`textChunkLimit` に制限されます。ブロックストリーミングが明示的に有効な場合、OpenClaw は二重ストリーミングを避けるため、プレビューストリームをスキップします。
    - `progress` は、最終配信まで編集可能なステータス下書きを 1 つ維持します。デフォルトでは、エージェントの最新の前置きまたはナレーションを 1 行表示し、生成されたラベル、スペーサー、ツール行は表示しません。
    - メディア、エラー、明示的な返信の最終出力は、保留中のプレビュー編集をキャンセルします。
    - `streaming.preview.toolProgress` のデフォルトは、`partial`/`block` モードでは `true` です。Discord の進行状況モードでは、デフォルトでツール行は表示されません。オプトインするには、`streaming.progress.toolProgress: true` を設定します。
    - `🛠️ Bash: run tests` や `🔎 Web Search: for "query"` などのコンパクトなツール/進行状況行を追加するには、`streaming.progress.toolProgress: true` を設定します。互換性のため、既存の `progress.label` または `progress.labels` 設定では、以前のツール行のデフォルトが維持されます。行を表示せずにカスタムラベルを使用するには、`toolProgress: false` を設定します。
    - `streaming.progress.commentary`（デフォルト `false`）は、一時的な進行状況の下書きで生のアシスタント解説を表示するようオプトインします。デフォルトの前置き/ナレーションのステータス行は、このオプションとは独立しています。解説は表示前にクリーンアップされ、一時的なままで、最終回答の配信には影響しません。
    - `streaming.progress.maxLineChars` は、行ごとの進行状況プレビューの上限を制御します。文章は単語の境界で短縮され、コマンドとパスの詳細では有用な末尾が維持されます。
    - `streaming.preview.commandText` / `streaming.progress.commandText` は、コンパクトな進行状況行のコマンド/実行詳細を制御します。`raw`（デフォルト）または `status`（ツールラベルのみ）です。

    コンパクトな進行状況行を維持しながら、生のコマンド/実行テキストを非表示にするには、次のように設定します。

    ```json
    {
      "channels": {
        "discord": {
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

    プレビューストリーミングはテキスト専用です。メディア返信は通常の配信にフォールバックします。

  </Accordion>

  <Accordion title="履歴、コンテキスト、スレッドの動作">
    ギルド履歴のコンテキスト:

    - `channels.discord.historyLimit` のデフォルトは `20`
    - フォールバック: `messages.groupChat.historyLimit`
    - `0` で無効化

    DM 履歴の制御:

    - `channels.discord.dmHistoryLimit`
    - `channels.discord.dms["<user_id>"].historyLimit`

    スレッドの動作:

    - Discord スレッドはチャンネルセッションとしてルーティングされ、オーバーライドされない限り親チャンネルの設定を継承します。
    - スレッドセッションは、モデル専用のフォールバックとして親チャンネルのセッションレベルの `/model` 選択を継承します。スレッドローカルの `/model` 選択が優先され、トランスクリプトの継承が有効でない限り、親のトランスクリプト履歴はコピーされません。
    - `channels.discord.thread.inheritParent`（デフォルト `false`）は、新しい自動スレッドで親のトランスクリプトを初期データとして使用するようオプトインします。アカウントごとのオーバーライド: `channels.discord.accounts.<id>.thread.inheritParent`。
    - メッセージツールのリアクションは、`user:<id>` の DM ターゲットを解決できます。
    - `guilds.<guild>.channels.<channel>.requireMention: false` は、返信ステージのアクティベーションのフォールバック中も維持されます。

    チャンネルトピックは **信頼されていない** コンテキストとして挿入されます。許可リストはエージェントをトリガーできるユーザーを制限しますが、補足コンテキスト全体の秘匿化境界ではありません。

  </Accordion>

  <Accordion title="サブエージェントのスレッド連携セッション">
    Discord はスレッドをセッションターゲットに関連付けることができ、そのスレッド内の後続メッセージを同じセッション（サブエージェントセッションを含む）に引き続きルーティングできます。

    コマンド:

    - `/focus <target>` 現在または新しいスレッドをサブエージェント/セッションターゲットに関連付けます
    - `/unfocus` 現在のスレッドの関連付けを解除します
    - `/agents` アクティブな実行と関連付けの状態を表示します
    - `/session idle <duration|off>` フォーカスされた関連付けの非アクティブ時の自動フォーカス解除を確認/更新します
    - `/session max-age <duration|off>` フォーカスされた関連付けの最大存続時間を確認/更新します

    設定:

```json5
{
  session: {
    threadBindings: {
      enabled: true,
      idleHours: 24,
      maxAgeHours: 0,
      spawnSessions: true,
      defaultSpawnContext: "fork",
    },
  },
}
```

    注:

    - `session.threadBindings.*` は、Discord と Telegram の正規ポリシーです。
    - `spawnSessions` は、`sessions_spawn({ thread: true })` および ACP スレッド生成時のスレッドの自動作成/関連付けを制御します。デフォルト: `true`。
    - `defaultSpawnContext` は、スレッド連携による生成のネイティブサブエージェントコンテキストを制御します。デフォルト: `"fork"`。
    - 非推奨の `spawnSubagentSessions`/`spawnAcpSessions` キーは、`openclaw doctor --fix` によって移行されます。
    - スレッドの関連付けが無効な場合、`/focus` および関連操作は使用できません。

    [サブエージェント](/ja-JP/tools/subagents)、[ACP エージェント](/ja-JP/tools/acp-agents)、[設定リファレンス](/ja-JP/gateway/configuration-reference)を参照してください。

  </Accordion>

  <Accordion title="送信元メッセージ上のサブエージェント進行状況">
    親の実行を開始した Discord メッセージ上にバックグラウンドの子アクティビティを表示するには、`channels.discord.subagentProgress: true` を設定します。

```json5
{
  channels: {
    discord: {
      subagentProgress: true,
    },
  },
}
```

    子の実行がアクティブな間、OpenClaw は Discord の入力中表示を最大 1 時間維持し、同時実行数の変化に応じて 1 つのカウントリアクション（`1️⃣` から `🔟`）を置き換えます。`🔟` は 10 以上も表します。最後の子が終了すると、カウントリアクションは削除されます。失敗、タイムアウト、または強制終了した子は、`🔴` リアクションを残します。

    これはオプトイン機能で、固定された内部タイミングと絵文字のデフォルトを使用します。リアクションによるフィードバックには、ボットに **Add Reactions** 権限が必要です。アカウントレベルの `channels.discord.accounts.<id>.subagentProgress` は、トップレベルの値をオーバーライドします。

  </Accordion>

  <Accordion title="永続的な ACP チャンネル関連付け">
    安定した「常時稼働」の ACP ワークスペースでは、Discord の会話を対象とするトップレベルの型付き ACP 関連付けを設定します。

    設定パス: `bindings[]`、`type: "acp"`、および `match.channel: "discord"`。

```json5
{
  agents: {
    entries: {
      codex: {
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
    },
  },
  bindings: [
    {
      type: "acp",
      agentId: "codex",
      match: {
        channel: "discord",
        accountId: "default",
        peer: { kind: "channel", id: "222222222222222222" },
      },
      acp: { label: "codex-main" },
    },
  ],
  channels: {
    discord: {
      guilds: {
        "111111111111111111": {
          channels: {
            "222222222222222222": {
              requireMention: false,
            },
          },
        },
      },
    },
  },
}
```

    注:

    - `/acp spawn codex --bind here` は現在のチャンネルまたはスレッドをその場で関連付け、以後のメッセージを同じ ACP セッションに維持します。スレッドメッセージは親チャンネルの関連付けを継承します。
    - 関連付けられたチャンネルまたはスレッドでは、`/new` と `/reset` が同じ ACP セッションをその場でリセットします。一時的なスレッドの関連付けがアクティブな間は、ターゲット解決をオーバーライドできます。
    - `spawnSessions` は、`--thread auto|here` を介した子スレッドの作成/関連付けを制限します。

    関連付けの動作の詳細については、[ACP エージェント](/ja-JP/tools/acp-agents)を参照してください。

  </Accordion>

  <Accordion title="リアクション通知">
    ギルドごとのリアクション通知モード（`guilds.<id>.reactionNotifications`）:

    - `off`
    - `own`（デフォルト）
    - `all`
    - `allowlist`（`guilds.<id>.users` を使用）

    リアクションイベントはシステムイベントに変換され、ルーティングされた Discord セッションに付加されます。

  </Accordion>

  <Accordion title="オンライン状態イベント">
    人間のメンバーがオフラインからオンラインに移行したときに、ルーティングされたエージェントを起動するようギルドをオプトインします。

    ```json5
    {
      channels: {
        discord: {
          intents: { presence: true },
          guilds: {
            "111111111111111111": {
              presenceEvents: {
                channelId: "222222222222222222",
                users: ["333333333333333333"], // 任意。チャンネルの閲覧者をさらに絞り込む
                reconnectSuppressSeconds: 300, // 任意。新規セッションの抑制時間（0 で無効化）
                burstLimit: 8, // 任意。バーストウィンドウあたりの最大イベント数
                burstWindowSeconds: 60, // 任意。スライド式バースト検出ウィンドウ
              },
            },
          },
        },
      },
    }
    ```

    `presenceEvents` には、ルーティング先エージェントで Heartbeat が有効になっていることと、Discord Developer Portal にあるアプリケーションの Bot ページで特権 **Presence Intent** が有効になっていることが必要です。OpenClaw は、完全な各 `GUILD_CREATE` スナップショットから現在オンラインのメンバーを初期登録し、観測されたオフラインからオンラインへの遷移をルーティングします。また、それまで観測されていなかったメンバーから後で初めてオンライン信号を受信した場合も、新たに利用可能になったものとして扱います。そのメンバーはスナップショット後にオンラインになったか参加した可能性があるため、このイベントは正確な直前のステータスを断定しません。対象となるのは、`channelId` を表示できる人間のみです。チャンネルと公開スレッドでは、そのチャンネルまたは親に対する **View Channel** が必要で、非公開スレッドではさらにメンバーであるか **Manage Threads** が必要です。`users` で対象者をさらに絞り込めます。OpenClaw はボットと変化のないオンライン状態を無視し、ユーザーごとの 8 時間のクールダウンを Gateway の再起動後も保持します。Discord が新しい Gateway セッションを確立して `READY` を送信すると、OpenClaw はギルドのプレゼンス状態が再構築される間、`reconnectSuppressSeconds`（デフォルトは 300、`0` で無効化）にわたってプレゼンス由来のイベントを抑制し、再観測されたメンバーが 1 人ずつエージェントを起動しないようにします。さらに、正常にキューへ追加されたイベントをギルドごとに、`burstWindowSeconds` のスライドウィンドウ（デフォルトは 60）あたり `burstLimit` 件（デフォルトは 8）にレート制限し、各ギルドの抑制期間につき一度だけログを記録します。再開されたセッションは新しいセッションとして扱われません。Discord はメンバー数が 75,000 を超えるギルドのスナップショットを制限します。その場合、OpenClaw が挨拶するには明示的なオフライン更新が事前に必要です。システムイベントは、変更可能な表示名を埋め込まず、不変のユーザー ID、ギルド ID、チャンネル ID を保持します。挨拶するかどうか、およびその方法はエージェントが決定します。

  </Accordion>

  <Accordion title="確認リアクション">
    `ackReaction` は、OpenClaw が受信メッセージを処理している間、確認用の絵文字を送信します。

    解決順序：

    - `channels.discord.accounts.<accountId>.ackReaction`
    - `channels.discord.ackReaction`
    - `messages.ackReaction`
    - エージェントのアイデンティティ絵文字へのフォールバック（`agents.entries.*.identity.emoji`、なければ「👀」）

    注意事項：

    - Discord は Unicode 絵文字またはカスタム絵文字名を受け付けます。
    - チャンネルまたはアカウントでリアクションを無効にするには、`""` を使用します。

    **スコープ（`messages.ackReactionScope`）：**

    値：`"all"`（DM とグループ。アンビエントルームイベントを含む）、`"direct"`（DM のみ）、`"group-all"`（アンビエントルームイベントを除くすべてのグループメッセージ。DM は除く）、`"group-mentions"`（ボットがメンションされたグループ。**DM は除く**、デフォルト）、`"off"` / `"none"`（無効）。

    <Note>
    デフォルトのスコープ（`"group-mentions"`）では、ダイレクトメッセージやアンビエントルームイベントに対して確認リアクションは実行されません。受信した Discord の DM と静かなルームイベントに確認リアクションを付けるには、`messages.ackReactionScope` を `"all"` に設定します。
    </Note>

  </Accordion>

  <Accordion title="設定の書き込み">
    チャンネルから開始される設定の書き込みは、デフォルトで有効です。これは `/config set|unset` のフローに影響します（コマンド機能が有効な場合）。

    無効化：

```json5
{
  channels: {
    discord: {
      configWrites: false,
    },
  },
}
```

  </Accordion>

  <Accordion title="Gateway プロキシ">
    `channels.discord.proxy` を使用して、Discord Gateway の WebSocket トラフィックと起動時の REST 参照（アプリケーション ID と許可リストの解決）を HTTP(S) プロキシ経由でルーティングします。
    Discord Gateway の WebSocket プロキシは明示的に設定する必要があります。WebSocket 接続は Gateway プロセスの環境にあるプロキシ環境変数を継承しません。`channels.discord.proxy` が設定されている場合、起動時の REST 参照はこのプロキシを使用します。

```json5
{
  channels: {
    discord: {
      proxy: "http://proxy.example:8080",
    },
  },
}
```

    アカウントごとの上書き：

```json5
{
  channels: {
    discord: {
      accounts: {
        primary: {
          proxy: "http://proxy.example:8080",
        },
      },
    },
  },
}
```

  </Accordion>

  <Accordion title="PluralKit のサポート">
    PluralKit の解決を有効にして、プロキシされたメッセージをシステムメンバーのアイデンティティに対応付けます。

```json5
{
  channels: {
    discord: {
      pluralkit: {
        enabled: true,
        token: "pk_live_...", // 任意。非公開システムに必要
      },
    },
  },
}
```

    注意事項：

    - 許可リストでは `pk:<memberId>` を使用できます
    - メンバーの表示名は、`channels.discord.dangerouslyAllowNameMatching: true` の場合にのみ名前またはスラッグで照合されます
    - 参照では、元のメッセージ ID を使用して PluralKit API に問い合わせます
    - 参照に失敗した場合、プロキシされたメッセージはボットメッセージとして扱われ、`allowBots` で通過が許可されていない限り破棄されます

  </Accordion>

  <Accordion title="送信メンションのエイリアス">
    エージェントが既知の Discord ユーザーに対して決定的な送信メンションを必要とする場合は、`mentionAliases` を使用します。キーは先頭の `@` を除いたハンドルで、値は Discord ユーザー ID です。不明なハンドル、`@everyone`、`@here`、および Markdown のコードスパン内にあるメンションは変更されません。

```json5
{
  channels: {
    discord: {
      mentionAliases: {
        SupportLead: "123456789012345678",
      },
      accounts: {
        ops: {
          mentionAliases: {
            OpsLead: "234567890123456789",
          },
        },
      },
    },
  },
}
```

  </Accordion>

  <Accordion title="プレゼンスの設定">
    ステータスまたはアクティビティのフィールドを設定するか、自動プレゼンスを有効にすると、プレゼンスの更新が適用されます。

    ステータスのみ：

```json5
{
  channels: {
    discord: {
      status: "idle",
    },
  },
}
```

    アクティビティ（`activity` が設定されている場合、カスタムステータスがデフォルトのアクティビティ種別です）：

```json5
{
  channels: {
    discord: {
      activity: "集中時間",
      activityType: 4,
    },
  },
}
```

    ストリーミング：

```json5
{
  channels: {
    discord: {
      activity: "ライブコーディング",
      activityType: 1,
      activityUrl: "https://twitch.tv/openclaw",
    },
  },
}
```

    アクティビティ種別の対応表：

    - 0：プレイ中
    - 1：配信中（`activityUrl` が必要。さらに `activityUrl` には `activityType: 1` が必要）
    - 2：再生中
    - 3：視聴中
    - 4：カスタム（アクティビティのテキストをステータス状態として使用。絵文字は任意）
    - 5：対戦中

    自動プレゼンス（ランタイムの正常性シグナル）：

```json5
{
  channels: {
    discord: {
      autoPresence: {
        enabled: true,
        intervalMs: 30000,
        minUpdateIntervalMs: 15000,
        exhaustedText: "トークン枯渇",
      },
    },
  },
}
```

    自動プレゼンスは、ランタイムの可用性を Discord のステータスに対応付けます。正常 => オンライン、劣化または不明 => アイドル、枯渇または利用不可 => 取り込み中。デフォルト：`intervalMs` は 30000、`minUpdateIntervalMs` は 15000（`intervalMs` 以下である必要があります）。任意のテキスト上書き：

    - `autoPresence.healthyText`
    - `autoPresence.degradedText`
    - `autoPresence.exhaustedText`（`{reason}` プレースホルダーをサポート）

  </Accordion>

  <Accordion title="Discord での承認">
    Discord は DM でのボタンベースの承認処理をサポートし、必要に応じて承認プロンプトを元のチャンネルにも投稿できます。

    設定パス：

    - `channels.discord.execApprovals.enabled`
    - `channels.discord.execApprovals.approvers`（任意。可能な場合は `commands.ownerAllowFrom` にフォールバック）
    - `channels.discord.execApprovals.target`（`dm` | `channel` | `both`、デフォルト：`dm`）
    - `agentFilter`、`sessionFilter`、`cleanupAfterResolve`

    `enabled` が未設定または `"auto"` で、`execApprovals.approvers` または `commands.ownerAllowFrom` から少なくとも 1 人の承認者を解決できる場合、Discord はネイティブの実行承認を自動的に有効にします。Discord は、チャンネルの `allowFrom`、従来の `dm.allowFrom`、またはダイレクトメッセージの `defaultTo` から実行承認者を推測しません。Discord をネイティブ承認クライアントとして明示的に無効にするには、`enabled: false` を設定します。

    `/diagnostics` や `/export-trajectory` のような機密性の高い所有者専用グループコマンドでは、OpenClaw は承認プロンプトと最終結果を非公開で送信します。呼び出した所有者に Discord の所有者ルートがある場合は、まず Discord の DM を試します。それ以外の場合は、Telegram など、`commands.ownerAllowFrom` で利用可能な最初の所有者ルートにフォールバックします。

    `target` が `channel` または `both` の場合、承認プロンプトはチャンネルに表示されます。ボタンを使用できるのは解決済みの承認者のみです。それ以外のユーザーには一時的な拒否メッセージが表示されます。承認プロンプトにはコマンドテキストが含まれるため、チャンネルへの配信は信頼できるチャンネルでのみ有効にしてください。セッションキーからチャンネル ID を取得できない場合、OpenClaw は DM 配信にフォールバックします。

    Discord は、他のチャットチャンネルでも使用される共通の承認ボタンをレンダリングします。Discord のネイティブアダプターが主に追加するのは、承認者への DM ルーティングとチャンネルへのファンアウトです。これらのボタンがある場合、それが主要な承認 UX となります。OpenClaw が手動の `/approve` コマンドを含めるのは、ツールの結果がチャット承認を利用できないと示す場合、または手動承認が唯一の経路である場合に限ります。Discord のネイティブ承認ランタイムが有効でない場合、OpenClaw はローカルで決定的な `/approve <id> <decision>` プロンプトを表示し続けます。ランタイムが有効でもネイティブカードをどの宛先にも配信できない場合、OpenClaw は保留中の承認に含まれる正確な `/approve` コマンドとともに、同じチャットへフォールバック通知を送信します。

    Gateway の認証と承認の解決は、共通の Gateway クライアント契約に従います（`plugin:` ID は `plugin.approval.resolve` を通じて解決し、それ以外の ID は `exec.approval.resolve` を通じて解決します）。承認はデフォルトで 30 分後に期限切れになります。

    [実行承認](/ja-JP/tools/exec-approvals)を参照してください。

  </Accordion>
</AccordionGroup>

## ツールとアクションゲート

Discord のメッセージアクションは、メッセージング、チャンネル管理、モデレーション、プレゼンス、メタデータを対象とします。

主な例：

- メッセージング：`sendMessage`、`readMessages`、`editMessage`、`deleteMessage`、`threadReply`
- リアクション：`react`、`reactions`、`emojiList`
- モデレーション：`timeout`、`kick`、`ban`
- プレゼンス：`setPresence`

`event-create` アクションは、スケジュール済みイベントのカバー画像を設定するために、任意の `image` パラメーター（URL またはローカルファイルパス）を受け付けます。

アクションゲートは `channels.discord.actions.*` の配下にあります。

デフォルトのゲート動作：

| アクショングループ                                                                                                                                                         | デフォルト |
| ------------------------------------------------------------------------------------------------------------------------------------------------------------------------ | ---------- |
| reactions, messages, threads, pins, polls, search, memberInfo, roleInfo, channelInfo, channels, voiceStatus, events, stickers, emojiUploads, stickerUploads, permissions | 有効       |
| roles                                                                                                                                                                    | 無効       |
| moderation                                                                                                                                                               | 無効       |
| presence                                                                                                                                                                 | 無効       |

## Components v2 UI

OpenClaw は、実行承認とコンテキスト間マーカーに Discord components v2 を使用します。Discord のメッセージアクションでは、カスタム UI 用に `components` も指定できます（高度な使用方法です。discord ツールを介してコンポーネントペイロードを構築する必要があります）。従来の `embeds` も引き続き使用できますが、推奨されません。

- `channels.discord.ui.components.accentColor` は、Discord コンポーネントコンテナで使用するアクセントカラー（16 進数）を設定します。アカウントごと: `channels.discord.accounts.<id>.ui.components.accentColor`。
- `channels.discord.agentComponents.ttlMs` は、送信された Discord コンポーネントのコールバックを登録したままにする期間を制御します（デフォルト `1800000`、最大 `86400000`）。アカウントごと: `channels.discord.accounts.<id>.agentComponents.ttlMs`。
- `embeds` は、components v2 が存在する場合は無視されます。
- プレーン URL のプレビューはデフォルトで抑制されます。単一の外部リンクを展開する必要がある場合は、メッセージアクションに `suppressEmbeds: false` を設定します。

例:

```json5
{
  channels: {
    discord: {
      ui: {
        components: {
          accentColor: "#5865F2",
        },
      },
    },
  },
}
```

## 音声

Discord には、リアルタイムの **ボイスチャンネル**（継続的な会話）と **ボイスメッセージの添付ファイル**（波形プレビュー形式）という 2 つの異なる音声サーフェスがあります。Gateway は両方をサポートしています。

### ボイスチャンネル

セットアップチェックリスト:

1. Discord Developer Portal で Message Content Intent を有効にします。
2. ロールまたはユーザーの許可リストを使用する場合は、Server Members Intent を有効にします。
3. `bot` および `applications.commands` スコープを指定してボットを招待します。
4. 対象のボイスチャンネルで Connect、Speak、Send Messages、Read Message History を付与します。
5. ネイティブコマンド（`commands.native` または `channels.discord.commands.native`）を有効にします。
6. `channels.discord.voice` を設定します。

セッションを制御するには `/vc join|leave|status` を使用します。このコマンドはアカウントのデフォルトエージェントを使用し、他の Discord コマンドと同じ許可リストおよびグループポリシーのルールに従います。

```bash
/vc join channel:<voice-channel-id>
/vc status
/vc leave
```

参加する前にボットの実効権限を確認するには:

```bash
openclaw channels capabilities --channel discord --target channel:<voice-channel-id>
```

自動参加の例:

```json5
{
  channels: {
    discord: {
      voice: {
        enabled: true,
        model: "openai/gpt-5.6-sol",
        autoJoin: [
          {
            guildId: "123456789012345678",
            channelId: "234567890123456789",
          },
        ],
        allowedChannels: [
          {
            guildId: "123456789012345678",
            channelId: "234567890123456789",
          },
        ],
        daveEncryption: true,
        decryptionFailureTolerance: 24,
        connectTimeoutMs: 30000,
        reconnectGraceMs: 15000,
        realtime: {
          provider: "openai",
          model: "gpt-realtime-2.1",
          speakerVoice: "cedar",
        },
      },
    },
  },
}
```

注記:

- Discord 音声は、テキストのみの構成ではオプトインです。`channels.discord.voice.enabled=true` を設定する（または既存の `channels.discord.voice` ブロックを維持する）と、`/vc` コマンド、音声ランタイム、および `GuildVoiceStates` Gateway インテントが有効になります。`channels.discord.intents.voiceStates` ではインテントのサブスクリプションを明示的に上書きできます。実効的な音声の有効化設定に従うには、未設定のままにしてください。
- `voice.mode` は会話パスを制御します。デフォルトは `agent-proxy` です。リアルタイム音声フロントエンドがターンのタイミング、中断、再生を処理し、実質的な処理を `openclaw_agent_consult` 経由でルーティング先の OpenClaw エージェントに委任し、その結果をその話者が入力した Discord プロンプトと同様に扱います。`stt-tts` は、従来のバッチ STT と TTS のフローを維持します。`bidi` では、OpenClaw の頭脳向けに `openclaw_agent_consult` を公開しながら、リアルタイムモデルが直接会話できます。
- `voice.agentSession` は、どの OpenClaw 会話が音声ターンを受信するかを制御します。音声チャンネル独自のセッションを使用する場合は未設定のままにします。または、`{ mode: "target", target: "channel:<text-channel-id>" }` に設定すると、音声チャンネルを `#maintainers` などの既存の Discord テキストチャンネルセッションに対するマイク／スピーカー拡張として機能させることができます。
- `voice.model` は、Discord 音声応答およびリアルタイム相談に使用する OpenClaw エージェントの頭脳を上書きします。ルーティング先エージェントのモデルを継承する場合は、未設定のままにしてください。これは `voice.realtime.model` とは別です。
- `voice.followUsers` を使用すると、ボットは選択されたユーザーとともに Discord 音声へ参加、移動、退出できます。[音声でユーザーを追跡する](#follow-users-in-voice)を参照してください。
- `agent-proxy` は、音声を `discord-voice` 経由でルーティングします。これにより、話者と対象セッションに対する通常の所有者／ツール認可は維持されますが、再生は Discord 音声が担当するため、エージェントの `tts` ツールは非表示になります。デフォルトでは、`agent-proxy` により、所有者である話者（`voice.realtime.toolPolicy: "owner"`）の相談には所有者と同等の完全なツールアクセスが付与され、実質的な回答を行う前に OpenClaw エージェントへ相談することが強く優先されます（`voice.realtime.consultPolicy: "always"`）。このデフォルトの `always` モードでは、リアルタイムレイヤーは相談の回答前に間をつなぐ発話を自動再生しません。音声を取得して文字起こしした後、ルーティング先の OpenClaw の回答を読み上げます。Discord が最初の回答を再生している間に複数の強制相談回答が完了した場合、後続の完全一致発話回答は文の途中で音声を置き換えず、再生がアイドル状態になるまでキューに入れられます。
- `stt-tts` モードでは、STT は `tools.media.audio` を使用します。`voice.model` は文字起こしに影響しません。
- リアルタイムモードでは、`voice.realtime.provider`、`voice.realtime.model`、および `voice.realtime.speakerVoice` がリアルタイム音声セッションを構成します。OpenAI Realtime 2.1 と Codex の頭脳を組み合わせる場合は、`voice.realtime.model: "gpt-realtime-2.1"` と `voice.model: "openai/gpt-5.6-sol"` を使用します。
- リアルタイム音声モードでは、デフォルトで小規模な `IDENTITY.md`、`USER.md`、および `SOUL.md` プロファイルファイルがリアルタイムプロバイダーの指示に含まれるため、高速な直接ターンでも、ルーティング先の OpenClaw エージェントと同じアイデンティティ、ユーザーに関する基盤情報、ペルソナが維持されます。これをカスタマイズするには `voice.realtime.bootstrapContextFiles` をサブセットに設定し、無効にするには `[]` を設定します。サポートされるのはこれらのプロファイルファイルのみです。`AGENTS.md` は通常のエージェントコンテキストに残ります。挿入されたプロファイルコンテキストは、ワークスペースでの作業、現在の事実、メモリ検索、またはツールを使用する操作における `openclaw_agent_consult` の代わりにはなりません。
- OpenAI の `agent-proxy` リアルタイムモードでは、ウェイクネームのゲートがデフォルトでルームの状況に適応します。人間が 1 人の場合はウェイクネームなしで自然に話せますが、2 人以上の場合は、ターンの先頭または末尾にウェイクネームを付ける必要があります。他のボットは人数に含まれません。ウェイクネームを常に必須にするには `voice.realtime.requireWakeName: true`、一切必須にしない場合は `false` を設定します。構成するウェイクネームは 1 語または 2 語でなければなりません。`voice.realtime.wakeNames` が未設定の場合、OpenClaw はルーティング先エージェントの `name` と `OpenClaw` を使用し、それらがない場合はエージェント ID と `OpenClaw` を使用します。ウェイクネームゲートが有効な場合、リアルタイムプロバイダーの自動応答が無効になり、受け付けたターンは OpenClaw エージェントへの相談パスを経由します。また、最終的な文字起こしが到着する前に、部分的な文字起こしから先頭のウェイクネームが認識されると、短い音声確認が返されます。このポリシーは、音声へ再接続することなく、リアルタイムの参加と退出に追随します。
- OpenAI リアルタイムプロバイダーは、出力音声イベントと文字起こしイベントについて、現在の Realtime 2 イベント名と従来の Codex 互換エイリアスを受け付けます。そのため、互換性のあるプロバイダースナップショットに差異が生じても、アシスタントの音声が欠落することはありません。
- `voice.realtime.bargeIn` は、Discord の話者開始イベントによって進行中のリアルタイム再生を中断するかどうかを制御します。未設定の場合は、リアルタイムプロバイダーの入力音声中断設定に従います。
- `voice.realtime.minBargeInAudioEndMs` は、OpenAI リアルタイムの割り込みによって音声が切り詰められるまでの、アシスタントの最小再生時間を制御します。デフォルト: `250`。エコーが少ないルームですぐに中断するには `0` を設定し、エコーが多いスピーカー環境では値を大きくします。
- `voice.tts` は、`stt-tts` 音声再生に限り `tts` を上書きします。リアルタイムモードでは代わりに `voice.realtime.speakerVoice` を使用します。Discord 再生で OpenAI の音声を使用するには、`voice.tts.provider: "openai"` を設定し、`voice.tts.providers.openai.speakerVoice` でテキスト読み上げ音声を選択します。`cedar` は、現在の OpenAI TTS モデルで男性的に聞こえる適切な選択肢です。
- チャンネル単位の Discord `systemPrompt` の上書きは、その音声チャンネルの音声文字起こしターンに適用されます。
- OpenClaw が音声チャンネルに参加すると、ルーティング先のエージェントセッションは、現在の参加者一覧を含む無音のシステムイベントを受信します。その後の参加者の参加と退出によってそのセッションは更新されますが、要求されていない音声応答は発生しません。Discord の表示名は信頼されていないラベルとして扱われます。認可された音声ターンも最新の参加者一覧のスナップショットを受信します。
- 音声文字起こしターンと `/vc` コマンドは、所有者ステータスの判定に `commands.ownerAllowFrom` 内の Discord エントリを使用します。Discord コマンドの所有者が構成されていない場合でも、選択された Discord アカウントの `allowFrom`（または従来の `dm.allowFrom`）によって、所有者ステータスを付与せずに音声アクセスを認可できます。エージェントのツール表示可否は、ルーティング先セッションに構成されたツールポリシーに従います。
- `voice.autoJoin` に同じギルドのエントリが複数ある場合、OpenClaw はそのギルドで最後に構成されたチャンネルに参加します。
- `voice.allowedChannels` は、任意の滞在許可リストです。`/vc join` が認可済みの任意の Discord 音声チャンネルに参加できるようにするには、未設定のままにします。設定すると、`/vc join`、起動時の自動参加、およびボットの音声状態による移動が、一覧に含まれる `{ guildId, channelId }` エントリに制限されます。Discord 音声へのすべての参加を拒否するには、空の配列に設定します。Discord がボットを許可リスト外へ移動した場合、OpenClaw はそのチャンネルから退出し、構成済みの自動参加先が利用可能であれば、そこへ再参加します。
- `voice.daveEncryption` と `voice.decryptionFailureTolerance` は、`@discordjs/voice` の参加オプションへそのまま渡されます。アップストリームのデフォルトは `daveEncryption=true` と `decryptionFailureTolerance=24` です。
- OpenClaw は、Discord 音声の受信とリアルタイムの raw PCM 再生に、同梱の `libopus-wasm` コーデックを使用します。固定バージョンの libopus WebAssembly ビルドが同梱されており、ネイティブの opus アドオンは必要ありません。
- `voice.connectTimeoutMs` は、`/vc join` および自動参加の試行における、初回の `@discordjs/voice` Ready 待機時間を制御します。デフォルト: `30000`。
- `voice.reconnectGraceMs` は、切断された音声セッションが再接続を開始するまで、OpenClaw がそのセッションを破棄せずに待機する時間を制御します。デフォルト: `15000`。
- `stt-tts` モードでは、別のユーザーが話し始めただけでは音声再生は停止しません。フィードバックループを避けるため、OpenClaw は TTS の再生中に新しい音声取得を無視します。次のターンは、再生の終了後に話してください。リアルタイムモードでは、話者の発話開始が割り込み信号としてリアルタイムプロバイダーへ転送されます。
- リアルタイムモードでは、スピーカーから開放状態のマイクに入るエコーが割り込みとして認識され、再生を中断することがあります。エコーが多い Discord ルームでは、入力音声による OpenAI の自動中断を防ぐために `voice.realtime.providers.openai.interruptResponseOnInputAudio: false` を設定します。それでも Discord の話者開始イベントで進行中の再生を中断する場合は、`voice.realtime.bargeIn: true` を追加します。OpenAI リアルタイムブリッジは、`voice.realtime.minBargeInAudioEndMs` 未満の再生切り詰めをエコー／ノイズの可能性が高いものとして無視し、Discord の再生をクリアせず、スキップとしてログに記録します。
- `voice.captureSilenceGraceMs` は、Discord が話者の発話終了を報告してから、OpenClaw がその音声セグメントを STT 用に確定するまでの待機時間を制御します。デフォルト: `2000`。Discord が通常の間を細切れの部分文字起こしに分割する場合は、値を大きくしてください。
- ElevenLabs が選択された TTS プロバイダーの場合、Discord の音声再生ではストリーミング TTS が使用され、プロバイダーの応答ストリームから再生が開始されます。ストリーミングをサポートしないプロバイダーは、合成された一時ファイルを使用するパスにフォールバックします。
- OpenClaw は受信時の復号失敗を監視し、短時間に失敗が繰り返された場合は、音声チャンネルから退出して再参加することで自動復旧します。
- 更新後に受信ログで `DecryptionFailed(UnencryptedWhenPassthroughDisabled)` が繰り返し表示される場合は、依存関係レポートとログを収集してください。同梱の `@discordjs/voice` 系列には、discord.js issue #11419 を解決した discord.js PR #11449 のアップストリームのパディング修正が含まれています。
- `The operation was aborted` 受信イベントは、OpenClaw が取得した話者セグメントを確定するときに想定されるものです。これは詳細診断であり、警告ではありません。
- Discord の詳細音声ログには、受け付けた各話者セグメントについて、長さが制限された 1 行の STT 文字起こしプレビューが含まれます。そのため、無制限の文字起こしテキストを出力することなく、デバッグ時にユーザー側とエージェントの応答側の両方を確認できます。
- `agent-proxy` モードでは、強制相談のフォールバックは、`...` で終わるテキストや、末尾の「and」のような接続語など、未完了である可能性が高い文字起こし断片、および「be right back」や「bye」のような明らかに操作を必要としない締めくくりをスキップします。これにより古いキュー内の回答が防止された場合、ログには `forced agent consult skipped reason=...` が表示されます。

### 音声でユーザーを追跡する

起動時に固定チャンネルへ参加したり、`/vc join` を待機したりする代わりに、Discord 音声ボットを既知の 1 人以上の Discord ユーザーと常に同じ場所にいさせる場合は、`voice.followUsers` を使用します。

```json5
{
  channels: {
    discord: {
      voice: {
        enabled: true,
        followUsersEnabled: true,
        followUsers: ["discord:123456789012345678"],
        allowedChannels: [
          {
            guildId: "123456789012345678",
            channelId: "234567890123456789",
          },
        ],
      },
    },
  },
}
```

動作:

- `followUsers` は、生の Discord ユーザー ID と `discord:<id>` 値を受け付けます。OpenClaw は、音声状態イベントと照合する前に両方の形式を正規化します。
- `followUsers` が設定されている場合、`followUsersEnabled` のデフォルトは `true` です。保存済みのリストを維持しつつ、音声の自動追従を停止するには、`false` に設定します。
- `followUsers` は、音声への在室のみを制御します。発言者アクセス権や所有者権限は付与しません。`commands.ownerAllowFrom` と、ギルドまたはチャンネルのユーザーおよびロールは個別に設定してください。
- 追従対象ユーザーが許可された音声チャンネルに参加すると、OpenClaw はそのチャンネルに参加します。ユーザーが移動すると、OpenClaw も一緒に移動します。アクティブな追従対象ユーザーが切断すると、OpenClaw は退出します。
- 複数の追従対象ユーザーが同じギルドにいて、アクティブな追従対象ユーザーが退出した場合、OpenClaw はギルドから退出する前に、追跡中の別の追従対象ユーザーのチャンネルへ移動します。複数の追従対象ユーザーが同時に移動した場合は、最後に観測された音声状態イベントが優先されます。
- `allowedChannels` は引き続き適用されます。許可されていないチャンネルにいる追従対象ユーザーは無視され、追従所有のセッションは別の追従対象ユーザーへ移動するか退出します。
- OpenClaw は、起動時および上限付きの間隔で、取りこぼした音声状態イベントを照合します。照合では設定済みのギルドを抽出し、1 回の実行あたりの REST ルックアップ数に上限を設けるため、非常に大きな `followUsers` リストは収束までに複数回の間隔を要する場合があります。
- ユーザーの追従中に Discord または管理者がボットを移動した場合、OpenClaw は音声セッションを再構築し、移動先が許可されていれば追従所有を維持します。ボットが `allowedChannels` の外へ移動された場合、OpenClaw は退出し、設定済みの対象が存在すればそこへ再参加します。
- DAVE の受信復旧では、復号に繰り返し失敗した後、同じチャンネルから退出して再参加する場合があります。追従所有のセッションは、その復旧経路でも追従所有を維持するため、その後に追従対象ユーザーが切断した場合もチャンネルから退出します。

参加モードを選択します。

- 自分が音声に参加しているときにボットも自動的に参加する必要がある、個人用またはオペレーター用のセットアップでは、`followUsers` を使用します。
- 追跡対象ユーザーが音声に参加していないときでも常駐する必要がある、固定ルーム用ボットでは、`autoJoin` を使用します。
- 単発の参加や、音声への自動参加が予想外となるルームでは、`/vc join` を使用します。

Discord 音声コーデック：

- 音声受信ログには `discord voice: opus decoder: libopus-wasm` が表示されます。
- リアルタイム再生では、生の 48 kHz ステレオ PCM を、同梱されている同じ `libopus-wasm` パッケージで Opus にエンコードしてから、パケットを `@discordjs/voice` に渡します。
- ファイルおよびプロバイダーストリームの再生では、ffmpeg を使用して生の 48 kHz ステレオ PCM にトランスコードしてから、Discord に送信する Opus パケットストリームに `libopus-wasm` を使用します。

STT と TTS のパイプライン：

- Discord の PCM キャプチャは、一時 WAV ファイルに変換されます。
- `tools.media.audio` が STT を処理します。たとえば `openai/gpt-4o-mini-transcribe` です。
- 文字起こしは Discord の受信処理とルーティングを通じて送信されます。その間、応答 LLM は、エージェントの `tts` ツールを非表示にし、テキストを返すよう求める音声出力ポリシーで実行されます。これは、最終的な TTS 再生を Discord 音声が所有するためです。
- `voice.model` が設定されている場合、この音声チャンネルのターンについてのみ応答 LLM を上書きします。
- `voice.tts` は `tts` にマージされます。ストリーミング対応のプロバイダーはプレーヤーへ直接データを送り、それ以外の場合は生成された音声ファイルが参加中のチャンネルで再生されます。

デフォルトのエージェントプロキシ音声チャンネルセッションの例：

```json5
{
  channels: {
    discord: {
      voice: {
        enabled: true,
        model: "openai/gpt-5.6-sol",
        followUsersEnabled: true,
        followUsers: ["123456789012345678"],
        realtime: {
          provider: "openai",
          model: "gpt-realtime-2.1",
          speakerVoice: "cedar",
        },
      },
    },
  },
}
```

`voice.agentSession` ブロックがない場合、音声チャンネルごとに独自のルーティング済み OpenClaw セッションが割り当てられます。たとえば、`/vc join channel:234567890123456789` はその Discord 音声チャンネルのセッションと対話します。リアルタイムモデルは音声フロントエンドにすぎず、実質的なリクエストは設定済みの OpenClaw エージェントへ渡されます。リアルタイムモデルが相談ツールを呼び出さずに最終的な文字起こしを生成した場合、OpenClaw はフォールバックとして相談を強制し、デフォルトでもエージェントと会話しているように動作させます。

従来の STT と TTS の例：

```json5
{
  channels: {
    discord: {
      voice: {
        enabled: true,
        mode: "stt-tts",
        model: "openai/gpt-5.4-mini",
        tts: {
          provider: "openai",
          providers: {
            openai: {
              model: "gpt-4o-mini-tts",
              speakerVoice: "cedar",
            },
          },
        },
      },
    },
  },
}
```

リアルタイム双方向通信の例：

```json5
{
  channels: {
    discord: {
      voice: {
        enabled: true,
        mode: "bidi",
        model: "openai/gpt-5.6-sol",
        realtime: {
          provider: "openai",
          model: "gpt-realtime-2.1",
          speakerVoice: "cedar",
          toolPolicy: "safe-read-only",
          consultPolicy: "always",
        },
      },
    },
  },
}
```

既存の Discord チャンネルセッションの拡張として音声を使用する例：

```json5
{
  channels: {
    discord: {
      voice: {
        enabled: true,
        mode: "agent-proxy",
        model: "openai/gpt-5.6-sol",
        agentSession: {
          mode: "target",
          target: "channel:123456789012345678",
        },
        realtime: {
          provider: "openai",
          model: "gpt-realtime-2.1",
          speakerVoice: "cedar",
        },
      },
    },
  },
}
```

`agent-proxy` モードでは、ボットは設定済みの音声チャンネルに参加しますが、OpenClaw エージェントのターンでは、対象チャンネルの通常のルーティング済みセッションとエージェントが使用されます。リアルタイム音声セッションは、返された結果を音声チャンネルで読み上げます。スーパーバイザーエージェントは、ツールポリシーに従って通常のメッセージツールを引き続き使用できます。適切なアクションであれば、別の Discord メッセージを送信することもできます。

委任された OpenClaw の実行がアクティブな間、新しい Discord 音声の文字起こしは、別のエージェントターンを開始する前に、進行中の実行を制御する入力として扱われます。「ステータス」、「それをキャンセル」、「より小規模な修正を使用」、「完了したらテストも確認」などのフレーズは、アクティブなセッションに対するステータス、キャンセル、方向修正、またはフォローアップ入力として分類されます。ステータス、キャンセル、受理された方向修正、およびフォローアップの結果は音声チャンネルで読み上げられるため、呼び出し元は OpenClaw がリクエストを処理したかどうかを把握できます。

有用な対象形式：

- `target: "channel:123456789012345678"` は、Discord テキストチャンネルセッションを通じてルーティングされます。
- `target: "123456789012345678"` は、チャンネル対象として扱われます。
- `target: "dm:123456789012345678"` または `target: "user:123456789012345678"` は、そのダイレクトメッセージセッションを通じてルーティングされます。

エコーが多い環境向けの OpenAI Realtime の例：

```json5
{
  channels: {
    discord: {
      voice: {
        enabled: true,
        mode: "bidi",
        model: "openai/gpt-5.6-sol",
        realtime: {
          provider: "openai",
          model: "gpt-realtime-2.1",
          speakerVoice: "cedar",
          bargeIn: true,
          minBargeInAudioEndMs: 500,
          consultPolicy: "always",
          providers: {
            openai: {
              interruptResponseOnInputAudio: false,
            },
          },
        },
      },
    },
  },
}
```

開いたマイクを通じてモデルが自身の Discord 再生音を拾う一方で、発話による割り込みも可能にしたい場合に使用します。OpenClaw は、生の入力音声による OpenAI の自動割り込みを抑止します。一方、`bargeIn: true` により、次にキャプチャされたターンが OpenAI に到達する前に、Discord の発言者開始イベントと、すでにアクティブな発言者の音声が、進行中のリアルタイム応答をキャンセルできます。`audioEndMs` が `minBargeInAudioEndMs` 未満の非常に早い割り込み信号は、エコーまたはノイズの可能性が高いものとして無視されるため、モデルが最初の再生フレームで中断されることはありません。

想定される音声ログ：

- 参加時：`discord voice: joining ... voiceSession=... supervisorSession=... agentSessionMode=... voiceModel=... realtimeModel=...`
- リアルタイム開始時：`discord voice: realtime bridge starting ... autoRespond=false interruptResponse=false bargeIn=false minBargeInAudioEndMs=...`
- 発言者音声の受信時：`discord voice: realtime speaker turn opened ...`、`discord voice: realtime input audio started ... outputAudioMs=... outputActive=...`、および `discord voice: realtime speaker turn closed ... chunks=... discordBytes=... realtimeBytes=... interruptedPlayback=...`
- 古い発話のスキップ時：`discord voice: realtime forced agent consult skipped reason=incomplete-transcript ...` または `reason=non-actionable-closing ...`
- リアルタイム応答の完了時：`discord voice: realtime audio playback finishing reason=response.done ... audioMs=... chunks=...`
- 再生の停止／リセット時：`discord voice: realtime audio playback stopped reason=... audioMs=... elapsedMs=... chunks=...`
- リアルタイム相談時：`discord voice: realtime consult requested ... voiceSession=... supervisorSession=... question=...`
- エージェント応答時：`discord voice: agent turn answer ...`
- 正確な発話のキュー登録時：`discord voice: realtime exact speech queued ... queued=... outputAudioMs=... outputActive=...`、続いて `discord voice: realtime exact speech dequeued reason=player-idle ...`
- 割り込みの検出時：`discord voice: realtime barge-in detected source=speaker-start ...` または `discord voice: realtime barge-in detected source=active-speaker-audio ...`、続いて `discord voice: realtime barge-in requested reason=... outputAudioMs=... outputActive=...`
- リアルタイム割り込み時：`discord voice: realtime model interrupt requested client:response.cancel reason=barge-in`、続いて `discord voice: realtime model audio truncated client:conversation.item.truncate reason=barge-in audioEndMs=...` または `discord voice: realtime model interrupt confirmed server:response.done status=cancelled ...`
- エコー／ノイズの無視時：`discord voice: realtime model interrupt ignored client:conversation.item.truncate.skipped reason=barge-in audioEndMs=0 minAudioEndMs=250`
- 割り込みの無効時：`discord voice: realtime capture ignored during playback (barge-in disabled) ...`
- アイドル再生時：`discord voice: realtime barge-in ignored reason=... outputActive=false ... playbackChunks=0`

音声が途中で切れる問題をデバッグするには、リアルタイム音声ログを時系列として確認します。

1. `realtime audio playback started` は、Discord がアシスタント音声の再生を開始したことを示します。この時点から、ブリッジはアシスタント出力チャンク、Discord PCM バイト、プロバイダーのリアルタイムバイト、合成音声の長さをカウントし始めます。
2. `realtime speaker turn opened` は、Discord の発言者がアクティブになったことを示します。再生がすでにアクティブで、`bargeIn` が有効な場合、その後に `barge-in detected source=speaker-start` が続くことがあります。
3. `realtime input audio started` は、その発言者ターンで最初の実際の音声フレームを受信したことを示します。ここで `outputActive=true`、またはゼロ以外の `outputAudioMs` が記録されている場合、アシスタントの再生がまだアクティブな間に、マイクが入力を送信していることを意味します。
4. `barge-in detected source=active-speaker-audio` は、アシスタントの再生中に OpenClaw が発言者のライブ音声を検出したことを意味します。これは、実際の割り込みと、有用な音声を伴わない Discord の発言者開始イベントを区別するのに役立ちます。
5. `barge-in requested reason=...` は、OpenClaw がリアルタイムプロバイダーに対し、進行中の応答のキャンセルまたは切り詰めを要求したことを意味します。`outputAudioMs`、`outputActive`、および `playbackChunks` が含まれているため、割り込み前に実際に再生されたアシスタント音声の量を確認できます。
6. `realtime audio playback stopped reason=...` は、ローカルの Discord 再生のリセット地点です。理由には、再生を停止した主体として、`barge-in`、`player-idle`、`provider-clear-audio`、`forced-agent-consult`、`stream-close`、または `session-close` が示されます。
7. `realtime speaker turn closed` は、キャプチャされた入力ターンを要約します。`chunks=0` または `hasAudio=false` は、発言者ターンは開始されたものの、使用可能な音声がリアルタイムブリッジに到達しなかったことを意味します。`interruptedPlayback=true` は、その入力ターンがアシスタント出力と重なり、割り込みロジックが発動したことを意味します。

有用なフィールド：

- `outputAudioMs`：そのログ行までにリアルタイムプロバイダーが生成したアシスタント音声の長さ。
- `audioMs`：再生が停止するまでに OpenClaw がカウントしたアシスタント音声の長さ。
- `elapsedMs`：再生ストリームまたは発言者ターンを開始してから終了するまでの実時間。
- `discordBytes`：Discord 音声へ送信、または Discord 音声から受信した 48 kHz ステレオ PCM バイト数。
- `realtimeBytes`：リアルタイムプロバイダーへ送信、またはリアルタイムプロバイダーから受信したプロバイダー形式の PCM バイト数。
- `playbackChunks`：進行中の応答について Discord へ転送されたアシスタント音声チャンク数。
- `sinceLastAudioMs`：最後にキャプチャされた発言者音声フレームから発言者ターンが終了するまでの間隔。

一般的なパターン：

- `source=active-speaker-audio` による即時の打ち切り、小さい `outputAudioMs`、および近くにいる同じユーザーは、通常、スピーカーのエコーがマイクに入っていることを示します。`voice.realtime.minBargeInAudioEndMs` を上げ、スピーカー音量を下げ、ヘッドフォンを使用するか、`voice.realtime.providers.openai.interruptResponseOnInputAudio: false` を設定してください。
- `source=speaker-start` の後に `speaker turn closed ... hasAudio=false` が続く場合、Discord は話者の発話開始を報告したものの、音声が OpenClaw に届かなかったことを意味します。これは一時的な Discord 音声イベント、ノイズゲートの動作、またはクライアントが一時的にマイクをオンにしたことが原因の場合があります。
- 近くでの割り込みまたは `provider-clear-audio` がない `audio playback stopped reason=stream-close` は、ローカルの Discord 再生ストリームが予期せず終了したことを意味します。直前のプロバイダーおよび Discord プレーヤーのログを確認してください。
- `capture ignored during playback (barge-in disabled)` は、アシスタント音声がアクティブな間、OpenClaw が意図的に入力を破棄したことを意味します。発話によって再生を中断する場合は、`voice.realtime.bargeIn` を有効にしてください。
- `barge-in ignored ... outputActive=false` は、Discord またはプロバイダーの VAD が発話を報告したものの、OpenClaw に中断対象のアクティブな再生がなかったことを意味します。これによって音声が打ち切られることはありません。

認証情報はコンポーネントごとに解決されます。`voice.model` の LLM ルート認証、`tools.media.audio` の STT 認証、`tts`/`voice.tts` の TTS 認証、および `voice.realtime.providers` またはプロバイダーの通常の認証設定によるリアルタイムプロバイダー認証です。

### ボイスメッセージ

Discord のボイスメッセージには波形プレビューが表示され、OGG/Opus 音声が必要です。OpenClaw は波形を自動生成しますが、検査と変換のために Gateway ホスト上の `ffmpeg` と `ffprobe` が必要です。

- **ローカルファイルパス**を指定してください（URL は拒否されます）。
- テキストコンテンツは省略してください（Discord は同じペイロード内のテキストとボイスメッセージを拒否します）。
- 任意の音声形式を使用できます。OpenClaw は必要に応じて OGG/Opus に変換します。

```bash
message(action="send", channel="discord", target="channel:123", path="/path/to/audio.mp3", asVoice=true)
```

## トラブルシューティング

<AccordionGroup>
  <Accordion title="許可されていないインテントが使用された、またはボットにギルドメッセージが表示されない">

    - Message Content Intent を有効にする
    - ユーザーまたはメンバーの解決に依存する場合は Server Members Intent を有効にする
    - インテントを変更した後に Gateway を再起動する

  </Accordion>

  <Accordion title="ギルドメッセージが予期せずブロックされる">

    - `groupPolicy` を確認する
    - `channels.discord.guilds` 配下のギルド許可リストを確認する
    - ギルドの `channels` マップが存在する場合、一覧にあるチャンネルのみが許可される
    - `requireMention` の動作とメンションパターンを確認する

    便利な確認コマンド：

```bash
openclaw doctor
openclaw channels status --probe
openclaw logs --follow
```

  </Accordion>

  <Accordion title="メンション必須を false にしてもブロックされる">
    一般的な原因：

    - 一致するギルドまたはチャンネルの許可リストがない `groupPolicy="allowlist"`
    - `requireMention` が誤った場所に設定されている（`channels.discord.guilds` またはチャンネルエントリ配下に配置する必要があります）
    - 送信者がギルドまたはチャンネルの `users` 許可リストによってブロックされている

  </Accordion>

  <Accordion title="長時間実行される Discord ターンまたは重複した返信">

    典型的なログ：

    - `Slow listener detected ...`
    - `stuck session: sessionKey=agent:...:discord:... state=processing ...`

    Discord は、キューに入ったエージェントターンにチャンネル固有のタイムアウトを適用しません。メッセージリスナーは直ちに処理を引き渡し、キューに入った Discord 実行は、セッション、ツール、またはランタイムのライフサイクルが完了するか処理を中止するまで、セッションごとの順序を維持します。

  </Accordion>

  <Accordion title="Gateway メタデータ検索のタイムアウト警告">
    OpenClaw は接続前に Discord の `/gateway/bot` メタデータを取得します。一時的な失敗時には Discord のデフォルト Gateway URL にフォールバックし、ログ出力はレート制限されます。

    メタデータのタイムアウトはデフォルトで 30 秒です。通常とは異なるホスト環境では、`OPENCLAW_DISCORD_GATEWAY_INFO_TIMEOUT_MS` で上書きできます。

  </Accordion>

  <Accordion title="Gateway READY タイムアウトによる再起動">
    OpenClaw は起動時およびランタイム再接続後に、Discord Gateway の `READY` イベントを待機します。起動を時間差で行う複数アカウント構成では、デフォルトより長い起動時 READY 待機時間が必要になる場合があります。

    起動時は 15 秒、ランタイム再接続時は 30 秒待機します。通常とは異なるホスト環境向けに、`OPENCLAW_DISCORD_READY_TIMEOUT_MS` と `OPENCLAW_DISCORD_RUNTIME_READY_TIMEOUT_MS` も引き続き使用できます。

  </Accordion>

  <Accordion title="権限監査の不一致">
    `channels status --probe` の権限チェックは、数値のチャンネル ID でのみ機能します。

    スラッグキーを使用している場合でもランタイムの照合は機能しますが、プローブでは権限を完全に検証できません。

  </Accordion>

  <Accordion title="DM とペアリングの問題">

    - DM が無効：`channels.discord.dm.enabled=false`
    - DM ポリシーが無効：`channels.discord.dmPolicy="disabled"`（レガシー：`channels.discord.dm.policy`）
    - `pairing` モードでペアリングの承認を待機中

  </Accordion>

  <Accordion title="ボット間のループ">
    デフォルトでは、ボットが作成したメッセージは無視されます。

    `channels.discord.allowBots=true` を設定する場合は、ループ動作を避けるために厳格なメンションおよび許可リストルールを使用してください。
    ボットへのメンションを含むボットメッセージのみを受け入れるには、`channels.discord.allowBots="mentions"` を推奨します。

    OpenClaw には共有の[ボットループ保護](/ja-JP/channels/bot-loop-protection)も同梱されています。`allowBots` によってボットが作成したメッセージがディスパッチに到達できる場合、Discord は受信イベントを `(account, channel, bot pair)` のファクトにマッピングし、汎用のペアガードは設定されたイベント予算を超えたペアを抑制します。このガードは、以前は Discord のレート制限によって停止する必要があった、制御不能な 2 ボット間ループを防ぎます。単一ボットのデプロイや、予算内に収まる単発のボット返信には影響しません。

    デフォルト設定（`allowBots` が設定されている場合に有効）：

    - `maxEventsPerWindow: 20` -- ボットペアはスライディングウィンドウ内で 20 件のメッセージを交換可能
    - `windowSeconds: 60` -- スライディングウィンドウの長さ
    - `cooldownSeconds: 60` -- 予算を超えると、どちらの方向でも以降のボット間メッセージはすべて 1 分間破棄される

    共有デフォルトを `channels.defaults.botLoopProtection` 配下で一度設定し、正当なワークフローでより大きな余裕が必要な場合に Discord 側で上書きします。優先順位は次のとおりです：

    - `channels.discord.accounts.<account>.botLoopProtection`
    - `channels.discord.botLoopProtection`
    - `channels.defaults.botLoopProtection`
    - 組み込みのデフォルト

    Discord は汎用の `maxEventsPerWindow`、`windowSeconds`、`cooldownSeconds` キーを使用します。

```json5
{
  channels: {
    defaults: {
      botLoopProtection: {
        maxEventsPerWindow: 20,
        windowSeconds: 60,
        cooldownSeconds: 60,
      },
    },
    discord: {
      // 任意の Discord 全体の上書き。アカウントブロックは個々の
      // フィールドを上書きし、省略されたフィールドをここから継承します。
      botLoopProtection: {
        maxEventsPerWindow: 4,
      },
      accounts: {
        alpha: {
          // Alpha は、他のボットが自身をメンションした場合にのみ、そのメッセージを受信します。
          allowBots: "mentions",
        },
        bravo: {
          // Bravo は、ボットが作成したすべての Discord メッセージを受信します。
          allowBots: true,
          mentionAliases: {
            // 設定されたユーザー ID を使用して、Bravo が Alpha の Discord メンションを記述できるようにします。
            Alpha: "ALPHA_DISCORD_USER_ID",
          },
          botLoopProtection: {
            // ペアを抑制する前に、1 分あたり最大 5 件のメッセージを許可します。
            maxEventsPerWindow: 5,
            windowSeconds: 60,
            cooldownSeconds: 90,
          },
        },
      },
    },
  },
}
```

  </Accordion>

  <Accordion title="DecryptionFailed(...) による音声 STT の欠落">

    - Discord 音声受信の復旧ロジックが含まれるよう、OpenClaw を最新の状態（`openclaw update`）に保つ
    - `channels.discord.voice.daveEncryption=true`（デフォルト）を確認する
    - `channels.discord.voice.decryptionFailureTolerance=24`（アップストリームのデフォルト）から開始し、必要な場合にのみ調整する
    - ログで以下を確認する：
      - `discord voice: DAVE decrypt failures detected`
      - `discord voice: repeated decrypt failures; attempting rejoin`
    - 自動再参加後も失敗が続く場合は、ログを収集し、[discord.js #11419](https://github.com/discordjs/discord.js/issues/11419) および [discord.js #11449](https://github.com/discordjs/discord.js/pull/11449) にあるアップストリームの DAVE 受信履歴と比較する

  </Accordion>
</AccordionGroup>

## 設定リファレンス

主要リファレンス：[設定リファレンス - Discord](/ja-JP/gateway/config-channels#discord)。

<Accordion title="重要度の高い Discord フィールド">

- 起動/認証：`enabled`、`token`、`applicationId`、`accounts.*`、`allowBots`
- ポリシー：`groupPolicy`、`dmPolicy`、`allowFrom`、`dm.*`、`guilds.*`、`guilds.*.channels.*`
- コマンド：`commands.native`、`commands.useAccessGroups`（グローバル）、`configWrites`、`slashCommand.ephemeral`
- Gateway：`proxy`
- 返信/履歴：`replyToMode`、`historyLimit`、`dmHistoryLimit`、`dms.*.historyLimit`
- 配信：`textChunkLimit`（デフォルト `2000`）、`maxLinesPerMessage`（デフォルト `17`）
- ストリーミング：`streaming.mode`、`streaming.chunkMode`、`streaming.preview.*`、`streaming.progress.*`、`streaming.block.*`（レガシーのフラットな `streamMode`、`draftChunk`、`blockStreaming`、`blockStreamingCoalesce`、`chunkMode` キーは、`openclaw doctor --fix` によって `streaming.*` に移行されます）
- メディア：`mediaMaxMb`（Discord への送信アップロードを制限、デフォルト `100`）
- アクション：`actions.*`
- プレゼンス：`activity`、`status`、`activityType`、`activityUrl`、`autoPresence.*`
- UI：`ui.components.accentColor`
- 機能：`threadBindings`、トップレベルの `bindings[]`（`type: "acp"`）、`pluralkit`、`execApprovals`、`intents`、`agentComponents.enabled`、`agentComponents.ttlMs`、`activities`、`heartbeat`、`responsePrefix`

</Accordion>

### Discord Activities

`channels.discord.activities` を設定すると、エージェントが Discord 内で開く自己完結型 HTML ウィジェットを投稿できるようになります。このブロックはオプトインです。存在しない場合、OpenClaw は Activity のルート、ツール、インタラクションハンドラーを登録しません。Developer Portal、トンネル、セキュリティ、トラブルシューティングの設定については、[Discord Activities](/channels/discord-activities) を参照してください。

- `activities.clientSecret`：Discord アプリケーションの OAuth2 クライアントシークレット。`DISCORD_CLIENT_SECRET` にフォールバックします
- `activities.applicationId`：任意の Activity アプリケーション ID。デフォルトでは Gateway 起動時に取得したボットアプリケーション ID を使用します

## 安全性と運用

- ボットトークンはシークレットとして扱ってください（監視環境では `DISCORD_BOT_TOKEN` を推奨）。
- Discord の権限は最小権限で付与してください。
- コマンドのデプロイまたは状態が古い場合は、Gateway を再起動し、`openclaw channels status --probe` でもう一度確認してください。

## 関連項目

<CardGroup cols={2}>
  <Card title="Discord Activities" icon="window" href="/channels/discord-activities">
    Discord 内でインタラクティブな HTML ウィジェットを起動します。
  </Card>
  <Card title="ペアリング" icon="link" href="/ja-JP/channels/pairing">
    Discord ユーザーを Gateway とペアリングします。
  </Card>
  <Card title="グループ" icon="users" href="/ja-JP/channels/groups">
    グループチャットと許可リストの動作。
  </Card>
  <Card title="チャンネルルーティング" icon="route" href="/ja-JP/channels/channel-routing">
    受信メッセージをエージェントにルーティングします。
  </Card>
  <Card title="セキュリティ" icon="shield" href="/ja-JP/gateway/security">
    脅威モデルと堅牢化。
  </Card>
  <Card title="マルチエージェントルーティング" icon="sitemap" href="/ja-JP/concepts/multi-agent">
    ギルドとチャンネルをエージェントにマッピングします。
  </Card>
  <Card title="スラッシュコマンド" icon="terminal" href="/ja-JP/tools/slash-commands">
    ネイティブコマンドの動作。
  </Card>
</CardGroup>
