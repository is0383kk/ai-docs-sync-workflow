---
read_when:
    - CLAW.md マニフェストを作成または検証しています
    - Claw から 1 つのエージェントをプレビューまたは追加したい場合
    - Claw の所有権、ドリフト、またはクリーンアップ動作を調査する必要がある
summary: 実験的な Claw エージェントパッケージを作成、追加、更新、削除する
title: クローズ
x-i18n:
    generated_at: "2026-07-26T08:56:42Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: da4b52bdee2b4cf4898677aadeeabb2c0cf98e7c3c53cec6f0b4c6d0b8ab3ae5
    source_path: cli/claws.md
    workflow: 16
---

# `openclaw claws`

Claw は、新しい OpenClaw エージェント 1 つのためのバージョン管理されたセットアップです。
エージェントのポータブルなアイデンティティ、ワークスペースファイル、Skills、Plugin、MCP サーバー、
Cron ジョブを記述できます。ハーネス固有のエージェント設定は、参照される
パッケージプロファイルに含めることができます。Claw は既存のエージェントを置き換えたり変更したりしません。

Claw は試験的機能です。そのスキーマ、コマンド出力、ライフサイクルは変更される可能性があります。
コマンドサーフェスを明示的に有効化してください。

```bash
export OPENCLAW_EXPERIMENTAL_CLAWS=1
```

現在の CLI は、ローカルパッケージディレクトリ、`CLAW.md`、またはグループ化された JSON マニフェストを読み取ります。
ClawHub を介した Claw 全体の公開、検索、インストールは
別のレジストリトラックであり、まだこのコマンドサーフェスには含まれていません。

## Claw パッケージを作成する

パッケージには、`package.json`、`CLAW.md` マニフェスト、およびそのマニフェストから参照される
プロファイルやワークスペースサイドカーが含まれます。

```json
{
  "name": "@acme/incident-triage-claw",
  "version": "1.0.0",
  "type": "module",
  "openclaw": { "claw": "CLAW.md" }
}
```

`CLAW.md` は YAML フロントマターで始まります。その Markdown 本文は、人向けに Claw を説明するものであり、
エージェント設定の一部ではありません。

```md
---
schemaVersion: 1
agent:
  id: incident-triage
  name: Incident triage
metadata:
  openclaw.config: profiles/openclaw.yml
workspace:
  bootstrapFiles: {}
packages: []
mcpServers: {}
cronJobs: []
---

# インシデントトリアージ

インシデントをレビューして振り分けるエージェントを 1 つ作成します。
```

`metadata` は、ポータブルなコンシューマーヒントのための文字列から文字列へのマップです。OpenClaw の
`openclaw.config` キーは、任意のパッケージ相対 YAML プロファイルを指します。
エクスポートされるデフォルトは `profiles/openclaw.yml` です。このポインターが規範となるため、
パッケージは別の安全な相対 `.yml` または `.yaml` パスを選択できます。

```yaml
schemaVersion: 1
agent:
  tools:
    profile: coding
    alsoAllow: [cron]
    deny: [exec]
    fs:
      workspaceOnly: true
  memory:
    search:
      enabled: true
      rememberAcrossConversations: true
      sources: [memory, sessions]
```

このプロファイルは Claw パッケージ内にのみ存在します。OpenClaw は、その Claw の検査、追加、更新、エクスポート時に
このプロファイルを検証して使用しますが、ユーザーの通常の OpenClaw 設定パスにはコピーしません。
他のハーネスは、名前空間付きメタデータキーを無視し、ポータブルなマニフェストフィールドを利用できます。

同じ厳密なバージョン 1 スキーマは、引き続きグループ化された JSON マニフェストを受け入れます。
グループ化された JSON では、OpenClaw プロファイルの 2 つ目のコピーを埋め込むのではなく、
同じ `metadata.openclaw.config` ポインターを使用します。このページの残りのスキーマ断片では JSON を使用しますが、
`CLAW.md` フロントマターでも同等のキーを使用できます。

