---
read_when:
    - Yapılandırma/durum üzerinde hızlı bir güvenlik denetimi çalıştırmak istiyorsunuz
    - Güvenli "düzeltme" önerilerini uygulamak istiyorsunuz (izinler, varsayılanları sıkılaştırma)
summary: '`openclaw security` için CLI başvurusu (yaygın güvenlik tuzaklarını denetleme ve düzeltme)'
title: Güvenlik
x-i18n:
    generated_at: "2026-07-26T22:42:01Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 6b5f9ea5cb746bfd29ff4d096062e81595abe99a883fc3b1113b45a3527d42d9
    source_path: cli/security.md
    workflow: 16
---

# `openclaw security`

Güvenlik araçları: denetim ve isteğe bağlı güvenli düzeltmeler. İlgili: [Güvenlik](/tr/gateway/security).

```bash
openclaw security audit
openclaw security audit --deep
openclaw security audit --deep --password <password>
openclaw security audit --deep --token <token>
openclaw security audit --auth password --password <password>
openclaw security audit --fix
openclaw security audit --json
```

## Denetim modları

Düz `security audit`, soğuk yapılandırma/dosya sistemi/salt okunur yolunda kalır: Plugin çalışma zamanı güvenlik toplayıcılarını keşfetmez; böylece rutin denetimler, kurulu her Plugin çalışma zamanını yüklemez. `--deep`, mümkün olduğunca canlı Gateway yoklamaları ve Plugin'e ait güvenlik denetimi toplayıcıları ekler (uygun bir çalışma zamanı kapsamına zaten sahip olan açıkça belirtilmiş dahili çağıranlar da bu toplayıcıları etkinleştirebilir).

Gateway parola kimlik doğrulaması yalnızca başlangıçta sağlanıyorsa denetimin bunu `hooks.token` ile karşılaştırabilmesi için aynı değeri `--auth password --password <password>` ile iletin.

## Neleri denetler

**DM/güven modeli**

- Birden fazla DM göndericisi ana oturumu paylaştığında uyarır ve paylaşılan gelen kutuları için güvenli DM modunu önerir: `session.dmScope="per-channel-peer"` (veya çok hesaplı kanallar için `per-account-channel-peer`). Bu, karşılıklı olarak güvenilmeyen operatörler arasında yalıtım değil, iş birliğine dayalı/paylaşılan gelen kutusu sağlamlaştırmasıdır; böyle durumlarda güven sınırlarını ayrı Gateway'lerle (veya ayrı işletim sistemi kullanıcıları/ana makineleriyle) ayırın.
- Yapılandırma olası paylaşılan kullanıcı girişine işaret ettiğinde (örneğin açık DM/grup politikası, yapılandırılmış grup hedefleri veya joker karakterli gönderici kuralları) `security.trust_model.multi_user_heuristic` üretir — OpenClaw'ın varsayılan güven modeli kişisel asistandır (tek operatör); düşmanca çok kiracılı yalıtım değildir. Bilinçli paylaşılan kullanıcı kurulumlarında: tüm oturumları korumalı alanda çalıştırın, dosya sistemi erişimini çalışma alanıyla sınırlı tutun ve kişisel/özel kimlikleri veya kimlik bilgilerini bu çalışma zamanından uzak tutun.
- Küçük modeller (`<=300B` parametre) korumalı alan olmadan ve web/tarayıcı araçları etkin şekilde kullanıldığında uyarır.

**Webhook/hook'lar**

Başlangıçta ölümcül olmayan bir güvenlik uyarısı günlüğe kaydedilir ve denetim, etkin Gateway paylaşılan gizli anahtar kimlik doğrulama değerlerinin (`gateway.auth.token` / `OPENCLAW_GATEWAY_TOKEN`, `gateway.auth.password` / `OPENCLAW_GATEWAY_PASSWORD`) `hooks.token` tarafından yeniden kullanılmasını işaretler. Ayrıca şu durumlarda uyarır:

