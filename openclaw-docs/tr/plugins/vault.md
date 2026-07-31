---
read_when:
    - OpenClaw'ın API anahtarlarını HashiCorp Vault'tan okumasını istiyorsunuz
    - Yerel bir makinede veya sunucuda SecretRefs ayarlıyorsunuz
    - Vault destekli model sağlayıcı kimlik bilgilerini yapılandırmanız gerekir
summary: HashiCorp Vault'tan SecretRef'leri çözümlemek için paketle birlikte gelen Vault Plugin'ini kullanın
title: Kasa SecretRef'leri
x-i18n:
    generated_at: "2026-07-26T23:34:31Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: c1fa4895414e8cf44bb4ada191a7f7aa7b4eeda58f16be04d0c77080b7af96e3
    source_path: plugins/vault.md
    workflow: 16
---

# Vault SecretRef'leri

Birlikte gelen Vault plugin'i, OpenClaw'ın Gateway başlatılırken ve yeniden yükleme sırasında
HashiCorp Vault'tan `exec` SecretRef'lerini çözümlemesini sağlar. OpenClaw, Vault
referanslarını yapılandırmada saklar, çözümlenen değerleri bellek içi gizli bilgiler anlık görüntüsünde tutar
ve çözümlenen API anahtarlarını `openclaw.json` içine geri yazmaz.

Vault'u zaten çalıştırıyorsanız veya model sağlayıcısı anahtarlarını
OpenClaw yapılandırma dosyalarının dışında tutmak istiyorsanız bunu kullanın. SecretRef çalışma zamanı modeli için
[Gizli bilgi yönetimi](/tr/gateway/secrets) bölümüne bakın.

## Başlamadan önce

Gereksinimler:

- birlikte gelen `vault` plugin'inin kullanılabildiği OpenClaw
- erişilebilir bir Vault sunucusu
- OpenClaw'ın çözümlemesi gereken gizli bilgi
  yollarına okuma erişimi olan bir istemci belirteci üretebilen Vault kimlik doğrulaması
- Gateway'i başlatan ortamda `VAULT_ADDR` ve şunlardan biri bulunmalıdır:
  `VAULT_TOKEN`, `VAULT_TOKEN_FILE` ile birlikte `OPENCLAW_VAULT_AUTH_METHOD=token_file`
  veya yapılandırılmış bir JWT/Kubernetes oturum açma yöntemi

Çözümleyici, Node üzerinden HTTP ile Vault'a bağlanır. Gateway'in SecretRef'leri çözümlemek için
Vault CLI'a ihtiyacı yoktur.

`openclaw vault` komutlarını çalıştırmadan önce birlikte gelen plugin'i etkinleştirin:

```bash
openclaw plugins enable vault
```

## Vault'ta sağlayıcı anahtarı saklama

OpenClaw varsayılan olarak `secret` konumuna bağlanan KV v2'yi kullanır; bu,
Vault geliştirme sunucusu örnekleriyle uyumludur. Üretim Vault'u için SecretRef kimlikleri oluşturmadan önce
`OPENCLAW_VAULT_KV_MOUNT` değerini gerçek KV bağlama yolunuza ayarlayın. OpenClaw varsayılanlarıyla şu
SecretRef kimliği:

```text
providers/openrouter/apiKey
```

şu Vault alanını okur:

```text
secret/data/providers/openrouter -> apiKey
```

Bunu Vault CLI ile oluşturmanın bir yolu şudur:

```bash
export OPENROUTER_API_KEY=<openrouter-api-key>
vault kv put secret/providers/openrouter apiKey="$OPENROUTER_API_KEY"
```

OpenClaw için kök belirteç değil, kapsamı sınırlandırılmış bir istemci belirteci kullanın. Varsayılan KV v2
düzeni için model sağlayıcısı anahtarlarına yönelik asgari bir politika şöyledir:

```hcl
path "secret/data/providers/*" {
  capabilities = ["read"]
}
```

## Vault'u Gateway'e görünür kılma

