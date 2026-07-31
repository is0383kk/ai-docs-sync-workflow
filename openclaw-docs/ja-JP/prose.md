---
read_when:
    - .prose ワークフローファイルを実行または作成したい場合
    - OpenProse Plugin を有効にする場合
    - OpenProse が OpenClaw のプリミティブにどのように対応するかを理解する必要があります
sidebarTitle: OpenProse
summary: OpenProse は、マルチエージェント AI セッション向けの Markdown ファーストなワークフロー形式です。OpenClaw では、`/prose` スラッシュコマンドとスキルパックを備えた Plugin として提供されます。
title: OpenProse
x-i18n:
    generated_at: "2026-07-26T09:13:54Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 8b04eb23bf827fbec6db11c1e95993e7f6c617451c5f4fda771ad078674c12bc
    source_path: prose.md
    workflow: 16
---

OpenProse は、AI セッションをオーケストレーションするための、移植可能で Markdown を中心としたワークフロー形式です。OpenClaw では、OpenProse のスキルパックと `/prose` スラッシュコマンドをインストールする Plugin として提供されます。プログラムは `.prose` ファイルに保存され、明示的な制御フローで複数のサブエージェントを起動できます。

<CardGroup cols={3}>
  <Card title="インストール" icon="download" href="#install">
    OpenProse Plugin を有効にして、Gateway を再起動します。
  </Card>
  <Card title="プログラムを実行" icon="play" href="#slash-command">
    `/prose run` を使用して、`.prose` ファイルまたはリモートプログラムを実行します。
  </Card>
  <Card title="プログラムを作成" icon="pencil" href="#example-parallel-research-and-synthesis">
    並列および逐次ステップを使用して、マルチエージェントワークフローを作成します。
  </Card>
</CardGroup>

## インストール

<Steps>
  <Step title="Plugin を有効にする">
    OpenProse はバンドルされていますが、デフォルトでは無効です。有効にするには、次を実行します。

    ```bash
    openclaw plugins enable open-prose
    ```

  </Step>
  <Step title="Gateway を再起動する">
    ```bash
    openclaw gateway restart
    ```
  </Step>
  <Step title="確認する">
    ```bash
    openclaw plugins list | grep prose
    ```

    `open-prose` が有効として表示されます。これで、チャットで `/prose` スキルコマンドを使用できます。

  </Step>
</Steps>

リポジトリのチェックアウトから Plugin を直接インストールすることもできます。
`openclaw plugins install ./extensions/open-prose`

## スラッシュコマンド

OpenProse は、ユーザーが呼び出せるスキルコマンドとして `/prose` を登録します。

```text
/prose help
/prose run <file.prose>
/prose run <handle/slug>
/prose run <https://example.com/file.prose>
/prose compile <file.prose>
/prose examples
/prose update
```

`/prose run <handle/slug>` は `https://p.prose.md/<handle>/<slug>` に解決されます。
直接 URL は、`web_fetch` ツールを使用してそのまま取得されます。

トップレベルのリモート実行は明示的に行われます。`.prose` プログラム内のリモートインポートは推移的なコード依存関係です。OpenProse がリモートの `use` ターゲットを取得する前に、解決済みのインポート一覧を表示し、その実行についてオペレーターが正確に `approve remote prose imports` と返信することを要求します。

## できること

- 明示的な並列処理によるマルチエージェントの調査と統合。
- 反復可能で承認を安全に行えるワークフロー（コードレビュー、インシデントのトリアージ、コンテンツパイプライン）。
- サポートされているエージェントランタイム全体で実行できる、再利用可能な `.prose` プログラム。

## 例：並列調査と統合

```prose
# 2 つのエージェントを並列実行する調査と統合。

input topic: "何を調査しますか？"

agent researcher:
  model: sonnet
  prompt: "徹底的に調査し、出典を引用します。"

agent writer:
  model: opus
  prompt: "簡潔な要約を作成します。"

parallel:
  findings = session: researcher
    prompt: "{topic}を調査してください。"
  draft = session: writer
    prompt: "{topic}を要約してください。"

session "調査結果と下書きを統合して、最終回答を作成してください。"
  context: { findings, draft }
```