OpenClaw パッケージプロファイルは、実行中の OpenClaw バージョンに登録されている任意の組み込みツールプロファイルを選択し、
`alsoAllow`、`deny`、`tools.fs.workspaceOnly: true` で調整できます。
Claw はこのフィールドを `false` に設定して、ホストファイルシステムの制限を弱めることはできません。
`tools.allow` は明示的な許可リストとして引き続き使用できますが、`alsoAllow` と組み合わせることはできません。
Claw は `memory.search.enabled` も設定でき、ポータブルな `memory` および `sessions` ソースを選択し、
`rememberAcrossConversations` により会話をまたぐメモリをオプトインできます。
`sessions` ソースを宣言するには、そのオプトインが必要です。
ホストポリシーは引き続きこれらの設定を制約し、Claw にカスタムプロファイル定義、プロバイダー、
認証情報、バインディング、ローカルメモリパスを含めることはできません。
参照されるプロファイルは 256 KiB 以下で、JSON 互換の YAML でなければならず、
エイリアス、アンカー、タグ、マージキーを使用できません。また、パッケージ内の通常ファイルであり、
シンボリックリンクやハードリンクであってはなりません。

パッケージおよびワークスペースのパスは、パッケージルート内に収める必要があります。マニフェストは
1 MiB、パッケージメタデータは 256 KiB に制限され、ワークスペースソースには
ファイルごとおよび合計の個別制限が適用されます。ワークスペースソースは、シンボリックリンクされた
親ディレクトリも拒否します。

ワークスペースファイルはパスで宣言され、パッケージサイドカーから読み取られます。`SOUL.md` などの
ブートストラップファイルには名前付きエントリを使用し、追加ファイルにはパッケージ相対の
ソースとワークスペース相対のターゲットを使用します。

```json
{
  "workspace": {
    "bootstrapFiles": {
      "SOUL.md": { "source": "workspace/SOUL.md" }
    },
    "files": [
      {
        "source": "workspace/reference/policy.md",
        "path": "reference/policy.md"
      }
    ]
  }
}
```

Skills と Plugin には、ClawHub の正確なバージョンを使用します。

```json
{
  "packages": [
    {
      "kind": "skill",
      "source": "clawhub",
      "ref": "incident-triage",
      "version": "1.0.0"
    },
    {
      "kind": "plugin",
      "source": "clawhub",
      "ref": "@acme/audit-plugin",
      "version": "2.0.0"
    }
  ]
}
```

ドライランでは、既存の Skills および Plugin のプリフライトパスを使用し、同意前に正確なアーティファクト、
整合性、および ClawHub の信頼性に関する警告を解決します。警告は、整合性に紐づけられたプラン内でも
引き続き表示されます。適用時には、不足しているアーティファクトをインストールするか、一致するものを再利用し、
各リソースが Claw によって導入されたか参照されたかを記録します。Plugin はエージェントごとの
インストールではなく、プロセス全体の OpenClaw 機能として維持されます。

Cron ジョブは、新しいエージェントのスケジュールされた作業を宣言します。

```json
{
  "cronJobs": [
    {
      "id": "daily-summary",
      "name": "Daily incident summary",
      "schedule": { "cron": "0 9 * * *", "timezone": "UTC" },
      "session": "isolated",
      "message": "Summarize active incidents."
    }
  ]
}
```

Claw は既存の Gateway スケジューラーを使用し、作成されたジョブを新しい
エージェントにバインドします。プレビュー、プロベナンス、ステータス、削除は、通常の Cron コマンドの
動作を変更せずに、それらのジョブを対象とします。削除時には Gateway を介して稼働中のジョブを再読み込みし、
プラン作成後に所有対象の定義が変更されていた場合は、そのジョブを保持します。

MCP 宣言では、既存の `mcp.servers` 設定モデルを使用します。

```json
{
  "mcpServers": {
    "statuspage": {
      "command": "npx",
      "args": ["--yes", "@acme/statuspage-mcp@1.0.0"],
      "env": { "STATUSPAGE_TOKEN": "${STATUSPAGE_TOKEN}" }
    }
  }
}
```

環境参照は参照のまま維持され、Claw が解決済みのシークレット値を埋め込むことはありません。
競合のない宣言は管理対象となり、完全に一致する既存または共有の宣言は参照対象となります。
プレビュー、プロベナンス、ステータス、エクスポート、削除には、他の Claw リソースと同じ
所有権ポリシーが適用されます。

