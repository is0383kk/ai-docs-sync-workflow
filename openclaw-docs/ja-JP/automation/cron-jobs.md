---
read_when:
    - バックグラウンドジョブまたはウェイクアップのスケジュール設定
    - 外部トリガー（Webhook、Gmail）を OpenClaw に接続する
    - スケジュールされたタスクで Heartbeat と Cron のどちらを使用するかの判断
sidebarTitle: Scheduled tasks
summary: Gateway スケジューラ向けのスケジュール済みジョブ、Webhook、Gmail PubSub トリガー
title: スケジュールされたタスク
x-i18n:
    generated_at: "2026-07-26T10:04:22Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: dd889cf8e45196eda3ec7c2af930abcb2cc2bae8bad2dbdcaf3cd521a9e884b2
    source_path: automation/cron-jobs.md
    workflow: 16
---

Cron は Gateway に組み込まれたスケジューラです。ジョブを永続化し、適切な時刻にエージェントを起動し、出力をチャットチャンネルや Webhook に配信することも、どこにも配信しないこともできます。

## クイックスタート

<Steps>
  <Step title="1 回限りのリマインダーを追加する">
    ```bash
    openclaw cron create "2027-02-01T16:00:00Z" \
      --name "リマインダー" \
      --session main \
      --system-event "リマインダー: Cron ドキュメントの下書きを確認する" \
      --wake now \
      --delete-after-run
    ```
  </Step>
  <Step title="ジョブを確認する">
    ```bash
    openclaw cron list
    openclaw cron get <job-id>
    openclaw cron show <job-id>
    ```
  </Step>
  <Step title="実行履歴を確認する">
    ```bash
    openclaw cron runs --id <job-id>
    ```
  </Step>
</Steps>

## Cron の仕組み

- Cron はモデル内ではなく、**Gateway プロセス内**で実行されます。スケジュールを実行するには Gateway が稼働している必要があります。
- ジョブ定義、ランタイム状態、実行履歴は OpenClaw の共有 SQLite 状態データベースに永続化されるため、再起動してもスケジュールは失われません。
- Cron を実行するたびに、[バックグラウンドタスク](/ja-JP/automation/tasks)のレコードが作成されます。
- 1 回限りのジョブ（`--at`）は、デフォルトでは成功後に自動削除されます。保持するには `--keep-after-run` を渡します。
- 実行ごとの実時間上限: 設定されている場合は `--timeout-seconds`。それ以外の場合、分離またはデタッチされたエージェントターンのジョブは、基盤となるエージェントターンのタイムアウト（`agents.defaults.timeoutSeconds`、デフォルトは 48 時間）が適用されるより前に、Cron 独自の 60 分間のウォッチドッグによって制限されます。コマンドジョブのデフォルトは 10 分、スクリプトペイロードのデフォルトは 5 分です。
- Gateway の起動時、期限を過ぎた分離エージェントターンのジョブは即座に再実行されず、再スケジュールされます。これにより、モデルやツールのブートストラップ処理がチャンネル接続期間中に行われることを防ぎます。
- システム Cron または別の外部スケジューラから `openclaw agent` を実行する場合、CLI がすでに `SIGTERM`/`SIGINT` を処理していても、強制終了へ移行する仕組みでラップしてください。Gateway による実行では、受理された実行を中止するよう Gateway に要求します。`--local` の実行にも同じ中止シグナルが送られます。GNU `timeout` では、単純な `timeout 600 ...` より `timeout -k 60 600 openclaw agent ...` を推奨します。プロセスを時間内に終了できない場合、`-k` の値が最後の安全策になります。systemd ユニットでは、最終的な強制終了の前に猶予期間（`TimeoutStopSec`）を設け、`SIGTERM` 停止シグナルを使用します。元の Gateway 実行がまだアクティブな間に `--run-id` を再利用すると、2 回目の実行を開始する代わりに、重複が実行中として報告されます。

<AccordionGroup>
  <Accordion title="分離実行の堅牢化">
    - 分離実行は完了時に、`cron:<jobId>` セッションで追跡されているブラウザタブやプロセスをベストエフォートで閉じます。また、ジョブ用に作成されたバンドル済み MCP ランタイムインスタンスを、メインセッションやカスタムセッションの実行と同じ共有終了処理によって破棄します。クリーンアップに失敗しても無視されるため、Cron の結果が優先されます。
    - 限定的な Cron 自己クリーンアップ権限を持つ分離実行は、スケジューラの状態、自身のジョブのみを含む自己フィルタリング済み一覧、およびそのジョブの実行履歴を読み取ることができ、自身のジョブのみを削除できます。
    - 分離実行では、古い確認応答が返されないよう防止します。最初の結果が一時的な状態更新（`on it`、`pulling everything together`、および同様の通知）だけで、子孫サブエージェントのいずれも最終回答を担当していない場合、OpenClaw は配信前に実際の結果を得るため 1 回だけ再プロンプトします。
    - 構造化された実行拒否メタデータ（ネストされたエラーが `SYSTEM_RUN_DENIED` または `INVALID_REQUEST` で始まる Node ホストの `UNAVAILABLE` ラッパーを含む）が認識されるため、ブロックされたコマンドが成功した実行として報告されることはありません。一方、通常のアシスタントの文章が拒否と誤認されることもありません。
    - 返信ペイロードがない場合でも、実行レベルのエージェント障害はジョブエラーとして扱われます。そのため、モデルやプロバイダーの障害ではエラーカウンターが増加し、ジョブを成功として処理する代わりに障害通知がトリガーされます。
    - ジョブが `timeoutSeconds` に達すると、Cron は実行を中止し、短いクリーンアップ期間を設けます。期間内に終了しない場合、Gateway が所有するクリーンアップ処理によってその実行のセッション所有権が強制的に解除されてから、Cron がタイムアウトを記録します。これにより、キューに入ったチャット処理が古い処理セッションによってブロックされることを防ぎます。
    - セットアップや起動の停止には、フェーズ固有のタイムアウト（例: `cron: isolated agent setup timed out before runner start` または `cron: isolated agent run stalled before execution start (last phase: context-engine)`）が適用されます。これらのウォッチドッグは、外部 CLI プロセスが開始される前でも、組み込みプロバイダーと CLI ベースのプロバイダーの両方を対象とします。また、長い `timeoutSeconds` の値とは独立して上限が設定されるため、コールドスタート、認証、コンテキストの障害が速やかに表面化します。

  </Accordion>
  <Accordion title="タスクの整合">
    Cron タスクの整合では、まずランタイムの所有状態を使用し、次に永続化された履歴を使用します。古い子セッション行が残っていても、Cron ランタイムがそのジョブを実行中として追跡している間、アクティブな Cron タスクは実行中のままです。ランタイムがジョブを所有しなくなり、5 分間の猶予期間が経過すると、メンテナンス処理は一致する `cron:<jobId>:<startedAt>` の実行について、永続化された実行ログとジョブ状態を確認します。そこに終了結果があればタスク台帳を確定します。それ以外の場合、Gateway が所有するメンテナンス処理はタスクを `lost` としてマークできます。オフラインの CLI 監査では永続化された履歴から復旧できますが、そのプロセス内のアクティブジョブ集合が空であることだけでは、Gateway が所有する実行が終了した証明にはなりません。
  </Accordion>
