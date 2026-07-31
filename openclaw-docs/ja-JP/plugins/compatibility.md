---
read_when:
    - OpenClaw Plugin を保守している場合
    - Plugin の互換性に関する警告が表示される
    - Plugin SDK またはマニフェストの移行を計画している場合
summary: Plugin の互換性契約、非推奨メタデータ、移行に関する要件
title: Plugin の互換性
x-i18n:
    generated_at: "2026-07-26T10:09:33Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 80cf1dfce9e0538e78138ff80a6807ee36267a07d3eee6f19bd8e56e5c0c9cd3
    source_path: plugins/compatibility.md
    workflow: 16
---

OpenClaw は、古い Plugin コントラクトを削除する前に、名前付き互換性
アダプターを通じて接続を維持します。これにより、SDK、マニフェスト、セットアップ、設定、
およびエージェントランタイムのコントラクトが進化する間も、既存のバンドル済みおよび外部
Plugin が保護されます。

## 互換性レジストリ

Plugin の互換性コントラクトは、`src/plugins/compat/registry.ts` にあるコアレジストリで
追跡されます。各レコードには以下が含まれます。

- 安定した互換性コード
- ステータス: `active`、`deprecated`、`removal-pending`、または `removed`
- 所有者: `sdk`、`config`、`setup`、`channel`、`provider`、`plugin-execution`、
  `agent-runtime`、または `core`
- 該当する場合は導入日と非推奨化日
- 担当メンテナーが承認した後の正確な削除日。`removeAfter` が省略されている
  場合、非推奨のサーフェスは削除対象になりません
- 移行先に関するガイダンス
- 旧動作と新動作を対象とするドキュメント、診断、およびテスト

このレジストリは、メンテナーによる計画と将来の Plugin
インスペクターチェックの情報源です。Plugin 向けの動作が変更された場合は、
アダプターを追加する変更と同じ変更内で互換性レコードを追加または更新してください。

Doctor の修復および移行の互換性は、
`src/commands/doctor/shared/deprecation-compat.ts` で別途追跡されます。これらのレコードは、ランタイム互換性パスの削除後も
利用可能な状態を維持する必要がある可能性のある、古い設定形式、インストール台帳のレイアウト、
および修復用の互換シムを対象とします。

リリース時の確認では、両方のレジストリをチェックする必要があります。一致するランタイムまたは
設定の互換性レコードが期限切れになったという理由だけで、Doctor の移行を削除しないでください。
まず、その修復を依然として必要とするサポート対象のアップグレードパスがないことを確認してください。
また、プロバイダーやチャネルがコア外へ移動するにつれて Plugin の所有権や設定の範囲が変わる可能性があるため、
リリース計画時には各移行先の注釈も再検証してください。

## 非推奨化ポリシー

OpenClaw は、ドキュメント化された Plugin コントラクトを、その移行先を導入するリリースと同じ
リリースで削除すべきではありません。移行手順は以下のとおりです。

1. 新しいコントラクトを追加します。
2. 名前付き互換性アダプターを通じて古い動作の接続を維持します。
3. Plugin 作者が対応可能になった時点で、診断または警告を出力します。
4. 移行先とスケジュールをドキュメント化します。
5. 新旧両方のパスをテストします。
6. 告知した移行期間が経過するまで待ちます。
7. 破壊的変更を含むリリースとして明示的に承認された場合にのみ削除します。

非推奨レコードには、警告開始日、移行先、ドキュメントへの
リンク、および警告開始から 3 か月以内の最終削除日を含める必要があります。
メンテナーが永続的な互換性であると明示的に決定し、代わりに
`active` とマークしない限り、削除期限が未定の非推奨互換性パスを追加しないでください。

## 現在の互換性領域

2026 年 7 月の確認では、期限切れとなったルート SDK、マニフェスト、プロバイダー、ランタイム、
レジストリフラグ、および Plugin 所有の Web 設定エイリアスが削除されました。サポート対象の
アップグレードパスで古い設定を引き続き修復できるよう、Doctor の移行は別途追跡されています。

日付が設定された残りの互換性領域は以下のとおりです。

- 移行ガイドに記載されている 8 月および 9 月の SDK サブパス期間
- `api.on("deactivate", ...)` および `api.on("subagent_spawning", ...)` フックエイリアス
- メモリ固有の埋め込み登録および beta.5 セッションストアブリッジ
- 以下で説明する WhatsApp の受信コールバックエイリアス
- 明示的なチャネルターゲット解析および `openclaw/plugin-sdk/messaging-targets`
- 組み込み Pi エージェントエイリアス
- 出荷済みのエージェントハーネス SDK エイリアス。これらの削除は、外部向けに
  ドキュメント化された新たな移行判断を待っています

日付が設定されていないアクティブなレジストリレコードは、削除すべき負債ではなく、サポート対象の動作を
対象としており、アクティベーションヒント、Plugin キャプチャ、バンドル済み Plugin の有効化、
および生成されたチャネル設定のフォールバックが含まれます。

### WhatsApp 受信コールバックのフラットエイリアス

