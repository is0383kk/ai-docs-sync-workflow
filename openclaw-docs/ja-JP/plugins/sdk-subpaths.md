---
read_when:
    - Plugin のインポートに適した plugin-sdk サブパスの選択
    - バンドル済み Plugin のサブパスとヘルパーサーフェスの監査
summary: Plugin SDK サブパスカタログ：各インポートの配置先（領域別）
title: Plugin SDK のサブパス
x-i18n:
    generated_at: "2026-07-26T10:13:47Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 58df43436d0e26f1ffa1383be47fd108655e57d61cf5534d650a4fa2fb7b364c
    source_path: plugins/sdk-subpaths.md
    workflow: 16
---

Plugin SDK には、限定された公開サブパスと、`openclaw/plugin-sdk/` 配下のリポジトリ専用バンドル
ヘルパーが含まれています。このページでは両方を一覧化し、
private-local のエントリを明示的に示します。境界は次の 3 ファイルで定義されます。

- `scripts/lib/plugin-sdk-entrypoints.json`: ビルドでコンパイルされる、保守対象のエントリポイント一覧。
- `scripts/lib/plugin-sdk-private-local-only-subpaths.json`: 型付きで文書化された SDK から
  除外される内部サブパス。本番用エントリは、別途公開される公式
  Plugin 向けに JavaScript 専用のホストランタイムエクスポートとして引き続き利用できますが、
  テスト専用エントリはエクスポートされません。
- `src/plugin-sdk/entrypoints.ts`: 非推奨
  サブパス、予約済みバンドルヘルパー、サポート対象のバンドルファサード、および
  Plugin 所有の公開サーフェスに関する分類メタデータ。

メンテナーは `pnpm plugin-sdk:surface` で公開エクスポート数を、
`pnpm plugins:boundary-report:summary` で使用中の予約済みヘルパーサブパスを監査します。
未使用の予約済みヘルパーエクスポートは、休眠中の互換性負債として
公開 SDK に残るのではなく、CI レポートを失敗させます。

Plugin 作成ガイドについては、[Plugin SDK の概要](/ja-JP/plugins/sdk-overview)を参照してください。

## Plugin エントリ

| サブパス                        | 主なエクスポート                                                                                                                                                                                             |
| ------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `plugin-sdk/plugin-entry`      | `definePluginEntry`                                                                                                                                                                                     |
| `plugin-sdk/core`              | `defineChannelPluginEntry`, `createChatChannelPlugin`, `createChannelPluginBase`, `defineSetupPluginEntry`, `buildChannelConfigSchema`, `buildJsonChannelConfigSchema`, `resolveTailscalePublishedHost` |
| `plugin-sdk/provider-entry`    | 2026 年 7 月以降は private-local。`defineSingleProviderPluginEntry`                                                                                                                                        |
| `plugin-sdk/migration`         | 2026 年 7 月以降は private-local。`createMigrationItem` などの移行プロバイダー項目ヘルパー、理由定数、項目ステータスマーカー、秘匿化ヘルパー、および `summarizeMigrationItems`                   |
| `plugin-sdk/migration-runtime` | 2026 年 7 月以降は private-local。`copyMigrationFileItem`、`resolvePlannedMigrationTargets`、`withCachedMigrationConfigRuntime`、`writeMigrationReport` などのランタイム移行ヘルパー              |
| `plugin-sdk/health`            | バンドルされたヘルスチェック利用側向けの Doctor ヘルスチェック登録、検出、修復、選択、重大度、および検出事項の型                                                                                |

### 互換性ヘルパーと private-local ヘルパー

後期期間の非推奨サブパスのみが引き続きエクスポートされます。2026 年 7 月のエイリアスと
未使用のサブパスは削除され、バンドル専用ヘルパーは
公開パッケージから削除されて、以下で private-local として示されています。保守対象のリストは
`scripts/lib/plugin-sdk-deprecated-public-subpaths.json` です。CI はバンドルされた
`plugin-sdk/text-runtime` を拒否します。これらは互換性専用であり、`plugin-sdk/zod` は
互換性再エクスポートです。`zod` を `zod` から直接インポートしてください。広範なドメイン
バレル `plugin-sdk/agent-runtime`、`plugin-sdk/channel-lifecycle`、
`plugin-sdk/conversation-runtime`、`plugin-sdk/hook-runtime`、
`plugin-sdk/media-runtime`、`plugin-sdk/plugin-runtime`、および
`plugin-sdk/security-runtime` も同様に、対象を限定した
サブパスを優先するため非推奨です。

OpenClaw の Vitest ベースのテストヘルパーサブパスはリポジトリローカル専用であり、
パッケージからはエクスポートされなくなりました: `agent-runtime-test-contracts`、
`channel-contract-testing`、`channel-target-testing`、`channel-test-helpers`、
`plugin-state-test-runtime`、`plugin-test-api`、`plugin-test-contracts`、
`plugin-test-runtime`、`provider-http-test-mocks`、`provider-test-contracts`、
`reply-payload-testing`、`sqlite-runtime-testing`、`test-env`、`test-fixtures`、
`test-live`、`test-live-auth`、`test-media-generation`、
`test-media-understanding`、`test-node-mocks`、および `testing`。非公開のバンドルヘルパーサーフェス
`ssrf-runtime-internal` と `codex-native-task-runtime` もリポジトリローカル
専用です。

### バンドル Plugin ヘルパーのサブパス

バンドル専用ヘルパーモジュールは、2026 年 7 月の整理以降 private-local です。所有者をまたぐインポートは、パッケージ契約のガードレールによってブロックされます。`src/plugin-sdk/entrypoints.ts` は、引き続き公開されるサポート対象のバンドルファサードを別途追跡します。これらは、汎用契約が
`plugin-sdk/qa-runner-runtime`、`plugin-sdk/telegram-account` を置き換えるまで、対応するバンドル Plugin によって実装される SDK
エントリポイントであり、新規コードでは非推奨です。以下の各行の注記を参照してください。

