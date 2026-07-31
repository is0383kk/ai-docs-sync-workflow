---
read_when:
    - Control UI で1日の流れを Dayflow 風のタイムラインとして表示したい場合
    - バンドルされている Logbook Plugin を有効化または設定しています
    - 画面上のアクティビティに基づくスタンドアップ要約や一日の振り返りが必要な場合
summary: 定期的な画面スナップショットから自動生成されるオプションの作業ジャーナル
title: ログブック Plugin
x-i18n:
    generated_at: "2026-07-26T10:10:22Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 19197e580421dfe81f82f8599578e4c68a15004813bb2b6c3de761c14f426b08
    source_path: plugins/logbook.md
    workflow: 16
---

Logbook Plugin は、画面上のアクティビティを自動的な作業日誌に変換します。
ペアリングされた Node から画面のスナップショットを定期的に取得し、タイムスタンプ付きの
観察結果として要約して、[Control UI](/ja-JP/web/control-ui) にタイムラインカードを
作成します。また、日次スタンドアップノートを生成したり、記録した日についての
質問に回答したりすることもできます。

OpenClaw が所有する状態は Gateway 上の `<state-dir>/logbook/` に保持されますが、
モデル処理が必ずしもローカルで行われるとは限りません。サンプリングされたスクリーンショットは
設定済みのビジョンルートに送信され、観察結果とタイムラインテキストはデフォルトの
エージェントモデルに送信されます。画面の内容と、それから生成されたアクティビティテキストを
マシン上に保持する必要がある場合は、両方の段階でローカルモデルルートを使用してください。

Logbook はバンドルされていますが、デフォルトでは無効です。Plugin を有効にすると、
`captureEnabled` のデフォルトが `true` であるため、Gateway で
画面キャプチャが有効になります。

## はじめる前に

以下が必要です。

- `screen.snapshot` または `logbook.snapshot` を公開する、接続済みの Node。
  macOS アプリの Node には画面収録の権限が必要です。ヘッドレス macOS Node ホスト
  (`openclaw node host run`) では、システムの `screencapture` ツールを使用する、
  Plugin 提供の `logbook.snapshot` コマンドを利用できます。
- バンドルされた Codex Plugin が有効化および認証されていること。現在 Codex は、
  Logbook に必要な構造化画像抽出コントラクトを提供します。
  `openclaw models auth login --provider openai` でサインインしてください。その他の認証方法については、
  [Codex ハーネス](/ja-JP/plugins/codex-harness) を参照してください。
- 動作するデフォルトのエージェントモデル。Logbook はビジョン処理後に、カード、
  スタンドアップノート、日単位の Q&A を合成するためにこのモデルを使用します。

## クイックスタート

Codex と Logbook の Plugin を有効にします。

```bash
openclaw plugins enable codex
openclaw plugins enable logbook
```

確定的に起動できるよう、ビジョンモデルを明示的に設定します。

```json5
{
  plugins: {
    entries: {
      codex: {
        enabled: true,
      },
      logbook: {
        enabled: true,
        config: {
          visionModel: "codex/gpt-5.6-sol",
        },
      },
    },
  },
}
```

`plugins.allow` を使用する場合は、`codex` と `logbook` の
両方を含めてください。Plugin の設定を変更した後に Gateway を再起動し、
登録内容を確認してダッシュボードを開きます。

```bash
openclaw gateway restart
openclaw plugins inspect logbook --runtime --json
openclaw nodes status --connected
openclaw nodes describe --node <idOrNameOrIp>
openclaw dashboard
```

Node の説明には `screen.snapshot` または `logbook.snapshot` が含まれている
必要があります。ヘッドレス Node は Plugin がアクティブになった後にのみ
`logbook.snapshot` を通知します。コマンドがない場合は、
[Node のトラブルシューティング](/ja-JP/nodes/troubleshooting) を参照してください。

Logbook タブは、Plugin が有効であり、かつ `operator.write` の
Control UI セッションである場合にのみ表示されます。ステータス行にはエラーなしで
**キャプチャ中**と表示されるはずです。分析ウィンドウが終了するとタイムラインカードが
表示されます。または、アクティビティのキャプチャ後に **今すぐ分析** を選択できます。

