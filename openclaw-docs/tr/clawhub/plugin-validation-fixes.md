---
read_when:
    - clawhub package validate komutunu çalıştırdınız ve Plugin bulgularını düzeltmeniz gerekiyor
    - ClawHub, bir plugin paketinin yayımlanmasını reddetti veya bu konuda uyarı verdi
    - Yayın öncesinde plugin paketi meta verilerini güncelliyorsunuz
summary: Yayımlamadan önce ClawHub Plugin paketi doğrulama bulgularını düzeltin
title: Plugin doğrulama düzeltmeleri
x-i18n:
    generated_at: "2026-07-26T23:34:10Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 2cb869e41c9a9f1c0725f514f6b48095eb3838bf61aaf06c2474a18192f0e819
    source_path: clawhub/plugin-validation-fixes.md
    workflow: 16
---

# Plugin doğrulama düzeltmeleri

ClawHub, Plugin paketlerini yayımlanmadan önce doğrular ve ayrıca otomatik paket
taramalarından elde edilen bulguları gösterebilir. Bu sayfa, yazarın paket
meta verilerinde, manifestinde, SDK içe aktarımlarında veya yayımlanan
yapıtında düzeltebileceği, yazara yönelik bulguları kapsar.

Dahili Plugin Inspector kapsam bulgularını kapsamaz. Tam raporda yazarın
uygulayabileceği düzeltme yönergeleri bulunmayan tarayıcı bakım kodları varsa,
bunlar Plugin yazarlarına değil OpenClaw bakımcılarına yöneliktir.

Herhangi bir düzeltmeyi uyguladıktan sonra yeniden çalıştırın:

```bash
clawhub package validate <path-to-plugin>
```

## Yazara yönelik bulgular

