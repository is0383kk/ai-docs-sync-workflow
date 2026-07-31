---
read_when:
    - Codex, Claude Code veya başka bir MCP istemcisini OpenClaw destekli kanallara bağlama
    - '`openclaw mcp serve` çalıştırılıyor'
    - OpenClaw tarafından kaydedilen MCP sunucu tanımlarını yönetme
sidebarTitle: MCP
summary: OpenClaw kanal konuşmalarını MCP üzerinden kullanıma açın ve kaydedilmiş MCP sunucu tanımlarını yönetin
title: MCP
x-i18n:
    generated_at: "2026-07-26T23:13:11Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: ee6146bbc0181d10997336094d1bd693d0afb0985f1febef8e8c6b0d6e656cf9
    source_path: cli/mcp.md
    workflow: 16
---

`openclaw mcp` iki göreve sahiptir:

- `openclaw mcp serve` ile OpenClaw'ı bir MCP sunucusu olarak çalıştırmak
- `list`, `show`, `status`, `doctor`, `probe`, `add`, `set`, `configure`, `tools`, `login`, `logout`, `reload` ve `unset` ile OpenClaw tarafından yönetilen giden MCP sunucusu tanımlarını yönetmek

`serve`, OpenClaw'ın bir MCP sunucusu olarak çalışmasıdır. Diğer alt komutlar ise OpenClaw'ın, kendi çalışma zamanlarının daha sonra kullanabileceği sunucular için MCP istemci tarafı kayıt defteri olarak çalışmasıdır.

<Note>
  `list`, `show`, `set` ve `unset`, OpenClaw yapılandırmasında yalnızca OpenClaw tarafından yönetilen `mcp.servers` girdilerini okur ve yazar. `config/mcporter.json` içindeki mcporter sunucularını içermezler; bu kayıt defteri için `mcporter list` kullanın.
</Note>

OpenClaw'ın bir kodlama altyapısı oturumunu kendisinin barındırması ve bu çalışma zamanını ACP üzerinden yönlendirmesi gerektiğinde [`openclaw acp`](/tr/cli/acp) kullanın.

## Doğru MCP yolunu seçme

| Amaç                                                                | Kullanım                                                              | Nedeni                                                                                                          |
| ------------------------------------------------------------------- | -------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------- |
| Harici bir MCP istemcisinin OpenClaw kanal konuşmalarını okumasına/göndermesine izin vermek | `openclaw mcp serve`                                                 | OpenClaw MCP sunucusudur ve Gateway destekli konuşmaları stdio üzerinden sunar.                                 |
| OpenClaw tarafından yönetilen ajan çalıştırmaları için üçüncü taraf MCP sunucularını kaydetmek | `openclaw mcp add`, `set`, `configure`, `tools`, `login`             | OpenClaw, MCP istemci tarafı kayıt defteridir ve daha sonra bu sunucuları uygun çalışma zamanlarına yansıtır.    |
| Bir ajan turu çalıştırmadan kayıtlı bir sunucuyu denetlemek          | `openclaw mcp status`, `doctor`, `probe`                             | `status` ve `doctor` yapılandırmayı inceler; `probe` canlı bir MCP bağlantısı açar ve yetenekleri listeler. |
| MCP yapılandırmasını tarayıcıdan düzenlemek                          | Control UI `/settings/mcp` (`/mcp` diğer adı)                            | Sayfa; envanteri, etkinleştirme durumunu, OAuth/filtre özetlerini, komut ipuçlarını ve kapsamlı bir `mcp` düzenleyicisini gösterir. |
| Codex app-server'a kapsamlı bir yerel MCP sunucusu vermek            | `mcp.servers.<name>.codex`                                           | `codex` bloğu yalnızca Codex app-server iş parçacığı yansıtmasını etkiler ve yerel yapılandırma devredilmeden önce kaldırılır. |
| ACP tarafından barındırılan altyapı oturumlarını çalıştırmak         | [`openclaw acp`](/tr/cli/acp) ve [ACP Ajanları](/tr/tools/acp-agents-setup) | ACP köprü modu oturum başına MCP sunucusu eklenmesini kabul etmez; bunun yerine gateway/plugin köprülerini yapılandırın. |

<Tip>
Hangi yola ihtiyacınız olduğundan emin değilseniz `openclaw mcp status --verbose` ile başlayın. Bu, herhangi bir MCP sunucusu başlatmadan OpenClaw'ın kaydettiklerini gösterir.
</Tip>

## MCP sunucusu olarak OpenClaw

Bu, `openclaw mcp serve` yoludur.

### serve ne zaman kullanılmalı?

Şu durumlarda `openclaw mcp serve` kullanın:

- Codex, Claude Code veya başka bir MCP istemcisi doğrudan OpenClaw destekli kanal konuşmalarıyla iletişim kurmalıysa
- yönlendirilmiş oturumlara sahip yerel veya uzak bir OpenClaw Gateway zaten varsa
- kanal başına ayrı köprüler çalıştırmak yerine OpenClaw'ın tüm kanal arka uçlarında çalışan tek bir MCP sunucusu istiyorsanız

OpenClaw'ın kodlama çalışma zamanını kendisinin barındırması ve ajan oturumunu OpenClaw içinde tutması gerektiğinde bunun yerine [`openclaw acp`](/tr/cli/acp) kullanın.

### Nasıl çalışır?

`openclaw mcp serve`, bir stdio MCP sunucusu başlatır. Bu işlemin sahibi MCP istemcisidir. İstemci stdio oturumunu açık tuttuğu sürece köprü, WebSocket üzerinden yerel veya uzak bir OpenClaw Gateway'e bağlanır ve yönlendirilmiş kanal konuşmalarını MCP üzerinden sunar.

<Steps>
  <Step title="İstemci köprüyü başlatır">
    MCP istemcisi `openclaw mcp serve` işlemini başlatır.
  </Step>
  <Step title="Köprü Gateway'e bağlanır">
    Köprü, WebSocket üzerinden OpenClaw Gateway'e bağlanır.
  </Step>
  <Step title="Oturumlar MCP konuşmalarına dönüşür">
    Yönlendirilmiş oturumlar MCP konuşmalarına ve transkript/geçmiş araçlarına dönüşür.
  </Step>
  <Step title="Canlı olaylar kuyruğa alınır">
    Köprü bağlıyken canlı olaylar bellekte kuyruğa alınır.
  </Step>
  <Step title="İsteğe bağlı Claude iletimi">
    Claude kanal modu etkinse aynı oturum Claude'a özgü anlık bildirimleri de alabilir.
  </Step>
</Steps>

<AccordionGroup>
  <Accordion title="Önemli davranışlar">
    - canlı kuyruk durumu köprü bağlandığında başlar
    - eski transkript geçmişi `messages_read` ile okunur
    - Claude anlık bildirimleri yalnızca MCP oturumu etkin olduğu sürece bulunur
    - istemci bağlantıyı kestiğinde köprü kapanır ve canlı kuyruk silinir
    - `openclaw agent` ve `openclaw infer model run` gibi tek seferlik ajan giriş noktaları, yanıt tamamlandığında açtıkları tüm paketlenmiş MCP çalışma zamanlarını sonlandırır; böylece tekrarlanan betik çalıştırmaları stdio MCP alt süreçlerinin birikmesine yol açmaz
    - OpenClaw tarafından başlatılan stdio MCP sunucuları (paketlenmiş veya kullanıcı tarafından yapılandırılmış), kapatma sırasında bir süreç ağacı olarak sonlandırılır; böylece sunucunun başlattığı alt süreçler, üst stdio istemcisi kapandıktan sonra çalışmaya devam etmez
    - bir oturumun silinmesi veya sıfırlanması, paylaşılan çalışma zamanı temizleme yolu üzerinden o oturumun MCP istemcilerini sonlandırır; böylece kaldırılmış bir oturuma bağlı kalan stdio bağlantıları olmaz

  </Accordion>
