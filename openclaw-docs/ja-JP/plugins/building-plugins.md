---
doc-schema-version: 1
read_when:
    - 新しい OpenClaw Plugin を作成する場合
    - Plugin 開発のクイックスタートが必要です
    - チャネル、プロバイダー、CLI バックエンド、ツール、フックのドキュメントから選択しようとしている場合
sidebarTitle: Getting Started
summary: 数分で初めての OpenClaw Plugin を作成する
title: Plugin の構築
x-i18n:
    generated_at: "2026-07-26T09:30:59Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 9d156ea305e46d3ca311a0b2cfc42e2c4522f6f10eb70cdd5526d9e9fcd7d4ef
    source_path: plugins/building-plugins.md
    workflow: 16
---

Plugin はコアを変更せずに OpenClaw を拡張します。Plugin はメッセージング
チャネル、モデルプロバイダー、ローカル CLI バックエンド、エージェントツール、フック、メディアプロバイダー、
または Plugin が所有する別の機能を追加できます。

外部 Plugin を OpenClaw リポジトリに追加する必要はありません。パッケージを
[ClawHub](/clawhub) に公開すると、ユーザーは次のコマンドでインストールできます。

```bash
openclaw plugins install clawhub:<package-name>
```

ローンチ移行期間中は、プレフィックスなしのパッケージ指定も引き続き npm からインストールされます。ClawHub で解決する場合は
`clawhub:` プレフィックスを使用してください。

## 要件

- Node 22.22.3+、Node 24.15+、または Node 25.9+、および `npm` または `pnpm`。
- TypeScript ESM モジュール。
- リポジトリ内のバンドル済み Plugin を扱う場合は、リポジトリをクローンして `pnpm install` を実行します。
  OpenClaw は `extensions/*` ワークスペースパッケージから
  バンドル済み Plugin を検出するため、ソースチェックアウトでの Plugin 開発では pnpm のみを使用できます。

## Plugin の形態を選択する

<CardGroup cols={2}>
  <Card title="チャネル Plugin" icon="messages-square" href="/ja-JP/plugins/sdk-channel-plugins">
    OpenClaw をメッセージングプラットフォームに接続します。
  </Card>
  <Card title="プロバイダー Plugin" icon="cpu" href="/ja-JP/plugins/sdk-provider-plugins">
    モデル、メディア、検索、取得、音声、またはリアルタイムプロバイダーを追加します。
  </Card>
  <Card title="CLI バックエンド Plugin" icon="terminal" href="/ja-JP/plugins/cli-backend-plugins">
    OpenClaw のモデルフォールバックを通じてローカル AI CLI を実行します。
  </Card>
  <Card title="ツール Plugin" icon="wrench" href="/ja-JP/plugins/tool-plugins">
    エージェントツールを登録します。
  </Card>
</CardGroup>

## クイックスタート

必須のエージェントツールを 1 つ登録して、最小構成のツール Plugin を構築します。これは
実用的な Plugin の最小構成であり、パッケージ、マニフェスト、エントリポイント、および
ローカルでの検証を網羅します。

<Steps>
  <Step title="パッケージメタデータを作成する">
    <CodeGroup>

```json package.json
{
  "name": "@myorg/openclaw-my-plugin",
  "version": "1.0.0",
  "type": "module",
  "dependencies": {
    "typebox": "1.1.39"
  },
  "peerDependencies": {
    "openclaw": ">=2026.3.24-beta.2"
  },
  "openclaw": {
    "extensions": ["./index.ts"],
    "compat": {
      "pluginApi": ">=2026.3.24-beta.2",
      "minGatewayVersion": "2026.3.24-beta.2"
    },
    "build": {
      "openclawVersion": "2026.3.24-beta.2",
      "pluginSdkVersion": "2026.3.24-beta.2"
    }
  }
}
```

