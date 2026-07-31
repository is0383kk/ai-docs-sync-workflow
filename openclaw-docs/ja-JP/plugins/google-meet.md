---
read_when:
    - OpenClaw エージェントを Google Meet の通話に参加させたい場合
    - OpenClaw エージェントに新しい Google Meet 通話を作成させたい場合
    - Google Meet のトランスポートとして Chrome、Chrome node、または Twilio を設定しています
summary: Google Meet Plugin：Chrome または Twilio を介して明示的な Meet URL に参加し、エージェントの応答発話をデフォルトで有効化
title: Google Meet Plugin
x-i18n:
    generated_at: "2026-07-26T09:08:51Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 8a611e283fe900984a29b563969936a641c7af430b05933eb03b98dc93b5d0c8
    source_path: plugins/google-meet.md
    workflow: 16
---

`google-meet` Plugin は、OpenClaw エージェントに代わって明示的な Meet URL に参加します。意図的に対象を限定しています。

- `https://meet.google.com/...` URL にのみ参加します。自ら検出した電話番号を使って会議にダイヤルインすることはありません。
- `googlemeet create` は、Google Meet API（またはブラウザのフォールバック）を介して新しい Meet URL を発行し、デフォルトでその会議に参加できます。
- Chrome での参加には、ログイン済みの Chrome プロファイルを使用します。必要に応じて、ペアリング済み Node 上で実行できます。Twilio での参加では、[音声通話 Plugin](/ja-JP/plugins/voice-call)を介して電話番号と PIN/DTMF をダイヤルします。Meet URL に直接ダイヤルすることはできません。
- `mode: "agent"`（デフォルト）は、リアルタイムプロバイダーで参加者の発話を文字起こしし、設定済みの OpenClaw エージェントに転送して、通常の OpenClaw TTS で回答を読み上げます。`mode: "bidi"` では、リアルタイム音声モデルが直接回答します。`mode: "transcribe"` は、応答せず監視のみで参加します。
- Plugin が通話に参加した際、同意に関するアナウンスは自動では行われません。
- CLI コマンドは `googlemeet` です。`meet` は、より広範なエージェントの電話会議ワークフロー用に予約されています。

## クイックスタート

Plugin とローカル音声依存関係をインストールし、リアルタイムプロバイダーのキーを設定します。OpenAI は `agent` モードのデフォルトの文字起こしプロバイダーです。Google Gemini Live は `bidi` モードの音声プロバイダーとして使用できます。

```bash
openclaw plugins install npm:@openclaw/google-meet
brew install blackhole-2ch sox
export OPENAI_API_KEY=sk-...
# bidi モードで realtime.voiceProvider が "google" の場合にのみ必要
export GEMINI_API_KEY=...
```

`blackhole-2ch` は、Chrome が音声を経由させる `BlackHole 2ch` 仮想オーディオデバイスをインストールします。Homebrew のインストーラーでは、macOS がデバイスを認識する前に再起動が必要です。

```bash
sudo reboot
```

再起動後、両方を確認します。

```bash
system_profiler SPAudioDataType | grep -i BlackHole
command -v sox
```

Plugin はインストール後、デフォルトで有効になります。カスタマイズする場合にのみエントリを追加してください。

```json5
{
  plugins: {
    entries: {
      "google-meet": {
        config: {},
      },
    },
  },
}
```

Plugin を有効にしたくない場合は、`openclaw plugins disable google-meet` を実行します。

セットアップを確認してから参加します。

```bash
openclaw googlemeet setup
openclaw googlemeet join https://meet.google.com/abc-defg-hij
```

`setup` の出力はエージェントが読み取れる形式で、モードとトランスポートに対応しています。Chrome プロファイル、Node の固定、およびリアルタイム Chrome 参加の場合は BlackHole/SoX オーディオブリッジと遅延イントロの確認結果を報告します。監視のみの参加では、リアルタイムの前提条件を確認しません。

```bash
openclaw googlemeet setup --transport chrome-node --mode transcribe
```

Twilio への委任が設定されている場合、`setup` は、`voice-call`、Twilio 認証情報、および公開 Webhook の外部公開が準備できているかどうかも報告します。エージェントが参加する前に、該当するトランスポート／モードの `ok: false` チェックをすべてブロッカーとして扱ってください。機械可読出力には `--json` を使用し、特定のトランスポートを事前にプリフライトするには `--transport chrome|chrome-node|twilio` を使用します。

```bash
openclaw googlemeet setup --transport twilio
```

または、エージェントに `google_meet` ツールを介して参加させます。

```json
{
  "action": "join",
  "url": "https://meet.google.com/abc-defg-hij",
  "transport": "chrome-node",
  "mode": "agent"
}
```

macOS 以外の Gateway ホストでも、`google_meet` はアーティファクト、カレンダー、セットアップ、文字起こし、Twilio、および `chrome-node` アクションで引き続き表示されます。ただし、ローカル Chrome の音声応答（`mode: "agent"` または `"bidi"` を伴う `transport: "chrome"`）は、オーディオブリッジに到達する前にブロックされます。この経路は現在、macOS の `BlackHole 2ch` に依存しているためです。代わりに、`mode: "transcribe"`、Twilio ダイヤルイン、または macOS の `chrome-node` ホストを使用してください。

### 会議を作成する

```bash
openclaw googlemeet create --transport chrome-node --mode agent
openclaw googlemeet create --no-join
```

`create` には 2 つの経路があり、結果の `source` フィールドで報告されます。

- **`api`**：Google Meet OAuth 認証情報が設定されている場合に使用されます。決定的に動作し、ブラウザ UI の状態には依存しません。
- **`browser`**：OAuth 認証情報がない場合に使用されます。OpenClaw は固定された Chrome Node 上で `https://meet.google.com/new` を開き、Google が実際の会議コード URL にリダイレクトするまで待機します。その Node 上の OpenClaw Chrome プロファイルは、事前に Google にログインしている必要があります。参加と作成のどちらでも、新しいタブを開く前に、既存の Meet タブ（または処理中の `.../new`／Google アカウントのプロンプトタブ）を再利用します。タブの照合では、`authuser` のような無害なクエリ文字列は無視されます。

`create` はデフォルトで参加し、`joined: true` と参加セッションを返します。URL のみを発行するには、`--no-join`（CLI）または `"join": false`（ツール）を渡します。

API で作成したルームでは、Google アカウントのデフォルト設定を継承せず、明示的なアクセスポリシーを設定します。

```bash
openclaw googlemeet create --access-type OPEN --transport chrome-node --mode agent
```

| `--access-type` | ノックせずに参加できるユーザー                                       |
| --------------- | ------------------------------------------------------------------- |
| `OPEN`          | Meet URL を知っているすべてのユーザー                                            |
| `TRUSTED`       | ホスト組織の信頼済みユーザー、招待された外部ユーザー、およびダイヤルインユーザー |
| `RESTRICTED`    | 招待されたユーザーのみ                                                       |

これは API で作成したルームにのみ適用されるため、OAuth を設定する必要があります。このオプションが追加される前に認証した場合は、OAuth 同意画面に `meetings.space.settings` スコープを追加した後、`openclaw googlemeet auth login --json` を再実行してください。

ブラウザのフォールバックで Google ログインまたは Meet の権限に関するブロッカーが発生した場合、ツールは `manualActionReason`、`manualActionMessage`、および `browser.nodeId`／`browser.targetId`／`browserUrl` とともに `manualActionRequired: true` を返します。そのメッセージを報告し、オペレーターがブラウザでの手順を完了するまで、新しい Meet タブを開かないでください。

### 監視のみで参加する

`"mode": "transcribe"` を設定すると、双方向リアルタイムブリッジをスキップします（BlackHole／SoX は不要で、音声応答もありません）。文字起こしモードの Chrome 参加では、OpenClaw によるマイク／カメラの権限付与と Meet の **Use microphone** 経路もスキップします。Meet に音声選択の中間画面が表示された場合、自動処理は最初に **Continue without microphone** を試みます。管理対象の Chrome トランスポートは、すべてのモードでベストエフォートの Meet 字幕オブザーバーをインストールします。これにより、ライブのエージェント相談経路を変更せずに、永続的なメモを利用できます。`googlemeet status --json` と `googlemeet doctor` は、`captioning`、`captionsEnabledAttempted`、`transcriptLines`、`lastCaptionAt`、`lastCaptionSpeaker`、`lastCaptionText`、および `recentTranscript` の末尾を報告します。

範囲が制限されたセッショントランスクリプトについては、追跡対象の正確な Meet タブを読み取ります。

```bash
openclaw googlemeet transcript <session-id>
openclaw googlemeet transcript <session-id> --since <next-index> --json
```

オブザーバーは Meet ページ内に完了済みの字幕行を最大 2,000 行保持します。表示中の進行中テキストは、字幕行が完了するまでステータスのヘルス末尾に残るため、`nextIndex` を保存しても、その後のテキスト展開が欠落することはありません。退出時には、スナップショットの前に表示中の行が確定されます。上限を超えた場合、`droppedLines` は先頭から失われた行を報告します。範囲が制限された `googlemeet transcript` の末尾では、直近に終了した 4 セッションのみを保持し、Gateway とともにリセットされます。これとは別に、OpenClaw は会議中、完了した字幕行を共有状態データベースに追記し、退出時に派生サマリーを書き込みます。永続的なメモを確認またはエクスポートするには、[`openclaw transcripts`](/ja-JP/cli/transcripts)を使用します。

