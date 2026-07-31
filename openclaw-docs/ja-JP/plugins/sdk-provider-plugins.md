---
read_when:
    - 新しいモデルプロバイダー Plugin を構築する場合
    - OpenClaw に OpenAI 互換プロキシまたはカスタム LLM を追加する場合
    - プロバイダーの認証、カタログ、ランタイムフックを理解する必要があります
sidebarTitle: Provider plugins
summary: OpenClaw 用モデルプロバイダー Plugin の構築手順ガイド
title: プロバイダー Plugin の構築
x-i18n:
    generated_at: "2026-07-26T09:38:48Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: f9d175fafc034bd52e996d47e047df104f079f2aba66662b22e8dbdf6c21e7e0
    source_path: plugins/sdk-provider-plugins.md
    workflow: 16
---

OpenClaw にモデルプロバイダー（LLM）を追加するためのプロバイダー Plugin を構築します。モデル
カタログ、API キー認証、動的なモデル解決を実装します。

<Info>
  OpenClaw Plugin を初めて作成する場合は、まずパッケージ構造とマニフェストの設定について
  [はじめに](/ja-JP/plugins/building-plugins)を参照してください。
</Info>

<Tip>
  プロバイダー Plugin は、OpenClaw の通常の推論ループにモデルを追加します。モデルを、
  スレッド、Compaction、またはツールイベントを所有するネイティブエージェントデーモン経由で
  実行する必要がある場合は、デーモンプロトコルの詳細をコアに組み込むのではなく、
  プロバイダーを[エージェントハーネス](/ja-JP/plugins/sdk-agent-harness)と組み合わせてください。
</Tip>

## 手順

