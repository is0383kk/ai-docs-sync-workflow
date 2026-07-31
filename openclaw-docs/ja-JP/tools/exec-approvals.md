---
read_when:
    - exec の承認または許可リストの設定
    - macOS アプリでの exec 承認 UX の実装
    - サンドボックス脱出を促すプロンプトとその影響のレビュー
sidebarTitle: Exec approvals
summary: ホストでのコマンド実行承認：ポリシー設定、許可リスト、YOLO/厳格ワークフロー
title: 実行の承認
x-i18n:
    generated_at: "2026-07-26T09:20:29Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 2bd09746375061232e9094b8803d33859cac4c13c7bde14a059b7d52e48b5de8
    source_path: tools/exec-approvals.md
    workflow: 16
---

Exec 承認は、サンドボックス化されたエージェントが実ホスト（`gateway` または `node`）上でコマンドを実行できるようにするための **コンパニオンアプリ / Node ホストのガードレール**です。コマンドは、ポリシー、許可リスト、および（任意の）ユーザー承認のすべてが一致した場合にのみ実行されます。
承認は、ツールポリシーと昇格ゲートの **上に重ねて** 適用されます（昇格された
`full` は承認をスキップします）。

`deny`、`allowlist`、`ask`、`auto`、`full`、
Codex Guardian のマッピング、および ACPX ハーネス権限についてモードを軸に確認するには、
[権限モード](/ja-JP/tools/permission-modes)を参照してください。

<Note>
有効なポリシーは、`tools.exec.*` と承認の
デフォルトのうち、より **厳しい** 方です。承認によって設定から導出されたセキュリティ/確認要件を厳しくすることはできますが、
緩和することはできません。承認フィールドを省略すると、`tools.exec` の値が
使用されます。ホストでの Exec は、そのマシンのローカル承認状態も使用します。実行ホストの承認ファイル内にホストローカルの `ask: "always"` がある場合、
セッションまたは設定のデフォルトが `ask: "on-miss"` を要求していても、確認が継続されます。
</Note>

## 適用範囲

Exec 承認は、実行ホスト上でローカルに適用されます。

- **Gateway ホスト** -> Gateway マシン上の `openclaw` プロセス。
- **Node ホスト** -> Node ランナー（macOS コンパニオンアプリまたはヘッドレス Node ホスト）。

### 信頼モデル

- Gateway で認証された呼び出し元は、その Gateway の信頼されたオペレーターです。
- ペアリングされた Node は、その信頼されたオペレーターの権限を Node ホストに拡張します。
- 承認は偶発的な実行のリスクを軽減しますが、ユーザー単位の認証境界やファイルシステムの読み取り専用ポリシーでは **ありません**。
- 承認されると、コマンドは選択されたホストまたはサンドボックスのファイルシステム権限に従ってファイルを変更できます。
- 承認された Node ホストでの実行には、正規化された実行コンテキスト（cwd、正確な argv、存在する場合は環境変数のバインディング、および該当する場合は固定された実行可能ファイルのパス）が結び付けられます。
- シェルスクリプトおよびインタープリター/ランタイムによるファイルの直接実行では、OpenClaw は具体的なローカルファイルのオペランドを 1 つ結び付けることも試みます。承認後から実行前までの間にそのファイルが変更された場合、変更後の内容を実行するのではなく、実行を拒否します。
- ファイルのバインディングはベストエフォートであり、すべてのインタープリター/ランタイムのロード経路を完全にモデル化するものではありません。具体的なローカルファイルを正確に 1 つ特定できない場合、OpenClaw は完全にカバーしているかのように扱わず、承認に基づく実行の生成を拒否します。

### macOS での分担

- **Node ホストサービス**は、ローカル IPC 経由で `system.run` を **macOS アプリ**に転送します。
- **macOS アプリ**は承認を適用し、UI コンテキストでコマンドを実行します。

## 有効なポリシーの確認