自動メモはデフォルトで有効です。永続的なメモをグローバルに
無効にするには、`transcripts.enabled: false` を設定します。明示的な `transcribe` モードでも、
公開されるのは範囲が制限されたライブ末尾のみです。Twilio 参加にはブラウザの字幕ストリームがなく、
この経路では記録されません。

はい／いいえで判定するリスニングプローブの場合：

```bash
openclaw googlemeet test-listen <meet-url> --transport chrome-node
```

文字起こしモードで参加し、新しい字幕／トランスクリプトの更新を待って、`listenVerified`、`listenTimedOut`、手動操作フィールド、および現在の字幕ヘルスを返します。

### リアルタイムセッションのヘルス

音声応答セッション中、`google_meet` のステータスは Chrome／オーディオブリッジのヘルスを報告します。対象には、`inCall`、`manualActionRequired`、`providerConnected`、`realtimeReady`、`audioInputActive`、`audioOutputActive`、最後の入出力タイムスタンプ、バイトカウンター、およびブリッジが閉じている状態が含まれます。管理対象の Chrome セッションでは、ヘルスが `inCall: true` を報告した後にのみ、イントロ／テストフレーズを読み上げます。それ以外の場合は `speechReady: false` となり、何もせず暗黙に終了するのではなく、発話の試行がブロックされます。

ローカル Chrome は、ログイン済みの OpenClaw ブラウザプロファイルを介して参加し、マイク／スピーカー経路には `BlackHole 2ch` が必要です。最初のスモークテストには BlackHole デバイス 1 台で十分ですが、エコーが発生する場合があります。クリーンな双方向音声には、別々の仮想デバイスまたは Loopback 形式のグラフを使用してください。

## ローカル Gateway + Parallels Chrome

macOS VM に Chrome を提供するだけであれば、その VM 内に完全な Gateway やモデル API キーは必要ありません。Gateway とエージェントはローカルで実行し、Node ホストを VM 内で実行します。

| 実行場所           | 内容                                                                                            |
| -------------------- | ----------------------------------------------------------------------------------------------- |
| Gateway ホスト         | OpenClaw Gateway、エージェントワークスペース、モデル／API キー、リアルタイムプロバイダー、Google Meet Plugin 設定 |
| Parallels macOS VM   | OpenClaw CLI／Node ホスト、Chrome、SoX、BlackHole 2ch、Google にログイン済みの Chrome プロファイル        |
| VM では不要 | Gateway サービス、エージェント設定、モデルプロバイダーのセットアップ                                             |

VM の依存関係をインストールし、再起動して確認します。

```bash
brew install blackhole-2ch sox
sudo reboot
system_profiler SPAudioDataType | grep -i BlackHole
command -v sox
```

デフォルトで有効になる Plugin を VM にインストールし、Node ホストを起動します。

```bash
openclaw plugins install npm:@openclaw/google-meet
openclaw node run --host <gateway-host> --port 18789 --display-name parallels-macos
```

`<gateway-host>` が TLS を使用しない LAN IP の場合は、その信頼できるプライベートネットワークの使用を明示的に許可します。

```bash
OPENCLAW_ALLOW_INSECURE_PRIVATE_WS=1 \
  openclaw node run --host <gateway-lan-ip> --port 18789 --display-name parallels-macos
```

LaunchAgent としてインストールする場合も同じフラグを使用します（これはプロセス環境であり、インストールコマンドに指定されている場合は LaunchAgent 環境に保存されます。`openclaw.json` の設定ではありません）。

```bash
OPENCLAW_ALLOW_INSECURE_PRIVATE_WS=1 \
  openclaw node install --host <gateway-lan-ip> --port 18789 --display-name parallels-macos --force
openclaw node restart
```

Gateway ホストから Node を承認し、`googlemeet.chrome` とブラウザ機能／`browser.proxy` の両方が公開されていることを確認します。

```bash
openclaw devices list
openclaw devices approve <requestId>
openclaw nodes status
```

Meet をその Node 経由にルーティングします。

```json5
{
  gateway: {
    nodes: {
      commands: { allow: ["googlemeet.chrome", "browser.proxy"] },
    },
  },
  plugins: {
    entries: {
      "google-meet": {
        enabled: true,
        config: {
          defaultTransport: "chrome-node",
          chrome: {
            guestName: "OpenClaw Agent",
            autoJoin: true,
            reuseExistingTab: true,
          },
          chromeNode: {
            node: "parallels-macos",
          },
        },
      },
    },
  },
}
```

これで、Gateway ホストから通常どおり参加できます。

```bash
openclaw googlemeet join https://meet.google.com/abc-defg-hij
```

セッションを作成または再利用し、既知のフレーズを読み上げて、セッションのヘルスを出力するワンコマンドのスモークテスト：

```bash
openclaw googlemeet test-speech https://meet.google.com/abc-defg-hij
```

リアルタイム参加中、ブラウザ自動化はゲスト名を入力し、Join/Ask to join をクリックし、Meet の初回実行時の「Use microphone」プロンプトが表示された場合は承認します（監視専用の参加およびブラウザのみでの会議作成時は「Continue without microphone」）。プロファイルがサインアウトしている、Meet がホストによる参加承認を待っている、Chrome にマイク/カメラの権限が必要、または Meet が未解決のプロンプトで停止している場合、結果は `manualActionReason` および `manualActionMessage` とともに `manualActionRequired: true` を報告します。再試行を停止し、そのメッセージと `browserUrl`/`browserTitle` を報告して、手動操作の完了後にのみ再試行してください。

`chromeNode.node` を省略した場合、接続中の Node のうち `googlemeet.chrome` とブラウザ制御の両方を通知するものが正確に 1 つだけである場合に限り、OpenClaw が自動選択します。対応可能な Node が複数接続されている場合は、`chromeNode.node`（Node ID、表示名、またはリモート IP）を固定してください。

### 一般的な障害チェック

| 症状                                                  | 修正                                                                                                                                                                                                                                                                                   |
| -------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `Configured Google Meet node ... is not usable: offline` | 固定された Node は認識されていますが、利用できません。セットアップを妨げている問題を報告し、要求されない限り別のトランスポートへ暗黙的にフォールバックしないでください。                                                                                                                                                      |
| `No connected Google Meet-capable node`                  | VM に `npm:@openclaw/google-meet` をインストールし、`openclaw plugins enable browser` を実行して、`openclaw node run` を起動し、ペアリングを承認してください。Google Meet が明示的に無効化されている場合は、それも有効にしてください。`gateway.nodes.commands.allow` に `googlemeet.chrome` と `browser.proxy` が含まれていることを確認してください。 |
| `BlackHole 2ch audio device not found`                   | 確認対象のホストに `blackhole-2ch` をインストールし、再起動してください。                                                                                                                                                                                                                         |
| `BlackHole 2ch audio device not found on the node`       | VM に `blackhole-2ch` をインストールし、VM を再起動してください。                                                                                                                                                                                                                                  |
| Chrome は開くが参加できない                             | VM 内のブラウザプロファイルにサインインするか、`chrome.guestName` を設定したままにしてください。ゲストの自動参加では、Node のブラウザプロキシを介した OpenClaw ブラウザ自動化を使用します。Node の `browser.defaultProfile`（または名前付きの既存セッションプロファイル）を、使用するプロファイルに指定してください。                   |
| Meet タブが重複する                                      | `chrome.reuseExistingTab: true` のままにしてください。OpenClaw は同じ URL の既存タブをアクティブ化し、別のタブを開く前に、作成処理で進行中の `.../new` または Google アカウントのプロンプトタブを再利用します。                                                                                        |
| 音声がない                                                 | Meet のマイク/スピーカーを、OpenClaw が使用する仮想オーディオパス経由でルーティングしてください。クリーンな全二重音声には、個別の仮想デバイスまたは Loopback 形式のルーティングを使用してください。                                                                                                                                |

## インストールに関する注意事項

Chrome のトークバックのデフォルトでは、OpenClaw がバンドルも再配布もしない 2 つの外部ツールを使用します。Homebrew を介してホスト依存関係としてインストールしてください。

- `sox`: コマンドライン音声ユーティリティ。Plugin は、デフォルトの 24 kHz PCM16 オーディオブリッジ向けに明示的な CoreAudio デバイスコマンドを発行します。
- `blackhole-2ch`: Chrome/Meet が経由する `BlackHole 2ch` デバイスを提供する macOS 仮想オーディオドライバー。

SoX は `LGPL-2.0-only AND GPL-2.0-only` でライセンスされています。BlackHole は GPL-3.0 です。BlackHole と OpenClaw をバンドルするインストーラーまたはアプライアンスを構築する場合は、BlackHole のアップストリームライセンスを確認するか、Existential Audio から別途ライセンスを取得してください。

## トランスポート

| トランスポート     | 使用する状況                                                                                     |
| ------------- | -------------------------------------------------------------------------------------------- |
| `chrome`      | Chrome/音声が Gateway ホスト上で動作する場合                                                        |
| `chrome-node` | Chrome/音声がペアリング済み Node 上（例: Parallels macOS VM）で動作する場合                        |
| `twilio`      | Chrome での参加を利用できない場合に、Voice Call Plugin を介して電話ダイヤルインへフォールバックする場合 |

### Chrome

OpenClaw ブラウザ制御を介して Meet URL を開き、サインイン済みの OpenClaw ブラウザプロファイルとして参加します。macOS では、Plugin は起動前に `BlackHole 2ch` を確認し、設定されている場合は Chrome を開く前にオーディオブリッジの正常性確認/起動コマンドを実行します。ローカル Chrome の場合は `browser.defaultProfile` でプロファイルを選択してください。代わりに `chrome.browserProfile` が `chrome-node` ホストへ渡されます。

