---
read_when:
    - Politika Pluginini yüklüyor, yapılandırıyor veya denetliyorsunuz
summary: Çalışma alanı uygunluğu için politika destekli doctor denetimleri ekler.
title: Politika plugini
x-i18n:
    generated_at: "2026-07-26T23:33:30Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 440f2f46e4149fdd5e65bf0140d4981c6d840e8e8c8a85d05eeb23a0839a61ac
    source_path: plugins/reference/policy.md
    workflow: 16
---

# Policy plugin

Çalışma alanı uyumluluğu için policy destekli doctor kontrolleri ekler.

## Dağıtım

- Paket: `@openclaw/policy`
- Kurulum yolu: OpenClaw'a dahildir

## Yüzey

plugin

<!-- openclaw-plugin-reference:manual-start -->

## Davranış

Policy plugin, policy ile yönetilen OpenClaw ayarları ve yönetişim altındaki çalışma alanı bildirimleri için doctor durum kontrolleri sağlar. Policy şu anda kanal uyumluluğunu, yönetişim altındaki araç meta verilerini, MCP sunucusu güvenlik duruşunu, model sağlayıcısı güvenlik duruşunu, özel ağ erişimi güvenlik duruşunu, Gateway dışa açılma güvenlik duruşunu, agent çalışma alanı/araç güvenlik duruşunu, yapılandırılmış genel/agent başına araç güvenlik duruşunu, yapılandırılmış sandbox çalışma zamanı güvenlik duruşunu, giriş/kanal erişimi güvenlik duruşunu, veri işleme güvenlik duruşunu ve OpenClaw yapılandırma sırrı sağlayıcısı/kimlik doğrulama profili güvenlik duruşunu kapsar.

Policy, yazılmış gereksinimleri `policy.jsonc` içinde saklar, mevcut OpenClaw ayarlarını ve çalışma alanı bildirimlerini kanıt olarak gözlemler ve sapmaları `openclaw policy check` ile `openclaw doctor --lint` üzerinden bildirir. Sorunsuz bir policy kontrolü; operatörlerin denetim için kaydedebileceği policy, kanıt, bulgular ve doğrulama karmaları üretir.

`openclaw policy compare --baseline <file>`, bir policy dosyasını başka bir policy dosyasıyla karşılaştırır. Yalnızca yapılandırma düzeyinde uyumluluğu denetler: kontrol edilen policy'nin yazılmış temel çizgideki gereksinimleri eksik bırakmadığını veya bunlardan daha zayıf olmadığını doğrulamak için policy kuralı meta verilerini kullanır ve çalışma zamanı durumunu, kimlik bilgilerini ya da sır değerlerini incelemez.

Araç güvenlik duruşu kuralları; onaylı profilleri, yalnızca çalışma alanında kullanılabilen dosya sistemi araçlarını, sınırlandırılmış exec güvenliği/izin isteme/ana makine ayarlarını, devre dışı bırakılmış yükseltilmiş modu, tam `alsoAllow` girdilerini ve gerekli araç reddetme girdilerini zorunlu kılabilir. Kanıt kayıtları, etkili araç güvenlik duruşunu genişletebildikleri için ek `alsoAllow` girdilerini içerir. Bu kontroller yalnızca yapılandırma uyumluluğunu gözlemler; çalışma zamanı onay durumunu okumaz veya çalışma zamanı yaptırımı eklemez.

Sandbox güvenlik duruşu kuralları; onaylı sandbox modlarını/arka uçlarını zorunlu kılabilir, ana makine konteyner ağını ve konteyner ad alanlarına katılmayı reddedebilir, salt okunur konteyner bağlamalarını zorunlu kılabilir, konteyner çalışma zamanı soketi bağlamalarını ve sınırlandırılmamış konteyner profillerini reddedebilir ve sandbox tarayıcısı CDP kaynak aralıklarını zorunlu kılabilir.
Bu kontroller yalnızca yapılandırma uyumluluğunu gözlemler; çalışma zamanı onay durumunu okumaz, çalışan konteynerleri incelemez veya çalışma zamanı yaptırımı eklemez.

Veri işleme kuralları; hassas günlük kaydı bilgilerinin gizlenmesini ve oturum saklama bakımını zorunlu kılabilir, telemetri içeriğinin yakalanmasını ve oturum dökümü belleğinin dizine eklenmesini reddedebilir. Bu kontroller yalnızca yapılandırma uyumluluğunu gözlemler; ham günlükleri, telemetri dışa aktarımlarını, dökümleri, bellek dosyalarını, sırları veya kişisel verileri incelemez.

`scopes.<scopeName>` altındaki adlandırılmış policy kapsamları, listeledikleri seçici için normal policy bölümlerine daha katı kurallar ekleyebilir. `agentIds`; `tools`, `agents.workspace`, `sandbox` ve `dataHandling.memory` değerlerini destekler; `channelIds` ise `ingress.channels` değerini destekler.
`agents.entries.*` içinde açıkça listelenmeyen çalışma zamanı agent kimlikleri, hiçbir kanıt olmadan sessizce başarılı sayılmak yerine devralınan genel/varsayılan güvenlik duruşuna göre kontrol edilir. `policy.jsonc` içinde bulunan her kapsam, seçicisi için geçerli ve uygulanabilir olmalıdır. Katman kuralları ek iddialardır; bu nedenle üst düzey policy'yi zayıflatmaz ve gözlemlenen aynı yapılandırma her iki kapsamı da ihlal ettiğinde kendi bulgularını üretebilir.

<!-- openclaw-plugin-reference:manual-end -->

## İlgili belgeler

- [policy](/tr/cli/policy)
