---
read_when:
    - Plugin のテストを作成しています
    - Plugin SDK のテストユーティリティが必要です
    - バンドル済みプラグインのコントラクトテストについて理解したい場合
sidebarTitle: Testing
summary: OpenClaw Pluginのテストユーティリティとパターン
title: Plugin のテスト
x-i18n:
    generated_at: "2026-07-26T10:26:14Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 9c6c050826dae3cd2c794d50b2dd95e20e6533d838161cce037742ee5fdf7e0e
    source_path: plugins/sdk-testing.md
    workflow: 16
---

OpenClaw Pluginのテストユーティリティ、パターン、lint 適用に関するリファレンスです。

<Tip>
  **テスト例をお探しですか？** ハウツーガイドには実践的なテスト例があります：
  [チャンネル Pluginのテスト](/ja-JP/plugins/sdk-channel-plugins#step-6-test)と
  [プロバイダー Pluginのテスト](/ja-JP/plugins/sdk-provider-plugins#step-6-test)。
</Tip>

## テストユーティリティ

これらのサブパスは、OpenClaw 独自のバンドル済み Pluginテスト向けのリポジトリローカルなソースエントリポイントです。サードパーティ
Plugin向けの公開 `package.json` エクスポートではなく、
Vitest やその他のリポジトリ専用テスト依存関係をインポートする場合があります。

```typescript
import {
  shouldAckReaction,
  removeAckReactionAfterReply,
} from "openclaw/plugin-sdk/channel-feedback";
import { installCommonResolveTargetErrorCases } from "openclaw/plugin-sdk/channel-target-testing";
import { AUTH_PROFILE_RUNTIME_CONTRACT } from "openclaw/plugin-sdk/agent-runtime-test-contracts";
import { createTestPluginApi } from "openclaw/plugin-sdk/plugin-test-api";
import { expectChannelInboundContextContract } from "openclaw/plugin-sdk/channel-contract-testing";
import { createStartAccountContext } from "openclaw/plugin-sdk/channel-test-helpers";
import { describePluginRegistrationContract } from "openclaw/plugin-sdk/plugin-test-contracts";
import { registerSingleProviderPlugin } from "openclaw/plugin-sdk/plugin-test-runtime";
import { describeOpenAIProviderRuntimeContract } from "openclaw/plugin-sdk/provider-test-contracts";
import { getProviderHttpMocks } from "openclaw/plugin-sdk/provider-http-test-mocks";
import { withEnv, withFetchPreconnect, withServer } from "openclaw/plugin-sdk/test-env";
import { isLiveTestEnabled } from "openclaw/plugin-sdk/test-live";
import { createRequestCaptureJsonFetch } from "openclaw/plugin-sdk/test-media-understanding";
import {
  bundledPluginRoot,
  createCliRuntimeCapture,
  typedCases,
} from "openclaw/plugin-sdk/test-fixtures";
import { mockNodeBuiltinModule } from "openclaw/plugin-sdk/test-node-mocks";
```

バンドル済み Pluginのテストには、これらの用途別サブパスを使用してください。以前の
`openclaw/plugin-sdk/testing` バレルはリポジトリローカルであり、リリース済み
パッケージから除外されていたため、削除されました。以前の `openclaw/plugin-sdk/test-utils`
エイリアスも同時に削除されました。`pnpm run lint:plugins:no-extension-test-core-imports`
（`scripts/check-no-extension-test-core-imports.ts`）により、拡張機能のテストでは
上記の用途別テストサブパスが引き続き使用されます。

### 利用可能なエクスポート

| エクスポート                                         | 目的                                                                                                                                     |
| ---------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------- |
| `createTestPluginApi`                                | 直接登録の単体テスト用に最小限の Plugin API モックを構築します。`plugin-sdk/plugin-test-api` からインポートします                             |
| `AUTH_PROFILE_RUNTIME_CONTRACT`                      | ネイティブエージェントランタイムアダプター用の共有認証プロファイル契約フィクスチャ。`plugin-sdk/agent-runtime-test-contracts` からインポートします            |
| `DELIVERY_NO_REPLY_RUNTIME_CONTRACT`                 | ネイティブエージェントランタイムアダプター用の共有配信抑制契約フィクスチャ。`plugin-sdk/agent-runtime-test-contracts` からインポートします    |
| `OUTCOME_FALLBACK_RUNTIME_CONTRACT`                  | ネイティブエージェントランタイムアダプター用の共有フォールバック分類契約フィクスチャ。`plugin-sdk/agent-runtime-test-contracts` からインポートします |
| `createParameterFreeTool`                            | ネイティブランタイム契約テスト用の動的ツールスキーマフィクスチャを構築します。`plugin-sdk/agent-runtime-test-contracts` からインポートします              |
| `expectChannelInboundContextContract`                | チャネルの受信コンテキストの形式を検証します。`plugin-sdk/channel-contract-testing` からインポートします                                                  |
| `installChannelOutboundPayloadContractSuite`         | チャネルの送信ペイロード契約テストケースをインストールします。`plugin-sdk/channel-contract-testing` からインポートします                                       |
| `createStartAccountContext`                          | チャネルアカウントのライフサイクルコンテキストを構築します。`plugin-sdk/channel-test-helpers` からインポートします                                                  |
| `installChannelActionsContractSuite`                 | 汎用チャネルメッセージアクション契約テストケースをインストールします。`plugin-sdk/channel-test-helpers` からインポートします                                     |
| `installChannelSetupContractSuite`                   | 汎用チャネルセットアップ契約テストケースをインストールします。`plugin-sdk/channel-test-helpers` からインポートします                                              |
| `installChannelStatusContractSuite`                  | 汎用チャネルステータス契約テストケースをインストールします。`plugin-sdk/channel-test-helpers` からインポートします                                             |
| `expectDirectoryIds`                                 | ディレクトリ一覧関数から取得したチャネルディレクトリ ID を検証します。`plugin-sdk/channel-test-helpers` からインポートします                               |
| `assertBundledChannelEntries`                        | バンドルされたチャネルのエントリポイントが期待される公開契約を公開していることを検証します。`plugin-sdk/channel-test-helpers` からインポートします                    |
| `formatEnvelopeTimestamp`                            | 決定論的なエンベロープのタイムスタンプを整形します。`plugin-sdk/channel-test-helpers` からインポートします                                                  |
| `expectPairingReplyText`                             | チャネルのペアリング応答テキストを検証し、そのコードを抽出します。`plugin-sdk/channel-test-helpers` からインポートします                                    |
| `describePluginRegistrationContract`                 | Plugin 登録契約チェックをインストールします。`plugin-sdk/plugin-test-contracts` からインポートします                                              |
| `registerSingleProviderPlugin`                       | ローダーのスモークテストでプロバイダー Plugin を 1 つ登録します。`plugin-sdk/plugin-test-runtime` からインポートします                                         |
| `registerProviderPlugin`                             | 1 つの Plugin からすべてのプロバイダー種別をキャプチャします。`plugin-sdk/plugin-test-runtime` からインポートします                                                 |
| `registerProviderPlugins`                            | 複数の Plugin にわたるプロバイダー登録をキャプチャします。`plugin-sdk/plugin-test-runtime` からインポートします                                     |
| `requireRegisteredProvider`                          | プロバイダーコレクションに ID が含まれていることを検証します。`plugin-sdk/plugin-test-runtime` からインポートします                                           |
| `createRuntimeEnv`                                   | モック化された CLI/Plugin ランタイム環境を構築します。`plugin-sdk/plugin-test-runtime` からインポートします                                              |
| `createPluginRuntimeMock`                            | モック化された Plugin ランタイムサーフェスを構築します。`plugin-sdk/plugin-test-runtime` からインポートします                                                      |
| `createPluginSetupWizardStatus`                      | チャネル Plugin 用のセットアップステータスヘルパーを構築します。`plugin-sdk/plugin-test-runtime` からインポートします                                             |
| `createTestWizardPrompter`                           | モック化されたセットアップウィザードのプロンプターを構築します。`plugin-sdk/plugin-test-runtime` からインポートします                                                       |
| `createRuntimeTaskFlow`                              | 分離されたランタイム TaskFlow 状態を作成します。`plugin-sdk/plugin-test-runtime` からインポートします                                                    |
| `runProviderCatalog`                                 | テスト用依存関係を使用してプロバイダーカタログフックを実行します。`plugin-sdk/plugin-test-runtime` からインポートします                                     |
| `resolveProviderWizardOptions`                       | 契約テストでプロバイダーセットアップウィザードの選択肢を解決します。`plugin-sdk/plugin-test-runtime` からインポートします                                    |
| `resolveProviderModelPickerEntries`                  | 契約テストでプロバイダーモデルピッカーのエントリを解決します。`plugin-sdk/plugin-test-runtime` からインポートします                                    |
| `buildProviderPluginMethodChoice`                    | 検証用のプロバイダーウィザード選択肢 ID を構築します。`plugin-sdk/plugin-test-runtime` からインポートします                                            |
| `setProviderWizardProvidersResolverForTest`          | 分離テスト用にプロバイダーウィザードのプロバイダーを注入します。`plugin-sdk/plugin-test-runtime` からインポートします                                        |
| `describeOpenAIProviderRuntimeContract`              | プロバイダーファミリーのランタイム契約チェックをインストールします。`plugin-sdk/provider-test-contracts` からインポートします                                        |
| `expectPassthroughReplayPolicy`                      | プロバイダーのリプレイポリシーがプロバイダー所有のツールとメタデータを通過することを検証します。`plugin-sdk/provider-test-contracts` からインポートします         |
| `runRealtimeSttLiveTest`                             | 共有オーディオフィクスチャを使用してリアルタイム STT プロバイダーのライブテストを実行します。`plugin-sdk/provider-test-contracts` からインポートします                       |
| `normalizeTranscriptForMatch`                        | あいまい検証の前にライブ文字起こし出力を正規化します。`plugin-sdk/provider-test-contracts` からインポートします                               |
| `expectExplicitVideoGenerationCapabilities`          | 動画プロバイダーが明示的な生成モード機能を宣言していることを検証します。`plugin-sdk/provider-test-contracts` からインポートします                   |
| `expectExplicitMusicGenerationCapabilities`          | 音楽プロバイダーが明示的な生成/編集機能を宣言していることを検証します。`plugin-sdk/provider-test-contracts` からインポートします                   |
| `mockSuccessfulDashscopeVideoTask`                   | 成功する DashScope 互換の動画タスク応答をインストールします。`plugin-sdk/provider-test-contracts` からインポートします                          |
| `getProviderHttpMocks`                               | オプトイン方式のプロバイダー HTTP/認証 Vitest モックにアクセスします。`plugin-sdk/provider-http-test-mocks` からインポートします                                         |
| `installProviderHttpMockCleanup`                     | 各テスト後にプロバイダー HTTP/認証モックをリセットします。`plugin-sdk/provider-http-test-mocks` からインポートします                                        |
| `installCommonResolveTargetErrorCases`               | ターゲット解決のエラー処理に関する共有テストケース。`plugin-sdk/channel-target-testing` からインポートします                                  |
| `shouldAckReaction`                                  | チャネルが確認リアクションを追加すべきかどうかを確認します。`plugin-sdk/channel-feedback` からインポートします                                            |
| `removeAckReactionAfterReply`                        | 応答配信後に確認リアクションを削除します。`plugin-sdk/channel-feedback` からインポートします                                                      |
| `createTestRegistry`                                 | チャネル Plugin レジストリフィクスチャを構築します。`plugin-sdk/plugin-test-runtime` または `plugin-sdk/channel-test-helpers` からインポートします               |
| `createEmptyPluginRegistry`                          | 空の Plugin レジストリフィクスチャを構築します。`plugin-sdk/plugin-test-runtime` または `plugin-sdk/channel-test-helpers` からインポートします                |
| `setActivePluginRegistry`                            | Plugin ランタイムテスト用のレジストリフィクスチャをインストールします。`plugin-sdk/plugin-test-runtime` または `plugin-sdk/channel-test-helpers` からインポートします   |
| `createRequestCaptureJsonFetch`                      | メディアヘルパーテストで JSON フェッチリクエストをキャプチャします。`plugin-sdk/test-media-understanding` からインポートします                                     |
| `isLiveTestEnabled`                                  | オプトイン方式のライブプロバイダーテストを制御します。`plugin-sdk/test-live` からインポートします                                                                      |
| `collectProviderApiKeys`                             | ライブプロバイダーテスト用の認証情報を検出します。`plugin-sdk/test-live-auth` からインポートします                                                    |
| `parseProviderModelMap`                              | 音楽/動画のライブテスト用モデルオーバーライドを解析します。`plugin-sdk/test-media-generation` からインポートします                                              |
| `withServer`                                         | 使い捨てのローカル HTTP サーバーに対してテストを実行します。`plugin-sdk/test-env` からインポートします                                                      |
| `createMockIncomingRequest`                          | 最小限の受信 HTTP リクエストオブジェクトを構築します。`plugin-sdk/test-env` からインポートします                                                          |
| `withFetchPreconnect`                                | プリコネクトフックをインストールした状態でフェッチテストを実行します。`plugin-sdk/test-env` からインポートします                                                       |
| `withEnv` / `withEnvAsync`                           | 環境変数を一時的にパッチします。`plugin-sdk/test-env` からインポートします                                                               |
| `createTempHomeEnv` / `withTempHome` / `withTempDir` | 分離されたファイルシステムテストフィクスチャを作成します。`plugin-sdk/test-env` からインポートします                                                              |
| `createMockServerResponse`                           | 最小限の HTTP サーバー応答モックを作成します。`plugin-sdk/test-env` からインポートします                                                            |
| `createProviderUsageFetch`                           | プロバイダー使用量フェッチフィクスチャを構築します。`plugin-sdk/test-env` からインポートします                                                                   |
| `useFrozenTime` / `useRealTime`                      | 時間依存テスト用にタイマーを固定し、復元します。`plugin-sdk/test-env` からインポートします                                                    |
| `createCliRuntimeCapture`                            | テストで CLI ランタイム出力をキャプチャします。`plugin-sdk/test-fixtures` からインポートします                                                              |
| `importFreshModule`                                  | モジュールキャッシュを回避するため、新しいクエリトークン付きで ESM モジュールをインポートします。`plugin-sdk/test-fixtures` からインポートします                             |
| `bundledPluginRoot` / `bundledPluginFile`            | バンドルされた Plugin のソースまたは dist フィクスチャパスを解決します。`plugin-sdk/test-fixtures` からインポートします                                              |
| `mockNodeBuiltinModule`                              | 対象を限定した Node 組み込み Vitest モックをインストールします。`plugin-sdk/test-node-mocks` からインポートします                                                       |
| `createSandboxTestContext`                           | サンドボックステストコンテキストを構築します。`plugin-sdk/test-fixtures` からインポートします                                                                      |
| `writeSkill`                                         | Skills フィクスチャを書き込みます。`plugin-sdk/test-fixtures` からインポートします                                                                             |
| `makeAgentAssistantMessage`                          | エージェントのトランスクリプトメッセージフィクスチャを構築します。`plugin-sdk/test-fixtures` からインポートします                                                          |
| `peekSystemEvents` / `resetSystemEventsForTest`      | システムイベントフィクスチャを検査してリセットします。`plugin-sdk/test-fixtures` からインポートします                                                          |
| `sanitizeTerminalText`                               | 検証用にターミナル出力をサニタイズします。`plugin-sdk/test-fixtures` からインポートします                                                          |
| `countLines` / `hasBalancedFences`                   | チャンク分割出力の形状を検証します。`plugin-sdk/test-fixtures` からインポートします                                                                     |
| `typedCases`                                         | テーブル駆動テスト用にリテラル型を保持します。`plugin-sdk/test-fixtures` からインポートします                                                    |

バンドル Plugin のコントラクトスイートでも、テスト専用のレジストリ、マニフェスト、公開アーティファクト、およびランタイムフィクスチャのヘルパーとして、これらの SDK テスト用サブパスを使用します。
バンドルされた OpenClaw インベントリに依存するコア専用スイートは、代わりに
`src/plugins/contracts` 配下に置きます。

### 型

目的別のテスト用サブパスでは、テストファイルで役立つ型も再エクスポートされます。

```typescript
import type {
  ChannelAccountSnapshot,
  ChannelGatewayContext,
} from "openclaw/plugin-sdk/channel-contract";
import type { OpenClawConfig } from "openclaw/plugin-sdk/config-contracts";
import type { MockFn, PluginRuntime, RuntimeEnv } from "openclaw/plugin-sdk/plugin-test-runtime";
```

## テスト対象の解決

チャネルの対象解決に標準のエラーケースを追加するには、
`installCommonResolveTargetErrorCases` を使用します。

```typescript
import { describe } from "vitest";
import { installCommonResolveTargetErrorCases } from "openclaw/plugin-sdk/channel-target-testing";

describe("my-channel の対象解決", () => {
  installCommonResolveTargetErrorCases({
    resolveTarget: ({ to, mode, allowFrom }) => {
      // チャネルの対象解決ロジック
      return myChannelResolveTarget({ to, mode, allowFrom });
    },
    implicitAllowFrom: ["user1", "user2"],
  });

  // チャネル固有のテストケースを追加
  it("@username の対象を解決する", () => {
    // ...
  });
});
```

## テストパターン

### 登録コントラクトのテスト

手書きの `api` モックを `register(api)` に渡す単体テストでは、
OpenClaw のローダー受け入れゲートは検証されません。Plugin が依存する各登録サーフェスについて、ローダーを介したスモークテストを少なくとも 1 つ追加してください。特に、フックやメモリのような排他的機能では重要です。

必要なメタデータがない場合や、Plugin が所有していない機能 API を呼び出した場合、実際のローダーでは Plugin の登録に失敗します。たとえば、
`api.registerHook(...)` にはフック名が必要であり、
`api.registerMemoryCapability(...)` では Plugin のマニフェストまたはエクスポートされた
エントリで `kind: "memory"` を宣言する必要があります。

### ランタイム設定アクセスのテスト

`openclaw/plugin-sdk/plugin-test-runtime` の共有 Plugin ランタイムモックを優先してください。
そのランタイム設定ヘルパーは、現在のスナップショット API と変更 API をモデル化しています。

### チャネル Plugin の単体テスト

```typescript
import { describe, it, expect, vi } from "vitest";

describe("my-channel Plugin", () => {
  it("設定からアカウントを解決する", () => {
    const cfg = {
      channels: {
        "my-channel": {
          token: "test-token",
          allowFrom: ["user1"],
        },
      },
    };

    const account = myPlugin.setup.resolveAccount(cfg, undefined);
    expect(account.token).toBe("test-token");
  });

  it("シークレットを実体化せずにアカウントを検査する", () => {
    const cfg = {
      channels: {
        "my-channel": { token: "test-token" },
      },
    };

    const inspection = myPlugin.setup.inspectAccount(cfg, undefined);
    expect(inspection.configured).toBe(true);
    expect(inspection.tokenStatus).toBe("available");
    // トークン値は公開されない
    expect(inspection).not.toHaveProperty("token");
  });
});
```

### プロバイダー Plugin の単体テスト

```typescript
import { describe, it, expect } from "vitest";

describe("my-provider Plugin", () => {
  it("動的モデルを解決する", () => {
    const model = myProvider.resolveDynamicModel({
      modelId: "custom-model-v2",
      // ... コンテキスト
    });

    expect(model.id).toBe("custom-model-v2");
    expect(model.provider).toBe("my-provider");
    expect(model.api).toBe("openai-completions");
  });

  it("API キーが利用可能な場合にカタログを返す", async () => {
    const result = await myProvider.catalog.run({
      resolveProviderApiKey: () => ({ apiKey: "test-key" }),
      // ... コンテキスト
    });

    expect(result?.provider?.models).toHaveLength(2);
  });
});
```

### Plugin ランタイムのモック

`createPluginRuntimeStore` を使用するコードでは、テスト内でランタイムをモックします。

```typescript
import { createPluginRuntimeStore } from "openclaw/plugin-sdk/runtime-store";
import type { PluginRuntime } from "openclaw/plugin-sdk/runtime-store";

const store = createPluginRuntimeStore<PluginRuntime>({
  pluginId: "test-plugin",
  errorMessage: "テストランタイムが設定されていません",
});

// テストのセットアップ時
const mockRuntime = {
  agent: {
    resolveAgentDir: vi.fn().mockReturnValue("/tmp/agent"),
    // ... その他のモック
  },
  config: {
    current: vi.fn(() => ({}) as const),
    mutateConfigFile: vi.fn(),
    replaceConfigFile: vi.fn(),
  },
  // ... その他の名前空間
} as unknown as PluginRuntime;

store.setRuntime(mockRuntime);

// テスト後
store.clearRuntime();
```

### インスタンス単位のスタブを使用したテスト

プロトタイプの変更よりも、インスタンス単位のスタブを優先してください。

```typescript
// 推奨: インスタンス単位のスタブ
const client = new MyChannelClient();
client.sendMessage = vi.fn().mockResolvedValue({ id: "msg-1" });

// 非推奨: プロトタイプの変更
// MyChannelClient.prototype.sendMessage = vi.fn();
```

## コントラクトテスト（リポジトリ内 Plugin）

バンドル Plugin には、登録の所有権を検証するコントラクトテストがあります。

```bash
pnpm test src/plugins/contracts/
```

これらのテストでは以下を検証します。

- どの Plugin がどのプロバイダーを登録するか
- どの Plugin がどの音声プロバイダーを登録するか
- 登録形式の正しさ
- ランタイムコントラクトへの準拠

### スコープを限定したテストの実行

特定の Plugin の場合:

```bash
pnpm test <bundled-plugin-root>/my-channel/
```

コントラクトテストのみの場合:

```bash
pnpm test src/plugins/contracts/shape.contract.test.ts
pnpm test src/plugins/contracts/auth-choice.contract.test.ts
pnpm test src/plugins/contracts/runtime-seams.contract.test.ts
```

## lint の適用（リポジトリ内 Plugin）

`scripts/run-additional-boundary-checks.mjs` は CI で一連の `lint:plugins:*`
インポート境界チェックを実行します。それぞれローカルで個別に実行することもできます。

| コマンド                                                        | 適用する規則                                                                                     |
| -------------------------------------------------------------- | -------------------------------------------------------------------------------------------- |
| `pnpm run lint:plugins:no-monolithic-plugin-sdk-entry-imports` | バンドル Plugin は、モノリシックな `openclaw/plugin-sdk` ルートバレルをインポートできません。              |
| `pnpm run lint:plugins:no-extension-src-imports`               | 本番用拡張ファイルは、リポジトリの `src/**` ツリーを直接インポートできません（`../../src/...`）。  |
| `pnpm run lint:plugins:no-extension-test-core-imports`         | 拡張機能のテストファイルは、削除された SDK テストエイリアスや、その他のコア専用テストヘルパーをインポートできません。 |

外部 Plugin はこれらの lint ルールの対象ではありませんが、同じパターンに従うことを推奨します。

## テスト設定

OpenClaw は、参考情報として V8 カバレッジをレポートする Vitest 4 を使用します。Plugin のテストでは以下を実行します。

```bash
# すべてのテストを実行
pnpm test

# 特定の Plugin のテストを実行
pnpm test <bundled-plugin-root>/my-channel/src/channel.test.ts

# 特定のテスト名フィルターを指定して実行
pnpm test <bundled-plugin-root>/my-channel/ -t "resolves account"

# カバレッジを有効にして実行
pnpm test:coverage
```

ローカル実行でメモリ不足が発生する場合:

```bash
OPENCLAW_VITEST_MAX_WORKERS=1 pnpm test
```

## 関連項目

- [SDK の概要](/ja-JP/plugins/sdk-overview) -- インポート規則
- [SDK チャネル Plugin](/ja-JP/plugins/sdk-channel-plugins) -- チャネル Plugin インターフェース
- [SDK プロバイダー Plugin](/ja-JP/plugins/sdk-provider-plugins) -- プロバイダー Plugin のフック
- [Plugin の構築](/ja-JP/plugins/building-plugins) -- はじめにガイド
