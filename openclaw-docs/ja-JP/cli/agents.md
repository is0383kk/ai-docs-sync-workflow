---
read_when:
    - 複数の分離されたエージェント（ワークスペース + ルーティング + 認証）が必要な場合
summary: '`openclaw agents` の CLI リファレンス（一覧表示/追加/削除/バインディング/バインド/バインド解除/アイデンティティの設定）'
title: エージェント
x-i18n:
    generated_at: "2026-07-26T08:55:23Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 76a2e50462f6a52760dcb639405ed5f23857f2fa429469281e3acfa1eb61e974
    source_path: cli/agents.md
    workflow: 16
---

# `openclaw agents`

分離されたエージェント（ワークスペース + 認証 + ルーティング）を管理します。サブコマンドなしで `openclaw agents` を実行することは、`openclaw agents list` と同等です。

関連項目:

- [マルチエージェントルーティング](/ja-JP/concepts/multi-agent)
- [エージェントワークスペース](/ja-JP/concepts/agent-workspace)
- [Skills 設定](/ja-JP/tools/skills-config): スキルの可視性設定。

## 例

```bash
openclaw agents list
openclaw agents list --bindings
openclaw agents add work --workspace ~/.openclaw/workspace-work
openclaw agents add work --workspace ~/.openclaw/workspace-work --bind telegram:*
openclaw agents add ops --workspace ~/.openclaw/workspace-ops --bind telegram:ops --non-interactive
openclaw agents bindings
openclaw agents bind --agent work --bind telegram:ops
openclaw agents unbind --agent work --bind telegram:ops
openclaw agents set-identity --workspace ~/.openclaw/workspace --from-identity
openclaw agents set-identity --agent main --avatar avatars/openclaw.png
openclaw agents delete work
```

## コマンド一覧

### `agents list`

オプション: `--json`、`--bindings`（エージェントごとの件数や概要だけでなく、完全なルーティングルールを含めます）。

### `agents add [name]`

オプション: `--workspace <dir>`、`--model <id>`、`--agent-dir <dir>`、`--bind <channel[:accountId]>`（繰り返し指定可能）、`--non-interactive`、`--json`。

- 明示的な追加フラグをいずれか指定すると、コマンドは非対話パスに切り替わります。
- 非対話モードでは、エージェント名と `--workspace` の両方が必要です。
- `main` は予約済みのため、新しいエージェント ID として使用できません。
- 対話モードでは、認証情報が `copyToAgents: false` でコピーを拒否していない限り、移植可能な静的認証情報（`api_key` および静的な `token` プロファイル）のみをコピーして認証を初期化します。OAuth リフレッシュトークンのプロファイルは、プロバイダーが `copyToAgents: true` で明示的に許可していない限りコピーされません。コピーされない場合、OAuth は実際の `main` エージェントストアからの読み取り継承を通じてのみ引き続き利用できます。設定されたデフォルトエージェントが `main` でない場合は、新しいエージェントで OAuth プロファイルごとに個別にサインインしてください。

### `agents bindings`

オプション: `--agent <id>`、`--json`。

### `agents bind`

オプション: `--agent <id>`（現在のデフォルトエージェントが既定値）、`--bind <channel[:accountId]>`（繰り返し指定可能）、`--json`。

### `agents unbind`

オプション: `--agent <id>`（現在のデフォルトエージェントが既定値）、`--bind <channel[:accountId]>`（繰り返し指定可能）、`--all`、`--json`。`--all` または 1 つ以上の `--bind` 値のいずれかを指定できますが、両方は指定できません。

### `agents set-identity`

