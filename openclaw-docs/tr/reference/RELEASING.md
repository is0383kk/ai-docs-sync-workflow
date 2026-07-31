---
doc-schema-version: 1
read_when:
    - Herkese açık sürüm kanalı tanımları aranıyor
    - Sürüm doğrulamasını veya paket kabulünü çalıştırma
    - Sürüm adlandırması ve yayın sıklığı aranıyor
summary: Sürüm kanalları, operatör kontrol listesi, doğrulama kutuları, sürüm adlandırma ve yayın sıklığı
title: Sürüm politikası
x-i18n:
    generated_at: "2026-07-26T23:59:39Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: de2429f039bb42deabdcfe280b7d91afac3bae3dc24714203ab7a67672dcc10c
    source_path: reference/RELEASING.md
    workflow: 16
---

OpenClaw, kullanıcıya yönelik dört güncelleme kanalı sunar:

- stable: npm'de öne çıkarılan normal sürüm `latest`
- extended-stable: tamamlanan önceki ayın npm'deki `.33+` bakım hattı
  `extended-stable`
- beta: npm'deki ön sürüm etiketleri `beta`
- dev: `main` dalının ilerleyen en son ucu

Extended-stable, normal `latest` veya `main` seçicilerini değiştirmeden önceki ayın Gateway'ini, resmî npm pluginlerini ve
Docker imajlarını yayımlar.

