---
read_when:
    - Thinking Machines Lab'in Inkling'ini OpenClaw'da çalıştırmak istiyorsunuz
    - Baseten'in barındırdığı modeller için OpenAI uyumlu tek bir API istiyorsunuz
summary: Inkling ve barındırılan Model API'leri için temel kurulum
title: Baseten
x-i18n:
    generated_at: "2026-07-27T00:14:51Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 2ccc3b5cf64b01859f9f022d7bc15a69a1cb42c87d4f914c118276c1151020de
    source_path: providers/baseten.md
    workflow: 16
---

[Baseten Model API'leri](https://docs.baseten.co/inference/model-apis/overview), öncü modellere barındırılan ve OpenAI uyumlu erişim sağlar. Resmî harici Plugin, kimliği doğrulanmış keşif kullandığından OpenClaw, Baseten hesabınız için etkinleştirilmiş eksiksiz model kümesini izler. Çevrimdışı geri dönüşü, bu OpenClaw sürümü oluşturulduğunda kullanılabilen tüm Model API'lerini içerir.

| Özellik        | Değer                                                    |
| --------------- | -------------------------------------------------------- |
| Sağlayıcı kimliği     | `baseten`                                                |
| Plugin          | resmî harici paket (`@openclaw/baseten-provider`) |
| Kimlik doğrulama ortam değişkeni    | `BASETEN_API_KEY`                                        |
| İlk kurulum bayrağı | `--auth-choice baseten-api-key`                          |
| Doğrudan CLI bayrağı | `--baseten-api-key <key>`                                |
| API             | OpenAI uyumlu (`openai-completions`)                 |
| Temel URL        | `https://inference.baseten.co/v1`                        |
| Varsayılan model   | `baseten/thinkingmachines/inkling`                       |

## Plugin'i yükleme

```bash
openclaw plugins install @openclaw/baseten-provider
openclaw gateway restart
```

## Başlarken

<Steps>
  <Step title="Bir Baseten hesabı ve API anahtarı oluşturun">
    Baseten'in Basic planında aylık platform ücreti yoktur; Model API çağrıları kullanıma göre ücretlendirilir. [Baseten API key settings](https://app.baseten.co/settings/api_keys) bölümünde bir anahtar oluşturun ve güncel ücretleri [fiyatlandırma sayfasından](https://www.baseten.co/pricing) kontrol edin.
  </Step>
  <Step title="İlk kurulumu çalıştırın">
    <CodeGroup>

```bash Onboarding
openclaw onboard --auth-choice baseten-api-key
```

```bash Direct flag
openclaw onboard --non-interactive \
  --auth-choice baseten-api-key \
  --baseten-api-key "$BASETEN_API_KEY"
```

```bash Env only
export BASETEN_API_KEY=...
```

    </CodeGroup>

  </Step>
  <Step title="Canlı kataloğu doğrulayın">
    ```bash
    openclaw models list --provider baseten
    ```

    Kullanılabilir kimlik doğrulama bilgileriyle Plugin, `GET /v1/models` isteğinde bulunur ve hesap için döndürülen tüm modelleri listeler. Kimlik doğrulama bilgileri olmadan çevrimdışı kalır ve paketlenmiş geri dönüşü kullanır.

  </Step>
</Steps>

## Inkling

[Thinking Machines Lab'in Inkling modeli](https://thinkingmachines.ai/news/introducing-inkling/) varsayılan modeldir. OpenClaw'da metin ve görüntü girdisini, araç çağırmayı, yapılandırılmış araç şemalarını, yapılandırılabilir akıl yürütme eforunu, 1.048M tokenlık bağlam penceresini ve en fazla 32k çıktı tokenını destekler:

```json5
{
  agents: {
    defaults: {
      model: { primary: "baseten/thinkingmachines/inkling" },
    },
  },
}
```

Mevcut bir sohbeti değiştirmek için `/model baseten/thinkingmachines/inkling` kullanın.

## Paketlenmiş geri dönüş kataloğu

Kimliği doğrulanmış canlı katalog belirleyicidir. Bu satırlar, keşif başarılı olmadan önce kurulumu ve model seçimini kullanılabilir durumda tutar:

| Model referansı                                          | Girdi       | Bağlam | En fazla çıktı |
| -------------------------------------------------- | ----------- | ------: | ---------: |
| `baseten/deepseek-ai/DeepSeek-V4-Pro`              | metin        |    262k |       262k |
| `baseten/zai-org/GLM-4.7`                          | metin        |    200k |       200k |
| `baseten/zai-org/GLM-5`                            | metin        |    202k |       202k |
| `baseten/zai-org/GLM-5.1`                          | metin        |    202k |       202k |
| `baseten/zai-org/GLM-5.2`                          | metin        |    202k |       202k |
| `baseten/thinkingmachines/inkling`                 | metin, görüntü |  1.048M |        32k |
| `baseten/moonshotai/Kimi-K2.5`                     | metin, görüntü |    262k |       262k |
| `baseten/moonshotai/Kimi-K2.6`                     | metin, görüntü |    262k |       262k |
| `baseten/moonshotai/Kimi-K2.7-Code`                | metin, görüntü |    262k |       262k |
| `baseten/nvidia/Nemotron-120B-A12B`                | metin        |    202k |       202k |
| `baseten/nvidia/NVIDIA-Nemotron-3-Ultra-550B-A55B` | metin        |    202k |       202k |
| `baseten/openai/gpt-oss-120b`                      | metin        |    128k |       128k |

Paketlenmiş tüm modeller araç çağırmayı ve akıl yürütmeyi destekler. OpenClaw, düşünme düzeylerini yerel `reasoning_effort` özelliğine sahip modellerle eşler. Baseten'in isteğe bağlı GLM, Kimi ve Nemotron modellerinde düşünme varsayılan olarak kapalıdır; çoğu ikili kapalı/açık denetimi sunarken GLM 5.2 kapalı, yüksek ve maksimum seçeneklerini sunar. OpenClaw bu seçimleri Baseten'in `chat_template_args.enable_thinking` denetimi ve GLM 5.2 için doğrulanmış üst düzey `reasoning_effort` parametresi üzerinden gönderir.

<Note>
Baseten, OpenClaw sürümlerinden bağımsız olarak Model API'leri ekleyebilir, kaldırabilir veya değiştirebilir. Plugin, modele özgü OpenClaw aktarım politikasını korurken model kimliklerini, bağlam sınırlarını, çıktı sınırlarını ve girdi, önbelleğe alınmış girdi ve çıktı fiyatlandırmasını kimliği doğrulanmış API'den yeniler.
</Note>

## Manuel yapılandırma

Çoğu kurulum yalnızca API anahtarına ihtiyaç duyar. Sağlayıcıyı açıkça sabitlemek için:

```json5
{
  env: { BASETEN_API_KEY: "..." },
  agents: {
    defaults: {
      model: { primary: "baseten/thinkingmachines/inkling" },
    },
  },
  models: {
    mode: "merge",
    providers: {
      baseten: {
        baseUrl: "https://inference.baseten.co/v1",
        apiKey: "${BASETEN_API_KEY}",
        api: "openai-completions",
        models: [
          {
            id: "thinkingmachines/inkling",
            name: "Inkling",
            reasoning: true,
            input: ["text", "image"],
            contextWindow: 1048000,
            maxTokens: 32000,
          },
        ],
      },
    },
  },
}
```

<Note>
Gateway bir arka plan programı (launchd, systemd, Docker) olarak çalışıyorsa `BASETEN_API_KEY` değişkeninin bu işlem tarafından kullanılabildiğinden emin olun. Yalnızca etkileşimli bir kabukta dışa aktarılan anahtar, zaten çalışmakta olan yönetilen bir hizmet tarafından görülemez.
</Note>

## İlgili

<CardGroup cols={2}>
  <Card title="Model sağlayıcıları" href="/tr/concepts/model-providers" icon="layers">
    Sağlayıcıları, model referanslarını ve yük devretme davranışını seçme.
  </Card>
  <Card title="Düşünme modları" href="/tr/tools/thinking" icon="brain">
    OpenClaw akıl yürütme eforu düzeylerini seçin.
  </Card>
  <Card title="Modeller CLI'si" href="/tr/cli/models" icon="terminal">
    Keşfedilen modelleri listeleyin, inceleyin ve seçin.
  </Card>
  <Card title="Modeller hakkında SSS" href="/tr/help/faq-models" icon="circle-question">
    Kimlik doğrulama profilleri ve model seçimi sorunlarını giderme.
  </Card>
</CardGroup>