- `hooks.token` kısa olduğunda
- `hooks.path="/"`
- `hooks.defaultSessionKey` ayarlanmadığında
- `hooks.allowedAgentIds` sınırsız olduğunda
- istek `sessionKey` geçersiz kılmaları etkinleştirildiğinde
- geçersiz kılmalar `hooks.allowedSessionKeyPrefixes` olmadan etkinleştirildiğinde

Kalıcı olarak saklanan ve yeniden kullanılan bir `hooks.token` değerini yenilemek için `openclaw doctor --fix` komutunu çalıştırın, ardından harici hook göndericilerini yeni token'ı kullanacak şekilde güncelleyin.

**Korumalı alan/araçlar**

- Korumalı alan modu kapalıyken korumalı alan Docker ayarları yapılandırılmışsa uyarır.
- `gateway.nodes.commands.deny` etkisiz desen benzeri/bilinmeyen girdiler kullandığında uyarır (eşleştirme yalnızca tam Node komut adına göre yapılır, kabuk metni filtrelemesi değildir).
- `gateway.nodes.commands.allow` tehlikeli Node komutlarını açıkça etkinleştirdiğinde uyarır.
- Genel `tools.profile="minimal"` değeri, aracı profilleri tarafından geçersiz kılındığında uyarır.
- Yazma/düzenleme araçları devre dışı olduğu hâlde `exec` sınırlayıcı bir korumalı alan dosya sistemi sınırı olmadan hâlâ kullanılabiliyorsa uyarır.
- Açık DM'ler veya gruplar, çalışma zamanı/dosya sistemi araçlarını korumalı alan/çalışma alanı korumaları olmadan erişime açtığında uyarır.
- Kurulu Plugin araçlarına izin verici araç politikası kapsamında erişilebileceğinde uyarır.

**Korumalı alan tarayıcısı**

- Korumalı alan tarayıcısı, `sandbox.browser.cdpSourceRange` olmadan Docker `bridge` ağını kullandığında uyarır.
- `host` ve `container:*` ad alanı katılımları dâhil olmak üzere tehlikeli korumalı alan Docker ağ modlarını işaretler.
- Mevcut korumalı alan tarayıcısı Docker konteynerlerinde eksik/eski hash etiketleri olduğunda (örneğin `openclaw.browserConfigEpoch` bulunmayan geçiş öncesi konteynerler) uyarır ve `openclaw sandbox recreate --browser --all` önerir.

**Ağ/keşif**

