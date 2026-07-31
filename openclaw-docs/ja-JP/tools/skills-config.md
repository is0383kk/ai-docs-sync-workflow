---
read_when:
    - Skills の読み込み、インストール、またはゲーティング動作の設定
    - エージェントごとの Skills の表示設定
    - Skill Workshop の制限または承認ポリシーの調整
sidebarTitle: Skills config
summary: skills.* 設定スキーマ、エージェントの許可リスト、ワークショップ設定、サンドボックスの環境変数処理に関する完全なリファレンス。
title: Skills 設定
x-i18n:
    generated_at: "2026-07-26T09:23:54Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: bc154bdf8a8537095a4d39bc6e86ebfd716e6beacd45def9c8a1c15fcdc93698
    source_path: tools/skills-config.md
    workflow: 16
---

Skills の設定の大半は `~/.openclaw/openclaw.json` 内の
`skills` にあります。エージェント固有の可視性は
`agents.defaults.skills` と `agents.entries.*.skills` にあります。

```json5
{
  skills: {
    allowBundled: ["gemini", "peekaboo"],
    load: {
      extraDirs: ["~/Projects/agent-scripts/skills"],
      allowSymlinkTargets: ["~/Projects/manager/skills"],
      watch: true,
    },
    install: {
      preferBrew: true,
      nodeManager: "npm",
      allowUploadedArchives: false,
    },
    workshop: {
      autonomous: { enabled: false },
      allowSymlinkTargetWrites: false,
      approvalPolicy: "auto",
      maxPending: 50,
      maxSkillBytes: 40000,
    },
    entries: {
      "image-lab": {
        enabled: true,
        apiKey: { source: "env", provider: "default", id: "GEMINI_API_KEY" },
        env: { GEMINI_API_KEY: "GEMINI_KEY_HERE" },
      },
      peekaboo: { enabled: true },
      sag: { enabled: false },
    },
  },
}
```

<Note>
  組み込みの画像生成には、`skills.entries` ではなく `agents.defaults.mediaModels.image`
  とコアの `image_generate` ツールを使用してください。Skill
  エントリは、カスタムまたはサードパーティの Skill ワークフロー専用です。
</Note>

## 読み込み（`skills.load`）

<ParamField path="skills.load.extraDirs" type="string[]">
  スキャン対象に追加する Skill ディレクトリです。優先順位は最も低く（バンドルおよび
  Plugin の Skills より下）、パスは `~` をサポートして展開されます。
</ParamField>

<ParamField path="skills.load.allowSymlinkTargets" type="string[]">
  シンボリックリンクされた Skill フォルダーの解決先として許可する、信頼済みの実体ディレクトリです。
  シンボリックリンクが設定済みルートの外部にある場合でも使用できます。
  `<workspace>/skills/manager -> ~/Projects/manager/skills` のような意図的な兄弟リポジトリ構成に使用してください。
  このリストは限定的に保ち、`~` や `~/Projects` のような
  広範なルートを指定しないでください。
</ParamField>

<ParamField path="skills.load.watch" type="boolean" default="true">
  Skill フォルダーを監視し、`SKILL.md` ファイルが変更されたときに
  Skills のスナップショットを更新します。グループ化された Skill ルート配下のネストされたファイルも対象です。
</ParamField>

## インストール（`skills.install`）

<ParamField path="skills.install.preferBrew" type="boolean" default="true">
  `brew` が利用可能な場合は、Homebrew インストーラーを優先します。
</ParamField>

<ParamField path="skills.install.nodeManager" type='"npm" | "pnpm" | "yarn" | "bun"' default='"npm"'>
  Skill のインストールに使用する Node パッケージマネージャーの優先設定です。これは Skill の
  インストールのみに影響します。正規の状態ストアが `node:sqlite` を使用するため、
  OpenClaw CLI と Gateway ランタイムには Node が必要です。`openclaw setup --node-manager` と
  `openclaw onboard --node-manager` は `npm`、`pnpm`、`bun` を受け付けます。
  Yarn を使用する Skill のインストールでは、設定内で `"yarn"` を直接指定してください。
</ParamField>

