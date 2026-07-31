---
read_when:
    - before_tool_call、before_agent_reply、メッセージフック、またはライフサイクルフックを必要とする Plugin を構築している場合
    - Plugin からのツール呼び出しをブロック、書き換え、または承認必須にする必要がある場合
    - 内部フックとプラグインフックのどちらを使用するか決めようとしています
    - OpenClaw の Cron ウェイクを外部ホストのスケジューラに反映しています
summary: Plugin フック：エージェント、ツール、メッセージ、セッション、Gateway のライフサイクルイベントをインターセプトする
title: Plugin フック
x-i18n:
    generated_at: "2026-07-26T09:32:04Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 95d7ea2f7bfe26b5904ea3cd8f8db85ffd8163af58e03ec56d11eee992bc13d2
    source_path: plugins/hooks.md
    workflow: 16
---

Plugin フックは、OpenClaw Plugin 向けのインプロセス拡張ポイントです。エージェント実行、ツール呼び出し、メッセージフロー、セッションライフサイクル、サブエージェントの
ルーティング、インストール、または Gateway の起動を検査または変更できます。

`/new`、
`/reset`、`/stop`、`agent:bootstrap`、`gateway:startup` などのコマンドおよび Gateway イベントに反応する、オペレーターがインストールする小規模な
`HOOK.md` スクリプトには、代わりに[内部フック](/ja-JP/automation/hooks)を使用してください。

## クイックスタート

Plugin のエントリから `api.on(...)` を使用して、型付きフックを登録します。

```typescript
import { definePluginEntry } from "openclaw/plugin-sdk/plugin-entry";

export default definePluginEntry({
  id: "tool-preflight",
  name: "Tool Preflight",
  register(api) {
    api.on(
      "before_tool_call",
      async (event) => {
        if (event.toolName !== "web_search") {
          return;
        }

        return {
          requireApproval: {
            title: "Web 検索を実行",
            description: `検索クエリを許可: ${String(event.params.query ?? "")}`,
            severity: "info",
            timeoutMs: 60_000,
          },
        };
      },
      { priority: 50 },
    );
  },
});
```

判断または変更を返せるハンドラーは、`priority` の降順で
逐次実行されます。同じ優先度のハンドラーでは登録順が維持されます。
監視専用ハンドラーは並列実行され、ファイア・アンド・フォーゲット方式の監視
ディスパッチは後続のイベントと重なる場合があります。監視による副作用の順序付けに
優先度を使用しないでください。

`api.on(name, handler, opts?)` は次を受け付けます。

| オプション      | 効果                                                                                                                                                                                            |
| ----------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `priority`  | 実行順序。値が大きいほど先に実行されます。                                                                                                                                                                      |
| `timeoutMs` | フックごとの待機時間枠。期限が切れると、OpenClaw はそのハンドラーの待機を停止して次へ進みます。ハンドラーやその副作用はキャンセルされません。省略すると、ランナーのフックごとのデフォルトタイムアウトが使用されます。 |

オペレーターは Plugin コードにパッチを適用せずに、フックの時間枠を設定できます。

```json
{
  "plugins": {
    "entries": {
      "my-plugin": {
        "hooks": {
          "timeoutMs": 30000,
          "timeouts": {
            "before_prompt_build": 90000,
            "agent_end": 60000
          }
        }
      }
    }
  }
}
```

`hooks.timeouts.<hookName>` は `hooks.timeoutMs` より優先され、さらにこれは
Plugin 側で指定された `api.on(..., { timeoutMs })` の値より優先されます。各値は
600000 ms 以下の正の整数である必要があります。1 つの Plugin の時間枠があらゆる場所で長くならないように、
処理が遅いことが判明しているフックにはフックごとのオーバーライドを推奨します。

フックのコールバックはキャンセルシグナルを受信しないため、タイムアウトしたハンドラーの Promise は
実行を継続します。その Plugin の処理がまだ進行中でも、フックのディスパッチは
Gateway の受け入れ枠を解放できます。長時間実行される処理を所有する
Plugin は、独自のキャンセルおよびシャットダウンライフサイクルを提供する必要があります。

送信内容を変更するフック `message_sending` と `reply_payload_sending` では、
ハンドラーごとに 15 秒のデフォルト値が使用されます。いずれかがタイムアウトすると、OpenClaw は Plugin のエラーをログに記録し、
最新のペイロードを使用して処理を続行するため、直列化された配信レーンを
完了させることができます。配信前に意図的に時間のかかる処理を行う Plugin には、
フックごとにより長い時間枠を設定してください。

`createReplyDispatcher` を使用するチャンネル Plugin も同様に、
`beforeDeliverOptions: { timeoutMs }` でステージごとのより長い正の時間枠を宣言できます。または、
`dispatcher.appendBeforeDeliver(handler, { timeoutMs })` で処理を追加するときにも宣言できます。
所有者が時間枠を宣言していない場合、それらのコールバックには同じ 15 秒の
デフォルト値が使用されるため、停止したコールバックが直列化された配信レーンを占有し続けることはありません。

各フックは `event.context.pluginConfig`、つまりそのハンドラーを登録した
plugin の解決済み設定を受け取ります。OpenClaw は、他の plugin が参照する共有イベントオブジェクトを
変更せず、ハンドラーごとにこれを注入します。

## フックカタログ

フックは、拡張するサーフェスごとにグループ化されています。**太字**の名前は判断結果
（ブロック、キャンセル、上書き、または承認の要求）を受け付け、それ以外は
監視専用です。

**エージェントターン**

| フック                            | 目的                                                                                  |
| ------------------------------- | ---------------------------------------------------------------------------------------- |
| `before_model_resolve`          | セッションメッセージを読み込む前にプロバイダーまたはモデルを上書きする                                  |
| `agent_turn_prepare`            | キューに入った plugin のターン注入を処理し、プロンプトフックの前に同一ターンのコンテキストを追加する      |
| `before_prompt_build`           | モデル呼び出しの前に動的コンテキストまたはシステムプロンプトのテキストを追加する                          |
| **`before_agent_run`**          | モデルへの送信前に最終プロンプトとセッションメッセージを検査する。実行をブロック可能 |
| **`before_agent_reply`**        | 合成応答または無応答によってモデルターンを早期終了する                           |
| **`before_agent_finalize`**     | 自然な最終回答を検査し、モデルの追加実行を1回要求する                         |
| `agent_end`                     | 最終メッセージ、成功状態、および実行時間を監視する                                  |
| `heartbeat_prompt_contribution` | バックグラウンドモニターおよびライフサイクル plugin 用に Heartbeat 専用コンテキストを追加する                  |

