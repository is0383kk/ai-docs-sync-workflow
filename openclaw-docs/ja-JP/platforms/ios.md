---
read_when:
    - iOS Node のペアリングまたは再接続
    - 直接接続の Apple Watch Node の有効化またはトラブルシューティング
    - ソースから iOS アプリを実行する
    - Gateway 検出または canvas コマンドのデバッグ
summary: iOS Node アプリ：Gateway への接続、ペアリング、キャンバス、トラブルシューティング
title: iOS アプリ
x-i18n:
    generated_at: "2026-07-26T09:07:48Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 2b01a63fa1e2c445f7fb35843536f7f5918e94bfe885dac19c852d7d52d86342
    source_path: platforms/ios.md
    workflow: 16
---

提供状況：iPhone アプリのビルドは、リリースで有効になっている場合、Apple のチャネルを通じて配布されます。ローカル開発ビルドはソースから実行することもできます。

## 機能

- WebSocket 経由で Gateway（LAN または tailnet）に接続します。
- ノード機能を公開します：Canvas、画面スナップショット、カメラ撮影、位置情報、トークモード、音声ウェイク、オプトインのヘルスケア概要。
- `node.invoke` コマンドを受信し、ノードのステータスイベントを報告します。
- 「Agents」画面（「Files」）から、選択したエージェントのワークスペースを読み取り専用で閲覧できます。ディレクトリのドリルダウン、構文強調表示付きのテキストプレビュー、画像プレビュー、共有シートへのエクスポートに対応します。書き込み操作はできません。プレビューのサイズは Gateway によって制限されます。
- ペアリング済み Gateway ごとに、最近のチャットセッションとトランスクリプトの小規模な読み取り専用オフラインキャッシュを保持します。コールド起動時には最後に確認されたトランスクリプトをすぐに表示し、Gateway が応答すると更新します。切断中も最近のチャットを閲覧でき、リセットまたは登録解除を行うと、保護されたローカルキャッシュが消去されます。
- 切断中に送信したテキストメッセージを、Gateway ごとの永続的な送信トレイ（最大 50 件）にキューイングします。キュー内の吹き出しはトランスクリプトに表示され、再接続時に冪等な再試行を使用して順番に送信されます。正規履歴で送信が確認されるまで永続的に保持され、再試行または削除アクションを表示する前にバックオフ付きで再試行されます。オフライン状態が 48 時間続くと送信せずに期限切れになります。リセットまたは登録解除を行うと、キャッシュとともにキューも消去されます。
- チャットは、テキストと音声を扱う単一の画面です。チャットから離れることなく、チャットのアクションで「Sessions」画面全体を開いたり、アシスタントの推論とツールのアクティビティを表示または非表示にしたりできます。下書きを音声入力するにはマイクをタップし、音声メモを録音するにはそのメニューを開き、リアルタイム音声を使用するにはインラインの「Talk」コントロールを使用します。「Talk」コントロールは、聞き取り中または発話中に、マイクのライブ入力レベルまたは再生レベルに応じてアニメーション表示されます。
- **Settings -> OpenClaw** を開くと、オペレーター接続に `operator.admin` があり、Gateway が `openclaw.chat` をサポートしている場合、Gateway 専用の設定アシスタントが表示されます。そのセットアップ会話は通常のチャットとは分離されたままで、機密情報を含む応答はローカルで秘匿化されます。**Open Chat** をタップした後にのみチャットへ移動します。
- 必要に応じてアシスタントのメッセージを読み上げます。チャット内のメッセージを長押しし、**Listen** を選択します。アプリは、設定済みの TTS プロバイダーを使用して、Gateway がサポートする `tts.speak` クリップを再生します。Gateway の音声が利用できないか再生できない場合は、デバイス上の音声合成にフォールバックします。セッションを切り替えるか、アプリがバックグラウンドに移行すると、再生は停止します。

## 要件

- 別のデバイス（macOS、Linux、または WSL2 経由の Windows）で Gateway が実行されていること。
- ネットワーク経路：
  - Bonjour を使用した同一 LAN、**または**
  - ユニキャスト DNS-SD を使用した tailnet（ドメイン例：`openclaw.internal.`）、**または**
  - ホスト／ポートの手動指定（フォールバック）。