</AccordionGroup>

### İstemci modu seçme

<Tabs>
  <Tab title="Genel MCP istemcileri">
    Yalnızca standart MCP araçları. `conversations_list`, `messages_read`, `events_poll`, `events_wait`, `messages_send` ve onay araçlarını kullanın.
  </Tab>
  <Tab title="Claude Code">
    Standart MCP araçlarına ek olarak Claude'a özgü kanal bağdaştırıcısı. `--claude-channel-mode on` seçeneğini etkinleştirin veya varsayılan `auto` değerini koruyun.
  </Tab>
</Tabs>

<Note>
Şu anda `auto`, `on` ile aynı şekilde davranır. Henüz istemci yeteneği algılama özelliği yoktur.
</Note>

### serve tarafından sunulanlar

Köprü, kanal destekli konuşmaları sunmak için mevcut Gateway oturum yönlendirme meta verilerini kullanır. OpenClaw, aşağıdaki gibi bilinen bir yönlendirmeye sahip oturum durumunu zaten içeriyorsa bir konuşma görünür:

- `channel`
- alıcı veya hedef meta verileri
- isteğe bağlı `accountId`
- isteğe bağlı `threadId`

Bu, MCP istemcilerine aşağıdaki işlemler için tek bir yer sağlar:

- son yönlendirilmiş konuşmaları listelemek
- son transkript geçmişini okumak
- yeni gelen olayları beklemek
- aynı yönlendirme üzerinden yanıt göndermek
- köprü bağlıyken gelen onay isteklerini görmek

### Kullanım

<Tabs>
  <Tab title="Yerel Gateway">
    ```bash
    openclaw mcp serve
    ```
  </Tab>
  <Tab title="Uzak Gateway (token)">
    ```bash
    openclaw mcp serve --url wss://gateway-host:18789 --token-file ~/.openclaw/gateway.token
    ```
  </Tab>
  <Tab title="Uzak Gateway (parola)">
    ```bash
    openclaw mcp serve --url wss://gateway-host:18789 --password-file ~/.openclaw/gateway.password
    ```
  </Tab>
  <Tab title="Ayrıntılı / Claude kapalı">
    ```bash
    openclaw mcp serve --verbose
    openclaw mcp serve --claude-channel-mode off
    ```
  </Tab>
</Tabs>

### Köprü araçları

<AccordionGroup>
  <Accordion title="conversations_list">
    Gateway oturum durumunda zaten yönlendirme meta verilerine sahip olan son oturum destekli konuşmaları listeler.

    Filtreler: `limit` (en fazla 500), `search`, `channel`, `includeDerivedTitles`, `includeLastMessage`.

  </Accordion>
  <Accordion title="conversation_get">
    Doğrudan Gateway oturum araması kullanarak `session_key` değerine göre bir konuşma döndürür.
  </Accordion>
  <Accordion title="messages_read">
    Oturum destekli bir konuşmanın son transkript iletilerini okur. `limit` varsayılan olarak 20, en fazla 200'dür.
  </Accordion>
  <Accordion title="attachments_fetch">
    Bir transkript iletisindeki metin dışı ileti içerik bloklarını ayıklar. Bu, bağımsız ve kalıcı bir ek ikili nesne deposu değil, transkript içeriğinin meta veri görünümüdür.
  </Accordion>
  <Accordion title="events_poll">
    Sayısal bir imleçten itibaren kuyruğa alınmış canlı olayları okur. `limit` en fazla 200'dür.
  </Accordion>
  <Accordion title="events_wait">
    Bir sonraki eşleşen kuyruk olayı gelene veya zaman aşımı süresi dolana kadar uzun yoklama yapar (varsayılan 30s, en fazla 300s).

    Genel bir MCP istemcisinin Claude'a özgü bir anlık bildirim protokolü olmadan gerçek zamana yakın teslimata ihtiyacı olduğunda bunu kullanın.

  </Accordion>
  <Accordion title="messages_send">
    Metni, oturumda zaten kayıtlı olan aynı yönlendirme üzerinden geri gönderir.

    Mevcut davranış:

    - mevcut bir konuşma yönlendirmesi gerektirir
    - oturumun kanalını, alıcısını, hesap kimliğini ve iş parçacığı kimliğini kullanır
    - yalnızca metin gönderir

  </Accordion>
  <Accordion title="permissions_list_open">
    Köprünün Gateway'e bağlandığından beri gözlemlediği bekleyen yürütme/plugin onay isteklerini listeler.
  </Accordion>
  <Accordion title="permissions_respond">
    Bekleyen bir yürütme/plugin onay isteğini aşağıdakilerden biriyle çözümler:

    - `allow-once`
    - `allow-always`
    - `deny`

  </Accordion>
</AccordionGroup>

### Olay modeli

Köprü, bağlı olduğu sürece bellekte bir olay kuyruğu tutar.

Mevcut olay türleri:

- `message`
- `exec_approval_requested`
- `exec_approval_resolved`
- `plugin_approval_requested`
- `plugin_approval_resolved`
- `claude_permission_request`

<Warning>
- kuyruk yalnızca canlıdır; MCP köprüsü başlatıldığında başlar
- `events_poll` ve `events_wait`, eski Gateway geçmişini kendi başlarına yeniden oynatmaz
- kalıcı birikmiş işler `messages_read` ile okunmalıdır

</Warning>

### Claude kanal bildirimleri

Köprü ayrıca Claude'a özgü kanal bildirimlerini sunabilir. Bu, Claude Code kanal bağdaştırıcısının OpenClaw karşılığıdır: standart MCP araçları kullanılabilir durumda kalırken canlı gelen iletiler Claude'a özgü MCP bildirimleri olarak da ulaşabilir.

<Tabs>
  <Tab title="kapalı">
    `--claude-channel-mode off`: yalnızca standart MCP araçları.
  </Tab>
  <Tab title="açık">
    `--claude-channel-mode on`: Claude kanal bildirimlerini etkinleştirir.
  </Tab>
  <Tab title="otomatik (varsayılan)">
    `--claude-channel-mode auto`: mevcut varsayılan; `on` ile aynı köprü davranışı.
  </Tab>
</Tabs>

Claude kanal modu etkinleştirildiğinde sunucu, Claude deneysel yeteneklerini bildirir ve şunları gönderebilir:

- `notifications/claude/channel`
- `notifications/claude/channel/permission`

Mevcut köprü davranışı:

- gelen `user` transkript iletileri `notifications/claude/channel` olarak iletilir
- MCP üzerinden alınan Claude izin istekleri bellekte izlenir
- bağlantılı konuşmadaki komut sahibi daha sonra `yes <id>` veya `no <id>` gönderirse (`<id>`, `l` hariç 5 harfli istek kimliğidir), köprü bunu `notifications/claude/channel/permission` değerine dönüştürür
- bu bildirimler yalnızca canlı oturum içindir; MCP istemcisi bağlantıyı keserse anlık bildirim hedefi kalmaz

