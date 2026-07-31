---
doc-schema-version: 1
read_when:
    - OpenClaw'u birden fazla kullanıcı veya kuruluş için barındırıyorsunuz
    - Kiracı iş yükleri için bir yalıtım sınırı seçmeniz gerekir
summary: Birden çok kiracı güven etki alanını, kiracı başına bir yalıtılmış OpenClaw Gateway hücresi olarak barındırın
title: Çok kiracılı barındırma
x-i18n:
    generated_at: "2026-07-26T23:19:57Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 383d32331b45d40db6fb4ff8242dd9a3cf8898a3ccab19f0372cd06bbd83fc05
    source_path: gateway/multi-tenant-hosting.md
    workflow: 16
---

# Çok kiracılı barındırma

OpenClaw'ın varsayılan güvenlik modeli, paylaşılan tek bir Gateway içinde kötü niyetli çok kiracılı yalıtım değil, Gateway başına tek bir güvenilir operatör sınırıdır. Bu nedenle, aynı güven sınırını paylaşmayan kullanıcıları veya kuruluşları barındırmak, her kiracı için ayrı ve eksiksiz bir OpenClaw örneği çalıştırmak anlamına gelir.

`openclaw fleet` her yalıtılmış örneği bir **hücre** olarak adlandırır. Hücre; kendi durumu, kimlik bilgileri, çalışma alanı, kanal hesapları, token'ı ve yalnızca geri döngüye açık ana makine bağlantı noktası bulunan, güçlendirilmiş bir konteyner içindeki eksiksiz bir Gateway'dir.

Fleet **deneyseldir**: Komutları, bayrakları ve konteyner profili, kullanımdan kaldırma süresi olmadan sürümler arasında değişebilir.

Fleet, Linux ve macOS ana makinelerinde test edilmiştir. Windows ana makineleri şu anda test edilmemiştir.

## Her kiracı neden bir hücreye ihtiyaç duyar?

Tek bir Gateway içindeki kimliği doğrulanmış operatör, güvenilir bir kontrol düzlemi rolüne sahiptir. Oturum kimlikleri yönlendirmeyi seçer; bir kiracıyı diğerine karşı yetkilendirmez. Ajan korumalı alanı, güvenilmeyen içeriğin ve araç yürütmenin etkisini azaltabilir ancak paylaşılan tek bir Gateway'i kiracı yetkilendirme sınırına dönüştürmez.

Her güven etki alanının ayrı bir Gateway sürecine, konteynere, kalıcı durum ağacına ve Gateway kimlik bilgisine sahip olması için kiracı başına bir hücre kullanın. Bu, [Gateway güvenlik modelini](/tr/gateway/security) izler: birbirine güvenmeyen kullanıcıları tek bir OpenClaw sürecinde veya tek bir işletim sistemi kullanıcısı altında birlikte barındırmayın.

## Mimari

Fleet CLI, ana makine tarafında çalışan bir yaşam döngüsü gözetmenidir. Hücreleri OpenClaw durum veritabanına kaydeder ve yerel Docker veya Podman çalışma zamanından bunların konteynerlerini oluşturmasını, incelemesini, başlatmasını, durdurmasını, değiştirmesini ve kaldırmasını ister. Fleet'in bağlama yolları ve geri döngü URL'leri yerel ana makineye ait olduğundan uzak çalışma zamanı uç noktaları desteklenmez. Fleet, kiracı mesajlarına proxy görevi yapmaz ve hücreler arasında paylaşılan uygulama düzeyinde bir veri yolu eklemez.

Her hücre, kendi kullanıcı tanımlı köprü ağında resmi `ghcr.io/openclaw/openclaw` imajını çalıştırır. Ayrı köprüler, sağlayıcılar ve kanallar için giden NAT erişimini korurken hücreler arasındaki doğrudan konteyner IP trafiğini engeller. Giden trafik varsayılan olarak kısıtlanmaz. Podman hücreleri, yayımlanan geri döngü Gateway bağlantı noktasını korurken giden trafiği engellemek için `--network internal` kullanabilir. Docker'ın dahili ağları bu yayımlanan bağlantı noktasını bozduğundan Fleet bu birleşimi reddeder; bunun yerine Docker giden trafik politikasını `DOCKER-USER` zinciri gibi ana makine güvenlik duvarı kurallarıyla uygulayın. Hücre Gateway'i konteyner içinde `18789` bağlantı noktasını dinlerken çalışma zamanı bunu ana makinede yalnızca `127.0.0.1:<allocated-port>` adresinde yayımlar. Uzaktan erişim gerektiğinde operatör, bu geri döngü uç noktasının önüne onaylı bir ters proxy, SSH tüneli veya tailnet yerleştirebilir.

