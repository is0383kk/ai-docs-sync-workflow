---
read_when:
    - Telegram özellikleri veya webhook'lar üzerinde çalışma
summary: Telegram bot desteğinin durumu, özellikleri ve yapılandırması
title: Telegram
x-i18n:
    generated_at: "2026-07-26T23:31:06Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: f34067478f4a5a71ed8f18503234b4cfcf573ac740aa887b65d13d0e1f09ba54
    source_path: channels/telegram.md
    workflow: 16
---

Bot DM'leri ve gruplar için grammY üzerinden üretime hazırdır. Uzun yoklama varsayılan aktarımdır; webhook modu isteğe bağlıdır.

<CardGroup cols={3}>
  <Card title="Eşleştirme" icon="link" href="/tr/channels/pairing">
    Telegram için varsayılan DM ilkesi eşleştirmedir.
  </Card>
  <Card title="Kanal sorunlarını giderme" icon="wrench" href="/tr/channels/troubleshooting">
    Kanallar arası tanılama ve onarım çalışma kitapları.
  </Card>
  <Card title="Gateway yapılandırması" icon="settings" href="/tr/gateway/configuration">
    Eksiksiz kanal yapılandırma kalıpları ve örnekleri.
  </Card>
</CardGroup>

## Hızlı kurulum