<ParamField path="skills.install.allowUploadedArchives" type="boolean" default="false">
  信頼済みの `operator.admin` Gateway クライアントが、`skills.upload.*` を通じて
  ステージングされた非公開 zip アーカイブをインストールできるようにします。
  通常の ClawHub インストールでは、この設定は不要です。
</ParamField>

## オペレーターのインストールポリシー（`security.installPolicy`）

ホスト固有のポリシーに基づいて Skill と Plugin のインストールを承認またはブロックする、
信頼済みのローカルコマンドがオペレーターに必要な場合は、`security.installPolicy` を使用します。
このポリシーは、OpenClaw がソース素材をステージングした後、インストールまたは更新を続行する前に実行されます。
ClawHub の Skills、アップロードされた Skills、Git/ローカルの Skills、Skill の依存関係インストーラー、
および Plugin のインストール/更新ソースに適用されます。

```json5
{
  security: {
    installPolicy: {
      enabled: true,
      // サポートされているすべての対象を含める場合は、targets を省略します。
      targets: ["skill", "plugin"],
      exec: {
        source: "exec",
        command: "/usr/local/bin/openclaw-install-policy",
        args: ["--json"],
        timeoutMs: 10000,
        noOutputTimeoutMs: 10000,
        maxOutputBytes: 1048576,
        passEnv: ["OPENCLAW_STATE_DIR", "PATH"],
        env: { POLICY_MODE: "strict" },
        trustedDirs: ["/usr/local/bin"],
      },
    },
  },
}
```

<ParamField path="security.installPolicy.enabled" type="boolean" default="false">
  オペレーターが管理するインストールポリシーを有効にします。有効な `exec`
  コマンドがない状態で有効にすると、インストールはフェイルクローズします。
</ParamField>

<ParamField path="security.installPolicy.targets" type='("skill" | "plugin")[]'>
  省略可能な対象フィルターです。省略した場合は、すべてのサポート対象にポリシーが適用されるため、
  新しいインストールが予期せずフェイルオープンすることはありません。
</ParamField>

<ParamField path="security.installPolicy.exec.command" type="string">
  信頼済みポリシー実行ファイルへの絶対パスです。OpenClaw はシェルを介さずに実行し、
  使用前にパスを検証します。
</ParamField>

<ParamField path="security.installPolicy.exec.args" type="string[]">
  `command` の後に渡す固定引数です。
</ParamField>

<ParamField path="security.installPolicy.exec.timeoutMs" type="number" default="10000">
  1 回のポリシー判定に許可する最大実時間です。
</ParamField>

<ParamField path="security.installPolicy.exec.noOutputTimeoutMs" type="number" default="timeoutMs">
  stdout または stderr からの出力がない状態で許可する最大時間です。
  超過するとポリシーはフェイルクローズします。
</ParamField>

<ParamField path="security.installPolicy.exec.maxOutputBytes" type="number" default="1048576">
  ポリシープロセスから受け付ける stdout と stderr の合計最大バイト数です。
</ParamField>

<ParamField path="security.installPolicy.exec.env" type="Record<string, string>">
  ポリシープロセスに渡すリテラル環境変数です。
</ParamField>

<ParamField path="security.installPolicy.exec.passEnv" type="string[]">
  OpenClaw プロセスからポリシープロセスへコピーする環境変数名です。
  指定された変数のみが渡されます。
</ParamField>

<ParamField path="security.installPolicy.exec.trustedDirs" type="string[]">
  ポリシー実行ファイルの配置を許可するディレクトリの省略可能な許可リストです。
</ParamField>

<ParamField path="security.installPolicy.exec.allowInsecurePath" type="boolean" default="false">
  コマンドパスの所有権および権限チェックを回避します。
  パスが別の仕組みによって保護されている場合にのみ使用してください。
</ParamField>

<ParamField path="security.installPolicy.exec.allowSymlinkCommand" type="boolean" default="false">
  設定されたコマンドパスがシンボリックリンクであることを許可します。解決後の対象は、
  引き続き他のパスチェックを満たす必要があります。インタープリターのスクリプト引数は、
  シンボリックリンクではなく、直接の通常ファイルでなければなりません。
