---
read_when:
    - チャネルアカウント（Discord、Google Chat、iMessage、Matrix、Signal、Slack、Telegram、WhatsApp など）を追加または削除する場合
    - チャンネルの状態を確認するか、チャンネルのログを追跡する場合
    - 失敗した受信チャネルイベントを調査または再送信する必要がある場合
summary: '`openclaw channels` の CLI リファレンス（アカウント、ステータス、デッドレター、機能、解決、ログ、ログイン／ログアウト）'
title: チャンネル
x-i18n:
    generated_at: "2026-07-26T10:08:04Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 8e5b7d674264af51d6fec34c8c95256129d66918b7c4515ac0f2c2bd311f2c3b
    source_path: cli/channels.md
    workflow: 16
---

# `openclaw channels`

Gateway 上でチャットチャネルのアカウントとその実行時ステータスを管理します。

関連ドキュメント:

- チャネルガイド: [チャネル](/ja-JP/channels)
- Gateway の設定: [設定](/ja-JP/gateway/configuration)

## 共通コマンド

```bash
openclaw channels list
openclaw channels list --all
openclaw channels status
openclaw channels capabilities
openclaw channels capabilities --channel discord --target channel:123
openclaw channels resolve --channel slack "#general" "@jane"
openclaw channels logs --channel all
openclaw channels dead-letters list --channel telegram --account default
```

`channels list` にはチャットチャネルのみが表示されます。デフォルトでは設定済みのアカウントが表示され、アカウントごとに `installed`、`configured`、`enabled` のステータスタグが付きます（機械処理用の出力には `--json` を使用します）。`--all` を指定すると、まだアカウントが設定されていないバンドル済みチャネルと、まだディスク上にないインストール可能なカタログチャネルも表示されます。プロバイダー認証とモデル使用量は別の場所で管理します。プロバイダー認証プロファイルには `openclaw models auth list`、使用量やクォータには `openclaw status` または `openclaw models list` を使用します。

## ステータス / 機能 / 解決 / ログ

- `channels status`: `--channel <name>`、`--probe`、`--timeout <ms>`（デフォルトは `10000`）、`--json`
- `channels capabilities`: `--channel <name>`、`--account <id>`（`--channel` が必要）、`--target <dest>`（`--channel` が必要）、`--timeout <ms>`（デフォルトは `10000`、上限は `30000`）、`--json`
- `channels resolve <entries...>`: `--channel <name>`、`--account <id>`、`--kind <auto|user|group>`（デフォルトは `auto`）、`--json`
- `channels logs`: `--channel <name|all>`（デフォルトは `all`）、`--lines <n>`（デフォルトは `200`）、`--json`

`channels status --probe` はライブパスです。到達可能な Gateway では、アカウントごとに
`probeAccount` と任意の `auditAccount` チェックを実行するため、出力にはトランスポートの
状態に加えて、`works`、`probe failed`、`audit ok`、`audit failed` などのプローブ結果が含まれる場合があります。
Gateway に到達できない場合、`channels status` はライブプローブ出力ではなく、
設定のみの概要にフォールバックします。

## 受信デッドレター

再試行ポリシーを使い切った受信イベントは、キューの既存の失敗エントリ保持期間中、共有状態データベースに残ります。次のコマンドで、1 つのチャネルアカウントを調査できます。

```bash
openclaw channels dead-letters list --channel telegram --account default
openclaw channels dead-letters list --channel telegram --account default --json
```

テキスト表示には、イベント ID、失敗理由、試行回数、失敗からの経過時間が表示されます。JSON 出力には、診断用として保持されたペイロード、メタデータ、レーン、試行タイムスタンプも含まれます。

根本的な問題を修正した後、元のイベント ID を使用してイベントを 1 件再キューイングします。

```bash
openclaw channels dead-letters resubmit <event-id> --channel telegram --account default
```

チャネルランタイムと同じ共有状態データベースにアクセスできるように、これらのコマンドは Gateway ホスト上で実行してください。再送信ではペイロード、メタデータ、レーンが保持されますが、試行カウンターとキュー内経過時間はリセットされます。この処理はイベントの失敗マーカーをアトミックに置き換えるため、イベントが保留中または取得済みの間にコマンドを繰り返すと、2 回目のディスパッチを作成せず拒否されます。実行中のチャネルは、次回の受信ドレイン時にそのイベントを取得します。完了済みイベントは終端状態のままであり、再送信できません。ペイロード保持機能が追加される前に作成された失敗行も一覧に表示される場合がありますが、ペイロードを利用できないため再送信は拒否されます。

`openclaw health` は、チャネルアカウントごとのデッドレター数と最も古い失敗の経過時間を報告します。`openclaw doctor` は影響を受けるアカウントを示し、調査コマンドを案内します。

