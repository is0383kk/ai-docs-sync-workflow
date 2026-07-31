---
read_when:
    - Tokenlar, API anahtarları veya kimlik bilgisi parçacıkları içeren belgeler yazma
    - Gizli bilgi algılama araçları tarafından taranabilecek örnekleri güncelleme
summary: Belgeler ve örnekler için gizli bilgi tarayıcısıyla uyumlu yer tutucu kuralları
title: Gizli Bilgi Yer Tutucu Kuralları
x-i18n:
    generated_at: "2026-07-27T00:00:34Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 0864f0fcc6fb1e4a3147b4b2ce0aac475437a19d694f3d059374782428c7f248
    source_path: reference/secret-placeholder-conventions.md
    workflow: 16
---

# Gizli bilgi yer tutucu kuralları

İnsanlar tarafından okunabilir ancak gerçek gizli bilgilere benzemeyen yer tutucular kullanın.

## Önerilen stil

- `example-openai-key-not-real` veya `example-discord-bot-token` gibi açıklayıcı değerleri tercih edin.
- Kabuk kod parçacıklarında, satır içi token benzeri dizeler yerine `${OPENAI_API_KEY}` kullanmayı tercih edin.
- Örneklerin açıkça sahte ve amaca (sağlayıcı, kanal, kimlik doğrulama türü) uygun kapsamda olmasını sağlayın.

## Dokümanlarda bu kalıplardan kaçının

- Değişmez PEM özel anahtar üstbilgi veya altbilgi metni.
- Canlı kimlik bilgilerine benzeyen ön ekler; ör. `sk-...`, `xoxb-...`, `AKIA...`.
- Çalışma zamanı günlüklerinden kopyalanmış, gerçekçi görünen bearer token'ları.

## Örnek

```bash
# İyi
export OPENAI_API_KEY="example-openai-key-not-real"

# Daha iyi (doküman ortam değişkeni bağlantılarıyla ilgili olduğunda)
export OPENAI_API_KEY="${OPENAI_API_KEY}"
```
