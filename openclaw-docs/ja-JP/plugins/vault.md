---
read_when:
    - OpenClaw が HashiCorp Vault から API キーを読み取るようにしたい場合
    - ローカルマシンまたはサーバーで SecretRefs を設定しています
    - Vault を基盤とするモデルプロバイダーの認証情報を設定する必要があります
summary: バンドルされた Vault Plugin を使用して、HashiCorp Vault から SecretRefs を解決する
title: Vault SecretRef 系列
x-i18n:
    generated_at: "2026-07-26T09:38:56Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: c1fa4895414e8cf44bb4ada191a7f7aa7b4eeda58f16be04d0c77080b7af96e3
    source_path: plugins/vault.md
    workflow: 16
---

# Vault SecretRef

バンドルされた Vault Plugin を使用すると、OpenClaw は Gateway の起動時およびリロード時に、HashiCorp Vault から `exec` SecretRef を解決できます。OpenClaw は Vault への参照を設定に保存し、解決された値をメモリ内のシークレットスナップショットに保持します。解決された API キーを `openclaw.json` に書き戻すことはありません。

すでに Vault を運用している場合や、モデルプロバイダーのキーを OpenClaw の設定ファイル外に保存したい場合に使用します。SecretRef のランタイムモデルについては、[シークレット管理](/ja-JP/gateway/secrets)を参照してください。

## 始める前に

必要なもの：

- バンドルされた `vault` Plugin を利用できる OpenClaw
- 到達可能な Vault サーバー
- OpenClaw が解決するシークレットパスへの読み取りアクセス権を持つクライアントトークンを生成できる Vault 認証
- Gateway を起動する環境には、`VAULT_ADDR` と、`VAULT_TOKEN`、`VAULT_TOKEN_FILE` を伴う `OPENCLAW_VAULT_AUTH_METHOD=token_file`、または設定済みの JWT/Kubernetes ログインのいずれかが必要

リゾルバーは Node から HTTP 経由で Vault と通信します。Gateway が SecretRef を解決するために Vault CLI は必要ありません。

`openclaw vault` コマンドを実行する前に、バンドルされた Plugin を有効にします。

```bash
openclaw plugins enable vault
```

## プロバイダーキーを Vault に保存する

OpenClaw のデフォルトは `secret` にマウントされた KV v2 で、Vault 開発サーバーの例と一致します。本番環境の Vault では、SecretRef ID を作成する前に、`OPENCLAW_VAULT_KV_MOUNT` を実際の KV マウントパスに設定してください。OpenClaw のデフォルトでは、次の SecretRef ID：

```text
providers/openrouter/apiKey
```

は、次の Vault フィールドを読み取ります。

```text
secret/data/providers/openrouter -> apiKey
```

Vault CLI で作成する方法の一例：

```bash
export OPENROUTER_API_KEY=<openrouter-api-key>
vault kv put secret/providers/openrouter apiKey="$OPENROUTER_API_KEY"
```

OpenClaw には root トークンではなく、スコープを限定したクライアントトークンを使用してください。デフォルトの KV v2 レイアウトでは、モデルプロバイダーキーに対する最小限のポリシーは次のようになります。

```hcl
path "secret/data/providers/*" {
  capabilities = ["read"]
}
```

## Gateway から Vault を利用可能にする

コンテナ化されていないローカル Gateway では、OpenClaw を起動するのと同じシェルで Vault の設定をエクスポートします。デフォルトの認証方式では、`VAULT_TOKEN` から Vault クライアントトークンを読み取ります。

```bash
export VAULT_ADDR=https://vault.example.com
export VAULT_TOKEN=<vault-client-token>
```

Vault Agent がトークンシンクファイルを書き込む場合は、トークンファイル認証を使用します。

```bash
export VAULT_ADDR=https://vault.example.com
export OPENCLAW_VAULT_AUTH_METHOD=token_file
export VAULT_TOKEN_FILE=/vault/secrets/token
```

プライベート CA によって署名された Vault サーバーでは、その CA をホストの信頼ストアにインストールし、Node のシステム信頼を有効にします。

```bash
export NODE_USE_SYSTEM_CA=1
```

または、PEM バンドルを直接指定します。

```bash
export NODE_EXTRA_CA_CERTS=/path/to/vault-ca.pem
```

これらの変数は OpenClaw の起動時に存在している必要があります。Vault Plugin は、それらをリゾルバープロセスに転送します。

非対話型 JWT 認証では、ワークロード JWT ファイルと、タイプが `jwt` の Vault ロールを使用します。

