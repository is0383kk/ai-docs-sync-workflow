---
read_when:
    - ACP を介したコーディングハーネスの実行
    - メッセージングチャネルで会話に紐づく ACP セッションを設定する
    - メッセージチャンネルの会話を永続的な ACP セッションに関連付ける
    - ACP バックエンド、Plugin の接続、または完了通知のトラブルシューティング
    - チャットから /acp コマンドを操作する
sidebarTitle: ACP agents
summary: ACP バックエンドを介して外部コーディングハーネス（Claude Code、Cursor、Gemini CLI、明示的な Codex ACP、OpenClaw ACP、OpenCode）を実行する
title: ACP エージェント
x-i18n:
    generated_at: "2026-07-26T09:20:14Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: fc7f32ff927c7e949be1595f6aa00ed034a51185c6a6b1e0df01a242954667d1
    source_path: tools/acp-agents.md
    workflow: 16
---

[Agent Client Protocol (ACP)](https://agentclientprotocol.com/) セッションを使用すると、
OpenClaw は ACP バックエンド Plugin を介して、外部コーディングハーネス（Claude Code、Cursor、Copilot、Droid、
OpenClaw ACP、OpenCode、Gemini CLI、およびその他の対応 ACPX ハーネス）を
実行できます。各起動は
[バックグラウンドタスク](/ja-JP/automation/tasks)として追跡されます。

<Note>
**ACP は外部ハーネス用の経路であり、デフォルトの Codex 経路ではありません。** ネイティブの
Codex app-server Plugin は `/codex ...` コントロールと、エージェントターン用のデフォルトの
`openai/gpt-*` 組み込みランタイムを所有します。一方、ACP は `/acp ...` コントロール
と `sessions_spawn({ runtime: "acp" })` セッションを所有します。

Codex または Claude Code を外部 MCP クライアントとして既存の
OpenClaw チャンネル会話に直接接続するには、ACP ではなく
[`openclaw mcp serve`](/ja-JP/cli/mcp) を使用します。
</Note>

## どのページを選べばよいですか？

| 目的                                                                                           | 使用するもの                          | 注記                                                                                                                                                                        |
| ----------------------------------------------------------------------------------------------- | ------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 現在の会話で Codex をバインドまたは制御する                                                    | `/codex bind`、`/codex threads`       | `codex` Plugin が有効な場合のネイティブ Codex app-server 経路：バインドされたチャット応答、画像転送、モデル／高速／権限、停止、方向修正。ACP は明示的なフォールバックです |
| OpenClaw _経由で_ Claude Code、Gemini CLI、明示的な Codex ACP、または別の外部ハーネスを実行する | このページ                            | チャットにバインドされたセッション、`/acp spawn`、`sessions_spawn({ runtime: "acp" })`、バックグラウンドタスク、ランタイム制御 |
| エディターまたはクライアント向けの ACP サーバーとして OpenClaw Gateway セッションを公開する    | [`openclaw acp`](/ja-JP/cli/acp)            | ブリッジモード：IDE／クライアントが stdio／WebSocket 経由で OpenClaw と ACP 通信します                                                                                       |
| ローカル AI CLI をテキスト専用のフォールバックモデルとして再利用する                           | [CLI バックエンド](/ja-JP/gateway/cli-backends) | ACP ではありません：OpenClaw ツール、ACP コントロール、ハーネスランタイムはありません                                                                                       |

## そのまま使えますか？

はい。公式 ACP ランタイム Plugin をインストールすると使用できます。

```bash
openclaw plugins install @openclaw/acpx
openclaw config set plugins.entries.acpx.enabled true
```

ソースチェックアウトでは、`pnpm install` の後にローカルの
`extensions/acpx` ワークスペース Plugin を使用できます。準備状況を確認するには `/acp doctor` を実行します。

OpenClaw がエージェントに ACP 起動を案内するのは、ACP が**実際に使用可能**な場合のみです。
ACP が有効であること、ディスパッチが無効化されていないこと、現在のセッションが
サンドボックスによってブロックされていないこと、ランタイムバックエンドが読み込まれて正常であることが
必要です。いずれかの条件を満たさない場合、エージェントが利用できないバックエンドを提案しないよう、
ACP Skills と `sessions_spawn` ACP ガイダンスは非表示のままになります。

<AccordionGroup>
  <Accordion title="初回実行時の注意点">
    - `plugins.allow` が設定されている場合、それは制限付き Plugin インベントリであり、`acpx` を**必ず**含める必要があります。含まれていない場合、インストール済み ACP バックエンドは意図的にブロックされます（`/acp doctor` は許可リストにエントリがないことを報告します）。
    - Codex ACP アダプターは `acpx` Plugin に同梱され、可能な場合はローカルで起動します。
    - Codex ACP は分離された `CODEX_HOME` で実行されます。OpenClaw は、信頼済みプロジェクトの信頼エントリと、安全なモデル／プロバイダーのルーティング設定（`model`、`model_provider`、`model_reasoning_effort`、`sandbox_mode`、および安全な `model_providers.<name>` フィールド）をホストの Codex 設定からコピーします。認証、通知、フックはホスト設定にのみ保持されます。
    - その他の対象ハーネスアダプターは、初回使用時に `npx` を使用してオンデマンドで取得される場合があります。
    - そのハーネスのベンダー認証は、ホスト上にあらかじめ存在している必要があります。
    - ホストで npm またはネットワークにアクセスできない場合、キャッシュを事前にウォームアップするか、別の方法でアダプターをインストールするまで、初回実行時のアダプター取得は失敗します。

  </Accordion>
  <Accordion title="ランタイムの前提条件">
    ACP は実際の外部ハーネスプロセスを起動します。OpenClaw はルーティング、
    バックグラウンドタスクの状態、配信、バインディング、ポリシーを所有し、ハーネスは
    プロバイダーへのログイン、モデルカタログ、ファイルシステムの動作、ネイティブツールを所有します。

    OpenClaw に原因があると判断する前に、次を確認してください。

    - `/acp doctor` が、有効で正常なバックエンドを報告すること。
    - その許可リストが設定されている場合、対象 ID が `acp.allowedAgents` で許可されていること。
    - Gateway ホスト上でハーネスコマンドを起動できること。
    - そのハーネス用のプロバイダー認証が存在すること（`claude`、`codex`、`gemini`、`opencode`、`droid` など）。
    - 選択したモデルがそのハーネスに存在すること。モデル ID はハーネス間で移植できません。
    - 要求した `cwd` が存在し、アクセス可能であること。または `cwd` を省略し、バックエンドのデフォルトを使用すること。
    - 権限モードが作業内容に合っていること。非対話型セッションではネイティブの権限プロンプトをクリックできないため、書き込みや実行を多用するコーディング処理では通常、ヘッドレスで処理を続行できる ACPX 権限プロファイルが必要です。

  </Accordion>
</AccordionGroup>

OpenClaw Plugin ツールと OpenClaw 組み込みツールは、デフォルトでは ACP
ハーネスに公開されません。ハーネスがこれらのツールを直接呼び出す必要がある場合にのみ、
[ACP エージェント - セットアップ](/ja-JP/tools/acp-agents-setup)で明示的な MCP ブリッジを有効にしてください。

## 対応ハーネス対象

`acpx` バックエンドでは、次の ID を `/acp spawn <id>` または
`sessions_spawn({ runtime: "acp", agentId: "<id>" })` の対象として使用します。

| ハーネス ID   | 一般的なバックエンド                           | 注記                                                                                |
| ------------ | ---------------------------------------------- | ----------------------------------------------------------------------------------- |
| `claude`     | Claude Code ACP アダプター                      | ホスト上の Claude Code 認証が必要です。                                             |
| `codex`      | Codex ACP アダプター                            | ネイティブの `/codex` が利用できない場合、または ACP が要求された場合に限る明示的な ACP フォールバックです。 |
| `copilot`    | GitHub Copilot ACP アダプター                   | Copilot CLI／ランタイム認証が必要です。                                             |
| `cursor`     | Cursor CLI ACP（`cursor-agent acp`）            | ローカルインストールで別の ACP エントリポイントが公開される場合は、acpx コマンドを上書きします。 |
| `droid`      | Factory Droid CLI                              | Factory／Droid 認証、またはハーネス環境内の `FACTORY_API_KEY` が必要です。          |
| `fast-agent` | fast-agent-mcp ACP アダプター                   | `uvx` を使用してオンデマンドで取得されます。                            |
| `gemini`     | Gemini CLI ACP アダプター                       | Gemini CLI 認証または API キーの設定が必要です。                                    |
| `iflow`      | iFlow CLI                                      | アダプターの可用性とモデル制御は、インストール済み CLI に依存します。                |
| `kilocode`   | Kilo Code CLI                                  | アダプターの可用性とモデル制御は、インストール済み CLI に依存します。                |
| `kimi`       | Kimi／Moonshot CLI                              | ホスト上の Kimi／Moonshot 認証が必要です。                                          |
| `kiro`       | Kiro CLI                                       | アダプターの可用性とモデル制御は、インストール済み CLI に依存します。                |
| `mux`        | Mux CLI ACP アダプター                          | `npx` を使用してオンデマンドで取得されます。                            |
| `opencode`   | OpenCode ACP アダプター                         | OpenCode CLI／プロバイダー認証が必要です。                                          |
| `openclaw`   | `openclaw acp` 経由の OpenClaw Gateway ブリッジ | ACP 対応ハーネスが OpenClaw Gateway セッションと通信できるようにします。             |
| `qoder`      | Qoder CLI                                      | アダプターの可用性とモデル制御は、インストール済み CLI に依存します。                |
| `qwen`       | Qwen Code／Qwen CLI                             | ホスト上の Qwen 互換認証が必要です。                                                 |
| `trae`       | Trae CLI ACP アダプター                         | アダプターの可用性とモデル制御は、インストール済み CLI に依存します。                |

`pi`（pi-acp）も acpx バックエンドに登録されていますが、上記の他のものと
同じ意味でのコーディングハーネスではありません。

カスタム acpx エージェントエイリアスは acpx 自体で設定できますが、OpenClaw
ポリシーはディスパッチ前に、引き続き `acp.allowedAgents` と
`agents.entries.*.runtime.acp.agent` のマッピングを確認します。

## 運用者向け手順書

チャットからの簡単な `/acp` の流れ：

<Steps>
  <Step title="起動">
    `/acp spawn claude --bind here`、
    `/acp spawn gemini --mode persistent --thread auto`、または明示的な
    `/acp spawn codex --bind here`。
  </Step>
  <Step title="作業">
    バインドされた会話またはスレッドで続行します（またはセッションキーを
    明示的に指定します）。
  </Step>
  <Step title="状態を確認">
    `/acp status`
  </Step>
  <Step title="調整">
    `/acp model <provider/model>`、`/acp permissions <profile>`、
    `/acp timeout <seconds>`。
  </Step>
  <Step title="方向修正">
    コンテキストを置き換えずに：`/acp steer tighten logging and continue`。
  </Step>
  <Step title="停止">
    `/acp cancel`（現在のターン）または `/acp close`（セッション＋バインディング）。
  </Step>
</Steps>

<AccordionGroup>
  <Accordion title="ライフサイクルの詳細">
    - 生成処理では、ACP ランタイムセッションを作成または再開し、OpenClaw セッションストアに ACP メタデータを記録します。また、実行が親によって所有されている場合は、バックグラウンドタスクを作成することがあります。
    - 親所有の ACP セッションは、ランタイムセッションが永続的であってもバックグラウンド処理として扱われます。完了通知とサーフェス間の配信は、通常のユーザー向けチャットセッションのようには動作せず、親タスクの通知機構を通じて行われます。
    - タスクのメンテナンスでは、終了済みまたは孤立した親所有のワンショット ACP セッションを閉じます。永続 ACP セッションは、有効な会話バインディングが残っている間は保持されます。有効なバインディングがない古い永続セッションは、所有タスクの完了後やタスクレコードの消失後に暗黙的に再開されないよう閉じられます。
    - バインドされた後続メッセージは、バインディングが閉じられる、フォーカスが解除される、リセットされる、または期限切れになるまで、ACP セッションへ直接送られます。
    - Gateway コマンドはローカルに留まります。`/acp ...`、`/status`、`/unfocus` が、バインドされた ACP ハーネスへ通常のプロンプトテキストとして送信されることはありません。
    - `cancel` は、バックエンドがキャンセルをサポートしている場合にアクティブなターンを中止します。バインディングやセッションメタデータは削除しません。
    - `close` は、OpenClaw から見た ACP セッションを終了し、バインディングを削除します。ハーネスが再開をサポートしている場合、ハーネス側では独自のアップストリーム履歴を保持することがあります。
    - acpx Plugin は、`close` 後に OpenClaw 所有のラッパーおよびアダプタープロセスツリーをクリーンアップし、Gateway 起動時に古い OpenClaw 所有の ACPX 孤立プロセスを回収します。
    - アイドル状態のランタイムワーカーは、組み込みのアイドル期間を過ぎるとクリーンアップ対象になります。保存されたセッションメタデータは、`/acp sessions` で引き続き利用できます。

  </Accordion>
  <Accordion title="ネイティブ Codex のルーティング規則">
    有効な場合に **ネイティブ Codex Plugin** へルーティングされるべき
    自然言語トリガー：

    - 「この Discord チャンネルを Codex にバインドしてください。」
    - 「このチャットを Codex スレッド `<id>` に接続してください。」
    - 「Codex スレッドを表示してから、これをバインドしてください。」

    ネイティブ Codex の会話バインディングが、チャット制御のデフォルト経路です。
    OpenClaw の動的ツールは引き続き OpenClaw を通じて実行されますが、シェルや
    apply-patch などの Codex ネイティブツールは Codex 内で実行されます。Codex
    ネイティブのツールイベントに対して、OpenClaw はターンごとのネイティブフック
    リレーを挿入します。これにより、Plugin フックは `before_tool_call` をブロックし、
    `after_tool_call` を監視し、Codex の `PermissionRequest` イベントを OpenClaw の
    承認経由でルーティングできます。Codex の `Stop` フックは
    OpenClaw の `before_agent_finalize` に中継され、そこで Plugin は Codex が回答を
    確定する前に、モデルをもう一度実行するよう要求できます。このリレーは意図的に
    保守的な動作を維持します。Codex ネイティブツールの引数を変更したり、Codex
    スレッドレコードを書き換えたりすることはありません。ACP ランタイム／セッション
    モデルを使用したい場合にのみ、明示的な ACP を使用してください。組み込み Codex
    のサポート境界については、
    [Codex ハーネス v1 サポート契約](/ja-JP/plugins/codex-harness-runtime#v1-support-contract)
    を参照してください。

  </Accordion>
  <Accordion title="モデル／プロバイダー／ランタイム選択早見表">
    - 従来の Codex モデル参照 - doctor によって修復される、従来の Codex OAuth／サブスクリプションモデル経路。
    - `openai/*` - OpenAI エージェントターン用のネイティブ Codex app-server 組み込みランタイム。
    - `/codex ...` - ネイティブ Codex の会話制御。
    - `/acp ...` または `runtime: "acp"` - 明示的な ACP／acpx 制御。

  </Accordion>
  <Accordion title="ACP ルーティング用の自然言語トリガー">
    ACP ランタイムへルーティングされるべきトリガー：

    - 「これをワンショットの Claude Code ACP セッションとして実行し、結果を要約してください。」
    - 「このタスクにはスレッド内で Gemini CLI を使用し、その後のやり取りも同じスレッドで続けてください。」
    - 「バックグラウンドスレッドで ACP 経由の Codex を実行してください。」

    OpenClaw は `runtime: "acp"` を選択し、ハーネス `agentId` を解決し、
    サポートされている場合は現在の会話またはスレッドにバインドして、閉じられるか
    期限切れになるまで後続メッセージをそのセッションへルーティングします。Codex が
    この経路を使用するのは、ACP／acpx が明示されている場合、または要求された操作で
    ネイティブ Codex Plugin を利用できない場合のみです。

    `sessions_spawn` では、ACP が有効であり、リクエスト元がサンドボックス化されて
    おらず、ACP ランタイムバックエンドが読み込まれている場合にのみ
    `runtime: "acp"` が提示されます。`acp.dispatch.enabled=false` は ACP スレッドの自動
    ディスパッチを一時停止しますが、明示的な `sessions_spawn({ runtime: "acp" })` 呼び出しを非表示に
    したりブロックしたりはしません。対象には `codex`、`claude`、
    `droid`、`gemini`、`opencode` などの ACP ハーネス ID
    を指定します。`agents_list` の通常の OpenClaw 設定エージェント ID は、その
    エントリに `agents.entries.*.runtime.type="acp"` が明示的に設定されていない限り渡さないでください。
    それ以外の場合は、デフォルトのサブエージェントランタイムを使用します。OpenClaw
    エージェントに `runtime.type="acp"` が設定されている場合、OpenClaw は基盤となる
    ハーネス ID として `runtime.acp.agent` を使用します。

  </Accordion>
</AccordionGroup>

## ACP とサブエージェントの比較

外部ハーネスランタイムを使用する場合は ACP を使用します。`codex` Plugin
が有効な場合、Codex の会話バインディング／制御には **ネイティブ Codex
app-server** を使用します。OpenClaw ネイティブの委任実行を使用する場合は、
**サブエージェント**を使用します。

| 領域          | ACP セッション                           | サブエージェント実行                      |
| ------------- | ------------------------------------- | ---------------------------------- |
| ランタイム       | ACP バックエンド Plugin（例：acpx） | OpenClaw ネイティブのサブエージェントランタイム  |
| セッションキー   | `agent:<agentId>:acp:<uuid>`          | `agent:<agentId>:subagent:<uuid>`  |
| 主要コマンド | `/acp ...`                            | `/subagents ...`                   |
| 生成ツール    | `runtime:"acp"` を指定した `sessions_spawn` | `sessions_spawn`（デフォルトランタイム） |

[サブエージェント](/ja-JP/tools/subagents)も参照してください。

## ACP が Claude Code を実行する仕組み

ACP 経由の Claude Code では、スタックは次のとおりです。

1. OpenClaw ACP セッション制御プレーン。
2. 公式 `@openclaw/acpx` ランタイム Plugin。
3. Claude ACP アダプター。
4. Claude 側のランタイム／セッション機構。

ACP Claude は、ACP 制御、セッション再開、バックグラウンドタスク追跡、および
オプションの会話／スレッドバインディングを備えた **ハーネスセッション**です。

CLI バックエンドは、独立したテキスト専用のローカルフォールバックランタイムです。
[CLI バックエンド](/ja-JP/gateway/cli-backends)を参照してください。

運用者向けの実用的な原則は次のとおりです。

- **`/acp spawn`、バインド可能なセッション、ランタイム制御、または永続的なハーネス処理が必要ですか？** ACP を使用します。
- **生の CLI を通じた単純なローカルテキストフォールバックが必要ですか？** CLI バックエンドを使用します。

## バインドされたセッション

### 基本概念

- **チャットサーフェス** - ユーザーが会話を続ける場所（Discord チャンネル、Telegram トピック、iMessage チャット）。
- **ACP セッション** - OpenClaw がルーティングする、永続的な Codex／Claude／Gemini ランタイム状態。
- **子スレッド／トピック** - `--thread ...` によってのみ作成される、オプションの追加メッセージングサーフェス。
- **ランタイムワークスペース** - ハーネスが実行されるファイルシステム上の場所（`cwd`、リポジトリのチェックアウト、バックエンドワークスペース）。チャットサーフェスとは独立しています。

### 現在の会話へのバインド

`/acp spawn <harness> --bind here` は、現在の会話を生成された ACP セッションに固定します。
子スレッドは作成されず、同じチャットサーフェスを使用します。OpenClaw は引き続き
トランスポート、認証、安全性、配信を管理します。その会話内の後続メッセージは
同じセッションへルーティングされます。`/new` と `/reset` は
セッションをその場でリセットし、`/acp close` はバインディングを削除します。

例：

```text
/codex bind                                              # ネイティブ Codex をバインドし、以後のメッセージをここへルーティング
/codex model gpt-5.4                                     # バインドされたネイティブ Codex スレッドを調整
/codex stop                                              # アクティブなネイティブ Codex ターンを制御
/acp spawn codex --bind here                             # Codex 用の明示的な ACP フォールバック
/acp spawn codex --thread auto                           # 子スレッド／トピックを作成し、そこへバインドする場合がある
/acp spawn codex --bind here --cwd /workspace/repo       # 同じチャットバインディングを使用し、Codex は /workspace/repo で実行
```

<AccordionGroup>
  <Accordion title="バインディングの規則と排他性">
    - `--bind here` と `--thread ...` は同時に指定できません。
    - `--bind here` は、現在の会話へのバインディング対応を提示するチャンネルでのみ機能します。それ以外の場合、OpenClaw は未対応であることを示す明確なメッセージを返します。バインディングは Gateway の再起動後も保持されます。
    - Discord では、`spawnSessions` が制御するのは `--thread auto|here` の子スレッド作成であり、`--bind here` ではありません。
    - `--cwd` を指定せずに別の ACP エージェントを生成すると、OpenClaw はデフォルトで **対象エージェントの** ワークスペースを継承します。継承対象のパス（`ENOENT`／`ENOTDIR`）が存在しない場合はバックエンドのデフォルトにフォールバックします。それ以外のアクセスエラー（例：`EACCES`）は生成エラーとして提示されます。
    - Gateway 管理コマンドは、バインドされた会話内でもローカルに留まります。通常の後続テキストがバインドされた ACP セッションへルーティングされる場合でも、`/acp ...` コマンドは OpenClaw が処理します。また、そのサーフェスでコマンド処理が有効な場合、`/status` と `/unfocus` も常にローカルに留まります。

  </Accordion>
  <Accordion title="スレッドにバインドされたセッション">
    チャンネルアダプターでスレッドバインディングが有効な場合：

    - OpenClaw はスレッドを対象 ACP セッションにバインドします。
    - そのスレッド内の後続メッセージは、バインドされた ACP セッションへルーティングされます。
    - ACP の出力は同じスレッドへ返されます。
    - フォーカス解除、クローズ、アーカイブ、アイドルタイムアウト、または最大存続期間の満了により、バインディングが削除されます。
    - `/acp close`、`/acp cancel`、`/acp status`、`/status`、`/unfocus` は Gateway コマンドであり、ACP ハーネスへのプロンプトではありません。

    スレッドにバインドされた ACP に必要な機能フラグ：

    - `acp.enabled=true`
    - `acp.dispatch.enabled` はデフォルトで有効です（ACP スレッドの自動ディスパッチを一時停止するには `false` を設定します。明示的な `sessions_spawn({ runtime: "acp" })` 呼び出しは引き続き機能します）。
    - チャンネルアダプターによるスレッドセッション生成が有効（デフォルト：`true`）：
      - Discord／Telegram：`session.threadBindings.spawnSessions=true`

    スレッドバインディングの対応状況はアダプターごとに異なります。アクティブな
    チャンネルアダプターがスレッドバインディングをサポートしていない場合、
    OpenClaw は未対応または利用不可であることを示す明確なメッセージを返します。

  </Accordion>
  <Accordion title="スレッド対応チャンネル">
    - セッション／スレッドバインディング機能を公開するすべてのチャンネルアダプター。
    - 現在の組み込み対応：**Discord** のスレッド／チャンネル、**Telegram** のトピック（グループ／スーパーグループのフォーラムトピックおよび DM トピック）。
    - Plugin チャンネルは、同じバインディングインターフェースを通じて対応を追加できます。

  </Accordion>
</AccordionGroup>

## 永続チャンネルバインディング

一時的ではないワークフローでは、トップレベルの `bindings[]` エントリに
永続 ACP バインディングを設定します。

### バインディングモデル

<ParamField path="bindings[].type" type='"acp"'>
  永続 ACP 会話バインディングであることを示します。
</ParamField>
<ParamField path="bindings[].match" type="object">
  対象の会話を識別します。チャンネルごとの形式：

- **Discord チャンネル/スレッド:** `match.channel="discord"` + `match.peer.id="<channelOrThreadId>"`
- **Slack チャンネル/DM:** `match.channel="slack"` + `match.peer.id="<channelId|channel:<channelId>|#<channelId>|userId|user:<userId>|slack:<userId>|<@userId>>"`。安定した Slack ID を優先してください。チャンネルバインディングは、そのチャンネルのスレッド内の返信にも一致します。
- **Telegram フォーラムトピック:** `match.channel="telegram"` + `match.peer.id="<chatId>:topic:<topicId>"`
- **WhatsApp DM/グループ:** `match.channel="whatsapp"` + `match.peer.id="<E.164|group JID>"`。ダイレクトチャットには `+15555550123` のような E.164 番号を、グループには `120363424282127706@g.us` のような WhatsApp グループ JID を使用します。
- **iMessage DM/グループ:** `match.channel="imessage"` + `match.peer.id="<handle|chat_id:*|chat_guid:*|chat_identifier:*>"`。安定したグループバインディングには `chat_id:*` を優先してください。

</ParamField>
<ParamField path="bindings[].agentId" type="string">
  所有する OpenClaw エージェント ID。
</ParamField>
<ParamField path="bindings[].acp.mode" type='"persistent" | "oneshot"'>
  オプションの ACP オーバーライド。
</ParamField>
<ParamField path="bindings[].acp.label" type="string">
  オプションのオペレーター向けラベル。
</ParamField>
<ParamField path="bindings[].acp.cwd" type="string">
  オプションのランタイム作業ディレクトリ。
</ParamField>
<ParamField path="bindings[].acp.backend" type="string">
  オプションのバックエンドオーバーライド。
</ParamField>

### エージェントごとのランタイムデフォルト

`agents.entries.*.runtime` を使用して、エージェントごとに ACP デフォルトを一度定義します。

- `agents.entries.*.runtime.type="acp"`
- `agents.entries.*.runtime.acp.agent`（ハーネス ID。例: `codex` または `claude`）
- `agents.entries.*.runtime.acp.backend`
- `agents.entries.*.runtime.acp.mode`
- `agents.entries.*.runtime.acp.cwd`

**ACP バインド済みセッションのオーバーライド優先順位:**

1. `bindings[].acp.*`
2. `agents.entries.*.runtime.acp.*`
3. グローバル ACP デフォルト（例: `acp.backend`）

### 例

```json5
{
  agents: {
    list: [
      {
        id: "codex",
        runtime: {
          type: "acp",
          acp: {
            agent: "codex",
            backend: "acpx",
            mode: "persistent",
            cwd: "/workspace/openclaw",
          },
        },
      },
      {
        id: "claude",
        runtime: {
          type: "acp",
          acp: { agent: "claude", backend: "acpx", mode: "persistent" },
        },
      },
    ],
  },
  bindings: [
    {
      type: "acp",
      agentId: "codex",
      match: {
        channel: "discord",
        accountId: "default",
        peer: { kind: "channel", id: "222222222222222222" },
      },
      acp: { label: "codex-main" },
    },
    {
      type: "acp",
      agentId: "claude",
      match: {
        channel: "telegram",
        accountId: "default",
        peer: { kind: "group", id: "-1001234567890:topic:42" },
      },
      acp: { cwd: "/workspace/repo-b" },
    },
    {
      type: "route",
      agentId: "main",
      match: { channel: "discord", accountId: "default" },
    },
    {
      type: "route",
      agentId: "main",
      match: { channel: "telegram", accountId: "default" },
    },
  ],
  channels: {
    discord: {
      guilds: {
        "111111111111111111": {
          channels: {
            "222222222222222222": { requireMention: false },
          },
        },
      },
    },
    telegram: {
      groups: {
        "-1001234567890": {
          topics: { "42": { requireMention: false } },
        },
      },
    },
  },
}
```

### 動作

- OpenClaw は、チャンネル固有の受け入れ判定後かつ使用前に、設定された ACP セッションが存在することを保証します。
- そのチャンネル、トピック、またはチャット内のメッセージは、設定された ACP セッションにルーティングされます。
- 設定された ACP バインディングは、そのセッションルートを所有します。チャンネルのブロードキャストファンアウトは、一致したバインディングに設定された ACP セッションを置き換えません。
- バインド済みの会話では、`/new` と `/reset` は、同じ ACP セッションキーをその場でリセットします。
- 一時的なランタイムバインディング（たとえば、スレッドフォーカスフローによって作成されたもの）は、存在する場合には引き続き適用されます。
- 明示的な `cwd` を指定しないエージェント間 ACP スポーンでは、OpenClaw はエージェント設定から対象エージェントのワークスペースを継承します。
- 継承されたワークスペースパスが存在しない場合は、バックエンドのデフォルト cwd にフォールバックします。存在するパスへのアクセス失敗は、スポーンエラーとして表示されます。

## ACP セッションを開始する

ACP セッションを開始する方法は 2 つあります。

<Tabs>
  <Tab title="sessions_spawn から">
    エージェントターンまたはツール呼び出しから ACP セッションを開始するには、
    `runtime: "acp"` を使用します。

    ```json
    {
      "task": "リポジトリを開き、失敗しているテストを要約してください",
      "runtime": "acp",
      "agentId": "codex",
      "thread": true,
      "mode": "session"
    }
    ```

    <Note>
    `runtime` のデフォルトは `subagent` なので、ACP セッションでは
    `runtime: "acp"` を明示的に設定してください。`agentId` を省略すると、
    設定されている場合は OpenClaw が `acp.defaultAgent` を使用します。
    `mode: "session"` で永続的なバインド済み会話を維持するには、
    `thread: true` が必要です。
    </Note>

  </Tab>
  <Tab title="/acp コマンドから">
    チャットからオペレーターが明示的に制御するには、`/acp spawn` を使用します。

    ```text
    /acp spawn codex --mode persistent --thread auto
    /acp spawn codex --mode oneshot --thread off
    /acp spawn codex --bind here
    /acp spawn codex --thread here
    ```

    主なフラグ:

    - `--mode persistent|oneshot`
    - `--bind here|off`
    - `--thread auto|here|off`
    - `--cwd <absolute-path>`
    - `--label <name>`

    [スラッシュコマンド](/ja-JP/tools/slash-commands)を参照してください。

  </Tab>
</Tabs>

### `sessions_spawn` のパラメーター

<ParamField path="task" type="string" required>
  ACP セッションに送信される最初のプロンプト。
</ParamField>
<ParamField path="runtime" type='"acp"' required>
  ACP セッションでは `"acp"` である必要があります。
</ParamField>
<ParamField path="agentId" type="string">
  ACP 対象ハーネス ID。`acp.defaultAgent` が設定されている場合は、これにフォールバックします。
</ParamField>
<ParamField path="thread" type="boolean" default="false">
  サポートされている場合にスレッドバインディングフローを要求します。
</ParamField>
<ParamField path="mode" type='"run" | "session"' default="run">
  `"run"` はワンショット、`"session"` は永続的です。`thread: true` で
  `mode` が省略された場合、OpenClaw はランタイムパスに応じて
  デフォルトで永続的な動作を使用することがあります。`mode: "session"` には
  `thread: true` が必要です。
</ParamField>
<ParamField path="cwd" type="string">
  要求するランタイム作業ディレクトリ（バックエンド/ランタイムポリシーによって検証されます）。
  省略した場合、ACP スポーンは、設定されていれば対象エージェントのワークスペースを継承します。
  継承されたパスが存在しない場合はバックエンドのデフォルトにフォールバックし、
  実際のアクセスエラーは返されます。
</ParamField>
<ParamField path="label" type="string">
  セッション/バナーテキストで使用されるオペレーター向けラベル。
</ParamField>
<ParamField path="resumeSessionId" type="string">
  新しい ACP セッションを作成する代わりに、既存のセッションを再開します。エージェントは
  `session/load` を介して会話履歴を再生します。
  `runtime: "acp"` が必要です。
</ParamField>
<ParamField path="streamTo" type='"parent"'>
  `"parent"` は、最初の ACP 実行の進捗概要をシステムイベントとして要求元
  セッションにストリーミングします。OpenClaw は完全なリレー履歴を子エージェントの
  SQLite 状態に記録し、子セッションとともに削除します。`streaming.progress.commentary=false` でない限り、
  親の進捗ストリームにはデフォルトでアシスタントのコメントと ACP ステータス進捗が表示されます。
  ストリームモードが設定されていない場合、Discord もデフォルトで親プレビューに
  進捗モードを使用します。ステータス進捗では引き続き `acp.stream.tagVisibility` が適用されるため、
  `plan` のようなタグは明示的に有効にしない限り非表示のままです。
</ParamField>

ACP `sessions_spawn` の実行では、デフォルトの子ターン上限として
`agents.defaults.subagents.runTimeoutSeconds` を使用します。このツールは呼び出しごとの
タイムアウトオーバーライドを受け付けません（`runTimeoutSeconds`/`timeoutSeconds` は、
デフォルトを設定するよう求めるエラーで拒否されます）。

<ParamField path="model" type="string">
  ACP 子セッションの明示的なモデルオーバーライド。Codex ACP スポーンでは、
  `openai/gpt-5.4` のような OpenAI 参照を、`session/new` の前に
  Codex ACP 起動設定へ正規化します。`openai/gpt-5.4/high` のようなスラッシュ形式では、
  Codex ACP の推論強度も設定されます。省略した場合、`sessions_spawn({ runtime: "acp" })` は、
  設定されていれば既存のサブエージェントモデルデフォルト（`agents.defaults.subagents.model` または
  `agents.entries.*.subagents.model`）を使用します。それ以外の場合は、ACP ハーネス独自の
  デフォルトモデルを使用させます。その他のハーネスは ACP `models` を通知し、
  `session/set_model` をサポートする必要があります。そうでなければ、OpenClaw/acpx は
  対象エージェントのデフォルトへ暗黙的にフォールバックせず、明確に失敗します。
</ParamField>
<ParamField path="thinking" type="string">
  明示的な思考/推論強度。Codex ACP では、`minimal` は低い強度にマッピングされ、
  `low`/`medium`/`high`/`xhigh` はそのままマッピングされ、
  `off` では推論強度の起動オーバーライドが省略されます。省略した場合、
  ACP スポーンは既存のサブエージェントの思考デフォルトと、選択されたモデルの
  モデルごとの `agents.defaults.models["provider/model"].params.thinking` を使用します。
</ParamField>

## スポーンのバインドモードとスレッドモード

<Tabs>
  <Tab title="--bind here|off">
    | モード   | 動作                                                               |
    | ------ | ----------------------------------------------------------------------- |
    | `here` | 現在アクティブな会話をその場でバインドします。アクティブな会話がなければ失敗します。 |
    | `off`  | 現在の会話へのバインディングを作成しません。                          |

    注意:

    - `--bind here` は、「このチャンネルまたはチャットを Codex バックエンドにする」ための最も簡単なオペレーターパスです。
    - `--bind here` は子スレッドを作成しません。
    - `--bind here` は、現在の会話へのバインディングをサポートするチャンネルでのみ使用できます。
    - `--bind` と `--thread` は、同じ `/acp spawn` 呼び出しで併用できません。

  </Tab>
  <Tab title="--thread auto|here|off">
    | モード   | 動作                                                                                            |
    | ------ | ------------------------------------------------------------------------------------------------- |
    | `auto` | アクティブなスレッド内では、そのスレッドをバインドします。スレッド外では、サポートされている場合に子スレッドを作成してバインドします。 |
    | `here` | 現在アクティブなスレッドを必須とし、スレッド内でなければ失敗します。                                                  |
    | `off`  | バインディングなし。セッションはバインドされていない状態で開始します。                                                                 |

    注意:

    - スレッド以外のバインディングサーフェスでは、デフォルトの動作は実質的に `off` です。
    - スレッドバインド済みスポーンには、チャンネルポリシーによるサポートが必要です。
      - Discord/Telegram: `session.threadBindings.spawnSessions=true`
    - 子スレッドを作成せずに現在の会話を固定する場合は、`--bind here` を使用します。

  </Tab>
</Tabs>

## 配信モデル

ACP セッションは、対話型ワークスペースまたは親が所有するバックグラウンド作業の
いずれかです。配信経路はその形態によって異なります。

<AccordionGroup>
  <Accordion title="対話型 ACP セッション">
    対話型セッションは、表示されているチャットサーフェス上で会話を継続することを目的としています。

    - `/acp spawn ... --bind here` は、現在の会話を ACP セッションにバインドします。
    - `/acp spawn ... --thread ...` は、チャンネルのスレッド/トピックを ACP セッションにバインドします。
    - 永続的に設定された `bindings[].type="acp"` は、一致する会話を同じ ACP セッションにルーティングします。

    バインド済み会話の後続メッセージは ACP セッションに直接ルーティングされ、
    ACP の出力は同じチャンネル/スレッド/トピックに返されます。

    OpenClaw がハーネスに送信する内容:

    - 通常の制限付きフォローアップはプロンプトテキストとして送信され、ハーネス/バックエンドが対応している場合にのみ添付ファイルも送信されます。
    - `/acp` 管理コマンドとローカル Gateway コマンドは、ACP ディスパッチの前にインターセプトされます。
    - ランタイムが生成する完了イベントは、ターゲットごとに具体化されます。OpenClaw エージェントは OpenClaw の内部ランタイムコンテキストエンベロープを受け取り、外部 ACP ハーネスは子の結果と指示を含むプレーンなプロンプトを受け取ります。生の `<<<BEGIN_OPENCLAW_INTERNAL_CONTEXT>>>` エンベロープを外部ハーネスに送信したり、ACP ユーザートランスクリプトのテキストとして永続化したりしてはなりません。
    - ACP トランスクリプトのエントリには、ユーザーに表示されるトリガーテキストまたはプレーンな完了プロンプトが使用されます。内部イベントメタデータは可能な限り OpenClaw 内で構造化されたまま維持され、ユーザーが作成したチャットコンテンツとして扱われません。

  </Accordion>
  <Accordion title="親が所有するワンショット ACP セッション">
    別のエージェント実行によって生成されたワンショット ACP セッションは、
    サブエージェントと同様のバックグラウンドの子です。

    - 親は `sessions_spawn({ runtime: "acp", mode: "run" })` を使用して作業を依頼します。
    - 子は独自の ACP ハーネスセッションで実行されます。
    - 子のターンはネイティブのサブエージェント生成と同じバックグラウンドレーンで実行されるため、低速な ACP ハーネスが無関係なメインセッションの作業をブロックすることはありません。
    - 完了はタスク完了通知パスを通じて親に報告されます。OpenClaw は内部完了メタデータをプレーンな ACP プロンプトに変換してから外部ハーネスへ送信するため、ハーネスに OpenClaw 専用のランタイムコンテキストマーカーが表示されることはありません。
    - ユーザー向けの応答が有用な場合、親は子の結果を通常のアシスタントの口調で書き直します。

    このパスを、親と子の間のピアツーピアチャットとして扱わないで
    **ください**。子にはすでに親へ返す完了チャネルがあります。

  </Accordion>
  <Accordion title="sessions_send と A2A 配信">
    `sessions_send` は生成後に別のセッションをターゲットにできます。通常のピア
    セッションでは、OpenClaw はメッセージを注入した後に
    エージェント間（A2A）フォローアップパスを使用します。

    - ターゲットセッションからの応答を待ちます。
    - 必要に応じて、依頼元とターゲットの間で制限された回数のフォローアップターンをやり取りさせます。
    - ターゲットに通知メッセージの生成を依頼します。
    - その通知を表示中のチャネルまたはスレッドに配信します。

    この A2A パスは、送信者が表示可能なフォローアップを必要とする
    ピア送信のフォールバックです。たとえば広範な `tools.sessions.visibility`
    設定の下で、無関係なセッションが ACP ターゲットを参照して
    メッセージを送信できる場合にも有効なままです。

    OpenClaw が A2A フォローアップをスキップするのは、依頼元が
    自身の親所有ワンショット ACP 子の親である場合のみです。この場合、
    タスク完了に加えて A2A を実行すると、子の結果で親を起動し、
    親の応答を子へ送り返して、親子間のエコーループを
    作成する可能性があります。完了パスがすでに結果を処理するため、
    この所有された子の場合、`sessions_send` の結果は
    `delivery.status="skipped"` を報告します。

  </Accordion>
  <Accordion title="既存セッションの再開">
    新規に開始する代わりに、`resumeSessionId` を使用して以前の ACP セッションを
    続行します。エージェントは `session/load` を介して会話履歴を再生するため、
    以前の完全なコンテキストを引き継いで再開します。

    ```json
    {
      "task": "中断したところから続けてください - 残っているテスト失敗を修正してください",
      "runtime": "acp",
      "agentId": "codex",
      "resumeSessionId": "<previous-session-id>"
    }
    ```

    一般的なユースケース：

    - Codex セッションをノートパソコンからスマートフォンへ引き継ぎ、エージェントに中断したところから再開するよう指示します。
    - CLI で対話的に開始したコーディングセッションを、今度はエージェントを通じてヘッドレスで続行します。
    - Gateway の再起動またはアイドルタイムアウトによって中断された作業を再開します。

    注意事項：

    - `resumeSessionId` は `runtime: "acp"` の場合にのみ適用されます。デフォルトのサブエージェントランタイムは、この ACP 専用フィールドを無視します。
    - `streamTo` は `runtime: "acp"` の場合にのみ適用されます。デフォルトのサブエージェントランタイムは、この ACP 専用フィールドを無視します。
    - `resumeSessionId` はホストローカルの ACP/ハーネス再開 ID であり、OpenClaw チャネルセッションキーではありません。OpenClaw はディスパッチ前に ACP 生成ポリシーとターゲットエージェントポリシーを引き続き確認しますが、その上流 ID を読み込むための認可は ACP バックエンドまたはハーネスが所有します。
    - `resumeSessionId` は上流 ACP の会話履歴を復元します。`thread` と `mode` は新しく作成する OpenClaw セッションにも通常どおり適用されるため、`mode: "session"` には引き続き `thread: true` が必要です。
    - ターゲットエージェントは `session/load` をサポートしている必要があります（Codex と Claude Code はサポートしています）。
    - セッション ID が見つからない場合、生成は明確なエラーで失敗します。新しいセッションへの暗黙的なフォールバックはありません。

  </Accordion>
  <Accordion title="デプロイ後のスモークテスト">
    Gateway のデプロイ後は、単体テストを信頼するだけでなく、
    ライブのエンドツーエンドチェックを実行します。

    1. ターゲットホストにデプロイされた Gateway のバージョンとコミットを確認します。
    2. 稼働中のエージェントへの一時的な ACPX ブリッジセッションを開きます。
    3. そのエージェントに、`runtime: "acp"`、`agentId: "codex"`、`mode: "run"`、およびタスク `Reply with exactly LIVE-ACP-SPAWN-OK` を指定して `sessions_spawn` を呼び出すよう依頼します。
    4. `accepted=yes`、実際の `childSessionKey`、およびバリデーターエラーがないことを確認します。
    5. 一時的なブリッジセッションをクリーンアップします。

    ゲートは `mode: "run"` に維持し、`streamTo: "parent"` はスキップします。
    スレッドにバインドされた `mode: "session"` とストリームリレーパスは、
    より高度な別個の統合パスです。

  </Accordion>
</AccordionGroup>

## サンドボックスの互換性

ACP セッションは現在、OpenClaw サンドボックス内では**なく**、
ホストランタイム上で実行されます。

<Warning>
**セキュリティ境界：**

- 外部ハーネスは、独自の CLI 権限と選択された `cwd` に従って読み書きできます。
- OpenClaw のサンドボックスポリシーは、ACP ハーネスの実行を**ラップしません**。
- OpenClaw は、ACP の機能ゲート、許可されたエージェント、セッション所有権、チャネルバインディング、および Gateway 配信ポリシーを引き続き適用します。
- サンドボックスが適用される OpenClaw ネイティブの作業には `runtime: "subagent"` を使用します。

</Warning>

現在の制限事項：

- 依頼元セッションがサンドボックス化されている場合、`sessions_spawn({ runtime: "acp" })` と `/acp spawn` の両方で ACP の生成がブロックされます。
- `runtime: "acp"` を指定した `sessions_spawn` は `sandbox: "require"` をサポートしていません。

## セッションターゲットの解決

ほとんどの `/acp` アクションは、任意のセッションターゲット（`session-key`、
`session-id`、または `session-label`）を受け入れます。

**解決順序：**

1. 明示的なターゲット引数（または `/acp steer` の `--session`）
   - 最初にキーを試行
   - 次に UUID 形式のセッション ID
   - 次にラベル
2. 現在のスレッドバインディング（この会話/スレッドが ACP セッションにバインドされている場合）。
3. 現在の依頼元セッションへのフォールバック。

現在の会話のバインディングとスレッドのバインディングは、どちらもステップ 2 に関与します。

ターゲットを解決できない場合、OpenClaw は明確なエラー
（`Unable to resolve session target: ...`）を返します。

## ACP コントロール

| コマンド              | 動作                                              | 例                                                       |
| -------------------- | --------------------------------------------------------- | ------------------------------------------------------------- |
| `/acp spawn`         | ACP セッションを作成します。必要に応じて現在のバインドまたはスレッドバインドを指定できます。 | `/acp spawn codex --bind here --cwd /repo`                    |
| `/acp cancel`        | ターゲットセッションで進行中のターンをキャンセルします。                 | `/acp cancel agent:codex:acp:<uuid>`                          |
| `/acp steer`         | 実行中のセッションに誘導指示を送信します。                | `/acp steer --session support inbox prioritize failing tests` |
| `/acp close`         | セッションを閉じ、スレッドターゲットのバインドを解除します。                  | `/acp close`                                                  |
| `/acp status`        | バックエンド、モード、状態、ランタイムオプション、機能を表示します。 | `/acp status`                                                 |
| `/acp set-mode`      | ターゲットセッションのランタイムモードを設定します。                      | `/acp set-mode plan`                                          |
| `/acp set`           | 汎用ランタイム設定オプションを書き込みます。                      | `/acp set model openai/gpt-5.4`                               |
| `/acp cwd`           | ランタイムの作業ディレクトリのオーバーライドを設定します。                   | `/acp cwd /Users/user/Projects/repo`                          |
| `/acp permissions`   | 承認ポリシープロファイルを設定します。                              | `/acp permissions strict`                                     |
| `/acp timeout`       | ランタイムタイムアウト（秒）を設定します。                            | `/acp timeout 120`                                            |
| `/acp model`         | ランタイムモデルのオーバーライドを設定します。                               | `/acp model anthropic/claude-opus-4-6`                        |
| `/acp reset-options` | セッションのランタイムオプションのオーバーライドを削除します。                  | `/acp reset-options`                                          |
| `/acp sessions`      | ストアから最近の ACP セッションを一覧表示します。                      | `/acp sessions`                                               |
| `/acp doctor`        | バックエンドの健全性、機能、実行可能な修正方法を表示します。           | `/acp doctor`                                                 |
| `/acp install`       | 決定論的なインストールおよび有効化手順を出力します。             | `/acp install`                                                |

ランタイムコントロール（`spawn`、`cancel`、`steer`、`close`、`status`、`set-mode`、
`set`、`cwd`、`permissions`、`timeout`、`model`、および `reset-options`）には、
外部チャネルからの所有者 ID と、内部 Gateway クライアントからの
`operator.admin` が必要です。認可された所有者以外の送信者も、`sessions`、
`doctor`、`install`、および `help` は引き続き使用できます。所有者以外の送信者の場合、`/acp sessions`
は現在バインドされているセッションまたは依頼元セッションのみを一覧表示します。所有者 ID と
`operator.admin` クライアントは、最近のすべてのセッションを参照できます。

`/acp status` は、有効なランタイムオプションに加えて、ランタイムレベルおよび
バックエンドレベルのセッション識別子を表示します。バックエンドに機能がない場合、
サポートされていないコントロールのエラーが明確に表示されます。ターゲットトークン
（`session-key`、`session-id`、または `session-label`）を受け入れるコマンドは、エージェントごとのカスタム `session.store` ルートを含む、
Gateway のセッション検出を通じてそれらを解決します。`/acp sessions` は
ターゲットトークンを受け入れません。

### ランタイムオプションのマッピング

`/acp` には便利なコマンドと汎用セッターがあります。同等の操作：

| コマンド                      | 対応先                              | 注記                                                                                                                                                                                                      |
| ---------------------------- | ------------------------------------ | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `/acp model <id>`            | ランタイム設定キー `model`           | Codex ACP では、OpenClaw は `openai/<model>` をアダプターモデル ID に正規化し、`openai/gpt-5.4/high` などのスラッシュ形式の推論サフィックスを `reasoning_effort` に対応付けます。                                         |
| `/acp set thinking <level>`  | 標準オプション `thinking`          | OpenClaw は、バックエンドが通知した同等の値が存在する場合にそれを送信し、`thinking`、次に `effort`、`reasoning_effort`、`thought_level` の順で優先します。Codex ACP では、アダプターが値を `reasoning_effort` に対応付けます。 |
| `/acp permissions <profile>` | 標準オプション `permissionProfile` | OpenClaw は、`approval_policy`、`permission_profile`、`permissions`、`permission_mode` など、バックエンドが通知した同等の値が存在する場合にそれを送信します。                                                       |
| `/acp timeout <seconds>`     | 標準オプション `timeoutSeconds`    | OpenClaw は、`timeout` や `timeout_seconds` など、バックエンドが通知した同等の値が存在する場合にそれを送信します。                                                                                                     |
| `/acp cwd <path>`            | ランタイムの cwd オーバーライド                 | 直接更新します。                                                                                                                                                                                             |
| `/acp set <key> <value>`     | 汎用                              | `key=cwd` は cwd オーバーライドパスを使用します。                                                                                                                                                                      |
| `/acp reset-options`         | すべてのランタイムオーバーライドを解除         | -                                                                                                                                                                                                          |

## acpx ハーネス、Plugin のセットアップ、権限

acpx ハーネスの設定（Claude Code / Codex / Gemini CLI のエイリアス）、
plugin-tools および OpenClaw-tools MCP ブリッジ、ACP 権限モードについては、
[ACP エージェントのセットアップ](/ja-JP/tools/acp-agents-setup)を参照してください。

## トラブルシューティング

| 症状                                                                                   | 考えられる原因                                                                                                           | 修正方法                                                                                                                                                                      |
| ----------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `ACP runtime backend is not configured`                                                   | バックエンド Plugin がない、無効になっている、または `plugins.allow` によってブロックされています。                                                       | バックエンド Plugin をインストールして有効化し、その許可リストが設定されている場合は `acpx` を `plugins.allow` に含めてから、`/acp doctor` を実行します。                                                 |
| `ACP is disabled by policy (acp.enabled=false)`                                           | ACP が全体で無効になっています。                                                                                                 | `acp.enabled=true` を設定します。                                                                                                                                                  |
| `ACP dispatch is disabled by policy (acp.dispatch.enabled=false)`                         | 通常のスレッドメッセージからの自動ディスパッチが無効になっています。                                                               | 自動スレッドルーティングを再開するには `acp.dispatch.enabled=true` を設定します。明示的な `sessions_spawn({ runtime: "acp" })` 呼び出しは引き続き機能します。                                      |
| `ACP agent "<id>" is not allowed by policy`                                               | エージェントが許可リストに含まれていません。                                                                                                | 許可されている `agentId` を使用するか、`acp.allowedAgents` を更新します。                                                                                                                     |
| `/acp doctor` が起動直後にバックエンドの準備ができていないと報告する                               | バックエンド Plugin がない、無効になっている、許可・拒否ポリシーによってブロックされている、または設定された実行可能ファイルを利用できません。        | バックエンド Plugin をインストールまたは有効化し、`/acp doctor` を再実行します。正常な状態にならない場合は、バックエンドのインストールエラーまたはポリシーエラーを確認します。                                           |
| ハーネスコマンドが見つからない                                                                 | アダプター CLI がインストールされていない、外部 Plugin がない、または Codex 以外のアダプターで初回実行時の `npx` の取得に失敗しました。 | `/acp doctor` を実行し、Gateway ホストにアダプターをインストールまたは事前準備するか、acpx エージェントコマンドを明示的に設定します。                                                      |
| ハーネスからモデルが見つからないというエラーが返される                                                          | モデル ID は別のプロバイダーやハーネスでは有効ですが、この ACP ターゲットでは有効ではありません。                                                | そのハーネスに一覧表示されるモデルを使用するか、ハーネスでモデルを設定するか、オーバーライドを省略します。                                                                            |
| ハーネスからベンダー認証エラーが返される                                                        | OpenClaw は正常ですが、ターゲットの CLI またはプロバイダーにログインしていません。                                                     | Gateway ホスト環境でログインするか、必要なプロバイダーキーを指定します。                                                                                             |
| `Unable to resolve session target: ...`                                                   | キー、ID、またはラベルトークンが正しくありません。                                                                                                | `/acp sessions` を実行し、正確なキーまたはラベルをコピーして再試行します。                                                                                                                        |
| `--bind here requires running /acp spawn inside an active ... conversation`               | アクティブでバインド可能な会話がない状態で `--bind here` が使用されました。                                                            | 対象のチャットまたはチャンネルに移動して再試行するか、バインドなしで生成します。                                                                                                         |
| `Conversation bindings are unavailable for <channel>.`                                    | アダプターに現在の会話を ACP にバインドする機能がありません。                                                             | サポートされている場合は `/acp spawn ... --thread ...` を使用するか、トップレベルの `bindings[]` を設定するか、サポートされているチャンネルに移動します。                                                     |
| `--thread here requires running /acp spawn inside an active ... thread`                   | スレッドコンテキストの外で `--thread here` が使用されました。                                                                         | 対象のスレッドに移動するか、`--thread auto`/`off` を使用します。                                                                                                                      |
| `Only <user-id> can rebind this channel/conversation/thread.`                             | 別のユーザーがアクティブなバインド対象を所有しています。                                                                           | 所有者として再バインドするか、別の会話またはスレッドを使用します。                                                                                                               |
| `Thread bindings are unavailable for <channel>.`                                          | アダプターにスレッドバインド機能がありません。                                                                               | `--thread off` を使用するか、サポートされているアダプターまたはチャンネルに移動します。                                                                                                                 |
| `Sandboxed sessions cannot spawn ACP sessions ...`                                        | ACP ランタイムはホスト側にありますが、リクエスト元のセッションはサンドボックス化されています。                                                              | サンドボックス化されたセッションから `runtime="subagent"` を使用するか、サンドボックス化されていないセッションから ACP の生成を実行します。                                                                         |
| `sessions_spawn sandbox="require" is unsupported for runtime="acp" ...`                   | ACP ランタイムに対して `sandbox="require"` が要求されました。                                                                         | サンドボックス化が必要な場合は `runtime="subagent"` を使用するか、サンドボックス化されていないセッションから `sandbox="inherit"` を指定して ACP を使用します。                                                      |
| `Cannot apply --model ... did not advertise model support`                                | 対象のハーネスは汎用 ACP モデル切り替え機能を公開していません。                                                        | ACP の `models`/`session/set_model` を通知するハーネスまたは Codex ACP モデル参照を使用するか、ハーネス独自の起動フラグがある場合はハーネスでモデルを直接設定します。 |
| バインド済みセッションの ACP メタデータがない                                                    | ACP セッションのメタデータが古いか削除されています。                                                                                    | `/acp spawn` で再作成してから、スレッドを再バインドまたはフォーカスします。                                                                                                                    |
| `PermissionPromptUnavailableError: Permission prompt unavailable in non-interactive mode` | `permissionMode` が非対話型 ACP セッションでの書き込みや実行をブロックしています。                                                    | `plugins.entries.acpx.config.permissionMode` を `approve-all` に設定し、Gateway を再起動します。[権限の設定](/ja-JP/tools/acp-agents-setup#permission-configuration)を参照してください。 |
| ACP セッションがほとんど出力せず早期に失敗する                                                | 権限プロンプトが `permissionMode`/`nonInteractivePermissions` によってブロックされています。                                        | Gateway ログで `AcpRuntimeError` を確認します。完全な権限を付与するには `permissionMode=approve-all` を設定し、機能を段階的に縮退させるには `nonInteractivePermissions=deny` を設定します。        |
| ACP セッションが作業完了後も無期限に停止する                                     | ハーネスプロセスは終了しましたが、ACP セッションが完了を報告していません。                                                    | OpenClaw を更新してください。現在の acpx クリーンアップ処理では、クローズ時および Gateway 起動時に、OpenClaw が所有する古いラッパープロセスとアダプタープロセスを終了します。                                             |
| ハーネスに `<<<BEGIN_OPENCLAW_INTERNAL_CONTEXT>>>` が表示される                                      | 内部イベントエンベロープが ACP 境界を越えて漏洩しました。                                                                | OpenClaw を更新して完了フローを再実行してください。外部ハーネスはプレーンな完了プロンプトのみを受信する必要があります。                                                          |

<Note>
`Command blocked by PreToolUse hook: Native hook relay unavailable` は、
ACP/acpx ではなくネイティブ Codex フックリレーに属します。バインドされた Codex チャットでは、
`/new` または `/reset` で新しいセッションを開始してください。一度は機能しても、
次のネイティブツール呼び出しで再び発生する場合は、`/new` を繰り返すのではなく、
Codex app-server または OpenClaw Gateway を再起動してください。
[Codex ハーネスのトラブルシューティング](/ja-JP/plugins/codex-harness#troubleshooting)を参照してください。
</Note>

## 関連項目

- [ACP エージェント - セットアップ](/ja-JP/tools/acp-agents-setup)
- [エージェント送信](/ja-JP/tools/agent-send)
- [CLI バックエンド](/ja-JP/gateway/cli-backends)
- [Codex ハーネス](/ja-JP/plugins/codex-harness)
- [Codex ハーネスランタイム](/ja-JP/plugins/codex-harness-runtime)
- [マルチエージェントサンドボックスツール](/ja-JP/tools/multi-agent-sandbox-tools)
- [`openclaw acp`（ブリッジモード）](/ja-JP/cli/acp)
- [サブエージェント](/ja-JP/tools/subagents)
