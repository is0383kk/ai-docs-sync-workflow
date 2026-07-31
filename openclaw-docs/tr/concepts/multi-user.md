---
read_when:
    - Bir OpenClaw aracısını diğer operatörlerle paylaşıyorsunuz
    - Oturum sahibi ve çevrimiçi olma göstergelerini anlamanız gerekir
    - Paylaşılan tek bir agent'ın yeterli yalıtım sağlayıp sağlamadığına karar veriyorsunuz
summary: Birden fazla kişi tek bir agent'ı çalıştırdığında oturum sahipliği ve iletişim durumunun işleyişi
title: Çok kullanıcılı mod
x-i18n:
    generated_at: "2026-07-26T23:55:12Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: c6a5a0e37b8dbeb2ebb7f32c3518acc6f3995dbfc09102f4d58c85e9cd62dfc2
    source_path: concepts/multi-user.md
    workflow: 16
---

Çok kullanıcılı mod, birkaç güvenilir kişinin aynı OpenClaw aracısını çalıştırmasına olanak tanır. Bir ekibin işi kimin başlattığını ve şu anda kimin izlediğini anlayabilmesi için oturum sahipliği, canlı mevcudiyet ve oluşturan kişiye göre filtreleme özellikleri ekler.

## Güven sınırı

Bir aracıyı çalıştırabilen herkes, aracıya yapabildiği her şeyi yaptırabilir. Oturum sahipliği, kenar çubuğundaki görünürlük ve mevcudiyet göstergeleri kullanılabilirlik özellikleridir; güvenlik sınırları değildir.

Kişilerin birbirlerinin oturumlarına, araçlarına, kimlik bilgilerine veya dosyalarına erişmemesi gerekiyorsa onlara ayrı aracılar ya da ayrı Gateway/ana makine güven sınırları sağlayın. Yalıtım için sahip avatarlarına veya filtrelere güvenmeyin.

## Sahiplik ve mevcudiyet

Yeni oturumlar, oluşturma yolu oturuma kimin neden olduğunu kanıtlayabildiğinde bir kez yazılabilen `createdActor` değerini kaydeder. Kimliği doğrulanmış kişiler kalıcı Gateway profil kimliklerini kullanır; istekte bulunan aracılar ve sistem yolları da aynı aktör alanını kullanır. Kanıtlanmış bir aktör olmadan oluşturulan oturumlar ilişkilendirilmemiş olarak kalır.

Oturum satırları döndürüldüğünde kişilerin görünen adları mevcut Gateway profilinden çözümlenir. OpenClaw, oturum girdilerinde etiketleri saklamaz; bu nedenle bir profil adının değiştirilmesi, oturum geçmişini yeniden yazmadan sahiplik kullanıcı arayüzünü günceller.

Web uygulaması, sahiplik ile mevcudiyeti görsel olarak birbirinden ayrı tutar:

- Dolu bir sahip avatarı, ilgili oturumun kullanım ömrü boyunca kalıcıdır.
- Halkalı veya yarı saydam mevcudiyet avatarları, şu anda bağlı olan ya da izleyen kişileri gösterir.
- Kenar çubuğundaki kişi filtresi, mevcut özel grupları korurken tek bir kimliğin oluşturduğu oturumları gösterir.

Yüklenen oturum listesinde ikiden az farklı oluşturan kişi göründüğünde OpenClaw, sahiplik ve kişi filtresiyle ilgili tüm arayüz öğelerini gizler. Bu nedenle tek kullanıcılı bir Gateway değişmemiş görünür.

## Taslaklar

Devam eden çalışmaları yayımlayana kadar ekip arkadaşlarının kenar çubuklarından uzak tutmak için bir oturumu taslak olarak başlatın. Taslaklar, diğer kişilerin taslaklarını soluk bir hayalet işaretiyle gören yöneticilerden hiçbir zaman gizlenmez. Bu bir koordinasyon özelliğidir; güvenlik sınırı değildir.

## Tur ilişkilendirmesi

Tur göndericisi ilişkilendirmesi mümkün olan en iyi şekilde yapılır. Yönlendirme, girdiyi etkin bir turla birleştirebildiğinden döküm her kişinin katkısını her zaman ayrı bir tur olarak gösteremez.

## İlgili

- [Ana oturum](/concepts/main-session)
- [Oturum yönetimi](/tr/concepts/session)
- [Mevcudiyet](/tr/concepts/presence)
- [Gateway güvenliği](/tr/gateway/security)
