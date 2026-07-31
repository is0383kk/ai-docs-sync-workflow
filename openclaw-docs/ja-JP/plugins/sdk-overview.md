---
read_when:
    - どの SDK サブパスからインポートするかを把握しておく必要があります
    - OpenClawPluginApi のすべての登録メソッドに関するリファレンスが必要な場合
    - 特定の SDK エクスポートを検索しています
sidebarTitle: Plugin SDK overview
summary: インポートマップ、登録 API リファレンス、SDK アーキテクチャ
title: Plugin SDK の概要
x-i18n:
    generated_at: "2026-07-26T09:13:26Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 4f490aa8670c57cfc1a635fb1f5d9950fa1cabdb3d45abbc2295da796edcd52e
    source_path: plugins/sdk-overview.md
    workflow: 16
---

Plugin SDK は、Plugin とコア間の型付きコントラクトです。このページでは、
**何をインポートするか**、および**何を登録できるか**を説明します。

<Note>
  このページは、OpenClaw 内で `openclaw/plugin-sdk/*` を使用する
  Plugin 作成者向けです。Gateway を介してエージェントを実行する外部アプリ、
  スクリプト、ダッシュボード、CI ジョブ、IDE 拡張機能については、代わりに
  [外部アプリ向け Gateway 連携](/ja-JP/gateway/external-apps)を使用してください。
</Note>

<Tip>
代わりにハウツーガイドをお探しですか？まずは [Plugin の構築](/ja-JP/plugins/building-plugins)をご覧ください。チャンネルには[チャンネル Plugin](/ja-JP/plugins/sdk-channel-plugins)、モデルプロバイダーには[プロバイダー Plugin](/ja-JP/plugins/sdk-provider-plugins)、ローカル AI CLI バックエンドには[CLI バックエンド Plugin](/ja-JP/plugins/cli-backend-plugins)、ネイティブエージェント実行機能には[エージェントハーネス Plugin](/ja-JP/plugins/sdk-agent-harness)、ツールまたはライフサイクルフックには[Plugin フック](/ja-JP/plugins/hooks)を使用してください。
</Tip>

## インポート規則

必ず特定のサブパスからインポートしてください。

```typescript
import { definePluginEntry } from "openclaw/plugin-sdk/plugin-entry";
import { defineChannelPluginEntry } from "openclaw/plugin-sdk/channel-core";
```

各サブパスは、小さく自己完結したモジュールです。これにより、起動が高速に保たれ、
循環依存関係の問題を防止できます。チャンネル固有のエントリ／ビルドヘルパーには、
`openclaw/plugin-sdk/channel-core` を優先し、より広範な包括的サーフェスと
`buildChannelConfigSchema` などの共有ヘルパーには
`openclaw/plugin-sdk/core` を使用してください。

チャンネル設定では、チャンネルが所有する JSON Schema を
`openclaw.plugin.json#channelConfigs` を通じて公開してください。`plugin-sdk/channel-config-schema`
サブパスは、共有スキーマプリミティブと汎用ビルダー用です。OpenClaw の
同梱 Plugin は、維持されている同梱チャンネルスキーマに
`plugin-sdk/bundled-channel-config-schema` を使用します。この同梱スキーマのサブパスは、新しい
Plugin のためのパターンではありません。

<Warning>
  プロバイダー名またはチャンネル名を冠した便利な接続面（たとえば
  `openclaw/plugin-sdk/slack`、`.../discord`、`.../signal`、`.../whatsapp`）を
  インポートしないでください。同梱 Plugin は、独自の `api.ts` /
  `runtime-api.ts` バレル内で汎用 SDK サブパスを組み合わせます。コアの利用側は、
  これらの Plugin ローカルのバレルを使用するか、必要性が真にチャンネル横断的な場合に
  限定された汎用 SDK コントラクトを追加してください。

所有者による使用が追跡されている場合、少数の同梱 Plugin 用ヘルパー接続面が、
生成されたエクスポートマップに引き続き表示されます。これらは同梱 Plugin の
保守専用であり、新しいサードパーティ Plugin に推奨されるインポートパスではありません。

`openclaw/plugin-sdk/discord` と `openclaw/plugin-sdk/telegram-account` も、
追跡対象の所有者による使用のため、非推奨の互換性ファサードとして維持されています。
これらのインポートパスを新しい Plugin にコピーしないでください。代わりに、
注入されたランタイムヘルパーと汎用チャンネル SDK サブパスを使用してください。
</Warning>

## サブパスリファレンス

Plugin SDK は、領域（Plugin エントリ、チャンネル、プロバイダー、認証、
ランタイム、ケイパビリティ、メモリ、および予約済みの同梱 Plugin ヘルパー）別に
グループ化された限定的なサブパスのセットとして公開されています。グループ化され、
リンクされた完全なカタログについては、[Plugin SDK サブパス](/ja-JP/plugins/sdk-subpaths)を
参照してください。

コンパイラのエントリポイント一覧は
`scripts/lib/plugin-sdk-entrypoints.json` にあります。型付きの公開エクスポートでは、
`scripts/lib/plugin-sdk-private-local-only-subpaths.json` に記載された内部サブパスが除外されます。
この一覧にある本番用エントリは、個別に公開される公式 Plugin 向けに
JavaScript のみのホストランタイムエクスポートを維持しますが、テスト専用エントリは
引き続きエクスポートされません。公開エクスポート数を監査するには、
`pnpm plugin-sdk:surface` を実行してください。十分に古く、同梱拡張機能の本番コードで
使用されていない非推奨の公開サブパスは
`scripts/lib/plugin-sdk-deprecated-public-subpaths.json` で追跡され、広範な非推奨再エクスポートバレルは
`scripts/lib/plugin-sdk-deprecated-barrel-subpaths.json` で追跡されます。

## 登録 API

`register(api)` コールバックは、次のメソッドを持つ
`OpenClawPluginApi` オブジェクトを受け取ります。

セッションに外部チームチャットサーフェスを提供する Plugin は、
`openclaw/plugin-sdk/session-discussion` からエクスポートされるプロセス全体で単一のプロバイダーを
登録できます。その `info({ sessionKey })` メソッドは、ディスカッションが利用不可、
オープン可能、またはすでにオープン済みのいずれであるかを報告します。
`open({ sessionKey })` はディスカッションを作成または解決し、その埋め込み URL と
外部 URL を返します。別のプロバイダーを登録すると、現在のプロバイダーが置き換えられます。

### ケイパビリティの登録

