---
read_when:
    - Sesli arama eklentisini kullanıyorsunuz ve her CLI giriş noktasını istiyorsunuz
    - Kurulum, smoke, call, continue, speak, dtmf, end, status, tail, latency, expose ve start için bayrak tablolarına ve varsayılan değerlere ihtiyacınız var
summary: '`openclaw voicecall` için CLI başvurusu (voice-call Plugin komut yüzeyi)'
title: Sesli arama
x-i18n:
    generated_at: "2026-07-26T23:36:49Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: aec445886cccb79c9212dd9f1f448ff9634274deb380632be786478c9bb29670
    source_path: cli/voicecall.md
    workflow: 16
---

# `openclaw voicecall`

`voicecall`, Plugin tarafından sağlanan bir komuttur. Yalnızca voice-call
Plugin'i yüklü ve etkin olduğunda görünür.

Gateway çalışırken operasyonel komutlar (`call`, `start`,
`continue`, `speak`, `dtmf`, `end`, `status`) ilgili Gateway'in
voice-call çalışma zamanına yönlendirilir. Erişilebilir bir Gateway yoksa bağımsız
CLI çalışma zamanına geri dönerler.

## Alt komutlar

```bash
openclaw voicecall setup    [--json]
openclaw voicecall smoke    [-t <phone>] [--message <text>] [--mode <m>] [--yes] [--json]
openclaw voicecall call     -m <text> [-t <phone>] [--mode <m>]
openclaw voicecall start    --to <phone> [--message <text>] [--mode <m>]
openclaw voicecall continue --call-id <id> --message <text>
openclaw voicecall speak    --call-id <id> --message <text>
openclaw voicecall dtmf     --call-id <id> --digits <digits>
openclaw voicecall end      --call-id <id>
openclaw voicecall status   [--call-id <id>] [--json]
openclaw voicecall tail     [--file <path>] [--since <n>] [--poll <ms>]
openclaw voicecall latency  [--file <path>] [--last <n>]
openclaw voicecall expose   [--mode <m>] [--path <p>] [--port <port>] [--serve-path <p>]
```

| Alt komut | Açıklama                                                        |
| ---------- | --------------------------------------------------------------- |
| `setup`    | Sağlayıcı ve Webhook hazırlık denetimlerini gösterir.            |
| `smoke`    | Hazırlık denetimlerini çalıştırır; yalnızca `--yes` ile canlı bir test araması yapar. |
| `call`     | Giden bir sesli arama başlatır.                                 |
| `start`    | `call` için, `--to` zorunlu ve `--message` isteğe bağlı olacak şekilde bir diğer addır. |
| `continue` | Bir mesajı seslendirir ve sonraki yanıtı bekler.                 |
| `speak`    | Yanıt beklemeden bir mesajı seslendirir.                         |
| `dtmf`     | Etkin bir aramaya DTMF rakamları gönderir.                       |
| `end`      | Etkin bir aramayı sonlandırır.                                  |
| `status`   | Etkin aramaları (veya `--call-id` ile belirtilen bir aramayı) inceler. |
| `tail`     | `calls.jsonl` dosyasını izler (sağlayıcı testleri sırasında kullanışlıdır). |
| `latency`  | `calls.jsonl` içindeki tur gecikmesi ölçümlerini özetler.       |
| `expose`   | Webhook uç noktası için Tailscale serve/funnel özelliğini açar veya kapatır. |

## Kurulum ve smoke testi

### `setup`

Varsayılan olarak insan tarafından okunabilir hazırlık denetimleri yazdırır. Betikler için `--json` geçirin.

```bash
openclaw voicecall setup
openclaw voicecall setup --json
```

### `smoke`

Aynı hazırlık denetimlerini çalıştırır. Yalnızca hem
`--to` hem de `--yes` mevcut olduğunda gerçek bir telefon araması yapar.

| Bayrak             | Varsayılan                        | Açıklama                                  |
| ------------------ | --------------------------------- | ----------------------------------------- |
| `-t, --to <phone>` | (yok)                             | Canlı smoke testi için aranacak telefon numarası. |
| `--message <text>` | `OpenClaw voice call smoke test.` | Smoke araması sırasında seslendirilecek mesaj. |
| `--mode <mode>`    | `notify`                          | Arama modu: `notify` veya `conversation`. |
| `--yes`            | `false`                           | Canlı giden aramayı gerçekten gerçekleştirir. |
| `--json`           | `false`                           | Makine tarafından okunabilir JSON yazdırır. |

```bash
openclaw voicecall smoke
openclaw voicecall smoke --to "+15555550123"        # deneme çalıştırması
openclaw voicecall smoke --to "+15555550123" --yes  # canlı bildirim araması
```

<Note>
Harici sağlayıcılarda (`plivo`, `telnyx`, `twilio`), `setup` ve `smoke`; `publicUrl`, bir tünel veya Tailscale üzerinden dışa açma yöntemiyle sağlanan genel bir Webhook URL'si gerektirir. Operatörler erişemeyeceği için geri döngü veya özel serve geri dönüşü reddedilir.
</Note>

## Arama yaşam döngüsü

### `call`

Giden bir sesli arama başlatır.