`openclaw sessions`、Gateway の `sessions.list`、またはエージェントの
`sessions_list` ツールを、チャネルソケットの正常性を示すシグナルとして使用しないでください。これらのサーフェスが報告するのは
保存された会話行であり、プロバイダーの実行時状態ではありません。Discord プロバイダーの
再起動後、接続済みでも通信のないアカウントは正常である可能性があり、次の受信または送信会話イベントが発生するまで Discord セッション行が
表示されない場合があります。

## アカウントの追加 / 削除

```bash
openclaw channels add --channel telegram --token <bot-token>
openclaw channels add --channel nostr --private-key "$NOSTR_PRIVATE_KEY"
openclaw channels remove --channel telegram --delete
```

<Tip>
`openclaw channels add telegram --help` または `openclaw channels add --channel telegram --help` には、Telegram のセットアップフラグのみが表示されます。`openclaw channels add --help` には、共有コマンドエンベロープのみが表示されます。
</Tip>

`channels remove` は、インストール済みまたは設定済みのチャネル Plugin のみを操作します。インストール可能なカタログチャネルについては、先に `channels add` を使用してください。`--delete` を指定しない場合は、アカウントを無効にするか確認し、その設定を保持します。`--delete` を指定すると、確認なしで設定エントリが削除されます。
ランタイムに支えられたチャネル Plugin では、`channels remove` は設定を更新する前に、実行中の Gateway に対して選択したアカウントの停止も要求します。そのため、アカウントを無効化または削除しても、再起動まで古いリスナーが稼働し続けることはありません。

共有制御エンベロープに含まれるのは、`--channel`、`--account`、および任意のアカウント表示用 `--name` のみです。各モダンチャネル Plugin は、それぞれの認証情報、トランスポート、プロバイダー固有のセマンティクスを所有します。位置引数の ID または `--channel <id>` でチャネルが選択されると、CLI はチャネルのランタイムコードを読み込まず、バンドル済みまたはインストール済みの Plugin パッケージメタデータから、そのチャネルのオプションだけを構築します。

`--token`、`--url`、`--use-env` のような共通に見えるフラグも、モダンなコントラクトが処理する場合はチャネルが所有します。選択されたサードパーティー Plugin が引き続き従来の共有セットアップアダプターを使用している場合、コアはそのチャネルについてのみ、リリース済みの互換性フラグセットを従来の `cliAddOptions` とともに登録します。無関係な従来フィールドが他のチャネルに漏れることはなく、選択されたモダンチャネルは、自身が宣言していない互換性フラグを拒否します。

チャネル所有フラグの例:

| チャネル     | フラグ                                                                                                |
| ----------- | ---------------------------------------------------------------------------------------------------- |
| Google Chat | `--webhook-path`、`--webhook-url`、`--audience-type`、`--audience`                                   |
| iMessage    | `--cli-path`、`--db-path`、`--service`、`--region`                                                   |
| Matrix      | `--homeserver`、`--user-id`、`--access-token`、`--password`、`--device-name`、`--initial-sync-limit` |
| Nostr       | `--private-key`、`--relay-urls`                                                                      |
| Signal      | `--signal-number`、`--signal-transport`、`--cli-path`、`--http-url`、`--http-host`、`--http-port`    |
| Tlon        | `--ship`、`--url`、`--code`、`--group-channels`、`--dm-allowlist`、`--auto-discover-channels`        |
| WhatsApp    | `--auth-dir`                                                                                         |

フラグ駆動の追加コマンド中にチャネル Plugin のインストールが必要な場合、OpenClaw は対話型の Plugin インストールプロンプトを開かず、そのチャネルのデフォルトインストールソースを使用します。

ガイド付きセットアップとフラグ駆動セットアップはどちらも、選択されたチャネルのパーサー、検証、アカウント解決、設定ライター、書き込み後フックを通過します。サポートされていないフラグは、グローバル入力バッグを介して受け入れられるのではなく、所有チャネルのセットアップエラーで失敗します。

アカウント、認証情報、またはチャネル設定のフラグを直接指定せずに `openclaw channels add` を実行すると、対話型ウィザードが入力を求めることができます。位置引数のチャネル ID と `--channel <id>` はどちらも、ガイダンスを省略せずにそのチャネルを事前選択します。

```bash
openclaw channels add telegram
openclaw channels add --channel telegram
```

ウィザードは次の項目を入力するよう求めることができます。

- 選択したチャネルごとのアカウント ID
- それらのアカウントの任意の表示名
- `Route these channel accounts to agents now?`

ここでバインドすることを確認すると、ウィザードは設定済みの各チャネルアカウントをどのエージェントが所有するかを確認し、アカウントスコープのルーティングバインディングを書き込みます。

後から `openclaw agents bindings`、`openclaw agents bind`、`openclaw agents unbind` を使用して、同じルーティングルールを管理することもできます（[エージェント](/ja-JP/cli/agents)を参照）。

