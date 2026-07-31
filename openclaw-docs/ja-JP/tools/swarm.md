---
read_when:
    - 複数のエージェントに作業を分散する Code Mode スクリプトが必要な場合
    - 構造化された子結果、決定ゲート、または最初の完了を採用するパイプラインが必要である場合
    - tools.swarm の制限を有効化または調整しています
    - セッションダッシュボードでコレクターの子を確認する場合
sidebarTitle: Swarm
summary: 構造化された結果、制限されたファンアウト、リアルタイムの進捗表示を備えた Code Mode スクリプトから、複数のサブエージェントを並行してオーケストレーションする
title: 群れ
x-i18n:
    generated_at: "2026-07-26T10:34:12Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: f0bec17da7a2e144df35189a65d9b35d829815b545a4bb89652e6a681ca971a9
    source_path: tools/swarm.md
    workflow: 16
---

Swarm は、[Code Mode](/tools/code-mode) スクリプトから多数のサブエージェントをオーケストレーションするための、実験的なオプトイン方式です。`Promise.all`、`while`、`if` など、通常の JavaScript または TypeScript の制御フローを使用して、作業のファンアウト、結果の収集、意思決定を行います。

グラフ DSL や独立したワークフロー形式はありません。プログラム自体がオーケストレーションです。Swarm はそのプログラムに、await 可能なコレクター子エージェント、構造化された結果、制限付き並行処理、進捗レポートを追加します。

## Swarm を有効にする

推奨される方法は、Control UI の **Settings → Labs → Swarm** です。このトグルはすぐに反映され、設定に `tools.swarm.enabled` を書き込みます。

`openclaw.json` で Swarm を直接有効にすることもできます。

```json5
{
  tools: {
    swarm: {
      enabled: true,
      maxConcurrent: 8,
      maxChildrenPerGroup: 50,
      maxTotalPerGroup: 200,
      waitTimeoutSecondsMax: 600,
      defaultAgentId: "",
    },
  },
}
```

真偽値の省略記法では、その他のすべての値をデフォルトのまま、機能を有効または無効にできます。

```json5
{
  tools: {
    swarm: true,
  },
}
```

| フィールド                   | デフォルト | 説明                                                                                                                    |
| ----------------------- | ------- | ------------------------------------------------------------------------------------------------------------------------------ |
| `enabled`               | `false` | コレクターモードのスポーンオプション、`agents_wait`、Code Mode の `agents.*` ゲスト API を公開します。                                   |
| `maxConcurrent`         | `8`     | 1 つの Swarm グループ内で同時に実行できるコレクター子エージェントの最大数です。追加で受け付けた子エージェントは FIFO 順でキューに入ります。          |
| `maxChildrenPerGroup`   | `50`    | 1 つのグループ内で存続できるコレクター子エージェントの最大数です。                                                                                  |
| `maxTotalPerGroup`      | `200`   | グループが存続期間中にスポーンできるコレクター子エージェントの最大数です。これは暴走スポーンを防ぐ最後の安全策です。                            |
| `waitTimeoutSecondsMax` | `600`   | 1 回の `agents_wait` 呼び出しで受け付ける最大タイムアウトです。呼び出しのデフォルトは 30 秒です。                                            |
| `defaultAgentId`        | `""`    | スポーンで `agentId` が省略されたときに使用する対象エージェントです。空の値では要求元エージェントを使用します。既存のサブエージェント許可リストが適用されます。 |

数値は正の整数でなければなりません。OpenClaw は、`maxConcurrent` を `1`～`1000`、`maxChildrenPerGroup` を `1`～`10000`、`maxTotalPerGroup` を `1`～`100000`、`waitTimeoutSecondsMax` を `1`～`86400` の範囲に制限します。

設定済みの 1 エージェントについて、`agents.entries.*.tools.swarm` で Swarm を上書きできます。エージェントごとのオブジェクトは、最上位の `tools.swarm` オブジェクトにマージされ、同名の値を上書きします。

## 要件

`agents.run`、`phase`、`log` のゲストグローバルには、Swarm と OpenClaw Code Mode の両方が必要です。

```json5
{
  tools: {
    codeMode: true,
    swarm: true,
  },
}
```

