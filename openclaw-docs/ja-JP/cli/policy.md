---
read_when:
    - 作成済みの policy.jsonc に照らして OpenClaw の設定を確認する場合
    - doctor lint にポリシー違反の検出結果を表示したい場合
    - 監査証跡にはポリシー証明ハッシュが必要です
summary: '`openclaw policy` 適合性チェックの CLI リファレンス'
title: ポリシー
x-i18n:
    generated_at: "2026-07-26T08:59:31Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 63e4faeab8dd6535e3d517439d3f58cdc167b6b7fade808a6482742ec9b5acf1
    source_path: cli/policy.md
    workflow: 16
---

# `openclaw policy`

`openclaw policy` は、バンドルされた Policy Plugin によって提供されます。これは既存の OpenClaw 設定上に構築されたエンタープライズ向けの
適合性レイヤーであり、2つ目の設定システムではありません。要件は `policy.jsonc` に記述します。OpenClaw はアクティブな
ワークスペースを証拠として観測し、Policy は `doctor --lint` を通じて逸脱を報告します。Policy は
ツール呼び出しを強制したり、リクエスト時にランタイムの動作を書き換えたりせず、
`auth-profiles.json` などのエージェント別認証情報ストアを証明することもありません。

Policy は、設定済みチャンネル、MCP サーバー、モデルプロバイダー、ネットワークの SSRF
態勢、イングレスおよびチャンネルアクセス、Gateway の公開範囲と Node コマンド態勢、
記述済みのメッセージルーティングプローブ、
エージェントのワークスペースアクセス、サンドボックス態勢、データ処理態勢、シークレット
プロバイダーおよび認証プロファイルの態勢、ならびに管理対象ツールのメタデータ（`TOOLS.md`）を検査します。
ワークスペースに「Telegram を有効にしてはならない」や「管理対象ツールはリスクと所有者の
メタデータを宣言しなければならない」といった、永続的で検査可能な宣言が必要な場合に使用します。
証明や逸脱検出を伴わないローカル動作だけが必要な場合は、通常の
設定で十分です。

## クイックスタート

```bash
openclaw plugins enable policy
```

`policy.jsonc` が存在しない場合でも Plugin は有効なままなので、doctor は
検査を暗黙にスキップするのではなく、不足している成果物を報告できます。

`policy.jsonc` は手動で記述します。現在の設定から生成されるものではありません。各
トップレベルセクションはルールの名前空間です。具体的なルールがその配下に存在する場合にのみ
検査が実行されます（未対応のセクションやキーは暗黙に無視されず、
`policy/policy-jsonc-invalid` として失敗します）。サポートされるすべてのセクションを網羅する
最小限の例を次に示します。

```jsonc
{
  "channels": {
    "denyRules": [
      {
        "id": "no-telegram",
        "when": { "provider": "telegram" },
        "reason": "Telegram はこのワークスペースでは承認されていません。",
      },
    ],
  },
  "mcp": {
    "servers": {
      "allow": ["docs"],
      "deny": ["untrusted"],
    },
  },
  "models": {
    "providers": {
      "allow": ["openai", "anthropic"],
      "deny": ["openrouter"],
    },
  },
  "network": {
    "privateNetwork": {
      "allow": false,
    },
  },
  "routing": {
    "requireBindings": true,
    "requireConfiguredChannels": true,
    "probes": [
      {
        "id": "family-dm",
        "route": {
          "channel": "imessage",
          "peer": { "kind": "direct", "id": "+15555550123" },
        },
        "expect": {
          "agentId": "family",
          "matchedBy": ["binding.peer"],
        },
      },
    ],
  },
  "ingress": {
    "session": {
      "requireDmScope": "per-channel-peer",
    },
    "channels": {
      "allowDmPolicies": ["pairing", "allowlist", "disabled"],
      "denyOpenGroups": true,
      "requireMentionInGroups": true,
    },
  },
  "gateway": {
    "exposure": {
      "allowNonLoopbackBind": false,
      "allowTailscaleFunnel": false,
    },
    "auth": {
      "requireAuth": true,
      "requireExplicitRateLimit": true,
    },
    "controlUi": {
      "allowInsecure": false,
    },
    "remote": {
      "allow": false,
    },
    "http": {
      "denyEndpoints": ["chatCompletions", "responses"],
      "requireUrlAllowlists": true,
    },
    "nodes": {
      "denyCommands": ["system.run"],
    },
  },
  "agents": {
    "workspace": {
      "allowedAccess": ["none", "ro"],
      "denyTools": ["exec", "process", "write", "edit", "apply_patch"],
    },
  },
  "dataHandling": {
    "sensitiveLogging": {
      "requireRedaction": true,
    },
    "telemetry": {
      "denyContentCapture": true,
    },
    "retention": {
      "requireSessionMaintenance": true,
    },
    "memory": {
      "denySessionTranscriptIndexing": true,
    },
  },
  "secrets": {
    "requireManagedProviders": true,
    "denySources": ["exec"],
    "allowInsecureProviders": false,
  },
  "auth": {
    "profiles": {
      "requireMetadata": ["provider", "mode"],
      "allowModes": ["api_key", "token"],
    },
  },
  "execApprovals": {
    "requireFile": true,
    "defaults": { "allowSecurity": ["deny"] },
    "agents": {
      "allowSecurity": ["deny", "allowlist"],
      "allowAutoAllowSkills": false,
      "allowlist": { "expected": ["deploy", "status"] },
    },
  },
  "tools": {
    "requireMetadata": ["risk", "sensitivity", "owner"],
    "profiles": {
      "allow": ["messaging", "minimal"],
    },
    "fs": {
      "requireWorkspaceOnly": true,
    },
    "exec": {
      "allowSecurity": ["deny", "allowlist"],
      "requireAsk": ["always"],
      "allowHosts": ["sandbox"],
    },
    "elevated": {
      "allow": false,
    },
    "denyTools": ["group:runtime", "group:fs"],
  },
}
```

以下のルール表からは明確でない、複数領域にまたがる注意事項を示します。

- local loopback 以外へのバインドを拒否しながら `gateway.bind` を省略すると、ランタイムの
  デフォルトを受け入れることになります。厳密な適合性を確保するには `gateway.bind: "loopback"` を設定してください。
- 読み取り専用エージェントの場合、該当するデフォルトまたはエージェントでサンドボックスの `mode` を
  `all` または `non-main` に、`workspaceAccess` を `none` または `ro` に設定してください。サンドボックスモードが未指定または
  `off` の場合、読み取り専用ポリシーを満たしません。
- `agents.workspace.denyTools` は `exec`、`process`、`write`、`edit`、
  `apply_patch` を受け入れます。設定のツール拒否グループ `group:fs`（ファイル変更）と
  `group:runtime`（シェル／プロセス）は、同等の態勢を満たします。
- 実行承認の検査は、`execApprovals` ルールが存在する場合にのみ、
  稼働中の `exec-approvals.json` 成果物を読み取ります。成果物が存在しないか無効な場合、それは
  観測不能な証拠であり、合格として合成されるものではありません。
- シークレットおよび認証プロファイルの証拠には、プロバイダー／ソースの態勢と
  SecretRef メタデータのみが記録され、生の値は一切記録されません。Policy は
  `auth-profiles.json` などのエージェント別認証情報ストアを読み取ったり証明したりしません。