Bu, bilinçli olarak istemciye özgüdür. Genel MCP istemcileri standart yoklama araçlarını kullanmalıdır.

### MCP istemci yapılandırması

Örnek stdio istemci yapılandırması:

```json
{
  "mcpServers": {
    "openclaw": {
      "command": "openclaw",
      "args": [
        "mcp",
        "serve",
        "--url",
        "wss://gateway-host:18789",
        "--token-file",
        "/path/to/gateway.token"
      ]
    }
  }
}
```

Çoğu genel MCP istemcisi için standart araç yüzeyiyle başlayın ve Claude modunu dikkate almayın. Claude modunu yalnızca Claude'a özgü bildirim yöntemlerini gerçekten anlayan istemciler için açın.

### Seçenekler

`openclaw mcp serve` şunları destekler:

<ParamField path="--url" type="string">
  Gateway WebSocket URL'si. Yapılandırıldığında varsayılan değer `gateway.remote.url` olur.
</ParamField>
<ParamField path="--token" type="string">
  Gateway belirteci.
</ParamField>
<ParamField path="--token-file" type="string">
  Belirteci dosyadan oku.
</ParamField>
<ParamField path="--password" type="string">
  Gateway parolası.
</ParamField>
<ParamField path="--password-file" type="string">
  Parolayı dosyadan oku.
</ParamField>
<ParamField path="--claude-channel-mode" type='"auto" | "on" | "off"'>
  Claude bildirim modu. Varsayılan değer `auto`.
</ParamField>
<ParamField path="-v, --verbose" type="boolean">
  stderr üzerinde ayrıntılı günlükler.
</ParamField>

<Tip>
Mümkün olduğunda satır içi gizli bilgiler yerine `--token-file` veya `--password-file` tercih edin.
</Tip>

### Güvenlik ve güven sınırı

Köprü yönlendirme oluşturmaz. Yalnızca Gateway'in zaten nasıl yönlendireceğini bildiği konuşmaları kullanıma sunar.

Bunun anlamı şudur:

- gönderen izin listeleri, eşleştirme ve kanal düzeyindeki güven, temel OpenClaw kanal yapılandırmasına ait olmaya devam eder
- `messages_send` yalnızca mevcut bir kayıtlı rota üzerinden yanıt verebilir
- onay durumu yalnızca geçerli köprü oturumu için canlı/bellek içindedir
- köprü kimlik doğrulaması, diğer tüm uzak Gateway istemcileri için güveneceğiniz aynı Gateway belirteci veya parola denetimlerini kullanmalıdır

`conversations_list` içinde bir konuşma eksikse olağan neden MCP yapılandırması değildir. Temel Gateway oturumundaki rota meta verilerinin eksik veya tamamlanmamış olmasıdır.

### Test

OpenClaw bu köprü için deterministik bir Docker duman testiyle birlikte gelir:

```bash
pnpm test:docker:mcp-channels
```

Bu duman testi tek bir kapsayıcı çalıştırır: konuşma durumunu başlangıç verileriyle doldurur, Gateway'i başlatır, ardından `openclaw mcp serve` öğesini bir stdio alt süreci olarak oluşturur ve onu bir MCP istemcisi olarak çalıştırır. Gerçek stdio MCP köprüsü üzerinden konuşma keşfini, transkript okumalarını, ek meta verisi okumalarını, canlı olay kuyruğu davranışını ve Claude tarzı kanal ve izin bildirimlerini doğrular. Giden gönderim yönlendirmesi (kayıtlı konuşma rotasını yeniden kullanan `messages_send`) `src/mcp/channel-server.test.ts` içindeki birim testleri tarafından ayrıca kapsanır.

Bu, test çalıştırmasına gerçek bir Telegram, Discord veya iMessage hesabı bağlamadan köprünün çalıştığını kanıtlamanın en hızlı yoludur.

Daha geniş test bağlamı için [Test](/tr/help/testing) bölümüne bakın.

### Sorun giderme

<AccordionGroup>
  <Accordion title="Hiçbir konuşma döndürülmüyor">
    Genellikle Gateway oturumunun henüz yönlendirilebilir olmadığı anlamına gelir. Temel oturumda kayıtlı kanal/sağlayıcı, alıcı ve isteğe bağlı hesap/ileti dizisi rota meta verilerinin bulunduğunu doğrulayın.
  </Accordion>
  <Accordion title="events_poll veya events_wait eski mesajları kaçırıyor">
    Beklenen davranıştır. Canlı kuyruk, köprü bağlandığında başlar. Eski transkript geçmişini `messages_read` ile okuyun.
  </Accordion>
  <Accordion title="Claude bildirimleri görünmüyor">
    Şunların tümünü kontrol edin:

    - istemci stdio MCP oturumunu açık tuttu
    - `--claude-channel-mode`, `on` veya `auto` değerindedir
    - istemci Claude'a özgü bildirim yöntemlerini gerçekten anlıyor
    - gelen mesaj köprü bağlandıktan sonra gerçekleşti

  </Accordion>
  <Accordion title="Onaylar eksik">
    `permissions_list_open` yalnızca köprü bağlıyken gözlemlenen onay isteklerini gösterir. Kalıcı bir onay geçmişi API'si değildir.
  </Accordion>
</AccordionGroup>

## MCP istemci kayıt defteri olarak OpenClaw

Bu, `openclaw mcp list`, `show`, `status`, `doctor`, `probe`, `add`, `set`,
`configure`, `tools`, `login`, `logout`, `reload` ve `unset` yoludur.

Bu komutlar OpenClaw'u MCP üzerinden kullanıma sunmaz. OpenClaw yapılandırmasında `mcp.servers` altındaki OpenClaw tarafından yönetilen MCP sunucu tanımlarını yönetirler. `config/mcporter.json` içindeki mcporter sunucularını okumazlar.

Kaydedilen bu tanımlar, gömülü OpenClaw ve diğer çalışma zamanı bağdaştırıcıları gibi OpenClaw'un daha sonra başlattığı veya yapılandırdığı çalışma zamanları içindir. OpenClaw, bu çalışma zamanlarının kendi yinelenen MCP sunucu listelerini tutmak zorunda kalmaması için tanımları merkezi olarak depolar.

