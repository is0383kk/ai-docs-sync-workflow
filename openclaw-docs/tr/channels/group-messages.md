---
read_when:
    - WhatsApp gruplarını özel olarak yapılandırma
    - WhatsApp etkinleştirme modlarını değiştirme (`mention` ile `always` karşılaştırması)
    - WhatsApp grup oturumu anahtarlarını veya bekleyen ileti bağlamını ayarlama
sidebarTitle: WhatsApp groups
summary: WhatsApp grup mesajı işleme — etkinleştirme, izin listeleri, oturumlar ve bağlam ekleme
title: WhatsApp grup mesajları
x-i18n:
    generated_at: "2026-07-26T22:34:30Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 7325dd3ae64d7abca8c1de0504f294ae280394fa5dd336d2532c5eaefcb03828
    source_path: channels/group-messages.md
    workflow: 16
---

Kanallar arası gruplar modeli (Discord, iMessage, Matrix, Microsoft Teams, QQBot, Signal, Slack, Telegram, WhatsApp, Zalo) için [Gruplar](/tr/channels/groups) bölümüne bakın. Bu sayfa, söz konusu modele ek olarak WhatsApp'a özgü davranışı ele alır: etkinleştirme, grup izin listeleri, grup başına oturum anahtarları ve bekleyen mesaj bağlamının eklenmesi.

Amaç: OpenClaw'ın WhatsApp gruplarında bulunmasını, yalnızca kendisine seslenildiğinde etkinleşmesini ve bu yazışmayı kişisel doğrudan mesaj oturumundan ayrı tutmasını sağlamak.

<Note>
`agents.entries.*.groupChat.mentionPatterns`, diğer kanalların bahsetme geçidiyle paylaşılır. Çok aracılı kurulumlarda bunu aracı başına ayarlayın veya genel geri dönüş olarak `messages.groupChat.mentionPatterns` kullanın. Hiçbiri ayarlanmamışsa kalıplar, aracının kimlik adından/emojisinden türetilir.
</Note>

## Davranış

- Etkinleştirme modları: `mention` (varsayılan) veya `always`. `mention` bir seslenme gerektirir: gerçek bir WhatsApp @-bahsetmesi (`mentionedJids`), yapılandırılmış bir regex kalıbı, metnin herhangi bir yerindeki botun E.164 rakamları veya botun mesajlarından birine alıntılı yanıt (paylaşılan numaralı kendi kendine sohbet kurulumları hariç). `always`, aracıyı her mesajda etkinleştirir; ancak eklenen grup istemi, yalnızca değer kattığında yanıt vermesini, aksi takdirde tam sessizlik belirteci `NO_REPLY` değerini (büyük/küçük harfe duyarsız) döndürmesini söyler. Varsayılanlar yapılandırmadan (`channels.whatsapp.groups` `requireMention`) gelir ve `/activation` aracılığıyla grup başına geçersiz kılınabilir.
- Grup izin listesi: `channels.whatsapp.groups` ayarlandığında yalnızca listelenen grup JID'leri kabul edilir (tümüne izin vermek için `"*"` değerini ekleyin); listelenmemiş gruplardan gelen mesajlar, günlükte bir ipucuyla birlikte bırakılır.
- Grup ilkesi: `channels.whatsapp.groupPolicy`, grup mesajlarının kabul edilip edilmediğini (`open|disabled|allowlist`) denetler. `allowlist`, `channels.whatsapp.groupAllowFrom` değerini kullanır (geri dönüş: açıkça belirtilen `channels.whatsapp.allowFrom`). Varsayılan `allowlist` değeridir (gönderenleri ekleyene kadar engellenir).
- Grup başına oturumlar: oturum anahtarları `agent:<agentId>:whatsapp:group:<jid>` biçimindedir (varsayılan olmayan hesaplara `:thread:whatsapp-account-<accountId>` eklenir); böylece `/verbose on`, `/trace on` veya `/think high` gibi yönergeler (bağımsız mesajlar olarak gönderildiğinde) yalnızca o grupla sınırlı olur; kişisel doğrudan mesaj durumu değişmeden kalır.
- Bağlam ekleme: bir çalıştırmayı _tetiklememiş_, **yalnızca bekleyen** grup mesajları (varsayılan 50), `[Chat messages since your last reply - for context]` altında öne eklenir; tetikleyen satır ise `[Current message - respond to this]` altında yer alır. Bekleme penceresi çalıştırmadan sonra temizlenir; zaten oturumda bulunan mesajlar yeniden eklenmez.
- Gönderen ilişkilendirmesi: her grup satırı, ileti zarfının içinde gönderen etiketini taşır; örneğin `[WhatsApp <groupJid> <timestamp>] Alice (+447700900123): text`. Gönderen kimliği ile grup konusu/üyeleri de güvenilmeyen konuşma meta verileri bloğunda birlikte aktarılır.
- Geçici/bir kez görüntülenebilir: metin/bahsetmeler çıkarılmadan önce sarmalayıcılar açılır; dolayısıyla içlerindeki seslenmeler yine de tetikleme yapar.
- Grup sistem istemi: bir grup oturumunun ilk turunda (ve `/activation` modu değiştirdikten sonraki herhangi bir turda), sistem istemine etkinleştirme yönlendirmesi (`Activation: trigger-only ...` veya `Activation: always-on ...`, ayrıca "belirli gönderene hitap et") eklenir. Kalıcı grup sohbeti teslimat yönlendirmesi ("Bir WhatsApp grup sohbetindesiniz...") her zaman dahil edilir.