- データ処理の証拠は、設定レベルの態勢（秘匿化モード、
  テレメトリ取得の切り替え、セッション保守モード、トランスクリプトのインデックス設定）のみです。
  ログ、テレメトリエクスポート、トランスクリプト、メモリファイルは検査されず、
  検査結果に問題がなくても、それらに個人データやシークレットが存在しないことは証明されません。
- ルーティングプローブは、OpenClaw のランタイムバインディングリゾルバーを再利用します。ルーティングの証拠には、
  プローブ ID、解決されたエージェント、照合種別、秘匿化されたバインディング
  メタデータのみが記録されます。ピア、アカウント、ギルド、チーム、ロールの識別子は一切記録されません。
  ルーティングセクションを追加すると、ポリシーと証明の
  ハッシュが意図的に変更されます。ルーティングのないポリシーでは、既存の証拠形式が維持されます。

### Policy ルールリファレンス

以下のすべてのルールは任意です。ルールが存在する場合にのみ検査が実行されます。
観測される状態は、既存の OpenClaw 設定またはワークスペースのメタデータです。

#### スコープ付きオーバーレイ

特定のエージェントまたはチャンネルで、トップレベルのベースラインより厳格なポリシーが必要な場合は、
`scopes.<scopeName>` を使用します。スコープ名は単なるラベルです。照合には
スコープ内のセレクターが使用されます。オーバーレイは加算的です。グローバルルールは引き続き実行され、
スコープ付きルールは同じ証拠に対して独自の検出結果を追加できます。

| セレクター     | サポートされるセクション                                                             | 使用する場合                                          |
| ------------ | ------------------------------------------------------------------------------ | ------------------------------------------------- |
| `agentIds`   | `tools`、`agents.workspace`、`sandbox`、`dataHandling.memory`、`execApprovals` | 1つ以上のランタイムエージェントに、より厳格なルールが必要な場合。   |
| `channelIds` | `ingress.channels`                                                             | 1つ以上のチャンネルに、より厳格なイングレスルールが必要な場合。 |

`agentIds` エントリが `agents.entries.*` に存在しない場合、OpenClaw は
スコープ付きルールをスキップせず、そのランタイムエージェント ID に継承された
グローバル／デフォルトの態勢に対して評価します。

```jsonc
{
  "tools": {
    "exec": {
      "allowHosts": ["sandbox", "node"],
    },
  },
  "sandbox": {
    "requireMode": ["all", "non-main"],
  },
  "scopes": {
    "release-workspace": {
      "agentIds": ["release-agent", "review-agent"],
      "agents": {
        "workspace": {
          "allowedAccess": ["none", "ro"],
        },
      },
    },
    "release-lockdown": {
      "agentIds": ["release-agent"],
      "tools": {
        "exec": {
          "allowHosts": ["sandbox"],
          "allowSecurity": ["deny", "allowlist"],
          "requireAsk": ["always"],
        },
        "denyTools": ["exec", "process", "write", "edit", "apply_patch"],
      },
      "sandbox": {
        "requireMode": ["all"],
        "allowBackends": ["docker"],
      },
      "dataHandling": {
        "memory": {
          "denySessionTranscriptIndexing": true,
        },
      },
    },
    "shell-sandbox": {
      "agentIds": ["shell-agent"],
      "sandbox": {
        "allowBackends": ["openshell"],
        "containers": {
          "requireReadOnlyMounts": false,
        },
      },
    },
    "telegram-ingress": {
      "channelIds": ["telegram"],
      "ingress": {
        "channels": {
          "allowDmPolicies": ["pairing"],
          "denyOpenGroups": true,
          "requireMentionInGroups": true,
        },
      },
    },
  },
}
```

上記のように、各スコープが異なるフィールドを管理する場合、同じエージェントを複数のスコープに含めることができます。
同じエージェントに対してスコープ付きフィールドを繰り返す場合は、同等以上に
厳格でなければなりません。より緩い重複宣言は拒否されます（許可リストは
部分集合、拒否リストは上位集合、必須の真偽値は固定です）。

コンテナ態勢ルール（`sandbox.containers.*`）は、
照合されたエージェントのサンドボックスバックエンドが公開できる証拠に対してのみ検査されます。バックエンドが
有効化されたルールを観測できない場合、Policy は合格とする代わりに
`policy/sandbox-container-posture-unobservable` を報告します。コンテナルールは、
そのルールを公開できるバックエンドを使用するエージェントグループに限定してください。

トップレベルの `ingress.session.requireDmScope` はグローバルなままです。`session.dmScope` は
チャンネルに帰属可能な証拠ではないため、`channelIds` でスコープを限定できません。

`policy.jsonc` に存在するすべてのスコープは、有効かつ適用可能でなければなりません。

#### チャンネル

| ポリシーフィールド                         | 観測される状態                          | 使用する場合                                                     |
| ------------------------------------ | --------------------------------------- | ------------------------------------------------------------ |
| `channels.denyRules[].when.provider` | `channels.*` のプロバイダーと有効化状態 | `telegram` などのプロバイダーから設定されたチャンネルを拒否する場合。 |
| `channels.denyRules[].reason`        | 検出結果のメッセージと修復ヒントのコンテキスト | プロバイダーが拒否される理由を説明する場合。                          |

#### MCP サーバー

| ポリシーフィールド        | 観測される状態      | 使用する場合                                                   |
| ------------------- | ------------------- | ---------------------------------------------------------- |
| `mcp.servers.allow` | `mcp.servers.*` の ID | 設定されたすべての MCP サーバーが許可リストに含まれることを必須にする場合。 |
| `mcp.servers.deny`  | `mcp.servers.*` の ID | 設定された特定の MCP サーバー ID を拒否する場合。                   |

#### モデルプロバイダー

| ポリシーフィールド             | 観測される状態                                   | 使用する場合                                                                        |
| ------------------------ | ------------------------------------------------ | ------------------------------------------------------------------------------- |
| `models.providers.allow` | `models.providers.*` の ID と選択されたモデル参照 | 設定されたプロバイダーと選択されたモデル参照で、承認済みプロバイダーの使用を必須にする場合。 |
| `models.providers.deny`  | `models.providers.*` の ID と選択されたモデル参照 | 設定されたプロバイダーと選択されたモデル参照を、プロバイダー ID に基づいて拒否する場合。               |

#### ネットワーク

| ポリシーフィールド                   | 観測された状態                      | 使用する場合                                                           |
| ------------------------------ | ----------------------------------- | ------------------------------------------------------------------ |
| `network.privateNetwork.allow` | プライベートネットワーク向け SSRF エスケープハッチ | プライベートネットワークアクセスを無効のままにするには、`false` に設定します。 |

#### メッセージルーティング

| ポリシーフィールド                        | 観測された状態                                      | 使用する場合                                                               |
| ----------------------------------- | --------------------------------------------------- | ---------------------------------------------------------------------- |
| `routing.requireBindings`           | ACP バインディングを除くチャネルルートバインディング      | メッセージルーティングバインディングを少なくとも 1 つ必須にします。                          |
| `routing.requireConfiguredChannels` | バインディングのチャネル ID と設定済みの `channels.*` ID | 古くなった、またはスペルが誤っているバインディングチャネル ID を検出します。                        |
| `routing.probes[].route`            | OpenClaw の公開ルートリゾルバー                  | メッセージを送信せずに、代表的な受信ルートを記述します。     |
| `routing.probes[].expect.agentId`   | 解決されたエージェント ID                                   | ルートがレビュー済みエージェントに到達することを必須にします。                         |
| `routing.probes[].expect.matchedBy` | リゾルバーの一致種別                                 | ピア、アカウント、チャネル、またはその他のレビュー済みバインディングの具体性を必須にします。 |

