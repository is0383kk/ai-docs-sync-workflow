---
read_when:
    - スクリプトまたはコマンドラインからエージェントの実行を開始したい場合
    - チャットチャンネルにエージェントの返信をプログラムで配信する必要があります
summary: CLI からエージェントターンを実行し、必要に応じてチャンネルに返信を配信する
title: エージェント送信
x-i18n:
    generated_at: "2026-07-26T09:19:40Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: ad3da0feea102725ebb5555e0dd375ed6f3a0396d8ffd0ab916ced303201eabc
    source_path: tools/agent-send.md
    workflow: 16
---

`openclaw agent` は、受信チャットメッセージなしでコマンドラインから単一のエージェントターンを実行します。スクリプト化されたワークフロー、テスト、プログラムによる配信に使用します。フラグと動作の完全なリファレンス：
[エージェント CLI リファレンス](/ja-JP/cli/agent)。

## クイックスタート

<Steps>
  <Step title="シンプルなエージェントターンを実行する">
    ```bash
    openclaw agent --agent main --message "今日の天気は？"
    ```

    Gateway 経由でメッセージを送信し、応答を出力します。

  </Step>

  <Step title="ファイルから複数行のプロンプトを送信する">
    ```bash
    openclaw agent --agent ops --message-file ./task.md
    ```

    有効な UTF-8 ファイルをエージェントメッセージ本文として読み込みます。

  </Step>

  <Step title="特定のエージェントまたはセッションを指定する">
    ```bash
    # 特定のエージェントを指定
    openclaw agent --agent ops --message "ログを要約"

    # 電話番号を指定（セッションキーを生成）
    openclaw agent --to +15555550123 --message "ステータス更新"

    # 既存のセッションを再利用
    openclaw agent --session-id abc123 --message "タスクを続行"

    # 完全一致するセッションキーを指定
    openclaw agent --session-key agent:ops:incident-42 --message "ステータスを要約"
    ```

  </Step>

  <Step title="応答をチャンネルに配信する">
    ```bash
    # WhatsApp に配信（デフォルトチャンネル）
    openclaw agent --to +15555550123 --message "レポートの準備ができました" --deliver

    # Slack に配信
    openclaw agent --agent ops --message "レポートを生成" \
      --deliver --reply-channel slack --reply-to "#reports"
    ```

  </Step>
</Steps>

## フラグ

| フラグ                      | 説明                                                                 |
| --------------------------- | -------------------------------------------------------------------- |
| `--message <text>`          | 送信するインラインメッセージ                                         |
| `--message-file <path>`     | 有効な UTF-8 ファイルからメッセージを読み込む（最大 4 MiB）          |
| `--to <dest>`               | ターゲット（電話番号、チャット ID）からセッションキーを生成する     |
| `--session-key <key>`       | 明示的なセッションキーを使用する                                     |
| `--agent <id>`              | 設定済みのエージェントを指定する（その `main` セッションを使用） |
| `--session-id <id>`         | ID で既存のセッションを再利用する                                    |
| `--model <id>`              | この実行のモデルを上書きする（`provider/model` またはモデル ID）  |
| `--local`                   | ローカルの組み込みランタイムを強制する（Gateway をスキップ）        |
| `--deliver`                 | 応答をチャットチャンネルに送信する                                   |
| `--channel <name>`          | 配信チャンネル。`--agent` + `--to` と併用すると、DM スコープにも適用 |
| `--reply-to <target>`       | 配信先を上書きする                                                   |
| `--reply-channel <name>`    | 配信チャンネルを上書きする                                           |
| `--reply-account <id>`      | 配信アカウント ID を上書きする                                       |
| `--thinking <level>`        | 選択したモデルプロファイルの思考レベルを設定する                     |
| `--verbose <on\|full\|off>` | セッションの詳細出力レベルを永続化する（`full` ではツール出力もログに記録） |
| `--timeout <seconds>`       | エージェントのタイムアウトを上書きする（デフォルトは 600、または設定値） |
| `--json`                    | 構造化 JSON を出力する                                                |

## 動作

