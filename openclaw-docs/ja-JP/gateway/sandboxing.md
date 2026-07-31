---
read_when: You want a dedicated explanation of sandboxing or need to tune agents.defaults.sandbox.
sidebarTitle: Sandboxing
status: active
summary: OpenClaw のサンドボックスの仕組み：モード、スコープ、ワークスペースへのアクセス、イメージ
title: サンドボックス化
x-i18n:
    generated_at: "2026-07-26T09:36:00Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: a3668dc512a8ff30732290ee68e9dd29a3a2e9c106e6e39077a97bfbd90098f7
    source_path: gateway/sandboxing.md
    workflow: 16
---

OpenClaw は、影響範囲を縮小するため、サンドボックスバックエンド内でツールを実行できます。サンドボックス化はデフォルトで無効であり、`agents.defaults.sandbox`（グローバル）または `agents.entries.*.sandbox`（エージェントごと）で制御します。Gateway プロセスは常にホスト上に留まり、有効にした場合にサンドボックスへ移動するのはツールの実行だけです。

<Note>
これは完全なセキュリティ境界ではありませんが、モデルが不適切な動作をした場合のファイルシステムおよびプロセスへのアクセスを大幅に制限します。
</Note>

## サンドボックス化されるもの

- ツールの実行：`exec`、`read`、`write`、`edit`、`apply_patch`、`process` など。
- オプションのサンドボックス化されたブラウザ（`agents.defaults.sandbox.browser`）。

サンドボックス化されないもの：

- Gateway プロセス自体。
- `tools.elevated` を介してサンドボックス外での実行を明示的に許可されたツール。昇格 exec はサンドボックス化を迂回し、設定されたエスケープパス（デフォルトでは `gateway`、exec ターゲットが `node` の場合は `node`）で実行されます。サンドボックス化が無効の場合、exec はすでにホスト上で実行されるため、`tools.elevated` を指定しても何も変わりません。[昇格モード](/ja-JP/tools/elevated)を参照してください。

## モード、スコープ、バックエンド

サンドボックスの動作は、相互に独立した 3 つの設定で制御します。

| 設定 | キー                               | 値                       | デフォルト  |
| ------- | --------------------------------- | ---------------------------- | -------- |
| モード    | `agents.defaults.sandbox.mode`    | `off`、`non-main`、`all`     | `off`    |
| スコープ   | `agents.defaults.sandbox.scope`   | `agent`、`session`、`shared` | `agent`  |
| バックエンド | `agents.defaults.sandbox.backend` | `docker`、`ssh`、`openshell` | `docker` |

**モード**は、サンドボックス化を適用するタイミングを制御します。

- `off`：サンドボックス化しません。
- `non-main`：エージェントのメインセッションを除くすべてのセッションをサンドボックス化します。メインセッションのキーは常に `agent:<agentId>:main`（`session.scope` が `"global"` の場合は `global`）であり、設定できません。グループ／チャンネルセッションは独自のキーを使用するため、常に非メインセッションと見なされ、サンドボックス化されます。
- `all`：すべてのセッションをサンドボックス内で実行します。

**スコープ**は、作成するコンテナ／環境の数を制御します。

- `agent`：エージェントごとに 1 つのコンテナ。
- `session`：セッションごとに 1 つのコンテナ。
- `shared`：サンドボックス化されたすべてのセッションで 1 つのコンテナを共有します（このスコープでは、エージェントごとの `docker`／`ssh`／`browser` オーバーライドは無視されます）。

**バックエンド**は、サンドボックス化されたツールを実行するランタイムを制御します。SSH 固有の設定は `agents.defaults.sandbox.ssh`、OpenShell 固有の設定は `plugins.entries.openshell.config` にあります。

|                     | Docker                           | SSH                            | OpenShell                                           |
| ------------------- | -------------------------------- | ------------------------------ | --------------------------------------------------- |
| **実行場所**   | ローカルコンテナ                  | SSH でアクセス可能な任意のホスト        | OpenShell 管理のサンドボックス                           |
| **セットアップ**           | `scripts/sandbox-setup.sh`       | SSH キー＋ターゲットホスト          | OpenShell Plugin を有効化                            |
| **ワークスペースモデル** | バインドマウントまたはコピー               | リモートを正本とする（初回のみシード）   | `mirror` または `remote`                                |
| **ネットワーク制御** | `docker.network`（デフォルト：なし） | リモートホストに依存         | OpenShell に依存                                |
| **ブラウザサンドボックス** | 対応                        | 非対応                  | 現時点では非対応                                   |
| **バインドマウント**     | `docker.binds`                   | 該当なし                            | 該当なし                                                 |
| **最適な用途**        | ローカル開発、完全な分離        | リモートマシンへのオフロード | オプションの双方向同期を備えた管理対象リモートサンドボックス |

