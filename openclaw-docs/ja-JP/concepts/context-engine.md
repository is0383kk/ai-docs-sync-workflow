---
read_when:
    - OpenClaw がモデルコンテキストをどのように構成するかを理解したい場合
    - レガシーエンジンとPluginエンジンを切り替えています
    - コンテキストエンジン Plugin を構築する
sidebarTitle: Context engine
summary: コンテキストエンジン：差し替え可能なコンテキスト構築、Compaction、サブエージェントのライフサイクル
title: コンテキストエンジン
x-i18n:
    generated_at: "2026-07-26T09:37:32Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 721780790dacebec44e3c7540b225bd853ee66bf5ae066b84df4344614d93a62
    source_path: concepts/context-engine.md
    workflow: 16
---

**コンテキストエンジン**は、OpenClaw が各実行のモデルコンテキストを構築する方法を制御します。どのメッセージを含めるか、古い履歴をどのように要約するか、サブエージェント境界をまたいでコンテキストをどのように管理するかを決定します。

OpenClaw には組み込みの `legacy` エンジンが付属し、デフォルトで使用されます。アセンブリ、Compaction、またはセッションをまたぐ記憶の動作を変更したい場合にのみ、Plugin エンジンをインストールして選択してください。

## クイックスタート

<Steps>
  <Step title="有効なエンジンを確認する">
    ```bash
    openclaw doctor
    # または設定を直接確認:
    cat ~/.openclaw/openclaw.json | jq '.plugins.slots.contextEngine'
    ```
  </Step>
  <Step title="Plugin エンジンをインストールする">
    コンテキストエンジンの Plugin は、他の OpenClaw Plugin と同じ方法でインストールします。

    <Tabs>
      <Tab title="npm から">
        ```bash
        openclaw plugins install @martian-engineering/lossless-claw
        ```
      </Tab>
      <Tab title="ローカルパスから">
        ```bash
        openclaw plugins install -l ./my-context-engine
        ```
      </Tab>
    </Tabs>

  </Step>
  <Step title="エンジンを有効化して選択する">
    ```json5
    // openclaw.json
    {
      plugins: {
        slots: {
          contextEngine: "lossless-claw", // Plugin に登録されたエンジン ID と一致する必要があります
        },
        entries: {
          "lossless-claw": {
            enabled: true,
            // Plugin 固有の設定をここに記述します（Plugin のドキュメントを参照）
          },
        },
      },
    }
    ```

    インストールと設定の完了後、Gateway を再起動します。

  </Step>
  <Step title="レガシーに戻す（任意）">
    `contextEngine` を `"legacy"` に設定します（またはキーを完全に削除します。デフォルトは `"legacy"` です）。
  </Step>
</Steps>

## 仕組み

OpenClaw がモデルプロンプトを実行するたびに、コンテキストエンジンはライフサイクルの4つの時点で処理に参加します。

<AccordionGroup>
  <Accordion title="1. 取り込み">
    セッションに新しいメッセージが追加されたときに呼び出されます。エンジンは、そのメッセージを独自のデータストアに保存またはインデックス化できます。
  </Accordion>
  <Accordion title="2. アセンブル">
    各モデル実行の前に呼び出されます。エンジンは、トークン予算内に収まる順序付きメッセージセット（および任意の `systemPromptAddition`）を返します。
  </Accordion>
  <Accordion title="3. Compact">
    コンテキストウィンドウがいっぱいになったとき、またはユーザーが `/compact` を実行したときに呼び出されます。エンジンは古い履歴を要約して空き領域を確保します。
  </Accordion>
  <Accordion title="4. ターン後">
    実行の完了後に呼び出されます。エンジンは状態の永続化、バックグラウンドでの Compaction の開始、またはインデックスの更新を行えます。
  </Accordion>
</AccordionGroup>

エンジンは、ブートストラップ後、ターンの正常完了後、または Compaction 後にトランスクリプトを保守するための任意の `maintain()` メソッドも実装できます（`runtimeContext.rewriteTranscriptEntries()` を介した安全な書き換え）。応答をブロックせず遅延処理として実行するには、`info.turnMaintenanceMode: "background"` を設定します。

バンドルされた非 ACP Codex ハーネスでは、OpenClaw はアセンブル済みコンテキストを Codex の開発者向け指示と現在のターンのプロンプトへ投影することで、同じライフサイクルを適用します。Codex は引き続き、ネイティブのスレッド履歴とネイティブのコンパクターを管理します。

