---
read_when:
    - BlueBubbles'dan paketle birlikte sunulan iMessage pluginine geçişi planlama
    - BlueBubbles yapılandırma anahtarlarını iMessage eşdeğerlerine çevirme
    - iMessage Plugin'ini etkinleştirmeden önce imsg'yi doğrulama
summary: 'Eski BlueBubbles yapılandırmalarını paketle birlikte gelen iMessage Plugin''e taşıma: anahtar eşleme, grup izin listesi geçitleri ve geçiş doğrulaması.'
title: BlueBubbles'dan geçiş yapma
x-i18n:
    generated_at: "2026-07-26T23:09:42Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 5984ad1319b4bb3060496666bea6de663eba0105a89f82d13030c015c5df159d
    source_path: channels/imessage-from-bluebubbles.md
    workflow: 16
---

BlueBubbles desteği kaldırıldı. OpenClaw, iMessage'ı yalnızca paketle birlikte sunulan `imessage` Plugin aracılığıyla destekler; bu Plugin, [`steipete/imsg`](https://github.com/steipete/imsg) aracını JSON-RPC üzerinden çalıştırır ve BlueBubbles'ın eriştiği aynı özel API yüzeyine (`react`, `edit`, `unsend`, `reply`, `sendWithEffect`, yerel anketler, grup yönetimi, ekler) erişir. Tek bir CLI ikili dosyası; BlueBubbles sunucusunun, istemci uygulamasının ve webhook altyapısının yerini alır: REST uç noktası ve webhook kimlik doğrulaması yoktur.

Bu kılavuz, eski `channels.bluebubbles` yapılandırmalarını `channels.imessage` biçimine geçirir. Desteklenen başka bir geçiş yolu yoktur. Güncel OpenClaw'da geride kalan bir `channels.bluebubbles` bloğu etkisizdir; hiçbir çalışma zamanı bunu okumaz.

