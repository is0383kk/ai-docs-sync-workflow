---
read_when:
    - チャットコマンドの使用または設定
    - コマンドルーティングまたは権限のデバッグ
    - スキルコマンドの登録方法を理解する
sidebarTitle: Slash commands
summary: 利用可能なすべてのスラッシュコマンド、ディレクティブ、インラインショートカット — 設定、ルーティング、各サーフェス固有の動作。
title: スラッシュコマンド
x-i18n:
    generated_at: "2026-07-26T09:55:49Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: ee5ee5e46d632a54ea92dea7ca61046288bf1998d05b08396107bec90e646fff
    source_path: tools/slash-commands.md
    workflow: 16
---

Gateway は、`/` で始まる単独メッセージとして送信されたコマンドを処理します。
ホスト専用の bash コマンドでは `! <cmd>` を使用します（`/bash <cmd>` はエイリアスです）。

会話が ACP セッションにバインドされている場合、通常のテキストは ACP
ハーネスにルーティングされます。Gateway 管理コマンドはローカルに残ります。`/acp ...` は常に
OpenClaw コマンドハンドラーに到達し、`/status` と `/unfocus` は、そのサーフェスで
コマンド処理が有効な場合は常にローカルに残ります。

## 3 種類のコマンド

<CardGroup cols={3}>
  <Card title="コマンド" icon="terminal">
    Gateway が処理する単独の `/...` メッセージです。メッセージ内の
    唯一の内容として送信する必要があります。
  </Card>
  <Card title="ディレクティブ" icon="sliders">
    `/think`、`/fast`、`/verbose`、`/trace`、`/reasoning`、`/elevated`、
    `/exec`、`/model`、`/queue` — モデルがメッセージを
    認識する前に取り除かれます。単独で送信した場合はセッション設定を永続化し、他のテキストと
    一緒に送信した場合はインラインヒントとして機能します。
  </Card>
  <Card title="インラインショートカット" icon="bolt">
    `/help`、`/commands`、`/status`、`/whoami` — 即座に実行され、
    モデルが残りのテキストを認識する前に取り除かれます。許可された送信者のみ使用できます。
  </Card>
</CardGroup>

<AccordionGroup>
  <Accordion title="ディレクティブの動作の詳細">
    - ディレクティブは、モデルがメッセージを認識する前に取り除かれます。
    - **ディレクティブのみ**のメッセージ（メッセージの内容がディレクティブのみ）では、
      セッションに永続化され、確認応答が返されます。
    - 他のテキストを含む**通常のチャット**メッセージでは、インラインヒントとして機能し、
      セッション設定は永続化**されません**。
    - ディレクティブは**許可された送信者**にのみ適用されます。`commands.allowFrom` が
      設定されている場合、それが使用される唯一の許可リストです。それ以外の場合、認可は
      チャンネルの許可リスト、ペアリング、および常時有効なアクセスグループの適用によって決まります。許可されていない
      送信者からのディレクティブはプレーンテキストとして扱われます。
  </Accordion>
</AccordionGroup>

## 設定

```json5
{
  commands: {
    native: "auto",
    nativeSkills: "auto",
    text: true,
    bash: false,
    bashForegroundMs: 2000,
    config: false,
    mcp: false,
    plugins: false,
    debug: false,
    restart: true,
    ownerAllowFrom: ["discord:123456789012345678"],
    ownerDisplay: "raw",
    ownerDisplaySecret: "${OWNER_ID_HASH_SECRET}",
    allowFrom: {
      "*": ["user1"],
      discord: ["user:123"],
    },
    useAccessGroups: true,
  },
}
```

<ParamField path="commands.text" type="boolean" default="true">
  チャットメッセージ内の `/...` の解析を有効にします。ネイティブコマンドがないサーフェス
  （WhatsApp、WebChat、Signal、iMessage、Google Chat、Microsoft Teams）では、`false` に
  設定されている場合でもテキストコマンドが機能します。
