---
read_when:
    - Agent çalıştırmalarını betiklerden veya komut satırından tetiklemek istiyorsunuz
    - Agent yanıtlarını programlı olarak bir sohbet kanalına iletmeniz gerekiyor
summary: CLI üzerinden ajan turlarını çalıştırın ve isteğe bağlı olarak yanıtları kanallara iletin
title: Ajan gönderimi
x-i18n:
    generated_at: "2026-07-26T23:02:59Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: ad3da0feea102725ebb5555e0dd375ed6f3a0396d8ffd0ab916ced303201eabc
    source_path: tools/agent-send.md
    workflow: 16
---

`openclaw agent`, gelen bir sohbet mesajı olmadan komut satırından tek bir ajan turu çalıştırır. Betikli iş akışları, test ve programatik teslimat için kullanın. Bayrakların ve davranışların tam başvurusu:
[Ajan CLI başvurusu](/tr/cli/agent).

## Hızlı başlangıç

<Steps>
  <Step title="Basit bir ajan turu çalıştırın">
    ```bash
    openclaw agent --agent main --message "Bugün hava nasıl?"
    ```

    Mesajı Gateway üzerinden gönderir ve yanıtı yazdırır.

  </Step>

  <Step title="Bir dosyadan çok satırlı istem gönderin">
    ```bash
    openclaw agent --agent ops --message-file ./task.md
    ```

    Geçerli bir UTF-8 dosyasını ajan mesajının gövdesi olarak okur.

  </Step>

  <Step title="Belirli bir ajanı veya oturumu hedefleyin">
    ```bash
    # Belirli bir ajanı hedefle
    openclaw agent --agent ops --message "Günlükleri özetle"

    # Bir telefon numarasını hedefle (oturum anahtarını türetir)
    openclaw agent --to +15555550123 --message "Durum güncellemesi"

    # Mevcut bir oturumu yeniden kullan
    openclaw agent --session-id abc123 --message "Göreve devam et"

    # Tam bir oturum anahtarını hedefle
    openclaw agent --session-key agent:ops:incident-42 --message "Durumu özetle"
    ```

  </Step>

  <Step title="Yanıtı bir kanala teslim edin">
    ```bash
    # WhatsApp'a teslim et (varsayılan kanal)
    openclaw agent --to +15555550123 --message "Rapor hazır" --deliver

    # Slack'e teslim et
    openclaw agent --agent ops --message "Rapor oluştur" \
      --deliver --reply-channel slack --reply-to "#reports"
    ```

  </Step>
</Steps>

## Bayraklar

