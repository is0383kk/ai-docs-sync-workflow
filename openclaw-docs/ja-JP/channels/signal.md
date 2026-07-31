---
read_when:
    - Signal サポートの設定
    - Signal の送受信のデバッグ
summary: signal-cli（ネイティブデーモンまたは bbernhard コンテナ）による Signal 対応、セットアップ方法、電話番号モデル
title: Signal
x-i18n:
    generated_at: "2026-07-26T08:53:15Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 744f817e425d378e9f3e160df534019a6fc865227eb3fc68959a12ad46c0b714
    source_path: channels/signal.md
    workflow: 16
---

Signal はダウンロード可能なチャンネル Plugin（`@openclaw/signal`）です。Gateway は HTTP 経由で `signal-cli` と通信します。ネイティブデーモン（JSON-RPC + SSE）または [bbernhard/signal-cli-rest-api](https://github.com/bbernhard/signal-cli-rest-api) コンテナ（REST + WebSocket）のいずれかを使用します。OpenClaw は libsignal を組み込みません。

## 番号モデル（最初にお読みください）

- Gateway は **Signal デバイス**、つまり `signal-cli` アカウントに接続します。
- ボットを**個人用の Signal アカウント**で実行すると、ループ防止のため自分自身のメッセージは無視されます。
- 「ボットにメッセージを送ると返信される」ようにするには、**ボット専用の別番号**を使用してください。

## インストール

```bash
openclaw plugins install @openclaw/signal
```

修飾なしの Plugin 指定では、最初に ClawHub を試し、次に npm にフォールバックします。`openclaw plugins install clawhub:@openclaw/signal` または `npm:@openclaw/signal` でソースを指定できます。`plugins install` によって Plugin が登録され、有効化されるため、別途 `enable` を実行する必要はありません。一般的なインストール規則については、[Plugin](/ja-JP/tools/plugin)を参照してください。

## クイックセットアップ

<Steps>
  <Step title="番号を選択">
    ボットには**別の Signal 番号**を使用してください（推奨）。
  </Step>
  <Step title="Plugin をインストール">
    ```bash
    openclaw plugins install @openclaw/signal
    ```
  </Step>
  <Step title="ガイド付きセットアップを実行">
    ```bash
    openclaw channels add
    ```
    ウィザードは `signal-cli` が `PATH` に存在するかを検出し、存在しない場合はインストールを提案します。Linux x86-64 では公式のネイティブ GraalVM ビルドをダウンロードし、macOS およびその他のアーキテクチャでは Homebrew 経由でインストールします。その後、ボット番号と `signal-cli` のパスを入力するよう求めます。

    非対話型セットアップでは、`openclaw channels add --channel signal` はボットの電話番号を指定する `--signal-number <e164>` に加え、Signal デーモンのエンドポイント（デフォルトは `127.0.0.1:8080`）を指定する `--http-host <host>` と `--http-port <port>` も受け付けます。

  </Step>
  <Step title="アカウントをリンクまたは登録">
    - **QR リンク（最速）：** `signal-cli link -n "OpenClaw"` を実行し、Signal でスキャンします。[パス A](#setup-path-a-link-existing-signal-account-qr)を参照してください。
    - **SMS 登録：** 専用番号を使用し、captcha と SMS 認証を行います。[パス B](#setup-path-b-register-dedicated-bot-number-sms-linux)を参照してください。

  </Step>
  <Step title="検証してペアリング">
    ```bash
    openclaw gateway call channels.status --params '{"probe":true}'
    ```
    最初の DM を送信し、ペアリングを承認します：`openclaw pairing approve signal <CODE>`。
  </Step>
</Steps>

最小構成：

```json5
{
  channels: {
    signal: {
      enabled: true,
      account: "+15551234567",
      transport: {
        kind: "managed-native",
        cliPath: "signal-cli",
      },
      dmPolicy: "pairing",
      allowFrom: ["+15557654321"],
    },
  },
}
```

| フィールド       | 説明                                       |
| ----------- | ------------------------------------------------- |
| `account`   | E.164 形式のボット電話番号（`+15551234567`） |
| `transport` | アカウントが所有する Signal 接続とプロセスモード  |
| `dmPolicy`  | DM アクセスポリシー（`pairing` を推奨）          |
| `allowFrom` | DM を許可する電話番号または `uuid:<id>` 値 |

複数アカウントのサポート：アカウントごとの設定と任意の `name` を含む `channels.signal.accounts` を使用します。名前付きアカウントはそれぞれ固有の `transport` を所有し、トップレベルのトランスポートを継承しません。トップレベルのトランスポートは、暗黙の `default` アカウントだけに属します。共通パターンについては、[複数アカウントのチャンネル](/ja-JP/gateway/config-channels#multi-account-all-channels)を参照してください。

## 概要

- 決定的なルーティング：返信は常に Signal に返されます。
- DM はエージェントのメインセッションを共有し、グループは分離されます（`agent:<agentId>:signal:group:<groupId>`）。
- デフォルトでは、Signal は `/config set|unset` によってトリガーされた設定更新を書き込む場合があります（`commands.config: true` が必要です）。`channels.signal.configWrites: false` で無効化できます。

## セットアップパス A：既存の Signal アカウントをリンク（QR）

1. `signal-cli`（JVM またはネイティブビルド）をインストールするか、`openclaw channels add` にインストールさせます。
2. ボットアカウントをリンクします：`signal-cli link -n "OpenClaw"` を実行し、Signal で QR をスキャンします。
3. Signal を設定し、Gateway を起動します。

## セットアップパス B：ボット専用番号を登録（SMS、Linux）

既存の Signal アプリアカウントをリンクする代わりに、ボット専用番号を使用する場合にこの手順を使用します。以下のフローは Ubuntu 24 でテスト済みです。

1. SMS（固定電話の場合は音声認証）を受信できる番号を用意します。ボット専用番号を使用すると、アカウントやセッションの競合を回避できます。
2. Gateway ホストに `signal-cli` をインストールします：

```bash
VERSION=$(curl -Ls -o /dev/null -w %{url_effective} https://github.com/AsamK/signal-cli/releases/latest | sed -e 's/^.*\/v//')
curl -L -O "https://github.com/AsamK/signal-cli/releases/download/v${VERSION}/signal-cli-${VERSION}-Linux-native.tar.gz"
sudo tar xf "signal-cli-${VERSION}-Linux-native.tar.gz" -C /opt
sudo ln -sf /opt/signal-cli /usr/local/bin/
signal-cli --version
```

JVM ビルド（`signal-cli-${VERSION}.tar.gz`）を使用する場合は、先に JRE をインストールしてください。`signal-cli` は常に最新の状態に保ってください。Signal サーバー API の変更により、古いリリースが動作しなくなる可能性があるとアップストリームで説明されています。

3. 番号を登録して認証します：

```bash
signal-cli -a +<BOT_PHONE_NUMBER> register
```

captcha が必要な場合（この手順を完了するにはブラウザへのアクセスが必要です）：

1. `https://signalcaptchas.org/registration/generate.html` を開きます。
2. captcha を完了し、「Open Signal」から `signalcaptcha://...` リンクのターゲットをコピーします。
3. 可能であれば、ブラウザセッションと同じ外部 IP から実行してください（captcha トークンはすぐに期限切れになります）。
4. 直ちに登録して認証します：

```bash
signal-cli -a +<BOT_PHONE_NUMBER> register --captcha '<SIGNALCAPTCHA_URL>'
signal-cli -a +<BOT_PHONE_NUMBER> verify <VERIFICATION_CODE>
```

4. OpenClaw を設定し、Gateway を再起動して、チャンネルを検証します：

```bash
# Gateway をユーザー systemd サービスとして実行している場合：
systemctl --user restart openclaw-gateway.service

# 次に検証：
openclaw doctor
openclaw channels status --probe
```

5. DM の送信者をペアリングします：
   - ボット番号に任意のメッセージを送信します。
   - サーバーで承認します：`openclaw pairing approve signal <PAIRING_CODE>`。
   - 「Unknown contact」と表示されないように、ボット番号をスマートフォンの連絡先に保存します。

<Warning>
`signal-cli` で電話番号アカウントを登録すると、その番号のメイン Signal アプリセッションが認証解除される場合があります。ボット専用番号を使用するか、既存のスマートフォンアプリの設定を維持するために QR リンクモードを使用してください。
</Warning>

アップストリームの参考資料：

- `signal-cli` README：`https://github.com/AsamK/signal-cli`
- Captcha フロー：`https://github.com/AsamK/signal-cli/wiki/Registration-with-captcha`
- リンクフロー：`https://github.com/AsamK/signal-cli/wiki/Linking-other-devices-(Provisioning)`

## 外部ネイティブデーモンモード

`signal-cli` を自分で管理する場合（JVM のコールドスタートが遅い、コンテナの初期化、共有 CPU など）は、デーモンを別途実行し、OpenClaw が接続するように指定します：

非対話型セットアップでは、必要に応じてエンドポイントの種類を明示的に選択します：

```bash
openclaw channels add --channel signal --signal-number +15551234567 \
  --http-url http://127.0.0.1:8080 --signal-transport external-native
```

```json5
{
  channels: {
    signal: {
      transport: {
        kind: "external-native",
        url: "http://127.0.0.1:8080",
      },
    },
  },
}
```

この場合、自動起動と OpenClaw の起動待機はスキップされます。起動が遅い管理対象デーモンでは、`channels.signal.transport.startupTimeoutMs` を設定してください。

## コンテナモード（bbernhard/signal-cli-rest-api）

`signal-cli` をネイティブで実行する代わりに、`signal-cli` を REST + WebSocket インターフェースでラップする [bbernhard/signal-cli-rest-api](https://github.com/bbernhard/signal-cli-rest-api) Docker コンテナを使用します。

```bash
openclaw channels add --channel signal --signal-number +15551234567 \
  --http-url http://signal-cli:8080 --signal-transport container
```

要件：

- リアルタイムでメッセージを受信するには、コンテナを `MODE=json-rpc` で実行する**必要があります**。
- OpenClaw に接続する前に、コンテナ内で Signal アカウントを登録またはリンクしてください。

`docker-compose.yml` サービスの例：

```yaml
signal-cli:
  image: bbernhard/signal-cli-rest-api:latest
  environment:
    MODE: json-rpc
  ports:
    - "8080:8080"
  volumes:
    - signal-cli-data:/home/.local/share/signal-cli
```

OpenClaw の設定：

```json5
{
  channels: {
    signal: {
      enabled: true,
      account: "+15551234567",
      transport: {
        kind: "container",
        url: "http://signal-cli:8080",
      },
    },
  },
}
```

`transport.kind` は、OpenClaw が使用するプロトコルとプロセスのライフサイクルを制御します：

| 値               | 動作                                                                                                                                                     |
| ------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `"managed-native"`  | ネイティブの signal-cli を起動し、`/api/v1/rpc` の JSON-RPC と `/api/v1/events` の SSE を使用します。`url` では、デーモンのバインド先とは異なる接続エンドポイントを選択できます |
| `"external-native"` | すでに実行中のネイティブ signal-cli デーモンに接続します                                                                                                       |
| `"container"`       | `/v2/send` の bbernhard REST と `/v1/receive/{account}` の WebSocket に接続します                                                                             |

セットアップと `openclaw doctor --fix` では、具体的な種類を識別するために、既存のエンドポイントを一度プローブする場合があります。ランタイム操作では、プロトコルの自動検出や切り替えは行いません。

コンテナモードでは、コンテナが対応する API を公開している場合、ネイティブモードと同じ Signal 操作をサポートします。これには、送信、受信、添付ファイル、入力中インジケーター、既読・閲覧済み通知、リアクション、グループ、スタイル付きテキストが含まれます。OpenClaw は、`group.{base64(internal_id)}` グループ ID や書式付きテキスト用の `text_mode: "styled"` を含め、ネイティブ Signal RPC 呼び出しをコンテナの REST ペイロードに変換します。

運用上の注意：

- 受信には `MODE=json-rpc` を使用してください。`MODE=normal` では `/v1/about` が正常に見える場合がありますが、`/v1/receive/{account}` は WebSocket にアップグレードされないため、コンテナの受信ストリーミングはプローブに失敗します。
- bbernhard REST API には `kind: "container"` を設定し、ネイティブ `signal-cli` の JSON-RPC/SSE には `kind: "external-native"` を設定します。
- コンテナの添付ファイルダウンロードには、ネイティブモードと同じメディアバイト制限が適用されます。サーバーが `Content-Length` を送信する場合、サイズ超過のレスポンスは完全にバッファリングされる前に拒否され、それ以外の場合はストリーミング中に拒否されます。

## アクセス制御（DM + グループ）

DM：

- デフォルト：`channels.signal.dmPolicy = "pairing"`。
- 不明な送信者にはペアリングコードが送られ、承認されるまでメッセージは無視されます（コードは 1 時間後に期限切れになります）。
- `openclaw pairing list signal` と `openclaw pairing approve signal <CODE>` を使用して承認します。
- ペアリングは Signal DM のデフォルトのトークン交換方式です。詳細：[ペアリング](/ja-JP/channels/pairing)
- UUID のみの送信者（`sourceUuid` 由来）は、`channels.signal.allowFrom` に `uuid:<id>` として保存されます。

グループ：

- `channels.signal.groupPolicy = open | allowlist | disabled`。
- `channels.signal.groupAllowFrom` は、`allowlist` が設定されている場合に、グループ返信をトリガーできるグループまたは送信者を制御します。エントリには、Signal グループ ID（raw、`group:<id>`、または `signal:group:<id>`）、送信者の電話番号、`uuid:<id>` の値、または `*` を指定できます。
- `channels.signal.groups["<group-id>" | "*"]` では、`requireMention`、`tools`、および `toolsBySender` を使用してグループの動作を上書きできます。
- マルチアカウント構成でアカウントごとに上書きするには、`channels.signal.accounts.<id>.groups` を使用します。
- `groupAllowFrom` を通じて Signal グループを許可リストに追加しても、それだけではメンション制限は無効になりません。明示的に設定された `channels.signal.groups["<group-id>"]` エントリは、`requireMention=true` が設定されていない限り、すべてのグループメッセージを処理します。
- `requireMention=true` を使用すると、Signal ネイティブの @メンションは、構造化されたメンションメタデータからボットアカウントの電話番号または `accountUuid` と照合されます。設定された `mentionPatterns` は、プレーンテキストのフォールバックとして残ります。
- ランタイムに関する注意: `channels.signal` が完全に欠落している場合、ランタイムはグループチェックに `groupPolicy="allowlist"` をフォールバックとして使用します（`channels.defaults.groupPolicy` が設定されている場合でも同様です）。

コンテキストを制限したメンション必須グループ:

```json5
{
  channels: {
    signal: {
      account: "+15551234567",
      accountUuid: "bot-signal-uuid",
      groupPolicy: "allowlist",
      groupAllowFrom: ["group:<signal-group-id>"],
      historyLimit: 8,
      groups: {
        "<signal-group-id>": { requireMention: true },
      },
    },
  },
  messages: {
    groupChat: {
      mentionPatterns: ["\\bopenclaw\\b"],
    },
  },
}
```

ボットへのメンションを含まない許可済みグループメッセージには応答せず、制限された保留履歴ウィンドウ内にのみ保持されます。その後、ネイティブの @メンションまたはフォールバックのテキストメンションによってボットがトリガーされると、OpenClaw はその直近のコンテキストを含め、同じグループに返信します。スキップされた添付ファイルの本体はダウンロードされません。保留中のコンテキストには、簡潔なメディアプレースホルダーとしてのみ表示される場合があります。

## 仕組み（動作）

- ネイティブモード: `signal-cli` はデーモンとして実行され、Gateway は SSE 経由でイベントを読み取ります。
- コンテナモード: Gateway は REST API 経由で送信し、WebSocket 経由で受信します。
- 受信メッセージは共有チャネルエンベロープに正規化されます。
- 返信は常に同じ番号またはグループにルーティングされます。
- バックエンドが受信タイムスタンプと作成者を受け付ける場合、受信メッセージへの返信には Signal ネイティブの引用メタデータが含まれます。引用メタデータが欠落しているか拒否された場合、OpenClaw は通常のメッセージとして返信を送信します。
- ネイティブ引用の使用は `channels.signal.replyToMode = off | first | all | batched` で設定し、チャット種別ごとの上書きには `channels.signal.replyToModeByChatType.direct/group` を使用します。`channels.signal.accounts.<id>` 配下のアカウントレベルの値が優先されます。

## メディアと制限

- 送信テキストは `channels.signal.textChunkLimit` ごとに分割されます（デフォルトは 4000）。
- 任意の改行単位の分割: 長さによる分割の前に空行（段落境界）で分割するには、`channels.signal.streaming.chunkMode="newline"` を設定します。
- 添付ファイルがサポートされています（`signal-cli` から base64 を取得）。
- `contentType` が欠落している場合、ボイスメモの添付ファイルは `signal-cli` のファイル名を MIME のフォールバックとして使用するため、音声文字起こしで AAC ボイスメモを引き続き分類できます。
- デフォルトのメディア上限: `channels.signal.mediaMaxMb`（デフォルトは 8）。
- すべてのトランスポートでメディアのダウンロードをスキップするには、`channels.signal.ignoreAttachments` を使用します。
- グループ履歴コンテキストでは `channels.signal.historyLimit`（または `channels.signal.accounts.*.historyLimit`）を使用し、`messages.groupChat.historyLimit` にフォールバックします。無効にするには `0` を設定します（デフォルトは 50）。

## 入力中表示と既読通知

- **入力中インジケーター**: OpenClaw は `signal-cli sendTyping` 経由で入力中シグナルを送信し、返信の処理中はそれを更新します。
- **既読通知**: `channels.signal.sendReadReceipts` が true の場合、OpenClaw は許可された DM の既読通知を転送します。
- `signal-cli` はグループの既読通知を公開しません。

## ライフサイクルステータスのリアクション

受信ターンで、共有のキュー待ち／思考中／ツール／Compaction／完了／エラーのリアクションライフサイクルを Signal に表示させるには、`messages.statusReactions.enabled: true` を設定します。Signal は受信メッセージのタイムスタンプをリアクション対象として使用します。グループリアクションは、Signal グループ ID と元の送信者を対象作成者として指定して送信されます。

ステータスリアクションには、確認リアクションと、一致する `messages.ackReactionScope`（`direct`、`group-all`、`group-mentions`、または `all`）も必要です。Signal のステータスリアクションを無効にするには、`channels.signal.reactionLevel: "off"` を設定します。

Signal は、最終的な完了／エラー状態の後に最初の確認リアクションを復元します。

## リアクション（メッセージツール）

`message action=react` を `channel=signal` とともに使用します。

- 対象: 送信者の E.164 または UUID（ペアリング出力の `uuid:<id>` を使用します。プレフィックスなしの UUID も使用できます）。
- `messageId` は、リアクション対象メッセージの Signal タイムスタンプです。
- グループリアクションには `targetAuthor` または `targetAuthorUuid` が必要です。

```text
message action=react channel=signal target=uuid:123e4567-e89b-12d3-a456-426614174000 messageId=1737630212345 emoji=🔥
message action=react channel=signal target=+15551234567 messageId=1737630212345 emoji=🔥 remove=true
message action=react channel=signal target=signal:group:<groupId> targetAuthor=uuid:<sender-uuid> messageId=1737630212345 emoji=✅
```

設定:

- `channels.signal.actions.reactions`: リアクションアクションを有効化／無効化します（デフォルトは true）。
- `channels.signal.reactionLevel`: `off | ack | minimal | extensive`（デフォルトは `minimal`）。
  - `off`/`ack` はエージェントのリアクションを無効にします（メッセージツールの `react` はエラーになります）。
  - `minimal`/`extensive` はエージェントのリアクションを有効にし、ガイダンスレベルを設定します。
- アカウントごとの上書き: `channels.signal.accounts.<id>.actions.reactions`、`channels.signal.accounts.<id>.reactionLevel`。

## 承認リアクション

Signal の exec および Plugin 承認プロンプトは、トップレベルの `approvals.exec` および `approvals.plugin` ルーティングブロックを使用します。Signal には `channels.signal.execApprovals` ブロックはありません。

- `👍` は一度だけ承認します。
- `👎` は拒否します。
- リクエストに永続的な承認の選択肢がある場合は、`/approve <id> allow-always` を使用します。

承認リアクションの解決には、`channels.signal.allowFrom`、`channels.signal.defaultTo`、または一致するアカウントレベルのフィールドで明示的に指定された Signal 承認者が必要です。同じチャット内での直接的な exec 承認プロンプトでは、明示的な承認者がいなくても、重複するローカルの `/approve` フォールバックを非表示にできます。承認者がいないグループ承認では、ローカルフォールバックが引き続き表示されます。

## 質問リアクション

秘密ではない単一選択式の質問が 1 件、選択肢が 1～4 件ある `ask_user` プロンプトでは、Signal は選択肢ラベルの横に `1️⃣` から `4️⃣` を表示します。配信されたプロンプトに対応する数字でリアクションすると回答できます。OpenClaw は、リアクションがボットによって作成されたメッセージを対象としていることを検証し、その数字を Gateway 経由で正規の選択肢にマッピングします。古いタップや重複したタップは無視されます。複数質問、複数選択、および自由記述のプロンプトは、引き続きテキスト返信のみで回答できます。通常の Signal の DM／グループ受け入れルールにより送信者が認可されます。

## 配信先（CLI/Cron）

- DM: `signal:+15551234567`（またはプレフィックスなしの E.164）。
- UUID DM: `uuid:<id>`（またはプレフィックスなしの UUID）。
- グループ: `signal:group:<groupId>`。
- ユーザー名: `username:<name>`（使用している Signal アカウントでサポートされている場合）。

## エイリアス

繰り返し使用する Signal の配信先に、安定した名前のエイリアスを設定します。エイリアスは OpenClaw 側の設定にすぎず、Signal の連絡先を作成または編集するものではありません。

```json5
{
  channels: {
    signal: {
      aliases: {
        me: "+15557654321",
        jane: "uuid:123e4567-e89b-12d3-a456-426614174000",
        ops: "group:<groupId>",
      },
      defaultTo: "signal:me",
    },
  },
}
```

Signal の配信先を指定できる場所であれば、どこでもエイリアスを使用できます。

```bash
openclaw message send --channel signal --target signal:ops --message "デプロイが完了しました"
```

アカウントごとのエイリアスはトップレベルのエイリアスを継承し、名前を追加または上書きできます。

```json5
{
  channels: {
    signal: {
      aliases: {
        me: "+15557654321",
      },
      accounts: {
        work: {
          aliases: {
            ops: "group:<workGroupId>",
          },
        },
      },
    },
  },
}
```

`openclaw directory peers list --channel signal` および `openclaw directory groups list --channel signal` は、設定済みのエイリアスを一覧表示します。Signal ディレクトリは設定を基盤としており、Signal の連絡先をリアルタイムで照会したり、Signal アカウントを変更したりすることはありません。

## トラブルシューティング

まず次の手順を実行します。

```bash
openclaw status
openclaw gateway status
openclaw logs --follow
openclaw doctor
openclaw channels status --probe
```

必要に応じて、次に DM のペアリング状態を確認します。

```bash
openclaw pairing list signal
```

よくある問題:

- デーモンに到達できるが返信がない: `account`、`transport.kind`、トランスポート URL、および受信モードを確認してください。
- DM が無視される: 送信者のペアリング承認が保留中です。
- グループメッセージが無視される: グループ送信者またはメンションの制限によって配信がブロックされています。
- 編集後に設定検証エラーが発生する: `openclaw doctor --fix` を実行してください。
- 診断に Signal が表示されない: `channels.signal.enabled: true` を確認してください。

追加の確認:

```bash
openclaw pairing list signal
pgrep -af signal-cli
openclaw logs --plain --limit 500 | grep -i "signal" | tail -20
```

トリアージの流れについては、[チャネルのトラブルシューティング](/ja-JP/channels/troubleshooting)を参照してください。

## セキュリティに関する注意事項

- `signal-cli` はアカウントキーをローカルに保存します（通常は `~/.local/share/signal-cli/data/`）。
- サーバーを移行または再構築する前に、Signal アカウントの状態をバックアップしてください。
- DM へのアクセス範囲を明示的に広げる場合を除き、`channels.signal.dmPolicy: "pairing"` を維持してください。
- SMS 認証が必要なのは登録または復旧フローのみですが、番号やアカウントの制御を失うと、再登録が複雑になる可能性があります。

## 設定リファレンス（Signal）

完全な設定: [設定](/ja-JP/gateway/configuration)

プロバイダーオプション:

- `channels.signal.enabled`: チャネルの起動を有効化/無効化します。
- `channels.signal.account`: ボットアカウントの E.164。
- `channels.signal.accountUuid`: ネイティブな @メンション検出とループ防止に使用する、任意のボットアカウント UUID。
- `channels.signal.transport`: アカウント所有のトランスポート。管理対象のネイティブデフォルトを使用する場合は省略します。
- `channels.signal.transport.kind`: `managed-native | external-native | container`。
- `channels.signal.transport.url`: `external-native` と `container` では必須です。接続エンドポイントがデーモンのバインド先と異なる場合、`managed-native` では任意です。
- `channels.signal.transport.cliPath`: `signal-cli` への管理対象ネイティブパス。
- `channels.signal.transport.configPath`: 任意の管理対象ネイティブ `signal-cli --config` ディレクトリ。
- `channels.signal.transport.httpHost`, `channels.signal.transport.httpPort`: 管理対象ネイティブデーモンのバインド先（デフォルト `127.0.0.1:8080`）。
- `channels.signal.transport.startupTimeoutMs`: 管理対象ネイティブの起動待機時間（ミリ秒、最小 1000、上限 120000、デフォルト 30000）。
- `channels.signal.transport.receiveMode`: 管理対象ネイティブの `on-start | manual`。
- `channels.signal.ignoreAttachments`: このアカウントで受信添付ファイルのダウンロードをスキップします。
- `channels.signal.transport.ignoreStories`: 管理対象ネイティブのストーリー切り替え。
- `channels.signal.sendReadReceipts`: 既読通知を転送します。
- `channels.signal.dmPolicy`: `pairing | allowlist | open | disabled`（デフォルト: ペアリング）。
- `channels.signal.allowFrom`: DM 許可リスト（E.164 または `uuid:<id>`）。`open` には `"*"` が必要です。Signal にはユーザー名がないため、電話番号/UUID ID を使用してください。
- `channels.signal.aliases`: DM またはグループの配信先に対する OpenClaw 側のエイリアス。
- `channels.signal.groupPolicy`: `open | allowlist | disabled`（デフォルト: 許可リスト）。
- `channels.signal.groupAllowFrom`: グループ許可リスト。Signal グループ ID（未加工、`group:<id>`、または `signal:group:<id>`）、送信者の E.164 番号、または `uuid:<id>` 値を使用できます。
- `channels.signal.groups`: Signal グループ ID（または `"*"`）をキーとするグループ単位のオーバーライド。サポートされるフィールド: `requireMention`、`tools`、`toolsBySender`。
- `channels.signal.accounts.<id>.groups`: 複数アカウント構成向けの、アカウント単位の `channels.signal.groups`。
- `channels.signal.accounts.<id>.aliases`: アカウント単位のエイリアス。トップレベルのエイリアスとマージされます。
- `channels.signal.replyToMode`: ネイティブ返信の引用モード、`off | first | all | batched`（デフォルト: `all`）。
- `channels.signal.replyToModeByChatType.direct`, `channels.signal.replyToModeByChatType.group`: チャット種別ごとのネイティブ返信引用のオーバーライド。
- `channels.signal.accounts.<id>.replyToMode`, `channels.signal.accounts.<id>.replyToModeByChatType.direct`, `channels.signal.accounts.<id>.replyToModeByChatType.group`: アカウント単位の返信引用のオーバーライド。
- `channels.signal.historyLimit`: コンテキストに含めるグループメッセージの最大数（0 で無効化）。
- `channels.signal.dmHistoryLimit`: ユーザーターン数で指定する DM 履歴の上限。ユーザー単位のオーバーライド: `channels.signal.dms["<phone_or_uuid>"].historyLimit`。
- `channels.signal.textChunkLimit`: 送信チャンクの文字数（デフォルト 4000）。
- `channels.signal.streaming.chunkMode`: `length`（デフォルト）、または長さに基づくチャンク分割の前に空行（段落の境界）で分割する `newline`。
- `channels.signal.mediaMaxMb`: 受信/送信メディアの上限（MB、デフォルト 8）。
- `channels.signal.reactionLevel`: `off | ack | minimal | extensive`（デフォルト `minimal`）。[リアクション](#reactions-message-tool)を参照してください。
- `channels.signal.reactionNotifications`: `off | own | all | allowlist`（デフォルト `own`）- 他のユーザーからの受信リアクションがエージェントに通知される条件。
- `channels.signal.reactionAllowlist`: `reactionNotifications: "allowlist"` の場合に、リアクションによってエージェントへ通知する送信者。
- `channels.signal.streaming.block.enabled`, `channels.signal.streaming.block.coalesce`: チャネル間で共有されるブロックモードのストリーミング制御。[ストリーミング](/ja-JP/concepts/streaming)を参照してください。

関連するグローバルオプション:

- `agents.entries.*.groupChat.mentionPatterns`（プレーンテキストのフォールバック。ボットアカウントのアイデンティティが設定されている場合、Signal のネイティブ @メンションは構造化メタデータから検出されます）。
- `messages.groupChat.mentionPatterns`（グローバルフォールバック）。
- `channels.signal.responsePrefix` またはアカウントレベルの `responsePrefix`。

## 関連項目

- [チャネルの概要](/ja-JP/channels) - サポートされているすべてのチャネル
- [ペアリング](/ja-JP/channels/pairing) - DM 認証とペアリングの流れ
- [グループ](/ja-JP/channels/groups) - グループチャットの動作とメンションによる制御
- [チャネルルーティング](/ja-JP/channels/channel-routing) - メッセージのセッションルーティング
- [セキュリティ](/ja-JP/gateway/security) - アクセスモデルと堅牢化
