---
read_when:
    - Gateway tarafından başlatılan bulut çalışanlarını işletme veya hata ayıklama
    - Çalışan kabulünü, oturum atamasını veya yerel araç yalıtımını doğrulama
summary: Kısıtlı bulut worker çalışma zamanı için dahili operatör başvuru belgesi
title: Çalışan
x-i18n:
    generated_at: "2026-07-26T22:43:17Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 0c4749e2abaf4fca00d903114b0661454d67207547fe17711dc5315656e0cd14
    source_path: cli/worker.md
    workflow: 16
---

# `openclaw worker`

`openclaw worker`, bir bulut çalışanı orkestratörünün hazırlanmış bir çalışan ortamında başlatması için kısıtlanmış çalışma zamanı giriş noktasıdır. Çalışanların elle kaydedilmesi için genel amaçlı bir komut değildir.

Gateway, eşleşen OpenClaw paketini kurar ve ana bilgisayar anahtarına sabitlenmiş ters SSH tünelini açar. Çalışan başlatıcısı bu komutu hazırlanmış bir atamayla başlatır. Komut, tünel üzerinden iletilen yerel soket aracılığıyla bağlanır ve ayrılmış `worker` rolüyle kabul edilir.

## Başlatma sözleşmesi

Komut, standart girdiden tam olarak bir adet sınırlandırılmış JSON başlatma zarfı okur. Zarf; yerel soket konumunu, oluşturulmuş çalışan kimlik bilgisini, paket ve protokol kimliğini, sahip dönemini, atanmış tek oturum ile dönüşü ve bu dönüş için yetkilendirilmiş çalışan-yerel araç adlarını taşır. Gateway, devretmeden önce bu nihai araç kümesini geçerli politikadan çözümler; ham yapılandırma ve zamanlanmış sahip kimliği hiçbir zaman çalışan zarfına girmez.
Kimlik bilgisi hiçbir zaman komut satırı bağımsız değişkenleri üzerinden kabul edilmez ve bu sayfa kasıtlı olarak herhangi bir kimlik bilgisi veya elle yazılmış zarf örneği sunmaz.

Zarf geçersizse, kimlik bilgisi reddedilirse, paket ya da protokol özellikleri eşleşmezse veya oturum ile sahip dönemi artık geçerli değilse kabul güvenli biçimde başarısız olur. Eksik, yinelenen veya bilinmeyen araç adları da zarfı geçersiz kılar. Operatörler, bu giriş noktasını doğrudan çağırmak yerine çalışanları bulut çalışanı orkestratörü üzerinden başlatmalıdır.

## Çalışma zamanı sınırı

İşlem, normal gömülü aracı döngüsünü kısıtlanmış bir arka uçla çalıştırır:

- Gateway tarafından verilen dönüş yetkisinde mevcut olduklarında `read`, `write`, `edit`, `apply_patch`, `exec` ve `process` kodlama araçları
  çalışan çalışma alanında yerel olarak çalışır. Boş bir yetki, modeli araçsız çalıştırır.
- Model çağrıları Gateway çıkarım proxy'sini kullanır. Hiçbir yerel model kimlik doğrulama profili yüklenmez.
- Transkript yazımları Gateway transkript kaydetme RPC'sini kullanır.
- Akış ve araç yaşam döngüsü güncellemeleri Gateway canlı olay RPC'sini kullanır.
- Yalnızca atanmış oturum ve dönüş kabul edilir.

Çalışan modu, atanmış oturum araç kümesinin ötesinde kanalları, Gateway HTTP yüzeylerini veya Plugin otomatik başlatmayı başlatmaz. Tek kullanımlık bir durum dizini kullanır ve kalıcı sağlayıcı veya forge kimlik bilgilerine sahip değildir.

Çalışandan çalışana oturum gönderimi bu modda kullanıma sunulmaz. Yerleştirme ve gönderim Gateway'in sahipliğinde kalır: bir operatör mevcut yerel, yönetilen çalışma ağacı oturumunu Gateway üzerinden gönderebilirken bir çalışan işlemi kendisini veya başka bir çalışanı gönderemez.

Hazırlanmış atama; transkript bağlamını, kabul edilmiş temel yaprağı, kaydetme sırasını ve canlı olay imlecini taşır. Tünel yeniden bağlandığında işlem, aynı kimlik bilgisi ve sahip dönemiyle yeniden kabul edilir; kabul edilmiş transkript tabanını korur, onaylanmamış canlı olay kuyruğunu yeniden oynatır ve devam eden bir çıkarım dönüşüne aynı kimlikle yeniden bağlanır. Akış deltaları kaçırılmışsa son çıkarım iletisi belirleyicidir. Öncekinin yerini alan bir sahip dönemi işlemi sınırlar ve temiz biçimde çıkmasına neden olur.

Bir `stale-base-leaf` transkript reddi, geçerli çalıştırmayı hata anında durdurur. Çalışan modu, reddedilen sırayı farklı bir yaprağa karşı yeniden denemez; böylece yinelenen bir kayıt oluşturulmaz ve bu çalıştırmanın bellekteki henüz kaydedilmemiş kuyruğu kaybolur. Yeniden başlatma, Gateway'in yetkili transkriptinden ve kayıt defterinden yeni bir atama oluşturması gereken kilometre taşı 3 yerleştirme sahibinin sorumluluğundadır. Benzer şekilde, Gateway işleminin yeniden başlatılması bekleyen bir çıkarım dönüşünü sağlayıcı hatasıyla sonlandırır; yalnızca bir tünel veya çalışan WebSocket yeniden bağlantısı, aynı işlemdeki etkin bir çıkarım akışına yeniden bağlanabilir.

Kapalı çalışan RPC yüzeyi için [Gateway protokolü](/tr/gateway/protocol#worker-role-and-closed-protocol), mimari ve güvenlik modeli için [Bulut çalışanları planı](/tr/plan/cloud-workers) bölümüne bakın.
