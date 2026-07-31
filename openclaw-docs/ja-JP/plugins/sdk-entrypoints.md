---
read_when:
    - defineToolPlugin、definePluginEntry、または defineChannelPluginEntry の正確な型シグネチャが必要です
    - 登録モード（full、setup、CLI メタデータ）の違いを理解したい場合
    - エントリーポイントのオプションを確認しています
sidebarTitle: Entry Points
summary: defineToolPlugin、definePluginEntry、defineChannelPluginEntry、defineSetupPluginEntry のリファレンス
title: Plugin のエントリーポイント
x-i18n:
    generated_at: "2026-07-26T10:26:13Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: e64fe1d65531fea8f266aa23b73064daf2ed2c5c43af8bb08ea57e347fe566f4
    source_path: plugins/sdk-entrypoints.md
    workflow: 16
---

すべてのプラグインは、デフォルトのエントリオブジェクトをエクスポートします。SDK は、
各エントリ形状に対応するヘルパーとして `defineToolPlugin`、`definePluginEntry`、
`defineChannelPluginEntry`、`defineSetupPluginEntry` を提供します。

<Tip>
  **手順を追った説明をお探しですか？** ステップごとのガイドについては、[ツールプラグイン](/ja-JP/plugins/tool-plugins)、
  [チャネルプラグイン](/ja-JP/plugins/sdk-channel-plugins)、または
  [プロバイダープラグイン](/ja-JP/plugins/sdk-provider-plugins)を参照してください。
</Tip>

## パッケージエントリ

インストール済みプラグインでは、ソースエントリとビルド済みエントリの両方を
`package.json` の `openclaw` フィールドで指定します。

```json
{
  "openclaw": {
    "extensions": ["./src/index.ts"],
    "runtimeExtensions": ["./dist/index.js"],
    "setupEntry": "./src/setup-entry.ts",
    "runtimeSetupEntry": "./dist/setup-entry.js"
  }
}
```

- `extensions` と `setupEntry` はソースエントリであり、ワークスペースおよび git
  チェックアウトでの開発に使用されます。
- `runtimeExtensions` と `runtimeSetupEntry` は、インストール済み
  パッケージで推奨されます。これにより、npm パッケージは実行時の TypeScript コンパイルを省略できます。
- `runtimeExtensions` が存在する場合、配列の長さが `extensions` と
  一致している必要があります（エントリは位置ごとに対応します）。`runtimeSetupEntry` には `setupEntry` が必要です。
- `runtimeExtensions`/`runtimeSetupEntry` アーティファクトが宣言されているにもかかわらず
  存在しない場合、インストールまたは検出はパッケージングエラーで失敗します。OpenClaw は
  暗黙にソースへフォールバックしません。以下のソースフォールバックは、
  ランタイムエントリがまったく宣言されていない場合にのみ適用されます。
- インストール済みパッケージが TypeScript ソースエントリのみを宣言している場合、OpenClaw は
  対応するビルド済みの `dist/*.js`（または `.mjs`/`.cjs`）ピアを探して使用します。
  見つからない場合は TypeScript ソースへフォールバックします。
- すべてのエントリパスは、プラグインパッケージのディレクトリ内に収まる必要があります。ランタイム
  エントリや推論されたビルド済み JS ピアが存在しても、パッケージ外を指す `extensions` または
  `setupEntry` ソースパスが有効になるわけではありません。

## `defineToolPlugin`

**インポート:** `openclaw/plugin-sdk/tool-plugin`

エージェントツールのみを追加するプラグイン向けです。ソースを小さく保ち、TypeBox スキーマから設定と
ツールパラメーターの型を推論し、通常の戻り値を OpenClaw のツール結果形式でラップし、
`openclaw plugins build` がプラグインマニフェスト（`contracts.tools`、
`configSchema`）へ書き込む静的メタデータを公開します。

```typescript
import { Type } from "typebox";
import { defineToolPlugin } from "openclaw/plugin-sdk/tool-plugin";

export default defineToolPlugin({
  id: "stock-quotes",
  name: "Stock Quotes",
  description: "Fetch stock quotes.",
  configSchema: Type.Object({
    apiKey: Type.Optional(Type.String({ description: "API key." })),
  }),
  tools: (tool) => [
    tool({
      name: "quote",
      label: "Quote",
      description: "Fetch a quote.",
      parameters: Type.Object({
        symbol: Type.String({ description: "Ticker symbol." }),
      }),
      outputSchema: Type.Object(
        {
          symbol: Type.String(),
          hasKey: Type.Boolean(),
        },
        { additionalProperties: false },
      ),
      execute: async ({ symbol }, config) => ({ symbol, hasKey: Boolean(config.apiKey) }),
    }),
  ],
});
```

