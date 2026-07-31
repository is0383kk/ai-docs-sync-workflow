---
read_when:
    - OPENCLAW_PLUGIN_SDK_COMPAT_DEPRECATED 警告が表示される
    - OPENCLAW_EXTENSION_API_DEPRECATED 警告が表示される
    - OpenClaw 2026.4.25 より前に api.registerEmbeddedExtensionFactory を使用していた場合
    - Pluginを最新のPluginアーキテクチャに更新しています
    - 外部の OpenClaw Plugin を管理している場合
sidebarTitle: Migrate to SDK
summary: レガシーな後方互換性レイヤーから最新の Plugin SDK へ移行する
title: Plugin SDK の移行
x-i18n:
    generated_at: "2026-07-26T09:14:38Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: a483f9c0f8409505fc2688872995382944e002520ceb651214dbc5ad8e3554fb
    source_path: plugins/sdk-migration.md
    workflow: 16
---

OpenClaw は、広範な後方互換性レイヤーを、小規模で目的を絞ったインポートから構成される最新の Plugin
アーキテクチャに置き換えました。お使いの Plugin がこの変更より前に作成されたものである場合、
このガイドに従って現在のコントラクトへ移行できます。

## 変更点

以前は、非常に広範なインポートサーフェスにより、Plugin は単一のエントリポイントから
ほぼあらゆるものにアクセスできました。

- **`openclaw/plugin-sdk`** と **`openclaw/plugin-sdk/compat`** - 目的を絞った SDK の構築中に、
  数十個のヘルパーを再エクスポートしていました。現在は両方のルートが削除されています。
  代わりに、文書化されたサブパスをインポートしてください。
- **`openclaw/plugin-sdk/infra-runtime`** - システムイベント、Heartbeat の状態、
  配信キュー、フェッチ／プロキシヘルパー、ファイルヘルパー、承認型、
  および無関係なユーティリティを混在させていた広範なバレルです。
- **`openclaw/plugin-sdk/config-runtime`** - 後続の互換期間のためだけに維持されていた
  広範な設定バレルです。ランタイムを直接読み書きするヘルパーは削除されています。
- **`openclaw/extension-api`** - 組み込みエージェントランナーなどの
  ホスト側ヘルパーへの直接アクセスを Plugin に提供していた、削除済みのブリッジです。
