---
read_when:
    - OpenClaw エージェントをビデオ会議に参加させたい場合
    - Google Meet、Microsoft Teams 会議、Zoom 会議の各 Plugin から選択しています
    - 共有 Chrome、BlackHole、SoX、またはミーティングモードのセットアップが必要です
summary: Google Meet、Microsoft Teams、Zoom のいずれかを選択し、会議への参加を設定する
title: ミーティング用プラグイン
x-i18n:
    generated_at: "2026-07-26T09:09:20Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: f41488de018402e3d5cfd01fa5351cdb6107412477d5d54e2d9e186e0fc8ee94
    source_path: plugins/meeting-plugins.md
    workflow: 16
---

OpenClaw には、Google Meet、Microsoft Teams 会議、Zoom 用の個別のプラグインがあります。3 つすべてが Chrome 経由で参加でき、同じ参加モードを使用し、Gateway ホストまたはペアリングされた Node で Chrome を実行できます。プラットフォームの URL、インストール方式、追加機能はそれぞれ異なります。

これらのプラグインは会議に参加するためのものです。[Microsoft Teams チャンネル](/ja-JP/channels/msteams)などのメッセージングチャンネルや、[音声通話プラグイン](/ja-JP/plugins/voice-call)とは別のものです。

## プラグインを選択する

| プラットフォーム | プラグイン                                      | 対応する会議リンク                                                                                      | インストール                                    | 参加経路                                      | プラットフォーム固有の機能                                                                                |
| --------------- | ------------------------------------------- | ----------------------------------------------------------------------------------------------------------- | ----------------------------------------------- | -------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------- |
| Google Meet     | [`google-meet`](/ja-JP/plugins/google-meet)       | `meet.google.com/...`                                                                                       | npm または ClawHub からインストール。デフォルトで有効 | ローカル Chrome、ペアリングされた Node 上の Chrome、または Twilio ダイヤルイン | Meet API またはログイン済みブラウザから会議を作成可能。OAuth を使用して、対応する Meet アーティファクトを読み取り可能 |
| Microsoft Teams | [`teams-meetings`](/plugins/teams-meetings) | `teams.microsoft.com/l/meetup-join/...` 配下の職場向けリンクと `teams.live.com/meet/...` 配下の個人向けリンク | 同梱。デフォルトで有効                    | ローカル Chrome またはペアリングされた Node 上の Chrome                  | 職場向けおよび個人向け会議へのゲスト参加                                                                     |
| Zoom            | [`zoom-meetings`](/plugins/zoom-meetings)   | `zoom.us/j/...` および `example.zoom.us/j/...` などのアカウントサブドメイン                                      | 同梱。デフォルトで有効                    | ローカル Chrome またはペアリングされた Node 上の Chrome                  | Zoom Web App を介したゲスト参加                                                                           |

会議の作成、Google API アーティファクト、または Twilio の電話経路が必要な場合は、Google Meet を選択します。これらのプラットフォームでブラウザから直接ゲスト参加する場合は、Teams または Zoom を選択します。Teams および Zoom プラグインは、会議の作成、ダイヤルイン、ベンダー API の呼び出し、音声・映像の録画を行いません。

## モードを選択する

3 つのプラグインは同じモードを共有します。

| モード         | 動作                                                                                              | 音声要件                                      |
| ------------ | ----------------------------------------------------------------------------------------------------- | ------------------------------------------------------- |
| `agent`      | リアルタイム文字起こしが設定済みの OpenClaw エージェントに送られ、通常の OpenClaw TTS が応答を読み上げます。  | Chrome からの音声応答には BlackHole と SoX のブリッジが必要です。 |
| `bidi`       | リアルタイム音声モデルが音声を聞き取り、直接応答します。                                                  | Chrome からの音声応答には BlackHole と SoX のブリッジが必要です。 |
| `transcribe` | 聴講専用で参加し、プラットフォームが字幕を提供する場合は、上限付きのライブ字幕トランスクリプトを公開します。 | BlackHole または SoX の音声応答ブリッジは不要です。                   |

エージェントに必要なのが会議のテキストだけの場合は、`transcribe` を使用します。通常の OpenClaw の推論とツールには、`agent` を使用します。各ターンを通常のエージェント経由で処理することより、低遅延の直接音声を重視する場合は、`bidi` を使用します。

上限付きのライブトランスクリプトを利用できるのは、`transcribe` モードのみです。3 つすべての
モードで、ブラウザ参加は完了した字幕行と、そこから生成された
要約も共有状態データベースに永続化します。会議から退出すると表示中の
字幕が確定され、要約が書き込まれます。一覧表示、確認、またはエクスポートするには、
[`openclaw transcripts`](/ja-JP/cli/transcripts)
を使用します。この永続的な議事録経路によって、ライブの
エージェント参照用トランスクリプトが変更されたり、音声・映像録画が作成されたりすることはありません。

自動議事録はデフォルトで有効です。永続的な議事録を
グローバルに無効にするには、`transcripts.enabled: false` を設定します。明示的に選択された `transcribe` セッションでは、
永続的な行を書き込まずに、上限付きのライブ字幕末尾が保持されます。字幕を利用できるかどうかは、
引き続き会議プラットフォーム、アカウント、言語、およびホストのポリシーに依存します。

## Chrome と音声を準備する