</AccordionGroup>

## スケジュールの種類

| 種類      | CLI フラグ           | 説明                                                                                              |
| --------- | ------------------ | -------------------------------------------------------------------------------------------------------- |
| `at`      | `--at`             | 1 回限りのタイムスタンプ（ISO 8601、または `20m` のような相対指定）                                                     |
| `every`   | `--every`          | 固定間隔（`10m`、`1h`、`1d`）                                                                       |
| `cron`    | `--cron`           | オプションの `--tz` を伴う 5 フィールドまたは 6 フィールドの Cron 式                                                  |
| `on-exit` | `--on-exit`        | 監視対象のコマンドが終了したときに 1 回実行（イベントトリガー。ターンの終了後も存続。オプションの `--on-exit-cwd`） |
| `stream`  | `--stream-command` | 監視下で長期間実行されるコマンドが生成した行をバッチ処理して実行                                      |

タイムゾーンのないタイムスタンプは UTC として扱われます。オフセットのない `--at` 日時を解釈する場合、またはその IANA タイムゾーンで Cron 式を評価する場合は、`--tz America/New_York` を追加します。`--tz` のない Cron 式には Gateway ホストのタイムゾーンが使用されます。`--tz` は `--every` または `--on-exit` と併用できません。

毎時 0 分に繰り返す式（分フィールドが `0` で、時フィールドがワイルドカード）は、負荷の急増を抑えるため、最大 5 分間自動的に分散されます。正確なタイミングを強制するには `--exact` を使用し、明示的な時間枠を指定するには `--stagger 30s` を使用します（Cron スケジュールのみ）。

### Heartbeat タスクの移行

以前の Heartbeat スクラッチでは、構造化された `tasks:` ブロックがサポートされていました。アップグレード後に `openclaw doctor --fix` を実行すると、各エントリが通常の編集可能なメインセッションの Cron ジョブに変換されます。Doctor は間隔と以前の最終実行時刻を維持し、ブロックを削除する前にジョブを作成します。また、再実行時には同じ宣言キーを安全に収束させます。

移行されたジョブには公開 `systemEvent` ペイロードが含まれるため、`openclaw cron list`、`get`、`edit`、`remove`、および Cron ツールを使用して、ほかのジョブと同様に管理できます。実行には保護された Heartbeat タスクの起動処理が使用されます。アクティブ時間、最小間隔、フラッド制御、ビジー時の再試行は引き続き適用され、Cron が各タスクの独立した実行間隔を管理します。同じ統合時間枠内に期限を迎えるジョブは、1 回の Heartbeat ターンを共有できます。Heartbeat のアクティブ時間外にスケジュールされた実行はスキップされ、そのジョブの次の実行時に再試行されます。

Heartbeat スクラッチは、現在では監視用の文章のみを扱います。ランタイムの Heartbeat は `tasks:` のテキストをスケジュールとして解析しません。新しい繰り返し処理は Cron で作成してください。

### ストリームソース

ストリームスケジュールは、オペレーターが作成した argv コマンドを Gateway の管理下で実行し続け、その標準出力および標準エラー出力の行からジョブを実行します。ストリームスケジュールはイベント駆動型であり、時刻によって期限を迎えることはありません。また、長期間実行されるコマンドはトリガースクリプトと同じ無人実行の信頼クラスに属するため、`cron.triggers.enabled: true` が必要です。ジョブを無効化または削除すると、プロセスが停止します。Gateway のシャットダウン時には、プロセスツリーの終了を待機します。短時間での障害発生時には、Cron に組み込まれたエラーバックオフを使用して再起動します。60 秒未満の実行が 5 回連続すると、ジョブはエラー状態になり、通常の障害アラート処理が使用されます。再起動回数の上限を解除するには、ジョブを手動で再有効化してください。

```bash
openclaw cron add \
  --name "ビルドイベントストリーム" \
  --stream-command '["node","scripts/build-events.mjs"]' \
  --stream-mode match \
  --stream-match '^(failed|recovered):' \
  --stream-batch-ms 250 \
  --session isolated \
  --message "これらのビルドイベントを調査してください。"
```

`mode: "line"`（デフォルト）はすべての行を受け入れます。`mode: "match"` は、コンパイルされた `match` 正規表現に一致する行のみを受け入れます。バッチは、`batchMs` の無入力期間（デフォルトは 250 ms、50～5000 に制限）が経過するか、`maxBatchBytes`（デフォルトは 16384、1024～65536 に制限）に達すると閉じられます。バイト上限に達した場合、バッチの末尾には `[truncated]` が追加されます。マッチモードでは、`maxBatchBytes` を超えた場合でも、常に完全な行の全文に対して評価されます（切り詰められるのは配信されるバッチのみです）。制限された生入力の上限で切断された行は単なるプレフィックスであるため、末尾アンカー付きパターンが切断箇所で一致しないよう、不一致として扱われます。バッチはシステムイベントのテキストまたはエージェントターンのメッセージに追加されます。ソースコマンドとペイロードコマンドのプロセス所有権が曖昧になるため、ストリームスケジュールではコマンドペイロードが拒否されます。

ジョブごとに保持されるのは、実行中のペイロード 1 件と、サイズが制限された保留中のバッチ 1 件のみです。ペイロードの実行中、または組み込みの 30 秒間のトリガー間隔が経過する前に到着した行は、無制限のキューを構築する代わりに、その保留中のバッチへ統合されます。単一の直列化された所有者が、ゲートによる破棄、ペイロードエラー、未実行状態でのディスパッチを `streamDroppedBatches` に記録します。制限付きの統合では `streamCoalescedBatches` が増加します。失敗したペイロードは冪等でない可能性があるため、再試行されません。論理的なソース ID は監視対象の子プロセスが再起動しても維持されますが、ソースが無効化、削除、または置換されると変更されます。そのため、A から B、さらに A へ編集した場合でも、廃止されたソースのキュー済みバッチが実行されることはありません。停止が完了すると、古い子プロセスからの遅延コールバックは何も実行しません。V1 にはネイティブ WebSocket ソースが含まれていません。`websocat wss://example.invalid/events` のような argv コマンドを使用してブリッジしてください。

ストリームジョブに `trigger.script` も設定されている場合、閉じられたバッチごとにゲートが 1 回実行されます。現在のバッチは、`trigger.state` とともに、深く凍結された `trigger.streamBatch` 文字列として使用できます。`fire: false` はゲート状態を永続化した後、そのバッチを破棄します。`fire: true` は既存のトリガーメッセージのセマンティクスを維持し、その後、生成されたペイロードにバッチを追加します。ストリームジョブでは、条件ゲートを使用せず、代わりにスクリプトペイロードを使用することもできます。そのスクリプトは、同じ `trigger.streamBatch` の値を通じてバッチを受け取ります。スクリプトペイロードと条件ゲートを組み合わせると、両方が永続化された `trigger.state` スロットを所有することになるため、拒否されます。

### 動的な実行間隔（ペーシング）

繰り返しジョブでは、`pacing.min` と `pacing.max` の一方または両方を、`15m` や `4h` のような期間文字列に設定できます。少なくとも一方の境界値が必要です。`cron add|edit` とともに `--pacing-min` および `--pacing-max` を使用します（`--clear-pacing` は両方の境界値を削除します）。