### サブエージェントのライフサイクル（任意）

OpenClaw は、2つの任意のサブエージェントライフサイクルフックを呼び出します。

<ParamField path="prepareSubagentSpawn" type="method">
  子の実行が開始する前に、共有コンテキスト状態を準備します。このフックは、親と子のセッションキー、`contextMode`（`isolated` または `fork`）、利用可能なトランスクリプト ID／ファイル、および任意の TTL を受け取ります。ロールバックハンドルを返した場合、準備の成功後に生成が失敗すると、OpenClaw がそのハンドルを呼び出します。`lightContext` を要求し、`contextMode="isolated"` に解決されるネイティブなサブエージェント生成では、このフックを意図的にスキップします。これにより、子はコンテキストエンジンが管理する生成前状態を使用せず、軽量なブートストラップコンテキストから開始します。
</ParamField>
<ParamField path="onSubagentEnded" type="method">
  サブエージェントセッションが完了または掃除されたときにクリーンアップします。
</ParamField>

### システムプロンプトへの追加

`assemble` メソッドは、`systemPromptAddition` 文字列を返すことができます。OpenClaw はこれを実行時のシステムプロンプトの先頭に追加します。これにより、静的なワークスペースファイルを必要とせずに、エンジンが動的な記憶ガイダンス、検索指示、またはコンテキスト対応のヒントを注入できます。

## レガシーエンジン

組み込みの `legacy` エンジンは、OpenClaw の元の動作を維持します。

- **取り込み**：何もしません（セッションマネージャーがメッセージの永続化を直接処理します）。
- **アセンブル**：そのまま渡します（ランタイム内の既存のサニタイズ → 検証 → 制限パイプラインがコンテキストのアセンブリを処理します）。
- **Compact**：組み込みの要約 Compaction に委譲します。これは古いメッセージの単一の要約を作成し、最近のメッセージをそのまま維持します。
- **ターン後**：何もしません。

レガシーエンジンはツールを登録せず、`systemPromptAddition` も提供しません。

`plugins.slots.contextEngine` が設定されていない場合（または `"legacy"` に設定されている場合）、このエンジンが自動的に使用されます。

## Plugin エンジン

Plugin API を使用して、Plugin からコンテキストエンジンを登録できます。

```ts
import { buildMemorySystemPromptAddition } from "openclaw/plugin-sdk/core";

export default function register(api) {
  api.registerContextEngine("my-engine", (ctx) => ({
    info: {
      id: "my-engine",
      name: "My Context Engine",
      ownsCompaction: true,
    },

    async ingest({ sessionId, message, isHeartbeat }) {
      // メッセージをデータストアに保存
      return { ingested: true };
    },

    async assemble({
      sessionId,
      sessionKey,
      messages,
      tokenBudget,
      availableTools,
      citationsMode,
    }) {
      // 予算内に収まるメッセージを返す
      return {
        messages: buildContext(messages, tokenBudget),
        estimatedTokens: countTokens(messages),
        systemPromptAddition: buildMemorySystemPromptAddition({
          availableTools: availableTools ?? new Set(),
          citationsMode,
          agentSessionKey: sessionKey,
        }),
      };
    },

    async compact({ sessionId, force }) {
      // 古いコンテキストを要約
      return { ok: true, compacted: true };
    },
  }));
}
```

ファクトリー `ctx` には任意の `config`、`agentDir`、および `workspaceDir`
の値が含まれるため、Plugin は最初のライフサイクル呼び出し前に、エージェント単位またはワークスペース単位の状態を初期化できます。
非レガシーの `assemble()` 呼び出しの前に、ホストは登録済みの非同期メモリプロンプト準備を完了します。
同期の `buildMemorySystemPromptAddition(...)` ヘルパーは、その不変の実行スナップショットを読み取ります。
提供されたツール、引用、エージェント、およびセッションのコンテキストは変更せずに渡してください。

次に、設定で有効にします。

```json5
{
  plugins: {
    slots: {
      contextEngine: "my-engine",
    },
    entries: {
      "my-engine": {
        enabled: true,
      },
    },
  },
}
```

### ContextEngine インターフェース

必須メンバー：

