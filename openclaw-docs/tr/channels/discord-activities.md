---
read_when:
    - Discord Activity widget'larını ayarlama veya sorunlarını giderme
summary: Discord Activities içinde bağımsız OpenClaw HTML widget'larını başlatın
title: Discord Etkinlikleri
x-i18n:
    generated_at: "2026-07-26T23:12:04Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: b1bc04443aef89fd514290c3bebdbdd3e9972298b45cae3806bec99344f6d8cd
    source_path: channels/discord-activities.md
    workflow: 16
---

Discord Activities, bir aracının geçerli Discord kanalına etkileşimli, kendi kendine yeten bir HTML widget'ı göndermesini sağlar. Mesaj bir **Widget'ı aç** düğmesi içerir; bu düğmeye tıklandığında widget Discord içinde başlatılır.

Özellik varsayılan olarak kapalıdır. OpenClaw, Activity HTTP rotalarını, `show_widget` aracı aracını ve başlatma düğmesi işleyicisini yalnızca `channels.discord.activities` mevcut olduğunda ve bir istemci gizli anahtarı çözümlendiğinde kaydeder. Kullanımdan kaldırılan `discord_widget` diğer adı bir sürüm boyunca kullanılabilir kalır.

## Ön koşullar

- mevcut bir [OpenClaw Discord botu](/tr/channels/discord)
- OpenClaw Gateway'e ulaşan genel bir HTTPS ana makine adı
- botun Discord uygulaması için Activities ve OAuth2'yi yapılandırma izni

Herhangi bir HTTPS ters proxy'si veya tüneli kullanılabilir. Adlandırılmış bir Cloudflare Tunnel, Gateway bağlantı noktasını doğrudan açığa çıkarmadan kararlı bir ana makine adı sağlar.

```yaml
# ~/.cloudflared/config.yml
tunnel: openclaw-discord
credentials-file: /home/you/.cloudflared/TUNNEL-ID.json
ingress:
  - hostname: openclaw.example.com
    service: http://127.0.0.1:18789
  - service: http_status:404
```

```bash
cloudflared tunnel login
cloudflared tunnel create openclaw-discord
cloudflared tunnel route dns openclaw-discord openclaw.example.com
cloudflared tunnel run openclaw-discord
```

Normal Gateway kimlik doğrulamasını etkin tutun. Yalnızca Activity öneki geneldir; Plugin, OAuth'ı, Activity örneği üyeliğini, kanal bağlamasını, oturumları ve tek kullanımlık belge yeteneklerini kendisi doğrular.

## Kurulum