```json openclaw.plugin.json
{
  "id": "my-plugin",
  "name": "My Plugin",
  "description": "Adds a custom tool to OpenClaw",
  "contracts": {
    "tools": ["my_tool"]
  },
  "activation": {
    "onStartup": true
  },
  "configSchema": {
    "type": "object",
    "additionalProperties": false
  }
}
```

    </CodeGroup>

    公開する外部 Plugin のランタイムエントリは、ビルド済みの JavaScript
    ファイルを参照する必要があります。エントリポイントの完全な契約については、[SDK エントリポイント](/ja-JP/plugins/sdk-entrypoints)を
    参照してください。

    設定がない場合でも、すべての Plugin にマニフェストが必要です。OpenClaw が
    すべての Plugin ランタイムを即時に読み込むことなく所有者を検出できるように、ランタイムツールを
    `contracts.tools` に含める必要があります。`activation.onStartup` は意図を持って設定してください。
    この例では Gateway の起動時に読み込みます。

    ホストから信頼される Plugin サーフェスもマニフェストによって制限され、インストール済み
    Plugin では明示的な宣言が必要です。`api.registerAgentToolResultMiddleware(...)` では
    各対象ランタイムを `contracts.agentToolResultMiddleware` に列挙する必要があり、
    `api.registerTrustedToolPolicy(...)` では各ポリシー ID を
    `contracts.trustedToolPolicies` に含める必要があります。これらの宣言により、インストール時の
    検査とランタイム登録の整合性が保たれます。

    すべてのマニフェストフィールドについては、[Plugin マニフェスト](/ja-JP/plugins/manifest)を参照してください。

  </Step>

  <Step title="ツールを登録する">
    ```typescript index.ts
    import { Type } from "typebox";
    import { definePluginEntry } from "openclaw/plugin-sdk/plugin-entry";

    export default definePluginEntry({
      id: "my-plugin",
      name: "My Plugin",
      description: "Adds a custom tool to OpenClaw",
      register(api) {
        api.registerTool({
          name: "my_tool",
          description: "Echo one input value",
          parameters: Type.Object({ input: Type.String() }),
          outputSchema: Type.Object(
            { input: Type.String() },
            { additionalProperties: false },
          ),
          async execute(_id, params) {
            const details = { input: params.input };
            return {
              content: [{ type: "text", text: `Got: ${params.input}` }],
              details,
            };
          },
        });
      },
    });
    ```

    チャネル以外の Plugin では `definePluginEntry` を使用します。チャネル Plugin では代わりに
    `openclaw/plugin-sdk/core` の `defineChannelPluginEntry` を使用します。

  </Step>

  <Step title="ランタイムをテストする">
    インストール済みまたは外部の Plugin では、読み込まれたランタイムを確認します。

    ```bash
    openclaw plugins inspect my-plugin --runtime --json
    ```

    Plugin が CLI コマンドを登録する場合は、そのコマンドも実行して出力を確認します。
    例: `openclaw demo-plugin ping`。

    このリポジトリ内のバンドル済み Plugin では、OpenClaw は `extensions/*` ワークスペースから
    ソースチェックアウトの Plugin パッケージを検出します。最も対象範囲の近い
    テストを実行します。

    ```bash
    pnpm test extensions/my-plugin/
    pnpm check
    ```

  </Step>

  <Step title="パッケージのインストールをテストする">
    パッケージとして公開可能な Plugin を公開する前に、ユーザーが利用するものと同じ
    インストール形態をテストします。まずビルドステップを追加し、`openclaw.extensions` などの
    ランタイムエントリが `./dist/index.js` のようなビルド済み JavaScript を参照するようにして、
    `npm pack` にその `dist/` 出力が含まれていることを確認します。TypeScript のソースエントリは、
    ソースチェックアウトおよびローカル開発パス専用です。

    次に Plugin をパックし、`npm-pack:` を使用して tarball をインストールします。

    ```bash
    npm pack --pack-destination /tmp
    openclaw plugins install npm-pack:/tmp/<plugin-package>.tgz --force
    openclaw plugins inspect my-plugin --runtime --json
    ```

    `npm-pack:` は OpenClaw が管理する Plugin ごとの npm プロジェクトを使用するため、
    ソースチェックアウトのテストでは見逃す可能性のあるランタイム依存関係の誤りを検出します。これは
    パッケージと依存関係の構成を検証するものであり、カタログに紐付けられた公式の信頼性を検証するものではありません。
    ランタイムのインポートは `dependencies` または `optionalDependencies` に含める必要があります。
    `devDependencies` のみに残された依存関係は、管理対象の
    ランタイムプロジェクトにはインストールされません。

    公式または特権的な Plugin の動作に対する最終検証として、生のアーカイブやパスからのインストールを
    使用しないでください。生のソースはローカルデバッグには有用ですが、
    npm または ClawHub からのインストールと同じ依存関係パスを検証するものではありません。
    Plugin が信頼済みの公式 Plugin ステータスに依存する場合は、カタログに裏付けられた
    公式インストール、または公式の信頼性が記録される公開済みパッケージパスを通じた 2 つ目の検証を
    追加してください。インストールルートと依存関係の所有権の詳細については、
    [Plugin の依存関係解決](/ja-JP/plugins/dependency-resolution)を参照してください。

  </Step>

  <Step title="公開する">
    公開する前にパッケージを検証します。

    ```bash
    clawhub package publish your-org/your-plugin --dry-run
    clawhub package publish your-org/your-plugin
    ```

    正規の ClawHub パッケージスニペットは `docs/snippets/plugin-publish/` にあります。

  </Step>

  <Step title="インストールする">
    公開済みパッケージを ClawHub 経由でインストールします。

    ```bash
    openclaw plugins install clawhub:your-org/your-plugin
    ```

  </Step>