**会話の監視**

| フック                                      | 目的                                                                                                            |
| ----------------------------------------- | ------------------------------------------------------------------------------------------------------------------ |
| `model_call_started` / `model_call_ended` | サニタイズ済みのプロバイダー／モデル呼び出しメタデータ：タイミング、結果、長さを制限したリクエスト ID のハッシュ。プロンプトまたは応答の内容は含まれません。 |
| `llm_input`                               | プロバイダー入力：システムプロンプト、プロンプト、履歴                                                                     |
| `llm_output`                              | プロバイダー出力、使用量、および利用可能な場合は解決済みの `contextTokenBudget`                                       |

**ツール**

| フック                       | 目的                                                   |
| -------------------------- | --------------------------------------------------------- |
| **`before_tool_call`**     | ツールパラメーターの書き換え、実行のブロック、または承認の要求 |
| `after_tool_call`          | ツールの結果、エラー、および所要時間を監視する                |
| `resolve_exec_env`         | plugin が所有する環境変数を `exec` に提供する   |
| **`tool_result_persist`**  | ツール結果から生成されたアシスタントメッセージを書き換える |
| **`before_message_write`** | 進行中のメッセージ書き込みを検査またはブロックする（まれ）      |

**メッセージと配信**

| フック                            | 目的                                                           |
| ------------------------------- | ----------------------------------------------------------------- |
| **`inbound_claim`**             | エージェントへのルーティング前に受信メッセージを引き受ける（合成応答） |
| **`channel_pairing_requested`** | 新しく作成された DM ペアリング要求を監視する                         |
| `message_received`              | 受信コンテンツ、送信者、スレッド、およびメタデータを監視する             |
| **`message_sending`**           | 送信コンテンツを書き換えるか、配信をキャンセルする                       |
| **`reply_payload_sending`**     | 配信前に正規化された応答ペイロードを変更またはキャンセルする        |
| `message_sent`                  | 送信配信の成功または失敗を監視する                      |
| **`before_dispatch`**           | チャネルへの引き渡し前に送信ディスパッチを検査または書き換える    |
| **`reply_dispatch`**            | 最終応答ディスパッチパイプラインに参加する                  |

**セッションと Compaction**

| フック                                     | 目的                                                                                                                                                                                                                                                                                                                                                                                                                                                          |
| ---------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `session_start` / `session_end`          | セッションのライフサイクル境界を追跡します。`reason` は `new`、`reset`、`idle`、`daily`、`compaction`、`deleted`、`shutdown`、`restart`、または `unknown` のいずれかです。`shutdown`/`restart` は、アクティブなセッションがある状態でプロセスが停止または再起動すると、Gateway のシャットダウンファイナライザーから発火します。これにより、plugin（メモリ、トランスクリプトストア）は再起動をまたいでゴースト行を開いたままにせず、完了処理を行えます。低速な plugin が SIGTERM/SIGINT をブロックできないよう、ファイナライザーには時間制限があります。 |
| `before_compaction` / `after_compaction` | Compaction サイクルを監視または注釈付けする                                                                                                                                                                                                                                                                                                                                                                                                                            |
| `before_reset`                           | セッションリセットイベント（`/reset`、プログラムによるリセット）を監視する                                                                                                                                                                                                                                                                                                                                                                                                     |

`parentSessionKey` および `emitCommandHooks: true` を伴う `sessions.create` 呼び出しでは、個別の子は常に `session_start` を受け取ります。呼び出し元は、親も `succeedsParent` を伴う終端 `session_end` を受け取るかどうかを宣言します。`true` は後継、`false` は並列の子を意味します。省略すると、従来の親ロールオーバー動作が維持されます。どちらの場合も、`command:new` および `before_reset` フックは要求された `/new` アクションを引き続き表します。

**サブエージェント**

- `subagent_spawned` / `subagent_ended` - サブエージェントの起動と完了を監視します。
- `subagent_delivery_target` - コアセッションのバインディングでルートを投影できない場合に、完了を配信するための互換性フックです。
- `subagent_spawning` - 非推奨の互換性フックです。コアは現在、`subagent_spawned` が発火する前に、チャネルのセッションバインディングアダプターを通じて `thread: true` サブエージェントのバインディングを準備します。
- `subagent_spawned` には、OpenClaw が起動前に子セッションのネイティブモデルを解決した場合、`resolvedModel` と `resolvedProvider` が含まれます。
- `subagent_ended` は、`targetSessionKey`（ID。`subagent_spawned.childSessionKey` と一致）、`targetKind`（`"subagent"` または `"acp"`）、`reason`、省略可能な `outcome`（`"ok"`、`"error"`、`"timeout"`、`"killed"`、`"reset"`、または `"deleted"`）、省略可能な `error`、`runId`、`endedAt`、`accountId`、および `sendFarewell` を保持します。`agentId` または `childSessionKey` は含まれ**ません**。対応する `subagent_spawned` イベントとの関連付けには `targetSessionKey` を使用してください。

**ライフサイクル**

| フック                             | 目的                                                                                              |
| -------------------------------- | ---------------------------------------------------------------------------------------------------- |
| `gateway_start` / `gateway_stop` | Plugin が所有するサービスを Gateway とともに開始または停止します                                                 |
| `deactivate`                     | `gateway_stop` の非推奨の互換性エイリアスです。新しい Plugin では `gateway_stop` を使用してください                 |
| `cron_reconciled`                | 起動または再読み込み後に、Gateway の完全な Cron 状態と照合します                            |
| `cron_changed`                   | Gateway が所有する Cron のライフサイクル変更（追加、更新、削除、開始、完了、スケジュール設定）を監視します |
| **`before_install`**             | 読み込まれた Plugin ランタイムから、ステージング済みの Skills または Plugin のインストール素材を検査します                         |

### チャネルのペアリングリクエスト

未ペアリングの DM 送信者が保留中のペアリングリクエストを作成した後に、Plugin がオペレーターへ通知するか、監査記録を書き込む必要がある場合は、`channel_pairing_requested` を使用します。このフックはリクエストの作成時にディスパッチされます。フックハンドラーが低速または失敗しても、ペアリング応答のチャネル配信は遅延しません。

```typescript
api.on("channel_pairing_requested", async (event) => {
  await notifyOperator({
    text: `${event.senderId} から新しい ${event.channel} ペアリングリクエストが届きました: ${event.code}`,
  });
});
```

