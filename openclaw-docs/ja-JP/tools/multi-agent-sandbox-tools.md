---
read_when: You want per-agent sandboxing or per-agent tool allow/deny policies in a multi-agent gateway.
sidebarTitle: Multi-agent sandbox and tools
status: active
summary: エージェントごとのサンドボックスとツール制限、優先順位、および例
title: マルチエージェントのサンドボックスとツール
x-i18n:
    generated_at: "2026-07-26T09:23:05Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 0e07d07c30b844be1e1d93db62fcdaab72c47a5248367559642a959bf09ad193
    source_path: tools/multi-agent-sandbox-tools.md
    workflow: 16
---

マルチエージェント構成の各エージェントは、グローバルなサンドボックスおよびツールポリシーを上書きできます。このページでは、エージェントごとの設定、優先順位規則、例について説明します。

<CardGroup cols={3}>
  <Card title="サンドボックス化" href="/ja-JP/gateway/sandboxing">
    バックエンドとモード — サンドボックスの完全なリファレンス。
  </Card>
  <Card title="サンドボックス、ツールポリシー、昇格の違い" href="/ja-JP/gateway/sandbox-vs-tool-policy-vs-elevated">
    「なぜこれがブロックされるのか？」をデバッグします。
  </Card>
  <Card title="昇格モード" href="/ja-JP/tools/elevated">
    信頼できる送信者向けの昇格 exec。
  </Card>
</CardGroup>

<Warning>
認証はエージェント単位でスコープされます。各エージェントは `~/.openclaw/agents/<agentId>/agent/openclaw-agent.sqlite` に独自の `agentDir` 認証ストアを持ちます。エージェント間で `agentDir` を再利用しないでください。ローカルプロファイルがない場合、エージェントはデフォルト／メインエージェントの認証プロファイルを参照できますが、OAuth リフレッシュトークンはセカンダリエージェントのストアに複製されません。認証情報を手動でコピーする場合は、移植可能な静的 `api_key` または `token` プロファイルのみをコピーしてください。
</Warning>

---

## 設定例

<AccordionGroup>
  <Accordion title="例 1：個人用エージェントと制限付き家族用エージェント">
    ```json
    {
      "agents": {
        "list": [
          {
            "id": "main",
            "default": true,
            "name": "Personal Assistant",
            "workspace": "~/.openclaw/workspace",
            "sandbox": { "mode": "off" }
          },
          {
            "id": "family",
            "name": "Family Bot",
            "workspace": "~/.openclaw/workspace-family",
            "sandbox": {
              "mode": "all",
              "scope": "agent"
            },
            "tools": {
              "allow": ["read", "message"],
              "deny": ["exec", "write", "edit", "apply_patch", "process", "browser"],
              "message": {
                "crossContext": {
                  "allowWithinProvider": false,
                  "allowAcrossProviders": false
                }
              }
            }
          }
        ]
      },
      "bindings": [
        {
          "agentId": "family",
          "match": {
            "provider": "whatsapp",
            "accountId": "*",
            "peer": {
              "kind": "group",
              "id": "120363424282127706@g.us"
            }
          }
        }
      ]
    }
    ```

    **結果：**

    - `main` エージェント：ホスト上で実行され、すべてのツールにアクセスできます。
    - `family` エージェント：Docker 上で実行され（エージェントごとに 1 コンテナ）、`read` と現在の会話へのメッセージ送信のみを使用できます。

  </Accordion>
  <Accordion title="例 2：共有サンドボックスを使用する仕事用エージェント">
    ```json
    {
      "agents": {
        "list": [
          {
            "id": "personal",
            "workspace": "~/.openclaw/workspace-personal",
            "sandbox": { "mode": "off" }
          },
          {
            "id": "work",
            "workspace": "~/.openclaw/workspace-work",
            "sandbox": {
              "mode": "all",
              "scope": "shared",
              "workspaceRoot": "/tmp/work-sandboxes"
            },
            "tools": {
              "allow": ["read", "write", "apply_patch", "exec"],
              "deny": ["browser", "gateway", "discord"]
            }
          }
        ]
      }
    }
    ```
  </Accordion>
  <Accordion title="例 2b：グローバルなコーディングプロファイルとメッセージング専用エージェント">
    ```json
    {
      "tools": { "profile": "coding" },
      "agents": {
        "list": [
          {
            "id": "support",
            "tools": { "profile": "messaging", "allow": ["slack"] }
          }
        ]
      }
    }
    ```

    **結果：**

    - デフォルトのエージェントはコーディングツールを使用できます。
    - `support` エージェントはメッセージング専用です（+ Slack ツール）。

  </Accordion>
  <Accordion title="例 3：エージェントごとに異なるサンドボックスモード">
    ```json
    {
      "agents": {
        "defaults": {
          "sandbox": {
            "mode": "non-main",
            "scope": "session"
          }
        },
        "list": [
          {
            "id": "main",
            "workspace": "~/.openclaw/workspace",
            "sandbox": {
              "mode": "off"
            }
          },
          {
            "id": "public",
            "workspace": "~/.openclaw/workspace-public",
            "sandbox": {
              "mode": "all",
              "scope": "agent"
            },
            "tools": {
              "allow": ["read"],
              "deny": ["exec", "write", "edit", "apply_patch"]
            }
          }
        ]
      }
    }
    ```
  </Accordion>
