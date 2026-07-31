---
read_when:
    - 新しいカスタムスキルを作成しています
    - SKILL.md ベースの Skills 向けクイックスタートワークフローが必要です
    - Skill Workshop を使用して、エージェントレビュー用のスキルを提案する場合
sidebarTitle: Creating skills
summary: OpenClaw エージェント向けのカスタム SKILL.md ワークスペース Skills をビルド、テスト、公開します。
title: Skills の作成
x-i18n:
    generated_at: "2026-07-26T09:47:02Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: cba2aa863ebd083d4592e8a764dbdc2c30a0dd8aff49d273927e82df0069bc81
    source_path: tools/creating-skills.md
    workflow: 16
---

Skills は、ツールをいつ、どのように使用するかをエージェントに教えます。各 Skill は、YAML フロントマターと Markdown の手順を記述した
`SKILL.md` ファイルを含むディレクトリです。
OpenClaw は、定義された[優先順位](/ja-JP/tools/skills#loading-order)に従って複数のルートから Skills を読み込みます。

## 最初の Skill を作成する

<Steps>
  <Step title="Skill ディレクトリを作成する">
    Skills はワークスペースの `skills/` フォルダーに配置します。

    ```bash
    mkdir -p ~/.openclaw/workspace/skills/hello-world
    ```

    整理のために Skills をサブフォルダーにまとめることもできます。Skill の名前はフォルダーパスではなく、
    `SKILL.md` フロントマターによって決まります。

    ```bash
    mkdir -p ~/.openclaw/workspace/skills/personal/hello-world
    # Skill 名は引き続き「hello-world」で、/hello-world として呼び出す
    ```

  </Step>

  <Step title="SKILL.md を記述する">
    フロントマターでメタデータを定義し、本文でエージェントへの手順を示します。

    ```markdown
    ---
    name: hello-world
    description: 挨拶を出力するシンプルな Skill。
    ---

    # Hello World

    ユーザーが挨拶を求めたら、`exec` ツールを使用して次を実行します。

    ```bash
    echo "カスタム Skill からこんにちは！"
    ```
    ```

    命名規則:
    - `name` には小文字、数字、ハイフンを使用します。
    - ディレクトリ名とフロントマターの `name` を一致させます。
    - `description` はエージェントとスラッシュコマンドの検出結果に表示されます。
      1 行かつ 160 文字未満にします。

  </Step>

  <Step title="Skill が読み込まれたことを確認する">
    ```bash
    openclaw skills list
    ```

    OpenClaw はデフォルトで、Skills ルート配下の `SKILL.md` ファイルを監視します。監視機能が無効になっている場合や、
    既存のセッションを継続している場合は、エージェントが更新後の一覧を受け取れるように
    新しいセッションを開始します。

    ```bash
    # チャットから — 現在のセッションをアーカイブして新しく開始
    /new

    # または Gateway を再起動
    openclaw gateway restart
    ```

  </Step>

  <Step title="テストする">
    ```bash
    openclaw agent --message "挨拶して"
    ```

    または、チャットを開いてエージェントに直接依頼します。名前を明示して呼び出すには
    `/skill hello-world` を使用します。

  </Step>
</Steps>

## SKILL.md リファレンス

### 必須フィールド

| フィールド         | 説明                                                     |
| ------------- | --------------------------------------------------------------- |
| `name`        | 小文字、数字、ハイフンを使用した一意のスラッグ        |
| `description` | エージェントと検出結果に表示される 1 行の説明 |

### 任意のフロントマターキー

| フィールド                      | デフォルト | 説明                                                                      |
| -------------------------- | ------- | -------------------------------------------------------------------------------- |
| `user-invocable`           | `true`  | Skill をユーザー用スラッシュコマンドとして公開する                                         |
| `disable-model-invocation` | `false` | Skill をエージェントのシステムプロンプトに含めない（`/skill` を介した実行は可能）        |
| `command-dispatch`         | —       | `tool` に設定すると、モデルを経由せずにスラッシュコマンドをツールへ直接ルーティングする |
| `command-tool`             | —       | `command-dispatch: tool` が設定されている場合に呼び出すツール名                         |
| `command-arg-mode`         | `raw`   | ツールへのディスパッチ時に、生の引数文字列をツールへ転送する                      |
| `homepage`                 | —       | macOS の Skills UI で「Website」として表示される URL                                    |

ゲーティングフィールド（`requires.bins`、`requires.env` など）については、
[Skills — ゲーティング](/ja-JP/tools/skills#gating)を参照してください。

### `{baseDir}` の使用

パスをハードコードせずに Skill ディレクトリ内のファイルを参照できます。エージェントは
`{baseDir}` を Skill 自身のディレクトリを基準に解決します。

```markdown
`{baseDir}/scripts/run.sh` にあるヘルパースクリプトを実行します。
```

## 条件付き有効化を追加する

依存関係が利用できる場合にのみ読み込まれるよう、Skill にゲートを設定します。

```markdown
---
name: gemini-search
description: Gemini CLI を使用して検索する。
metadata: { "openclaw": { "requires": { "bins": ["gemini"] }, "primaryEnv": "GEMINI_API_KEY" } }
---
```

<AccordionGroup>
  <Accordion title="ゲーティングオプション">
    | キー | 説明 |
    | --- | --- |
    | `requires.bins` | すべてのバイナリが `PATH` に存在する必要がある |
    | `requires.anyBins` | 少なくとも 1 つのバイナリが `PATH` に存在する必要がある |
    | `requires.env` | 各環境変数がプロセスまたは設定に存在する必要がある |
    | `requires.config` | 各 `openclaw.json` パスが truthy である必要がある |
    | `os` | プラットフォームフィルター: `["darwin"]`、`["linux"]`、`["win32"]` |
    | `always` | `true` に設定すると、すべてのゲートをスキップして常に Skill を含める |

    完全なリファレンス: [Skills — ゲーティング](/ja-JP/tools/skills#gating)。

  </Accordion>
  <Accordion title="環境変数と API キー">
    `openclaw.json` の Skill エントリに API キーを関連付けます。

    ```json5
    {
      skills: {
        entries: {
          "gemini-search": {
            enabled: true,
            apiKey: { source: "env", provider: "default", id: "GEMINI_API_KEY" },
          },
        },
      },
    }
    ```

    キーは、そのエージェントターンの間だけホストプロセスに注入されます。
    サンドボックスには渡されません。詳しくは
    [サンドボックス化された環境変数](/ja-JP/tools/skills-config#sandboxed-skills-and-env-vars)を参照してください。

  </Accordion>
</AccordionGroup>

## Skill Workshop を介して提案する

エージェントが作成した Skills や、Skill を稼働させる前にオペレーターによるレビューが必要な場合は、
`SKILL.md` を直接記述する代わりに、[Skill Workshop](/ja-JP/tools/skill-workshop) の提案を使用します。

```bash
# 新しい Skill を提案
openclaw skills workshop propose-create \
  --name "hello-world" \
  --description "挨拶を出力するシンプルな Skill。" \
  --proposal ./PROPOSAL.md

# 既存の Skill の更新を提案
openclaw skills workshop propose-update hello-world \
  --proposal ./PROPOSAL.md \
  --description "更新された挨拶 Skill"
```

提案にサポートファイルが含まれる場合は、`--proposal-dir` を使用します。

```bash
openclaw skills workshop propose-create \
  --name "hello-world" \
  --description "挨拶を出力するシンプルな Skill。" \
  --proposal-dir ./hello-world-proposal/
```

ディレクトリのルートには `PROPOSAL.md` が必要です。サポートファイルは
`assets/`、`examples/`、`references/`、`scripts/`、または `templates/` の下に配置します。

レビュー後:

```bash
openclaw skills workshop inspect <proposal-id>
openclaw skills workshop apply <proposal-id>
```

提案のライフサイクル全体については、[Skill Workshop](/ja-JP/tools/skill-workshop)を参照してください。

## ClawHub に公開する

<Steps>
  <Step title="SKILL.md が完成していることを確認する">
    `name`、`description`、および `metadata.openclaw` のゲーティングフィールドが
    設定されていることを確認します。プロジェクトページがある場合は、`homepage` URL を追加します。
  </Step>
  <Step title="スタンドアロンの ClawHub CLI をインストールしてログインする">
    ```bash
    npm i -g clawhub
    clawhub login
    ```
  </Step>
  <Step title="公開する">
    ```bash
    clawhub skill publish ./path/to/hello-world
    ```

    推測されたバージョンを上書きするか、特定の所有者として公開するには、
    `--version <version>` または `--owner <owner>` を追加します。完全な手順、所有者スコープ、その他の
    メンテナンスコマンド（`clawhub sync`、`clawhub skill rename` など）については、
    [ClawHub — 公開](/ja-JP/clawhub/publishing)および
    [ClawHub CLI](/ja-JP/clawhub/cli)を参照してください。

  </Step>
</Steps>

## ベストプラクティス

<Tip>
  - **簡潔にする** — AI としてどう振る舞うかではなく、*何を*行うかをモデルに指示します。
  - **安全を最優先する** — Skill が `exec` を使用する場合、信頼できない入力から
    任意のコマンドを注入できないようにプロンプトを設計します。
  - **ローカルでテストする** — 共有する前に `openclaw agent --message "..."` を使用します。
  - **ClawHub を使用する** — ゼロから構築する前に、[clawhub.ai](https://clawhub.ai) で
    コミュニティの Skills を参照します。
</Tip>

## 関連項目

<CardGroup cols={2}>
  <Card title="Skills リファレンス" href="/ja-JP/tools/skills" icon="puzzle-piece">
    読み込み順序、ゲーティング、許可リスト、SKILL.md の形式。
  </Card>
  <Card title="Skill Workshop" href="/ja-JP/tools/skill-workshop" icon="flask">
    エージェントが作成した Skills の提案キュー。
  </Card>
  <Card title="Skills の設定" href="/ja-JP/tools/skills-config" icon="gear">
    `skills.*` の完全な設定スキーマ。
  </Card>
  <Card title="ClawHub" href="/clawhub" icon="cloud">
    公開レジストリで Skills を閲覧して公開します。
  </Card>
  <Card title="Plugin の構築" href="/ja-JP/plugins/building-plugins" icon="plug">
    Plugins は、説明対象のツールとともに Skills を配布できます。
  </Card>
</CardGroup>