- デフォルトでは、CLI は **Gateway 経由**で動作します。現在のマシン上の組み込みランタイムを強制するには、`--local` を追加します。
- `--message` または `--message-file` のいずれか一方だけを指定してください。ファイルメッセージでは、オプションの UTF-8 BOM を削除した後も複数行の内容が保持されます。4 MiB を超えるファイルは、ディスパッチ前に拒否されます。
- 一時的なハンドシェイクの再試行後、Gateway のタイムアウトまたは接続の切断が発生すると、CLI は stderr にヒントを出力してコマンドを失敗させます。組み込みランタイムでターンを暗黙に再実行することはありません。Gateway は受け付けたターンを完了する場合があるため、再試行または `--local` を使用した再実行の前に、Gateway とセッションの状態を確認してください。
- セッションの選択：`--to` はセッションキーを生成します（グループ／チャンネルのターゲットでは分離が維持され、ダイレクトチャットは `main` に統合されます）。`--agent`、`--channel`、`--to` を併用すると、ルーティングはチャンネルの正規の受信者と `session.dmScope` に従います。安定した送信専用 ID には、エージェントのメインセッションから分離された、プロバイダー所有のセッションが使用されます。
- `--session-key` は明示的なキーを選択します。エージェント接頭辞付きのキーでは `agent:<agent-id>:<session-key>` を使用する必要があり、両方が指定された場合、`--agent` はそのエージェント ID と一致する必要があります。センチネルではない接頭辞なしのキーは、`--agent` が指定されている場合、そのスコープに設定されます。たとえば、`--agent ops --session-key incident-42` は `agent:ops:incident-42` にルーティングされます。`--agent` がない場合、センチネルではない接頭辞なしのキーは、設定済みのデフォルトエージェントのスコープに設定されます。リテラルの `global` と `unknown` は、`--agent` が指定されていない場合にのみスコープなしのままになります。
- `--reply-channel` と `--reply-account` は配信のみに影響します。
- 思考フラグと詳細出力フラグは、セッションストアに永続化されます。
- 出力：デフォルトではプレーンテキストです。構造化ペイロードとメタデータを出力するには `--json` を使用します。
- `--json --deliver` を使用すると、JSON に送信済み、抑制済み、部分的、失敗の各配信ステータスが含まれます。[JSON 配信ステータス](/ja-JP/cli/agent#json-delivery-status)を参照してください。

## 例

```bash
# JSON 出力を伴うシンプルなターン
openclaw agent --to +15555550123 --message "ログをトレース" --verbose on --json

# モデルを上書きするターン
openclaw agent --agent ops --model openai/gpt-5.4 --message "ログを要約"

# 思考レベルを指定するターン
openclaw agent --session-id 1234 --message "受信トレイを要約" --thinking medium

# ファイルからの複数行プロンプト
openclaw agent --agent ops --message-file ./task.md

# 完全一致するセッションキー
openclaw agent --session-key agent:ops:incident-42 --message "ステータスを要約"

# エージェントにスコープ設定されたレガシーキー
openclaw agent --agent ops --session-key incident-42 --message "ステータスを要約"

# セッションとは異なるチャンネルに配信
openclaw agent --agent ops --message "アラート" --deliver --reply-channel telegram --reply-to "@admin"
```

## 関連項目

<CardGroup cols={2}>
  <Card title="エージェント CLI リファレンス" href="/ja-JP/cli/agent" icon="terminal">
    `openclaw agent` のフラグとオプションの完全なリファレンス。
  </Card>
  <Card title="サブエージェント" href="/ja-JP/tools/subagents" icon="users">
    バックグラウンドでのサブエージェントの生成。
  </Card>
  <Card title="セッション" href="/ja-JP/concepts/session" icon="comments">
    セッションキーの仕組みと、`--to`、`--agent`、`--session-id` による解決方法。
  </Card>
  <Card title="スラッシュコマンド" href="/ja-JP/tools/slash-commands" icon="slash">
    エージェントセッション内で使用されるネイティブコマンドカタログ。
  </Card>
</CardGroup>
