---
read_when:
    - 実行時にシークレット参照を再解決する
    - 平文の残存物と未解決の参照を監査する
    - SecretRefs の設定と一方向のスクラブ変更の適用
summary: '`openclaw secrets` の CLI リファレンス（再読み込み、監査、設定、適用）'
title: シークレット
x-i18n:
    generated_at: "2026-07-26T10:09:21Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 61f6f81e358ca2e6a97ac9498186b32f7a74d16052d226c398dad0030d47211e
    source_path: cli/secrets.md
    workflow: 16
---

# `openclaw secrets`

SecretRef を管理し、アクティブなランタイムスナップショットを健全な状態に保ちます。

| コマンド     | 役割                                                                                                                                                                                         |
| ----------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `reload`    | Gateway RPC（`secrets.reload`）：参照を再解決し、所有者を認識するランタイムスナップショットをアトミックに公開します（設定への書き込みなし）。対象となる所有者の障害は、コールドまたはステイルの警告として公開される場合があります |
| `audit`     | 設定、認証、生成済みモデルの各ストアとレガシー残留物を読み取り専用でスキャンし、平文、未解決の参照、優先順位のずれを検出します（`--allow-exec` でない限り exec 参照はスキップされます）                      |
| `configure` | プロバイダーのセットアップ、ターゲットのマッピング、プリフライトのための対話型プランナー（TTY が必要）                                                                                                       |
| `apply`     | 保存済みプランを実行し（`--dry-run` は検証のみを行い、デフォルトで exec チェックをスキップします。書き込みモードでは、`--allow-exec` でない限り exec を含むプランを拒否します）、その後、対象となる平文の残留物を消去します |

推奨されるオペレーターの実行手順：

```bash
openclaw secrets audit --check
openclaw secrets configure
openclaw secrets apply --from /tmp/openclaw-secrets-plan.json --dry-run
openclaw secrets apply --from /tmp/openclaw-secrets-plan.json
openclaw secrets audit --check
openclaw secrets reload
```

プランに `exec` SecretRef/プロバイダーが含まれる場合は、ドライランと書き込みの両方の `apply` コマンドに `--allow-exec` を渡してください。

CI/ゲート用の終了コード：

- `audit --check` は検出事項がある場合に `1` を返します。
- 未解決の参照は、`--check` に関係なく `2` を返します。

関連：[シークレット管理](/ja-JP/gateway/secrets) · [SecretRef 認証情報サーフェス](/ja-JP/reference/secretref-credential-surface) · [セキュリティ](/ja-JP/gateway/security)

## ランタイムスナップショットの再読み込み

```bash
openclaw secrets reload
openclaw secrets reload --json
openclaw secrets reload --url ws://127.0.0.1:18789 --token <token>
```

Gateway RPC メソッド `secrets.reload` を使用します。健全な所有者は個別に更新されます。対象となる障害が発生した所有者は、その参照 ID、プロバイダー定義、およびシークレット以外の所有者契約全体が変更されていない場合にのみステイルになります。新規または変更された障害はコールドになります。この縮退アクティベーションは成功し、`warningCount` を報告します。厳格な障害またはマッピングされていない障害はエラーを返し、それまでアクティブだったスナップショットを保持します。

オプション：`--url <url>`、`--token <token>`、`--timeout <ms>`、`--json`。

## 監査

OpenClaw の状態をスキャンして、以下を検出します：

- 平文でのシークレット保存
- 未解決の参照
- 優先順位のずれ（`auth-profiles.json` 認証情報が `openclaw.json` 参照を隠している状態）
- 生成済みの `agents/*/agent/models.json` 残留物（プロバイダーの `apiKey` 値と機密性の高いプロバイダーヘッダー）
- レガシー残留物（レガシー認証ストアのエントリ、OAuth のリマインダー）

`.env` スキャンは、有効な状態ディレクトリとアクティブな設定を含むディレクトリを対象とします。両方のパスが同じファイルを指す場合、スキャンは一度だけ行われます。

機密性の高いプロバイダーヘッダーの検出は、名前に基づくヒューリスティックを使用します。名前が一般的な認証または認証情報の断片（`authorization`、`x-api-key`、`token`、`secret`、`password`、`credential`）に一致するヘッダーを検出します。

```bash
openclaw secrets audit
openclaw secrets audit --check
openclaw secrets audit --json
openclaw secrets audit --allow-exec
```

レポートの形式：

- `status`：`clean | findings | unresolved`
- `resolution`：`refsChecked`、`skippedExecRefs`、`resolvabilityComplete`
- `summary`：`plaintextCount`、`unresolvedRefCount`、`shadowedRefCount`、`legacyResidueCount`
- 検出コード：`PLAINTEXT_FOUND`、`REF_UNRESOLVED`、`REF_SHADOWED`、`LEGACY_RESIDUE`

## 設定（対話型ヘルパー）

プロバイダーと SecretRef の変更を対話形式で構築し、プリフライトを実行して、必要に応じて適用します：

```bash
openclaw secrets configure
openclaw secrets configure --plan-out /tmp/openclaw-secrets-plan.json
openclaw secrets configure --apply --yes
openclaw secrets configure --providers-only
openclaw secrets configure --skip-provider-setup
openclaw secrets configure --agent ops
openclaw secrets configure --json
```