</ParamField>

<ParamField path="commands.native" type='boolean | "auto"' default='"auto"'>
  ネイティブコマンドを登録します。自動設定では Discord/Telegram で有効、Slack で無効となり、
  ネイティブ対応のないプロバイダーでは無視されます。チャンネルごとに
  `channels.<provider>.commands.native` で上書きできます。Discord では、`false` によってスラッシュコマンドの
  登録がスキップされます。以前に登録されたコマンドは、削除されるまで表示され続ける場合があります。
</ParamField>

<ParamField path="commands.nativeSkills" type='boolean | "auto"' default='"auto"'>
  対応している場合、スキルコマンドをネイティブに登録します。自動設定では
  Discord/Telegram で有効、Slack で無効となります。
  `channels.<provider>.commands.nativeSkills` で上書きできます。
</ParamField>

<ParamField path="commands.bash" type="boolean" default="false">
  `! <cmd>` によるホストのシェルコマンド実行を有効にします（`/bash <cmd>` はエイリアスです）。
  `tools.elevated` の許可リストが必要です。
</ParamField>

<ParamField path="commands.bashForegroundMs" type="number" default="2000">
  bash がバックグラウンドモードへ切り替わるまで待機する時間です（`0` は
  即座にバックグラウンド化します）。
</ParamField>

<ParamField path="commands.config" type="boolean" default="false">
  `/config` を有効にします（`openclaw.json` を読み書きします）。オーナーのみ使用できます。
</ParamField>

<ParamField path="commands.mcp" type="boolean" default="false">
  `/mcp` を有効にします（`mcp.servers` 配下にある OpenClaw 管理の MCP 設定を読み書きします）。オーナーのみ使用できます。
</ParamField>

<ParamField path="commands.plugins" type="boolean" default="false">
  `/plugins` を有効にします（Plugin の検出とステータス、およびインストールと有効化／無効化）。書き込みはオーナーのみ実行できます。
</ParamField>

<ParamField path="commands.debug" type="boolean" default="false">
  `/debug` を有効にします（実行時のみの設定上書き）。オーナーのみ使用できます。
</ParamField>

<ParamField path="commands.restart" type="boolean" default="true">
  `/restart` および外部からの `SIGUSR1` 再起動リクエストを有効にします。
</ParamField>

<ParamField path="commands.ownerAllowFrom" type="string[]">
  オーナー専用コマンドサーフェスに対する明示的なオーナー許可リストです。
  `commands.allowFrom` および DM ペアリングアクセスとは別です。
</ParamField>

<ParamField path="channels.<channel>.commands.enforceOwnerForCommands" type="boolean" default="false">
  チャンネルごとの設定です。オーナー専用コマンドにオーナー ID を要求します。`true` の場合、
  送信者は `commands.ownerAllowFrom` と一致するか、内部の `operator.admin`
  スコープを保持している必要があります。ワイルドカードの `allowFrom` エントリだけでは**不十分**です。
</ParamField>

<ParamField path="commands.ownerDisplay" type='"raw" | "hash"'>
  システムプロンプトでのオーナー ID の表示方法を制御します。
</ParamField>

<ParamField path="commands.ownerDisplaySecret" type="string">
  `commands.ownerDisplay: "hash"` の場合に使用される HMAC シークレットです。
</ParamField>

<ParamField path="commands.allowFrom" type="object">
  コマンド認可のためのプロバイダーごとの許可リストです。設定されている場合、これが
  コマンドとディレクティブに対する**唯一の**認可ソースになります。グローバルなデフォルトには
  `"*"` を使用します。プロバイダー固有のキーはこれを上書きします。
</ParamField>

## コマンド一覧

コマンドは次の 3 つのソースから提供されます。

- **コア組み込み:** `src/auto-reply/commands-registry.shared.ts`
- **生成されたドックコマンド:** `src/auto-reply/commands-registry.data.ts`
- **Plugin コマンド:** Plugin の `registerCommand()` 呼び出し