| メソッド                                           | 登録内容                                                                                                                         |
| ------------------------------------------------ | ----------------------------------------------------------------------------------------------------------------------------------------- |
| `api.registerProvider(...)`                      | テキスト推論（LLM）                                                                                                                      |
| `api.registerWorkerProvider(...)`                | クラウドワーカーのライフサイクルリース                                                                                                             |
| `api.registerModelCatalogProvider(...)`          | テキストおよびメディア生成用のモデルカタログ行                                                                                          |
| `api.registerAgentHarness(...)`                  | [試験的](/ja-JP/plugins/sdk-agent-harness)ネイティブエージェント実行機能（Codex、Copilot）                                                         |
| `api.registerCliBackend(...)`                    | ローカル CLI 推論バックエンド                                                                                                               |
| `api.registerChannel(...)`                       | メッセージングチャンネル                                                                                                                         |
| `api.registerEmbeddingProvider(...)`             | 再利用可能なベクトル埋め込みプロバイダー                                                                                                        |
| `api.registerSpeechProvider(...)`                | テキスト読み上げ／STT 合成                                                                                                            |
| `api.registerRealtimeTranscriptionProvider(...)` | ストリーミングリアルタイム文字起こし                                                                                                          |
| `api.registerRealtimeVoiceProvider(...)`         | 双方向リアルタイム音声セッション                                                                                                            |
| `api.registerMediaUnderstandingProvider(...)`    | 画像／音声／動画分析                                                                                                                |
| `api.registerTranscriptSourceProvider(...)`      | ライブまたはインポートされた会議文字起こしソース。会議 Plugin は `plugin-sdk/transcripts` の `createMeetingTranscriptSourceProvider` を使用可能 |
| `api.registerImageGenerationProvider(...)`       | 画像生成                                                                                                                          |
| `api.registerMusicGenerationProvider(...)`       | 音楽生成                                                                                                                          |
| `api.registerVideoGenerationProvider(...)`       | 動画生成                                                                                                                          |
| `api.registerWebFetchProvider(...)`              | Web 取得／スクレイピングプロバイダー                                                                                                               |
| `api.registerWebSearchProvider(...)`             | Web 検索                                                                                                                                |
| `api.registerCompactionProvider(...)`            | 差し替え可能な文字起こし Compaction バックエンド                                                                                                   |

ワーカープロバイダーは、その ID も `contracts.workerProviders` で宣言する必要があります。
コアは `provision(profile, operationId)` より前に永続的な意図を保存します。プロバイダーは外部リソースの割り当て前に設定を検証し、永続的なプロファイル拒否には `WorkerProviderError` をスローします。操作 ID が繰り返された場合、`provision` は同じリースを引き継ぐ必要があります。
コアは検証済みのプロファイル設定をリースとともに保存し、そのスナップショットを、べき等でなければならない `destroy({ leaseId, profile })` と、`active`、`destroyed`、または `unknown` を返す `inspect({ leaseId, profile })` に渡します。これにより、Gateway の再起動後や名前付きプロファイルの削除後でも、プロバイダーはライフサイクル呼び出しをルーティングできます。SSH エンドポイントは `keyRef` に `SecretRef` を使用し、鍵素材をインライン化してはなりません。また、信頼できるプロビジョニング出力から得た `hostKey` を、ホスト名やコメントを付けず、正確に `algorithm base64` として含めます。コアは `hostKey` を固定し、初回接続から得た鍵を決して信頼しません。動的な `keyRef` を発行するプロバイダーは `resolveSshIdentity({ leaseId, profile, keyRef })` を実装できます。存在する場合、そのリゾルバーが権威ある情報源となり、実装しないプロバイダーは設定済みの汎用シークレットリゾルバーを使用します。
更新可能なリースを持つプロバイダーは、`renew(leaseId)` も実装できます。
`inspect` は、一時的または不確定な障害でスローする必要があります。権威ある不在の場合にのみ `unknown` を返してください。コアは、アクティブなローカルレコードを孤立状態としてマークするか、永続化された破棄リクエスト後の不在を解体完了として扱います。

`api.registerEmbeddingProvider(...)` で登録された埋め込みプロバイダーは、
Plugin マニフェストの `contracts.embeddingProviders` にも記載する必要があります。これは、
再利用可能なベクトル生成のための汎用埋め込みサーフェスです。メモリ検索は、この汎用
プロバイダーサーフェスを利用できます。以前の
`api.registerMemoryEmbeddingProvider(...)` および
`contracts.memoryEmbeddingProviders` 接続面は、既存のメモリ固有プロバイダーが移行する間の
非推奨互換性機能です。

引き続きランタイム `batchEmbed(...)` を公開するメモリ固有プロバイダーは、
ランタイムで `sourceWideBatchEmbed: true` を明示的に設定しない限り、既存の
ファイル単位バッチ処理コントラクトを使用します。このオプトインにより、メモリホストは、
複数の変更済みメモリファイルと有効なソースからのチャンクを、ホストのバッチ上限まで
1 回の `batchEmbed(...)` 呼び出しで送信できます。JSONL リクエストファイルを
アップロードするバッチアダプターは、リクエスト数の上限だけでなく、アップロードサイズの
上限に達する前にもプロバイダージョブを分割する必要があります。プロバイダーは、
`batch.chunks` と同じ順序で、入力チャンクごとに 1 つの埋め込みを返す必要があります。
プロバイダーがファイルローカルのバッチを想定している場合、またはより大きなソース全体の
ジョブで入力順序を維持できない場合は、このフラグを省略してください。

### ツールとコマンド

固定されたツール名を持つ単純なツール専用 Plugin には、
[`defineToolPlugin`](/ja-JP/plugins/tool-plugins)を使用してください。混合 Plugin または
完全に動的なツール登録には、`api.registerTool(...)` を直接使用してください。

| メソッド                                 | 登録内容                                                                                                                        |
| -------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------- |
| `api.registerTool(tool, opts?)`        | エージェントツール（必須または `{ optional: true }`）                                                                                            |
| `api.registerCommand(def)`             | カスタムコマンド（LLM をバイパス）                                                                                                        |
| `api.registerNodeHostCommand(command)` | `openclaw node run` によって処理されるコマンド。オプションの `agentTool` メタデータにより、Node の接続中はエージェントから見えるツールとして公開可能 |

エージェントにコマンド所有の短いルーティングヒントが必要な場合、Plugin コマンドは
`agentPromptGuidance` を設定できます。そのテキストはコマンド自体に関する内容に限定し、
プロバイダーまたは Plugin 固有のポリシーをコアのプロンプトビルダーに追加しないでください。

ガイダンスエントリには、すべてのプロンプトサーフェスに適用される従来形式の文字列、
または構造化エントリを使用できます。

```ts
agentPromptGuidance: [
  "グローバルコマンドのヒント。",
  { text: "これは OpenClaw のメインプロンプトでのみ表示します。", surfaces: ["openclaw_main"] },
];
```

構造化された `surfaces` には、`openclaw_main`、`codex_app_server`、
`cli_backend`、`acp_backend`、または `subagent` を含められます。`pi_main` は
`openclaw_main` の非推奨エイリアスとして残されています。意図的にすべてのサーフェスを対象とするガイダンスでは、`surfaces` を省略してください。空の
`surfaces` 配列は渡さないでください。誤ってスコープが失われた場合に
グローバルなプロンプトテキストにならないよう、拒否されます。