## 検査とプレビュー

ローカル変更を計画せずにソースを検証します。

```bash
openclaw claws inspect ./incident-triage.claw.json
```

提案されるすべてのライフサイクルアクションをプレビューします。

```bash
openclaw claws add ./incident-triage.claw.json --dry-run --json
```

プランは、導出されたエージェントとワークスペース、提案されるすべてのアクション、
前提条件、ブロッカー、個別の権限昇格、および `planIntegrity`
ダイジェストを報告します。機能レコードには、パッケージ、MCP、スケジュールされた作業、サンドボックス、
ツール、または Heartbeat への正確な影響が示されます。エージェントを作成する前にプランを確認してください。

```bash
openclaw claws add ./incident-triage.claw.json \
  --yes \
  --plan-integrity <SHA256_FROM_DRY_RUN>
```

`--yes` だけでは不十分です。OpenClaw はプランを再構築し、プレビュー後にソース、
宛先、または稼働中の設定が変更された場合は、同意を拒否します。パッケージのデフォルトが
ローカル状態と競合する場合、プレビューと適用の両方で `--agent-id` または
`--workspace` を使用してください。一時的なプロファイルや並列検証では、
明示的な `--workspace` を渡してください。`OPENCLAW_STATE_DIR` はランタイム状態を移動しますが、
デフォルトのワークスペース位置は変更しません。

Claw を追加すると、新しいエージェントとワークスペース設定が作成され、宣言された
ワークスペースファイルが書き込まれ、宣言された Skills および Plugin アーティファクトが
インストールまたは再利用され、パッケージ、MCP、Cron のプロベナンスが記録されます。
既存のファイルは上書きされず、所有対象の内容にドリフトがある場合、再試行は安全側に失敗します。

## インストール済み状態を検査する

```bash
openclaw claws status
openclaw claws status incident-triage --json
openclaw doctor
```

`status` は、インストール済みのエージェントと、記録されたワークスペース、パッケージ、MCP、
Cron のプロベナンスを現在の状態と比較します。ローカル状態を変更せずに、不完全なインストール、
欠落したリソース、ドリフトを報告します。`openclaw doctor` は、不完全な所有権レコード、
安全でない管理対象ファイル、稼働中の Gateway インベントリで裏付けられない Cron ジョブについて、
Claw 固有の診断を追加します。

Claw のプロベナンスは、次の 2 つの関係を区別します。

- **管理対象:** Claw がそのリソースを導入し、現在管理しています。変更されておらず、
  競合する所有者が残っていない場合は、クリーンアップ候補になります。
- **参照対象:** リソースは独立して存在していたか、共有されています。削除時には
  この Claw の参照を解放し、デフォルトではリソースを保持します。

これは参照カウントではありません。通常の Plugin、Skills、エージェントの各コマンドは
既存の動作を維持し、Claw はその上にプロベナンスと保護されたライフサイクル操作を追加します。

## インストール済みの Claw を更新する

デフォルトでは、更新時に Claw の追加時に記録されたソースを使用します。そのソースが移動した場合や、
別のパッケージディレクトリをテストする場合は、`--from` を使用します。

```bash
openclaw claws update incident-triage --dry-run --json
openclaw claws update incident-triage \
  --from ./incident-triage-next \
  --dry-run --json
```

プランは、現在のプロベナンスおよび稼働中の状態と、対象マニフェストを比較します。
権限昇格やブロッカーを含む、エージェント、ワークスペース、パッケージ、MCP、Cron、
所有権の変更を報告します。権限昇格には個別の機械可読レコードがあり、人間向け出力には
正確に秘匿化された影響を示す `!` 行が含まれます。
解決されたパッケージ整合性、インストール識別情報、信頼性に関する警告も含まれます。
パッケージ宣言を削除すると、更新中にアーティファクトをアンインストールすることなく、
この Claw のエッジが解放されます。最終的な正確な `planIntegrity` 確認は、
通常の内容変更に加えて、開示されたその集合にも紐づきます。ホストは同じレコードを、
個別のダイアログや複数エージェントの集約レビューに使用できます。
明示的な同意により、レビュー済みの正確なプランを適用します。