```bash
export VAULT_ADDR=https://vault.example.com
export OPENCLAW_VAULT_AUTH_METHOD=jwt
export OPENCLAW_VAULT_AUTH_MOUNT=jwt
export OPENCLAW_VAULT_AUTH_ROLE=openclaw
export OPENCLAW_VAULT_JWT_FILE=/var/run/secrets/tokens/vault
```

JWT ファイルには、Vault ロールが受け入れるオーディエンスを持つ Kubernetes サービスアカウントトークンなど、投影されたワークロードトークンを使用してください。
対話型 OIDC ブラウザログインは人間には便利ですが、Gateway ランタイムには非対話型 JWT ログインまたはトークンファイルが必要です。

Vault の Kubernetes 認証方式では、`kubernetes` を使用します。これは Pod として稼働する Gateway を対象としています。デフォルトのマウントは `kubernetes`、デフォルトの JWT ファイルは標準のサービスアカウントトークンパスです。

```bash
export VAULT_ADDR=https://vault.example.com
export OPENCLAW_VAULT_AUTH_METHOD=kubernetes
export OPENCLAW_VAULT_AUTH_ROLE=openclaw
```

Vault が Kubernetes 認証を `auth/kubernetes` 以外の場所にマウントしている場合にのみ、`OPENCLAW_VAULT_AUTH_MOUNT` を設定します。サービスアカウントトークンがカスタムパスに投影される場合にのみ、`OPENCLAW_VAULT_JWT_FILE` を設定します。

任意の設定：

```bash
export VAULT_NAMESPACE=<namespace-name>
export OPENCLAW_VAULT_KV_MOUNT=secret
export OPENCLAW_VAULT_KV_VERSION=2
```

現在のシェルから確認できる内容をチェックします。

```bash
openclaw vault status
```

Vault を使用するシークレットプロバイダーが複数設定されている場合は、エイリアスで選択します。

```bash
openclaw vault status --provider-alias corp-vault
```

`openclaw vault status` が `VAULT_TOKEN` を出力することはありません。トークン、トークンファイル、JWT ファイルが設定されているかどうかのみを報告します。

<Warning>
Gateway がサービス、LaunchAgent、systemd ユニット、スケジュール済みタスク、またはコンテナとして実行される場合、そのランタイム環境にも同じ Vault 変数を渡す必要があります。
対話型シェルで変数を設定しても、そのシェルについて確認できるだけで、すでに実行中の Gateway については確認できません。
</Warning>

## SecretRef プランを生成して適用する

OpenRouter のモデルプロバイダー API キーを Vault にマッピングするプランを作成します。

```bash
openclaw vault setup \
  --plan-out ./vault-secrets-plan.json \
  --openrouter-id providers/openrouter/apiKey
```

プランを適用して検証します。

```bash
openclaw secrets apply --from ./vault-secrets-plan.json --dry-run --allow-exec
openclaw secrets apply --from ./vault-secrets-plan.json --allow-exec
openclaw secrets audit --check --allow-exec
openclaw secrets reload
```

Vault Plugin は OpenClaw が管理する exec SecretRef プロバイダーを通じて解決するため、`--allow-exec` を使用します。

Gateway がまだ実行されていない場合は、`openclaw secrets reload` を実行する代わりに、プランを適用した後で通常どおり起動します。

## さらにプロバイダーキーを設定する

組み込みのショートカット：

```bash
openclaw vault setup --openai-id providers/openai/apiKey
openclaw vault setup --anthropic-id providers/anthropic/apiKey
openclaw vault setup --openrouter-id providers/openrouter/apiKey
```

1 つのプランに複数のプロバイダーキーを含める場合：

```bash
openclaw vault setup \
  --plan-out ./vault-secrets-plan.json \
  --openai-id providers/openai/apiKey \
  --anthropic-id providers/anthropic/apiKey \
  --openrouter-id providers/openrouter/apiKey
```

ショートカットのないバンドル済みプロバイダー、または設定済みの OpenAI 互換およびカスタムモデルプロバイダーでは、`--provider-key` を使用します。

```bash
openclaw vault setup \
  --plan-out ./vault-secrets-plan.json \
  --provider-key local-openai=providers/local-openai/apiKey \
  --provider-key groq=providers/groq/apiKey
```

各 `--provider-key <provider=id>` は、`models.providers.<provider>.apiKey` に SecretRef を書き込みます。カスタムプロバイダーでは、プロバイダーの `baseUrl`、`api`、または `models` の設定は作成されないため、先にそれらを設定してください。

既知の SecretRef ターゲットパスには、`--target <path=id>` を使用します。