ネイティブ Codex app-server の開発者向け指示は、他のプロンプト
サーフェスよりも厳格です。`codex_app_server` に明示的にスコープ設定されたガイダンスだけが、
その高優先度レーンに昇格します。従来の文字列ガイダンスとスコープ設定されていない構造化
ガイダンスは、互換性のため Codex 以外のプロンプトサーフェスでは引き続き利用できます。

Node ホストコマンドは、Gateway
プロセス内ではなく、接続された Node ホスト上で実行されます。`agentTool` が存在する場合、Node は
Gateway への接続成功後に記述子を公開します。Gateway がその記述子をエージェント実行に公開するのは、その
Node が接続されており、かつ記述子の `command` が Node の
承認済みコマンドサーフェスに含まれている間だけです。危険でないコマンドを
デフォルトの Node コマンド許可リストに追加するには、`agentTool.defaultPlatforms` を設定します。それ以外の場合は、
明示的な `gateway.nodes.commands.allow` または Node 呼び出しポリシーが必要です。`agentTool.name` は
プロバイダーで安全に使用できる形式でなければなりません。先頭を文字にし、文字、数字、
アンダースコア、ハイフンだけを使用し、64 文字以内に収めてください。MCP を基盤とする Node ツールでは、
`agentTool.mcp` メタデータを設定すると、カタログおよびツール検索サーフェスに
リモート MCP サーバー／ツールの識別情報を表示できますが、実行は引き続き
公開された Node コマンドを経由します。

### インフラストラクチャ

| メソッド                                          | 登録するもの                                                      |
| ----------------------------------------------- | ---------------------------------------------------------------------- |
| `api.registerHook(events, handler, opts?)`      | イベントフック                                                             |
| `api.registerHttpRoute(params)`                 | Gateway HTTP エンドポイント                                                  |
| `api.registerGatewayMethod(name, handler)`      | Gateway RPC メソッド                                                     |
| `api.registerGatewayDiscoveryService(service)`  | ローカル Gateway 検出アドバタイザー                                     |
| `api.registerCli(registrar, opts?)`             | CLI サブコマンド                                                         |
| `api.registerNodeCliFeature(registrar, opts?)`  | `openclaw nodes` 配下の Node 機能 CLI                                |
| `api.registerService(service)`                  | バックグラウンドサービス                                                     |
| `api.registerInteractiveHandler(registration)`  | インタラクティブハンドラー                                                    |
| `api.registerAgentToolResultMiddleware(...)`    | ランタイムツール結果ミドルウェア                                         |
| `api.registerMemoryPromptSupplement(builder)`   | メモリに隣接する追加プロンプトセクション                                |
| `api.registerMemoryPromptPreparation(prepare)`  | メモリに隣接するプロンプトセクションの非同期準備                 |
| `api.registerMemoryCorpusSupplement(adapter)`   | 追加のメモリ検索／読み取りコーパス                                     |
| `api.registerHostedMediaResolver(resolver)`     | ブラウザースタイルのホストメディア URL 用リゾルバー                           |
| `api.registerMcpServerConnectionResolver(...)`  | 静的サーバー名用のリクエスター単位 MCP トランスポート（`url`/`headers`） |
| `api.registerTextTransforms(transforms)`        | Plugin が所有するプロンプト／メッセージ互換テキストの書き換え                |
| `api.registerConfigMigration(migrate)`          | Plugin ランタイムのロード前に実行される軽量な設定移行           |
| `api.registerMigrationProvider(provider)`       | `openclaw migrate` 用インポーター                                        |
| `api.registerAutoEnableProbe(probe)`            | この Plugin を自動有効化できる設定プローブ                          |
| `api.registerReload(registration)`              | リロード処理用の再起動／ホット／noop 設定プレフィックスポリシー              |
| `api.registerNodeHostCommand(command)`          | ペアリング済み Node に公開されるコマンドハンドラー                                |
| `api.registerNodeInvokePolicy(policy)`          | Node から呼び出されるコマンドの許可リスト／承認ポリシー                    |
| `api.registerSecurityAuditCollector(collector)` | `openclaw security audit` 用の所見コレクター                       |

#### 応答確認後の Webhook 処理

処理の完了前にリクエストへの応答確認を返す Webhook ルートでは、
切り離された処理を独自の追跡対象アドミッションルートへ移す必要があります。

```typescript
import { runDetachedWebhookWork } from "openclaw/plugin-sdk/webhook-request-guards";

void runDetachedWebhookWork(() => processWebhookEvent(event)).catch((error) => {
  runtime.error?.(`Webhook のディスパッチに失敗しました: ${String(error)}`);
});
```

HTTP リクエストがまだ受け入れられている間に、`runDetachedWebhookWork(...)` を同期的に呼び出してください。
このヘルパーは独立したルートを直ちに予約し、その後、次のマイクロタスクで
コールバックを開始するため、リクエストハンドラーは先に応答確認を書き込めます。
返された Promise はコールバックの結果を引き継ぎます。拒否処理の責任は引き続き呼び出し元にあります。
これにより、応答確認後のキュー処理が受け入れられた状態に保たれ、再起動または一時停止時の
ドレインがその完了を待つようになります。返る前にすべての処理を待機する
ハンドラーには、このヘルパーは必要ありません。

#### リクエスター単位の MCP 接続

MCP サーバーの **識別情報**（名前、ツールフィルター）は、`mcp.servers`、ネイティブ
Plugin の `mcpServers` マニフェストフィールド、またはバンドルマニフェストで静的に保ってください。必要に応じて接続リゾルバーを登録すると、信頼された各
メッセージリクエスターが独自のトランスポートを取得できます。

```ts
api.registerMcpServerConnectionResolver({
  serverName: "user-email",
  resolve: async (ctx) => {
    // ctx.requesterSenderId はホストが信頼する値です。ここで送信者の識別情報を作り出してはいけません。
    const token = await lookupUserToken(ctx.requesterSenderId);
    if (!token) {
      return null; // 現在の実行ではこのサーバーを除外します
    }
    return {
      url: "https://mcp.example.com/email",
      headers: { Authorization: `Bearer ${token}` },
    };
  },
});
```

契約上の注意事項:

- リゾルバーコンテキストが保持するのは、信頼されたホスト識別情報だけです（`requesterSenderId`、
  任意の `agentAccountId` / `messageChannel`）。将来、信頼されたフィールド（たとえば
  Cron／サブエージェントのユーザーコンテキスト）を追加的に加えられます。
- 1 つの Plugin が 1 つのサーバー名を所有します。別の
  Plugin から同じ `serverName` に対する重複した
  `registerMcpServerConnectionResolver` が登録されると、エラー診断とともに拒否されます（最初の登録が優先されます）。したがって、
  接続の所有権が Plugin のロード順序に依存することはありません。
