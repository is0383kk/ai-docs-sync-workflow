---
read_when:
    - İnsanlar veya diğer agent'lar bir oturumu kendilerinden habersiz değiştirdiğinde agent'ların bunu fark etmesini istiyorsunuz
    - Durum değişikliği bildirimleri, izleme imleçleri veya session_status changesSince ile ilgili hata ayıklıyorsunuz.
    - Üst ajanların alt oturumlarla nasıl eşzamanlı kaldığını anlamak istiyorsunuz
sidebarTitle: Session state awareness
summary: 'Kalıcı oturum durumu sinyal günlüğü: durum sürümleri, izleyiciler, eski durum bildirimleri ve uzlaştırma'
title: Oturum durumu farkındalığı
x-i18n:
    generated_at: "2026-07-26T23:19:14Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: bb4126a0802e1ca4418f225c792490493a78886089b81c3b4567f72090ce34f4
    source_path: concepts/session-state.md
    workflow: 16
---

Birkaç oturum aynı sorun üzerinde çalıştığında — bir yönetici alt oturumlara görev devrettiğinde, bir insan doğrudan bir çalışan oturumuna girdiğinde, iki aracı [`sessions_send`](/tr/concepts/session-tool) üzerinden koordinasyon kurduğunda — her oturum diğerleri hakkında varsayımlar oluşturur. Başka bir aktör müdahale ettiği anda bu varsayımlar güncelliğini yitirir. Oturum durumu farkındalığı, müdahaleyi algılayan, etkilenen oturuma bunu bir kez bildiren ve işlem yapmadan önce düşük maliyetle güncel durumu öğrenmesini sağlayan mekanizmadır.

Üç bileşen birlikte çalışır:

1. Bir **kalıcı sinyal günlüğü**, oturum başına seçili durum değişikliklerini kaydeder.
2. **İzleyiciler**, hedef başına imleçleri tutar ve birleştirilmiş tek bir eski durum bildirimi alır.
3. **Uzlaştırma**, `changesSince` ile `session_status` üzerinden kesin farkı alır.

## Sinyal günlüğü

OpenClaw, izlenen bir oturumda önemli bir değişiklik olduğunda paylaşılan durum veritabanına (`session_state_events`) türü belirlenmiş bir olay ekler. Olaylar meta veriler ve tek satırlık bir özet taşır; mesaj içeriğini asla taşımaz.

| Tür                    | Kaydedildiği durum                                      | İzleyicileri bilgilendirir |
| ---------------------- | ------------------------------------------------------- | -------------------------- |
| `human_direct_message` | Bir insan doğrudan izlenen bir oturuma ileti gönderir   | Evet                       |
| `upstream_missing`     | Benimsenmiş bir oturumun yukarı akış kaynağı kaybolur    | Evet                       |
| `goal_changed`         | Oturumun hedef durumu oluşturulur, güncellenir veya temizlenir | Evet               |
| `child_spawned`        | Bir alt aracı veya ACP alt oturumu oluşturulur           | Hayır (imleci başlatır)    |
| `run_completed`        | Bir alt çalıştırma başarıyla sona erer                   | Hayır (yalnızca günlük)    |
| `run_failed`           | Bir alt çalıştırma başarısız olur, zaman aşımına uğrar veya iptal edilir | Hayır (yalnızca günlük) |
| `compacted`            | Oturumun geçmişi sıkıştırılır                            | Hayır (yalnızca günlük)    |
| `adopted`              | Bir katalog oturumu OpenClaw'a benimsenir                | Hayır (yalnızca günlük)    |

Her olay, aktörünü (`human`, `agent` veya `system`) belirtir. İptal edilen ve zaman aşımına uğrayan alt çalıştırmalar, kesin sonuç (`cancelled`, `timeout` veya `error`) olay yükünde korunarak başarısızlık olarak kaydedilir.

Bir oturumun **durum sürümü**, budama sonrasında da korunan kalıcı oturum başına başlıkta izlenen, günlüğündeki en yüksek sıra numarasıdır. Bir oturum değişiklikleri günlüğe kaydettiğinde `sessions_list` satırları `stateVersion` değerini içerir; `session_status` bunu her zaman bildirir.

Yalnızca günlüğe kaydedilen türler bildirim için değil, uzlaştırma geçmişi için bulunur: olağan alt çalıştırma tamamlanma teslimi [alt aracı duyurularının](/tr/tools/subagents) sorumluluğunda kalır ve sinyal günlüğü bunu asla yinelemez.

