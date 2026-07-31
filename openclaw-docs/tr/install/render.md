---
read_when:
    - OpenClaw'u Render'a Dağıtma
    - Render Blueprints ile bildirimsel bir bulut dağıtımı istiyorsunuz
summary: OpenClaw'u Kod Olarak Altyapı ile Render üzerinde dağıtma
title: Render
x-i18n:
    generated_at: "2026-07-27T00:03:47Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: a5fbb3c6df04e186df958a62a6130da4e3e485acfeecc7e85fee0d5b69a0438f
    source_path: install/render.mdx
    workflow: 16
---

Deponun `render.yaml` Blueprint'unu kullanarak OpenClaw'u [Render](https://render.com) üzerinde dağıtın. Hizmeti, diski ve ortam değişkenlerini tek bir dosyada tanımlar.

## Ön koşullar

- Bir [Render hesabı](https://render.com) (ücretsiz katman mevcuttur)
- Tercih ettiğiniz [model sağlayıcısından](/tr/providers) bir API anahtarı

## Dağıtım

[Render'a dağıt](https://render.com/deploy?repo=https://github.com/openclaw/openclaw)

Bu işlem, `render.yaml` üzerinden bir Render hizmeti oluşturur, Docker imajını derler ve dağıtır. Hizmet URL'niz `https://<service-name>.onrender.com` kalıbını izler.

## Blueprint

```yaml
services:
  - type: web
    name: openclaw
    runtime: docker
    plan: starter
    healthCheckPath: /health
    envVars:
      - key: OPENCLAW_GATEWAY_PORT
        value: "8080"
      - key: OPENCLAW_STATE_DIR
        value: /data/.openclaw
      - key: OPENCLAW_WORKSPACE_DIR
        value: /data/workspace
      - key: OPENCLAW_GATEWAY_TOKEN
        generateValue: true # güvenli bir belirteci otomatik olarak oluşturur
    disk:
      name: openclaw-data
      mountPath: /data
      sizeGB: 1
```

| Özellik               | Amaç                                                    |
| --------------------- | ---------------------------------------------------------- |
| `runtime: docker`     | Deponun Dockerfile dosyasından derler                          |
| `healthCheckPath`     | Render, `/health` öğesini izler ve sağlıksız örnekleri yeniden başlatır |
| `generateValue: true` | Kriptografik olarak güvenli bir değeri otomatik oluşturur            |
| `disk`                | Yeniden dağıtımlarda korunan kalıcı depolama                 |

## Plan seçimi

| Plan      | Durdurma         | Disk          | En uygun kullanım                      |
| --------- | ----------------- | ------------- | ----------------------------- |
| Ücretsiz      | 15 dk. boşta kaldıktan sonra | Kullanılamaz | Testler, demolar                |
| Başlangıç   | Hiçbir zaman             | 1GB+          | Kişisel kullanım, küçük ekipler     |
| Standart+ | Hiçbir zaman             | 1GB+          | Üretim, birden fazla kanal |

Blueprint varsayılan olarak `starter` kullanır. Ücretsiz katmanı kullanmak için çatalınızdaki `render.yaml` içinde `plan: free` değerini değiştirin. Kalıcı disk olmadığında OpenClaw durumunun her dağıtımda sıfırlanacağını unutmayın.

## Dağıtımdan sonra

### Kontrol Kullanıcı Arayüzüne erişim

Web panosu `https://<your-service>.onrender.com/` adresinde kullanılabilir. Paylaşılan gizli anahtarı kullanarak bağlanın: otomatik oluşturulan `OPENCLAW_GATEWAY_TOKEN` (**Dashboard → your service → Environment** bölümünde bulabilirsiniz) veya parola kimlik doğrulamasına geçtiyseniz parolanız.

### Günlükler

**Dashboard → your service → Logs**, derleme günlüklerini (Docker imajının oluşturulması), dağıtım günlüklerini (hizmetin başlatılması) ve çalışma zamanı günlüklerini (uygulama çıktısı) gösterir.

### Kabuk erişimi

**Dashboard → your service → Shell** bir kabuk oturumu açar. Kalıcı disk `/data` konumuna bağlanır.

### Ortam değişkenleri

Değişkenleri **Dashboard → your service → Environment** bölümünde düzenleyin. Değişiklikler otomatik yeniden dağıtımı tetikler.

### Otomatik dağıtım

Bağlı deponun dalına yeni bir işleme eklendiğinde Render otomatik olarak yeniden dağıtır. Kendi çatalınız yerine doğrudan `openclaw/openclaw` üzerinden dağıtım yaptıysanız bunu tetikleyecek gönderme erişiminiz olmaz; bu nedenle Dashboard'dan manuel bir Blueprint eşitlemesi çalıştırarak güncelleyin veya hizmeti kendi çatalınıza yönlendirin.

## Özel alan adı

1. **Dashboard → your service → Settings → Custom Domains**
2. Alan adınızı ekleyin
3. DNS'yi belirtildiği şekilde yapılandırın (`*.onrender.com` hedefine CNAME)
4. Render otomatik olarak bir TLS sertifikası sağlar

## Ölçeklendirme

- **Dikey**: daha fazla CPU/RAM için planı değiştirin. OpenClaw için genellikle yeterlidir.
- **Yatay**: örnek sayısını artırın (Standart plan ve üzeri). OpenClaw çalışma zamanı durumunu yerel diskte tuttuğundan yapışkan oturumlar veya harici durum yönetimi gerektirir.

## Yedeklemeler ve geçiş

Render Dashboard kabuğundan durumu, yapılandırmayı, kimlik doğrulama profillerini ve çalışma alanını istediğiniz zaman dışa aktarın:

```bash
openclaw backup create
```

Bu işlem taşınabilir bir yedekleme arşivi oluşturur. Bkz. [Yedekleme](/tr/cli/backup).

## Sorun giderme

### Hizmet başlamıyor

Render Dashboard'daki dağıtım günlüklerini kontrol edin. Yaygın sorunlar:

- `OPENCLAW_GATEWAY_TOKEN` eksik — **Dashboard → Environment** bölümünde ayarlandığını doğrulayın
- Bağlantı noktası uyuşmazlığı — Gateway'in Render'ın beklediği bağlantı noktasına bağlanması için `OPENCLAW_GATEWAY_PORT=8080` olduğundan emin olun

### Yavaş soğuk başlatmalar (ücretsiz katman)

Ücretsiz katmandaki hizmetler 15 dakika işlem yapılmadığında durdurulur; durdurmadan sonraki ilk istek, kapsayıcı başlatılırken birkaç saniye sürer. Her zaman açık kullanım için Başlangıç planına yükseltin.

### Yeniden dağıtımdan sonra veri kaybı

Ücretsiz katmanda gerçekleşir (kalıcı disk yoktur). Ücretli bir plana yükseltin veya Render kabuğundan `openclaw backup create` ile düzenli olarak bir yedekleme dışa aktarın.

### Sistem durumu denetimi hataları

Derlemeler başarılı olduğu hâlde dağıtımlar başarısız oluyorsa hizmetin başlatılması çok uzun sürüyor veya `/health` erişilebilir olmayabilir. Şunları kontrol edin:

- Hatalar için derleme günlükleri
- Kapsayıcının `docker build && docker run` ile yerel olarak çalışıp çalışmadığı

## Sonraki adımlar

- Mesajlaşma kanallarını ayarlayın: [Kanallar](/tr/channels)
- Gateway'i yapılandırın: [Gateway yapılandırması](/tr/gateway/configuration)
- OpenClaw'u güncel tutun: [Güncelleme](/tr/install/updating)
