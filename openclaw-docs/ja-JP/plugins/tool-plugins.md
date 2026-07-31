---
read_when:
    - エージェントツールの追加のみを行うシンプルな OpenClaw Plugin を構築したい場合
    - プラグインマニフェストのメタデータを手動で記述する代わりに、defineToolPlugin を使用する場合
    - ツール専用Pluginのスキャフォールディング、生成、検証、テスト、または公開を行う必要がある場合
sidebarTitle: Tool Plugins
summary: defineToolPlugin と openclaw plugins init/build/validate を使用して、シンプルな型付きエージェントツールを構築する
title: ツールプラグイン
x-i18n:
    generated_at: "2026-07-26T09:13:49Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: ac23d15ba79cbdd1d8b8eab7c87007b44af16361b2866b14123e18f816bf4075
    source_path: plugins/tool-plugins.md
    workflow: 16
---

`defineToolPlugin` は、エージェントが呼び出せるツールのみを追加する Plugin を構築します。チャネル、モデルプロバイダー、フック、サービス、セットアップバックエンドは追加しません。Plugin のランタイムコードを読み込まずに OpenClaw がツールを検出するために必要なマニフェストメタデータを生成します。

プロバイダー、チャネル、フック、サービス、または複数の機能を持つ Plugin については、代わりに
[Plugin の構築](/ja-JP/plugins/building-plugins)、[チャネル Plugin](/ja-JP/plugins/sdk-channel-plugins)、
または[プロバイダー Plugin](/ja-JP/plugins/sdk-provider-plugins)から始めてください。

## 要件

- Node 22.22.3+、Node 24.15+、または Node 25.9+。
- TypeScript ESM パッケージ出力。
- `dependencies` 内の `typebox`（`devDependencies` だけでは不可。生成された
  Plugin が実行時にインポートします）。
- `openclaw/plugin-sdk/tool-plugin` をエクスポートする最初のバージョンである
  `openclaw >=2026.5.17`。
- `dist/`、`openclaw.plugin.json`、および
  `package.json` を同梱するパッケージルート。

## クイックスタート

```bash
openclaw plugins init stock-quotes --name "Stock Quotes"
cd stock-quotes
npm install
npm run plugin:build
npm run plugin:validate
npm test
```

`plugins init` は以下をスキャフォールドします。

| ファイル                   | 目的                                                           |
| ---------------------- | ----------------------------------------------------------------- |
| `src/index.ts`         | 1 つの `echo` ツールを持つ `defineToolPlugin` エントリ                     |
| `src/index.test.ts`    | ツール一覧を検証するメタデータテスト                             |
| `tsconfig.json`        | `dist/` への NodeNext TypeScript 出力                             |
| `vitest.config.ts`     | `src/**/*.test.ts` 用の Vitest 設定                              |
| `package.json`         | スクリプト、ランタイム依存関係、`openclaw.extensions: ["./dist/index.js"]` |
| `openclaw.plugin.json` | 初期ツール用に生成されたマニフェストメタデータ                  |

`npm run plugin:build` は `npm run build`（tsc）を実行してから
`openclaw plugins build --entry ./dist/index.js` を実行します。`npm run plugin:validate` は
再ビルドして `openclaw plugins validate --entry ./dist/index.js` を実行します。
検証に成功すると、次のように出力されます。

```text
Plugin stock-quotes is valid.
```

`openclaw plugins init <id>` のオプション：

| フラグ                 | デフォルト            | 効果                                 |
| -------------------- | ------------------ | -------------------------------------- |
| `--directory <path>` | `<id>`             | 出力ディレクトリ                       |
| `--name <name>`      | タイトルケースの `<id>` | 表示名                           |
| `--type <type>`      | `tool`             | スキャフォールドの種類：`tool` または `provider`    |
| `--force`            | オフ                | 既存の出力ディレクトリを上書き |

## ツールを作成する

`defineToolPlugin` は、Plugin の識別情報、オプションの設定スキーマ、静的なツール一覧を受け取ります。パラメーター型と設定型は TypeBox スキーマから推論されます。

