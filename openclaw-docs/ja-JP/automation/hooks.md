---
read_when:
    - /new、/reset、/stop、およびエージェントのライフサイクルイベントに対するイベント駆動型自動化が必要な場合
    - フックを作成、インストール、またはデバッグしたい場合
summary: フック：コマンドとライフサイクルイベントのためのイベント駆動型自動化
title: フック
x-i18n:
    generated_at: "2026-07-26T09:12:08Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 039a55cca60e0005d7b9c4d950a86aceb6e7c29d5768108b34011bfc21c85be6
    source_path: automation/hooks.md
    workflow: 16
---

Hook は、エージェントイベントの発火時に Gateway 内で実行される小さなスクリプトです。対象には、`/new`、`/reset`、`/stop` などのコマンド、セッションの Compaction、Gateway のライフサイクル、メッセージフローがあります。Hook はディレクトリから検出され、`openclaw hooks` で管理されます。Gateway が内部 Hook を読み込むのは、Hook を有効にするか、Hook エントリ、Hook パック、レガシーハンドラー、追加の Hook ディレクトリのいずれかを少なくとも 1 つ設定した後だけです。

OpenClaw には 2 種類の Hook があります。

- **内部 Hook**（このページ）：エージェントイベントの発火時に Gateway 内で実行されます。
- **Webhook**：他のシステムから OpenClaw の処理をトリガーできる外部 HTTP エンドポイントです。[Webhook](/ja-JP/automation/cron-jobs#webhooks)を参照してください。

Hook は Plugin 内にバンドルすることもできます。`openclaw hooks list` には、スタンドアロン Hook と Plugin 管理の Hook（`plugin:<id>` と表示）の両方が表示されます。

## 適切なサーフェスを選ぶ

OpenClaw には、見た目は似ていても異なる問題を解決する複数の拡張サーフェスがあります。

| やりたいこと                                                                                                     | 使用するもの                                | 理由                                                                                           |
| --------------------------------------------------------------------------------------------------------------------- | ------------------------------------- | --------------------------------------------------------------------------------------------- |
| `/new` 時にスナップショットを保存する、`/reset` をログに記録する、`message:sent` 後に外部 API を呼び出す、または大まかな運用者向け自動化を追加する | 内部 Hook（`HOOK.md`、このページ） | ファイルベースの Hook は、運用者が管理する副作用とコマンド／ライフサイクルの自動化を目的としています |
| プロンプトを書き換える、ツールをブロックする、送信メッセージをキャンセルする、または順序付きのミドルウェア／ポリシーを追加する                              | `api.on(...)` を介した型付き Plugin Hook  | 型付き Hook には、明示的な契約、優先順位、マージ規則、ブロック／キャンセルのセマンティクスがあります      |
| テレメトリ専用のエクスポートまたは可観測性を追加する                                                                            | 診断イベント                     | 可観測性は独立したイベントバスであり、ポリシー Hook のサーフェスではありません                              |

小規模なインストール済みインテグレーションのように動作する自動化が必要な場合は、内部 Hook を使用します。ランタイムのライフサイクル制御が必要な場合は、型付き Plugin Hook を使用します。

## クイックスタート

```bash
# 利用可能な Hook を一覧表示
openclaw hooks list

# Hook を有効化
openclaw hooks enable session-memory

# Hook の状態を確認
openclaw hooks check

# 詳細情報を取得
openclaw hooks info session-memory
```

## イベントの種類

Hook は、この表の特定のキー、または単独のファミリー名
（`command`、`session`、`agent`、`gateway`、`message`）をサブスクライブして、そのファミリー内のすべてのアクションを受信します。OpenClaw コアはそれ以外を発行しないため、他の名前はほぼ常に入力ミスであり、Hook は通知なしに動作しないままになります（カスタムイベントを発行する Plugin がある場合にのみ発火する可能性があります）。Hook ローダーは、そのような名前（たとえば `command:nwe`）について警告をログに記録し、`openclaw hooks info <name>` もそれらを検出するため、まったく実行されない Hook も診断できます。

| イベント                    | 発火するタイミング                                              |
| ------------------------ | ---------------------------------------------------------- |
| `command:new`            | `/new` コマンドが発行されたとき                                      |
| `command:reset`          | `/reset` コマンドが発行されたとき                                    |
| `command:stop`           | `/stop` コマンドが発行されたとき                                     |
| `command`                | 任意のコマンドイベント（汎用リスナー）                       |
| `session:compact:before` | Compaction が履歴を要約する前                       |
| `session:compact:after`  | Compaction の完了後                                 |
| `session:patch`          | セッションプロパティが変更されたとき                       |
| `agent:bootstrap`        | ワークスペースのブートストラップファイルが挿入される前              |
| `gateway:startup`        | チャンネルが起動し、Hook が読み込まれた後                  |
| `gateway:shutdown`       | Gateway のシャットダウン開始時                               |
| `gateway:pre-restart`    | 予定された Gateway 再起動の前                         |
| `message:received`       | 任意のチャンネルからメッセージを受信したとき                           |
| `message:transcribed`    | 音声文字起こしの完了後                        |
| `message:preprocessed`   | メディアとリンクの前処理が完了したか、スキップされた後 |
| `message:sent`           | 送信処理が試行されたとき（結果は `context.success` に格納） |

## Hook の作成

### Hook の構造

各 Hook は、2 つのファイルを含むディレクトリです。

```text
my-hook/
├── HOOK.md          # メタデータ + ドキュメント
└── handler.ts       # ハンドラー実装
```

ハンドラーファイルには、`handler.ts`、`handler.js`、`index.ts`、または `index.js` を使用できます。

### HOOK.md の形式

```markdown
---
name: my-hook
description: "この Hook の動作に関する短い説明"
metadata:
  { "openclaw": { "emoji": "🔗", "events": ["command:new"], "requires": { "bins": ["node"] } } }
---

# My Hook

詳細なドキュメントをここに記載します。
```

**メタデータフィールド**（`metadata.openclaw`）：

| フィールド      | 説明                                          |
| ---------- | ---------------------------------------------------- |
| `emoji`    | CLI に表示する絵文字                                |
| `events`   | リッスンするイベントの配列                        |
| `export`   | 使用する名前付きエクスポート（デフォルトは `"default"`）        |
| `os`       | 必要なプラットフォーム（例：`["darwin", "linux"]`）     |
| `requires` | 必要な `bins`、`anyBins`、`env`、または `config` のパス |
| `always`   | 適格性チェックを省略するかどうか（真偽値）                  |
| `hookKey`  | 設定キーの上書き（デフォルトは Hook 名）      |
| `homepage` | `openclaw hooks info` に表示されるドキュメント URL              |
| `install`  | インストール方法                                 |

### ハンドラーの実装

```typescript
const handler = async (event) => {
  if (event.type !== "command" || event.action !== "new") {
    return;
  }

  console.log(`[my-hook] 新しいコマンドがトリガーされました`);
  // ここにロジックを記述

  // 返信可能なサーフェスでは、必要に応じて返信を送信
  event.messages.push("Hook が実行されました！");
};

export default handler;
```

各イベントには、`type`、`action`、`sessionKey`、`timestamp`、`messages`、および `context`（イベント固有のデータ）が含まれます。エージェントおよびツールの Hook 用の型付き Plugin Hook コンテキストには、読み取り専用で W3C 互換の診断トレースコンテキストである `trace` も含まれる場合があり、Plugin は OTEL の相関付けのために構造化ログへ渡すことができます。

`event.messages` に追加された文字列がチャットへ返されるのは、
`command:new` と `command:reset`（元の会話への返信としてルーティング）、
および `session:compact:before` / `session:compact:after`
（Compaction のステータス通知として送信）の場合だけです。
`command:stop`、`message:*`、`agent:bootstrap`、`session:patch`、
`gateway:*` を含むその他すべてのイベントは、追加されたメッセージを無視します。

### イベントコンテキストの要点

**コマンドイベント**（`command:new`、`command:reset`）：`context.sessionEntry`、`context.previousSessionEntry`、`context.commandSource`、`context.senderId`、`context.workspaceDir`、`context.cfg`。

**コマンドイベント**（`command:stop`）：`context.sessionEntry`、`context.sessionId`、`context.commandSource`、`context.senderId`。

**メッセージイベント**（`message:received`）：`context.from`、`context.content`、`context.channelId`、`context.media`（順序付けされた段階的な添付ファイル情報）、リモートメディアがまだローカルにステージングされていない場合の `context.originalMedia` と `context.mediaStagingPending`、および `context.metadata`（`senderId`、`senderName`、`guildId` を含むプロバイダー固有のデータ）。`context.content` は、コマンド形式のメッセージでは空白でないコマンド本文を優先し、次に受信時の未加工本文、汎用本文の順でフォールバックします。スレッド履歴やリンク要約など、エージェント専用の拡張情報は含まれません。`metadata` 内のレガシーメディアエイリアスは非推奨です。

**メッセージイベント**（`message:sent`）：`context.to`、`context.content`、`context.success`、`context.channelId`、および送信に失敗した場合の `context.error`。

**メッセージイベント**（`message:transcribed`）：`context.transcript`、`context.from`、`context.channelId`、および `context.media`。`context.mediaPath` と `context.mediaType` は、最初の情報に対する非推奨のエイリアスとして残されています。

**メッセージイベント**（`message:preprocessed`）：`context.bodyForAgent`（最終的に拡張された本文）、`context.from`、`context.channelId`。

**ブートストラップイベント**（`agent:bootstrap`）：`context.bootstrapFiles`（変更可能な配列）、`context.agentId`。

**セッションパッチイベント**（`session:patch`）：`context.sessionEntry`、`context.patch`（変更されたフィールドのみ）、`context.cfg`。パッチイベントをトリガーできるのは特権クライアントだけです。コンテキストはクローンであるため、ハンドラーが実際のセッションエントリを変更することはできません。

**Compaction イベント**：`session:compact:before` には `messageCount`、`tokenCount` が含まれます。`session:compact:after` には、さらに `compactedCount`、`summaryLength`、`tokensBefore`、`tokensAfter` が追加されます。

`command:stop` は、ユーザーによる `/stop` の発行を監視します。これはキャンセル／コマンドのライフサイクルであり、エージェントの最終処理を制御するゲートではありません。自然な最終回答を検査し、エージェントにもう一度処理を行わせる必要がある Plugin は、代わりに型付き Plugin Hook `before_agent_finalize` を使用してください。[Plugin Hook](/ja-JP/plugins/hooks)を参照してください。

**Gateway ライフサイクルイベント**：`gateway:shutdown` には `reason` と `restartExpectedMs` が含まれ、Gateway のシャットダウン開始時に発火します。`gateway:pre-restart` には同じコンテキストが含まれますが、シャットダウンが予定された再起動の一部であり、有限の `restartExpectedMs` 値が指定された場合にのみ発火します。シャットダウン中の各ライフサイクル Hook の待機はベストエフォートかつ制限時間付きであり、ハンドラーが停止してもシャットダウンは続行されます。デフォルトの待機時間は、`gateway:shutdown` では 5 秒、`gateway:pre-restart` では 10 秒です。

チャンネルがまだ利用可能な間に短い再起動通知を送るには、`gateway:pre-restart` を使用します。

```typescript
import { execFile } from "node:child_process";
import { promisify } from "node:util";

const execFileAsync = promisify(execFile);

export default async function handler(event) {
  if (event.type !== "gateway" || event.action !== "pre-restart") {
    return;
  }

  const restartInSeconds = Math.ceil(event.context.restartExpectedMs / 1000);
  await execFileAsync("openclaw", [
    "system",
    "event",
    "--mode",
    "now",
    "--text",
    `Gateway は約${restartInSeconds}秒後に再起動します（${event.context.reason}）。今すぐチェックポイントを作成してください。`,
  ]);
}
```

`gateway:shutdown`（または `gateway:pre-restart`）イベントと、残りのシャットダウンシーケンスの間に、Gateway はプロセス停止時にまだアクティブだった各セッションに対して、型付き `session_end` Plugin Hook も発火します。イベントの `reason` は、通常の SIGTERM/SIGINT 停止では `shutdown`、予定された再起動の一部として終了がスケジュールされた場合は `restart` です。このドレイン処理には制限時間があるため、遅い `session_end` ハンドラーがプロセス終了を妨げることはありません。また、置換／リセット／削除／Compaction によってすでに終了処理されたセッションは、二重発火を避けるためスキップされます。

## Hook の検出

Hook は 4 つのソースから検出されます。

1. **同梱フック**: OpenClaw に同梱
2. **Plugin フック**: インストール済み Plugin に同梱。同名の同梱フックを上書き可能
3. **管理対象フック**: `~/.openclaw/hooks/`（ユーザーがインストールし、ワークスペース間で共有）。同梱フックと Plugin フックを上書き可能。`hooks.internal.load.extraDirs` で指定した追加ディレクトリにも同じ優先順位が適用されます。
4. **ワークスペースフック**: `<workspace>/hooks/`（エージェントごと。明示的に有効化されるまでデフォルトでは無効）

ワークスペースフックは新しいフック名を追加できますが、同名の同梱フック、管理対象フック、または Plugin 提供フックを上書きすることはできません。

内部フックが設定されるまで、Gateway は起動時の内部フック検出をスキップします。`openclaw hooks enable <name>` で同梱フックまたは管理対象フックを有効化するか、フックパックをインストールするか、`hooks.internal.enabled=true` を設定してオプトインします。名前を指定してフックを有効化すると、Gateway はそのフックのハンドラーのみを読み込みます。`hooks.internal.enabled=true`、追加のフックディレクトリ、およびレガシーハンドラーを使用すると、広範な検出にオプトインします。

### フックパック

フックパックは、`package.json` 内の `openclaw.hooks` を介してフックをエクスポートする npm パッケージです。次のコマンドでインストールします。

```bash
openclaw plugins install <path-or-spec>
```

npm 仕様ではレジストリのみが使用できます（パッケージ名と、任意の完全一致バージョンまたは dist-tag）。Git、URL、ファイルの仕様、および semver 範囲は拒否されます。以前の `openclaw hooks install` および `openclaw hooks update` コマンドは、`openclaw plugins install` / `openclaw plugins update` の非推奨エイリアスです。

## 同梱フック

| フック                  | イベント                                            | 動作                                                   |
| --------------------- | ------------------------------------------------- | -------------------------------------------------------------- |
| session-memory        | `command:new`, `command:reset`                    | セッションコンテキストを `<workspace>/memory/` に保存                 |
| bootstrap-extra-files | `agent:bootstrap`                                 | glob パターンから追加のブートストラップファイルを注入          |
| command-logger        | `command`                                         | すべてのコマンドを `~/.openclaw/logs/commands.log` に記録           |
| compaction-notifier   | `session:compact:before`, `session:compact:after` | セッションの Compaction の開始時と終了時に、チャット上で確認できる通知を送信 |
| boot-md               | `gateway:startup`                                 | Gateway の起動時に `BOOT.md` を実行                         |

任意の同梱フックを有効化します。

```bash
openclaw hooks enable <hook-name>
```

<a id="session-memory"></a>

### session-memory の詳細

最後のユーザーおよびアシスタントのメッセージ（デフォルトは 15 件、`hooks.internal.entries.session-memory.messages` で設定可能）を抽出し、ホストのローカル日付を使用して `<workspace>/memory/YYYY-MM-DD-HHMM.md` に保存します。メモリのキャプチャはバックグラウンドで実行されるため、トランスクリプトの読み取りや任意のスラッグ生成によって `/new` および `/reset` の確認応答が遅延することはありません。説明的なファイル名スラッグを生成するには `hooks.internal.entries.session-memory.llmSlug: true` を設定します。また、任意で `hooks.internal.entries.session-memory.model` に、`sonnet` などの設定済みエイリアス、エージェントのデフォルトプロバイダー上のモデル ID、または `provider/model` 参照を設定できます。`model` を省略した場合、スラッグ生成にはエージェントのデフォルトモデルが使用され、利用できない場合はタイムスタンプスラッグにフォールバックします。`workspace.dir` の設定が必要です。

<a id="bootstrap-extra-files"></a>

### bootstrap-extra-files の設定

```json
{
  "hooks": {
    "internal": {
      "entries": {
        "bootstrap-extra-files": {
          "enabled": true,
          "paths": ["packages/*/AGENTS.md", "packages/*/TOOLS.md"]
        }
      }
    }
  }
}
```

`patterns` および `files` は、`paths` のエイリアスとして受け入れられます。パスはワークスペースを基準に解決され、ワークスペース内に収まる必要があります。認識されるブートストラップのベース名（`AGENTS.md`、`SOUL.md`、`TOOLS.md`、`IDENTITY.md`、`USER.md`、`HEARTBEAT.md`、`BOOTSTRAP.md`、`MEMORY.md`）のみが読み込まれます。

<a id="command-logger"></a>

### command-logger の詳細

すべてのスラッシュコマンドを JSON 行（タイムスタンプ、アクション、セッションキー、送信者 ID、ソース）として `~/.openclaw/logs/commands.log` に記録します。

<a id="compaction-notifier"></a>

### compaction-notifier の詳細

OpenClaw がセッショントランスクリプトの Compaction を開始および完了したとき、現在の会話に短いステータスメッセージを送信します。これにより、アシスタントがコンテキストを要約中であり、Compaction 後に処理を続行することがユーザーに分かるため、チャット画面で長いターンを処理する際の混乱が軽減されます。

<a id="boot-md"></a>

### boot-md の詳細

設定済みの各エージェントスコープについて、そのエージェントで解決されたワークスペースにファイルが存在する場合、Gateway の起動時に `BOOT.md` を実行します。

## Plugin フック

Plugin は、より深い統合のため、Plugin SDK を介して型付きフックを登録できます。
ツール呼び出しのインターセプト、プロンプトの変更、メッセージフローの制御などが可能です。
`before_tool_call`、`before_agent_reply`、
`before_install`、またはその他のプロセス内ライフサイクルフックが必要な場合は、Plugin フックを使用します。

Plugin が管理する内部フックはこれとは異なります。このページで説明する
大まかなコマンド／ライフサイクルイベントシステムに参加し、`openclaw hooks list` では
`plugin:<id>` として表示されます。これらは副作用やフックパックとの互換性のために使用し、
順序付きミドルウェアやポリシーゲートには使用しないでください。

Plugin フックの完全なリファレンスについては、[Plugin フック](/ja-JP/plugins/hooks)を参照してください。

## 設定

```json
{
  "hooks": {
    "internal": {
      "enabled": true,
      "entries": {
        "session-memory": { "enabled": true },
        "command-logger": { "enabled": false }
      }
    }
  }
}
```

フックごとの環境値は、プロセス環境と併せてフックの `requires.env` 適格性チェックを満たすために使用され、ハンドラーはフックの設定エントリからその値を読み取ることができます。

```json
{
  "hooks": {
    "internal": {
      "entries": {
        "my-hook": {
          "enabled": true,
          "env": { "MY_CUSTOM_VAR": "value" }
        }
      }
    }
  }
}
```

追加のフックディレクトリ:

```json
{
  "hooks": {
    "internal": {
      "load": {
        "extraDirs": ["/path/to/more/hooks"]
      }
    }
  }
}
```

<Note>
後方互換性のため、従来の `hooks.internal.handlers` 配列設定形式も引き続きサポートされていますが、新しいフックでは検出ベースのシステムを使用してください。
</Note>

## CLI リファレンス

```bash
# すべてのフックを一覧表示（--eligible、--verbose、または --json を追加可能）
openclaw hooks list

# フックの詳細情報を表示
openclaw hooks info <hook-name>

# 適格性の概要を表示
openclaw hooks check

# 有効化／無効化
openclaw hooks enable <hook-name>
openclaw hooks disable <hook-name>
```

## ベストプラクティス

- **ハンドラーを高速に保つ。** フックはコマンド処理中に実行されます。負荷の高い処理は `void processInBackground(event)` を使用して、完了を待たずに実行します。
- **エラーを適切に処理する。** リスクのある操作は try/catch で囲み、他のハンドラーを実行できるように例外をスローしないでください。
- **イベントを早期に絞り込む。** イベントの種類やアクションが関連しない場合は、直ちに return します。
- **具体的なイベントキーを使用する。** オーバーヘッドを減らすため、`"events": ["command"]` よりも `"events": ["command:new"]` を優先します。

## トラブルシューティング

### フックが検出されない

```bash
# ディレクトリ構造を確認
ls -la ~/.openclaw/hooks/my-hook/
# 次のファイルが表示される必要があります: HOOK.md、handler.ts

# 検出されたすべてのフックを一覧表示
openclaw hooks list
```

### フックが適格でない

```bash
openclaw hooks info my-hook
```

不足しているバイナリ（PATH）、環境変数、設定値、または OS の互換性を確認してください。

### フックが実行されない

1. フックが有効になっていることを確認します: `openclaw hooks list`
2. フックを再読み込みするため、Gateway プロセスを再起動します。
3. Gateway のログを確認します: `openclaw logs --follow | grep -i hook`

## 関連項目

- [CLI リファレンス: フック](/ja-JP/cli/hooks)
- [Webhook](/ja-JP/automation/cron-jobs#webhooks)
- [Plugin フック](/ja-JP/plugins/hooks) — プロセス内 Plugin ライフサイクルフック
- [設定](/ja-JP/gateway/configuration-reference#hooks)