- `configSchema` は省略可能です。省略すると厳密な空オブジェクトスキーマが使用されます
  （生成されるマニフェストには引き続き `configSchema` が含まれます）。
- `execute` は通常の文字列または JSON シリアライズ可能な値を返します。ヘルパーは、
  元の（文字列化されていない）戻り値を `details` に設定した
  テキストツール結果としてラップします。
- `outputSchema` は、Code Mode および Tool Search 向けに、その元の `details` 値を
  必要に応じて記述します。カタログ呼び出しは、実行前に無効なスキーマを拒否し、
  返却前に最終値を検証します。
- カスタムツール結果向けに、`openclaw/plugin-sdk/tool-results` は
  `textResult` と `jsonResult` をエクスポートします。
- ツール名は静的であるため、`openclaw plugins build` は
  手作業で名前を重複記述せずに、宣言されたツールから `contracts.tools` を導出します。
- ランタイムの読み込みは引き続き厳密です。インストール済みプラグインには、
  `openclaw.plugin.json` および `package.json` の `openclaw.extensions` が必要です。OpenClaw は、
  不足しているマニフェストデータを推論するためにプラグインコードを実行することはありません。

## `definePluginEntry`

**インポート:** `openclaw/plugin-sdk/plugin-entry`

プロバイダープラグイン、高度なツールプラグイン、フックプラグイン、および
メッセージングチャネル**ではない**ものすべてに使用します。

```typescript
import { definePluginEntry } from "openclaw/plugin-sdk/plugin-entry";

export default definePluginEntry({
  id: "my-plugin",
  name: "My Plugin",
  description: "Short summary",
  register(api) {
    api.registerProvider({/* ... */});
    api.registerTool({/* ... */});
  },
});
```

| フィールド                | 型                                                               | 必須     | デフォルト          |
| ------------------------- | ---------------------------------------------------------------- | -------- | ------------------- |
| `id`                      | `string`                                                         | はい     | -                   |
| `name`                    | `string`                                                         | はい     | -                   |
| `description`             | `string`                                                         | はい     | -                   |
| `kind`                    | `string`（非推奨、以下を参照）                                   | いいえ   | -                   |
| `configSchema`            | `OpenClawPluginConfigSchema \| () => OpenClawPluginConfigSchema` | いいえ   | 空オブジェクトスキーマ |
| `reload`                  | `OpenClawPluginReloadRegistration`                               | いいえ   | -                   |
| `nodeHostCommands`        | `OpenClawPluginNodeHostCommand[]`                                | いいえ   | -                   |
| `securityAuditCollectors` | `OpenClawPluginSecurityAuditCollector[]`                         | いいえ   | -                   |
| `register`                | `(api: OpenClawPluginApi) => void`                               | はい     | -                   |

- `id` は、`openclaw.plugin.json` マニフェストと一致する必要があります。
- 外部セッションカタログでは、
  `openclaw/plugin-sdk/session-catalog` と
  `api.registerSessionCatalog({ id, label, list, read, continueSession?, archive? })` を使用します。
  コアは `sessions.catalog.*` Gateway メソッドを所有します。プロバイダーは RPC を登録せずに、
  ホスト、セッション、正規化されたトランスクリプトのプロジェクションを返します。
  リストプロバイダーは、各ホストの処理が確定するたびに、省略可能な `onHost(host)` コールバックを
  呼び出す必要があります。返されるホスト配列は、最終的な互換性スナップショットとして引き続き必須です。
- `kind` は非推奨です。代わりに、`openclaw.plugin.json` マニフェストの
  `kind` フィールドで排他的スロット（`"memory"` または
  `"context-engine"`）を宣言してください。ランタイムエントリの `kind` は、
  古いプラグイン向けの互換性フォールバックとしてのみ残されています。
- `configSchema` は遅延評価のための関数にできます。OpenClaw は最初のアクセス時に
  スキーマを解決してメモ化するため、負荷の高いスキーマビルダーは一度だけ実行されます。
- `nodeHostCommands` ディスクリプターでは `isAvailable({ config, env })` を定義できます。
  `false` を返すと、そのコマンドと機能はヘッドレス Node の Gateway 宣言から省略されます。
  OpenClaw は Node ローカルの起動設定に対してこれを評価します。コマンドハンドラーは、
  呼び出された際にも引き続き利用可能性を検証する必要があります。