<AccordionGroup>
  <Accordion title="チャンネルのサブパス">
    | サブパス | 主なエクスポート |
    | --- | --- |
    | `plugin-sdk/channel-core` | `defineChannelPluginEntry`、`defineSetupPluginEntry`、`createChatChannelPlugin`、`createChannelPluginBase`、`createChannelConfigUiHints` |
    | `plugin-sdk/json-schema-runtime` | 2026 年 7 月以降は private-local。Plugin 所有スキーマ向けのキャッシュ済み JSON Schema 検証ヘルパー |
    | `plugin-sdk/channel-setup` | `defineChannelSetupContract`、チャンネル所有のセットアップフィールド／入力型、`createOptionalChannelSetupSurface`、`createOptionalChannelSetupAdapter`、`createOptionalChannelSetupWizard`、さらに `DEFAULT_ACCOUNT_ID`、`createTopLevelChannelDmPolicy`、`setSetupChannelEnabled`、`splitSetupEntries` |
    | `plugin-sdk/setup` | 共有セットアップウィザードヘルパー、セットアップトランスレーター、許可リストプロンプト、セットアップステータスビルダー |
    | `plugin-sdk/setup-runtime` | `defineChannelSetupContract`、`createSetupTranslator`、`createPatchedAccountSetupAdapter`、`createEnvPatchedAccountSetupAdapter`、`createSetupInputPresenceValidator`、`noteChannelLookupFailure`、`noteChannelLookupSummary`、`promptResolvedAllowFrom`、`splitSetupEntries`、`createAllowlistSetupWizardProxy`、`createDelegatedSetupWizardProxy` |
    | `plugin-sdk/setup-tools` | `formatCliCommand`、`detectBinary`、`extractArchive`、`resolveBrewExecutable`、`formatDocsLink`、`CONFIG_DIR` |
    | `plugin-sdk/account-core` | マルチアカウント設定／アクションゲートヘルパー、デフォルトアカウントのフォールバックヘルパー |
    | `plugin-sdk/account-id` | `DEFAULT_ACCOUNT_ID`、アカウント ID 正規化ヘルパー |
    | `plugin-sdk/account-resolution` | アカウント検索およびデフォルトフォールバックヘルパー |
    | `plugin-sdk/account-helpers` | 限定的なアカウント一覧／アカウントアクションヘルパー |
    | `plugin-sdk/access-groups` | 2026 年 7 月以降は private-local。アクセスグループ許可リストの解析および秘匿化されたグループ診断ヘルパー |
    | `plugin-sdk/channel-pairing` | `createChannelPairingController` |
    | `plugin-sdk/channel-reply-pipeline` | 非推奨の互換性ファサード。`plugin-sdk/channel-outbound` を使用してください。 |
    | `plugin-sdk/channel-config-helpers` | `createHybridChannelConfigAdapter`、`resolveChannelDmAccess`、`resolveChannelDmAllowFrom`、`resolveChannelDmPolicy`、`normalizeChannelDmPolicy`、`normalizeLegacyDmAliases` |
    | `plugin-sdk/channel-config-schema` | 共有チャンネル設定スキーマプリミティブに加え、Zod および直接的な JSON/TypeBox ビルダー |
    | `plugin-sdk/bundled-channel-config-schema` | 2026 年 7 月以降は private-local。保守対象のバンドル Plugin 専用の、バンドルされた OpenClaw チャンネル設定スキーマ |
    | `plugin-sdk/chat-channel-ids` | 2026 年 7 月以降は private-local。`BUNDLED_CHAT_CHANNEL_IDS`、`BUNDLED_CHAT_CHANNEL_ENVELOPE_PREFIXES`、`ChatChannelId`。エンベロープ接頭辞付きテキストを独自テーブルへのハードコードなしで認識する必要がある Plugin 向けの、正規のバンドル／公式チャットチャンネル ID とフォーマッターラベル／エイリアス。 |
    | `plugin-sdk/channel-policy` | `resolveChannelGroupRequireMention` |
    | `plugin-sdk/channel-ingress-runtime` | 移行済みのチャンネル受信パス向けの、実験的な高レベルチャンネル受信ランタイムリゾルバー、暗黙メンションポリシーリゾルバー、およびルート情報ビルダー。各 Plugin で有効な許可リスト、コマンド許可リスト、レガシープロジェクションを組み立てるよりも、こちらを優先してください。[チャンネル受信 API](/ja-JP/plugins/sdk-channel-ingress)を参照してください。 |
    | `plugin-sdk/channel-lifecycle` | 非推奨の互換性ファサード。`plugin-sdk/channel-outbound` を使用してください。 |
    | `plugin-sdk/channel-outbound` | メッセージライフサイクル契約に加え、返信パイプラインオプション、受領確認、ライブプレビュー／ストリーミング、ライフサイクルヘルパー、送信元 ID、ペイロード計画、永続的送信、およびメッセージ送信コンテキストヘルパー。[チャンネル送信 API](/ja-JP/plugins/sdk-channel-outbound)を参照してください。 |
    | `plugin-sdk/channel-message` | `plugin-sdk/channel-outbound` の非推奨の互換性エイリアス。 |
    | `plugin-sdk/inbound-envelope` | 共有受信ルートおよびエンベロープビルダーヘルパー |
    | `plugin-sdk/inbound-reply-dispatch` | 非推奨の互換性ファサード。受信ランナーとディスパッチ述語には `plugin-sdk/channel-inbound` を、メッセージ配信ヘルパーには `plugin-sdk/channel-outbound` を使用してください。 |
    | `plugin-sdk/messaging-targets` | 非推奨のターゲット解析エイリアス。`plugin-sdk/channel-targets` を使用してください |
    | `plugin-sdk/outbound-media` | 2026 年 7 月以降は private-local。共有送信メディア読み込みおよびホスト済みメディア状態ヘルパー |
    | `plugin-sdk/poll-runtime` | 2026 年 7 月以降は private-local。限定的な投票正規化ヘルパー |
    | `plugin-sdk/thread-bindings-runtime` | 2026 年 7 月以降は private-local。スレッドバインディングのライフサイクルおよびアダプターヘルパー |
    | `plugin-sdk/agent-media-payload` | レガシー `Media*` ペイロードプロジェクション向けの非推奨の互換性ファサード。順序付けされた情報を `MsgContext.media` / `toInboundMediaFacts(...)` 経由で渡し、ローカルルートポリシーを `plugin-sdk/media-local-roots` からインポートしてください。 |
    | `plugin-sdk/conversation-runtime` | 会話／スレッドバインディング、ペアリング、設定済みバインディングヘルパー向けの非推奨の広範なバレル。`plugin-sdk/thread-bindings-runtime` や `plugin-sdk/session-binding-runtime` など、対象を限定したバインディングサブパスを優先してください |
    | `plugin-sdk/runtime-group-policy` | ランタイムグループポリシー解決ヘルパー |
    | `plugin-sdk/channel-status` | 共有チャンネルステータスのスナップショット／概要ヘルパー |
    | `plugin-sdk/channel-config-primitives` | 限定的なチャンネル設定スキーマプリミティブ |
    | `plugin-sdk/channel-config-writes` | 2026 年 7 月以降は private-local。チャンネル設定書き込み認可ヘルパー |
    | `plugin-sdk/channel-plugin-common` | 共有チャンネル Plugin のプレリュードエクスポート |
    | `plugin-sdk/allowlist-config-edit` | 許可リスト設定の編集／読み取りヘルパー |
    | `plugin-sdk/group-access` | 非推奨のグループアクセス判定ヘルパー。`plugin-sdk/channel-ingress-runtime` の `resolveChannelMessageIngress` を使用してください |
    | `plugin-sdk/direct-dm-guard-policy` | 2026 年 7 月以降は private-local。暗号化前の限定的なダイレクト DM ガードポリシーヘルパー |
    | `plugin-sdk/discord` | 公開済み `@openclaw/discord@2026.3.13` および追跡対象の所有者互換性向けの、非推奨の Discord 互換性ファサード。新しい Plugin では汎用チャンネル SDK サブパスを使用してください |
    | `plugin-sdk/telegram-account` | 追跡対象の所有者互換性向けの、非推奨の Telegram アカウント解決互換性ファサード。新しい Plugin では注入されたランタイムヘルパーまたは汎用チャンネル SDK サブパスを使用してください |
    | `plugin-sdk/interactive-runtime` | セマンティックなメッセージ表示、配信、およびレガシーな対話型返信ヘルパー。[メッセージ表示](/ja-JP/plugins/message-presentation)を参照してください |
    | `plugin-sdk/question-gateway-runtime` | チャンネル操作ハンドラーから、ランタイムが作成した `ask_user` の選択肢を Gateway 経由で解決 |
    | `plugin-sdk/channel-inbound` | イベント分類、コンテキスト構築、書式設定、ルート、デバウンス、メンション照合、メンションポリシー、および受信ログ向けの共有受信ヘルパー |
    | `plugin-sdk/channel-inbound-debounce` | 限定的な受信デバウンスヘルパー |
    | `plugin-sdk/channel-mention-gating` | 2026 年 7 月以降は private-local。広範な受信ランタイムサーフェスを含まない、限定的なメンションポリシー、メンションマーカー、およびメンションテキストヘルパー |
    | `plugin-sdk/channel-streaming` | 非推奨の互換性ファサード。`plugin-sdk/channel-outbound` を使用してください。 |
    | `plugin-sdk/channel-send-result` | 返信結果の型 |
    | `plugin-sdk/channel-actions` | チャンネルメッセージアクションヘルパーに加え、Plugin 互換性のために維持される非推奨のネイティブスキーマヘルパー |
    | `plugin-sdk/channel-route` | 2026 年 7 月以降は private-local。共有ルート正規化、パーサー駆動のターゲット解決、スレッド ID の文字列化、重複排除／コンパクトなルートキー、解析済みターゲット型、およびルート／ターゲット比較ヘルパー |
    | `plugin-sdk/channel-targets` | 2026 年 7 月以降は private-local。ターゲット解析ヘルパー。ルート比較の呼び出し元は `plugin-sdk/channel-route` を使用してください |
    | `plugin-sdk/channel-contract` | チャンネル契約の型 |
    | `plugin-sdk/channel-feedback` | フィードバック／リアクションの配線 |
  </Accordion>