プローブ ID は一意でなければなりません。ルートは `channel`、任意の `accountId`、
`peer`、`parentPeer`、`guildId`、`teamId`、および `memberRoleIds` をサポートします。ピア種別は
`direct`、`group`、および `channel` です。`matchedBy` には、
`binding.peer`、`binding.account`、`binding.channel`、
`default` など、1 つ以上のランタイム一致種別を含めることができます。

ルーティングチェックは適合性チェックにすぎません。起動、
メッセージ配信、バインディングの優先順位、フォールバック動作は変更しません。バインディングを自動的に変更すると
プライベートメッセージが別の宛先に転送される可能性があるため、検出事項には
オペレーターによるレビューが必要です。

#### イングレスとチャネルアクセス

| ポリシーフィールド                              | 観測された状態                                                 | 使用する場合                                                           |
| ----------------------------------------- | -------------------------------------------------------------- | ------------------------------------------------------------------ |
| `ingress.session.requireDmScope`          | `session.dmScope`                                              | レビュー済みのダイレクトメッセージ分離スコープを必須にします。                 |
| `ingress.channels.allowDmPolicies`        | `channels.*.dmPolicy` および従来のチャネル DM ポリシーフィールド      | レビュー済みのダイレクトメッセージチャネルポリシーのみを許可します。               |
| `ingress.channels.denyOpenGroups`         | チャネル、アカウント、およびグループのイングレスポリシー                     | 設定済みのチャネルとアカウントに対するオープンなグループイングレスを拒否します。      |
| `ingress.channels.requireMentionInGroups` | チャネル、アカウント、グループ、ギルド、およびネストされたメンションゲート設定 | グループイングレスがオープンまたはメンションゲート付きの場合、メンションゲートを必須にします。 |

#### Gateway

| ポリシーフィールド                            | 観測された状態                                 | 使用する場合                                                                             |
| --------------------------------------- | ---------------------------------------------- | ------------------------------------------------------------------------------------ |
| `gateway.exposure.allowNonLoopbackBind` | `gateway.bind`                                 | local loopback への Gateway バインディングを必須にするには、`false` に設定します。                                  |
| `gateway.exposure.allowTailscaleFunnel` | Tailscale serve/funnel の Gateway ポスチャー         | Tailscale Funnel への公開を拒否するには、`false` に設定します。                                    |
| `gateway.auth.requireAuth`              | `gateway.auth.mode`                            | 無効化された Gateway 認証を拒否するには、`true` に設定します。                                       |
| `gateway.auth.requireExplicitRateLimit` | `gateway.auth.rateLimit`                       | 明示的な認証レート制限設定を必須にするには、`true` に設定します。                            |
| `gateway.controlUi.allowInsecure`       | Control UI の安全でない認証、デバイス、オリジンの切り替え設定 | 安全でない Control UI 公開の切り替え設定を拒否するには、`false` に設定します。                         |
| `gateway.remote.allow`                  | リモート Gateway モード／設定                     | リモート Gateway モードを拒否するには、`false` に設定します。                                          |
| `gateway.http.denyEndpoints`            | Gateway HTTP API エンドポイント                     | `chatCompletions` や `responses` などのエンドポイント ID を拒否します。                          |
| `gateway.http.requireUrlAllowlists`     | Gateway HTTP URL 取得入力                  | URL 取得入力で URL 許可リストを必須にするには、`true` に設定します。                         |
| `gateway.nodes.denyCommands`            | `gateway.nodes.commands.deny`                  | `system.run` などの正確な Node コマンド ID が OpenClaw 設定で拒否されることを必須にします。 |

`gateway.nodes.denyCommands` は、大文字と小文字を区別する完全一致のポリシー拒否スーパーセットルールです。
特権 Node コマンドが OpenClaw 設定で明示的に
拒否されていることをポリシーで証明する必要がある場合に使用します。特権
Node コマンドを意図的に許可するデプロイでは、`gateway.nodes.commands.allow` のみに依存せず、
レビュー後に `policy.jsonc` を更新する必要があります。

#### エージェントワークスペース

| ポリシーフィールド                     | 観測された状態                                                                           | 使用する場合                                                                                 |
| -------------------------------- | ---------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------- |
| `agents.workspace.allowedAccess` | `agents.defaults.sandbox.workspaceAccess` および `agents.entries.*.sandbox.workspaceAccess` | `none` や `ro` などのサンドボックスワークスペースアクセス値のみを許可します。                       |
| `agents.workspace.denyTools`     | グローバルおよびエージェント単位のツール拒否設定                                                    | 変更ツール（`exec`、`process`、`write`、`edit`、`apply_patch`）が拒否されることを必須にします。 |

#### サンドボックスポスチャー

| ポリシーフィールド                                          | 観測された状態                                          | 使用する場合                                                       |
| ----------------------------------------------------- | ------------------------------------------------------- | -------------------------------------------------------------- |
| `sandbox.requireMode`                                 | `agents.defaults.sandbox.mode` およびエージェント単位のモード       | `all` や `non-main` などのレビュー済みサンドボックスモードのみを許可します。 |
| `sandbox.allowBackends`                               | `agents.defaults.sandbox.backend` およびエージェント単位のバックエンド | `docker` などのレビュー済みサンドボックスバックエンドのみを許可します。         |
| `sandbox.containers.denyHostNetwork`                  | コンテナベースのサンドボックス／ブラウザネットワークモード           | ホストネットワークモードを拒否します。                                        |
| `sandbox.containers.denyContainerNamespaceJoin`       | コンテナベースのサンドボックス／ブラウザネットワークモード           | 別のコンテナのネットワーク名前空間への参加を拒否します。              |
| `sandbox.containers.requireReadOnlyMounts`            | コンテナベースのサンドボックス／ブラウザマウントモード             | マウントが読み取り専用であることを必須にします。                                |
| `sandbox.containers.denyContainerRuntimeSocketMounts` | コンテナベースのサンドボックス／ブラウザマウント対象          | コンテナランタイムソケットのマウントを拒否します。                          |
| `sandbox.containers.denyUnconfinedProfiles`           | コンテナセキュリティプロファイルのポスチャー                      | 制限のないコンテナセキュリティプロファイルを拒否します。                   |
| `sandbox.browser.requireCdpSourceRange`               | サンドボックスブラウザの CDP ソース範囲                        | ブラウザの CDP 公開でソース範囲を宣言することを必須にします。        |

ポリシーでは、欠落した `sandbox.mode` を暗黙のデフォルト値 `off` として扱うため、
`sandbox.requireMode` は、新規または未設定のサンドボックスを
`["all"]` などの許可リスト外として報告します。

#### データ処理