## 仕組み

1. **キャプチャ**: `captureIntervalSeconds` ごと（デフォルトは 30 秒）に、Logbook は
   選択した Node のキャプチャコマンドを呼び出し、縮小した JPEG フレームを保存します。
   連続して同一のフレームはアイドルとしてマークされ、分析から除外されます。
2. **観察**: 分析ウィンドウ（デフォルトは 15 分）が経過すると、
   Plugin は最大 16 個のアクティブなフレームをサンプリングしてビジョンモデルに送信し、
   モデルはタイムスタンプ付きのアクティビティ観察結果（「VS Code: store.ts を編集し、
   型エラーを修正」）を返します。2 分を超えるキャプチャの中断、またはローカルの午前 0 時でも、
   現在のウィンドウが終了します。
3. **合成**: 観察結果と既存カードの直近 45 分を、タイトル、要約、
   カテゴリ、メインアプリ、短時間の気の散りを含むタイムラインカード
   （各 10～60 分）に再構成します。
4. **削除**: `retentionDays` 日（デフォルトは 14 日）より古いフレームを削除します。
   カード、観察結果、キャッシュされたスタンドアップは保持されます。

日の境界とタイムラインの時刻には、ブラウザのタイムゾーンではなく Gateway の
ローカルタイムゾーンが使用されます。フレームと SQLite タイムラインデータベースは
`<state-dir>/logbook/` に保存されます。

## モデルとデータの流れ

Logbook は、2 つの異なるモデルルートを使用します。

| 段階             | 送信されるデータ                                            | モデルルート                                                        |
| ---------------- | --------------------------------------------------------- | ----------------------------------------------------------------- |
| 観察             | 最大 16 個のサンプリング済み JPEG フレームとそのキャプチャ時刻 | `visionModel`、または互換性のある借用された `tools.media` Codex エントリ |
| カードの合成     | タイムスタンプ付きの観察結果と最近のタイムラインカード       | Plugin LLM ランタイムを介したデフォルトのエージェントモデル           |
| スタンドアップの生成 | 選択した日と前日のカード                                 | Plugin LLM ランタイムを介したデフォルトのエージェントモデル           |
| その日について質問 | 質問、選択した日のカード、最近の観察結果                  | Plugin LLM ランタイムを介したデフォルトのエージェントモデル           |

SQLite データベース全体が、どちらかのモデルに送信されることはありません。未加工の
スクリーンショットは観察段階にのみ送信されます。カードの合成、スタンドアップ、Q&A には、
生成されたテキストが渡されます。

## 設定

```json5
{
  plugins: {
    entries: {
      codex: {
        enabled: true,
      },
      logbook: {
        enabled: true,
        config: {
          captureEnabled: true,
          captureIntervalSeconds: 30,
          analysisIntervalMinutes: 15,
          nodeId: "my-mac",
          screenIndex: 0,
          maxWidth: 1440,
          visionModel: "codex/gpt-5.6-sol",
          retentionDays: 14,
        },
      },
    },
  },
}
```

Logbook のすべての設定キーは省略可能です。数値は整数に丸められ、
サポートされる範囲内に制限されます。

| キー                       | デフォルト | 範囲または値             | 動作                                                                                         |
| ------------------------- | ------- | ----------------------- | -------------------------------------------------------------------------------------------- |
| `captureEnabled`          | `true`  | 真偽値                  | 新しいスナップショット用の永続的なマスタースイッチ。`false` の場合もタイムラインは利用可能 |
| `captureIntervalSeconds`  | `30`    | `5`-`600`               | キャプチャ試行間の遅延                                                                         |
| `analysisIntervalMinutes` | `15`    | `3`-`120`               | 対象となる観察ウィンドウ。中断や午前 0 時によって早く終了することがあります                        |
| `nodeId`                  | 未設定   | Node ID または表示名     | キャプチャを 1 つの接続済み Node に固定します。照合では大文字と小文字を区別しません                 |
| `screenIndex`             | `0`     | `0`-`16`                | 0 から始まるディスプレイインデックス                                                           |
| `maxWidth`                | `1440`  | `480`-`3840`            | 要求するキャプチャサイズの上限。ヘッドレス macOS では最長辺に適用されます                          |
| `visionModel`             | 未設定   | `provider/model`        | 明示的な構造化ルート。不正な形式の参照では分析が一時停止し、未対応のプロバイダーではバッチが失敗します |
| `retentionDays`           | `14`    | `1`-`365`               | 古いフレームを削除します。カード、観察結果、スタンドアップは保持されます                             |