Kapsayıcıya alınmamış yerel bir Gateway için Vault ayarlarını OpenClaw'ı başlatan aynı kabukta
dışa aktarın. Varsayılan kimlik doğrulama yöntemi, Vault istemci belirtecini
`VAULT_TOKEN` konumundan okur:

```bash
export VAULT_ADDR=https://vault.example.com
export VAULT_TOKEN=<vault-client-token>
```

Vault Agent bir belirteç havuzu dosyası yazıyorsa belirteç dosyası kimlik doğrulamasını kullanın:

```bash
export VAULT_ADDR=https://vault.example.com
export OPENCLAW_VAULT_AUTH_METHOD=token_file
export VAULT_TOKEN_FILE=/vault/secrets/token
```

Özel bir CA tarafından imzalanmış Vault sunucusu için bu CA'yı ana makinenin
güven deposuna yükleyip Node sistem güvenini etkinleştirin:

```bash
export NODE_USE_SYSTEM_CA=1
```

Ya da doğrudan bir PEM paketi sağlayın:

```bash
export NODE_EXTRA_CA_CERTS=/path/to/vault-ca.pem
```

Bu değişkenler OpenClaw başlatıldığında mevcut olmalıdır. Vault plugin'i bunları
çözümleyici sürecine iletir.

Etkileşimsiz JWT kimlik doğrulaması için bir iş yükü JWT dosyası ve `jwt`
türünde bir Vault rolü kullanın:

```bash
export VAULT_ADDR=https://vault.example.com
export OPENCLAW_VAULT_AUTH_METHOD=jwt
export OPENCLAW_VAULT_AUTH_MOUNT=jwt
export OPENCLAW_VAULT_AUTH_ROLE=openclaw
export OPENCLAW_VAULT_JWT_FILE=/var/run/secrets/tokens/vault
```

JWT dosyası, Vault rolünün kabul ettiği bir hedef kitleye sahip Kubernetes hizmet hesabı
belirteci gibi yansıtılmış bir iş yükü belirteci olmalıdır.
Etkileşimli OIDC tarayıcı oturumu insanlar için kullanışlıdır ancak Gateway çalışma zamanı
etkileşimsiz JWT oturumu veya bir belirteç dosyası gerektirir.

Vault'un Kubernetes kimlik doğrulama yöntemi için `kubernetes` kullanın. Bu yöntem,
Pod olarak çalışan Gateway'ler için tasarlanmıştır; varsayılan bağlama noktası `kubernetes`, varsayılan JWT
dosyası ise standart hizmet hesabı belirteç yoludur:

```bash
export VAULT_ADDR=https://vault.example.com
export OPENCLAW_VAULT_AUTH_METHOD=kubernetes
export OPENCLAW_VAULT_AUTH_ROLE=openclaw
```

`OPENCLAW_VAULT_AUTH_MOUNT` değerini yalnızca Vault, Kubernetes kimlik doğrulamasını
`auth/kubernetes` dışında bir konuma bağladıysa ayarlayın. `OPENCLAW_VAULT_JWT_FILE` değerini yalnızca hizmet
hesabı belirteci özel bir yola yansıtılıyorsa ayarlayın.

İsteğe bağlı ayarlar:

```bash
export VAULT_NAMESPACE=<namespace-name>
export OPENCLAW_VAULT_KV_MOUNT=secret
export OPENCLAW_VAULT_KV_VERSION=2
```

Geçerli kabuğun neleri görebildiğini denetleyin:

```bash
openclaw vault status
```

Birden fazla Vault destekli gizli bilgi sağlayıcısı yapılandırılmışsa takma ada göre birini
seçin:

```bash
openclaw vault status --provider-alias corp-vault
```

`openclaw vault status`, `VAULT_TOKEN` değerini hiçbir zaman yazdırmaz; yalnızca
belirtecin, belirteç dosyasının ve JWT dosyasının ayarlanıp ayarlanmadığını bildirir.

