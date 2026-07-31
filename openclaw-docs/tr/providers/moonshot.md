---
read_when:
    - Moonshot Kimi K3/K2 (Moonshot Open Platform) ile Kimi Coding kurulumunu karşılaştırmak istiyorsunuz
    - Ayrı uç noktaları, anahtarları ve model referanslarını anlamanız gerekir
    - Her iki sağlayıcı için de kopyalayıp yapıştırabileceğiniz yapılandırma istiyorsunuz
summary: Moonshot Kimi modellerini Kimi Coding'e karşı yapılandırma (ayrı sağlayıcılar + anahtarlar)
title: Moonshot AI
x-i18n:
    generated_at: "2026-07-27T00:15:32Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 213379bf88fec26b052184a920e112f0887d6485601bfb47f590cf37ef983e58
    source_path: providers/moonshot.md
    workflow: 16
---

Moonshot, OpenAI uyumlu uç noktalarla Kimi API'yi sağlar. Kimi K3 için
`moonshot/kimi-k3` seçeneğini belirleyin, ilk kurulum varsayılanı olan
`moonshot/kimi-k2.6` değerini koruyun veya Kimi Coding için `kimi/kimi-for-coding` kullanın.

<Warning>
Moonshot ve Kimi Coding, her biri ayrı bir harici eklenti olarak sunulan **ayrı sağlayıcılardır**. Anahtarlar birbirinin yerine kullanılamaz, uç noktalar farklıdır ve model referansları farklıdır (`moonshot/...` ile `kimi/...`).
</Warning>

## Yerleşik model kataloğu

