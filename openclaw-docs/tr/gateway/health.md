---
read_when:
    - Kanal bağlantısını veya Gateway durumunu tanılama
    - Durum denetimi CLI komutlarını ve seçeneklerini anlama
summary: Durum denetimi komutları ve Gateway durum izleme
title: Sağlık kontrolleri
x-i18n:
    generated_at: "2026-07-26T23:20:45Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 59a7fbfb7fb86be7dbd3a03f96c7328c2bc8cc851230c0bdd1b1b750b3014be4
    source_path: gateway/health.md
    workflow: 16
---

Tahminde bulunmadan kanal bağlantısını doğrulamak için kısa kılavuz.

## Hızlı kontroller

- `openclaw status` - yerel özet: Gateway erişilebilirliği/modu, güncelleme ipucu, bağlı kanal kimlik doğrulama yaşı, oturumlar + son etkinlik.
- `openclaw status --all` - eksiksiz yerel tanılama (salt okunur, renkli, hata ayıklama amacıyla yapıştırılması güvenli).
- `openclaw status --deep` - çalışan Gateway'den canlı bir yoklama yapmasını ister (`health` ile `probe:true`), desteklendiğinde hesap başına kanal yoklamaları dâhil.
- `openclaw status --usage` - model sağlayıcısı kullanım/kota anlık görüntülerini gösterir.
- `openclaw health` - çalışan Gateway'den sistem durumu anlık görüntüsünü ister (yalnızca WS; CLI'dan doğrudan kanal soketi yoktur).
- `openclaw health --verbose` (`--debug` diğer adı) - canlı sistem durumu yoklamasını zorlar ve Gateway bağlantı ayrıntılarını yazdırır.
- `openclaw health --json` - makine tarafından okunabilir sistem durumu anlık görüntüsü çıktısı.
- Aracıyı çağırmadan durum yanıtı almak için herhangi bir kanalda bağımsız sohbet komutu olarak `/status` gönderin.
- Günlükler: `openclaw logs --follow` (veya `openclaw --profile <profile> logs --follow`) çalıştırın ve `web-heartbeat`, `web-reconnect`, `web-auto-reply`, `web-inbound` için filtreleyin.

Discord ve diğer sohbet sağlayıcılarında oturum satırları, soketin canlı olduğunu göstermez.
`openclaw sessions`, Gateway `sessions.list` ve aracının `sessions_list` aracı
depolanan konuşma durumunu okur. Bir sağlayıcı yeniden bağlanabilir ve yeni bir oturum
satırı oluşturulmadan önce sağlıklı kanal durumu gösterebilir. Canlı bağlantı kontrolleri
için yukarıdaki kanal durumu ve sistem durumu komutlarını kullanın.

## Ayrıntılı tanılama

