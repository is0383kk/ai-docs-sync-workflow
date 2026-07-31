---
read_when:
    - Gateway'i LAN, tailnet, Tailscale Serve, Funnel veya ters proxy üzerinden kullanıma açma
    - Gerçek mesajlaşma kullanıcılarına izin vermeden önce bir dağıtımı inceleme
    - Riskli bir uzaktan erişim veya DM yapılandırmasını geri alma
sidebarTitle: Exposure runbook
summary: Bir OpenClaw Gateway'i geri döngü arabiriminin ötesine açmadan önce ön kontrol ve geri alma denetim listesi
title: Gateway dışa açma çalışma kılavuzu
x-i18n:
    generated_at: "2026-07-26T22:47:31Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: fb8e66af57e804325afc91281122b822183337177c734efe065c5fc18b175e72
    source_path: gateway/security/exposure-runbook.md
    workflow: 16
---

<Warning>
Gateway'i yalnızca ona kimlerin erişebileceğini, bunların nasıl
kimlik doğrulamasından geçirildiğini, hangi ajanları tetikleyebileceklerini ve bu ajanların hangi araçları
kullanabileceğini açıklayabildiğinizde dış erişime açın. Şüphe durumunda yalnızca geri döngü erişimine dönün ve denetimi yeniden çalıştırın.
</Warning>

Bu çalışma kılavuzu, daha kapsamlı [Güvenlik](/tr/gateway/security) rehberini uzaktan erişim ve mesajlaşma yoluyla dış erişime açma için bir operatör kontrol listesine dönüştürür.

## Dış erişime açma modelini seçin

İş akışını karşılayan en dar kapsamlı modeli tercih edin.

| Model                      | Önerildiği durum                                 | Gerekli denetimler                                                                                                               |
| -------------------------- | ------------------------------------------------ | -------------------------------------------------------------------------------------------------------------------------------- |
| Geri döngü + SSH tüneli    | Kişisel kullanım, yönetici erişimi, hata ayıklama | `gateway.bind: "loopback"` değerini koruyun ve `127.0.0.1:18789` için tünel oluşturun                                                    |
| Geri döngü + Tailscale Serve | Control UI/WebSocket'e kişisel tailnet erişimi | Gateway'i yalnızca geri döngüde tutun; Tailscale kimlik üstbilgileri diğer kimlik doğrulama yollarını değil, yalnızca Control UI WebSocket yüzeyini doğrular |
| Tailnet/LAN bağlama        | Bilinen cihazların bulunduğu ayrılmış özel ağ    | Gateway kimlik doğrulaması, güvenlik duvarı izin listesi, herkese açık port yönlendirmesi olmaması                                 |
| Güvenilir ters proxy       | Gateway'in önünde kuruluş SSO/OIDC'si            | `trusted-proxy` kimlik doğrulaması, sıkı `trustedProxies`, üstbilgi üzerine yazma/kaldırma kuralları, açıkça izin verilen kullanıcılar |
| Herkese açık internet      | Nadir, yüksek riskli dağıtımlar                   | Kimlik farkındalığına sahip proxy, TLS, hız sınırları, sıkı izin listeleri, korumalı alandaki ana olmayan oturumlar                |

Gateway'e doğrudan herkese açık port yönlendirmesinden kaçının. Herkese açık erişim gerekiyorsa önüne kimlik farkındalığına sahip bir proxy yerleştirin ve proxy'yi Gateway'e giden tek ağ yolu hâline getirin.

## Ön kontrol envanteri

Bağlama, proxy, Tailscale veya kanal politikasını değiştirmeden önce şunları kaydedin:

- Gateway ana makinesi, işletim sistemi kullanıcısı ve durum dizini (varsayılan `~/.openclaw`).
- Gateway URL'si ve bağlama modu (`gateway.bind`; varsayılan port `18789`).
- Kimlik doğrulama modu, token/parola kaynağı veya güvenilir proxy kimlik kaynağı.
- Etkinleştirilen her kanal ve kanalın DM'leri, grupları veya webhook'ları kabul edip etmediği.
- Yerel olmayan göndericilerin erişebildiği ajanlar.
- Erişilebilen her ajanın araç profili, korumalı alan modu ve yükseltilmiş araç politikası.
- Bu ajanların erişebildiği harici kimlik bilgileri.
- `~/.openclaw/openclaw.json` ve kimlik bilgilerinin yedekleme konumu.