WhatsApp のランタイムコールバックは `WebInboundMessage` を渡します。これには、正規の
ネストされた `event`、`payload`、`quote`、`group`、および `platform` コンテキストに加え、
出荷済みコールバックフィールド用の非推奨フラットエイリアスが含まれます。新しいコールバックコードは、
ネストされたコンテキストを読み取る必要があります。クリーンなネスト済みコールバックメッセージを
構築するコードでは `WebInboundCallbackMessage` を使用できます。古いフラット形式のテストまたは Plugin
メッセージを引き続き注入する互換性リスナーでは、
`LegacyFlatWebInboundMessage` または `WebInboundMessageInput` を使用する必要があります。

フラットエイリアスは **2026-08-30** まで利用できます。この期間が適用されるのは
フラットエイリアスへのアクセスのみであり、正規のランタイムコントラクトであるネスト形式には
適用されません。各フラットエイリアスの TypeScript `@deprecated` 注釈には、
その正確なネスト先が記載されています。一般的な例は以下のとおりです。

- `id`、`timestamp`、および `isBatched` は `event` 配下に移動します。
- `body`、`mediaPath`、`mediaType`、`mediaFileName`、`mediaUrl`、`location`、
  および `untrustedStructuredContext` は `payload` 配下に移動します。
- `to`、`chatId`、送信者/自身のフィールド、`sendComposing`、`reply(...)`、および
  `sendMedia(...)` は `platform` 配下に移動します。
- `replyTo*` フィールドは `quote` 配下に移動します。グループの件名/参加者/メンション
  フィールドは `group` 配下に移動します。

`payload.untrustedStructuredContext` は受信したプロバイダーの
ペイロードから抽出されます。Plugin は、その `payload` を信頼できる情報として
扱う前に、`label`、`source`、および `type` を確認する必要があります。

### WhatsApp 受信許可フィールド

受理された WhatsApp コールバックメッセージには、メッセージを許可したアクセス制御判断の
公開しても安全なエンベロープである `admission` が含まれます。新しい
コールバックコードは、従来のトップレベル許可フィールドではなく、
`msg.admission` から許可情報を読み取る必要があります。

トップレベルフィールドは **2026-08-30** まで利用できます。各フィールドの
TypeScript `@deprecated` 注釈には、移行先が記載されています。

- `from` および `conversationId` は `admission.conversation.id` に移動します。
- `accountId` は `admission.accountId` に移動します。
- `accessControlPassed` は、
  `admission.ingress.decision === "allow"` から派生した互換性ビューです。すでに
  `admission` を持つメッセージでは、レガシーな真偽値を書き込んでも受信
  グラフは書き換えられません。
- `chatType` は `admission.conversation.kind` に移動します。

## Plugin インスペクターパッケージ

Plugin インスペクターは、バージョン管理された互換性コントラクトと
マニフェストコントラクトに基づく独立したパッケージ/リポジトリとして、
OpenClaw のコアリポジトリ外に配置する必要があります。初期リリースの CLI は以下のとおりです。

```sh
openclaw-plugin-inspector ./my-plugin
```

マニフェスト/スキーマ検証、チェック対象のコントラクト互換性
バージョン、インストール/ソースメタデータのチェック、コールドパスのインポート
チェック、および非推奨化/互換性の警告を出力する必要があります。CI 注釈向けの安定した
機械可読出力には `--json` を使用してください。OpenClaw コアは、
インスペクターが利用できるコントラクトとフィクスチャを公開する必要がありますが、
メインの `openclaw` パッケージからインスペクターバイナリを公開すべきではありません。

### メンテナー向け受け入れレーン

外部インスペクターを OpenClaw Plugin パッケージに対して検証する際は、
インストール可能パッケージの受け入れレーンに Crabbox ベースの Blacksmith Testbox を
使用してください。パッケージのビルド後、クリーンな OpenClaw チェックアウトから実行します。

```sh
pnpm crabbox:run -- --provider blacksmith-testbox --timing-json --shell -- "pnpm install && pnpm build && npm exec --yes @openclaw/plugin-inspector@0.1.0 -- ./extensions/telegram --json"
pnpm crabbox:run -- --provider blacksmith-testbox --timing-json --shell -- "npm exec --yes @openclaw/plugin-inspector@0.1.0 -- ./extensions/discord --json"
pnpm crabbox:run -- --provider blacksmith-testbox --timing-json --shell -- "npm exec --yes @openclaw/plugin-inspector@0.1.0 -- <clawhub-plugin-dir> --json"
```

このレーンは外部 npm パッケージをインストールし、リポジトリ外にクローンされた
Plugin パッケージを検査する可能性があるため、メンテナーによるオプトイン方式を維持してください。
ローカルリポジトリのガードは、SDK エクスポートマップ、互換性レジストリのメタデータ、
非推奨 SDK インポートの段階的削減、およびバンドル済み拡張機能のインポート境界を対象とします。
Testbox によるインスペクターの検証は、外部 Plugin 作者が利用する形でパッケージを検証します。

## リリースノート

リリースノートには、互換性パスが `removal-pending` または `removed` に
移行する前に、予定されている Plugin の非推奨化について、対象日および移行ドキュメントへの
リンクとともに記載する必要があります。
