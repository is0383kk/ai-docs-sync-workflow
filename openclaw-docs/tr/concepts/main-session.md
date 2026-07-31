---
read_when:
    - Temsilcinizin nerede "yaşadığını" anlamak istiyorsunuz
    - Telegram, WhatsApp veya web üzerinden yazdığınızda aynı bağlamın kullanılmasını beklersiniz.
    - Temsilcinizin gruplarda ve yan ileti dizilerinde neler olduğunu bilmesini istiyorsunuz
summary: 'Tüm kanallarınızda devam eden tek bir konuşma: kişisel aracı varsayılanı'
title: Ana oturum
x-i18n:
    generated_at: "2026-07-26T23:18:06Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: fb77382ebdce269a05a03ab6fa39b44b1e9f1856166f1d9cb79111dccb547f69
    source_path: concepts/main-session.md
    workflow: 16
---

OpenClaw her şeyden önce kişisel bir ajandır. Kullanıma hazır durumda ona
gönderdiğiniz her doğrudan mesaj — Telegram, WhatsApp, iMessage, Slack DM'leri,
web uygulaması veya herhangi bir yerden — **tek bir ilerleyen konuşmaya** ulaşır:
ana oturum. Telefonunuzdan bir şey sorun, dizüstü bilgisayarınızdan devam edin;
ajan her iki yerde de aynı bağlama sahiptir. Tek bir beyin vardır ve düşündüğü
yer burasıdır.

Altyapıda ana oturum, `agent:<agentId>:main` anahtarına sahip sıradan bir oturumdur
(örneğin `agent:main:main`). Onu özel kılan, varsayılan DM kapsamının tüm
doğrudan mesajları bu oturumda birleştirmesi ve sistemin geri kalanının onu
ajanın kökü olarak görmesidir: Heartbeat'ler onu uyandırır, arka plan çalışmaları
sonuçlarını ona bildirir ve başka yerlerdeki etkinlikler ona akar.

## Ana Sayfa

Web uygulamasında ana oturum, kenar çubuğundaki ilk giriş olan **Ana Sayfa**
sayfasıdır. Üstteki kimlik satırı ajanınızdır (ajan menüsü için buna tıklayın);
Ana Sayfa, onunla konuştuğunuz yerdir. Ana konuşmadan dallanan oturumlar
**İş Parçacıkları**, grup sohbetleri **Gruplar**, kodlama/CLI oturumları ise
**Kodlama** altında görünür.

## Ana oturuma neler akar?

Ana oturum yalnızca bir sohbet günlüğü değildir; ajanınızın dünyasının
birleştiği yerdir:

- **Grup etkinliği.** Grup ve oda oturumları yalıtılmış kalır (aşağıya bakın),
  ancak varsayılan DM kapsamında ana oturum bunları otomatik olarak izler.
  Etkinlikler kısa bildirimler hâlinde kuyruğa alınır — konuşma başına
  birleştirilir, hiçbir zaman mesaj başına bir uyandırma yapılmaz — ve ajan
  bunları bir sonraki çalışmasında görür: sonraki mesajınızda veya zamanlanmış
  bir Heartbeat sırasında. Ajan izlediği oturumları da okuyabildiğinden
  "aile grubunda neleri kaçırdım?" sorusu işe yarar.
- **Arka plan çalışması.** Alt ajanlar ve oluşturulan oturumlar sonuçlarını
  kendilerini başlatan oturuma bildirir; böylece ajanın Ana Sayfa'dan
  başlattığı çalışmaların sonuçları Ana Sayfa'ya döner.
- **Heartbeat'ler.** Zamanlanmış Heartbeat'ler ana oturumu hedefler; bu da
  hiçbir şey yazmadığınızda bile kuyruktaki bildirimlerin farkındalığa
  dönüşmesini sağlar.

## Sıfırlamalar ve konuşmalar arasında bellek

İlerleyen konuşma, modelin bağlam penceresiyle sınırlıdır; bu nedenle
devamlılık, çevresindeki katmanlardan gelir:

- `MEMORY.md`, ajanın düzenlenmiş uzun vadeli belleği, her yeni oturuma
  yüklenir. Günlük notlar (`memory/YYYY-MM-DD.md`) gerektiğinde aranabilir ve yakın
  tarihli olanlar bir `/new` veya `/reset` sonrasında
  yeniden hazırlanır. Compaction öncesinde ajan, kalıcı olguları günlük notlara
  aktarır; böylece uzun konuşmalar bunları fark edilmeden kaybetmez.
- **Konuşmalar arası bellek hatırlaması**, ajanın diğer özel oturumlarındaki
  içeriği hatırlamasını sağlar. Kişisel kurulumlarda — bağlama özel DM geçersiz
  kılmaları olmadan genel `session.dmScope` değerinin `main`
  olarak çözümlenmesi durumunda — varsayılan olarak etkindir; yapılandırılmış
  herhangi bir DM yalıtımı, açıkça etkinleştirmediğiniz sürece bunu kapatır.
  Bkz. [Bellek yapılandırması](/tr/reference/memory-config).

## Kalıcı geçmişe sahip ilerleyen oturum

Ana oturum, modelin tüm geçmişini tek seferde taşımasını sağlamak yerine
sıfırlamalar ve Compaction boyunca ilerler:

- Varsayılan olarak otomatik sıfırlama yoktur; Compaction, ilerleyen oturumu
  korurken etkin bağlamı sınırlar. Günlük ve boşta kalma sıfırlamaları isteğe
  bağlıdır (bkz. [Oturum yönetimi](/tr/concepts/session)). `/new` ve
  `/reset` sırasında sona eren konuşmanın son kısmı günlük bellek
  notlarına kaydedilir ve sonraki oturum yakın tarihli notları yeniden
  hazırlar. Sıfırlama yeni bir etkin oturum kimliği atar, ancak önceki SQLite
  transkriptini aynı ana oturum anahtarı altında aranabilir durumda tutar.
- Konuşma bağlam penceresine yaklaştığında Compaction, özeti çıkarır ve
  yerinde devam eder — transkript geçmişi oturum deposunda kalır.
- Oturum listeleri, arkasındaki tüm geçmiş oturum kimliklerini değil,
  geçerli etkin konuşmayı gösterir.
- Ajan başına deponun fiziksel veritabanı, WAL ve oturum yapıtları disk
  bütçesini (varsayılan 10 GB) aştığında OpenClaw, veritabanı satırlarını
  kaldırmadan önce referans verilmeyen en eski geçmişi doğrulanmış sıkıştırılmış
  bir arşive çıkarır. Etkin, yönlendirilmiş ve devam eden oturumlar hiçbir zaman
  bütçe nedeniyle kaldırılmaz.

## Bunun yerine yalıtım istediğinizde

Paylaşılan ana oturum, yalnızca sizin konuştuğunuz bir ajan için doğru
varsayılandır. Birden fazla kişi ajanınıza mesaj gönderebiliyorsa DM'leri
yalıtın:

```json5
{
  session: {
    dmScope: "per-channel-peer",
  },
}
```

Yalıtıcı bir kapsamla her gönderen kendi oturumuna sahip olur, ana oturumdan
grup izleme devre dışı bırakılır ve konuşmalar arası bellek hatırlaması
varsayılan olarak kapanır. `openclaw security audit`, birden fazla DM göndereni
algıladığında yalıtımı önerir. Kapsam matrisinin tamamı, kimlik bağlama ve rota
başına geçersiz kılmalar [Oturum yönetimi](/tr/concepts/session) ve
[Kanal yönlendirme](/tr/channels/channel-routing) bölümlerinde ele alınır.

## İlgili

- [Oturum yönetimi](/tr/concepts/session) — yönlendirme, kapsamlar, sıfırlamalar
- [Kanal yönlendirme](/tr/channels/channel-routing) — ajanların ve oturumların nasıl seçildiği
- [Bellek](/tr/concepts/memory) — kalıcı bellek katmanları
- [Çoklu ajan](/tr/concepts/multi-agent) — yalıtılmış birden fazla ajan çalıştırma