[//]: # "moonshot-kimi-k2-ids:start"

| Model referansı                      | Ad                       | Akıl yürütme | Girdi       | Bağlam    | Maks. çıktı |
| ----------------------------------- | ------------------------ | ------------ | ----------- | --------- | ----------- |
| `moonshot/kimi-k2.6`                | Kimi K2.6                | Hayır        | metin, görsel | 262,144   | 262,144     |
| `moonshot/kimi-k3`                  | Kimi K3                  | Daima maksimum | metin, görsel | 1,048,576 | 1,048,576   |
| `moonshot/kimi-k2.7-code`           | Kimi K2.7 Code           | Daima açık   | metin, görsel | 262,144   | 262,144     |
| `moonshot/kimi-k2.7-code-highspeed` | Kimi K2.7 Code HighSpeed | Daima açık   | metin, görsel | 262,144   | 262,144     |
| `moonshot/kimi-k2.5`                | Kimi K2.5                | Hayır        | metin, görsel | 262,144   | 262,144     |

[//]: # "moonshot-kimi-k2-ids:end"

Katalog maliyet tahminleri, Moonshot'ın yayımladığı kullandıkça öde tarifelerini kullanır. Maliyet
kararları vermeden önce [Kimi K3](https://platform.kimi.ai/docs/pricing/chat-k3),
[Kimi K2.7 Code](https://platform.kimi.ai/docs/pricing/chat-k27-code),
[Kimi K2.6](https://platform.kimi.ai/docs/pricing/chat-k26) ve
[Kimi K2.5](https://platform.kimi.ai/docs/pricing/chat-k25) için güncel sağlayıcı
sayfalarını kontrol edin.

Kimi K3, her zaman `reasoning_effort: "max"` düzeyinde akıl yürütür. OpenClaw yalnızca
`/think max` seçeneğini sunar, yalnızca K2'ye özgü `thinking` alanını dahil etmez ve K3'ün
sağlayıcı varsayılanlarına sabitlediği örnekleme geçersiz kılmalarını
(`temperature`, `top_p`, `n`, `presence_penalty` ve
`frequency_penalty`) kaldırır. Kimi K2.7 Code da her zaman yerel düşünmeyi
kullanır ancak hem `thinking` hem de
`reasoning_effort` alanlarının dahil edilmemesini gerektirir; HighSpeed varyantı aynı sözleşmeyi kullanır.
Kimi K2.6, ilk kurulum varsayılanı olarak kalır.
Moonshot'ın [Kimi K3 hızlı başlangıç kılavuzuna](https://platform.kimi.ai/docs/guide/kimi-k3-quickstart) bakın.

## Başlarken

Hem Moonshot hem de Kimi Coding harici eklentilerdir; ilk kurulumdan önce
bunlardan birini yükleyin.

<Tabs>
  <Tab title="Moonshot API">
    **En uygun olduğu durum:** Moonshot Open Platform üzerinden Kimi K3 ve K2 modelleri.

    <Steps>
      <Step title="Eklentiyi yükleyin">
        ```bash
        openclaw plugins install @openclaw/moonshot-provider
        openclaw gateway restart
        ```
      </Step>
      <Step title="Uç nokta bölgenizi seçin">
        | Kimlik doğrulama seçeneği | Uç nokta                       | Bölge         |
        | ------------------------- | ------------------------------ | ------------- |
        | `moonshot-api-key`     | `https://api.moonshot.ai/v1`   | Uluslararası |
        | `moonshot-api-key-cn`  | `https://api.moonshot.cn/v1`   | Çin           |
      </Step>
      <Step title="İlk kurulumu çalıştırın">
        ```bash
        openclaw onboard --auth-choice moonshot-api-key
        ```

        Çin uç noktası için ise:

        ```bash
        openclaw onboard --auth-choice moonshot-api-key-cn
        ```
      </Step>
      <Step title="Kimi K3'ü varsayılan model olarak ayarlayın">
        İlk kurulum, başlangıç varsayılanı olarak Kimi K2.6'yı korur. Kimi K3'ü
        kullanmak istediğinizde açıkça geçiş yapın:

        ```bash
        openclaw models set moonshot/kimi-k3
        ```
      </Step>
      <Step title="Modellerin kullanılabilir olduğunu doğrulayın">
        ```bash
        openclaw models list --provider moonshot
        ```
      </Step>
      <Step title="Canlı bir hızlı doğrulama testi çalıştırın">
        Normal oturumlarınıza dokunmadan model erişimini ve maliyet
        takibini doğrulamak istediğinizde yalıtılmış bir durum dizini kullanın:

        ```bash
        OPENCLAW_CONFIG_PATH=/tmp/openclaw-kimi/openclaw.json \
        OPENCLAW_STATE_DIR=/tmp/openclaw-kimi \
        openclaw agent --local \
          --session-id live-kimi-cost \
          --message 'Reply exactly: KIMI_LIVE_OK' \
          --thinking max \
          --json
        ```

        JSON yanıtı `provider: "moonshot"` ve
        `model: "kimi-k3"` değerlerini bildirmelidir. Moonshot kullanım meta verilerini
        döndürdüğünde asistan dökümündeki girdi, normalleştirilmiş
        token kullanımını ve tahmini maliyeti `usage.cost` altında saklar.
      </Step>
    </Steps>

    ### Yapılandırma örneği

    ```json5
    {
      env: { MOONSHOT_API_KEY: "sk-..." },
      agents: {
        defaults: {
          model: { primary: "moonshot/kimi-k2.6" },
          models: {
            // moonshot-kimi-k2-aliases:start
            "moonshot/kimi-k2.6": { alias: "Kimi K2.6" },
            "moonshot/kimi-k3": { alias: "Kimi K3" },
            "moonshot/kimi-k2.7-code": { alias: "Kimi K2.7 Code" },
            "moonshot/kimi-k2.7-code-highspeed": { alias: "Kimi K2.7 Code HighSpeed" },
            "moonshot/kimi-k2.5": { alias: "Kimi K2.5" },
            // moonshot-kimi-k2-aliases:end
          },
        },
      },
      models: {
        mode: "merge",
        providers: {
          moonshot: {
            baseUrl: "https://api.moonshot.ai/v1",
            apiKey: "${MOONSHOT_API_KEY}",
            api: "openai-completions",
            models: [
              // moonshot-kimi-k2-models:start
              {
                id: "kimi-k2.6",
                name: "Kimi K2.6",
                reasoning: false,
                input: ["text", "image"],
                cost: { input: 0.95, output: 4, cacheRead: 0.16, cacheWrite: 0 },
                contextWindow: 262144,
                maxTokens: 262144,
              },
              {
                id: "kimi-k3",
                name: "Kimi K3",
                reasoning: true,
                thinkingLevelMap: {
                  off: null,
                  minimal: null,
                  low: null,
                  medium: null,
                  high: null,
                  xhigh: "max",
                  max: "max",
                },
                input: ["text", "image"],
                cost: { input: 3, output: 15, cacheRead: 0.3, cacheWrite: 0 },
                contextWindow: 1048576,
                maxTokens: 1048576,
              },
              {
                id: "kimi-k2.7-code",
                name: "Kimi K2.7 Code",
                reasoning: true,
                input: ["text", "image"],
                cost: { input: 0.95, output: 4, cacheRead: 0.19, cacheWrite: 0 },
                contextWindow: 262144,
                maxTokens: 262144,
              },
              {
                id: "kimi-k2.7-code-highspeed",
                name: "Kimi K2.7 Code HighSpeed",
                reasoning: true,
                input: ["text", "image"],
                cost: { input: 1.9, output: 8, cacheRead: 0.38, cacheWrite: 0 },
                contextWindow: 262144,
                maxTokens: 262144,
              },
              {
                id: "kimi-k2.5",
                name: "Kimi K2.5",
                reasoning: false,
                input: ["text", "image"],
                cost: { input: 0.6, output: 3, cacheRead: 0.1, cacheWrite: 0 },
                contextWindow: 262144,
                maxTokens: 262144,
              },
              // moonshot-kimi-k2-models:end
            ],
          },
        },
      },
    }
    ```

  </Tab>

  <Tab title="Kimi Coding">
    **En uygun olduğu durum:** Kimi Coding uç noktası üzerinden kod odaklı görevler.

    <Note>
    Kimi Coding, Moonshot'tan (`moonshot/...`) farklı bir API anahtarı ve sağlayıcı öneki (`kimi/...`) kullanır. Güncel referanslar; 256K bağlam için `kimi/k3`, 1M katmanı için `kimi/k3[1m]`, ayrıca `kimi/kimi-for-coding` ve `kimi/kimi-for-coding-highspeed` değerleridir. Eski `kimi/kimi-code` ve `kimi/k2p5` referansları kabul edilmeye devam eder ve `kimi/kimi-for-coding` değerine normalleştirilir.
    </Note>

    Kodlama hizmeti hem OpenAI uyumlu
    `https://api.kimi.com/coding/v1` hem de Anthropic uyumlu
    `https://api.kimi.com/coding/` istemcilerini kabul eder. Bu eklenti Anthropic Messages kullanır.
    Üyelik anahtarlarını
    [Kimi Code Console](https://www.kimi.com/code/console) üzerinden oluşturun; güncel üyelik
    fiyatları [Kimi'nin fiyatlandırma sayfasında](https://www.kimi.com/membership/pricing) yer alır.

    <Steps>
      <Step title="Eklentiyi yükleyin">
        ```bash
        openclaw plugins install @openclaw/kimi-provider
        openclaw gateway restart
        ```
      </Step>
      <Step title="İlk kurulumu çalıştırın">
        ```bash
        openclaw onboard --auth-choice kimi-code-api-key
        ```
      </Step>
      <Step title="Varsayılan bir model ayarlayın">
        ```json5
        {
          agents: {
            defaults: {
              model: { primary: "kimi/kimi-for-coding" },
            },
          },
        }
        ```
      </Step>
      <Step title="Modelin kullanılabilir olduğunu doğrulayın">
        ```bash
        openclaw models list --provider kimi
        ```
      </Step>
    </Steps>

    Kimi Code K3, varsayılan olarak `max` düzeyinde derin düşünmeyi kullanır. `/think off`,
    `thinking.type: "disabled"` değerini gönderir; `/think max`, K3'ün uyarlanabilir düşünme
    isteğini maksimum çabayla gönderir. Güncelliğini yitirmiş daha düşük düşünme düzeyleri,
    desteklenen `max` düzeyine çözümlenir. 1M modeli, Allegretto veya daha yüksek bir Kimi
    üyeliği gerektirir; Moderato'da `kimi/k3` kullanın.

    Güncel plan kullanılabilirliği için resmî [Kimi Code model tablosuna](https://www.kimi.com/code/docs/en/kimi-code/models.html) bakın.

    ### Yapılandırma örneği

    ```json5
    {
      env: { KIMI_API_KEY: "sk-..." },
      agents: {
        defaults: {
          model: { primary: "kimi/kimi-for-coding" },
          models: {
            "kimi/kimi-for-coding": { alias: "Kimi" },
          },
        },
      },
    }
    ```

  </Tab>
</Tabs>

## Kimi web araması

Moonshot eklentisi ayrıca Moonshot web aramasıyla desteklenen **Kimi**'yi bir `web_search` sağlayıcısı olarak kaydeder.

<Steps>
  <Step title="Etkileşimli web araması kurulumunu çalıştırın">
    ```bash
    openclaw configure --section web
    ```

    `plugins.entries.moonshot.config.webSearch.*` değerini saklamak için web araması bölümünde
    **Kimi**'yi seçin.

  </Step>
  <Step title="Web araması bölgesini ve modelini yapılandırın">
    Etkileşimli kurulum şunları sorar:

    | Ayar                | Seçenekler                                                            |
    | ------------------- | -------------------------------------------------------------------- |
    | API bölgesi         | `https://api.moonshot.ai/v1` (uluslararası) veya `https://api.moonshot.cn/v1` (Çin) |
    | Web araması modeli  | Varsayılan olarak `kimi-k2.6`                                |

  </Step>
</Steps>

Yapılandırma `plugins.entries.moonshot.config.webSearch` altında bulunur:

```json5
{
  plugins: {
    entries: {
      moonshot: {
        config: {
          webSearch: {
            apiKey: "sk-...", // or use KIMI_API_KEY / MOONSHOT_API_KEY
            baseUrl: "https://api.moonshot.ai/v1",
            model: "kimi-k2.6",
          },
        },
      },
    },
  },
  tools: {
    web: {
      search: {
        provider: "kimi",
      },
    },
  },
}
```

## Gelişmiş yapılandırma

<AccordionGroup>
  <Accordion title="Yerel düşünme modu">
    Moonshot API Kimi K3, her zaman maksimum çabayla akıl yürütür. OpenClaw yalnızca
    `/think max` seçeneğini sunar, `reasoning_effort: "max"` değerini gönderir ve güncelliğini yitirmiş daha düşük veya
    `off` ayarlarını yok sayar.

    Kimi Code K3, `/think off|max` sunar. Anthropic uyumlu uç noktası,
    kapalı durum için `thinking.type: "disabled"`, maksimum durum için ise
    `output_config.effort: "max"` ile uyarlamalı düşünme alır. Bu, hem `kimi/k3`
    hem de `kimi/k3[1m]` için geçerlidir.
    Moonshot API K3; `auto`, `none`, `required` ve sabitlenmiş araç seçimlerini destekler,
    bu nedenle OpenClaw istenen `tool_choice` değerini korur. Çok turlu araç kullanımında
    OpenClaw, Moonshot'ın yeniden oynatma sözleşmesinin gerektirdiği
    asistan akıl yürütme içeriğini korur.

    Kimi K2.7 Code her zaman yerel düşünmeyi kullanır. Moonshot, istemcilerin
    bu model için `thinking` alanını çıkarmasını gerektirir; bu nedenle OpenClaw yalnızca `on` sunar ve
    eski `off` ayarlarını yok sayar. K2.7 ayrıca `temperature`, `top_p`, `n`,
    `presence_penalty` ve `frequency_penalty` değerlerini sabitler; OpenClaw bu alanlar için yapılandırılmış
    geçersiz kılmaları çıkarır.

    Diğer Moonshot Kimi modelleri ikili yerel düşünmeyi destekler:

    - `thinking: { type: "enabled" }`
    - `thinking: { type: "disabled" }`

    Bunu model başına `agents.defaults.models.<provider/model>.params` üzerinden yapılandırın:

    ```json5
    {
      agents: {
        defaults: {
          models: {
            "moonshot/kimi-k2.6": {
              params: {
                thinking: { type: "disabled" },
              },
            },
          },
        },
      },
    }
    ```

    OpenClaw, bu modeller için çalışma zamanı `/think` düzeylerini eşler:

    | `/think` düzeyi       | Moonshot davranışı          |
    | -------------------- | -------------------------- |
    | `/think off`         | `thinking.type=disabled`   |
    | Kapalı dışındaki herhangi bir düzey    | `thinking.type=enabled`    |

    <Warning>
    Moonshot K2 düşünmesi etkinleştirildiğinde, `tool_choice` değeri `auto` veya `none` olmalıdır. Sabitlenmiş bir araç seçimi (`type: "tool"` veya `type: "function"`), düşünmeyi bunun yerine zorunlu olarak `disabled` durumuna döndürür; böylece istenen araç yine çalışır. `tool_choice: "required"` ise bunun yerine `auto` olarak normalleştirilir. Kimi K2.7 Code düşünmeyi devre dışı bırakamaz; bu nedenle uyumsuz `tool_choice` değeri `auto` olarak normalleştirilir. Kimi K3, ayrı akıl yürütme çabası sözleşmesini kullanır ve desteklenen araç seçimlerini korur.
    </Warning>

    Kimi K2.6 ayrıca `reasoning_content` değerinin
    çok turlu korunmasını denetleyen isteğe bağlı bir `thinking.keep` alanını kabul eder. Turlar boyunca tam
    akıl yürütmeyi korumak için bunu `"all"` olarak ayarlayın; sunucunun
    varsayılan stratejisini kullanmak için alanı çıkartın (veya `null` olarak bırakın). OpenClaw, `thinking.keep` değerini yalnızca
    `moonshot/kimi-k2.6` için iletir ve diğer modellerden kaldırır. Kimi K2.7 Code,
    varsayılan olarak tam akıl yürütme geçmişini korurken OpenClaw
    `thinking` alanının tamamını çıkarır.

    ```json5
    {
      agents: {
        defaults: {
          models: {
            "moonshot/kimi-k2.6": {
              params: {
                thinking: { type: "enabled", keep: "all" },
              },
            },
          },
        },
      },
    }
    ```

  </Accordion>

  <Accordion title="Araç çağrısı kimliği temizleme">
    Moonshot Kimi, `functions.<name>:<index>` biçimindeki yerel tool_call kimliklerini sunar. OpenClaw, her yerel Kimi kimliğinin ilk örneğini korur ve sonraki yinelenenleri belirlenimci, OpenAI tarzı `call_*` kimlikleriyle yeniden yazar. Eşleşen araç sonuçları aynı kimlikle yeniden eşlenir; böylece Kimi'nin ilk yerel kimliği kaldırılmadan yeniden oynatma benzersiz kalır. Bu davranış, paketlenmiş Moonshot sağlayıcısına bağlıdır ve kullanıcı tarafından yapılandırılabilen bir ayar değildir.
  </Accordion>

  <Accordion title="Akış kullanımı uyumluluğu">
    Yerel Moonshot uç noktaları (`https://api.moonshot.ai/v1` ve
    `https://api.moonshot.cn/v1`) akış kullanımı uyumluluğunu bildirir.
    OpenClaw bunu sağlayıcı kimliğine göre değil, uç nokta ana bilgisayarına göre belirler; dolayısıyla aynı yerel
    Moonshot ana bilgisayarına yönlendirilen özel bir sağlayıcı kimliği, aynı
    akış kullanımı davranışını devralır.

    Katalogdaki K2.6 fiyatlandırmasıyla; girdi, çıktı
    ve önbellekten okuma token'larını içeren akış kullanımı, `/status`, `/usage full`, `/usage cost` ve transkript destekli oturum
    muhasebesi için yerel tahmini USD maliyetine de dönüştürülür.

  </Accordion>

  <Accordion title="Uç nokta ve model başvurusu referansı">
    | Sağlayıcı   | Model başvurusu ön eki | Uç nokta                      | Kimlik doğrulama ortam değişkeni        |
    | ---------- | ---------------- | ------------------------------ | ------------------- |
    | Moonshot   | `moonshot/`      | `https://api.moonshot.ai/v1`  | `MOONSHOT_API_KEY`  |
    | Moonshot CN| `moonshot/`      | `https://api.moonshot.cn/v1`  | `MOONSHOT_API_KEY`  |
    | Kimi Coding| `kimi/`          | Kimi Coding uç noktası           | `KIMI_API_KEY`      |
    | Web araması | Yok              | Moonshot API bölgesiyle aynı    | `KIMI_API_KEY` veya `MOONSHOT_API_KEY` |

    - Kimi web araması `KIMI_API_KEY` veya `MOONSHOT_API_KEY` kullanır ve `kimi-k2.6` modeliyle varsayılan olarak `https://api.moonshot.ai/v1` değerini kullanır.
    - Gerekirse `models.providers` içindeki fiyatlandırma ve bağlam meta verilerini geçersiz kılın.
    - Moonshot bir model için farklı bağlam sınırları yayımlarsa `contextWindow` değerini buna göre ayarlayın.

  </Accordion>
</AccordionGroup>

## İlgili içerikler

<CardGroup cols={2}>
  <Card title="Model seçimi" href="/tr/concepts/model-providers" icon="layers">
    Sağlayıcıları, model başvurularını ve yük devretme davranışını seçme.
  </Card>
  <Card title="Web araması" href="/tr/tools/web" icon="magnifying-glass">
    Kimi dâhil web araması sağlayıcılarını yapılandırma.
  </Card>
  <Card title="Yapılandırma referansı" href="/tr/gateway/configuration-reference" icon="gear">
    Sağlayıcılar, modeller ve Plugin'ler için tam yapılandırma şeması.
  </Card>
  <Card title="Moonshot Open Platform" href="https://platform.moonshot.ai" icon="globe">
    Moonshot API anahtarı yönetimi ve belgeleri.
  </Card>
</CardGroup>