<Note>
Kısa duyuru ve operatör özeti için [BlueBubbles'ın kaldırılması ve imsg iMessage yolu](/tr/announcements/bluebubbles-imessage) sayfasına bakın.
</Note>

## Geçiş kontrol listesi

Eski BlueBubbles yapılandırmanızı zaten biliyorsanız en kısa güvenli yol:

1. Messages.app'i çalıştıran Mac'te doğrudan `imsg` değerini doğrulayın (`imsg chats`, `imsg history`, `imsg send`, `imsg rpc --help`).
2. Davranış anahtarlarını `channels.bluebubbles` konumundan `channels.imessage` konumuna kopyalayın: `dmPolicy`, `allowFrom`, `groupPolicy`, `groupAllowFrom`, `groups`, `includeAttachments`, `attachmentRoots`, `mediaMaxMb`, `textChunkLimit` ve `actions`.
3. Artık mevcut olmayan aktarım anahtarlarını kaldırın: `serverUrl`, `password`, webhook URL'leri ve BlueBubbles sunucu kurulumu.
4. Gateway, Messages Mac'te çalışmıyorsa `channels.imessage.cliPath` değerini bir SSH sarmalayıcısına ayarlayın ve uzaktan ek getirme işlemleri için `remoteHost` değerini ayarlayın.
5. `channels.imessage` özelliğini etkinleştirin, Gateway'i yeniden başlatın, ardından `openclaw channels status --probe --channel imessage` komutunu çalıştırın.
6. Bir doğrudan mesajı, izin verilen bir grubu, etkinleştirilmişse ekleri ve aracının kullanmasını beklediğiniz her özel API eylemini test edin.
7. iMessage yolu doğrulandıktan sonra BlueBubbles sunucusunu ve eski `channels.bluebubbles` yapılandırmasını silin.

## imsg ne yapar?

`imsg`, Messages için yerel bir macOS CLI'sıdır. OpenClaw, `imsg rpc` aracını bir alt süreç olarak başlatır ve stdin/stdout üzerinden JSON-RPC ile iletişim kurar. Açığa çıkarılacak bir HTTP sunucusu, webhook URL'si, arka plan daemon'u, launch agent'ı veya bağlantı noktası yoktur.

- Okuma işlemleri, salt okunur bir SQLite tanıtıcısı kullanılarak `~/Library/Messages/chat.db` kaynağından gerçekleştirilir.
- Canlı gelen mesajlar, `chat.db` dosya sistemi olaylarını yoklama yedeğiyle takip eden `imsg watch` / `watch.subscribe` kaynağından gelir.
- Normal metin ve dosya gönderimleri, Messages.app otomasyonu kullanılarak gerçekleştirilir.
- Gelişmiş eylemler, `imsg` yardımcısını Messages.app'e enjekte etmek için `imsg launch` kullanır. Okundu bilgileri, yazıyor göstergeleri, zengin gönderimler, düzenleme, gönderimi geri alma, ileti dizili yanıt, tapback'ler, anketler ve grup yönetimi bu şekilde kullanılabilir hâle gelir.
- Linux derlemeleri, kopyalanmış bir `chat.db` dosyasını inceleyebilir ancak gönderim yapamaz, canlı Mac veritabanını izleyemez veya Messages.app'i çalıştıramaz. OpenClaw iMessage için `imsg` aracını oturum açılmış Mac'te veya bu Mac'e bağlanan bir SSH sarmalayıcısı üzerinden çalıştırın.

## Başlamadan önce

1. Messages.app'i çalıştıran Mac'e `imsg` yükleyin:

   ```bash
   brew install steipete/tap/imsg
   brew update && brew upgrade imsg
   imsg --version
   imsg chats --limit 3
   ```

   Olağan yerel kurulumda OpenClaw kurulumu, oturum açılmış Messages Mac'teki `imsg` için kullanıcı onaylı bir Homebrew yüklemesi veya güncellemesi sunabilir. Manuel kurulumlar ve SSH sarmalayıcısı topolojileri operatör tarafından yönetilmeye devam eder: Homebrew güncellemesini, `imsg` aracını çalıştıracak aynı yerel veya uzak kullanıcı bağlamında yineleyin. `imsg chats`; `unable to open database file`, boş çıktı veya `authorization denied` hatasıyla başarısız olursa `imsg` aracını başlatan terminale, düzenleyiciye, Node sürecine, Gateway hizmetine veya SSH üst sürecine Tam Disk Erişimi verin ve ardından bu üst süreci yeniden açın.

2. OpenClaw yapılandırmasını değiştirmeden önce okuma, izleme, gönderme ve RPC yüzeylerini doğrulayın:

   ```bash
   imsg chats --limit 10 --json | jq -s
   imsg history --chat-id 42 --limit 10 --attachments --json | jq -s
   imsg watch --chat-id 42 --reactions --json
   imsg send --chat-id 42 --text "OpenClaw imsg test"
   imsg rpc --help
   ```

   `42` değerini, `imsg chats` kaynağındaki gerçek bir sohbet kimliğiyle değiştirin. Gönderim için Messages.app'e Otomasyon izni verilmesi gerekir. OpenClaw SSH üzerinden çalışacaksa bu komutları OpenClaw'ın kullanacağı aynı SSH sarmalayıcısı veya kullanıcı bağlamı üzerinden çalıştırın. Okuma işlemleri çalışıyor ancak gönderimler AppleEvents `-1743` hatasıyla başarısız oluyorsa Otomasyon izninin `/usr/libexec/sshd-keygen-wrapper` üzerine atanıp atanmadığını kontrol edin; [SSH sarmalayıcısı gönderimleri AppleEvents -1743 hatasıyla başarısız oluyor](/tr/channels/imessage#requirements-and-permissions-macos) bölümüne bakın.

3. Özel API köprüsünü etkinleştirin. Yanıtlar, tapback'ler, efektler, anketler, ek yanıtları ve grup eylemleri buna bağlı olduğundan OpenClaw iMessage için kesinlikle önerilir:

   ```bash
   imsg launch
   imsg status --json
   ```

   `imsg launch`, SIP'nin devre dışı bırakılmasını gerektirir (modern macOS'te ayrıca kitaplık doğrulamasının gevşetilmesi gerekir; bkz. [imsg özel API'sini etkinleştirme](/tr/channels/imessage#enabling-the-imsg-private-api)). Temel gönderim, geçmiş ve izleme işlevleri `imsg launch` olmadan çalışır; OpenClaw iMessage'ın tam eylem yüzeyi çalışmaz.

4. `channels.imessage` özelliğini etkinleştirip Gateway'i başlattıktan sonra köprüyü OpenClaw üzerinden doğrulayın:

   ```bash
   openclaw channels status --probe
   ```

   iMessage hesabı `works` bildirmelidir; `--json` ile yoklama yükü `privateApi.available: true` içerir. `false` bildirirse önce bunu düzeltin; [Yetenek algılama](/tr/channels/imessage#private-api-actions) bölümüne bakın. Yoklama için erişilebilir bir Gateway gerekir (aksi hâlde CLI yalnızca yapılandırma çıktısına geri döner) ve yalnızca yapılandırılmış, etkin hesaplar yoklanır.

5. Yapılandırmanızın anlık görüntüsünü alın:

   ```bash
   cp ~/.openclaw/openclaw.json ~/.openclaw/openclaw.json.bak
   ```

## Yapılandırma dönüşümü

iMessage ve BlueBubbles, kanal düzeyindeki davranış anahtarlarının çoğunu paylaşır. Değişenler aktarım yöntemi (REST sunucusuna karşı yerel CLI) ve grup kayıt anahtarının biçimidir.

| BlueBubbles                                                | paketle gelen iMessage                          | Notlar                                                                                                                                                                                                                                                                            |
| ---------------------------------------------------------- | ----------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `channels.bluebubbles.enabled`                             | `channels.imessage.enabled`               | Aynı semantik (blok mevcut olduğunda varsayılan `true`).                                                                                                                                                                                                                           |
| `channels.bluebubbles.serverUrl`                           | _(kaldırıldı)_                               | REST sunucusu yoktur — plugin, stdio üzerinden `imsg rpc` başlatır.                                                                                                                                                                                                                        |
| `channels.bluebubbles.password`                            | _(kaldırıldı)_                               | Webhook kimlik doğrulaması gerekmez.                                                                                                                                                                                                                                                |
| _(örtük)_                                               | `channels.imessage.cliPath`               | `imsg` yolu (varsayılan `imsg`); SSH için bir sarmalayıcı betik kullanın.                                                                                                                                                                                                                   |
| _(örtük)_                                               | `channels.imessage.dbPath`                | İsteğe bağlı Messages.app `chat.db` geçersiz kılması; belirtilmediğinde otomatik olarak algılanır.                                                                                                                                                                                                            |
| _(örtük)_                                               | `channels.imessage.remoteHost`            | `host` veya `user@host` — yalnızca `cliPath` bir SSH sarmalayıcısıysa ve eklerin SCP ile alınmasını istiyorsanız gereklidir.                                                                                                                                                                        |
| `channels.bluebubbles.dmPolicy`                            | `channels.imessage.dmPolicy`              | Aynı değerler (`pairing` / `allowlist` / `open` / `disabled`); varsayılan `pairing`.                                                                                                                                                                                                  |
| `channels.bluebubbles.allowFrom`                           | `channels.imessage.allowFrom`             | Aynı tanıtıcı biçimleri (`+15555550123`, `user@example.com`). Eşleştirme deposu onayları aktarılmaz — aşağıya bakın.                                                                                                                                                                   |
| `channels.bluebubbles.groupPolicy`                         | `channels.imessage.groupPolicy`           | Aynı değerler (`allowlist` / `open` / `disabled`); varsayılan `allowlist`.                                                                                                                                                                                                            |
| `channels.bluebubbles.groupAllowFrom`                      | `channels.imessage.groupAllowFrom`        | Aynı. Ayarlanmadığında iMessage, `allowFrom` değerine geri döner; açıkça boş bırakılmış bir `groupAllowFrom: []`, `groupPolicy: "allowlist"` kapsamında tüm grupları engeller.                                                                                                                               |
| `channels.bluebubbles.groups`                              | `channels.imessage.groups`                | `"*"` joker karakter girdisini olduğu gibi kopyalayın; grup başına girdileri sayısal iMessage `chat_id` değerine göre yeniden anahtarlayın — bkz. "Grup kayıt defteri tuzağı". `requireMention`, `tools`, `toolsBySender`, `systemPrompt` aynen aktarılır.                                                                            |
| `channels.bluebubbles.sendReadReceipts`                    | `channels.imessage.sendReadReceipts`      | Varsayılan `true`. Paketle gelen plugin ile bu yalnızca özel API yoklaması çalışır durumdayken tetiklenir.                                                                                                                                                                                        |
| `channels.bluebubbles.includeAttachments`                  | `channels.imessage.includeAttachments`    | Aynı yapı, aynı şekilde varsayılan olarak kapalı. Ekler BlueBubbles üzerinde aktarılıyorsa bunu açıkça ayarlayın — bunu yapana kadar gelen fotoğraflar/medya sessizce atılır (`Inbound message` günlük satırı olmadan).                                                                                             |
| `channels.bluebubbles.attachmentRoots`                     | `channels.imessage.attachmentRoots`       | Yerel kökler; aynı joker karakter kuralları.                                                                                                                                                                                                                                                |
| _(Geçerli değil)_                                                    | `channels.imessage.remoteAttachmentRoots` | Yalnızca SCP ile alma işlemleri için `remoteHost` ayarlandığında kullanılır.                                                                                                                                                                                                                              |
| `channels.bluebubbles.mediaMaxMb`                          | `channels.imessage.mediaMaxMb`            | iMessage üzerinde varsayılan 16 MB'dir (BlueBubbles varsayılanı 8 MB idi). Daha düşük sınırı korumak için açıkça ayarlayın.                                                                                                                                                                                  |
| `channels.bluebubbles.textChunkLimit`                      | `channels.imessage.textChunkLimit`        | Her ikisinde de varsayılan 4000'dir.                                                                                                                                                                                                                                                            |
| `channels.bluebubbles.coalesceSameSenderDms`               | _(kaldırıldı)_                               | Bu anahtarı taşımayın. `imsg` 0.13.1 ve daha yeni sürümleri, OpenClaw bunları almadan önce Apple URL önizlemesinin bölünmüş gönderimlerini birleştirir; `openclaw doctor --fix` eski bir iMessage anahtarını kaldırır.                                                                                                    |
| `channels.bluebubbles.enrichGroupParticipantsFromContacts` | _(Geçerli değil)_                                   | `imsg`, `chat.db` kaynağındaki gönderen görünen adlarını zaten sunar.                                                                                                                                                                                                                     |
| `channels.bluebubbles.actions.*`                           | `channels.imessage.actions.*`             | Aynı eylem başına açma/kapama ayarları (`reactions`, `edit`, `unsend`, `reply`, `sendWithEffect`, `renameGroup`, `setGroupIcon`, `addParticipant`, `removeParticipant`, `leaveGroup`, `sendAttachment`) ve yeni `polls`. Tümü varsayılan olarak etkindir; özel API eylemleri yine de köprüyü gerektirir. |

Çok hesaplı yapılandırmalar (`channels.bluebubbles.accounts.*`), `channels.imessage.accounts.*` biçimine bire bir çevrilir.

## Grup kayıt defteri tuzağı

Paketle gelen iMessage plugin'i art arda iki grup geçidi çalıştırır. Bir grup mesajının aracıya ulaşabilmesi için her ikisinden de geçmesi gerekir:

1. **Gönderen / sohbet hedefi izin listesi** (`channels.imessage.groupAllowFrom`) — gönderen tanıtıcısıyla veya sohbet hedefiyle (`chat_id:`, `chat_guid:`, `chat_identifier:` girdileri) eşleşir. `groupAllowFrom` ayarlanmadığında bu geçit `allowFrom` değerine geri döner; açıkça belirtilen `groupAllowFrom: []` bu geri dönüşü devre dışı bırakır ve `groupPolicy: "allowlist"` kapsamında her grup mesajını atar.
2. **Grup kayıt defteri** (`channels.imessage.groups`) — sayısal iMessage `chat_id` ile anahtarlanır:
   - `groups` bloğu yoksa (veya boşsa): 1. geçidin boş olmayan etkin bir gönderen izin listesi olduğu sürece gruplar bu geçitten geçer; erişimi gönderen filtrelemesi yönetir ve başlangıçta tümünü atma uyarısı verilmez.
   - Girdileri olan ancak `"*"` içermeyen `groups`: yalnızca listelenen `chat_id` anahtarları geçer. Herhangi bir grubu listelemek, `groupPolicy: "open"` altında bile kayıt defterini bir izin listesine dönüştürür.
   - `groups: { "*": { ... } }`: her grup bu geçitten geçer.

Taşıma tuzağı: BlueBubbles, `groups` girdilerini sohbet GUID'si / sohbet tanımlayıcısıyla anahtarlarken iMessage kayıt defteri sayısal `chat_id` ile anahtarlar. Grup başına girdilerin olduğu gibi kopyalanması, anahtarları hiçbir zaman eşleşmeyen ve boş olmayan bir kayıt defteri oluşturur; bu nedenle her grup mesajı 2. geçitte atılır. `"*"` joker karakterini olduğu gibi kopyalayın; belirli grup girdilerini `imsg chats` kaynağındaki `chat_id` değerleriyle yeniden anahtarlayın.

Her iki atma yolu da varsayılan günlük düzeyinde `warn` satırları aracılığıyla görülebilir:

- `groupPolicy: "allowlist"` ayarlandığında ve etkin grup gönderen izin listesi boş olduğunda, başlangıçta hesap başına bir kez: `imessage: groupPolicy="allowlist" for account "<id>" but no group sender allowlist is configured ...`. Gönderenleri kabul etmek için `groupAllowFrom` (veya `allowFrom`) ayarlayın; yalnızca `groups` eklemek gönderen geçidini karşılamaz.
- Kayıt defteri bir grubu attığında çalışma zamanında `chat_id` başına bir kez: `imessage: dropping group message from chat_id=<id> ... not in channels.imessage.groups allowlist`; eklenecek tam anahtarı belirtir.

DM'ler her iki durumda da çalışmaya devam eder — farklı bir kod yolu kullanırlar, bu nedenle DM başarısı grup yönlendirmesinin çalıştığını kanıtlamaz.

`groupPolicy: "allowlist"` ile gönderen kapsamlı asgari yapılandırma:

```json5
{
  channels: {
    imessage: {
      groupPolicy: "allowlist",
      groupAllowFrom: ["+15555550123", "chat_guid:any;-;..."],
    },
  },
}
```

Bu, yapılandırılmış gönderenleri herhangi bir grupta kabul eder. İzin verilen sohbetleri sınırlandırmak veya `requireMention` gibi sohbet başına seçenekleri ayarlamak için `groups` girdileri ekleyin; BlueBubbles `"*"` girdisini olduğu gibi kopyalayın ancak belirli girdileri sayısal iMessage `chat_id` değerleriyle yeniden anahtarlayın.

## Adım adım

1. Yapılandırmayı çevirin. Düzenlerken yeni bloğu devre dışı bırakın; eski `channels.bluebubbles` bloğu güncel OpenClaw tarafından yok sayılır ve başvuru amacıyla yanında kalabilir:

   ```json5
   {
     channels: {
       imessage: {
         enabled: false, // geçişe hazır olduğunuzda true yapın
         cliPath: "/opt/homebrew/bin/imsg",
         dmPolicy: "pairing",
         allowFrom: ["+15555550123"], // bluebubbles.allowFrom değerinden kopyalayın
         groupPolicy: "allowlist",
         groupAllowFrom: [], // bluebubbles.groupAllowFrom değerinden kopyalayın
         groups: { "*": { requireMention: true } }, // joker karakteri olduğu gibi kopyalayın; sohbet başına girdileri chat_id ile yeniden anahtarlayın
         // eylemler varsayılan olarak etkindir; devre dışı bırakmak için ayrı ayrı açma/kapama ayarlarını false yapın
       },
     },
   }
   ```

2. **Geçişi yapın ve yoklayın.** `channels.imessage.enabled: true` değerini ayarlayın, Gateway'i yeniden başlatın ve kanalın sağlıklı olarak bildirildiğini doğrulayın:

   ```bash
   openclaw gateway restart
   openclaw channels status --probe --channel imessage   # "works" beklenir; --json, privateApi.available: true değerini gösterir
   ```

   Yoklama, erişilebilir bir Gateway gerektirir ve yalnızca yapılandırılmış, etkin hesapları yoklar. Mac'in kendisini doğrulamak için [Başlamadan önce](#before-you-start) bölümündeki doğrudan `imsg` komutlarını kullanın.

3. **DM'leri doğrulayın.** Ajana doğrudan mesaj gönderin; yanıtın ulaştığını doğrulayın.

4. **Grupları ayrı olarak doğrulayın.** DM'ler ve gruplar farklı kod yollarını kullanır — DM başarısı, grupların yönlendirildiğini kanıtlamaz. İzin verilen bir grup sohbetinde mesaj gönderin ve yanıtın ulaştığını doğrulayın. Grup sessiz kalırsa (ajan yanıtı ve hata yoksa), yukarıdaki "Grup kayıt defteri tuzağı" bölümünde belirtilen iki `warn` satırı için gateway günlüğünü kontrol edin. Başlangıç uyarısı, geçerli gönderen izin listesinin boş olduğu anlamına gelir; `chat_id` başına verilen uyarı ise doldurulmuş bir `groups` kayıt defterinin ilgili sohbeti içermediği anlamına gelir.

5. **Eylem yüzeyini doğrulayın.** Eşleştirilmiş bir DM'den ajandan tepki vermesini, düzenlemesini, göndermeyi geri almasını, yanıtlamasını, fotoğraf göndermesini ve (bir grupta) grubun adını değiştirmesini ya da katılımcı ekleyip kaldırmasını isteyin. Her eylem Messages.app içinde yerel olarak gerçekleşmelidir. Herhangi bir eylem `iMessage <action> requires the imsg private API bridge` hatası verirse `imsg launch` komutunu yeniden çalıştırın ve `openclaw channels status --probe` ile yenileyin.

6. iMessage DM'leri, grupları ve eylemleri doğrulandıktan sonra **BlueBubbles sunucusunu ve `channels.bluebubbles` bloğunu kaldırın**. OpenClaw, `channels.bluebubbles` değerini okumaz.

## Bir bakışta eylem eşdeğerliği

| Eylem                                               | eski BlueBubbles   | paketlenmiş iMessage                                                           |
| --------------------------------------------------- | ------------------ | ----------------------------------------------------------------------------- |
| Metin gönderme / SMS'e geri dönme                   | ✅                 | ✅                                                                            |
| Medya gönderme (fotoğraf, video, dosya, ses)        | ✅                 | ✅                                                                            |
| Konu dizili yanıt (`reply_to_guid`)              | ✅                 | ✅ ([#51892](https://github.com/openclaw/openclaw/issues/51892) kapatıldı)     |
| Tapback (`react`)                        | ✅                 | ✅                                                                            |
| Düzenleme / göndermeyi geri alma (macOS 13+ alıcılar) | ✅               | ✅                                                                            |
| Ekran efektiyle gönderme                            | ✅                 | ✅ ([#9394](https://github.com/openclaw/openclaw/issues/9394) sorununun bir bölümü kapatıldı) |
| Zengin metin kalın / italik / altı çizili / üstü çizili | ✅             | ✅ (attributedBody aracılığıyla tür belirtilmiş çalışma biçimlendirmesi)       |
| Yerel Messages anketleri (oluşturma ve oy verme)    | ❌                 | ✅ (`actions.polls`; yerel görüntüleme için alıcılarda iOS/macOS 26+ gerekir) |
| Grubu yeniden adlandırma / grup simgesini ayarlama  | ✅                 | ✅                                                                            |
| Katılımcı ekleme / kaldırma, gruptan ayrılma        | ✅                 | ✅                                                                            |
| Okundu bilgileri ve yazıyor göstergesi              | ✅                 | ✅ (özel API yoklamasına bağlıdır)                                            |
| Apple URL önizlemesi için bölünmüş gönderim birleştirme | ✅             | ✅ (`imsg` 0.13.1 ve daha yeni sürümler tarafından üst akışta işlenir; OpenClaw ayarı yoktur) |
| Yeniden başlatma sonrasında gelen iletileri kurtarma | ✅                | ✅ (otomatik: `since_rowid` yeniden oynatma + GUID tekilleştirme; yerel kurulumda daha geniş pencere) |

iMessage, gateway kapalıyken kaçırılan iletileri kurtarır: başlangıçta `imsg watch.subscribe` `since_rowid` aracılığıyla son iletilen rowid'den itibaren yeniden oynatır, GUID'ye göre tekilleştirir ve eski birikim yaş sınırı, Push boşaltımındaki "birikim bombasını" engeller. Bu işlem `imsg` RPC bağlantısı üzerinden yürütüldüğünden uzak SSH `cliPath` kurulumlarında da çalışır; yerel kurulumlar `chat.db` değerini okuyabildiğinden daha geniş bir kurtarma penceresi elde eder. Bkz. [Köprü veya gateway yeniden başlatıldıktan sonra gelen iletileri kurtarma](/tr/channels/imessage#inbound-recovery-after-a-bridge-or-gateway-restart).

## Eşleştirme, oturumlar ve ACP bağlamaları

- **İzin listeleri tanıtıcıya göre aktarılır.** `channels.imessage.allowFrom`, BlueBubbles'ın kullandığı aynı `+15555550123` / `user@example.com` dizelerini tanır — bunları aynen kopyalayın.
- **Eşleştirme deposu onayları aktarılmaz.** Eşleştirme deposu kanal başınadır ve eski BlueBubbles deposunu hiçbir şey taşımaz. Yalnızca eşleştirme yoluyla onaylanan gönderenler iMessage altında bir kez daha eşleşmelidir veya tanıtıcılarını `allowFrom` listesine eklemeniz gerekir.
- **Oturumlar**, ajan + sohbet başına kapsamlandırılmış olarak kalır. DM'ler varsayılan `session.dmScope=main` altında ajanın ana oturumunda birleştirilir; grup oturumları her `chat_id` (`agent:<agentId>:imessage:group:<chat_id>`) için ayrı tutulur. BlueBubbles oturum anahtarları altındaki eski konuşma geçmişi iMessage oturumlarına aktarılmaz.
- `match.channel: "bluebubbles"` değerine başvuran **ACP bağlamaları**, `"imessage"` olarak değiştirilmelidir. `match.peer.id` biçimleri (`chat_id:`, `chat_guid:`, `chat_identifier:`, yalın tanıtıcı) aynıdır.

## Geri dönüş kanalı yoktur

Geri dönülebilecek desteklenen bir BlueBubbles çalışma zamanı yoktur. iMessage doğrulaması başarısız olursa `channels.imessage.enabled: false` değerini ayarlayın, Gateway'i yeniden başlatın, `imsg` engelini giderin ve geçişi yeniden deneyin.

Yanıt önbelleği SQLite Plugin durumunda bulunur. `openclaw doctor --fix`, mevcut olduğunda eski `imessage/reply-cache.jsonl` yan dosyasını içe aktarır ve arşivler.

## İlgili içerikler

- [BlueBubbles'ın kaldırılması ve imsg iMessage yolu](/tr/announcements/bluebubbles-imessage) — kısa duyuru ve operatör özeti.
- [iMessage](/tr/channels/imessage) — `imsg launch` kurulumu ve yetenek algılama dâhil eksiksiz iMessage kanal başvurusu.
- `/channels/bluebubbles` — bu geçiş kılavuzuna yönlendiren eski URL.
- [Eşleştirme](/tr/channels/pairing) — DM kimlik doğrulaması ve eşleştirme akışı.
- [Kanal Yönlendirme](/tr/channels/channel-routing) — gateway'in giden yanıtlar için kanalı nasıl seçtiği.
