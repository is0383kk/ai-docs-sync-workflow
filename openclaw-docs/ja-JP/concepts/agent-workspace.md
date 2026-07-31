---
read_when:
    - エージェントワークスペースまたはそのファイル構成について説明する必要があります
    - エージェントのワークスペースをバックアップまたは移行したい場合
sidebarTitle: Agent workspace
summary: エージェントワークスペース：場所、構成、バックアップ戦略
title: エージェントワークスペース
x-i18n:
    generated_at: "2026-07-26T09:58:10Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: b58ead9079c3dda4bcaec3253f8d55e67e7e554d5c5b87ccfec6b08ec4ba038f
    source_path: concepts/agent-workspace.md
    workflow: 16
---

ワークスペースはエージェントのホームです。ファイルツールとワークスペースコンテキストで使用される作業ディレクトリです。非公開に保ち、メモリとして扱ってください。

これは、設定、認証情報、セッションを保存する `~/.openclaw/` とは別のものです。

<Warning>
ワークスペースは**デフォルトの cwd** であり、厳密なサンドボックスではありません。ツールはワークスペースを基準に相対パスを解決しますが、サンドボックスが有効でない限り、絶対パスを使用するとホスト上の別の場所にもアクセスできます。分離が必要な場合は、[`agents.defaults.sandbox`](/ja-JP/gateway/sandboxing)（および／またはエージェント単位のサンドボックス設定）を使用してください。

サンドボックスが有効で、`workspaceAccess` が `"rw"` でない場合、ツールはホストのワークスペースではなく、`~/.openclaw/sandboxes` 配下のサンドボックスワークスペース内で動作します。
</Warning>

## デフォルトの場所

- デフォルト: `~/.openclaw/workspace`
- `OPENCLAW_PROFILE` が設定され、`"default"` でない場合、デフォルトは `~/.openclaw/workspace-<profile>` になります。
- `OPENCLAW_WORKSPACE_DIR` が設定されている場合、上記の両方を上書きします。
- 明示的なワークスペースがない非デフォルトのエージェント（`agents.entries.*`）は、共有のデフォルトワークスペースではなく、`<state-dir>/workspace-<agentId>` に解決されます。

`~/.openclaw/openclaw.json` で上書きします。

```json5
{
  agents: {
    defaults: {
      workspace: "~/.openclaw/workspace",
    },
  },
}
```

エージェント単位の上書き: `agents.entries.*.workspace`。

`openclaw onboard`、`openclaw configure`、または `openclaw setup` は、ワークスペースを作成し、ブートストラップファイルがない場合は初期ファイルを配置します。

<Note>
サンドボックスへの初期ファイルのコピーでは、ワークスペース内の通常ファイルのみが受け入れられます。ソースワークスペース外に解決されるシンボリックリンク／ハードリンクのエイリアスは無視されます。
</Note>

ワークスペースファイルをすでに自身で管理している場合は、ブートストラップファイルの作成を無効にします。

```json5
{ agents: { defaults: { skipBootstrap: true } } }
```

## 追加のワークスペースフォルダー

古いインストールでは `~/openclaw` が作成されている場合があります。同時にアクティブになるワークスペースは1つだけなので、複数のワークスペースディレクトリを残しておくと、認証や状態に分かりにくいずれが生じることがあります。

<Note>
**推奨:** アクティブなワークスペースは1つだけにしてください。追加のフォルダーを使用しなくなった場合は、アーカイブするかゴミ箱に移動してください（例: `trash ~/openclaw`）。意図的に複数のワークスペースを保持する場合は、`agents.defaults.workspace`（またはエージェント単位の `workspace` キー）がアクティブなワークスペースを指していることを確認してください。
</Note>

## ワークスペースのファイル構成

OpenClaw がワークスペース内にあることを想定する標準ファイル:

<AccordionGroup>
  <Accordion title="AGENTS.md - 運用手順">
    エージェントの運用手順とメモリの使用方法です。各セッションの開始時に読み込まれます。ルール、優先事項、「どのように振る舞うか」の詳細を記載するのに適しています。
  </Accordion>
  <Accordion title="SOUL.md - ペルソナとトーン">
    ペルソナ、トーン、境界です。各セッションで読み込まれます。ガイド: [SOUL.md パーソナリティガイド](/ja-JP/concepts/soul)。
  </Accordion>
  <Accordion title="USER.md - ユーザーについて">
    ユーザーが誰で、どのように呼びかけるかを記載します。各セッションで読み込まれます。
  </Accordion>
  <Accordion title="IDENTITY.md - 名前、雰囲気、絵文字">
    エージェントの名前、雰囲気、絵文字です。ブートストラップ手順の実行中に作成／更新されます。
  </Accordion>
  <Accordion title="TOOLS.md - ローカルツールの規約">
    ローカルツールと規約に関する注記です。ツールの利用可否は制御せず、ガイダンスとしてのみ機能します。
  </Accordion>
  <Accordion title="HEARTBEAT.md - Heartbeat チェックリスト">
    Heartbeat 実行用の任意の小さなチェックリストです。トークン消費を避けるため、短く保ってください。
  </Accordion>
  <Accordion title="BOOT.md - 起動チェックリスト">
    Gateway の再起動時に自動実行される任意の起動チェックリストです（[内部フック](/ja-JP/automation/hooks)が有効な場合）。短く保ち、外部への送信にはメッセージツールを使用してください。
  </Accordion>
  <Accordion title="BOOTSTRAP.md - 初回実行手順">
    1回限りの初回実行手順です。新規ワークスペースに対してのみ作成されます。手順の完了後に削除してください。
  </Accordion>
  <Accordion title="memory/YYYY-MM-DD.md - 日次メモリログ">
    日次メモリログ（1日につき1ファイル）です。セッション開始時に今日と昨日の分を読むことを推奨します。
  </Accordion>
  <Accordion title="MEMORY.md - 整理された長期メモリ（任意）">
    整理された長期メモリです。長期間保持する事実、設定、決定事項、短い要約を記録します。詳細なログは `memory/YYYY-MM-DD.md` に保存し、すべてのプロンプトに挿入せず、必要に応じてメモリツールから取得できるようにしてください。`MEMORY.md` はメインの非公開セッションでのみ読み込み、共有／グループコンテキストでは読み込まないでください。ワークフローとメモリの自動フラッシュについては、[メモリ](/ja-JP/concepts/memory)を参照してください。
  </Accordion>
  <Accordion title="skills/ - ワークスペースの Skills（任意）">
    ワークスペース固有の Skills です。名前が競合する場合、そのワークスペースでは、プロジェクトのエージェント Skills、個人のエージェント Skills、管理対象の Skills、同梱の Skills、`skills.load.extraDirs` よりも優先される Skills の保存場所です。
  </Accordion>
  <Accordion title="canvas/ - Canvas UI ファイル（任意）">
    Node 表示用の Canvas UI ファイルです（例: `canvas/index.html`）。
  </Accordion>
</AccordionGroup>

<Note>
ブートストラップファイルがない場合、OpenClaw はセッションに「ファイル不足」マーカーを挿入して処理を続行します。大きなブートストラップファイルは挿入時に切り詰められます。上限は `agents.defaults.bootstrapMaxChars`（デフォルト: `20000`）および `agents.defaults.bootstrapTotalMaxChars`（デフォルト: `60000`）で調整できます。`openclaw setup` を使用すると、既存ファイルを上書きせずに不足しているデフォルトファイルを再作成できます。
</Note>

## ワークスペースに含まれないもの

以下は `~/.openclaw/` 配下にあり、ワークスペースのリポジトリにコミットしてはなりません。

- `~/.openclaw/openclaw.json`（設定）
- `~/.openclaw/state/openclaw.sqlite`（共有ワークスペースのセットアップ状態と証明）
- `~/.openclaw/agents/<agentId>/agent/auth-profiles.json`（モデル認証プロファイル: OAuth + API キー）
- `~/.openclaw/agents/<agentId>/agent/openclaw-agent.sqlite`（セッション行、トランスクリプト、エージェント単位のランタイム状態）
- `~/.openclaw/agents/<agentId>/agent/codex-home/`（エージェント単位の Codex ランタイムアカウント、設定、Skills、plugins、ネイティブスレッド状態）
- `~/.openclaw/credentials/`（チャンネル／プロバイダーの状態と旧 OAuth インポートデータ）
- `~/.openclaw/agents/<agentId>/sessions/`（旧移行元とアーカイブ／サポート成果物）
- `~/.openclaw/skills/`（管理対象の Skills）

セッションまたは設定を移行する必要がある場合は、それらを個別にコピーし、バージョン管理の対象外にしてください。

古い OpenClaw リリースでは、ワークスペースのサイドカーファイルとして `openclaw-workspace-state.json`、
`.openclaw/workspace-state.json`、および `.attested` が書き込まれていました。現在の
ランタイムでは、その状態に共有 SQLite データベースのみを使用します。Doctor が
これらのファイルのいずれかを報告した場合は、`openclaw doctor --fix` を実行してください。Doctor は有効な旧
状態をインポートし、データベースの行を検証した後にのみ移行元を削除します。