```typescript
import { Type } from "typebox";
import { defineToolPlugin } from "openclaw/plugin-sdk/tool-plugin";

export default defineToolPlugin({
  id: "stock-quotes",
  name: "Stock Quotes",
  description: "株価スナップショットを取得します。",
  configSchema: Type.Object({
    apiKey: Type.Optional(Type.String({ description: "株価 API キー。" })),
    baseUrl: Type.Optional(Type.String({ description: "株価 API のベース URL。" })),
  }),
  tools: (tool) => [
    tool({
      name: "stock_quote",
      label: "株価",
      description: "株価スナップショットを取得します。",
      parameters: Type.Object({
        symbol: Type.String({ description: "ティッカーシンボル（例：OPEN）。" }),
      }),
      outputSchema: Type.Object(
        {
          symbol: Type.String(),
          configured: Type.Boolean(),
          baseUrl: Type.String(),
        },
        { additionalProperties: false },
      ),
      async execute({ symbol }, config, context) {
        context.signal?.throwIfAborted();
        return {
          symbol: symbol.toUpperCase(),
          configured: Boolean(config.apiKey),
          baseUrl: config.baseUrl ?? "https://api.example.com",
        };
      },
    }),
  ],
});
```

ツール名は安定した API です。一意かつ小文字で、コアツールや他の Plugin との衝突を避けるのに十分具体的な名前を選んでください。

## オプションツールとファクトリーツール

モデルに送信する前にユーザーがツールを明示的に許可リストへ追加する必要がある場合は、`optional: true` を設定します。`openclaw plugins build` は対応する
`toolMetadata.<tool>.optional` マニフェストエントリを書き込むため、OpenClaw は Plugin のランタイムコードを読み込まずに、そのツールがオプションであることを認識できます。

```typescript
tool({
  name: "workflow_run",
  description: "外部ワークフローを実行します。",
  parameters: Type.Object({ goal: Type.String() }),
  optional: true,
  execute: ({ goal }) => ({ queued: true, goal }),
});
```

ツールを作成する前にランタイムのツールコンテキストが必要な場合、つまり特定の実行で無効にする、サンドボックスの状態を確認する、またはランタイムヘルパーをバインドする場合は、`factory` を使用します。具体的なツールは実行時に構築されますが、メタデータは静的なままです。

```typescript
tool({
  name: "local_workflow",
  description: "サンドボックス化されたセッションの外部でローカルワークフローを実行します。",
  parameters: Type.Object({ goal: Type.String() }),
  optional: true,
  factory({ api, toolContext }) {
    if (toolContext.sandboxed) {
      return null;
    }
    return createLocalWorkflowTool(api);
  },
});
```

ファクトリーでも、固定のツール名をあらかじめ宣言します。Plugin がツール名を動的に計算する場合や、ツールをフック、サービス、プロバイダー、またはコマンドと組み合わせる場合は、`definePluginEntry` を直接使用します。

## 戻り値

`defineToolPlugin` は、通常の戻り値を OpenClaw のツール結果形式でラップします。

- モデルにそのままのテキストを表示する場合は、文字列を返します。
- モデルに整形済み JSON を表示し、OpenClaw が元の値を `details` に保持する場合は、JSON 互換の値を返します。

```typescript
tool({
  name: "echo_text",
  description: "入力テキストをそのまま返します。",
  parameters: Type.Object({
    input: Type.String(),
  }),
  execute: ({ input }) => input,
});
```

```typescript
tool({
  name: "echo_json",
  description: "入力を構造化 JSON としてそのまま返します。",
  parameters: Type.Object({
    input: Type.String(),
  }),
  execute: ({ input }) => ({ input, length: input.length }),
});
```

カスタム `AgentToolResult` が必要な場合や、既存の `api.registerTool` 実装を再利用する場合は、ファクトリーツールを使用します。

## 出力コントラクト

ツールが安定した JSON 互換データを返す場合は、`outputSchema` を追加します。これは `content` 内の整形済みテキストではなく、`AgentToolResult.details` に保存された元の値を記述します。

```typescript
tool({
  name: "shipment_list",
  description: "出荷一覧を表示します。",
  parameters: Type.Object({
    buyer: Type.Optional(Type.String()),
  }),
  outputSchema: Type.Array(
    Type.Object(
      {
        id: Type.String(),
        buyer: Type.String(),
        paid: Type.Boolean(),
        tons: Type.Number(),
      },
      { additionalProperties: false },
    ),
  ),
  execute: ({ buyer }) => listShipments(buyer),
});
```

[コードモード](/ja-JP/tools/code-mode)と[ツール検索](/ja-JP/tools/tool-search)は、このスキーマを範囲が限定された TypeScript 形式の出力ヒントに変換します。これにより、モデルは結果の形状を確認するためにもう 1 回モデルターンを費やす代わりに、1 つのプログラム内で既知の結果を呼び出して変換できます。

