---
read_when:
    - Bir Cron işi oluşturmadan bir sistem olayını kuyruğa eklemek istiyorsunuz
    - Heartbeat'leri etkinleştirmeniz veya devre dışı bırakmanız gerekir
    - Sistem iletişim durumu girdilerini incelemek istiyorsunuz
summary: '`openclaw system` için CLI başvurusu (sistem olayları, heartbeat, çevrimiçi olma durumu)'
title: Sistem
x-i18n:
    generated_at: "2026-07-26T23:53:33Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: aaca206d8b463fd33f9e3cb21382bbf36469e9daa2706d8a9e2c7fab14b76e7a
    source_path: cli/system.md
    workflow: 16
---

# `openclaw system`

Gateway için sistem düzeyinde yardımcılar: sistem olaylarını kuyruğa alın, Heartbeat'leri denetleyin ve iletişim durumunu görüntüleyin.

Tüm `system` alt komutları Gateway RPC kullanır ve paylaşılan istemci bayraklarını kabul eder:

| Bayrak            | Varsayılan                           | Açıklama                                                                                                                                                                                               |
| ----------------- | ------------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `--url <url>`     | yapılandırıldığında `gateway.remote.url` | Gateway WebSocket URL'si.                                                                                                                                                                              |
| `--token <token>` | yok                                  | Gateway belirteci (gerekiyorsa).                                                                                                                                                                       |
| `--timeout <ms>`  | `30000`                              | Milisaniye cinsinden RPC zaman aşımı.                                                                                                                                                                  |
| `--expect-final`  | kapalı                               | Nihai yanıtı (agent) bekleyin.                                                                                                                                                                         |
| `--json`          | kapalı                               | JSON çıktısı verin. `heartbeat last/enable/disable` ve `system presence`, bu bayraktan bağımsız olarak her zaman ham RPC JSON yükünü yazdırır; `system event`, JSON ile düz bir `ok` satırı arasında geçiş yapmak için bunu kullanır. |

## Yaygın komutlar

```bash
openclaw system event --text "Acil takip işlemlerini denetle" --mode now
openclaw system event --text "Acil takip işlemlerini denetle" --url ws://127.0.0.1:18789 --token "$OPENCLAW_GATEWAY_TOKEN"
openclaw system heartbeat enable
openclaw system heartbeat last
openclaw system presence
```

## `system event`

Varsayılan olarak **ana** oturumda bir sistem olayını kuyruğa alın. Sonraki Heartbeat, bunu isteme bir `System:` satırı olarak ekler. Heartbeat'i hemen tetiklemek için `--mode now` kullanın; `next-heartbeat` (varsayılan), bir sonraki zamanlanmış çevrimi bekler.

Belirli bir oturumu hedeflemek için `--session-key` iletin; örneğin, zaman uyumsuz bir görevin tamamlanmasını görevi başlatan kanala geri aktarmak için.

<Note>
**`--session-key` ile zamanlama istisnası:** `--session-key` sağlandığında, `--mode next-heartbeat` bir sonraki zamanlanmış çevrimi beklemek yerine anında hedeflenmiş bir uyandırmaya dönüşür. Hedeflenmiş uyandırmalar, Heartbeat amacı olarak `immediate` kullanır; böylece aksi takdirde `event` amaçlı bir uyandırmayı erteleyecek (ve fiilen düşürecek) olan çalıştırıcının henüz zamanı gelmedi kapısını atlar. Gecikmeli teslimat istiyorsanız olayın ana oturuma ulaşması ve bir sonraki düzenli Heartbeat ile iletilmesi için `--session-key` seçeneğini kullanmayın.
</Note>

Bayraklar:

- `--text <text>`: gerekli sistem olayı metni.
- `--mode <mode>`: `now` veya `next-heartbeat` (varsayılan).
- `--session-key <sessionKey>`: isteğe bağlı; agent'ın ana oturumu yerine belirli bir agent oturumunu hedefler. Çözümlenen agent'a ait olmayan anahtarlar, agent'ın ana oturumuna geri döner.

## `system heartbeat last|enable|disable`

- `last`: son Heartbeat olayını gösterin.
- `enable`: Heartbeat'leri yeniden açın (devre dışı bırakıldılarsa bunu kullanın).
- `disable`: Heartbeat'leri duraklatın.

## `system presence`

Gateway'in bildiği mevcut sistem iletişim durumu girdilerini (Node'lar, örnekler ve benzer durum satırları) listeleyin.

## Notlar

- Geçerli yapılandırmanızla erişilebilen, çalışan bir Gateway gerektirir (yerel veya uzak).
- Sistem olayları geçicidir ve yeniden başlatmalar arasında kalıcı olarak saklanmaz.

## İlgili

- [CLI başvurusu](/tr/cli)
