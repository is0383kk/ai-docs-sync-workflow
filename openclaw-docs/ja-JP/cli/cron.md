---
read_when:
    - スケジュールされたジョブとウェイクアップが必要な場合
    - Cron の実行とログをデバッグしています
summary: '`openclaw cron` の CLI リファレンス（バックグラウンドジョブのスケジュール設定と実行）'
title: Cron
x-i18n:
    generated_at: "2026-07-26T08:56:13Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 5989a7558f4ae2f046480b6a52e3fa296c95d47b14b11c5bad709fea4af6af3e
    source_path: cli/cron.md
    workflow: 16
---

# `openclaw cron`

Gateway スケジューラの Cron ジョブを管理します。

<Tip>
コマンド全体については `openclaw cron --help` を実行してください。概念ガイドについては [Cron ジョブ](/ja-JP/automation/cron-jobs)を参照してください。
</Tip>

<Note>
Cron の変更操作（`add`/`create`、`update`/`edit`、`remove`、`run`）には、すべて `operator.admin` が必要です。コマンドペイロードの実行は、エージェントの `tools.exec` ツール呼び出しとしてではなく、Gateway プロセス内で直接実行されます。モデルから見える exec ツールには、引き続き `tools.exec.*` と exec 承認が適用されます。
</Note>

## ジョブをすばやく作成する

`openclaw cron create` は `openclaw cron add` のエイリアスです。新しいジョブでは、スケジュールを先に、プロンプトを後に指定します。

```bash
openclaw cron create "0 7 * * *" \
  "夜間の更新を要約する。" \
  --name "朝の概要" \
  --agent ops
```

チャットの送信先に配信する代わりに、完了したペイロードを POST する場合は `--webhook <url>` を使用します。

```bash
openclaw cron create "0 18 * * 1-5" \
  "今日のデプロイを JSON で要約する。" \
  --name "デプロイダイジェスト" \
  --webhook "https://example.invalid/openclaw/cron"
```

分離されたエージェント／モデル実行を開始せず、OpenClaw Cron 内で実行する決定論的なシェル形式のジョブには `--command` を使用します。

```bash
openclaw cron create "*/15 * * * *" \
  --name "キュー深度の調査" \
  --command "scripts/check-queue.sh" \
  --command-cwd "/srv/app" \
  --announce \
  --channel telegram \
  --to "-1001234567890"
```

`--command <shell>` は `argv: ["sh", "-lc", <shell>]` を保存します。正確な argv 実行には `--command-argv '["node","scripts/report.mjs"]'` を使用します。コマンドジョブは stdout／stderr を取得し、通常の Cron 履歴を記録して、分離ジョブと同じ `announce`、`webhook`、または `none` の配信モードで出力をルーティングします。`NO_REPLY` だけを出力するコマンドは抑制されます。

## セッション

`--session` には `main`、`isolated`、`current`、または `session:<id>` を指定できます。

<AccordionGroup>
  <Accordion title="セッションキー">
    - `main` はエージェントのメインセッションにバインドします。
    - `isolated` は実行ごとに新しいトランスクリプトとセッション ID を作成します。
    - `current` は作成時のアクティブセッションにバインドします。
    - `session:<id>` は明示的な永続セッションキーに固定します。

  </Accordion>
  <Accordion title="分離セッションのセマンティクス">
    分離実行では、周囲の会話コンテキストがリセットされます。新しい実行では、チャネルとグループのルーティング、送信／キューポリシー、昇格、オリジン、ACP ランタイムのバインドがリセットされます。安全な設定と、ユーザーが明示的に選択したモデルまたは認証のオーバーライドは、実行間で引き継ぐことができます。
  </Accordion>
</AccordionGroup>

## 配信

`openclaw cron list` と `openclaw cron show <job-id>` は、解決された配信ルートをプレビューします。`channel: "last"` の場合、プレビューには、ルートがメインセッションと現在のセッションのどちらから解決されたか、または安全側に失敗するかが表示されます。