| Kod                                     | Buradan başlayın                                                                                                             |
| --------------------------------------- | --------------------------------------------------------------------------------------------------------------------------- |
| `package-json-missing`                  | [Paket meta verilerini ekleyin](/tr/clawhub/plugin-validation-fixes#package-json-missing)                                          |
| `package-openclaw-metadata-missing`     | [Paketin openclaw bloğunu ekleyin](/tr/clawhub/plugin-validation-fixes#package-openclaw-metadata-missing)                           |
| `package-openclaw-entry-missing`        | [OpenClaw paket giriş noktalarını bildirin](/tr/clawhub/plugin-validation-fixes#package-openclaw-entry-missing)                     |
| `package-entrypoint-missing`            | [Bildirilen giriş noktasını yayımlayın](/tr/clawhub/plugin-validation-fixes#package-entrypoint-missing)                             |
| `package-install-metadata-incomplete`   | [Kurulum meta verilerini tamamlayın](/tr/clawhub/plugin-validation-fixes#package-install-metadata-incomplete)                       |
| `package-plugin-api-compat-missing`     | [Plugin API uyumluluğunu bildirin](/tr/clawhub/plugin-validation-fixes#package-plugin-api-compat-missing)                           |
| `package-min-host-version-drift`        | [Asgari ana makine sürümünü uyumlu hâle getirin](/tr/clawhub/plugin-validation-fixes#package-min-host-version-drift)                |
| `package-manifest-version-drift`        | [Paket ve manifest sürümlerini uyumlu hâle getirin](/tr/clawhub/plugin-validation-fixes#package-manifest-version-drift)             |
| `package-openclaw-unsupported-metadata` | [Desteklenmeyen OpenClaw paket meta verilerini kaldırın](/tr/clawhub/plugin-validation-fixes#package-openclaw-unsupported-metadata) |
| `package-npm-pack-unavailable`          | [npm yapıtını paketlenebilir hâle getirin](/tr/clawhub/plugin-validation-fixes#package-npm-pack-unavailable)                        |
| `package-npm-pack-entrypoint-missing`   | [Giriş noktalarını npm paketleme çıktısına dâhil edin](/tr/clawhub/plugin-validation-fixes#package-npm-pack-entrypoint-missing)      |
| `package-npm-pack-metadata-missing`     | [Meta verileri npm paketleme çıktısına dâhil edin](/tr/clawhub/plugin-validation-fixes#package-npm-pack-metadata-missing)            |
| `manifest-name-missing`                 | [Manifest görünen adı ekleyin](/tr/clawhub/plugin-validation-fixes#manifest-name-missing)                                          |
| `manifest-unknown-fields`               | [Desteklenmeyen manifest alanlarını kaldırın](/tr/clawhub/plugin-validation-fixes#manifest-unknown-fields)                          |
| `manifest-unknown-contracts`            | [Desteklenmeyen sözleşme anahtarlarını kaldırın](/tr/clawhub/plugin-validation-fixes#manifest-unknown-contracts)                    |
| `legacy-root-sdk-import`                | [Kök SDK içe aktarımlarını değiştirin](/tr/clawhub/plugin-validation-fixes#legacy-root-sdk-import)                                  |
| `reserved-sdk-import`                   | [Ayrılmış SDK içe aktarımlarını kaldırın](/tr/clawhub/plugin-validation-fixes#reserved-sdk-import)                                  |
| `sdk-load-session-store`                | [Tüm oturum deposuna erişimi değiştirin](/tr/clawhub/plugin-validation-fixes#sdk-load-session-store)                                |
| `sdk-session-store-write`               | [Tüm oturum deposuna yazma işlemlerini değiştirin](/tr/clawhub/plugin-validation-fixes#sdk-session-store-write)                     |
| `sdk-session-file-helper`               | [Oturum dosya yolu yardımcılarını değiştirin](/tr/clawhub/plugin-validation-fixes#sdk-session-file-helper)                          |
| `sdk-session-transcript-file-target`    | [Eski transkript dosyası hedeflerini değiştirin](/tr/clawhub/plugin-validation-fixes#sdk-session-transcript-file-target)            |
| `sdk-session-transcript-low-level`      | [Düşük seviyeli transkript yardımcılarını değiştirin](/tr/clawhub/plugin-validation-fixes#sdk-session-transcript-low-level)         |
| `legacy-before-agent-start`             | [before_agent_start öğesini değiştirin](/tr/clawhub/plugin-validation-fixes#legacy-before-agent-start)                              |
| `provider-auth-env-vars`                | [Sağlayıcı ortam değişkenlerini kurulum meta verilerine taşıyın](/tr/clawhub/plugin-validation-fixes#provider-auth-env-vars)        |
| `channel-env-vars`                      | [Kanal ortam değişkenlerini güncel meta verilere yansıtın](/tr/clawhub/plugin-validation-fixes#channel-env-vars)                    |
| `security-manifest-schema-unavailable`  | [Kullanılamayan güvenlik manifesti şema başvurularını kaldırın](/tr/clawhub/plugin-validation-fixes#security-manifest-schema-unavailable) |
| `unrecognized-security-manifest`        | [Desteklenmeyen güvenlik manifesti dosyalarını kaldırın](/tr/clawhub/plugin-validation-fixes#unrecognized-security-manifest)        |

## Paket meta verileri

### package-json-missing

Paket kökü `package.json` içermediğinden ClawHub npm paketini, sürümü,
giriş noktalarını veya OpenClaw meta verilerini tanımlayamaz.

- `name`, `version` ve `type` ile `package.json` ekleyin.
- Paket bir OpenClaw Plugin'i sağlıyorsa `openclaw` bloğu ekleyin.
- Asgari paket örneği için [Plugin oluşturma](/tr/plugins/building-plugins), paket ile
  manifest arasındaki ayrım için [Plugin manifesti](/tr/plugins/manifest#manifest-versus-packagejson)
  sayfasını kullanın.
- `clawhub package validate <path-to-plugin>` komutunu yeniden çalıştırın.

### package-openclaw-metadata-missing

Pakette `package.json` vardır, ancak OpenClaw paket meta verileri
bildirilmemiştir.

- `package.json#openclaw` ekleyin.
- `openclaw.extensions` veya `openclaw.runtimeExtensions` gibi giriş noktası meta verilerini
  ekleyin.
- Paket ClawHub aracılığıyla yayımlanacak veya kurulacaksa uyumluluk ve kurulum
  meta verilerini ekleyin.
- [Keşfi etkileyen package.json alanları](/tr/plugins/manifest#packagejson-fields-that-affect-discovery) bölümüne bakın.
- `clawhub package validate <path-to-plugin>` komutunu yeniden çalıştırın.

### package-openclaw-entry-missing

Paket meta verileri vardır, ancak bir OpenClaw çalışma zamanı giriş noktası
bildirilmemiştir.

- Yerel Plugin giriş noktaları için `openclaw.extensions` ekleyin.
- Yayımlanan paketin derlenmiş JavaScript'i yüklemesi gerekiyorsa
  `openclaw.runtimeExtensions` ekleyin.
- Tüm giriş noktası yollarını paket dizininin içinde tutun.
- [Plugin giriş noktaları](/tr/plugins/sdk-entrypoints) ve
  [keşfi etkileyen package.json alanları](/tr/plugins/manifest#packagejson-fields-that-affect-discovery)
  bölümlerine bakın.
- `clawhub package validate <path-to-plugin>` komutunu yeniden çalıştırın.

### package-entrypoint-missing

Paket bir OpenClaw giriş noktası bildiriyor, ancak başvurulan dosya doğrulanan
pakette bulunmuyor.

- `openclaw.extensions`, `openclaw.runtimeExtensions`,
  `openclaw.setupEntry` ve `openclaw.runtimeSetupEntry` içindeki her yolu denetleyin.
- Giriş noktası `dist` içine oluşturuluyorsa paketi derleyin.
- Giriş noktası taşındıysa meta verileri güncelleyin.
- [Plugin giriş noktaları](/tr/plugins/sdk-entrypoints) bölümüne bakın.
- `clawhub package validate <path-to-plugin>` komutunu yeniden çalıştırın.

### package-install-metadata-incomplete

ClawHub paketin nasıl kurulması veya güncellenmesi gerektiğini belirleyemiyor.

- `openclaw.install` alanını `clawhubSpec`,
  `npmSpec` veya `localPath` gibi desteklenen kurulum kaynağıyla doldurun.
- Birden fazla kurulum kaynağı varsa `openclaw.install.defaultChoice`
  değerini ayarlayın.
- Asgari OpenClaw ana makine sürümü için `openclaw.install.minHostVersion` kullanın.
- [Keşfi etkileyen package.json alanları](/tr/plugins/manifest#packagejson-fields-that-affect-discovery) bölümüne bakın.
- `clawhub package validate <path-to-plugin>` komutunu yeniden çalıştırın.

### package-plugin-api-compat-missing

Paket, desteklediği OpenClaw Plugin API aralığını bildirmiyor.

- `package.json` öğesine `openclaw.compat.pluginApi` ekleyin.
- Derleme ve test işlemlerini temel aldığınız OpenClaw Plugin API sürümünü veya
  asgari semver sürümünü kullanın.
- Bunu paket sürümünden ayrı tutun. Paket sürümü Plugin sürümünü, `openclaw.compat.pluginApi`
  ise ana makine API sözleşmesini tanımlar.
- [Keşfi etkileyen package.json alanları](/tr/plugins/manifest#packagejson-fields-that-affect-discovery) bölümüne bakın.
- `clawhub package validate <path-to-plugin>` komutunu yeniden çalıştırın.

### package-min-host-version-drift

Paketin asgari ana makine sürümü, paketin temel alınarak derlendiği OpenClaw
sürüm meta verileriyle eşleşmiyor.

- `openclaw.install.minHostVersion` öğesini denetleyin.
- Sürüm sırasında kullanılan OpenClaw sürümü gibi paketteki tüm OpenClaw derleme
  meta verilerini denetleyin.
- Asgari ana makine sürümünü, paketin gerçekten desteklediği ana makine sürüm
  aralığıyla uyumlu hâle getirin.
- [Keşfi etkileyen package.json alanları](/tr/plugins/manifest#packagejson-fields-that-affect-discovery) bölümüne bakın.
- `clawhub package validate <path-to-plugin>` komutunu yeniden çalıştırın.

### package-manifest-version-drift

Paket sürümü ile Plugin manifesti sürümü birbiriyle uyuşmuyor.

- Paket sürüm sürümü olarak `package.json#version` kullanmayı tercih edin.
- `openclaw.plugin.json` içinde `version` da varsa eşleşecek şekilde
  güncelleyin veya paket meta verileri esas alınıyorsa eski manifest sürümü
  meta verilerini kaldırın.
- Yayımlanan meta verileri değiştirdikten sonra yeni bir paket sürümü yayımlayın.
- [Plugin manifesti](/tr/plugins/manifest) bölümüne bakın.
- `clawhub package validate <path-to-plugin>` komutunu yeniden çalıştırın.

### package-openclaw-unsupported-metadata

`package.json#openclaw` bloğu, desteklenen OpenClaw paket meta verileri
arasında bulunmayan alanlar içeriyor.

- `openclaw.bundle` gibi desteklenmeyen alanları kaldırın.
- Yerel Plugin meta verilerini `openclaw.plugin.json` içinde tutun.
- Paket giriş noktalarını, uyumluluk, kurulum, yapılandırma ve katalog meta
  verilerini desteklenen `package.json#openclaw` alanlarında tutun.
- [Keşfi etkileyen package.json alanları](/tr/plugins/manifest#packagejson-fields-that-affect-discovery) bölümüne bakın.
- `clawhub package validate <path-to-plugin>` komutunu yeniden çalıştırın.

## Yayımlanan yapıt

### package-npm-pack-unavailable

Paket, ClawHub'ın inceleyeceği veya yayımlayacağı yapıt biçiminde
paketlenemiyor.

- Paket kökünden `npm pack --dry-run` komutunu çalıştırın.
- Paketlemenin başarısız olmasına neden olan geçersiz paket meta verilerini,
  bozuk yaşam döngüsü betiklerini veya dosya girdilerini düzeltin.
- Bu paket herkese açık yayımlama için tasarlanmışsa `private: true` öğesini kaldırın.
- `clawhub package validate <path-to-plugin>` komutunu yeniden çalıştırın.

### package-npm-pack-entrypoint-missing

Paket paketlenebiliyor, ancak paketlenmiş yapıt `package.json#openclaw` içinde bildirilen
giriş noktası dosyalarını içermiyor.

- `npm pack --dry-run` komutunu çalıştırın ve dâhil edilecek dosyaları inceleyin.
- Oluşturulan giriş noktalarını paketlemeden önce derleyin.
- Bildirilen giriş noktalarının dâhil edilmesi için `files`,
  `.npmignore` veya derleme çıktısını güncelleyin.
- [Plugin giriş noktaları](/tr/plugins/sdk-entrypoints) bölümüne bakın.
- `clawhub package validate <path-to-plugin>` komutunu yeniden çalıştırın.

### package-npm-pack-metadata-missing

Paketlenmiş yapıtta, kaynak paketinizde bulunan OpenClaw meta verileri
eksik.

- `npm pack --dry-run` komutunu çalıştırın ve dahil edilen meta veri dosyalarını inceleyin.
- `package.json` dosyasının paketlenmiş yapıda `openclaw` bloğunu içerdiğinden emin olun.
- Paket yerel bir OpenClaw plugin'i olduğunda `openclaw.plugin.json` dosyasının dahil
  edildiğinden emin olun.
- Paket meta verilerinin hariç tutulmaması için `files` veya `.npmignore` dosyasını güncelleyin.
- [Plugin oluşturma](/tr/plugins/building-plugins) bölümüne bakın.
- `clawhub package validate <path-to-plugin>` komutunu yeniden çalıştırın.

## Manifest meta verileri

### manifest-name-missing

Yerel plugin manifest'i bir görünen ad içermiyor.

- `openclaw.plugin.json` dosyasına boş olmayan bir `name` alanı ekleyin.
- `name` değerini insanlar tarafından okunabilir, `id` değerini ise kararlı makine kimliği olarak tutun.
- [Plugin manifest'i](/tr/plugins/manifest) bölümüne bakın.
- `clawhub package validate <path-to-plugin>` komutunu yeniden çalıştırın.

### manifest-unknown-fields

Plugin manifest'inde OpenClaw tarafından desteklenmeyen üst düzey alanlar bulunuyor.

- Her üst düzey alanı
  [manifest alanı referansı](/tr/plugins/manifest#top-level-field-reference) ile karşılaştırın.
- Özel alanları `openclaw.plugin.json` dosyasından kaldırın.
- Paket veya kurulum meta verilerini manifest yerine desteklenen `package.json#openclaw` alanlarına
  taşıyın.
- `clawhub package validate <path-to-plugin>` komutunu yeniden çalıştırın.

### manifest-unknown-contracts

Manifest, `contracts` içinde desteklenmeyen anahtarlar bildiriyor.

- `contracts` altındaki her anahtarı
  [sözleşmeler referansı](/tr/plugins/manifest#contracts-reference) ile karşılaştırın.
- Desteklenmeyen sözleşme anahtarlarını kaldırın.
- Çalışma zamanı davranışını plugin kayıt koduna taşıyın ve `contracts`
  kapsamını statik yetenek sahipliği meta verileriyle sınırlı tutun.
- `clawhub package validate <path-to-plugin>` komutunu yeniden çalıştırın.

## SDK ve uyumluluk geçişi

### legacy-root-sdk-import

Plugin, kullanımdan kaldırılmış kök SDK dışa aktarımından içe aktarıyor:
`openclaw/plugin-sdk`.

- Kök dışa aktarım içe aktarmalarını odaklanmış genel alt yol içe aktarmalarıyla değiştirin.
- `definePluginEntry` için `openclaw/plugin-sdk/plugin-entry` kullanın.
- Kanal giriş yardımcıları için `openclaw/plugin-sdk/channel-core` kullanın.
- Dar kapsamlı içe aktarmayı bulmak için [İçe aktarma kuralları](/tr/plugins/building-plugins#import-conventions) ve
  [Plugin SDK alt yolları](/tr/plugins/sdk-subpaths) bölümlerini kullanın.
- `clawhub package validate <path-to-plugin>` komutunu yeniden çalıştırın.

### reserved-sdk-import

Plugin, paketle birlikte gelen plugin'ler veya dahili uyumluluk için ayrılmış bir SDK yolundan
içe aktarıyor.

- Ayrılmış OpenClaw dahili SDK içe aktarmalarını belgelenmiş genel
  `openclaw/plugin-sdk/*` alt yollarıyla değiştirin.
- Davranış için genel bir SDK yoksa yardımcıyı paketinizin içinde tutun veya
  genel bir OpenClaw API'si talep edin.
- Desteklenen bir içe aktarma seçmek için [Plugin SDK alt yolları](/tr/plugins/sdk-subpaths) ve
  [SDK geçişi](/tr/plugins/sdk-migration) bölümlerini kullanın.
- `clawhub package validate <path-to-plugin>` komutunu yeniden çalıştırın.

### sdk-load-session-store

Plugin hâlâ kullanımdan kaldırılmış tüm oturum deposu yardımcısı
`loadSessionStore` kullanıyor.

- Oturum durumunu okurken `getSessionEntry(...)` veya `listSessionEntries(...)`
  kullanın.
- Oturum durumunu yazarken `patchSessionEntry(...)` veya `upsertSessionEntry(...)`
  kullanın.
- Tüm oturum deposu nesnesini yüklemekten, değiştirmekten ve kaydetmekten kaçının.
- `loadSessionStore(...)` yardımcısını yalnızca bildirdiğiniz uyumluluk aralığı bunu gerektiren
  eski OpenClaw sürümlerini hâlâ desteklerken koruyun.
- [Çalışma zamanı API'si](/tr/plugins/sdk-runtime#agent-session-state) ve
  [Plugin SDK alt yolları](/tr/plugins/sdk-subpaths) bölümlerine bakın.
- `clawhub package validate <path-to-plugin>` komutunu yeniden çalıştırın.

### sdk-session-store-write

Plugin hâlâ `saveSessionStore` veya `updateSessionStore` gibi kullanımdan kaldırılmış bir tüm oturum deposu yazma
yardımcısı kullanıyor.

- Mevcut bir oturum girdisindeki alanları güncellerken `patchSessionEntry(...)`
  kullanın.
- Bir oturum girdisini değiştirirken veya oluştururken `upsertSessionEntry(...)` kullanın.
- Tüm oturum deposu nesnesini yüklemekten, değiştirmekten ve kaydetmekten kaçının.
- Tüm depoya yazma yardımcılarını yalnızca bildirdiğiniz uyumluluk aralığı bunları gerektiren
  eski OpenClaw sürümlerini hâlâ desteklerken koruyun.
- [Çalışma zamanı API'si](/tr/plugins/sdk-runtime#agent-session-state) ve
  [Plugin SDK alt yolları](/tr/plugins/sdk-subpaths) bölümlerine bakın.
- `clawhub package validate <path-to-plugin>` komutunu yeniden çalıştırın.

### sdk-session-file-helper

Plugin hâlâ `resolveSessionFilePath` veya `resolveAndPersistSessionFile` gibi kullanımdan kaldırılmış oturum dosya yolu
yardımcılarını kullanıyor.

- Oturum meta verilerini aracı ve oturum kimliğine göre okumak için `getSessionEntry(...)`
  kullanın.
- Oturum meta verilerini kalıcı hâle getirmek için `patchSessionEntry(...)` veya `upsertSessionEntry(...)`
  kullanın.
- Kod bir transkript işlemi hazırlarken transkript kimliği veya hedef
  yardımcılarını kullanın.
- Eski transkript dosya yollarını kalıcı hâle getirmeyin veya bunlara bağımlı olmayın.
- [Çalışma zamanı API'si](/tr/plugins/sdk-runtime#agent-session-state) ve
  [Plugin SDK alt yolları](/tr/plugins/sdk-subpaths) bölümlerine bakın.
- `clawhub package validate <path-to-plugin>` komutunu yeniden çalıştırın.

### sdk-session-transcript-file-target

Plugin hâlâ kullanımdan kaldırılmış transkript dosyası hedef yardımcısı
`resolveSessionTranscriptLegacyFileTarget` kullanıyor.

- Kod yalnızca genel oturum kimliğine ihtiyaç duyduğunda
  `resolveSessionTranscriptIdentity(...)` kullanın.
- Kod yapılandırılmış bir transkript işlemi hedefine ihtiyaç duyduğunda
  `resolveSessionTranscriptTarget(...)` kullanın.
- Eski transkript dosyası hedeflerini doğrudan okumaktan veya oluşturmaktan kaçının.
- Eski yardımcıyı yalnızca bildirdiğiniz uyumluluk aralığı bunu gerektiren eski
  OpenClaw sürümlerini hâlâ desteklerken koruyun.
- [Çalışma zamanı API'si](/tr/plugins/sdk-runtime#agent-session-state) ve
  [Plugin SDK alt yolları](/tr/plugins/sdk-subpaths) bölümlerine bakın.
- `clawhub package validate <path-to-plugin>` komutunu yeniden çalıştırın.

### sdk-session-transcript-low-level

Plugin hâlâ `appendSessionTranscriptMessage` veya `emitSessionTranscriptUpdate` gibi kullanımdan kaldırılmış düşük düzeyli
transkript yardımcılarını kullanıyor.

- Transkripte ekleme işlemleri için `appendSessionTranscriptMessageByIdentity(...)` kullanın.
- Transkript güncelleme bildirimleri için `publishSessionTranscriptUpdateByIdentity(...)`
  kullanın.
- OpenClaw'ın doğru işlem sınırlarını ve kimlik işlemeyi uygulayabilmesi için
  yapılandırılmış transkript çalışma zamanı yüzeyini tercih edin.
- Düşük düzeyli transkript yardımcılarını yalnızca bildirdiğiniz uyumluluk aralığı bunları gerektiren
  eski OpenClaw sürümlerini hâlâ desteklerken koruyun.
- [Çalışma zamanı API'si](/tr/plugins/sdk-runtime#agent-session-state) ve
  [Plugin SDK alt yolları](/tr/plugins/sdk-subpaths) bölümlerine bakın.
- `clawhub package validate <path-to-plugin>` komutunu yeniden çalıştırın.

### legacy-before-agent-start

Plugin hâlâ eski `before_agent_start` kancasını kullanıyor.

- Model veya sağlayıcı geçersiz kılma işlemlerini `before_model_resolve` konumuna taşıyın.
- İstem veya bağlam değiştirme işlemlerini `before_prompt_build` konumuna taşıyın.
- `before_agent_start` kancasını yalnızca bildirdiğiniz uyumluluk aralığı bunu gerektiren eski
  OpenClaw sürümlerini hâlâ desteklerken koruyun.
- [Kancalar](/tr/plugins/hooks) ve
  [Plugin uyumluluğu](/tr/plugins/compatibility) bölümlerine bakın.
- `clawhub package validate <path-to-plugin>` komutunu yeniden çalıştırın.

### provider-auth-env-vars

Manifest hâlâ eski `providerAuthEnvVars` sağlayıcı kimlik doğrulama meta verilerini kullanıyor.

- Sağlayıcı ortam değişkeni meta verilerini `setup.providers[].envVars` içine yansıtın.
- `providerAuthEnvVars` değerini yalnızca desteklediğiniz OpenClaw aralığı buna hâlâ
  ihtiyaç duyarken uyumluluk meta verisi olarak koruyun.
- [Kurulum referansı](/tr/plugins/manifest#setup-reference) ve
  [SDK geçişi](/tr/plugins/sdk-migration) bölümlerine bakın.
- `clawhub package validate <path-to-plugin>` komutunu yeniden çalıştırın.

### channel-env-vars

Manifest, ClawHub'ın beklediği güncel kurulum veya yapılandırma meta verileri olmadan eski ya da
daha eski kanal ortam değişkeni meta verilerini kullanıyor.

- OpenClaw'ın kanal çalışma zamanını yüklemeden kurulum durumunu inceleyebilmesi için kanal ortam değişkeni meta verilerini
  bildirimsel tutun.
- Ortam tarafından yönlendirilen kanal kurulumunu plugin yapınızın kullandığı güncel kurulum, kanal yapılandırması veya
  paket kanal meta verilerine yansıtın.
- `channelEnvVars` değerini yalnızca desteklenen eski OpenClaw sürümleri bunu hâlâ
  gerektirirken uyumluluk meta verisi olarak koruyun.
- [Plugin manifest'i](/tr/plugins/manifest) ve
  [Kanal plugin'leri](/plugins/sdk-channel-plugins) bölümlerine bakın.
- `clawhub package validate <path-to-plugin>` komutunu yeniden çalıştırın.

## Güvenlik manifest'i

### security-manifest-schema-unavailable

Paket, ClawHub'ın kullanılabilir olarak tanımadığı bir şema referansıyla
`openclaw.security.json` dosyasını dağıtıyor.

- Şema URL'si yalnızca tavsiye amaçlıysa kaldırın.
- Belgelenmiş sürümlü bir şemayı yalnızca OpenClaw yayımladıktan sonra kullanın.
- `clawhub package validate <path-to-plugin>` komutunu yeniden çalıştırın.

### unrecognized-security-manifest

Paket, desteklenmeyen bir güvenlik manifest dosyası dağıtıyor.

- OpenClaw sürümlü bir güvenlik manifest şemasını ve ClawHub davranışını belgeleyene kadar
  `openclaw.security.json` dosyasını kaldırın.
- Manifest sözleşmesi mevcut olana kadar güvenliğe duyarlı davranışı genel paket belgelerinizde veya
  README dosyanızda belgeleyin.
- `clawhub package validate <path-to-plugin>` komutunu yeniden çalıştırın.

## İlgili

- [ClawHub CLI](/tr/clawhub/cli)
- [ClawHub'da yayımlama](/tr/clawhub/publishing)
- [Plugin oluşturma](/tr/plugins/building-plugins)
- [Plugin manifest'i](/tr/plugins/manifest)
- [Plugin giriş noktaları](/tr/plugins/sdk-entrypoints)
- [Plugin uyumluluğu](/tr/plugins/compatibility)
