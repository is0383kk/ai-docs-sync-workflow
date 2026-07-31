---
read_when:
    - Mevcut bir Matrix kurulumunu yükseltme
    - Şifrelenmiş Matrix geçmişini ve cihaz durumunu taşıma
summary: OpenClaw'un önceki Matrix Plugin'ini yerinde nasıl yükselttiği; şifrelenmiş durum kurtarma sınırları ve manuel kurtarma adımları dâhil.
title: Matrix geçişi
x-i18n:
    generated_at: "2026-07-26T22:38:02Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 475c96914900a5597f37001264bd3d8f69a69dbd0600f2704c2a1be46924fac4
    source_path: channels/matrix-migration.md
    workflow: 16
---

Önceki genel kullanıma açık `matrix` plugininden mevcut uygulamaya yükseltin.

Çoğu kullanıcı için yükseltme mevcut yapıyı korur:

- plugin `@openclaw/matrix` olarak kalır
- kanal `matrix` olarak kalır
- yapılandırmanız `channels.matrix` altında kalır
- önbelleğe alınmış kimlik bilgileri, paylaşılan `state/openclaw.sqlite` plugin durumuna taşınır
- çalışma zamanı durumu `~/.openclaw/matrix/` altında kalır

Yapılandırma anahtarlarını yeniden adlandırmanız veya plugini yeni bir adla yeniden yüklemeniz gerekmez.
Kök `openclaw` paketi artık Matrix çalışma zamanı kodunu veya Matrix SDK
bağımlılıklarını paketlemez. `openclaw channels status`, Matrix'in yapılandırıldığını ancak
pluginin yüklü olmadığını gösteriyorsa `openclaw doctor --fix` veya
`openclaw plugins install @openclaw/matrix` komutunu çalıştırın; Matrix SDK paketlerini
kök OpenClaw paketine yüklemeyin.

## Geçişin otomatik olarak yaptıkları

Matrix geçişi, [`openclaw doctor --fix`](/tr/gateway/doctor) çalıştırıldığında gerçekleşir. Özel Matrix deposunun yanındaki dosya tabanlı yan dosyalar, istemci başlatma geri dönüş mekanizmasını korur; ancak kimlik bilgisi dosyalarının içe aktarımı yalnızca Doctor tarafından yapılır. Çalışma zamanı yalnızca standart SQLite kimlik bilgisi durumunu okur.

Doctor geçişi şunları kapsar:

- kullanımdan kaldırılmış `~/.openclaw/credentials/matrix/credentials*.json` dosyalarını arşivlemeden önce içe aktarma ve doğrulama
- aynı hesap seçimini ve `channels.matrix` yapılandırmasını koruma
- dosya tabanlı yan dosya durumunu (`bot-storage.json` eşitleme önbelleği, `recovery-key.json`, `legacy-crypto-migration.json`, IndexedDB anlık görüntüleri) Matrix SQLite durumuna aktarma; taşınan dosyalar `.migrated` son ekiyle arşivlenir
- erişim belirteci daha sonra değiştiğinde aynı Matrix hesabı, ana sunucu, kullanıcı ve cihaz için mevcut en eksiksiz belirteç karması depolama kökünü yeniden kullanma

## 2026.4'ten eski OpenClaw sürümlerinden yükseltme

2026.6 serisine kadarki sürümler, özgün düz tek depolu
Matrix düzenini de (`~/.openclaw/matrix/bot-storage.json` ile
`~/.openclaw/matrix/crypto/`) taşıyor ve eski rust kripto deposundan
şifrelenmiş durum kurtarmaya hazırlıyordu. Mevcut sürümler artık bu geçişi içermez.

Hâlâ düz düzeni kullanan bir kurulumu yükseltiyorsanız önce
bir 2026.6 sürümüne yükseltin, `openclaw doctor --fix` komutunu çalıştırın ve düz deponun
ve kurtarılabilir tüm oda anahtarlarının taşınması için gateway'i bir kez başlatın. Ardından
en son sürüme güncelleyin.

Önceki genel kullanıma açık Matrix plugini, Matrix oda anahtarı yedeklerini otomatik olarak **oluşturmuyordu**. Eski kurulumunuzda hiç yedeklenmemiş, yalnızca yerelde bulunan şifrelenmiş geçmiş varsa geçiş yolundan bağımsız olarak bazı eski şifrelenmiş iletiler yükseltmeden sonra okunamaz durumda kalabilir.

## Önerilen yükseltme akışı

1. OpenClaw ve Matrix pluginini normal şekilde güncelleyin.
2. Şunu çalıştırın:

   ```bash
   openclaw doctor --fix
   ```

3. Gateway'i başlatın veya yeniden başlatın.
4. Mevcut doğrulama ve yedekleme durumunu kontrol edin:

   ```bash
   openclaw matrix verify status
   openclaw matrix verify backup status
   ```