Chrome は Gateway ホストまたはペアリングされた Node で実行できます。リモートの Chrome Node では、`browser.proxy` とプラットフォームコマンドを許可する必要があります。

| プラグイン          | Node コマンド           |
| --------------- | ---------------------- |
| Google Meet     | `googlemeet.chrome`    |
| Microsoft Teams | `teamsmeetings.chrome` |
| Zoom            | `zoommeetings.chrome`  |

Chrome 経由で `agent` または `bidi` モードを使用する場合は、macOS 上で Chrome を実行し、同じホストに共有音声依存関係をインストールします。

```bash
brew install blackhole-2ch sox
sudo reboot
system_profiler SPAudioDataType | grep -i BlackHole
command -v sox
```

Chrome をペアリングされた Node で実行する場合も、OpenClaw エージェントとモデルの認証情報は Gateway ホストが保持します。`agent` モードではリアルタイム文字起こしプロバイダーと OpenClaw TTS を設定し、`bidi` モードではリアルタイム音声プロバイダーを設定します。各プラットフォームのガイドに、プロバイダーと音声コマンドのオプションが記載されています。

## プラグインをインストールまたは無効化する

Google Meet は個別にインストールします。インストール後はデフォルトで有効になります。Teams 会議と Zoom は OpenClaw に同梱され、デフォルトで有効です。

```bash
# Google Meet のみ
openclaw plugins install npm:@openclaw/google-meet
```

使用しない会議プラグインは無効にします。

```bash
openclaw plugins disable google-meet
openclaw plugins disable teams-meetings
openclaw plugins disable zoom-meetings
```

プラグイン管理の処理経路で Gateway が自動的に再起動されない場合は、Gateway を再起動します。その後、参加する前にプラットフォームのセットアップチェックを実行します。

## 検証して参加する

| プラットフォーム | セットアップチェック                    | 参加コマンド                                                                  |
| --------------- | ------------------------------ | ----------------------------------------------------------------------------- |
| Google Meet     | `openclaw googlemeet setup`    | `openclaw googlemeet join 'https://meet.google.com/abc-defg-hij'`             |
| Microsoft Teams | `openclaw teamsmeetings setup` | `openclaw teamsmeetings join 'https://teams.microsoft.com/l/meetup-join/...'` |
| Zoom            | `openclaw zoommeetings setup`  | `openclaw zoommeetings join 'https://zoom.us/j/1234567890'`                   |

セットアップチェックが失敗した場合は、そのトランスポートとモードの進行を妨げる問題として扱います。聴講専用のスモークテストでは、`transcribe` モードを選択し、字幕テキストが表示されることを期待する前に、ステータスに通話中のセッションが表示されることを確認します。

音声応答のスモークテストでは、音声が検証済みと判断されるためには、再生コマンドがバイトを受け入れるだけでは不十分です。共有コマンドペアブリッジは、現在の出力生成から得た上限付きの波形フィンガープリントと、BlackHole マイクのキャプチャ経路に戻ってくる音声を照合します。Google Meet、Teams、Zoom は、出力バイトカウンターだけが増加している場合や、無関係な参加者の音声が存在する場合には、`speechOutputVerified: true` を報告しません。

## プラットフォームのポリシープロンプトに対処する

ブラウザ自動化は、通常のゲスト名、参加前のカメラとマイク、参加、通話中、退出のコントロールを処理します。プラットフォームや主催者のポリシーを回避するものではありません。

- Google Meet では、Google へのログイン、ホストによる入室許可、またはブラウザ権限の決定が必要になる場合があります。
- Microsoft Teams では、テナントへのログイン、メール確認、または主催者による入室許可が必要になる場合があります。
- Zoom では、認証、メール確認、パスコード、CAPTCHA の完了、またはホストによる入室許可が必要になる場合があります。また、アカウントの設定でブラウザからの参加が無効になっていることもあります。

参加またはステータスの結果で `manualActionRequired` が報告された場合は、再試行する前に、同じ OpenClaw Chrome プロファイルで報告された手順を完了します。新しいタブを繰り返し開いても、アカウント、テナント、ロビー、または CAPTCHA の制限は解決しません。

オペレーターがエージェントを追加する権限を持つ会議にのみ参加してください。地域のポリシーや同意規則により、自動参加、文字起こし、合成音声についての開示が必要な場合は、参加者に通知してください。

## Discord ボイスチャット

[Discord ボイスチャンネル](/ja-JP/channels/discord#voice-channels)では、ブラウザ会議の自動化を使用せず、音声のみのネイティブなリアルタイム会話を利用できます。OpenClaw はボイスチャンネルに参加し、音声を聞き取り、各ターンを OpenClaw エージェントまたはリアルタイム音声モデル経由で処理し、応答を読み上げることができます。同じ Discord チャンネルで参加者が映像を使用している場合でも、カメラ映像や画面共有を送受信しません。そのため、Discord ボイスは 4 つ目のブラウザ会議プラグインではなく、関連するライブ会話機能です。

## プラットフォームガイド

- [Google Meet プラグイン](/ja-JP/plugins/google-meet)
- [Microsoft Teams 会議プラグイン](/plugins/teams-meetings)
- [Zoom 会議プラグイン](/plugins/zoom-meetings)
- [プラグインを管理する](/ja-JP/plugins/manage-plugins)
- [ブラウザ制御](/ja-JP/tools/browser)