</ParamField>

ポリシーは stdin で、`protocolVersion: 1`、`openclawVersion`、`targetType`、
`targetName`、`sourcePath`、`sourcePathKind`、省略可能な構造化
`source`、構造化 `origin`、および `request` を含む
1 つの JSON オブジェクトを受け取ります。stdout には、`{ "protocolVersion": 1, "decision": "allow" }`
または `{ "protocolVersion": 1, "decision": "block", "reason": "..." }` の 1 つの JSON オブジェクトを書き込む必要があります。
ゼロ以外の終了コード、タイムアウト、不正な JSON、フィールド不足、または未対応のプロトコルバージョンでは
フェイルクローズします。

OpenClaw は通常の Gateway 起動時にはインストールポリシーを実行しません。
ポリシーが有効であるにもかかわらず利用できない場合、インストールと更新はフェイルクローズします。
`openclaw doctor` は静的検証を実行し、`openclaw doctor --deep` は
設定されたコマンドに対して合成インストールプローブを実行します。

一括更新では対象ごとにポリシーが適用されます。ブロックされた Skill または Plugin の更新は、
ポリシーを無効化したり、バッチ内の後続対象をスキップしたりすることなく、その対象のみ失敗します。

stdin の例：

```json
{
  "protocolVersion": 1,
  "openclawVersion": "2026.6.1",
  "targetType": "skill",
  "targetName": "weather",
  "sourcePath": "/var/folders/.../openclaw-skill-clawhub/root",
  "sourcePathKind": "directory",
  "source": {
    "kind": "clawhub",
    "authority": "openclaw",
    "mutable": false,
    "network": true
  },
  "origin": {
    "type": "clawhub",
    "registry": "https://clawhub.openclaw.ai",
    "slug": "weather",
    "version": "1.0.0"
  },
  "request": {
    "kind": "skill-install",
    "mode": "install",
    "requestedSpecifier": "clawhub:weather@1.0.0"
  },
  "skill": {
    "installId": "clawhub"
  }
}
```

最小限のポリシーコマンド：

```js
#!/usr/bin/env node

let input = "";
process.stdin.setEncoding("utf8");
process.stdin.on("data", (chunk) => {
  input += chunk;
});
process.stdin.on("end", () => {
  const request = JSON.parse(input);
  if (request.targetType === "plugin" && request.source?.kind === "local-path") {
    process.stdout.write(
      JSON.stringify({
        protocolVersion: 1,
        decision: "block",
        reason: "このホストではローカル Plugin パスは承認されていません",
      }),
    );
    return;
  }
  process.stdout.write(JSON.stringify({ protocolVersion: 1, decision: "allow" }));
});
```

## バンドルされた Skill の許可リスト

<ParamField path="skills.allowBundled" type="string[]">
  **バンドルされた** Skills のみに適用される省略可能な許可リストです。設定すると、
  リスト内のバンドルされた Skills のみが使用可能になります。管理対象、エージェントレベル、
  およびワークスペースの Skills には影響しません。
</ParamField>

## Skill ごとのエントリ（`skills.entries`）

`entries` 配下のキーは、デフォルトでは Skill の `name` と一致します。
Skill が `metadata.openclaw.skillKey` を定義している場合は、代わりにそのキーを使用します。
ハイフンを含む名前は引用符で囲んでください（JSON5 では引用符付きキーを使用できます）。

<ParamField path="skills.entries.<key>.enabled" type="boolean">
  `false` にすると、バンドル済みまたはインストール済みであっても Skill を無効にします。
  バンドルされた `coding-agent` Skill はオプトインです。`true` に設定し、
  `claude`、`codex`、`opencode`、または別のサポート対象 CLI のいずれかが
  インストールおよび認証済みであることを確認してください。
</ParamField>

<ParamField path="skills.entries.<key>.apiKey" type='string | { source, provider, id }'>
  `metadata.openclaw.primaryEnv` を宣言する Skills 向けの便利なフィールドです。
  平文文字列または SecretRef（`{ source: "env", provider: "default", id: "VAR_NAME" }`）をサポートします。
