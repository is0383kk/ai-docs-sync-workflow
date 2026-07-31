---
read_when:
    - Ağ mimarisi + güvenliğe genel bakışa ihtiyacınız var
    - Yerel erişim ile tailnet erişimi veya eşleştirme sorunlarını ayıklıyorsunuz
    - Ağ belgelerinin standart listesini istiyorsunuz
summary: 'Ağ merkezi: Gateway yüzeyleri, eşleştirme, keşif ve güvenlik'
title: Ağ
x-i18n:
    generated_at: "2026-07-27T00:03:57Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 9751bb0fe71009455b243b109ef7ef4eda08d58f940f7dcef305800a5ed89586
    source_path: network.md
    workflow: 16
---

Bu merkez, OpenClaw'un localhost, LAN ve tailnet üzerindeki cihazlara nasıl bağlandığına, cihazları nasıl eşleştirdiğine ve güvenliğini nasıl sağladığına ilişkin temel belgelere bağlantılar sunar.

## Temel model

Çoğu işlem, kanal bağlantılarını ve WebSocket kontrol düzlemini yöneten, uzun süre çalışan tek bir işlem olan Gateway (`openclaw gateway`) üzerinden gerçekleştirilir.

- **Önce geri döngü**: Gateway WS varsayılan olarak `ws://127.0.0.1:18789` kullanır.
  Geri döngü dışı bağlamalar, geçerli bir Gateway kimlik doğrulama yolu olmadan başlatılamaz:
  paylaşılan gizli anahtar belirteci/parola kimlik doğrulaması veya doğru yapılandırılmış geri döngü dışı
  `trusted-proxy` dağıtımı.
- **Ana bilgisayar başına bir Gateway** önerilir. Yalıtım için birbirinden yalıtılmış profiller ve bağlantı noktalarıyla birden fazla Gateway çalıştırın ([Birden Fazla Gateway](/tr/gateway/multiple-gateways)).
- **Canvas ana bilgisayarı**, Gateway ile aynı bağlantı noktasından sunulur (`/__openclaw__/canvas/`, `/__openclaw__/a2ui/`) ve geri döngü dışına bağlandığında Gateway kimlik doğrulamasıyla korunur.
- **Uzaktan erişim** genellikle bir SSH tüneli veya Tailscale VPN üzerinden sağlanır ([Uzaktan Erişim](/tr/gateway/remote)).

Temel başvuru kaynakları:

- [Gateway mimarisi](/tr/concepts/architecture)
- [Gateway protokolü](/tr/gateway/protocol)
- [Gateway çalışma kılavuzu](/tr/gateway)
- [Web yüzeyleri + bağlama modları](/tr/web)

## Eşleştirme + kimlik

- [Eşleştirmeye genel bakış (DM + Node'lar)](/tr/channels/pairing)
- [Gateway tarafından yönetilen Node eşleştirmesi](/tr/gateway/pairing)
- [Cihazlar CLI'sı (eşleştirme + belirteç döndürme)](/tr/cli/devices)
- [Eşleştirme CLI'sı (DM onayları)](/tr/cli/pairing)

Yerel güven:

- Doğrudan yerel geri döngü bağlantıları (iletilen/proxy üstbilgileri olmadan), aynı ana bilgisayardaki kullanıcı deneyimini sorunsuz tutmak için
  eşleştirme amacıyla otomatik olarak onaylanabilir.
- OpenClaw ayrıca güvenilir paylaşılan gizli anahtar yardımcı akışları için
  kapsamı dar bir arka uç/kapsayıcı içi kendi kendine bağlantı yoluna sahiptir.
- Aynı ana bilgisayardaki tailnet bağlamaları da dahil olmak üzere tailnet ve LAN istemcileri için
  yine de açık eşleştirme onayı gerekir.

## Keşif + aktarımlar

- [Keşif ve aktarımlar](/tr/gateway/discovery)
- [Bonjour / mDNS](/tr/gateway/bonjour)
- [Uzaktan erişim (SSH)](/tr/gateway/remote)
- [Tailscale](/tr/gateway/tailscale)

## Node'lar + aktarımlar

- [Node'lara genel bakış](/tr/nodes)
- [Köprü protokolü (eski Node'lar, tarihsel)](/tr/gateway/bridge-protocol)
- [Node çalışma kılavuzu: iOS](/tr/platforms/ios)
- [Node çalışma kılavuzu: Android](/tr/platforms/android)

## Güvenlik

- [Güvenliğe genel bakış](/tr/gateway/security)
- [Gateway yapılandırma başvurusu](/tr/gateway/configuration)
- [Sorun giderme](/tr/gateway/troubleshooting)
- [Doctor](/tr/gateway/doctor)

## İlgili

- [Gateway çalışma kılavuzu](/tr/gateway)
- [Uzaktan erişim](/tr/gateway/remote)