OpenClaw はカタログ呼び出しを実行する前にスキーマをコンパイルし、ツールフックの処理後に最終的な `details` の値を検証してから、ブリッジを通じて返します。
無効なスキーマではツールを実行できず、結果が一致しない場合は完了済みの呼び出しが失敗します。構造化されたエラーバリアントを含め、例外をスローしないすべての結果バリアントを含めてください。結果が安定していない場合は、スキーマを省略してください。信頼済みの出力メタデータはモデルから参照可能になる場合があるため、スキーマの説明にシークレットや機密値を含めないでください。
完全でコンパクトな出力ヒントが必要な場合は、オブジェクトの各階層で `{ additionalProperties: false }` を使用してください。オープンまたは切り詰められたスキーマも `tools.describe(...)` を通じて利用できますが、完全なクイックインデックスコントラクトとしては提示されません。

ファクトリーツールは、返す具体的な `AnyAgentTool` に `outputSchema` を宣言します。静的な `tool({ factory })` 宣言は、ランタイムツールと乖離する可能性があるため、個別の出力スキーマを受け付けません。

## 設定

`configSchema` はオプションです。省略すると OpenClaw は厳密な空オブジェクトスキーマを適用し、生成されたマニフェストには引き続き `configSchema` が含まれます。

```typescript
export default defineToolPlugin({
  id: "no-config-tools",
  name: "No Config Tools",
  description: "設定を必要としないツールを追加します。",
  tools: () => [],
});
```

`configSchema` がある場合、2 番目の `execute` 引数の型はそこから設定されます。

```typescript
const configSchema = Type.Object({
  apiKey: Type.String(),
});

export default defineToolPlugin({
  id: "configured-tools",
  name: "Configured Tools",
  description: "設定済みのツールを追加します。",
  configSchema,
  tools: (tool) => [
    tool({
      name: "configured_ping",
      description: "設定が利用可能かどうかを確認します。",
      parameters: Type.Object({}),
      execute: (_params, config) => ({ hasKey: config.apiKey.length > 0 }),
    }),
  ],
});
```

OpenClaw は Gateway 設定内の Plugin のエントリから Plugin 設定を読み取ります。ソースやドキュメントの例にシークレットをハードコードしないでください。Plugin のセキュリティモデルに従って、設定、環境変数、または SecretRef を使用してください。

## 生成されるメタデータ

OpenClaw は Plugin のランタイムコードをインポートする前に、Plugin マニフェストを読み取る必要があります。
`defineToolPlugin` はこのための静的メタデータを公開し、
`openclaw plugins build` はそれをパッケージに書き込みます。Plugin の ID、名前、説明、設定スキーマ、アクティベーション、またはツール名を変更した後は、ジェネレーターを再実行してください。

```bash
npm run build
openclaw plugins build --entry ./dist/index.js
```

1 ツールの Plugin 用に生成されるマニフェスト：

```json
{
  "id": "stock-quotes",
  "name": "Stock Quotes",
  "description": "株価スナップショットを取得します。",
  "version": "0.1.0",
  "configSchema": {
    "type": "object",
    "additionalProperties": false,
    "properties": {}
  },
  "activation": {
    "onStartup": true
  },
  "contracts": {
    "tools": ["stock_quote"]
  }
}
```

`contracts.tools` は重要な検出コントラクトです。インストール済みのすべての Plugin のランタイムを読み込むことなく、各ツールを所有する Plugin を OpenClaw に伝えます。マニフェストが古いと、ツールが検出対象から漏れたり、登録エラーが誤った Plugin のものと判断されたりする可能性があります。

## パッケージメタデータ

`openclaw plugins build` は、`package.json` も選択されたランタイムエントリに合わせます。

```json
{
  "type": "module",
  "files": ["dist", "openclaw.plugin.json", "README.md"],
  "dependencies": {
    "typebox": "^1.1.38"
  },
  "peerDependencies": {
    "openclaw": ">=2026.5.17"
  },
  "openclaw": {
    "extensions": ["./dist/index.js"]
  }
}
```

TypeScript のソースエントリではなく、ビルド済みの JavaScript（`./dist/index.js`）を同梱してください。
ソースエントリはワークスペース内のローカル開発でのみ機能します。

## CI で検証する

`plugins build --check` は、生成済みメタデータが古い場合、ファイルを書き換えずに失敗します。

```bash
npm run build
openclaw plugins build --entry ./dist/index.js --check
openclaw plugins validate --entry ./dist/index.js
npm test
```