<AccordionGroup>
  <Accordion title="Önemli davranış">
    - bu komutlar yalnızca OpenClaw yapılandırmasını okur veya yazar
    - `status`, `list`, `show`, `--probe` olmadan `doctor`, `set`, `configure`, `tools`, `logout`, `reload` ve `unset` hedef MCP sunucusuna bağlanmaz
    - `login`, yapılandırılmış HTTP sunucusu için MCP OAuth ağ akışını gerçekleştirir ve sonuçta elde edilen yerel kimlik bilgilerini kaydeder
    - `status --verbose`, bağlanmadan çözümlenmiş aktarım, kimlik doğrulama, zaman aşımı, filtre ve paralel araç çağrısı ipuçlarını yazdırır
    - `doctor`, kayıtlı tanımlarda eksik stdio komutları, geçersiz çalışma dizinleri, eksik TLS dosyaları, devre dışı sunucular, değişmez hassas üstbilgi/ortam değerleri ve tamamlanmamış OAuth yetkilendirmesi gibi yerel kurulum sorunlarını denetler
    - `doctor --probe`, statik denetimler geçtikten sonra `probe` ile aynı canlı bağlantı kanıtını ekler
    - `probe`, seçilen sunucuya veya yapılandırılmış tüm sunuculara bağlanır, araçları listeler ve yetenekleri/tanıları bildirir
    - `add`, bayraklardan bir tanım oluşturur ve `--no-probe` ayarlanmadığı veya önce OAuth yetkilendirmesi gerekmediği sürece kaydetmeden önce yoklama yapar
    - çalışma zamanı bağdaştırıcıları, yürütme sırasında gerçekte hangi aktarım biçimlerini desteklediklerine karar verir
    - `enabled: false`, bir sunucuyu kayıtlı tutar ancak gömülü çalışma zamanı keşfinin dışında bırakır
    - `requestTimeoutMs` ve `connectionTimeoutMs`, sunucu başına istek ve bağlantı zaman aşımlarını milisaniye cinsinden ayarlar
    - `supportsParallelToolCalls: true`, bağdaştırıcıların eşzamanlı olarak çağırabileceği sunucuları işaretler
    - HTTP sunucuları statik üstbilgileri, OAuth oturum açmayı, TLS doğrulama denetimini ve mTLS sertifika/anahtar yollarını kullanabilir
    - gömülü OpenClaw, yapılandırılmış MCP araçlarını normal `coding` ve `messaging` araç profillerinde kullanıma sunar; `minimal` bunları yine gizler ve `tools.deny: ["bundle-mcp"]` bunları açıkça devre dışı bırakır
    - sunucu başına `toolFilter.include` ve `toolFilter.exclude`, keşfedilen MCP araçlarını OpenClaw araçlarına dönüşmeden önce filtreler
    - kaynakları veya istemleri duyuran sunucular ayrıca kaynakları listelemek/okumak ve istemleri listelemek/getirmek için yardımcı araçları kullanıma sunar; oluşturulan bu yardımcı adları (`resources_list`, `resources_read`, `prompts_list`, `prompts_get`) aynı dahil etme/dışlama filtresini kullanır
    - dinamik MCP araç listesi değişiklikleri, o oturum için önbelleğe alınmış kataloğu geçersiz kılar; sonraki keşif/kullanım sunucudan yeniler
    - yinelenen MCP araç isteği/protokol hataları, bozuk bir sunucunun tüm turu tüketmemesi için o sunucuyu kısa süreliğine duraklatır
    - oturum kapsamındaki paketlenmiş MCP çalışma zamanları 10 dakikalık boşta kalma süresinden sonra sonlandırılır ve tek seferlik gömülü çalıştırmalar bunları çalıştırma sonunda temizler

  </Accordion>
</AccordionGroup>

Çalışma zamanı bağdaştırıcıları bu paylaşılan kayıt defterini, sonraki istemcilerinin beklediği biçime normalleştirebilir. Örneğin gömülü OpenClaw, OpenClaw `transport` değerlerini doğrudan tüketirken Claude Code ve Gemini, `http`, `sse` veya `stdio` gibi CLI'ya özgü `type` değerlerini alır.

Codex uygulama sunucusu ayrıca her sunucudaki isteğe bağlı `codex` bloğunu dikkate alır. Bu,
yalnızca Codex uygulama sunucusu ileti dizileri için OpenClaw projeksiyon meta verisidir; ACP
oturumlarını, genel Codex çalıştırma düzeneği yapılandırmasını veya diğer çalışma zamanı bağdaştırıcılarını
değiştirmez. Bir sunucuyu yalnızca belirli OpenClaw ajan kimliklerine yansıtmak için boş olmayan
`codex.agents` kullanın. Boş, yalnızca boşluk içeren veya geçersiz ajan listeleri, genel
hale gelmek yerine yapılandırma doğrulaması tarafından reddedilir ve çalışma zamanı projeksiyon
yolundan çıkarılır. Güvenilen bir sunucu için Codex'in yerel `default_tools_approval_mode` değerini
yaymak üzere `codex.defaultToolsApprovalMode` (`auto`, `prompt` veya `approve`)
kullanın. OpenClaw, yerel `mcp_servers` yapılandırmasını Codex'e aktarmadan önce
`codex` meta verilerini kaldırır.

### Kaydedilmiş MCP sunucu tanımları

Komutlar:

- `openclaw mcp list`
- `openclaw mcp show [name]`
- `openclaw mcp status [--verbose]`
- `openclaw mcp doctor [name] [--probe]`
- `openclaw mcp probe [name]`
- `openclaw mcp add <name> [flags]`
- `openclaw mcp set <name> <json>`
- `openclaw mcp configure <name> [flags]`
- `openclaw mcp tools <name> [--include csv] [--exclude csv] [--clear]`
- `openclaw mcp login <name> [--code code]`
- `openclaw mcp logout <name>`
- `openclaw mcp reload`
- `openclaw mcp unset <name>`

Notlar:

- `list`, sunucu adlarını sıralar.
- adsız `show`, yapılandırılmış MCP sunucu nesnesinin tamamını yazdırır.
- `status`, yapılandırılmış aktarımları bağlanmadan sınıflandırır. `--verbose`, depolanan OAuth belirteçlerinin ek yetkilendirme gerektirdiği durumlar dahil olmak üzere çözümlenmiş başlatma, zaman aşımı, OAuth, filtre ve paralel çağrı ayrıntılarını içerir. Kimlik bilgileri içeren stdio bağımsız değişkenleri metin ve JSON çıktısında gizlenir.
- `doctor`, bağlanmadan statik denetimler gerçekleştirir. Komutun etkin sunucuların bağlandığını da doğrulaması gerektiğinde `--probe` ekleyin.
- `probe`, bağlanır ve araç sayılarını, kaynak/istem desteğini, liste değişikliği desteğini ve tanıları bildirir.
- `add`; `--command`, `--arg`, `--env` ve `--cwd` gibi stdio bayraklarını veya `--url`, `--transport`, `--header`, `--auth oauth`, TLS, zaman aşımı ve araç seçimi bayrakları gibi HTTP bayraklarını kabul eder.
- `set`, komut satırında tek bir JSON nesne değeri bekler.
- `configure`, sunucu tanımının tamamını değiştirmeden etkinleştirmeyi, araç filtrelerini, zaman aşımlarını, OAuth'ı, TLS'yi ve paralel araç çağrısı ipuçlarını günceller. Güncellenen sunucuyu kaydetmeden önce doğrulamak için `--probe` ekleyin.
- `tools`, sunucu başına araç filtrelerini günceller. Dahil etme/dışlama girdileri MCP araç adları ve basit `*` glob kalıplarıdır.
- `login`, `auth: "oauth"` ile yapılandırılmış HTTP sunucuları için OAuth akışını çalıştırır. İlk çalıştırma bir yetkilendirme URL'si yazdırır; onaydan sonra `--code` ile yeniden çalıştırın.
- `logout`, kaydedilmiş sunucu tanımını kaldırmadan adlandırılmış sunucu için depolanan OAuth kimlik bilgilerini temizler.
- `reload`, yalnızca geçerli CLI süreci için önbelleğe alınmış süreç içi MCP çalışma zamanlarını sonlandırır. Başka bir süreçteki Gateway veya ajan süreçleri yine kendi yeniden yükleme veya yeniden başlatma yollarına ihtiyaç duyar.
- Streamable HTTP MCP sunucuları için `transport: "streamable-http"` kullanın. `openclaw mcp set`, uyumluluk amacıyla CLI'ya özgü `type: "http"` değerini de aynı kurallı yapılandırma biçimine normalleştirir.
- `unset`, adlandırılmış sunucu mevcut değilse başarısız olur.

