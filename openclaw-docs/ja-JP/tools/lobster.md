---
read_when:
    - 明示的な承認を伴う、決定論的な複数ステップのワークフローが必要な場合
    - 以前の手順を再実行せずにワークフローを再開する必要がある
summary: 再開可能な承認ゲートを備えた OpenClaw 用の型付きワークフローランタイム。
title: Lobster
x-i18n:
    generated_at: "2026-07-26T10:07:02Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 85b7900f86bfedc9d73fcc91c3d0dac37b81f7413b1e68c54dd8a797b70f79fc
    source_path: tools/lobster.md
    workflow: 16
---

Lobster は、明示的な承認チェックポイントと再開トークンを使用して、複数ステップのツールパイプラインを単一の決定論的なツール呼び出しとして実行します。
これは、デタッチされたバックグラウンド作業の一段上に位置します。多数のデタッチされたタスクにまたがるフローのオーケストレーションについては、
[Task Flow](/ja-JP/automation/taskflow)（`openclaw tasks flow`）を参照してください。タスクの
アクティビティ台帳については、[バックグラウンドタスク](/ja-JP/automation/tasks)を参照してください。

## 理由

Lobster を使用しない場合、複数ステップのジョブには多数の往復ツール呼び出しが必要となり、
モデルがすべてのステップをオーケストレーションします。Lobster は、そのオーケストレーションを型付き
ランタイムに移します。

- **多数ではなく 1 回の呼び出し**：単一の Lobster ツール呼び出しが、パイプライン全体の構造化された
  結果を返します。
- **組み込みの承認**：副作用（送信、投稿、削除）が発生する場合、明示的に承認されるまでワークフローを
  停止します。
- **再開可能**：停止したワークフローはトークンを返します。承認して再開すれば、
  それ以前のステップを再実行する必要はありません。

Lobster は汎用スクリプト言語ではなく、小規模で制約された DSL です。
承認と再開は永続的な組み込みプリミティブであり、パイプラインはデータであるため（ログ記録、差分確認、再実行、レビューが容易）、
小さな文法によって「創造的な」コードパスが制限され、現実的な検証が可能になります。また、
タイムアウト、出力上限、サンドボックスチェック、許可リストは、各スクリプトではなく
ランタイムによって適用されます。各ステップから任意の CLI やスクリプトを呼び出すことは引き続き可能です。
より高機能なオーサリング言語が必要な場合は、他のツールから `.lobster` ファイルを生成してください。

Lobster を使用しない場合、定期的なメールのトリアージは次のようになります。

```text
ユーザー：「メールを確認して返信の下書きを作成して」
→ OpenClaw が gmail.list を呼び出す
→ LLM が要約する
→ ユーザー：「#2 と #5 への返信を下書きして」
→ LLM が下書きを作成する
→ ユーザー：「#2 を送信して」
→ OpenClaw が gmail.send を呼び出す
（毎日繰り返され、何をトリアージしたかは記憶されない）
```

Lobster を使用すると、同じジョブが、承認のために停止して再開する 1 回の呼び出しになります。

```json
{ "action": "run", "pipeline": "email.triage --limit 20", "timeoutMs": 30000 }
```

```json
{
  "ok": true,
  "status": "needs_approval",
  "output": [{ "summary": "返信が必要なものが 5 件、対応が必要なものが 2 件" }],
  "requiresApproval": {
    "type": "approval_request",
    "prompt": "2 件の返信下書きを送信しますか？",
    "items": [],
    "resumeToken": "..."
  }
}
```

## 仕組み

OpenClaw は、バンドルされた
`@clawdbot/lobster` パッケージを組み込みランナーとして使用し、Lobster ワークフローを**プロセス内**で実行します。
外部の `lobster`
サブプロセスは起動されず、ツール呼び出しは JSON エンベロープを直接返します。
パイプラインが承認のために停止した場合、エンベロープには再開トークン（または短い
承認 ID）が含まれるため、後で続行できます。

## 有効化

Lobster は**任意**の Plugin ツールであり、デフォルトでは有効になっていません。バンドル済みであるため、
個別のインストール手順は不要です。ツールを許可するだけです。

```json
{
  "tools": {
    "alsoAllow": ["lobster"]
  }
}
```

またはエージェントごとに設定します。

