---
read_when:
    - Hata raporu veya destek talebi hazırlama
    - Gateway çökmelerinde, yeniden başlatmalarında, bellek baskısında veya aşırı büyük yüklerde hata ayıklama
    - Kaydedilen veya karartılan tanılama verilerini inceleme
summary: Hata raporları için paylaşılabilir Gateway tanılama paketleri oluşturun
title: Tanılama dışa aktarımı
x-i18n:
    generated_at: "2026-07-26T23:40:43Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 97a805fed8d51de2e63e5c6a12ce03e91701d69654882cca7795c9f3553b1c55
    source_path: gateway/diagnostics.md
    workflow: 16
---

OpenClaw, hata raporları için yerel bir tanılama `.zip` oluşturabilir: arındırılmış Gateway
durumu, sistem sağlığı, günlükler, yapılandırma şekli ve yük içermeyen son kararlılık olayları.

İncelenene kadar tanılama paketlerine gizli bilgiler gibi davranın. Yükler ve kimlik bilgileri
tasarım gereği sansürlenir, ancak paket yine de yerel Gateway günlüklerini ve
ana makine düzeyindeki çalışma zamanı durumunu özetler.

## Hızlı başlangıç

```bash
openclaw gateway diagnostics export
```

Yazılan zip dosyasının yolunu görüntüler. Bir çıktı yolu seçin:

```bash
openclaw gateway diagnostics export --output openclaw-diagnostics.zip
```

Otomasyon için:

```bash
openclaw gateway diagnostics export --json
```

## Sohbet komutu

Sahipler, tek parça hâlinde kopyalanıp yapıştırılabilen bir destek raporu olarak yerel bir
Gateway dışa aktarımı istemek için herhangi bir konuşmada `/diagnostics [note]` çalıştırabilir:

1. `/diagnostics` gönderin; isteğe bağlı olarak kısa bir not ekleyin (`/diagnostics bad tool choice`).
2. OpenClaw bir ön açıklama gönderir ve
   `openclaw gateway diagnostics export --json` komutunu çalıştıran tek bir açık yürütme onayı ister. Tanılamayı
   tümüne izin veren bir kural aracılığıyla onaylamayın.
3. Onaydan sonra OpenClaw; yerel paket yolu, bildirim özeti,
   gizlilik notları ve ilgili oturum kimlikleriyle yanıt verir.

Grup sohbetlerinde bir sahip yine de `/diagnostics` çalıştırabilir, ancak OpenClaw
dışa aktarma sonucunu, onay istemlerini ve Codex oturum/iş parçacığı dökümünü
sahibe özel olarak gönderir. Grup yalnızca tanılamanın özel olarak gönderildiğini
belirten kısa bir bildirim görür. Sahibe ulaşan özel bir yol yoksa komut güvenli biçimde
başarısız olur ve sahibin komutu bir DM üzerinden çalıştırmasını ister.

Etkin oturum yerel OpenAI Codex koşum takımını kullandığında, aynı yürütme
onayı OpenClaw'ın bildiği Codex iş parçacıkları için OpenAI'a geri bildirim yüklenmesini
de kapsar. Bu yükleme yerel Gateway zip dosyasından ayrıdır ve yalnızca
Codex koşum takımı oturumlarında gerçekleşir. Onay istemi, onaylamanın
Codex oturum veya iş parçacığı kimliklerini listelemeden Codex geri bildirimini de
göndereceğini belirtir. Onaydan sonra yanıt; OpenAI'a gönderilen iş parçacıkları için
kanalları, OpenClaw oturum kimliklerini, Codex iş parçacığı kimliklerini ve
yerel sürdürme komutlarını listeler. Onayın reddedilmesi veya
yok sayılması; dışa aktarımı, Codex geri bildirim yüklemesini ve
Codex kimlik listesini atlar.

