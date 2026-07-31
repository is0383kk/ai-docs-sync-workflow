---
read_when:
    - ACP ベースの IDE 統合を設定する
    - Gateway への ACP セッションルーティングのデバッグ
summary: IDE 統合用の ACP ブリッジを実行する
title: ACP
x-i18n:
    generated_at: "2026-07-26T09:55:06Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: becdcfdd1cc62b206cc92e9b8248c79a2ff63cfc3779d8a124b9713e779ad33c
    source_path: cli/acp.md
    workflow: 16
---

OpenClaw Gateway と通信する [Agent Client Protocol (ACP)](https://agentclientprotocol.com/) ブリッジを実行します。

`openclaw acp` は IDE 向けに stdio 経由で ACP を処理し、WebSocket 経由でプロンプトを Gateway に転送します。その際、ACP セッションと Gateway セッションキーの対応関係を維持します。これは Gateway を基盤とする ACP ブリッジであり、完全な ACP ネイティブのエディターランタイムではありません。セッションのルーティング、プロンプトの配信、更新のストリーミングに重点を置いています。

ACP ハーネスセッションをホストする代わりに、外部 MCP クライアントから OpenClaw のチャンネル会話へ直接接続する場合は、[`openclaw mcp serve`](/ja-JP/cli/mcp) を使用してください。

## これに該当しないもの

`openclaw acp` は、OpenClaw が ACP サーバーとして動作することを意味します。IDE または ACP クライアントが OpenClaw に接続し、OpenClaw がその処理を Gateway セッションへ転送します。

これは、OpenClaw が `acpx` を介して Codex や Claude Code などの外部ハーネスを実行する [ACP エージェント](/ja-JP/tools/acp-agents) とは異なります。

簡単な判断基準：

- エディター／クライアントから ACP で OpenClaw と通信する場合：`openclaw acp` を使用
- OpenClaw から Codex／Claude／Gemini を ACP ハーネスとして起動する場合：`/acp spawn` と [ACP エージェント](/ja-JP/tools/acp-agents)を使用

## 互換性マトリクス

| ACP の領域                                                              | ステータス      | 注記                                                                                                                                                                                                                                 |
| --------------------------------------------------------------------- | ----------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `initialize`、`newSession`、`prompt`、`cancel`                        | 実装済み | stdio から Gateway の chat/send および abort までのコアブリッジフロー。                                                                                                                                                                             |
| `listSessions`、スラッシュコマンド                                        | 実装済み | セッション一覧は、制限付きカーソルページネーションと、Gateway セッション行にワークスペースメタデータが含まれる場合の `cwd` フィルタリングを使用して、Gateway のセッション状態に対して動作します。コマンドは `available_commands_update` を介して通知されます。                     |
| セッション系統メタデータ                                              | 実装済み | セッション一覧とセッション情報のスナップショットには、`_meta` に OpenClaw の親子系統が含まれるため、ACP クライアントは Gateway の非公開サイドチャンネルを使用せずにサブエージェントグラフをレンダリングできます。                                                     |
| `resumeSession`、`closeSession`                                       | 実装済み | 再開では、履歴を再生せずに ACP セッションを既存の Gateway セッションへ再バインドします。終了では、進行中のブリッジ処理をキャンセルし、保留中のプロンプトをキャンセル済みとして解決し、ブリッジのセッション状態を解放します。                                   |
| `loadSession`                                                         | 部分対応     | ACP セッションを Gateway セッションキーに再バインドし、ブリッジが作成したセッションについて ACP イベント台帳の履歴を再生します。古いセッションや台帳のないセッションでは、保存されているユーザー／アシスタントのテキストにフォールバックします。                                                  |
| プロンプトの内容（`text`、埋め込み `resource`、画像）                  | 部分対応     | テキスト／リソースはチャット入力にフラット化され、画像は Gateway の添付ファイルになります。                                                                                                                                                            |
| セッションモード                                                         | 部分対応     | `session/set_mode` に対応しています。ブリッジは、思考レベル、ツールの詳細度、推論、使用量の詳細、昇格アクションについて、Gateway を基盤とするセッション制御を公開します。より広範な ACP ネイティブのモード／設定サーフェスは引き続き対象外です。 |
| 思考のストリーミング                                                     | 実装済み | モデルの思考内容は `agent_thought_chunk` セッション更新としてストリーミングされます。ACP ネイティブのセッションプランは送信されません。                                                                                                                    |
| セッション情報と使用量の更新                                        | 部分対応     | ブリッジは、キャッシュされた Gateway セッションスナップショットから `session_info_update` とベストエフォートの `usage_update` 通知を送信します。使用量は概算であり、Gateway のトークン合計が最新とマークされている場合にのみ送信されます。                             |
| ツールのストリーミング                                                        | 部分対応     | `tool_call`／`tool_call_update` イベントには、生の入出力、テキストコンテンツ、および Gateway のツール引数／結果から取得できる場合はベストエフォートのファイル位置が含まれます。埋め込みターミナルや、より高度な差分ネイティブ出力は公開されません。                     |
| exec の承認                                                        | 部分対応     | アクティブな ACP プロンプトターン中の Gateway exec 承認プロンプトは、`session/request_permission` を使用して ACP クライアントへ中継されます。                                                                                                               |
| セッション単位の MCP サーバー（`mcpServers`）                                | 非対応 | ブリッジモードでは、セッション単位の MCP サーバーリクエストを拒否します。代わりに OpenClaw Gateway またはエージェントで MCP を設定してください。                                                                                                                          |
| クライアントのファイルシステムメソッド（`fs/read_text_file`、`fs/write_text_file`） | 非対応 | ブリッジは ACP クライアントのファイルシステムメソッドを呼び出しません。                                                                                                                                                                               |
| クライアントのターミナルメソッド（`terminal/*`）                                | 非対応 | ブリッジは ACP クライアントのターミナルを作成せず、ツール呼び出しを介してターミナル ID をストリーミングしません。                                                                                                                                            |

## 既知の制限事項

- `loadSession` が完全な ACP イベント台帳履歴を再生するのは、ブリッジが作成したセッションのみです。古いセッションや台帳のないセッションではトランスクリプトへのフォールバックを使用し、過去のツール呼び出しやシステム通知は再構築されません。
- 複数の ACP クライアントが同じ Gateway セッションキーを共有する場合、イベントとキャンセルのルーティングはクライアントごとに厳密に分離されず、ベストエフォートになります。エディター内のターンを明確に分離する必要がある場合は、デフォルトの分離された `acp-bridge:<uuid>` セッションを使用してください。
- Gateway の停止状態は ACP の停止理由に変換されますが、そのマッピングは完全な ACP ネイティブランタイムほど表現力がありません。
- セッション制御で公開される Gateway の設定項目は、思考レベル、ツールの詳細度、推論、使用量の詳細、昇格アクションに限定されています。モデル選択と exec ホストの制御は ACP 設定オプションとして公開されません。
- `session_info_update` と `usage_update` は、ACP ネイティブランタイムのリアルタイムな計測ではなく、Gateway セッションのスナップショットから取得されます。使用量は概算で、コストデータを含まず、Gateway がトークン合計データを最新とマークしている場合にのみ送信されます。
- ツールの追跡データはベストエフォートです。ブリッジは既知のツール引数／結果に現れるファイルパスを公開しますが、ACP ターミナルや構造化されたファイル差分は送信しません。
- exec 承認の中継範囲は、アクティブな ACP プロンプトターンに限定されます。他の Gateway セッションからの承認は無視されます。

## 使用方法

```bash
openclaw acp

# リモート Gateway
openclaw acp --url wss://gateway-host:18789 --token <token>

# リモート Gateway（ファイルからトークンを取得）
openclaw acp --url wss://gateway-host:18789 --token-file ~/.openclaw/gateway.token

# 既存のセッションキーに接続
openclaw acp --session agent:main:main

# ラベルで接続（事前に存在している必要があります）
openclaw acp --session-label "support inbox"

# 最初のプロンプトの前にセッションキーをリセット
openclaw acp --session agent:main:main --reset-session
```

## ACP クライアント（デバッグ）

IDE を使用せずにブリッジを簡易確認するには、組み込みの ACP クライアントを使用します。このクライアントは ACP ブリッジを起動し、対話形式でプロンプトを入力できます。

```bash
openclaw acp client

# 起動したブリッジの接続先をリモート Gateway に設定
openclaw acp client --server-args --url wss://gateway-host:18789 --token-file ~/.openclaw/gateway.token

# サーバーコマンドを上書き（デフォルト：openclaw）
openclaw acp client --server "node" --server-args openclaw.mjs acp --url ws://127.0.0.1:19001
```

権限モデル（クライアントデバッグモード）：

- 自動承認は許可リストに基づき、信頼されたコアツール ID のみに適用されます。
- `read` の自動承認の範囲は、現在の作業ディレクトリ（設定されている場合は `--cwd`）に限定されます。
- ACP が自動承認するのは、限定された読み取り専用クラスのみです。具体的には、アクティブな cwd 配下を対象とする `read` 呼び出しと、読み取り専用の検索ツール（`search`、`web_search`、`memory_search`）です。不明なツールやコア以外のツール、範囲外の読み取り、exec 対応ツール、コントロールプレーンツール、変更を伴うツール、対話型フローには、常に明示的なプロンプト承認が必要です。
- サーバーが提供する `toolCall.kind` は信頼できないメタデータとして扱われ、認可の根拠としては使用されません。
- この ACP ブリッジポリシーは、ACPX ハーネスの権限とは別です。`acpx` バックエンドを介して OpenClaw を実行する場合、`plugins.entries.acpx.config.permissionMode=approve-all` はそのハーネスセッション用の緊急時「yolo」スイッチです。

## プロトコルのスモークテスト

プロトコルレベルのデバッグでは、分離された状態で Gateway を起動し、ACP JSON-RPC クライアントから stdio 経由で `openclaw acp` を操作します。`initialize`、`session/new`、絶対パスの `cwd` を指定した `session/list`、`session/resume`、`session/close`、重複する終了、存在しないセッションの再開を網羅してください。

検証には、通知されたライフサイクル機能、Gateway を基盤とするセッション行、更新通知、Gateway の `sessions.list` ログを含める必要があります。

```json
{
  "initialize": {
    "protocolVersion": 1,
    "agentCapabilities": {
      "sessionCapabilities": {
        "list": {},
        "resume": {},
        "close": {}
      }
    }
  },
  "listSessions": {
    "sessions": [
      {
        "sessionId": "agent:main:acp-smoke",
        "cwd": "/path/to/workspace",
        "_meta": {
          "sessionKey": "agent:main:acp-smoke",
          "kind": "direct"
        }
      }
    ],
    "nextCursor": null
  },
  "notifications": ["session_info_update", "available_commands_update", "usage_update"],
  "gatewayLogTail": ["[gateway] ready", "[ws] ⇄ res ✓ sessions.list 305ms"]
}
```

ACP の唯一の検証として `openclaw gateway call sessions.list` を使用しないでください。この CLI パスでは、新しいトークンによるオペレータースコープの昇格を要求する場合があります。ACP ブリッジの正当性は、ACP の stdio フレームと Gateway の `sessions.list` ログによって検証します。

## 使用方法

IDE（またはその他のクライアント）が Agent Client Protocol を使用し、そのクライアントから OpenClaw Gateway セッションを操作する場合に ACP を使用します。

1. Gateway が実行中であることを確認します（ローカルまたはリモート）。
2. Gateway の接続先を設定します（設定またはフラグ）。
3. IDE で、stdio 経由で `openclaw acp` を実行するように指定します。

設定例（永続化）：

```bash
openclaw config set gateway.remote.url wss://gateway-host:18789
openclaw config set gateway.remote.token <token>
```

直接実行の例（設定への書き込みなし）：

```bash
openclaw acp --url wss://gateway-host:18789 --token <token>
# ローカルプロセスの安全性を確保するために推奨
openclaw acp --url wss://gateway-host:18789 --token-file ~/.openclaw/gateway.token
```

## エージェントの選択

ACP はエージェントを直接選択しません。Gateway セッションキーに基づいてルーティングします。特定のエージェントを対象にするには、エージェントスコープのセッションキーを使用します。

```bash
openclaw acp --session agent:main:main
openclaw acp --session agent:design:main
openclaw acp --session agent:qa:bug-123
```

各 ACP セッションは、1 つの Gateway セッションキーに対応します。1 つのエージェントが複数のセッションを持つことができます。キーまたはラベルを上書きしない限り、ACP はデフォルトで分離された `acp-bridge:<uuid>` セッションを使用します。

セッションごとの `mcpServers` はブリッジモードではサポートされていません。ACP クライアントが `newSession` または `loadSession` の実行中にそれらを送信した場合、ブリッジは黙って無視するのではなく、明確なエラーを返します。

ACPX を利用するセッションから OpenClaw Plugin ツールや `cron` などの選択された組み込みツールを使用できるようにするには、セッションごとの `mcpServers` を渡そうとするのではなく、Gateway 側の ACPX MCP ブリッジを有効にします。[ACP エージェント](/ja-JP/tools/acp-agents-setup#plugin-tools-mcp-bridge)および[OpenClaw ツール MCP ブリッジ](/ja-JP/tools/acp-agents-setup#openclaw-tools-mcp-bridge)を参照してください。

## `acpx`（Codex、Claude、その他の ACP クライアント）から使用する

Codex や Claude Code などのコーディングエージェントから ACP 経由で OpenClaw ボットと通信するには、組み込みの `openclaw` ターゲットを備えた `acpx` を使用します。

一般的な流れ：

1. Gateway を実行し、ACP ブリッジから接続できることを確認します。
2. `acpx openclaw` の接続先を `openclaw acp` に設定します。
3. コーディングエージェントに使用させる OpenClaw セッションキーを指定します。

例：

```bash
# デフォルトの OpenClaw ACP セッションへの単発リクエスト
acpx openclaw exec "アクティブな OpenClaw セッションの状態を要約してください。"

# 後続ターン用の永続的な名前付きセッション
acpx openclaw sessions ensure --name codex-bridge
acpx openclaw -s codex-bridge --cwd /path/to/repo \
  "このリポジトリに関連する最近のコンテキストを、私の OpenClaw 作業エージェントに尋ねてください。"
```

`acpx openclaw` が毎回特定の Gateway とセッションキーを対象にするようにするには、`~/.acpx/config.json` 内の `openclaw` エージェントコマンドを上書きします。

```json
{
  "agents": {
    "openclaw": {
      "command": "env OPENCLAW_HIDE_BANNER=1 OPENCLAW_SUPPRESS_NOTES=1 openclaw acp --url ws://127.0.0.1:18789 --token-file ~/.openclaw/gateway.token --session agent:main:main"
    }
  }
}
```

リポジトリローカルの OpenClaw チェックアウトでは、ACP ストリームをクリーンに保つため、開発ランナーではなく直接 CLI エントリポイントを使用します。

```bash
env OPENCLAW_HIDE_BANNER=1 OPENCLAW_SUPPRESS_NOTES=1 node openclaw.mjs acp ...
```

これは、Codex、Claude Code、またはその他の ACP 対応クライアントから、ターミナルをスクレイピングせずに OpenClaw エージェントのコンテキスト情報を取得する最も簡単な方法です。

## Zed エディターのセットアップ

`~/.config/zed/settings.json` にカスタム ACP エージェントを追加します（または Zed の Settings UI を使用します）。

```json
{
  "agent_servers": {
    "OpenClaw ACP": {
      "type": "custom",
      "command": "openclaw",
      "args": ["acp"],
      "env": {}
    }
  }
}
```

特定の Gateway またはエージェントを対象にする場合：

```json
{
  "agent_servers": {
    "OpenClaw ACP": {
      "type": "custom",
      "command": "openclaw",
      "args": [
        "acp",
        "--url",
        "wss://gateway-host:18789",
        "--token",
        "<token>",
        "--session",
        "agent:design:main"
      ],
      "env": {}
    }
  }
}
```

Zed で Agent パネルを開き、「OpenClaw ACP」を選択してスレッドを開始します。

## セッションのマッピング

デフォルトでは、ACP ブリッジセッションには `acp-bridge:` プレフィックスを持つ分離された Gateway セッションキーが割り当てられます。これらの通常モデルのブリッジセッションは合成された破棄可能なセッションです。古いエントリの削除対象となり、保護対象の人間との会話サーフェスとしては扱われません。既知のセッションを再利用するには、セッションキーまたはラベルを渡します。

- `--session <key>`：特定の Gateway セッションキーを使用します。
- `--session-label <label>`：ラベルによって既存のセッションを解決します。
- `--reset-session`：そのキーに対して新しいセッション ID を発行します（同じキー、新しいトランスクリプト）。

ACP クライアントがメタデータをサポートしている場合は、セッションごとに上書きできます。

```json
{
  "_meta": {
    "sessionKey": "agent:main:main",
    "sessionLabel": "support inbox",
    "resetSession": true
  }
}
```

セッションキーの詳細については、[/concepts/session](/ja-JP/concepts/session)を参照してください。

## オプション

- `--url <url>`：Gateway WebSocket URL（設定されている場合、デフォルトは `gateway.remote.url`）。
- `--token <token>`：Gateway 認証トークン。
- `--token-file <path>`：ファイルから Gateway 認証トークンを読み取ります。
- `--password <password>`：Gateway 認証パスワード。
- `--password-file <path>`：ファイルから Gateway 認証パスワードを読み取ります。
- `--session <key>`：デフォルトのセッションキー。
- `--session-label <label>`：解決するデフォルトのセッションラベル。
- `--require-existing`：セッションキーまたはラベルが存在しない場合は失敗します。
- `--reset-session`：最初に使用する前にセッションキーをリセットします。
- `--no-prefix-cwd`：プロンプトの先頭に作業ディレクトリを付加しません。
- `--provenance <off|meta|meta+receipt>`：ACP の来歴メタデータまたは受領情報を含めます。
- `--verbose, -v`：詳細ログを stderr に出力します。

セキュリティに関する注意：

- `--token` と `--password` は、一部のシステムではローカルプロセス一覧に表示される可能性があります。`--token-file`/`--password-file` または環境変数（`OPENCLAW_GATEWAY_TOKEN`、`OPENCLAW_GATEWAY_PASSWORD`）を優先してください。
- Gateway 認証の解決は、他の Gateway クライアントが使用する共通の規約に従います。
  - ローカルモード：環境変数（`OPENCLAW_GATEWAY_*`）、次に `gateway.auth.*` を使用し、`gateway.auth.*` が未設定の場合に限り `gateway.remote.*` にフォールバックします（設定済みだが解決できないローカル SecretRef は、黙ってフォールバックせずフェイルクローズします）
  - リモートモード：リモートの優先順位規則に従い、環境変数または設定へのフォールバックを伴う `gateway.remote.*` を使用します
  - `--url` は上書き時にも安全で、暗黙の設定や環境変数の認証情報を再利用しません。明示的な `--token`/`--password`（またはファイル版）を渡してください

### `acp client` のオプション

- `--cwd <dir>`：ACP セッションの作業ディレクトリ。
- `--server <command>`：ACP サーバーコマンド（デフォルト：`openclaw`）。
- `--server-args <args...>`：ACP サーバーに渡す追加引数。
- `--server-verbose`：ACP サーバーの詳細ログを有効にします。
- `--verbose, -v`：クライアントの詳細ログ。
- `openclaw acp client` は、起動されたブリッジプロセスに `OPENCLAW_SHELL=acp-client` を設定します。これは、コンテキスト固有のシェルまたはプロファイル規則に使用できます。

## 関連項目

- [CLI リファレンス](/ja-JP/cli)
- [ACP エージェント](/ja-JP/tools/acp-agents)