- ツール名は宣言されたサーバーの完全な集合から導出されるため、部分的な解決によって
  リクエスター間またはターン間で安全なサーバー名が変化することはありません。コアは、
  リクエスターごとの異なるエンドポイントが同一のツールスキーマを提供するかどうかを検証しません。
  リゾルバーはすべてのリクエスターを同一の論理サービスに接続する必要があります。そうしないと、
  ツールスキーマ（およびプロンプトキャッシュの安定性）がリクエスターごとに分岐します。
- 信頼された `requesterSenderId` がない実行（Cron、サブエージェント、Heartbeat、パブリック
  Gateway）では、リクエスター単位のサーバーを実体化しません。共有の
  フォールバック接続はありません。
- `resolve` はサーバーごとに 10 秒を上限とします。タイムアウトまたは例外発生時は、
  静的 MCP を失敗させることなく、その実行から該当サーバーを除外します。
- 解決済み接続は、リクエスターごとに最大でも 5 分ごとに再検証されます。
  ローテーション時は新しい認証情報でトランスポートを再構築し、`null` の結果が返された場合は
  接続を無効化します（キャッシュされたランタイムはセッション途中でも破棄されます）。したがって、無効化または
  ローテーションされた認証情報が最大 5 分間使用され続ける可能性があります。
- 解決済みの `headers` がログ記録または永続化されることはありません。コアは認証情報の
  ローテーションを検出するために、エフェメラルなメモリ内キー付きダイジェスト（プロセスローカル HMAC）だけを保持し、
  解決済みのヘッダー／URL 認証情報値をログ／デバッグキャプチャの
  リダクションレジストリに登録します。
- リクエスター単位のサーバーは MCP App ビューを生成しません。ビューは
  リクエスター認証済みの実行より長く存続し、Gateway のビュー境界にはリクエスターの
  識別情報がないため、これらのサーバーではアプリのプレビューをフェイルクローズのままにします。ツール結果には
  影響しません。
- リゾルバーのない静的サーバーは、既存のセッション単位のライフサイクルを維持します。
- **ハーネス配信ルール:** リクエスター単位のサーバーがハーネスネイティブの
  MCP クライアント設定（Codex スレッド `mcp_servers`、CLI `-c mcp_servers=…`、またはその他の
  セッション共有 MCP プロジェクション）に入ることはありません。代わりに、ハーネスはそれらを実行単位の
  ツールとして配信します。
  - 組み込みランナー: セッション MCP ランタイム + バンドルツール（静的 + スコープ付き）。
  - Codex app-server: 
    `materializeRequesterScopedMcpToolsForHarnessRun` を介した動的ツール（スコープ付きのみ。静的
    サーバーは Codex のネイティブ MCP クライアントに残ります）。
- スコープ付きツールの **仕様** は、そのセッションで最初に正常に解決された後は
  セッションを通じて安定するため、共有スレッド型ハーネス（Codex）は
  送信者が変わってもスレッドをローテーションしません。どのリクエスターも解決される前は、
  スコープ付き仕様は公開されません。
- 共有スレッド型ハーネス上の未認証リクエスターにも、公開された
  スコープ付きツールは表示されます。そのいずれかを呼び出すと、そのリクエスターには
  明確な未接続ツールエラーが返されます。OpenClaw が別のリクエスターの認証情報へ
  フォールバックすることはありません。

メモリプロンプト補足ビルダーは、任意の `agentId`、
`agentSessionKey`、および `sandboxed` コンテキストを受け取ります。メモリコーパス補足の `search`
および `get` の呼び出しは、任意の `agentId` および `sandboxed` コンテキストを受け取ります。エージェント所有の
ストレージを持つ Plugin は、登録時に 1 つのグローバルパスを捕捉するのではなく、
呼び出しごとにそのストレージを解決する必要があります。マルチエージェント操作でエージェント ID が必須なのに
存在しない場合は、任意のエージェントを選択せず、フェイルクローズしてください。

プロンプトテキストが非同期の
Plugin 状態に依存する場合は、`registerMemoryPromptPreparation(...)` を使用してください。コールバックは完全なエージェントプロンプトの前に毎回 1 回実行され、
同期メモリプロンプトビルダーと同じツール、エージェント、セッション、およびサンドボックスの
コンテキストを受け取ります。永続化された状態をロードする前に現在のストレージ所有者インスタンスを
検証し、その実行用の行だけを返してください。OpenClaw はそれらの行を固定し、
不変の結果を同期プロンプトアセンブリに渡します。永続化、
アトミック置換、および所有者削除時の削除は、所有する Plugin 内に保持してください。プロンプトビルダーから
ポーリングしたりファイルを読み取ったりしないでください。

Telegram のインタラクティブハンドラーは `{ submitText }` を返すことで、ハンドラーの成功後にテキストを
Telegram の通常の受信エージェントパスへルーティングできます。受信ポリシーがテキストをスキップした場合や
処理が失敗した場合も、OpenClaw はコールバックボタンを維持するため、ユーザーは
妨げとなっていた条件が変化した後に再試行できます。この結果フィールドは
Telegram 固有です。他のチャンネルはそれぞれ独自のインタラクティブ結果契約を維持します。

### ワークフロー Plugin 用のホストフック

ホストフックは、プロバイダー、チャンネル、またはツールを追加するだけでなく、ホストの
ライフサイクルに参加する必要がある Plugin のための SDK 接続面です。これらは
汎用的な契約です。Plan Mode だけでなく、承認ワークフロー、
ワークスペースポリシーゲート、バックグラウンドモニター、セットアップウィザード、UI コンパニオン
Plugin でも使用できます。