| Bayrak                      | Açıklama                                                             |
| --------------------------- | -------------------------------------------------------------------- |
| `--message <text>`          | Gönderilecek satır içi mesaj                                         |
| `--message-file <path>`     | Mesajı geçerli bir UTF-8 dosyasından oku (en fazla 4 MiB)            |
| `--to <dest>`               | Oturum anahtarını bir hedeften türet (telefon, sohbet kimliği)       |
| `--session-key <key>`       | Açıkça belirtilmiş bir oturum anahtarı kullan                         |
| `--agent <id>`              | Yapılandırılmış bir ajanı hedefle (onun `main` oturumunu kullanır) |
| `--session-id <id>`         | Mevcut bir oturumu kimliğine göre yeniden kullan                     |
| `--model <id>`              | Bu çalıştırma için model geçersiz kılması (`provider/model` veya model kimliği) |
| `--local`                   | Yerel gömülü çalışma zamanını zorla (Gateway'i atla)                  |
| `--deliver`                 | Yanıtı bir sohbet kanalına gönder                                    |
| `--channel <name>`          | Teslimat kanalı; `--agent` + `--to` ile DM kapsamına da uygulanır |
| `--reply-to <target>`       | Teslimat hedefini geçersiz kıl                                       |
| `--reply-channel <name>`    | Teslimat kanalını geçersiz kıl                                       |
| `--reply-account <id>`      | Teslimat hesabı kimliğini geçersiz kıl                               |
| `--thinking <level>`        | Seçilen model profili için düşünme düzeyini ayarla                   |
| `--verbose <on\|full\|off>` | Oturum için ayrıntı düzeyini kalıcı hâle getir (`full` araç çıktısını da günlüğe kaydeder) |
| `--timeout <seconds>`       | Ajan zaman aşımını geçersiz kıl (varsayılan 600 veya yapılandırma değeri) |
| `--json`                    | Yapılandırılmış JSON çıktısı üret                                    |

## Davranış

- CLI, varsayılan olarak **Gateway üzerinden** çalışır. Geçerli makinedeki
  gömülü çalışma zamanını zorlamak için `--local` ekleyin.
- `--message` veya `--message-file` seçeneklerinden tam olarak birini belirtin. Dosya mesajları,
  isteğe bağlı UTF-8 BOM kaldırıldıktan sonra çok satırlı içeriği korur. 4 MiB'den
  büyük dosyalar gönderimden önce reddedilir.
- Geçici el sıkışma yeniden denemelerinden sonra Gateway zaman aşımı veya kapanmış bağlantı,
  komutun stderr üzerinde bir ipucuyla başarısız olmasına yol açar; CLI, turu gömülü olarak
  hiçbir zaman sessizce yeniden çalıştırmaz. Gateway kabul edilmiş bir turu yine de tamamlayabilir;
  bu nedenle `--local` ile yeniden denemeden veya yeniden çalıştırmadan önce Gateway ve oturum
  durumunu doğrulayın.
- Oturum seçimi: `--to` oturum anahtarını türetir (grup/kanal hedefleri
  yalıtımı korur; doğrudan sohbetler `main` olarak birleştirilir). `--agent`,
  `--channel` ve `--to` birlikte kullanıldığında yönlendirme, kanalın standart
  alıcısını ve `session.dmScope` değerini izler. Yalnızca giden iletiler için kullanılan kararlı kimlikler,
  ajanın ana oturumundan yalıtılmış, sağlayıcıya ait bir oturum kullanır.
- `--session-key`, açıkça belirtilmiş bir anahtar seçer. Ajan önekli anahtarlar
  `agent:<agent-id>:<session-key>` biçimini kullanmalıdır ve ikisi de sağlandığında
  `--agent` bu ajan kimliğiyle eşleşmelidir. Özel işaretçi olmayan yalın anahtarlar,
  sağlandığında `--agent` kapsamında değerlendirilir; örneğin `--agent ops --session-key incident-42`,
  `agent:ops:incident-42` hedefine yönlendirilir. `--agent` olmadan özel işaretçi olmayan
  yalın anahtarlar, yapılandırılmış varsayılan ajan kapsamında değerlendirilir. Değişmez
  `global` ve `unknown`, yalnızca `--agent` sağlanmadığında
  kapsam dışında kalır.
- `--reply-channel` ve `--reply-account` yalnızca teslimatı etkiler.
- Düşünme ve ayrıntı bayrakları oturum deposunda kalıcı hâle gelir.
- Çıktı: varsayılan olarak düz metin veya yapılandırılmış yük + meta veriler için `--json`.
- `--json --deliver` ile JSON; gönderilen, engellenen, kısmi ve başarısız
  teslimatların durumunu içerir. Bkz.
  [JSON teslimat durumu](/tr/cli/agent#json-delivery-status).

## Örnekler

```bash
# JSON çıktılı basit tur
openclaw agent --to +15555550123 --message "Günlükleri izle" --verbose on --json

# Model geçersiz kılmalı tur
openclaw agent --agent ops --model openai/gpt-5.4 --message "Günlükleri özetle"

# Düşünme düzeyli tur
openclaw agent --session-id 1234 --message "Gelen kutusunu özetle" --thinking medium

# Dosyadan çok satırlı istem
openclaw agent --agent ops --message-file ./task.md

# Tam oturum anahtarı
openclaw agent --session-key agent:ops:incident-42 --message "Durumu özetle"

# Bir ajan kapsamında eski anahtar
openclaw agent --agent ops --session-key incident-42 --message "Durumu özetle"

# Oturumdan farklı bir kanala teslim et
openclaw agent --agent ops --message "Uyarı" --deliver --reply-channel telegram --reply-to "@admin"
```

## İlgili

<CardGroup cols={2}>
  <Card title="Ajan CLI başvurusu" href="/tr/cli/agent" icon="terminal">
    `openclaw agent` bayraklarının ve seçeneklerinin tam başvurusu.
  </Card>
  <Card title="Alt ajanlar" href="/tr/tools/subagents" icon="users">
    Arka planda alt ajan başlatma.
  </Card>
  <Card title="Oturumlar" href="/tr/concepts/session" icon="comments">
    Oturum anahtarlarının nasıl çalıştığı ve `--to`, `--agent` ile `--session-id` tarafından nasıl çözümlendiği.
  </Card>
  <Card title="Eğik çizgi komutları" href="/tr/tools/slash-commands" icon="slash">
    Ajan oturumlarında kullanılan yerel komut kataloğu.
  </Card>
</CardGroup>