## OpenClaw ランタイムへのマッピング

OpenProse プログラムは、OpenClaw のプリミティブに次のように対応します。

| OpenProse の概念          | OpenClaw ツール                                  |
| ------------------------- | ----------------------------------------------- |
| セッションの起動 / Task ツール | `sessions_spawn`                                |
| ファイルの読み取り / 書き込み | `read` / `write`                                |
| Web 取得                  | `web_fetch`（POST が必要な場合は `exec` + curl） |

<Warning>
  ツールの許可リストで `sessions_spawn`、`read`、`write`、または
  `web_fetch` がブロックされている場合、OpenProse プログラムは失敗します。
  [ツール許可リストの設定](/ja-JP/gateway/config-tools)を確認してください。
</Warning>

## ファイルの保存場所

OpenProse は、ワークスペース内の `.prose/` に状態を保存します。

```text
.prose/
├── .env                      # 設定（key=value）、例：OPENPROSE_POSTGRES_URL
├── runs/
│   └── {YYYYMMDD}-{HHMMSS}-{random}/
│       ├── program.prose     # 実行中のプログラムのコピー
│       ├── state.md          # 実行状態
│       ├── bindings/
│       ├── imports/          # ネストされたリモートプログラムの実行
│       └── agents/
└── agents/                   # プロジェクトスコープの永続エージェント
```

プロジェクト間で共有されるユーザーレベルの永続エージェントは、次の場所に保存されます。

```text
~/.prose/agents/
```

## 状態バックエンド

<AccordionGroup>
  <Accordion title="filesystem（デフォルト）">
    状態はワークスペース内の `.prose/runs/...` に書き込まれます。追加の依存関係は必要ありません。
  </Accordion>
  <Accordion title="in-context">
    コンテキストウィンドウに一時的な状態を保持します。`--in-context` で選択します。
    小規模で短時間実行されるプログラムに適しています。
  </Accordion>
  <Accordion title="sqlite（実験的）">
    `--state=sqlite` で選択します。`PATH` に `sqlite3` バイナリが必要です
    （存在しない場合は filesystem にフォールバックします）。状態は
    `.prose/runs/{id}/state.db` に保存されます。
  </Accordion>
  <Accordion title="postgres（実験的）">
    `--state=postgres` で選択します。`psql` と、
    `OPENPROSE_POSTGRES_URL` に接続文字列が必要です（`.prose/.env` で設定します）。

    <Warning>
      Postgres の認証情報はサブエージェントのログに記録されます。専用の最小権限データベースを使用してください。
    </Warning>

  </Accordion>
</AccordionGroup>

## セキュリティ

`.prose` ファイルはコードとして扱ってください。リモートの `use` インポートも含め、実行前にレビューしてください。トップレベルの `/prose run https://...` リクエストは明示的ですが、推移的なリモートインポートについては、取得または実行される前に実行ごとの承認が必要です。副作用を制御するには、OpenClaw のツール許可リストと承認ゲートを使用してください。決定論的で承認ゲート付きのワークフローについては、[Lobster](/ja-JP/tools/lobster) と比較してください。

## 関連項目

<CardGroup cols={2}>
  <Card title="Skills リファレンス" href="/ja-JP/tools/skills" icon="puzzle-piece">
    OpenProse のスキルパックが読み込まれる仕組みと、適用されるゲートについて説明します。
  </Card>
  <Card title="サブエージェント" href="/ja-JP/tools/subagents" icon="users">
    OpenClaw ネイティブのマルチエージェント調整レイヤーです。
  </Card>
  <Card title="テキスト読み上げ" href="/ja-JP/tools/tts" icon="volume-high">
    ワークフローに音声出力を追加します。
  </Card>
  <Card title="スラッシュコマンド" href="/ja-JP/tools/slash-commands" icon="terminal">
    /prose を含む、使用可能なすべてのチャットコマンドです。
  </Card>
</CardGroup>

公式サイト：[https://www.prose.md](https://www.prose.md)
