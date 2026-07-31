---
read_when:
    - Çalışma zamanında gizli bilgi referanslarını yeniden çözümleme
    - Düz metin kalıntılarını ve çözümlenmemiş başvuruları denetleme
    - SecretRef’leri yapılandırma ve tek yönlü temizleme değişikliklerini uygulama
summary: '`openclaw secrets` için CLI başvurusu (yeniden yükleme, denetleme, yapılandırma, uygulama)'
title: Gizli Bilgiler
x-i18n:
    generated_at: "2026-07-26T23:54:38Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 61f6f81e358ca2e6a97ac9498186b32f7a74d16052d226c398dad0030d47211e
    source_path: cli/secrets.md
    workflow: 16
---

# `openclaw secrets`

SecretRef'leri yönetin ve etkin çalışma zamanı anlık görüntüsünü sağlıklı tutun.

| Komut       | Rol                                                                                                                                                                                                 |
| ----------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `reload`    | Gateway RPC (`secrets.reload`): referansları yeniden çözümler ve sahip bilgisini gözeten çalışma zamanı anlık görüntüsünü atomik olarak yayımlar (yapılandırmaya yazmaz); uygun sahip hataları soğuk veya eski uyarıları olarak yayımlanabilir |
| `audit`     | Düz metin, çözümlenmemiş referanslar ve öncelik sapması için yapılandırma/kimlik doğrulama/oluşturulan model depolarının ve eski kalıntıların salt okunur taraması (exec referansları, `--allow-exec` olmadığı sürece atlanır) |
| `configure` | Sağlayıcı kurulumu, hedef eşleme ve ön kontrol için etkileşimli planlayıcı (TTY gerektirir)                                                                                                           |
| `apply`     | Kaydedilmiş bir planı yürütür (`--dry-run` yalnızca doğrular ve varsayılan olarak exec denetimlerini atlar; yazma modu, `--allow-exec` olmadığı sürece exec içeren planları reddeder), ardından hedeflenen düz metin kalıntılarını temizler |

Önerilen operatör döngüsü:

```bash
openclaw secrets audit --check
openclaw secrets configure
openclaw secrets apply --from /tmp/openclaw-secrets-plan.json --dry-run
openclaw secrets apply --from /tmp/openclaw-secrets-plan.json
openclaw secrets audit --check
openclaw secrets reload
```

Planınız `exec` SecretRef'leri/sağlayıcıları içeriyorsa hem deneme çalıştırması hem de yazma `apply` komutlarında `--allow-exec` iletin.

CI/geçitler için çıkış kodları:

- `audit --check`, bulgular olduğunda `1` döndürür.
- Çözümlenmemiş referanslar (`--check` değerinden bağımsız olarak) `2` döndürür.

İlgili: [Gizli Bilgi Yönetimi](/tr/gateway/secrets) · [SecretRef Kimlik Bilgisi Yüzeyi](/tr/reference/secretref-credential-surface) · [Güvenlik](/tr/gateway/security)

## Çalışma zamanı anlık görüntüsünü yeniden yükleme

```bash
openclaw secrets reload
openclaw secrets reload --json
openclaw secrets reload --url ws://127.0.0.1:18789 --token <token>
```

Gateway RPC yöntemi `secrets.reload` kullanılır. Sağlıklı sahipler bağımsız olarak yenilenir. Uygun başarısız sahipler yalnızca referans kimlikleri, sağlayıcı tanımları ve gizli olmayan eksiksiz sahip sözleşmesi değişmediğinde eski duruma gelir; yeni veya değişmiş hatalar soğuk duruma gelir. Bu indirgenmiş etkinleştirme başarılı olur ve `warningCount` bildirir. Katı veya eşlenmemiş hatalar bir hata döndürür ve daha önce etkin olan anlık görüntüyü korur.

Seçenekler: `--url <url>`, `--token <token>`, `--timeout <ms>`, `--json`.

## Denetim

OpenClaw durumunu şunlar için tarar:

