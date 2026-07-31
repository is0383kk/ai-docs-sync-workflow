---
read_when:
    - DeepSeek'i OpenClaw ile kullanmak istiyorsunuz
    - API anahtarı ortam değişkeni veya CLI kimlik doğrulama seçeneği gereklidir
summary: DeepSeek kurulumu (kimlik doğrulama + model seçimi)
title: DeepSeek
x-i18n:
    generated_at: "2026-07-26T23:57:31Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 77e074756d593205d7d05f499da93b9bd3c63acdce7092b42fb5562023577925
    source_path: providers/deepseek.md
    workflow: 16
---

[DeepSeek](https://www.deepseek.com), OpenAI uyumlu bir API ile güçlü yapay zekâ modelleri sunar.

| Özellik   | Değer                      |
| --------- | -------------------------- |
| Sağlayıcı | `deepseek`         |
| Kimlik doğrulama | `DEEPSEEK_API_KEY` |
| API       | OpenAI uyumlu              |
| Temel URL | `https://api.deepseek.com`         |

## Plugin'i yükleme

Resmî Plugin'i yükleyin, ardından Gateway'i yeniden başlatın:

```bash
openclaw plugins install @openclaw/deepseek-provider
openclaw gateway restart
```

## Başlarken

<Steps>
  <Step title="API anahtarınızı alın">
    [platform.deepseek.com](https://platform.deepseek.com/api_keys) adresinde bir API anahtarı oluşturun.
  </Step>
  <Step title="İlk kurulumu çalıştırın">
    ```bash
    openclaw onboard --auth-choice deepseek-api-key
    ```

    API anahtarınızı ister ve `deepseek/deepseek-v4-flash` modelini varsayılan model olarak ayarlar.

  </Step>
  <Step title="Modellerin kullanılabilir olduğunu doğrulayın">
    ```bash
    openclaw models list --provider deepseek
    ```

    Çalışan bir Gateway olmadan Plugin'in statik kataloğunu incelemek için:

    ```bash
    openclaw models list --all --provider deepseek
    ```

  </Step>
</Steps>

<AccordionGroup>
  <Accordion title="Etkileşimsiz kurulum">
    Betik tabanlı veya ekransız kurulumlarda tüm bayrakları doğrudan iletin:

    ```bash
    openclaw onboard --non-interactive \
      --mode local \
      --auth-choice deepseek-api-key \
      --deepseek-api-key "$DEEPSEEK_API_KEY" \
      --skip-health \
      --accept-risk
    ```

  </Accordion>
</AccordionGroup>

<Warning>
Gateway bir arka plan hizmeti (launchd/systemd) olarak çalışıyorsa `DEEPSEEK_API_KEY`
değerinin bu işlem tarafından kullanılabildiğinden emin olun (örneğin
`~/.openclaw/.env` içinde veya `env.shellEnv` aracılığıyla).
</Warning>

## Yerleşik katalog

| Model referansı              | Ad                | Girdi | Bağlam    | Azami çıktı | Notlar                                              |
| ---------------------------- | ----------------- | ----- | --------- | ----------- | --------------------------------------------------- |
| `deepseek/deepseek-v4-flash` | DeepSeek V4 Flash | metin | 1,000,000 | 384,000     | Varsayılan model; V4 düşünme destekli yüzey          |
| `deepseek/deepseek-v4-pro`   | DeepSeek V4 Pro   | metin | 1,000,000 | 384,000     | V4 düşünme destekli yüzey                            |
| `deepseek/deepseek-chat`     | DeepSeek Chat     | metin | 1,000,000 | 384,000     | Kullanımdan kaldırılan V4 Flash düşünmesiz uyumluluk adı |
| `deepseek/deepseek-reasoner` | DeepSeek Reasoner | metin | 1,000,000 | 384,000     | Kullanımdan kaldırılan V4 Flash düşünmeli uyumluluk adı |

<Warning>
DeepSeek, `deepseek-chat` ve `deepseek-reasoner` modellerini 24 Temmuz 2026
tarihinde saat 15:59 UTC'de kullanımdan kaldıracaktır. Bunlar şu anda sırasıyla
düşünmesiz ve düşünmeli modlarda DeepSeek V4 Flash'a yönlendirilir. Son tarihten önce
yapılandırılmış model referanslarını `deepseek/deepseek-v4-flash` veya `deepseek/deepseek-v4-pro`
olarak değiştirin.
</Warning>

OpenClaw'ın yerel maliyet tahminleri, DeepSeek'in yayımladığı önbellek isabeti,
önbellek ıskalaması ve çıktı ücretlerini izler. DeepSeek bu ücretleri değiştirebilir;
faturalandırma konusunda [Modeller ve Fiyatlandırma](https://api-docs.deepseek.com/quick_start/pricing/)
sayfası belirleyicidir.

<Tip>
V4 modelleri DeepSeek'in `thinking` denetimini destekler. OpenClaw ayrıca,
araç çağrıları içeren düşünme oturumlarının devam edebilmesi için sonraki turlarda
DeepSeek `reasoning_content` verisini yeniden oynatır.
DeepSeek'in azami `reasoning_effort` değerini istemek için DeepSeek V4 modelleriyle
`/think xhigh` veya `/think max` kullanın; her ikisi de
`"max"` değerine eşlenir.
</Tip>

## Düşünme ve araçlar

DeepSeek V4 düşünme oturumlarında, düşünmenin etkin olduğu bir turdan yeniden
oynatılan asistan mesajlarının sonraki isteklerde `reasoning_content` içermesi gerekir.
OpenClaw'ın DeepSeek Plugin'i bu alanı otomatik olarak tamamlar; böylece geçmiş başka
bir OpenAI uyumlu sağlayıcıdan (yerel `reasoning_content` olmadan) veya düz bir asistan
mesajından gelmiş olsa bile `deepseek/deepseek-v4-flash` ve `deepseek/deepseek-v4-pro` üzerinde normal
çok turlu araç kullanımı çalışır. Oturumun ortasında sağlayıcı değiştirdikten sonra
`/new` gerekmez.

Düşünme devre dışı bırakıldığında (kullanıcı arayüzündeki **None** seçimi dâhil)
OpenClaw, `thinking: { type: "disabled" }` gönderir ve yeniden oynatılan `reasoning_content`
verilerini giden geçmişten çıkararak oturumu DeepSeek'in düşünmesiz yolunda tutar.

Varsayılan hızlı yol için `deepseek/deepseek-v4-flash` kullanın. Daha yüksek maliyeti veya
gecikmeyi kabul edebiliyorsanız daha güçlü model için `deepseek/deepseek-v4-pro` kullanın.

## Canlı test

Modern canlı model paketinden yalnızca doğrudan DeepSeek V4 model denetimlerini çalıştırmak için:

```bash
OPENCLAW_LIVE_PROVIDERS=deepseek \
OPENCLAW_LIVE_MODELS="deepseek/deepseek-v4-flash,deepseek/deepseek-v4-pro" \
pnpm test:live src/agents/models.profiles.live.test.ts
```

Her iki V4 modelinin de tamamlandığını ve düşünme/araç sonraki turlarının
DeepSeek'in gerektirdiği yeniden oynatma yükünü koruduğunu doğrular.

## Yapılandırma örneği

```json5
{
  env: { DEEPSEEK_API_KEY: "sk-..." },
  agents: {
    defaults: {
      model: { primary: "deepseek/deepseek-v4-flash" },
    },
  },
}
```

## İlgili

<CardGroup cols={2}>
  <Card title="Model seçimi" href="/tr/concepts/model-providers" icon="layers">
    Sağlayıcıları, model referanslarını ve yük devretme davranışını seçme.
  </Card>
  <Card title="Yapılandırma referansı" href="/tr/gateway/configuration-reference" icon="gear">
    Aracılar, modeller ve sağlayıcılar için tam yapılandırma referansı.
  </Card>
</CardGroup>