`nodeId` がない場合、Logbook はまず `screen.snapshot` を公開する
接続済みのアプリ Node を優先し、その後 `logbook.snapshot` を公開するヘッドレス Node に
フォールバックします。固定されていない構成では、障害が発生した Node は他の適格な Node の
後ろにローテーションされます。ダッシュボードの一時停止切り替えはセッション内でのみ有効で、
Gateway の再起動時にリセットされます。永続的に停止するには `captureEnabled: false` を使用してください。

### ビジョンモデルの選択

Logbook は、次の順序で観察モデルを解決します。

1. `plugins.entries.logbook.config.visionModel`
2. `tools.media.models` 配下にある、画像対応の最初の Codex エントリ

その他のメディアプロバイダーは、現在 Logbook に必要な構造化抽出コントラクトを
公開していないためスキップされます。`tools.media.image.enabled: false` を設定すると、借用された
メディアのデフォルトは無効になりますが、明示的な Logbook の
`visionModel` は引き続き適用されます。

## ダッシュボードタブ

- **タイムライン**: カテゴリの色、メインアプリ、気の散りを示すチップ、
  スナップショットのキーフレームを備えた、アクティビティごとの展開可能なカード。
- **一日の概要**: 集中時間の比率、カテゴリの内訳、よく使用したアプリ。
- **日次スタンドアップ**: 昨日と今日の情報を、すぐに貼り付けられる更新内容に変換します。
- **その日について質問**: 記録されたタイムラインに基づいて自然言語の質問に回答します
  （「Gateway の PR をレビューしたのはいつですか？」）。
- **今すぐ分析**: 分析間隔を待たず、現在のキャプチャウィンドウをただちに終了します。

## Gateway メソッド

Logbook は次の Gateway RPC メソッドを登録します。

| メソッド              | パラメーター             | スコープ          | 結果                                                                     |
| --------------------- | ------------------------ | ---------------- | ------------------------------------------------------------------------ |
| `logbook.status`      | なし                     | `operator.read`  | キャプチャ、分析、モデル、Node、Gateway の日付、Gateway のタイムゾーンの状態 |
| `logbook.days`        | なし                     | `operator.read`  | タイムラインカード数とカードの時間範囲を含む日付                           |
| `logbook.timeline`    | `{ day?: "YYYY-MM-DD" }` | `operator.read`  | 生成されたカードと日次統計。デフォルトは Gateway の現在の日付               |
| `logbook.frames`      | `{ startMs, endMs }`     | `operator.write` | 要求されたエポックミリ秒範囲内のフレームメタデータ                           |
| `logbook.frame`       | `{ frameId }`            | `operator.write` | base64 形式の未加工 JPEG フレーム 1 個                                    |
| `logbook.standup`     | `{ day?, refresh? }`     | `operator.write` | ある日についてキャッシュ済み、または再生成されたスタンドアップテキスト          |
| `logbook.ask`         | `{ day?, question }`     | `operator.write` | ある日のタイムラインに基づく回答                                            |
| `logbook.capture.set` | `{ paused }`             | `operator.write` | セッション内のみの一時停止状態と更新後のステータス                            |
| `logbook.analyze.now` | なし                     | `operator.write` | 保留中の分析を開始するか、開始できなかった理由を返す                           |