Kalıcı Gateway durumu `<state-dir>/fleet/cells/<tenant>/` kaynağından gelir ve `/home/node/.openclaw` konumuna bağlanır. Kimlik doğrulama profili şifreleme anahtarları ayrı `<state-dir>/fleet/auth-profile-secrets/<tenant>/` ana makine yolundan gelir ve resmi [Docker kalıcılık düzeniyle](/tr/install/docker#storage-and-persistence) eşleşecek şekilde `/home/node/.config/openclaw` konumuna bağlanır. Anahtar, sıradan durum bağlama noktasının altında iç içe değildir. Kiracı başına kanal hesapları, bunlara sahip olan hücre içinde sonlandırılır; Fleet, paylaşılan kanal hesabı veya gelen mesaj yönlendiricisi sağlamaz.

Resmi imaj, varsayılan olarak UID 1000'e sahip root olmayan `node` kullanıcısını kullanır. Fleet, özel bağlama noktalarının yazılabilir kalması için ana makineyle uyumlu kullanıcı eşlemeleri kullanır: Podman `keep-id` kullanır, root yetkili Docker komutu çağıran root olmayan kimliği kullanır ve root olmayan Docker konteyner root kullanıcısını ayrıcalıksız daemon kullanıcısına eşler. Ana makine SELinux etkin olduğunda Docker ve Podman özel bir `:Z` yeniden etiketlemesi uygular. Konteyner profili ayrıcalıklı ana makine özelliklerinden kaçınır ve root olmayan kullanıma uygundur; ancak root olmayan çalışma, ana makine çalışma zamanına ilişkin bir seçim ve ön koşuldur, Fleet'in otomatik olarak etkinleştirdiği bir özellik değildir.

## Güven sınırı

Çok kiracılık, kiracıları birbirinden korur. Fleet operatörüne ve ana makineye her kiracı güvenir. Güvenliği ihlal edilmiş bir ana makineye karşı dayanıklılık hedeflenmez.

Bu, ana makine yöneticisinin konteyner yapılandırmasını ve ortamını inceleyebileceği, bağlanan hücre verilerini okuyabileceği, imajları değiştirebileceği veya konteynerlere girebileceği anlamına gelir. Gateway token'ları ve `--env` ile geçirilen değerler, Docker veya Podman incelemesi aracılığıyla bir yönetici tarafından görülebilir. Ana makine denetimlerini, yönetim erişimi politikasını, izlemeyi, yedeklemeleri ve onaylı bir gizli bilgi yöneticisini buna göre kullanın.

Temel profil, yanlışlıkla joker karakterli ağ erişimine açılmayı önler ve yaygın konteyner ayrıcalık yükseltme araçlarını kaldırır; ancak güvenilmeyen bir ana makineyi güvenli hâle getirmez.

## Yalıtım basamakları

Barındırdığınız kiracılara uygun sınırı seçin:

1. **Güçlendirilmiş konteyner temel profili.** Fleet tüm Linux yeteneklerini kaldırır, `no-new-privileges` özelliğini etkinleştirir, PID, bellek, CPU ve isteğe bağlı yazılabilir katman disk sınırlarını uygular, ayrı kalıcı bağlama noktaları ve hücre başına ağlar kullanır ve yalnızca ana makine geri döngüsünde yayın yapar. Köprü ağı, giden trafiği kısıtlamaz; bir hücrenin giden bağlantılar başlatmaması gerektiğinde Podman `--network internal` veya Docker ana makine güvenlik duvarı politikasını kullanın. Bu, operatöre ve ana makineye güvenen kiracılar için varsayılan profildir.
2. **Daha güçlü konteyner veya sanal makine yalıtımı.** Daha yüksek riskli iş yükleri için Docker veya Podman'ı gVisor ya da Kata Containers gibi daha güçlü bir OCI yalıtım çalışma zamanı kullanacak şekilde yapılandırın veya hücreleri microVM'lere yerleştirin. Bu, çalışma zamanı veya altyapı yapılandırmasıdır; Fleet'in `--runtime docker|podman` seçeneği OCI yalıtım arka ucunu değil, konteyner CLI'sini seçer. Docker'ın [alternatif konteyner çalışma zamanlarına](https://docs.docker.com/engine/daemon/alternative-runtimes/) ve [Docker VM çalışma zamanı kılavuzuna](/tr/install/docker-vm-runtime) bakın.
3. **Kötü niyetli kiracılar için ayrı makineler.** Kötü niyetli kiracıları tek bir OpenClaw sürecinde veya işletim sistemi kullanıcısı altında birlikte barındırmayın. Kiracılar aynı ana makine operatörüne güvenmiyorsa veya daha güçlü bir yönetim sınırına ihtiyaç duyuyorsa ayrı çalışma zamanı yönetimine sahip ayrı sanal makineler veya fiziksel ana makineler kullanın.

Bu basamakların hiçbiri OpenClaw uygulamasının güven modelini değiştirmez: Bir Gateway, tek bir güvenilir operatör etki alanı olmaya devam eder.

## Hızlı başlangıç

Bir hücre oluşturun. Komut, oluşturulan Gateway token'ını bir kez yazdırır; bu nedenle hemen saklayın:

```bash
openclaw fleet create acme
```

Bildirilen `http://127.0.0.1:<port>` URL'sini Fleet ana makinesinde açın, ilgili kiracının token'ıyla kimlik doğrulaması yapın ve sağlayıcı kimlik bilgileriyle kanal hesaplarını hücre içinde yapılandırın.

Konteyner durumunu ve Gateway'in çalışır durumda olup olmadığını denetleyin:

```bash
openclaw fleet status acme
```

Ana makine bağlantı noktasını, bağlanan verileri, kaynak profilini, kullanıcı tarafından sağlanan ortamı ve Gateway token'ını koruyarak yükseltin:

```bash
openclaw fleet upgrade acme
```

Kiracı verilerini koruyarak konteyneri ve kayıt defteri satırını kaldırın:

```bash
openclaw fleet rm acme --force
```

Kalıcı kiracı verilerini de silmek için `--purge-data` ekleyin. Temizleme işlemi `--force` gerektirir, geri alınamaz ve herhangi bir şeyi silmeden önce çözümlenmiş yol kapsama denetimi gerçekleştirir:

```bash
openclaw fleet rm acme --purge-data --force
```

Tüm komutlar ve seçenekler için [`openclaw fleet` CLI başvurusuna](/tr/cli/fleet) bakın.

## Geçerli kapsam

Fleet şu yüzeyleri sağlamaz:

- Paylaşılan kanal hesapları veya paylaşılan bir giriş yönlendiricisi
- Eksiksiz OpenClaw örnekleri yerine sadeleştirilmiş kiracı başına ana makine süreçleri
- Tek bir gözetmen tarafından yönetilen uzak hücre ana makineleri
- Kiracı self servis portalı, faturalandırma düzlemi veya devredilmiş yönetim kullanıcı arayüzü

Bu yetenekler açık kimlik, yönlendirme, yetkilendirme ve hata etki alanı sözleşmeleri gerektirir. Tek bir Gateway'i veya kimlik bilgilerini kiracılar arasında paylaşarak bunları yaklaşık olarak uygulamaya çalışmayın. Fleet, tek ana makineli bir yaşam döngüsü gözetmenidir; çok makineli, kimlikle yönetilen filolar ayrı bir kontrol düzlemi katmanı gerektirir.

## İlgili

- [`openclaw fleet`](/tr/cli/fleet)
- [Gateway güvenliği](/tr/gateway/security)
- [Birden fazla Gateway](/tr/gateway/multiple-gateways)
- [Docker](/tr/install/docker)
- [Podman](/tr/install/podman)
