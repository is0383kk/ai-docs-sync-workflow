---
read_when:
    - Plugin にセットアップウィザードを追加しています
    - setup-entry.ts と index.ts の違いを理解する必要があります
    - Plugin の設定スキーマまたは package.json の openclaw メタデータを定義している場合
sidebarTitle: Setup and config
summary: セットアップウィザード、setup-entry.ts、設定スキーマ、package.json メタデータ
title: Plugin のセットアップと設定
x-i18n:
    generated_at: "2026-07-26T09:54:56Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: b07e3fa365939fa9c0885b31b7894f5e734313a7deef2297e316956063d97e45
    source_path: plugins/sdk-setup.md
    workflow: 16
---

Plugin のパッケージ化（`package.json` メタデータ）、マニフェスト（`openclaw.plugin.json`）、セットアップエントリ、設定スキーマのリファレンス。

<Tip>
**手順を追った解説をお探しですか？** ハウツーガイドでは、具体的な文脈に沿ってパッケージ化を説明しています：[チャンネル Plugin](/ja-JP/plugins/sdk-channel-plugins#step-1-package-and-manifest)と[プロバイダー Plugin](/ja-JP/plugins/sdk-provider-plugins#step-1-package-and-manifest)。
</Tip>

## パッケージメタデータ

`package.json` には、その Plugin が提供する内容を Plugin システムに伝える `openclaw` フィールドが必要です：

<Tabs>
  <Tab title="チャンネル Plugin">
    ```json
    {
      "name": "@myorg/openclaw-my-channel",
      "version": "1.0.0",
      "type": "module",
      "openclaw": {
        "extensions": ["./index.ts"],
        "setupEntry": "./setup-entry.ts",
        "channel": {
          "id": "my-channel",
          "label": "マイチャンネル",
          "blurb": "チャンネルの短い説明。"
        }
      }
    }
    ```
  </Tab>
  <Tab title="プロバイダー Plugin / ClawHub ベースライン">
    ```json openclaw-clawhub-package.json
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
  </Tab>
</Tabs>

<Note>
ClawHub で外部公開するには、`compat` と `build` が必要です。標準の公開用スニペットは `docs/snippets/plugin-publish/` にあります。
</Note>

### `openclaw` フィールド

<ParamField path="extensions" type="string[]">
  エントリポイントファイル（パッケージルートからの相対パス）。ワークスペースおよび git チェックアウトでの開発に使用できる有効なソースエントリです。
</ParamField>
<ParamField path="runtimeExtensions" type="string[]">
  `extensions` に対応するビルド済み JavaScript ファイルです。OpenClaw がインストール済み npm パッケージを読み込む場合に優先されます。ソースとビルド済みファイルの解決順序については、[SDK エントリポイント](/ja-JP/plugins/sdk-entrypoints)を参照してください。
</ParamField>
<ParamField path="setupEntry" type="string">
  セットアップ専用の軽量エントリ（任意）。
</ParamField>
<ParamField path="runtimeSetupEntry" type="string">
  `setupEntry` に対応するビルド済み JavaScript ファイルです。`setupEntry` も設定する必要があります。
</ParamField>
<ParamField path="plugin" type="object">
  `{ id, label }` のフォールバック用 Plugin 識別情報です。ID やラベルを導出できるチャンネル／プロバイダーメタデータが Plugin にない場合に使用されます。
</ParamField>
<ParamField path="channel" type="object">
  セットアップ、選択画面、クイックスタート、ステータス画面用のチャンネルカタログメタデータ。
</ParamField>
<ParamField path="install" type="object">
  インストールのヒント：`npmSpec`、`localPath`、`defaultChoice`、`minHostVersion`、`expectedIntegrity`、`allowInvalidConfigRecovery`、`requiredPlatformPackages`。
</ParamField>
<ParamField path="startup" type="object">
  起動時の動作フラグ。
</ParamField>
<ParamField path="compat" type="object">
  この Plugin がサポートする `pluginApi` バージョン範囲。ClawHub への外部公開には必須です。
</ParamField>

<Note>
プロバイダー ID（`providers: string[]`）はマニフェストメタデータであり、パッケージメタデータではありません。ここではなく `openclaw.plugin.json` で宣言してください。詳しくは [Plugin マニフェスト](/ja-JP/plugins/manifest)を参照してください。
</Note>

### `openclaw.channel`

`openclaw.channel` は、ランタイムを読み込む前にチャンネルを検出し、セットアップ画面に表示するための軽量なパッケージメタデータです。

### チャンネル所有のセットアップフィールド

チャンネル Plugin では、`defineChannelSetupContract(...)` を使用してランタイムコード内にセットアップフィールドを一度だけ定義し、それに対応するシリアライズ可能な投影を `openclaw.channel.setup.fields` で公開する必要があります。ランタイム定義は Plugin ローカルの入力型を推論し、ガイド付きと非対話型の両方の値を解析し、チャンネル固有のキーがコア型に混入しないようにします。パッケージメタデータにより、`openclaw channels add <channel-id> --help` と `openclaw channels add --channel <channel-id> --help` は Plugin を読み込まずに、選択されたチャンネルのオプションだけを検出できます。

```ts
import { defineChannelSetupContract } from "openclaw/plugin-sdk/channel-setup";

export const setupContract = defineChannelSetupContract({
  fields: {
    endpoint: {
      kind: "string",
      cli: { flags: "--endpoint <url>", description: "サービスエンドポイント" },
    },
    transport: {
      kind: "choice",
      choices: ["native", "container"],
      cli: { flags: "--transport <kind>", description: "トランスポート所有者" },
    },
  },
  adapter: {
    applyAccountConfig: ({ cfg, input }) => ({
      ...cfg,
      channels: { ...cfg.channels, example: input },
    }),
  },
});
```

```json
{
  "openclaw": {
    "channel": {
      "id": "example",
      "setup": {
        "fields": [
          {
            "key": "endpoint",
            "kind": "string",
            "cli": { "flags": "--endpoint <url>", "description": "サービスエンドポイント" }
          },
          {
            "key": "transport",
            "kind": "choice",
            "choices": ["native", "container"],
            "cli": { "flags": "--transport <kind>", "description": "トランスポート所有者" }
          }
        ]
      }
    }
  }
}
```

サポートされるフィールド種別は、`string`、`boolean`、`integer`、`string-list`、`choice` です。認証情報には `sensitive: true` を使用してください。各フィールドキーは、長形式 CLI フラグの属性名をキャメルケースにしたものと一致する必要があります。否定形も含まれ、たとえば `--api-token` に対しては `apiToken` となります。真偽値フィールドでは、肯定形と `--no-*` 形式の両方が必要な場合、`cli.negatedFlags` を追加できます。`channel`、`account`、アカウント表示用の `name` は、引き続き共有制御エンベロープです。

リリース済みの `setup`/`ChannelSetupInput` アダプターは、既存の外部 Plugin で引き続き使用できます。新しい Plugin では `setupContract` を公開してください。両方が存在する場合、OpenClaw は常にこちらを優先します。

| フィールド                                  | 型       | 意味                                                                 |
| -------------------------------------- | ---------- | ----------------------------------------------------------------------------- |
| `id`                                   | `string`   | 標準のチャンネル ID。                                                         |
| `label`                                | `string`   | プライマリチャンネルラベル。                                                        |
| `selectionLabel`                       | `string`   | `label` と異なる必要がある場合に使用する、選択／セットアップ用ラベル。                        |
| `detailLabel`                          | `string`   | 情報量の多いチャンネルカタログとステータス画面用のセカンダリ詳細ラベル。       |
| `docsPath`                             | `string`   | セットアップおよび選択リンク用のドキュメントパス。                                      |
| `docsLabel`                            | `string`   | チャンネル ID と異なる必要がある場合に、ドキュメントリンクで使用する上書きラベル。 |
| `blurb`                                | `string`   | オンボーディング／カタログ用の短い説明。                                         |
| `order`                                | `number`   | チャンネルカタログ内の並び順。                                               |
| `aliases`                              | `string[]` | チャンネル選択用の追加検索エイリアス。                                   |
| `preferOver`                           | `string[]` | このチャンネルより優先度を低くする Plugin／チャンネル ID。                |
| `systemImage`                          | `string`   | チャンネル UI カタログ用の任意のアイコン／システム画像名。                      |
| `selectionDocsPrefix`                  | `string`   | 選択画面でドキュメントリンクの前に表示するテキスト。                          |
| `selectionDocsOmitLabel`               | `boolean`  | 選択テキストで、ラベル付きドキュメントリンクの代わりにドキュメントパスを直接表示します。 |
| `selectionExtras`                      | `string[]` | 選択テキストに追加される短い文字列。                               |
| `markdownCapable`                      | `boolean`  | 送信時の書式設定判断において、このチャンネルが Markdown 対応であることを示します。      |
| `exposure`                             | `object`   | セットアップ、設定済み一覧、ドキュメント画面でのチャンネル表示制御。   |
| `quickstartAllowFrom`                  | `boolean`  | このチャンネルを標準クイックスタート `allowFrom` セットアップフローの対象にします。         |
| `forceAccountBinding`                  | `boolean`  | アカウントが 1 つしか存在しない場合でも、明示的なアカウント紐付けを必須にします。           |
| `preferSessionLookupForAnnounceTarget` | `boolean`  | このチャンネルの通知先を解決するときに、セッション検索を優先します。       |
| `setup`                                | `object`   | CLI オプションの遅延検出に使用する、シリアライズ可能なチャンネル所有のセットアップフィールド。   |

例：

```json
{
  "openclaw": {
    "channel": {
      "id": "my-channel",
      "label": "マイチャンネル",
      "selectionLabel": "マイチャンネル（セルフホスト）",
      "detailLabel": "マイチャンネル Bot",
      "docsPath": "/channels/my-channel",
      "docsLabel": "my-channel",
      "blurb": "Webhook ベースのセルフホスト型チャット連携。",
      "order": 80,
      "aliases": ["mc"],
      "preferOver": ["my-channel-legacy"],
      "selectionDocsPrefix": "ガイド：",
      "selectionExtras": ["Markdown"],
      "markdownCapable": true,
      "exposure": {
        "configured": true,
        "setup": true,
        "docs": true
      },
      "quickstartAllowFrom": true
    }
  }
}
```

`exposure` は以下をサポートします：

- `configured`：設定済み／ステータス形式の一覧画面にチャンネルを含める
- `setup`：対話型のセットアップ／設定選択画面にチャンネルを含める
- `docs`：ドキュメント／ナビゲーション画面でチャンネルを一般公開として示す

### `openclaw.install`

`openclaw.install` はパッケージメタデータであり、マニフェストメタデータではありません。

| フィールド                   | 型                                  | 意味                                                                              |
| ---------------------------- | ----------------------------------- | --------------------------------------------------------------------------------- |
| `clawhubSpec`                | `string`                            | インストール／更新およびオンボーディングのオンデマンドインストールフローに使用する正規の ClawHub 仕様。 |
| `npmSpec`                    | `string`                            | インストール／更新のフォールバックフローに使用する正規の npm 仕様。              |
| `localPath`                  | `string`                            | ローカル開発またはバンドルされたインストールのパス。                             |
| `defaultChoice`              | `"clawhub"` \| `"npm"` \| `"local"` | 複数のソースを利用できる場合に優先するインストールソース。                       |
| `minHostVersion`             | `string`                            | サポートされる最小 OpenClaw バージョン。`>=x.y.z` または `>=x.y.z-prerelease`。 |
| `expectedIntegrity`          | `string`                            | 固定インストールで期待される npm dist 整合性文字列。通常は `sha512-...`。  |
| `allowInvalidConfigRecovery` | `boolean`                           | バンドル Plugin の再インストールフローで、特定の古い設定による障害から復旧できるようにします。 |
| `requiredPlatformPackages`   | `string[]`                          | npm インストール時に検証される、必須のプラットフォーム固有 npm エイリアス。      |

<AccordionGroup>
  <Accordion title="オンボーディングの動作">
    対話型オンボーディングでは、オンデマンドインストールのサーフェスに `openclaw.install` を使用します。Plugin がランタイムの読み込み前にプロバイダー認証の選択肢またはチャネルのセットアップ／カタログメタデータを公開している場合、オンボーディングは ClawHub、npm、またはローカルインストールを提示し、Plugin をインストールまたは有効化してから、選択されたフローを続行できます。ClawHub の選択肢は `clawhubSpec` を使用し、存在する場合は優先されます。npm の選択肢には、レジストリの `npmSpec` を含む信頼済みカタログメタデータが必要です（正確なバージョンと `expectedIntegrity` は任意の固定値であり、設定されている場合はインストール／更新時に適用されます）。「何を表示するか」は `openclaw.plugin.json` に、「どのようにインストールするか」は `package.json` に保持してください。
  </Accordion>
  <Accordion title="minHostVersion の適用">
    `minHostVersion` が設定されている場合、インストールと非バンドルのマニフェストレジストリ読み込みの両方で適用されます。古いホストは外部 Plugin をスキップし、無効なバージョン文字列は拒否されます。バンドルされたソース Plugin は、ホストのチェックアウトと同じバージョンであると見なされます。
  </Accordion>
  <Accordion title="固定された npm インストール">
    固定された npm インストールでは、正確なバージョンを `npmSpec` に保持し、期待されるアーティファクトの整合性を追加します。

    ```json
    {
      "openclaw": {
        "install": {
          "npmSpec": "@wecom/wecom-openclaw-plugin@1.2.3",
          "expectedIntegrity": "sha512-REPLACE_WITH_NPM_DIST_INTEGRITY",
          "defaultChoice": "npm"
        }
      }
    }
    ```

  </Accordion>
  <Accordion title="allowInvalidConfigRecovery の適用範囲">
    `allowInvalidConfigRecovery` は、壊れた設定を一般的に迂回するためのものではありません。これはバンドル Plugin の限定的な復旧専用であり、バンドル Plugin のパスが欠落している場合や、同じ Plugin に対する古い `channels.<id>` エントリなど、既知のアップグレード残存物を再インストール／セットアップによって修復できるようにします。無関係な理由で設定が壊れている場合、インストールは引き続きフェイルクローズし、オペレーターに `openclaw doctor --fix` を実行するよう通知します。
  </Accordion>
</AccordionGroup>

### 完全読み込みの遅延

チャネル Plugin は、次の設定により遅延読み込みを選択できます。

```json
{
  "openclaw": {
    "extensions": ["./index.ts"],
    "setupEntry": "./setup-entry.ts",
    "startup": {
      "deferConfiguredChannelFullLoadUntilAfterListen": true
    }
  }
}
```

有効にすると、OpenClaw は設定済みのチャネルであっても、リッスン開始前の起動フェーズでは `setupEntry` のみを読み込みます。完全なエントリは、Gateway がリッスンを開始した後に読み込まれます。

<Warning>
`setupEntry` が、Gateway のリッスン開始前に必要なすべての要素（チャネル登録、HTTP ルート、Gateway メソッド）を登録する場合にのみ、遅延読み込みを有効にしてください。完全なエントリが必須の起動機能を所有している場合は、デフォルトの動作を維持してください。
</Warning>

セットアップ／完全エントリが Gateway RPC メソッドを登録する場合は、Plugin 固有のプレフィックスを使用してください。予約済みのコア管理名前空間（`config.*`、`exec.approvals.*`、`wizard.*`、`update.*`）は引き続きコアが所有し、常に `operator.admin` に正規化されます。

## Plugin マニフェスト

すべてのネイティブ Plugin は、パッケージルートに `openclaw.plugin.json` を含める必要があります。OpenClaw はこれを使用して、Plugin コードを実行せずに設定を検証します。

```json
{
  "id": "my-plugin",
  "name": "My Plugin",
  "description": "OpenClaw に My Plugin の機能を追加します",
  "configSchema": {
    "type": "object",
    "additionalProperties": false,
    "properties": {
      "webhookSecret": {
        "type": "string",
        "description": "Webhook 検証シークレット"
      }
    }
  }
}
```

チャネル Plugin では `channels` を追加します（プロバイダー Plugin では `providers` を追加します）。

```json
{
  "id": "my-channel",
  "channels": ["my-channel"],
  "configSchema": {
    "type": "object",
    "additionalProperties": false,
    "properties": {}
  }
}
```

設定のない Plugin でもスキーマを含める必要があります。空のスキーマも有効です。

```json
{
  "id": "my-plugin",
  "configSchema": {
    "type": "object",
    "additionalProperties": false
  }
}
```

完全なスキーマリファレンスについては、[Plugin マニフェスト](/ja-JP/plugins/manifest)を参照してください。

## ClawHub への公開

Skills と Plugin パッケージでは、ClawHub への公開コマンドが異なります。Plugin パッケージには、パッケージ固有のコマンドを使用してください。

```bash
clawhub package publish your-org/your-plugin --dry-run
clawhub package publish your-org/your-plugin
```

<Note>
`clawhub skill publish <path>` は、Plugin パッケージではなく Skills フォルダーを公開するための別のコマンドです。[ClawHub への公開](/ja-JP/clawhub/publishing)を参照してください。
</Note>

## セットアップエントリ

`setup-entry.ts` は `index.ts` の軽量な代替であり、OpenClaw がセットアップ用サーフェス（オンボーディング、設定修復、無効化されたチャネルの検査）のみを必要とする場合に読み込みます。

```typescript
// setup-entry.ts
import { defineSetupPluginEntry } from "openclaw/plugin-sdk/channel-core";
import { myChannelPlugin } from "./src/channel.js";

export default defineSetupPluginEntry(myChannelPlugin);
```

これにより、セットアップフロー中に重いランタイムコード（暗号化ライブラリ、CLI 登録、バックグラウンドサービス）を読み込まずに済みます。

セットアップに安全なエクスポートをサイドカーモジュールに保持するバンドル済みワークスペースチャネルでは、`defineSetupPluginEntry(...)` の代わりに `openclaw/plugin-sdk/channel-entry-contract` の `defineBundledChannelSetupEntry(...)` を使用できます。このバンドル契約では、オプションの `runtime` エクスポートもサポートされるため、セットアップ時のランタイム配線を軽量かつ明示的に保てます。

<AccordionGroup>
  <Accordion title="OpenClaw が完全なエントリの代わりに setupEntry を使用する場合">
    - チャネルが無効化されているものの、セットアップ／オンボーディング用サーフェスが必要な場合。
    - チャネルが有効化されているものの、未設定の場合。
    - 遅延読み込みが有効な場合（`deferConfiguredChannelFullLoadUntilAfterListen`）。

  </Accordion>
  <Accordion title="setupEntry が登録する必要があるもの">
    - チャネル Plugin オブジェクト（`defineSetupPluginEntry` 経由）。
    - Gateway のリッスン開始前に必要なすべての HTTP ルート。
    - 起動中に必要なすべての Gateway メソッド。

    これらの起動用 Gateway メソッドでも、`config.*` や `update.*` などの予約済みコア管理名前空間は避ける必要があります。

  </Accordion>
  <Accordion title="setupEntry に含めるべきでないもの">
    - CLI 登録。
    - バックグラウンドサービス。
    - 重いランタイムインポート（暗号化、SDK）。
    - 起動後にのみ必要な Gateway メソッド。

  </Accordion>
</AccordionGroup>

### 限定的なセットアップヘルパーのインポート

頻繁に実行されるセットアップ専用パスでは、セットアップサーフェスの一部のみが必要な場合、広範な `plugin-sdk/setup` アンブレラよりも限定的なセットアップヘルパーの境界を優先してください。

| インポートパス             | 用途                                                                                      | 主なエクスポート                                                                                                                                                                                                                                                                                                      |
| -------------------------- | ----------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `plugin-sdk/setup-runtime` | `setupEntry`／遅延チャネル起動でも利用可能な、セットアップ時のランタイムヘルパー | `createSetupTranslator`、`createPatchedAccountSetupAdapter`、`createEnvPatchedAccountSetupAdapter`、`createSetupInputPresenceValidator`、`noteChannelLookupFailure`、`noteChannelLookupSummary`、`promptResolvedAllowFrom`、`splitSetupEntries`、`createAllowlistSetupWizardProxy`、`createDelegatedSetupWizardProxy` |
| `plugin-sdk/setup-tools`   | セットアップ／インストール用 CLI、アーカイブ、ドキュメントのヘルパー                     | `formatCliCommand`、`detectBinary`、`extractArchive`、`resolveBrewExecutable`、`formatDocsLink`、`CONFIG_DIR`                                                                                                                                                                                                         |

`moveSingleAccountChannelSectionToDefaultAccount(...)` などの設定パッチヘルパーを含む、共有セットアップツールボックス全体が必要な場合は、より広範な `plugin-sdk/setup` の境界を使用してください。

固定されたセットアップウィザードの文言には `createSetupTranslator(...)` を使用してください。これは `OPENCLAW_LOCALE`、`LC_ALL`、`LC_MESSAGES`、`LANG` の順に、空白ではない最初の値を使用し、その後英語にフォールバックします。明示的な英語の上書きには `OPENCLAW_LOCALE=en` を設定してください。Plugin 固有のセットアップテキストは Plugin 所有のコードに保持し、共有カタログキーは共通のセットアップラベル、ステータステキスト、および公式のバンドル Plugin のセットアップ文言にのみ使用してください。

セットアップパッチアダプターは、インポート時にもホットパスで安全なままです。バンドルされた単一アカウント昇格の契約サーフェス検索は遅延実行されるため、`plugin-sdk/setup-runtime` をインポートしても、アダプターが実際に使用される前にバンドル契約サーフェスの検出が先行して読み込まれることはありません。

### チャネル所有のセットアップ入力フィールド

`ChannelSetupInput` は、セットアップ呼び出し元とチャネル
Plugin が共有する汎用エンベロープです。恒久的に型付けされるフィールドは `name`、`token`、`tokenFile`、
`useEnv`、`allowFrom`、`defaultTo` です。Plugin 所有の追加キーも
ランタイム入力オブジェクトに含めることができますが、共有型では
インデックスシグネチャを宣言しません。各 Plugin は、独自のセットアップフィールドを宣言して絞り込むか、
アダプター境界で Plugin 所有のスキーマを使用して検証する必要があります。

```typescript
import type { ChannelSetupAdapter, ChannelSetupInput } from "openclaw/plugin-sdk/channel-setup";

type AcmeSetupInput = ChannelSetupInput & {
  workspaceId?: string;
  webhookUrl?: string;
};

export const acmeSetupAdapter: ChannelSetupAdapter = {
  applyAccountConfig: ({ cfg, input }) => {
    const setupInput = input as AcmeSetupInput;
    return {
      ...cfg,
      channels: {
        ...cfg.channels,
        acme: {
          token: setupInput.token,
          workspaceId: setupInput.workspaceId,
          webhookUrl: setupInput.webhookUrl,
        },
      },
    };
  },
};
```

以前はチャンネル固有のフィールドが
`ChannelSetupInput` に直接宣言されていましたが、外部ソースとの互換性のため、一時的に型定義が維持されています。
これらは非推奨です。2026-07-22 に公開済みのツリー外
チャンネルプラグイン 426 件をレジストリで調査した結果、読み取り元のない 21 個のフィールドを削除し、既知の
読み取り元がある 22 個を維持しました。維持された各フィールドは、公開済みプラグインから読み取られなくなり次第削除されます。
バージョン境界は必要ありません。新規および同梱プラグインは、この
階層に依存してはなりません。所有するフィールドをローカルで宣言してください。

### チャンネル所有の単一アカウント昇格

チャンネルが単一アカウント用のトップレベル設定から `channels.<id>.accounts.*` にアップグレードされると、デフォルトの共有動作により、昇格対象のアカウントスコープ値が `accounts.default` に移動します。

各チャンネルプラグインは、そのセットアップアダプターを通じて、この昇格を拡張または限定できます。

- `singleAccountKeysToMove`: 昇格対象のアカウントに移動する追加のトップレベルキー
- `namedAccountPromotionKeys`: 名前付きアカウントがすでに存在する場合、これらのキーのみを昇格対象のアカウントに移動します。共有ポリシー／配信キーはチャンネルルートに残ります
- `resolveSingleAccountPromotionTarget(...)`: 昇格対象の値を受け取る既存アカウントを選択します

`singleAccountKeysToMove` が存在すると、昇格コントラクトが完全であることを示します。従来のキー昇格を無効にする場合でも、空の配列としてこのフィールドを宣言してください。このフィールドを省略したアダプターでは、すでに公開されているプラグイン向けに、読み取り元に裏付けられた宣言前昇格階層が維持されます。2026-07-22 のレジストリ調査では、公開済みの依存先がない 23 個のキーを削除し、6 個の共通キーとセットアップ専用の `rooms` キーを維持しました。維持された各キーは、公開済みの読み取り元が宣言へ移行し次第削除されます。バージョン境界は必要ありません。

doctor が軽量の同梱セットアップ成果物からこれらの宣言を読み込む必要がある場合は、プラグインパッケージのマニフェストで `openclaw.setupFeatures.configPromotion: true` を宣言してください。セットアップ専用のプラグインサーフェスと完全なチャンネルプラグインは、同じ宣言を公開する必要があります。

解決済みのプラグインを指定して `moveSingleAccountChannelSectionToDefaultAccount(...)` を呼び出す場合は、そのセットアップアダプターを `setupSurface` として渡してください。呼び出し元から渡されたセットアップサーフェスは、読み込み済みおよび同梱の検索結果より優先されるため、スコープ付きまたはセットアップ専用のプラグインをグローバル登録から独立させられます。

<Note>
Matrix は現在の同梱例です。名前付き Matrix アカウントがちょうど 1 つすでに存在する場合、または `defaultAccount` が `Ops` のような既存の非正規キーを指している場合、昇格では新しい `accounts.default` エントリを作成せず、そのアカウントを維持します。
</Note>

## 設定スキーマ

プラグイン設定は、マニフェスト内の JSON Schema に対して検証されます。ユーザーは次のようにプラグインを設定します。

```json5
{
  plugins: {
    entries: {
      "my-plugin": {
        config: {
          webhookSecret: "abc123",
        },
      },
    },
  },
}
```

登録時、プラグインはこの設定を `api.pluginConfig` として受け取ります。

チャンネル固有の設定には、代わりにチャンネル設定セクションを使用します。

```json5
{
  channels: {
    "my-channel": {
      token: "bot-token",
      allowFrom: ["user1", "user2"],
    },
  },
}
```

### チャンネル設定スキーマの構築

`buildChannelConfigSchema` を使用して、Zod スキーマをプラグイン所有の設定成果物で使用される `ChannelConfigSchema` ラッパーに変換します。

```typescript
import { z } from "zod";
import { buildChannelConfigSchema } from "openclaw/plugin-sdk/channel-config-schema";

const accountSchema = z.object({
  token: z.string().optional(),
  allowFrom: z.array(z.string()).optional(),
  accounts: z.object({}).catchall(z.any()).optional(),
  defaultAccount: z.string().optional(),
});

const configSchema = buildChannelConfigSchema(accountSchema);
```

コントラクトをすでに JSON Schema または TypeBox で記述している場合は、直接ヘルパーを使用すると、OpenClaw はメタデータパスで Zod から JSON Schema への変換を省略できます。

```typescript
import { Type } from "typebox";
import { buildJsonChannelConfigSchema } from "openclaw/plugin-sdk/channel-config-schema";

const configSchema = buildJsonChannelConfigSchema(
  Type.Object({
    token: Type.Optional(Type.String()),
    allowFrom: Type.Optional(Type.Array(Type.String())),
  }),
);
```

サードパーティ製プラグインでも、コールドパスのコントラクトは引き続きプラグインマニフェストです。生成された JSON Schema を `openclaw.plugin.json#channelConfigs` に反映し、設定スキーマ、セットアップ、UI サーフェスがランタイムコードを読み込まずに `channels.<id>` を検査できるようにしてください。

## セットアップウィザード

チャンネルプラグインは、`openclaw onboard` 用の対話型セットアップウィザードを提供できます。ウィザードは `ChannelPlugin` 上の `ChannelSetupWizard` オブジェクトです。

```typescript
import type { ChannelSetupWizard } from "openclaw/plugin-sdk/channel-setup";

const setupWizard: ChannelSetupWizard = {
  channel: "my-channel",
  status: {
    configuredLabel: "Connected",
    unconfiguredLabel: "Not configured",
    resolveConfigured: ({ cfg }) => Boolean((cfg.channels as any)?.["my-channel"]?.token),
  },
  credentials: [
    {
      inputKey: "token",
      providerHint: "my-channel",
      credentialLabel: "Bot token",
      preferredEnvVar: "MY_CHANNEL_BOT_TOKEN",
      envPrompt: "Use MY_CHANNEL_BOT_TOKEN from environment?",
      keepPrompt: "Keep current token?",
      inputPrompt: "Enter your bot token:",
      inspect: ({ cfg, accountId }) => {
        const token = (cfg.channels as any)?.["my-channel"]?.token;
        return {
          accountConfigured: Boolean(token),
          hasConfiguredValue: Boolean(token),
        };
      },
    },
  ],
};
```

`ChannelSetupWizard` は、`textInputs`、`dmPolicy`、`allowFrom`、`groupAccess`、`prepare`、`finalize` などもサポートします。完全な同梱例については、Discord プラグインの `src/setup-core.ts` を参照してください。

<AccordionGroup>
  <Accordion title="共有 allowFrom プロンプト">
    標準の `note -> prompt -> parse -> merge -> patch` フローだけを必要とする DM 許可リストプロンプトには、`openclaw/plugin-sdk/setup` の共有セットアップヘルパーである `createPromptParsedAllowFromForAccount(...)` と `createTopLevelChannelParsedAllowFromPrompt(...)` を優先して使用してください。
  </Accordion>
  <Accordion title="標準のチャンネルセットアップ状態">
    ラベル、スコア、任意の追加行だけが異なるチャンネルセットアップ状態ブロックには、各プラグインで同じ `status` オブジェクトを独自実装する代わりに、`openclaw/plugin-sdk/setup` の `createStandardChannelSetupStatus(...)` を優先して使用してください。
  </Accordion>
  <Accordion title="任意のチャンネルセットアップサーフェス">
    特定のコンテキストでのみ表示する任意のセットアップサーフェスには、`openclaw/plugin-sdk/channel-setup` の `createOptionalChannelSetupSurface` を使用します。

    ```typescript
    import { createOptionalChannelSetupSurface } from "openclaw/plugin-sdk/channel-setup";

    const setupSurface = createOptionalChannelSetupSurface({
      channel: "my-channel",
      label: "My Channel",
      npmSpec: "@myorg/openclaw-my-channel",
      docsPath: "/channels/my-channel",
    });
    // Returns { setupAdapter, setupWizard }
    ```

    任意インストールサーフェスの片方だけが必要な場合、`plugin-sdk/channel-setup` は低レベルの `createOptionalChannelSetupAdapter(...)` および `createOptionalChannelSetupWizard(...)` ビルダーも公開しています。

    生成された任意アダプター／ウィザードは、実際の設定書き込みに対してフェイルクローズします。`validateInput`、`applyAccountConfig`、`finalize` で同じインストール必須メッセージを再利用し、`docsPath` が設定されている場合はドキュメントへのリンクを追加します。

  </Accordion>
  <Accordion title="バイナリベースのセットアップヘルパー">
    バイナリベースのセットアップ UI では、同じバイナリ／状態連携を各チャンネルにコピーする代わりに、共有委譲ヘルパーを優先して使用してください。

    - `createDetectedBinaryStatus(...)`: ラベル、ヒント、スコア、バイナリ検出だけが異なる状態ブロック向け
    - `createCliPathTextInput(...)`: パスベースのテキスト入力向け
    - `createDelegatedSetupWizardProxy(...)`: `setupEntry` が状態、準備、または完了処理を、より重い完全版ウィザードへ遅延委譲する必要がある場合
    - `createDelegatedTextInputShouldPrompt(...)`: `setupEntry` が `textInputs[*].shouldPrompt` の判断だけを委譲する必要がある場合

  </Accordion>
</AccordionGroup>

## 公開とインストール

**外部プラグイン:** [ClawHub](/clawhub) に公開してから、次のようにインストールします。

<Tabs>
  <Tab title="npm">
    ```bash
    openclaw plugins install @myorg/openclaw-my-plugin
    ```

    ベアパッケージ指定は、起動移行中に npm からインストールされます。ただし、名前が同梱または公式のプラグイン ID と一致する場合、OpenClaw は代わりにそのローカル／公式コピーを使用します。決定的なソース選択には `clawhub:`、`npm:`、`git:`、または `npm-pack:` を使用してください。詳細は[プラグインの管理](/ja-JP/plugins/manage-plugins)を参照してください。

  </Tab>
  <Tab title="ClawHub のみ">
    ```bash
    openclaw plugins install clawhub:@myorg/openclaw-my-plugin
    ```
  </Tab>
  <Tab title="npm パッケージ指定">
    パッケージがまだ ClawHub に移行していない場合、または移行中に
    npm の直接インストールパスが必要な場合は、npm を使用します。

    ```bash
    openclaw plugins install npm:@myorg/openclaw-my-plugin
    ```

  </Tab>
</Tabs>

**リポジトリ内プラグイン:** 同梱プラグインのワークスペースツリー配下に配置します。ビルド時に自動的に検出されます。

<Info>
npm ソースのインストールでは、`openclaw plugins install` が `~/.openclaw/npm/projects` 配下のプラグインごとのプロジェクトにパッケージをインストールし、ライフサイクルスクリプトを無効化します（`--ignore-scripts`）。プラグインの依存関係ツリーは純粋な JS/TS に保ち、`postinstall` ビルドを必要とするパッケージは避けてください。
</Info>

<Note>
Gateway の起動時にプラグインの依存関係はインストールされません。npm／git／ClawHub のインストールフローが依存関係の収束を担います。ローカルプラグインでは、依存関係がすでにインストールされている必要があります。
</Note>

同梱パッケージのメタデータは明示的であり、Gateway の起動時にビルド済み JavaScript から推論されるものではありません。ランタイム依存関係は、それを所有するプラグインパッケージに属します。パッケージ化された OpenClaw の起動処理が、プラグインの依存関係を修復またはミラーリングすることはありません。

## 関連項目

- [プラグインの構築](/ja-JP/plugins/building-plugins) — ステップごとの「はじめに」ガイド
- [プラグインマニフェスト](/ja-JP/plugins/manifest) — 完全なマニフェストスキーマのリファレンス
- [SDK エントリーポイント](/ja-JP/plugins/sdk-entrypoints) — `definePluginEntry` と `defineChannelPluginEntry`