| メンバー             | 種別     | 目的                                                  |
| ------------------ | -------- | -------------------------------------------------------- |
| `info`             | プロパティ | エンジンの ID、名前、バージョン、および Compaction を管理するかどうか |
| `ingest(params)`   | メソッド   | 単一のメッセージを保存                                   |
| `assemble(params)` | メソッド   | モデル実行用のコンテキストを構築（`AssembleResult` を返す） |
| `compact(params)`  | メソッド   | コンテキストを要約／縮小                                 |

`assemble` は、次の内容を持つ `AssembleResult` を返します。

<ParamField path="messages" type="Message[]" required>
  モデルに送信する順序付きメッセージ。
</ParamField>
<ParamField path="estimatedTokens" type="number" required>
  アセンブルされたコンテキスト内の総トークン数に関するエンジンの推定値。OpenClaw はこれを Compaction しきい値の判断と診断レポートに使用します。
</ParamField>
<ParamField path="systemPromptAddition" type="string">
  システムプロンプトの先頭に追加されます。
</ParamField>
<ParamField path="promptAuthority" type='"assembled" | "preassembly_may_overflow"'>
  ランナーが予防的なオーバーフロー事前チェックに使用するトークン推定値を制御します。デフォルトは `"assembled"` です。これは、Compaction を管理しないエンジンでは、アセンブルされたプロンプトの推定値のみがチェックされることを意味します。`ownsCompaction: true` を設定するエンジンは独自にプロンプトの受け入れを管理するため、OpenClaw はデフォルトで汎用のプロンプト前事前チェックをスキップします。アセンブルされたビューによって基礎となるトランスクリプトのオーバーフローリスクが隠れる可能性がある場合にのみ、`"preassembly_may_overflow"` を設定してください。この場合、ランナーは汎用の事前チェックを有効なままにし、予防的に Compact するかどうかを判断するときに、アセンブル済みの推定値とアセンブル前（ウィンドウ化されていない）のセッション履歴の推定値の最大値を使用します。どちらの場合でも、返したメッセージがモデルに表示される内容であることに変わりはありません。`promptAuthority` が影響するのは事前チェックのみです。
</ParamField>
<ParamField path="contextProjection" type="ContextEngineProjection">
  永続的なバックエンドスレッドを持つホスト（Codex app-server など）向けの任意の投影ライフサイクル。安定した `epoch` を持つ `mode: "thread_bootstrap"` は、アセンブルされたコンテキストをエポックごとに一度注入し、エポックが変わるまでバックエンドスレッドを再利用するようホストに要求します。これにより、ターンごとの再投影を避けます。通常のターンごとの投影では、このフィールドを省略してください。
</ParamField>

`compact` は `CompactResult` を返します。Compaction によってアクティブなセッション ID が変わる場合、`result.sessionTarget`（セッション ID とストアのスコープを保持する型付きの `ContextEngineSessionTarget`）は、次の再試行またはターンで使用する必要がある後継セッションを識別します。`result.sessionId` は後継 ID を反映します。

任意のメンバー：

| メンバー                         | 種別   | 目的                                                                                                                                      |
| ------------------------------ | ------ | -------------------------------------------------------------------------------------------------------------------------------------------- |
| `bootstrap(params)`            | メソッド | セッションのエンジン状態を初期化します。エンジンがセッションを初めて認識したときに一度だけ呼び出されます（履歴のインポートなど）。                              |
| `maintain(params)`             | メソッド | ブートストラップ後、ターンの正常完了後、または Compaction 後のトランスクリプト保守。安全な書き換えには `runtimeContext.rewriteTranscriptEntries()` を使用します。 |
| `ingestBatch(params)`          | メソッド | 完了したターンをバッチとして取り込みます。実行完了後、そのターンのすべてのメッセージをまとめて受け取って呼び出されます。                                  |
| `afterTurn(params)`            | メソッド | 実行後のライフサイクル処理（状態の永続化、バックグラウンドでの Compaction の開始）。                                                                      |
| `prepareSubagentSpawn(params)` | メソッド | 子セッションが開始する前に共有状態を設定します。                                                                                    |
| `onSubagentEnded(params)`      | メソッド | サブエージェントの終了後にクリーンアップします。                                                                                                              |
| `dispose()`                    | メソッド | リソースを解放します。Gateway のシャットダウン時または Plugin の再読み込み時に呼び出され、セッションごとには呼び出されません。                                                        |

### ランタイム設定

OpenClaw 内で実行されるライフサイクルフックは、任意の
`runtimeSettings` オブジェクトを受け取ります。これは、バージョン管理された読み取り専用の内部
プロデューサー／コンシューマー API サーフェスです。OpenClaw が選択されたコンテキスト
エンジン向けに生成し、コンテキストエンジンがライフサイクルフック内で使用します。これは
ユーザーに直接表示されず、専用のレポートサーフェスも作成しません。

