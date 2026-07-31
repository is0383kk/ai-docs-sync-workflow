---
read_when:
    - エージェントランタイム、ワークスペースのブートストラップ、またはセッション動作の変更
summary: エージェントランタイム、ワークスペース契約、セッションのブートストラップ
title: エージェントランタイム
x-i18n:
    generated_at: "2026-07-26T10:10:19Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 4d3dd9c0c65e4ccd791a2a6131f1b7457c8cfee6da71502d93c355280e094390
    source_path: concepts/agent.md
    workflow: 16
---

OpenClaw には、1 つの**組み込みエージェントランタイム**が含まれています。これは、組み込みのエージェントループ、ツールの接続、プロンプトの組み立てで構成され、ターンを外部ハーネスプロセスに委任する方式とは異なります。構成された各エージェント（複数実行については[マルチエージェントルーティング](/ja-JP/concepts/multi-agent)を参照）には、独自のワークスペース、ブートストラップファイル、セッションストアがあります。このページでは、そのランタイム契約として、ワークスペースに必要な内容、挿入されるファイル、セッションがそれを使用してブートストラップされる仕組みを説明します。

## ワークスペース（必須）

各エージェントは、ツールとコンテキストの**唯一の**作業ディレクトリ（`cwd`）として、単一のワークスペースディレクトリ（`agents.defaults.workspace`、またはエージェントごとに `agents.entries.*.workspace`）を使用します。

推奨: `openclaw setup` を使用して、存在しない場合は `~/.openclaw/openclaw.json` を作成し、ワークスペースファイルを初期化します。

ワークスペースの完全なレイアウトとバックアップガイド: [エージェントワークスペース](/ja-JP/concepts/agent-workspace)

`agents.defaults.sandbox` が有効な場合、メイン以外のセッションでは、`agents.defaults.sandbox.workspaceRoot` 配下のセッションごとのワークスペースでこれを上書きできます（[Gateway の構成](/ja-JP/gateway/configuration)を参照）。

## ブートストラップファイル（挿入）

ワークスペース内で、OpenClaw は次のユーザー編集可能なファイルを想定します。

| ファイル           | 用途                                              |
| -------------- | ---------------------------------------------------- |
| `AGENTS.md`    | 操作指示と「メモリ」                    |
| `SOUL.md`      | ペルソナ、境界、トーン                            |
| `TOOLS.md`     | ユーザーが管理するツールのメモと規約           |
| `IDENTITY.md`  | エージェント名、雰囲気、絵文字                                |
| `USER.md`      | ユーザープロフィールと希望する呼び方                     |
| `HEARTBEAT.md` | Heartbeat 固有の指示                      |
| `BOOTSTRAP.md` | 初回実行時に一度だけ行う儀式（完了後に削除） |
| `MEMORY.md`    | 存在する場合のルート長期メモリファイル               |

新しいセッションの最初のターンで、OpenClaw はこれらのファイルの内容をシステムプロンプトのプロジェクトコンテキストに挿入します。`MEMORY.md` は、ワークスペースのルートに存在する場合にのみ挿入されます。

空のファイルはスキップされます。プロンプトを簡潔に保つため、大きなファイルはトリミングおよび切り詰められ、マーカーが付加されます（完全な内容についてはファイルを参照してください）。ファイルが存在しない場合（`MEMORY.md` を除く）は、代わりに「ファイルが見つかりません」というマーカー行が 1 行挿入されます。`openclaw setup` は、そのファイル用の安全なデフォルトテンプレートを作成します。

`BOOTSTRAP.md` は、**完全に新しいワークスペース**（ほかのブートストラップファイルが存在しない場合）にのみ作成されます。保留中の間、OpenClaw はこれをプロジェクトコンテキストに保持し、ユーザーメッセージへコピーする代わりに、最初の儀式用のブートストラップガイダンスをシステムプロンプトへ追加します。儀式の完了後に削除すると、その後の再起動時には再作成されません。

ワークスペースが一度確認されると、OpenClaw はそのセットアップ状態と証明を共有 SQLite データベースの `~/.openclaw/state/openclaw.sqlite` に保存します。最近証明されたワークスペースが消失または消去された場合、起動時に `BOOTSTRAP.md` が暗黙に再生成されることはありません。ワークスペースを復元するか、完全なオンボーディングリセットを使用して、ワークスペースとそのデータベース状態をまとめて消去してください。

以前のリリースでは、ワークスペース JSON と `.attested` サイドカーファイルが使用されていました。ランタイムはこれらのファイルを読み取りません。`openclaw doctor --fix` を実行して検証し、状態を SQLite にインポートして、インポートされた行の検証後に各ソースを削除してください。

ブートストラップファイルの作成を完全に無効化するには（事前に準備されたワークスペース向け）、次を設定します。

```json5
{ agents: { defaults: { skipBootstrap: true } } }
```

## 組み込みツール

コアツール（read/exec/edit/write および関連するシステムツール）は、ツールポリシーの制約に従って常に利用できます。`apply_patch` は OpenAI モデルでデフォルトで有効になっており、`tools.exec.applyPatch`（`enabled`、`workspaceOnly`、`allowModels`）によって制御されます。`TOOLS.md` は、存在するツールを制御するものでは**ありません**。これは、ツールをどのように使用させたいかについてのガイダンスです。