- gizli bilgilerin düz metin olarak depolanması
- çözümlenmemiş referanslar
- öncelik sapması (`openclaw.json` referanslarını gölgeleyen `auth-profiles.json` kimlik bilgileri)
- oluşturulan `agents/*/agent/models.json` kalıntıları (sağlayıcı `apiKey` değerleri ve hassas sağlayıcı üstbilgileri)
- eski kalıntılar (eski kimlik doğrulama deposu girdileri, OAuth hatırlatıcıları)

`.env` taraması, geçerli durum dizinini ve etkin yapılandırmayı içeren dizini kapsar. Her iki yol da aynı dosyayı belirtiyorsa dosya bir kez taranır.

Hassas sağlayıcı üstbilgisi algılama, ad sezgiselliğine dayanır: adı yaygın kimlik doğrulama/kimlik bilgisi parçalarıyla (`authorization`, `x-api-key`, `token`, `secret`, `password`, `credential`) eşleşen üstbilgileri işaretler.

```bash
openclaw secrets audit
openclaw secrets audit --check
openclaw secrets audit --json
openclaw secrets audit --allow-exec
```

Rapor biçimi:

- `status`: `clean | findings | unresolved`
- `resolution`: `refsChecked`, `skippedExecRefs`, `resolvabilityComplete`
- `summary`: `plaintextCount`, `unresolvedRefCount`, `shadowedRefCount`, `legacyResidueCount`
- bulgu kodları: `PLAINTEXT_FOUND`, `REF_UNRESOLVED`, `REF_SHADOWED`, `LEGACY_RESIDUE`

## Yapılandırma (etkileşimli yardımcı)

Sağlayıcı ve SecretRef değişikliklerini etkileşimli olarak oluşturun, ön kontrolü çalıştırın ve isteğe bağlı olarak uygulayın:

```bash
openclaw secrets configure
openclaw secrets configure --plan-out /tmp/openclaw-secrets-plan.json
openclaw secrets configure --apply --yes
openclaw secrets configure --providers-only
openclaw secrets configure --skip-provider-setup
openclaw secrets configure --agent ops
openclaw secrets configure --json
```

Akış: önce sağlayıcı kurulumu (`secrets.providers` takma adlarını ekleme/düzenleme/kaldırma), ardından kimlik bilgisi eşleme (alanları seçme, `{source, provider, id}` referanslarını atama), son olarak ön kontrol ve isteğe bağlı uygulama.

Bayraklar:

- `--providers-only`: yalnızca `secrets.providers` yapılandırın, kimlik bilgisi eşlemeyi atlayın
- `--skip-provider-setup`: sağlayıcı kurulumunu atlayın, kimlik bilgilerini mevcut sağlayıcılarla eşleyin
- `--agent <id>`: `auth-profiles.json` hedef keşfini ve yazma işlemlerini tek bir aracı deposuyla sınırlandırın
- `--allow-exec`: ön kontrol/uygulama sırasında exec SecretRef denetimlerine izin verin (sağlayıcı komutlarını çalıştırabilir)

`--providers-only` ve `--skip-provider-setup` birlikte kullanılamaz.

Notlar:

- Etkileşimli bir TTY gerektirir.
- Seçili aracı kapsamı için `openclaw.json` içindeki gizli bilgi taşıyan alanları ve `auth-profiles.json` hedefler; standart olarak desteklenen yüzey: [SecretRef Kimlik Bilgisi Yüzeyi](/tr/reference/secretref-credential-surface).
- Seçici akışında doğrudan yeni `auth-profiles.json` eşlemeleri oluşturmayı destekler.
- Uygulamadan önce ön kontrol çözümlemesini çalıştırır.
- Oluşturulan planlarda temizleme seçenekleri varsayılan olarak etkindir (`scrubEnv`, `scrubAuthProfilesForProviderTargets`, `scrubLegacyAuthJson`). Temizlenen düz metin değerleri için uygulama işlemi tek yönlüdür.
- `--plan-out`, UTF-8 ile serileştirilmiş biçimi 16 MiB (16,777,216 bayt) değerini aşan bir plan oluşturmayı, `apply --from` girdi sınırıyla uyumlu olarak reddeder.
- `--apply` olmadan CLI, ön kontrolden sonra yine de `Apply this plan now?` istemini gösterir.
- `--apply` ile (ve `--yes` olmadan) CLI, geri döndürülemez geçiş için ek bir onay ister.
- `--json` planı ve ön kontrol raporunu yazdırır ancak yine de etkileşimli bir TTY gerektirir.