<Steps>
  <Step title="パッケージとマニフェスト">
    ### ステップ 1：パッケージとマニフェスト

    <CodeGroup>
    ```json package.json
    {
      "name": "@myorg/openclaw-acme-ai",
      "version": "1.0.0",
      "type": "module",
      "openclaw": {
        "extensions": ["./index.ts"],
        "providers": ["acme-ai"],
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
      "id": "acme-ai",
      "name": "Acme AI",
      "description": "Acme AI model provider",
      "providers": ["acme-ai"],
      "modelSupport": {
        "modelPrefixes": ["acme-"]
      },
      "setup": {
        "providers": [
          {
            "id": "acme-ai",
            "envVars": ["ACME_AI_API_KEY"]
          }
        ]
      },
      "providerAuthAliases": {
        "acme-ai-coding": "acme-ai"
      },
      "providerAuthChoices": [
        {
          "provider": "acme-ai",
          "method": "api-key",
          "choiceId": "acme-ai-api-key",
          "choiceLabel": "Acme AI API key",
          "groupId": "acme-ai",
          "groupLabel": "Acme AI",
          "cliFlag": "--acme-ai-api-key",
          "cliOption": "--acme-ai-api-key <key>",
          "cliDescription": "Acme AI API key"
        }
      ],
      "configSchema": {
        "type": "object",
        "additionalProperties": false
      }
    }
    ```
    </CodeGroup>

    `setup.providers[].envVars` により、OpenClaw は Plugin ランタイムを読み込まずに
    認証情報を検出できます。プロバイダーのバリアントで別のプロバイダー ID の認証を
    再利用する場合は、`providerAuthAliases` を追加します。`modelSupport` は
    任意です。これにより、ランタイムフックが存在する前でも、OpenClaw は
    `acme-large` のような短縮モデル ID からプロバイダー Plugin を自動的に
    読み込めます。`package.json` 内の `openclaw.compat` と
    `openclaw.build` は、ClawHub への公開に必要です
    （`openclaw.compat.pluginApi` と `openclaw.build.openclawVersion` が必須の 2 フィールドで、
    `minGatewayVersion` を省略すると `openclaw.install.minHostVersion` が使用されます）。

  </Step>

  <Step title="プロバイダーを登録する">
    最小構成のテキストプロバイダーには、`id`、`label`、`auth`、`catalog` が必要です。
    `catalog` はプロバイダーが所有するランタイム／設定フックです。ベンダーの
    ライブ API を呼び出し、`models.providers` エントリを返せます。

    ```typescript index.ts
    import { definePluginEntry } from "openclaw/plugin-sdk/plugin-entry";
    import { createProviderApiKeyAuthMethod } from "openclaw/plugin-sdk/provider-auth";

    export default definePluginEntry({
      id: "acme-ai",
      name: "Acme AI",
      description: "Acme AI model provider",
      register(api) {
        api.registerProvider({
          id: "acme-ai",
          label: "Acme AI",
          docsPath: "/providers/acme-ai",
          envVars: ["ACME_AI_API_KEY"],

          auth: [
            createProviderApiKeyAuthMethod({
              providerId: "acme-ai",
              methodId: "api-key",
              label: "Acme AI API key",
              hint: "API key from your Acme AI dashboard",
              optionKey: "acmeAiApiKey",
              flagName: "--acme-ai-api-key",
              envVar: "ACME_AI_API_KEY",
              promptMessage: "Enter your Acme AI API key",
              defaultModel: "acme-ai/acme-large",
            }),
          ],

          catalog: {
            order: "simple",
            run: async (ctx) => {
              const apiKey =
                ctx.resolveProviderApiKey("acme-ai").apiKey;
              if (!apiKey) return null;
              return {
                provider: {
                  baseUrl: "https://api.acme-ai.com/v1",
                  apiKey,
                  api: "openai-completions",
                  models: [
                    {
                      id: "acme-large",
                      name: "Acme Large",
                      reasoning: true,
                      input: ["text", "image"],
                      cost: { input: 3, output: 15, cacheRead: 0.3, cacheWrite: 3.75 },
                      contextWindow: 200000,
                      maxTokens: 32768,
                    },
                    {
                      id: "acme-small",
                      name: "Acme Small",
                      reasoning: false,
                      input: ["text"],
                      cost: { input: 1, output: 5, cacheRead: 0.1, cacheWrite: 1.25 },
                      contextWindow: 128000,
                      maxTokens: 8192,
                    },
                  ],
                },
              };
            },
          },
        });

        api.registerModelCatalogProvider({
          provider: "acme-ai",
          kinds: ["text"],
          liveCatalog: async (ctx) => {
            const apiKey = ctx.resolveProviderApiKey("acme-ai").apiKey;
            if (!apiKey) return null;
            return [
              {
                kind: "text",
                provider: "acme-ai",
                model: "acme-large",
                label: "Acme Large",
                source: "live",
              },
            ];
          },
        });
      },
    });
    ```

    `registerModelCatalogProvider` は、一覧／ヘルプ／選択 UI 向けの新しい
    コントロールプレーンカタログサーフェスで、`text`、`voice`、`image_generation`、
    `video_generation`、`music_generation` の各行を扱います。ベンダーエンドポイントの
    呼び出しとレスポンスのマッピングは Plugin 内に保持してください。共有される行の形式、
    ソースラベル、ヘルプのレンダリングは OpenClaw が所有します。

    これでプロバイダーが動作します。ユーザーは
    `openclaw onboard --acme-ai-api-key <key>` を実行し、
    モデルとして `acme-ai/acme-large` を選択できるようになります。

    ### ライブモデル検出

    プロバイダーが OpenAI 互換の `/models` API を公開している場合は、
    単一プロバイダー用ヘルパーで共有検出を有効にします。

    ```typescript
    catalog: {
      buildProvider: () => ({
        api: "openai-completions",
        baseUrl: "https://api.acme-ai.com/v1",
        models: [...STATIC_MODELS],
      }),
      buildStaticProvider: () => ({
        api: "openai-completions",
        baseUrl: "https://api.acme-ai.com/v1",
        models: [...STATIC_MODELS],
      }),
      liveModelDiscovery: true,
    },
    ```

    `liveModelDiscovery: true` は、次の動作を定めた公開 Plugin SDK 契約です。

    | 領域 | 契約 |
    | --- | --- |
    | 認証情報 | 検出では、カタログで解決されたプロバイダー認証情報を使用し、認証によって提供される場合は `discoveryApiKey` を優先します。シークレット参照マーカーがトークンとして送信されることはありません。既定のリクエストでは `Authorization: Bearer <token>` を使用します。別のベンダー認証方式には `buildRequestHeaders` を使用してください。 |
    | エンドポイント | 既定の URL は、有効なプロバイダーの `baseUrl` を基準とした `models` です。`allowExplicitBaseUrl` が有効な場合は、オペレーターによるオーバーライドも反映されます。別の相対パスには `endpointPath` を使用してください。固定されたベンダー URL に限り `endpointUrl: { url, requireBaseUrl }` を使用してください。有効なベース URL が引き続き `requireBaseUrl` と一致する場合にのみ検出が行われるため、カスタムプロキシの認証情報がベンダーに送信されることはありません。 |
    | ネットワーク制限 | フェッチでは OpenClaw の SSRF ガードを使用し、ページネーション全体で 1 つの 5 秒のタイムアウト予算、ページごとに 4 MiB のレスポンス上限、50 ページの上限を適用します。オリジンをまたぐページネーションリンクは拒否されます。オリジンをまたぐリダイレクト後は認証情報が削除されます。 |
    | キャッシュ | 成功した空でないカタログは、プロバイダー、エンドポイント、解決済みの認証情報ごとに 60 秒間キャッシュされます。空または使用不能な結果はキャッシュされません。 |
    | フィルタリング | ライブ ID が完全に一致する場合、信頼済みの静的メタデータが維持されます。新しい行は、保守的にテキスト／チャットモデルとして投影されます。無効、アーカイブ済み、非推奨、明示的に非チャット、埋め込み、再ランキング、モデレーション、音声、画像専用、動画専用の行は除外されます。非標準のレスポンスエンベロープから行を選択する場合にのみ `readRows` を使用してください。プロバイダー固有のモデルセマンティクスは、引き続きカスタムカタログに実装する必要があります。 |
    | 失敗 | ライブ検出は補助的な機能です。認証、ネットワーク、タイムアウト、ページネーション、解析、空のカタログ、フィルタリングの失敗時には、プロバイダーを削除するのではなく、プロバイダーが所有する静的シードを返します。 |

    Bearer 以外の認証または非標準の一覧エンドポイントには、
    `true` の代わりにオプションを渡します。

    ```typescript
    liveModelDiscovery: {
      endpointPath: "model-catalog",
      buildRequestHeaders: ({ apiKey, discoveryApiKey }) => ({
        "vendor-version": "2026-01-01",
        "x-api-key": discoveryApiKey ?? apiKey ?? "",
      }),
      readRows: (body) =>
        body && typeof body === "object" &&
        Array.isArray((body as { models?: unknown }).models)
          ? (body as { models: unknown[] }).models
          : [],
    },
    ```

    `endpointUrl` を無条件の代替ホストとして使用しないでください。その
    `requireBaseUrl` チェックは、モデル一覧のホストと推論ホストが異なる
    プロバイダーにおける認証情報の分離境界です。

    保守的な OpenAI 互換の投影ではなく、プロバイダーにカスタムモデルセマンティクスが
    必要な場合は、その投影を Plugin 内に保持し、共有フェッチの
    ライフサイクルには `openclaw/plugin-sdk/provider-catalog-live-runtime` を使用してください。このヘルパーを使用すると、
    プロバイダーのポリシーを OpenClaw コアに組み込むことなく、保護された HTTP フェッチ、
    プロバイダー認証ヘッダー、構造化された HTTP エラー、TTL キャッシュ、静的フォールバック動作を
    利用できます。

    ライブ API から、プロバイダーが所有する静的カタログのどの行が現在利用可能かだけが
    返される場合は、`buildLiveModelProviderConfig` を使用します。

    ```typescript index.ts
    import { definePluginEntry } from "openclaw/plugin-sdk/plugin-entry";
    import {
      buildLiveModelProviderConfig,
      type LiveModelCatalogFetchGuard,
    } from "openclaw/plugin-sdk/provider-catalog-live-runtime";

    const STATIC_MODELS = [
      {
        id: "acme-large",
        name: "Acme Large",
        reasoning: true,
        input: ["text", "image"],
        cost: { input: 3, output: 15, cacheRead: 0.3, cacheWrite: 3.75 },
        contextWindow: 200000,
        maxTokens: 32768,
      },
      {
        id: "acme-small",
        name: "Acme Small",
        reasoning: false,
        input: ["text"],
        cost: { input: 1, output: 5, cacheRead: 0.1, cacheWrite: 1.25 },
        contextWindow: 128000,
        maxTokens: 8192,
      },
    ] as const;

    async function buildAcmeLiveProvider(params: {
      apiKey: string;
      discoveryApiKey?: string;
      fetchGuard?: LiveModelCatalogFetchGuard;
    }) {
      return await buildLiveModelProviderConfig({
        providerId: "acme-ai",
        endpoint: "https://api.acme-ai.com/v1/models",
        providerConfig: {
          baseUrl: "https://api.acme-ai.com/v1",
          api: "openai-completions",
        },
        models: STATIC_MODELS,
        apiKey: params.apiKey,
        discoveryApiKey: params.discoveryApiKey,
        fetchGuard: params.fetchGuard,
        ttlMs: 60_000,
        auditContext: "acme-ai-model-discovery",
      });
    }

    export default definePluginEntry({
      id: "acme-ai",
      name: "Acme AI",
      register(api) {
        api.registerProvider({
          id: "acme-ai",
          label: "Acme AI",
          catalog: {
            order: "simple",
            run: async (ctx) => {
              const auth = ctx.resolveProviderAuth("acme-ai");
              const apiKey =
                auth.apiKey ?? ctx.resolveProviderApiKey("acme-ai").apiKey;
              if (!apiKey) return null;
              return {
                provider: await buildAcmeLiveProvider({
                  apiKey,
                  discoveryApiKey: auth.discoveryApiKey,
                }),
              };
            },
          },
          staticCatalog: {
            order: "simple",
            run: async () => ({
              provider: {
                baseUrl: "https://api.acme-ai.com/v1",
                api: "openai-completions",
                models: [...STATIC_MODELS],
              },
            }),
          },
        });
      },
    });
    ```

    プロバイダー API がより豊富なメタデータを返し、Plugin 自身で行を OpenClaw のモデル定義に変換する必要がある場合は、`getCachedLiveProviderModelRows` を使用します。

    ```typescript index.ts
    import {
      getCachedLiveProviderModelRows,
      LiveModelCatalogHttpError,
    } from "openclaw/plugin-sdk/provider-catalog-live-runtime";

    async function discoverAcmeModels(apiKey: string) {
      try {
        const rows = await getCachedLiveProviderModelRows({
          providerId: "acme-ai",
          endpoint: "https://api.acme-ai.com/v1/models",
          apiKey,
          ttlMs: 60_000,
          auditContext: "acme-ai-model-discovery",
        });
        return rows
          .map((row) => projectAcmeModel(row))
          .filter((model) => model !== null);
      } catch (error) {
        if (error instanceof LiveModelCatalogHttpError) {
          return STATIC_MODELS;
        }
        throw error;
      }
    }
    ```

    `run` は認証で保護された状態を維持し、使用可能な認証情報がない場合は `null` を返す必要があります。セットアップ、ドキュメント、テスト、ピッカー画面がライブネットワークアクセスに依存しないように、オフラインの `staticRun` または静的フォールバックを維持してください。モデルリストの鮮度に適した TTL を使用し、リクエスト時のファイルシステムポーリングを避け、アップストリームのレスポンスが OpenAI 互換の `{ data: [{ id, object }] }` 形式でない場合にのみ、プロバイダー固有の `readRows` / `readModelId` を渡してください。

    アップストリームプロバイダーが OpenClaw とは異なる制御トークンを使用する場合は、ストリーム経路を置き換える代わりに、小さな双方向テキスト変換を追加します。

    ```typescript
    api.registerTextTransforms({
      input: [
        { from: /red basket/g, to: "blue basket" },
        { from: /paper ticket/g, to: "digital ticket" },
        { from: /left shelf/g, to: "right shelf" },
      ],
      output: [
        { from: /blue basket/g, to: "red basket" },
        { from: /digital ticket/g, to: "paper ticket" },
        { from: /right shelf/g, to: "left shelf" },
      ],
    });
    ```

    `input` は、転送前に最終的なシステムプロンプトとテキストメッセージの内容を書き換えます。`output` は、OpenClaw が独自の制御マーカーを解析するか、チャンネルへ配信する前に、アシスタントのテキスト差分と最終テキストを書き換えます。

    API キー認証と単一のカタログベースのランタイムを持つテキストプロバイダーを 1 つだけ登録するバンドルプロバイダーでは、より限定的な `defineSingleProviderPluginEntry(...)` ヘルパーを優先してください。

    ```typescript
    import { defineSingleProviderPluginEntry } from "openclaw/plugin-sdk/provider-entry";

    export default defineSingleProviderPluginEntry({
      id: "acme-ai",
      name: "Acme AI",
      description: "Acme AI model provider",
      provider: {
        label: "Acme AI",
        docsPath: "/providers/acme-ai",
        auth: [
          {
            methodId: "api-key",
            label: "Acme AI API key",
            hint: "API key from your Acme AI dashboard",
            optionKey: "acmeAiApiKey",
            flagName: "--acme-ai-api-key",
            envVar: "ACME_AI_API_KEY",
            promptMessage: "Enter your Acme AI API key",
            defaultModel: "acme-ai/acme-large",
          },
        ],
        catalog: {
          buildProvider: () => ({
            api: "openai-completions",
            baseUrl: "https://api.acme-ai.com/v1",
            models: [{ id: "acme-large", name: "Acme Large" }],
          }),
          buildStaticProvider: () => ({
            api: "openai-completions",
            baseUrl: "https://api.acme-ai.com/v1",
            models: [{ id: "acme-large", name: "Acme Large" }],
          }),
        },
      },
    });
    ```

    `buildProvider` は、OpenClaw が実際のプロバイダー認証を解決できる場合に使用されるライブカタログ経路です。プロバイダー固有の検出を実行する場合があります。認証の設定前に安全に表示できるオフライン行にのみ `buildStaticProvider` を使用してください。認証情報を要求したり、ネットワークリクエストを行ったりしてはなりません。OpenClaw の `models list --all` 表示は現在、空の設定、空の環境、エージェント／ワークスペースパスなしで、バンドルされたプロバイダー Plugin に対してのみ静的カタログを実行します。

    認証フローでオンボーディング中に `models.providers.*`、エイリアス、エージェントのデフォルトモデルも修正する必要がある場合は、`openclaw/plugin-sdk/provider-onboard` のプリセットヘルパーを使用してください。最も限定的なヘルパーは、`createDefaultModelPresetAppliers(...)`、`createDefaultModelsPresetAppliers(...)`、`createModelCatalogPresetAppliers(...)` です。

    プロバイダーのネイティブエンドポイントが通常の `openai-completions` 転送でストリーミングされる使用量ブロックをサポートする場合は、プロバイダー ID のチェックをハードコードする代わりに、`openclaw/plugin-sdk/provider-catalog-shared` の共有カタログヘルパーを優先してください。`supportsNativeStreamingUsageCompat(...)` と `applyProviderNativeStreamingUsageCompat(...)` はエンドポイント機能マップからサポートを検出するため、Plugin がカスタムプロバイダー ID を使用している場合でも、ネイティブの Moonshot/DashScope 形式のエンドポイントはオプトインできます。

    上記のライブ検出例は、`/models` 形式のプロバイダー API を対象としています。この検出を `catalog.run` 内に配置し、使用可能な認証を条件とし、オフラインカタログ生成のために `staticRun` をネットワーク非依存に保ってください。

  </Step>

  <Step title="動的モデル解決を追加する">
    プロバイダーが任意のモデル ID（プロキシやルーターなど）を受け付ける場合は、`resolveDynamicModel` を追加します。

    ```typescript
    api.registerProvider({
      // ... id, label, auth, catalog from above

      resolveDynamicModel: (ctx) => ({
        id: ctx.modelId,
        name: ctx.modelId,
        provider: "acme-ai",
        api: "openai-completions",
        baseUrl: "https://api.acme-ai.com/v1",
        reasoning: false,
        input: ["text"],
        cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 },
        contextWindow: 128000,
        maxTokens: 8192,
      }),
    });
    ```

    解決にネットワーク呼び出しが必要な場合は、非同期のウォームアップに `prepareDynamicModel` を使用します。完了後に `resolveDynamicModel` が再度実行されます。

  </Step>

  <Step title="ランタイムフックを追加する（必要に応じて）">
    ほとんどのプロバイダーに必要なのは `catalog` と `resolveDynamicModel` だけです。プロバイダーの要件に応じて、フックを段階的に追加してください。

    共有ヘルパービルダーが、最も一般的なリプレイ／ツール互換ファミリーをカバーするようになったため、通常、Plugin で各フックを 1 つずつ手動接続する必要はありません。

    ```typescript
    import { buildProviderReplayFamilyHooks } from "openclaw/plugin-sdk/provider-model-shared";
    import { buildProviderStreamFamilyHooks } from "openclaw/plugin-sdk/provider-stream";
    import { buildProviderToolCompatFamilyHooks } from "openclaw/plugin-sdk/provider-tools";

    const GOOGLE_FAMILY_HOOKS = {
      ...buildProviderReplayFamilyHooks({ family: "google-gemini" }),
      ...buildProviderStreamFamilyHooks("google-thinking"),
      ...buildProviderToolCompatFamilyHooks("gemini"),
    };

    api.registerProvider({
      id: "acme-gemini-compatible",
      // ...
      ...GOOGLE_FAMILY_HOOKS,
    });
    ```

    現在利用可能なリプレイファミリー：

    | ファミリー | 接続される機能 | バンドル例 |
    | --- | --- | --- |
    | `openai-compatible` | OpenAI 互換転送向けの共有 OpenAI 形式リプレイポリシー。ツール呼び出し ID のサニタイズ、アシスタント先頭の順序修正、および転送で必要な場合の汎用 Gemini ターン検証を含む | `moonshot`、`ollama`、`xai`、`zai` |
    | `anthropic-by-model` | `modelId` によって選択される Claude 対応リプレイポリシー。解決されたモデルが実際に Claude ID の場合にのみ、Anthropic メッセージ転送に Claude 固有の思考ブロックのクリーンアップを適用する | `amazon-bedrock` |
    | `native-anthropic-by-model` | `anthropic-by-model` と同じモデル別 Claude ポリシーに加えて、ベンダーネイティブ ID を維持する必要がある転送向けに、ツール呼び出し ID のサニタイズとネイティブ Anthropic ツール使用 ID の保持を行う | `anthropic-vertex`、`clawrouter` |
    | `google-gemini` | ネイティブ Gemini リプレイポリシーとブートストラップリプレイのサニタイズ。共有ファミリーでは、テキスト出力の Gemini CLI でタグ付き推論を維持する。直接の `google` プロバイダーは、Gemini API の思考がネイティブな思考パーツとして届くため、`resolveReasoningOutputMode` を `native` に上書きする。 | `google`、`google-gemini-cli` |
    | `passthrough-gemini` | OpenAI 互換プロキシ転送を介して実行される Gemini モデル向けの Gemini 思考署名サニタイズ。ネイティブ Gemini リプレイ検証やブートストラップの書き換えは有効にしない | `openrouter`、`kilocode`、`opencode`、`opencode-go` |
    | `hybrid-anthropic-openai` | 1 つの Plugin 内に Anthropic メッセージと OpenAI 互換モデルのサーフェスが混在するプロバイダー向けのハイブリッドポリシー。オプションの Claude 専用思考ブロック破棄は Anthropic 側のみに限定される | `minimax` |

    現在利用可能なストリームファミリー：

    | ファミリー | 組み込む機能 | バンドル例 |
    | --- | --- | --- |
    | `google-thinking` | 共有ストリームパスでの Gemini thinking ペイロードの正規化 | `google`, `google-gemini-cli` |
    | `kilocode-thinking` | 共有プロキシストリームパスでの Kilo reasoning ラッパー。`kilo-auto/balanced` および未対応のプロキシ reasoning ID では thinking の注入をスキップ | `kilocode` |
    | `moonshot-thinking` | config と `/think` レベルから Moonshot のバイナリ形式のネイティブ thinking ペイロードへのマッピング | `moonshot` |
    | `minimax-fast-mode` | 共有ストリームパスでの MiniMax 高速モードのモデル書き換え | `minimax`, `minimax-portal` |
    | `openai-responses-defaults` | 共有ネイティブ OpenAI/Codex Responses ラッパー：帰属ヘッダー、`/fast`/`serviceTier`、テキストの詳細度、ネイティブ Codex ウェブ検索、reasoning 互換ペイロードの整形、Responses のコンテキスト管理 | `openai` |
    | `openrouter-thinking` | プロキシルート用の OpenRouter reasoning ラッパー。未対応モデル/`auto` のスキップを一元的に処理 | `openrouter` |
    | `tool-stream-default-on` | 明示的に無効化されない限りツールストリーミングを使用する Z.AI などのプロバイダー向け、デフォルトで有効な `tool_stream` ラッパー | `zai` |

    <Accordion title="ファミリービルダーを支える SDK の接続面">
      各ファミリービルダーは、同じパッケージからエクスポートされる低レベルの公開ヘルパーを組み合わせて構成されています。プロバイダーが共通パターンから外れる必要がある場合に利用できます。

      - `openclaw/plugin-sdk/provider-model-shared` - `ProviderReplayFamily`、`buildProviderReplayFamilyHooks(...)`、および未加工のリプレイビルダー（`buildOpenAICompatibleReplayPolicy`、`buildAnthropicReplayPolicyForModel`、`buildGoogleGeminiReplayPolicy`、`buildHybridAnthropicOrOpenAIReplayPolicy`）。Gemini リプレイヘルパー（`sanitizeGoogleGeminiReplayHistory`、`resolveTaggedReasoningOutputMode`）と、エンドポイント/モデルヘルパー（`resolveProviderEndpoint`、`normalizeProviderId`、`normalizeGooglePreviewModelId`）もエクスポートします。
      - `openclaw/plugin-sdk/provider-stream` - `ProviderStreamFamily`、`buildProviderStreamFamilyHooks(...)`、`composeProviderStreamWrappers(...)` に加え、共有 OpenAI/Codex ラッパー（`createOpenAIAttributionHeadersWrapper`、`createOpenAIFastModeWrapper`、`createOpenAIServiceTierWrapper`、`createOpenAIResponsesContextManagementWrapper`、`createCodexNativeWebSearchWrapper`）、DeepSeek V4 OpenAI 互換ラッパー（`createDeepSeekV4OpenAICompatibleThinkingWrapper`）、Anthropic Messages thinking プリフィルのクリーンアップ（`createAnthropicThinkingPrefillPayloadWrapper`）、プレーンテキストのツール呼び出し互換機能（`createPlainTextToolCallCompatWrapper`）、共有プロキシ/プロバイダーラッパー（`createOpenRouterWrapper`、`createToolStreamWrapper`、`createMinimaxFastModeWrapper`）。
      - `openclaw/plugin-sdk/provider-stream-shared` - ホットなプロバイダーパス向けの軽量なペイロードおよびイベントラッパー。`createOpenAICompatibleCompletionsThinkingOffWrapper`、`createPayloadPatchStreamWrapper`、`createPlainTextToolCallCompatWrapper`、`normalizeOpenAICompatibleReasoningPayload(...)`、`setQwenChatTemplateThinking(...)` を含みます。
      - `openclaw/plugin-sdk/provider-tools` - `ProviderToolCompatFamily`、`buildProviderToolCompatFamilyHooks("deepseek" | "gemini" | "openai")`、および基盤となるプロバイダースキーマヘルパー。

      Gemini ファミリーのプロバイダーでは、reasoning 出力モードを
      トランスポートと一致させてください。Google Gemini API に直接接続するプロバイダーは、
      `native` reasoning 出力を使用する必要があります。これにより OpenClaw は、
      `<think>` / `<final>` プロンプトディレクティブを追加せずに、
      ネイティブな thought パートを処理できます。最終的な JSON/テキスト応答を解析する、
      テキスト専用の Gemini CLI 形式バックエンドでは、共有の
      `google-gemini` タグ付き契約を維持できます。

      一部のストリームヘルパーは、意図的にプロバイダー内に留められています。`@openclaw/anthropic-provider` は、`wrapAnthropicProviderStream`、`resolveAnthropicBetas`、`resolveAnthropicFastMode`、`resolveAnthropicServiceTier`、および低レベルの Anthropic ラッパービルダーを、独自の公開 `api.ts` / `contract-api.ts` 接続面に保持しています。これらは Claude OAuth ベータ処理と `context1m` ゲーティングをエンコードするためです。同様に、xAI Plugin はネイティブ xAI Responses の整形を独自の `wrapStreamFn` に保持しています（`/fast` エイリアス、デフォルトの `tool_stream`、未対応の厳格ツールのクリーンアップ、xAI 固有の reasoning ペイロード削除）。

      同じパッケージルートパターンは、`@openclaw/openai-provider`（プロバイダービルダー、デフォルトモデルヘルパー、リアルタイムプロバイダービルダー）と `@openclaw/openrouter-provider`（プロバイダービルダーおよびオンボーディング/config ヘルパー）の基盤にもなっています。
    </Accordion>

    <Tabs>
      <Tab title="トークン交換">
        各推論呼び出しの前にトークン交換が必要なプロバイダー向けです。

        ```typescript
        prepareRuntimeAuth: async (ctx) => {
          const exchanged = await exchangeToken(ctx.apiKey);
          return {
            apiKey: exchanged.token,
            baseUrl: exchanged.baseUrl,
            expiresAt: exchanged.expiresAt,
          };
        },
        ```
      </Tab>
      <Tab title="カスタムヘッダー">
        カスタムリクエストヘッダーやリクエスト本文の変更が必要なプロバイダー向けです。

        ```typescript
        // wrapStreamFn は ctx.streamFn から派生した StreamFn を返す
        wrapStreamFn: (ctx) => {
          if (!ctx.streamFn) return undefined;
          const inner = ctx.streamFn;
          return async (params) => {
            params.headers = {
              ...params.headers,
              "X-Acme-Version": "2",
            };
            return inner(params);
          };
        },
        ```
      </Tab>
      <Tab title="ネイティブトランスポートの識別情報">
        汎用 HTTP または WebSocket トランスポートで、
        ネイティブのリクエスト/セッションヘッダーやメタデータが必要なプロバイダー向けです。

        ```typescript
        resolveTransportTurnState: (ctx) => ({
          headers: {
            "x-request-id": ctx.turnId,
          },
          metadata: {
            session_id: ctx.sessionId ?? "",
            turn_id: ctx.turnId,
          },
        }),
        resolveWebSocketSessionPolicy: (ctx) => ({
          headers: {
            "x-session-id": ctx.sessionId ?? "",
          },
          degradeCooldownMs: 60_000,
        }),
        ```
      </Tab>
      <Tab title="使用量と請求">
        使用量/請求データを公開するプロバイダー向けです。

        ```typescript
        resolveUsageAuth: async (ctx) => {
          const auth = await ctx.resolveOAuthToken();
          return auth ? { token: auth.token } : null;
        },
        fetchUsageSnapshot: async (ctx) => {
          return await fetchAcmeUsage(ctx.token, ctx.timeoutMs);
        },
        ```

        `resolveUsageAuth` には 3 つの結果があります。
        プロバイダーに使用量/請求用の資格情報がある場合は
        `{ token, accountId?, subscriptionType?, rateLimitTier? }` を返します（省略可能なフィールドは、
        解決済みプロファイルから `fetchUsageSnapshot` へ、
        秘密情報ではないプランメタデータを渡します）。
        プロバイダーが使用量認証を明確に処理したものの、
        使用可能な使用量トークンがなく、OpenClaw が汎用の
        API キー/OAuth フォールバックをスキップする必要がある場合にのみ
        `{ handled: true }` を返します。プロバイダーがリクエストを
        処理せず、OpenClaw が汎用フォールバックを続行すべき場合は、
        `null` または `undefined` を返します。

        `contracts.usageProviders` でプロバイダー ID を宣言します。このマニフェスト契約と
        **両方**のフックが存在すると、OpenClaw は無関係なプロバイダー Plugin を
        読み込まずに、そのプロバイダーを使用量収集へ自動的に含めます。
        コアの許可リストを更新する必要はありません。
        `fetchUsageSnapshot` は、共有のプロバイダー中立形式を返します。

        - `plan`: プロバイダーが報告するサブスクリプションまたはキーのラベル
        - `windows`: 使用済み割合で表した、リセット可能なクォータ期間
        - `billing`: 型付きの `balance`、`spend`、または `budget` エントリ。`unit` には、
          ISO 通貨または `credits` のようなプロバイダー単位を指定できます
        - `summary`: これらの構造化フィールドに収まらない、簡潔なプロバイダー固有のコンテキスト

        通貨の意味は厳密に維持してください。上流の契約でそう定められていない限り、
        プロバイダーのクレジットは USD ではありません。
        `fetchUsageSnapshot` のみを実装する Plugin は、明示的/合成的な呼び出し元では
        引き続き利用できますが、OpenClaw が使用量用の資格情報を解決できないため、
        自動検出されません。
      </Tab>
    </Tabs>

    <Accordion title="共通のプロバイダーフック">
      OpenClaw は、モデル/プロバイダー Plugin に対して、おおよそ次の順序でフックを呼び出します。
      ほとんどのプロバイダーが使用するのは 2〜3 個だけです。これは完全な
      `ProviderPlugin` 契約ではありません。現在の正確なフック一覧と
      フォールバックに関する注記については、[内部構造：プロバイダーランタイム
      フック](/ja-JP/plugins/architecture-internals#provider-runtime-hooks)を参照してください。
      `ProviderPlugin.capabilities` や `suppressBuiltInModel` など、
      OpenClaw が呼び出さなくなった互換性専用のプロバイダーフィールドは、
      ここには記載していません。

      | フック | 使用する場面 |
      | --- | --- |
      | `catalog` | モデルカタログまたはベース URL のデフォルト |
      | `applyConfigDefaults` | config の実体化時に適用する、プロバイダー所有のグローバルデフォルト |
      | `normalizeModelId` | 参照前に行うレガシー/プレビューモデル ID エイリアスのクリーンアップ |
      | `normalizeTransport` | 汎用モデルの組み立て前に行う、プロバイダーファミリーの `api` / `baseUrl` クリーンアップ |
      | `normalizeConfig` | `models.providers.<id>` config の正規化 |
      | `applyNativeStreamingUsageCompat` | config プロバイダー向けネイティブストリーミング使用量の互換書き換え |
      | `resolveConfigApiKey` | プロバイダー所有の環境マーカー認証の解決 |
      | `resolveSyntheticAuth` | ローカル/セルフホストまたは config ベースの合成認証 |
      | `resolveExternalAuthProfiles` | CLI/アプリ管理の資格情報向けに、プロバイダー所有の外部認証プロファイルを重ね合わせる |
      | `shouldDeferSyntheticProfileAuth` | 環境/config 認証より下位に合成保存プロファイルのプレースホルダーを配置 |
      | `resolveDynamicModel` | 任意の上流モデル ID を受け入れる |
      | `prepareDynamicModel` | 解決前の非同期メタデータ取得 |
      | `normalizeResolvedModel` | ランナーの前でのトランスポート書き換え |
      | `normalizeToolSchemas` | 登録前に行う、プロバイダー所有のツールスキーマのクリーンアップ |
      | `inspectToolSchemas` | プロバイダー所有のツールスキーマ診断 |
      | `resolveReasoningOutputMode` | タグ付き reasoning 出力とネイティブ reasoning 出力の契約 |
      | `prepareExtraParams` | デフォルトのリクエストパラメーター |
      | `createStreamFn` | 完全にカスタムな StreamFn トランスポート |
      | `wrapStreamFn` | 通常のストリームパスでのカスタムヘッダー/本文ラッパー |
      | `resolveTransportTurnState` | ターンごとのネイティブヘッダー/メタデータ |
      | `resolveWebSocketSessionPolicy` | ネイティブ WS セッションヘッダー/クールダウン |
      | `formatApiKey` | カスタムランタイムトークン形式 |
      | `refreshOAuth` | カスタム OAuth 更新 |
      | `buildAuthDoctorHint` | 認証修復のガイダンス |
      | `matchesContextOverflowError` | プロバイダー所有のオーバーフロー検出 |
      | `classifyFailoverReason` | プロバイダー所有のレート制限/過負荷の分類 |
      | `isCacheTtlEligible` | プロンプトキャッシュ TTL のゲーティング |
      | `buildMissingAuthMessage` | 認証情報不足時のカスタムヒント |
      | `augmentModelCatalog` | 合成的な前方互換行（非推奨 - `registerModelCatalogProvider` を推奨） |
      | `resolveThinkingProfile` | モデル固有の `/think` オプションセット |
      | `isBinaryThinking` | バイナリ thinking のオン/オフ互換性（非推奨 - `resolveThinkingProfile` を推奨） |
      | `supportsXHighThinking` | `xhigh` reasoning サポートの互換性（非推奨 - `resolveThinkingProfile` を推奨） |
      | `resolveDefaultThinkingLevel` | デフォルトの `/think` ポリシー互換性（非推奨 - `resolveThinkingProfile` を推奨） |
      | `isModernModelRef` | ライブ/スモークテスト用モデルの照合 |
      | `prepareRuntimeAuth` | 推論前のトークン交換 |
      | `resolveUsageAuth` | カスタム使用量資格情報の解析 |
      | `fetchUsageSnapshot` | カスタム使用量エンドポイント |
      | `createEmbeddingProvider` | メモリ/検索向けのプロバイダー所有埋め込みアダプター |
      | `buildReplayPolicy` | カスタムトランスクリプトのリプレイ/Compaction ポリシー |
      | `sanitizeReplayHistory` | 汎用クリーンアップ後のプロバイダー固有リプレイ書き換え |
      | `validateReplayTurns` | 組み込みランナーの前での厳格なリプレイターン検証 |
      | `onModelSelected` | 選択後のコールバック（例：テレメトリー） |

      ランタイムのフォールバックに関する注記：

      - `normalizeConfig` は、プロバイダー ID ごとに所有する Plugin を 1 つ解決し（まずバンドル済みプロバイダー、次に一致したランタイム Plugin）、そのフックのみを呼び出します。他のプロバイダーを横断するスキャンはありません。Google 独自の `normalizeConfig` フックが `google` / `google-vertex` / `google-antigravity` の設定エントリを正規化します。これは独立したコアのフォールバックではありません。
      - `resolveConfigApiKey` は、公開されている場合にプロバイダーフックを使用します。Amazon Bedrock は AWS 環境マーカーの解決をそのプロバイダー Plugin 内に保持します。ランタイム認証自体は、`auth: "aws-sdk"` で設定されている場合も AWS SDK のデフォルトチェーンを使用します。
      - `resolveThinkingProfile(ctx)` は、選択された `provider`、`modelId`、任意のマージ済み `reasoning` カタログヒント、および任意のマージ済みモデル `compat` 情報を受け取ります。`compat` は、プロバイダーの思考 UI／プロファイルを選択するためにのみ使用してください。
      - `resolveSystemPromptContribution` を使用すると、プロバイダーはモデルファミリー向けにキャッシュを考慮したシステムプロンプトのガイダンスを注入できます。動作が 1 つのプロバイダー／モデルファミリーに属し、安定部分と動的部分のキャッシュ分割を維持すべき場合は、従来の Plugin 全体を対象とする `before_prompt_build` フックよりもこちらを優先してください。

    </Accordion>

  </Step>

  <Step title="追加機能を加える（任意）">
    ### ステップ 5：追加機能を加える

    プロバイダー Plugin は、テキスト推論に加えて、埋め込み、音声、リアルタイム文字起こし、
    リアルタイム音声、メディア理解、画像生成、動画生成、
    Web 取得、Web 検索を登録できます。OpenClaw はこれを
    **ハイブリッド機能** Plugin として分類します。企業 Plugin に推奨されるパターン
    （ベンダーごとに 1 つの Plugin）です。以下を参照してください：
    [内部構造：機能の所有権](/ja-JP/plugins/architecture#capability-ownership-model)。

    既存の `api.registerProvider(...)` 呼び出しとともに、各機能を `register(api)`
    内で登録します。必要なタブのみを選択してください：

    <Tabs>
      <Tab title="音声（TTS）">
        ```typescript
        import {
          assertOkOrThrowProviderError,
          postJsonRequest,
        } from "openclaw/plugin-sdk/provider-http";

        api.registerSpeechProvider({
          id: "acme-ai",
          label: "Acme Speech",
          defaultTimeoutMs: 120_000,
          isConfigured: ({ config }) => Boolean(config.messages?.tts),
          synthesize: async (req) => {
            const { response, release } = await postJsonRequest({
              url: "https://api.example.com/v1/speech",
              headers: new Headers({ "Content-Type": "application/json" }),
              body: { text: req.text },
              timeoutMs: req.timeoutMs,
              fetchFn: fetch,
              auditContext: "acme speech",
            });
            try {
              await assertOkOrThrowProviderError(response, "Acme Speech API error");
              return {
                audioBuffer: Buffer.from(await response.arrayBuffer()),
                outputFormat: "mp3",
                fileExtension: ".mp3",
                voiceCompatible: false,
              };
            } finally {
              await release();
            }
          },
        });
        ```

        プロバイダーの HTTP エラーには `assertOkOrThrowProviderError(...)` を使用してください。これにより
        Plugin 間で、上限付きのエラー本文読み取り、JSON エラー解析、および
        リクエスト ID のサフィックスを共有できます。
      </Tab>
      <Tab title="リアルタイム文字起こし">
        `createRealtimeTranscriptionWebSocketSession(...)` を優先してください。共有
        ヘルパーがプロキシの取得、再接続バックオフ、切断時のフラッシュ、準備完了
        ハンドシェイク、音声のキューイング、および切断イベントの診断を処理します。Plugin は
        アップストリームイベントをマッピングするだけです。

        ```typescript
        api.registerRealtimeTranscriptionProvider({
          id: "acme-ai",
          label: "Acme Realtime Transcription",
          isConfigured: () => true,
          createSession: (req) => {
            const apiKey = String(req.providerConfig.apiKey ?? "");
            return createRealtimeTranscriptionWebSocketSession({
              providerId: "acme-ai",
              callbacks: req,
              url: "wss://api.example.com/v1/realtime-transcription",
              headers: { Authorization: `Bearer ${apiKey}` },
              onMessage: (event, transport) => {
                if (event.type === "session.created") {
                  transport.sendJson({ type: "session.update" });
                  transport.markReady();
                  return;
                }
                if (event.type === "transcript.final") {
                  req.onTranscript?.(event.text);
                }
              },
              sendAudio: (audio, transport) => {
                transport.sendJson({
                  type: "audio.append",
                  audio: audio.toString("base64"),
                });
              },
              onClose: (transport) => {
                transport.sendJson({ type: "audio.end" });
              },
            });
          },
        });
        ```

        マルチパート音声を POST するバッチ STT プロバイダーは、
        `openclaw/plugin-sdk/provider-http` の
        `buildAudioTranscriptionFormData(...)` を使用してください。このヘルパーは、互換性のある文字起こし API のために
        M4A 形式のファイル名を必要とする AAC アップロードを含め、アップロード
        ファイル名を正規化します。
      </Tab>
      <Tab title="リアルタイム音声">
        ```typescript
        api.registerRealtimeVoiceProvider({
          id: "acme-ai",
          label: "Acme Realtime Voice",
          capabilities: {
            transports: ["gateway-relay"],
            inputAudioFormats: [{ encoding: "pcm16", sampleRateHz: 24000, channels: 1 }],
            outputAudioFormats: [{ encoding: "pcm16", sampleRateHz: 24000, channels: 1 }],
            supportsBargeIn: true,
            handlesInputAudioBargeIn: true,
            supportsToolCalls: true,
          },
          isConfigured: ({ providerConfig }) => Boolean(providerConfig.apiKey),
          createBridge: (req) => ({
            // プロバイダーが 1 回の呼び出しに対して複数のツール応答を受け付ける場合にのみ設定します。
            // たとえば、即時の「処理中」応答の後に
            // 最終結果を返す場合です。
            supportsToolResultContinuation: false,
            connect: async () => {},
            sendAudio: () => {},
            setMediaTimestamp: () => {},
            handleBargeIn: () => {},
            submitToolResult: () => {},
            acknowledgeMark: () => {},
            close: () => {},
            isConnected: () => true,
          }),
        });
        ```

        `talk.catalog` が有効なモード、トランスポート、音声形式、および機能フラグをブラウザーとネイティブの Talk
        クライアントに公開できるように、`capabilities` を宣言してください。トランスポートが人間による
        アシスタント再生の中断を検出でき、プロバイダーがアクティブな音声応答の
        切り詰めまたはクリアをサポートする場合は、`handleBargeIn` を実装してください。
        `submitToolResult` は、同期送信用に `void` を返すか、プロバイダー
        ブリッジが公開できる非同期完了境界として `Promise<void>` を返せます。
        Gateway リレーセッションは、最終結果を確認するかリンクされた実行をクリアする前に
        その Promise を待機します。送信に失敗した場合は拒否してください。
        プロバイダーが `options.suppressResponse` に対応できない場合は、
        `supportsToolResultSuppression: false` を設定してください。これにより OpenClaw は、
        内部の強制コンサルト結果およびキャンセル結果に対する抑制を回避し、応答を暗黙的に開始する代わりに
        抑制された結果を直接要求する処理を拒否します。
        `createRealtimeVoiceBridgeSession` の利用側も同様に、`onToolCall` から
        Promise を返せます。同期的なスローと Promise の拒否は、セッションの
        `onError` コールバックにルーティングされます。
        プロバイダーの VAD が `onClearAudio("barge-in")` を呼び出して
        中断を確認する場合にのみ、`handlesInputAudioBargeIn` を設定してください。このフラグを省略する
        プロバイダーでは、OpenClaw のローカル入力音声フォールバック検出が使用されます。
      </Tab>
      <Tab title="メディア理解">
        ```typescript
        api.registerMediaUnderstandingProvider({
          id: "acme-ai",
          capabilities: ["image", "audio"],
          describeImage: async (req) => ({ text: "A photo of..." }),
          transcribeAudio: async (req) => ({ text: "Transcript..." }),
        });
        ```

        意図的に認証情報を必要としないローカルまたはセルフホスト型のメディアプロバイダーは、
        `resolveAuth` を公開して `kind: "none"` を返せます。
        明示的にオプトインしていないプロバイダーについては、OpenClaw は引き続き通常の認証ゲートを
        維持します。既存のプロバイダーは引き続き `req.apiKey` を読み取れますが、
        新しいプロバイダーでは `req.auth` を優先してください。

        ```typescript
        api.registerMediaUnderstandingProvider({
          id: "local-audio",
          capabilities: ["audio"],
          resolveAuth: () => ({
            kind: "none",
            source: "local-audio plugin no-auth",
          }),
          transcribeAudio: async (req) => ({ text: "Transcript..." }),
        });
        ```
      </Tab>
      <Tab title="埋め込み">
        ```typescript
        api.registerEmbeddingProvider({
          id: "acme-ai",
          defaultModel: "acme-embed",
          transport: "remote",
          authProviderId: "acme-ai",
          create: async ({ model }) => ({
            provider: {
              id: "acme-ai",
              model,
              dimensions: 1536,
              embed: async (input) => {
                const text = typeof input === "string" ? input : input.text;
                return fetchAcmeEmbedding(text);
              },
              embedBatch: async (inputs) =>
                Promise.all(
                  inputs.map((input) =>
                    fetchAcmeEmbedding(typeof input === "string" ? input : input.text),
                  ),
                ),
            },
          }),
        });
        ```

        `contracts.embeddingProviders` で同じ ID を宣言してください。これは、
        メモリ検索を含む、再利用可能なベクトル生成のための一般的な埋め込み契約です。
        `registerMemoryEmbeddingProvider(...)` は、既存のメモリ固有アダプター向けの
        非推奨の互換性機能です。
      </Tab>
      <Tab title="画像と動画の生成">
        画像および動画の機能は、**モード対応**の形式を使用します。画像
        プロバイダーは必須の `generate` および `edit` 機能ブロックを宣言し、
        動画プロバイダーは `generate`、`imageToVideo`、および
        `videoToVideo` を宣言します。`maxInputImages` /
        `maxInputVideos` / `maxDurationSeconds` のようなフラットな集約フィールドだけでは、
        変換モードのサポートや無効化されたモードを明確に通知するには不十分です。音楽生成も
        同じ `generate` / `edit` パターンに従います。

        ```typescript
        api.registerImageGenerationProvider({
          id: "acme-ai",
          label: "Acme 画像",
          capabilities: {
            generate: { maxCount: 4, supportsSize: true },
            edit: { enabled: false },
          },
          generateImage: async (req) => ({ images: [] }),
        });

        api.registerVideoGenerationProvider({
          id: "acme-ai",
          label: "Acme 動画",
          defaultTimeoutMs: 600_000,
          models: ["acme-video", "acme-image-video"],
          capabilities: {
            generate: { maxVideos: 1, maxDurationSeconds: 10, supportsResolution: true },
            imageToVideo: {
              enabled: true,
              maxVideos: 1,
              maxInputImages: 1,
              maxInputImagesByModel: { "acme/reference-to-video": 9 },
              maxDurationSeconds: 5,
            },
            videoToVideo: { enabled: false },
          },
          catalogByModel: {
            "acme-image-video": {
              modes: ["imageToVideo"],
              capabilities: {
                imageToVideo: {
                  enabled: true,
                  maxVideos: 1,
                  maxInputImages: 1,
                  resolutions: ["480P", "720P", "1080P"],
                  supportsResolution: true,
                },
                videoToVideo: { enabled: false },
              },
            },
          },
          generateVideo: async (req) => ({ videos: [] }),
        });
        ```

        `capabilities` は両方のプロバイダータイプで必須です。`edit` と
        動画変換ブロック（`imageToVideo`、`videoToVideo`）には、常に
        明示的な `enabled` フラグが必要です。

        一覧に含まれるモデルの静的なモードまたは機能がプロバイダーのデフォルトと
        異なる場合は、`catalogByModel` を使用します。このメタデータにより、
        プロバイダーコードを呼び出すことなく、`video_generate action=list` とモデルカタログの
        正確性が維持されます。リクエスト時の機能検索と適用は、引き続き
        `resolveModelCapabilities` と `generateVideo` で行います。可能な場合は、
        両方のパスで同じ機能定数を再利用してください。
      </Tab>
      <Tab title="Web フェッチと検索">
        ```typescript
        api.registerWebFetchProvider({
          id: "acme-ai-fetch",
          label: "Acme フェッチ",
          hint: "Acme のレンダリングバックエンドを介してページを取得します。",
          envVars: ["ACME_FETCH_API_KEY"],
          placeholder: "acme-...",
          signupUrl: "https://acme.example.com/fetch",
          credentialPath: "plugins.entries.acme.config.webFetch.apiKey",
          getCredentialValue: (fetchConfig) => fetchConfig?.acme?.apiKey,
          setCredentialValue: (fetchConfigTarget, value) => {
            const acme = (fetchConfigTarget.acme ??= {});
            acme.apiKey = value;
          },
          createTool: () => ({
            description: "Acme フェッチを介してページを取得します。",
            parameters: {},
            execute: async (args) => ({ content: [] }),
          }),
        });

        api.registerWebSearchProvider({
          id: "acme-ai-search",
          label: "Acme 検索",
          hint: "Acme の検索バックエンドを介して Web を検索します。",
          envVars: ["ACME_SEARCH_API_KEY"],
          placeholder: "acme-...",
          signupUrl: "https://acme.example.com/search",
          credentialPath: "plugins.entries.acme.config.webSearch.apiKey",
          getCredentialValue: (searchConfig) => searchConfig?.acme?.apiKey,
          setCredentialValue: (searchConfigTarget, value) => {
            const acme = (searchConfigTarget.acme ??= {});
            acme.apiKey = value;
          },
          createTool: () => ({
            description: "Acme 検索を介して Web を検索します。",
            parameters: {},
            execute: async (args) => ({ content: [] }),
          }),
        });
        ```

        両方のプロバイダータイプは、同じ認証情報の接続形式を共有します。
        `hint`、`envVars`、`placeholder`、`signupUrl`、`credentialPath`、
        `getCredentialValue`、`setCredentialValue`、および `createTool` はすべて
        必須です。
      </Tab>
    </Tabs>

  </Step>

  <Step title="テスト">
    ### ステップ 6：テスト

    ```typescript src/provider.test.ts
    import { describe, it, expect } from "vitest";
    // index.ts または専用ファイルからプロバイダー設定オブジェクトをエクスポートします
    import { acmeProvider } from "./provider.js";

    describe("acme-ai プロバイダー", () => {
      it("動的モデルを解決する", () => {
        const model = acmeProvider.resolveDynamicModel!({
          modelId: "acme-beta-v3",
        } as any);
        expect(model.id).toBe("acme-beta-v3");
        expect(model.provider).toBe("acme-ai");
      });

      it("キーが利用可能な場合にカタログを返す", async () => {
        const result = await acmeProvider.catalog!.run({
          resolveProviderApiKey: () => ({ apiKey: "test-key" }),
        } as any);
        expect(result?.provider?.models).toHaveLength(2);
      });

      it("キーがない場合に null カタログを返す", async () => {
        const result = await acmeProvider.catalog!.run({
          resolveProviderApiKey: () => ({ apiKey: undefined }),
        } as any);
        expect(result).toBeNull();
      });
    });
    ```

  </Step>
</Steps>

## ClawHub に公開する

プロバイダー Plugin は、他の外部コード Plugin と同じ方法で公開します。

```bash
clawhub package publish your-org/your-plugin --dry-run
clawhub package publish your-org/your-plugin
```

`clawhub skill publish <path>` は Plugin パッケージではなく Skills フォルダーを公開するための
別のコマンドです。ここでは使用しないでください。

## ファイル構造

```
<bundled-plugin-root>/acme-ai/
├── package.json              # openclaw.providers メタデータ
├── openclaw.plugin.json      # プロバイダー認証メタデータを含むマニフェスト
├── index.ts                  # definePluginEntry + registerProvider
└── src/
    ├── provider.test.ts      # テスト
    └── usage.ts              # 使用量エンドポイント（任意）
```

## カタログ順序のリファレンス

`catalog.order` は、組み込みプロバイダーに対してカタログがいつマージされるかを
制御します。

| 順序     | タイミング          | ユースケース                                        |
| --------- | ------------- | ----------------------------------------------- |
| `simple`  | 最初のパス    | 単純な API キープロバイダー                         |
| `profile` | 単純なプロバイダーの後  | 認証プロファイルによって制限されるプロバイダー                |
| `paired`  | プロファイルの後 | 関連する複数のエントリを合成する             |
| `late`    | 最後のパス     | 既存のプロバイダーを上書きする（競合時に優先） |

## 次のステップ

- [チャンネル Plugin](/ja-JP/plugins/sdk-channel-plugins) - Plugin がチャンネルも提供する場合
- [SDK ランタイム](/ja-JP/plugins/sdk-runtime) - `api.runtime` ヘルパー（TTS、検索、サブエージェント）
- [SDK の概要](/ja-JP/plugins/sdk-overview) - サブパスインポートの完全なリファレンス
- [Plugin の内部構造](/ja-JP/plugins/architecture-internals#provider-runtime-hooks) - フックの詳細とバンドルされた例

## 関連項目

- [Plugin SDK のセットアップ](/ja-JP/plugins/sdk-setup)
- [Plugin の構築](/ja-JP/plugins/building-plugins)
- [チャンネル Plugin の構築](/ja-JP/plugins/sdk-channel-plugins)