## クイックスタート（ペアリングと接続）

初回起動時に、アプリは簡単なペアリングの説明と、
権限ページ（通知、カメラ、マイク、写真、連絡先、
カレンダー、リマインダー、位置情報）を順に表示します。すべての許可は任意であり、
後から **Settings** -> **Permissions** または iOS の設定アプリで
変更できます。

1. スマートフォンから到達可能な経路を持つ、認証済み Gateway を起動します。リモート接続には Tailscale
   Serve を推奨します。

```bash
openclaw gateway --port 18789 --tailscale serve
```

信頼できる同一 LAN のセットアップでは、代わりに認証済みの `gateway.bind: "lan"` を
使用します。デフォルトのループバックバインドにはスマートフォンから到達できません。
Gateway がまだ構成されていない場合は、最初に `openclaw onboard` を実行し、セットアップコードの
作成に使用できるトークンまたはパスワード認証経路を用意します。

2. [Control UI](/ja-JP/web/control-ui) を開いて **Nodes** を選択し、
   **Devices** ページで **Pair mobile device** をクリックします。フルアクセスが推奨され、
   デフォルトで選択されています。管理用 Gateway コントロールを省略する場合にのみ
   制限付きアクセスを選択し、**Create setup code** をクリックします。

3. iOS アプリで **Settings** -> **Gateway** を開き、QR コードをスキャンするか
   セットアップコードを貼り付けて接続します。

   セットアップコードに LAN と Tailscale Serve の両方の経路が含まれている場合、アプリは
   それらを順番に検査し、最初に到達できたエンドポイントを保存します。

   ペアリング済みの Gateway は **Gateways** リストに残ります。チェックマークは
   フォーカス中の Gateway を示します。別の行の稲妻コントロールを使用すると、その
   オペレーターセッションも同時に接続状態に保てます。フォーカスを切り替えても、
   有効になっている他の Gateway は切断されません。iPhone の機能を提供する
   ノードセッションを受信するのはフォーカス中の Gateway のみであるため、カメラ、画面、位置情報、
   その他のデバイスコマンドの所有者は常に 1 つに明確化されます。アプリがバックグラウンドに移行すると、
   iOS によってこれらのフォアグラウンド接続が一時停止される場合があります。

4. 公式アプリは自動的に接続します。**Pending approval** にリクエストが
   表示された場合は、承認する前にそのロールとスコープを確認します。

   **Settings → Gateway** には、保存されたオペレーター接続のアクセス権が
   **Full** か **Limited** かが表示されます。平文 LAN の `ws://` セットアップは、ベアラートークンの安全性を確保するため、
   自動的に制限されます。制限されている場合は、`wss://` または
   Tailscale Serve を構成し、Control UI または `openclaw qr` から新しいフルアクセスコードをスキャンして、
   再接続すると設定とアップグレードが有効になります。

Control UI ボタンを使用するには、`operator.admin` を持つペアリング済みセッションが必要です。
ターミナルをフォールバックとして使用する場合は、iOS アプリで検出された Gateway を選択するか、
Manual Host を有効にしてホスト／ポートを入力し、Gateway ホスト上でリクエストを承認します。

```bash
openclaw devices list
openclaw devices approve <requestId>
```

アプリが認証の詳細（ロール／スコープ／公開鍵）を変更してペアリングを再試行すると、以前の保留中のリクエストは置き換えられ、新しい `requestId` が作成されます。承認前に `openclaw devices list` をもう一度実行します。

任意：iOS ノードが常に厳密に管理されたサブネットから接続する場合、CIDR または正確な IP アドレスを明示して、初回ノードの自動承認をオプトインで有効にできます。

```json5
{
  gateway: {
    nodes: {
      pairing: {
        autoApproveCidrs: ["192.168.1.0/24"],
      },
    },
  },
}
```

これはデフォルトで無効になっています。要求されたスコープがない、新規の `role: node` ペアリングにのみ適用されます。オペレーター／ブラウザーのペアリング、およびロール、スコープ、メタデータ、公開鍵の変更には、引き続き手動承認が必要です。

