---
read_when:
    - Heartbeat の間隔またはメッセージングの調整
    - スケジュールされたタスクで Heartbeat と Cron のどちらを使用するかの判断
sidebarTitle: Heartbeat
summary: Heartbeat ポーリングメッセージと通知ルール
title: Heartbeat
x-i18n:
    generated_at: "2026-07-26T09:42:14Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 44c78e797987d8dccab910cd82fc1f482df86afce40677846d8f26522d33f6fa
    source_path: gateway/heartbeat.md
    workflow: 16
---

<Note>
**Heartbeat と cron の違いは？** それぞれをいつ使用するかについては、[自動化](/ja-JP/automation)を参照してください。
</Note>

Heartbeat はメインセッションで**定期的なエージェントターン**を実行し、煩雑な通知を送ることなく、注意が必要な事項をモデルが提示できるようにします。

Heartbeat はスケジュールされたメインセッションのターンであり、[バックグラウンドタスク](/ja-JP/automation/tasks)のレコードは作成**しません**。タスクレコードは、切り離された作業（ACP の実行、サブエージェント、分離された cron ジョブ）用です。

内部では、Heartbeat の実行間隔は cron スケジューラによって管理されます。Gateway は、Heartbeat が有効なエージェントごとにシステム所有の cron ジョブを 1 つ維持します（`openclaw cron list --all` では `Heartbeat (agent-id)` として表示されます）。Heartbeat 設定は望ましい状態の入力として保持される一方、永続化されたモニタースケジュールが実際の実行タイミングと、その後のランナーのクールダウンを管理します。Gateway は起動時および設定の再読み込み時に設定変更を書き込みます。`openclaw doctor --fix` は、次回の Gateway 起動前に欠落または古くなったモニター行を具体化できます。cron ジョブではなく、`agents.*.heartbeat` を編集してください。

スケジュールされた Heartbeat には cron が必要です。`cron.enabled` が `false` または `OPENCLAW_SKIP_CRON=1` の場合、Gateway は起動時に警告をログに記録し、スケジュールされた Heartbeat を実行しません。手動およびイベント駆動の Heartbeat のウェイクアップは引き続き利用できます。Heartbeat 専用のフォールバックタイマーはありません。

