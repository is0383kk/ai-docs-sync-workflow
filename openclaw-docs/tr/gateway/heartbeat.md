---
read_when:
    - Heartbeat sıklığını veya mesajlaşmayı ayarlama
    - Zamanlanmış görevler için Heartbeat ile Cron arasında karar verme
sidebarTitle: Heartbeat
summary: Heartbeat yoklama mesajları ve bildirim kuralları
title: Heartbeat
x-i18n:
    generated_at: "2026-07-26T23:41:15Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 44c78e797987d8dccab910cd82fc1f482df86afce40677846d8f26522d33f6fa
    source_path: gateway/heartbeat.md
    workflow: 16
---

<Note>
**Heartbeat mı, cron mu?** Her birinin ne zaman kullanılacağına ilişkin rehberlik için [Otomasyon](/tr/automation) bölümüne bakın.
</Note>

Heartbeat, modelin sizi ileti yağmuruna tutmadan ilgilenilmesi gereken her şeyi ortaya çıkarabilmesi için ana oturumda **düzenli aralıklarla agent turları** çalıştırır.

Heartbeat, zamanlanmış bir ana oturum turudur; [arka plan görevi](/tr/automation/tasks) kaydı **oluşturmaz**. Görev kayıtları, ayrılmış çalışmalar (ACP çalıştırmaları, alt agent'lar, yalıtılmış cron işleri) içindir.

Arka planda Heartbeat sıklığı cron zamanlayıcısı tarafından yönetilir: Gateway, Heartbeat'in etkin olduğu her agent için sisteme ait bir cron işi tutar (`openclaw cron list --all` içinde `Heartbeat (agent-id)` olarak görünür). Heartbeat yapılandırması istenen durum girdisi olarak kalırken kalıcı izleyici zamanlaması gerçek tetiklemeyi ve çalıştırıcının sonraki bekleme süresini yönetir. Gateway, yapılandırma değişikliklerini başlangıçta ve yapılandırma yeniden yüklendiğinde uygular; `openclaw doctor --fix`, bir sonraki Gateway başlangıcından önce eksik veya güncelliğini yitirmiş izleyici satırlarını oluşturabilir. Cron işini değil, `agents.*.heartbeat` öğesini düzenleyin.

Zamanlanmış Heartbeat'ler cron gerektirir. `cron.enabled`, `false` veya `OPENCLAW_SKIP_CRON=1` olduğunda Gateway başlangıçta bir uyarı kaydeder ve zamanlanmış Heartbeat'leri çalıştırmaz; elle ve olaylarla tetiklenen Heartbeat uyandırmaları kullanılabilir durumda kalır. Ayrı bir Heartbeat yedek zamanlayıcısı yoktur.

Sorun giderme: [Zamanlanmış Görevler](/tr/automation/cron-jobs#troubleshooting)

## Hızlı başlangıç (yeni başlayanlar)

<Steps>
  <Step title="Bir sıklık seçin">
    Heartbeat'leri etkin bırakın (varsayılan `30m`; Claude CLI'ın yeniden kullanımı dâhil olmak üzere Anthropic OAuth/token kimlik doğrulaması yapılandırıldığında `1h`) veya kendi sıklığınızı ayarlayın.
  </Step>
  <Step title="İzleyici notları ekleyin (isteğe bağlı)">
    `openclaw cron scratch <jobId> --set "..."` ile Heartbeat izleyicisinin notlarına kısa bir kontrol listesi kaydedin.
  </Step>
  <Step title="Heartbeat iletilerinin nereye gideceğine karar verin">
    Varsayılan `target: "none"` değeridir; son kişiye yönlendirmek için `target: "last"` değerini ayarlayın.
  </Step>
  <Step title="İsteğe bağlı ayarlamalar">
    - Heartbeat çalıştırmalarında yalnızca izleyici notları gerekiyorsa hafif başlangıç bağlamını kullanın.
    - Her Heartbeat'te tüm konuşma geçmişini göndermemek için yalıtılmış oturumları etkinleştirin.
    - Heartbeat'leri etkin saatlerle (yerel saat) sınırlandırın.

  </Step>
</Steps>

Örnek yapılandırma:

```json5
{
  agents: {
    defaults: {
      heartbeat: {
        every: "30m",
        target: "last", // son kişiye açıkça teslim et (varsayılan "none")
        directPolicy: "allow", // varsayılan: doğrudan/DM hedeflerine izin ver; engellemek için "block" olarak ayarla
        lightContext: true, // isteğe bağlı: Heartbeat çalıştırmalarında çalışma alanı başlangıç dosyalarını atla
        isolatedSession: true, // isteğe bağlı: her çalıştırmada yeni oturum (konuşma geçmişi yok)
        // activeHours: { start: "08:00", end: "24:00" },
      },
    },
  },
}
```

## Varsayılanlar

- Aralık: `30m`. Anthropic sağlayıcı varsayılanlarının uygulanması, çözümlenen kimlik doğrulama modu OAuth/token olduğunda (Claude CLI'ın yeniden kullanımı dâhil) bunu `1h` değerine yükseltir, ancak yalnızca `heartbeat.every` ayarlanmamışsa. `agents.defaults.heartbeat.every` veya agent başına `agents.entries.*.heartbeat.every` değerini ayarlayın; devre dışı bırakmak için `0m` kullanın.
- İstem gövdesi (`agents.defaults.heartbeat.prompt` aracılığıyla yapılandırılabilir): `Follow the heartbeat monitor scratch context when provided. Recurring tasks are cron jobs; create or change their schedules with cron tools or the openclaw cron CLI, not heartbeat scratch. Do not infer or repeat old tasks from prior chats. If nothing needs attention, reply HEARTBEAT_OK.`
- Zaman aşımı: zaman aşımı ayarlanmamış Heartbeat turları, `agents.defaults.timeoutSeconds` ayarlandığında bu değeri kullanır. Aksi takdirde 600 saniyeyle sınırlandırılmış Heartbeat sıklığını kullanırlar. Daha uzun Heartbeat çalışmaları için `agents.defaults.heartbeat.timeoutSeconds` veya agent başına `agents.entries.*.heartbeat.timeoutSeconds` değerini ayarlayın.
- Heartbeat istemi, kullanıcı iletisi olarak **aynen** gönderilir. Varsayılan agent için Heartbeat'ler etkinleştirildiğinde sistem istemi bir "Heartbeat'ler" bölümü içerir ve çalıştırma dâhili olarak işaretlenir.
- Heartbeat'ler `0m` ile devre dışı bırakıldığında izleyici cron işi kalır ancak devre dışı olur ve sıklığı yeniden etkinleştirdiğinizde kullanılmak üzere notları korunur.
- Cron'ın kendisi devre dışı bırakıldığında Heartbeat sıklığı etkin kalmış olsa bile zamanlanmış Heartbeat'ler çalışmaz.
- Etkin saatler (`heartbeat.activeHours`) yapılandırılmış saat diliminde denetlenir. Pencerenin dışında Heartbeat'ler, pencere içindeki bir sonraki tetiklemeye kadar atlanır.
- Cron çalışması etkin veya kuyruktayken ya da söz konusu agent'ın oturum anahtarlı alt agent veya iç içe komut hatları meşgulken Heartbeat'ler otomatik olarak ertelenir. Eşdüzey agent'lar birbirini duraklatmaz.

## Heartbeat isteminin amacı

Varsayılan istem kasıtlı olarak geniş kapsamlıdır:

- **Arka plan görevleri**: "Bekleyen görevleri göz önünde bulundurun" ifadesi, agent'ı takip işlerini (gelen kutusu, takvim, anımsatıcılar, kuyruktaki çalışmalar) incelemeye ve acil olan her şeyi belirtmeye yönlendirir.
- **Kullanıcıyla durum kontrolü**: "Gündüzleri bazen kullanıcınızın durumunu kontrol edin" ifadesi, ara sıra kısa bir "bir şeye ihtiyacınız var mı?" iletisini teşvik eder ancak yapılandırılmış yerel saat diliminizi kullanarak gece ileti yağmurunu önler (bkz. [Saat Dilimi](/tr/concepts/timezone)).

Heartbeat, tamamlanan [arka plan görevlerine](/tr/automation/tasks) tepki verebilir ancak Heartbeat çalıştırmasının kendisi bir görev kaydı oluşturmaz.

Heartbeat'in çok belirli bir şey yapmasını istiyorsanız (ör. "Gmail PubSub istatistiklerini denetle" veya "Gateway durumunu doğrula"), `agents.defaults.heartbeat.prompt` (veya `agents.entries.*.heartbeat.prompt`) değerini özel bir gövdeye ayarlayın (aynen gönderilir).

## Yanıt sözleşmesi

- İlgilenilmesi gereken bir şey yoksa **`HEARTBEAT_OK`** ile yanıt verin.
- Heartbeat çalıştırmaları bunun yerine görünür bir güncelleme olmaması için `notify: false` ile `heartbeat_respond` çağrısı veya uyarı için `notificationText` ile `notify: true` çağrısı yapabilir. Yapılandırılmış araç yanıtı mevcut olduğunda metin yedeğine göre önceliklidir.
- `notify: false` içeren anlamlı bir `heartbeat_respond` sonucu sessiz kalır ancak söz konusu oturumdaki bir sonraki kullanıcı turu için sınırlı dâhili bağlam olarak hatırlanır. `no_change` onayları ve görünür bildirimler bu şekilde depolanmaz.
- Heartbeat çalıştırmaları sırasında OpenClaw, yanıtın **başında veya sonunda** görünen `HEARTBEAT_OK` değerini onay olarak kabul eder. Belirteç kaldırılır ve kalan içerik en fazla 300 karakterse yanıt bırakılır.
- `HEARTBEAT_OK` bir yanıtın **ortasında** görünürse özel olarak işlenmez.
- Uyarılar için `HEARTBEAT_OK` değerini **eklemeyin**; yalnızca uyarı metnini döndürün.

Heartbeat'lerin dışında, bir iletinin başındaki/sonundaki ilgisiz `HEARTBEAT_OK` kaldırılır ve kaydedilir; yalnızca `HEARTBEAT_OK` içeren bir ileti bırakılır.

## Yapılandırma

```json5
{
  agents: {
    defaults: {
      heartbeat: {
        every: "30m", // varsayılan: 30m (0m devre dışı bırakır)
        model: "anthropic/claude-opus-4-6",
        lightContext: false, // varsayılan: false; true, Heartbeat çalıştırmalarında çalışma alanı başlangıç dosyalarını atlar
        isolatedSession: false, // varsayılan: false; true, her Heartbeat'i yeni bir oturumda çalıştırır (konuşma geçmişi yok)
        target: "last", // varsayılan: none | seçenekler: last | none | <channel id> (çekirdek veya Plugin, ör. "imessage")
        to: "+15551234567", // isteğe bağlı kanala özgü geçersiz kılma
        accountId: "ops-bot", // isteğe bağlı çok hesaplı kanal kimliği
        prompt: "Sağlandığında Heartbeat izleyicisinin not bağlamını izleyin. Yinelenen görevler cron işleridir; bunların zamanlamalarını Heartbeat notlarıyla değil, cron araçları veya openclaw cron CLI ile oluşturun ya da değiştirin. Önceki sohbetlerdeki eski görevleri çıkarsamayın veya yinelemeyin. İlgilenilmesi gereken bir şey yoksa HEARTBEAT_OK yanıtını verin.",
      },
    },
  },
}
```

### Kapsam ve öncelik

- `agents.defaults.heartbeat` genel Heartbeat davranışını ayarlar.
- `agents.entries.*.heartbeat` bunun üzerine birleştirilir; herhangi bir agent'ın `heartbeat` bloğu varsa Heartbeat'leri **yalnızca bu agent'lar** çalıştırır.
- `channels.defaults.heartbeatVisibility` tüm kanallar için görünürlük varsayılanlarını ayarlar.
- `channels.<channel>.heartbeatVisibility` kanal varsayılanlarını geçersiz kılar.
- `channels.<channel>.accounts.<id>.heartbeatVisibility` (çok hesaplı kanallar) kanal başına ayarları geçersiz kılar.

### Agent başına Heartbeat'ler

Herhangi bir `agents.entries.*` girdisi bir `heartbeat` bloğu içeriyorsa Heartbeat'leri **yalnızca bu agent'lar** çalıştırır. Agent başına blok, `agents.defaults.heartbeat` üzerine birleştirilir (böylece paylaşılan varsayılanları bir kez ayarlayıp agent başına geçersiz kılabilirsiniz).

Örnek: iki agent; yalnızca ikinci agent Heartbeat'leri çalıştırır.

```json5
{
  agents: {
    defaults: {
      heartbeat: {
        every: "30m",
        target: "last", // son kişiye açıkça teslim et (varsayılan "none")
      },
    },
    list: [
      { id: "main", default: true },
      {
        id: "ops",
        heartbeat: {
          every: "1h",
          target: "whatsapp",
          to: "+15551234567",
          timeoutSeconds: 45,
          prompt: "Sağlandığında Heartbeat izleyicisinin not bağlamını izleyin. Yinelenen görevler cron işleridir; bunların zamanlamalarını Heartbeat notlarıyla değil, cron araçları veya openclaw cron CLI ile oluşturun ya da değiştirin. Önceki sohbetlerdeki eski görevleri çıkarsamayın veya yinelemeyin. İlgilenilmesi gereken bir şey yoksa HEARTBEAT_OK yanıtını verin.",
        },
      },
    ],
  },
}
```

### Etkin saatler örneği

Heartbeat'leri belirli bir saat dilimindeki çalışma saatleriyle sınırlandırın:

```json5
{
  agents: {
    defaults: {
      heartbeat: {
        every: "30m",
        target: "last", // son kişiye açıkça teslim et (varsayılan "none")
        activeHours: {
          start: "09:00",
          end: "22:00",
          timezone: "America/New_York", // isteğe bağlı; ayarlanmışsa userTimezone değerinizi, aksi takdirde ana makinenin saat dilimini kullanır
        },
      },
    },
  },
}
```

Bu pencerenin dışında (Doğu saatine göre sabah 9'dan önce veya akşam 10'dan sonra) Heartbeat'ler atlanır. Pencere içindeki bir sonraki zamanlanmış tetikleme normal şekilde çalışır.

### 24/7 kurulumu

Heartbeat'lerin tüm gün çalışmasını istiyorsanız şu kalıplardan birini kullanın:

- `activeHours` öğesini tamamen çıkarın (zaman penceresi kısıtlaması yoktur; varsayılan davranış budur).
- Tam günlük bir pencere ayarlayın: `activeHours: { start: "00:00", end: "24:00" }`.

<Warning>
Aynı `start` ve `end` saatini ayarlamayın (örneğin `08:00` ile `08:00`). Bu, sıfır genişlikli bir pencere olarak kabul edilir; dolayısıyla Heartbeat'ler her zaman atlanır.
</Warning>

### Çok hesaplı örnek

Telegram gibi çok hesaplı kanallarda belirli bir hesabı hedeflemek için `accountId` kullanın:

```json5
{
  agents: {
    list: [
      {
        id: "ops",
        heartbeat: {
          every: "1h",
          target: "telegram",
          to: "12345678:topic:42", // isteğe bağlı: belirli bir konuya/iş parçacığına yönlendir
          accountId: "ops-bot",
        },
      },
    ],
  },
  channels: {
    telegram: {
      accounts: {
        "ops-bot": { botToken: "YOUR_TELEGRAM_BOT_TOKEN" },
      },
    },
  },
}
```

### Alan notları

<ParamField path="every" type="string">
  Heartbeat aralığı (süre dizesi; varsayılan birim = dakika).
</ParamField>
<ParamField path="model" type="string">
  Heartbeat çalıştırmaları için isteğe bağlı model geçersiz kılması (`provider/model`).
</ParamField>
<ParamField path="lightContext" type="boolean" default="false">
  true olduğunda Heartbeat çalıştırmaları hafif başlangıç bağlamını kullanır ve çalışma alanı başlangıç dosyalarını atlar. İzleyici notları her iki durumda da Heartbeat çalıştırıcısı tarafından eklenir.
</ParamField>
<ParamField path="isolatedSession" type="boolean" default="false">
  true olduğunda her Heartbeat, önceki konuşma geçmişi olmadan yeni bir oturumda çalışır. Cron `sessionTarget: "isolated"` ile aynı yalıtım kalıbını kullanır. Heartbeat başına token maliyetini önemli ölçüde azaltır. En yüksek tasarruf için `lightContext: true` ile birleştirin. Teslim yönlendirmesi yine de ana oturum bağlamını kullanır.
</ParamField>
<ParamField path="session" type="string">
  Heartbeat çalıştırmaları için isteğe bağlı oturum anahtarı.

- `main` (varsayılan): agent'ın ana oturumu.
- Açık oturum anahtarı (`openclaw sessions --json` veya [oturumlar CLI'ından](/tr/cli/sessions) kopyalayın).
- Oturum anahtarı biçimleri: [Oturumlar](/tr/concepts/session) ve [Gruplar](/tr/channels/groups) bölümlerine bakın.

</ParamField>
<ParamField path="target" type="string">
- `last`: son kullanılan harici kanala teslim eder.
- açık kanal: yapılandırılmış herhangi bir kanal veya plugin kimliği; örneğin `discord`, `matrix`, `telegram` ya da `whatsapp`.
- `none` (varsayılan): heartbeat'i çalıştırır ancak harici olarak **teslim etmez**.

</ParamField>
<ParamField path="directPolicy" type='"allow" | "block"' default="allow">
  Doğrudan/DM teslim davranışını denetler. `allow`: doğrudan/DM heartbeat teslimine izin verir. `block`: doğrudan/DM teslimini engeller (`reason=dm-blocked`).

</ParamField>
<ParamField path="to" type="string">
  İsteğe bağlı alıcı geçersiz kılma değeri (kanala özgü kimlik; ör. WhatsApp için E.164 veya Telegram sohbet kimliği). Telegram konuları/ileti dizileri için `<chatId>:topic:<messageThreadId>` kullanın.

</ParamField>
<ParamField path="accountId" type="string">
  Birden fazla hesaplı kanallar için isteğe bağlı hesap kimliği. `target: "last"` olduğunda hesap kimliği, hesapları destekliyorsa çözümlenen son kanala uygulanır; aksi takdirde yok sayılır. Hesap kimliği, çözümlenen kanal için yapılandırılmış bir hesapla eşleşmezse teslimat atlanır.

</ParamField>
<ParamField path="prompt" type="string">
  Varsayılan istem gövdesini geçersiz kılar (birleştirilmez).

</ParamField>
<ParamField path="timeoutSeconds" type="number" default="global timeout or min(every, 600)">
  Bir heartbeat ajan turunun iptal edilmeden önce çalışmasına izin verilen azami saniye sayısı. Ayarlanmışsa `agents.defaults.timeoutSeconds` değerini, aksi takdirde 600 saniyeyle sınırlandırılmış heartbeat sıklığını kullanmak için ayarlamadan bırakın.

</ParamField>
<ParamField path="activeHours" type="object">
  Heartbeat çalıştırmalarını bir zaman aralığıyla sınırlar. `start` (HH:MM, dahil; gün başlangıcı için `00:00` kullanın), `end` (HH:MM, hariç; gün sonu için `24:00` kullanılabilir) ve isteğe bağlı `timezone` içeren nesne.

- Belirtilmezse veya `"user"` ise: ayarlanmışsa `agents.defaults.userTimezone` değerinizi kullanır, aksi takdirde ana sistemin saat dilimine geri döner.
- `"local"`: her zaman ana sistemin saat dilimini kullanır.
- Herhangi bir IANA tanımlayıcısı (ör. `America/New_York`): doğrudan kullanılır; geçersizse yukarıdaki `"user"` davranışına geri döner.
- Etkin bir zaman aralığı için `start` ve `end` eşit olmamalıdır; eşit değerler sıfır genişliğinde kabul edilir (her zaman aralığın dışında).
- Etkin zaman aralığının dışında heartbeat'ler, aralık içindeki bir sonraki tetiklemeye kadar atlanır.

</ParamField>

## Teslim davranışı

<AccordionGroup>
  <Accordion title="Oturum ve hedef yönlendirmesi">
    - Heartbeat'ler varsayılan olarak ajanın ana oturumunda (`agent:<id>:<mainKey>`) veya `session.scope = "global"` olduğunda `global` içinde çalışır. Belirli bir kanal oturumunu (Discord/WhatsApp/vb.) geçersiz kılmak için `session` ayarlayın.
    - `session` yalnızca çalıştırma bağlamını etkiler; teslimat `target` ve `to` tarafından denetlenir.
    - Belirli bir kanala/alıcıya teslim etmek için `target` + `to` ayarlayın. `target: "last"` ile teslimat, söz konusu oturumun son harici kanalını kullanır.
    - Heartbeat teslimatları varsayılan olarak doğrudan/DM hedeflerine izin verir. Heartbeat turunu çalıştırmaya devam ederken doğrudan hedeflere gönderimi engellemek için `directPolicy: "block"` ayarlayın.
    - Ana kuyruk, hedef oturum şeridi, cron şeridi veya etkin bir cron işi meşgulse heartbeat atlanır ve daha sonra yeniden denenir.
    - `target` herhangi bir harici hedefe çözümlenmezse çalıştırma yine gerçekleşir ancak giden mesaj gönderilmez.

  </Accordion>
  <Accordion title="Görünürlük ve atlama davranışı">
    - `showOk`, `showAlerts` ve `useIndicator` seçeneklerinin tümü devre dışıysa çalıştırma baştan `reason=alerts-disabled` olarak atlanır.
    - Yalnızca uyarı teslimatı devre dışıysa OpenClaw yine de heartbeat'i çalıştırabilir, zamanı gelen görevlerin zaman damgalarını güncelleyebilir, oturumun boşta kalma zaman damgasını geri yükleyebilir ve dışarıya gönderilecek uyarı yükünü engelleyebilir.
    - Çözümlenen heartbeat hedefi yazıyor göstergesini destekliyorsa heartbeat çalıştırması etkinken OpenClaw yazıyor göstergesini gösterir. Bu, heartbeat'in sohbet çıktısını göndereceği hedefle aynı hedefi kullanır ve `typingMode: "never"` tarafından devre dışı bırakılır.

  </Accordion>
  <Accordion title="Oturum yaşam döngüsü ve denetim">
    - Yalnızca heartbeat içeren yanıtlar oturumu etkin **tutmaz**. Heartbeat meta verileri oturum satırını güncelleyebilir ancak boşta kalma süresinin dolması, son gerçek kullanıcı/kanal mesajındaki `lastInteractionAt` değerini; günlük süre dolumu ise `sessionStartedAt` değerini kullanır.
    - Control UI ve WebChat geçmişi, heartbeat istemlerini ve yalnızca OK içeren onayları gizler. Temel oturum dökümü, denetim/yeniden oynatma amacıyla bu turları yine de içerebilir.
    - Bağımsız [arka plan görevleri](/tr/automation/tasks), ana oturumun bir şeyi hızla fark etmesi gerektiğinde bir sistem olayını kuyruğa alabilir ve heartbeat'i uyandırabilir. Bu uyandırma, heartbeat çalıştırmasını bir arka plan görevine dönüştürmez.

  </Accordion>
</AccordionGroup>

## Görünürlük denetimleri

Varsayılan olarak uyarı içeriği teslim edilirken `HEARTBEAT_OK` onayları engellenir. Bunu kanal veya hesap bazında ayarlayabilirsiniz:

```yaml
channels:
  defaults:
    heartbeat:
      showOk: false # HEARTBEAT_OK değerini gizle (varsayılan)
      showAlerts: true # Uyarı mesajlarını göster (varsayılan)
      useIndicator: true # Gösterge olayları yayınla (varsayılan)
  telegram:
    heartbeat:
      showOk: true # Telegram'da OK onaylarını göster
  whatsapp:
    accounts:
      work:
        heartbeat:
          showAlerts: false # Bu hesap için uyarı teslimatını engelle
```

Öncelik: hesap bazında → kanal bazında → kanal varsayılanları → yerleşik varsayılanlar.

### Her bayrağın işlevi

- `showOk`: model yalnızca OK içeren bir yanıt döndürdüğünde `HEARTBEAT_OK` onayı gönderir.
- `showAlerts`: model OK olmayan bir yanıt döndürdüğünde uyarı içeriğini gönderir.
- `useIndicator`: UI durum yüzeyleri için gösterge olayları yayınlar.

**Üçü de** false ise OpenClaw heartbeat çalıştırmasını tamamen atlar (model çağrısı yapılmaz).

### Kanal ve hesap bazında örnekler

```yaml
channels:
  defaults:
    heartbeat:
      showOk: false
      showAlerts: true
      useIndicator: true
  slack:
    heartbeat:
      showOk: true # tüm Slack hesapları
    accounts:
      ops:
        heartbeat:
          showAlerts: false # yalnızca ops hesabının uyarılarını engelle
  telegram:
    heartbeat:
      showOk: true
```

### Yaygın kalıplar

| Amaç                                             | Yapılandırma                                                                            |
| ------------------------------------------------ | ---------------------------------------------------------------------------------------- |
| Varsayılan davranış (sessiz OK'ler, uyarılar açık) | _(yapılandırma gerekmez)_                                                               |
| Tamamen sessiz (mesaj ve gösterge yok)            | `channels.defaults.heartbeat: { showOk: false, showAlerts: false, useIndicator: false }` |
| Yalnızca gösterge (mesaj yok)                     | `channels.defaults.heartbeat: { showOk: false, showAlerts: false, useIndicator: true }`  |
| Yalnızca bir kanalda OK'ler                       | `channels.telegram.heartbeat: { showOk: true }`                                          |

## İzleyici karalama alanı (isteğe bağlı)

Her heartbeat izleyici cron işi, paylaşılan durum veritabanında saklanan özel bir karalama belgesine sahiptir. Bunu "heartbeat kontrol listeniz" olarak düşünün: küçüktür, kararlıdır ve her 30 dakikada bir göz önünde bulundurulması güvenlidir. Karalama alanı varsa içeriği heartbeat istemine eklenir.

Bunu cron CLI ile yönetin (iş kimliği `openclaw cron list --all` kaynağından gelir):

```bash
openclaw cron scratch <jobId>                 # mevcut karalama alanını yazdır
openclaw cron scratch <jobId> --set "..."     # tam metinle değiştir
openclaw cron scratch <jobId> --file notes.md # bir dosyadan değiştir (stdin için -)
openclaw cron scratch <jobId> --unset         # kaldır
```

Yazma işlemleri karşılaştır-ve-değiştir korumalıdır: eşzamanlı bir düzenlemenin üzerine yazmak yerine başarısız olmak için `--expected-revision <n>` iletin. Karalama alanı 256 KiB ile sınırlıdır ve `cron list`/`cron runs` çıktısında hiçbir zaman görünmez.

Ajan kendi karalama alanını da güncelleyebilir: bir heartbeat turu sırasında `heartbeat_respond`, izleyicinin gelecekteki heartbeat'leri için karalama alanını tamamen değiştiren isteğe bağlı bir `scratch` dizesini kabul eder.

<Note>
**HEARTBEAT.md veya yalnızca yapılandırmaya dayalı sıklıktan geçiş mi yapıyorsunuz?** `openclaw doctor --fix` çalıştırın. Doctor önce `agents.*.heartbeat` üzerinden sistemin sahip olduğu izleyici satırlarını oluşturur veya günceller; ardından her ajanın çalışma alanındaki `HEARTBEAT.md` dosyasını izleyicinin karalama alanına aktarır, geçerli eski `tasks:` girdilerini cron işlerine dönüştürür, özgün dosyayı durum dizini altında arşivler (`backups/heartbeat-migration/`) ve dosyayı kaldırır. Çalışma zamanı heartbeat talimatları yalnızca veritabanındaki karalama alanından gelir; çalışma zamanı `HEARTBEAT.md` dosyasını hiçbir zaman okumaz.
</Note>

Karalama alanı mevcut ancak fiilen boşsa (yalnızca boş satırlar, Markdown/HTML yorumları, `# Heading` gibi Markdown başlıkları, çit işaretleri veya boş kontrol listesi taslakları içeriyorsa), OpenClaw API çağrılarından tasarruf etmek için heartbeat çalıştırmasını atlar. Bu atlama `reason=empty-heartbeat-file` olarak bildirilir. Karalama alanı yoksa heartbeat yine de çalışır ve ne yapılacağına model karar verir.

İstem şişmesini önlemek için küçük tutun (kısa kontrol listesi veya hatırlatıcılar).

Karalama alanı örneği:

```md
# Heartbeat kontrol listesi

- Hızlı tarama: gelen kutularında acil bir şey var mı?
- Gündüzse ve bekleyen başka bir şey yoksa kısa bir yoklama yap.
- Bir görev engellenmişse _neyin eksik olduğunu_ yaz ve bir dahaki sefere Peter'a sor.
```

### Yinelenen kontrolleri cron ile zamanlayın

Heartbeat karalama alanı, zamanlayıcı değil istem bağlamıdır. Her yinelenen kontrolü kendi sıklığına, etkin/devre dışı durumuna ve çalıştırma geçmişine sahip olması için bir [cron işi](/tr/automation/cron-jobs) olarak oluşturun. Kontrolün normal konuşma bağlamını kullanması gerektiğinde cron işleri yine de ana oturumu hedefleyebilir.

Eski karalama alanları yapılandırılmış bir `tasks:` bloğu içerebilir. Yükseltmeden sonra `openclaw doctor --fix` komutunu bir kez çalıştırın: Doctor her geçerli girdiyi bağımsız olarak zamanlanmış bir cron işine dönüştürür, aralığını ve önceki son çalıştırma zamanlamasını korur ve çevresindeki karalama metnini koruyarak kullanımdan kaldırılan bloğu siler. Çalışma zamanı heartbeat turları `tasks:` metnini zamanlama olarak ayrıştırmaz.

Doctor tarafından oluşturulan heartbeat görev işleri, heartbeat'in etkin saatlerini, bekleme süresini, taşma ve meşguliyet korumalarını muhafaza eder. Aynı anda zamanı gelen işler tek bir heartbeat turunda birleştirilebilir. Etkin saatlerin dışındaki bir oluşum atlanır ve bir sonraki cron oluşumunda yeniden denenir.

### Ajan kendi karalama alanını güncelleyebilir mi?

Evet. Bir heartbeat turu sırasında ajan, gelecekteki heartbeat'ler için izleyici metnini tamamen değiştirmek üzere `heartbeat_respond` öğesine bir `scratch` değeri iletebilir. Ayrıca normal bir sohbette `openclaw cron scratch <jobId> --set ...` komutunu çalıştırmasını isteyebilir veya aynı komutla karalama alanını kendiniz düzenleyebilirsiniz. Karalama alanına zamanlayıcı söz dizimi yazmak yerine yinelenen zamanlamaları cron ile yönetin.

<Warning>
İzleyici karalama alanına gizli bilgiler (API anahtarları, telefon numaraları, özel token'lar) koymayın; bunlar istem bağlamının bir parçası hâline gelir.
</Warning>

## Manuel uyandırma (isteğe bağlı)

Bir sistem olayını kuyruğa almak ve isteğe bağlı olarak anında heartbeat tetiklemek için `openclaw system event` kullanın:

```bash
openclaw system event --text "Acil takipleri kontrol et" --mode now
```

| Bayrak                         | Açıklama                                                                                      |
| ---------------------------- | ------------------------------------------------------------------------------------------------ |
| `--text <text>`              | Sistem olayı metni (zorunlu).                                                                    |
| `--mode <mode>`              | `now` hemen bir Heartbeat çalıştırır; `next-heartbeat` (varsayılan) bir sonraki zamanlanmış çevrimi bekler. |
| `--session-key <sessionKey>` | Olay için belirli bir oturumu hedefler; varsayılan olarak aracının ana oturumunu kullanır.                   |
| `--json`                     | JSON çıktısı verir.                                                                                     |

`--session-key` belirtilmezse ve birden fazla aracıda `heartbeat` yapılandırılmışsa, `--mode now` bu aracıların her birinin Heartbeat'ini hemen çalıştırır.

Aynı CLI grubundaki ilgili Heartbeat denetimleri:

```bash
openclaw system heartbeat last     # son Heartbeat olayını göster
openclaw system heartbeat enable   # Heartbeat'leri etkinleştir
openclaw system heartbeat disable  # Heartbeat'leri devre dışı bırak
```

## Maliyet bilinci

Heartbeat'ler tam aracı turları çalıştırır. Daha kısa aralıklar daha fazla token tüketir. Maliyeti azaltmak için:

- Tam konuşma geçmişini göndermekten kaçınmak için `isolatedSession: true` kullanın (çalıştırma başına ~100K tokendan ~2-5K tokene).
- Heartbeat çalıştırmalarında çalışma alanı başlangıç dosyalarını atlamak için `lightContext: true` kullanın.
- Daha ucuz bir `model` ayarlayın (ör. `ollama/llama3.2:1b`).
- İzleyici karalama alanını küçük tutun.
- Yalnızca iç durum güncellemelerini istiyorsanız `target: "none"` kullanın.

## Heartbeat sonrası bağlam taşması

Heartbeat'ler, çalıştırma tamamlandıktan sonra paylaşılan oturumun mevcut çalışma zamanı modelini korur; dolayısıyla bir oturumu daha küçük bir yerel modele (örneğin 32k pencereli bir Ollama modeline) geçiren Heartbeat, bu modeli bir sonraki ana oturum turu için etkin bırakabilir. Sonraki tur bağlam taşması bildirirse ve oturumun son çalışma zamanı modeli yapılandırılmış `heartbeat.model` ile eşleşirse OpenClaw'ın kurtarma mesajı, olası neden olarak Heartbeat modeli sızıntısını belirtir ve bir düzeltme önerir.

Bunu önlemek için: Heartbeat'leri yeni bir oturumda çalıştırmak üzere `isolatedSession: true` kullanın (isteğe bağlı olarak en küçük istem için `lightContext: true` ile birlikte) veya paylaşılan oturum için yeterince büyük bir bağlam penceresine sahip bir Heartbeat modeli seçin.

## İlgili

- [Otomasyon](/tr/automation) - tüm otomasyon mekanizmalarına genel bakış
- [Arka Plan Görevleri](/tr/automation/tasks) - ayrılmış işlerin nasıl izlendiği
- [Saat Dilimi](/tr/concepts/timezone) - saat diliminin Heartbeat zamanlamasını nasıl etkilediği
- [Sorun Giderme](/tr/automation/cron-jobs#troubleshooting) - otomasyon sorunlarını ayıklama
