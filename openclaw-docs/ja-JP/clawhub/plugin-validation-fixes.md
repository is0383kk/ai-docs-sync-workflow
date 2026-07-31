---
read_when:
    - clawhub package validate を実行し、Plugin に関する指摘を修正する必要がある
    - ClawHub が Plugin パッケージの公開を拒否した、または警告を表示した
    - リリース前に Plugin パッケージのメタデータを更新しています
summary: 公開前に ClawHub Plugin パッケージの検証で見つかった問題を修正する
title: Plugin 検証の修正
x-i18n:
    generated_at: "2026-07-26T09:34:20Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 2cb869e41c9a9f1c0725f514f6b48095eb3838bf61aaf06c2474a18192f0e819
    source_path: clawhub/plugin-validation-fixes.md
    workflow: 16
---

# Plugin 検証の修正

ClawHub は公開前に Plugin パッケージを検証し、自動パッケージスキャンの検出結果も表示できます。このページでは、Plugin 作成者がパッケージメタデータ、マニフェスト、SDK インポート、または公開済みアーティファクトで修正できる、作成者向けの検出結果について説明します。

内部の Plugin Inspector カバレッジに関する検出結果は対象外です。完全なレポートに、作成者向けの修正ガイダンスがないスキャナーメンテナンスコードが含まれている場合、それらは Plugin 作成者ではなく OpenClaw メンテナー向けです。

修正を適用した後、次を再実行します。

```bash
clawhub package validate <path-to-plugin>
```

## 作成者向けの検出結果