<Warning>
Gateway bir hizmet, LaunchAgent, systemd birimi, zamanlanmış görev veya
kapsayıcı olarak çalışıyorsa bu çalışma zamanı ortamına aynı Vault değişkenleri aktarılmalıdır.
Değişkenlerin etkileşimli bir kabukta ayarlanması yalnızca o kabuk için kanıttır; zaten
çalışan Gateway için değildir.
</Warning>

## SecretRef planı oluşturma ve uygulama

OpenRouter'ın model sağlayıcısı API anahtarını Vault ile eşleyen bir plan oluşturun:

```bash
openclaw vault setup \
  --plan-out ./vault-secrets-plan.json \
  --openrouter-id providers/openrouter/apiKey
```

Planı uygulayın ve doğrulayın:

```bash
openclaw secrets apply --from ./vault-secrets-plan.json --dry-run --allow-exec
openclaw secrets apply --from ./vault-secrets-plan.json --allow-exec
openclaw secrets audit --check --allow-exec
openclaw secrets reload
```

Vault plugin'i OpenClaw tarafından yönetilen bir exec SecretRef sağlayıcısı üzerinden
çözümleme yaptığı için `--allow-exec` kullanın.

Gateway henüz çalışmıyorsa planı uyguladıktan sonra `openclaw secrets reload` komutunu çalıştırmak
yerine Gateway'i normal şekilde başlatın.

## Daha fazla sağlayıcı anahtarı yapılandırma

Yerleşik kısayollar:

```bash
openclaw vault setup --openai-id providers/openai/apiKey
openclaw vault setup --anthropic-id providers/anthropic/apiKey
openclaw vault setup --openrouter-id providers/openrouter/apiKey
```

Tek planda birden fazla sağlayıcı anahtarı:

```bash
openclaw vault setup \
  --plan-out ./vault-secrets-plan.json \
  --openai-id providers/openai/apiKey \
  --anthropic-id providers/anthropic/apiKey \
  --openrouter-id providers/openrouter/apiKey
```

Kısayolu olmayan birlikte gelen sağlayıcılar veya önceden yapılandırılmış OpenAI uyumlu ve
özel model sağlayıcıları `--provider-key` kullanır:

```bash
openclaw vault setup \
  --plan-out ./vault-secrets-plan.json \
  --provider-key local-openai=providers/local-openai/apiKey \
  --provider-key groq=providers/groq/apiKey
```

Her `--provider-key <provider=id>`, `models.providers.<provider>.apiKey` konumuna
bir SecretRef yazar. Özel sağlayıcılarda sağlayıcının `baseUrl`, `api`
veya `models` ayarlarını oluşturmaz; önce bunları yapılandırın.

Bilinen herhangi bir SecretRef hedef yolu için `--target <path=id>` kullanın:

```bash
openclaw vault setup \
  --target channels.telegram.botToken=channels/telegram/botToken \
  --target models.providers.openai.headers.x-api-key=providers/openai/proxyKey \
  --target auth-profiles:main:profiles.openai.key=providers/openai/apiKey
```

Öneksiz hedef yollar `openclaw.json` için geçerlidir. Mevcut
`auth-profiles.json` hedefleri için `auth-profiles:<agentId>:<path>` kullanın.
Hedef yol, kayıtlı bir OpenClaw SecretRef hedefi olmalıdır. Kurulum
komutu OpenClaw'da rastgele adlandırılmış gizli bilgiler oluşturmaz; gizli bilgi deposu Vault olarak kalır
ve OpenClaw yalnızca desteklenen yapılandırma alanlarında SecretRef'leri saklar.

## SecretRef kimliği biçimi

Vault SecretRef kimlikleri şu kuralı kullanır:

```text
<vault-secret-path>/<field>
```

Örnekler:

| SecretRef kimliği             | Varsayılan KV v2 Vault okuması     | Döndürülen alan |
| ----------------------------- | ---------------------------------- | --------------- |
| `providers/openrouter/apiKey` | `secret/data/providers/openrouter` | `apiKey`       |
| `providers/openai/apiKey`     | `secret/data/providers/openai`     | `apiKey`       |
| `teams/agent-prod/openrouter` | `secret/data/teams/agent-prod`     | `openrouter`   |