## Docker バックエンド

サンドボックス化を有効にすると、Docker がデフォルトのバックエンドになります。Docker デーモンソケット（`/var/run/docker.sock`）を介してツールとサンドボックスブラウザをローカルで実行し、Docker の名前空間によって分離します。

デフォルト：`network: "none"`（外向き通信なし）、`readOnlyRoot: true`、`capDrop: ["ALL"]`、イメージ `openclaw-sandbox:bookworm-slim`。

ホストの GPU を公開するには、`agents.defaults.sandbox.docker.gpus`（またはエージェントごとのオーバーライド）を `"all"` や `"device=GPU-uuid"` などの値に設定します。これは Docker の `--gpus` フラグに渡され、NVIDIA Container Toolkit など、互換性のあるホストランタイムが必要です。

<Warning>
**Docker-out-of-Docker（DooD）の制約**

OpenClaw Gateway 自体を Docker コンテナとしてデプロイする場合、ホストの Docker ソケット（DooD）を使用して兄弟サンドボックスコンテナをオーケストレーションします。これにより、パスマッピングに関する次の制約が生じます。

- **設定にはホストのパスが必要**：`openclaw.json` `workspace` には、Gateway コンテナ内部のパスではなく、**ホストの絶対パス**（例：`/home/user/.openclaw/workspaces`）を指定する必要があります。Docker デーモンは、Gateway 自体の名前空間ではなく、ホスト OS の名前空間を基準にパスを評価します。
- **一致するボリュームマッピングが必要**：Gateway プロセスも、その `workspace` パスに Heartbeat ファイルとブリッジファイルを書き込みます。同じホストパスが Gateway コンテナ内からも正しく解決されるように、Gateway コンテナに同一のボリュームマッピング（`-v /home/user/.openclaw:/home/user/.openclaw`）を設定してください。マッピングが一致しない場合、Gateway が Heartbeat を書き込もうとした際に `EACCES` が発生します。
- **Codex コードモード**：OpenClaw サンドボックスが有効な場合、そのターンでは Codex app-server ネイティブのコードモード、ユーザー MCP サーバー、およびアプリを基盤とする Plugin の実行を OpenClaw が無効にします（これらは OpenClaw サンドボックスバックエンドではなく、Gateway ホスト上の app-server プロセスから実行されるためです）。ただし、サンドボックスのツールポリシーで必要なツールを公開し、実験的なサンドボックス exec-server パスを明示的に有効化した場合を除きます。その場合、シェルアクセスは `sandbox_exec` や `sandbox_process` など、OpenClaw のサンドボックスを基盤とするツールを経由します。ホストの Docker ソケットを、エージェントのサンドボックスコンテナやカスタム Codex サンドボックスにマウントしないでください。完全な動作については、[Codex ハーネス](/ja-JP/plugins/codex-harness)を参照してください。

Docker サンドボックスモードを有効にした Ubuntu／AppArmor ホストでは、Codex app-server の `workspace-write` シェル実行に、サンドボックスコンテナ内の非特権ユーザー名前空間が必要です。サービスユーザーがこれを作成できない場合、シェルの起動前に失敗する可能性があります。Docker サンドボックスの外向き通信が無効（`network: "none"`、デフォルト）の場合は、非特権ネットワーク名前空間も必要です。一般的な症状は `bwrap: setting up uid map: Permission denied` および `bwrap: loopback: Failed RTM_NEWADDR: Operation not permitted` です。`openclaw doctor` を実行してください。Codex bwrap 名前空間プローブの失敗が報告された場合は、OpenClaw サービスプロセスに必要な名前空間を許可する AppArmor プロファイルを推奨します。`kernel.apparmor_restrict_unprivileged_userns=0` はセキュリティ上のトレードオフを伴うホスト全体のフォールバックです。そのホストのセキュリティ方針として許容できる場合にのみ使用してください。
</Warning>

