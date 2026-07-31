---
read_when:
    - Çalışan Gateway'in durumunu hızlıca kontrol etmek istiyorsunuz
summary: RPC aracılığıyla Gateway sağlık anlık görüntüsü için `openclaw health` CLI başvurusu
title: Sağlık
x-i18n:
    generated_at: "2026-07-26T23:35:47Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 51cc0e3dd61af3e6fa460dd646bfa1c3e5bd1a52da860eac26c12101151d081d
    source_path: cli/health.md
    workflow: 16
---

# `openclaw health`

Çalışan Gateway'den WebSocket RPC üzerinden bir sistem durumu anlık görüntüsü alın (CLI'dan doğrudan kanal soketleri kullanılmaz).

## Seçenekler

| Bayrak           | Varsayılan | Açıklama                                                                                                  |
| ---------------- | ---------- | --------------------------------------------------------------------------------------------------------- |
| `--json`         | `false` | Metin yerine makine tarafından okunabilir JSON yazdırır.                                                  |
| `--timeout <ms>` | `10000` | Milisaniye cinsinden bağlantı zaman aşımı.                                                                |
| `--verbose`      | `false` | Canlı yoklamayı zorlar ve çıktıyı yapılandırılmış tüm hesapları ve ajanları kapsayacak şekilde genişletir. |
| `--debug`        | `false` | `--verbose` için takma ad.                                                                         |

Örnekler:

```bash
openclaw health
openclaw health --json
openclaw health --timeout 2500
openclaw health --verbose
openclaw health --debug
```

## Davranış

- `--verbose` olmadan Gateway, önbelleğe alınmış bir anlık görüntü döndürebilir (60 saniyeye kadar günceldir ve canlı kanal çalışma zamanı durumuyla aynıdır) ve sonraki çağıran için bunu arka planda yenileyebilir.
- `--verbose` canlı yoklamayı (kanal başına hesap yoklamalarını) zorlar, Gateway bağlantı ayrıntılarını yazdırır ve insan tarafından okunabilir çıktıyı yalnızca varsayılan ajan yerine yapılandırılmış tüm hesapları ve ajanları kapsayacak şekilde genişletir.
- `--json` her zaman tam anlık görüntüyü döndürür: kanallar, hesap başına yoklamalar, Plugin yükleme durumu, bağlam motoru karantina durumu, model fiyatlandırma önbelleği durumu, olay döngüsü sistem durumu, teslimat kuyruğu işlenemeyen iletileri ve ajan başına oturum depoları.
- Giden teslimatlar veya gelen kanal olayları işlenemeyen ileti olarak ayrıldığında, metin çıktısı bunların sayılarını ve en eski hatanın yaşını bildirir. Gelen ileti sayıları kanal hesabına göre gruplandırılır; tek tek olayları [`openclaw channels dead-letters`](/tr/cli/channels#inbound-dead-letters) ile inceleyin veya kurtarın.

## İlgili

- [CLI referansı](/tr/cli)
- [`openclaw status`](/tr/cli/status) — tam sistem durumu anlık görüntüsü olmadan yerel tanılama ve kanal yoklamaları
- [Gateway sistem durumu](/tr/gateway/health)
