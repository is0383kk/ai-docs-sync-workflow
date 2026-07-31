---
read_when:
    - OpenClaw'da gizlilik odaklı çıkarım istiyorsunuz
    - Venice AI kurulum yönergelerini istiyorsunuz
summary: OpenClaw'da gizlilik odaklı Venice AI modellerini kullanın
title: Venice AI
x-i18n:
    generated_at: "2026-07-27T00:15:12Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 13c32b783394eb3092ff94a532b69e34c00624127b0e76e4e2812751d39073a1
    source_path: providers/venice.md
    workflow: 16
---

[Venice AI](https://venice.ai), günlük tutmadan çalışan açık modellerin yanı sıra Claude, GPT, Gemini ve Grok'a anonimleştirilmiş proxy erişimiyle gizlilik odaklı çıkarım sağlar.
Tüm uç noktalar OpenAI uyumludur (`/v1`).

## Gizlilik modları

| Mod                 | Davranış                                                                 | Modeller                                                       |
| ------------------- | ------------------------------------------------------------------------ | -------------------------------------------------------------- |
| **Özel**            | İstemler/yanıtlar hiçbir zaman saklanmaz veya günlüğe kaydedilmez. Geçicidir. | Llama, Qwen, DeepSeek, Kimi, MiniMax, Venice Uncensored vb. |
| **Anonimleştirilmiş** | İletilmeden önce meta verileri kaldırılarak Venice üzerinden proxy'lenir. | Claude, GPT, Gemini, Grok                                  |

<Warning>
Anonimleştirilmiş modeller tamamen özel değildir. Venice, iletmeden önce meta verileri kaldırır ancak temel sağlayıcı (OpenAI, Anthropic, Google, xAI) isteği işlemeye devam eder. Tam gizlilik gerektiğinde Özel modelleri kullanın.
</Warning>

## Başlarken

<Steps>
  <Step title="Plugin'i yükleyin">
    ```bash
    openclaw plugins install @openclaw/venice-provider
    ```
  </Step>
  <Step title="API anahtarınızı alın">
    1. [venice.ai](https://venice.ai) adresinde kaydolun
    2. **Settings > API Keys > Create new key** bölümüne gidin
    3. API anahtarınızı kopyalayın (biçim: `vapi_xxxxxxxxxxxx`)
  </Step>
  <Step title="OpenClaw'ı yapılandırın">
    <Tabs>
      <Tab title="Etkileşimli (önerilen)">
        ```bash
        openclaw onboard --auth-choice venice-api-key
        ```

        API anahtarını ister (veya mevcut bir `VENICE_API_KEY` değerini yeniden kullanır), kullanılabilir Venice modellerini listeler ve varsayılan modelinizi ayarlar.
      </Tab>
      <Tab title="Ortam değişkeni">
        ```bash
        export VENICE_API_KEY="vapi_xxxxxxxxxxxx"
        ```
      </Tab>
      <Tab title="Etkileşimsiz">
        ```bash
        openclaw onboard --non-interactive \
          --auth-choice venice-api-key \
          --venice-api-key "vapi_xxxxxxxxxxxx"
        ```
      </Tab>
    </Tabs>

  </Step>
  <Step title="Kurulumu doğrulayın">
    ```bash
    openclaw agent --model venice/kimi-k2-5 --message "Merhaba, çalışıyor musun?"
    ```
  </Step>
</Steps>

## Model seçimi

- **Varsayılan**: `venice/kimi-k2-5` (özel, akıl yürütme, görsel).
- **En güçlü anonimleştirilmiş seçenek**: `venice/claude-opus-4-6`.

```bash
openclaw models set venice/kimi-k2-5
openclaw models list --all --provider venice
```

Ayrıca `openclaw configure` komutunu çalıştırıp **Model/auth provider > Venice AI** seçeneğini belirleyebilirsiniz.

<Tip>
| Kullanım alanı             | Model                                        | Nedeni                                         |
| -------------------------- | -------------------------------------------- | ---------------------------------------------- |
| Genel sohbet (varsayılan)  | `kimi-k2-5`                           | Görsel desteğiyle güçlü özel akıl yürütme      |
| Genel olarak en iyi kalite | `claude-opus-4-6`                           | En güçlü anonimleştirilmiş Venice seçeneği     |
| Gizlilik + kodlama         | `qwen3-coder-480b-a35b-instruct-turbo`                           | Geniş bağlama sahip özel kodlama modeli        |
| Hızlı + ucuz               | `llama-3.2-3b`                           | Kompakt özel model                             |
| Karmaşık özel görevler     | `deepseek-v3.2`                           | Güçlü akıl yürütme; araç çağırma devre dışı    |
| Sansürsüz                  | `venice-uncensored-1-2`                           | Güncel sansürsüz Venice modeli                 |
</Tip>

## Yerleşik katalog (30 model)

<AccordionGroup>
  <Accordion title="Özel modeller (20) — tamamen özel, günlük tutma yok">
    | Model kimliği                          | Ad                                    | Bağlam  | Notlar                              |
    | -------------------------------------- | ------------------------------------- | ------- | ----------------------------------- |
    | `kimi-k2-5`                     | Kimi K2.5                             | 256k    | Varsayılan, akıl yürütme, görsel    |
    | `llama-3.3-70b`                     | Llama 3.3 70B                         | 128k    | Genel                               |
    | `llama-3.2-3b`                     | Llama 3.2 3B                          | 128k    | Genel                               |
    | `hermes-3-llama-3.1-405b`                     | Hermes 3 Llama 3.1 405B               | 128k    | Genel, araçlar devre dışı           |
    | `qwen3-235b-a22b-thinking-2507`                     | Qwen3 235B Thinking                   | 128k    | Akıl yürütme                        |
    | `qwen3-235b-a22b-instruct-2507`                     | Qwen3 235B Instruct                   | 128k    | Genel                               |
    | `qwen3-coder-480b-a35b-instruct-turbo`                     | Qwen3 Coder 480B Turbo                | 256k    | Kodlama                             |
    | `qwen3-5-35b-a3b`                     | Qwen3.5 35B A3B                       | 256k    | Akıl yürütme, görsel                |
    | `qwen3-next-80b`                     | Qwen3 Next 80B                        | 256k    | Genel                               |
    | `qwen3-vl-235b-a22b`                     | Qwen3 VL 235B (Vision)                | 256k    | Görsel                              |
    | `deepseek-v3.2`                     | DeepSeek V3.2                         | 160k    | Akıl yürütme, araçlar devre dışı    |
    | `google-gemma-3-27b-it`                     | Google Gemma 3 27B Instruct           | 198k    | Görsel                              |
    | `openai-gpt-oss-120b`                     | OpenAI GPT OSS 120B                   | 128k    | Genel                               |
    | `nvidia-nemotron-3-nano-30b-a3b`                     | NVIDIA Nemotron 3 Nano 30B            | 128k    | Genel                               |
    | `olafangensan-glm-4.7-flash-heretic`                     | GLM 4.7 Flash Heretic                 | 128k    | Akıl yürütme                        |
    | `zai-org-glm-4.6`                     | GLM 4.6                               | 198k    | Genel                               |
    | `zai-org-glm-4.7`                     | GLM 4.7                               | 198k    | Akıl yürütme                        |
    | `zai-org-glm-4.7-flash`                     | GLM 4.7 Flash                         | 128k    | Akıl yürütme                        |
    | `zai-org-glm-5`                     | GLM 5                                 | 198k    | Akıl yürütme                        |
    | `minimax-m25`                     | MiniMax M2.5                          | 198k    | Akıl yürütme                        |
  </Accordion>

  <Accordion title="Anonimleştirilmiş modeller (10) — Venice proxy'si üzerinden">
    | Model kimliği                   | Ad                             | Bağlam  | Notlar                           |
    | ------------------------------- | ------------------------------ | ------- | -------------------------------- |
    | `claude-opus-4-6`              | Claude Opus 4.6 (Venice üzerinden)   | 1M      | Akıl yürütme, görsel             |
    | `claude-sonnet-4-6`              | Claude Sonnet 4.6 (Venice üzerinden) | 1M      | Akıl yürütme, görsel             |
    | `openai-gpt-54`              | GPT-5.4 (Venice üzerinden)           | 1M      | Akıl yürütme, görsel             |
    | `openai-gpt-53-codex`              | GPT-5.3 Codex (Venice üzerinden)     | 400k    | Akıl yürütme, görsel, kodlama    |
    | `openai-gpt-52`              | GPT-5.2 (Venice üzerinden)           | 256k    | Akıl yürütme                     |
    | `openai-gpt-52-codex`              | GPT-5.2 Codex (Venice üzerinden)     | 256k    | Akıl yürütme, görsel, kodlama    |
    | `openai-gpt-4o-2024-11-20`              | GPT-4o (Venice üzerinden)            | 128k    | Görsel                           |
    | `openai-gpt-4o-mini-2024-07-18`              | GPT-4o Mini (Venice üzerinden)       | 128k    | Görsel                           |
    | `gemini-3-1-pro-preview`              | Gemini 3.1 Pro (Venice üzerinden)    | 1M      | Akıl yürütme, görsel             |
    | `gemini-3-flash-preview`              | Gemini 3 Flash (Venice üzerinden)    | 256k    | Akıl yürütme, görsel             |
  </Accordion>
</AccordionGroup>

Grok destekli Venice modelleri (`grok-4-3` ve benzerleri), aynı yukarı akış
araç çağrısı biçimini paylaştıkları için yerel xAI sağlayıcısıyla aynı araç şeması
uyumluluk yamasını alır.

## Model keşfi

Yukarıdaki paketle sunulan katalog, manifest destekli bir başlangıç listesidir. OpenClaw çalışma zamanında
bu listeyi Venice `/models` API'sinden yeniler ve API'ye erişilemezse
başlangıç listesine geri döner. `/models` uç noktası herkese açıktır (listeleme için
kimlik doğrulama gerekmez), ancak çıkarım için geçerli bir API anahtarı gerekir.

Venice, kullanımdan kaldırılmış model kimliklerini sağlayıcının yönettiği takma adlar olarak kabul etmeye devam edebilir.
OpenClaw kataloğu yalnızca `/models` tarafından döndürülen standart model kimliklerini yayınlar.

## DeepSeek V4 yeniden oynatma davranışı

Venice, `deepseek-v4-pro` veya
`deepseek-v4-flash` gibi DeepSeek V4 modellerini kullanıma sunarsa OpenClaw, Venice bu alanı atladığında
asistan mesajlarındaki gerekli `reasoning_content` yeniden oynatma alanını doldurur ve istek yükünden
`thinking`/`reasoning`/`reasoning_effort` alanlarını çıkarır
(Venice, bu modellerde DeepSeek'in yerel `thinking` denetimini reddeder). Bu yeniden oynatma düzeltmesi,
yerel DeepSeek sağlayıcısının kendi düşünme denetimlerinden ayrıdır.

## Akış ve araç desteği

| Özellik          | Destek                                                    |
| ---------------- | --------------------------------------------------------- |
| Akış             | Tüm modeller                                               |
| İşlev çağırma    | Çoğu model; yukarıda belirtildiği durumlarda model bazında devre dışı |
| Görsel/Görüntüler | Yukarıda "Görsel" olarak işaretlenen modeller             |
| JSON modu        | `response_format` üzerinden                              |

## Fiyatlandırma

Venice kredi tabanlı bir sistem kullanır. Anonimleştirilmiş modellerin maliyeti,
doğrudan API fiyatlandırmasına eklenen küçük bir Venice ücretiyle yaklaşık olarak aynıdır. Güncel ücretler için
[venice.ai/pricing](https://venice.ai/pricing) sayfasına bakın.

## Kullanım örnekleri

```bash
# Varsayılan özel model
openclaw agent --model venice/kimi-k2-5 --message "Hızlı durum denetimi"

# Venice üzerinden Claude Opus (anonimleştirilmiş)
openclaw agent --model venice/claude-opus-4-6 --message "Bu görevi özetle"

# Sansürsüz model
openclaw agent --model venice/venice-uncensored-1-2 --message "Seçenekler tasarla"

# Görüntü içeren görsel modeli
openclaw agent --model venice/qwen3-vl-235b-a22b --message "Ekli görüntüyü incele"

# Kodlama modeli
openclaw agent --model venice/qwen3-coder-480b-a35b-instruct-turbo --message "Bu işlevi yeniden düzenle"
```

## Sorun giderme

<AccordionGroup>
  <Accordion title="API anahtarı tanınmıyor">
    ```bash
    echo $VENICE_API_KEY
    openclaw models list | grep venice
    ```

    Anahtarın `vapi_` ile başladığını doğrulayın.

  </Accordion>

  <Accordion title="Model kullanılamıyor">
    Şu anda kullanılabilir modelleri görmek için `openclaw models list --all --provider venice` komutunu çalıştırın;
    Venice model ekledikçe veya kullanımdan kaldırdıkça katalog değişir.
  </Accordion>

  <Accordion title="Bağlantı sorunları">
    Venice API, `https://api.venice.ai/api/v1` adresindedir. Ağınızın bu ana makineye HTTPS erişimine izin verdiğini doğrulayın.
  </Accordion>
</AccordionGroup>

<Note>
Daha fazla yardım: [Sorun giderme](/tr/help/troubleshooting) ve [SSS](/tr/help/faq).
</Note>

## Gelişmiş yapılandırma

<AccordionGroup>
  <Accordion title="Yapılandırma dosyası örneği">
    ```json5
    {
      env: { VENICE_API_KEY: "vapi_..." },
      agents: { defaults: { model: { primary: "venice/kimi-k2-5" } } },
      models: {
        mode: "merge",
        providers: {
          venice: {
            baseUrl: "https://api.venice.ai/api/v1",
            apiKey: "${VENICE_API_KEY}",
            api: "openai-completions",
            models: [
              {
                id: "kimi-k2-5",
                name: "Kimi K2.5",
                reasoning: true,
                input: ["text", "image"],
                cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 },
                contextWindow: 256000,
                maxTokens: 65536,
              },
            ],
          },
        },
      },
    }
    ```
  </Accordion>
</AccordionGroup>

## İlgili

<CardGroup cols={2}>
  <Card title="Model seçimi" href="/tr/concepts/model-providers" icon="layers">
    Sağlayıcıları, model referanslarını ve yük devretme davranışını seçme.
  </Card>
  <Card title="Venice AI" href="https://venice.ai" icon="globe">
    Venice AI ana sayfası ve hesap kaydı.
  </Card>
  <Card title="API belgeleri" href="https://docs.venice.ai" icon="book">
    Venice API referansı ve geliştirici belgeleri.
  </Card>
  <Card title="Fiyatlandırma" href="https://venice.ai/pricing" icon="credit-card">
    Güncel Venice kredi ücretleri ve planları.
  </Card>
</CardGroup>
