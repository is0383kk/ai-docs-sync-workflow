---
read_when:
    - OpenClaw を初めてセットアップする
    - 一般的な設定パターンを探す
    - 特定の設定セクションへの移動
summary: 設定の概要：一般的なタスク、クイックセットアップ、完全なリファレンスへのリンク
title: 設定
x-i18n:
    generated_at: "2026-07-26T09:34:56Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 09cc04efa16f32e12d6ebcea7a1d36b336df32227fe66953c5d70107708ee6c3
    source_path: gateway/configuration.md
    workflow: 16
---

OpenClaw は、`~/.openclaw/openclaw.json` からオプションの <Tooltip tip="JSON5 はコメントと末尾のカンマをサポートします">**JSON5**</Tooltip> 設定を読み込みます。ファイルが存在しない場合、OpenClaw は安全なデフォルトを使用します。

有効な設定パスは通常のファイルでなければなりません。OpenClaw による書き込みでは、ファイルをアトミックに置き換える（パス上にリネームする）ため、シンボリックリンクされた `openclaw.json` ではリンク先への書き込みではなく、リンク先自体が置き換えられます。シンボリックリンクを使用した設定レイアウトは避けてください。設定をデフォルトの状態ディレクトリ外に置く場合は、`OPENCLAW_CONFIG_PATH` が実ファイルを直接指すようにしてください。

設定を追加する一般的な理由：

- チャンネルを接続し、ボットにメッセージを送信できるユーザーを制御する
- モデル、ツール、サンドボックス化、自動化（cron、フック）を設定する
- セッション、メディア、ネットワーク、UI を調整する

利用可能なすべてのフィールドについては、[完全なリファレンス](/ja-JP/gateway/configuration-reference)を参照してください。

設定は 2 つの領域に分けるルールに従います。ルートの兄弟項目にはインフラストラクチャとエージェント間で共通のデフォルトを配置し、`agents.defaults` にはエージェントループの動作を配置します。スキーマがエージェントごとのオーバーライドをサポートしている場合、`agents.entries` 配下のエントリでどちらの領域もオーバーライドできます。

エージェントと自動化では、設定を編集する前に、フィールド単位の正確な
ドキュメントを `config.schema.lookup` で確認してください。このページはタスク指向のガイダンスに使用し、
より広範なフィールド一覧とデフォルトについては
[設定リファレンス](/ja-JP/gateway/configuration-reference)を参照してください。

<Tip>
**設定は初めてですか？** 対話形式のセットアップには `openclaw onboard` から始めるか、完全なコピー＆ペースト用設定について[設定例](/ja-JP/gateway/configuration-examples)ガイドを確認してください。
</Tip>

## 最小構成

```json5
// ~/.openclaw/openclaw.json
{
  agents: { defaults: { workspace: "~/.openclaw/workspace" } },
  channels: { whatsapp: { allowFrom: ["+15555550123"] } },
}
```

## 設定の編集