## Skills

OpenClaw は、次の場所から Skills を読み込みます（優先順位の高い順）。

- ワークスペース: `<workspace>/skills`
- プロジェクトエージェント Skills: `<workspace>/.agents/skills`
- 個人エージェント Skills: `~/.agents/skills`
- 管理対象/ローカル: `~/.openclaw/skills`
- バンドル済み（インストールに同梱）
- 追加の Skills フォルダー: `skills.load.extraDirs`

Skills のルートには、`<workspace>/skills/personal/foo/SKILL.md` のようなグループ化されたフォルダーを含めることができます。その場合でも、Skills はフラットな frontmatter 名（例: `foo`）で公開されます。

Skills は構成や環境変数によって制限できます（[Gateway の構成](/ja-JP/gateway/configuration)の `skills` を参照）。

## ランタイムの境界

組み込みエージェントランタイムは OpenClaw が所有します。モデル検出、ツールの接続、プロンプトの組み立て、セッション管理、チャネル配信は、統合された 1 つのランタイムサーフェスを共有します。

## セッション

セッション行は、エージェントごとの SQLite データベースに保存されます。

- `~/.openclaw/agents/<agentId>/agent/openclaw-agent.sqlite`

トランスクリプト JSONL ファイルは、従来の移行入力、削除済みまたはリセット済みのアーカイブ、インポート、エクスポート、サポート用成果物として、引き続き `~/.openclaw/agents/<agentId>/sessions/` 配下に配置できます。アクティブなエージェント履歴は、セッション行とともに SQLite に保存されます。セッション ID は安定しており、OpenClaw によって選択されます。OpenClaw は、ほかのツールのセッションフォルダーを読み取りません。

## ストリーミング中のステアリング

実行中に到着した受信プロンプトは、デフォルトで現在の実行へステアリングされます。ステアリングは、**現在のアシスタントターンがツール呼び出しの実行を完了した後**、次の LLM 呼び出しの前に配信されます。現在のアシスタントメッセージに残っているツール呼び出しがスキップされることはなくなりました。

`/queue steer` は、アクティブな実行に対するデフォルトの動作です。`/queue followup` と `/queue collect` では、メッセージはステアリングされず、後続のターンまで待機します。`/queue interrupt` は、代わりにアクティブな実行を中止します。キューと境界の動作については、[キュー](/ja-JP/concepts/queue)および[ステアリングキュー](/ja-JP/concepts/queue-steering)を参照してください。

ブロックストリーミングは、完了したアシスタントブロックを完了直後に送信します。これは**デフォルトでは無効**です（`agents.defaults.blockStreamingDefault: "off"`）。
境界は `agents.defaults.blockStreamingBreak`（`text_end` と `message_end`。デフォルトは `text_end`）で調整します。
ソフトブロックのチャンク分割は `agents.defaults.blockStreamingChunk` で制御します（デフォルトは 800～1200 文字。段落区切り、改行、文の順で優先されます）。
ストリーミングされたチャンクを `agents.defaults.blockStreamingCoalesce` で結合し、単一行の連続送信を減らします（送信前にアイドル時間に基づいて結合）。Telegram 以外のチャネルでブロック応答を有効にするには、明示的に `*.streaming.block.enabled: true` を指定する必要があります（QQ Bot は、`channels.qqbot.streaming.mode` が `"off"` でない限り、ブロック応答をストリーミングします）。
詳細なツール概要はツールの開始時に出力されます（デバウンスなし）。Control UI は、利用可能な場合、エージェントイベントを介してツール出力をストリーミングします。
詳細: [ストリーミングとチャンク分割](/ja-JP/concepts/streaming)。

## モデル参照

構成内のモデル参照（例: `agents.defaults.model` と `agents.defaults.models`）は、**最初の** `/` で分割して解析されます。

- モデルを構成する際は、`provider/model` を使用します。
- モデル ID 自体に `/` が含まれる場合（OpenRouter 形式）は、プロバイダープレフィックスを含めます（例: `openrouter/moonshotai/kimi-k2`）。
- プロバイダーを省略すると、OpenClaw はまずエイリアスを試し、次にその正確なモデル ID に一致する一意の構成済みプロバイダーを試し、最後に構成済みのデフォルトプロバイダーへフォールバックします。そのプロバイダーが構成済みのデフォルトモデルを提供しなくなった場合、OpenClaw は削除済みプロバイダーの古いデフォルトをエラーとして表示するのではなく、最初に構成されたプロバイダー/モデルへフォールバックします。

## 構成（最小）

最低限、次を設定します。

- `agents.defaults.workspace`
- `channels.whatsapp.allowFrom`（強く推奨）

## 関連項目

- [エージェントワークスペース](/ja-JP/concepts/agent-workspace)
- [マルチエージェントルーティング](/ja-JP/concepts/multi-agent)
- [セッション管理](/ja-JP/concepts/session)
- [グループチャット](/ja-JP/channels/group-messages)