| ポリシーフィールド                                        | 観測された状態                                                                                     | 使用する場合                                                               |
| --------------------------------------------------- | -------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------- |
| `dataHandling.sensitiveLogging.requireRedaction`    | `logging.redactSensitive`                                                                          | `logging.redactSensitive: "off"` を拒否するには、`true` に設定します。              |
| `dataHandling.telemetry.denyContentCapture`         | `diagnostics.otel.captureContent`                                                                  | テレメトリコンテンツのキャプチャを拒否するには、`true` に設定します。                     |
| `dataHandling.retention.requireSessionMaintenance`  | `session.maintenance.mode`                                                                         | 有効なセッションメンテナンスモード `enforce` を必須にするには、`true` に設定します。 |
| `dataHandling.memory.denySessionTranscriptIndexing` | `memory.qmd.sessions.enabled`、`memory.search.experimental.sessionMemory`、およびエージェント単位のオーバーライド | セッショントランスクリプトのメモリへのインデックス作成を拒否するには、`true` に設定します。       |

#### シークレット

| ポリシーフィールド                      | 観測された状態                                           | 使用する場合                                                                |
| --------------------------------- | -------------------------------------------------------- | ----------------------------------------------------------------------- |
| `secrets.requireManagedProviders` | 設定の SecretRef と `secrets.providers.*` 宣言 | SecretRef が宣言済みプロバイダーを指すことを必須にするには、`true` に設定します。     |
| `secrets.denySources`             | シークレットプロバイダーのソースと SecretRef のソース            | `exec`、`file`、または別の設定済みソース名などのソースを拒否します。 |
| `secrets.allowInsecureProviders`  | 安全でないシークレットプロバイダーのポスチャーフラグ                   | 安全でないポスチャーを有効にするプロバイダーを拒否するには、`false` に設定します。      |

#### Exec 承認

Exec 承認チェックは、ランタイムの `exec-approvals.json` アーティファクトを読み取ります。
デフォルトでは `~/.openclaw/exec-approvals.json`、`OPENCLAW_STATE_DIR` が設定されている場合は
`$OPENCLAW_STATE_DIR/exec-approvals.json` です。
`execApprovals.defaults.*` または `execApprovals.agents.*` 配下の
ポスチャールールでは、読み取り可能なアーティファクトの証拠が必要です。アーティファクトが欠落または無効な場合、
ベストエフォートで合格とするのではなく、観測不能な証拠として報告されます。読み取り可能になると、省略された
フィールドはランタイムのデフォルト値を継承します。欠落した `defaults.security` は `full` となり、
欠落したエージェントセキュリティもそのデフォルト値を継承します。証拠には `defaults`、
`agents.*`、`agents.*.allowlist[].pattern`、任意の `argPattern`、有効な
`autoAllowSkills` ポスチャー、およびエントリソースが含まれます。ソケットパス／トークン、
`commandText`、`lastUsedCommand`、解決済みパス、タイムスタンプは決して含まれません。

| ポリシーフィールド                                | 観測された状態                                                                         | 使用する場合                                                                                |
| ------------------------------------------- | -------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------- |
| `execApprovals.requireFile`                 | アクティブなランタイムの `exec-approvals.json` パス                                              | 承認アーティファクトが存在し、解析できることを必須にするには、`true` に設定します。                     |
| `execApprovals.defaults.allowSecurity`      | `defaults.security`（デフォルトは `full`）                                              | 承認済みのデフォルト承認セキュリティモードのみを許可します。                                    |
| `execApprovals.agents.allowSecurity`        | `agents.*.security`（デフォルトを継承）                                               | エージェントごとに有効な承認済みの承認セキュリティモードのみを許可します。                        |
| `execApprovals.agents.allowAutoAllowSkills` | `defaults.autoAllowSkills` および `agents.*.autoAllowSkills`（ランタイムのデフォルトを継承） | Skills の CLI を暗黙的に承認せず、厳格な手動許可リストを必須にするには、`false` に設定します。 |
| `execApprovals.agents.allowlist.expected`   | 集約された `agents.*.allowlist[]` パターンおよび任意の argPattern エントリ               | 承認許可リストがレビュー済みのパターンセットと一致することを必須にします。                      |

例: 承認アーティファクトを必須にし、制限の緩いデフォルトを拒否して、選択したエージェントに対して
レビュー済みの exec 承認態勢のみを許可します。

```jsonc
{
  "execApprovals": {
    "requireFile": true,
    "defaults": {
      // セキュリティモード: "deny"、"allowlist"、または "full"。
      // このデフォルトでは、厳格に制限された deny 態勢のみが許可されます。
      "allowSecurity": ["deny"],
    },
  },
  "scopes": {
    "restricted-shell": {
      "agentIds": ["family-agent", "groups-agent"],
      "execApprovals": {
        "agents": {
          // 選択したエージェントは、レビュー済みの allowlist 態勢を使用できますが、"full" は使用できません。
          "allowSecurity": ["allowlist"],
          // false は、Skills の CLI が autoAllowSkills によって暗黙的に承認されるのではなく、
          // レビュー済みの許可リストに含まれている必要があることを意味します。
          "allowAutoAllowSkills": false,
          "allowlist": {
            "expected": [
              // 単純なエントリ: argPattern を伴わない、レビュー済みの完全一致実行可能ファイルパターン。
              "travel-hub",
              // 制約付きエントリ: パターンとレビュー済みの引数正規表現。
              { "pattern": "calendar-cli", "argPattern": "^sync\\b" },
              "/bin/date",
            ],
          },
        },
      },
    },
  },
}
```

#### 認証プロファイル

| ポリシーフィールド                    | 観測された状態                               | 使用する場合                                                                                   |
| ------------------------------- | -------------------------------------------- | ------------------------------------------------------------------------------------------ |
| `auth.profiles.requireMetadata` | `auth.profiles.*` のプロバイダーおよびモードのメタデータ | 設定の認証プロファイルで、`provider` や `mode` などのメタデータキーを必須にします。               |
| `auth.profiles.allowModes`      | `auth.profiles.*.mode`                       | `api_key`、`aws-sdk`、`oauth`、`token` など、サポートされている認証プロファイルモードのみを許可します。 |

#### ツールメタデータ

| ポリシーフィールド            | 観測された状態                   | 使用する場合                                                                                   |
| ----------------------- | -------------------------------- | ------------------------------------------------------------------------------------------ |
| `tools.requireMetadata` | 管理対象の `TOOLS.md` 宣言 | 管理対象ツールが、`risk`、`sensitivity`、`owner` などのメタデータキーを宣言することを必須にします。 |

#### ツール態勢

| ポリシーフィールド                    | 観測された状態                                              | 使用する場合                                                                                                 |
| ------------------------------- | ----------------------------------------------------------- | -------------------------------------------------------------------------------------------------------- |
| `tools.profiles.allow`          | `tools.profile` および `agents.entries.*.tools.profile`        | `minimal`、`messaging`、`coding` などのツールプロファイル ID のみを許可します。                                 |
| `tools.fs.requireWorkspaceOnly` | `tools.fs.workspaceOnly` およびエージェントごとの `tools.fs` オーバーライド | ファイルシステムツールの態勢をワークスペース内のみに制限するには、`true` に設定します。                                         |
| `tools.exec.allowSecurity`      | `tools.exec.security` およびエージェントごとの exec セキュリティ           | `deny` や `allowlist` などの exec セキュリティモードのみを許可します。                                            |
| `tools.exec.requireAsk`         | `tools.exec.ask` およびエージェントごとの exec 確認モード                | `always` などの承認態勢を必須にします。                                                               |
| `tools.exec.allowHosts`         | `tools.exec.host` およびエージェントごとの exec ホストルーティング           | `sandbox` などの exec ホストルーティングモードのみを許可します。                                                    |
| `tools.elevated.allow`          | `tools.elevated.enabled` およびエージェントごとの昇格態勢     | 昇格ツールモードを無効のままにすることを必須にするには、`false` に設定します。                                           |
| `tools.alsoAllow.expected`      | `tools.alsoAllow` およびエージェントごとの `tools.alsoAllow`           | `alsoAllow` エントリとの完全一致を必須にし、不足している、または予期しない追加のツール権限を報告します。                 |
| `tools.denyTools`               | `tools.deny` および `agents.entries.*.tools.deny`              | 設定されたツール拒否リストに、`group:runtime` や `group:fs` などのツール ID またはグループが含まれることを必須にします。 |

