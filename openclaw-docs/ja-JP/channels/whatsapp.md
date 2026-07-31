---
read_when:
    - WhatsApp/web チャネルの動作または受信トレイのルーティングに取り組む
summary: WhatsApp チャネルのサポート、アクセス制御、配信動作、運用
title: WhatsApp
x-i18n:
    generated_at: "2026-07-26T09:13:54Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 7489b37f91775868d0694daea8a0958ee000d1411674d1800bb1e77df5961e68
    source_path: channels/whatsapp.md
    workflow: 16
---

Status: WhatsApp Web（Baileys）経由で本番利用可能。Gateway がリンク済みセッションを管理します。独立した Twilio WhatsApp チャンネルはありません。

## インストール

`openclaw onboard` と `openclaw channels add --channel whatsapp` では、初めて選択したときに Plugin のインストールを求められます。Plugin がない場合、`openclaw channels login --channel whatsapp` でも同じインストールフローが提示されます。開発用チェックアウトではローカル Plugin パスを使用します。stable/beta では、まず ClawHub から `@openclaw/whatsapp` をインストールし、失敗した場合は npm にフォールバックします。WhatsApp ランタイムは OpenClaw のコア npm パッケージとは別に提供されるため、ランタイム依存関係は外部 Plugin 側に保持されます。手動でインストールするには、次を実行します。

```bash
openclaw plugins install clawhub:@openclaw/whatsapp
```

裸の npm パッケージ（`@openclaw/whatsapp`）はレジストリへのフォールバックにのみ使用してください。再現可能なインストールが必要な場合に限り、正確なバージョンを固定してください。

<CardGroup cols={3}>
  <Card title="ペアリング" icon="link" href="/ja-JP/channels/pairing">
    不明な送信者に対するデフォルトの DM ポリシーはペアリングです。
  </Card>
  <Card title="チャンネルのトラブルシューティング" icon="wrench" href="/ja-JP/channels/troubleshooting">
    チャンネル横断の診断と修復手順です。
  </Card>
  <Card title="Gateway の設定" icon="settings" href="/ja-JP/gateway/configuration">
    チャンネル設定の完全なパターンと例です。
  </Card>
</CardGroup>

## クイックセットアップ

<Steps>
  <Step title="アクセスポリシーを設定する">

```json5
{
  channels: {
    whatsapp: {
      dmPolicy: "pairing",
      allowFrom: ["+15551234567"],
      groupPolicy: "allowlist",
      groupAllowFrom: ["+15551234567"],
    },
  },
}
```

  </Step>

  <Step title="WhatsApp をリンクする（QR）">

```bash
openclaw channels login --channel whatsapp
```

    ログインは QR のみに対応しています。リモートまたはヘッドレスホストでは、ログインを開始する前に、表示中の QR を電話へ確実に届ける手段を用意してください。ターミナルに表示した QR、スクリーンショット、チャットの添付ファイルは、転送中に期限切れになる場合があります。

    特定のアカウントを使用する場合:

```bash
openclaw channels login --channel whatsapp --account work
```

    ログイン前に既存またはカスタムの認証ディレクトリを関連付ける場合:

```bash
openclaw channels add --channel whatsapp --account work --auth-dir /path/to/wa-auth
openclaw channels login --channel whatsapp --account work
```

  </Step>

  <Step title="Gateway を起動する">

```bash
openclaw gateway
```

  </Step>

  <Step title="最初の DM アクセスリクエストを承認する（ペアリングモード）">

    **Settings → Channels → DM access requests** を開き、WhatsApp アカウントを見つけて、
    送信者を承認します。CLI を使用する場合:

```bash
openclaw pairing list whatsapp
openclaw pairing approve whatsapp <CODE>
```

    DM アクセスリクエストは 1 hour 後に期限切れになります。保留中のリクエストはアカウントごとに最大 3 件です。
    この承認は、アカウント自体をリンクするために使用する WhatsApp ログイン QR とは別です。

  </Step>
</Steps>

<Note>
別の WhatsApp 番号を使用することを推奨します（セットアップとメタデータはそれに最適化されています）が、個人番号や自分宛てチャットを使用する構成も完全にサポートされています。
</Note>

## デプロイパターン

<AccordionGroup>
  <Accordion title="専用番号（推奨）">
    - OpenClaw 専用の WhatsApp ID
    - より明確な DM 許可リストとルーティング境界
    - 自分宛てチャットで混乱する可能性を低減

    ```json5
    {
      channels: {
        whatsapp: {
          dmPolicy: "allowlist",
          allowFrom: ["+15551234567"],
        },
      },
    }
    ```

  </Accordion>

  <Accordion title="個人番号へのフォールバック">
    オンボーディングは個人番号モードをサポートし、自分宛てチャットに適したベースラインを書き込みます。`dmPolicy: "allowlist"`、自分の番号を含む `allowFrom`、`selfChatMode: true`。ランタイムの自分宛てチャット保護では、リンク済みの自分の番号と `allowFrom` をキーとして使用します。
  </Accordion>
</AccordionGroup>

## ランタイムモデル

