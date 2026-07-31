---
read_when:
    - Kararlı/uzatılmış kararlı/beta/geliştirme sürümleri arasında geçiş yapmak istiyorsunuz
    - Belirli bir sürümü, etiketi veya SHA'yı sabitlemek istiyorsunuz
    - Ön sürümleri etiketliyor veya yayımlıyorsunuz
sidebarTitle: Release Channels
summary: 'Kararlı, genişletilmiş kararlı, beta ve geliştirme kanalları: anlamları, kanal değiştirme, sabitleme ve etiketleme'
title: Sürüm kanalları
x-i18n:
    generated_at: "2026-07-26T23:43:19Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: a99e31f5121c0ab8696e638cb10a7ce16e8f32c81e4b2bef1f703eef71191494
    source_path: install/development-channels.md
    workflow: 16
---

OpenClaw dört güncelleme kanalıyla sunulur:

- **stable**: npm dist-tag `latest`. Çoğu kullanıcı için önerilir.
- **extended-stable**: npm dist-tag `extended-stable`. Tamamen yeni, geriden gelen
  desteklenen ay paket kanalıdır. Yalnızca paket olarak sunulur ve kurulum
  yalnızca ön planda gerçekleştirilir. Kaydedilmiş bir seçim,
  `update.checkOnStart` etkinleştirildiğinde salt okunur güncelleme ipuçları alır,
  ancak güncellemeleri hiçbir zaman otomatik olarak uygulamaz.
- **beta**: npm dist-tag `beta`. `beta` yoksa
  veya mevcut kararlı sürümden eskiyse `latest` değerine geri döner.
- **dev**: `main` dalının hareketli ucu (git). Yayımlandığında npm dist-tag `dev`. `main`
  denemeler ve etkin geliştirme içindir; tamamlanmamış özellikler veya geriye
  dönük uyumsuz değişiklikler içerebilir. Üretim Gateway'leri için çalıştırmayın.

Kararlı derlemeler genellikle önce **beta** kanalında yayımlanır, burada
incelendikten sonra sürüm numarası artırılmadan **latest** kanalına yükseltilir.
Bakımcılar doğrudan `latest` kanalında da yayımlayabilir. npm
kurulumları için doğruluğun kaynağı dist-tag'lerdir.

## Kanallar arasında geçiş yapma

```bash
openclaw update --channel stable
openclaw update --channel extended-stable
openclaw update --channel beta
openclaw update --channel dev
```

`--channel` seçimi yapılandırmadaki `update.channel` alanında kalıcı
hale getirir ve her iki kurulum yolunu da yönetir:

| Kanal             | npm/paket kurulumları                                                                                                                                                                  | git kurulumları                                                                                                                                                    |
| ----------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `stable`          | dist-tag `latest`                                                                                                                                                                      | en son kararlı git etiketi (`-alpha.N`, `-beta.N`, `-rc.N`, `-dev.N`, `-next.N`, `-preview.N`, `-canary.N`, `-nightly.N` ve diğer adlandırılmış ön sürüm sonekleri hariç) |
| `extended-stable` | genel npm `extended-stable` seçicisini çözümler, seçilen paketin tam olarak doğru olduğunu doğrular ve tam olarak bu sürümü kurar. `latest`, `beta` veya `dev` seçeneklerine geri dönmeden güvenli biçimde başarısız olur. | desteklenmez: OpenClaw çalışma kopyasını değiştirmeden bırakır ve paket kurulumu kullanmanızı ister                                                                  |
| `beta`            | dist-tag `beta`; `beta` yoksa veya daha eskiyse `latest` değerine geri döner                                                                                                              | en son beta git etiketi; beta yoksa veya daha eskiyse en son kararlı git etiketine geri döner                                                                       |
| `dev`             | dist-tag `dev` (nadiren kullanılır; çoğu dev kullanıcısı git kurulumlarını çalıştırır)                                                                                                                         | verileri getirir, çalışma kopyasını üst kaynaktaki `main` dalı üzerine yeniden temellendirir, derler ve genel CLI'yi yeniden kurar                        |

`dev` git kurulumlarında varsayılan çalışma kopyası
`~/openclaw`'dir (`OPENCLAW_HOME` ayarlandığında
`$OPENCLAW_HOME/openclaw`); `OPENCLAW_GIT_DIR` ile geçersiz kılın.

<Tip>
Kararlı ve dev sürümlerini paralel olarak tutmak için iki ayrı çalışma kopyası kullanın ve her Gateway'i kendi çalışma kopyasına yönlendirin.
</Tip>

## Tek seferlik sürüm veya etiket hedefleme

Kalıcı kanalı değiştirmeden tek bir güncelleme için belirli bir dist-tag, sürüm
veya paket belirtimini hedeflemek üzere `--tag` kullanın:

```bash
# Belirli bir sürümü kur
openclaw update --tag 2026.4.1-beta.1

# Beta dist-tag'inden kur (tek seferliktir, kalıcı olmaz)
openclaw update --tag beta

# Hareketli GitHub main çalışma kopyasına geç (kalıcı)
openclaw update --channel dev

# Belirli bir npm paket belirtimini kur
openclaw update --tag openclaw@2026.4.1-beta.1

# Kanalı kalıcı hale getirmeden GitHub main'den bir kez kur
openclaw update --tag main
```