## `defineChannelPluginEntry`

**インポート:** `openclaw/plugin-sdk/channel-core`

`definePluginEntry` をチャネル固有の配線でラップします。`api.registerChannel({ plugin })` を自動的に
呼び出し、省略可能なルートヘルプ CLI メタデータの接続点を公開し、
登録モードに基づいて `registerFull` を制限します。

```typescript
import { defineChannelPluginEntry } from "openclaw/plugin-sdk/channel-core";

export default defineChannelPluginEntry({
  id: "my-channel",
  name: "My Channel",
  description: "Short summary",
  plugin: myChannelPlugin,
  setRuntime: setMyRuntime,
  registerCliMetadata(api) {
    api.registerCli(/* ... */);
  },
  registerFull(api) {
    api.registerGatewayMethod(/* ... */);
  },
});
```

| フィールド            | 型                                                               | 必須     | デフォルト          |
| --------------------- | ---------------------------------------------------------------- | -------- | ------------------- |
| `id`                  | `string`                                                         | はい     | -                   |
| `name`                | `string`                                                         | はい     | -                   |
| `description`         | `string`                                                         | はい     | -                   |
| `plugin`              | `ChannelPlugin`                                                  | はい     | -                   |
| `configSchema`        | `OpenClawPluginConfigSchema \| () => OpenClawPluginConfigSchema` | いいえ   | 空オブジェクトスキーマ |
| `setRuntime`          | `(runtime: PluginRuntime) => void`                               | いいえ   | -                   |
| `registerCliMetadata` | `(api: OpenClawPluginApi) => void`                               | いいえ   | -                   |
| `registerFull`        | `(api: OpenClawPluginApi) => void`                               | いいえ   | -                   |

