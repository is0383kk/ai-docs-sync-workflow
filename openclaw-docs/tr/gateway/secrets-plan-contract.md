---
read_when:
    - '`openclaw secrets apply` planları oluşturma veya inceleme'
    - '`Invalid plan target path` hatalarında hata ayıklama'
    - Hedef türünü ve yol doğrulama davranışını anlama
summary: '`secrets apply` planları için sözleşme: hedef doğrulama, yol eşleştirme ve `auth-profiles.json` hedef kapsamı'
title: Gizli bilgileri uygulama planı sözleşmesi
x-i18n:
    generated_at: "2026-07-26T23:42:38Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 71ee8afd958646930af4db3bbad08e033ff79da48890a989d72b361abcbda3bb
    source_path: gateway/secrets-plan-contract.md
    workflow: 16
---

Bu sayfa, `openclaw secrets apply` tarafından uygulanan katı sözleşmeyi tanımlar. Bir hedef bu kurallarla eşleşmezse apply, herhangi bir dosyayı değiştirmeden önce başarısız olur.

## Plan dosyası gereksinimleri

`openclaw secrets apply --from <plan.json>`, 16 MiB'ye (16,777,216 bayt) kadar normal dosyaları kabul eder. Sınır, boşluklar dahil olmak üzere serileştirilmiş dosyanın tamamına uygulanır. Dizinler, FIFO'lar, aygıt dosyaları ve sınırdan daha büyük dosyalar JSON ayrıştırmasından veya hedef doğrulamasından önce reddedilir.

`openclaw secrets configure --plan-out <plan.json>`, dosyayı oluşturmadan önce UTF-8 olarak serileştirilmiş çıktıya aynı sınırı uygular. Elle yazılan planlar ve harici plan oluşturucular da serileştirilmiş dosyayı bu sınır içinde tutmalıdır.

## Plan dosyasının yapısı

`openclaw secrets apply --from <plan.json>`, plan hedeflerinden oluşan bir `targets` dizisi bekler:

```json5
{
  version: 1,
  protocolVersion: 1,
  targets: [
    {
      type: "models.providers.apiKey",
      path: "models.providers.openai.apiKey",
      pathSegments: ["models", "providers", "openai", "apiKey"],
      providerId: "openai",
      ref: { source: "env", provider: "default", id: "OPENAI_API_KEY" },
    },
    {
      type: "auth-profiles.api_key.key",
      path: "profiles.openai:default.key",
      pathSegments: ["profiles", "openai:default", "key"],
      agentId: "main",
      ref: { source: "env", provider: "default", id: "OPENAI_API_KEY" },
    },
  ],
}
```

`openclaw secrets configure`, planları bu yapıda oluşturur. Bir planı elle de yazabilir veya düzenleyebilirsiniz.

## Sağlayıcı ekleme/güncellemeleri ve silmeleri

Planlar, hedef başına yazmaların yanı sıra `secrets.providers` eşlemesini değiştiren iki isteğe bağlı üst düzey alan da içerebilir:

- `providerUpserts` -- sağlayıcı takma adına göre anahtarlanmış bir nesne. Her değer bir sağlayıcı tanımıdır (`openclaw.json` içindeki `secrets.providers.<alias>` altında kabul edilen yapıyla aynıdır; örneğin bir `exec` veya `file` sağlayıcısı).
- `providerDeletes` -- kaldırılacak sağlayıcı takma adlarından oluşan bir dizi.

`providerUpserts`, `targets` işleminden önce çalışır; böylece bir `target.ref.provider`, aynı planın `providerUpserts` içinde eklediği bir sağlayıcı takma adına başvurabilir. Bu sıralama olmadan, `openclaw.json` içinde henüz yapılandırılmamış bir takma ada başvuran planlar `provider "<alias>" is not configured` hatasıyla başarısız olur.

```json5
{
  version: 1,
  protocolVersion: 1,
  providerUpserts: {
    onepassword_anthropic: {
      source: "exec",
      command: "/usr/bin/op",
      args: ["read", "op://Vault/Anthropic/credential"],
    },
  },
  providerDeletes: ["legacy_unused_alias"],
  targets: [
    {
      type: "models.providers.apiKey",
      path: "models.providers.anthropic.apiKey",
      pathSegments: ["models", "providers", "anthropic", "apiKey"],
      providerId: "anthropic",
      ref: { source: "exec", provider: "onepassword_anthropic", id: "credential" },
    },
  ],
}
```