<Tabs>
  <Tab title="対話形式のウィザード">
    ```bash
    openclaw onboard       # 完全なオンボーディングフロー
    openclaw configure     # 設定ウィザード
    ```
  </Tab>
  <Tab title="CLI（ワンライナー）">
    ```bash
    openclaw config get agents.defaults.workspace
    openclaw config set agents.defaults.heartbeat.every "2h"
    openclaw config unset plugins.entries.brave.config.webSearch.apiKey
    ```
  </Tab>
  <Tab title="コントロール UI">
    [http://127.0.0.1:18789](http://127.0.0.1:18789) を開き、**Config** タブを使用します。
    コントロール UI は、ライブ設定スキーマからフォームをレンダリングします。利用可能な場合は、
    フィールドの `title` / `description` ドキュメントメタデータ、Plugin とチャンネルのスキーマも含まれ、
    非常時の手段として **Raw JSON** エディターも使用できます。詳細表示
    UI やその他のツール向けに、Gateway は `config.schema.lookup` も公開しており、
    パスで範囲指定された 1 つのスキーマノードと、直下の子要素の概要を取得できます。
    設定では一般的なフィールドが最初に表示されます。各セクションの高度なフィールドは、
    折りたたまれた **Advanced (N)** グループに保持されます。すべての
    グループを展開するには **Show advanced** を使用します。設定検索では常に両方の階層が検索対象となり、
    必要に応じて一致する高度なグループが開きます。
  </Tab>
  <Tab title="直接編集">
    `~/.openclaw/openclaw.json` を直接編集します。Gateway はファイルを監視し、変更を自動的に適用します（[ホットリロード](#config-hot-reload)を参照）。
  </Tab>
</Tabs>

## 厳格な検証

<Warning>
OpenClaw は、スキーマに完全に一致する設定のみを受け入れます。不明なキー、不正な型、無効な値がある場合、Gateway は**起動を拒否します**。ルートレベルで唯一の例外は `$schema`（文字列）で、エディターが JSON Schema メタデータを付加できるようにするものです。
</Warning>

`openclaw config schema` は、コントロール UI と検証で使用される正規の JSON Schema を出力します。
`config.schema.lookup` は、詳細表示ツール向けに、パスで範囲指定された単一ノードと
子要素の概要を取得します。フィールドの `title`/`description` ドキュメントメタデータは、
ネストされたオブジェクト、ワイルドカード（`*`）、配列項目（`[]`）、および `anyOf`/
`oneOf`/`allOf` の分岐にも引き継がれます。マニフェストレジストリが読み込まれると、
実行時の Plugin とチャンネルのスキーマが統合されます。

すべての設定リーフには、`uiHints` で一般または高度の表示階層が設定されています。
`advanced: false` は一般設定、`advanced: true` は高度な
設定を示します。リーフに直接のヒントがない場合は、最も近い祖先の階層を継承します。
宣言された祖先がないパスは、デフォルトで高度になります。これは表示にのみ影響し、
検証、デフォルト、リロード動作、キーを設定できるかどうかには影響しません。

検証に失敗した場合：

- Gateway は起動しません
- 診断コマンドのみ動作します（`openclaw doctor`、`openclaw logs`、`openclaw health`、`openclaw status`）
- 正確な問題を確認するには `openclaw doctor` を実行します
- 修復を適用するには `openclaw doctor --fix` を実行します（`--repair` は同じフラグで、`--yes` はプロンプトを省略します）

Gateway は正常に起動するたびに、信頼済みの最後の正常なコピーを保持しますが、
起動時やホットリロード時には自動的に復元されません。復元するのは `openclaw doctor --fix`
のみです。`openclaw.json` の検証（Plugin ローカルの検証を含む）が失敗した場合、Gateway の
起動は失敗するか、リロードがスキップされ、現在のランタイムは最後に受け入れられた
設定を使用し続けます。拒否された書き込みも、確認用に `<path>.rejected.<timestamp>` として保存されます。
Gateway は、誤って上書きしたように見える書き込みをブロックします。具体的には、`gateway.mode` の削除、
`meta` ブロックの喪失、またはファイルサイズが半分を超えて縮小する場合です。ただし、書き込みで
破壊的変更を明示的に許可している場合を除きます。候補に `***` や `[redacted]` のような
秘匿化されたシークレットのプレースホルダーが含まれる場合、最後の正常なコピーへの昇格はスキップされます。

## 一般的なタスク

<AccordionGroup>
  <Accordion title="チャンネルをセットアップする（WhatsApp、Telegram、Discord など）">
    各チャンネルには、`channels.<provider>` 配下に独自の設定セクションがあります。セットアップ手順については、各チャンネル専用ページを参照してください：

    - [Discord](/ja-JP/channels/discord) - `channels.discord`
    - [Feishu](/ja-JP/channels/feishu) - `channels.feishu`
    - [Google Chat](/ja-JP/channels/googlechat) - `channels.googlechat`
    - [iMessage](/ja-JP/channels/imessage) - `channels.imessage`
    - [Mattermost](/ja-JP/channels/mattermost) - `channels.mattermost`
    - [Microsoft Teams](/ja-JP/channels/msteams) - `channels.msteams`
    - [Signal](/ja-JP/channels/signal) - `channels.signal`
    - [Slack](/ja-JP/channels/slack) - `channels.slack`
    - [Telegram](/ja-JP/channels/telegram) - `channels.telegram`
    - [WhatsApp](/ja-JP/channels/whatsapp) - `channels.whatsapp`

    すべてのチャンネルで同じ DM ポリシーパターンを使用します：

    ```json5
    {
      channels: {
        telegram: {
          enabled: true,
          botToken: "123:abc",
          dmPolicy: "pairing",   // ペアリング | 許可リスト | 開放 | 無効
          allowFrom: ["tg:123"], // 許可リスト/開放の場合のみ
        },
      },
    }
    ```

  </Accordion>

  <Accordion title="モデルを選択して設定する">
    プライマリモデルとオプションのフォールバックを設定します：

    ```json5
    {
      agents: {
        defaults: {
          model: {
            primary: "anthropic/claude-sonnet-4-6",
            fallbacks: ["openai/gpt-5.4"],
          },
          models: {
            "anthropic/claude-sonnet-4-6": { alias: "Sonnet" },
            "openai/gpt-5.4": { alias: "GPT" },
          },
        },
      },
    }
    ```

    - `agents.defaults.models` はエイリアスとモデルごとの設定を保存します。エントリを追加しても、`/model` または `--model` のオーバーライドが制限されることはありません。
    - `agents.defaults.modelPolicy.allow` は、オーバーライドとモデル選択用の明示的な許可リストです。完全一致する参照と `provider/*` ワイルドカードを受け入れます。任意のモデルを許可するには、省略するか `[]` を使用します。
    - モデル参照には `provider/model` 形式を使用します（例：`anthropic/claude-opus-4-6`）。
    - `agents.defaults.imageMaxDimensionPx` は、トランスクリプト/ツール画像のダウンスケーリングを制御します（デフォルトは `1200`）。値を小さくすると通常、スクリーンショットが多い実行でビジョントークンの使用量が減少します。
    - チャット内でのモデル切り替えについては[モデル CLI](/ja-JP/concepts/models)、認証のローテーションとフォールバック動作については[モデルのフェイルオーバー](/ja-JP/concepts/model-failover)を参照してください。
    - カスタム/セルフホストのプロバイダーについては、リファレンスの[カスタムプロバイダー](/ja-JP/gateway/config-tools#custom-providers-and-base-urls)を参照してください。

  </Accordion>

  <Accordion title="ボットにメッセージを送信できるユーザーを制御する">
    DM アクセスは、チャンネルごとに `dmPolicy`（デフォルトは `"pairing"`）で制御します：

    - `"pairing"`：不明な送信者には、承認用の 1 回限りのペアリングコードが発行されます
    - `"allowlist"`：`allowFrom`（またはペアリング済み許可ストア）に含まれる送信者のみ
    - `"open"`：すべての受信 DM を許可します（`allowFrom: ["*"]` が必要）
    - `"disabled"`：すべての DM を無視します

    グループでは、`groupPolicy`（`"allowlist" | "open" | "disabled"`）と `groupAllowFrom`、またはチャンネル固有の許可リストを使用します。

    チャンネルごとの詳細については、[完全なリファレンス](/ja-JP/gateway/config-channels#dm-and-group-access)を参照してください。

  </Accordion>

  <Accordion title="グループチャットのメンションゲートを設定する">
    グループメッセージでは、デフォルトで**メンションが必須**です。エージェントごとにトリガーパターンを設定します。通常のグループ/チャンネル返信は自動的に投稿されます。エージェントが発言するタイミングを判断する必要がある共有ルームでは、メッセージツール経由の処理を明示的に有効にします：

    ```json5
    {
      messages: {
        visibleReplies: "automatic", // すべての場所でメッセージツールによる送信を必須にするには "message_tool" を設定
        groupChat: {
          visibleReplies: "message_tool", // 明示的に有効化。表示される出力には message(action=send) が必要
          unmentionedInbound: "room_event", // メンションなしで常時流れるグループ会話は、通知されないコンテキスト
        },
      },
      agents: {
        list: [
          {
            id: "main",
            groupChat: {
              mentionPatterns: ["@openclaw", "openclaw"],
            },
          },
        ],
      },
      channels: {
        whatsapp: {
          groups: { "*": { requireMention: true } },
        },
      },
    }
    ```

    - **メタデータのメンション**：ネイティブの @メンション（WhatsApp のタップによるメンション、Telegram の @bot など）
    - **テキストパターン**：`mentionPatterns` 内の安全な正規表現パターン
    - **表示される返信**：`messages.visibleReplies` ではメッセージツールによる送信を全体で必須にでき、`messages.groupChat.visibleReplies` でグループ/チャンネル向けにオーバーライドできます。
    - 表示される返信モード、チャンネルごとのオーバーライド、セルフチャットモードについては、[完全なリファレンス](/ja-JP/gateway/config-channels#group-chat-mention-gating)を参照してください。

  </Accordion>

  <Accordion title="エージェントごとに Skills を制限する">
    共有ベースラインには `agents.defaults.skills` を使用し、特定の
    エージェントを `agents.entries.*.skills` でオーバーライドします：

    ```json5
    {
      agents: {
        defaults: {
          skills: ["github", "weather"],
        },
        list: [
          { id: "writer" }, // github、weather を継承
          { id: "docs", skills: ["docs-search"] }, // デフォルトを置換
          { id: "locked-down", skills: [] }, // Skills なし
        ],
      },
    }
    ```

    - デフォルトで Skills を無制限にするには、`agents.defaults.skills` を省略します。
    - デフォルトを継承するには、`agents.entries.*.skills` を省略します。
    - Skills なしにするには、`agents.entries.*.skills: []` を設定します。
    - [Skills](/ja-JP/tools/skills)、[Skills の設定](/ja-JP/tools/skills-config)、および
      [設定リファレンス](/ja-JP/gateway/config-agents#agents-defaults-skills)を参照してください。

  </Accordion>

  <Accordion title="チャンネルごとのヘルスモニタリングを設定する">
    チャンネルまたはアカウントの自動ヘルス再起動を無効または有効にします：

    ```json5
    {
      channels: {
        telegram: {
          healthMonitor: { enabled: false },
          accounts: {
            alerts: {
              healthMonitor: { enabled: true },
            },
          },
        },
      },
    }
    ```

    - 1 つのチャンネルまたはアカウントの自動再起動を制御するには、`channels.<provider>.healthMonitor.enabled` または `channels.<provider>.accounts.<id>.healthMonitor.enabled` を使用します。
    - 運用上のデバッグについては[ヘルスチェック](/ja-JP/gateway/health)、すべてのフィールドについては[完全なリファレンス](/ja-JP/gateway/configuration-reference#gateway)を参照してください。

  </Accordion>

  <Accordion title="セッションとリセットを設定する">
    セッションは会話の継続性と分離を制御します。

    ```json5
    {
      session: {
        dmScope: "per-channel-peer",  // recommended for multi-user
        threadBindings: {
          enabled: true,
          idleHours: 24,
          maxAgeHours: 0,
        },
        reset: {
          mode: "daily",
          atHour: 4,
          idleMinutes: 120,
        },
      },
    }
    ```

    - `dmScope`: `main`（共有）| `per-peer` | `per-channel-peer` | `per-account-channel-peer`
    - `threadBindings`: スレッドにバインドされたセッションルーティングのグローバルなデフォルト。`/focus`、`/unfocus`、`/agents`、`/session idle`、`/session max-age`を使用して、セッションごとにバインド、バインド解除、一覧表示、調整を行います（Discord はスレッド、Telegram はトピック／会話をバインドします）。
    - スコープ、ID リンク、送信ポリシーについては、[セッション管理](/ja-JP/concepts/session)を参照してください。
    - すべてのフィールドについては、[完全なリファレンス](/ja-JP/gateway/config-agents#session)を参照してください。

  </Accordion>

  <Accordion title="サンドボックス化を有効にする">
    エージェントセッションを分離されたサンドボックスランタイムで実行します。

    ```json5
    {
      agents: {
        defaults: {
          sandbox: {
            mode: "non-main",  // off | non-main | all
            scope: "agent",    // session | agent | shared
          },
        },
      },
    }
    ```

    まずイメージをビルドします。ソースチェックアウトでは `scripts/sandbox-setup.sh` を実行し、npm インストールでは[サンドボックス化 § イメージとセットアップ](/ja-JP/gateway/sandboxing#images-and-setup)に記載されたインラインの `docker build` コマンドを参照してください。

    詳細なガイドについては[サンドボックス化](/ja-JP/gateway/sandboxing)、すべてのオプションについては[完全なリファレンス](/ja-JP/gateway/config-agents#agentsdefaultssandbox)を参照してください。

  </Accordion>

  <Accordion title="公式 iOS ビルドのリレー経由プッシュを有効にする">
    公開 App Store ビルドのリレー経由プッシュでは、ホストされた OpenClaw リレー `https://ios-push-relay.openclaw.ai` を使用します。

    カスタムリレーのデプロイには、リレー URL が Gateway のリレー URL と一致する、意図的に分離された iOS ビルド／デプロイパスが必要です。カスタムリレービルドを使用している場合は、Gateway 設定で次のように指定します。

    ```json5
    {
      gateway: {
        push: {
          apns: {
            relay: {
              baseUrl: "https://relay.example.com",
              // Optional. Default: 10000
              timeoutMs: 10000,
            },
          },
        },
      },
    }
    ```

    CLI での同等の設定:

    ```bash
    openclaw config set gateway.push.apns.relay.baseUrl https://relay.example.com
    ```

    この設定の動作:

    - Gateway が外部リレーを介して `push.test`、ウェイク通知、再接続ウェイクを送信できるようにします。
    - ペアリングされた iOS アプリから転送される、登録単位の送信許可を使用します。Gateway にデプロイ全体で共通のリレートークンは必要ありません。
    - リレー経由の各登録を、iOS アプリがペアリングした Gateway の ID にバインドし、別の Gateway が保存済み登録を再利用できないようにします。
    - ローカル／手動の iOS ビルドでは直接 APNs を使用し続けます。リレー経由の送信は、リレーを介して登録された公式配布ビルドにのみ適用されます。
    - 登録トラフィックと送信トラフィックが同じリレーデプロイに到達するよう、iOS ビルドに組み込まれたリレーベース URL と一致させる必要があります。

    エンドツーエンドのフロー:

    1. 公式 iOS アプリをインストールします。
    2. 任意: 意図的に分離されたカスタムリレービルドを使用する場合に限り、Gateway で `gateway.push.apns.relay.baseUrl` を設定します。
    3. iOS アプリを Gateway とペアリングし、Node セッションとオペレーターセッションの両方を接続します。
    4. iOS アプリは Gateway の ID を取得し、App Attest とアプリのレシートを使用してリレーに登録した後、リレー経由の `push.apns.register` ペイロードをペアリング済み Gateway に公開します。
    5. Gateway はリレーハンドルと送信許可を保存し、それらを `push.test`、ウェイク通知、再接続ウェイクに使用します。

    運用上の注意:

    - iOS アプリを別の Gateway に切り替えた場合は、その Gateway にバインドされた新しいリレー登録を公開できるよう、アプリを再接続してください。
    - 異なるリレーデプロイを参照する新しい iOS ビルドを配布した場合、アプリは古いリレーオリジンを再利用せず、キャッシュ済みのリレー登録を更新します。

    互換性に関する注意:

    - `OPENCLAW_APNS_RELAY_BASE_URL` と `OPENCLAW_APNS_RELAY_TIMEOUT_MS` は、一時的な環境変数によるオーバーライドとして引き続き機能します。
    - カスタム Gateway のリレー URL は、iOS ビルドに組み込まれたリレーベース URL と一致する必要があります。公開 App Store リリースレーンでは、カスタム iOS リレー URL のオーバーライドが拒否されます。
    - `OPENCLAW_APNS_RELAY_ALLOW_HTTP=true` は local loopback 専用の開発用エスケープハッチとして引き続き使用できます。HTTP リレー URL を設定に永続化しないでください。

    エンドツーエンドのフローについては[iOS アプリ](/ja-JP/platforms/ios#relay-backed-push-for-official-builds)、リレーのセキュリティモデルについては[認証と信頼のフロー](/ja-JP/platforms/ios#authentication-and-trust-flow)を参照してください。

  </Accordion>

  <Accordion title="Heartbeat（定期チェックイン）を設定する">
    ```json5
    {
      agents: {
        defaults: {
          heartbeat: {
            every: "30m",
            target: "last",
          },
        },
      },
    }
    ```

    - `every`: 期間文字列（`30m`、`2h`）。無効にするには `0m` を設定します。デフォルト: `30m`。
    - `target`: `last` | `none` | `<channel-id>`（例: `discord`、`matrix`、`telegram`、`whatsapp`）
    - `directPolicy`: DM 形式の Heartbeat ターゲットでは `allow`（デフォルト）または `block`
    - 詳細なガイドについては、[Heartbeat](/ja-JP/gateway/heartbeat)を参照してください。

  </Accordion>

  <Accordion title="Cron ジョブを設定する">
    ```json5
    {
      cron: {
        enabled: true,
        sessionRetention: "24h",
      },
    }
    ```

    - `sessionRetention`: 完了した分離実行セッションを SQLite のセッション行から削除します（デフォルトは `24h`。無効にするには `false` を設定）。
    - 実行履歴では、ジョブごとに最新のターミナル行 2000 件が自動的に保持されます。失われた行には 24 時間のクリーンアップ期間が引き続き適用されます。
    - 機能の概要と CLI の例については、[Cron ジョブ](/ja-JP/automation/cron-jobs)を参照してください。

  </Accordion>

  <Accordion title="Webhook（フック）を設定する">
    Gateway で HTTP Webhook エンドポイントを有効にします。

    ```json5
    {
      hooks: {
        enabled: true,
        token: "shared-secret",
        path: "/hooks",
        defaultSessionKey: "hook:ingress",
        allowRequestSessionKey: false,
        allowedSessionKeyPrefixes: ["hook:"],
        mappings: [
          {
            match: { path: "gmail" },
            action: "agent",
            agentId: "main",
            deliver: true,
          },
        ],
      },
    }
    ```

    セキュリティ上の注意:
    - すべてのフック／Webhook ペイロードの内容を信頼できない入力として扱ってください。
    - 専用の `hooks.token` を使用し、使用中の Gateway 認証シークレット（`gateway.auth.token` / `OPENCLAW_GATEWAY_TOKEN` または `gateway.auth.password` / `OPENCLAW_GATEWAY_PASSWORD`）を再利用しないでください。
    - フック認証はヘッダーのみ（`Authorization: Bearer ...` または `x-openclaw-token`）を使用します。クエリ文字列のトークンは拒否されます。
    - `hooks.path` を `/` にすることはできません。Webhook の受信パスには `/hooks` などの専用サブパスを使用してください。
    - 厳密に範囲を限定したデバッグを行う場合を除き、安全でないコンテンツのバイパスフラグ（`hooks.gmail.allowUnsafeExternalContent`、`hooks.mappings[].allowUnsafeExternalContent`）は無効のままにしてください。
    - `hooks.allowRequestSessionKey` を有効にする場合は、呼び出し元が選択するセッションキーを制限するため、`hooks.allowedSessionKeyPrefixes` も設定してください。
    - フックによって駆動されるエージェントには、性能の高い最新モデルの階層と厳格なツールポリシー（可能であれば、メッセージングのみに制限し、サンドボックス化を併用するなど）を推奨します。

    すべてのマッピングオプションと Gmail 連携については、[完全なリファレンス](/ja-JP/gateway/configuration-reference#hooks)を参照してください。

  </Accordion>

  <Accordion title="マルチエージェントルーティングを設定する">
    個別のワークスペースとセッションを持つ複数の分離エージェントを実行します。

    ```json5
    {
      agents: {
        list: [
          { id: "home", default: true, workspace: "~/.openclaw/workspace-home" },
          { id: "work", workspace: "~/.openclaw/workspace-work" },
        ],
      },
      bindings: [
        { agentId: "home", match: { channel: "whatsapp", accountId: "personal" } },
        { agentId: "work", match: { channel: "whatsapp", accountId: "biz" } },
      ],
    }
    ```

    バインドルールとエージェントごとのアクセスプロファイルについては、[マルチエージェント](/ja-JP/concepts/multi-agent)と[完全なリファレンス](/ja-JP/gateway/config-agents#multi-agent-routing)を参照してください。

  </Accordion>

  <Accordion title="設定を複数ファイルに分割する（$include）">
    大規模な設定を整理するには、`$include` を使用します。

    ```json5
    // ~/.openclaw/openclaw.json
    {
      gateway: { port: 18789 },
      agents: { $include: "./agents.json5" },
      broadcast: {
        $include: ["./clients/a.json5", "./clients/b.json5"],
      },
    }
    ```

    - **単一ファイル**: それを含むオブジェクトを置き換えます
    - **ファイルの配列**: 順番にディープマージされ（後の値が優先）、最大 10 階層までネストできます
    - **兄弟キー**: インクルード後にマージされます（インクルードされた値を上書きします）
    - **相対パス**: インクルード元ファイルを基準に解決されます
    - **パス形式**: インクルードパスに null バイトを含めることはできず、解決前と解決後の両方で 4096 文字未満である必要があります
    - **OpenClaw による書き込み**: 書き込みによって、`plugins: { $include: "./plugins.json5" }` のような単一ファイルのインクルードに基づく最上位セクションが 1 つだけ変更される場合、
      OpenClaw はそのインクルードファイルを更新し、`openclaw.json` はそのまま維持します
    - **サポートされていないライトスルー**: ルートインクルード、インクルード配列、兄弟キーによるオーバーライドを持つインクルードでは、
      設定をフラット化せず、OpenClaw による書き込みをフェイルクローズします
    - **制限**: `$include` パスは、
      `openclaw.json` を格納するディレクトリ配下に解決される必要があります。複数のマシンまたはユーザー間でツリーを共有するには、
      `OPENCLAW_INCLUDE_ROOTS` に、インクルードから参照可能な追加ディレクトリのパスリスト（POSIX では `:`、Windows では `;`）を設定します。
      シンボリックリンクは解決後に再チェックされるため、字句上は設定ディレクトリ内に存在していても、
      実際のターゲットが許可されたすべてのルート外にあるパスは引き続き拒否されます。
    - **エラー処理**: ファイルの欠落、解析エラー、循環インクルード、無効なパス形式、長さ超過に対して明確なエラーを表示します

  </Accordion>
</AccordionGroup>

## 設定のホットリロード

Gateway は `~/.openclaw/openclaw.json` を監視して変更を自動的に適用します。ほとんどの設定では手動で再起動する必要はありません。

ファイルへの直接編集は、検証されるまで信頼できないものとして扱われます。ウォッチャーは、
エディターによる一時書き込み／名前変更の連続処理が落ち着くのを待ってから最終ファイルを読み取り、
無効な外部編集を拒否しますが、`openclaw.json` は書き換えません。OpenClaw による設定の
書き込みも、書き込み前に同じスキーマゲートを使用します（すべての書き込みに適用される上書き／ロールバックのルールについては、
[厳格な検証](#strict-validation)を参照してください）。

`config reload skipped (invalid config)` が表示される場合、または起動時に `Invalid
config` が報告される場合は、設定を確認し、`openclaw config validate` を実行してから、修復のために `openclaw
doctor --fix` を実行してください。チェックリストについては、[Gateway のトラブルシューティング](/ja-JP/gateway/troubleshooting#gateway-rejected-invalid-config)
を参照してください。

### リロードモード

| モード                   | 動作                                                                                |
| ---------------------- | --------------------------------------------------------------------------------------- |
| **`hybrid`**（デフォルト） | 安全な変更を即座にホット適用します。重大な変更の場合は自動的に再起動します。           |
| **`hot`**              | 安全な変更のみをホット適用します。再起動が必要な場合は警告をログに記録し、再起動は利用者が行います。 |
| **`restart`**          | 設定の変更が安全かどうかにかかわらず、変更のたびに Gateway を再起動します。                                 |
| **`off`**              | ファイル監視を無効にします。変更は次回の手動再起動時に反映されます。                 |

```json5
{
  gateway: {
    reload: { mode: "hybrid", debounceMs: 300 },
  },
}
```

### ホット適用される変更と再起動が必要な変更

ほとんどのフィールドはダウンタイムなしでホット適用されます。一部のホット適用対象セクションでは、Gateway 全体ではなく、そのサブシステム（チャンネル、Cron、Heartbeat、ヘルスモニター）のみが再起動されます。`hybrid` モードでは、Gateway の再起動が必要な変更は自動的に処理されます。

| カテゴリ            | フィールド                                                                  | Gateway の再起動が必要か      |
| ------------------- | ----------------------------------------------------------------------- | ---------------------------- |
| チャンネル            | `channels.*`、`web`（WhatsApp）- すべての組み込みおよび Plugin チャンネル       | いいえ（該当チャンネルを再起動）   |
| エージェントとモデル      | `agent`、`agents`、`models`、`routing`                                  | いいえ                           |
| 自動化          | `hooks`、`cron`、`agent.heartbeat`                                      | いいえ（該当サブシステムを再起動） |
| セッションとメッセージ | `session`、`messages`                                                   | いいえ                           |
| ツールとメディア       | `tools`、`skills`、`mcp`、`audio`、`talk`                               | いいえ                           |
| Plugin 設定       | `plugins.entries.*`、`plugins.allow`、`plugins.deny`、`plugins.enabled` | いいえ（Plugin ランタイムを再読み込み）  |
| UI とその他           | `ui`、`logging`、`identity`、`bindings`                                 | いいえ                           |
| Gateway サーバー      | `gateway.*`（ポート、バインド、認証、Tailscale、TLS、HTTP、プッシュ）              | **はい**                      |
| インフラストラクチャ      | `discovery`、`browser`、`plugins.load`、`plugins.installs`              | **はい**                      |

<Note>
`gateway.reload` と `gateway.remote` は `gateway.*` における例外であり、これらを変更しても再起動は**トリガーされません**。個々の Plugin がこの表を上書きすることもできます。読み込まれた Plugin は、再起動をトリガーする独自の設定プレフィックスを宣言できます（たとえば、組み込みの Canvas Plugin は、自身の `plugins.entries.canvas` だけでなく、`plugins.enabled`、`plugins.allow`、`plugins.deny` の変更でも Gateway を再起動します）。そのため、実際の動作は有効な Plugin によって異なります。
</Note>

### 再読み込みの計画

`$include` を介して参照されているソースファイルを編集すると、OpenClaw はフラット化されたメモリ内ビューではなく、ソースに記述されたレイアウトに基づいて再読み込みを計画します。
これにより、単一のトップレベルセクションが `plugins: { $include: "./plugins.json5" }` のような独立したインクルードファイルにある場合でも、ホット再読み込みの判断（ホット適用か再起動か）が予測可能になります。ソースレイアウトが曖昧な場合、再読み込みの計画は安全側に倒して失敗します。

## 設定 RPC（プログラムによる更新）

Gateway API 経由で設定を書き込むツールでは、次のフローを推奨します。

- `config.schema.lookup` で単一のサブツリーを調査する（浅いスキーマノードと子の概要）
- `config.get` で現在のスナップショットと `hash` を取得する
- `config.patch` で部分更新を行う（JSON マージパッチ：オブジェクトはマージ、`null` は削除。エントリが削除される場合は `replacePaths` で明示的に確認したときのみ配列を置換）
- 設定全体を置き換える場合にのみ `config.apply` を使用する
- 明示的な自己更新と再起動には `update.run` を使用する。再起動後のセッションで追加のターンを 1 回実行する場合は `continuationMessage` を含める
- 最新の更新再起動センチネルを調査し、再起動後に実行中のバージョンを確認するには `update.status` を使用する

エージェントは、フィールド単位の正確なドキュメントと制約を確認する際、まず `config.schema.lookup` を参照する必要があります。より広範な設定マップ、デフォルト、または各サブシステムのリファレンスへのリンクが必要な場合は、[設定リファレンス](/ja-JP/gateway/configuration-reference)を使用してください。

<Note>
コントロールプレーンへの書き込み（`config.apply`、`config.patch`、`update.run`）は、`deviceId+clientIp` ごと、メソッドごとに、60 秒あたり 30 リクエストに制限されます。[レート制限](/gateway/security/rate-limiting)を参照してください。再起動リクエストは統合された後、再起動サイクル間に 30 秒のクールダウンが適用されます。
`update.status` は読み取り専用ですが、再起動センチネルに更新手順の概要やコマンド出力の末尾が含まれる可能性があるため、管理者スコープです。
</Note>

部分パッチの例：

```bash
openclaw gateway call config.get --params '{}'  # payload.hash を取得
openclaw gateway call config.patch --params '{
  "raw": "{ channels: { telegram: { groups: { \"*\": { requireMention: false } } } } }",
  "baseHash": "<hash>"
}'
```

`config.apply` と `config.patch` はどちらも、`raw`、`baseHash`、`sessionKey`、`note`、`restartDelayMs` を受け付けます。設定ファイルがすでに存在する場合、どちらのメソッドでも `baseHash` が必須です（既存の設定がない状態での初回書き込みでは、このチェックは省略されます）。

`config.patch` は、配列の置換が意図的である設定パスの配列である `replacePaths` も受け付けます。パッチによって既存の配列がより少ないエントリの配列に置換または削除される場合、その正確なパスが `replacePaths` に含まれていない限り、Gateway は書き込みを拒否します。配列エントリ内のネストされた配列では、`agents.entries.*.skills` のように `[]` を使用します。これにより、切り詰められた `config.get` スナップショットが、ルーティング配列や許可リスト配列を意図せず上書きするのを防ぎます。設定全体を置き換える場合は `config.apply` を使用してください。

## 環境変数

OpenClaw は親プロセスに加えて、次の場所から環境変数を読み取ります。

- 現在の作業ディレクトリにある `.env`（存在する場合）
- `~/.openclaw/.env`（グローバルフォールバック）

どちらのファイルも既存の環境変数を上書きしません。設定内でインライン環境変数を設定することもできます。

```json5
{
  env: {
    OPENROUTER_API_KEY: "sk-or-...",
    vars: { GROQ_API_KEY: "gsk-..." },
  },
}
```

<Accordion title="シェル環境変数のインポート（任意）">
  有効にした場合、必要なキーが設定されていなければ、OpenClaw はログインシェルを実行し、不足しているキーのみをインポートします。

```json5
{
  env: {
    shellEnv: { enabled: true, timeoutMs: 15000 },
  },
}
```

対応する環境変数：`OPENCLAW_LOAD_SHELL_ENV=1`。デフォルトの `timeoutMs`：`15000`。
</Accordion>

<Accordion title="設定値での環境変数の置換">
  任意の設定文字列値で `${VAR_NAME}` を使用して環境変数を参照します。

```json5
{
  gateway: { auth: { token: "${OPENCLAW_GATEWAY_TOKEN}" } },
  models: { providers: { custom: { apiKey: "${CUSTOM_API_KEY}" } } },
}
```

ルール：

- 一致するのは大文字の名前のみ：`[A-Z_][A-Z0-9_]*`
- 変数が未設定または空の場合、読み込み時にエラーが発生する
- リテラルとして出力するには `$${VAR}` でエスケープする
- `$include` ファイル内でも機能する
- インライン置換：`"${BASE}/v1"` → `"https://api.example.com/v1"`

</Accordion>

<Accordion title="シークレット参照（環境変数、ファイル、実行）">
  SecretRef オブジェクトをサポートするフィールドでは、次のように使用できます。

```json5
{
  models: {
    providers: {
      openai: { apiKey: { source: "env", provider: "default", id: "OPENAI_API_KEY" } },
    },
  },
  skills: {
    entries: {
      "image-lab": {
        apiKey: {
          source: "file",
          provider: "filemain",
          id: "/skills/entries/image-lab/apiKey",
        },
      },
    },
  },
  channels: {
    googlechat: {
      serviceAccount: {
        source: "exec",
        provider: "vault",
        id: "channels/googlechat/serviceAccount",
      },
    },
  },
}
```

SecretRef の詳細（`env`/`file`/`exec` の `secrets.providers` を含む）は、[シークレット管理](/ja-JP/gateway/secrets)に記載されています。
サポートされる認証情報パスは、[SecretRef 認証情報サーフェス](/ja-JP/reference/secretref-credential-surface)に一覧表示されています。
</Accordion>

優先順位とソースの詳細については、[環境](/ja-JP/help/environment)を参照してください。

## 完全なリファレンス

フィールドごとの完全なリファレンスについては、**[設定リファレンス](/ja-JP/gateway/configuration-reference)**を参照してください。

---

_関連項目：[設定例](/ja-JP/gateway/configuration-examples) · [設定リファレンス](/ja-JP/gateway/configuration-reference) · [Doctor](/ja-JP/gateway/doctor)_

## 関連項目

- [設定リファレンス](/ja-JP/gateway/configuration-reference)
- [設定例](/ja-JP/gateway/configuration-examples)
- [Gateway ランブック](/ja-JP/gateway)
