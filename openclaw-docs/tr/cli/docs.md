---
read_when:
    - Canlı OpenClaw belgelerinde terminalden arama yapmak istiyorsunuz
    - Dokümanlar CLI'sinin hangi barındırılan arama API'sini çağırdığını bilmeniz gerekir
summary: '`openclaw docs` için CLI başvurusu (canlı doküman dizininde arama yapın)'
title: Belgeler
x-i18n:
    generated_at: "2026-07-26T23:53:41Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: b0b575f0b76d40a53dd4f79c55fd65969a24eae27e27bd1c46d395f61fe89e42
    source_path: cli/docs.md
    workflow: 16
---

# `openclaw docs`

Canlı OpenClaw dokümantasyon dizininde terminalden arama yapın.

## Kullanım

```bash
openclaw docs                       # dokümantasyon giriş noktasını ve örnek aramayı yazdır
openclaw docs <query...>            # canlı dokümantasyon dizininde ara
```

| Argüman     | Açıklama                                                                        |
| ------------ | ---------------------------------------------------------------------------------- |
| `[query...]` | Serbest biçimli arama sorgusu. Birden fazla sözcük içeren sorgular boşluklarla birleştirilip tek bir sorgu olarak gönderilir. |

Sorgu verilmediğinde `openclaw docs`, arama çalıştırmak yerine dokümantasyon giriş noktasının URL'sini ve örnek bir arama komutunu yazdırır.

## Örnekler

```bash
openclaw docs browser existing-session
openclaw docs sandbox allowHostControl
openclaw docs gateway token secretref
```

## Nasıl çalışır?

`openclaw docs`, `https://docs.openclaw.ai/api/search` çağrısını yapar ve JSON sonuçlarını işler. Arama isteği 30 saniyelik sabit bir zaman aşımı kullanır.

## Çıktı

Zengin (TTY) bir terminalde sonuçlar, bir başlığın ardından madde işaretli bir liste olarak işlenir: sayfa başlığı, bağlantılı dokümantasyon URL'si ve sonraki satırda kısa bir alıntı. Boş sonuçlarda "Sonuç yok." yazdırılır.

Zengin olmayan çıktıda (veri yolu üzerinden aktarılan, `--no-color`, betikler) aynı veriler Markdown olarak işlenir:

```markdown
# Dokümantasyon araması: <query>

- [Başlık](https://docs.openclaw.ai/...) - alıntı
- [Başlık](https://docs.openclaw.ai/...) - alıntı
```

## Çıkış kodları

| Kod | Anlamı                                                                  |
| ---- | ------------------------------------------------------------------------ |
| `0`  | Sıfır sonuçlu yanıtlar dâhil olmak üzere arama başarılı oldu.                       |
| `1`  | Barındırılan dokümantasyon arama API'si çağrısı başarısız oldu; stderr hata mesajını yazdırır. |

## İlgili

- [CLI referansı](/tr/cli)
- [Canlı dokümantasyon](https://docs.openclaw.ai)
