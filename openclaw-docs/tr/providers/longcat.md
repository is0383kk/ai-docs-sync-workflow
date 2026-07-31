---
read_when:
    - OpenClaw ile LongCat-2.0 kullanmak istiyorsunuz
    - LongCat API anahtarına veya model sınırlarına ihtiyacınız var
summary: LongCat-2.0 için LongCat API kurulumu
title: LongCat
x-i18n:
    generated_at: "2026-07-26T22:59:06Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 7c447f9c42e6547a69d2124debcb685c32fe59de29bfc551e18e791d9f280584
    source_path: providers/longcat.md
    workflow: 16
---

[LongCat](https://longcat.ai), kodlama ve ajan tabanlı iş yükleri için geliştirilmiş bir akıl yürütme modeli olan LongCat-2.0 için barındırılan bir API sağlar. OpenClaw, LongCat'in OpenAI uyumlu uç noktası için resmî `longcat` pluginini sağlar.

| Özellik    | Değer                              |
| ---------- | ---------------------------------- |
| Sağlayıcı  | `longcat`                 |
| Kimlik doğrulama | `LONGCAT_API_KEY`          |
| API        | OpenAI uyumlu Chat Completions     |
| Temel URL  | `https://api.longcat.chat/openai`                 |
| Model      | `longcat/LongCat-2.0`                 |
| Bağlam     | 1,048,576 token                    |
| Azami çıktı | 131,072 token                     |
| Girdi      | Metin                              |

## Plugini yükleme

Resmî paketi yükleyin, ardından Gateway'i yeniden başlatın:

```bash
openclaw plugins install @openclaw/longcat-provider
openclaw gateway restart
```

## Başlarken

<Steps>
  <Step title="API anahtarı oluşturma">
    [LongCat API Platformu](https://longcat.chat/platform/) üzerinde oturum açın ve
    [API Keys](https://longcat.chat/platform/api_keys) sayfasında bir anahtar
    oluşturun.
  </Step>
  <Step title="İlk kurulumu çalıştırma">
    ```bash
    openclaw onboard --auth-choice longcat-api-key
    ```
  </Step>
  <Step title="Modeli doğrulama">
    ```bash
    openclaw models list --provider longcat
    ```
  </Step>
</Steps>

İlk kurulum, barındırılan kataloğu ekler ve birincil model henüz yapılandırılmamışsa
`longcat/LongCat-2.0` seçeneğini belirler.

### Etkileşimsiz kurulum

```bash
openclaw onboard --non-interactive \
  --mode local \
  --auth-choice longcat-api-key \
  --longcat-api-key "$LONGCAT_API_KEY"
```

## Akıl yürütme davranışı

LongCat, ikili düşünme denetimi sunar. OpenClaw, etkinleştirilmiş düşünme düzeylerini
`thinking: { type: "enabled" }` ile, `/think off` düzeyini ise
`thinking: { type: "disabled" }` ile eşler. LongCat şu anda
`reasoning_effort` öğesini belgelemediğinden OpenClaw bunu göndermez.

LongCat, akıl yürütmeyi `reasoning_content` içinde döndürür. OpenClaw, çok turlu ajan oturumlarının
sağlayıcının beklediği ileti biçimini koruması için asistan araç çağrısı turlarını yeniden oynatırken
bu alanı korur.

## Fiyatlandırma

Yerleşik katalog, LongCat'in milyon token başına USD cinsinden kullandıkça öde liste fiyatlarını
kullanır: önbelleğe alınmamış girdi için $0.75, önbelleğe alınmış girdi için $0.015 ve çıktı için $2.95.
LongCat geçici indirimler sunabilir; [fiyatlandırma sayfası](https://longcat.chat/platform/docs/Pricing/LongCat-2.0.html)
ve faturalandırma kayıtlarınız belirleyicidir.

## Kendi ortamınızda barındırılan LongCat-2.0

`longcat` sağlayıcısı, LongCat'in barındırılan API'sini hedefler. [Hugging Face](https://huggingface.co/meituan-longcat/LongCat-2.0)
üzerindeki açık ağırlıklar için modeli OpenAI uyumlu bir çalışma zamanı üzerinden sunun ve bunun yerine
OpenClaw'ın mevcut [vLLM](/tr/providers/vllm) veya [SGLang](/tr/providers/sglang) sağlayıcısını kullanın.

Kendi ortamınızda barındırılan sağlayıcı kataloğunda çalışma zamanının tam model tanımlayıcısını koruyun;
yerel bir dağıtımı `longcat/LongCat-2.0` üzerinden yönlendirmeyin.

## Sorun giderme

<AccordionGroup>
  <Accordion title="Anahtar kabukta çalışıyor ancak Gateway'de çalışmıyor">
    Daemon tarafından yönetilen Gateway işlemleri, etkileşimli kabuktaki tüm değişkenleri
    devralmaz. `LONGCAT_API_KEY` öğesini `~/.openclaw/.env` içine yerleştirin, ilk kurulum
    aracılığıyla yapılandırın veya onaylı bir gizli bilgi başvurusu kullanın.
  </Accordion>

  <Accordion title="İstekler 402 veya 429 hatasıyla başarısız oluyor">
    `402`, hesabın token kotasının yetersiz olduğu anlamına gelir. `429`,
    API anahtarının hız sınırına ulaştığı anlamına gelir. [LongCat kullanımını](https://longcat.chat/platform/usage)
    kontrol edin ve hız sınırına takılan istekleri sağlayıcının geri çekilme süresi dolduktan sonra yeniden deneyin.
  </Accordion>

  <Accordion title="Model görünmüyor">
    `openclaw plugins list` komutunu çalıştırıp `longcat` plugininin
    etkin olduğunu doğrulayın, ardından `openclaw models list --provider longcat` komutunu çalıştırın.
  </Accordion>
</AccordionGroup>

## İlgili içerikler

<CardGroup cols={2}>
  <Card title="Model sağlayıcıları" href="/tr/concepts/model-providers" icon="layers">
    Sağlayıcı yapılandırması, model başvuruları ve yük devretme davranışı.
  </Card>
  <Card title="LongCat API belgeleri" href="https://longcat.chat/platform/docs/" icon="arrow-up-right-from-square">
    Barındırılan API uç noktaları, kimlik doğrulama, sınırlar ve örnekler.
  </Card>
  <Card title="LongCat-2.0 model kartı" href="https://huggingface.co/meituan-longcat/LongCat-2.0" icon="arrow-up-right-from-square">
    Mimari, dağıtım yönergeleri ve model ayrıntıları.
  </Card>
  <Card title="Gizli bilgiler" href="/tr/gateway/secrets" icon="key">
    Sağlayıcı kimlik bilgilerini yapılandırmaya düz metin olarak gömmeden saklayın.
  </Card>
</CardGroup>