<Steps>
  <Step title="BotFather'da bot belirtecini oluşturun">
    Her iki akışın sonunda da OpenClaw'a yapıştıracağınız bir belirteç elde edilir — birini seçin:

    - **Sohbet akışı**: Telegram'ı açın, **@BotFather** ile sohbet edin (kullanıcı adının tam olarak `@BotFather` olduğunu doğrulayın), `/newbot` komutunu çalıştırın, istemleri izleyin ve belirteci kaydedin.
    - **Web akışı**: [BotFather'ın web uygulamasını](https://t.me/BotFather?startapp) açın — [web.telegram.org](https://web.telegram.org) dahil her Telegram istemcisinde çalışır — kullanıcı arayüzünde botu oluşturun ve belirtecini kopyalayın.

  </Step>

  <Step title="Belirteci ve DM ilkesini yapılandırın">

```json5
{
  channels: {
    telegram: {
      enabled: true,
      botToken: "123:abc",
      dmPolicy: "pairing",
      groups: { "*": { requireMention: true } },
    },
  },
}
```

    Ortam değişkeni geri dönüşü: `TELEGRAM_BOT_TOKEN` (yalnızca varsayılan hesap; adlandırılmış hesaplar `botToken` veya `tokenFile` kullanmalıdır).
    Telegram, `openclaw channels login telegram` kullanmaz; belirteci yapılandırmada/ortam değişkeninde ayarlayın, ardından Gateway'i başlatın.

  </Step>

  <Step title="Gateway'i başlatın ve ilk DM'yi onaylayın">

```bash
openclaw gateway
openclaw pairing list telegram
openclaw pairing approve telegram <CODE>
```

    Eşleştirme kodlarının süresi 1 saat sonra dolar.

  </Step>

  <Step title="Botu bir gruba ekleyin">
    Botu grubunuza ekleyin, ardından grup erişimi için gereken iki kimliği alın:

    - `allowFrom` / `groupAllowFrom` için Telegram kullanıcı kimliğiniz
    - `channels.telegram.groups` altında anahtar olarak Telegram grup sohbeti kimliği

    Grup sohbeti kimliğini `openclaw logs --follow`, iletilen kimlikleri gösteren bir bot veya Bot API `getUpdates` üzerinden alın. Gruba izin verildikten sonra `/whoami@<bot_username>`, kullanıcı ve grup kimliklerini doğrular.

    `-100` ile başlayan negatif süper grup kimlikleri, grup sohbeti kimlikleridir. Bunlar `groupAllowFrom` altına değil, `channels.telegram.groups` altına yazılır.

  </Step>
</Steps>

<Note>
Belirteç çözümlemesi hesaba duyarlıdır: `tokenFile`, `botToken` değerine; o da ortam değişkenine üstün gelir ve yapılandırma her zaman `TELEGRAM_BOT_TOKEN` değerine üstün gelir (bu değer yalnızca varsayılan hesap için çözümlenir). Başarılı bir başlangıcın ardından OpenClaw, yeniden başlatmalarda fazladan bir `getMe` çağrısını atlamak için bot kimliğini en fazla 24 saat önbelleğe alır; belirtecin değiştirilmesi veya kaldırılması bu önbelleği temizler.
</Note>

## Telegram tarafındaki ayarlar

<AccordionGroup>
  <Accordion title="Gizlilik modu ve grup görünürlüğü">
    Telegram botları varsayılan olarak **Privacy Mode** kullanır; bu, hangi grup mesajlarını alabileceklerini sınırlar.

    Tüm grup mesajlarını görmek için şunlardan birini yapın:

    - `/setprivacy` üzerinden gizlilik modunu devre dışı bırakın veya
    - botu grup yöneticisi yapın.

    Gizlilik modunu değiştirdikten sonra Telegram'ın değişikliği uygulaması için botu her gruptan kaldırıp yeniden ekleyin.

  </Accordion>

  <Accordion title="Grup izinleri">
    Yönetici durumu Telegram grup ayarlarından denetlenir. Yönetici botlar tüm grup mesajlarını alır; bu, sürekli etkin grup davranışı için yararlıdır.
  </Accordion>

  <Accordion title="Yararlı BotFather seçenekleri">

    - `/setjoingroups` — gruplara eklemeye izin ver/reddet
    - `/setprivacy` — grup görünürlüğü davranışı

    Sohbet komutları yerine kullanıcı arayüzünü tercih ediyorsanız aynı ayarlar [BotFather'ın web uygulamasında](https://t.me/BotFather?startapp) da bulunur.

  </Accordion>
</AccordionGroup>

## Kontrol Paneli Mini Uygulaması

OpenClaw kontrol panelini Telegram içinde açmak için botla yapılan bir DM'de `/dashboard` komutunu çalıştırın.

Gereksinimler:

- Yayımlanmış HTTPS Mini App URL'si için `gateway.tailscale.mode: "serve"` veya `"funnel"`.
- Sayısal Telegram kullanıcı kimliğiniz, seçilen hesabın etkin `allowFrom` listesinde veya `commands.ownerAllowFrom` içinde bulunmalıdır.
- DM kullanın. Gruplarda `/dashboard`, `open this in a DM with the bot` ile yanıt verir ve düğme göndermez.
- Docker kurulumları: Serve/Funnel modlarında Gateway'in `tailscaled` yanında geri döngü adresine bağlanması gerekir; yayımlanmış bağlantı noktalarıyla köprü ağ iletişimi bunu sağlayamaz. Gateway konteynerini `network_mode: host` ile çalıştırın ve ana makinedeki `tailscaled` soketini (`/var/run/tailscale`) ve `tailscale` CLI'ını konteynere bağlayın.

Mini App, yalnızca Tailscale kullanan bir v1 yoludur ve Telegram Web iframe'ini desteklemez.

## Erişim denetimi ve etkinleştirme

### Grup bot kimliği

Gruplarda ve forum konularında, yapılandırılmış bot kullanıcı adının açıkça belirtilmesi (örneğin `@my_bot`), temsilci kişilik adı Telegram kullanıcı adından farklı olsa bile seçilen OpenClaw temsilcisine hitap eder. Grup sessizliği ilkesi ilgisiz trafik için hâlâ geçerlidir ancak bot kullanıcı adı hiçbir zaman "başka biri" değildir.

<Tabs>
  <Tab title="DM ilkesi">
    `channels.telegram.dmPolicy`, doğrudan mesaj erişimini denetler:

    - `pairing` (varsayılan)
    - `allowlist` (`allowFrom` içinde en az bir gönderen kimliği gerektirir)
    - `open` (`allowFrom` değerinin `"*"` içermesini gerektirir)
    - `disabled`

    `allowFrom: ["*"]` ile birlikte `dmPolicy: "open"`, bot kullanıcı adını bulan veya tahmin eden herhangi bir Telegram hesabının bota komut vermesine olanak tanır. Bunu yalnızca araçları sıkı biçimde kısıtlanmış, bilinçli olarak herkese açık botlar için kullanın; tek sahipli botlar sayısal kullanıcı kimlikleriyle `allowlist` kullanmalıdır.

    `channels.telegram.allowFrom`, sayısal Telegram kullanıcı kimliklerini kabul eder. `telegram:` / `tg:` ön ekleri kabul edilir ve normalleştirilir.
    Çok hesaplı yapılandırmalarda kısıtlayıcı bir üst düzey `channels.telegram.allowFrom` güvenlik sınırıdır: hesap düzeyindeki bir `allowFrom: ["*"]`, birleştirilmiş etkin izin listesi hâlâ açık bir joker karakter içermedikçe o hesabı herkese açık hâle getirmez.
    Boş `allowFrom` ile `dmPolicy: "allowlist"`, tüm DM'leri engeller ve yapılandırma doğrulaması tarafından reddedilir.
    Kurulum yalnızca sayısal kullanıcı kimliklerini ister. Yapılandırmanızda eski bir kurulumdan kalma `@username` izin listesi girdileri varsa bunları sayısal kimliklere çözümlemek için `openclaw doctor --fix` komutunu çalıştırın (azami çabayla; Telegram bot belirteci gerektirir).
    Daha önce eşleştirme deposundaki izin listesi dosyalarına güveniyorsanız `openclaw doctor --fix`, izin listesi akışları için girdileri `channels.telegram.allowFrom` içine kurtarabilir (örneğin `dmPolicy: "allowlist"` henüz açık kimlik içermiyorsa).

    Tek sahipli botlarda, önceki eşleştirme onaylarına güvenmek yerine açık sayısal `allowFrom` kimlikleriyle `dmPolicy: "allowlist"` kullanmayı tercih edin.

    Yaygın karışıklık: DM eşleştirme onayı, "bu gönderen her yerde yetkilidir" anlamına gelmez. Eşleştirme yalnızca DM erişimi verir. Henüz bir komut sahibi yoksa ilk onaylanan eşleştirme ayrıca `commands.ownerAllowFrom` değerini ayarlayarak yalnızca sahibe özel komutlara ve yürütme onaylarına açık bir operatör hesabı atar. Grup göndereni yetkilendirmesi yine açık yapılandırma izin listelerinden gelir.
    Tek bir kimlikle hem DM'ler hem de grup komutları için yetkilendirilmek üzere: sayısal Telegram kullanıcı kimliğinizi `channels.telegram.allowFrom` içine ekleyin ve yalnızca sahibe özel komutlar için `commands.ownerAllowFrom` değerinin `telegram:<your user id>` içerdiğinden emin olun.

    ### Telegram kullanıcı kimliğinizi bulma

    Daha güvenli (üçüncü taraf bot yok): botunuza DM gönderin, `openclaw logs --follow` komutunu çalıştırın, `from.id` değerini okuyun.

    Resmî Bot API yöntemi:

```bash
curl "https://api.telegram.org/bot<bot_token>/getUpdates"
```

    Üçüncü taraf (daha az özel): `@userinfobot` veya `@getidsbot`.

  </Tab>

  <Tab title="Grup ilkesi ve izin listeleri">
    İki denetim birlikte uygulanır:

    1. **Hangi gruplara izin verilir** (`channels.telegram.groups`)
       - `groups` yapılandırması yok, `groupPolicy: "open"`: tüm gruplar grup kimliği denetimlerini geçer
       - `groups` yapılandırması yok, `groupPolicy: "allowlist"` (varsayılan): `groups` girdileri (veya `"*"`) eklenene kadar tüm gruplar engellenir
       - `groups` yapılandırılmış: izin listesi olarak işlev görür (açık kimlikler veya `"*"`)

    2. **Gruplarda hangi gönderenlere izin verilir** (`channels.telegram.groupPolicy`)
       - `open` / `allowlist` (varsayılan) / `disabled`

    `groupAllowFrom`, grup gönderenlerini filtreler; ayarlanmamışsa Telegram, `allowFrom` değerine geri döner (eşleştirme deposuna değil — grup göndereni yetkilendirmesi, `2026.2.25` sürümünden beri bir güvenlik sınırı olarak DM eşleştirme deposu onaylarını hiçbir zaman devralmaz).
    `groupAllowFrom` girdileri sayısal Telegram kullanıcı kimlikleri olmalıdır (`telegram:` / `tg:` ön ekleri normalleştirilir); sayısal olmayan girdiler yok sayılır. Grup veya süper grup sohbet kimliklerini buraya koymayın — negatif sohbet kimlikleri `channels.telegram.groups` altında yer alır.
    Tek sahipli botlar için pratik kalıp: kullanıcı kimliğinizi `channels.telegram.allowFrom` içine ayarlayın, `groupAllowFrom` değerini ayarlamadan bırakın ve hedef gruplara `channels.telegram.groups` altında izin verin.
    `channels.telegram` yapılandırmada tamamen eksikse `channels.defaults.groupPolicy` açıkça ayarlanmadığı sürece çalışma zamanı varsayılan olarak erişimi kapatan `groupPolicy="allowlist"` değerini kullanır.

    Yalnızca sahip için grup kurulumu:

```json5
{
  channels: {
    telegram: {
      enabled: true,
      dmPolicy: "pairing",
      allowFrom: ["<YOUR_TELEGRAM_USER_ID>"],
      groupPolicy: "allowlist",
      groups: {
        "<GROUP_CHAT_ID>": {
          requireMention: true,
        },
      },
    },
  },
}
```

    Gruptan `@<bot_username> ping` ile test edin. `requireMention: true` olduğu sürece düz grup mesajları botu tetiklemez.

    Belirli bir gruptaki tüm üyelere izin verin:

```json5
{
  channels: {
    telegram: {
      groups: {
        "-1001234567890": {
          groupPolicy: "open",
          requireMention: false,
        },
      },
    },
  },
}
```

    Belirli bir grup içinde yalnızca belirli kullanıcılara izin verin:

```json5
{
  channels: {
    telegram: {
      groups: {
        "-1001234567890": {
          requireMention: true,
          allowFrom: ["8734062810", "745123456"],
        },
      },
    },
  },
}
```

    <Warning>
      Yaygın hata: `groupAllowFrom` bir grup izin listesi değildir.

      - Negatif Telegram grup/süper grup sohbet kimlikleri (`-1001234567890`) `channels.telegram.groups` altında yer alır.
      - İzin verilen bir grup içinde hangi kişilerin botu tetikleyebileceğini sınırlamak için Telegram kullanıcı kimlikleri (`8734062810`) `groupAllowFrom` altında yer alır.
      - Yalnızca izin verilen bir grubun herhangi bir üyesinin botla konuşmasına olanak tanımak için `groupAllowFrom: ["*"]` kullanın.

    </Warning>

  </Tab>

  <Tab title="Bahsetme davranışı">
    Grup yanıtları varsayılan olarak bahsetme gerektirir. Bahsetme şu yollardan gelebilir:

    - yerel bir `@botusername` bahsetmesi veya
    - `agents.entries.*.groupChat.mentionPatterns` ya da `messages.groupChat.mentionPatterns` içinde bir bahsetme kalıbı

    Oturum düzeyindeki geçişler (yalnızca durum, kalıcı değil): `/activation always`, `/activation mention`. Kalıcılık için yapılandırmayı kullanın:

```json5
{
  channels: {
    telegram: {
      groups: {
        "*": { requireMention: false },
      },
    },
  },
}
```

    Grup geçmişi bağlamı her zaman açıktır ve `historyLimit` ile sınırlandırılır. Grup geçmişi penceresini devre dışı bırakmak için `channels.telegram.historyLimit: 0` değerini ayarlayın. `openclaw doctor --fix`, kullanımdan kaldırılmış `includeGroupHistoryContext` anahtarını kaldırır.

    Grup sohbeti kimliğini alma: bir grup mesajını `@userinfobot` / `@getidsbot` adresine iletin, `openclaw logs --follow` içinden `chat.id` değerini okuyun, Bot API `getUpdates` sonucunu inceleyin veya (gruba izin verildikten sonra) `/whoami@<bot_username>` komutunu çalıştırın.

  </Tab>
</Tabs>

## Çalışma zamanı davranışı

- Telegram, gateway işlemi içinde çalışır.
- Yönlendirme belirlenimlidir: Telegram'dan gelen iletilerin yanıtları Telegram'a geri gönderilir (kanalları model seçmez).
- Gelen iletiler; yanıt meta verileri, medya yer tutucuları ve gateway'in gözlemlediği yanıtlar için kalıcı yanıt zinciri bağlamıyla birlikte paylaşılan kanal zarfına normalleştirilir.
- Grup oturumları, grup kimliğine göre yalıtılır. Forum konuları `:topic:<threadId>` ekler.
- DM iletileri `message_thread_id` taşıyabilir; OpenClaw bunu yanıtlar için korur. DM konu oturumları yalnızca Telegram `getMe` bot için `has_topics_enabled: true` bildirdiğinde ayrılır; aksi takdirde DM'ler düz oturumda kalır.
- Uzun yoklama, sohbet ve ileti dizisi başına sıralamayla grammY çalıştırıcısını kullanır. Çalıştırıcı havuzu eşzamanlılığı `agents.defaults.maxConcurrent` kullanır.
- Çok hesaplı başlatma, eşzamanlı `getMe` yoklamalarını sınırlar; böylece büyük bot filoları tüm hesap yoklamalarını aynı anda dağıtmaz.
- Her gateway işlemi, bir bot belirtecini aynı anda yalnızca bir etkin yoklayıcının kullanabilmesi için uzun yoklamayı korur. Kalıcı `getUpdates` 409 çakışmaları, aynı belirteci kullanan başka bir OpenClaw gateway'ine, betiğe veya harici yoklayıcıya işaret eder.
- Yoklama gözetmeni, tamamlanmış `getUpdates` canlılığı olmadan 120 saniye geçtikten sonra yeniden başlatılır.
- Telegram Bot API, okundu bilgisi desteğine sahip değildir (`sendReadReceipts` geçerli değildir).

<Note>
  `channels.telegram.dm.threadReplies` ve `channels.telegram.direct.<chatId>.threadReplies` kaldırıldı. Yapılandırmanızda hâlâ bu anahtarlar varsa yükseltmeden sonra `openclaw doctor --fix` çalıştırın. DM konu yönlendirmesi artık Telegram `getMe.has_topics_enabled` davranışını izler (BotFather ileti dizili modu tarafından denetlenir): konuları etkin botlar, Telegram `message_thread_id` gönderdiğinde ileti dizisi kapsamlı DM oturumlarını kullanır; diğer DM'ler düz oturumda kalır.
</Note>

## Özellik başvurusu

<AccordionGroup>
  <Accordion title="Canlı akış önizlemesi (ileti düzenlemeleri)">
    OpenClaw, doğrudan sohbetlerde, gruplarda ve konularda kısmi yanıtları gerçek zamanlı olarak yayımlar: bir önizleme iletisi gönderir, ardından `editMessageText` işlemini art arda gerçekleştirerek yerinde sonlandırır.

    - `channels.telegram.streaming`, `off | partial | block | progress` değerindedir (varsayılan: `partial`)
    - kısa ilk yanıt önizlemelerine geri sekme uygulanır; ardından çalıştırma hâlâ etkinse sınırlı bir gecikmeden sonra somutlaştırılır
    - `progress`, araç ilerlemesi için düzenlenebilir tek bir durum taslağını korur; yanıt etkinliği araç ilerlemesinden önce geldiğinde kararlı durum etiketini gösterir, tamamlandığında taslağı temizler ve son yanıtı normal bir ileti olarak gönderir
    - `streaming.preview.toolProgress`, araç/ilerleme güncellemelerinin aynı düzenlenmiş önizleme iletisini yeniden kullanıp kullanmayacağını denetler (varsayılan: önizleme akışı etkinken `true`)
    - `streaming.preview.commandText`, bu satırlardaki komut/yürütme ayrıntısını denetler: `raw` (varsayılan) veya `status` (yalnızca araç etiketi)
    - `streaming.progress.commentary` (varsayılan: `false`), geçici ilerleme taslağında asistan açıklaması/giriş metnini etkinleştirir
    - eski `channels.telegram.streamMode`, mantıksal `streaming` değerleri ve kullanımdan kaldırılmış yerel taslak önizleme anahtarları algılanır; bunları taşımak için `openclaw doctor --fix` çalıştırın

    Araç ilerleme satırları, araçlar çalışırken gösterilen kısa durum güncellemeleridir (komut yürütme, dosya okuma, planlama güncellemeleri, yama özetleri, uygulama sunucusu modunda Codex girişleri/açıklamaları). Telegram bunları varsayılan olarak açık tutar (`v2026.4.22`+ sürümünden itibaren yayımlanan davranışla eşleşir).

    Yanıt önizleme düzenlemelerini koruyup araç ilerleme satırlarını gizleyin:

    ```json
    {
      "channels": {
        "telegram": {
          "streaming": {
            "mode": "partial",
            "preview": { "toolProgress": false }
          }
        }
      }
    }
    ```

    Araç ilerlemesini görünür tutup komut/yürütme metnini gizleyin:

    ```json
    {
      "channels": {
        "telegram": {
          "streaming": {
            "mode": "partial",
            "preview": { "commandText": "status" }
          }
        }
      }
    }
    ```

    `progress` modu, son yanıtı bu iletinin içine düzenlemeden araç ilerlemesini gösterir. Komut metni ilkesini `streaming.progress` altına yerleştirin:

    ```json
    {
      "channels": {
        "telegram": {
          "streaming": {
            "mode": "progress",
            "progress": {
              "toolProgress": true,
              "commandText": "status"
            }
          }
        }
      }
    }
    ```

    `streaming.mode: "off"`, önizleme düzenlemelerini devre dışı bırakır ve genel araç/ilerleme bildirimlerini bağımsız durum iletileri olarak göndermek yerine bastırır; onay istemleri, medya ve hatalar yine normal son teslim yoluyla yönlendirilir. `streaming.preview.toolProgress: false` yalnızca yanıt önizleme düzenlemelerini korur.

    <Note>
      Seçili alıntı yanıtları istisnadır. `replyToMode`; `first`, `all` veya `batched` olduğunda ve gelen ileti seçili alıntı metni içerdiğinde OpenClaw, yanıt önizlemesini düzenlemek yerine son yanıtı Telegram'ın yerel alıntı yanıtlama yolu üzerinden gönderir; dolayısıyla `streaming.preview.toolProgress` o turda durum satırlarını gösteremez. Seçili alıntı metni bulunmayan geçerli ileti yanıtları akışa devam eder. Araç ilerlemesi görünürlüğü yerel alıntı yanıtlarından daha önemliyse `replyToMode: "off"`, bu ödünleşimi kabul etmek içinse `streaming.preview.toolProgress: false` ayarlayın.
    </Note>

    Yalnızca metin içeren yanıtlar için: kısa önizlemelerde son düzenleme yerinde yapılır; birden fazla iletiye bölünen uzun son yanıtlar önizlemeyi ilk parça olarak yeniden kullanır, ardından yalnızca kalan kısmı gönderir; ilerleme modu son yanıtları durum taslağını temizleyip normal son teslimi kullanır; tamamlanma onaylanmadan önce son düzenleme başarısız olursa OpenClaw normal son teslime geri döner ve eski önizlemeyi temizler. Karmaşık yanıtlar (medya yükleri) için OpenClaw her zaman normal son teslime geri döner ve önizlemeyi temizler.

    Önizleme akışı ile blok akışı birbirini dışlar — blok akışı açıkça etkinleştirildiğinde OpenClaw çift akışı önlemek için önizleme akışını atlar.

    Akıl yürütme: `/reasoning stream`, oluşturma sırasında akıl yürütmeyi canlı önizlemeye aktarır ve son teslimden sonra akıl yürütme önizlemesini siler (görünür tutmak için `/reasoning on` kullanın). Son yanıt, akıl yürütme metni olmadan gönderilir.

  </Accordion>

  <Accordion title="Zengin ileti biçimlendirmesi">
    Giden metin varsayılan olarak güncel istemcilerde okunabilen standart Telegram HTML iletilerini kullanır: kalın, italik, bağlantılar, kod, spoiler'lar, alıntılar — Bot API 10.2'ye özgü zengin bloklar (yerel tablolar, ayrıntılar, zengin medya, formüller) değil.

    Bot API 10.2 zengin iletilerini etkinleştirin:

```json5
{
  channels: {
    telegram: {
      richMessages: true,
    },
  },
}
```

    Etkinleştirildiğinde: aracıya bu bot/hesap için zengin iletilerin kullanılabildiği bildirilir (desteklenen Markdown + HTML adacığı yazım sözleşmesiyle birlikte); Markdown metni, OpenClaw'ın Markdown IR'si üzerinden türü belirlenmiş Bot API 10.2 zengin blokları (başlıklar, tablolar, ayrıntılar, denetim listeleri, zengin medya, formüller, haritalar, kolajlar) olarak işlenir; medya açıklamaları yine Telegram HTML açıklamalarını kullanır (zengin iletiler açıklamaların yerini almaz ve açıklamalar en fazla 1024 karakter olabilir).

    Bu, model metnini Telegram'ın zengin Markdown işaretlerinden uzak tutar; böylece `$400-600K` gibi para birimleri matematik olarak ayrıştırılmaz. Uzun zengin metin, Telegram'ın sınırlarına göre otomatik olarak bölünür. 20 sütun sınırını aşan tablolar kod bloğuna geri döner.

    Varsayılan: istemci uyumluluğu için kapalıdır — bazı güncel Masaüstü, Web, Android ve üçüncü taraf istemciler kabul edilen zengin iletileri desteklenmiyor olarak işler. Botla kullanılan her istemci bunları işleyemiyorsa bu özelliği kapalı tutun. `/status`, geçerli oturumda zengin iletilerin açık mı kapalı mı olduğunu gösterir.

    Bağlantı önizlemeleri varsayılan olarak açıktır. `channels.telegram.linkPreview: false`, zengin metin için otomatik varlık algılamayı devre dışı bırakır.

  </Accordion>

  <Accordion title="Yerel komutlar ve özel komutlar">
    Telegram'ın komut menüsü, başlatma sırasında `setMyCommands` ile kaydedilir. `commands.native: "auto"`, Telegram için yerel komutları etkinleştirir.

    Özel komut menüsü girdileri ekleyin:

```json5
{
  channels: {
    telegram: {
      customCommands: [
        { command: "backup", description: "Git yedeklemesi" },
        { command: "generate", description: "Görsel oluştur" },
      ],
    },
  },
}
```

    Kurallar: adlar normalleştirilir (baştaki `/` kaldırılır, küçük harfe dönüştürülür); geçerli desen `a-z`, `0-9`, `_`, uzunluk 1-32; özel komutlar yerel komutları geçersiz kılamaz; çakışmalar/yinelenenler atlanır ve günlüğe kaydedilir.

    Özel komutlar yalnızca menü girdileridir — davranışı otomatik olarak uygulamazlar. Plugin/skill komutları, Telegram menüsünde gösterilmese bile yazıldıklarında çalışmaya devam edebilir. Yerel komutlar devre dışıysa yerleşik komutlar kaldırılır; özel/Plugin komutları yapılandırılmışsa yine kaydedilebilir.

    Yaygın kurulum hataları:

    - Kırpma yeniden denemesinden sonra `BOT_COMMANDS_TOO_MUCH` içeren `setMyCommands failed`, menünün hâlâ sınırı aştığı anlamına gelir; Plugin/skill/özel komutları azaltın veya `channels.telegram.commands.native` özelliğini devre dışı bırakın.
    - Doğrudan Bot API curl komutları çalışırken `deleteWebhook`, `deleteMyCommands` veya `setMyCommands` işleminin `404: Not Found` ile başarısız olması genellikle `channels.telegram.apiRoot` değerinin tam `/bot<TOKEN>` uç noktasına ayarlandığı anlamına gelir. `apiRoot` yalnızca Bot API kökü olmalıdır; `openclaw doctor --fix`, yanlışlıkla eklenmiş sondaki `/bot<TOKEN>` bölümünü kaldırır.
    - `getMe returned 401`, Telegram'ın yapılandırılan bot belirtecini reddettiği anlamına gelir. `botToken`, `tokenFile` veya `TELEGRAM_BOT_TOKEN` (varsayılan hesap) değerini geçerli BotFather belirteciyle güncelleyin; OpenClaw yoklamadan önce durduğu için bu durum Webhook temizleme hatası olarak bildirilmez.
    - Ağ/getirme hataları içeren `setMyCommands failed` genellikle `api.telegram.org` adresine giden DNS/HTTPS trafiğinin engellendiği anlamına gelir.

    ### Cihaz eşleştirme komutları (`device-pair` Plugin'i)

    Kurulduğunda:

    1. `/pair` bir kurulum kodu oluşturur
    2. kodu iOS uygulamasına yapıştırın
    3. `/pair pending` bekleyen istekleri listeler (rol/kapsamlar dâhil)
    4. onaylama: `/pair approve <requestId>`, `/pair approve` (yalnızca bekleyen istek) veya `/pair approve latest`

    Bir cihaz değiştirilmiş kimlik doğrulama ayrıntılarıyla (rol, kapsamlar, ortak anahtar) yeniden denerse önceki bekleyen isteğin yerini yeni bir `requestId` alır; onaylamadan önce `/pair pending` komutunu yeniden çalıştırın.

    Daha fazla ayrıntı: [Eşleştirme](/tr/channels/pairing#pair-via-telegram).

  </Accordion>

  <Accordion title="Satır içi düğmeler">
    Satır içi klavye kapsamını yapılandırın:

```json5
{
  channels: {
    telegram: {
      capabilities: {
        inlineButtons: "allowlist",
      },
    },
  },
}
```

    Hesap başına geçersiz kılma:

```json5
{
  channels: {
    telegram: {
      accounts: {
        main: {
          capabilities: {
            inlineButtons: "allowlist",
          },
        },
      },
    },
  },
}
```

    Kapsamlar: `off`, `dm`, `group`, `all`, `allowlist` (varsayılan). Eski `capabilities: ["inlineButtons"]`, `"all"` değerine eşlenir.

    İleti eylemi örneği:

```json5
{
  action: "send",
  channel: "telegram",
  to: "123456789",
  message: "Bir seçenek belirleyin:",
  buttons: [
    [
      { text: "Evet", callback_data: "yes" },
      { text: "Hayır", callback_data: "no" },
    ],
    [{ text: "İptal", callback_data: "cancel" }],
  ],
}
```

    Mini Uygulama düğmesi örneği:

```json5
{
  action: "send",
  channel: "telegram",
  to: "123456789",
  message: "Uygulamayı açın:",
  presentation: {
    blocks: [
      {
        type: "buttons",
        buttons: [{ label: "Başlat", web_app: { url: "https://example.com/app" } }],
      },
    ],
  },
}
```

    `web_app` düğmeleri yalnızca bir kullanıcı ile bot arasındaki özel sohbetlerde çalışır.

    Kayıtlı bir Plugin etkileşimli işleyicisi tarafından sahiplenilmeyen geri çağırma tıklamaları aracıya metin olarak iletilir: `callback_data: <value>`.

  </Accordion>

  <Accordion title="Aracılar ve otomasyon için Telegram ileti eylemleri">
    Eylemler:

    - `sendMessage` (`to`, `content`, isteğe bağlı `mediaUrl`, `replyToMessageId`, `messageThreadId`)
    - `react` (`chatId`, `messageId`, `emoji`)
    - `deleteMessage` (`chatId`, `messageId`)
    - `editMessage` (`chatId`, `messageId`, `content` veya `caption`, isteğe bağlı `presentation` satır içi düğmeleri; yalnızca düğme düzenlemeleri yanıt işaretlemesini günceller)
    - `createForumTopic` (`chatId`, `name`, isteğe bağlı `iconColor`, `iconCustomEmojiId`)

    Kullanışlı takma adlar: `send`, `react`, `delete`, `edit`, `sticker`, `sticker-search`, `topic-create`.

    Etkinleştirme koşulları: `channels.telegram.actions.sendMessage`, `deleteMessage`, `reactions`, `sticker` (varsayılan: devre dışı). `edit`, `createForumTopic` ve `editForumTopic`, özel bir geçiş anahtarı olmadan varsayılan olarak etkindir.
    Çalışma zamanı gönderimleri, başlangıçta/yeniden yüklemede alınan etkin yapılandırma/gizli bilgiler anlık görüntüsünü kullanır; bu nedenle eylem yolları her gönderimde `SecretRef` değerlerini yeniden çözümlemez.

    Tepki kaldırma semantiği: [/tools/reactions](/tr/tools/reactions).

  </Accordion>

  <Accordion title="Yanıt iş parçacığı etiketleri">
    Oluşturulan çıktıdaki açık yanıt iş parçacığı etiketleri:

    - `[[reply_to_current]]` — tetikleyici iletiyi yanıtlar
    - `[[reply_to:<id>]]` — belirli bir ileti kimliğini yanıtlar

    `channels.telegram.replyToMode`: `off` (varsayılan), `first`, `all`.

    Yanıt iş parçacığı etkinleştirildiğinde ve özgün metin/açıklama mevcut olduğunda OpenClaw, otomatik olarak yerel bir alıntı kesiti ekler. Telegram, yerel alıntı metnini 1024 UTF-16 kod birimiyle sınırlar; daha uzun iletiler baştan itibaren alıntılanır ve Telegram alıntıyı reddederse düz yanıta geri dönülür.

    `off` yalnızca örtük yanıt iş parçacığını devre dışı bırakır; açık `[[reply_to_*]]` etiketleri yine de dikkate alınır.

  </Accordion>

  <Accordion title="Forum konuları ve iş parçacığı davranışı">
    Forum süper grupları: konu oturumu anahtarlarının sonuna `:topic:<threadId>` eklenir; yanıtlar ve yazıyor göstergesi konu iş parçacığını hedefler; konu yapılandırma yolu `channels.telegram.groups.<chatId>.topics.<threadId>` şeklindedir.

    Genel konu (`threadId=1`) özel bir durumdur: ileti gönderimlerinde `message_thread_id` kullanılmaz (Telegram, `sendMessage(...thread_id=1)` değerini "iş parçacığı bulunamadı" hatasıyla reddeder), ancak yazıyor eylemleri yine de `message_thread_id` içerir (yazıyor göstergesinin görünmesi için deneysel olarak gerekli olduğu belirlenmiştir).

    Konu girdileri, geçersiz kılınmadığı sürece grup ayarlarını devralır (`requireMention`, `allowFrom`, `skills`, `systemPrompt`, `enabled`, `groupPolicy`). `agentId` yalnızca konuya özeldir ve grup varsayılanlarından devralınmaz. `topics."*"`, o gruptaki her konu için varsayılanları belirler; tam konu kimlikleri yine de `"*"` değerine göre önceliklidir.

    **Konu başına agent yönlendirmesi**: Her konu, konu yapılandırmasındaki `agentId` aracılığıyla farklı bir agent'a yönlendirilebilir; böylece kendine ait bir çalışma alanına, belleğe ve oturuma sahip olur:

    ```json5
    {
      channels: {
        telegram: {
          groups: {
            "-1001234567890": {
              topics: {
                "1": { agentId: "main" },      // Genel konu -> ana agent
                "3": { agentId: "zu" },        // Geliştirme konusu -> zu agent'ı
                "5": { agentId: "coder" }      // Kod incelemesi -> coder agent'ı
              }
            }
          }
        }
      }
    }
    ```

    Ardından her konunun kendi oturum anahtarı olur; örneğin `agent:zu:telegram:group:-1001234567890:topic:3`.

    **Kalıcı ACP konu bağlama**: Forum konuları, üst düzey türü belirlenmiş bağlamalar aracılığıyla ACP çalıştırma düzeneği oturumlarını sabitleyebilir (`bindings[]`; `type: "acp"`, `match.channel: "telegram"`, `peer.kind: "group"` ve `-1001234567890:topic:42` gibi konu nitelemeli bir kimlikle). Şu anda gruplardaki/süper gruplardaki forum konularıyla sınırlıdır. Bkz. [ACP Agent'ları](/tr/tools/acp-agents).

    **Sohbetten iş parçacığına bağlı ACP başlatma**: `/acp spawn <agent> --thread here|auto`, geçerli konuyu yeni bir ACP oturumuna bağlar; devam iletileri doğrudan buraya yönlendirilir ve OpenClaw, başlatma onayını konu içinde sabitler. `session.threadBindings.spawnSessions` tarafından denetlenir (varsayılan: `true`).

    Şablon bağlamı, `MessageThreadId` ve `IsForum` değerlerini kullanıma sunar. `message_thread_id` içeren DM sohbetleri yanıt meta verilerini korur, ancak yalnızca Telegram `getMe`, `has_topics_enabled: true` bildirdiğinde iş parçacığına duyarlı oturum anahtarlarını kullanır.
    Kullanımdan kaldırılan `dm.threadReplies` ve `direct.*.threadReplies` geçersiz kılmaları kaldırılmıştır; BotFather iş parçacığı modu tek doğruluk kaynağıdır. Eski yapılandırma anahtarlarını kaldırmak için `openclaw doctor --fix` çalıştırın.

  </Accordion>

  <Accordion title="Ses, video ve çıkartmalar">
    ### Sesli iletiler

    Telegram, sesli notları ses dosyalarından ayırır. Varsayılan: ses dosyası davranışı; sesli not olarak göndermeyi zorlamak için agent yanıtında `[[audio_as_voice]]` etiketini kullanın. Gelen sesli not dökümleri, agent bağlamında makine tarafından oluşturulmuş ve güvenilmeyen metin olarak çerçevelenir; ancak bahsetme algılaması yine de ham dökümü kullanır, böylece bahsetme koşullu sesli iletiler çalışmaya devam eder.

```json5
{
  action: "send",
  channel: "telegram",
  to: "123456789",
  media: "https://example.com/voice.ogg",
  asVoice: true,
}
```

    ### Görüntülü iletiler

    Telegram, video dosyalarını görüntülü notlardan ayırır. Görüntülü notlar açıklamaları desteklemez; sağlanan ileti metni ayrı olarak gönderilir.

```json5
{
  action: "send",
  channel: "telegram",
  to: "123456789",
  media: "https://example.com/video.mp4",
  asVideoNote: true,
}
```

    ### Konumlar ve mekânlar

    Tek bir bağımsız `location` nesnesiyle mevcut `send` eylemini kullanın. Koordinatlar yerel bir konum işareti gönderir; hem `name` hem de `address` eklendiğinde yerel bir mekân kartı gönderilir. Konum gönderimleri, ileti metni veya medyayla birleştirilemez.

```json5
{
  action: "send",
  channel: "telegram",
  to: "123456789",
  location: {
    latitude: 48.858844,
    longitude: 2.294351,
    accuracy: 12,
    name: "Eiffel Kulesi",
    address: "Champ de Mars, Paris",
  },
}
```

    ### Çıkartmalar

    Gelen: statik WEBP indirilir ve işlenir (yer tutucu `<media:sticker>`); animasyonlu TGS ve video WEBM atlanır.

    Çıkartma bağlamı alanları: `Sticker.emoji`, `Sticker.setName`, `Sticker.fileId`, `Sticker.fileUniqueId`, `Sticker.cachedDescription`. Tekrarlanan görüntü çağrılarını azaltmak için açıklamalar, OpenClaw SQLite Plugin durumunda önbelleğe alınır.

    Çıkartma eylemlerini etkinleştirin:

```json5
{
  channels: {
    telegram: {
      actions: {
        sticker: true,
      },
    },
  },
}
```

    Gönderin:

```json5
{
  action: "sticker",
  channel: "telegram",
  to: "123456789",
  fileId: "CAACAgIAAxkBAAI...",
}
```

    Önbelleğe alınmış çıkartmaları arayın:

```json5
{
  action: "sticker-search",
  channel: "telegram",
  query: "el sallayan kedi",
  limit: 5,
}
```

  </Accordion>

  <Accordion title="Tepki bildirimleri">
    Telegram tepkileri, ileti yüklerinden ayrı olarak `message_reaction` güncellemeleri biçiminde gelir. Etkinleştirildiğinde OpenClaw, `Telegram reaction added: 👍 by Alice (@alice) on msg 42` gibi sistem olaylarını kuyruğa alır.

    - `channels.telegram.reactionNotifications`: `off | own | all` (varsayılan: `own`)
    - `channels.telegram.reactionLevel`: `off | ack | minimal | extensive` (varsayılan: `minimal`)

    `own`, yalnızca bot tarafından gönderilen iletilere yönelik kullanıcı tepkileri anlamına gelir (gönderilen ileti önbelleği aracılığıyla azami gayret esasına göre). Tepki olayları, Telegram erişim denetimlerine (`dmPolicy`, `allowFrom`, `groupPolicy`, `groupAllowFrom`) yine de uyar; yetkisiz göndericiler yok sayılır.

    Telegram, tepki güncellemelerinde iş parçacığı kimliklerini sağlamaz: forum dışı gruplar grup sohbeti oturumuna; forum gruplarıysa tam kaynak konuya değil, genel konu oturumuna (`:topic:1`) yönlendirilir.

    Yoklama/webhook için `allowed_updates`, `message_reaction` değerini otomatik olarak içerir.

  </Accordion>

  <Accordion title="Alındı tepkileri">
    `ackReaction`, OpenClaw gelen bir iletiyi işlerken bir alındı emojisi gönderir. `messages.ackReactionScope`, bunun *ne zaman* gönderileceğini belirler.

    **Emoji çözümleme sırası:**

    - `channels.telegram.accounts.<accountId>.ackReaction`
    - `channels.telegram.ackReaction`
    - `messages.ackReaction`
    - agent kimliği emojisi geri dönüşü (`agents.entries.*.identity.emoji`, aksi takdirde "👀")

    Telegram bir Unicode emojisi (örneğin "👀") bekler; bir kanal veya hesap için tepkiyi devre dışı bırakmak üzere `""` kullanın.

    **Kapsam (`messages.ackReactionScope`, varsayılan `"group-mentions"`; şu anda Telegram hesabı veya Telegram kanalı geçersiz kılması yoktur):**

    `all` (DM'ler + gruplar, ortam odası olayları dâhil), `direct` (yalnızca DM'ler), `group-all` (ortam odası olayları hariç tüm grup iletileri, DM yok), `group-mentions` (bottan bahsedildiğinde gruplar; **DM yok** — varsayılan), `off` / `none` (devre dışı).

    <Note>
    Varsayılan kapsam (`group-mentions`), DM'lerde veya ortam odası olaylarında alındı tepkilerini tetiklemez. DM'ler için `direct` veya `all` kullanın; ortam odası olaylarını yalnızca `all` onaylar. Bu değer Telegram sağlayıcısı başlatılırken okunur; dolayısıyla değişikliğin yürürlüğe girmesi için Gateway'in yeniden başlatılması gerekir.
    </Note>

  </Accordion>

  <Accordion title="Telegram olaylarından ve komutlarından yapılandırma yazma">
    Kanal yapılandırması yazma işlemleri varsayılan olarak etkindir (`configWrites !== false`). Telegram tarafından tetiklenen yazma işlemleri; grup taşıma olaylarını (`migrate_to_chat_id`, `channels.telegram.groups` değerini günceller) ve `/config set` / `/config unset` işlemlerini (komutun etkinleştirilmesini gerektirir) içerir.

    Devre dışı bırakın:

```json5
{
  channels: {
    telegram: {
      configWrites: false,
    },
  },
}
```

  </Accordion>

  <Accordion title="Uzun yoklama ve webhook karşılaştırması">
    Varsayılan, uzun yoklamadır. Webhook modu için `channels.telegram.webhookUrl` ve `channels.telegram.webhookSecret` ayarlayın; isteğe bağlı olarak `webhookPath` (varsayılan `/telegram-webhook`), `webhookHost` (varsayılan `127.0.0.1`), `webhookPort` (varsayılan `8787`), `webhookCertPath` (doğrudan IP veya alan adsız kurulumlar için kendinden imzalı sertifika PEM'i) ayarlanabilir.

    Uzun yoklama modunda OpenClaw, yeniden başlatma filigranını yalnızca bir güncelleme başarıyla gönderildikten sonra kalıcılaştırır; başarısız bir işleyici, söz konusu güncellemeyi tamamlandı olarak işaretlemek yerine aynı süreçte yeniden denenebilir durumda bırakır.

    Yerel dinleyici varsayılan olarak `127.0.0.1:8787` adresine bağlanır. Genel erişim için yerel bağlantı noktasının önüne bir ters proxy yerleştirin veya `webhookHost: "0.0.0.0"` değerini bilinçli olarak ayarlayın.

    Webhook modu istek korumalarını, Telegram gizli belirtecini ve JSON gövdesini doğrular; ardından boş bir `200` döndürmeden önce güncellemeyi dayanıklı giriş kuyruğuna kaydeder. Başarılı dayanıklı kabul işlemi `x-openclaw-delivery-accepted: durable` içerir; sağlık, yönlendirme, kimlik doğrulama, doğrulama ve depolama hatası yanıtlarında bu üstbilgi bulunmaz. Ters proxy'ler ve ana makine denetleyicileri, kabulü yanıt zamanlamasından çıkarsamadan OpenClaw kabulünü genel bir boş `200` yanıtından ayırt etmek için bu üstbilgiyi zorunlu tutabilir.

    Dayanıklı yazma işleminden sonra OpenClaw, güncellemeleri çekirdek kanal girişi boşaltma mekanizması üzerinden talep eder ve işler (sohbet/konu başına işlem hatları, tur kabulünde tamamlama, kabul öncesi duraklama zaman aşımı). Yavaş agent turları Telegram'ın teslimat ACK'sini bekletmez.

  </Accordion>

  <Accordion title="Sınırlar ve CLI hedefleri">
    - `channels.telegram.textChunkLimit` varsayılan olarak 4000'dir; `streaming.chunkMode="newline"`, uzunluğa göre bölmeden önce paragraf sınırlarını (boş satırlar) tercih eder.
    - `channels.telegram.mediaMaxMb` (varsayılan 100), gelen ve giden medya boyutunu sınırlar.
    - grup bağlamı geçmişi `channels.telegram.historyLimit` veya `messages.groupChat.historyLimit` (varsayılan 50) kullanır; `0` devre dışı bırakır.
    - yanıt/alıntı/iletme ek bağlamı, Gateway üst mesajları gözlemlediğinde seçili tek bir konuşma bağlamı penceresinde normalleştirilir; gözlemlenen mesaj önbelleği OpenClaw SQLite Plugin durumunda tutulur ve `openclaw doctor --fix` eski yan dosyaları içe aktarır. Telegram, güncelleme başına yalnızca bir sığ `reply_to_message` içerdiğinden, önbellekten daha eski zincirler bu yükle sınırlıdır.
    - Telegram izin listeleri esas olarak ek bağlam için tam bir sansürleme sınırı oluşturmayı değil, aracıyı kimlerin tetikleyebileceğini denetler.
    - DM geçmişi: `channels.telegram.dmHistoryLimit`, `channels.telegram.dms["<user_id>"].historyLimit`.

    CLI ve mesaj aracı gönderme hedefleri; sayısal sohbet kimliğini, kullanıcı adını veya forum konusu hedefini kabul eder:

```bash
openclaw message send --channel telegram --target 123456789 --message "hi"
openclaw message send --channel telegram --target @name --message "hi"
openclaw message send --channel telegram --target -1001234567890:topic:42 --message "hi topic"
```

    Anketler `openclaw message poll` kullanır ve forum konularını destekler:

```bash
openclaw message poll --channel telegram --target 123456789 \
  --poll-question "Ship it?" --poll-option "Yes" --poll-option "No"
openclaw message poll --channel telegram --target -1001234567890:topic:42 \
  --poll-question "Pick a time" --poll-option "10am" --poll-option "2pm" \
  --poll-duration-seconds 300 --poll-public
```

    Yalnızca Telegram'a özgü anket bayrakları: `--poll-duration-seconds` (5-600), `--poll-anonymous`, `--poll-public`, `--thread-id` (veya bir `:topic:` hedefi). `--poll-option`, 2-12 kez tekrarlar (Telegram'ın seçenek sınırı).

    Telegram gönderimi ayrıca satır içi klavyeler için `buttons` bloklarıyla `--presentation` seçeneğini (`channels.telegram.capabilities.inlineButtons` izin verdiğinde), bot ilgili sohbette sabitleme yapabiliyorsa sabitlenmiş teslimat istemek için `--pin` veya `--delivery '{"pin":true}'` seçeneğini ve giden görüntüleri, GIF'leri ve videoları sıkıştırılmış/animasyonlu/video yüklemeleri yerine belge olarak göndermek için `--force-document` seçeneğini destekler.

    Eylem denetimi: `channels.telegram.actions.sendMessage=false`, anketler dâhil tüm giden mesajları devre dışı bırakır; `channels.telegram.actions.poll=false`, normal gönderimleri etkin bırakırken anket oluşturmayı devre dışı bırakır.

  </Accordion>

  <Accordion title="Telegram'da exec onayları">
    Telegram, onaylayıcıların DM'lerinde exec onaylarını destekler ve isteğe bağlı olarak istemleri kaynak sohbette veya konuda yayımlayabilir. Onaylayıcılar sayısal Telegram kullanıcı kimlikleri olmalıdır.

    - `channels.telegram.execApprovals.enabled` (en az bir onaylayıcı çözümlenebildiğinde `"auto"` etkinleştirir)
    - `channels.telegram.execApprovals.approvers` (`commands.ownerAllowFrom` içindeki sayısal sahip kimliklerine geri döner)
    - `channels.telegram.execApprovals.target`: `dm` (varsayılan) | `channel` | `both`
    - `agentFilter`, `sessionFilter`

    `channels.telegram.allowFrom`, `groupAllowFrom` ve `defaultTo`, botla kimlerin konuşabileceğini ve normal yanıtları nereye gönderdiğini denetler; bir kişiyi exec onaylayıcısı yapmaz. Henüz komut sahibi yoksa onaylanan ilk DM eşleştirmesi `commands.ownerAllowFrom` için başlangıç yapılandırmasını oluşturur; böylece tek sahipli kurulumlar, `execApprovals.approvers` altında kimlikleri yinelemeden çalışır.

    Kanal teslimatı komut metnini sohbette gösterir; `channel` veya `both` yalnızca güvenilir gruplarda/konularda etkinleştirilmelidir. İstem bir forum konusuna ulaştığında OpenClaw, onay istemi ve devamındaki iletiler için konuyu korur. Exec onaylarının süresi varsayılan olarak 30 dakika sonra dolar.

    Satır içi onay düğmeleri ayrıca `channels.telegram.capabilities.inlineButtons` değerinin hedef yüzeye (`dm`, `group` veya `all`) izin vermesini gerektirir. `plugin:` ön ekli onay kimlikleri Plugin onayları üzerinden çözümlenir; diğerleri önce exec onayları üzerinden çözümlenir.

    Bkz. [Exec onayları](/tr/tools/exec-approvals).

  </Accordion>
</AccordionGroup>

## Hata yanıtı denetimleri

Aracı bir teslimat veya sağlayıcı hatasıyla karşılaştığında hata ilkesi, hata mesajlarının Telegram sohbetine ulaşıp ulaşmayacağını denetler:

| Anahtar                         | Değerler                   | Varsayılan | Açıklama                                                                                                                                                                   |
| ------------------------------- | -------------------------- | ---------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `channels.telegram.errorPolicy` | `always`, `once`, `silent` | `always` | `always`, her hata mesajını sohbete gönderir. `once`, her benzersiz hata mesajını yerleşik bekleme süresi penceresi başına bir kez gönderir. `silent`, hata mesajlarını hiçbir zaman sohbete göndermez. |

Hesap, grup ve konu başına geçersiz kılmalar desteklenir (diğer Telegram yapılandırma anahtarlarıyla aynı kalıtım).

```json5
{
  channels: {
    telegram: {
      errorPolicy: "always",
      groups: {
        "-1001234567890": {
          errorPolicy: "silent", // bu gruptaki hataları engelle
        },
      },
    },
  },
}
```

## Sorun giderme

<AccordionGroup>
  <Accordion title="Bot, etiketleme içermeyen grup mesajlarına yanıt vermiyor">

    - `requireMention=false` ise Telegram gizlilik modu tam görünürlüğe izin vermelidir: BotFather `/setprivacy` -> Disable; ardından botu gruptan kaldırıp yeniden ekleyin.
    - Yapılandırma, etiketleme içermeyen grup mesajlarını beklediğinde `openclaw channels status` uyarı verir.
    - `openclaw channels status --probe`, açıkça belirtilen sayısal grup kimliklerini denetler; joker karakter `"*"` için üyelik sorgulanamaz.
    - Hızlı oturum testi: `/activation always`.

  </Accordion>

  <Accordion title="Bot grup mesajlarını hiç görmüyor">

    - `channels.telegram.groups` mevcutsa grup listelenmelidir (veya `"*"` içermelidir).
    - Botun gruba üye olduğunu doğrulayın.
    - Atlama nedenleri için `openclaw logs --follow` değerini inceleyin.

  </Accordion>

  <Accordion title="Komutlar kısmen çalışıyor veya hiç çalışmıyor">

    - Gönderen kimliğinizi yetkilendirin (eşleştirme ve/veya sayısal `allowFrom`); grup ilkesi `open` olsa bile komut yetkilendirmesi geçerliliğini korur.
    - `BOT_COMMANDS_TOO_MUCH` ile birlikte `setMyCommands failed`, yerel menüde çok fazla girdi olduğu anlamına gelir; Plugin/Skill/özel komutların sayısını azaltın veya yerel menüleri devre dışı bırakın.
    - `deleteMyCommands` / `setMyCommands` başlangıç çağrıları ve `sendChatAction` yazma çağrıları sınırlıdır ve istek zaman aşımında Telegram'ın aktarım geri dönüşü üzerinden bir kez yeniden denenir. Sürekli ağ/fetch hataları genellikle `api.telegram.org` adresine DNS/HTTPS üzerinden erişilemediği anlamına gelir.

  </Accordion>

  <Accordion title="Başlangıçta yetkisiz token bildiriliyor">

    - `getMe returned 401`, yapılandırılan bot token'ı için bir Telegram kimlik doğrulama hatasıdır. BotFather'da token'ı yeniden kopyalayın veya oluşturun; ardından `channels.telegram.botToken`, `tokenFile`, `accounts.<id>.botToken` ya da `TELEGRAM_BOT_TOKEN` (varsayılan hesap) değerini güncelleyin.
    - Başlangıç sırasındaki `deleteWebhook 401 Unauthorized` de bir kimlik doğrulama hatasıdır; bunu "Webhook yok" olarak değerlendirmek, aynı geçersiz token hatasını yalnızca daha sonraki bir API çağrısına erteler.

  </Accordion>

  <Accordion title="Yoklama veya ağ kararsızlığı">

    - Özel bir fetch/proxy ile Node 22+, `AbortSignal` türleri eşleşmezse anında iptal davranışını tetikleyebilir.
    - Bazı ana makineler `api.telegram.org` adresini önce IPv6'ya çözümler; bozuk IPv6 çıkışı aralıklı API hatalarına neden olur.
    - `TypeError: fetch failed` veya `Network request for 'getUpdates' failed!` içeren günlükler, kurtarılabilir ağ hataları olarak yeniden denenir.
    - Yoklama başlangıcı sırasında OpenClaw, grammY için başarılı başlangıç `getMe` yoklamasını yeniden kullanır; böylece çalıştırıcının ilk `getUpdates` öncesinde ikinci bir `getMe` çağrısına ihtiyacı olmaz.
    - `deleteWebhook`, yoklama başlangıcı sırasında geçici bir ağ hatasıyla başarısız olursa OpenClaw, yoklama öncesi başka bir denetim düzlemi çağrısı yapmak yerine uzun yoklamaya devam eder. Hâlâ etkin olan bir Webhook daha sonra `getUpdates` çakışması olarak ortaya çıkar; OpenClaw aktarımı yeniden oluşturur ve Webhook temizliğini yeniden dener.
    - Günlüklerdeki `Polling stall detected`, varsayılan olarak tamamlanmış uzun yoklama canlılığı olmadan 120 saniye geçtikten sonra OpenClaw'ın yoklamayı yeniden başlatıp aktarımı yeniden oluşturduğu anlamına gelir.
    - `openclaw channels status --probe` ve `openclaw doctor`; çalışan bir yoklama hesabı başlangıç ek süresinden sonra `getUpdates` işlemini tamamlamadığında, çalışan bir Webhook hesabı başlangıç ek süresinden sonra `setWebhook` işlemini tamamlamadığında veya son başarılı yoklama aktarımı etkinliği eski kaldığında uyarı verir.
    - Telegram, Bot API aktarımı için işlem proxy ortam değişkenlerini dikkate alır: `HTTP_PROXY`, `HTTPS_PROXY`, `ALL_PROXY` ve küçük harfli çeşitleri. `NO_PROXY` / `no_proxy` yine de `api.telegram.org` değerini atlayabilir.
    - Bir hizmet ortamı için `OPENCLAW_PROXY_URL` ayarlanmışsa ve standart proxy ortam değişkeni yoksa Telegram, Bot API aktarımı için de bu URL'yi kullanır.
    - Doğrudan çıkış/TLS bağlantısı kararsız VPS ana makinelerinde Telegram API çağrılarını bir proxy üzerinden yönlendirin:

```yaml
channels:
  telegram:
    proxy: socks5://<user>:<password>@proxy-host:1080
```

    - Node 22+, varsayılan olarak `autoSelectFamily=true` kullanır (WSL2 hariç). Telegram DNS sonuç sırası önce `OPENCLAW_TELEGRAM_DNS_RESULT_ORDER`, ardından `channels.telegram.network.dnsResultOrder`, sonrasında işlem varsayılanını (örneğin `NODE_OPTIONS=--dns-result-order=ipv4first`) dikkate alır; hiçbiri geçerli değilse Node 22+ üzerinde `ipv4first` değerine geri döner.
    - WSL2'de veya yalnızca IPv4 davranışı daha iyi çalıştığında adres ailesi seçimini zorlayın:

```yaml
channels:
  telegram:
    network:
      autoSelectFamily: false
```

    - RFC 2544 kıyaslama aralığı yanıtlarına (`198.18.0.0/15`) Telegram medya indirmeleri için varsayılan olarak zaten izin verilir. Güvenilir bir sahte IP veya şeffaf proxy, medya indirmeleri sırasında `api.telegram.org` değerini başka bir özel/dâhilî/özel kullanım adresine dönüştürüyorsa yalnızca Telegram'a özgü atlamayı etkinleştirin:

```yaml
channels:
  telegram:
    network:
      dangerouslyAllowPrivateNetwork: true
```

    - Aynı etkinleştirme seçeneği hesap başına `channels.telegram.accounts.<accountId>.network.dangerouslyAllowPrivateNetwork` konumunda kullanılabilir.
    - Proxy'niz Telegram medya ana makinelerini `198.18.x.x` aralığına çözümlüyorsa önce tehlikeli bayrağı kapalı bırakın; bu aralığa varsayılan olarak zaten izin verilir.

    <Warning>
      `channels.telegram.network.dangerouslyAllowPrivateNetwork`, Telegram medya SSRF korumalarını zayıflatır. Yalnızca RFC 2544 kıyaslama aralığı dışında özel veya özel kullanım yanıtları üreten, güvenilir ve operatör denetimindeki proxy ortamlarında (Clash, Mihomo, Surge sahte IP yönlendirmesi) kullanın. Normal genel internet Telegram erişimi için kapalı bırakın.
    </Warning>

    - Geçici ortam geçersiz kılmaları: `OPENCLAW_TELEGRAM_DISABLE_AUTO_SELECT_FAMILY=1`, `OPENCLAW_TELEGRAM_ENABLE_AUTO_SELECT_FAMILY=1`, `OPENCLAW_TELEGRAM_DNS_RESULT_ORDER=ipv4first`.
    - DNS yanıtlarını doğrulayın:

```bash
dig +short api.telegram.org A
dig +short api.telegram.org AAAA
```

  </Accordion>
</AccordionGroup>

Daha fazla yardım: [Kanal sorunlarını giderme](/tr/channels/troubleshooting).

## Yapılandırma başvurusu

Birincil başvuru: [Yapılandırma başvurusu - Telegram](/tr/gateway/config-channels#telegram).

<Accordion title="Yüksek öneme sahip Telegram alanları">

- başlatma/kimlik doğrulama: `enabled`, `botToken`, `tokenFile` (normal bir dosya olmalıdır; sembolik bağlantılar reddedilir), `accounts.*`
- erişim denetimi: `dmPolicy`, `allowFrom`, `groupPolicy`, `groupAllowFrom`, `groups`, `groups.*.topics.*`, üst düzey `bindings[]` (`type: "acp"`)
- konu varsayılanları: `groups.<chatId>.topics."*"` eşleşmeyen forum konularına uygulanır; tam konu kimlikleri bunu geçersiz kılar
- çalıştırma onayları: `execApprovals`, `accounts.*.execApprovals`
- komut/menü: `commands.native`, `commands.nativeSkills`, `customCommands`
- iş parçacıkları/yanıtlar: `replyToMode`, `threadBindings`
- akış: `streaming` (modlar `off | partial | block | progress`), `streaming.preview.toolProgress`
- biçimlendirme/teslim: `textChunkLimit`, `streaming.chunkMode`, `richMessages`, `markdown.tables` (`off | bullets | code | block`), `linkPreview`, `responsePrefix`
- medya/ağ: `mediaMaxMb`, `network.autoSelectFamily`, `network.dangerouslyAllowPrivateNetwork`, `proxy`
- özel API kökü: `apiRoot` (yalnızca Bot API kökü; `/bot<TOKEN>` eklemeyin), `trustedLocalFileRoots` (kendi barındırdığınız Bot API'nin mutlak `file_path` kökleri)
- Webhook: `webhookUrl`, `webhookSecret`, `webhookPath`, `webhookHost`, `webhookPort`, `webhookCertPath`
- eylemler/yetenekler: `capabilities.inlineButtons`, `actions.sendMessage|editMessage|deleteMessage|reactions|sticker|createForumTopic|editForumTopic`
- tepkiler: `reactionNotifications`, `reactionLevel`
- hatalar: `errorPolicy`, `silentErrorReplies`
- yazma/geçmiş: `configWrites`, `historyLimit`, `dmHistoryLimit`, `dms.*.historyLimit`

</Accordion>

<Note>
Çoklu hesap önceliği: İki veya daha fazla hesap kimliği yapılandırıldığında, varsayılan yönlendirmeyi açıkça belirtmek için `channels.telegram.defaultAccount` ayarlayın (veya `channels.telegram.accounts.default` ekleyin). Aksi takdirde OpenClaw, normalleştirilmiş ilk hesap kimliğine geri döner ve `openclaw doctor` uyarı verir. Adlandırılmış hesaplar `channels.telegram.allowFrom` / `groupAllowFrom` değerlerini devralır, ancak `accounts.default.*` değerlerini devralmaz.
</Note>

## İlgili

<CardGroup cols={2}>
  <Card title="Eşleştirme" icon="link" href="/tr/channels/pairing">
    Bir Telegram kullanıcısını Gateway ile eşleştirin.
  </Card>
  <Card title="Gruplar" icon="users" href="/tr/channels/groups">
    Grup ve konu izin listesi davranışı.
  </Card>
  <Card title="Kanal yönlendirme" icon="route" href="/tr/channels/channel-routing">
    Gelen iletileri aracılara yönlendirin.
  </Card>
  <Card title="Güvenlik" icon="shield" href="/tr/gateway/security">
    Tehdit modeli ve sağlamlaştırma.
  </Card>
  <Card title="Çok aracılı yönlendirme" icon="sitemap" href="/tr/concepts/multi-agent">
    Grupları ve konuları aracılarla eşleyin.
  </Card>
  <Card title="Sorun giderme" icon="wrench" href="/tr/channels/troubleshooting">
    Kanallar arası tanılama.
  </Card>
</CardGroup>
