---
read_when: You hit 'sandbox jail' or see a tool/elevated refusal and want the exact config key to change.
status: active
summary: ツールがブロックされる理由：サンドボックスランタイム、ツールの許可／拒否ポリシー、および権限昇格された実行のゲート
title: サンドボックス、ツールポリシー、昇格の違い
x-i18n:
    generated_at: "2026-07-26T09:23:13Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: c4da521215fe55bf2774008a53d896d5c00b8babcbca2005dc4593ebfebc5343
    source_path: gateway/sandbox-vs-tool-policy-vs-elevated.md
    workflow: 16
---

OpenClaw には、関連しているものの異なる 3 つの制御があります。

1. **サンドボックス**（`agents.defaults.sandbox.*` / `agents.entries.*.sandbox.*`）は、**ツールをどこで実行するか**（サンドボックスバックエンドまたはホスト）を決定します。
2. **ツールポリシー**（`tools.*`、`tools.sandbox.tools.*`、`agents.entries.*.tools.*`）は、**どのツールを利用可能／許可するか**を決定します。
3. **昇格**（`tools.elevated.*`、`agents.entries.*.tools.elevated.*`）は、サンドボックス化されている場合にサンドボックス外で実行するための **exec 専用のエスケープハッチ**です（デフォルトでは `gateway`、または exec ターゲットが `node` に設定されている場合は `node`）。

## クイックデバッグ

インスペクターを使用して、OpenClaw が_実際に_何をしているかを確認します。

```bash
openclaw sandbox explain
openclaw sandbox explain --session agent:main:main
openclaw sandbox explain --agent work
openclaw sandbox explain --json
```

次の情報が出力されます。

- 有効なサンドボックスのモード／スコープ／ワークスペースアクセス
- セッションが現在サンドボックス化されているかどうか（メインか非メインか）
- 有効なサンドボックスツールの許可／拒否（およびエージェント／グローバル／デフォルトのどれに由来するか）
- 昇格のゲートと修正用キーのパス

## サンドボックス：ツールを実行する場所

サンドボックス化は `agents.defaults.sandbox.mode` によって制御されます。

- `"off"`：すべてがホスト上で実行されます。
- `"non-main"`：非メインセッションのみがサンドボックス化されます（グループ／チャンネルでよくある「予想外」の挙動）。
- `"all"`：すべてがサンドボックス化されます。

`agents.defaults.sandbox.workspaceAccess` は、サンドボックスから見える範囲を制御します：`"none"`、`"ro"`、または `"rw"`。

完全なマトリクス（スコープ、ワークスペースのマウント、イメージ）については、[サンドボックス化](/ja-JP/gateway/sandboxing)を参照してください。

### バインドマウント（セキュリティのクイックチェック）

- `docker.binds` はサンドボックスのファイルシステムを_貫通_します。マウントしたものはすべて、設定したモード（`:ro` または `:rw`）でコンテナ内から参照できます。
- モードを省略した場合、デフォルトは読み書き可能です。ソース／シークレットには `:ro` を推奨します。
- `scope: "shared"` はエージェントごとのバインドを無視します（グローバルバインドのみが適用されます）。
- OpenClaw はバインド元を 2 回検証します。最初に正規化されたソースパスを検証し、次に最も深い既存の祖先を経由して解決した後で再度検証します。親ディレクトリのシンボリックリンクを利用した脱出によって、ブロック対象パスまたは許可ルートのチェックを回避することはできません。
- 存在しない末端パスも安全にチェックされます。`/workspace/alias-out/new-file` がシンボリックリンクされた親を経由してブロック対象パスまたは設定済みの許可ルート外に解決される場合、バインドは拒否されます。
- `/var/run/docker.sock` をバインドすると、実質的にサンドボックスへホストの制御権を渡すことになります。意図している場合にのみ実行してください。
- ワークスペースアクセス（`workspaceAccess`）はバインドモードとは独立しています。