プロバイダーのプレフィックスが付いた送信先を使用すると、未解決の通知チャネルを明確に指定できます。たとえば、`delivery.channel` が省略されているか `last` の場合、`to: "telegram:123"` は Telegram を選択します。読み込まれた Plugin が公開しているプレフィックスだけが、プロバイダーセレクターとして機能します。`delivery.channel` が明示されている場合、プレフィックスはそのチャネルと一致する必要があります。`to: "telegram:123"` と組み合わせた `channel: "whatsapp"` は拒否されます。`imessage:` や `sms:` などのサービスプレフィックスは、引き続きチャネルが所有する送信先構文です。

<Note>
分離された `cron add` ジョブでは、デフォルトで `--announce` 配信が使用されます。出力を内部に保持するには `--no-deliver` を使用します。`--deliver` は、非推奨の `--announce` のエイリアスとして残されています。
</Note>

### 配信の所有権

分離された Cron のチャット配信は、エージェントとランナーが共同で担当します。

- チャットルートが利用可能な場合、エージェントは `message` ツールを使用して直接送信できます。
- `announce` は、エージェントが解決済みの送信先に直接送信しなかった場合に限り、最終応答をフォールバック配信します。
- `webhook` は、完了したペイロードを URL に送信します。
- `none` は、ランナーによるフォールバック配信を無効にします。

Webhook 配信を設定するには、`cron add|create --webhook <url>` または `cron edit <job-id> --webhook <url>` を使用します。`--webhook` を、`--announce`、`--no-deliver`、`--channel`、`--to`、`--thread-id`、`--account` などのチャット配信フラグと組み合わせないでください。

`cron edit <job-id>` では、`--clear-channel`、`--clear-to`、`--clear-thread-id`、`--clear-account` を使って配信ルーティングの各フィールドを個別に設定解除できます（それぞれ、対応する設定フラグと組み合わせると拒否されます）。ランナーによるフォールバック配信を無効にするだけの `--no-deliver` とは異なり、これらは保存済みフィールドを削除するため、ジョブはルートの該当部分を再びデフォルトから解決します。

`--announce` は、最終応答に対するランナーのフォールバック配信です。`--no-deliver` はそのフォールバックを無効にしますが、チャットルートが利用可能な場合にエージェントの `message` ツールを削除することはありません。

アクティブなチャットから作成されたリマインダーは、フォールバック通知配信用に、現在のチャットの配信先を保持します。内部セッションキーは小文字の場合があります。Matrix のルーム ID など、大文字と小文字を区別するプロバイダー ID の信頼できる情報源として使用しないでください。

### 失敗時の配信

失敗通知は次の順序で解決されます。

1. ジョブの `delivery.failureDestination`。
2. グローバルな `cron.failureDestination`。
3. ジョブの主要通知先（上記のいずれも具体的な送信先に解決されない場合）。

<Note>
メインセッションのジョブで `delivery.failureDestination` を使用できるのは、主要配信モードが `webhook` の場合だけです。分離ジョブでは、すべてのモードで使用できます。
</Note>

分離された Cron 実行では、応答ペイロードが生成されなかった場合でも、実行レベルのエージェント障害をジョブエラーとして扱います。そのため、モデル／プロバイダーの障害でもエラーカウンターが増加し、失敗通知がトリガーされます。

コマンド Cron ジョブは、分離されたエージェントターンを開始しません。終了コードがゼロの場合は `ok` を記録します。ゼロ以外の終了、シグナル、タイムアウト、または無出力タイムアウトの場合は `error` を記録し、同じ失敗通知経路をトリガーできます。

