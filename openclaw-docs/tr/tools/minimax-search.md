---
read_when:
    - web_search için MiniMax kullanmak istiyorsunuz
    - Bir MiniMax Token Plan anahtarına veya OAuth jetonuna ihtiyacınız var
    - MiniMax CN/global arama ana makinesi yönlendirmesi istiyorsunuz
summary: Token Plan arama API'si üzerinden MiniMax Search
title: MiniMax araması
x-i18n:
    generated_at: "2026-07-26T23:05:32Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: cb851614bbe43f011e07fe3e80d5390f1ba515f3e00ba749c91999617ad2d1e2
    source_path: tools/minimax-search.md
    workflow: 16
---

OpenClaw, MiniMax Token Plan arama API'si üzerinden bir `web_search` sağlayıcısı olarak MiniMax'i destekler. API; başlıklar, URL'ler, kısa açıklamalar ve ilgili sorgular içeren yapılandırılmış arama sonuçları döndürür.

## Token Plan kimlik bilgisi edinme

<Steps>
  <Step title="Anahtar oluşturma">
    [MiniMax Platform](https://platform.minimax.io/user-center/basic-information/interface-key) üzerinden bir MiniMax Token Plan anahtarı oluşturun veya kopyalayın.
    OAuth kurulumları bunun yerine `MINIMAX_OAUTH_TOKEN` öğesini yeniden kullanabilir.
  </Step>
  <Step title="Anahtarı saklama">
    Gateway ortamında `MINIMAX_CODE_PLAN_KEY` değişkenini ayarlayın veya şununla yapılandırın:

    ```bash
    openclaw configure --section web
    ```

  </Step>
</Steps>

OpenClaw ayrıca `MINIMAX_CODING_API_KEY`, `MINIMAX_OAUTH_TOKEN` ve
`MINIMAX_API_KEY` değerlerini ortam değişkeni takma adları olarak kabul eder; bunlar
`MINIMAX_CODE_PLAN_KEY` sonrasında bu sırayla kontrol edilir. `MINIMAX_API_KEY`, arama özelliği etkinleştirilmiş
bir Token Plan kimlik bilgisini göstermelidir; sıradan MiniMax model API anahtarları
Token Plan arama uç noktası tarafından kabul edilmeyebilir.

## Yapılandırma

```json5
{
  plugins: {
    entries: {
      minimax: {
        config: {
          webSearch: {
            apiKey: "sk-cp-...", // MiniMax Token Plan ortam değişkeni ayarlanmışsa isteğe bağlıdır
            region: "global", // veya "cn"
          },
        },
      },
    },
  },
  tools: {
    web: {
      search: {
        provider: "minimax",
      },
    },
  },
}
```

**Ortam alternatifi:** Gateway ortamında `MINIMAX_CODE_PLAN_KEY`, `MINIMAX_CODING_API_KEY`,
`MINIMAX_OAUTH_TOKEN` veya `MINIMAX_API_KEY` değerini ayarlayın.
Gateway kurulumu için bunu `~/.openclaw/.env` içine yerleştirin.

## Bölge seçimi

MiniMax Search şu uç noktaları kullanır:

- Küresel: `https://api.minimax.io/v1/coding_plan/search`
- Çin: `https://api.minimaxi.com/v1/coding_plan/search`

`plugins.entries.minimax.config.webSearch.region` ayarlanmamışsa OpenClaw,
bölgeyi şu sırayla belirler:

1. Plugin'e ait `webSearch.region`
2. `MINIMAX_API_HOST`
3. `models.providers.minimax.baseUrl`
4. `models.providers.minimax-portal.baseUrl`

Bu, Çin ilk kurulumunun veya `MINIMAX_API_HOST=https://api.minimaxi.com/...` değerinin
MiniMax Search'ü de otomatik olarak Çin ana makinesinde tuttuğu anlamına gelir.

MiniMax kimlik doğrulamasını OAuth `minimax-portal` yolu üzerinden gerçekleştirmiş olsanız bile
web araması yine `minimax` sağlayıcı kimliğiyle kaydedilir; OAuth sağlayıcısının temel URL'si,
Çin/küresel ana makine seçimi için bölge ipucu olarak kullanılır ve `MINIMAX_OAUTH_TOKEN`
MiniMax Search taşıyıcı kimlik bilgisini karşılayabilir.

## Desteklenen parametreler

| Parametre | Tür     | Kısıtlamalar       | Açıklama                                                                       |
| --------- | ------- | ------------------ | ------------------------------------------------------------------------------ |
| `query`   | dize    | zorunlu            | Arama sorgusu dizesi.                                                          |
| `count`   | tamsayı | 1-10, varsayılan 5 | Döndürülecek sonuç sayısı. OpenClaw, döndürülen listeyi bu boyuta kadar kırpar. |

Sağlayıcıya özgü filtreler şu anda desteklenmemektedir.

## İlgili

- [Web Aramasına genel bakış](/tr/tools/web) -- tüm sağlayıcılar ve otomatik algılama
- [MiniMax](/tr/providers/minimax) -- model, görüntü, konuşma ve kimlik doğrulama kurulumu