## チェックの実行

作成中にポリシーのみのチェックを実行します。

```bash
openclaw policy check
openclaw policy check --json
openclaw policy check --severity-min error
```

`policy check` はポリシーチェックセットのみを実行し、証拠、検出事項、
およびアテステーションハッシュを出力します。Policy Plugin が有効な場合は、
同じ検出事項が `openclaw doctor --lint` にも表示されます。

オペレーターのポリシーファイルを作成済みのベースラインと比較します。

```bash
openclaw policy compare --baseline official.policy.jsonc
openclaw policy compare --baseline official.policy.jsonc --policy policy.jsonc --json
```

`policy compare` はポリシーファイルの構文をポリシーファイルの構文と照合します。
ランタイムの状態、証拠、認証情報、またはシークレットは検査しません。スコープ付きオーバーレイを
管理するものと同じルールメタデータを使用します。許可リストは同等かより狭く、
拒否リストは同等かより広く、必須のブール値はその値を維持する必要があります。
順序付き文字列は設定された順序のより厳格な側にのみ移動でき、完全一致リストは一致する必要があります。
ベースラインには組織が作成したポリシーを使用でき、チェック対象のポリシーには、より厳格な値や
追加のルールを含めることができます。トップレベルのチェック対象ルールは、同等以上に制限的であれば、
スコープ付きのベースラインルールを満たすことができます。ファイル間でスコープ名を一致させる必要はありません。
比較はセレクター（`agentIds`/`channelIds`）とフィールドをキーとして行われます。
ルーティングプローブでは、すべてのベースラインプローブ ID が、同じルートおよび想定エージェントとともに
維持される必要があります。チェック対象ポリシーでは、プローブの追加や `matchedBy` の範囲縮小は可能ですが、
プローブの削除、ルートやエージェントの変更、または許容される一致種別の拡大は、より弱い設定です。

正常な比較（`--json`）:

```json
{
  "ok": true,
  "baselinePath": "official.policy.jsonc",
  "policyPath": "policy.jsonc",
  "rulesChecked": 3,
  "findings": []
}
```

正常な `policy check --json` の出力には、オペレーターまたは
監督者が記録できる安定したハッシュが含まれます。

```json
{
  "ok": true,
  "attestation": {
    "policy": {
      "path": "policy.jsonc",
      "hash": "sha256:..."
    },
    "workspace": {
      "scope": "policy",
      "hash": "sha256:..."
    },
    "findingsHash": "sha256:...",
    "attestationHash": "sha256:..."
  },
  "checksRun": 5,
  "checksSkipped": 0,
  "findings": []
}
```

## ポリシーの設定

ポリシー設定は `plugins.entries.policy.config` の下にあります。

```jsonc
{
  "plugins": {
    "entries": {
      "policy": {
        "enabled": true,
        "config": {
          "enabled": true,
          "path": "policy.jsonc",
          "workspaceRepairs": false,
          "expectedHash": "sha256:...",
          "expectedAttestationHash": "sha256:...",
        },
      },
    },
  },
}
```

| 設定                   | 目的                                                         |
| ------------------------- | --------------------------------------------------------------- |
| `enabled`                 | `policy.jsonc` が存在する前でもポリシーチェックを有効にします。         |
| `workspaceRepairs`        | `doctor --fix` がポリシー管理対象のワークスペース設定を編集できるようにします。 |
| `expectedHash`            | 承認済みポリシーアーティファクトに対する任意のハッシュロック。            |
| `expectedAttestationHash` | 最後に受け入れられた正常なポリシーチェックに対する任意のハッシュロック。    |
| `path`                    | ポリシーアーティファクトのワークスペース相対位置。             |

Plugin をインストールしたままワークスペースのポリシーチェックを無効にするには、
`plugins.entries.policy.config.enabled` を `false` に設定します。

## ポリシー状態の受け入れ

JSON 出力の例:

```json
{
  "ok": true,
  "attestation": {
    "checkedAt": "2026-05-10T20:00:00.000Z",
    "policy": {
      "path": "policy.jsonc",
      "hash": "sha256:..."
    },
    "workspace": {
      "scope": "policy",
      "hash": "sha256:..."
    },
    "findingsHash": "sha256:...",
    "attestationHash": "sha256:..."
  },
  "evidence": {
    "channels": [
      {
        "id": "telegram",
        "provider": "telegram",
        "source": "oc://openclaw.config/channels/telegram",
        "enabled": false
      }
    ],
    "mcpServers": [
      {
        "id": "docs",
        "transport": "stdio",
        "source": "oc://openclaw.config/mcp/servers/docs",
        "command": "npx"
      }
    ],
    "modelProviders": [
      {
        "id": "openai",
        "source": "oc://openclaw.config/models/providers/openai"
      }
    ],
    "modelRefs": [
      {
        "ref": "openai/gpt-5.6-sol",
        "provider": "openai",
        "model": "gpt-5.6-sol",
        "source": "oc://openclaw.config/agents/defaults/model"
      }
    ],
    "network": [
      {
        "id": "browser-private-network",
        "source": "oc://openclaw.config/browser/ssrfPolicy/dangerouslyAllowPrivateNetwork",
        "value": false
      }
    ],
    "gatewayExposure": [
      {
        "id": "gateway-bind",
        "kind": "bind",
        "source": "oc://openclaw.config/gateway/bind",
        "value": "loopback",
        "nonLoopback": false,
        "explicit": true
      }
    ],
    "agentWorkspace": [
      {
        "id": "agents-defaults-workspace-access",
        "kind": "workspaceAccess",
        "source": "oc://openclaw.config/agents/defaults/sandbox/workspaceAccess",
        "scope": "defaults",
        "value": "ro",
        "sandboxMode": "all",
        "sandboxModeSource": "oc://openclaw.config/agents/defaults/sandbox/mode",
        "sandboxEnabled": true,
        "explicit": true
      },
      {
        "id": "agents-defaults-tool-exec",
        "kind": "toolDeny",
        "source": "oc://openclaw.config/tools/deny",
        "scope": "defaults",
        "tool": "exec",
        "denied": true,
        "explicit": true
      }
    ],
    "secrets": [
      {
        "id": "vault",
        "kind": "provider",
        "source": "oc://openclaw.config/secrets/providers/vault",
        "providerSource": "env"
      },
      {
        "id": "oc://openclaw.config/models/providers/openai/apiKey",
        "kind": "input",
        "source": "oc://openclaw.config/models/providers/openai/apiKey",
        "provenance": "secretRef",
        "refSource": "env",
        "refProvider": "vault"
      }
    ],
    "authProfiles": [
      {
        "id": "github",
        "source": "oc://openclaw.config/auth/profiles/github",
        "validMetadata": true,
        "provider": "github",
        "mode": "token"
      }
    ],
    "tools": [
      {
        "id": "deploy",
        "source": "oc://TOOLS.md/tools/deploy",
        "line": 12,
        "risk": "critical",
        "sensitivity": "restricted",
        "capabilities": ["IRREVERSIBLE_EXTERNAL"]
      }
    ]
  },
  "checksRun": 30,
  "checksSkipped": 0,
  "findings": []
}
```

