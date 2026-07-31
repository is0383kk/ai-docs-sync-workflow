---
read_when:
    - エージェント実行で OpenClaw コードモードを有効にする場合
    - Code Mode が Codex Code Mode と異なる理由を説明する必要があります
    - コンパクトなツールコントラクト、QuickJS-WASI サンドボックス、TypeScript 変換、または非表示のツールカタログブリッジをレビューしています
    - 内部のコードモード名前空間レジストリ統合を追加またはレビューしています
sidebarTitle: Code Mode
summary: OpenClaw Code Mode を使用して、大規模なツールカタログを検出、呼び出し、組み合わせ、コンパクトな JavaScript または TypeScript ワークフローを構築する
title: コードモード
x-i18n:
    generated_at: "2026-07-26T09:20:20Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: a21df3bcfb11668da6dde1f7c69adcc284a28dc491c95f95097ce7f41e5c45bf
    source_path: tools/code-mode.md
    workflow: 16
---

コードモードは、実験的でオプトインの OpenClaw エージェントランタイム機能です。有効にすると、
モデルには有効なすべてのツールスキーマが表示されなくなり、代わりに
`exec`、`wait`、および構造化された結果を JSON 専用のゲストブリッジ経由で渡せない
直接実行専用ツールが表示されます。モデルは、非表示のツールカタログを検索、記述、呼び出す
小さな JavaScript または TypeScript プログラムを記述します。

このページでは、Codex Code Mode ではなく OpenClaw コードモードについて説明します。2 つの機能は
名前と同じ制御ツール名（`exec`、`wait`）を共有していますが、
実装は別です。

- Codex Code Mode は Codex コーディングハーネス内で動作します。その `exec` ツールは
  自由形式文法のツールです。モデルは生の JavaScript ソースを記述し（実行オプション用の
  `// @exec: {...}` プラグマ行を先頭に付けることも可能）、Codex のプロセス内 V8 Code Mode ランタイムで
  実行されます。
- OpenClaw コードモードは汎用 OpenClaw エージェントランタイムで動作し、
  `tools.codeMode.enabled: true` が設定されていない限り無効です。その `exec`
  ツールは JSON の `{ code, language }` ペイロードを受け取り、QuickJS-WASI
  ワーカーで実行します。

どちらも JavaScript の実行サーフェスであり、シェルコマンドの実行サーフェスではありません。
同じ名前の `exec`/`wait` ツールを公開しているだけの、
実装が異なる独立した機能として扱ってください。

## 機能

- モデルに表示されるツール一覧は、`exec`、`wait`、および
  `computer` や、画像結果をゲストブリッジ経由で維持できないネイティブビジョンの
  `image` ローダーなどの直接実行専用ツールになります。
- `exec` は、モデルが生成した JavaScript または TypeScript を、
  分離された QuickJS-WASI ワーカースレッドで評価します。
- カタログに登録可能な有効ツール（OpenClaw コア、Plugin、MCP、クライアント）はすべて、
  モデルの単独ツールとしては非表示になり、ゲストプログラム内で `ALL_TOOLS`
  および `tools` を通じて公開されます。
- `exec` の説明には、正確な OpenClaw/Plugin カタログ ID の件数制限付きクイックインデックス、
  簡潔な入力ヒント、および信頼されたツールが出力スキーマを提供する場合の
  簡潔な宣言済み出力ヒントが含まれます。説明、完全なスキーマ、
  MCP エントリ、上限を超えたエントリは省略されます。ゲスト側のカタログ検索は引き続きフォールバックとして機能します。
- ゲストコードは非表示のカタログを検索し、ツールのスキーマを記述し、
  通常のエージェントターンと同じ実行パスを通じてツールを呼び出します（ポリシー、
  承認、フック、テレメトリはすべて引き続き適用されます）。
- MCP ツールは `MCP` 名前空間にまとめられます。コードモードでは、
  これが MCP ツールを呼び出す唯一の対応方法です。
- `wait` は、ネストされたツール呼び出しがまだ保留中の場合に、
  中断されたコードモード実行を再開します。

コードモードが変更するのは、モデル向けのオーケストレーションサーフェスだけです。
ツール、Plugin ツール、MCP ツール、認証、承認ポリシー、チャネルの
動作、モデル選択を置き換えるものではありません。

## 使用する理由

- プロンプトサーフェスの縮小：プロバイダーには、数十または数百の完全なツールスキーマの代わりに、
  2 つの制御ツール、件数制限付きのネイティブツールインデックス、および必要最小限の直接ツールだけが
  提供されます。
- オーケストレーションの向上：モデルは 1 つのコードセル内で、ループ、結合、小規模な変換、
  条件ロジック、並列のネストされたツール呼び出しを使用できます。
- モデルの往復回数を削減：宣言済みの出力契約により、モデルは 1 回の `exec` で
  ツール結果を呼び出して変換できます。出力が不明な場合は、最初に未加工の結果が返されます。
- プロバイダー非依存：プロバイダー固有のコード実行に依存せず、
  OpenClaw、Plugin、MCP、クライアントツールで動作します。
- フェイルクローズ：コードモードが有効でも QuickJS-WASI ランタイムを
  使用できない場合、広範な直接ツール公開へ暗黙的にフォールバックせず、
  実行は失敗します。

有効なツールカタログが大規模なエージェントや、応答する前にモデルが
複数のツールを検索、組み合わせ、呼び出す必要があるワークフローで特に有用です。

カタログが小規模な場合や、短いプログラムを確実に記述できないモデルでは、
ツールを直接公開したままにしてください。コンパクトなカタログを使用しつつ、
QuickJS-WASI ゲストではなく構造化された検索・記述・呼び出し制御を使用する場合は、
[ツール検索](/ja-JP/tools/tool-search)を使用してください。

## クイックスタート

### コードモードを有効にする

```json5
{
  tools: {
    codeMode: {
      enabled: true,
    },
  },
}
```

短縮形：

```json5
{
  tools: {
    codeMode: true,
  },
}
```

`tools.codeMode` が省略されている場合、`false` の場合、または
`enabled: true` を含まないオブジェクトの場合、コードモードは無効のままです。

