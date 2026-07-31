---
read_when:
    - '`openclaw secrets apply` プランの生成またはレビュー'
    - '`Invalid plan target path` エラーのデバッグ'
    - ターゲットの種類とパス検証の動作を理解する
summary: '`secrets apply` プランの契約: ターゲット検証、パスマッチング、`auth-profiles.json` ターゲットスコープ'
title: シークレット適用プランの契約
x-i18n:
    generated_at: "2026-07-26T09:43:33Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 71ee8afd958646930af4db3bbad08e033ff79da48890a989d72b361abcbda3bb
    source_path: gateway/secrets-plan-contract.md
    workflow: 16
---

このページでは、`openclaw secrets apply` によって適用される厳密な契約を定義します。ターゲットがこれらのルールに一致しない場合、ファイルを変更する前に適用が失敗します。

## プランファイルの要件

`openclaw secrets apply --from <plan.json>` は、最大 16 MiB（16,777,216 バイト）の通常ファイルを受け付けます。この制限は、空白を含むシリアライズ済みファイル全体に適用されます。ディレクトリ、FIFO、デバイスファイル、および制限を超えるファイルは、JSON の解析またはターゲットの検証前に拒否されます。

`openclaw secrets configure --plan-out <plan.json>` は、ファイルを作成する前に、UTF-8 でシリアライズされた出力にも同じ制限を適用します。手書きのプランと外部プランジェネレーターでも、シリアライズ済みファイルをこの制限内に収める必要があります。

## プランファイルの形式

`openclaw secrets apply --from <plan.json>` は、プランターゲットの `targets` 配列を想定します。

```json5
{
  version: 1,
  protocolVersion: 1,
  targets: [
    {
      type: "models.providers.apiKey",
      path: "models.providers.openai.apiKey",
      pathSegments: ["models", "providers", "openai", "apiKey"],
      providerId: "openai",
      ref: { source: "env", provider: "default", id: "OPENAI_API_KEY" },
    },
    {
      type: "auth-profiles.api_key.key",
      path: "profiles.openai:default.key",
      pathSegments: ["profiles", "openai:default", "key"],
      agentId: "main",
      ref: { source: "env", provider: "default", id: "OPENAI_API_KEY" },
    },
  ],
}
```

`openclaw secrets configure` は、この形式でプランを生成します。プランを手書きまたは編集することもできます。

## プロバイダーのアップサートと削除

プランには、ターゲットごとの書き込みと併せて `secrets.providers` マップを変更する、次の 2 つの任意のトップレベルフィールドを含めることもできます。

- `providerUpserts` -- プロバイダーエイリアスをキーとするオブジェクト。各値はプロバイダー定義です（`openclaw.json` の `secrets.providers.<alias>` で受け付けるものと同じ形式。たとえば、`exec` または `file` プロバイダー）。
- `providerDeletes` -- 削除するプロバイダーエイリアスの配列。

`providerUpserts` は `targets` より前に実行されるため、`target.ref.provider` は、同じプランが `providerUpserts` で導入するプロバイダーエイリアスを参照できます。この順序でない場合、`openclaw.json` にまだ設定されていないエイリアスを参照するプランは、`provider "<alias>" is not configured` で失敗します。

```json5
{
  version: 1,
  protocolVersion: 1,
  providerUpserts: {
    onepassword_anthropic: {
      source: "exec",
      command: "/usr/bin/op",
      args: ["read", "op://Vault/Anthropic/credential"],
    },
  },
  providerDeletes: ["legacy_unused_alias"],
  targets: [
    {
      type: "models.providers.apiKey",
      path: "models.providers.anthropic.apiKey",
      pathSegments: ["models", "providers", "anthropic", "apiKey"],
      providerId: "anthropic",
      ref: { source: "exec", provider: "onepassword_anthropic", id: "credential" },
    },
  ],
}
```