| コード                                    | まずはこちら                                                                                                                  |
| --------------------------------------- | --------------------------------------------------------------------------------------------------------------------------- |
| `package-json-missing`                  | [パッケージメタデータを追加する](/ja-JP/clawhub/plugin-validation-fixes#package-json-missing)                                                   |
| `package-openclaw-metadata-missing`     | [パッケージの openclaw ブロックを追加する](/ja-JP/clawhub/plugin-validation-fixes#package-openclaw-metadata-missing)                            |
| `package-openclaw-entry-missing`        | [OpenClaw パッケージのエントリポイントを宣言する](/ja-JP/clawhub/plugin-validation-fixes#package-openclaw-entry-missing)                         |
| `package-entrypoint-missing`            | [宣言したエントリポイントを公開する](/ja-JP/clawhub/plugin-validation-fixes#package-entrypoint-missing)                                  |
| `package-install-metadata-incomplete`   | [インストールメタデータを完成させる](/ja-JP/clawhub/plugin-validation-fixes#package-install-metadata-incomplete)                               |
| `package-plugin-api-compat-missing`     | [Plugin API の互換性を宣言する](/ja-JP/clawhub/plugin-validation-fixes#package-plugin-api-compat-missing)                          |
| `package-min-host-version-drift`        | [ホストの最小バージョンを一致させる](/ja-JP/clawhub/plugin-validation-fixes#package-min-host-version-drift)                                   |
| `package-manifest-version-drift`        | [パッケージとマニフェストのバージョンを一致させる](/ja-JP/clawhub/plugin-validation-fixes#package-manifest-version-drift)                          |
| `package-openclaw-unsupported-metadata` | [サポートされていない OpenClaw パッケージメタデータを削除する](/ja-JP/clawhub/plugin-validation-fixes#package-openclaw-unsupported-metadata)          |
| `package-npm-pack-unavailable`          | [npm アーティファクトをパック可能にする](/ja-JP/clawhub/plugin-validation-fixes#package-npm-pack-unavailable)                                 |
| `package-npm-pack-entrypoint-missing`   | [npm pack の出力にエントリポイントを含める](/ja-JP/clawhub/plugin-validation-fixes#package-npm-pack-entrypoint-missing)                  |
| `package-npm-pack-metadata-missing`     | [npm pack の出力にメタデータを含める](/ja-JP/clawhub/plugin-validation-fixes#package-npm-pack-metadata-missing)                       |
| `manifest-name-missing`                 | [マニフェストの表示名を追加する](/ja-JP/clawhub/plugin-validation-fixes#manifest-name-missing)                                           |
| `manifest-unknown-fields`               | [サポートされていないマニフェストフィールドを削除する](/ja-JP/clawhub/plugin-validation-fixes#manifest-unknown-fields)                                  |
| `manifest-unknown-contracts`            | [サポートされていないコントラクトキーを削除する](/ja-JP/clawhub/plugin-validation-fixes#manifest-unknown-contracts)                                 |
| `legacy-root-sdk-import`                | [ルート SDK インポートを置き換える](/ja-JP/clawhub/plugin-validation-fixes#legacy-root-sdk-import)                                             |
| `reserved-sdk-import`                   | [予約済み SDK インポートを削除する](/ja-JP/clawhub/plugin-validation-fixes#reserved-sdk-import)                                             |
| `sdk-load-session-store`                | [セッションストア全体へのアクセスを置き換える](/ja-JP/clawhub/plugin-validation-fixes#sdk-load-session-store)                                   |
| `sdk-session-store-write`               | [セッションストア全体への書き込みを置き換える](/ja-JP/clawhub/plugin-validation-fixes#sdk-session-store-write)                                  |
| `sdk-session-file-helper`               | [セッションファイルパスヘルパーを置き換える](/ja-JP/clawhub/plugin-validation-fixes#sdk-session-file-helper)                                   |
| `sdk-session-transcript-file-target`    | [従来のトランスクリプトファイルターゲットを置き換える](/ja-JP/clawhub/plugin-validation-fixes#sdk-session-transcript-file-target)                   |
| `sdk-session-transcript-low-level`      | [低レベルのトランスクリプトヘルパーを置き換える](/ja-JP/clawhub/plugin-validation-fixes#sdk-session-transcript-low-level)                       |
| `legacy-before-agent-start`             | [before_agent_start を置き換える](/ja-JP/clawhub/plugin-validation-fixes#legacy-before-agent-start)                                        |
| `provider-auth-env-vars`                | [プロバイダーの環境変数をセットアップメタデータへ移動する](/ja-JP/clawhub/plugin-validation-fixes#provider-auth-env-vars)                             |
| `channel-env-vars`                      | [現在のメタデータにチャンネル環境変数を反映する](/ja-JP/clawhub/plugin-validation-fixes#channel-env-vars)                                |
| `security-manifest-schema-unavailable`  | [利用できないセキュリティマニフェストスキーマ参照を削除する](/ja-JP/clawhub/plugin-validation-fixes#security-manifest-schema-unavailable) |
| `unrecognized-security-manifest`        | [サポートされていないセキュリティマニフェストファイルを削除する](/ja-JP/clawhub/plugin-validation-fixes#unrecognized-security-manifest)                   |

## パッケージメタデータ

### package-json-missing

パッケージルートに `package.json` が含まれていないため、ClawHub は npm パッケージ、バージョン、エントリポイント、または OpenClaw メタデータを識別できません。

- `name`、`version`、および `type` を指定した `package.json` を追加します。
- パッケージに OpenClaw Plugin が含まれる場合は、`openclaw` ブロックを追加します。
- 最小限のパッケージ例については [Plugin の構築](/ja-JP/plugins/building-plugins)を、パッケージとマニフェストの区分については [Plugin マニフェスト](/ja-JP/plugins/manifest#manifest-versus-packagejson)を参照してください。
- `clawhub package validate <path-to-plugin>` を再実行します。

### package-openclaw-metadata-missing

パッケージには `package.json` がありますが、OpenClaw パッケージメタデータが宣言されていません。

- `package.json#openclaw` を追加します。
- `openclaw.extensions` や `openclaw.runtimeExtensions` などのエントリポイントメタデータを含めます。
- パッケージを ClawHub を通じて公開またはインストールする場合は、互換性とインストールのメタデータを追加します。
- [検出に影響する package.json フィールド](/ja-JP/plugins/manifest#packagejson-fields-that-affect-discovery)を参照してください。
- `clawhub package validate <path-to-plugin>` を再実行します。

### package-openclaw-entry-missing

パッケージメタデータは存在しますが、OpenClaw ランタイムのエントリポイントが宣言されていません。

- ネイティブ Plugin のエントリポイントには `openclaw.extensions` を追加します。
- 公開済みパッケージでビルド済み JavaScript を読み込む場合は、`openclaw.runtimeExtensions` を追加します。
- すべてのエントリポイントパスをパッケージディレクトリ内に保持します。
- [Plugin のエントリポイント](/ja-JP/plugins/sdk-entrypoints)および[検出に影響する package.json フィールド](/ja-JP/plugins/manifest#packagejson-fields-that-affect-discovery)を参照してください。
- `clawhub package validate <path-to-plugin>` を再実行します。

### package-entrypoint-missing

パッケージでは OpenClaw エントリポイントが宣言されていますが、参照先ファイルが検証対象のパッケージにありません。

- `openclaw.extensions`、`openclaw.runtimeExtensions`、`openclaw.setupEntry`、および `openclaw.runtimeSetupEntry` の各パスを確認します。
- エントリポイントが `dist` に生成される場合は、パッケージをビルドします。
- エントリポイントが移動した場合は、メタデータを更新します。
- [Plugin のエントリポイント](/ja-JP/plugins/sdk-entrypoints)を参照してください。
- `clawhub package validate <path-to-plugin>` を再実行します。

### package-install-metadata-incomplete

ClawHub は、パッケージをどのようにインストールまたは更新すべきか判断できません。

- `clawhubSpec`、`npmSpec`、`localPath` など、サポートされているインストール元を `openclaw.install` に設定します。
- 複数のインストール元を利用できる場合は、`openclaw.install.defaultChoice` を設定します。
- OpenClaw ホストの最小バージョンには `openclaw.install.minHostVersion` を使用します。
- [検出に影響する package.json フィールド](/ja-JP/plugins/manifest#packagejson-fields-that-affect-discovery)を参照してください。
- `clawhub package validate <path-to-plugin>` を再実行します。

### package-plugin-api-compat-missing

パッケージでは、サポートする OpenClaw Plugin API の範囲が宣言されていません。

- `package.json` に `openclaw.compat.pluginApi` を追加します。
- ビルドおよびテストの対象とした OpenClaw Plugin API のバージョンまたは semver の下限を使用します。
- これはパッケージバージョンとは別に管理します。パッケージバージョンは Plugin リリースを表し、`openclaw.compat.pluginApi` はホスト API コントラクトを表します。
- [検出に影響する package.json フィールド](/ja-JP/plugins/manifest#packagejson-fields-that-affect-discovery)を参照してください。
- `clawhub package validate <path-to-plugin>` を再実行します。

### package-min-host-version-drift

パッケージのホスト最小バージョンが、そのパッケージのビルド対象となった OpenClaw バージョンのメタデータと一致しません。

- `openclaw.install.minHostVersion` を確認します。
- リリース時に使用した OpenClaw バージョンなど、パッケージ内の OpenClaw ビルドメタデータを確認します。
- ホストの最小バージョンを、パッケージが実際にサポートするホストバージョン範囲と一致させます。
- [検出に影響する package.json フィールド](/ja-JP/plugins/manifest#packagejson-fields-that-affect-discovery)を参照してください。
- `clawhub package validate <path-to-plugin>` を再実行します。

### package-manifest-version-drift

パッケージバージョンと Plugin マニフェストのバージョンが一致しません。

- パッケージのリリースバージョンとして `package.json#version` を優先します。
- `openclaw.plugin.json` に `version` も含まれている場合は、一致するように更新するか、パッケージメタデータが正となる場合は古いマニフェストバージョンのメタデータを削除します。
- 公開済みメタデータを変更した後は、新しいパッケージバージョンを公開します。
- [Plugin マニフェスト](/ja-JP/plugins/manifest)を参照してください。
- `clawhub package validate <path-to-plugin>` を再実行します。

### package-openclaw-unsupported-metadata

`package.json#openclaw` ブロックに、OpenClaw パッケージメタデータとしてサポートされていないフィールドが含まれています。

- `openclaw.bundle` など、サポートされていないフィールドを削除します。
- ネイティブ Plugin のメタデータは `openclaw.plugin.json` に保持します。
- パッケージのエントリポイント、互換性、インストール、セットアップ、およびカタログのメタデータは、サポートされている `package.json#openclaw` フィールドに保持します。
- [検出に影響する package.json フィールド](/ja-JP/plugins/manifest#packagejson-fields-that-affect-discovery)を参照してください。
- `clawhub package validate <path-to-plugin>` を再実行します。

## 公開済みアーティファクト

### package-npm-pack-unavailable

パッケージを、ClawHub が検査または公開するアーティファクトとしてパックできません。

- パッケージルートから `npm pack --dry-run` を実行します。
- パックを失敗させる、無効なパッケージメタデータ、壊れたライフサイクルスクリプト、または files エントリを修正します。
- このパッケージを一般公開する場合は、`private: true` を削除します。
- `clawhub package validate <path-to-plugin>` を再実行します。

### package-npm-pack-entrypoint-missing

パッケージはパックできますが、パック済みアーティファクトに `package.json#openclaw` で宣言されたエントリポイントファイルが含まれていません。

- `npm pack --dry-run` を実行し、含まれる予定のファイルを確認します。
- パックする前に、生成されるエントリポイントをビルドします。
- `files`、`.npmignore`、またはビルド出力を更新し、宣言されたエントリポイントが含まれるようにします。
- [Plugin のエントリポイント](/ja-JP/plugins/sdk-entrypoints)を参照してください。
- `clawhub package validate <path-to-plugin>` を再実行します。

### package-npm-pack-metadata-missing

パック済みアーティファクトに、ソースパッケージには存在する OpenClaw メタデータがありません。

- `npm pack --dry-run` を実行し、含まれているメタデータファイルを確認します。
- パックされた成果物に、`package.json` の `openclaw` ブロックが含まれていることを確認します。
- パッケージがネイティブ OpenClaw plugin の場合は、`openclaw.plugin.json` が含まれていることを確認します。
- パッケージメタデータが除外されないように、`files` または `.npmignore` を更新します。
- [Plugin のビルド](/ja-JP/plugins/building-plugins)を参照してください。
- `clawhub package validate <path-to-plugin>` を再実行します。

## マニフェストメタデータ

### manifest-name-missing

ネイティブ plugin のマニフェストに表示名が含まれていません。

- `openclaw.plugin.json` に空でない `name` フィールドを追加します。
- `name` は人が読める形式にし、`id` は安定したマシン ID として維持します。
- [Plugin マニフェスト](/ja-JP/plugins/manifest)を参照してください。
- `clawhub package validate <path-to-plugin>` を再実行します。

### manifest-unknown-fields

plugin のマニフェストに、OpenClaw がサポートしていないトップレベルフィールドがあります。

- 各トップレベルフィールドを[マニフェストフィールドリファレンス](/ja-JP/plugins/manifest#top-level-field-reference)と比較します。
- `openclaw.plugin.json` からカスタムフィールドを削除します。
- パッケージまたはインストールのメタデータを、マニフェストではなく、サポートされている `package.json#openclaw` フィールドに移動します。
- `clawhub package validate <path-to-plugin>` を再実行します。

### manifest-unknown-contracts

マニフェストの `contracts` 内で、サポートされていないキーが宣言されています。

- `contracts` 配下の各キーを[コントラクトリファレンス](/ja-JP/plugins/manifest#contracts-reference)と比較します。
- サポートされていないコントラクトキーを削除します。
- ランタイム動作を plugin 登録コードに移動し、`contracts` は静的な機能所有権メタデータのみに限定します。
- `clawhub package validate <path-to-plugin>` を再実行します。

## SDK と互換性の移行

### legacy-root-sdk-import

plugin が非推奨のルート SDK バレルからインポートしています：
`openclaw/plugin-sdk`。

- ルートバレルからのインポートを、対象を絞った公開サブパスからのインポートに置き換えます。
- `definePluginEntry` には `openclaw/plugin-sdk/plugin-entry` を使用します。
- チャネルエントリヘルパーには `openclaw/plugin-sdk/channel-core` を使用します。
- [インポート規約](/ja-JP/plugins/building-plugins#import-conventions)と [Plugin SDK サブパス](/ja-JP/plugins/sdk-subpaths)を使用して、対象を絞ったインポートを見つけます。
- `clawhub package validate <path-to-plugin>` を再実行します。

### reserved-sdk-import

plugin が、バンドル済み plugin または内部互換性のために予約された SDK パスをインポートしています。

- 予約された OpenClaw 内部 SDK インポートを、文書化された公開 `openclaw/plugin-sdk/*` サブパスに置き換えます。
- 該当する動作に公開 SDK がない場合は、ヘルパーをパッケージ内に保持するか、公開 OpenClaw API をリクエストします。
- [Plugin SDK サブパス](/ja-JP/plugins/sdk-subpaths)と [SDK の移行](/ja-JP/plugins/sdk-migration)を使用して、サポートされているインポートを選択します。
- `clawhub package validate <path-to-plugin>` を再実行します。

### sdk-load-session-store

plugin が非推奨のセッションストア全体を扱うヘルパー `loadSessionStore` を引き続き使用しています。

- セッション状態の読み取りには、`getSessionEntry(...)` または `listSessionEntries(...)` を使用します。
- セッション状態の書き込みには、`patchSessionEntry(...)` または `upsertSessionEntry(...)` を使用します。
- セッションストアオブジェクト全体の読み込み、変更、保存は避けます。
- 宣言した互換性範囲が、それを必要とする古い OpenClaw バージョンを引き続きサポートしている間だけ、`loadSessionStore(...)` を維持します。
- [ランタイム API](/ja-JP/plugins/sdk-runtime#agent-session-state)と [Plugin SDK サブパス](/ja-JP/plugins/sdk-subpaths)を参照してください。
- `clawhub package validate <path-to-plugin>` を再実行します。

### sdk-session-store-write

plugin が、`saveSessionStore` や `updateSessionStore` など、非推奨のセッションストア全体を書き込むヘルパーを引き続き使用しています。

- 既存のセッションエントリのフィールドを更新する場合は、`patchSessionEntry(...)` を使用します。
- セッションエントリを置換または作成する場合は、`upsertSessionEntry(...)` を使用します。
- セッションストアオブジェクト全体の読み込み、変更、保存は避けます。
- 宣言した互換性範囲が、それらを必要とする古い OpenClaw バージョンを引き続きサポートしている間だけ、ストア全体の書き込みヘルパーを維持します。
- [ランタイム API](/ja-JP/plugins/sdk-runtime#agent-session-state)と [Plugin SDK サブパス](/ja-JP/plugins/sdk-subpaths)を参照してください。
- `clawhub package validate <path-to-plugin>` を再実行します。

### sdk-session-file-helper

plugin が、`resolveSessionFilePath` や `resolveAndPersistSessionFile` など、非推奨のセッションファイルパスヘルパーを引き続き使用しています。

- エージェントとセッションの ID に基づいてセッションメタデータを読み取るには、`getSessionEntry(...)` を使用します。
- セッションメタデータを永続化するには、`patchSessionEntry(...)` または `upsertSessionEntry(...)` を使用します。
- コードがトランスクリプト操作を準備している場合は、トランスクリプト ID またはターゲットヘルパーを使用します。
- 従来のトランスクリプトファイルパスを永続化したり、それに依存したりしないでください。
- [ランタイム API](/ja-JP/plugins/sdk-runtime#agent-session-state)と [Plugin SDK サブパス](/ja-JP/plugins/sdk-subpaths)を参照してください。
- `clawhub package validate <path-to-plugin>` を再実行します。

### sdk-session-transcript-file-target

plugin が非推奨のトランスクリプトファイルターゲットヘルパー `resolveSessionTranscriptLegacyFileTarget` を引き続き使用しています。

- コードが公開セッション ID のみを必要とする場合は、`resolveSessionTranscriptIdentity(...)` を使用します。
- コードが構造化されたトランスクリプト操作ターゲットを必要とする場合は、`resolveSessionTranscriptTarget(...)` を使用します。
- 従来のトランスクリプトファイルターゲットを直接読み取ったり構築したりすることは避けます。
- 宣言した互換性範囲が、それを必要とする古い OpenClaw バージョンを引き続きサポートしている間だけ、従来のヘルパーを維持します。
- [ランタイム API](/ja-JP/plugins/sdk-runtime#agent-session-state)と [Plugin SDK サブパス](/ja-JP/plugins/sdk-subpaths)を参照してください。
- `clawhub package validate <path-to-plugin>` を再実行します。

### sdk-session-transcript-low-level

plugin が、`appendSessionTranscriptMessage` や `emitSessionTranscriptUpdate` など、非推奨の低レベルトランスクリプトヘルパーを引き続き使用しています。

- トランスクリプトへの追記には `appendSessionTranscriptMessageByIdentity(...)` を使用します。
- トランスクリプト更新通知には `publishSessionTranscriptUpdateByIdentity(...)` を使用します。
- OpenClaw が正しいトランザクション境界と ID 処理を適用できるように、構造化されたトランスクリプトランタイムサーフェスを優先します。
- 宣言した互換性範囲が、それらを必要とする古い OpenClaw バージョンを引き続きサポートしている間だけ、低レベルトランスクリプトヘルパーを維持します。
- [ランタイム API](/ja-JP/plugins/sdk-runtime#agent-session-state)と [Plugin SDK サブパス](/ja-JP/plugins/sdk-subpaths)を参照してください。
- `clawhub package validate <path-to-plugin>` を再実行します。

### legacy-before-agent-start

plugin が従来の `before_agent_start` フックを引き続き使用しています。

- モデルまたはプロバイダーのオーバーライド処理を `before_model_resolve` に移動します。
- プロンプトまたはコンテキストの変更処理を `before_prompt_build` に移動します。
- 宣言した互換性範囲が、それを必要とする古い OpenClaw バージョンを引き続きサポートしている間だけ、`before_agent_start` を維持します。
- [フック](/ja-JP/plugins/hooks)と [Plugin の互換性](/ja-JP/plugins/compatibility)を参照してください。
- `clawhub package validate <path-to-plugin>` を再実行します。

### provider-auth-env-vars

マニフェストが従来の `providerAuthEnvVars` プロバイダー認証メタデータを引き続き使用しています。

- プロバイダーの環境変数メタデータを `setup.providers[].envVars` に反映します。
- サポート対象の OpenClaw 範囲で引き続き必要な間だけ、`providerAuthEnvVars` を互換性メタデータとして維持します。
- [セットアップリファレンス](/ja-JP/plugins/manifest#setup-reference)と [SDK の移行](/ja-JP/plugins/sdk-migration)を参照してください。
- `clawhub package validate <path-to-plugin>` を再実行します。

### channel-env-vars

マニフェストが、ClawHub で想定されている現在のセットアップまたは設定メタデータを伴わない、従来または旧形式のチャネル環境変数メタデータを使用しています。

- OpenClaw がチャネルランタイムを読み込まずにセットアップ状態を確認できるように、チャネル環境変数メタデータを宣言的に維持します。
- 環境変数駆動のチャネルセットアップを、plugin の構成で使用される現在のセットアップ、チャネル設定、またはパッケージのチャネルメタデータに反映します。
- サポート対象の古い OpenClaw バージョンで引き続き必要な間だけ、`channelEnvVars` を互換性メタデータとして維持します。
- [Plugin マニフェスト](/ja-JP/plugins/manifest)と [チャネル plugin](/ja-JP/plugins/sdk-channel-plugins)を参照してください。
- `clawhub package validate <path-to-plugin>` を再実行します。

## セキュリティマニフェスト

### security-manifest-schema-unavailable

パッケージに含まれる `openclaw.security.json` が、ClawHub で利用可能と認識されていないスキーマを参照しています。

- スキーマ URL が参考情報にすぎない場合は削除します。
- OpenClaw が公開した後にのみ、文書化されたバージョン付きスキーマを使用します。
- `clawhub package validate <path-to-plugin>` を再実行します。

### unrecognized-security-manifest

パッケージにサポートされていないセキュリティマニフェストファイルが含まれています。

- OpenClaw がバージョン付きセキュリティマニフェストスキーマと ClawHub の動作を文書化するまで、`openclaw.security.json` を削除します。
- マニフェストのコントラクトが確立されるまで、セキュリティ上重要な動作を公開パッケージのドキュメントまたは README に記載します。
- `clawhub package validate <path-to-plugin>` を再実行します。

## 関連項目

- [ClawHub CLI](/ja-JP/clawhub/cli)
- [ClawHub への公開](/ja-JP/clawhub/publishing)
- [Plugin のビルド](/ja-JP/plugins/building-plugins)
- [Plugin マニフェスト](/ja-JP/plugins/manifest)
- [Plugin エントリポイント](/ja-JP/plugins/sdk-entrypoints)
- [Plugin の互換性](/ja-JP/plugins/compatibility)
