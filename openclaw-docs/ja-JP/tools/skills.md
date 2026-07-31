---
read_when:
    - Skills の追加または変更
    - スキルのゲーティング、許可リスト、読み込みルールの変更
    - Skills の優先順位とスナップショットの動作を理解する
sidebarTitle: Skills
summary: Skills は、ツールの使い方をエージェントに教えます。読み込みの仕組み、優先順位の適用方法、ゲーティング、許可リスト、環境変数の注入を設定する方法について説明します。
title: Skills
x-i18n:
    generated_at: "2026-07-26T09:49:06Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 6925add85652023e3dd2f51f607412fd0bf00581923f76ab2aafd2ca5b8d72be
    source_path: tools/skills.md
    workflow: 16
---

Skills は、ツールをいつどのように使用するかをエージェントに教える Markdown 形式の指示ファイルです。各 Skills は、YAML フロントマターと Markdown 本文を持つ `SKILL.md` ファイルを含むディレクトリに配置されます。OpenClaw はバンドルされた Skills とローカルのオーバーライドを読み込み、環境、設定、バイナリの有無に基づいて読み込み時にフィルタリングします。

<CardGroup cols={2}>
  <Card title="Skills の作成" href="/ja-JP/tools/creating-skills" icon="hammer">
    カスタム Skills をゼロから作成してテストします。
  </Card>
  <Card title="Skills ワークショップ" href="/ja-JP/tools/skill-workshop" icon="flask">
    エージェントが作成した Skills の提案をレビューして承認します。
  </Card>
  <Card title="Skills の設定" href="/ja-JP/tools/skills-config" icon="gear">
    `skills.*` の完全な設定スキーマとエージェントの許可リストです。
  </Card>
  <Card title="ClawHub" href="/clawhub" icon="cloud">
    コミュニティの Skills を参照してインストールします。
  </Card>
</CardGroup>

## 読み込み順序

OpenClaw は次のソースから、**優先順位が高い順**に読み込みます。同じ Skills 名が複数の場所に存在する場合、最も優先順位の高いソースが使用されます。

| 優先順位    | ソース                 | パス                                    |
| ----------- | ---------------------- | --------------------------------------- |
| 1 — 最高 | ワークスペースの Skills       | `<workspace>/skills`                    |
| 2           | プロジェクトエージェントの Skills   | `<workspace>/.agents/skills`            |
| 3           | 個人エージェントの Skills  | `~/.agents/skills`                      |
| 4           | 管理対象 / ローカルの Skills | `~/.openclaw/skills`                    |
| 5           | バンドルされた Skills         | インストールに同梱                |
| 6 — 最低  | 追加ディレクトリ      | `skills.load.extraDirs` + Plugin の Skills |

Skills のルートでは、グループ化されたレイアウトを使用できます。設定されたルート配下の任意の場所（最大 6 階層）に `SKILL.md` がある場合、OpenClaw はその Skills を検出します。

```text
<workspace>/skills/research/SKILL.md          ✓ 「research」として検出
<workspace>/skills/personal/research/SKILL.md ✓ こちらも「research」として検出
```

フォルダパスは整理のためだけに使用されます。Skills の名前とスラッシュコマンドは、`name` フロントマターフィールドから取得されます（`name` がない場合はディレクトリ名）。エージェントの許可リスト（後述）も、この `name` に対して照合されます。

<Note>
  Codex CLI ネイティブの `$CODEX_HOME/skills` ディレクトリは、OpenClaw の Skills ルートでは**ありません**。これらの Skills を一覧化するには `openclaw migrate plan codex` を使用し、OpenClaw ワークスペースへコピーするには `openclaw migrate codex` を使用してください。
</Note>

## Node でホストされる Skills

接続されたヘッドレス Node は、アクティブな OpenClaw Skills ディレクトリ（デフォルトでは `~/.openclaw/skills`。プロファイル環境のオーバーライドが適用されます）にインストールされた Skills を公開できます。Node が接続されている間は通常のエージェント Skills リストに表示され、切断されると表示されなくなります。名前が競合した場合、ローカルまたは Gateway の Skills はその名前を維持し、Node の Skills には決定的な Node プレフィックス付きの名前が割り当てられます。Node ホスト型 v1 では、ディレクトリ名が Skills の `name` フロントマターフィールドと一致している必要があります。