### サンドボックス化されたブラウザ

- ブラウザツールで必要になると、サンドボックスブラウザが自動起動します（CDP に到達可能であることを保証します）。`agents.defaults.sandbox.browser.autoStart`（デフォルト `true`）および `autoStartTimeoutMs`（デフォルト 12s）で設定します。
- サンドボックスブラウザコンテナは、グローバルな `bridge` ネットワークではなく、専用の Docker ネットワーク（`openclaw-sandbox-browser`）を使用します。`agents.defaults.sandbox.browser.network` で設定します。
- `agents.defaults.sandbox.browser.cdpSourceRange` は、CIDR 許可リスト（例：`172.21.0.1/32`）によって、コンテナ境界での CDP 受信アクセスを制限します。
- noVNC の監視アクセスはデフォルトでパスワード保護されています。OpenClaw は、有効期間の短いトークン URL を発行します。この URL はローカルのブートストラップページを配信し、URL フラグメント（クエリ文字列やヘッダーログではありません）にパスワードを含めて noVNC を開きます。
- `agents.defaults.sandbox.browser.allowHostControl`（デフォルト `false`）を使用すると、サンドボックス化されたセッションからホストブラウザを明示的に対象にできます。
- オプションの許可リストで `target: "custom"` を制限します：`allowedControlUrls`、`allowedControlHosts`、`allowedControlPorts`。

## SSH バックエンド

任意の SSH アクセス可能なマシン上で `exec`、ファイルツール、メディア読み取りをサンドボックス化するには、`backend: "ssh"` を使用します。

```json5
{
  agents: {
    defaults: {
      sandbox: {
        mode: "all",
        backend: "ssh",
        scope: "session",
        workspaceAccess: "rw",
        ssh: {
          target: "user@gateway-host:22",
          workspaceRoot: "/tmp/openclaw-sandboxes",
          strictHostKeyChecking: true,
          updateHostKeys: true,
          identityFile: "~/.ssh/id_ed25519",
          certificateFile: "~/.ssh/id_ed25519-cert.pub",
          knownHostsFile: "~/.ssh/known_hosts",
          // または、ローカルファイルの代わりに SecretRefs／インラインコンテンツを使用します：
          // identityData: { source: "env", provider: "default", id: "SSH_IDENTITY" },
          // certificateData: { source: "env", provider: "default", id: "SSH_CERTIFICATE" },
          // knownHostsData: { source: "env", provider: "default", id: "SSH_KNOWN_HOSTS" },
        },
      },
    },
  },
}
```

デフォルト：`command: "ssh"`、`workspaceRoot: "/tmp/openclaw-sandboxes"`、`strictHostKeyChecking: true`、`updateHostKeys: true`。

- **ライフサイクル**：OpenClaw は、`sandbox.ssh.workspaceRoot` 配下にスコープごとのリモートルートを作成します。作成または再作成後の初回使用時に、ローカルワークスペースからリモートワークスペースへ一度だけシードします。それ以降、`exec`、`read`、`write`、`edit`、`apply_patch`、プロンプトメディアの読み取り、および受信メディアのステージングは、SSH 経由でリモートワークスペースを直接操作します。OpenClaw は、リモートでの変更をローカルワークスペースへ自動的に同期しません。
- **認証情報**：`identityFile`／`certificateFile`／`knownHostsFile` は既存のローカルファイルを参照します。`identityData`／`certificateData`／`knownHostsData` は、インライン文字列または SecretRefs を受け付けます。これらは通常のシークレットランタイムスナップショットを介して解決され、モード `0600` の一時ファイルに書き込まれ、SSH セッションの終了時に削除されます。同じ項目に `*File` と `*Data` の両方のバリアントが設定されている場合、そのセッションでは `*Data` が優先されます。
- **リモートを正本とすることによる影響**：最初のシード後は、リモート SSH ワークスペースが実際のサンドボックス状態になります。シード手順の後に OpenClaw 外部で行ったホストローカルの編集は、サンドボックスを再作成するまでリモートには反映されません。`openclaw sandbox recreate` はスコープごとのリモートルートを削除し、次回使用時にローカルから再度シードします。このバックエンドではブラウザのサンドボックス化はサポートされず、`sandbox.docker.*` の設定も適用されません。