Döndürülen Vault alanı bir dize olmalıdır.

KV v1 için şunu ayarlayın:

```bash
export OPENCLAW_VAULT_KV_VERSION=1
```

Ardından `providers/openrouter/apiKey` şunu okur:

```text
secret/providers/openrouter -> apiKey
```

## OpenClaw'ın sakladıkları

Vault kurulum planının uygulanması, plugin tarafından yönetilen bir sağlayıcıyı saklar:

```json
{
  "source": "exec",
  "pluginIntegration": {
    "pluginId": "vault",
    "integrationId": "vault"
  }
}
```

Kimlik bilgisi alanları bu sağlayıcıya işaret eder:

```json
{ "source": "exec", "provider": "vault", "id": "providers/openrouter/apiKey" }
```

Çözümlenen değer yalnızca etkin çalışma zamanı gizli bilgiler anlık görüntüsünde bulunur.

## Kapsayıcılar ve yönetilen dağıtımlar

Kapsayıcıya alınmış Gateway'ler de aynı plugin'i ve SecretRef yapılandırmasını kullanır.
Kapsayıcı şunları almalıdır:

- `VAULT_ADDR`
- bir kimlik doğrulama kaynağı:
  - `VAULT_TOKEN`
  - `OPENCLAW_VAULT_AUTH_METHOD=token_file` ve `VAULT_TOKEN_FILE`
  - `OPENCLAW_VAULT_AUTH_METHOD=jwt` ile birlikte `OPENCLAW_VAULT_AUTH_MOUNT`,
    `OPENCLAW_VAULT_AUTH_ROLE` ve `OPENCLAW_VAULT_JWT_FILE`
  - `OPENCLAW_VAULT_AUTH_METHOD=kubernetes` ve `OPENCLAW_VAULT_AUTH_ROLE`; isteğe bağlı olarak
    `OPENCLAW_VAULT_AUTH_MOUNT` veya `OPENCLAW_VAULT_JWT_FILE` değerini geçersiz kılın
- isteğe bağlı `VAULT_NAMESPACE`, `OPENCLAW_VAULT_KV_MOUNT` ve
  `OPENCLAW_VAULT_KV_VERSION`

Kubernetes kullanırken Vault'ta küme için Kubernetes kimlik doğrulaması yapılandırılmışsa
`OPENCLAW_VAULT_AUTH_METHOD=kubernetes` yöntemini tercih edin. `OPENCLAW_VAULT_AUTH_METHOD=jwt` yöntemini yalnızca Vault
kümeyi genel bir JWT/OIDC yayımlayıcısı olarak ele alacak şekilde yapılandırılmışsa kullanın.
Her iki seçenek de Kubernetes Secret içindeki uzun ömürlü bir Vault belirtecinden daha iyidir.
Vault Agent yardımcı kapsayıcısı veya enjektör dağıtımları bunun yerine `token_file` kullanabilir.

Çok kiracılı Vault kurulumlarında kiracı yönlendirmesini Vault politikası ve
dağıtım yapılandırmasında tutun. OpenClaw sabit bir bağlama noktası, rol veya yol gerektirmez: her
Gateway ortamı kendi `OPENCLAW_VAULT_KV_MOUNT`, `OPENCLAW_VAULT_AUTH_ROLE`
ve SecretRef kimliklerini ayarlayabilir. Paylaşılan tek bir Gateway'in aynı anda
farklı Vault kullanıcılarını çözümlemesi gerekiyorsa farklı kimlik doğrulama ortamlarını sarmalayan, elle yapılandırılmış exec sağlayıcılarını
kullanın veya kiracıları ayrı Vault ortam değişkenlerine sahip Gateway
ortamlarına bölün.

## İlgili

- [Gizli bilgi yönetimi](/tr/gateway/secrets)
- [`openclaw secrets`](/tr/cli/secrets)
- [Plugin envanteri](/tr/plugins/plugin-inventory)