分離実行が最初のモデルリクエストより前にタイムアウトした場合、`openclaw cron show` と `openclaw cron runs` には、`setup timed out before runner start` などのフェーズ固有のエラー、または最後に判明していた起動フェーズを示す停止メッセージ（例: `context-engine`）が含まれます。CLI ベースのプロバイダーでは、外部 CLI のターンが開始するまでモデル実行前のウォッチドッグが動作し続けるため、セッション検索、フック、認証、プロンプト、CLI セットアップの停止は、モデル実行前の Cron 障害として報告されます。

## スケジューリング

### 1 回限りのジョブ

`--at <datetime>` は 1 回限りの実行をスケジュールします。オフセットのない日時は UTC として扱われますが、`--tz <iana>` も渡した場合は、指定されたタイムゾーンの現地時刻として解釈されます。

<Note>
1 回限りのジョブは、デフォルトでは成功後に削除されます。保持するには `--keep-after-run` を使用します。
</Note>

### 定期ジョブ

定期ジョブでは、連続してエラーが発生すると、30s、1m、5m、15m、60m の指数バックオフで再試行します。次回の実行が成功すると、通常のスケジュールに戻ります。

スキップされた実行は、実行エラーとは別に追跡されます。再試行のバックオフには影響しませんが、`openclaw cron edit <job-id> --failure-alert-include-skipped` を使用すると、失敗アラートでスキップの繰り返しを通知できます。

ローカルに設定されたモデルプロバイダー（ベース URL がループバック、プライベートネットワーク、または `.local`）を対象とする分離ジョブでは、Cron はエージェントターンを開始する前に軽量なプロバイダー事前チェックを実行します。`api: "ollama"` プロバイダーは `/api/tags` で、その他のローカルな OpenAI 互換プロバイダー（`api: "openai-completions"`。例: vLLM、SGLang、LM Studio）は `/models` でプローブされます。エンドポイントに到達できない場合、その実行は `skipped` として記録され、後続のスケジュールで再試行されます。同じローカルサーバーを対象とする多数のジョブが繰り返しプローブを実行しないよう、到達可能性の結果はエンドポイントごとに 5 分間キャッシュされます。

Cron ジョブ、保留中のランタイム状態、実行履歴は、共有 SQLite 状態データベースに保存されます。従来の `jobs.json`、`<name>-state.json`、`runs/*.jsonl` ファイルは一度だけインポートされ、`.migrated` サフィックスを付けて名前が変更されます。インポート後は、JSON ファイルを編集する代わりに `openclaw cron add|edit|remove` でスケジュールを編集してください。

### 手動実行

`openclaw cron run <job-id>` はデフォルトで強制実行し、手動実行がキューに追加されるとすぐに返ります。成功時の応答には `{ ok: true, enqueued: true, runId }` が含まれます。返された `runId` を使用して、後から結果を確認します。

```bash
openclaw cron run <job-id>
openclaw cron runs --id <job-id> --run-id <run-id>
```

スクリプトで、そのキューに追加された実行が終了状態を記録するまでブロックする必要がある場合は、`--wait` を追加します。

```bash
openclaw cron run <job-id> --wait --wait-timeout 10m --poll-interval 2s
```

`--wait` を指定した場合でも、CLI は最初に `cron.run` を呼び出し、返された `runId` について `cron.runs` をポーリングします。実行がステータス `ok` で完了した場合に限り、コマンドは `0` で終了します。実行が `error` または `skipped` で完了した場合、Gateway の応答に `runId` が含まれていない場合、または `--wait-timeout` が期限切れになった場合（デフォルトは `10m`、デフォルトでは `2s` ごとにポーリング）、ゼロ以外で終了します。`--poll-interval` はゼロより大きい値でなければなりません。

<Note>
ジョブが現在実行予定の場合に限り手動コマンドを実行するには、`--due` を使用します。`--due --wait` が実行をキューに追加しなかった場合、コマンドはポーリングせず、通常の未実行応答を返します。
</Note>

## モデル