## İzleyiciler

İzleyici, bir hedef üzerinde imleç (`session_watch_cursors`) tutan oturumdur. İmleçler iki kaynaktan gelir:

- **Örtük (oluşturma kenarları).** Bir oturum alt aracı veya ACP alt oturumu oluşturduğunda, üst oturumun imleci alt oturumun oluşturulma sürümünde otomatik olarak başlatılır. Üst oturumlar hiçbir zaman elle abone olmaz.
- **Açık (`sessions_send watch: true`).** Herhangi bir koordinatör, oluşturmadığı bir hedefi izleyebilir: `sessions_send` üzerinde `watch: true` iletin ve gönderim başarıyla yönlendirildikten sonra gönderen, mesajı gerçekten alan oturumun izleyicisi olarak kaydedilir. Kayıt, hedefin geçerli durum sürümünde başlar; önceki geçmiş hiçbir zaman bildirim oluşturmaz. Parametre ayarlandığında araç sonucu `watched: true|false` değerini bildirir.

İzleyici kimliği, aracı nitelemesi içeren bir oturum anahtarı olmalıdır. `session.scope="global"` altında paylaşılan `global` anahtarı aracılar arasında belirsizdir; bu nedenle bu tür oturumlar kalıcı günlüğü ve `changesSince` değerini alır ancak proaktif bildirim almaz.

İzlemeler kendilerini temizler: imleç satırlarının süresi sinyal günlüğü saklama süresiyle birlikte dolar, izleyici oturumu sıfırlandığında kaldırılır ve iki oturumdan herhangi biriyle birlikte silinir. v1'de izlemeyi bırakma eylemi yoktur.

Oturum kataloğundan benimsenen izlenen oturumlar, sabit bir sıklıkta doğrudan yukarı akış insan etkinliği açısından denetlenir. Algılanan etkinlik, diğer doğrudan insan iletileriyle aynı sinyal günlüğüne ve izleyici akışına girer.

Benimsenmiş bir oturumun yukarı akış kaynağı harici olarak silinirse art arda üç başarısız denetim (yaklaşık üç izleme adımı), izleyicileri için tek bir `upstream_missing` sinyali oluşturur ve yukarı akış bağlantısını kaldırır. Katalog oturumunu yeniden sürdürmek yeni bir bağlantı oluşturur.

## Bildirimler: çok değil, bir tane

Bildirime uygun bir olay gerçekleştiğinde ve izleyicinin imleci geride olduğunda izleyici, bir sonraki iletisinde tek bir sistem bildirimi alır:

```
"agent:main:subagent:child" oturumu değişti (başka aktör). İşlem yapmadan önce uzlaştırın: session_status sessionKey "agent:main:subagent:child" changesSince 12.
```

Ana oturum izleyicileri ayrıca bir Heartbeat uyandırmasıyla hemen uyandırılır; iç içe alt aracı izleyicileri bildirimi bir sonraki iletilerinde alır.

Protokol özellikle istenmeyen bildirimleri önleyecek şekilde tasarlanmıştır:

- **İzleyici/hedef çifti başına bir bekleyen bildirim.** Bildirim metni bekleme sırasında bayt düzeyinde sabit kalır ve sistem olayı kuyruğu bunu tekilleştirir; böylece aynı hedefteki yirmi hızlı değişiklik bile izleyicinin isteminde yalnızca tek bir satır oluşturur.
- **Dondurulmuş filigran.** İmleç, bir bildirim kuyruğa alındığında bildirilen konumunu dondurur. Sonraki önemli olaylar yalnızca önemli değişiklik filigranını ilerletir; yeniden bildirim oluşturmaz.
- **Boşaltmada onaylama, yalnızca araya giren çalışma için yeniden açma.** İzleyicinin iletisi bildirimi tükettiğinde imleç ilerler. Kuyruğa alma ile boşaltma arasında başka önemli olaylar geldiyse kalan bölüm için tam olarak bir yeni bildirim açılır.
- **Kendi kendini engelleme.** Bir izleyici, kendisinin neden olduğu olaylar hakkında hiçbir zaman bilgilendirilmez.
- **Yeniden başlatma kurtarması.** Bekleyen bildirimler bellek içi bir kuyrukta bulunur; başlangıç taraması, bir Gateway yeniden başlatıldıktan sonra bunları kalıcı imleçlerden yeniden oluşturur.