このフックは監視専用です。ペアリング応答の承認、拒否、抑制、書き換えは行いません。ペイロードには、チャネル、省略可能な `accountId`、チャネルスコープの `senderId`、ペアリングの `code`、およびチャネルメタデータが含まれます。ペアリングコードは有効な一回限りの承認資格情報として扱い、信頼できるオペレーターの送信先にのみ配信してください。`metadata` は、送信者が提供した信頼できない ID テキストとして扱ってください。このフックには、受信メッセージの本文やメディアは含まれません。

## ランタイムフックのデバッグ

エージェントのターンでプロバイダーまたはモデルを切り替えるには `before_model_resolve` を使用します。これはモデルの解決前に実行されます。`llm_output` は、モデルの試行によってアシスタント出力が生成された後にのみ実行されます。

実際に有効なセッションモデルを確認するには、ランタイム登録を検査してから、`openclaw sessions` または Gateway のセッション／ステータスサーフェスを使用します。プロバイダーのペイロードをデバッグするには、`--raw-stream` および `--raw-stream-path <path>` を指定して Gateway を起動し、生のモデルストリームイベントを jsonl ファイルへ書き込みます。

## ツール呼び出しポリシー

`before_tool_call` は以下を受け取ります。

- `event.toolName`
- `event.params`
- 省略可能な `event.toolKind` と `event.toolInputKind`。意図的に同じ名前を共有するツール向けの、ホストを信頼できる情報源とする識別子です。たとえば、外側のコードモードによる `exec` 呼び出しでは `toolKind: "code_mode_exec"` を使用し、入力言語が既知の場合は `toolInputKind: "javascript" | "typescript"` を含めます
- 省略可能な `event.derivedPaths`。`apply_patch` など、既知のツールエンベロープについてホストが導出するベストエフォートの対象パスヒントです。これらのパスは不完全な場合や、ツールが実際に変更する範囲を過大に近似する場合があります（たとえば、不正な形式または部分的な入力の場合）
- 省略可能な `event.runId`
- 省略可能な `event.toolCallId`
- `ctx.agentId`、`ctx.sessionKey`、`ctx.sessionId`、`ctx.runId`、`ctx.toolKind`、`ctx.toolInputKind`、診断用の `ctx.trace` などのコンテキストフィールド
- 省略可能な `ctx.requester`。現在のメッセージ実行を開始した、ホストが導出するリクエスターです。`channel`、`accountId`、`senderId`、`senderIsOwner`、およびプロバイダーネイティブの `roleIds` を含めることができます。欠落しているフィールドは未証明であり、安全であることを保証するものではありません。ポリシーで必要な場合は、フェイルクローズにしてください。

以下を返すことができます。

```typescript
type BeforeToolCallResult = {
  params?: Record<string, unknown>;
  block?: boolean;
  blockReason?: string;
  requireApproval?: {
    title: string;
    description: string;
    severity?: "info" | "warning" | "critical";
    timeoutMs?: number;
    /** @deprecated 未解決の承認は常に拒否されます。 */
    timeoutBehavior?: "allow" | "deny";
    allowedDecisions?: Array<"allow-once" | "allow-always" | "deny">;
    pluginId?: string;
    onResolution?: (
      decision: "allow-once" | "allow-always" | "deny" | "timeout" | "cancelled",
    ) => Promise<void> | void;
  };
};
```

型付きライフサイクルフックのガード動作:

- `block: true` は終端となり、優先度の低いハンドラーをスキップします。
- `block: false` は決定なしとして扱われます。
- `params` は実行用のツールパラメーターを書き換えます。
- `requireApproval` はエージェントの実行を一時停止し、Plugin の承認を通じてユーザーに確認します。`/approve` は exec と Plugin の両方の承認を承認できます。Codex app-server のレポートモードによるネイティブ `PreToolUse` リレーでは、対応する app-server の承認リクエストに委ねられます。[Codex ハーネスランタイム](/ja-JP/plugins/codex-harness-runtime#hook-boundaries)を参照してください。
- 優先度の高いフックが承認を要求した後でも、優先度の低い `block: true` はブロックできます。
- `onResolution` は、解決された決定（`allow-once`、`allow-always`、`deny`、`timeout`、または `cancelled`）を受け取ります。

### 単一ファイルでの送信者対応ポリシー

スタンドアロンの Plugin ファイルを使用すると、別の設定スキーマを追加せずに、デプロイ固有のポリシーをコード内に保持できます。この例では、所有者にすべてのツールを許可し、設定済みのメンテナーには保守的なツールとメッセージアクションのセットを許可し、チャネル設定ですでに認可されている送信者に `/fix` を公開します。

```typescript
import { definePluginEntry } from "openclaw/plugin-sdk/plugin-entry";

const AGENT_ID = "maintenance-agent";
const MAINTAINER_SCOPES = [
  {
    channel: "discord",
    accountId: "operations",
    senderIds: new Set(["maintainer-user-id"]),
    roleIds: new Set(["maintainer-role-id"]),
  },
];
const MAINTAINER_TOOLS = new Set(["read", "web_fetch", "web_search", "session_status", "message"]);
const MAINTAINER_MESSAGE_ACTIONS = new Set(["react", "reply", "thread-create", "thread-reply"]);

export default definePluginEntry({
  id: "maintenance-access",
  name: "メンテナンスアクセス",
  description: "メンテナンスエージェントに送信者対応のツールポリシーを適用します。",
  register(api) {
    api.on("before_tool_call", (event, ctx) => {
      if (ctx.agentId !== AGENT_ID) {
        return;
      }

      const requester = ctx.requester;
      if (requester?.senderIsOwner === true) {
        return;
      }

      const maintainerScope = requester
        ? MAINTAINER_SCOPES.find(
            (scope) =>
              scope.channel === requester.channel && scope.accountId === requester.accountId,
          )
        : undefined;
      const isMaintainer =
        maintainerScope !== undefined &&
        ((requester?.senderId !== undefined && maintainerScope.senderIds.has(requester.senderId)) ||
          requester?.roleIds?.some((roleId) => maintainerScope.roleIds.has(roleId)) === true);
      if (!isMaintainer) {
        return { block: true, blockReason: "メンテナーアクセスが必要です。" };
      }

      if (event.toolName === "message") {
        const action = typeof event.params.action === "string" ? event.params.action : "";
        if (MAINTAINER_MESSAGE_ACTIONS.has(action)) {
          return;
        }
        return { block: true, blockReason: `message.${action || "unknown"} には所有者権限が必要です。` };
      }

      if (MAINTAINER_TOOLS.has(event.toolName)) {
        return;
      }
      return { block: true, blockReason: `${event.toolName} には所有者権限が必要です。` };
    });

    api.registerCommand({
      name: "fix",
      description: "メンテナンスエージェントに問題の調査と修正を依頼します。",
      acceptsArgs: true,
      requireAuth: true,
      handler: async (ctx) =>
        ctx.agentId === AGENT_ID
          ? { continueAgent: true }
          : { text: "このコマンドはメンテナンス会話でのみ使用できます。" },
    });
  },
});
```

ファイルを直接読み込み、Gateway を再起動します。

```json5
{
  agents: {
    list: [
      {
        id: "maintenance-agent",
        workspace: "~/.openclaw/workspace-maintenance",
      },
    ],
  },
  bindings: [
    {
      agentId: "maintenance-agent",
      match: {
        channel: "discord",
        accountId: "operations",
        peer: { kind: "channel", id: "maintenance-channel-id" },
      },
    },
  ],
  plugins: {
    load: { paths: ["~/.openclaw/policies/maintenance-access.ts"] },
  },
}
```

`AGENT_ID` には、メンテナンス会話にバインドされたエージェントを指定する必要があります。このバインディングは、通常のメッセージと `/fix` に使用するエージェントを選択します。スタンドアロンファイルは、所有者とメンテナーを区別するツールポリシーの唯一の所有者であり続けます。

`requireAuth: true` は、各チャネルの既存の送信者受け入れ設定を再利用します。Discord では、ギルドまたはチャネルの `users`/`roles` 許可リストでメンテナンス対象者を認可できます。他のチャネルでは、安定した送信者 ID を使用できます。その後、フックは Codex のネイティブ `PreToolUse` 呼び出しを含む、実行内のすべてのツール呼び出しに対して、より細かなツール単位の決定を適用します。モデルに見えているツールを拒否することはできますが、ホストによって省略されたツールを追加することはできません。既存のサンドボックス、exec 承認、所有者専用のコアツール、およびチャネルポリシーは引き続き適用され、フックでそれらを越える権限を付与することはできません。

例に示すように、送信者 ID とロール ID のスコープは正確なチャネル／アカウントのペアに限定してください。どちらもプロバイダー内でのみ有効な名前空間です。許可リストは保守的に保ってください。書き込みまたは実行ツールは、デプロイのサンドボックスと承認ポリシーによって安全性が確保される場合にのみ追加してください。自動実行またはシステム実行については、`ctx.requester` が存在しない場合に通過を許可するかを明示的に決定してください。この例では、対象エージェントに対して拒否します。

承認のルーティング、決定時の動作、および省略可能なツールや exec 承認の代わりに `requireApproval` を使用する状況については、[Plugin の権限リクエスト](/ja-JP/plugins/plugin-permission-requests)を参照してください。

ホストレベルのポリシーを必要とする Plugin は、`api.registerTrustedToolPolicy(...)` を使用して信頼済みツールポリシーを登録できます。これらは通常の `before_tool_call` フックおよび通常のフックによる決定より前に実行されます。バンドル済みの信頼済みポリシーが最初に実行され、次にインストール済み Plugin の信頼済みポリシーが Plugin の読み込み順に実行され、その後に通常の `before_tool_call` フックが実行されます。バンドル済み Plugin は既存の信頼済みポリシー経路を維持します。インストール済み Plugin は明示的に有効化し、すべてのポリシー ID を `contracts.trustedToolPolicies` で宣言する必要があります。未宣言の ID は登録前に拒否されます。ポリシー ID は登録元の Plugin にスコープされるため、異なる Plugin が同じローカル ID を再利用できます。この階層は、ワークスペースポリシー、予算の適用、予約済みワークフローの安全性など、ホストから信頼されるゲートにのみ使用してください。

### Exec 環境フック

`resolve_exec_env` を使用すると、コマンド実行前にプラグインが `exec`
ツール呼び出しへ環境変数を提供できます。受け取る内容は次のとおりです。

- `event.sessionKey`
- `event.toolName`。現在は常に `"exec"`
- `event.host`。`"gateway"`、`"sandbox"`、または `"node"` のいずれか
- `ctx.agentId`、`ctx.sessionKey`、
  `ctx.messageProvider`、`ctx.channelId` などのコンテキストフィールド

Exec 環境へマージする `Record<string, string>` を返します。ハンドラーは
優先順位に従って実行されます。同じキーについては、後の結果が前の結果を
上書きします。

フック出力は、マージ前にホストの Exec 環境キーポリシーによって
フィルタリングされます。`PATH` は常に削除されます（コマンド解決と安全なバイナリのチェックが
これに依存するためです）。無効なキー、および `LD_*`、
`DYLD_*`、`NODE_OPTIONS` などの危険なホスト上書きキー、プロキシ変数（`HTTP_PROXY`、`HTTPS_PROXY`、
`ALL_PROXY`、`NO_PROXY`）、TLS 上書き変数（`NODE_TLS_REJECT_UNAUTHORIZED`、
`SSL_CERT_FILE` など）は削除されます。フィルタリングされたプラグイン環境は
Gateway の承認／監査メタデータに含まれ、Node ホストの実行
リクエストへ転送されます。

### ツール結果の永続化

ツール結果には、UI レンダリング、診断、
メディアルーティング、またはプラグイン所有のメタデータ向けの構造化された `details` を含められます。`details` はプロンプト内容ではなく、ランタイムメタデータとして扱います。

- OpenClaw は、メタデータがモデルコンテキストにならないよう、プロバイダーへの再送と Compaction
  入力の前に `toolResult.details` を除去します。
- 永続化されたセッションエントリには、上限内の `details` のみが保持されます。上限を超える詳細は、
  簡潔な要約と `persistedDetailsTruncated: true` に置き換えられます。
- `tool_result_persist` と `before_message_write` は、最終的な
  永続化上限が適用される前に実行されます。返す `details` は小さく保ち、
  プロンプトに関連するテキストを `details` のみに配置しないでください。モデルから見えるツール出力は
  `content` に配置してください。

## プロンプトとモデルのフック

新しいプラグインには、フェーズ固有のフックを使用します。

- `before_model_resolve`: 現在のプロンプトと添付ファイルの
  メタデータのみを受け取ります。`providerOverride` または `modelOverride` を返します。
- `agent_turn_prepare`: 現在のプロンプト、準備済みのセッション
  メッセージ、およびこのセッション用に取り出された「厳密に一度」のキュー済み注入を受け取ります。
  `prependContext` または `appendContext` を返します。
- `before_prompt_build`: 現在のプロンプトとセッションメッセージを受け取ります。
  `prependContext`、`appendContext`、`systemPrompt`、
  `prependSystemContext`、または `appendSystemContext` を返します。
- `heartbeat_prompt_contribution`: Heartbeat ターンでのみ実行され、
  `prependContext` または `appendContext` を返します。ユーザーが開始したターンを変更せずに、
  現在の状態を要約する必要があるバックグラウンドモニター向けです。

`before_agent_run` は、プロンプトの構築後、かつプロンプトローカルな画像の読み込みや
`llm_input` の観測を含む、あらゆるモデル入力の前に実行されます。現在のユーザー入力を `prompt` として受け取り、
さらに `messages` 内の読み込み済みセッション履歴と、アクティブなシステムプロンプトを受け取ります。モデルがプロンプトを読む前に実行を停止するには、
`{ outcome: "block", reason, message? }` を返します。`reason` は内部用で、
`message` はユーザー向けの置換テキストです。サポートされるのは `pass` と `block` の結果のみであり、
未サポートの決定形式はフェイルクローズされます。

実行がブロックされた場合、OpenClaw は `message.content` に置換テキストのみを保存し、
さらにブロックしたプラグイン ID やタイムスタンプなど、機密性のないブロックメタデータを保存します。
元のユーザーテキストは、トランスクリプトにも将来のコンテキストにも保持されません。
内部ブロック理由は機密情報として扱われ、トランスクリプト、履歴、ブロードキャスト、
ログ、診断ペイロードから除外されます。可観測性には、ブロッカー ID、結果、
タイムスタンプ、安全なカテゴリなど、サニタイズされたフィールドを使用してください。

`agent_end` を含むエージェントターンフックには、OpenClaw が
アクティブな実行を識別できる場合、`event.runId` が含まれます。同じ値は `ctx.runId` にもあります。Cron によって開始された
実行では、エージェントターンコンテキストに `ctx.jobId`（開始元の Cron ジョブ ID）も公開されるため、
フックはメトリクス、副作用、状態のスコープを特定の
スケジュール済みジョブに限定できます。`ctx.jobId` は `before_tool_call` ツールコンテキストには含まれません。

チャネルから開始された実行では、`ctx.channel` と `ctx.messageProvider` が
`discord` や `telegram` などのプロバイダーサーフェスを識別します。一方、`ctx.channelId` は、
OpenClaw がセッションキーまたは配信メタデータから導出できる場合の
会話ターゲット識別子です。

送信者の識別情報を利用できる場合、エージェントフックのコンテキストには次の情報も含まれます。

- `ctx.senderId` - チャネルスコープの送信者 ID（例: Feishu の `open_id`、Discord の
  ユーザー ID）。既知の送信者メタデータを持つユーザーメッセージから
  実行が開始された場合に設定されます。
- `ctx.chatId` - トランスポートネイティブな会話識別子（例: Feishu の
  `chat_id`、Telegram の `chat_id`）。開始元チャネルが
  ネイティブの会話 ID を提供する場合に設定されます。
- `ctx.channelContext.sender.id` - `ctx.senderId` と同じ送信者 ID です。プラグインがチャネル固有のフィールドで拡張できる、
  チャネル所有のオブジェクト内にあります。
- `ctx.channelContext.chat.id` - `ctx.chatId` と同じ会話 ID です。
  プラグインがチャネル固有のフィールドで拡張できる、チャネル所有の
  オブジェクト内にあります。

コアが定義するのは、ネストされた `id` フィールドのみです。受信ヘルパーを通じて
より豊富な送信者またはチャットメタデータを渡すチャネルプラグインは、
`openclaw/plugin-sdk/channel-inbound` から `PluginHookChannelSenderContext` または `PluginHookChannelChatContext` を
拡張できます。

```ts
declare module "openclaw/plugin-sdk/channel-inbound" {
  interface PluginHookChannelSenderContext {
    unionId?: string;
    userId?: string;
  }
}
```

チャネルプラグインは、受信 SDK ヘルパーを通じてこれらのフィールドを渡します。

```ts
buildChannelInboundEventContext({
  // ...
  channelContext: {
    sender: { id: senderOpenId, unionId, userId },
    chat: { id: chatId },
  },
});
```

これらのフィールドは任意であり、システムから開始された実行（Heartbeat、
Cron、Exec イベント）には存在しません。

`ctx.senderExternalId` は、古いプラグイン向けの非推奨のソース互換性フィールドとして
残されています。コアはこれを設定しません。新しいチャネル固有の送信者
識別情報は、モジュール拡張を通じて `ctx.channelContext.sender` の下に
配置する必要があります。

`agent_end` は観測フックです。Gateway と永続ハーネスのパスでは
ターン後に fire-and-forget 方式で実行されます。一方、短時間で終了するワンショット CLI パスでは、
信頼されたプラグインが終了時の可観測性データをフラッシュしたり状態を取得したりできるよう、
プロセスのクリーンアップ前にフックの Promise を待機します。フックランナーは 30 秒の
タイムアウトを適用するため、停止したプラグインや埋め込みエンドポイントによってフックの Promise が
無期限に保留されることはありません。タイムアウトはログに記録され、OpenClaw は処理を続行します。
プラグイン自身も独自の中断シグナルを使用していない限り、
プラグイン所有のネットワーク処理はキャンセルされません。

生のプロンプト、履歴、レスポンス、ヘッダー、リクエスト本文、
またはプロバイダーリクエスト ID を受け取るべきでないプロバイダー呼び出しテレメトリには、`model_call_started` と `model_call_ended` を使用します。これらのフックには、
`runId`、`callId`、`provider`、`model`、任意の `api`/`transport`、終了時の
`durationMs`/`outcome`、および OpenClaw が上限内のプロバイダーリクエスト ID ハッシュを導出できる場合の
`upstreamRequestIdHash` など、安定したメタデータが含まれます。ランタイムが
コンテキストウィンドウのメタデータを解決済みの場合、フックイベントとコンテキストには
`contextTokenBudget`（モデル／設定／エージェントの
上限適用後の有効なトークン予算）に加えて、より低い上限が適用された場合は
`contextWindowSource` と `contextWindowReferenceTokens` も含まれます。

`before_agent_finalize` は、ハーネスが自然な最終アシスタント回答を
受け入れようとしている場合にのみ実行されます。これは `/stop` のキャンセルパスではなく、
ユーザーがターンを中断した場合には実行されません。確定前にもう一度モデル処理を行うよう
ハーネスに要求するには `{ action: "revise", reason }` を返し、確定を強制するには `{ action:
"finalize", reason? }` を返します。結果を省略すると続行します。
ハンドラーのデフォルトの処理時間上限は 15s です。タイムアウト時、OpenClaw は失敗をログに記録し、
元の最終回答で処理を続行します。
Codex ネイティブの `Stop` フックは、OpenClaw の
`before_agent_finalize` の決定としてこのフックへ中継されます。

`action: "revise"` を返す場合、追加のモデル処理を上限付きかつ再実行安全にするため、
プラグインは `retry` メタデータを含められます。

```typescript
type BeforeAgentFinalizeRetry = {
  instruction: string;
  idempotencyKey?: string;
  maxAttempts?: number;
};
```

`instruction` は、ハーネスへ送信される修正理由に追加されます。
`idempotencyKey` により、同等の確定決定をまたいで同じプラグインリクエストの
再試行回数をホストがカウントできます。`maxAttempts` は、自然な最終回答で続行するまでに
ホストが許可する追加処理回数の上限を設定します。

生の会話フック（`before_model_resolve`、
`before_agent_reply`、`llm_input`、`llm_output`、`before_agent_finalize`、
`agent_end`、または `before_agent_run`）を必要とする非バンドルプラグインは、
次を設定する必要があります。

```json
{
  "plugins": {
    "entries": {
      "my-plugin": {
        "hooks": {
          "allowConversationAccess": true
        }
      }
    }
  }
}
```

プロンプトを変更するフックと、次ターンへの永続的な注入は、
`plugins.entries.<id>.hooks.allowPromptInjection=false` を使用してプラグインごとに無効化できます。

### セッション拡張と次ターンへの注入

ワークフロープラグインは、`api.session.state.registerSessionExtension(...)` を使用して
小さな JSON 互換のセッション状態を永続化し、Gateway の `sessions.pluginPatch` メソッドを通じて更新できます。
セッション行は、登録済みの拡張状態を `pluginExtensions` を通じて投影するため、
Control UI やその他のクライアントは、プラグインの内部実装を知らずに
プラグイン所有の状態をレンダリングできます。
`api.registerSessionExtension(...)` も引き続き機能しますが、
`api.session.state` 名前空間の使用が推奨され、非推奨になっています。

プラグインが、厳密に一度だけ次のモデルターンへ到達する
永続的なコンテキストを必要とする場合は、`api.session.workflow.enqueueNextTurnInjection(...)` を使用します
（トップレベルの `api.enqueueNextTurnInjection(...)` は同じ
動作を持つ非推奨のエイリアスです）。OpenClaw はプロンプトフックの前にキュー済みの注入を取り出し、
期限切れの注入を破棄し、プラグインごとに `idempotencyKey` で重複排除します。これは、
承認からの再開、ポリシー要約、バックグラウンドモニターの差分、および次のターンで
モデルから見える必要はあるものの、永続的なシステムプロンプトのテキストには
すべきでないコマンド継続に適した接点です。

クリーンアップのセマンティクスは契約の一部です。セッション拡張のクリーンアップと
ランタイムライフサイクルのクリーンアップコールバックは、`reset`、`delete`、`disable`、または
`restart` を受け取ります。ホストは、リセット／削除／無効化の際に、所有プラグインの永続的なセッション拡張
状態と保留中の次ターン注入を削除します。再起動時には永続的なセッション状態を
保持しつつ、クリーンアップコールバックによってプラグインは、古いランタイム世代の
スケジューラージョブ、実行コンテキスト、その他の帯域外リソースを解放できます。

## メッセージフック

チャネルレベルのルーティングと配信ポリシーには、メッセージフックを使用します。

- `message_received`: 受信コンテンツ、送信者、`threadId`、
  `messageId`、`senderId`、任意の実行／セッション相関、順序付きの `media`、
  およびメタデータを観測します。
- `message_sending`: `content` を書き換えるか、`{ cancel: true }` を返します。
- `reply_payload_sending`: 正規化された `ReplyPayload` オブジェクト
  （`presentation`、`delivery`、メディア参照、テキストを含む）を書き換えるか、
  `{ cancel: true }` を返します。
- `message_sent`: 最終的な成功または失敗を観測します。

音声のみの TTS 応答では、チャネルペイロードに表示可能なテキスト／キャプションがない場合でも、
`content` に非表示の読み上げトランスクリプトが含まれることがあります。
その `content` を書き換えても、フックから見えるトランスクリプトのみが更新され、
メディアキャプションとしてはレンダリングされません。

`reply_payload_sending` イベントには、ベストエフォートのリアルタイムな
ターン単位のモデル／使用量／コンテキストのスナップショットである `usageState` が含まれることがあります。永続的な配信、復旧された再送、
および正確な実行相関がない応答では省略されます。

メッセージフックのコンテキストは、利用可能な場合、安定した相関フィールドを公開します:
`ctx.sessionKey`、`ctx.runId`、`ctx.messageId`、`ctx.senderId`、`ctx.trace`、
`ctx.traceId`、`ctx.spanId`、`ctx.parentSpanId`、および `ctx.callDepth`。受信
および `before_dispatch` のコンテキストでは、チャネルに
可視性でフィルタリングされた引用メッセージデータがある場合、返信メタデータも公開されます: `replyToId`、`replyToIdFull`、
`replyToBody`、`replyToSender`、および `replyToIsQuote`。従来のメタデータを読み取る前に、
これらの第一級フィールドを優先してください。

チャネル固有のメタデータを使用する前に、型付きの `threadId` および `replyToId` フィールドを
優先してください。

受信クレームおよびメッセージ受信イベントは、正規の添付ファイル API として `media?:
PluginHookMediaFact[]` を公開します。各ファクトは
`path`、`url`、`contentType`、`kind`、`transcribed`、`messageId`、および
`workspaceDir` を保持できます。配列内の位置が添付ファイルの識別子です。リモート添付ファイルが
まだローカルにステージングされていない場合、`media` は省略され、
`mediaStagingPending: true` となり、`originalMedia` にはプロバイダー側の
ファクトが含まれます。後続のステージング済みイベントによって `media` が提供されるまでは、
`originalMedia.path` をローカルで読み取り可能なものとして扱わないでください。

単数形／複数形の `mediaPath`、`mediaUrl`、`mediaType`、`mediaPaths`、
`mediaUrls`、`mediaTypes`、および対応する `originalMedia*` メタデータプロパティは、
非推奨の互換性エイリアスです。新しいフックでは、型付きのトップレベル
配列を使用してください。

判定ルール:

- `message_sending` と `cancel: true` の組み合わせは終端です。
- `message_sending` と `cancel: false` の組み合わせは、判定なしとして扱われます。
- 書き換えられた `content` は、後続のフックによって
  配信がキャンセルされない限り、優先度の低いフックへ処理を継続します。
- `reply_payload_sending` は、ペイロードの正規化後、チャネルへの
  配信前に実行されます。これには、送信元チャネルへ戻される返信も含まれます。
  ハンドラーは順次実行され、各ハンドラーには優先度の高いハンドラーが生成した
  最新のペイロードが渡されます。
- `reply_payload_sending` ペイロードは、
  `trustedLocalMedia` などのランタイム信頼マーカーを公開しません。Plugin はペイロードの形状を編集できますが、
  ローカルメディアへの信頼を付与することはできません。
- `message_sending` は、キャンセル時に `cancelReason` と、上限付きの `metadata` を
  返せます。新しいメッセージライフサイクル API では、これは理由 `cancelled_by_message_sending_hook` を伴う
  抑止された配信結果として公開されます。従来の
  直接配信では、互換性のため引き続き空の結果配列を返します。
- `message_sent` は監視専用です。ハンドラーの失敗はログに記録され、
  配信結果には影響しません。

## インストールフック

運用者が管理する許可／ブロック判定には `security.installPolicy` を使用してください。この
ポリシーは OpenClaw の設定から実行され、CLI のインストールおよび更新パスを対象とし、
有効化されているにもかかわらず利用できない場合はフェイルクローズします。

`before_install` は Plugin ランタイムのライフサイクルフックです。これは、
Gateway を介したインストールフローなど、Plugin フックがすでに読み込まれている
OpenClaw プロセスでのみ、`security.installPolicy` の後に実行されます。Plugin が管理する
監視、警告、互換性チェックには有用ですが、インストールにおける企業またはホストの
主要なセキュリティ境界ではありません。`builtinScan` フィールドは互換性のため
イベントペイロードに残りますが、OpenClaw は組み込みのインストール時危険コードブロックを
実行しなくなったため、空の `ok` 結果になります。そのプロセスでインストールを停止するには、
追加の検出結果または `{ block: true, blockReason }` を返してください。

`block: true` は終端です。`block: false` は判定なしとして扱われます。ハンドラーの
失敗はフェイルクローズでインストールをブロックします。

## Gateway のライフサイクル

一般的な Plugin サービスの開始には `gateway_start` を、長時間実行されるリソースの
クリーンアップには `gateway_stop` を使用してください。`gateway_start` の実行時点では
Cron スケジューラーがまだ読み込み中の場合があるため、外部 Cron プロジェクションの
ベースライン信号として使用しないでください。

Plugin が管理するランタイムサービスでは、内部の `gateway:startup` フックに
依存しないでください。

`cron_reconciled` は、Gateway の Cron スケジューラーと終了時
ウォッチャーが永続状態を調整した後に発火します。初回
起動時と、設定の再読み込み中にスケジューラーが置き換えられた場合の両方で発火します。イベントは
`reason`（`startup` または `reload`）および有効な `enabled` 状態を報告します。Cron が無効でも
`enabled: false` とともに発火するため、外部プロジェクションは
古いウェイクを消去できます。調整を完了した正確なスケジューラーインスタンスには
`ctx.getCron?.()` を使用してください。後続の再読み込みによってそのコールバックの対象が変更されることはありません。
`ctx.abortSignal` は同じスケジューラースナップショットを所有します。Gateway は、
新しいスケジューラーが準備されるか、シャットダウンが開始されると直ちにこれを中止します。あらゆる
永続的な副作用にこれを渡し、中止後はそのスナップショットを受け入れないでください。
これはスケジューラーのライフサイクル信号であり、Plugin のアクティベーション信号ではありません。
Plugin のみのホットリロードでは再発火しません。新たに有効化されたコンシューマーが
最初のベースラインを受け取るのは、次回のスケジューラー置換時または Gateway 起動時です。

他の監視フックと同様に、`gateway_start` および `cron_reconciled` のコールバックは
重複して実行される可能性があります。両方のハンドラーが Plugin の初期化を共有する場合は、
コールバックの順序に依存せず、Plugin ローカルの準備完了 Promise を使用して調整してください。

`cron_changed` は、Gateway が管理する Cron ライフサイクルイベントに対して、
`added`、`updated`、`removed`、`started`、`finished`、
および `scheduled` の理由を網羅する型付きイベントペイロードとともに発火します。イベントは、
`PluginHookGatewayCronJob` スナップショット（存在する場合は `state.nextRunAtMs`、`state.lastRunStatus`、
および `state.lastError` を含む）に加えて、
`not-requested` | `delivered` | `not-delivered` | `unknown` の
`PluginHookGatewayCronDeliveryStatus` を保持します。削除イベントはコミット後に発火します。つまり、
永続的な削除が成功した後にのみ発火し、外部スケジューラーが状態を調整できるよう、
削除されたジョブのスナップショットも引き続き保持します。

`scheduled` イベントはコミット後に発火します。既存ジョブの有効な `nextRunAtMs` が
永続的な書き込みの成功によって変更された後にのみ発火し、そのジョブの明示的な
`added`、`updated`、または `removed` ライフサイクルイベントは除外されます。トップレベルの
`event.nextRunAtMs` はコミット済みの次回ウェイクです。これが存在しない場合、そのジョブに
次回ウェイクはありません。これらのイベントは、順序付きの差分ログではなく調整のヒントとして
扱ってください。これらをまとめることのできるヒントとして使用し、`cron_reconciled` が最後に取得した
スケジューラーを再読み込みしてください。`cron_changed` コンテキストからスケジューラーを採用しないでください。
期限チェックと実行については、OpenClaw を信頼できる唯一の情報源として維持してください。

### 安全な外部 Cron プロジェクション

Cron イベントの差分を転送する代わりに、完全なウェイクスナップショットをプロジェクションします。
外部アダプターの `replaceAll` 操作はアトミックかつ冪等でなければならず、
ホストがスナップショットを永続的に受け入れた後にのみ解決されなければなりません。また、
渡された中止シグナルにも従う必要があります。永続的に受け入れられる前にシグナルが中止された場合、
アダプターはそのスナップショットを受け入れてはなりません。

このパターンでは、最新状態のワーカーを常に 1 つだけ実行中に保ちます。スケジューラーインスタンスを
採用するのは `cron_reconciled` のみです。`cron_changed` はそのワーカーに
信頼できるインスタンスの再読み込みを要求するだけなので、遅れて届いたヒントによって古いスケジューラーが
復元されることはありません。新しいリビジョンは、古いスナップショットが受け入れられる前に、
アクティブなホスト試行を中止します。

```typescript
import { setTimeout as sleep } from "node:timers/promises";
import type { OpenClawPluginApi } from "openclaw/plugin-sdk/plugin-entry";

type ExternalWake = { jobId: string; runAtMs: number };

type ExternalWakeHost = {
  replaceAll(wakes: readonly ExternalWake[], options: { signal: AbortSignal }): Promise<void>;
  close(): Promise<void>;
};

type CronReader = {
  list(options: { includeDisabled: true }): Promise<
    Array<{
      id: string;
      enabled?: boolean;
      state?: { nextRunAtMs?: number };
    }>
  >;
};

export function registerCronProjection(api: OpenClawPluginApi, host: ExternalWakeHost) {
  const lifecycle = new AbortController();
  let cron: CronReader | undefined;
  let enabled = false;
  let hasBaseline = false;
  let reconciliationSignal: AbortSignal | undefined;
  let requestedRevision = 0;
  let appliedRevision = 0;
  let worker = Promise.resolve();
  let activeAttempt: AbortController | undefined;

  const projectLatest = async () => {
    let retryMs = 1_000;

    while (!lifecycle.signal.aborted && appliedRevision < requestedRevision) {
      const ownerSignal = reconciliationSignal;
      if (!ownerSignal || ownerSignal.aborted) {
        return;
      }
      const targetRevision = requestedRevision;
      const attempt = new AbortController();
      const signal = AbortSignal.any([lifecycle.signal, ownerSignal, attempt.signal]);
      activeAttempt = attempt;

      try {
        const jobs = enabled && cron ? await cron.list({ includeDisabled: true }) : [];
        if (signal.aborted || targetRevision !== requestedRevision) {
          continue;
        }
        const wakes = jobs
          .flatMap((job): ExternalWake[] => {
            const runAtMs = job.enabled === false ? undefined : job.state?.nextRunAtMs;
            return runAtMs === undefined ? [] : [{ jobId: job.id, runAtMs }];
          })
          .sort((a, b) => a.runAtMs - b.runAtMs || a.jobId.localeCompare(b.jobId));

        await host.replaceAll(wakes, { signal });
        if (signal.aborted || targetRevision !== requestedRevision) {
          continue;
        }
        appliedRevision = targetRevision;
        retryMs = 1_000;
      } catch {
        if (lifecycle.signal.aborted || ownerSignal.aborted) {
          return;
        }
        if (attempt.signal.aborted) {
          continue;
        }
        api.logger.warn(`external cron projection failed; retrying in ${retryMs}ms`);
        try {
          await sleep(retryMs, undefined, { signal });
        } catch {
          if (lifecycle.signal.aborted) {
            return;
          }
          if (attempt.signal.aborted) {
            continue;
          }
        }
        retryMs = Math.min(retryMs * 2, 30_000);
      } finally {
        if (activeAttempt === attempt) {
          activeAttempt = undefined;
        }
      }
    }
  };

  const requestProjection = () => {
    const targetRevision = ++requestedRevision;
    activeAttempt?.abort();
    worker = worker.then(async () => {
      if (!lifecycle.signal.aborted && appliedRevision < targetRevision) {
        await projectLatest();
      }
    });
    return worker;
  };

  api.on("cron_reconciled", (event, ctx) => {
    const reconciledCron = ctx.getCron?.();
    if (event.enabled && !reconciledCron) {
      api.logger.warn("cron reconciliation did not expose a scheduler");
      return;
    }
    cron = reconciledCron;
    enabled = event.enabled;
    hasBaseline = true;
    reconciliationSignal = ctx.abortSignal;
    return requestProjection();
  });

  api.on("cron_changed", () => {
    if (hasBaseline) {
      return requestProjection();
    }
  });

  api.on("gateway_stop", async () => {
    lifecycle.abort();
    await worker;
    await host.close();
  });
}
```

`cron_reconciled` が `enabled: false` を報告すると、同じパスが
`replaceAll([])` を呼び出し、古い外部ウェイクを消去します。この例の再試行／バックオフは
プロセスローカルであり、ランタイムアダプターの失敗を一時的なものとして扱います。再試行不能な
設定は登録前に検証してください。OpenClaw は Plugin フックの効果に対する
アウトボックスを提供しません。永続的に受け入れられる前にプロセスが終了した場合、
次回の Gateway 起動時に新しい信頼できる `cron_reconciled` スナップショットが発行されます。
`gateway_stop` は実行中のホスト処理を中止し、ワーカーの完了を待ってから、
アダプターを閉じます。

## 今後の非推奨化

フックに隣接するいくつかのサーフェスは非推奨ですが、引き続きサポートされています。次回の
メジャーリリース前に移行してください:

- `inbound_claim` および `message_received` ハンドラー内の**プレーンテキストのチャネルエンベロープ**。
  フラットなエンベロープテキストを解析する代わりに、`BodyForAgent` と構造化されたユーザーコンテキストブロックを
  読み取ってください。詳しくは、
  [プレーンテキストのチャネルエンベロープ → BodyForAgent](/ja-JP/plugins/sdk-migration#active-deprecations)を参照してください。
- **`subagent_spawning`** は古い plugins との互換性のために残されていますが、
  新しい plugins はこれからスレッドルーティングを返さないでください。コアは、
  `subagent_spawned` が発火する前に、チャネルセッションバインディングアダプターを通じて
  `thread: true` サブエージェントバインディングを準備します。
- **`deactivate`** は、2026-08-16 より後まで非推奨のクリーンアップ互換エイリアスとして
  残されます。新しい plugins は `gateway_stop` を使用してください。
- **`before_tool_call` 内の `onResolution`** は、自由形式の `string` の代わりに、
  型付けされた `PluginApprovalResolution` ユニオン（`allow-once` / `allow-always` / `deny` /
  `timeout` / `cancelled`）を使用するようになりました。
- **`api.registerSessionExtension` / `api.enqueueNextTurnInjection`** は、
  トップレベルの互換エイリアスとして残されています。新しい plugins は
  `api.session.state.registerSessionExtension(...)` および
  `api.session.workflow.enqueueNextTurnInjection(...)` を使用してください。

メモリ機能の登録、プロバイダーの思考プロファイル、外部認証プロバイダー、プロバイダー検出型、タスクランタイムの
アクセサー、`command-auth` → `command-status` への名称変更を含む完全な一覧については、
[Plugin SDK の移行 → 有効な非推奨項目](/ja-JP/plugins/sdk-migration#active-deprecations)を参照してください。

## 関連項目

- [Plugin SDK の移行](/ja-JP/plugins/sdk-migration) - 有効な非推奨項目と削除スケジュール
- [plugins の構築](/ja-JP/plugins/building-plugins)
- [Plugin SDK の概要](/ja-JP/plugins/sdk-overview)
- [Plugin のエントリーポイント](/ja-JP/plugins/sdk-entrypoints)
- [内部フック](/ja-JP/automation/hooks)
- [Plugin アーキテクチャの内部構造](/ja-JP/plugins/architecture-internals)