```bash
openclaw googlemeet join https://meet.google.com/abc-defg-hij --transport chrome
openclaw googlemeet join https://meet.google.com/abc-defg-hij --transport chrome-node
```

Chrome のマイク/スピーカー音声は、ローカルの OpenClaw オーディオブリッジを経由します。`BlackHole 2ch` がインストールされていない場合、音声パスなしで参加するのではなく、セットアップエラーで参加に失敗します。

### Twilio

[Voice Call Plugin](/ja-JP/plugins/voice-call) に委任される厳密なダイヤルプランです。Meet ページから電話番号を解析することはありません。Google Meet が会議用の電話ダイヤルイン番号と PIN を公開している必要があります。

Voice Call は Chrome Node ではなく、Gateway ホストで有効にしてください。

```json5
{
  plugins: {
    allow: ["google-meet", "voice-call", "google"],
    entries: {
      "google-meet": {
        enabled: true,
        config: {
          defaultTransport: "chrome-node",
          // Twilio をデフォルトにする場合は、代わりに "twilio" を設定
        },
      },
      "voice-call": {
        enabled: true,
        config: {
          provider: "twilio",
          inboundPolicy: "allowlist",
          realtime: {
            enabled: true,
            provider: "google",
            instructions: "OpenClaw エージェントとしてこの Google Meet に参加してください。簡潔に応答してください。",
            toolPolicy: "safe-read-only",
            providers: {
              google: {
                silenceDurationMs: 500,
                startSensitivity: "high",
              },
            },
          },
        },
      },
      google: {
        enabled: true,
      },
    },
  },
}
```

シークレットを `openclaw.json` に含めないように、Twilio 認証情報は環境変数で指定してください。

```bash
export TWILIO_ACCOUNT_SID=AC...
export TWILIO_AUTH_TOKEN=...
export TWILIO_FROM_NUMBER=+15550001234
export GEMINI_API_KEY=...
```

OpenAI がリアルタイム音声プロバイダーの場合は、代わりに `OPENAI_API_KEY` とともに `realtime.provider: "openai"` を使用してください。

`voice-call` を有効にした後、Gateway を再起動または再読み込みしてください。Plugin 設定の変更は再読み込みするまで反映されません。以下で確認します。

```bash
openclaw config validate
openclaw plugins list | grep -E 'google-meet|voice-call'
openclaw googlemeet setup
```

Twilio 委任が接続されている場合、`googlemeet setup` には `twilio-voice-call-plugin`、`twilio-voice-call-credentials`、および `twilio-voice-call-webhook` のチェックが含まれます。

```bash
openclaw googlemeet join https://meet.google.com/abc-defg-hij \
  --transport twilio \
  --dial-in-number +15551234567 \
  --pin 123456
```

カスタムシーケンスには `--dtmf-sequence` を使用し、PIN の前に一時停止するには先頭に `w` またはコンマを付けます。

```bash
openclaw googlemeet join https://meet.google.com/abc-defg-hij \
  --transport twilio \
  --dial-in-number +15551234567 \
  --dtmf-sequence ww123456#
```

## OAuth と事前確認

`googlemeet create` はブラウザ自動化へフォールバックできるため、Meet リンクの作成に OAuth は任意です。公式 API による作成、スペース解決、または Meet Media API の事前確認には OAuth を設定してください。Chrome/Chrome-node による参加は OAuth に依存しません。どちらの場合も、サインイン済みの Chrome プロファイル、BlackHole/SoX、および（`chrome-node` の場合は）接続済み Node を使用します。

### Google 認証情報を作成する

Google Cloud Console で次の操作を行います。

<Steps>
<Step title="プロジェクトを作成または選択する">
</Step>
<Step title="Google Meet REST API を有効にする">
</Step>
<Step title="OAuth 同意画面を設定する">
Google Workspace 組織では Internal が最も簡単です。個人用/テスト用のセットアップでは External を使用できます。アプリが Testing の間は、認可に使用する各 Google アカウントをテストユーザーとして追加してください。
</Step>
<Step title="要求されたスコープを追加する">
- `https://www.googleapis.com/auth/meetings.space.created`
- `https://www.googleapis.com/auth/meetings.space.readonly`
- `https://www.googleapis.com/auth/meetings.space.settings`
- `https://www.googleapis.com/auth/meetings.conference.media.readonly`
- `https://www.googleapis.com/auth/calendar.events.readonly`（カレンダー検索）
- `https://www.googleapis.com/auth/drive.meet.readonly`（文字起こし/スマートノートのドキュメント本文エクスポート）

</Step>
<Step title="OAuth クライアント ID を作成する">
アプリケーションの種類は **Web application**。承認済みのリダイレクト URI:

```text
http://localhost:8085/oauth2callback
```

</Step>
<Step title="クライアント ID とクライアントシークレットをコピーする">
</Step>
</Steps>

`spaces.create` には `meetings.space.created` が必要です。`meetings.space.readonly` は Meet URL/コードをスペースに解決します。`meetings.space.settings` により、OpenClaw は API によるルーム作成時に `accessType` などの `SpaceConfig` 設定を渡せます。`meetings.conference.media.readonly` は Meet Media API の事前確認とメディア処理に使用します。実際に Media API を使用するには、Google により Developer Preview への登録が必要になる場合があります。`calendar.events.readonly` は、`--today`/`--event` によるカレンダー検索にのみ必要です。`drive.meet.readonly` は、`--include-doc-bodies` のエクスポートにのみ必要です。ブラウザベースの Chrome 参加のみが必要な場合は、OAuth を完全に省略してください。

### リフレッシュトークンを発行する

`oauth.clientId` と、必要に応じて `oauth.clientSecret` を設定（または環境変数として渡す）してから、次を実行します。

```bash
openclaw googlemeet auth login --json
```

これにより、`http://localhost:8085/oauth2callback` 上の localhost コールバックを使用する PKCE フローが実行され、リフレッシュトークンを含む `oauth` 設定ブロックが出力されます。ブラウザからローカルコールバックに到達できない場合のコピー＆ペーストフローには、`--manual` を追加します。

```bash
OPENCLAW_GOOGLE_MEET_CLIENT_ID="your-client-id" \
OPENCLAW_GOOGLE_MEET_CLIENT_SECRET="your-client-secret" \
openclaw googlemeet auth login --json --manual
```

JSON 出力:

```json
{
  "oauth": {
    "clientId": "your-client-id",
    "clientSecret": "your-client-secret",
    "refreshToken": "refresh-token",
    "accessToken": "access-token",
    "expiresAt": 1770000000000
  },
  "scope": "..."
}
```

`oauth` オブジェクトを Plugin 設定の下に保存します。

```json5
{
  plugins: {
    entries: {
      "google-meet": {
        enabled: true,
        config: {
          oauth: {
            clientId: "your-client-id",
            clientSecret: "your-client-secret",
            refreshToken: "refresh-token",
          },
        },
      },
    },
  },
}
```

リフレッシュトークンを設定に含めたくない場合は、環境変数を優先してください。最初に設定が解決され、その後にフォールバックとして環境変数が使用されます。会議作成、カレンダー検索、またはドキュメント本文エクスポートのサポートが存在する前に認証した場合は、リフレッシュトークンが現在のスコープセットを含むように `openclaw googlemeet auth login --json` を再実行してください。

### doctor で OAuth を確認する

```bash
openclaw googlemeet doctor --oauth --json
```

これにより、Chrome ランタイムを読み込んだり、接続済みの Node を必要としたりせずに、OAuth 設定が存在し、リフレッシュトークンからアクセストークンを発行できることを確認します。レポートにはステータスフィールド（`ok`、`configured`、`tokenSource`、`expiresAt`、チェックメッセージ）のみが含まれ、アクセストークン、リフレッシュトークン、クライアントシークレットは決して出力されません。

| チェック                | 意味                                                                          |
| -------------------- | -------------------------------------------------------------------------------- |
| `oauth-config`       | `oauth.clientId` と `oauth.refreshToken`、またはキャッシュ済みアクセストークンが存在する |
| `oauth-token`        | キャッシュ済みアクセストークンがまだ有効であるか、リフレッシュトークンから新しいトークンが発行された    |
| `meet-spaces-get`    | オプションの `--meeting` チェックで既存の Meet スペースを解決できた                       |
| `meet-spaces-create` | オプションの `--create-space` チェックで新しい Meet スペースを作成できた                         |

副作用を伴う作成チェックで、Meet API が有効であることと `spaces.create` スコープを実証します。

```bash
openclaw googlemeet doctor --oauth --create-space --json
```

既存スペースへの読み取りアクセスを実証します。

```bash
openclaw googlemeet doctor --oauth --meeting https://meet.google.com/abc-defg-hij --json
openclaw googlemeet resolve-space --meeting https://meet.google.com/abc-defg-hij
```

これらのチェックで `403` が発生した場合、通常は Meet REST API が無効になっている、リフレッシュトークンに必要なスコープがない、または Google アカウントがそのスペースにアクセスできないことを意味します。リフレッシュトークンエラーが発生した場合は、`openclaw googlemeet auth login --json` を再実行し、新しい `oauth` ブロックを保存してください。

ブラウザフォールバックに OAuth は必要ありません。この場合、Google 認証は OpenClaw の設定ではなく、選択した Node でログイン済みの Chrome プロファイルから取得されます。

次の環境変数がフォールバックとして使用できます。

