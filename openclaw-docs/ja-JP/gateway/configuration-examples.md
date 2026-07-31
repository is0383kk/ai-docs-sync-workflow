---
read_when:
    - OpenClaw の設定方法を学ぶ
    - 設定例を探す
    - OpenClaw を初めてセットアップする
summary: 一般的な OpenClaw セットアップ向けのスキーマに準拠した設定例
title: 設定例
x-i18n:
    generated_at: "2026-07-26T09:02:10Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: ade743a23e24f2e927d1bb1e1828893e24d3d718ec321dd8fda3932830be8331
    source_path: gateway/configuration-examples.md
    workflow: 16
---

以下の例は、現在の設定スキーマに準拠しています。網羅的なリファレンスと各フィールドの注記については、[設定](/ja-JP/gateway/configuration)を参照してください。

## クイックスタート

### 最小限の設定

```json5
{
  agents: { defaults: { workspace: "~/.openclaw/workspace" } },
  channels: { whatsapp: { allowFrom: ["+15555550123"] } },
}
```

`~/.openclaw/openclaw.json` に保存すると、その番号からボットに DM を送信できます。

### 推奨スターター設定

```json5
{
  agents: {
    defaults: {
      workspace: "~/.openclaw/workspace",
      model: { primary: "anthropic/claude-sonnet-4-6" },
    },
    entries: {
      main: {
        identity: {
          name: "Clawd",
          theme: "helpful assistant",
          emoji: "🦞",
        },
      },
    },
  },
  channels: {
    whatsapp: {
      allowFrom: ["+15555550123"],
      groups: { "*": { requireMention: true } },
    },
  },
  messages: {
    visibleReplies: "automatic",
    groupChat: {
      visibleReplies: "message_tool", // オプトイン。表示可能な出力には message(action=send) が必要
      unmentionedInbound: "room_event",
    },
  },
}
```

## 拡張例（主要なオプション）

> JSON5 ではコメントと末尾のカンマを使用できます。通常の JSON も使用できます。

