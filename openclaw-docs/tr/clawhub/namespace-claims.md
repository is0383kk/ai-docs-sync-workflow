---
read_when:
    - Bir kuruluşu, markayı, paket kapsamını, sahip kullanıcı adını, skill kısa adını veya paket ad alanını sahiplenme
    - Zaten alınmış veya ayrılmış bir ad alanını çözümleme
    - Rapor, itiraz veya ad alanı talebinden hangisinin kullanılacağına karar verme
sidebarTitle: Org and Namespace Claims
summary: Kuruluş, marka, sahip kullanıcı adı, paket kapsamı, skill slug'ı veya ad alanı sahipliği anlaşmazlıkları için ClawHub incelemesi nasıl talep edilir?
title: Kuruluş ve Ad Alanı Talepleri
x-i18n:
    generated_at: "2026-07-26T23:14:03Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 77a4d8090b55298c401154d116d93d4f8139d40983a45982288d8e48bcea40fb
    source_path: clawhub/namespace-claims.md
    workflow: 16
---

# Kuruluş ve Ad Alanı Talepleri

ClawHub, sahip tanıtıcılarını, kuruluş tanıtıcılarını, skill kısa adlarını, Plugin paket adlarını ve
paket kapsamlarını genel ad alanları olarak kullanır. Bir ad alanı gerçek dünyadaki bir
projeye, markaya, paket ekosistemine veya kuruluşa ait gibi görünmesine rağmen ClawHub'da
zaten talep edilmiş, ayrılmış, yanıltıcı veya ihtilaflıysa personelden bunu
[Kuruluş / Ad Alanı Talebi sorun formu](https://github.com/openclaw/clawhub/issues/new?template=org-namespace-claim.yml)
ile incelemesini isteyin.

Bu yolu genel ve hassas olmayan sahiplik incelemeleri için kullanın. Ad alanı talepleri için
ürün içi bildirimleri veya hesap itiraz formunu kullanmayın.

## Ne Zaman Talep Açılmalı?

Gerçek dünyadaki sahiplik nedeniyle bir ad alanının ayrılması, devredilmesi, yeniden adlandırılması,
gizlenmesi, karantinaya alınması, başka bir adla eşlenmesi veya farklı şekilde değiştirilmesi gerekip
gerekmediğini ClawHub personelinin incelemesi gerektiğini düşünüyorsanız bir ad alanı talebi açın.

Örnekler:

- GitHub kuruluşunuzla, projenizle, şirketinizle veya topluluğunuzla eşleşen bir kuruluş tanıtıcısı
- yalnızca eşleşen ClawHub sahibi altında yayımlanması gereken `@example-org/*` gibi bir paket kapsamı
- bir projeyi taklit ediyor gibi görünen bir skill kısa adı veya Plugin paket adı
- bir marka, ticari marka, proje yeniden adlandırması veya paket geçmişi ihtilafı
- hak sahibi ad alanı sahibini engelleyen silinmiş, etkin olmayan veya ulaşılamayan bir sahip

Listeleme, sahiplik ihtilafının ötesinde güvensiz, kötü amaçlı veya yanıltıcıysa
ilgili moderasyon veya güvenlik yönergelerini de izleyin. Ad alanı talep
formu, acil güvenlik açığı bildirimi için değil, sahiplik incelemesi içindir.

## Başvurmadan Önce

Öncelikle ad alanıyla eşleşen sahip üzerinden yayımladığınızı doğrulayın.
Plugin paketlerinde `@example-org/example-plugin` gibi kapsamlı adlar,
eşleşen `example-org` sahibi olarak yayımlanmalıdır.

Mevcut sahibi yönetebiliyorsanız etkilenen kaynağı yayımlayarak, yeniden
adlandırarak, devrederek, gizleyerek veya silerek ad alanını doğrudan düzeltin. Mevcut
sahibi yönetemiyorsanız veya personelin bir ihtilafı çözmesi gerekiyorsa talepte bulunun.

## Eklenecek Kanıtlar

Genel ve hassas olmayan kanıtlar kullanın. Yararlı kanıtlar şunlardır:

- GitHub kuruluşu, deposu, sürümü veya bakım sorumlusu geçmişi
- ad alanını belirten resmî proje belgeleri
- alan adı veya resmî e-posta alan adı kanıtı
- npm, PyPI, crates.io veya başka bir paket kayıt sistemi kapsamının denetimi
- genel olarak güvenle tartışılabilecek ticari marka, marka veya proje sahipliği kanıtı
- kaynak deposu geçmişi, paket geçmişi veya genel yeniden adlandırma bildirimleri
- ihtilaflı ClawHub sahibi, skill, Plugin, paket veya sorun bağlantıları

Her bağlantının neyi kanıtladığını açıklayın. Personel, özel kimlik bilgilerine veya
gizli bilgilere ihtiyaç duymadan ilişkiyi anlayabilmelidir.

## Eklenmemesi Gerekenler

Genel bir GitHub sorununa gizli bilgiler veya özel kanıtlar koymayın. Şunları eklemeyin:

- API token'ları, imzalama anahtarları veya kimlik bilgileri
- DNS doğrulama token'ları
- özel hukuki dosyalar veya sözleşmeler
- kişisel kimlik belgeleri
- özel e-postalar, özel güvenlik raporları veya gizli müşteri verileri

Talep formu, hassas kanıtlar için özel bir personel kanalının gerekip gerekmediğini sorar.
Hassas materyalleri genel olarak yayımlamak yerine bu seçeneği kullanın.

## Olası Sonuçlar

Kanıtlara ve riske bağlı olarak ClawHub personeli bir ad alanını ayırabilir,
sahipliği devredebilir, bir kaynağı yeniden adlandırabilir, mevcut bir listelemeyi gizleyebilir veya karantinaya alabilir,
bir başka ad veya yönlendirme ekleyebilir, daha fazla kanıt isteyebilir ya da talebi reddedebilir.

Ad alanı incelemesi, eşleşen her adın devredileceğini garanti etmez.
Personel; genel kanıtları, mevcut kullanımı, güvenlik riskini ve kullanıcılar üzerindeki etkiyi değerlendirir.

## İlgili Belgeler

- [Yayımlama](/tr/clawhub/publishing)
- [Sorun Giderme](/clawhub/troubleshooting#publish-fails-because-a-namespace-is-claimed-or-reserved)
- [Moderasyon ve Hesap Güvenliği](/clawhub/moderation)
- [Güvenlik](/clawhub/security)