分離実行中、ペーシング対象のジョブは `action: "next_check"` および `in: "30m"` を指定して `cron` ツールを呼び出せます。この提案は現在実行中のそのジョブにのみ適用され、実行が正常に完了した時点から計測されます。OpenClaw は、設定された範囲内に暗黙的に収めます。

提案なしのペーシングでは、通常のスケジュールは変更されません。失敗、タイムアウト、スキップされた実行では提案が破棄されるため、既存の再試行およびエラーバックオフの動作が優先されます。定期ジョブを手動で強制実行する操作は帯域外として扱われ、保留中の通常スロットまたはペーシング済みスロットは維持されます。条件トリガー型ジョブでは、提案によってより早いチェックが要求された場合でも、組み込みの最小間隔が下限として維持されます。

### 日付と曜日には OR ロジックを使用する

Cron 式は [croner](https://github.com/Hexagon/croner) によって解析されます。日付フィールドと曜日フィールドの両方がワイルドカードでない場合、croner は両方ではなく、**いずれか**のフィールドが一致すると一致と判定します。これは標準的な Vixie cron の動作です。

```bash
# 意図: 「15日が月曜日の場合にのみ、午前9時」
# 実際: 「毎月15日の午前9時、かつ毎週月曜日の午前9時」
0 9 15 * 1
```

この場合、月に 0～1 回ではなく、およそ 5～6 回実行されます。両方の条件を必須にするには、croner の曜日修飾子 `+`（`0 9 15 * +1`）を使用するか、一方のフィールドでスケジュールし、もう一方をジョブのプロンプトまたはコマンドでチェックしてください。

## イベントトリガー（条件ウォッチャー）

イベントトリガーは、`every`、`cron`、または `stream` スケジュールにヘッドレス条件スクリプトを追加します。時刻スケジュールでは期限到来時に評価し、ストリームスケジュールではクローズされたバッチごとに評価します。スクリプトが `fire: true` を返した場合にのみ、Cron は通常のペイロードを実行します。

```json5
{
  schedule: { kind: "every", everyMs: 30000 },
  trigger: {
    // 観測されたステータスが前回の評価と異なる場合にのみ発火します。
    script: "const res = await tools.call('exec', { command: 'gh pr checks 123 --json state -q \\'.[].state\\' | sort -u' }); const status = String(res?.result?.details?.aggregated ?? '').trim(); json({ fire: status !== trigger.state?.status, message: `PR 123 CI: ${trigger.state?.status ?? 'unknown'} -> ${status}`, state: { status } });",
    once: false,
  },
  payload: { kind: "agentTurn", message: "CI ステータスの変更を調査してください。" },
}
```

スクリプトは `{ fire, message?, state? }` を返す必要があります。以前の JSON 状態は、深く凍結された `trigger.state` として使用できます。ストリームゲートは、現在のバッチも `trigger.streamBatch` として受け取ります。永続化するには、新しい `state` 値を返してください。状態の上限は 16 KB です。発火結果に `message` が含まれる場合、Cron は実行前にそれをシステムイベントのテキストまたはエージェントターンのメッセージへ追加します。`once: true` は、最初に発火したペイロードが正常に完了した後、ジョブを無効にします。

`fire: false` は評価状態とカウンターを永続化し、実行履歴を作成せずに再スケジュールします。発火したペイロードの実行が失敗した場合、返された `state` は永続化**されません**。次の評価では以前の状態が参照され、再び発火する可能性があります。そのため、スクリプトは読み取り専用のチェックとして記述し、アクションはペイロードに保持してください。トリガースケジュールには、組み込みの最小間隔として 30 秒が設定されています。各評価には実時間で 30 秒の制限があり、ツール呼び出しは最大 5 回です。

ウォッチャーは成功だけでなく、**対応可能な状態**を中心に設計してください。チェックが失敗またはタイムアウトしたときに沈黙するウォッチャーは、壊れていても正常に見えます。観測結果を `trigger.state` と比較し、重複を排除するために新しい状態を返してください。モデルやプロセスのメモリに依存しないでください。発火時には、`message` が発火した実行の完全なイベントコンテキストになるため、自己完結する内容にしてください。

<Warning>
`cron.triggers.enabled` を有効にすると、条件トリガースクリプトと `script` ペイロードの両方が、所有エージェントの **`exec` を含む完全なツールポリシー**を使用してヘッドレスで実行できるようになります。これは、そのエージェントの権限を持つ無人コード実行として扱ってください。Cron ジョブの作成を許可されたすべてのエージェントが、それに見合う信頼性を備えている場合を除き、無効のままにしてください。
</Warning>

ローカルスクリプトファイルからウォッチャーを作成します（`-` は標準入力からスクリプトを読み取ります）。

```bash
openclaw cron add \
  --name "PR CI watcher" \
  --every 30s \
  --trigger-script ./watch-pr-ci.js \
  --message "Respond to the CI status change" \
  --session isolated
```

## ペイロード

各ジョブには、フラグで選択されたペイロード種別が正確に 1 つ含まれます。

| ペイロード       | フラグ                                           | 実行内容                                                       |
| ------------- | ---------------------------------------------- | ---------------------------------------------------------- |
| システムイベント  | `--system-event <text>`                        | メインセッションのキューに追加され、それ自体ではモデルを呼び出さない    |
| エージェントメッセージ | `--message <text>`                             | モデルを使用するエージェントターン                                  |
| コマンド       | `--command <shell>` または `--command-argv <json>` | Gateway ホスト上のシェル／プロセス。モデルは呼び出さない         |
| スクリプト        | `--script <file\|->`                           | 所有エージェントのツールを使用するヘッドレスのコードモードスクリプト |

追加のペイロード種別 `heartbeat` はシステム所有です。Gateway は、Heartbeat が有効なエージェントごとに 1 つの Heartbeat 監視ジョブへ収束させます（[Heartbeat](/ja-JP/gateway/heartbeat) を参照）。これは `cron list --all` に表示されますが、CLI または API を介して作成・編集することはできません。Heartbeat の設定は、起動時、設定の再読み込み時、または `openclaw doctor --fix` によって、永続化された監視スケジュールへ反映されます。Cron が無効な場合、監視ジョブはティックせず、代替の Heartbeat タイマーも実行されません。

### エージェントターンのオプション

<ParamField path="--message" type="string" required>
  プロンプトテキスト（分離／現在／カスタムセッションのジョブでは必須）。
</ParamField>
<ParamField path="--model" type="string">
  モデルの上書き。許可されたモデルに解決される必要があり、解決されない場合、実行は検証エラーで失敗します。
</ParamField>
<ParamField path="--fallbacks" type="string">
  ジョブ単位のフォールバックモデル一覧。例: `--fallbacks openai/gpt-5.6-sol,openrouter/meta-llama/llama-3.3-70b-instruct:free`。フォールバックなしの厳密な実行には `--fallbacks ""` を渡します。
</ParamField>
<ParamField path="--clear-fallbacks" type="boolean">
  `cron edit` で、ジョブ単位のフォールバック上書きを削除し、設定済みのフォールバック優先順位に従うようにします。`--fallbacks` とは併用できません。
</ParamField>
<ParamField path="--clear-model" type="boolean">
  `cron edit` で、ジョブ単位のモデル上書きを削除し、通常の Cron モデル優先順位（保存された Cron セッションの上書き、それがなければエージェント／デフォルトモデル）に従うようにします。`--model` とは併用できません。
</ParamField>
<ParamField path="--thinking" type="string">
  思考レベルの上書き（`off|minimal|low|medium|high|xhigh|adaptive|max|ultra`）。使用可能なレベルは、選択されたモデルとエージェントランタイムにも依存します。
</ParamField>
<ParamField path="--clear-thinking" type="boolean">
  `cron edit` で、ジョブ単位の思考レベル上書きを削除します。`--thinking` とは併用できません。
</ParamField>
<ParamField path="--light-context" type="boolean">
  ワークスペースのブートストラップファイルの注入をスキップします。
</ParamField>
<ParamField path="--tools" type="string">
  ジョブが使用できるツールを制限します。例: `--tools exec,read`。
</ParamField>

ツールを実行できる新しいジョブには、常に明示的なツールポリシーが保存されます。エージェントが作成したジョブは、作成元のターンで使用可能なツールを上限とし、エージェントは保存された一覧を拡張できません。`--tools` を指定せずに認証済みオペレーターが作成したジョブには、制限なしの `*` ポリシーが保存されます。`cron edit --clear-tools` は、その明示的な制限なしポリシーを復元します。明示的なツールポリシーの導入前に作成された既存のジョブは、ツールポリシーが明示的に編集されるかジョブが再作成されるまで、現在の動作を維持します。

`--model` はジョブのプライマリモデルを設定します。セッションの `/model` 上書きは置き換えないため、設定済みのフォールバックチェーンは引き続きその上に適用されます。解決できない、または許可されていないモデルの場合、デフォルトへ暗黙的にフォールバックせず、明示的な検証エラーで実行が失敗します。ジョブに `--model` があり、明示的または設定済みのフォールバック一覧がない場合、OpenClaw はエージェントのプライマリを隠れた再試行先として暗黙的に追加せず、空のフォールバック上書きを渡します。

分離ジョブのモデル選択優先順位は、上から順に次のとおりです。

1. ジョブ単位のペイロード `model`（明示的な設定。許可されていないモデルでは実行が失敗）
2. Gmail フックのモデル上書き（Gmail から開始された実行で、かつその上書きが許可されている場合のみ）
3. ユーザーが選択し、保存された Cron セッションのモデル上書き
4. エージェント／デフォルトのモデル選択

高速モードは、解決された現在の選択に従います。選択されたモデル設定に `params.fastMode` がある場合、分離 Cron はそれをデフォルトで使用します。ただし、保存されたセッションの `fastMode` 上書き（次にエージェントの `fastModeDefault`）は、どちらの方向でもモデル設定より優先されます。自動モードはモデルの `params.fastAutoOnSeconds` しきい値を使用し、デフォルトは 60 秒です。

実行中にライブモデル切り替えの引き継ぎが発生した場合、Cron は切り替え後のプロバイダー／モデルで再試行し、その選択（および新しい認証プロファイルがある場合はそれも）を現在の実行に対して永続化します。再試行回数には上限があります。最初の試行に加えて切り替え再試行を 2 回行った後は、ループせずに Cron が中止します。

分離実行を開始する前に、OpenClaw は、`baseUrl` が local loopback、プライベートネットワーク、または `.local` である、設定済みの `api: "ollama"` および `api: "openai-completions"` プロバイダーについて、到達可能なローカルエンドポイントを確認します。この事前確認では、ジョブに設定されたフォールバックチェーンを順に調べ、すべての候補に到達できない場合にのみ実行を `skipped` としてマークします。`--fallbacks ""` は、この確認をプライマリモデルのみに厳密に限定します。停止中のエンドポイントでは、モデル呼び出しを開始せず、明確なエラーとともに実行を `skipped` として記録します。結果はエンドポイントごと（ジョブやモデルごとではなく）に 5 分間キャッシュされるため、停止中のローカル Ollama/vLLM/SGLang/LM Studio サーバーを共有する多数の期限到来ジョブでも、リクエストの集中ではなく 1 回のプローブで済みます。事前確認でスキップされた実行は、実行エラーのバックオフ回数を増加させません。スキップ通知を繰り返し受け取るには、`failureAlert.includeSkipped` を設定します。

### コマンドペイロード

コマンドペイロードは、モデルを使用するターンを開始せずに、Gateway スケジューラー内で決定論的なスクリプトを実行します。Gateway ホスト上で実行され、標準出力／標準エラーをキャプチャし、Cron 履歴に実行を記録します。また、エージェントターンのジョブと同じ `announce`、`webhook`、`none` 配信モードを再利用します。

<Note>
コマンド Cron はオペレーター管理者向けの Gateway 自動化サーフェスであり、エージェントの `tools.exec` 呼び出しではありません。Cron ジョブの作成、更新、削除、手動実行には `operator.admin` が必要です。後でスケジュール実行されるコマンドは、その管理者が作成した自動化として Gateway プロセス内で実行されます。エージェントの exec ポリシー（`tools.exec.mode`、承認プロンプト、エージェント単位のツール許可リスト）は、モデルに公開される exec ツールを管理するものであり、コマンド Cron ペイロードには適用されません。
</Note>

```bash
openclaw cron create "*/15 * * * *" \
  --name "Queue depth probe" \
  --command "scripts/check-queue.sh" \
  --command-cwd "/srv/app" \
  --announce \
  --channel telegram \
  --to "-1001234567890"
```

`--command <shell>` は `argv: ["sh", "-lc", <shell>]` を保存します。シェル解析を行わず、正確な argv で実行するには `--command-argv '["node","scripts/report.mjs"]'` を使用します。オプションの `--command-env KEY=VALUE`（繰り返し指定可能）、`--command-input`、`--timeout-seconds`（デフォルトは 10 分）、`--no-output-timeout-seconds`、`--output-max-bytes` で、プロセス環境、標準入力、出力上限を制御します。

配信されるテキストはプロセス出力から生成されます。空でない標準出力が優先されます。標準出力が空で標準エラーが空でない場合は、標準エラーが配信されます。両方が存在する場合、Cron は小さな `stdout:`／`stderr:` ブロックを送信します。終了コード `0` は実行を `ok` として記録します。0 以外の終了、シグナル、タイムアウト、または無出力タイムアウトは `error` として記録され、失敗アラートをトリガーすることがあります。`NO_REPLY` のみを出力するコマンドでは、通常の Cron サイレントトークン抑制が適用され、チャットには何も投稿されません。

### スクリプトペイロード

スクリプトペイロードは、会話エージェントのターンを開始せずに、トリガースクリプトと同じコードモード実行環境でヘッドレスに実行されます。作成または実行する前に `cron.triggers.enabled` を有効にしてください。この危険な自動化に対するゲートは、トリガースクリプトとスクリプトペイロードの両方に適用されます。スクリプトジョブがサポートするセッションターゲットは `main` と `isolated` のみです。

```bash
openclaw cron create "0 * * * *" \
  --name "Hourly queue check" \
  --script ./automation/check-queue.js \
  --script-timeout-seconds 300 \
  --script-tool-budget 50 \
  --session isolated \
  --announce
```

ファイルまたは標準入力から JavaScript を読み取るには `--script <file|->` を使用します。タイムアウトのデフォルトは 300 秒、上限は 900 秒です。ツール予算のデフォルトは 50 回、上限は 200 回です。これらのペイロード予算は、より小さいトリガーゲート評価予算とは別です。

スクリプトは、次の任意フィールドを持つオブジェクトを返せます。

- `notify`: ジョブの `announce`、`webhook`、または `none` 配信モードを通じて配信されるテキストです。省略した場合は何も配信されません。`main` ジョブでは、テキストがシステムイベントになります。
- `wake`: `"now"` は、`notify`（または簡潔な完了イベント）をキューに追加した後、即時 Heartbeat を要求します。`"next-heartbeat"` は、次回の Heartbeat 用にイベントをキューに追加します。
- `state`: JSON 状態です。上限は 16 KB で、正常に実行された後にのみ永続化されます。次回の実行では、トリガースクリプトと同様に、凍結されたコピーが `trigger.state` として渡されます。この名前空間の永続化所有者は 1 つだけであるため、同じジョブでスクリプトペイロードと条件トリガーを組み合わせることはできません。
- `nextCheck`: `"15m"` などの期間です。ペーシングが有効なジョブでのみ有効で、エージェントターンの提案と同じペーシング制限を使用します。

例外、タイムアウト、ツール予算の枯渇、無効な結果、およびペーシングなしの `nextCheck` は、通常の Cron 実行エラーです。返された状態を永続化せず、実行履歴、バックオフ、失敗アラート処理の対象になります。

## 実行スタイル

| スタイル           | `--session` の値   | 実行場所                  | 最適な用途                        |
| --------------- | ------------------- | ------------------------ | ------------------------------- |
| メインセッション    | `main`              | 専用 Cron ウェイクレーン | リマインダー、システムイベント        |
| 分離        | `isolated`          | 専用 `cron:<jobId>` | レポート、バックグラウンド作業      |
| 現在のセッション | `current`           | 作成時にバインド   | コンテキストを考慮した反復作業    |
| カスタムセッション  | `session:custom-id` | 永続的な名前付きセッション | 履歴を基に構築するワークフロー |

<AccordionGroup>
  <Accordion title="メインセッション、分離、カスタムの違い">
    **メインセッション**ジョブは、Cron が所有する実行レーンにシステムイベントをキューに追加し、必要に応じて Heartbeat（`--wake now` または `--wake next-heartbeat`）をウェイクします。返信にはターゲットのメインセッションにおける最後の配信コンテキストを使用できますが、通常の Cron ターンを人間とのチャットレーンに追加せず、ターゲットセッションの日次リセットやアイドルリセットの鮮度も延長しません。**分離**ジョブは、新しいセッションで専用のエージェントターンを実行します。**カスタムセッション**（`session:xxx`）は実行間でコンテキストを保持するため、以前の要約を基に構築する日次スタンドアップなどのワークフローが可能になります。

    メインセッションの Cron イベントは、それ自体で完結したシステムイベントのリマインダーです。デフォルトの Heartbeat プロンプトや Heartbeat モニターのスクラッチは自動的には含まれません。リマインダーでそのコンテキストを参照する必要がある場合は、Cron イベントのテキストに明示してください。

  </Accordion>
  <Accordion title="分離ジョブにおける「新しいセッション」の意味">
    実行ごとに新しいトランスクリプト／セッション ID が作成されます。OpenClaw は安全な設定（思考／高速／詳細設定、ラベル、ユーザーが明示的に選択したモデル／認証のオーバーライド）を引き継ぎますが、以前の Cron 行から周辺の会話コンテキスト（チャンネル／グループルーティング、送信またはキューポリシー、権限昇格、オリジン、ACP ランタイムバインド）は継承しません。反復ジョブで意図的に同じ会話コンテキストを基に構築する必要がある場合は、`current` または `session:<id>` を使用します。
  </Accordion>
  <Accordion title="無人実行の契約">
    分離された Cron とフックのエージェントターンは、明示的に無人です。内容の確認や承認を行う人はいません。最終返信は、計画、確認応答、入力要求ではなく、成果物そのものでなければなりません。何もする必要がない場合、エージェントは `HEARTBEAT_OK` を返し、失敗は明確に記述します。再試行と失敗アラートのポリシーは Cron が管理します。

    信頼されたスケジュール済みジョブでは、質問や計画を意図的に求めるジョブ固有の指示が優先され、不要になったジョブをエージェントが削除することもできます。外部フックのターンに渡されるのは共通の無人実行契約のみです。外部コンテンツ境界を越えて、そのオーバーライドや自己削除に関するガイダンスが渡されることはありません。

  </Accordion>
  <Accordion title="サブエージェントと Discord 配信">
    分離された Cron 実行がサブエージェントをオーケストレーションする場合、配信では古い親の中間テキストよりも、最後の子孫の出力が優先されます。子孫がまだ実行中の場合、OpenClaw は親の部分的な更新をアナウンスせずに抑制します。

    テキストのみの Discord アナウンスターゲットでは、OpenClaw はストリーミング／中間テキストと最終回答の両方を再生する代わりに、正規の最終アシスタントテキストを 1 回送信します。メディアおよび構造化された Discord ペイロードは引き続き個別に配信されるため、添付ファイルやコンポーネントが失われることはありません。

  </Accordion>
</AccordionGroup>

## 配信と出力

| モード       | 動作                                                        |
| ---------- | ------------------------------------------------------------------- |
| `announce` | エージェントが送信しなかった場合、最終テキストをターゲットへフォールバック配信 |
| `webhook`  | 完了イベントのペイロードを URL に POST                                |
| `none`     | ランナーによるフォールバック配信なし                                         |

チャンネル配信には `--announce --channel telegram --to "-1001234567890"` を使用します。Telegram フォーラムトピックには `-1001234567890:topic:123` を使用します。OpenClaw は Telegram が所有する省略形 `-1001234567890:123` も受け付けます。RPC／設定の直接呼び出し元は、`delivery.threadId` を文字列または数値として渡せます。Slack／Discord／Mattermost のターゲットでは、明示的なプレフィックス（`channel:<id>`、`user:<id>`）を使用します。Matrix のルーム ID は大文字と小文字を区別します。Matrix から取得した正確なルーム ID または `room:!room:server` 形式を使用してください。

アナウンス配信で `channel: "last"` を使用するか、`channel` を省略すると、Cron がセッション履歴または設定済みの単一チャンネルにフォールバックする前に、`telegram:123` のようなプロバイダープレフィックス付きターゲットでチャンネルを選択できます。読み込まれた Plugin が通知するプレフィックスのみがプロバイダーセレクターです。`delivery.channel` を明示した場合、ターゲットプレフィックスは同じプロバイダーを指定しなければなりません。WhatsApp が Telegram ID を電話番号として解釈できるようにするのではなく、`channel: "whatsapp"` と `to: "telegram:123"` の組み合わせは拒否されます。ターゲット種別とサービスのプレフィックス（`channel:<id>`、`user:<id>`、`imessage:<handle>`、`sms:<number>`）は、プロバイダーセレクターではなく、引き続きチャンネルが所有するターゲット構文です。

分離ジョブでは、チャット配信は共有されます。チャットルートが利用可能な場合、`--no-deliver` であっても、エージェントは `message` ツールを使用できます。エージェントが設定済みまたは現在のターゲットへ送信すると、OpenClaw はフォールバックアナウンスをスキップします。それ以外の場合、`announce`、`webhook`、`none` が制御するのは、エージェントターン後にランナーが最終返信をどう処理するかだけです。

エージェントがアクティブなチャットから分離リマインダーを作成すると、OpenClaw は保持されたライブ配信ターゲットをフォールバックアナウンスのルートとして保存します。内部セッションキーは小文字の場合がありますが、現在のチャットコンテキストが利用可能な場合、プロバイダーの配信ターゲットがそれらのキーから再構築されることはありません。

暗黙のアナウンス配信では、設定済みのチャンネル許可リストを使用して、古いターゲットを検証し、ルーティングし直します。DM ペアリングストアの承認先は、フォールバック自動化の受信者にはなりません。スケジュール済みジョブから DM へ能動的に送信する必要がある場合は、`delivery.to` を設定するか、チャンネルの `allowFrom` エントリを設定します。

### 失敗通知

失敗通知は別の宛先パスに従います。

- `cron.failureDestination` は、失敗通知のグローバルデフォルトを設定します。
- `job.delivery.failureDestination` は、ジョブごとにこれをオーバーライドします。
- どちらも設定されておらず、ジョブがすでに `announce` 経由で配信している場合、失敗通知はその主要アナウンスターゲットにフォールバックします。
- `delivery.failureDestination` は、主要な配信モードが `webhook` でない限り、`sessionTarget="isolated"` ジョブでのみサポートされます。
- `failureAlert.includeSkipped: true` は、ジョブまたはグローバル Cron アラートポリシーで、繰り返し発生するスキップ実行のアラートを有効にします。スキップされた実行では、連続スキップ回数が別に記録されるため、実行エラーのバックオフには影響しません。
- `openclaw cron edit` は、ジョブごとのアラート調整項目として、`--failure-alert`/`--no-failure-alert`、`--failure-alert-after <n>`、`--failure-alert-channel`、`--failure-alert-to`、`--failure-alert-cooldown`、`--failure-alert-include-skipped`/`--failure-alert-exclude-skipped`、`--failure-alert-mode`、および `--failure-alert-account-id` を公開します。

### 出力言語

Cron ジョブは、チャンネル、ロケール、以前のメッセージから返信言語を推測しません。スケジュール済みのメッセージまたはテンプレートに言語規則を含めてください。

```bash
openclaw cron edit <jobId> \
  --message "更新内容を要約してください。中国語で回答し、URL、コード、製品名は変更しないでください。"
```

テンプレートファイルでは、レンダリングされるプロンプトに言語指示を含め、ジョブが実行される前に `{{language}}` などのプレースホルダーが埋められていることを確認します。出力で言語が混在する場合は、「叙述テキストには中国語を使用し、技術用語は英語のままにしてください」のように規則を明示します。

## CLI の例

<Tabs>
  <Tab title="単発リマインダー">
    ```bash
    openclaw cron add \
      --name "Calendar check" \
      --at "20m" \
      --session main \
      --system-event "Next heartbeat: check calendar." \
      --wake now
    ```
  </Tab>
  <Tab title="反復する分離ジョブ">
    ```bash
    openclaw cron create "0 7 * * *" \
      "Summarize overnight updates." \
      --name "Morning brief" \
      --tz "America/Los_Angeles" \
      --session isolated \
      --announce \
      --channel slack \
      --to "channel:C1234567890"
    ```
  </Tab>
  <Tab title="モデルと思考のオーバーライド">
    ```bash
    openclaw cron add \
      --name "Deep analysis" \
      --cron "0 6 * * 1" \
      --tz "America/Los_Angeles" \
      --session isolated \
      --message "Weekly deep analysis of project progress." \
      --model "opus" \
      --thinking high \
      --announce
    ```
  </Tab>
  <Tab title="Webhook 出力">
    ```bash
    openclaw cron create "0 18 * * 1-5" \
      "Summarize today's deploys as JSON." \
      --name "Deploy digest" \
      --webhook "https://example.invalid/openclaw/cron"
    ```
  </Tab>
  <Tab title="コマンド出力">
    ```bash
    openclaw cron create "*/15 * * * *" \
      --name "Queue depth probe" \
      --command "scripts/check-queue.sh" \
      --command-cwd "/srv/app" \
      --announce \
      --channel telegram \
      --to "-1001234567890"
    ```
  </Tab>
</Tabs>

## ジョブの管理

```bash
# 有効なジョブを一覧表示
openclaw cron list

# 無効なジョブも含める
openclaw cron list --all

# 保存済みのジョブを1件JSONとして取得
openclaw cron get <jobId>

# 解決済みの配信ルートを含め、ジョブを1件表示
openclaw cron show <jobId>

# 削除せずに有効化／無効化
openclaw cron enable <jobId>
openclaw cron disable <jobId>

# ジョブを編集
openclaw cron edit <jobId> --message "更新されたプロンプト" --model "opus"

# ジョブを今すぐ強制実行
openclaw cron run <jobId>

# ジョブを今すぐ強制実行し、最終ステータスまで待機
openclaw cron run <jobId> --wait --wait-timeout 10m --poll-interval 2s

# 実行期限に達している場合のみ実行
openclaw cron run <jobId> --due

# 実行履歴を表示
openclaw cron runs --id <jobId> --limit 50

# 特定の実行を1件表示
openclaw cron runs --id <jobId> --run-id <runId>

# ジョブを削除
openclaw cron remove <jobId>

# エージェントの選択（マルチエージェント構成）
openclaw cron create "0 6 * * *" "運用キューを確認" --name "運用確認" --session isolated --agent ops
openclaw cron edit <jobId> --clear-agent
```

セッションをアーカイブすると（Control UI、または operator-admin 呼び出し元からの `sessions.patch { archived: true }`）、そのセッションに関連付けられている有効な cron ジョブがすべて無効になります。対象は、そのセッションの isolated `cron:<jobId>` セッション、`session:<key>` ターゲット、または配信／ウェイク用の `sessionKey` レーンです。セッションを復元しても、それらのジョブは再度有効になりません。`openclaw cron enable <jobId>` を使用してください。有効な関連付け済みジョブがあるセッションには、Control UI のサイドバーに時計バッジが表示されます。

`openclaw cron run <jobId>` は手動実行をキューに追加した後に返ります。シャットダウンフック、メンテナンススクリプト、またはキューに入った実行が完了するまでブロックする必要があるその他の自動処理では、`--wait` を使用してください。返された `runId` をポーリングし（デフォルトのタイムアウトは `10m`、ポーリング間隔は `2s`）、ステータスが `ok` の場合は `0` で終了し、`error`、`skipped`、または待機タイムアウトの場合はゼロ以外で終了します。

エージェントの `cron` ツールは、`cron(action: "list")` から簡潔なジョブ概要（`id`、`name`、`enabled`、`nextRunAtMs`、`scheduleKind`、`lastRunStatus`）を返します。完全なジョブ定義を1件取得するには `cron(action: "get", jobId: "...")` を使用してください。Gateway を直接呼び出す場合は、`cron.list` に `compact: true` を渡せます。省略すると、配信プレビューを含む完全なレスポンスが維持されます。

`openclaw cron create` は `openclaw cron add` のエイリアスです。新しいジョブでは、位置引数のスケジュール（`"0 9 * * 1"`、`"every 1h"`、`"20m"`、または ISO タイムスタンプ）の後に、位置引数のエージェントプロンプトを指定できます。完了した実行ペイロードを HTTP エンドポイントへ POST するには、`cron add|create` または `cron edit` で `--webhook <url>` を使用してください。Webhook 配信とチャット配信フラグ（`--announce`、`--channel`、`--to`、`--thread-id`、`--account`）は併用できません。`cron edit`、`--clear-channel`、`--clear-to`、`--clear-thread-id`、`--clear-account` は、それらのルーティングフィールドを個別に解除します（それぞれ対応する設定フラグとの併用は拒否されます）。これは、ランナーのフォールバック配信のみを無効にする `--no-deliver` とは異なります。

<Note>
モデルオーバーライドに関する注意事項：

- `openclaw cron add|edit --model ...` はジョブで選択されるモデルを変更します。
- モデルが許可されている場合、指定したプロバイダー／モデルがそのまま isolated エージェント実行に渡されます。
- 許可されていない場合、または解決できない場合、cron は明示的な検証エラーで実行を失敗させます。
- API の `cron.update` ペイロードパッチでは、`model: null` を設定して、保存済みジョブのモデルオーバーライドを消去できます。
- `openclaw cron edit <job-id> --clear-model` は CLI からそのオーバーライドを消去し（`model: null` パッチと同じ効果）、`--model` とは併用できません。
- cron の `--model` はジョブのプライマリであり、セッションの `/model` オーバーライドではないため、設定済みのフォールバックチェーンは引き続き適用されます。
- `openclaw cron add|edit --fallbacks ...` はペイロードの `fallbacks` を設定し、そのジョブに設定されたフォールバックを置き換えます。`--fallbacks ""` はフォールバックを無効にして実行を厳格化します。`openclaw cron edit <job-id> --clear-fallbacks` はジョブごとのオーバーライドを消去します。
- 明示的または設定済みのフォールバックリストがない単独の `--model` は、暗黙の追加再試行先としてエージェントのプライマリにフォールスルーしません。

</Note>

## Webhook

Gateway は、外部トリガー用の HTTP Webhook エンドポイントを公開できます。設定で有効にします：

```json5
{
  hooks: {
    enabled: true,
    token: "shared-secret",
    path: "/hooks",
  },
}
```

### 認証

すべてのリクエストで、次のいずれかのヘッダーを介してフックトークンを含める必要があります：

- `Authorization: Bearer <token>`（推奨）
- `x-openclaw-token: <token>`

クエリ文字列のトークンは拒否されます。

<AccordionGroup>
  <Accordion title="POST /hooks/wake">
    メインセッションのシステムイベントをキューに追加します：

    ```bash
    curl -X POST http://127.0.0.1:18789/hooks/wake \
      -H 'Authorization: Bearer SECRET' \
      -H 'Content-Type: application/json' \
      -d '{"text":"新しいメールを受信しました","mode":"now"}'
    ```

    <ParamField path="text" type="string" required>
      イベントの説明。
    </ParamField>
    <ParamField path="mode" type="string" default="now">
      `now` または `next-heartbeat`。
    </ParamField>

  </Accordion>
  <Accordion title="POST /hooks/agent">
    isolated エージェントターンを実行します：

    ```bash
    curl -X POST http://127.0.0.1:18789/hooks/agent \
      -H 'Authorization: Bearer SECRET' \
      -H 'Content-Type: application/json' \
      -d '{"message":"受信トレイを要約","name":"メール","model":"openai/gpt-5.6-sol"}'
    ```

    フィールド：`message`（必須）、`name`、`agentId`、`sessionKey`（`hooks.allowRequestSessionKey=true` が必要）、`idempotencyKey`、`wakeMode`、`deliver`、`channel`、`to`、`model`、`thinking`、`timeoutSeconds`。

  </Accordion>
  <Accordion title="マッピングされたフック（POST /hooks/<name>）">
    カスタムフック名は、設定内の `hooks.mappings` を介して解決されます。マッピングでは、テンプレートまたはコード変換を使用して、任意のペイロードを `wake` または `agent` アクションに変換できます。
  </Accordion>
</AccordionGroup>

<Warning>
フックエンドポイントは、ループバック、tailnet、または信頼できるリバースプロキシの背後に配置してください。

- 専用のフックトークンを使用し、Gateway の認証トークンを再利用しないでください。
- `hooks.path` は専用のサブパスに設定してください。`/` は拒否されます。
- フックが対象にできる実効エージェントを制限するには、`hooks.allowedAgentIds` を設定してください。これには、`agentId` が省略された場合のデフォルトエージェントも含まれます。
- 呼び出し元によるセッション選択が必要な場合を除き、`hooks.allowRequestSessionKey=false` を維持してください。
- `hooks.allowRequestSessionKey` を有効にする場合は、許可するセッションキーの形式を制限するために `hooks.allowedSessionKeyPrefixes` も設定してください。
- フックペイロードは、デフォルトで安全境界によってラップされます。

</Warning>

## Gmail PubSub 連携

Google PubSub を介して Gmail の受信トリガーを OpenClaw に接続します。

<Note>
**前提条件：** `gcloud` CLI、`gog`（gogcli）、有効化済みの OpenClaw フック、公開 HTTPS エンドポイント用の Tailscale。
</Note>

### ウィザードによるセットアップ（推奨）

```bash
openclaw webhooks gmail setup --account openclaw@gmail.com
```

これにより `hooks.gmail` 設定が書き込まれ、Gmail プリセットが有効になり、プッシュエンドポイントのデフォルトとして Tailscale Funnel（`--tailscale funnel|serve|off`）が使用されます。

<Warning>
Gmail プリセットのメッセージごとのセッションは会話コンテキストを分離しますが、対象エージェントのツールやワークスペースを制限するものではありません。`agentId` を設定するカスタムマッピングがない場合、Gmail フックはデフォルトエージェントとして実行されます。

信頼できない受信トレイでは、フックを専用のリーダーエージェントにルーティングし、そのエージェントには読み取り専用またはワークスペースへのアクセス権なしを設定し、ファイルシステムへの書き込み、シェル、ブラウザー、およびその他の不要なツールを拒否してください。メインエージェントへの通知が必要な場合は、必要なエージェント間ハンドオフのみを許可してください。[プロンプトインジェクション](/ja-JP/gateway/security#prompt-injection)、[マルチエージェントのサンドボックスとツール](/ja-JP/tools/multi-agent-sandbox-tools)、[`tools.agentToAgent`](/ja-JP/gateway/config-tools#toolsagenttoagent)を参照してください。
</Warning>

### Gateway の自動起動

`hooks.enabled=true` および `hooks.gmail.account` が設定されている場合、Gateway は起動時に `gog gmail watch serve` を開始し、監視を自動更新します。オプトアウトするには `OPENCLAW_SKIP_GMAIL_WATCHER=1` を設定してください。

### 手動による1回限りのセットアップ

<Steps>
  <Step title="GCP プロジェクトを選択">
    `gog` が使用する OAuth クライアントを所有する GCP プロジェクトを選択します：

    ```bash
    gcloud auth login
    gcloud config set project <project-id>
    gcloud services enable gmail.googleapis.com pubsub.googleapis.com
    ```

  </Step>
  <Step title="トピックを作成し、Gmail のプッシュアクセスを許可">
    ```bash
    gcloud pubsub topics create gog-gmail-watch
    gcloud pubsub topics add-iam-policy-binding gog-gmail-watch \
      --member=serviceAccount:gmail-api-push@system.gserviceaccount.com \
      --role=roles/pubsub.publisher
    ```
  </Step>
  <Step title="監視を開始">
    ```bash
    gog gmail watch start \
      --account openclaw@gmail.com \
      --label INBOX \
      --topic projects/<project-id>/topics/gog-gmail-watch
    ```
  </Step>
</Steps>

### Gmail のモデルオーバーライド

```json5
{
  hooks: {
    gmail: {
      model: "openai/gpt-5.6-sol",
      thinking: "high",
    },
  },
}
```

信頼できない受信トレイには、プロバイダーで利用可能な最新世代かつ最上位のモデルを使用してください。上記の値は例です。モデルは設定済みのカタログと許可リストに存在する必要があります。

## 設定

```json5
{
  cron: {
    enabled: true,
    store: "~/.openclaw/cron/jobs.json",
    triggers: {
      enabled: false,
    },
    webhookToken: "replace-with-dedicated-webhook-token",
    sessionRetention: "24h",
  },
}
```

`webhookToken` は、cron の Webhook POST で `Authorization: Bearer <token>` として送信されます。

`cron.store` は論理ストアキーおよび doctor の移行パスであり、手作業で編集する実体のある JSON ファイルではありません。ジョブデータは SQLite に保存されます。変更には CLI または Gateway API を使用してください。

cron を無効にするには、`cron.enabled: false` または `OPENCLAW_SKIP_CRON=1` を使用します。

<AccordionGroup>
  <Accordion title="再試行動作">
    **単発ジョブの再試行**：一時的なエラー（レート制限、過負荷、ネットワーク、タイムアウト、サーバーエラー）では、組み込みの再試行スケジュールが使用されます。永続的なエラーでは、ジョブが直ちに無効になります。

    **定期ジョブの再試行**：連続する実行エラーでは、延長されたスケジュール（30s、60s、5m、15m、60m）でバックオフします。次回の実行が成功すると、バックオフはリセットされます。

  </Accordion>
  <Accordion title="メンテナンス">
    `cron.sessionRetention`（デフォルトは `24h`、`false` で無効化）は、isolated 実行セッションのエントリを削除します。実行履歴では、ジョブごとに最新の最終状態行を2000件保持します。消失した行には、24時間のクリーンアップ期間が維持されます。
  </Accordion>
  <Accordion title="従来のストアの移行">
    アップグレード時に `openclaw doctor --fix` を実行すると、従来の `~/.openclaw/cron/jobs.json`、`jobs-state.json`、`runs/*.jsonl` ファイルが SQLite にインポートされ、`.migrated` サフィックスを付けて名前が変更されます。不正なジョブ行はランタイムではスキップされ、後で修復または確認できるように `jobs-quarantine.json` にコピーされます。
  </Accordion>
</AccordionGroup>

## トラブルシューティング

### コマンドの確認手順

```bash
openclaw status
openclaw gateway status
openclaw cron status
openclaw cron list
openclaw cron runs --id <jobId> --limit 20
openclaw system heartbeat last
openclaw logs --follow
openclaw doctor
```

<AccordionGroup>
  <Accordion title="Cron が実行されない">
    - `cron.enabled` と `OPENCLAW_SKIP_CRON` 環境変数を確認してください。
    - Gateway が継続的に実行されていることを確認してください。
    - `cron` スケジュールでは、タイムゾーン（`--tz`）とホストのタイムゾーンを確認してください。
    - 実行出力に `reason: not-due` と表示される場合、手動実行が `openclaw cron run <jobId> --due` を使用して確認され、ジョブの実行期限にまだ達していなかったことを意味します。

  </Accordion>
  <Accordion title="Cron は実行されたが配信されない">
    - 配信モード `none` では、ランナーによるフォールバック送信は行われません。チャットルートが利用可能な場合、エージェントは `message` ツールを使用して直接送信できます。
    - 配信先がない、または無効（`channel`/`to`）な場合、送信はスキップされます。
    - Matrix では、小文字化された `delivery.to` ルーム ID を含むコピー済みまたはレガシーのジョブは、Matrix のルーム ID で大文字と小文字が区別されるため失敗することがあります。ジョブを編集し、Matrix から取得した正確な `!room:server` または `room:!room:server` の値を指定してください。
    - チャンネル認証エラー（`unauthorized`、`Forbidden`）は、認証情報によって配信がブロックされたことを意味します。
    - 分離実行がサイレントトークン（`NO_REPLY` / `no_reply`）のみを返す場合、OpenClaw は直接の外部配信とフォールバックのキュー済み要約パスを抑制するため、チャットには何も投稿されません。
    - エージェント自身がユーザーにメッセージを送信する必要がある場合は、ジョブに利用可能なルート（以前のチャットを伴う `channel: "last"`、または明示的なチャンネル／配信先）があることを確認してください。

  </Accordion>
  <Accordion title="Cron または Heartbeat によって /new 形式のロールオーバーが妨げられているように見える">
    - 日次リセットとアイドルリセットの経過時間は `updatedAt` に基づきません。[セッション管理](/ja-JP/concepts/session#session-lifecycle)を参照してください。
    - Cron のウェイクアップ、Heartbeat の実行、exec 通知、Gateway の記録処理によって、ルーティングやステータス用のセッション行が更新される場合がありますが、`sessionStartedAt` または `lastInteractionAt` が延長されることはありません。
    - これらのフィールドが存在する前に作成されたレガシー行では、ファイルがまだ利用可能な場合、OpenClaw はトランスクリプト JSONL のセッションヘッダーから `sessionStartedAt` を復元できます。`lastInteractionAt` のないレガシーのアイドル行では、その復元された開始時刻がアイドル時間の基準として使用されます。

  </Accordion>
  <Accordion title="タイムゾーンに関する注意点">
    - `--tz` を指定しない Cron では、Gateway ホストのタイムゾーンが使用されます。
    - タイムゾーンを指定しない `at` スケジュールは UTC として扱われます。
    - Heartbeat の `activeHours` では、設定されたタイムゾーン解決が使用されます。

  </Accordion>
</AccordionGroup>

## 関連項目

- [自動化](/ja-JP/automation) — すべての自動化メカニズムの概要
- [バックグラウンドタスク](/ja-JP/automation/tasks) — Cron 実行のタスク台帳
- [Heartbeat](/ja-JP/gateway/heartbeat) — メインセッションの定期ターン
- [タイムゾーン](/ja-JP/concepts/timezone) — タイムゾーンの設定