Code Mode には、`sessions_spawn` への実効的なアクセス権も必要です。ツールプロファイル、許可／拒否ポリシー、プロバイダールール、サンドボックスポリシーによって、そのツールが除外される場合があります。スクリプトから `sessions_spawn` が利用できないと報告された場合は、[Code Mode の有効化](/tools/code-mode#activation)および[サブエージェント](/ja-JP/tools/subagents)を参照してください。

`defaultAgentId` および実行ごとの `agentId` の値には、要求元の `subagents.allowAgents` ポリシーで許可された設定済みの対象を指定する必要があります。OpenClaw は別のエージェントにフォールバックせず、不明または許可されていない対象を拒否します。

## Swarm スクリプトを記述する

Swarm を有効にすると、Code Mode は次のゲスト API を公開します。

```typescript
type AgentRunOptions = {
  label?: string;
  model?: string;
  thinking?: string;
  fastMode?: boolean | "auto";
  agentId?: string;
  schema?: Record<string, unknown>;
  phase?: string;
};

agents.run(prompt: string, options?: AgentRunOptions & { schema?: undefined }): Promise<string>;
agents.run<T>(prompt: string, options: AgentRunOptions & { schema: Record<string, unknown> }): Promise<T>;
phase(title: string): void;
log(message: string): void;
```

`schema` がない場合、`agents.run()` は子エージェントの最終テキストに解決されます。JSON Schema がある場合は、子エージェントの `structured_output` ツールを通じて送信された値に解決されます。失敗、強制終了、タイムアウト、またはスキーマ不正となった子エージェントは、`SwarmAgentError` で Promise を拒否します。生成された正確な宣言と短いオーケストレーションのイディオムは、Code Mode 内の `API.read("agents.d.ts")` から確認できます。

ダッシュボードとサイドバーで識別しやすい子エージェント名を付けるには、`label` を使用します。子エージェントが開始する直前にフェーズを公開するには、オプションで `phase` を使用します。複数の子エージェントが同じ段階に属する場合は、`phase()` を呼び出します。`log()` は短い進捗メモを公開します。進捗呼び出しは fire-and-forget 方式であり、UI が利用できなくてもスクリプトを遅延させません。

### 構造化された結果を使用して並列にファンアウトする

この例では、トピックごとに 1 つのリサーチ担当を起動し、すべての完了を待ってから、最後の子エージェントに構造化レポートの統合を依頼します。

```javascript
const reportSchema = {
  type: "object",
  properties: {
    finding: { type: "string" },
    evidence: { type: "array", items: { type: "string" } },
    confidence: { type: "number" },
  },
  required: ["finding", "evidence", "confidence"],
  additionalProperties: false,
};

const topics = ["authentication", "storage", "recovery"];
phase("Independent review");

const reports = await Promise.all(
  topics.map((topic) =>
    agents.run(`Review the ${topic} path. Return one finding with evidence.`, {
      label: `review-${topic}`,
      thinking: "high",
      fastMode: "auto",
      schema: reportSchema,
    }),
  ),
);

phase("Synthesis");
log(`Collected ${reports.length} independent reports.`);

return await agents.run(
  `Reconcile these reports and explain disagreements:\n${JSON.stringify(reports)}`,
  { label: "synthesis" },
);
```

`Promise.all` はファンアウトとファンインの境界です。OpenClaw はグループに対して最大 `maxConcurrent` 個の子エージェントを開始し、残りを送信順でキューに入れます。

Code Mode は、`tools.codeMode.maxPendingToolCalls`（デフォルト `16`、最大 `128`）によって、ゲストブリッジの同時呼び出し数も別途制限します。非常に大きなグループでは、その制限を下回るサイズのバッチで起動し、`phase()`、`log()`、子エージェントの待機遷移用に余裕を残してください。`maxConcurrent` は実行中の子エージェント数を制限するものであり、ゲストブリッジ呼び出しの上限を引き上げるものではありません。

### 判断ゲートをループする

各パスで次のパスが必要かどうかを判断する場合は、上限を設定した `while` ループを使用します。

```javascript
const gateSchema = {
  type: "object",
  properties: {
    ready: { type: "boolean" },
    reason: { type: "string" },
    nextAction: { type: "string" },
  },
  required: ["ready", "reason", "nextAction"],
  additionalProperties: false,
};

let pass = 0;
let decision = { ready: false, reason: "Not checked", nextAction: "Review" };

while (!decision.ready && pass < 4) {
  pass += 1;
  phase(`Decision pass ${pass}`);
  decision = await agents.run(
    `Check whether the release evidence is complete. Previous decision: ${JSON.stringify(decision)}`,
    {
      label: `release-gate-${pass}`,
      schema: gateSchema,
    },
  );
  log(decision.reason);
}

if (!decision.ready) {
  throw new Error(`Gate still closed after ${pass} passes: ${decision.nextAction}`);
}

return decision;
```

判断ループには必ず上限を設定してください。`maxTotalPerGroup` は最後の安全策であり、明確な停止条件の代わりにはなりません。

### 最初に完了した子エージェントを処理する

`agents.run()` は通常の Promise を返すため、`Promise.race` は最初に完了した Code Mode の子エージェントに応答できます。低レベルのツールを呼び出すハーネスでは、`agents_wait` が同じ最初の完了境界を提供します。要求した実行のうち少なくとも 1 つが完了するか、制限付きタイムアウトが経過するとすぐに返ります。完全なドレインループについては、[他のハーネスから Swarm を使用する](#use-swarm-from-other-harnesses)を参照してください。

## コレクター子エージェントの動作

コレクター子エージェントは、完了経路が異なる通常の分離されたサブエージェントセッションです。親セッションに返信を通知または誘導する代わりに、親が待機できる永続的なコレクター結果を書き込みます。

対象エージェントは次の順序で解決されます。

1. スポーンまたは `agents.run()` 呼び出しの `agentId`。
2. `tools.swarm.defaultAgentId`。
3. 要求元エージェント。

Swarm の子エージェントに、より小さなツールサーフェス、より安価なモデル、またはより厳格なサンドボックスポリシーが必要な場合は、専用の軽量ワーカーエージェントが役立ちます。OpenClaw には組み込みの `worker` エージェント ID は付属していません。デフォルトとして指定する前に設定してください。そのワーカーをスポーン可能にしつつ、自身の最上位セッションから Swarm を開始できないようにするには、エージェントごとの設定で `tools.swarm: false` を使用して制限を強化します。

```json5
{
  tools: { swarm: { enabled: true, defaultAgentId: "worker" } },
  agents: {
    list: [
      {
        id: "main",
        default: true,
        subagents: { allowAgents: ["worker"] },
      },
      { id: "worker", tools: { swarm: false } },
    ],
  },
}
```

コレクターの承認はフェイルクローズします。子エージェントがオペレーター承認プロンプトを開くことはありません。承認を必要とするツール操作は拒否されます。子エージェントはその拒否を結果で報告できるため、スクリプトは次の対応を判断できます。

構造化出力では、OpenClaw は合成 `structured_output` ツールを子エージェントに追加し、そのペイロードを指定された JSON Schema に対して検証します。無効または欠落したペイロードには、修正を促す通知が 1 回送られます。再試行後も検証に失敗した場合、コレクターの完了結果には子エージェントの生テキストが保持され、`structured` は未設定のままとなり、`schemaError` が含まれます。低レベルの `agents_wait` の結果では、明示的な復旧ロジックのためにこれらのフィールドを確認できます。

### 子エージェントはリーフである

Swarm の子エージェントは、デフォルトではリーフです。汎用の `agents.defaults.subagents.maxSpawnDepth` ガードにより、デフォルト深度 `1` では子エージェントが自身の子エージェントをスポーンできません。通常のオーケストレーションのイディオムは、子エージェントからさらに作業をスポーンするのではなく、作業を親に返すことです。

```javascript
const plan = await agents.run("Plan this job as independent tasks.", {
  schema: {
    type: "object",
    properties: { tasks: { type: "array", items: { type: "string" } } },
    required: ["tasks"],
    additionalProperties: false,
  },
});
return await Promise.all(plan.tasks.map((task) => agents.run(task)));
```

ネストされたサブエージェントは、`agents.defaults.subagents.maxSpawnDepth` を通じてオペレーターがオプトインする機能であり、Swarm での使用は推奨されません。グループ上限、予算、可観測性はすべて、フラットなコレクターグループを前提としています。

各子エージェントには 1 つのアドミッション所有者があります。通知および対話型の子エージェントは `agents.defaults.subagents.maxChildrenPerAgent`（デフォルト `5`）を使用し、コレクター子エージェントをカウントしません。コレクター子エージェントは `maxChildrenPerGroup` と `maxTotalPerGroup` のみを使用し、セッションごとの子エージェント予算を消費しません。スポーン深度ガードは、引き続き両方のモードに適用されます。

アドミッション後、`maxConcurrent` を超える子エージェントは、グローバルなサブエージェントレーン内にネストされた各 Swarm グループで FIFO 順にキューに入ります。これらの並行処理レイヤーは、作業を拒否せずキューに入れます。いずれかのグループ上限を超えるコレクタースポーンは、エラー内に該当する設定キーを含めて拒否されます。

## Swarm を観察する

Swarm の実行中に、Control UI で親セッションのダッシュボードを開きます。Swarm ウィジェットは、アクティブな各コレクターグループについて、キュー待ち、実行中、完了、失敗の状態を子エージェントごとの 1 つのドットとして表示します。ラベルはドットのツールチップに表示されるため、短く安定したラベルを使うと、大規模な Swarm を読み取りやすくなります。

セッションサイドバーでは、通常の親子ツリーが維持されます。親の行を展開すると、Swarm の階層を失うことなく、コレクター子エージェントを調査したり、そのトランスクリプトを開いたりできます。

コレクターの結果は、そのグループがアーカイブされるまで待機対象として維持されます。すべての
メンバーが保持期限に達すると、OpenClaw はグループの子を
一括でアーカイブするため、完了した Swarm がライブセッションツリーに残りません。

## 他のハーネスから Swarm を使用する

OpenClaw Code Mode がなくても Swarm を使用できます。そのコアツールは
ハーネスに依存しません。`sessions_spawn({ collect: true })` でコレクターの子を開始し、
回数を制限した `agents_wait` 呼び出しで結果を取得します。

Codex Code Mode は、対象となる動的 OpenClaw ツールを
`tools.*` 配下に自動的に公開します。OpenClaw の QuickJS ゲスト API を使用せず、
`tools.codeMode` も必要としませんが、`tools.swarm` は引き続き有効にする必要があります。Codex ハーネスの
`agents_wait` 呼び出しでは、最大 600 秒のタイムアウトが完全にサポートされます。

現在サポートされている Codex ランタイムでは、動的 OpenClaw ツールの結果は
JSON テキストとして Code Mode に渡されます。フィールドを読み取る前に各結果を解析してください。また、Codex は
動的ツール呼び出しを直列化するため、`Promise.all` で複数の
`sessions_spawn` 呼び出しを同時に送信することはできません。コレクターは回数を制限したループで起動してください。
後続の起動が送信されている間も、すでに受理された子は実行を継続できます。

```javascript
function parseToolResult(value) {
  if (typeof value !== "string") return value;
  return JSON.parse(value);
}

const tasks = [
  "認証パスを確認する。",
  "ストレージパスを確認する。",
  "リカバリーパスを確認する。",
];
const launches = [];

for (const [index, task] of tasks.entries()) {
  const launch = parseToolResult(
    await tools.sessions_spawn({
      task,
      collect: true,
      label: `review-${index + 1}`,
    }),
  );
  if (launch.status !== "accepted") {
    throw new Error(launch.error ?? "コレクターの生成が受理されませんでした。");
  }
  launches.push(launch);
}

const pending = new Set(launches.map((launch) => launch.runId));
const completed = [];

while (pending.size > 0) {
  const ids = [...pending].slice(0, 1000);
  const batch = parseToolResult(
    await tools.agents_wait({
      ids,
      timeoutSeconds: 30,
    }),
  );

  // まだ確認していない ID の後ろに、この制限付きウィンドウを移動する。
  for (const runId of ids) {
    if (pending.delete(runId)) pending.add(runId);
  }

  for (const item of batch.completed) {
    pending.delete(item.runId);
    if (item.status !== "done") {
      throw new Error(item.schemaError ?? item.result ?? `${item.runId}: ${item.status}`);
    }
    completed.push(item); // 各結果が完了したらすぐに処理する。
  }

  for (const failure of batch.errors ?? []) {
    pending.delete(failure.runId);
    throw new Error(`${failure.runId}: ${failure.error}`);
  }
}

return completed;
```

各 `agents_wait` 呼び出しは、1～1000 個の実行 ID を受け取ります。戻り値は次のとおりです。

```typescript
type AgentsWaitResult = {
  completed: Array<{
    runId: string;
    status: "done" | "failed" | "killed" | "timeout";
    result: string;
    structured?: unknown;
    schemaError?: string;
    sessionKey: string;
    label?: string;
    usage?: { inputTokens: number; outputTokens: number };
  }>;
  pending: string[];
  errors?: Array<{
    runId: string;
    error: "not_found" | "not_owner";
  }>;
};
```

要求した子のいずれかがすでに完了している場合、
保留中の子が少なくとも 1 つ完了した場合、有効な保留中 ID がなくなった場合、
またはタイムアウトした場合、この呼び出しは直ちに返ります。完了レコードはべき等であるため、
すでに完了した実行 ID を渡すと、その結果が再び返されます。コレクターを待機できるのは、
生成元のセッションまたは承認された親チェーンのみです。

これはビジーなステータスループではなく、制限付きのロングポーリングです。`pending` が空になるまで、
残りの実行 ID だけを渡し続けてください。コレクターモードは OpenClaw ネイティブの
サブエージェントをサポートしますが、ACP ランタイム、スレッドのバインド、可視セッション、
永続セッションモードはサポートしません。

## 制限とロードマップ

Swarm v1 は、単発実行のコレクターの子を実行します。計画中の `agents.session()` API では、
状態を保持するマルチターンワーカーが追加される予定です。現在、子はローカル
Gateway のサブエージェントレーンで実行されます。クラウドへの配置は、明示的な生成
オプションとして計画されています。保存済みワークフロー定義とグラフ DSL は、Swarm の
現在の方針には含まれていません。

## 関連項目

- [Code Mode](/tools/code-mode)：QuickJS ゲストランタイムと有効化ルール
- [サブエージェント](/ja-JP/tools/subagents)：子のポリシー、分離、セッション動作
- [マルチエージェントサンドボックスツール](/ja-JP/tools/multi-agent-sandbox-tools)：エージェントごとの制限
- [ツールの概要](/ja-JP/tools)：ツールプロファイルとポリシールーティング
