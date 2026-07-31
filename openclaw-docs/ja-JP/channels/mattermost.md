---
read_when:
    - Mattermost のセットアップ
    - Mattermost ルーティングのデバッグ
sidebarTitle: Mattermost
summary: Mattermost ボットのセットアップと OpenClaw の設定
title: Mattermost
x-i18n:
    generated_at: "2026-07-26T09:53:16Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: ea41fb9a7e4e9ea6bd8d04a4f2c6d2d7f2e43cf71830e445f1e28e2e8737f3cb
    source_path: channels/mattermost.md
    workflow: 16
---

ステータス: ダウンロード可能なPlugin（ボットトークン + WebSocketイベント）。チャンネル、プライベートチャンネル、グループDM、DMに対応しています。Mattermostはセルフホスト可能なチームメッセージングプラットフォームです（[mattermost.com](https://mattermost.com)）。

## インストール

<Tabs>
  <Tab title="npmレジストリ">
    ```bash
    openclaw plugins install @openclaw/mattermost
    ```
  </Tab>
  <Tab title="ローカルチェックアウト">
    ```bash
    openclaw plugins install ./path/to/local/mattermost-plugin
    ```
  </Tab>
</Tabs>

詳細: [Plugin](/ja-JP/tools/plugin)

## クイックセットアップ

<Steps>
  <Step title="Pluginが利用可能であることを確認する">
    上記のコマンドで`@openclaw/mattermost`をインストールし、Gatewayがすでに実行中の場合は再起動します。
  </Step>
  <Step title="Mattermostボットを作成する">
    Mattermostボットアカウントを作成し、**ボットトークン**をコピーして、ボットが読み取る必要のあるチームとチャンネルに追加します。
  </Step>
  <Step title="ベースURLをコピーする">
    Mattermostの**ベースURL**（例: `https://chat.example.com`）をコピーします。末尾の`/api/v4`は自動的に削除されます。
  </Step>
  <Step title="OpenClawを設定してGatewayを起動する">
    最小構成:

    ```json5
    {
      channels: {
        mattermost: {
          enabled: true,
          botToken: "mm-token",
          baseUrl: "https://chat.example.com",
          dmPolicy: "pairing",
        },
      },
    }
    ```

    非対話形式の代替方法:

    ```bash
    openclaw channels add --channel mattermost --bot-token <token> --http-url https://chat.example.com
    ```

  </Step>
</Steps>

<Note>
プライベート/LAN/tailnetアドレス上でセルフホストされているMattermost: Mattermost APIへの送信リクエストは、デフォルトでプライベートIPと内部IPをブロックするSSRFガードを通過します。`channels.mattermost.network.dangerouslyAllowPrivateNetwork: true`でオプトインします（アカウントごと: `channels.mattermost.accounts.<id>.network.dangerouslyAllowPrivateNetwork`）。
</Note>

## ネイティブスラッシュコマンド

ネイティブスラッシュコマンドはオプトインです。有効にすると、OpenClawはボットが所属するすべてのチームに`oc_*`スラッシュコマンドを登録し、Gateway HTTPサーバーでコールバックPOSTを受信します。

```json5
{
  channels: {
    mattermost: {
      commands: {
        native: true,
        nativeSkills: true,
        callbackPath: "/api/channels/mattermost/command",
        // MattermostからGatewayへ直接到達できない場合に使用します（リバースプロキシ/公開URL）。
        callbackUrl: "https://gateway.example.com/api/channels/mattermost/command",
      },
    },
  },
}
```

登録されるコマンド: `/oc_status`、`/oc_model`、`/oc_models`、`/oc_new`、`/oc_help`、`/oc_think`、`/oc_reasoning`、`/oc_verbose`、`/oc_queue`。`nativeSkills: true`を指定すると、スキルコマンドも`/oc_<skill>`として登録されます。

<AccordionGroup>
  <Accordion title="動作に関する注意事項">
    - `native`と`nativeSkills`のデフォルトは`"auto"`で、Mattermostでは無効として解決されます。明示的に`true`に設定してください。
    - `callbackPath`のデフォルトは`/api/channels/mattermost/command`です。
    - `callbackUrl`を省略すると、OpenClawは`http://<gateway.customBindHost or localhost>:<gateway.port, default 18789><callbackPath>`を導出します。ワイルドカードバインドホスト（`0.0.0.0`、`::`）は`localhost`にフォールバックします。
    - マルチアカウント構成では、`commands`をトップレベルまたは`channels.mattermost.accounts.<id>.commands`の下に設定できます（アカウントの値がトップレベルのフィールドを上書きします）。
    - 他のインテグレーションによって作成された同じトリガーの既存スラッシュコマンドは変更されません（登録時にスキップされます）。ボットが作成したコマンドは、コールバックURLにずれが生じると更新または再作成されます。
    - コマンドコールバックは、OpenClawが`oc_*`コマンドを登録した際にMattermostから返されるコマンドごとのトークンで検証されます。
    - OpenClawは各コールバックを受け入れる前に現在のMattermostコマンド登録を更新するため、削除または再生成されたスラッシュコマンドの古いトークンは、Gatewayを再起動しなくても受け入れられなくなります。
    - Mattermost APIがコマンドが引き続き最新であることを確認できない場合、コールバック検証はフェイルクローズします。失敗した検証は短時間キャッシュされ、同時ルックアップはまとめられ、新しいルックアップの開始はリプレイ負荷を制限するためコマンドごとにレート制限されます。
    - 登録に失敗した場合、起動が部分的だった場合、またはコールバックトークンが解決されたコマンドの登録済みトークンと一致しない場合、スラッシュコールバックはフェイルクローズします（あるコマンドで有効なトークンを使用して、別のコマンドの上流検証に到達することはできません）。
    - 受け入れられたコールバックには、一時的な「処理中...」という応答で受信確認が行われます。実際の回答は通常のメッセージとして届きます。

  </Accordion>
  <Accordion title="到達可能性の要件">
    コールバックエンドポイントはMattermostサーバーから到達可能でなければなりません。

    - MattermostがOpenClawと同じホスト/ネットワーク名前空間で実行されている場合を除き、`callbackUrl`を`localhost`に設定しないでください。
    - そのURLが`/api/channels/mattermost/command`をOpenClawにリバースプロキシしている場合を除き、`callbackUrl`をMattermostのベースURLに設定しないでください。
    - 簡単な確認方法は`curl https://<gateway-host>/api/channels/mattermost/command`です。GETは`404`ではなく、OpenClawから`405 Method Not Allowed`を返す必要があります。

  </Accordion>
  <Accordion title="Mattermostの送信許可リスト">
    コールバック先がプライベート/tailnet/内部アドレスの場合は、コールバックのホスト/ドメインを含めるようにMattermostの`ServiceSettings.AllowedUntrustedInternalConnections`を設定します。

    完全なURLではなく、ホスト/ドメインのエントリを使用してください。

    - 良い例: `gateway.tailnet-name.ts.net`
    - 悪い例: `https://gateway.tailnet-name.ts.net`

  </Accordion>
</AccordionGroup>

## 環境変数（デフォルトアカウント）

環境変数を使用する場合は、Gatewayホストに以下を設定します。

- `MATTERMOST_BOT_TOKEN=...`
- `MATTERMOST_URL=https://chat.example.com`

<Note>
環境変数は**デフォルト**アカウント（`default`）にのみ適用されます。他のアカウントでは設定値を使用する必要があります。

`MATTERMOST_URL`はワークスペースの`.env`から設定できません。[ワークスペースの.envファイル](/ja-JP/gateway/security)を参照してください。
</Note>

## チャットモード

Mattermost は DM に自動的に応答します。チャンネルでの動作は `chatmode` で制御します。

<Tabs>
  <Tab title="oncall (default)">
    チャンネルでは @メンションされた場合にのみ応答します。
  </Tab>
  <Tab title="onmessage">
    チャンネルのすべてのメッセージに応答します。
  </Tab>
  <Tab title="onchar">
    メッセージがトリガープレフィックスで始まる場合に応答します。
  </Tab>
</Tabs>

設定例：

```json5
{
  channels: {
    mattermost: {
      chatmode: "onchar",
      oncharPrefixes: [">", "!"], // デフォルト
    },
  },
}
```

注：

- `onchar` でも明示的な @メンションには応答します。
- `channels.mattermost.requireMention` も引き続き適用されますが、`chatmode` が推奨されます。チャンネルごとの `groups.<channelId>.requireMention` 設定は、どちらよりも優先されます。
- ボットがチャンネルのスレッドに表示される返信を送信した後は、同じスレッド内の後続メッセージに新たな @メンションや `onchar` プレフィックスがなくても応答するため、複数ターンのスレッド会話が継続します。参加状態は、そのスレッドでボットが最後に返信してから 7 日間記憶され、Gateway の再起動後も保持されます。ボットが閲覧しただけのスレッドには影響しません。明示的なメンションを再び必須にするには、新しいトップレベルメッセージを開始してください。
- 参加済みスレッドのフォローアップがメンションゲートを回避しないようにするには、`channels.mattermost.implicitMentions.threadParticipation: false` を設定します。アカウント単位のオーバーライドには `channels.mattermost.accounts.<id>.implicitMentions` を使用します。現在 Mattermost は `replyToBot` または `quotedBot` の情報を生成しないため、これらのフラグはここでは効果がありません。

## スレッドとセッション

チャンネルおよびグループへの返信をメインチャンネルに残すか、トリガーとなった投稿の下にスレッドを開始するかを制御するには、`channels.mattermost.replyToMode` を使用します。

- `off`（デフォルト）：受信した投稿がすでにスレッド内にある場合にのみ、スレッド内で返信します。
- `first`：トップレベルのチャンネル／グループ投稿では、その投稿の下にスレッドを開始し、会話をスレッドスコープのセッションにルーティングします。
- `all` と `batched`：現在の Mattermost では `first` と同じ動作です。Mattermost にスレッドルートが作成されると、後続のチャンクとメディアも同じスレッド内で継続するためです。
- `replyToMode` が設定されている場合でも、ダイレクトメッセージのデフォルトは `off` です。

`direct`、`group`、または `channel` のチャットでモードをオーバーライドするには、`channels.mattermost.replyToModeByChatType` を使用します。ダイレクトメッセージでスレッドを使用するには、`direct` を設定します。

- `off`（デフォルト）：ダイレクトメッセージはスレッド化されず、1 つの継続的なセッションに保持されます。
- `first`、`all`、または `batched`：トップレベルのダイレクトメッセージごとに、新しい独立したセッションを使用する Mattermost スレッドを開始します。

```json5
{
  channels: {
    mattermost: {
      replyToMode: "all",
      replyToModeByChatType: {
        direct: "first",
      },
    },
  },
}
```

注：

- スレッドスコープのセッションでは、トリガーとなった投稿 ID をスレッドルートとして使用します。
- `first` と `all` は現在同等です。Mattermost にスレッドルートが作成されると、後続のチャンクとメディアも同じスレッド内で継続するためです。
- チャット種別ごとのオーバーライドは `replyToMode` より優先されます。`direct` のオーバーライドがなければ、既存のデプロイではスレッド化されていないフラットな DM が維持されます。

## アクセス制御（DM）

- デフォルト：`channels.mattermost.dmPolicy = "pairing"`（不明な送信者にはペアリングコードが発行されます）。その他の値：`allowlist`、`open`、`disabled`。
- 承認方法：
  - `openclaw pairing list mattermost`
  - `openclaw pairing approve mattermost <CODE>`
- 公開 DM：`channels.mattermost.dmPolicy="open"` と `channels.mattermost.allowFrom=["*"]`（設定スキーマによりワイルドカードが必須になります）。
- `channels.mattermost.allowFrom` にはユーザー ID（推奨）と `accessGroup:<name>` エントリを指定できます。[アクセスグループ](/ja-JP/channels/access-groups)を参照してください。

## チャンネル（グループ）

- デフォルト：`channels.mattermost.groupPolicy = "allowlist"`（メンション必須）。
- `channels.mattermost.groupAllowFrom` で送信者を許可リストに登録します（ユーザー ID を推奨）。
- `channels.mattermost.groupAllowFrom` には `accessGroup:<name>` エントリを指定できます。[アクセスグループ](/ja-JP/channels/access-groups)を参照してください。
- チャンネルごとのメンションのオーバーライドは `channels.mattermost.groups.<channelId>.requireMention` に、デフォルト設定は `channels.mattermost.groups["*"].requireMention` に配置します。
- `@username` の照合は変更可能で、`channels.mattermost.dangerouslyAllowNameMatching: true` の場合にのみ有効になります。
- 公開チャンネル：`channels.mattermost.groupPolicy="open"`（メンション必須）。
- 解決順序：`channels.mattermost.groupPolicy`、次に `channels.defaults.groupPolicy`、最後に `"allowlist"`。
- ランタイムに関する注記：`channels.mattermost` セクションが完全に欠落している場合、グループチェックではランタイムがフェイルクローズして `groupPolicy="allowlist"` になります（`channels.defaults.groupPolicy` が設定されている場合も同様）。また、警告が一度だけログに記録されます。

例：

```json5
{
  channels: {
    mattermost: {
      groupPolicy: "open",
      groups: {
        "*": { requireMention: true },
        "team-channel-id": { requireMention: false },
      },
    },
  },
}
```

## 外部配信のターゲット

`openclaw message send` または cron/webhooks では、次のターゲット形式を使用します。

| ターゲット                              | 配信先                                                   |
| ----------------------------------- | ------------------------------------------------------------- |
| `channel:<id>`                      | ID で指定したチャンネル                                                 |
| `channel:<name>` または `#channel-name` | ボットが所属するチーム全体から名前で検索したチャンネル |
| `user:<id>` または `mattermost:<id>`    | そのユーザーとの DM                                             |
| `@username`                         | DM（Mattermost API でユーザー名を解決）                 |

外部への送信では、メッセージごとに添付ファイルを最大 1 個までサポートします。複数のファイルは別々に分けて送信してください。

<Warning>
修飾のない不透明な ID（`64ifufp...` など）は、Mattermost では **曖昧** です（ユーザー ID かチャンネル ID かを判別できません）。

OpenClaw は **ユーザーを優先** して解決します。

- ID がユーザーとして存在する場合（`GET /api/v4/users/<id>` が成功する場合）、OpenClaw は `/api/v4/channels/direct` を介してダイレクトチャンネルを解決し、**DM** を送信します。
- それ以外の場合、ID は **チャンネル ID** として扱われます。

動作を確定的にする必要がある場合は、常に明示的なプレフィックス（`user:<id>` / `channel:<id>`）を使用してください。
</Warning>

## DM チャンネルの再試行

OpenClaw が Mattermost の DM ターゲットに送信する際、最初にダイレクトチャンネルを解決する必要がある場合、デフォルトでは一時的なダイレクトチャンネル作成エラーを再試行します。

Mattermost plugin 全体でこの動作を調整するには `channels.mattermost.dmChannelRetry` を、1 つのアカウントで調整するには `channels.mattermost.accounts.<id>.dmChannelRetry` を使用します。デフォルト値：

```json5
{
  channels: {
    mattermost: {
      dmChannelRetry: {
        maxRetries: 3,
        initialDelayMs: 1000,
        maxDelayMs: 10000,
        timeoutMs: 30000,
      },
    },
  },
}
```

注：

- これは DM チャンネルの作成（`/api/v4/channels/direct`）のみに適用され、すべての Mattermost API 呼び出しには適用されません。
- 再試行ではジッター付き指数バックオフを使用し、レート制限、5xx レスポンス、ネットワークエラー、タイムアウトエラーなどの一時的なエラーに適用されます。
- `429` 以外の 4xx クライアントエラーは永続的なエラーとして扱われ、再試行されません。

## プレビューストリーミング

Mattermost は、思考、ツールのアクティビティ、返信の部分的なテキストを **下書きプレビュー投稿** にストリーミングし、最終回答を安全に送信できるようになると、その場で確定します。`partial` モードでは、チャンクごとのメッセージでチャンネルを埋め尽くす代わりに、同じ投稿 ID のプレビューを更新します。`block` モードでは、完了したテキストとツールアクティビティのブロックの間でプレビューが切り替わるため、前のブロックは次のブロックで上書きされず、それぞれ独立した投稿として表示されたままになります。メディアまたはエラーを含む最終回答では、保留中のプレビュー編集をキャンセルし、使い捨てのプレビュー投稿を確定する代わりに通常の配信を使用します。

プレビューストリーミングは `partial` モードで **デフォルトで有効** です。`channels.mattermost.streaming.mode` で設定します（従来のスカラー値またはブール値の `streaming` は `openclaw doctor --fix` によって移行されます）：

```json5
{
  channels: {
    mattermost: {
      streaming: { mode: "partial" }, // off | partial | block | progress
    },
  },
}
```

<AccordionGroup>
  <Accordion title="ストリーミングモード">
    - `partial`（デフォルト）：返信が増えるにつれて編集され、最後に完全な回答で確定される 1 つのプレビュー投稿です。
    - `block` は、完了したテキストとツールアクティビティのブロックの間でプレビューを切り替えるため、各ブロックはその場で上書きされず、それぞれ独立した投稿として表示されたままになります。並列および連続するツール更新は、現在のツールアクティビティ投稿を共有します。
    - `progress` は生成中にステータスプレビューを表示し、完了時にのみ最終回答を投稿します。
    - `off` はプレビューストリーミングを無効にします。`streaming.block.enabled: true` を指定すると、完了したアシスタントブロックは、1 つにまとめられた最終投稿ではなく、通常のブロック返信（個別の投稿）として引き続き配信されます。

  </Accordion>
  <Accordion title="ストリーミング動作に関する注記">
    - ストリームをその場で確定できない場合（たとえば、ストリーミング中に投稿が削除された場合）、OpenClaw は新しい最終投稿の送信にフォールバックするため、返信が失われることはありません。
    - 思考のみのペイロードは、`> Thinking` ブロック引用として届くテキストも含め、チャンネル投稿では抑制されます。他の画面で思考を表示するには `/reasoning on` を設定します。Mattermost の最終投稿には回答のみが保持されます。
    - チャンネルマッピングの対応表については、[ストリーミング](/ja-JP/concepts/streaming#preview-streaming-modes)を参照してください。

  </Accordion>
</AccordionGroup>

## リアクション（メッセージツール）

- `message action=react` を `channel=mattermost` とともに使用します。
- `messageId` は Mattermost の投稿 ID です。
- `emoji` には、`thumbsup` や `:+1:` のような名前を指定できます（コロンは省略可能です）。
- リアクションを削除するには `remove=true`（ブール値）を設定します。
- リアクションの追加および削除イベントは、メッセージと同じ DM／グループポリシーチェックに従い、ルーティングされたエージェントセッションへシステムイベントとして転送されます。

例：

```text
message action=react channel=mattermost target=channel:<channelId> messageId=<postId> emoji=thumbsup
message action=react channel=mattermost target=channel:<channelId> messageId=<postId> emoji=thumbsup remove=true
```

設定：

- `channels.mattermost.actions.reactions`：リアクション操作を有効または無効にします（デフォルトは true）。
- アカウントごとの上書き：`channels.mattermost.accounts.<id>.actions.reactions`。

## インタラクティブボタン（メッセージツール）

クリック可能なボタン付きのメッセージを送信します。ユーザーがボタンをクリックすると、エージェントは選択内容を受信して応答できます。

ボタンは、セマンティックな `presentation` ペイロード（通常のエージェント返信および `message action=send` 内）から生成されます。OpenClaw は値ボタンを Mattermost のインタラクティブボタンとしてレンダリングし、URL ボタンをメッセージテキスト内に表示したままにし、選択メニューを読みやすいテキストにダウングレードします。

```text
message action=send channel=mattermost target=channel:<channelId> presentation={"blocks":[{"type":"buttons","buttons":[{"label":"Yes","value":"yes"},{"label":"No","value":"no"}]}]}
```

プレゼンテーションボタンのフィールド：

<ParamField path="label" type="string" required>
  表示ラベル（別名：`text`）。
</ParamField>
<ParamField path="value" type="string">
  クリック時に返され、アクション ID として使用される値（別名：`callback_data`、`callbackData`）。`url` が設定されていない限り、クリック可能なボタンには必須です。
</ParamField>
<ParamField path="url" type="string">
  リンクボタン。インタラクティブボタンではなく、メッセージ本文内の `label: url` テキストとしてレンダリングされます。
</ParamField>
<ParamField path="style" type='"primary" | "secondary" | "success" | "danger"'>
  ボタンのスタイル。Mattermost は、サポートしていない値にデフォルトのスタイルを適用します。
</ParamField>

エージェントのシステムプロンプトでボタンのサポートを通知するには、チャンネル機能に `inlineButtons` を追加します：

```json5
{
  channels: {
    mattermost: {
      capabilities: ["inlineButtons"],
    },
  },
}
```

ユーザーがボタンをクリックした場合：

<Steps>
  <Step title="アクセスチェック">
    クリックしたユーザーは、メッセージ送信者と同じ DM／グループポリシーチェックに合格する必要があります。権限のないクリックには一時的な通知が表示され、無視されます。
  </Step>
  <Step title="ボタンを確認表示に置換">
    すべてのボタンが確認行（例：「✓ **Yes** selected by @user」）に置き換えられます。
  </Step>
  <Step title="エージェントが選択内容を受信">
    エージェントは選択内容を受信メッセージ（およびシステムイベント）として受け取り、応答します。
  </Step>
</Steps>

<AccordionGroup>
  <Accordion title="実装に関する注記">
    - ボタンのコールバックでは HMAC-SHA256 検証が使用されます（自動で行われ、設定は不要です）。
    - クリック時に添付ブロック全体が置き換えられるため、すべてのボタンがまとめて削除されます。一部のみを削除することはできません。
    - ハイフンまたはアンダースコアを含むアクション ID は自動的にサニタイズされます（Mattermost のルーティング制限）。
    - `action_id` が元の投稿上のアクションと一致しないクリックは、`403`（「不明なアクション」）として拒否されます。

  </Accordion>
  <Accordion title="設定と到達可能性">
    - `channels.mattermost.capabilities`：機能文字列の配列。エージェントのシステムプロンプトでボタンツールの説明を有効にするには、`"inlineButtons"` を追加します。
    - `channels.mattermost.interactions.callbackBaseUrl`：ボタンコールバック用の外部ベース URL（任意。例：`https://gateway.example.com`）。Mattermost から Gateway のバインドホストへ直接到達できない場合に使用します。
    - 複数アカウントの構成では、`channels.mattermost.accounts.<id>.interactions.callbackBaseUrl` 配下に同じフィールドを設定することもできます。
    - `interactions.callbackBaseUrl` を省略すると、OpenClaw は `gateway.customBindHost` + `gateway.port`（デフォルトは 18789）からコールバック URL を導出し、その後 `http://localhost:<port>` にフォールバックします。コールバックパスは `/mattermost/interactions/<accountId>` です。
    - 到達可能性のルール：ボタンのコールバック URL は Mattermost サーバーから到達可能である必要があります。`localhost` が機能するのは、Mattermost と OpenClaw が同じホスト／ネットワーク名前空間で実行されている場合のみです。
    - `channels.mattermost.interactions.allowedSourceIps`：ボタンコールバックの送信元 IP 許可リスト。指定しない場合、ループバックの送信元（`127.0.0.1`、`::1`）のみが許可されるため、リモートの Mattermost サーバーをここで許可リストに追加しないと、そのクリックは `403` で拒否されます。リバースプロキシの背後では、転送ヘッダーから実際のクライアント IP を導出できるように `gateway.trustedProxies` も設定します。
    - コールバック先がプライベート、tailnet、または内部にある場合は、そのホスト／ドメインを Mattermost の `ServiceSettings.AllowedUntrustedInternalConnections` に追加します。

  </Accordion>
</AccordionGroup>

### API の直接統合（外部スクリプト）

外部スクリプトと Webhook は、エージェントの `message` ツールを経由せず、Mattermost REST API を介してボタンを直接投稿できます。OpenClaw の `message` ツールを推奨します。直接統合する場合は、`@openclaw/mattermost/api.js` から `buildButtonAttachments` をインポートしてください。生の JSON を投稿する場合は、次のルールに従ってください：

**ペイロード構造：**

```json5
{
  channel_id: "<channelId>",
  message: "Choose an option:",
  props: {
    attachments: [
      {
        actions: [
          {
            id: "mybutton01", // alphanumeric only - see below
            type: "button", // required, or clicks are silently ignored
            name: "Approve", // display label
            style: "primary", // optional: "default", "primary", "danger"
            integration: {
              url: "https://gateway.example.com/mattermost/interactions/default",
              context: {
                action_id: "mybutton01", // must match button id
                action: "approve",
                // ... any custom fields ...
                _token: "<hmac>", // see HMAC section below
              },
            },
          },
        ],
      },
    ],
  },
}
```

<Warning>
**重要なルール**

1. 添付はトップレベルの `attachments` ではなく、`props.attachments` 内に配置します（そうしないと通知なしに無視されます）。
2. 各アクションには `type: "button"` が必要です。これがない場合、クリックは通知なしに破棄されます。
3. 各アクションには `id` フィールドが必要です。Mattermost は ID のないアクションを無視します。
4. アクションの `id` は **英数字のみ**（`[a-zA-Z0-9]`）である必要があります。ハイフンとアンダースコアは Mattermost のサーバー側アクションルーティングを破損させます（404 が返されます）。使用前に削除してください。
5. `context.action_id` はボタンの `id` と一致する必要があります。Gateway は、投稿に存在しない `action_id` を含むクリックを拒否します。
6. `context.action_id` は必須です。これがない場合、インタラクションハンドラーは 400 を返します。
7. コールバックの送信元 IP は許可されている必要があります（上記の `interactions.allowedSourceIps` を参照）。

</Warning>

**HMAC トークンの生成**

Gateway は HMAC-SHA256 を使用してボタンのクリックを検証します。外部スクリプトでは、Gateway の検証ロジックと一致するトークンを生成する必要があります：

<Steps>
  <Step title="ボットトークンからシークレットを導出">
    `HMAC-SHA256(key="openclaw-mattermost-interactions", data=botToken)`。16 進数でエンコードします。
  </Step>
  <Step title="コンテキストオブジェクトを構築">
    `_token` **以外**のすべてのフィールドを含むコンテキストオブジェクトを構築します。
  </Step>
  <Step title="ソート済みキーでシリアライズ">
    **再帰的にソートしたキー**を使用し、**空白なし**でシリアライズします（Gateway はネストされたオブジェクトも正規化し、コンパクトな JSON を生成します）。
  </Step>
  <Step title="ペイロードに署名">
    `HMAC-SHA256(key=secret, data=serializedContext)`
  </Step>
  <Step title="トークンを追加">
    生成された 16 進ダイジェストをコンテキスト内の `_token` として追加します。
  </Step>
</Steps>

Python の例：

```python
import hmac, hashlib, json

secret = hmac.new(
    b"openclaw-mattermost-interactions",
    bot_token.encode(), hashlib.sha256
).hexdigest()

ctx = {"action_id": "mybutton01", "action": "approve"}
payload = json.dumps(ctx, sort_keys=True, separators=(",", ":"))
token = hmac.new(secret.encode(), payload.encode(), hashlib.sha256).hexdigest()

context = {**ctx, "_token": token}
```

<AccordionGroup>
  <Accordion title="HMAC でよくある落とし穴">
    - Python の `json.dumps` はデフォルトでスペースを追加します（`{"key": "val"}`）。JavaScript のコンパクトな出力（`{"key":"val"}`）に一致させるには、`separators=(",", ":")` を使用してください。
    - 常にコンテキストの**すべての**フィールド（`_token` を除く）に署名してください。Gateway は `_token` を取り除き、残りすべてに署名します。一部だけに署名すると、検証が通知なく失敗します。
    - `sort_keys=True` を使用してください。Gateway は署名前にキーをソートし、Mattermost はペイロードの保存時にコンテキストフィールドの順序を変更する場合があります。
    - シークレットはランダムバイトではなく、bot トークンから導出してください（決定論的）。ボタンを作成するプロセスと検証を行う Gateway の間で、シークレットが同一である必要があります。

  </Accordion>
</AccordionGroup>

## ディレクトリアダプター

Mattermost Plugin には、Mattermost API を介してチャンネル名とユーザー名を解決するディレクトリアダプターが含まれています。これにより、`openclaw message send` および cron/webhook 配信で `#channel-name` と `@username` のターゲットを使用できます。

設定は不要です。アダプターはアカウント設定の bot トークンを使用します。

## 複数アカウント

Mattermost は `channels.mattermost.accounts` 配下で複数のアカウントをサポートします。

```json5
{
  channels: {
    mattermost: {
      accounts: {
        default: { name: "Primary", botToken: "mm-token", baseUrl: "https://chat.example.com" },
        alerts: { name: "Alerts", botToken: "mm-token-2", baseUrl: "https://alerts.example.com" },
      },
    },
  },
}
```

アカウントの値はトップレベルのフィールドを上書きします。アカウントが指定されていない場合に使用するアカウントは、`channels.mattermost.defaultAccount` で選択します。

## トラブルシューティング

<AccordionGroup>
  <Accordion title="チャンネルで返信がない">
    bot がチャンネルに参加していることを確認してメンションする（oncall）か、トリガープレフィックスを使用する（onchar）か、`chatmode: "onmessage"` を設定してください。
  </Accordion>
  <Accordion title="認証または複数アカウントのエラー">
    - bot トークン、ベース URL、およびアカウントが有効になっているかを確認してください。
    - 複数アカウントの問題：環境変数は `default` アカウントにのみ適用されます。
    - プライベート/LAN の Mattermost ホストには `network.dangerouslyAllowPrivateNetwork: true` が必要です（SSRF ガードはデフォルトでプライベート IP をブロックします）。

  </Accordion>
  <Accordion title="ネイティブスラッシュコマンドが失敗する">
    - `Unauthorized: invalid command token.`：OpenClaw がコールバックトークンを受け入れませんでした。一般的な原因：
      - スラッシュコマンドの登録が失敗したか、起動時に一部しか完了しなかった
      - コールバックが誤った Gateway/アカウントに到達している
      - Mattermost に、以前のコールバックターゲットを指す古いコマンドがまだ残っている
      - スラッシュコマンドを再有効化せずに Gateway が再起動した
    - ネイティブスラッシュコマンドが動作しなくなった場合は、ログで `mattermost: failed to register slash commands` または `mattermost: native slash commands enabled but no commands could be registered` を確認してください。
    - `callbackUrl` が省略され、コールバックが `http://localhost:18789/...` のようなループバック URL に解決されたという警告がログに表示される場合、その URL は Mattermost が OpenClaw と同じホスト/ネットワーク名前空間で実行されている場合にしか到達できない可能性があります。代わりに、外部から到達可能な `commands.callbackUrl` を明示的に設定してください。

  </Accordion>
  <Accordion title="ボタンの問題">
    - ボタンが白いボックスとして表示されるか、まったく表示されない：ボタンデータの形式が不正です。各プレゼンテーションボタンには `label` と `value` が必要です（どちらかが欠けているボタンは破棄されます）。
    - ボタンは表示されるが、クリックしても何も起きない：Mattermost サーバーから Gateway に到達できること、Mattermost サーバーの IP が `channels.mattermost.interactions.allowedSourceIps` に含まれていること（これがない場合はループバックのみ許可されます）、およびプライベートターゲットの場合は `ServiceSettings.AllowedUntrustedInternalConnections` にコールバックホストが含まれていることを確認してください。
    - ボタンをクリックすると 404 が返される：ボタンの `id` にハイフンまたはアンダースコアが含まれている可能性があります。Mattermost のアクションルーターは英数字以外の ID では機能しません。`[a-zA-Z0-9]` のみを使用してください。
    - Gateway のログに `rejected callback source` が記録される：クリック元の IP が `interactions.allowedSourceIps` の範囲外です。Mattermost サーバーまたはイングレスを許可リストに追加し、リバースプロキシの背後では `gateway.trustedProxies` を設定してください。
    - Gateway のログに `invalid _token` が記録される：HMAC が一致しません。コンテキストフィールドの一部ではなくすべてに署名していること、キーをソートしていること、およびコンパクトな JSON（スペースなし）を使用していることを確認してください。上記の HMAC セクションを参照してください。
    - Gateway のログに `missing _token in context` が記録される：`_token` フィールドがボタンのコンテキストにありません。インテグレーションペイロードを構築するときに含まれていることを確認してください。
    - Gateway が `Unknown action` でクリックを拒否する：`context.action_id` が投稿上のどのアクション `id` とも一致しません。両方を同じサニタイズ済みの値に設定してください。
    - エージェントがボタンを提示しない：Mattermost チャンネル設定に `capabilities: ["inlineButtons"]` を追加してください。

  </Accordion>
</AccordionGroup>

## 関連項目

- [チャンネルルーティング](/ja-JP/channels/channel-routing) - メッセージのセッションルーティング
- [チャンネルの概要](/ja-JP/channels) - サポートされているすべてのチャンネル
- [グループ](/ja-JP/channels/groups) - グループチャットの動作とメンションによる制御
- [ペアリング](/ja-JP/channels/pairing) - DM の認証とペアリングの流れ
- [セキュリティ](/ja-JP/gateway/security) - アクセスモデルと堅牢化