後期期間のチャンネル互換性サブパスは、それぞれの
レジストリ日付までのみ公開されたままです。ダイレクト DM アクセス、返信オプション、ペアリング
パス、チャンネルランタイムの分割サブパスなど、7 月のエイリアスは削除され、バンドル専用ヘルパーは
private-local になりました。

  <Accordion title="プロバイダーのサブパス">
    | サブパス | 主なエクスポート |
    | --- | --- |
    | `plugin-sdk/provider-entry` | 2026年7月以降はプライベートローカル。`defineSingleProviderPluginEntry` |
    | `plugin-sdk/provider-setup` | 2026年7月以降はプライベートローカル。厳選されたローカル／セルフホスト型プロバイダーのセットアップヘルパー |
    | `plugin-sdk/cli-backend` | 2026年7月以降はプライベートローカル。CLI バックエンドのデフォルト値とウォッチドッグ定数 |
    | `plugin-sdk/provider-auth-runtime` | 2026年7月以降はプライベートローカル。OAuth local loopback フロー、トークン交換、認証情報の永続化、API キー解決などのプロバイダー認証ランタイムヘルパー |
    | `plugin-sdk/provider-oauth-runtime` | 2026年7月以降はプライベートローカル。汎用プロバイダー OAuth コールバック型、コールバックページのレンダリング、PKCE／state ヘルパー、認可入力の解析、トークン有効期限ヘルパー、中断ヘルパー |
    | `plugin-sdk/provider-auth-api-key` | 2026年7月以降はプライベートローカル。`upsertApiKeyProfile` などの API キーのオンボーディング／プロファイル書き込みヘルパー |
    | `plugin-sdk/provider-auth-result` | 2026年7月以降はプライベートローカル。標準 OAuth 認証結果ビルダー |
    | `plugin-sdk/provider-env-vars` | 2026年7月以降はプライベートローカル。プロバイダー認証用環境変数の検索ヘルパー |
    | `plugin-sdk/provider-auth` | `createProviderApiKeyAuthMethod`、`ensureApiKeyFromOptionEnvOrPrompt`、`upsertAuthProfile`、`upsertApiKeyProfile`、`writeOAuthCredentials`、OpenAI Codex 認証インポートヘルパー、非推奨の `resolveOpenClawAgentDir` 互換エクスポート |
    | `plugin-sdk/provider-model-shared` | 2026年7月以降はプライベートローカル。`ProviderReplayFamily`、`buildProviderReplayFamilyHooks`、`selectPreferredLocalModelId`、`normalizeModelCompat`、共有リプレイポリシービルダー、プロバイダーエンドポイントヘルパー、共有モデル ID 正規化ヘルパー |
    | `plugin-sdk/provider-catalog-live-runtime` | 2026年7月以降はプライベートローカル。保護された `/models` 形式の検出に使用するライブプロバイダーモデルカタログヘルパー：`buildLiveModelProviderConfig`、`fetchLiveProviderModelRows`、`getCachedLiveProviderModelRows`、`fetchLiveProviderModelIds`、`LiveModelCatalogHttpError`、`clearLiveCatalogCacheForTests`、モデル ID フィルタリング、TTL キャッシュ、静的フォールバック |
    | `plugin-sdk/provider-catalog-runtime` | プロバイダーカタログ拡張ランタイムフックと、コントラクトテスト用のプラグインプロバイダーレジストリ境界 |
    | `plugin-sdk/provider-catalog-shared` | 2026年7月以降はプライベートローカル。`findCatalogTemplate`、`buildSingleProviderApiKeyCatalog`、`buildManifestModelProviderConfig`、`supportsNativeStreamingUsageCompat`、`applyProviderNativeStreamingUsageCompat` |
    | `plugin-sdk/provider-http` | 2026年7月以降はプライベートローカル。汎用プロバイダー HTTP／エンドポイント機能ヘルパー、プロバイダー HTTP エラー、音声文字起こし用マルチパートフォームヘルパー |
    | `plugin-sdk/provider-web-fetch-contract` | 2026年7月以降はプライベートローカル。`enablePluginInConfig` や `WebFetchProviderPlugin` などの限定的なウェブ取得設定／選択コントラクトヘルパー |
    | `plugin-sdk/provider-web-fetch` | 2026年7月以降はプライベートローカル。ウェブ取得プロバイダーの登録／キャッシュヘルパー |
    | `plugin-sdk/provider-web-search-config-contract` | 2026年7月以降はプライベートローカル。プラグイン有効化の配線を必要としないプロバイダー向けの限定的なウェブ検索設定／認証情報ヘルパー |
    | `plugin-sdk/provider-web-search-contract` | 2026年7月以降はプライベートローカル。`createWebSearchProviderContractFields`、`enablePluginInConfig`、`resolveProviderWebSearchPluginConfig`、およびスコープ付き認証情報のセッター／ゲッターなどの限定的なウェブ検索設定／認証情報コントラクトヘルパー |
    | `plugin-sdk/provider-web-search` | 2026年7月以降はプライベートローカル。ウェブ検索プロバイダーの登録／キャッシュ／ランタイムヘルパー |
    | `plugin-sdk/embedding-providers` | 2026年7月以降はプライベートローカル。`EmbeddingProviderAdapter`、`getEmbeddingProvider(...)`、`listEmbeddingProviders(...)` などの汎用埋め込みプロバイダー型と読み取りヘルパー。マニフェストの所有権を強制するため、プラグインは `api.registerEmbeddingProvider(...)` を介してプロバイダーを登録 |
    | `plugin-sdk/provider-tools` | 2026年7月以降はプライベートローカル。`ProviderToolCompatFamily`、`buildProviderToolCompatFamilyHooks`、DeepSeek／Gemini／OpenAI のスキーマクリーンアップと診断 |
    | `plugin-sdk/provider-usage` | 2026年7月以降はプライベートローカル。プロバイダー使用量スナップショット型、共有使用量取得ヘルパー、`fetchClaudeUsage` などのプロバイダー取得関数 |
    | `plugin-sdk/provider-stream` | 2026年7月以降はプライベートローカル。`ProviderStreamFamily`、`buildProviderStreamFamilyHooks`、`composeProviderStreamWrappers`、ストリームラッパー型、プレーンテキストのツール呼び出し互換機能、共有 Anthropic／Google／Kilocode／MiniMax／Moonshot／OpenAI／OpenRouter／Z.AI ラッパーヘルパー |
    | `plugin-sdk/provider-stream-shared` | 2026年7月以降はプライベートローカル。`composeProviderStreamWrappers`、`createOpenAICompatibleCompletionsThinkingOffWrapper`、`createPlainTextToolCallCompatWrapper`、`createPayloadPatchStreamWrapper`、`createToolStreamWrapper`、`normalizeOpenAICompatibleReasoningPayload`、`setQwenChatTemplateThinking`、および Anthropic／DeepSeek／OpenAI 互換ストリームユーティリティを含む公開共有プロバイダーストリームラッパーヘルパー |
    | `plugin-sdk/provider-transport-runtime` | 2026年7月以降はプライベートローカル。保護付き取得、ツール結果テキスト抽出、トランスポートメッセージ変換、書き込み可能なトランスポートイベントストリームなどのネイティブプロバイダートランスポートヘルパー |
    | `plugin-sdk/provider-onboard` | 2026年7月以降はプライベートローカル。オンボーディング設定パッチヘルパー |
    | `plugin-sdk/global-singleton` | 2026年7月以降はプライベートローカル。プロセスローカルのシングルトン／マップ／キャッシュヘルパー |
    | `plugin-sdk/group-activation` | 2026年7月以降はプライベートローカル。限定的なグループ有効化モードとコマンド解析ヘルパー |
  </Accordion>