| コマンド                                                          | 表示内容                                                                          |
| ---------------------------------------------------------------- | -------------------------------------------------------------------------------------- |
| `openclaw approvals get` / `--gateway` / `--node <id\|name\|ip>` | 要求されたポリシー、ホストポリシーのソース、および有効な結果。                       |
| `openclaw exec-policy show`                                      | ローカルマシンでマージされたビュー。                                                             |
| `openclaw exec-policy set` / `preset`                            | ローカルで要求されたポリシーを、ローカルホストの承認ファイルと 1 ステップで同期します。 |

<Note>
セッション単位の `/exec` オーバーライドは含まれません。関連するセッションで `/exec` を実行して、現在のデフォルトを確認してください。[セッションのオーバーライド](/ja-JP/tools/exec#session-overrides-exec)を参照してください。
</Note>

CLI の完全なリファレンス（フラグ、JSON 出力、許可リストへの追加/削除）については、[承認 CLI](/ja-JP/cli/approvals)を参照してください。

ローカルスコープが `host=node` を要求すると、`exec-policy show` は、
ローカルの承認ファイルを信頼できる情報源として扱う代わりに、そのスコープを実行時に Node 管理として報告します。

コンパニオンアプリの UI が **利用できない** 場合、通常であれば確認を求めるリクエストは、
**確認時のフォールバック**（デフォルト: `deny`）によって解決されます。

<Tip>
ネイティブチャットの承認クライアントは、保留中の承認メッセージにチャンネル固有の操作手段を追加できます。Matrix はリアクションのショートカット（`✅` は 1 回のみ許可、
`♾️` は常に許可、`❌` は拒否）を追加しつつ、フォールバックとしてメッセージ内に `/approve ...` も残します。
</Tip>

## 設定と保存場所

承認は、実行ホスト上のローカル JSON ファイルに保存されます。
`OPENCLAW_STATE_DIR` が設定されている場合、ファイルはその状態ディレクトリに配置されます。
それ以外の場合は、OpenClaw のデフォルトの状態ディレクトリが使用されます。

```text
$OPENCLAW_STATE_DIR/exec-approvals.json
# それ以外の場合
~/.openclaw/exec-approvals.json
```

デフォルトの承認ソケットも同じルートに配置されます。
`$OPENCLAW_STATE_DIR/exec-approvals.sock`、または
変数が未設定の場合は `~/.openclaw/exec-approvals.sock` です。

状態ディレクトリは、それぞれ独立した信頼スコープです。`OPENCLAW_STATE_DIR` が
別の場所を指している場合、OpenClaw は
`~/.openclaw/exec-approvals.json` をインポートもアーカイブもしません。カスタム状態ディレクトリ用の承認は個別に設定してください。Doctor も、アクティブな状態ディレクトリに属する場合にのみ、従来の
`plugin-binding-approvals.json` をインポートします。

スキーマの例:

```json
{
  "version": 1,
  "socket": {
    "path": "~/.openclaw/exec-approvals.sock",
    "token": "base64url-token"
  },
  "defaults": {
    "security": "deny",
    "ask": "on-miss",
    "askFallback": "deny",
    "autoAllowSkills": false
  },
  "agents": {
    "main": {
      "security": "allowlist",
      "ask": "on-miss",
      "askFallback": "deny",
      "autoAllowSkills": true,
      "allowlist": [
        {
          "id": "B0C8C0B3-2C2D-4F8A-9A3C-5A4B3C2D1E0F",
          "pattern": "~/Projects/**/bin/rg",
          "argPattern": "sha256:argv:...",
          "source": "allow-always",
          "lastUsedAt": 1737150000000,
          "lastResolvedPath": "/Users/user/Projects/.../bin/rg"
        },
        {
          "pattern": "~/Projects/**/bin/git"
        }
      ]
    }
  }
}
```

## ポリシー設定項目

### `tools.exec.mode`

`tools.exec.mode` は、ホストでの Exec に推奨される正規化済みポリシーサーフェスです。

| 値       | 動作                                                                                                                                                                  |
| ----------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `deny`      | ホストでの Exec をブロックします。                                                                                                                                                          |
| `allowlist` | 許可リストに登録されたコマンドのみ、確認せずに実行します。                                                                                                                             |
| `ask`       | 許可リストポリシーを使用し、一致しない場合に確認します。                                                                                                                                   |
| `auto`      | 許可リストポリシーを使用して決定論的に一致するものを直接実行し、承認が必要な不一致項目を OpenClaw のネイティブ自動レビュアーに送り、人による承認経路へフォールバックします。 |
| `full`      | 承認プロンプトなしでホスト上の Exec を実行します。                                                                                                                                   |

Doctor は、廃止された永続化済みの `tools.exec.security` / `tools.exec.ask`
の組を `tools.exec.mode` に移行します。

### `exec.security`

<ParamField path="security" type='"deny" | "allowlist" | "full"'>
  - `deny` - ホストでのすべての Exec リクエストをブロックします。
  - `allowlist` - 許可リストに登録されたコマンドのみ許可します。
  - `full` - すべてを許可します（昇格と同等）。

Gateway/Node ホストのデフォルトは `full` です。`sandbox` ホストでは、代わりに
`deny` がデフォルトになります。
</ParamField>

### `exec.ask`

<ParamField path="ask" type='"off" | "on-miss" | "always"'>
  ホストでの Exec に設定された確認ポリシーです。`tools.exec.ask` とホスト承認のデフォルトから得られる、
  承認プロンプトの基準動作を制御します。
  デフォルトは `off` です。呼び出し単位の `ask` ツールパラメーター（
  [Exec ツール](/ja-JP/tools/exec#parameters)を参照）は、この基準を厳しくすることしかできません。また、
  チャンネル由来のモデル呼び出しでは、ホストの有効な確認設定が `off` の場合、このパラメーターは無視されます。

- `off` - 確認しません。
- `on-miss` - 許可リストに一致しない場合のみ確認します。
- `always` - すべてのコマンドで確認します。有効な確認モードが `always` の場合、`allow-always` による永続的な信頼があっても、プロンプトは抑止され **ません**。

</ParamField>

### `askFallback`

<ParamField path="askFallback" type='"deny" | "allowlist" | "full"'>
  プロンプトが必要であるものの、到達可能な UI がない場合（またはプロンプトがタイムアウトした場合）の
  解決方法です。省略時のデフォルトは `deny` です。

- `deny` - ブロックします。
- `allowlist` - 許可リストに一致する場合のみ許可します。
- `full` - 許可します。

</ParamField>

### `tools.exec.strictInlineEval`

<ParamField path="strictInlineEval" type="boolean">
  `true` の場合、インタープリターのバイナリ自体が許可リストに登録されていても、
  インラインコード評価形式を承認必須として扱います。安定した 1 つのファイルオペランドに明確に対応しない
  インタープリターのローダーに対する多層防御です。
</ParamField>

厳格モードで検出される例: `python -c`、`node -e`/`--eval`/`-p`、
`ruby -e`、`perl -e`/`-E`、`php -r`、`lua -e`、`osascript -e`（`awk`、
`sed`、`make`、`find -exec`、および `xargs` のインライン形式も含む）。

厳格モードでは、これらのコマンドにレビュアーまたは明示的な承認が必要です。
`tools.exec.mode: "auto"` の場合、コマンドに強制可能なプランがあれば、レビュアーは低リスクの実行を 1 回だけ許可できます。それ以外の場合、OpenClaw は人に承認を求めます。
レビュアーへのフォールバックに到達した `Codex app-server` コマンド承認では、承認リクエストに強制可能な解決済み実行可能ファイルが提示されないため、
人に確認します。
`allow-always` は、インライン評価コマンドの新しい許可リストエントリを永続化しません。

### `tools.exec.commandHighlighting`

<ParamField path="commandHighlighting" type="boolean" default="false">
  表示専用です。有効にすると、OpenClaw はパーサーから導出されたコマンド範囲を添付し、
  Web 承認プロンプトでコマンドトークンを強調表示できるようにします。
  `security`、`ask`、許可リストの照合、厳格なインライン評価の
  動作、承認の転送、またはコマンドの実行は変更 **しません**。
</ParamField>

`tools.exec.commandHighlighting` でグローバルに設定するか、
`agents.entries.*.tools.exec.commandHighlighting` でエージェント単位に設定します。

## YOLO モード（承認なし）

承認プロンプトなしでホスト上の Exec を実行するには、両方のポリシーレイヤーを開放します。
OpenClaw 設定内の要求された Exec ポリシー（`tools.exec.*`）**と**
実行ホストの承認ファイル内のホストローカル承認ポリシーです。

省略された `askFallback` のデフォルトは `deny` です。UI がない場合の承認プロンプトを許可にフォールバックさせるには、
ホストの `askFallback` を明示的に `full` に設定します。

| レイヤー              | YOLO 設定               |
| ------------------ | -------------------------- |
| `tools.exec.mode`  | `gateway`/`node` の `full` |
| ホストの `askFallback` | `full`                     |

<Warning>
**重要な相違点:**

- `tools.exec.host=auto` は、exec を実行する**場所**を選択します。利用可能な場合はサンドボックス、それ以外の場合は Gateway です。
- YOLO は、ホスト exec を承認する**方法**を選択します：`security=full` と `ask=off`。
- YOLO は、設定されたホスト exec ポリシーに加えて、ヒューリスティックなコマンド難読化の承認ゲートやスクリプト事前チェックの拒否レイヤーを別途追加するものでは**ありません**。
- `auto` によって、サンドボックス化されたセッションから Gateway へのルーティングを自由に上書きできるようになるわけではありません。呼び出しごとの `host=node` リクエストは `auto` から許可されます。`host=gateway` は、サンドボックスランタイムがアクティブでない場合に限り、`auto` から許可されます。安定した非自動のデフォルトを設定するには、`tools.exec.host` を設定するか、`/exec host=...` を明示的に使用してください。

</Warning>

独自の非対話型権限モードを公開する CLI ベースのプロバイダーは、
このポリシーに従うことができます。OpenClaw の有効な exec
ポリシーが YOLO の場合、Claude CLI は
`--permission-mode bypassPermissions` を追加します。OpenClaw が管理する Claude ライブセッションでは、OpenClaw の
有効な exec ポリシーが Claude のネイティブ権限モードより優先されます。
生の Claude バックエンド引数で別のモードが指定されていても、
YOLO はライブ起動を `--permission-mode bypassPermissions` に正規化し、
制限的な有効 exec ポリシーはライブ起動を
`--permission-mode default` に正規化します。

より保守的な設定にする場合は、OpenClaw の exec ポリシーを
`allowlist` / `on-miss` または `deny` に戻して制限してください。

### Gateway ホストで永続的に「プロンプトを表示しない」設定

<Steps>
  <Step title="要求する設定ポリシーを設定する">
    ```bash
    openclaw config set tools.exec.host gateway
    openclaw config set tools.exec.mode full
    openclaw gateway restart
    ```
  </Step>
  <Step title="ホストの承認ファイルを一致させる">
    ```bash
    openclaw approvals set --stdin <<'EOF'
    {
      version: 1,
      defaults: {
        security: "full",
        ask: "off",
        askFallback: "full"
      }
    }
    EOF
    ```
  </Step>
</Steps>

### ローカルショートカット

```bash
openclaw exec-policy preset yolo
```

ローカルの `tools.exec.host/security/ask` と、ローカル承認
ファイルのデフォルト（`askFallback: "full"` を含む）の両方を更新します。これは意図的に
ローカル専用です。Gateway ホストまたは Node ホストの承認をリモートで変更するには、
`openclaw approvals set --gateway` または `openclaw approvals set --node
<id|name|ip>` を使用してください。

その他の組み込みプリセット：`cautious`（`host=gateway`、`security=allowlist`、
`ask=on-miss`、`askFallback=deny`）および `deny-all`（`host=gateway`、
`security=deny`、`ask=off`、`askFallback=deny`）。同じ方法で適用します：
`openclaw exec-policy preset cautious`。

完全なプリセットの代わりに個別のフィールドを設定するには、
これらのフラグの任意のサブセットとともに `openclaw exec-policy set --host <auto|sandbox|gateway|node> --security
<deny|allowlist|full> --ask <off|on-miss|always> --ask-fallback
<deny|allowlist|full>` を使用してください。

### Node ホスト

代わりに Node で同じ承認ファイルを適用します：

```bash
openclaw approvals set --node <id|name|ip> --stdin <<'EOF'
{
  version: 1,
  defaults: {
    security: "full",
    ask: "off",
    askFallback: "full"
  }
}
EOF
```

<Note>
**ローカル専用の制限：**

- `openclaw exec-policy` は Node の承認を同期しません。
- `openclaw exec-policy set --host node` は拒否されます。
- Node の exec 承認は実行時に Node から取得されるため、Node を対象とする更新では `openclaw approvals --node ...` を使用する必要があります。

</Note>

### セッション専用ショートカット

- `/exec security=full ask=off` は現在のセッションのみを変更します。
- `/elevated full` は、要求されたポリシーとホスト承認ファイルの両方が
`security: "full"` と `ask: "off"` に解決される場合に限り、exec 承認をスキップする緊急用ショートカットです。`ask:
"always"` など、より厳格なホストファイルでは引き続きプロンプトが表示されます。

ホスト承認ファイルが設定より厳格なままの場合は、より厳格なホスト
ポリシーが引き続き優先されます。

## 許可リスト（エージェントごと）

許可リストは**エージェントごと**です。複数のエージェントが存在する場合は、
macOS アプリで編集するエージェントを切り替えてください。パターンは glob マッチです。

パターンには、解決済みバイナリパスの glob またはコマンド名のみの glob を使用できます。
名前のみのパターンは、`PATH` を通じて呼び出されたコマンドだけに一致します。そのため、コマンドが `rg` の場合、`rg` は
`/opt/homebrew/bin/rg` に一致できますが、`./rg` や
`/tmp/rg` には一致**しません**。特定のバイナリの場所を 1 つだけ信頼するには、パス glob を使用してください。

従来の `agents.default` エントリは、読み込み時に `agents.main` に移行されます。
`echo ok && pwd` のようなシェルチェーンでも、最上位の各セグメントが
許可リストのルールを満たす必要があります。

例：

- `rg`
- `~/Projects/**/bin/peekaboo`
- `~/.local/bin/*`
- `/opt/homebrew/bin/rg`

### argPattern による引数の制限

許可リストのエントリを、バイナリと特定の引数形式に一致させる場合は、
`argPattern` を追加します。OpenClaw はすべてのホストで ECMAScript（JavaScript）の正規
表現セマンティクスを使用し、実行可能ファイルのトークン（`argv[0]`）を除いた、
解析済みのコマンド引数に対して式を評価します。
手動作成したエントリでは、引数は単一のスペースで連結されるため、
完全一致が必要な場合はパターンをアンカーしてください。

```json
{
  "version": 1,
  "agents": {
    "main": {
      "allowlist": [
        {
          "pattern": "python3",
          "argPattern": "^safe\\.py$"
        }
      ]
    }
  }
}
```

このエントリでは `python3 safe.py` が許可され、`python3 other.py` は許可リストに
一致しません。同じバイナリにパスのみのエントリも存在する場合、一致しない
引数でも、そのパスのみのエントリにフォールバックできます。バイナリを宣言した引数のみに
制限することが目的の場合は、パスのみのエントリを省略してください。

承認フローによって保存されたエントリでは、argv の完全一致に内部セパレーター形式が使用されます。
エンコードされた値を手動で編集するのではなく、UI または承認フローでこれらのエントリを
再生成することを推奨します。OpenClaw がコマンドセグメントの argv を解析できない場合、
`argPattern` を含むエントリは一致しません。

生成された `allow-always` エントリは argv に結び付けられます。新しく生成されるエントリには
`argPattern` が含まれます。古い、生成されたパスのみのエントリは無視されるため、新たに
承認する必要があります。手動のパスのみのルールでは、`source` と `argPattern` の両方を省略してください。

各許可リストエントリがサポートする項目：

| フィールド              | 意味                                                                  |
| ------------------ | ------------------------------------------------------------------------ |
| `pattern`          | 解決済みバイナリパスの glob またはコマンド名のみの glob                      |
| `argPattern`       | ECMAScript の argv 正規表現または生成された argv 完全一致ハッシュ。省略時はパスのみ |
| `id`               | 安定した不透明な ID。存在しない場合は UUID として生成                        |
| `source`           | `allow-always` などの生成済みエントリのソース。手動エントリでは省略  |
| `commandText`      | 従来のプレーンテキスト入力。読み込み時に破棄                            |
| `lastUsedAt`       | 最終使用時刻                                                      |
| `lastUsedCommand`  | 最後に一致したコマンド。生成されたハッシュ化 argv エントリでは省略     |
| `lastResolvedPath` | 最後に解決されたバイナリパス                                                |

## Skills CLI の自動許可

**Skills CLI の自動許可**（`autoAllowSkills`）を有効にすると、既知の Skills から
参照される実行可能ファイルは、Node（macOS Node
またはヘッドレス Node ホスト）で許可リストに登録済みとして扱われます。これは Gateway RPC 経由で `skills.bins` を使用して、
Skills のバイナリ一覧を取得します。厳格な手動
許可リストを使用する場合は無効にしてください。

<Warning>
- これは、手動のパス許可リストエントリとは別の、**暗黙的な利便性のための許可リスト**です。
- Gateway と Node が同じ信頼境界内にある、信頼できるオペレーター環境を対象としています。
- 厳格で明示的な信頼が必要な場合は、`autoAllowSkills: false` のままにして、手動のパス許可リストエントリのみを使用してください。

</Warning>

## 安全なバイナリと承認の転送

安全なバイナリ（標準入力のみの高速パス）、インタープリターのバインドに関する詳細、および
承認プロンプトを Slack/Discord/Telegram に転送する方法（またはネイティブ承認クライアントとして実行する方法）については、
[exec 承認 - 詳細](/ja-JP/tools/exec-approvals-advanced)を参照してください。

## Control UI での編集

**Control UI -> Nodes -> Exec approvals** カードを使用して、デフォルト、
エージェントごとの上書き、および許可リストを編集します。スコープ（Defaults またはエージェント）を選択し、
ポリシーを調整して、許可リストパターンを追加または削除してから、**Save** を選択します。UI には
パターンごとの最終使用メタデータが表示されるため、一覧を整理された状態に保てます。

対象セレクターでは、**Gateway**（ローカル承認）または **Node** を選択します。
Node は `system.execApprovals.get/set` を通知する必要があります（macOS アプリまたはヘッドレス
Node ホスト）。Node が exec 承認をまだ通知しない場合は、その Node の
ローカル承認ファイルを直接編集してください。

Windows コンパニオンを含む一部の Node ホストは、異なる承認
ポリシー形式を管理します。Control UI では、これらのホストネイティブなポリシーが読み取り専用で表示されます。
編集するには、コンパニオンアプリまたはネイティブの
ポリシー形式を指定した `openclaw approvals set --node <id|name|ip>` を使用してください。[承認 CLI](/ja-JP/cli/approvals)を参照してください。

CLI：`openclaw approvals` は Gateway または Node の編集をサポートします。
[承認 CLI](/ja-JP/cli/approvals)を参照してください。

## 承認フロー

プロンプトが必要な場合、Gateway はオペレータークライアントに
`exec.approval.requested` をブロードキャストします。Control UI と macOS
アプリは `exec.approval.resolve` を介してそれを解決し、Gateway が承認済みの
リクエストを Node ホストへ転送します。

`host=node` の場合、承認リクエストには正規の `systemRunPlan`
ペイロードが含まれます。Gateway は、承認済みの `system.run` リクエストを転送するときに、
そのプランをコマンド/cwd/セッションコンテキストの信頼できる情報源として使用します：

- Node の exec パスは、最初に 1 つの正規プランを準備します。
- 承認レコードには、そのプランとバインディングメタデータが保存されます。
- 承認後、最終的に転送される `system.run` 呼び出しでは、後続の呼び出し元による編集を信頼せず、保存されたプランを再利用します。
- 承認リクエストの作成後に呼び出し元が `command`、`rawCommand`、`cwd`、`agentId`、または `sessionKey` を変更した場合、Gateway は承認の不一致として転送された実行を拒否します。

## システムイベントと拒否

Node が完了を報告すると、exec のライフサイクルはエージェントの
セッションに `Exec finished` システムメッセージを投稿します。OpenClaw は、承認が付与された後、
`tools.exec.approvalRunningNoticeMs` が経過すると、進行中の通知も発行できます（デフォルトは `10000`、`0` で
無効化）。拒否された exec 承認は、そのホストコマンドにとって終端状態です。コマンドは
実行されません。

- 発生元セッションがあるメインエージェントの非同期承認では、OpenClaw は
  そのセッションに内部フォローアップとして拒否を投稿します。これにより、エージェントは
  非同期コマンドの待機を終了し、結果欠落の修復を回避できます。
- セッションがない場合、またはセッションを再開できない場合でも、OpenClaw は
  オペレーターまたは直接チャットルートに簡潔な拒否を報告できます。
- サブエージェントおよび Cron セッションの拒否は、その
  セッションには投稿されません。

Gateway ホストの exec 承認でも、同じ完了ライフサイクルイベントが発行されます。
承認によって制御される exec は承認 ID を再利用し、保留中の
リクエストと、その完了/拒否メッセージ（`Exec finished (gateway
id=...)` / `Exec denied (gateway id=...)`）を関連付けます。

## 影響

- **`full`** は強力です。可能な場合は許可リストを推奨します。
- **`ask`** は迅速な承認を可能にしながら、ユーザーの関与を維持します。
- エージェントごとの許可リストにより、あるエージェントの承認が他のエージェントへ漏れるのを防ぎます。
- 承認は、**認可された送信者**によるホスト exec リクエストにのみ適用されます。認可されていない送信者は `/exec` を発行できません。
- `/exec security=full` は、認可されたオペレーター向けのセッションレベルの利便機能であり、設計上、承認をスキップします。ホスト exec を完全にブロックするには、承認セキュリティを `deny` に設定するか、ツールポリシーで `exec` ツールを拒否してください。

## 関連項目

<CardGroup cols={2}>
  <Card title="実行承認 - 高度な設定" href="/ja-JP/tools/exec-approvals-advanced" icon="gear">
    安全なバイナリ、インタープリターのバインド、チャットへの承認転送。
  </Card>
  <Card title="実行ツール" href="/ja-JP/tools/exec" icon="terminal">
    シェルコマンド実行ツール。
  </Card>
  <Card title="昇格モード" href="/ja-JP/tools/elevated" icon="shield-exclamation">
    承認もスキップする緊急時用の経路。
  </Card>
  <Card title="サンドボックス化" href="/ja-JP/gateway/sandboxing" icon="box">
    サンドボックスモードとワークスペースへのアクセス。
  </Card>
  <Card title="セキュリティ" href="/ja-JP/gateway/security" icon="lock">
    セキュリティモデルと堅牢化。
  </Card>
  <Card title="サンドボックス、ツールポリシー、昇格モードの比較" href="/ja-JP/gateway/sandbox-vs-tool-policy-vs-elevated" icon="sliders">
    各制御を使用する場面。
  </Card>
  <Card title="Skills" href="/ja-JP/tools/skills" icon="sparkles">
    Skills に基づく自動許可の動作。
  </Card>
</CardGroup>