```json5
{
  // 環境変数とシェル
  env: {
    OPENROUTER_API_KEY: "sk-or-...",
    vars: {
      GROQ_API_KEY: "gsk-...",
    },
    shellEnv: {
      enabled: true,
      timeoutMs: 15000,
    },
  },

  // 認証プロファイルのメタデータ（シークレットは auth-profiles.json に保存）
  auth: {
    profiles: {
      "anthropic:default": { provider: "anthropic", mode: "api_key" },
      "anthropic:work": { provider: "anthropic", mode: "api_key" },
      "openai:default": { provider: "openai", mode: "api_key" },
      "openai:personal": { provider: "openai", mode: "oauth" },
    },
    order: {
      anthropic: ["anthropic:default", "anthropic:work"],
      openai: ["openai:personal", "openai:default"],
    },
  },

  // ID はエージェントごとに設定します。以下の agents.entries.<id>.identity に設定してください。

  // ロギング
  logging: {
    level: "info",
    file: "/tmp/openclaw/openclaw.log",
    consoleLevel: "info",
    consoleStyle: "pretty",
    redactSensitive: "tools",
  },

  // メッセージの書式設定
  messages: {
    visibleReplies: "automatic",
    responsePrefix: ">",
    ackReaction: "👀",
    ackReactionScope: "group-mentions",
    groupChat: {
      historyLimit: 50,
      visibleReplies: "message_tool", // ツールを確実に使用できるモデルを共有ルームで使う場合にオプトイン
      unmentionedInbound: "room_event",
    },
    queue: {
      mode: "followup",
      cap: 20,
      drop: "summarize",
      byChannel: {
        whatsapp: "followup",
        telegram: "followup",
        discord: "collect",
        slack: "collect",
        signal: "followup",
        imessage: "followup",
        webchat: "followup",
      },
    },
  },

  // セッションの動作
  session: {
    scope: "per-sender",
    dmScope: "per-channel-peer", // 複数ユーザーの受信トレイに推奨
    reset: {
      mode: "daily",
      atHour: 4,
      idleMinutes: 60,
    },
    resetByChannel: {
      discord: { mode: "idle", idleMinutes: 10080 },
    },
    resetTriggers: ["/new", "/reset"],
    store: "~/.openclaw/agents/main/sessions/sessions.json",
    maintenance: {
      mode: "warn",
      pruneAfter: "30d",
      maxEntries: 500,
      resetArchiveRetention: "30d", // 期間または false
      maxDiskBytes: "500mb", // 省略可能
      highWaterBytes: "400mb", // 省略可能（デフォルトは maxDiskBytes の 80%）
    },
    sendPolicy: {
      default: "allow",
      rules: [{ action: "deny", match: { channel: "discord", chatType: "group" } }],
    },
  },

  // チャンネル
  channels: {
    whatsapp: {
      dmPolicy: "pairing",
      allowFrom: ["+15555550123"],
      groupPolicy: "allowlist",
      groupAllowFrom: ["+15555550123"],
      groups: { "*": { requireMention: true } },
    },

    telegram: {
      enabled: true,
      botToken: "YOUR_TELEGRAM_BOT_TOKEN",
      allowFrom: ["123456789"],
      groupPolicy: "allowlist",
      groupAllowFrom: ["123456789"],
      groups: { "*": { requireMention: true } },
    },

    discord: {
      enabled: true,
      token: "YOUR_DISCORD_BOT_TOKEN",
      dmPolicy: "allowlist",
      allowFrom: ["123456789012345678"],
      guilds: {
        "123456789012345678": {
          slug: "friends-of-openclaw",
          requireMention: false,
          channels: {
            general: { enabled: true },
            help: { enabled: true, requireMention: true },
          },
        },
      },
    },

    slack: {
      enabled: true,
      botToken: "xoxb-REPLACE_ME",
      appToken: "xapp-REPLACE_ME",
      channels: {
        "#general": { enabled: true, requireMention: true },
      },
      dmPolicy: "allowlist",
      allowFrom: ["U123"],
      slashCommand: {
        enabled: true,
        name: "openclaw",
        sessionPrefix: "slack:slash",
        ephemeral: true,
      },
    },
  },

  // エージェントランタイム
  agents: {
    defaults: {
      workspace: "~/.openclaw/workspace",
      userTimezone: "America/Chicago",
      model: {
        primary: "anthropic/claude-sonnet-4-6",
        fallbacks: ["anthropic/claude-opus-4-6", "openai/gpt-5.4"],
      },
      imageModel: {
        primary: "openrouter/anthropic/claude-sonnet-4-6",
      },
      models: {
        "anthropic/claude-opus-4-6": { alias: "opus" },
        "anthropic/claude-sonnet-4-6": { alias: "sonnet" },
        "openai/gpt-5.4": { alias: "gpt" },
      },
      skills: ["github", "weather"], // list[].skills を省略したエージェントに継承
      thinkingDefault: "low",
      verboseDefault: "off",
      toolProgressDetail: "explain",
      reasoningDefault: "off",
      elevatedDefault: "on",
      blockStreamingDefault: "off",
      blockStreamingBreak: "text_end",
      blockStreamingChunk: {
        minChars: 800,
        maxChars: 1200,
        breakPreference: "paragraph",
      },
      blockStreamingCoalesce: {
        idleMs: 1000,
      },
      humanDelay: {
        mode: "natural",
      },
      timeoutSeconds: 600,
      mediaMaxMb: 5,
      typingIntervalSeconds: 5,
      maxConcurrent: 3,
      heartbeat: {
        every: "30m",
        model: "anthropic/claude-sonnet-4-6",
        target: "last",
        directPolicy: "allow", // allow（デフォルト）| block
        to: "+15555550123",
        prompt: "HEARTBEAT",
        ackMaxChars: 300,
      },
      sandbox: {
        mode: "non-main",
        scope: "session", // 従来の perSession: true より推奨
        workspaceRoot: "~/.openclaw/sandboxes",
        docker: {
          image: "openclaw-sandbox:bookworm-slim",
          workdir: "/workspace",
          readOnlyRoot: true,
          tmpfs: ["/tmp", "/var/tmp", "/run"],
          network: "none",
          user: "1000:1000",
        },
        browser: {
          enabled: false,
        },
      },
    },
    entries: {
      main: {
        default: true,
        identity: {
          name: "Samantha",
          theme: "helpful sloth",
          emoji: "🦥",
        },
        // defaults.skills（github、weather）を継承
        groupChat: {
          mentionPatterns: ["@openclaw", "openclaw"],
        },
        thinkingDefault: "high", // エージェントごとの思考設定の上書き
        reasoningDefault: "on", // エージェントごとの推論表示設定
        fastModeDefault: false, // エージェントごとの高速モード
      },
      quick: {
        skills: [], // このエージェントでは Skills を使用しない
        fastModeDefault: true, // このエージェントは常に高速で実行
        thinkingDefault: "off",
      },
    },
  },

  memory: {
    search: {
      provider: "gemini",
      model: "gemini-embedding-001",
      remote: {
        apiKey: "${GEMINI_API_KEY}",
      },
      extraPaths: ["../team-docs", "/srv/shared-notes"],
    },
  },

  tools: {
    media: {
      models: [
        { provider: "openai", model: "gpt-4o-transcribe", capabilities: ["audio"] },
        { provider: "google", model: "gemini-3-flash-preview", capabilities: ["video"] },
      ],
      audio: { enabled: true, maxBytes: 20971520, timeoutSeconds: 120 },
      video: { enabled: true, maxBytes: 52428800 },
    },
    allow: ["exec", "process", "read", "write", "edit", "apply_patch"],
    deny: ["browser", "canvas"],
    exec: {
      backgroundMs: 10000,
      timeoutSeconds: 1800,
      cleanupMs: 1800000,
    },
    elevated: {
      enabled: true,
      allowFrom: {
        whatsapp: ["+15555550123"],
        telegram: ["123456789"],
        discord: ["123456789012345678"],
        slack: ["U123"],
        signal: ["+15555550123"],
        imessage: ["user@example.com"],
        webchat: ["session:demo"],
      },
    },
  },

  // カスタムモデルプロバイダー
  models: {
    mode: "merge",
    providers: {
      "custom-proxy": {
        baseUrl: "http://localhost:4000/v1",
        apiKey: "LITELLM_KEY",
        api: "openai-responses",
        authHeader: true,
        headers: { "X-Proxy-Region": "us-west" },
        models: [
          {
            id: "llama-3.1-8b",
            name: "Llama 3.1 8B",
            api: "openai-responses",
            reasoning: false,
            input: ["text"],
            cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 },
            contextWindow: 128000,
            maxTokens: 32000,
          },
        ],
      },
    },
  },

  // Cron ジョブ
  cron: {
    enabled: true,
    store: "~/.openclaw/cron/jobs.json",
    sessionRetention: "24h",
  },

  // Webhook
  hooks: {
    enabled: true,
    path: "/hooks",
    token: "shared-secret",
    presets: ["gmail"],
    transformsDir: "~/.openclaw/hooks/transforms",
    mappings: [
      {
        id: "gmail-hook",
        match: { path: "gmail" },
        action: "agent",
        wakeMode: "now",
        name: "Gmail",
        sessionKey: "hook:gmail:{{messages[0].id}}",
        messageTemplate: "送信元: {{messages[0].from}}\n件名: {{messages[0].subject}}",
        textTemplate: "{{messages[0].snippet}}",
        deliver: true,
        channel: "last",
        to: "+15555550123",
        thinking: "low",
        timeoutSeconds: 300,
        transform: {
          module: "gmail.js",
          export: "transformGmail",
        },
      },
    ],
    gmail: {
      account: "openclaw@gmail.com",
      label: "INBOX",
      topic: "projects/<project-id>/topics/gog-gmail-watch",
      subscription: "gog-gmail-watch-push",
      pushToken: "shared-push-token",
      hookUrl: "http://127.0.0.1:18789/hooks/gmail",
      includeBody: true,
      maxBytes: 20000,
      renewEveryMinutes: 720,
      serve: { bind: "127.0.0.1", port: 8788, path: "/" },
      tailscale: { mode: "funnel", path: "/gmail-pubsub" },
    },
  },

  // Gateway とネットワーク
  gateway: {
    mode: "local",
    port: 18789,
    bind: "loopback",
    controlUi: { enabled: true, basePath: "/openclaw" },
    auth: {
      mode: "token",
      token: "gateway-token",
      allowTailscale: true,
    },
    tailscale: { mode: "serve", resetOnExit: false },
    remote: { url: "ws://gateway-host.ts.net:18789", token: "remote-token" },
    reload: { mode: "hybrid" },
  },

  skills: {
    allowBundled: ["gemini", "peekaboo"],
    load: {
      extraDirs: ["~/Projects/agent-scripts/skills"],
      allowSymlinkTargets: ["~/Projects/agent-scripts/skills"],
    },
    install: {
      preferBrew: true,
      nodeManager: "npm", // npm | pnpm | yarn | bun
      allowUploadedArchives: false,
    },
    entries: {
      "image-lab": {
        enabled: true,
        apiKey: "GEMINI_KEY_HERE",
        env: { GEMINI_API_KEY: "GEMINI_KEY_HERE" },
      },
      peekaboo: { enabled: true },
    },
  },
}
```