- Gateway が WhatsApp ソケットと再接続ループを管理します。
- ウォッチドッグは、WhatsApp Web の生のトランスポートアクティビティと、アプリケーションメッセージのアクティビティという 2 つのシグナルを個別に追跡します。接続中だが通信が少ないセッションは、最近メッセージを受信していないという理由だけでは再起動されません。一定の内部時間枠（ユーザーによる設定不可）にわたってトランスポートフレームが届かなくなった場合、または通常のメッセージタイムアウトの 4x を超えてアプリケーションメッセージが届かない場合にのみ、強制的に再接続します。最近アクティブだったセッションを再接続した直後は、最初の時間枠に 4x の時間枠ではなく、より短い通常のメッセージタイムアウトを使用します。OpenClaw は、その再接続時に Baileys が早期に配信するオフラインメッセージへ自動返信できます。対象範囲は受信メッセージ ID の重複排除期間によって制限されます。初回起動時には、短い古い履歴ガードが維持されます。
- 送信には、対象アカウントのアクティブな WhatsApp リスナーが必要です。存在しない場合は即座に失敗します。
- グループ送信では、トークンが現在の参加者メタデータと一致する場合、`@+<digits>` および `@<digits>` トークン（テキストとメディアのキャプション内）にネイティブのメンションメタデータを付加します。LID ベースのグループも対象です。
- ステータスチャットとブロードキャストチャット（`@status`、`@broadcast`）は無視されます。
- ダイレクトチャットには DM セッションルールを使用します（`session.dmScope`。デフォルトの `main` では、DM をエージェントのメインセッションに統合します）。グループセッションは JID ごとに分離されます（`agent:<agentId>:whatsapp:group:<jid>`）。
- WhatsApp Channels/Newsletters は、ネイティブの `@newsletter` JID を使用して、明示的な送信先に指定できます。この場合、DM のセマンティクスではなく、チャンネルセッションのメタデータ（`agent:<agentId>:whatsapp:channel:<jid>`）を使用します。
- WhatsApp Web トランスポートは、Gateway ホストの標準プロキシ環境変数（`HTTPS_PROXY`、`HTTP_PROXY`、`NO_PROXY`、および小文字のバリアント）に従います。チャンネルごとの設定よりも、ホストレベルのプロキシ設定を推奨します。

## MeowCaller で現在のリクエスト元に電話する（実験的）