`providerUpserts` を介して導入された exec プロバイダーにも、[exec プロバイダーの同意動作](#exec-provider-consent-behavior)に記載された exec 同意ルールが適用されます。exec プロバイダーを含むプランでは、書き込みモードで `--allow-exec` が必要です。

## サポートされるターゲット範囲

[SecretRef 認証情報サーフェス](/ja-JP/reference/secretref-credential-surface)に記載された、サポート対象の認証情報パスのプランターゲットが受け付けられます。

## ターゲットタイプの動作

`target.type` は認識されるターゲットタイプでなければならず、正規化された `target.path` は、そのタイプに登録されたパス形式と一致する必要があります。

一部のターゲットタイプでは、正規のタイプ名に加え、既存のプラン向けに `target.type` として互換性エイリアスを受け付けます。

| 正規タイプ                       | 受け付けるエイリアス                                  |
| ------------------------------------ | ----------------------------------------------- |
| `models.providers.apiKey`            | `models.providers.*.apiKey`                     |
| `skills.entries.apiKey`              | `skills.entries.*.apiKey`                       |
| `channels.googlechat.serviceAccount` | `channels.googlechat.accounts.*.serviceAccount` |

## パス検証ルール

各ターゲットは、次のすべての条件に基づいて検証されます。

- `type` は、認識されるターゲットタイプである必要があります。
- `path` は、空でないドット区切りのパスである必要があります。
- `pathSegments` は省略できます。指定する場合は、正規化後に `path` と完全に同じパスになる必要があります。
- 禁止されているセグメントは拒否されます：`__proto__`、`prototype`、`constructor`。
- 正規化されたパスは、ターゲットタイプに登録されたパス形式と一致する必要があります。
- `providerId` または `accountId` が設定されている場合、パスにエンコードされた ID と一致する必要があります。
- `auth-profiles.json` ターゲットには `agentId` が必要です。
- 新しい `auth-profiles.json` マッピングを作成する場合は、`authProfileProvider` を含めます。

## 失敗時の動作

ターゲットの検証に失敗すると、適用処理は次のようなエラーで終了します。

```text
models.providers.apiKey のプランターゲットパスが無効です：models.providers.openai.baseUrl
```

無効なプランでは、書き込みは一切コミットされません。ターゲットの解決とパスの検証は、ファイルに触れる前に実行されます。また、有効なプランが書き込みを開始すると、適用処理は最初に変更対象のすべてのファイルのスナップショットを作成します。同じ実行内で後続の書き込みが失敗した場合は、それらのスナップショットを復元するため、部分的な書き込みによって設定、認証プロファイル、または環境変数の状態に不整合が残ることはありません。

## exec プロバイダーの同意動作

- `--dry-run` は、デフォルトで exec SecretRef のチェックをスキップします。
- exec SecretRef またはプロバイダーを含むプランは、`--allow-exec` が設定されていない限り、書き込みモードで拒否されます。
- exec を含むプランを検証または適用する場合は、ドライランと書き込みの両方のコマンドで `--allow-exec` を渡します。

## ランタイムと監査範囲に関する注意事項

- 参照のみの `auth-profiles.json` エントリ（`keyRef`/`tokenRef`）も、ランタイムの認証情報解決と監査の対象に含まれます。
- `secrets apply` は、サポート対象の `openclaw.json` ターゲット、サポート対象の `auth-profiles.json` ターゲット、およびデフォルトでそれぞれ有効な 3 つの任意のスクラブ処理を書き込みます。`scrubEnv`（有効な状態ディレクトリとアクティブ設定ディレクトリにある `.env` ファイルから、移行済みの平文値を削除）、`scrubAuthProfilesForProviderTargets`（プランで移行したプロバイダーについて、`auth-profiles.json` 内の平文および未使用参照の残存データを消去）、`scrubLegacyAuthJson`（従来の `auth.json` ストアから移行済みの `api_key` エントリを削除）です。いずれかの処理をスキップするには、プラン内の `options.scrubEnv`、`options.scrubAuthProfilesForProviderTargets`、`options.scrubLegacyAuthJson` の該当する値を `false` に設定します。

## オペレーターによる確認

```bash
# 書き込みを行わずにプランを検証
openclaw secrets apply --from /tmp/openclaw-secrets-plan.json --dry-run

# 次に実際に適用
openclaw secrets apply --from /tmp/openclaw-secrets-plan.json

# exec を含むプランでは、両方のモードで明示的にオプトイン
openclaw secrets apply --from /tmp/openclaw-secrets-plan.json --dry-run --allow-exec
openclaw secrets apply --from /tmp/openclaw-secrets-plan.json --allow-exec
```

無効なターゲットパスを示すメッセージで適用が失敗した場合は、`openclaw secrets configure` でプランを再生成するか、ターゲットパスを上記のサポート対象形式に修正します。

## 関連ドキュメント

- [シークレット管理](/ja-JP/gateway/secrets)
- [CLI `secrets`](/ja-JP/cli/secrets)
- [SecretRef 認証情報サーフェス](/ja-JP/reference/secretref-credential-surface)
- [設定リファレンス](/ja-JP/gateway/configuration-reference)
