---
read_when:
    - Bir OpenClaw sürümünde npm shrinkwrap'un ne anlama geldiğini öğrenmek istiyorsunuz
    - Paket kilit dosyalarını, bağımlılık değişikliklerini veya tedarik zinciri riskini inceliyorsunuz
    - Yayımlamadan önce kök veya plugin npm paketlerini doğruluyorsunuz
summary: OpenClaw sürümlerinde npm shrinkwrap'un sade ve teknik açıklaması
title: npm shrinkwrap
x-i18n:
    generated_at: "2026-07-26T23:42:35Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: d1e6c0d4541da9220d50cde0b9db064e5a91b81d6562cb16ac697de7d4017098
    source_path: gateway/security/shrinkwrap.md
    workflow: 16
---

OpenClaw kaynak çalışma kopyaları `pnpm-lock.yaml` kullanır. Yayımlanan OpenClaw npm paketleri, npm'in yayımlanabilir bağımlılık kilit dosyası olan `npm-shrinkwrap.json` kullanır; böylece paket kurulumları, sürüm sırasında incelenen bağımlılık grafiğini kullanır.

## Neden önemlidir?

Shrinkwrap, bir npm paketiyle gönderilen bağımlılık ağacının makbuzudur: npm'e hangi kesin geçişli sürümlerin kurulacağını bildirir.

| Dosya                  | Önemli olduğu yer         | Anlamı                     |
| --------------------- | ------------------------ | --------------------------------- |
| `pnpm-lock.yaml`      | OpenClaw kaynak çalışma kopyası | Bakımcı bağımlılık grafiği       |
| `npm-shrinkwrap.json` | Yayımlanan npm paketi    | Kullanıcılar için npm kurulum grafiği       |
| `package-lock.json`   | Yerel npm uygulamaları           | OpenClaw yayımlama sözleşmesi değildir |

OpenClaw sürümleri açısından bunun anlamı şudur:

- yayımlanan paket, kurulum sırasında npm'den yeni bir bağımlılık grafiği oluşturmasını istemez;
- bağımlılık değişiklikleri bir kilit dosyası farkında yer aldığından incelenebilir;
- sürüm doğrulaması, kullanıcıların kuracağı grafiğin aynısını test eder;
- paket boyutu veya yerel bağımlılıklarla ilgili beklenmedik durumlar yayımlamadan önce ortaya çıkar.

Shrinkwrap bir korumalı alan değildir. Tek başına bir bağımlılığı güvenli hâle getirmez ve ana makine yalıtımının, `openclaw security audit`, paket kaynağı doğrulamasının veya kurulum duman testlerinin yerini almaz.

OpenClaw bir gateway, plugin ana makinesi, model yönlendiricisi ve ajan çalışma zamanıdır; bu nedenle varsayılan kurulum başlatma süresini, disk kullanımını, yerel paket indirmelerini ve tedarik zinciri risklerini etkiler. Shrinkwrap, sürüm incelemesine kararlı bir sınır sağlar: inceleyiciler geçişli bağımlılık hareketlerini görür, doğrulayıcılar beklenmeyen kilit dosyası sapmalarını reddeder ve plugin paketleri kök pakete dayanmak yerine kendi kilitli bağımlılık grafiklerini taşır.

## Oluşturma ve denetleme

Kök `openclaw` npm paketi, OpenClaw'a ait npm plugin paketleri (örneğin `@openclaw/discord`) ve [`@openclaw/ai`](/tr/reference/openclaw-ai) gibi yayımlanabilir çalışma alanı paketleri, yayımlanırken `npm-shrinkwrap.json` içerir. Çalışma alanı bağımlılıkları, kök paketle birlikte yayımlandıkları için kök shrinkwrap dosyasına dahil edilmez; bunun yerine yayımlanabilir her çalışma alanı paketi kendi geçişli ağacını sabitler. Uygun plugin paketleri, yalnızca kurulum zamanı çözümlemesine güvenmek yerine çalışma zamanı bağımlılık dosyalarını plugin tarball dosyasında taşıyan açık `bundledDependencies` ile de yayımlanabilir.

```bash
# Shrinkwrap ile yönetilen tüm paketler (kök + yayımlanabilir pluginler)
pnpm deps:shrinkwrap:generate
pnpm deps:shrinkwrap:check

# Yalnızca kök paket
pnpm deps:shrinkwrap:root:generate
pnpm deps:shrinkwrap:root:check

# Yalnızca mevcut değişiklik kümesinden etkilenen paketler
pnpm deps:shrinkwrap:changed:generate
pnpm deps:shrinkwrap:changed:check
```

Oluşturucu, npm'in yayımlanabilir kilit biçimini çözümler ancak `pnpm-lock.yaml` içinde zaten bulunmayan oluşturulmuş paket sürümlerini reddeder. Bu, pnpm bağımlılık yaşı, geçersiz kılma ve yama inceleme sınırını korur.

Şunları güvenlik açısından hassas olarak inceleyin:

- `pnpm-lock.yaml`
- `npm-shrinkwrap.json`
- pakete dahil edilen plugin bağımlılık yükleri
- tüm `package-lock.json` farkları

OpenClaw paket doğrulayıcıları, yeni kök paket tarball dosyalarında shrinkwrap bulunmasını zorunlu kılar ve yayımlanan paketlerde `package-lock.json` kullanımını reddeder. Plugin npm yayımlama yolu, plugin'e özgü shrinkwrap dosyasını denetler, pakete özgü dahil edilmiş bağımlılıkları kurar ve ardından paketi oluşturur veya yayımlar.

## Yayımlanmış bir paketi inceleme

Kök paket:

```bash
npm pack openclaw@<version> --json --pack-destination /tmp/openclaw-pack
tar -tf /tmp/openclaw-pack/openclaw-<version>.tgz | grep '^package/npm-shrinkwrap.json$'
```

Plugin paketi:

```bash
npm pack @openclaw/discord@<version> --json --pack-destination /tmp/openclaw-plugin-pack
tar -tf /tmp/openclaw-plugin-pack/openclaw-discord-<version>.tgz | grep '^package/npm-shrinkwrap.json$'
tar -tf /tmp/openclaw-plugin-pack/openclaw-discord-<version>.tgz | grep '^package/node_modules/'
```

Arka plan bilgisi: [npm-shrinkwrap.json](https://docs.npmjs.com/cli/v11/configuring-npm/npm-shrinkwrap-json).
