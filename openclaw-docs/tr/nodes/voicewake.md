---
read_when:
    - Sesle uyandırma sözcüklerinin davranışını veya varsayılanlarını değiştirme
    - Uyandırma sözcüğü eşitlemesi gerektiren yeni Node platformları ekleme
summary: Genel sesli uyandırma sözcükleri (Gateway tarafından yönetilen) ve bunların Node'lar arasında nasıl eşitlendiği
title: Sesle uyandırma
x-i18n:
    generated_at: "2026-07-26T23:24:38Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: aef2a5bba664ce10fb6ab457bb6d202639dcc6c0a9df61567e7cb402c290bbec
    source_path: nodes/voicewake.md
    workflow: 16
---

Uyandırma sözcükleri, **Gateway'in sahip olduğu tek bir genel listedir** — düğüm başına özel listeler yoktur. Herhangi bir düğüm veya uygulama kullanıcı arayüzü listeyi düzenleyebilir; Gateway değişikliği kalıcı hâle getirir ve bağlı tüm istemcilere yayınlar.

- **macOS**: yerel Sesle Uyandırma etkinleştirme/devre dışı bırakma anahtarı. macOS 26+ gerektirir; çalışma zamanı/PTT ayrıntıları için [Sesle uyandırma (macOS)](/tr/platforms/mac/voicewake) bölümüne bakın.
- **iOS**: Ayarlar'da yerel Sesle Uyandırma etkinleştirme/devre dışı bırakma anahtarı.
- **Android**: Ayarlar → Ses bölümünde yerel Sesle Uyandırma etkinleştirme/devre dışı bırakma anahtarı ve uyandırma sözcüğü düzenleyicisi. Android'in cihaz üzerindeki konuşma tanıma özelliğini gerektirir.

## Depolama

Uyandırma sözcükleri ve yönlendirme kuralları Gateway durum veritabanında, varsayılan olarak `~/.openclaw/state/openclaw.sqlite` konumunda (geçersiz kılmak için `OPENCLAW_STATE_DIR`), `voicewake_triggers`, `voicewake_routing_config`, `voicewake_routing_routes` tablolarında bulunur. Eski `settings/voicewake.json` ve `settings/voicewake-routing.json` yalnızca `openclaw doctor --fix` geçiş girdileridir — çalışma zamanı bunları hiçbir zaman okumaz.

## Protokol

### Tetikleyici listesi

| Yöntem          | Parametreler                   | Sonuç                   |
| --------------- | ------------------------ | ------------------------ |
| `voicewake.get` | yok                     | `{ triggers: string[] }` |
| `voicewake.set` | `{ triggers: string[] }` | `{ triggers: string[] }` |

`voicewake.set` girdiyi normalleştirir: boşlukları kırpar, boş girdileri kaldırır, en fazla 32 tetikleyiciyi tutar ve vekil çiftleri bölmeden her birini 64 UTF-16 kod birimine kısaltır. Boş bir sonuç, yerleşik varsayılanlara (`openclaw`, `claude`, `computer`) geri döner.

### Yönlendirme (tetikleyiciden hedefe)

| Yöntem                  | Parametreler                               | Sonuç                               |
| ----------------------- | ------------------------------------ | ------------------------------------ |
| `voicewake.routing.get` | yok                                 | `{ config: VoiceWakeRoutingConfig }` |
| `voicewake.routing.set` | `{ config: VoiceWakeRoutingConfig }` | `{ config: VoiceWakeRoutingConfig }` |

```json
{
  "version": 1,
  "defaultTarget": { "mode": "current" },
  "routes": [{ "trigger": "robot wake", "target": { "sessionKey": "agent:main:main" } }],
  "updatedAtMs": 1730000000000
}
```

Her yönlendirme `target` aşağıdakilerden tam olarak birini destekler:

- `{ "mode": "current" }`
- `{ "agentId": "main" }`
- `{ "sessionKey": "agent:main:main" }`

Sınırlar: en fazla 32 yönlendirme, tetikleyici metni en fazla 64 karakter. Yönlendirme tetikleyicileri; küçük harfe dönüştürülerek, her sözcüğün başındaki/sonundaki noktalama işaretleri kaldırılarak ve boşluklar birleştirilerek eşleştirme ve yinelenen öğe algılama amacıyla normalleştirilir (`"Hey, Bot!!"` ve `"hey bot"` eşleşir ve yinelenen öğeler olarak sayılır) — bu, yukarıdaki genel tetikleyici listesi için kullanılan basit kırpma işleminden daha katı bir normalleştirmedir.

### Olaylar

| Olay                       | Yük                              |
| --------------------------- | ------------------------------------ |
| `voicewake.changed`         | `{ triggers: string[] }`             |
| `voicewake.routing.changed` | `{ config: VoiceWakeRoutingConfig }` |

Her ikisi de okuma kapsamına sahip tüm WebSocket istemcilerine (macOS uygulaması, WebChat ve benzerleri) ve bağlı tüm düğümlere yayınlanır. Bir düğüm ayrıca bağlandıktan hemen sonra başlangıç anlık görüntüsü gönderimi olarak her ikisini de alır.

## İstemci davranışı

- **macOS**: `voicewake.set`/`voicewake.get` çağrılarını yapar ve diğer istemcilerle eşzamanlı kalmak için `voicewake.changed` olayını dinler.
- **iOS**: `voicewake.set`/`voicewake.get` çağrılarını yapar ve yerel uyandırma sözcüğü algılamasının duyarlı kalması için `voicewake.changed` olayını dinler.
- **Android**: `voicewake.set`/`voicewake.get` çağrılarını yapar, `voicewake.changed` olayını dinler ve etkinken `voiceWake` özelliğini duyurur. Tanıma cihaz üzerinde ve yalnızca ön planda kalır; Talk, manuel dikte, sesli not kaydı veya mesaj seslendirme sesi kullandığında duraklatılır.

## İlgili

- [Talk modu](/tr/nodes/talk)
- [Ses ve sesli notlar](/tr/nodes/audio)
- [Medya anlama](/tr/nodes/media-understanding)