利用可否は、設定フラグ、チャンネルサーフェス、およびインストール済みで有効な
Plugin によって決まります。

### コアコマンド

<AccordionGroup>
  <Accordion title="セッションと実行">
    | コマンド | 説明 |
    | --- | --- |
    | `/new [model]` | 現在のセッションをアーカイブし、新しいセッションを開始します |
    | `/reset [soft [message]]` | 現在のセッションをその場でリセットします。`soft` はトランスクリプトを保持し、再利用された CLI バックエンドのセッション ID を破棄して、起動処理を再実行します |
    | `/name <title>` | 現在のセッションに名前を付けるか、名前を変更します。タイトルを省略すると、現在の名前と候補が表示されます |
    | `/compact [instructions]` | セッションコンテキストを圧縮します。[Compaction](/ja-JP/concepts/compaction) を参照してください |
    | `/stop` | 現在の実行を中止します |
    | `/session idle <duration\|off>` | スレッドバインディングのアイドル有効期限を管理します |
    | `/session max-age <duration\|off>` | スレッドバインディングの最大期間による有効期限を管理します |
    | `/export-session [path]` | オーナーのみ使用できます。現在のセッションをワークスペース内の HTML にエクスポートします。エイリアス: `/export` |
    | `/export-trajectory [path]` | 現在のセッションの JSONL 軌跡バンドルをエクスポートします。エイリアス: `/trajectory` |

    明示的な `/export-session` パスを指定すると、ワークスペース内の既存ファイルが
    置き換えられます。パスを省略すると、名前の衝突を回避したファイル名が生成されます。

    <Note>
      Control UI は、入力された `/new` をインターセプトし、新しい
      ダッシュボードセッションを作成して切り替えます。ただし、`session.dmScope: "main"` が設定されていて、
      現在の親がエージェントのメインセッションである場合は、`/new` が
      メインセッションをその場でリセットします。入力された `/reset` は引き続き Gateway の
      インプレースリセットを実行します。固定されたセッションのモデル選択をクリアする場合は
      `/model default` を使用します。
    </Note>

  </Accordion>

  <Accordion title="モデルと実行の制御">
    | コマンド | 説明 |
    | --- | --- |
    | `/think <level\|default>` | 思考レベルを設定するか、セッションの上書きを解除します。エイリアス: `/thinking`、`/t` |
    | `/verbose on\|off\|full` | 詳細出力を切り替えます。エイリアス: `/v` |
    | `/trace on\|off` | 現在のセッションの Plugin トレース出力を切り替えます |
    | `/fast [status\|auto\|on\|off\|default]` | 高速モードを表示、設定、または解除します |
    | `/reasoning [on\|off\|stream]` | 推論の表示／非表示を切り替えます。エイリアス: `/reason` |
    | `/elevated [on\|off\|ask\|full]` | 昇格モードを切り替えます。エイリアス: `/elev` |
    | `/exec host=<auto\|sandbox\|gateway\|node> security=<deny\|allowlist\|full> ask=<off\|on-miss\|always> node=<id>` | exec のデフォルトを表示または設定します |
    | `/login [codex\|openai\|openai-codex]` | プライベートチャットまたは Web UI セッションから Codex/OpenAI ログインをペアリングします。オーナーまたは管理者のみ使用できます |
    | `/model [name\|#\|status]` | モデルを表示または設定します |
    | `/models [provider] [page] [limit=<n>\|all]` | 設定済み、または認証により利用可能なプロバイダーやモデルを一覧表示します |
    | `/queue <mode>` | アクティブな実行キューの動作を管理します。[キュー](/ja-JP/concepts/queue) および [キューステアリング](/ja-JP/concepts/queue-steering) を参照してください |
    | `/steer <message>` | アクティブな実行に指示を挿入します。エイリアス: `/tell`。[ステアリング](/ja-JP/tools/steer) を参照してください |

    <AccordionGroup>
      <Accordion title="verbose / trace / fast / reasoning の安全性">
        - `/verbose` はデバッグ用です。通常の使用時は**オフ**にしてください。
        - `/trace` は Plugin が所有するトレース／デバッグ行のみを表示します。通常の詳細メッセージはオフのままです。
        - `/fast auto|on|off` はセッションの上書きを永続化します。解除するには Sessions UI の `inherit` オプションを使用します。
        - `/fast` はプロバイダー固有です。OpenAI/Codex では `service_tier=priority` に、Anthropic への直接リクエストでは `service_tier=auto` または `standard_only` にマッピングされます。
        - `/reasoning`、`/verbose`、`/trace` はグループ環境では危険です。内部推論や Plugin の診断情報が公開される可能性があります。グループチャットではオフにしてください。

      </Accordion>
      <Accordion title="モデル切り替えの詳細">
        - `/model` は新しいモデルを即座にセッションへ永続化します。
        - エージェントがアイドル状態の場合、次回の実行から直ちに使用されます。
        - 実行中の場合、切り替えは保留としてマークされ、次の正常な再試行ポイントで適用されます。

      </Accordion>
    </AccordionGroup>

  </Accordion>

  <Accordion title="検出とステータス">
    | コマンド | 説明 |
    | --- | --- |
    | `/help` | 短いヘルプ概要を表示します |
    | `/commands` | 生成されたコマンドカタログを表示します |
    | `/tools [compact\|verbose]` | 現在のエージェントが現時点で使用できるものを表示します |
    | `/status` | 実行／ランタイムのステータス、Gateway とシステムの稼働時間、Plugin の健全性、およびプロバイダーの使用量／クォータを表示します |
    | `/status plugins` | Plugin の詳細な健全性を表示します。読み込みエラー、隔離、チャンネル Plugin の障害、依存関係の問題、互換性に関する通知が含まれます。`commands.plugins: true` が必要です |
    | `/goal [status\|start\|edit\|pause\|resume\|complete\|block\|clear] ...` | 現在のセッションの永続的な[目標](/ja-JP/tools/goal)を管理します |
    | `/diagnostics [note]` | オーナー専用のサポートレポートフローです。毎回 exec の承認を求めます |
    | `/openclaw <request>` | オーナーの DM から OpenClaw のセットアップおよび修復ヘルパーを実行します |
    | `/tasks` | 現在のセッションのアクティブまたは最近のバックグラウンドタスクを一覧表示します |
    | `/context [list\|detail\|map\|json]` | コンテキストがどのように構成されるかを説明します |
    | `/whoami` | 送信者 ID を表示します。エイリアス: `/id` |
    | `/usage off\|tokens\|full\|reset\|cost` | 応答ごとの使用量フッターを制御するか（`reset`/`inherit`/`clear`/`default` はセッションの上書きを解除し、設定済みのデフォルトを再継承します）、ローカルのコスト概要を出力します |
  </Accordion>

  <Accordion title="Skills、許可リスト、承認">
    | コマンド | 説明 |
    | --- | --- |
    | `/skill <name> [input]` | 名前を指定してスキルを実行 |
    | `/learn [request]` | 現在の会話または指定したソースから、レビュー可能なスキルを1つ[スキルワークショップ](/ja-JP/tools/skill-workshop)で下書き |
    | `/allowlist [list\|add\|remove] ...` | 許可リストのエントリを管理。テキストのみ |
    | `/approve <id> <decision>` | exec または Plugin の承認プロンプトを処理 |
    | `/btw <question>` | セッションコンテキストを変更せずに補足質問をする。エイリアス: `/side`。[BTW](/ja-JP/tools/btw)を参照 |
  </Accordion>

  <Accordion title="サブエージェントと ACP">
    | コマンド | 説明 |
    | --- | --- |
    | `/subagents list\|log\|info` | 現在のセッションのサブエージェント実行を確認 |
    | `/acp spawn\|cancel\|steer\|close\|sessions\|status\|set-mode\|set\|cwd\|permissions\|timeout\|model\|reset-options\|doctor\|install\|help` | ACP セッションとランタイムオプションを管理。ランタイム制御には外部オーナーまたは内部 Gateway 管理者のアイデンティティが必要 |
    | `/focus <target>` | 現在の Discord スレッドまたは Telegram トピックをセッションターゲットにバインド |
    | `/unfocus` | 現在のスレッドのバインドを解除 |
    | `/agents` | 現在のセッションでスレッドにバインドされているエージェントを一覧表示 |
  </Accordion>

  <Accordion title="オーナー限定の書き込みと管理">
    | コマンド | 要件 | 説明 |
    | --- | --- | --- |
    | `/config show\|get\|set\|unset` | `commands.config: true` | `openclaw.json` を読み書き。オーナー限定 |
    | `/mcp show\|get\|set\|unset` | `commands.mcp: true` | OpenClaw が管理する MCP サーバー設定を読み書き。オーナー限定 |
    | `/plugins list\|inspect\|show\|get\|install\|enable\|disable` | `commands.plugins: true` | Plugin の状態を確認または変更。書き込みはオーナー限定。エイリアス: `/plugin` |
    | `/debug show\|set\|unset\|reset` | `commands.debug: true` | ランタイム限定の設定オーバーライド。オーナー限定 |
    | `/restart` | `commands.restart: true`（デフォルト） | OpenClaw を再起動 |
    | `/send on\|off\|inherit` | オーナー | 送信ポリシーを設定 |
  </Accordion>

  <Accordion title="音声、TTS、チャンネル制御">
    | コマンド | 説明 |
    | --- | --- |
    | `/tts on\|off\|status\|chat\|latest\|provider\|limit\|summary\|audio\|help` | TTS を制御。[TTS](/ja-JP/tools/tts)を参照 |
    | `/activation mention\|always` | グループのアクティベーションモードを設定 |
    | `/bash <command>` | ホストのシェルコマンドを実行。エイリアス: `! <command>`。`commands.bash: true` が必要 |
    | `!poll [sessionId]` | バックグラウンドの bash ジョブを確認 |
    | `!stop [sessionId]` | バックグラウンドの bash ジョブを停止 |
  </Accordion>