`attestation.policy.hash` は作成されたルール成果物を識別します。`evidence` は
チェックで使用された、観測時点の OpenClaw の状態を記録し、
`workspace.hash` はその証拠ペイロードを識別します。`findingsHash` は
正確な検出事項の集合を識別します。`checkedAt` はチェックの実行日時を記録します。
`attestationHash` は安定したクレーム（ポリシーハッシュ、証拠ハッシュ、
検出事項ハッシュ、およびクリーン／ダーティ状態）を識別し、意図的に `checkedAt` を
除外するため、同じポリシー状態からは常に同じアテステーションハッシュが生成されます。
これら4つの値が、1回のポリシーチェックの監査タプルを構成します。

Gateway またはスーパーバイザーがポリシーを使用してランタイムアクションをブロック、
承認、または注釈付けする場合、直近のクリーンなチェックのアテステーションハッシュを
記録する必要があります。`checkedAt` は監査ログ用に JSON 出力へ残りますが、
安定したハッシュには含まれません。

ポリシー状態を受け入れるためのライフサイクル：

1. `policy.jsonc` を作成またはレビューします。
2. `openclaw policy check --json` を実行します。
3. クリーンな場合、`attestation.policy.hash` を `expectedHash` として記録します。
4. `attestation.attestationHash` を `expectedAttestationHash` として記録します。
5. CI またはリリースゲートで `openclaw doctor --lint` を再実行します。

ポリシールールを意図的に変更した場合は、クリーンなチェックから受け入れ済みの
両方のハッシュを更新します。ワークスペース設定のみを変更した場合（ポリシーは同じ）、
通常は `expectedAttestationHash` のみが変更されます。

`agents.workspace` ルールを有効化またはアップグレードすると、`agentWorkspace` の証拠が
ワークスペースハッシュとアテステーションハッシュに追加されます。有効化後に新しい証拠を
レビューし、受け入れ済みのアテステーションハッシュを更新してください。ツール態勢ルールを
有効化またはアップグレードした場合も、同様に `toolPosture` の証拠が追加されます。

`openclaw policy watch` はチェックを再実行し、現在の証拠が
`expectedAttestationHash` と一致しなくなった場合に報告します：

```bash
openclaw policy watch --json
```

単一のドリフト評価が必要な CI またはスクリプトでは、`--once` を使用します。
`--once` を指定しない場合、デフォルトでは2秒ごとにポーリングします。
間隔を変更するには `--interval-ms` を使用します。

## 検出事項