トップレベルの単一アカウント設定を引き続き使用しているチャネルにデフォルト以外のアカウントを追加すると、OpenClaw は新しいアカウントを書き込む前に、それらのトップレベル値をチャネルのアカウントマップへ昇格します。チャネルに名前付きアカウントが 1 つだけ存在する場合、または `defaultAccount` がそのいずれかを指している場合は、昇格時に既存の名前付きアカウントを再利用します。それ以外の場合、値は `channels.<channel>.accounts.default` に格納されます。

ルーティング動作の一貫性は維持されます。

- 既存のチャネルのみのバインディング（`accountId` なし）は、引き続きデフォルトアカウントに一致します。
- `channels add` は、非対話モードでバインディングを自動作成または書き換えません。
- 対話型セットアップでは、必要に応じてアカウントスコープのバインディングを追加できます。

設定がすでに混在状態（名前付きアカウントが存在し、トップレベルの単一アカウント値も引き続き設定されている状態）の場合は、`openclaw doctor --fix` を実行して、アカウントスコープの値をそのチャネル用に選択された昇格先アカウントへ移動します。

## ログインとログアウト（対話型）

```bash
openclaw channels login --channel whatsapp
openclaw channels logout --channel whatsapp
```

- `channels login` は `--account <id>` と `--verbose` をサポートし、`channels logout` は `--account <id>` をサポートします。
- `channels login` と `logout` は、その操作をサポートする設定済みチャネルが 1 つだけの場合、チャネルを推測できます。複数ある場合は `--channel` を指定してください。
- `channels logout` は、到達可能な場合はライブ Gateway パスを優先するため、チャネル認証状態を消去する前に、アクティブなリスナーが停止します。ローカル Gateway に到達できない場合は、ローカルの認証クリーンアップにフォールバックします。`gateway.mode: "remote"` を指定すると、代わりに Gateway エラーによってコマンドが失敗します。
- ログインに成功すると、CLI は到達可能なローカル Gateway にアカウントの起動を要求します。リモートモードでは認証をローカルに保存し、リモートランタイムが再起動されなかったことを通知します。
- Gateway ホスト上のターミナルから `channels login` を実行してください。エージェントの `exec` は、この対話型ログインフローをブロックします。利用可能な場合は、`whatsapp_login` などのチャネルネイティブなエージェントログインツールをチャットから使用してください。

## トラブルシューティング

- 広範なプローブには `openclaw status --deep` を実行します。
- ガイド付き修正には `openclaw doctor` を使用します。
- `openclaw channels status` は、Gateway に到達できない場合、設定のみの概要にフォールバックします。サポートされているチャネルの認証情報が SecretRef を介して設定されていても、現在のコマンドパスで利用できない場合、そのアカウントを未設定として表示するのではなく、機能低下に関する注記付きで設定済みとして報告します。

## 機能プローブ

プロバイダーの機能ヒント（利用可能な場合はインテント/スコープ）と静的な機能サポートを取得します。

```bash
openclaw channels capabilities
openclaw channels capabilities --channel discord --target channel:123
```

注記:

- `--channel` は省略可能です。省略すると、すべてのチャンネル（Plugin が提供するチャンネルを含む）が一覧表示されます。
- `--account` は `--channel` と組み合わせた場合にのみ有効です。
- `--target` には `channel:<id>` または数値のチャンネル ID を直接指定でき、Discord にのみ適用されます。Discord ボイスチャンネルの場合、権限チェックにより、`ViewChannel`、`Connect`、`Speak`、`SendMessages`、`ReadMessageHistory` の不足が報告されます。
- プローブはプロバイダー固有です。Discord では Bot の ID とインテント、および必要に応じてチャンネル権限、Slack では Bot とユーザーのスコープ、Telegram では Bot のフラグと Webhook、Signal ではデーモンのバージョン、Microsoft Teams ではアプリトークンと Graph のロール／スコープ（判明している場合は注釈付き）を確認します。プローブのないチャンネルでは `Probe: unavailable` が報告されます。

## 名前を ID に解決する

プロバイダーディレクトリを使用して、チャンネル名やユーザー名を ID に解決します。

```bash
openclaw channels resolve --channel slack "#general" "@jane"
openclaw channels resolve --channel discord "My Server/#support" "@someone"
openclaw channels resolve --channel matrix "Project Room"
```

注記：

- ターゲットの種類を強制するには `--kind user|group|auto` を使用します。
- 複数のエントリが同じ名前を共有している場合、解決ではアクティブな一致が優先されます。
- `channels resolve` は読み取り専用です。選択したアカウントが SecretRef を介して設定されていても、現在のコマンドパスでその認証情報を使用できない場合、コマンドは実行全体を中止せず、注記付きの機能低下した未解決結果を返します。
- `channels resolve` はチャンネル Plugin をインストールしません。インストール可能なカタログチャンネルの名前を解決する前に、`channels add --channel <name>` を使用してください。

## 関連項目

- [CLI リファレンス](/ja-JP/cli)
- [チャンネルの概要](/ja-JP/channels)
