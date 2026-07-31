---
read_when:
    - Scriptlerden tek bir ajan turu çalıştırmak istiyorsunuz (isteğe bağlı olarak yanıtı iletmek için)
summary: '`openclaw agent` için CLI referansı (Gateway aracılığıyla bir agent turu gönderme)'
title: Ajan
x-i18n:
    generated_at: "2026-07-26T23:52:54Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 1a4c139a3b235d6a56ba63063737b80f93448c2dbb7a92c6d0756fb19a9f95e4
    source_path: cli/agent.md
    workflow: 16
---

# `openclaw agent`

Gateway üzerinden tek bir ajan turu çalıştırın. Açık `--local` bayrağı, tek gömülü yürütme yoludur.

En az bir oturum seçici iletin: `--to`, `--session-key`, `--session-id` veya `--agent`.

İlgili: [Ajan gönderme aracı](/tr/tools/agent-send)

## Seçenekler

- `-m, --message <text>`: ileti gövdesi
- `--message-file <path>`: ileti gövdesini bir UTF-8 dosyasından oku
- `-t, --to <dest>`: oturum anahtarını türetmek için kullanılan alıcı
- `--session-key <key>`: yönlendirme için kullanılacak açık oturum anahtarı
- `--session-id <id>`: açık oturum kimliği
- `--agent <id>`: ajan kimliği; yönlendirme bağlamalarını geçersiz kılar
- `--model <id>`: bu çalıştırma için model geçersiz kılma (`provider/model` veya model kimliği)
- `--thinking <level>`: ajan düşünme düzeyi (`off`, `minimal`, `low`, `medium`, `high` ve `xhigh`, `adaptive` veya `max` gibi sağlayıcı tarafından desteklenen özel düzeyler)
- `--verbose <on|off>`: ayrıntı düzeyini oturum için kalıcı hâle getir
- `--channel <channel>`: teslimat kanalı; ana oturum kanalını kullanmak için atlayın
- `--reply-to <target>`: teslimat hedefini geçersiz kılma
- `--reply-channel <channel>`: teslimat kanalını geçersiz kılma
- `--reply-account <id>`: teslimat hesabını geçersiz kılma
- `--local`: gömülü ajanı doğrudan çalıştır (Plugin kayıt defteri önceden yüklendikten sonra)
- `--deliver`: yanıtı seçilen kanala/hedefe geri gönder
- `--timeout <seconds>`: bu komutun ajan turu son tarihini geçersiz kıl (varsayılan 600 veya `agents.defaults.timeoutSeconds`); `0` genel son tarihi devre dışı bırakır. 600 saniyelik geri dönüş değeri, varsayılan süresi 48 saat olan sıradan Gateway turlarına değil, bu CLI komutuna aittir.
- `--json`: JSON çıktısı ver

## Örnekler

```bash
openclaw agent --to +15555550123 --message "durum güncellemesi" --deliver
openclaw agent --agent ops --message "Günlükleri özetle"
openclaw agent --agent ops --message-file ./task.md
openclaw agent --agent ops --model openai/gpt-5.4 --message "Günlükleri özetle"
openclaw agent --session-key agent:ops:incident-42 --message "Durumu özetle"
openclaw agent --agent ops --session-key incident-42 --message "Durumu özetle"
openclaw agent --session-id 1234 --message "Gelen kutusunu özetle" --thinking medium
openclaw agent --to +15555550123 --message "Günlükleri izle" --verbose on --json
openclaw agent --agent ops --message "Rapor oluştur" --deliver --reply-channel slack --reply-to "#reports"
openclaw agent --agent ops --message "Yerel olarak çalıştır" --local
```

## Notlar