```bash
openclaw claws update incident-triage \
  --yes \
  --plan-integrity <SHA256_FROM_DRY_RUN>
```

OpenClaw はプランを再構築し、各変更前に所有対象の状態を比較交換します。
削除されたパッケージ宣言は、アーティファクトをアンインストールせずに依存関係エッジを解放します。
Cron の変更では、稼働中のスケジューラー定義を再読み込みし、オペレーターによるドリフトがある場合は停止します。
パッケージインストーラー、ソース設定ライター、Gateway スケジューラーは 1 つのトランザクションではありません。
外部変更後に補償を証明できない場合、OpenClaw は構造化された
`status: partial` とともにエラーコード `update_partial` を報告し、
不確実なプロベナンスを保持して停止します。`claws status`、影響を受けたリソース、
`openclaw doctor` を検査し、その後、再試行または何かを削除する前にもう一度プレビューしてください。

## インストール済みの Claw を削除する

クリーンアップを選択する前に削除をプレビューします。

```bash
openclaw claws remove incident-triage --dry-run --json
openclaw claws remove incident-triage \
  --yes \
  --plan-integrity <SHA256_FROM_DRY_RUN>
```

デフォルトでは、対象となる管理状態を削除し、参照状態を解放します。
変更されたファイルや、現在の別の所有者がいるリソースは保持されるか、ブロックされます。
クリーンアップの選択はプランダイジェストの一部であり、`--yes` によって
その範囲が拡張されることはありません。グローバルにインストールされた Plugin は、
この Claw の参照が解放されても保持されます。プロセス全体の Plugin をアンインストールする場合は、
通常の Plugin ライフサイクルを別途使用してください。

他に現在の所有者がいない、変更されていない Claw 導入の参照を削除するには、
プレビューと適用の両方に `--remove-unused` を含めます。代わりに特定の参照リソースを
選択するには、`--remove-referenced` を繰り返し指定します。

```bash
openclaw claws remove incident-triage \
  --dry-run \
  --remove-referenced 'plugin:@acme/audit-plugin@2.0.0'
```

表示された依存対象、独立した所有者、既存の由来を確認した後でのみ、
`--force-referenced` を使用してください。これにより、それらの競合があっても選択した
クリーンアップが可能になりますが、プラン整合性への同意は省略されません。

## インストール済みエージェントをエクスポートする

エクスポートは新しいパッケージディレクトリを作成し、出力先がすでに存在する場合や
管理対象の状態にドリフトがある場合は失敗します。

```bash
openclaw claws export incident-triage --out ./incident-triage-export --json
```

結果には `package.json`、正規の `CLAW.md`、および管理対象ワークスペースの
サイドカーが含まれます。これはインスタンス全体のバックアップではなく、移植可能な Claw パッケージです。無関係な
エージェント、認証情報、セッション、および所有対象外のローカル状態は除外されます。

## コマンドリファレンス

| コマンド                             | 目的                                             |
| ----------------------------------- | --------------------------------------------------- |
| `claws inspect <source>`            | パッケージディレクトリまたはグループ化されたマニフェストを検証します。   |
| `claws add <source>`                | 1 つの新しいエージェントとワークスペースをプレビューまたは作成します。      |
| `claws status [claw-or-agent]`      | インストール済みの状態、所有権、ドリフトを報告します。       |
| `claws update <claw-or-agent>`      | 選択したソースからの変更をプレビューまたは適用します。  |
| `claws remove <claw-or-agent>`      | エージェントと削除対象となるリソースをプレビューまたは削除します。 |
| `claws export <agent> --out <path>` | インストール済みエージェントから移植可能なパッケージを作成します。  |

試験的な機械可読出力には `--json` を使用します。

## 関連項目

- [エージェント](/ja-JP/cli/agents)
- [Skills](/ja-JP/tools/skills)
- [Plugin](/ja-JP/tools/plugin)
- [Cron ジョブ](/ja-JP/automation/cron-jobs)
- [MCP 設定](/ja-JP/gateway/configuration-reference#mcp)
