---
read_when:
    - İş akışlarının içinde yalnızca JSON çıktısı veren bir LLM adımı istiyorsunuz
    - Otomasyon için şemaya göre doğrulanmış LLM çıktısına ihtiyacınız var
summary: İş akışları için yalnızca JSON kullanan LLM görevleri (isteğe bağlı plugin aracı)
title: LLM görevi
x-i18n:
    generated_at: "2026-07-26T23:39:09Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 78ea533f43546fbdd66c7f7138b8dea0b12b02d38925689324b390a12d0c4c5a
    source_path: tools/llm-task.md
    workflow: 16
---

`llm-task`, yalnızca JSON kullanan tek bir LLM çağrısı çalıştıran ve isteğe bağlı olarak bir JSON Şemasına göre doğrulanan yapılandırılmış çıktı döndüren, paketle birlikte sunulan **isteğe bağlı bir plugin aracıdır**. Lobster gibi iş akışı motorlarına, her iş akışı için özel OpenClaw kodu gerektirmeden bir LLM adımı sağlar.

## Etkinleştirme

1. Plugini etkinleştirin:

```json
{
  "plugins": {
    "entries": {
      "llm-task": { "enabled": true }
    }
  }
}
```

2. Araca izin verin:

```json
{
  "tools": {
    "alsoAllow": ["llm-task"]
  }
}
```

`alsoAllow`, diğer temel araçları kısıtlamadan etkin araç profilinin üzerine `llm-task` ekler. Bunun yerine yalnızca kısıtlayıcı bir izin listesi modu istiyorsanız `tools.allow` kullanın.

## Yapılandırma (isteğe bağlı)

```json
{
  "plugins": {
    "entries": {
      "llm-task": {
        "enabled": true,
        "config": {
          "defaultProvider": "openai",
          "defaultModel": "gpt-5.6-sol",
          "defaultAuthProfileId": "main",
          "allowedModels": ["openai/gpt-5.6-sol"],
          "maxTokens": 800,
          "timeoutMs": 30000
        }
      }
    }
  }
}
```

`allowedModels`, `provider/model` dizelerinden oluşan bir izin listesidir; başka herhangi bir model için yapılan istek reddedilir. Diğer tüm anahtarlar, araç çağrısı ilgili parametreyi atladığında kullanılan çağrı bazında geri dönüş değerleridir.

## Araç parametreleri

| Parametre       | Tür    | Notlar                                                                                                                                         |
| --------------- | ------ | --------------------------------------------------------------------------------------------------------------------------------------------- |
| `prompt`        | dize   | Zorunlu. LLM için görev talimatı.                                                                                                       |
| `input`         | herhangi biri | İsteğe bağlı yük; JSON olarak serileştirilir ve isteme eklenir.                                                                              |
| `schema`        | nesne | Ayrıştırılan çıktının doğrulanması gereken isteğe bağlı JSON Şeması.                                                                                 |
| `provider`      | dize | `defaultProvider` / aracının varsayılan sağlayıcısını geçersiz kılar.                                                                                   |
| `model`         | dize | `defaultModel` değerini geçersiz kılar; yalın model kimliklerini, diğer adları veya bir `provider/model` referansını kabul eder (yinelenen sağlayıcı ön eki otomatik olarak kaldırılır). |
| `thinking`      | dize | Akıl yürütme düzeyi (ör. `low`, `medium`); çözümlenen modelin desteklediği düzeylerden biri olmalıdır.                                                          |
| `authProfileId` | dize | `defaultAuthProfileId` değerini geçersiz kılar.                                                                                                             |
| `temperature`   | sayı | En iyi çaba temelinde uygulanır; tüm sağlayıcılar buna uymaz.                                                                                                      |
| `maxTokens`     | sayı | Çıktı tokenleri için en iyi çaba temelinde üst sınır.                                                                                                             |
| `timeoutMs`     | sayı | Çalıştırma zaman aşımı; varsayılan değer `30000`.                                                                                                                 |

## Çıktı

`details.json` (ayrıştırılmış ve şemaya göre doğrulanmış JSON) ile gerçekte neyin çalıştırıldığını belirten `details.provider` ve `details.model` değerlerini döndürür.

## Örnek: Lobster iş akışı adımı

### Önemli sınırlama

Aşağıdaki örnek, **bağımsız Lobster CLI**'ın `openclaw.invoke` için doğru gateway URL'sinin/kimlik doğrulama bağlamının zaten bulunduğu yerde çalıştığını varsayar.

OpenClaw içindeki paketle sunulan **yerleşik** Lobster çalıştırıcısı için bu iç içe CLI düzeni **şu anda güvenilir değildir**:

```lobster
openclaw.invoke --tool llm-task --action json --args-json '{ ... }'
```

Yerleşik Lobster bu akış için desteklenen bir köprüye sahip olana kadar şunlardan birini tercih edin:

- Lobster dışında doğrudan `llm-task` araç çağrıları veya
- iç içe `openclaw.invoke` çağrılarına dayanmayan Lobster adımları.

Bağımsız Lobster CLI örneği:

```lobster
openclaw.invoke --tool llm-task --action json --args-json '{
  "prompt": "Girdi e-postasına göre amacı ve taslağı döndür.",
  "thinking": "low",
  "input": {
    "subject": "Merhaba",
    "body": "Yardımcı olabilir misiniz?"
  },
  "schema": {
    "type": "object",
    "properties": {
      "intent": { "type": "string" },
      "draft": { "type": "string" }
    },
    "required": ["intent", "draft"],
    "additionalProperties": false
  }
}'
```

## Güvenlik notları

- **Yalnızca JSON**: modele kod çitleri veya yorumlar olmadan yalnızca bir JSON değeri döndürmesi talimatı verilir.
- **Araç yok**: temel çalıştırmada araçlar devre dışıdır, bu nedenle model görevin ortasında harici çağrı yapamaz.
- Çıktıyı `schema` ile doğrulamadığınız sürece güvenilmeyen veri olarak değerlendirin.
- Bu çıktıyı kullanan yan etkili tüm adımlardan (gönderme, yayımlama, çalıştırma) önce onay alın.

## İlgili

- [Akıl yürütme düzeyleri](/tr/tools/thinking)
- [Alt aracılar](/tr/tools/subagents)
- [Eğik çizgi komutları](/tr/tools/slash-commands)