プロバイダー使用量スナップショットは通常、1つ以上のクォータ `windows` を報告し、それぞれに
ラベル、使用率、任意のリセット時刻が含まれます。リセット可能なクォータ期間ではなく残高または
アカウント状態のテキストを公開するプロバイダーは、パーセンテージを捏造せず、空の `windows`
配列を持つ `summary` を返す必要があります。
OpenClaw はその概要テキストをステータス出力に表示します。使用量エンドポイントが失敗した場合、または
利用可能な使用量データを返さなかった場合にのみ `error` を使用してください。

  <Accordion title="認証とセキュリティのサブパス">
    | サブパス | 主なエクスポート |
    | --- | --- |
    | `plugin-sdk/command-auth` | 非推奨の広範なコマンド認可サーフェス（`resolveControlCommandGate`、動的引数メニューの書式設定を含むコマンドレジストリヘルパー、送信者認可ヘルパー）。チャネルの受信処理／ランタイム認可またはコマンドステータスヘルパーを使用 |
    | `plugin-sdk/command-status` | `buildCommandsMessagePaginated` や `buildHelpMessage` などのコマンド／ヘルプメッセージビルダー |
    | `plugin-sdk/approval-auth-runtime` | 承認者の解決と同一チャットのアクション認証ヘルパー |
    | `plugin-sdk/approval-client-runtime` | ネイティブ実行承認のプロファイル／フィルターヘルパー |
    | `plugin-sdk/approval-delivery-runtime` | ネイティブ承認機能／配信アダプター |
    | `plugin-sdk/approval-gateway-runtime` | 共有承認 Gateway リゾルバー |
    | `plugin-sdk/approval-reference-runtime` | 2026年7月以降はプライベートローカル。トランスポートの制限を受ける承認コールバック向けの決定論的な永続ロケーターヘルパー |
    | `plugin-sdk/approval-handler-adapter-runtime` | 高頻度チャネルエントリーポイント向けの軽量ネイティブ承認アダター読み込みヘルパー |
    | `plugin-sdk/approval-handler-runtime` | より広範な承認ハンドラーのランタイムヘルパー。より限定的なアダプター／Gateway 境界で十分な場合はそちらを優先 |
    | `plugin-sdk/approval-native-runtime` | ネイティブ承認ターゲット、アカウントバインディング、ルートゲート、転送フォールバック、ローカルネイティブ実行プロンプト抑制ヘルパー |
    | `plugin-sdk/approval-reaction-runtime` | 2026年7月以降はプライベートローカル。ハードコードされた承認リアクションバインディング、リアクションプロンプトペイロード、リアクションターゲットストア、リアクションヒントテキストヘルパー、ローカルネイティブ実行プロンプト抑制用の互換エクスポート |
    | `plugin-sdk/approval-reply-runtime` | 実行／プラグイン承認返信ペイロードヘルパー |
    | `plugin-sdk/approval-runtime` | 実行／プラグイン承認ペイロードヘルパー、承認機能ビルダー、承認認証／プロファイルヘルパー、ネイティブ承認ルーティング／ランタイムヘルパー、`formatApprovalDisplayPath` などの構造化承認表示ヘルパー |
    | `plugin-sdk/command-auth-native` | ネイティブコマンド認証、動的引数メニューの書式設定、ネイティブセッションターゲットヘルパー |
    | `plugin-sdk/command-detection` | 共有コマンド検出ヘルパー |
    | `plugin-sdk/command-primitives-runtime` | 高頻度チャネルパス向けの軽量コマンドテキスト述語 |
    | `plugin-sdk/command-surface` | 2026年7月以降はプライベートローカル。コマンド本文の正規化とコマンドサーフェスヘルパー |
    | `plugin-sdk/allow-from` | `formatAllowFromLowercase` |
    | `plugin-sdk/provider-auth-login-flow-runtime` | 2026年7月以降はプライベートローカル。プライベートチャネルおよび Web UI のデバイスコードペアリング向け遅延プロバイダー認証ログインフローヘルパー |
    | `plugin-sdk/channel-secret-runtime` | 非推奨の広範なシークレットコントラクトサーフェス（`collectSimpleChannelFieldAssignments`、`getChannelSurface`、`pushAssignment`、シークレットターゲット型）。以下の限定的なサブパスを優先 |
    | `plugin-sdk/channel-secret-basic-runtime` | TTS 以外のチャネル／プラグインのシークレットサーフェス向けの限定的なシークレットコントラクトエクスポートとターゲットレジストリビルダー |
    | `plugin-sdk/channel-secret-tts-runtime` | 2026年7月以降はプライベートローカル。限定的なネストされたチャネル TTS シークレット割り当てヘルパー |
    | `plugin-sdk/secret-ref-runtime` | シークレットコントラクト／設定解析向けの限定的な SecretRef 型付け、解決、プランターゲットパス検索 |
    | `plugin-sdk/security-runtime` | 信頼、DM ゲート、作成専用書き込み、同期／非同期アトミックファイル置換、同階層の一時書き込み、デバイス間移動フォールバック、プライベートファイルストアヘルパー、シンボリックリンク親ガード、外部コンテンツ、機密テキストの秘匿化、定時間シークレット比較、シークレット収集ヘルパーを含むルート境界付きファイル／パスヘルパー向けの非推奨の広範なバレル。限定的なセキュリティ／SSRF／シークレットサブパスを優先 |
    | `plugin-sdk/ssrf-policy` | ホスト許可リストとプライベートネットワーク SSRF ポリシーヘルパー |
    | `plugin-sdk/ssrf-dispatcher` | 2026年7月以降はプライベートローカル。広範なインフラランタイムサーフェスを含まない限定的な固定ディスパッチャーヘルパー |
    | `plugin-sdk/ssrf-runtime` | 固定ディスパッチャー、SSRF 保護付き取得、SSRF エラー、SSRF ポリシーヘルパー |
    | `plugin-sdk/secret-input` | シークレット入力解析ヘルパー |
    | `plugin-sdk/webhook-ingress` | Webhook リクエスト／ターゲットヘルパーと、生の websocket／本文の型変換 |
    | `plugin-sdk/webhook-request-guards` | リクエスト本文のサイズ／タイムアウトヘルパーと、追跡対象の確認応答後処理用 `runDetachedWebhookWork` |
  </Accordion>

  <Accordion title="Runtime and storage subpaths">
    | サブパス | 主なエクスポート |
    | --- | --- |
    | `plugin-sdk/runtime` | ランタイム、ロギング、バックアップのヘルパー、Plugin インストールパスの警告、プロセスヘルパー |
    | `plugin-sdk/runtime-env` | 限定的なランタイム環境、ロガー、タイムアウト、再試行、バックオフのヘルパー |
    | `plugin-sdk/browser-config` | 2026 年 7 月以降はプライベートローカル。正規化されたプロファイルとデフォルト、CDP URL の解析、ブラウザー制御認証ヘルパーを提供する、サポート対象のブラウザー設定ファサード |
    | `plugin-sdk/agent-harness-task-runtime` | 2026 年 7 月以降はプライベートローカル。ホストが発行したタスクスコープを使用するハーネスベースのエージェント向けの汎用タスクライフサイクルおよび完了配信ヘルパー |
    | `plugin-sdk/codex-mcp-projection` | 2026 年 7 月以降はプライベートローカル。ユーザーの MCP サーバー設定を Codex スレッド設定に投影するために予約されたバンドル Codex ヘルパー。サードパーティ Plugin 用ではありません |
    | `plugin-sdk/codex-native-task-runtime` | ネイティブタスクのミラーリングおよびランタイム配線用のリポジトリローカルなバンドル Codex ヘルパー。パッケージのエクスポートではありません |
    | `plugin-sdk/channel-runtime-context` | 汎用チャネルランタイムコンテキストの登録および検索ヘルパー |
    | `plugin-sdk/matrix` | 古いサードパーティ製チャネルパッケージ向けの非推奨 Matrix 互換性ファサード。新しい Plugin は `plugin-sdk/run-command` を直接インポートしてください |
    | `plugin-sdk/runtime-store` | `createPluginRuntimeStore` |
    | `plugin-sdk/plugin-runtime` | Plugin コマンド、フック、HTTP、インタラクティブヘルパー用の非推奨の広範なバレル。用途を限定した Plugin ランタイムサブパスを推奨します |
    | `plugin-sdk/hook-runtime` | Webhook および内部フックパイプラインヘルパー用の非推奨の広範なバレル。用途を限定したフックおよび Plugin ランタイムサブパスを推奨します |
    | `plugin-sdk/lazy-runtime` | `createLazyRuntimeModule`、`createLazyRuntimeMethod`、`createLazyRuntimeSurface` などの遅延ランタイムインポートおよびバインディングヘルパー |
    | `plugin-sdk/process-runtime` | 2026 年 7 月以降はプライベートローカル。プロセス実行ヘルパー |
    | `plugin-sdk/node-host` | 2026 年 7 月以降はプライベートローカル。Node ホストの実行可能ファイル解決および PTY 再開ヘルパー |
    | `plugin-sdk/cli-runtime` | 2026 年 7 月以降はプライベートローカル。CLI の書式設定、待機、バージョン、引数による呼び出し、遅延コマンドグループの各ヘルパー用の非推奨の広範なバレル。用途を限定した CLI およびランタイムサブパスを推奨します |
    | `plugin-sdk/qa-runner-runtime` | 2026 年 7 月以降はプライベートローカル。CLI コマンドサーフェスを介して Plugin QA シナリオを公開する、サポート対象のファサード |
    | `plugin-sdk/tts-runtime` | 2026 年 7 月以降はプライベートローカル。テキスト読み上げ設定スキーマおよびランタイムヘルパー用のサポート対象ファサード |
    | `plugin-sdk/gateway-method-runtime` | `contracts.gatewayMethodDispatch: ["authenticated-request"]` を宣言する Plugin HTTP ルート用に予約された Gateway メソッドディスパッチヘルパー |
    | `plugin-sdk/gateway-runtime` | Gateway クライアント、イベントループの準備完了時にクライアントを起動するヘルパー、Gateway CLI RPC、Gateway プロトコルエラー、通知された LAN ホストの解決、チャネルステータスのパッチヘルパー |
    | `plugin-sdk/config-contracts` | `OpenClawConfig`、チャネルおよびプロバイダー設定型など、Plugin 設定形状向けの用途を限定した型のみの設定サーフェス |
    | `plugin-sdk/plugin-config-runtime` | ランタイム Plugin 設定ヘルパー用の非推奨の互換性ファサード。新しい Plugin は `api.pluginConfig` と、用途を限定した設定コントラクト、スナップショット、変更ヘルパーを使用します |
    | `plugin-sdk/config-mutation` | `mutateConfigFile`、`replaceConfigFile`、`logConfigUpdated` などのトランザクション対応設定変更ヘルパー |
    | `plugin-sdk/message-tool-delivery-hints` | 2026 年 7 月以降はプライベートローカル。共有メッセージツール配信メタデータのヒント文字列 |
    | `plugin-sdk/runtime-config-snapshot` | `getRuntimeConfig`、`getRuntimeConfigSnapshot`、テスト用スナップショットセッターなどの現在のプロセス設定スナップショットヘルパー |
    | `plugin-sdk/text-autolink-runtime` | 2026 年 7 月以降はプライベートローカル。広範なテキストバレルを使用しないファイル参照の自動リンク検出 |
    | `plugin-sdk/reply-runtime` | 共有の受信および返信ランタイムヘルパー、チャンク分割、ディスパッチ、Heartbeat、返信プランナー |
    | `plugin-sdk/reply-dispatch-runtime` | 用途を限定した返信のディスパッチおよび確定、会話ラベルのヘルパー |
    | `plugin-sdk/reply-history` | 共有の短時間枠返信履歴ヘルパー。新しいメッセージターンコードでは `createChannelHistoryWindow` を使用してください。低レベルのマップヘルパーは、非推奨の互換性エクスポートとしてのみ残されています |
    | `plugin-sdk/reply-reference` | 2026 年 7 月以降はプライベートローカル。`createReplyReferencePlanner` |
    | `plugin-sdk/reply-chunking` | 用途を限定したテキストおよび Markdown チャンク分割ヘルパー |
    | `plugin-sdk/session-store-runtime` | セッションワークフローヘルパー（`getSessionEntry`、`listSessionEntries`、`patchSessionEntry`、`upsertSessionEntry`）、修復およびライフサイクルヘルパー（`deleteSessionEntry`、`cleanupSessionLifecycleArtifacts`、`resolveSessionStoreBackupPaths`）、移行中の `sessionFile` 値用のマーカーヘルパー、セッション ID に基づく直近のユーザーおよびアシスタントのトランスクリプトテキストの上限付き読み取り、セッションストアパスおよびセッションキーヘルパー、更新日時の読み取り。広範な設定書き込みおよびメンテナンスのインポートは含みません |
    | `plugin-sdk/session-transcript-runtime` | 2026 年 7 月以降はプライベートローカル。トランスクリプト ID、上限付きの未加工および可視カーソル、スコープ指定されたターゲット、読み取り、書き込みのヘルパー、可視メッセージエントリの投影、更新の発行、書き込みロック、トランスクリプトメモリのヒットキー |
    | `plugin-sdk/sqlite-runtime` | 2026 年 7 月以降はプライベートローカル。データベースライフサイクル制御を含まない、ファーストパーティランタイム向けの用途を限定した SQLite エージェントスキーマ、パス、トランザクションのヘルパー |
    | `plugin-sdk/cron-store-runtime` | 2026 年 7 月以降はプライベートローカル。Cron ストアのパス、読み込み、保存ヘルパー |
    | `plugin-sdk/state-paths` | 状態および OAuth ディレクトリのパスヘルパー |
    | `plugin-sdk/plugin-state-runtime` | 2026 年 7 月以降はプライベートローカル。Plugin スコープのキー付き状態、BLOB、協調的 SQLite リースのコントラクトに加え、接続 pragma、検証済み WAL メンテナンス、アトミックな STRICT スキーマ移行のヘルパー。リースコールバックは中止シグナルを受け取り、型付きエラーによってタイムアウト、キャンセル、所有権の喪失、無効な入力、ストレージ障害を区別します |
    | `plugin-sdk/routing` | `resolveAgentRoute`、`buildAgentSessionKey`、`resolveDefaultAgentBoundAccountId` などのルート、セッションキー、アカウントのバインディングヘルパー |
    | `plugin-sdk/status-helpers` | 共有のチャネルおよびアカウントステータス概要ヘルパー、ランタイム状態のデフォルト、問題メタデータヘルパー |
    | `plugin-sdk/target-resolver-runtime` | 2026 年 7 月以降はプライベートローカル。共有ターゲットリゾルバーヘルパー |
    | `plugin-sdk/string-normalization-runtime` | 2026 年 7 月以降はプライベートローカル。スラッグおよび文字列の正規化ヘルパー |
    | `plugin-sdk/request-url` | 2026 年 7 月以降はプライベートローカル。fetch または request のような入力から文字列 URL を抽出 |
    | `plugin-sdk/run-command` | 正規化された stdout および stderr の結果を返す時間制限付きコマンドランナー |
    | `plugin-sdk/param-readers` | 共通のツールおよび CLI パラメータリーダー |
    | `plugin-sdk/tool-plugin` | 単純な型付きエージェントツール Plugin を定義し、マニフェスト生成用の静的メタデータを公開 |
    | `plugin-sdk/tool-payload` | 2026 年 7 月以降はプライベートローカル。ツール結果オブジェクトから正規化されたペイロードを抽出 |
    | `plugin-sdk/tool-send` | ツール引数から正規の送信先フィールドを抽出 |
    | `plugin-sdk/sandbox` | 2026 年 7 月以降はプライベートローカル。サンドボックスバックエンド型、SSH および OpenShell コマンドヘルパー。早期失敗する実行コマンドの事前チェックを含みます |
    | `plugin-sdk/temp-path` | 共有の一時ダウンロードパスヘルパーおよびプライベートな安全な一時ワークスペース |
    | `plugin-sdk/logging-core` | サブシステムロガーおよび編集除去ヘルパー |
    | `plugin-sdk/markdown-table-runtime` | 2026 年 7 月以降はプライベートローカル。Markdown テーブルモードおよび変換ヘルパー |
    | `plugin-sdk/model-session-runtime` | `applyModelOverrideToSessionEntry`、`resolveAgentMaxConcurrent` などのモデルおよびセッションのオーバーライドヘルパー |
    | `plugin-sdk/talk-config-runtime` | 2026 年 7 月以降はプライベートローカル。トークプロバイダー設定の解決ヘルパー |
    | `plugin-sdk/json-store` | 小規模な JSON 状態の読み取りおよび書き込みヘルパー |
    | `plugin-sdk/json-unsafe-integers` | 2026 年 7 月以降はプライベートローカル。安全でない整数リテラルを文字列として保持する JSON 解析ヘルパー |
    | `plugin-sdk/file-lock` | 2026 年 7 月以降はプライベートローカル。再入可能なファイルロックヘルパーに加え、確実に古く、変更されていない廃止済みロックサイドカーを Doctor から安全に回収する機能 |
    | `plugin-sdk/persistent-dedupe` | ディスクベースの重複排除キャッシュヘルパー |
    | `plugin-sdk/ingress-effect-once` | 非冪等な受信処理の副作用に対する永続的なクレームおよびコミットガード |
    | `plugin-sdk/acp-runtime` | 2026 年 7 月以降はプライベートローカル。ACP ランタイム、セッション、返信ディスパッチのヘルパー |
    | `plugin-sdk/acp-runtime-backend` | 2026 年 7 月以降はプライベートローカル。起動時に読み込まれる Plugin 向けの軽量 ACP バックエンド登録および返信ディスパッチヘルパー |
    | `plugin-sdk/acp-binding-resolve-runtime` | 2026 年 7 月以降はプライベートローカル。ライフサイクル起動のインポートを伴わない、読み取り専用の ACP バインディング解決 |
    | `plugin-sdk/agent-config-primitives` | 非推奨のエージェントランタイム設定スキーマプリミティブ。メンテナンスされている Plugin 所有のサーフェスからスキーマプリミティブをインポートしてください |
    | `plugin-sdk/boolean-param` | 緩やかな真偽値パラメータリーダー |
    | `plugin-sdk/dangerous-name-runtime` | 2026 年 7 月以降はプライベートローカル。危険な名前の一致解決ヘルパー |
    | `plugin-sdk/device-bootstrap` | `BOOTSTRAP_HANDOFF_OPERATOR_SCOPES` を含むデバイスのブートストラップおよびペアリングトークンヘルパー |
    | `plugin-sdk/extension-shared` | 共有のパッシブチャネル、ステータス、アンビエントプロキシのヘルパープリミティブ |
    | `plugin-sdk/models-provider-runtime` | `/models` コマンドおよびプロバイダー返信ヘルパー |
    | `plugin-sdk/skill-commands-runtime` | Skill コマンド一覧ヘルパー |
    | `plugin-sdk/native-command-registry` | ネイティブコマンドのレジストリ、構築、シリアル化ヘルパー |
    | `plugin-sdk/agent-harness` | 低レベルのエージェントハーネス向けの実験的な信頼済み Plugin サーフェス：ハーネス型、アクティブな実行の誘導および中止ヘルパー、OpenClaw ツールブリッジヘルパー、ランタイムプランのツールポリシーヘルパー、ターミナル結果の分類、ツール進捗の書式設定および詳細ヘルパー、試行結果ユーティリティ |
    | `plugin-sdk/async-lock-runtime` | 2026 年 7 月以降はプライベートローカル。小規模なランタイム状態ファイル向けのプロセスローカルな非同期ロックヘルパー |
    | `plugin-sdk/channel-activity-runtime` | 2026 年 7 月以降はプライベートローカル。チャネルアクティビティのテレメトリヘルパー |
    | `plugin-sdk/concurrency-runtime` | 2026 年 7 月以降はプライベートローカル。上限付きの非同期タスク並行処理ヘルパー |
    | `plugin-sdk/dedupe-runtime` | メモリ内および永続化バックエンド付き重複排除キャッシュヘルパー |
    | `plugin-sdk/delivery-queue-runtime` | 2026 年 7 月以降はプライベートローカル。保留中の送信配信を排出するヘルパー |
    | `plugin-sdk/file-access-runtime` | 2026 年 7 月以降はプライベートローカル。安全なローカルファイルおよびメディアソースのパスヘルパー |
    | `plugin-sdk/heartbeat-runtime` | 2026 年 7 月以降はプライベートローカル。Heartbeat のウェイクアップ、イベント、可視性のヘルパー |
    | `plugin-sdk/expect-runtime` | 2026 年 7 月以降はプライベートローカル。証明可能なランタイム不変条件のための必須値アサーションヘルパー |
    | `plugin-sdk/number-runtime` | 2026 年 7 月以降はプライベートローカル。数値型強制変換ヘルパー |
    | `plugin-sdk/secure-random-runtime` | 2026 年 7 月以降はプライベートローカル。安全なトークンおよび UUID ヘルパー |
    | `plugin-sdk/system-event-runtime` | 2026 年 7 月以降はプライベートローカル。システムイベントキューヘルパー |
    | `plugin-sdk/transport-ready-runtime` | 2026 年 7 月以降はプライベートローカル。トランスポート準備完了の待機ヘルパー |
    | `plugin-sdk/exec-approvals-runtime` | 2026 年 7 月以降はプライベートローカル。広範なインフラランタイムバレルを使用しない実行承認ポリシーファイルヘルパー |
    | `plugin-sdk/infra-runtime` | 非推奨の互換性シム。上記の用途を限定したランタイムサブパスを使用してください |
    | `plugin-sdk/collection-runtime` | 小規模な上限付きキャッシュヘルパー |
    | `plugin-sdk/diagnostic-runtime` | 診断フラグ、イベント、トレースコンテキストのヘルパー |
    | `plugin-sdk/error-runtime` | エラーグラフ、書式設定、不明な値の型強制変換、共有エラー分類ヘルパー、`PlatformMessageNotDispatchedError`、`isApprovalNotFoundError` |
    | `plugin-sdk/fetch-runtime` | 2026 年 7 月以降はプライベートローカル。ラップされた fetch、プロキシ、EnvHttpProxyAgent オプション、固定ルックアップのヘルパー |
    | `plugin-sdk/runtime-fetch` | 2026 年 7 月以降はプライベートローカル。プロキシおよび guarded-fetch のインポートを伴わない、ディスパッチャー対応ランタイム fetch |
    | `plugin-sdk/inline-image-data-url-runtime` | 2026 年 7 月以降はプライベートローカル。広範なメディアランタイムサーフェスを使用しない、インライン画像データ URL のサニタイザーおよびシグネチャ判別ヘルパー |
    | `plugin-sdk/response-limit-runtime` | 2026 年 7 月以降はプライベートローカル。広範なメディアランタイムサーフェスを使用しない、バイト数、アイドル時間、期限に上限を設けたレスポンス本文リーダー |
    | `plugin-sdk/session-binding-runtime` | 2026 年 7 月以降はプライベートローカル。設定済みのバインディングルーティングまたはペアリングストアを含まない、現在の会話バインディング状態 |
    | `plugin-sdk/context-visibility-runtime` | 2026 年 7 月以降はプライベートローカル。広範な設定およびセキュリティのインポートを伴わない、コンテキスト可視性の解決および補足コンテキストのフィルタリング |
    | `plugin-sdk/string-coerce-runtime` | Markdown およびロギングのインポートを伴わない、用途を限定したプリミティブなレコードおよび文字列の型強制変換、正規化ヘルパー |
    | `plugin-sdk/html-entity-runtime` | 2026 年 7 月以降はプライベートローカル。広範なテキストユーティリティを使用しない、セミコロンで終端された HTML5 エンティティの単一パスデコード |
    | `plugin-sdk/text-utility-runtime` | 2026年7月以降はプライベートローカル。5つのエンティティの HTML エスケープを含む、低レベルのテキストおよびパス用ヘルパー |
    | `plugin-sdk/widget-html` | 自己完結型 HTML ウィジェット向けの完全なドキュメントの検出、サイズ検証、およびツール入力エラー |
    | `plugin-sdk/host-runtime` | 2026年7月以降はプライベートローカル。ホスト名および SCP ホストの正規化ヘルパー |
    | `plugin-sdk/retry-runtime` | 2026年7月以降はプライベートローカル。再試行設定および再試行ランナーのヘルパー |
    | `plugin-sdk/agent-runtime` | `resolveAgentDir`、`resolveDefaultAgentDir`、および非推奨の `resolveOpenClawAgentDir` 互換エクスポートを含む、エージェントのディレクトリ、ID、ワークスペース用ヘルパーの非推奨の包括的バレル。用途を限定したエージェント／ランタイムのサブパスを推奨 |
    | `plugin-sdk/directory-runtime` | 設定に基づくディレクトリのクエリ／重複排除 |
    | `plugin-sdk/keyed-async-queue` | 2026年7月以降はプライベートローカル。`KeyedAsyncQueue` |
  </Accordion>

  <Accordion title="機能およびテストのサブパス">
    | サブパス | 主なエクスポート |
    | --- | --- |
    | `plugin-sdk/media-runtime` | `saveRemoteMedia`、`saveResponseMedia`、`readRemoteMediaBuffer`、および非推奨の `fetchRemoteMedia` を含む、非推奨の広範なメディアバレル。`plugin-sdk/media-store`、`plugin-sdk/media-mime`、`plugin-sdk/outbound-media`、および機能ランタイムのサブパスを推奨する。また、URL を OpenClaw メディアに変換する場合は、バッファー読み取りよりストアヘルパーを優先する |
    | `plugin-sdk/media-local-roots` | Plugin が所有するローカルメディア読み取り向けの、対象を絞った `getAgentScopedMediaLocalRoots(...)` およびポリシー対応の `getAgentScopedMediaLocalRootsForSources(...)` ヘルパー |
    | `plugin-sdk/media-mime` | 対象を絞った MIME 正規化、ファイル拡張子マッピング、MIME 検出、およびメディア種別ヘルパー |
    | `plugin-sdk/media-store` | `saveMediaBuffer` や `saveMediaStream` などの、対象を絞ったメディアストアヘルパー |
    | `plugin-sdk/media-generation-runtime` | 2026年7月以降はプライベートローカル。共有のメディア生成フェイルオーバーヘルパー、候補選択、およびモデル欠落時のメッセージ |
    | `plugin-sdk/media-understanding` | メディア理解プロバイダーの型およびヘルパー向けの、非推奨の互換性ファサード。新しいプロバイダーは注入された Plugin API を通じて登録し、リクエストヘルパーは Plugin 側で所有する |
    | `plugin-sdk/text-chunking` | 送信テキストおよびオフセットを保持する範囲チャンク分割、Markdown のチャンク分割／レンダリングヘルパー、引用符を考慮した HTML タグのトークン化、Markdown テーブル変換、ディレクティブタグの除去、および安全なテキストユーティリティ |
    | `plugin-sdk/speech` | 2026年7月以降はプライベートローカル。音声プロバイダーの型に加え、プロバイダー向けディレクティブ、レジストリ、検証、OpenAI 互換 TTS ビルダー、および音声ヘルパーのエクスポート |
    | `plugin-sdk/speech-core` | 2026年7月以降はプライベートローカル。共有の音声プロバイダー型、レジストリ、ディレクティブ、正規化、および音声ヘルパーのエクスポート |
    | `plugin-sdk/speech-settings` | プロバイダーレジストリや合成ランタイムを含まない、軽量な TTS 設定の解決および正規化プリミティブ |
    | `plugin-sdk/realtime-transcription` | 2026年7月以降はプライベートローカル。リアルタイム文字起こしプロバイダーの型、レジストリヘルパー、および共有 WebSocket セッションヘルパー |
    | `plugin-sdk/realtime-bootstrap-context` | 2026年7月以降はプライベートローカル。制限付きの `IDENTITY.md`、`USER.md`、および `SOUL.md` コンテキスト注入向けのリアルタイムプロファイル初期化ヘルパー |
    | `plugin-sdk/realtime-voice` | 2026年7月以降はプライベートローカル。リアルタイム音声プロバイダーの型、レジストリヘルパー、共有の音声エネルギー／発話開始ゲート、およびトランスポート非依存のセッションハーネスと出力アクティビティ追跡を含むリアルタイム音声動作ヘルパー |
    | `plugin-sdk/meeting-runtime` | ブラウザー会議セッションランタイム、リアルタイム音声エンジン／トランスポート、`MeetingPlatformAdapter`、ブラウザー／Node 制御、エージェント相談、音声通話の委任、セットアップチェック、および SoX コマンドヘルパー |
    | `plugin-sdk/image-generation` | 2026年7月以降はプライベートローカル。画像生成プロバイダーの型に加え、画像アセット／データ URL ヘルパー、および OpenAI 互換画像プロバイダービルダー |
    | `plugin-sdk/image-generation-core` | 2026年7月以降はプライベートローカル。共有の画像生成型、フェイルオーバー、認証、およびレジストリヘルパー |
    | `plugin-sdk/music-generation` | 2026年7月以降はプライベートローカル。音楽生成のプロバイダー／リクエスト／結果の型 |
    | `plugin-sdk/video-generation` | 2026年7月以降はプライベートローカル。動画生成のプロバイダー／リクエスト／結果の型 |
    | `plugin-sdk/video-generation-core` | 2026年7月以降はプライベートローカル。共有の動画生成型、フェイルオーバーヘルパー、プロバイダー検索、およびモデル参照の解析 |
    | `plugin-sdk/transcripts` | 2026年7月以降はプライベートローカル。共有の文字起こしソースプロバイダー型、レジストリヘルパー、会議プロバイダーブリッジファクトリー、セッション記述子、および発話メタデータ |
    | `plugin-sdk/webhook-targets` | 2026年7月以降はプライベートローカル。Webhook ターゲットレジストリおよびルートインストールヘルパー |
    | `plugin-sdk/web-media` | 共有のリモート／ローカルメディア読み込みヘルパー |
    | `plugin-sdk/zod` | 非推奨の互換性再エクスポート。`zod` から `zod` を直接インポートする |
    | `plugin-sdk/plugin-test-api` | リポジトリのテストヘルパーブリッジをインポートせずに、Plugin の直接登録ユニットテストを行うためのリポジトリローカルな最小 `createTestPluginApi` ヘルパー |
    | `plugin-sdk/agent-runtime-test-contracts` | 認証、配信、フォールバック、ツールフック、プロンプトオーバーレイ、スキーマ、およびトランスクリプト射影テスト向けの、リポジトリローカルなネイティブエージェントランタイムアダプター契約フィクスチャ |
    | `plugin-sdk/channel-test-helpers` | 汎用アクション／セットアップ／ステータス契約、ディレクトリのアサーション、アカウント起動ライフサイクル、送信設定の受け渡し、ランタイムモック、ステータス問題、送信配信、およびフック登録向けの、リポジトリローカルなチャネル指向テストヘルパー |
    | `plugin-sdk/channel-target-testing` | チャネルテスト向けの、リポジトリローカルな共有ターゲット解決エラーケーススイート |
    | `plugin-sdk/channel-contract-testing` | 広範なテストバレルを使用しない、リポジトリローカルな対象を絞ったチャネル契約テストヘルパー |
    | `plugin-sdk/plugin-test-contracts` | リポジトリローカルな Plugin パッケージ、登録、公開アーティファクト、直接インポート、ランタイム API、およびインポート副作用の契約ヘルパー |
    | `plugin-sdk/plugin-state-test-runtime` | リポジトリローカルな Plugin 状態ストア、受信キュー、および状態 DB のテストヘルパー |
    | `plugin-sdk/provider-test-contracts` | リポジトリローカルなプロバイダーランタイム、認証、検出、オンボーディング、カタログ、ウィザード、メディア機能、再生ポリシー、リアルタイム STT ライブ音声、Web 検索／取得、およびストリーム契約のヘルパー |
    | `plugin-sdk/provider-http-test-mocks` | 2026年7月以降はプライベートローカル。`plugin-sdk/provider-http` を実行するプロバイダーテスト向けの、リポジトリローカルなオプトイン式 Vitest HTTP／認証モック |
    | `plugin-sdk/reply-payload-testing` | 返信ペイロードのフィクスチャにメタデータを付加するためのリポジトリローカルヘルパー |
    | `plugin-sdk/sqlite-runtime-testing` | ファーストパーティテスト向けのリポジトリローカルな SQLite ライフサイクルヘルパー |
    | `plugin-sdk/test-fixtures` | リポジトリローカルな汎用 CLI ランタイムキャプチャ、サンドボックスコンテキスト、スキルライター、エージェントメッセージ、システムイベント、モジュール再読み込み、バンドル済み Plugin パス、ターミナルテキスト、チャンク分割、認証トークン、および型付きケースのフィクスチャ |
    | `plugin-sdk/test-node-mocks` | Vitest の `vi.mock("node:*")` ファクトリー内で使用する、リポジトリローカルな対象を絞った Node 組み込みモックヘルパー |
  </Accordion>

  <Accordion title="メモリのサブパス">
    | サブパス | 主なエクスポート |
    | --- | --- |
    | `plugin-sdk/memory-core-host-embedding-registry` | 2026年7月以降はプライベートローカル。軽量なメモリ埋め込みプロバイダーレジストリヘルパー |
    | `plugin-sdk/memory-core-host-engine-foundation` | メモリホスト基盤エンジンのエクスポート |
    | `plugin-sdk/memory-core-host-engine-embeddings` | 2026年7月以降はプライベートローカル。メモリホストの埋め込み契約、レジストリアクセス、ローカルプロバイダー、および汎用バッチ／リモートヘルパー。このサーフェス上の `registerMemoryEmbeddingProvider` は非推奨。新しいプロバイダーには汎用埋め込みプロバイダー API を使用する。 |
    | `plugin-sdk/memory-core-host-engine-qmd` | 2026年7月以降はプライベートローカル。メモリホスト QMD エンジンのエクスポート |
    | `plugin-sdk/memory-core-host-engine-storage` | 2026年7月以降はプライベートローカル。メモリホストストレージエンジンのエクスポート |
    | `plugin-sdk/memory-core-host-secret` | 2026年7月以降はプライベートローカル。メモリホストのシークレットヘルパー |
    | `plugin-sdk/memory-core-host-status` | 2026年7月以降はプライベートローカル。メモリホストのステータスヘルパー |
    | `plugin-sdk/memory-core-host-runtime-cli` | 2026年7月以降はプライベートローカル。メモリホストの CLI ランタイムヘルパー |
    | `plugin-sdk/memory-core-host-runtime-core` | 2026年7月以降はプライベートローカル。メモリホストのコアランタイムヘルパー |
    | `plugin-sdk/memory-core-host-runtime-files` | 2026年7月以降はプライベートローカル。メモリホストのファイル／ランタイムヘルパー |
    | `plugin-sdk/memory-host-core` | ベンダー中立なメモリホストヘルパー向けの、非推奨の互換性ファサード。新しいメモリ Plugin は注入されたメモリ機能とホストが準備したプロンプトを使用する。関連 Plugin は、対象を絞った読み取りシームが用意されるまで、公開アーティファクトの検出に保持されたファサードを引き続き使用する。 |
    | `plugin-sdk/memory-host-events` | 2026年7月以降はプライベートローカル。メモリホストのイベントジャーナルヘルパーに対するベンダー中立なエイリアス |
    | `plugin-sdk/memory-host-markdown` | 2026年7月以降はプライベートローカル。メモリ関連 Plugin 向けの共有管理対象 Markdown ヘルパー |
    | `plugin-sdk/memory-host-search` | 2026年7月以降はプライベートローカル。検索マネージャーアクセス向けの Active Memory ランタイムファサード |
  </Accordion>

  <Accordion title="予約済みバンドルヘルパーのサブパス">
    予約済みバンドルヘルパー SDK サブパスは、バンドルされた
    Plugin コード向けの、所有者固有の対象を絞ったサーフェスです。
    パッケージのビルドとエイリアス処理を決定的に保つため SDK
    インベントリで追跡されますが、一般的な Plugin 作成 API ではありません。
    新しい再利用可能なホスト契約には、`plugin-sdk/gateway-runtime` や
    `plugin-sdk/ssrf-runtime` などの汎用 SDK サブパスを使用してください。

    | サブパス | 所有者と目的 |
    | --- | --- |
    | `plugin-sdk/codex-mcp-projection` | 2026年7月以降はプライベートローカル。ユーザーの MCP サーバー設定を Codex app-server のスレッド設定に射影するための、バンドル済み Codex Plugin ヘルパー（予約済みパッケージエクスポート） |
    | `plugin-sdk/codex-native-task-runtime` | Codex app-server のネイティブサブエージェントを OpenClaw のタスク状態にミラーリングするための、バンドル済み Codex Plugin ヘルパー（リポジトリローカルのみ。パッケージエクスポートではない） |

  </Accordion>
</AccordionGroup>

## 関連項目

- [Plugin SDK の概要](/ja-JP/plugins/sdk-overview)
- [Plugin SDK のセットアップ](/ja-JP/plugins/sdk-setup)
- [Plugin の構築](/ja-JP/plugins/building-plugins)
