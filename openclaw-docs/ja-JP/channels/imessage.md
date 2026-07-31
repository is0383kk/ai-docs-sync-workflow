---
read_when:
    - iMessage サポートの設定
    - iMessage の送受信のデバッグ
summary: imsg（stdio 経由の JSON-RPC）によるネイティブ iMessage 対応。返信、Tapback、エフェクト、投票、添付ファイル、グループ管理のためのプライベート API アクションを備えています。ホスト要件を満たす場合、新しい OpenClaw iMessage セットアップに推奨されます。
title: iMessage
x-i18n:
    generated_at: "2026-07-26T09:26:59Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: f3e8b1a65c76b25d03615c06a976f86a8af555cd96d5bfdb10cef9c955893ddc
    source_path: channels/imessage.md
    workflow: 16
---

<Note>
通常の OpenClaw iMessage デプロイでは、Gateway と `imsg` を、同じサインイン済み macOS Messages ホスト上で実行します。Gateway を別の場所で実行する場合は、Mac 上で `imsg` を実行する透過的な SSH ラッパーを `channels.imessage.cliPath` に指定します。

**受信復旧は自動です。** ブリッジまたは Gateway の再起動後、iMessage は停止中に受信できなかったメッセージを再生し、Push 復旧後に Apple が一斉送信する可能性のある古い「バックログ爆弾」を抑制します。また、重複排除により同じメッセージが二重にディスパッチされることはありません。有効化するための設定はありません。詳細は[ブリッジまたは Gateway の再起動後の受信復旧](#inbound-recovery-after-a-bridge-or-gateway-restart)を参照してください。
</Note>

<Warning>
BlueBubbles のサポートは削除されました。`channels.bluebubbles` の設定を `channels.imessage` に移行してください。OpenClaw が iMessage をサポートするのは `imsg` 経由のみです。短い告知については[BlueBubbles の削除と imsg iMessage パス](/ja-JP/announcements/bluebubbles-imessage)を、完全な移行表については[BlueBubbles からの移行](/ja-JP/channels/imessage-from-bluebubbles)を参照してください。
</Warning>

状態: ネイティブ外部 CLI 統合。Gateway は `imsg rpc` を起動し、stdio 経由で JSON-RPC を使用して通信します。個別のデーモンやポートはありません。完全な iMessage チャネルを利用するには、Private API モードを強く推奨します。返信、Tapback、エフェクト、投票、添付ファイルへの返信、グループ操作には、`imsg launch` と Private API プローブの成功が必要です。

一般的なローカルセットアップでは、OpenClaw のセットアップにより、サインイン済みの Messages Mac 上で `imsg` を Homebrew 経由でインストールまたは更新するかどうかをユーザーに確認できます。手動セットアップと SSH ラッパートポロジーは引き続きオペレーターが管理します。Gateway またはラッパーを実行するのと同じユーザーコンテキストで、`imsg` をインストールまたは更新してください。

<CardGroup cols={3}>
  <Card title="Private API の操作" icon="wand-sparkles" href="#private-api-actions">
    返信、Tapback、エフェクト、投票、添付ファイル、グループ管理。
  </Card>
  <Card title="ペアリング" icon="link" href="/ja-JP/channels/pairing">
    iMessage の DM はデフォルトでペアリングモードになります。
  </Card>
  <Card title="リモート Mac" icon="terminal" href="#remote-mac-over-ssh">
    Gateway が Messages Mac 上で実行されていない場合は、SSH ラッパーを使用します。
  </Card>
  <Card title="設定リファレンス" icon="settings" href="/ja-JP/gateway/config-channels#imessage">
    iMessage の全フィールドのリファレンス。
  </Card>
</CardGroup>

## クイックセットアップ

<Tabs>
  <Tab title="ローカル Mac（最短手順）">
    <Steps>
      <Step title="imsg のインストールと確認">

```bash
brew install steipete/tap/imsg
brew update && brew upgrade imsg
imsg rpc --help
imsg launch
openclaw channels status --probe
```

        ローカルセットアップのウィザードが、デフォルトの `imsg` コマンドが存在しないことを検出すると、Homebrew 経由で `steipete/tap/imsg` をインストールするよう促すことができます。Homebrew で管理されている `imsg` を検出した場合は、再インストールまたは更新するよう促すことができます。カスタムの `cliPath` ラッパーは変更されません。

      </Step>

      <Step title="OpenClaw の設定">

```json5
{
  channels: {
    imessage: {
      enabled: true,
      cliPath: "/usr/local/bin/imsg",
      dbPath: "/Users/user/Library/Messages/chat.db",
    },
  },
}
```

      </Step>

      <Step title="Gateway の起動">

```bash
openclaw gateway
```

      </Step>

      <Step title="最初の DM ペアリングを承認（デフォルトの dmPolicy）">

```bash
openclaw pairing list imessage
openclaw pairing approve imessage <CODE>
```

        ペアリングリクエストは 1 時間後に期限切れになります。
      </Step>
    </Steps>

  </Tab>

  <Tab title="SSH 経由のリモート Mac">
    ほとんどのセットアップでは SSH は不要です。このトポロジーは、サインイン済みの Messages Mac 上で Gateway を実行できない場合にのみ使用してください。OpenClaw が必要とするのは stdio 互換の `cliPath` だけなので、リモート Mac に SSH 接続して `imsg` を実行するラッパースクリプトを `cliPath` に指定できます。
    `imsg` は Gateway ホストではなく、そのリモート Mac にインストールし、更新してください。

```bash
ssh messages-mac 'brew install steipete/tap/imsg && brew update && brew upgrade imsg'
```

```bash
#!/usr/bin/env bash
exec ssh -T messages-mac imsg "$@"
```

    添付ファイルを有効にする場合の推奨設定:

```json5
{
  channels: {
    imessage: {
      enabled: true,
      cliPath: "~/.openclaw/scripts/imsg-ssh",
      remoteHost: "user@gateway-host", // SCP による添付ファイル取得に使用
      includeAttachments: true,
      // オプション: 追加で許可する添付ファイルのルート（デフォルトの
      // /Users/*/Library/Messages/Attachments とマージされます）。
      attachmentRoots: ["/Users/*/Library/Messages/Attachments"],
      remoteAttachmentRoots: ["/Users/*/Library/Messages/Attachments"],
    },
  },
}
```

    `remoteHost` が設定されていない場合、OpenClaw は SSH ラッパースクリプトを解析して自動検出を試みます。
    `remoteHost` は `host` または `user@host` でなければなりません（空白や SSH オプションは使用不可）。安全でない値は無視されます。
    OpenClaw は SCP に厳格なホストキー検証を使用するため、リレーホストのキーが `~/.ssh/known_hosts` にすでに存在している必要があります。
    添付ファイルのパスは、許可されたルート（`attachmentRoots` / `remoteAttachmentRoots`）に照らして検証されます。

<Warning>
`imsg` の前段に配置する `cliPath` ラッパーまたは SSH プロキシはすべて、長時間接続の JSON-RPC に対する透過的な stdio パイプとして動作しなければなりません。OpenClaw はチャネルの存続中、ラッパーの stdin/stdout 経由で改行区切りの小さな JSON-RPC メッセージを交換します。

- バイトを受信したら、各 stdin チャンクまたは行を**直ちに**転送してください。EOF を待たないでください。
- 逆方向でも、各 stdout チャンクまたは行を速やかに転送してください。
- 改行を保持してください。
- 小さなフレームを滞留させる可能性がある固定サイズのブロッキング読み取り（`read(4096)`、`cat | buffer`、シェルのデフォルトの `read`）は避けてください。
- stderr を JSON-RPC の stdout ストリームから分離してください。

大きなブロックが埋まるまで stdin をバッファリングするラッパーでは、`imsg rpc` 自体が正常でも、`imsg rpc timeout (chats.list)` やチャネルの反復再起動など、iMessage の停止に見える症状が発生します。上記の `ssh -T host imsg "$@"` は、`rpc` や `--db` などの OpenClaw の `cliPath` 引数を転送するため安全です。`ssh host imsg | grep -v '^DEBUG'` のようなパイプラインは安全ではありません。行バッファリングするツールでもフレームを保持する可能性があります。フィルタリングが必要な場合は、すべてのステージで `stdbuf -oL -eL` を使用してください。
</Warning>

  </Tab>
</Tabs>

## 要件と権限（macOS）

- `imsg` を実行する Mac で Messages にサインインしている必要があります。
- OpenClaw/`imsg` を実行するプロセスコンテキストには、フルディスクアクセスが必要です（Messages データベースへのアクセス）。
- Messages.app 経由でメッセージを送信するには、オートメーション権限が必要です。
- 高度な操作（リアクション / 編集 / 送信取り消し / スレッド返信 / エフェクト / 投票 / グループ操作）には、システム整合性保護を無効にする必要があります。詳細は[imsg Private API の有効化](#enabling-the-imsg-private-api)を参照してください。基本的なテキストおよびメディアの送受信は、無効にしなくても動作します。

<Tip>
権限はプロセスコンテキストごとに付与されます。Gateway をヘッドレス（LaunchAgent/SSH）で実行する場合は、同じコンテキストで一度だけ対話型コマンドを実行し、プロンプトを表示させてください。

```bash
imsg chats --limit 1
# または
imsg send <handle> "test"
```

</Tip>

<Accordion title="SSH ラッパーからの送信が AppleEvents -1743 で失敗する">
  リモート SSH セットアップでは、チャットの読み取り、`channels status --probe` の通過、受信メッセージの処理ができても、送信だけが AppleEvents の認可エラーで失敗することがあります。

```text
Messages への Apple Event の送信が認可されていません。(-1743)
```

サインイン済み Mac ユーザーの TCC データベース、または System Settings > Privacy & Security > Automation を確認してください。Automation のエントリが `imsg` またはローカルシェルプロセスではなく `/usr/libexec/sshd-keygen-wrapper` に記録されている場合、macOS はその SSH サーバー側クライアントに対して使用可能な Messages のトグルを表示しないことがあります。

```text
kTCCServiceAppleEvents | /usr/libexec/sshd-keygen-wrapper | auth_value=0 | com.apple.MobileSMS
```

この状態では、Messages のオートメーションを必要とするプロセスコンテキストは UI から権限を付与できるアプリではなく SSH ラッパーであるため、`tccutil reset AppleEvents` を繰り返したり、同じ SSH ラッパー経由で `imsg send` を再実行したりしても、失敗し続ける可能性があります。

代わりに、サポートされている次のいずれかの `imsg` プロセスコンテキストを使用してください。

- Gateway、または少なくとも `imsg` ブリッジを、ログイン済み Messages ユーザーのローカルセッションで実行します。
- 同じセッションからフルディスクアクセスとオートメーションを付与した後、そのユーザーの LaunchAgent で Gateway を起動します。
- 2 ユーザーの SSH トポロジーを維持する場合は、チャネルを有効にする前に、実際の送信 `imsg send` がそのラッパー経由で成功することを確認します。オートメーションを付与できない場合は、送信を SSH ラッパーに依存させず、単一ユーザーの `imsg` セットアップに再構成してください。

</Accordion>

## imsg Private API の有効化

`imsg` には、2 つの動作モードがあります。OpenClaw では、ユーザーが期待するネイティブな iMessage 操作をチャネルで利用できるため、Private API モードを推奨します。基本モードは、リスクを抑えたいインストール、初期検証、または SIP を無効にできないホストで引き続き有用です。

- **基本モード**（デフォルト、SIP の変更は不要）: `send` 経由のテキストおよびメディアの送信、受信の監視と履歴、チャット一覧。新規の `brew install steipete/tap/imsg` と、上記の標準 macOS 権限だけで利用できます。
- **Private API モード**: `imsg` はヘルパー dylib を `Messages.app` に注入し、内部の `IMCore` 関数を呼び出します。これにより、`react`、`edit`、`unsend`、`reply`（スレッド形式）、`sendWithEffect`、`poll` と `poll-vote`（Messages ネイティブの投票）、`renameGroup`、`setGroupIcon`、`addParticipant`、`removeParticipant`、`leaveGroup` に加え、入力中インジケーターと既読通知が利用可能になります。

このページで推奨する操作機能には、Private API モードが必要です。`imsg` の README には、この要件が明記されています。

> `read`、`typing`、`launch`、ブリッジを使用したリッチ送信、メッセージの変更、チャット管理などの高度な機能はオプトインです。これらを使用するには SIP を無効にし、ヘルパー dylib を `Messages.app` に注入する必要があります。SIP が有効な場合、`imsg launch` は注入を拒否します。

このヘルパー注入手法では、Messages の Private API にアクセスするために `imsg` 自体の dylib を使用します。OpenClaw の iMessage パスには、サードパーティ製サーバーも BlueBubbles ランタイムもありません。

<Warning>
**SIP の無効化には、実際のセキュリティ上のトレードオフがあります。** SIP は、改変されたシステムコードの実行を防ぐ macOS の中核的な保護機能の 1 つです。システム全体で無効にすると、攻撃対象領域が拡大し、副作用も生じます。特に、**Apple Silicon Mac で SIP を無効にすると、Mac に iOS アプリをインストールして実行する機能も無効になります**。

特にメインの個人用 Mac では、意図的な運用上の選択として扱ってください。本番品質の OpenClaw iMessage には、ブリッジを有効にしても問題のない専用 Mac またはボット用 macOS ユーザーを推奨します。脅威モデル上、いかなる環境でも SIP の無効化を許容できない場合、組み込みの iMessage は基本モードに限定されます。テキストおよびメディアの送受信のみ利用でき、リアクション / 編集 / 送信取り消し / エフェクト / グループ操作は利用できません。
</Warning>

### セットアップ

1. Messages.app を実行する Mac に **`imsg` をインストール（またはアップグレード）**します。

   ```bash
   brew install steipete/tap/imsg
   brew update && brew upgrade imsg
   imsg --version
   imsg status --json
   ```

   `imsg status --json` の出力には、`bridge_version`、`rpc_methods`、およびメソッドごとの `selectors` が表示されるため、開始前に現在のビルドでサポートされる機能を確認できます。

2. **System Integrity Protection、および（最新の macOS では）ライブラリ検証を無効にします。** Apple 署名済みの `Messages.app` に Apple 製ではないヘルパー dylib を注入するには、SIP をオフにし、**かつ**ライブラリ検証を緩和する必要があります。リカバリモードでの SIP の手順は、macOS のバージョンによって異なります。
   - **macOS 10.13-10.15（Sierra-Catalina）：** Terminal でライブラリ検証を無効にし、Recovery Mode で再起動して `csrutil disable` を実行し、再起動します。
   - **macOS 11 以降（Big Sur 以降）、Intel：** Recovery Mode（または Internet Recovery）で `csrutil disable` を実行し、再起動します。
   - **macOS 11 以降、Apple Silicon：** 電源ボタンを使用した起動手順で Recovery に入ります。最近の macOS バージョンでは、Continue をクリックするときに **Left Shift** キーを押したままにしてから、`csrutil disable` を実行します。仮想マシン環境では別の手順になるため、最初に VM スナップショットを作成してください。

   **macOS 11 以降では、通常、`csrutil disable` だけでは不十分です。** Apple はプラットフォームバイナリである `Messages.app` に対して引き続きライブラリ検証を適用するため、SIP がオフでもアドホック署名されたヘルパーは拒否されます（`Library Validation failed: ... platform binary, but mapped file is not`）。SIP を無効にした後、ライブラリ検証も無効にして再起動します。

   ```bash
   sudo defaults write /Library/Preferences/com.apple.security.libraryvalidation.plist DisableLibraryValidation -bool true
   ```

   **macOS 26（Tahoe）、26.5.1 で検証済み：** SIP のオフに**加えて**上記の `DisableLibraryValidation` コマンドを実行すれば、26.0 から 26.5.x までヘルパーを注入できます。**boot-args は必要ありません。** plist が決定的な要因であり、Tahoe で注入に失敗する場合に最もよく欠けている手順です。
   - **plist がある場合：** `imsg launch` による注入が成功し、`imsg status` は `advanced_features: true` を報告します。
   - **plist がない場合（SIP がオフでも）：** `imsg launch` は `Failed to launch: Timeout waiting for Messages.app to initialize` で失敗します。AMFI がロード時にアドホックヘルパーを拒否するため、ブリッジは準備完了にならず、起動がタイムアウトします。このタイムアウトは Tahoe で大多数のユーザーが遭遇する症状です。修正方法は上記の plist であり、より抜本的な対処ではありません。

   macOS のアップグレード後に `imsg launch` の注入または特定の `selectors` が false を返し始めた場合、通常はこのゲートが原因です。SIP の手順自体が失敗したと判断する前に、SIP とライブラリ検証の状態を確認してください。これらの設定が正しいにもかかわらずブリッジを注入できない場合は、システム全体の追加のセキュリティ制御を弱めるのではなく、`imsg status --json` と `imsg launch` の出力を収集し、`imsg` プロジェクトに報告してください。

3. **ヘルパーを注入します。** SIP を無効にし、Messages.app にサインインした状態で次を実行します。

   ```bash
   imsg launch
   ```

   SIP がまだ有効な場合、`imsg launch` は注入を拒否するため、これは手順 2 が有効になったことの確認にもなります。

4. **OpenClaw からブリッジを検証します。**

   ```bash
   openclaw channels status --probe
   ```

   iMessage のエントリは `works` を報告し、`imsg status --json | jq '{rpc_methods, selectors}'` には使用中の macOS ビルドで公開されている機能が表示されるはずです。投票の作成には `selectors.pollPayloadMessage` が必要で、投票には `selectors.pollVoteMessage` と `poll.vote` RPC メソッドの両方が必要です。OpenClaw Plugin はキャッシュされたプローブでサポートされているアクションのみを公開しますが、キャッシュが空の場合は楽観的に扱い、最初のディスパッチ時にプローブします。

`openclaw channels status --probe` がチャンネルを `works` と報告しているにもかかわらず、特定のアクションがディスパッチ時に「iMessage `<action>` requires the imsg private API bridge」というエラーをスローする場合は、`imsg launch` を再度実行してください。Messages.app の再起動や OS のアップデートなどによりヘルパーが外れることがあり、キャッシュされた `available: true` ステータスは、次のプローブで更新されるまでアクションを公開し続けます。

### SIP を有効にしたままにする場合

脅威モデル上、SIP を無効にできない場合：

- `imsg` は基本モードにフォールバックします。利用できるのはテキスト、メディア、受信のみです。
- OpenClaw Plugin は引き続きテキスト／メディアの送信と受信監視を公開しますが、アクションサーフェスから `react`、`edit`、`unsend`、`reply`、`sendWithEffect`、およびグループ操作を非表示にします（メソッド単位の機能ゲートに従います）。
- プライマリデバイスでは SIP を有効にしたまま、iMessage のワークロード用に SIP をオフにした別の非 Apple Silicon Mac（または専用のボット Mac）を実行できます。以下の[専用ボット用 macOS ユーザー（個別の iMessage ID）](#deployment-patterns)を参照してください。

## アクセス制御とルーティング

<Tabs>
  <Tab title="DM ポリシー">
    `channels.imessage.dmPolicy` はダイレクトメッセージを制御します。

    - `pairing`（デフォルト）
    - `allowlist`（少なくとも 1 つの `allowFrom` エントリが必要）
    - `open`（`allowFrom` に `"*"` を含める必要があります）
    - `disabled`

    許可リストフィールド：`channels.imessage.allowFrom`。

    許可リストのエントリでは、送信者を識別する必要があります。ハンドルまたは静的な送信者アクセスグループ（`accessGroup:<name>`）を使用します。`chat_id:*`、`chat_guid:*`、`chat_identifier:*` などのチャットターゲットには `channels.imessage.groupAllowFrom` を使用し、数値の `chat_id` レジストリキーには `channels.imessage.groups` を使用します。

  </Tab>

  <Tab title="グループポリシーとメンション">
    `channels.imessage.groupPolicy` はグループの処理を制御します。

    - `allowlist`（デフォルト）
    - `open`
    - `disabled`

    グループ送信者の許可リスト：`channels.imessage.groupAllowFrom`。

    `groupAllowFrom` のエントリでは、静的な送信者アクセスグループ（`accessGroup:<name>`）も参照できます。

    実行時のフォールバック：`groupAllowFrom` が未設定の場合、iMessage のグループ送信者チェックは `allowFrom` を使用します。DM とグループで受け入れ条件を分ける場合は、`groupAllowFrom` を設定してください。明示的に空の `groupAllowFrom: []` はフォールバックせず、`allowlist` ではすべてのグループ送信者をブロックします。
    実行時の注意：`channels.imessage` が完全に存在しない場合、ランタイムは `groupPolicy="allowlist"` にフォールバックし、警告をログに記録します（`channels.defaults.groupPolicy` が設定されている場合も同様です）。

    <Warning>
    `groupPolicy: "allowlist"` でのグループルーティングでは、**2 つ**のゲートが連続して実行されます。

    1. **送信者許可リスト**（`channels.imessage.groupAllowFrom`）— ハンドル、`accessGroup:<name>`、`chat_guid`、`chat_identifier`、または `chat_id`。有効なリストが空の場合（`groupAllowFrom` も `allowFrom` のフォールバックもない場合）、すべてのグループ送信者がブロックされます。
    2. **グループレジストリ**（`channels.imessage.groups`）— マップにエントリがある場合に適用されます。チャットは、明示的な `chat_id` ごとのエントリまたは `groups: { "*": { ... } }` ワイルドカードと一致する必要があります。`groups` が空または存在しない場合、送信者許可リストだけで受け入れが決まります。

    有効なグループ送信者許可リストが構成されていない場合、すべてのグループメッセージはレジストリゲートに到達する前に破棄されます。各ゲートにはデフォルトのログレベルで独自の `warn` レベルのシグナルがあり、それぞれ異なる修正方法を示します。

    - 有効なグループ送信者許可リストが空の場合、起動時にアカウントごとに 1 回：`imessage: groupPolicy="allowlist" for account "<id>" but no group sender allowlist is configured ...` — `channels.imessage.groupAllowFrom`（または `allowFrom`）を設定して修正します。`groups` のエントリを追加するだけでは、ゲート 1 が引き続きすべての送信者をブロックします。
    - 送信者がゲート 1 を通過したものの、設定済みの `groups` レジストリにチャットが存在しない場合、実行時に `chat_id` ごとに 1 回：`imessage: dropping group message from chat_id=<id> ...` — `channels.imessage.groups` の下にその `chat_id`（または `"*"`）を追加して修正します。

    DM は影響を受けません。DM では別のコードパスが使用されます。

    `groupPolicy: "allowlist"` でのグループフローに推奨される構成：

    ```json5
    {
      channels: {
        imessage: {
          groupPolicy: "allowlist",
          groupAllowFrom: ["+15555550123"],
          groups: { "*": { "requireMention": true } },
        },
      },
    }
    ```

    `groupAllowFrom` だけで、それらの送信者はどのグループでも受け入れられます。許可するチャットの範囲を限定する（および `requireMention` などのチャットごとのオプションを設定する）には、`groups` ブロックを追加します。
    </Warning>

    グループのメンションゲート：

    - iMessage にはネイティブのメンションメタデータがありません
    - メンションの検出には正規表現パターンを使用します（`agents.entries.*.groupChat.mentionPatterns`、フォールバックは `messages.groupChat.mentionPatterns`）
    - パターンが構成されていない場合、メンションゲートは適用できません
    - 承認された送信者からの制御コマンドは、メンションゲートをバイパスします

    グループごとの `systemPrompt`：

    `channels.imessage.groups.*` の各エントリでは、オプションの `systemPrompt` 文字列を指定できます。この文字列は、そのグループのメッセージを処理するすべてのターンで、エージェントのシステムプロンプトに注入されます。解決方法は `channels.whatsapp.groups` と同じです。

    1. **グループ固有のシステムプロンプト**（`groups["<chat_id>"].systemPrompt`）：特定のグループエントリがマップに存在し、**かつ**その `systemPrompt` キーが定義されている場合に使用されます。`systemPrompt` が空文字列（`""`）の場合、ワイルドカードは抑制され、そのグループにはシステムプロンプトが適用されません。
    2. **グループワイルドカードのシステムプロンプト**（`groups["*"].systemPrompt`）：特定のグループエントリがマップにまったく存在しない場合、または存在していても `systemPrompt` キーが定義されていない場合に使用されます。

    ```json5
    {
      channels: {
        imessage: {
          groupPolicy: "allowlist",
          groupAllowFrom: ["+15555550123"],
          groups: {
            "*": { systemPrompt: "イギリス式のスペルを使用してください。" },
            "8421": {
              requireMention: true,
              systemPrompt: "これはオンコール当番のチャットです。返信は 3 文未満にしてください。",
            },
            "9907": {
              // 明示的な抑制：ワイルドカード「イギリス式のスペルを使用してください。」はここでは適用されません
              systemPrompt: "",
            },
          },
        },
      },
    }
    ```

    グループごとのプロンプトはグループメッセージにのみ適用されます。ダイレクトメッセージには影響しません。

  </Tab>

  <Tab title="セッションと決定論的な返信">
    - DM では直接ルーティングを使用し、グループではグループルーティングを使用します。
    - デフォルトの `session.dmScope=main` では、iMessage の DM はエージェントのメインセッションに集約されます。
    - グループセッションは分離されます（`agent:<agentId>:imessage:group:<chat_id>`）。
    - 返信は、送信元のチャンネル／ターゲットのメタデータを使用して iMessage にルーティングされます。

    グループに類似したスレッドの動作：

    一部の複数参加者による iMessage スレッドは、`is_group=false` を伴って到着する場合があります。
    その `chat_id` が `channels.imessage.groups` の下に明示的に構成されている場合、OpenClaw はこれをグループトラフィックとして扱います（グループゲートとグループセッションの分離）。

  </Tab>
</Tabs>

## ACP 会話のバインディング

iMessage チャットを ACP セッションにバインドできます。

オペレーター向けの迅速な手順：

- DM または許可されたグループチャット内で `/acp spawn codex --bind here` を実行します。
- 以後、同じ iMessage 会話のメッセージは、生成された ACP セッションにルーティングされます。
- `/new` と `/reset` は、同じバインド済み ACP セッションをその場でリセットします。
- `/acp close` は ACP セッションを閉じ、バインディングを削除します。

構成済みの永続的なバインディングでは、トップレベルの `bindings[]` エントリと `type: "acp"` および `match.channel: "imessage"` を使用します。

`match.peer.id` には次を使用できます。

- `+15555550123` や `user@example.com` などの正規化された DM ハンドル
- `chat_id:<id>`（安定したグループバインディングに推奨）
- `chat_guid:<guid>`
- `chat_identifier:<identifier>`

例：

```json5
{
  agents: {
    list: [
      {
        id: "codex",
        runtime: {
          type: "acp",
          acp: { agent: "codex", backend: "acpx", mode: "persistent" },
        },
      },
    ],
  },
  bindings: [
    {
      type: "acp",
      agentId: "codex",
      match: {
        channel: "imessage",
        accountId: "default",
        peer: { kind: "group", id: "chat_id:123" },
      },
      acp: { label: "codex-group" },
    },
  ],
}
```

共通の ACP バインディングの動作については、[ACP エージェント](/ja-JP/tools/acp-agents)を参照してください。

## デプロイパターン

<AccordionGroup>
  <Accordion title="専用ボット用 macOS ユーザー（個別の iMessage ID）">
    専用の Apple ID と macOS ユーザーを使用して、ボットのトラフィックを個人用の Messages プロファイルから分離します。

    一般的な手順：

    1. 専用の macOS ユーザーを作成し、サインインします。
    2. そのユーザーで、ボット用 Apple ID を使用して Messages にサインインします。
    3. そのユーザーに `imsg` をインストールします。
    4. OpenClaw がそのユーザーコンテキストで `imsg` を実行できるように、SSH ラッパーを作成します。
    5. `channels.imessage.accounts.<id>.cliPath` と `.dbPath` がそのユーザープロファイルを参照するように設定します。

    初回実行時は、そのボットユーザーのセッションで GUI の承認（Automation + Full Disk Access）が必要になる場合があります。

  </Accordion>

  <Accordion title="Tailscale 経由のリモート Mac（例）">
    一般的な構成：

    - Gateway は Linux/VM 上で動作
    - iMessage + `imsg` は tailnet 内の Mac 上で動作
    - `cliPath` ラッパーは SSH を使用して `imsg` を実行
    - `remoteHost` は SCP による添付ファイル取得を有効化

    例：

    ```json5
    {
      channels: {
        imessage: {
          enabled: true,
          cliPath: "~/.openclaw/scripts/imsg-ssh",
          remoteHost: "bot@mac-mini.tailnet-1234.ts.net",
          includeAttachments: true,
          dbPath: "/Users/bot/Library/Messages/chat.db",
        },
      },
    }
    ```

    ```bash
    #!/usr/bin/env bash
    exec ssh -T bot@mac-mini.tailnet-1234.ts.net imsg "$@"
    ```

    SSH と SCP の両方が非対話式になるよう、SSH キーを使用します。
    `known_hosts` に情報が登録されるよう、最初にホストキーを信頼済みにしてください（例：`ssh bot@mac-mini.tailnet-1234.ts.net`）。

  </Accordion>

  <Accordion title="複数アカウントのパターン">
    iMessage は `channels.imessage.accounts` 配下のアカウント別設定をサポートしています。

    各アカウントでは、`cliPath`、`dbPath`、`allowFrom`、`groupPolicy`、`mediaMaxMb`、履歴設定、添付ファイルルートの許可リストなどのフィールドを上書きできます。

  </Accordion>

  <Accordion title="ダイレクトメッセージ履歴">
    `channels.imessage.dmHistoryLimit` を設定すると、新しいダイレクトメッセージセッションに、その会話のデコード済み `imsg` 履歴の直近分を投入できます。送信者ごとの上書きには `channels.imessage.dms["<sender>"].historyLimit` を使用します。これには、送信者の履歴を無効にする `0` も含まれます。

    iMessage の DM 履歴は、必要に応じて `imsg` から取得されます。`dmHistoryLimit` を未設定にすると、グローバルな DM 履歴の投入は無効になりますが、送信者ごとの `channels.imessage.dms["<sender>"].historyLimit` が正の値であれば、その送信者に対する投入は引き続き有効になります。

  </Accordion>
</AccordionGroup>

## メディア、チャンク分割、配信先

<AccordionGroup>
  <Accordion title="添付ファイルとメディア">
    - 受信添付ファイルの取り込みは**デフォルトで無効**です。写真、ボイスメモ、動画、その他の添付ファイルをエージェントに転送するには、`channels.imessage.includeAttachments: true` を設定します。無効の場合、添付ファイルのみの iMessage はエージェントに到達する前に破棄され、`Inbound message` ログ行がまったく生成されないことがあります。
    - `remoteHost` が設定されている場合、リモートの添付ファイルパスを SCP 経由で取得可能
    - 添付ファイルパスは、許可されたルートと一致する必要があります：
      - `channels.imessage.attachmentRoots`（ローカル）
      - `channels.imessage.remoteAttachmentRoots`（リモート SCP モード）
      - 設定したルートは、デフォルトのルートパターン `/Users/*/Library/Messages/Attachments` を拡張します（置換ではなくマージ）
    - SCP は厳格なホストキーチェック（`StrictHostKeyChecking=yes`）を使用
    - 送信メディアのサイズには `channels.imessage.mediaMaxMb` を使用（デフォルト 16 MB）

  </Accordion>

  <Accordion title="送信テキストとチャンク分割">
    - テキストのチャンク上限：`channels.imessage.textChunkLimit`（デフォルト 4000）
    - チャンクモード：`channels.imessage.streaming.chunkMode`
      - `length`（デフォルト）
      - `newline`（段落を優先して分割）
    - 送信時の Markdown の太字／斜体／下線／取り消し線は、ネイティブのスタイル付きテキストに変換されます（macOS 15 以降の受信者ではスタイルが表示され、古い環境の受信者にはマーカーなしのプレーンテキストが表示されます）。Markdown テーブルは、チャンネルの Markdown テーブルモードに従って変換されます
    - `channels.imessage.sendTransport`（デフォルトは `auto`、ほかに `bridge`、`applescript`）で、`imsg` による送信方法を選択

  </Accordion>

  <Accordion title="宛先指定形式">
    推奨される明示的な送信先：

    - `chat_id:123`（安定したルーティングに推奨）
    - `chat_guid:...`
    - `chat_identifier:...`

    ハンドルによる送信先もサポートされています：

    - `imessage:+1555...`
    - `sms:+1555...`
    - `user@example.com`

    ```bash
    imsg chats --limit 20
    ```

  </Accordion>
</AccordionGroup>

## プライベート API アクション

`imsg launch` が実行中で、`openclaw channels status --probe` が `privateApi.available: true` を報告している場合、メッセージツールは通常のテキスト送信に加えて、iMessage ネイティブのアクションを使用できます。

すべてのアクションはデフォルトで有効です。個々のアクションを無効にするには `channels.imessage.actions` を使用します：

```json5
{
  channels: {
    imessage: {
      actions: {
        reactions: true,
        edit: true,
        unsend: true,
        reply: true,
        sendWithEffect: true,
        sendAttachment: true,
        renameGroup: true,
        setGroupIcon: true,
        addParticipant: true,
        removeParticipant: true,
        leaveGroup: true,
        polls: true,
      },
    },
  },
}
```

<AccordionGroup>
  <Accordion title="利用可能なアクション">
    - **react**：iMessage の Tapback を追加／削除します（`messageId`、`emoji`、`remove`）。サポートされる Tapback は、love、like、dislike、laugh、emphasize、question に対応します。絵文字を指定せずに削除すると、設定されている Tapback が消去されます。
    - **reply**：既存のメッセージにスレッド形式で返信します（`messageId`、`text` または `message` に加え、`chatGuid`、`chatId`、`chatIdentifier`、または `to`）。添付ファイル付きの返信には、`--file` をサポートする `send-rich` を備えた `imsg` ビルドも必要です。
    - **sendWithEffect**：iMessage エフェクト付きでテキストを送信します（`text` または `message`、`effect` または `effectId`）。短縮名：slam、loud、gentle、invisibleink、confetti、lasers、fireworks、balloon、heart、echo、happybirthday、shootingstar、sparkles、spotlight。
    - **edit**：対応する macOS／プライベート API バージョンで、送信済みメッセージを編集します（`messageId`、`text` または `newText`）。Gateway 自身が送信したメッセージのみ編集できます。
    - **unsend**：対応する macOS／プライベート API バージョンで、送信済みメッセージを取り消します（`messageId`）。Gateway 自身が送信したメッセージのみ取り消せます。
    - **upload-file**：メディア／ファイルを送信します（base64 の `buffer`、または実体化された `media`／`path`／`filePath`、`filename`、任意の `asVoice`）。従来のエイリアス：`sendAttachment`。
    - **renameGroup**、**setGroupIcon**、**addParticipant**、**removeParticipant**、**leaveGroup**：現在の送信先がグループ会話の場合に、グループチャットを管理します。これらはホストの Messages ID を変更するため、所有者である送信者、または `operator.admin` Gateway クライアントが必要です。
    - **poll**：Apple Messages ネイティブの投票を作成します（`pollQuestion`、2～12 回繰り返す `pollOption` に加え、`chatGuid`、`chatId`、`chatIdentifier`、または `to`）。iOS／iPadOS／macOS 26 以降の受信者はネイティブに表示して投票できます。古い OS バージョンでは「投票を送信しました」という代替テキストが表示されます。`selectors.pollPayloadMessage` が必要です。
    - **poll-vote**：既存の投票に投票します（`pollId` または `messageId` に加え、`pollOptionIndex`、`pollOptionId`、`pollOptionText` のいずれか 1 つのみ）。`selectors.pollVoteMessage` と `poll.vote` RPC メソッドが必要です。

    受け付けられた受信投票は、質問、番号付きの選択肢ラベル、得票数、`poll-vote` に必要な投票メッセージ ID とともにエージェント向けにレンダリングされます。

  </Accordion>

  <Accordion title="メッセージ ID">
    受信した iMessage のコンテキストには、短い `MessageSid` 値と、利用可能な場合は完全なメッセージ GUID（`MessageSidFull`）の両方が含まれます。短い ID は、SQLite ベースの直近の返信キャッシュをスコープとし、使用前に現在のチャットと照合されます。短い ID の有効期限が切れた場合は、その ID を提供した会話を送信先に指定し、対応する `MessageSidFull` で再試行してください。完全な ID でも会話やアカウントへの紐付けは回避できないため、別のチャットの ID は現在の送信先の ID に置き換えてください。リモート委任呼び出しでは、現在の会話を示す証拠が利用できない場合、古い完全 ID が拒否されることがあります。

  </Accordion>

  <Accordion title="機能検出">
    OpenClaw がプライベート API アクションを非表示にするのは、キャッシュされたプローブステータスがブリッジを利用不可と示している場合のみです。ステータスが不明な場合、アクションは表示されたままとなり、ディスパッチ時にプローブが遅延実行されます。そのため、`imsg launch` の後、手動でステータスを別途更新しなくても最初のアクションを成功させられます。

  </Accordion>

  <Accordion title="開封確認と入力中表示">
    プライベート API ブリッジが稼働している場合、受け付けられた受信チャットは既読になり、ダイレクトチャットではターンが受け付けられるとすぐに入力中の吹き出しが表示されます。その間、エージェントはコンテキストを準備して生成を行います。既読化を無効にするには、次のように設定します：

    ```json5
    {
      channels: {
        imessage: {
          sendReadReceipts: false,
        },
      },
    }
    ```

    メソッド別の機能リストより前の古い `imsg` ビルドでは、入力中表示／開封確認が通知なしにゲートで無効化されます。OpenClaw は再起動ごとに一度だけ警告を記録するため、確認通知がない原因を特定できます。

  </Accordion>

  <Accordion title="受信 Tapback">
    OpenClaw は iMessage の Tapback を購読し、受け付けられたリアクションを通常のメッセージテキストではなくシステムイベントとしてルーティングします。そのため、ユーザーの Tapback によって通常の返信ループがトリガーされることはありません。

    通知モードは `channels.imessage.reactionNotifications` で制御します：

    - `"own"`（デフォルト）：ユーザーがボット作成のメッセージにリアクションした場合のみ通知します。
    - `"all"`：許可された送信者からのすべての受信 Tapback を通知します。
    - `"off"`：受信 Tapback を無視します。

    アカウント別の上書きには `channels.imessage.accounts.<id>.reactionNotifications` を使用します。

  </Accordion>

  <Accordion title="承認リアクション（👍 / 👎）">
    `approvals.exec.enabled` または `approvals.plugin.enabled` が true で、リクエストが iMessage にルーティングされる場合、Gateway は承認プロンプトをネイティブに配信し、Tapback による解決を受け付けます：

    - `👍`（Like Tapback）→ `allow-once`
    - `👎`（Dislike Tapback）→ `deny`
    - `allow-always` は手動の代替手段として引き続き利用できます。`/approve <id> allow-always` を通常の返信として送信します。

    リアクション処理では、リアクションしたユーザーのハンドルが明示的な承認者である必要があります。承認者リストは `channels.imessage.allowFrom`（または `channels.imessage.accounts.<id>.allowFrom`）から読み込まれます。ユーザーの電話番号を E.164 形式で追加するか、Apple ID のメールアドレスを追加してください（`chat_id:*` のようなチャット送信先は、有効な承認者エントリではありません）。ワイルドカードエントリ `"*"` は有効ですが、すべての送信者による承認を許可します。承認者リストが空の場合、リアクションによるショートカットは完全に無効になります。承認の解決で重要なのは明示的な承認者の許可リストだけであるため、このリアクションショートカットは意図的に `reactionNotifications`、`dmPolicy`、`groupAllowFrom` を迂回します。

    `/approve` テキストコマンドの認可も同じリストに従います。`channels.imessage.allowFrom` が空でない場合、`/approve <id> <decision>` はより広範な DM 許可リストではなく、その承認者リストに照らして認可されます。DM 許可リストでは許可されていても `allowFrom` に含まれない送信者には、明示的な拒否が返されます。`allowFrom` が空の場合、同一チャットの代替動作が引き続き有効となり、`/approve` は DM 許可リストで許可されたすべてのユーザーを認可します。`/approve` またはリアクションを通じて承認する必要があるすべてのオペレーターを、`allowFrom` に追加してください。

    オペレーター向けメモ:
    - リアクションの関連付けは、メモリと Gateway の永続的なキー付きストア（TTL は承認の有効期限に合わせられます）の両方に保存されます。また、Gateway は保留中のプロンプトに対するタップバックもポーリングするため、Gateway の再起動直後に届いたタップバックでも承認を解決できます。
    - オペレーター自身の `is_from_me=true` タップバック（たとえば、ペアリング済みの Apple デバイスからのもの）は、そのハンドルが明示的な承認者である場合に承認を解決します。
    - 承認プロンプトがグループ会話にルーティングされるのは、明示的な承認者が設定されている場合のみです。そうでなければ、任意のグループメンバーが承認できてしまいます。
    - 従来のテキスト形式のタップバック（非常に古い Apple クライアントからの `Liked "…"` プレーンテキスト）はメッセージ GUID を持たないため、承認を解決できません。リアクションによる解決には、現在の macOS / iOS クライアントが送信する構造化されたタップバックメタデータが必要です。

  </Accordion>

  <Accordion title="質問へのリアクション（1️⃣ / 2️⃣ / 3️⃣ / 4️⃣）">
    機密ではない単一選択の質問が 1 つあり、選択肢が 1～4 個の `ask_user` プロンプトでは、OpenClaw が番号付きの絵文字の選択肢を追加します。配信されたプロンプトに対応する番号でリアクションすると回答できます。リアクションには、ボットが作成したメッセージの安定した GUID が含まれている必要があります。その後、OpenClaw は Gateway を通じて番号を正規の選択肢に対応付けます。古いタップや重複したタップは無視されます。

    複数の質問、複数選択、自由記述のプロンプトは、引き続きテキスト返信でのみ回答できます。質問へのリアクションには、通常の iMessage の DM／グループ受け入れルールが適用されます。一般的な `reactionNotifications` が `"off"` の場合でも認識されますが、無関係なリアクションがエージェントイベントになることはありません。

  </Accordion>
</AccordionGroup>

## 設定の書き込み

iMessage では、デフォルトでチャンネルから開始される設定の書き込みが許可されます（`commands.config: true` の場合の `/config set|unset`）。

無効化するには:

```json5
{
  channels: {
    imessage: {
      configWrites: false,
    },
  },
}
```

<a id="coalescing-split-send-dms-command--url-in-one-composition"></a>

## 分割送信された DM の結合（1 回の作成操作でコマンドと URL を送信）

Apple は、コマンドとその URL プレビューを別々の物理的な `chat.db` 行として保存する場合があります。`imsg` 0.13.1 以降では、監視、履歴、検索がメッセージを返す前にこれらの行を結合するため、チャンネル固有の DM 遅延を追加せずに、OpenClaw は 1 つの論理的な受信メッセージを受け取ります。

iMessage の結合設定は必要ありません。廃止された `channels.imessage.coalesceSameSenderDms` キーは `openclaw doctor --fix` によって削除されます。チャンネル全体で短時間に連続するテキストメッセージを意図的にまとめたい場合は、汎用の `messages.inbound` デバウンスを引き続き使用できます。

コマンドと URL を組み合わせた送信が別々のエージェントターンとして到着する場合は、Messages Mac 上の `imsg` を更新します:

```bash
brew update && brew upgrade imsg
```

## ブリッジまたは Gateway の再起動後の受信復旧

iMessage は Gateway の停止中に取りこぼしたメッセージを復旧すると同時に、Push の復旧後に Apple が一気に送信する可能性がある古い「バックログ爆弾」を抑制します。デフォルトの動作は常に有効で、永続的な受信処理と経過時間フェンスに基づいています。

- **永続的なリプレイ保護。** 復旧カーソルを進める前に、OpenClaw は各未加工行を共有 SQLite 受信キューに記録し、その Apple GUID をイベント ID として使用します。完了した行は約 4 時間、最大 10,000 件のトゥームストーンを残すため、再起動後でも同じ GUID のリプレイは破棄されます。保留中の行は、ディスパッチが引き取るまで復旧可能な状態を維持します。
- **停止中の復旧。** 起動時に、モニターは永続的に受け入れられた最後の `chat.db` rowid（アカウントごとに永続化されたカーソル）を記憶し、それを `since_rowid` として `imsg watch.subscribe` に渡します。これにより、imsg はまだ記録されていない行をリプレイした後、ライブ監視を続けます。クラッシュ前に記録された行は SQLite から再開されます。リプレイは直近の 500 行、および最大約 2 時間前までのメッセージに制限され、GUID トゥームストーンによって処理済みのものはすべて破棄されます。
- **古いバックログの経過時間フェンス。** 起動時の境界より後の行は実際のライブメッセージです。そのうち送信日時が到着日時より約 15 分以上古いものは Push によって一気に送信されたバックログとして抑制されます。リプレイされた行（境界以前）は代わりに、より広い復旧期間を使用するため、最近取りこぼしたメッセージは配信されますが、古すぎる履歴は配信されません。

復旧はローカルとリモートの両方の `cliPath` セットアップで機能します。これは、`since_rowid` のリプレイが同じ `imsg` RPC 接続を介して実行されるためです。違いは期間です。Gateway が `chat.db` を読み取れる場合（ローカル）は、起動時の rowid 境界を基準としてリプレイ範囲を制限し、最大で数時間前までの取りこぼしたメッセージを配信します。リモート SSH の `cliPath` 経由ではデータベースを読み取れないため、リプレイは制限されず、すべての行にライブ経過時間フェンスが適用されます。それでも最近取りこぼしたメッセージは復旧され、古いバックログも抑制されますが、ライブ期間はより狭くなります。より広い復旧期間を使用するには、Messages Mac 上で Gateway を実行してください。

### オペレーターが確認できるシグナル

抑制されたバックログはデフォルトレベルでログに記録され、通知なしに破棄されることはありません（`recovery` フラグは、どの期間が適用されたかを示します）:

```text
imessage: 古い受信バックログを抑制しました account=<id> sent=<iso> recovery=<bool>（起動後に <N> 件を抑制）
```

### 移行

`channels.imessage.catchup.*` は非推奨です。停止中の復旧は自動で行われ、新しいセットアップでは設定は不要です。`catchup.enabled: true` を含む既存の設定は、復旧リプレイ期間の互換性プロファイルとして引き続き尊重されます。無効化されたキャッチアップブロック（`enabled: false` または `enabled: true` なし）は廃止され、`openclaw doctor --fix` によって削除されます。

## トラブルシューティング

<AccordionGroup>
  <Accordion title="imsg が見つからない、または RPC がサポートされていない">
    バイナリと RPC のサポートを検証します:

    ```bash
    imsg rpc --help
    imsg status --json
    openclaw channels status --probe
    ```

    プローブで RPC がサポートされていないと報告された場合は、`imsg` を更新してください。プライベート API のアクションを利用できない場合は、ログイン中の macOS ユーザーセッションで `imsg launch` を実行し、再度プローブしてください。Gateway が macOS 上で実行されていない場合は、デフォルトのローカル `imsg` パスではなく、前述の SSH 経由のリモート Mac セットアップを使用してください。

  </Accordion>

  <Accordion title="メッセージは送信されるが、受信 iMessage が届かない">
    まず、メッセージがローカル Mac に到達したかどうかを確認します。`chat.db` が変化しない場合、`imsg status --json` が正常なブリッジを報告していても、OpenClaw はメッセージを受信できません。

```bash
imsg chats --limit 10 --json
imsg watch --chat-id <chat-id> --json
sqlite3 ~/Library/Messages/chat.db \
  "select datetime(max(date)/1000000000 + 978307200, 'unixepoch', 'localtime'), max(ROWID) from message;"
```

    電話から送信したメッセージによって新しい行が作成されない場合は、OpenClaw の設定を変更する前に、macOS の Messages と Apple Push レイヤーを修復してください。通常は 1 回限りのサービス更新で十分です:

```bash
launchctl kickstart -k system/com.apple.apsd
launchctl kickstart -k gui/$(id -u)/com.apple.CommCenter
launchctl kickstart -k gui/$(id -u)/com.apple.identityservicesd
launchctl kickstart -k gui/$(id -u)/com.apple.imagent
imsg launch
openclaw gateway restart
```

    電話から新しい iMessage を送信し、新しい `chat.db` 行または `imsg watch` イベントを確認してから、OpenClaw セッションをデバッグしてください。これを定期的なブリッジ再起動ループとして実行しないでください。作業中に `imsg launch` と Gateway の再起動を繰り返すと、配信が中断され、処理中のチャンネル実行が取り残される可能性があります。

  </Accordion>

  <Accordion title="Gateway が macOS 上で実行されていない">
    デフォルトの `cliPath: "imsg"` は、Messages にサインインしている Mac 上で実行する必要があります。Linux または Windows では、`channels.imessage.cliPath` に、その Mac へ SSH 接続して `imsg "$@"` を実行するラッパースクリプトを設定します。

```bash
#!/usr/bin/env bash
exec ssh -T messages-mac imsg "$@"
```

    次に実行します:

```bash
openclaw channels status --probe --channel imessage
```

  </Accordion>

  <Accordion title="DM が無視される">
    以下を確認します:

    - `channels.imessage.dmPolicy`
    - `channels.imessage.allowFrom`
    - ペアリングの承認（`openclaw pairing list imessage`）

  </Accordion>

  <Accordion title="グループメッセージが無視される">
    以下を確認します:

    - `channels.imessage.groupPolicy`
    - `channels.imessage.groupAllowFrom`
    - `channels.imessage.groups` 許可リストの動作
    - メンションパターンの設定（`agents.entries.*.groupChat.mentionPatterns`）

  </Accordion>

  <Accordion title="リモート添付ファイルが失敗する">
    以下を確認します:

    - `channels.imessage.remoteHost`
    - `channels.imessage.remoteAttachmentRoots`
    - Gateway ホストからの SSH/SCP キー認証
    - Gateway ホスト上の `~/.ssh/known_hosts` にホストキーが存在すること
    - Messages を実行している Mac 上でリモートパスを読み取れること

  </Accordion>

  <Accordion title="macOS の権限プロンプトを見逃した">
    同じユーザー／セッションコンテキストの対話型 GUI ターミナルで再実行し、プロンプトを承認します:

    ```bash
    imsg chats --limit 1
    imsg send <handle> "test"
    ```

    OpenClaw／`imsg` を実行するプロセスコンテキストに「フルディスクアクセス」と「オートメーション」が許可されていることを確認してください。

  </Accordion>
</AccordionGroup>

## 設定リファレンスへのリンク

- [設定リファレンス - iMessage](/ja-JP/gateway/config-channels#imessage)
- [Gateway の設定](/ja-JP/gateway/configuration)
- [ペアリング](/ja-JP/channels/pairing)

## 関連項目

- [チャンネルの概要](/ja-JP/channels) — サポートされているすべてのチャンネル
- [BlueBubbles の削除と imsg iMessage パス](/ja-JP/announcements/bluebubbles-imessage) — お知らせと移行の概要
- [BlueBubbles からの移行](/ja-JP/channels/imessage-from-bluebubbles) — 設定変換表と段階的な切り替え手順
- [ペアリング](/ja-JP/channels/pairing) — DM 認証とペアリングの流れ
- [グループ](/ja-JP/channels/groups) — グループチャットの動作とメンションによる制御
- [チャンネルルーティング](/ja-JP/channels/channel-routing) — メッセージのセッションルーティング
- [セキュリティ](/ja-JP/gateway/security) — アクセスモデルと強化策