トラブルシューティング：[スケジュールされたタスク](/ja-JP/automation/cron-jobs#troubleshooting)

## クイックスタート（初心者向け）

<Steps>
  <Step title="実行間隔を選択する">
    Heartbeat を有効のままにする（デフォルトは `30m`。Claude CLI の再利用を含む Anthropic OAuth／トークン認証が設定されている場合は `1h`）か、独自の実行間隔を設定します。
  </Step>
  <Step title="モニターのスクラッチを追加する（任意）">
    `openclaw cron scratch <jobId> --set "..."` を使用して、Heartbeat モニターのスクラッチに簡単なチェックリストを保存します。
  </Step>
  <Step title="Heartbeat メッセージの送信先を決める">
    デフォルトは `target: "none"` です。最後の連絡先にルーティングするには、`target: "last"` を設定します。
  </Step>
  <Step title="任意の調整">
    - Heartbeat の実行にモニターのスクラッチのみが必要な場合は、軽量なブートストラップコンテキストを使用します。
    - Heartbeat ごとに会話履歴全体を送信しないようにするには、分離セッションを有効にします。
    - Heartbeat をアクティブ時間帯（現地時刻）に制限します。

  </Step>
</Steps>

設定例：

```json5
{
  agents: {
    defaults: {
      heartbeat: {
        every: "30m",
        target: "last", // 最後の連絡先に明示的に配信（デフォルトは "none"）
        directPolicy: "allow", // デフォルト：ダイレクト／DM ターゲットを許可。抑止するには "block" を設定
        lightContext: true, // 任意：Heartbeat 実行時にワークスペースのブートストラップファイルをスキップ
        isolatedSession: true, // 任意：実行ごとに新しいセッションを使用（会話履歴なし）
        // activeHours: { start: "08:00", end: "24:00" },
      },
    },
  },
}
```

## デフォルト

- 間隔：`30m`。解決された認証モードが OAuth／トークン（Claude CLI の再利用を含む）の場合、Anthropic プロバイダーのデフォルトを適用すると `1h` に引き上げられますが、これは `heartbeat.every` が未設定の場合に限られます。`agents.defaults.heartbeat.every` またはエージェントごとの `agents.entries.*.heartbeat.every` を設定します。無効にするには `0m` を使用します。
- プロンプト本文（`agents.defaults.heartbeat.prompt` で設定可能）：`Follow the heartbeat monitor scratch context when provided. Recurring tasks are cron jobs; create or change their schedules with cron tools or the openclaw cron CLI, not heartbeat scratch. Do not infer or repeat old tasks from prior chats. If nothing needs attention, reply HEARTBEAT_OK.`
- タイムアウト：Heartbeat ターンにタイムアウトが設定されていない場合、`agents.defaults.timeoutSeconds` が設定されていればそれを使用します。それ以外の場合は、最大 600 秒に制限された Heartbeat の実行間隔を使用します。より長い Heartbeat 作業には、`agents.defaults.heartbeat.timeoutSeconds` またはエージェントごとの `agents.entries.*.heartbeat.timeoutSeconds` を設定します。
- Heartbeat プロンプトは、ユーザーメッセージとして**そのまま**送信されます。デフォルトエージェントで Heartbeat が有効な場合、システムプロンプトには「Heartbeats」セクションが含まれ、実行には内部的にフラグが設定されます。
- `0m` で Heartbeat を無効にしても、モニターの cron ジョブは残りますが無効化され、そのスクラッチは実行間隔を再度有効にしたときのために保持されます。
- cron 自体が無効な場合、Heartbeat の実行間隔が有効なままでも、スケジュールされた Heartbeat は実行されません。
- アクティブ時間帯（`heartbeat.activeHours`）は、設定されたタイムゾーンで判定されます。時間帯の範囲外では、範囲内の次の実行タイミングまで Heartbeat はスキップされます。
- cron の作業が実行中またはキューにある場合、あるいはそのエージェントのセッションキーに紐づくサブエージェントまたはネストされたコマンドレーンがビジー状態の場合、Heartbeat は自動的に延期されます。兄弟エージェント同士は互いを一時停止しません。

## Heartbeat プロンプトの用途

デフォルトのプロンプトは、意図的に幅広い内容になっています。

- **バックグラウンドタスク**：「未処理のタスクを検討する」という指示により、エージェントはフォローアップ（受信トレイ、カレンダー、リマインダー、キュー内の作業）を確認し、緊急の事項を提示するよう促されます。
- **利用者への確認**：「日中は時々利用者の様子を確認する」という指示により、ときどき簡単な「何か必要ですか？」というメッセージを送るよう促されます。ただし、設定された現地タイムゾーンを使用することで夜間の煩雑な通知を避けます（[タイムゾーン](/ja-JP/concepts/timezone)を参照）。

Heartbeat は完了した[バックグラウンドタスク](/ja-JP/automation/tasks)に反応できますが、Heartbeat の実行自体はタスクレコードを作成しません。

Heartbeat に非常に具体的な処理（例：「Gmail PubSub の統計情報を確認する」または「Gateway の正常性を検証する」）を実行させるには、`agents.defaults.heartbeat.prompt`（または `agents.entries.*.heartbeat.prompt`）をカスタム本文（そのまま送信されます）に設定します。

## 応答規約

- 注意が必要な事項がない場合は、**`HEARTBEAT_OK`** と応答します。
- Heartbeat の実行では、表示する更新がない場合は `notify: false` を指定して `heartbeat_respond` を呼び出し、アラートの場合は `notificationText` を指定して `notify: true` を呼び出すこともできます。構造化されたツール応答が存在する場合は、テキストによるフォールバックよりも優先されます。
- `notify: false` を伴う意味のある `heartbeat_respond` の結果は表示されませんが、そのセッションの次のユーザーターンに向けた限定的な内部コンテキストとして記憶されます。`no_change` の確認応答と表示される通知は、この方法では保存されません。
- Heartbeat の実行中、OpenClaw は `HEARTBEAT_OK` が応答の**先頭または末尾**にある場合、それを確認応答として扱います。トークンは削除され、残りの内容が 300 文字以下であれば応答は破棄されます。
- `HEARTBEAT_OK` が応答の**途中**にある場合、特別な処理は行われません。
- アラートでは `HEARTBEAT_OK` を含め**ないでください**。アラートのテキストのみを返します。

Heartbeat 以外では、メッセージの先頭／末尾にある不要な `HEARTBEAT_OK` は削除され、ログに記録されます。`HEARTBEAT_OK` のみで構成されるメッセージは破棄されます。

## 設定

```json5
{
  agents: {
    defaults: {
      heartbeat: {
        every: "30m", // デフォルト：30m（0m で無効化）
        model: "anthropic/claude-opus-4-6",
        lightContext: false, // デフォルト：false。true の場合、Heartbeat 実行時にワークスペースのブートストラップファイルをスキップ
        isolatedSession: false, // デフォルト：false。true の場合、Heartbeat ごとに新しいセッションで実行（会話履歴なし）
        target: "last", // デフォルト：none | 選択肢：last | none | <channel id>（コアまたは Plugin、例："imessage"）
        to: "+15551234567", // 任意のチャンネル固有オーバーライド
        accountId: "ops-bot", // 任意のマルチアカウントチャンネル ID
        prompt: "Heartbeat モニターのスクラッチコンテキストが提供されている場合は、それに従ってください。定期タスクは cron ジョブです。そのスケジュールの作成または変更には、Heartbeat のスクラッチではなく、cron ツールまたは openclaw cron CLI を使用してください。以前のチャットから古いタスクを推測したり繰り返したりしないでください。注意が必要な事項がない場合は、HEARTBEAT_OK と応答してください。",
      },
    },
  },
}
```

### 適用範囲と優先順位

- `agents.defaults.heartbeat` はグローバルな Heartbeat の動作を設定します。
- `agents.entries.*.heartbeat` はその上にマージされます。いずれかのエージェントに `heartbeat` ブロックがある場合、**それらのエージェントのみ**が Heartbeat を実行します。
- `channels.defaults.heartbeatVisibility` はすべてのチャンネルの表示設定のデフォルトを設定します。
- `channels.<channel>.heartbeatVisibility` はチャンネルのデフォルトをオーバーライドします。
- `channels.<channel>.accounts.<id>.heartbeatVisibility`（マルチアカウントチャンネル）は、チャンネルごとの設定をオーバーライドします。

### エージェントごとの Heartbeat

いずれかの `agents.entries.*` エントリに `heartbeat` ブロックが含まれている場合、**それらのエージェントのみ**が Heartbeat を実行します。エージェントごとのブロックは `agents.defaults.heartbeat` の上にマージされます（共有デフォルトを一度設定し、エージェントごとにオーバーライドできます）。

例：2 つのエージェントのうち、2 番目のエージェントのみが Heartbeat を実行します。

```json5
{
  agents: {
    defaults: {
      heartbeat: {
        every: "30m",
        target: "last", // 最後の連絡先に明示的に配信（デフォルトは "none"）
      },
    },
    list: [
      { id: "main", default: true },
      {
        id: "ops",
        heartbeat: {
          every: "1h",
          target: "whatsapp",
          to: "+15551234567",
          timeoutSeconds: 45,
          prompt: "Heartbeat モニターのスクラッチコンテキストが提供されている場合は、それに従ってください。定期タスクは cron ジョブです。そのスケジュールの作成または変更には、Heartbeat のスクラッチではなく、cron ツールまたは openclaw cron CLI を使用してください。以前のチャットから古いタスクを推測したり繰り返したりしないでください。注意が必要な事項がない場合は、HEARTBEAT_OK と応答してください。",
        },
      },
    ],
  },
}
```

### アクティブ時間帯の例

特定のタイムゾーンの営業時間内に Heartbeat を制限します。

```json5
{
  agents: {
    defaults: {
      heartbeat: {
        every: "30m",
        target: "last", // 最後の連絡先に明示的に配信（デフォルトは "none"）
        activeHours: {
          start: "09:00",
          end: "22:00",
          timezone: "America/New_York", // 任意。設定されている場合は userTimezone を使用し、それ以外の場合はホストのタイムゾーンを使用
        },
      },
    },
  },
}
```

この時間帯の範囲外（東部時間の午前 9 時より前、または午後 10 時より後）では、Heartbeat はスキップされます。範囲内の次のスケジュールされた実行タイミングでは、通常どおり実行されます。

### 24/7 の設定

Heartbeat を終日実行するには、次のいずれかのパターンを使用します。

- `activeHours` を完全に省略します（時間帯による制限なし。これがデフォルトの動作です）。
- 終日の時間帯を設定します：`activeHours: { start: "00:00", end: "24:00" }`。

<Warning>
`start` と `end` に同じ時刻（例：`08:00` から `08:00`）を設定しないでください。これは幅がゼロの時間帯として扱われるため、Heartbeat は常にスキップされます。
</Warning>

### マルチアカウントの例

Telegram などのマルチアカウントチャンネルで特定のアカウントを対象にするには、`accountId` を使用します。

```json5
{
  agents: {
    list: [
      {
        id: "ops",
        heartbeat: {
          every: "1h",
          target: "telegram",
          to: "12345678:topic:42", // 任意：特定のトピック／スレッドにルーティング
          accountId: "ops-bot",
        },
      },
    ],
  },
  channels: {
    telegram: {
      accounts: {
        "ops-bot": { botToken: "YOUR_TELEGRAM_BOT_TOKEN" },
      },
    },
  },
}
```

### フィールドの注記

<ParamField path="every" type="string">
  Heartbeat の間隔（期間を表す文字列。デフォルトの単位は分）。
</ParamField>
<ParamField path="model" type="string">
  Heartbeat 実行時の任意のモデルオーバーライド（`provider/model`）。
</ParamField>
<ParamField path="lightContext" type="boolean" default="false">
  true の場合、Heartbeat の実行では軽量なブートストラップコンテキストを使用し、ワークスペースのブートストラップファイルをスキップします。どちらの場合も、モニターのスクラッチは Heartbeat ランナーによって注入されます。
</ParamField>
<ParamField path="isolatedSession" type="boolean" default="false">
  true の場合、各 Heartbeat は以前の会話履歴がない新しいセッションで実行されます。cron `sessionTarget: "isolated"` と同じ分離パターンを使用します。Heartbeat ごとのトークンコストを大幅に削減します。最大限に節約するには `lightContext: true` と組み合わせます。配信ルーティングでは引き続きメインセッションのコンテキストを使用します。
</ParamField>
<ParamField path="session" type="string">
  Heartbeat 実行時の任意のセッションキー。

- `main`（デフォルト）：エージェントのメインセッション。
- 明示的なセッションキー（`openclaw sessions --json` または [sessions CLI](/ja-JP/cli/sessions) からコピー）。
- セッションキーの形式：[セッション](/ja-JP/concepts/session)および[グループ](/ja-JP/channels/groups)を参照してください。

</ParamField>
<ParamField path="target" type="string">
- `last`: 最後に使用した外部チャネルへ配信します。
- 明示的なチャネル: 設定済みの任意のチャネルまたは Plugin ID（例: `discord`、`matrix`、`telegram`、`whatsapp`）。
- `none`（デフォルト）: Heartbeat を実行しますが、外部には**配信しません**。

</ParamField>
<ParamField path="directPolicy" type='"allow" | "block"' default="allow">
  ダイレクトメッセージ/DM の配信動作を制御します。`allow`: ダイレクトメッセージ/DM への Heartbeat 配信を許可します。`block`: ダイレクトメッセージ/DM への配信を抑止します（`reason=dm-blocked`）。

</ParamField>
<ParamField path="to" type="string">
  受信者を任意で上書きします（チャネル固有の ID。例: WhatsApp の E.164、Telegram のチャット ID）。Telegram のトピック/スレッドには `<chatId>:topic:<messageThreadId>` を使用します。

</ParamField>
<ParamField path="accountId" type="string">
  複数アカウント対応チャネル用の任意のアカウント ID。`target: "last"` の場合、解決された最後のチャネルがアカウントをサポートしていれば、そのチャネルにアカウント ID が適用されます。それ以外の場合は無視されます。アカウント ID が、解決されたチャネルに設定済みのアカウントと一致しない場合、配信はスキップされます。

</ParamField>
<ParamField path="prompt" type="string">
  デフォルトのプロンプト本文を上書きします（マージされません）。

</ParamField>
<ParamField path="timeoutSeconds" type="number" default="global timeout or min(every, 600)">
  Heartbeat のエージェントターンが中止されるまでに許容される最大秒数。未設定の場合、`agents.defaults.timeoutSeconds` が設定されていればその値を使用し、それ以外の場合は最大 600 秒に制限された Heartbeat の間隔を使用します。

</ParamField>
<ParamField path="activeHours" type="object">
  Heartbeat の実行を時間帯に制限します。`start`（HH:MM、指定時刻を含む。日の開始には `00:00` を使用）、`end`（HH:MM、指定時刻を含まない。日の終了には `24:00` を使用可能）、および任意の `timezone` を持つオブジェクトです。

- 省略または `"user"`: `agents.defaults.userTimezone` が設定されていればそれを使用し、それ以外の場合はホストシステムのタイムゾーンにフォールバックします。
- `"local"`: 常にホストシステムのタイムゾーンを使用します。
- 任意の IANA 識別子（例: `America/New_York`）: そのまま使用します。無効な場合は、上記の `"user"` の動作にフォールバックします。
- 有効な時間帯では、`start` と `end` を同じ値にしてはなりません。同じ値は幅がゼロ（常に時間帯の範囲外）として扱われます。
- 有効な時間帯の範囲外では、範囲内の次のティックまで Heartbeat はスキップされます。

</ParamField>

## 配信動作

<AccordionGroup>
  <Accordion title="セッションと配信先のルーティング">
    - デフォルトでは、Heartbeat はエージェントのメインセッション（`agent:<id>:<mainKey>`）で実行されます。`session.scope = "global"` の場合は `global` で実行されます。特定のチャネルセッション（Discord/WhatsApp など）に上書きするには、`session` を設定します。
    - `session` が影響するのは実行コンテキストのみです。配信は `target` と `to` によって制御されます。
    - 特定のチャネル/受信者へ配信するには、`target` と `to` を設定します。`target: "last"` の場合、そのセッションで最後に使用した外部チャネルへ配信されます。
    - デフォルトでは、Heartbeat はダイレクトメッセージ/DM の配信先へ配信できます。Heartbeat ターンを実行したまま、ダイレクトメッセージの配信先への送信を抑止するには、`directPolicy: "block"` を設定します。
    - メインキュー、対象セッションのレーン、Cron レーン、または実行中の Cron ジョブがビジー状態の場合、Heartbeat はスキップされ、後で再試行されます。
    - `target` を解決しても外部の配信先が得られない場合、実行は行われますが、送信メッセージはありません。

  </Accordion>
  <Accordion title="表示とスキップの動作">
    - `showOk`、`showAlerts`、`useIndicator` がすべて無効の場合、実行は開始前に `reason=alerts-disabled` としてスキップされます。
    - アラート配信のみが無効の場合でも、OpenClaw は Heartbeat を実行し、期限が来たタスクのタイムスタンプを更新し、セッションのアイドルタイムスタンプを復元し、外部向けのアラートペイロードを抑止できます。
    - 解決された Heartbeat の配信先が入力中表示をサポートしている場合、OpenClaw は Heartbeat の実行中に入力中であることを表示します。これは Heartbeat がチャット出力を送信するのと同じ配信先を使用し、`typingMode: "never"` によって無効になります。

  </Accordion>
  <Accordion title="セッションのライフサイクルと監査">
    - Heartbeat のみの応答によってセッションが維持されることは**ありません**。Heartbeat のメタデータによってセッション行が更新される場合がありますが、アイドル期限切れには最後の実際のユーザー/チャネルメッセージの `lastInteractionAt` が使用され、日次の期限切れには `sessionStartedAt` が使用されます。
    - Control UI と WebChat の履歴では、Heartbeat のプロンプトと OK のみの確認応答は非表示になります。ただし、基礎となるセッションのトランスクリプトには、監査/再生用にそれらのターンが引き続き含まれる場合があります。
    - 切り離された[バックグラウンドタスク](/ja-JP/automation/tasks)は、メインセッションが何かにすぐ気付く必要がある場合、システムイベントをキューに追加して Heartbeat を起動できます。この起動によって、Heartbeat の実行がバックグラウンドタスクになることはありません。

  </Accordion>
</AccordionGroup>

## 表示制御

デフォルトでは、アラート内容が配信される一方で、`HEARTBEAT_OK` の確認応答は抑止されます。これはチャネルごと、またはアカウントごとに調整できます。

```yaml
channels:
  defaults:
    heartbeat:
      showOk: false # HEARTBEAT_OK を非表示（デフォルト）
      showAlerts: true # アラートメッセージを表示（デフォルト）
      useIndicator: true # インジケーターイベントを生成（デフォルト）
  telegram:
    heartbeat:
      showOk: true # Telegram で OK の確認応答を表示
  whatsapp:
    accounts:
      work:
        heartbeat:
          showAlerts: false # このアカウントへのアラート配信を抑止
```

優先順位: アカウントごと → チャネルごと → チャネルのデフォルト → 組み込みのデフォルト。

### 各フラグの動作

- `showOk`: モデルが OK のみの応答を返したとき、`HEARTBEAT_OK` の確認応答を送信します。
- `showAlerts`: モデルが OK 以外の応答を返したとき、アラート内容を送信します。
- `useIndicator`: UI のステータス表示用にインジケーターイベントを生成します。

**3 つすべて**が false の場合、OpenClaw は Heartbeat の実行を完全にスキップします（モデルの呼び出しもありません）。

### チャネルごととアカウントごとの例

```yaml
channels:
  defaults:
    heartbeat:
      showOk: false
      showAlerts: true
      useIndicator: true
  slack:
    heartbeat:
      showOk: true # すべての Slack アカウント
    accounts:
      ops:
        heartbeat:
          showAlerts: false # ops アカウントのアラートのみを抑止
  telegram:
    heartbeat:
      showOk: true
```

### 一般的なパターン

| 目的                                     | 設定                                                                                   |
| ---------------------------------------- | ---------------------------------------------------------------------------------------- |
| デフォルトの動作（OK は非表示、アラートは有効） | _（設定不要）_                                                                     |
| 完全に無音（メッセージもインジケーターもなし） | `channels.defaults.heartbeat: { showOk: false, showAlerts: false, useIndicator: false }` |
| インジケーターのみ（メッセージなし）             | `channels.defaults.heartbeat: { showOk: false, showAlerts: false, useIndicator: true }`  |
| 1 つのチャネルのみで OK を表示                  | `channels.telegram.heartbeat: { showOk: true }`                                          |

## モニターのスクラッチ（任意）

各 Heartbeat モニターの Cron ジョブは、共有状態データベースに保存される非公開のスクラッチ文書を所有します。これは「Heartbeat チェックリスト」と考えてください。小さく安定しており、30 分ごとに確認しても安全な内容にします。スクラッチが存在する場合、その内容が Heartbeat のプロンプトに追加されます。

Cron CLI で管理します（ジョブ ID は `openclaw cron list --all` から取得します）。

```bash
openclaw cron scratch <jobId>                 # 現在のスクラッチを表示
openclaw cron scratch <jobId> --set "..."     # 正確なテキストで置き換え
openclaw cron scratch <jobId> --file notes.md # ファイルから置き換え（標準入力には -）
openclaw cron scratch <jobId> --unset         # 削除
```

書き込みは compare-and-swap で保護されています。同時編集を上書きせず失敗させるには、`--expected-revision <n>` を渡します。スクラッチの上限は 256 KiB で、`cron list`/`cron runs` の出力には表示されません。

エージェントは自身のスクラッチも更新できます。Heartbeat ターン中、`heartbeat_respond` は任意の `scratch` 文字列を受け取り、以降の Heartbeat で使用するモニターのスクラッチを完全に置き換えます。

<Note>
**HEARTBEAT.md または設定のみの実行間隔から移行する場合** `openclaw doctor --fix` を実行します。Doctor はまず `agents.*.heartbeat` からシステム所有のモニター行を作成または更新し、次に各エージェントのワークスペースの `HEARTBEAT.md` をモニターのスクラッチへインポートし、有効な従来の `tasks:` エントリを Cron ジョブへ変換し、元のファイルを状態ディレクトリ（`backups/heartbeat-migration/`）の下へアーカイブしてから削除します。実行時の Heartbeat 命令はデータベースのスクラッチのみから取得されます。ランタイムが `HEARTBEAT.md` を読み取ることはありません。
</Note>

スクラッチが存在しても実質的に空の場合（空行、Markdown/HTML コメント、`# Heading` のような Markdown 見出し、フェンスマーカー、または空のチェックリストのひな形のみ）、OpenClaw は API 呼び出しを節約するため Heartbeat の実行をスキップします。このスキップは `reason=empty-heartbeat-file` として報告されます。スクラッチが存在しない場合でも Heartbeat は実行され、何を行うかはモデルが判断します。

プロンプトの肥大化を避けるため、短いチェックリストやリマインダーだけにしておきます。

スクラッチの例:

```md
# Heartbeat チェックリスト

- 簡単に確認: 受信トレイに緊急のものはあるか？
- 日中で、ほかに保留中のものがなければ、簡単な状況確認を行う。
- タスクがブロックされている場合は、_不足しているもの_を書き留め、次回 Peter に尋ねる。
```

### Cron で定期チェックをスケジュールする

Heartbeat のスクラッチはプロンプトのコンテキストであり、スケジューラーではありません。各定期チェックを [Cron ジョブ](/ja-JP/automation/cron-jobs)として作成すると、独自の実行間隔、有効/無効状態、実行履歴を持たせることができます。通常の会話コンテキストを使用する必要があるチェックでは、Cron ジョブの対象を引き続きメインセッションにできます。

古いスクラッチには、構造化された `tasks:` ブロックが含まれている場合があります。アップグレード後に `openclaw doctor --fix` を 1 回実行してください。Doctor は有効な各エントリを個別にスケジュールされた Cron ジョブへ変換し、その間隔と以前の最終実行時刻を保持し、周囲のスクラッチ本文を残したまま廃止されたブロックを削除します。実行時の Heartbeat ターンは、`tasks:` のテキストをスケジュールとして解析しません。

Doctor が作成した Heartbeat タスクジョブでは、Heartbeat の有効時間帯、クールダウン、過剰実行防止、ビジー状態のガードが維持されます。同時に期限を迎えたジョブは、1 回の Heartbeat ターンにまとめられる場合があります。有効時間帯の範囲外にある実行回はスキップされ、次の Cron 実行時に再試行されます。

### エージェントはスクラッチを更新できますか？

はい。Heartbeat ターン中、エージェントは `heartbeat_respond` に `scratch` の値を渡し、以降の Heartbeat で使用するモニター本文を完全に置き換えることができます。通常のチャットで `openclaw cron scratch <jobId> --set ...` を実行するよう依頼するか、同じコマンドを使用して自身でスクラッチを編集することもできます。スクラッチにスケジューラー構文を書くのではなく、定期スケジュールは Cron で管理してください。

<Warning>
シークレット（API キー、電話番号、非公開トークン）をモニターのスクラッチに含めないでください。プロンプトコンテキストの一部になります。
</Warning>

## 手動起動（オンデマンド）

`openclaw system event` を使用してシステムイベントをキューに追加し、必要に応じて Heartbeat を即座にトリガーします。

```bash
openclaw system event --text "緊急のフォローアップを確認" --mode now
```

| フラグ                         | 説明                                                                                      |
| ---------------------------- | ------------------------------------------------------------------------------------------------ |
| `--text <text>`              | システムイベントのテキスト（必須）。                                                                    |
| `--mode <mode>`              | `now` は即座に Heartbeat を実行し、`next-heartbeat`（デフォルト）は次に予定されたティックまで待機します。 |
| `--session-key <sessionKey>` | イベントの対象を特定のセッションにします。デフォルトはエージェントのメインセッションです。                   |
| `--json`                     | JSON を出力します。                                                                                     |

`--session-key` が指定されておらず、複数のエージェントに `heartbeat` が設定されている場合、`--mode now` はそれらの各エージェントの Heartbeat を即座に実行します。

同じ CLI グループ内の関連する Heartbeat 制御：

```bash
openclaw system heartbeat last     # 最後の Heartbeat イベントを表示
openclaw system heartbeat enable   # Heartbeat を有効化
openclaw system heartbeat disable  # Heartbeat を無効化
```

## コストへの配慮

Heartbeat はエージェントの完全なターンを実行します。間隔を短くすると、より多くのトークンを消費します。コストを削減するには：

- 会話履歴全体の送信を避けるため、`isolatedSession: true` を使用します（実行ごとに約 100K トークンから約 2～5K トークンに削減）。
- Heartbeat の実行時にワークスペースのブートストラップファイルを省略するには、`lightContext: true` を使用します。
- より安価な `model` を設定します（例：`ollama/llama3.2:1b`）。
- モニターのスクラッチ領域を小さく保ちます。
- 内部状態の更新のみが必要な場合は、`target: "none"` を使用します。

## Heartbeat 後のコンテキストオーバーフロー

Heartbeat の実行完了後も、共有セッションの既存のランタイムモデルは維持されます。そのため、Heartbeat によってセッションがより小さなローカルモデル（たとえば、32k ウィンドウの Ollama モデル）に切り替わると、次のメインセッションのターンでもそのモデルが使用される場合があります。その次のターンでコンテキストオーバーフローが報告され、セッションの最後のランタイムモデルが設定済みの `heartbeat.model` と一致する場合、OpenClaw の復旧メッセージは Heartbeat モデルの持ち越しが原因である可能性を示し、修正方法を提案します。

これを回避するには、`isolatedSession: true` を使用して新しいセッションで Heartbeat を実行するか（最小のプロンプトにするため、必要に応じて `lightContext: true` と組み合わせます）、共有セッションに十分な大きさのコンテキストウィンドウを持つ Heartbeat モデルを選択します。

## 関連項目

- [自動化](/ja-JP/automation) - すべての自動化メカニズムの概要
- [バックグラウンドタスク](/ja-JP/automation/tasks) - 分離された処理を追跡する方法
- [タイムゾーン](/ja-JP/concepts/timezone) - タイムゾーンが Heartbeat のスケジュールに与える影響
- [トラブルシューティング](/ja-JP/automation/cron-jobs#troubleshooting) - 自動化に関する問題のデバッグ方法