Plugin は、WhatsApp から開始されたエージェントターンで `whatsapp_call` を公開できます。[MeowCaller](https://github.com/purpshell/meowcaller) を使用して、現在承認されているリクエスト元に WhatsApp 音声通話を発信し、相手が応答した後に OpenClaw の TTS メッセージを再生します。このツールには宛先番号のパラメータがないため、プロンプトによって通話先を変更することはできません。デフォルトでは無効です。

<Warning>
MeowCaller は実験的で、タグ付きリリースがなく、別途ペアリングした whatsmeow のリンク済みデバイスセッションを使用します。Plugin の Baileys 認証情報を再利用することはできません。ペアリングすると、同じ WhatsApp アカウントに別のリンク済みデバイスが追加されます。OpenClaw が使用する ID でスキャンしてください。個人番号または自分宛てチャットのモードでは、自分自身に電話できません。個人番号へ電話するには、OpenClaw 専用番号を使用してください。
</Warning>

<Steps>
  <Step title="実験的な通話を有効にする">

    WhatsApp チャンネル設定に `actions.calls: true` を追加し、Gateway を再起動します。

```json
{
  "channels": {
    "whatsapp": {
      "actions": {
        "calls": true
      }
    }
  }
}
```

    設定がない場合、または `false` の場合、OpenClaw は `whatsapp_call` ツールを公開しません。

  </Step>

  <Step title="レビュー済みの MeowCaller CLI をインストールする">

    アダプターは、Gateway ホストの `PATH` にある `meowcaller` 実行ファイルを必要とします。[MeowCaller PR #7](https://github.com/purpshell/meowcaller/pull/7) がマージされるまでは、レビュー済みのブランチをビルドしてください。

```bash
git clone --branch feat/send-only-notify https://github.com/steipete/meowcaller.git
cd meowcaller
git checkout 752050471fc2bf7a8cdfbf7dbd3cd4e865d85d3f
mkdir -p "$HOME/.local/bin"
go build -o "$HOME/.local/bin/meowcaller" ./cmd/meowcaller
```

    `$HOME/.local/bin` が Gateway サービスの `PATH` に含まれていることを確認してください。このリビジョンには、明示的な `pair` コマンドと送信専用の `notify` コマンドがあります。`notify` は、マイク、スピーカー、ビデオデバイス、診断キャプチャのいずれも開きません。アップストリームのサンプル CLI にある `play` コマンドで代用しないでください。

  </Step>

  <Step title="MeowCaller のリンク済みデバイスをペアリングする">

    WhatsApp エージェントに通話設定の確認を依頼します（`whatsapp_call` のステータスアクションは、アカウント固有の状態ディレクトリとペアリングコマンドを報告します）。デフォルトアカウントの場合:

```bash
state_dir="$HOME/.openclaw/credentials/whatsapp-calls/default"
mkdir -p "$state_dir"
chmod 700 "$state_dir"
meowcaller pair --store "$state_dir/wa-voip.db"
```

    これを対話形式で実行し、**WhatsApp > Linked devices** から QR をスキャンして、`MeowCaller linked device ready` を待ちます。`wa-voip.db` は非公開にしてください。これは MeowCaller のセッションです。デフォルト以外のアカウントには、ステータスアクションで示される個別のストアパスがあります。Windows では、その PowerShell コマンドを実行してください。

  </Step>

  <Step title="TTS を設定し、WhatsApp から電話する">

    通話対応の [TTS プロバイダー](/ja-JP/tools/tts)を設定し、Gateway を再起動してから、`Call me and say the build finished.` のようなリクエストを送信します。ツールは信頼済みの受信コンテキストから送信者を解決し、一時的な非公開 WAV ファイルを合成して、制限された通話時間内で MeowCaller を実行し、その後に音声ファイルを削除します。OpenClaw はアカウントのストアを明示的に渡し、応答、再生、切断の後に終了ステータスがゼロになるまで待機します。タイムアウトまたはゼロ以外の終了は、ツール呼び出しの失敗として扱われます。

  </Step>
</Steps>

制限: 1 対 1 の音声発信のみ、任意の宛先番号は不可、チャット接続との認証共有は不可、個人番号または自分宛てチャットのモードからの自分自身への通話は不可、合成音声は最大 60 seconds、MeowCaller による応答、再生、切断の完了以外に受話器側で聞こえたことを示す受信確認はありません。また、OpenClaw は 115-175 second の制限時間後にコンパニオンプロセスを停止します。この時間枠には、MeowCaller の接続、応答、再生、シャットダウンの各フェーズが含まれます。

## 承認プロンプト

WhatsApp は、exec および Plugin の承認プロンプトを `👍`/`👎` リアクションとして表示できます。これはトップレベルの承認転送設定で制御されます。

```json5
{
  approvals: {
    exec: {
      enabled: true,
      mode: "session",
    },
    plugin: {
      enabled: true,
      mode: "targets",
      targets: [{ channel: "whatsapp", to: "+15551234567" }],
    },
  },
}
```

`approvals.exec` と `approvals.plugin` は独立しています。WhatsApp をチャンネルとして有効にしても、トランスポートがリンクされるだけです。対応する承認種別が有効化され、WhatsApp にルーティングされていない限り、何も送信されません。セッションモードでは、WhatsApp から発生した承認に限り、ネイティブの絵文字承認を配信します。ターゲットモードでは、明示的なターゲットに対して共有転送パイプラインを使用し、承認者 DM への個別のファンアウトは作成しません。

WhatsApp の承認リアクションには、`allowFrom`（または `"*"`）で承認者を明示的に指定する必要があります。`defaultTo` は通常のデフォルトメッセージ送信先を設定するものであり、承認者リストではありません。手動の `/approve` コマンドも、承認を解決する前に通常の WhatsApp 送信者認可フローを通過します。

## 質問へのリアクション

非機密の単一選択質問が 1 つあり、選択肢が 1〜4 個ある `ask_user` プロンプトでは、WhatsApp は選択肢ラベルの横に `1️⃣` から `4️⃣` を表示します。配信されたプロンプトに対応する番号でリアクションすると回答できます。OpenClaw は Gateway を通じて、その番号を正規の選択肢に対応付けます。期限切れまたは重複したタップは無視されます。複数質問、複数選択、自由入力のプロンプトは、引き続きテキスト返信のみです。通常の WhatsApp DM/グループ参加ルールによって、リアクションした送信者が認可されます。

## Plugin フックとプライバシー

WhatsApp の受信メッセージには、個人的な内容、電話番号、グループ識別子、送信者名、セッション相関フィールドが含まれる場合があります。オプトインしない限り、WhatsApp は受信した `message_received` フックのペイロードを Plugin にブロードキャストしません。

```json5
{
  channels: {
    whatsapp: {
      pluginHooks: {
        messageReceived: true,
      },
    },
  },
}
```

オプトインの範囲を `channels.whatsapp.accounts.<id>.pluginHooks.messageReceived` 配下の 1 つのアカウントに限定してください。WhatsApp の受信内容と識別子を信頼して預けられる Plugin に対してのみ有効にしてください。

## アクセス制御と有効化

<Tabs>
  <Tab title="DM ポリシー">
    `channels.whatsapp.dmPolicy`:

    | 値 | 動作 |
    | --- | --- |
    | `pairing`（デフォルト） | 不明な送信者がペアリングをリクエストし、所有者が承認 |
    | `allowlist` | `allowFrom` の送信者のみ許可 |
    | `open` | `allowFrom` に `"*"` を含める必要あり |
    | `disabled` | すべての DM をブロック |

    `allowFrom` は E.164 形式の番号を受け付けます（内部で正規化されます）。これは DM 送信者専用のアクセス制御リストであり、グループ JID または `@newsletter` チャンネル JID への明示的な送信を制限するものではありません。

    複数アカウントのオーバーライド: `channels.whatsapp.accounts.<id>.dmPolicy`（および `.allowFrom`）は、そのアカウントのチャンネルレベルのデフォルトより優先されます。

    ランタイムに関する注意:

    - ペアリングはチャンネルの許可ストアに保持され、設定済みの `allowFrom` とマージされます
    - スケジュールされた自動化と Heartbeat の受信者フォールバックでは、明示的な配信先または設定済みの `allowFrom` が使用されます。DM ペアリングの承認が暗黙的に Cron/Heartbeat の受信者になることはありません
    - 許可リストが設定されていない場合、リンク済みの自己番号はデフォルトで許可されます
    - OpenClaw は、送信した `fromMe` DM（リンク済みデバイスから自分自身に送信したメッセージ）を自動的にペアリングすることはありません

  </Tab>

  <Tab title="グループポリシーと許可リスト">
    グループアクセスには 2 つの層があります:

    1. **グループメンバーシップ許可リスト**（`channels.whatsapp.groups`）: `groups` が省略されている場合、すべてのグループが対象になります。指定されている場合はグループ許可リストとして機能します（`"*"` はすべてを許可します）。
    2. **グループ送信者ポリシー**（`channels.whatsapp.groupPolicy` + `groupAllowFrom`）: `open` は送信者許可リストをバイパスし、`allowlist` は `groupAllowFrom`（または `*`）との一致を要求し、`disabled` はグループからのすべての受信をブロックします。

    `groupAllowFrom` が未設定の場合、`allowFrom` にエントリがあれば、送信者チェックはそれをフォールバックとして使用します。送信者許可リストは、メンション/返信によるアクティベーションより前に評価されます。

    `channels.whatsapp` ブロックがまったく存在しない場合、`channels.defaults.groupPolicy` が別の値に設定されていても、ランタイムは（警告ログを出力して）`groupPolicy: "allowlist"` にフォールバックします。

    <Note>
    グループメンバーシップの解決には、単一アカウント向けの安全策があります。WhatsApp アカウントが 1 つだけ設定され、その `accounts.<id>.groups` が明示的な空オブジェクト（`{}`）である場合、これは「未設定」として扱われ、すべてのグループを暗黙的にブロックする代わりに、ルートの `channels.whatsapp.groups` マップへフォールバックします。2 つ以上のアカウントが設定されている場合、明示的に空のアカウントマップは空のままで、フォールバックしません。これにより、他のアカウントに影響を与えず、1 つのアカウントですべてのグループを意図的に無効化できます。
    </Note>

  </Tab>

  <Tab title="メンションと /activation">
    グループへの返信には、デフォルトでメンションが必要です。メンション検出には以下が含まれます:

    - ボット ID への明示的な WhatsApp メンション
    - 設定済みのメンション正規表現パターン（`agents.entries.*.groupChat.mentionPatterns`、フォールバックは `messages.groupChat.mentionPatterns`）
    - 承認済みグループメッセージの受信ボイスノート文字起こし
    - 暗黙的なボットへの返信検出（返信送信者がボット ID と一致）

    セキュリティ: 引用/返信はメンションのゲート条件を満たすだけであり、送信者の認可を付与するものでは**ありません**。`groupPolicy: "allowlist"` では、許可リストにない送信者は、許可リストにあるユーザーのメッセージに返信してもブロックされたままです。

    セッションレベルのアクティベーションコマンド: `/activation mention` または `/activation always`。これは（グローバル設定ではなく）セッション状態を更新し、所有者によって制限されます。

  </Tab>
</Tabs>

## 設定済み ACP バインディング

WhatsApp は、トップレベルの `bindings[]` による永続的な ACP バインディングをサポートします:

```json5
{
  bindings: [
    {
      type: "acp",
      agentId: "codex",
      match: {
        channel: "whatsapp",
        accountId: "work",
        peer: { kind: "direct", id: "+15555550123" },
      },
    },
    {
      type: "acp",
      agentId: "codex",
      match: {
        channel: "whatsapp",
        accountId: "work",
        peer: { kind: "group", id: "120363424282127706@g.us" },
      },
    },
  ],
}
```

ダイレクトチャットは E.164 番号と照合され、グループは WhatsApp グループ JID と照合されます。グループ許可リスト、送信者ポリシー、およびメンション/アクティベーションのゲート処理は、OpenClaw がバインドされた ACP セッションの存在を確保する前に実行されます。一致したバインディングがそのルートを所有します。ブロードキャストグループが、そのターンを通常の WhatsApp セッションへファンアウトすることはありません。

## 個人番号とセルフチャットの動作

リンク済みの自己番号が `allowFrom` にも含まれている場合、セルフチャットの安全策が有効になります。セルフチャットのターンでは既読通知をスキップし、自分自身に通知してしまうメンション JID の自動トリガー動作を無視し、チャンネル/アカウントの `responsePrefix` が未設定の場合は返信先をデフォルトで `[{identity.name}]`（または `[openclaw]`）にします。

## メッセージの正規化とコンテキスト

<AccordionGroup>
  <Accordion title="受信エンベロープと返信コンテキスト">
    受信メッセージは共有受信エンベロープでラップされます。引用返信では、次の形式でコンテキストが追加されます:

    ```text
    [Replying to <sender> id:<stanzaId>]
    <quoted body or media placeholder>
    [/Replying]
    ```

    利用可能な場合、返信メタデータ（`ReplyToId`、`ReplyToBody`、`ReplyToSender`、送信者 JID/E.164）が設定されます。引用対象がダウンロード可能なメディアの場合、OpenClaw は通常の受信メディアストアを通じて保存し、`MediaPath`/`MediaType` を公開します。これにより、エージェントは `<media:image>` だけを見るのではなく、メディアを直接検査できます。

  </Accordion>

  <Accordion title="メディアプレースホルダーと位置情報/連絡先の抽出">
    メディアのみのメッセージは、プレースホルダー `<media:image>`、`<media:video>`、`<media:audio>`、`<media:document>`、`<media:sticker>` に正規化されます。

    本文が `<media:audio>` のみの場合、承認済みグループのボイスノートはメンションのゲート処理より前に文字起こしされるため、ボイスノート内でボットへのメンションを発話すると返信をトリガーできます。文字起こしに引き続きボットへのメンションがない場合、生のプレースホルダーではなく、保留中のグループ履歴に残ります。

    位置情報の本文は、簡潔な座標テキストとしてレンダリングされます。位置情報のラベル/コメントと連絡先/vCard の詳細は、インラインのプロンプトテキストではなく、フェンスで囲まれた信頼できないメタデータとしてレンダリングされます。

  </Accordion>

  <Accordion title="保留中グループ履歴の注入">
    未処理のグループメッセージはバッファされ、最終的にボットがトリガーされたときにコンテキストとして注入されます。

    - デフォルト上限: `50`
    - 設定: `channels.whatsapp.historyLimit`、フォールバックは `messages.groupChat.historyLimit`
    - `0` で無効化

    注入マーカー: `[Chat messages since your last reply - for context]` および `[Current message - respond to this]`。

  </Accordion>

  <Accordion title="既読通知">
    受理された受信メッセージではデフォルトで有効です。グローバルに無効化するには:

    ```json5
    { channels: { whatsapp: { sendReadReceipts: false } } }
    ```

    アカウントごとのオーバーライド: `channels.whatsapp.accounts.<id>.sendReadReceipts`。セルフチャットのターンでは、グローバルに有効な場合でも既読通知をスキップします。

  </Accordion>
</AccordionGroup>

## 配信、チャンク分割、メディア

<AccordionGroup>
  <Accordion title="テキストのチャンク分割">
    - デフォルトのチャンク上限: `channels.whatsapp.textChunkLimit = 4000`
    - `channels.whatsapp.streaming.chunkMode = "length" | "newline"`; `newline` は段落境界（空行）を優先し、その後、長さの上限を超えないチャンク分割にフォールバックします

  </Accordion>

  <Accordion title="送信メディアの動作">
    - 画像、動画、音声（PTT ボイスノート）、ドキュメントのペイロードをサポート
    - 音声は、`ptt: true` を指定した Baileys の `audio` ペイロードとして送信され、プッシュトゥトークのボイスノートとしてレンダリングされます。返信ペイロードでは `audioAsVoice` が維持されるため、プロバイダーのソース形式にかかわらず、TTS ボイスノート出力はこの経路を使用し続けます
    - ネイティブの Ogg/Opus 音声は `audio/ogg; codecs=opus` として送信されます。それ以外の音声（Microsoft Edge TTS の MP3/WebM 出力を含む）は、PTT 配信前に `ffmpeg` を使用して 48 kHz モノラルの Ogg/Opus にトランスコードされます
    - `/tts latest` は、最新のアシスタント返信を 1 つのボイスノートとして送信し、同じ返信の重複送信を抑止します。`/tts chat on|off|default` は現在のチャットの自動 TTS を制御します
    - 動画送信時に `gifPlayback: true` を指定すると、アニメーション GIF の再生が有効になります
    - `forceDocument`/`asDocument` は、WhatsApp のメディア圧縮を回避するため、送信する画像、GIF、動画を Baileys のドキュメントペイロード経由でルーティングし、解決済みのファイル名と MIME タイプを維持します
    - 複数メディアを含む返信では、キャプションは最初のメディア項目に適用されます。ただし PTT ボイスノートは例外です。音声がキャプションなしで先に送信され、その後キャプションが個別のテキストメッセージとして送信されます（WhatsApp クライアントではボイスノートのキャプションが一貫して表示されないためです）
    - メディアのソースには HTTP(S)、`file://`、またはローカルパスを使用できます

  </Accordion>

  <Accordion title="メディアサイズの上限とフォールバック動作">
    - 受信時の保存上限と送信時の上限: `channels.whatsapp.mediaMaxMb`（デフォルトは `50`）
    - アカウントごとのオーバーライド: `channels.whatsapp.accounts.<id>.mediaMaxMb`
    - `forceDocument`/`asDocument` でドキュメントとしての配信が要求されていない限り、画像は上限に収まるよう自動最適化（サイズ変更/品質の段階的調整）されます
    - メディア送信に失敗した場合、最初の項目のフォールバックとして、応答を暗黙的に破棄する代わりにテキスト警告を送信します

  </Accordion>
</AccordionGroup>

## 返信の引用

`channels.whatsapp.replyToMode` は、ネイティブの返信引用（送信返信で受信メッセージを視覚的に引用する機能）を制御します:

| 値             | 動作                                                       |
| ----------------- | -------------------------------------------------------------- |
| `"off"`（デフォルト） | 引用せず、プレーンメッセージとして送信                           |
| `"first"`         | 最初の送信返信チャンクのみを引用                      |
| `"all"`           | すべての送信返信チャンクを引用                               |
| `"batched"`       | キューに入ったバッチ返信を引用し、即時返信は引用しない |

アカウントごとのオーバーライド: `channels.whatsapp.accounts.<id>.replyToMode`。

```json5
{ channels: { whatsapp: { replyToMode: "first" } } }
```

## リアクションレベル

`channels.whatsapp.reactionLevel` は、エージェントが絵文字リアクションを使用する範囲を制御します:

| レベル                 | 確認リアクション | エージェント主導のリアクション  |
| --------------------- | ------------- | -------------------------- |
| `"off"`               | いいえ            | いいえ                         |
| `"ack"`               | はい           | いいえ                         |
| `"minimal"`（デフォルト） | はい           | はい、控えめに使用 |
| `"extensive"`         | はい           | はい、積極的な使用を推奨   |

アカウントごとのオーバーライド: `channels.whatsapp.accounts.<id>.reactionLevel`。

```json5
{ channels: { whatsapp: { reactionLevel: "ack" } } }
```

## 確認リアクション

`channels.whatsapp.ackReaction` は、受信時に即座にリアクションを送信します。これは `reactionLevel` によって制御されます（`"off"` の場合は抑止されます）:

```json5
{
  channels: {
    whatsapp: {
      ackReaction: {
        emoji: "👀",
        direct: true,
        group: "mentions", // always | mentions | never
      },
    },
  },
}
```

注意: 受信が受理された直後（返信前）に送信されます。`emoji` なしで `ackReaction` が存在する場合、WhatsApp はルーティングされたエージェントの ID 絵文字を使用し、なければ「👀」にフォールバックします（確認リアクションを使用しない場合は `ackReaction` を省略するか、`emoji: ""` を設定します）。失敗はログに記録されますが、返信の配信はブロックされません。グループモードの `mentions` はメンションによってトリガーされたターンにのみリアクションしますが、グループアクティベーションの `always` はこのチェックをバイパスします。WhatsApp では `channels.whatsapp.ackReaction` のみが使用されます（従来の `messages.ackReaction` はここでは適用されません）。

## ライフサイクル状態リアクション

`messages.statusReactions.enabled: true` を設定すると、WhatsApp はターン中に静的な受信絵文字を残す代わりに確認リアクションを置き換え、キュー待ち、思考中、ツール実行、Compaction、完了、エラーなどの状態を順に表示します:

```json5
{
  messages: {
    statusReactions: {
      enabled: true,
    },
  },
}
```

注意: `channels.whatsapp.ackReaction` は引き続きダイレクトメッセージとグループの適格性を制御します。キュー待ち状態では、通常の確認リアクションと同じ有効な絵文字が使用されます。WhatsApp ではメッセージごとにボット用のリアクションスロットが 1 つだけなので、ライフサイクルの更新は現在のリアクションをその場で置き換え、最終的な完了/エラー状態の後に確認リアクションを復元します。

## 複数アカウントと認証情報

<AccordionGroup>
  <Accordion title="アカウントの選択とデフォルト">
    アカウント ID は `channels.whatsapp.accounts` から取得されます。デフォルトのアカウントには、存在する場合は `default` が選択され、それ以外の場合は設定済みのアカウント ID のうち、アルファベット順で最初のものが選択されます。アカウント ID は、検索用に内部で正規化されます。
  </Accordion>

  <Accordion title="認証情報のパスとレガシー互換性">
    - 現在の認証パス: `~/.openclaw/credentials/whatsapp/<accountId>/creds.json`（バックアップ: `creds.json.bak`）
    - `~/.openclaw/credentials/` 内のレガシーなデフォルト認証は、デフォルトアカウントのフローで引き続き認識され、移行されます

  </Accordion>

  <Accordion title="ログアウトの動作">
    `openclaw channels logout --channel whatsapp [--account <id>]` は、そのアカウントの WhatsApp 認証状態を消去します。Gateway に到達可能な場合、ログアウトでは最初にそのアカウントの稼働中のリスナーが停止されるため、次回の再起動を待たずに、リンク済みセッションによるメッセージの受信が停止します。`openclaw channels remove --channel whatsapp` でも、アカウント設定を無効化または削除する前に稼働中のリスナーが停止されます。

    レガシーな認証ディレクトリでは、Baileys の認証ファイルが削除される一方、`oauth.json` は保持されます。

  </Accordion>
</AccordionGroup>

## ツール、アクション、設定の書き込み

- エージェントツールのサポートには、WhatsApp のリアクションアクション（`react`）が含まれます。
- アクションゲート: `channels.whatsapp.actions.reactions`、`channels.whatsapp.actions.polls`（既存のアクションのデフォルトは `true`）、`channels.whatsapp.actions.calls`（デフォルトは `false`。上記の MeowCaller を参照）。
- チャンネルから開始される設定の書き込みはデフォルトで有効です。無効にするには `channels.whatsapp.configWrites: false` を使用します。

## トラブルシューティング

<AccordionGroup>
  <Accordion title="リンクされていない（QR が必要）">
    症状: チャンネルのステータスで、リンクされていないと報告されます。

```bash
openclaw channels login --channel whatsapp
openclaw channels status
```

  </Accordion>

  <Accordion title="リンク済みだが切断される／再接続ループ">
    症状: リンク済みのアカウントで、切断または再接続の試行が繰り返されます。

    通信量の少ないアカウントは、通常のメッセージタイムアウトを過ぎても接続を維持できます。ウォッチドッグが再起動するのは、WhatsApp Web トランスポートのアクティビティが停止した場合、ソケットが閉じた場合、またはアプリケーションレベルのアクティビティがより長い安全期間を超えて停止した場合のみです（上記のランタイムモデルを参照）。

    修正方法:

    ```bash
    openclaw channels status --probe
    openclaw doctor
    openclaw logs --follow
    openclaw gateway status
    ```

    ホストの接続性とタイミングを修正してもループが続く場合は、アカウントの認証ディレクトリをバックアップしてから再リンクします。

    ```bash
    cp -a ~/.openclaw/credentials/whatsapp/<accountId> \
      ~/.openclaw/credentials/whatsapp/<accountId>.bak
    openclaw channels logout --channel whatsapp --account <accountId>
    openclaw channels login --channel whatsapp --account <accountId>
    ```

    `~/.openclaw/logs/whatsapp-health.log` に `Gateway inactive` と表示される一方で、`openclaw gateway status` と `openclaw channels status --probe` の両方が正常と表示される場合は、`openclaw doctor` を実行します。Linux では、廃止された `~/.openclaw/bin/ensure-whatsapp.sh` スクリプトを呼び出すレガシーな crontab エントリについて doctor が警告します。`crontab -e` を使用してそれらのエントリを削除してください。Cron には systemd のユーザーバス環境がない場合があり、その古いスクリプトが Gateway の稼働状態を誤って報告する原因になります。

  </Accordion>

  <Accordion title="プロキシ経由で QR ログインがタイムアウトする">
    症状: `openclaw channels login --channel whatsapp` が、使用可能な QR を表示する前に `status=408 Request Time-out` または TLS ソケットの切断によって失敗します。

    WhatsApp Web のログインでは、Gateway ホストの標準的なプロキシ環境（`HTTPS_PROXY`、`HTTP_PROXY`、それらの小文字表記、`NO_PROXY`）が使用されます。Gateway プロセスがプロキシ環境を継承していること、および `NO_PROXY` が `mmg.whatsapp.net` に一致していないことを確認してください。

  </Accordion>

  <Accordion title="送信時にアクティブなリスナーがない">
    対象アカウントにアクティブな Gateway リスナーが存在しない場合、送信メッセージは即座に失敗します。Gateway が実行中で、アカウントがリンクされていることを確認してください。
  </Accordion>

  <Accordion title="返信がトランスクリプトには表示されるが WhatsApp には表示されない">
    トランスクリプトの行にはエージェントが生成した内容が記録されます。WhatsApp への配信は別途確認されます。OpenClaw は、表示可能なテキストまたはメディアの送信について、少なくとも 1 件の送信メッセージ ID が Baileys から返された場合にのみ、自動返信を送信済みとして扱います。

    Ack リアクションは、返信前に独立して送信される受領通知です。リアクションが成功しても、後続のテキスト／メディア返信が受け付けられたことの証明にはなりません。Gateway のログで `auto-reply delivery failed` または `auto-reply was not accepted by WhatsApp provider` を確認してください。

  </Accordion>

  <Accordion title="グループメッセージが予期せず無視される">
    次の順序で確認してください: `groupPolicy`、`groupAllowFrom`／`allowFrom`、`groups` の許可リストエントリ、メンションゲート（`requireMention` とメンションパターン）、および `openclaw.json` 内の重複キー（JSON5 では後のエントリが前のエントリを上書きします。スコープごとに `groupPolicy` を 1 つだけ保持してください）。

    `channels.whatsapp.groups` が存在する場合、WhatsApp は他のグループからのメッセージを引き続き認識できますが、OpenClaw はセッションルーティングの前にそれらを破棄します。グループ JID を `channels.whatsapp.groups` に追加するか、`groups["*"]` を追加してすべてのグループを許可しつつ、送信者の認可は `groupPolicy`／`groupAllowFrom` で管理します。

  </Accordion>

  <Accordion title="Bun ランタイムの警告">
    OpenClaw Gateway には Node が必要です。Bun は正規の状態ストアで使用される `node:sqlite` API を提供しておらず、doctor はレガシーな Bun サービスを Node に移行します。
  </Accordion>
</AccordionGroup>

## システムプロンプト

WhatsApp は、`groups` および `direct` のマップを介して、グループとダイレクトチャット向けの Telegram 形式のシステムプロンプトをサポートします。

グループメッセージの解決: まず、有効な `groups` マップが決定されます。アカウントが独自の `groups` キーを定義している場合、その内容にかかわらず、ルートの `groups` マップを完全に置き換えます（ディープマージは行われません）。その後、プロンプトの検索は、結果として得られた単一のマップに対して実行されます。

1. **グループ固有のプロンプト**（`groups["<groupId>"].systemPrompt`）: グループのエントリが存在し、**かつ**その `systemPrompt` キーが定義されている場合に使用されます。空文字列（`""`）を指定するとワイルドカードが抑制され、プロンプトは適用されません。
2. **グループのワイルドカードプロンプト**（`groups["*"].systemPrompt`）: 特定のグループエントリが存在しない場合、または存在していても `systemPrompt` キーがない場合に使用されます。

ダイレクトメッセージの解決では、`direct` マップと `direct["*"]` に対して同一のパターンが適用されます。

<Note>
`dms` は、DM ごとの軽量な履歴上書き用バケット（`dms.<id>.historyLimit`）として引き続き使用されます。プロンプトの上書きは `direct` の下に配置されます。
</Note>

<Note>
プロンプト解決におけるこの「アカウントがルートを置き換える」動作は、単純な浅い上書きです。明示的な空のオブジェクトを含め、アカウントの `groups`／`direct` キーは、ルートマップを置き換えます。これは上記のグループ所属許可リストの確認とは異なります。後者には、誤って空にされた `groups: {}` に対する単一アカウント用の安全策があります。
</Note>

**Telegram との違い:** Telegram は、複数アカウント構成におけるすべてのアカウントで、ルートの `groups` を抑制します（独自の `groups` を持たないアカウントも含みます）。これは、Bot が所属していないグループのグループメッセージを受信することを防ぐためです。WhatsApp はこのガードを適用しません。アカウント数にかかわらず、独自の上書きを持たないすべてのアカウントは、ルートの `groups`／`direct` を継承します。複数アカウントの WhatsApp 構成でアカウントごとのプロンプトを使用する場合は、各アカウントの下に完全なマップを明示的に定義してください。

重要な動作:

- `channels.whatsapp.groups` は、グループごとの設定マップであると同時に、チャットレベルのグループ許可リストでもあります。ルートまたはアカウントのいずれのスコープでも、`groups["*"]` は、そのスコープで「すべてのグループを許可する」ことを意味します。
- そのスコープですべてのグループを許可する必要がある場合にのみ、ワイルドカード `systemPrompt` を追加してください。対象を固定されたグループ ID のセットのみに限定するには、`groups["*"]` を使用せず、明示的に許可リストへ追加した各エントリでプロンプトを繰り返し指定します。
- グループの許可と送信者の認可は別々に確認されます。`groups["*"]` は、グループ処理に到達できるグループの範囲を広げますが、それらのグループ内のすべての送信者を認可するものではありません。送信者の認可は引き続き `groupPolicy`／`groupAllowFrom` で管理されます。
- `channels.whatsapp.direct` には、DM に対する同等の副作用はありません。`direct["*"]` は、DM が `dmPolicy` と `allowFrom`、またはペアリングストアのルールによってすでに許可された後にのみ、デフォルト設定を提供します。

例:

```json5
{
  channels: {
    whatsapp: {
      groups: {
        // ルートスコープですべてのグループを許可する場合にのみ使用します。
        // 独自の groups マップを定義していないすべてのアカウントに適用されます。
        "*": { systemPrompt: "すべてのグループ向けのデフォルトプロンプト。" },
      },
      direct: {
        // 独自の direct マップを定義していないすべてのアカウントに適用されます。
        "*": { systemPrompt: "すべてのダイレクトチャット向けのデフォルトプロンプト。" },
      },
      accounts: {
        work: {
          groups: {
            // このアカウントは独自の groups を定義しているため、ルートの groups は完全に
            // 置き換えられます。ワイルドカードを維持するには、ここでも "*" を明示的に定義します。
            "120363406415684625@g.us": {
              requireMention: false,
              systemPrompt: "プロジェクト管理に集中する。",
            },
            // このアカウントですべてのグループを許可する場合にのみ使用します。
            "*": { systemPrompt: "仕事用グループ向けのデフォルトプロンプト。" },
          },
          direct: {
            // このアカウントは独自の direct マップを定義しているため、ルートの direct エントリは
            // 完全に置き換えられます。ワイルドカードを維持するには、ここでも "*" を明示的に定義します。
            "+15551234567": { systemPrompt: "特定の仕事用ダイレクトチャット向けのプロンプト。" },
            "*": { systemPrompt: "仕事用ダイレクトチャット向けのデフォルトプロンプト。" },
          },
        },
      },
    },
  },
}
```

## 設定リファレンスへのポインター

主要なリファレンス: [設定リファレンス - WhatsApp](/ja-JP/gateway/config-channels#whatsapp)

| 領域             | フィールド                                                                                                         |
| ---------------- | -------------------------------------------------------------------------------------------------------------- |
| アクセス           | `dmPolicy`、`allowFrom`、`groupPolicy`、`groupAllowFrom`、`groups`                                             |
| 配信         | `textChunkLimit`、`streaming.chunkMode`、`mediaMaxMb`、`sendReadReceipts`、`ackReaction`、`reactionLevel`      |
| 複数アカウント    | `accounts.<id>.enabled`、`accounts.<id>.authDir`、およびその他のアカウントごとの上書き                              |
| 運用       | `configWrites`、`enabled`                                                                                      |
| 受信のバッチ処理 | `messages.inbound.debounceMs`、`messages.inbound.byChannel.whatsapp`                                           |
| セッションの動作 | `session.dmScope`、`historyLimit`、`dmHistoryLimit`、`dms.<id>.historyLimit`                                   |
| プロンプト          | `groups.<id>.systemPrompt`、`groups["*"].systemPrompt`、`direct.<id>.systemPrompt`、`direct["*"].systemPrompt` |

## 関連項目

- [ペアリング](/ja-JP/channels/pairing)
- [グループ](/ja-JP/channels/groups)
- [セキュリティ](/ja-JP/gateway/security)
- [チャンネルルーティング](/ja-JP/channels/channel-routing)
- [マルチエージェントルーティング](/ja-JP/concepts/multi-agent)
- [トラブルシューティング](/ja-JP/channels/troubleshooting)