| メソッド                                                                               | 所有する契約                                                                                                                                           |
| ------------------------------------------------------------------------------------ | ---------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `api.session.state.registerSessionExtension(...)`                                    | Gateway セッションを通じて投影される、Plugin が所有する JSON 互換のセッション状態                                                                             |
| `api.session.workflow.enqueueNextTurnInjection(...)`                                 | 1 つのセッションについて次のエージェントターンに挿入される、永続的かつ厳密に 1 回だけ適用されるコンテキスト                                                                             |
| `api.registerTrustedToolPolicy(...)`                                                 | ツールパラメータをブロックまたは書き換え可能な、マニフェストで制御される信頼済みの Plugin 実行前ツールポリシー                                                                        |
| `api.registerToolMetadata(...)`                                                      | ツールの実装を変更しない、ツールカタログ表示用メタデータ                                                                                     |
| `api.registerCommand(...)`                                                           | スコープ付き Plugin コマンド。コマンド結果は `continueAgent: true` または `suppressReply: true` を設定可能。Discord のネイティブコマンドは `descriptionLocalizations` をサポート |
| `api.session.controls.registerControlUiDescriptor(...)`                              | セッション、ツール、実行、設定、またはタブの各サーフェス向け Control UI コントリビューション記述子                                                                      |
| `api.lifecycle.registerRuntimeLifecycle(...)`                                        | リセット、削除、再読み込みの各パスで Plugin が所有するランタイムリソースを解放するコールバック                                                                          |
| `api.agent.events.registerAgentEventSubscription(...)`                               | ワークフロー状態およびモニター向けのサニタイズ済みイベント購読                                                                                              |
| `api.runContext.setRunContext(...)` / `getRunContext(...)` / `clearRunContext(...)`  | 実行終了ライフサイクル時に消去される、実行ごとの Plugin 一時状態                                                                                             |
| `api.session.workflow.registerSessionSchedulerJob(...)`                              | Plugin が所有するスケジューラージョブのクリーンアップメタデータ。処理のスケジュールやタスクレコードの作成は行わない                                                            |
| `api.session.workflow.sendSessionAttachment(...)`                                    | バンドル版限定の、ホストを介したアクティブな直接送信セッションルートへのファイル添付配信                                                            |
| `api.session.workflow.scheduleSessionTurn(...)` / `unscheduleSessionTurnsByTag(...)` | バンドル版限定の、Cron を基盤とするスケジュール済みセッションターンとタグベースのクリーンアップ                                                                                    |
| `api.session.controls.registerSessionAction(...)`                                    | クライアントが Gateway を通じてディスパッチできる型付きセッションアクション                                                                                             |

`surface: "tab"` 記述子は、Control UI にサイドバータブを追加します。アクティブな
Plugin のタブ記述子は、Gateway の hello（`controlUiTabs`）でダッシュボードクライアントに
通知されるため、タブは Plugin が有効な間だけ表示されます。
バンドルされた Plugin は、そのタブ向けのファーストクラスのダッシュボードビューを同梱できます。それ以外の
Plugin は `path` を Plugin の HTTP ルート（
`api.registerHttpRoute(...)` を参照）に設定でき、ダッシュボードはそれをサンドボックス化されたフレーム内に表示します。
`icon` はダッシュボードアイコン名のヒント、`group` はサイドバーセクション
（`control` または `agent`）を選択し、`order` は Plugin タブ間の並び順を決定します。また、`requiredScopes`
は、それらのオペレータースコープを持たない接続に対してタブを非表示にします。

Gateway で保護された外部タブの場合、同一 Plugin の
`auth: "gateway"` HTTP ルート配下に記述子 `path` を登録します。認証済みのブートストラップ後、ブラウザーは
その Plugin とルートのルートパスにスコープされた有効期間の短い HttpOnly グラントを取得するため、
Gateway の bearer トークンを URL
や JavaScript にコピーせずにサンドボックス化されたフレームを読み込めます。認証済みの親は、外部タブが
アクティブな間、およびナビゲーション後やブラウザー再開後にマウントする前に、グラントを更新します。また、
マウント前に同じ不透明なサンドボックスからグラントを検査するため、
Cookie をブロックするブラウザーのプライバシーモードでは安全側に失敗し、パネルは利用不可になります。
フレームグラントが受け付けるのは `GET` と `HEAD` のみで、常に
`operator.read` が付与されます。`requiredScopes` はタブの可視性を制御しますが、
Cookie グラントの範囲を拡大することはありません。変更操作は、明示的に Gateway 認証された親サーフェスまたは
bearer サーフェス上にとどまります。外部タブには HTTPS/Tailscale Serve または
ブラウザーが信頼するループバックオリジンが必要です。LAN ホスト上のプレーン HTTP では、
認証できないパネルをマウントする代わりにセキュアコンテキストエラーが表示されます。
サードパーティ Cookie を完全にブロックした場合も、Gateway で保護されたタブは利用できなくなります。
すべてのネイティブ Plugin サーフェスと同様に、フレームはインストール済み
Plugin の信頼境界内にとどまります。OpenClaw は、インストール済み Plugin を相互に
分離されたブラウザーセキュリティプリンシパルとして扱いません。
Cookie グラントでは、ポート境界ではなくブラウザーのホスト名境界が使用されます。
たとえ別のポートであっても、相互に信頼されていないサービスを Gateway のホスト名で共同ホストしないでください。
Plugin 管理の認証を基盤とするタブは、直接 iframe を使用する動作を維持し、
この Gateway グラントを要求も必要ともしません。

```typescript
api.session.controls.registerControlUiDescriptor({
  surface: "tab",
  id: "logbook",
  label: "日誌",
  description: "画面のスナップショットから構築された、タイムライン形式の一日。",
  icon: "sun",
  group: "control",
  requiredScopes: ["operator.write"],
});
```

新しい Plugin コードでは、グループ化された名前空間を使用してください。

- `api.session.state.registerSessionExtension(...)`
- `api.session.workflow.enqueueNextTurnInjection(...)`
- `api.session.workflow.registerSessionSchedulerJob(...)`
- `api.session.workflow.sendSessionAttachment(...)`
- `api.session.workflow.scheduleSessionTurn(...)`
- `api.session.workflow.unscheduleSessionTurnsByTag(...)`
- `api.session.controls.registerSessionAction(...)`
- `api.session.controls.registerControlUiDescriptor(...)`
- `api.agent.events.registerAgentEventSubscription(...)`
- `api.agent.events.emitAgentEvent(...)`
- `api.runContext.setRunContext(...)` / `getRunContext(...)` / `clearRunContext(...)`
- `api.lifecycle.registerRuntimeLifecycle(...)`

同等のフラットなメソッドは、既存の Plugin 向けに非推奨の互換性
エイリアスとして引き続き利用できます。
`api.registerSessionExtension`、`api.enqueueNextTurnInjection`、
`api.registerControlUiDescriptor`、`api.registerRuntimeLifecycle`、
`api.registerAgentEventSubscription`、`api.emitAgentEvent`、
`api.setRunContext`、`api.getRunContext`、`api.clearRunContext`、
`api.registerSessionSchedulerJob`、`api.registerSessionAction`、
`api.sendSessionAttachment`、`api.scheduleSessionTurn`、または
`api.unscheduleSessionTurnsByTag` を直接呼び出す新しい Plugin コードを追加しないでください。

`scheduleSessionTurn(...)` は、Gateway の
Cron スケジューラーをセッション単位で使いやすくしたものです。Cron はタイミングを所有し、
ターンの実行時にバックグラウンドタスクレコードを作成します。Plugin SDK が制約するのは、対象セッション、
Plugin が所有する命名、およびクリーンアップだけです。処理自体に永続的な複数ステップの Task Flow 状態が必要な場合は、
スケジュール済みターン内で `api.runtime.tasks.managedFlows` を使用してください。

これらの契約は意図的に権限を分割しています。