5. 接続を確認します。

```bash
openclaw nodes status
openclaw gateway call node.list --params "{}"
```

## ヘルスケア概要

iOS ノードは、現在の暦日に対するオプトイン方式の読み取り専用 HealthKit 集計を返すことができます。
iOS デバイスの同意と明示的な Gateway コマンドの認可は、
互いに独立したゲートです。セットアップ、呼び出し、ペイロードフィールド、プライバシー動作、
トラブルシューティングについては、[HealthKit の概要](/ja-JP/platforms/ios-healthkit)を参照してください。

デフォルトでは、Apple Watch コンパニオンは既存の iPhone リレーを引き続き使用するため、
別途 Gateway とペアリングする必要はありません。Apple の Watch アプリで Watch を iPhone と
ペアリングし、**Watch app -> My Watch -> Available
Apps** から OpenClaw をインストールして、両方のデバイスで OpenClaw を一度開きます。

## コマンドの承認を確認する

`operator.admin` を持つオペレーター接続、または Gateway によって明示的に対象として指定された
ペアリング済みの `operator.approvals` 接続は、iPhone 上で保留中の実行リクエストを確認できます。
承認カードには、Gateway によってサニタイズされたコマンドのプレビュー、警告、ホストのコンテキスト、
有効期限、そのリクエストで提示される選択肢のみが表示されます。ペアリング済みの Apple Watch は、
既存の iPhone リレーを通じて同じレビュアー向けの安全なプロンプトを受信し、
1 回のみ許可／拒否の簡略化された選択肢を提示します。Watch から Gateway へ直接接続するモードでは、
承認プロンプトは送信されません。

承認状態は Control UI および対応するチャット画面と共有されます。
最初に確定した回答が採用されます。別の画面でリクエストが解決された後、リモートから
解決済みの通知を受信した後、または解決確認応答が失われた可能性がある場合、
iPhone と Watch は Gateway の正規の最終記録を取得します。その読み戻しによって
リクエストが引き続き保留中かどうか確認されるまで、アクションは利用できません。

承認の所有権は、選択した Gateway に結び付けられます。Gateway を切り替えても、
古いプロンプトを切り替え後の接続に適用することはできません。統合された承認メソッドより前の
Gateway では、リリース済みの実行固有メソッドにフォールバックします。
保持される最終状態と、より充実した画面間の結果を使用するには、更新済みの
Gateway が必要です。

## エージェントの質問に回答する

チャットでは、`operator.questions`（または `operator.admin`）を持つオペレーター接続に対する
保留中の Gateway の質問が、ネイティブカードとして表示されます。カードは単一選択と
複数選択のオプション、オプションの説明、自由記述の **Other** 回答、
有効期限のカウントダウンをサポートします。再接続すると、Gateway から保留中の質問が再読み込みされます。
このデバイスが回答した場合、別の画面が先に回答した場合、または質問が期限切れになるか
キャンセルされた場合、カードはロックされます。

## 任意の Apple Watch 直接ノード

直接モードでは、Watch が独自の署名済みノード ID と Gateway 接続を持ちます。
OpenClaw がアクティブであれば、ペアリング済みの iPhone を利用できない場合でも、
対応するノードコマンドは Watch の Wi-Fi またはモバイル通信経由で引き続き機能します。

要件：

- iPhone が `operator.admin` スコープで Gateway に接続されていること。
- セットアップコードで、watchOS が信頼する証明書を持つ `wss://` Gateway エンドポイントが通知されること。
  Watch は対応する `https://` オリジンをポーリングします。平文 HTTP、および
  自己署名証明書またはフィンガープリントのみの信頼には対応していません。エンドポイントの構成については、
  [Gateway 所有のペアリング](/ja-JP/gateway/pairing)を参照してください。ループバック、iPhone 専用、
  tailnet 専用の経路には、Watch から単独で到達できません。
