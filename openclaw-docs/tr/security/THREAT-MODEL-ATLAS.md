---
read_when:
    - Güvenlik duruşunu veya tehdit senaryolarını inceleme
    - Güvenlik özellikleri veya denetim yanıtları üzerinde çalışma
summary: MITRE ATLAS çerçevesiyle eşleştirilmiş OpenClaw tehdit modeli
title: Tehdit modeli (MITRE ATLAS)
x-i18n:
    generated_at: "2026-07-26T23:02:01Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: c88ffdef850bd2afaf835baab2555304c914a0be1df6b6b9109e0f55d1448392
    source_path: security/THREAT-MODEL-ATLAS.md
    workflow: 16
---

**Sürüm:** 1.0-draft | **Çerçeve:** [MITRE ATLAS](https://atlas.mitre.org/) (Yapay Zekâ Sistemleri için Düşmanca Tehdit Ortamı) + veri akışı diyagramları

Bu tehdit modeli, OpenClaw yapay zekâ ajan platformuna ve ClawHub skill pazarına yönelik düşmanca tehditleri belgeler. OpenClaw topluluğu tarafından sürdürülen, sürekli gelişen bir belgedir. Yeni tehditlerin nasıl bildirileceği, saldırı zincirlerinin nasıl önerileceği veya azaltma önlemlerinin nasıl sunulacağı hakkında bilgi için [Tehdit modeline katkıda bulunma](/tr/security/CONTRIBUTING-THREAT-MODEL) bölümüne bakın.

**Temel ATLAS kaynakları:** [Teknikler](https://atlas.mitre.org/techniques/) | [Taktikler](https://atlas.mitre.org/tactics/) | [Vaka çalışmaları](https://atlas.mitre.org/studies/) | [ATLAS GitHub](https://github.com/mitre-atlas/atlas-data) | [ATLAS'a katkıda bulunma](https://atlas.mitre.org/resources/contribute)

---

## 1. Kapsam

| Bileşen                | Dahil   | Notlar                                            |
| ---------------------- | ------- | ------------------------------------------------- |
| OpenClaw ajan çalışma zamanı | Evet    | Temel ajan yürütme, araç çağrıları, oturumlar     |
| Gateway                | Evet    | Kimlik doğrulama, yönlendirme, kanal entegrasyonu |
| Kanal entegrasyonları  | Evet    | WhatsApp, Telegram, Discord, Signal, Slack vb.    |
| ClawHub pazarı         | Evet    | Skill yayımlama, moderasyon, dağıtım              |
| MCP sunucuları         | Evet    | Harici araç sağlayıcıları                         |
| Kullanıcı cihazları    | Kısmen  | Mobil uygulamalar, masaüstü istemcileri           |

Kapsam dışı bildirimler ve yanlış pozitif kalıpları (genel internete açık olma, sınır aşma içermeyen yalnızca istem enjeksiyonuna dayalı zincirler, karşılıklı olarak güvenilmeyen operatörlerin tek bir gateway ana makinesini paylaşması ve diğerleri) [`SECURITY.md`](https://github.com/openclaw/openclaw/blob/main/SECURITY.md) içinde sıralanmıştır; güvenlik açığı bildirim kapsamı için güncel ve yetkili kaynak bu dosyadır, bu sayfa değildir.

## 2. Sistem mimarisi

### 2.1 Güven sınırları

```text
┌─────────────────────────────────────────────────────────────────┐
│                    GÜVENİLMEYEN BÖLGE                            │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐              │
│  │  WhatsApp   │  │  Telegram   │  │   Discord   │  ...         │
│  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘              │
│         │                │                │                      │
└─────────┼────────────────┼────────────────┼──────────────────────┘
          │                │                │
          ▼                ▼                ▼
┌─────────────────────────────────────────────────────────────────┐
│                 GÜVEN SINIRI 1: Kanal Erişimi                    │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │                      GATEWAY                              │   │
│  │  • Cihaz eşleştirme (1 sa DM eşleştirme / 5 dk node eşleştirme TTL) │
│  │  • AllowFrom / izin listesi doğrulaması                   │   │
│  │  • Token / parola / Tailscale kimlik doğrulaması          │   │
│  └──────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                 GÜVEN SINIRI 2: Oturum Yalıtımı                  │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │                   AJAN OTURUMLARI                         │   │
│  │  • Oturum anahtarı = agent:channel:peer                   │   │
│  │  • Ajan başına araç politikaları                          │   │
│  │  • Transkript günlükleme                                  │   │
│  └──────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                 GÜVEN SINIRI 3: Araç Yürütme                     │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │                  YÜRÜTME KORUMALI ALANI                   │   │
│  │  • Docker korumalı alanı (varsayılan) veya ana makine (exec onayları) │
│  │  • Node uzaktan yürütme                                   │   │
│  │  • SSRF koruması (DNS sabitleme + IP engelleme)           │   │
│  └──────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                 GÜVEN SINIRI 4: Harici İçerik                    │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │              GETİRİLEN URL'LER / E-POSTALAR / WEBHOOK'LAR │   │
│  │  • Harici içerik sarmalama (rastgele sınırlı XML etiketleri) │
│  │  • Güvenlik bildirimi enjeksiyonu                         │   │
│  └──────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                 GÜVEN SINIRI 5: Tedarik Zinciri                  │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │                      CLAWHUB                              │   │
│  │  • Skill yayımlama (semver, SKILL.md zorunludur)          │   │
│  │  • Statik kalıp + AST'ye yakın moderasyon taraması        │   │
│  │  • LLM tabanlı ajansal risk incelemesi + VirusTotal taraması │
│  │  • GitHub hesap yaşı doğrulaması (14 gün)                 │   │
│  └──────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
```

### 2.2 Veri akışları

| Akış | Kaynak  | Hedef       | Veri                       | Koruma                 |
| ---- | ------- | ----------- | -------------------------- | ---------------------- |
| F1   | Kanal   | Gateway     | Kullanıcı mesajları        | TLS, AllowFrom         |
| F2   | Gateway | Ajan        | Yönlendirilmiş mesajlar    | Oturum yalıtımı        |
| F3   | Ajan    | Araçlar     | Araç çağrıları             | Politika uygulaması    |
| F4   | Ajan    | Harici      | `web_fetch` istekleri | SSRF engelleme         |
| F5   | ClawHub | Ajan        | Skill kodu                 | Moderasyon, tarama     |
| F6   | Ajan    | Kanal       | Yanıtlar                   | Çıktı filtreleme       |

---

## 3. ATLAS taktiğine göre tehdit analizi

### 3.1 Keşif (AML.TA0002)

#### T-RECON-001: Ajan uç noktası keşfi

| Nitelik                 | Değer                                                                |
| ----------------------- | -------------------------------------------------------------------- |
| **ATLAS kimliği**       | AML.T0006 - Etkin Tarama                                             |
| **Açıklama**            | Saldırgan, açık OpenClaw gateway uç noktalarını tarar                |
| **Saldırı vektörü**     | Ağ taraması, Shodan sorguları, DNS numaralandırması                  |
| **Etkilenen bileşenler** | Gateway, açık API uç noktaları                                      |
| **Mevcut azaltma önlemleri** | Tailscale kimlik doğrulama seçeneği, varsayılan olarak geri döngü adresine bağlanma |
| **Kalan risk**          | Orta - genel kullanıma açık gateway'ler keşfedilebilir               |
| **Öneriler**            | Güvenli dağıtımı belgeleyin, keşif uç noktalarına hız sınırlaması ekleyin |

#### T-RECON-002: Kanal entegrasyonu yoklaması

| Nitelik                 | Değer                                                              |
| ----------------------- | ------------------------------------------------------------------ |
| **ATLAS kimliği**       | AML.T0006 - Etkin Tarama                                           |
| **Açıklama**            | Saldırgan, yapay zekâ tarafından yönetilen hesapları belirlemek için mesajlaşma kanallarını yoklar |
| **Saldırı vektörü**     | Test mesajları gönderme, yanıt kalıplarını gözlemleme              |
| **Etkilenen bileşenler** | Tüm kanal entegrasyonları                                         |
| **Mevcut azaltma önlemleri** | Belirli bir önlem yok                                           |
| **Kalan risk**          | Düşük - yalnızca keşfin değeri sınırlıdır                          |
| **Öneriler**            | Yanıt zamanlamasının rastgeleleştirilmesini değerlendirin          |

---

### 3.2 İlk erişim (AML.TA0004)

#### T-ACCESS-001: Eşleştirme kodunun ele geçirilmesi

| Öznitelik               | Değer                                                                                                              |
| ----------------------- | ------------------------------------------------------------------------------------------------------------------ |
| **ATLAS Kimliği**       | AML.T0040 - AI Modeli Çıkarım API'sine Erişim                                                                     |
| **Açıklama**            | Saldırgan, eşleştirme süresi içinde bir eşleştirme kodunu ele geçirir (1 sa. DM/genel eşleştirme, 5 dk. Node eşleştirmesi) |
| **Saldırı vektörü**     | Omuz üzerinden gözetleme, ağ trafiğini dinleme, sosyal mühendislik                                                |
| **Etkilenen bileşenler** | Cihaz eşleştirme sistemi                                                                                           |
| **Mevcut önlemler**     | 1 sa. TTL (DM/genel eşleştirme), 5 dk. TTL (Node eşleştirmesi); kodlar mevcut kanal üzerinden gönderilir          |
| **Kalan risk**          | Orta - eşleştirme süresi istismar edilebilir                                                                      |
| **Öneriler**            | Eşleştirme süresini kısaltın, bir onay adımı ekleyin                                                              |

#### T-ACCESS-002: AllowFrom sahteciliği

| Öznitelik               | Değer                                                                                         |
| ----------------------- | --------------------------------------------------------------------------------------------- |
| **ATLAS Kimliği**       | AML.T0040 - AI Modeli Çıkarım API'sine Erişim                                                 |
| **Açıklama**            | Saldırgan, bir kanalda izin verilen gönderici kimliğinin sahtesini oluşturur                  |
| **Saldırı vektörü**     | Kanala bağlıdır - telefon numarası sahteciliği, kullanıcı kimliğine bürünme                   |
| **Etkilenen bileşenler** | Kanal başına AllowFrom doğrulaması                                                             |
| **Mevcut önlemler**     | Kanala özgü kimlik doğrulama                                                                  |
| **Kalan risk**          | Orta - bazı kanallar sahteciliğe karşı savunmasız kalır                                       |
| **Öneriler**            | Kanala özgü riskleri belgeleyin, mümkün olduğunda kriptografik doğrulama ekleyin              |

#### T-ACCESS-003: Token hırsızlığı

| Öznitelik               | Değer                                                                           |
| ----------------------- | ------------------------------------------------------------------------------- |
| **ATLAS Kimliği**       | AML.T0040 - AI Modeli Çıkarım API'sine Erişim                                   |
| **Açıklama**            | Saldırgan, yapılandırma/kimlik bilgisi dosyalarından kimlik doğrulama tokenlarını çalar |
| **Saldırı vektörü**     | Kötü amaçlı yazılım, yetkisiz cihaz erişimi, yapılandırma yedeğinin açığa çıkması |
| **Etkilenen bileşenler** | Kanal/sağlayıcı kimlik bilgisi depolaması, yapılandırma depolaması               |
| **Mevcut önlemler**     | Dosya izinleri                                                                  |
| **Kalan risk**          | Yüksek - tokenlar diskte düz metin olarak saklanır                              |
| **Öneriler**            | Bekleyen tokenlar için şifreleme uygulayın, token rotasyonu ekleyin             |

---

### 3.3 Yürütme (AML.TA0005)

#### T-EXEC-001: Doğrudan istem enjeksiyonu

| Öznitelik               | Değer                                                                                                                                                    |
| ----------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **ATLAS Kimliği**       | AML.T0051.000 - LLM İstem Enjeksiyonu: Doğrudan                                                                                                         |
| **Açıklama**            | Saldırgan, aracının davranışını manipüle etmek için özel olarak hazırlanmış istemler gönderir                                                           |
| **Saldırı vektörü**     | Düşmanca talimatlar içeren kanal mesajları                                                                                                              |
| **Etkilenen bileşenler** | Aracı LLM'si, tüm giriş yüzeyleri                                                                                                                        |
| **Mevcut önlemler**     | Örüntü algılama, harici içeriği sarmalama; sınır aşımı yoksa güvenlik açığı raporlarının kapsamı dışında değerlendirilir (bkz. `SECURITY.md`)       |
| **Kalan risk**          | Kritik - yalnızca algılama yapılır, engelleme yapılmaz; gelişmiş saldırılar bunu aşabilir                                                               |
| **Öneriler**            | Mevcut algılamaya ek olarak hassas eylemler için çıktı doğrulaması ve kullanıcı onayı                                                                   |

#### T-EXEC-002: Dolaylı istem enjeksiyonu

| Öznitelik               | Değer                                                                                                                              |
| ----------------------- | ---------------------------------------------------------------------------------------------------------------------------------- |
| **ATLAS Kimliği**       | AML.T0051.001 - LLM İstem Enjeksiyonu: Dolaylı                                                                                     |
| **Açıklama**            | Saldırgan, alınan içeriğe kötü amaçlı talimatlar yerleştirir                                                                       |
| **Saldırı vektörü**     | Kötü amaçlı URL'ler, zehirlenmiş e-postalar, güvenliği ihlal edilmiş Webhook'lar                                                   |
| **Etkilenen bileşenler** | `web_fetch`, e-posta alımı, harici veri kaynakları                                                                          |
| **Mevcut önlemler**     | Rastgele sınırlı XML tarzı işaretçilerle içerik sarmalama, benzer glif/özel token normalizasyonu ve güvenlik bildirimi             |
| **Kalan risk**          | Yüksek - LLM yine de sarmalayıcı talimatlarını yok sayabilir                                                                       |
| **Öneriler**            | Sarmalanmış içerik için ayrı yürütme bağlamları                                                                                    |

#### T-EXEC-003: Araç argümanı enjeksiyonu

| Öznitelik               | Değer                                                                  |
| ----------------------- | ---------------------------------------------------------------------- |
| **ATLAS Kimliği**       | AML.T0051.000 - LLM İstem Enjeksiyonu: Doğrudan                        |
| **Açıklama**            | Saldırgan, istem enjeksiyonu yoluyla araç argümanlarını manipüle eder  |
| **Saldırı vektörü**     | Araç parametre değerlerini etkileyen özel hazırlanmış istemler         |
| **Etkilenen bileşenler** | Tüm araç çağrıları                                                      |
| **Mevcut önlemler**     | Tehlikeli komutlar için yürütme onayları                               |
| **Kalan risk**          | Yüksek - kullanıcı muhakemesine dayanır                                |
| **Öneriler**            | Argüman doğrulaması, parametreli araç çağrıları                         |

#### T-EXEC-004: Yürütme onayını atlama

| Öznitelik               | Değer                                                                                                                                                                                                       |
| ----------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **ATLAS Kimliği**       | AML.T0043 - Düşmanca Veri Hazırlama                                                                                                                                                                          |
| **Açıklama**            | Saldırgan, onay izin listesini atlayan komutlar hazırlar                                                                                                                                                     |
| **Saldırı vektörü**     | Komut gizleme, diğer ad istismarı, yol manipülasyonu                                                                                                                                                         |
| **Etkilenen bileşenler** | `src/infra/exec-approvals*.ts`, komut izin listesi                                                                                                                                                                       |
| **Mevcut önlemler**     | İzin listesi + sorma modu ve komut normalizasyonu (yönlendirme sarmalayıcısını açma, satır içi değerlendirme algılama, kabuk zinciri analizi)                                                                |
| **Kalan risk**          | Yüksek - normalizasyon, gizleme yoluyla atlamayı daraltır ancak ortadan kaldırmaz; yürütme yolları arasındaki yalnızca eşlik bulguları güvenlik açığı değil, sağlamlaştırma olarak değerlendirilir (bkz. `SECURITY.md`) |
| **Öneriler**            | Yeni gizleme tekniklerine karşı komut normalizasyonu kapsamını genişletmeye devam edin                                                                                                                       |

---

### 3.4 Kalıcılık (AML.TA0006)

#### T-PERSIST-001: Kötü amaçlı Skills kurulumu

| Öznitelik               | Değer                                                                                                                               |
| ----------------------- | ----------------------------------------------------------------------------------------------------------------------------------- |
| **ATLAS Kimliği**       | AML.T0010.001 - Tedarik Zincirinin Ele Geçirilmesi: AI Yazılımı                                                                     |
| **Açıklama**            | Saldırgan, ClawHub'da kötü amaçlı bir Skills yayımlar                                                                                |
| **Saldırı vektörü**     | Hesap oluşturma, gizli kötü amaçlı kod içeren Skills yayımlama                                                                      |
| **Etkilenen bileşenler** | ClawHub, Skills yükleme, aracı yürütme                                                                                               |
| **Mevcut önlemler**     | GitHub hesap yaşı doğrulaması, statik örüntü/AST'ye yakın tarama, LLM tabanlı aracısal risk incelemesi, VirusTotal taraması          |
| **Kalan risk**          | Yüksek - algılama katmanları mevcuttur ancak Skills yine de aracının ayrıcalıklarıyla ve yürütme korumalı alanı olmadan çalışır     |
| **Öneriler**            | Skills yürütmesi için korumalı alan, genişletilmiş topluluk incelemesi                                                               |

#### T-PERSIST-002: Skills güncelleme zehirlemesi

| Öznitelik               | Değer                                                                                 |
| ----------------------- | ------------------------------------------------------------------------------------- |
| **ATLAS Kimliği**       | AML.T0010.001 - Tedarik Zincirinin Ele Geçirilmesi: AI Yazılımı                       |
| **Açıklama**            | Saldırgan, popüler bir Skills'in güvenliğini ihlal eder ve kötü amaçlı güncelleme gönderir |
| **Saldırı vektörü**     | Hesabın ele geçirilmesi, Skills sahibine yönelik sosyal mühendislik                   |
| **Etkilenen bileşenler** | ClawHub sürümleme, otomatik güncelleme akışları                                       |
| **Mevcut önlemler**     | Sürüm parmak izi oluşturma, yeni sürümlerde moderasyon/taramanın yeniden çalıştırılması |
| **Kalan risk**          | Yüksek - otomatik güncellemeler, inceleme tamamlanmadan kötü amaçlı sürümleri alabilir |
| **Öneriler**            | Güncelleme imzalama, geri alma yeteneği, sürüm sabitleme                              |

#### T-PERSIST-003: Aracı yapılandırmasının kurcalanması

| Öznitelik               | Değer                                                           |
| ----------------------- | --------------------------------------------------------------- |
| **ATLAS Kimliği**       | AML.T0010.002 - Tedarik Zincirinin Ele Geçirilmesi: Veri        |
| **Açıklama**            | Saldırgan, erişimi kalıcı hâle getirmek için aracı yapılandırmasını değiştirir |
| **Saldırı vektörü**     | Yapılandırma dosyasının değiştirilmesi, ayar yerleştirme        |
| **Etkilenen bileşenler** | Aracı yapılandırması, araç politikaları                         |
| **Mevcut azaltımlar**   | Dosya izinleri                                                  |
| **Kalan risk**          | Orta - yerel erişim gerektirir                                  |
| **Öneriler**            | Yapılandırma bütünlüğünün doğrulanması, yapılandırma değişikliklerinin denetim günlüğüne kaydedilmesi |

---

### 3.5 Savunmadan kaçınma (AML.TA0007)

#### T-EVADE-001: Denetleme örüntüsünü atlatma

| Öznitelik               | Değer                                                                                 |
| ----------------------- | ------------------------------------------------------------------------------------- |
| **ATLAS Kimliği**       | AML.T0043 - Çekişmeli Veri Oluşturma                                                   |
| **Açıklama**            | Saldırgan, ClawHub denetleme kontrollerinden kaçınmak için skill içeriği oluşturur     |
| **Saldırı vektörü**     | Unicode benzer karakterleri, kodlama hileleri, dinamik yükleme                         |
| **Etkilenen bileşenler** | ClawHub denetleme/tarama işlem hattı                                                   |
| **Mevcut azaltımlar**   | Statik örüntü kuralları, AST'ye bitişik kod taraması, LLM aracılı risk incelemesi, VirusTotal |
| **Kalan risk**          | Orta - yeni gizleme yöntemleri katmanlı sezgisel kontrolleri yine de aşabilir          |
| **Öneriler**            | Yeni kaçınma yöntemleri bulundukça örüntü/davranış derlemini genişletmeye devam edilmesi |

#### T-EVADE-002: İçerik sarmalayıcısından kaçış

| Öznitelik               | Değer                                                                                                         |
| ----------------------- | ------------------------------------------------------------------------------------------------------------- |
| **ATLAS Kimliği**       | AML.T0043 - Çekişmeli Veri Oluşturma                                                                          |
| **Açıklama**            | Saldırgan, harici içerik sarmalayıcısı bağlamından kaçan içerik oluşturur                                     |
| **Saldırı vektörü**     | Etiket manipülasyonu, bağlam karışıklığı, talimatları geçersiz kılma                                          |
| **Etkilenen bileşenler** | Harici içerik sarmalama                                                                                       |
| **Mevcut azaltımlar**   | Rastgele sınırlı XML tarzı işaretçiler ve güvenlik bildirimi; ayrıca benzer karakter/boşluk varyantlı işaretçi sahteciliği algılama |
| **Kalan risk**          | Orta - yeni kaçış yöntemleri düzenli olarak keşfedilir                                                        |
| **Öneriler**            | Girdi tarafındaki sarmalamaya ek olarak çıktı tarafında doğrulama                                             |

---

### 3.6 Keşif (AML.TA0008)

#### T-DISC-001: Araçları numaralandırma

| Öznitelik               | Değer                                                 |
| ----------------------- | ----------------------------------------------------- |
| **ATLAS Kimliği**       | AML.T0040 - Yapay Zekâ Modeli Çıkarım API'sine Erişim |
| **Açıklama**            | Saldırgan, istemler aracılığıyla kullanılabilir araçları numaralandırır |
| **Saldırı vektörü**     | "Hangi araçlara sahipsin?" tarzı sorgular             |
| **Etkilenen bileşenler** | Aracı araç kayıt defteri                              |
| **Mevcut azaltımlar**   | Belirli bir azaltım yok                                |
| **Kalan risk**          | Düşük - araçlar genellikle belgelenmiştir             |
| **Öneriler**            | Araç görünürlüğü denetimlerinin değerlendirilmesi     |

#### T-DISC-002: Oturum verilerini çıkarma

| Öznitelik               | Değer                                                   |
| ----------------------- | ------------------------------------------------------- |
| **ATLAS Kimliği**       | AML.T0040 - Yapay Zekâ Modeli Çıkarım API'sine Erişim   |
| **Açıklama**            | Saldırgan, oturum bağlamından hassas verileri çıkarır   |
| **Saldırı vektörü**     | "Ne hakkında konuştuk?" sorguları, bağlam yoklaması     |
| **Etkilenen bileşenler** | Oturum dökümleri, bağlam penceresi                     |
| **Mevcut azaltımlar**   | Gönderen başına oturum yalıtımı (`agent:channel:peer` anahtarı) |
| **Kalan risk**          | Orta - oturum içi verilere tasarım gereği erişilebilir  |
| **Öneriler**            | Bağlamdaki hassas verilerin karartılması                |

---

### 3.7 Toplama ve dışarı veri sızdırma (AML.TA0009, AML.TA0010)

#### T-EXFIL-001: web_fetch aracılığıyla veri hırsızlığı

| Öznitelik               | Değer                                                                            |
| ----------------------- | -------------------------------------------------------------------------------- |
| **ATLAS Kimliği**       | AML.T0009 - Toplama                                                              |
| **Açıklama**            | Saldırgan, aracıya verileri harici bir URL'ye göndermesi talimatını vererek verileri dışarı sızdırır |
| **Saldırı vektörü**     | Aracının saldırgan sunucusuna POST ile veri göndermesine neden olan istem yerleştirme |
| **Etkilenen bileşenler** | `web_fetch` aracı                                                        |
| **Mevcut azaltımlar**   | Dahili/özel ağlar için SSRF engelleme (DNS sabitleme + IP engelleme)             |
| **Kalan risk**          | Yüksek - rastgele harici URL'lere izin verilmeye devam edilir                    |
| **Öneriler**            | URL izin listesi, veri sınıflandırması farkındalığı                              |

#### T-EXFIL-002: Yetkisiz mesaj gönderme

| Öznitelik               | Değer                                                                |
| ----------------------- | -------------------------------------------------------------------- |
| **ATLAS Kimliği**       | AML.T0009 - Toplama                                                   |
| **Açıklama**            | Saldırgan, aracının hassas veriler içeren mesajlar göndermesine neden olur |
| **Saldırı vektörü**     | Aracının saldırgana mesaj göndermesine neden olan istem yerleştirme   |
| **Etkilenen bileşenler** | Mesaj aracı, kanal entegrasyonları                                   |
| **Mevcut azaltımlar**   | Giden mesajlaşma geçidi                                               |
| **Kalan risk**          | Orta - geçit atlatılabilir                                           |
| **Öneriler**            | Yeni alıcılar için açık onay                                         |

#### T-EXFIL-003: Kimlik bilgilerini toplama

| Öznitelik               | Değer                                                                                                                                                   |
| ----------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **ATLAS Kimliği**       | AML.T0009 - Toplama                                                                                                                                     |
| **Açıklama**            | Kötü amaçlı skill, aracı bağlamındaki kimlik bilgilerini toplar                                                                                         |
| **Saldırı vektörü**     | Skill kodu ortam değişkenlerini ve yapılandırma dosyalarını okur                                                                                        |
| **Etkilenen bileşenler** | Skill yürütme ortamı                                                                                                                                   |
| **Mevcut azaltımlar**   | ClawHub kimlik bilgisi örüntüsü taraması (sabit kodlanmış gizli bilgiler, ağ gönderimleriyle eşleştirilen kimlik bilgisi ortam erişimi); çalışma zamanında skill'ler için yürütme korumalı alanı yoktur |
| **Kalan risk**          | Kritik - skill'ler aracı ayrıcalıklarıyla çalışır                                                                                                       |
| **Öneriler**            | Skill yürütmesini korumalı alana alma, kimlik bilgilerini yalıtma                                                                                       |

---

### 3.8 Etki (AML.TA0011)

#### T-IMPACT-001: Yetkisiz komut yürütme

| Öznitelik               | Değer                                                                                                |
| ----------------------- | ---------------------------------------------------------------------------------------------------- |
| **ATLAS Kimliği**       | AML.T0031 - Yapay Zekâ Modeli Bütünlüğünü Aşındırma                                                  |
| **Açıklama**            | Saldırgan, kullanıcı sisteminde rastgele komutlar yürütür                                            |
| **Saldırı vektörü**     | Yürütme onayını atlatmayla birleştirilen istem yerleştirme                                           |
| **Etkilenen bileşenler** | Bash aracı, komut yürütme                                                                           |
| **Mevcut azaltımlar**   | Yürütme onayları, Docker korumalı alan seçeneği (varsayılan çalışma zamanı arka ucu)                 |
| **Kalan risk**          | Kritik - korumalı alan devre dışı bırakıldığında ana makinede yürütme mümkündür                       |
| **Öneriler**            | Onay kullanıcı deneyiminin iyileştirilmesi; korumalı alanın kapalı olduğu dağıtımlar, bu şekilde belgelenmiş bilinçli bir operatör tercihi olmaya devam eder |

#### T-IMPACT-002: Kaynakların tükenmesi (DoS)

| Öznitelik               | Değer                                              |
| ----------------------- | -------------------------------------------------- |
| **ATLAS Kimliği**       | AML.T0031 - Yapay Zekâ Modeli Bütünlüğünü Aşındırma |
| **Açıklama**            | Saldırgan, API kredilerini veya işlem kaynaklarını tüketir |
| **Saldırı vektörü**     | Otomatik mesaj yağmuru, pahalı araç çağrıları      |
| **Etkilenen bileşenler** | Gateway, aracı oturumları, API sağlayıcısı         |
| **Mevcut azaltımlar**   | Yok                                                |
| **Kalan risk**          | Yüksek - gönderen başına hız sınırlaması yok       |
| **Öneriler**            | Gönderen başına hız sınırları, maliyet bütçeleri   |

#### T-IMPACT-003: İtibar zedelenmesi

| Öznitelik               | Değer                                                       |
| ----------------------- | ----------------------------------------------------------- |
| **ATLAS Kimliği**       | AML.T0031 - Yapay Zekâ Modeli Bütünlüğünü Aşındırma         |
| **Açıklama**            | Saldırgan, aracının zararlı/saldırgan içerik göndermesine neden olur |
| **Saldırı vektörü**     | Uygunsuz yanıtlara neden olan istem yerleştirme              |
| **Etkilenen bileşenler** | Çıktı oluşturma, kanal mesajlaşması                          |
| **Mevcut azaltımlar**   | LLM sağlayıcısının içerik politikaları                       |
| **Kalan risk**          | Orta - sağlayıcı filtreleri kusursuz değildir                |
| **Öneriler**            | Çıktı filtreleme katmanı, kullanıcı denetimleri              |

---

## 4. ClawHub tedarik zinciri analizi

### 4.1 Mevcut güvenlik denetimleri

| Denetim                        | Uygulama                                                                        | Etkinlik                                                       |
| ------------------------------ | ------------------------------------------------------------------------------------- | ------------------------------------------------------------------- |
| GitHub hesap yaşı             | `requireGitHubAccountAge()` (en az 14 gün)                                          | Orta - yeni saldırganlar için çıtayı yükseltir                           |
| Yol temizleme              | `sanitizePath()`                                                                      | Yüksek - yol geçişini önler                                      |
| Dosya türü doğrulaması           | `isTextFile()`                                                                        | Orta - yalnızca metin dosyaları taranır, ancak yine de istismar edilebilir             |
| Boyut sınırları                    | Toplam paket 50MB (`MAX_PUBLISH_TOTAL_BYTES`)                                         | Yüksek - kaynak tükenmesini önler                                 |
| Zorunlu SKILL.md              | Yayımlama sırasında zorunlu benioku                                                           | Düşük güvenlik değeri - yalnızca bilgilendirme amaçlı                             |
| Statik + AST'ye komşu tarama | Çalıştırma, veri sızdırma, kimlik bilgisi toplama, gizleme ve daha fazlasını kapsayan örüntü motoru | Orta-Yüksek - bilinen birçok kötüye kullanım örüntüsünü kapsar, yine de örüntü tabanlıdır |
| LLM tabanlı aracılı risk incelemesi  | Yayımlama sırasında güvenlik istemiyle yönlendirilen karar                                             | Orta-Yüksek - statik örüntülerin kaçırdığı davranışları yakalar                 |
| VirusTotal taraması            | Skill ve paket sürümü yayımlama/yeniden tarama akışlarına bağlıdır, operatör API anahtarına göre etkinleştirilir    | Etkinleştirildiğinde yüksek - statik motor algılaması                         |
| Moderasyon durumu              | `moderationStatus` alanı                                                              | Orta - manuel inceleme mümkündür                                     |

### 4.2 Moderasyon sınırlamaları

ClawHub'ın statik taraması, yalnızca kısa ad/meta veri/frontmatter yerine doğrudan Skill kodu içeriğini inceler; tehlikeli çalıştırma çağrılarını, dinamik kod yürütmeyi, kimlik bilgisi toplamayı, veri sızdırma örüntülerini, gizlenmiş yükleri ve daha fazlasını kapsar. Bilinen eksiklikler:

- Örüntü tabanlı algılama, yeterince yeni gizleme yöntemleriyle yine de atlatılabilir.
- LLM tabanlı inceleme ve VirusTotal taraması, operatör tarafındaki API anahtarlarının/yapılandırmanın etkinleştirilmesine bağlıdır.
- Kurulduktan sonra hiçbir çalışma zamanı yürütme sandbox'ı bir Skill'i aracının kendi ayrıcalıklarından yalıtmaz.

### 4.3 Rozetler

Skills ve paketler, moderatör tarafından atanan şu rozetleri taşır: `highlighted`, `official`, `deprecated`, `redactionApproved` (yalnızca Skills). Topluluk bildirimi (`skillReports`) ve denetim günlüğü (`auditLogs`) moderasyon iş akışlarını destekler.

---

## 5. Risk matrisi

### 5.1 Olasılık ve etki

| Tehdit kimliği     | Olasılık | Etki   | Risk düzeyi   | Öncelik |
| ------------- | ---------- | -------- | ------------ | -------- |
| T-EXEC-001    | Yüksek       | Kritik | **Kritik** | P0       |
| T-PERSIST-001 | Yüksek       | Kritik | **Kritik** | P0       |
| T-EXFIL-003   | Orta     | Kritik | **Kritik** | P0       |
| T-IMPACT-001  | Orta     | Kritik | **Yüksek**     | P1       |
| T-EXEC-002    | Yüksek       | Yüksek     | **Yüksek**     | P1       |
| T-EXEC-004    | Orta     | Yüksek     | **Yüksek**     | P1       |
| T-ACCESS-003  | Orta     | Yüksek     | **Yüksek**     | P1       |
| T-EXFIL-001   | Orta     | Yüksek     | **Yüksek**     | P1       |
| T-IMPACT-002  | Yüksek       | Orta   | **Yüksek**     | P1       |
| T-EVADE-001   | Yüksek       | Orta   | **Orta**   | P2       |
| T-ACCESS-001  | Düşük        | Yüksek     | **Orta**   | P2       |
| T-ACCESS-002  | Düşük        | Yüksek     | **Orta**   | P2       |
| T-PERSIST-002 | Düşük        | Yüksek     | **Orta**   | P2       |

### 5.2 Kritik yol saldırı zincirleri

**Zincir 1: Skill tabanlı veri hırsızlığı**

```text
T-PERSIST-001 → T-EVADE-001 → T-EXFIL-003
(Kötü amaçlı Skill yayımla) → (Moderasyonu atlat) → (Kimlik bilgilerini topla)
```

**Zincir 2: İstem enjeksiyonundan RCE'ye**

```text
T-EXEC-001 → T-EXEC-004 → T-IMPACT-001
(İstem enjekte et) → (Çalıştırma onayını atlat) → (Komutları yürüt)
```

**Zincir 3: Getirilen içerik üzerinden dolaylı enjeksiyon**

```text
T-EXEC-002 → T-EXFIL-001 → Haricî veri sızdırma
(URL içeriğini zehirle) → (Aracı içeriği getirir ve talimatları izler) → (Veriler saldırgana gönderilir)
```

---

## 6. Önerilerin özeti

### 6.1 Acil (P0)

| Kimlik    | Öneri                              | Ele aldığı tehditler                  |
| ----- | ------------------------------------------- | -------------------------- |
| R-002 | Skill yürütme sandbox'ı uygulayın        | T-PERSIST-001, T-EXFIL-003 |
| R-003 | Hassas eylemler için çıktı doğrulaması ekleyin | T-EXEC-001, T-EXEC-002     |

### 6.2 Kısa vadeli (P1)

| Kimlik    | Öneri                                                        | Ele aldığı tehditler    |
| ----- | --------------------------------------------------------------------- | ------------ |
| R-004 | Gönderici başına hız sınırlaması uygulayın                                    | T-IMPACT-002 |
| R-005 | Bekleyen token'lar için şifreleme ekleyin                                          | T-ACCESS-003 |
| R-006 | Çalıştırma onayı kullanıcı deneyimini iyileştirin ve komut normalleştirmeyi genişletmeye devam edin | T-EXEC-004   |
| R-007 | `web_fetch` için URL izin listesi uygulayın                            | T-EXFIL-001  |

### 6.3 Orta vadeli (P2)

| Kimlik    | Öneri                                        | Ele aldığı tehditler     |
| ----- | ----------------------------------------------------- | ------------- |
| R-008 | Mümkün olduğunda kriptografik kanal doğrulaması ekleyin | T-ACCESS-002  |
| R-009 | Yapılandırma bütünlüğü doğrulaması uygulayın               | T-PERSIST-003 |
| R-010 | Güncelleme imzalama ve sürüm sabitleme ekleyin                | T-PERSIST-002 |

---

## 7. Ekler

### 7.1 ATLAS teknik eşlemesi

| ATLAS kimliği      | Teknik adı                 | OpenClaw tehditleri                                                 |
| ------------- | ------------------------------ | ---------------------------------------------------------------- |
| AML.T0006     | Etkin Tarama                | T-RECON-001, T-RECON-002                                         |
| AML.T0009     | Toplama                     | T-EXFIL-001, T-EXFIL-002, T-EXFIL-003                            |
| AML.T0010.001 | Tedarik Zinciri: Yapay Zekâ Yazılımı      | T-PERSIST-001, T-PERSIST-002                                     |
| AML.T0010.002 | Tedarik Zinciri: Veri             | T-PERSIST-003                                                    |
| AML.T0031     | Yapay Zekâ Modeli Bütünlüğünü Aşındırma       | T-IMPACT-001, T-IMPACT-002, T-IMPACT-003                         |
| AML.T0040     | Yapay Zekâ Modeli Çıkarım API'sine Erişim  | T-ACCESS-001, T-ACCESS-002, T-ACCESS-003, T-DISC-001, T-DISC-002 |
| AML.T0043     | Hasımlı Veri Oluşturma         | T-EXEC-004, T-EVADE-001, T-EVADE-002                             |
| AML.T0051.000 | LLM İstem Enjeksiyonu: Doğrudan   | T-EXEC-001, T-EXEC-003                                           |
| AML.T0051.001 | LLM İstem Enjeksiyonu: Dolaylı | T-EXEC-002                                                       |

### 7.2 Temel güvenlik dosyaları

| Yol                                | Amaç                        | Risk düzeyi   |
| ----------------------------------- | ------------------------------ | ------------ |
| `src/infra/exec-approvals.ts`       | Komut onayı mantığı         | **Kritik** |
| `src/gateway/auth.ts`               | Gateway kimlik doğrulaması         | **Kritik** |
| `src/infra/net/ssrf.ts`             | SSRF koruması                | **Kritik** |
| `src/security/external-content.ts`  | İstem enjeksiyonu azaltma    | **Kritik** |
| `src/agents/sandbox/tool-policy.ts` | Sandbox aracı izin/verme politikası | **Kritik** |
| `src/routing/resolve-route.ts`      | Oturum yalıtımı / yönlendirme    | **Orta**   |

### 7.3 Terimler sözlüğü

| Terim                 | Tanım                                                |
| -------------------- | --------------------------------------------------------- |
| **ATLAS**            | MITRE'ın Yapay Zekâ Sistemleri için Hasımlı Tehdit Ortamı       |
| **ClawHub**          | OpenClaw'ın Skill pazaryeri                              |
| **Gateway**          | OpenClaw'ın mesaj yönlendirme ve kimlik doğrulama katmanı       |
| **MCP**              | Model Bağlam Protokolü - araç sağlayıcı arayüzü          |
| **İstem enjeksiyonu** | Kötü amaçlı talimatların girdiye gömüldüğü saldırı |
| **Skill**            | OpenClaw aracıları için indirilebilir uzantı                |
| **SSRF**             | Sunucu Taraflı İstek Sahteciliği                               |

---

_Bu tehdit modeli yaşayan bir belgedir. Güvenlik sorunlarını `security@openclaw.ai` adresine bildirin veya [Güven sayfasına](https://trust.openclaw.ai) bakın._

## İlgili

- [Tehdit modeline katkıda bulunma](/tr/security/CONTRIBUTING-THREAT-MODEL)
- [Olay müdahalesi](/tr/security/incident-response)
- [Ağ proxy'si](/tr/security/network-proxy)
- [Biçimsel doğrulama](/tr/security/formal-verification)