- 外部 Plugin は、セッション拡張、UI 記述子、コマンド、ツール
  メタデータ、次ターンへの挿入、および通常のフックを所有できます。
- 信頼済みツールポリシーは通常の `before_tool_call` フックより前に実行され、
  ホストから信頼されます。バンドルされたポリシーが最初に実行されます。インストール済み Plugin のポリシーには、
  明示的な有効化と、
  `contracts.trustedToolPolicies` 内のローカル ID が必要であり、その後 Plugin の読み込み順に実行されます。ポリシー ID
  は、登録した Plugin のスコープ内に限定されます。
- 予約済みコマンドの所有権はバンドル版に限定されます。外部 Plugin は独自の
  コマンド名またはエイリアスを使用する必要があります。
- `allowPromptInjection=false` は、
  `agent_turn_prepare`、`before_prompt_build`、`heartbeat_prompt_contribution`、
  および `enqueueNextTurnInjection` を含む、プロンプトを変更するフックを無効にします。

Plan 以外のコンシューマーの例:

| Plugin の類型             | 使用するフック                                                                                                                             |
| ---------------------------- | -------------------------------------------------------------------------------------------------------------------------------------- |
| 承認ワークフロー            | セッション拡張、コマンド継続、次ターンへの挿入、UI 記述子                                                            |
| 予算／ワークスペースポリシーゲート | 信頼済みツールポリシー、ツールメタデータ、セッション投影                                                                                 |
| バックグラウンドライフサイクルモニター | ランタイムライフサイクルのクリーンアップ、エージェントイベント購読、セッションスケジューラーの所有権／クリーンアップ、Heartbeat プロンプトへの寄与、UI 記述子 |
| セットアップまたはオンボーディングウィザード   | セッション拡張、スコープ付きコマンド、Control UI 記述子                                                                              |

<Note>
  予約済みのコア管理名前空間（`config.*`、`exec.approvals.*`、`wizard.*`、
  `update.*`）は、Plugin がより狭い Gateway メソッドスコープを割り当てようとしても、
  常に `operator.admin` のままです。Plugin が所有するメソッドには、
  Plugin 固有のプレフィックスを推奨します。
</Note>

<Accordion title="ツール結果ミドルウェアを使用する場面">
  バンドルされた Plugin、および一致するマニフェスト契約を持ち明示的に有効化されたインストール済み Plugin は、
  実行後かつランタイムが結果をモデルに返す前にツール結果を書き換える必要がある場合、
  `api.registerAgentToolResultMiddleware(...)` を使用できます。これは、tokenjuice のような
  非同期出力リデューサー向けの、信頼済みでランタイムに依存しない接続点です。

Plugin は対象とする各ランタイムについて `contracts.agentToolResultMiddleware` を宣言する必要があります。
例: `["openclaw", "codex"]`。その契約を持たない、または明示的に有効化されていないインストール済み Plugin は、
このミドルウェアを登録できません。モデルに渡す直前のツール結果処理を必要としない作業には、
通常の OpenClaw Plugin フックを使用してください。以前の
組み込みランナー専用の拡張ファクトリ登録パスは削除されました。
</Accordion>

### Gateway 探索の登録

`api.registerGatewayDiscoveryService(...)` を使用すると、Plugin はアクティブな
Gateway を mDNS/Bonjour などのローカル探索トランスポートで通知できます。ローカル探索が有効な場合、
OpenClaw は Gateway の起動時にサービスを呼び出し、現在の Gateway ポートと秘密ではない TXT ヒントデータを渡し、
Gateway のシャットダウン時に返された `stop` ハンドラーを呼び出します。

```typescript
api.registerGatewayDiscoveryService({
  id: "my-discovery",
  async advertise(ctx) {
    const handle = await startMyAdvertiser({
      gatewayPort: ctx.gatewayPort,
      tls: ctx.gatewayTlsEnabled,
      displayName: ctx.machineDisplayName,
    });
    return { stop: () => handle.stop() };
  },
});
```

Gateway 探索 Plugin は、通知された TXT 値をシークレットや
認証として扱ってはなりません。探索はルーティングのヒントにすぎず、信頼は引き続き Gateway 認証と TLS ピン留めが
担います。

### CLI 登録メタデータ

`api.registerCli(registrar, opts?)` は、2 種類のコマンドメタデータを受け付けます。

- `commands`: 登録元が所有する明示的なコマンド名
- `descriptors`: CLI ヘルプ、
  ルーティング、および遅延 Plugin CLI 登録に使用される解析時コマンド記述子
- `parentPath`: `["nodes"]` など、ネストされたコマンドグループ向けの
  オプションの親コマンドパス

ペアリング済み Node の機能には、
`api.registerNodeCliFeature(registrar, opts?)` を推奨します。これは
`api.registerCli(..., { parentPath: ["nodes"] })` の小さなラッパーであり、
`openclaw nodes canvas` のようなコマンドが Plugin 所有の Node 機能であることを明示します。

通常のルート CLI パスで Plugin コマンドの遅延読み込みを維持する場合は、
その登録元が公開するすべてのトップレベルコマンドルートを網羅する `descriptors` を指定してください。

```typescript
api.registerCli(
  async ({ program }) => {
    const { registerMatrixCli } = await import("./src/cli.js");
    registerMatrixCli({ program });
  },
  {
    descriptors: [
      {
        name: "matrix",
        description: "Matrix のアカウント、検証、デバイス、プロファイル状態を管理",
        hasSubcommands: true,
      },
    ],
  },
);
```

ネストされたコマンドは、解決済みの親コマンドを `program` として受け取ります。

```typescript
api.registerCli(
  async ({ program }) => {
    const { registerNodesCanvasCommands } = await import("./src/cli.js");
    registerNodesCanvasCommands(program);
  },
  {
    parentPath: ["nodes"],
    descriptors: [
      {
        name: "canvas",
        description: "ペアリングされた Node からキャンバスコンテンツをキャプチャまたはレンダリング",
        hasSubcommands: true,
      },
    ],
  },
);
```

遅延ルート CLI 登録が不要な場合にのみ、`commands` を単独で使用してください。
この即時互換パスは引き続きサポートされますが、解析時の遅延読み込み用に
ディスクリプターで裏付けられたプレースホルダーはインストールされません。

### CLI バックエンドの登録

`api.registerCliBackend(...)` を使用すると、Plugin が `claude-cli` や `my-cli` などの
ローカル AI CLI バックエンドのデフォルト設定を所有できます。

- バックエンドの `id` は、`my-cli/gpt-5` のようなモデル参照のプロバイダープレフィックスになります。
- バックエンドの `config` は正式なコマンドアダプターです。argv、環境、
  パーサー、セッション、画像、信頼性に関する動作は Plugin コードに実装されます。
- ユーザーはモデル参照またはモデルスコープの `agentRuntime.id` を介してバックエンドを選択します。
  `openclaw.json` はアダプターを書き換えません。