Skills エントリには Node ロケーターが含まれます。そのファイル、相対参照、バイナリは Node 上に存在するため、`exec host=node node=<node-id>` を使用して読み込み、実行してください。Skills ファイルを変更した後は、Node ホストを再起動してください。ペアリングと無効化スイッチについては、[Node](/ja-JP/nodes#node-hosted-skills)を参照してください。

## エージェント単位と共有 Skills

マルチエージェント構成では、各エージェントが独自のワークスペースを持ちます。必要な可視性に対応するパスを使用してください。

| スコープ          | パス                         | 表示対象                  |
| -------------- | ---------------------------- | --------------------------- |
| エージェント単位      | `<workspace>/skills`         | そのエージェントのみ             |
| プロジェクトエージェント  | `<workspace>/.agents/skills` | そのワークスペースのエージェントのみ |
| 個人エージェント | `~/.agents/skills`           | このマシン上のすべてのエージェント  |
| 共有管理対象 | `~/.openclaw/skills`         | このマシン上のすべてのエージェント  |
| 追加ディレクトリ     | `skills.load.extraDirs`      | このマシン上のすべてのエージェント  |

## エージェントの許可リスト

Skills の**場所**（優先順位）と Skills の**可視性**（どのエージェントが使用できるか）は、別々に制御されます。許可リストを使用すると、読み込み元に関係なく、エージェントに表示される Skills を制限できます。

```json5
{
  agents: {
    defaults: {
      skills: ["github", "weather"], // 共有ベースライン
    },
    list: [
      { id: "writer" }, // github、weather を継承
      { id: "docs", skills: ["docs-search"] }, // デフォルトを完全に置換
      { id: "locked-down", skills: [] }, // Skills なし
    ],
  },
}
```

<AccordionGroup>
  <Accordion title="許可リストのルール">
    - デフォルトですべての Skills を制限なしにするには、`agents.defaults.skills` を省略します。
    - `agents.defaults.skills` を継承するには、`agents.entries.*.skills` を省略します。
    - そのエージェントに Skills を一切公開しない場合は、`agents.entries.*.skills: []` を設定します。
    - 空でない `agents.entries.*.skills` リストが**最終的な**セットになります。デフォルトとはマージされません。
    - 有効な許可リストは、プロンプトの構築、スラッシュコマンドの検出、サンドボックスの同期、Skills のスナップショット全体に適用されます。
    - これはホストシェルの認可境界ではありません。同じエージェントが `exec` を使用できる場合は、サンドボックス化、OS ユーザー分離、exec の拒否リスト / 許可リスト、リソースごとの認証情報を使用して、そのシェルを別途制限してください。

  </Accordion>
</AccordionGroup>

## Plugin と Skills

Plugin は、`openclaw.plugin.json` に `skills` ディレクトリ（Plugin ルートからの相対パス）を列挙することで、独自の Skills を同梱できます。Plugin の Skills は、その Plugin が有効な場合に読み込まれます。たとえば、ブラウザー Plugin には、複数ステップのブラウザー制御用の `browser-automation` Skills が同梱されています。

Plugin の Skills ディレクトリは `skills.load.extraDirs` と同じ低優先順位レベルでマージされるため、同じ名前のバンドル済み、管理対象、エージェント、またはワークスペースの Skills があれば、それらが優先されます。Plugin の Skills 自体の適格性は、他の Skills と同様に、フロントマター内の `metadata.openclaw.requires` で制御します。

Plugin システム全体については、[Plugin](/ja-JP/tools/plugin)と[ツール](/ja-JP/tools)を参照してください。

## Skills ワークショップ

[Skills ワークショップ](/ja-JP/tools/skill-workshop)は、エージェントとアクティブな Skills ファイルの間にある提案キューです。エージェントが再利用可能な作業を検出すると、`SKILL.md` に直接書き込む代わりに提案の下書きを作成します。変更が行われる前に、内容をレビューして承認します。

```bash
openclaw skills workshop list
openclaw skills workshop inspect <proposal-id>
openclaw skills workshop apply <proposal-id>
```

完全なライフサイクル、CLI リファレンス、設定については、[Skills ワークショップ](/ja-JP/tools/skill-workshop)を参照してください。

## ClawHub からのインストール

[ClawHub](https://clawhub.ai)は公開 Skills レジストリです。インストールと更新には `openclaw skills` コマンドを使用し、公開と同期には `clawhub` CLI を使用します。

| 操作                             | コマンド                                                |
| ---------------------------------- | ------------------------------------------------------ |
| Skills をワークスペースにインストール | `openclaw skills install @owner/<slug>`                |
| Git リポジトリからインストール      | `openclaw skills install git:owner/repo@ref`           |
| ローカルの Skills ディレクトリをインストール    | `openclaw skills install ./path/to/skill --as my-tool` |
| すべてのローカルエージェント向けにインストール       | `openclaw skills install @owner/<slug> --global`       |
| ワークスペースのすべての Skills を更新        | `openclaw skills update --all`                         |
| 共有管理対象の Skills を更新      | `openclaw skills update @owner/<slug> --global`        |
| 共有管理対象のすべての Skills を更新   | `openclaw skills update --all --global`                |
| Skills の信頼エンベロープを検証    | `openclaw skills verify @owner/<slug>`                 |
| 生成された Skills カードを出力     | `openclaw skills verify @owner/<slug> --card`          |
| ClawHub CLI で公開 / 同期     | `clawhub sync --all`                                   |

<AccordionGroup>
  <Accordion title="インストールの詳細">
    `openclaw skills install` は、デフォルトでアクティブなワークスペースの `skills/` ディレクトリにインストールします。`--global` を追加すると共有の `~/.openclaw/skills` ディレクトリにインストールされ、エージェントの許可リストで制限されていない限り、すべてのローカルエージェントから表示できます。

    Git およびローカルインストールでは、ソースルートに `SKILL.md` が必要です。スラッグは、有効な場合は `SKILL.md` フロントマターの `name` から取得され、それ以外の場合はディレクトリ名またはリポジトリ名が使用されます。上書きするには `--as <slug>` を使用します。`openclaw skills update` が追跡するのは ClawHub からのインストールだけです。Git またはローカルソースを更新するには再インストールしてください。

  </Accordion>
  <Accordion title="検証とセキュリティスキャン">
    `openclaw skills verify @owner/<slug>` は、Skills の `clawhub.skill.verify.v1` 信頼エンベロープを ClawHub に問い合わせます。インストール済みの ClawHub Skills は、`.clawhub/origin.json` に記録されたバージョンとレジストリに対して検証されます。既存のインストール済み Skills または一意に特定できる Skills では、所有者なしのスラッグも引き続き受け付けられますが、所有者修飾付きの参照を使うことで公開者の曖昧さを回避できます。

    ClawHub の Skills ページには、インストール前に最新のセキュリティスキャン状態が表示され、VirusTotal、ClawScan、静的解析の詳細ページも用意されています。ClawHub が検証失敗と判定した場合、コマンドは 0 以外で終了します。公開者は、ClawHub ダッシュボードまたは `clawhub skill rescan @owner/<slug>` を通じて誤検知に対処できます。

  </Accordion>
  <Accordion title="プライベートアーカイブのインストール">
    ClawHub 以外の配布を必要とする Gateway クライアントは、`skills.upload.begin`、`skills.upload.chunk`、`skills.upload.commit` を使用して ZIP 形式の Skills アーカイブをステージングし、`skills.install({ source: "upload", ... })` でインストールできます。この経路はデフォルトで無効になっており、`openclaw.json` に `skills.install.allowUploadedArchives: true` が必要です。通常の ClawHub インストールでは、この設定は不要です。
  </Accordion>
</AccordionGroup>

## セキュリティ

<Warning>
  サードパーティ製の Skills は**信頼できないコード**として扱ってください。有効にする前に内容を確認してください。信頼できない入力やリスクの高いツールでは、サンドボックス内での実行を推奨します。エージェント側の制御については、[サンドボックス化](/ja-JP/gateway/sandboxing)を参照してください。
</Warning>

<AccordionGroup>
  <Accordion title="パスの封じ込め">
    ワークスペース、プロジェクトエージェント、追加ディレクトリでの Skills 検出では、`skills.load.allowSymlinkTargets` が対象ルートを明示的に信頼している場合を除き、解決後の realpath が設定されたルート内に収まる Skills ルートのみを受け付けます。`skills.workshop.allowSymlinkTargetWrites` が有効な場合に限り、Skills ワークショップはこれらの信頼された対象を通じて書き込みを行います。管理対象の `~/.openclaw/skills` と個人用の `~/.agents/skills` にはシンボリックリンクされた Skills フォルダを含められますが、すべての `SKILL.md` の realpath は、解決後の Skills ディレクトリ内に収まっている必要があります。
  </Accordion>
  <Accordion title="オペレーターのインストールポリシー">
    Skills のインストールを続行する前に信頼済みのローカルポリシーコマンドを実行するには、`security.installPolicy` を設定します。このポリシーはメタデータとステージング済みのソースパスを受け取り、ClawHub、アップロード、Git、ローカル、更新、依存関係インストーラーの各経路に適用されます。コマンドが有効な判定を返せない場合は、拒否側に倒れます。
  </Accordion>
  <Accordion title="シークレットの注入範囲">
    `skills.entries.*.env` と `skills.entries.*.apiKey` は、そのエージェントターンの間だけシークレットを**ホスト**プロセスに注入します。サンドボックスには注入されません。シークレットをプロンプトやログに含めないでください。
  </Accordion>
</AccordionGroup>

より広範な脅威モデルとセキュリティチェックリストについては、[セキュリティ](/ja-JP/gateway/security)を参照してください。

## SKILL.md の形式

すべての Skills では、フロントマターに少なくとも `name` と `description` が必要です。

```markdown
---
name: image-lab
description: プロバイダー対応の画像ワークフローを通じて画像を生成または編集する
---

ユーザーが画像の生成を依頼した場合は、`image_generate` ツールを使用します...
```

<Note>
  OpenClaw は [AgentSkills](https://agentskills.io) 仕様に従います。フロントマターは最初に YAML として解析され、失敗した場合は単一行専用のパーサーにフォールバックします。ネストされた `metadata` ブロック（複数行の YAML マッピングを含む）は JSON 文字列に平坦化され、JSON5 として再解析されるため、[ゲーティング](#gating)に示すブロック形式を使用できます。Skills フォルダのパスを参照するには、本文で `{baseDir}` を使用してください。
</Note>

### 任意のフロントマターキー

<ParamField path="homepage" type="string">
  macOS の Skills UI で「Website」として表示される URL です。`metadata.openclaw.homepage` でもサポートされています。
</ParamField>

<ParamField path="user-invocable" type="boolean" default="true">
  `true` の場合、スキルはユーザーが呼び出せるスラッシュコマンドとして公開されます。
</ParamField>

<ParamField path="disable-model-invocation" type="boolean" default="false">
  `true` の場合、OpenClaw はスキルの指示をエージェントの通常の
  プロンプトに含めません。`user-invocable` も `true`
  の場合、スキルは引き続きスラッシュコマンドとして使用できます。
</ParamField>

<ParamField path="command-dispatch" type='"tool"'>
  `tool` に設定すると、スラッシュコマンドはモデルを経由せず、
  登録済みツールに直接ディスパッチされます。
</ParamField>

<ParamField path="command-tool" type="string">
  `command-dispatch: tool` が設定されている場合に呼び出すツール名。
</ParamField>

<ParamField path="command-arg-mode" type='"raw"' default="raw">
  ツールへのディスパッチでは、コアによる解析を行わず、生の引数文字列を
  ツールへ転送します。ツールは
  `{ command: "<raw args>", commandName: "<slash command>", skillName: "<skill name>" }` を受け取ります。
</ParamField>

## ゲーティング

OpenClaw はロード時に `metadata.openclaw`（フロントマターに埋め込まれた
JSON5 オブジェクト。上記の解析に関する注記を参照）を使用してスキルをフィルタリングします。
`metadata.openclaw` ブロックがないスキルは、明示的に無効化されていない限り常に対象となります。

```markdown
---
name: image-lab
description: プロバイダーを利用した画像ワークフローで画像を生成または編集
metadata:
  {
    "openclaw":
      {
        "requires": { "bins": ["uv"], "env": ["GEMINI_API_KEY"], "config": ["browser.enabled"] },
        "primaryEnv": "GEMINI_API_KEY",
      },
  }
---
```

<ParamField path="always" type="boolean">
  `true` の場合、スキルを常に含め、他のすべてのゲートをスキップします。
</ParamField>

<ParamField path="emoji" type="string">
  macOS の Skills UI に表示される省略可能な絵文字。
</ParamField>

<ParamField path="homepage" type="string">
  macOS の Skills UI で「Website」として表示される省略可能な URL。
</ParamField>

<ParamField path="os" type='("darwin" | "linux" | "win32")[]'>
  プラットフォームフィルター。設定した場合、一覧にある OS でのみスキルが対象となります。
</ParamField>

<ParamField path="requires.bins" type="string[]">
  各バイナリが `PATH` に存在する必要があります。
</ParamField>

<ParamField path="requires.anyBins" type="string[]">
  少なくとも 1 つのバイナリが `PATH` に存在する必要があります。
</ParamField>

<ParamField path="requires.env" type="string[]">
  各環境変数がプロセスに存在するか、設定を通じて提供されている必要があります。
</ParamField>

<ParamField path="requires.config" type="string[]">
  各 `openclaw.json` パスが truthy である必要があります。
</ParamField>

<ParamField path="primaryEnv" type="string">
  `skills.entries.<name>.apiKey` に関連付けられた環境変数名。
</ParamField>

<ParamField path="install" type="object[]">
  macOS の Skills UI で使用される省略可能なインストーラー仕様（brew / node / go / uv / download）。
</ParamField>

<Note>
  `metadata.openclaw` がない場合、従来の `metadata.clawdbot` ブロックも引き続き
  受け付けられるため、以前にインストールされたスキルでも依存関係ゲートと
  インストーラーのヒントが維持されます。新しいスキルでは
  `metadata.openclaw` を使用してください。
</Note>

### インストーラー仕様

インストーラー仕様は、依存関係のインストール方法を macOS の Skills UI に指示します。

```markdown
---
name: gemini
description: コーディング支援と Google 検索に Gemini CLI を使用します。
metadata:
  {
    "openclaw":
      {
        "emoji": "♊️",
        "requires": { "bins": ["gemini"] },
        "install":
          [
            {
              "id": "brew",
              "kind": "brew",
              "formula": "gemini-cli",
              "bins": ["gemini"],
              "label": "Gemini CLI をインストール（brew）",
            },
          ],
      },
  }
---
```

<AccordionGroup>
  <Accordion title="インストーラーの選択ルール">
    - 複数のインストーラーが一覧にある場合、Gateway は優先する
      オプションを 1 つ選択します（利用可能なら brew、それ以外は node）。
    - すべてのインストーラーが `download` の場合、OpenClaw は各エントリを一覧表示し、
      利用可能なすべてのアーティファクトを確認できるようにします。
    - 仕様には、プラットフォームでフィルタリングするための `os: ["darwin"|"linux"|"win32"]` を含めることができます。
    - Node によるインストールでは、`openclaw.json` の `skills.install.nodeManager` が使用されます
      （デフォルト: npm、選択肢: npm / pnpm / yarn / bun）。これはスキルの
      インストールにのみ影響し、Gateway ランタイムには引き続き Node を使用する必要があります。
    - Gateway のインストーラー優先順位: Homebrew → uv → 設定済みの node マネージャー →
      go → download。
  </Accordion>
  <Accordion title="インストーラーごとの詳細">
    - **Homebrew:** OpenClaw は Homebrew を自動インストールせず、brew の
      formula をシステムパッケージのコマンドに変換することもありません。
      `brew` がない Linux コンテナでは、brew 専用インストーラーは非表示になります。カスタムイメージを使用するか、
      依存関係を手動でインストールしてください。
    - **Go:** OpenClaw でスキルを自動インストールするには Go 1.21 以降が必要です。
      `go` がなく Homebrew が利用可能な場合、OpenClaw はまず Homebrew から Go をインストールします。
      Homebrew のない Linux では、更新後の `golang-go`
      候補が最低バージョンを満たしていれば、代わりに root として、またはパスワード不要の `sudo` を通じて
      `apt-get` を使用できます。依存関係に対する実際の `go install` は、
      設定済みの `GOBIN` ではなく、常に OpenClaw が管理する専用の bin ディレクトリ
      （新規インストールでは Homebrew の `bin`、それ以外では `~/.local/bin`）を対象とします。
      独自の `GOBIN`、`GOPATH`、`GOTOOLCHAIN`
      環境変数は読み取られますが、上書きされることはありません。
    - **ダウンロード:** `url`（必須）、`archive`（`tar.gz` | `tar.bz2` | `zip`）、
      `extract`（デフォルト: アーカイブ検出時は auto）、`stripComponents`、
      `targetDir`（デフォルト: `~/.openclaw/tools/<skillKey>`）。
  </Accordion>
  <Accordion title="サンドボックス化に関する注記">
    `requires.bins` はスキルのロード時に**ホスト**上で確認されます。エージェントが
    サンドボックスで実行される場合、バイナリは**コンテナ内**にも存在する必要があります。
    `agents.defaults.sandbox.docker.setupCommand` またはカスタム
    イメージを使用してインストールしてください。`setupCommand` はコンテナ作成後に 1 回実行され、
    ネットワークへの送信、書き込み可能なルート FS、およびサンドボックス内の root ユーザーが必要です。
  </Accordion>
</AccordionGroup>

## 設定のオーバーライド

`~/.openclaw/openclaw.json` の `skills.entries` で、バンドルまたは管理対象のスキルを
切り替えて設定します。

```json5
{
  skills: {
    entries: {
      "image-lab": {
        enabled: true,
        apiKey: { source: "env", provider: "default", id: "GEMINI_API_KEY" },
        env: { GEMINI_API_KEY: "GEMINI_KEY_HERE" },
        config: {
          endpoint: "https://example.invalid",
          model: "nano-pro",
        },
      },
      peekaboo: { enabled: true },
      sag: { enabled: false },
    },
  },
}
```

<ParamField path="enabled" type="boolean">
  `false` を指定すると、バンドル済みまたはインストール済みでもスキルが無効になります。バンドルされた
  `coding-agent` スキルはオプトインです。`skills.entries.coding-agent.enabled: true` を設定し、
  `claude`、`codex`、`opencode`、またはその他の対応 CLI のいずれかが
  インストールされ、認証済みであることを確認してください。
</ParamField>

<ParamField path="apiKey" type='string | { source, provider, id }'>
  `metadata.openclaw.primaryEnv` を宣言するスキル向けの便利なフィールドです。
  プレーンテキスト文字列または SecretRef オブジェクトに対応します。
</ParamField>

<ParamField path="env" type="Record<string, string>">
  エージェント実行用に注入される環境変数。変数がプロセスにまだ設定されていない場合にのみ
  注入されます。
</ParamField>

<ParamField path="config" type="object">
  スキルごとのカスタム設定フィールドを格納する省略可能なオブジェクト。
</ParamField>

<ParamField path="allowBundled" type="string[]">
  **バンドル済み**スキル専用の省略可能な許可リスト。設定すると、リスト内のバンドル済みスキルのみが
  対象となります。管理対象スキルとワークスペーススキルには影響しません。
</ParamField>

<Note>
  デフォルトでは、設定キーは**スキル名**と一致します。スキルで
  `metadata.openclaw.skillKey` が定義されている場合は、代わりに `skills.entries` 配下でそのキーを使用してください。
  ハイフンを含む名前は引用符で囲んでください。JSON5 では引用符付きキーを使用できます。
</Note>

## 環境変数の注入

エージェント実行の開始時に、OpenClaw は次の処理を行います。

<Steps>
  <Step title="スキルのメタデータを読み取る">
    OpenClaw は、ゲーティングルール、許可リスト、設定のオーバーライドを適用し、
    エージェントに対する有効なスキル一覧を解決します。
  </Step>
  <Step title="環境変数と API キーを注入する">
    実行中は、`skills.entries.<key>.env` と `skills.entries.<key>.apiKey` が
    `process.env` に適用されます。
  </Step>
  <Step title="システムプロンプトを構築する">
    対象となるスキルはコンパクトな XML ブロックにコンパイルされ、
    システムプロンプトに注入されます。
  </Step>
  <Step title="環境を復元する">
    実行終了後、元の環境が復元されます。
  </Step>
</Steps>

<Warning>
  環境変数の注入範囲は**ホスト**上のエージェント実行であり、サンドボックスではありません。
  サンドボックス内では、`env` と `apiKey` は効果がありません。
  サンドボックス化された実行にシークレットを渡す方法については、
  [Skills の設定](/ja-JP/tools/skills-config#sandboxed-skills-and-env-vars)を参照してください。
</Warning>

バンドルされた `claude-cli` バックエンドの場合、OpenClaw は同じ対象スキルの
スナップショットを一時的な Claude Code Plugin として実体化し、
`--plugin-dir` を通じて渡します。他の CLI バックエンドはプロンプトカタログのみを使用します。

## スナップショットと更新

OpenClaw は**セッション開始時**に対象スキルのスナップショットを作成し、
セッション内の以降のすべてのターンでその一覧を再利用します。スキルまたは設定への変更は、
次の新しいセッションで有効になります。

セッション途中でスキルが更新されるのは、次の 2 つの場合です。

- スキルウォッチャーが `SKILL.md` の変更を検出した場合。
- 新しい対象リモートノードが接続した場合。

更新された一覧は、次のエージェントターンで使用されます。有効なエージェントの
許可リストが変更された場合、OpenClaw は表示されるスキルとの整合性を保つために
スナップショットを更新します。

<AccordionGroup>
  <Accordion title="Skills ウォッチャー">
    デフォルトでは、OpenClaw はスキルフォルダーを監視し、
    `SKILL.md` ファイルが変更されるとスナップショットを更新します。`skills.load` で設定します。

    ```json5
    {
      skills: {
        load: {
          extraDirs: ["~/Projects/agent-scripts/skills"],
          allowSymlinkTargets: ["~/Projects/manager/skills"],
          watch: true, // デフォルト
        },
      },
    }
    ```

    ウォッチャーイベントでは、組み込みの 250 ms デバウンスが使用されます。スキルの
    ルートシンボリックリンクが設定済みルートの外部を指す、意図的なシンボリックリンク構成では、
    `allowSymlinkTargets` を使用してください（例:
    `<workspace>/skills/manager -> ~/Projects/manager/skills`）。
    Skill Workshop が信頼済みのシンボリックリンクパスを通じても
    提案を適用する必要がある場合にのみ、`skills.workshop.allowSymlinkTargetWrites` を有効にしてください。

  </Accordion>
  <Accordion title="リモート macOS ノード（Linux Gateway）">
    Gateway が Linux で実行されていても、`system.run` が許可された
    **macOS ノード**が接続されている場合、必要なバイナリがそのノードに存在すれば、
    OpenClaw は macOS 専用スキルを対象として扱うことができます。エージェントは、
    `host=node` を指定した `exec` ツールを介してこれらのスキルを実行する必要があります。

    オフラインのノードによって、リモート専用スキルが表示されることは**ありません**。ノードが
    bin プローブに応答しなくなると、OpenClaw はキャッシュされた bin の一致情報を消去します。

  </Accordion>
</AccordionGroup>

## トークンへの影響

スキルが対象となる場合、OpenClaw はコンパクトな XML ブロックをシステム
プロンプトに注入します。コストは決定論的で、スキルごとに線形に増加します。

- **基本オーバーヘッド**（1 つ以上のスキルが対象の場合のみ）: 導入文と
  `<available_skills>` ラッパーからなる固定ブロック。
- **スキルごと:** 約 97 文字 + `name`、`description`、`location`
  フィールドの長さ。
- XML エスケープによって `& < > " '` がエンティティに展開され、出現するたびに数文字追加されます。
- 約 4 文字/トークンとして、フィールド長を含める前の 97 文字はスキルごとに約 24 トークンです。

レンダリングされたブロックが設定済みのプロンプト予算
（`skills.limits.maxSkillsPromptChars`）を超える場合、OpenClaw はまず、説明なしのコンパクト形式に
収まる範囲で、できるだけ多くのスキル識別情報（名前、場所、バージョン）を保持します。
次に、残りの予算を短縮された説明に使用します。説明用の予算が
残っていない場合、説明は省略されます。コンパクト形式またはリストの
切り詰めが必要な場合、プロンプトには `openclaw skills check` を示す注記が含まれます。

プロンプトのオーバーヘッドを最小限に抑えるため、説明は短く、内容が明確なものにしてください。

## 関連項目

<CardGroup cols={2}>
  <Card title="Skillsの作成" href="/ja-JP/tools/creating-skills" icon="hammer">
    カスタムスキルを作成するためのステップバイステップガイドです。
  </Card>
  <Card title="スキルワークショップ" href="/ja-JP/tools/skill-workshop" icon="flask">
    エージェントが下書きしたスキルの提案キューです。
  </Card>
  <Card title="Skillsの設定" href="/ja-JP/tools/skills-config" icon="gear">
    完全な `skills.*` 設定スキーマとエージェントの許可リストです。
  </Card>
  <Card title="スラッシュコマンド" href="/ja-JP/tools/slash-commands" icon="terminal">
    スキルのスラッシュコマンドが登録され、ルーティングされる仕組みです。
  </Card>
  <Card title="ClawHub" href="/clawhub" icon="cloud">
    公開レジストリでスキルを閲覧、公開できます。
  </Card>
  <Card title="プラグイン" href="/ja-JP/tools/plugin" icon="plug">
    プラグインは、説明対象のツールとともにスキルを配布できます。
  </Card>
</CardGroup>
