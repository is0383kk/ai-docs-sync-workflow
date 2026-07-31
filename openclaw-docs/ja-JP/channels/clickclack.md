---
read_when:
    - OpenClaw を ClickClack ワークスペースに接続する
    - ClickClack ボット ID のテスト
summary: ClickClack ボットトークンチャネルのセットアップとターゲット構文
title: ClickClack
x-i18n:
    generated_at: "2026-07-26T09:12:18Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 761538cdd7a916415719131b9ff2f40bf3e3e0eab0f7bda450250886acde8a64
    source_path: channels/clickclack.md
    workflow: 16
---

ClickClack は、正式にサポートされる ClickClack ボットトークンを介して、OpenClaw をセルフホスト型 ClickClack ワークスペースに接続します。

OpenClaw エージェントを ClickClack ボットユーザーとして表示する場合に使用します。ClickClack は、独立したサービスボットとユーザー所有のボットをサポートしています。ユーザー所有のボットは `owner_user_id` を保持し、付与したトークンスコープのみを受け取ります。

## クイックセットアップ

ClickClack で **Workspace settings → Integrations → OpenClaw** を開き、**Setup code (recommended)** を使用して
ボットを作成し、生成されたコマンドをコピーします。

```bash
openclaw channels add clickclack --code 'https://clickclack.example.com/#XXXX-XXXX-XXXX'
```

フロントエンドと API のオリジンが別々の場合や、API がパスにマウントされている場合、ClickClack は代わりに
正確なクレームエンドポイントを生成します。

```bash
openclaw channels add clickclack --code 'https://api.example.com/services/clickclack/api/bot-setup-codes/claim#XXXX-XXXX-XXXX'
```

セットアップコードは一度しか使用できず、10 分後に期限切れになります。OpenClaw はコードをクレームし、
新しく発行されたボットトークンとワークスペース設定を受け取り、アカウントを保存して
接続を検証し、実行中の Gateway がその設定を取得したかどうかを報告します。
バージョン付きの正確なエンドポイントでは、OpenClaw は ClickClack から返された正規の API
ベースを、パスプレフィックスも含めて検証し保存します。セットアップコード自体は
OpenClaw の設定には保存されません。

セットアップコードのクレームでは、公開サーバーに HTTPS を使用します。プレーン HTTP も、
`localhost` や `127.0.0.1` などのループバックアドレス上の
ローカルインストールでサポートされています。

OpenClaw がすでに実行中の場合、ClickClack は自動的に接続され、2 つ目の
コマンドは必要ありません。それ以外の場合は、次のコマンドで起動します。

```bash
openclaw gateway
```

コードをサーバー URL とは別に渡すこともできます。

```bash
openclaw channels add clickclack --code XXXX-XXXX-XXXX --base-url https://clickclack.example.com
```

ガイド付きセットアップを行うには、次を実行します。

```bash
openclaw onboard
```

ClickClack を選択し、プロンプトが表示されたらサーバー URL、ボットトークン、ワークスペースを
入力します。ガイド付きセットアップでは、保存後にサーバー、トークン、ワークスペースを確認します。
確認に失敗しても設定は破棄されません。

### 代替方法：手動トークン

OpenClaw 以外のクライアントを設定する場合や、
トークンを明示的に自分で管理する必要がある場合は、ClickClack で **Manual token** を選択します。

```bash
openclaw channels add clickclack --base-url https://clickclack.example.com --token ccb_... --workspace default
```

`workspace` には、ワークスペース ID（`wsp_...`）、スラッグ、または表示名を指定できます。
`--code` は、`--token`、`--token-file`、または `--use-env` と併用できません。

### 代替方法：環境変数ベースのトークン

デフォルトアカウントでは、トークンを設定に保存する代わりに `CLICKCLACK_BOT_TOKEN` を
読み取ることができます。

```bash
export CLICKCLACK_BOT_TOKEN="ccb_..."
openclaw channels add clickclack --base-url https://clickclack.example.com --workspace default --use-env
openclaw gateway
```

名前付きアカウントでは、設定済みのトークンまたはトークンファイルを使用する必要があります。共有環境
変数は意図的にデフォルトアカウントのみに制限されています。

### JSON5 リファレンス

同等の設定形式は次のとおりです。

```json5
{
  channels: {
    clickclack: {
      enabled: true,
      baseUrl: "https://clickclack.example.com",
      token: { source: "env", provider: "default", id: "CLICKCLACK_BOT_TOKEN" },
      workspace: "default",
      defaultTo: "channel:general",
    },
  },
}
```