</AccordionGroup>

### ドックコマンド

ドックコマンドは、アクティブなセッションの返信経路を、リンク済みの別のチャンネルへ切り替えます。
セットアップとトラブルシューティングについては、[チャンネルのドッキング](/ja-JP/concepts/channel-docking)を参照してください。

ネイティブコマンドをサポートするチャンネル Plugin から生成されます。

- `/dock-discord`（エイリアス: `/dock_discord`）
- `/dock-mattermost`（エイリアス: `/dock_mattermost`）
- `/dock-slack`（エイリアス: `/dock_slack`）
- `/dock-telegram`（エイリアス: `/dock_telegram`）

ドックコマンドには `session.identityLinks` が必要です。送信元の送信者とターゲットのピアは、
同じアイデンティティグループに属している必要があります。

### バンドルされた Plugin のコマンド

| コマンド                                                 | 説明                                                                                                                                                                                    |
| ------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `/dreaming [on\|off\|status\|help]`                     | メモリの Dreaming を切り替え（オーナーまたは Gateway 管理者）。[Dreaming](/ja-JP/concepts/dreaming)を参照                                                                                                            |
| `/pair [qr\|status\|pending\|approve\|cleanup\|notify]` | デバイスのペアリングを管理。[ペアリング](/ja-JP/channels/pairing)を参照                                                                                                                                        |
| `/phone status\|arm ...\|disarm`                        | リスクの高い Node コマンド（カメラ／画面／コンピューター／書き込み）を一時的に許可。[コンピューター操作](/ja-JP/nodes/computer-use)を参照                                                                               |
| `/voice status\|list\|set <voiceId>`                    | Talk の音声設定を管理。Discord でのネイティブ名: `/talkvoice`                                                                                                                                    |
| `/card ...`                                             | LINE のリッチカードプリセットを送信。[LINE](/ja-JP/channels/line)を参照                                                                                                                                        |
| `/codex <action> ...`                                   | Codex app-server ハーネスのバインド、制御、確認（ステータス、スレッド、再開、モデル、高速化、権限、コンパクト化、レビュー、mcp、Skills など）。[Codex ハーネス](/ja-JP/plugins/codex-harness)を参照 |