</AccordionGroup>

---

## 設定の優先順位

グローバル（`agents.defaults.*`）設定とエージェント固有（`agents.entries.*.*`）設定の両方が存在する場合：

### サンドボックス設定

エージェント固有の設定がグローバル設定を上書きします。

```text
agents.entries.*.sandbox.mode > agents.defaults.sandbox.mode
agents.entries.*.sandbox.scope > agents.defaults.sandbox.scope
agents.entries.*.sandbox.workspaceRoot > agents.defaults.sandbox.workspaceRoot
agents.entries.*.sandbox.workspaceAccess > agents.defaults.sandbox.workspaceAccess
agents.entries.*.sandbox.docker.* > agents.defaults.sandbox.docker.*
agents.entries.*.sandbox.browser.* > agents.defaults.sandbox.browser.*
agents.entries.*.sandbox.prune.* > agents.defaults.sandbox.prune.*
```

<Note>
`agents.entries.*.sandbox.{docker,browser,prune}.*` は、そのエージェントの `agents.defaults.sandbox.{docker,browser,prune}.*` を上書きします（サンドボックスのスコープが `"shared"` に解決される場合は無視されます）。
</Note>

### ツール制限

フィルタリングの順序は次のとおりです。

<Steps>
  <Step title="ツールプロファイル">
    `tools.profile` または `agents.entries.*.tools.profile`。
  </Step>
  <Step title="プロバイダーのツールプロファイル">
    `tools.byProvider[provider].profile` または `agents.entries.*.tools.byProvider[provider].profile`。
  </Step>
  <Step title="グローバルツールポリシー">
    `tools.allow` / `tools.deny`。
  </Step>
  <Step title="プロバイダーのツールポリシー">
    `tools.byProvider[provider].allow/deny`。
  </Step>
  <Step title="エージェント固有のツールポリシー">
    `agents.entries.*.tools.allow/deny`。
  </Step>
  <Step title="エージェントのプロバイダーポリシー">
    `agents.entries.*.tools.byProvider[provider].allow/deny`。
  </Step>
  <Step title="サンドボックスのツールポリシー">
    `tools.sandbox.tools` または `agents.entries.*.tools.sandbox.tools`。
  </Step>
  <Step title="サブエージェントのツールポリシー">
    該当する場合は `tools.subagents.tools`。
  </Step>
</Steps>