`cron add|edit --model <ref>` は、ジョブで許可されたモデルを選択します。`cron add|edit --fallbacks <list>` はジョブごとのフォールバックモデルを設定します（例: `--fallbacks openrouter/gpt-4.1-mini,openai/gpt-5`）。フォールバックを使用しない厳密な実行には `--fallbacks ""` を渡します。`cron edit <job-id> --clear-fallbacks` は、ジョブごとのフォールバックのオーバーライドを削除します。`cron edit <job-id> --clear-model` はジョブごとのモデルオーバーライドを削除し、ジョブが通常の Cron モデル選択の優先順位（保存済みの Cron セッションオーバーライドが存在する場合はそれを使用し、それ以外はエージェント／デフォルトモデルを使用）に従うようにします。`--model` と組み合わせることはできません。`cron add|edit --thinking <level>` はジョブごとの思考オーバーライドを設定します。`cron edit <job-id> --clear-thinking` はそれを削除し、ジョブが通常の Cron 思考設定の優先順位に従うようにします。`--thinking` と組み合わせることはできません。

<Warning>
モデルが許可されていないか解決できない場合、Cron はジョブのエージェントまたはデフォルトのモデル選択にフォールバックせず、明示的な検証エラーで実行を失敗させます。
</Warning>

Cron の `--model` はチャットセッションの `/model` オーバーライドではなく、**ジョブのプライマリ**です。つまり、次のようになります。

- 選択されたジョブモデルが失敗した場合でも、設定済みのモデルフォールバックが適用されます。
- ジョブごとのペイロードの `fallbacks` が存在する場合、設定済みのフォールバックリストを置き換えます。
- ジョブごとのフォールバックリストが空の場合（ジョブペイロード／API の `--fallbacks ""` または `fallbacks: []`）、Cron 実行は厳密になります。
- ジョブに `--model` が設定されているものの、フォールバックリストが設定されていない場合、OpenClaw は明示的に空のフォールバックオーバーライドを渡すため、エージェントのプライマリが非表示の再試行先として追加されることはありません。
- ローカルプロバイダーの事前チェックでは、Cron 実行を `skipped` として記録する前に、設定済みのフォールバックを順番に確認します。

`openclaw doctor` は、`payload.model` がすでに設定されているジョブについて、プロバイダー名前空間ごとの件数と `agents.defaults.model` との不一致を含めて報告します。ライブチャットとスケジュール済みジョブで認証、プロバイダー、または課金の動作が異なるように見える場合は、このチェックを使用してください。

### 分離 Cron のモデル優先順位

分離 Cron は、次の順序でアクティブなモデルを解決します。

1. Gmail フックのオーバーライド。
2. ジョブごとの `--model`。
3. 保存済みの Cron セッションモデルオーバーライド（ユーザーが選択した場合）。
4. エージェントまたはデフォルトのモデル選択。

### 高速モード

分離 cron の高速モードは、解決されたライブモデル選択に従います。モデル設定 `params.fastMode` がデフォルトで適用されますが、保存済みセッションの `fastMode` オーバーライドは引き続き設定より優先されます。解決されたモードが `auto` の場合、カットオフには選択されたモデルの `params.fastAutoOnSeconds` 値が使用され、デフォルトは 60 秒です。

### ライブモデル切り替え時の再試行

分離実行が `LiveSessionModelSwitchError` をスローした場合、cron は再試行前に、切り替え後のプロバイダーとモデル（切り替え後の認証プロファイルオーバーライドがある場合はそれも）をアクティブな実行に対して永続化します。外側の再試行ループは、最初の試行後に切り替え再試行を 2 回までに制限し、その後は無限ループせずに中止します。

## 実行出力と拒否

### 古い確認応答の抑制

分離 cron ターンでは、古い確認応答のみの返信を抑制します。最初の結果が単なる中間ステータス更新であり、最終的な回答を担当する子孫サブエージェント実行がない場合、cron は配信前に実際の結果を求めて 1 回だけ再プロンプトします。

### サイレントトークンの抑制

