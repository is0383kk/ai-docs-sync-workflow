---
read_when:
    - OpenClaw'u ilk kez kurma
    - Yaygın yapılandırma kalıpları aranıyor
    - Belirli yapılandırma bölümlerine gitme
summary: 'Yapılandırmaya genel bakış: yaygın görevler, hızlı kurulum ve tam başvuruya bağlantılar'
title: Yapılandırma
x-i18n:
    generated_at: "2026-07-26T23:20:13Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 09cc04efa16f32e12d6ebcea7a1d36b336df32227fe66953c5d70107708ee6c3
    source_path: gateway/configuration.md
    workflow: 16
---

OpenClaw, `~/.openclaw/openclaw.json` konumundan isteğe bağlı bir <Tooltip tip="JSON5 yorumları ve sondaki virgülleri destekler">**JSON5**</Tooltip> yapılandırması okur. Dosya yoksa OpenClaw güvenli varsayılanları kullanır.

Etkin yapılandırma yolu normal bir dosya olmalıdır. OpenClaw tarafından gerçekleştirilen yazma işlemleri dosyayı atomik olarak değiştirir (yolun üzerine yeniden adlandırır); bu nedenle sembolik bağlantı olan bir `openclaw.json` üzerinden yazmak yerine bağlantının hedefi değiştirilir. Sembolik bağlantılı yapılandırma düzenlerinden kaçının. Yapılandırmayı varsayılan durum dizininin dışında tutuyorsanız `OPENCLAW_CONFIG_PATH` değişkenini doğrudan gerçek dosyaya yönlendirin.

Yapılandırma eklemenin yaygın nedenleri:

- Kanalları bağlamak ve bota kimlerin mesaj gönderebileceğini denetlemek
- Modelleri, araçları, korumalı alanı veya otomasyonu (cron, kancalar) ayarlamak
- Oturumları, medyayı, ağı veya kullanıcı arayüzünü ayarlamak

Kullanılabilir tüm alanlar için [tam başvuruya](/tr/gateway/configuration-reference) bakın.

Yapılandırma iki bölümlü bir kural izler: kök düzeyindeki eş düzey alanlar altyapıyı ve aracılar arası varsayılanları içerirken `agents.defaults` aracı döngüsü davranışını içerir. `agents.entries` altındaki girdiler, şemanın aracı başına geçersiz kılmayı desteklediği yerlerde iki bölümü de geçersiz kılabilir.

Aracılar ve otomasyon, yapılandırmayı düzenlemeden önce alan düzeyindeki kesin
belgeler için `config.schema.lookup` kullanmalıdır. Görev odaklı rehberlik için bu sayfayı,
daha geniş alan haritası ve varsayılanlar için
[Yapılandırma başvurusunu](/tr/gateway/configuration-reference) kullanın.

<Tip>
**Yapılandırmayı ilk kez mi kullanıyorsunuz?** Etkileşimli kurulum için `openclaw onboard` ile başlayın veya eksiksiz, kopyalanıp yapıştırılabilir yapılandırmalar için [Yapılandırma Örnekleri](/tr/gateway/configuration-examples) rehberine bakın.
</Tip>

## Asgari yapılandırma

```json5
// ~/.openclaw/openclaw.json
{
  agents: { defaults: { workspace: "~/.openclaw/workspace" } },
  channels: { whatsapp: { allowFrom: ["+15555550123"] } },
}
```

## Yapılandırmayı düzenleme