5. Onardığınız Matrix hesabının kurtarma anahtarını hesaba özel bir ortam değişkenine koyun. Tek bir varsayılan hesap için `MATRIX_RECOVERY_KEY` uygundur. Birden fazla hesap için hesap başına bir değişken kullanın; örneğin `MATRIX_RECOVERY_KEY_ASSISTANT`. Ayrıca komuta `--account assistant` ekleyin.

6. OpenClaw bir kurtarma anahtarının gerekli olduğunu bildirirse eşleşen hesap için komutu çalıştırın:

   ```bash
   printf '%s\n' "$MATRIX_RECOVERY_KEY" | openclaw matrix verify backup restore --recovery-key-stdin
   printf '%s\n' "$MATRIX_RECOVERY_KEY_ASSISTANT" | openclaw matrix verify backup restore --recovery-key-stdin --account assistant
   ```

7. Bu cihaz hâlâ doğrulanmamışsa eşleşen hesap için komutu çalıştırın:

   ```bash
   printf '%s\n' "$MATRIX_RECOVERY_KEY" | openclaw matrix verify device --recovery-key-stdin
   printf '%s\n' "$MATRIX_RECOVERY_KEY_ASSISTANT" | openclaw matrix verify device --recovery-key-stdin --account assistant
   ```

   Kurtarma anahtarı kabul edilir ve yedek kullanılabilir durumdaysa ancak `Cross-signing verified`
   hâlâ `no` ise başka bir Matrix istemcisinden öz doğrulamayı tamamlayın:

   ```bash
   openclaw matrix verify self
   ```

   İsteği başka bir Matrix istemcisinde kabul edin, emojileri veya ondalık sayıları karşılaştırın
   ve yalnızca eşleşiyorlarsa `yes` yazın. Komut, başarı bildirmeden önce tam Matrix
   kimlik güveninin kurulmasını bekler.

8. Kurtarılamayan eski geçmişi bilerek terk ediyor ve gelecekteki iletiler için yeni bir yedekleme temeli istiyorsanız şunu çalıştırın:

   ```bash
   openclaw matrix verify backup reset --yes
   ```

   Yalnızca eski kurtarma anahtarının yeni yedeğin kilidini artık açmaması gerekiyorsa `--rotate-recovery-key` ekleyin.

9. Henüz sunucu tarafında anahtar yedeği yoksa gelecekteki kurtarmalar için bir tane oluşturun:

   ```bash
   openclaw matrix verify bootstrap
   ```

## Yaygın iletiler ve anlamları

`Failed migrating legacy Matrix client storage: ...`

- Anlamı: Matrix istemci tarafı geri dönüş mekanizması dosya tabanlı yan dosya durumu buldu ancak SQLite'a içe aktarma başarısız oldu. OpenClaw, yeni bir depoyla sessizce başlamak yerine tamamlanan taşımaları geri alır ve bu geri dönüş işlemini iptal eder.
- Yapılması gereken: dosya sistemi izinlerini veya çakışmaları inceleyin, eski durumu olduğu gibi koruyun ve hatayı düzelttikten sonra yeniden deneyin.

`Matrix is installed from a custom path: ...`

- Anlamı: Matrix bir yol kurulumuna sabitlenmiştir; bu nedenle ana sürüm güncellemeleri onu varsayılan Matrix paketiyle otomatik olarak değiştirmez.
- Yapılması gereken: varsayılan Matrix pluginine dönmek istediğinizde `openclaw plugins install @openclaw/matrix` ile yeniden yükleyin.

`Matrix is installed from a custom path that no longer exists: ...`

- Anlamı: plugin kurulum kaydınız artık bulunmayan bir yerel yolu gösteriyor.
- Yapılması gereken: `openclaw plugins install @openclaw/matrix` ile veya bir depo çalışma kopyasından çalışıyorsanız `openclaw plugins install ./path/to/local/matrix-plugin` ile yeniden yükleyin. `openclaw doctor --fix`, eski Matrix plugin başvurularını da sizin için kaldırabilir.

### Manuel kurtarma iletileri

`openclaw matrix verify status` ve `openclaw matrix verify backup status`, oda anahtarı yedeği bu cihazda sağlıklı değilse bir `Backup issue:` satırıyla birlikte `Next steps:` yönlendirmesi yazdırır:

| Yedekleme sorunu                                                       | Anlamı                                             | Düzeltme                                                                                                                                  |
| --------------------------------------------------------------------- | -------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------- |
| `no room-key backup exists on the homeserver`                         | geri yüklenecek bir şey yok                        | oda anahtarı yedeği oluşturmak için `openclaw matrix verify bootstrap`                                                                    |
| `backup decryption key is not loaded on this device`                  | anahtar mevcut ancak burada etkin değil            | `openclaw matrix verify backup restore`; anahtar hâlâ yüklenemiyorsa kurtarma anahtarını `--recovery-key-stdin` üzerinden aktarın                   |
| `backup decryption key could not be loaded from secret storage (...)` | gizli veri deposu yüklenemedi veya desteklenmiyor  | kurtarma anahtarını aktarın: `printf '%s\n' "$MATRIX_RECOVERY_KEY" \| openclaw matrix verify backup restore --recovery-key-stdin`                                                       |
| `backup key mismatch (...)`                                           | depolanan anahtar etkin sunucu yedeğiyle eşleşmiyor | etkin sunucu yedekleme anahtarıyla `verify backup restore --recovery-key-stdin` komutunu yeniden çalıştırın veya yeni bir temel için `verify backup reset --yes` kullanın |
| `backup signature chain is not trusted by this device`                | cihaz henüz çapraz imzalama zincirine güvenmiyor   | `verify device --recovery-key-stdin`, ardından güven hâlâ eksikse başka bir doğrulanmış istemciden `verify self`                         |
| `backup exists but is not active on this device`                      | sunucu yedeği mevcut, yerel oturum etkin değil     | önce cihazı doğrulayın, ardından `openclaw matrix verify backup status` ile yeniden kontrol edin                                                 |
| `backup trust state could not be fully determined`                    | tanılama sonuçsuz kaldı                            | `openclaw matrix verify status --verbose`                                                                                                         |

Diğer kurtarma hataları:

`Matrix recovery key is required`

- Anlamı: gerekli olduğu hâlde bir kurtarma anahtarı sağlamadan kurtarma adımını denediniz.
- Yapılması gereken: komutu `--recovery-key-stdin` ile yeniden çalıştırın; örneğin `printf '%s\n' "$MATRIX_RECOVERY_KEY" | openclaw matrix verify device --recovery-key-stdin`.

`Invalid Matrix recovery key: ...`

- Anlamı: sağlanan anahtar ayrıştırılamadı veya beklenen biçimle eşleşmedi.
- Yapılması gereken: Matrix istemcinizdeki veya kurtarma anahtarı dışa aktarımındaki tam kurtarma anahtarıyla yeniden deneyin.

`Matrix recovery key was applied, but this device still lacks full Matrix identity trust.`

- Anlamı: kurtarma anahtarı kullanılabilir yedekleme malzemesinin kilidini açtı ancak Matrix bu cihaz için tam çapraz imzalama kimlik güvenini henüz kurmadı. Komut çıktısında `Recovery key accepted`, `Backup usable`, `Cross-signing verified` ve `Device verified by owner` değerlerini kontrol edin.
- Yapılması gereken: `openclaw matrix verify self` komutunu çalıştırın, isteği başka bir Matrix istemcisinde kabul edin, SAS'ı karşılaştırın ve yalnızca eşleştiğinde `yes` yazın. Yalnızca mevcut çapraz imzalama kimliğini bilerek değiştirmek istediğinizde `printf '%s\n' "$MATRIX_RECOVERY_KEY" | openclaw matrix verify bootstrap --recovery-key-stdin --force-reset-cross-signing` kullanın.

Kurtarılamayan eski şifrelenmiş geçmişi kaybetmeyi kabul ediyorsanız bunun yerine
`openclaw matrix verify backup reset --yes` ile mevcut yedekleme temelini sıfırlayabilirsiniz. Depolanan
yedekleme gizli değeri bozuksa bu sıfırlama, yeni yedekleme anahtarının yeniden başlatma sonrasında
doğru şekilde yüklenebilmesi için gizli veri deposunu da onarır.

## Şifrelenmiş geçmiş hâlâ geri gelmiyorsa

Bu kontrolleri sırayla çalıştırın:

```bash
openclaw matrix verify status --verbose
openclaw matrix verify backup status --verbose
printf '%s\n' "$MATRIX_RECOVERY_KEY" | openclaw matrix verify backup restore --recovery-key-stdin --verbose
```

Yedek başarıyla geri yüklenmesine rağmen bazı eski odalarda geçmiş hâlâ eksikse bu eksik anahtarlar muhtemelen önceki plugin tarafından hiç yedeklenmemiştir.

## Gelecekteki iletiler için yeni bir başlangıç yapmak istiyorsanız

Kurtarılamayan eski şifrelenmiş geçmişi kaybetmeyi kabul ediyor ve bundan sonrası için yalnızca temiz bir yedekleme temeli istiyorsanız bu komutları sırayla çalıştırın:

```bash
openclaw matrix verify backup reset --yes
openclaw matrix verify backup status --verbose
openclaw matrix verify status
```

Bundan sonra cihaz hâlâ doğrulanmamışsa SAS emojilerini veya ondalık kodlarını karşılaştırıp eşleştiklerini onaylayarak Matrix istemcinizden doğrulamayı tamamlayın.

## İlgili

- [Matrix](/tr/channels/matrix): kanal kurulumu ve yapılandırması.
- [Matrix anlık bildirim kuralları](/tr/channels/matrix-push-rules): bildirim yönlendirme.
- [Doctor](/tr/gateway/doctor): sistem durumu kontrolü ve otomatik geçiş tetikleyicisi.
- [Geçiş kılavuzu](/tr/install/migrating): tüm geçiş yolları (makine taşımaları, sistemler arası içe aktarımlar).
- [Pluginler](/tr/tools/plugin): plugin yükleme ve kaydetme.
