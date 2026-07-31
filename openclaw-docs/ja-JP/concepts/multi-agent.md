---
read_when: You want multiple agents with separate workspaces, auth, and sessions in one Gateway process.
sidebarTitle: Multi-agent routing
status: active
summary: マルチエージェントルーティング：エージェントの境界、チャネルアカウント、バインディング
title: マルチエージェントルーティング
x-i18n:
    generated_at: "2026-07-26T09:38:42Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 46df162388205e46d5a4ea3567c8c8f7016117d2ecafe1184a35b4c95798fd80
    source_path: concepts/multi-agent.md
    workflow: 16
---

1 つの Gateway プロセスで複数の_分離された_エージェントを実行します。各エージェントは独自のワークスペース、状態ディレクトリ（`agentDir`）、SQLite ベースのセッション履歴を持ち、さらに複数のチャンネルアカウント（例: 2 つの WhatsApp 番号）を利用できます。受信メッセージは **バインディング**を通じて適切なエージェントへルーティングされます。

**エージェント**とは、ワークスペースファイル、認証プロファイル、モデルレジストリ、セッションストアを含む、ペルソナごとの完全なスコープです。**バインディング**は、チャンネルアカウント（Slack ワークスペース、WhatsApp 番号など）をいずれかのエージェントに対応付けます。

## 1 つのエージェントとは

各エージェントは以下を個別に持ちます。

- **ワークスペース**: ファイル、`AGENTS.md`/`SOUL.md`/`USER.md`、ローカルノート、ペルソナルール。
- **状態ディレクトリ**（`agentDir`）: 認証プロファイル、モデルレジストリ、エージェントごとの設定。
- **セッションストア**: `~/.openclaw/agents/<agentId>/agent/openclaw-agent.sqlite` 内のチャット履歴とルーティング状態。

認証プロファイルはエージェントごとに管理され、以下から読み込まれます。

```text
~/.openclaw/agents/<agentId>/agent/auth-profiles.json
```

<Note>
`sessions_history` は、セッションをまたいで記憶を呼び出すための、より安全な経路です。生のトランスクリプトをそのまま返すのではなく、範囲が制限され、秘匿化されたビューを返します。思考ブロックの署名、ツール結果ペイロードの詳細、`<relevant-memories>` の足場、ツール呼び出しの XML タグ（`<tool_call>`、`<function_call>`、およびそれらの複数形やダウングレード形式）、MiniMax のツール呼び出し XML を除去したうえで、出力を切り詰め、バイトサイズで上限を設定します。
</Note>

<Warning>
エージェント間で `agentDir` を再利用しないでください。認証状態やセッション状態の衝突を引き起こします。セカンダリエージェントのローカル OAuth 資格情報が期限切れになった場合や更新に失敗した場合、OpenClaw は同じプロファイル ID を持つデフォルト／メインエージェントの資格情報を読み取り、最も新しいトークンを採用します。このとき、更新トークンはセカンダリエージェントのストアへコピーされません。完全に独立した OAuth アカウントを使用する場合は、そのエージェントからサインインしてください。資格情報を手動でコピーする場合は、移植可能な静的 `api_key` または `token` プロファイルだけをコピーしてください。OAuth の更新情報はデフォルトでは移植できません（`copyToAgents` により、プロファイルごとに明示的に有効化できます）。
</Warning>

