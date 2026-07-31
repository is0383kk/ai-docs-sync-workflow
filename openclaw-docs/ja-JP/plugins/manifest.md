---
read_when:
    - OpenClaw Pluginを構築しています
    - プラグイン設定スキーマをリリースするか、プラグインの検証エラーをデバッグする必要がある場合
summary: Plugin マニフェスト + JSON スキーマ要件（厳格な設定検証）
title: Plugin マニフェスト
x-i18n:
    generated_at: "2026-07-26T09:10:25Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 244e5c8265ff79b0ff6e8f4b60c9635cccc3ba66093cecab458676beb9578264
    source_path: plugins/manifest.md
    workflow: 16
---

このページでは、**ネイティブ OpenClaw plugin マニフェスト**である `openclaw.plugin.json` について説明します。互換性のあるバンドルレイアウト（Codex、Claude、Cursor）については、[Plugin バンドル](/ja-JP/plugins/bundles)を参照してください。

互換性のあるバンドル形式では、代わりに独自のマニフェストファイルを使用します。

- Codex バンドル: `.codex-plugin/plugin.json`
- Claude バンドル: `.claude-plugin/plugin.json`、またはマニフェストを使用しないデフォルトの Claude コンポーネントレイアウト
- Cursor バンドル: `.cursor-plugin/plugin.json`

OpenClaw はこれらのレイアウトを自動検出しますが、以下の `openclaw.plugin.json` スキーマに対する検証は行いません。互換性のあるバンドルでは、レイアウトが OpenClaw のランタイム要件に一致する場合、OpenClaw はバンドルメタデータ、宣言された skill ルート、Claude コマンドルート、Claude の `settings.json` デフォルト、Claude LSP デフォルト、およびサポートされているフックパックを読み取ります。

すべてのネイティブ OpenClaw plugin は、**plugin ルート**に `openclaw.plugin.json` を**必ず**含める必要があります。OpenClaw は、**plugin コードを実行せずに**設定を検証するためにこれを読み取ります。マニフェストが存在しないか無効な場合、設定の検証はブロックされ、plugin エラーとして扱われます。