- `OPENCLAW_GOOGLE_MEET_CLIENT_ID` または `GOOGLE_MEET_CLIENT_ID`
- `OPENCLAW_GOOGLE_MEET_CLIENT_SECRET` または `GOOGLE_MEET_CLIENT_SECRET`
- `OPENCLAW_GOOGLE_MEET_REFRESH_TOKEN` または `GOOGLE_MEET_REFRESH_TOKEN`
- `OPENCLAW_GOOGLE_MEET_ACCESS_TOKEN` または `GOOGLE_MEET_ACCESS_TOKEN`
- `OPENCLAW_GOOGLE_MEET_ACCESS_TOKEN_EXPIRES_AT` または `GOOGLE_MEET_ACCESS_TOKEN_EXPIRES_AT`
- `OPENCLAW_GOOGLE_MEET_DEFAULT_MEETING` または `GOOGLE_MEET_DEFAULT_MEETING`
- `OPENCLAW_GOOGLE_MEET_PREVIEW_ACK` または `GOOGLE_MEET_PREVIEW_ACK`

### アーティファクトの解決、事前確認、読み取り

```bash
openclaw googlemeet resolve-space --meeting https://meet.google.com/abc-defg-hij
openclaw googlemeet preflight --meeting https://meet.google.com/abc-defg-hij
```

Meet が会議レコードを作成した後は、次を実行します。

```bash
openclaw googlemeet artifacts --meeting https://meet.google.com/abc-defg-hij
openclaw googlemeet attendance --meeting https://meet.google.com/abc-defg-hij
openclaw googlemeet export --meeting https://meet.google.com/abc-defg-hij --output ./meet-export
```

`--meeting` を指定すると、`artifacts` と `attendance` はデフォルトで最新の会議レコードを使用します。保持されているすべてのレコードを対象にするには、`--all-conference-records` を渡します。

カレンダー検索では、アーティファクトを読み取る前に Google Calendar から会議 URL を解決します（Calendar イベントの読み取り専用スコープを含むリフレッシュトークンが必要です）。

```bash
openclaw googlemeet latest --today
openclaw googlemeet calendar-events --today --json
openclaw googlemeet artifacts --event "Weekly sync"
openclaw googlemeet attendance --today --format csv --output attendance.csv
```

`--today` は、今日の `primary` カレンダーから Meet リンクを含むイベントを検索します。`--event <query>` は一致するイベントテキストを検索し、`--calendar <id>` はプライマリ以外のカレンダーを対象にします。`calendar-events` は一致するイベントをプレビューし、`latest`/`artifacts`/`attendance`/`export` がどれを選択するかを示します。

会議レコード ID がすでに分かっている場合は、直接指定します。

```bash
openclaw googlemeet latest --meeting https://meet.google.com/abc-defg-hij
openclaw googlemeet artifacts --conference-record conferenceRecords/abc123 --json
openclaw googlemeet attendance --conference-record conferenceRecords/abc123 --json
```

API で作成したスペースのルームを閉じます。

```bash
openclaw googlemeet end-active-conference https://meet.google.com/abc-defg-hij
```

`spaces.endActiveConference` を呼び出します。認可済みアカウントが管理できるスペースに対して、`meetings.space.created` スコープを持つ OAuth が必要です。Meet URL、会議コード、または `spaces/{id}` を受け付け、最初に API スペースリソースへ解決します。これは `googlemeet leave` とは別です。`leave` は OpenClaw のローカルまたはセッションでの参加を停止し、`end-active-conference` はそのスペースの進行中の会議を終了するよう Google Meet に要求します。

読みやすいレポートを書き出します。

```bash
openclaw googlemeet artifacts --conference-record conferenceRecords/abc123 \
  --format markdown --output meet-artifacts.md
openclaw googlemeet attendance --conference-record conferenceRecords/abc123 \
  --format csv --output meet-attendance.csv
openclaw googlemeet export --conference-record conferenceRecords/abc123 \
  --include-doc-bodies --zip --output meet-export
openclaw googlemeet export --conference-record conferenceRecords/abc123 \
  --include-doc-bodies --dry-run
```

`artifacts` は、Google が公開している場合、会議レコードのメタデータに加え、参加者、録画、文字起こし、構造化された文字起こしエントリ、スマートノートの各リソースメタデータを返します。`--no-transcript-entries` は、大規模な会議でエントリの検索を省略します。`attendance` は、参加者を参加者セッションの行に展開し、最初と最後の確認時刻、セッション合計時間、遅刻と早退のフラグを含めます。また、ログインユーザーまたは表示名に基づいて重複する参加者リソースを統合します。`--no-merge-duplicates` は生のリソースを分離したままにし、`--late-after-minutes`/`--early-before-minutes` でしきい値を調整します。

`export` は、`summary.md`、`attendance.csv`、`transcript.md`、`artifacts.json`、`attendance.json`、`manifest.json` を含むフォルダーを書き出します。`manifest.json` には、選択された入力、エクスポートオプション、会議レコード、出力ファイル、件数、トークンソース、使用した Calendar イベント、部分取得の警告が記録されます。`--zip` は、フォルダーの隣にポータブルアーカイブも書き出します。`--include-doc-bodies` は、リンクされた文字起こしまたはスマートノートの Google Docs テキストを Drive `files.export` 経由でエクスポートします（Drive Meet 読み取り専用スコープが必要です）。これを指定しない場合、エクスポートには Meet のメタデータと構造化された文字起こしエントリのみが含まれます。アーティファクトの一部でエラー（スマートノート一覧、文字起こしエントリ、ドキュメント本文のエラー）が発生しても、エクスポート全体を失敗させるのではなく、概要またはマニフェストに警告が保持されます。`--dry-run` は同じデータを取得し、フォルダーや ZIP を作成せずにマニフェストの JSON を出力します。