| チェック ID                                              | 検出事項                                                                          |
| -------------------------------------------------------- | --------------------------------------------------------------------------------- |
| `policy/policy-jsonc-missing`                            | ポリシーは有効ですが、`policy.jsonc` がありません。                               |
| `policy/policy-jsonc-invalid`                            | ポリシーを解析できないか、不正なルールエントリが含まれています。                  |
| `policy/policy-hash-mismatch`                            | ポリシーが設定済みの `expectedHash` と一致しません。                               |
| `policy/attestation-hash-mismatch`                       | 現在のポリシー証拠が、受理済みの証明と一致しなくなっています。                    |
| `policy/policy-conformance-invalid`                      | ベースラインまたは検査対象のポリシーファイルに無効な比較構文があります。          |
| `policy/policy-conformance-missing`                      | 検査対象のポリシーファイルに、ベースラインポリシーファイルで必須のルールがありません。 |
| `policy/policy-conformance-weaker`                       | 検査対象のポリシーファイルの値が、ベースラインポリシーファイルより弱くなっています。 |
| `policy/channels-denied-provider`                        | 有効なチャンネルがチャンネル拒否ルールに一致しています。                          |
| `policy/mcp-denied-server`                               | 設定済みの MCP サーバーがポリシーによって拒否されています。                       |
| `policy/mcp-unapproved-server`                           | 設定済みの MCP サーバーが許可リストに含まれていません。                           |
| `policy/models-denied-provider`                          | 設定済みのモデルプロバイダーまたはモデル参照が、拒否されたプロバイダーを使用しています。 |
| `policy/models-unapproved-provider`                      | 設定済みのモデルプロバイダーまたはモデル参照が許可リストに含まれていません。      |
| `policy/network-private-access-enabled`                  | ポリシーで禁止されているにもかかわらず、プライベートネットワーク向けの SSRF 回避機構が有効です。 |
| `policy/routing-bindings-required`                       | ポリシーでチャンネルルートのバインディングが必須ですが、何も設定されていません。  |
| `policy/routing-binding-channel-unconfigured`            | ルートバインディングで指定されたチャンネルが `channels.*` にありません。         |
| `policy/routing-agent-mismatch`                          | 記述されたルートが別のエージェントに解決されます。                                |
| `policy/routing-match-kind-mismatch`                     | 記述されたルートが予期しないバインディング詳細度で一致します。                    |
| `policy/ingress-dm-policy-unapproved`                    | チャンネルの DM ポリシーがポリシー許可リストに含まれていません。                  |
| `policy/ingress-dm-scope-unapproved`                     | `session.dmScope` がポリシーで必須の DM 分離スコープと一致しません。               |
| `policy/ingress-open-groups-denied`                      | ポリシーでオープングループへの流入が禁止されているにもかかわらず、チャンネルのグループポリシーが `open` です。 |
| `policy/ingress-group-mention-required`                  | ポリシーでメンションゲートが必須にもかかわらず、チャンネルまたはグループのエントリで無効化されています。 |
| `policy/gateway-non-loopback-bind`                       | ポリシーで禁止されているにもかかわらず、Gateway のバインド構成で非ループバックへの公開が許可されています。 |
| `policy/gateway-auth-disabled`                           | ポリシーで認証が必須にもかかわらず、Gateway 認証が無効です。                      |
| `policy/gateway-rate-limit-missing`                      | ポリシーで明示が必須にもかかわらず、Gateway 認証のレート制限構成が明示されていません。 |
| `policy/gateway-control-ui-insecure`                     | Gateway Control UI の安全でない公開を許可する切り替えが有効です。                 |
| `policy/gateway-tailscale-funnel`                        | ポリシーで禁止されているにもかかわらず、Gateway の Tailscale Funnel 公開が有効です。 |
| `policy/gateway-remote-enabled`                          | ポリシーで禁止されているにもかかわらず、Gateway のリモートモードが有効です。      |
| `policy/gateway-http-endpoint-enabled`                   | ポリシーで拒否されている Gateway HTTP API エンドポイントが有効です。              |
| `policy/gateway-http-url-fetch-unrestricted`             | Gateway HTTP の URL 取得入力に、必須の URL 許可リストがありません。               |
| `policy/gateway-node-command-denied`                     | ポリシーで拒否された Node コマンドが OpenClaw 設定では拒否されていません。        |
| `policy/agents-workspace-access-denied`                  | エージェントのサンドボックスモードまたはワークスペースアクセスがポリシー許可リストに含まれていません。 |
| `policy/agents-tool-not-denied`                          | エージェントまたはデフォルト設定で、ポリシーにより拒否が必須のツールが拒否されていません。 |
| `policy/tools-profile-unapproved`                        | 設定済みのグローバルまたはエージェント別ツールプロファイルが許可リストに含まれていません。 |
| `policy/tools-fs-workspace-only-required`                | ファイルシステムツールが、パスをワークスペース内のみに制限する構成になっていません。 |
| `policy/tools-exec-security-unapproved`                  | Exec セキュリティモードがポリシー許可リストに含まれていません。                   |
| `policy/tools-exec-ask-unapproved`                       | Exec 確認モードがポリシー許可リストに含まれていません。                           |
| `policy/tools-exec-host-unapproved`                      | Exec ホストルーティングがポリシー許可リストに含まれていません。                   |
| `policy/tools-elevated-enabled`                          | ポリシーで禁止されているにもかかわらず、昇格ツールモードが有効です。              |
| `policy/tools-also-allow-missing`                        | 設定済みの `alsoAllow` リストに、ポリシーで必須のエントリがありません。          |
| `policy/tools-also-allow-unexpected`                     | 設定済みの `alsoAllow` リストに、ポリシーで想定されていないエントリが含まれています。 |
| `policy/tools-required-deny-missing`                     | グローバルまたはエージェント別のツール拒否リストに、拒否が必須のツールが含まれていません。 |
| `policy/sandbox-mode-unapproved`                         | サンドボックスモードがポリシー許可リストに含まれていません。                     |
| `policy/sandbox-backend-unapproved`                      | サンドボックスバックエンドがポリシー許可リストに含まれていません。               |
| `policy/sandbox-container-posture-unobservable`          | コンテナ構成ルールが、それを監視できないバックエンドで有効になっています。        |
| `policy/sandbox-container-host-network-denied`           | コンテナベースのサンドボックスまたはブラウザーがホストネットワークモードを使用しています。 |
| `policy/sandbox-container-namespace-join-denied`         | コンテナベースのサンドボックスまたはブラウザーが別のコンテナ名前空間に参加しています。 |
| `policy/sandbox-container-mount-mode-required`           | コンテナベースのサンドボックスまたはブラウザーのマウントが読み取り専用ではありません。 |
| `policy/sandbox-container-runtime-socket-mount`          | コンテナベースのサンドボックスまたはブラウザーのマウントによって、コンテナランタイムソケットが公開されています。 |
| `policy/sandbox-container-unconfined-profile`            | ポリシーで禁止されているにもかかわらず、コンテナサンドボックスのプロファイルが制限なしです。 |
| `policy/sandbox-browser-cdp-source-range-missing`        | ポリシーで必須にもかかわらず、サンドボックスブラウザーの CDP ソース範囲がありません。 |
| `policy/data-handling-redaction-disabled`                | ポリシーで必須にもかかわらず、機密ログの秘匿化が無効です。                        |
| `policy/data-handling-telemetry-content-capture`         | ポリシーで禁止されているにもかかわらず、テレメトリのコンテンツ取得が有効です。    |
| `policy/data-handling-session-retention-not-enforced`    | ポリシーで必須にもかかわらず、セッション保持のメンテナンスが強制されていません。  |
| `policy/data-handling-session-transcript-memory-enabled` | ポリシーで禁止されているにもかかわらず、セッショントランスクリプトのメモリインデックス作成が有効です。 |
| `policy/secrets-unmanaged-provider`                      | 設定の SecretRef が、`secrets.providers` で宣言されていないプロバイダーを参照しています。 |
| `policy/secrets-denied-provider-source`                  | 設定のシークレットプロバイダーまたは SecretRef が、ポリシーで拒否されたソースを使用しています。 |
| `policy/secrets-insecure-provider`                       | ポリシーで禁止されているにもかかわらず、シークレットプロバイダーが安全でない構成を明示的に選択しています。 |
| `policy/auth-profile-invalid-metadata`                   | 設定の認証プロファイルに、有効なプロバイダーまたはモードのメタデータがありません。 |
| `policy/auth-profile-unapproved-mode`                    | 設定の認証プロファイルモードがポリシー許可リストに含まれていません。              |
| `policy/exec-approvals-missing`                          | ポリシーで `exec-approvals.json` が必須ですが、アーティファクトがありません。     |
| `policy/exec-approvals-invalid`                          | 設定済みの Exec 承認アーティファクトを解析できません。                            |
| `policy/exec-approvals-default-security-unapproved`      | Exec 承認のデフォルトが、ポリシー許可リストに含まれないセキュリティモードを使用しています。 |
| `policy/exec-approvals-agent-security-unapproved`        | エージェント別の実効 Exec 承認セキュリティモードが許可リストに含まれていません。 |
| `policy/exec-approvals-auto-allow-skills-enabled`        | ポリシーで禁止されているにもかかわらず、Exec 承認エージェントが Skills の CLI を暗黙的に自動許可します。 |
| `policy/exec-approvals-allowlist-missing`                | 承認許可リストに、ポリシーで必須のパターンがありません。                          |
| `policy/exec-approvals-allowlist-unexpected`             | 承認許可リストに、ポリシーで想定されていないパターンが含まれています。            |
| `policy/tools-missing-risk-level`                        | 管理対象ツールの宣言にリスクメタデータがありません。                             |
| `policy/tools-unknown-risk-level`                        | 管理対象ツールの宣言で不明なリスク値が使用されています。                         |
| `policy/tools-missing-sensitivity-token`                 | 管理対象ツールの宣言に機密度メタデータがありません。                              |
| `policy/tools-missing-owner`                             | 管理対象ツールの宣言に所有者メタデータがありません。                             |
| `policy/tools-unknown-sensitivity-token`                 | 管理対象ツールの宣言で不明な機密度値が使用されています。                          |

検出事項には、`target`（準拠していない、観測されたワークスペース内の対象）と
`requirement`（その対象を検出事項とした、記述済みのルール）の両方を含めることができます。
現在、どちらも `oc://` アドレス文字列ですが、フィールド名はアドレス形式ではなく
ポリシー上の役割を表します。

検出事項の例：

```json
{
  "checkId": "policy/channels-denied-provider",
  "severity": "error",
  "message": "チャンネル 'telegram' は拒否されたプロバイダー 'telegram' を使用しています。",
  "source": "policy",
  "path": "openclaw config",
  "ocPath": "oc://openclaw.config/channels/telegram",
  "target": "oc://openclaw.config/channels/telegram",
  "requirement": "oc://policy.jsonc/channels/denyRules/#0",
  "fixHint": "Telegram はこのワークスペースでは承認されていません。"
}
```

```json
{
  "checkId": "policy/tools-missing-risk-level",
  "severity": "error",
  "message": "TOOLS.md のツール 'deploy' に明示的なリスク分類がありません。",
  "source": "policy",
  "path": "TOOLS.md",
  "line": 12,
  "ocPath": "oc://TOOLS.md/tools/deploy",
  "target": "oc://TOOLS.md/tools/deploy",
  "requirement": "oc://policy.jsonc/tools/requireMetadata"
}
```

```json
{
  "checkId": "policy/mcp-unapproved-server",
  "severity": "error",
  "message": "MCP サーバー 'remote' はポリシー許可リストに含まれていません。",
  "source": "policy",
  "path": "openclaw config",
  "ocPath": "oc://openclaw.config/mcp/servers/remote",
  "target": "oc://openclaw.config/mcp/servers/remote",
  "requirement": "oc://policy.jsonc/mcp/servers/allow"
}
```