## OpenShell バックエンド

OpenShell が管理するリモート環境内でツールをサンドボックス化するには、`backend: "openshell"` を使用します。OpenShell は汎用 SSH バックエンドと同じ SSH トランスポートおよびリモートファイルシステムブリッジを再利用し、OpenShell ライフサイクル（`sandbox create/get/delete/ssh-config`）と、オプションの `mirror` ワークスペース同期モードを追加します。

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
          mode: "remote", // mirror | remote
        },
      },
    },
  },
}
```

`mode: "mirror"`（デフォルト）は、ローカルワークスペースを正規のソースとして維持します。OpenClaw は `exec` の前にローカルからサンドボックスへ同期し、その後に同期を戻します。`mode: "remote"` は、最初に一度だけローカルからリモートワークスペースへ初期データを投入し、その後は同期を戻さずに `exec`/`read`/`write`/`edit`/`apply_patch` をリモートワークスペースに対して直接実行します。初期データ投入後のローカル編集は、`openclaw sandbox recreate` を実行するまで反映されません。`scope: "agent"` または `scope: "shared"` では、そのリモートワークスペースは同じスコープで共有されます。現在の制限事項として、サンドボックスブラウザはまだサポートされておらず、`sandbox.docker.binds` はこのバックエンドには適用されません。

`openclaw sandbox list`/`recreate`/prune は、いずれも OpenShell ランタイムを Docker ランタイムと同様に扱います。prune ロジックはバックエンドを認識します。

完全な前提条件、設定リファレンス、ワークスペースモードの比較、ライフサイクルの詳細については、[OpenShell](/ja-JP/gateway/openshell)を参照してください。

## ワークスペースへのアクセス

`agents.defaults.sandbox.workspaceAccess` は、サンドボックスから参照できる範囲を制御します。

| 値            | 動作                                                                                  |
| ---------------- | ----------------------------------------------------------------------------------------- |
| `none`（デフォルト） | ツールは `~/.openclaw/sandboxes` 配下の分離されたサンドボックスワークスペースを参照します。                    |
| `ro`             | エージェントワークスペースを `/agent` に読み取り専用でマウントします（`write`/`edit`/`apply_patch` は無効になります）。 |
| `rw`             | エージェントワークスペースを `/workspace` に読み書き可能でマウントします。                                    |

OpenShell バックエンドでは、`mirror` モードは引き続き exec ターン間でローカルワークスペースを正規のソースとして使用し、`remote` モードは最初の初期データ投入後にリモート OpenShell ワークスペースを正規のソースとして使用します。また、`workspaceAccess: "ro"`/`"none"` は引き続き同じ方法で書き込み動作を制限します。

受信メディアは、アクティブなサンドボックスワークスペース（`media/inbound/*`）にコピーされます。

<Note>
**Skills**：`read` ツールのルートはサンドボックスです。`workspaceAccess: "none"` では、OpenClaw は読み取り可能にするため、対象となる Skills をサンドボックスワークスペース（`.../skills`）へミラーリングします。`"rw"` では、ワークスペースの Skills は `/workspace/skills` から読み取り可能で、対象となる管理対象、バンドル済み、または Plugin の Skills は、生成された読み取り専用パス `/workspace/.openclaw/sandbox-skills/skills` に配置されます。
</Note>

## 1 つのエージェントで複数のフォルダを使用する

サンドボックス化された 1 つのエージェントがプライマリワークスペース以外にもアクセスする必要がある場合は、Docker バインドマウントを使用します。各エントリは、明示的なアクセスモードを指定してホストフォルダをコンテナパスに対応付けます。

```text
host-directory:container-directory:ro
host-directory:container-directory:rw
```

- `ro` は、マウントされたフォルダをサンドボックス内で読み取り専用にします。
- `rw` は、サンドボックス化されたツールとプロセスによるホストフォルダの変更を許可します。
- コンテナパスは、エージェントが使用するパスです。ホストパスは自動的には公開されません。

次の例では、`research` エージェントに、書き込み可能なプライマリワークスペース、`/reference` にある読み取り専用の参考資料、`/drafts` にある独立した書き込み可能な出力フォルダを提供します。

```json5
{
  agents: {
    defaults: {
      sandbox: {
        mode: "all",
        scope: "agent",
      },
    },
    list: [
      {
        id: "research",
        workspace: "/srv/openclaw/research-workspace",
        sandbox: {
          workspaceAccess: "rw",
          docker: {
            binds: ["/srv/shared/reference:/reference:ro", "/srv/shared/drafts:/drafts:rw"],
            // これらのソースはエージェントワークスペース外にあるため必須です。
            dangerouslyAllowExternalBindSources: true,
          },
        },
      },
    ],
  },
}
```

`workspaceAccess` とバインドモードは独立しています。

| 設定                          | 制御対象                                                                    |
| -------------------------------- | --------------------------------------------------------------------------- |
| `workspaceAccess: "none"`        | 分離されたサンドボックスワークスペースを使用し、エージェントワークスペースを公開しません。    |
| `workspaceAccess: "ro"`          | エージェントワークスペースを `/agent` に読み取り専用でマウントします。                           |
| `workspaceAccess: "rw"`          | エージェントワークスペースを `/workspace` に読み書き可能でマウントします。                      |
| `docker.binds` エントリの `:ro`/`:rw` | 設定されたコンテナパスにある、その追加ホストフォルダのみを制御します。 |

`workspaceAccess` を変更しても、追加のバインドが `ro` から `rw` に、またはその逆に変更されることはありません。グローバルとエージェント単位の `docker.binds` はマージされます。エージェント単位のバインドには `scope: "agent"` または `"session"` を維持してください。`scope: "shared"` はエージェント単位の Docker オーバーライドをすべて無視し、グローバルバインドのみを使用します。

バインドマウントがサポート対象の複数フォルダ境界である理由は、Docker がマウント分離を使用してコンテナのファイルシステムビューを構築し、`ro`/`rw` モードがサンドボックス内のすべてのプロセスに適用されるためです。この境界は、OpenClaw の各コードパスでパス認可チェックを重複実装することなく、`exec`、ファイルシステムツール、子プロセス、ライブラリを対象にします。許可されたシェルまたは依存関係がファイルへ直接アクセスできる場合、ホスト側のパス許可リストでは同等の完全な境界を提供できません。

オプトインの `dangerouslyAllowExternalBindSources` は、ワークスペースルート外のソースを許可するだけです。OpenClaw によるシステム、認証情報、Docker ソケット、シンボリックリンクの親、予約済みターゲットのブロックチェックを無効にはしません。最小限のフォルダを選び、書き込みが必要でない限り `ro` を使用し、マウントの変更後はサンドボックスを再作成してください。

```bash
openclaw sandbox recreate --agent research
```

### その他のバインド動作

`agents.defaults.sandbox.docker.binds` はグローバルマウントを設定します。形式は同じ `host:container:mode` 形式です（例：`"/home/user/source:/source:rw"`）。

`agents.defaults.sandbox.browser.binds` は、追加のホストディレクトリを **サンドボックスブラウザ** コンテナのみにマウントします。設定されている場合（`[]` を含む）、ブラウザコンテナでは `docker.binds` を置き換えます。省略した場合、ブラウザコンテナは `docker.binds` にフォールバックします。

```json5
{
  agents: {
    defaults: {
      sandbox: {
        docker: {
          binds: ["/home/user/source:/source:ro", "/var/data/myapp:/data:ro"],
        },
      },
    },
    list: [
      {
        id: "build",
        sandbox: {
          docker: {
            binds: ["/mnt/cache:/cache:rw"],
          },
        },
      },
    ],
  },
}
```

<Warning>
**バインドのセキュリティ**

- バインドはサンドボックスのファイルシステムを迂回します。設定したモード（`:ro` または `:rw`）でホストパスを公開します。
- OpenClaw は、危険なバインドソースをデフォルトでブロックします。対象には、システムパス（`/etc`、`/proc`、`/sys`、`/dev`、`/root`、`/boot`）、Docker ソケットディレクトリ（`/run`、`/var/run`、およびそれらの `docker.sock` バリアント）、一般的なホームディレクトリ内の認証情報ルート（`~/.aws`、`~/.cargo`、`~/.config`、`~/.docker`、`~/.gnupg`、`~/.netrc`、`~/.npm`、`~/.ssh`）が含まれます。
- 検証ではソースパスを正規化した後、存在する最深の祖先を通じて再度解決してから、ブロック対象パスと許可ルートを再チェックします。そのため、最終リーフがまだ存在しない場合でも、シンボリックリンクの親を使用した脱出はフェイルクローズになります（例：`run-link` が対象を指している場合、`/workspace/run-link/new-file` は引き続き `/var/run/...` として解決されます）。
- 予約済みのコンテナマウントポイント（`/workspace`、`/agent`）を隠すバインドターゲットも、デフォルトでブロックされます。`agents.defaults.sandbox.docker.dangerouslyAllowReservedContainerTargets: true` でオーバーライドできます。
- ワークスペースまたはエージェントワークスペースの許可リスト対象ルート外にあるバインドソースは、デフォルトでブロックされます。`agents.defaults.sandbox.docker.dangerouslyAllowExternalBindSources: true` でオーバーライドできます。許可ルートも同じ方法で正規化されるため、シンボリックリンクを解決する前だけ許可リスト内に見えるパスも、許可ルート外として拒否されます。
- 機密性の高いマウント（シークレット、SSH キー、サービス認証情報）は、絶対に必要な場合を除き `:ro` にする必要があります。
- ワークスペースへの読み取りアクセスだけが必要な場合は、`workspaceAccess: "ro"` と組み合わせてください。バインドモードは引き続き独立しています。
- バインドとツールポリシーおよび昇格 exec の相互作用については、[サンドボックス、ツールポリシー、昇格の比較](/ja-JP/gateway/sandbox-vs-tool-policy-vs-elevated)を参照してください。

</Warning>

## イメージとセットアップ

デフォルトの Docker イメージ：`openclaw-sandbox:bookworm-slim`

<Note>
**ソースチェックアウトと npm インストールの比較**

`scripts/sandbox-setup.sh`、`scripts/sandbox-common-setup.sh`、`scripts/sandbox-browser-setup.sh` ヘルパースクリプトは、[ソースチェックアウト](https://github.com/openclaw/openclaw)から実行する場合のみ使用できます。npm パッケージには含まれていません。

`npm install -g openclaw` 経由で OpenClaw をインストールした場合は、代わりに以下に示すインラインの `docker build` コマンドを使用してください。
</Note>

<Steps>
  <Step title="デフォルトイメージをビルドする">
    ソースチェックアウトの場合：

    ```bash
    scripts/sandbox-setup.sh
    ```

    npm インストールの場合（ソースチェックアウトは不要）：

    ```bash
    docker build -t openclaw-sandbox:bookworm-slim - <<'DOCKERFILE'
    FROM debian:bookworm-slim
    ENV DEBIAN_FRONTEND=noninteractive
    RUN apt-get update && apt-get install -y --no-install-recommends \
      bash ca-certificates curl git jq python3 ripgrep \
      && rm -rf /var/lib/apt/lists/*
    RUN useradd --create-home --shell /bin/bash sandbox
    USER sandbox
    WORKDIR /home/sandbox
    CMD ["sleep", "infinity"]
    DOCKERFILE
    ```

    デフォルトイメージには Node が含まれていません。Skills で Node（またはその他のランタイム）が必要な場合は、カスタムイメージに組み込むか、`sandbox.docker.setupCommand` を介してインストールしてください（ネットワークへの外向き通信、書き込み可能なルート、root ユーザーが必要です）。

    `openclaw-sandbox:bookworm-slim` が存在しない場合、OpenClaw が通常の `debian:bookworm-slim` を暗黙的に代用することはありません。デフォルトイメージを対象とするサンドボックス実行は、イメージをビルドするまでビルド手順を表示して即座に失敗します。これは、バンドル済みイメージにサンドボックスの書き込み・編集ヘルパー用の `python3` が含まれているためです。

  </Step>
  <Step title="任意：共通イメージをビルドする">
    一般的なツール（例：`curl`、`jq`、Node 24、pnpm、`python3`、`git`）を備えた、より多機能なサンドボックスイメージを使用する場合：

    ソースチェックアウトの場合：

    ```bash
    scripts/sandbox-common-setup.sh
    ```

    npm インストールの場合は、まずデフォルトイメージをビルドし（前述を参照）、次にリポジトリの [`scripts/docker/sandbox/Dockerfile.common`](https://github.com/openclaw/openclaw/blob/main/scripts/docker/sandbox/Dockerfile.common) を使用して、その上に共通イメージをビルドします。

    次に、`agents.defaults.sandbox.docker.image` を `openclaw-sandbox-common:bookworm-slim` に設定します。

  </Step>
  <Step title="任意：サンドボックスブラウザのイメージをビルドする">
    ソースチェックアウトの場合：

    ```bash
    scripts/sandbox-browser-setup.sh
    ```

    npm インストールの場合は、リポジトリの [`scripts/docker/sandbox/Dockerfile.browser`](https://github.com/openclaw/openclaw/blob/main/scripts/docker/sandbox/Dockerfile.browser) を使用してビルドします。

  </Step>
</Steps>

デフォルトでは、Docker サンドボックスコンテナは **ネットワークなし** で実行されます。`agents.defaults.sandbox.docker.network` でオーバーライドできます。

<AccordionGroup>
  <Accordion title="サンドボックスブラウザにおける Chromium のデフォルト">
    バンドル済みのサンドボックスブラウザイメージは、コンテナ化されたワークロード向けに保守的な Chromium 起動フラグを適用します。

    - `--remote-debugging-address=127.0.0.1`
    - `--remote-debugging-port=<derived from OPENCLAW_BROWSER_CDP_PORT>`
    - `--user-data-dir=${HOME}/.chrome`
    - `--no-first-run`
    - `--no-default-browser-check`
    - `--disable-dev-shm-usage`
    - `--disable-background-networking`
    - `--disable-breakpad`
    - `--disable-crash-reporter`
    - `--no-zygote`
    - `--metrics-recording-only`
    - `--password-store=basic`
    - `--use-mock-keychain`
    - `--headless=new`（`browser.headless` が有効な場合）。
    - `--no-sandbox --disable-setuid-sandbox`（`browser.noSandbox` が有効な場合）。
    - デフォルトでは `--disable-3d-apis`、`--disable-gpu`、`--disable-software-rasterizer`。これらのグラフィックス強化フラグは、GPU をサポートしないコンテナに役立ちます。ワークロードで WebGL やその他の 3D 機能が必要な場合は、`OPENCLAW_BROWSER_DISABLE_GRAPHICS_FLAGS=0` を設定してください。
    - デフォルトでは `--disable-extensions`。拡張機能に依存するフローでは `OPENCLAW_BROWSER_DISABLE_EXTENSIONS=0` を設定してください。
    - デフォルトでは `--renderer-process-limit=2`。`OPENCLAW_BROWSER_RENDERER_PROCESS_LIMIT=<N>` で制御され、`0` にすると Chromium のデフォルトが維持されます。

    別のランタイムプロファイルが必要な場合は、カスタムブラウザーイメージを使用し、独自のエントリポイントを指定してください。ローカル（コンテナ外）の Chromium プロファイルでは、追加の起動フラグを付加するために `browser.extraArgs` を使用してください。

  </Accordion>
  <Accordion title="ネットワークセキュリティのデフォルト">
    - `network: "host"` はブロックされます。
    - `network: "container:<id>"` はデフォルトでブロックされます（名前空間への参加によるバイパスのリスク）。
    - 緊急時のオーバーライド: `agents.defaults.sandbox.docker.dangerouslyAllowContainerNamespaceJoin: true`。

  </Accordion>
</AccordionGroup>

Docker のインストールとコンテナ化された Gateway については、こちらを参照してください: [Docker](/ja-JP/install/docker)

Docker Gateway のデプロイでは、`scripts/docker/setup.sh` でサンドボックス設定を初期構成できます。このパスを有効にするには、`OPENCLAW_SANDBOX=1`（または `true`/`yes`/`on`）を設定します。ソケットの場所を変更するには `OPENCLAW_DOCKER_SOCKET` を使用します。完全なセットアップと環境変数のリファレンス: [Docker](/ja-JP/install/docker#agent-sandbox)。

## setupCommand（コンテナの一度限りのセットアップ）

`setupCommand` は、サンドボックスコンテナの作成後に **1 回だけ** 実行されます（実行のたびではありません）。コンテナ内で `sh -lc` を介して実行されます。

パス:

- グローバル: `agents.defaults.sandbox.docker.setupCommand`
- エージェントごと: `agents.entries.*.sandbox.docker.setupCommand`

<AccordionGroup>
  <Accordion title="よくある落とし穴">
    - デフォルトの `docker.network` は `"none"`（外部への通信なし）であるため、パッケージのインストールは失敗します。
    - `docker.network: "container:<id>"` には `dangerouslyAllowContainerNamespaceJoin: true` が必要で、緊急時にのみ使用してください。
    - `readOnlyRoot: true` は書き込みを禁止します。`readOnlyRoot: false` を設定するか、カスタムイメージをビルドしてください。
    - パッケージをインストールするには、`user` が root である必要があります（`user` を省略するか、`user: "0:0"` を設定します）。
    - サンドボックスでの exec は、ホストの `process.env` を継承**しません**。Skill の API キーには `agents.defaults.sandbox.docker.env`（またはカスタムイメージ）を使用してください。
    - `agents.defaults.sandbox.docker.env` の値は、明示的な Docker コンテナ環境変数として渡されます。Docker デーモンにアクセスできるユーザーは、`docker inspect` などの Docker メタデータコマンドでこれらを確認できます。このメタデータへの露出を許容できない場合は、カスタムイメージ、マウントされたシークレットファイル、または別のシークレット配信経路を使用してください。

  </Accordion>
</AccordionGroup>

## ツールポリシーとエスケープハッチ

ツールの許可・拒否ポリシーは、サンドボックスのルールより先に適用されます。ツールがグローバルまたはエージェント単位で拒否されている場合、サンドボックス化しても利用可能にはなりません。

`tools.elevated` は、`exec` をサンドボックス外で実行する明示的なエスケープハッチです（デフォルトでは `gateway`、exec のターゲットが `node` の場合は `node`）。`/exec` ディレクティブは承認済みの送信者にのみ適用され、セッションごとに保持されます。`exec` を完全に無効化するには、ツールポリシーで拒否してください（[サンドボックス、ツールポリシー、Elevated の比較](/ja-JP/gateway/sandbox-vs-tool-policy-vs-elevated)を参照）。

デバッグ:

- `openclaw sandbox list` は、サンドボックスコンテナ、ステータス、イメージの一致状況、経過時間、アイドル時間、および関連付けられたセッション/エージェントを表示します。
- `openclaw sandbox explain [--session <key>] [--agent <id>]` は、実際に適用されるサンドボックスモード、ホストワークスペース、ランタイム作業ディレクトリ、Docker マウント、ツールポリシー、および修正用の設定キーを検査します。その `workspaceRoot` フィールドは設定されたサンドボックスルートを示し、`effectiveHostWorkspaceRoot` はアクティブなワークスペースが実際に配置されている場所を示します。
- `openclaw sandbox recreate [--all | --session <key> | --agent <id>] [--browser] [--force]` はコンテナ/環境を削除し、次回使用時に現在の設定で再作成されるようにします。
- 「なぜこれがブロックされるのか？」を理解するための考え方については、[サンドボックス、ツールポリシー、Elevated の比較](/ja-JP/gateway/sandbox-vs-tool-policy-vs-elevated)を参照してください。

## マルチエージェントのオーバーライド

各エージェントは、サンドボックスとツールをオーバーライドできます: `agents.entries.*.sandbox` および `agents.entries.*.tools`（さらに、サンドボックスのツールポリシーには `agents.entries.*.tools.sandbox.tools`）。優先順位については、[マルチエージェントのサンドボックスとツール](/ja-JP/tools/multi-agent-sandbox-tools)を参照してください。

## 最小限の有効化例

```json5
{
  agents: {
    defaults: {
      sandbox: {
        mode: "non-main",
        scope: "session",
        workspaceAccess: "none",
      },
    },
  },
}
```

## 関連項目

- [マルチエージェントのサンドボックスとツール](/ja-JP/tools/multi-agent-sandbox-tools) -- エージェントごとのオーバーライドと優先順位
- [OpenShell](/ja-JP/gateway/openshell) -- マネージドサンドボックスバックエンドのセットアップ、ワークスペースモード、設定リファレンス
- [サンドボックス設定](/ja-JP/gateway/config-agents#agentsdefaultssandbox)
- [サンドボックス、ツールポリシー、Elevated の比較](/ja-JP/gateway/sandbox-vs-tool-policy-vs-elevated) -- 「なぜこれがブロックされるのか？」のデバッグ
- [セキュリティ](/ja-JP/gateway/security)