アカウントが設定済みと見なされるのは、`baseUrl`、トークンソース、および
`workspace` がすべて設定されている場合のみです。デフォルトアカウントのトークンソースには、`token`、`tokenFile`、または
`CLICKCLACK_BOT_TOKEN` を使用できます。`workspace` には、ワークスペース
ID（`wsp_...`）、スラッグ、または名前を指定できます。Gateway は起動時にこれを ID に解決します。

### アカウント設定キー

| キー                     | デフォルト             | 備考                                                                                   |
| ----------------------- | ------------------- | --------------------------------------------------------------------------------------- |
| `baseUrl`               | なし（必須）     | ブラウザー向けリンクに使用する公開 ClickClack URL。                                    |
| `apiBaseUrl`            | `baseUrl`           | REST およびリアルタイム WebSocket 通信用の、サーバー間接続向けの任意のエンドポイント。             |
| `token`                 | なし                | プレーン文字列またはシークレット参照（`source: "env" \| "file" \| "exec"`）として指定するボットトークン。        |
| `tokenFile`             | なし                | ボットトークンファイルへのパス。`token` より優先されます。                                |
| `workspace`             | なし（必須）     | ワークスペース ID、スラッグ、または名前。                                                            |
| `replyMode`             | `"agent"`           | `"agent"` はエージェントパイプライン全体を実行し、`"model"` は短い直接モデル補完を送信します。 |
| `defaultTo`             | `"channel:general"` | 送信パスでターゲットが指定されていない場合に使用するターゲット。                                      |
| `allowFrom`             | `["*"]`             | 受信 DM およびチャンネルメッセージに対するユーザー ID の許可リスト。                                 |
| `botUserId`             | 自動検出       | 起動時にボットトークンの ID から解決されます。                                        |
| `agentId`               | ルートのデフォルト       | このアカウントの受信メッセージを 1 つのエージェントに固定します。                                       |
| `toolsAllow`            | なし                | このアカウントからのエージェント返信に使用できるツールの許可リスト。                                     |
| `model`, `systemPrompt` | なし                | `replyMode: "model"` 補完で使用されます。                                               |
| `commandMenu`           | `true`              | ClickClack コンポーザーのオートコンプリートにネイティブコマンドを公開します。                            |
| `reconnectMs`           | `1500`              | リアルタイム再接続の遅延（100～60000）。                                                |
| `discussions`           | 無効            | セッションごとに管理されるチャンネル設定。[セッションディスカッション](#session-discussions)を参照してください。  |

### 認証で保護された公開ホスト名を維持する

ClickClack と OpenClaw Gateway が同じホストで実行されているものの、
公開 ClickClack ホスト名が Cloudflare Access などの認証 Gateway で
保護されている場合は、`apiBaseUrl` を使用します。

```json5
{
  channels: {
    clickclack: {
      baseUrl: "https://clack.openclaw.ai",
      apiBaseUrl: "http://127.0.0.1:8484",
      token: { source: "env", provider: "default", id: "CLICKCLACK_BOT_TOKEN" },
      workspace: "default",
    },
  },
}
```

公開ホスト名は、ブラウザーユーザー向けに完全に認証で保護されたままにできます。OpenClaw は、
REST リクエスト、セットアップ検証、リアルタイム WebSocket にループバックエンドポイントを
使用します。一方で、ディスカッションの `embedUrl` と `openUrl` のリンクは引き続き
公開 `baseUrl` を使用します。`apiBaseUrl` を省略した場合、すべての通信で
`baseUrl` が使用され、既存の動作が維持されます。

`plugins.allow` が空でない制限付きリストの場合、チャンネルセットアップで
ClickClack を明示的に選択するか、`openclaw plugins enable clickclack` を実行すると、
そのリストに `clickclack` が追加されます。オンボーディングによるインストールでも、同じ
明示的選択の動作が使用されます。これらのパスは、`plugins.deny` や
グローバルな `plugins.enabled: false` 設定を上書きしません。直接
`openclaw plugins install @openclaw/clickclack` を実行した場合は、通常の
Plugin インストールポリシーに従い、既存の許可リストにも ClickClack を記録します。

## 複数のボット

各アカウントは独自の ClickClack リアルタイム接続を開き、独自のボットトークンを使用します。

```json5
{
  channels: {
    clickclack: {
      enabled: true,
      baseUrl: "https://clickclack.example.com",
      defaultAccount: "service",
      accounts: {
        service: {
          token: { source: "env", provider: "default", id: "CLICKCLACK_SERVICE_BOT_TOKEN" },
          workspace: "default",
          defaultTo: "channel:general",
          agentId: "service-bot",
        },
        support: {
          token: { source: "env", provider: "default", id: "CLICKCLACK_SUPPORT_BOT_TOKEN" },
          workspace: "default",
          defaultTo: "dm:usr_...",
          agentId: "support-bot",
        },
      },
    },
  },
}
```

## セッションディスカッション

1 つの ClickClack アカウントでディスカッションを有効にすると、各 OpenClaw セッションに
専用の ClickClack チャンネルが割り当てられます。アカウントトークンには
`channels:write` が含まれている必要があります（`bot:admin` バンドルには含まれています）。通常の `bot:write`
セットアップトークンでは、チャンネルを作成または同期できません。

```json5
{
  channels: {
    clickclack: {
      enabled: true,
      baseUrl: "https://clickclack.example.com",
      token: { source: "env", provider: "default", id: "CLICKCLACK_BOT_TOKEN" },
      workspace: "default",
      discussions: {
        enabled: true,
        workspace: "default",
        controlUrlBase: "https://team.openclaw.ai",
        section: "Sessions",
      },
    },
  },
}
```

`discussions.workspace` には、アカウントレベルの `workspace` と同じ
ワークスペース ID、スラッグ、または表示名を指定でき、デフォルトではその値が使用されます。`section` は
ClickClack サイドバーのセクションを制御し、デフォルトは `Sessions` です。
`controlUrlBase` が設定されている場合、管理対象チャンネルは実際の Control UI
セッションルートである `/chat?session=<encoded-session-key>` にリンクします。

ディスカッションは、ClickClack アカウントのうち正確に 1 つだけで有効にしてください。Gateway プロバイダーには
アカウントセレクターがないため、複数のディスカッションアカウントが有効になっている場合は、
設定順で 1 つを選択せずに拒否します。

ディスカッションを開くと、外部管理としてマークされた公開 ClickClack チャンネルが
作成されます。Plugin は、セッションのラベル、カテゴリ、アーカイブ状態の
同期を維持します。セッションを復元すると、そのチャンネルも復元されます。セッションカテゴリを
クリアすると、チャンネルは設定済みのデフォルトセクションに戻ります。
OpenClaw セッションを削除すると、ClickClack チャンネルは削除されずアーカイブされるため、
履歴は引き続き利用できます。Plugin は、ディスカッション RPC が
使用されたとき、およびバインディングが存在する間は約 1 分ごとに、バインディングを調整します。

管理対象チャンネルの受信メッセージは、接続されたメインセッションと同じ
エージェント ID の決定論的なサイドセッションを使用します。サイドエージェントには、監視対象の
メインセッションが通知され、`sessions_history` と `session_status` を使用できます
（差分確認には `changesSince` が便利です）。ディスカッションの参加者から、
メインセッションへの中継または誘導を求められた場合にのみ、`sessions_send` を使用します。
バインディング、管理対象の所有権参照、およびサイドセッションのピア ID には、
固定された ClickClack サーバーとチャンネルに加えて、具体的な OpenClaw セッション ID が含まれます。
再利用可能なセッションキーをリセットしたり、アカウントのターゲットを変更したりすると、
古いチャンネルはローカルで失効し、古い認証情報が引き続き使用可能な場合はアーカイブされ、
そのサイドトランスクリプトは再利用できなくなります。アーカイブ済み、リセット済み、無効化済み、
またはターゲット変更済みのバインディングを通じて届いたメッセージは、アカウントの通常の
チャンネルルーティングにフォールバックせず、破棄されます。解放されたバインディングには、
遅延したリアルタイムイベントに対して引き続きフェイルクローズとなるよう、永続的な
失効チャンネルマーカーが残ります。リモート所有権は ClickClack サーバーとチャンネル ID を
キーとするため、ローカルアカウントの名前を変更しても、管理対象チャンネルが通常のチャンネルに
変わることはありません。

`tools.sessions.visibility` は、より安全なデフォルト値 `tree` のままにしてください。Plugin は、
各サイドセッションとそれに接続されたメインセッションの間にのみホストスコープの許可を
設定し、さらにセッション探索とセッション間ターゲットをブロックするツールポリシーフックを
設定します。`sessions_history`、`session_status`、および
`sessions_send` は、接続されたメインセッションに対してのみ許可され、ステータス呼び出しによる
そのセッションのモデル変更を防止します。これらのツールは、エージェントの実効ツール許可リストにも
引き続き含まれている必要があります。システムプロンプトはガイダンスです。ホスト許可と
フックが認可境界です。

ClickClack サーバーは、チャンネルの作成および更新時に管理対象チャンネルのフィールド（`external_managed`、
`external_ref`、`external_url`、および `sidebar_section`）をサポートし、
チャンネルのレスポンスでそれらを返す必要があります。OpenClaw はバインディングを永続化する前に、
その契約を検証します。作成レスポンスが失われた場合、次回のオープン時には別のチャンネルを作成せず、
サーバーによって強制される `external_ref` に基づいてそのチャンネルを採用します。
その結果が調整されるまで、保留中の予約によって、送信先ワークスペース内の
他の方法では未バインドとなるイベントが隔離されます。粗粒度の調整処理は、
同じセッションがまだ有効な場合はチャンネルを採用し、リセット後はアーカイブします。
リモートチャンネルが作成されていない場合は予約を解除します。
この参照には、OpenClaw のインストールごとに永続化される名前空間に加えて、
セッションキー、具体的なセッション ID、ClickClack の送信先、および永続的な
バインディング世代のハッシュが含まれます。別々の Gateway が互いのチャンネルを採用することはできず、
リセットされたセッションが古いチャンネル履歴を継承することも、アカウントまたはワークスペースを
往復した後に以前のチャンネルを再採用することもできません。バインディングはさらに、
設定された ClickClack サーバー URL に固定され、アカウントの接続先が変更されると
無効になります。`controlUrlBase` を変更または削除すると、次回の調整処理で管理対象
チャンネルのリンクが更新または解除されます。
`discussions.workspace` を変更すると、古いワークスペースの認証情報が引き続き
設定されている場合、新しいワークスペースでチャンネルを開く前に古いバインディングが
アーカイブされ、解放されます。古いワークスペースにアクセスできないワークスペーススコープの
認証情報でトークンが置き換えられた場合、OpenClaw は置換後のトークンを試さずに
古いチャンネルを失効済みとして記録し、バインディングを解放します。残されたそのチャンネルは
ClickClack からアーカイブしてください。

接続されたメインセッションには、プル専用の `discussion` ツールも提供されます。
このツールは、最新のメッセージと最近のスレッド返信を、メッセージごとにエスケープされ、
帰属情報が付与された 1 件のレコードとして読み取り、書き込みやライフサイクル上の副作用はありません。
チャンネルルートとスレッドの検索には固定のリクエスト予算があり、その安全上の上限によって
以前からアクティブなスレッドが省略される可能性がある場合、結果に明示的な警告が表示されます。

## 返信モード

- `replyMode: "agent"`（デフォルト）は、セッション記録やツールポリシーを含む通常のエージェントパイプラインを通じて受信メッセージを処理します。
- `replyMode: "model"` はエージェントパイプラインを省略し、Plugin ランタイムの `llm.complete` を使用してボットが直接返信します。必要に応じて `model` と `systemPrompt` で形式を調整できます。選択されたプロバイダーとモデルが補完予算を管理します。

モデルモードは、解決されたボットのエージェント ID に対して補完を実行します。そのためには、
明示的な `plugins.entries.clickclack.llm.allowAgentIdOverride: true` 信頼
ビットが必要です。

```json5
{
  plugins: {
    entries: {
      clickclack: {
        llm: {
          allowAgentIdOverride: true,
        },
      },
    },
  },
}
```

デフォルトの `agent` 返信モードのみを使用する場合は、信頼ビットをオフのままにしてください。
そのモードでは必要ありません。

## コマンドメニュー

Gateway の起動時に、設定された各アカウントが OpenClaw のネイティブ
コマンドを ClickClack に公開します。これらは、ボットのハンドル名が付いた状態で
コンポーザーのオートコンプリートに表示されます。公開されるセットは起動のたびに
全体が置き換えられ、ネイティブコマンドカタログが空の場合は古いメニューも消去されます。

コマンドメニューの同期はデフォルトで有効です。無効にするには、アカウントで
`commandMenu: false` を設定します。

```json5
{
  channels: {
    clickclack: {
      enabled: true,
      token: { source: "env", provider: "default", id: "CLICKCLACK_BOT_TOKEN" },
      workspace: "default",
      commandMenu: false,
    },
  },
}
```

トークンには `commands:write` が必要です。現在の ClickClack の `bot:write` と
`bot:admin` のバンドルにはこのスコープが含まれており、個別に付与することもできます。
コマンドメニューが導入される前に作成されたトークンでは、スコープの追加または
トークンの置き換えが必要になる場合があります。

同期はベストエフォートで、Gateway の起動ごとに 1 回実行されます。スコープの不足やネットワーク
障害があると警告がログに記録され、エンドポイントを備えていない古い ClickClack サーバーの場合は
デバッグレベルで記録されます。これらの障害によってリアルタイム起動が妨げられることはありません。
メニューはエージェントがオフラインの間も利用可能で、ボットがワークスペースから退出すると削除されます。

このリリースでは、ネイティブコマンド仕様のみが公開されます。エイリアス、および
スキル、Plugin、カスタムコマンドのカタログはメニューに追加されません。ある名前が
HTTP スラッシュコマンドとしても登録されている場合、ClickClack はその登録を先に処理します。
その他のメニューコマンドは通常のメッセージ配信を通じて引き続き処理されます。

サービス間の相関を示す証拠には `agent` モードを使用します。正規の `msg_<ulid>`
形式を持つ正式な ClickClack メッセージ ID に対して、チャンネルは決定論的な
OpenClaw 実行 ID `clickclack:<message-id>` を導出します。各モデル呼び出しは、
診断で `clickclack:<message-id>:model:<n>` として確認でき、そのターンで ClawRouter を使用する場合は、
同じモデル呼び出し ID が `X-Request-ID` として送信されます。
`model` モードは通常のエージェント実行／セッション診断を迂回するため、
この証拠経路には適していません。

リアルタイムイベントに検証済みの `payload.correlation_id` が含まれる場合、
チャンネルは正式なメッセージ取得と、そこから生じる ClickClack の返信リクエストで、
それを `X-Correlation-ID` として引き継ぎます。値には ClickClack の安全な
128 文字セット（`A-Z`、`a-z`、`0-9`、`.`、`_`、`:`、および `-`）を使用します。無効な値は
省略されます。これらの結合には識別子のみが含まれ、メッセージ本文、
プロンプト、補完、認証情報、ツール出力は一切含まれません。

## 永続的なメディア配信

メディアを含むエージェントの返信では、永続的な配信が必須です。OpenClaw は、
ClickClack への最初の書き込み前に、パートごとの安定したメッセージおよびアップロードの nonce を
割り当てます。そのため、再試行時にはストレージ容量を消費したり重複を公開したりせず、
同じアップロードとメッセージを再利用します。再起動後にアップロードがすでに存在する場合、
OpenClaw は元のローカルパスやリモートメディア URL を再読み込みしません。

この復旧契約には、以下をサポートする ClickClack サーバーが必要です。

- `GET /api/uploads/by-nonce`。検出結果と未検出結果の両方で
  `X-ClickClack-Upload-Nonce: supported` を使用します。
- `GET /api/messages/by-nonce`。検出結果と未検出結果の両方で
  `X-ClickClack-Message-Nonce: supported` を使用します。
- 同じ所有者スコープの nonce とアップロードに対する、べき等なメッセージ作成および添付の関連付け。
  
古いサーバーの汎用的な 404 は、送信が存在しない証拠とは見なされません。
OpenClaw は重複の危険を冒す代わりに、配信を未解決のままにします。メディアを生成する
エージェント返信を有効にする前に ClickClack を更新してください。

## エージェントアクティビティ行

デフォルトでは、エージェントのターン実行中、ClickClack チャンネルには何も表示されず、最終返信のみが投稿されます。ターンの進行中に永続的な `agent_commentary` および `agent_tool` メッセージ行を公開するには、アカウントで `agentActivity: true` を設定します。

```json5
{
  channels: {
    clickclack: {
      enabled: true,
      token: { source: "env", provider: "default", id: "CLICKCLACK_BOT_TOKEN" },
      workspace: "default",
      agentActivity: true,
    },
  },
}
```

要件と動作：

- **デフォルトではオフです。** 標準設定および古い ClickClack サーバーには影響しません。
- **`agent_activity:write` トークンスコープが必要です。** このスコープは `bot:write` とは別であり、そこから継承されません。このオプションを有効にする前に、`--scopes bot:write,agent_activity:write` を指定してボットトークンを作成するか、既存のトークンにスコープを付与してください。
- **ベストエフォートで機能を縮退します。** トークンに `agent_activity:write` がない場合や、サーバーがアクティビティの書き込みを拒否した場合、障害はログに記録されますが、最終返信は引き続き正常に配信され、アクティビティ行は表示されません。
- 行はターンごと（`turn_id`）にグループ化され、1 つの論理ステップが 1 行になるようにまとめられます。ツール行には Discord／Slack／Telegram と同じ進捗表示形式（ツール名とコマンドの詳細）が使用されます。
- **帰属メタデータ。** エージェントが作成した投稿（アクティビティ行と最終返信）には、ターンで実際に使用されたモデル（フォールバック後を含む）から解決された `author_model` および `author_thinking` フィールドが付与されます。これらの列が定義されていないサーバーは未知の JSON フィールドを無視します。これらを永続化するサーバーでは、メッセージごとに「どのモデルが、どの思考レベルでこの行を生成したか」を確認できます。

## 送信先

- `channel:<name-or-id>` はワークスペースのチャンネルに送信します。修飾子のない送信先では、デフォルトで `channel:` が使用されます。
- `dm:<user_id>` は、そのユーザーとのダイレクト会話を作成または再利用します。
- `thread:<message_id>` は、そのメッセージをルートとするスレッドに返信します。

明示的な送信先には、`clickclack:` または `cc:` プロバイダープレフィックスを含めることもできます。

送信メディアでは ClickClack のアップロード API を使用し、作成されたチャンネルメッセージ、
スレッド返信、または DM に永続的なアップロードを添付します。ローカルファイルとサポート対象の
リモートメディア URL には、ファイルごとに 64 MiB の上限を設けた OpenClaw の通常の
メディアアクセス・ポリシーが適用されます。永続的なキュー送信では、各アップロードと
メッセージパートに別々の所有者スコープの nonce を使用し、その後、同じオブジェクトを使って
添付の関連付けを再試行します。サーバー契約と復旧時の動作については、
[永続的なメディア配信](#durable-media-delivery)を参照してください。

例：

```bash
openclaw message send --channel clickclack --target channel:general --message "hello"
openclaw message send --channel clickclack --target dm:usr_123 --message "hello"
openclaw message send --channel clickclack --target thread:msg_123 --message "following up"
```

## 権限

ClickClack のトークンスコープは ClickClack API によって適用されます。

- `bot:read`：ワークスペース、チャンネル、メッセージ、スレッド、DM、リアルタイム、プロフィールのデータを読み取ります。
- `bot:write`：`bot:read` に加えて、チャンネルメッセージ、スレッド返信、DM、アップロード、コマンドメニューの公開を許可します。
- `bot:admin`：`bot:write` に加えて、チャンネルの作成を許可します。
- `commands:write`：ボットのコマンドメニューを公開します。現在の `bot:write` および `bot:admin` バンドルに含まれ、個別に付与することもできます。
- `agent_activity:write`：永続的なエージェントアクティビティ行（`agent_commentary`／`agent_tool`）。`bot:write` または `bot:admin` からは継承されません。`agentActivity: true` が設定されている場合にのみ必要です。

通常のエージェントチャットとコマンドメニューの同期には、現在の `bot:write` のみが必要です。[エージェントアクティビティ行](#agent-activity-rows)を有効にする場合は、`agent_activity:write` を追加してください。

## トラブルシューティング

- `ClickClack is not configured for account "<id>"`：そのアカウントに `baseUrl`、`token`（たとえば `CLICKCLACK_BOT_TOKEN` 経由）、および `workspace` を設定します。
- `ClickClack workspace not found: <value>`：`workspace` を ClickClack が返したワークスペース ID、スラッグ、または名前に設定します。
- 受信返信がない：トークンにリアルタイム読み取りアクセス権があることを確認してください。また、ボットは自身のメッセージと他のボットからのメッセージを無視する点に注意してください。
- チャンネルへの送信に失敗する：ボットがワークスペースのメンバーであり、`bot:write` を持っていることを確認してください。
- コマンドメニューが表示されない：`commandMenu` が `false` ではないこと、ClickClack サーバーが `PUT /api/bots/self/commands` をサポートしていること、およびトークンに `commands:write` があることを確認してください。