Bota birden fazla kişi mesaj gönderebiliyorsa bunu kullanıcı başına ana makine yalıtımı olarak değil, paylaşılan ve devredilmiş araç yetkisi olarak değerlendirin.

## Temel kontroller

Erişimi açmadan önce çalıştırın:

```bash
openclaw doctor
openclaw security audit
openclaw security audit --deep
openclaw health
```

Önce kritik bulguları giderin. Uyarıları yalnızca dağıtım için kasıtlı ve belgelenmiş olduklarında kabul edin. Her bir `checkId` değerinin anlamı ve düzeltme anahtarı için [Güvenlik denetimi kontrolleri](/tr/gateway/security/audit-checks) bölümüne bakın.

Uzaktan CLI doğrulaması için kimlik bilgilerini açıkça iletin:

```bash
openclaw gateway probe --url ws://127.0.0.1:18789 --token "$OPENCLAW_GATEWAY_TOKEN"
```

Yerel yapılandırmadaki kimlik bilgilerinin açıkça belirtilmiş bir uzak URL için geçerli olduğunu varsaymayın.

## Asgari güvenli temel

Dış erişime açık dağıtımlar için başlangıç noktası olarak bu yapıyı kullanın:

```json5
{
  gateway: {
    bind: "loopback",
    auth: {
      mode: "token",
      token: "replace-with-a-long-random-token",
    },
  },
  session: {
    dmScope: "per-channel-peer",
  },
  agents: {
    defaults: {
      sandbox: { mode: "non-main" },
    },
  },
  tools: {
    profile: "messaging",
    exec: { security: "deny", ask: "always" },
    elevated: { enabled: false },
  },
}
```

Her seferinde bir denetimin kapsamını genişletin: yazma yeteneğine sahip araçları etkinleştirmeden önce belirli bir kanal izin listesi ekleyin veya uzaktaki Control UI trafiğini kabul etmeden önce bir ters proxy etkinleştirin.

`tools.exec.security: "deny"`, zararsız tanılamalar dâhil tüm exec çağrılarını engeller. Tanılamalar veya düşük riskli komutlar gerekiyorsa bunu yalnızca tehdit modelinizle eşleşen belirli göndericileri, ajanları, komutları ve onay modunu seçtikten sonra gevşetin.

## DM ve grup üzerinden dış erişim

Mesajlaşma kanalları güvenilmeyen girdi yüzeyleridir. DM'lere veya gruplara izin vermeden önce:

- `dmPolicy: "open"` yerine `dmPolicy: "pairing"` veya sıkı bir `allowFrom` listesi tercih edin.
- `"*"` izin listelerini geniş araç erişimiyle birleştirmeyin.
- Oda sıkı biçimde denetlenmiyorsa gruplarda bahsetme zorunluluğu getirin.
- Birden fazla kişi bota DM gönderebiliyorsa DM oturumlarının bağlam paylaşmaması için `session.dmScope: "per-channel-peer"` (veya çok hesaplı kanallar için `"per-account-channel-peer"`) ayarlayın.
- Paylaşılan kanalları asgari araçlara sahip ve kişisel kimlik bilgileri bulunmayan ajanlara yönlendirin.

Eşleştirme, göndericinin botu tetiklemesini onaylar. Bu, göndericiyi ayrı bir ana makine güvenlik sınırı hâline getirmez.

## Ters proxy kontrolleri

Kimlik farkındalığına sahip proxy'ler için:

- Proxy, Gateway'e yönlendirmeden önce kullanıcıların kimliğini doğrulamalıdır.
- Güvenlik duvarı veya ağ politikası, Gateway portuna doğrudan erişimi engellemelidir.
- `gateway.trustedProxies` yalnızca proxy kaynak IP'lerini listelemelidir.
- Proxy, istemci tarafından sağlanan kimlik ve yönlendirme üstbilgilerini kaldırmalı veya bunların üzerine yazmalıdır.
- Proxy birden fazla hedef kitleye hizmet veriyorsa `gateway.auth.trustedProxy.allowUsers` ayarlayın.
- `gateway.auth.trustedProxy.allowLoopback` değerini yalnızca yerel süreçlerin güvenilir olduğu ve kimlik üstbilgilerinin proxy'nin denetiminde bulunduğu aynı ana makinedeki bir proxy için kullanın.

