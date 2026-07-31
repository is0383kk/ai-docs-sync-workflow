---
read_when:
    - Kimlik doğrulama profili çözümleme veya kimlik bilgisi yönlendirme üzerinde çalışma
    - Model kimlik doğrulama hatalarında veya profil sıralamasında hata ayıklama
summary: Kimlik doğrulama profilleri için standart kimlik bilgisi uygunluğu ve çözümleme semantiği
title: Kimlik doğrulama kimlik bilgisi semantiği
x-i18n:
    generated_at: "2026-07-26T23:11:59Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 6b0516b1bb23f400d5ac5fd39a628736034440216ac22823eef061b38564dff0
    source_path: auth-credential-semantics.md
    workflow: 16
---

Bu semantikler, seçim zamanı ve çalışma zamanı kimlik doğrulama davranışlarını uyumlu tutar. Şunlar tarafından paylaşılır:

- `resolveAuthProfileOrder` (profil sıralaması)
- `resolveApiKeyForProfile` (çalışma zamanı kimlik bilgisi çözümleme)
- `openclaw models status --probe`
- `openclaw doctor` kimlik doğrulama denetimleri (`doctor-auth`)

## Kararlı yoklama neden kodları

Yoklama sonuçları, yoklama bir model çağrısına hiç ulaşmadığında bir `status` kategorisi (`ok`, `auth`, `rate_limit`, `billing`, `timeout`, `format`, `unknown`, `no_model`) ile birlikte kararlı bir `reasonCode` taşır:

| `reasonCode`             | Anlamı                                                                       |
| ------------------------ | ---------------------------------------------------------------------------- |
| `excluded_by_auth_order` | Profil, sağlayıcısı için açık kimlik doğrulama sıralamasından çıkarılmıştır. |
| `missing_credential`     | Satır içi kimlik bilgisi veya SecretRef yapılandırılmamıştır.                |
| `expired`                | Token `expires` geçmiştedir.                                                |
| `invalid_expires`        | `expires` geçerli bir pozitif Unix ms zaman damgası değildir.                |
| `unresolved_ref`         | Yapılandırılan SecretRef çözümlenememiştir.                                  |
| `ineligible_profile`     | Profil, sağlayıcı yapılandırmasıyla uyumsuzdur (hatalı biçimlendirilmiş anahtar girdisi dâhil). |
| `no_model`               | Kimlik bilgileri mevcuttur ancak yoklanabilir bir model adayı çözümlenememiştir. |

Uygunluk denetimleri, kullanılabilir kimlik bilgilerinin neden kodunu `ok` olarak bildirir.

## Token kimlik bilgileri

Token kimlik bilgileri (`type: "token"`), satır içi `token` ve/veya `tokenRef` desteği sunar.

### Uygunluk kuralları

1. Hem `token` hem de `tokenRef` yoksa token profili uygun değildir (`missing_credential`).
2. `expires` isteğe bağlıdır. Mevcut olduğunda, `0` değerinden büyük ve en yüksek JavaScript `Date` zaman damgasından (8640000000000000) büyük olmayan, Unix başlangıç zamanından itibaren milisaniye cinsinde sonlu bir sayı olmalıdır.
3. `expires` geçersizse (yanlış tür, `NaN`, `0`, negatif, sonlu olmayan veya bu üst sınırı aşan), profil `invalid_expires` ile uygun değildir.
4. `expires` geçmişteyse profil `expired` ile uygun değildir.
5. `tokenRef`, `expires` doğrulamasını atlamaz.

### Çözümleme kuralları

1. Çözümleyici semantikleri, `expires` için uygunluk semantikleriyle eşleşir.
2. Uygun profillerde token materyali, satır içi değerden veya `tokenRef` üzerinden çözümlenebilir.
3. Çözümlenemeyen başvurular, `models status --probe` çıktısında `unresolved_ref` üretir.

## Agent kopyalama taşınabilirliği

Agent kimlik doğrulama devralımı, doğrudan okuma yoluyla gerçekleştirilir. Bir agent'ın yerel profili olmadığında, gizli materyali kendi kimlik bilgisi deposuna kopyalamadan çalışma zamanında varsayılan/ana agent deposundaki profilleri çözümler (`agents/<agentId>/agent/openclaw-agent.sqlite`).

`openclaw agents add` gibi açık kopyalama akışları şu taşınabilirlik politikasını kullanır:

- `copyToAgents: false` olmadığı sürece `api_key` ve `token` profilleri taşınabilirdir.
- Yenileme token'ları tek kullanımlık veya rotasyona duyarlı olabileceğinden `oauth` profilleri varsayılan olarak taşınabilir değildir.
- Sağlayıcının sahip olduğu OAuth akışları, yenileme materyalinin agent'lar arasında kopyalanmasının güvenli olduğu biliniyorsa `copyToAgents: true` ile etkinleştirilebilir; bu etkinleştirme yalnızca profil satır içi erişim/yenileme materyali taşıdığında geçerlidir.