</ParamField>

<ParamField path="skills.entries.<key>.env" type="Record<string, string>">
  エージェント実行時に注入する環境変数です。プロセス内でその変数がまだ設定されていない場合にのみ注入されます。
</ParamField>

<ParamField path="skills.entries.<key>.config" type="object">
  Skill ごとのカスタム設定フィールドを格納する省略可能なオブジェクトです。
</ParamField>

## エージェントの許可リスト（`agents`）

同じマシン/ワークスペースの Skill ルートを使用しながら、エージェントごとに表示する Skill セットを
変えたい場合は、エージェント設定を使用します。

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

<ParamField path="agents.defaults.skills" type="string[]">
  `agents.entries.*.skills` を省略したエージェントが継承する共有ベースライン許可リストです。
  デフォルトで Skills を制限しない場合は、設定自体を省略してください。
</ParamField>

<ParamField path="agents.entries.*.skills" type="string[]">
  そのエージェントに対する明示的な最終 Skill セットです。明示的なリストは継承したデフォルトを
  **置き換え**、マージしません。そのエージェントに Skills を一切公開しない場合は、
  `[]` に設定します。
</ParamField>

<Warning>
  エージェントの Skill 許可リストは、OpenClaw の Skill 検出、プロンプト、スラッシュコマンド検出、
  サンドボックス同期、および Skill スナップショットに対する可視性と読み込みのフィルターです。
  シェル実行時の認可境界ではありません。エージェントがホストの `exec` を実行できる場合、
  そのシェルは外部クライアントを実行したり、実行ユーザーから見えるホストファイルを読み取ったりできます。
  これには `~/.openclaw/skills/config/mcporter.json` のような MCP クライアントレジストリも含まれます。
  エージェントごとに MCP を分離するには、Skill 許可リストをサンドボックス/OS ユーザー分離と組み合わせ、
  ホストでの exec を拒否するか厳密に許可リスト化し、MCP サーバーではエージェントごとの認証情報を優先してください。
</Warning>

## Workshop（`skills.workshop`）

<ParamField path="skills.workshop.autonomous.enabled" type="boolean" default="false">
  `true` の場合、OpenClaw は永続的な修正から保留中の提案を作成し、
  システムがアイドル状態になった後、正常に完了した実質的な作業をレビューできます。
  これにより、対象となるターンの後にバックグラウンドでモデルが実行される場合があります。
  この設定が `false` の場合でも、ユーザーが指示した
  skill の作成と `/learn` は引き続き機能します。
</ParamField>

対象条件、プライバシー、コスト、提案のみに付与される権限、トラブルシューティングについては、
[自己学習](/ja-JP/tools/self-learning)を参照してください。

<ParamField path="skills.workshop.approvalPolicy" type='"pending" | "auto"' default='"auto"'>
  `auto` では、追加の承認プロンプトなしで、エージェントが適用、却下、
  または隔離を開始できます。`pending` ではオペレーターの承認が必要です。
</ParamField>

<ParamField path="skills.workshop.allowSymlinkTargetWrites" type="boolean" default="false">
  Skill Workshop の適用時に、実体のターゲットが `skills.load.allowSymlinkTargets` によって
  すでに信頼されているワークスペース skill のシンボリックリンクを介した書き込みを許可します。
  生成された提案の適用によって、その共有 skill ルートを変更する必要がある場合を除き、
  この設定は無効のままにしてください。
</ParamField>

<ParamField path="skills.workshop.maxPending" type="number" default="50">
  ワークスペースごとに保持する保留中および隔離済みの提案の最大数
  （許容範囲: 1-200）。
</ParamField>

<ParamField path="skills.workshop.maxSkillBytes" type="number" default="40000">
  提案本文の最大サイズ（バイト単位、許容範囲: 1024-200000）。
  提案の説明は検出結果と一覧出力に表示されるため、これとは別に
  160 バイトにハード制限されます。
</ParamField>

この設定で制御される提案のライフサイクル、CLI コマンド、エージェントツールのパラメーター、
Gateway メソッドについては、[Skill Workshop](/ja-JP/tools/skill-workshop)を参照してください。