複数のホストフォルダー、アクセスモード、および外部ソースに関する安全性のオプトインを含むエージェントごとの設定については、[1 つのエージェントで複数のフォルダーを使用する](/ja-JP/gateway/sandboxing#multiple-folders-for-one-agent)を参照してください。

## ツールポリシー：存在し、呼び出し可能なツール

重要なレイヤーは次のとおりです。

- **ツールプロファイル**：`tools.profile` および `agents.entries.*.tools.profile`（基本許可リスト）
- **プロバイダーツールプロファイル**：`tools.byProvider[provider].profile` および `agents.entries.*.tools.byProvider[provider].profile`
- **グローバル／エージェントごとのツールポリシー**：`tools.allow`／`tools.deny` および `agents.entries.*.tools.allow`／`agents.entries.*.tools.deny`
- **プロバイダーツールポリシー**：`tools.byProvider[provider].allow/deny` および `agents.entries.*.tools.byProvider[provider].allow/deny`
- **サンドボックスツールポリシー**（サンドボックス化されている場合にのみ適用）：`tools.sandbox.tools.allow`／`tools.sandbox.tools.deny` および `agents.entries.*.tools.sandbox.tools.*`

目安となるルール：

- `deny` が常に優先されます。
- `allow` が空でない場合、それ以外はすべてブロック対象として扱われます。
- ツールポリシーは最終的な制限です。`/exec` で拒否された `exec` ツールを上書きすることはできません。
- ツールポリシーは名前によってツールの可用性をフィルタリングします。`exec` 内部の副作用は検査しません。`exec` が許可されている場合、`write`、`edit`、または `apply_patch` を拒否しても、シェルコマンドは読み取り専用にはなりません。
- `/exec` は、承認済み送信者のセッションデフォルトを変更するだけであり、ツールへのアクセス権は付与しません。
- プロバイダーツールキーでは、`provider`（例：`google-antigravity`）または `provider/model`（例：`openai/gpt-5.4`）のいずれかを使用できます。
- ツールポリシーのステップでツールが除外された場合、またはサンドボックスツールポリシーによって呼び出しがブロックされた場合、Gateway のログには `agents/tool-policy` 監査エントリが含まれます。ルールラベル、設定キー、および影響を受けるツール名を確認するには、`openclaw logs` を使用してください。

### ツールグループ（短縮表記）

ツールポリシー（グローバル、エージェント、サンドボックス）は、複数のツールに展開される `group:*` エントリをサポートします。

```json5
{
  tools: {
    sandbox: {
      tools: {
        allow: ["group:runtime", "group:fs", "group:sessions", "group:memory"],
      },
    },
  },
}
```

利用可能なグループ：

| グループ              | ツール                                                                                                                                                                                                                                                  |
| ------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `group:runtime`    | `exec`、`process`、`code_execution`（`bash` は `exec` のエイリアスとして使用可能）                                                                                                                                                                        |
| `group:fs`         | `read`、`write`、`edit`、`apply_patch`                                                                                                                                                                                                                 |
| `group:sessions`   | `sessions`、`sessions_list`、`sessions_history`、`sessions_search`、`conversations_list`、`conversations_send`、`conversations_turn`、`sessions_send`、`sessions_spawn`、`sessions_yield`、`subagents`、`session_status`、`spawn_task`、`dismiss_task` |
| `group:memory`     | `memory_search`、`memory_get`                                                                                                                                                                                                                          |
| `group:web`        | `web_search`、`x_search`、`web_fetch`                                                                                                                                                                                                                  |
| `group:ui`         | `browser`、`screen`、`terminal`、`canvas`、`show_widget`                                                                                                                                                                                               |
| `group:automation` | `heartbeat_respond`、`cron`、`gateway`                                                                                                                                                                                                                 |
| `group:messaging`  | `message`                                                                                                                                                                                                                                              |
| `group:nodes`      | `nodes`、`computer`                                                                                                                                                                                                                                    |
| `group:agents`     | `agents_list`、`get_goal`、`create_goal`、`update_goal`、`update_plan`、`ask_user`、`skill_workshop`                                                                                                                                                   |
| `group:media`      | `image`、`image_generate`、`music_generate`、`video_generate`、`tts`                                                                                                                                                                                   |
| `group:openclaw`   | OpenClaw のほとんどの組み込みツール（`read`／`write`／`edit`／`apply_patch`／`exec`／`process` のファイルシステムおよびランタイムプリミティブ、`canvas`、プロバイダー Plugin を除く）                                                                                             |
| `group:plugins`    | `bundle-mcp` を通じて公開される設定済み MCP サーバーを含む、読み込まれたすべての Plugin 所有ツール                                                                                                                                                           |

読み取り専用エージェントでは、サンドボックスのファイルシステムポリシーまたは別のホスト境界によって読み取り専用制約が適用されていない限り、ファイルシステムを変更するツールに加えて `group:runtime` も拒否してください。

サンドボックス化された MCP サーバーでは、サンドボックスツールポリシーが 2 つ目の許可ゲートになります。`mcp.servers` が設定されているにもかかわらず、サンドボックス化されたターンで組み込みツールしか表示されない場合は、`bundle-mcp`、`group:plugins`、または `outlook__send_mail` や `outlook__*` のようなサーバープレフィックス付き MCP ツール名／グロブを `tools.sandbox.tools.alsoAllow` に追加し、Gateway を再起動／再読み込みしてツールリストを再取得してください。サーバーグロブには、プロバイダーで安全に使用できる MCP サーバープレフィックスが使用されます。`[A-Za-z0-9_-]` 以外の文字は `-` になり、文字で始まらない名前には `mcp-` プレフィックスが付き、長いプレフィックスまたは重複するプレフィックスは切り詰められるかサフィックスが付く場合があります。

`openclaw doctor` は現在、`mcp.servers` 内の OpenClaw 管理サーバーについてこの形式をチェックします。バンドルされた Plugin マニフェストまたは Claude `.mcp.json` から読み込まれた MCP サーバーも同じサンドボックスゲートを使用しますが、この診断ではまだこれらのソースを列挙しません。サンドボックス化されたターンでそれらのツールが表示されなくなった場合は、同じ許可リストエントリを使用してください。

## 昇格：exec 専用の「ホスト上で実行」

昇格によって追加のツールが付与されることは**ありません**。影響するのは `exec` だけです。

- サンドボックス化されている場合、`/elevated on`（または `elevated: true` を伴う `exec`）はサンドボックス外で実行されます（引き続き承認が必要な場合があります）。
- セッションで exec の承認を省略するには、`/elevated full` を使用します。
- すでに直接実行されている場合、昇格は実質的に何もしません（ただし、引き続きゲートの制約を受けます）。
- 昇格は Skills 単位ではなく、ツールの許可／拒否を上書きすることも**ありません**。
- 昇格によって `host=auto` から任意のクロスホスト上書きが許可されることはありません。通常の exec ターゲットルールに従い、設定済み／セッションのターゲットがすでに `node` である場合にのみ `node` を維持します。
- `/exec` は昇格とは別のものです。承認済み送信者について、セッションごとの exec デフォルトを調整するだけです。

ゲート：

- 有効化：`tools.elevated.enabled`（および必要に応じて `agents.entries.*.tools.elevated.enabled`）
- 送信者の許可リスト：`tools.elevated.allowFrom.<provider>`（および必要に応じて `agents.entries.*.tools.elevated.allowFrom.<provider>`）

[昇格モード](/ja-JP/tools/elevated)を参照してください。

## よくある「サンドボックスへの閉じ込め」の修正

### 「ツール X がサンドボックスツールポリシーによってブロックされました」

修正用キー（いずれか 1 つを選択）：

- サンドボックスを無効化: `agents.defaults.sandbox.mode=off`（またはエージェントごとの `agents.entries.*.sandbox.mode=off`）
- サンドボックス内でツールを許可:
  - `tools.sandbox.tools.deny` から削除する（またはエージェントごとの `agents.entries.*.tools.sandbox.tools.deny`）
  - または `tools.sandbox.tools.allow` に追加する（またはエージェントごとに許可）
- `openclaw logs` で `agents/tool-policy` エントリを確認してください。このエントリには、サンドボックスモードと、許可ルールまたは拒否ルールのどちらによってツールがブロックされたかが記録されます。

### 「main だと思っていたのに、なぜサンドボックス化されているのですか？」

`"non-main"` モードでは、グループ／チャンネルキーは main ではありません。main セッションキー（`sandbox explain` で表示）を使用するか、モードを `"off"` に切り替えてください。

## 関連項目

- [サンドボックス化](/ja-JP/gateway/sandboxing) -- サンドボックスの完全なリファレンス（モード、スコープ、バックエンド、イメージ）
- [マルチエージェントのサンドボックスとツール](/ja-JP/tools/multi-agent-sandbox-tools) -- エージェントごとの上書きと優先順位
- [昇格モード](/ja-JP/tools/elevated)