## Uzlaştırma

Bildirim, izleyiciye tam olarak ne yapması gerektiğini söyler. `changesSince: <version>` ile `session_status`, herhangi bir imleci ilerletmeden bu sürümden sonraki türü belirlenmiş olayları (en fazla 200) döndürür:

```json
{
  "stateVersion": 19,
  "stateChanges": {
    "events": [
      {
        "sequence": 14,
        "kind": "human_direct_message",
        "actorType": "human",
        "summary": "telegram üzerinden insan mesajı"
      },
      { "sequence": 19, "kind": "goal_changed", "actorType": "human", "summary": "hedef güncellendi" }
    ],
    "historyGap": false
  }
}
```

`historyGap: true`, istenen sürümün saklanan geçmişten daha eski olduğu anlamına gelir; yanıtı kesin bir fark olarak değerlendirmek yerine tüm oturum durumunu (`sessions_history`, `session_status`) yenileyin. Boşluk sinyali kesindir: sıra numarası aritmetiğinden çıkarılmaz, oturum başına budanmış bir filigrandan gelir.

## Depolama ve sınırlar

Geçmiş, paylaşılan durum veritabanında tutulur ve 30 gün ile 50.000 satırla sınırlandırılır; oturum başına başlıklar budama sonrasında monoton kalır. Kayıt işlemi en iyi çaba esasına dayanır; başarısız bir ekleme günlüğe kaydedilir ve kaynak iletiyi asla başarısız kılmaz. Bu nedenle `stateVersion` işlemsel bir değişiklik verisi yakalama sürümü değil, sinyal günlüğü başlığıdır.

Geçerli sınırlar:

- Bildirim teslimi, paylaşılan durum veritabanının tek bir Gateway işlemi tarafından yönetildiğini varsayar. Birden fazla Gateway kalıcı günlüğü ve `changesSince` değerini paylaşır ancak v1, işlemler arasında bildirim göndermez.
- Compaction olayları, gömülü çalışma zamanının Compaction sahiplerini kapsar; yalnızca yerel altyapıda gerçekleşen Compaction tam olarak günlüğe kaydedilmez.
- İptal edilen sonuç yükü ayrıntısı şu anda ACP alt çalıştırmaları tarafından üretilir; yerel alt aracı iptalleri genel başarısızlık olarak gösterilir.
- Yukarı akış kendi yankısını algılama, normalleştirilmiş kullanıcı metnini karşılaştırır. Oturumun OpenClaw tarafındaki en son 10 kullanıcı mesajından biriyle eşleşen harici bir istem, kendi yankısı olarak değerlendirilir.
- Her sıklık için 1 MiB tarama sınırından büyük tek bir yerel Claude JSONL satırı, v1'de söz konusu oturumun imlecini engeller; sınıflandırılmamış baytlar hiçbir zaman atlanmaz.
- Eşleştirilmiş Node Claude denetimleri, her sıklıkta en son 50 döküm öğesini sınıflandırır. Daha büyük yoğunluklar v1 tarama penceresinin dışında kalabilir.
- Eşleştirilmiş Node Claude geçmiş okumaları kesin bir ileti dizisi bulunamadı sonucu sunmadığından, uzaktaki Claude silme işlemleri v1'de `upstream_missing` olarak sınıflandırılmaz.
- Benimsenmemiş katalog oturumları v1'de farkındalık katmanının dışında kalır.
- Bu özellikten önce benimsenen oturumlar yukarı akış bağlantısı taşımaz; yukarı akış izlemesini başlatmak için bunları katalogdan bir kez sürdürün.
- Yukarı akış bağlantıları, benimsenen her oturum anahtarının tek bir sahip aracıyla eşleştiğini varsayar (benimseme, varsayılan depo aracısını kullanır). Aynı harici ileti dizisinin birden fazla aracı tarafından benimsenmesi v1'de izlenmez.

## İlgili

- [Oturum araçları](/tr/concepts/session-tool) — `sessions_send`, `session_status`, `sessions_list`
- [Alt aracılar](/tr/tools/subagents) — oluşturma kenarları ve tamamlanma duyuruları
- [Heartbeat](/tr/gateway/heartbeat) — kuyruğa alınan bildirimlerin ana oturumları nasıl uyandırdığı
- [Oturum yönetimi](/tr/concepts/session) — oturum anahtarları, kapsamlar, yaşam döngüsü