QQBot 限定: `/bot-ping`、`/bot-version`、`/bot-help`、`/bot-upgrade`、`/bot-logs`

### スキルコマンド

ユーザーが呼び出せるスキルは、スラッシュコマンドとして公開されます。

- `/skill <name> [input]` は汎用エントリポイントとして常に使用できます。
- Skills は直接コマンドとして登録できます（例: OpenProse の `/prose`）。
- ネイティブのスキルコマンド登録は `commands.nativeSkills` と
  `channels.<provider>.commands.nativeSkills` で制御されます。
- 名前は `a-z0-9_` にサニタイズされ（最大32文字）、重複時には数字のサフィックスが付きます。

<AccordionGroup>
  <Accordion title="スキルコマンドのディスパッチ">
    デフォルトでは、スキルコマンドは通常のリクエストとしてモデルにルーティングされます。

    Skills は `command-dispatch: tool` を宣言することで、ツールへ直接ルーティングできます
    （決定論的で、モデルは関与しません）。例: `/prose`（OpenProse Plugin）
    — [OpenProse](/ja-JP/prose)を参照してください。

  </Accordion>
  <Accordion title="ネイティブコマンドの引数">
    Discord は、必須の引数が省略された場合、動的オプションにはオートコンプリートを、
    ボタンメニューには必要に応じてボタンを使用します。Telegram と Slack は、選択肢のある
    コマンドにボタンメニューを表示します。動的な選択肢はターゲットセッションのモデルに
    基づいて解決されるため、`/think` レベルのようなモデル固有のオプションは、
    セッションの `/model` オーバーライドに従います。
  </Accordion>