Skills は各エージェントのワークスペースと `~/.openclaw/skills` などの共有ルートから読み込まれ、その後、有効なエージェントの Skills 許可リストによって絞り込まれます。共有ベースラインには `agents.defaults.skills` を使用し、エージェントごとの置き換えには `agents.entries.*.skills` を使用します（明示的なエントリはデフォルトを置き換え、マージはしません）。[Skills: エージェントごとと共有の違い](/ja-JP/tools/skills#per-agent-vs-shared-skills)および[Skills: エージェントの許可リスト](/ja-JP/tools/skills#agent-allowlists)を参照してください。

Plugin が所有するストレージは、その Plugin の設定に従います。2 番目のエージェントを追加しても、
すべてのグローバル Plugin ストアが自動的に分割されるわけではありません。たとえば、ペルソナ間で
コンパイル済みの Wiki ナレッジを共有してはならない場合は、
[エージェントごとの Memory Wiki ボールト](/ja-JP/concepts/multi-agent#per-agent-memory-wiki-vaults)
を設定してください。

<Note>
**ワークスペースに関する注意:** 各エージェントのワークスペースは**デフォルトの cwd** であり、厳格なサンドボックスではありません。相対パスはワークスペース内で解決されますが、サンドボックス化が有効でない限り、絶対パスからホスト上の他の場所にアクセスできます。[サンドボックス化](/ja-JP/gateway/sandboxing)を参照してください。
</Note>

## パス

| 対象                             | デフォルト                                                                                | 上書き                                                                                    |
| -------------------------------- | -------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------- |
| 設定                           | `~/.openclaw/openclaw.json`                                                            | `OPENCLAW_CONFIG_PATH`                                                                      |
| 状態ディレクトリ                        | `~/.openclaw`                                                                          | `OPENCLAW_STATE_DIR`                                                                        |
| デフォルトエージェントのワークスペース        | `~/.openclaw/workspace`（`OPENCLAW_PROFILE` が設定されている場合は `workspace-<profile>`）      | `agents.entries.*.workspace`、次に `agents.defaults.workspace`、または `OPENCLAW_WORKSPACE_DIR` |
| その他のエージェントのワークスペース          | `<stateDir>/workspace-<agentId>`（設定されている場合は `<agents.defaults.workspace>/<agentId>`） | `agents.entries.*.workspace`                                                                |
| エージェントディレクトリ                        | `~/.openclaw/agents/<agentId>/agent`                                                   | `agents.entries.*.agentDir`                                                                 |
| セッションとトランスクリプト         | `~/.openclaw/agents/<agentId>/agent/openclaw-agent.sqlite`                             | —                                                                                           |
| レガシー／アーカイブ済みセッション成果物 | `~/.openclaw/agents/<agentId>/sessions`                                                | —                                                                                           |

### 単一エージェントモード（デフォルト）

何も設定しない場合、OpenClaw は 1 つのエージェントを実行します。

- `agentId` のデフォルトは `main` です。
- セッションのキーは `agent:main:<mainKey>` です（デフォルトの `mainKey` は `main`）。
- ワークスペースのデフォルトは `~/.openclaw/workspace` です（`OPENCLAW_PROFILE` が `default` 以外に設定されている場合は `workspace-<profile>`）。
- 状態のデフォルトは `~/.openclaw/agents/main/agent` です。

## エージェントヘルパー

新しい分離エージェントを追加します。

```bash
openclaw agents add work
```

フラグ: `--workspace <dir>`、`--model <id>`、`--agent-dir <dir>`、`--bind <channel[:accountId]>`（繰り返し指定可能）、`--non-interactive`（`--workspace` が必要）。

受信メッセージをルーティングするには `bindings` を追加し（ウィザードでも設定できます）、その後、以下で確認します。

```bash
openclaw agents list --bindings
```

## クイックスタート

<Steps>
  <Step title="各エージェントのワークスペースを作成する">
    ```bash
    openclaw agents add coding
    openclaw agents add social
    ```

    各エージェントには、`SOUL.md`、`AGENTS.md`、任意の `USER.md` を含む独自のワークスペースに加え、専用の `agentDir` と `~/.openclaw/agents/<agentId>` 配下のセッションストアが作成されます。

  </Step>
  <Step title="チャンネルアカウントを作成する">
    使用するチャンネルごとに、各エージェント用のアカウントを 1 つ作成します。

    - Discord: エージェントごとに 1 つのボットを作成し、Message Content Intent を有効にして、各トークンをコピーします。
    - Telegram: BotFather を使用してエージェントごとに 1 つのボットを作成し、各トークンをコピーします。
    - WhatsApp: アカウントごとに各電話番号をリンクします。

    ```bash
    openclaw channels login --channel whatsapp --account work
    ```

    チャンネルガイドを参照してください: [Discord](/ja-JP/channels/discord)、[Telegram](/ja-JP/channels/telegram)、[WhatsApp](/ja-JP/channels/whatsapp)。

  </Step>
  <Step title="エージェント、アカウント、バインディングを追加する">
    `agents.entries` にエージェント、`channels.<channel>.accounts` にチャンネルアカウントを追加し、`bindings` で接続します（以下に例があります）。
  </Step>
  <Step title="再起動して確認する">
    ```bash
    openclaw gateway restart
    openclaw agents list --bindings
    openclaw channels status --probe
    ```
  </Step>
</Steps>

## 複数のエージェント、複数のペルソナ

設定された各 `agentId` は、エージェントのコア状態について個別のペルソナ境界となります。

- チャンネルごとに異なるアカウント（`accountId` ごと）。
- 異なるパーソナリティ（エージェントごとの `AGENTS.md`/`SOUL.md`）。
- 認証とセッションを分離し、エージェント間アクセスは明示的な機能または Plugin 設定によってのみ有効化。

これにより、エージェントのコア状態を分離したまま、複数のユーザーが 1 つの Gateway を共有できます。

## エージェントごとの Memory Wiki ボールト

Memory Wiki はデフォルトで 1 つのグローバルボールトを使用します。サポートエージェントの
コンパイル済みナレッジをマーケティングエージェントのものと分離するには、
`plugins.entries.memory-wiki.config.vault.scope` を `agent` に設定します。

```json5
{
  plugins: {
    entries: {
      "memory-wiki": {
        enabled: true,
        config: {
          vault: {
            scope: "agent",
            path: "~/.openclaw/wiki",
          },
        },
      },
    },
  },
}
```

設定したパスは親ディレクトリです。OpenClaw は正規化された
エージェント ID を追加し、`~/.openclaw/wiki/support` や
`~/.openclaw/wiki/marketing` のようなパスを生成します。複数のエージェントが設定されている場合、
エージェントスコープの CLI および Gateway 操作には
エージェントの明示的な指定が必要です。ブリッジの
フィルタリング、移行、信頼境界の詳細については、[エージェントごとの Memory Wiki ボールト](/ja-JP/plugins/memory-wiki#per-agent-vaults)を参照してください。

## エージェント間の QMD メモリ検索

あるエージェントから別のエージェントの QMD セッショントランスクリプトを検索できるようにするには、`agents.entries.*.memory.search.qmd.extraCollections` 配下に追加のコレクションを設定します。すべてのエージェントで同じコレクションを共有する場合は、`memory.search.qmd.extraCollections` を使用します。

```json5
{
  agents: {
    defaults: {
      workspace: "~/workspaces/main",
    },
    entries: {
      main: {
        workspace: "~/workspaces/main",
        memory: {
          search: {
            qmd: {
              extraCollections: [{ path: "notes" }], // ワークスペース内で解決される -> "notes-main" という名前のコレクション
            },
          },
        },
      },
      family: { workspace: "~/workspaces/family" },
    },
  },
  memory: {
    backend: "qmd",
    search: {
      qmd: {
        extraCollections: [{ path: "~/agents/family/sessions", name: "family-sessions" }],
      },
    },
    qmd: { includeDefaultMemory: false },
  },
}
```

追加コレクションのパスはエージェント間で共有できますが、そのパスがエージェントのワークスペース外にある場合、`name` は明示的なままです。ワークスペース内のパスはエージェントスコープのままなので、各エージェントは独自のトランスクリプト検索セットを維持できます。

## 1 つの WhatsApp 番号を複数人で使用する（DM の分割）

`peer.kind: "direct"` で送信者の E.164（`+15551234567`）を照合することにより、**1 つ**の WhatsApp アカウント上で異なる WhatsApp DM を別々のエージェントへルーティングします。返信は引き続き同じ WhatsApp 番号から送信され、エージェントごとの送信者 ID はありません。

<Note>
ダイレクトチャットはデフォルトでエージェントのメインセッションキーに集約されるため、完全な分離にはユーザーごとに 1 つのエージェントが必要です。
</Note>

```json5
{
  agents: {
    list: [
      { id: "alex", workspace: "~/.openclaw/workspace-alex" },
      { id: "mia", workspace: "~/.openclaw/workspace-mia" },
    ],
  },
  bindings: [
    {
      agentId: "alex",
      match: { channel: "whatsapp", peer: { kind: "direct", id: "+15551230001" } },
    },
    {
      agentId: "mia",
      match: { channel: "whatsapp", peer: { kind: "direct", id: "+15551230002" } },
    },
  ],
  channels: {
    whatsapp: {
      dmPolicy: "allowlist",
      allowFrom: ["+15551230001", "+15551230002"],
    },
  },
}
```

DM のアクセス制御（ペアリング／許可リスト）はエージェントごとではなく、WhatsApp アカウントごとにグローバルです。共有グループでは、グループを 1 つのエージェントにバインドするか、[ブロードキャストグループ](/ja-JP/channels/broadcast-groups)を使用してください。

## ルーティングルール

バインディングは決定的であり、最も具体的なものが優先されます。階層の完全な順序（完全一致するピア、親ピア、ピアのワイルドカード、ギルド＋ロール、ギルド、チーム、アカウント、チャンネル、デフォルトエージェント）については、[チャンネルルーティング](/ja-JP/channels/channel-routing#routing-rules-how-an-agent-is-chosen)を参照してください。ここでは、特に重要ないくつかのルールを示します。

- 同じ階層内で複数のバインディングが一致する場合、設定内で最初に記述されたものが優先されます。
- バインディングに複数の照合フィールド（例: `peer` ＋ `guildId`）が設定されている場合、指定されたすべてのフィールドが一致する必要があります（`AND` のセマンティクス）。
- `accountId` を省略したバインディングは、すべてのアカウントではなく、デフォルトアカウントのみに一致します。チャンネル全体のフォールバックには `accountId: "*"` を使用し、特定の 1 アカウントには `accountId: "<name>"` を使用します。同じバインディングを明示的なアカウント ID とともに再度追加すると、重複を作成せず、既存のチャンネルのみのバインディングが更新されます。

## 複数のアカウント／電話番号

複数のアカウントをサポートするチャンネル（例: WhatsApp）は、各ログインの識別に `accountId` を使用します。各 `accountId` はそれぞれ独自のエージェントへルーティングされるため、1 台のサーバーでセッションを混在させることなく、複数の電話番号をホストできます。

`accountId` が省略された場合に使用するアカウントを選択するには、`channels.<channel>.defaultAccount` を設定します。未設定の場合、OpenClaw は `default` があればそれを使用し、なければ設定済みアカウント ID の先頭（並べ替え後）を使用します。

複数のアカウントをサポートするチャンネル: `discord`、`feishu`、`googlechat`、`imessage`、`irc`、`line`、`mattermost`、`matrix`、`nextcloud-talk`、`nostr`、`signal`、`slack`、`telegram`、`whatsapp`、`zalo`、`zalouser`。

## 概念

- `agentId`: 1 つの「頭脳」（ワークスペース、エージェントごとの認証、エージェントごとのセッションストア）。
- `accountId`: 1 つのチャンネルアカウントインスタンス（例: WhatsApp アカウント `personal` と `biz`）。
- `binding`: `(channel, accountId, peer)`、および必要に応じてギルド ID／チーム ID に基づいて、受信メッセージを `agentId` にルーティングします。
- ダイレクトチャットは `agent:<agentId>:<mainKey>` に集約されます（エージェントごとの「メイン」。`session.mainKey` を参照）。

## プラットフォーム別の例

<AccordionGroup>
  <Accordion title="エージェントごとの Discord ボット">
    各 Discord ボットアカウントは、一意の `accountId` に対応します。各アカウントをエージェントにバインドし、ボットごとに許可リストを維持します。

    ```json5
    {
      agents: {
        list: [
          { id: "main", workspace: "~/.openclaw/workspace-main" },
          { id: "coding", workspace: "~/.openclaw/workspace-coding" },
        ],
      },
      bindings: [
        { agentId: "main", match: { channel: "discord", accountId: "default" } },
        { agentId: "coding", match: { channel: "discord", accountId: "coding" } },
      ],
      channels: {
        discord: {
          groupPolicy: "allowlist",
          accounts: {
            default: {
              token: "DISCORD_BOT_TOKEN_MAIN",
              guilds: {
                "123456789012345678": {
                  channels: {
                    "222222222222222222": { allow: true, requireMention: false },
                  },
                },
              },
            },
            coding: {
              token: "DISCORD_BOT_TOKEN_CODING",
              guilds: {
                "123456789012345678": {
                  channels: {
                    "333333333333333333": { allow: true, requireMention: false },
                  },
                },
              },
            },
          },
        },
      },
    }
    ```

    - 各ボットをギルドに招待し、Message Content Intent を有効にします。
    - トークンは `channels.discord.accounts.<id>.token` に保存します（デフォルトアカウントでは `DISCORD_BOT_TOKEN` を使用できます）。

  </Accordion>
  <Accordion title="エージェントごとの Telegram ボット">
    ```json5
    {
      agents: {
        list: [
          { id: "main", workspace: "~/.openclaw/workspace-main" },
          { id: "alerts", workspace: "~/.openclaw/workspace-alerts" },
        ],
      },
      bindings: [
        { agentId: "main", match: { channel: "telegram", accountId: "default" } },
        { agentId: "alerts", match: { channel: "telegram", accountId: "alerts" } },
      ],
      channels: {
        telegram: {
          accounts: {
            default: {
              botToken: "123456:ABC...",
              dmPolicy: "pairing",
            },
            alerts: {
              botToken: "987654:XYZ...",
              dmPolicy: "allowlist",
              allowFrom: ["tg:123456789"],
            },
          },
        },
      },
    }
    ```

    - BotFather でエージェントごとに 1 つのボットを作成し、それぞれのトークンをコピーします。
    - トークンは `channels.telegram.accounts.<id>.botToken` に保存します（デフォルトアカウントでは `TELEGRAM_BOT_TOKEN` を使用できます）。
    - 同じ Telegram グループで複数のボットを使用する場合は、各ボットを招待し、応答させるボットをメンションします。
    - 各グループボットについて BotFather の Privacy Mode を無効にし（`/setprivacy` -> Disable）、Telegram に設定を適用させるため、ボットを削除してから再度追加します。
    - `channels.telegram.groups` でグループを許可するか、信頼できるグループへのデプロイに限って `groupPolicy: "open"` を使用します。
    - 送信者のユーザー ID は `groupAllowFrom` に指定します。グループ ID とスーパーグループ ID は `groupAllowFrom` ではなく `channels.telegram.groups` に指定します。
    - 各ボットがそれぞれ専用のエージェントにルーティングされるよう、`accountId` でバインドします。

  </Accordion>
  <Accordion title="エージェントごとの WhatsApp 番号">
    Gateway を起動する前に、各アカウントをリンクします。

    ```bash
    openclaw channels login --channel whatsapp --account personal
    openclaw channels login --channel whatsapp --account biz
    ```

    `~/.openclaw/openclaw.json`（JSON5）:

    ```js
    {
      agents: {
        list: [
          {
            id: "home",
            default: true,
            name: "Home",
            workspace: "~/.openclaw/workspace-home",
            agentDir: "~/.openclaw/agents/home/agent",
          },
          {
            id: "work",
            name: "Work",
            workspace: "~/.openclaw/workspace-work",
            agentDir: "~/.openclaw/agents/work/agent",
          },
        ],
      },

      // 決定的なルーティング: 最初に一致したものが優先されます（具体性の高いものを先にします）。
      bindings: [
        { agentId: "home", match: { channel: "whatsapp", accountId: "personal" } },
        { agentId: "work", match: { channel: "whatsapp", accountId: "biz" } },

        // ピアごとの任意のオーバーライド（例: 特定のグループを仕事用エージェントに送信）。
        {
          agentId: "work",
          match: {
            channel: "whatsapp",
            accountId: "personal",
            peer: { kind: "group", id: "1203630...@g.us" },
          },
        },
      ],

      // デフォルトでは無効: エージェント間メッセージングは明示的に有効化し、許可リストに追加する必要があります。
      tools: {
        agentToAgent: {
          enabled: false,
          allow: ["home", "work"],
        },
      },

      channels: {
        whatsapp: {
          accounts: {
            personal: {
              // 任意のオーバーライド。デフォルト: ~/.openclaw/credentials/whatsapp/personal
              // authDir: "~/.openclaw/credentials/whatsapp/personal",
            },
            biz: {
              // 任意のオーバーライド。デフォルト: ~/.openclaw/credentials/whatsapp/biz
              // authDir: "~/.openclaw/credentials/whatsapp/biz",
            },
          },
        },
      },
    }
    ```

  </Accordion>
</AccordionGroup>

## 一般的なパターン

<Tabs>
  <Tab title="日常用途の WhatsApp + 集中作業用の Telegram">
    チャンネルごとに分割します。WhatsApp は高速な日常用途のエージェントに、Telegram は Opus エージェントにルーティングします。

    ```json5
    {
      agents: {
        list: [
          {
            id: "chat",
            name: "Everyday",
            workspace: "~/.openclaw/workspace-chat",
            model: "anthropic/claude-sonnet-4-6",
          },
          {
            id: "opus",
            name: "Deep Work",
            workspace: "~/.openclaw/workspace-opus",
            model: "anthropic/claude-opus-4-6",
          },
        ],
      },
      bindings: [
        { agentId: "chat", match: { channel: "whatsapp", accountId: "*" } },
        { agentId: "opus", match: { channel: "telegram", accountId: "*" } },
      ],
    }
    ```

    これらの例では `accountId: "*"` を使用しているため、後からアカウントを追加してもバインディングは引き続き機能します。その他は chat に維持したまま、単一の DM／グループを Opus にルーティングするには、そのピア用の `match.peer` バインディングを追加します。ピアの一致は常にチャンネル全体のルールより優先されます。

  </Tab>
  <Tab title="同じチャンネルで 1 つのピアだけを Opus にルーティング">
    WhatsApp は高速なエージェントに維持しつつ、1 つの DM だけを Opus にルーティングします。

    ```json5
    {
      agents: {
        list: [
          {
            id: "chat",
            name: "Everyday",
            workspace: "~/.openclaw/workspace-chat",
            model: "anthropic/claude-sonnet-4-6",
          },
          {
            id: "opus",
            name: "Deep Work",
            workspace: "~/.openclaw/workspace-opus",
            model: "anthropic/claude-opus-4-6",
          },
        ],
      },
      bindings: [
        {
          agentId: "opus",
          match: { channel: "whatsapp", accountId: "*", peer: { kind: "direct", id: "+15551234567" } },
        },
        { agentId: "chat", match: { channel: "whatsapp", accountId: "*" } },
      ],
    }
    ```

    ピアバインディングは常に優先されるため、チャンネル全体のルールより上に配置します。

  </Tab>
  <Tab title="WhatsApp グループにバインドされた家族用エージェント">
    メンションによる制限と、より厳格なツールポリシーを設定した専用の家族用エージェントを、単一の WhatsApp グループにバインドします。

    ```json5
    {
      agents: {
        list: [
          {
            id: "family",
            name: "Family",
            workspace: "~/.openclaw/workspace-family",
            identity: { name: "Family Bot" },
            groupChat: {
              mentionPatterns: ["@family", "@familybot", "@Family Bot"],
            },
            sandbox: {
              mode: "all",
              scope: "agent",
            },
            tools: {
              allow: [
                "exec",
                "read",
                "sessions_list",
                "sessions_history",
                "sessions_send",
                "sessions_spawn",
                "session_status",
              ],
              deny: ["write", "edit", "apply_patch", "browser", "canvas", "nodes", "cron"],
            },
          },
        ],
      },
      bindings: [
        {
          agentId: "family",
          match: {
            channel: "whatsapp",
            peer: { kind: "group", id: "120363999999999999@g.us" },
          },
        },
      ],
    }
    ```

    ツールの許可／拒否リストは Skills ではなく、**ツール**を指定します。スキルがバイナリを実行する必要がある場合は、`exec` が許可され、そのバイナリがサンドボックス内に存在することを確認します。より厳格に制限するには、`agents.entries.*.groupChat.mentionPatterns` を設定し、そのチャンネルでグループの許可リストを有効に保ちます。

  </Tab>
</Tabs>

## エージェントごとのサンドボックスとツール設定

各エージェントには、個別のサンドボックスとツール制限を設定できます。

```js
{
  agents: {
    list: [
      {
        id: "personal",
        workspace: "~/.openclaw/workspace-personal",
        sandbox: {
          mode: "off",  // 個人用エージェントにはサンドボックスを使用しない
        },
        // ツール制限なし - すべてのツールを利用可能
      },
      {
        id: "family",
        workspace: "~/.openclaw/workspace-family",
        sandbox: {
          mode: "all",     // 常にサンドボックス化
          scope: "agent",  // エージェントごとに 1 つのコンテナ
          docker: {
            // コンテナ作成後に任意で 1 回だけ実行するセットアップ
            setupCommand: "apt-get update && apt-get install -y git curl",
          },
        },
        tools: {
          allow: ["read"],                    // read ツールのみ
          deny: ["exec", "write", "edit", "apply_patch"],    // その他を拒否
        },
      },
    ],
  },
}
```

<Note>
`setupCommand` は `sandbox.docker` 配下にあり、コンテナの作成時に 1 回だけ実行されます。解決されたスコープが `"shared"` の場合、エージェントごとの `sandbox.docker.*` オーバーライドは無視されます。
</Note>

これにより、次のことが可能になります。

- **セキュリティの分離**: 信頼できないエージェントのツールを制限します。
- **リソース制御**: 一部のエージェントをサンドボックス化し、その他はホスト上で実行します。
- **柔軟なポリシー**: エージェントごとに異なる権限を設定します。

<Note>
`tools.elevated` には、グローバルゲート（`tools.elevated.enabled`/`allowFrom`）とエージェントごとのゲート（`agents.entries.*.tools.elevated.enabled`/`allowFrom`）の両方があります。エージェントごとのゲートでは、グローバルゲートよりもさらに制限することしかできません。昇格コマンドを実行するには、両方で送信者が許可されている必要があります。グループを対象にする場合は、@メンションが対象のエージェントに明確に対応するよう、`agents.entries.*.groupChat.mentionPatterns` を使用します。
</Note>

詳細な例については、[マルチエージェントのサンドボックスとツール](/ja-JP/tools/multi-agent-sandbox-tools)を参照してください。

## 関連項目

- [ACP エージェント](/ja-JP/tools/acp-agents) — 外部コーディングハーネスの実行
- [チャネルルーティング](/ja-JP/channels/channel-routing) — メッセージがエージェントにルーティングされる仕組み
- [プレゼンス](/ja-JP/concepts/presence) — エージェントのプレゼンスと可用性
- [セッション](/ja-JP/concepts/session) — セッションの分離とルーティング
- [サブエージェント](/ja-JP/tools/subagents) — バックグラウンドでのエージェント実行の生成