- `schemaVersion`: 現在は `1`
- `runtime`: OpenClaw ホスト、ランタイムモード（`normal`、`fallback`、または
  `degraded`）、および任意のハーネス／ランタイム ID
- `contextEngineSelection`: 選択されたコンテキストエンジン ID と選択元
- `executionHost`: フックを呼び出すサーフェスのホスト ID とラベル
- `model`: 要求されたモデル、解決されたモデル、プロバイダー、および任意のモデルファミリー
- `limits`: 判明している場合は、プロンプトのトークン予算と最大出力トークン数
- `diagnostics`: 判明している場合は、フェイルクローズおよび機能低下の理由コード

不明になり得るフィールドは `null` として表されます。ランタイムモードや選択元などの
判別フィールドは null 非許容のままです。古いエンジンとの互換性も維持されます。
厳格なレガシーエンジンが `runtimeSettings` を未知のプロパティとして拒否した場合、
OpenClaw はエンジンを隔離する代わりに、それを除いてライフサイクル呼び出しを再試行します。

### ホスト要件

コンテキストエンジンは、`info.hostRequirements` でホストのケイパビリティ要件を宣言できます。
OpenClaw は操作を開始する前にこれらの要件を確認し、選択されたランタイムが要件を
満たせない場合は、説明的なエラーを返してフェイルクローズします。

エージェント実行では、エンジンが `assemble()` を介して実際のモデルプロンプトを
制御する必要がある場合、`assemble-before-prompt` を宣言します。

```ts
info: {
  id: "my-context-engine",
  name: "My Context Engine",
  hostRequirements: {
    "agent-run": {
      requiredCapabilities: ["assemble-before-prompt"],
      unsupportedMessage:
        "ネイティブ Codex または OpenClaw 組み込みランタイムを使用するか、レガシーコンテキストエンジンを選択してください。",
    },
  },
}
```

ネイティブ Codex および OpenClaw 組み込みのエージェント実行は `assemble-before-prompt` を満たします。
汎用 CLI バックエンドは満たさないため、これを必要とするエンジンは CLI プロセスの
開始前に拒否されます。

### 障害の分離

OpenClaw は、選択された Plugin エンジンをコアの応答パスから分離します。
非レガシーエンジンが見つからない、コントラクト検証に失敗する、ファクトリー作成中に
例外をスローする、またはライフサイクルメソッドから例外をスローする場合、OpenClaw は
現在の Gateway プロセスでそのエンジンを隔離し、コンテキストエンジンの処理を組み込みの
`legacy` エンジンへダウングレードします。エラーは失敗した操作とともにログに記録されるため、
オペレーターはエージェントを応答不能にすることなく、Plugin を修復、更新、または無効化できます。

ホスト要件の失敗はこれとは異なります。エンジンがランタイムに必要なケイパビリティが
ないと宣言した場合、OpenClaw は実行を開始する前にフェイルクローズします。これにより、
サポートされていないホストで実行すると状態を破損する可能性があるエンジンを保護します。

### ownsCompaction

`ownsCompaction` は、OpenClaw ランタイムに組み込まれた試行中の自動 Compaction を、その実行で有効なままにするかどうかを制御します。

<AccordionGroup>
  <Accordion title="ownsCompaction: true">
    エンジンが Compaction の動作を所有します。OpenClaw はその実行について、OpenClaw ランタイムに組み込まれた自動 Compaction と汎用的なプロンプト前オーバーフロー事前チェックを無効にします。また、エンジンの `compact()` 実装が、`/compact`、プロバイダーのオーバーフロー回復 Compaction、および `afterTurn()` で実行する任意の予防的 Compaction を担います。エンジンが `assemble()` から `promptAuthority: "preassembly_may_overflow"` を返した場合、OpenClaw は引き続きプロンプト前オーバーフロー保護処理を実行します。
  </Accordion>
  <Accordion title="ownsCompaction: false または未設定">
    OpenClaw ランタイムに組み込まれた自動 Compaction はプロンプト実行中にも動作する可能性がありますが、`/compact` およびオーバーフロー回復のために、アクティブなエンジンの `compact()` メソッドも引き続き呼び出されます。
  </Accordion>
</AccordionGroup>

