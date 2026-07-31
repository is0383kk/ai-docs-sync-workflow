---
read_when:
    - Android Node のペアリングまたは再接続
    - Android での Gateway 検出または認証のデバッグ
    - リモートの Mac から Android デバイスをミラーリングまたは操作する
    - クライアント間のチャット履歴の同等性を検証する
summary: Android アプリ（Node）：接続ランブック + Connect/Chat/OpenClaw/Voice/Canvas コマンドサーフェス
title: Android アプリ
x-i18n:
    generated_at: "2026-07-26T09:39:56Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: a134a678e26924abc24dd107c3feaad9d09e83e3829eef73514c7ef078d578f1
    source_path: platforms/android.md
    workflow: 16
---

<Note>
公式 Android アプリは [Google Play](https://play.google.com/store/apps/details?id=ai.openclaw.app&hl=en_IN) および、対応する [GitHub Releases](https://github.com/openclaw/openclaw/releases) の署名済みスタンドアロン APK として提供されています。これはコンパニオン Node であり、稼働中の OpenClaw Gateway が必要です。ソース: [apps/android](https://github.com/openclaw/openclaw/tree/main/apps/android)（[ビルド手順](https://github.com/openclaw/openclaw/blob/main/apps/android/README.md)）。
</Note>

## サポート概要

- 役割: コンパニオン Node アプリ（Android は Gateway をホストしません）。
- Gateway の要否: 必須（macOS、Linux、または WSL2 経由の Windows で実行します）。
- インストール: [Google Play](https://play.google.com/store/apps/details?id=ai.openclaw.app&hl=en_IN)、または対応する [GitHub Release](https://github.com/openclaw/openclaw/releases) から `OpenClaw-Android.apk` を入手し、Gateway の[はじめに](/ja-JP/start/getting-started)、続いて[ペアリング](/ja-JP/channels/pairing)を参照してください。
- Gateway: [運用手順書](/ja-JP/gateway) + [設定](/ja-JP/gateway/configuration)。
  - プロトコル: [Gateway プロトコル](/ja-JP/gateway/protocol)（Node + コントロールプレーン）。
- オペレーター接続に `operator.admin` があり、Gateway が `openclaw.chat` をサポートしている場合、**Settings → OpenClaw** を開くと専用の Gateway 設定アシスタントが起動します。そのセットアップ会話は通常のチャットとは分離されたままになり、秘密情報を含む返信はローカルで秘匿化され、**Open Chat** をタップした後にのみチャットへ移動します。

システム制御（launchd/systemd）は Gateway ホスト上で行います。[Gateway](/ja-JP/gateway)を参照してください。

## 複数の Gateway セッションの同時利用

各 Gateway を一度ずつペアリングしてから、**Settings → Gateway** を開きます。チェックマークはフォーカス中の Gateway を示し、各スイッチはフォーカスされていない Gateway のオペレーターセッションを接続したままにするかどうかを制御します。有効化された Gateway はアプリがフォアグラウンドにある間、それぞれ独立して再接続するため、フォーカスを切り替えても他の接続は切断されません。フォーカス中の Gateway だけが Android Node セッションとデバイス機能を使用できます。これにより、複数の Gateway が同じスマートフォンに対してカメラ、位置情報、画面、通知のコマンドを同時に発行することを防ぎます。アプリがフォアグラウンドを離れると、Android がセカンダリ接続を一時停止する場合があります。

## Wear OS コンパニオン

Wear OS コンパニオンは、ペアリング済み Android スマートフォンの認証済み Gateway 接続を使用します。ウォッチが Gateway の認証情報を受信または保存することはありません。エージェントとセッションの選択、範囲が制限されたトランスクリプトの閲覧、テキストまたは音声入力による返信の送信、実行中の処理の中止、選択したセッション内でのリアルタイム Talk の開始、ペアリング済みスマートフォンの Gateway への接続または切断が可能です。また、ローカル返信通知、ダークまたはライトの外観、返信のオプションの自動読み上げも提供します。スマートフォンとウォッチが異なるタイミングで更新されても対応できるよう、エージェントと Gateway の制御機能はケイパビリティネゴシエーションによって決定されます。リアルタイム Talk は、一時的な Wear OS Data Layer チャネル経由でマイク音声と再生音声をストリーミングし、選択したスマートフォン、Gateway 接続、または音声チャネルが失われると停止します。

## Google Play 以外からのインストール

通常の最終版および修正版の GitHub Releases には、ユニバーサル `OpenClaw-Android.apk` と `OpenClaw-Android-SHA256SUMS.txt` が含まれます。APK はリリースタグからビルドされ、OpenClaw Android リリースキーで署名され、GitHub Actions の来歴情報が付与されています。

両方のアセットが記載されている[リリース](https://github.com/openclaw/openclaw/releases)を選択し、サイドロードする前に、その正確なタグをダウンロードして検証します。

```bash
release_tag=vYYYY.M.PATCH
gh release download "$release_tag" \
  --repo openclaw/openclaw \
  --pattern OpenClaw-Android.apk \
  --pattern OpenClaw-Android-SHA256SUMS.txt
sha256sum --check OpenClaw-Android-SHA256SUMS.txt
gh attestation verify OpenClaw-Android.apk \
  --repo openclaw/openclaw \
  --signer-workflow openclaw/openclaw/.github/workflows/android-release.yml \
  --source-ref "refs/tags/${release_tag}" \
  --deny-self-hosted-runners
```

<Warning>
Google Play とスタンドアロン APK のインストールでは更新チャネルが異なり、署名元の識別情報も異なる場合があります。チャネルを切り替える際、Android では既存アプリのアンインストールが必要になることがあり、その場合はローカルのアプリデータが削除されます。通常の更新では同じチャネルを使用し続けてください。
</Warning>

## リモートの Mac から Android をミラーリングして操作する

[scrcpy](https://github.com/Genymobile/scrcpy) は、Android の画面を macOS のウィンドウにミラーリングし、Android Debug Bridge（ADB）を介してキーボードとポインターの入力を転送します。これは OpenClaw Node 接続とは別の、オペレーター側のワークフローです。Android デバイスと Mac が異なる場所にあり、同じプライベート Tailscale ネットワークを共有している場合に便利です。

### 始める前に

- Android デバイスと Mac に Tailscale をインストールし、両方を同じ tailnet に接続します。
- Android で **Developer options** と **USB debugging** を有効にします。Android 16 では **Wireless
  debugging** は **Settings > System > Developer options** にあります。[Android の開発者向け
  オプション](https://developer.android.com/studio/debug/dev-options)を参照してください。
- Mac に scrcpy と ADB をインストールします。

  ```bash
  brew install scrcpy
  brew install --cask android-platform-tools
  ```

- 最初の接続時には Android デバイスを操作できる状態にしておきます。各 Mac がデバイスを制御するには、その Mac の ADB キーを Android 側で承認する必要があります。

### TCP 経由の ADB を有効にする

初期セットアップでは、Android デバイスを USB で信頼できるコンピューターに接続し、デバッグの確認プロンプトを承認します。続いて以下を実行します。

```bash
adb devices
adb tcpip 5555
```

これで USB を切断できます。デバイスの再起動またはデバッグのリセット後にポート 5555 がリッスンしなくなった場合は、このローカルセットアップ手順を繰り返してください。Android 11 以降では、**Wireless debugging > Pair device with pairing code** と `adb pair` を使用して初期信頼を確立することもできます。

### コントローラーの Mac だけを許可する

制限付きの許可設定を持つ tailnet では、コントローラーの Mac から Android デバイスの TCP ポート 5555 への到達を明示的に許可する必要があります。以下の例のアドレスを 2 台のデバイスの固定 Tailscale IP に置き換え、範囲を限定したルールを tailnet ポリシーに追加します。

```json5
{
  grants: [
    {
      src: ["<remote-mac-tailnet-ip>"],
      dst: ["<android-tailnet-ip>"],
      ip: ["tcp:5555"],
    },
  ],
}
```

ホストエイリアスやその他のセレクターについては、[Tailscale grants](https://tailscale.com/docs/reference/syntax/grants)を参照してください。このポートを公開インターネットに許可したり、Funnel で公開したりしないでください。承認された ADB クライアントはデバイスを広範囲に制御できます。

### 接続してミラーリングを開始する

リモートの Mac で以下を実行します。

```bash
adb connect <android-tailnet-ip>:5555
adb devices
scrcpy --serial <android-tailnet-ip>:5555
```

この Mac から初めて `adb connect` を実行すると、Android に承認ダイアログが表示されます。デバイスのロックを解除してキーフィンガープリントを確認し、Mac が信頼できる場合にのみ **Always allow from this computer** を選択してください。成功した `adb devices` エントリの末尾は `device` になります。`unauthorized` は、デバイス上のプロンプトがまだ承認されていないことを示します。

scrcpy ウィンドウが開いたら、直接使用するか、[Peekaboo](https://peekaboo.sh/) などの macOS 画面自動化ツールから操作対象にします。scrcpy が画面表示と入力を伝送し、Tailscale はプライベートネットワーク経路のみを提供します。

### トラブルシューティング

- `Connection timed out`: TCP 5555 に対する tailnet の許可設定を確認してください。`tailscale ping` が成功しても証明されるのはピアへの到達性だけであり、ポリシーがこの TCP ポートを許可していることではありません。Mac から `nc -vz <android-tailnet-ip> 5555` でテストしてください。
- `unauthorized`: Android のロックを解除してリモート Mac の ADB キーを承認するか、**Wireless debugging > Paired devices** で古いワークステーションを削除してから再度ペアリングしてください。
- `Connection refused`: ローカルで再接続し、`adb tcpip 5555` を再度実行してください。
- 複数のデバイスが表示される: 明示的な `--serial <android-tailnet-ip>:5555` 引数を使用し続けてください。

終了したら、scrcpy を閉じて ADB を切断します。

```bash
adb disconnect <android-tailnet-ip>:5555
```

## 接続運用手順書

Android Node アプリ ⇄（mDNS/NSD + WebSocket）⇄ **Gateway**

Android は Gateway WebSocket に直接接続し、デバイスペアリング（`role: node`）を使用します。

Tailscale または公開ホストの場合、Android にはセキュアなエンドポイントが必要です。

- 推奨: `https://<magicdns>` / `wss://<magicdns>` を使用する Tailscale Serve / Funnel
- その他の対応方式: 実際の TLS エンドポイントを持つ任意の `wss://` Gateway URL
- 平文の `ws://` は、プライベート LAN アドレス / `.local` ホストに加え、`localhost`、`127.0.0.1`、Android エミュレーターブリッジ（`10.0.2.2`）でも引き続きサポートされます。非ループバックのセットアップでは、制限付きオペレーターアクセスが自動的に使用されます。

### 前提条件

- 別のマシンで Gateway が稼働していること（または SSH 経由で到達可能であること）。
- Android デバイスまたはエミュレーターから Gateway WebSocket に到達できること:
  - mDNS/NSD を使用する同じ LAN、**または**
  - Wide-Area Bonjour / ユニキャスト DNS-SD を使用する同じ Tailscale tailnet（以下を参照）、**または**
  - Gateway のホスト/ポートを手動指定（フォールバック）
- tailnet/公開モバイルペアリングでは、生の tailnet IP の `ws://` エンドポイントを使用しません。代わりに Tailscale Serve または別の `wss://` URL を使用してください。
- ペアリング要求を承認するため、Gateway マシン上（または SSH 経由）で `openclaw` CLI が使用可能であること。

### 1. Gateway を起動する

```bash
openclaw gateway --port 18789 --verbose
```

ログに次のような内容が表示されることを確認します。

- `listening on ws://0.0.0.0:18789`

Tailscale 経由で Android からリモートアクセスする場合は、生の tailnet バインドではなく Serve/Funnel を使用することを推奨します。

```bash
openclaw gateway --tailscale serve
```

これにより、Android でセキュアな `wss://` / `https://` エンドポイントを利用できます。初回のリモート Android ペアリングでは、TLS を別途終端しない限り、単純な `gateway.bind: "tailnet"` セットアップだけでは不十分です。

### 2. 検出を確認する（オプション）

Gateway マシンから以下を実行します。

```bash
dns-sd -B _openclaw-gw._tcp local.
```

デバッグに関するその他の注意事項は [Bonjour](/ja-JP/gateway/bonjour) を参照してください。

広域検出ドメインも設定している場合は、以下と比較してください。

```bash
openclaw gateway discover --json
```

これにより、TXT のヒントだけでなく解決済みサービスエンドポイントを使用し、`local.` と設定済みの広域ドメインが一度に表示されます。

#### ユニキャスト DNS-SD を使用したネットワーク間検出

Android の NSD/mDNS 検出はネットワークをまたぎません。Android Node と Gateway が異なるネットワーク上にあり、Tailscale 経由で接続されている場合は、代わりに Wide-Area Bonjour / ユニキャスト DNS-SD を使用します。tailnet/公開 Android ペアリングでは、検出だけでは不十分です。検出された経路にもセキュアなエンドポイント（`wss://` または Tailscale Serve）が必要です。

1. Gateway ホストに DNS-SD ゾーン（例: `openclaw.internal.`）を設定し、`_openclaw-gw._tcp` レコードを公開します。
2. 選択したドメインの参照先をその DNS サーバーにするよう、Tailscale のスプリット DNS を設定します。

詳細および CoreDNS の設定例については、[Bonjour](/ja-JP/gateway/bonjour)を参照してください。

### 3. Android から接続する

Android アプリで以下を行います。

- アプリは **foreground service**（常駐通知）を使用して Gateway 接続を維持します。
- **Connect** タブを開きます。
- **Setup Code** または **Manual** モードを使用します。
- 検出がブロックされる場合は、**Advanced controls** でホスト/ポートを手動指定します。プライベート LAN ホストでは、`ws://` を引き続き使用できます。Tailscale/公開ホストでは TLS を有効にし、`wss://` / Tailscale Serve エンドポイントを使用してください。

最初のペアリングに成功すると、Android は起動時にアクティブなペアリング済み Gateway へ自動的に再接続します（検出された Gateway についてはベストエフォートであり、ネットワーク上で可視である必要があります）。

公式セットアップコードは、Android を Node として接続し、デフォルトでは `wss://` 経由で Gateway オペレーターのフルアクセスを付与します。平文の非ループバック `ws://` セットアップでは、Bearer トークンの安全性を確保するため、自動的に制限付きアクセスが使用されます。**Settings → Gateway** には **Full** または **Limited** アクセスが表示されます。制限付き接続の場合は、`wss://` または Tailscale Serve を設定し、Control UI または `openclaw qr` を使用して新しいフルアクセスコードを生成してから、そのページでスキャンまたは貼り付けて再接続します。権限を縮小したプロファイルを使用するオペレーターは、Control UI で **Limited access** を選択するか、`openclaw qr --limited` を実行できます。

### ペアリング済み Gateway の管理

アプリはペアリングしたすべての Gateway のレジストリを保持するため、オペレーターセッションの接続を維持したまま、再度ペアリングせずにフォーカスを切り替えられます。

- **Settings → Gateway** にはペアリング済みの Gateway が一覧表示され、フォーカス中のものには印が付きます。項目をタップするとフォーカスが切り替わります。他の有効なオペレーターセッションは接続されたままです。
- 各スイッチでは、アプリがフォアグラウンドにある間、フォーカスされていない Gateway の接続を維持するかどうかを制御します。フォーカス中の Gateway は有効なままで、スマートフォンの Node 接続とデバイス機能を所有します。
- 複数の Gateway がペアリングされている場合、**Connect** タブにクイックスイッチャーが表示されます。
- 認証情報、デバイストークン、TLS の信頼情報、チャット履歴、キューに入ったオフラインメッセージは、Gateway ごとに保存されます。フォーカスを変更しても Gateway 間で状態が混在することはなく、オフライン中にキューへ追加されたメッセージは、そのメッセージの送信先として作成された Gateway にのみ配信されます。
- **Forget** は、Gateway のレジストリエントリと、その認証情報、デバイストークン、TLS ピン、キャッシュ済みチャットを削除します。

### プレゼンス稼働ビーコン

認証済み Node セッションが接続した後、およびフォアグラウンドサービスが接続されたままアプリがバックグラウンドに移行したとき、Android は `event: "node.presence.alive"` を指定して `node.event` を呼び出します。Gateway は、認証済み Node のデバイス ID が判明した後にのみ、ペアリング済み Node／デバイスのメタデータへこれを `lastSeenAtMs`/`lastSeenReason` として記録します。

Gateway の応答に `handled: true` が含まれる場合にのみ、アプリはビーコンが正常に記録されたとみなします。古い Gateway は、`{ "ok": true }` を指定して `node.event` を確認応答することがあります。この応答には互換性がありますが、永続的な最終確認時刻の更新としてはカウントされません。

### 4. ペアリングの承認（CLI）

Gateway マシン上で次を実行します。

```bash
openclaw devices list
openclaw devices approve <requestId>
openclaw devices reject <requestId>
```

ペアリングの詳細：[ペアリング](/ja-JP/channels/pairing)。

任意：Android Node が常に厳格に管理されたサブネットから接続する場合、明示的な CIDR または正確な IP を使用して、初回の Node 自動承認を有効化できます。

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

これはデフォルトでは無効です。要求されたスコープがない新規の `role: node` ペアリングにのみ適用されます。オペレーター／ブラウザーのペアリング、およびロール、スコープ、メタデータ、公開鍵の変更には、引き続き手動承認が必要です。

### 5. Node が接続されていることの確認

```bash
openclaw nodes status
openclaw gateway call node.list --params "{}"
```

### 6. チャットと履歴

Android の Chat タブでは、セッションを選択できます（デフォルトは `main`。その他の既存セッションも選択可能）。

- 履歴：`chat.history`（表示用に正規化済み — インラインディレクティブタグ、平文のツール呼び出し XML ペイロード（`<tool_call>`、`<function_call>`、`<tool_calls>`、`<function_calls>`、および切り詰められた派生形）、漏出した ASCII／全角のモデル制御トークンは除去されます。正確に `NO_REPLY` / `no_reply` であるサイレントトークンのアシスタント行などは省略されます。サイズ超過の行はプレースホルダーに置き換えられることがあります）
- 送信：`chat.send`
- 永続的な送信：すべての送信内容（テキスト、選択した画像、ボイスメモ）は、ネットワーク接続を試みる前に、Gateway ごとのデバイス内送信トレイへ記録されるため、アプリが終了しても送信済みの入力が失われることはありません。オフライン中にキューへ追加された送信内容は、再接続時に安定した冪等性キーを使用して順番に配信され、ターンが正規の `chat.history` に表示された後にのみ送信済みとして除去されます。確認応答だけでは配信の証明とはみなされません。不確定な結果（確認応答の消失、送信途中でのアプリ終了、トランスクリプト書き込み前の Gateway 再起動）は、自動再送信せず、明示的な **Retry**／**Delete** を備えた行として表示されます。スラッシュコマンドが再接続をまたいで自動的に再実行されることはなく、明示的な再試行を待つ状態になります。キューには上限があり（Gateway ごとに 50 件のメッセージと 48 MB の添付ファイルデータ）、未送信の行は 48 時間後に期限切れになります。送信されなかった入力欄の下書きは、プロセスをまたいで永続化されません。
- プッシュ更新（ベストエフォート）：`chat.subscribe` -> `event:"chat"`
- 読み上げ：アシスタントメッセージを長押しして **Listen** を選択すると、内容を音声で聞けます。音声は、設定済みの TTS プロバイダーチェーンを使用して Gateway の `tts.speak` により生成され、Gateway が音声を生成できない場合はデバイス内のシステム TTS が使用されます。セッションの切り替え、新しいチャットの開始、アプリのバックグラウンド移行、またはチャットを閉じると再生が停止します。

### 7. Canvas とカメラ

#### Gateway Canvas ホスト（Web コンテンツに推奨）

エージェントがディスク上で編集できる実際の HTML/CSS/JS を Node に表示させるには、Node の接続先を Gateway Canvas ホストに設定します。

<Note>
Node は Gateway HTTP サーバー（`gateway.port` と同じポート、デフォルトは `18789`）から Canvas を読み込みます。
</Note>

1. Gateway ホスト上に `~/.openclaw/workspace/canvas/index.html` を作成します。
2. Node からそこへ移動します（LAN）。

```bash
openclaw nodes invoke --node "<Android Node>" --command canvas.navigate --params '{"url":"http://<gateway-hostname>.local:18789/__openclaw__/canvas/"}'
```

Tailnet（任意）：両方のデバイスが Tailscale 上にある場合、`.local` の代わりに MagicDNS 名または tailnet IP（例：`http://<gateway-magicdns>:18789/__openclaw__/canvas/`）を使用します。

このサーバーは HTML にライブリロードクライアントを注入し、ファイルの変更時に再読み込みします。Gateway は `/__openclaw__/a2ui/` も提供しますが、Android アプリはリモート A2UI ページを表示専用として扱います。アクション対応の A2UI コマンドでは、アプリにバンドルされたアプリ所有の A2UI ページが使用されます。

Canvas コマンド（フォアグラウンドのみ）：

- `canvas.eval`、`canvas.snapshot`、`canvas.navigate`（デフォルトのスキャフォールドに戻るには `{"url":""}` または `{"url":"/"}` を使用します）。`canvas.snapshot` は `{ format, base64 }`（デフォルトは `format="jpeg"`）を返します。
- A2UI：`canvas.a2ui.push`、`canvas.a2ui.reset`（`canvas.a2ui.pushJSONL` はレガシーエイリアス）。これらは、アクション対応のレンダリングに、アプリにバンドルされたアプリ所有の A2UI ページを使用します。

カメラコマンド（フォアグラウンドのみ。権限が必要）：`camera.snap`（jpg）、`camera.clip`（mp4）。パラメーターと CLI ヘルパーについては、[カメラ Node](/ja-JP/nodes/camera)を参照してください。

### 8. 音声と拡張された Android コマンドサーフェス

- Android のシェルナビゲーションは **Home**、**Chat**、**Settings** です。音声入力は Chat の入力欄に含まれ、独立した Voice タブはありません。
- 入力欄のマイクをタップすると、デバイス内音声認識によってトランスクリプトが下書きに挿入されます。マイクを長押しすると、ボイスメモの添付ファイルを録音します。UI は試行を黙って破棄せず、音声認識が利用できない場合、権限がない場合、ビジー状態／ネットワーク障害、および音声が検出されなかった結果を報告します。
- Chat の波形から連続 **Talk** を開始します。ディクテーション、ボイスメモ録音、Talk は、互いに排他的なマイク使用経路です。
- Talk Mode は、キャプチャの開始前に既存のフォアグラウンドサービスを `connectedDevice` から `connectedDevice|microphone` に昇格させ、Talk Mode の停止時に降格させます。Node サービスは、`CHANGE_NETWORK_STATE` を指定して `FOREGROUND_SERVICE_CONNECTED_DEVICE` を宣言します。Android 14 以降では、`FOREGROUND_SERVICE_MICROPHONE` の宣言、`RECORD_AUDIO` のランタイム許可、ランタイムでのマイクサービス種別も必要です。
- デフォルトでは、Android Talk はネイティブ音声認識、Gateway チャット、および設定済みの Gateway Talk プロバイダー経由の `talk.speak` を使用します。ローカルのシステム TTS は、`talk.speak` が利用できない場合にのみ使用されます。
- Android Talk がリアルタイム Gateway リレーを使用するのは、`talk.realtime.mode` が `realtime` であり、かつ `talk.realtime.transport` が `gateway-relay` の場合のみです。
- Android は `voiceWake` 機能を公開しません。音声入力には、Chat のディクテーション、ボイスメモ、または Talk を使用します。
- その他の Android コマンドファミリー（利用可否はデバイス、権限、ユーザー設定によって異なります）：
  - `device.status`、`device.info`、`device.permissions`、`device.health`
  - `device.apps` は、**Settings > Phone Capabilities > Installed Apps** が有効な場合にのみ使用できます。デフォルトではランチャーに表示されるアプリを一覧表示します（完全な一覧を取得するには `includeNonLaunchable` を渡します）。
  - `notifications.list`、`notifications.actions`（後述の[通知転送](#notification-forwarding)を参照）
  - `photos.latest`
  - `contacts.search`、`contacts.add`
  - `calendar.events`、`calendar.add`
  - `callLog.search`
  - `sms.search`
  - `motion.activity`、`motion.pedometer`

### 9. ワークスペースファイル（読み取り専用）

Home の概要には **Files** カードがあり、読み取り専用の `agents.workspace.list` / `agents.workspace.get` Gateway RPC を介して、アクティブなエージェントのワークスペースを参照できます。ディレクトリの階層移動、テキストと画像のプレビュー、Android 共有シートを介したエクスポートに対応しています。書き込み操作はなく、プレビューのサイズは Gateway によって制限されます。

## コマンド承認のレビュー

`operator.admin` を持つオペレーター接続、または Gateway が明示的に対象として指定したペアリング済みの `operator.approvals` 接続は、**Settings -> Approvals** で保留中の exec リクエストをレビューできます。アプリはボタンを有効にする前に Gateway のサニタイズ済み承認レコードを読み込み、セキュリティ警告と、そのリクエストで提示される正確な選択肢を表示し、承認 ID と所有者種別を Gateway に送信します。

承認状態は Control UI および対応するチャットサーフェスと共有されます。最初に確定された回答が優先されます。別のサーフェスが先に回答した場合でも、Android はその正規の結果を表示します。解決応答が失われた場合や Gateway が切断された場合、アプリはアクションをロックしたままにし、別の判断を提示する前に承認を再度読み込みます。

統合承認メソッドより前の Gateway では、リリース済みの exec 固有メソッドへフォールバックします。保留中のレビューは引き続き機能しますが、保持されるターミナル状態と、より詳細なサーフェス横断の結果を利用するには、Gateway の更新が必要です。

## エージェントからの質問への回答

Chat は、`operator.questions`（または `operator.admin`）を持つオペレーター接続に対し、保留中の Gateway の質問をネイティブカードとして表示します。カードは、単一選択と複数選択のオプション、オプションの説明、自由入力の **Other** 回答、期限切れまでのカウントダウンに対応しています。再接続時には、Gateway から保留中の質問が再読み込みされます。このデバイスが回答した場合、別のサーフェスが先に回答した場合、または質問が期限切れもしくはキャンセルされた場合、カードはロックされます。

## アシスタントのエントリーポイント

Android は、システムアシスタントのトリガー（Google Assistant）から OpenClaw を起動できます。ホームボタンを長押しする（または別の `ACTION_ASSIST` トリガーを使用する）とアプリが開きます。「Hey Google, ask OpenClaw `<prompt>`」と話すと、アプリで宣言された App Actions のクエリパターンに一致し、プロンプトが自動送信されずにチャット入力欄へ渡されます。

これは、アプリマニフェストで宣言された Android の **App Actions**（`shortcuts.xml` 機能）を使用します。Gateway 側の設定は不要です。アシスタントのインテントは Android アプリ内で完全に処理されます。

<Note>
App Actions を利用できるかどうかは、デバイス、Google Play Services のバージョン、およびユーザーが OpenClaw をデフォルトのアシスタントアプリとして設定しているかどうかによって異なります。
</Note>

## 通知転送

Android は、デバイスの通知を `node.event` 項目として Gateway に転送できます。これは Gateway／`openclaw.json` の設定ではなく、**デバイス上**のアプリの Settings シートで設定します。

| 設定                        | 説明                                                                                                                                                                                           |
| --------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Forward Notification Events | マスタートグル。デフォルトではオフです。最初に Notification Listener Access を許可する必要があります。                                                                                         |
| Package Filter              | **Allowlist**（一覧にあるパッケージ ID のみ転送）または **Blocklist**（デフォルト：一覧にある ID を除くすべてのパッケージ）。転送ループを防ぐため、Blocklist モードでは OpenClaw 自身のパッケージが常に除外されます。 |
| Quiet Hours                 | 転送を抑制する、ローカル時刻の HH:mm 形式の開始／終了時間帯。デフォルトでは無効です。有効にすると、デフォルトは `22:00`-`07:00` です。                                  |
| Max Events / Minute         | 転送される通知に対するデバイスごとのレート制限。デフォルトは 20 です。                                                                                                                        |
| Route Session Key           | 任意。転送された通知イベントを、デバイスのデフォルト通知ルートではなく、特定のセッションに固定します。                                                                                       |

<Note>
通知の転送には、Android の Notification Listener 権限が必要です。アプリはセットアップ中にこの権限を求めます。
</Note>

WhatsApp、WhatsApp Business、Telegram、Telegram X、Discord、Signal の通知は常に除外されます。これらのメッセージはすでに OpenClaw のネイティブチャネルセッションによって管理されています。Android の通知を個別の Node イベントとして転送すると、返信が誤った会話にルーティングされる可能性があります。

## 関連項目

- [iOS アプリ](/ja-JP/platforms/ios)
- [Node](/ja-JP/nodes)
- [Android Node のトラブルシューティング](/ja-JP/nodes/troubleshooting)