plugin システムの完全なガイドについては[Plugin](/ja-JP/tools/plugin)を、ネイティブのケイパビリティモデルと現在の外部互換性に関するガイダンスについては[ケイパビリティモデル](/ja-JP/plugins/architecture#public-capability-model)を参照してください。

## このファイルの役割

`openclaw.plugin.json` は、OpenClaw が**plugin コードを読み込む前に**読み取るメタデータです。ここに含まれるすべての情報は、plugin ランタイムを起動せずに確認できるほど軽量である必要があります。

**用途:**

- plugin の識別、設定の検証、設定 UI のヒント
- 認証、オンボーディング、セットアップのメタデータ（エイリアス、自動有効化、プロバイダー環境変数、認証方式の選択肢）
- コントロールプレーン画面向けの有効化ヒント
- モデルファミリー所有権の短縮表記
- 静的なケイパビリティ所有権のスナップショット（`contracts`）
- ダッシュボードウィジェットのデータバインディングとアクション動詞
- plugin が有効な間に存在すべき静的 MCP サーバー
- 共有 `openclaw qa` ホストが確認できる QA ランナーのメタデータ
- カタログおよび検証画面にマージされる、チャンネル固有の設定メタデータ

**用途に含まれないもの:** ネイティブランタイムフックの登録、plugin コードのエントリポイントの宣言、npm インストールメタデータ。これらは plugin コードおよび `package.json` に含めます。

## 最小構成の例

```json
{
  "id": "voice-call",
  "configSchema": {
    "type": "object",
    "additionalProperties": false,
    "properties": {}
  }
}
```

## 詳細な例

```json
{
  "id": "openrouter",
  "name": "OpenRouter",
  "description": "OpenRouter プロバイダー plugin",
  "version": "1.0.0",
  "providers": ["openrouter"],
  "modelSupport": {
    "modelPrefixes": ["router-"]
  },
  "modelIdNormalization": {
    "providers": {
      "openrouter": {
        "prefixWhenBare": "openrouter"
      }
    }
  },
  "providerEndpoints": [
    {
      "endpointClass": "openrouter",
      "hostSuffixes": ["openrouter.ai"]
    }
  ],
  "providerRequest": {
    "providers": {
      "openrouter": {
        "family": "openrouter"
      }
    }
  },
  "cliBackends": ["openrouter-cli"],
  "syntheticAuthRefs": ["openrouter-cli"],
  "setup": {
    "providers": [
      {
        "id": "openrouter",
        "envVars": ["OPENROUTER_API_KEY"]
      }
    ]
  },
  "providerAuthAliases": {
    "openrouter-coding": "openrouter"
  },
  "providerAuthChoices": [
    {
      "provider": "openrouter",
      "method": "api-key",
      "choiceId": "openrouter-api-key",
      "choiceLabel": "OpenRouter API キー",
      "groupId": "openrouter",
      "groupLabel": "OpenRouter",
      "optionKey": "openrouterApiKey",
      "cliFlag": "--openrouter-api-key",
      "cliOption": "--openrouter-api-key <key>",
      "cliDescription": "OpenRouter API キー",
      "onboardingScopes": ["text-inference"]
    }
  ],
  "uiHints": {
    "apiKey": {
      "label": "API キー",
      "placeholder": "sk-or-v1-...",
      "sensitive": true
    }
  },
  "configSchema": {
    "type": "object",
    "additionalProperties": false,
    "properties": {
      "apiKey": {
        "type": "string"
      }
    }
  }
}
```

## トップレベルフィールドのリファレンス

| フィールド                           | 必須 | 型                           | 意味                                                                                                                                                                                                                                                                                           |
| ------------------------------------ | -------- | ---------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `id`                                 | はい      | `string`                     | 正規の Plugin ID。`plugins.entries.<id>` で使用される ID です。                                                                                                                                                                                                                            |
| `configSchema`                       | はい      | `object`                     | この Plugin の設定用インライン JSON Schema。                                                                                                                                                                                                                                                   |
| `requiresPlugins`                    | いいえ       | `string[]`                   | この Plugin が機能するために、併せてインストールする必要がある Plugin ID。検出時、Plugin は読み込み可能なまま維持されますが、必須 Plugin が不足している場合は警告されます。                                                                                                                                   |
| `enabledByDefault`                   | いいえ       | `true`                       | バンドルされた Plugin をデフォルトで有効として指定します。Plugin をデフォルトで無効のままにするには、省略するか、`true` 以外の値を設定します。                                                                                                                                                                   |
| `enabledByDefaultOnPlatforms`        | いいえ       | `string[]`                   | バンドルされた Plugin を、一覧にある Node.js プラットフォーム上でのみデフォルトで有効として指定します（例: `["darwin"]`）。明示的な設定が常に優先されます。                                                                                                                                                       |
| `legacyPluginIds`                    | いいえ       | `string[]`                   | この正規 Plugin ID に正規化されるレガシー ID。                                                                                                                                                                                                                                         |
| `autoEnableWhenConfiguredProviders`  | いいえ       | `string[]`                   | 認証、設定、またはモデル参照で言及された場合に、この Plugin を自動的に有効化するプロバイダー ID。                                                                                                                                                                                                |
| `kind`                               | いいえ       | `PluginKind \| PluginKind[]` | `plugins.slots.*` で使用される、1 つ以上の排他的な Plugin 種別（`"memory"`、`"context-engine"`）を宣言します。両方のスロットを所有する Plugin は、両方の種別を 1 つの配列で宣言します。                                                                                                                        |
| `channels`                           | いいえ       | `string[]`                   | この Plugin が所有するチャンネル ID。検出と設定の検証に使用されます。                                                                                                                                                                                                                    |
| `providers`                          | いいえ       | `string[]`                   | この Plugin が所有するプロバイダー ID。                                                                                                                                                                                                                                                             |
| `providerCatalogEntry`               | いいえ       | `string`                     | Plugin ルートからの相対パスで指定する軽量なプロバイダーカタログモジュールのパス。Plugin ランタイム全体を有効化せずに読み込める、マニフェストスコープのプロバイダーカタログメタデータに使用します。                                                                                                            |
| `modelSupport`                       | いいえ       | `object`                     | ランタイムの前に Plugin を自動読み込みするために使用される、マニフェスト所有のモデルファミリー略記メタデータ。                                                                                                                                                                                                    |
| `modelCatalog`                       | いいえ       | `object`                     | この Plugin が所有するプロバイダー用の宣言的なモデルカタログメタデータ。Plugin ランタイムを読み込まずに行う、将来の読み取り専用一覧表示、オンボーディング、モデル選択、エイリアス、抑制のためのコントロールプレーン契約です。                                                                    |
| `modelPricing`                       | いいえ       | `object`                     | プロバイダー所有の外部料金検索ポリシー。ローカルまたはセルフホスト型プロバイダーをリモート料金カタログの対象外にしたり、コアにプロバイダー ID をハードコードせずにプロバイダー参照を OpenRouter/LiteLLM カタログ ID にマッピングしたりするために使用します。                                                                        |
| `modelIdNormalization`               | いいえ       | `object`                     | プロバイダーランタイムの読み込み前に実行する必要がある、プロバイダー所有のモデル ID エイリアス／プレフィックスのクリーンアップ。                                                                                                                                                                                                      |
| `providerEndpoints`                  | いいえ       | `object[]`                   | プロバイダーランタイムの読み込み前にコアが分類する必要があるプロバイダールート向けの、マニフェスト所有のエンドポイントホスト／baseUrl メタデータ。                                                                                                                                                                       |
| `providerRequest`                    | いいえ       | `object`                     | プロバイダーランタイムの読み込み前に汎用リクエストポリシーが使用する、軽量なプロバイダーファミリーおよびリクエスト互換性メタデータ。                                                                                                                                                                         |
| `secretProviderIntegrations`         | いいえ       | `Record<string, object>`     | コアにプロバイダー固有の統合をハードコードすることなく、セットアップまたはインストール画面で提供できる宣言的な SecretRef exec プロバイダープリセット。                                                                                                                                                |
| `cliBackends`                        | いいえ       | `string[]`                   | この Plugin が所有する CLI 推論バックエンド ID。明示的な設定参照に基づく起動時の自動有効化に使用されます。                                                                                                                                                                                    |
| `syntheticAuthRefs`                  | いいえ       | `string[]`                   | ランタイムの読み込み前に行われるコールドモデル検出中に、Plugin 所有の合成認証フックを調査する対象となるプロバイダーまたは CLI バックエンド参照。                                                                                                                                                         |
| `nonSecretAuthMarkers`               | いいえ       | `string[]`                   | 非シークレットのローカル、OAuth、または環境由来の認証情報状態を表す、バンドル Plugin 所有のプレースホルダー API キー値。                                                                                                                                                                           |
| `commandAliases`                     | いいえ       | `object[]`                   | ランタイムの読み込み前に Plugin 対応の設定および CLI 診断を生成する、この Plugin が所有するコマンド名。                                                                                                                                                                           |
| `providerUsageAuthEnvVars`           | いいえ       | `Record<string, string[]>`   | 使用量／請求専用のプロバイダー認証情報。OpenClaw はこれらの名前を使用量検出とシークレットの除去に使用しますが、推論認証には決して使用しません。                                                                                                                                                      |
| `providerAuthAliases`                | いいえ       | `Record<string, string>`     | 認証検索時に別のプロバイダー ID を再利用するプロバイダー ID。たとえば、基本プロバイダーの API キーと認証プロファイルを共有するコーディングプロバイダーが該当します。                                                                                                                                     |
| `providerAuthChoices`                | いいえ       | `object[]`                   | オンボーディングの選択画面、優先プロバイダーの解決、単純な CLI フラグの配線に使用される軽量な認証選択メタデータ。                                                                                                                                                                                  |
| `activation`                         | いいえ       | `object`                     | 起動、プロバイダー、コマンド、チャンネル、ルート、ケイパビリティをトリガーとする読み込み用の軽量な有効化プランナーメタデータ。メタデータのみであり、実際の動作は引き続き Plugin ランタイムが所有します。                                                                                                                  |
| `setup`                              | いいえ       | `object`                     | Plugin ランタイムを読み込まずに検出およびセットアップ画面から参照できる、軽量なセットアップ／オンボーディング記述子。                                                                                                                                                                               |
| `qaRunners`                          | いいえ       | `object[]`                   | Plugin ランタイムの読み込み前に共有 `openclaw qa` ホストが使用する、軽量な QA ランナー記述子。                                                                                                                                                                                                 |
| `dashboard`                          | いいえ       | `object`                     | ダッシュボードウィジェットのデータバインディングとアクション動詞。各エントリは、この Plugin が登録した Gateway メソッドに対して、必要な読み取りまたは書き込みスコープを使用して検証されます。[ダッシュボードリファレンス](#dashboard-reference)を参照してください。                                                                            |
| `mcpServers`                         | いいえ       | `Record<string, object>`     | この Plugin が有効な間に提供される静的 MCP サーバー定義。相対コマンド引数と作業ディレクトリは、Plugin のルートを基準に解決されます。オペレーターの `mcp.servers` エントリは、同名の定義を上書きまたは無効化します。[MCP サーバーリファレンス](#mcp-server-reference)を参照してください。 |
| `contracts`                          | いいえ       | `object`                     | 外部認証フック、埋め込み、音声、リアルタイム文字起こし、リアルタイム音声、メディア理解、画像／動画／音楽生成、Web フェッチ、Web 検索、ワーカープロバイダー、ドキュメント／Web コンテンツ抽出、ツール所有権に関する静的な機能所有権スナップショット。                     |
| `configContracts`                    | いいえ       | `object`                     | 汎用コアヘルパーが使用する、マニフェスト所有の設定動作：危険なフラグの検出、SecretRef の移行先、レガシー設定パスの絞り込み。[configContracts リファレンス](#configcontracts-reference)を参照してください。                                                                         |
| `mediaUnderstandingProviderMetadata` | いいえ       | `Record<string, object>`     | `contracts.mediaUnderstandingProviders` で宣言されたプロバイダー ID 向けの低コストなメディア理解のデフォルト。                                                                                                                                                                                       |
| `imageGenerationProviderMetadata`    | いいえ       | `Record<string, object>`     | `contracts.imageGenerationProviders` で宣言されたプロバイダー ID 向けの低コストな画像生成認証メタデータ。プロバイダー所有の認証エイリアスとベース URL ガードを含みます。                                                                                                                             |
| `videoGenerationProviderMetadata`    | いいえ       | `Record<string, object>`     | `contracts.videoGenerationProviders` で宣言されたプロバイダー ID 向けの低コストな動画生成認証メタデータ。プロバイダー所有の認証エイリアスとベース URL ガードを含みます。                                                                                                                             |
| `musicGenerationProviderMetadata`    | いいえ       | `Record<string, object>`     | `contracts.musicGenerationProviders` で宣言されたプロバイダー ID 向けの低コストな音楽生成認証メタデータ。プロバイダー所有の認証エイリアスとベース URL ガードを含みます。                                                                                                                             |
| `toolMetadata`                       | いいえ       | `Record<string, object>`     | `contracts.tools` で宣言された Plugin 所有ツール向けの低コストな可用性メタデータ。設定、環境変数、または認証の証拠が存在しない限りツールがランタイムを読み込むべきでない場合に使用します。                                                                                                                      |
| `channelConfigs`                     | いいえ       | `Record<string, object>`     | ランタイムの読み込み前に、検出および検証サーフェスへ統合される、マニフェスト所有のチャンネル設定メタデータ。                                                                                                                                                                                     |
| `skills`                             | いいえ       | `string[]`                   | Plugin のルートを基準とした、読み込む Skills ディレクトリ。                                                                                                                                                                                                                                        |
| `name`                               | いいえ       | `string`                     | 人間が判読できる Plugin 名。                                                                                                                                                                                                                                                                    |
| `description`                        | いいえ       | `string`                     | Plugin の各サーフェスに表示される短い概要。                                                                                                                                                                                                                                                        |
| `catalog`                            | いいえ       | `object`                     | Plugin カタログのサーフェス向けの任意の表示ヒント。このメタデータによって Plugin がインストールまたは有効化されたり、Plugin に信頼が付与されたりすることはありません。                                                                                                                                                                   |
| `icon`                               | いいえ       | `string`                     | マーケットプレイス／カタログカード向けの HTTPS 画像 URL。ClawHub は有効な `https://` URL をすべて受け入れ、省略されているか無効な場合はデフォルトの Plugin アイコンを使用します。                                                                                                                             |
| `version`                            | いいえ       | `string`                     | 情報提供用の Plugin バージョン。                                                                                                                                                                                                                                                                  |
| `uiHints`                            | いいえ       | `Record<string, object>`     | 設定フィールドの UI ラベル、プレースホルダー、機密性に関するヒント。                                                                                                                                                                                                                              |

## MCP サーバーリファレンス

`mcpServers` を使用すると、ネイティブ Plugin は MCP App を含む MCP サーバーを同梱でき、運用者がその静的なプロセス定義を `openclaw.json` に重複して記述する必要がなくなります。

```json
{
  "mcpServers": {
    "example": {
      "transport": "stdio",
      "command": "node",
      "args": ["./mcp-server.js"]
    }
  }
}
```

OpenClaw は、所有元の Plugin が有効な間のみ、これらのサーバーを含めます。相対 `command`、`args`、`cwd`、および `workingDirectory` パスは、Plugin のルートを基準に解決されます。ユーザー設定が引き続き優先されます。`mcp.servers.<name>` で Plugin のデフォルトを置き換えるか、`enabled: false` を設定して除外できます。MCP App のレンダリングとサーバーツールの呼び出しには、引き続き通常の MCP Apps 設定と有効なツールポリシーが必要です。サーバーを宣言しても、どちらの境界も回避されません。

## ダッシュボードリファレンス

`dashboard` を使用すると、有効な Plugin は、コアに Plugin ポリシーを追加することなく、権限を付与されたダッシュボードウィジェットへ既存の Gateway RPC を公開できます。データバインディングには、同じ Plugin が `operator.read` で登録するメソッドを指定する必要があります。アクション動詞には、同じ Plugin が `operator.write` で登録するメソッドを指定する必要があります。不一致がある場合、登録時に Plugin が拒否されます。

```json
{
  "dashboard": {
    "dataBindings": [
      {
        "id": "items.list",
        "method": "example.items.list",
        "description": "サンプル項目を一覧表示します。"
      }
    ],
    "actionVerbs": [
      {
        "id": "refresh",
        "method": "example.items.refresh",
        "description": "サンプル項目を更新します。",
        "paramShape": {
          "type": "object",
          "additionalProperties": false,
          "properties": {
            "force": { "type": "boolean" }
          }
        }
      }
    ]
  }
}
```

マニフェスト ID は Plugin ローカルです。ウィジェットの権限付与では、`example.items.list` や `example.refresh` などの `<plugin-id>.<id>` を使用します。永続化された権限付与の名前空間を曖昧さなく保つため、OpenClaw は Plugin ID セグメント内の `%` と `.` を、それぞれ `%25` と `%2E` としてエスケープします。通常の Plugin ID は自然な形式のままです。`paramShape` は任意の JSON Schema であり、OpenClaw が Plugin RPC を呼び出す前にアクションパラメータオブジェクトへ適用されます。

## カタログリファレンス

`catalog` は、Plugin ブラウザーに任意の表示ヒントを提供します。ホストはこれらのヒントを無視できます。これらによって Plugin がインストールまたは有効化されることはなく、ランタイム動作や信頼レベルも変更されません。

```json
{
  "catalog": {
    "featured": true,
    "order": 10
  }
}
```

| フィールド      | 型      | 意味                                                              |
| ---------- | --------- | -------------------------------------------------------------------------- |
| `featured` | `boolean` | カタログ画面でこの Plugin を特集対象にするかどうか。                       |
| `order`    | `number`  | 選定された Plugin 間での昇順の表示ヒント。値が小さいほど先に表示されます。 |

## 生成プロバイダーメタデータリファレンス

生成プロバイダーメタデータフィールドは、対応する `contracts.*GenerationProviders` リストで宣言されたプロバイダーの静的認証シグナルを記述します。OpenClaw は、プロバイダーのランタイムが読み込まれる前にこれらのフィールドを読み取るため、コアツールはすべてのプロバイダー Plugin をインポートせずに、生成プロバイダーが利用可能かどうかを判断できます。

これらのフィールドは、低コストな宣言的情報にのみ使用してください。トランスポート、リクエスト変換、トークン更新、認証情報の検証、および実際の生成動作は、Plugin ランタイムに残します。

```json
{
  "contracts": {
    "imageGenerationProviders": ["example-image"]
  },
  "imageGenerationProviderMetadata": {
    "example-image": {
      "aliases": ["example-image-oauth"],
      "authProviders": ["example-image"],
      "configSignals": [
        {
          "rootPath": "plugins.entries.example-image.config",
          "overlayPath": "image",
          "mode": {
            "path": "mode",
            "default": "local",
            "allowed": ["local"]
          },
          "requiredAny": ["workflow", "workflowPath"],
          "required": ["promptNodeId"]
        }
      ],
      "authSignals": [
        {
          "provider": "example-image"
        },
        {
          "provider": "example-image-oauth",
          "providerBaseUrl": {
            "provider": "example-image",
            "defaultBaseUrl": "https://api.example.com/v1",
            "allowedBaseUrls": ["https://api.example.com/v1"]
          }
        }
      ]
    }
  }
}
```

各メタデータエントリでサポートされるフィールドは次のとおりです。

| フィールド                  | 必須 | 型       | 意味                                                                                                                                       |
| ---------------------- | -------- | ---------- | --------------------------------------------------------------------------------------------------------------------------------------------------- |
| `aliases`              | いいえ       | `string[]` | 生成プロバイダーの静的認証エイリアスとして扱う追加のプロバイダー ID。                                                       |
| `authProviders`        | いいえ       | `string[]` | 設定済みの認証プロファイルを、この生成プロバイダーの認証として扱うプロバイダー ID。                                                      |
| `configSignals`        | いいえ       | `object[]` | 認証プロファイルや環境変数なしで設定できる、ローカルまたはセルフホスト型プロバイダー向けの低コストな設定限定の可用性シグナル。                 |
| `authSignals`          | いいえ       | `object[]` | 明示的な認証シグナル。指定した場合、プロバイダー ID、`aliases`、および `authProviders` から得られるデフォルトのシグナルセットを置き換えます。                     |
| `referenceAudioInputs` | いいえ       | `boolean`  | 動画生成専用。プロバイダーが参照音声アセットを受け入れる場合は `true` に設定します。それ以外の場合、`video_generate` は音声参照パラメータを非表示にします。 |

各 `configSignals` エントリでサポートされるフィールドは次のとおりです。

| フィールド            | 必須 | 型       | 意味                                                                                                                                                                             |
| ---------------- | -------- | ---------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `rootPath`       | はい      | `string`   | 検査する Plugin 所有の設定オブジェクトへのドットパス。例: `plugins.entries.example.config`。                                                                                      |
| `overlayPath`    | いいえ       | `string`   | シグナルを評価する前に、そのオブジェクトをルートオブジェクトへオーバーレイする、ルート設定内のドットパス。`image`、`video`、`music` などの機能固有の設定に使用します。   |
| `overlayMapPath` | いいえ       | `string`   | その各オブジェクト値をルートオブジェクトへオーバーレイする、ルート設定内のドットパス。設定済みの任意のアカウントを適格とする `accounts` などの名前付きアカウントマップに使用します。 |
| `required`       | いいえ       | `string[]` | 有効な設定内で、設定済みの値を持つ必要があるドットパス。文字列は空であってはならず、オブジェクトと配列も空であってはなりません。                                                  |
| `requiredAny`    | いいえ       | `string[]` | 有効な設定内で、少なくとも 1 つが設定済みの値を持つ必要があるドットパス。                                                                                                    |
| `mode`           | いいえ       | `object`   | 有効な設定内にある任意の文字列モードガード。設定限定の可用性が 1 つのモードにのみ適用される場合に使用します。                                                                  |

各 `mode` ガードでサポートされるフィールドは次のとおりです。

| フィールド        | 必須 | 型       | 意味                                                                      |
| ------------ | -------- | ---------- | ---------------------------------------------------------------------------------- |
| `path`       | いいえ       | `string`   | 有効な設定内のドットパス。デフォルトは `mode` です。                          |
| `default`    | いいえ       | `string`   | 設定でパスが省略されている場合に使用するモード値。                                  |
| `allowed`    | いいえ       | `string[]` | 指定した場合、有効なモードがこれらの値のいずれかである場合にのみ、シグナルが成立します。 |
| `disallowed` | いいえ       | `string[]` | 指定した場合、有効なモードがこれらの値のいずれかである場合、シグナルは成立しません。       |

各 `authSignals` エントリでサポートされるフィールドは次のとおりです。

| フィールド             | 必須 | 型     | 意味                                                                                                                                                                 |
| ----------------- | -------- | -------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `provider`        | はい      | `string` | 設定済みの認証プロファイルで確認するプロバイダー ID。                                                                                                                             |
| `providerBaseUrl` | いいえ       | `object` | 参照される設定済みプロバイダーが許可されたベース URL を使用している場合にのみ、シグナルを有効として扱う任意のガード。認証エイリアスが特定の API に対してのみ有効な場合に使用します。 |

各 `providerBaseUrl` ガードでサポートされるフィールドは次のとおりです。

| フィールド             | 必須 | 型       | 意味                                                                                                                                        |
| ----------------- | -------- | ---------- | ---------------------------------------------------------------------------------------------------------------------------------------------------- |
| `provider`        | はい      | `string`   | `baseUrl` を確認するプロバイダー設定 ID。                                                                                                |
| `defaultBaseUrl`  | いいえ       | `string`   | プロバイダー設定で `baseUrl` が省略されている場合に想定するベース URL。                                                                                         |
| `allowedBaseUrls` | はい      | `string[]` | この認証シグナルで許可されるベース URL。設定済みまたはデフォルトのベース URL が、正規化されたこれらの値のいずれとも一致しない場合、シグナルは無視されます。 |

## ツールメタデータリファレンス

`toolMetadata` は、ツール名をキーとして、生成プロバイダーメタデータと同じ `configSignals` および `authSignals` の形式を使用します。`contracts.tools` は所有権を宣言します。`toolMetadata` は低コストな可用性の根拠を宣言するため、OpenClaw は、ツールファクトリーから `null` を返させるためだけに Plugin ランタイムをインポートせずに済みます。

```json
{
  "setup": {
    "providers": [
      {
        "id": "example",
        "envVars": ["EXAMPLE_API_KEY"]
      }
    ]
  },
  "contracts": {
    "tools": ["example_search"]
  },
  "toolMetadata": {
    "example_search": {
      "authSignals": [
        {
          "provider": "example"
        }
      ],
      "configSignals": [
        {
          "rootPath": "plugins.entries.example.config",
          "overlayPath": "search",
          "required": ["apiKey"]
        }
      ]
    }
  }
}
```

`toolMetadata` エントリでは、上記の共通 `configSignals`/`authSignals` フィールドに加えて、`optional`（Plugin の有効化にそのツールが必須ではないことを示す）および `replaySafe`（モデルのターンが未完了に終わった後でも、ツールの実行を安全に繰り返せることを示す）も指定できます。

ツールに `toolMetadata` がない場合、OpenClaw は既存の動作を維持し、ツールのコントラクトがポリシーに一致すると、そのツールを所有する Plugin を読み込みます。ファクトリが認証や設定に依存するホットパスのツールでは、Plugin の作成者は、問い合わせのためにコアからランタイムをインポートさせるのではなく、`toolMetadata` を宣言する必要があります。

## providerAuthChoices リファレンス

各 `providerAuthChoices` エントリは、オンボーディングまたは認証の選択肢を 1 つ記述します。OpenClaw は、プロバイダーのランタイムが読み込まれる前にこれを読み取ります。プロバイダーのセットアップリストでは、プロバイダーのランタイムを読み込まずに、これらのマニフェストの選択肢、ディスクリプターから派生したセットアップの選択肢、およびインストールカタログのメタデータを使用します。

| フィールド                 | 必須 | 型                                                                  | 意味                                                                                             |
| --------------------- | -------- | --------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------- |
| `provider`            | はい      | `string`                                                              | この選択肢が属するプロバイダー ID。                                                                       |
| `method`              | はい      | `string`                                                              | ディスパッチ先の認証方式 ID。                                                                            |
| `choiceId`            | はい      | `string`                                                              | オンボーディングおよび CLI フローで使用される安定した認証選択肢 ID。                                                   |
| `choiceLabel`         | いいえ       | `string`                                                              | ユーザー向けラベル。省略した場合、OpenClaw は `choiceId` にフォールバックします。                                         |
| `choiceHint`          | いいえ       | `string`                                                              | 選択画面用の短い補足テキスト。                                                                         |
| `icon`                | いいえ       | HTTPS URL                                                             | 対応するオンボーディングクライアントで、この選択肢の横に表示されるアートワーク。                                         |
| `website`             | いいえ       | HTTPS URL                                                             | 対応するオンボーディングクライアントに表示される製品、サインイン、またはインストールのページ。                             |
| `assistantPriority`   | いいえ       | `number`                                                              | アシスタント主導の対話型選択画面では、値が小さいほど先に並びます。                                        |
| `assistantVisibility` | いいえ       | `"visible"` \| `"manual-only"`                                        | CLI での手動選択は許可したまま、アシスタントの選択画面ではこの選択肢を非表示にします。                         |
| `deprecatedChoiceIds` | いいえ       | `string[]`                                                            | ユーザーをこの代替選択肢にリダイレクトする必要がある、従来の選択肢 ID。                                  |
| `groupId`             | いいえ       | `string`                                                              | 関連する選択肢をグループ化するための任意のグループ ID。                                                           |
| `groupLabel`          | いいえ       | `string`                                                              | そのグループのユーザー向けラベル。                                                                         |
| `groupHint`           | いいえ       | `string`                                                              | グループ用の短い補足テキスト。                                                                          |
| `onboardingFeatured`  | いいえ       | `boolean`                                                             | 対話型オンボーディング選択画面で、「More...」エントリより前の注目階層にこのグループを表示します。 |
| `optionKey`           | いいえ       | `string`                                                              | 単純な単一フラグ認証フロー用の内部オプションキー。                                                       |
| `cliFlag`             | いいえ       | `string`                                                              | `--openrouter-api-key` などの CLI フラグ名。                                                            |
| `cliOption`           | いいえ       | `string`                                                              | `--openrouter-api-key <key>` などの完全な CLI オプション形式。                                              |
| `cliDescription`      | いいえ       | `string`                                                              | CLI ヘルプで使用される説明。                                                                             |
| `appGuidedSecret`     | いいえ       | `boolean`                                                             | 貼り付けられた 1 つのシークレットとプロバイダーのデフォルト値だけで、アプリに案内されるセットアップには十分です。                              |
| `appGuidedDiscovery`  | いいえ       | `boolean`                                                             | 対応するランタイム認証方式が、`appGuidedSetup` を介した読み取り専用のローカル検出を所有します。                 |
| `appGuidedAuth`       | いいえ       | `"oauth"` \| `"device-code"`                                          | ネイティブのセットアップクライアントが汎用的にレンダリングできる、プロバイダー所有の対話型ログイン。                        |
| `onboardingScopes`    | いいえ       | `Array<"text-inference" \| "image-generation" \| "music-generation">` | この選択肢を表示するオンボーディング画面。省略した場合、デフォルトは `["text-inference"]` です。  |

`appGuidedDiscovery` が true の場合、対応するプロバイダー認証方式は
`appGuidedSetup.detect` および `appGuidedSetup.prepare` を公開する必要があります。検出は
読み取り専用でなければなりません。ログイン、モデルのプル、ダウンロード、設定の書き込みは行いません。準備処理では、
選択された正確なモデルを再確認して設定案を返します。OpenClaw はその
設定案を分離環境でライブテストし、成功した場合にのみコミットします。

## commandAliases リファレンス

Plugin が、ユーザーが誤って `plugins.allow` に指定したり、ルート CLI コマンドとして実行しようとしたりする可能性のあるランタイムコマンド名を所有する場合は、`commandAliases` を使用します。OpenClaw は、Plugin のランタイムコードをインポートせずに、このメタデータを診断に使用します。

```json
{
  "commandAliases": [
    {
      "name": "dreaming",
      "kind": "runtime-slash",
      "cliCommand": "memory"
    }
  ]
}
```

| フィールド        | 必須 | 型              | 意味                                                           |
| ------------ | -------- | ----------------- | ----------------------------------------------------------------------- |
| `name`       | はい      | `string`          | この Plugin に属するコマンド名。                               |
| `kind`       | いいえ       | `"runtime-slash"` | エイリアスを、ルート CLI コマンドではなくチャットのスラッシュコマンドとして示します。 |
| `cliCommand` | いいえ       | `string`          | 存在する場合、CLI 操作用に提案する関連ルート CLI コマンド。  |

## activation リファレンス

Plugin を有効化または読み込みプランに含める必要があるコントロールプレーンイベントを、低コストで宣言できる場合は、`activation` を使用します。

このブロックはプランナーのメタデータであり、ライフサイクル API ではありません。ランタイム動作を登録せず、`register(...)` を置き換えず、Plugin コードがすでに実行されたことも保証しません。有効化プランナーは、これらのフィールドを使用して候補となる Plugin を絞り込み、その後、`providers`、`channels`、`commandAliases`、`setup.providers`、`contracts.tools`、フックなど、既存のマニフェスト所有権メタデータにフォールバックします。

所有権をすでに表している最も限定的なメタデータを優先してください。`providers`、`channels`、`commandAliases`、セットアップディスクリプター、または `contracts` で関係を表現できる場合は、それらのフィールドを使用します。これらの所有権フィールドでは表現できない追加のプランナーヒントには、`activation` を使用します。`claude-cli`、`my-cli`、`google-gemini-cli` などの CLI ランタイムエイリアスには、トップレベルの `cliBackends` を使用します。`activation.onAgentHarnesses` は、所有権フィールドがまだ存在しない組み込みエージェントハーネス ID 専用です。

すべての Plugin で `activation.onStartup` を意図的に設定する必要があります。Gateway の起動中に Plugin を実行する必要がある場合にのみ、`true` に設定します。Plugin が起動時には何もせず、より限定的なトリガーによってのみ読み込まれるべき場合は、`false` に設定します。`onStartup` を省略しても、Plugin が起動時に暗黙的に読み込まれることはなくなりました。起動、チャネル、設定、エージェントハーネス、メモリ、またはその他のより限定的な有効化トリガーには、明示的な有効化メタデータを使用してください。

```json
{
  "activation": {
    "onStartup": false,
    "onProviders": ["openai"],
    "onCommands": ["models"],
    "onChannels": ["web"],
    "onRoutes": ["gateway-webhook"],
    "onConfigPaths": ["browser"],
    "onCapabilities": ["provider", "tool"]
  }
}
```

| フィールド              | 必須 | 型                                                 | 意味                                                                                                                                                                               |
| ------------------ | -------- | ---------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `onStartup`        | いいえ       | `boolean`                                            | Gateway の明示的な起動時有効化です。すべての Plugin でこれを設定する必要があります。`true` は起動時に Plugin をインポートし、`false` は一致する別のトリガーによって読み込みが必要になるまで、起動時の遅延読み込みを維持します。 |
| `onProviders`      | いいえ       | `string[]`                                           | 有効化／読み込み計画にこの Plugin を含める必要があるプロバイダー ID です。                                                                                                                      |
| `onAgentHarnesses` | いいえ       | `string[]`                                           | 有効化／読み込み計画にこの Plugin を含める必要がある、組み込みエージェントハーネスのランタイム ID です。CLI バックエンドのエイリアスにはトップレベルの `cliBackends` を使用します。                                           |
| `onCommands`       | いいえ       | `string[]`                                           | 有効化／読み込み計画にこの Plugin を含める必要があるコマンド ID です。                                                                                                                       |
| `onChannels`       | いいえ       | `string[]`                                           | 有効化／読み込み計画にこの Plugin を含める必要があるチャンネル ID です。                                                                                                                       |
| `onRoutes`         | いいえ       | `string[]`                                           | 有効化／読み込み計画にこの Plugin を含める必要があるルート種別です。                                                                                                                       |
| `onConfigPaths`    | いいえ       | `string[]`                                           | パスが存在し、明示的に無効化されていない場合に、起動／読み込み計画にこの Plugin を含める必要があるルート相対の設定パスです。                                                      |
| `onCapabilities`   | いいえ       | `Array<"provider" \| "channel" \| "tool" \| "hook">` | コントロールプレーンの有効化計画で使用される広範な機能ヒントです。可能な場合は、より限定的なフィールドを優先してください。                                                                                     |

現在の実稼働コンシューマー：

- Gateway の起動計画では、明示的な起動時インポートに `activation.onStartup` を使用します。
- コマンドによってトリガーされる CLI 計画では、従来の `commandAliases[].cliCommand` または `commandAliases[].name` にフォールバックします。
- エージェントランタイムの起動計画では、組み込みハーネスに `activation.onAgentHarnesses`、CLI ランタイムのエイリアスにトップレベルの `cliBackends[]` を使用します。
- チャンネルによってトリガーされるセットアップ／チャンネル計画では、明示的なチャンネル有効化メタデータがない場合、従来の `channels[]` の所有権にフォールバックします。
- 起動時の Plugin 計画では、バンドルされたブラウザー Plugin の `browser` ブロックなど、チャンネル以外のルート設定サーフェスに `activation.onConfigPaths` を使用します。
- プロバイダーによってトリガーされるセットアップ／ランタイム計画では、明示的なプロバイダー有効化メタデータがない場合、従来の `providers[]` およびトップレベルの `cliBackends[]` の所有権にフォールバックします。

プランナー診断では、明示的な有効化ヒントとマニフェスト所有権へのフォールバックを区別できます。たとえば、`activation-command-hint` は `activation.onCommands` が一致したことを意味し、`manifest-command-alias` はプランナーが代わりに `commandAliases` の所有権を使用したことを意味します。これらの理由ラベルはホスト診断とテスト用です。Plugin 作成者は、所有権を最も適切に表すメタデータを引き続き宣言する必要があります。

## qaRunners リファレンス

Plugin が共有 `openclaw qa` ルート配下に 1 つ以上のトランスポートランナーを提供する場合は、`qaRunners` を使用します。
このメタデータは低コストかつ静的に保ってください。実際の CLI 登録は引き続き Plugin
ランタイムが担当し、一致する `qaRunnerCliRegistrations` をエクスポートする軽量な
`runtime-api.ts` サーフェスを介して行います。オプションの
`adapterFactory` は、登録済みコマンドのランナーを変更せずに、共有 QA シナリオへ
トランスポートを公開します。

```json
{
  "qaRunners": [
    {
      "commandName": "matrix",
      "description": "使い捨てのホームサーバーに対して、Docker ベースの Matrix ライブ QA レーンを実行する"
    }
  ]
}
```

| フィールド         | 必須 | 型     | 意味                                                      |
| ------------- | -------- | -------- | ------------------------------------------------------------------ |
| `commandName` | はい      | `string` | `openclaw qa` 配下にマウントされるサブコマンドです。例：`matrix`。    |
| `description` | いいえ       | `string` | 共有ホストでスタブコマンドが必要な場合に使用されるフォールバックのヘルプテキストです。 |

`adapterFactory` ID は `commandName` と一致する必要があります。マニフェストに存在しない
コマンドの登録をエクスポートしないでください。

## setup リファレンス

セットアップおよびオンボーディングのサーフェスで、ランタイムが読み込まれる前に低コストな Plugin 所有メタデータが必要な場合は、`setup` を使用します。

```json
{
  "setup": {
    "providers": [
      {
        "id": "openai",
        "authMethods": ["api-key"],
        "envVars": ["OPENAI_API_KEY"],
        "authEvidence": [
          {
            "type": "local-file-with-env",
            "fileEnvVar": "OPENAI_CREDENTIALS_FILE",
            "requiresAllEnv": ["OPENAI_PROJECT"],
            "credentialMarker": "openai-local-credentials",
            "source": "openai のローカル認証情報"
          }
        ]
      }
    ],
    "cliBackends": ["openai-cli"],
    "configMigrations": ["legacy-openai-auth"],
    "requiresRuntime": false
  }
}
```

トップレベルの `cliBackends` は引き続き有効で、CLI 推論バックエンドを引き続き記述します。`setup.cliBackends` は、メタデータのみに留める必要があるコントロールプレーン／セットアップフロー向けの、セットアップ固有の記述子サーフェスです。

存在する場合、`setup.providers` と `setup.cliBackends` は、セットアップ検出で優先される記述子優先のルックアップサーフェスです。記述子が候補 Plugin の絞り込みのみを行い、セットアップでより高度なセットアップ時ランタイムフックが引き続き必要な場合は、`requiresRuntime: true` を設定し、フォールバック実行パスとして `setup-api` を維持してください。

OpenClaw は、汎用的なプロバイダー認証および環境変数のルックアップに `setup.providers[].envVars` を含めます。セットアップおよびステータス用の環境メタデータはそこに配置してください。

請求または組織レベルの認証情報によって、推論用認証情報にすることなく `resolveUsageAuth` を有効化する必要がある場合は、`providerUsageAuthEnvVars` を使用します。これらの名前は、ワークスペースの dotenv ブロック、ACP 子プロセスからの除去、サンドボックスのシークレットフィルタリング、および広範なシークレット消去の対象に加わります。プロバイダーランタイムは引き続き `resolveUsageAuth` 内で値を読み取り、分類します。

セットアップエントリがない場合、または `setup.requiresRuntime: false` がセットアップランタイムは不要であると宣言している場合、OpenClaw は `setup.providers[].authMethods` から単純なセットアップ選択肢を導出することもできます。カスタムラベル、CLI フラグ、オンボーディング範囲、アシスタントメタデータには、明示的な `providerAuthChoices` エントリが引き続き優先されます。

これらの記述子だけでセットアップサーフェスに十分な場合にのみ、`requiresRuntime: false` を設定してください。OpenClaw は明示的な `false` を記述子のみの契約として扱い、セットアップのルックアップで `setup-api` または `openclaw.setupEntry` を実行しません。記述子のみの Plugin がこれらのセットアップランタイムエントリのいずれかを引き続き同梱している場合、OpenClaw は追加の診断を報告し、それを無視し続けます。`requiresRuntime` を省略すると従来のフォールバック動作が維持されるため、フラグなしで記述子を追加した既存の Plugin が破損することはありません。

セットアップのルックアップでは Plugin 所有の `setup-api` コードを実行できるため、正規化された `setup.providers[].id` および `setup.cliBackends[]` の値は、検出された Plugin 間で一意に保つ必要があります。所有権が曖昧な場合は、検出順序から採用するものを選ばず、フェイルクローズします。

セットアップランタイムが実行される場合、`setup-api` がマニフェスト記述子で宣言されていないプロバイダーまたは CLI バックエンドを登録したとき、あるいは記述子に対応するランタイム登録がないとき、セットアップレジストリ診断は記述子のずれを報告します。これらの診断は追加的なものであり、従来の Plugin を拒否しません。

### setup.providers リファレンス

| フィールド          | 必須 | 型       | 意味                                                                                    |
| -------------- | -------- | ---------- | ------------------------------------------------------------------------------------------------ |
| `id`           | はい      | `string`   | セットアップまたはオンボーディング中に公開されるプロバイダー ID です。正規化された ID はグローバルに一意に保ってください。             |
| `authMethods`  | いいえ       | `string[]` | このプロバイダーが完全なランタイムを読み込まずにサポートするセットアップ／認証方式の ID です。                       |
| `envVars`      | いいえ       | `string[]` | Plugin ランタイムが読み込まれる前に、汎用的なセットアップ／ステータスサーフェスが確認できる環境変数です。               |
| `authEvidence` | いいえ       | `object[]` | シークレットではないマーカーを介して認証できるプロバイダー向けの、低コストなローカル認証証拠チェックです。 |

`authEvidence` は、ランタイムコードを読み込まずに検証できる、プロバイダー所有のローカル認証情報マーカー用です。これらのチェックは低コストかつローカルに保つ必要があります。ネットワーク呼び出し、キーチェーンまたはシークレットマネージャーの読み取り、シェルコマンド、プロバイダー API のプローブは禁止です。

サポートされる証拠エントリ：

| フィールド              | 必須 | 型       | 意味                                                                                                  |
| ------------------ | -------- | ---------- | -------------------------------------------------------------------------------------------------------------- |
| `type`             | はい      | `string`   | 現在は `local-file-with-env` です。                                                                               |
| `fileEnvVar`       | いいえ       | `string`   | 明示的な認証情報ファイルのパスを含む環境変数です。                                                           |
| `fallbackPaths`    | いいえ       | `string[]` | `fileEnvVar` が存在しないか空の場合に確認される、ローカル認証情報ファイルのパスです。`${HOME}` および `${APPDATA}` をサポートします。 |
| `requiresAnyEnv`   | いいえ       | `string[]` | 証拠が有効になるには、一覧の環境変数のうち少なくとも 1 つが空でない必要があります。                                    |
| `requiresAllEnv`   | いいえ       | `string[]` | 証拠が有効になるには、一覧のすべての環境変数が空でない必要があります。                                           |
| `credentialMarker` | はい      | `string`   | 証拠が存在する場合に返される、シークレットではないマーカーです。                                                       |
| `source`           | いいえ       | `string`   | 認証／ステータス出力用のユーザー向けソースラベルです。                                                               |

### setup フィールド

| フィールド              | 必須 | 型       | 意味                                                                                       |
| ------------------ | -------- | ---------- | --------------------------------------------------------------------------------------------------- |
| `providers`        | いいえ       | `object[]` | セットアップおよびオンボーディング中に公開されるプロバイダーセットアップ記述子。                                     |
| `cliBackends`      | いいえ       | `string[]` | 記述子優先のセットアップ検索に使用されるセットアップ時のバックエンド ID。正規化された ID はグローバルに一意にしてください。 |
| `configMigrations` | いいえ       | `string[]` | この Plugin のセットアップサーフェスが所有する設定移行 ID。                                          |
| `requiresRuntime`  | いいえ       | `boolean`  | 記述子検索後もセットアップで `setup-api` の実行が必要かどうか。                            |

## uiHints リファレンス

`uiHints` は、設定フィールド名から小さなレンダリングヒントへのマップです。ネストされた設定フィールドではキーにドットを使用できますが、パスセグメントを `__proto__`、`constructor`、または `prototype` にすることはできません。セットアップではこれらの名前が拒否されます。

```json
{
  "uiHints": {
    "apiKey": {
      "label": "API キー",
      "help": "OpenRouter リクエストに使用",
      "placeholder": "sk-or-v1-...",
      "sensitive": true
    }
  }
}
```

各フィールドヒントには次を含められます。

| フィールド          | 型             | 意味                                                                                                     |
| -------------- | ---------------- | ----------------------------------------------------------------------------------------------------------------- |
| `label`        | `string`         | ユーザー向けのフィールドラベル。                                                                                          |
| `help`         | `string`         | 短いヘルプテキスト。                                                                                                |
| `tags`         | `string[]`       | オプションの UI タグ。                                                                                                 |
| `advanced`     | `boolean`        | フィールドを詳細設定としてマークします。                                                                                      |
| `sensitive`    | `boolean`        | フィールドをシークレットまたは機密としてマークします。                                                                           |
| `placeholder`  | `string`         | フォーム入力用のプレースホルダーテキスト。                                                                                 |
| `presentation` | `"phone-number"` | 解析可能な国際形式（`+...`）の値に対する表示専用のローカライズされた電話番号形式。未加工の値は変更されません。 |

## contracts リファレンス

`contracts` は、Plugin ランタイムをインポートせずに OpenClaw が読み取れる静的なケイパビリティ所有権メタデータにのみ使用してください。

```json
{
  "contracts": {
    "agentToolResultMiddleware": ["openclaw", "codex"],
    "trustedToolPolicies": ["workflow-budget"],
    "externalAuthProviders": ["acme-ai"],
    "embeddingProviders": ["openai-compatible"],
    "speechProviders": ["openai"],
    "realtimeTranscriptionProviders": ["openai"],
    "realtimeVoiceProviders": ["openai"],
    "memoryEmbeddingProviders": ["local"],
    "mediaUnderstandingProviders": ["openai"],
    "imageGenerationProviders": ["openai"],
    "videoGenerationProviders": ["qwen"],
    "musicGenerationProviders": ["stability-audio"],
    "documentExtractors": ["example-docs"],
    "webContentExtractors": ["firecrawl"],
    "webFetchProviders": ["firecrawl"],
    "webSearchProviders": ["gemini"],
    "workerProviders": ["example-worker"],
    "usageProviders": ["acme-ai"],
    "migrationProviders": ["hermes"],
    "gatewayMethodDispatch": ["authenticated-request"],
    "tools": ["firecrawl_search", "firecrawl_scrape"]
  }
}
```

各リストは省略可能です。

| フィールド                            | 型       | 意味                                                                                                                        |
| -------------------------------- | ---------- | ------------------------------------------------------------------------------------------------------------------------------------ |
| `embeddedExtensionFactories`     | `string[]` | Codex app-server 拡張ファクトリー ID。現在は `codex-app-server`。                                                                |
| `agentToolResultMiddleware`      | `string[]` | この Plugin がツール結果ミドルウェアを登録できるランタイム ID。                                                                     |
| `trustedToolPolicies`            | `string[]` | インストール済み Plugin が登録できる、Plugin ローカルの信頼済みツール実行前ポリシー ID。バンドル済み Plugin はこのフィールドなしでポリシーを登録できます。 |
| `externalAuthProviders`          | `string[]` | この Plugin が外部認証プロファイルフックを所有するプロバイダー ID。                                                                      |
| `embeddingProviders`             | `string[]` | メモリを含む再利用可能なベクトル埋め込み用途向けに、この Plugin が所有する汎用埋め込みプロバイダー ID。                                 |
| `speechProviders`                | `string[]` | この Plugin が所有する音声プロバイダー ID。                                                                                                |
| `realtimeTranscriptionProviders` | `string[]` | この Plugin が所有するリアルタイム文字起こしプロバイダー ID。                                                                                |
| `realtimeVoiceProviders`         | `string[]` | この Plugin が所有するリアルタイム音声プロバイダー ID。                                                                                        |
| `memoryEmbeddingProviders`       | `string[]` | この Plugin が所有する非推奨のメモリ固有埋め込みプロバイダー ID。                                                                  |
| `mediaUnderstandingProviders`    | `string[]` | この Plugin が所有するメディア理解プロバイダー ID。                                                                                   |
| `transcriptSourceProviders`      | `string[]` | この Plugin が所有するトランスクリプトソースプロバイダー ID。                                                                                     |
| `documentExtractors`             | `string[]` | この Plugin が所有するドキュメント（PDF など）抽出プロバイダー ID。                                                                  |
| `imageGenerationProviders`       | `string[]` | この Plugin が所有する画像生成プロバイダー ID。                                                                                      |
| `videoGenerationProviders`       | `string[]` | この Plugin が所有する動画生成プロバイダー ID。                                                                                      |
| `musicGenerationProviders`       | `string[]` | この Plugin が所有する音楽生成プロバイダー ID。                                                                                      |
| `webContentExtractors`           | `string[]` | この Plugin が所有するウェブページコンテンツ抽出プロバイダー ID。                                                                           |
| `webFetchProviders`              | `string[]` | この Plugin が所有するウェブ取得プロバイダー ID。                                                                                             |
| `webSearchProviders`             | `string[]` | この Plugin が所有するウェブ検索プロバイダー ID。                                                                                            |
| `workerProviders`                | `string[]` | プロビジョニングおよびプロファイルに基づくリースライフサイクル向けに、この Plugin が所有するクラウドワーカープロバイダー ID。                                      |
| `usageProviders`                 | `string[]` | この Plugin が使用量認証フックと使用量スナップショットフックを所有するプロバイダー ID。                                                             |
| `migrationProviders`             | `string[]` | この Plugin が `openclaw migrate` 用に所有するインポートプロバイダー ID。                                                                         |
| `gatewayMethodDispatch`          | `string[]` | プロセス内で Gateway メソッドをディスパッチする、認証済み Plugin HTTP ルート用に予約された権限。                                  |
| `tools`                          | `string[]` | この Plugin が所有するエージェントツール名。                                                                                                   |

`contracts.embeddedExtensionFactories` は、バンドル済みの Codex app-server 専用拡張ファクトリー用に保持されています。バンドル済みのツール結果変換では、代わりに `contracts.agentToolResultMiddleware` を宣言し、`api.registerAgentToolResultMiddleware(...)` で登録する必要があります。インストール済み Plugin は、明示的に有効化され、かつ `contracts.agentToolResultMiddleware` で宣言したランタイムに対してのみ、同じミドルウェア接続点を使用できます。

ホストから信頼されるツール実行前ポリシー階層を必要とするインストール済み Plugin は、登録する各ローカル ID を `contracts.trustedToolPolicies` で宣言し、明示的に有効化される必要があります。バンドル済み Plugin は既存の信頼済みポリシーパスを維持しますが、未宣言のポリシー ID を持つインストール済み Plugin は登録前に拒否されます。ポリシー ID は登録元 Plugin のスコープ内にあるため、2 つの Plugin が両方とも `workflow-budget` を宣言して登録できますが、1 つの Plugin が同じローカル ID を 2 回登録することはできません。

ランタイムの `api.registerTool(...)` 登録は `contracts.tools` と一致する必要があります。ツール検出ではこのリストを使用し、要求されたツールを所有できる Plugin ランタイムだけを読み込みます。

`resolveExternalAuthProfiles` を実装するプロバイダー Plugin は、`contracts.externalAuthProviders` を宣言する必要があります。未宣言の外部認証フックは無視されます。

`resolveUsageAuth` と `fetchUsageSnapshot` の両方を実装するプロバイダー Plugin は、自動検出される各プロバイダー ID を `contracts.usageProviders` で宣言する必要があります。使用量検出はランタイムコードを読み込む前にこのコントラクトを読み取り、宣言された所有者のみを読み込んだ後に両方のフックを検証します。

汎用埋め込みプロバイダーは、`api.registerEmbeddingProvider(...)` で登録する各アダプターについて `contracts.embeddingProviders` を宣言する必要があります。メモリ検索で使用されるプロバイダーを含む、再利用可能なベクトル生成には汎用コントラクトを使用してください。`contracts.memoryEmbeddingProviders` は非推奨のメモリ固有互換機能であり、既存のプロバイダーが汎用埋め込みプロバイダー接続点へ移行する間だけ維持されます。

ワーカープロバイダーは、各 `api.registerWorkerProvider(...)` ID を `contracts.workerProviders` で宣言する必要があります。コアは `provision` を呼び出す前に永続的な意図を保存します。プロバイダーは外部割り当て前に設定を検証し、同じ操作 ID による呼び出しが繰り返された場合は、同じリースを引き継ぐ必要があります。また、コアは検証済みの設定スナップショットを永続化し、名前付きプロファイルが変更または削除された後も含め、`leaseId` とともに `inspect({ leaseId, profile })` および `destroy({ leaseId, profile })` に渡します。破棄は冪等であり、検査ではクローズドな `active` / `destroyed` / `unknown` ステータスユニオンを返します。SSH 秘密鍵の内容は `SecretRef` を介してのみ参照されます。プロビジョニングされた SSH エンドポイントには、信頼済みプロビジョニング出力から得た公開 `hostKey` も、ホスト名やコメントを付けずに正確に `algorithm base64` として含める必要があります。これにより、コアは接続前にホストをピン留めできます。動的な ID 参照を発行するプロバイダーは、権威ある `resolveSshIdentity({ leaseId, profile, keyRef })` を実装できます。実装しないプロバイダーでは、コアの汎用シークレットリゾルバーが使用されます。権威ある `unknown` はアクティブなローカルレコードを孤立状態にし、破棄リクエストの永続化後は解体を確認します。

`contracts.gatewayMethodDispatch` は現在 `"authenticated-request"` を受け入れます。これは、意図的にプロセス内で Gateway コントロールプレーンメソッドをディスパッチするネイティブ Plugin HTTP ルート向けの API 衛生ゲートであり、悪意のあるネイティブ Plugin に対するサンドボックスではありません。Gateway HTTP 認証をすでに必要とする、厳密にレビューされたバンドル済みまたはオペレーター向けサーフェスにのみ使用してください。権限が付与されたルートは、Gateway のルートワーク受付が閉じている間でも、そのルートが `auth: "gateway"` とルート固有の `gatewayRuntimeScopeSurface: "trusted-operator"` も宣言している場合に限り到達可能なままです。同じ Plugin の通常の兄弟ルートは、引き続き受付境界の内側に留まります。これにより、Plugin 全体に受付バイパスを付与することなく、一時停止状態の確認と再開を利用可能なまま維持できます。解析とレスポンス整形はディスパッチの外側で限定的に行ってください。実質的な処理や変更を伴う処理は、受付とスコープの適用を担う Gateway メソッドディスパッチを経由する必要があります。

## configContracts リファレンス

Plugin ランタイムをインポートせずに汎用コアヘルパーが必要とする、マニフェスト所有の設定動作（危険なフラグの検出、SecretRef の移行先、レガシー設定パスの絞り込み）には `configContracts` を使用します。

```json
{
  "configContracts": {
    "compatibilityMigrationPaths": ["legacyProvider"],
    "compatibilityRuntimePaths": ["legacyProvider.webhook"],
    "dangerousFlags": [
      {
        "path": "accounts.*.allowUnverifiedSenders",
        "equals": true
      }
    ],
    "secretInputs": {
      "bundledDefaultEnabled": false,
      "paths": [
        {
          "path": "routes.*.secret",
          "expected": "string",
          "ownerKind": "route"
        }
      ]
    }
  }
}
```

| フィールド                         | 必須 | 型       | 意味                                                                                                                                                                                                                          |
| ----------------------------- | -------- | ---------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `compatibilityMigrationPaths` | いいえ       | `string[]` | この Plugin のセットアップ時の互換性移行が適用される可能性を示す、ルート相対の設定パス。設定内で Plugin が参照されていない場合、汎用ランタイムによる設定読み取りで、その Plugin のすべてのセットアップサーフェスをスキップできます。                 |
| `compatibilityRuntimePaths`   | いいえ       | `string[]` | Plugin コードが完全に有効化される前に、この Plugin がランタイム中に処理できるルート相対の互換性パス。互換性のあるすべての Plugin ランタイムをインポートせずに、バンドル済み候補セットを絞り込む必要があるレガシーサーフェスに使用します。 |
| `dangerousFlags`              | いいえ       | `object[]` | 有効になっている場合に `openclaw doctor` が安全でない、または危険であるとフラグ付けすべき設定リテラル。以下を参照してください。                                                                                                                                   |
| `secretInputs`                | いいえ       | `object`   | SecretRef の移行、監査、起動時の実体化、およびオプションのランタイム所有者分離に使用する、`plugins.entries.<id>.config` 配下の設定パス。以下を参照してください。                                                                             |

各 `dangerousFlags` エントリは以下をサポートします。

| フィールド    | 必須 | 型                                  | 意味                                                                                                       |
| -------- | -------- | ------------------------------------- | ------------------------------------------------------------------------------------------------------------------- |
| `path`   | はい      | `string`                              | `plugins.entries.<id>.config` からの相対指定となる、ドット区切りの設定パス。マップまたは配列セグメントに `*` ワイルドカードを使用できます。 |
| `equals` | はい      | `string \| number \| boolean \| null` | この設定値が危険であることを示す正確なリテラル。                                                            |

`secretInputs` は以下をサポートします。

| フィールド                   | 必須 | 型       | 意味                                                                                                                                                                                                                                                                                                                                              |
| ----------------------- | -------- | ---------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `bundledDefaultEnabled` | いいえ       | `boolean`  | この SecretRef サーフェスが有効かどうかを判断するときに、バンドル済み Plugin のデフォルトの有効化状態を上書きします。Plugin はバンドル済みでも、設定で明示的に有効化されるまでサーフェスを無効のままにする必要がある場合に使用します。                                                                                                                                            |
| `paths`                 | はい      | `object[]` | シークレット形式の設定パス。各パスには `path`（ドット区切り、`plugins.entries.<id>.config` からの相対指定、`*` ワイルドカードをサポート）、オプションの `expected`（現在は `"string"` のみ）、およびオプションの `ownerKind`（現在は `"route"` のみ）が含まれます。宣言された所有者は、解決に失敗した場合に、その正確に一致したパスのみを分離します。その所有者 ID は完全な設定パスです。 |

## mediaUnderstandingProviderMetadata リファレンス

メディア理解プロバイダーにデフォルトモデル、自動認証フォールバックの優先順位、またはランタイムのロード前に汎用コアヘルパーが必要とするネイティブドキュメント対応がある場合は、`mediaUnderstandingProviderMetadata` を使用します。キーは `contracts.mediaUnderstandingProviders` 内でも宣言する必要があります。

```json
{
  "contracts": {
    "mediaUnderstandingProviders": ["example"]
  },
  "mediaUnderstandingProviderMetadata": {
    "example": {
      "capabilities": ["image", "audio"],
      "defaultModels": {
        "image": "example-vision-latest",
        "audio": "example-transcribe-latest"
      },
      "autoPriority": {
        "image": 40
      },
      "nativeDocumentInputs": ["pdf"],
      "documentModels": {
        "pdf": {
          "textExtraction": "example-doc-text-latest",
          "image": "example-doc-vision-latest"
        }
      }
    }
  }
}
```

各プロバイダーエントリには以下を含めることができます。

| フィールド                  | 型                                                             | 意味                                                                                                   |
| ---------------------- | ---------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------- |
| `capabilities`         | `("image" \| "audio" \| "video")[]`                              | このプロバイダーが公開するメディア機能。                                                                    |
| `defaultModels`        | `Record<string, string>`                                         | 設定でモデルが指定されていない場合に使用される、機能からモデルへのデフォルト。                                         |
| `autoPriority`         | `Record<string, number>`                                         | 資格情報に基づくプロバイダーの自動フォールバックでは、数値が小さいほど先に並びます。                                    |
| `nativeDocumentInputs` | `"pdf"[]`                                                        | プロバイダーがサポートするネイティブドキュメント入力。                                                               |
| `documentModels`       | `{ pdf?: { textExtraction?: string; image?: string \| false } }` | ドキュメント種別ごとのモデル上書き。そのドキュメント種別で画像ベースの抽出を無効にするには、`image: false` を設定します。 |

## channelConfigs リファレンス

チャンネル Plugin がランタイムのロード前に低コストの設定メタデータを必要とする場合は、`channelConfigs` を使用します。セットアップエントリが利用できない場合、または `setup.requiresRuntime: false` がセットアップランタイムは不要であると宣言している場合、読み取り専用のチャンネルセットアップ／状態検出では、設定済みの外部チャンネルに対してこのメタデータを直接使用できます。

`channelConfigs` は Plugin マニフェストのメタデータであり、新しいトップレベルのユーザー設定セクションではありません。ユーザーは引き続き `channels.<channel-id>` 配下でチャンネルインスタンスを設定します。OpenClaw は、Plugin ランタイムコードが実行される前に、設定済みチャンネルをどの Plugin が所有するかを判断するためにマニフェストメタデータを読み取ります。

チャンネル Plugin では、`configSchema` と `channelConfigs` は異なるパスを記述します。

- `configSchema` は `plugins.entries.<plugin-id>.config` を検証します
- `channelConfigs.<channel-id>.schema` は `channels.<channel-id>` を検証します

`channels[]` を宣言する非バンドル Plugin は、一致する `channelConfigs` エントリも宣言する必要があります。これらがなくても OpenClaw は Plugin をロードできますが、コールドパスの設定スキーマ、セットアップ、および Control UI サーフェスは、Plugin ランタイムが実行されるまで、チャンネル所有オプションの形状や表示専用 UI ヒントを認識できません。

`channelConfigs.<channel-id>.commands.nativeCommandsAutoEnabled` と `nativeSkillsAutoEnabled` は、チャンネルランタイムのロード前に実行されるコマンド設定チェック向けに、静的な `auto` デフォルトを宣言できます。バンドル済みチャンネルは、パッケージ所有の他のチャンネルカタログメタデータとともに、`package.json#openclaw.channel.commands` を通じて同じデフォルトを公開することもできます。

```json
{
  "channelConfigs": {
    "matrix": {
      "schema": {
        "type": "object",
        "additionalProperties": false,
        "properties": {
          "homeserverUrl": { "type": "string" }
        }
      },
      "uiHints": {
        "homeserverUrl": {
          "label": "Homeserver URL",
          "placeholder": "https://matrix.example.com"
        }
      },
      "label": "Matrix",
      "description": "Matrix homeserver connection",
      "commands": {
        "nativeCommandsAutoEnabled": true,
        "nativeSkillsAutoEnabled": true
      },
      "preferOver": ["matrix-legacy"]
    }
  }
}
```

各チャンネルエントリには以下を含めることができます。

| フィールド         | 型                     | 意味                                                                                                    |
| ------------- | ------------------------ | ---------------------------------------------------------------------------------------------------------------- |
| `schema`      | `object`                 | `channels.<id>` の JSON Schema。宣言された各チャンネル設定エントリで必須です。                                |
| `uiHints`     | `Record<string, object>` | そのチャンネル設定セクション向けのオプションのラベル、プレースホルダー、機密性、および表示専用の表現ヒント。 |
| `label`       | `string`                 | ランタイムメタデータの準備ができていない場合に、選択画面と検査サーフェスへマージされるチャンネルラベル。                        |
| `description` | `string`                 | 検査およびカタログサーフェス向けの短いチャンネル説明。                                                      |
| `commands`    | `object`                 | ランタイム前の設定チェック向けの、ネイティブコマンドとネイティブ Skills の静的な自動デフォルト。                              |
| `preferOver`  | `string[]`               | 選択サーフェスで、このチャンネルが優先すべきレガシーまたは低優先度の Plugin ID。                           |

### 別のチャンネル Plugin の置き換え

別の Plugin も提供できるチャンネル ID に対して、自身の Plugin を優先所有者にする場合は `preferOver` を使用します。一般的なケースとしては、名前が変更された Plugin ID、バンドル済み Plugin を置き換えるスタンドアロン Plugin、または設定互換性のために同じチャンネル ID を維持するメンテナンス済みフォークがあります。

```json
{
  "id": "acme-chat",
  "channels": ["chat"],
  "channelConfigs": {
    "chat": {
      "schema": {
        "type": "object",
        "additionalProperties": false,
        "properties": {
          "webhookUrl": { "type": "string" }
        }
      },
      "preferOver": ["chat"]
    }
  }
}
```

`channels.chat` が設定されている場合、OpenClaw はチャンネル ID と優先される Plugin ID の両方を考慮します。優先度の低い Plugin が、バンドルされているかデフォルトで有効であるという理由だけで選択されていた場合、OpenClaw は有効なランタイム設定でその Plugin を無効にし、1 つの Plugin がチャンネルとそのツールを所有するようにします。明示的なユーザー選択は引き続き優先されます。ユーザーが両方の Plugin を明示的に有効にした場合（`plugins.allow` または実質的な `plugins.entries` 設定を使用）、OpenClaw は要求された Plugin セットを暗黙に変更する代わりに、その選択を維持し、チャンネルまたはツールの重複に関する診断を報告します。

`preferOver` は、実際に同じチャンネルを提供できる Plugin ID のみに限定してください。これは汎用的な優先度フィールドではなく、ユーザー設定キーの名前を変更するものでもありません。

## modelSupport リファレンス

Plugin ランタイムを読み込む前に、OpenClaw が `gpt-5.6-sol` や `claude-sonnet-4.6` のような短縮モデル ID からプロバイダー Plugin を推測する必要がある場合は、`modelSupport` を使用します。

```json
{
  "modelSupport": {
    "modelPrefixes": ["gpt-", "o1", "o3", "o4"],
    "modelPatterns": ["^computer-use-preview"]
  }
}
```

OpenClaw は次の優先順位を適用します。

- 明示的な `provider/model` 参照では、所有元の `providers` マニフェストメタデータを使用する
- `modelPatterns` は `modelPrefixes` より優先される
- バンドルされていない Plugin とバンドルされている Plugin がどちらも一致する場合、バンドルされていない Plugin が優先される
- 残りの曖昧性は、ユーザーまたは設定がプロバイダーを指定するまで無視される

フィールド：

| フィールド           | 型       | 意味                                                                   |
| --------------- | ---------- | ------------------------------------------------------------------------------- |
| `modelPrefixes` | `string[]` | 短縮モデル ID に対して `startsWith` で照合されるプレフィックス。                 |
| `modelPatterns` | `string[]` | プロファイルサフィックスの除去後に、短縮モデル ID に対して照合される正規表現ソース。 |

`modelPatterns` エントリは `compileSafeRegex` を介してコンパイルされ、入れ子になった繰り返しを含むパターン（例：`(a+)+$`）は拒否されます。安全性チェックに失敗したパターンは、構文的に無効な正規表現と同様に暗黙にスキップされます。パターンは単純に保ち、量指定子の入れ子は避けてください。

## modelCatalog リファレンス

Plugin ランタイムを読み込む前に、OpenClaw がプロバイダーモデルのメタデータを認識する必要がある場合は、`modelCatalog` を使用します。これは、固定カタログ行、プロバイダーエイリアス、抑制ルール、検出モードについて、マニフェストが所有するソースです。ランタイムでの更新は引き続きプロバイダーのランタイムコードが担いますが、ランタイムが必要になるタイミングはマニフェストによってコアに通知されます。

```json
{
  "providers": ["openai"],
  "modelCatalog": {
    "providers": {
      "openai": {
        "baseUrl": "https://api.openai.com/v1",
        "api": "openai-responses",
        "models": [
          {
            "id": "gpt-5.4",
            "name": "GPT-5.4",
            "input": ["text", "image"],
            "reasoning": true,
            "contextWindow": 256000,
            "maxTokens": 128000,
            "cost": {
              "input": 1.25,
              "output": 10,
              "cacheRead": 0.125
            },
            "status": "available",
            "tags": ["default"]
          }
        ]
      }
    },
    "aliases": {
      "azure-openai-responses": {
        "provider": "openai",
        "api": "azure-openai-responses"
      }
    },
    "suppressions": [
      {
        "provider": "azure-openai-responses",
        "model": "gpt-5.3-codex-spark",
        "reason": "not available on Azure OpenAI Responses"
      }
    ],
    "discovery": {
      "openai": "static"
    }
  }
}
```

トップレベルのフィールド：

| フィールド            | 型                                                     | 意味                                                                                               |
| ---------------- | -------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------- |
| `providers`      | `Record<string, object>`                                 | この Plugin が所有するプロバイダー ID のカタログ行。キーはトップレベルの `providers` にも含める必要があります。       |
| `aliases`        | `Record<string, object>`                                 | カタログまたは抑制の計画時に、所有プロバイダーへ解決されるプロバイダーエイリアス。              |
| `suppressions`   | `object[]`                                               | この Plugin がプロバイダー固有の理由で抑制する、別のソース由来のモデル行。                  |
| `discovery`      | `Record<string, "static" \| "refreshable" \| "runtime">` | プロバイダーカタログをマニフェストメタデータから読み取れるか、キャッシュへ更新できるか、またはランタイムが必要か。 |
| `runtimeAugment` | `boolean`                                                | マニフェストおよび設定の計画後にプロバイダーランタイムがカタログ行を追加する必要がある場合にのみ、`true` に設定します。       |

`aliases` は、モデルカタログの計画におけるプロバイダー所有権の検索に関与します。エイリアスのターゲットは、同じ Plugin が所有するトップレベルプロバイダーでなければなりません。プロバイダーでフィルタリングされたリストがエイリアスを使用する場合、OpenClaw はプロバイダーランタイムを読み込まずに、所有元のマニフェストを読み取り、エイリアスの API／ベース URL オーバーライドを適用できます。エイリアスは、フィルタリングされていないカタログ一覧を展開しません。広範な一覧では、所有元の正規プロバイダー行のみが出力されます。

`suppressions` は、以前のプロバイダーランタイムの `suppressBuiltInModel` フックを置き換えます。抑制エントリが適用されるのは、プロバイダーがその Plugin によって所有されているか、所有プロバイダーをターゲットとする `modelCatalog.aliases` キーとして宣言されている場合のみです。モデル解決中にランタイム抑制フックが呼び出されることはなくなりました。

プロバイダーのフィールド：

| フィールド                 | 型                     | 意味                                                                                                                                                                                                     |
| --------------------- | ------------------------ | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `baseUrl`             | `string`                 | このプロバイダーカタログ内のモデルに対する、オプションのデフォルトベース URL。                                                                                                                                                    |
| `api`                 | `ModelApi`               | このプロバイダーカタログ内のモデルに対する、オプションのデフォルト API アダプター。                                                                                                                                                 |
| `headers`             | `Record<string, string>` | このプロバイダーカタログに適用される、オプションの静的ヘッダー。                                                                                                                                                      |
| `defaultUtilityModel` | `string`                 | 短い内部ユーティリティタスク（タイトル、進捗の説明）向けにプロバイダーが推奨する、オプションの小型モデル ID。`agents.defaults.utilityModel` が未設定で、このプロバイダーがエージェントのプライマリモデルを提供する場合に使用されます。 |
| `models`              | `object[]`               | 必須のモデル行。`id` がない行は無視されます。                                                                                                                                                            |

モデルのフィールド：

| フィールド              | 型                                                           | 意味                                                               |
| ------------------ | -------------------------------------------------------------- | --------------------------------------------------------------------------- |
| `id`               | `string`                                                       | `provider/` プレフィックスを含まない、プロバイダー内のモデル ID。                    |
| `name`             | `string`                                                       | オプションの表示名。                                                      |
| `api`              | `ModelApi`                                                     | オプションのモデル単位の API オーバーライド。                                            |
| `baseUrl`          | `string`                                                       | オプションのモデル単位のベース URL オーバーライド。                                       |
| `headers`          | `Record<string, string>`                                       | オプションのモデル単位の静的ヘッダー。                                          |
| `input`            | `Array<"text" \| "image" \| "document">`                       | モデルが受け付けるモダリティ。その他の値は暗黙に破棄されます。            |
| `reasoning`        | `boolean`                                                      | モデルが推論動作を公開するかどうか。                               |
| `contextWindow`    | `number`                                                       | プロバイダー固有のコンテキストウィンドウ。                                             |
| `contextTokens`    | `number`                                                       | `contextWindow` と異なる場合の、オプションの有効なランタイムコンテキスト上限。 |
| `maxTokens`        | `number`                                                       | 既知の場合の最大出力トークン数。                                           |
| `thinkingLevelMap` | `Record<string, string \| null>`                               | オプションの思考レベル単位のモデル ID またはパラメーターのオーバーライド。                    |
| `cost`             | `object`                                                       | オプションの `tieredPricing` を含む、100 万トークンあたりのオプションの USD 料金。 |
| `compat`           | `object`                                                       | OpenClaw のモデル設定の互換性に対応する、オプションの互換性フラグ。  |
| `mediaInput`       | `object`                                                       | オプションのモダリティ単位の入力設定。現在は画像のみに対応しています。                   |
| `status`           | `"available"` \| `"preview"` \| `"deprecated"` \| `"disabled"` | 一覧表示ステータス。行を一切表示してはならない場合にのみ抑制します。          |
| `statusReason`     | `string`                                                       | 利用不可ステータスとともに表示される、オプションの理由。                            |
| `replaces`         | `string[]`                                                     | このモデルが後継となる、以前のプロバイダー内モデル ID。                       |
| `replacedBy`       | `string`                                                       | 非推奨の行に対する、後継のプロバイダー内モデル ID。                    |
| `tags`             | `string[]`                                                     | ピッカーとフィルターで使用される安定したタグ。                                    |

抑制フィールド：

| フィールド                      | 型       | 意味                                                                                             |
| -------------------------- | ---------- | --------------------------------------------------------------------------------------------------------- |
| `provider`                 | `string`   | 抑制する上流行のプロバイダー ID。このプラグインが所有しているか、所有するエイリアスとして宣言されている必要があります。 |
| `model`                    | `string`   | 抑制するプロバイダー内のモデル ID。                                                                      |
| `reason`                   | `string`   | 抑制された行が直接要求されたときに表示する任意のメッセージ。                                     |
| `when.baseUrlHosts`        | `string[]` | 抑制を適用するために必要な、有効なプロバイダーのベース URL ホストの任意のリスト。               |
| `when.providerConfigApiIn` | `string[]` | 抑制を適用するために必要な、プロバイダー設定の正確な `api` 値の任意のリスト。              |

ランタイム専用データを `modelCatalog` に含めないでください。マニフェストの行が十分に完全で、プロバイダーで絞り込んだリストおよびピッカーの画面でレジストリやランタイムの検出を省略できる場合にのみ、`static` を使用します。マニフェストの行が一覧表示可能なシードまたは補足として役立つものの、後から更新やキャッシュによって行を追加できる場合は、`refreshable` を使用します。更新可能な行だけでは信頼できる情報源になりません。リストを把握するために OpenClaw がプロバイダーのランタイムを読み込む必要がある場合は、`runtime` を使用します。

## modelIdNormalization リファレンス

プロバイダーのランタイムを読み込む前に実行する必要がある、低コストでプロバイダー所有のモデル ID クリーンアップには、`modelIdNormalization` を使用します。これにより、短いモデル名、プロバイダー内のレガシー ID、プロキシのプレフィックスルールなどのエイリアスを、コアのモデル選択テーブルではなく、所有元プラグインのマニフェストに保持できます。

```json
{
  "providers": ["anthropic", "openrouter"],
  "modelIdNormalization": {
    "providers": {
      "anthropic": {
        "aliases": {
          "sonnet-4.6": "claude-sonnet-4-6"
        }
      },
      "openrouter": {
        "prefixWhenBare": "openrouter"
      }
    }
  }
}
```

プロバイダーのフィールド:

| フィールド                                | 型                    | 意味                                                                             |
| ------------------------------------ | ----------------------- | ----------------------------------------------------------------------------------------- |
| `aliases`                            | `Record<string,string>` | 大文字と小文字を区別しない、モデル ID の完全一致エイリアス。値は記述どおりに返されます。                  |
| `stripPrefixes`                      | `string[]`              | エイリアス検索の前に削除するプレフィックス。レガシーなプロバイダー名とモデル名の重複に役立ちます。     |
| `prefixWhenBare`                     | `string`                | 正規化されたモデル ID に `/` がまだ含まれていない場合に追加するプレフィックス。                  |
| `prefixWhenBareAfterAliasStartsWith` | `object[]`              | エイリアス検索後に適用する、裸の ID に対する条件付きプレフィックスルール。`modelPrefix` と `prefix` をキーとします。 |

## providerEndpoints リファレンス

プロバイダーのランタイムを読み込む前に汎用リクエストポリシーが把握する必要のあるエンドポイント分類には、`providerEndpoints` を使用します。各 `endpointClass` の意味は引き続きコアが所有し、ホストとベース URL のメタデータはプラグインのマニフェストが所有します。

正式に外部化されたプロバイダープラグインはコアの配布物から除外されるため、
インストールされるまでそのマニフェストは参照できません。プラグインがなくても
エンドポイント分類が機能し続けるように、その `providerEndpoints` も
`scripts/lib/official-external-provider-catalog.json` にミラーリングする必要があります。このミラーリングは
コントラクトテストによって強制されます。

エンドポイントのフィールド:

| フィールド                          | 型       | 意味                                                                                  |
| ------------------------------ | ---------- | ---------------------------------------------------------------------------------------------- |
| `endpointClass`                | `string`   | `openrouter`、`moonshot-native`、`google-vertex` など、既知のコアエンドポイントクラス。        |
| `hosts`                        | `string[]` | エンドポイントクラスに対応付ける正確なホスト名。                                                |
| `hostSuffixes`                 | `string[]` | エンドポイントクラスに対応付けるホストのサフィックス。ドメインサフィックスのみの一致には `.` を先頭に付けます。 |
| `baseUrls`                     | `string[]` | エンドポイントクラスに対応付ける、正規化済みの正確な HTTP(S) ベース URL。                             |
| `googleVertexRegion`           | `string`   | 完全一致するグローバルホスト向けの静的な Google Vertex リージョン。                                            |
| `googleVertexRegionHostSuffix` | `string`   | 一致するホストから削除して Google Vertex のリージョンプレフィックスを抽出するサフィックス。                 |

## providerRequest リファレンス

プロバイダーのランタイムを読み込まずに汎用リクエストポリシーが必要とする、低コストなリクエスト互換性メタデータには、`providerRequest` を使用します。動作固有のペイロード書き換えは、プロバイダーのランタイムフックまたは共有プロバイダーファミリーヘルパーに保持します。

```json
{
  "providerRequest": {
    "providers": {
      "vllm": {
        "family": "vllm",
        "openAICompletions": {
          "supportsStreamingUsage": true
        }
      }
    }
  }
}
```

プロバイダーのフィールド:

| フィールド                 | 型         | 意味                                                                          |
| --------------------- | ------------ | -------------------------------------------------------------------------------------- |
| `family`              | `string`     | 汎用的なリクエスト互換性の判断と診断で使用するプロバイダーファミリーのラベル。 |
| `compatibilityFamily` | `"moonshot"` | 共有リクエストヘルパー向けの任意のプロバイダーファミリー互換性バケット。              |
| `openAICompletions`   | `object`     | OpenAI 互換の補完リクエストフラグ。現在は `supportsStreamingUsage`。       |

## secretProviderIntegrations リファレンス

プラグインが再利用可能な SecretRef exec プロバイダープリセットを公開できる場合は、`secretProviderIntegrations` を使用します。OpenClaw はプラグインのランタイムを読み込む前にこのメタデータを読み取り、プラグインの所有権を `secrets.providers.<alias>.pluginIntegration` に保存し、実際のシークレット解決は SecretRef ランタイムに委ねます。プリセットは、バンドルされたプラグインと、git や ClawHub からのインストールなど、管理対象のプラグインインストールルートから検出されたインストール済みプラグインに対してのみ公開されます。

```json
{
  "secretProviderIntegrations": {
    "secret-store": {
      "providerAlias": "team-secrets",
      "displayName": "Team secrets",
      "source": "exec",
      "command": "${node}",
      "args": ["./bin/resolve-secrets.mjs"]
    }
  }
}
```

マップのキーはインテグレーション ID です。`providerAlias` を省略した場合、OpenClaw はインテグレーション ID を SecretRef プロバイダーのエイリアスとして使用します。プロバイダーのエイリアスは、通常の SecretRef プロバイダーエイリアスのパターン（たとえば `team-secrets` や `onepassword-work`）に一致する必要があります。

オペレーターがプリセットを選択すると、OpenClaw は次のようなプロバイダー参照を書き込みます。

```json
{
  "secrets": {
    "providers": {
      "team-secrets": {
        "source": "exec",
        "pluginIntegration": {
          "pluginId": "acme-secrets",
          "integrationId": "secret-store"
        }
      }
    }
  }
}
```

起動時または再読み込み時に、OpenClaw は現在のプラグインマニフェストのメタデータを読み込み、所有元プラグインがインストール済みで有効であることを確認し、マニフェストから exec コマンドを実体化して、そのプロバイダーを解決します。プラグインを無効化または削除すると、有効な SecretRef に対するプロバイダーが取り消されます。スタンドアロンの exec 設定を使用するオペレーターは、手動の `command`/`args` プロバイダーを引き続き直接記述できます。

現在サポートされているのは `source: "exec"` プリセットのみです。`command` は `${node}` である必要があり、`args[0]` はプラグインルートを基準とする `./` リゾルバースクリプトである必要があります。OpenClaw は起動時または再読み込み時に、現在の Node 実行ファイルとプラグイン内スクリプトの絶対パスへ実体化します。`--require`、`--import`、`--loader`、`--env-file`、`--eval`、`--print` などの Node オプションは、マニフェストプリセットのコントラクトには含まれません。Node 以外のコマンドが必要なオペレーターは、スタンドアロンの手動 exec プロバイダーを直接設定できます。

OpenClaw は、マニフェストプリセットの `trustedDirs` をプラグインルートから導出し、`${node}` プリセットの場合は現在の Node 実行ファイルのディレクトリからも導出します。マニフェストに記述された `trustedDirs` は無視されます。`timeoutMs`、`noOutputTimeoutMs`、`maxOutputBytes`、`jsonOnly`、`env`、`passEnv`、`allowInsecurePath` など、その他の exec プロバイダーオプションは、通常の SecretRef exec プロバイダー設定へそのまま渡されます。

## modelPricing リファレンス

ランタイムの読み込み前にプロバイダーがコントロールプレーンの価格設定動作を制御する必要がある場合は、`modelPricing` を使用します。Gateway の価格キャッシュは、プロバイダーのランタイムコードをインポートせずにこのメタデータを読み取ります。

```json
{
  "providers": ["ollama", "openrouter"],
  "modelPricing": {
    "providers": {
      "ollama": {
        "external": false
      },
      "openrouter": {
        "openRouter": {
          "passthroughProviderModel": true
        },
        "liteLLM": false
      }
    }
  }
}
```

プロバイダーのフィールド:

| フィールド        | 型              | 意味                                                                                      |
| ------------ | ----------------- | -------------------------------------------------------------------------------------------------- |
| `external`   | `boolean`         | OpenRouter または LiteLLM の価格情報を取得してはならないローカルまたはセルフホスト型のプロバイダーには、`false` を設定します。 |
| `openRouter` | `false \| object` | OpenRouter の価格検索マッピング。`false` は、このプロバイダーの OpenRouter 検索を無効にします。           |
| `liteLLM`    | `false \| object` | LiteLLM の価格検索マッピング。`false` は、このプロバイダーの LiteLLM 検索を無効にします。                 |

ソースのフィールド:

| フィールド                      | 型               | 意味                                                                                                        |
| -------------------------- | ------------------ | -------------------------------------------------------------------------------------------------------------------- |
| `provider`                 | `string`           | 外部カタログのプロバイダー ID が OpenClaw のプロバイダー ID と異なる場合の ID。たとえば `zai` プロバイダーでは `z-ai`。 |
| `passthroughProviderModel` | `boolean`          | スラッシュを含むモデル ID をネストされたプロバイダー/モデル参照として扱います。OpenRouter などのプロキシプロバイダーに役立ちます。       |
| `modelIdTransforms`        | `"version-dots"[]` | 外部カタログ用の追加モデル ID バリアント。`version-dots` は `claude-opus-4.6` のようなドット区切りのバージョン ID を試します。            |

### OpenClaw プロバイダーインデックス

OpenClaw プロバイダーインデックスは、プラグインがまだインストールされていない可能性があるプロバイダー向けの、OpenClaw が所有するプレビューメタデータです。これはプラグインマニフェストの一部ではありません。インストール済みプラグインについては、引き続きプラグインマニフェストが信頼できる情報源です。プロバイダーインデックスは、プロバイダープラグインがインストールされていない場合に、将来のインストール可能プロバイダー画面やインストール前のモデルピッカー画面が利用する内部フォールバックコントラクトです。

カタログの優先順位:

1. ユーザー設定。
2. インストール済みプラグインマニフェスト `modelCatalog`。
3. 明示的な更新によるモデルカタログキャッシュ。
4. OpenClaw プロバイダーインデックスのプレビュー行。

Provider Index には、シークレット、有効化状態、ランタイムフック、またはアカウント固有のライブモデルデータを含めてはなりません。そのプレビューカタログは、Plugin マニフェストと同じ `modelCatalog` プロバイダー行形式を使用しますが、`api`、`baseUrl`、価格設定、互換性フラグなどのランタイムアダプターフィールドを、インストール済み Plugin マニフェストと意図的に整合させる場合を除き、安定した表示メタデータのみに限定する必要があります。ライブ `/models` 検出を備えたプロバイダーは、通常の一覧表示やオンボーディングからプロバイダー API を呼び出すのではなく、明示的なモデルカタログキャッシュパスを通じて更新済みの行を書き込む必要があります。

Provider Index エントリには、Plugin がコアから移動済み、またはまだインストールされていないプロバイダー向けに、インストール可能な Plugin のメタデータを含めることもできます。このメタデータはチャンネルカタログのパターンを踏襲します。パッケージ名、npm インストール仕様、想定される整合性、簡易な認証選択肢ラベルがあれば、インストール可能なセットアップオプションを表示するには十分です。Plugin がインストールされると、そのマニフェストが優先され、そのプロバイダーの Provider Index エントリは無視されます。

`openclaw doctor --fix` は、従来のトップレベルマニフェスト機能キーのうち、小規模で閉じた一式を `contracts.*` に移行します。対象は `speechProviders`、`mediaUnderstandingProviders`、`imageGenerationProviders`、`tools` です。これら（およびその他の機能リスト）は、トップレベルのマニフェストフィールドとしては読み込まれなくなりました。通常のマニフェスト読み込みでは、`contracts` 配下にあるものだけが認識されます。

## マニフェストと package.json の違い

この 2 つのファイルは異なる役割を担います。

| ファイル                   | 用途                                                                                                                       |
| ---------------------- | -------------------------------------------------------------------------------------------------------------------------------- |
| `openclaw.plugin.json` | Plugin コードの実行前に存在している必要がある、検出、設定検証、認証選択肢のメタデータ、UI ヒント                         |
| `package.json`         | npm メタデータ、依存関係のインストール、およびエントリポイント、インストール制御、セットアップ、カタログメタデータに使用される `openclaw` ブロック |

メタデータをどちらに配置すべきか不明な場合は、次の規則に従ってください。

- OpenClaw が Plugin コードを読み込む前に把握する必要がある場合は、`openclaw.plugin.json` に配置する
- パッケージ化、エントリファイル、または npm のインストール動作に関する場合は、`package.json` に配置する

### 検出に影響する package.json フィールド

一部のランタイム前 Plugin メタデータは、意図的に `openclaw.plugin.json` ではなく、`package.json` の `openclaw` ブロック配下に置かれます。`openclaw.bundle` と `openclaw.bundle.json` は OpenClaw Plugin の契約ではありません。ネイティブ Plugin は、`openclaw.plugin.json` と、以下に示すサポート対象の `package.json#openclaw` フィールドを使用する必要があります。

主な例：

| フィールド                                                                                      | 意味                                                                                                                                                                             |
| ------------------------------------------------------------------------------------------ | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `openclaw.extensions`                                                                      | ネイティブ Plugin のエントリポイントを宣言します。Plugin パッケージディレクトリ内に収める必要があります。                                                                                                        |
| `openclaw.runtimeExtensions`                                                               | インストール済みパッケージのビルド済み JavaScript ランタイムエントリポイントを宣言します。Plugin パッケージディレクトリ内に収める必要があります。                                                                      |
| `openclaw.setupEntry`                                                                      | オンボーディング、遅延チャンネル起動、および読み取り専用のチャンネル状態／SecretRef 検出時に使用される、軽量なセットアップ専用エントリポイントです。Plugin パッケージディレクトリ内に収める必要があります。      |
| `openclaw.runtimeSetupEntry`                                                               | インストール済みパッケージのビルド済み JavaScript セットアップエントリポイントを宣言します。`setupEntry` が必須で、実在し、Plugin パッケージディレクトリ内に収める必要があります。                              |
| `openclaw.channel`                                                                         | ラベル、ドキュメントパス、エイリアス、選択肢の文言など、簡易なチャンネルカタログメタデータです。                                                                                                      |
| `openclaw.channel.approvalFlags`                                                           | ランタイム読み込み前に利用可能な、閉じた承認動作フラグです。`native` は、チャンネルがネイティブの承認 UI と同一ターン内での解決を担うことを意味します。                                                |
| `openclaw.channel.commands`                                                                | チャンネルランタイムの読み込み前に、設定、監査、コマンド一覧の各サーフェスで使用される、静的なネイティブコマンドおよびネイティブ Skills の自動デフォルトメタデータです。                                               |
| `openclaw.channel.cliAddOptions`                                                           | Plugin が所有する `openclaw channels add` オプションです。各エントリは、`flags`、`description`、任意の `defaultValue`、および汎用入力変換用の任意の `valueType`（`int` または `list`）を宣言します。 |
| `openclaw.channel.configuredState`                                                         | 完全なチャンネルランタイムを読み込まずに「環境変数のみのセットアップがすでに存在するか」に回答できる、軽量な設定済み状態チェッカーのメタデータです。                                              |
| `openclaw.channel.persistedAuthState`                                                      | 完全なチャンネルランタイムを読み込まずに「すでにサインイン済みのものがあるか」に回答できる、軽量な永続化済み認証チェッカーのメタデータです。                                                    |
| `openclaw.install.clawhubSpec` / `openclaw.install.npmSpec` / `openclaw.install.localPath` | バンドル済みおよび外部公開された Plugin 向けのインストール／更新ヒントです。                                                                                                                        |
| `openclaw.install.defaultChoice`                                                           | 複数のインストール元が利用可能な場合に優先されるインストールパスです。                                                                                                                       |
| `openclaw.install.minHostVersion`                                                          | サポートされる OpenClaw ホストの最低バージョンです。`>=2026.3.22` や `>=2026.5.1-beta.1` のような semver の下限を使用します。                                                                                  |
| `openclaw.compat.pluginApi`                                                                | このパッケージに必要な OpenClaw Plugin API の最小範囲です。`>=2026.5.27` のような semver の下限を使用します。                                                                                      |
| `openclaw.install.expectedIntegrity`                                                       | `sha512-...` など、想定される npm dist 整合性文字列です。インストールおよび更新フローでは、取得したアーティファクトをこれと照合して検証します。                                                                 |
| `openclaw.install.allowInvalidConfigRecovery`                                              | 設定が無効な場合に、限定的なバンドル済み Plugin の再インストール復旧パスを許可します。                                                                                                            |
| `openclaw.install.requiredPlatformPackages`                                                | lockfile のプラットフォーム制約が現在のホストと一致する場合に実体化される必要がある npm パッケージエイリアスです。                                                                                |
| `openclaw.startup.deferConfiguredChannelFullLoadUntilAfterListen`                          | セットアップランタイムのチャンネルサーフェスを listen 前に読み込めるようにし、設定済みの完全なチャンネル Plugin の読み込みを listen 後の有効化まで延期します。                                                      |

マニフェストメタデータは、ランタイムの読み込み前にオンボーディングに表示されるプロバイダー／チャンネル／セットアップの選択肢を決定します。`package.json#openclaw.install` は、ユーザーがそれらの選択肢のいずれかを選んだとき、その Plugin を取得または有効化する方法をオンボーディングに伝えます。インストールヒントを `openclaw.plugin.json` に移動しないでください。

`openclaw.channel.cliAddOptions` には、`--initial-sync-limit <n>` のような Commander の長形式オプション構文を使用してください。Plugin セットアップアダプターが受け取る前に、負でない整数として解析するには `valueType: "int"` を、カンマ、セミコロン、または改行で区切られた入力を文字列に分割するには `valueType: "list"` を設定します。解析済みの Commander 値を変更せずに渡す場合は、`valueType` を省略します。

`openclaw.install.minHostVersion` は、バンドルされていない Plugin ソースのインストール時およびマニフェストレジストリ読み込み時に適用されます。無効な値は拒否されます。有効ではあるものの新しすぎる値の場合、古いホストでは外部 Plugin がスキップされます。バンドル済みソース Plugin は、ホストのチェックアウトと同じバージョンであると見なされます。

`openclaw.install.requiredPlatformPackages` は、任意のプラットフォーム固有エイリアスを通じて必須のネイティブバイナリを公開する npm パッケージ向けです。サポートするすべてのプラットフォームエイリアスについて、修飾なしの npm パッケージ名を記載します。npm のインストール中、OpenClaw は lockfile の制約が現在のホストと一致する宣言済みエイリアスのみを検証します。npm が成功を報告してもそのエイリアスが欠落している場合、OpenClaw は新しいキャッシュで 1 回再試行し、それでもエイリアスが欠落していればインストールをロールバックします。

`openclaw.compat.pluginApi` は、バンドルされていない Plugin ソースのパッケージインストール時に適用されます。パッケージのビルド時に基準とした OpenClaw Plugin SDK／ランタイム API の下限として使用してください。Plugin パッケージがより新しい API を必要としつつ、他のフロー向けには低いインストールヒントを維持する場合、これは `minHostVersion` より厳しくできます。公式の OpenClaw リリース同期では、既存の公式 Plugin API の下限がデフォルトで OpenClaw のリリースバージョンに引き上げられますが、パッケージが古いホストを意図的にサポートする場合、Plugin のみのリリースでは低い下限を維持できます。パッケージバージョンだけを互換性契約として使用しないでください。`peerDependencies.openclaw` は引き続き npm パッケージメタデータです。OpenClaw はインストール互換性の判定に `openclaw.compat.pluginApi` 契約を使用します。

公式のオンデマンドインストールメタデータでは、Plugin が ClawHub で公開されている場合、`clawhubSpec` を使用する必要があります。オンボーディングはこれを優先リモートソースとして扱い、インストール後に ClawHub アーティファクト情報を記録します。`npmSpec` は、まだ ClawHub に移行していないパッケージ向けの互換性フォールバックとして残ります。

npm の正確なバージョン固定は、すでに `npmSpec` に存在します。たとえば `"npmSpec": "@wecom/wecom-openclaw-plugin@1.2.3"` です。公式の外部カタログエントリでは、正確な仕様と `expectedIntegrity` を組み合わせ、取得した npm アーティファクトが固定されたリリースと一致しなくなった場合に更新フローがフェイルクローズするようにする必要があります。対話型オンボーディングでは、互換性のため、修飾なしのパッケージ名や dist-tag を含む、信頼済みレジストリの npm 仕様も引き続き提供されます。カタログ診断では、正確なソース、可変ソース、整合性固定済みソース、整合性欠落ソース、パッケージ名不一致ソース、無効なデフォルト選択ソースを区別できます。また、`expectedIntegrity` が存在しても、それを固定できる有効な npm ソースがない場合は警告します。`expectedIntegrity` が存在する場合、インストール／更新フローはそれを適用します。省略された場合、レジストリ解決結果は整合性固定なしで記録されます。

状態、チャンネル一覧、または SecretRef スキャンで、完全なランタイムを読み込まずに設定済みアカウントを識別する必要がある場合、チャンネル Plugin は `openclaw.setupEntry` を提供する必要があります。セットアップエントリでは、チャンネルメタデータに加え、セットアップ時に安全な設定、状態、シークレットの各アダプターを公開してください。ネットワーククライアント、Gateway リスナー、トランスポートランタイムは、メインの拡張機能エントリポイントに保持してください。

ランタイムエントリポイントのフィールドは、ソースエントリポイントのフィールドに対するパッケージ境界チェックを上書きしません。たとえば、`openclaw.runtimeExtensions` を使用しても、境界外へ抜ける `openclaw.extensions` パスを読み込み可能にはできません。

`openclaw.install.allowInvalidConfigRecovery` は意図的に限定されています。任意の壊れた設定をインストール可能にするものではありません。現在は、バンドル済みプラグインのパスが見つからない場合や、同じバンドル済みプラグインの古い `channels.<id>` エントリがある場合など、バンドル済みプラグインのアップグレードに関する特定の古い状態からインストールフローを復旧できるようにするだけです。無関係な設定エラーは引き続きインストールをブロックし、運用担当者を `openclaw doctor --fix` に誘導します。

`openclaw.channel.persistedAuthState` は、小規模なチェッカーモジュール用のパッケージメタデータです。

```json
{
  "openclaw": {
    "channel": {
      "id": "whatsapp",
      "persistedAuthState": {
        "specifier": "./auth-presence",
        "exportName": "hasAnyWhatsAppAuth"
      }
    }
  }
}
```

セットアップ、doctor、ステータス、または読み取り専用の存在確認フローで、完全なチャンネルプラグインを読み込む前に低コストな yes/no 認証プローブが必要な場合に使用します。永続化された認証状態は、設定済みのチャンネル状態ではありません。このメタデータを使用してプラグインを自動的に有効化したり、ランタイム依存関係を修復したり、チャンネルランタイムを読み込むべきかどうかを判断したりしないでください。対象のエクスポートは、永続化された状態のみを読み取る小さな関数にしてください。完全なチャンネルランタイムのバレルを経由させないでください。

`openclaw.channel.configuredState` は、低コストな設定済みチェックをサポートします。環境変数で十分な場合は、宣言的な環境メタデータを優先してください。

```json
{
  "openclaw": {
    "channel": {
      "id": "telegram",
      "configuredState": {
        "env": {
          "allOf": ["TELEGRAM_BOT_TOKEN"]
        }
      }
    }
  }
}
```

列挙されたすべての変数が必須の場合は `env.allOf` を使用し、空でない変数がいずれか 1 つあれば十分な場合は `env.anyOf` を使用します。小規模な非ランタイムチェックで環境メタデータ以上のものが必要な場合は、`persistedAuthState` の例のように `specifier` と `exportName` を使用します。`env` が存在する場合、OpenClaw はそのモジュールを読み込まずにこれを使用します。チェックに完全な設定解決または実際のチャンネルランタイムが必要な場合は、そのロジックをプラグインの `config.hasConfiguredState` フックに残してください。

## 検出の優先順位（重複するプラグイン ID）

OpenClaw は 3 つのルートからプラグインを検出し、次の順序で確認します。OpenClaw に同梱されるバンドル済みプラグイン、グローバルインストールルート（`~/.openclaw/extensions`）、現在のワークスペースルート（`<workspace>/.openclaw/extensions`）に加え、明示的な `plugins.load.paths` エントリです。

2 つの検出結果が同じ `id` を共有する場合、**最も優先順位の高い**マニフェストだけが保持されます。優先順位の低い重複は、並べて読み込まれるのではなく破棄されます。優先順位は高い順に次のとおりです。

1. **設定で選択済み** — `plugins.entries.<id>` で明示的に固定されたパス
2. **追跡対象のインストール記録と一致するグローバルインストール** — `openclaw plugin install`/`openclaw plugin update` を介してインストールされ、その ID がバンドル済みプラグインにも属している場合でも、同じ ID に対して OpenClaw のインストール追跡が認識しているプラグイン
3. **バンドル済み** — OpenClaw に同梱されるプラグイン
4. **ワークスペース** — 現在のワークスペースを基準に検出されたプラグイン
5. その他の検出候補

影響は次のとおりです。

- ワークスペースまたはグローバルルートに未追跡の状態で置かれた、バンドル済みプラグインのフォークまたは古いコピーが、バンドル済みビルドを隠すことはありません。
- バンドル済みプラグインを上書きするには、その ID に対して `openclaw plugin install` を実行して、追跡対象のグローバルインストールがバンドル済みコピーより高い優先順位になるようにするか、`plugins.entries.<id>` で特定のパスを固定し、設定で選択済みの優先順位によって勝つようにします。
- 重複の破棄はログに記録されるため、Doctor と起動診断で破棄されたコピーを示せます。
- 設定で選択された重複の上書きは、診断では明示的な上書きとして表現されますが、古いフォークや意図しない隠蔽を可視化したままにするため、引き続き警告されます。

## JSON Schema の要件

- **設定を受け付けない場合でも、すべてのプラグインに JSON Schema を同梱する必要があります**。
- 空のスキーマでも問題ありません（例: `{ "type": "object", "additionalProperties": false }`）。
- スキーマは、ランタイムではなく設定の読み書き時に検証されます。
- 新しい設定キーを追加してバンドル済みプラグインを拡張またはフォークする場合は、同時にそのプラグインの `openclaw.plugin.json` `configSchema` を更新してください。バンドル済みプラグインのスキーマは厳格であるため、`configSchema.properties` に `myNewKey` を追加せず、ユーザー設定に `plugins.entries.<id>.config.myNewKey` を追加すると、プラグインランタイムが読み込まれる前に拒否されます。

スキーマ拡張の例:

```json
{
  "configSchema": {
    "type": "object",
    "additionalProperties": false,
    "properties": {
      "myNewKey": {
        "type": "string"
      }
    }
  }
}
```

## 検証の動作

- 不明な `channels.*` キーは、チャンネル ID がプラグインマニフェストで宣言されている場合を除き、**エラー**です。同じ ID が `plugins.allow`、`plugins.entries`、または `plugins.installs`（参照されているものの、現在は検出できないプラグイン）にも存在する場合、OpenClaw はこれを代わりに**警告**へ格下げします。
- 不明なプラグイン ID を参照する `plugins.entries.<id>`、`plugins.allow`、および `plugins.deny` は、エラーではなく**警告**（「古い設定エントリを無視しました」）になります。そのため、アップグレードや削除済みまたは名前変更済みのプラグインによって Gateway の起動がブロックされることはありません。
- 不明なプラグイン ID を参照する `plugins.slots.memory` は**エラー**です。ただし、既知の公式外部プラグイン `memory-lancedb` の場合は、代わりに警告になります。
- プラグインがインストールされていても、マニフェストまたはスキーマが壊れているか欠落している場合、検証は失敗し、Doctor がプラグインエラーを報告します。
- プラグイン設定が存在していてもプラグインが**無効**の場合、設定は保持され、Doctor とログに**警告**が表示されます。

完全な `plugins.*` スキーマについては、[設定リファレンス](/ja-JP/gateway/configuration)を参照してください。

## 注記

- ローカルファイルシステムから読み込む場合を含め、**ネイティブ OpenClaw プラグインにはマニフェストが必須です**。ランタイムは引き続きプラグインモジュールを別途読み込みます。マニフェストは検出と検証にのみ使用されます。
- ネイティブマニフェストは JSON5 で解析されるため、最終的な値がオブジェクトである限り、コメント、末尾のカンマ、引用符なしのキーを使用できます。
- マニフェストローダーが読み取るのは、文書化されたマニフェストフィールドだけです。独自のトップレベルキーは避けてください。
- プラグインで不要な場合、`channels`、`providers`、`cliBackends`、および `skills` はすべて省略できます。
- `providerCatalogEntry` は軽量なままにし、広範なランタイムコードをインポートしないでください。リクエスト時の実行ではなく、静的なプロバイダーカタログメタデータまたは限定的な検出記述子に使用してください。
- 排他的なプラグイン種別は `plugins.slots.*` を介して選択されます。`kind: "memory"` は `plugins.slots.memory`（デフォルトは `memory-core`）を介し、`kind: "context-engine"` は `plugins.slots.contextEngine`（デフォルトは `legacy`）を介して選択されます。
- 排他的なプラグイン種別は、このマニフェストで宣言してください。ランタイムエントリの `OpenClawPluginDefinition.kind` は非推奨であり、古いプラグインとの互換性のためのフォールバックとしてのみ残されています。
- `setup.providers[].envVars` の環境変数メタデータは宣言的なものに限られます。ステータス、監査、Cron 配信の検証、その他の読み取り専用サーフェスでは、環境変数を設定済みとして扱う前に、引き続きプラグインの信頼性と有効なアクティベーションポリシーが適用されます。
- プロバイダーコードを必要とするランタイムウィザードのメタデータについては、[プロバイダーランタイムフック](/ja-JP/plugins/architecture-internals#provider-runtime-hooks)を参照してください。
- プラグインがネイティブモジュールに依存する場合は、ビルド手順とパッケージマネージャーの許可リスト要件（例: pnpm `allow-build-scripts` + `pnpm rebuild <package>`）を文書化してください。

## 関連項目

<CardGroup cols={3}>
  <Card title="プラグインの構築" href="/ja-JP/plugins/building-plugins" icon="rocket">
    プラグインのはじめに。
  </Card>
  <Card title="プラグインアーキテクチャ" href="/ja-JP/plugins/architecture" icon="diagram-project">
    内部アーキテクチャとケイパビリティモデル。
  </Card>
  <Card title="SDK の概要" href="/ja-JP/plugins/sdk-overview" icon="book">
    プラグイン SDK のリファレンスとサブパスインポート。
  </Card>
</CardGroup>