オプション: `--agent <id>`、`--workspace <dir>`、`--identity-file <path>`、`--from-identity`、`--name <name>`、`--theme <theme>`、`--emoji <emoji>`、`--avatar <value>`、`--json`。後述の[アイデンティティの設定](#set-identity)を参照してください。

### `agents delete <id>`

オプション: `--force`、`--json`。

- `main` は削除できません。
- `--force` を指定しない場合、対話的な確認が必要です（非 TTY セッションでは失敗します。`--force` を指定して再実行してください）。
- ワークスペース、エージェント状態、およびセッショントランスクリプトのディレクトリは完全に削除されず、ゴミ箱に移動されます。ゴミ箱を利用できない場合でも、エージェント設定の削除は成功し、手動でクリーンアップする必要があるパスが報告されます。
- Gateway に到達可能な場合、削除は Gateway 経由で処理されるため、設定とセッションストアのクリーンアップではランタイムトラフィックと同じ書き込み処理が使用されます。Gateway に到達できない場合、CLI はオフラインのローカルパスにフォールバックします。
- 別のエージェントのワークスペースが同じパス、このワークスペース内のパス、またはこのワークスペースを含むパスである場合、ワークスペースは保持され、`--json` は `workspaceRetained`、`workspaceRetainedReason`、`workspaceSharedWith` を報告します。

## ルーティングバインディング

ルーティングバインディングを使用して、受信チャネルトラフィックを特定のエージェントに固定します。

エージェントごとに表示されるスキルも変更する場合は、`openclaw.json` で `agents.defaults.skills` と `agents.entries.*.skills` を設定します。[Skills 設定](/ja-JP/tools/skills-config)および[設定リファレンス](/ja-JP/gateway/config-agents#agentsdefaultsskills)を参照してください。

バインディングを一覧表示する:

```bash
openclaw agents bindings
openclaw agents bindings --agent work
openclaw agents bindings --json
```

バインディングを追加する:

```bash
openclaw agents bind --agent work --bind telegram:ops --bind discord:guild-a
```

エージェントの作成時にバインディングを追加することもできます:

```bash
openclaw agents add work --workspace ~/.openclaw/workspace-work --bind telegram:* --bind discord:*
```

`accountId`（`--bind <channel>`）を省略すると、OpenClaw は Plugin のセットアップフック、強制されたアカウントバインディング、またはチャネルに設定されたアカウント数からこれを解決します。

`bind` または `unbind` で `--agent` を省略すると、OpenClaw は現在のデフォルトエージェントを対象とします。

### `--bind` の形式

| 形式                       | 意味                                                                                            |
| ---------------------------- | -------------------------------------------------------------------------------------------------- |
| `--bind <channel>:*`         | チャネル上のすべてのアカウントに一致します。                                                                 |
| `--bind <channel>:<account>` | 1 つのアカウントに一致します。                                                                                 |
| `--bind <channel>`           | CLI が Plugin 固有のアカウントスコープを安全に解決できる場合を除き、デフォルトアカウントのみに一致します。 |

### バインディングスコープの動作

- `accountId` のない保存済みバインディングは、チャネルのデフォルトアカウントのみに一致します。
- `accountId: "*"` はチャネル全体のフォールバック（すべてのアカウント）であり、明示的なアカウントバインディングよりも優先度が低くなります。
- 同じエージェントに `accountId` のない一致するチャネルバインディングがすでに存在し、その後、明示的または解決済みの `accountId` を指定してバインドすると、OpenClaw は重複を追加せず、既存のバインディングをその場でアップグレードします。

例:

```bash
# チャネル上のすべてのアカウントに一致
openclaw agents bind --agent work --bind telegram:*

# 特定のアカウントに一致
openclaw agents bind --agent work --bind telegram:ops

# 最初のチャネルのみのバインディング
openclaw agents bind --agent work --bind telegram

# 後からアカウントスコープのバインディングにアップグレード
openclaw agents bind --agent work --bind telegram:alerts
```

アップグレード後、そのバインディングのルーティングは `telegram:alerts` に限定されます。デフォルトアカウントへのルーティングも必要な場合は、明示的に追加してください（例: `--bind telegram:default`）。

バインディングを削除する:

```bash
openclaw agents unbind --agent work --bind telegram:ops
openclaw agents unbind --agent work --all
```

## アイデンティティファイル

各エージェントワークスペースには、ワークスペースルートに `IDENTITY.md` を含めることができます:

- パスの例: `~/.openclaw/workspace/IDENTITY.md`
- `set-identity --from-identity` はワークスペースルート（または明示的な `--identity-file`）から読み取ります。

アバターのパスはワークスペースルートを基準に解決され、シンボリックリンクを介した場合でも、その外部には出られません。

## アイデンティティの設定

`set-identity` は `agents.entries.*.identity` に次のフィールドを書き込みます: `name`、`theme`、`emoji`、`avatar`（ワークスペース相対パス、http(s) URL、またはデータ URI）。

- `--agent` または `--workspace` で対象エージェントを選択します。`--workspace` が複数のエージェントに一致する場合、コマンドは失敗し、`--agent` を指定するよう求めます。
- ローカルのワークスペース相対アバター画像ファイルは 2 MB に制限されます。HTTP(S) URL と `data:` URI には、ローカルファイルサイズの上限チェックは適用されません。
- 明示的なアイデンティティフィールドが指定されていない場合、コマンドは `IDENTITY.md` からアイデンティティデータを読み取ります。

`IDENTITY.md` から読み込む:

```bash
openclaw agents set-identity --workspace ~/.openclaw/workspace --from-identity
```

フィールドを明示的に上書きする:

```bash
openclaw agents set-identity --agent main --name "OpenClaw" --emoji "🦞" --avatar avatars/openclaw.png
```

設定例:

```json5
{
  agents: {
    list: [
      {
        id: "main",
        identity: {
          name: "OpenClaw",
          theme: "space lobster",
          emoji: "🦞",
          avatar: "avatars/openclaw.png",
        },
      },
    ],
  },
}
```

## 関連項目

- [CLI リファレンス](/ja-JP/cli)
- [マルチエージェントルーティング](/ja-JP/concepts/multi-agent)
- [エージェントワークスペース](/ja-JP/concepts/agent-workspace)
