---
read_when:
    - ローカル Docker ではなく、クラウド管理のサンドボックスを使用する場合
    - OpenShell Plugin をセットアップしています
    - ミラーモードとリモートワークスペースモードのどちらかを選択する必要があります
summary: OpenClaw エージェントのマネージドサンドボックスバックエンドとして OpenShell を使用する
title: OpenShell
x-i18n:
    generated_at: "2026-07-26T09:04:20Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: bf5c33912bd0db759a01cf58ea26712a8ada68c0804bf16f69f1f7cdd496828c
    source_path: gateway/openshell.md
    workflow: 16
---

OpenShell はマネージドサンドボックスバックエンドです。Docker コンテナを
ローカルで実行する代わりに、OpenClaw はサンドボックスのライフサイクルを `openshell` CLI に委任し、
この CLI がリモート環境をプロビジョニングして SSH 経由でコマンドを実行します。

このプラグインは、汎用の [SSH バックエンド](/ja-JP/gateway/sandboxing#ssh-backend)と同じ SSH トランスポートおよびリモートファイルシステムブリッジを再利用し、OpenShell の
ライフサイクル（`sandbox create/get/delete/ssh-config`）と、オプションの `mirror`
ワークスペース同期モードを追加します。

## 前提条件

- OpenShell プラグインがインストール済み（`openclaw plugins install @openclaw/openshell-sandbox`）
- `openshell` CLI が `PATH` に存在（または
  `plugins.entries.openshell.config.command` でカスタムパスを指定）
- サンドボックスへのアクセス権を持つ OpenShell アカウント
- ホスト上で OpenClaw Gateway が実行中

## クイックスタート

```bash
openclaw plugins install @openclaw/openshell-sandbox
```

```json5
{
  agents: {
    defaults: {
      sandbox: {
        mode: "all",
        backend: "openshell",
        scope: "session",
        workspaceAccess: "rw",
      },
    },
  },
  plugins: {
    entries: {
      openshell: {
        enabled: true,
        config: {
          from: "openclaw",
          mode: "remote",
        },
      },
    },
  },
}
```

Gateway を再起動します。次のエージェントターンで OpenClaw が OpenShell
サンドボックスを作成し、ツールの実行をそこにルーティングします。次のコマンドで確認します。

```bash
openclaw sandbox list
openclaw sandbox explain
```

## ワークスペースモード

これは OpenShell に関する最も重要な選択です。

### mirror（デフォルト）

`plugins.entries.openshell.config.mode: "mirror"` は **ローカルワークスペースを
正規のワークスペース**として維持します。

- `exec` の前に、OpenClaw はローカルワークスペースをサンドボックスへ同期します。
- `exec` の後に、OpenClaw はリモートワークスペースをローカルへ同期します。
- ファイルツールはサンドボックスブリッジを経由しますが、ターン間ではローカルが
  信頼できる情報源として維持されます。

開発ワークフローに最適です。OpenClaw 外部で行ったローカル編集は次回の
実行時に反映され、サンドボックスは Docker バックエンドに近い動作をします。

トレードオフ：実行ターンごとにアップロードとダウンロードのコストが発生します。

### remote

`mode: "remote"` は **OpenShell ワークスペースを正規のワークスペース**にします。

- サンドボックスの初回作成時に、OpenClaw はローカルからリモートワークスペースへ
  一度だけ初期データを投入します。
- その後、`exec`、`read`、`write`、`edit`、`apply_patch` は
  リモートワークスペースを直接操作します。OpenClaw はリモートでの変更を
  ローカルへ同期**しません**。
- プロンプト処理時のメディア読み取りも引き続き機能します（ファイル／メディアツールは
  サンドボックスブリッジ経由で読み取ります）。

長時間稼働するエージェントや CI に最適です。ターンごとのオーバーヘッドが低く、
ホスト側のローカル編集によってリモートの状態が暗黙に上書きされることもありません。

<Warning>
初期データ投入後に OpenClaw 外部からホスト上のファイルを編集しても、リモートサンドボックスには反映されません。再度初期データを投入するには `openclaw sandbox recreate` を実行してください。
</Warning>

### モードの選択

|                          | `mirror`                   | `remote`                  |
| ------------------------ | -------------------------- | ------------------------- |
| **正規のワークスペース**  | ローカルホスト                 | リモート OpenShell          |
| **同期方向**       | 双方向（実行ごと） | 1 回限りの初期データ投入             |
| **ターンごとのオーバーヘッド**    | 高い（アップロード＋ダウンロード） | 低い（リモートで直接操作） |
| **ローカル編集を反映？** | はい、次回の実行時          | いいえ、再作成するまで        |
| **最適な用途**             | 開発ワークフロー      | 長時間稼働するエージェント、CI   |

## 設定リファレンス

すべての OpenShell 設定は `plugins.entries.openshell.config` 配下にあります。

| キー                       | 型                     | デフォルト       | 説明                                                                            |
| ------------------------- | ------------------------ | ------------- | -------------------------------------------------------------------------------------- |
| `mode`                    | `"mirror"` または `"remote"` | `"mirror"`    | ワークスペース同期モード                                                                    |
| `command`                 | `string`                 | `"openshell"` | `openshell` CLI のパスまたは名前                                                    |
| `from`                    | `string`                 | `"openclaw"`  | 初回作成時のサンドボックスソース                                                   |
| `gateway`                 | `string`                 | 未設定         | OpenShell Gateway 名（トップレベルの `--gateway`）                                         |
| `gatewayEndpoint`         | `string`                 | 未設定         | OpenShell Gateway エンドポイント（トップレベルの `--gateway-endpoint`）                            |
| `policy`                  | `string`                 | 未設定         | サンドボックス作成用の OpenShell ポリシー ID                                               |
| `providers`               | `string[]`               | `[]`          | サンドボックス作成時に関連付けるプロバイダー名（重複を除去し、各エントリにつき `--provider` フラグを 1 つ指定） |
| `gpu`                     | `boolean`                | `false`       | GPU リソースを要求（`--gpu`）                                                        |
| `autoProviders`           | `boolean`                | `true`        | 作成時に `--auto-providers`（false の場合は `--no-auto-providers`）を渡す            |
| `remoteWorkspaceDir`      | `string`                 | `"/sandbox"`  | サンドボックス内のプライマリ書き込み可能ワークスペース                                          |
| `remoteAgentWorkspaceDir` | `string`                 | `"/agent"`    | エージェントワークスペースのマウントパス（ワークスペースアクセスが `rw` でない場合は読み取り専用）               |
| `timeoutSeconds`          | `number`                 | `120`         | `openshell` CLI 操作のタイムアウト                                                 |

`remoteWorkspaceDir` と `remoteAgentWorkspaceDir` は絶対パスでなければならず、
マネージドルート `/sandbox` または `/agent` の配下に収まる必要があります。それ以外の絶対パスは
拒否されます。

サンドボックスレベルの設定（`mode`、`scope`、`workspaceAccess`）は、他のバックエンドと同様に
`agents.defaults.sandbox` 配下にあります。完全な対応表については
[サンドボックス化](/ja-JP/gateway/sandboxing)を参照してください。

## 例

### 最小構成のリモートセットアップ

```json5
{
  agents: {
    defaults: {
      sandbox: {
        mode: "all",
        backend: "openshell",
      },
    },
  },
  plugins: {
    entries: {
      openshell: {
        enabled: true,
        config: {
          from: "openclaw",
          mode: "remote",
        },
      },
    },
  },
}
```

### GPU を使用する mirror モード

```json5
{
  agents: {
    defaults: {
      sandbox: {
        mode: "all",
        backend: "openshell",
        scope: "agent",
        workspaceAccess: "rw",
      },
    },
  },
  plugins: {
    entries: {
      openshell: {
        enabled: true,
        config: {
          from: "openclaw",
          mode: "mirror",
          gpu: true,
          providers: ["openai"],
          timeoutSeconds: 180,
        },
      },
    },
  },
}
```

### カスタム Gateway を使用するエージェント単位の OpenShell

```json5
{
  agents: {
    defaults: {
      sandbox: { mode: "off" },
    },
    list: [
      {
        id: "researcher",
        sandbox: {
          mode: "all",
          backend: "openshell",
          scope: "agent",
          workspaceAccess: "rw",
        },
      },
    ],
  },
  plugins: {
    entries: {
      openshell: {
        enabled: true,
        config: {
          from: "openclaw",
          mode: "remote",
          gateway: "lab",
          gatewayEndpoint: "https://lab.example",
          policy: "strict",
        },
      },
    },
  },
}
```

## ライフサイクル管理

```bash
# すべてのサンドボックスランタイムを一覧表示（Docker＋OpenShell）
openclaw sandbox list

# 有効なポリシーを確認
openclaw sandbox explain

# 再作成（リモートワークスペースを削除し、次回使用時に初期データを再投入）
openclaw sandbox recreate --all
```

`remote` モードでは、再作成が特に重要です。そのスコープの正規の
リモートワークスペースが削除され、次回使用時にローカルから新しい初期データが
投入されます。`mirror` モードではローカルが正規の状態として維持されるため、再作成は主にリモート実行
環境をリセットします。

次のいずれかを変更した後は再作成してください。

- `agents.defaults.sandbox.backend`
- `plugins.entries.openshell.config.from`
- `plugins.entries.openshell.config.mode`
- `plugins.entries.openshell.config.policy`

## セキュリティ強化

mirror モードのファイルシステムブリッジはローカルワークスペースのルートを固定し、
読み取り、書き込み、ディレクトリ作成、削除、名前変更の前に毎回
正規パスを realpath で再確認し、パス途中のシンボリックリンクを拒否します。シンボリックリンクの差し替えやワークスペースの再マウントによって、
ミラーリングされたツリーの外部へファイルアクセスをリダイレクトすることはできません。

## 現在の制限事項

- OpenShell バックエンドではサンドボックスブラウザはサポートされません。
- `sandbox.docker.binds` は OpenShell には適用されません。バインドが設定されている場合、
  サンドボックスの作成は失敗します。
- `sandbox.docker.*` 配下の Docker 固有のランタイム設定（`env` を除く）は、
  Docker バックエンドにのみ適用されます。

## 仕組み

1. OpenClaw はサンドボックス名に対して `sandbox get` を実行します（設定されている
   `--gateway`/`--gateway-endpoint` を使用）。失敗した場合は
   `sandbox create` でサンドボックスを作成し、`--name`、`--from`、設定されている場合は `--policy`、有効な場合は `--gpu`、
   `--auto-providers`/`--no-auto-providers`、および設定された各プロバイダーにつき 1 つの
   `--provider` フラグを渡します。
2. OpenClaw はサンドボックス名に対して `sandbox ssh-config` を実行し、SSH
   接続情報を取得します。
3. コアは SSH 設定を一時ファイルへ書き込み、汎用 SSH バックエンドと同じ
   リモートファイルシステムブリッジ経由で SSH セッションを開きます。
4. `mirror` モードでは、実行前にローカルからリモートへ同期し、実行後に同期して戻します。
5. `remote` モードでは、作成時に一度だけ初期データを投入し、その後はリモート
   ワークスペースを直接操作します。

## 関連項目

- [サンドボックス化](/ja-JP/gateway/sandboxing) - モード、スコープ、バックエンドの比較
- [サンドボックス、ツールポリシー、昇格の違い](/ja-JP/gateway/sandbox-vs-tool-policy-vs-elevated) - ブロックされたツールのデバッグ
- [マルチエージェントのサンドボックスとツール](/ja-JP/tools/multi-agent-sandbox-tools) - エージェント単位のオーバーライド
- [サンドボックス CLI](/ja-JP/cli/sandbox) - `openclaw sandbox` コマンド