`providerUpserts` aracılığıyla eklenen exec sağlayıcıları, [Exec sağlayıcısı onay davranışı](#exec-provider-consent-behavior) bölümündeki exec onay kurallarına yine tabidir: exec sağlayıcıları içeren planlar yazma modunda `--allow-exec` gerektirir.

## Desteklenen hedef kapsamı

Plan hedefleri, [SecretRef Kimlik Bilgisi Yüzeyi](/tr/reference/secretref-credential-surface) bölümündeki desteklenen kimlik bilgisi yolları için kabul edilir.

## Hedef türü davranışı

`target.type`, tanınan bir hedef türü olmalı ve normalleştirilmiş `target.path`, bu türün kayıtlı yol yapısıyla eşleşmelidir.

Bazı hedef türleri, kurallı tür adlarına ek olarak mevcut planlar için `target.type` biçiminde bir uyumluluk takma adı kabul eder:

| Kurallı tür                          | Kabul edilen takma ad                           |
| ------------------------------------ | ----------------------------------------------- |
| `models.providers.apiKey`            | `models.providers.*.apiKey`                     |
| `skills.entries.apiKey`              | `skills.entries.*.apiKey`                       |
| `channels.googlechat.serviceAccount` | `channels.googlechat.accounts.*.serviceAccount` |

## Yol doğrulama kuralları

Her hedef aşağıdakilerin tümüyle doğrulanır:

- `type`, tanınan bir hedef türü olmalıdır.
- `path`, boş olmayan noktayla ayrılmış bir yol olmalıdır.
- `pathSegments` atlanabilir. Sağlanırsa tam olarak `path` ile aynı yola normalleştirilmelidir.
- Yasaklanmış segmentler reddedilir: `__proto__`, `prototype`, `constructor`.
- Normalleştirilmiş yol, hedef türü için kayıtlı yol yapısıyla eşleşmelidir.
- `providerId` veya `accountId` ayarlanmışsa yolda kodlanan kimlikle eşleşmelidir.
- `auth-profiles.json` hedefleri `agentId` gerektirir.
- Yeni bir `auth-profiles.json` eşlemesi oluştururken `authProfileProvider` değerini ekleyin.

## Başarısızlık davranışı

Bir hedef doğrulamadan geçemezse apply, aşağıdakine benzer bir hatayla çıkar:

```text
models.providers.apiKey için geçersiz plan hedef yolu: models.providers.openai.baseUrl
```

Geçersiz bir plan için hiçbir yazma işlemi kaydedilmez: hedef çözümleme ve yol doğrulaması herhangi bir dosyaya dokunulmadan önce çalışır. Ayrıca geçerli bir plan yazmaya başladıktan sonra apply, önce dokunulan her dosyanın anlık görüntüsünü alır ve aynı çalıştırmadaki sonraki bir yazma işlemi başarısız olursa bu anlık görüntüleri geri yükler; böylece kısmi bir yazma işlemi yapılandırma, kimlik doğrulama profili veya env durumunu hiçbir zaman eşzamanlılıktan çıkarmaz.

## Exec sağlayıcısı onay davranışı

- `--dry-run`, varsayılan olarak exec SecretRef denetimlerini atlar.
- Exec SecretRef'leri/sağlayıcıları içeren planlar, `--allow-exec` ayarlanmadığı sürece yazma modunda reddedilir.
- Exec içeren planları doğrularken/uygularken hem deneme çalıştırması hem de yazma komutlarında `--allow-exec` değerini geçirin.

## Çalışma zamanı ve denetim kapsamı notları

- Yalnızca ref içeren `auth-profiles.json` girdileri (`keyRef`/`tokenRef`), çalışma zamanı kimlik bilgisi çözümlemesine ve denetim kapsamına dahil edilir.
- `secrets apply`, desteklenen `openclaw.json` hedeflerini, desteklenen `auth-profiles.json` hedeflerini ve her biri varsayılan olarak etkin olan üç isteğe bağlı temizleme geçişini yazar: `scrubEnv` (geçerli durum ve etkin yapılandırma dizinlerindeki `.env` dosyalarından taşınmış düz metin değerlerini kaldırır), `scrubAuthProfilesForProviderTargets` (bir planın az önce taşıdığı sağlayıcılar için `auth-profiles.json` içindeki düz metin/kullanılmayan ref kalıntılarını temizler) ve `scrubLegacyAuthJson` (eski `auth.json` depolarından taşınmış `api_key` girdilerini kaldırır). İlgili geçişi atlamak için plandaki `options.scrubEnv`, `options.scrubAuthProfilesForProviderTargets`, `options.scrubLegacyAuthJson` değerlerinden herhangi birini `false` olarak ayarlayın.

## Operatör denetimleri

```bash
# Planı yazma işlemi olmadan doğrulayın
openclaw secrets apply --from /tmp/openclaw-secrets-plan.json --dry-run

# Ardından gerçekten uygulayın
openclaw secrets apply --from /tmp/openclaw-secrets-plan.json

# Exec içeren planlar için her iki modda da açıkça onay verin
openclaw secrets apply --from /tmp/openclaw-secrets-plan.json --dry-run --allow-exec
openclaw secrets apply --from /tmp/openclaw-secrets-plan.json --allow-exec
```

apply geçersiz bir hedef yolu iletisiyle başarısız olursa planı `openclaw secrets configure` ile yeniden oluşturun veya hedef yolunu yukarıdaki desteklenen yapılardan birine uygun hâle getirin.

## İlgili belgeler

- [Gizli Bilgi Yönetimi](/tr/gateway/secrets)
- [CLI `secrets`](/tr/cli/secrets)
- [SecretRef Kimlik Bilgisi Yüzeyi](/tr/reference/secretref-credential-surface)
- [Yapılandırma Referansı](/tr/gateway/configuration-reference)
