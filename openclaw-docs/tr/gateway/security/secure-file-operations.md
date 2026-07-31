---
read_when:
    - Dosya erişimini, arşiv ayıklamayı, çalışma alanı depolamasını veya plugin dosya sistemi yardımcılarını değiştirme
summary: OpenClaw yerel dosya erişimini nasıl güvenli bir şekilde yönetir ve isteğe bağlı fs-safe Python yardımcısı neden varsayılan olarak kapalıdır
title: Güvenli dosya işlemleri
x-i18n:
    generated_at: "2026-07-26T23:22:05Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 5c8edf36ddbb8c8bc1edc52ecdf481affe5395d1779c679a40439167dfe70299
    source_path: gateway/security/secure-file-operations.md
    workflow: 16
---

OpenClaw, güvenlik açısından hassas yerel dosya işlemleri için [`@openclaw/fs-safe`](https://github.com/openclaw/fs-safe) kullanır: kökle sınırlandırılmış okuma/yazma, atomik değiştirme, arşiv çıkarma, geçici çalışma alanları, JSON durumu ve gizli dosyaların işlenmesi.

Bu, güvenilmeyen yol adlarını alan güvenilir OpenClaw kodu için bir **kütüphane korumasıdır**; bir korumalı alan değildir. Gerçek etki alanını yine ana makine dosya sistemi izinleri, işletim sistemi kullanıcıları, kapsayıcılar ve agent/araç politikası belirler.

## Varsayılan: Python yardımcısı yok

OpenClaw, fs-safe POSIX Python yardımcısını varsayılan olarak **kapalı** duruma ayarlar:

- bir operatör etkinleştirmeyi seçmediği sürece gateway kalıcı bir Python yardımcı işlemi başlatmamalıdır;
- çoğu kurulum, üst dizin değişikliklerine yönelik ek sağlamlaştırmaya ihtiyaç duymaz;
- Python'ın devre dışı bırakılması, masaüstü, Docker, CI ve paketlenmiş uygulama ortamlarında çalışma zamanı davranışını öngörülebilir tutar.

OpenClaw yalnızca _varsayılanı_ değiştirir. Açık bir ayar her zaman önceliklidir:

```bash
# Varsayılan OpenClaw davranışı: Yalnızca Node kullanan fs-safe geri dönüşleri.
OPENCLAW_FS_SAFE_PYTHON_MODE=off

# Kullanılabiliyorsa yardımcıyı etkinleştir, kullanılamıyorsa geri dönüşü kullan.
OPENCLAW_FS_SAFE_PYTHON_MODE=auto

# Yardımcı başlatılamazsa güvenli biçimde başarısız ol.
OPENCLAW_FS_SAFE_PYTHON_MODE=require

# İsteğe bağlı açık yorumlayıcı yolu.
OPENCLAW_FS_SAFE_PYTHON=/usr/bin/python3
```

Genel fs-safe ortam değişkeni adları da çalışır: `FS_SAFE_PYTHON_MODE` ve `FS_SAFE_PYTHON`.

Yardımcı, güvenlik yaklaşımınızın bir parçasıysa `auto` değil, `require` kullanın; yardımcı başlatılamazsa `auto` sessizce yalnızca Node kullanan davranışa geri döner.

## Python olmadan korunanlar

Yardımcı kapalıyken OpenClaw yine fs-safe'in yalnızca Node kullanan korumalarından yararlanır:

- göreli yol kaçışlarını (`..`), mutlak yolları ve yalnızca yalın adlara izin verilen yerlerde yol ayırıcılarını reddeder;
- işlemleri geçici `path.resolve(...).startsWith(...)` denetimleri yerine güvenilir bir kök tanıtıcısı üzerinden çözümler;
- ilgili politikayı gerektiren API'lerde sembolik bağlantı ve sabit bağlantı kalıplarını reddeder;
- API'nin dosya içeriği döndürdüğü veya tükettiği durumlarda dosyaları kimlik denetimleriyle açar;
- durum/yapılandırma dosyalarını aynı dizinde geçici dosya oluşturup atomik olarak yeniden adlandırarak yazar;
- okuma ve arşiv çıkarma işlemleri için bayt sınırlarını uygular;
- API'nin gerektirdiği durumlarda gizli bilgiler ve durum dosyaları için özel dosya modları uygular.

Bu, OpenClaw'ın normal tehdit modelini kapsar: tek bir güvenilir operatör sınırı içinde güvenilmeyen model/Plugin/kanal yol girdilerini işleyen güvenilir gateway kodu.

## Python'ın ekledikleri

POSIX'te isteğe bağlı yardımcı, tek bir kalıcı Python işlemini çalışır durumda tutar ve üst dizin değişiklikleri için dosya tanımlayıcısına göreli dosya sistemi işlemlerini kullanır: yeniden adlandırma, kaldırma, dizin oluşturma, durum bilgisi alma/listeleme ve bazı yazma yolları.

Bu, güvenilmeyen yerel işlemlerin OpenClaw'ın üzerinde işlem yaptığı dizinleri değiştirebildiği ana makinelerde derinlemesine savunma sağlayarak başka bir işlemin doğrulama ile değişiklik arasında bir üst dizini değiştirdiği aynı UID'ye ait yarış durumu aralıklarını daraltır.

Dağıtımınızda bu risk varsa ve Python'ın mevcut olacağı garanti ediliyorsa şunu ayarlayın:

```bash
OPENCLAW_FS_SAFE_PYTHON_MODE=require
```

## Plugin ve çekirdek rehberi

- Bir yol mesajdan, model çıktısından, yapılandırmadan veya Plugin girdisinden geliyorsa Plugin'e yönelik dosya erişimi ham `fs` yerine `openclaw/plugin-sdk/*` yardımcıları üzerinden gerçekleştirilmelidir.
- OpenClaw'ın işlem politikasının tutarlı biçimde uygulanması için çekirdek kodu `src/infra/*` altındaki fs-safe sarmalayıcılarını kullanmalıdır.
- Arşiv çıkarma işlemi; açık boyut, girdi sayısı, bağlantı ve hedef sınırlarıyla fs-safe arşiv yardımcılarını kullanmalıdır.
- Gizli bilgiler, OpenClaw gizli bilgi yardımcılarını veya fs-safe gizli/özel durum yardımcılarını kullanmalıdır; `fs.writeFile` çevresinde özel mod denetimleri yazmayın.
- Güvenilmeyen yerel kullanıcıları yalıtmak için yalnızca fs-safe'e güvenmeyin. Ayrı işletim sistemi kullanıcıları/ana makineleri altında ayrı gateway'ler çalıştırın veya korumalı alan kullanın.

İlgili konular: [Güvenlik](/tr/gateway/security), [Korumalı alan](/tr/gateway/sandboxing), [Exec onayları](/tr/tools/exec-approvals), [Gizli bilgiler](/tr/gateway/secrets).