### シンボリックリンクされた兄弟 Skills リポジトリ

組み込み Skills のルートに兄弟リポジトリへのシンボリックリンクが含まれている場合に使用します。たとえば `~/.agents/skills/manager -> ~/Projects/manager/skills` です。

```json5
{
  skills: {
    load: {
      extraDirs: ["~/Projects/manager/skills"],
      allowSymlinkTargets: ["~/Projects/manager/skills"],
    },
  },
}
```

- `extraDirs` は、隣接するリポジトリを明示的な Skills ルートとしてスキャンします。
- `allowSymlinkTargets` を使用すると、任意のシンボリックリンクによる脱出を許可せずに、シンボリックリンクされた Skills フォルダーを、その信頼済みの
  実ターゲットルートへ解決できます。
- Skill Workshop が同じ信頼済みシンボリックリンクターゲットを介して書き込みを適用できるようにするには、
  `skills.workshop.allowSymlinkTargetWrites: true` を設定します。

## 一般的なパターン

### 1 つのオーバーライドを持つ共有 Skills ベースライン

```json5
{
  agents: {
    defaults: {
      workspace: "~/.openclaw/workspace",
      skills: ["github", "weather"],
    },
    entries: {
      main: { default: true },
      docs: { workspace: "~/.openclaw/workspace-docs", skills: ["docs-search"] },
    },
  },
}
```

