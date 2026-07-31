---
read_when:
    - Discord kanal özellikleri üzerinde çalışma
summary: Discord bot kurulumu, yapılandırma anahtarları, bileşenler, ses ve sorun giderme
title: Discord
x-i18n:
    generated_at: "2026-07-26T23:29:23Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 52a2926217f3a8dfb9398551ddacb0bc6aae6de0a164b215c55256eda9b6245e
    source_path: channels/discord.md
    workflow: 16
---

OpenClaw, resmi Discord gateway üzerinden bir bot olarak Discord'a bağlanır. DM'ler ve sunucu kanalları desteklenir.

<CardGroup cols={3}>
  <Card title="Eşleştirme" icon="link" href="/tr/channels/pairing">
    Discord DM'leri varsayılan olarak eşleştirme modunu kullanır.
  </Card>
  <Card title="Eğik çizgi komutları" icon="terminal" href="/tr/tools/slash-commands">
    Yerel komut davranışı ve komut kataloğu.
  </Card>
  <Card title="Kanal sorunlarını giderme" icon="wrench" href="/tr/channels/troubleshooting">
    Kanallar arası tanılama ve onarım akışı.
  </Card>
</CardGroup>

## Hızlı kurulum

Bot içeren bir Discord uygulaması oluşturun, botu sunucunuza ekleyin ve OpenClaw ile eşleştirin. Mümkünse özel bir sunucu kullanın; gerekirse önce [bir tane oluşturun](https://support.discord.com/hc/en-us/articles/204849977-How-do-I-create-a-server) (**Create My Own > For me and my friends**).

<Steps>
  <Step title="Bir Discord uygulaması ve bot oluşturun">
    [Discord Developer Portal](https://discord.com/developers/applications) içinde **New Application** öğesine tıklayın ve uygulamayı adlandırın (örneğin "OpenClaw").

    Kenar çubuğunda **Bot** öğesini açın ve **Username** alanını agent'ınızın adı olarak ayarlayın.

  </Step>

  <Step title="Ayrıcalıklı intent'leri etkinleştirin">
    **Bot** sayfasında, **Privileged Gateway Intents** altında şunları etkinleştirin:

    - **Message Content Intent** (zorunlu)
    - **Server Members Intent** (önerilir; rol izin listeleri, addan kimlik eşleştirme ve kanal hedef kitlesi erişim grupları için zorunludur)
    - **Presence Intent** (isteğe bağlı; yalnızca iletişim durumu güncellemeleri için)

  </Step>

  <Step title="Bot token'ınızı kopyalayın">
    **Bot** sayfasında **Reset Token** öğesine tıklayın ve token'ı kopyalayın.

    <Note>
    Adının aksine bu işlem ilk token'ınızı oluşturur; hiçbir şey "sıfırlanmaz."
    </Note>

  </Step>

  <Step title="Davet URL'si oluşturun ve botu sunucunuza ekleyin">
    Kenar çubuğunda **OAuth2** öğesini açın. **OAuth2 URL Generator** içinde şu kapsamları etkinleştirin:

    - `bot`
    - `applications.commands`

    Görüntülenen **Bot Permissions** bölümünde en azından şunları etkinleştirin:

    **General Permissions**
      - View Channels

    **Text Permissions**
      - Send Messages
      - Read Message History
      - Embed Links
      - Attach Files
      - Add Reactions (isteğe bağlı)

    Bu, normal metin kanalları için temel yapılandırmadır. Bot, ileti dizisi oluşturan veya mevcut bir ileti dizisini sürdüren forum ya da medya kanalı iş akışları dahil olmak üzere ileti dizilerine gönderi yayımlayacaksa **Send Messages in Threads** öğesini de etkinleştirin.

    Oluşturulan URL'yi kopyalayın, tarayıcıda açın, sunucunuzu seçin ve **Continue** öğesine tıklayın. Bot artık sunucunuzda görünmelidir.

  </Step>

  <Step title="Developer Mode'u etkinleştirin ve kimliklerinizi alın">
    Kimlikleri kopyalayabilmek için Discord uygulamasında Developer Mode'u etkinleştirin:

    1. **User Settings** (dişli simgesi) → **Developer** → **Developer Mode** seçeneğini açın
       *(mobilde: **App Settings** → **Advanced**)*
    2. **Sunucu simgenize** sağ tıklayın → **Copy Server ID**
    3. **Kendi avatarınıza** sağ tıklayın → **Copy User ID**

    Sunucu Kimliğini ve Kullanıcı Kimliğini bot token'ınızla birlikte saklayın; sonraki adımda üçünün de bulunması gerekir.

  </Step>

  <Step title="Sunucu üyelerinden gelen DM'lere izin verin">
    Eşleştirmenin çalışması için Discord, botun size DM göndermesine izin vermelidir. **Sunucu simgenize** sağ tıklayın → **Privacy Settings** → **Direct Messages** seçeneğini açın.

    OpenClaw ile Discord DM'lerini kullanıyorsanız bu seçeneği açık tutun. Yalnızca sunucu kanallarını kullanıyorsanız eşleştirmeden sonra devre dışı bırakabilirsiniz.

  </Step>

  <Step title="Bot token'ınızı güvenli biçimde ayarlayın (sohbette göndermeyin)">
    Bot token'ı gizli bir bilgidir. Agent'ınıza mesaj göndermeden önce OpenClaw çalıştıran makinede ayarlayın:

```bash
export DISCORD_BOT_TOKEN="YOUR_BOT_TOKEN"
cat > discord.patch.json5 <<'JSON5'
{
  channels: {
    discord: {
      enabled: true,
      token: { source: "env", provider: "default", id: "DISCORD_BOT_TOKEN" },
    },
  },
}
JSON5
openclaw config patch --file ./discord.patch.json5 --dry-run
openclaw config patch --file ./discord.patch.json5
openclaw gateway
```

    OpenClaw zaten arka plan hizmeti olarak çalışıyorsa OpenClaw Mac uygulaması üzerinden veya `openclaw gateway run` işlemini durdurup yeniden başlatarak yeniden başlatın.
    Yönetilen hizmet kurulumlarında `openclaw gateway install` komutunu `DISCORD_BOT_TOKEN` ayarlanmış bir kabuktan çalıştırın veya hizmetin yeniden başlatıldıktan sonra env SecretRef'i çözümleyebilmesi için değişkeni `~/.openclaw/.env` içinde saklayın.
    Ana makineniz Discord'un başlangıçtaki uygulama sorgusu tarafından engelleniyor veya hız sınırına tabi tutuluyorsa başlangıcın bu REST çağrısını atlayabilmesi için Developer Portal'daki uygulama/istemci kimliğini ayarlayın: varsayılan hesap için `channels.discord.applicationId`, bot başına ise `channels.discord.accounts.<accountId>.applicationId`.

  </Step>

  <Step title="OpenClaw'ı yapılandırın ve eşleştirin">

    <Tabs>
      <Tab title="Agent'ınıza sorun">
        Mevcut bir kanalda (örneğin Telegram) OpenClaw agent'ınızla sohbet edin ve talebinizi iletin. Discord ilk kanalınızsa bunun yerine CLI / yapılandırma sekmesini kullanın.

        > "Discord bot token'ımı yapılandırmada zaten ayarladım. Lütfen `<user_id>` Kullanıcı Kimliği ve `<server_id>` Sunucu Kimliği ile Discord kurulumunu tamamla."
      </Tab>
      <Tab title="CLI / yapılandırma">
        Dosya tabanlı yapılandırma:

```json5
{
  channels: {
    discord: {
      enabled: true,
      token: {
        source: "env",
        provider: "default",
        id: "DISCORD_BOT_TOKEN",
      },
    },
  },
}
```

        Varsayılan hesap için env geri dönüşü:

```bash
DISCORD_BOT_TOKEN=...
```

        Betikli veya uzak kurulum için aynı JSON5 bloğunu `openclaw config patch --file ./discord.patch.json5 --dry-run` ile yazın, ardından `--dry-run` olmadan yeniden çalıştırın. Düz metin `token` dizeleri de çalışır ve env/file/exec sağlayıcılarında `channels.discord.token` için SecretRef değerleri desteklenir. Bkz. [Gizli Bilgi Yönetimi](/tr/gateway/secrets).

        Birden fazla Discord botu için her botun token'ını ve uygulama kimliğini kendi hesabı altında tutun. Üst düzey `channels.discord.applicationId` hesaplar tarafından devralınır; bu nedenle yalnızca tüm hesaplar aynı uygulama kimliğini kullanıyorsa orada ayarlayın.

```json5
{
  channels: {
    discord: {
      enabled: true,
      accounts: {
        personal: {
          token: { source: "env", provider: "default", id: "DISCORD_PERSONAL_TOKEN" },
          applicationId: "111111111111111111",
        },
        work: {
          token: { source: "env", provider: "default", id: "DISCORD_WORK_TOKEN" },
          applicationId: "222222222222222222",
        },
      },
    },
  },
}
```

      </Tab>
    </Tabs>

  </Step>

  <Step title="İlk DM eşleştirmesini onaylayın">
    Gateway çalışmaya başladıktan sonra Discord'da botunuza DM gönderin. Bot bir eşleştirme koduyla yanıt verir.

    <Tabs>
      <Tab title="Agent'ınıza sorun">
        Eşleştirme kodunu mevcut kanalınızda agent'ınıza gönderin:

        > "Bu Discord eşleştirme kodunu onayla: `<CODE>`"
      </Tab>
      <Tab title="CLI">

```bash
openclaw pairing list discord
openclaw pairing approve discord <CODE>
```

      </Tab>
    </Tabs>

    Eşleştirme kodlarının süresi 1 saat sonra dolar. Onaydan sonra Discord DM'sinde agent'ınızla sohbet edin.

  </Step>
</Steps>

<Note>
Token çözümleme işlemi hesapları dikkate alır. Yapılandırmadaki token değerleri env geri dönüşüne göre önceliklidir ve `DISCORD_BOT_TOKEN` yalnızca varsayılan hesap için kullanılır.
Etkinleştirilmiş iki Discord hesabı aynı bot token'ına çözümlenirse OpenClaw bu token için yalnızca bir Gateway izleyicisi başlatır: yapılandırmadan alınan token env geri dönüşüne göre önceliklidir; aksi takdirde ilk etkin hesap öncelik kazanır ve yinelenen hesap `duplicate bot token` nedeniyle devre dışı bırakılmış olarak bildirilir.
Gelişmiş giden çağrılarda (mesaj aracı/kanal eylemleri), çağrı başına açıkça belirtilen `token` o çağrı için kullanılır. Bu, gönderme ve okuma/yoklama tarzı eylemler (okuma/arama/getirme/ileti dizisi/sabitlenenler/izinler) için geçerlidir. Hesap ilkesi/yeniden deneme ayarları yine etkin çalışma zamanı anlık görüntüsünde seçilen hesaptan alınır.
</Note>

## Önerilen: Bir sunucu çalışma alanı kurun

DM'ler çalışmaya başladıktan sonra sunucunuzu, her kanalın kendi bağlamıyla birlikte kendi agent oturumunu aldığı tam bir çalışma alanına dönüştürebilirsiniz. Yalnızca sizin ve botunuzun bulunduğu özel sunucular için önerilir.

<Steps>
  <Step title="Sunucunuzu sunucu izin listesine ekleyin">
    Bu, agent'ınızın yalnızca DM'lerde değil, sunucunuzdaki herhangi bir kanalda yanıt vermesini sağlar.

    <Tabs>
      <Tab title="Agent'ınıza sorun">
        > "Discord Sunucu Kimliğim `<server_id>` değerini sunucu izin listesine ekle"
      </Tab>
      <Tab title="Yapılandırma">

```json5
{
  channels: {
    discord: {
      groupPolicy: "allowlist",
      guilds: {
        YOUR_SERVER_ID: {
          requireMention: true,
          users: ["YOUR_USER_ID"],
        },
      },
    },
  },
}
```

      </Tab>
    </Tabs>

  </Step>

  <Step title="@bahsetme olmadan yanıtlara izin verin">
    Varsayılan olarak agent, sunucu kanallarında yalnızca kendisinden @bahsedildiğinde yanıt verir. Özel bir sunucuda muhtemelen her mesaja yanıt vermesini istersiniz.

    Sunucu kanallarında normal yanıtlar varsayılan olarak otomatik biçimde gönderilir. Her zaman etkin olan paylaşımlı odalarda agent'ın sessizce takip edebilmesi ve yalnızca kanal yanıtının yararlı olduğuna karar verdiğinde gönderi yayımlaması için `messages.groupChat.visibleReplies: "message_tool"` özelliğini etkinleştirin. Bu özellik, GPT-5.6 Sol gibi en yeni nesil ve araç kullanımında güvenilir modellerle en iyi şekilde çalışır. Araç göndermediği sürece ortam odası olayları sessiz kalır. Tam sessiz takip modu yapılandırması için [Ortam odası olayları](/tr/channels/ambient-room-events) bölümüne bakın.

    Discord yazıyor göstergesini gösteriyor ve günlükler token kullanımını gösteriyor ancak mesaj yayımlanmıyorsa turun bir ortam odası olayı olarak yapılandırılıp yapılandırılmadığını veya mesaj aracıyla görünür yanıtların etkinleştirilip etkinleştirilmediğini kontrol edin.

    <Tabs>
      <Tab title="Agent'ınıza sorun">
        > "Agent'ımın @bahsedilmesine gerek kalmadan bu sunucuda yanıt vermesine izin ver"
      </Tab>
      <Tab title="Yapılandırma">
        Sunucu yapılandırmanızda `requireMention: false` değerini ayarlayın:

```json5
{
  channels: {
    discord: {
      guilds: {
        YOUR_SERVER_ID: {
          requireMention: false,
        },
      },
    },
  },
}
```

        Görünür grup/kanal yanıtları için mesaj aracıyla gönderimi zorunlu kılmak üzere `messages.groupChat.visibleReplies: "message_tool"` değerini ayarlayın.

      </Tab>
    </Tabs>

  </Step>

  <Step title="Sunucu kanallarındaki belleği planlayın">
    Uzun süreli bellek (MEMORY.md) yalnızca DM oturumlarında otomatik olarak yüklenir; sunucu kanalları bunu yüklemez.

    <Tabs>
      <Tab title="Agent'ınıza sorun">
        > "Discord kanallarında soru sorduğumda MEMORY.md dosyasındaki uzun süreli bağlama ihtiyacın olursa memory_search veya memory_get kullan."
      </Tab>
      <Tab title="Manuel">
        Her kanalda paylaşılan bağlam için kararlı talimatları `AGENTS.md` veya `USER.md` içine yerleştirin (her oturuma eklenir). Uzun süreli notları `MEMORY.md` içinde tutun ve gerektiğinde bellek araçlarıyla bunlara erişin.
      </Tab>
    </Tabs>

  </Step>
</Steps>

Şimdi kanallar oluşturun ve sohbet etmeye başlayın. Agent kanal adını görür ve her kanal yalıtılmış bir oturumdur; iş akışınıza uygun olarak `#coding`, `#home`, `#research` veya başka kanallar oluşturun.

## Çalışma zamanı modeli

- Gateway, Discord bağlantısının sahibidir.
- Yanıt yönlendirmesi belirlenimseldir: Discord'dan gelen iletilerin yanıtları Discord'a geri gönderilir.
- Discord sunucu/kanal meta verileri, kullanıcıya görünür yanıt ön eki olarak değil, güvenilmeyen bağlam olarak model istemine eklenir. Bir model bu zarfı yanıta kopyalarsa OpenClaw, kopyalanan meta verileri giden yanıtlardan ve gelecekte yeniden oynatılacak bağlamdan kaldırır.
- Varsayılan olarak (`session.dmScope=main`), doğrudan sohbetler agent'ın ana oturumunu (`agent:main:main`) paylaşır.
- Sunucu kanalları yalıtılmış oturum anahtarlarıdır (`agent:<agentId>:discord:channel:<channelId>`).
- Grup DM'leri varsayılan olarak yok sayılır (`channels.discord.dm.groupEnabled=false`).
- Yerel eğik çizgi komutları yalıtılmış komut oturumlarında (`agent:<agentId>:discord:slash:<userId>`) çalışırken `CommandTargetSessionKey` değerini yönlendirilen konuşma oturumuna taşımaya devam eder.
- Discord'a yalnızca metin içeren Cron/Heartbeat duyurularının teslimi, bir kez gönderilen ve asistana görünür olan son yanıta indirgenir. Agent birden fazla teslim edilebilir yük oluşturduğunda medya ve yapılandırılmış bileşen yükleri birden fazla mesaj olarak kalır.

## Forum kanalları

Discord forum ve medya kanalları yalnızca ileti dizisi gönderilerini kabul eder. OpenClaw bunları oluşturmak için iki yöntem destekler:

- Bir ileti dizisini otomatik olarak oluşturmak için forum üst kanalına (`channel:<forumId>`) mesaj gönderin. İleti dizisi başlığı, mesajın boş olmayan ilk satırıdır (Discord'un 100 karakterlik ileti dizisi adı sınırına göre kısaltılır).
- Doğrudan bir ileti dizisi oluşturmak için `openclaw message thread create` kullanın. Forum kanalları için `--message-id` iletmeyin.

Bir ileti dizisi oluşturmak için forum üst kanalına gönderin:

```bash
openclaw message send --channel discord --target channel:<forumId> \
  --message "Konu başlığı\nGönderinin içeriği"
```

Açıkça bir forum ileti dizisi oluşturun:

```bash
openclaw message thread create --channel discord --target channel:<forumId> \
  --thread-name "Konu başlığı" --message "Gönderinin içeriği"
```

Forum üst kanalları Discord bileşenlerini kabul etmez. Bileşenlere ihtiyacınız varsa doğrudan ileti dizisine (`channel:<threadId>`) gönderin.

## Etkileşimli bileşenler

OpenClaw, ajan mesajları için Discord bileşenleri v2 kapsayıcılarını destekler. Mesaj aracını bir `components` yüküyle kullanın. Etkileşim sonuçları normal gelen mesajlar olarak ajana geri yönlendirilir ve mevcut Discord `replyToMode` ayarlarını izler.

Desteklenen bloklar:

- `text`, `section`, `separator`, `actions`, `media-gallery`, `file`
- Eylem satırları en fazla 5 düğmeye veya tek bir seçim menüsüne izin verir
- Seçim türleri: `string`, `user`, `role`, `mentionable`, `channel`

Bileşenler varsayılan olarak tek kullanımlıktır. Düğmelerin, seçimlerin ve formların süreleri dolana kadar birden çok kez kullanılmasına izin vermek için `components.reusable=true` ayarını yapın.

Bir düğmeye kimlerin tıklayabileceğini kısıtlamak için o düğmede `allowedUsers` ayarını yapın (Discord kullanıcı kimlikleri, etiketleri veya `*`). Eşleşmeyen kullanıcılar yalnızca kendilerine görünen bir ret bildirimi alır.

Bileşen geri çağrılarının süresi varsayılan olarak 30 dakika sonra dolar. Varsayılan hesabın geri çağrı kayıt defteri ömrünü değiştirmek için `channels.discord.agentComponents.ttlMs`, hesap başına değiştirmek için ise `channels.discord.accounts.<accountId>.agentComponents.ttlMs` ayarını yapın. Değer milisaniye cinsindedir, pozitif bir tam sayı olmalıdır ve `86400000` (24 saat) ile sınırlandırılmıştır. Daha uzun TTL'ler, düğmelerin kullanılabilir kalmasını gerektiren inceleme/onay iş akışlarına uygundur; ancak eski bir Discord mesajının hâlâ bir eylemi tetikleyebileceği süreyi uzatırlar. İhtiyacı karşılayan en kısa TTL'yi tercih edin ve eski geri çağrıların beklenmedik olacağı durumlarda varsayılanı koruyun.

`/model` ve `/models` eğik çizgi komutları; sağlayıcı, model ve uyumlu çalışma zamanı açılır listelerinin yanı sıra bir Gönder adımı içeren etkileşimli bir model seçici açar. `/models add` kullanım dışıdır ve modelleri sohbetten kaydetmek yerine kullanımdan kaldırma mesajı döndürür. Seçici yanıtı yalnızca çağıran kullanıcı tarafından görülebilir ve kullanılabilir. Discord seçim menüleri 25 seçenekle sınırlıdır; bu nedenle seçicinin dinamik olarak keşfedilen modelleri yalnızca `openai` veya `vllm` gibi seçili sağlayıcılar için göstermesini istediğinizde `agents.defaults.modelPolicy.allow` içine `provider/*` girdileri ekleyin.

Dosya ekleri:

- `file` blokları bir ek referansını (`attachment://<filename>`) göstermelidir
- Eki `media`/`path`/`filePath` üzerinden sağlayın (tek dosya); birden çok dosya için `media-gallery` kullanın
- Yükleme adının ek referansıyla eşleşmesi gerektiğinde adı geçersiz kılmak için `filename` kullanın

Kalıcı pencere formları:

- En fazla 5 alanla `components.modal` ekleyin
- Alan türleri: `text`, `checkbox`, `radio`, `select`, `role-select`, `user-select`
- OpenClaw otomatik olarak bir tetikleme düğmesi ekler

Örnek:

```json5
{
  channel: "discord",
  action: "send",
  to: "channel:123456789012345678",
  message: "İsteğe bağlı yedek metin",
  components: {
    reusable: true,
    text: "Bir yol seçin",
    blocks: [
      {
        type: "actions",
        buttons: [
          {
            label: "Onayla",
            style: "success",
            allowedUsers: ["123456789012345678"],
          },
          { label: "Reddet", style: "danger" },
        ],
      },
      {
        type: "actions",
        select: {
          type: "string",
          placeholder: "Bir seçenek belirleyin",
          options: [
            { label: "Seçenek A", value: "a" },
            { label: "Seçenek B", value: "b" },
          ],
        },
      },
    ],
    modal: {
      title: "Ayrıntılar",
      triggerLabel: "Formu aç",
      fields: [
        { type: "text", label: "Talep eden" },
        {
          type: "select",
          label: "Öncelik",
          options: [
            { label: "Düşük", value: "low" },
            { label: "Yüksek", value: "high" },
          ],
        },
      ],
    },
  },
}
```

## Erişim denetimi ve yönlendirme

<Tabs>
  <Tab title="DM ilkesi">
    `channels.discord.dmPolicy`, DM erişimini denetler. `channels.discord.allowFrom` standart DM izin listesidir.

    - `pairing` (varsayılan)
    - `allowlist` (en az bir `allowFrom` göndericisi gerektirir)
    - `open` (`channels.discord.allowFrom` değerinin `"*"` içermesini gerektirir)
    - `disabled`

    DM ilkesi açık değilse bilinmeyen kullanıcılar engellenir (veya `pairing` modunda eşleştirme yapmaları istenir).

    Çok hesaplı kullanımda öncelik:

    - `channels.discord.accounts.default.allowFrom` yalnızca `default` hesabına uygulanır.
    - Bir hesap için `allowFrom`, eski `dm.allowFrom` değerine göre önceliklidir.
    - Adlandırılmış hesapların kendi `allowFrom` ve eski `dm.allowFrom` değerleri ayarlanmamışsa bu hesaplar `channels.discord.allowFrom` değerini devralır.
    - Adlandırılmış hesaplar `channels.discord.accounts.default.allowFrom` değerini devralmaz.

    Eski `channels.discord.dm.policy` ve `channels.discord.dm.allowFrom` uyumluluk amacıyla okunmaya devam eder. `openclaw doctor --fix`, erişimi değiştirmeden yapabildiği durumlarda bunları `dmPolicy` ve `allowFrom` konumuna taşır.

    Teslimat için DM hedef biçimi:

    - `user:<id>`
    - `<@id>` bahsetmesi

    Yalın sayısal kimlikler, bir kanal varsayılanı etkinken normalde kanal kimlikleri olarak çözümlenir; ancak hesabın etkin DM `allowFrom` listesinde bulunan kimlikler, uyumluluk amacıyla kullanıcı DM hedefleri olarak değerlendirilir.

  </Tab>

  <Tab title="Erişim grupları">
    Discord DM'leri ve metin komutu yetkilendirmesi, `channels.discord.allowFrom` içindeki dinamik `accessGroup:<name>` girdilerini kullanabilir.

    Erişim grubu adları mesaj kanalları arasında paylaşılır. Üyeleri her kanalın normal `allowFrom` söz dizimiyle ifade edilen statik bir grup için `type: "message.senders"` kullanın veya bir Discord kanalının mevcut `ViewChannel` kitlesinin üyeliği dinamik olarak belirlemesi gerektiğinde `type: "discord.channelAudience"` kullanın. Paylaşılan erişim grubu davranışı: [Erişim grupları](/tr/channels/access-groups).

```json5
{
  accessGroups: {
    operators: {
      type: "message.senders",
      members: {
        "*": ["global-owner-id"],
        discord: ["discord:123456789012345678"],
        telegram: ["987654321"],
      },
    },
  },
  channels: {
    discord: {
      dmPolicy: "allowlist",
      allowFrom: ["accessGroup:operators"],
    },
  },
}
```

    Bir Discord metin kanalının ayrı bir üye listesi yoktur. `type: "discord.channelAudience"` üyeliği şu şekilde modeller: DM göndericisi yapılandırılmış sunucunun bir üyesidir ve rol ile kanal geçersiz kılmaları uygulandıktan sonra yapılandırılmış kanalda etkin `ViewChannel` iznine sahiptir.

    Örnek: `#maintainers` kanalını görebilen herkesin bota DM göndermesine izin verirken DM'leri diğer herkese kapalı tutun.

```json5
{
  accessGroups: {
    maintainers: {
      type: "discord.channelAudience",
      guildId: "1456350064065904867",
      channelId: "1456744319972282449",
      membership: "canViewChannel",
    },
  },
  channels: {
    discord: {
      dmPolicy: "allowlist",
      allowFrom: ["accessGroup:maintainers"],
    },
  },
}
```

    Dinamik ve statik girdileri birlikte kullanabilirsiniz:

```json5
{
  accessGroups: {
    maintainers: {
      type: "discord.channelAudience",
      guildId: "1456350064065904867",
      channelId: "1456744319972282449",
    },
  },
  channels: {
    discord: {
      dmPolicy: "allowlist",
      allowFrom: ["accessGroup:maintainers", "discord:123456789012345678"],
    },
  },
}
```

    Aramalar başarısızlık durumunda erişimi kapatır. Discord `Missing Access` döndürürse, üye araması başarısız olursa veya kanal farklı bir sunucuya aitse DM göndericisi yetkisiz kabul edilir.

    Kanal kitlesi erişim gruplarını kullanırken Discord Developer Portal'da **Server Members Intent** seçeneğini etkinleştirin. DM'ler sunucu üyesi durumunu içermez; bu nedenle OpenClaw, yetkilendirme sırasında üyeyi Discord REST üzerinden çözümler.

  </Tab>

  <Tab title="Sunucu ilkesi">
    Sunucu işleme davranışı `channels.discord.groupPolicy` tarafından denetlenir:

    - `open`
    - `allowlist`
    - `disabled`

    `channels.discord` mevcut olduğundaki güvenli temel değer `allowlist` şeklindedir.

    `allowlist` davranışı:

    - sunucu `channels.discord.guilds` ile eşleşmelidir (`id` tercih edilir, kısa ad kabul edilir)
    - isteğe bağlı gönderici izin listeleri: `users` (kararlı kimlikler önerilir) ve `roles` (yalnızca rol kimlikleri); bunlardan herhangi biri yapılandırılmışsa göndericiler `users` VEYA `roles` ile eşleştiklerinde kabul edilir
    - doğrudan ad/etiket eşleştirmesi varsayılan olarak devre dışıdır; `channels.discord.dangerouslyAllowNameMatching: true` değerini yalnızca acil durum uyumluluk modu olarak etkinleştirin
    - adlar/etiketler `users` için desteklenir ancak kimlikler daha güvenlidir; ad/etiket girdileri kullanıldığında `openclaw security audit` uyarı verir
    - bir sunucuda `channels` yapılandırılmışsa listelenmeyen kanallar reddedilir
    - bir sunucuda `channels` bloğu yoksa izin listesindeki o sunucunun tüm kanallarına izin verilir

    Örnek:

```json5
{
  channels: {
    discord: {
      groupPolicy: "allowlist",
      guilds: {
        "123456789012345678": {
          requireMention: true,
          ignoreOtherMentions: true,
          users: ["987654321098765432"],
          roles: ["123456789012345678"],
          channels: {
            general: { enabled: true },
            help: { enabled: true, requireMention: true },
          },
        },
      },
    },
  },
}
```

    Eski kanal başına `allow` anahtarı, `openclaw doctor --fix` tarafından `enabled` konumuna taşınır.

    Yalnızca `DISCORD_BOT_TOKEN` ayarlayıp bir `channels.discord` bloğu oluşturmazsanız `channels.defaults.groupPolicy`, `open` olsa bile çalışma zamanı geri dönüş değeri `groupPolicy="allowlist"` olur (günlüklerde bir uyarıyla).

  </Tab>

  <Tab title="Bahsetmeler ve grup DM'leri">
    Sunucu mesajları varsayılan olarak bahsetme koşuluna bağlıdır.

    Bahsetme algılaması şunları içerir:

    - botun açıkça belirtilmesi
    - yapılandırılmış bahsetme kalıpları (`agents.entries.*.groupChat.mentionPatterns`, geri dönüş `messages.groupChat.mentionPatterns`)
    - desteklenen durumlarda örtük bota yanıt davranışı

    Giden Discord mesajlarını yazarken standart bahsetme söz dizimini kullanın: kullanıcılar için `<@USER_ID>`, kanallar için `<#CHANNEL_ID>` ve roller için `<@&ROLE_ID>`. Eski `<@!USER_ID>` takma ad bahsetme biçimini kullanmayın.

    `requireMention` sunucu/kanal başına yapılandırılır (`channels.discord.guilds...`).
    `ignoreOtherMentions`, isteğe bağlı olarak başka bir kullanıcıdan/rolden bahsedip bottan bahsetmeyen mesajları yok sayar (@everyone/@here hariç).

    Grup DM'leri:

    - varsayılan: yok sayılır (`dm.groupEnabled=false`)
    - `dm.groupChannels` üzerinden isteğe bağlı izin listesi (kanal kimlikleri veya kısa adlar)

  </Tab>
</Tabs>

### Rol tabanlı ajan yönlendirmesi

Discord sunucu üyelerini rol kimliğine göre farklı ajanlara yönlendirmek için `bindings[].match.roles` kullanın. Rol tabanlı bağlamalar yalnızca rol kimliklerini kabul eder ve eş veya üst eş bağlamalarından sonra, yalnızca sunucu bağlamalarından önce değerlendirilir. Bir bağlama başka eşleşme alanları da ayarlıyorsa (örneğin `peer` + `guildId` + `roles`), yapılandırılmış tüm alanlar eşleşmelidir.

```json5
{
  bindings: [
    {
      agentId: "opus",
      match: {
        channel: "discord",
        guildId: "123456789012345678",
        roles: ["111111111111111111"],
      },
    },
    {
      agentId: "sonnet",
      match: {
        channel: "discord",
        guildId: "123456789012345678",
      },
    },
  ],
}
```

## Yerel komutlar ve komut kimlik doğrulaması

- `commands.native` varsayılan olarak `"auto"` değerini alır ve Discord için etkindir.
- Kanal bazında geçersiz kılma: `channels.discord.commands.native`.
- `commands.native=false`, başlangıç sırasında Discord eğik çizgi komutlarının kaydedilmesini ve temizlenmesini atlar. Daha önce kaydedilmiş komutlar, Discord uygulamasından kaldırılana kadar Discord'da görünür kalabilir.
- Yerel komut kimlik doğrulaması, normal mesaj işleme ile aynı Discord izin listelerini/ilkelerini kullanır.
- Komutlar, yetkisiz kullanıcılar için Discord kullanıcı arayüzünde görünmeye devam edebilir; yürütme sırasında OpenClaw kimlik doğrulaması uygulanır ve "yetkili değil" yanıtı verilir.
- Varsayılan eğik çizgi komutu ayarları: `ephemeral: true` (`channels.discord.slashCommand.ephemeral`).

Komut kataloğu ve davranışı için [Eğik çizgi komutları](/tr/tools/slash-commands) bölümüne bakın.

## Özellik ayrıntıları

<AccordionGroup>
  <Accordion title="Yanıt etiketleri ve yerel yanıtlar">
    Discord, agent çıktısındaki yanıt etiketlerini destekler:

    - `[[reply_to_current]]`
    - `[[reply_to:<id>]]`

    `channels.discord.replyToMode` tarafından denetlenir:

    - `off` (varsayılan): örtük yanıt iş parçacığı oluşturulmaz; açık `[[reply_to_*]]` etiketleri yine de dikkate alınır
    - `first`: örtük yerel yanıt referansını turun ilk giden Discord mesajına ekler
    - `all`: bunu giden her mesaja ekler
    - `batched`: bunu yalnızca gelen olay, gecikmeli olarak toplu işlenen birden fazla mesajdan oluştuğunda ekler — yerel yanıtları her tek mesajlı turda değil, öncelikle belirsiz ve yoğun mesaj akışlı sohbetlerde kullanmak istediğinizde yararlıdır

    Mesaj kimlikleri, agent'ların belirli mesajları hedefleyebilmesi için bağlamda/geçmişte sunulur.

  </Accordion>

  <Accordion title="Bağlantı önizlemeleri">
    Discord, varsayılan olarak URL'ler için zengin bağlantı yerleştirmeleri oluşturur. OpenClaw, agent tarafından gönderilen URL'lerin siz etkinleştirmediğiniz sürece düz bağlantılar olarak kalması için giden Discord mesajlarında oluşturulan bu yerleştirmeleri varsayılan olarak engeller:

```json5
{
  channels: {
    discord: {
      suppressEmbeds: false,
    },
  },
}
```

    Tek bir hesabı geçersiz kılmak için `channels.discord.accounts.<id>.suppressEmbeds` değerini ayarlayın. Agent mesaj aracı gönderimleri, tek bir mesaj için `suppressEmbeds: false` de iletebilir. Açık Discord `embeds` yükleri, varsayılan bağlantı önizleme ayarı tarafından engellenmez.

  </Accordion>

  <Accordion title="Canlı akış önizlemesi">
    OpenClaw, geçici bir mesaj gönderip metin geldikçe bu mesajı düzenleyerek taslak yanıtları aktarabilir. `channels.discord.streaming.mode`, `off` | `partial` | `block` | `progress` değerlerini alır (herhangi bir `streaming`/eski `streamMode` anahtarı ayarlanmamışsa varsayılan). `streamMode` eski bir diğer addır; kalıcı yapılandırmayı standart iç içe `streaming` biçimine yeniden yazmak için `openclaw doctor --fix` komutunu çalıştırın.

```json5
{
  channels: {
    discord: {
      streaming: {
        mode: "progress",
        progress: {
          maxLines: 8,
          maxLineChars: 120,
          toolProgress: false,
          commentary: false,
        },
      },
    },
  },
}
```

    - `off`, Discord önizleme düzenlemelerini devre dışı bırakır.
    - `partial`, belirteçler geldikçe tek bir önizleme mesajını düzenler.
    - `block`, taslak boyutunda parçalar gönderir; boyutu ve kesme noktalarını `textChunkLimit` ile sınırlanan `streaming.preview.chunk` (`minChars`, `maxChars`, `breakPreference`) ile ayarlayın. Blok akışı açıkça etkinleştirildiğinde OpenClaw, çift akışı önlemek için önizleme akışını atlar.
    - `progress`, son teslimata kadar düzenlenebilir tek bir durum taslağını korur. Varsayılan olarak agent'ın en son ön açıklamasından veya anlatımından bir satır gösterir; oluşturulmuş etiket, ayırıcı ya da araç satırı göstermez.
    - Medya, hata ve açık yanıt sonlandırmaları, bekleyen önizleme düzenlemelerini iptal eder.
    - `streaming.preview.toolProgress`, `partial`/`block` modunda varsayılan olarak `true` değerini alır. Discord ilerleme modunda varsayılan olarak araç satırları yoktur; etkinleştirmek için `streaming.progress.toolProgress: true` değerini ayarlayın.
    - `🛠️ Bash: run tests` veya `🔎 Web Search: for "query"` gibi kompakt araç/ilerleme satırları eklemek için `streaming.progress.toolProgress: true` değerini ayarlayın. Uyumluluk amacıyla mevcut bir `progress.label` veya `progress.labels` yapılandırması önceki araç satırı varsayılanını korur; satır olmadan özel bir etiket için `toolProgress: false` değerini ayarlayın.
    - `streaming.progress.commentary` (varsayılan `false`), geçici ilerleme taslağında işlenmemiş asistan yorumlarını etkinleştirir. Varsayılan ön açıklama/anlatım durum satırı bu seçenekten bağımsızdır. Yorumlar görüntülenmeden önce temizlenir, geçici kalır ve nihai yanıtın teslimini değiştirmez.
    - `streaming.progress.maxLineChars`, satır başına ilerleme önizleme bütçesini denetler. Düzyazı, sözcük sınırlarında kısaltılır; komut ve yol ayrıntılarının yararlı son ekleri korunur.
    - `streaming.preview.commandText` / `streaming.progress.commandText`, kompakt ilerleme satırlarındaki komut/yürütme ayrıntısını denetler: `raw` (varsayılan) veya `status` (yalnızca araç etiketi).

    Kompakt ilerleme satırlarını korurken işlenmemiş komut/yürütme metnini gizleyin:

    ```json
    {
      "channels": {
        "discord": {
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

    Önizleme akışı yalnızca metin içindir; medya yanıtları normal teslime geri döner.

  </Accordion>

  <Accordion title="Geçmiş, bağlam ve iş parçacığı davranışı">
    Sunucu geçmişi bağlamı:

    - `channels.discord.historyLimit` varsayılan `20`
    - geri dönüş: `messages.groupChat.historyLimit`
    - `0` devre dışı bırakır

    DM geçmişi denetimleri:

    - `channels.discord.dmHistoryLimit`
    - `channels.discord.dms["<user_id>"].historyLimit`

    İş parçacığı davranışı:

    - Discord iş parçacıkları kanal oturumları olarak yönlendirilir ve geçersiz kılınmadığı sürece üst kanal yapılandırmasını devralır.
    - İş parçacığı oturumları, yalnızca modele yönelik bir geri dönüş olarak üst kanalın oturum düzeyindeki `/model` seçimini devralır; iş parçacığına özgü `/model` seçimleri önceliklidir ve transkript devralma etkinleştirilmedikçe üst transkript geçmişi kopyalanmaz.
    - `channels.discord.thread.inheritParent` (varsayılan `false`), yeni otomatik iş parçacıklarının üst transkriptle başlatılmasını etkinleştirir. Hesap bazında geçersiz kılma: `channels.discord.accounts.<id>.thread.inheritParent`.
    - Mesaj aracı tepkileri, `user:<id>` DM hedeflerini çözümleyebilir.
    - `guilds.<guild>.channels.<channel>.requireMention: false`, yanıt aşaması etkinleştirme geri dönüşü sırasında korunur.

    Kanal konuları, **güvenilmeyen** bağlam olarak eklenir. İzin listeleri, agent'ı kimlerin tetikleyebileceğini sınırlar; eksiksiz bir ek bağlam sansürleme sınırı değildir.

  </Accordion>

  <Accordion title="Alt agent'lar için iş parçacığına bağlı oturumlar">
    Discord, bir iş parçacığını oturum hedefine bağlayabilir; böylece o iş parçacığındaki takip mesajları aynı oturuma (alt agent oturumları dâhil) yönlendirilmeye devam eder.

    Komutlar:

    - `/focus <target>`, mevcut/yeni iş parçacığını bir alt agent/oturum hedefine bağlar
    - `/unfocus`, mevcut iş parçacığı bağını kaldırır
    - `/agents`, etkin çalıştırmaları ve bağ durumunu gösterir
    - `/session idle <duration|off>`, odaklanmış bağlar için hareketsizlikte otomatik odaktan çıkarma ayarını inceler/günceller
    - `/session max-age <duration|off>`, odaklanmış bağlar için kesin azami yaşı inceler/günceller

    Yapılandırma:

```json5
{
  session: {
    threadBindings: {
      enabled: true,
      idleHours: 24,
      maxAgeHours: 0,
      spawnSessions: true,
      defaultSpawnContext: "fork",
    },
  },
}
```

    Notlar:

    - `session.threadBindings.*`, Discord ve Telegram için standart ilkedir.
    - `spawnSessions`, `sessions_spawn({ thread: true })` ve ACP iş parçacığı başlatmaları için iş parçacıklarının otomatik oluşturulmasını/bağlanmasını denetler. Varsayılan: `true`.
    - `defaultSpawnContext`, iş parçacığına bağlı başlatmaların yerel alt agent bağlamını denetler. Varsayılan: `"fork"`.
    - Kullanımdan kaldırılmış `spawnSubagentSessions`/`spawnAcpSessions` anahtarları, `openclaw doctor --fix` tarafından taşınır.
    - İş parçacığı bağları devre dışıysa `/focus` ve ilgili işlemler kullanılamaz.

    [Alt agent'lar](/tr/tools/subagents), [ACP Agent'ları](/tr/tools/acp-agents) ve [Yapılandırma Başvurusu](/tr/gateway/configuration-reference) bölümlerine bakın.

  </Accordion>

  <Accordion title="Kaynak mesajdaki alt agent ilerlemesi">
    Üst çalıştırmayı başlatan Discord mesajında arka plandaki alt etkinliği göstermek için `channels.discord.subagentProgress: true` değerini ayarlayın.

```json5
{
  channels: {
    discord: {
      subagentProgress: true,
    },
  },
}
```

    Alt çalıştırmalar etkinken OpenClaw, Discord yazıyor göstergesini bir saate kadar etkin tutar ve eşzamanlı sayı değiştikçe tek bir sayı tepkisini (`1️⃣` ile `🔟` arasında) değiştirir; `🔟` ayrıca 10 veya daha fazlasını temsil eder. Son alt çalıştırma bittikten sonra sayı tepkisi kaldırılır. Başarısız olan, zaman aşımına uğrayan veya sonlandırılan bir alt çalıştırma, `🔴` tepkisi bırakır.

    Bu özellik isteğe bağlıdır ve sabit dâhilî zamanlama ile emoji varsayılanlarını kullanır. Tepki geri bildirimi için botun **Add Reactions** iznine ihtiyacı vardır. Hesap düzeyindeki `channels.discord.accounts.<id>.subagentProgress`, üst düzey değeri geçersiz kılar.

  </Accordion>

  <Accordion title="Kalıcı ACP kanal bağları">
    Kararlı ve "her zaman açık" ACP çalışma alanları için Discord konuşmalarını hedefleyen üst düzey türü belirlenmiş ACP bağlarını yapılandırın.

    Yapılandırma yolu: `type: "acp"` ve `match.channel: "discord"` ile `bindings[]`.

```json5
{
  agents: {
    entries: {
      codex: {
        runtime: {
          type: "acp",
          acp: {
            agent: "codex",
            backend: "acpx",
            mode: "persistent",
            cwd: "/workspace/openclaw",
          },
        },
      },
    },
  },
  bindings: [
    {
      type: "acp",
      agentId: "codex",
      match: {
        channel: "discord",
        accountId: "default",
        peer: { kind: "channel", id: "222222222222222222" },
      },
      acp: { label: "codex-main" },
    },
  ],
  channels: {
    discord: {
      guilds: {
        "111111111111111111": {
          channels: {
            "222222222222222222": {
              requireMention: false,
            },
          },
        },
      },
    },
  },
}
```

    Notlar:

    - `/acp spawn codex --bind here`, mevcut kanalı veya iş parçacığını yerinde bağlar ve gelecekteki mesajları aynı ACP oturumunda tutar. İş parçacığı mesajları üst kanal bağını devralır.
    - Bağlı bir kanalda veya iş parçacığında `/new` ve `/reset`, aynı ACP oturumunu yerinde sıfırlar. Geçici iş parçacığı bağları etkinken hedef çözümlemesini geçersiz kılabilir.
    - `spawnSessions`, `--thread auto|here` aracılığıyla alt iş parçacığı oluşturulmasını/bağlanmasını denetler.

    Bağ davranışı ayrıntıları için [ACP Agent'ları](/tr/tools/acp-agents) bölümüne bakın.

  </Accordion>

  <Accordion title="Tepki bildirimleri">
    Sunucu bazında tepki bildirimi modu (`guilds.<id>.reactionNotifications`):

    - `off`
    - `own` (varsayılan)
    - `all`
    - `allowlist` (`guilds.<id>.users` kullanır)

    Tepki olayları sistem olaylarına dönüştürülür ve yönlendirilen Discord oturumuna eklenir.

  </Accordion>

  <Accordion title="Çevrimiçi bulunma olayları">
    Bir insan üye çevrimdışından çevrimiçine geçtiğinde yönlendirilmiş agent uyandırmalarını almak üzere bir sunucuyu etkinleştirin:

    ```json5
    {
      channels: {
        discord: {
          intents: { presence: true },
          guilds: {
            "111111111111111111": {
              presenceEvents: {
                channelId: "222222222222222222",
                users: ["333333333333333333"], // isteğe bağlı; kanal görüntüleyicilerini daha da daraltır
                reconnectSuppressSeconds: 300, // isteğe bağlı; yeni oturum sessizlik aralığı (0 devre dışı bırakır)
                burstLimit: 8, // isteğe bağlı; ani yoğunluk aralığı başına maksimum olay
                burstWindowSeconds: 60, // isteğe bağlı; kayan ani yoğunluk algılama aralığı
              },
            },
          },
        },
      },
    }
    ```

    `presenceEvents`, yönlendirilen aracı için etkinleştirilmiş bir Heartbeat ve Discord Developer Portal'daki uygulamanın Bot sayfasında ayrıcalıklı **Presence Intent** gerektirir. OpenClaw, her eksiksiz `GUILD_CREATE` anlık görüntüsünden o anda çevrimiçi olan üyeleri başlangıç durumuna ekler, gözlemlenen çevrimdışından çevrimiçine geçişleri yönlendirir ve daha önce görülmemiş bir üyeden sonradan gelen ilk çevrimiçi sinyalini de yeni erişilebilirlik olarak değerlendirir. Bu üye, anlık görüntüden sonra çevrimiçi olmuş veya katılmış olabilir; bu nedenle olay, önceki kesin bir durumu ileri sürmez. Yalnızca `channelId` öğesini görüntüleyebilen insanlar uygundur: kanallar ve herkese açık ileti dizileri, kanalda veya üst öğesinde **View Channel** izni gerektirirken özel ileti dizileri ek olarak üyelik veya **Manage Threads** izni gerektirir. `users` bu kitleyi daha da daraltabilir. OpenClaw, botları ve değişmeyen çevrimiçi durumları yok sayar ve kullanıcı başına sekiz saatlik bekleme süresini Gateway yeniden başlatmaları boyunca kalıcı olarak saklar. Discord yeni bir Gateway oturumu kurup `READY` gönderdiğinde OpenClaw, lonca iletişim durumu yeniden oluşturulurken `reconnectSuppressSeconds` boyunca (varsayılan 300, `0` devre dışı bırakır) iletişim durumundan türetilen olayları bastırır; böylece yeniden gözlemlenen üyeler aracıyı tek tek uyandıramaz. Ayrıca başarıyla kuyruğa alınan olayları lonca başına, her `burstWindowSeconds` kayan aralığında (varsayılan 60) `burstLimit` olayla (varsayılan 8) sınırlar ve her loncanın bastırma dönemini bir kez günlüğe kaydeder. Devam ettirilen bir oturum yeni oturum olarak değerlendirilmez. Discord, 75,000'den fazla üyesi olan loncaların anlık görüntülerini sınırlar; OpenClaw burada selamlamadan önce açık bir çevrimdışı güncelleme gerektirir. Sistem olayı, değişken görünen adları gömmeden değişmez kullanıcı, lonca ve kanal kimliklerini taşır. Selam verilip verilmeyeceğine ve nasıl verileceğine aracı karar verir.

  </Accordion>

  <Accordion title="Alındı tepkileri">
    `ackReaction`, OpenClaw gelen bir mesajı işlerken bir alındı emojisi gönderir.

    Çözümleme sırası:

    - `channels.discord.accounts.<accountId>.ackReaction`
    - `channels.discord.ackReaction`
    - `messages.ackReaction`
    - aracı kimliği emojisi geri dönüşü (`agents.entries.*.identity.emoji`, aksi takdirde "👀")

    Notlar:

    - Discord, Unicode emojileri veya özel emoji adlarını kabul eder.
    - Bir kanal veya hesap için tepkiyi devre dışı bırakmak üzere `""` kullanın.

    **Kapsam (`messages.ackReactionScope`):**

    Değerler: `"all"` (DM'ler + ortam odası olayları dâhil gruplar), `"direct"` (yalnızca DM'ler), `"group-all"` (ortam odası olayları dışındaki tüm grup mesajları, DM yok), `"group-mentions"` (bottan bahsedildiği gruplar; **DM yok**, varsayılan), `"off"` / `"none"` (devre dışı).

    <Note>
    Varsayılan kapsam (`"group-mentions"`), doğrudan mesajlarda veya ortam odası olaylarında alındı tepkilerini tetiklemez. Gelen Discord DM'lerinde ve sessiz oda olaylarında alındı tepkisi almak için `messages.ackReactionScope` değerini `"all"` olarak ayarlayın.
    </Note>

  </Accordion>

  <Accordion title="Yapılandırma yazma işlemleri">
    Kanal tarafından başlatılan yapılandırma yazma işlemleri varsayılan olarak etkindir. Bu, `/config set|unset` akışlarını etkiler (komut özellikleri etkinleştirildiğinde).

    Devre dışı bırakma:

```json5
{
  channels: {
    discord: {
      configWrites: false,
    },
  },
}
```

  </Accordion>

  <Accordion title="Gateway proxy'si">
    Discord Gateway WebSocket trafiğini ve başlangıç REST aramalarını (uygulama kimliği + izin verilenler listesi çözümlemesi) `channels.discord.proxy` ile bir HTTP(S) proxy'si üzerinden yönlendirin.
    Discord Gateway WebSocket proxy'lemesi açıkça yapılandırılır; WebSocket bağlantıları, Gateway işlemindeki ortam proxy değişkenlerini devralmaz. `channels.discord.proxy` yapılandırıldığında başlangıç REST aramaları bu proxy'yi kullanır.

```json5
{
  channels: {
    discord: {
      proxy: "http://proxy.example:8080",
    },
  },
}
```

    Hesap başına geçersiz kılma:

```json5
{
  channels: {
    discord: {
      accounts: {
        primary: {
          proxy: "http://proxy.example:8080",
        },
      },
    },
  },
}
```

  </Accordion>

  <Accordion title="PluralKit desteği">
    Proxy'lenen mesajları sistem üyesi kimliğiyle eşlemek için PluralKit çözümlemesini etkinleştirin:

```json5
{
  channels: {
    discord: {
      pluralkit: {
        enabled: true,
        token: "pk_live_...", // isteğe bağlı; özel sistemler için gereklidir
      },
    },
  },
}
```

    Notlar:

    - izin verilenler listeleri `pk:<memberId>` kullanabilir
    - üye görünen adları yalnızca `channels.discord.dangerouslyAllowNameMatching: true` olduğunda ad/slug ile eşleştirilir
    - aramalar, özgün mesaj kimliğiyle PluralKit API'sini sorgular
    - arama başarısız olursa proxy'lenen mesajlar bot mesajı olarak değerlendirilir ve `allowBots` bunların geçmesine izin vermediği sürece bırakılır

  </Accordion>

  <Accordion title="Giden bahsetme takma adları">
    Aracıların bilinen Discord kullanıcılarından deterministik biçimde bahsetmesi gerektiğinde `mentionAliases` kullanın. Anahtarlar, baştaki `@` olmadan kullanıcı adlarıdır; değerler ise Discord kullanıcı kimlikleridir. Bilinmeyen kullanıcı adları, `@everyone`, `@here` ve Markdown kod parçaları içindeki bahsetmeler değiştirilmeden bırakılır.

```json5
{
  channels: {
    discord: {
      mentionAliases: {
        SupportLead: "123456789012345678",
      },
      accounts: {
        ops: {
          mentionAliases: {
            OpsLead: "234567890123456789",
          },
        },
      },
    },
  },
}
```

  </Accordion>

  <Accordion title="İletişim durumu yapılandırması">
    Bir durum veya etkinlik alanı ayarladığınızda ya da otomatik iletişim durumunu etkinleştirdiğinizde iletişim durumu güncellemeleri uygulanır.

    Yalnızca durum:

```json5
{
  channels: {
    discord: {
      status: "idle",
    },
  },
}
```

    Etkinlik (`activity` ayarlandığında özel durum varsayılan etkinlik türüdür):

```json5
{
  channels: {
    discord: {
      activity: "Odaklanma zamanı",
      activityType: 4,
    },
  },
}
```

    Yayın:

```json5
{
  channels: {
    discord: {
      activity: "Canlı kodlama",
      activityType: 1,
      activityUrl: "https://twitch.tv/openclaw",
    },
  },
}
```

    Etkinlik türü eşlemesi:

    - 0: Oynuyor
    - 1: Yayın yapıyor (`activityUrl` gerektirir; `activityUrl` ise `activityType: 1` gerektirir)
    - 2: Dinliyor
    - 3: İzliyor
    - 4: Özel (etkinlik metnini durum hâli olarak kullanır; emoji isteğe bağlıdır)
    - 5: Yarışıyor

    Otomatik iletişim durumu (çalışma zamanı sağlık sinyali):

```json5
{
  channels: {
    discord: {
      autoPresence: {
        enabled: true,
        intervalMs: 30000,
        minUpdateIntervalMs: 15000,
        exhaustedText: "token tükendi",
      },
    },
  },
}
```

    Otomatik iletişim durumu, çalışma zamanı kullanılabilirliğini Discord durumuyla eşler: sağlıklı => çevrimiçi, bozulmuş veya bilinmiyor => boşta, tükenmiş veya kullanılamıyor => rahatsız etmeyin. Varsayılanlar: `intervalMs` 30000, `minUpdateIntervalMs` 15000 (`intervalMs` değerinden küçük veya ona eşit olmalıdır). İsteğe bağlı metin geçersiz kılmaları:

    - `autoPresence.healthyText`
    - `autoPresence.degradedText`
    - `autoPresence.exhaustedText` (`{reason}` yer tutucusunu destekler)

  </Accordion>

  <Accordion title="Discord'daki onaylar">
    Discord, DM'lerde düğme tabanlı onay işlemeyi destekler ve isteğe bağlı olarak onay istemlerini kaynak kanala gönderebilir.

    Yapılandırma yolu:

    - `channels.discord.execApprovals.enabled`
    - `channels.discord.execApprovals.approvers` (isteğe bağlı; mümkün olduğunda `commands.ownerAllowFrom` değerine geri döner)
    - `channels.discord.execApprovals.target` (`dm` | `channel` | `both`, varsayılan: `dm`)
    - `agentFilter`, `sessionFilter`, `cleanupAfterResolve`

    `enabled` ayarlanmamış veya `"auto"` olduğunda ve `execApprovals.approvers` ya da `commands.ownerAllowFrom` üzerinden en az bir onaylayan çözümlenebildiğinde Discord, yerel yürütme onaylarını otomatik olarak etkinleştirir. Discord; yürütme onaylayanlarını kanal `allowFrom`, eski `dm.allowFrom` veya doğrudan mesaj `defaultTo` değerlerinden çıkarmaz. Discord'u yerel onay istemcisi olarak açıkça devre dışı bırakmak için `enabled: false` ayarlayın.

    `/diagnostics` ve `/export-trajectory` gibi hassas, yalnızca sahiplerin kullanabildiği grup komutlarında OpenClaw, onay istemlerini ve nihai sonuçları özel olarak gönderir. Komutu çağıran sahibin bir Discord sahip rotası varsa önce Discord DM'ini dener; aksi takdirde Telegram gibi `commands.ownerAllowFrom` içindeki ilk kullanılabilir sahip rotasına geri döner.

    `target`, `channel` veya `both` olduğunda onay istemi kanalda görünür. Düğmeleri yalnızca çözümlenen onaylayanlar kullanabilir; diğer kullanıcılar geçici bir ret bildirimi alır. Onay istemleri komut metnini içerir; bu nedenle kanal teslimini yalnızca güvenilir kanallarda etkinleştirin. Kanal kimliği oturum anahtarından türetilemiyorsa OpenClaw, DM teslimine geri döner.

    Discord, diğer sohbet kanallarının kullandığı ortak onay düğmelerini oluşturur; yerel Discord bağdaştırıcısı temel olarak onaylayan DM yönlendirmesi ve kanal dağıtımı ekler. Bu düğmeler mevcut olduğunda birincil onay kullanıcı deneyimini oluştururlar; OpenClaw yalnızca araç sonucu sohbet onaylarının kullanılamadığını veya tek yolun manuel onay olduğunu belirttiğinde manuel bir `/approve` komutu içermelidir. Discord yerel onay çalışma zamanı etkin değilse OpenClaw, yerel deterministik `/approve <id> <decision>` istemini görünür tutar. Çalışma zamanı etkin olduğu hâlde yerel kart hiçbir hedefe teslim edilemiyorsa OpenClaw, bekleyen onaydaki tam `/approve` komutunu içeren aynı sohbet içi bir geri dönüş bildirimi gönderir.

    Gateway kimlik doğrulaması ve onay çözümlemesi, ortak Gateway istemci sözleşmesini izler (`plugin:` kimlikleri `plugin.approval.resolve` üzerinden; diğer kimlikler `exec.approval.resolve` üzerinden çözümlenir). Onayların süresi varsayılan olarak 30 dakika sonra dolar.

    Bkz. [Yürütme onayları](/tr/tools/exec-approvals).

  </Accordion>
</AccordionGroup>

## Araçlar ve eylem geçitleri

Discord mesaj eylemleri; mesajlaşma, kanal yönetimi, moderasyon, iletişim durumu ve meta verileri kapsar.

Temel örnekler:

- mesajlaşma: `sendMessage`, `readMessages`, `editMessage`, `deleteMessage`, `threadReply`
- tepkiler: `react`, `reactions`, `emojiList`
- moderasyon: `timeout`, `kick`, `ban`
- iletişim durumu: `setPresence`

`event-create` eylemi, planlanmış olayın kapak görselini ayarlamak için isteğe bağlı bir `image` parametresi (URL veya yerel dosya yolu) kabul eder.

Eylem geçitleri `channels.discord.actions.*` altında bulunur.

Varsayılan geçit davranışı:

| Eylem grubu                                                                                                                                                              | Varsayılan |
| ------------------------------------------------------------------------------------------------------------------------------------------------------------------------ | ---------- |
| tepkiler, mesajlar, ileti dizileri, sabitlenenler, anketler, arama, üye bilgileri, rol bilgileri, kanal bilgileri, kanallar, ses durumu, etkinlikler, çıkartmalar, emoji yüklemeleri, çıkartma yüklemeleri, izinler | etkin      |
| roller                                                                                                                                                                   | devre dışı |
| moderasyon                                                                                                                                                               | devre dışı |
| mevcudiyet                                                                                                                                                               | devre dışı |

## Bileşenler v2 kullanıcı arayüzü

OpenClaw, yürütme onayları ve bağlamlar arası işaretçiler için Discord bileşenleri v2'yi kullanır. Discord mesaj eylemleri, özel kullanıcı arayüzü için `components` değerini de kabul edebilir (ileri düzey; discord aracı aracılığıyla bir bileşen yükü oluşturulmasını gerektirir); eski `embeds` ise kullanılmaya devam edilebilir ancak önerilmez.

- `channels.discord.ui.components.accentColor`, Discord bileşen kapsayıcılarının kullandığı vurgu rengini (onaltılık) ayarlar. Hesap başına: `channels.discord.accounts.<id>.ui.components.accentColor`.
- `channels.discord.agentComponents.ttlMs`, gönderilen Discord bileşeni geri çağırmalarının ne kadar süreyle kayıtlı kalacağını denetler (varsayılan `1800000`, en fazla `86400000`). Hesap başına: `channels.discord.accounts.<id>.agentComponents.ttlMs`.
- `embeds`, bileşenler v2 mevcut olduğunda yok sayılır.
- Düz URL önizlemeleri varsayılan olarak engellenir. Tek bir giden bağlantının genişletilmesi gerektiğinde mesaj eyleminde `suppressEmbeds: false` değerini ayarlayın.

Örnek:

```json5
{
  channels: {
    discord: {
      ui: {
        components: {
          accentColor: "#5865F2",
        },
      },
    },
  },
}
```

## Ses

Discord'un iki farklı ses yüzeyi vardır: gerçek zamanlı **ses kanalları** (sürekli görüşmeler) ve **sesli mesaj ekleri** (dalga biçimi önizleme formatı). Gateway her ikisini de destekler.

### Ses kanalları

Kurulum denetim listesi:

1. Discord Developer Portal'da Message Content Intent seçeneğini etkinleştirin.
2. Rol/kullanıcı izin listeleri kullanıldığında Server Members Intent seçeneğini etkinleştirin.
3. Botu `bot` ve `applications.commands` kapsamlarıyla davet edin.
4. Hedef ses kanalında Connect, Speak, Send Messages ve Read Message History izinlerini verin.
5. Yerel komutları (`commands.native` veya `channels.discord.commands.native`) etkinleştirin.
6. `channels.discord.voice` yapılandırmasını yapın.

Oturumları denetlemek için `/vc join|leave|status` kullanın. Komut, hesabın varsayılan aracısını kullanır ve diğer Discord komutlarıyla aynı izin listesi ve grup politikası kurallarına uyar.

```bash
/vc join channel:<voice-channel-id>
/vc status
/vc leave
```

Katılmadan önce botun geçerli izinlerini incelemek için:

```bash
openclaw channels capabilities --channel discord --target channel:<voice-channel-id>
```

Otomatik katılım örneği:

```json5
{
  channels: {
    discord: {
      voice: {
        enabled: true,
        model: "openai/gpt-5.6-sol",
        autoJoin: [
          {
            guildId: "123456789012345678",
            channelId: "234567890123456789",
          },
        ],
        allowedChannels: [
          {
            guildId: "123456789012345678",
            channelId: "234567890123456789",
          },
        ],
        daveEncryption: true,
        decryptionFailureTolerance: 24,
        connectTimeoutMs: 30000,
        reconnectGraceMs: 15000,
        realtime: {
          provider: "openai",
          model: "gpt-realtime-2.1",
          speakerVoice: "cedar",
        },
      },
    },
  },
}
```

Notlar:

- Discord ses özelliği, yalnızca metin yapılandırmaları için isteğe bağlıdır; `/vc` komutlarını, ses çalışma zamanını ve `GuildVoiceStates` gateway intent'ini etkinleştirmek için `channels.discord.voice.enabled=true` ayarını yapın (veya mevcut bir `channels.discord.voice` bloğunu koruyun). `channels.discord.intents.voiceStates`, intent aboneliğini açıkça geçersiz kılabilir; etkin ses ayarını izlemesi için ayarlamadan bırakın.
- `voice.mode`, konuşma yolunu denetler. Varsayılan değer `agent-proxy` şeklindedir: gerçek zamanlı bir ses ön ucu söz alma zamanlamasını, kesintiyi ve oynatmayı yönetir; esas işleri `openclaw_agent_consult` aracılığıyla yönlendirilmiş OpenClaw aracısına devreder ve sonucu, o konuşmacıdan yazılı bir Discord istemi gibi değerlendirir. `stt-tts`, eski toplu STT ve TTS akışını korur. `bidi`, OpenClaw beyni için `openclaw_agent_consult` özelliğini sunarken gerçek zamanlı modelin doğrudan konuşmasına olanak tanır.
- `voice.agentSession`, sesli konuşma sıralarını hangi OpenClaw konuşmasının alacağını denetler. Ses kanalının kendi oturumu için ayarlamadan bırakın veya ses kanalını `#maintainers` gibi mevcut bir Discord metin kanalı oturumunun mikrofon/hoparlör uzantısı hâline getirmek için `{ mode: "target", target: "channel:<text-channel-id>" }` olarak ayarlayın.
- `voice.model`, Discord sesli yanıtları ve gerçek zamanlı danışmalar için OpenClaw aracı beynini geçersiz kılar. Yönlendirilmiş aracı modelini devralması için ayarlamadan bırakın. Bu ayar `voice.realtime.model` ayarından ayrıdır.
- `voice.followUsers`, botun seçili kullanıcılarla birlikte Discord ses kanallarına katılmasına, kanallar arasında geçiş yapmasına ve kanallardan ayrılmasına olanak tanır. Bkz. [Ses kanalında kullanıcıları takip etme](#follow-users-in-voice).
- `agent-proxy`, konuşmayı `discord-voice` üzerinden yönlendirir; bu, konuşmacı ve hedef oturum için normal sahip/araç yetkilendirmesini korur ancak oynatma Discord ses özelliğine ait olduğundan aracının `tts` aracını gizler. Varsayılan olarak `agent-proxy`, sahip konuşmacılar (`voice.realtime.toolPolicy: "owner"`) için danışmaya sahip eşdeğeri tam araç erişimi verir ve esas yanıtları vermeden önce OpenClaw aracısına danışılmasını özellikle tercih eder (`voice.realtime.consultPolicy: "always"`). Bu varsayılan `always` modunda gerçek zamanlı katman, danışma yanıtından önce dolgu ifadelerini otomatik olarak seslendirmez; konuşmayı yakalayıp metne dönüştürür ve ardından yönlendirilmiş OpenClaw yanıtını seslendirir. Discord ilk yanıtı oynatmaya devam ederken birden fazla zorunlu danışma yanıtı tamamlanırsa sonraki tam konuşma yanıtları, cümlenin ortasında konuşmanın yerini almak yerine oynatma boşta kalana kadar kuyruğa alınır.
- `stt-tts` modunda STT, `tools.media.audio` kullanır; `voice.model`, transkripsiyonu etkilemez.
- Gerçek zamanlı modlarda `voice.realtime.provider`, `voice.realtime.model` ve `voice.realtime.speakerVoice`, gerçek zamanlı ses oturumunu yapılandırır. OpenAI Realtime 2.1 ve Codex beyni için `voice.realtime.model: "gpt-realtime-2.1"` ve `voice.model: "openai/gpt-5.6-sol"` kullanın.
- Gerçek zamanlı ses modları, hızlı doğrudan konuşma sıralarının yönlendirilmiş OpenClaw aracısıyla aynı kimliği, kullanıcı temellendirmesini ve kişiliği koruması için varsayılan olarak küçük `IDENTITY.md`, `USER.md` ve `SOUL.md` profil dosyalarını gerçek zamanlı sağlayıcı talimatlarına dahil eder. Bunu özelleştirmek için `voice.realtime.bootstrapContextFiles` değerini bir alt küme olarak, devre dışı bırakmak içinse `[]` olarak ayarlayın. Yalnızca bu profil dosyaları desteklenir; `AGENTS.md` normal aracı bağlamında kalır. Eklenen profil bağlamı; çalışma alanı çalışmaları, güncel bilgiler, bellek araması veya araç destekli eylemler için `openclaw_agent_consult` yerine geçmez.
- OpenAI `agent-proxy` gerçek zamanlı modunda uyandırma adı denetimi varsayılan olarak odaya uyum sağlar: tek bir kişi uyandırma adı kullanmadan doğal biçimde konuşabilirken iki veya daha fazla kişinin konuşma sırasına bir uyandırma adıyla başlaması veya sırayı bu adla bitirmesi gerekir. Diğer botlar kişi sayılmaz. Her zaman uyandırma adı gerektirmek için `voice.realtime.requireWakeName: true`, hiçbir zaman gerektirmemek için `false` olarak ayarlayın. Yapılandırılan uyandırma adları bir veya iki sözcükten oluşmalıdır. `voice.realtime.wakeNames` ayarlanmamışsa OpenClaw, yönlendirilmiş aracının `name` değerini ve `OpenClaw` değerini kullanır; bunlar yoksa aracı kimliği ile `OpenClaw` değerini kullanır. Etkin bir uyandırma adı denetimi, gerçek zamanlı sağlayıcının otomatik yanıtını devre dışı bırakır, kabul edilen konuşma sıralarını OpenClaw aracı danışma yolu üzerinden yönlendirir ve son transkript ulaşmadan önce kısmi transkripsiyonda baştaki bir uyandırma adı algılandığında kısa bir sesli onay verir. Politika, ses bağlantısını yeniden kurmadan canlı katılma ve ayrılmaları izler.
- OpenAI gerçek zamanlı sağlayıcısı, çıkış sesi ve transkript olayları için güncel Realtime 2 olay adlarını ve eski Codex uyumlu diğer adları kabul eder; böylece uyumlu sağlayıcı anlık görüntüleri, asistan sesini kaybetmeden farklılaşabilir.
- `voice.realtime.bargeIn`, Discord konuşmacı başlangıcı olaylarının etkin gerçek zamanlı oynatmayı kesip kesmeyeceğini denetler. Ayarlanmamışsa gerçek zamanlı sağlayıcının giriş sesi kesintisi ayarını izler.
- `voice.realtime.minBargeInAudioEndMs`, OpenAI gerçek zamanlı araya girme işleminin sesi kesmesinden önce gereken minimum asistan oynatma süresini denetler. Varsayılan: `250`. Yankının az olduğu odalarda anında kesinti için `0` olarak ayarlayın; hoparlör kurulumunda yoğun yankı varsa değeri yükseltin.
- `voice.tts`, yalnızca `stt-tts` ses oynatımı için `tts` ayarını geçersiz kılar; gerçek zamanlı modlar bunun yerine `voice.realtime.speakerVoice` kullanır. Discord oynatımında bir OpenAI sesi kullanmak için `voice.tts.provider: "openai"` ayarını yapın ve `voice.tts.providers.openai.speakerVoice` altında bir Metinden sese ses seçin. `cedar`, güncel OpenAI TTS modelinde erkeksi tınıya sahip iyi bir seçenektir.
- Kanal bazındaki Discord `systemPrompt` geçersiz kılmaları, ilgili ses kanalının ses transkripti konuşma sıralarına uygulanır.
- OpenClaw bir ses kanalına katıldığında yönlendirilmiş aracı oturumu, güncel katılımcı listesini içeren sessiz bir sistem olayı alır. Katılımcıların daha sonra katılması ve ayrılması, istenmeyen bir sesli yanıtı tetiklemeden bu oturumu günceller; Discord görünen adları güvenilmeyen etiketler olarak değerlendirilir. Yetkilendirilmiş sesli konuşma sıraları da güncel bir katılımcı listesi anlık görüntüsü alır.
- Ses transkripti konuşma sıraları ve `/vc` komutları, sahip durumu için `commands.ownerAllowFrom` içindeki Discord girdilerini kullanır. Hiçbir Discord komut sahibi yapılandırılmadığında seçili Discord hesabının `allowFrom` değeri (veya eski `dm.allowFrom`) sahip durumu vermeden ses erişimini yetkilendirmeye devam edebilir. Aracı araçlarının görünürlüğü, yönlendirilmiş oturum için yapılandırılan araç politikasını izler.
- `voice.autoJoin` aynı sunucu için birden fazla girdi içeriyorsa OpenClaw, o sunucu için son yapılandırılan kanala katılır.
- `voice.allowedChannels`, isteğe bağlı bir bulunma izin listesidir. `/vc join` komutunun yetkilendirilmiş herhangi bir Discord ses kanalına katılmasına izin vermek için ayarlamadan bırakın. Ayarlandığında `/vc join`, başlangıçta otomatik katılma ve bot ses durumu geçişleri, listelenen `{ guildId, channelId }` girdileriyle sınırlandırılır. Tüm Discord ses kanalı katılımlarını reddetmek için boş bir dizi olarak ayarlayın. Discord botu izin listesinin dışına taşırsa OpenClaw o kanaldan ayrılır ve varsa yapılandırılmış otomatik katılma hedefine yeniden katılır.
- `voice.daveEncryption` ve `voice.decryptionFailureTolerance`, `@discordjs/voice` katılma seçeneklerine doğrudan aktarılır; üst kaynak varsayılanları `daveEncryption=true` ve `decryptionFailureTolerance=24` değerleridir.
- OpenClaw, Discord ses alımı ve gerçek zamanlı ham PCM oynatımı için paketle gelen `libopus-wasm` codec'ini kullanır. Sabitlenmiş bir libopus WebAssembly derlemesiyle gelir ve yerel opus eklentileri gerektirmez.
- `voice.connectTimeoutMs`, `/vc join` ve otomatik katılma denemeleri için ilk `@discordjs/voice` Ready bekleme süresini denetler. Varsayılan: `30000`.
- `voice.reconnectGraceMs`, bağlantısı kesilmiş bir ses oturumunu yok etmeden önce OpenClaw'ın yeniden bağlanmaya başlaması için ne kadar bekleyeceğini denetler. Varsayılan: `15000`.
- `stt-tts` modunda ses oynatımı, yalnızca başka bir kullanıcı konuşmaya başladığı için durmaz. Geri besleme döngülerini önlemek üzere OpenClaw, TTS oynatılırken yeni ses yakalamayı yok sayar; sonraki konuşma sırası için oynatma bittikten sonra konuşun. Gerçek zamanlı modlar, konuşmacı başlangıçlarını araya girme sinyalleri olarak gerçek zamanlı sağlayıcıya iletir.
- Gerçek zamanlı modlarda hoparlörlerden açık mikrofona giren yankı, araya girme olarak algılanıp oynatmayı kesebilir. Yankının yoğun olduğu Discord odalarında OpenAI'ın giriş sesi nedeniyle otomatik kesinti yapmasını önlemek için `voice.realtime.providers.openai.interruptResponseOnInputAudio: false` ayarını yapın. Discord konuşmacı başlangıcı olaylarının etkin oynatmayı yine de kesmesini istiyorsanız `voice.realtime.bargeIn: true` ekleyin. OpenAI gerçek zamanlı köprüsü, `voice.realtime.minBargeInAudioEndMs` değerinden kısa oynatma kesmelerini olası yankı/gürültü olarak yok sayar ve Discord oynatımını temizlemek yerine bunları atlandı olarak günlüğe kaydeder.
- `voice.captureSilenceGraceMs`, Discord bir konuşmacının durduğunu bildirdikten sonra OpenClaw'ın bu ses parçasını STT için sonlandırmadan önce ne kadar bekleyeceğini denetler. Varsayılan: `2000`; Discord normal duraklamaları kesintili kısmi transkriptlere bölüyorsa değeri yükseltin.
- Seçili TTS sağlayıcısı ElevenLabs olduğunda Discord ses oynatımı akışlı TTS kullanır ve sağlayıcının yanıt akışından başlar. Akış desteği olmayan sağlayıcılar, sentezlenmiş geçici dosya yoluna geri döner.
- OpenClaw, alım şifresi çözme hatalarını izler ve kısa bir zaman aralığında tekrarlanan hatalardan sonra ses kanalından ayrılıp yeniden katılarak otomatik olarak kurtarır.
- Güncellemeden sonra alım günlüklerinde tekrar tekrar `DecryptionFailed(UnencryptedWhenPassthroughDisabled)` görülüyorsa bir bağımlılık raporu ve günlükleri toplayın. Paketle gelen `@discordjs/voice` satırı, discord.js PR #11449'daki üst kaynak dolgu düzeltmesini içerir; bu düzeltme discord.js issue #11419'u kapatmıştır.
- `The operation was aborted` alım olayları, OpenClaw yakalanmış bir konuşmacı parçasını sonlandırdığında beklenir; bunlar ayrıntılı tanılama iletileridir, uyarı değildir.
- Ayrıntılı Discord ses günlükleri, kabul edilen her konuşmacı parçası için sınırlandırılmış tek satırlık bir STT transkript önizlemesi içerir; böylece hata ayıklama sırasında sınırsız transkript metni dökülmeden hem kullanıcı tarafı hem de aracı yanıtı tarafı gösterilir.
- `agent-proxy` modunda zorunlu danışma geri dönüşü; `...` ile biten metinler, "and" gibi sondaki bağlaçlar veya "be right back" ya da "bye" gibi açıkça eylem gerektirmeyen kapanışlar gibi tamamlanmamış olması muhtemel transkript parçalarını atlar. Bu işlem eski bir kuyruklanmış yanıtı önlediğinde günlüklerde `forced agent consult skipped reason=...` gösterilir.

### Ses kanalında kullanıcıları takip etme

Discord ses botunun başlangıçta sabit bir kanala katılması veya `/vc join` komutunu beklemesi yerine bilinen bir ya da daha fazla Discord kullanıcısıyla birlikte kalmasını istediğinizde `voice.followUsers` kullanın.

```json5
{
  channels: {
    discord: {
      voice: {
        enabled: true,
        followUsersEnabled: true,
        followUsers: ["discord:123456789012345678"],
        allowedChannels: [
          {
            guildId: "123456789012345678",
            channelId: "234567890123456789",
          },
        ],
      },
    },
  },
}
```

Davranış:

- `followUsers`, ham Discord kullanıcı kimliklerini ve `discord:<id>` değerlerini kabul eder. OpenClaw, ses durumu olaylarını eşleştirmeden önce her iki biçimi de normalleştirir.
- `followUsersEnabled`, `followUsers` yapılandırıldığında varsayılan olarak `true` değerini alır. Kaydedilen listeyi koruyup otomatik ses takibini durdurmak için bunu `false` olarak ayarlayın.
- `followUsers` yalnızca ses kanalında bulunmayı denetler. Konuşmacı erişimi veya sahip yetkisi vermez; `commands.ownerAllowFrom` ile sunucu ya da kanal kullanıcılarını ve rollerini ayrı ayrı yapılandırın.
- Takip edilen bir kullanıcı izin verilen bir ses kanalına katıldığında OpenClaw da bu kanala katılır. Kullanıcı başka bir kanala geçtiğinde OpenClaw da onunla birlikte geçer. Etkin olarak takip edilen kullanıcının bağlantısı kesildiğinde OpenClaw kanaldan ayrılır.
- Aynı sunucuda birden fazla takip edilen kullanıcı varsa ve etkin olarak takip edilen kullanıcı ayrılırsa OpenClaw, sunucudan ayrılmadan önce izlenen başka bir takip edilen kullanıcının kanalına geçer. Birkaç takip edilen kullanıcı aynı anda hareket ederse en son gözlemlenen ses durumu olayı geçerli olur.
- `allowedChannels` uygulanmaya devam eder. İzin verilmeyen bir kanaldaki takip edilen kullanıcı yok sayılır ve takip sahipliğindeki oturum başka bir takip edilen kullanıcıya geçer veya kanaldan ayrılır.
- OpenClaw, başlangıçta ve sınırlandırılmış aralıklarla kaçırılan ses durumu olaylarını uzlaştırır. Uzlaştırma, yapılandırılmış sunuculardan örnekleme yapar ve her çalıştırmadaki REST sorgularını sınırlar; bu nedenle çok büyük `followUsers` listelerinin yakınsaması birden fazla aralık sürebilir.
- Discord veya bir yönetici, bot bir kullanıcıyı takip ederken botu taşırsa OpenClaw ses oturumunu yeniden oluşturur ve hedefe izin veriliyorsa takip sahipliğini korur. Bot `allowedChannels` dışına taşınırsa OpenClaw ayrılır ve varsa yapılandırılmış hedefe yeniden katılır.
- DAVE alma kurtarması, tekrarlanan şifre çözme hatalarından sonra aynı kanaldan ayrılıp yeniden katılabilir. Takip sahipliğindeki oturumlar bu kurtarma yolu boyunca takip sahipliğini korur; böylece takip edilen kullanıcının daha sonra bağlantısının kesilmesi hâlinde kanaldan yine ayrılır.

Katılma modları arasından seçim yapın:

- Botun siz ses kanalındayken otomatik olarak orada bulunması gereken kişisel veya operatör kurulumlarında `followUsers` kullanın.
- İzlenen hiçbir kullanıcı ses kanalında olmasa bile bulunması gereken sabit oda botları için `autoJoin` kullanın.
- Tek seferlik katılımlar veya otomatik ses mevcudiyetinin beklenmedik olacağı odalar için `/vc join` kullanın.

Discord ses codec'i:

- Ses alma günlükleri `discord voice: opus decoder: libopus-wasm` gösterir.
- Gerçek zamanlı oynatma, paketleri `@discordjs/voice` bileşenine aktarmadan önce ham 48 kHz stereo PCM verisini aynı paketlenmiş `libopus-wasm` paketiyle Opus biçimine kodlar.
- Dosya ve sağlayıcı akışı oynatma, ffmpeg ile ham 48 kHz stereo PCM biçimine dönüştürür; ardından Discord'a gönderilen Opus paket akışı için `libopus-wasm` kullanır.

STT ve TTS işlem hattı:

- Discord PCM yakalaması geçici bir WAV dosyasına dönüştürülür.
- `tools.media.audio`, STT işlemini gerçekleştirir; örneğin `openai/gpt-4o-mini-transcribe`.
- Transkript, Discord girişi ve yönlendirmesi üzerinden gönderilirken yanıt LLM'si, ajan `tts` aracını gizleyen ve döndürülen metni isteyen bir ses çıkışı politikasıyla çalışır; çünkü nihai TTS oynatmasının sahibi Discord sestir.
- `voice.model`, ayarlandığında yalnızca bu ses kanalı turunun yanıt LLM'sini geçersiz kılar.
- `voice.tts`, `tts` üzerine birleştirilir; akış özellikli sağlayıcılar oynatıcıyı doğrudan besler, aksi hâlde ortaya çıkan ses dosyası katılınan kanalda oynatılır.

Varsayılan ajan vekili ses kanalı oturumu örneği:

```json5
{
  channels: {
    discord: {
      voice: {
        enabled: true,
        model: "openai/gpt-5.6-sol",
        followUsersEnabled: true,
        followUsers: ["123456789012345678"],
        realtime: {
          provider: "openai",
          model: "gpt-realtime-2.1",
          speakerVoice: "cedar",
        },
      },
    },
  },
}
```

`voice.agentSession` bloğu olmadığında her ses kanalı kendi yönlendirilmiş OpenClaw oturumunu alır. Örneğin `/vc join channel:234567890123456789`, ilgili Discord ses kanalının oturumuyla konuşur. Gerçek zamanlı model yalnızca ses ön ucudur; esas istekler yapılandırılmış OpenClaw ajanına devredilir. Gerçek zamanlı model danışma aracını çağırmadan nihai bir transkript üretirse OpenClaw, varsayılan davranışın yine ajanla konuşuyormuş gibi çalışması için geri dönüş olarak danışmayı zorunlu kılar.

Eski STT ve TTS örneği:

```json5
{
  channels: {
    discord: {
      voice: {
        enabled: true,
        mode: "stt-tts",
        model: "openai/gpt-5.4-mini",
        tts: {
          provider: "openai",
          providers: {
            openai: {
              model: "gpt-4o-mini-tts",
              speakerVoice: "cedar",
            },
          },
        },
      },
    },
  },
}
```

Gerçek zamanlı çift yönlü örnek:

```json5
{
  channels: {
    discord: {
      voice: {
        enabled: true,
        mode: "bidi",
        model: "openai/gpt-5.6-sol",
        realtime: {
          provider: "openai",
          model: "gpt-realtime-2.1",
          speakerVoice: "cedar",
          toolPolicy: "safe-read-only",
          consultPolicy: "always",
        },
      },
    },
  },
}
```

Mevcut bir Discord kanal oturumunun uzantısı olarak ses:

```json5
{
  channels: {
    discord: {
      voice: {
        enabled: true,
        mode: "agent-proxy",
        model: "openai/gpt-5.6-sol",
        agentSession: {
          mode: "target",
          target: "channel:123456789012345678",
        },
        realtime: {
          provider: "openai",
          model: "gpt-realtime-2.1",
          speakerVoice: "cedar",
        },
      },
    },
  },
}
```

`agent-proxy` modunda bot yapılandırılmış ses kanalına katılır, ancak OpenClaw ajan turları hedef kanalın normal yönlendirilmiş oturumunu ve ajanını kullanır. Gerçek zamanlı ses oturumu, döndürülen sonucu ses kanalında seslendirir. Gözetmen ajan, doğru eylem buysa ayrı bir Discord mesajı göndermek de dâhil olmak üzere, araç politikasına göre normal mesaj araçlarını kullanmaya devam edebilir.

Yetkilendirilmiş bir OpenClaw çalıştırması etkinken yeni Discord ses transkriptleri, başka bir ajan turu başlatılmadan önce canlı çalıştırma denetimi olarak değerlendirilir. "durum", "bunu iptal et", "daha küçük düzeltmeyi kullan" veya "işin bittiğinde testleri de kontrol et" gibi ifadeler, etkin oturum için durum, iptal, yönlendirme veya takip girdisi olarak sınıflandırılır. Durum, iptal, kabul edilen yönlendirme ve takip sonuçları ses kanalında seslendirilir; böylece arayan kişi OpenClaw'un isteği işleyip işlemediğini bilir.

Kullanışlı hedef biçimleri:

- `target: "channel:123456789012345678"`, bir Discord metin kanalı oturumu üzerinden yönlendirir.
- `target: "123456789012345678"`, bir kanal hedefi olarak değerlendirilir.
- `target: "dm:123456789012345678"` veya `target: "user:123456789012345678"`, ilgili doğrudan mesaj oturumu üzerinden yönlendirir.

Yankının yoğun olduğu OpenAI Realtime örneği:

```json5
{
  channels: {
    discord: {
      voice: {
        enabled: true,
        mode: "bidi",
        model: "openai/gpt-5.6-sol",
        realtime: {
          provider: "openai",
          model: "gpt-realtime-2.1",
          speakerVoice: "cedar",
          bargeIn: true,
          minBargeInAudioEndMs: 500,
          consultPolicy: "always",
          providers: {
            openai: {
              interruptResponseOnInputAudio: false,
            },
          },
        },
      },
    },
  },
}
```

Model açık bir mikrofon üzerinden kendi Discord oynatmasını duyduğunda, ancak konuşarak yine de sözünü kesmek istediğinizde bunu kullanın. OpenClaw, OpenAI'ın ham giriş sesi nedeniyle otomatik olarak kesintiye uğramasını önlerken `bargeIn: true`, Discord konuşmacı başlangıcı olaylarının ve hâlihazırda etkin olan konuşmacı sesinin, yakalanan bir sonraki tur OpenAI'a ulaşmadan önce etkin gerçek zamanlı yanıtları iptal etmesini sağlar. `audioEndMs` değeri `minBargeInAudioEndMs` altında olan çok erken araya girme sinyalleri muhtemel yankı/gürültü olarak değerlendirilip yok sayılır; böylece model ilk oynatma karesinde kesilmez.

Beklenen ses günlükleri:

- Katılma sırasında: `discord voice: joining ... voiceSession=... supervisorSession=... agentSessionMode=... voiceModel=... realtimeModel=...`
- Gerçek zamanlı başlatma sırasında: `discord voice: realtime bridge starting ... autoRespond=false interruptResponse=false bargeIn=false minBargeInAudioEndMs=...`
- Konuşmacı sesi sırasında: `discord voice: realtime speaker turn opened ...`, `discord voice: realtime input audio started ... outputAudioMs=... outputActive=...` ve `discord voice: realtime speaker turn closed ... chunks=... discordBytes=... realtimeBytes=... interruptedPlayback=...`
- Eski konuşma atlandığında: `discord voice: realtime forced agent consult skipped reason=incomplete-transcript ...` veya `reason=non-actionable-closing ...`
- Gerçek zamanlı yanıt tamamlandığında: `discord voice: realtime audio playback finishing reason=response.done ... audioMs=... chunks=...`
- Oynatma durdurulduğunda/sıfırlandığında: `discord voice: realtime audio playback stopped reason=... audioMs=... elapsedMs=... chunks=...`
- Gerçek zamanlı danışma sırasında: `discord voice: realtime consult requested ... voiceSession=... supervisorSession=... question=...`
- Ajan yanıtında: `discord voice: agent turn answer ...`
- Kesin konuşma kuyruğa alındığında: `discord voice: realtime exact speech queued ... queued=... outputAudioMs=... outputActive=...`, ardından `discord voice: realtime exact speech dequeued reason=player-idle ...`
- Araya girme algılandığında: `discord voice: realtime barge-in detected source=speaker-start ...` veya `discord voice: realtime barge-in detected source=active-speaker-audio ...`, ardından `discord voice: realtime barge-in requested reason=... outputAudioMs=... outputActive=...`
- Gerçek zamanlı kesinti sırasında: `discord voice: realtime model interrupt requested client:response.cancel reason=barge-in`, ardından `discord voice: realtime model audio truncated client:conversation.item.truncate reason=barge-in audioEndMs=...` ya da `discord voice: realtime model interrupt confirmed server:response.done status=cancelled ...`
- Yankı/gürültü yok sayıldığında: `discord voice: realtime model interrupt ignored client:conversation.item.truncate.skipped reason=barge-in audioEndMs=0 minAudioEndMs=250`
- Araya girme devre dışı bırakıldığında: `discord voice: realtime capture ignored during playback (barge-in disabled) ...`
- Boşta oynatma sırasında: `discord voice: realtime barge-in ignored reason=... outputActive=false ... playbackChunks=0`

Kesilen ses sorununu ayıklamak için gerçek zamanlı ses günlüklerini bir zaman çizelgesi olarak okuyun:

1. `realtime audio playback started`, Discord'un asistan sesini oynatmaya başladığı anlamına gelir. Köprü bu noktadan itibaren asistan çıkış parçalarını, Discord PCM baytlarını, sağlayıcı gerçek zamanlı baytlarını ve sentezlenen ses süresini saymaya başlar.
2. `realtime speaker turn opened`, bir Discord konuşmacısının etkinleştiğini belirtir. Oynatma zaten etkinse ve `bargeIn` etkinleştirilmişse bunu `barge-in detected source=speaker-start` izleyebilir.
3. `realtime input audio started`, ilgili konuşmacı turu için alınan ilk gerçek ses karesini belirtir. Buradaki `outputActive=true` veya sıfır olmayan `outputAudioMs`, asistan oynatması hâlâ etkinken mikrofonun giriş gönderdiği anlamına gelir.
4. `barge-in detected source=active-speaker-audio`, OpenClaw'un asistan oynatması etkinken canlı konuşmacı sesi gördüğü anlamına gelir. Bu, gerçek bir kesintiyi yararlı ses içermeyen bir Discord konuşmacı başlangıcı olayından ayırt etmek için kullanışlıdır.
5. `barge-in requested reason=...`, OpenClaw'un gerçek zamanlı sağlayıcıdan etkin yanıtı iptal etmesini veya kısaltmasını istediği anlamına gelir. Kesintiden önce asistan sesinin gerçekte ne kadarının oynatıldığını görebilmeniz için `outputAudioMs`, `outputActive` ve `playbackChunks` içerir.
6. `realtime audio playback stopped reason=...`, yerel Discord oynatma sıfırlama noktasıdır. Neden, oynatmayı kimin durdurduğunu belirtir: `barge-in`, `player-idle`, `provider-clear-audio`, `forced-agent-consult`, `stream-close` veya `session-close`.
7. `realtime speaker turn closed`, yakalanan giriş turunu özetler. `chunks=0` veya `hasAudio=false`, konuşmacı turunun açıldığı ancak gerçek zamanlı köprüye kullanılabilir ses ulaşmadığı anlamına gelir. `interruptedPlayback=true`, ilgili giriş turunun asistan çıkışıyla çakıştığı ve araya girme mantığını tetiklediği anlamına gelir.

Kullanışlı alanlar:

- `outputAudioMs`: günlük satırından önce gerçek zamanlı sağlayıcı tarafından oluşturulan asistan sesi süresi.
- `audioMs`: oynatma durmadan önce OpenClaw'un saydığı asistan sesi süresi.
- `elapsedMs`: oynatma akışının veya konuşmacı turunun açılmasıyla kapanması arasındaki gerçek zaman.
- `discordBytes`: Discord sese gönderilen veya Discord sesten alınan 48 kHz stereo PCM baytları.
- `realtimeBytes`: gerçek zamanlı sağlayıcıya gönderilen veya sağlayıcıdan alınan, sağlayıcı biçimindeki PCM baytları.
- `playbackChunks`: etkin yanıt için Discord'a iletilen asistan sesi parçaları.
- `sinceLastAudioMs`: yakalanan son konuşmacı ses karesi ile konuşmacı turunun kapanması arasındaki boşluk.

Yaygın kalıplar:

- `source=active-speaker-audio` ile anında kesilme, küçük `outputAudioMs` ve aynı kullanıcının yakında olması, genellikle hoparlör yankısının mikrofona girdiğini gösterir. `voice.realtime.minBargeInAudioEndMs` değerini artırın, hoparlör sesini azaltın, kulaklık kullanın veya `voice.realtime.providers.openai.interruptResponseOnInputAudio: false` ayarını yapın.
- `source=speaker-start` ardından `speaker turn closed ... hasAudio=false` gelmesi, Discord'un bir konuşmacının konuşmaya başladığını bildirdiği ancak OpenClaw'a ses ulaşmadığı anlamına gelir. Bunun nedeni geçici bir Discord ses olayı, gürültü kapısı davranışı veya bir istemcinin mikrofonu kısa süreliğine etkinleştirmesi olabilir.
- Yakında bir araya girme veya `provider-clear-audio` olmadan `audio playback stopped reason=stream-close` görülmesi, yerel Discord oynatma akışının beklenmedik şekilde sona erdiği anlamına gelir. Önceki sağlayıcı ve Discord oynatıcı günlüklerini kontrol edin.
- `capture ignored during playback (barge-in disabled)`, asistan sesi etkinken OpenClaw'ın girdiyi kasıtlı olarak bıraktığı anlamına gelir. Konuşmanın oynatmayı kesmesini istiyorsanız `voice.realtime.bargeIn` özelliğini etkinleştirin.
- `barge-in ignored ... outputActive=false`, Discord veya sağlayıcı VAD sisteminin konuşma bildirdiği ancak OpenClaw'ın kesilecek etkin bir oynatmasının olmadığı anlamına gelir. Bu durum sesi kesmemelidir.

Kimlik bilgileri bileşen başına çözümlenir: `voice.model` için LLM rota kimlik doğrulaması, `tools.media.audio` için STT kimlik doğrulaması, `tts`/`voice.tts` için TTS kimlik doğrulaması ve `voice.realtime.providers` ya da sağlayıcının normal kimlik doğrulama yapılandırması için gerçek zamanlı sağlayıcı kimlik doğrulaması.

### Sesli mesajlar

Discord sesli mesajları bir dalga biçimi önizlemesi gösterir ve OGG/Opus ses gerektirir. OpenClaw dalga biçimini otomatik olarak oluşturur ancak inceleme ve dönüştürme işlemleri için gateway ana makinesinde `ffmpeg` ve `ffprobe` bulunması gerekir.

- Bir **yerel dosya yolu** sağlayın (URL'ler reddedilir).
- Metin içeriğini atlayın (Discord aynı yükte metin + sesli mesajı reddeder).
- Herhangi bir ses biçimi kabul edilir; OpenClaw gerektiğinde OGG/Opus biçimine dönüştürür.

```bash
message(action="send", channel="discord", target="channel:123", path="/path/to/audio.mp3", asVoice=true)
```

## Sorun giderme

<AccordionGroup>
  <Accordion title="İzin verilmeyen intent'ler kullanılıyor veya bot sunucu mesajlarını göremiyor">

    - Message Content Intent özelliğini etkinleştirin
    - kullanıcı/üye çözümlemesine bağlıysanız Server Members Intent özelliğini etkinleştirin
    - intent'leri değiştirdikten sonra gateway'i yeniden başlatın

  </Accordion>

  <Accordion title="Sunucu mesajları beklenmedik şekilde engelleniyor">

    - `groupPolicy` değerini doğrulayın
    - `channels.discord.guilds` altındaki sunucu izin listesini doğrulayın
    - bir sunucu `channels` eşlemesi varsa yalnızca listelenen kanallara izin verilir
    - `requireMention` davranışını ve bahsetme kalıplarını doğrulayın

    Yararlı kontroller:

```bash
openclaw doctor
openclaw channels status --probe
openclaw logs --follow
```

  </Accordion>

  <Accordion title="Bahsetme gereksinimi kapalı ancak hâlâ engelleniyor">
    Yaygın nedenler:

    - eşleşen sunucu/kanal izin listesi olmadan `groupPolicy="allowlist"`
    - `requireMention` yanlış yerde yapılandırılmıştır (`channels.discord.guilds` veya bir kanal girdisi altında olmalıdır)
    - gönderen, sunucu/kanal `users` izin listesi tarafından engellenmiştir

  </Accordion>

  <Accordion title="Uzun süren Discord turları veya yinelenen yanıtlar">

    Tipik günlükler:

    - `Slow listener detected ...`
    - `stuck session: sessionKey=agent:...:discord:... state=processing ...`

    Discord, kuyruğa alınmış ajan turlarına kanalın sahip olduğu bir zaman aşımı uygulamaz. Mesaj dinleyicileri denetimi hemen devreder ve kuyruğa alınmış Discord çalıştırmaları, oturum/araç/çalışma zamanı yaşam döngüsü işi tamamlayana veya iptal edene kadar oturum başına sıralamayı korur.

  </Accordion>

  <Accordion title="Gateway meta verisi arama zaman aşımı uyarıları">
    OpenClaw, bağlanmadan önce Discord `/gateway/bot` meta verilerini getirir. Geçici hatalarda Discord'un varsayılan gateway URL'sine geri dönülür ve günlük kayıtları hız sınırına tabi tutulur.

    Meta veri zaman aşımı varsayılan olarak 30 saniyedir. Olağandışı ana makine ortamları için `OPENCLAW_DISCORD_GATEWAY_INFO_TIMEOUT_MS` bu değeri geçersiz kılabilir.

  </Accordion>

  <Accordion title="Gateway READY zaman aşımı nedeniyle yeniden başlatmalar">
    OpenClaw, başlatma sırasında ve çalışma zamanı yeniden bağlantılarından sonra Discord gateway `READY` olayını bekler. Başlatma işlemlerinin kademeli yapıldığı çok hesaplı kurulumlarda varsayılandan daha uzun bir başlangıç READY penceresi gerekebilir.

    Başlatma 15 saniye, çalışma zamanı yeniden bağlantıları ise 30 saniye bekler. `OPENCLAW_DISCORD_READY_TIMEOUT_MS` ve `OPENCLAW_DISCORD_RUNTIME_READY_TIMEOUT_MS`, olağandışı ana makine ortamları için kullanılabilir olmaya devam eder.

  </Accordion>

  <Accordion title="İzin denetimi uyuşmazlıkları">
    `channels status --probe` izin kontrolleri yalnızca sayısal kanal kimlikleriyle çalışır.

    Kısa ad anahtarları kullanırsanız çalışma zamanı eşleştirmesi yine çalışabilir ancak yoklama izinleri tam olarak doğrulayamaz.

  </Accordion>

  <Accordion title="DM ve eşleştirme sorunları">

    - DM devre dışı: `channels.discord.dm.enabled=false`
    - DM ilkesi devre dışı: `channels.discord.dmPolicy="disabled"` (eski: `channels.discord.dm.policy`)
    - `pairing` modunda eşleştirme onayı bekleniyor

  </Accordion>

  <Accordion title="Bottan bota döngüler">
    Varsayılan olarak botlar tarafından yazılan mesajlar yok sayılır.

    `channels.discord.allowBots=true` ayarını yaparsanız döngü davranışını önlemek için katı bahsetme ve izin listesi kuralları kullanın.
    Yalnızca bottan bahseden bot mesajlarını kabul etmek için `channels.discord.allowBots="mentions"` seçeneğini tercih edin.

    OpenClaw ayrıca paylaşılan [bot döngüsü korumasıyla](/tr/channels/bot-loop-protection) birlikte gelir. `allowBots` botlar tarafından yazılan mesajların dağıtıma ulaşmasına izin verdiğinde Discord, gelen olayı `(account, channel, bot pair)` olgularına eşler ve genel çift koruması, yapılandırılmış olay bütçesi aşıldıktan sonra çifti engeller. Koruma, daha önce Discord hız sınırlarıyla durdurulması gereken kontrolden çıkmış iki botlu döngüleri önler; tek botlu dağıtımları veya bütçenin altında kalan tek seferlik bot yanıtlarını etkilemez.

    Varsayılan ayarlar (`allowBots` ayarlandığında etkindir):

    - `maxEventsPerWindow: 20` -- bot çifti, kayan pencere içinde 20 mesaj alışverişi yapabilir
    - `windowSeconds: 60` -- kayan pencerenin uzunluğu
    - `cooldownSeconds: 60` -- bütçe aşıldığında, iki yöndeki her ek bottan bota mesaj bir dakika boyunca bırakılır

    Paylaşılan varsayılanı `channels.defaults.botLoopProtection` altında bir kez yapılandırın, ardından meşru bir iş akışı daha fazla hareket alanına ihtiyaç duyduğunda Discord için geçersiz kılın. Öncelik sırası şöyledir:

    - `channels.discord.accounts.<account>.botLoopProtection`
    - `channels.discord.botLoopProtection`
    - `channels.defaults.botLoopProtection`
    - yerleşik varsayılanlar

    Discord genel `maxEventsPerWindow`, `windowSeconds` ve `cooldownSeconds` anahtarlarını kullanır.

```json5
{
  channels: {
    defaults: {
      botLoopProtection: {
        maxEventsPerWindow: 20,
        windowSeconds: 60,
        cooldownSeconds: 60,
      },
    },
    discord: {
      // İsteğe bağlı Discord geneli geçersiz kılma. Hesap blokları tek tek
      // alanları geçersiz kılar ve atlanan alanları buradan devralır.
      botLoopProtection: {
        maxEventsPerWindow: 4,
      },
      accounts: {
        alpha: {
          // Alpha yalnızca kendisinden bahsettiklerinde diğer botları dinler.
          allowBots: "mentions",
        },
        bravo: {
          // Bravo, botlar tarafından yazılan tüm Discord mesajlarını dinler.
          allowBots: true,
          mentionAliases: {
            // Bravo'nun yapılandırılmış kullanıcı kimliğiyle bir Alpha Discord bahsetmesi yazmasını sağlar.
            Alpha: "ALPHA_DISCORD_USER_ID",
          },
          botLoopProtection: {
            // Çifti engellemeden önce dakikada en fazla beş mesaja izin ver.
            maxEventsPerWindow: 5,
            windowSeconds: 60,
            cooldownSeconds: 90,
          },
        },
      },
    },
  },
}
```

  </Accordion>

  <Accordion title="DecryptionFailed(...) ile ses STT kayıpları">

    - Discord ses alma kurtarma mantığının mevcut olması için OpenClaw'ı güncel tutun (`openclaw update`)
    - `channels.discord.voice.daveEncryption=true` değerini doğrulayın (varsayılan)
    - `channels.discord.voice.decryptionFailureTolerance=24` değerinden (yukarı akış varsayılanı) başlayın ve yalnızca gerekirse ayarlayın
    - günlüklerde şunları izleyin:
      - `discord voice: DAVE decrypt failures detected`
      - `discord voice: repeated decrypt failures; attempting rejoin`
    - otomatik yeniden katılımdan sonra hatalar devam ederse günlükleri toplayın ve [discord.js #11419](https://github.com/discordjs/discord.js/issues/11419) ile [discord.js #11449](https://github.com/discordjs/discord.js/pull/11449) içindeki yukarı akış DAVE alma geçmişiyle karşılaştırın

  </Accordion>
</AccordionGroup>

## Yapılandırma referansı

Birincil referans: [Yapılandırma referansı - Discord](/tr/gateway/config-channels#discord).

<Accordion title="Yüksek sinyalli Discord alanları">

- başlatma/kimlik doğrulama: `enabled`, `token`, `applicationId`, `accounts.*`, `allowBots`
- ilke: `groupPolicy`, `dmPolicy`, `allowFrom`, `dm.*`, `guilds.*`, `guilds.*.channels.*`
- komut: `commands.native`, `commands.useAccessGroups` (genel), `configWrites`, `slashCommand.ephemeral`
- gateway: `proxy`
- yanıt/geçmiş: `replyToMode`, `historyLimit`, `dmHistoryLimit`, `dms.*.historyLimit`
- teslim: `textChunkLimit` (varsayılan `2000`), `maxLinesPerMessage` (varsayılan `17`)
- akış: `streaming.mode`, `streaming.chunkMode`, `streaming.preview.*`, `streaming.progress.*`, `streaming.block.*` (eski düz `streamMode`, `draftChunk`, `blockStreaming`, `blockStreamingCoalesce`, `chunkMode` anahtarları `openclaw doctor --fix` tarafından `streaming.*` içine taşınır)
- medya: `mediaMaxMb` (giden Discord yüklemelerini sınırlar, varsayılan `100`)
- eylemler: `actions.*`
- durum: `activity`, `status`, `activityType`, `activityUrl`, `autoPresence.*`
- kullanıcı arayüzü: `ui.components.accentColor`
- özellikler: `threadBindings`, üst düzey `bindings[]` (`type: "acp"`), `pluralkit`, `execApprovals`, `intents`, `agentComponents.enabled`, `agentComponents.ttlMs`, `activities`, `heartbeat`, `responsePrefix`

</Accordion>

### Discord Activities

Ajanların Discord içinde açılan bağımsız HTML araç takımları gönderebilmesi için `channels.discord.activities` ayarını yapın. Blok isteğe bağlıdır; mevcut olmadığında OpenClaw hiçbir Activity rotası, aracı veya etkileşim işleyicisi kaydetmez. Developer Portal, tünel, güvenlik ve sorun giderme kurulumu için [Discord Activities](/channels/discord-activities) sayfasına bakın.

- `activities.clientSecret`: Discord uygulamasının OAuth2 istemci sırrı; `DISCORD_CLIENT_SECRET` değerine geri döner
- `activities.applicationId`: isteğe bağlı Activity uygulama kimliği; varsayılan olarak gateway başlatılırken öğrenilen bot uygulama kimliğini kullanır

## Güvenlik ve operasyonlar

- Bot token'larını sır olarak değerlendirin (denetimli ortamlarda `DISCORD_BOT_TOKEN` tercih edilir).
- Discord'a en düşük ayrıcalıklı izinleri verin.
- Komut dağıtımı/durumu güncel değilse gateway'i yeniden başlatın ve `openclaw channels status --probe` ile tekrar kontrol edin.

## İlgili

<CardGroup cols={2}>
  <Card title="Discord Activities" icon="window" href="/channels/discord-activities">
    Discord içinde etkileşimli HTML araç takımları başlatın.
  </Card>
  <Card title="Eşleştirme" icon="link" href="/tr/channels/pairing">
    Bir Discord kullanıcısını gateway ile eşleştirin.
  </Card>
  <Card title="Gruplar" icon="users" href="/tr/channels/groups">
    Grup sohbeti ve izin listesi davranışı.
  </Card>
  <Card title="Kanal yönlendirme" icon="route" href="/tr/channels/channel-routing">
    Gelen mesajları ajanlara yönlendirin.
  </Card>
  <Card title="Güvenlik" icon="shield" href="/tr/gateway/security">
    Tehdit modeli ve sağlamlaştırma.
  </Card>
  <Card title="Çok ajanlı yönlendirme" icon="sitemap" href="/tr/concepts/multi-agent">
    Sunucuları ve kanalları ajanlarla eşleyin.
  </Card>
  <Card title="Eğik çizgi komutları" icon="terminal" href="/tr/tools/slash-commands">
    Yerel komut davranışı.
  </Card>
</CardGroup>
