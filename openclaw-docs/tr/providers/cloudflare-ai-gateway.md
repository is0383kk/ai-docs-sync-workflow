---
read_when:
    - Cloudflare AI Gateway'i OpenClaw ile kullanmak istiyorsunuz
    - Hesap kimliği, Gateway kimliği veya API anahtarı ortam değişkeni gereklidir
summary: Cloudflare AI Gateway kurulumu (kimlik doğrulama + model seçimi)
title: Cloudflare AI gateway'i
x-i18n:
    generated_at: "2026-07-27T00:14:48Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 02c7785616e7aee645bb3fc41ef6a3585e1f2f9d886fab1a06231e497effd045
    source_path: providers/cloudflare-ai-gateway.md
    workflow: 16
---

[Cloudflare AI Gateway](https://developers.cloudflare.com/ai-gateway/), sağlayıcı API'lerinin önünde yer alır ve analiz, önbelleğe alma ve denetimler ekler. Anthropic için OpenClaw, Gateway uç noktanız üzerinden Anthropic Messages API'sini kullanır.

| Özellik       | Değer                                                                                    |
| ------------- | ---------------------------------------------------------------------------------------- |
| Sağlayıcı     | `cloudflare-ai-gateway`                                                                  |
| Plugin        | resmi harici paket (`@openclaw/cloudflare-ai-gateway-provider`)                   |
| Temel URL     | `https://gateway.ai.cloudflare.com/v1/<account_id>/<gateway_id>/anthropic`               |
| Varsayılan model | `cloudflare-ai-gateway/claude-sonnet-4-6`                                                |
| API anahtarı  | `CLOUDFLARE_AI_GATEWAY_API_KEY` (Gateway üzerinden yapılan istekler için sağlayıcı API anahtarınız) |

<Note>
Cloudflare AI Gateway üzerinden yönlendirilen Anthropic modellerinde sağlayıcı anahtarı olarak **Anthropic API anahtarınızı** kullanın.
</Note>

Anthropic Messages modellerinde düşünme etkinleştirildiğinde OpenClaw, yükü Cloudflare AI Gateway üzerinden göndermeden önce sondaki
asistan ön doldurma turlarını kaldırır.
Anthropic, genişletilmiş düşünmeyle yanıt ön doldurmayı reddederken sıradan
düşünmesiz ön doldurma kullanılabilir durumda kalır.

## Plugin'i yükleme

Resmi Plugin'i yükleyin, ardından Gateway'i yeniden başlatın:

```bash
openclaw plugins install @openclaw/cloudflare-ai-gateway-provider
openclaw gateway restart
```

## Başlarken

<Steps>
  <Step title="Sağlayıcı API anahtarını ve Gateway ayrıntılarını ayarlayın">
    İlk katılımı çalıştırın ve Cloudflare AI Gateway kimlik doğrulama seçeneğini belirleyin:

    ```bash
    openclaw onboard --auth-choice cloudflare-ai-gateway-api-key
    ```

    Bu işlem hesap kimliğinizi, gateway kimliğinizi ve API anahtarınızı girmenizi ister.

  </Step>
  <Step title="Varsayılan model ayarlayın">
    Modeli OpenClaw yapılandırmanıza ekleyin:

    ```json5
    {
      agents: {
        defaults: {
          model: { primary: "cloudflare-ai-gateway/claude-sonnet-4-6" },
        },
      },
    }
    ```

  </Step>
  <Step title="Modelin kullanılabilir olduğunu doğrulayın">
    ```bash
    openclaw models list --provider cloudflare-ai-gateway
    ```
  </Step>
</Steps>

## Etkileşimsiz örnek

Betik tabanlı veya CI kurulumlarında tüm değerleri komut satırında iletin:

```bash
openclaw onboard --non-interactive \
  --mode local \
  --auth-choice cloudflare-ai-gateway-api-key \
  --cloudflare-ai-gateway-account-id "your-account-id" \
  --cloudflare-ai-gateway-gateway-id "your-gateway-id" \
  --cloudflare-ai-gateway-api-key "$CLOUDFLARE_AI_GATEWAY_API_KEY"
```

## Gelişmiş yapılandırma

<AccordionGroup>
  <Accordion title="Kimliği doğrulanmış gateway'ler">
    Cloudflare'de Gateway kimlik doğrulamasını etkinleştirdiyseniz `cf-aig-authorization` üst bilgisini ekleyin. Bu, sağlayıcı API anahtarınıza **ek olarak** gereklidir.

    ```json5
    {
      models: {
        providers: {
          "cloudflare-ai-gateway": {
            headers: {
              "cf-aig-authorization": "Bearer <cloudflare-ai-gateway-token>",
            },
          },
        },
      },
    }
    ```

    <Tip>
    `cf-aig-authorization` üst bilgisi Cloudflare Gateway'in kendisinde kimlik doğrulaması yaparken sağlayıcı API anahtarı (örneğin Anthropic anahtarınız) üst akış sağlayıcısında kimlik doğrulaması yapar.
    </Tip>

  </Accordion>

  <Accordion title="Ortam notu">
    Gateway bir daemon (launchd/systemd) olarak çalışıyorsa `CLOUDFLARE_AI_GATEWAY_API_KEY` değerinin bu işlem tarafından kullanılabildiğinden emin olun.

    <Warning>
    Yalnızca etkileşimli bir kabukta dışa aktarılan anahtar, söz konusu ortam oraya da aktarılmadığı sürece launchd/systemd daemon'una yardımcı olmaz. Gateway işleminin anahtarı okuyabilmesini sağlamak için anahtarı `~/.openclaw/.env` içinde veya `env.shellEnv` aracılığıyla ayarlayın.
    </Warning>

  </Accordion>
</AccordionGroup>

## İlgili

<CardGroup cols={2}>
  <Card title="Model seçimi" href="/tr/concepts/model-providers" icon="layers">
    Sağlayıcıları, model referanslarını ve yük devretme davranışını seçme.
  </Card>
  <Card title="Sorun giderme" href="/tr/help/troubleshooting" icon="wrench">
    Genel sorun giderme ve SSS.
  </Card>
</CardGroup>