エージェントは、`google_meet` ツール（`export`、`accessType` を指定した `create`、`end_active_conference`、`test_listen`）を通じて同じアクションを使用します。[ツール](#tool)を参照してください。

### ライブスモークテスト

```bash
OPENCLAW_LIVE_TEST=1 \
OPENCLAW_GOOGLE_MEET_LIVE_MEETING=https://meet.google.com/abc-defg-hij \
pnpm test:live -- extensions/google-meet/google-meet.live.test.ts
```

```bash
openclaw googlemeet setup --transport chrome-node --mode transcribe
openclaw googlemeet test-listen https://meet.google.com/abc-defg-hij --transport chrome-node --timeout-ms 30000
```

| 変数                                                                                                                  | 用途                                                                |
| ------------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------- |
| `OPENCLAW_LIVE_TEST=1`                                                                                                    | 保護されたライブテストを有効にする                                             |
| `OPENCLAW_GOOGLE_MEET_LIVE_MEETING`                                                                                       | 保持されている Meet URL、コード、または `spaces/{id}`                              |
| `OPENCLAW_GOOGLE_MEET_CLIENT_ID` / `GOOGLE_MEET_CLIENT_ID`                                                                | OAuth クライアント ID                                                        |
| `OPENCLAW_GOOGLE_MEET_REFRESH_TOKEN` / `GOOGLE_MEET_REFRESH_TOKEN`                                                        | リフレッシュトークン                                                          |
| `OPENCLAW_GOOGLE_MEET_CLIENT_SECRET`、`OPENCLAW_GOOGLE_MEET_ACCESS_TOKEN`、`OPENCLAW_GOOGLE_MEET_ACCESS_TOKEN_EXPIRES_AT` | オプション。`OPENCLAW_` プレフィックスのない同じフォールバック名も使用できる |

基本的なアーティファクトまたは出席状況のスモークテストには、`meetings.space.readonly` と `meetings.conference.media.readonly` が必要です。カレンダー検索には `calendar.events.readonly` が必要です。Drive のドキュメント本文のエクスポートには `drive.meet.readonly` が必要です。

### 作成例

```bash
openclaw googlemeet create
```

新しい会議 URI、ソース、参加セッションを出力します。OAuth がある場合は Meet API を使用し、ない場合は固定された Chrome Node のログイン済みプロファイルを使用します。ブラウザフォールバックの JSON：

```json
{
  "source": "browser",
  "meetingUri": "https://meet.google.com/abc-defg-hij",
  "joined": true,
  "browser": {
    "nodeId": "ba0f4e4bc...",
    "targetId": "tab-1"
  },
  "join": {
    "session": {
      "id": "meet_...",
      "url": "https://meet.google.com/abc-defg-hij"
    }
  }
}
```

ブラウザフォールバックで最初に Google ログインまたは Meet の権限ブロッカーに遭遇した場合、`google_meet` は単純な文字列ではなく、構造化された詳細を返します。

```json
{
  "source": "browser",
  "error": "google-login-required: OpenClaw のブラウザプロファイルで Google にログインしてから、会議の作成を再試行してください。",
  "manualActionRequired": true,
  "manualActionReason": "google-login-required",
  "manualActionMessage": "OpenClaw のブラウザプロファイルで Google にログインしてから、会議の作成を再試行してください。",
  "browser": {
    "nodeId": "ba0f4e4bc...",
    "targetId": "tab-1",
    "browserUrl": "https://accounts.google.com/signin",
    "browserTitle": "ログイン - Google アカウント"
  }
}
```

API 作成の JSON：

```json
{
  "source": "api",
  "meetingUri": "https://meet.google.com/abc-defg-hij",
  "joined": true,
  "space": {
    "name": "spaces/abc-defg-hij",
    "meetingCode": "abc-defg-hij",
    "meetingUri": "https://meet.google.com/abc-defg-hij"
  },
  "join": {
    "session": {
      "id": "meet_...",
      "url": "https://meet.google.com/abc-defg-hij"
    }
  }
}
```

作成時はデフォルトで参加しますが、Chrome または Chrome Node からブラウザ経由で参加する場合は、引き続きログイン済みの Google プロファイルが必要です。ログアウトしている場合、OpenClaw は `manualActionRequired: true` またはブラウザフォールバックエラーを報告し、再試行する前に Google ログインを完了するようオペレーターに求めます。

Cloud プロジェクト、OAuth プリンシパル、会議参加者が Meet メディア API の Google Workspace Developer Preview Program に登録されていることを確認した後にのみ、`preview.enrollmentAcknowledged: true` を設定してください。

## 設定

共通の Chrome エージェントパスに必要なのは、有効化された Plugin、BlackHole、SoX、リアルタイムプロバイダーキー、設定済みの OpenClaw TTS プロバイダーだけです。

```json5
{
  plugins: {
    entries: {
      "google-meet": {
        enabled: true,
        config: {},
      },
    },
  },
}
```

### デフォルト

| キー                               | デフォルト                                  | 注記                                                                                                                                                                                                             |
| --------------------------------- | ---------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `defaultTransport`                | `"chrome"`                               |                                                                                                                                                                                                                   |
| `defaultMode`                     | `"agent"`                                | `"realtime"` は `"agent"` のレガシーエイリアスとして受け入れられます。新しい呼び出し元では `"agent"` を使用してください                                                                                                                        |
| `chromeNode.node`                 | 未設定                                    | `chrome-node` の Node ID/名前/IP。対応可能な Node が複数接続される可能性がある場合は必須です                                                                                                                      |
| `chrome.launch`                   | `true`                                   | 参加時に Chrome を起動します。すでに開いているセッションを再利用する場合のみ `false` を設定してください                                                                                                                                 |
| `chrome.audioBackend`             | `"blackhole-2ch"`                        |                                                                                                                                                                                                                   |
| `chrome.guestName`                | `"OpenClaw Agent"`                       | サインアウト状態の Meet ゲスト画面に表示されます                                                                                                                                                                         |
| `chrome.autoJoin`                 | `true`                                   | `chrome-node` でゲスト名の入力と Join Now のクリックをベストエフォートで行います                                                                                                                                                   |
| `chrome.reuseExistingTab`         | `true`                                   | 重複したタブを開く代わりに、既存の Meet タブをアクティブ化します                                                                                                                                                      |
| `chrome.waitForInCallMs`          | `20000`                                  | トークバックの導入が開始される前に、Meet タブが通話中と報告するまで待機します                                                                                                                                          |
| `chrome.audioFormat`              | `"pcm16-24khz"`                          | コマンドペアの音声形式。`"g711-ulaw-8khz"` は、電話音声を出力するレガシーまたはカスタムのコマンドペア専用です                                                                                                   |
| `chrome.audioBufferBytes`         | `4096`                                   | 生成されたコマンドペア音声コマンド用の SoX 処理バッファ（SoX のデフォルトである 8192 バイトの半分で、パイプのレイテンシーを低減）。値は最小 17 バイトに制限されます                                         |
| `chrome.audioInputCommand`        | 生成された SoX コマンド                    | CoreAudio `BlackHole 2ch` から読み取り、`chrome.audioFormat` 形式で音声を書き込みます                                                                                                                                        |
| `chrome.audioOutputCommand`       | 生成された SoX コマンド                    | `chrome.audioFormat` 形式で音声を読み取り、CoreAudio `BlackHole 2ch` に書き込みます                                                                                                                                          |
| `chrome.bargeInInputCommand`      | 未設定                                    | アシスタントの再生中に人間による割り込みを検出するため、符号付き 16 ビットのリトルエンディアン・モノラル PCM を書き込むオプションのローカルマイクコマンド。Gateway でホストされるコマンドペアブリッジに適用されます                          |
| `chrome.bargeInRmsThreshold`      | `650`                                    | 人間による割り込みと見なされる RMS レベル                                                                                                                                                                           |
| `chrome.bargeInPeakThreshold`     | `2500`                                   | 人間による割り込みと見なされるピークレベル                                                                                                                                                                          |
| `chrome.bargeInCooldownMs`        | `900`                                    | 繰り返される割り込み解除の最小間隔                                                                                                                                                                |
| `mode`（リクエストごと）              | `"agent"`                                | トークバックモード。[エージェントモードと双方向モード](#agent-and-bidi-modes)の表を参照してください                                                                                                                                       |
| `realtime.provider`               | `"openai"`                               | 以下のスコープ付きフィールドが未設定の場合に使用される互換性フォールバック                                                                                                                                                |
| `realtime.transcriptionProvider`  | `"openai"`                               | リアルタイム文字起こしの `agent` モードで使用されるプロバイダー ID                                                                                                                                                       |
| `realtime.voiceProvider`          | 未設定                                    | 直接リアルタイム音声の `bidi` モードで使用されるプロバイダー ID。エージェントモードの文字起こしを OpenAI のまま維持しながら Gemini Live を使用するには、`"google"` に設定します。特定の Gemini Live モデルを選択するには `realtime.model` と組み合わせます。 |
| `realtime.toolPolicy`             | `"safe-read-only"`                       | [エージェントモードと双方向モード](#agent-and-bidi-modes)を参照してください                                                                                                                                                                 |
| `realtime.instructions`           | 簡潔な音声応答の指示          | 簡潔に話し、より詳しい回答には `openclaw_agent_consult` を使用するようモデルに指示します                                                                                                                              |
| `realtime.introMessage`           | `"Say exactly: I'm here and listening."` | リアルタイムブリッジの接続時に一度だけ読み上げられます。無音で参加するには `""` に設定します                                                                                                                                       |
| `realtime.agentId`                | `"main"`                                 | `openclaw_agent_consult` に使用される OpenClaw エージェント ID                                                                                                                                                               |
| `voiceCall.enabled`               | `true`                                   | Twilio PSTN 通話、DTMF、導入時の挨拶を Voice Call Plugin に委任します                                                                                                                                 |
| `voiceCall.dtmfDelayMs`           | `12000`                                  | Twilio 経由で PIN から生成された DTMF シーケンスを再生する前の待機時間                                                                                                                                               |
| `voiceCall.postDtmfSpeechDelayMs` | `5000`                                   | Voice Call が Twilio 側の通話を開始してから、リアルタイムの導入時の挨拶を要求するまでの遅延                                                                                                                        |

`chrome.audioBridgeCommand` と `chrome.audioBridgeHealthCommand` を使用すると、`chrome.audioInputCommand`/`chrome.audioOutputCommand` の代わりに外部ブリッジがローカル音声パス全体を管理できます。これらを使用できるモードの制約については、[注記](#notes)を参照してください。

レガシーの `realtime.provider: "google"` 形式には `openclaw doctor --fix` 移行が用意されています。これらのフィールドがまだ設定されていない場合、その意図を `realtime.voiceProvider: "google"` と `realtime.transcriptionProvider: "openai"` に移します。

### オプションのオーバーライド

```json5
{
  defaults: {
    meeting: "https://meet.google.com/abc-defg-hij",
  },
  browser: {
    defaultProfile: "openclaw",
  },
  chrome: {
    guestName: "OpenClaw Agent",
    waitForInCallMs: 30000,
    bargeInInputCommand: [
      "sox",
      "-q",
      "-t",
      "coreaudio",
      "External Microphone",
      "-r",
      "24000",
      "-c",
      "1",
      "-b",
      "16",
      "-e",
      "signed-integer",
      "-t",
      "raw",
      "-",
    ],
  },
  chromeNode: {
    node: "parallels-macos",
  },
  defaultMode: "agent",
  realtime: {
    provider: "openai",
    transcriptionProvider: "openai",
    voiceProvider: "google",
    model: "gemini-3.1-flash-live-preview",
    agentId: "jay",
    toolPolicy: "owner",
    introMessage: "正確に次のように言ってください: I'm here.",
    providers: {
      google: {
        speakerVoice: "Kore",
      },
    },
  },
}
```

エージェントモードの聞き取りと発話の両方に ElevenLabs を使用する場合:

```json5
{
  tts: {
    provider: "elevenlabs",
    providers: {
      elevenlabs: {
        modelId: "eleven_v3",
        speakerVoiceId: "pMsXgVXv3BLzUgSXRplE",
      },
    },
  },
  plugins: {
    entries: {
      "google-meet": {
        config: {
          realtime: {
            transcriptionProvider: "elevenlabs",
            providers: {
              elevenlabs: {
                modelId: "scribe_v2_realtime",
                audioFormat: "ulaw_8000",
                sampleRate: 8000,
                commitStrategy: "vad",
              },
            },
          },
        },
      },
    },
  },
}
```

Meet で常用される音声は `tts.providers.elevenlabs.speakerVoiceId` から取得されます。TTS モデルのオーバーライドが有効な場合、エージェントの応答では応答ごとの `[[tts:speakerVoiceId=... model=eleven_v3]]` ディレクティブも使用できますが、ミーティングでは設定が決定的なデフォルトです。参加時にはログに `transcriptionProvider=elevenlabs` が表示され、音声応答ごとに `provider=elevenlabs model=eleven_v3 speakerVoiceId=<voiceId>` が記録されます。

Twilio 専用設定:

```json5
{
  defaultTransport: "twilio",
  twilio: {
    defaultDialInNumber: "+15551234567",
    defaultPin: "123456",
  },
  voiceCall: {
    gatewayUrl: "ws://127.0.0.1:18789",
  },
}
```

`voiceCall.enabled: true`（デフォルト）と Twilio トランスポートを使用すると、Voice Call はリアルタイムメディアストリームを開く前に DTMF シーケンスを送信し、その後、保存された導入テキストを最初のリアルタイム挨拶として使用します。`voice-call` が有効でない場合でも、Google Meet はダイヤルプランを検証して記録できますが、Twilio 通話を発信することはできません。

`voiceCall.gatewayUrl` を未設定のままにすると、ローカルの信頼済み Gateway ランタイムが使用され、呼び出し元のエージェントが呼び出し全体を通して維持されます。設定済みの Gateway URL は引き続き明示的な WebSocket ターゲットであり、Plugin の生成元を認証できません。デフォルト以外のエージェントによる参加は、別のエージェントを暗黙に使用するのではなく、安全側に倒して失敗します。エージェントごとのルーティングが必要な場合は、Google Meet と Voice Call を同じ Gateway プロセスで実行してください。

## ツール

エージェントは `google_meet` ツールを使用します。

```json
{
  "action": "join",
  "url": "https://meet.google.com/abc-defg-hij",
  "transport": "chrome-node",
  "mode": "agent"
}
```

| `action`                | 目的                                                                                              |
| ----------------------- | ------------------------------------------------------------------------------------------------- |
| `join`                  | 明示的な Meet URL に参加する                                                                       |
| `create`                | スペースを作成する（デフォルトでは参加も行う）。`accessType`/`entryPointAccess` をサポートする |
| `status`                | アクティブなセッションを一覧表示するか、`sessionId` で指定したセッションを調査する              |
| `setup_status`          | `googlemeet setup` と同じチェックを実行する                                                        |
| `resolve_space`         | `spaces.get` を介して URL、コード、または `spaces/{id}` を解決する                         |
| `preflight`             | OAuth と会議解決の前提条件を検証する                                                               |
| `latest`                | 会議の最新のカンファレンスレコードを検索する                                                       |
| `calendar_events`       | Meet リンクを含む Calendar イベントをプレビューする                                               |
| `artifacts`             | カンファレンスレコードと、参加者、録画、文字起こし、スマートノートのメタデータを一覧表示する          |
| `attendance`            | 参加者と参加者セッションを一覧表示する                                                             |
| `export`                | アーティファクト、出席情報、文字起こし、マニフェストのバンドルを書き込む。マニフェストのみの場合は `"dryRun": true` を設定する |
| `recover_current_tab`   | 新しいタブを開かずに、既存の Meet タブにフォーカスするか調査する                                   |
| `transcript`            | 上限付きの字幕文字起こしを読み取る。`sinceIndex` は直前の `nextIndex` から再開する         |
| `leave`                 | セッションを終了する（Chrome は「Leave」ボタンをクリックし、自身が開いたタブのみを閉じる。Twilio は通話を切断する） |
| `end_active_conference` | API で管理されるスペースのアクティブな Google Meet カンファレンスを終了する                         |
| `speak`                 | `sessionId` と `message` を指定して、リアルタイムエージェントに直ちに発話させる           |
| `test_speech`           | セッションを作成または再利用し、既知のフレーズをトリガーして、Chrome の稼働状態を返す                |
| `test_listen`           | 観察専用セッションを作成または再利用し、字幕または文字起こしの進行を待つ                            |

`test_speech` は常に `mode: "agent"` または `"bidi"` を強制し、`mode: "transcribe"` での実行を要求された場合は失敗します。観察専用セッションは音声を出力できないためです。`speechOutputVerified` では、最新のリアルタイム出力バイトに加えて、その出力中にブリッジのマイクキャプチャパスへ戻ってくる、無音ではない最新の音声も必要です。再利用されたセッションの古い出力やループバック信号は対象にならず、シンクバイトの増加だけでは、検証済みの発話として報告されなくなりました。

Chrome トランスポートでは、`leave` は Meet の Leave call ボタンをクリックした後も、再利用されたユーザー所有のタブを開いたままにします。OpenClaw が開いたタブは退出後に閉じられます。

Chrome が Gateway ホスト上で動作する場合は `transport: "chrome"` を、ペアリング済み Node 上で動作する場合は `transport: "chrome-node"` を使用します。どちらの場合も、モデルプロバイダーと `openclaw_agent_consult` は Gateway ホスト上で実行されるため、モデル認証情報はそこに保持されます。エージェントモードのログには、ブリッジ起動時に解決された文字起こしプロバイダーとモデルが記録され、合成された各応答の後には TTS プロバイダー、モデル、音声、出力形式、サンプルレートが記録されます。未加工の `mode: "realtime"` は、`mode: "agent"` のレガシー互換エイリアスとして引き続き受け付けられますが、ツールの `mode` 列挙型には表示されなくなりました。

API ベースのルームと明示的なアクセスポリシーを使用する `create`:

```json
{
  "action": "create",
  "transport": "chrome-node",
  "mode": "agent",
  "accessType": "OPEN"
}
```

既知のルームでアクティブなカンファレンスを終了する場合:

```json
{
  "action": "end_active_conference",
  "meeting": "https://meet.google.com/abc-defg-hij"
}
```

会議が有用だと判断する前に行う、リスニング優先の検証:

```json
{
  "action": "test_listen",
  "url": "https://meet.google.com/abc-defg-hij",
  "transport": "chrome-node",
  "timeoutMs": 30000
}
```

オンデマンドで発話する場合:

```json
{
  "action": "speak",
  "sessionId": "meet_...",
  "message": "次のとおり正確に発話してください: 参加して聞いています。"
}
```

`status` には、利用可能な場合に Chrome の稼働状態が含まれます。

| フィールド                                                                 | 意味                                                                                                                   |
| --------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------- |
| `inCall`                                                              | Chrome が Meet 通話内にいるように見える                                                                                |
| `micMuted`                                                            | ベストエフォートで判定した Meet のマイク状態                                                                            |
| `manualActionRequired` / `manualActionReason` / `manualActionMessage` | 発話を機能させるには、ブラウザプロファイルでの手動ログイン、Meet ホストによる参加承認、権限の付与、またはブラウザ制御の修復が必要 |
| `speechReady` / `speechBlockedReason` / `speechBlockedMessage`        | 管理対象 Chrome で現在発話が許可されているかどうか。`speechReady: false` は、OpenClaw が導入フレーズまたはテストフレーズを送信しなかったことを意味する |
| `providerConnected` / `realtimeReady`                                 | リアルタイム音声ブリッジの状態                                                                                         |
| `lastInputAt` / `lastOutputAt`                                        | ブリッジから最後に受信した音声、またはブリッジへ最後に送信した音声                                                     |
| `audioOutputRouted` / `audioOutputDeviceLabel`                        | Meet タブのメディア出力がブリッジの BlackHole デバイスへアクティブにルーティングされたかどうか                          |
| `lastOutputLoopbackAt` / `outputLoopbackSignalBytes`                  | BlackHole のマイクキャプチャパス上で波形フィンガープリントが相関付けられた最新の出力                                    |
| `lastOutputLoopbackCorrelation`                                       | キャプチャされた信号と現在のアシスタント出力生成を結び付ける相関スコア                                                 |
| `outputGeneration` / `verifiedOutputGeneration`                       | 単調増加する ID。両者が等しい場合、古い発話ではなく現在の出力がループバック検証を通過したことを意味する                  |
| `lastOutputLoopbackRms` / `lastOutputLoopbackPeak`                    | 最新の検証済みループバックキャプチャチャンクの音声エネルギー診断                                                       |
| `lastSuppressedInputAt` / `suppressedInputBytes`                      | アシスタントの再生中に無視されたループバック入力                                                                       |

## エージェントモードと双方向モード

| モード    | 応答を決定する主体              | 音声出力パス                           | 使用する場面                                             |
| ------- | ----------------------------- | -------------------------------------- | ----------------------------------------------------- |
| `agent` | 設定された OpenClaw エージェント | 通常の OpenClaw TTS ランタイム          | 「自分のエージェントが会議に参加している」動作が必要な場合 |
| `bidi`  | リアルタイム音声モデル            | リアルタイム音声プロバイダーの音声応答   | 最低レイテンシーの対話型音声ループが必要な場合             |

`agent` モード: リアルタイム文字起こしプロバイダーが会議音声を認識し、参加者の確定済み文字起こしが設定済みの OpenClaw エージェントを経由し、応答は通常の OpenClaw TTS によって発話されます。1 回の発話ターンから複数の古い部分応答が生成されないように、時間的に近い確定済み文字起こし断片は問い合わせ前にまとめられます。キュー内のアシスタント音声がまだ再生されている間はリアルタイム入力が抑制され、BlackHole ループバックによってエージェントが自身の発話に応答しないように、直近のアシスタントに似た文字起こしエコーは問い合わせ前に無視されます。

`bidi` モード: リアルタイム音声モデルが直接応答し、より深い推論、最新情報、または通常の OpenClaw ツールが必要な場合は `openclaw_agent_consult` を呼び出せます。問い合わせツールは、直近の会議文字起こしコンテキストを使用して通常の OpenClaw エージェントをバックグラウンドで実行し、簡潔な音声応答を返します。`agent` モードでは OpenClaw がその応答を TTS に直接送信し、`bidi` モードではリアルタイム音声モデルがそれを発話できます。Voice Call と同じ共有問い合わせ機構を使用します。

デフォルトでは、問い合わせは `main` エージェントに対して実行されます。Meet レーンを専用のエージェントワークスペース、モデルのデフォルト設定、ツールポリシー、メモリ、セッション履歴に関連付けるには、`realtime.agentId` を設定します。エージェントモードの問い合わせは、会議ごとの `agent:<id>:subagent:google-meet:<session>` セッションキーを使用するため、フォローアップの質問でも通常のエージェントポリシーを継承しながら会議コンテキストが維持されます。エージェントがエージェントモードで `google_meet` を呼び出すと、参加者の発話に応答する前に、コンサルタントセッションが呼び出し元の現在のトランスクリプトからフォークされます。Meet セッションは分離されたままなので、会議でのフォローアップによって呼び出し元のトランスクリプトが直接変更されることはありません。

`realtime.toolPolicy` は問い合わせの実行を制御します。

| ポリシー           | 動作                                                                                                                               |
| ---------------- | -------------------------------------------------------------------------------------------------------------------------------- |
| `safe-read-only` | 問い合わせツールを公開する。通常のエージェントを `read`、`web_search`、`web_fetch`、`x_search`、`memory_search`、`memory_get` に制限する |
| `owner`          | 問い合わせツールを公開する。通常のエージェントに通常のツールポリシーの使用を許可する                                               |
| `none`           | リアルタイム音声モデルに問い合わせツールを公開しない                                                                               |

問い合わせセッションキーは Meet セッションごとにスコープ設定されるため、同じ会議中の後続の問い合わせ呼び出しでは以前の問い合わせコンテキストが再利用されます。

Chrome が完全に参加した後、音声による準備完了チェックを強制する場合:

```bash
openclaw googlemeet speak meet_... "Say exactly: I'm here and listening."
```

参加から発話までを確認する完全なスモークテスト:

```bash
openclaw googlemeet test-speech https://meet.google.com/abc-defg-hij \
  --transport chrome-node \
  --message "Say exactly: I'm here and listening."
```

## ライブテストのチェックリスト

無人エージェントに会議を任せる前に、次を実行します。

```bash
openclaw googlemeet setup
openclaw nodes status
openclaw googlemeet test-speech https://meet.google.com/abc-defg-hij \
  --transport chrome-node \
  --message "Say exactly: Google Meet speech test complete."
```

想定される Chrome-node の状態:

- `googlemeet setup` はすべて正常で、Chrome-node がデフォルトのトランスポートである場合、または Node が固定されている場合は `chrome-node-connected` が含まれます。
- `nodes status` には、選択した Node が接続済みとして表示され、`googlemeet.chrome` と `browser.proxy` の両方がアドバタイズされます。
- Meet タブが参加し、`test-speech` は `inCall: true` を含む Chrome のヘルス情報を返します。

Parallels macOS VM などのリモート Chrome ホストでは、Gateway または VM の更新後に行う最短の安全な確認手順は次のとおりです。

```bash
openclaw googlemeet setup
openclaw nodes status --connected
openclaw nodes invoke \
  --node parallels-macos \
  --command googlemeet.chrome \
  --params '{"action":"setup"}'
```

これにより、Gateway Plugin が読み込まれていること、VM Node が現在のトークンで接続されていること、エージェントが実際の会議タブを開く前に Meet オーディオブリッジが利用可能であることを確認できます。

Twilio のスモークテストには、電話によるダイヤルイン情報が提示される会議を使用します。

```bash
openclaw googlemeet setup
openclaw googlemeet join https://meet.google.com/abc-defg-hij \
  --transport twilio \
  --dial-in-number +15551234567 \
  --pin 123456
```

期待される Twilio の状態：

- `googlemeet setup` には、正常な `twilio-voice-call-plugin`、`twilio-voice-call-credentials`、`twilio-voice-call-webhook` のチェックが含まれます。
- Gateway の再読み込み後、CLI で `voicecall` が利用できます。
- 返されたセッションには `transport: "twilio"` と `twilio.voiceCallId` があります。
- `openclaw logs --follow` には、リアルタイム TwiML より先に DTMF TwiML が配信され、その後、最初の挨拶がキューに追加されたリアルタイムブリッジが表示されます。
- `googlemeet leave <sessionId>` は委任された音声通話を切断します。

## トラブルシューティング

### エージェントに Google Meet ツールが表示されない

Plugin が有効になっていることを確認し、Gateway を再読み込みします。実行中のエージェントに表示されるのは、現在の Gateway プロセスによって登録された Plugin ツールのみです。

```bash
openclaw plugins list | grep google-meet
openclaw googlemeet setup
```

macOS 以外の Gateway ホストでも `google_meet` は表示されますが、ローカル Chrome のトークバックアクションは、オーディオブリッジに到達する前にブロックされます。デフォルトのローカル Chrome エージェントパスではなく、`mode: "transcribe"`、Twilio ダイヤルイン、または macOS の `chrome-node` ホストを使用してください。

### Google Meet 対応の接続済み Node がない

Node ホストで：

```bash
openclaw plugins install npm:@openclaw/google-meet
openclaw plugins enable browser
OPENCLAW_ALLOW_INSECURE_PRIVATE_WS=1 \
  openclaw node run --host <gateway-lan-ip> --port 18789 --display-name parallels-macos
```

Gateway ホストで：

```bash
openclaw devices list
openclaw devices approve <requestId>
openclaw nodes status
```

Node は接続済みで、`googlemeet.chrome` と `browser.proxy` が一覧に含まれている必要があります。また、Gateway 設定で両方を許可する必要があります。

```json5
{
  gateway: {
    nodes: {
      commands: { allow: ["browser.proxy", "googlemeet.chrome"] },
    },
  },
}
```

`googlemeet setup` が `chrome-node-connected` で失敗する場合、または Gateway ログに `gateway token mismatch` が記録される場合は、現在の Gateway トークンを使用して Node を再インストールまたは再起動します。

```bash
OPENCLAW_ALLOW_INSECURE_PRIVATE_WS=1 \
  openclaw node install \
  --host <gateway-lan-ip> \
  --port 18789 \
  --display-name parallels-macos \
  --force
```

その後、Node サービスを再読み込みし、次を再実行します。

```bash
openclaw googlemeet setup
openclaw nodes status --connected
```

### ブラウザーは開くがエージェントが参加できない

観察専用の参加には `googlemeet test-listen`、リアルタイム参加には `googlemeet test-speech` を実行し、返された Chrome のヘルス情報を確認します。いずれかが `manualActionRequired: true` を報告した場合は、オペレーターに `manualActionMessage` を表示し、ブラウザー操作が完了するまで再試行を停止します。

一般的な手動操作：Chrome プロファイルにログインする、Meet ホストアカウントからゲストの参加を承認する、ネイティブプロンプトが表示されたら Chrome にマイクとカメラの権限を付与する、停止した Meet の権限ダイアログを閉じるか修復する。

Meet に「Do you want people to hear you in the meeting?」と表示されただけで「ログインしていない」と報告しないでください。これは Meet の音声選択インタースティシャルです。利用可能な場合、OpenClaw はブラウザー自動化を通じて **Use microphone** をクリックし、実際の会議状態になるまで待機を続けます。作成専用のブラウザーフォールバックでは、URL の生成にリアルタイム音声パスは不要なため、代わりに **Continue without microphone** をクリックすることがあります。

### 会議の作成に失敗する

OAuth が設定されている場合、`googlemeet create` は Meet API の `spaces.create` を使用し、それ以外の場合は固定された Chrome Node のブラウザーを使用します。次を確認してください。

- **API による作成**：`oauth.clientId` と `oauth.refreshToken`（または対応する `OPENCLAW_GOOGLE_MEET_*` 環境変数）が存在し、作成サポートの追加後にリフレッシュトークンが発行されていること。古いトークンには `meetings.space.created` が含まれていない可能性があるため、`openclaw googlemeet auth login --json` を再実行してください。
- **ブラウザーフォールバック**：`defaultTransport: "chrome-node"` と `chromeNode.node` が、`browser.proxy` と `googlemeet.chrome` を備えた接続済み Node を指していること。その Node の OpenClaw Chrome プロファイルがログイン済みで、`https://meet.google.com/new` を開けること。
- **ブラウザーフォールバックの再試行**：新しいタブを開く前に、既存の `.../new` または Google アカウントのプロンプトタブを再利用します。別のタブを手動で開くのではなく、ツール呼び出しを再試行してください。
- **手動操作**：ツールが `manualActionRequired: true` を返した場合は、`browser.nodeId`、`browser.targetId`、`browserUrl`、`manualActionMessage` を使用してオペレーターを案内してください。ループで再試行しないでください。
- **音声選択インタースティシャル**：Meet に「Do you want people to hear you in the meeting?」と表示された場合は、タブを開いたままにしてください。OpenClaw は **Use microphone** または（作成専用では）**Continue without microphone** をクリックし、生成された URL を待ち続ける必要があります。実行できない場合、エラーには `google-login-required` ではなく `meet-audio-choice-required` が記載される必要があります。

### エージェントは参加するが発話しない

```bash
openclaw googlemeet setup
openclaw googlemeet doctor
```

STT -> OpenClaw エージェント -> TTS パスには `mode: "agent"`、直接のリアルタイム音声フォールバックには `mode: "bidi"` を使用します。`mode: "transcribe"` は意図的にトークバックブリッジを開始しません。観察専用のデバッグでは、参加者が発話した後に `openclaw googlemeet status --json <session-id>` を実行し、`captioning`、`transcriptLines`、`lastCaptionText` を確認します。`inCall` が true でも `transcriptLines` が `0` のままである場合は、Meet の字幕が無効になっている、オブザーバーのインストール後に誰も発話していない、Meet の UI が変更された、または会議の言語やアカウントでライブ字幕を利用できない可能性があります。

`googlemeet test-speech` は常にリアルタイムパスを確認し、その呼び出しでブリッジの出力バイトが観測されたかどうかを報告します。`speechOutputVerified` が false で `speechOutputTimedOut` が true の場合、リアルタイムプロバイダーは発話を受け入れたものの、OpenClaw が Chrome オーディオブリッジに到達する新しい出力バイトを確認できなかった可能性があります。

次の点も確認してください。リアルタイムプロバイダーキー（`OPENAI_API_KEY` または `GEMINI_API_KEY`）が Gateway ホストで利用可能であること、Chrome ホストに `BlackHole 2ch` が表示されること、そこに `sox` が存在すること、Meet のマイクとスピーカーが仮想オーディオパスを経由してルーティングされていること（ローカル Chrome のリアルタイム参加では、`doctor` に `meet output routed: yes` が表示される必要があります）。

`googlemeet doctor [session-id]` は、セッション、Node、通話中の状態、手動操作の理由、リアルタイムプロバイダー接続、`realtimeReady`、音声入出力アクティビティ、最終音声タイムスタンプ、バイトカウンター、ブラウザー URL を出力します。生の JSON には `googlemeet status [session-id] --json`、トークンを公開せずに OAuth の更新を検証するには `googlemeet doctor --oauth`（`--meeting` または `--create-space` を追加）を使用します。

エージェントがタイムアウトし、Meet タブがすでに開いている場合は、新しいタブを開かずに確認します。

```bash
openclaw googlemeet recover-tab
openclaw googlemeet recover-tab https://meet.google.com/abc-defg-hij
```

同等のツールアクションは `recover_current_tab` です。新しいタブやセッションを開かず、選択したトランスポート（`chrome` ではローカルブラウザー制御、`chrome-node` では設定済み Node）にある既存の Meet タブにフォーカスして調査し、現在の障害要因（ログイン、参加承認、権限、音声選択の状態）を報告します。CLI コマンドは設定済みの Gateway と通信するため、Gateway が実行中である必要があります。`chrome-node` では Node も接続済みである必要があります。

### Twilio のセットアップチェックに失敗する

`voice-call` が許可または有効化されていない場合、`twilio-voice-call-plugin` は失敗します。`plugins.allow` に追加し、`plugins.entries.voice-call` を有効にして、Gateway を再読み込みしてください。

Twilio バックエンドにアカウント SID、認証トークン、発信者番号がない場合、`twilio-voice-call-credentials` は失敗します。

```bash
export TWILIO_ACCOUNT_SID=AC...
export TWILIO_AUTH_TOKEN=...
export TWILIO_FROM_NUMBER=+15550001234
```

`voice-call` に公開 Webhook がない場合、または `publicUrl` がループバックやプライベートネットワーク空間を指している場合、`twilio-voice-call-webhook` は失敗します。`localhost`、`127.0.0.1`、`0.0.0.0`、`10.x`、`172.16.x`-`172.31.x`、`192.168.x`、`169.254.x`、`fc00::/7`、`fd00::/8` を `publicUrl` として使用しないでください。通信事業者のコールバックはこれらに到達できません。`plugins.entries.voice-call.config.publicUrl` を公開 URL に設定するか、トンネルまたは Tailscale による公開を設定してください。

```json5
{
  plugins: {
    entries: {
      "voice-call": {
        enabled: true,
        config: {
          provider: "twilio",
          fromNumber: "+15550001234",
          publicUrl: "https://voice.example.com/voice/webhook",
        },
      },
    },
  },
}
```

ローカル開発では、プライベートホスト URL の代わりに、トンネルまたは Tailscale による公開を使用します。

```json5
{
  plugins: {
    entries: {
      "voice-call": {
        config: {
          tunnel: { provider: "ngrok" },
          // または
          tailscale: { mode: "funnel", path: "/voice/webhook" },
        },
      },
    },
  },
}
```

Gateway を再起動または再読み込みしてから、次を実行します。

```bash
openclaw googlemeet setup --transport twilio
openclaw voicecall setup
openclaw voicecall smoke
```

`voicecall smoke` はデフォルトでは準備状況の確認のみを行います。特定の番号でドライランを実行するには：

```bash
openclaw voicecall smoke --to "+15555550123"
```

実際の発信通話を意図的に行う場合にのみ、`--yes` を追加します。

```bash
openclaw voicecall smoke --to "+15555550123" --yes
```

### Twilio 通話は開始するが会議に参加しない

Meet イベントに電話によるダイヤルイン情報が提示されていることを確認し、正確なダイヤルイン番号と PIN、またはカスタム DTMF シーケンスを渡します。

```bash
openclaw googlemeet join https://meet.google.com/abc-defg-hij \
  --transport twilio \
  --dial-in-number +15551234567 \
  --dtmf-sequence ww123456#
```

PIN の前に一時停止を入れるには、`--dtmf-sequence` の先頭に `w` またはカンマを使用します。

通話が作成されても、Meet の参加者一覧にダイヤルイン参加者が表示されない場合：

- `openclaw googlemeet doctor <session-id>`：委任された Twilio 通話 ID、DTMF がキューに追加されたかどうか、導入の挨拶が要求されたかどうかを確認します。
- `openclaw voicecall status --call-id <id>`：通話がまだアクティブであることを確認します。
- `openclaw voicecall tail`：Twilio の Webhook が Gateway に到達していることを確認します。
- `openclaw logs --follow`：Twilio Meet シーケンスを確認します。Google Meet が参加を委任し、Voice Call が接続前の DTMF TwiML を保存して配信し、Voice Call が Twilio 通話用のリアルタイム TwiML を配信した後、Google Meet が `voicecall.speak` を使用して導入の発話を要求します。
- `openclaw googlemeet setup --transport twilio` を再実行します。正常なセットアップチェックは必須ですが、会議の PIN シーケンスが正しいことまでは証明しません。
- ダイヤルイン番号が、PIN と同じ Meet の招待およびリージョンに属していることを確認します。
- Meet の応答が遅い場合、または接続前の DTMF 送信後も通話トランスクリプトに PIN プロンプトが表示されている場合は、`voiceCall.dtmfDelayMs` をデフォルトの 12 秒から増やします。
- 参加者が参加しても挨拶が聞こえない場合は、DTMF 後の `voicecall.speak` リクエスト、およびメディアストリームの TTS 再生または Twilio の `<Say>` フォールバックについて `openclaw logs --follow` を確認します。トランスクリプトに「enter the meeting PIN」と表示されたままの場合、電話回線側はまだ Meet ルームに参加していないため、参加者には音声が聞こえません。

Webhook が届かない場合は、まず Voice Call plugin をデバッグしてください。プロバイダーが `plugins.entries.voice-call.config.publicUrl` または設定済みのトンネルに到達できる必要があります。[音声通話のトラブルシューティング](/ja-JP/plugins/voice-call#troubleshooting)を参照してください。

## 注意事項

Google Meet の公式メディア API は受信向けであるため、通話内で発話するには引き続き参加者経由の経路が必要です。この Plugin では、その境界を明示しています。Chrome はブラウザでの参加とローカル音声ルーティングを処理し、Twilio は電話によるダイヤルイン参加を処理します。

Chrome のトークバックモードには、`BlackHole 2ch` に加えて次のいずれかが必要です。

- `chrome.audioInputCommand` と `chrome.audioOutputCommand`：OpenClaw がブリッジを管理し、それらのコマンドと選択したプロバイダーの間で `chrome.audioFormat` の音声をパイプします。`agent` モードではリアルタイム文字起こしと通常の TTS を使用し、`bidi` モードではリアルタイム音声プロバイダーを使用します。デフォルトの経路は、`chrome.audioBufferBytes: 4096` を使用する 24 kHz PCM16 です。従来のコマンドペア向けには、8 kHz G.711 mu-law も引き続き利用できます。
- `chrome.audioBridgeCommand`：外部ブリッジコマンドがローカル音声経路全体を管理し、デーモンを起動または検証した後に終了する必要があります。`bidi` でのみ有効です。`agent` モードでは、TTS のためにコマンドペアへ直接アクセスする必要があるためです。

コマンドペア方式の Chrome ブリッジでは、`chrome.bargeInInputCommand` が別のローカルマイクを監視し、人が話し始めたときにアシスタントの再生をクリアできます。これにより、アシスタントの再生中に共有 BlackHole loopback 入力が一時的に抑制されている場合でも、人間の発話がアシスタントの出力より優先されます。`chrome.audioInputCommand`/`chrome.audioOutputCommand` と同様、これはオペレーターが設定するローカルコマンドです。明示的な信頼済みコマンドパスまたは引数リストを使用し、信頼できない場所にあるスクリプトは決して使用しないでください。

明瞭な双方向音声を実現するには、Meet の出力と Meet のマイクを別々の仮想デバイス、または Loopback 形式の仮想デバイスグラフを介してルーティングしてください。単一の共有 BlackHole デバイスでは、他の参加者の音声が通話へエコーバックされる可能性があります。

`googlemeet speak` は Chrome セッションのアクティブなトークバック音声ブリッジを開始し、`googlemeet leave` は停止します（Voice Call を介して委任された Twilio セッションの場合は、基盤となる通話も切断します）。API で管理されるスペースのアクティブな Google Meet 会議も終了するには、`googlemeet end-active-conference` を使用してください。

## 関連項目

- [会議 Plugin の概要](/ja-JP/plugins/meeting-plugins)
- [Voice Call plugin](/ja-JP/plugins/voice-call)
- [トークモード](/ja-JP/nodes/talk)
- [Plugin の構築](/ja-JP/plugins/building-plugins)
