---
read_when:
    - Slack のセットアップ、または Slack のソケット、HTTP、リレーモードのデバッグ
summary: Slack のセットアップと実行時の動作（Socket Mode、HTTP Request URL、リレーモード）
title: Slack
x-i18n:
    generated_at: "2026-07-26T08:55:04Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: e0f974ddf8e6965b09cede6a16f171434915a994fa3c1fc744d2350399941bee
    source_path: channels/slack.md
    workflow: 16
---

Slack サポートは、Slack アプリ連携を介した DM とチャンネルに対応しています。デフォルトのトランスポートは Socket Mode です。HTTP Request URL にも対応しています。リレーモードは、信頼できるルーターが Slack の受信を管理するマネージドデプロイ向けです。

<CardGroup cols={3}>
  <Card title="ペアリング" icon="link" href="/ja-JP/channels/pairing">
    Slack DM のデフォルトはペアリングモードです。
  </Card>
  <Card title="スラッシュコマンド" icon="terminal" href="/ja-JP/tools/slash-commands">
    ネイティブコマンドの動作とコマンドカタログ。
  </Card>
  <Card title="チャンネルのトラブルシューティング" icon="wrench" href="/ja-JP/channels/troubleshooting">
    チャンネル横断の診断と修復手順。
  </Card>
</CardGroup>

## トランスポートの選択

Socket Mode と HTTP Request URL は、メッセージング、スラッシュコマンド、App Home、インタラクティブ機能について同等の機能を提供します。機能ではなく、デプロイ構成に応じて選択してください。

| 検討事項                     | Socket Mode（デフォルト）                                                                                                                           | HTTP Request URL                                                                                               |
| ---------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------- |
| 公開 Gateway URL             | 不要                                                                                                                                                 | 必須（DNS、TLS、リバースプロキシまたはトンネル）                                                               |
| 外向きネットワーク           | `wss-primary.slack.com` への外向き WSS に到達できる必要があります                                                                                         | 外向き WS は不要。受信 HTTPS のみ                                                                              |
| 必要なトークン               | Bot アイデンティティ: bot token + `connections:write` を持つ App-Level Token。ユーザーアイデンティティ: user token + App-Level Token                  | Bot アイデンティティ: bot token + Signing Secret。ユーザーアイデンティティ: user token + Signing Secret       |
| 開発用ノート PC / ファイアウォール内 | そのまま動作します                                                                                                                            | 公開トンネル（ngrok、Cloudflare Tunnel、Tailscale Funnel）またはステージング Gateway が必要です                 |
| 水平スケーリング             | ホストごと、アプリごとに 1 つの Socket Mode セッション。複数の Gateway には個別の Slack アプリが必要です                                              | ステートレスな POST ハンドラー。複数の Gateway レプリカがロードバランサーの背後で 1 つのアプリを共有できます   |
| 1 つの Gateway での複数アカウント | 対応。各アカウントが独自の WS を開きます                                                                                                       | 対応。登録が衝突しないよう、各アカウントに一意の `webhookPath`（デフォルト `/slack/events`）が必要です |
| スラッシュコマンドのトランスポート | WS 接続経由で配信され、`slash_commands[].url` は無視されます                                                                                        | Slack が `slash_commands[].url` に POST します。コマンドをディスパッチするには、このフィールドが必須です           |
| リクエスト署名               | 使用しません（認証には App-Level Token を使用）                                                                                                      | Slack がすべてのリクエストに署名し、OpenClaw が `signingSecret` で検証します                                |
| 接続切断時の復旧             | Slack SDK の自動再接続が有効です。OpenClaw も、失敗した Socket Mode セッションを上限付きバックオフで再起動します。Pong タイムアウトのトランスポート調整が適用されます。 | 切断される永続接続はありません。再試行は Slack からリクエスト単位で行われます                                  |

<Note>
  外向きの `*.slack.com` に到達できる一方で受信 HTTPS を受け付けられない、単一 Gateway のホスト、開発用ノート PC、オンプレミスネットワークでは、**Socket Mode を選択**してください。

ロードバランサーの背後で複数の Gateway レプリカを実行する場合、外向き WSS がブロックされている一方で受信 HTTPS が許可されている場合、またはリバースプロキシですでに Slack Webhook を終端している場合は、**HTTP Request URL を選択**してください。
</Note>