Bu, Codex hata ayıklama döngüsünü kısaltır: bir kanalda hatalı davranışı fark edin,
`/diagnostics` çalıştırın, bir kez onaylayın, raporu paylaşın ve ardından iş parçacığını
kendiniz incelemek istiyorsanız görüntülenen `codex resume <thread-id>` komutunu
yerel olarak çalıştırın. Bkz. [Codex koşum takımı](/tr/plugins/codex-harness#inspect-codex-threads-locally).

## Dışa aktarımın içeriği

- `summary.md`: destek için insanlar tarafından okunabilir genel bakış.
- `diagnostics.json`: yapılandırma, günlükler, durum, sistem sağlığı
  ve kararlılık verilerinin makine tarafından okunabilir özeti.
- `manifest.json`: dışa aktarma meta verileri ve dosya listesi.
- Arındırılmış yapılandırma şekli ve gizli olmayan yapılandırma ayrıntıları.
- Arındırılmış günlük özetleri ve sansürlenmiş son günlük satırları.
- Olanaklar dâhilinde alınan Gateway durum ve sistem sağlığı anlık görüntüleri.
- `stability/latest.json`: mevcut olduğunda kalıcı hâle getirilmiş en yeni kararlılık paketi.

Dışa aktarım, Gateway sağlıksız olduğunda da yararlıdır: durum/sistem sağlığı
istekleri başarısız olursa yerel günlükler, yapılandırma şekli ve en son kararlılık paketi
mevcut olduklarında yine toplanır.

## Gizlilik modeli

Saklananlar: alt sistem adları, plugin kimlikleri, sağlayıcı kimlikleri, kanal kimlikleri, yapılandırılmış
modlar, durum kodları, süreler, bayt sayıları, kuyruk durumu, bellek okumaları,
arındırılmış günlük meta verileri, sansürlenmiş operasyonel mesajlar, yapılandırma şekli ve
gizli olmayan özellik ayarları.

Atlanan veya sansürlenenler: sohbet metni, istemler, talimatlar, webhook gövdeleri, araç
çıktıları, kimlik bilgileri, API anahtarları, belirteçler, çerezler, gizli değerler, ham
istek/yanıt gövdeleri, hesap kimlikleri, mesaj kimlikleri, ham oturum kimlikleri,
ana makine adları ve yerel kullanıcı adları.

Bir günlük mesajı kullanıcı, sohbet, istem veya araç yükü metni gibi göründüğünde
dışa aktarım yalnızca bir mesajın atlandığı bilgisini ve bayt sayısını saklar.

## Kararlılık kaydedicisi

Gateway, tanılama etkinleştirildiğinde varsayılan olarak sınırlı ve yük içermeyen bir
kararlılık akışı kaydeder. İçeriği değil, operasyonel olguları yakalar.

Aynı heartbeat, olay döngüsü veya CPU doygun göründüğünde canlılığı da
örnekleyerek olay döngüsü gecikmesi, olay döngüsü kullanımı, CPU çekirdeği oranı,
etkin/bekleyen/kuyruğa alınmış oturum sayıları, geçerli başlatma/çalışma zamanı aşaması
(biliniyorsa), son aşama aralıkları ve sınırlı iş etiketleri içeren
`diagnostic.liveness.warning` olaylarını yayınlar. Bunlar yalnızca iş bekliyorsa veya kuyruğa alınmışsa
ya da etkin iş sürekli olay döngüsü gecikmesiyle çakışıyorsa Gateway `warn`
düzeyinde günlük satırlarına dönüşür; aksi takdirde `debug` düzeyinde günlüğe kaydedilir.
Boştaki canlılık örnekleri yine tanılama olayları olarak kaydedilir, ancak tek başlarına
hiçbir zaman uyarı düzeyine yükseltilmez.

Başlatma aşamaları, duvar saati ve CPU zamanlamasıyla `diagnostic.phase.completed`
olaylarını yayınlar. Takılı kalan gömülü çalıştırma tanılamaları, son köprü ilerlemesi
sonlandırıcı görünüyorsa (örneğin ham bir yanıt öğesi veya yanıt tamamlama olayı)
ancak Gateway gömülü çalıştırmayı hâlâ etkin kabul ediyorsa `terminalProgressStale=true`
işaretini ekler.

Canlı kaydediciyi inceleyin:

```bash
openclaw gateway stability
openclaw gateway stability --type payload.large
openclaw gateway stability --json
```

Önemli bir çıkış, kapatma zaman aşımı veya yeniden başlatma hatasından sonra kalıcı hâle
getirilmiş en yeni paketi inceleyin:

```bash
openclaw gateway stability --bundle latest
```

Kalıcı hâle getirilmiş en yeni paketten bir tanılama zip dosyası oluşturun:

```bash
openclaw gateway stability --bundle latest --export
```

Olaylar mevcut olduğunda kalıcı paketler `~/.openclaw/logs/stability/` altında bulunur.

## Yararlı seçenekler

```bash
openclaw gateway diagnostics export \
  --output openclaw-diagnostics.zip \
  --log-lines 5000 \
  --log-bytes 1000000
```

| Bayrak                  | Varsayılan                                                                    | Açıklama                                           |
| ----------------------- | ----------------------------------------------------------------------------- | -------------------------------------------------- |
| `--output <path>`       | `$OPENCLAW_STATE_DIR/logs/support/openclaw-diagnostics-<timestamp>-<pid>.zip` | Belirli bir zip yoluna (veya dizine) yaz.          |
| `--log-lines <count>`   | `5000`                                                                        | Eklenecek en fazla arındırılmış günlük satırı.      |
| `--log-bytes <bytes>`   | `1000000`                                                                     | İncelenecek en fazla günlük baytı.                  |
| `--url <url>`           | -                                                                             | Durum/sistem sağlığı anlık görüntüleri için Gateway WebSocket URL'si. |
| `--token <token>`       | -                                                                             | Durum/sistem sağlığı anlık görüntüleri için Gateway belirteci. |
| `--password <password>` | -                                                                             | Durum/sistem sağlığı anlık görüntüleri için Gateway parolası. |
| `--timeout <ms>`        | `3000`                                                                        | Durum/sistem sağlığı anlık görüntüsü zaman aşımı.   |
| `--no-stability-bundle` | kapalı                                                                        | Kalıcı kararlılık paketi aramasını atla.            |
| `--json`                | kapalı                                                                        | Makine tarafından okunabilir dışa aktarma meta verilerini görüntüle. |

## Tanılamayı devre dışı bırakma

Tanılama varsayılan olarak etkindir. Kararlılık kaydedicisini ve
tanılama olayı toplamayı devre dışı bırakmak için:

```json5
{
  diagnostics: {
    enabled: false,
  },
}
```

Tanılamanın devre dışı bırakılması hata raporu ayrıntılarını azaltır; normal
Gateway günlük kaydını etkilemez.

Bellek baskısı olayları, dosya sistemi taraması yapmadan veya OOM öncesi
anlık görüntü yazmadan RSS, yığın, eşik ve büyüme olgularını
(`rss_threshold`, `heap_threshold`, `rss_growth`) kaydeder.

## İlgili konular

- [Sistem sağlığı kontrolleri](/tr/gateway/health)
- [Gateway CLI](/tr/cli/gateway#gateway-diagnostics-export)
- [Gateway protokolü](/tr/gateway/protocol#rpc-method-families)
- [Günlük kaydı](/tr/logging)
- [OpenTelemetry dışa aktarımı](/tr/gateway/opentelemetry) - tanılamayı bir toplayıcıya aktarmaya yönelik ayrı akış