- `agents.defaults.skills` は共有ベースラインです。
- `agents.entries.*.skills` は、1 つのエージェントについてそのベースラインを置き換えます。
- エージェントに Skills を一切表示しない場合は、`skills: []` を使用します。

### マルチプラットフォーム設定

```json5
{
  agents: { defaults: { workspace: "~/.openclaw/workspace" } },
  channels: {
    whatsapp: { allowFrom: ["+15555550123"], responsePrefix: "[openclaw]" },
    telegram: {
      enabled: true,
      botToken: "YOUR_TOKEN",
      allowFrom: ["123456789"],
    },
    discord: {
      enabled: true,
      token: "YOUR_TOKEN",
      allowFrom: ["123456789012345678"],
    },
  },
}
```

### 信頼済み Node ネットワークの自動承認

ネットワーク経路を管理している場合を除き、デバイスのペアリングは手動のままにしてください。専用の
ラボまたは tailnet サブネットでは、正確な CIDR または IP を指定して、初回の Node デバイスの自動承認を
有効にできます。

```json5
{
  gateway: {
    nodes: {
      pairing: {
        autoApproveCidrs: ["192.168.1.0/24", "fd00:1234:5678::/64"],
      },
    },
  },
}
```

未設定の場合、これは無効のままです。要求されたスコープがない、新規の `role: node` ペアリングにのみ
適用されます。オペレーター／ブラウザクライアント、およびロール、スコープ、メタデータ、または
公開鍵のアップグレードには、引き続き手動承認が必要です。