</AccordionGroup>

## `/tools`: エージェントが現在使用できるもの

`/tools` は、ランタイムに関する質問、つまり**この会話でこのエージェントが今すぐ
使用できるもの**に答えます。静的な設定カタログではありません。

```text
/tools         # コンパクト表示
/tools verbose # 短い説明付き
```

結果はセッション単位です。エージェント、チャンネル、スレッド、送信者の
認可、またはモデルを変更すると、出力が変わる場合があります。プロファイルとオーバーライドを編集するには、
Control UI の Tools パネルまたは設定サーフェスを使用してください。

## `/model`: モデルの選択

```text
/model             # モデルピッカーを表示
/model list        # 同上
/model 3           # ピッカーの番号で選択
/model openai/gpt-5.4
/model opus@anthropic:default
/model default     # セッションのモデル選択をクリア
/model status      # エンドポイントと API モードを含む詳細表示
```

Discord では、`/model` と `/models` を使用すると、プロバイダーと
モデルのドロップダウンを備えた対話型ピッカーが開きます。ピッカーは
`provider/*` エントリを含む `agents.defaults.modelPolicy.allow` に従います。明示的な許可リストがない場合、
モデルエントリとエイリアスによって選択が制限されることはありません。

## `/config`: ディスク上の設定への書き込み

<Note>
  オーナー限定。デフォルトでは無効です。`commands.config: true` で有効にしてください。
</Note>

```text
/config show
/config show channels.whatsapp.responsePrefix
/config get channels.whatsapp.responsePrefix
/config set channels.whatsapp.responsePrefix="[openclaw]"
/config unset channels.whatsapp.responsePrefix
```

設定は書き込み前に検証されます。無効な変更は拒否されます。`/config` の
更新内容は再起動後も保持されます。

## `/mcp`: MCP サーバー設定