- Diskteki kimlik bilgileri: `ls -l ~/.openclaw/credentials/whatsapp/<accountId>/creds.json` (mtime yakın tarihli olmalıdır).
- Oturum deposu: `ls -l ~/.openclaw/agents/<agentId>/agent/openclaw-agent.sqlite`. Sayı ve son alıcılar `status` aracılığıyla gösterilir.
- Yeniden bağlama akışı: günlüklerde 409-515 durum kodları veya `loggedOut` göründüğünde `openclaw channels logout && openclaw channels login --verbose`. QR ile oturum açma akışı, eşleştirmeden sonra 515 durumu için bir kez otomatik olarak yeniden başlatılır.
- Tanılama varsayılan olarak etkindir (`diagnostics.enabled: false` bunları devre dışı bırakır). Bellek olayları RSS/heap bayt sayılarını ve eşik/büyüme baskısını kaydeder. Canlılık uyarıları, işlem çalışırken ancak doygun olduğunda olay döngüsü gecikmesini/kullanımını, CPU çekirdeği oranını ve etkin/bekleyen/kuyruktaki oturum sayılarını kaydeder. Aşırı büyük veri yükü olayları, neyin reddedildiğini/kırpıldığını/parçalara ayrıldığını ve boyutlarla sınırları kaydeder; ileti metnini, ek içeriklerini, Webhook gövdelerini, ham istek/yanıt gövdelerini, belirteçleri, çerezleri veya gizli değerleri asla kaydetmez.
- Aynı Heartbeat, sınırlandırılmış kararlılık kaydedicisini çalıştırır: `openclaw gateway stability` (veya `diagnostics.stability` Gateway RPC'si). Ölümcül Gateway çıkışları, kapatma zaman aşımları ve yeniden başlatma sırasındaki başlatma hataları, en son anlık görüntüyü `~/.openclaw/logs/stability/` altında kalıcı olarak saklar. En yeni paketi `openclaw gateway stability --bundle latest` ile inceleyin.
- Hata raporları için `openclaw gateway diagnostics export` çalıştırın ve oluşturulan zip dosyasını ekleyin: bir Markdown özeti, en yeni kararlılık paketi, temizlenmiş günlük meta verileri, temizlenmiş Gateway durum/sistem durumu anlık görüntüleri ve yapılandırma şekli. Sohbet metni, Webhook gövdeleri, araç çıktıları, kimlik bilgileri, çerezler, hesap/ileti tanımlayıcıları ve gizli değerler atlanır veya sansürlenir. Bkz. [Tanılama Dışa Aktarımı](/tr/gateway/diagnostics).

## Sistem durumu izleyicisi yapılandırması

- `channels.<provider>.healthMonitor.enabled`: genel izlemeyi etkin bırakırken belirli bir kanal için sistem durumu izleyicisinin yeniden başlatmalarını devre dışı bırakır.
- `channels.<provider>.accounts.<accountId>.healthMonitor.enabled`: kanal düzeyindeki ayardan öncelikli olan çok hesaplı geçersiz kılma.
- Bu kanal başına geçersiz kılmalar, günümüzde bunları sunan yerleşik kanallara uygulanır: Discord, Google Chat, iMessage, IRC, Microsoft Teams, Signal, Slack, Telegram ve WhatsApp.

## Çalışma süresi izleme

Harici çalışma süresi izleme hizmetleri `/v1/chat/completions` yerine özel `/health` uç noktasını kullanmalıdır.

- **KULLANIN:** `GET /health` - anında yanıt verir, oturum oluşturmaz, LLM çağrısı yapmaz, `{"ok":true,"status":"live"}` döndürür
- **KULLANMAYIN:** sistem durumu kontrolleri için `/v1/chat/completions` - her istek Skills anlık görüntüsü, bağlam derlemesi ve LLM çağrılarıyla eksiksiz bir aracı oturumu oluşturur

`x-openclaw-session-key` üstbilgisi veya `user` alanı sağlanmadığında, `/v1/chat/completions` her istek için yeni ve rastgele bir oturum oluşturur. Her 15 dakikada bir yoklama yapan izleme hizmetleri, her biri 4-22KB tüketen günde yaklaşık 96 oturum oluşturur. Bu durum zamanla oturum deposunun şişmesine neden olur ve bağlam penceresinin taşmasına yol açabilir.

### İzleme hizmeti kurulum örnekleri

- **BetterStack:** Sistem durumu kontrolü URL'sini `https://<your-gateway-host>:<port>/health` olarak ayarlayın
- **UptimeRobot:** `https://<your-gateway-host>:<port>/health` URL'siyle yeni bir HTTP izleyicisi ekleyin
- **Genel:** Gateway sağlıklı olduğunda `/health` adresine yapılan herhangi bir HTTP GET isteği, `{"ok":true}` ile 200 döndürür

## Bir şey başarısız olduğunda

- `logged out` veya 409-515 durumu -> `openclaw channels logout`, ardından `openclaw channels login` ile yeniden bağlayın.
- Gateway'e erişilemiyor -> başlatın: `openclaw gateway --port 18789` (bağlantı noktası meşgulse `--force` kullanın).
- Gelen ileti yok -> bağlı telefonun çevrimiçi ve gönderenin izinli olduğunu doğrulayın (`channels.whatsapp.allowFrom`); grup sohbetleri için izin verilenler listesi + bahsetme kurallarının eşleştiğinden emin olun (`channels.whatsapp.groups`, `agents.entries.*.groupChat.mentionPatterns`).

## Özel "health" komutu

`openclaw health`, çalışan Gateway'den sistem durumu anlık görüntüsünü ister (CLI'dan doğrudan
kanal soketi yoktur). Varsayılan olarak güncel, önbelleğe alınmış bir Gateway anlık görüntüsü döndürür ve
Gateway bu önbelleği arka planda yeniler; `--verbose` bunun yerine canlı yoklamayı zorlar.
Komut, mevcut olduğunda bağlı kimlik bilgilerini/kimlik doğrulama yaşını, kanal başına yoklama özetlerini,
oturum deposu özetini ve yoklama süresini bildirir. Gateway'e erişilemiyorsa veya yoklama
başarısız olur/zaman aşımına uğrarsa sıfır olmayan kodla çıkar.

Seçenekler:

- `--json`: makine tarafından okunabilir JSON çıktısı
- `--timeout <ms>`: varsayılan 10s yoklama zaman aşımını geçersiz kılar
- `--verbose`: canlı yoklamayı zorlar ve Gateway bağlantı ayrıntılarını yazdırır
- `--debug`: `--verbose` için diğer ad

Sistem durumu anlık görüntüsü şunları içerir: `ok` (boole), `ts` (zaman damgası), `durationMs` (yoklama süresi), kanal başına durum, aracı kullanılabilirliği ve oturum deposu özeti.

## İlgili

- [Gateway işletim kılavuzu](/tr/gateway)
- [Tanılama dışa aktarımı](/tr/gateway/diagnostics)
- [Gateway sorun giderme](/tr/gateway/troubleshooting)