## シンボリックリンクされた skill ルート

デフォルトでは、ワークスペース、プロジェクトエージェント、追加ディレクトリ、
およびバンドル済みの skill ルートが包含境界になります。`<workspace>/skills` 配下にある
シンボリックリンクされた skill フォルダーがルート外に解決される場合は、
ログメッセージを出力してスキップされます。

意図的なシンボリックリンク構成を許可するには、信頼するターゲットを宣言します。

```json5
{
  skills: {
    load: {
      extraDirs: ["~/Projects/manager/skills"],
      allowSymlinkTargets: ["~/Projects/manager/skills"],
    },
  },
}
```

この設定では、realpath の解決後に `<workspace>/skills/manager -> ~/Projects/manager/skills` が許可されます。
`extraDirs` は隣接するリポジトリを直接スキャンし、
`allowSymlinkTargets` は既存の構成のためにシンボリックリンクされたパスを維持します。

デフォルトでは、Skill Workshop の適用処理はこれらのシンボリックリンクを介して
書き込みません。すでに信頼されているシンボリックリンクのターゲット配下にある
skill を Workshop の適用処理で変更できるようにするには、別途オプトインします。

```json5
{
  skills: {
    load: {
      allowSymlinkTargets: ["~/Projects/manager/skills"],
    },
    workshop: {
      allowSymlinkTargetWrites: true,
    },
  },
}
```

管理対象の `~/.openclaw/skills` ディレクトリと個人用の `~/.agents/skills`
ディレクトリでは、skill ディレクトリのシンボリックリンクがすでに無条件で許可されています
（skill ごとの `SKILL.md` 包含は引き続き適用されます）。
`allowSymlinkTargets` が必要なのは、ワークスペース、追加ディレクトリ、
およびプロジェクトエージェント（`<workspace>/.agents/skills`）のルートだけです。

## サンドボックス化された skill と環境変数

<Warning>
  `skills.entries.<skill>.env` と `apiKey` は、**ホスト**での実行にのみ適用されます。
  サンドボックス内では効果がありません。`GEMINI_API_KEY` に依存する skill は、
  サンドボックスにその変数を別途渡さない限り、`apiKey not configured` で失敗します。
</Warning>

Docker サンドボックスにシークレットを渡すには、次のようにします。

```json5
{
  agents: {
    defaults: {
      sandbox: {
        docker: {
          env: { GEMINI_API_KEY: "your-key-here" },
        },
      },
    },
  },
}
```

<Note>
  Docker デーモンへのアクセス権を持つユーザーは、Docker メタデータを通じて
  `sandbox.docker.env` の値を確認できます。そのような露出を許容できない場合は、
  マウントしたシークレットファイル、カスタムイメージ、または別の受け渡し方法を使用してください。
</Note>

## 読み込み順序の再確認

```text
workspace/skills      (最高)
workspace/.agents/skills
~/.agents/skills
~/.openclaw/skills
バンドル済みの skill
skills.load.extraDirs (最低)
```

skill と設定への変更は、ウォッチャーが有効な場合は次の新規セッションで、
またはウォッチャーが変更を検出した場合は次のエージェントターンで反映されます。

## 関連項目

<CardGroup cols={2}>
  <Card title="Skills リファレンス" href="/ja-JP/tools/skills" icon="puzzle-piece">
    skill の概要、読み込み順序、ゲーティング、および SKILL.md の形式。
  </Card>
  <Card title="skill の作成" href="/ja-JP/tools/creating-skills" icon="hammer">
    カスタムワークスペース skill の作成。
  </Card>
  <Card title="Skill Workshop" href="/ja-JP/tools/skill-workshop" icon="flask">
    エージェントが下書きした skill の提案キュー。
  </Card>
  <Card title="自己学習" href="/ja-JP/tools/self-learning" icon="brain">
    完了した作業から生成される、保守的なオプトイン方式の提案。
  </Card>
  <Card title="スラッシュコマンド" href="/ja-JP/tools/slash-commands" icon="terminal">
    ネイティブのスラッシュコマンドカタログとチャットディレクティブ。
  </Card>
</CardGroup>
