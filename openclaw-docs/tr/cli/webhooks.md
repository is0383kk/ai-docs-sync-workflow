---
read_when:
    - Gmail Pub/Sub olaylarını OpenClaw'a bağlamak istiyorsunuz
    - Tam bayrak listesine ve varsayılan değerlere ihtiyacınız var
summary: '`openclaw webhooks` için CLI referansı (Gmail Pub/Sub kurulumu ve çalıştırıcısı)'
title: Webhook'lar
x-i18n:
    generated_at: "2026-07-26T23:53:54Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 83fff0ac2ce247402f45523eda0b5cdd551bd65212636118698e45cb8740236c
    source_path: cli/webhooks.md
    workflow: 16
---

# `openclaw webhooks`

Webhook yardımcıları ve entegrasyonları. Şu anda bu yüzeyin kapsamı, paketle birlikte sunulan `gog` izleyicisini temel alan Gmail Pub/Sub akışlarıyla sınırlıdır.

## Alt komutlar

```bash
openclaw webhooks gmail setup --account <email> [...]
openclaw webhooks gmail run   [--account <email>] [...]
```

| Alt komut    | Açıklama                                                                           |
| ------------- | ------------------------------------------------------------------------------------- |
| `gmail setup` | Tek seferlik sihirbaz: Gmail izleme, Pub/Sub konusu/aboneliği ve OpenClaw kanca teslimi. |
| `gmail run`   | `gog watch serve` ile izleme otomatik yenileme döngüsünü ön planda çalıştırır.               |