フロー：最初にプロバイダーをセットアップし（`secrets.providers` エイリアスの追加、編集、削除）、次に認証情報をマッピングし（フィールドを選択して `{source, provider, id}` 参照を割り当て）、その後プリフライトと任意の適用を行います。

フラグ：

- `--providers-only`：`secrets.providers` のみを設定し、認証情報のマッピングをスキップします
- `--skip-provider-setup`：プロバイダーのセットアップをスキップし、認証情報を既存のプロバイダーにマッピングします
- `--agent <id>`：`auth-profiles.json` ターゲットの検出と書き込みを 1 つのエージェントストアに限定します
- `--allow-exec`：プリフライトまたは適用中の exec SecretRef チェックを許可します（プロバイダーコマンドが実行される場合があります）

`--providers-only` と `--skip-provider-setup` は併用できません。

注記：

- 対話型 TTY が必要です。
- 選択したエージェントスコープについて、`openclaw.json` 内のシークレットを含むフィールドと `auth-profiles.json` を対象とします。正規のサポート対象サーフェス：[SecretRef 認証情報サーフェス](/ja-JP/reference/secretref-credential-surface)。
- 選択フロー内で新しい `auth-profiles.json` マッピングを直接作成できます。
- 適用前にプリフライト解決を実行します。
- 生成されるプランでは、消去オプション（`scrubEnv`、`scrubAuthProfilesForProviderTargets`、`scrubLegacyAuthJson`）がデフォルトで有効です。消去された平文値への適用は元に戻せません。
- `--plan-out` は、UTF-8 でシリアライズされた形式が 16 MiB（16,777,216 バイト）を超えるプランの作成を拒否します。これは `apply --from` の入力上限と一致します。
- `--apply` がない場合でも、CLI はプリフライト後に `Apply this plan now?` の入力を求めます。
- `--apply` があり、`--yes` がない場合、CLI は不可逆な移行について追加の確認を求めます。
- `--json` はプランとプリフライトレポートを出力しますが、対話型 TTY は引き続き必要です。

### exec プロバイダーの安全性

Homebrew のインストールでは、`/opt/homebrew/bin/*` 配下にシンボリックリンクされたバイナリが公開されることがよくあります。信頼できるパッケージマネージャーのパスに必要な場合にのみ、`trustedDirs`（例：`["/opt/homebrew"]`）と組み合わせて `allowSymlinkCommand: true` を設定してください。Windows でプロバイダーパスの ACL 検証が利用できない場合、OpenClaw はフェイルクローズします。信頼できるパスに限り、そのプロバイダーで `allowInsecurePath: true` を設定してパスのセキュリティチェックを回避してください。

## 保存済みプランの適用

```bash
openclaw secrets apply --from /tmp/openclaw-secrets-plan.json
openclaw secrets apply --from /tmp/openclaw-secrets-plan.json --allow-exec
openclaw secrets apply --from /tmp/openclaw-secrets-plan.json --dry-run
openclaw secrets apply --from /tmp/openclaw-secrets-plan.json --dry-run --allow-exec
openclaw secrets apply --from /tmp/openclaw-secrets-plan.json --json
```

`--dry-run` はファイルを書き込まずにプリフライトを検証します。ドライランでは、exec SecretRef チェックがデフォルトでスキップされます。書き込みモードでは、`--allow-exec` でない限り、exec SecretRef/プロバイダーを含むプランを拒否します。いずれのモードでも exec プロバイダーのチェックまたは実行を明示的に有効にするには、`--allow-exec` を使用します。

`--from` は 16 MiB（16,777,216 バイト）以下の通常ファイルを指す必要があります。このバイト上限は、空白を含むシリアライズ済みファイル全体に適用されます。

`apply` が更新する可能性があるもの：

- `openclaw.json`（SecretRef ターゲットとプロバイダーの upsert/削除）
- `auth-profiles.json`（プロバイダーターゲットの消去）
- レガシーの `auth.json` 残留物
- 値が移行された既知のシークレットキーについて、有効な状態ディレクトリとアクティブな設定ディレクトリ内の `.env` ファイル

プラン契約の詳細（許可されるターゲットパス、検証ルール、失敗時のセマンティクス）：[シークレット適用プラン契約](/ja-JP/gateway/secrets-plan-contract)。

### ロールバック用バックアップがない理由

`secrets apply` は意図的に、古い平文値を含むロールバック用バックアップを書き込みません。安全性は、厳格なプリフライトと準アトミックな適用、および失敗時のベストエフォートによるメモリ内復元によって確保されます。

## 例

```bash
openclaw secrets audit --check
openclaw secrets configure
openclaw secrets audit --check
```

`audit --check` が引き続き平文の検出事項を報告する場合は、報告された残りのターゲットパスを更新し、監査を再実行してください。

## 関連項目

- [CLI リファレンス](/ja-JP/cli)
- [シークレット管理](/ja-JP/gateway/secrets)
- [Vault SecretRef](/ja-JP/plugins/vault)