- **`api.registerEmbeddedExtensionFactory(...)`** - `tool_result` などの組み込みランナーイベントを
  監視していた、組み込みランナー専用の削除済みフックです。代わりにエージェントの
  ツール結果ミドルウェアを使用してください（[組み込みツール結果拡張機能を
  ミドルウェアへ移行する](#how-to-migrate)を参照）。

ルート SDK、互換バレル、拡張機能ブリッジ、および組み込み拡張機能ファクトリは
削除されました。`infra-runtime` と `config-runtime` は、個別に記録された後続の
互換期間のためだけに残されています。新しい Plugin では、目的を絞ったサブパスを使用してください。

<Warning>
  削除済みのルート、互換、または拡張機能サーフェスをインポートする Plugin は、
  読み込まれなくなりました。アップグレードする前に、以下の対応表に従ってください。
</Warning>

OpenClaw は、代替機能を導入するのと同じ変更で、文書化された Plugin の動作を
削除したり再解釈したりすることはありません。コントラクトを破壊する変更には、
まず互換アダプター、診断、ドキュメント、および非推奨期間が設けられます。
これは SDK インポート、マニフェストフィールド、セットアップ API、フック、
およびランタイム登録の動作に適用されます。

### 理由

- **起動の遅延** - 1 つのヘルパーをインポートすると、数十個の無関係なモジュールが読み込まれていました。
- **循環依存** - 広範な再エクスポートにより、インポートサイクルが
  容易に発生していました。
- **不明確な API サーフェス** - 安定したエクスポートと内部用エクスポートを
  区別する方法がありませんでした。

現在、各 `openclaw/plugin-sdk/<subpath>` は、文書化されたコントラクトを持つ
小規模で自己完結したモジュールになっています。

バンドルされたチャンネル向けの従来のプロバイダー便利機能も廃止されました。
チャンネル固有のヘルパーショートカットは、非公開のモノリポ用便利機能であり、
安定した Plugin コントラクトではありませんでした。代わりに、範囲の狭い汎用 SDK
サブパスを使用してください。バンドルされた Plugin ワークスペース内では、
プロバイダー所有のヘルパーを、その Plugin 自身の `api.ts` または
`runtime-api.ts` に配置してください。

- Anthropic は、Claude 固有のストリームヘルパーを自身の `api.ts` /
  `contract-api.ts` シームに保持します。
- OpenAI は、プロバイダービルダー、デフォルトモデルヘルパー、およびリアルタイムプロバイダー
  ビルダーを自身の `api.ts` に保持します。
- OpenRouter は、プロバイダービルダーおよびオンボーディング／設定ヘルパーを自身の
  `api.ts` に保持します。

## 互換性ポリシー

外部 Plugin の互換性対応は、次の順序で進めます。

1. 新しいコントラクトを追加する。
2. 互換アダプターを介して従来の動作を維持する。
3. 従来のパスと代替先を明記した診断または警告を出力する。
4. 両方のパスをテストで網羅する。
5. 非推奨化と移行手順を文書化する。
6. 告知した移行期間が終了した後、通常はメジャーリリースでのみ
   削除する。

マニフェストフィールドが引き続き受け入れられている場合は、ドキュメントと
診断で別途指示されるまで使用を続けてください。新しいコードでは文書化された
代替機能を優先する必要がありますが、既存の Plugin が通常のマイナーリリースで
動作しなくなることはありません。

### 公開済みチャンネルセットアップの互換性

`2026.7.1` を通じて公開された Slack、Discord、Signal、および Microsoft Teams
パッケージは、`openclaw/plugin-sdk/bundled-channel-config-schema` からチャンネル固有の設定スキーマを
インポートします。公開済みの Slack および Discord パッケージは、
`openclaw/plugin-sdk/setup-runtime` から `createLegacyCompatChannelDmPolicy` と
`promptLegacyChannelAllowFromForAccount` もインポートします。

これらのエクスポートは、非推奨のランタイム互換アダプターとして引き続き利用できます。
新しい Plugin および再公開される Plugin は、`channel-config-schema` と
`setup-runtime` の汎用プリミティブを使用し、設定スキーマとセットアップポリシーを
ローカルで所有する必要があります。互換エクスポートを削除できるのは、
サポート対象となる公開済みパッケージの最小バージョンが、それらをインポートしなくなった後だけです。

### チャンネルセットアップ入力フィールドの互換性

`ChannelSetupInput` では、チャンネル間で共通のセットアップエンベロープだけが
恒久的に型付けされるようになりました。チャンネル固有のフィールドは、
既存の外部 Plugin が引き続きコンパイルできるよう、Plugin 作成者がそれらのフィールドを
Plugin ローカルのセットアップ入力型へ移行するまで、非推奨の互換性階層で型付けされます。

OpenClaw はメジャーリリースを提供しません。2026-07-22 のレジストリ調査では、
公開済みのツリー外チャンネル Plugin 426 個を検査し、読み取り元のない 21 個のフィールドを削除しました。
維持された 22 個のフィールドには、それぞれ既知の公開済み読み取り元があります。以後の各フィールドは、
公開済み Plugin から読み取られなくなり次第削除されます。Plugin 作成者が Plugin ローカルの
セットアップ入力型へ移行するにつれて、維持されるフィールド群は縮小します。

同じ調査で、公開済みの依存先がない従来の未宣言アダプター昇格キー 23 個も削除されました。
一般的な 6 個のキーと、セットアップ専用の `rooms` キーは残されています。
公開済み Plugin が `singleAccountKeysToMove` を宣言するにつれて、このキー群も縮小します。

共有型にはインデックスシグネチャがありません。Plugin 所有のキーはランタイム入力オブジェクト上に
引き続き存在できますが、Plugin ローカルの交差型で宣言するか、所有する Plugin の
セットアップスキーマを通じて絞り込んでください。

| `code`                                  | `owner`   | `replacement`                                                                                    | 削除条件                                                               |
| --------------------------------------- | --------- | ------------------------------------------------------------------------------------------------ | --------------------------------------------------------------------- |
| `plugin-sdk-channel-setup-input-fields` | `channel` | `ChannelSetupInput` と、所有するチャンネルのフィールドを宣言する Plugin ローカル型との交差型を作成する | 公開済み Plugin のレジストリ調査で読み取り元がなくなった時点でフィールドを削除する |

従来の未宣言アダプター昇格階層も、同じ読み取り元主導のポリシーに従います。
Plugin に追加の昇格キーが不要な場合は空配列を含めて `singleAccountKeysToMove` を宣言し、
共有フォールバックをキーごとに廃止できるようにしてください。

#### 読み取り元の検証

1. 各 `nextCursor` を使用して `https://clawhub.ai/api/v1/packages?family=code-plugin&limit=100` をページ送りし、`categories` に `channels` が含まれるパッケージを保持します。
2. `npm search --json --searchlimit=1000 "openclaw channel plugin"` から npm の候補を追加します。`openclaw/plugin-sdk/channel-setup`、`openclaw/plugin-sdk/setup`、および `openclaw/plugin-sdk/core` を対象とした GitHub コード検索から、ソースのみの候補を追加します。
3. 各候補の公開済み最新バージョンを特定します。`npm pack <package>@<version> --json --pack-destination <temp-dir>` を実行して展開し、出荷された `dist` の JavaScript と宣言を調べ、フィールドの直接読み取りまたは分割代入による読み取りを確認します。パッケージに npm リリースがない場合は、ClawHub アーティファクトをダウンロードします。
4. パッケージ、バージョン、フィールドまたは昇格キー、および一致したファイルを記録します。フィールドまたはキーを削除できるのは、公開済みの Plugin アーティファクトから読み取られていない場合だけです。維持されるフィールドおよびキーのリストの横にあるコードコメント内の読み取り元名を、調査結果と同期させてください。

これはソース／型の互換性に関する記録だけです。ランタイムのセットアップ入力オブジェクトと
セットアップ動作は変更されていないため、ランタイムアダプターや互換性レジストリの
エントリはありません。

現在の移行キューを `pnpm plugins:boundary-report` で監査します。

| フラグ                                                    | 効果                                                                           |
| ------------------------------------------------------- | ------------------------------------------------------------------------------ |
| `--summary`（または `pnpm plugins:boundary-report:summary`） | 完全な詳細ではなく、簡潔な件数を表示します。                                   |
| `--json`                                                | 機械可読レポートを出力します。                                                 |
| `--owner <id>`                                          | 1 つの Plugin または互換性所有者に絞り込みます。                               |
| `--fail-on-cross-owner`                                 | 所有者をまたぐ予約済み SDK インポートがある場合、0 以外で終了します。          |
| `--fail-on-eligible-compat`                             | 非推奨の互換性レコードの `removeAfter` 日付を過ぎている場合、0 以外で終了します。 |
| `--fail-on-unclassified-unused-reserved`                | 使用されていない予約済み SDK シムがある場合、0 以外で終了します。              |

`pnpm plugins:boundary-report:ci` は、3 つすべての失敗フラグを指定して実行されます。非推奨の
レコードには通常、曖昧な「次のメジャーリリース」ではなく、明示的な
`removeAfter` 日付が設定されます。所有者が日付を承認していないレコードでは、
`removeAfter` が存在せず、`no-date` として表示され、削除対象にはなりません。
レポートは非推奨レコードを日付別にグループ化し、ローカルのコード／ドキュメント参照を数え、
所有者をまたぐ予約済み SDK インポートを表示し、非公開のメモリホスト SDK ブリッジを
要約します。予約済み SDK サブパスには、追跡対象となる所有者の使用実績が必要です。
使用されていない予約済みエクスポートは、公開 SDK から削除する必要があります。

### メディアの従来形式への投影

`media-legacy-projection` 互換性レコードは、従来の並列メディアフィールド、
ペイロードビルダー、フックメタデータのエイリアス、およびメディアテンプレート名を対象とします。
承認済みの `removeAfter` 日付は **2026-10-01** です（ファクト優先の代替機能が
出荷されてから 2 回のリリース周期後）。削除には、その時点で公開済み Plugin
アーティファクトの調査結果がクリーンであることも必要です。日付より前に移行してください。

チャンネルの受信処理では、単数形／複数形の `MediaPath`、`MediaUrl`、
`MediaType`、`MediaPaths`、`MediaUrls`、`MediaTypes`、
`MediaTranscribedIndexes`、`MediaWorkspaceDir`、および `MediaStaged` を、順序付きの
ファクトに置き換えてください。

```ts
import { toInboundMediaFacts } from "openclaw/plugin-sdk/channel-inbound";

const media = toInboundMediaFacts([
  { path: saved.path, url: nativeUrl, contentType: saved.contentType, messageId },
]);

const ctx = finalizeInboundContext({ Body: caption, media });
```

`inbound_claim` および `message_received` フックでは `event.media` を使用してください。
リモートメディアがローカルにステージングされていない場合、識別／診断には
`event.originalMedia` を使用し、`event.media` を待機してください。
`event.mediaStagingPending` はその状態を区別します。`event.metadata` から非推奨の
単数形／複数形プロパティを読み取らないでください。

CLI メディアモデルでは、`{{MediaPath}}`、`{{MediaUrl}}`、`{{MediaType}}`、
および `{{MediaDir}}` を、`{{AttachmentPath}}`、`{{AttachmentUrl}}`、
`{{AttachmentContentType}}`、および `{{AttachmentDir}}` に置き換えてください。
添付ファイルの位置が重要な場合は、`{{AttachmentIndex}}` を使用してください。

ローカルメディアの読み取りポリシーには、`openclaw/plugin-sdk/media-local-roots` から
`getAgentScopedMediaLocalRoots(...)` または `getAgentScopedMediaLocalRootsForSources(...)` をインポートしてください。
`openclaw/plugin-sdk/agent-media-payload` ファサードと、その `buildAgentMediaPayload(...)` 投影は非推奨です。

## 移行方法

<Steps>
  <Step title="ランタイム設定の読み書きヘルパーを移行する">
    バンドルされた Plugin は、`api.runtime.config.loadConfig()` と
    `api.runtime.config.writeConfigFile(...)` を直接呼び出さないようにしてください。
    アクティブな呼び出しパスへすでに渡されている設定を優先してください。
    現在のプロセススナップショットを必要とする長寿命ハンドラーでは、
    `api.runtime.config.current()` を使用できます。長寿命のエージェントツールでは、
    設定の書き込み前に作成されたツールでも更新後の設定を参照できるよう、
    `execute` 内で `ctx.getRuntimeConfig()` を読み取る必要があります。

    設定の書き込みには、書き込み後のポリシーを明示したトランザクションヘルパーを使用します。

    ```typescript
    await api.runtime.config.mutateConfigFile({
      afterWrite: { mode: "auto" },
      mutate(draft) {
        draft.plugins ??= {};
      },
    });
    ```

    クリーンな Gateway の再起動が必要な変更には `afterWrite: { mode: "restart", reason: "..." }` を使用し、
    `afterWrite: { mode: "none", reason: "..." }` は、呼び出し元が後続処理を所有し、再読み込みプランナーを
    意図的に抑制する場合にのみ使用します。ミューテーション結果には、テストと
    ロギング用の型付き `followUp` サマリーが含まれます。再起動の適用または
    スケジュールは、引き続き Gateway が担います。

    `loadConfig` と `writeConfigFile` は、Plugin
    ランタイムから削除されました。バンドル済み Plugin とリポジトリのランタイムコードは、
    `pnpm check:deprecated-api-usage` と
    `pnpm check:no-runtime-action-load-config` によって保護されています。新しい本番 Plugin での使用は
    即座に失敗し、設定への直接書き込みも失敗します。Gateway サーバーメソッドは
    リクエストランタイムのスナップショットを使用する必要があり、ランタイムのチャンネル送信、
    アクション、クライアントヘルパーは境界から設定を受け取る必要があります。また、
    長期間存続するランタイムモジュールでは、暗黙的な `loadConfig()` 呼び出しは
    一切許可されません。

    新しい Plugin コードでは、広範な `openclaw/plugin-sdk/config-runtime`
    バレルを避けてください。用途に応じた限定的なサブパスを使用します。

    | 用途 | インポート |
    | --- | --- |
    | `OpenClawConfig` などの設定型 | `openclaw/plugin-sdk/config-contracts` |
    | Plugin エントリでの設定検索 | `api.pluginConfig` |
    | 設定のマージ | 設定境界にある Plugin ローカルのロジック |
    | 現在のランタイムスナップショットの読み取り | `openclaw/plugin-sdk/runtime-config-snapshot` |
    | 設定の書き込み | `openclaw/plugin-sdk/config-mutation` |
    | セッションストアヘルパー | `openclaw/plugin-sdk/session-store-runtime` |
    | Markdown テーブル設定 | `openclaw/plugin-sdk/markdown-table-runtime` |
    | グループポリシーのランタイムヘルパー | `openclaw/plugin-sdk/runtime-group-policy` |
    | シークレット入力の解決 | `openclaw/plugin-sdk/secret-input-runtime` |
    | モデル／セッションのオーバーライド | `openclaw/plugin-sdk/model-session-runtime` |

    バンドル済み Plugin とそのテストでは、必要な動作にインポートとモックを
    局所化するため、広範なバレルの使用がスキャナーによって禁止されています。
    外部互換性のためにバレル自体は引き続き存在しますが、新しいコードでは
    依存しないでください。

  </Step>

  <Step title="埋め込みツール結果拡張をミドルウェアへ移行する">
    バンドル済み Plugin では、埋め込みランナー専用の
    `api.registerEmbeddedExtensionFactory(...)` ツール結果ハンドラーを、
    ランタイムに依存しないミドルウェアへ置き換える必要があります。

    ```typescript
    // OpenClaw ランタイムツールと Codex ランタイムの動的ツール（結果は
    // 変換される場合があります）。Codex ネイティブのツール結果も監視用に中継されますが、
    // 変換後の出力がモデルに届くことはありません。Codex の
    // PostToolUse フック契約では、ネイティブツールの応答を置き換えられないためです。
    api.registerAgentToolResultMiddleware(async (event) => {
      return compactToolResult(event);
    }, {
      runtimes: ["openclaw", "codex"],
    });
    ```

    同時に Plugin マニフェストも更新します。

    ```json
    {
      "contracts": {
        "agentToolResultMiddleware": ["openclaw", "codex"]
      }
    }
    ```

    インストール済み Plugin でも、明示的に有効化され、対象となるすべてのランタイムが
    `contracts.agentToolResultMiddleware` で宣言されている場合は、ツール結果ミドルウェアを
    登録できます。宣言されていないインストール済みミドルウェアの登録は拒否されます。

  </Step>

  <Step title="承認ネイティブハンドラーをケイパビリティ情報へ移行する">
    承認に対応するチャンネル Plugin は、
    `approvalCapability.nativeRuntime` と共有ランタイムコンテキスト
    レジストリを通じて、ネイティブの承認動作を公開します。

    - `approvalCapability.handler.loadRuntime(...)` を
      `approvalCapability.nativeRuntime` に置き換えます。
    - 承認固有の認証／配信を、従来の `plugin.auth` /
      `plugin.approvals` の接続から `approvalCapability` へ移行します。
    - `ChannelPlugin.approvals` は公開チャンネル Plugin 契約から
      削除されました。配信、ネイティブ、レンダリングの各フィールドを
      `approvalCapability` へ移動します。
    - `plugin.auth` はチャンネルのログイン／ログアウトフロー専用として
      残ります。コアは承認用の認証フックをここから読み取らなくなりました。
    - チャンネルが所有するランタイムオブジェクト（クライアント、トークン、Bolt アプリ）は、
      `openclaw/plugin-sdk/channel-runtime-context` を通じて登録します。
    - ネイティブ承認ハンドラーから、Plugin が所有する再ルーティング通知を送信しないでください。
      実際の配信結果に基づく別経路へのルーティング通知は、コアが所有します。
    - `channelRuntime` を `createChannelManager(...)` に渡す場合は、
      実体のある `createPluginRuntime().channel` サーフェスを指定してください。不完全なスタブは
      拒否されます。

    現在の承認ケイパビリティの構成については、[チャンネル Plugin](/ja-JP/plugins/sdk-channel-plugins)を
    参照してください。

  </Step>

  <Step title="Windows ラッパーのフォールバック動作を監査する">
    Plugin で `openclaw/plugin-sdk/windows-spawn` を使用している場合、解決できない Windows の
    `.cmd`/`.bat` ラッパーは、`allowShellFallback: true` を明示的に渡さない限り、
    フェイルクローズするようになりました。

    ```typescript
    // 変更前
    const program = applyWindowsSpawnProgramPolicy({ candidate });

    // 変更後
    const program = applyWindowsSpawnProgramPolicy({
      candidate,
      // シェルを介したフォールバックを意図的に許容する、信頼済みの
      // 互換性維持用呼び出し元でのみ設定してください。
      allowShellFallback: true,
    });
    ```

    呼び出し元がシェルフォールバックへ意図的に依存していない場合は、
    `allowShellFallback` を設定せず、代わりにスローされたエラーを処理してください。

  </Step>

  <Step title="非推奨のインポートを検索する">
    ```bash
    grep -r "plugin-sdk/compat" my-plugin/
    grep -r "plugin-sdk/infra-runtime" my-plugin/
    grep -r "plugin-sdk/config-runtime" my-plugin/
    grep -r "openclaw/extension-api" my-plugin/
    ```
  </Step>

  <Step title="用途別のインポートへ置き換える">
    旧サーフェスの各エクスポートは、対応する最新のインポートパスへ移行します。

    ```typescript
    // 変更前（非推奨の後方互換レイヤー）
    import {
      createChannelReplyPipeline,
      createPluginRuntimeStore,
      resolveControlCommandGate,
    } from "openclaw/plugin-sdk/compat";

    // 変更後（最新の用途別インポート）
    import { createChannelReplyPipeline } from "openclaw/plugin-sdk/channel-reply-pipeline";
    import { createPluginRuntimeStore } from "openclaw/plugin-sdk/runtime-store";
    import { resolveControlCommandGate } from "openclaw/plugin-sdk/command-auth";
    ```

    ホスト側ヘルパーでは、直接インポートする代わりに、
    注入された Plugin ランタイムを使用します。

    ```typescript
    // 変更前（非推奨の extension-api ブリッジ）
    import { runEmbeddedAgent } from "openclaw/extension-api";
    const result = await runEmbeddedAgent({ sessionId, prompt });

    // 変更後（注入されたランタイム）
    const result = await api.runtime.agent.runEmbeddedAgent({ sessionId, prompt });
    ```

    その他の従来のブリッジヘルパーにも同じパターンを適用します。

    | 旧インポート | 最新の代替手段 |
    | --- | --- |
    | `resolveAgentDir` | `api.runtime.agent.resolveAgentDir` |
    | `resolveAgentWorkspaceDir` | `api.runtime.agent.resolveAgentWorkspaceDir` |
    | `resolveAgentIdentity` | `api.runtime.agent.resolveAgentIdentity` |
    | `resolveThinkingDefault` | `api.runtime.agent.resolveThinkingDefault` |
    | `resolveAgentTimeoutMs` | `api.runtime.agent.resolveAgentTimeoutMs` |
    | `ensureAgentWorkspace` | `api.runtime.agent.ensureAgentWorkspace` |
    | セッションストアヘルパー | `api.runtime.agent.session.*` |

  </Step>

  <Step title="広範な infra-runtime インポートを置き換える">
    `openclaw/plugin-sdk/infra-runtime` は外部互換性のために引き続き存在しますが、
    新しいコードでは、実際に必要な用途別サーフェスをインポートしてください。

    | 用途 | インポート |
    | --- | --- |
    | システムイベントキューヘルパー | `openclaw/plugin-sdk/system-event-runtime` |
    | Heartbeat のウェイク、イベント、可視性ヘルパー | `openclaw/plugin-sdk/heartbeat-runtime` |
    | 保留中の配信キューのドレイン | `openclaw/plugin-sdk/delivery-queue-runtime` |
    | チャンネルアクティビティのテレメトリ | `openclaw/plugin-sdk/channel-activity-runtime` |
    | インメモリおよび永続ストレージを基盤とする重複排除キャッシュ | `openclaw/plugin-sdk/dedupe-runtime` |
    | 安全なローカルファイル／メディアパスヘルパー | `openclaw/plugin-sdk/file-access-runtime` |
    | ディスパッチャー対応のフェッチ | `openclaw/plugin-sdk/runtime-fetch` |
    | プロキシおよび保護付きフェッチヘルパー | `openclaw/plugin-sdk/fetch-runtime` |
    | SSRF ディスパッチャーポリシー型 | `openclaw/plugin-sdk/ssrf-dispatcher` |
    | 承認リクエスト／解決型 | `openclaw/plugin-sdk/approval-runtime` |
    | 承認応答ペイロードおよびコマンドヘルパー | `openclaw/plugin-sdk/approval-reply-runtime` |
    | エラー整形ヘルパー | `openclaw/plugin-sdk/error-runtime` |
    | トランスポート準備完了の待機 | `openclaw/plugin-sdk/transport-ready-runtime` |
    | セキュアトークンヘルパー | `openclaw/plugin-sdk/secure-random-runtime` |
    | 制限付き非同期タスク並行処理 | `openclaw/plugin-sdk/concurrency-runtime` |
    | 証明可能な不変条件に対する必須値アサーション | `openclaw/plugin-sdk/expect-runtime` |
    | 数値への型強制 | `openclaw/plugin-sdk/number-runtime` |
    | プロセスローカルの非同期ロック | `openclaw/plugin-sdk/async-lock-runtime` |
    | ファイルロック | `openclaw/plugin-sdk/file-lock` |

    バンドル済み Plugin では `infra-runtime` の使用がスキャナーによって禁止されているため、
    リポジトリコードが広範なバレルへ戻ることはありません。

  </Step>

  <Step title="チャンネルルートヘルパーを移行する">
    新しいチャンネルルートコードでは `openclaw/plugin-sdk/channel-route` を使用します。旧ルートキー名は、
    互換性エイリアスとして残ります。

    | 旧ヘルパー | 最新のヘルパー |
    | --- | --- |
    | `channelRouteIdentityKey(...)` | `channelRouteDedupeKey(...)` |
    | `channelRouteKey(...)` | `channelRouteCompactKey(...)` |

    最新のルートヘルパーは、ネイティブ承認、応答抑制、受信重複排除、
    Cron 配信、セッションルーティングの全体で `{ channel, to, accountId, threadId }` を
    一貫して正規化します。

    `plugin-sdk/channel-route` の `ChannelMessagingAdapter.parseExplicitTarget` または
    `resolveChannelRouteTargetWithParser(...)` を新たに使用しないでください。これらは非推奨であり、
    古い Plugin のためだけに残されています。新しいチャンネル Plugin では、
    ターゲット ID の正規化とディレクトリ検索失敗時のフォールバックに
    `messaging.targetResolver.resolveTarget(...)`、
    コアで早期にピア種別が必要な場合に `messaging.inferTargetChatType(...)`、
    プロバイダー固有のセッションおよびスレッド識別に
    `messaging.resolveOutboundSessionRoute(...)` を使用してください。

  </Step>

  <Step title="ビルドしてテストする">
    ```bash
    pnpm build
    pnpm test my-plugin/
    ```
  </Step>
</Steps>

## インポートパスのリファレンス

インポート可能な SDK サブパスについては、公開パッケージのエクスポートマップが
信頼できる唯一の情報源です。[SDK の概要](/ja-JP/plugins/sdk-overview)からリンクされている
トピック別 SDK ガイドを使用し、文書化された公開サブパスのうち最も限定的なものを
優先してください。`scripts/lib/plugin-sdk-entrypoints.json` のコンパイラインベントリには、
バンドル済み Plugin のビルドに使用されるプライベートローカルエントリも含まれますが、
そこに存在していても公開パッケージのエクスポートにはなりません。

この表は一般的な移行対象の一部であり、SDK サーフェス全体ではありません。
コンパイラのエントリポイントインベントリは `scripts/lib/plugin-sdk-entrypoints.json` にあり、
パッケージのエクスポートは公開対象のサブセットから生成されます。

バンドル済み Plugin 専用として予約されていたヘルパー境界は、公開 SDK
エクスポートマップから廃止されました。ただし、公開済みの `@openclaw/discord`
パッケージを引き続き直接インポートする外部 Plugin のために保持されている、
非推奨の `plugin-sdk/discord` シムなど、明示的に文書化された互換性ファサードは
除きます。所有者固有のヘルパーは、その所有元の Plugin パッケージ内に配置されます。
共有ホスト動作は、`plugin-sdk/gateway-runtime`、`plugin-sdk/security-runtime`、
注入された Plugin API などの汎用 SDK 契約を通じて提供されます。

用途に合致する最も限定的なインポートを使用してください。エクスポートが見つからない場合は、
`src/plugin-sdk/` のソースを確認するか、どの汎用契約が所有すべきかを
メンテナーに確認してください。

## 削除された互換性サーフェス

2026 年 7 月の整理では、ルート SDK および compat バレル、extension API
ブリッジ、期限切れの SDK サブパスエイリアス、未使用の SDK サブパス、
バンドル専用 SDK モジュールの公開エクスポートが削除されました。バンドル専用モジュールは、
プライベートローカルのビルドマッピングを通じてリポジトリ内の所有者が引き続き利用できますが、
公開パッケージからはインポートできません。

### プロセスグローバルな API プロバイダーの公開

`registerApiProvider(...)` と `unregisterApiProviders(...)` は、
`openclaw/plugin-sdk/llm` から削除されました。これらは API トランスポートを
プロセスグローバル状態へ公開していたため、ライフサイクルによって所有されるモデルランタイムが、
準備済みの各レジストリへコピーする必要がありました。

プロバイダー Plugin は、`api.registerProvider(...)` を通じて
テキスト推論プロバイダーを登録してください。`ApiRegistry` を構築する
ホスト所有のコードとテストは、そのレジストリへ直接登録し、プロバイダーの所有権と
破棄処理が準備済みランタイムのスコープ内に収まるようにしてください。

### プライベートテスト用バレル

`openclaw/plugin-sdk/testing` はリポジトリローカルであり、配布パッケージの
成果物から除外されていたため、2026-07-28 の `removeAfter` 日より前に
削除されました。リポジトリのテストでは、`plugin-sdk/plugin-test-runtime`、
`plugin-sdk/channel-test-helpers`、`plugin-sdk/channel-target-testing`、
`plugin-sdk/test-env`、`plugin-sdk/test-fixtures` などの用途別サブパスを使用します。

## 移行リファレンス

  これらのマッピングは、2026年7月に削除されたサーフェスと、それ以降の期間に有効な非推奨項目の両方を対象としています。マッピングは移行ガイダンスであり、古いサーフェスが引き続き利用可能であることを示すものではありません。現在の状態については、互換性レジストリと削除タイムラインを参照してください。

  <AccordionGroup>
  <Accordion title="command-auth ヘルプビルダー -> command-status">
    **旧 (`openclaw/plugin-sdk/command-auth`)**: `buildCommandsMessage`,
    `buildCommandsMessagePaginated`, `buildHelpMessage`。

    **新 (`openclaw/plugin-sdk/command-status`)**: シグネチャは同じで、より限定的なサブパスから
    インポートします。`command-auth` の互換性再エクスポートは
    削除されました。

    ```typescript
    // 変更前
    import { buildHelpMessage } from "openclaw/plugin-sdk/command-auth";

    // 変更後
    import { buildHelpMessage } from "openclaw/plugin-sdk/command-status";
    ```

  </Accordion>

  <Accordion title="メンションゲーティングヘルパー -> resolveInboundMentionDecision">
    **旧**: `openclaw/plugin-sdk/channel-inbound` または
    `openclaw/plugin-sdk/channel-mention-gating` の
    `resolveMentionGating(params)` と
    `resolveMentionGatingWithBypass(params)`。

    **新**: `resolveInboundMentionDecision({ facts, policy })` - 2つに分かれた呼び出し形式ではなく、
    1つの判定オブジェクトを使用します。

    Discord、iMessage、Matrix、MS Teams、QQBot、Signal、
    Telegram、WhatsApp、Zaloで採用されています。Slack独自の `app_mention` イベントモデルでは、
    このヘルパーを使用しません。

  </Accordion>

  <Accordion title="チャンネルランタイムシムとチャンネルアクションヘルパー">
    `openclaw/plugin-sdk/channel-runtime` は削除されました。ランタイムオブジェクトの登録には
    `openclaw/plugin-sdk/channel-runtime-context` を使用してください。

    `openclaw/plugin-sdk/channel-actions` のネイティブメッセージスキーマヘルパーは、
    生の「actions」チャンネルエクスポートとともに削除されました。代わりに、セマンティックな
    `presentation` サーフェスを通じて機能を公開してください。チャンネルPluginは、
    受け付ける生のアクション名ではなく、レンダリングするもの（カード、ボタン、選択項目）を
    宣言します。

  </Accordion>

  <Accordion title="Web検索プロバイダーの tool() ヘルパー -> Plugin上の createTool()">
    **旧**: `openclaw/plugin-sdk/provider-web-search` の `tool()` ファクトリ。

    **新**: プロバイダーPluginに `createTool(...)` を直接実装します。
    OpenClawでは、ツールラッパーの登録にSDKヘルパーが不要になりました。

  </Accordion>

  <Accordion title="プレーンテキストのチャンネルエンベロープ -> BodyForAgent">
    **旧**: 受信チャンネルメッセージからフラットなプレーンテキストのプロンプトエンベロープを構築する
    `api.runtime.channel.reply.formatInboundEnvelope(...)`（および受信メッセージオブジェクトの
    `channelEnvelope` フィールド）。

    **新**: `BodyForAgent` と構造化されたユーザーコンテキストブロック。チャンネル
    Pluginは、ルーティングメタデータ（スレッド、トピック、返信先、リアクション）を
    プロンプト文字列に連結するのではなく、型付きフィールドとして添付します。
    `formatAgentEnvelope(...)` ヘルパーは、合成された
    アシスタント向けエンベロープでは引き続きサポートされますが、受信プレーンテキストエンベロープは
    廃止される予定です。

    影響を受ける領域: `inbound_claim`、`message_received`、および古い
    エンベロープテキストを後処理していたカスタムチャンネルPlugin。

  </Accordion>

  <Accordion title="deactivate フック -> gateway_stop">
    **旧**: `api.on("deactivate", handler)`。

    **新**: `api.on("gateway_stop", handler)`。シャットダウン時のクリーンアップ契約は
    同じで、変更されるのはフック名のみです。

    ```typescript
    // 変更前
    api.on("deactivate", async (event, ctx) => {
      await stopPluginService(ctx);
    });

    // 変更後
    api.on("gateway_stop", async (event, ctx) => {
      await stopPluginService(ctx);
    });
    ```

    `deactivate` は、2026-08-16より後に削除されるまで、非推奨の互換性エイリアスとして
    引き続き接続されています。

  </Accordion>

  <Accordion title="subagent_spawning フック -> コアのスレッドバインディング">
    **旧**: `threadBindingReady` または `deliveryOrigin` を返す
    `api.on("subagent_spawning", handler)`。

    **新**: チャンネルセッションバインディングアダプターを通じて、コアに
    `thread: true` サブエージェントバインディングを準備させます。起動後の監視にのみ
    `api.on("subagent_spawned", handler)` を使用してください。

    ```typescript
    // 変更前
    api.on("subagent_spawning", async () => ({
      status: "ok",
      threadBindingReady: true,
      deliveryOrigin: { channel: "discord", to: "channel:123", threadId: "456" },
    }));

    // 変更後
    api.on("subagent_spawned", async (event) => {
      await observeSubagentLaunch(event);
    });
    ```

    `subagent_spawning`、`PluginHookSubagentSpawningEvent`、
    `PluginHookSubagentSpawningResult`、および
    `SubagentLifecycleHookRunner.runSubagentSpawning(...)` は、外部Pluginの移行中に限り
    非推奨の互換性サーフェスとして残り、2026-08-30より後に
    削除されます。

  </Accordion>

  <Accordion title="プロバイダー検出型 -> プロバイダーカタログ型">
    4つの検出型エイリアスは、現在ではカタログ時代の型に対する薄いラッパーです。

    | 旧エイリアス                 | 新しい型                  |
    | ------------------------- | ------------------------- |
    | `ProviderDiscoveryOrder`  | `ProviderCatalogOrder`    |
    | `ProviderDiscoveryContext`| `ProviderCatalogContext`  |
    | `ProviderDiscoveryResult` | `ProviderCatalogResult`   |
    | `ProviderPluginDiscovery` | `ProviderPluginCatalog`   |

    エイリアスとレガシーな `ProviderCapabilities` 静的バッグは
    削除されました。プロバイダーPluginは、静的オブジェクトではなく、
    `buildReplayPolicy`、`normalizeToolSchemas`、
    `wrapStreamFn` などの明示的なプロバイダーフックを使用してください。

  </Accordion>

  <Accordion title="思考ポリシーフック -> resolveThinkingProfile">
    **旧**（`ProviderThinkingPolicy` 上の3つの個別フック）:
    `isBinaryThinking(ctx)`、`supportsXHighThinking(ctx)`、
    `resolveDefaultThinkingLevel(ctx)`。

    **新**: 正規の `id`、任意の `label`、
    および順位付けされたレベルリストを持つ `ProviderThinkingProfile` を返す、
    単一の `resolveThinkingProfile(ctx)`。OpenClawは、保存済みの古い値を
    プロファイル順位に基づいて自動的にダウングレードします。

    コンテキストには、`provider`、`modelId`、任意のマージ済み
    `reasoning`、および任意のマージ済みモデル
    `compat` の情報が含まれます。プロバイダーPluginは、設定された
    リクエスト契約がサポートする場合にのみ、これらのカタログ情報を使用して
    モデル固有のプロファイルを公開できます。

    3つではなく1つのフックを実装してください。レガシーフックは削除されました。

  </Accordion>

  <Accordion title="外部認証プロバイダー -> contracts.externalAuthProviders">
    **旧**: Pluginマニフェストでプロバイダーを宣言せずに、
    外部認証フックを実装します。

    **新**: Pluginマニフェストで `contracts.externalAuthProviders` を宣言し、
    **かつ** `resolveExternalAuthProfiles(...)` を実装します。

    ```json
    {
      "contracts": {
        "externalAuthProviders": ["anthropic", "openai"]
      }
    }
    ```

  </Accordion>

  <Accordion title="プロバイダー環境変数検索 -> setup.providers[].envVars">
    **旧**マニフェストフィールド: `providerAuthEnvVars: { anthropic: ["ANTHROPIC_API_KEY"] }`。

    **新**: 同じ環境変数検索をマニフェストの `setup.providers[].envVars` に
    反映します。これにより、セットアップとステータスの環境メタデータが1か所に統合され、
    環境変数検索に応答するためだけにPluginランタイムを起動する必要がなくなります。

    `providerAuthEnvVars` は受け付けられなくなりました。

  </Accordion>

  <Accordion title="メモリPluginの登録 -> registerMemoryCapability">
    **旧**: 3つの個別呼び出し - `api.registerMemoryPromptSection(...)`、
    `api.registerMemoryFlushPlan(...)`、`api.registerMemoryRuntime(...)`。

    **新**: メモリ状態APIに対する1回の呼び出し -
    `registerMemoryCapability(pluginId, { promptBuilder, flushPlanResolver, runtime })`。

    スロットは同じで、登録呼び出しは1回です。追加型のプロンプトおよびコーパスヘルパー
    （`registerMemoryPromptSupplement`、`registerMemoryCorpusSupplement`）には
    影響しません。

  </Accordion>

  <Accordion title="メモリ埋め込みプロバイダーAPI">
    **旧**: `api.registerMemoryEmbeddingProvider(...)` と
    `contracts.memoryEmbeddingProviders`。

    **新**: `api.registerEmbeddingProvider(...)` と
    `contracts.embeddingProviders`。

    汎用埋め込みプロバイダー契約はメモリ以外でも再利用でき、
    新しいプロバイダーでサポートされるパスです。既存プロバイダーの
    移行中、メモリ固有の登録APIは非推奨の互換性機能として
    引き続き接続されています。Plugin検査では、非バンドルでの使用が互換性の
    負債として報告されます。

  </Accordion>

  <Accordion title="生のチャンネル送信結果 -> OutboundDeliveryResult">
    **旧**: `ChannelSendRawResult` を通じて `{ ok, messageId, error }` を返し、
    `createRawChannelSendResultAdapter(...)` で正規化します。

    **新**: `OutboundDeliveryResult` フィールドを返し、
    `createAttachedChannelResultAdapter(...)` でチャンネルを添付します。送信失敗時は、
    エラー文字列を返すのではなく例外をスローしてください。生の結果型は、
    次のPlugin SDKメジャーリリースまで引き続き利用できます。

  </Accordion>

  <Accordion title="サブエージェントセッションメッセージ型の名称変更">
    `src/plugins/runtime/types.ts` から引き続きエクスポートされる2つのレガシー型エイリアス:

    | 旧                           | 新                              |
    | ----------------------------- | ------------------------------- |
    | `SubagentReadSessionParams`   | `SubagentGetSessionMessagesParams` |
    | `SubagentReadSessionResult`   | `SubagentGetSessionMessagesResult` |

    ランタイムメソッド `readSession` は、
    `getSessionMessages` を優先して非推奨になりました。シグネチャは同じで、
    古いメソッドは新しいメソッドを呼び出します。

  </Accordion>

  <Accordion title="削除されたセッションおよびトランスクリプトファイルAPI">
    SQLiteセッション／トランスクリプトへの切り替えにより、アクティブな
    `sessions.json` ストア、JSONLトランスクリプトパス、またはセッションファイルの
    リストを公開していたPlugin向けAPIが削除または非推奨になります。ランタイムPluginは、
    アクティブなファイルを解決または変更するのではなく、セッションIDとSDKランタイム
    ヘルパーを使用してください。

    | 移行対象サーフェス | 代替 |
    | ----------------- | ----------- |
    | 非推奨の `loadSessionStore(...)`、`updateSessionStore(...)`、および `resolveSessionStoreEntry(...)` | `getSessionEntry(...)`、`listSessionEntries(...)`、および行単位のセッション変更。 |
    | 非推奨の `resolveSessionFilePath(...)` | セッションID（`sessionKey`、`sessionId`、およびSDKランタイムターゲットヘルパー）と、現在のセッションを操作するGatewayメソッド。 |
    | 削除された `saveSessionStore(...)` | Gateway所有のセッションランタイムAPI。Pluginコードは、アクティブなストアファイルに書き込むのではなく、文書化されたランタイム／コンテキストヘルパーを通じてセッション状態を要求または変更してください。 |
    | 削除された `resolveSessionTranscriptPathInDir(...)` および `resolveAndPersistSessionFile(...)` | セッションIDと、現在のセッションを操作するGatewayメソッド。 |
    | `readLatestAssistantTextFromSessionTranscript(...)` | 現在のランタイムコンテキストによって公開されるIDベースのトランスクリプトリーダー、またはPluginがトランスクリプト所有者のパス外にある場合はGatewayの履歴／セッションメソッド。 |
    | `SessionTranscriptUpdate.sessionFile` | `agentId`、`sessionKey`、および `sessionId` を指定した `SessionTranscriptUpdate.target`。 |
    | `sessionFiles` などのメモリ同期入力 | ホストが提供するIDベースのトランスクリプト／セッションソース。ライブセッションのアクティブなJSONLファイルをクロールしないでください。 |
    | アクティブなセッション向けの `transcriptPath` または `sessionFile` という名前のランタイムオプション | ストレージに依存しないセッションIDを保持する `sessionTarget`／ランタイムターゲットオブジェクト。 |

    レガシーJSONLトランスクリプトファイルは、インポート、アーカイブ、エクスポート、
    およびサポート用の成果物として引き続き有効です。アクティブなセッションにおける
    定常状態のランタイム契約ではなくなりました。

    `v2026.7.1-beta.5` とともにリリースされた公式Pluginは、前述の4つの
    非推奨ヘルパーをインポートしていました。`openclaw/plugin-sdk/session-store-runtime` は、
    2026-10-12までそのブリッジを正確に維持します。新しいPluginは代替機能を使用する必要があります。
    `resolveStorePath(...)` は引き続きサポートされるSDKヘルパーであり、
    この非推奨化には含まれません。

    `openclaw plugins inspect --all --runtime` は、読み込みエラーまたは診断で
    これらの削除済みファイルAPIを引き続き参照している非バンドルPluginを報告します。
    `@openclaw/plugin-inspector` アドバイザリスイープでは、バージョン `0.3.17` 以降を
    使用する必要があります。これにより、外部パッケージのスキャンでも、リリース前に
    ストア全体のセッションヘルパー、セッションファイルパスヘルパー、
    レガシートランスクリプトファイルターゲット、および低レベルのトランスクリプトヘルパーが
    検出されます。

  </Accordion>

  <Accordion title="runtime.tasks.flow -> runtime.tasks.managedFlows">
    **旧**: `runtime.tasks.flow`（単数形）は、ライブのタスクフロー
    アクセサーを返していました。

    **新**: `runtime.tasks.managedFlows` は、フローから子タスクを作成、更新、
    キャンセル、または実行するPlugin向けに、管理対象TaskFlow変更ランタイムを維持します。
    PluginがDTOベースの読み取りのみを必要とする場合は、`runtime.tasks.flows` を使用してください。

    ```typescript
    // 変更前
    const flow = api.runtime.tasks.flow.fromToolContext(ctx);
    // 変更後
    const flow = api.runtime.tasks.managedFlows.fromToolContext(ctx);
    ```

    レガシーエイリアスは2026年7月に削除されました。

  </Accordion>

  <Accordion title="埋め込み拡張ファクトリ -> エージェントツール結果ミドルウェア">
    詳細は上記の[移行方法](#how-to-migrate)で説明しています。完全を期すために
    ここにも記載します。削除された埋め込みランナー専用の
    `api.registerEmbeddedExtensionFactory(...)` パスは、`contracts.agentToolResultMiddleware` 内の
    明示的なランタイムリストを持つ `api.registerAgentToolResultMiddleware(...)` に
    置き換えられます。
  </Accordion>

  <Accordion title="OpenClawSchemaType エイリアス -> OpenClawConfig">
    `OpenClawSchemaType` ルート SDK エイリアスは削除されました。正規の
    `OpenClawConfig` 名を使用してください。

    ```typescript
    // 変更前
    import type { OpenClawSchemaType } from "openclaw/plugin-sdk";
    // 変更後
    import type { OpenClawConfig } from "openclaw/plugin-sdk/config-contracts";
    ```

  </Accordion>
</AccordionGroup>

<Note>
拡張レベルの非推奨項目（`extensions/` 配下のバンドル済みチャネル／プロバイダー
Plugin 内）は、それぞれ独自の `api.ts` および `runtime-api.ts`
バレル内で追跡されます。これらはサードパーティ Plugin の契約には影響せず、
ここには記載されていません。バンドル済み Plugin のローカルバレルを直接使用している場合は、
アップグレード前にそのバレル内の非推奨コメントを確認してください。
</Note>

## Talk とリアルタイム音声の移行

リアルタイム音声、電話、会議、およびブラウザの Talk コードは、
`openclaw/plugin-sdk/realtime-voice` によってエクスポートされる単一の Talk
セッションコントローラーを共有します。このコントローラーは、共通の Talk イベントエンベロープ、
アクティブなターン状態、キャプチャ状態、出力音声状態、最近のイベント履歴、
および古いターンの拒否を管理します。プロバイダー Plugin はベンダー固有のリアルタイムセッションを
管理します。ブラウザ会議 Plugin は、セッション、ブラウザ、音声、Node ホスト、
エージェント相談、および音声通話の仕組みに `openclaw/plugin-sdk/meeting-runtime` を使用し、その後、
URL ルール、DOM スクリプト、手動アクションのマッピング、字幕、作成、およびダイヤルイン
プランのために `MeetingPlatformAdapter` を実装します。プラットフォームの REST API、
OAuth、アーティファクト、セレクター、およびワイヤー名は Plugin 内に残ります。
ブラウザ権限プランは要求された会議 URL を受け取り、各プラットフォームが正確にサポートする
オリジンだけを許可できるようにします。セッションランタイムは、ブラウザからの退出が確認された後、
プラットフォーム固有のライブ健全性も正規化する必要があります。過去の文字起こしフィールドは
残っていてもかまいませんが、退出後に字幕と音声の準備状態がアクティブなままであってはなりません。

すべてのバンドル済みサーフェスは共有コントローラー上で動作します。対象はブラウザリレー、
管理対象ルームへの引き継ぎ、音声通話リアルタイム、音声通話ストリーミング STT、Google
Meet リアルタイム、およびネイティブのプッシュトゥトークです。Gateway は
`hello-ok.features.events` で単一のライブ Talk イベントチャネル
`talk.event` を通知します。

低レベルのアダプターまたはテストフィクスチャを実装する場合を除き、新しいコードから
`createTalkEventSequencer(...)` を直接呼び出さないでください。共有コントローラーを使用することで、
ターン ID なしにターンスコープのイベントが発行されること、古い `turnEnd` /
`turnCancel` 呼び出しがより新しいアクティブなターンをクリアすることを防ぎ、
電話、会議、ブラウザリレー、管理対象ルームへの引き継ぎ、およびネイティブ Talk クライアント間で
出力音声のライフサイクルイベントの一貫性を維持できます。

公開 API の形状は次のとおりです。

```typescript
// Gateway が所有する Talk セッション API。
await gateway.request("talk.session.create", {
  mode: "realtime",
  transport: "gateway-relay",
  brain: "agent-consult",
  sessionKey: "main",
});
await gateway.request("talk.session.appendAudio", { sessionId, audioBase64 });
await gateway.request("talk.session.cancelOutput", { sessionId, reason: "barge-in" });
await gateway.request("talk.session.submitToolResult", {
  sessionId,
  callId,
  result: { status: "working" },
  options: { willContinue: true },
});
await gateway.request("talk.session.submitToolResult", {
  sessionId,
  callId,
  result: { status: "already_delivered" },
  options: { suppressResponse: true },
});
await gateway.request("talk.session.submitToolResult", { sessionId, callId, result });
await gateway.request("talk.session.close", { sessionId });

// クライアントが所有するプロバイダーセッション API。
await gateway.request("talk.client.create", {
  mode: "realtime",
  transport: "webrtc",
  brain: "agent-consult",
  sessionKey: "main",
});
await gateway.request("talk.client.toolCall", { sessionKey, callId, name, args });
await gateway.request("talk.client.steer", { sessionKey, text, mode: "steer" });
```

ブラウザが所有する WebRTC／プロバイダー WebSocket セッションでは
`talk.client.create` を使用します。これは、ブラウザがプロバイダーとのネゴシエーションと
メディア転送を所有し、Gateway が認証情報、指示、およびツールポリシーを所有するためです。
`talk.session.*` は、Gateway リレーのリアルタイム、Gateway リレーの文字起こし、
および管理対象ルームのネイティブ STT／TTS セッションに共通する Gateway 管理サーフェスです。

`talk.provider` / `talk.providers` の横にリアルタイムセレクターを
配置しているレガシー設定は、`openclaw doctor --fix` で修復する必要があります。ランタイムの
Talk は、音声／TTS プロバイダー設定をリアルタイムプロバイダー設定として再解釈しません。

サポートされる `talk.session.create` の組み合わせは、意図的に少数に限定されています。

| モード            | 転送       | ブレイン           | 所有者              | 備考                                                                                                              |
| --------------- | --------------- | --------------- | ------------------ | ------------------------------------------------------------------------------------------------------------------ |
| `realtime`      | `gateway-relay` | `agent-consult` | Gateway            | 全二重のプロバイダー音声を Gateway 経由で中継します。ツール呼び出しはエージェント相談ツールを経由します。           |
| `transcription` | `gateway-relay` | `none`          | Gateway            | ストリーミング STT のみです。呼び出し元は入力音声を送信し、文字起こしイベントを受信します。                                        |
| `stt-tts`       | `managed-room`  | `agent-consult` | ネイティブ／クライアントルーム | クライアントがキャプチャ／再生を所有し、Gateway がターン状態を所有する、プッシュトゥトークおよびトランシーバー形式のルームです。 |
| `stt-tts`       | `managed-room`  | `direct-tools`  | ネイティブ／クライアントルーム | Gateway のツールアクションを直接実行する、信頼済みファーストパーティサーフェス向けの管理者専用ルームモードです。                  |

古い `talk.realtime.*` / `talk.transcription.*` / `talk.handoff.*`
ファミリー（すべて削除済み）から移行する読者向けのメソッド対応表です。

| 旧                              | 新                                                      |
| -------------------------------- | -------------------------------------------------------- |
| `talk.realtime.session`          | `talk.client.create`                                     |
| `talk.realtime.toolCall`         | `talk.client.toolCall`                                   |
| `talk.realtime.relayAudio`       | `talk.session.appendAudio`                               |
| `talk.realtime.relayCancel`      | `talk.session.cancelOutput` または `talk.session.cancelTurn` |
| `talk.realtime.relayToolResult`  | `talk.session.submitToolResult`                          |
| `talk.realtime.relayStop`        | `talk.session.close`                                     |
| `talk.transcription.session`     | `talk.session.create({ mode: "transcription" })`         |
| `talk.transcription.relayAudio`  | `talk.session.appendAudio`                               |
| `talk.transcription.relayCancel` | `talk.session.cancelTurn`                                |
| `talk.transcription.relayStop`   | `talk.session.close`                                     |
| `talk.handoff.create`            | `talk.session.create({ transport: "managed-room" })`     |
| `talk.handoff.join`              | `talk.session.join`                                      |
| `talk.handoff.revoke`            | `talk.session.close`                                     |

統合された制御用語も、意図的に狭く限定されています。

| メソッド                          | 適用対象                                              | 契約                                                                                                                                                                                                                  |
| ------------------------------- | ------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `talk.session.appendAudio`      | `realtime/gateway-relay`、`transcription/gateway-relay` | 同じ Gateway 接続が所有するプロバイダーセッションに、base64 PCM 音声チャンクを追加します。                                                                                                                             |
| `talk.session.startTurn`        | `stt-tts/managed-room`                                  | 管理対象ルームのユーザーターンを開始します。                                                                                                                                                                                           |
| `talk.session.endTurn`          | `stt-tts/managed-room`                                  | 古いターンの検証後にアクティブなターンを終了します。                                                                                                                                                                          |
| `talk.session.cancelTurn`       | Gateway が所有するすべてのセッション                              | ターンに対してアクティブなキャプチャ／プロバイダー／エージェント／TTS の処理をキャンセルします。                                                                                                                                                                 |
| `talk.session.cancelOutput`     | `realtime/gateway-relay`                                | ユーザーターンを必ずしも終了せずに、アシスタントの音声出力を停止します。                                                                                                                                                     |
| `talk.session.submitToolResult` | `realtime/gateway-relay`                                | ブリッジが公開する非同期処理の完了後にプロバイダーのツール呼び出しを完了します。中間出力には `options.willContinue` を渡し、サポートされている場合に別のアシスタント応答を回避するには `options.suppressResponse` を渡します。 |
| `talk.session.steer`            | エージェントを基盤とする Talk セッション                              | Talk セッションから解決されたアクティブな埋め込み実行に、音声による `status`、`steer`、`cancel`、または `followup` 制御を送信します。                                                                                                 |
| `talk.session.close`            | すべての統合セッション                                    | リレーセッションを停止するか管理対象ルームの状態を取り消し、その後、統合セッション ID を破棄します。                                                                                                                                     |

これを機能させるために、コアへプロバイダーまたはプラットフォームの特殊ケースを導入しないでください。
コアは Talk セッションのセマンティクスを所有します。プロバイダー Plugin はベンダーセッションのセットアップを
所有します。音声通話と Google Meet は電話／会議アダプターを所有します。ブラウザアプリとネイティブ
アプリはデバイスのキャプチャ／再生 UX を所有します。

## 削除タイムライン

| 時期                                        | 発生すること                                                                                                                              |
| ------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------- |
| **現在**                                     | 警告に対応した非推奨のサーフェスは実行時警告を発し、リポジトリのガードはコアおよびバンドル済み Plugin からの非推奨 SDK インポートを拒否します。 |
| **所有者の決定待ち**                  | 日付のないレコードは、所有者が `removeAfter` の日付を公開するまで非推奨のままで、削除対象にはなりません。                          |
| **各互換性レコードの `removeAfter` 日付** | その特定のサーフェスが削除対象になります。日付を過ぎると `pnpm plugins:boundary-report --fail-on-eligible-compat` により CI が失敗します。    |
| **次回のメジャーリリース**                      | 日付が設定されたサーフェスは、その `removeAfter` 日付以降にのみ削除できます。日付のないレコードには、引き続き所有者の承認と公開された日付が必要です。   |

以下の残りの公開 SDK サブパスには、レジストリに基づく削除猶予期間があります。
7月30日の行は、メンテナーが事前承認した早期整理の後に削除されました。
未使用のサブパスと以前の互換性エイリアスは削除され、
バンドル専用モジュールはプライベートなローカルビルドマッピングに降格されました。

| `removeAfter` | 区分                               | SDK サブパス                                                                                                                                                                        |
| ------------- | ---------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `2026-08-15`  | 以前の互換性機能の非推奨化 | `agent-config-primitives`, `channel-logging`, `channel-secret-runtime`, `channel-streaming`, `group-access`, `inbound-reply-dispatch`, `matrix`, `text-runtime`, `zod`              |
| `2026-09-01`  | 以前の互換性機能の非推奨化 | `channel-lifecycle`, `channel-message`, `channel-reply-pipeline`, `config-runtime`, `infra-runtime`                                                                                 |
| `2026-10-01`  | メディアのレガシープロジェクション            | `agent-media-payload`、およびサブパスではない `MsgContext Media*` フィールド、チャンネルの受信メディアペイロードビルダー、`buildMediaPayload`、フックのメディアエイリアス、`{{Media*}}` テンプレート |

すべてのコア Plugin はすでに移行済みです。外部 Plugin は、
次回のメジャーリリースまでに移行する必要があります。Plugin が使用しているサーフェスについて、
期限が最も近い互換性レコードを確認するには `pnpm plugins:boundary-report` を実行してください。

## 警告を一時的に抑制する

```bash
OPENCLAW_SUPPRESS_PLUGIN_SDK_COMPAT_WARNING=1 openclaw gateway run
OPENCLAW_SUPPRESS_EXTENSION_API_WARNING=1 openclaw gateway run
```

これは一時的な回避策であり、恒久的な解決策ではありません。

## 関連項目

- [はじめに](/ja-JP/plugins/building-plugins) - 最初の Plugin を構築する
- [SDK の概要](/ja-JP/plugins/sdk-overview) - サブパスインポートの完全なリファレンス
- [チャンネル Plugin](/ja-JP/plugins/sdk-channel-plugins) - チャンネル Plugin の構築
- [プロバイダー Plugin](/ja-JP/plugins/sdk-provider-plugins) - プロバイダー Plugin の構築
- [Plugin の内部構造](/ja-JP/plugins/architecture) - アーキテクチャの詳細解説
- [Plugin マニフェスト](/ja-JP/plugins/manifest) - マニフェストスキーマのリファレンス