OpenClaw SDK の互換性フィールドには TypeScript の `@deprecated` アノテーションがあり、エディターでは移行警告として表示されます。CI でこれらを強制するには、
[`@typescript-eslint/no-deprecated`](https://typescript-eslint.io/rules/no-deprecated/)
などの型情報を使用するルールを有効にしてください。
Oxlint は型情報を使用しないため、これらのアノテーションを強制できません。そのため、生成される
`plugins init` スキャフォールドには非推奨 API 用の lint 設定は追加されません。

`plugins validate` は以下を確認します。

- `openclaw.plugin.json` が存在し、通常のマニフェストローダーを通過します。
- 現在のエントリは `defineToolPlugin` メタデータをエクスポートします。
- 生成されたマニフェストフィールドがエントリメタデータと一致します。
- `contracts.tools` が宣言されたツール名と一致します。
- `package.json` は `openclaw.extensions` が選択したランタイムエントリを指すようにします。

## ローカルでインストールして確認する

別の OpenClaw チェックアウトまたはインストール済み CLI から、パッケージパスをインストールします。

```bash
openclaw plugins install ./stock-quotes
openclaw plugins inspect stock-quotes --runtime
```

パッケージ化されたスモークテストでは、まずパッケージ化してから tarball をインストールします。

```bash
npm pack
openclaw plugins install npm-pack:./openclaw-plugin-stock-quotes-0.1.0.tgz
openclaw plugins inspect stock-quotes --runtime --json
```

インストール後、Gateway を再起動または再読み込みし、エージェントに
ツールの使用を依頼します。ツールが表示されない場合は、コードを変更する前に Plugin のランタイムと
有効なツールカタログを確認してください（[トラブルシューティング](#troubleshooting)を参照）。

## 公開

パッケージの準備ができたら、ClawHub を通じて公開します。`clawhub package publish`
はソースとして、ローカルフォルダー、GitHub リポジトリ（`owner/repo[@ref]`）、または
tarball URL を受け取ります。

```bash
clawhub package publish ./stock-quotes --dry-run
clawhub package publish ./stock-quotes
```

明示的な ClawHub ロケーターを指定してインストールします。

```bash
openclaw plugins install clawhub:your-org/stock-quotes
```

起動移行期間中は修飾なしの npm パッケージ指定でも引き続き npm からインストールされますが、
OpenClaw plugins の検索と配布には ClawHub が推奨されます。
所有者スコープとリリースレビューについては、[ClawHub での公開](/ja-JP/clawhub/publishing)を参照してください。

## トラブルシューティング

### `plugin entry not found: ./dist/index.js`

選択したエントリファイルが存在しません。`npm run build` を実行してから、
`openclaw plugins build --entry ./dist/index.js` または
`openclaw plugins validate --entry ./dist/index.js` を再実行してください。

### `plugin entry does not expose defineToolPlugin metadata`

エントリが `defineToolPlugin` によって作成された値をエクスポートしていません。モジュールの
デフォルトエクスポートが `defineToolPlugin(...)` の結果であることを確認するか、
`--entry` で正しいエントリを渡してください。

### `openclaw.plugin.json generated metadata is stale`

マニフェストがエントリメタデータと一致しなくなっています。次を実行してください。

```bash
npm run build
openclaw plugins build --entry ./dist/index.js
```

`openclaw.plugin.json` と `package.json` の両方の変更をコミットしてください。

### `package.json openclaw.extensions must include ./dist/index.js`

パッケージメタデータが別のランタイムエントリを指しています。ジェネレーターが
パッケージメタデータをリリース対象のエントリに合わせるように、
`openclaw plugins build --entry ./dist/index.js` を実行してください。

### `Cannot find package 'typebox'`

ビルドされた Plugin が実行時に `typebox` をインポートしています。これを `dependencies` に残したまま、
再インストールと再ビルドを行い、検証を再実行してください。

### インストール後にツールが表示されない

次の項目を順番に確認してください。

1. `openclaw plugins inspect <plugin-id> --runtime`
2. `openclaw plugins validate --root <plugin-root> --entry ./dist/index.js`
3. `openclaw.plugin.json` に、想定されるツール名を含む `contracts.tools` があります。
4. `package.json` に `openclaw.extensions: ["./dist/index.js"]` があります。
5. Plugin のインストール後に Gateway が再起動または再読み込みされています。

## 関連項目

- [Plugin のビルド](/ja-JP/plugins/building-plugins)
- [Plugin のエントリポイント](/ja-JP/plugins/sdk-entrypoints)
- [Plugin SDK のサブパス](/ja-JP/plugins/sdk-subpaths)
- [Plugin マニフェスト](/ja-JP/plugins/manifest)
- [Plugins CLI](/ja-JP/cli/plugins)
- [ClawHub での公開](/ja-JP/clawhub/publishing)