- 登録済みの静的フィールドにランタイム対応の
  正規化処理が必要な場合は、`normalizeConfig` を使用します。
- OpenClaw の思考レベルをネイティブの effort
  フラグにマッピングするなど、CLI 方言に属するリクエストスコープの argv 書き換えには、`resolveExecutionArgs` を使用します。このフックは `ctx.executionMode` を受け取ります。一時的な `/btw` 呼び出しに
  バックエンドネイティブの分離フラグを追加するには、`"side-question"` を使用します。通常は常時有効な CLI で、それらのフラグによって
  ネイティブツールを確実に無効化できる場合は、`sideQuestionToolMode: "disabled"` も宣言します。
- バックエンドが所有する起動環境または一時的な
  認証／設定ブリッジには、`prepareExecution` を使用します。その `ctx.contextTokenBudget` は実行用に選択された有効なトークン
  上限であるため、ネイティブ Compaction バックエンドは、プロバイダー固有のコア分岐を設けずに
  独自のしきい値を調整できます。また、バックエンドのステージングで同梱 MCP 設定を拡張する必要がある場合は、
  コアで準備された `ctx.env` も受け取ります。
- 特定の実行ですべてのネイティブツールを無効化できるバックエンドは、
  `nativeToolMode: "selectable"` を宣言できます。制限された呼び出しでは、正確な
  `ctx.toolAvailability.native` リストと正規の
  `ctx.toolAvailability.openClaw` 名が渡されます。
  `toolAvailabilityEnforcement: "execution-args"` を宣言して最終的な新規／再開 argv で契約を適用するか、
  `"prepare-execution"` を宣言してステージングされたポリシーで適用し、
  `toolAvailabilityEnforced: true` を返します。OpenClaw は Cron `toolsAllow` などの
  ランタイム上限のためにネイティブツールを無効化し、宣言された適用パスが不完全な場合は
  フェイルクローズします。

エンドツーエンドの作成ガイドについては、
[CLI バックエンド Plugin](/ja-JP/plugins/cli-backend-plugins)を参照してください。

### 排他的スロット

| メソッド                                     | 登録内容                                                                                                                                                                                  |
| ------------------------------------------ | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `api.registerContextEngine(id, factory)`   | コンテキストエンジン（一度に 1 つのみアクティブ）。ホストがモデル／プロバイダー／モードの診断情報を提供できる場合、ライフサイクルコールバックは `runtimeSettings` を受け取ります。以前の厳格なエンジンでは、このキーを除いて再試行されます。 |
| `api.registerMemoryCapability(capability)` | 統合メモリ機能                                                                                                                                                                          |

### 非推奨のメモリ埋め込みアダプター

| メソッド                                         | 登録内容                              |
| ---------------------------------------------- | ---------------------------------------------- |
| `api.registerMemoryEmbeddingProvider(adapter)` | アクティブな Plugin 用のメモリ埋め込みアダプター |

- `registerMemoryCapability` は排他的なメモリ Plugin API です。
- `registerMemoryCapability` は、ホスト管理のエクスポート用に `publicArtifacts.listArtifacts(...)`
  も公開できます。宣言されたアーティファクトを列挙するコンパニオン Plugin は、対象を絞った公開コンシューマー
  API が提供されるまで、維持されている
  `openclaw/plugin-sdk/memory-host-core` ファサードの `listActiveMemoryPublicArtifacts(...)` を引き続き使用します。別の Plugin の非公開レイアウトに直接アクセスしてはなりません。
- `MemoryFlushPlan.model` は、アクティブなフォールバック
  チェーンを継承せずに、フラッシュターンを `ollama/qwen3:8b` などの正確な `provider/model`
  参照に固定できます。
- `registerMemoryEmbeddingProvider` は非推奨です。新しい埋め込みプロバイダーは、
  `api.registerEmbeddingProvider(...)` と
  `contracts.embeddingProviders` を使用してください。
- 既存のメモリ固有プロバイダーは移行
  期間中も引き続き動作しますが、Plugin の検査では、同梱されていない Plugin に対する
  互換性上の負債として報告されます。

### イベントとライフサイクル

| メソッド                                       | 動作                  |
| -------------------------------------------- | ----------------------------- |
| `api.on(hookName, handler, opts?)`           | 型付きライフサイクルフック          |
| `api.onConversationBindingResolved(handler)` | 会話バインディングコールバック |

例、一般的なフック名、ガードの
セマンティクスについては、[Plugin フック](/ja-JP/plugins/hooks)を参照してください。

### フックの判定セマンティクス

`before_install` は Plugin ランタイムのライフサイクルフックであり、オペレーター向けのインストール
ポリシーサーフェスではありません。許可／ブロックの判定を CLI と Gateway 経由のインストールまたは更新パスの両方に
適用する必要がある場合は、`security.installPolicy` を使用します。

- `before_tool_call`: `{ block: true }` を返すと最終判定になります。いずれかのハンドラーがこれを設定すると、優先度の低いハンドラーはスキップされます。
- `before_tool_call`: `{ block: false }` を返すと、上書きではなく判定なし（`block` の省略と同じ）として扱われます。
- `before_install`: `{ block: true }` を返すと最終判定になります。いずれかのハンドラーがこれを設定すると、優先度の低いハンドラーはスキップされます。
- `before_install`: `{ block: false }` を返すと、上書きではなく判定なし（`block` の省略と同じ）として扱われます。
- `reply_dispatch`: `{ handled: true, ... }` を返すと最終判定になります。いずれかのハンドラーがディスパッチを引き受けると、優先度の低いハンドラーとデフォルトのモデルディスパッチパスはスキップされます。
- `message_sending`: `{ cancel: true }` を返すと最終判定になります。いずれかのハンドラーがこれを設定すると、優先度の低いハンドラーはスキップされます。
- `message_sending`: `{ cancel: false }` を返すと、上書きではなく判定なし（`cancel` の省略と同じ）として扱われます。
- `message_received`: 受信スレッド／トピックのルーティングが必要な場合は、型付きの `threadId` フィールドを使用します。チャネル固有の追加情報には `metadata` を使用します。
- `message_sending`: チャネル固有の `metadata` にフォールバックする前に、型付きの `replyToId`／`threadId` ルーティングフィールドを使用します。
- `gateway_start`: 内部の `gateway:startup` フックに依存せず、Gateway が所有する起動状態には `ctx.config`、`ctx.workspaceDir`、`ctx.getCron?.()` を使用します。この時点では Cron がまだ読み込み中の場合があります。
- `cron_reconciled`: 起動後またはスケジューラーの再読み込み後に、完全な外部 Cron プロジェクションを再構築します。これには `reason` と、`enabled: false` を含む有効な `enabled` 状態が含まれ、`ctx.getCron?.()` は正確に調整されたスケジューラーを返します。永続的なプロジェクション処理には `ctx.abortSignal` を渡します。そのスケジューラーのスナップショットが置き換えられるか、Gateway が閉じると中止されます。
- `cron_changed`: Gateway が所有する Cron のライフサイクル変更を監視します。`scheduled` および `removed` イベントはコミット後の調整ヒントであり、順序付けされた差分ログではありません。スケジュールされたイベントの `event.nextRunAtMs` は、ジョブに次回の起動がない場合は存在しません。削除イベントには、削除されたジョブのスナップショットが引き続き含まれます。