Notlar:

- `--tag` yalnızca **paket (npm) kurulumlarına** uygulanır; git kurulumları bunu yok sayar.
- Etiket kalıcı hale getirilmez; sonraki `openclaw update` yapılandırılmış
  kanalı kullanır.
- `--tag main`, bu tek çalıştırma için npm uyumlu
  `github:openclaw/openclaw#main` belirtimine eşlenir. Kalıcı ve hareketli bir
  `main` kurulumu için `openclaw update --channel dev` kullanın
  (paket kurulumları bir git çalışma kopyasına geçer) veya yükleyicinin git
  yöntemiyle yeniden kurun: `curl -fsSL https://openclaw.ai/install.sh | bash -s -- --install-method git --version main`.
  npm kurulum yolu, GitHub/git kaynak hedeflerini doğrudan reddeder ve bunun
  yerine sizi git yöntemine yönlendirir.
- Sürüm düşürme koruması: hedef sürüm mevcut sürümden eskiyse OpenClaw
  onay ister (`--yes` ile atlayın).
- Extended-stable her zaman doğrulanmış tam paket hedefini kullanır.
  `--tag extended-stable` için tek seferlik bir diğer ad değildir ve
  `--tag` etkin bir extended-stable kanalıyla birlikte kullanılamaz.
- `--channel beta`, `--tag beta` değerinden farklıdır: kanal akışı,
  beta yoksa veya daha eskiyse stable/latest değerine geri dönebilir;
  `--tag beta` ise bu tek çalıştırmada her zaman ham
  `beta` dist-tag'ini hedefler.

## Deneme çalıştırması

`openclaw update` komutunun değişiklik yapmadan ne gerçekleştireceğini
önizleyin:

```bash
openclaw update --dry-run
openclaw update --channel beta --dry-run
openclaw update --tag 2026.4.1-beta.1 --dry-run
openclaw update --dry-run --json
```

Deneme çalıştırması; etkin kanalı, hedef sürümü, planlanan işlemleri ve sürüm
düşürme onayı gerekip gerekmediğini bildirir.

## Plugin'ler ve kanallar

`openclaw update` ile kanal değiştirmek Plugin kaynaklarını da eşitler:

- `dev`, paketle gelen karşılığı bulunan kurulu Plugin'leri
  yeniden paketle gelen (git çalışma kopyası) kaynaklarına geçirir.
- `stable` ve `beta`, npm veya ClawHub üzerinden
  kurulmuş Plugin paketlerini geri yükler.
- `extended-stable`, yalın/varsayılan ya da `latest` amaçlı uygun
  resmî npm Plugin'lerini kurulu çekirdeğin tam sürümüne çözümler. Çalışma
  zamanında Plugin `@extended-stable` etiketlerini sorgulamaz.
- npm ile kurulan Plugin'ler, çekirdek güncellemesi tamamlandıktan sonra
  güncellenir.

## Mevcut durumu denetleme

```bash
openclaw update status
```

Etkin kanalı (bunu belirleyen kaynakla birlikte: yapılandırma, git etiketi,
git dalı, kurulu sürüm veya varsayılan), kurulum türünü (git veya paket),
mevcut sürümü ve güncelleme kullanılabilirliğini gösterir.

## Etiketleme için en iyi uygulamalar

- Git çalışma kopyalarının ulaşmasını istediğiniz sürümleri etiketleyin:
  kararlı sürüm için `vYYYY.M.PATCH`, beta için `vYYYY.M.PATCH-beta.N`.
  `-alpha.N`, `-rc.N` ve `-next.N` gibi
  adlandırılmış ön sürüm sonekleri kararlı veya beta hedefleri değildir.
- `vYYYY.M.PATCH-1` ve `v1.0.1-1` gibi eski sayısal kararlı
  etiketler, uyumluluk amacıyla hâlâ kararlı git etiketleri olarak tanınır.
- `vYYYY.M.PATCH.beta.N` (noktayla ayrılmış) da uyumluluk amacıyla tanınır;
  `-beta.N` tercih edilir.
- Etiketleri değişmez tutun: bir etiketi hiçbir zaman taşımayın veya yeniden kullanmayın.
- npm kurulumları için doğruluğun kaynağı npm dist-tag'leri olmaya devam eder:
  - `latest` -> kararlı
  - `extended-stable` -> geriden gelen desteklenen ay paket sürümü
  - `beta` -> aday derleme veya önce beta olarak sunulan kararlı derleme
  - `dev` -> main anlık görüntüsü (isteğe bağlı)

## macOS uygulamasının kullanılabilirliği

Beta ve dev derlemeleri bir macOS uygulama sürümü **içermeyebilir**. Bu sorun
değildir:

- Git etiketi ve npm dist-tag'i yine de bağımsız olarak yayımlanabilir.
- Sürüm notlarında veya değişiklik günlüğünde "bu beta için macOS derlemesi yok" ifadesini belirtin.

## İlgili

- [Güncelleme](/tr/install/updating)
- [Yükleyicinin iç işleyişi](/tr/install/installer)