### Exec sağlayıcısı güvenliği

Homebrew kurulumları, genellikle `/opt/homebrew/bin/*` altında sembolik bağlantılı ikili dosyalar sunar. `allowSymlinkCommand: true` değerini yalnızca güvenilir paket yöneticisi yolları için gerektiğinde, `trustedDirs` ile eşleştirerek (örneğin `["/opt/homebrew"]`) ayarlayın. Windows'ta bir sağlayıcı yolu için ACL doğrulaması kullanılamıyorsa OpenClaw güvenli biçimde başarısız olur; yalnızca güvenilir yollar için yol güvenlik denetimini atlamak üzere ilgili sağlayıcıda `allowInsecurePath: true` ayarlayın.

## Kaydedilmiş bir planı uygulama

```bash
openclaw secrets apply --from /tmp/openclaw-secrets-plan.json
openclaw secrets apply --from /tmp/openclaw-secrets-plan.json --allow-exec
openclaw secrets apply --from /tmp/openclaw-secrets-plan.json --dry-run
openclaw secrets apply --from /tmp/openclaw-secrets-plan.json --dry-run --allow-exec
openclaw secrets apply --from /tmp/openclaw-secrets-plan.json --json
```

`--dry-run`, dosyalara yazmadan ön kontrolü doğrular; exec SecretRef denetimleri deneme çalıştırmasında varsayılan olarak atlanır. Yazma modu, `--allow-exec` olmadığı sürece exec SecretRef'leri/sağlayıcıları içeren planları reddeder. Her iki modda da exec sağlayıcısı denetimlerini/yürütmesini etkinleştirmek için `--allow-exec` kullanın.

`--from`, 16 MiB (16,777,216 bayt) değerinden büyük olmayan normal bir dosyayı göstermelidir. Bayt sınırı, boşluklar dâhil olmak üzere serileştirilmiş dosyanın tamamına uygulanır.

`apply` şunları güncelleyebilir:

- `openclaw.json` (SecretRef hedefleri + sağlayıcı ekleme/güncelleme/silme işlemleri)
- `auth-profiles.json` (sağlayıcı hedeflerini temizleme)
- eski `auth.json` kalıntıları
- değerleri taşınmış bilinen gizli anahtarlar için geçerli durum ve etkin yapılandırma dizinlerindeki `.env` dosyaları

Plan sözleşmesi ayrıntıları (izin verilen hedef yolları, doğrulama kuralları, hata semantiği): [Gizli Bilgileri Uygulama Planı Sözleşmesi](/tr/gateway/secrets-plan-contract).

### Neden geri alma yedekleri yok?

`secrets apply`, eski düz metin değerlerini içeren geri alma yedeklerini kasıtlı olarak yazmaz. Güvenlik, katı ön kontrol ve yaklaşık atomik uygulamanın yanı sıra hata durumunda en iyi çabayla bellek içi geri yükleme yoluyla sağlanır.

## Örnek

```bash
openclaw secrets audit --check
openclaw secrets configure
openclaw secrets audit --check
```

`audit --check` hâlâ düz metin bulguları bildiriyorsa bildirilen kalan hedef yollarını güncelleyin ve denetimi yeniden çalıştırın.

## İlgili

- [CLI başvurusu](/tr/cli)
- [Gizli bilgi yönetimi](/tr/gateway/secrets)
- [Vault SecretRef'leri](/tr/plugins/vault)