読み取りメソッドは、運用状態または生成されたテキストを返します。未加工のスクリーンショットの
ピクセル、モデルの費用が発生するアクション、ランタイムの変更には
`operator.write` が必要です。Control UI タブでは、これらのアクションと未加工フレームの
プレビューを公開するため、`operator.write` も必要です。読み取り専用クライアントでも、
生成テキストのメソッドを直接呼び出すことはできます。

## プライバシーに関する注意事項

- スナップショットには、シークレットを含め、画面上のあらゆるものが含まれる可能性があります。フレームがマシンの外部に送信されるのは、設定された観測モデルへのサンプリング入力として使用される場合に限られます。
- 観測、最近のカード、質問は、カードの合成、スタンドアップの生成、Q&A の際に、デフォルトのエージェントモデルを通じてマシンの外部に送信される場合があります。両方のモデルルートにプロバイダーのデータ処理ポリシーを適用してください。
- 完全にローカルなパイプラインが必要な場合は、構造化観測モデルとデフォルトのエージェントモデルの両方にローカルルートを使用してください。
- フレーム、タイムラインデータベース、一時キャプチャは、所有者のみがアクセスできるファイル権限で書き込まれます。
- `screen.snapshot` を `gateway.nodes.commands.deny` に追加すると、画面キャプチャのキルスイッチとして機能し、アプリ Node のキャプチャと Logbook 自体の `logbook.snapshot` コマンドの両方をブロックします。
- `tools.media.image.enabled: false` を設定すると、Logbook が分析にメディア画像モデルを借用することも停止します。この場合、Plugin 設定で明示的に指定された `visionModel` のみが使用されます。

## トラブルシューティング

### Logbook タブが表示されない

次の 3 つのゲートをすべて確認してください。

1. `openclaw plugins list --enabled` に `logbook` が含まれている。
2. Plugin または許可リストを変更した後に Gateway が再起動されている。
3. Control UI 接続に `operator.write` がある。読み取り専用セッションにはインタラクティブなタブ記述子は送信されません。

`plugins.allow` が設定されている場合、推奨設定では `logbook` と `codex` の両方を含める必要があります。

### キャプチャでエラーが報告される

```bash
openclaw nodes status --connected
openclaw nodes describe --node <idOrNameOrIp>
openclaw logs --follow
```

- Node が `screen.snapshot` または `logbook.snapshot` を公開していることを確認してください。
- キャプチャを行う Mac で Screen Recording 権限を付与してください。
- `nodeId` が設定されている場合、Node ID または表示名と一致することを確認してください。
- `gateway.nodes.commands.deny` に `screen.snapshot` が含まれていないことを確認してください。

3 回連続で失敗すると、Logbook は 10 回のキャプチャティックの間バックオフしてから再試行します。固定されていない設定では、別の適格な Node に切り替わる場合があります。

### キャプチャは成功するがカードが表示されない

- **モデルがありません**というステータスは、互換性のある構造化ビジョンルートが見つからなかったことを意味します。Codex Plugin を有効化して認証するか、有効な `visionModel` を明示的に設定してください。モデルがない間、キャプチャされたフレームは保留状態のままになり、設定を修正した後に分析できます。
- `analysisIntervalMinutes` を待つか、アクティビティがキャプチャされた後に **今すぐ分析** を選択してください。
- 連続する同一フレームはアイドル状態の証拠と見なされ、分析バッチには入りません。テストする前に、表示されている画面を変更してください。
- 最新のバッチにエラーが表示されている場合は、モデルまたは認証の問題を修正して **今すぐ分析** を選択してください。モデルの使用料金が繰り返し発生するのを避けるため、失敗したバッチはこの明示的な操作を行った場合にのみ再試行されます。

## 関連項目

- [Plugin を管理する](/ja-JP/plugins/manage-plugins)
- [Codex ハーネス](/ja-JP/plugins/codex-harness)
- [メディア理解](/ja-JP/nodes/media-understanding)
- [Node](/ja-JP/nodes)
- [Node のトラブルシューティング](/ja-JP/nodes/troubleshooting)
- [Control UI](/ja-JP/web/control-ui)