Proxy değişikliklerinden sonra `openclaw security audit --deep` çalıştırın. Proxy kimlik doğrulama sınırı hâline geldiğinden, güvenilir proxy bulguları güçlü sinyallerdir.

## Araç ve korumalı alan incelemesi

Bir ajanı uzaktaki göndericilerin erişimine açmadan önce:

- Hangi oturumların ana makinede, hangilerinin korumalı alanda çalıştığını doğrulayın.
- Ana makinede exec kullanımını reddedin veya onay zorunluluğu getirin.
- Belirli ve güvenilir bir gönderici ihtiyaç duymadıkça yükseltilmiş araçları devre dışı tutun.
- Açık veya yarı açık mesajlaşma yüzeylerinde tarayıcı, canvas, node, cron, gateway ve oturum oluşturma araçlarından kaçının.
- Bağlama noktalarını dar kapsamlı tutun; kimlik bilgisi, ev dizini, Docker soketi ve sistem yollarından kaçının.
- Önemli ölçüde farklı güven sınırları için ayrı gateway'ler, işletim sistemi kullanıcıları veya ana makineler kullanın.

Uzaktaki kullanıcılara tamamen güvenilmiyorsa yalıtım yalnızca istemlerden veya oturum etiketlerinden değil, ayrı dağıtımlardan sağlanmalıdır.

## Değişiklik sonrası doğrulama

Her dış erişim değişikliğinden sonra:

1. `openclaw security audit --deep` komutunu yeniden çalıştırın.
2. Yetkilendirilmiş bir bağlantının başarıyla kurulduğunu doğrulayın.
3. Yetkisiz bir göndericinin veya tarayıcı oturumunun reddedildiğini doğrulayın.
4. Günlüklerde gizli bilgilerin maskelendiğini doğrulayın.
5. DM/grup yönlendirmesinin yalnızca amaçlanan ajana ulaştığını doğrulayın.
6. Yüksek etkili araçların onay istediğini veya reddedildiğini doğrulayın.
7. Kabul edilen kalan uyarıları belgeleyin.

Mevcut dış erişim değişikliği anlaşılmadan bir sonrakine geçmeyin.

## Geri alma planı

Gateway gereğinden fazla dış erişime açılmış olabilir:

```json5
{
  gateway: {
    bind: "loopback",
  },
  channels: {
    whatsapp: { dmPolicy: "disabled" },
    telegram: { dmPolicy: "disabled" },
    discord: { dmPolicy: "disabled" },
    slack: { dmPolicy: "disabled" },
  },
  tools: {
    exec: { security: "deny", ask: "always" },
    elevated: { enabled: false },
  },
}
```

Ardından:

1. Herkese açık yönlendirmeyi, Tailscale Funnel'ı veya ters proxy yollarını durdurun.
2. Gateway token'larını/parolalarını ve etkilenen entegrasyon kimlik bilgilerini yenileyin.
3. `"*"` ve beklenmeyen göndericileri izin listelerinden kaldırın.
4. Son denetim günlüklerini, çalıştırma geçmişini, araç çağrılarını ve yapılandırma değişikliklerini inceleyin.
5. `openclaw security audit --deep` komutunu yeniden çalıştırın.
6. İş akışını karşılayan en dar kapsamlı modelle erişimi yeniden etkinleştirin.

## İnceleme kontrol listesi

- Belgelenmiş bir gerekçe olmadıkça Gateway yalnızca geri döngüde kalır.
- Geri döngü dışı erişimde kimlik doğrulama ve güvenlik duvarı vardır; doğrudan herkese açık yol yoktur.
- Güvenilir proxy dağıtımlarında sıkı proxy IP'leri ve üstbilgi denetimleri vardır.
- DM'ler varsayılan olarak açık erişimi değil, eşleştirme veya izin listelerini kullanır.
- Gruplar bahsetme veya açık izin listeleri gerektirir.
- Paylaşılan kanallar kişisel kimlik bilgilerine erişmez.
- Ana olmayan oturumlar korumalı alan modunda çalışır.
- Ana makinede exec ve yükseltilmiş araçlar reddedilir veya onaya bağlanır.
- Günlüklerde gizli bilgiler maskelenir.
- Kritik denetim bulguları giderilir.
- Geri alma adımları test edilir ve belgelenir.