分離 cron 実行がサイレントトークン（`NO_REPLY` または `no_reply`）のみを返した場合、cron は直接の外向き配信とフォールバックのキュー済み要約経路の両方を抑制するため、チャットには何も投稿されません。

### 構造化された拒否

分離 cron 実行は、埋め込み実行からの構造化された実行拒否メタデータ（コードが `SYSTEM_RUN_DENIED` または `INVALID_REQUEST` の致命的な exec ツールエラー）を正式な拒否シグナルとして使用します。また、これらのコードのいずれかを含むネストされた構造化エラーをラップする Node ホストの `UNAVAILABLE` ラッパーも認識します。

埋め込み実行が構造化された拒否メタデータも提供しない限り、cron は最終出力の文章や承認を求めているように見える拒否表現を拒否として分類しません。そのため、通常のアシスタントテキストがブロックされたコマンドとして扱われることはありません。

`cron list` と実行履歴は、ブロックされたコマンドを `ok` として報告する代わりに、拒否理由を表示します。

## 保持

保持動作：

- `cron.sessionRetention`（デフォルトは `24h`、無効化する場合は `false`）は、完了した分離実行セッションを削除します。
- 実行履歴は、cron ジョブごとに最新のターミナル行を 2000 件保持します。消失した行には、標準の 24 時間の消失タスククリーンアップ期間が適用されます。

## 古いジョブの移行

<Note>
現在の配信および保存形式より前の cron ジョブがある場合は、`openclaw doctor --fix` を実行してください。Doctor は従来の cron フィールド（`jobId`、`schedule.cron`、従来の `threadId` を含むトップレベルの配信フィールド、ペイロードの `provider` 配信エイリアス）を正規化し、廃止された生の `cron.webhook` 値を使用している `notify: true` Webhook フォールバックジョブを、該当する設定キーを削除する前に明示的な Webhook 配信へ移行します。すでにチャットへ通知するジョブはその配信を維持し、完了 Webhook の送信先が追加されます。従来の Webhook がない場合、移行先がないジョブから不活性なトップレベルの `notify` マーカーが削除され（既存の配信は変更されずに維持されます）、`doctor --fix` がそれらについて警告を繰り返すことはなくなります。
</Note>

## よく使う編集操作

メッセージを変更せずに配信設定を更新する：

```bash
openclaw cron edit <job-id> --announce --channel telegram --to "123456789"
```

分離ジョブの配信を無効にする：

```bash
openclaw cron edit <job-id> --no-deliver
```

分離ジョブで軽量ブートストラップコンテキストを有効にする：

```bash
openclaw cron edit <job-id> --light-context
```

特定のチャンネルに通知する：

```bash
openclaw cron edit <job-id> --announce --channel slack --to "channel:C1234567890"
```

Telegram フォーラムのトピックに通知する：

```bash
openclaw cron edit <job-id> --announce --channel telegram --to "-1001234567890" --thread-id 42
```

軽量ブートストラップコンテキストを使用する分離ジョブを作成する：

```bash
openclaw cron create "0 7 * * *" \
  "夜間の更新を要約する。" \
  --name "軽量な朝の概要" \
  --session isolated \
  --light-context \
  --no-deliver
```

`--light-context` は分離エージェントターンジョブにのみ適用されます。cron 実行では、軽量モードはワークスペースの完全なブートストラップセットを注入せず、ブートストラップコンテキストを空のままにします。

正確な argv、cwd、env、stdin、出力制限を指定してコマンドジョブを作成する：

```bash
openclaw cron create "*/30 * * * *" \
  --name "ポジションのエクスポート" \
  --command-argv '["node","scripts/export-position.mjs"]' \
  --command-cwd "/srv/app" \
  --command-env "NODE_ENV=production" \
  --command-input '{"mode":"summary"}' \
  --timeout-seconds 120 \
  --no-output-timeout-seconds 30 \
  --output-max-bytes 65536 \
  --webhook "https://example.invalid/openclaw/cron"
```