Tideclaw alfa derlemeleri ayrı bir dâhilî ön sürüm hattıdır (npm dist-tag `alpha`); [NPM iş akışı girdileri](#npm-workflow-inputs) ve [Sürüm test kutuları](#release-test-boxes) altında ele alınır.

## Sürüm adlandırması

- Aylık Gateway extended-stable sürüm versiyonu: `YYYY.M.PATCH`; `PATCH >= 33` ile, git etiketi `vYYYY.M.PATCH`
- Günlük/normal nihai sürüm versiyonu: `YYYY.M.PATCH`; `PATCH < 33` ile, git etiketi `vYYYY.M.PATCH`
- Normal geri dönüş düzeltme sürümü versiyonu: `YYYY.M.PATCH-N`, git etiketi `vYYYY.M.PATCH-N`
- Beta ön sürüm versiyonu: `YYYY.M.PATCH-beta.N`, git etiketi `vYYYY.M.PATCH-beta.N`
- Alfa ön sürüm versiyonu: `YYYY.M.PATCH-alpha.N`, git etiketi `vYYYY.M.PATCH-alpha.N`
- Ay veya yama numarasını hiçbir zaman başına sıfır ekleyerek yazmayın
- `PATCH` bir takvim günü değil, sıralı aylık sürüm treni numarasıdır. Normal nihai ve beta sürümleri mevcut treni ilerletir; yalnızca alfa etiketleri beta/normal yama numarasını hiçbir zaman tüketmez veya ilerletmez. Bu nedenle bir beta ya da normal tren seçerken daha yüksek yama numaralarına sahip eski, yalnızca alfa etiketlerini yok sayın.
- Alfa/gecelik derlemeler, yayımlanmamış bir sonraki yama trenini kullanır ve yinelenen derlemelerde yalnızca `alpha.N` değerini artırır. Bu yamanın beta sürümü çıktığında yeni alfa derlemeleri sonraki yamaya geçer.
- npm sürümleri değiştirilemez: yayımlanmış bir etiketi hiçbir zaman silmeyin, yeniden yayımlamayın veya tekrar kullanmayın. Bunun yerine bir sonraki ön sürüm numarasını ya da aylık yamayı çıkarın.
- `latest` mevcut normal/günlük npm hattını izlemeye devam eder; `beta` mevcut beta kurulum hedefidir
- `extended-stable`, `33` yamasından başlayarak desteklenen önceki ay Gateway dağıtımı anlamına gelir; `34` ve sonraki yamalar bu aylık hattın bakım sürümleridir
- Normal nihai ve normal düzeltme sürümleri varsayılan olarak npm `beta` hedefine yayımlanır; sürüm operatörleri açıkça `latest` hedefini seçebilir veya daha sonra incelenmiş bir beta derlemesini öne çıkarabilir
- Gateway extended-stable; çekirdeği, npm'de yayımlanabilen tüm resmî pluginleri
  ve Docker imajlarını aynı kesin sürümle yayımlar; aşağıdaki özel iş akışına bakın.
- Her normal nihai sürüm; npm paketini, macOS uygulamasını, imzalı bağımsız Android APK'sını ve imzalı Windows Hub yükleyicilerini birlikte yayımlar. Beta sürümleri normalde önce npm/paket yolunu doğrular ve yayımlar; yerel uygulamaların derlenmesi/imzalanması/noter tasdiki/öne çıkarılması, açıkça istenmediği sürece normal nihai sürüme ayrılır.

## Sürüm sıklığı

- Sürümler önce beta olarak ilerler; stable ancak en son beta doğrulandıktan sonra gelir
- Bakımcılar, sürüm doğrulaması ve düzeltmelerin `main` üzerindeki yeni geliştirmeleri engellememesi için normalde mevcut `main` dalından oluşturulan bir `release/YYYY.M.PATCH` dalından sürüm çıkarır
- Bir beta etiketi gönderilmiş veya yayımlanmışsa ve düzeltilmesi gerekiyorsa bakımcılar eski etiketi silmek ya da yeniden oluşturmak yerine sonraki `-beta.N` etiketini çıkarır
- Ayrıntılı sürüm prosedürü, onaylar, kimlik bilgileri ve kurtarma notları yalnızca bakımcılara özeldir

## Aylık Gateway extended-stable yayını

Tamamlanan `YYYY.M` ayı için `extended-stable/YYYY.M.33` oluşturun ve
bu daldan `.33+` yayımlayın. Etiket, dal, çalışma kopyası, paket sürümü, ön kontrol ve
doğrulama aynı commit'i belirtmelidir. `.33` öncesinde korumalı `main`, yama
`33` değerinden düşük olan sonraki bir ayın nihai sürümünü içermelidir; sonraki bakım yamaları
uygun olmaya devam eder.

### Adayı hazırlama ve kararlı hâle getirme

Denetlenmemiş ana geliştirme hattı aralığını denetleyin, özel güvenlik çalışmalarını uzlaştırın, sınırlı bir geri taşıma kümesini onaylayın ve eş güdümlü tek bir PR'ı birleştirin. Standart
dalı doğrudan göndermeyin.

Standart dalda `YYYY.M.P` ayarlayın, `pnpm release:prep` çalıştırın ve
yayımlanabilir tüm resmî pluginlerde bu sürümü zorunlu kılın. Onaylanmış kayıt defterinden,
eş değer geri taşımalar için özgün birleştirilmiş `main` PR'larına atıfta bulunarak `### Highlights`,
`### Changes` ve `### Fixes` içeren eksiksiz bir `## YYYY.M.P` bölümü oluşturup commit edin.
Ön kontrol, eksik veya boş bir bölümü reddeder.

Mevcut ana dalın Docker sürüm kanalı biriminin tamamını taşıyın: iş akışı, öne çıkarıcı,
politika, paylaşılan sınıflandırıcı, testler ve iş akışı doğrulaması. GitHub, etiket
iş akışlarını etiketlenmiş commit'ten yükler; eksik bir kopya derlemeden sonra başarısız olabilir veya
normal takma adları taşıyabilir. Odaklanmış kontrolleri çalıştırın.

Tam dal ucu SHA'sını dondurun. Etiketlemeden önce tam npm baytlarını
ön kontrolden geçirin ve bu SHA'ya karşı Tam Sürüm Doğrulamasını çalıştırın:

```bash
RELEASE_SHA="$(git rev-parse HEAD)"

gh workflow run openclaw-npm-release.yml \
  --ref extended-stable/YYYY.M.33 \
  -f tag="$RELEASE_SHA" \
  -f preflight_only=true \
  -f npm_dist_tag=extended-stable

gh workflow run full-release-validation.yml \
  --ref extended-stable/YYYY.M.33 \
  -f ref=extended-stable/YYYY.M.33 \
  -f release_profile=stable
```

SHA biçimi yalnızca ön kontrol içindir. Doğrulamayı standart dalda çalıştırın; yayımlama,
iş akışı ref'ini, head/hedef SHA'sını, çalıştırma kimliğini ve denemeyi bağlar. Her iki kimliği ve
başarılı `run_attempt` değerini kaydedin; `release-ci/*` kanıtını reddedin.

Düzenleme yapmadan önce hataları sınıflandırın:

- Ürün: onaylanmış başka bir geri taşıma PR'ını birleştirin.
- Dondurulmuş hedef araçları: yalnızca eski ürünü değiştirmeden test eden en küçük uyumluluk onarımını
  geri taşıyın.
- Sağlayıcı, onay, çalıştırıcı veya hizmet: adayı değiştirmeyin ve
  sınırlı yeniden deneme yolunu kullanın.

Her dal değişikliği iki geçidi de geçersiz kılar. Geçtikten sonra ucun hâlâ
`RELEASE_SHA` değerine eşit olmasını zorunlu kılın, ardından imzalı `vYYYY.M.P` etiketini gönderin. Sonraki değişiklikler için
bir sonraki yama gerekir; etiketi hiçbir zaman taşımayın veya silmeyin. Etiketin gönderilmesi `Docker Release` sürecini başlatır.

### npm paketlerini yayımlama

npm'de yayımlanabilen tüm resmî pluginleri aynı SHA'dan yayımlayın ve
başarılı çalıştırma kimliğini kaydedin:

```bash
RELEASE_SHA="$(git rev-parse HEAD)"
gh workflow run plugin-npm-release.yml \
  --ref extended-stable/YYYY.M.33 \
  -f publish_scope=all-publishable \
  -f ref="$RELEASE_SHA" \
  -f npm_dist_tag=extended-stable
```

İş akışı, değişmemiş olanlar dâhil tüm `all-publishable` paketlerini kapsar
ve her kesin sürümü ve seçiciyi doğrular. Yeniden çalıştırmalar yayımlanmış sürümleri yeniden kullanır.

Ardından hazırlanmış çekirdek tarball'ını kaydedilmiş üç çalıştırma kimliğinin tümüyle yayımlayın:

```bash
gh workflow run openclaw-npm-release.yml \
  --ref extended-stable/YYYY.M.33 \
  -f tag=vYYYY.M.P \
  -f preflight_only=false \
  -f npm_dist_tag=extended-stable \
  -f preflight_run_id=<npm-preflight-run-id> \
  -f full_release_validation_run_id=<full-validation-run-id> \
  -f full_release_validation_run_attempt=<full-validation-run-attempt> \
  -f plugin_npm_run_id=<plugin-npm-run-id>
```

Yalnızca üretim dışı prova için ön kontrole ve yayımlamaya
`-f bypass_extended_stable_guard=true` ekleyin. Bu yalnızca ay korumasını atlar;
standart ref, SHA/etiket/sürüm eşitliği, kaynak kanıtı,
onay veya geri okuma kontrollerini asla atlamaz. Üretimde hiçbir zaman kullanmayın.

### Doğrulama ve kurtarma

Dondurulmuş daldan değil, ayrı ve temiz bir güncel `main` çalışma kopyasından şunu çalıştırın:

```bash
node --import tsx scripts/openclaw-npm-postpublish-verify.ts YYYY.M.P
npm view openclaw@YYYY.M.P version --userconfig "$(mktemp)"
npm view openclaw@extended-stable version --userconfig "$(mktemp)"
```

Standart dal için imzaları ve npm kaynak kanıtını; ayrıca yayımlama,
ön kontrol ve tarball özeti bağının sürüm SHA'sına bağlı olmasını zorunlu kılın. Her iki komut da
`YYYY.M.P` döndürmelidir. Hazırlanmış her çekirdek paketini ve `all-publishable`
resmî plugini kesin sürümü ve seçicisiyle doğrulayın.

Yalnızca kök seçici başarısız olursa iş akışı özetinde yazdırılan, oluşturulmuş
`npm dist-tag add openclaw@YYYY.M.P extended-stable` onarım komutunu kullanın.
Mevcut plugin veya diğer hazırlanmış çekirdek seçicilerini,
onaylanmış ve kimlik bilgilerinden yalıtılmış araçlarla onarın; OIDC kaynağı bunları değiştiremez.
Değiştirilemez bir sürümü hiçbir zaman yeniden yayımlamayın.

`Docker Release` işleminin GHCR ve Docker Hub'daki kesin varsayılan, slim, tarayıcı ve mimari
imajlarını; tasdikler ve platform sürümleri dâhil olmak üzere doğrulamasını zorunlu kılın. İşlem,
özet aracılığıyla yalnızca
`extended-stable`, `extended-stable-slim` ve `extended-stable-browser` değerlerini
ilerletmelidir; normal takma adlar değişmeden kalır ve otomatik geri alma reddedilir.

Takma ad onarımı için onay geçitli `Docker Channel Promotion` işlemini, etiketi kullanarak güncel
`main` dalından çalıştırın. Özet, tasdik ve platform kontrollerini yineler, açık bir geri almaya
izin verir ve imajları hiçbir zaman yeniden derlemez.

Slack, Discord ve Codex ilk belgelenmiş destek yüzeyleridir; bir
sürüm izin listesi değildir: npm'de yayımlanabilen tüm resmî pluginler yayımlanır. Beta/`latest`, GitHub Releases, ClawHub, yerel uygulamalar, mobil,
web sitesi ve özel dist-tag'ler yalnızca normal
kontrol listesinin sorumluluğundadır; bu Gateway yolu için bu adımları çalıştırmayın.

## Normal sürüm operatörü kontrol listesi

Bu kontrol listesi sürüm akışının herkese açık biçimidir. Özel kimlik bilgileri, imzalama, noter tasdiki, dist-tag kurtarma ve acil geri alma ayrıntıları yalnızca bakımcılara açık sürüm çalıştırma kitabında kalır.

1. Güncel `main` dalından başlayın: en son değişiklikleri çekin, hedef commit'in gönderildiğini doğrulayın ve `main` CI'ın dallanmak için yeterince yeşil olduğunu doğrulayın.
2. Bu commit'ten `release/YYYY.M.PATCH` oluşturun. Geri taşımalar isteğe bağlıdır; yalnızca operatörün seçtiği kümeyi uygulayın. Gerekli tüm sürüm konumlarını artırın, `pnpm release:prep` çalıştırın, sürüm düzeltmelerini ve gerekli ileri taşımaları tamamlayın ve `src/plugins/compat/registry.ts` ile `src/commands/doctor/shared/deprecation-compat.ts` öğelerini inceleyin.
3. Ürün açısından tamamlanmış, değişiklik günlüğü öncesi commit'i **Kod SHA'sı** olarak dondurun. Belirlenimci kaynak ön kontrolünü çalıştırın, ardından `node scripts/full-release-validation-at-sha.mjs --sha <code-sha> --target-ref release/YYYY.M.PATCH` kullanın. Bu işlem güvenilir iş akışı araçlarını sabitlerken tam Vitest, Docker, QA, paket ve performans matrisi tam Kod SHA'sını hedefler.
4. Düzenleme yapmadan önce hataları sınıflandırın. Bir ürün/kod hatası yeni bir Kod SHA'sı oluşturur ve bu SHA için yeşil tam doğrulama gerektirir. Bir iş akışı, test düzeneği, kimlik bilgisi, onay veya altyapı hatası kendi sorumluluk yüzeyinde onarılır ve aynı Kod SHA'sına karşı yeniden çalıştırılır.
5. Yalnızca Kod SHA'sı yeşil olduktan sonra, erişilebilen son yayımlanmış etiketten bu yana birleştirilen PR'lardan ve doğrudan commit'lerden en üstteki `CHANGELOG.md` bölümünü oluşturun. Girdileri kullanıcıya yönelik ve yinelenmeyen biçimde tutun. Ayrışmış bir yayımlanmış etiket veya sonraki ileri taşıma, daha önce yayımlanmış PR'ları yeniden ilişkilendirirse bunu açıkça `--shipped-ref` olarak geçirin.
6. Yalnızca `CHANGELOG.md` öğesini commit edin. Bu commit **Sürüm SHA'sıdır**. Kod SHA'sından Sürüm SHA'sına kadar olan eksiksiz fark tam olarak `CHANGELOG.md` olmalıdır; değişen başka bir yol sürümü 2. adıma döndürür.
7. Kanıtın yeniden kullanımı etkin olarak Sürüm SHA'sı için SHA'ya sabitlenmiş Tam Sürüm Doğrulamasını çalıştırın. Hafif üst iş akışı `changelog-only-release-v1` değerini kaydetmeli, yeşil Kod SHA'sını göstermeli ve hiçbir ürün alt hattını tetiklememelidir. Bu, ürün kanıtını yeniden kullanır; paket baytlarını yeniden kullanmaz.
8. Sürüm SHA'sına/etiketine karşı `preflight_only=true` ile `OpenClaw NPM Release` çalıştırın. Başarılı `preflight_run_id` değerini kaydedin. Bu işlem, nihai değişiklik günlüğünü içeren tam paket baytlarını derler ve denetler.
9. Sürüm SHA'sını etiketleyin, ardından ikisini de yeniden tetiklemek yerine başarılı Sürüm SHA'sı doğrulama üst iş akışı ve npm ön kontrolüyle aday yardımcısını çalıştırın:

   ```bash
   pnpm release:candidate -- \
     --tag vYYYY.M.PATCH-beta.N \
     --full-release-run <release-sha-validation-run-id> \
     --npm-preflight-run <preflight-run-id> \
     --skip-dispatch
   ```

   Kararlı sürüm için ayrıca `--windows-node-tag vX.Y.Z` parametresini geçirin. Yardımcı; sürüm notu kaynağını, npm ön kontrol baytlarını, Parallels kurulum/güncelleme kanıtını, Telegram paket kanıtını ve plugin yayımlama planlarını doğrular, ardından yayımlama komutunu yazdırır.

   `OpenClaw Release Publish`, seçilen veya yayımlanabilir tüm plugin paketlerini paralel olarak npm'e ve aynı kümedekileri ClawHub'a gönderir; ardından plugin'in npm'de yayımlanması başarıyla tamamlandığında, hazırlanmış OpenClaw npm ön kontrol yapıtını eşleşen dist-tag ile yükseltir. Sürüm checkout'u ürün/veri kökü olarak kalırken planlama ve son doğrulama, eski bir sürüm commit'inin güncelliğini yitirmiş sürüm araçlarını sessizce kullanamaması için tam olarak güvenilen iş akışı kaynağı checkout'undan yürütülür. Herhangi bir yayımlama alt süreci başlamadan önce tam GitHub sürüm gövdesini oluşturur ve önbelleğe alır. Eşleşen `CHANGELOG.md` bölümünün tamamı GitHub'ın 125,000 karakter sınırına ve oluşturucunun eşleşen 125,000 baytlık güvenlik tavanına sığdığında sayfa, başlığıyla birlikte tam olarak bu `## YYYY.M.PATCH` bölümünü içerir. Kaynak bölüm sığmadığında sayfa, gruplandırılmış editoryal notları aynen korur ve aşırı büyük katkı kaydını, etikete sabitlenmiş `CHANGELOG.md` içindeki tam kayda yönlendiren kararlı bir bağlantıyla değiştirir; kısmi kayıtlar ve kesilmiş madde işaretleri hiçbir zaman yayımlanmaz. İş akışı, `### Release verification` eklenmeden önce tam veya kompakt gövdeyi seçer; kanıt kuyruğu sınırı aşacaksa kurallı gövdeyi korur ve bunun yerine değiştirilemez ekli kanıta dayanır. npm'de `latest` olarak yayımlanan kararlı sürümler en son GitHub sürümü olurken npm'de `beta` olarak tutulan kararlı bakım sürümleri GitHub `latest=false` ile oluşturulur. İş akışı ayrıca sürüm sonrası olay müdahalesi için ön kontrol bağımlılık kanıtını, tam doğrulama manifestini ve yayımlama sonrası kayıt defteri doğrulama kanıtını GitHub sürümüne yükler. Alt çalışma kimliklerini hemen yazdırır, iş akışı token'ının onaylamasına izin verilen sürüm ortamı kapılarını otomatik olarak onaylar, başarısız alt işleri günlük sonlarıyla özetler, taslak GitHub sürüm sayfasını en başta oluşturur ve Windows ile Android varlıklarını OpenClaw npm yayınıyla eşzamanlı olarak yükseltir, bu aşamalar başarıyla tamamlandığında sürüm sayfasını ve bağımlılık kanıtını sonuçlandırır, OpenClaw npm'de yayımlanırken ClawHub'ı bekler, ardından güvenilen-main beta doğrulayıcısını çalıştırır ve GitHub sürümü, npm paketi, seçilen plugin npm paketleri, seçilen ClawHub paketleri, alt iş akışı çalışma kimlikleri ve isteğe bağlı NPM Telegram çalışma kimliği için yayımlama sonrası kanıtı yükler. ClawHub önyükleme doğrulayıcısı; tam güvenilen-main iş akışı yolunu ve SHA'yı, üretici ve terminal çalışma denemelerini, sürüm SHA'sını, istenen paket kümesini, değiştirilemez paket yapıtı demetini ve terminal kayıt defteri geri okuma yapıtını gerektirir; başarılı bir eski sürüm-ref çalışması kabul edilmez.

   Ardından yayımlanan `openclaw@YYYY.M.PATCH-beta.N` veya `openclaw@beta` paketine karşı yayımlama sonrası paket kabulünü çalıştırın. Gönderilmiş veya yayımlanmış bir ön sürüm düzeltme gerektiriyorsa eşleşen bir sonraki ön sürüm numarasını oluşturun; eskisini hiçbir zaman silmeyin veya yeniden yazmayın.

10. Başarısız bir yayımlama denemesinde, hata bir ürün veya değişiklik günlüğü kusurunu kanıtlamadığı sürece Sürüm SHA'sını değiştirmeyin. Başarılı değiştirilemez alt süreçleri ve yapıtları devam ettirin; zaten başarıyla tamamlanmış bir paket sürümünü hiçbir zaman yeniden oluşturmayın veya yayımlamayın.
11. Kararlı sürüm için yalnızca incelenmiş beta veya sürüm adayı gerekli doğrulama kanıtına sahip olduktan sonra devam edin. Kararlı npm yayını da başarılı ön kontrol yapıtını `preflight_run_id` aracılığıyla yeniden kullanarak `OpenClaw Release Publish` üzerinden gerçekleştirilir. Kararlı macOS sürüm hazırlığı ayrıca paketlenmiş `.zip`, `.dmg`, `.dSYM.zip` ve `main` üzerindeki güncellenmiş `appcast.xml` gerektirir; macOS yayımlama iş akışı, sürüm varlıkları doğrulandıktan sonra imzalı appcast'i herkese açık `main` hedefine otomatik olarak yayımlar veya dal koruması doğrudan göndermeyi engelliyorsa bir appcast PR'ı açar/günceller. Kararlı Windows Hub hazırlığı, OpenClaw GitHub sürümünde imzalı `OpenClawCompanion-Setup-x64.exe`, `OpenClawCompanion-Setup-arm64.exe` ve `OpenClawCompanion-SHA256SUMS.txt` varlıklarını gerektirir. Tam imzalı `openclaw/openclaw-windows-node` sürüm etiketini `windows_node_tag` olarak ve aday tarafından onaylanmış yükleyici özet eşlemesini `windows_node_installer_digests` olarak geçirin; `OpenClaw Release Publish` sürüm taslağını korur, `Windows Node Release` gönderimini yapar ve yayımlamadan önce üç varlığın tümünü doğrular.
12. Yayımlamadan sonra npm yayımlama sonrası doğrulayıcısını, yayımlama sonrası kanal kanıtına ihtiyaç duyulduğunda isteğe bağlı bağımsız yayımlanmış-npm Telegram E2E'sini ve gerektiğinde dist-tag yükseltmesini çalıştırın; oluşturulan GitHub sürüm sayfasını doğrulayın, sürüm duyurusu adımlarını çalıştırın ve ardından kararlı sürümü tamamlanmış olarak kabul etmeden önce [Kararlı main sonuçlandırması](#stable-main-closeout) işlemini tamamlayın.

## Kararlı main sonuçlandırması

`main` gerçek yayımlanmış sürüm durumunu taşımadıkça kararlı yayımlama tamamlanmış değildir.

1. Güncel en son `main` ile başlayın. `release/YYYY.M.PATCH` içeriğini buna göre denetleyin ve `main` içinde bulunmayan gerçek düzeltmeleri ileri taşıyın. Yalnızca sürüme özgü uyumluluk, test veya doğrulama bağdaştırıcılarını daha yeni `main` içine körü körüne birleştirmeyin.
2. Normal yol için `main` değerini yayımlanmış kararlı sürüme ayarlayın. Geç yapılan bir sonuçlandırma, daha sonraki bir kararlı OpenClaw CalVer sürümüne ilerledikten sonra `main` kullanabilir; yalnızca önceki sürümü sonuçlandırmak için başlamış bir sürüm sürecini eski sürüme düşürmeyin. Doğrulayıcı yine de tam yayımlanmış değişiklik günlüğü bölümünü ve appcast girdisini gerektirir ve gerçek `main` sürümünü ve SHA'sını kaydeder. Herhangi bir kök sürüm değişikliğinden sonra `pnpm release:prep`, ardından `pnpm deps:shrinkwrap:generate` çalıştırın.
3. `CHANGELOG.md` dosyasının `main` üzerindeki `## YYYY.M.PATCH` bölümünü etiketlenmiş sürüm dalıyla tam olarak eşleştirin. Mac sürümü bir tane yayımladıysa kararlı `appcast.xml` güncellemesini ekleyin.
4. Operatör ilgili sürüm sürecini açıkça başlatana kadar `main` içine `YYYY.M.PATCH+1`, bir beta sürümü veya boş bir gelecek değişiklik günlüğü bölümü eklemeyin.
5. `pnpm release:generated:check`, `pnpm deps:shrinkwrap:check` ve `OPENCLAW_TESTBOX=1 pnpm check:changed` çalıştırın. Gönderin, ardından kararlı sürümü tamamlanmış olarak kabul etmeden önce `origin/main` öğesinin yayımlanmış sürümü ve değişiklik günlüğünü içerdiğini doğrulayın.
6. Her özel geri alma tatbikatından sonra `RELEASE_ROLLBACK_DRILL_ID` ve `RELEASE_ROLLBACK_DRILL_DATE` depo değişkenlerini güncel tutun.

`OpenClaw Stable Main Closeout`, kararlı yayımlamadan sonra yayımlanmış sürümü, değişiklik günlüğünü ve appcast'i taşıyan `main` gönderiminden başlar. Yayımlanmış etiketi Tam Sürüm Doğrulaması ve Yayımlama çalışmalarıyla ilişkilendirmek için değiştirilemez yayımlama sonrası kanıtı okur; ardından kararlı main durumunu, sürümü, zorunlu kararlı bekleme süresini ve engelleyici performans kanıtını doğrular. GitHub sürümüne değiştirilemez bir sonuçlandırma manifesti ve sağlama toplamı ekler. Otomatik gönderim tetikleyicisi, değiştirilemez yayımlama sonrası kanıttan önceki eski sürümleri atlar ve bu atlamayı hiçbir zaman tamamlanmış sonuçlandırma olarak değerlendirmez.

Tam bir sonuçlandırma, hem varlıkları hem de eşleşen bir sağlama toplamını gerektirir. Kısmi bir manifest, aynı baytları yeniden oluşturmak için kaydedilmiş `main` SHA'sını ve geri alma tatbikatını yeniden oynatır, ardından eksik sağlama toplamını ekler; geçersiz bir çift veya manifestsiz bir sağlama toplamı engelleyici olmaya devam eder. Geri alma tatbikatı depo değişkenleri olmayan gönderim tetiklemeli bir çalışma, sonuçlandırmayı tamamlamadan atlanır; eksik veya 90 günden eski bir tatbikat kaydı da manuel, kanıta dayalı sonuçlandırmayı engellemeye devam eder. Özel kurtarma komutları yalnızca bakımcıya açık çalışma kılavuzunda kalır. Manuel gönderimi yalnızca kanıta dayalı kararlı sonuçlandırmayı onarmak veya yeniden oynatmak için kullanın.

Sürüm Yayımlama üst süreci yalnızca değiştirilemez npm/plugin kanıtı eklendikten sonra başarısız olduysa önce tüm kararlı platform varlıklarını onarın ve yayımlayın. Ardından bir bakımcı, `allow_failed_publish_recovery=true` ile sonuçlandırmayı manuel olarak gönderebilir; bu mod yalnızca tamamlanmış başarısız bir üst süreci kabul eder ve normal macOS/appcast kontrollerinin yanı sıra tam Android ve Windows varlık sözleşmelerini, GitHub SHA-256 özetlerini, sağlama toplamı doğrulamasını, Android kaynağını ve Authenticode kontrolleri ile aday tarafından onaylanmış özetleri yayımlanmış yükleyicilerle eşleşen, üst süreç tarafından gönderilmiş başarılı bir Windows yükseltmesini ek olarak gerektirir. Otomatik gönderim sonuçlandırması bu kurtarma modunu hiçbir zaman etkinleştirmez.

Eski bir geri dönüş düzeltme etiketi, yalnızca düzeltme etiketi temel kararlı etiketle aynı kaynak commit'ine çözümleniyorsa temel paket kanıtını yeniden kullanabilir. Android sürümü, temel etiketin doğrulanmış APK'sını yeniden kullanır ve düzeltme etiketi için kaynak kanıtı ekler. Farklı kaynak içeren bir düzeltme kendi paket kanıtını yayımlayıp doğrulamalı ve daha yüksek bir Android `versionCode` kullanmalıdır.

## Sürüm ön kontrolü

- Test TypeScript'inin daha hızlı yerel `pnpm check` kapısının dışında da kapsanması için sürüm ön kontrolünden önce `pnpm check:test-types` çalıştırın.
- Daha kapsamlı içe aktarma döngüsü ve mimari sınır kontrollerinin daha hızlı yerel kapının dışında başarılı olması için sürüm ön kontrolünden önce `pnpm check:architecture` çalıştırın.
- Paket doğrulama adımı için beklenen `dist/*` sürüm yapıtlarının ve Control UI paketinin mevcut olması amacıyla `pnpm release:check` öncesinde `pnpm build && pnpm ui:build` çalıştırın.
- Kök sürüm yükseltmesinden sonra ve etiketlemeden önce `pnpm release:prep` çalıştırın. Sürüm/yapılandırma/API değişikliğinden sonra sıkça farklılaşan tüm belirlenimci sürüm oluşturucularını çalıştırır: plugin sürümleri, npm shrinkwrap'ları, plugin envanteri, temel yapılandırma şeması, paketlenmiş kanal yapılandırma meta verileri, yapılandırma belgeleri temel çizgisi, plugin SDK dışa aktarımları, Plugin SDK API sözleşmesi manifesti ve Control UI yerel ayar paketleri. Ayrıca yerel uygulama çevirileri ve platform tarafından oluşturulan yerel ayar kaynakları kaynak envanteriyle eşleşene kadar engeller; geride kalmışlarsa Kod SHA'sını dondurmadan önce `Native App Locale Refresh` çalışmasını bekleyin veya gönderin. `pnpm release:check`, bu korumaları kontrol modunda yeniden çalıştırır (katı yerel ayar kapıları ve plugin SDK yüzey bütçesi dâhil) ve paket sürüm kontrollerini çalıştırmadan önce oluşturulan tüm farklılaşma hatalarını tek geçişte bildirir.
- Plugin sürüm eşitlemesi, yayımlanabilir `@openclaw/ai` çalışma zamanı paketini, resmî plugin paket sürümlerini ve mevcut `openclaw.compat.pluginApi` alt sınırlarını varsayılan olarak OpenClaw sürümüne günceller. Bu alanı yalnızca paket sürümünün bir kopyası olarak değil, plugin SDK/çalışma zamanı API alt sınırı olarak değerlendirin: eski OpenClaw ana makineleriyle kasıtlı olarak uyumlu kalan yalnızca plugin sürümlerinde alt sınırı desteklenen en eski ana makine API'sinde tutun ve bu seçimi plugin sürüm kanıtında belgeleyin.
- Tüm ön sürüm test kutularını tek bir giriş noktasından başlatmak için sürüm onayından önce manuel `Full Release Validation` iş akışını çalıştırın. Bir dalı, etiketi veya tam commit SHA'sını kabul eder; manuel `CI` gönderimini ve kurulum duman testi, paket kabulü, işletim sistemleri arası paket kontrolleri, QA Lab eşliği, Matrix ve Telegram hatları için `OpenClaw Release Checks` gönderimini yapar. Kararlı ve tam çalışmalar her zaman kapsamlı canlı/E2E ve Docker sürüm yolu bekleme testini içerir; `run_release_soak=true` açık bir beta bekleme testi için korunur. Paket Kabulü, aday doğrulaması sırasında kurallı paket Telegram E2E'sini sağlayarak eşzamanlı ikinci bir canlı yoklayıcıya duyulan ihtiyacı ortadan kaldırır.

  Sürüm tarball'unu yeniden oluşturmadan yayımlanmış npm paketini sürüm kontrolleri, Paket Kabulü ve paket Telegram E2E'si genelinde yeniden kullanmak için bir beta yayımladıktan sonra `release_package_spec` sağlayın. Yalnızca Telegram'ın sürüm doğrulamasının geri kalanından farklı bir yayımlanmış paket kullanması gerektiğinde `npm_telegram_package_spec` sağlayın. Paket Kabulünün sürüm paketi belirtiminden farklı bir yayımlanmış paket kullanması gerektiğinde `package_acceptance_package_spec` sağlayın. Sürüm kanıtı raporunun Telegram E2E'sini zorunlu kılmadan doğrulamanın yayımlanmış bir npm paketiyle eşleştiğini kanıtlaması gerektiğinde `evidence_package_spec` sağlayın.

  ```bash
  node scripts/full-release-validation-at-sha.mjs \
    --sha <code-sha> \
    --target-ref release/YYYY.M.PATCH
  ```

- Sürüm çalışması devam ederken bir paket adayı için yan kanal kanıtı istediğinizde manuel `Package Acceptance` iş akışını çalıştırın. `openclaw@beta`, `openclaw@latest` veya tam bir sürüm versiyonu için `source=npm`; mevcut `workflow_ref` düzeneğiyle güvenilir bir `package_ref` dalını/etiketini/SHA'sını paketlemek için `source=ref`; gerekli bir SHA-256 ve katı genel URL politikası bulunan genel bir HTTPS tar arşivi için `source=url`; gerekli `trusted_source_id` ve SHA-256'yı kullanan, adlandırılmış bir güvenilir kaynak politikası için `source=trusted-url`; başka bir GitHub Actions çalıştırması tarafından yüklenen bir tar arşivi için ise `source=artifact` kullanın.

  İş akışı adayı `package-under-test` olarak çözümler, bu tar arşivine karşı Docker E2E sürüm zamanlayıcısını yeniden kullanır ve `telegram_mode=mock-openai` veya `telegram_mode=live-frontier` ile aynı tar arşivine karşı Telegram QA çalıştırabilir. Seçilen Docker hatları `published-upgrade-survivor` öğesini içerdiğinde paket yapıtı adaydır ve yayımlanmış temel çizgiyi `published_upgrade_survivor_baseline` seçer. `update-restart-auth`, aday güncelleme komutunun yönetilen yeniden başlatma yolunu sınaması için aday paketi hem kurulu CLI hem de test edilen paket olarak kullanır.

  Örnek:

  ```bash
  gh workflow run package-acceptance.yml --ref main -f workflow_ref=main -f source=npm -f package_spec=openclaw@beta -f suite_profile=product -f published_upgrade_survivor_baseline=openclaw@2026.4.26 -f telegram_mode=mock-openai
  ```

  Yaygın profiller:
  - `smoke`: kurulum/kanal/agent, Gateway ağı ve yapılandırmayı yeniden yükleme hatları
  - `package`: OpenWebUI veya canlı ClawHub olmadan yapıta özgü paket/güncelleme/yeniden başlatma/plugin hatları
  - `product`: paket profiline ek olarak MCP kanalları, cron/alt agent temizliği, OpenAI web araması ve OpenWebUI
  - `full`: OpenWebUI içeren Docker sürüm yolu parçaları
  - `custom`: odaklı bir yeniden çalıştırma için tam `docker_lanes` seçimi

- Yalnızca sürüm adayı için belirlenimci normal CI kapsamına ihtiyacınız olduğunda manuel `CI` iş akışını doğrudan çalıştırın. Manuel CI tetiklemeleri, değişiklik kapsamlandırmasını atlar ve Linux Node parçalarını, paketlenmiş plugin parçalarını, plugin ve kanal sözleşmesi parçalarını, Node 22 uyumluluğunu, `check-*`, `check-additional-*`, derlenmiş yapıt duman kontrollerini, dokümantasyon kontrollerini, Python Skills'ı, Windows'u, macOS'i ve Control UI i18n hatlarını zorunlu kılar. Bağımsız manuel CI çalıştırmaları Android'i yalnızca `include_android=true` ile tetiklendiğinde çalıştırır; `Full Release Validation` bu girdiyi CI alt iş akışına geçirir.

  ```bash
  gh workflow run ci.yml --ref release/YYYY.M.PATCH -f include_android=true
  ```

- Sürüm telemetrisini doğrularken `pnpm qa:otel:smoke` çalıştırın. Yerel bir OTLP/HTTP alıcısı üzerinden QA-lab'i sınar ve Opik, Langfuse veya başka bir harici toplayıcı gerektirmeden iz, metrik ve günlük dışa aktarımının yanı sıra sınırlandırılmış iz özniteliklerini ve içerik/tanımlayıcı redaksiyonunu doğrular.
- Toplayıcı uyumluluğunu doğrularken `pnpm qa:otel:collector-smoke` çalıştırın. Yerel alıcı doğrulamalarından önce aynı QA-lab OTLP dışa aktarımını gerçek bir OpenTelemetry Collector Docker konteyneri üzerinden yönlendirir.
- Korumalı Prometheus kazımasını doğrularken `pnpm qa:prometheus:smoke` çalıştırın. QA-lab'i sınar, kimliği doğrulanmamış kazımaları reddeder ve sürüm açısından kritik metrik ailelerinin istem içeriği, ham tanımlayıcılar, kimlik doğrulama token'ları ve yerel yollar içermediğini doğrular.
- Kaynak çalışma kopyasındaki OpenTelemetry ve Prometheus duman hatlarını art arda çalıştırmak için `pnpm qa:observability:smoke` çalıştırın.
- Etiketlenmiş her sürümden önce `pnpm release:check` çalıştırın.
- `OpenClaw NPM Release` ön kontrolü, npm tar arşivini paketlemeden önce bağımlılık sürüm kanıtlarını oluşturur. npm güvenlik bildirimi güvenlik açığı geçidi sürümü engeller. Geçişli manifest riski, bağımlılık sahipliği/kurulum yüzeyi ve bağımlılık değişikliği raporları yalnızca sürüm kanıtıdır. Bağımlılık değişikliği raporu, sürüm adayını erişilebilir önceki sürüm etiketiyle karşılaştırır. Ön kontrol, bağımlılık kanıtlarını `openclaw-release-dependency-evidence-<tag>` olarak yükler ve ayrıca hazırlanmış npm ön kontrol yapıtının içinde `dependency-evidence/` altına gömer. Gerçek yayımlama yolu bu ön kontrol yapıtını yeniden kullanır, ardından aynı kanıtları GitHub sürümüne `openclaw-<version>-dependency-evidence.zip` olarak ekler.
- Etiket oluşturulduktan sonraki durum değiştiren yayımlama sırası için `OpenClaw Release Publish` çalıştırın. Normal beta ve kararlı sürüm yayımlamalarını güvenilir `main` üzerinden tetikleyin; sürüm etiketi yine tam hedef commit'i seçer ve `release/YYYY.M.PATCH` içine işaret edebilir. Tideclaw alfa yayımlamaları eşleşen alfa dallarında kalır. Başarılı OpenClaw npm `preflight_run_id`, başarılı `full_release_validation_run_id` ve tam `full_release_validation_run_attempt` değerlerini iletin; bilinçli olarak odaklı bir onarım çalıştırmıyorsanız varsayılan plugin yayımlama kapsamını `all-publishable` olarak tutun. İş akışı, çekirdek paketin dışsallaştırılmış plugin'lerinden önce yayımlanmaması için plugin npm yayımlamasını, plugin ClawHub yayımlamasını ve OpenClaw npm yayımlamasını sıralı hâle getirir; Windows ve Android tanıtımı ise taslak sürüm sayfasına karşı çekirdek npm yayımlamasıyla eş zamanlı çalışır. Yayımlama yeniden çalıştırmaları kaldığı yerden sürdürülebilir: zaten yayımlanmış bir çekirdek npm versiyonu, iş akışı kayıt defterindeki tar arşivinin etiketin ön kontrol yapıtıyla eşleştiğini kanıtladıktan sonra çekirdek tetiklemesini atlar; sürüm doğrulanmış yapıt sözleşmesini zaten içeriyorsa Windows/Android tanıtımı da atlanır, böylece yeniden deneme yalnızca başarısız aşamaları tekrarlar. Yalnızca plugin'e yönelik odaklı onarımlar `plugin_publish_scope=selected` ve boş olmayan bir plugin listesi gerektirir. Yalnızca plugin'e yönelik `all-publishable` çalıştırmaları eksiksiz, değişmez ön kontrol ve Tam Sürüm Doğrulaması kanıtı gerektirir; kısmi kanıt reddedilir.
- Kararlı `OpenClaw Release Publish`, eşleşen ön sürüm olmayan `openclaw/openclaw-windows-node` sürümü mevcut olduktan sonra tam bir `windows_node_tag` ve aday için onaylanmış `windows_node_installer_digests` eşlemesini gerektirir. Herhangi bir yayımlama alt iş akışını tetiklemeden önce bu kaynak sürümün yayımlanmış, ön sürüm olmayan, gerekli x64/ARM64 yükleyicilerini içeren ve hâlâ bu onaylı eşlemeyle uyuşan bir sürüm olduğunu doğrular. Ardından OpenClaw sürümü hâlâ taslakken, sabitlenmiş yükleyici özet eşlemesini değiştirmeden taşıyarak `Windows Node Release` iş akışını tetikler. Alt iş akışı, imzalı Windows Hub yükleyicilerini tam olarak bu etiketten indirir, bunları sabitlenmiş özetlerle eşleştirir, Authenticode imzalarının bir Windows çalıştırıcısında beklenen OpenClaw Foundation imzalayıcısını kullandığını doğrular, bir SHA-256 manifesti yazar ve yükleyicilerle manifesti kurallı OpenClaw GitHub sürümüne yükler; ardından tanıtılan yapıtları yeniden indirerek manifest üyeliğini ve karmaları doğrular. Üst iş akışı, yayımlamadan önce mevcut x64, ARM64 ve sağlama toplamı yapıt sözleşmesini doğrular. Doğrudan kurtarma, beklenen sözleşme yapıtlarını sabitlenmiş kaynak baytlarıyla değiştirmeden önce beklenmeyen `OpenClawCompanion-*` yapıt adlarını reddeder.

  `Windows Node Release` iş akışını yalnızca kurtarma amacıyla manuel olarak tetikleyin ve her zaman `latest` yerine tam bir etiket ile onaylanmış kaynak sürümden alınan açık `expected_installer_digests` JSON eşlemesini iletin. Web sitesi indirme bağlantıları, mevcut kararlı sürüm için tam OpenClaw sürüm yapıtı URL'lerini veya yalnızca GitHub'ın en son yönlendirmesinin aynı sürüme işaret ettiği doğrulandıktan sonra `releases/latest/download/...` değerini hedeflemelidir; yalnızca eşlik eden depo sürüm sayfasına bağlantı vermeyin.

- Sürüm kontrolleri artık ayrı bir manuel iş akışında çalışır: `OpenClaw Release Checks`. Ayrıca sürüm onayından önce QA Lab sahte eşlik düzeyini, Matrix sürüm profilini ve Telegram QA hattını çalıştırır. Canlı hatlar `qa-live-shared` ortamını kullanır; Telegram ayrıca Convex CI kimlik bilgisi kiralamalarını kullanır. Bakımı yapılan tüm Matrix senaryolarını çalıştırmak istediğinizde manuel `QA-Lab - All Lanes` iş akışını `matrix_profile=all` ile çalıştırın; iş akışı, eksiksiz doğrulamayı iş başına zaman aşımı sınırları içinde tutmak için bu seçimi aktarım, medya ve E2EE profillerine dağıtır.
- İşletim sistemleri arası kurulum ve yükseltme çalışma zamanı doğrulaması, yeniden kullanılabilir `.github/workflows/openclaw-cross-os-release-checks-reusable.yml` iş akışını doğrudan çağıran herkese açık `OpenClaw Release Checks` ve `Full Release Validation` kapsamındadır. Bu ayrım bilinçlidir: gerçek npm sürüm yolunu kısa, belirlenimci ve yapıt odaklı tutarken daha yavaş canlı kontroller kendi hatlarında kalır; böylece yayımlamayı geciktirmez veya engellemezler.
- Gizli bilgi içeren sürüm kontrolleri, iş akışı mantığının ve gizli bilgilerin denetim altında kalması için `Full Release Validation` üzerinden veya `main`/release iş akışı referansından tetiklenmelidir.
- `OpenClaw Release Checks`, çözümlenen commit bir OpenClaw dalından veya sürüm etiketinden erişilebilir olduğu sürece dal, etiket veya tam commit SHA'sı kabul eder.
- `OpenClaw NPM Release` yalnızca doğrulama amaçlı ön kontrolü, gönderilmiş bir etiket gerektirmeden iş akışı dalının mevcut 40 karakterlik tam commit SHA'sını da kabul eder. Bu SHA yolu yalnızca doğrulama içindir ve gerçek yayımlamaya yükseltilemez. SHA modunda iş akışı, yalnızca paket meta verisi kontrolü için `v<package.json version>` sentezler; gerçek yayımlama yine gerçek bir sürüm etiketi gerektirir.
- Her iki iş akışı da gerçek yayımlama ve yükseltme yolunu GitHub tarafından barındırılan çalıştırıcılarda tutarken, değişiklik yapmayan doğrulama yolu daha büyük Blacksmith Linux çalıştırıcılarını kullanabilir.
- Bu iş akışı, hem `OPENAI_API_KEY` hem de `ANTHROPIC_API_KEY` iş akışı gizli bilgilerini kullanarak `OPENCLAW_LIVE_TEST=1 OPENCLAW_LIVE_CACHE_TEST=1 pnpm test:live:cache` çalıştırır.
- npm sürüm ön kontrolü artık ayrı sürüm kontrolleri hattını beklemez.
- Yerel olarak bir sürüm adayı etiketlemeden önce `RELEASE_TAG=vYYYY.M.PATCH-beta.N pnpm release:fast-pretag-check` çalıştırın. Yardımcı, GitHub yayımlama iş akışı başlamadan önce onayı engelleyen yaygın hataları yakalayacak sırayla hızlı sürüm korumalarını, plugin npm/ClawHub sürüm kontrollerini, derlemeyi, kullanıcı arayüzü derlemesini ve `release:openclaw:npm:check` çalıştırır.
- Onaydan önce `RELEASE_TAG=vYYYY.M.PATCH node --import tsx scripts/openclaw-npm-release-check.ts` (veya eşleşen ön sürüm/düzeltme etiketi) çalıştırın.
- npm yayımlamasından sonra, yayımlanmış kayıt defteri kurulum yolunu yeni bir geçici önekte doğrulamak için `node --import tsx scripts/openclaw-npm-postpublish-verify.ts YYYY.M.PATCH` (veya eşleşen beta/düzeltme sürümü) çalıştırın.
- Bir beta yayımlamasından sonra, paylaşılan kiralık Telegram kimlik bilgisi havuzunu kullanarak yayımlanmış npm paketine karşı kurulu paket ilk katılımını, Telegram kurulumunu ve gerçek Telegram E2E'yi doğrulamak için `OPENCLAW_NPM_TELEGRAM_PACKAGE_SPEC=openclaw@YYYY.M.PATCH-beta.N OPENCLAW_NPM_TELEGRAM_CREDENTIAL_SOURCE=convex OPENCLAW_NPM_TELEGRAM_CREDENTIAL_ROLE=ci pnpm test:docker:npm-telegram-live` çalıştırın. Bakımcıların tek seferlik yerel çalıştırmaları Convex değişkenlerini atlayabilir ve üç `OPENCLAW_QA_TELEGRAM_*` ortam kimlik bilgisini doğrudan geçirebilir.
- Yayımlama sonrası beta duman testinin tamamını bir bakımcı makinesinden çalıştırmak için `pnpm release:beta-smoke -- --beta betaN` kullanın. Yardımcı; Parallels npm güncelleme/yeni hedef doğrulamasını çalıştırır, `NPM Telegram Beta E2E` tetikler, tam iş akışı çalıştırmasını yoklar, yapıtı indirir ve Telegram raporunu yazdırır.
- Bakımcılar, aynı yayımlama sonrası kontrolü GitHub Actions üzerinden manuel `NPM Telegram Beta E2E` iş akışıyla çalıştırabilir. Bilinçli olarak yalnızca manueldir ve her birleştirmede çalışmaz.
- Bakımcı sürüm otomasyonu, önce ön kontrol ardından yükseltme yaklaşımını kullanır:
  - Gerçek npm yayımlaması, başarılı bir npm `preflight_run_id` geçmelidir.
  - Normal beta ve kararlı sürüm yayımlama düzenlemesi ve ön kontrolü, tam hedef etikete karşı güvenilir `main` kullanır. Tideclaw alfa yayımlaması ve ön kontrolü, eşleşen alfa dalını kullanır.
  - Kararlı npm sürümleri varsayılan olarak `beta` kullanır; kararlı npm yayımlaması, iş akışı girdisi üzerinden açıkça `latest` hedefleyebilir.
  - Token tabanlı npm dist-tag değişikliği `openclaw/releases/.github/workflows/openclaw-npm-dist-tags.yml` içinde bulunur; çünkü kaynak depo yalnızca OIDC yayımlamasını sürdürürken `npm dist-tag add` hâlâ `NPM_TOKEN` gerektirir.
  - Herkese açık `macOS Release` yalnızca doğrulama içindir; bir etiket yalnızca bir sürüm dalında bulunuyor ancak iş akışı `main` üzerinden tetikleniyorsa `public_release_branch=release/YYYY.M.PATCH` ayarlayın.
  - Gerçek macOS yayımlaması, başarılı macOS `preflight_run_id` ve `validate_run_id` kontrollerini geçmelidir.
  - Gerçek yayımlama yolları, yeniden derlemek yerine hazırlanmış yapıtları yükseltir.
- `YYYY.M.PATCH-N` gibi kararlı düzeltme sürümlerinde yayımlama sonrası doğrulayıcı, sürüm düzeltmelerinin eski global kurulumları temel kararlı yükte sessizce bırakmaması için `YYYY.M.PATCH` sürümünden `YYYY.M.PATCH-N` sürümüne aynı geçici önek yükseltme yolunu da kontrol eder.
- npm sürüm ön kontrolü, tarball hem `dist/control-ui/index.html` hem de boş olmayan bir `dist/control-ui/assets/` yükü içermediği sürece kapalı biçimde başarısız olur; böylece boş bir tarayıcı panosunu tekrar dağıtmayız.
- Yayımlama sonrası doğrulama, yayımlanmış plugin giriş noktalarının ve paket meta verilerinin kurulu kayıt defteri düzeninde bulunduğunu da kontrol eder. Eksik plugin çalışma zamanı yükleriyle dağıtılan bir sürüm, yayımlama sonrası doğrulayıcıda başarısız olur ve `latest` seviyesine yükseltilemez.
- `pnpm test:install:smoke`, aday güncelleme tarball'ında npm pack `unpackedSize` bütçesini de uygular; böylece kurucu e2e, sürüm yayımlama yolundan önce yanlışlıkla oluşan paket şişmesini yakalar.
- Sürüm çalışması CI planlamasına, uzantı zamanlama manifestlerine veya uzantı test matrislerine dokunduysa, sürüm notlarının eski bir CI düzenini açıklamaması için onaydan önce planlayıcının sahip olduğu `plugin-prerelease-extension-shard` matris çıktılarını `.github/workflows/plugin-prerelease.yml` üzerinden yeniden oluşturup inceleyin.
- Kararlı macOS sürüm hazırlığı, güncelleyici yüzeylerini de kapsar: GitHub sürümü paketlenmiş `.zip`, `.dmg` ve `.dSYM.zip` ile sonuçlanmalıdır; `main` üzerindeki `appcast.xml`, yayımlamadan sonra yeni kararlı zip dosyasını göstermelidir (macOS yayımlama iş akışı bunu otomatik olarak commit eder veya doğrudan gönderme engellenirse bir appcast pull request'i açar); paketlenmiş uygulama hata ayıklama dışı bir paket kimliğini, boş olmayan bir Sparkle akış URL'sini ve söz konusu sürüm için standart Sparkle derleme alt sınırında veya üzerinde bir `CFBundleVersion` değerini korumalıdır.

## Sürüm test kutuları

`Full Release Validation`, operatörlerin tam ürün matrisini tek bir giriş noktasından başlatma yöntemidir. Her alt iş akışının, istenen commit test edilen aday olarak kalırken tek bir güvenilir `main` iş akışı SHA'sına sabitlenmiş geçici bir daldan çalışması için yardımcıyı kullanın:

```bash
pnpm ci:full-release \
  --sha <code-sha> \
  --target-ref release/YYYY.M.PATCH
```

Yardımcı, mevcut `origin/main` verisini getirir, `release-ci/<workflow-sha>-...` dalını bu güvenilir iş akışı commit'ine gönderir, alfa/beta paket sürümlerinden `beta` ve diğer durumlarda `stable` çıkarımını yapar, geçici daldan `Full Release Validation` iş akışını `ref=<target-sha>` ile tetikler, her alt iş akışının `headSha` değerinin sabitlenmiş üst iş akışı SHA'sıyla eşleştiğini doğrular ve ardından geçici dalı siler. Yeni bir çalıştırmayı zorlamak için `-f reuse_evidence=false`, geniş danışma taraması için `-f release_profile=full` veya mevcut `origin/main` üzerinden hâlâ erişilebilen eski bir commit'i sabitlemek için `--workflow-sha <trusted-main-sha>` geçirin. İş akışının kendisi hiçbir zaman depo referanslarına yazmaz. Bu, adaya araç commit'leri eklemeden yalnızca main dalındaki sürüm araçlarını kullanılabilir tutar ve yanlışlıkla daha yeni bir `main` alt çalıştırmasını doğrulamayı önler.

Kod SHA'sı yeşil olduktan sonra yalnızca `CHANGELOG.md` commit'ini oluşturun ve aynı yardımcıyı Sürüm SHA'sıyla çalıştırın:

```bash
pnpm ci:full-release \
  --sha <release-sha> \
  --target-ref release/YYYY.M.PATCH
```

İkinci üst iş akışı, ürün kanıtını yalnızca GitHub Sürüm SHA'sının Kod SHA'sından türediğini ve değiştirilen yolların tam kümesinin tam olarak `CHANGELOG.md` olduğunu kanıtladığında yeniden kullanır. `changelog-only-release-v1` kaydeder ve hiçbir ürün alt iş akışını tetiklemez. Tarball baytları değiştiği için npm ön kontrolü ve paket/kurulum kabulü yine Sürüm SHA'sında çalışır.

Yeni bir Kod SHA'sı için iş akışı hedefi çözümler, manuel `CI` iş akışını ve ardından `OpenClaw Release Checks` iş akışını tetikler. `OpenClaw Release Checks`; kurulum duman testini, işletim sistemleri arası sürüm kontrollerini, bekletme etkinleştirildiğinde canlı/E2E Docker sürüm yolu kapsamını, standart Telegram paket E2E'siyle Paket Kabulünü, QA Lab eşliğini, canlı Matrix'i ve canlı Telegram'ı dağıtımlı olarak çalıştırır. Tam/tümü çalıştırması, odaklı bir yeniden çalıştırma ayrı `Plugin Prerelease` alt iş akışını kasıtlı olarak atlamadığı sürece, yalnızca `Full Release Validation` özeti `normal_ci`, `plugin_prerelease` ve `release_checks` öğelerini başarılı gösterdiğinde kabul edilebilir. Bağımsız `npm-telegram` alt iş akışını yalnızca `release_package_spec` veya `npm_telegram_package_spec` ile yayımlanmış pakete yönelik odaklı yeniden çalıştırma için kullanın. Son doğrulayıcı özeti, her alt çalıştırma için en yavaş iş tablolarını içerir; böylece sürüm yöneticisi günlükleri indirmeden mevcut kritik yolu görebilir.

Ürün performansı alt iş akışı bu sürüm yolunda yalnızca yapıt üretir.
Üst iş akışı onu `publish_reports=false` ile tetikler ve yalnızca yapıt
koruması Clawgrit rapor yayımlayıcısının atlanmış durumda kaldığını
kanıtlamadıkça doğrulama reddedilir.

Tam aşama matrisi, tam iş akışı işi adları, kararlı ve tam profil farklılıkları, yapıtlar ve odaklı yeniden çalıştırma tanıtıcıları için [Tam sürüm doğrulaması](/tr/reference/full-release-validation) sayfasına bakın.

Alt iş akışları, `Full Release Validation` çalıştıran SHA'ya sabitlenmiş güvenilir referanstan tetiklenir. Her alt çalıştırma tam üst iş akışı SHA'sını kullanmalıdır. Sürüm kanıtı için ham `--ref main -f ref=<sha>` tetiklemelerini kullanmayın; `pnpm ci:full-release --sha <target-sha> --target-ref release/YYYY.M.PATCH` kullanın.

Canlı/sağlayıcı kapsamını seçmek için `release_profile` kullanın:

- `beta`: sürüm açısından kritik en hızlı OpenAI/çekirdek canlı ve Docker yolu
- `stable`: sürüm onayı için beta ile kararlı sağlayıcı/arka uç kapsamı
- `full`: kararlı ile geniş danışma sağlayıcı/medya kapsamı

Kararlı ve tam doğrulama, yükseltmeden önce her zaman kapsamlı canlı/E2E ve Docker sürüm yolu taramasını ve sınırlandırılmış yayımlanmış yükseltmeden sağ çıkma taramasını çalıştırır. Bir beta için aynı taramayı istemek üzere `run_release_soak=true` kullanın. Bu tarama, yinelenen temel sürümler kaldırılmış ve her temel sürüm kendi Docker çalıştırıcı işine bölünmüş olarak en son dört kararlı paketi, sabitlenmiş `2026.4.23` ve `2026.5.2` temel sürümlerini ve daha eski `2026.4.15` kapsamını içerir.

`OpenClaw Release Checks`, hedef referansı bir kez `release-package-under-test` olarak çözümlemek için güvenilir iş akışı referansını kullanır ve bekletme çalıştığında bu yapıtı işletim sistemleri arası kontrollerde, Paket Kabulünde ve sürüm yolu Docker kontrollerinde yeniden kullanır. Bu, paketle ilgili tüm kutuların aynı baytları kullanmasını sağlar ve tekrarlanan paket derlemelerini önler. Bir beta zaten npm üzerindeyse sürüm kontrollerinin dağıtılmış paketi bir kez indirmesi, derleme kaynak SHA'sını `dist/build-info.json` içinden çıkarması ve bu yapıtı işletim sistemleri arası kontroller, Paket Kabulü, sürüm yolu Docker ve paket Telegram hatlarında yeniden kullanması için `release_package_spec=openclaw@YYYY.M.PATCH-beta.N` ayarlayın.

İşletim sistemleri arası OpenAI kurulum duman testi, depo/kuruluş değişkeni ayarlandığında `OPENCLAW_CROSS_OS_OPENAI_MODEL`, aksi takdirde `openai/gpt-5.6-luna` kullanır; çünkü bu hat en yetenekli modeli karşılaştırmalı olarak değerlendirmek yerine paket kurulumunu, ilk katılımı, Gateway başlatmayı ve tek bir canlı ajan turunu doğrular. Daha geniş canlı sağlayıcı matrisi, modele özgü kapsamın yeri olmaya devam eder.

Sürüm aşamasına göre şu varyantları kullanın:

```bash
# Ürün açısından eksiksiz Code SHA'yı doğrulayın.
pnpm ci:full-release \
  --sha <code-sha> \
  --target-ref release/YYYY.M.PATCH

# Code SHA ürün kanıtını yeniden kullanarak yalnızca değişiklik günlüğünü içeren Release SHA'yı doğrulayın.
pnpm ci:full-release \
  --sha <release-sha> \
  --target-ref release/YYYY.M.PATCH

# Bir beta yayımladıktan sonra, yayımlanmış paket Telegram E2E'sini ekleyin.
pnpm ci:full-release \
  --sha <release-sha> \
  --target-ref release/YYYY.M.PATCH \
  -f release_package_spec=openclaw@YYYY.M.PATCH-beta.N \
  -f evidence_package_spec=openclaw@YYYY.M.PATCH-beta.N \
  -f npm_telegram_provider_mode=mock-openai
```

Odaklı bir düzeltmeden sonraki ilk yeniden çalıştırmada tam şemsiye iş akışını kullanmayın. Bir kutu başarısız olursa sonraki kanıt için başarısız alt iş akışını, işi, Docker hattını, paket profilini, model sağlayıcısını veya QA hattını kullanın. Tam şemsiye iş akışını yalnızca düzeltme paylaşılan sürüm düzenlemesini değiştirdiyse veya önceki tüm kutulara ait kanıtları geçersiz kıldıysa yeniden çalıştırın. Şemsiyenin son doğrulayıcısı kaydedilen alt iş akışı çalıştırma kimliklerini yeniden denetler; bu nedenle bir alt iş akışı başarıyla yeniden çalıştırıldıktan sonra yalnızca başarısız `Verify full validation` üst işini yeniden çalıştırın.

`rerun_group=all`; sürüm profili, geçerli bekletme ayarı ve doğrulama girdileri eşleştiğinde ve hedef SHA aynı olduğunda ya da yeni hedef, eksiksiz değişen yol kümesi tam olarak `CHANGELOG.md` olan bir alt öğe olduğunda önceki başarılı bir şemsiye çalıştırmasını yeniden kullanabilir. Tam hedefin yeniden kullanılması `exact-target-full-validation-v1` kaydını; doğrulama sonrası Release SHA ise `changelog-only-release-v1` kaydını oluşturur. İkincisi yalnızca ürün doğrulamasını yeniden kullanır. Npm ön kontrolü, paket baytları, sürüm notu kaynağı ve kurulum/güncelleme kabulü yine de Release SHA üzerinde çalıştırılmalıdır. Herhangi bir sürüm, kaynak, oluşturulan içerik, bağımlılık, paket veya iş akışına ait hedef değişikliği yeni bir Code SHA ve yeni bir tam doğrulama gerektirir. Aynı `release/*` referansı ve yeniden çalıştırma grubu için daha yeni şemsiye çalıştırmaları, devam edenlerin otomatik olarak yerini alır. Yeni bir tam çalıştırmayı zorlamak için `reuse_evidence=false` iletin.

Sınırlandırılmış kurtarma için şemsiyeye `rerun_group` iletin. `all` gerçek sürüm adayı çalıştırmasıdır; `ci` yalnızca normal CI alt iş akışını, `plugin-prerelease` yalnızca sürüme özel plugin alt iş akışını, `release-checks` her sürüm kutusunu çalıştırır; daha dar sürüm grupları ise `install-smoke`, `cross-os`, `live-e2e`, `package`, `qa`, `qa-parity`, `qa-live` ve `npm-telegram` şeklindedir. Odaklı `npm-telegram` yeniden çalıştırmaları `release_package_spec` veya `npm_telegram_package_spec` gerektirir; tam/tüm çalıştırmalar Package Acceptance içindeki standart paket Telegram E2E'sini kullanır. Odaklı işletim sistemleri arası yeniden çalıştırmalara `cross_os_suite_filter=windows/packaged-upgrade` veya başka bir işletim sistemi/paket filtresi eklenebilir. QA sürüm denetimi hataları, çekirdek çalışma zamanı çifti hattındaki OpenClaw dinamik araç sapması dâhil olmak üzere normal sürüm doğrulamasını engeller. Tideclaw alfa çalıştırmaları, paket güvenliği dışındaki sürüm denetimi hatlarını yine de bilgilendirici olarak değerlendirebilir. `release_profile=beta` ile `Run repo/live E2E validation` canlı sağlayıcı paketleri bilgilendiricidir (uyarı oluşturur, engellemez); kararlı ve tam profiller bunları engelleyici olarak tutar. `live_suite_filter` açıkça Discord, WhatsApp veya Slack gibi geçitli bir QA canlı hattı istediğinde eşleşen `OPENCLAW_RELEASE_QA_*_LIVE_CI_ENABLED` depo değişkeni etkinleştirilmelidir; aksi takdirde hat sessizce atlanmak yerine girdi yakalama başarısız olur.

### Vitest

Vitest kutusu, manuel `CI` alt iş akışıdır. Manuel CI, değişiklik kapsamını kasıtlı olarak atlar ve sürüm adayı için normal test grafiğini zorunlu kılar: Linux Node parçaları, paketlenmiş plugin parçaları, plugin ve kanal sözleşmesi parçaları, Node 22 uyumluluğu, `check-*`, `check-additional-*`, derlenmiş yapıt duman denetimleri, belge denetimleri, Python skills, Windows, macOS ve Control UI i18n. Şemsiye `include_android=true` ilettiği için kutuyu `Full Release Validation` çalıştırdığında Android dâhil edilir; bağımsız manuel CI, Android kapsamı için `include_android=true` gerektirir.

"Kaynak ağacı tam normal test paketini geçti mi?" sorusunu yanıtlamak için bu kutuyu kullanın. Bu, sürüm yolu ürün doğrulamasıyla aynı değildir. Saklanacak kanıtlar:

- `Full Release Validation` özeti, başlatılan `CI` çalıştırma URL'sini göstermelidir
- `CI` çalıştırması tam hedef SHA üzerinde başarılı olmalıdır
- gerilemeler incelenirken CI işlerindeki başarısız veya yavaş parça adları
- bir çalıştırma performans analizi gerektirdiğinde `.artifacts/vitest-shard-timings.json` gibi Vitest zamanlama yapıtları

Manuel CI'ı doğrudan yalnızca sürüm deterministik normal CI gerektiriyor ancak Docker, QA Lab, canlı, işletim sistemleri arası veya paket kutularını gerektirmiyorsa çalıştırın. Android içermeyen doğrudan CI için ilk komutu kullanın. Doğrudan sürüm adayı CI'ın Android'i kapsaması gerektiğinde `include_android=true` ekleyin:

```bash
gh workflow run ci.yml --ref main -f target_ref=release/YYYY.M.PATCH
gh workflow run ci.yml --ref main -f target_ref=release/YYYY.M.PATCH -f include_android=true
```

### Docker

Docker kutusu, `OpenClaw Release Checks` ile `openclaw-live-and-e2e-checks-reusable.yml` ve ayrıca sürüm modundaki `install-smoke` iş akışında bulunur. Sürüm adayını yalnızca kaynak düzeyindeki testlerle değil, paketlenmiş Docker ortamları üzerinden doğrular.

Sürüm Docker kapsamı şunları içerir:

- yavaş Bun genel kurulum duman denetimi etkinleştirilmiş tam kurulum duman denetimi
- QR, kök/gateway ve yükleyici/Bun duman işleri ayrı kurulum duman parçaları olarak çalışırken, hedef SHA'ya göre kök Dockerfile duman görüntüsünün hazırlanması/yeniden kullanılması
- depo E2E hatları
- sürüm yolu Docker parçaları: `core`, `package-update-openai`, `package-update-anthropic`, `package-update-core`, `plugins-runtime-plugins`, `plugins-runtime-services`, `plugins-runtime-install-a` ile `plugins-runtime-install-h` ve `openwebui`
- istendiğinde özel büyük diskli bir çalıştırıcıda OpenWebUI kapsamı
- bölünmüş paketlenmiş plugin kurma/kaldırma hatları: `bundled-plugin-install-uninstall-0` ile `bundled-plugin-install-uninstall-23`
- sürüm denetimleri canlı paketleri içerdiğinde canlı/E2E sağlayıcı paketleri ve Docker canlı model kapsamı

Yeniden çalıştırmadan önce Docker yapıtlarını kullanın. Sürüm yolu zamanlayıcısı; hat günlükleri, `summary.json`, `failures.json`, aşama zamanlamaları, zamanlayıcı planı JSON'u ve yeniden çalıştırma komutlarıyla birlikte `.artifacts/docker-tests/` yükler. Odaklı kurtarma için tüm sürüm parçalarını yeniden çalıştırmak yerine yeniden kullanılabilir canlı/E2E iş akışında `docker_lanes=<lane[,lane]>` kullanın. Oluşturulan yeniden çalıştırma komutları, mevcut olduğunda önceki `package_artifact_run_id` ve hazırlanmış Docker görüntüsü girdilerini içerir; böylece başarısız bir hat aynı tarball ve GHCR görüntülerini yeniden kullanabilir.

### QA Lab

QA Lab kutusu da `OpenClaw Release Checks` kapsamındadır. Vitest ve Docker paket mekaniklerinden ayrı olarak, aracı davranış ve kanal düzeyindeki sürüm geçididir.

Sürüm QA Lab kapsamı şunları içerir:

- aracı eşlik paketi kullanılarak OpenAI aday hattını `anthropic/claude-opus-4-8` temel çizgisiyle karşılaştıran sahte eşlik hattı
- `qa-live-shared` ortamını kullanan Matrix canlı bağdaştırıcı sürüm profili
- Convex CI kimlik bilgisi kiralamalarını kullanan canlı Telegram QA hattı
- sürüm telemetrisi açık yerel kanıt gerektirdiğinde `pnpm qa:otel:smoke`, `pnpm qa:otel:collector-smoke`, `pnpm qa:prometheus:smoke` veya `pnpm qa:observability:smoke`

"Sürüm QA senaryolarında ve canlı kanal akışlarında doğru davranıyor mu?" sorusunu yanıtlamak için bu kutuyu kullanın. Sürümü onaylarken eşlik, Matrix ve Telegram hatlarının yapıt URL'lerini saklayın. Tam Matrix kapsamı, varsayılan sürüm açısından kritik hat yerine manuel parçalı bir QA-Lab çalıştırması olarak kullanılabilir durumda kalır.

### Paket

Paket kutusu, kurulabilir ürün geçididir. `Package Acceptance` ve `scripts/resolve-openclaw-package-candidate.mjs` çözümleyicisi tarafından desteklenir. Çözümleyici, bir adayı Docker E2E tarafından tüketilen `package-under-test` tarball'ına normalleştirir, paket envanterini doğrular, paket sürümü ile SHA-256 değerini kaydeder ve iş akışı test düzeneği referansını paket kaynak referansından ayrı tutar.

Desteklenen aday kaynakları:

- `source=npm`: `openclaw@beta`, `openclaw@latest` veya tam bir OpenClaw sürümü
- `source=ref`: seçilen `workflow_ref` test düzeneğiyle güvenilir bir `package_ref` dalını, etiketini veya tam commit SHA'sını paketleme
- `source=url`: gerekli `package_sha256` ile genel bir HTTPS `.tgz` indirme; URL kimlik bilgileri, varsayılan olmayan HTTPS bağlantı noktaları, özel/dâhilî/özel kullanımlı ana bilgisayar adları veya çözümlenen adresler ve güvenli olmayan yönlendirmeler reddedilir
- `source=trusted-url`: `.github/package-trusted-sources.json` içindeki adlandırılmış bir ilkeden gerekli `package_sha256` ve `trusted_source_id` ile bir HTTPS `.tgz` indirme; `source=url` için girdi düzeyinde özel ağ atlatması eklemek yerine bakımcıya ait kurumsal yansılar veya özel paket depoları için bunu kullanın
- `source=artifact`: başka bir GitHub Actions çalıştırmasının yüklediği `.tgz` öğesini yeniden kullanma

`OpenClaw Release Checks`; `source=artifact`, hazırlanmış sürüm paketi yapıtı, `suite_profile=custom`, `docker_lanes=doctor-switch update-channel-switch skill-install update-corrupt-plugin upgrade-survivor published-upgrade-survivor root-managed-vps-upgrade update-restart-auth plugins-offline plugin-update plugin-binding-command-escape` ve `telegram_mode=mock-openai` ile Package Acceptance'ı çalıştırır. Package Acceptance; geçiş, güncelleme, kök tarafından yönetilen VPS yükseltmesi, yapılandırılmış kimlik doğrulama güncellemesi sonrası yeniden başlatma, canlı ClawHub skill kurulumu, eski plugin bağımlılıklarını temizleme, çevrimdışı plugin sabitleri, plugin güncelleme, plugin komut bağlama kaçışı sağlamlaştırması ve Telegram paket QA işlemlerini aynı çözümlenmiş tarball üzerinde tutar. Engelleyici sürüm denetimleri varsayılan olarak yayımlanmış en son paket temel çizgisini kullanır; `run_release_soak=true`, `release_profile=stable` veya `release_profile=full` içeren beta profili, yayımlanmış yükseltmeden sağ çıkma taramasını `last-stable-4` ile sabitlenmiş `2026.4.23`, `2026.5.2` ve `2026.4.15` temel çizgilerine, `reported-issues` senaryolarıyla genişletir. Zaten yayımlanmış bir aday için `source=npm`, yayımlamadan önce SHA destekli yerel npm tarball'ı için `source=ref`, bakımcıya ait kurumsal/özel yansı için `source=trusted-url` veya başka bir GitHub Actions çalıştırmasının yüklediği hazırlanmış tarball için `source=artifact` ile Package Acceptance'ı kullanın.

Bu, daha önce Parallels gerektiren paket/güncelleme kapsamının çoğu için GitHub'a özgü alternatiftir. İşletim sistemine özgü ilk katılım, yükleyici ve platform davranışı açısından işletim sistemleri arası sürüm denetimleri önemini korur; ancak paket/güncelleme ürün doğrulamasında Package Acceptance tercih edilmelidir.

Güncelleme ve plugin doğrulamasına yönelik standart denetim listesi [Güncellemeleri ve pluginleri test etme](/tr/help/testing-updates-plugins) sayfasındadır. Bir plugin kurulumu/güncellemesini, doctor temizliğini veya yayımlanmış paket geçişi değişikliğini hangi yerel, Docker, Package Acceptance ya da sürüm denetimi hattının kanıtladığına karar verirken bunu kullanın. Her kararlı `2026.4.23+` paketinden kapsamlı yayımlanmış güncelleme geçişi, Full Release CI'ın parçası değil, ayrı bir manuel `Update Migration` iş akışıdır.

Eski paket kabul esnekliği kasıtlı olarak zamanla sınırlandırılmıştır. `2026.4.25` dâhil olmak üzere bu sürüme kadar olan paketler, npm'de zaten yayımlanmış meta veri eksiklikleri için uyumluluk yolunu kullanabilir: tarball'da eksik özel QA envanter girdileri, eksik `gateway install --wrapper`, tarball'dan türetilmiş git sabitinde eksik yama dosyaları, eksik kalıcı `update.channel`, eski plugin kurulum kaydı konumları, eksik pazar yeri kurulum kaydı kalıcılığı ve `plugins update` sırasında yapılandırma meta verisi geçişi. Yayımlanmış `2026.4.26` paketi, daha önce dağıtılmış yerel derleme meta verisi damga dosyaları için uyarı verebilir. Daha sonraki paketler modern paket sözleşmelerini karşılamalıdır; aynı eksiklikler sürüm doğrulamasını başarısız kılar.

Sürüm sorusu gerçek bir kurulabilir paketle ilgili olduğunda daha geniş Package Acceptance profilleri kullanın:

```bash
gh workflow run package-acceptance.yml \
  --ref main \
  -f workflow_ref=main \
  -f source=npm \
  -f package_spec=openclaw@beta \
  -f suite_profile=product \
  -f published_upgrade_survivor_baseline=openclaw@2026.4.26
```

Yaygın paket profilleri:

- `smoke`: hızlı paket kurulum/kanal/ajan, Gateway ağı ve yapılandırmayı yeniden yükleme hatları
- `package`: kurulum/güncelleme/yeniden başlatma/plugin paketi sözleşmeleri ve canlı ClawHub skill kurulum kanıtı; bu, sürüm denetiminin varsayılanıdır
- `product`: `package` ile birlikte MCP kanalları, cron/alt ajan temizliği, OpenAI web araması ve OpenWebUI
- `full`: OpenWebUI içeren Docker sürüm yolu parçaları
- `custom`: odaklı yeniden çalıştırmalar için tam `docker_lanes` listesi

Paket adayı Telegram kanıtı için Package Acceptance üzerinde `telegram_mode=mock-openai` veya `telegram_mode=live-frontier` seçeneğini etkinleştirin. İş akışı, çözümlenen `package-under-test` tarball dosyasını Telegram hattına aktarır; bağımsız Telegram iş akışı, yayın sonrası denetimler için yayımlanmış bir npm belirtimini kabul etmeye devam eder.

## Normal sürüm yayımlama otomasyonu

Beta, `latest`, plugin, GitHub Release ve platform yayını için
`OpenClaw Release Publish`, normal değişiklik yapan giriş noktasıdır. Aylık
`.33+` Gateway extended-stable yolu bu orkestratörü kullanmaz. Normal
iş akışı, güvenilir yayımcı iş akışlarını sürümün gerektirdiği sırayla
orkestre eder:

1. Sürüm etiketini teslim alın ve commit SHA değerini çözümleyin.
2. Etikete `main` veya `release/*` üzerinden (ya da alpha ön sürümleri için bir Tideclaw alpha dalından) erişilebildiğini doğrulayın.
3. `pnpm plugins:sync:check` öğesini çalıştırın.
4. `Plugin NPM Release` öğesini `publish_scope=all-publishable` ve `ref=<release-sha>` ile tetikleyin.
5. `Plugin ClawHub Release` öğesini aynı kapsam ve SHA ile tetikleyin.
6. Kaydedilen `full_release_validation_run_id` ve tam çalıştırma denemesi doğrulandıktan sonra `OpenClaw NPM Release` öğesini sürüm etiketi, npm dist-tag ve kaydedilmiş `preflight_run_id` ile tetikleyin.
7. Kararlı sürümler için GitHub sürümünü taslak olarak oluşturun veya güncelleyin, `Windows Node Release` öğesini açık `windows_node_tag` ve aday tarafından onaylanmış `windows_node_installer_digests` ile tetikleyin ve standart Windows yükleyici/sağlama toplamı varlıklarını doğrulayın. Ayrıca tam etikete ait imzalı APK'yı, sağlama toplamını ve köken bilgisini oluşturmak için `Android Release` öğesini tetikleyin. Taslağı yayımlamadan önce her iki yerel varlık sözleşmesini de doğrulayın.

Beta yayımlama örneği:

```bash
gh workflow run openclaw-release-publish.yml \
  --ref main \
  -f tag=vYYYY.M.PATCH-beta.N \
  -f preflight_run_id=<successful-openclaw-npm-preflight-run-id> \
  -f full_release_validation_run_id=<successful-full-release-validation-run-id> \
  -f full_release_validation_run_attempt=<successful-full-release-validation-run-attempt> \
  -f npm_dist_tag=beta
```

Varsayılan beta dist-tag değerine kararlı sürüm yayımlama:

```bash
gh workflow run openclaw-release-publish.yml \
  --ref main \
  -f tag=vYYYY.M.PATCH \
  -f windows_node_tag=vX.Y.Z \
  -f windows_node_installer_digests='{"OpenClawCompanion-Setup-x64.exe":"sha256:<approved-x64-sha256>","OpenClawCompanion-Setup-arm64.exe":"sha256:<approved-arm64-sha256>"}' \
  -f preflight_run_id=<successful-openclaw-npm-preflight-run-id> \
  -f full_release_validation_run_id=<successful-full-release-validation-run-id> \
  -f full_release_validation_run_attempt=<successful-full-release-validation-run-attempt> \
  -f npm_dist_tag=beta
```

Doğrudan `latest` sürümüne kararlı yükseltme açıkça belirtilmelidir:

```bash
gh workflow run openclaw-release-publish.yml \
  --ref main \
  -f tag=vYYYY.M.PATCH \
  -f windows_node_tag=vX.Y.Z \
  -f windows_node_installer_digests='{"OpenClawCompanion-Setup-x64.exe":"sha256:<approved-x64-sha256>","OpenClawCompanion-Setup-arm64.exe":"sha256:<approved-arm64-sha256>"}' \
  -f preflight_run_id=<successful-openclaw-npm-preflight-run-id> \
  -f full_release_validation_run_id=<successful-full-release-validation-run-id> \
  -f full_release_validation_run_attempt=<successful-full-release-validation-run-attempt> \
  -f npm_dist_tag=latest
```

Alt düzey `Plugin NPM Release` ve `Plugin ClawHub Release` iş akışlarını yalnızca odaklı onarım veya yeniden yayımlama çalışmaları için kullanın. `OpenClaw Release Publish`, `publish_openclaw_npm=true` olduğunda `plugin_publish_scope=selected` öğesini reddeder; böylece çekirdek paket, `@openclaw/diffs-language-pack` dahil yayımlanabilir tüm resmî pluginler olmadan yayımlanamaz. Seçili bir plugin onarımı için `publish_openclaw_npm=false` değerini `plugin_publish_scope=selected` ve `plugins=@openclaw/name` ile ayarlayın veya alt iş akışını doğrudan tetikleyin.

İlk yayımlama ClawHub önyüklemesi istisnadır: `Plugin ClawHub New` öğesini
güvenilir `main` üzerinden tetikleyin ve hedef sürümün tam SHA değerini `ref` aracılığıyla geçirin.
Önyükleme iş akışının kendisini hiçbir zaman sürüm etiketi veya dalından çalıştırmayın:

```bash
gh workflow run plugin-clawhub-new.yml \
  --ref main \
  -f plugins=@openclaw/name \
  -f ref=<full-40-character-release-sha> \
  -f pretag_validation=true \
  -f dry_run=true
```

Etiket öncesi doğrulama `dry_run=true` gerektirir, sürüm etiketi ve üst çalıştırma
girdilerini reddeder ve yalnızca `main` veya `release/*` üzerinden erişilebilen tam bir hedefi kabul eder.
ClawHub kimlik bilgilerini yüklemez, paket baytlarını yayımlamaz veya güvenilir
yayımcı yapılandırmasını değiştirmez. İş akışı yine de canlı kayıt defteri planını çözümler,
hedefi yalnızca secretsız bir işte teslim alıp paketler, kilitli ClawHub araç zincirini
hazırlar ve sürüm etiketi mevcut olmadan önce değişmez yapıyı ve paket
slug/kimliğini doğrular. `clawhub-plugin-bootstrap` ortamını yalnızca secretsız paketleme işleri
tamamlandıktan sonra onaylayın; bu korumalı doğrulama işi kimlik bilgileri veya değişiklik komutları içermez.

Onaylanmış bir deneme çalıştırması veya etiketleme sonrasında gerçek önyükleme, tam
sürüm etiketinin yanı sıra üst `OpenClaw Release Publish` çalıştırma kimliğini, denemesini ve
dalını içermelidir. Üst öğe, kendi iş akışı SHA değerini ve `Plugin ClawHub New` için ayrı bir tam güvenilir
`main` SHA değerini doğrular; alt çalıştırma ve korumalı
ortam onaylarının tümü bu onaylanmış alt SHA değeriyle eşleşmelidir. Sürüm etiketi,
her yayımlama denemesi ve güvenilir yayımcı değişikliği öncesinde yeniden denetlenir.

Paketleme işi;
adı, Actions yapı kimliği/özeti, üretici çalıştırması/denemesi, hedef SHA değeri ve paket başına tarball SHA-256/boyutu
doğrulama ve korumalı işlere taşınan tek bir değişmez yapı
yükler. Korumalı iş yalnızca güvenilir `main`
araçlarını teslim alır, yapı demetini GitHub API üzerinden doğrular, tam yapı kimliğine
göre indirir, her tarball dosyasının hash değerini yeniden hesaplar ve sabitlenmiş CLI'nin USTAR standartlaştırma kurallarıyla yerel TAR yollarını ve
paket kimliğini doğrular. Ardından her aday, kayıt defteri
araması veya kimlik doğrulamasından önce dönen, sabitlenmiş CLI yayımlama deneme çalıştırmasından geçer. Kimlik bilgisi işi ön filtresi, sıkıştırılmış ClawPack dosyalarını
120 MiB, toplam dosya yükünü 50 MiB, genişletilmiş TAR verisini 64 MiB ve
TAR girdi sayısını 10,000 ile sınırlar. Mevcut paketlerin güvenilir yayımcı onarımı yalnızca
yapılandırma olarak kalır; ancak yine de hedefi paketler ve güvenilir yayımcı
yapılandırmasını değiştirmeden önce istenen etiketin yanı sıra kayıt defterindeki baytların ve meta verilerin tam eşitliğini gerektirir.
Yayımlama sonrası doğrulama, ClawHub yapısını indirir ve
aynı SHA-256 ile boyutu gerektirir. Başarısız işleri yeniden çalıştırarak yapılan kurtarma, yalnızca tam üretici işi
başarıyla tamamlandıysa önceki bir denemenin paket yapısını yeniden kullanabilir.
Nihai kanıt ayrıca kilitli ClawHub sürümünü, kilit
SHA-256 değerini ve npm bütünlüğünü bağlar. Eşleşmeme durumunda yeni bir paket sürümü gerekir.

## NPM iş akışı girdileri

`OpenClaw NPM Release` şu operatör denetimli girdileri kabul eder:

- `tag`: `v2026.4.2`, `v2026.4.2-1`, `v2026.4.2-beta.1` veya `v2026.4.2-alpha.1` gibi gerekli sürüm etiketi; `preflight_only=true` olduğunda, yalnızca doğrulama amaçlı ön kontrol için mevcut tam 40 karakterli iş akışı dalı commit SHA değeri de olabilir
- `preflight_only`: yalnızca doğrulama/derleme/paketleme için `true`, gerçek yayımlama yolu için `false`
- `preflight_run_id`: mevcut başarılı ön kontrol çalıştırma kimliği; iş akışının tarball dosyasını yeniden oluşturmak yerine hazırlanmış olanı yeniden kullanması için gerçek yayımlama yolunda gereklidir
- `full_release_validation_run_id`: bu etiket/SHA için başarılı `Full Release Validation` çalıştırma kimliği; gerçek yayımlama için gereklidir. Beta yayımlamaları yalnızca ön kontrol ile bir uyarı eşliğinde ilerleyebilir, ancak kararlı/`latest` yükseltmesi yine de bunu gerektirir.
- `full_release_validation_run_attempt`: `full_release_validation_run_id` ile eşleştirilmiş tam pozitif çalıştırma denemesi; yeniden çalıştırmaların yayımlama sırasında yetkilendirme kanıtını değiştirememesi için çalıştırma kimliği verildiğinde zorunludur.
- `release_publish_run_id`: onaylanmış `OpenClaw Release Publish` çalıştırma kimliği; bu iş akışı söz konusu üst öğe tarafından tetiklendiğinde gereklidir (bot aktörünün gerçek yayımlama çağrıları)
- `plugin_npm_run_id`: başarılı tam HEAD `Plugin NPM Release` çalıştırma kimliği; gerçek bir `extended-stable` çekirdek yayımlaması için gereklidir
- `npm_dist_tag`: yayımlama yolunun npm hedef etiketi; `alpha`, `beta`, `latest` veya `extended-stable` değerlerini kabul eder ve varsayılan olarak `beta` kullanır. Nihai yama `33` ve sonrakiler `extended-stable` kullanmalıdır; varsayılan olarak `extended-stable` önceki yamaları reddeder ve nihai olmayan etiketleri her zaman reddeder.
- `bypass_extended_stable_guard`: yalnızca test amaçlı boolean, varsayılan `false`; `npm_dist_tag=extended-stable` ile sürüm kimliği, yapı, onay ve geri okuma denetimlerini korurken aylık extended-stable uygunluğunu atlar.

`Plugin NPM Release`, mevcut sürüm
davranışı için `npm_dist_tag=default` veya korumalı aylık yol için `npm_dist_tag=extended-stable` kabul eder. extended-stable seçeneği `publish_scope=all-publishable`, boş bir
`plugins` girdisi, `33` veya üzerinde nihai bir yama ve tam ucundaki standart
`extended-stable/YYYY.M.33` dalını gerektirir. Plugin
`latest` veya `beta` etiketlerini hiçbir zaman taşımaz. Yeni paket sürümleri `extended-stable` değerini OIDC güvenilir yayını (`npm publish --tag extended-stable`) aracılığıyla atomik olarak
alır; bu kaynak iş akışı token ile kimlik doğrulamalı `npm dist-tag add` kullanmaz. Yeniden denemeler
npm'de zaten bulunan tam sürümleri atlar, ardından tam
geri okuma her bir tam paketin ve `extended-stable` etiketinin yakınsadığını doğrulamadığı sürece kapalı biçimde başarısız olur.

`OpenClaw Release Publish` şu operatör denetimli girdileri kabul eder:

- `tag`: gerekli sürüm etiketi; zaten mevcut olmalıdır
- `preflight_run_id`: başarılı `OpenClaw NPM Release` ön kontrol çalıştırma kimliği; `publish_openclaw_npm=true` veya `plugin_publish_scope=all-publishable` olduğunda gereklidir
- `full_release_validation_run_id`: başarılı `Full Release Validation` çalıştırma kimliği; `publish_openclaw_npm=true` veya `plugin_publish_scope=all-publishable` olduğunda gereklidir
- `full_release_validation_run_attempt`: `full_release_validation_run_id` ile eşleştirilmiş tam pozitif deneme; çalıştırma kimliği verildiğinde zorunludur
- `windows_node_tag`: tam ön sürüm olmayan `openclaw/openclaw-windows-node` sürüm etiketi; kararlı OpenClaw yayımlaması için gereklidir
- `windows_node_installer_digests`: mevcut Windows yükleyici adlarını sabitlenmiş `sha256:` özetleriyle eşleyen, aday tarafından onaylanmış kompakt JSON eşlemesi; kararlı OpenClaw yayımlaması için gereklidir
- `npm_telegram_run_id`: nihai sürüm kanıtına dahil edilecek isteğe bağlı başarılı `NPM Telegram Beta E2E` çalıştırma kimliği
- `npm_dist_tag`: OpenClaw paketinin npm hedef etiketi; `alpha`, `beta` veya `latest` değerlerinden biri
- `plugin_publish_scope`: varsayılan olarak `all-publishable`; `selected` değerini yalnızca `publish_openclaw_npm=false` ile odaklı ve yalnızca plugine yönelik onarım çalışmalarında kullanın
- `plugins`: `plugin_publish_scope=selected` olduğunda virgülle ayrılmış `@openclaw/*` paket adları
- `publish_openclaw_npm`: varsayılan olarak `true`; yalnızca iş akışını sadece plugine yönelik bir onarım orkestratörü olarak kullanırken `false` değerini ayarlayın
- `release_profile`: sürüm kanıtı özetleri için kullanılan sürüm kapsam profili; varsayılan olarak doğrulama bildiriminden okuyan `from-validation` kullanılır veya `beta`, `stable` ya da `full` ile geçersiz kılınır
- `wait_for_clawhub`: npm kullanılabilirliğinin ClawHub yan bileşeni tarafından engellenmemesi için varsayılan olarak `false`; yalnızca iş akışının tamamlanmasının ClawHub'ın tamamlanmasını da içermesi gerektiğinde `true` değerini ayarlayın

`OpenClaw Release Checks` şu operatör denetimli girdileri kabul eder:

- `ref`: doğrulanacak dal, etiket veya tam commit SHA'sı. Gizli bilgi içeren denetimler, çözümlenen commit'in bir OpenClaw dalından veya sürüm etiketinden erişilebilir olmasını gerektirir.
- `run_release_soak`: beta sürüm denetimleri için kapsamlı canlı/E2E, Docker sürüm yolu ve tüm sürümlerden yükseltmede ayakta kalma dayanıklılık testlerini etkinleştirir. `release_profile=stable` ve `release_profile=full` tarafından zorunlu olarak etkinleştirilir.

Kurallar:

- Yama `33` altındaki normal nihai ve düzeltme sürümleri, `beta` veya `latest` hedeflerinden birinde yayımlanabilir. Yama `33` veya üzerindeki nihai sürümler `extended-stable` hedefinde yayımlanmalıdır ve bu sınırdaki düzeltme son ekli sürümler reddedilir.
- Beta ön sürüm etiketleri yalnızca `beta` hedefinde; alfa ön sürüm etiketleri yalnızca `alpha` hedefinde yayımlanabilir
- `OpenClaw NPM Release` için tam commit SHA girdisine yalnızca `preflight_only=true` olduğunda izin verilir
- `OpenClaw Release Checks` ve `Full Release Validation` her zaman yalnızca doğrulama amaçlıdır
- Gerçek yayımlama yolu, ön kontrolde kullanılan `npm_dist_tag` ile aynı değeri kullanmalıdır; iş akışı, yayımlama devam etmeden önce bu meta verileri doğrular

## Normal beta/en son kararlı sürüm sırası

Bu eski sıra; Pluginler, GitHub Release, Windows ve diğer platform çalışmalarını da yöneten normal düzenlenmiş sürüm içindir. Bu sayfanın üst kısmında belgelenen aylık `.33+` Gateway genişletilmiş kararlı sürüm yolu değildir.

Normal bir düzenlenmiş kararlı sürüm hazırlanırken:

1. `OpenClaw NPM Release` komutunu `preflight_only=true` ile çalıştırın. Bir etiket mevcut olmadan önce, ön kontrol iş akışının yalnızca doğrulama amaçlı deneme çalıştırması için geçerli tam iş akışı dalı commit SHA'sını kullanabilirsiniz.
2. Normal, önce beta akışı için `npm_dist_tag=beta` seçeneğini; yalnızca bilinçli olarak doğrudan kararlı yayımlama istediğinizde `latest` seçeneğini belirleyin.
3. Tek bir manuel iş akışından normal CI ile birlikte canlı istem önbelleği, Docker, QA Lab, Matrix ve Telegram kapsamı istediğinizde sürüm dalı, sürüm etiketi veya tam commit SHA'sı üzerinde `Full Release Validation` komutunu çalıştırın. Bilinçli olarak yalnızca deterministik normal test grafiğine ihtiyacınız varsa bunun yerine sürüm referansında manuel `CI` iş akışını çalıştırın.
4. İmzalı x64 ve ARM64 yükleyicileri dağıtılacak olan, ön sürüm niteliğinde olmayan tam `openclaw/openclaw-windows-node` sürüm etiketini seçin. Bunu `windows_node_tag` olarak, doğrulanmış özet eşlemelerini ise `windows_node_installer_digests` olarak kaydedin. Sürüm adayı yardımcısı her ikisini de kaydeder ve oluşturduğu yayımlama komutuna ekler.
5. Başarılı `preflight_run_id`, `full_release_validation_run_id` ve tam `full_release_validation_run_attempt` değerlerini kaydedin.
6. `OpenClaw Release Publish` komutunu güvenilir `main` üzerinden aynı `tag`, aynı `npm_dist_tag`, seçilen `windows_node_tag`, bunun kaydedilmiş `windows_node_installer_digests` değeri, kaydedilmiş `preflight_run_id`, `full_release_validation_run_id` ve `full_release_validation_run_attempt` ile çalıştırın. OpenClaw npm paketini yükseltmeden önce dışsallaştırılmış Pluginleri npm ve ClawHub'da yayımlar.
7. Sürüm `beta` üzerinde yayımlandıysa bu kararlı sürümü `beta` üzerinden `latest` hedefine yükseltmek için `openclaw/releases/.github/workflows/openclaw-npm-dist-tags.yml` iş akışını kullanın.
8. Sürüm bilinçli olarak doğrudan `latest` hedefinde yayımlandıysa ve `beta` aynı kararlı derlemeyi hemen izlemeliyse her iki dist-tag'i de kararlı sürüme yönlendirmek için aynı sürüm iş akışını kullanın veya zamanlanmış kendi kendini onarma eşitlemesinin `beta` değerini daha sonra taşımasına izin verin.

Dist-tag değişikliği, hâlâ `NPM_TOKEN` gerektirdiğinden sürüm kayıt deposunda bulunurken kaynak deposu yalnızca OIDC ile yayımlamayı sürdürür. Böylece hem doğrudan yayımlama yolu hem de önce beta yükseltme yolu belgelenmiş ve operatörler tarafından görülebilir kalır.

Bir bakım sorumlusunun yerel npm kimlik doğrulamasına geri dönmesi gerekirse tüm 1Password CLI (`op`) komutlarını yalnızca özel bir tmux oturumu içinde çalıştırın. `op` komutunu doğrudan ana ajan kabuğundan çağırmayın; bunu tmux içinde tutmak istemlerin, uyarıların ve OTP işlemenin gözlemlenebilmesini sağlar ve yinelenen ana makine uyarılarını önler.

## Herkese açık referanslar

- [`.github/workflows/full-release-validation.yml`](https://github.com/openclaw/openclaw/blob/main/.github/workflows/full-release-validation.yml)
- [`.github/workflows/package-acceptance.yml`](https://github.com/openclaw/openclaw/blob/main/.github/workflows/package-acceptance.yml)
- [`.github/workflows/openclaw-npm-release.yml`](https://github.com/openclaw/openclaw/blob/main/.github/workflows/openclaw-npm-release.yml)
- [`.github/workflows/openclaw-release-checks.yml`](https://github.com/openclaw/openclaw/blob/main/.github/workflows/openclaw-release-checks.yml)
- [`.github/workflows/openclaw-cross-os-release-checks-reusable.yml`](https://github.com/openclaw/openclaw/blob/main/.github/workflows/openclaw-cross-os-release-checks-reusable.yml)
- [`.github/workflows/docker-release.yml`](https://github.com/openclaw/openclaw/blob/main/.github/workflows/docker-release.yml)
- [`scripts/resolve-openclaw-package-candidate.mjs`](https://github.com/openclaw/openclaw/blob/main/scripts/resolve-openclaw-package-candidate.mjs)
- [`scripts/openclaw-npm-release-check.ts`](https://github.com/openclaw/openclaw/blob/main/scripts/openclaw-npm-release-check.ts)
- [`scripts/package-mac-dist.sh`](https://github.com/openclaw/openclaw/blob/main/scripts/package-mac-dist.sh)
- [`scripts/make_appcast.sh`](https://github.com/openclaw/openclaw/blob/main/scripts/make_appcast.sh)

Bakım sorumluları gerçek çalışma kılavuzu için [`openclaw/maintainers/release/README.md`](https://github.com/openclaw/maintainers/blob/main/release/README.md) içindeki özel sürüm belgelerini kullanır.

## İlgili

- [Sürüm kanalları](/tr/install/development-channels)