<Warning>
`ownsCompaction: false` は、OpenClaw がレガシーエンジンの Compaction パスへ自動的にフォールバックすることを意味しません。
</Warning>

したがって、有効な Plugin パターンは 2 つあります。

<Tabs>
  <Tab title="所有モード">
    独自の Compaction アルゴリズムを実装し、`ownsCompaction: true` を設定します。
  </Tab>
  <Tab title="委譲モード">
    `ownsCompaction: false` を設定し、`compact()` から `delegateCompactionToRuntime(...)` を `openclaw/plugin-sdk/core` で呼び出して、OpenClaw の組み込み Compaction 動作を使用します。
  </Tab>
</Tabs>

何もしない `compact()` は、アクティブな非所有エンジンでは安全ではありません。そのエンジンスロットの通常の `/compact` およびオーバーフロー回復 Compaction パスが無効になるためです。

## 設定リファレンス

```json5
{
  plugins: {
    slots: {
      // アクティブなコンテキストエンジンを選択します。デフォルト: "legacy"。
      // Plugin エンジンを使用するには、Plugin ID を設定します。
      contextEngine: "legacy",
    },
  },
}
```

<Note>
このスロットは実行時に排他的です。特定の実行または Compaction 操作に対して解決される登録済みコンテキストエンジンは 1 つだけです。有効な他の `kind: "context-engine"` Plugin も読み込まれ、登録コードを実行できます。`plugins.slots.contextEngine` は、OpenClaw がコンテキストエンジンを必要とするときに解決する登録済みエンジン ID を選択するだけです。
</Note>

<Note>
**Plugin のアンインストール:** `plugins.slots.contextEngine` として現在選択されている Plugin をアンインストールすると、OpenClaw はスロットをデフォルト（`legacy`）に戻します。同じリセット動作が `plugins.slots.memory` にも適用されます。設定を手動で編集する必要はありません。
</Note>

## Compaction およびメモリとの関係

<AccordionGroup>
  <Accordion title="Compaction">
    Compaction はコンテキストエンジンの責務の 1 つです。レガシーエンジンは OpenClaw の組み込み要約処理に委譲します。Plugin エンジンは任意の Compaction 戦略（DAG 要約、ベクトル検索など）を実装できます。
  </Accordion>
  <Accordion title="メモリ Plugin">
    メモリ Plugin（`plugins.slots.memory`）はコンテキストエンジンとは別のものです。メモリ Plugin は検索／取得を提供し、コンテキストエンジンはモデルに何を見せるかを制御します。両者は連携できます。たとえば、コンテキストエンジンは組み立て中にメモリ Plugin のデータを使用できます。アクティブなメモリプロンプトパスを使用する Plugin エンジンは、`openclaw/plugin-sdk/core` の `buildMemorySystemPromptAddition(...)` を使用する必要があります。これは、ホストが準備したメモリプロンプトのセクションを、メモリ Plugin のレイアウトを公開せずに、先頭へ追加できる `systemPromptAddition` に変換します。
  </Accordion>
  <Accordion title="セッションのプルーニング">
    メモリ内の古いツール結果のトリミングは、アクティブなコンテキストエンジンに関係なく引き続き実行されます。
  </Accordion>
</AccordionGroup>

## ヒント

- エンジンが正しく読み込まれていることを確認するには、`openclaw doctor` を使用します。
- エンジンを切り替えても、既存のセッションは現在の履歴で継続します。新しいエンジンは以後の実行を引き継ぎます。
- エンジンのエラーはログに記録され、選択された Plugin エンジンは現在の Gateway プロセスで隔離されます。応答を継続できるよう、OpenClaw はユーザーターンで `legacy` にフォールバックしますが、壊れた Plugin は修復、更新、無効化、またはアンインストールする必要があります。
- 開発時には、ローカルの Plugin ディレクトリをコピーせずにリンクするために `openclaw plugins install -l ./my-engine` を使用します。

## 関連項目

- [Compaction](/ja-JP/concepts/compaction) - 長い会話の要約
- [コンテキスト](/ja-JP/concepts/context) - エージェントターンのコンテキストが構築される仕組み
- [Plugin アーキテクチャ](/ja-JP/plugins/architecture) - コンテキストエンジン Plugin の登録
- [Plugin マニフェスト](/ja-JP/plugins/manifest) - Plugin マニフェストのフィールド
- [Plugins](/ja-JP/tools/plugin) - Plugin の概要