```json
{
  "agents": {
    "list": [
      {
        "id": "main",
        "tools": {
          "alsoAllow": ["lobster"]
        }
      }
    ]
  }
}
```

<Note>
`alsoAllow` は、他のコアツールを制限せずに、アクティブなツールプロファイルへ `lobster` を追加します。
制限的な許可リストモードにする場合にのみ、代わりに `tools.allow` を使用してください。
</Note>

サンドボックス化されたツールコンテキストでは、このツールは完全に無効になります。

開発用または外部パイプライン用にスタンドアロンの Lobster CLI が必要な場合
（組み込み Gateway ランナーの外部で使用する場合）は、
[Lobster リポジトリ](https://github.com/openclaw/lobster)からインストールし、`lobster` を
`PATH` に配置してください。

## パターン：小さな CLI + JSON パイプ + 承認

JSON を扱う小さなコマンドを作成し、それらを 1 回の Lobster 呼び出しに連結します。
（以下のコマンド名は例です。独自のものに置き換えてください。）

```bash
inbox list --json
inbox categorize --json
inbox apply --json
```

```json
{
  "action": "run",
  "pipeline": "exec --json --shell 'inbox list --json' | exec --stdin json --shell 'inbox categorize --json' | exec --stdin json --shell 'inbox apply --json' | approve --preview-from-stdin --limit 5 --prompt '変更を適用しますか？'",
  "timeoutMs": 30000
}
```

パイプラインが承認を要求した場合は、トークンを使用して再開します。

```json
{
  "action": "resume",
  "token": "<resumeToken>",
  "approve": true
}
```

例：入力項目をツール呼び出しにマッピングします。

```bash
gog.gmail.search --query 'newer_than:1d' \
  | openclaw.invoke --tool message --action send --each --item-key message --args-json '{"provider":"telegram","to":"..."}'
```

## JSON のみの LLM ステップ（llm-task）

ワークフロー内で**構造化された LLM ステップ**を使用するには、任意の
`llm-task` Plugin ツールを有効にし、Lobster から呼び出します。

```json
{
  "plugins": {
    "entries": {
      "llm-task": { "enabled": true }
    }
  },
  "agents": {
    "list": [
      {
        "id": "main",
        "tools": { "alsoAllow": ["llm-task"] }
      }
    ]
  }
}
```

### 重要な制限：組み込み Lobster と `openclaw.invoke`

バンドルされた Lobster Plugin は、Gateway 内の**プロセス内**でワークフローを実行します。
この組み込みモードでは、ネストされた OpenClaw CLI ツール呼び出しのための
Gateway URL／認証コンテキストが `openclaw.invoke` に自動的に継承されることは**ありません**。

そのため、次のパターンは**現在、組み込みランナーでは信頼できません**。

```lobster
openclaw.invoke --tool llm-task --action json --args-json '{ ... }'
```

以下の例は、正しい Gateway／認証コンテキストが `openclaw.invoke` にすでに設定されている環境で、
**スタンドアロン Lobster CLI** を実行する場合にのみ使用してください。

```lobster
openclaw.invoke --tool llm-task --action json --args-json '{
  "prompt": "入力されたメールに基づいて、意図と返信下書きを返してください。",
  "thinking": "low",
  "input": { "subject": "こんにちは", "body": "手伝ってもらえますか？" },
  "schema": {
    "type": "object",
    "properties": {
      "intent": { "type": "string" },
      "draft": { "type": "string" }
    },
    "required": ["intent", "draft"],
    "additionalProperties": false
  }
}'
```

現在、組み込み Lobster Plugin を使用している場合は、次のいずれかを推奨します。

- Lobster の外部で `llm-task` ツールを直接呼び出す、または
- サポートされる組み込みブリッジが追加されるまで、Lobster パイプライン内で `openclaw.invoke` 以外の
  ステップを使用する。

詳細と設定オプションについては、[LLM タスク](/ja-JP/tools/llm-task)を参照してください。

## ワークフローファイル（.lobster）

Lobster は、`name`、`args`、`steps`、`env`、
`condition`、`approval` フィールドを持つ YAML／JSON ワークフローファイルを実行できます。ツール
呼び出しで `pipeline` にファイルパスを設定します。

```yaml
name: inbox-triage
args:
  tag:
    default: "family"
steps:
  - id: collect
    command: inbox list --json
  - id: categorize
    command: inbox categorize --json
    stdin: $collect.stdout
  - id: approve
    command: inbox apply --approve
    stdin: $categorize.stdout
    approval: required
  - id: execute
    command: inbox apply --execute
    stdin: $categorize.stdout
    condition: $approve.approved
```

注：

- `stdin: $step.stdout` と `stdin: $step.json` は、前のステップの出力を渡します。
- `condition`（または `when`）は、`$step.approved` に基づいてステップの実行を制御できます。

### 注入される環境変数

各ステップのシェルは、親環境に加えて次の Lobster 注入変数を継承します。
これにより、コマンド文字列に未加工の値を埋め込まずに、解決済みのワークフロー引数をコマンドから参照できます。

- `LOBSTER_ARG_<NAME>` - ワークフロー引数ごとに 1 つ。名前は大文字に変換され、
  英数字以外の文字が連続する部分はそれぞれ `_` にまとめられます。そのため、引数 `user-id` は
  `LOBSTER_ARG_USER_ID` になります。
- `LOBSTER_ARGS_JSON` - 解決済みのすべての引数を単一の JSON 文字列として格納します。

注入される変数はこれがすべてです。`LOBSTER_STEP_<id>_STDOUT` や `LOBSTER_STEP_<id>_JSON_<field>` のような
ステップごとの出力変数は**ありません**。シェルはこれらの名前を未設定として扱うため、
パラメーター展開のデフォルト値によってエラーが隠れる場合があります。
前のステップの出力は、代わりにステップ参照（`$step.stdout`、
`$step.json`、または `$step.json.<field>`）を `stdin:`、`env:`、または `condition:`
の値で使用して読み取ってください。（`LOBSTER_STATE_DIR` は状態
ディレクトリ用の独立したランタイム設定であり、実行ごとの引数ではありません。）

## ツールパラメーター

### `run`

```json
{
  "action": "run",
  "pipeline": "gog.gmail.search --query 'newer_than:1d' | email.triage",
  "cwd": "workspace",
  "timeoutMs": 30000,
  "maxStdoutBytes": 512000
}
```

引数を指定してワークフローファイルを実行します。

```json
{
  "action": "run",
  "pipeline": "/path/to/inbox-triage.lobster",
  "argsJson": "{\"tag\":\"family\"}"
}
```

| フィールド            | デフォルト     | 注記                                                                                                        |
| ---------------- | ----------- | ------------------------------------------------------------------------------------------------------------ |
| `pipeline`       | 必須    | インラインパイプライン文字列、またはワークフローファイルを示す `.lobster`/`.yaml`/`.yml`/`.json` で終わるパス。           |
| `cwd`            | Gateway の cwd | 相対作業ディレクトリ。Gateway の作業ディレクトリ内に解決される必要があります（絶対パスは拒否されます）。 |
| `timeoutMs`      | `20000`     | 超過した場合は実行を中止します。                                                                                  |
| `maxStdoutBytes` | `512000`    | 取得された stdout または stderr がこのサイズを超えた場合、実行を中止します。                                               |
| `argsJson`       | -           | ワークフローファイル用の引数を表す JSON 文字列（インラインパイプラインでは無視されます）。                                      |

### `resume`

```json
{
  "action": "resume",
  "token": "<resumeToken>",
  "approve": true
}
```

`resume` は、`token`（`requiresApproval` の完全な再開トークン）または
`approvalId`（同じオブジェクトの短い ID）のいずれかを受け入れます。停止した
実行が返した方を使用してください。`approve` は必須です。

### 管理対象 Task Flow モード

`run` に `flowControllerId` と `flowGoal` を渡す（または `resume` に
`flowId` と `flowExpectedRevision` を渡す）と、裸のエンベロープを返す代わりに、Plugin
ランタイムの管理対象 [Task Flow](/ja-JP/automation/taskflow) API を介して呼び出しが処理されます。
OpenClaw は永続的なフローレコードを作成または再開し、Lobster エンベロープを適用して
（承認時は `waiting`、完了時は `succeeded`/`failed`）、
`{ ok, envelope, flow, mutation }` を返します。このモードには、バインドされた Task Flow ランタイムが必要です。
通常のアドホックなエージェント利用ではなく、Gateway の再起動後も永続的なフロー状態を必要とする
Plugin／コントローラーコードを対象としています。

## 出力エンベロープ

Lobster は、次の 3 つのステータスのいずれかを持つ JSON エンベロープを返します。

- `ok` - 正常に完了
- `needs_approval` - 一時停止。`requiresApproval` には `resumeToken` と短い
  `approvalId` が含まれ、どちらを使用しても実行を再開できます
- `cancelled` - 明示的に拒否またはキャンセル

ツールは、エンベロープを `content`（整形済み JSON）と `details`
（未加工オブジェクト）の両方で公開します。

## 承認

`requiresApproval` が存在する場合は、プロンプトを確認して判断します。

- `approve: true` - 再開して副作用の処理を続行
- `approve: false` - キャンセルしてワークフローを終了

カスタムの jq／heredoc 処理を使用せずに、承認リクエストへ JSON プレビューを添付するには、
`approve --preview-from-stdin --limit N` を使用します。再開状態は、Lobster 状態ディレクトリ（デフォルトでは
`~/.lobster/state`、`LOBSTER_STATE_DIR` で変更可能）配下に小さな JSON ファイルとして保存されます。
トークン自体にはパイプライン状態全体ではなく、その状態へのポインターのみがエンコードされます。

## OpenProse

OpenProse は Lobster と相性よく連携します。`/prose` を使用して複数エージェントによる
準備をオーケストレーションし、決定論的な承認処理には Lobster パイプラインを実行します。Prose
プログラムで Lobster が必要な場合は、`tools.subagents.tools` を介してサブエージェントに
`lobster` ツールを許可します。[OpenProse](/ja-JP/prose)を参照してください。

## 安全性

- **ローカルのプロセス内のみ** - ワークフローは Gateway プロセス内で実行され、Plugin 自体からの
  ネットワーク呼び出しはありません。
- **シークレットなし** - Lobster は OAuth を管理せず、それを行う OpenClaw ツールを
  呼び出します。
- **サンドボックス対応** - ツールのコンテキストがサンドボックス化されている場合は無効になります。
- **堅牢化** - 組み込みランナーによってタイムアウトと出力上限が適用されます。

## トラブルシューティング

| エラー                                                         | 原因 / 修正方法                                                                      |
| ------------------------------------------------------------- | -------------------------------------------------------------------------------- |
| `lobster runtime timed out`                                   | パイプラインが `timeoutMs` を超過しました。値を増やすか、パイプラインを分割してください。                |
| `lobster stdout exceeded maxStdoutBytes`（または `stderr`）        | 取得された出力が上限を超過しました。`maxStdoutBytes` を増やすか、出力を減らしてください。       |
| `run --args-json must be valid JSON`                          | `argsJson`（ワークフローファイルの実行）の解析に失敗しました。JSON 文字列を修正してください。            |
| `lobster runtime failed`（または別の `runtime_error` メッセージ） | 組み込みランタイムがエラーエンベロープを返しました。詳細については Gateway のログを確認してください。 |

## 詳細情報

- [Plugin](/ja-JP/tools/plugin)
- [Plugin ツールの作成](/ja-JP/plugins/building-plugins#registering-agent-tools)

## ケーススタディ：コミュニティのワークフロー

公開されている例の 1 つに、3 つの Markdown Vault（個人用、パートナー用、共有用）を管理する
「セカンドブレイン」CLI と Lobster パイプラインがあります。CLI は統計、
受信トレイの一覧、古くなった項目のスキャン結果を JSON として出力し、Lobster はそれらのコマンドを
`weekly-review`、`inbox-triage`、`memory-consolidation`、
`shared-task-sync` のようなワークフローに連結します。それぞれに承認ゲートがあります。AI が利用可能な場合は
判断（分類）を処理し、利用できない場合は決定論的なルールに
フォールバックします。

- スレッド：[https://x.com/plattenschieber/status/2014508656335770033](https://x.com/plattenschieber/status/2014508656335770033)
- リポジトリ：[https://github.com/bloomedai/brain-cli](https://github.com/bloomedai/brain-cli)

## 関連項目

- [自動化](/ja-JP/automation) - すべての自動化メカニズム
- [ツールの概要](/ja-JP/tools) - 利用可能なすべてのエージェントツール
