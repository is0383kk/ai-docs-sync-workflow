---
read_when:
    - チャンネル Plugin の設定（認証、アクセス制御、複数アカウント）
    - チャンネル別設定キーのトラブルシューティング
    - DM ポリシー、グループポリシー、メンションゲートの監査
summary: チャンネル設定：Slack、Discord、Telegram、WhatsApp、Matrix、iMessage などにおけるアクセス制御、ペアリング、チャンネルごとのキー
title: 設定 — チャンネル
x-i18n:
    generated_at: "2026-07-26T10:13:24Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: e346648287d275d84a9c082a3bb13edaee751d53546d8231dcf1525bf9adafc2
    source_path: gateway/config-channels.md
    workflow: 16
---

`channels.*` 配下のチャンネル別設定キー：DM とグループのアクセス、マルチアカウント構成、メンションゲーティング、および Slack、Discord、Telegram、WhatsApp、Matrix、iMessage、その他のチャンネル Plugin 向けのチャンネル別キー。

エージェント、ツール、Gateway ランタイム、その他のトップレベルキーについては、[設定リファレンス](/ja-JP/gateway/configuration-reference)を参照してください。

## チャンネル

各チャンネルは、その設定セクションが存在すると自動的に起動します（`enabled: false` の場合を除く）。Telegram と iMessage はコア `openclaw` パッケージに同梱されています。その他の公式チャンネル（Discord、Slack、WhatsApp、Matrix、Microsoft Teams、IRC、Google Chat、Signal、Mattermost など）は、`openclaw plugins install <spec>` を使用して個別の Plugin としてインストールします。完全な一覧とインストール仕様については、[チャンネル](/ja-JP/channels)を参照してください。

### DM とグループのアクセス

すべてのチャンネルで DM ポリシーとグループポリシーがサポートされています。

| DM ポリシー           | 動作                                                        |
| ------------------- | --------------------------------------------------------------- |
| `pairing`（デフォルト） | 不明な送信者には一度限りのペアリングコードが発行され、所有者の承認が必要 |
| `allowlist`         | `allowFrom`（またはペアリング済み許可ストア）内の送信者のみ             |
| `open`              | すべての受信 DM を許可（`allowFrom: ["*"]` が必要）             |
| `disabled`          | すべての受信 DM を無視                                          |

| グループポリシー          | 動作                                               |
| --------------------- | ------------------------------------------------------ |
| `allowlist`（デフォルト） | 設定済みの許可リストに一致するグループのみ          |
| `open`                | グループの許可リストをバイパス（メンションゲーティングは引き続き適用） |
| `disabled`            | すべてのグループ／ルームメッセージをブロック                          |

<Note>
プロバイダーの `groupPolicy` が未設定の場合、`channels.defaults.groupPolicy` がデフォルトを設定します。
ペアリングコードは 1 時間後に期限切れになります。保留中のペアリング要求は、**アカウントごとに 3 件**（チャンネルとアカウント ID ごとにスコープ）に制限されます。
プロバイダーブロック自体が完全に欠けている場合（`channels.<provider>` が存在しない場合）、ランタイムのグループポリシーは起動時の警告を伴って `allowlist`（フェイルクローズ）にフォールバックします。
</Note>

### チャンネルのモデルオーバーライド

`channels.modelByChannel` を使用して、特定のチャンネル ID またはダイレクトメッセージの相手をモデルに固定します。値には `provider/model` または設定済みのモデルエイリアスを指定できます。チャンネルマッピングは、セッションに有効なモデルオーバーライドがまだない場合にのみ適用されます（たとえば、`/model` で設定されたもの）。

グループ／スレッド会話の場合、キーはチャンネル固有のグループ ID、トピック ID、またはチャンネル名です。ダイレクトメッセージ（DM）会話の場合、キーはチャンネルの送信者 ID（`nativeDirectUserId`、`origin.from`、`origin.to`、`OriginatingTo`、`From`、または `SenderId`）から派生した相手の識別子です。正確なキー形式はチャンネルによって異なります。

| チャンネル  | DM キー形式         | 例                                      |
| -------- | ------------------- | -------------------------------------------- |
| Discord  | 生のユーザー ID         | `987654321`                                  |
| Feishu   | `feishu:ou_...`     | `feishu:ou_a8b6cab7e945387de5f253775d9b4d85` |
| Matrix   | Matrix ユーザー ID      | `@user:matrix.org`                           |
| Slack    | `user:U...`         | `user:U12345`                                |
| Telegram | 生のユーザー ID         | `123456789`                                  |
| WhatsApp | 電話番号または JID | `15551234567`                                |

```json5
{
  channels: {
    modelByChannel: {
      discord: {
        "123456789012345678": "anthropic/claude-opus-4-6",
      },
      slack: {
        C1234567890: "openai/gpt-5.6-sol",
        "user:U12345": "openai/gpt-5.4-mini",
      },
      telegram: {
        "-1001234567890": "openai/gpt-5.4-mini",
        "-1001234567890:topic:99": "anthropic/claude-sonnet-4-6",
        "123456789": "openai/gpt-4.1",
      },
    },
  },
}
```

DM 固有のキーはダイレクトメッセージ会話でのみ一致し、グループ／スレッドのルーティングには影響しません。

### チャンネルのデフォルトと Heartbeat

プロバイダー間で共有されるグループポリシー、暗黙的メンション、Heartbeat の動作には `channels.defaults` を使用します。

```json5
{
  channels: {
    defaults: {
      groupPolicy: "allowlist", // open | allowlist | disabled
      contextVisibility: "all", // all | allowlist | allowlist_quote
      implicitMentions: {
        replyToBot: true,
        quotedBot: true,
        threadParticipation: true,
      },
      heartbeat: {
        showOk: false,
        showAlerts: true,
        useIndicator: true,
      },
    },
  },
}
```