<AccordionGroup>
  <Accordion title="優先順位規則">
    - 各レベルでツールをさらに制限できますが、それ以前のレベルで拒否されたツールを再び許可することはできません。
    - `agents.entries.*.tools.sandbox.tools` が設定されている場合、そのエージェントでは `tools.sandbox.tools` が置き換えられます。
    - `agents.entries.*.tools.profile` が設定されている場合、そのエージェントでは `tools.profile` が上書きされます。
    - プロバイダーのツールキーには、`provider`（例：`google-antigravity`）または `provider/model`（例：`openai/gpt-5.4`）のいずれかを使用できます。

  </Accordion>
  <Accordion title="空の許可リストの動作">
    このチェーン内の明示的な許可リストのいずれかにより、実行可能なツールがなくなった場合、OpenClaw はモデルにプロンプトを送信する前に停止します。これは意図された動作です。`agents.entries.*.tools.allow: ["query_db"]` のような存在しないツールを設定したエージェントは、`query_db` を登録する Plugin が有効になるまで明示的に失敗する必要があり、テキスト専用エージェントとして処理を継続してはなりません。
  </Accordion>
</AccordionGroup>

ツールポリシーでは、複数のツールに展開される `group:*` の短縮表記をサポートしています。完全な一覧については、[ツールグループ](/ja-JP/gateway/sandbox-vs-tool-policy-vs-elevated#tool-groups-shorthands)を参照してください。

エージェントごとの昇格オーバーライド（`agents.entries.*.tools.elevated`）により、特定のエージェントに対する昇格 exec をさらに制限できます。詳細については、[昇格モード](/ja-JP/tools/elevated)を参照してください。

---

## 単一エージェントからの移行

<Tabs>
  <Tab title="移行前（単一エージェント）">
    ```json
    {
      "agents": {
        "defaults": {
          "workspace": "~/.openclaw/workspace",
          "sandbox": {
            "mode": "non-main"
          }
        }
      },
      "tools": {
        "sandbox": {
          "tools": {
            "allow": ["read", "write", "apply_patch", "exec"],
            "deny": []
          }
        }
      }
    }
    ```
  </Tab>
  <Tab title="移行後（マルチエージェント）">
    ```json
    {
      "agents": {
        "list": [
          {
            "id": "main",
            "default": true,
            "workspace": "~/.openclaw/workspace",
            "sandbox": { "mode": "off" }
          }
        ]
      }
    }
    ```
  </Tab>
</Tabs>

<Note>
従来の `agents.defaults.*`/`agents.entries.*.*` 設定キー（`sandbox.perSession`、`agentRuntime`、`embeddedPi` など）は `openclaw doctor` によって移行されます。今後は `agents.defaults` + `agents.entries` を使用してください。
</Note>

---

## ツール制限の例

<Tabs>
  <Tab title="読み取り専用エージェント">
    ```json
    {
      "tools": {
        "allow": ["read"],
        "deny": ["exec", "write", "edit", "apply_patch", "process"]
      }
    }
    ```
  </Tab>
  <Tab title="ファイルシステムツールを無効にしたシェル実行">
    ```json
    {
      "tools": {
        "allow": ["read", "exec", "process"],
        "deny": ["write", "edit", "apply_patch", "browser", "gateway"]
      }
    }
    ```

    <Warning>
    このポリシーは OpenClaw のファイルシステムツールを無効にしますが、`exec` は引き続きシェルであり、選択したホストまたはサンドボックスのファイルシステムで許可されている任意の場所にファイルを書き込めます。読み取り専用エージェントにするには、`exec` と `process` を拒否するか、シェルアクセスと `agents.defaults.sandbox.workspaceAccess: "ro"` または `"none"` などのサンドボックスファイルシステム制御を組み合わせてください。
    </Warning>

  </Tab>
  <Tab title="通信専用">
    ```json
    {
      "tools": {
        "sessions": { "visibility": "tree" },
        "allow": ["sessions_list", "sessions_send", "sessions_history", "session_status"],
        "deny": ["exec", "write", "edit", "apply_patch", "read", "browser"]
      }
    }
    ```

    このプロファイルの `sessions_history` も、生のトランスクリプトダンプではなく、範囲が制限されサニタイズされた想起ビューを返します。アシスタントの想起では、編集／切り詰めの前に、思考タグ、`<relevant-memories>` スキャフォールディング、プレーンテキストのツール呼び出し XML ペイロード（`<tool_call>...</tool_call>`、`<function_call>...</function_call>`、`<tool_calls>...</tool_calls>`、`<function_calls>...</function_calls>`、および途中で切り詰められたツール呼び出しブロックを含む）、ダウングレードされたツール呼び出しスキャフォールディング、漏出した ASCII／全角のモデル制御トークン、不正な MiniMax ツール呼び出し XML が除去されます。

  </Tab>
</Tabs>

---

## よくある落とし穴：「non-main」

<Warning>
`agents.defaults.sandbox.mode: "non-main"` は、エージェント ID ではなく、セッションキーをメインセッションキー（常に `"main"`。`session.mainKey` はユーザーが設定できず、他の値を指定すると OpenClaw が警告して無視します）と照合します。グループ／チャンネルセッションには常に独自のキーが割り当てられるため、non-main として扱われ、サンドボックス化されます。エージェントをサンドボックス化しない場合は、`agents.entries.*.sandbox.mode: "off"` を設定してください。
</Warning>

---

## テスト

マルチエージェントのサンドボックスとツールを設定した後：

<Steps>
  <Step title="エージェントの解決を確認">
    ```bash
    openclaw agents list --bindings
    ```
  </Step>
  <Step title="サンドボックスコンテナを確認">
    ```bash
    docker ps --filter "name=openclaw-sbx-"
    ```
  </Step>
  <Step title="ツール制限をテスト">
    - 制限されたツールを必要とするメッセージを送信します。
    - エージェントが拒否されたツールを使用できないことを確認します。

  </Step>
  <Step title="ログを監視">
    ```bash
    openclaw logs --follow | grep -E "routing|sandbox|tools"
    ```
  </Step>
</Steps>

---

## トラブルシューティング

<AccordionGroup>
  <Accordion title="`mode: 'all'` にもかかわらずエージェントがサンドボックス化されない">
    - それを上書きするグローバルな `agents.defaults.sandbox.mode` が存在するか確認します。
    - エージェント固有の設定が優先されるため、`agents.entries.*.sandbox.mode: "all"` を設定します。

  </Accordion>
  <Accordion title="拒否リストがあっても使用可能なツール">
    - [フィルタリングの全順序](#tool-restrictions)を確認してください：プロファイル → プロバイダープロファイル → グローバルポリシー → プロバイダーポリシー → エージェントポリシー → エージェントプロバイダーポリシー → サンドボックス → サブエージェント。
    - 各レベルでは制限をさらに追加できるだけで、権限を再付与することはできません。
    - 段階的なデバッグ方法については、[サンドボックスとツールポリシーと昇格モードの比較](/ja-JP/gateway/sandbox-vs-tool-policy-vs-elevated)を参照してください。

  </Accordion>
  <Accordion title="エージェントごとにコンテナが分離されていない">
    - デフォルトの `scope` は `"agent"`（エージェント ID ごとに 1 つのコンテナ）です。
    - セッションごとに 1 つのコンテナを使用するには `scope: "session"` を設定し、エージェント間で 1 つのコンテナを再利用するには `scope: "shared"` を設定します。

  </Accordion>
</AccordionGroup>

---

## 関連項目

- [昇格モード](/ja-JP/tools/elevated)
- [マルチエージェントルーティング](/ja-JP/concepts/multi-agent)
- [サンドボックス設定](/ja-JP/gateway/config-agents#agentsdefaultssandbox)
- [サンドボックスとツールポリシーと昇格モードの比較](/ja-JP/gateway/sandbox-vs-tool-policy-vs-elevated) — 「なぜブロックされるのか？」のデバッグ
- [サンドボックス化](/ja-JP/gateway/sandboxing) — サンドボックスの完全なリファレンス（モード、スコープ、バックエンド、イメージ）
- [セッション管理](/ja-JP/concepts/session)