## Yapılandırma örneği (WhatsApp)

WhatsApp, görsel `@` işaretini metin gövdesinden çıkarsa bile görünen adla seslenmelerin çalışmasını sağlayın:

```json5
{
  channels: {
    whatsapp: {
      groups: {
        "*": { requireMention: true },
      },
      historyLimit: 50, // bekleyen grup bağlamı penceresi (varsayılan 50)
    },
  },
  agents: {
    entries: {
      main: {
        groupChat: {
          mentionPatterns: ["@?openclaw", "\\+?15555550123"],
        },
      },
    },
  },
}
```

Notlar:

- Regex'ler büyük/küçük harfe duyarsızdır ve diğer yapılandırma regex yüzeyleriyle aynı güvenli regex korumalarını kullanır; geçersiz kalıplar ve güvenli olmayan iç içe yinelemeler yok sayılır.
- Birisi kişiye dokunduğunda WhatsApp, standart bahsetmeleri yine `mentionedJids` aracılığıyla gönderir; bu nedenle numara geri dönüşüne nadiren ihtiyaç duyulur, ancak yararlı bir güvenlik ağıdır.
- Bekleyen bağlam penceresi şu sırayla çözümlenir: `channels.whatsapp.accounts.<id>.historyLimit` → `channels.whatsapp.historyLimit` → `messages.groupChat.historyLimit` → 50.

### Etkinleştirme komutu (yalnızca sahip)

Grup sohbeti komutunu kullanın:

- `/activation mention`
- `/activation always`

Bunu yalnızca sahip numaraları (`channels.whatsapp.allowFrom` içinden veya ayarlanmamışsa botun kendi E.164 numarası) değiştirebilir; başka herhangi bir kişiden gelen `/activation` yok sayılır ve yalnızca bağlam olarak saklanır. Geçerli etkinleştirme modunu görmek için grupta `/status` değerini bağımsız bir mesaj olarak gönderin.

## Kullanım

1. WhatsApp hesabınızı (OpenClaw'ı çalıştıran hesap) gruba ekleyin.
2. `@openclaw ...` deyin (veya numarayı ekleyin). `groupPolicy: "open"` ayarlanmadıkça yalnızca izin listesindeki gönderenler bunu tetikleyebilir.
3. Aracı istemi, doğru kişiye hitap edebilmesi için bekleyen grup bağlamını ve gönderen etiketli satırları içerir.
4. Oturum yönergeleri (`/verbose on`, `/trace on`, `/think high`, `/new` veya `/reset`, `/compact`) yalnızca o grubun oturumuna uygulanır; kaydedilmeleri için bunları bağımsız mesajlar olarak gönderin. Kişisel doğrudan mesaj oturumunuz bağımsız kalır.

## Test / doğrulama

- Elle hızlı kontrol:
  - Grupta bir `@openclaw` seslenmesi gönderin ve gönderenin adına atıfta bulunan bir yanıt geldiğini doğrulayın.
  - İkinci bir seslenme gönderin ve geçmiş bloğunun dahil edildiğini, ardından sonraki turda temizlendiğini doğrulayın.
- `from: <groupJid>` değerini ve gönderen etiketli gövdeyi gösteren `inbound web message` girdileri için Gateway günlüklerini (`--verbose` ile çalıştırın) denetleyin.

## Bilinen hususlar

- Heartbeat'ler aracının ana oturumunda çalışır; grup oturumlarında hiçbir zaman heartbeat çalıştırması yapılmaz.
- Yankı bastırma, botun teslim edilen kendi mesajlarının yeniden tetiklememesi için birleştirilmiş istemi (geçmiş + geçerli mesaj) oturum başına hatırlar; aynı mesaj topluluğunun yinelenmesi yankı olarak atlanabilir.
- Oturum deposu girdileri, aracı başına SQLite oturum deposunda `agent:<agentId>:whatsapp:group:<jid>` olarak görünür; girdinin olmaması yalnızca grubun henüz bir çalıştırmayı tetiklemediği anlamına gelir.
- Yazıyor göstergeleri `agents.entries.*.typingMode` / `agents.defaults.typingMode` ayarlarını izler. Görünür yanıtlar yalnızca mesaj aracı modunu kullanacak şekilde etkinleştirildiğinde, varsayılan olarak yazıyor göstergesi hemen başlar; böylece otomatik bir son yanıt gönderilmese bile grup üyeleri aracının çalıştığını görebilir. Açıkça belirtilen yazıyor modu yapılandırması yine önceliklidir.

## İlgili

- [Gruplar](/tr/channels/groups)
- [Kanal yönlendirme](/tr/channels/channel-routing)
- [Yayın grupları](/tr/channels/broadcast-groups)