Örnekler:

```bash
openclaw mcp list
openclaw mcp show context7 --json
openclaw mcp status --verbose
openclaw mcp doctor --probe
openclaw mcp probe context7 --json
openclaw mcp add memory --command npx --arg -y --arg @modelcontextprotocol/server-memory
openclaw mcp set context7 '{"command":"uvx","args":["context7-mcp"]}'
openclaw mcp tools context7 --include 'resolve-library-id,get-library-docs'
openclaw mcp set docs '{"url":"https://mcp.example.com","transport":"streamable-http"}'
openclaw mcp configure docs --timeout 20 --connect-timeout 5 --include 'search,read_*'
openclaw mcp configure docs --auth oauth --oauth-scope 'docs.read'
openclaw mcp login docs
openclaw mcp logout docs
openclaw mcp unset context7
```

### Yaygın sunucu tarifleri

Bu örnekler yalnızca sunucu tanımlarını kaydeder. Sunucunun başlatıldığını ve araçları kullanıma sunduğunu doğrulamak için daha sonra `openclaw mcp doctor --probe` komutunu çalıştırın.

<Tabs>
  <Tab title="Dosya sistemi">
    ```bash
    openclaw mcp add files \
      --command npx \
      --arg -y \
      --arg @modelcontextprotocol/server-filesystem \
      --arg "$HOME/Documents" \
      --include 'read_file,list_directory,search_files'
    openclaw mcp doctor files --probe
    ```

    Dosya sistemi sunucularının kapsamını, aracının okuması veya düzenlemesi gereken en küçük dizin ağacıyla sınırlayın.

  </Tab>
  <Tab title="Bellek">
    ```bash
    openclaw mcp add memory \
      --command npx \
      --arg -y \
      --arg @modelcontextprotocol/server-memory
    openclaw mcp probe memory --json
    ```

    Sunucu, normal aracılar tarafından kullanılamaması gereken yazma araçları sunuyorsa bir araç filtresi kullanın.

  </Tab>
  <Tab title="Yerel betik">
    ```bash
    openclaw mcp add local-tools \
      --command node \
      --arg ./dist/mcp-server.js \
      --cwd /srv/openclaw-tools \
      --env API_BASE=https://internal.example
    openclaw mcp status --verbose
    ```

    `doctor`, `cwd` öğesinin var olduğunu ve komutun yapılandırılan ortamdan çözümlendiğini denetler.

  </Tab>
  <Tab title="Uzak HTTP">
    ```bash
    openclaw mcp add docs \
      --url https://mcp.example.com/mcp \
      --transport streamable-http \
      --auth oauth \
      --oauth-scope docs.read \
      --timeout 20 \
      --connect-timeout 5 \
      --include 'search,read_*'
    openclaw mcp doctor docs --probe
    ```

    Uzak sunucu destekliyorsa OAuth kullanın. Sunucu statik üstbilgiler gerektiriyorsa değişmez taşıyıcı belirteçlerini depoya kaydetmekten kaçının.

  </Tab>
  <Tab title="Masaüstü/CUA">
    ```bash
    openclaw mcp set cua-driver '{"command":"cua-driver","args":["mcp"]}'
    openclaw mcp tools cua-driver --include 'list_apps,get_window_state,click,type_text'
    openclaw mcp doctor cua-driver --probe
    ```

    Doğrudan masaüstü denetim sunucuları, başlattıkları işlemin izinlerini devralır. Dar kapsamlı araç filtreleri ve işletim sistemi düzeyindeki izin istemlerini kullanın.

  </Tab>
</Tabs>

### JSON çıktı biçimleri

Betikler ve panolar için `--json` kullanın. Alan kümeleri zamanla büyüyebileceğinden tüketiciler bilinmeyen anahtarları yok saymalıdır.

<AccordionGroup>
  <Accordion title="status --json">
    ```json
    {
      "path": "/home/user/.openclaw/openclaw.json",
      "servers": [
        {
          "name": "docs",
          "configured": true,
          "enabled": true,
          "ok": true,
          "transport": "streamable-http",
          "launch": "streamable-http https://mcp.example.com/mcp",
          "auth": "oauth",
          "authStatus": {
            "hasTokens": true,
            "requiresAuthorization": false,
            "hasClientInformation": true,
            "hasCodeVerifier": false,
            "hasDiscoveryState": true,
            "hasLastAuthorizationUrl": false
          },
          "requestTimeoutMs": 20000,
          "connectionTimeoutMs": 5000,
          "toolFilter": {
            "include": ["search", "read_*"],
            "exclude": []
          },
          "supportsParallelToolCalls": true
        }
      ]
    }
    ```
  </Accordion>
  <Accordion title="doctor --json">
    ```json
    {
      "ok": true,
      "path": "/home/user/.openclaw/openclaw.json",
      "servers": [
        {
          "name": "docs",
          "ok": true,
          "issues": [
            {
              "level": "warning",
              "message": "OAuth kimlik bilgileri yetkilendirilmemiş; openclaw mcp login docs komutunu çalıştırın"
            }
          ]
        }
      ]
    }
    ```

    Etkinleştirilmiş ve denetlenmiş herhangi bir sunucuda `error` düzeyinde bir sorun olduğunda `doctor --json` sıfırdan farklı bir kodla çıkar. `warning` ve `info` sorunları bildirilir ancak tek başlarına komutun başarısız olmasına neden olmaz.

  </Accordion>
  <Accordion title="probe --json">
    ```json
    {
      "generatedAt": "2026-05-31T09:00:00.000Z",
      "servers": {
        "docs": {
          "launch": "streamable-http https://mcp.example.com/mcp",
          "tools": 2,
          "resources": true,
          "listChanged": {
            "tools": true,
            "resources": false,
            "prompts": false
          }
        }
      },
      "tools": ["docs__read_page", "docs__search"],
      "diagnostics": []
    }
    ```

    `probe --json` canlı bir MCP istemci oturumu açar ve sonucunu doğrudan yazdırır; `status`/`doctor` aksine çıktıda üst düzey bir `path` alanı yoktur. `resources` ve `prompts` anahtarları yalnızca sunucu gerçekten bu yeteneği duyurduğunda bulunur (istemleri olmayan bir sunucu, `false` bildirmek yerine `prompts` anahtarını çıkarır). `probe` öğesini statik yapılandırma denetimleri için değil, erişilebilirlik ve yetenek kanıtı için kullanın.

  </Accordion>
</AccordionGroup>

Örnek yapılandırma biçimi:

```json
{
  "mcp": {
    "servers": {
      "context7": {
        "command": "uvx",
        "args": ["context7-mcp"]
      },
      "docs": {
        "url": "https://mcp.example.com",
        "transport": "streamable-http",
        "requestTimeoutMs": 20000,
        "connectionTimeoutMs": 5000,
        "supportsParallelToolCalls": true,
        "auth": "oauth",
        "oauth": {
          "scope": "docs.read"
        },
        "sslVerify": true,
        "clientCert": "/path/to/client.crt",
        "clientKey": "/path/to/client.key",
        "toolFilter": {
          "include": ["search_*"],
          "exclude": ["admin_*"]
        }
      }
    }
  }
}
```