- `--message` veya `--message-file` seçeneklerinden tam olarak birini iletin. `--message-file`, baştaki UTF-8 BOM'u kaldırır ve çok satırlı içeriği korur; geçerli UTF-8 olmayan dosyaları reddeder. 4 MiB'den büyük dosyalar gönderimden önce reddedilir.
- Eğik çizgi komutları (örneğin `/compact`) `--message` üzerinden çalıştırılamaz. CLI bunları reddeder ve bunun yerine birinci sınıf komuta yönlendirir (Compaction için `openclaw sessions compact <key>`).
- `--local` çalıştırmaları tek seferliktir: çalıştırma için açılan paketlenmiş MCP geri döngü kaynakları ve sıcak Claude stdio oturumları yanıttan sonra sonlandırılır; böylece betik tabanlı çağrılar yerel alt süreçleri çalışır durumda bırakmaz. Gateway destekli çalıştırmalar ise Gateway'e ait MCP geri döngü kaynaklarını çalışan Gateway süreci altında tutar.
- `--local` ile bağımsız gömülü yürütme, yeniden başlatma kurtarması beklemedeyken mevcut bir ana oturumu yeniden kullanmayı reddeder. Turu sağlıklı bir Gateway üzerinden çalıştırın veya orada `/new` ya da `/reset` ile sıfırlayın; bağımsız bir gömülü süreç, bu kurtarma sahibini Gateway tarayıcısıyla güvenli biçimde koordine edemez.
- `--agent`, `--channel` ve `--to` birlikte kullanıldığında oturum yönlendirmesi, kanalın kurallı alıcısını ve `session.dmScope` değerini izler. Kararlı ve yalnızca giden bir alıcı kimliğine sahip kanallar, ajanın ana oturumundan yalıtılmış, sağlayıcıya ait bir oturum kullanır. `--reply-channel` ve `--reply-account` yalnızca teslimatı etkiler.
- `--session-key`, açık bir oturum anahtarı seçer. Ajan ön ekli anahtarlar `agent:<agent-id>:<session-key>` kullanmalıdır ve her ikisi de verildiğinde `--agent`, anahtarın ajan kimliğiyle eşleşmelidir. Öneksiz ve sentinel olmayan anahtarlar, sağlanmışsa `--agent` kapsamına; aksi hâlde yapılandırılmış varsayılan ajanın kapsamına girer. Örneğin `--agent ops --session-key incident-42`, `agent:ops:incident-42` hedefine yönlendirilir. `global` ve `unknown` değişmez anahtarları yalnızca `--agent` sağlanmadığında kapsamsız kalır.
- `--json`, stdout'u JSON yanıtına ayırır; betiklerin stdout'u doğrudan ayrıştırabilmesi için Gateway, Plugin ve `--local` tanılamaları stderr'e gider.
- Geçici el sıkışma yeniden denemeleri tükendikten sonra Gateway zaman aşımı veya kapalı bağlantı komutun başarısız olmasına neden olur; CLI turu hiçbir zaman sessizce gömülü olarak yeniden çalıştırmaz. Aktarım kaybı belirsizdir — Gateway turu kabul etmiş ve hâlâ tamamlıyor olabilir — bu nedenle turun iki kez yürütülmesini önlemek amacıyla stderr ipucu, yeniden denemeden veya `--local` ile yeniden çalıştırmadan önce `openclaw gateway status` değerini ve oturum dökümünü kontrol etmenizi söyler.
- `SIGTERM`/`SIGINT`, bekleyen Gateway destekli bir isteği keser; Gateway çalıştırmayı zaten kabul ettiyse CLI, çıkmadan önce bu çalıştırma kimliği için ayrıca `chat.abort` gönderir. `--local` çalıştırmaları aynı sinyali alır ancak `chat.abort` göndermez. İlk iletilen `SIGINT` veya `SIGTERM` nedeniyle sonlanan bir başlatıcı alt süreci, sırasıyla 130 veya 143 durumuyla çıkar. Dahili çalıştırma tekilleştirme anahtarının bu oturum için zaten etkin bir çalıştırması varsa yanıt `status: "in_flight"` bildirir ve JSON olmayan CLI, boş yanıt yerine stderr'e bir tanılama yazdırır. Harici cron/systemd sarmalayıcılarında, kapanma boşaltılamazsa denetleyicinin süreci temizleyebilmesi için `timeout -k 60 600 openclaw agent ...` gibi zorla sonlandırma amaçlı bir güvenlik mekanizması bulundurun.
- Bu komut `models.json` yeniden oluşturmasını tetiklediğinde, SecretRef tarafından yönetilen sağlayıcı kimlik bilgileri çözümlenmiş gizli düz metin olarak değil, gizli olmayan işaretçiler (örneğin ortam değişkeni adları, `secretref-env:ENV_VAR_NAME` veya `secretref-managed`) olarak kalıcılaştırılır. İşaretçi yazımları çözümlenmiş çalışma zamanı gizli değerlerinden değil, etkin kaynak yapılandırma anlık görüntüsünden gelir.