<Tabs>
  <Tab title="Etkileşimli sihirbaz">
    ```bash
    openclaw onboard       # eksiksiz ilk kullanım akışı
    openclaw configure     # yapılandırma sihirbazı
    ```
  </Tab>
  <Tab title="CLI (tek satırlık komutlar)">
    ```bash
    openclaw config get agents.defaults.workspace
    openclaw config set agents.defaults.heartbeat.every "2h"
    openclaw config unset plugins.entries.brave.config.webSearch.apiKey
    ```
  </Tab>
  <Tab title="Denetim kullanıcı arayüzü">
    [http://127.0.0.1:18789](http://127.0.0.1:18789) adresini açın ve **Config** sekmesini kullanın.
    Denetim kullanıcı arayüzü, varsa alan
    `title` / `description` belge meta verilerinin yanı sıra plugin ve kanal şemalarını da içeren
    canlı yapılandırma şemasından bir form oluşturur; gerektiğinde kullanılmak üzere bir **Raw JSON** düzenleyicisi de sunar. Ayrıntılı
    kullanıcı arayüzleri ve diğer araçlar için Gateway ayrıca, yol kapsamlı tek bir şema
    düğümünü ve doğrudan alt öğelerinin özetlerini getirmek üzere `config.schema.lookup` sunar.
    Ayarlar önce yaygın alanları gösterir. Her bölüm, gelişmiş alanlarını
    daraltılmış bir **Advanced (N)** grubunda tutar; tüm grupları genişletmek için **Show advanced**
    seçeneğini kullanın. Ayarlar araması her zaman iki düzeyi de kapsar ve gerektiğinde
    eşleşen gelişmiş grubu açar.
  </Tab>
  <Tab title="Doğrudan düzenleme">
    `~/.openclaw/openclaw.json` dosyasını doğrudan düzenleyin. Gateway dosyayı izler ve değişiklikleri otomatik olarak uygular ([çalışırken yeniden yükleme](#config-hot-reload) bölümüne bakın).
  </Tab>
</Tabs>

## Katı doğrulama

<Warning>
OpenClaw yalnızca şemayla tamamen eşleşen yapılandırmaları kabul eder. Bilinmeyen anahtarlar, hatalı türler veya geçersiz değerler Gateway'in **başlatmayı reddetmesine** neden olur. Kök düzeyindeki tek istisna `$schema` (dize) alanıdır; böylece düzenleyiciler JSON Schema meta verilerini iliştirebilir.
</Warning>

`openclaw config schema`, Denetim kullanıcı arayüzü ve doğrulama tarafından kullanılan standart JSON Schema'yı yazdırır.
`config.schema.lookup`, ayrıntılı inceleme araçları için yol kapsamlı tek bir düğümü ve
alt öğe özetlerini getirir. Alan `title`/`description` belge meta verileri;
iç içe nesneler, joker karakter (`*`), dizi öğesi (`[]`) ve `anyOf`/
`oneOf`/`allOf` dalları boyunca aktarılır. Bildirim kayıt defteri yüklendiğinde
çalışma zamanı plugin ve kanal şemaları birleştirilir.

Her yapılandırma yaprağının `uiHints` içinde yaygın veya gelişmiş bir sunum düzeyi vardır.
`advanced: false` yaygın ayarları, `advanced: true` ise gelişmiş
ayarları işaretler. Bir yaprağın doğrudan ipucu yoksa en yakın üst öğenin düzeyini devralır;
bildirilmiş bir üst öğesi olmayan yollar varsayılan olarak gelişmiş kabul edilir. Bu yalnızca sunumu
etkiler; doğrulamayı, varsayılanları, yeniden yükleme davranışını veya anahtarın ayarlanıp ayarlanamayacağını etkilemez.

Doğrulama başarısız olduğunda:

- Gateway başlatılmaz
- Yalnızca tanılama komutları çalışır (`openclaw doctor`, `openclaw logs`, `openclaw health`, `openclaw status`)
- Kesin sorunları görmek için `openclaw doctor` komutunu çalıştırın
- Onarımları uygulamak için `openclaw doctor --fix` komutunu çalıştırın (`--repair` aynı bayraktır; `--yes` istemleri atlar)

Gateway, her başarılı başlangıçtan sonra güvenilir bir son bilinen iyi kopyayı saklar
ancak başlangıç ve çalışırken yeniden yükleme bu kopyayı otomatik olarak geri yüklemez; bunu yalnızca `openclaw doctor --fix`
yapar. `openclaw.json` doğrulamadan geçemezse (plugin içi doğrulama dâhil), Gateway
başlangıcı başarısız olur veya yeniden yükleme atlanır ve mevcut çalışma zamanı son kabul edilen
yapılandırmayı kullanmayı sürdürür. Reddedilen yazma işlemi, incelenebilmesi için `<path>.rejected.<timestamp>` olarak da kaydedilir.
Gateway; `gateway.mode` öğesinin kaldırılması, `meta` bloğunun kaybedilmesi veya dosyanın
yarıdan fazla küçültülmesi gibi yanlışlıkla üzerine yazma izlenimi veren yazma işlemlerini, işlem
yıkıcı değişikliklere açıkça izin vermediği sürece engeller. Bir aday `***` veya `[redacted]` gibi
sansürlenmiş bir gizli bilgi yer tutucusu içeriyorsa son bilinen iyi sürüme yükseltilmez.

## Yaygın görevler

<AccordionGroup>
  <Accordion title="Bir kanal kurma (WhatsApp, Telegram, Discord vb.)">
    Her kanalın `channels.<provider>` altında kendi yapılandırma bölümü vardır. Kurulum adımları için ilgili kanal sayfasına bakın:

    - [Discord](/tr/channels/discord) - `channels.discord`
    - [Feishu](/tr/channels/feishu) - `channels.feishu`
    - [Google Chat](/tr/channels/googlechat) - `channels.googlechat`
    - [iMessage](/tr/channels/imessage) - `channels.imessage`
    - [Mattermost](/tr/channels/mattermost) - `channels.mattermost`
    - [Microsoft Teams](/tr/channels/msteams) - `channels.msteams`
    - [Signal](/tr/channels/signal) - `channels.signal`
    - [Slack](/tr/channels/slack) - `channels.slack`
    - [Telegram](/tr/channels/telegram) - `channels.telegram`
    - [WhatsApp](/tr/channels/whatsapp) - `channels.whatsapp`

    Tüm kanallar aynı DM ilkesi kalıbını paylaşır:

    ```json5
    {
      channels: {
        telegram: {
          enabled: true,
          botToken: "123:abc",
          dmPolicy: "pairing",   // pairing | allowlist | open | disabled
          allowFrom: ["tg:123"], // yalnızca allowlist/open için
        },
      },
    }
    ```

  </Accordion>

  <Accordion title="Modelleri seçme ve yapılandırma">
    Birincil modeli ve isteğe bağlı yedek modelleri ayarlayın:

    ```json5
    {
      agents: {
        defaults: {
          model: {
            primary: "anthropic/claude-sonnet-4-6",
            fallbacks: ["openai/gpt-5.4"],
          },
          models: {
            "anthropic/claude-sonnet-4-6": { alias: "Sonnet" },
            "openai/gpt-5.4": { alias: "GPT" },
          },
        },
      },
    }
    ```

    - `agents.defaults.models` takma adları ve model başına ayarları saklar; bir girdi eklemek `/model` veya `--model` geçersiz kılmalarını hiçbir zaman kısıtlamaz.
    - `agents.defaults.modelPolicy.allow`, geçersiz kılmalar ve model seçiciler için açık izin listesidir. Kesin başvuruları ve `provider/*` joker karakterlerini kabul eder; herhangi bir modele izin vermek için bunu atlayın veya `[]` kullanın.
    - Model başvuruları `provider/model` biçimini kullanır (ör. `anthropic/claude-opus-4-6`).
    - `agents.defaults.imageMaxDimensionPx`, transkript/araç görüntülerinin küçültülmesini denetler (varsayılan `1200`); daha düşük değerler, ekran görüntüsü ağırlıklı çalıştırmalarda genellikle görsel token kullanımını azaltır.
    - Sohbette model değiştirmek için [Modeller CLI'sına](/tr/concepts/models), kimlik doğrulama dönüşümü ve yedek davranışı için [Model Yük Devretme](/tr/concepts/model-failover) sayfasına bakın.
    - Özel/kendi barındırdığınız sağlayıcılar için başvurudaki [Özel sağlayıcılar](/tr/gateway/config-tools#custom-providers-and-base-urls) bölümüne bakın.

  </Accordion>

  <Accordion title="Bota kimlerin mesaj gönderebileceğini denetleme">
    DM erişimi, kanal başına `dmPolicy` aracılığıyla denetlenir (varsayılan `"pairing"`):

    - `"pairing"`: bilinmeyen gönderenler, onaylanması için tek kullanımlık bir eşleştirme kodu alır
    - `"allowlist"`: yalnızca `allowFrom` içindeki (veya eşleştirilmiş izin deposundaki) gönderenler
    - `"open"`: gelen tüm DM'lere izin verir (`allowFrom: ["*"]` gerektirir)
    - `"disabled"`: tüm DM'leri yok sayar

    Gruplar için `groupPolicy` (`"allowlist" | "open" | "disabled"`) ile birlikte `groupAllowFrom` veya kanala özgü izin listelerini kullanın.

    Kanal başına ayrıntılar için [tam başvuruya](/tr/gateway/config-channels#dm-and-group-access) bakın.

  </Accordion>

  <Accordion title="Grup sohbeti bahsetme kapısını ayarlama">
    Grup mesajları varsayılan olarak **bahsetme gerektirir**. Tetikleyici kalıplarını aracı başına yapılandırın. Normal grup/kanal yanıtları otomatik olarak gönderilir; aracının ne zaman konuşacağına karar vermesi gereken paylaşımlı odalarda mesaj aracı yolunu etkinleştirin:

    ```json5
    {
      messages: {
        visibleReplies: "automatic", // her yerde mesaj aracı gönderimlerini zorunlu kılmak için "message_tool" olarak ayarlayın
        groupChat: {
          visibleReplies: "message_tool", // isteğe bağlı; görünür çıktı message(action=send) gerektirir
          unmentionedInbound: "room_event", // bahsetme içermeyen, sürekli grup konuşmaları sessiz bağlamdır
        },
      },
      agents: {
        list: [
          {
            id: "main",
            groupChat: {
              mentionPatterns: ["@openclaw", "openclaw"],
            },
          },
        ],
      },
      channels: {
        whatsapp: {
          groups: { "*": { requireMention: true } },
        },
      },
    }
    ```

    - **Meta veri bahsetmeleri**: yerel @bahsetmeleri (WhatsApp dokunarak bahsetme, Telegram @bot vb.)
    - **Metin kalıpları**: `mentionPatterns` içindeki güvenli düzenli ifade kalıpları
    - **Görünür yanıtlar**: `messages.visibleReplies` mesaj aracı gönderimlerini genel olarak zorunlu kılabilir; `messages.groupChat.visibleReplies` bunu gruplar/kanallar için geçersiz kılar.
    - Görünür yanıt modları, kanal başına geçersiz kılmalar ve kendi kendine sohbet modu için [tam başvuruya](/tr/gateway/config-channels#group-chat-mention-gating) bakın.

  </Accordion>

  <Accordion title="Aracı başına becerileri kısıtlama">
    Paylaşılan bir temel için `agents.defaults.skills` kullanın, ardından belirli
    aracıları `agents.entries.*.skills` ile geçersiz kılın:

    ```json5
    {
      agents: {
        defaults: {
          skills: ["github", "weather"],
        },
        list: [
          { id: "writer" }, // github ve weather öğelerini devralır
          { id: "docs", skills: ["docs-search"] }, // varsayılanların yerini alır
          { id: "locked-down", skills: [] }, // Skills yok
        ],
      },
    }
    ```

    - Varsayılan olarak sınırsız Skills için `agents.defaults.skills` öğesini atlayın.
    - Varsayılanları devralmak için `agents.entries.*.skills` öğesini atlayın.
    - Skills olmaması için `agents.entries.*.skills: []` ayarlayın.
    - [Skills](/tr/tools/skills), [Skills yapılandırması](/tr/tools/skills-config) ve
      [Yapılandırma Başvurusuna](/tr/gateway/config-agents#agents-defaults-skills) bakın.

  </Accordion>

  <Accordion title="Kanal başına sistem durumu izlemeyi yapılandırma">
    Bir kanal veya hesap için otomatik sistem durumu yeniden başlatmalarını devre dışı bırakın ya da etkinleştirin:

    ```json5
    {
      channels: {
        telegram: {
          healthMonitor: { enabled: false },
          accounts: {
            alerts: {
              healthMonitor: { enabled: true },
            },
          },
        },
      },
    }
    ```

    - Bir kanal veya hesap için otomatik yeniden başlatmaları denetlemek üzere `channels.<provider>.healthMonitor.enabled` ya da `channels.<provider>.accounts.<id>.healthMonitor.enabled` kullanın.
    - Operasyonel hata ayıklama için [Sistem Durumu Denetimleri](/tr/gateway/health), tüm alanlar için [tam başvuruya](/tr/gateway/configuration-reference#gateway) bakın.

  </Accordion>

  <Accordion title="Oturumları ve sıfırlamaları yapılandırın">
    Oturumlar, konuşma sürekliliğini ve yalıtımını denetler:

    ```json5
    {
      session: {
        dmScope: "per-channel-peer",  // çok kullanıcılı kullanım için önerilir
        threadBindings: {
          enabled: true,
          idleHours: 24,
          maxAgeHours: 0,
        },
        reset: {
          mode: "daily",
          atHour: 4,
          idleMinutes: 120,
        },
      },
    }
    ```

    - `dmScope`: `main` (paylaşılan) | `per-peer` | `per-channel-peer` | `per-account-channel-peer`
    - `threadBindings`: ileti dizisine bağlı oturum yönlendirmesinin genel varsayılanları. `/focus`, `/unfocus`, `/agents`, `/session idle` ve `/session max-age`, bunu her oturum için bağlar, bağlantısını kaldırır, listeler ve ayarlar (Discord ileti dizilerini, Telegram ise konuları/konuşmaları bağlar).
    - Kapsamlandırma, kimlik bağlantıları ve gönderim politikası için [Oturum Yönetimi](/tr/concepts/session) bölümüne bakın.
    - Tüm alanlar için [tam başvuruya](/tr/gateway/config-agents#session) bakın.

  </Accordion>

  <Accordion title="Korumalı alanı etkinleştirin">
    Aracı oturumlarını yalıtılmış korumalı alan çalışma zamanlarında çalıştırın:

    ```json5
    {
      agents: {
        defaults: {
          sandbox: {
            mode: "non-main",  // off | non-main | all
            scope: "agent",    // session | agent | shared
          },
        },
      },
    }
    ```

    Önce imajı oluşturun: kaynak kod deposundan çalışıyorsanız `scripts/sandbox-setup.sh` komutunu çalıştırın; npm kurulumundan çalışıyorsanız [Korumalı Alan § İmajlar ve kurulum](/tr/gateway/sandboxing#images-and-setup) bölümündeki satır içi `docker build` komutuna bakın.

    Tam kılavuz için [Korumalı Alan](/tr/gateway/sandboxing), tüm seçenekler için [tam başvuru](/tr/gateway/config-agents#agentsdefaultssandbox) bölümüne bakın.

  </Accordion>

  <Accordion title="Resmî iOS derlemeleri için aktarıcı destekli anlık bildirimleri etkinleştirin">
    Herkese açık App Store derlemelerinde aktarıcı destekli anlık bildirimler, barındırılan OpenClaw aktarıcısını kullanır: `https://ios-push-relay.openclaw.ai`.

    Özel aktarıcı dağıtımları, aktarıcı URL'si gateway aktarıcı URL'siyle eşleşen, bilinçli olarak ayrı tutulmuş bir iOS derleme/dağıtım yolu gerektirir. Özel bir aktarıcı derlemesi kullanıyorsanız gateway yapılandırmasında şunu ayarlayın:

    ```json5
    {
      gateway: {
        push: {
          apns: {
            relay: {
              baseUrl: "https://relay.example.com",
              // İsteğe bağlı. Varsayılan: 10000
              timeoutMs: 10000,
            },
          },
        },
      },
    }
    ```

    CLI eşdeğeri:

    ```bash
    openclaw config set gateway.push.apns.relay.baseUrl https://relay.example.com
    ```

    Bunun yaptığı işlemler:

    - Gateway'in `push.test`, uyandırma dürtmeleri ve yeniden bağlantı uyandırmalarını haricî aktarıcı üzerinden göndermesini sağlar.
    - Eşleştirilmiş iOS uygulamasının ilettiği, kayıt kapsamlı bir gönderim izni kullanır. Gateway'in dağıtım genelinde geçerli bir aktarıcı belirtecine ihtiyacı yoktur.
    - Aktarıcı destekli her kaydı, iOS uygulamasının eşleştirildiği gateway kimliğine bağlar; böylece başka bir gateway saklanan kaydı yeniden kullanamaz.
    - Yerel/elle oluşturulan iOS derlemelerinin doğrudan APNs kullanmasını sürdürür. Aktarıcı destekli gönderimler yalnızca aktarıcı üzerinden kaydolmuş resmî dağıtım derlemelerine uygulanır.
    - Kayıt ve gönderim trafiğinin aynı aktarıcı dağıtımına ulaşması için iOS derlemesine gömülü aktarıcı temel URL'siyle eşleşmelidir.

    Uçtan uca akış:

    1. Resmî iOS uygulamasını yükleyin.
    2. İsteğe bağlı: yalnızca bilinçli olarak ayrı tutulmuş özel bir aktarıcı derlemesi kullanırken gateway üzerinde `gateway.push.apns.relay.baseUrl` yapılandırmasını yapın.
    3. iOS uygulamasını gateway ile eşleştirin ve hem Node hem de operatör oturumlarının bağlanmasına izin verin.
    4. iOS uygulaması gateway kimliğini alır, App Attest ve uygulama makbuzunu kullanarak aktarıcıya kaydolur, ardından aktarıcı destekli `push.apns.register` yükünü eşleştirilmiş gateway'e yayımlar.
    5. Gateway, aktarıcı tanıtıcısını ve gönderim iznini saklar; ardından bunları `push.test`, uyandırma dürtmeleri ve yeniden bağlantı uyandırmaları için kullanır.

    İşletim notları:

    - iOS uygulamasını farklı bir gateway'e geçirirseniz uygulamanın o gateway'e bağlı yeni bir aktarıcı kaydı yayımlayabilmesi için uygulamayı yeniden bağlayın.
    - Farklı bir aktarıcı dağıtımına işaret eden yeni bir iOS derlemesi yayımlarsanız uygulama, eski aktarıcı kaynağını yeniden kullanmak yerine önbelleğe alınmış aktarıcı kaydını yeniler.

    Uyumluluk notu:

    - `OPENCLAW_APNS_RELAY_BASE_URL` ve `OPENCLAW_APNS_RELAY_TIMEOUT_MS`, geçici ortam değişkeni geçersiz kılmaları olarak çalışmaya devam eder.
    - Özel gateway aktarıcı URL'leri, iOS derlemesine gömülü aktarıcı temel URL'siyle eşleşmelidir; herkese açık App Store sürüm hattı, özel iOS aktarıcı URL'si geçersiz kılmalarını reddeder.
    - `OPENCLAW_APNS_RELAY_ALLOW_HTTP=true`, yalnızca geri döngüye yönelik bir geliştirme kaçış yolu olmaya devam eder; HTTP aktarıcı URL'lerini yapılandırmada kalıcı olarak saklamayın.

    Uçtan uca akış için [iOS Uygulaması](/tr/platforms/ios#relay-backed-push-for-official-builds), aktarıcı güvenlik modeli için [Kimlik doğrulama ve güven akışı](/tr/platforms/ios#authentication-and-trust-flow) bölümüne bakın.

  </Accordion>

  <Accordion title="Heartbeat'i ayarlayın (düzenli yoklamalar)">
    ```json5
    {
      agents: {
        defaults: {
          heartbeat: {
            every: "30m",
            target: "last",
          },
        },
      },
    }
    ```

    - `every`: süre dizesi (`30m`, `2h`). Devre dışı bırakmak için `0m` olarak ayarlayın. Varsayılan: `30m`.
    - `target`: `last` | `none` | `<channel-id>` (örneğin `discord`, `matrix`, `telegram` veya `whatsapp`)
    - `directPolicy`: DM tarzı heartbeat hedefleri için `allow` (varsayılan) veya `block`
    - Tam kılavuz için [Heartbeat](/tr/gateway/heartbeat) bölümüne bakın.

  </Accordion>

  <Accordion title="Cron işlerini yapılandırın">
    ```json5
    {
      cron: {
        enabled: true,
        sessionRetention: "24h",
      },
    }
    ```

    - `sessionRetention`: tamamlanan yalıtılmış çalıştırma oturumlarını SQLite oturum satırlarından temizler (varsayılan `24h`; devre dışı bırakmak için `false` olarak ayarlayın).
    - Çalıştırma geçmişi, iş başına en yeni 2000 terminal satırını otomatik olarak tutar; kayıp satırlar 24 saatlik temizleme sürelerini korur.
    - Özelliğe genel bakış ve CLI örnekleri için [Cron işleri](/tr/automation/cron-jobs) bölümüne bakın.

  </Accordion>

  <Accordion title="Webhook'ları (hook'ları) ayarlayın">
    Gateway üzerinde HTTP webhook uç noktalarını etkinleştirin:

    ```json5
    {
      hooks: {
        enabled: true,
        token: "shared-secret",
        path: "/hooks",
        defaultSessionKey: "hook:ingress",
        allowRequestSessionKey: false,
        allowedSessionKeyPrefixes: ["hook:"],
        mappings: [
          {
            match: { path: "gmail" },
            action: "agent",
            agentId: "main",
            deliver: true,
          },
        ],
      },
    }
    ```

    Güvenlik notu:
    - Tüm hook/webhook yükü içeriğini güvenilmeyen girdi olarak değerlendirin.
    - Özel bir `hooks.token` kullanın; etkin Gateway kimlik doğrulama gizli bilgilerini (`gateway.auth.token` / `OPENCLAW_GATEWAY_TOKEN` veya `gateway.auth.password` / `OPENCLAW_GATEWAY_PASSWORD`) yeniden kullanmayın.
    - Hook kimlik doğrulaması yalnızca üstbilgiyle yapılır (`Authorization: Bearer ...` veya `x-openclaw-token`); sorgu dizesi belirteçleri reddedilir.
    - `hooks.path`, `/` olamaz; webhook girişini `/hooks` gibi özel bir alt yolda tutun.
    - Sıkı kapsamlı hata ayıklama yapmadığınız sürece güvenli olmayan içerik atlama bayraklarını (`hooks.gmail.allowUnsafeExternalContent`, `hooks.mappings[].allowUnsafeExternalContent`) devre dışı tutun.
    - `hooks.allowRequestSessionKey` özelliğini etkinleştirirseniz çağıranın seçtiği oturum anahtarlarını sınırlamak için `hooks.allowedSessionKeyPrefixes` değerini de ayarlayın.
    - Hook ile çalıştırılan aracılar için güçlü ve modern model katmanlarını ve sıkı bir araç politikasını tercih edin (örneğin yalnızca mesajlaşma ve mümkün olduğunda korumalı alan).

    Tüm eşleme seçenekleri ve Gmail entegrasyonu için [tam başvuruya](/tr/gateway/configuration-reference#hooks) bakın.

  </Accordion>

  <Accordion title="Çok aracılı yönlendirmeyi yapılandırın">
    Ayrı çalışma alanları ve oturumları olan birden çok yalıtılmış aracı çalıştırın:

    ```json5
    {
      agents: {
        list: [
          { id: "home", default: true, workspace: "~/.openclaw/workspace-home" },
          { id: "work", workspace: "~/.openclaw/workspace-work" },
        ],
      },
      bindings: [
        { agentId: "home", match: { channel: "whatsapp", accountId: "personal" } },
        { agentId: "work", match: { channel: "whatsapp", accountId: "biz" } },
      ],
    }
    ```

    Bağlama kuralları ve aracı başına erişim profilleri için [Çok Aracılı](/tr/concepts/multi-agent) ve [tam başvuru](/tr/gateway/config-agents#multi-agent-routing) bölümlerine bakın.

  </Accordion>

  <Accordion title="Yapılandırmayı birden çok dosyaya ayırın ($include)">
    Büyük yapılandırmaları düzenlemek için `$include` kullanın:

    ```json5
    // ~/.openclaw/openclaw.json
    {
      gateway: { port: 18789 },
      agents: { $include: "./agents.json5" },
      broadcast: {
        $include: ["./clients/a.json5", "./clients/b.json5"],
      },
    }
    ```

    - **Tek dosya**: kapsayıcı nesnenin yerini alır
    - **Dosya dizisi**: sırayla derinlemesine birleştirilir (sonraki kazanır), en fazla 10 iç içe düzey
    - **Eş düzey anahtarlar**: eklemelerden sonra birleştirilir (eklenen değerleri geçersiz kılar)
    - **Göreli yollar**: eklemeyi yapan dosyaya göre çözümlenir
    - **Yol biçimi**: ekleme yolları null baytları içermemeli ve çözümlemeden önce ve sonra kesinlikle 4096 karakterden kısa olmalıdır
    - **OpenClaw'a ait yazma işlemleri**: bir yazma işlemi yalnızca `plugins: { $include: "./plugins.json5" }` gibi tek dosyalı bir eklemeyle
      desteklenen bir üst düzey bölümü değiştirdiğinde OpenClaw, eklenen bu dosyayı
      günceller ve `openclaw.json` dosyasını olduğu gibi bırakır
    - **Desteklenmeyen doğrudan yazma**: kök eklemeler, ekleme dizileri ve eş düzey
      geçersiz kılmaları olan eklemeler, yapılandırmayı düzleştirmek yerine
      OpenClaw'a ait yazma işlemlerinde güvenli biçimde başarısız olur
    - **Sınırlandırma**: `$include` yolları, `openclaw.json` dosyasını içeren
      dizinin altında çözümlenmelidir. Bir dizin ağacını makineler veya kullanıcılar
      arasında paylaşmak için `OPENCLAW_INCLUDE_ROOTS` değerini, eklemelerin başvurabileceği ek
      dizinlerden oluşan bir yol listesine (POSIX'te `:`, Windows'ta `;`) ayarlayın.
      Sembolik bağlantılar çözümlenir ve yeniden denetlenir; bu nedenle sözcüksel olarak
      bir yapılandırma dizininde bulunan ancak gerçek hedefi izin verilen tüm köklerin
      dışına çıkan bir yol yine de reddedilir.
    - **Hata işleme**: eksik dosyalar, ayrıştırma hataları, döngüsel eklemeler, geçersiz yol biçimi ve aşırı uzunluk için açık hatalar

  </Accordion>
</AccordionGroup>

## Yapılandırmayı çalışırken yeniden yükleme

Gateway, `~/.openclaw/openclaw.json` dosyasını izler ve değişiklikleri otomatik olarak uygular; çoğu ayar için elle yeniden başlatma gerekmez.

Doğrudan dosya düzenlemeleri doğrulanana kadar güvenilmeyen olarak değerlendirilir. İzleyici,
düzenleyicinin geçici yazma/yeniden adlandırma hareketliliğinin durulmasını bekler, son dosyayı
okur ve geçersiz haricî düzenlemeleri `openclaw.json` dosyasını yeniden yazmadan reddeder.
OpenClaw'a ait yapılandırma yazma işlemleri de yazmadan önce aynı şema geçidini kullanır (her
yazmaya uygulanan üzerine yazma/geri alma kuralları için [Sıkı doğrulama](#strict-validation)
bölümüne bakın).

`config reload skipped (invalid config)` görürseniz veya başlangıç sırasında `Invalid
config` bildirilirse yapılandırmayı inceleyin, `openclaw config validate` komutunu ve ardından onarım için `openclaw
doctor --fix` komutunu çalıştırın. Denetim listesi için [Gateway sorun giderme](/tr/gateway/troubleshooting#gateway-rejected-invalid-config)
bölümüne bakın.

### Yeniden yükleme modları

| Mod                   | Davranış                                                                                |
| ---------------------- | --------------------------------------------------------------------------------------- |
| **`hybrid`** (varsayılan) | Güvenli değişiklikleri anında çalışırken uygular. Kritik değişikliklerde otomatik olarak yeniden başlatır.           |
| **`hot`**              | Yalnızca güvenli değişiklikleri çalışırken uygular. Yeniden başlatma gerektiğinde bir uyarı kaydeder; işlemi siz gerçekleştirirsiniz. |
| **`restart`**          | Güvenli olsun veya olmasın, herhangi bir yapılandırma değişikliğinde Gateway'i yeniden başlatır.                                 |
| **`off`**              | Dosya izlemeyi devre dışı bırakır. Değişiklikler bir sonraki manuel yeniden başlatmada etkili olur.                 |

```json5
{
  gateway: {
    reload: { mode: "hybrid", debounceMs: 300 },
  },
}
```

### Çalışırken uygulananlar ve yeniden başlatma gerektirenler

Çoğu alan kesinti olmadan çalışırken uygulanır; çalışırken uygulanan bazı bölümler ise tüm Gateway yerine yalnızca ilgili
alt sistemi (kanal, cron, heartbeat, sistem durumu izleyicisi) yeniden başlatır.
`hybrid` modunda, Gateway'in yeniden başlatılmasını gerektiren değişiklikler otomatik olarak işlenir.

| Kategori            | Alanlar                                                                  | Gateway'in yeniden başlatılması gerekiyor mu?      |
| ------------------- | ----------------------------------------------------------------------- | ---------------------------- |
| Kanallar            | `channels.*`, `web` (WhatsApp) - tüm yerleşik kanallar ve plugin kanalları       | Hayır (ilgili kanalı yeniden başlatır)   |
| Aracı ve modeller      | `agent`, `agents`, `models`, `routing`                                  | Hayır                           |
| Otomasyon          | `hooks`, `cron`, `agent.heartbeat`                                      | Hayır (ilgili alt sistemi yeniden başlatır) |
| Oturumlar ve mesajlar | `session`, `messages`                                                   | Hayır                           |
| Araçlar ve medya       | `tools`, `skills`, `mcp`, `audio`, `talk`                               | Hayır                           |
| Plugin yapılandırması       | `plugins.entries.*`, `plugins.allow`, `plugins.deny`, `plugins.enabled` | Hayır (plugin çalışma zamanını yeniden yükler)  |
| Kullanıcı arayüzü ve diğerleri           | `ui`, `logging`, `identity`, `bindings`                                 | Hayır                           |
| Gateway sunucusu      | `gateway.*` (bağlantı noktası, bağlama, kimlik doğrulama, tailscale, TLS, HTTP, gönderim)              | **Evet**                      |
| Altyapı      | `discovery`, `browser`, `plugins.load`, `plugins.installs`              | **Evet**                      |

<Note>
`gateway.reload` ve `gateway.remote`, `gateway.*` altındaki istisnalardır; bunların değiştirilmesi yeniden başlatmayı **tetiklemez**. Her plugin de bu tabloyu geçersiz kılabilir: yüklenmiş bir plugin, yeniden başlatmayı tetikleyen kendi yapılandırma öneklerini bildirebilir (örneğin paketle birlikte gelen Canvas plugin'i, yalnızca kendi `plugins.entries.canvas` değeri için değil, `plugins.enabled`, `plugins.allow` ve `plugins.deny` için de Gateway'i yeniden başlatır); dolayısıyla gerçek davranış hangi plugin'lerin etkin olduğuna bağlıdır.
</Note>

### Yeniden yükleme planlaması

`$include` üzerinden başvurulan bir kaynak dosyayı düzenlediğinizde OpenClaw,
yeniden yüklemeyi düzleştirilmiş bellek içi görünümden değil, kaynakta yazıldığı düzenden planlar.
Bu, tek bir üst düzey bölüm `plugins: { $include: "./plugins.json5" }` gibi kendisine ait bir dahil edilen dosyada bulunsa bile
çalışırken yeniden yükleme kararlarının (çalışırken uygulama veya yeniden başlatma) öngörülebilir kalmasını sağlar.
Kaynak düzeni belirsizse yeniden yükleme planlaması güvenli biçimde başarısız olur.

## Yapılandırma RPC'si (programatik güncellemeler)

Gateway API'si üzerinden yapılandırma yazan araçlar için şu akışı tercih edin:

- `config.schema.lookup`: tek bir alt ağacı incelemek için (sığ şema düğümü + alt öğe
  özetleri)
- `config.get`: geçerli anlık görüntüyü ve `hash` değerini almak için
- `config.patch`: kısmi güncellemeler için (JSON birleştirme yaması: nesneler birleştirilir, `null`
  siler; girişler kaldırılacaksa `replacePaths` ile açıkça onaylandığında diziler değiştirilir)
- `config.apply`: yalnızca yapılandırmanın tamamını değiştirmek istediğinizde
- `update.run`: açık bir kendi kendini güncelleme ve ardından yeniden başlatma için; yeniden başlatma sonrası oturumun bir takip turu çalıştırması gerekiyorsa `continuationMessage` değerini ekleyin
- `update.status`: en son güncelleme yeniden başlatma işaretçisini incelemek ve yeniden başlatma sonrasında çalışan sürümü doğrulamak için

Aracılar, alan düzeyindeki kesin belgeler ve kısıtlamalar için ilk olarak `config.schema.lookup` değerine başvurmalıdır.
Daha geniş yapılandırma haritasına, varsayılanlara veya özel alt sistem başvurularının bağlantılarına ihtiyaç duyduklarında
[Yapılandırma başvurusu](/tr/gateway/configuration-reference) sayfasını kullanın.

<Note>
Kontrol düzlemi yazma işlemleri (`config.apply`, `config.patch`, `update.run`),
`deviceId+clientIp` başına ve yöntem başına 60 saniyede 30 istekle
sınırlandırılır; bkz. [Hız sınırlama](/gateway/security/rate-limiting). Yeniden başlatma
istekleri birleştirilir ve ardından yeniden başlatma döngüleri arasında 30 saniyelik bekleme süresi uygulanır.
`update.status` salt okunurdur ancak yeniden başlatma işaretçisi güncelleme adımı özetlerini ve
komut çıktılarının son kısımlarını içerebildiğinden yönetici kapsamındadır.
</Note>

Kısmi yama örneği:

```bash
openclaw gateway call config.get --params '{}'  # payload.hash değerini yakala
openclaw gateway call config.patch --params '{
  "raw": "{ channels: { telegram: { groups: { \"*\": { requireMention: false } } } } }",
  "baseHash": "<hash>"
}'
```

Hem `config.apply` hem de `config.patch`; `raw`, `baseHash`, `sessionKey`,
`note` ve `restartDelayMs` değerlerini kabul eder. Bir yapılandırma dosyası zaten mevcut olduğunda
`baseHash` her iki yöntem için de gereklidir (mevcut yapılandırma olmadan yapılan ilk yazma denetimi atlar).

`config.patch` ayrıca, dizi değiştirme işleminin kasıtlı olduğu yapılandırma yollarını içeren
bir dizi olan `replacePaths` değerini kabul eder. Bir yama mevcut bir diziyi daha az giriş içeren bir diziyle
değiştirecek veya silecekse, tam olarak o yol `replacePaths` içinde yer almadığı sürece Gateway yazma işlemini reddeder;
dizi girişlerinin altındaki iç içe diziler, `agents.entries.*.skills` gibi `[]` kullanır.
Bu, kısaltılmış `config.get` anlık görüntülerinin yönlendirme veya izin listesi dizilerinin sessizce üzerine yazmasını önler.
Yapılandırmanın tamamını değiştirmek istediğinizde `config.apply` kullanın.

## Ortam değişkenleri

OpenClaw, ortam değişkenlerini üst süreçten ve ayrıca şuralardan okur:

- `.env`: geçerli çalışma dizininden (varsa)
- `~/.openclaw/.env` (genel geri dönüş)

İki dosya da mevcut ortam değişkenlerini geçersiz kılmaz. Yapılandırmada satır içi ortam değişkenleri de ayarlayabilirsiniz:

```json5
{
  env: {
    OPENROUTER_API_KEY: "sk-or-...",
    vars: { GROQ_API_KEY: "gsk-..." },
  },
}
```

<Accordion title="Kabuk ortamını içe aktarma (isteğe bağlı)">
  Etkinleştirilmişse ve beklenen anahtarlar ayarlanmamışsa OpenClaw, oturum açma kabuğunuzu çalıştırır ve yalnızca eksik anahtarları içe aktarır:

```json5
{
  env: {
    shellEnv: { enabled: true, timeoutMs: 15000 },
  },
}
```

Ortam değişkeni eşdeğeri: `OPENCLAW_LOAD_SHELL_ENV=1`. Varsayılan `timeoutMs`: `15000`.
</Accordion>

<Accordion title="Yapılandırma değerlerinde ortam değişkeni ikamesi">
  Herhangi bir yapılandırma dize değerinde ortam değişkenlerine `${VAR_NAME}` ile başvurun:

```json5
{
  gateway: { auth: { token: "${OPENCLAW_GATEWAY_TOKEN}" } },
  models: { providers: { custom: { apiKey: "${CUSTOM_API_KEY}" } } },
}
```

Kurallar:

- Yalnızca büyük harfli adlar eşleştirilir: `[A-Z_][A-Z0-9_]*`
- Eksik/boş değişkenler yükleme sırasında hata oluşturur
- Değişmez çıktı için `$${VAR}` ile kaçış uygulayın
- `$include` dosyalarının içinde çalışır
- Satır içi ikame: `"${BASE}/v1"` → `"https://api.example.com/v1"`

</Accordion>

<Accordion title="Gizli bilgi başvuruları (ortam, dosya, çalıştırma)">
  SecretRef nesnelerini destekleyen alanlarda şunları kullanabilirsiniz:

```json5
{
  models: {
    providers: {
      openai: { apiKey: { source: "env", provider: "default", id: "OPENAI_API_KEY" } },
    },
  },
  skills: {
    entries: {
      "image-lab": {
        apiKey: {
          source: "file",
          provider: "filemain",
          id: "/skills/entries/image-lab/apiKey",
        },
      },
    },
  },
  channels: {
    googlechat: {
      serviceAccount: {
        source: "exec",
        provider: "vault",
        id: "channels/googlechat/serviceAccount",
      },
    },
  },
}
```

SecretRef ayrıntıları (`env`/`file`/`exec` için `secrets.providers` dahil) [Gizli Bilgi Yönetimi](/tr/gateway/secrets) sayfasındadır.
Desteklenen kimlik bilgisi yolları [SecretRef Kimlik Bilgisi Yüzeyi](/tr/reference/secretref-credential-surface) sayfasında listelenmiştir.
</Accordion>

Tam öncelik sırası ve kaynaklar için [Ortam](/tr/help/environment) sayfasına bakın.

## Tam başvuru

Alanların tamamını tek tek açıklayan başvuru için **[Yapılandırma Başvurusu](/tr/gateway/configuration-reference)** sayfasına bakın.

---

_İlgili: [Yapılandırma Örnekleri](/tr/gateway/configuration-examples) · [Yapılandırma Başvurusu](/tr/gateway/configuration-reference) · [Doctor](/tr/gateway/doctor)_

## İlgili

- [Yapılandırma başvurusu](/tr/gateway/configuration-reference)
- [Yapılandırma örnekleri](/tr/gateway/configuration-examples)
- [Gateway operasyon kılavuzu](/tr/gateway)