外部ウェイクスケジューラーは `cron_changed` イベントをデバウンスまたは統合し、
その後、`cron_reconciled` が最後に取得したスケジューラーから完全な永続ビューを再読み込みする必要があります。
`cron_changed` コンテキストのスケジューラーを採用しないでください。以前のスケジューラーから切り離された
ヒントが、後の再読み込みと重なる可能性があります。

Gateway の起動時またはスケジューラーの置換時に読み込まれる永続状態の完全スナップショットトリガーとして、
`cron_reconciled` を使用します。Plugin のみのホットリロードでは再実行されません。
監視ハンドラーは並列で実行され、fire-and-forget の
ディスパッチは重なる可能性があるため、コンシューマーはイベントの完了順序に依存してはなりません。
期限チェックと実行の信頼できる情報源は OpenClaw のままにしてください。

永続的な置換、再試行／バックオフ、クリーンな
シャットダウンを備えたシングルフライトアダプターについては、[安全な外部 Cron プロジェクション](/ja-JP/plugins/hooks#safe-external-cron-projection)を参照してください。

### API オブジェクトのフィールド

| フィールド                    | 型                      | 説明                                                                                 |
| ------------------------ | ------------------------- | ------------------------------------------------------------------------------------------- |
| `api.id`                 | `string`                  | Plugin ID                                                                                   |
| `api.name`               | `string`                  | 表示名                                                                                |
| `api.version`            | `string?`                 | Plugin のバージョン（省略可能）                                                                   |
| `api.description`        | `string?`                 | Plugin の説明（省略可能）                                                               |
| `api.source`             | `string`                  | Plugin のソースパス                                                                          |
| `api.rootDir`            | `string?`                 | Plugin のルートディレクトリ（省略可能）                                                            |
| `api.config`             | `OpenClawConfig`          | 現在の設定スナップショット（利用可能な場合は、アクティブなインメモリのランタイムスナップショット）                  |
| `api.pluginConfig`       | `Record<string, unknown>` | `plugins.entries.<id>.config` の Plugin 固有設定                                   |
| `api.runtime`            | `PluginRuntime`           | [ランタイムヘルパー](/ja-JP/plugins/sdk-runtime)                                                     |
| `api.logger`             | `PluginLogger`            | スコープ付きロガー（`debug`、`info`、`warn`、`error`）                                            |
| `api.registrationMode`   | `PluginRegistrationMode`  | 現在の読み込みモード。`"setup-runtime"` は完全なエントリの前にある軽量な起動／セットアップ期間です |
| `api.resolvePath(input)` | `(string) => string`      | Plugin のルートからの相対パスを解決                                                        |

## 内部モジュールの規約

Plugin 内では、内部インポートにローカルのバレルファイルを使用します。

```text
my-plugin/
  api.ts            # 外部コンシューマー向けの公開エクスポート
  runtime-api.ts    # 内部専用のランタイムエクスポート
  index.ts          # Plugin のエントリポイント
  setup-entry.ts    # セットアップ専用の軽量エントリ（省略可能）
```

<Warning>
  本番コードから `openclaw/plugin-sdk/<your-plugin>` を介して自身のプラグインを
  インポートしないでください。内部インポートは `./api.ts` または
  `./runtime-api.ts` を介して行ってください。SDK パスは外部コントラクト専用です。
</Warning>

ファサードで読み込まれるバンドル済みプラグインの公開サーフェス（`api.ts`、`runtime-api.ts`、
`index.ts`、`setup-entry.ts`、および同様の公開エントリファイル）は、OpenClaw がすでに実行中の場合、
アクティブなランタイム設定スナップショットを優先します。ランタイム
スナップショットがまだ存在しない場合は、ディスク上の解決済み設定ファイルにフォールバックします。
パッケージ化されたバンドル済みプラグインのファサードは、OpenClaw のプラグイン
ファサードローダーを介して読み込む必要があります。`dist/extensions/...` から直接インポートすると、
パッケージ化されたインストールがプラグイン所有コードに対して使用するマニフェストと
ランタイムサイドカーのチェックが迂回されます。

プロバイダープラグインでは、ヘルパーが意図的にプロバイダー固有であり、まだ汎用 SDK
サブパスに属さない場合、限定的なプラグインローカルのコントラクトバレルを公開できます。
バンドル済みの例：

- **Anthropic**：Claude のベータヘッダーおよび
  `service_tier` ストリームヘルパー向けの公開 `api.ts` / `contract-api.ts` 境界。
- **`@openclaw/openai-provider`**：`api.ts` はプロバイダービルダー、
  デフォルトモデルヘルパー、リアルタイムプロバイダービルダーをエクスポートします。
- **`@openclaw/openrouter-provider`**：`api.ts` はプロバイダービルダーに加えて、
  オンボーディングおよび設定ヘルパーをエクスポートします。

<Warning>
  拡張機能の本番コードでも `openclaw/plugin-sdk/<other-plugin>`
  のインポートを避ける必要があります。ヘルパーが本当に共有されるものであれば、2 つのプラグインを
  結合するのではなく、`openclaw/plugin-sdk/speech`、`.../provider-model-shared`、または別の
  機能指向サーフェスなど、中立的な SDK サブパスに昇格させてください。
</Warning>

## 関連項目

<CardGroup cols={2}>
  <Card title="エントリポイント" icon="door-open" href="/ja-JP/plugins/sdk-entrypoints">
    `definePluginEntry` および `defineChannelPluginEntry` のオプション。
  </Card>
  <Card title="ランタイムヘルパー" icon="gears" href="/ja-JP/plugins/sdk-runtime">
    `api.runtime` 名前空間の完全なリファレンス。
  </Card>
  <Card title="セットアップと設定" icon="sliders" href="/ja-JP/plugins/sdk-setup">
    パッケージ化、マニフェスト、設定スキーマ。
  </Card>
  <Card title="テスト" icon="vial" href="/ja-JP/plugins/sdk-testing">
    テストユーティリティと lint ルール。
  </Card>
  <Card title="SDK の移行" icon="arrows-turn-right" href="/ja-JP/plugins/sdk-migration">
    非推奨サーフェスからの移行。
  </Card>
  <Card title="プラグイン内部構造" icon="diagram-project" href="/ja-JP/plugins/architecture">
    詳細なアーキテクチャと機能モデル。
  </Card>
</CardGroup>