- `gateway.allowRealIpFallback=true` değerini işaretler (proxy'ler yanlış yapılandırılırsa üstbilgi sahteciliği riski).
- `discovery.mdns.mode="full"` değerini işaretler (mDNS TXT kayıtları üzerinden meta veri sızıntısı).
- `gateway.auth.mode="none"`, Gateway HTTP API'lerini paylaşılan bir gizli anahtar olmadan erişilebilir bıraktığında (`/tools/invoke` ve etkinleştirilmiş herhangi bir `/v1/*` uç noktası) uyarır.

**Plugin'ler/kanallar**

- npm tabanlı Plugin/hook kurulum kayıtları belirli bir sürüme sabitlenmemişse, bütünlük meta verileri eksikse veya şu anda kurulu paket sürümlerinden sapmışsa uyarır.
- Kanal izin listeleri kararlı kimlikler yerine değiştirilebilir adlara/e-postalara/etiketlere dayanıyorsa uyarır (uygun olduğu yerlerde Discord, Slack, Google Chat, Microsoft Teams, Mattermost ve IRC kapsamları).

`dangerous`/`dangerously` önekli ayarlar, operatörlerin acil durumlarda kullandığı açık geçersiz kılmalardır; bunlardan birini etkinleştirmek tek başına bir güvenlik açığı bildirimi değildir. Tehlikeli parametrelerin tam envanteri için [Güvenlik](/tr/gateway/security) bölümündeki "Güvenli olmayan veya tehlikeli bayrakların özeti" başlığına bakın.

## SecretRef davranışı

`security audit`, hedeflenen yolları için desteklenen SecretRef'leri salt okunur modda çözümler. Bir SecretRef mevcut komut yolunda kullanılamıyorsa denetim çökmek yerine devam eder ve `secretDiagnostics` bildirir. `--token` ve `--password`, derin yoklama kimlik doğrulamasını yalnızca ilgili komut çağrısı için geçersiz kılar; yapılandırmayı veya SecretRef eşlemelerini yeniden yazmaz.

## Gizlemeler

Bilinçli olarak sürdürülen bulguları `security.audit.suppressions` ile kabul edin. Her gizleme tam bir `checkId` ile eşleşir ve büyük/küçük harfe duyarsız `titleIncludes` ve/veya `detailIncludes` alt dizeleriyle daraltılabilir:

```json
{
  "security": {
    "audit": {
      "suppressions": [
        {
          "checkId": "plugins.tools_reachable_permissive_policy",
          "detailIncludes": "Etkin uzantı Plugin'leri: gbrain",
          "reason": "güvenilir yerel operatör Plugin'i"
        }
      ]
    }
  }
}
```

Gizlenen bulgular etkin `summary` ve `findings` listesinden kaldırılır. JSON çıktısı, denetlenebilirlik için bunları `suppressedFindings` altında tutar. Gizlemeler yapılandırıldığında etkin çıktı, okuyucuların denetimin filtrelendiğini anlayabilmesi için gizlenemeyen bir `security.audit.suppressions.active` bilgi bulgusunu da tutar. Tehlikeli yapılandırma bayrakları bulgu başına bir bayrak olarak üretilir; böylece bir tehlikeli bayrağın kabul edilmesi, aynı `config.insecure_or_dangerous_flags` checkId değerini paylaşan diğer etkin bayrakları gizlemez.

Gizlemeler süregelen riskleri saklayabildiğinden, exec zaten güvenilir yerel otomasyon için `security="full"` ve `ask="off"` ile çalışmıyorsa bunları aracı tarafından çalıştırılan kabuk komutlarıyla eklemek veya kaldırmak exec onayı gerektirir.

## JSON çıktısı

```bash
openclaw security audit --json | jq '.summary'
openclaw security audit --deep --json | jq '.findings[] | select(.severity=="critical") | .checkId'
```

`--fix --json` ile çıktı hem düzeltme eylemlerini hem de son raporu içerir:

```bash
openclaw security audit --fix --json | jq '{fix: .fix.ok, summary: .report.summary}'
```

## `--fix` neleri değiştirir

Güvenli ve deterministik düzeltmeleri uygular:

- yaygın `groupPolicy="open"` değerlerini `groupPolicy="allowlist"` olarak değiştirir (desteklenen kanallardaki hesap çeşitleri dâhil)
- WhatsApp grup politikası `allowlist` olarak değiştiğinde, bu liste mevcutsa ve yapılandırma henüz `allowFrom` tanımlamıyorsa `groupAllowFrom` değerini saklanan `allowFrom` dosyasından doldurur
- `logging.redactSensitive` değerini `"off"` yerine `"tools"` olarak ayarlar
- durum/yapılandırma ve yaygın hassas dosyaların (`credentials/*.json`, `auth-profiles.json`, `openclaw-agent.sqlite` ve eski oturum yapıtları) izinlerini sıkılaştırır
- ayrıca `openclaw.json` içinden başvurulan yapılandırma içerme dosyalarının izinlerini sıkılaştırır
- POSIX ana makinelerinde `chmod`, Windows'ta ise `icacls` sıfırlamalarını kullanır

`--fix` şunları **yapmaz**:

- token'ları/parolaları/API anahtarlarını yenilemez
- araçları (`gateway`, `cron`, `exec` vb.) devre dışı bırakmaz
- Gateway bağlama/kimlik doğrulama/ağ erişimi tercihlerini değiştirmez
- Plugin'leri/Skills'i kaldırmaz veya yeniden yazmaz

## İlgili

- [CLI başvurusu](/tr/cli)
- [Güvenlik denetimi](/tr/gateway/security)