```bash
openclaw vault setup \
  --target channels.telegram.botToken=channels/telegram/botToken \
  --target models.providers.openai.headers.x-api-key=providers/openai/proxyKey \
  --target auth-profiles:main:profiles.openai.key=providers/openai/apiKey
```

修飾子のないターゲットパスは `openclaw.json` に適用されます。既存の `auth-profiles.json` ターゲットには、`auth-profiles:<agentId>:<path>` を使用します。
ターゲットパスは、登録済みの OpenClaw SecretRef ターゲットである必要があります。setup コマンドは、OpenClaw に任意の名前付きシークレットを作成しません。シークレットストアは引き続き Vault であり、OpenClaw はサポート対象の設定フィールドにのみ SecretRef を保存します。

## SecretRef ID の形式

Vault SecretRef ID は次の規則を使用します。

```text
<vault-secret-path>/<field>
```

例：

| SecretRef ID                  | デフォルトの KV v2 Vault 読み取り           | 返されるフィールド |
| ----------------------------- | ---------------------------------- | -------------- |
| `providers/openrouter/apiKey` | `secret/data/providers/openrouter` | `apiKey`       |
| `providers/openai/apiKey`     | `secret/data/providers/openai`     | `apiKey`       |
| `teams/agent-prod/openrouter` | `secret/data/teams/agent-prod`     | `openrouter`   |

返される Vault フィールドは文字列である必要があります。

KV v1 では、次のように設定します。

```bash
export OPENCLAW_VAULT_KV_VERSION=1
```

その場合、`providers/openrouter/apiKey` は次を読み取ります。

```text
secret/providers/openrouter -> apiKey
```

## OpenClaw が保存する内容

Vault setup プランを適用すると、Plugin が管理するプロバイダーが保存されます。

```json
{
  "source": "exec",
  "pluginIntegration": {
    "pluginId": "vault",
    "integrationId": "vault"
  }
}
```

認証情報フィールドは、そのプロバイダーを参照します。

```json
{ "source": "exec", "provider": "vault", "id": "providers/openrouter/apiKey" }
```

解決された値は、アクティブなランタイムシークレットスナップショットにのみ存在します。

## コンテナとマネージドデプロイ

コンテナ化された Gateway でも、同じ Plugin と SecretRef 設定を使用します。コンテナには次のものを渡す必要があります。

- `VAULT_ADDR`
- いずれか 1 つの認証ソース：
  - `VAULT_TOKEN`
  - `OPENCLAW_VAULT_AUTH_METHOD=token_file` と `VAULT_TOKEN_FILE`
  - `OPENCLAW_VAULT_AUTH_METHOD=jwt` と `OPENCLAW_VAULT_AUTH_MOUNT`、`OPENCLAW_VAULT_AUTH_ROLE`、および `OPENCLAW_VAULT_JWT_FILE`
  - `OPENCLAW_VAULT_AUTH_METHOD=kubernetes` と `OPENCLAW_VAULT_AUTH_ROLE`。必要に応じて `OPENCLAW_VAULT_AUTH_MOUNT` または `OPENCLAW_VAULT_JWT_FILE` を上書き
- 任意の `VAULT_NAMESPACE`、`OPENCLAW_VAULT_KV_MOUNT`、および `OPENCLAW_VAULT_KV_VERSION`

Kubernetes を使用する場合、Vault にクラスター用の Kubernetes 認証が設定されているときは、`OPENCLAW_VAULT_AUTH_METHOD=kubernetes` を優先してください。Vault がクラスターを汎用 JWT/OIDC 発行者として扱うよう設定されている場合にのみ、`OPENCLAW_VAULT_AUTH_METHOD=jwt` を使用します。どちらの方法も、Kubernetes Secret に長期間有効な Vault トークンを保存するより適切です。Vault Agent サイドカーまたはインジェクターを使用するデプロイでは、代わりに `token_file` を使用できます。

マルチテナントの Vault 構成では、テナントのルーティングを Vault ポリシーとデプロイ設定に保持します。OpenClaw では、固定のマウント、ロール、またはパスは必要ありません。各 Gateway 環境で独自の `OPENCLAW_VAULT_KV_MOUNT`、`OPENCLAW_VAULT_AUTH_ROLE`、および SecretRef ID を設定できます。1 つの共有 Gateway で異なる Vault ユーザーを同時に解決する必要がある場合は、異なる認証環境をラップする手動設定の exec プロバイダーを使用するか、個別の Vault 環境変数を持つ Gateway 環境にテナントを分割してください。

## 関連項目

- [シークレット管理](/ja-JP/gateway/secrets)
- [`openclaw secrets`](/ja-JP/cli/secrets)
- [Plugin 一覧](/ja-JP/plugins/plugin-inventory)