コールバックは登録モードごとに実行されます（完全な表は
[登録モード](#registration-mode)を参照）。

- `setRuntime` は、`"cli-metadata"` と
  `"tool-discovery"` を除くすべてのモードで実行されます。通常は
  `createPluginRuntimeStore` を介して、ここにランタイム参照を保存します。
- `registerCliMetadata` は、`"cli-metadata"`、`"discovery"`、
  `"full"` で実行されます。チャネルが所有する CLI ディスクリプターの標準的な配置場所として使用すると、
  ルートヘルプによるアクティベーションを防ぎ、検出スナップショットに静的コマンドメタデータを含め、
  通常の CLI 登録とプラグインの完全読み込みとの互換性を維持できます。
- `registerFull` は、`"full"` と `"tool-discovery"` でのみ実行されます。
  `"tool-discovery"` では、チャネル登録の_代わりに_実行されます。OpenClaw は
  `registerChannel`/`setRuntime` を完全にスキップし、`registerFull` のみを
  呼び出します。そのため、スタンドアロンのツール検出または実行にチャネルが必要とするプロバイダーやツールの登録は、
  通常のチャネル設定の背後ではなく、ここに配置する必要があります。
- 検出登録は非アクティベーション型ですが、インポートを行わないわけではありません。OpenClaw は、
  スナップショットを構築するために、信頼済みのプラグインエントリとチャネルプラグインモジュールを評価する場合があります。
  トップレベルのインポートに副作用を持たせず、ソケット、クライアント、ワーカー、サービスは
  `"full"` 専用のパス内に配置してください。
- `definePluginEntry` と同様に、`configSchema` は遅延ファクトリーにできます。OpenClaw は、
  最初のアクセス時に解決されたスキーマをメモ化します。

CLI 登録:

- 遅延読み込みしつつルート CLI
  の解析ツリーから消えないようにする、プラグイン所有のルート CLI コマンドには `api.registerCli(..., { descriptors: [...] })` を使用します。
  ディスクリプター名は、先頭を英字または数字とし、英字、数字、ハイフン、
  アンダースコアのみに一致する必要があります。OpenClaw はそれ以外の
  形式を拒否し、ヘルプを表示する前に説明から端末制御シーケンスを
  除去します。レジストラーが公開するすべてのトップレベルコマンドルートを網羅してください。
  `commands` のみの場合は、引き続き先行読み込みされる互換性パスが使用されます。
- ペアリング済み Node の機能コマンドには `api.registerNodeCliFeature(...)` を使用し、
  `openclaw nodes` の配下（
  `registerCli(registrar, { parentPath: ["nodes"], ... })` と同等）に配置されるようにします。
- その他のネストされたプラグインコマンドでは、`parentPath` を追加し、レジストラーに渡される
  `program` オブジェクトにコマンドを登録します。OpenClaw はプラグインを呼び出す前に、
  これを親コマンドへ解決します。
- チャンネルプラグインでは、`registerCliMetadata` から CLI ディスクリプターを登録し、
  `registerFull` はランタイム専用の処理に集中させます。
- `registerFull` で Gateway RPC メソッドも登録する場合は、
  プラグイン固有のプレフィックス配下に置きます。予約済みのコア管理名前空間（`config.*`、
  `exec.approvals.*`、`wizard.*`、`update.*`）は常に
  `operator.admin` に強制変換されます。

## `defineSetupPluginEntry`

**インポート:** `openclaw/plugin-sdk/channel-core`

軽量な `setup-entry.ts` ファイル用です。ランタイムや CLI の配線を行わず、
`{ plugin }` のみを返します。

```typescript
import { defineSetupPluginEntry } from "openclaw/plugin-sdk/channel-core";

export default defineSetupPluginEntry(myChannelPlugin);
```

チャンネルが無効または未設定の場合や、遅延読み込みが有効な場合、
OpenClaw は完全なエントリの代わりにこれを読み込みます。これが重要になる状況については、
[セットアップと設定](/ja-JP/plugins/sdk-setup#setup-entry)を参照してください。

`defineSetupPluginEntry(...)` は、用途を限定した次のセットアップヘルパーファミリーと組み合わせます。

| インポート                              | 用途                                                                                                                                                                            |
| ----------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `openclaw/plugin-sdk/setup-runtime` | ランタイムセーフなセットアップヘルパー: `createSetupTranslator`、インポートセーフなセットアップパッチアダプター、ルックアップ注記の出力、`promptResolvedAllowFrom`、`splitSetupEntries`、委譲セットアッププロキシ |
| `openclaw/plugin-sdk/channel-setup` | オプションインストールのセットアップサーフェス                                                                                                                                                    |
| `openclaw/plugin-sdk/setup-tools`   | セットアップ／インストール CLI、アーカイブ、ドキュメントのヘルパー                                                                                                                                       |

重量な SDK、CLI 登録、長期間稼働するランタイムサービスは、
完全なエントリに保持してください。

セットアップとランタイムのサーフェスを分離するバンドル済みワークスペースチャンネルでは、
代わりに `openclaw/plugin-sdk/channel-entry-contract` の
`defineBundledChannelSetupEntry(...)` を使用できます。これにより、セットアップ
エントリでセットアップセーフなプラグイン／シークレットのエクスポートを維持しながら、ランタイム
セッターも公開できます。

```typescript
import { defineBundledChannelSetupEntry } from "openclaw/plugin-sdk/channel-entry-contract";

export default defineBundledChannelSetupEntry({
  importMetaUrl: import.meta.url,
  plugin: {
    specifier: "./channel-plugin-api.js",
    exportName: "myChannelPlugin",
  },
  runtime: {
    specifier: "./runtime-api.js",
    exportName: "setMyChannelRuntime",
  },
  registerSetupRuntime(api) {
    api.registerHttpRoute({
      path: "/my-channel/events",
      auth: "plugin",
      handler: async (req, res) => {
        /* セットアップセーフなルート */
      },
    });
  },
});
```

セットアップフローで、完全なチャンネルエントリが読み込まれる前に軽量なランタイムセッターまたは
セットアップセーフな Gateway サーフェスが本当に必要な場合にのみ使用してください。
`registerSetupRuntime` は `"setup-runtime"` の読み込み時にのみ実行されます。遅延された
完全なアクティベーションより前に存在する必要がある、設定専用のルートまたはメソッドに
限定してください。

## 登録モード

`api.registrationMode` は、プラグインがどのように読み込まれたかを示します。

| モード               | タイミング                                               | 登録するもの                                                                                                        |
| ------------------ | -------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------- |
| `"full"`           | 通常の Gateway 起動                             | すべて                                                                                                              |
| `"discovery"`      | 読み取り専用のケイパビリティ検出                     | チャンネル登録と静的 CLI ディスクリプター。エントリコードは読み込まれる場合がありますが、ソケット、ワーカー、クライアント、サービスは起動しません |
| `"tool-discovery"` | 特定のプラグインのツールを一覧表示または実行するためのスコープ付き読み込み | ケイパビリティ／ツール登録のみ。チャンネルのアクティベーションは行いません                                                                |
| `"setup-only"`     | 無効または未設定のチャンネル                      | チャンネル登録のみ                                                                                               |
| `"setup-runtime"`  | ランタイムを利用できるセットアップフロー                  | チャンネル登録と、完全なエントリが読み込まれる前に必要な軽量ランタイムのみ                               |
| `"cli-metadata"`   | ルートヘルプ／CLI メタデータの取得                   | CLI ディスクリプターのみ                                                                                                    |

`defineChannelPluginEntry` はこの分離を自動的に処理します。チャンネルに
`definePluginEntry` を直接使用する場合は、自分でモードを確認し、
`"tool-discovery"` ではチャンネル登録がスキップされることに注意してください。

```typescript
register(api) {
  if (
    api.registrationMode === "cli-metadata" ||
    api.registrationMode === "discovery" ||
    api.registrationMode === "full"
  ) {
    api.registerCli(/* ... */);
    if (api.registrationMode === "cli-metadata") return;
  }

  if (api.registrationMode === "tool-discovery") {
    // ケイパビリティ専用サーフェス（プロバイダー／ツール）を登録し、チャンネルは登録しません。
    return;
  }

  api.registerChannel({ plugin: myPlugin });
  if (api.registrationMode !== "full") return;

  // 重量なランタイム専用登録
  api.registerService(/* ... */);
}
```

長期間稼働するサービスは、そのサービスコンテキストを通じて小さな無効化イベントまたは
ライフサイクルイベントを送出できます。

```typescript
api.registerService({
  id: "index-events",
  start(ctx) {
    ctx.gatewayEvents?.emit("changed", { revision: 1 }, { scope: "operator.read" });
  },
});
```

OpenClaw はこれに `plugin.<plugin-id>.changed` という名前空間を付けます。イベント名は
小文字の単一セグメントである必要があり、ペイロードはサイズが制限された JSON、
スコープは `operator.read`、`operator.write`、または `operator.admin` でなければなりません。
エミッターはサービスの存続期間中のみ存在し、停止後または起動失敗後には無効化されます。
認可済みクライアントがプラグインのスコープ付き Gateway メソッドを通じて
正規状態を再読み込みできるように、完全なレコードよりもバージョンまたは無効化の
ペイロードを優先してください。

検出モードでは、アクティベーションを行わないレジストリスナップショットを構築します。OpenClaw が
チャンネルケイパビリティと静的 CLI ディスクリプターを登録できるように、
プラグインエントリとチャンネルプラグインオブジェクトが評価される場合があります。検出時のモジュール
評価は信頼済みではあるものの軽量なものとして扱ってください。トップレベルでネットワーククライアント、
サブプロセス、リスナー、データベース接続、バックグラウンドワーカー、
認証情報の読み取り、その他の稼働中ランタイムの副作用を発生させないでください。

`"setup-runtime"` は、バンドル済みチャンネルの完全なランタイムへ再進入せずに、
セットアップ専用の起動サーフェスが存在しなければならない期間として扱います。適しているのは、
チャンネル登録、セットアップセーフな HTTP ルート、セットアップセーフな Gateway メソッド、
委譲セットアップヘルパーです。重量なバックグラウンドサービス、CLI レジストラー、
プロバイダー／クライアント SDK のブートストラップは、引き続き `"full"` に置きます。

## プラグインの形態

OpenClaw は、読み込まれたプラグインを登録動作によって分類します。

| 形態                 | 説明                                        |
| --------------------- | -------------------------------------------------- |
| **plain-capability**  | 1 種類のケイパビリティ（例: プロバイダーのみ）           |
| **hybrid-capability** | 複数種類のケイパビリティ（例: プロバイダー + 音声） |
| **hook-only**         | フックのみで、ケイパビリティなし                        |
| **non-capability**    | ツール／コマンド／サービスはあるが、ケイパビリティなし        |

プラグインの形態を確認するには `openclaw plugins inspect <id>` を使用します。

## 関連項目

- [SDK の概要](/ja-JP/plugins/sdk-overview) - 登録 API とサブパスのリファレンス
- [ランタイムヘルパー](/ja-JP/plugins/sdk-runtime) - `api.runtime` と `createPluginRuntimeStore`
- [セットアップと設定](/ja-JP/plugins/sdk-setup) - マニフェスト、セットアップエントリ、遅延読み込み
- [チャンネルプラグイン](/ja-JP/plugins/sdk-channel-plugins) - `ChannelPlugin` オブジェクトの構築
- [プロバイダープラグイン](/ja-JP/plugins/sdk-provider-plugins) - プロバイダーの登録とフック