<Note>
`hooks.enabled=true` ve `hooks.gmail.account` ayarlandıktan sonra (bunlar `gmail setup` tarafından ayarlanır) Gateway, başlatılırken `gog gmail watch serve` öğesini de otomatik olarak başlatır. `gmail run`, aynı mantığı ön planda çalıştırır; hata ayıklama için veya Gateway izleyicisi devre dışı bırakıldığında kullanışlıdır. Otomatik başlatma ayrıntıları ve `OPENCLAW_SKIP_GMAIL_WATCHER` ile devre dışı bırakma seçeneği için [Gmail Pub/Sub entegrasyonu](/tr/automation/cron-jobs#gmail-pubsub-integration) bölümüne bakın.
</Note>

## `webhooks gmail setup`

```bash
openclaw webhooks gmail setup --account you@example.com
openclaw webhooks gmail setup --account you@example.com --project my-gcp-project --json
openclaw webhooks gmail setup --account you@example.com --hook-url https://gateway.example.com/hooks/gmail
```

Eksikse `gcloud` ve `gog` öğelerini yükler, `gcloud` kimlik doğrulamasını gerçekleştirir, Pub/Sub konusunu ve aboneliğini oluşturur, Gmail izlemeyi başlatır ve `hooks.enabled=true` ile `hooks.gmail` yapılandırmasını yazar. `Next: openclaw webhooks gmail run` değerini yazdırır.

### Zorunlu

| Bayrak                | Açıklama             |
| ------------------- | ----------------------- |
| `--account <email>` | İzlenecek Gmail hesabı. |

### Pub/Sub seçenekleri

| Bayrak                    | Varsayılan                | Açıklama                                                                                                                             |
| ----------------------- | ---------------------- | --------------------------------------------------------------------------------------------------------------------------------------- |
| `--project <id>`        | (yok)                 | GCP proje kimliği (OAuth istemcisinin sahibi). Önce konunun kendi proje kimliğine, ardından `gog` kimlik bilgilerinden çözümlenen projeye geri döner. |
| `--topic <name>`        | `gog-gmail-watch`      | Pub/Sub konu adı.                                                                                                                     |
| `--subscription <name>` | `gog-gmail-watch-push` | Pub/Sub abonelik adı.                                                                                                              |
| `--label <label>`       | `INBOX`                | İzlenecek Gmail etiketi.                                                                                                                   |
| `--push-endpoint <url>` | (yok)                 | Açık Pub/Sub anında iletim uç noktası. Tailscale'i geçersiz kılar.                                                                                    |

### OpenClaw teslim seçenekleri

| Bayrak                   | Varsayılan                                      | Açıklama                                |
| ---------------------- | -------------------------------------------- | ------------------------------------------ |
| `--hook-url <url>`     | `hooks.path` ve Gateway bağlantı noktasından oluşturulur | OpenClaw webhook URL'si.                      |
| `--hook-token <token>` | `hooks.token` veya oluşturulan bir belirteç          | OpenClaw webhook belirteci.                    |
| `--push-token <token>` | Oluşturulan belirteç                              | `gog watch serve` öğesine iletilen anında iletim belirteci. |

### `gog watch serve` seçenekleri

| Bayrak                  | Varsayılan         | Açıklama                                                                                                                                  |
| --------------------- | --------------- | -------------------------------------------------------------------------------------------------------------------------------------------- |
| `--bind <host>`       | `127.0.0.1`     | `gog watch serve` bağlama ana makinesi.                                                                                                                 |
| `--port <port>`       | `8788`          | `gog watch serve` bağlantı noktası.                                                                                                                      |
| `--path <path>`       | `/gmail-pubsub` | `gog watch serve` yolu. Tailscale, proxy işlemi öncesinde yolu kaldırdığı için açık bir hedef olmadan etkinleştirildiğinde `/` olarak zorlanır. |
| `--include-body`      | `true`          | E-posta gövdesi parçacıklarını dahil eder. Bunu kapatacak bir CLI bayrağı yoktur; bunun yerine yapılandırmada `hooks.gmail.includeBody: false` ayarını belirleyin.                  |
| `--max-bytes <n>`     | `20000`         | Gövde parçacığı başına en fazla bayt.                                                                                                                  |
| `--renew-minutes <n>` | `720` (12h)     | Gmail izlemeyi her N dakikada bir yeniler.                                                                                                           |

### Tailscale üzerinden erişime açma

| Bayrak                      | Varsayılan  | Açıklama                                                      |
| ------------------------- | -------- | ---------------------------------------------------------------- |
| `--tailscale <mode>`      | `funnel` | Anında iletim uç noktasını tailscale üzerinden kullanıma açar: `funnel`, `serve` veya `off`. |
| `--tailscale-path <path>` | (yok)   | tailscale serve/funnel yolu.                                 |
| `--tailscale-target <t>`  | (yok)   | Tailscale serve/funnel hedefi (bağlantı noktası, `host:port` veya URL).       |

### Çıktı

| Bayrak     | Açıklama                                       |
| -------- | ------------------------------------------------- |
| `--json` | Metin yerine makine tarafından okunabilir bir özet yazdırır. |

## `webhooks gmail run`

```bash
openclaw webhooks gmail run --account you@example.com
```

`gog watch serve` ile izleme otomatik yenileme döngüsünü ön planda çalıştırır ve `gog watch serve` beklenmedik şekilde sonlanırsa 2s gecikmenin ardından yeniden başlatır.

`run`, aşağıdaki istisnalar dışında `setup` ile aynı Pub/Sub, OpenClaw teslim, `gog watch serve` ve Tailscale bayraklarını kabul eder:

- `--account`, `run` üzerinde **isteğe bağlıdır**; `hooks.gmail.account` değerine geri döner.
- `run`; `--project`, `--push-endpoint` veya `--json` değerlerini kabul **etmez**.
- Her bayrak önce eşleşen `hooks.gmail.*` yapılandırma değerine (`setup` tarafından yazılır), ardından `setup` tarafından kullanılan aynı yerleşik varsayılana geri döner; tek istisna şudur: ne bayrak ne de `hooks.gmail.tailscale.mode` ayarlanmışsa `--tailscale`, `run` üzerinde `funnel` yerine `off` değerini varsayılan olarak kullanır.

| Kategori          | Bayraklar                                                                            |
| ----------------- | -------------------------------------------------------------------------------- |
| Pub/Sub           | `--account`, `--topic`, `--subscription`, `--label`                              |
| OpenClaw teslimi | `--hook-url`, `--hook-token`, `--push-token`                                     |
| `gog watch serve` | `--bind`, `--port`, `--path`, `--include-body`, `--max-bytes`, `--renew-minutes` |
| Tailscale         | `--tailscale`, `--tailscale-path`, `--tailscale-target`                          |

<Note>
`run` için `--topic` değeri, yalnızca kısa konu adı değil, tam Pub/Sub konu yoludur (`projects/.../topics/...`).
</Note>

## İlgili

- [CLI referansı](/tr/cli)
- [Webhook otomasyonu](/tr/automation/cron-jobs)
- [Gmail Pub/Sub entegrasyonu](/tr/automation/cron-jobs#gmail-pubsub-integration)
