---
read_when: You are managing sandbox runtimes or debugging sandbox/tool-policy behavior.
status: active
summary: サンドボックスランタイムを管理し、有効なサンドボックスポリシーを確認する
title: サンドボックス CLI
x-i18n:
    generated_at: "2026-07-26T09:56:46Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: ea8311de7702222295f3ba8753304e30f6ed21958e2843f62db5d064f06e24ae
    source_path: cli/sandbox.md
    workflow: 16
---

分離されたエージェント実行用のサンドボックスランタイム（Docker コンテナ、SSH ターゲット、OpenShell バックエンド）を管理します。

## コマンド

### `openclaw sandbox list`

サンドボックスランタイムを、ステータス、バックエンド、設定一致状況、経過時間、アイドル時間、関連付けられたセッション／エージェントとともに一覧表示します。

```bash
openclaw sandbox list
openclaw sandbox list --browser  # ブラウザコンテナのみ
openclaw sandbox list --json
```

### `openclaw sandbox recreate`

現在の設定で強制的に再作成するため、サンドボックスランタイムを削除します。ランタイムは、次回エージェントを使用するときに自動的に再作成されます。

```bash
openclaw sandbox recreate --all
openclaw sandbox recreate --agent mybot        # agent:mybot:* サブセッションを含む
openclaw sandbox recreate --session "agent:main:main"
openclaw sandbox recreate --browser --all      # ブラウザコンテナのみ
openclaw sandbox recreate --all --force        # 確認を省略
```

オプション：

- `--all`：すべてのサンドボックスコンテナを再作成
- `--session <key>`：この正確なスコープキー（`sandbox list` で表示されるもの）を持つランタイムを再作成。短縮名は展開されません
- `--agent <id>`：1 つのエージェントのランタイムを再作成（`agent:<id>` および `agent:<id>:*` に一致）
- `--browser`：ブラウザコンテナのみに適用
- `--force`：確認プロンプトを省略

`--all`、`--session`、`--agent` のいずれか 1 つだけを指定してください。

`ssh` および OpenShell `remote` では、再作成は Docker の場合より重要です。最初のシード後はリモートワークスペースが正規のワークスペースとなり、`recreate` は選択したスコープの正規リモートワークスペースを削除し、次回の実行時に現在のローカルワークスペースから再シードします。

### `openclaw sandbox explain`

有効なサンドボックスのモード／スコープ／ワークスペースアクセス、サンドボックスツールポリシー、および昇格ツールのゲートを、修正用の設定キーパスとともに調査します。

レポートでは、`workspaceRoot` を設定済みのサンドボックスルートとして維持し、有効なホストワークスペース、バックエンドランタイムの作業ディレクトリ、Docker マウントテーブルを個別に表示します。`workspaceAccess: "rw"` の場合、有効なホストワークスペースは `workspaceRoot` 配下のディレクトリではなく、エージェントワークスペースです。

```bash
openclaw sandbox explain
openclaw sandbox explain --session agent:main:main
openclaw sandbox explain --agent work
openclaw sandbox explain --json
```

`recreate --session` とは異なり、これは短いセッション名（たとえば `main`）を受け付け、解決済みのエージェントに基づいて展開します。

## 再作成が必要な理由

サンドボックス設定を更新しても、実行中のコンテナには影響しません。既存のランタイムは以前の設定を維持し、アイドル状態のランタイムは `prune.idleHours`（デフォルトは 24h）後にのみ削除されます。定期的に使用されるエージェントでは、古いランタイムが無期限に存続する可能性があります。`openclaw sandbox recreate` は古いランタイムを削除し、次回使用時に現在の設定から再構築されるようにします。

<Tip>
バックエンド固有の手動クリーンアップよりも `openclaw sandbox recreate` を使用してください。Gateway のランタイムレジストリを使用するため、スコープまたはセッションキーが変更された場合の不一致を回避できます。
</Tip>

## 一般的な契機

| 変更                                                                                                                                                         | コマンド                                                             |
| -------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------- |
| Docker イメージの更新（`agents.defaults.sandbox.docker.image`）                                                                                                   | `openclaw sandbox recreate --all`                                   |
| サンドボックス設定（`agents.defaults.sandbox.*`）                                                                                                                   | `openclaw sandbox recreate --all`                                   |
| SSH ターゲット／認証（`agents.defaults.sandbox.ssh.{target,workspaceRoot,identityFile,certificateFile,knownHostsFile,identityData,certificateData,knownHostsData}`） | `openclaw sandbox recreate --all`                                   |
| OpenShell のソース／ポリシー／モード（`plugins.entries.openshell.config.{from,mode,policy}`）                                                                           | `openclaw sandbox recreate --all`                                   |
| `setupCommand`                                                                                                                                                 | `openclaw sandbox recreate --all`（または 1 つのエージェントには `--agent <id>`） |

<Note>
ランタイムは、次回エージェントが使用されたときに自動的に再作成されます。
</Note>

## レジストリの移行

サンドボックスランタイムのメタデータは、共有 SQLite 状態データベースに保存されます。古いインストールには、通常の読み取りでは書き換えられなくなったレガシーレジストリファイルが存在する場合があります。

- `~/.openclaw/sandbox/containers.json`
- `~/.openclaw/sandbox/browsers.json`
- `~/.openclaw/sandbox/containers/` または `~/.openclaw/sandbox/browsers/` 配下にある、コンテナ／ブラウザごとの 1 つの JSON シャード

`openclaw doctor --fix` を実行して、有効なレガシーエントリを SQLite に移行します。無効なレガシーファイルは隔離されるため、破損した古いレジストリによって現在のランタイムエントリが隠されることはありません。

## 設定

サンドボックス設定は、`~/.openclaw/openclaw.json` 内の `agents.defaults.sandbox` にあります（エージェントごとの上書きは `agents.entries.*.sandbox` に配置します）。

```jsonc
{
  "agents": {
    "defaults": {
      "sandbox": {
        "mode": "all", // off、non-main、all
        "backend": "docker", // docker、ssh、openshell（Plugin によって提供）
        "scope": "agent", // session、agent、shared
        "docker": {
          "image": "openclaw-sandbox:bookworm-slim",
          "containerPrefix": "openclaw-sbx-",
          // ... その他の Docker オプション
        },
        "prune": {
          "idleHours": 24, // 24h のアイドル後に自動削除
          "maxAgeDays": 7, // 7 日後に自動削除
        },
      },
    },
  },
}
```

## 関連項目

- [CLI リファレンス](/ja-JP/cli)
- [サンドボックス化](/ja-JP/gateway/sandboxing)
- [エージェントワークスペース](/ja-JP/concepts/agent-workspace)
- [Doctor](/ja-JP/gateway/doctor)：サンドボックスのセットアップを確認します。
