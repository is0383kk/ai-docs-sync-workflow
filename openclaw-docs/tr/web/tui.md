---
read_when:
    - TUI için yeni başlayanlara uygun bir adım adım kılavuz istiyorsunuz
    - TUI özelliklerinin, komutlarının ve kısayollarının tam listesine ihtiyacınız var
summary: 'Terminal Kullanıcı Arayüzü (TUI): Gateway''e bağlanın veya gömülü modda yerel olarak çalıştırın'
title: TUI
x-i18n:
    generated_at: "2026-07-26T23:41:20Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: dc4dc5e2a408b5097b3615283b5a4590e8b55bccb15c26d8e38ab2c84b902f4a
    source_path: web/tui.md
    workflow: 16
---

## Hızlı başlangıç

### Gateway modu

1. Gateway'i başlatın.

```bash
openclaw gateway
```

2. TUI'yi açın.

```bash
openclaw tui
```

3. Bir ileti yazıp Enter tuşuna basın.

Uzak Gateway:

```bash
openclaw tui --url ws://<host>:<port> --token <gateway-token>
```

Gateway'iniz parola kimlik doğrulaması kullanıyorsa `--password` kullanın.

### Yerel mod

TUI'yi Gateway olmadan çalıştırın:

```bash
openclaw chat
# veya
openclaw tui --local
```

- `openclaw chat` ve `openclaw terminal`, `openclaw tui --local` için diğer adlardır.
- `--local`; `--url`, `--token` veya `--password` ile birlikte kullanılamaz.
- Yerel mod, gömülü aracı çalışma zamanını doğrudan kullanır. Çoğu yerel araç çalışır ancak yalnızca Gateway'e özgü özellikler kullanılamaz.
- Alt komut içermeyen `openclaw`, hedefi otomatik olarak seçer: yapılandırılmamış bir kurulum çıkarım ilk katılımını çalıştırır; geçersiz yapılandırma klasik Doctor yönlendirmesini açar; erişilebilir ve yapılandırılmış bir Gateway bu TUI kabuğunu Gateway modunda açar; aksi takdirde yapılandırılmış bir yerel model kabuğu yerel modda açar.

## Görüntülenenler

- Üstbilgi: bağlantı URL'si, geçerli aracı, geçerli oturum.
- Sohbet günlüğü: kullanıcı iletileri, asistan yanıtları, sistem bildirimleri, araç kartları.
- Durum satırı: bağlantı/çalıştırma durumu (bağlanıyor, çalışıyor, akış yapılıyor, boşta, hata).
- Altbilgi: aracı + oturum + model + hedef durumu + düşünme/hızlı/ayrıntılı/izleme/akıl yürütme + belirteç sayıları + teslimat.
- Giriş: otomatik tamamlamalı metin düzenleyici.

## Zihinsel model: aracılar + oturumlar

- Aracılar benzersiz kısa adlardır (ör. `main`, `research`). Gateway listeyi sunar.
- Oturumlar geçerli aracıya aittir.
- Oturum anahtarları `agent:<agentId>:<sessionKey>` olarak saklanır.
  - `/session main` yazarsanız TUI bunu `agent:<currentAgent>:main` olarak genişletir.
  - `/session agent:other:main` yazarsanız açıkça o aracı oturumuna geçersiniz.
- Oturum kapsamı:
  - `per-sender` (varsayılan): her aracının birden çok oturumu vardır.
  - `global`: TUI her zaman `global` oturumunu kullanır (seçici boş olabilir).
- Geçerli aracı + oturum her zaman altbilgide görünür.
- Oturumun bir [hedefi](/tr/tools/goal) varsa altbilgi bunun kısa durumunu gösterir:
  `Pursuing goal`, `Goal paused (/goal resume)`, `Goal blocked (/goal resume)` veya `Goal achieved`.
- `--session` olmadan başlatıldığında Gateway modundaki TUI, oturum hâlâ mevcutsa aynı Gateway, aracı ve oturum kapsamı için son seçilen oturumu sürdürür. `--session`, `/session`, `/new` veya `/reset` geçirmek açık bir seçim olmaya devam eder.

## Gönderme + teslimat