<Note>
  オーナー限定。デフォルトでは無効です。`commands.mcp: true` で有効にしてください。
</Note>

```text
/mcp show
/mcp show context7
/mcp set context7={"command":"uvx","args":["context7-mcp"]}
/mcp unset context7
```

`/mcp` は、組み込みエージェントのプロジェクト設定ではなく、OpenClaw の設定に保存されます。
`/mcp show` は、認証情報を含むフィールド、認識された認証情報フラグの値、
および既知のシークレット形式の引数をマスキングします。グループから実行した場合、
設定はオーナーへ非公開で送信されます。オーナーへの非公開経路が利用できない場合、
コマンドはフェイルクローズし、ダイレクトチャットから再試行するようオーナーに求めます。

## `/debug`: ランタイム限定のオーバーライド

<Note>
  オーナー限定。デフォルトでは無効です。`commands.debug: true` で有効にしてください。
  オーバーライドは新しい設定の読み取りに即座に適用されますが、ディスクには**書き込まれません**。
</Note>

```text
/debug show
/debug set channels.whatsapp.responsePrefix="[openclaw]"
/debug set channels.whatsapp.allowFrom=["+1555","+4477"]
/debug unset channels.whatsapp.responsePrefix
/debug reset
```

## `/plugins`: Plugin の管理

<Note>
  書き込みはオーナー限定。デフォルトでは無効です。`commands.plugins: true` で有効にしてください。
</Note>

```text
/plugins
/plugins list
/plugin show context7
/plugins enable context7
/plugins disable context7
/plugins install clawhub:<package>
/plugins install npm:@openclaw/<official-package>
/plugins install npm:<package> --force
/plugins install git:<repository>@<ref> --force
```

`/plugins enable|disable` は Plugin の設定を更新し、新しいエージェントターンに向けて Gateway の
Plugin ランタイムをホットリロードします。`/plugins install` は Plugin のソースモジュールが
変更されたため、管理対象の Gateway を自動的に再起動します。信頼済みの ClawHub および
公式カタログからのインストールには、追加の確認は不要です。任意の npm、git、アーカイブ、
`npm-pack:`、ローカルパスのソースでは、来歴に関する警告が表示され、
ソースを確認した後、末尾に `--force` を付ける必要があります。このフラグは
ソースを確認したことを示し、既存のインストールの置き換えを許可しますが、
`security.installPolicy` やインストーラーのセキュリティチェックを回避するものではありません。
リスク警告がある ClawHub リリースでは、引き続きシェル限定の別フラグ
`--acknowledge-clawhub-risk` が必要です。マーケットプレイス、リンク済み、固定済みの
インストールも、引き続きシェル限定です。

## `/trace`: Plugin のトレース出力

```text
/trace          # 現在のトレース状態を表示
/trace on
/trace off
```

`/trace` は、完全な詳細モードを使わずに、セッション単位の Plugin のトレース／デバッグ行を
表示します。`/debug`（ランタイムオーバーライド）や `/verbose`（通常の
ツール出力）の代わりにはなりません。

## `/btw`: 補足質問

`/btw` は、現在のセッションコンテキストに関する簡単な補足質問です。エイリアス: `/side`。

```text
/btw 今、何をしているのですか？
/side メインの実行が続いている間に何が変わりましたか？
```

通常のメッセージとは異なり、次のように動作します。

- 現在のセッションを背景コンテキストとして使用します。
- Codex ハーネスのセッションでは、一時的な Codex サイドスレッドとして実行されます。
- 今後のセッションコンテキストは**変更しません**。
- トランスクリプト履歴には書き込まれません。

完全な動作については、[BTW の補足質問](/ja-JP/tools/btw)を参照してください。

## サーフェスに関する注記