## JSON teslimat durumu

`--json --deliver` ile CLI JSON yanıtı, betiklerin teslim edilmiş, engellenmiş, kısmi ve başarısız gönderimleri ayırt edebilmesi için üst düzey `deliveryStatus` alanını içerir:

```json
{
  "payloads": [{ "text": "Rapor hazır", "mediaUrl": null }],
  "meta": { "durationMs": 1200 },
  "deliveryStatus": {
    "requested": true,
    "attempted": true,
    "status": "sent",
    "succeeded": true,
    "resultCount": 1
  }
}
```

Gateway destekli CLI yanıtları, ham Gateway sonuç biçimini `result.deliveryStatus` konumunda da korur.

`deliveryStatus.status` şunlardan biridir:

| Durum           | Anlamı                                                                                                                                    |
| ---------------- | ------------------------------------------------------------------------------------------------------------------------------------------ |
| `sent`           | Teslimat tamamlandı.                                                                                                                        |
| `suppressed`     | Teslimat kasıtlı olarak gönderilmedi (örneğin bir ileti gönderme kancası teslimatı iptal etti veya görünür bir sonuç yoktu). Son durumdur, yeniden denenmez. |
| `partial_failed` | Sonraki bir veri yükü başarısız olmadan önce en az bir veri yükü gönderildi.                                                                                   |
| `failed`         | Kalıcı hiçbir gönderim tamamlanmadı veya teslimat ön kontrolü başarısız oldu.                                                                                   |

Ortak alanlar:

- `requested`: nesne mevcut olduğunda her zaman `true`.
- `attempted`: kalıcı gönderim yolu çalıştığında `true`; ön kontrol hatalarında veya görünür veri yükü olmadığında `false`.
- `succeeded`: `true`, `false` veya `"partial"`; `"partial"`, `status: "partial_failed"` ile eşleşir.
- `reason`: kalıcı teslimattan veya ön kontrol doğrulamasından gelen küçük harfli snake-case neden. Bilinen değerler arasında `cancelled_by_message_sending_hook`, `no_visible_payload`, `no_visible_result`, `channel_resolved_to_internal`, `unknown_channel`, `invalid_delivery_target` ve `no_delivery_target` bulunur; başarısız kalıcı gönderimler başarısız aşamayı da bildirebilir. Değer kümesi genişleyebileceğinden bilinmeyen değerleri opak kabul edin.
- `resultCount`: mevcut olduğunda kanal gönderme sonuçlarının sayısı.
- `sentBeforeError`: kısmi bir hata, hata oluşmadan önce en az bir veri yükü gönderdiyse `true`.
- `error`: başarısız veya kısmen başarısız gönderimler için `true`.
- `errorMessage`: yalnızca temel teslimat hata iletisi yakalandığında bulunur. Ön kontrol hataları `error`/`reason` taşır ancak `errorMessage` taşımaz.
- `payloadOutcomes`: mevcut olduğunda `index`, `status`, `reason`, `resultCount`, `error`, `stage`, `sentBeforeError` veya kanca meta verileri içeren isteğe bağlı veri yükü başına sonuçlar.

## İlgili

- [CLI başvurusu](/tr/cli)
- [Ajan çalışma zamanı](/tr/concepts/agent)