- İletiler her zaman Gateway'e (veya yerel modda gömülü çalışma zamanına) gider; asistanın yanıtını yeniden bir sohbet sağlayıcısına teslim etmek, varsayılan olarak kapalı olan ayrı bir adımdır.
- TUI, genel amaçlı bir giden kanal değil, WebChat gibi dahili bir kaynak yüzeyidir. Görünür yanıtlar için `tools.message` gerektiren test düzenekleri, etkin TUI dönüşünü hedefsiz bir `message.send` ile karşılayabilir; açık sağlayıcı teslimatı yine normal yapılandırılmış kanalları kullanır ve hiçbir zaman `lastChannel` seçeneğine geri dönmez.
- Teslimat, başlatma sırasında TUI oturumunun tamamı için sabitlenir: açmak için `openclaw tui --deliver` ile başlatın. Oturum sırasında değiştirmek için `/deliver` eğik çizgi komutu veya Ayarlar anahtarı yoktur; değiştirmek için TUI'yi yeniden başlatın.

## Seçiciler + katmanlar

- Model seçici: kullanılabilir modelleri listeler ve oturum geçersiz kılmasını ayarlar.
- Aracı seçici: farklı bir aracı seçer.
- Oturum seçici: geçerli aracının son 7 gün içinde güncellenmiş en fazla 50 oturumunu gösterir. Bilinen daha eski bir oturuma geçmek için `/session <key>` kullanın.
- Ayarlar (`/settings`): araç çıktısının genişletilmesini ve düşünmenin görünürlüğünü açıp kapatır. Bu panel teslimatı denetlemez.

## Klavye kısayolları

- Enter: ileti gönder
- Esc: etkin çalıştırmayı iptal et
- Ctrl+C: girişi temizle (çıkmak için iki kez basın)
- Ctrl+D: çık
- Ctrl+L: model seçici
- Ctrl+G: aracı seçici
- Ctrl+P: oturum seçici
- Ctrl+O: araç çıktısının genişletilmesini aç/kapat
- Ctrl+T: düşünmenin görünürlüğünü aç/kapat (geçmişi yeniden yükler)

## Eğik çizgi komutları

Çekirdek:

- `/help`
- `/status` (Gateway'e iletilir; oturum/model özetini gösterir)
- `/gateway-status` (diğer adı `/gwstatus`; Gateway bağlantı durumunu doğrudan gösterir)
- `/agent <id>` (veya `/agents`)
- `/session <key>` (veya `/sessions`)
- `/model <provider/model>` (veya `/models`)

Oturum denetimleri:

- `/think <off|minimal|low|medium|high>` (daha yüksek katmanlar modele bağlı olarak `xhigh`/`max` gibi düzeyler ekleyebilir)
- `/fast <status|auto|on|off>`
- `/verbose <on|full|off>`
- `/trace <on|off>`
- `/reasoning <on|off|stream>`
- `/usage <off|tokens|full|reset>` (`reset`/`inherit`/`clear`/`default` oturum geçersiz kılmasını temizler)
- `/goal [status] | /goal start <objective> | /goal edit <objective> | /goal pause|resume|complete|block|clear`
- `/elevated <on|off|ask|full>` (diğer adı: `/elev`)
- `/activation <mention|always>`
- `/queue <steer|followup|collect|interrupt> [debounce:<duration>] [cap:<n>] [drop:<summarize|old|new>]`
- `/queue default` (veya `/queue reset`) oturum geçersiz kılmasını temizler

Oturum yaşam döngüsü:

- `/new` (yeni bir anahtar altında yeni ve yalıtılmış bir oturum oluşturur; eski oturumdaki diğer TUI istemcilerini etkilemez)
- `/reset` (geçerli oturum anahtarını yerinde sıfırlar)
- `/abort` (etkin çalıştırmayı iptal eder)
- `/settings`
- `/exit` (veya `/quit`)

Yalnızca yerel mod:

- `/auth [provider]`, TUI içinde sağlayıcı kimlik doğrulama/oturum açma akışını açar.

Yerel mod, gömülü çalışma zamanı içinde aynı kuyruk modlarını uygular. Çalıştırma
ortasında verilen bir istem, oturumun `/queue` politikasını izler: `steer`, çalışma
zamanı kabul edebildiğinde istemi ekler; `followup` ayrı bir dönüşü bekler; `collect`, bekleyen
istemleri birleştirir ve `interrupt`, yenisini başlatmadan önce geçerli çalıştırmayı
durdurur. Açık `/steer <message>` yalnızca Gateway içindir; yerel modda `/queue steer` ile
normal bir ileti kullanın.

OpenClaw:

- `/openclaw [request]`, isteğe bağlı olarak bir isteği iletip normal aracı TUI'sinden [OpenClaw](#openclaw-setup-and-repair-helper) kurulum/onarım sohbetine döner.

Diğer Gateway eğik çizgi komutları (örneğin `/context`) Gateway'e iletilir ve sistem çıktısı olarak gösterilir. Bkz. [Eğik çizgi komutları](/tr/tools/slash-commands).

## Yerel kabuk komutları

- TUI ana makinesinde yerel bir kabuk komutu çalıştırmak için satırın başına `!` ekleyin.
- TUI, yerel çalıştırmaya izin vermek için oturum başına bir kez onay ister; reddedilirse `!` oturum boyunca devre dışı kalır.
- Komutlar TUI çalışma dizininde yeni ve etkileşimsiz bir kabukta çalışır (kalıcı `cd`/ortam yoktur).
- Yerel kabuk komutlarının ortamına `OPENCLAW_SHELL=tui-local` aktarılır.
- Tek başına bir `!` normal ileti olarak gönderilir; baştaki boşluklar yerel çalıştırmayı tetiklemez.

## OpenClaw kurulum ve onarım yardımcısı

OpenClaw, yapılandırılmış varsayılan model canlı çıkarım denetimini geçtikten sonra `openclaw setup` olarak sunulan, sıfırıncı halka kurulum/onarım asistanıdır. Çıkarım kullanılamıyorsa etkileşimli çağrı çıkarım ilk katılımına döner ve otomasyon onarım yönlendirmesiyle başarısız olur. `openclaw tui --local` ile aynı yerel TUI kabuğunda çalışır ve OpenClaw'ın türü belirlenmiş, onaya tabi işlemleriyle sınırlandırılmış bir yapay zekâ aracısı tarafından desteklenir:

```bash
openclaw setup                       # etkileşimli olarak başlat
openclaw setup -m "status"           # bir istek çalıştır ve çık
openclaw setup -m "set default model openai/gpt-5.2" --yes   # yapılandırma yazımını uygula
```

- Kalıcı yapılandırma yazımları onay gerektirir: etkileşimli olarak onaylayın veya `--yes` geçirin.
- `--json`, sohbeti başlatmak yerine başlangıç genel görünümünü JSON olarak yazdırır.
- OpenClaw içindeyken bir `open-tui` isteği (örneğin normal bir aracıyla konuşma isteği) OpenClaw'dan çıkar ve normal aracı TUI'sini açar; geri dönmek için orada `/openclaw` kullanın.

Geçerli yapılandırma zaten doğrulanıyorsa ve gömülü aracının bunu aynı makinede incelemesini, belgelerle karşılaştırmasını ve çalışan bir Gateway'e bağlı olmadan sapmaları onarmaya yardımcı olmasını istiyorsanız yerel modu kullanın.

`openclaw config validate` zaten başarısız oluyorsa önce `openclaw configure` veya `openclaw doctor --fix` ile başlayın; `openclaw chat` başlamak için yine de yüklenebilir bir yapılandırmaya ihtiyaç duyar.

Tipik döngü:

1. Yerel modu başlatın:

```bash
openclaw chat
```

2. Aracıdan denetlemek istediğiniz şeyi isteyin, örneğin:

```text
Gateway kimlik doğrulama yapılandırmamı belgelerle karşılaştır ve en küçük düzeltmeyi öner.
```

3. Kesin kanıt ve doğrulama için yerel kabuk komutlarını kullanın:

```text
!openclaw config file
!openclaw docs gateway auth token secretref
!openclaw config validate
!openclaw doctor
```

4. `openclaw config set` veya `openclaw configure` ile dar kapsamlı değişiklikler uygulayın, ardından `!openclaw config validate` komutunu yeniden çalıştırın.
5. Doctor otomatik bir geçiş veya onarım önerirse bunu gözden geçirin ve `!openclaw doctor --fix` komutunu çalıştırın.

İpuçları:

- `openclaw.json` dosyasını elle düzenlemek yerine `openclaw config set` veya `openclaw configure` kullanmayı tercih edin.
- `openclaw docs "<query>"`, aynı makinedeki canlı belge dizininde arama yapar.
- Yapılandırılmış şema ve SecretRef/çözümlenebilirlik hataları istediğinizde `openclaw config validate --json` kullanışlıdır.

## Araç çıktısı

- Araç çağrıları, bağımsız değişkenler + sonuçlar içeren kartlar olarak gösterilir.
- Ctrl+O daraltılmış/genişletilmiş görünümler arasında geçiş yapar.
- Araçlar çalışırken kısmi güncellemeler aynı karta aktarılır.

## Terminal renkleri

- TUI, hem koyu hem açık terminallerin okunabilir kalması için asistan gövde metnini terminalinizin varsayılan ön plan renginde tutar.
- Terminaliniz açık renkli bir arka plan kullanıyorsa ve otomatik algılama yanlışsa `openclaw tui` başlatılmadan önce `OPENCLAW_THEME=light` ayarlayın.
- Bunun yerine özgün koyu paleti zorlamak için `OPENCLAW_THEME=dark` ayarlayın.

## Geçmiş + akış

- TUI bağlandığında en son geçmişi yükler (varsayılan 200 ileti).
- Akış yanıtları tamamlanana kadar yerinde güncellenir.
- TUI, daha zengin araç kartları için aracı araç olaylarını da dinler.

## Bağlantı ayrıntıları

- TUI, Gateway politikası için Control UI ve WebChat'in kullandığı modla aynı olan genel `ui` istemci modu altında `openclaw-tui` istemci kimliğiyle bağlanır.
- Yeniden bağlantılar bir sistem iletisi gösterir; olay boşlukları günlükte görünür hâle getirilir.

## Seçenekler

- `--local`: Yerel gömülü agent çalışma zamanında çalıştır
- `--url <url>`: Gateway WebSocket URL'si (varsayılan olarak yapılandırmadaki `gateway.remote.url` veya geri döngüde `ws://127.0.0.1:<port>`)
- `--token <token>`: Gateway token'ı (gerekiyorsa)
- `--password <password>`: Gateway parolası (gerekiyorsa)
- `--tls-fingerprint <sha256>`: Sabitlenmiş bir `wss://` Gateway için beklenen TLS sertifikası parmak izi
- `--session <key>`: Oturum anahtarı (varsayılan: `main`; kapsam genelse `global`)
- `--deliver`: Asistan yanıtlarını sağlayıcıya ilet (varsayılan olarak kapalı)
- `--thinking <level>`: Gönderimler için düşünme düzeyini geçersiz kıl
- `--message <text>`: Bağlandıktan sonra ilk mesajı gönder
- `--timeout-ms <ms>`: Milisaniye cinsinden agent zaman aşımı (varsayılan olarak `agents.defaults.timeoutSeconds`)
- `--history-limit <n>`: Yüklenecek geçmiş girdileri (varsayılan `200`)

<Warning>
`--url` ayarlandığında TUI, yapılandırmadaki veya ortam değişkenlerindeki kimlik bilgilerine geri dönmez. `--token` veya `--password` seçeneğini açıkça; hedef sabitlenmiş bir sertifika kullanıyorsa ayrıca `--tls-fingerprint` seçeneğini iletin. Açık kimlik bilgilerinin eksik olması bir hatadır. Yerel modda `--url`, `--token`, `--password` veya `--tls-fingerprint` seçeneklerini iletmeyin.
</Warning>

## Sorun giderme

Mesaj gönderdikten sonra çıktı yoksa:

- Gateway'in bağlı ve boşta/meşgul olduğunu doğrulamak için TUI'da `/status` komutunu çalıştırın.
- Gateway günlüklerini kontrol edin: `openclaw logs --follow`.
- Agent'ın çalışabildiğini doğrulayın: `openclaw status` ve `openclaw models status`.
- Bir sohbet kanalında mesaj bekliyorsanız TUI'ın `--deliver` ile başlatıldığını doğrulayın (bu seçenek daha sonra yeniden başlatmadan etkinleştirilemez).

## Bağlantı sorunlarını giderme

- `disconnected`: Gateway'in çalıştığından ve `--url/--token/--password` değerlerinizin doğru olduğundan emin olun.
- Seçicide agent yok: `openclaw agents list` ve yönlendirme yapılandırmanızı kontrol edin.
- Oturum seçici boş: genel kapsamda olabilirsiniz veya henüz oturumunuz olmayabilir.

## İlgili içerikler

- [Kontrol Arayüzü](/tr/web/control-ui) — web tabanlı kontrol arayüzü
- [Yapılandırma](/tr/cli/config) — `openclaw.json` öğesini inceleyin, doğrulayın ve düzenleyin
- [Doctor](/tr/cli/doctor) — yönlendirmeli onarım ve geçiş kontrolleri
- [CLI Başvurusu](/tr/cli) — eksiksiz CLI komut başvurusu