| Bayrak                 | Zorunlu | Varsayılan        | Açıklama                                                                   |
| ---------------------- | ------- | ----------------- | -------------------------------------------------------------------------- |
| `-m, --message <text>` | evet    | (yok)             | Arama bağlandığında seslendirilecek mesaj.                                 |
| `-t, --to <phone>`     | hayır   | config `toNumber` | Aranacak E.164 telefon numarası.                                           |
| `--mode <mode>`        | hayır   | `conversation`    | Arama modu: `notify` (mesajdan sonra kapat) veya `conversation` (açık tut). |

```bash
openclaw voicecall call --to "+15555550123" --message "Merhaba"
openclaw voicecall call -m "Bilginize" --mode notify
```

### `start`

Farklı bir varsayılan bayrak biçimine sahip `call` diğer adıdır.

| Bayrak             | Zorunlu | Varsayılan     | Açıklama                                  |
| ------------------ | ------- | -------------- | ----------------------------------------- |
| `--to <phone>`     | evet    | (yok)          | Aranacak telefon numarası.                |
| `--message <text>` | hayır   | (yok)          | Arama bağlandığında seslendirilecek mesaj. |
| `--mode <mode>`    | hayır   | `conversation` | Arama modu: `notify` veya `conversation`. |

### `continue`

Bir mesajı seslendirir ve yanıt bekler.

| Bayrak             | Zorunlu | Açıklama                  |
| ------------------ | ------- | ------------------------- |
| `--call-id <id>`   | evet    | Arama kimliği.            |
| `--message <text>` | evet    | Seslendirilecek mesaj.    |

### `speak`

Yanıt beklemeden bir mesajı seslendirir.

| Bayrak             | Zorunlu | Açıklama                  |
| ------------------ | ------- | ------------------------- |
| `--call-id <id>`   | evet    | Arama kimliği.            |
| `--message <text>` | evet    | Seslendirilecek mesaj.    |

### `dtmf`

Etkin bir aramaya DTMF rakamları gönderir.

| Bayrak              | Zorunlu | Açıklama                                          |
| ------------------- | ------- | ------------------------------------------------- |
| `--call-id <id>`    | evet    | Arama kimliği.                                    |
| `--digits <digits>` | evet    | DTMF rakamları (beklemeler için örneğin `ww123456#`). |

### `end`

Etkin bir aramayı sonlandırır.

| Bayrak           | Zorunlu | Açıklama       |
| ---------------- | ------- | -------------- |
| `--call-id <id>` | evet    | Arama kimliği. |

### `status`

Etkin aramaları inceler.

| Bayrak           | Varsayılan | Açıklama                              |
| ---------------- | ---------- | ------------------------------------- |
| `--call-id <id>` | (yok)      | Çıktıyı tek bir aramayla sınırlar.    |
| `--json`         | `false` | Makine tarafından okunabilir JSON yazdırır. |

```bash
openclaw voicecall status
openclaw voicecall status --json
openclaw voicecall status --call-id <id>
```

## Günlükler ve ölçümler

### `tail`

Voice-call JSONL günlüğünü izler. Başlangıçta son `--since` satırı yazdırır,
ardından yazıldıkça yeni satırları akış olarak aktarır.

| Bayrak          | Varsayılan                 | Açıklama                             |
| --------------- | -------------------------- | ------------------------------------ |
| `--file <path>` | Plugin deposundan çözümlenir | `calls.jsonl` yolu.          |
| `--since <n>`   | `25`                       | İzlemeye başlamadan önce yazdırılacak satır sayısı. |
| `--poll <ms>`   | `250` (minimum 50)         | Milisaniye cinsinden yoklama aralığı. |

### `latency`

`calls.jsonl` içindeki tur gecikmesi ve dinleme-bekleme ölçümlerini özetler. Çıktı;
`recordsScanned`, `turnLatency` ve `listenWait` özetlerini içeren JSON biçimindedir.

| Bayrak          | Varsayılan                 | Açıklama                                  |
| --------------- | -------------------------- | ----------------------------------------- |
| `--file <path>` | Plugin deposundan çözümlenir | `calls.jsonl` yolu.               |
| `--last <n>`    | `200` (minimum 1)          | Analiz edilecek son kayıtların sayısı. |

## Webhook'ları dışa açma

### `expose`

Ses Webhook'u için Tailscale serve/funnel yapılandırmasını etkinleştirir, devre dışı bırakır
veya değiştirir.

| Bayrak                | Varsayılan                                | Açıklama                                        |
| --------------------- | ----------------------------------------- | ----------------------------------------------- |
| `--mode <mode>`       | `funnel`                                  | `off`, `serve` (tailnet) veya `funnel` (genel). |
| `--path <path>`       | config `tailscale.path` veya `--serve-path` | Dışa açılacak Tailscale yolu.                   |
| `--port <port>`       | config `serve.port` veya `3334`             | Yerel Webhook portu.                            |
| `--serve-path <path>` | config `serve.path` veya `/voice/webhook`   | Yerel Webhook yolu.                             |

```bash
openclaw voicecall expose --mode serve
openclaw voicecall expose --mode funnel
openclaw voicecall expose --mode off
```

<Warning>
Webhook uç noktasını yalnızca güvendiğiniz ağlara açın. Mümkün olduğunda Funnel yerine Tailscale Serve'ü tercih edin.
</Warning>

## İlgili

- [CLI referansı](/tr/cli)
- [Sesli arama Plugin'i](/tr/plugins/voice-call)