- `channels.defaults.groupPolicy`：プロバイダーレベルの `groupPolicy` が未設定の場合のフォールバックグループポリシー。
- `channels.defaults.contextVisibility`：すべてのチャンネルに対する補足コンテキストのデフォルト表示モード。値：`all`（デフォルト、引用／スレッド／履歴のすべてのコンテキストを含める）、`allowlist`（許可リスト内の送信者からのコンテキストのみを含める）、`allowlist_quote`（許可リストと同じだが、明示的な引用／返信コンテキストは保持）。チャンネル別のオーバーライド：`channels.<channel>.contextVisibility`。
- `channels.defaults.implicitMentions`：サポートされている受信情報のうち、どれをメンションとして扱うかを制御します。`replyToBot`、`quotedBot`、`threadParticipation` はそれぞれデフォルトで `true` となり、現在の動作を維持します。チャンネルごとに `channels.<channel>.implicitMentions`、アカウントごとに `channels.<channel>.accounts.<id>.implicitMentions` でオーバーライドできます。各フラグは、アカウント → チャンネル → デフォルトの順に個別に解決されます。名前は肯定形です。その情報がメンションゲーティングをバイパスしないようにするには、フラグを `false` に設定します。ネイティブの明示的メンションは常に許可され、チャンネルがその情報を生成しない場合、フラグは効果を持ちません。現在の生成元マトリクスについては、[メンションゲーティング](/ja-JP/channels/groups#mention-gating-default)を参照してください。これらの設定は、送信時の返信／スレッドモードや、承認済みコマンドの処理には影響しません。
- `channels.defaults.heartbeat.showOk`：Heartbeat 出力に正常なチャンネルステータスを含めます（デフォルトは `false`）。
- `channels.defaults.heartbeat.showAlerts`：Heartbeat 出力に劣化／エラーステータスを含めます（デフォルトは `true`）。
- `channels.defaults.heartbeat.useIndicator`：コンパクトなインジケータースタイルの Heartbeat 出力をレンダリングします（デフォルトは `true`）。

### WhatsApp

WhatsApp は Gateway の Web チャンネル（Baileys Web）を介して動作します。リンク済みセッションが存在すると自動的に起動します。

```json5
{
  web: {
    enabled: true,
  },
  channels: {
    whatsapp: {
      dmPolicy: "pairing", // pairing | allowlist | open | disabled
      allowFrom: ["+15555550123", "+447700900123"],
      textChunkLimit: 4000,
      streaming: { chunkMode: "length" }, // length | newline
      mediaMaxMb: 50,
      sendReadReceipts: true, // blue ticks (false in self-chat mode)
      groups: {
        "*": { requireMention: true },
      },
      groupPolicy: "allowlist",
      groupAllowFrom: ["+15551234567"],
    },
  },
}
```

- トップレベルの `bindings[]` エントリで `type: "acp"` を使用すると、WhatsApp の DM とグループ向けの永続的な ACP バインディングを設定できます。`match.peer.id` には E.164 形式の直接番号または WhatsApp グループ JID を使用します。フィールドの意味は [ACP エージェント](/ja-JP/tools/acp-agents#persistent-channel-bindings)で共通化されています。

<Accordion title="マルチアカウント WhatsApp">

```json5
{
  channels: {
    whatsapp: {
      accounts: {
        default: {},
        personal: {},
        biz: {
          // authDir: "~/.openclaw/credentials/whatsapp/biz",
        },
      },
    },
  },
}
```

- 送信コマンドでは、アカウント `default` が存在する場合はそれがデフォルトになります。存在しない場合は、設定済みの最初のアカウント ID（ソート順）が使用されます。
- 省略可能な `channels.whatsapp.defaultAccount` が設定済みのアカウント ID と一致する場合、そのフォールバックのデフォルトアカウント選択をオーバーライドします。
- 従来のシングルアカウント Baileys 認証ディレクトリは、`openclaw doctor` によって `whatsapp/default` に移行されます。
- アカウント別のオーバーライド：`channels.whatsapp.accounts.<id>.sendReadReceipts`、`channels.whatsapp.accounts.<id>.dmPolicy`、`channels.whatsapp.accounts.<id>.allowFrom`。

</Accordion>

### Telegram

```json5
{
  channels: {
    telegram: {
      enabled: true,
      botToken: "your-bot-token",
      dmPolicy: "pairing",
      allowFrom: ["tg:123456789"],
      groups: {
        "*": { requireMention: true },
        "-1001234567890": {
          allowFrom: ["@admin"],
          systemPrompt: "Keep answers brief.",
          topics: {
            "99": {
              requireMention: false,
              skills: ["search"],
              systemPrompt: "Stay on topic.",
            },
          },
        },
      },
      customCommands: [
        { command: "backup", description: "Git backup" },
        { command: "generate", description: "Create an image" },
      ],
      historyLimit: 50,
      replyToMode: "first", // off | first | all | batched
      linkPreview: true,
      streaming: { mode: "partial" }, // off | partial | block | progress (default: partial)
      actions: { reactions: true, sendMessage: true },
      reactionNotifications: "own", // off | own | all
      mediaMaxMb: 100,
      retry: {
        attempts: 3,
        minDelayMs: 400,
        maxDelayMs: 30000,
        jitter: 0.1,
      },
      network: {
        autoSelectFamily: true,
        dnsResultOrder: "ipv4first",
      },
      apiRoot: "https://api.telegram.org",
      trustedLocalFileRoots: ["/srv/telegram-bot-api-data"],
      proxy: "socks5://localhost:9050",
      webhookUrl: "https://example.com/telegram-webhook",
      webhookSecret: "secret",
      webhookPath: "/telegram-webhook",
    },
  },
}
```

- Bot トークン：`channels.telegram.botToken` または `channels.telegram.tokenFile`（通常ファイルのみ。シンボリックリンクは拒否）を使用し、デフォルトアカウントでは `TELEGRAM_BOT_TOKEN` がフォールバックになります。
- `apiRoot` は Telegram Bot API のルート専用です。`https://api.telegram.org/bot<TOKEN>` ではなく、`https://api.telegram.org` またはセルフホスト／プロキシのルートを使用してください。`openclaw doctor --fix` は誤って末尾に付いた `/bot<TOKEN>` サフィックスを削除します。
- `--local` モードのセルフホスト Bot API サーバーでは、`trustedLocalFileRoots` に OpenClaw が読み取り可能なホストパスを列挙します。OpenClaw ホストにサーバーのデータボリュームをマウントし、そのデータルートまたはトークン別ディレクトリのいずれかを設定します。`/var/lib/telegram-bot-api` 配下のコンテナパスは、それらのルートにマッピングされます。その他の絶対パスは引き続き拒否されます。
- 省略可能な `channels.telegram.defaultAccount` が設定済みのアカウント ID と一致する場合、デフォルトのアカウント選択をオーバーライドします。
- マルチアカウント構成（アカウント ID が 2 つ以上）では、フォールバックルーティングを避けるため、明示的なデフォルト（`channels.telegram.defaultAccount` または `channels.telegram.accounts.default`）を設定します。これが欠けているか無効な場合、`openclaw doctor` が警告します。
- `configWrites: false` は、Telegram によって開始される設定の書き込み（スーパーグループ ID の移行、`/config set|unset`）をブロックします。
- トップレベルの `bindings[]` エントリで `type: "acp"` を使用すると、フォーラムトピック向けの永続的な ACP バインディングを設定できます（`match.peer.id` では正規形式の `chatId:topic:topicId` を使用）。フィールドの意味は [ACP エージェント](/ja-JP/tools/acp-agents#persistent-channel-bindings)で共通化されています。
- Telegram のストリームプレビューでは `sendMessage` と `editMessageText` を使用します（ダイレクトチャットとグループチャットの両方で動作）。
- `network.dnsResultOrder` のデフォルトは `"ipv4first"` で、一般的な IPv6 取得エラーを回避します。
- 再試行ポリシーについては、[再試行ポリシー](/ja-JP/concepts/retry)を参照してください。

### Discord

```json5
{
  channels: {
    discord: {
      enabled: true,
      token: "your-bot-token",
      mediaMaxMb: 100,
      allowBots: false,
      actions: {
        reactions: true,
        stickers: true,
        polls: true,
        permissions: true,
        messages: true,
        threads: true,
        pins: true,
        search: true,
        memberInfo: true,
        roleInfo: true,
        roles: false,
        channelInfo: true,
        voiceStatus: true,
        events: true,
        moderation: false,
      },
      replyToMode: "off", // off | first | all | batched
      dmPolicy: "pairing",
      allowFrom: ["1234567890", "123456789012345678"],
      dm: { enabled: true, groupEnabled: false, groupChannels: ["openclaw-dm"] },
      guilds: {
        "123456789012345678": {
          slug: "friends-of-openclaw",
          requireMention: false,
          ignoreOtherMentions: true,
          reactionNotifications: "own",
          users: ["987654321098765432"],
          channels: {
            general: { allow: true },
            help: {
              allow: true,
              requireMention: true,
              users: ["987654321098765432"],
              skills: ["docs"],
              systemPrompt: "短い回答のみ。",
            },
          },
        },
      },
      historyLimit: 20,
      textChunkLimit: 2000,
      suppressEmbeds: true,
      streaming: {
        mode: "progress", // off | partial | block | progress（Discord のデフォルト: progress）
        chunkMode: "length", // length | newline
        progress: {
          label: "auto",
          maxLines: 8,
          maxLineChars: 120,
          toolProgress: true,
        },
      },
      maxLinesPerMessage: 17,
      ui: {
        components: {
          accentColor: "#5865F2",
        },
      },
      threadBindings: {
        enabled: true,
        idleHours: 24,
        maxAgeHours: 0,
        spawnSessions: true,
        defaultSpawnContext: "fork",
      },
      voice: {
        enabled: true,
        autoJoin: [
          {
            guildId: "123456789012345678",
            channelId: "234567890123456789",
          },
        ],
        daveEncryption: true,
        decryptionFailureTolerance: 24,
        connectTimeoutMs: 30000,
        reconnectGraceMs: 15000,
        tts: {
          provider: "openai",
          openai: { voice: "alloy" },
        },
      },
      execApprovals: {
        enabled: "auto", // true | false | "auto"
        approvers: ["987654321098765432"],
        agentFilter: ["default"],
        sessionFilter: ["discord:"],
        target: "dm", // dm | channel | both
        cleanupAfterResolve: false,
      },
      retry: {
        attempts: 3,
        minDelayMs: 500,
        maxDelayMs: 30000,
        jitter: 0.1,
      },
    },
  },
}
```

- トークン: `channels.discord.token`。デフォルトアカウントのフォールバックとして `DISCORD_BOT_TOKEN` を使用します。
- 明示的な Discord `token` を指定する直接の送信呼び出しでは、その呼び出しに該当トークンが使用されます。アカウントの再試行およびポリシー設定は、引き続きアクティブなランタイムスナップショットで選択されたアカウントから取得されます。
- オプションの `channels.discord.defaultAccount` は、設定済みのアカウント ID と一致する場合、デフォルトのアカウント選択を上書きします。
- 配信先には `user:<id>`（DM）または `channel:<id>`（ギルドチャンネル）を使用します。数字だけの ID は拒否されます。
- ギルドのスラッグは小文字で、スペースを `-` に置き換えます。チャンネルキーにはスラッグ化した名前を使用します（`#` は付けません）。ギルド ID の使用を推奨します。
- Bot が作成したメッセージはデフォルトで無視されます。`allowBots: true` で有効にできます。Bot にメンションした Bot メッセージのみを受け入れるには `allowBots: "mentions"` を使用します（自身のメッセージは引き続き除外されます）。
- Bot が作成した受信メッセージをサポートするチャンネルでは、共通の [Bot ループ保護](/ja-JP/channels/bot-loop-protection)を使用できます。基本となるペア予算を `channels.defaults.botLoopProtection` で設定し、特定のサーフェスに異なる制限が必要な場合にのみ、チャンネルまたはアカウントで上書きします。
- `channels.discord.guilds.<id>.ignoreOtherMentions`（およびチャンネルごとの上書き）は、Bot ではなく別のユーザーまたはロールにメンションしているメッセージを除外します（@everyone/@here は除きます）。
- `channels.discord.mentionAliases` は、送信前に安定した送信 `@handle` テキストを Discord ユーザー ID にマッピングします。これにより、一時的なディレクトリキャッシュが空でも、既知のチームメイトを決定論的にメンションできます。アカウントごとの上書きは `channels.discord.accounts.<accountId>.mentionAliases` に配置します。
- `maxLinesPerMessage`（デフォルト `17`）は、2000 文字未満でも縦に長いメッセージを分割します。
- `channels.discord.suppressEmbeds` のデフォルトは `true` であるため、無効にしない限り、送信 URL は Discord のリンクプレビューとして展開されません。明示的な `embeds` ペイロードは通常どおり送信されます。メッセージごとのツール呼び出しでは `suppressEmbeds` によって上書きできます。
- `channels.discord.threadBindings` は、Discord のスレッドにバインドされたルーティングを制御します。
  - `enabled`: スレッドにバインドされたセッション機能（`/focus`、`/unfocus`、`/agents`、`/session idle`、`/session max-age`、およびバインドされた配信・ルーティング）に対する Discord の上書き
  - `idleHours`: 非アクティブ時に自動でフォーカスを解除するまでの時間数に対する Discord の上書き（`0` で無効化）
  - `maxAgeHours`: 最大存続時間（時間単位）に対する Discord の上書き（`0` で無効化）
  - `spawnSessions`: `sessions_spawn({ thread: true })` および ACP のスレッド生成時における、スレッドの自動作成・バインド用スイッチ（デフォルト: `true`）
  - `defaultSpawnContext`: スレッドにバインドされた生成用のネイティブサブエージェントコンテキスト（デフォルトは `"fork"`）
- `type: "acp"` を持つトップレベルの `bindings[]` エントリは、チャンネルおよびスレッドの永続的な ACP バインドを設定します（`match.peer.id` にはチャンネルまたはスレッド ID を使用します）。フィールドのセマンティクスについては、[ACP エージェント](/ja-JP/tools/acp-agents#persistent-channel-bindings)で共通に説明しています。
- `channels.discord.ui.components.accentColor` は、Discord components v2 コンテナのアクセントカラーを設定します。
- `channels.discord.agentComponents.ttlMs` は、送信済みの Discord コンポーネントコールバックを登録状態に保つ時間を制御します。デフォルトは `1800000`（30 分）、最大は `86400000`（24 時間）です。アカウントごとの上書きは `channels.discord.accounts.<accountId>.agentComponents.ttlMs` に配置します。ワークフローに適合する最短の TTL を推奨します。
- `channels.discord.voice` は、Discord ボイスチャンネルでの会話と、オプションの自動参加、LLM、TTS の上書きを有効にします。テキストのみの Discord 設定では、ボイスはデフォルトで無効です。有効にするには `channels.discord.voice.enabled=true` を設定します。
- `channels.discord.voice.model` は、Discord ボイスチャンネルへの応答に使用する LLM モデルを任意で上書きします。
- `channels.discord.voice.daveEncryption`（デフォルト `true`）および `channels.discord.voice.decryptionFailureTolerance`（デフォルト `24`）は、`@discordjs/voice` の DAVE オプションにそのまま渡されます。
- `channels.discord.voice.connectTimeoutMs` は、`/vc join` と自動参加試行における、最初の `@discordjs/voice` Ready 待機を制御します（デフォルト `30000`）。
- `channels.discord.voice.reconnectGraceMs` は、切断されたボイスセッションが再接続シグナリングに移行するまで、OpenClaw がそのセッションを破棄せずに待機する時間を制御します（デフォルト `15000`）。
- Discord の音声再生は、別のユーザーの発話開始イベントによって中断されません。フィードバックループを防ぐため、OpenClaw は TTS の再生中に新しい音声キャプチャを無視します。
- さらに OpenClaw は、復号の失敗が繰り返された場合にボイスセッションから退出して再参加し、音声受信の復旧を試みます。
- `channels.discord.streaming` は正規のストリームモードキーです。Discord のデフォルトは `streaming.mode: "progress"` であるため、ツールや作業の進行状況は、編集される 1 件のプレビューメッセージに表示されます。無効にするには `streaming.mode: "off"` を設定します。従来のフラットキー（`streamMode`、`chunkMode`、`blockStreaming`、`draftChunk`、`blockStreamingCoalesce`）はランタイムで読み込まれなくなりました。永続化された設定を移行するには `openclaw doctor --fix` を実行します。
- `channels.discord.autoPresence` は、ランタイムの可用性を Bot のプレゼンスにマッピングし（正常 => オンライン、劣化 => アイドル、枯渇 => dnd）、任意のステータステキスト上書きを許可します。
- `channels.discord.guilds.<id>.presenceEvents` は、ユーザーが利用可能になったことを示す到着イベントを、設定済みの 1 つの Discord チャンネルへエージェントのシステムイベントとしてルーティングします。対象メンバーは `channelId` を閲覧できる必要があります。公開スレッドは親の可視性を継承し、非公開スレッドではさらにメンバーシップまたは Manage Threads が必要です。`users` で対象者をさらに絞り込めます。完全な `GUILD_CREATE` スナップショットから現在オンラインのメンバーを初期登録し、観測されたオフラインからオンラインへの遷移をルーティングします。また、未確認のメンバーから後で初めてオンライン信号を受信した場合、スナップショット後にオンラインになったのか参加したのかは断定せず、新たに利用可能になったものとして扱います。Discord の 75,000 メンバーのスナップショット上限を超えるギルドでは、最初に明示的なオフライン更新が必要です。スロットリング設定: `reconnectSuppressSeconds`（ギルドのプレゼンス状態を再構築している間、新しい Gateway セッションの後に設ける静止期間。デフォルト 300、`0` で無効化）および `burstLimit`/`burstWindowSeconds`（ギルドごとに正常にキュー投入されたイベントのレート制限。デフォルトは 60s のスライディングウィンドウあたり 8 イベント）。再開されたセッションでは、再接続抑制ウィンドウは開始されません。既存のユーザーごとの再挨拶クールダウンは 8 時間のままです。これには `channels.discord.intents.presence=true`、Discord の Developer Portal にある特権 Presence Intent、および有効なエージェント Heartbeat が必要です。
- `channels.discord.dangerouslyAllowNameMatching` は、変更可能な名前・タグによる照合を再度有効にします（緊急時の互換モード）。
- `channels.discord.execApprovals`: Discord ネイティブの実行承認配信および承認者の認可。
  - `enabled`: `true`、`false`、または `"auto"`（デフォルト）。自動モードでは、`approvers` または `commands.ownerAllowFrom` から承認者を解決できる場合、実行承認が有効になります。
  - `approvers`: 実行リクエストの承認を許可する Discord ユーザー ID。省略した場合は `commands.ownerAllowFrom` にフォールバックします。
  - `agentFilter`: オプションのエージェント ID 許可リスト。すべてのエージェントの承認を転送する場合は省略します。
  - `sessionFilter`: オプションのセッションキーパターン（部分文字列または正規表現）。
  - `target`: 承認プロンプトの送信先。`"dm"`（デフォルト）は承認者の DM に送信し、`"channel"` は送信元チャンネルに送信し、`"both"` は両方に送信します。送信先に `"channel"` が含まれる場合、ボタンを使用できるのは解決済みの承認者のみです。
  - `cleanupAfterResolve`: `true` の場合、承認、拒否、またはタイムアウト後に承認 DM を削除します。

**リアクション通知モード:** `off`（なし）、`own`（Bot のメッセージ、デフォルト）、`all`（すべてのメッセージ）、`allowlist`（すべてのメッセージに対する `guilds.<id>.users` からのリアクション）。

### Google Chat

```json5
{
  channels: {
    googlechat: {
      enabled: true,
      serviceAccountFile: "/path/to/service-account.json",
      audienceType: "app-url", // app-url | project-number
      audience: "https://gateway.example.com/googlechat",
      webhookPath: "/googlechat",
      botUser: "users/1234567890",
      dmPolicy: "pairing",
      allowFrom: ["users/1234567890"],
      groupPolicy: "allowlist",
      groups: {
        "spaces/AAAA": { allow: true, requireMention: true },
      },
      actions: { reactions: true },
      typingIndicator: "message",
      mediaMaxMb: 20,
    },
  },
}
```

- サービスアカウント JSON: インライン（`serviceAccount`）またはファイルベース（`serviceAccountFile`）。
- `serviceAccount` は SecretRef を直接受け入れます。
- 環境変数のフォールバック: `GOOGLE_CHAT_SERVICE_ACCOUNT` または `GOOGLE_CHAT_SERVICE_ACCOUNT_FILE`（デフォルトアカウントのみ）。
- 配信先には `spaces/<spaceId>` または `users/<userId>` を使用します。
- `channels.googlechat.dangerouslyAllowNameMatching` は、変更可能なメールプリンシパルによる照合を再度有効にします（緊急時の互換モード）。

### Slack

```json5
{
  channels: {
    slack: {
      enabled: true,
      botToken: "xoxb-...",
      appToken: "xapp-...",
      socketMode: {
        clientPingTimeout: 15000,
        serverPingTimeout: 30000,
        pingPongLoggingEnabled: false,
      },
      dmPolicy: "pairing",
      allowFrom: ["U123", "U456", "*"],
      dm: { enabled: true, groupEnabled: false, groupChannels: ["G123"] },
      channels: {
        C123: { enabled: true, requireMention: true, allowBots: false },
        "#general": {
          enabled: true,
          requireMention: true,
          allowBots: false,
          users: ["U123"],
          skills: ["docs"],
          systemPrompt: "短い回答のみ。",
        },
      },
      historyLimit: 50,
      allowBots: false,
      reactionNotifications: "own",
      reactionAllowlist: ["U123"],
      replyToMode: "off", // off | first | all | batched
      thread: {
        historyScope: "thread", // thread | channel
        inheritParent: false,
        initialHistoryLimit: 20,
      },
      actions: {
        reactions: true,
        messages: true,
        pins: true,
        memberInfo: true,
        emojiList: true,
      },
      slashCommand: {
        enabled: true,
        name: "openclaw",
        sessionPrefix: "slack:slash",
        ephemeral: true,
      },
      typingReaction: "hourglass_flowing_sand",
      unfurlLinks: false,
      unfurlMedia: false,
      textChunkLimit: 4000,
      streaming: {
        mode: "partial", // off | partial | block | progress
        chunkMode: "length", // length | newline
        nativeTransport: true, // mode=partial の場合に Slack ネイティブストリーミング API を使用
      },
      mediaMaxMb: 20,
      execApprovals: {
        enabled: "auto", // true | false | "auto"
        approvers: ["U123"],
        agentFilter: ["default"],
        sessionFilter: ["slack:"],
        target: "dm", // dm | channel | both
      },
    },
  },
}
```

- **Socket モード**には、`botToken` と `appToken` の両方が必要です（デフォルトアカウントの環境変数フォールバックには `SLACK_BOT_TOKEN` + `SLACK_APP_TOKEN`）。
- **HTTP モード**には、`botToken` に加えて `signingSecret`（ルートまたはアカウント単位）が必要です。
- **ユーザー ID**（`identity: "user"`）は、認可した人間として投稿および読み取りを行います。Socket モードでは `userToken` と `appToken`、HTTP モードでは `userToken` と `signingSecret` が必要です。ボットトークンやボットユーザーは必要ありません。ユーザースコープとイベントサブスクリプションについては、[ユーザー ID](/ja-JP/channels/slack#user-identity-post-as-a-real-person)を参照してください。
- `enterpriseOrgInstall: true` は、アカウントを Slack Enterprise Grid の
  組織全体イベントパスにオプトインします。起動時に `auth.test` でボットトークンを検証し、
  設定されたモードが Slack のインストール ID と一致しない場合は
  失敗します。Enterprise DM は無効にするか、有効な
  `allowFrom: ["*"]` とともに `dmPolicy: "open"` を使用する必要があります。
  チャンネルおよびユーザーポリシーでは、安定した Slack ID を使用する必要があります。
  変更可能な名前やサポートされていないチャンネルプレフィックスがあると起動に失敗します。V1 が処理するのは、
  直接の Socket モードまたは HTTP の `message` および `app_mention` イベントと即時応答のみです。
  リレー、コマンド、インタラクション、App Home、リアクションイベントリスナー、
  ピン、アクションツール、ネイティブ承認、バインディング、遅延配信、
  プロアクティブ送信は利用できません。リスナーが所有する確認応答、入力中表示、
  ステータスリアクションは `reactions:write` で引き続き利用できますが、受信リアクション
  通知とリアクションアクションツールは利用できません。最小権限のマニフェスト、
  セットアップワークフロー、完全な制限事項については、
  [Enterprise Grid の組織全体インストール](/ja-JP/channels/slack#enterprise-grid-org-wide-installs)
  を参照してください。
- `socketMode` は、Slack SDK Socket モードのトランスポート調整を公開 Bolt レシーバー API に渡します。ping/pong タイムアウトや古い WebSocket の動作を調査する場合にのみ使用してください。`clientPingTimeout` のデフォルトは `15000` です。`serverPingTimeout` と `pingPongLoggingEnabled` は、設定されている場合にのみ渡されます。
- `botToken`、`appToken`、`signingSecret`、`userToken` は、プレーンテキストの
  文字列または SecretRef オブジェクトを受け入れます。
- Slack アカウントのスナップショットは、
  `botTokenSource`、`botTokenStatus`、`userTokenSource`、`userTokenStatus`、
  `appTokenStatus`、および HTTP モードでは `signingSecretStatus` など、認証情報ごとのソース／ステータスフィールドを公開します。
  `configured_unavailable` は、アカウントが
  SecretRef を介して設定されているものの、現在のコマンド／ランタイムパスでは
  シークレット値を解決できなかったことを意味します。
- `configWrites: false` は、Slack から開始される設定の書き込みをブロックします。
- オプションの `channels.slack.defaultAccount` は、設定済みのアカウント ID と一致する場合、デフォルトのアカウント選択を上書きします。
- `channels.slack.streaming.mode` は、Slack ストリームモードの正規キーです（デフォルトは `"partial"`）。`channels.slack.streaming.nativeTransport` は、Slack のネイティブストリーミングトランスポートを制御します（デフォルトは `true`）。従来の `streamMode`、ブール値の `streaming`、`chunkMode`、`blockStreaming`、`blockStreamingCoalesce`、`nativeStreaming` の値は、ランタイムでは読み取られなくなりました。`openclaw doctor --fix` を実行して、永続化された設定を `streaming.{mode,chunkMode,block.enabled,block.coalesce,nativeTransport}` に移行してください。
- `unfurlLinks` と `unfurlMedia` は、ボットの返信について Slack の `chat.postMessage` リンクおよびメディア展開のブール値を渡します。`unfurlLinks` のデフォルトは `false` であるため、有効にしない限り、送信ボットリンクはインライン展開されません。`unfurlMedia` は、設定されていない場合は省略されます。1 つのアカウントでトップレベルの値を上書きするには、いずれかの値を `channels.slack.accounts.<accountId>` に設定します。
- 配信先には `user:<id>`（DM）または `channel:<id>` を使用します。

**リアクション通知モード：** `off`、`own`（デフォルト）、`all`、`allowlist`（`reactionAllowlist` から）。

**スレッドセッションの分離：** `thread.historyScope` はスレッドごと（デフォルト）、またはチャンネル全体で共有されます。`thread.inheritParent` は、親チャンネルのトランスクリプトを新しいスレッドにコピーします。`thread.initialHistoryLimit`（デフォルトは `20`）は、新しいスレッドセッションの開始時に取得する既存のスレッドメッセージ数を制限します。`0` は、スレッド履歴の取得を無効にします。

- Slack ネイティブストリーミングと、Slack アシスタント形式の「入力中...」スレッドステータスには、返信スレッドの配信先が必要です。トップレベルの DM はデフォルトでスレッド外のままであるため、スレッド形式のネイティブストリーム／ステータスプレビューを表示する代わりに、Slack の下書き投稿・編集プレビューを通じて引き続きストリーミングできます。
- `typingReaction` は、返信の実行中に受信 Slack メッセージへ一時的なリアクションを追加し、完了時に削除します。`"hourglass_flowing_sand"` のような Slack 絵文字ショートコードを使用してください。
- `channels.slack.execApprovals`：Slack ネイティブの承認クライアント配信および実行承認者の認可。Discord と同じスキーマです：`enabled`（`true`/`false`/`"auto"`）、`approvers`（Slack ユーザー ID）、`agentFilter`、`sessionFilter`、`target`（`"dm"`、`"channel"`、または `"both"`）。Slack Plugin の承認者を解決できる場合、Plugin の承認は Slack からのリクエストにこのネイティブクライアントパスを使用できます。Slack ネイティブの Plugin 承認配信は、Slack 由来のセッションまたは Slack の配信先に対して `approvals.plugin` を介して有効にすることもできます。Plugin の承認では、実行承認者ではなく、`allowFrom` の Slack Plugin 承認者とデフォルトルーティングを使用します。

| アクショングループ | デフォルト | 注記                       |
| ------------------ | ---------- | -------------------------- |
| reactions          | 有効       | リアクションの追加と一覧表示 |
| messages           | 有効       | 読み取り／送信／編集／削除 |
| pins               | 有効       | ピン留め／解除／一覧表示   |
| memberInfo         | 有効       | メンバー情報               |
| emojiList          | 有効       | カスタム絵文字一覧         |

### Mattermost

Mattermost は、Discord、Slack、WhatsApp と同様に、独立した Plugin としてインストールします。

```bash
openclaw plugins install @openclaw/mattermost
```

バージョンを固定する前に、現在の dist-tag を [npmjs.com/package/@openclaw/mattermost](https://www.npmjs.com/package/@openclaw/mattermost) で確認してください。

```json5
{
  channels: {
    mattermost: {
      enabled: true,
      botToken: "mm-token",
      baseUrl: "https://chat.example.com",
      dmPolicy: "pairing",
      chatmode: "oncall", // oncall | onmessage | onchar
      oncharPrefixes: [">", "!"],
      groups: {
        "*": { requireMention: true },
        "team-channel-id": { requireMention: false },
      },
      commands: {
        native: true, // オプトイン
        nativeSkills: true,
        callbackPath: "/api/channels/mattermost/command",
        // リバースプロキシ／公開デプロイ向けのオプションの明示的 URL
        callbackUrl: "https://gateway.example.com/api/channels/mattermost/command",
      },
      textChunkLimit: 4000,
      streaming: { chunkMode: "length" },
    },
  },
}
```

チャットモード：`oncall`（@メンションに応答、デフォルト）、`onmessage`（すべてのメッセージ）、`onchar`（トリガープレフィックスで始まるメッセージ）。

Mattermost のネイティブコマンドを有効にする場合：

- `commands.callbackPath` は完全な URL ではなく、パス（例：`/api/channels/mattermost/command`）である必要があります。
- `commands.callbackUrl` は OpenClaw Gateway エンドポイントに解決され、Mattermost サーバーから到達可能である必要があります。
- ネイティブスラッシュコールバックは、スラッシュコマンドの登録時に
  Mattermost から返されるコマンドごとのトークンで認証されます。登録に失敗した場合や、
  有効化されたコマンドがない場合、OpenClaw は
  `Unauthorized: invalid command token.` でコールバックを拒否します。
- プライベート／tailnet／内部コールバックホストの場合、Mattermost では
  `ServiceSettings.AllowedUntrustedInternalConnections` にコールバックのホスト／ドメインを含める必要があることがあります。
  完全な URL ではなく、ホスト／ドメイン値を使用してください。
- `channels.mattermost.configWrites`：Mattermost から開始される設定の書き込みを許可または拒否します。
- `channels.mattermost.requireMention`：チャンネル内で返信する前に `@mention` を必須にします。
- `channels.mattermost.groups.<channelId>.requireMention`：チャンネル単位のメンションゲート上書き（デフォルトには `"*"`）。
- オプションの `channels.mattermost.defaultAccount` は、設定済みのアカウント ID と一致する場合、デフォルトのアカウント選択を上書きします。

### Signal

```json5
{
  channels: {
    signal: {
      enabled: true,
      account: "+15555550123", // オプションのアカウントバインディング
      dmPolicy: "pairing",
      allowFrom: ["+15551234567", "uuid:123e4567-e89b-12d3-a456-426614174000"],
      configWrites: true,
      reactionNotifications: "own", // off | own | all | allowlist
      reactionAllowlist: ["+15551234567", "uuid:123e4567-e89b-12d3-a456-426614174000"],
      historyLimit: 50,
    },
  },
}
```

**リアクション通知モード：** `off`、`own`（デフォルト）、`all`、`allowlist`（`reactionAllowlist` から）。

- `channels.signal.account`：チャンネルの起動を特定の Signal アカウント ID に固定します。
- `channels.signal.configWrites`：Signal から開始される設定の書き込みを許可または拒否します。
- オプションの `channels.signal.defaultAccount` は、設定済みのアカウント ID と一致する場合、デフォルトのアカウント選択を上書きします。

### iMessage

OpenClaw は `imsg rpc`（stdio 経由の JSON-RPC）を起動します。デーモンやポートは不要です。ホストで Messages データベースと Automation の権限を付与できる場合、新しい OpenClaw iMessage セットアップではこれが推奨される方法です。

BlueBubbles のサポートは削除されました。現在の OpenClaw では、`channels.bluebubbles` はサポート対象のランタイム設定サーフェスではありません。古い設定を `channels.imessage` に移行してください。概要については[BlueBubbles の削除と imsg iMessage パス](/ja-JP/announcements/bluebubbles-imessage)、完全な変換表については[BlueBubbles からの移行](/ja-JP/channels/imessage-from-bluebubbles)を参照してください。

Gateway が Messages にサインインしている Mac 上で動作していない場合は、`channels.imessage.enabled=true` を維持し、`channels.imessage.cliPath` を、その Mac 上で `imsg "$@"` を実行する SSH ラッパーに設定します。デフォルトのローカル `imsg` パスは macOS 専用です。

本番送信で SSH ラッパーに依存する前に、そのラッパーを実際に経由する送信 `imsg send` を検証してください。一部の macOS TCC 状態では、メッセージのオートメーション権限が `/usr/libexec/sshd-keygen-wrapper` に割り当てられるため、読み取りとプローブは動作しても、送信は AppleEvents `-1743` で失敗することがあります。[iMessage](/ja-JP/channels/imessage) の SSH ラッパーのトラブルシューティングセクションを参照してください。

```json5
{
  channels: {
    imessage: {
      enabled: true,
      cliPath: "imsg",
      dbPath: "~/Library/Messages/chat.db",
      remoteHost: "user@gateway-host",
      dmPolicy: "pairing",
      allowFrom: ["+15555550123", "user@example.com", "chat_id:123"],
      historyLimit: 50,
      includeAttachments: false,
      attachmentRoots: ["/Users/*/Library/Messages/Attachments"],
      remoteAttachmentRoots: ["/Users/*/Library/Messages/Attachments"],
      mediaMaxMb: 16,
      service: "auto",
      sendTransport: "auto",
      region: "US",
      actions: {
        reactions: true,
        edit: true,
        unsend: true,
        reply: true,
        sendWithEffect: true,
        sendAttachment: true,
      },
    },
  },
}
```

- 任意の `channels.imessage.defaultAccount` は、設定済みのアカウント ID と一致する場合にデフォルトのアカウント選択を上書きします。
- Messages DB へのフルディスクアクセスが必要です。
- `chat_id:<id>` ターゲットを優先してください。チャットを一覧表示するには `imsg chats --limit 20` を使用します。
- `cliPath` は SSH ラッパーを指定できます。SCP による添付ファイル取得には `remoteHost`（`host` または `user@host`）を設定します。
- `attachmentRoots` と `remoteAttachmentRoots` は受信添付ファイルのパスを制限します（デフォルト: `/Users/*/Library/Messages/Attachments`）。
- SCP は厳密なホストキー検証を使用するため、リレーホストのキーが `~/.ssh/known_hosts` にすでに存在することを確認してください。
- `channels.imessage.configWrites`: iMessage から開始された設定書き込みを許可または拒否します。
- `channels.imessage.sendTransport`: 通常の送信返信に使用する優先 `imsg` RPC 送信トランスポートです。`auto`（デフォルト）は、実行中であれば既存チャットに IMCore ブリッジを使用し、その後 AppleScript にフォールバックします。`bridge` はプライベート API による配信を必須とし、`applescript` は公開の Messages オートメーション経路を強制します。
- `channels.imessage.actions.*`: `imsg status` / `openclaw channels status --probe` によっても制御されるプライベート API アクションを有効にします。
- `channels.imessage.includeAttachments` はデフォルトで無効です。エージェントターンで受信メディアを利用するには、`true` に設定してください。
- ブリッジまたは Gateway の再起動後の受信復旧は自動です（GUID 重複排除と古いバックログに対する経過時間制限）。既存の `channels.imessage.catchup.enabled: true` 設定は非推奨の互換性プロファイルとして引き続き尊重されます。`catchup` はデフォルトで無効です。
- `channels.imessage.groups`: グループレジストリとグループごとの設定です。`groupPolicy: "allowlist"` を使用する場合、グループメッセージがレジストリゲートを通過できるように、明示的な `chat_id` キーまたは `"*"` ワイルドカードエントリのいずれかを設定してください。
- `type: "acp"` を持つトップレベルの `bindings[]` エントリは、iMessage の会話を永続 ACP セッションにバインドできます。`match.peer.id` には、正規化されたハンドルまたは明示的なチャットターゲット（`chat_id:*`、`chat_guid:*`、`chat_identifier:*`）を使用してください。共通フィールドのセマンティクス: [ACP エージェント](/ja-JP/tools/acp-agents#persistent-channel-bindings)。

<Accordion title="iMessage SSH ラッパーの例">

```bash
#!/usr/bin/env bash
exec ssh -T gateway-host imsg "$@"
```

</Accordion>

### Matrix

Matrix は Plugin によって提供され、`channels.matrix` 配下で設定します。

```json5
{
  channels: {
    matrix: {
      enabled: true,
      homeserver: "https://matrix.example.org",
      accessToken: "syt_bot_xxx",
      proxy: "http://127.0.0.1:7890",
      encryption: true,
      initialSyncLimit: 20,
      defaultAccount: "ops",
      accounts: {
        ops: {
          name: "Ops",
          userId: "@ops:example.org",
          accessToken: "syt_ops_xxx",
        },
        alerts: {
          userId: "@alerts:example.org",
          password: "secret",
          proxy: "http://127.0.0.1:7891",
        },
      },
    },
  },
}
```

- トークン認証では `accessToken` を使用し、パスワード認証では `userId` + `password` を使用します。
- `channels.matrix.proxy` は Matrix の HTTP トラフィックを明示的な HTTP(S) プロキシ経由でルーティングします。名前付きアカウントでは `channels.matrix.accounts.<id>.proxy` によって上書きできます。
- `channels.matrix.network.dangerouslyAllowPrivateNetwork` はプライベートまたは内部のホームサーバーを許可します。`proxy` とこのネットワークのオプトインは独立した制御です。
- `channels.matrix.defaultAccount` は、複数アカウント構成で優先するアカウントを選択します。
- `channels.matrix.autoJoin` のデフォルトは `"off"` です。そのため、`autoJoinAllowlist` または `autoJoin: "always"` を使用して `autoJoin: "allowlist"` を設定するまで、招待されたルームと新しい DM 形式の招待は無視されます。
- `channels.matrix.execApprovals`: Matrix ネイティブの実行承認配信と承認者の認可です。
  - `enabled`: `true`、`false`、または `"auto"`（デフォルト）。自動モードでは、`approvers` または `commands.ownerAllowFrom` から承認者を解決できる場合に実行承認が有効になります。
  - `approvers`: 実行リクエストの承認を許可された Matrix ユーザー ID（例: `@owner:example.org`）。
  - `agentFilter`: 任意のエージェント ID 許可リストです。すべてのエージェントの承認を転送する場合は省略します。
  - `sessionFilter`: 任意のセッションキーパターン（部分文字列または正規表現）。
  - `target`: 承認プロンプトの送信先です。`"dm"`（デフォルト）、`"channel"`（送信元ルーム）、または `"both"`。
  - アカウントごとの上書き: `channels.matrix.accounts.<id>.execApprovals`。
- `channels.matrix.dm.sessionScope` は Matrix の DM をセッションにまとめる方法を制御します。`per-user`（デフォルト）はルーティングされたピア単位で共有し、`per-room` は各 DM ルームを分離します。
- Matrix のステータスプローブとライブディレクトリ検索は、ランタイムトラフィックと同じプロキシポリシーを使用します。
- Matrix の完全な設定、ターゲット指定ルール、セットアップ例については [Matrix](/ja-JP/channels/matrix) を参照してください。

### Microsoft Teams

Microsoft Teams は Plugin によって提供され、`channels.msteams` 配下で設定します。

```json5
{
  channels: {
    msteams: {
      enabled: true,
      configWrites: true,
      // appId、appPassword、tenantId、webhook、チーム/チャネルポリシー:
      // /channels/msteams を参照
    },
  },
}
```

- ここで扱う主要なキーパス: `channels.msteams`、`channels.msteams.configWrites`。
- Teams の完全な設定（認証情報、Webhook、DM/グループポリシー、チームごと/チャネルごとの上書き）については [Microsoft Teams](/ja-JP/channels/msteams) を参照してください。

### IRC

IRC は Plugin によって提供され、`channels.irc` 配下で設定します。

```json5
{
  channels: {
    irc: {
      enabled: true,
      dmPolicy: "pairing",
      configWrites: true,
      nickserv: {
        enabled: true,
        service: "NickServ",
        password: "${IRC_NICKSERV_PASSWORD}",
        register: false,
        registerEmail: "bot@example.com",
      },
    },
  },
}
```

- ここで扱う主要なキーパス: `channels.irc`、`channels.irc.dmPolicy`、`channels.irc.configWrites`、`channels.irc.nickserv.*`。
- 任意の `channels.irc.defaultAccount` は、設定済みのアカウント ID と一致する場合にデフォルトのアカウント選択を上書きします。
- IRC チャネルの完全な設定（ホスト/ポート/TLS/チャネル/許可リスト/メンションゲート）については [IRC](/ja-JP/channels/irc) を参照してください。

### 複数アカウント（すべてのチャネル）

チャネルごとに複数のアカウントを実行します（各アカウントは独自の `accountId` を持ちます）。

```json5
{
  channels: {
    telegram: {
      accounts: {
        default: {
          name: "Primary bot",
          botToken: "123456:ABC...",
        },
        alerts: {
          name: "Alerts bot",
          botToken: "987654:XYZ...",
        },
      },
    },
  },
}
```

- `accountId` が省略された場合、`default` が使用されます（CLI + ルーティング）。
- 環境変数のトークンは **デフォルト** アカウントにのみ適用されます。
- 基本チャネル設定は、アカウントごとに上書きしない限り、すべてのアカウントに適用されます。
- 各アカウントを異なるエージェントにルーティングするには `bindings[].match.accountId` を使用します。
- 単一アカウントのトップレベルチャネル設定を使用している状態で、`openclaw channels add`（またはチャネルのオンボーディング）によってデフォルト以外のアカウントを追加すると、OpenClaw は最初にアカウントスコープのトップレベル単一アカウント値をチャネルアカウントマップへ昇格させ、元のアカウントが引き続き動作するようにします。ほとんどのチャネルではそれらを `channels.<channel>.accounts.default` に移動します。Matrix では、既存の一致する名前付きまたはデフォルトのターゲットを代わりに保持できます。
- 既存のチャネルのみのバインディング（`accountId` なし）は引き続きデフォルトアカウントに一致します。アカウントスコープのバインディングは任意のままです。
- `openclaw doctor --fix` は、アカウントスコープのトップレベル単一アカウント値を、そのチャネルで選択された昇格先アカウントへ移動することで、混在した形式も修復します。ほとんどのチャネルでは `accounts.default` を使用します。Matrix では、既存の一致する名前付きまたはデフォルトのターゲットを代わりに保持できます。

### その他の Plugin チャネル

多くの Plugin チャネルは `channels.<id>` として設定され、それぞれ専用のチャネルページに記載されています（例: Feishu、LINE、Nextcloud Talk、Nostr、QQ Bot、Synology Chat、Twitch、Zalo）。
チャネルの完全な一覧については [チャネル](/ja-JP/channels) を参照してください。

### グループチャットのメンションゲート

グループメッセージでは、デフォルトで **メンションが必須** です（メタデータのメンションまたは安全な正規表現パターン）。WhatsApp、Telegram、Discord、Google Chat、iMessage のグループチャットに適用されます。

表示される返信は個別に制御されます。通常のグループ、チャネル、内部 WebChat の直接リクエストでは、デフォルトで最終回答が自動配信されます。つまり、最終的なアシスタントテキストは従来の表示返信経路を通じて投稿されます。モデルが作成した送信元への返信を、エージェントが `message(action=send)` を呼び出した後にのみ投稿する場合は、`messages.visibleReplies: "message_tool"` または `messages.groupChat.visibleReplies: "message_tool"` をオプトインしてください。オプトインしたツール専用モードで、モデルがメッセージツールを呼び出さずに実質的な最終回答を返した場合、その最終テキストは非公開のままとなり、Gateway の詳細ログには抑制されたペイロードのメタデータが記録され、OpenClaw は同じ返信を `message(action=send)` 経由で配信するようモデルに求める復旧リトライを 1 回キューに追加します。

ツール専用ポリシーは、アシスタントの送信元への返信と汎用ツールメディアを管理します。認可されたコマンド応答、永続的な完了通知、所有ハーネスが明示的にホスト所有として分類したプロバイダーネイティブのアーティファクトなど、ランタイム所有のターミナル出力は抑制しません。ホスト所有のアーティファクトは通常のチャネルディスパッチ経路で配信され、送信 `sendPolicy` の拒否にも引き続き従います。周辺的な `room_event` ターンは、ランタイム出力がホスト所有とマークされていても、明示的なコマンドでない限り何も送信しません。

ツール専用の表示返信には、ツールを確実に呼び出すモデル/ランタイムが必要です。また、GPT-5.6 Sol などの最新世代モデルを使用する共有の周辺的なルームに推奨されます。一部の性能が低いモデルは最終テキストで回答できますが、送信元に表示する出力を `message(action=send)` で送信する必要があることを理解できない場合があります。OpenClaw は、最終回答が実質的であり、送信元ターンがルームイベントではなく、送信ポリシーが配信を拒否しておらず、送信元への返信がまだ送信されていない場合に限り、一般的な未配信の最終回答をデフォルトで復旧します。復旧は 1 回のリトライに制限されます。合成リトライプロンプトの永続化を抑制し、そのリトライを収集バッチの対象外にするため、無関係なキュー済みプロンプトと統合されることはありません。リトライでも未配信になるか、キューに追加できない場合、OpenClaw は「返信を生成しましたが、このチャットに配信できませんでした。もう一度お試しください。」のようなサニタイズ済み診断のみを配信します。元の非公開の最終テキストが送信元への自動配信対象としてマークされることはありません。返信が繰り返し未配信になるモデルでは、最終アシスタントターンを表示返信経路にするために `"automatic"` を使用するか、より強力なツール呼び出しモデルへ切り替えるか、Gateway の詳細ログで抑制されたペイロードの概要を確認するか、すべてのグループ/チャネルリクエストで表示される最終返信を使用するよう `messages.groupChat.visibleReplies: "automatic"` を設定してください。

アクティブなツールポリシーでメッセージツールを利用できない場合、OpenClaw は応答を黙って抑制するのではなく、自動的に表示される返信へフォールバックします。`openclaw doctor` は、この不一致について警告します。

このルールは、通常のエージェント最終テキストに適用されます。Plugin が所有する会話バインディングでは、バインドされたスレッドで引き受けたターンについて、所有する Plugin が返した返信を表示応答として使用します。Plugin は、それらのバインディング返信のために `message(action=send)` を呼び出す必要はありません。

**トラブルシューティング：グループでの @メンションにより入力中と表示された後、エラーなしで応答がない**

症状：グループ／チャンネルでの @メンションに入力中インジケーターが表示され、Gateway ログに `dispatch complete (queuedFinal=false, replies=0)` と記録されますが、ルームにはメッセージが届きません。同じエージェントへの DM には通常どおり返信があります。

原因：グループ／チャンネルの表示返信モードが `"message_tool"` に解決されるため、OpenClaw はターンを実行しますが、エージェントが `message(action=send)` を呼び出さない限り、アシスタントの最終テキストを抑制します。このモードには `NO_REPLY` の契約はありません。メッセージツールが呼び出されなければ、元の最終テキストは非公開になります。実質的な内容を含むソースターンについて、OpenClaw は保護された復旧再試行を 1 回行うようになりました。短いメモ、明示的な無応答、ルームイベント、送信ポリシーにより拒否されたターン、すでに配信済みのターンは再試行されません。通常のグループおよびチャンネルのターンはデフォルトで `"automatic"` になるため、この症状が発生するのは、`messages.groupChat.visibleReplies`（またはグローバルな `messages.visibleReplies`）が明示的に `"message_tool"` に設定されている場合だけです。ハーネスの `defaultVisibleReplies` はここには適用されません。グループ／チャンネルのリゾルバーはこれを無視し、直接／ソースチャットにのみ影響します（Codex ハーネスは、この方法で直接チャットの最終テキストを抑制します）。

修正：ツール呼び出し能力がより高いモデルを選択するか、明示的な `"message_tool"` オーバーライドを削除して `"automatic"` のデフォルトへフォールバックするか、`messages.groupChat.visibleReplies: "automatic"` を設定して、すべてのグループ／チャンネルリクエストで表示返信を強制します。実質的な内容を含むにもかかわらず配信されなかった最終テキストが、無応答のまま成功として終了することはなくなります。`message(action=send)` による 1 回の再試行で復旧するか、サニタイズされた配信失敗の診断が表示されます。ファイルの保存後、Gateway は `messages` 設定をホットリロードします。デプロイ環境でファイル監視または設定のリロードが無効になっている場合にのみ、Gateway を再起動してください。

**メンションの種類：**

- **メタデータメンション**：プラットフォームネイティブの @メンションです。WhatsApp のセルフチャットモードでは無視されます。
- **テキストパターン**：`agents.entries.*.groupChat.mentionPatterns` 内の安全な正規表現パターンです。無効なパターンと、安全でない入れ子状の繰り返しは無視されます。
- メンションによるゲーティングは、検出が可能な場合（ネイティブメンション、または少なくとも 1 つのパターンがある場合）にのみ適用されます。

```json5
{
  messages: {
    visibleReplies: "automatic", // 直接／ソースチャットで従来の自動最終返信を強制
    groupChat: {
      historyLimit: 50,
      unmentionedInbound: "room_event", // 常時有効なメンションなしのルーム会話を静かなコンテキストにする
      visibleReplies: "message_tool", // オプトイン。ルームに表示される返信には message(action=send) を要求
    },
  },
  agents: {
    list: [{ id: "main", groupChat: { mentionPatterns: ["@openclaw", "openclaw"] } }],
  },
}
```

`messages.groupChat.historyLimit` はグローバルデフォルトを設定します。チャンネルでは `channels.<channel>.historyLimit`（またはアカウント単位の設定）でオーバーライドできます。無効にするには `0` を設定します。

`messages.groupChat.unmentionedInbound: "room_event"` は、サポートされているチャンネルで、常時有効なグループ／チャンネルのメンションなしメッセージを静かなルームコンテキストとして送信します。メンションされたメッセージ、コマンド、ダイレクトメッセージは、引き続きユーザーリクエストとして扱われます。Discord、Slack、Telegram の完全な例については、[アンビエントルームイベント](/ja-JP/channels/ambient-room-events)を参照してください。

`messages.visibleReplies` はグローバルなソースイベントのデフォルトであり、`messages.groupChat.visibleReplies` はグループ／チャンネルのソースイベントについてこれをオーバーライドします。`messages.visibleReplies` が未設定の場合、直接／ソースチャットでは選択されたランタイムまたはハーネスのデフォルトが使用されますが、内部 WebChat の直接ターンでは、Pi／Codex のプロンプトと同等の動作にするため、最終出力が自動配信されます。表示出力に意図的に `message(action=send)` を必須とするには、`messages.visibleReplies: "message_tool"` を設定します。イベントを処理するかどうかは、引き続きチャンネルの許可リストとメンションによるゲーティングによって決まります。

#### DM 履歴の上限

```json5
{
  channels: {
    telegram: {
      dmHistoryLimit: 30,
      dms: {
        "123456789": { historyLimit: 50 },
      },
    },
  },
}
```

解決順序：DM 単位のオーバーライド → プロバイダーのデフォルト → 上限なし（すべて保持）。

このリゾルバーは、セッションキーが標準の `provider:direct:<id>`（またはレガシーな `provider:dm:<id>`）形式に従うすべてのチャンネルについて、`channels.<provider>.dmHistoryLimit` と `channels.<provider>.dms.<id>.historyLimit` を読み取ります。そのため、固定されたチャンネル一覧だけでなく、バンドル済みチャンネルと Plugin チャンネルの両方で機能します。

#### セルフチャットモード

セルフチャットモードを有効にするには、自分の番号を `allowFrom` に含めます（ネイティブの @メンションを無視し、テキストパターンにのみ応答します）。

```json5
{
  channels: {
    whatsapp: {
      allowFrom: ["+15555550123"],
      groups: { "*": { requireMention: true } },
    },
  },
  agents: {
    list: [
      {
        id: "main",
        groupChat: { mentionPatterns: ["reisponde", "@openclaw"] },
      },
    ],
  },
}
```

### コマンド（チャットコマンドの処理）

```json5
{
  commands: {
    native: "auto", // サポートされている場合にネイティブコマンドを登録
    nativeSkills: "auto", // サポートされている場合にネイティブ Skills コマンドを登録
    text: true, // チャットメッセージ内の /commands を解析
    bash: false, // ! を許可（エイリアス：/bash）
    bashForegroundMs: 2000,
    config: false, // /config を許可
    mcp: false, // /mcp を許可
    plugins: false, // /plugins を許可
    debug: false, // /debug を許可
    restart: true, // /restart と外部からの SIGUSR1 再起動リクエストを許可
    ownerAllowFrom: ["discord:123456789012345678"],
    ownerDisplay: "raw", // raw | hash
    ownerDisplaySecret: "${OWNER_ID_HASH_SECRET}",
    allowFrom: {
      "*": ["user1"],
      discord: ["user:123"],
    },
    useAccessGroups: true,
  },
}
```

<Accordion title="コマンドの詳細">

- このブロックではコマンドのサーフェスを設定します。現在の組み込みおよびバンドル済みコマンドのカタログについては、[スラッシュコマンド](/ja-JP/tools/slash-commands)を参照してください。
- このページは完全なコマンドカタログではなく、**設定キーのリファレンス**です。QQ Bot の `/bot-ping` `/bot-help` `/bot-logs`、LINE の `/card`、デバイスペアリングの `/pair`、メモリの `/dreaming`、電話制御の `/phone`、Talk の `/voice` など、チャンネル／Plugin が所有するコマンドについては、それぞれのチャンネル／Plugin ページおよび[スラッシュコマンド](/ja-JP/tools/slash-commands)に記載されています。
- テキストコマンドは、先頭に `/` を付けた**単独の**メッセージでなければなりません。
- `native: "auto"` は Discord／Telegram でネイティブコマンドを有効にし、Slack では無効のままにします。
- `nativeSkills: "auto"` は Discord／Telegram でネイティブ Skills コマンドを有効にし、Slack では無効のままにします。
- チャンネル単位でオーバーライドするには、`channels.discord.commands.native`（真偽値または `"auto"`）を使用します。Discord では、`false` により、起動時のネイティブコマンドの登録とクリーンアップが省略されます。
- チャンネル単位のネイティブ Skills 登録は、`channels.<provider>.commands.nativeSkills` でオーバーライドします。
- `channels.telegram.customCommands` は Telegram ボットのメニュー項目を追加します。
- `bash: true` は、ホストシェル用の `! <cmd>` を有効にします。`tools.elevated.enabled` が必要であり、送信者が `tools.elevated.allowFrom.<channel>` に含まれている必要があります。
- `config: true` は `/config` を有効にします（`openclaw.json` を読み書きします）。Gateway の `chat.send` クライアントでは、永続的な `/config set|unset` の書き込みには `operator.admin` も必要です。読み取り専用の `/config show` は、通常の書き込みスコープを持つオペレータークライアントでも引き続き利用できます。
- `mcp: true` は、`mcp.servers` 配下にある OpenClaw 管理の MCP サーバー設定用の `/mcp` を有効にします。
- `plugins: true` は、Plugin の検出、インストール、有効化／無効化を制御する `/plugins` を有効にします。
- `channels.<provider>.configWrites` は、チャンネル単位の設定変更を制限します（デフォルト：true）。
- 複数アカウント対応チャンネルでは、`channels.<provider>.accounts.<id>.configWrites` により、そのアカウントを対象とする書き込み（たとえば `/allowlist --config --account <id>` や `/config set channels.<provider>.accounts.<id>...`）も制限されます。
- `restart: false` は、`/restart` および外部からの `SIGUSR1` 再起動リクエストを無効にします。デフォルト：`true`。
- `ownerAllowFrom` は、所有者限定コマンドおよび所有者によって制限されるチャンネル操作のための明示的な所有者許可リストです。`allowFrom` とは別のものです。
- `ownerDisplay: "hash"` は、システムプロンプト内の所有者 ID をハッシュ化します。ハッシュ化を制御するには `ownerDisplaySecret` を設定します。
- `allowFrom` はプロバイダー単位です。設定されている場合、これが**唯一の**認可ソースになります（チャンネルの許可リスト／ペアリングおよび `useAccessGroups` は無視されます）。
- `useAccessGroups: false` は、`allowFrom` が設定されていない場合に、コマンドがアクセスグループポリシーを回避できるようにします。
- コマンドドキュメントの一覧：
  - 組み込みおよびバンドル済みカタログ：[スラッシュコマンド](/ja-JP/tools/slash-commands)
  - チャンネル固有のコマンドサーフェス：[チャンネル](/ja-JP/channels)
  - QQ Bot コマンド：[QQ Bot](/ja-JP/channels/qqbot)
  - ペアリングコマンド：[ペアリング](/ja-JP/channels/pairing)
  - LINE カードコマンド：[LINE](/ja-JP/channels/line)
  - メモリ Dreaming：[Dreaming](/ja-JP/concepts/dreaming)

</Accordion>

---

## 関連項目

- [設定リファレンス](/ja-JP/gateway/configuration-reference) — トップレベルキー
- [設定 — エージェント](/ja-JP/gateway/config-agents)
- [チャンネルの概要](/ja-JP/channels)