### Stdio aktarımı

Yerel bir alt işlem başlatır ve stdin/stdout üzerinden iletişim kurar.

| Alan                       | Açıklama                              |
| -------------------------- | ------------------------------------- |
| `command`         | Başlatılacak yürütülebilir dosya (zorunlu) |
| `args`         | Komut satırı bağımsız değişkenleri dizisi |
| `env`         | Ek ortam değişkenleri                 |
| `cwd` / `workingDirectory` | İşlemin çalışma dizini       |

<Warning>
**Stdio ortam güvenliği filtresi**

OpenClaw, bir stdio MCP sunucusunu başlatmadan önce sunucunun `env` bloğunda yer alsalar bile yorumlayıcı başlatma, yükleyici ele geçirme ve kabuk başlatma ortam anahtarlarını reddeder. Bu işlem, OpenClaw tarafından başlatılan diğer işlemlerle aynı ana makine ortamı güvenlik politikasını kullanır: bilinen yorumlayıcı başlatma kancalarını (örneğin `NODE_OPTIONS`, `PYTHONSTARTUP`, `PERL5OPT`, `RUBYOPT`, `BASHOPTS`, `KSH_ENV`), paylaşılan kitaplık ve işlev ekleme öneklerini (`DYLD_*`, `LD_*`, `BASH_FUNC_*`) ve benzer çalışma zamanı denetim değişkenlerini engeller. Başlatma sırasında bunlar sessizce kaldırılır ve örtük bir ön bölüm ekleyememeleri, yorumlayıcıyı değiştirememeleri, hata ayıklayıcıyı etkinleştirememeleri veya stdio işlemine karşı dinamik bağlayıcıyı ele geçirememeleri için bir uyarı günlüğe kaydedilir. Açık bir izin listesi, sıradan MCP kimlik bilgisi ortam değişkenlerinin kullanılabilir kalmasını sağlar (`GITHUB_TOKEN`, `GH_TOKEN`, `GITLAB_TOKEN`, `NPM_TOKEN`, `NODE_AUTH_TOKEN`, `DATABASE_URL`, `MONGODB_URI`, `REDIS_URL`, `AMQP_URL`, `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, `AWS_SESSION_TOKEN`, `AZURE_CLIENT_ID`, `AZURE_CLIENT_SECRET`); sıradan proxy ve sunucuya özgü ortam değişkenleri de buna dahildir (`HTTP_PROXY`, özel `*_API_KEY` vb.). `AWS_CONFIG_FILE` ve `AWS_SHARED_CREDENTIALS_FILE` gibi diğer `AWS_*` anahtarları, kimlik bilgisi değerini doğrudan taşımak yerine kimlik bilgisi dosyalarına işaret ettikleri için engellenmeye devam eder.

MCP sunucunuzun engellenen değişkenlerden birine gerçekten ihtiyacı varsa bunu stdio sunucusunun `env` öğesi altında değil, gateway ana makine işleminde ayarlayın.
</Warning>

### SSE / HTTP aktarımı

HTTP Sunucu Gönderimli Olaylar üzerinden uzak bir MCP sunucusuna bağlanır.

| Alan                        | Açıklama                                                               |
| --------------------------- | ---------------------------------------------------------------------- |
| `url`          | Uzak sunucunun HTTP veya HTTPS URL'si (zorunlu)                         |
| `headers`          | İsteğe bağlı HTTP üstbilgileri anahtar-değer eşlemesi (örneğin kimlik doğrulama belirteçleri) |
| `connectionTimeoutMs`          | Sunucu başına bağlantı zaman aşımı, ms cinsinden (isteğe bağlı)         |
| `requestTimeoutMs`          | Sunucu başına MCP isteği zaman aşımı, milisaniye cinsinden              |
| `auth: "oauth"`          | `openclaw mcp login` tarafından kaydedilen MCP OAuth kimlik bilgilerini kullanır |
| `sslVerify`          | Yalnızca açıkça güvenilen özel HTTPS uç noktaları için false olarak ayarlayın |
| `clientCert` / `clientKey` | mTLS istemci sertifikası ve anahtar yolları              |
| `supportsParallelToolCalls`          | Bu sunucu için eşzamanlı çağrıların güvenli olduğuna dair ipucu         |

Örnek:

```json
{
  "mcp": {
    "servers": {
      "remote-tools": {
        "url": "https://mcp.example.com",
        "auth": "oauth",
        "requestTimeoutMs": 20000,
        "headers": {
          "Authorization": "Bearer <token>"
        }
      }
    }
  }
}
```

`url` (kullanıcı bilgileri) ve `headers` içindeki hassas değerler günlüklerde ve durum çıktısında gizlenir. `openclaw mcp doctor`, hassas görünümlü `headers` veya `env` girdileri değişmez değerler içerdiğinde uyarır; böylece operatörler bu değerleri depoya kaydedilmiş yapılandırmanın dışına taşıyabilir.

### OAuth iş akışı

OAuth, MCP OAuth akışını duyuran HTTP MCP sunucuları içindir. `auth: "oauth"` etkinleştirildiğinde sunucu için statik `Authorization` üstbilgileri yok sayılır. `openclaw mcp login` tarafından kaydedilen kimlik bilgileri; gömülü MCP, CLI çalıştırıcıları ve yerel Codex uygulama sunucusuyla çalışır.

Yerel MCP OAuth oturumları, `<state-dir>/state/openclaw.sqlite` konumundaki yalnızca sahibinin erişebildiği paylaşılan SQLite veritabanında (`mcp_oauth_stores`) bulunur. Satır; erişim ve yenileme belirteçlerini, dinamik istemci kaydı sırlarını, keşif meta verilerini ve geçici PKCE doğrulayıcısını içerebilir. Yenileme, oturum açma ve oturumu kapatma aynı SQLite kirasını kullanır; böylece paralel OpenClaw işlemleri tek bir yenileme belirtecini tüketemez veya oturumu kapatılmış bir oturumu yeniden etkinleştiremez.

Kullanımdan kaldırılmış `<state-dir>/mcp-oauth/*.json` deposundan yükseltmeler yalnızca `openclaw doctor --fix` tarafından gerçekleştirilir. Çalışma zamanı kodu bu dosyaları hiçbir zaman okumaz, yazmaz veya bunlara geri dönmez.

Kimlik bilgileri kullanılabilir olana kadar OpenClaw, aracı turunu başarısız kılmak yerine yalnızca ilgili MCP sunucusunu aracı çalışma zamanından çıkarır. Ardından operatör veya kabuk erişimi olan bir aracı `openclaw mcp login <name>` komutunu çalıştırabilir ve sonraki bir turda sunucuyu kullanabilir.

Bir sunucu belirteci `insufficient_scope` ile reddederse OpenClaw, istenen kapsamı korur ve yeni kapsam veremeyecek bir yenilemeyi tekrarlamak yerine `openclaw mcp login <name>` ister. Bu oturum açma işlemi, yerine geçecek kimlik bilgileri kaydedilene kadar önceki belirteci koruyarak yeni bir yetkilendirme isteği başlatır.

Uzak bir MCP hizmeti zaten yenileme özellikli ayrı bir OpenClaw kimlik doğrulama profiliyle destekleniyorsa isteğe bağlı olarak `oauth.authProfileId` ayarlanabilir. OpenClaw, çalışma zamanı yansıtmasından önce iki kimlik bilgisi kaynağından birini yeniler ve aşağı akış MCP istemcisine yalnızca geçerli erişim belirtecini iletir.

<Steps>
  <Step title="Sunucuyu kaydedin">
    Sunucuyu `auth: "oauth"` ve isteğe bağlı OAuth meta verileriyle ekleyin veya güncelleyin.

    ```bash
    openclaw mcp set docs '{"url":"https://mcp.example.com/mcp","transport":"streamable-http","auth":"oauth","oauth":{"scope":"docs.read"}}'
    ```

    Kimlik doğrulama profiliyle desteklenen bearer için profil bağlamasını kaydedin:

    ```bash
    openclaw mcp set docs '{"url":"https://mcp.example.com/mcp","transport":"streamable-http","auth":"oauth","oauth":{"authProfileId":"docs:mcp"}}'
    ```

  </Step>
  <Step title="Oturum açmayı başlatın">
    Yetkilendirme isteğini oluşturmak için oturum açma komutunu çalıştırın.

    ```bash
    openclaw mcp login docs
    ```

    OpenClaw, yetkilendirme URL'sini yazdırır ve geçici OAuth doğrulayıcı durumunu paylaşılan SQLite'ta saklar.

  </Step>
  <Step title="Kodla tamamlayın">
    Tarayıcıda onayladıktan sonra döndürülen kodu OpenClaw'a geri iletin.

    ```bash
    openclaw mcp login docs --code abc123
    ```

  </Step>
  <Step title="Yetkilendirmeyi kontrol edin">
    Tokenların mevcut olduğunu ve ek yetkilendirme gerektirmediğini doğrulamak için durum veya doctor komutunu kullanın. Durum `authorization-required` bildirirse veya doctor ek yetkilendirme isterse `openclaw mcp login <name>` komutunu yeniden çalıştırın.

    ```bash
    openclaw mcp status --verbose
    openclaw mcp doctor docs --probe
    ```

  </Step>
  <Step title="Kimlik bilgilerini temizleyin">
    Oturumu kapatma, saklanan OAuth kimlik bilgilerini kaldırır ancak kaydedilmiş sunucu tanımını korur.

    ```bash
    openclaw mcp logout docs
    ```

  </Step>
</Steps>

Sağlayıcı tokenları döndürürse veya yetkilendirme durumu takılı kalırsa `openclaw mcp logout <name>` komutunu çalıştırın, ardından `login` işlemini tekrarlayın. Sunucu adı ve URL kimlik bilgisi deposu girdisini tanımlamaya devam ettiği sürece `logout`, `auth: "oauth"` yapılandırmadan kaldırıldıktan sonra bile kaydedilmiş bir HTTP sunucusunun kimlik bilgilerini temizleyebilir.

### Akış özellikli HTTP aktarımı

`streamable-http`, `sse` ve `stdio` seçeneklerinin yanında ek bir aktarım seçeneğidir. Uzak MCP sunucularıyla çift yönlü iletişim için HTTP akışını kullanır.

| Alan                        | Açıklama                                                                               |
| --------------------------- | -------------------------------------------------------------------------------------- |
| `url`                       | Uzak sunucunun HTTP veya HTTPS URL'si (zorunlu)                                        |
| `transport`                 | Bu aktarımı seçmek için `"streamable-http"` olarak ayarlayın; belirtilmediğinde OpenClaw `sse` kullanır |
| `headers`                   | HTTP üst bilgilerinin isteğe bağlı anahtar-değer eşlemesi (örneğin kimlik doğrulama tokenları) |
| `connectionTimeoutMs`       | Sunucu başına bağlantı zaman aşımı, ms cinsinden (isteğe bağlı)                        |
| `requestTimeoutMs`          | Sunucu başına MCP isteği zaman aşımı, milisaniye cinsinden                             |
| `auth: "oauth"`             | `openclaw mcp login` tarafından kaydedilen MCP OAuth kimlik bilgilerini kullanır       |
| `sslVerify`                 | Yalnızca açıkça güvenilen özel HTTPS uç noktaları için false olarak ayarlayın          |
| `clientCert` / `clientKey`  | mTLS istemci sertifikası ve anahtar yolları                                             |
| `supportsParallelToolCalls` | Bu sunucu için eşzamanlı çağrıların güvenli olduğuna ilişkin ipucu                      |

OpenClaw yapılandırması, kurallı yazım olarak `transport: "streamable-http"` kullanır. CLI'ye özgü MCP `type: "http"` değerleri, `openclaw mcp set` aracılığıyla kaydedildiğinde kabul edilir ve mevcut yapılandırmada `openclaw doctor --fix` tarafından düzeltilir; ancak yerleşik OpenClaw'ın doğrudan tükettiği değer `transport` değeridir.

Örnek:

```json
{
  "mcp": {
    "servers": {
      "streaming-tools": {
        "url": "https://mcp.example.com/stream",
        "transport": "streamable-http",
        "connectionTimeoutMs": 10000,
        "requestTimeoutMs": 30000,
        "headers": {
          "Authorization": "Bearer <token>"
        }
      }
    }
  }
}
```

<Note>
Kayıt defteri komutları kanal köprüsünü başlatmaz. Hedef sunucuya erişilebildiğini kanıtlamak için yalnızca `probe` ve `doctor --probe` canlı bir MCP istemci oturumu açar.
</Note>

## Kontrol kullanıcı arayüzü

Tarayıcıdaki Kontrol kullanıcı arayüzü, `/settings/mcp` konumunda özel bir MCP ayarları sayfası içerir; önceki `/mcp` yolu takma ad olarak kalır. Sayfa; yapılandırılmış sunucu sayılarını, etkin/OAuth/filtre özetlerini, sunucu başına aktarım satırlarını, etkinleştirme/devre dışı bırakma denetimlerini, yaygın CLI komutlarını ve `mcp` yapılandırma bölümü için kapsamlı bir düzenleyiciyi gösterir.

Sayfayı operatör düzenlemeleri ve hızlı envanter için kullanın. Canlı sunucu kanıtına ihtiyaç duyduğunuzda `openclaw mcp doctor --probe` veya `openclaw mcp probe` kullanın.

Operatör iş akışı:

1. Kontrol kullanıcı arayüzünü açın ve **MCP** seçeneğini belirleyin.
2. Toplam, etkin, OAuth ve filtrelenmiş sunucular için özet kartlarını inceleyin.
3. Aktarım, kimlik doğrulama, filtre, zaman aşımı ve komut ipuçları için her sunucu satırını kullanın.
4. Bir tanımı korumak ancak çalışma zamanı keşfinden hariç tutmak istediğinizde etkinleştirme durumunu değiştirin.
5. Yeni sunucular, üst bilgiler, TLS, OAuth meta verileri veya araç filtreleri gibi yapısal değişiklikler için kapsamlı `mcp` yapılandırma bölümünü düzenleyin.
6. Yalnızca yapılandırmayı kalıcılaştırmak için **Save**, Gateway yapılandırma yolu üzerinden uygulamak için **Save & Publish** seçeneğini belirleyin.
7. Düzenlenen sunucunun başlatıldığına ve araçları listelediğine dair canlı kanıt gerektiğinde `openclaw mcp doctor --probe` komutunu çalıştırın.

Notlar:

- komut parçacıkları, sıra dışı adların kabukta kopyalanabilir kalması için sunucu adlarını tırnak içine alır
- görüntülenen URL benzeri değerler, gömülü kimlik bilgileri içerdiğinde işlenmeden önce karartılır
- sayfa MCP aktarımlarını kendi başına başlatmaz
- MCP istemcilerinin sahibi olan sürece bağlı olarak etkin çalışma zamanları `openclaw mcp reload`, Gateway yapılandırmasının yayımlanması veya işlemin yeniden başlatılmasını gerektirebilir

## MCP Uygulamaları

OpenClaw, kararlı [MCP Uygulamaları uzantısını](https://modelcontextprotocol.io/extensions/apps) uygulayan araçları işleyebilir. Uygulamalar isteğe bağlıdır; çünkü HTML'leri yapılandırılmış MCP sunucusundan gelir ve aynı sunucudan uygulama tarafından görülebilen araçları veya kaynakları isteyebilir.

Ana makine köprüsünü etkinleştirin:

```bash
openclaw config set mcp.apps.enabled true --strict-json
```

Bu ayarı değiştirdikten sonra Gateway'i yeniden başlatın. Etkinleştirildiğinde OpenClaw, Gateway portunun bir fazlasında yalnızca korumalı alana yönelik bir HTTP(S) dinleyicisi başlatır (varsayılan Gateway için `18790`). Kontrol kullanıcı arayüzü Uygulamaları bu ayrı kaynaktan yükler; dinleyici hiçbir zaman Kontrol kullanıcı arayüzünü, kimliği doğrulanmış Gateway rotalarını veya kullanıcı verilerini sunmaz.

Doğrudan Gateway bağlantılarının her iki porta da erişmesi gerekir. Bir ters proxy veya TLS sonlandırıcısı Kontrol kullanıcı arayüzünü açığa çıkarıyorsa Uygulamalara özel bir genel kaynak verin ve yalnızca bu kaynağı korumalı alan dinleyicisine yönlendirin:

```json5
{
  mcp: {
    apps: {
      enabled: true,
      sandboxOrigin: "https://mcp-apps.example.com",
      sandboxPort: 18790,
    },
  },
}
```

Korumalı alan kaynağı, Kontrol kullanıcı arayüzü kaynağından farklı olmalıdır. Üzerinde kimliği doğrulanmış veya hassas başka içerikler barındırmayın.

Örneğin, resmî temel React demosu şu şekilde yapılandırılabilir:

```json5
{
  mcp: {
    apps: { enabled: true },
    servers: {
      "basic-react": {
        command: "npx",
        args: ["-y", "@modelcontextprotocol/server-basic-react", "--stdio"],
      },
    },
  },
}
```

Davranış ve güvenlik sınırları:

- OpenClaw, `io.modelcontextprotocol/ui` uzantısını yalnızca Uygulamalar etkinleştirildiğinde duyurur.
- Yalnızca tam olarak `text/html;profile=mcp-app` MIME türüne sahip `ui://` kaynakları işlenir.
- Kullanıcı arayüzü kaynakları 2 MiB ile sınırlandırılır, özel bir dış kaynakta çift iframe proxy'sinin arkasına yerleştirilir, opak bir iç Uygulama kaynağına yüklenir ve kaynak meta verilerinden türetilen CSP ile kısıtlanır.
- Yalnızca Uygulamaya özgü araçlar (`_meta.ui.visibility: ["app"]`) model araç listelerinin dışında kalır. Uygulamalar yalnızca kendi sunucularındaki, uygulama tarafından görülebilen ve görünümü oluşturan çalıştırma için geçerli OpenClaw araç politikasından da geçen araçları çağırabilir.
- İç Uygulama belgeleri, Uygulamalar arası yalıtım için opak kaynaklar kullandığı sürece kamera, mikrofon ve coğrafi konum gibi kaynağa bağlı Uygulama izinleri verilmez.
- Uygulama HTML'si, eksiksiz araç bağımsız değişkenleri ve ham sonuçlar, on dakikalık sınırlı bir bellek içi görünüm kiralamasında tutulur ve diske yazılmaz veya transkript önizleme meta verilerine kopyalanmaz. Transkript yalnızca özgün araç çağrısı kimliğine bağlı, sınırlı bir sunucu/araç/kaynak tanımlayıcısı saklar. Gateway yeniden başlatıldıktan sonra Kontrol kullanıcı arayüzü bu tanımlayıcıyı kimliği doğrulanmış oturum transkriptine göre doğrulayabilir ve `ui://` kaynağını yeniden getirebilir; yeni bir çalıştırma güncel araç izinlerini belirleyene kadar yeniden oluşturulan görünümler salt okunurdur.
- Kanal konuşmalarında, bir turdaki en son başarılı Uygulama görünümü son asistan yanıtına **Uygulamayı Aç** tarzı tek bir eylem ekler. Telegram DM'leri yerel bir Mini App düğmesi kullanır; Slack ve Discord aynı taşınabilir eylemi bağlantı olarak işler. Diğer kanallar özgün yanıt metnini korur ve anlaşılır bir HTTPS bağlantısı ekler.
- Kanal başlatma bağlantıları yalnızca Gateway Tailscale erişimi yayımlanmış bir HTTPS kaynağı hazırladığında kullanılabilir. `gateway.tailscale.mode: "serve"` yalnızca tailnet üzerinden erişilebilir; `"funnel"` genel internetten erişilebilir. `gateway.tailscale.preserveFunnel` tarafından korunan, haricî olarak yönetilen bir Funnel da internetten erişilebilir kabul edilir. Bkz. [Tailscale](/tr/gateway/tailscale).
- Başlatma biletleri opaktır, yalnızca son kanal yanıtı oluşturulurken üretilir ve en fazla iki dakika sonra ya da temel görünüm kiralamasının süresi dolduğunda (hangisi önce gerçekleşirse) sona erer. URL; Gateway bearer kimlik bilgilerini, oturum anahtarlarını, görünüm meta verilerini, Uygulama HTML'sini, araç girdisini veya araç sonuçlarını içermez.
- Yayımlanmış bir kaynak veya bilet kapasitesi yoksa, görünümün ya da biletin süresi dolmuşsa veya aktarım yerel denetimleri işleyemiyorsa özgün asistan metni kullanılabilir kalır. Kontrol kullanıcı arayüzü mevcut satır içi Uygulama tuvalini korur ve yinelenen bir başlatma eylemi almaz.
- `openclaw security audit`, köprü etkinken uyarı verir. Gerekli olmadığında `openclaw config set mcp.apps.enabled false --strict-json` ile devre dışı bırakın.

## Mevcut sınırlar

Bu sayfa, köprüyü bugün yayımlandığı hâliyle belgeler.

Mevcut sınırlar:

- konuşma keşfi, mevcut Gateway oturum rotası meta verilerine bağlıdır
- Claude'a özgü bağdaştırıcının ötesinde genel bir gönderim protokolü yoktur
- henüz mesaj düzenleme veya tepki araçları yoktur
- HTTP/SSE/streamable-http aktarımı tek bir uzak sunucuya bağlanır; henüz çoğullanmış üst akış yoktur
- `permissions_list_open` yalnızca köprü bağlıyken gözlemlenen onayları içerir

## İlgili

- [CLI referansı](/tr/cli)
- [Pluginler](/tr/cli/plugins)