## よく使う管理コマンド

手動実行と確認：

```bash
openclaw cron list
openclaw cron list --agent ops
openclaw cron get <job-id>
openclaw cron show <job-id>
openclaw cron run <job-id>
openclaw cron run <job-id> --due
openclaw cron run <job-id> --wait --wait-timeout 10m
openclaw cron run <job-id> --wait --wait-timeout 10m --poll-interval 2s
openclaw cron runs --id <job-id> --limit 50
openclaw cron runs --id <job-id> --run-id <run-id>
```

`openclaw cron list` はデフォルトで有効なジョブを表示します。無効なジョブも含めるには `--all` を渡し、有効な正規化済みエージェント ID が一致するジョブのみを表示するには `--agent <id>` を渡します。保存済みのエージェント ID がないジョブは、設定されたデフォルトエージェントとして扱われます。

`openclaw cron get <job-id>` は保存されたジョブ JSON を直接返します。配信経路のプレビューを含む人間が読みやすい表示が必要な場合は、`cron show <job-id>` を使用します。

`cron list --json` と `cron show <job-id> --json` では、各ジョブにトップレベルの `status` フィールドが含まれます。このフィールドは `enabled`、`state.runningAtMs`、`state.lastRunStatus` から算出されます。値は `disabled`、`running`、`ok`、`error`、`skipped`、または `idle` です。外部ツールがジョブ状態を再導出せずに読み取れるように、JSON のステータスは正規かつ装飾なしのまま維持されます。人間向け出力では、繰り返される `error` ステータスに失敗回数が付加される場合があります。

`cron runs` のエントリには、意図された cron ターゲット、解決済みターゲット、メッセージツールによる送信、フォールバックの使用、配信済み状態を含む配信診断が含まれます。

ジョブごとの非公開スクラッチ（Heartbeat チェックリストや同様の監視コンテキスト）：

```bash
openclaw cron scratch <job-id>                  # 現在のスクラッチ内容を出力
openclaw cron scratch <job-id> --json           # スクラッチとリビジョンメタデータ
openclaw cron scratch <job-id> --set "text"     # スクラッチを指定したテキストで置換
openclaw cron scratch <job-id> --file notes.md  # ファイルからスクラッチを置換（stdin の場合は -）
openclaw cron scratch <job-id> --unset          # スクラッチ行を削除
```

スクラッチは共有状態データベースに保存され、上限は 256 KiB で、`cron list`/`cron get`/`cron runs` の出力には一切含まれません。書き込みは、コマンド開始時に読み取ったリビジョンに対する compare-and-swap で保護されます。代わりに明示的なリビジョンを固定するには、`--expected-revision <n>` を渡します。Heartbeat モニターがスクラッチを使用する方法については、[Heartbeat](/ja-JP/gateway/heartbeat#monitor-scratch-optional)を参照してください。

エージェントとセッションの再ターゲット：

```bash
openclaw cron edit <job-id> --agent ops
openclaw cron edit <job-id> --clear-agent
openclaw cron edit <job-id> --session current
openclaw cron edit <job-id> --session "session:daily-brief"
```

`openclaw cron add` は、エージェントターンジョブで `--agent` が省略されると警告し、デフォルトエージェント（`main`）にフォールバックします。特定のエージェントに固定するには、作成時に `--agent <id>` を渡します。

配信の調整：

```bash
openclaw cron edit <job-id> --announce --channel slack --to "channel:C1234567890"
openclaw cron edit <job-id> --webhook "https://example.invalid/openclaw/cron"
openclaw cron edit <job-id> --best-effort-deliver
openclaw cron edit <job-id> --no-best-effort-deliver
openclaw cron edit <job-id> --no-deliver
```

## 関連項目

- [CLI リファレンス](/ja-JP/cli)
- [スケジュールされたタスク](/ja-JP/automation/cron-jobs)