</Steps>

<a id="registering-agent-tools"></a>

## ツールの登録

ツールは必須または任意にできます。必須ツールは、Plugin が有効な場合は常に
使用できます。任意ツールでは、OpenClaw が所有元の Plugin ランタイムを
読み込む前に、ユーザーによる明示的なオプトインが必要です。

ツールファクトリーは、`deliveryContext`、利用可能な場合はアクティブなプラットフォーム会話の
`nativeChannelId`、および `requesterSenderId` を含む、信頼済みのランタイムコンテキストを受け取ります。

```typescript
register(api) {
  api.registerTool(
    {
      name: "workflow_tool",
      description: "Run a workflow",
      parameters: Type.Object({ pipeline: Type.String() }),
      outputSchema: Type.Object(
        { pipeline: Type.String() },
        { additionalProperties: false },
      ),
      async execute(_id, params) {
        return {
          content: [{ type: "text", text: params.pipeline }],
          details: { pipeline: params.pipeline },
        };
      },
    },
    { optional: true },
  );
}
```

`outputSchema` は任意です。[コードモード](/tools/code-mode)と[ツール検索](/ja-JP/tools/tool-search)で使用される
構造化された `details` 値を記述します。カタログ呼び出しは実行前に無効なスキーマを拒否し、
ツールフックの後に最終値を検証します。安定した JSON 結果を持たないツールでは省略してください。
完全な契約については、[ツール Plugin](/ja-JP/plugins/tool-plugins#output-contracts)を参照してください。

`api.registerTool(...)` で登録するすべてのツールは、Plugin マニフェストでも
宣言する必要があります。

```json
{
  "contracts": {
    "tools": ["workflow_tool"]
  },
  "toolMetadata": {
    "workflow_tool": {
      "optional": true
    }
  }
}
```

ユーザーは `tools.allow` でオプトインします。

```json5
{
  tools: { allow: ["workflow_tool"] }, // or ["my-plugin"] for every tool from one plugin
}
```

任意ツールは、ツールをモデルに公開するかどうかを制御します。モデルがツールまたはフックを選択した後、
アクションを実行する前に承認を求める必要がある場合は、
[Plugin 権限リクエスト](/ja-JP/plugins/plugin-permission-requests)を使用してください。

副作用、一般的でないバイナリ、またはデフォルトで公開すべきでない機能には、
任意ツールを使用します。ツール名はコアツール名と競合してはなりません。競合した場合はスキップされ、
Plugin 診断で報告されます。不正な登録も同じ方法でスキップされ、報告されます。たとえば、空でない
`name` がない場合、`execute` が関数でない場合、またはツール記述子に `parameters`
オブジェクトがない場合です。

ツールファクトリーは、ランタイムから提供されるコンテキストオブジェクトを受け取ります。ツールが現在の
ターンでアクティブなモデルに応じてログ記録、表示、または動作の調整を行う必要がある場合は、`ctx.activeModel`
を使用します。これには `provider`、`modelId`、および `modelRef` が含まれることがあります。これは
情報提供用のランタイムメタデータとして扱い、ローカルオペレーター、インストール済み Plugin コード、
または変更された OpenClaw ランタイムに対するセキュリティ境界として扱わないでください。機密性の高い
ローカルツールでは、引き続き Plugin またはオペレーターによる明示的なオプトインを必須とし、
アクティブモデルのメタデータがない、または適切でない場合は安全側に失敗させる必要があります。

マニフェストは所有権と検出方法を宣言しますが、実行時には引き続き登録済みの
稼働中のツール実装が呼び出されます。OpenClaw がツールを明示的に許可リストへ追加するまで
その Plugin ランタイムを読み込まずに済むように、`toolMetadata.<tool>.optional: true` と
`api.registerTool(..., { optional: true })` の整合性を保ってください。

## インポート規則

目的別の SDK サブパスからインポートします。

```typescript
import { definePluginEntry } from "openclaw/plugin-sdk/plugin-entry";
import { createPluginRuntimeStore } from "openclaw/plugin-sdk/runtime-store";
```

Plugin パッケージ内では、内部インポートに `api.ts` や
`runtime-api.ts` などのローカルバレルファイルを使用します。自身の Plugin を
SDK パス経由でインポートしないでください。プロバイダー固有のヘルパーは、
その境界が真に汎用的でない限り、プロバイダーパッケージ内に保持する必要があります。

カスタム Gateway RPC メソッドは高度なエントリポイントです。Plugin 固有の
プレフィックスを使用してください。`config.*`、`exec.approvals.*`、`operator.admin.*`、`wizard.*`、
`update.*` などのコア管理名前空間は予約済みであり、
`operator.admin` として解決されます。
`openclaw/plugin-sdk/gateway-method-runtime` ブリッジは、`contracts.gatewayMethodDispatch: ["authenticated-request"]` を宣言する Plugin HTTP
ルート用に予約されています。

完全なインポートマップについては、[Plugin SDK の概要](/ja-JP/plugins/sdk-overview)を参照してください。

OpenClaw SDK の互換性フィールドには TypeScript の `@deprecated` アノテーションが付いており、
エディターでは移行に関する警告として表示されます。ビルド時にこれを強制するには、
[`@typescript-eslint/no-deprecated`](https://typescript-eslint.io/rules/no-deprecated/) のような
型情報を利用するルールを有効にしてください。
Oxlint は型情報を利用しないため、これらのアノテーションを強制できません。

## 提出前チェックリスト

<Check>**package.json** に正しい `openclaw` メタデータがある</Check>
<Check>**openclaw.plugin.json** マニフェストが存在し、有効である</Check>
<Check>エントリポイントが `defineChannelPluginEntry` または `definePluginEntry` を使用している</Check>
<Check>すべてのインポートが対象を絞った `plugin-sdk/<subpath>` パスを使用している</Check>
<Check>内部インポートが SDK の自己インポートではなく、ローカルモジュールを使用している</Check>
<Check>テストが成功する（`pnpm test <bundled-plugin-root>/my-plugin/`）</Check>
<Check>`pnpm check` が成功する（リポジトリ内 Plugin）</Check>

## ベータリリースに対するテスト

1. [openclaw/openclaw](https://github.com/openclaw/openclaw/releases) のリリース（`Watch` > `Releases`）をウォッチしてください。ベータタグは `v2026.3.N-beta.1` のような形式です。リリースのお知らせについては、X で [@openclaw](https://x.com/openclaw) をフォローすることもできます。
2. ベータタグが公開されたら、できるだけ早く Plugin をテストしてください。安定版までの猶予は通常、わずか数時間です。
3. テスト後、`plugin-forum` Discord チャンネル（[discord.gg/clawd](https://discord.gg/clawd)）にある Plugin のスレッドへ、`all good` または問題が発生した内容を投稿してください。スレッドがまだない場合は作成してください。
4. 問題が発生した場合は、`Beta blocker: <plugin-name> - <summary>` というタイトルの Issue を作成または更新し、`beta-blocker` ラベルを付けてください。スレッドに Issue へのリンクを記載してください。
5. `main` に、`fix(<plugin-id>): beta blocker - <summary>` というタイトルの PR を作成し、PR と Discord スレッドの両方に Issue へのリンクを記載してください。コントリビューターは PR にラベルを付けられないため、このタイトルがメンテナーと自動化システムに対する PR 側の合図になります。PR があるブロッカーはマージされますが、PR がないブロッカーがあってもそのままリリースされる可能性があります。
6. 連絡がなければ問題なしと見なされます。この期間を逃した場合、通常は修正が次のサイクルで取り込まれます。

## 次のステップ

<CardGroup cols={2}>
  <Card title="チャンネル Plugin" icon="messages-square" href="/ja-JP/plugins/sdk-channel-plugins">
    メッセージングチャンネル Plugin を構築する
  </Card>
  <Card title="プロバイダー Plugin" icon="cpu" href="/ja-JP/plugins/sdk-provider-plugins">
    モデルプロバイダー Plugin を構築する
  </Card>
  <Card title="CLI バックエンド Plugin" icon="terminal" href="/ja-JP/plugins/cli-backend-plugins">
    ローカル AI CLI バックエンドを登録する
  </Card>
  <Card title="SDK の概要" icon="book-open" href="/ja-JP/plugins/sdk-overview">
    インポートマップと登録 API のリファレンス
  </Card>
  <Card title="ランタイムヘルパー" icon="settings" href="/ja-JP/plugins/sdk-runtime">
    api.runtime を介した TTS、検索、サブエージェント
  </Card>
  <Card title="テスト" icon="test-tubes" href="/ja-JP/plugins/sdk-testing">
    テスト用ユーティリティとパターン
  </Card>
  <Card title="Plugin マニフェスト" icon="file-json" href="/ja-JP/plugins/manifest">
    完全なマニフェストスキーマのリファレンス
  </Card>
</CardGroup>

## 関連項目

- [Plugin フック](/ja-JP/plugins/hooks)
- [Plugin アーキテクチャ](/ja-JP/plugins/architecture)