### セキュア DM モード（共有受信トレイ／複数ユーザーの DM）

複数の人がボットに DM を送信できる場合（`allowFrom` に複数のエントリがある、複数の人のペアリングが承認されている、または `dmPolicy: "open"`）、**セキュア DM モード**を有効にすると、異なる送信者からの DM がデフォルトで同じコンテキストを共有しなくなります。

```json5
{
  // Secure DM mode (recommended for multi-user or sensitive DM agents)
  session: { dmScope: "per-channel-peer" },

  channels: {
    // Example: WhatsApp multi-user inbox
    whatsapp: {
      dmPolicy: "allowlist",
      allowFrom: ["+15555550123", "+15555550124"],
    },

    // Example: Discord multi-user inbox
    discord: {
      enabled: true,
      token: "YOUR_DISCORD_BOT_TOKEN",
      allowFrom: ["123456789012345678", "987654321098765432"],
    },
  },
}
```

Discord／Google Chat／IRC／Mattermost／Microsoft Teams／Slack では、送信者の認可はデフォルトで ID を優先します。
そのリスクを明示的に受け入れる場合に限り、各チャネルの `dangerouslyAllowNameMatching: true` で、変更可能な名前／メールアドレス／ニックネームの直接照合を有効にしてください。

### Anthropic API キーと MiniMax フォールバック

```json5
{
  auth: {
    profiles: {
      "anthropic:api": {
        provider: "anthropic",
        mode: "api_key",
      },
    },
    order: {
      anthropic: ["anthropic:api"],
    },
  },
  models: {
    providers: {
      minimax: {
        baseUrl: "https://api.minimax.io/anthropic",
        api: "anthropic-messages",
        apiKey: "${MINIMAX_API_KEY}",
      },
    },
  },
  agents: {
    defaults: {
      workspace: "~/.openclaw/workspace",
      model: {
        primary: "anthropic/claude-opus-4-6",
        fallbacks: ["minimax/MiniMax-M2.7"],
      },
    },
  },
}
```

### 業務用ボット（アクセス制限付き）

```json5
{
  agents: {
    defaults: {
      workspace: "~/work-openclaw",
      elevatedDefault: "off",
    },
    entries: {
      main: {
        identity: {
          name: "WorkBot",
          theme: "professional assistant",
        },
      },
    },
  },
  channels: {
    slack: {
      enabled: true,
      botToken: "xoxb-...",
      channels: {
        "#engineering": { enabled: true, requireMention: true },
        "#general": { enabled: true, requireMention: true },
      },
    },
  },
}
```

### ローカルモデルのみ

```json5
{
  agents: {
    defaults: {
      workspace: "~/.openclaw/workspace",
      model: { primary: "lmstudio/my-local-model" },
    },
  },
  models: {
    mode: "merge",
    providers: {
      lmstudio: {
        baseUrl: "http://127.0.0.1:1234/v1",
        apiKey: "lmstudio",
        api: "openai-responses",
        models: [
          {
            id: "my-local-model",
            name: "Local Model",
            reasoning: false,
            input: ["text"],
            cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 },
            contextWindow: 196608,
            maxTokens: 8192,
          },
        ],
      },
    },
  },
}
```

## ヒント

- `dmPolicy: "open"` を設定する場合、対応する `allowFrom` リストに `"*"` を含める必要があります。
- プロバイダー ID の形式は異なります（電話番号、ユーザー ID、チャネル ID）。形式を確認するには、プロバイダーのドキュメントを参照してください。
- 後で追加できる任意のセクション：`web`、`browser`、`ui`、`discovery`、`plugins`、`talk`、`signal`、`imessage`。
- 設定に関する詳細な注意事項については、[プロバイダー](/ja-JP/providers)と[トラブルシューティング](/ja-JP/gateway/troubleshooting)を参照してください。

## 関連項目

- [設定リファレンス](/ja-JP/gateway/configuration-reference)
- [設定](/ja-JP/gateway/configuration)