- モバイル通信を使用するには、モバイル通信対応の Apple Watch と有効な通信サービスが必要です。
- Watch 上で OpenClaw がアクティブであること。Apple は通常の watchOS アプリによる
  汎用 WebSocket／TCP 接続の維持を許可していないため、直接ノードは短時間の HTTPS
  ポーリングを使用し、アプリがフォアグラウンドに戻ると再接続します。Apple の
  [watchOS 低レベルネットワークガイダンス](https://developer.apple.com/documentation/technotes/tn3135-low-level-networking-on-watchOS)を参照してください。

セットアップ：

1. iPhone で **Settings -> Apple Watch** を開きます。
2. __**Enable Direct Gateway Connection** をタップします。
3. 有効期間の短いセットアップコードが期限切れになる前に、Watch で OpenClaw を開きます。
4. `openclaw nodes status` を使用して、個別の Apple Watch 行を確認します。

セットアップコードには、有効期間が短いノード専用のブートストラップ認証情報が含まれます。
期限が切れるまではパスワードと同様に扱ってください。iPhone に保存された Gateway の
パスワードやトークンが含まれることはありません。ペアリング後、Watch は独自のデバイストークンを保存し、
ブートストラップ認証情報を削除します。直接モードで対応するのは、以下のコマンドのみです。
チャット、Talk、承認、および既存の `watch.*` 通知フローは、引き続き
iPhone リレーの機能であり、ペアリング済みの iPhone が必要です。

watchOS 直接ノードのコマンド：

| 画面          | コマンド                       | 備考                                                   |
| ------------- | ------------------------------ | ------------------------------------------------------- |
| デバイス      | `device.info`, `device.status` | Watch の ID、バッテリー、温度、ストレージ、ネットワーク。 |
| 通知          | `system.notify`                | アプリがアクティブな間。Watch の権限が必要です。       |

watchOS は WebKit をサードパーティー製アプリに公開していないため、Watch の直接ノードは
Canvas コマンドを公開しません。

## 公式ビルド向けのリレー経由プッシュ

公式に配布される iOS ビルドでは、生の APNs トークンを Gateway に公開する代わりに、外部プッシュリレーを使用します。公開リリースレーンの公式 App Store ビルドは、`https://ios-push-relay.openclaw.ai` のホスト型リレーを使用します。このベース URL は App Store 配布用にハードコードされており、オーバーライドを読み取りません。

カスタムリレーのデプロイには、リレー URL が Gateway のリレー URL と一致する、明確に分離された iOS ビルド／デプロイ経路が必要です。App Store リリースレーンでは、カスタムリレー URL は一切受け付けません。カスタムリレービルドを使用している場合は、一致する Gateway リレー URL を設定します。

```json5
{
  gateway: {
    push: {
      apns: {
        relay: {
          baseUrl: "https://relay.example.com",
        },
      },
    },
  },
}
```

フローの仕組み:

- iOS アプリは、App Attest と StoreKit アプリトランザクション JWS を使用してリレーに登録します。
- リレーは、不透明なリレーハンドルと登録スコープの送信許可を返します。
- iOS アプリはペアリング済み Gateway の ID（`gateway.identity.get`）を取得してリレー登録に含めるため、リレーを利用する登録はその特定の Gateway に委任されます。
- アプリは、そのリレーを利用する登録を `push.apns.register` でペアリング済み Gateway に転送します。
- Gateway は、`push.test`、バックグラウンド復帰、復帰促進に、保存されたリレーハンドルを使用します。
- その後、アプリが別の Gateway、またはリレーのベース URL が異なるビルドに接続した場合は、古いバインディングを再利用せずにリレー登録を更新します。

この経路で Gateway が**必要としないもの**: デプロイ全体のリレートークン、および公式 App Store ビルドがリレーを利用して送信するための直接の APNs キーは不要です。

想定されるオペレーターのフロー:

1. 公式 iOS アプリをインストールします。
2. 省略可能: 意図的に分離したカスタムリレービルドを使用する場合に限り、Gateway に `gateway.push.apns.relay.baseUrl` を設定します。
3. アプリを Gateway とペアリングし、接続が完了するまで待ちます。
4. APNs トークンを取得し、オペレーターセッションが接続され、リレー登録が成功すると、アプリは `push.apns.register` を公開します。
5. その後、`push.test`、再接続時の復帰、復帰促進で、保存されたリレーを利用する登録を使用できるようになります。

## バックグラウンド稼働ビーコン

サイレントプッシュ、バックグラウンド更新、または位置情報の大幅な変化イベントによって iOS がアプリを復帰させると、アプリは Node への短時間の再接続を試み、続いて `event: "node.presence.alive"` を指定して `node.event` を呼び出します。Gateway は、認証済み Node デバイスの ID が判明した後に限り、これをペアリング済み Node／デバイスのメタデータに `lastSeenAtMs`/`lastSeenReason` として記録します。

アプリは、Gateway の応答に `handled: true` が含まれる場合に限り、バックグラウンド復帰が正常に記録されたものとして扱います。古い Gateway は `{ "ok": true }` で `node.event` を確認応答することがあります。この応答には互換性がありますが、永続的な最終確認日時の更新としては扱われません。

互換性に関する注意:

- `OPENCLAW_APNS_RELAY_BASE_URL` は、Gateway の一時的な環境変数オーバーライドとして引き続き機能します（設定優先の経路は `gateway.push.apns.relay.baseUrl` です）。
- App Store リリースビルドのプッシュモードでは、ホストされるリレーのホストがハードコードされており、リレー URL のオーバーライドは読み取りません。ビルド時環境変数 `OPENCLAW_PUSH_RELAY_BASE_URL` が影響するのは、ローカル／サンドボックスの iOS ビルドモードだけです。

## 認証と信頼のフロー

リレーは、公式 iOS ビルドにおいて Gateway から APNs に直接送信する方式では実現できない、次の 2 つの制約を適用するために存在します。

- Apple を通じて配布された正規の OpenClaw iOS ビルドだけが、ホストされるリレーを使用できます。
- Gateway は、その特定の Gateway とペアリングした iOS デバイスにのみ、リレーを利用したプッシュを送信できます。

ホップごとの流れ:

1. `iOS app -> gateway`: アプリは通常の Gateway 認証フローを通じて Gateway とペアリングし、認証済み Node セッションと認証済みオペレーターセッションを取得します。オペレーターセッションは `gateway.identity.get` を呼び出します。
2. `iOS app -> relay`: アプリは、App Attest の証明と StoreKit アプリトランザクション JWS を使用し、HTTPS 経由でリレー登録エンドポイントを呼び出します。リレーはバンドル ID、App Attest の証明、Apple の配布証明を検証し、公式／本番の配布経路を必須とします。ローカルビルドでは公式の Apple 配布証明を満たせないため、これによりローカルの Xcode／開発ビルドがホストされるリレーを使用できないようにします。
3. `gateway identity delegation`: リレー登録の前に、アプリは `gateway.identity.get` からペアリング済み Gateway の ID を取得し、リレー登録ペイロードに含めます。リレーは、リレーハンドルと、その Gateway の ID に委任された登録スコープの送信許可を返します。
4. `gateway -> relay`: Gateway は、`push.apns.register` から取得したリレーハンドルと送信許可を保存します。`push.test`、再接続時の復帰、復帰促進では、Gateway が自身のデバイス ID で送信リクエストに署名します。リレーは、保存された送信許可と Gateway の署名の両方を、登録時に委任された Gateway の ID に照らして検証します。別の Gateway が何らかの方法でハンドルを取得しても、保存された登録を再利用することはできません。
5. `relay -> APNs`: リレーは、本番用 APNs 認証情報と公式ビルドの生の APNs トークンを管理します。リレーを利用する公式ビルドについて、Gateway が生の APNs トークンを保存することはありません。リレーがペアリング済み Gateway に代わって、最終的なプッシュを APNs に送信します。

この設計が作られた理由: 本番用 APNs 認証情報をユーザーの Gateway に置かず、公式ビルドの生の APNs トークンを Gateway に保存することを避け、公式 OpenClaw iOS ビルドに限ってホストされるリレーを利用できるようにし、ある Gateway が別の Gateway に属する iOS デバイスへ復帰プッシュを送信できないようにするためです。

ローカル／手動ビルドでは、引き続き APNs に直接送信します。リレーを使用せずにこれらのビルドをテストする場合、Gateway には引き続き直接の APNs 認証情報が必要です。

```bash
export OPENCLAW_APNS_TEAM_ID="TEAMID"
export OPENCLAW_APNS_KEY_ID="KEYID"
export OPENCLAW_APNS_PRIVATE_KEY_P8="$(cat /path/to/AuthKey_KEYID.p8)"
```

これらは Gateway ホストのランタイム環境変数であり、Fastlane の設定ではありません。`apps/ios/fastlane/.env` に保存されるのは `APP_STORE_CONNECT_KEY_ID` や `APP_STORE_CONNECT_ISSUER_ID` などの App Store Connect 認証情報だけであり、ローカル iOS ビルド向けの APNs 直接配信は設定されません。

`~/.openclaw/credentials/` 配下の他のプロバイダー認証情報と一貫性のある、推奨される Gateway ホスト上の保存方法:

```bash
mkdir -p ~/.openclaw/credentials/apns
chmod 700 ~/.openclaw/credentials/apns
mv /path/to/AuthKey_KEYID.p8 ~/.openclaw/credentials/apns/AuthKey_KEYID.p8
chmod 600 ~/.openclaw/credentials/apns/AuthKey_KEYID.p8
export OPENCLAW_APNS_PRIVATE_KEY_PATH="$HOME/.openclaw/credentials/apns/AuthKey_KEYID.p8"
```

`.p8` ファイルをコミットしたり、リポジトリのチェックアウト配下に置いたりしないでください。

## 検出経路

### Bonjour（LAN）

iOS アプリは `local.` 上で `_openclaw-gw._tcp` を参照し、設定されている場合は同じ広域 DNS-SD 検出ドメインも参照します。同一 LAN 内の Gateway は `local.` から自動的に表示されます。ネットワークをまたぐ検出では、ビーコンタイプを変更せずに、設定された広域ドメインを使用できます。

### Tailnet（ネットワーク間）

mDNS がブロックされている場合は、ユニキャスト DNS-SD ゾーン（ドメインを選択。例: `openclaw.internal.`）と Tailscale のスプリット DNS を使用します。CoreDNS の例については、[Bonjour](/ja-JP/gateway/bonjour) を参照してください。

### ホスト／ポートの手動指定

Settings で **Manual Host** を有効にし、Gateway のホストとポート（デフォルトは `18789`）を入力します。

## 複数の Gateway

アプリはペアリングしたすべての Gateway のレジストリを保持するため、再度ペアリングせずに切り替えられます。

- **Settings -> Gateway** には、アクティブな Gateway が示された **Paired Gateways** リストが表示されます。項目をタップすると切り替わります。アプリは現在のセッションを切断し、選択した Gateway に再接続します。複数の Gateway がペアリングされている場合は、接続行の横にクイック切り替えメニューが表示されます。
- 認証情報、TLS の信頼に関する判断、Gateway ごとの設定、キャッシュされたチャット履歴は、Gateway ごとに保存されます。切り替えても Gateway 間で状態が混在することはなく、プッシュ登録はアクティブな Gateway に従います。
- ペアリング済み Gateway をスワイプ（またはコンテキストメニューを使用）して **Forget** を選択すると、その認証情報、デバイストークン、TLS ピン、キャッシュされたチャットが削除されます。
- 検出された Gateway に切り替えるには、その Gateway がネットワーク上で認識可能である必要があります。手動設定した Gateway は、保存されたホストとポートを使用して再接続します。

## Canvas + A2UI

iOS Node は WKWebView Canvas をレンダリングします。`node.invoke` を使用して操作します。

```bash
openclaw nodes invoke --node "iOS Node" --command canvas.navigate --params '{"url":"http://<gateway-host>:18789/__openclaw__/canvas/"}'
```

注意:

- Gateway の Canvas ホストは、Gateway HTTP サーバー（`gateway.port` と同じポート、デフォルトは `18789`）から `/__openclaw__/canvas/` と `/__openclaw__/a2ui/` を配信します。
- iOS Node は、組み込みのスキャフォールドを接続時のデフォルトビューとして維持します。`canvas.a2ui.push` と `canvas.a2ui.reset` は、バンドルされたアプリ所有の A2UI ページを使用します。
- リモート Gateway の A2UI ページは、iOS ではレンダリング専用です。ネイティブ A2UI ボタンアクションは、バンドルされたアプリ所有のページからのみ受け付けられます。
- `canvas.navigate` と `{"url":""}` を使用して、組み込みのスキャフォールドに戻ります。

## Computer Use との関係

iOS アプリはモバイル Node サーフェスであり、Codex Computer Use のバックエンドではありません。Codex Computer Use と `cua-driver mcp` は MCP ツールを通じてローカルの macOS デスクトップを制御します。一方、iOS アプリは `canvas.*`、`camera.*`、`screen.*`、`location.*`、`talk.*` などの OpenClaw Node コマンドを通じて iPhone の機能を公開します。

エージェントは Node コマンドを呼び出すことで OpenClaw を通じて iOS アプリを操作できますが、これらの呼び出しは Gateway の Node プロトコルを経由し、iOS のフォアグラウンド／バックグラウンド制限に従います。ローカルデスクトップの制御には [Codex Computer Use](/ja-JP/plugins/codex-computer-use) を使用し、iOS Node の機能についてはこのページを参照してください。

### Canvas の評価／スナップショット

```bash
openclaw nodes invoke --node "iOS Node" --command canvas.eval --params '{"javaScript":"(() => { const {ctx} = window.__openclaw; ctx.clearRect(0,0,innerWidth,innerHeight); ctx.lineWidth=6; ctx.strokeStyle=\"#ff2d55\"; ctx.beginPath(); ctx.moveTo(40,40); ctx.lineTo(innerWidth-40, innerHeight-40); ctx.stroke(); return \"ok\"; })()"}'
```

```bash
openclaw nodes invoke --node "iOS Node" --command canvas.snapshot --params '{"maxWidth":900,"format":"jpeg"}'
```

## 音声による復帰 + トークモード

- 音声による復帰とトークモードは Settings で利用できます。
- `talk.realtime.transport` が `webrtc` の場合、OpenAI リアルタイム Talk はクライアント所有の WebRTC を使用します。明示的な `gateway-relay` 設定は引き続き Gateway が所有します。[トークモード](/ja-JP/nodes/talk) を参照してください。
- Talk 対応の iOS Node は `talk` 機能を公開し、`talk.ptt.start`、`talk.ptt.stop`、`talk.ptt.cancel`、`talk.ptt.once` を宣言できます。Gateway は、信頼済みの Talk 対応 Node に対して、これらのプッシュトゥトークコマンドをデフォルトで許可します。
- iOS はバックグラウンドオーディオを一時停止する場合があります。アプリがアクティブでない場合、音声機能はベストエフォートとして扱ってください。

## よくあるエラー

- `NODE_BACKGROUND_UNAVAILABLE`: iOS アプリをフォアグラウンドに移動してください（Canvas／カメラ／画面コマンドでは必要です）。
- `A2UI_HOST_UNAVAILABLE`: バンドルされた A2UI ページにアプリの WebView から到達できませんでした。アプリを Screen タブでフォアグラウンドに維持し、再試行してください。
- ペアリングのプロンプトが表示されない: `openclaw devices list` を実行し、手動で承認してください。
- Watch に iPhone の状態が表示されない: iPhone が `watch.status` で `watchPaired: true`
  と `watchAppInstalled: true` を報告していることを確認してください。ペアリングが false の場合は、Apple の Watch アプリで
  Watch をペアリングしてください。インストールが false の場合は、**My Watch -> Available Apps** から
  コンパニオンアプリをインストールしてください。いずれかを変更した後、Watch で OpenClaw を一度開いてください。
  即時の到達可能性には引き続き両方のアプリが実行中である必要がありますが、キューに入った更新は後で
  バックグラウンドで到着する場合があります。
- 再インストール後に再接続できない: Keychain のペアリングトークンが消去されています。Node を再ペアリングしてください。

## 関連ドキュメント

- [ペアリング](/ja-JP/channels/pairing)
- [検出](/ja-JP/gateway/discovery)
- [Bonjour](/ja-JP/gateway/bonjour)