<AccordionGroup>
  <Accordion title="サーフェスごとのセッションスコープ">
    - **テキストコマンド:** 通常のチャットセッションで実行されます（DM は `main` を共有し、グループには独自のセッションがあります）。
    - **Discord のネイティブコマンド:** `agent:<agentId>:discord:slash:<userId>`
    - **Slack のネイティブコマンド:** `agent:<agentId>:slack:slash:<userId>`（プレフィックスは `channels.slack.slashCommand.sessionPrefix` で設定可能）
    - **Telegram のネイティブコマンド:** `telegram:slash:<userId>`（`CommandTargetSessionKey` を介してチャットセッションをターゲットにします）
    - **`/login codex`** は、プライベートチャットまたは Web UI の応答経路を通じてのみデバイスのペアリングコードを送信します。Telegram のグループ／トピックから呼び出した場合、代わりにボットへ DM するようオーナーに求めます。
    - **`/stop`** は、現在の実行を中止するため、アクティブなチャットセッションをターゲットにします。

  </Accordion>
  <Accordion title="Slack 固有の事項">
    `channels.slack.slashCommand` は単一の `/openclaw` 形式のコマンドをサポートします。
    `commands.native: true` を使用する場合は、組み込みコマンドごとに Slack スラッシュコマンドを1つ作成します。
    Slack が `/status` を予約しているため、`/status` ではなく `/agentstatus` を登録してください。
    Slack メッセージ内では、テキスト `/status` も引き続き機能します。
  </Accordion>
  <Accordion title="高速処理とインラインショートカット">
    - 許可リストに登録された送信者からのコマンドのみのメッセージは、即座に処理されます（キューとモデルをバイパスします）。
    - インラインショートカット（`/help`、`/commands`、`/status`、`/whoami`）は通常のメッセージ内に埋め込んでも機能し、残りのテキストがモデルに渡される前に取り除かれます。
    - 未承認のコマンドのみのメッセージは通知なく無視されます。インラインの `/...` トークンはプレーンテキストとして扱われます。

  </Accordion>
  <Accordion title="引数に関する注意事項">
    - コマンドと引数の間には、任意で `:` を指定できます（`/think: high`、`/send: on`）。
    - `/new <model>` には、モデルのエイリアス、`provider/model`、またはプロバイダー名（あいまい一致）を指定できます。一致するものがない場合、そのテキストはメッセージ本文として扱われます。
    - `/allowlist add|remove` には `commands.config: true` が必要であり、チャンネルの `configWrites` に従います。

  </Accordion>
</AccordionGroup>

## プロバイダーの使用量とステータス

- **プロバイダーの使用量／割り当て**（例：「Claude の残り 80%」）は、使用量追跡が有効な場合、現在のモデルプロバイダーについて `/status` に表示されます。
- `/status` の**トークン／キャッシュ行**は、ライブセッションのスナップショットに十分な情報がない場合、最新のトランスクリプト使用量エントリにフォールバックできます。
- **実行とランタイムの違い：** `/status` は、有効なサンドボックスパスを `Execution` として、セッションの実行主体を `Runtime` として報告します。実行主体は、`OpenClaw Default`、`OpenAI Codex`、CLI バックエンド、または ACP バックエンドのいずれかです。
- **応答ごとのトークン数／コスト：** `/usage off|tokens|full` で制御します。
- `/model status` は使用量ではなく、モデル／認証／エンドポイントに関するものです。

## 関連項目

<CardGroup cols={2}>
  <Card title="Skills" href="/ja-JP/tools/skills" icon="puzzle-piece">
    skill のスラッシュコマンドが登録され、利用を制御される仕組み。
  </Card>
  <Card title="skill の作成" href="/ja-JP/tools/creating-skills" icon="hammer">
    独自のスラッシュコマンドを登録する skill を構築します。
  </Card>
  <Card title="BTW" href="/ja-JP/tools/btw" icon="comments">
    セッションコンテキストを変更せずに補足的な質問をします。
  </Card>
  <Card title="ステアリング" href="/ja-JP/tools/steer" icon="compass">
    実行中に `/steer` を使用してエージェントを誘導します。
  </Card>
</CardGroup>