## Git バックアップ（推奨、非公開）

ワークスペースを非公開のメモリとして扱ってください。バックアップと復元ができるように、**非公開**の git リポジトリに保存します。

以下の手順は Gateway が動作しているマシン（ワークスペースが存在する場所）で実行してください。

<Steps>
  <Step title="リポジトリを初期化する">
    git がインストールされている場合、新規ワークスペースは自動的に初期化されます。このワークスペースがまだリポジトリでない場合は、次を実行します。

    ```bash
    cd ~/.openclaw/workspace
    git init
    git add AGENTS.md SOUL.md TOOLS.md IDENTITY.md USER.md HEARTBEAT.md memory/
    git commit -m "Add agent workspace"
    ```

  </Step>
  <Step title="非公開リモートを追加する">
    <Tabs>
      <Tab title="GitHub Web UI">
        1. GitHub で新しい**非公開**リポジトリを作成します。
        2. README で初期化しないでください（マージ競合を回避するため）。
        3. HTTPS リモート URL をコピーします。
        4. リモートを追加してプッシュします。

        ```bash
        git branch -M main
        git remote add origin <https-url>
        git push -u origin main
        ```
      </Tab>
      <Tab title="GitHub CLI (gh)">
        ```bash
        gh auth login
        gh repo create openclaw-workspace --private --source . --remote origin --push
        ```
      </Tab>
      <Tab title="GitLab Web UI">
        1. GitLab で新しい**非公開**リポジトリを作成します。
        2. README で初期化しないでください（マージ競合を回避するため）。
        3. HTTPS リモート URL をコピーします。
        4. リモートを追加してプッシュします。

        ```bash
        git branch -M main
        git remote add origin <https-url>
        git push -u origin main
        ```
      </Tab>
    </Tabs>

  </Step>
  <Step title="継続的な更新">
    ```bash
    git status
    git add .
    git commit -m "Update memory"
    git push
    ```
  </Step>
</Steps>

## シークレットをコミットしない

<Warning>
非公開リポジトリであっても、ワークスペースにシークレットを保存しないでください。

- API キー、OAuth トークン、パスワード、または非公開の認証情報。
- `~/.openclaw/` 配下にあるもの。
- チャットの未加工ダンプや機密性の高い添付ファイル。

機密情報への参照を保存する必要がある場合は、プレースホルダーを使用し、実際のシークレットは別の場所（パスワードマネージャー、環境変数、または `~/.openclaw/`）に保管してください。
</Warning>

推奨される `.gitignore` の初期設定:

```gitignore
.DS_Store
.env
**/*.key
**/*.pem
**/secrets*
```

## ワークスペースを新しいマシンに移動する

<Steps>
  <Step title="リポジトリをクローンする">
    リポジトリを目的のパス（デフォルトは `~/.openclaw/workspace`）にクローンします。
  </Step>
  <Step title="設定を更新する">
    `~/.openclaw/openclaw.json` で `agents.defaults.workspace` をそのパスに設定します。
  </Step>
  <Step title="不足しているファイルを配置する">
    `openclaw setup --workspace <path>` を実行して、不足しているファイルを配置します。
  </Step>
  <Step title="セッションをコピーする（任意）">
    セッションが必要な場合は、古いマシンから `~/.openclaw/agents/<agentId>/agent/openclaw-agent.sqlite`
    を個別にコピーします。旧移行入力またはアーカイブ／サポート成果物も必要な場合にのみ、`~/.openclaw/agents/<agentId>/sessions/`
    をコピーしてください。
  </Step>
</Steps>

## 高度な注記

- マルチエージェントルーティングでは、`agents.entries.*.workspace` を使用してエージェントごとに異なるワークスペースを使用できます。ルーティング設定については、[チャンネルルーティング](/ja-JP/channels/channel-routing)を参照してください。
- `agents.defaults.sandbox` が有効な場合、メイン以外のセッションは `agents.defaults.sandbox.workspaceRoot` 配下にあるセッション単位のサンドボックスワークスペースを使用できます。

## 関連項目

- [Heartbeat](/ja-JP/gateway/heartbeat) - ワークスペースファイル HEARTBEAT.md
- [サンドボックス化](/ja-JP/gateway/sandboxing) - サンドボックス環境でのワークスペースへのアクセス
- [セッション](/ja-JP/concepts/session) - セッションの保存パス
- [常設指示](/ja-JP/automation/standing-orders) - ワークスペースファイル内の永続的な指示
