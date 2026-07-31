---
read_when:
    - Together AI'ı OpenClaw ile kullanmak istiyorsunuz
    - API anahtarı ortam değişkenine veya CLI kimlik doğrulama seçeneğine ihtiyacınız var
summary: Together AI kurulumu (kimlik doğrulama + model seçimi)
title: Together AI
x-i18n:
    generated_at: "2026-07-26T23:58:53Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 9b08cae93c1ea7df46e1d2fbe78692f73bb3e56809122f70a56eec8b3dc5d8a4
    source_path: providers/together.md
    workflow: 16
---

[Together AI](https://together.ai), birleşik bir API aracılığıyla Llama, DeepSeek, Kimi ve daha fazlası dâhil olmak üzere önde gelen açık kaynaklı
modellere erişim sağlar.
OpenClaw bunu `together` sağlayıcısı olarak paketler.

| Özellik   | Değer                         |
| --------- | ----------------------------- |
| Sağlayıcı | `together`            |
| Kimlik doğrulama | `TOGETHER_API_KEY`    |
| API       | OpenAI uyumlu                 |
| Temel URL | `https://api.together.xyz/v1`            |

## Başlarken

<Steps>
  <Step title="API anahtarı edinin">
    [api.together.ai/settings/api-keys](https://api.together.ai/settings/api-keys)
    adresinde bir API anahtarı oluşturun.
  </Step>
  <Step title="İlk kurulumu çalıştırın">
    ```bash
    openclaw onboard --auth-choice together-api-key
    ```
  </Step>
  <Step title="Varsayılan model ayarlayın">
    ```json5
    {
      agents: {
        defaults: {
          model: {
            primary: "together/meta-llama/Llama-3.3-70B-Instruct-Turbo",
          },
        },
      },
    }
    ```
  </Step>
</Steps>

### Etkileşimsiz örnek

```bash
openclaw onboard --non-interactive \
  --mode local \
  --auth-choice together-api-key \
  --together-api-key "$TOGETHER_API_KEY"
```

<Note>
İlk kurulum, `together/meta-llama/Llama-3.3-70B-Instruct-Turbo` modelini
varsayılan model olarak ayarlar.
</Note>

## Yerleşik katalog

Maliyet, milyon token başına USD cinsindendir.

| Model referansı                                    | Ad                           | Girdi       | Bağlam  | Azami çıktı | Maliyet (girdi/çıktı) | Notlar                    |
| -------------------------------------------------- | ---------------------------- | ----------- | ------- | ----------- | --------------------- | ------------------------- |
| `together/meta-llama/Llama-3.3-70B-Instruct-Turbo` | Llama 3.3 70B Instruct Turbo | metin       | 131,072 | 8,192       | 0.88 / 0.88           | Varsayılan model          |
| `together/moonshotai/Kimi-K2.6`                    | Kimi K2.6 FP4                | metin, görsel | 262,144 | 32,768    | 1.20 / 4.50           | Akıl yürütme modeli       |
| `together/deepseek-ai/DeepSeek-V4-Pro`             | DeepSeek V4 Pro              | metin       | 512,000 | 8,192       | 2.10 / 4.40           | Akıl yürütme modeli       |
| `together/Qwen/Qwen2.5-7B-Instruct-Turbo`          | Qwen2.5 7B Instruct Turbo    | metin       | 32,768  | 8,192       | 0.30 / 0.30           | Hızlı, akıl yürütmesiz    |
| `together/zai-org/GLM-5.1`                         | GLM 5.1 FP4                  | metin       | 202,752 | 8,192       | 1.40 / 4.40           | Akıl yürütme modeli       |

## Video oluşturma

Paketlenmiş `together` Plugin'i, paylaşılan `video_generate` aracı üzerinden video oluşturma özelliğini de kaydeder.

| Özellik              | Değer                                                                                     |
| -------------------- | ----------------------------------------------------------------------------------------- |
| Varsayılan video modeli | `Wan-AI/Wan2.2-T2V-A14B`                                                                     |
| Diğer modeller       | `Wan-AI/Wan2.2-I2V-A14B`, `minimax/hailuo-02`, `kwaivgI/kling-2.1-master`                                |
| Modlar               | metinden videoya; yalnızca `Wan-AI/Wan2.2-I2V-A14B` ile görselden videoya (tek referans görseli) |
| Süre                 | 1-10 saniye                                                                               |
| Desteklenen parametreler | `size` (`<width>x<height>` olarak ayrıştırılır); `aspectRatio`/`resolution` okunmaz |

Together'ı varsayılan video sağlayıcısı olarak kullanmak için:

```json5
{
  agents: {
    defaults: {
      videoGenerationModel: {
        primary: "together/Wan-AI/Wan2.2-T2V-A14B",
      },
    },
  },
}
```

<Tip>
Paylaşılan araç parametreleri, sağlayıcı seçimi ve yük devretme davranışı için
[Video oluşturma](/tr/tools/video-generation) bölümüne bakın.
</Tip>

<AccordionGroup>
  <Accordion title="Ortam notu">
    Gateway bir arka plan hizmeti (launchd/systemd) olarak çalışıyorsa
    `TOGETHER_API_KEY` değerinin bu işlem tarafından kullanılabildiğinden emin olun
    (örneğin `~/.openclaw/.env` içinde veya `env.shellEnv` aracılığıyla).

    <Warning>
    Yalnızca etkileşimli kabuğunuzda ayarlanan anahtarlar, arka plan hizmeti tarafından
    yönetilen Gateway işlemleri tarafından görülemez. Kalıcı kullanılabilirlik için
    `~/.openclaw/.env` veya `env.shellEnv` yapılandırmasını kullanın.
    </Warning>

  </Accordion>

  <Accordion title="Sorun giderme">
    - Anahtarınızın çalıştığını doğrulayın: `openclaw models list --provider together`
    - Modeller görünmüyorsa API anahtarının Gateway işleminiz için doğru
      ortamda ayarlandığını doğrulayın.
    - Model referansları `together/<model-id>` biçimini kullanır.

  </Accordion>
</AccordionGroup>

## İlgili içerikler

<CardGroup cols={2}>
  <Card title="Model sağlayıcıları" href="/tr/concepts/model-providers" icon="layers">
    Sağlayıcı kuralları, model referansları ve yük devretme davranışı.
  </Card>
  <Card title="Video oluşturma" href="/tr/tools/video-generation" icon="video">
    Paylaşılan video oluşturma aracı parametreleri ve sağlayıcı seçimi.
  </Card>
  <Card title="Yapılandırma referansı" href="/tr/gateway/configuration-reference" icon="gear">
    Sağlayıcı ayarları dâhil eksiksiz yapılandırma şeması.
  </Card>
  <Card title="Together AI" href="https://together.ai" icon="arrow-up-right-from-square">
    Together AI kontrol paneli, API belgeleri ve fiyatlandırması.
  </Card>
</CardGroup>