```json
{
  "checkId": "policy/models-unapproved-provider",
  "severity": "error",
  "message": "モデル参照 'anthropic/claude-sonnet-4.7' は未承認のプロバイダー 'anthropic' を使用しています。",
  "source": "policy",
  "path": "openclaw config",
  "ocPath": "oc://openclaw.config/agents/defaults/model/fallbacks/#0",
  "target": "oc://openclaw.config/agents/defaults/model/fallbacks/#0",
  "requirement": "oc://policy.jsonc/models/providers/allow"
}
```

```json
{
  "checkId": "policy/network-private-access-enabled",
  "severity": "error",
  "message": "ネットワーク設定 'browser-private-network' では、プライベートネットワークへのアクセスが許可されています。",
  "source": "policy",
  "path": "openclaw config",
  "ocPath": "oc://openclaw.config/browser/ssrfPolicy/dangerouslyAllowPrivateNetwork",
  "target": "oc://openclaw.config/browser/ssrfPolicy/dangerouslyAllowPrivateNetwork",
  "requirement": "oc://policy.jsonc/network/privateNetwork/allow"
}
```

```json
{
  "checkId": "policy/gateway-non-loopback-bind",
  "severity": "error",
  "message": "Gateway のバインド設定 'gateway-bind' は非ループバックへの公開を許可しています。",
  "source": "policy",
  "path": "openclaw config",
  "ocPath": "oc://openclaw.config/gateway/bind",
  "target": "oc://openclaw.config/gateway/bind",
  "requirement": "oc://policy.jsonc/gateway/exposure/allowNonLoopbackBind"
}
```

```json
{
  "checkId": "policy/gateway-node-command-denied",
  "severity": "error",
  "message": "Gateway Node コマンド 'system.run' はポリシーで拒否されていますが、OpenClaw の設定では拒否されていません。",
  "source": "policy",
  "path": "openclaw config",
  "ocPath": "oc://openclaw.config/gateway/nodes/commands/deny",
  "target": "oc://openclaw.config/gateway/nodes/commands/deny",
  "requirement": "oc://policy.jsonc/gateway/nodes/denyCommands",
  "fixHint": "gateway.nodes.commands.deny に 'system.run' を追加するか、レビュー後にポリシーを更新してください。"
}
```

```json
{
  "checkId": "policy/agents-workspace-access-denied",
  "severity": "error",
  "message": "agents.defaults のサンドボックス workspaceAccess 'rw' はポリシーで許可されていません。",
  "source": "policy",
  "path": "openclaw config",
  "ocPath": "oc://openclaw.config/agents/defaults/sandbox/workspaceAccess",
  "target": "oc://openclaw.config/agents/defaults/sandbox/workspaceAccess",
  "requirement": "oc://policy.jsonc/agents/workspace/allowedAccess"
}
```

## 修復

`doctor --lint` と `policy check` は読み取り専用です。

`doctor --fix` は、`workspaceRepairs` が明示的に有効になっている場合にのみ、ポリシーで管理されるワークスペース設定を編集します。それ以外の場合、チェックでは修復対象を報告し、設定は変更しません。

このバージョンでは、修復によって `channels.denyRules` で拒否されたチャンネルを無効化し、以下に示す自動的な制限強化の修復を適用できます。有効なルールによってワークスペース設定が変更される可能性があるため、ポリシーファイルをレビューした後にのみ `workspaceRepairs` を有効にしてください。

- グローバルポリシーで昇格ツールが禁止されている場合、`tools.elevated.enabled=false` を設定する
- ポリシーで対象ツールの拒否が必須とされている場合、不足している必須拒否ツール ID を `tools.deny` または
  `agents.entries.*.tools.deny` に追加する
- 安全でない `gateway.controlUi.*` の切り替え設定を `false` に設定する
- ポリシーでリモート Gateway モードが拒否されている場合、`gateway.mode=local` を設定する
- ポリシーで Gateway HTTP API エンドポイントが拒否されている場合、報告された `gateway.http.endpoints.*.enabled` パスを `false` に設定する
- ポリシーでオープングループからの受信が拒否されている場合、報告されたチャンネル受信 `groupPolicy` パスを `allowlist` に設定する
- ポリシーでグループメンションが必須とされている場合、報告されたチャンネル受信 `requireMention` パスを `true` に設定する
- ポリシーで機密ログの秘匿化が必須とされている場合、`logging.redactSensitive=tools` を設定する
- ポリシーでテレメトリ内容のキャプチャが拒否されている場合、`diagnostics.otel.captureContent=false`、または
  オブジェクト形式のテレメトリキャプチャ設定では
  `diagnostics.otel.captureContent.enabled=false` を設定する

スコープ指定された昇格ツールの修復は検出のみです。検出結果で共有のログ設定またはテレメトリ設定が報告されている場合も、スコープ指定されたデータ処理の修復はスキップされます。共有設定を変更すると、スコープ指定されたポリシー対象以外にも影響が及ぶためです。

検出結果で継承されたルート `tools.deny` が報告されている場合、スコープ指定された必須拒否の修復はスキップされます。必須ツールをルート設定に追加すると、スコープ指定されたポリシー対象以外にも影響が及ぶためです。エージェントローカルの必須拒否の修復では、報告された `agents.entries.*.tools.deny` パスを更新できます。

検出結果で継承された `channels.defaults.*` が報告されている場合、スコープ指定されたチャンネル受信の修復はスキップされます。共有チャンネルのデフォルトを変更すると、スコープ指定されたポリシー対象以外にも影響が及ぶためです。自動修復では正しいエンドポイント URL の許可リスト値を選択できないため、Gateway HTTP URL 取得の許可リストに関する検出結果は手動対応のままです。

Gateway のバインドと Node コマンドに関する検出結果には、引き続きレビューが必要です。`policy/gateway-non-loopback-bind` または `policy/gateway-node-command-denied` を設定パスにマッピングできる場合、`doctor --fix` は提案された `gateway.bind` または `gateway.nodes.commands.deny` の変更を、スキップされたプレビューガイダンスとして報告します。変更は適用されず、オペレーターがレビューして設定またはポリシーを更新するまで、検出結果は修復済みとしてカウントされません。

```jsonc
{
  "plugins": {
    "entries": {
      "policy": {
        "config": {
          "workspaceRepairs": true,
        },
      },
    },
  },
}
```

## 終了コード

| コマンド          | `0`                                                    | `1`                                                                 | `2`                          |
| ---------------- | ------------------------------------------------------ | ------------------------------------------------------------------- | ---------------------------- |
| `policy check`   | しきい値に該当する検出結果はありません。                          | 1 件以上の検出結果がしきい値に達しました。                             | 引数または実行時の失敗。 |
| `policy compare` | ポリシーファイルは少なくともベースラインと同等に厳格です。 | ポリシーファイルが無効、存在しない、またはベースラインルールより緩い状態です。 | 引数または実行時の失敗。 |
| `policy watch`   | 検出結果がなく、承認済みハッシュは最新です。              | 検出結果が存在するか、承認済みの証明が古くなっています。                    | 引数または実行時の失敗。 |

## 関連項目

- [Doctor の lint モード](/ja-JP/cli/doctor#lint-mode)
- [パス CLI](/ja-JP/cli/path)