MCP サーバーが設定されたサンドボックス化エージェントを使用する場合は、
サンドボックスのツールポリシーで同梱の MCP Plugin も許可してください。例：
`tools.sandbox.tools.alsoAllow: ["bundle-mcp"]`。詳細は
[設定 - ツールとカスタムプロバイダー](/ja-JP/gateway/config-tools#mcp-and-plugin-tools-inside-sandbox-tool-policy)を参照してください。

境界をより厳しくするには、明示的な上限を設定します。

```json5
{
  tools: {
    codeMode: {
      enabled: true,
      timeoutMs: 10000,
      memoryLimitBytes: 67108864,
      maxOutputBytes: 65536,
      maxSnapshotBytes: 10485760,
      maxPendingToolCalls: 16,
      snapshotTtlSeconds: 900,
      searchDefaultLimit: 8,
      maxSearchLimit: 50,
    },
  },
}
```

### モデルの動作

`Array<{ id: string; paid: boolean; tons: number }>` のような出力が宣言されたツールでは、
1 つのゲストプログラムでツールを選択、呼び出し、変換できます。

```javascript
const [shipmentTool] = await tools.search("list shipments");
const shipments = await tools.callValue(shipmentTool.id, {});
return shipments.filter((shipment) => !shipment.paid && shipment.tons > 10);
```

クイックインデックス行が `-> ?` で終わる場合、出力形式は不明です。最初の
`exec` は `await tools.callValue(...)` を変更せずに返す必要があります。後続の `exec` では、
観測された値を変換できます。この処理にはモデルターンが 1 回余分に必要ですが、
モデルがフィールド名を推測することを防ぎます。

### 有効なサーフェスを確認する

デバッグ中にモデルペイロードの形式を確認するには、対象を絞ったログを有効にして
Gateway を実行します。

```bash
OPENCLAW_DEBUG_CODE_MODE=1 \
OPENCLAW_DEBUG_MODEL_TRANSPORT=1 \
OPENCLAW_DEBUG_MODEL_PAYLOAD=tools \
openclaw gateway
```

コードモードが有効な場合、ログに記録されるモデル向けツール名は `exec` と
`wait` になるはずです。編集済みの完全なプロバイダーペイロードを確認するには、
短時間のデバッグセッションで `OPENCLAW_DEBUG_MODEL_PAYLOAD=full-redacted` を追加します。

## エージェントのファンアウトに Swarm を使用する

[Swarm](/tools/swarm)は、Code Mode スクリプトから並行サブエージェントをオーケストレーションするための
`agents.run()`、`phase()`、`log()` ゲストグローバルを追加します。
`tools.codeMode` と `tools.swarm` の両方を有効にし、通常の JavaScript 制御フローを使用して
ファンアウト、判断ゲート、構造化された収集を行います。Swarm は独立したオプトインゲートです。
Code Mode を有効にするだけでは `agents.*` API は公開されません。

## 技術解説

このページの残りでは、メンテナー、ツール公開をデバッグする Plugin 作成者、
高リスクなデプロイを検証する運用担当者向けに、ランタイム契約と実装の詳細を説明します。

## ランタイムの状態

|                     |                                                                                             |
| ------------------- | ------------------------------------------------------------------------------------------- |
| ランタイム             | [`quickjs-wasi`](https://github.com/vercel-labs/quickjs-wasi)                               |
| デフォルト状態       | 無効                                                                                    |
| 安定性           | 実験的な OpenClaw サーフェス（Codex Code Mode は独立した安定版 Codex ハーネスサーフェス） |
| 対象サーフェス      | 汎用 OpenClaw エージェント実行                                                                 |
| セキュリティ方針    | モデルコードは敵対的なものとして扱う                                                                       |
| ユーザー向けの保証 | コードモードを有効にしても、広範な直接ツール公開へ暗黙的にフォールバックしない                  |

## スコープ

コードモードは、準備された実行におけるモデル向けオーケストレーション形式を担当します。
モデル選択、チャネルの動作、認証、ツールポリシー、ツール実装は
担当しません。

対象範囲：モデルに表示される制御ツールと直接ツールの定義、非表示ツールカタログの
構築、JavaScript/TypeScript ゲスト実行、QuickJS-WASI ワーカー
ランタイム、検索・記述・呼び出し用のホストコールバック、中断されたゲストプログラム用の
再開可能な状態、出力・タイムアウト・メモリ・保留中呼び出し・スナップショットの上限、
およびネストされたツール呼び出しのテレメトリ／軌跡への投影。

対象外：プロバイダー固有のリモートコード実行、シェル実行の
セマンティクス、既存のツール認可の変更、ユーザーが作成した永続スクリプト、
ゲストコードでのパッケージマネージャー・ファイル・ネットワーク・モジュールへのアクセス、
および Codex Code Mode 内部実装の直接再利用。

リモート Python サンドボックスなどのプロバイダー所有ツールは別のツールです。
[コード実行](/ja-JP/tools/code-execution)を参照してください。

## 用語

- **コードモード**：カタログ対応のモデルツールを非表示にし、
  `exec`、`wait`、および必要な直接実行専用ツールを公開する OpenClaw ランタイムモード。
- **ゲストランタイム**：モデルコードを評価する QuickJS-WASI JavaScript VM。
- **ホストブリッジ**：ゲストコードから OpenClaw に戻る、
  JSON 互換の限定的なコールバックサーフェス。
- **カタログ**：通常のツールポリシー、Plugin、MCP、
  クライアントツールの解決後に得られる、実行スコープの有効なツール一覧。
- **ネストされたツール呼び出し**：ゲストコードからホストブリッジを通じて行われる
  ツール呼び出し。
- **スナップショット**：`wait` が中断されたコードモード実行を継続できるように
  保存される、シリアライズ済み QuickJS-WASI VM 状態。

## 設定

`tools.codeMode.enabled` は有効化ゲートです。他のフィールドを設定しても、
それだけではこの機能は有効になりません。

| フィールド                 | デフォルト                        | 制限                                           |
| --------------------- | ------------------------------ | ----------------------------------------------- |
| `enabled`             | `false`                        | ブール値。`true` のみがコードモードを有効にする          |
| `runtime`             | `"quickjs-wasi"`               | 対応している唯一の値                            |
| `mode`                | `"only"`                       | 制御ツールと直接ツールを公開し、残りをカタログ化する |
| `languages`           | `["javascript", "typescript"]` | 2 つの任意のサブセット                           |
| `timeoutMs`           | `10000`                        | `100`-`60000`                                   |
| `memoryLimitBytes`    | `67108864`                     | `1048576`-`1073741824`                          |
| `maxOutputBytes`      | `65536`                        | `1024`-`10485760`                               |
| `maxSnapshotBytes`    | `10485760`                     | `1024`-`268435456`                              |
| `maxPendingToolCalls` | `16`                           | `1`-`128`                                       |
| `snapshotTtlSeconds`  | `900`                          | `1`-`86400`                                     |
| `searchDefaultLimit`  | `8`                            | `maxSearchLimit` に制限                     |
| `maxSearchLimit`      | `50`                           | `1`-`50`                                        |

コードモードが有効でも QuickJS-WASI を読み込めない場合、OpenClaw はその実行を
フェイルクローズします。フォールバックとして通常のツールを暗黙的に公開することはありません。

## 有効化

コードモードは、有効なツールポリシーが確定した後、最終的なモデルリクエストが
組み立てられる前に評価されます。

1. エージェント、モデル、プロバイダー、サンドボックス、チャネル、送信者、実行
   ポリシーを解決します。
2. 有効な OpenClaw ツールリストを構築し、対象となる plugin、MCP、および
   クライアントツールを追加します。
3. 許可/拒否ポリシーを適用します。
4. `tools.codeMode.enabled` が false の場合、通常のツール公開を続行します。
5. 有効で、実行時にツールがアクティブな場合、必要な直接専用
   ツールを保持し、カタログ対象となるすべての有効なツールをコードモード
   カタログに登録します。
6. カタログ登録されたツールをモデルに表示されるリストから削除し、保持された直接専用ツールとともに `exec` および
   `wait` を追加します。

意図的にツールを使用しない実行（モデルの生呼び出し、`disableTools: true`、
または空の `tools.allow` リスト）では、`tools.codeMode.enabled: true` が設定されていても
コードモードサーフェスは有効になりません。コードモードと OpenClaw Tool
Search は実行単位で相互排他的です。コードモードが有効になると、Tool Search の
compaction は行われません。

コードモードカタログは実行スコープであり、別の
エージェント、セッション、送信者、または実行からツールが漏洩してはなりません。

## モデルに表示されるツール

コードモードがアクティブな場合、モデルには `exec`、`wait`、および必要な
直接専用ツールが表示されます。その他の有効なツールはすべてモデル向け
ツールリストから非表示になり、コードモードカタログに登録されます。

ツールのオーケストレーション、データ結合、ループ、並列のネストされた呼び出し、
および構造化変換には `exec` を使用します。`exec` が再開可能な
`waiting` 結果を返した場合にのみ `wait` を使用します。

## `exec`

`exec` はコードモードセルを開始し、1 つの結果を返します。入力コードはモデルによって
生成されるため、敵対的なものとして扱う必要があります。

入力:

```typescript
type CodeModeExecInput = {
  code?: string;
  command?: string;
  language?: "javascript" | "typescript";
};
```

ルール:

- `code` または `command` のいずれかが空でない必要があります。
- `code` は、文書化されているモデル向けフィールドです。
- `command` は、フックポリシーおよび信頼済みの書き換え用に exec 互換のエイリアスとして受け付けられます
  （通常の OpenClaw シェル exec ツールも `command`
  フィールドを使用します）。両方が指定された場合、値は一致する必要があります。
- `language` のデフォルトは `"javascript"` です。一部のプロバイダーはこれらの形式を拒否するため、
  スキーマでは `oneOf`/`anyOf` ユニオンではなく、フラットな
  文字列列挙型（`"javascript" | "typescript"`）として公開されます。
- `language` が `"typescript"` の場合、OpenClaw は評価前にトランスパイルします。
- `exec` は `import`、`require`、動的インポート、およびモジュールローダー
  パターンを拒否します。
- `exec` は通常のシェル `exec` 実装を再帰的に公開することはありません。
- 外側のコードモード `exec` フックイベントには `toolKind: "code_mode_exec"` と
  `toolInputKind: "javascript" | "typescript"`（既知の場合）が含まれるため、ポリシーは
  同じツール名を共有するコードモードセルとシェル形式の `exec` 呼び出しを
  区別できます。

結果:

```typescript
type CodeModeResult = CodeModeCompletedResult | CodeModeWaitingResult | CodeModeFailedResult;

type CodeModeCompletedResult = {
  status: "completed";
  value: unknown;
  output?: CodeModeOutput[];
  telemetry: CodeModeTelemetry;
};

type CodeModeWaitingResult = {
  status: "waiting";
  runId: string;
  reason: "pending_tools" | "yield";
  pendingToolCalls?: CodeModePendingToolCall[];
  output?: CodeModeOutput[];
  telemetry: CodeModeTelemetry;
};

type CodeModeFailedResult = {
  status: "failed";
  error: string;
  code?: CodeModeErrorCode;
  output?: CodeModeOutput[];
  telemetry: CodeModeTelemetry;
};
```

ゲストが、モデルから見える継続処理をまだ必要とする再開可能な状態（明示的な `yield_control(...)`、または
exec の期限内に解決しなかったブリッジツール呼び出し）で中断すると、
`exec` は `waiting` を返します。結果には `wait` 用の
`runId` が含まれます。ブリッジツール呼び出し（`tools.search`/`describe`/
`call` および MCP 名前空間呼び出しを含む名前空間呼び出し）は、期限内に解決される間は
同じ `exec`/`wait` 呼び出し内で自動的にドレインされるため、
複数のツールを await するコンパクトなコードブロックは、await ごとにモデルツール呼び出しを
強制することなく、モデルの 1 ターンで完了します。再起動セーフな実行では
自動ドレインは行われません。保留中の処理には引き続きリプレイセーフなチェックが適用されます。

`exec` が `completed` を返すのは、ゲスト VM に保留中の処理がなく、
OpenClaw の出力アダプター実行後の最終値が JSON 互換である場合のみです。

## `wait`

`wait` は中断されたコードモード VM を継続します。

入力:

```typescript
type CodeModeWaitInput = {
  runId: string;
};
```

出力は、`exec` が返すものと同じ `CodeModeResult` ユニオンです。

`wait` が存在するのは、ネストされた OpenClaw ツールが低速、対話的、承認ゲート付き、
または部分的な更新をストリーミングする可能性があり、ホストが外部処理を待つ間、
モデルが 1 つの長時間の `exec` 呼び出しを開いたままにする必要がないようにするためです。

QuickJS-WASI のスナップショット/復元が再開メカニズムです:

1. `exec` は、完了、失敗、または中断までコードを評価します。
2. 中断時に、OpenClaw は QuickJS VM のスナップショットを作成し、保留中のホスト
   処理を記録します。
3. 保留中の処理が完了すると、`wait` は VM スナップショットを復元し、
   安定した名前でホストコールバックを再登録します。
4. OpenClaw はネストされたツールの結果を復元済み VM に渡し、
   QuickJS の保留中ジョブをドレインします。
5. `wait` は `completed`、`failed`、または別の `waiting` 結果を返します。

スナップショットはユーザー成果物ではなくランタイム状態です。スナップショットは
プロセス内マップにのみ存在し（データベースやディスクへの書き込みはありません）、サイズ制限と有効期限があり、
作成元の実行およびセッションにスコープされます。

次の場合、`wait` は（`failed` 結果として）失敗します:

- `runId` が不明であるか、そのスナップショットがすでに期限切れです。
- 呼び出し元が、中断された実行と同じ実行/セッションスコープ内にありません。
- その `runId` に対する `wait` がすでに実行中です。
- QuickJS-WASI の復元に失敗します。
- 再開すると `maxOutputBytes` または `maxSnapshotBytes` を超過します。

## ゲストランタイム API

```typescript
declare const ALL_TOOLS: ToolCatalogEntry[];
declare const tools: ToolCatalog;
declare const MCP: Record<string, unknown>;
declare const namespaces: Record<string, unknown>;

declare function text(value: unknown): void;
declare function json(value: unknown): void;
declare function yield_control(reason?: string): Promise<void>;
```

`ALL_TOOLS` は実行スコープのカタログ用のコンパクトなメタデータであり、デフォルトでは
完全なスキーマを含みません。モデルに表示される `exec` の説明には、
正確な OpenClaw/plugin ID の制限付きかつ決定論的なサブセット、コンパクトな入力
ヒント、および信頼済みの宣言済み出力ヒントも含まれます。敵対的なカタログの文言が
モデルを誘導できないように、説明の読み込みは遅延されたままです。そのインデックスにツールが含まれていない場合は、
`ALL_TOOLS` を読み取るか、ゲストプログラム内で `tools.search(...)` を呼び出します。

各クイックインデックス行の矢印は `tools.callValue(...)` 値を表します。
`-> Array<{ id: string }>` は宣言済みの出力ヒントであり、`-> ?` は出力不明です。
不明な出力では生の値を優先します。フィールド名を推測せず、値を変更せずに返して確認し、
後続の `exec` でフィルタリングまたはマッピングします。これは、
宣言済み出力の読み取り結果を最終的な `-> ?` 呼び出しに渡す場合にも適用されます。
その呼び出しの生の値を、要求された回答形式でラップせずに返します。

```typescript
type ToolCatalogEntry = {
  id: string;
  name: string;
  label?: string;
  description: string;
  source: "openclaw" | "mcp" | "client";
  sourceName?: string;
  input: string;
  output?: string;
};
```

`input` は一般的なケース向けの制限付き TypeScript 形式シグネチャです。正確で完全な
スキーマが引き続き必要な場合は `tools.describe(...)` を使用します。リモート MCP
およびクライアントのエントリは `input: "unknown"` を使用するため、信頼されていないスキーマは
`describe` まで遅延されたままです。`output` が
存在するのは、信頼済みの OpenClaw コアまたは plugin `outputSchema` から導出された
完全なコンパクトヒントの場合のみです。MCP およびクライアントの出力スキーマに関する宣言は、
この信頼済みカタログヒントには昇格されません。

Plugin ツールは、`sourceName` に所有元の
plugin ID を設定した `source: "openclaw"` を使用します。個別の `"plugin"` ソース値はありません。`source: "mcp"` は
`sourceName`/`mcp` メタデータ内の MCP エントリにのみ使用されます
（`ALL_TOOLS`/`tools.*` からは除外されます。以下を参照してください）。

完全なスキーマは必要な場合にのみ読み込まれます:

```typescript
type ToolCatalogEntryWithSchema = ToolCatalogEntry & {
  parameters: unknown;
  outputSchema?: unknown;
};
```

カタログヘルパー:

```typescript
type ToolCatalog = {
  search(query: string, options?: { limit?: number }): Promise<ToolCatalogEntry[]>;
  describe(id: string): Promise<ToolCatalogEntryWithSchema>;
  callValue(id: string, input?: unknown): Promise<unknown>;
  call(id: string, input?: unknown): Promise<unknown>;
  [safeToolName: string]: unknown;
};
```

便利なツール関数は、曖昧さのない安全な名前に対してのみインストールされます:

```typescript
const files = await tools.search("read local file");
const fileRead = await tools.describe(files[0].id);
const content = await tools.callValue(fileRead.id, { path: "README.md" });

// If the hidden catalog has an unambiguous `web_search` entry:
const hits = await tools.web_search({ query: "OpenClaw code mode" });
```

`tools.callValue(...)` は通常のツールの JSON `details` 値を直接返します。
`tools.call(...)` は、コンテンツブロックまたはその他の結果メタデータを必要とする呼び出し元のために、
生の `{ tool, result }` エンベロープを保持します。

## 宣言済み出力コントラクト

OpenClaw ツールは、`AgentToolResult.details` に配置される構造化された値について
`outputSchema` を宣言できます。これは Code Mode および Tool Search に役立ちますが、
プロバイダー固有のツール応答スキーマではなく、直接的なツール公開を変更しません。

`defineToolPlugin` で作成されたツールでは、`parameters` の横に
スキーマを宣言します:

```typescript
import { Type } from "typebox";
import { defineToolPlugin } from "openclaw/plugin-sdk/tool-plugin";

const Shipment = Type.Object(
  {
    id: Type.String(),
    paid: Type.Boolean(),
    tons: Type.Number(),
  },
  { additionalProperties: false },
);

export default defineToolPlugin({
  id: "shipping",
  name: "Shipping",
  description: "Shipment tools.",
  tools: (tool) => [
    tool({
      name: "shipping_list",
      description: "List shipments.",
      parameters: Type.Object({}),
      outputSchema: Type.Array(Shipment),
      execute: async () => loadShipments(),
    }),
  ],
});
```

`api.registerTool(...)` またはファクトリーツールの場合は、返される `AnyAgentTool` オブジェクトに
同じ `outputSchema` プロパティを設定します。

現在の組み込みコントラクトには、`agents_list`、`apply_patch`、
`conversations_list`、`conversations_send`、`conversations_turn`、`edit`、
`openclaw`、`read`、`screen`、
`sessions_history`、`sessions_list`、`sessions_search`、`sessions_send`、
`session_status`、`spawn_task`、`terminal`、`web_fetch`、および `web_search` が含まれます。
完全なパススルーでは、モデル専用コントラクトを重複して定義する代わりに、
その所有元のプロトコルスキーマを再利用できます。たとえば、会話ツールは
`conversations.list`、`conversations.send`、および `conversations.turn` で使用されるものと
同じ Gateway 結果スキーマを公開します。`web_fetch` は、安定したメタデータ、
テキスト、キャッシュ状態、およびネストされたスピルメタデータをヒントとして公開する
ツールローカルスキーマを所有します。`web_search` は、正規化された結果／回答／エラー／raw の
完全なユニオンを、完全なクイックインデックスヒントとして宣言します。ファイルシステムコントラクトは、
構造化された読み取りテキスト、画像、切り捨て、およびオプションの未検出結果、明示的な編集の
変更状態と diff／パッチデータ、ならびに apply-patch のパス概要を返します。
クイックインデックスでフィールドが宣言されている場合、1 つのセルで検出と送信を構成でき、
別途確認ターンを設ける必要はありません。

```javascript
const listed = await tools.conversations_list({ query: "build bot" });
const target = listed.conversations.find((item) => item.label === "Build bot");
if (!target) throw new Error("conversation not found");
return await tools.conversations_send({
  conversationRef: target.conversationRef,
  message: "Build finished.",
});
```

ネストされた呼び出しでも、通常のツールポリシー、フック、および承認が使用されます。完全な
コントラクトが厳密であっても、境界付きクイックインデックスには大きすぎる場合は、
`tools.describe(...)` を通じて引き続き利用でき、矢印は `-> ?` のままです。

コントラクトの規則は厳格です。

- レンダリングされた `content`
  ブロックやプロバイダーエンベロープではなく、JSON 互換の正確な `details` 値を記述します。
- 例外をスローしない成功またはエラーのすべてのバリアントを含めます。ツールに
  安定した構造化結果がない場合は、`outputSchema` を省略します。
- 完全なクイックインデックスヒントにするには、オブジェクト層を
  `{ additionalProperties: false }` で閉じます。オープンなスキーマ、サイズ超過のスキーマ、またはその他の
  部分的なスキーマは `tools.describe(...)` を通じて引き続き利用できますが、
  1 ターンでのフィールド利用は有効になりません。
- OpenClaw はツールを実行する前にスキーマをコンパイルし、その後、通常のツールフックの
  完了後かつカタログ呼び出しが返る前に、最終的な `details` を検証します。
  無効なスキーマではツールを実行できません。不一致がある場合は、値を出力せずに失敗します。
- コンパクトなヒントは決定論的で、サイズが制限されています。コンパクトなヒントでは
  不十分な場合、`tools.describe(...)` で信頼済みの完全なスキーマを公開します。
- インストール済みの Plugin コードは、すでに信頼済みのローカルコードです。リモート MCP と
  クライアントのメタデータは引き続き信頼されず、これらのクイックインデックスヒントを有効にできません。

Plugin の作成方法の詳細については、[ツール Plugin](/ja-JP/plugins/tool-plugins#output-contracts)を
参照してください。

MCP カタログエントリは、コードモードで `tools.callValue(...)`、
`tools.call(...)`、または便利関数を通じて呼び出すことはできません。生成された
`MCP` 名前空間を通じてのみ公開されます。TypeScript 形式の宣言ファイルは、
読み取り専用の `API` 仮想ファイルサーフェスを通じて利用できるため、エージェントは
MCP スキーマをプロンプトに追加せずに MCP シグネチャを確認できます。

```typescript
const files = await API.list("mcp");
const githubApi = await API.read("mcp/github.d.ts");

const issue = await MCP.github.createIssue({
  owner: "openclaw",
  repo: "openclaw",
  title: "Investigate gateway logs",
});

const snapshot = await MCP.chromeDevtools.takeSnapshot({ output: "markdown" });
const resource = await MCP.docs.resources.read({ uri: "memo://one" });
const prompt = await MCP.docs.prompts.get({
  name: "brief",
  arguments: { topic: "release" },
});
```

`API.read("mcp/<server>.d.ts")` は、MCP ツールメタデータから推論されたコンパクトな宣言を返します。

```typescript
type McpToolResult = {
  content?: unknown[];
  structuredContent?: unknown;
  isError?: boolean;
  [key: string]: unknown;
};

declare namespace MCP.github {
  /** この TypeScript 形式の API ヘッダーを返します。 */
  function $api(toolName?: string, options?: { schema?: boolean }): Promise<McpApiHeader>;

  /**
   * GitHub issue を作成します。
   * @param owner リポジトリの所有者
   * @param repo リポジトリ名
   * @param title issue のタイトル
   */
  function createIssue(input: {
    owner: string;
    repo: string;
    title: string;
    body?: string;
  }): Promise<McpToolResult>;
}
```

宣言ファイルは仮想であり、ワークスペースや状態ディレクトリ配下には
書き込まれません。コードモードの `exec` 呼び出しごとに、OpenClaw は実行スコープのツール
カタログを構築し、表示可能な MCP エントリを保持して、`mcp/index.d.ts` と、表示可能なサーバーごとに 1 つの
`mcp/<server>.d.ts` をレンダリングし、その小さな読み取り専用テーブルを
QuickJS ワーカーに注入します。ゲストコードから見えるのは `API` オブジェクトのみです。
`API.list(prefix?)` はファイルメタデータを返し、`API.read(path)` は
選択された宣言の内容を返します。不明なパス、および `.`／`..` セグメントは
拒否されます。

これにより、大きな MCP スキーマがモデルプロンプトに含まれなくなります。エージェントは
`exec` ツールの説明から仮想 API の存在を把握し、必要な
宣言ファイルだけを読み取ってから、1 つのオブジェクト引数で `MCP.<server>.<tool>()` を呼び出します。
プログラム内で単一ツールのスキーマ応答を得るためのインラインフォールバックとして、
`MCP.<server>.$api()` も引き続き利用できます。

ゲストランタイムがホストオブジェクトを直接参照することはありません。入力と出力は、
明示的なサイズ上限を持つ JSON 互換値としてブリッジを通過します。

## 内部名前空間

内部名前空間により、モデルに表示されるツールを増やすことなく、コードモードに
簡潔なドメイン API を提供できます。ローダーが所有する統合は、`Issues` や
`Calendar` などの名前空間を登録します。その後、ゲストコードは QuickJS プログラム内でその名前空間を
呼び出しますが、モデルからは引き続きコンパクトな制御／直接サーフェスだけが見えます。

現時点では、名前空間は内部用です。公開 Plugin SDK の名前空間 API はありません。
外部 Plugin の名前空間にはローダー所有のコントラクトが必要です。これにより、Plugin の ID、
インストール済みマニフェスト、認証状態、およびキャッシュされたカタログ記述子が、その名前空間を支える
Plugin ツールとずれることを防ぎます。コアのコードモードが所有するのは、サンドボックス、
シリアライズ、カタログゲーティング、およびブリッジディスパッチのみです。

ゲストコードでは、直接グローバルまたは `namespaces` マップのいずれかを使用できます。

```javascript
const open = await Issues.list({ state: "open" });
const alsoOpen = await namespaces.Issues.list({ state: "open" });
return { count: open.length, alsoCount: alsoOpen.length };
```

### レジストリのライフサイクル

名前空間レジストリはプロセスローカルであり、名前空間 ID をキーとします。

1. 信頼済みローダーが `registerCodeModeNamespaceForPlugin(pluginId, registration)` を呼び出します。
2. コードモードは実行用の非表示の `ToolSearchRuntime` を作成し、その
   実行スコープのカタログを読み取ります。
3. `createCodeModeNamespaceRuntime(ctx, catalog)` は、すべての `requiredToolNames` が表示可能で、同じ
   `pluginId` によって所有されている登録のみを保持します。
4. 表示可能な各名前空間は、現在の実行について `createScope(ctx)` を呼び出し、
   `agentId`、`sessionKey`、`sessionId`、
   `runId`、設定、ならびに中止状態などの実行コンテキストを受け取ります。
5. スコープデータはプレーンな記述子にシリアライズされ、直接グローバルおよび
   `namespaces.<globalName>` として QuickJS に注入されます。
6. ゲスト呼び出しはワーカーブリッジを通じて一時停止し、ホスト上で名前空間パスを
   解決し、その呼び出しを宣言済みの Plugin 所有カタログツールにマッピングして、
   `ToolSearchRuntime.callExactId` を通じてそのツールを実行します。
7. 準備完了の名前空間ブリッジ呼び出しは、アクティブな
   `exec`／`wait` 呼び出し内で自動的に排出されます。タイムアウト時に
   名前空間の処理がまだ保留中である場合、またはゲストが明示的に処理を譲る場合は、
   `wait` が後で同じ名前空間ランタイムを再開します。
8. Plugin のロールバックまたはアンインストールでは、
   `clearCodeModeNamespacesForPlugin(pluginId)` を呼び出し、失敗した Plugin の読み込み後も古いグローバルが
   残らないようにします。

名前空間呼び出しはカタログツール呼び出しです。`tools.call(...)` と同じポリシーフック、
承認、中止処理、テレメトリ、トランスクリプト投影、および一時停止／再開動作を使用します。

### 登録形式

バックエンドツールを所有する統合から名前空間を登録します。スコープを小さく保ち、
宣言済みのカタログツールにマッピングされるドメイン動詞のみを公開します。

```typescript
import {
  createCodeModeNamespaceTool,
  registerCodeModeNamespaceForPlugin,
} from "../agents/code-mode-namespaces.js";

const pluginId = "github";

registerCodeModeNamespaceForPlugin(pluginId, {
  id: "github-issues",
  globalName: "Issues",
  description: "GitHub issue helpers for the current repository.",
  requiredToolNames: ["github_list_issues", "github_update_issue"],
  prompt: "Use Issues.list(params) and Issues.update(number, patch).",
  createScope: (ctx) => ({
    repository: ctx.config,
    list: createCodeModeNamespaceTool("github_list_issues", ([params]) => params ?? {}),
    update: createCodeModeNamespaceTool("github_update_issue", ([number, patch]) => ({
      number,
      patch,
    })),
  }),
});
```

`createCodeModeNamespaceTool(toolName, inputMapper)` は、スコープメンバーを呼び出し可能な
名前空間関数としてマークします。オプションの `inputMapper` はゲスト引数を受け取り、
バックエンドのカタログツールに渡す入力オブジェクトを返します。指定しない場合は、
最初のゲスト引数が使用され、省略時は `{}` が使用されます。

raw ホスト関数は、ゲストコードの実行前に拒否されます。

```typescript
createScope: () => ({
  // 誤り：これはカタログツールのライフサイクルを迂回するため、拒否されます。
  list: async () => githubClient.listIssues(),
});
```

### 所有権と可視性

名前空間の所有権は、登録呼び出し元の `pluginId` に紐付けられます。
`requiredToolNames` は、可視性ゲートであると同時に所有権チェックでもあります。

- 必要なすべてのツールが実行カタログに存在する必要があります
- 必要なすべてのツールに `sourceName === pluginId` が必要です
- 必要なツールのいずれかが存在しないか、別の Plugin によって所有されている場合、
  名前空間は非表示になります
- 呼び出し可能な各パスが対象にできるのは、`requiredToolNames` に指定された
  ツールのみです

これにより、別の Plugin が同名のツールを登録して名前空間を公開することを防ぎ、
名前空間を通常のエージェントポリシーと整合させます。実行からバックエンドツールが
見えない場合、名前空間も見えません。

たとえば、GitHub 名前空間は、GitHub の認証、REST／GraphQL クライアント、
レート制限、書き込み承認、およびテストを所有する、GitHub 所有の Plugin の背後に置く必要があります。
コアのコードモードに GitHub 固有の API、トークン処理、またはプロバイダーポリシーを
埋め込むべきではありません。

### スコープのシリアライズ規則

`createScope(ctx)` は、JSON 互換値、配列、ネストされたオブジェクト、および
`createCodeModeNamespaceTool(...)` 呼び出しマーカーを含むプレーンオブジェクトを返せます。
ホストオブジェクトが QuickJS に直接入ることはありません。

シリアライザーは次を拒否します。

- raw 関数
- 循環するオブジェクトグラフ
- 安全でないパスセグメント：`__proto__`、`constructor`、`prototype`、空のキー、
  または内部パス区切り文字を含むキー
- JavaScript 識別子ではない `globalName` 値
- `globalName` と、`tools`、
  `namespaces`、`text`、`json`、`yield_control`、`MCP`、`API`、`ALL_TOOLS`、または
  `__openclaw*` などの組み込みコードモードグローバルとの衝突

JSON にシリアライズできない値は、ブリッジを通過する前に JSON セーフな
フォールバック値へ変換されます。バイナリデータ、ハンドル、ソケット、クライアント、および
クラスインスタンスは、通常のカタログツールの背後に留める必要があります。

### プロンプト

名前空間の `description` とオプションの `prompt` は、その実行で名前空間が
表示可能な場合にのみ、モデルに表示される `exec` スキーマへ追加されます。
最小限の有用なサーフェスを教えるために使用します。

```typescript
{
  description: "フィクション制作サービスのヘルパー。",
  prompt:
    "Fictions.riskAudit()、Fictions.promoteIfReady(id, status)、Fictions.unpaidOver(amount) を使用します。",
}
```

プロンプトには、認証設定、実装履歴、または無関係な Plugin の動作ではなく、名前空間の契約について記述してください。

### クリーンアップ

名前空間はプロセスローカルな登録です。所有する Plugin が無効化、アンインストール、またはロールバックされた場合は削除します。

```typescript
clearCodeModeNamespacesForPlugin(pluginId);
```

コードモードのクリーンアップは Plugin が所有します。名前空間ごとの破棄ハンドルを保持するのではなく、Plugin のライフサイクルが終了した時点で、その Plugin の名前空間登録をクリアしてください。
テストでは、ケース間で登録が漏れないように `clearCodeModeNamespacesForTest()` を呼び出せます。

### テストチェックリスト

名前空間の変更では、セキュリティ境界とゲストの動作を網羅する必要があります。

- 名前空間のプロンプトテキストは、基盤となるツールが表示されている場合にのみ表示される
- 別の `sourceName` にある同名のツールは、名前空間を公開しない
- 生のスコープ関数は拒否される
- 偽造された名前空間 ID と偽造されたパスは拒否される
- 呼び出し可能なパスは、宣言されていないツールを対象にできない
- ネストされたオブジェクトと共有参照が正しくシリアライズされる
- 名前空間の呼び出しはカタログツールを介して実行され、JSON セーフな詳細を返す
- 失敗をゲストコードで捕捉できる
- 一時停止された名前空間の呼び出しは、`wait` を介して再開される
- Plugin のロールバックにより、所有する名前空間登録がクリアされる

名前空間は汎用の `tools.search`/`tools.call` カタログを補完します。任意の有効な OpenClaw、Plugin、およびクライアントツールにはカタログを使用し、MCP ツールには `MCP` を使用します。その他の名前空間は、スキーマ検索を繰り返すより簡潔なコードの方が信頼性の高い、Plugin 所有の文書化されたドメイン API に使用します。

## 出力 API

- `text(value)` は、人間が読める出力を `output` 配列に追加します。
- `json(value)` は、JSON 互換のシリアライズ後に構造化された出力項目を追加します。
- ゲストコードが最後に返した値は、`completed` 結果内の `value` になります。

```typescript
type CodeModeOutput = { type: "text"; text: string } | { type: "json"; value: unknown };
```

ルール: 出力順序はゲストの呼び出し順序と一致します。出力は `maxOutputBytes` によって上限が設定されます。シリアライズできない値はプレーン文字列またはエラーに変換されます。バイナリ値はサポートされません。画像とファイルはコードモードブリッジではなく、通常の OpenClaw ツールを介して転送されます。

## ツールカタログ

非表示のカタログには、有効なポリシーフィルタリング後のツールが次の順序で含まれます。OpenClaw コアツール、同梱 Plugin ツール、外部 Plugin ツール、MCP ツール、そして現在の実行に対してクライアントが提供したツールです。

カタログ ID は単一の実行内では安定しており、可能な場合は同等のツールセット間で決定論的です。実際の形式:

```text
<source>:<owner>:<tool-name>
```

ここで `<source>` は `openclaw`、`mcp`、または `client` です（Plugin ツールでは Plugin ID を `<owner>` として `openclaw` を使用し、コアツールでは `openclaw:core:*` を使用します）。
例:

```text
openclaw:core:message
openclaw:browser:browser_request
mcp:github:create_issue
client:app:select_file
```

カタログでは、コードモード制御ツール（`exec`、`wait`、`tool_search_code`、`tool_search`、`tool_describe`、`tool_call`）と直接実行専用ツールが省略されます。制御ツールはカタログを介して再帰的に呼び出してはなりません。直接実行専用ツールは、その構造化された結果が QuickJS ブリッジを通過できないため、モデルから引き続き参照できます。

MCP エントリは実行スコープのカタログに残るため、ポリシー、承認、フック、テレメトリ、トランスクリプトへの投影、および正確なツール ID が通常のツール実行と共有されます。ゲスト向けの `ALL_TOOLS`、`tools.search(...)`、`tools.describe(...)`、`tools.callValue(...)`、および `tools.call(...)` ビューでは、MCP エントリが省略されます。生成された `MCP.<server>.<tool>({ ...input })` 名前空間は、正確なカタログ ID に逆解決され、同じエグゼキューターパスを介してディスパッチされます。

## Tool Search との連携

コードモードが有効な実行では、OpenClaw の Tool Search モデルサーフェスよりコードモードが優先されます。

`tools.codeMode.enabled` が true でコードモードが有効になる場合:

- OpenClaw は、`tool_search_code`、`tool_search`、`tool_describe`、または `tool_call` をモデルから参照可能なツールとして公開しない。
- 同じカタログ化の概念がゲストランタイム内に移動する。
- ゲストランタイムは、コンパクトな `ALL_TOOLS` メタデータと、MCP 以外のツール向けの検索、説明、呼び出しヘルパーを受け取る。
- MCP 呼び出しでは、`tools.call(...)` の代わりに、生成された `MCP` 名前空間とその `$api()` ヘッダーを使用する。
- ネストされた呼び出しは、Tool Search が使用するものと同じ OpenClaw エグゼキューターパスを介してディスパッチされる。

アクティブな実行においてコードモードが置き換える OpenClaw のコンパクトカタログブリッジについては、[Tool Search](/ja-JP/tools/tool-search) を参照してください。

## ツール名と衝突

モデルから参照可能な `exec` ツールが、コードモードツールです。通常の OpenClaw シェル `exec` ツールが有効な場合、モデルからは非表示になり、他のツールと同様にカタログ化されます。

ゲストランタイム内では:

- ポリシーで許可されている場合、`tools.call("openclaw:core:exec", input)` はシェル exec ツールを呼び出せる。
- `tools.exec(...)` は、シェル exec のカタログエントリに曖昧さのない安全な名前がある場合にのみインストールされる。
- コードモードの `exec` ツールを、`tools` を介して再帰的に利用することはできない。

2 つのツールが同じ安全な簡易名に正規化される場合、OpenClaw は簡易関数を省略し、`tools.call(id, input)` の使用を必須とします。

## ネストされたツール実行

ネストされたすべてのツール呼び出しはホストブリッジを通過して OpenClaw に再び入り、アクティブなエージェント ID、セッション ID とキー、送信者とチャネルのコンテキスト、サンドボックスポリシー、承認ポリシー、Plugin の `before_tool_call` フック、中止シグナル、利用可能な場合のストリーミング更新、および軌跡／監査イベントを保持します。

ネストされた呼び出しは実際のツール呼び出しとしてトランスクリプトに投影されるため、サポートバンドルでは発生した内容を確認できます。この投影では、親のコードモードツール呼び出しとネストされたツール ID が識別されます。

ネストされた呼び出しは、`maxPendingToolCalls` まで並列実行できます。

## 実行とスナップショットのライフサイクル

各コードモード実行は、`runId` をキーとするプロセス内マップで追跡されます（ディスクやデータベースには永続化されません）。`exec`/`wait` は、`completed`、`waiting`、または `failed` の 3 つの結果ステータスのいずれかを返します。

- `waiting` 結果は、`wait` が再開するか有効期限が切れるまで、QuickJS スナップショット、保留中のブリッジリクエスト、およびスコープメタデータ（エージェント実行 ID、セッション ID／キー）を保存する。
- 期限切れ、誤ったセッション、誤った実行、および不明または再開中の `runId` 値は、個別の終了ステータスを生成しない。代わりに、`code mode
run is unavailable or expired.` や `code mode run belongs to a different
session.` などのメッセージを持つ `failed` 結果（`code: "invalid_input"`）として表面化する。
- 実行のスナップショットは、`completed` または `failed` に確定すると直ちにマップから削除される。また、Gateway のシャットダウン時にも破棄される（再起動後には何も残らず、これは一時的なランタイム状態である）。
- 読み取り専用の処理では、`exec` で `restartSafe: true` を設定できる。その場合 OpenClaw は、副作用のあるカタログ呼び出しと Plugin 名前空間を実行前に拒否し、一時停止された結果を再実行可能としてマークする。再起動によって `wait` が中断された場合、[再起動からの復旧](/ja-JP/gateway/restart-recovery) はプロセスローカルなスナップショットを復元する代わりに、トランスクリプトからターンを再構築する。復旧ターン自体は、監査済みの読み取り専用コアツールと、明示的に再実行可能な Plugin ツールに引き続き制限される。
- OpenClaw は、プロセスごとに同時に一時停止できる実行数を制限し（64）、その上限を超える新しい一時停止を `too many suspended code mode
runs.` で拒否する。

スナップショットストレージは、実行ごとの `maxSnapshotBytes`、前述のプロセスごとの一時停止実行上限、および `snapshotTtlSeconds` によって制限されます。

## QuickJS-WASI ランタイム

OpenClaw は `quickjs-wasi` を所有パッケージの直接依存関係として読み込み、無関係な依存関係のためにインストールされた推移的なコピーには依存しません。

ランタイムの責務: QuickJS-WASI WebAssembly モジュールをコンパイル／読み込みする。コードモードの実行または再開ごとに分離された VM を 1 つ作成する。安定した名前でホストコールバックを登録する。メモリと割り込みの制限を設定する。JavaScript を評価する。保留中のジョブを処理し終える。一時停止された VM の状態をスナップショット化する。`wait` 用にスナップショットを復元する。終了ステータスの後に VM ハンドルとスナップショットを破棄する。

ランタイムは OpenClaw のメインイベントループ外にある Node.js ワーカースレッドで実行されます。ゲストの無限ループが Gateway プロセスを無期限にブロックしてはなりません。ワーカーの割り込みハンドラーは、ゲストコードの協調とは無関係に実時間タイムアウトを適用します。

## TypeScript

TypeScript のサポートはソース変換のみです。入力として受け付けるのは 1 つの TypeScript コード文字列で、出力は QuickJS-WASI によって評価される JavaScript 文字列です。型チェック、モジュール解決、および `import`/`require` はありません。診断は `failed` 結果として返されます。

TypeScript コンパイラは TypeScript セルに対してのみ遅延読み込みされます。プレーン JavaScript セルおよび無効なコードモードでは読み込まれません。

## セキュリティ境界

モデルコードは信頼できません。ランタイムは多層防御を使用します。

- QuickJS-WASI をメインイベントループ外のワーカースレッドで実行する
- `quickjs-wasi` を Codex や推移的パッケージ経由ではなく、直接依存関係として読み込む
- ゲスト内にはファイルシステム、ネットワーク、サブプロセス、モジュールインポート、環境変数、またはホストのグローバルオブジェクトがない
- QuickJS のメモリ制限と割り込み制限に加え、親プロセスの実時間タイムアウトを使用する
- 出力、スナップショット、ログ、および保留中の呼び出しに上限を適用する
- 狭く制限された JSON アダプターを介してホストブリッジの値をシリアライズする
- ホストエラーをプレーンなゲストエラーに変換し、ホストレルムのオブジェクトは決して渡さない
- タイムアウト、中止、セッション終了、または期限切れの際にスナップショットを破棄する
- `exec`、`wait`、および Tool Search 制御ツールへの再帰的アクセスを拒否する
- 簡易名の衝突によってカタログヘルパーが隠されることを防ぐ

サンドボックスはセキュリティ層の 1 つです。高リスクなデプロイでは、運用者が OS レベルの強化を引き続き必要とする場合があります。

## エラーコード

```typescript
type CodeModeErrorCode =
  | "invalid_input"
  | "runtime_unavailable"
  | "timeout"
  | "output_limit_exceeded"
  | "snapshot_limit_exceeded"
  | "internal_error";
```

`invalid_input` は、不正な `exec`/`wait` 引数、無効化された言語、拒否されたモジュールアクセス、TypeScript 変換の失敗、不明／期限切れ／スコープ不一致の `runId` 値、および一時停止された実行が多すぎる場合を対象とします。`runtime_unavailable` は、起動に失敗した、またはゼロ以外の終了コードで終了した QuickJS ワーカーを対象とします。

ゲストに返されるエラーはプレーンデータです。ホストの `Error` インスタンス、スタックオブジェクト、プロトタイプ、およびホスト関数は QuickJS 内に渡されません。

## テレメトリ

各結果の `telemetry` フィールドは、非表示カタログのサイズとソース別内訳（`openclaw`/`mcp`/`client` の件数）、実行カタログに対する累積の検索／説明／呼び出し回数、およびモデルから参照可能なツール名（`exec`、`wait`、ならびに保持された直接実行専用ツール）を報告します。

テレメトリには、既存の OpenClaw 軌跡ポリシーで許可される範囲を超えて、シークレット、生の環境値、または秘匿化されていないツール入力を含めてはなりません。

## デバッグ

コードモードの動作が通常のツール実行と異なる場合は、対象を絞ったモデル転送ログを使用します。

```bash
OPENCLAW_DEBUG_CODE_MODE=1 \
OPENCLAW_DEBUG_MODEL_TRANSPORT=1 \
OPENCLAW_DEBUG_MODEL_PAYLOAD=tools \
OPENCLAW_DEBUG_SSE=events \
openclaw gateway
```

ペイロード形状のデバッグには、`OPENCLAW_DEBUG_MODEL_PAYLOAD=full-redacted` を使用します。
これは、モデルリクエストのサイズ制限および編集済み JSON スナップショットをログに記録します。プロンプトやメッセージテキストが引き続き表示される可能性があるため、
デバッグ中にのみ使用してください。

ストリームのデバッグには、`OPENCLAW_DEBUG_SSE=peek` を使用して、最初の 5 件の
編集済み SSE イベントをログに記録します。また、コードモードのサーフェスが有効になった後、最終的なプロバイダー
ペイロードに `exec` が正確に 1 つ、`wait` が正確に 1 つ、および承認済みの
直接専用ツールのみが含まれていない場合、コードモードはフェイルクローズします。

## 実装の構成

- 設定コントラクト: `tools.codeMode`
- カタログビルダー: 有効なツールからコンパクトなエントリおよび ID マップへ変換
- モデルサーフェスアダプター: 表示ツールを制御ツールおよび直接ツールに置換
- QuickJS-WASI ランタイムアダプター: 読み込み、評価、スナップショット、復元、破棄
- ワーカースーパーバイザー: タイムアウト、中止、クラッシュの分離
- ブリッジアダプター: JSON セーフなホストコールバックおよび結果の配信
- TypeScript 変換アダプター
- スナップショットストア: TTL、サイズ上限、実行／セッションのスコープ
- ネストされたツール呼び出しの軌跡プロジェクション
- テレメトリカウンターおよび診断

実装では Tool Search のカタログおよびエグゼキューターの概念を再利用しますが、
サンドボックスとして `node:vm` の子を使用しません。

## 検証チェックリスト

コードモードのカバレッジでは、以下を実証する必要があります。

- 無効な設定では、既存のツール公開状態が変更されない
- `enabled: true` を含まないオブジェクト設定では、コードモードが無効なままになる
- 有効な設定では、実行時にツールが有効な場合、`exec`、`wait`、および必須の直接専用ツールのみが
  モデルに公開される
- ツールを使用しない未加工の実行、`disableTools`、および空の許可リストでは、
  コードモードのペイロード検証がトリガーされない
- カタログの対象となる有効な非 MCP ツールがすべて `ALL_TOOLS` に表示される
- 直接専用ツールはモデルに表示されたままとなり、`ALL_TOOLS` には表示されない
- 拒否されたツールは `ALL_TOOLS` に表示されない
- `tools.search`、`tools.describe`、`tools.callValue`、および `tools.call` が OpenClaw ツールで動作する
- `API.list("mcp")` および `API.read("mcp/<server>.d.ts")` は、ブリッジ／ツール呼び出しなしで TypeScript 形式の
  MCP 宣言を公開する
- MCP 名前空間 `$api()` は、スキーマのインラインフォールバックとして引き続き利用できる
- MCP 名前空間の呼び出しは、1 つのオブジェクト入力を持つ表示可能な MCP ツールで動作し、
  直接 MCP カタログエントリは `tools.*` に存在しない
- Tool Search の制御ツールは、モデルサーフェスと非表示カタログの両方から非表示になる
- ネストされた呼び出しで、承認およびフックの動作が維持される
- シェル `exec` はモデルには表示されないが、許可されている場合はカタログ ID で呼び出せる
- 再帰的なコードモードの `exec` および `wait` は、ゲストコードから呼び出せない
- 無効なパスまたは JavaScript のみのパスで TypeScript を読み込まずに、TypeScript 入力が変換および評価される
- `import`、`require`、ファイルシステム、ネットワーク、および環境へのアクセスが失敗する
- 無限ループがタイムアウトし、Gateway をブロックできない
- メモリ上限の超過によりゲスト VM が終了する
- 完了した呼び出しと中断された呼び出しの両方で、出力およびスナップショットの上限が適用される
- `wait` が中断されたスナップショットを再開し、最終値を返す
- 期限切れ、中止済み、誤ったセッション、および不明な `runId` の値が失敗する
- トランスクリプトの再生および永続化で、コードモードの制御呼び出しが維持される
- トランスクリプトおよびテレメトリに、ネストされたツール呼び出しが明確に表示される

## E2E テスト計画

ランタイムを変更する場合は、以下を統合テストまたはエンドツーエンドテストとして実行します。

1. `tools.codeMode.enabled: false` を使用して Gateway を起動する。
2. 少数の直接ツールセットを使用してエージェントターンを送信する。
3. モデルに表示されるツールが変更されていないことを確認する。
4. `tools.codeMode.enabled: true` を使用して再起動する。
5. OpenClaw、Plugin、MCP、およびクライアントのテストツールを使用してエージェントターンを送信する。
6. モデルに表示されるツールのリストが `exec`、`wait`、および設定された
   直接専用ツールのみであることを確認する。
7. `exec` で `ALL_TOOLS` を読み取り、カタログの対象となる有効なテスト
   ツールが存在し、直接専用ツールが存在しないことを確認する。
8. `exec` で、`tools.search`、`tools.describe`、および `tools.callValue`
   （または未加工の `tools.call`）を介して OpenClaw／Plugin／クライアントツールを呼び出す。
9. `exec` で `API.list("mcp")` および `API.read("mcp/<server>.d.ts")` を呼び出し、
   宣言ファイルが表示可能な MCP ツールを記述していることを確認する。
10. `exec` で `MCP.<server>.<tool>({ ...input })` を介して MCP ツールを呼び出し、
    直接 MCP カタログエントリが `ALL_TOOLS` および
    `tools.*` に存在しないことを確認する。
11. 拒否されたツールが存在せず、推測した ID では呼び出せないことを確認する。
12. `exec` が `waiting` を返した後に解決される、ネストされたツール呼び出しを開始する。
13. `wait` を呼び出し、復元された VM がツール結果を受け取ることを確認する。
14. 最終回答に、復元後に生成された出力が含まれることを確認する。
15. タイムアウト、中止、およびスナップショットの期限切れによってランタイム状態がクリーンアップされることを確認する。
16. 軌跡をエクスポートし、ネストされた呼び出しが親の
    コードモード呼び出しの下に表示されることを確認する。

このページのドキュメントのみを変更する場合でも、`pnpm check:docs` を実行する必要があります。

## 関連項目

- コードモードスクリプトからエージェントをファンアウトオーケストレーションするための [Swarm](/tools/swarm)
- [Tool Search](/ja-JP/tools/tool-search)
- [エージェントランタイム](/ja-JP/concepts/agent-runtimes)
- [Exec ツール](/ja-JP/tools/exec)
- [コード実行](/ja-JP/tools/code-execution)