<Warning>
  Slack は 1 つのアプリに対して複数の Socket Mode 接続を維持でき、各ペイロードを任意の接続に配信する可能性があります。そのため、Slack アプリを共有する個別の OpenClaw Gateway には、同等のルーティングおよび認可設定が必要です。それ以外の場合は、Gateway ごとに個別の Slack アプリ、単一のリレー受信口、またはロードバランサーの背後にある HTTP Request URL を使用してください。[Socket Mode の使用](https://docs.slack.dev/apis/events-api/using-socket-mode#using-multiple-connections)を参照してください。
</Warning>

### リレーモード

リレーモードは、Slack の受信を OpenClaw Gateway から分離します。信頼できるルーターが単一の Slack Socket Mode 接続を管理し、宛先 Gateway を選択して、認証済み WebSocket 経由で型付きイベントを転送します。Gateway は、Slack Web API の外向き呼び出しに引き続き独自の bot token を使用します。

```json5
{
  channels: {
    slack: {
      mode: "relay",
      botToken: { source: "env", provider: "default", id: "SLACK_BOT_TOKEN" },
      relay: {
        url: "wss://router.example.com/gateway/ws",
        authToken: { source: "env", provider: "default", id: "SLACK_RELAY_AUTH_TOKEN" },
        gatewayId: "team-gateway",
      },
    },
  },
}
```

リレー URL は、localhost を対象とする場合を除き、`wss://` を使用する必要があります。Bearer Token とルーターのルートテーブルは Slack の認可境界の一部として扱ってください。ルーティングされたイベントは、認可済みのアクティベーションとして通常の Slack メッセージハンドラーに入ります。WebSocket の `hello` フレーム内でルーターが提供する `slack_identity` により、デフォルトの外向きユーザー名とアイコンを設定できますが、呼び出し元が明示的に指定したアイデンティティが引き続き優先されます。リレー接続は Socket Mode と同じ上限付きバックオフのタイミングで再接続し、切断するたびにルーターが提供したアイデンティティをクリアします。

### Enterprise Grid の組織全体へのインストール

1 つの Slack アカウントで、Enterprise Grid の組織全体へのインストールの対象となるすべてのワークスペースからメッセージを受信できます。直接の Socket Mode または HTTP Request URL を選択してください。リレーモードは Enterprise アカウントではサポートされません。以下の最小権限マニフェストはいずれも、V1 の `message` および `app_mention` イベントパス、即時返信、リスナーが管理するステータスリアクションのみを有効にします。

#### Socket Mode

```json
{
  "display_information": {
    "name": "OpenClaw",
    "description": "OpenClaw 用 Slack コネクター"
  },
  "features": {
    "bot_user": { "display_name": "OpenClaw", "always_online": true }
  },
  "oauth_config": {
    "scopes": {
      "bot": [
        "app_mentions:read",
        "channels:history",
        "channels:read",
        "chat:write",
        "files:read",
        "files:write",
        "groups:history",
        "groups:read",
        "im:history",
        "im:read",
        "mpim:history",
        "mpim:read",
        "reactions:write",
        "users:read"
      ]
    }
  },
  "settings": {
    "org_deploy_enabled": true,
    "socket_mode_enabled": true,
    "event_subscriptions": {
      "bot_events": [
        "app_mention",
        "message.channels",
        "message.groups",
        "message.im",
        "message.mpim"
      ]
    }
  }
}
```

Enterprise Grid Org Admin または Org Owner にアプリの承認を依頼し、組織レベルでインストールして、インストールの対象となるワークスペースを選択します。OpenClaw を起動する前に、対象とするすべてのワークスペースでアプリを利用できることを確認してください。Socket Mode 用に `connections:write` を持つ App-Level Token を生成し、組織インストールから bot token をコピーします。組織インストール済みの bot token を使用するアカウントを設定します。

```json5
{
  channels: {
    slack: {
      enabled: true,
      mode: "socket",
      enterpriseOrgInstall: true,
      appToken: { source: "env", provider: "default", id: "SLACK_APP_TOKEN" },
      botToken: { source: "env", provider: "default", id: "SLACK_BOT_TOKEN" },
      dmPolicy: "open",
      allowFrom: ["*"],
      groupPolicy: "allowlist",
      channels: {
        C0123456789: { requireMention: true },
      },
    },
  },
}
```

#### HTTP Request URL

Gateway に公開 HTTPS エンドポイントがあり、Socket Mode 接続を開かない場合は HTTP モードを使用します。例の URL を Gateway の公開 `webhookPath` URL（デフォルト `/slack/events`）に置き換えてください。

```json
{
  "display_information": {
    "name": "OpenClaw",
    "description": "OpenClaw 用 Slack コネクター"
  },
  "features": {
    "bot_user": { "display_name": "OpenClaw", "always_online": true }
  },
  "oauth_config": {
    "scopes": {
      "bot": [
        "app_mentions:read",
        "channels:history",
        "channels:read",
        "chat:write",
        "files:read",
        "files:write",
        "groups:history",
        "groups:read",
        "im:history",
        "im:read",
        "mpim:history",
        "mpim:read",
        "reactions:write",
        "users:read"
      ]
    }
  },
  "settings": {
    "org_deploy_enabled": true,
    "event_subscriptions": {
      "request_url": "https://gateway-host.example.com/slack/events",
      "bot_events": [
        "app_mention",
        "message.channels",
        "message.groups",
        "message.im",
        "message.mpim"
      ]
    }
  }
}
```

Enterprise Grid Org Admin または Org Owner にアプリの承認を依頼し、組織レベルでインストールして、インストールの対象となるワークスペースを選択します。Slack が Request URL を検証した後、組織インストールの bot token と、アプリの **Basic Information -> App Credentials -> Signing Secret** をコピーします。同じ Request URL パスを使用して Enterprise アカウントを設定します。

```json5
{
  channels: {
    slack: {
      enabled: true,
      mode: "http",
      enterpriseOrgInstall: true,
      botToken: { source: "env", provider: "default", id: "SLACK_BOT_TOKEN" },
      signingSecret: {
        source: "env",
        provider: "default",
        id: "SLACK_SIGNING_SECRET",
      },
      webhookPath: "/slack/events",
      dmPolicy: "open",
      allowFrom: ["*"],
      groupPolicy: "allowlist",
      channels: {
        C0123456789: { requireMention: true },
      },
    },
  },
}
```

起動時に、OpenClaw は Slack の `auth.test` を使用して `enterpriseOrgInstall` を検証します。フラグのない組織インストール済みトークン、またはフラグのあるワークスペーストークンでは、起動に失敗します。どのワークスペースがインストールを許可したかについては、Slack が引き続き信頼できる唯一の情報源です。その後、OpenClaw は配信された各イベントに、設定済みのチャンネル、ユーザー、DM、メンションのポリシーを適用します。Enterprise V1 は、組織インストールではループ防止用の安定したワークスペース修飾付き Bot アイデンティティが提供されないため、`allowBots` に関係なく、Bot が生成したすべての `message` および `app_mention` イベントをディスパッチ前に拒否します。

Enterprise サポートは、直接の Socket Mode または HTTP の `message` および `app_mention` イベントと、それらへの即時返信に意図的に限定されています。Enterprise アカウントでは、リレーモード、スラッシュコマンド、インタラクション、App Home、リアクションイベントリスナー、ピン、Slack アクションツール、Slack ネイティブ承認、バインディング、キューまたはスケジュールによる配信、プロアクティブ送信は利用できません。外向きの確認応答、入力中、ステータスのリアクションは、リスナーが管理する Slack クライアントを通じてサポートされ、`reactions:write` が必要です。受信リアクション通知とリアクションアクションツールは引き続き利用できません。

即時返信では、チャンク、メディア、メタデータ、ID のフォールバック、リンク展開、配信確認について標準の Slack 配信動作を再利用します。ただし、検証済みのリスナー所有クライアントがアクティブなイベントターン内にある間に限ります。メモリ内の送信キューとスレッド参加レコードは、そのイベントのワークスペースごとに分割されます。クライアント自体がシリアライズまたは永続化されることはありません。

チャンネルポリシーキーと `dm.groupChannels` エントリには、加工されていない安定した Slack チャンネル ID、または `channel:<id>` 形式を使用する必要があります。OpenClaw は、どちらの形式も実行時の照合用に加工されていないチャンネル ID へ正規化します。`slack:`、`group:`、`mpim:` のプレフィックスを使用すると起動に失敗します。
ユーザーポリシーエントリには、安定した Slack ユーザー ID を使用する必要があります。名前、スラッグ、表示名、メールアドレスを使用すると起動に失敗します。ID には、Slack の正規の大文字プレフィックスと本体（例: `C0123456789` または `U0123456789`）を使用する必要があります。小文字や短い類似文字列を使用すると起動に失敗します。Enterprise アカウントでは `dangerouslyAllowNameMatching` を有効にできません。Enterprise アカウントではグローバルな `mentionPatterns.mode` を設定できますが、`mentionPatterns.allowIn` と `mentionPatterns.denyIn` は起動に失敗します。これは、単独の Slack チャンネル ID がワークスペースで修飾されておらず、複数のワークスペースで再利用される可能性があるためです。ワークスペースへのインストールでは、既存のスコープ付きメンションパターン動作が維持されます。受け入れられた各ワークスペースには、Slack ID が重複している場合でも、個別のルーティング、セッション、トランスクリプト、重複排除、履歴、キャッシュ ID が割り当てられます。`message` ストリーム内では、通常のユーザーメッセージとユーザーが作成した `file_share` イベントがサポートされます。それ以外のメッセージサブタイプは、認可またはシステムイベント処理の前に拒否されます。

Enterprise の DM は、無効にする（`dm.enabled=false` または `dmPolicy="disabled"`）か、`dmPolicy="open"` で明示的に開放し、有効なアカウントの `allowFrom` にリテラルの `"*"` を含める必要があります。空の許可リスト、または `"*"` を含まないユーザー固有 ID を設定すると起動に失敗します。ペアリングとユーザーごとの DM 許可リストは拒否されます。これは、それらの認可ストア内で Slack ユーザー ID がワークスペースによって修飾されていないためです。チャンネルメッセージには、引き続きチャンネルポリシーと送信者ポリシーが適用されます。

## インストール

```bash
openclaw plugins install @openclaw/slack
```

`plugins install` は Plugin を登録して有効にします。以下で Slack アプリとチャンネル設定を構成するまでは何も行いません。Plugin の一般的なインストール規則については、[Plugin](/ja-JP/tools/plugin)を参照してください。

## クイックセットアップ

このセクションのマニフェストは、ワークスペースをスコープとするインストールを作成します。Enterprise Grid の組織インストールでは、代わりに専用の[組織全体向けマニフェストとワークフロー](#enterprise-grid-org-wide-installs)を使用してください。

<Tabs>
  <Tab title="Socket Mode（デフォルト）">
    <Steps>
      <Step title="新しい Slack アプリを作成">
        [api.slack.com/apps](https://api.slack.com/apps/new) を開き、→ **Create New App** → **From a manifest** → ワークスペースを選択 → 以下のいずれかのマニフェストを貼り付け → **Next** → **Create** の順に進みます。

        <CodeGroup>

```json Recommended
{
  "display_information": {
    "name": "OpenClaw",
    "description": "OpenClaw 用 Slack コネクター"
  },
  "features": {
    "bot_user": { "display_name": "OpenClaw", "always_online": true },
    "app_home": {
      "home_tab_enabled": true,
      "messages_tab_enabled": true,
      "messages_tab_read_only_enabled": false
    },
    "agent_view": {
      "agent_description": "OpenClaw は Slack Agent View の会話を OpenClaw エージェントに接続します。",
      "suggested_prompts": [
        { "title": "何ができますか？", "message": "どのようなことを手伝えますか？" },
        {
          "title": "このチャンネルを要約",
          "message": "このチャンネルの最近のアクティビティを要約してください。"
        },
        { "title": "返信を下書き", "message": "返信の下書きを手伝ってください。" }
      ]
    },
    "slash_commands": [
      {
        "command": "/openclaw",
        "description": "OpenClaw にメッセージを送信",
        "should_escape": false
      }
    ]
  },
  "oauth_config": {
    "scopes": {
      "bot": [
        "app_mentions:read",
        "assistant:write",
        "channels:history",
        "channels:read",
        "chat:write",
        "commands",
        "emoji:read",
        "files:read",
        "files:write",
        "groups:history",
        "groups:read",
        "im:history",
        "im:read",
        "im:write",
        "mpim:history",
        "mpim:read",
        "mpim:write",
        "pins:read",
        "pins:write",
        "reactions:read",
        "reactions:write",
        "usergroups:read",
        "users:read"
      ]
    }
  },
  "settings": {
    "socket_mode_enabled": true,
    "event_subscriptions": {
      "bot_events": [
        "app_home_opened",
        "app_mention",
        "app_context_changed",
        "channel_rename",
        "member_joined_channel",
        "member_left_channel",
        "message.channels",
        "message.groups",
        "message.im",
        "message.mpim",
        "pin_added",
        "pin_removed",
        "reaction_added",
        "reaction_removed"
      ]
    }
  }
}
```

```json Minimal
{
  "display_information": {
    "name": "OpenClaw",
    "description": "OpenClaw 用 Slack コネクター"
  },
  "features": {
    "bot_user": { "display_name": "OpenClaw", "always_online": true },
    "app_home": {
      "home_tab_enabled": true,
      "messages_tab_enabled": true,
      "messages_tab_read_only_enabled": false
    },
    "agent_view": {
      "agent_description": "OpenClaw は Slack Agent View の会話を OpenClaw エージェントに接続します。",
      "suggested_prompts": [
        { "title": "何ができますか？", "message": "どのようなことを手伝えますか？" },
        {
          "title": "このチャンネルを要約",
          "message": "このチャンネルの最近のアクティビティを要約してください。"
        },
        { "title": "返信を下書き", "message": "返信の下書きを手伝ってください。" }
      ]
    },
    "slash_commands": [
      {
        "command": "/openclaw",
        "description": "OpenClaw にメッセージを送信",
        "should_escape": false
      }
    ]
  },
  "oauth_config": {
    "scopes": {
      "bot": [
        "app_mentions:read",
        "assistant:write",
        "channels:history",
        "channels:read",
        "chat:write",
        "commands",
        "groups:history",
        "groups:read",
        "im:history",
        "im:read",
        "im:write",
        "users:read"
      ]
    }
  },
  "settings": {
    "socket_mode_enabled": true,
    "event_subscriptions": {
      "bot_events": [
        "app_home_opened",
        "app_mention",
        "app_context_changed",
        "message.channels",
        "message.groups",
        "message.im"
      ]
    }
  }
}
```

        </CodeGroup>

        <Note>
          **推奨**は、App Home、スラッシュコマンド、ファイル、リアクション、ピン、グループ DM、絵文字とユーザーグループの読み取りを含む、Slack Plugin の完全な機能セットに対応します。ワークスペースのポリシーでスコープが制限されている場合は、**最小構成**を選択してください。これは DM、チャンネルとグループの履歴、メンション、スラッシュコマンドに対応しますが、ファイル、リアクション、ピン、グループ DM（`mpim:*`）、`emoji:read`、`usergroups:read` は含まれません。各スコープの根拠と、追加のスラッシュコマンドなどの付加オプションについては、[マニフェストとスコープのチェックリスト](#manifest-and-scope-checklist)を参照してください。
        </Note>

        Slack がアプリを作成した後:

        - **Basic Information -> App-Level Tokens -> Generate Token and Scopes**: `connections:write` を追加して保存し、App-Level Token をコピーします。
        - **Install App -> Install to Workspace**: Bot User OAuth Token をコピーします。

      </Step>

      <Step title="OpenClaw を構成">

        推奨される SecretRef 設定:

```bash
export SLACK_APP_TOKEN=slack-app-token-example
export SLACK_BOT_TOKEN=slack-bot-token-example
cat > slack.socket.patch.json5 <<'JSON5'
{
  channels: {
    slack: {
      enabled: true,
      mode: "socket",
      appToken: { source: "env", provider: "default", id: "SLACK_APP_TOKEN" },
      botToken: { source: "env", provider: "default", id: "SLACK_BOT_TOKEN" },
    },
  },
}
JSON5
openclaw config patch --file ./slack.socket.patch.json5 --dry-run
openclaw config patch --file ./slack.socket.patch.json5
```

        環境変数によるフォールバック（デフォルトアカウントのみ）:

```bash
SLACK_APP_TOKEN=slack-app-token-example
SLACK_BOT_TOKEN=slack-bot-token-example
```

      </Step>

      <Step title="Gateway を起動">

```bash
openclaw gateway
```

      </Step>
    </Steps>

  </Tab>

  <Tab title="HTTP リクエスト URL">
    <Steps>
      <Step title="新しい Slack アプリを作成">
        [api.slack.com/apps](https://api.slack.com/apps/new) を開き、→ **Create New App** → **From a manifest** → ワークスペースを選択 → 以下のいずれかのマニフェストを貼り付け → `https://gateway-host.example.com/slack/events` を公開 Gateway URL に置き換え → **Next** → **Create** の順に進みます。

        <CodeGroup>

```json Recommended
{
  "display_information": {
    "name": "OpenClaw",
    "description": "OpenClaw 用 Slack コネクター"
  },
  "features": {
    "bot_user": { "display_name": "OpenClaw", "always_online": true },
    "app_home": {
      "home_tab_enabled": true,
      "messages_tab_enabled": true,
      "messages_tab_read_only_enabled": false
    },
    "agent_view": {
      "agent_description": "OpenClaw は Slack Agent View の会話を OpenClaw エージェントに接続します。",
      "suggested_prompts": [
        { "title": "何ができますか？", "message": "どのようなことを手伝えますか？" },
        {
          "title": "このチャンネルを要約",
          "message": "このチャンネルの最近のアクティビティを要約してください。"
        },
        { "title": "返信を下書き", "message": "返信の下書きを手伝ってください。" }
      ]
    },
    "slash_commands": [
      {
        "command": "/openclaw",
        "description": "OpenClaw にメッセージを送信",
        "should_escape": false,
        "url": "https://gateway-host.example.com/slack/events"
      }
    ]
  },
  "oauth_config": {
    "scopes": {
      "bot": [
        "app_mentions:read",
        "assistant:write",
        "channels:history",
        "channels:read",
        "chat:write",
        "commands",
        "emoji:read",
        "files:read",
        "files:write",
        "groups:history",
        "groups:read",
        "im:history",
        "im:read",
        "im:write",
        "mpim:history",
        "mpim:read",
        "mpim:write",
        "pins:read",
        "pins:write",
        "reactions:read",
        "reactions:write",
        "usergroups:read",
        "users:read"
      ]
    }
  },
  "settings": {
    "event_subscriptions": {
      "request_url": "https://gateway-host.example.com/slack/events",
      "bot_events": [
        "app_home_opened",
        "app_mention",
        "app_context_changed",
        "channel_rename",
        "member_joined_channel",
        "member_left_channel",
        "message.channels",
        "message.groups",
        "message.im",
        "message.mpim",
        "pin_added",
        "pin_removed",
        "reaction_added",
        "reaction_removed"
      ]
    },
    "interactivity": {
      "is_enabled": true,
      "request_url": "https://gateway-host.example.com/slack/events",
      "message_menu_options_url": "https://gateway-host.example.com/slack/events"
    }
  }
}
```

```json Minimal
{
  "display_information": {
    "name": "OpenClaw",
    "description": "OpenClaw 用 Slack コネクタ"
  },
  "features": {
    "bot_user": { "display_name": "OpenClaw", "always_online": true },
    "app_home": {
      "home_tab_enabled": true,
      "messages_tab_enabled": true,
      "messages_tab_read_only_enabled": false
    },
    "agent_view": {
      "agent_description": "OpenClaw は Slack Agent View の会話を OpenClaw エージェントに接続します。",
      "suggested_prompts": [
        { "title": "何ができますか？", "message": "どのようなことを手伝えますか？" },
        {
          "title": "このチャンネルを要約",
          "message": "このチャンネルの最近のアクティビティを要約してください。"
        },
        { "title": "返信の下書きを作成", "message": "返信の下書き作成を手伝ってください。" }
      ]
    },
    "slash_commands": [
      {
        "command": "/openclaw",
        "description": "OpenClaw にメッセージを送信",
        "should_escape": false,
        "url": "https://gateway-host.example.com/slack/events"
      }
    ]
  },
  "oauth_config": {
    "scopes": {
      "bot": [
        "app_mentions:read",
        "assistant:write",
        "channels:history",
        "channels:read",
        "chat:write",
        "commands",
        "groups:history",
        "groups:read",
        "im:history",
        "im:read",
        "im:write",
        "users:read"
      ]
    }
  },
  "settings": {
    "event_subscriptions": {
      "request_url": "https://gateway-host.example.com/slack/events",
      "bot_events": [
        "app_home_opened",
        "app_mention",
        "app_context_changed",
        "message.channels",
        "message.groups",
        "message.im"
      ]
    },
    "interactivity": {
      "is_enabled": true,
      "request_url": "https://gateway-host.example.com/slack/events",
      "message_menu_options_url": "https://gateway-host.example.com/slack/events"
    }
  }
}
```

        </CodeGroup>

        <Note>
          **推奨**は Slack Plugin の全機能セットに対応します。**最小構成**では、制限の厳しいワークスペース向けに、ファイル、リアクション、ピン、グループ DM（`mpim:*`）、`emoji:read`、および `usergroups:read` を除外します。各スコープの理由については、[マニフェストとスコープのチェックリスト](#manifest-and-scope-checklist)を参照してください。
        </Note>

        <Info>
          3 つの URL フィールド（`slash_commands[].url`、`event_subscriptions.request_url`、および `interactivity.request_url` / `message_menu_options_url`）は、すべて同じ OpenClaw エンドポイントを指します。Slack のマニフェストスキーマでは別々の名前が必要ですが、OpenClaw はペイロードタイプに基づいてルーティングするため、単一の `webhookPath`（デフォルトは `/slack/events`）で十分です。`slash_commands[].url` のないスラッシュコマンドは、HTTP モードでは何もせずに終了します。
        </Info>

        Slack がアプリを作成した後：

        - **Basic Information → App Credentials**：リクエスト検証用に **Signing Secret** をコピーします。
        - **Install App -> Install to Workspace**：Bot User OAuth Token をコピーします。

      </Step>

      <Step title="OpenClaw を構成">

        推奨される SecretRef の設定：

```bash
export SLACK_BOT_TOKEN=slack-bot-token-example
export SLACK_SIGNING_SECRET=...
cat > slack.http.patch.json5 <<'JSON5'
{
  channels: {
    slack: {
      enabled: true,
      mode: "http",
      botToken: { source: "env", provider: "default", id: "SLACK_BOT_TOKEN" },
      signingSecret: { source: "env", provider: "default", id: "SLACK_SIGNING_SECRET" },
      webhookPath: "/slack/events",
    },
  },
}
JSON5
openclaw config patch --file ./slack.http.patch.json5 --dry-run
openclaw config patch --file ./slack.http.patch.json5
```

        <Note>
        複数アカウントの HTTP には一意の Webhook パスを使用する

        登録が衝突しないように、各アカウントに異なる `webhookPath`（デフォルトは `/slack/events`）を割り当てます。
        </Note>

      </Step>

      <Step title="Gateway を起動">

```bash
openclaw gateway
```

      </Step>
    </Steps>

  </Tab>
</Tabs>

## ユーザー ID（実在の人物として投稿）

ユーザー ID を使用すると、OpenClaw は Slack アプリを認可した人間として読み取りと投稿を行えます。`userToken` が動作主体の ID であり、付随する Slack アプリが Socket Mode または HTTP Request URL を介して Events API トラフィックを伝送します。付随アプリにボットユーザーやボットトークンは必要ありません。

付随アプリを次のように設定します：

1. **OAuth & Permissions -> User Token Scopes** で、次のユーザースコープ権限を追加します：

   - 履歴：`channels:history`、`groups:history`、`im:history`、`mpim:history`
   - 会話の検索：`channels:read`、`groups:read`、`im:read`、`mpim:read`
   - ユーザー：`users:read`
   - 投稿：`chat:write`（メッセージは認可したユーザーとして投稿されます）
   - DM の開始：`im:write`、`mpim:write`

2. **Event Subscriptions -> Subscribe to events on behalf of users** で、次のユーザーイベントを追加します。ボットイベントのリストだけには追加しないでください：

   - `message.channels`
   - `message.groups`
   - `message.im`
   - `message.mpim`

3. イベント伝送方式を 1 つ選択します：

   - **Socket Mode：**Socket Mode を有効にし、`connections:write` を持つアプリレベルトークンを作成します。`appToken` として設定します。
   - **HTTP Request URL：**Event Subscriptions で公開 OpenClaw Slack エンドポイントを指定し、**Basic Information -> App Credentials -> Signing Secret** をコピーします。`signingSecret` として設定します。

4. アプリをインストールまたは再インストールし、対象の人間として認可して、生成されたユーザー OAuth トークンを `userToken` にコピーします。

Socket Mode の設定：

```json5
{
  channels: {
    slack: {
      identity: "user",
      userToken: "<xoxp>",
      appToken: "<xapp>",
    },
  },
}
```

HTTP Request URL の設定：

```json5
{
  channels: {
    slack: {
      identity: "user",
      mode: "http",
      userToken: "<xoxp>",
      signingSecret: "<signing-secret>",
      webhookPath: "/slack/events",
    },
  },
}
```

<Warning>
  DM とグループ DM は、上記のユーザースコープのイベントサブスクリプションを介した場合にのみ機能します。ボットは人間の 1:1 DM に参加したり、既存のグループ DM に追加されたりすることはできません。付随アプリは裏側で動作する目に見えない仕組みです。他の Slack メンバーには、OpenClaw ボットではなく、認可した人間からのメッセージとして表示されます。
</Warning>

OpenClaw は、解決された人間の ID が作成したユーザースコープのメッセージイベントを自動的に破棄するため、OpenClaw が送信したメッセージによって自己返信がトリガーされることはありません。

## Socket Mode の伝送設定

OpenClaw は、Socket Mode における Slack SDK クライアントの pong タイムアウトをデフォルトで 15 秒に設定します。ワークスペースまたはホスト固有の調整が必要な場合にのみ、伝送設定を上書きしてください：

```json5
{
  channels: {
    slack: {
      mode: "socket",
      socketMode: {
        clientPingTimeout: 20000,
        serverPingTimeout: 30000,
        pingPongLoggingEnabled: false,
      },
    },
  },
}
```

これは、Slack WebSocket の pong/server-ping タイムアウトがログに記録される Socket Mode ワークスペース、またはイベントループの枯渇が既知のホストでのみ使用してください。`clientPingTimeout` は SDK がクライアント ping を送信した後の pong 待機時間で、`serverPingTimeout` は Slack サーバー ping の待機時間です。アプリのメッセージとイベントは、伝送の稼働状態を示すシグナルではなく、引き続きアプリケーション状態です。

注：

- `socketMode` は HTTP Request URL モードでは無視されます。
- 基本の `channels.slack.socketMode` 設定は、上書きされない限りすべての Slack アカウントに適用されます。アカウントごとの上書きには `channels.slack.accounts.<accountId>.socketMode` を使用します。これはオブジェクトの上書きであるため、そのアカウントで使用するすべての Socket 調整フィールドを含めてください。
- OpenClaw のデフォルト（`15000`）があるのは `clientPingTimeout` のみです。`serverPingTimeout` と `pingPongLoggingEnabled` は、設定されている場合にのみ Slack SDK に渡されます。
- Socket Mode の再起動バックオフは約 2 秒から始まり、約 30 秒が上限です。回復可能な起動、起動待機、および切断の失敗は、チャンネルが停止するまで再試行されます。無効な認証、取り消されたトークン、スコープ不足などの永続的なアカウントおよび認証情報のエラーは、無期限に再試行せず即座に失敗します。

## マニフェストとスコープのチェックリスト

基本の Slack アプリマニフェストは、Socket Mode と HTTP Request URL で共通です。異なるのは `settings` ブロック（およびスラッシュコマンドの `url`）のみです。

基本マニフェスト（Socket Mode がデフォルト）：

```json
{
  "display_information": {
    "name": "OpenClaw",
    "description": "OpenClaw 用 Slack コネクタ"
  },
  "features": {
    "bot_user": { "display_name": "OpenClaw", "always_online": true },
    "app_home": {
      "home_tab_enabled": true,
      "messages_tab_enabled": true,
      "messages_tab_read_only_enabled": false
    },
    "agent_view": {
      "agent_description": "OpenClaw は Slack Agent View の会話を OpenClaw エージェントに接続します。",
      "suggested_prompts": [
        { "title": "何ができますか？", "message": "どのようなことを手伝えますか？" },
        {
          "title": "このチャンネルを要約",
          "message": "このチャンネルの最近のアクティビティを要約してください。"
        },
        { "title": "返信の下書きを作成", "message": "返信の下書き作成を手伝ってください。" }
      ]
    },
    "slash_commands": [
      {
        "command": "/openclaw",
        "description": "OpenClaw にメッセージを送信",
        "should_escape": false
      }
    ]
  },
  "oauth_config": {
    "scopes": {
      "bot": [
        "app_mentions:read",
        "assistant:write",
        "channels:history",
        "channels:read",
        "chat:write",
        "commands",
        "emoji:read",
        "files:read",
        "files:write",
        "groups:history",
        "groups:read",
        "im:history",
        "im:read",
        "im:write",
        "mpim:history",
        "mpim:read",
        "mpim:write",
        "pins:read",
        "pins:write",
        "reactions:read",
        "reactions:write",
        "usergroups:read",
        "users:read"
      ]
    }
  },
  "settings": {
    "socket_mode_enabled": true,
    "event_subscriptions": {
      "bot_events": [
        "app_home_opened",
        "app_mention",
        "app_context_changed",
        "channel_rename",
        "member_joined_channel",
        "member_left_channel",
        "message.channels",
        "message.groups",
        "message.im",
        "message.mpim",
        "pin_added",
        "pin_removed",
        "reaction_added",
        "reaction_removed"
      ]
    }
  }
}
```

**HTTP Request URL モード**では、`settings` を HTTP バリアントに置き換え、各スラッシュコマンドに `url` を追加します。公開 URL が必要です：

```json
{
  "features": {
    "slash_commands": [
      {
        "command": "/openclaw",
        "description": "OpenClaw にメッセージを送信",
        "should_escape": false,
        "url": "https://gateway-host.example.com/slack/events"
      }
    ]
  },
  "settings": {
    "event_subscriptions": {
      "request_url": "https://gateway-host.example.com/slack/events",
      "bot_events": [
        "app_home_opened",
        "app_mention",
        "app_context_changed",
        "channel_rename",
        "member_joined_channel",
        "member_left_channel",
        "message.channels",
        "message.groups",
        "message.im",
        "message.mpim",
        "pin_added",
        "pin_removed",
        "reaction_added",
        "reaction_removed"
      ]
    },
    "interactivity": {
      "is_enabled": true,
      "request_url": "https://gateway-host.example.com/slack/events",
      "message_menu_options_url": "https://gateway-host.example.com/slack/events"
    }
  }
}
```

### マニフェストの追加設定

上記のデフォルトを拡張する各種機能を表示します。

デフォルトのマニフェストでは、Slack App Home の **Home** タブが有効になり、`app_home_opened` をサブスクライブします。ワークスペースのメンバーが Home タブを開くと、OpenClaw は `views.publish` を含む安全なデフォルトの Home ビューを公開します。会話ペイロードや非公開設定は含まれません。単一スラッシュコマンドモードが有効な場合、コマンドのヒントには `channels.slack.slashCommand.name` が使用されます。ネイティブコマンドを使用するインストール、またはスラッシュコマンドを使用しないインストールでは、このヒントは省略されます。Slack DM 用の **Messages** タブは引き続き有効です。新しいアプリでは、`features.agent_view`、`assistant:write`、および `app_context_changed` を通じて Slack Agent View を使用します。表示される各 Agent View ルートは、それぞれ独自の OpenClaw スレッドセッションにルーティングされ、Slack の順序付けされたアクティブビューエンティティは、信頼されていないコンテキストとしてのみエージェントに渡されます。

すでに `features.assistant_view` を使用している既存のアプリは、現在のマニフェストを維持できます。OpenClaw は、そのようなインストールに対して `assistant_thread_started` と `assistant_thread_context_changed` の処理を継続します。Slack では Assistant View から Agent View への移行を元に戻すことができず、移行後はユーザーによるハードリフレッシュが必要です。そのため、ワークスペース全体を移行する意図がある場合を除き、既存のアプリの `assistant_view` を置き換えないでください。

<AccordionGroup>
  <Accordion title="オプションのネイティブスラッシュコマンド">

    単一の設定済みコマンドの代わりに、複数の[ネイティブスラッシュコマンド](#commands-and-slash-behavior)を使用できますが、次の点に注意してください。

    - `/status` コマンドは予約されているため、`/status` ではなく `/agentstatus` を使用してください。
    - 1 つの Slack アプリに一度に登録できるスラッシュコマンドは最大 25 個です（Slack プラットフォームの制限）。

    OpenClaw は有効なネイティブコマンドのハンドラーを登録しますが、Slack マニフェストのエントリは引き続き管理者が管理し、実行時には同期されません。`/login` をマニフェストに手動で追加してください。以下の例では、コマンド数を 25 個に収めるため、オプションの `/side` エイリアスの代わりにこれを含めています。`/login` はどこにでも表示できますが、ペアリングコードを発行するのはプライベートチャットまたは Web UI のみです。

    既存の `features.slash_commands` セクションを、[利用可能なコマンド](/ja-JP/tools/slash-commands#command-list)のサブセットに置き換えてください。

    <Tabs>
      <Tab title="Socket Mode（デフォルト）">

```json
{
  "slash_commands": [
    {
      "command": "/new",
      "description": "新しいセッションを開始",
      "usage_hint": "[model]"
    },
    {
      "command": "/reset",
      "description": "現在のセッションをリセット"
    },
    {
      "command": "/compact",
      "description": "セッションのコンテキストを圧縮",
      "usage_hint": "[instructions]"
    },
    {
      "command": "/stop",
      "description": "現在の実行を停止"
    },
    {
      "command": "/session",
      "description": "スレッドバインディングの有効期限を管理",
      "usage_hint": "アイドル <duration|off> または最大経過時間 <duration|off>"
    },
    {
      "command": "/think",
      "description": "思考レベルを設定",
      "usage_hint": "<level>"
    },
    {
      "command": "/verbose",
      "description": "詳細出力を切り替え",
      "usage_hint": "on|off|full"
    },
    {
      "command": "/fast",
      "description": "高速モードを表示または設定",
      "usage_hint": "[status|on|off]"
    },
    {
      "command": "/reasoning",
      "description": "推論の表示を切り替え",
      "usage_hint": "[on|off|stream]"
    },
    {
      "command": "/elevated",
      "description": "昇格モードを切り替え",
      "usage_hint": "[on|off|ask|full]"
    },
    {
      "command": "/exec",
      "description": "実行のデフォルトを表示または設定",
      "usage_hint": "host=<auto|sandbox|gateway|node> security=<deny|allowlist|full> ask=<off|on-miss|always> node=<id>"
    },
    {
      "command": "/approve",
      "description": "保留中の承認リクエストを承認または拒否",
      "usage_hint": "<id> <decision>"
    },
    {
      "command": "/model",
      "description": "モデルを表示または設定",
      "usage_hint": "[name|#|status]"
    },
    {
      "command": "/models",
      "description": "プロバイダー／モデルを一覧表示",
      "usage_hint": "[provider] [page] [limit=<n>|size=<n>|all]"
    },
    {
      "command": "/help",
      "description": "短いヘルプ概要を表示"
    },
    {
      "command": "/commands",
      "description": "生成されたコマンドカタログを表示"
    },
    {
      "command": "/tools",
      "description": "現在のエージェントが今使用できるものを表示",
      "usage_hint": "[compact|verbose]"
    },
    {
      "command": "/agentstatus",
      "description": "利用可能な場合はプロバイダーの使用量／クォータを含む実行時ステータスを表示"
    },
    {
      "command": "/tasks",
      "description": "現在のセッションのアクティブ／最近のバックグラウンドタスクを一覧表示"
    },
    {
      "command": "/context",
      "description": "コンテキストがどのように構成されるかを説明",
      "usage_hint": "[list|detail|json]"
    },
    {
      "command": "/whoami",
      "description": "送信者 ID を表示"
    },
    {
      "command": "/skill",
      "description": "名前を指定してスキルを実行",
      "usage_hint": "<name> [input]"
    },
    {
      "command": "/btw",
      "description": "セッションコンテキストを変更せずに補足質問をする",
      "usage_hint": "<question>"
    },
    {
      "command": "/login",
      "description": "Codex ログインをペアリング",
      "usage_hint": "[codex|openai]"
    },
    {
      "command": "/usage",
      "description": "使用量フッターを制御、またはコスト概要を表示",
      "usage_hint": "off|tokens|full|cost"
    }
  ]
}
```

      </Tab>
      <Tab title="HTTP リクエスト URL">
        上記の Socket Mode と同じ `slash_commands` リストを使用し、すべてのエントリに `"url": "https://gateway-host.example.com/slack/events"` を追加します。例：

```json
{
  "slash_commands": [
    {
      "command": "/new",
      "description": "新しいセッションを開始",
      "usage_hint": "[model]",
      "url": "https://gateway-host.example.com/slack/events"
    },
    {
      "command": "/help",
      "description": "短いヘルプ概要を表示",
      "url": "https://gateway-host.example.com/slack/events"
    }
  ]
}
```

        リスト内のすべてのコマンドに、その `url` 値を繰り返し指定してください。

      </Tab>
    </Tabs>

  </Accordion>
  <Accordion title="オプションの作成者スコープ（書き込み操作）">
    送信メッセージでデフォルトの Slack アプリ ID ではなく、アクティブなエージェント ID（カスタムユーザー名とアイコン）を使用する場合は、`chat:write.customize` ボットスコープを追加してください。

    絵文字アイコンを使用する場合、Slack では `:emoji_name:` 構文が必要です。

  </Accordion>
  <Accordion title="オプションのユーザートークンスコープ（読み取り操作）">
    `channels.slack.userToken` を設定する場合、一般的な読み取りスコープは次のとおりです。

    - `channels:history`、`groups:history`、`im:history`、`mpim:history`
    - `channels:read`、`groups:read`、`im:read`、`mpim:read`
    - `users:read`
    - `reactions:read`
    - `pins:read`
    - `emoji:read`
    - `search:read`（Slack 検索の読み取りに依存する場合）

  </Accordion>
</AccordionGroup>

## トークンモデル

- ボット ID（デフォルト）には、Socket Mode の場合は `botToken` + `appToken`、HTTP モードの場合は `botToken` + `signingSecret` が必要です。
- ユーザー ID には、Socket Mode の場合は `userToken` + `appToken`、HTTP モードの場合は `userToken` + `signingSecret` が必要です。ボットトークンは使用しません。
- リレーモードには、`botToken` に加えて `relay.url`、`relay.authToken`、および `relay.gatewayId` が必要です。アプリトークンや署名シークレットは使用しません。
- `botToken`、`appToken`、`signingSecret`、`relay.authToken`、および `userToken` は、プレーンテキストの
  文字列または SecretRef オブジェクトを受け入れます。
- 設定内のトークンは、環境変数のフォールバックより優先されます。
- `SLACK_BOT_TOKEN`、`SLACK_APP_TOKEN`、および `SLACK_USER_TOKEN` の各環境変数フォールバックは、デフォルトアカウントにのみ適用されます。
- `userToken` のデフォルトは読み取り専用動作（`userTokenReadOnly: true`）です。

ステータススナップショットの動作：

- Slack アカウントの検査では、認証情報ごとの `*Source` フィールドと `*Status`
  フィールド（`botToken`、`appToken`、`signingSecret`、`userToken`）を追跡します。
- ステータスは `available`、`configured_unavailable`、または `missing` です。
- `configured_unavailable` は、アカウントが SecretRef
  または別の非インラインシークレットソースを通じて設定されているものの、現在のコマンド／実行時パスで
  実際の値を解決できなかったことを意味します。
- HTTP モードでは、`signingSecretStatus` が含まれます。Socket Mode では、
  ボット ID に `botTokenStatus` + `appTokenStatus`、
  ユーザー ID に `userTokenStatus` + `appTokenStatus` を使用します。

<Tip>
ボット ID では、アクションとディレクトリの読み取りにオプションのユーザートークンを優先できます。`userTokenReadOnly: false` でフォールバックが許可されていない限り、書き込みには引き続きボットトークンを使用します。`identity: "user"` では、読み取りと書き込みの両方で常に `userToken` を使用します。
</Tip>

## アクションとゲート

Slack アクションは `channels.slack.actions.*` によって制御されます。

現在の Slack ツールで利用可能なアクショングループ：

| グループ   | デフォルト |
| ---------- | ---------- |
| messages   | 有効       |
| reactions  | 有効       |
| pins       | 有効       |
| memberInfo | 有効       |
| emojiList  | 有効       |

現在の Slack メッセージアクションには、`send`、`upload-file`、`download-file`、`read`、`edit`、`delete`、`pin`、`unpin`、`list-pins`、`member-info`、および `emoji-list` が含まれます。`download-file` は、受信ファイルのプレースホルダーに表示される Slack ファイル ID を受け入れ、画像の場合は画像プレビューを、それ以外のファイルタイプの場合はローカルファイルのメタデータを返します。

## アクセス制御とルーティング

<Tabs>
  <Tab title="DM ポリシー">
    `channels.slack.dmPolicy` は DM アクセスを制御します。`channels.slack.allowFrom` は正規の DM 許可リストです。

    - `pairing`（デフォルト）
    - `allowlist`
    - `open`（`channels.slack.allowFrom` に `"*"` が含まれている必要があります）
    - `disabled`

    DM フラグ：

    - `dm.enabled`（デフォルトは true）
    - `channels.slack.allowFrom`
    - `dm.allowFrom`（レガシー）
    - `dm.groupEnabled`（グループ DM のデフォルトは false）
    - `dm.groupChannels`（オプションの MPIM 許可リスト）

    複数アカウントでの優先順位：

    - `channels.slack.accounts.default.allowFrom` は `default` アカウントにのみ適用されます。
    - 名前付きアカウントでは、独自の `allowFrom` が未設定の場合、`channels.slack.allowFrom` を継承します。
    - 名前付きアカウントは `channels.slack.accounts.default.allowFrom` を継承しません。

    レガシーの `channels.slack.dm.policy` と `channels.slack.dm.allowFrom` は、互換性のため引き続き読み取られます。アクセスを変更せずに実行できる場合、`openclaw doctor --fix` はそれらを `dmPolicy` と `allowFrom` に移行します。

    DM でのペアリングには `openclaw pairing approve slack <code>` を使用します。

  </Tab>

  <Tab title="チャンネルポリシー">
    `channels.slack.groupPolicy` はチャンネルの処理を制御します。

    - `open`
    - `allowlist`
    - `disabled`

    チャンネル許可リストは `channels.slack.channels` 配下にあり、設定キーとして**安定した Slack チャンネル ID を使用する必要があります**（例：`C12345678`）。

    実行時の注意：`channels.slack` が完全に欠落している場合（環境変数のみのセットアップ）、実行時は `groupPolicy="allowlist"` にフォールバックし、警告をログに記録します（`channels.defaults.groupPolicy` が設定されている場合でも同様です）。

    名前／ID の解決：

    - チャンネル許可リストと DM 許可リストのエントリは、トークンアクセスで可能な場合、起動時に解決されます
    - 解決されなかったチャンネル名のエントリは設定どおり保持されますが、デフォルトではルーティングで無視されます
    - 受信認可とチャンネルルーティングでは、デフォルトで ID が優先されます。ユーザー名／スラッグの直接照合には `channels.slack.dangerouslyAllowNameMatching: true` が必要です

    <Warning>
    名前ベースのキー（`#channel-name` または `channel-name`）は、`groupPolicy: "allowlist"` では一致しません。チャンネル検索はデフォルトで ID を優先するため、名前ベースのキーではルーティングに成功することはなく、そのチャンネル内のすべてのメッセージが通知なしにブロックされます。これは、ルーティングにチャンネルキーが必須ではなく、名前ベースのキーが機能しているように見える `groupPolicy: "open"` とは異なります。

    キーには必ず Slack チャンネル ID を使用してください。確認するには、Slack でチャンネルを右クリック → **Copy link** — URL の末尾に ID（`C...`）が表示されます。

    正しい例：

    ```json5
    {
      channels: {
        slack: {
          groupPolicy: "allowlist",
          channels: {
            C12345678: { enabled: true, requireMention: true },
          },
        },
      },
    }
    ```

    誤った例（`groupPolicy: "allowlist"` では通知なしにブロックされます）：

    ```json5
    {
      channels: {
        slack: {
          groupPolicy: "allowlist",
          channels: {
            "#eng-my-channel": { enabled: true, requireMention: true },
          },
        },
      },
    }
    ```
    </Warning>

  </Tab>

  <Tab title="メンションとチャンネルユーザー">
    チャンネルメッセージは、デフォルトでメンションが必要です。

    メンション元：

    - 明示的なアプリメンション（`<@botId>`）
    - ボットユーザーがそのユーザーグループのメンバーである場合の Slack ユーザーグループメンション（`<!subteam^S...>`）。`usergroups:read` が必要
    - メンション正規表現パターン（`agents.entries.*.groupChat.mentionPatterns`、フォールバックは `messages.groupChat.mentionPatterns`）
    - ボット自身の Slack メッセージへの返信（`implicitMentions.replyToBot`）
    - ボットが参加したスレッドでの後続メッセージ（`implicitMentions.threadParticipation`）

    チャンネルごとの制御（`channels.slack.channels.<id>`。名前は起動時の解決または `dangerouslyAllowNameMatching` でのみ使用可能）：

    - `requireMention`
    - `ignoreOtherMentions`
    - `replyToMode`（`off|first|all|batched`。このチャンネルについて、アカウントまたはチャット種別の返信モードを上書き）
    - `users`（許可リスト）
    - `allowBots`
    - `skills`
    - `systemPrompt`
    - `tools`、`toolsBySender`
    - `toolsBySender` のキー形式：`channel:`、`id:`、`e164:`、`username:`、`name:`、または `"*"` ワイルドカード
      （従来のプレフィックスなしキーは、引き続き `id:` のみにマッピングされます）

    `ignoreOtherMentions`（デフォルトは `false`）は、別のユーザーまたはユーザーグループにメンションしているものの、このボットにはメンションしていないチャンネルメッセージを破棄します。DM とグループ DM（MPIM）は影響を受けません。このフィルターには、`auth.test` から解決されたボットユーザー ID が必要です。その ID を利用できない場合（たとえば、ユーザートークンのみの ID）、ゲートはフェイルオープンとなり、メッセージは変更されずに通過します。

    `allowBots` は、チャンネルとプライベートチャンネルに対して保守的に動作します。ボットが作成したルームメッセージは、送信ボットがそのルームの `users` 許可リストに明示的に記載されている場合、または `channels.slack.allowFrom` の明示的な Slack 所有者 ID のうち少なくとも 1 つが現在ルームのメンバーである場合にのみ受け入れられます。ワイルドカードおよび表示名による所有者エントリは、所有者の在室条件を満たしません。所有者の在室確認には Slack の `conversations.members` を使用します。アプリに、ルーム種別に対応する読み取りスコープ（パブリックチャンネルでは `channels:read`、プライベートチャンネルでは `groups:read`）があることを確認してください。メンバー検索に失敗した場合、OpenClaw はボットが作成したルームメッセージを破棄します。

    受け入れられた、ボットが作成した Slack メッセージには、共有の[ボットループ保護](/ja-JP/channels/bot-loop-protection)が適用されます。デフォルトの上限には `channels.defaults.botLoopProtection` を設定し、ワークスペースまたはチャンネルで異なる上限が必要な場合は、`channels.slack.botLoopProtection` または `channels.slack.channels.<id>.botLoopProtection` で上書きします。

  </Tab>
</Tabs>

## スレッド、セッション、返信タグ

- DM は `direct`、チャンネルは `channel`、MPIM は `group` としてルーティングされます。
- Slack ルートバインディングは、生のピア ID に加え、`channel:C12345678`、`user:U12345678`、`<@U12345678>` などの Slack ターゲット形式を受け付けます。
- デフォルトの `session.dmScope=main` では、通常の Slack DM はエージェントのメインセッションに統合されます。Agent View のルートと既存の Assistant View スレッドは、`:thread:<threadTs>` セッションとして分離されたままです。
- チャンネルセッション：`agent:<agentId>:slack:channel:<channelId>`。
- 通常のトップレベルのチャンネルメッセージは、`replyToMode` が `off` 以外の場合でも、チャンネルごとのセッションに残ります。
- Slack チャンネル、MPIM、Agent View、Assistant View のスレッド返信では、親 Slack の `thread_ts` がセッションのサフィックス（`:thread:<threadTs>`）に使用されます。通常の DM 返信スレッドは、ベースの DM セッション上の UI 機能として残ります。
- OpenClaw は、表示される Slack スレッドを開始すると見込まれる対象のトップレベルチャンネルルートを `agent:<agentId>:slack:channel:<channelId>:thread:<rootTs>` にシードし、ルートとその後のスレッド返信が同じ OpenClaw セッションを共有するようにします。これは、`app_mention` イベント、明示的なボットメンションまたは設定済みメンションパターンとの一致、および `replyToMode` が `off` 以外に設定された `requireMention: false` チャンネルに適用されます。
- `channels.slack.thread.historyScope` のデフォルトは `thread`、`thread.inheritParent` のデフォルトは `false` です。
- `channels.slack.thread.initialHistoryLimit` は、新しいスレッドセッションの開始時に取得する既存スレッドメッセージ数を制御します（デフォルトは `20`。無効にするには `0` を設定）。
- `channels.slack.implicitMentions.replyToBot` は、ボット自身のメッセージへの返信がメンションゲートを回避するかどうかを制御します（デフォルトは `true`）。
- `channels.slack.implicitMentions.threadParticipation` は、ボットが返信したスレッドでの後続メッセージがメンションゲートを回避するかどうかを制御します（デフォルトは `true`）。これらの後続メッセージで新たな明示的メンションを必須にするには、`false` に設定します。`openclaw doctor --fix` は、以前の `channels.slack.thread.requireExplicitMention` キーを、この肯定形の正規フラグへ移行します。
- アカウントの上書き設定は `channels.slack.accounts.<id>.implicitMentions`、共有デフォルトは `channels.defaults.implicitMentions` にあります。

返信スレッドの制御：

- `channels.slack.channels.<id>.replyToMode`：Slack チャンネル／プライベートチャンネルメッセージに対するチャンネルごとの上書き
- `channels.slack.replyToMode`：`off|first|all|batched`（デフォルトは `off`）
- `channels.slack.replyToModeByChatType`：`direct|group|channel` ごと
- ダイレクトチャット用の従来のフォールバック：`channels.slack.dm.replyToMode`

手動返信タグがサポートされています：

- `[[reply_to_current]]`
- `[[reply_to:<id>]]`

`message` ツールから Slack スレッドへ明示的に返信する場合、`action: "send"` と `threadId`、または `replyTo` とともに `replyBroadcast: true` を設定すると、Slack にスレッド返信を親チャンネルにもブロードキャストするよう要求できます。これは Slack の `chat.postMessage` `reply_broadcast` フラグにマッピングされ、テキストまたは Block Kit の送信でのみサポートされます。メディアアップロードではサポートされません。

`message` ツール呼び出しが Slack スレッド内で実行され、同じチャンネルを対象とする場合、OpenClaw は通常、有効なアカウント、チャット種別、またはチャンネルごとの `replyToMode` に従って、現在の Slack スレッドを継承します。自動返信と、同じチャンネルへの `send` または `upload-file` 呼び出しにも、同じチャンネルごとの上書きが使用されます。代わりに親チャンネルへ新しいメッセージを強制するには、`action: "send"` または `action: "upload-file"` に `topLevel: true` を設定します。`threadId: null` も、同じトップレベルへのオプトアウトとして受け付けられます。

<Note>
`replyToMode="off"` は、明示的な `[[reply_to_*]]` タグを含む、任意の送信 Slack 返信スレッド化を無効にします。Agent View と Assistant View は Slack が管理するスレッド型のエクスペリエンスであるため、この設定にかかわらず、返信とステータスは表示されるルートに残ります。その他の受信 Slack スレッドセッションをフラット化することはありません。これは、`"off"` モードでも明示的なタグが引き続き適用される Telegram とは異なります。Slack スレッドではメッセージがチャンネルから非表示になりますが、Telegram の返信はインラインで表示されたままです。
</Note>

## 確認リアクション

`ackReaction` は、OpenClaw が受信メッセージを処理している間、確認用の絵文字を送信します。`ackReactionScope` は、その絵文字を実際に送信するタイミングを決定します。

デフォルトでは、Slack ネイティブのエージェント／アシスタントスレッドステータスがローテーションする読み込みメッセージで進捗を示す間、確認リアクションは固定されたままです。キュー待ち／思考中／ツール／完了／エラーのリアクションライフサイクルを有効にするには、`messages.statusReactions.enabled: true` を設定します。

### 絵文字（`ackReaction`）

解決順序：

- `channels.slack.accounts.<accountId>.ackReaction`
- `channels.slack.ackReaction`
- `messages.ackReaction`
- エージェント ID の絵文字フォールバック（`agents.entries.*.identity.emoji`。なければ `"eyes"` / 👀）

注：

- Slack ではショートコード（たとえば `"eyes"`）が必要です。
- Slack アカウントまたはグローバルでリアクションを無効にするには、`""` を使用します。

### スコープ（`messages.ackReactionScope`）

Slack プロバイダーは `messages.ackReactionScope`（デフォルトは `"group-mentions"`）からスコープを読み取ります。現在、Slack アカウント単位または Slack チャンネル単位の上書きはありません。この値は Gateway 全体に適用されます。

値：

- `"all"`：アンビエントなルームイベントを含む、DM とグループでリアクションします。
- `"direct"`：DM でのみリアクションします。
- `"group-all"`：アンビエントなルームイベントを除く、すべてのグループメッセージでリアクションします（DM は対象外）。
- `"group-mentions"`（デフォルト）：グループ内で、ボットがメンションされた場合にのみリアクションします（またはオプトインしたグループのメンション可能対象の場合）。**DM は対象外です。**
- `"off"` / `"none"`：リアクションしません。

<Note>
デフォルトのスコープ（`"group-mentions"`）では、ダイレクトメッセージまたはアンビエントなルームイベントで確認リアクションは実行されません。受信 Slack DM と通知のないルームイベントで、設定済みの `ackReaction`（たとえば `"eyes"`）を表示するには、`messages.ackReactionScope` を `"all"` に設定します。`messages.ackReactionScope` は Slack プロバイダーの起動時に読み取られるため、変更を反映するには Gateway の再起動が必要です。
</Note>

```json5
{
  messages: {
    ackReaction: "eyes",
    ackReactionScope: "all", // DM とグループでリアクションする
  },
}
```

## テキストストリーミング

`channels.slack.streaming` はライブプレビューの動作を制御します：

- `off`：ライブプレビューのストリーミングを無効にします。
- `partial`（デフォルト）：プレビューテキストを最新の部分出力に置き換えます。
- `block`：分割されたプレビュー更新を追加します。
- `progress`：生成中は進捗ステータステキストを表示し、その後、最終テキストを送信します。
- `streaming.preview.toolProgress`：下書きプレビューが有効な場合、ツール／進捗の更新を、同じ編集対象のプレビューメッセージにルーティングします（デフォルト：`true`）。ツール／進捗メッセージを分けて維持するには、`false` を設定します。
- `streaming.preview.commandText` / `streaming.progress.commandText`：生のコマンド／実行テキストを非表示にしながら、コンパクトなツール進捗行を維持するには、`status` に設定します（デフォルト：`raw`）。

コンパクトな進捗行を維持しながら、生のコマンド／実行テキストを非表示にする：

```json
{
  "channels": {
    "slack": {
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

`channels.slack.streaming.nativeTransport` は、`channels.slack.streaming.mode` が `partial` の場合に Slack ネイティブのテキストストリーミングを制御します（デフォルト：`true`）。

Slack ネイティブの進捗タスクカードは、進捗モードでオプトインする必要があります。作業中に Slack ネイティブの計画／タスクカードを送信し、完了時に同じタスクカードを更新するには、`channels.slack.streaming.mode="progress"` とともに `channels.slack.streaming.progress.nativeTaskCards` を `true` に設定します。このフラグがない場合、進捗モードでは移植可能な下書きプレビューの動作が維持されます。

- ネイティブテキストストリーミングと Slack アシスタントのスレッドステータスを表示するには、返信スレッドが利用可能である必要があります。スレッドの選択は引き続き `replyToMode` に従います。
- ネイティブストリーミングが利用できない場合や返信スレッドが存在しない場合でも、チャンネル、グループチャット、およびトップレベルの DM ルートでは通常の下書きプレビューを使用できます。
- トップレベルの Slack DM はデフォルトでスレッド外のままであるため、Slack のスレッド形式のネイティブストリーム／ステータスプレビューは表示されません。代わりに、OpenClaw が DM 内で下書きプレビューを投稿および編集します。
- メディアおよびテキスト以外のペイロードは通常の配信にフォールバックします。
- メディア／エラーの最終ペイロードは保留中のプレビュー編集をキャンセルします。対象となるテキスト／ブロックの最終ペイロードは、プレビューをその場で編集できる場合にのみフラッシュされます。
- 返信の途中でストリーミングに失敗した場合、OpenClaw は残りのペイロードを通常の配信にフォールバックします。

Slack ネイティブテキストストリーミングの代わりに下書きプレビューを使用するには、次のようにします。

```json5
{
  channels: {
    slack: {
      streaming: {
        mode: "partial",
        nativeTransport: false,
      },
    },
  },
}
```

Slack ネイティブの進捗タスクカードを有効にするには、次のようにします。

```json5
{
  channels: {
    slack: {
      streaming: {
        mode: "progress",
        progress: {
          nativeTaskCards: true,
          render: "rich",
        },
      },
    },
  },
}
```

レガシーキー：

- `channels.slack.streamMode`（`replace | status_final | append`）は `channels.slack.streaming.mode` のレガシーエイリアスです。
- ブール値 `channels.slack.streaming` は `channels.slack.streaming.mode` および `channels.slack.streaming.nativeTransport` のレガシーエイリアスです。
- トップレベルの `channels.slack.chunkMode` および `channels.slack.nativeStreaming` は `channels.slack.streaming.chunkMode` および `channels.slack.streaming.nativeTransport` のレガシーエイリアスです。
- レガシーエイリアスは実行時には読み込まれません。`openclaw doctor --fix` を実行して、永続化された Slack ストリーミング設定を正規キーに書き換えてください。

## 入力中リアクションのフォールバック

`typingReaction` は、OpenClaw が返信を処理している間、受信した Slack メッセージに一時的なリアクションを追加し、実行が終了すると削除します。これは、デフォルトの「入力中...」ステータスインジケーターを使用するスレッド返信以外で特に役立ちます。

解決順序：

- `channels.slack.accounts.<accountId>.typingReaction`
- `channels.slack.typingReaction`

注：

- Slack ではショートコードが必要です（例：`"hourglass_flowing_sand"`）。
- リアクションはベストエフォートであり、返信または失敗処理の完了後に自動的にクリーンアップが試行されます。

## 音声入力

現在 Slack で OpenClaw に話しかけるには、OpenClaw アプリに Slack オーディオクリップを送信します。Slackbot の音声入力マイクは Slack が所有する別機能であり、アプリ API ではありません。

- **[Slackbot の音声入力](https://slack.com/help/articles/202026038-How-to-use-Slackbot)**は、ユーザーの非公開 Slackbot 会話内で動作します。Slack は録音を Slackbot のプロンプトに変換しますが、Events API を通じてサードパーティの Slack アプリにオーディオファイル、音声入力イベント、プロンプト、入力ソースマーカーを送出することはありません。OpenClaw Slack plugin はこれを有効化したり受信したりできません。
- **[Slack オーディオクリップ](https://slack.com/help/articles/4406235165587-Record-audio-and-video-clips-in-Slack)**は、OpenClaw の DM、チャンネル、またはスレッドに投稿できる Slack の保存済みファイルです。OpenClaw はアクセス可能なクリップをボットトークンでダウンロードし、Slack のクリップ MIME メタデータを正規化して、共有の[音声文字起こしパイプライン](/ja-JP/nodes/audio)に送信します。推奨アプリマニフェストには、必要な `files:read` スコープが含まれています。

オーディオクリップと Slackbot の音声入力では、プライバシー上の意味が異なります。クリップには Slack のファイル保持ポリシーが適用され、OpenClaw が文字起こしのためにダウンロードします。一方、Slack によると音声入力の音声は保存されません。

`requireMention: true` が設定されたチャンネルでは、キャプションのないオーディオクリップでも、設定済みのメンションパターン（`agents.entries.*.groupChat.mentionPatterns`、フォールバックは `messages.groupChat.mentionPatterns`）を発話することでゲートを満たせます。OpenClaw はクリップをダウンロードまたは文字起こしする前に送信者を認可し、文字起こしが一致した場合にのみ受け入れます。失敗した、または一致しなかった推測的な文字起こしは、ダウンロードしたクリップとともに破棄され、チャンネル履歴には保持されません。Slack ネイティブの `@bot` ID は音声から推測できないため、発話名のパターンを設定するか、入力したメンションを含めてください。文字起こしのエコーが有効な場合、エコーは受け入れ後にのみ送信されます。

## メディア、分割、および配信

<AccordionGroup>
  <Accordion title="受信添付ファイル">
    Slack の添付ファイルは、Slack がホストする非公開 URL からダウンロードされ（トークン認証済みリクエストフロー）、取得に成功しサイズ制限内であればメディアストアに書き込まれます。ファイルプレースホルダーには Slack の `fileId` が含まれるため、エージェントは `download-file` を使用して元のファイルを取得できます。

    ダウンロードには、上限付きのアイドルタイムアウトと合計タイムアウトが適用されます。Slack のファイル取得が停止または失敗した場合でも、OpenClaw はメッセージの処理を継続し、ファイルプレースホルダーにフォールバックします。

    実行時の受信サイズ上限は、`channels.slack.mediaMaxMb` で上書きしない限り、デフォルトで `20MB` です。

  </Accordion>

  <Accordion title="送信テキストとファイル">
    - テキストチャンクは `channels.slack.textChunkLimit` を使用します（デフォルトは `8000`、Slack 自体のメッセージ長制限が上限です）
    - `channels.slack.streaming.chunkMode="newline"` は段落優先の分割を有効にします
    - ファイル送信は Slack アップロード API を使用し、スレッド返信（`thread_ts`）を含めることができます
    - 長いファイルキャプションでは、Slack に安全な最初のテキストチャンクをアップロードコメントとして使用し、残りのチャンクを後続メッセージとして送信します
    - 送信メディアの上限は、設定されている場合は `channels.slack.mediaMaxMb` に従います。それ以外の場合、チャンネル送信ではメディアパイプラインの MIME 種別ごとのデフォルトが使用されます

  </Accordion>

  <Accordion title="配信先">
    推奨される明示的な送信先：

    - DM には `user:<id>`
    - チャンネルには `channel:<id>`

    テキスト／ブロックのみの Slack DM はユーザー ID に直接投稿できます。ファイルアップロードとスレッド送信では具体的な会話 ID が必要になるため、最初に Slack 会話 API で DM を開きます。

  </Accordion>
</AccordionGroup>

## コマンドとスラッシュ動作

スラッシュコマンドは、単一の設定済みコマンドまたは複数のネイティブコマンドとして Slack に表示されます。コマンドのデフォルトを変更するには、`channels.slack.slashCommand` を設定します。

- `enabled: false`
- `name: "openclaw"`
- `sessionPrefix: "slack:slash"`
- `ephemeral: true`

```txt
/openclaw /help
```

ネイティブコマンドには Slack アプリの[追加のマニフェスト設定](#additional-manifest-settings)が必要です。代わりにグローバル設定で `channels.slack.commands.native: true` または `commands.native: true` を使用して有効にします。

- Slack ではネイティブコマンドの自動モードが**オフ**であるため、`commands.native: "auto"` では Slack ネイティブコマンドは有効になりません。

```txt
/help
```

ネイティブ引数メニューは、優先順位に従って次のいずれかとしてレンダリングされます。

- 十分に短いオプションが 3～5 個：オーバーフロー（「...」）メニュー
- オプションが 100 個を超え、非同期オプションフィルタリングが利用可能：外部選択
- オプションが 1～2 個、または選択肢として使用するにはエンコード値が長すぎるオプションがある場合：ボタンブロック
- それ以外（オプションが 6～100 個、または非同期フィルタリングなしで 100 個超）：静的選択メニュー（メニューごとに 100 オプション単位で分割）

```txt
/think
```

スラッシュセッションは `agent:<agentId>:slack:slash:<userId>` のような分離されたキーを使用し、引き続き `CommandTargetSessionKey` を使用してコマンド実行を対象会話セッションにルーティングします。

## ネイティブチャート

Slack の公開 [`data_visualization` Block Kit ブロック](https://docs.slack.dev/reference/block-kit/blocks/data-visualization-block/)は、
メッセージ内に折れ線、棒、面、および円グラフをレンダリングします。OpenClaw は移植可能な
`presentation` `chart` ブロックをそのネイティブ形式にマッピングします。通常の
`chat:write` メッセージアクセス以外に、追加の OAuth スコープ、
ファイルアップロード、画像レンダラー、Slack 設定は必要ありません。

```json
{
  "blocks": [
    {
      "type": "chart",
      "chartType": "bar",
      "title": "四半期売上高",
      "categories": ["Q1", "Q2"],
      "series": [{ "name": "売上高", "values": [120, 145] }],
      "xLabel": "四半期"
    }
  ]
}
```

Slack の制限はネイティブレンダリングの前に適用されます。

- タイトルおよび任意の軸ラベル：50 文字
- 円グラフ：1～12 個の正のセグメント
- 折れ線／棒／面グラフ：一意の名前を持つ 1～12 個の系列と、共有される 1～20 個のカテゴリ
- セグメント、カテゴリ、および系列のラベル：20 文字
- すべての系列には、各カテゴリに対して 1 つの有限値が必要です。円グラフ以外の値は
  負でもかまいません

すべてのネイティブチャートには、スクリーンリーダー、通知、セッションミラーリング、および
ブロックをレンダリングできないクライアント向けに、トップレベルのテキスト表現も含まれます。
他の OpenClaw チャンネルへの標準プレゼンテーション送信では、ネイティブチャート対応を通知していない限り、
同じ決定的なチャートデータがテキストとして送信されます。段階的ロールアウト中に Slack が
`invalid_blocks` でチャートを拒否した場合、OpenClaw は拒否されたネイティブデータブロックを
削除し、同階層のコントロールを保持したうえで、完全なチャート表現を表示テキストとして送信します。

Slack は現在、メッセージごとに最大 2 個の `data_visualization` ブロックを受け付けます。
プレゼンテーションに 2 個を超える有効なチャートが含まれる場合、OpenClaw はその順序を維持し、
後続メッセージでネイティブレンダリングを続行します。各メッセージに含まれるチャートは
2 個以下です。

Slack の[開発者向けリリース](https://docs.slack.dev/changelog/2026/06/16/block-kit-data-visualization-block/)では、
このブロックをアプリ向けの Block Kit 機能として説明しており、有料プランの制限は公開されていません。
Business+/Enterprise の利用資格に関する記述は Slackbot の自動 AI チャート生成に適用され、
構造化済みの Block Kit チャートをアプリが送信する場合とは別です。チャートはメッセージ専用ブロックであり、
App Home、モーダル、Canvas のコンテンツではありません。

## ネイティブテーブル

Slack の現在の [`data_table` Block Kit ブロック](https://docs.slack.dev/reference/block-kit/blocks/data-table-block/)は、
メッセージ内に構造化された行と列をレンダリングします。OpenClaw は明示的な移植可能
`presentation` `table` ブロックを `data_table` にマッピングします。Slack の
レガシー [`table` ブロック](https://docs.slack.dev/reference/block-kit/blocks/table-block/)は使用しません。
通常の `chat:write` メッセージアクセス以外に、追加の OAuth スコープや
Slack 設定は必要ありません。

```json
{
  "blocks": [
    {
      "type": "table",
      "caption": "進行中のパイプライン",
      "headers": ["アカウント", "ステージ", "ARR"],
      "rows": [
        ["Acme", "獲得済み", 125000],
        ["Globex", "レビュー", 82000]
      ],
      "rowHeaderColumnIndex": 0
    }
  ]
}
```

OpenClaw はヘッダーと文字列セルを Slack の `raw_text` セルにマッピングします。数値セルは
`raw_number` にマッピングされ、ネイティブの並べ替えとフィルタリングのために有限の数値が保持されます。
`rowHeaderColumnIndex` が存在する場合、そのゼロ始まりの列を
Slack の行ヘッダーとして指定します。

Slack が公開している `data_table` の制限は、ネイティブレンダリングの前に適用されます。

- 1～20 列
- 1～100 データ行、およびヘッダー行
- すべての行でセル数が同一
- 1 つのメッセージ内の全テーブルセルを合計して最大 10,000 文字

メッセージが合計文字数制限内に収まる限り、複数の有効なテーブルブロックを
ネイティブにレンダリングできます。ネイティブの範囲内でレンダリングできないテーブルは、
行やセルを失うことなく、完全で決定的なテキストになります。そのテキストが Slack の
1 メッセージ分を超える場合、送信とスラッシュ応答では順序付きテキストチャンクを使用します。
テーブルの編集時には、既存メッセージから行を暗黙に切り捨てる代わりに、
明示的なサイズエラーが発生します。

ポータブルプレゼンテーションから生成されたすべてのネイティブテーブルには、スクリーンリーダー、通知、セッションミラーリング、およびブロックをレンダリングできないクライアント向けに、トップレベルのテキスト表現も含まれます。フォールバックではチャートとテーブルの生の値がリテラルのまま保持されるため、`<@U123>` のようなセルデータが Slack のメンションになることはありません。
Slack が `invalid_blocks` でネイティブのチャートまたはテーブルブロックを拒否した場合、OpenClaw は範囲を限定した1回の復旧ステップですべてのネイティブデータブロックを削除し、ボタンや選択メニューなどの有効な兄弟ブロックを保持したうえで、Slack の書式設定を無効にして、完全に表示可能なチャートとテーブルのテキストを送信します。スラッシュコマンドの配信では、コマンド全体を通して Slack の5回の呼び出しという `response_url` 予算を追跡します。各返信バッチの前に、残りの呼び出し回数に収まる完全なプランを選択し、収まらない場合はそのバッチを投稿する前に失敗します。

ネイティブテーブルに昇格されるのは、明示的な `presentation` テーブルブロックだけです。
Markdown のパイプテーブルは作成時のテキストのまま保持され、OpenClaw がテーブル構造やセル型を推測することはありません。既存の信頼済み Slack ネイティブ生成元は、引き続き生のブロックを `channelData.slack.blocks` 経由で渡せます。OpenClaw は有効な生の `data_table` セルからフォールバックテキストを生成しますが、不正なカスタムブロックはキャプションまたは一般的な Block Kit フォールバックに縮退する場合があります。ポータブルなエージェント、CLI、および Plugin の出力では `presentation` を使用してください。

## インタラクティブな返信

Slack はエージェントが作成したインタラクティブな返信コントロールをレンダリングできますが、この機能はデフォルトで無効になっています。
新しいエージェント、CLI、および Plugin の出力では、共有の
`presentation` ボタンまたは選択ブロックを優先してください。これらは同じ Slack インタラクション経路を使用し、他のチャンネルでも縮退表示できます。

グローバルに有効にするには、次のように設定します。

```json5
{
  channels: {
    slack: {
      capabilities: {
        interactiveReplies: true,
      },
    },
  },
}
```

特定の Slack アカウントだけで有効にすることもできます。

```json5
{
  channels: {
    slack: {
      accounts: {
        ops: {
          capabilities: {
            interactiveReplies: true,
          },
        },
      },
    },
  },
}
```

有効にすると、エージェントは非推奨の Slack 専用返信ディレクティブも引き続き出力できます。

- `[[slack_buttons: Approve:approve, Reject:reject]]`
- `[[slack_select: Choose a target | Canary:canary, Production:production]]`

これらのディレクティブは Slack Block Kit にコンパイルされ、クリックまたは選択を既存の Slack インタラクションイベント経路へ戻します。古いプロンプトと Slack 固有のエスケープハッチ用に維持し、新しいポータブルコントロールには共有プレゼンテーションを使用してください。

ディレクティブコンパイラ API も、新しい生成元コードでは非推奨です。

- `compileSlackInteractiveReplies(...)`
- `parseSlackOptionsLine(...)`
- `isSlackInteractiveRepliesEnabled(...)`
- `buildSlackInteractiveBlocks(...)`

新しい Slack レンダリングコントロールには、`presentation` ペイロードと `buildSlackPresentationBlocks(...)` を使用してください。

注:

- これは Slack 固有のレガシー UI です。他のチャンネルは Slack Block
  Kit ディレクティブを独自のボタンシステムに変換しません。
- インタラクティブコールバックの値は、エージェントが直接作成した生の値ではなく、OpenClaw が生成した不透明なトークンです。
- 生成されたインタラクティブブロックが Slack Block Kit の上限を超える場合、OpenClaw は無効なブロックペイロードを送信せず、元のテキスト返信にフォールバックします。

### Plugin が所有するモーダル送信

インタラクティブハンドラーを登録する Slack Plugin は、OpenClaw がペイロードをエージェントに表示されるシステムイベント用に圧縮する前に、モーダルの
`view_submission` および `view_closed` ライフサイクルイベントも受信できます。Slack モーダルを開く際は、次のいずれかのルーティングパターンを使用してください。

- `callback_id` を `openclaw:<namespace>:<payload>` に設定します。
- または、既存の `callback_id` を維持し、モーダルの `private_metadata` に `pluginInteractiveData:
"<namespace>:<payload>"` を配置します。

ハンドラーは、`view_submission` または
`view_closed` としての `ctx.interaction.kind`、正規化された `inputs`、および Slack からの完全な生の `stateValues` オブジェクトを受信します。Plugin ハンドラーの呼び出しにはコールバック ID のみのルーティングで十分です。モーダルからエージェントに表示されるシステムイベントも生成する場合は、既存のモーダルの `private_metadata` ユーザー／セッションルーティングフィールドを含めてください。エージェントは、圧縮および秘匿化された `Slack interaction: ...` システムイベントを受信します。ハンドラーが
`systemEvent.summary`、`systemEvent.reference`、または `systemEvent.data` を返す場合、それらのフィールドが圧縮イベントに含まれるため、エージェントは完全なフォームペイロードを見ることなく、Plugin が所有するストレージを参照できます。

## Slack のネイティブ承認

Slack は Web UI やターミナルへのフォールバックではなく、インタラクティブなボタンとインタラクションを備えたネイティブ承認クライアントとして機能できます。

- 実行および Plugin の承認は、Slack ネイティブの Block Kit プロンプトとしてレンダリングできます。
- `channels.slack.execApprovals.*` は引き続き、ネイティブ実行承認クライアントの有効化と DM／チャンネルルーティングの設定です。
- 実行承認の DM は `channels.slack.execApprovals.approvers` または `commands.ownerAllowFrom` を使用します。
- 発生元セッションで Slack がネイティブ承認クライアントとして有効になっている場合、または `approvals.plugin` が発生元の Slack セッションか Slack ターゲットへルーティングする場合、Plugin の承認には Slack ネイティブボタンが使用されます。
- Plugin 承認の DM には、`channels.slack.allowFrom`、名前付きアカウントの `allowFrom`、またはアカウントのデフォルトルートから取得した Slack Plugin 承認者が使用されます。
- 承認者の認可は引き続き適用されます。実行専用の承認者は、Plugin 承認者でもない限り、Plugin リクエストを承認できません。

これは他のチャンネルと同じ共有承認ボタン画面を使用します。Slack アプリ設定で `interactivity` が有効な場合、承認プロンプトは会話内に直接 Block Kit ボタンとしてレンダリングされます。
これらのボタンが表示されている場合、それが主要な承認 UX です。ツールの結果でチャット承認が利用できないことが示される場合、または手動承認が唯一の経路である場合にのみ、OpenClaw は手動の `/approve` コマンドを含める必要があります。

設定パス:

- `channels.slack.execApprovals.enabled`
- `channels.slack.execApprovals.approvers`（省略可能。可能な場合は `commands.ownerAllowFrom` にフォールバック）
- `channels.slack.execApprovals.target`（`dm` | `channel` | `both`、デフォルト: `dm`）
- `agentFilter`、`sessionFilter`

`enabled` が未設定または `"auto"` で、少なくとも1人の実行承認者を解決できる場合、Slack はネイティブ実行承認を自動的に有効にします。Slack Plugin 承認者を解決でき、リクエストがネイティブクライアントのフィルターに一致する場合、Slack はこのネイティブクライアント経路を通じてネイティブ Plugin 承認も処理できます。Slack をネイティブ承認クライアントとして明示的に無効にするには、`enabled: false` を設定します。承認者を解決できる場合にネイティブ承認を強制的に有効にするには、`enabled: true` を設定します。Slack の実行承認を無効にしても、`approvals.plugin` を通じて有効になっているネイティブ Slack Plugin 承認の配信は無効になりません。Plugin 承認の配信では、代わりに Slack Plugin 承認者が使用されます。

Slack の実行承認を明示的に設定していない場合のデフォルト動作:

```json5
{
  commands: {
    ownerAllowFrom: ["slack:U12345678"],
  },
}
```

Slack ネイティブ設定を明示的に指定する必要があるのは、承認者の上書き、フィルターの追加、または発生元チャットへの配信を有効にする場合だけです。

```json5
{
  channels: {
    slack: {
      execApprovals: {
        enabled: true,
        approvers: ["U12345678"],
        target: "both",
      },
    },
  },
}
```

共有の `approvals.exec` 転送は別の機能です。実行承認プロンプトを他のチャットや明示的な帯域外ターゲットにもルーティングする必要がある場合にのみ使用してください。共有の `approvals.plugin` 転送も別の機能です。Slack が Plugin 承認リクエストをネイティブに処理できる場合にのみ、Slack ネイティブ配信はそのフォールバックを抑制します。

同一チャットの `/approve` は、すでにコマンドをサポートしている Slack チャンネルと DM でも機能します。承認転送モデル全体については、[実行承認](/ja-JP/tools/exec-approvals)を参照してください。

## イベントと運用動作

- メッセージの編集／削除はシステムイベントにマッピングされます。
- スレッドブロードキャスト（「Also send to channel」を指定したスレッド返信）は、通常のユーザーメッセージとして処理されます。
- リアクションの追加／削除イベントはシステムイベントにマッピングされます。
- メンバーの参加／退出、チャンネルの作成／名前変更、ピンの追加／削除イベントはシステムイベントにマッピングされます。
- オプションのプレゼンスポーリングでは、観測された人間の参加者の `away` から `active` への遷移を、その参加者が最後にアクティブだった適格な Slack セッションにマッピングできます。デフォルトでは無効です。
- `configWrites` が有効な場合、`channel_id_changed` はチャンネル設定キーを移行できます。
- チャンネルのトピック／目的メタデータは信頼されていないコンテキストとして扱われ、ルーティングコンテキストに注入できます。
- Agent View の `app_context` エンティティは Slack の関連度順で検証され、構造化された信頼されていないコンテキストとしてのみ公開されます。コンテキストが省略された場合、古いエンティティを再利用せず、そのターンのコンテキストをクリアします。
- スレッド開始メッセージと初期スレッド履歴のコンテキストシードは、該当する場合、設定された送信者許可リストによってフィルタリングされます。
- ブロックアクション、ショートカット、モーダルインタラクションは、豊富なペイロードフィールドを持つ構造化された `Slack interaction: ...` システムイベントを出力します。
  - ブロックアクション: 選択された値、ラベル、ピッカーの値、`workflow_*` メタデータ
  - グローバルショートカット: コールバックとアクターのメタデータ。アクターのダイレクトセッションへルーティング
  - メッセージショートカット: コールバック、アクター、チャンネル、スレッド、選択されたメッセージのコンテキスト
  - モーダルの `view_submission` および `view_closed` イベント。ルーティングされたチャンネルメタデータとフォーム入力を含む

Slack アプリ設定でグローバルショートカットまたはメッセージショートカットを定義し、空でない任意のコールバック ID を使用してください。OpenClaw は一致するショートカットペイロードを確認応答し、他の Slack インタラクションと同じ DM／チャンネル送信者ポリシーを適用して、サニタイズされたイベントをルーティング先のエージェントセッションにキューイングします。トリガー ID とレスポンス URL はエージェントコンテキストから秘匿化されます。

### プレゼンスイベント

Slack は Events API または Socket Mode を通じてプレゼンスの変更を送信しません。代わりに OpenClaw は、通常の Slack アクセスおよびルーティングチェックを通過したメッセージを送信した人間の参加者について、[`users.getPresence`](https://docs.slack.dev/reference/methods/users.getPresence/) をポーリングできます。

```json5
{
  channels: {
    slack: {
      presenceEvents: { mode: "auto" },
      channels: {
        C0123456789: { presenceEvents: { mode: "on" } },
        C0987654321: { presenceEvents: { mode: "off" } },
      },
    },
  },
}
```

- `off`（デフォルト）: プレゼンスタイマーも Slack API 呼び出しもありません。
- `auto`: 過去24時間以内にアクティブで、観測された人間の参加者が最大8人の DM、MPIM、および Slack スレッドを監視します。トップレベルのチャンネルセッションは除外されます。
- `on`: 参加者数の上限なしで同じ会話を監視し、トップレベルのチャンネルセッションも含めます。チャンネルごとの上書きを使用して、特定のチャンネルを強制的に有効または無効にできます。

OpenClaw は Slack アカウントごとに毎分最大45人の一意なユーザーをポーリングし、最初の結果ではエージェントを起動せずに初期状態を設定し、観測された `away` から `active` への遷移時にのみ起動します。Slack アカウントとユーザーの組み合わせごとに永続的な8時間のクールダウンが適用され、その人物が複数のスレッドに参加していても変わりません。イベントは、その人物が最後にアクティブだった適格な会話だけにルーティングされ、短い挨拶を1件送るかどうかを判断する前に、メモリ／Wiki と既知のタイムゾーンコンテキストを参照するようエージェントに指示します。エージェントは何も送信しないこともできます。

ボットトークンには `users:read` が必要です。これは推奨マニフェストにすでに含まれています。Enterprise Grid の組織全体インストールではプレゼンスイベントを利用できません。

## 設定リファレンス

主要なリファレンス: [設定リファレンス - Slack](/ja-JP/gateway/config-channels#slack)。

<Accordion title="重要な Slack フィールド">

- モード/認証: `identity`, `mode`, `enterpriseOrgInstall`, `botToken`, `appToken`, `userToken`, `signingSecret`, `webhookPath`, `accounts.*`
- DM アクセス: `dm.enabled`, `dmPolicy`, `allowFrom`（レガシー: `dm.policy`, `dm.allowFrom`）, `dm.groupEnabled`, `dm.groupChannels`
- 互換性切り替え: `dangerouslyAllowNameMatching`（緊急時用。必要な場合を除きオフのままにしてください）
- チャンネルアクセス: `groupPolicy`, `channels.*`, `channels.*.users`, `channels.*.requireMention`, `implicitMentions.*`
- スレッド/履歴: `replyToMode`, `replyToModeByChatType`, `thread.*`, `historyLimit`, `dmHistoryLimit`, `dms.*.historyLimit`
- プレゼンスによる起動: `presenceEvents.mode`, `channels.*.presenceEvents.mode`（`off|auto|on`、デフォルトは `off`）
- 配信: `textChunkLimit`, `streaming.chunkMode`, `mediaMaxMb`, `streaming`, `streaming.nativeTransport`, `streaming.preview.toolProgress`
- 展開プレビュー: `unfurlLinks`（デフォルト: `false`）、`chat.postMessage` のリンク/メディアプレビュー制御には `unfurlMedia`。リンクプレビューを再び有効にするには `unfurlLinks: true` を設定します
- 運用/機能: `configWrites`, `commands.native`, `slashCommand.*`, `actions.*`, `userToken`, `userTokenReadOnly`

</Accordion>

## トラブルシューティング

<AccordionGroup>
  <Accordion title="チャンネルで返信がない">
    次の順序で確認します:

    - `groupPolicy`
    - チャンネル許可リスト（`channels.slack.channels`）— **キーにはチャンネル名（`#channel-name`）ではなく、チャンネル ID（`C12345678`）を使用する必要があります**。チャンネルルーティングはデフォルトで ID を優先するため、`groupPolicy: "allowlist"` では名前ベースのキーは何の通知もなく失敗します。ID を確認するには、Slack でチャンネルを右クリック → **Copy link** — URL の末尾にある `C...` の値がチャンネル ID です。
    - `requireMention`
    - チャンネルごとの `users` 許可リスト
    - `messages.groupChat.visibleReplies`: 通常のグループ/チャンネルリクエストのデフォルトは `"automatic"` です。`"message_tool"` を有効にしており、ログにはアシスタントのテキストが表示されるものの `message(action=send)` 呼び出しがない場合、モデルは表示されているメッセージツールの経路を使用しませんでした。このモードでは最終テキストは非公開のままです。抑制されたペイロードのメタデータを Gateway の詳細ログで確認するか、通常のアシスタントの最終返信をすべてレガシー経路から投稿する場合は `"automatic"` に設定してください。
    - `messages.groupChat.unmentionedInbound`: `"room_event"` の場合、メンションされていない許可済みチャンネルの会話は周辺コンテキストとなり、エージェントが `message` ツールを呼び出さない限り応答しません。[ルームの周辺イベント](/ja-JP/channels/ambient-room-events)を参照してください。

```json5
{
  messages: {
    groupChat: {
      visibleReplies: "automatic",
    },
  },
}
```

    便利なコマンド:

```bash
openclaw channels status --probe
openclaw logs --follow
openclaw doctor
```

  </Accordion>

  <Accordion title="DM メッセージが無視される">
    確認事項:

    - `channels.slack.dm.enabled`
    - `channels.slack.dmPolicy`（またはレガシーの `channels.slack.dm.policy`）
    - ペアリングの承認/許可リストのエントリ（`dmPolicy: "open"` でも `channels.slack.allowFrom: ["*"]` が必要です）
    - グループ DM には MPIM 処理が使用されます。`channels.slack.dm.groupEnabled` を有効にし、設定している場合は MPIM を `channels.slack.dm.groupChannels` に含めます
    - Slack Assistant の DM イベント: `drop message_changed` に言及する詳細ログは通常、Slack が編集済みの Assistant スレッドイベントを送信し、メッセージメタデータから人間の送信者を復元できなかったことを意味します

```bash
openclaw pairing list slack
```

  </Accordion>

  <Accordion title="Socket モードが接続されない">
    Slack アプリの設定で、ボットトークンとアプリトークン、および Socket Mode が有効になっていることを検証します。
    App-Level Token には `connections:write` が必要であり、Bot User OAuth Token
    のボットトークンは、アプリトークンと同じ Slack アプリ/ワークスペースに属している必要があります。

    `openclaw channels status --probe --json` に `botTokenStatus` または
    `appTokenStatus: "configured_unavailable"` が表示される場合、Slack アカウントは
    設定されていますが、現在のランタイムは SecretRef によって参照される
    値を解決できませんでした。

    `slack socket mode failed to start; retry ...` などのログは、復旧可能な
    起動エラーです。一方、スコープの欠落、失効したトークン、無効な認証では直ちに失敗します。
    `slack token mismatch ...` ログは、ボットトークンとアプリトークンが
    異なる Slack アプリに属していると考えられることを意味します。Slack アプリの認証情報を修正してください。

  </Accordion>

  <Accordion title="HTTP モードでイベントを受信しない">
    次を検証します:

    - 署名シークレット
    - Webhook パス
    - Slack Request URLs（Events + Interactivity + Slash Commands）
    - HTTP アカウントごとに一意の `webhookPath`
    - 公開 URL で TLS を終端し、リクエストを Gateway パスに転送していること
    - Slack アプリの `request_url` パスが `channels.slack.webhookPath`（デフォルトは `/slack/events`）と完全に一致していること

    アカウントのスナップショットに `signingSecretStatus: "configured_unavailable"` が表示される場合、
    HTTP アカウントは設定されていますが、現在のランタイムは
    SecretRef によって参照される署名シークレットを解決できませんでした。

    `slack: webhook path ... already registered` ログが繰り返し出力される場合、2 つの HTTP
    アカウントが同じ `webhookPath` を使用しています。各アカウントに異なるパスを指定してください。

  </Accordion>

  <Accordion title="ネイティブ/スラッシュコマンドが実行されない">
    意図していたモードを確認します:

    - Slack に登録された対応するスラッシュコマンドを使用するネイティブコマンドモード（`channels.slack.commands.native: true`）
    - または単一スラッシュコマンドモード（`channels.slack.slashCommand.enabled: true`）

    Slack はスラッシュコマンドを自動的に作成または削除しません。`commands.native: "auto"` では Slack のネイティブコマンドは有効になりません。`true` を使用し、対応するコマンドを Slack アプリで作成してください。HTTP モードでは、すべての Slack スラッシュコマンドに Gateway URL を含める必要があります。Socket Mode では、コマンドペイロードは websocket 経由で届き、Slack は `slash_commands[].url` を無視します。

    `commands.useAccessGroups`、DM の認可、チャンネル許可リスト、
    チャンネルごとの `users` 許可リストも確認してください。Slack は、
    ブロックされたスラッシュコマンド送信者に対して、次のような一時的エラーを返します:

    - `This channel is not allowed.`
    - `You are not authorized to use this command here.`

  </Accordion>
</AccordionGroup>

## 添付メディアのリファレンス

Slack ファイルのダウンロードに成功し、サイズ制限内であれば、Slack はダウンロードしたメディアをエージェントのターンに添付できます。音声クリップは文字起こしでき、画像ファイルはメディア理解経路またはビジョン対応の返信モデルに直接渡すことができます。その他のファイルは、ダウンロード可能なファイルコンテキストとして引き続き利用できます。

### 対応するメディアタイプ

| メディアタイプ                 | ソース                 | 現在の動作                                                                        | 注記                                                                      |
| ------------------------------ | -------------------- | --------------------------------------------------------------------------------- | ------------------------------------------------------------------------- |
| Slack 音声クリップ             | Slack ファイル URL    | ダウンロードされ、共有音声文字起こしを経由してルーティングされる                  | `files:read` と、動作する `tools.media.audio` モデルまたは CLI が必要です |
| JPEG / PNG / GIF / WebP 画像   | Slack ファイル URL    | ダウンロードされ、ビジョン対応処理のためにターンへ添付される                      | ファイルごとの上限: `channels.slack.mediaMaxMb`（デフォルト 20 MB）                |
| PDF ファイル                   | Slack ファイル URL    | ダウンロードされ、`download-file` や `pdf` などのツール向けファイルコンテキストとして公開される | Slack の受信処理では PDF は自動的に画像ビジョン入力へ変換されません |
| その他のファイル               | Slack ファイル URL    | 可能な場合はダウンロードされ、ファイルコンテキストとして公開される                | バイナリファイルは画像入力として扱われません                              |
| スレッドの返信                 | スレッド開始メッセージのファイル | 返信に直接メディアがない場合、ルートメッセージのファイルをコンテキストとして取り込める | ファイルのみの開始メッセージでは添付プレースホルダーが使用されます        |
| 複数ファイルのメッセージ       | 複数の Slack ファイル | 各ファイルが個別に評価される                                                      | Slack の処理はメッセージあたり 8 ファイルが上限です                       |

### 受信パイプライン

ファイルが添付された Slack メッセージを受信すると:

1. OpenClaw はボットトークンを使用して Slack の非公開 URL からファイルをダウンロードします。
2. 成功すると、ファイルはメディアストアに書き込まれます。
3. ダウンロードされたメディアのパスとコンテンツタイプが受信コンテキストに追加されます。
4. 音声クリップは共有文字起こしパイプラインにルーティングされます。同じコンテキストの画像添付ファイルは、画像対応のモデル/ツール経路で使用できます。
5. その他のファイルは、それらを処理できるツール向けのファイルメタデータまたはメディア参照として引き続き利用できます。

### スレッドルートの添付ファイル継承

メッセージがスレッド内で受信された場合（`thread_ts` の親を持つ場合）:

- 返信自体に直接メディアがなく、含まれるルートメッセージにファイルがある場合、Slack はルートファイルをスレッド開始時のコンテキストとして取り込めます。
- ルートファイルが取り込まれるのは、新規またはリセットされたスレッドセッションを初期化するときだけです。それ以降のテキストのみの返信では既存のセッションコンテキストが再利用され、ルートファイルが新しいメディアとして再添付されることはありません。
- 返信に直接添付されたファイルは、ルートメッセージの添付ファイルより優先されます。
- ファイルのみでテキストがないルートメッセージは添付プレースホルダーで表されるため、フォールバックでもそのファイルを含めることができます。

### 複数添付ファイルの処理

1 件の Slack メッセージに複数のファイルが添付されている場合:

- 各添付ファイルは、メディアパイプラインで個別に処理されます。
- ダウンロードされたメディア参照はメッセージコンテキストに集約されます。
- 処理順序は、イベントペイロード内の Slack のファイル順に従います。
- 1 つの添付ファイルのダウンロードに失敗しても、他のファイルはブロックされません。

### サイズ、ダウンロード、モデルの制限

- **サイズ上限**: デフォルトはファイルごとに 20 MB。`channels.slack.mediaMaxMb` で設定できます。
- **音声文字起こしの上限**: ダウンロードされたファイルが文字起こしプロバイダーまたは CLI に送信される場合、選択された音声対応 `tools.media.models[]` エントリの `maxBytes` も適用されます。
- **ダウンロード失敗**: Slack が提供できないファイル、期限切れの URL、アクセスできないファイル、サイズ上限を超えたファイル、Slack の認証/ログイン HTML 応答は、未対応形式として報告されるのではなくスキップされます。
- **ビジョンモデル**: 画像分析では、ビジョンに対応している場合は有効な返信モデルを使用し、それ以外の場合は `agents.defaults.imageModel` で設定された画像モデルを使用します。

### 既知の制限

| シナリオ                                      | 現在の動作                                                                   | 回避策                                                                    |
| --------------------------------------------- | ---------------------------------------------------------------------------------- | ----------------------------------------------------------------------------- |
| 有効期限切れの Slack ファイル URL                        | ファイルはスキップされ、エラーは表示されない                                                       | Slack にファイルを再アップロードする                                                   |
| 音声文字起こしが利用できない               | クリップは添付されたままだが、文字起こしは生成されない                                | `tools.media.audio` を設定するか、サポートされているローカル文字起こし CLI をインストールする  |
| キャプションのないクリップがメンションゲートを通過しない | 非公開の推測的な文字起こし後に破棄され、文字起こしとダウンロードも破棄される | 発話された名前のメンションパターンを設定する、入力したボットメンションを追加する、または DM を使用する |
| ビジョンモデルが設定されていない                   | 画像添付ファイルはメディア参照として保存されるが、画像としては分析されない       | `agents.defaults.imageModel` を設定するか、ビジョン対応の応答モデルを使用する    |
| 非常に大きな画像（デフォルトでは > 20 MB）        | サイズ上限に従ってスキップされる                                                               | Slack で許可されている場合は `channels.slack.mediaMaxMb` を増やす                          |
| 転送／共有された添付ファイル                  | テキストと Slack でホストされる画像／ファイルメディアはベストエフォートで処理される                             | OpenClaw スレッドで直接再共有する                                      |
| PDF 添付ファイル                               | ファイル／メディアコンテキストとして保存され、画像ビジョンには自動的にルーティングされない        | ファイルメタデータには `download-file` を、PDF 分析には `pdf` ツールを使用する      |

### 関連ドキュメント

- [メディア理解パイプライン](/ja-JP/nodes/media-understanding)
- [音声とボイスメモ](/ja-JP/nodes/audio)
- [PDF ツール](/ja-JP/tools/pdf)

## 関連項目

<CardGroup cols={2}>
  <Card title="ペアリング" icon="link" href="/ja-JP/channels/pairing">
    Slack ユーザーを Gateway とペアリングします。
  </Card>
  <Card title="グループ" icon="users" href="/ja-JP/channels/groups">
    チャンネルとグループ DM の動作。
  </Card>
  <Card title="チャンネルルーティング" icon="route" href="/ja-JP/channels/channel-routing">
    受信メッセージをエージェントにルーティングします。
  </Card>
  <Card title="セキュリティ" icon="shield" href="/ja-JP/gateway/security">
    脅威モデルと堅牢化。
  </Card>
  <Card title="設定" icon="sliders" href="/ja-JP/gateway/configuration">
    設定のレイアウトと優先順位。
  </Card>
  <Card title="スラッシュコマンド" icon="terminal" href="/ja-JP/tools/slash-commands">
    コマンドカタログと動作。
  </Card>
</CardGroup>