<Steps>
  <Step title="Gateway'i HTTPS üzerinden erişilebilir hâle getirin">
    Tünelinizi veya ters proxy'nizi başlatın ve Activities yapılandırması eklendikten sonra `https://openclaw.example.com/discord/activity/` adresinin Gateway'e ulaştığını doğrulayın. Örnek ana makine adını kendi adınızla değiştirin.
  </Step>

  <Step title="Discord'da Activities'i etkinleştirin">
    Mevcut bot uygulamasını [Discord Developer Portal](https://discord.com/developers/applications) içinde açın. **Activities** bölümünü açın, Activities'i etkinleştirin ve bir URL eşlemesi oluşturun:

    - önek: `ROOT` (`/`)
    - hedef: `openclaw.example.com/discord/activity`

    Hedef, genel ana makine adı ile `/discord/activity` birleşimidir ve sonunda eğik çizgi bulunmaz.

  </Step>

  <Step title="OAuth2 istemci gizli anahtarını kopyalayın">
    Developer Portal'da **OAuth2** bölümünü açın. Discord en az bir yönlendirme URI'si gerektirir; uygulamada henüz yoksa geri döngü adresi gibi yerel bir yer tutucu ekleyin. Activity dönüş akışını Embedded App SDK yönetir. Uygulama istemci gizli anahtarını kopyalayın veya sıfırlayın. Bunu bir kimlik bilgisi olarak değerlendirin: sohbete, günlüklere veya kaydedilmiş bir yapılandırma dosyasına yapıştırmayın.
  </Step>

  <Step title="OpenClaw'u yapılandırın">
    Widget sunması gereken Discord hesabına bir blok ekleyin:

    ```json5
    {
      channels: {
        discord: {
          token: "${DISCORD_BOT_TOKEN}",
          activities: {
            clientSecret: "${DISCORD_CLIENT_SECRET}",
            // İsteğe bağlıdır. Varsayılan olarak başlangıçta öğrenilen bot uygulaması kimliği kullanılır.
            applicationId: "YOUR_DISCORD_APPLICATION_ID",
          },
        },
      },
    }
    ```

    `DISCORD_CLIENT_SECRET` ayarlandığında `clientSecret` değerini bloktan çıkarabilirsiniz. Etkinleştirmeyi seçmek için bloğun kendisi mevcut kalmalıdır.

    Normal Discord erişim ayarları ayrı kalır. Örneğin `allowFrom`, aracıya kimlerin DM gönderebileceğini denetlemeye devam eder; bir kanalda daha önce gönderilmiş bir widget'ı kimlerin açabileceğini denetlemez.

  </Step>

  <Step title="Yeniden başlatın ve test edin">
    Gateway'i yeniden başlatın. Bir Discord görüşmesinde aracıdan etkileşimli bir widget göstermesini isteyin. Araç `show_widget` çağrısını yapar; gönderilen mesajdaki **Widget'ı aç** düğmesine tıklayın.
  </Step>
</Steps>

## Güvenlik modeli

- OAuth, widget meta verileri döndürülmeden önce Discord kullanıcısını tanımlar.
- Discord'un Get Activity Instance API'si, OAuth kullanıcısının geçerli Activity örneğinde bulunduğunu doğrulamalıdır. Örnek kanalı, widget'ın gönderildiği kanalla eşleşmelidir.
- Discord'un ilgili kanala girmesine izin verdiği herkes kanalın widget'larını açabilir. Hedef kitleyi daraltmak için Discord kanal izinlerini kullanın. OpenClaw komut ve DM izin listeleri, daha önce gönderilmiş kanal içeriğine erişim sağlamaz veya bu erişimi kaldırmaz.
- OAuth oturumlarının süresi 15 dakika sonra dolar. Widget belge yeteneklerinin süresi 60 saniye sonra dolar ve yalnızca bir kez çalışır.
- Widget'ların süresi yedi gün sonra dolar ve Discord Plugin örneği başına en fazla 64 widget saklanır.
- Widget HTML'si aracınız tarafından yazılır ve güvenilir içerik olarak değerlendirilmelidir. Hatalı bir widget'ın açığa çıkarmasını istemeyeceğiniz gizli bilgileri yerleştirmeyin.
- Widget, kendi iç içe çerçevesi içinde gezinebilir. `sandbox="allow-scripts"` iframe'i üst düzey gezinmeyi, açılır pencereleri ve aynı kaynağa erişimi engellerken İçerik Güvenliği Politikası ağ bağlantılarını ve harici kaynakları engeller. Bu denetimler derinlemesine savunma sağlar; widget'ı yazan aracıya karşı bir güvenlik sınırı değildir.
- Activities devre dışı bırakıldığında `/discord/activity` hiçbir şekilde kaydedilmez.

Genel Activity kabuğu ve token değişim rotası, etkinleştirildiklerinde tüneliniz üzerinden erişilebilir hâle gelir. Geçerli bir OAuth oturumu ve tek kullanımlık belge yeteneği olmadan widget HTML'sini açığa çıkarmazlar.

## Sorun giderme

### Activity “Gateway çevrimdışı” diyor

- tünelin çalıştığını ve Gateway'in gerçek bağlama bağlantı noktasına yönlendirildiğini doğrulayın
- Developer Portal hedefinin `/discord/activity` içerdiğini doğrulayın
- Discord veya OpenClaw yapılandırmasını değiştirdikten sonra Gateway'i yeniden başlatın
- eksik Activities istemci gizli anahtarıyla ilgili tek satırlık uyarı için Gateway günlüklerini denetleyin

### Discord boş bir sayfa açıyor veya `blocked:csp` bildiriyor

- URL eşlemesinin `ROOT` kullandığını ve ikinci bir `/discord/activity` segmenti eklemediğini doğrulayın
- kabuğun, `shell.js` öğesinin ve SDK modülünün tamamının Discord proxy'si üzerinden döndüğünü doğrulayın
- `/discord/activity/` altındaki istekler için Gateway günlüklerini inceleyin

Widget ağ istekleri kasıtlı olarak engellenir. Widget'ın ihtiyaç duyduğu tüm CSS, JavaScript, görseller ve verileri satır içine yerleştirin.

### “Widget kullanılamıyor”

Düğmeyi, aracının gönderdiği kanaldan başlatın. OpenClaw, tıklanan başlatmaları sunucu tarafında izler; böylece Discord düğmenin özel kimliğini çıkarsa veya bozsa bile yeni bir başlatma kaydı doğru widget'ı çözümleyebilir. Ne özel kimlik ne de bir başlatma kaydı çözümlenebildiğinde OpenClaw, ilgili kanalda en son gönderilen ve hâlâ etkin olan widget'ı açar. Eski widget'lara, özel kimliklerini koruyan düğmeler üzerinden erişilmeye devam edilebilir.

### “Bu kanalda Activities başlatamazsınız”

Discord, forum gönderisi ileti dizilerinden Activities başlatmaz. OpenClaw widget mesajını ve düğmesini buraya gönderebilir ancak Activity'yi bunun yerine normal bir metin kanalından başlatın. Bu kısıtlama OpenClaw'dan değil, Discord'dan kaynaklanır.