Taşınabilir olmayan profiller, hedef agent ayrı olarak oturum açıp kendi yerel profilini oluşturmadığı sürece doğrudan okuma yoluyla devralım üzerinden kullanılabilir kalır.

## Yalnızca yapılandırmaya dayalı kimlik doğrulama rotaları

`mode: "aws-sdk"` içeren `auth.profiles` girdileri, depolanan kimlik bilgileri değil, yönlendirme meta verileridir. Hedef sağlayıcı, Plugin'in sahip olduğu Amazon Bedrock kurulumunun yazdığı rota olan `models.providers.<id>.auth: "aws-sdk"` kullandığında geçerlidir. Kimlik bilgisi deposunda eşleşen bir girdi bulunmasa bile bu profil kimlikleri `auth.order` ve oturum geçersiz kılmalarında görünebilir.

Kimlik bilgisi deposuna `type: "aws-sdk"` yazmayın; depolanan kimlik bilgileri yalnızca `api_key`, `token` veya `oauth` olabilir. Eski bir `auth-profiles.json` böyle bir işaretçiye sahipse `openclaw doctor --fix`, bunu `auth.profiles` konumuna taşır ve işaretçiyi depodan kaldırır.

## Açık kimlik doğrulama sıralaması filtrelemesi

- Bir sağlayıcı için `auth.order.<provider>` veya kimlik doğrulama deposu sıralaması geçersiz kılması ayarlandığında `models status --probe`, yalnızca o sağlayıcının çözümlenen kimlik doğrulama sıralamasında kalan profil kimliklerini yoklar. Depolanan geçersiz kılma, `auth.order` yapılandırmasına göre önceliklidir.
- Bu sağlayıcı için depolanmış ancak açık sıralamadan çıkarılmış bir profil daha sonra sessizce denenmez. Yoklama çıktısı bunu `reasonCode: excluded_by_auth_order` ve `Excluded by auth.order for this provider.` ayrıntısıyla bildirir

## Yoklama hedefi çözümleme

- Yoklama hedefleri kimlik doğrulama profillerinden, ortam kimlik bilgilerinden veya `models.json` üzerinden gelebilir (sonuç `source`: `profile`, `env`, `models.json`).
- Bir sağlayıcının kimlik bilgileri varsa ancak OpenClaw bu sağlayıcı için yoklanabilir bir model adayı çözümleyemiyorsa `models status --probe`, `reasonCode: no_model` ile `status: no_model` bildirir.

## Harici CLI kimlik bilgisi keşfi

- Harici CLI'ların sahip olduğu yalnızca çalışma zamanına özgü kimlik bilgileri (`claude-cli` için Claude CLI, `openai` için Codex CLI, `minimax-portal` için MiniMax CLI) yalnızca sağlayıcı, çalışma zamanı veya kimlik doğrulama profili mevcut işlemin kapsamındaysa ya da bu harici kaynak için depolanmış yerel bir profil zaten varsa keşfedilir.
- Kimlik doğrulama deposu çağıranları açık bir harici CLI keşif modu seçer: yalnızca kalıcı/Plugin kimlik doğrulaması için `none`, önceden depolanmış harici CLI profillerini yenilemek için `existing` veya somut bir sağlayıcı/profil kümesi için `scoped`.
- Salt okunur/durum yolları `allowKeychainPrompt: false` geçirir; yalnızca dosya tabanlı harici CLI kimlik bilgilerini kullanır ve macOS Keychain sonuçlarını okumaz veya yeniden kullanmaz.

## OAuth SecretRef Politikası Koruması

SecretRef girdisi yalnızca statik kimlik bilgileri içindir. OAuth kimlik bilgileri çalışma zamanında değiştirilebilir (yenileme akışları rotasyona uğramış token'ları kalıcı olarak kaydeder), bu nedenle SecretRef destekli OAuth materyali değiştirilebilir durumu depolar arasında böler.

- Bir profil kimlik bilgisi `type: "oauth"` ise o profildeki tüm kimlik bilgisi materyali alanlarında SecretRef nesneleri reddedilir.
- `auth.profiles.<id>.mode`, `"oauth"` ise bu profil için SecretRef destekli `keyRef`/`tokenRef` girdisi reddedilir.
- İhlaller, başlangıç/yeniden yükleme gizli bilgi hazırlama ve profil çözümleme yollarında kesin hatalardır (hata fırlatılır).

## Eski Sürümlerle Uyumlu Mesajlaşma

Betik uyumluluğu için yoklama hataları bu ilk satırı değiştirmeden korur:

`Auth profile credentials are missing or expired.`

İnsanların kolayca anlayabileceği ayrıntı ve kararlı neden kodu, sonraki satırlarda `↳ Auth reason [code]: ...` biçiminde yer alır.

## İlgili

- [Gizli bilgi yönetimi](/tr/gateway/secrets)
- [Kimlik doğrulama depolaması](/tr/concepts/oauth)
