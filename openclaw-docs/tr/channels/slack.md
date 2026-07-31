---
read_when:
    - Slack'i ayarlama veya Slack soket, HTTP ya da aktarma modunda hata ayıklama
summary: Slack kurulumu ve çalışma zamanı davranışı (Socket Mode, HTTP Request URL'leri ve aktarma modu)
title: Slack
x-i18n:
    generated_at: "2026-07-26T22:38:42Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: e0f974ddf8e6965b09cede6a16f171434915a994fa3c1fc744d2350399941bee
    source_path: channels/slack.md
    workflow: 16
---

Slack desteği, Slack uygulama entegrasyonları aracılığıyla DM'leri ve kanalları kapsar. Varsayılan aktarım Socket Mode'dur; HTTP Request URL'leri de desteklenir. Relay modu, güvenilir bir yönlendiricinin Slack girişini yönettiği yönetilen dağıtımlar içindir.

<CardGroup cols={3}>
  <Card title="Eşleştirme" icon="link" href="/tr/channels/pairing">
    Slack DM'leri varsayılan olarak eşleştirme modunu kullanır.
  </Card>
  <Card title="Eğik çizgi komutları" icon="terminal" href="/tr/tools/slash-commands">
    Yerel komut davranışı ve komut kataloğu.
  </Card>
  <Card title="Kanal sorunlarını giderme" icon="wrench" href="/tr/channels/troubleshooting">
    Kanallar arası tanılama ve onarım çalışma planları.
  </Card>
</CardGroup>

## Aktarım seçme

Socket Mode ve HTTP Request URL'leri; mesajlaşma, eğik çizgi komutları, App Home ve etkileşim özelliklerinde eşdeğerdir. Özelliklere göre değil, dağıtım yapısına göre seçim yapın.

| Konu                         | Socket Mode (varsayılan)                                                                                                                             | HTTP Request URL'leri                                                                                           |
| ---------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------- |
| Genel Gateway URL'si         | Gerekli değil                                                                                                                                         | Gerekli (DNS, TLS, ters proxy veya tünel)                                                                       |
| Giden ağ                     | `wss-primary.slack.com` adresine giden WSS erişilebilir olmalıdır                                                                                          | Giden WS yoktur; yalnızca gelen HTTPS                                                                           |
| Gerekli token'lar            | Bot kimliği: bot token'ı + `connections:write` içeren App-Level Token; kullanıcı kimliği: kullanıcı token'ı + App-Level Token                         | Bot kimliği: bot token'ı + Signing Secret; kullanıcı kimliği: kullanıcı token'ı + Signing Secret               |
| Geliştirme dizüstü bilgisayarı / güvenlik duvarı arkası | Olduğu gibi çalışır                                                                                                                  | Genel bir tünel (ngrok, Cloudflare Tunnel, Tailscale Funnel) veya hazırlama Gateway'i gerekir                   |
| Yatay ölçeklendirme          | Her ana makinedeki her uygulama için bir Socket Mode oturumu; birden fazla Gateway ayrı Slack uygulamaları gerektirir                                 | Durumsuz POST işleyicisi; birden fazla Gateway replikası, yük dengeleyicinin arkasında tek bir uygulamayı paylaşabilir |
| Tek Gateway'de birden fazla hesap | Desteklenir; her hesap kendi WS bağlantısını açar                                                                                               | Desteklenir; kayıtların çakışmaması için her hesap benzersiz bir `webhookPath` (varsayılan `/slack/events`) gerektirir |
| Eğik çizgi komutu aktarımı   | WS bağlantısı üzerinden teslim edilir; `slash_commands[].url` yok sayılır                                                                                 | Slack, `slash_commands[].url` adresine POST gönderir; komutun yönlendirilmesi için alan gereklidir                  |
| İstek imzalama               | Kullanılmaz (kimlik doğrulama App-Level Token ile yapılır)                                                                                            | Slack her isteği imzalar; OpenClaw, `signingSecret` ile doğrular                                             |
| Bağlantı kesildiğinde kurtarma | Slack SDK otomatik yeniden bağlanmayı etkinleştirir; OpenClaw ayrıca başarısız Socket Mode oturumlarını sınırlı geri çekilmeyle yeniden başlatır. Pong zaman aşımı aktarım ayarı uygulanır. | Kesilecek kalıcı bir bağlantı yoktur; yeniden denemeler Slack tarafından istek başına yapılır                  |

<Note>
  Tek Gateway'li ana makineler, geliştirme dizüstü bilgisayarları ve `*.slack.com` adresine giden bağlantı kurabilen ancak gelen HTTPS'yi kabul edemeyen şirket içi ağlar için **Socket Mode'u seçin**.

Bir yük dengeleyicinin arkasında birden fazla Gateway replikası çalıştırırken, giden WSS engellenmiş ancak gelen HTTPS'ye izin verilmişse veya Slack webhook'larını zaten bir ters proxy'de sonlandırıyorsanız **HTTP Request URL'lerini seçin**.
</Note>

<Warning>
  Slack, tek bir uygulama için birden fazla Socket Mode bağlantısını sürdürebilir ve her yükü herhangi bir bağlantıya teslim edebilir. Bu nedenle, bir Slack uygulamasını paylaşan ayrı OpenClaw Gateway'lerinin eşdeğer yönlendirme ve yetkilendirme yapılandırmasına sahip olması gerekir. Aksi takdirde, Gateway başına ayrı bir Slack uygulaması, tek bir Relay girişi veya yük dengeleyicinin arkasında HTTP Request URL'leri kullanın. Bkz. [Socket Mode'u Kullanma](https://docs.slack.dev/apis/events-api/using-socket-mode#using-multiple-connections).
</Warning>

### Relay modu

Relay modu, Slack girişini OpenClaw Gateway'inden ayırır. Güvenilir bir yönlendirici tek Slack Socket Mode bağlantısını yönetir, hedef Gateway'i seçer ve kimliği doğrulanmış bir websocket üzerinden türü belirlenmiş bir olay iletir. Gateway, giden Slack Web API çağrıları için yine kendi bot token'ını kullanır.

```json5
{
  channels: {
    slack: {
      mode: "relay",
      botToken: { source: "env", provider: "default", id: "SLACK_BOT_TOKEN" },
      relay: {
        url: "wss://router.example.com/gateway/ws",
        authToken: { source: "env", provider: "default", id: "SLACK_RELAY_AUTH_TOKEN" },
        gatewayId: "team-gateway",
      },
    },
  },
}
```

Relay URL'si localhost'u hedeflemediği sürece `wss://` kullanmalıdır. Bearer token'ını ve yönlendiricinin rota tablosunu Slack yetkilendirme sınırının bir parçası olarak değerlendirin: yönlendirilen olaylar, yetkilendirilmiş etkinleştirmeler olarak normal Slack mesaj işleyicisine girer. Yönlendiricinin websocket `hello` çerçevesinde sağladığı `slack_identity`, varsayılan giden kullanıcı adını ve simgeyi ayarlayabilir; çağıranın açıkça sağladığı kimlik yine önceliklidir. Relay bağlantısı, Socket Mode ile aynı sınırlı geri çekilme zamanlamasıyla yeniden bağlanır ve bağlantı her kesildiğinde yönlendiricinin sağladığı kimliği temizler.

### Enterprise Grid kuruluş genelindeki kurulumlar

Tek bir Slack hesabı, Enterprise Grid kuruluş genelindeki bir kurulumun kapsadığı her çalışma alanından mesaj alabilir. Doğrudan Socket Mode veya HTTP Request URL'lerini seçin; Relay modu kurumsal hesaplar için desteklenmez. Aşağıdaki iki asgari ayrıcalıklı manifest de yalnızca V1 `message` ve `app_mention` olay yolunu, anlık yanıtları ve dinleyicinin yönettiği durum tepkilerini etkinleştirir.

#### Socket Mode

```json
{
  "display_information": {
    "name": "OpenClaw",
    "description": "OpenClaw için Slack bağlayıcısı"
  },
  "features": {
    "bot_user": { "display_name": "OpenClaw", "always_online": true }
  },
  "oauth_config": {
    "scopes": {
      "bot": [
        "app_mentions:read",
        "channels:history",
        "channels:read",
        "chat:write",
        "files:read",
        "files:write",
        "groups:history",
        "groups:read",
        "im:history",
        "im:read",
        "mpim:history",
        "mpim:read",
        "reactions:write",
        "users:read"
      ]
    }
  },
  "settings": {
    "org_deploy_enabled": true,
    "socket_mode_enabled": true,
    "event_subscriptions": {
      "bot_events": [
        "app_mention",
        "message.channels",
        "message.groups",
        "message.im",
        "message.mpim"
      ]
    }
  }
}
```

Bir Enterprise Grid Org Admin veya Org Owner'ın uygulamayı onaylamasını, kuruluş düzeyinde yüklemesini ve kurulumun kapsayacağı çalışma alanlarını seçmesini sağlayın. OpenClaw'ı başlatmadan önce uygulamanın hedeflenen her çalışma alanında kullanılabildiğini doğrulayın. Socket Mode için `connections:write` içeren uygulama düzeyinde bir token oluşturun, ardından kuruluş kurulumundaki bot token'ını kopyalayın. Kuruluşta yüklenen bot token'ını kullanan hesabı yapılandırın:

```json5
{
  channels: {
    slack: {
      enabled: true,
      mode: "socket",
      enterpriseOrgInstall: true,
      appToken: { source: "env", provider: "default", id: "SLACK_APP_TOKEN" },
      botToken: { source: "env", provider: "default", id: "SLACK_BOT_TOKEN" },
      dmPolicy: "open",
      allowFrom: ["*"],
      groupPolicy: "allowlist",
      channels: {
        C0123456789: { requireMention: true },
      },
    },
  },
}
```

#### HTTP Request URL'leri

Gateway'in genel bir HTTPS uç noktası olduğunda ve Socket Mode bağlantısı açmadığında HTTP modunu kullanın. Örnek URL'yi Gateway'in genel `webhookPath` URL'siyle (varsayılan `/slack/events`) değiştirin:

```json
{
  "display_information": {
    "name": "OpenClaw",
    "description": "OpenClaw için Slack bağlayıcısı"
  },
  "features": {
    "bot_user": { "display_name": "OpenClaw", "always_online": true }
  },
  "oauth_config": {
    "scopes": {
      "bot": [
        "app_mentions:read",
        "channels:history",
        "channels:read",
        "chat:write",
        "files:read",
        "files:write",
        "groups:history",
        "groups:read",
        "im:history",
        "im:read",
        "mpim:history",
        "mpim:read",
        "reactions:write",
        "users:read"
      ]
    }
  },
  "settings": {
    "org_deploy_enabled": true,
    "event_subscriptions": {
      "request_url": "https://gateway-host.example.com/slack/events",
      "bot_events": [
        "app_mention",
        "message.channels",
        "message.groups",
        "message.im",
        "message.mpim"
      ]
    }
  }
}
```

Bir Enterprise Grid Org Admin veya Org Owner'ın uygulamayı onaylamasını, kuruluş düzeyinde yüklemesini ve kurulumun kapsayacağı çalışma alanlarını seçmesini sağlayın. Slack, Request URL'yi doğruladıktan sonra kuruluş kurulumunun bot token'ını ve uygulamanın **Basic Information -> App Credentials -> Signing Secret** değerini kopyalayın. Kurumsal hesabı aynı Request URL yolu ile yapılandırın:

```json5
{
  channels: {
    slack: {
      enabled: true,
      mode: "http",
      enterpriseOrgInstall: true,
      botToken: { source: "env", provider: "default", id: "SLACK_BOT_TOKEN" },
      signingSecret: {
        source: "env",
        provider: "default",
        id: "SLACK_SIGNING_SECRET",
      },
      webhookPath: "/slack/events",
      dmPolicy: "open",
      allowFrom: ["*"],
      groupPolicy: "allowlist",
      channels: {
        C0123456789: { requireMention: true },
      },
    },
  },
}
```

Başlangıçta OpenClaw, `enterpriseOrgInstall` değerini Slack `auth.test` ile doğrular. İşaret olmadan kuruluşta yüklenmiş bir token veya işaretle birlikte bir çalışma alanı token'ı başlangıcın başarısız olmasına neden olur. Kuruluma hangi çalışma alanlarının izin verdiği konusunda doğruluk kaynağı Slack olmaya devam eder; ardından OpenClaw, teslim edilen her olaya yapılandırılmış kanal, kullanıcı, DM ve bahsetme politikalarını uygular. Enterprise V1, kuruluş kurulumları döngü önleme için çalışma alanıyla nitelendirilmiş kararlı bir bot kimliği sağlamadığından, `allowBots` değerinden bağımsız olarak bot tarafından oluşturulan tüm `message` ve `app_mention` olaylarını yönlendirmeden önce reddeder.

Enterprise desteği kasıtlı olarak doğrudan Socket Mode veya HTTP `message` ve `app_mention` olaylarıyla ve bunların anlık yanıtlarıyla sınırlıdır. Relay modu, eğik çizgi komutları, etkileşimler, App Home, tepki olayı dinleyicileri, sabitlenen öğeler, Slack eylem araçları, Slack'e özgü onaylar, bağlamalar, kuyruğa alınmış veya zamanlanmış teslimat ve proaktif gönderimler kurumsal hesaplarda kullanılamaz. Giden alındı bildirimi, yazıyor ve durum tepkileri, dinleyicinin yönettiği Slack istemcisi üzerinden desteklenir ve `reactions:write` gerektirir; gelen tepki bildirimleri ve tepki eylemi araçları kullanılamaz.

Anında yanıtlar; parçalar, medya, meta veriler, kimlik yedeği, bağlantı önizlemeleri ve alındılar için standart Slack teslim davranışını yeniden kullanır, ancak yalnızca doğrulanmış, dinleyiciye ait istemci etkin olay çevrimi içinde kaldığı sürece. Bellek içi gönderim kuyruğu ve ileti dizisine katılım kayıtları, söz konusu olayın çalışma alanına göre bölümlenir; istemcinin kendisi hiçbir zaman serileştirilmez veya kalıcı hâle getirilmez.

Kanal ilkesi anahtarları ve `dm.groupChannels` girdileri, ham ve kararlı Slack kanal kimliklerini veya
`channel:<id>` biçimini kullanmalıdır. OpenClaw, çalışma zamanı eşleştirmesi için her iki biçimi de ham kanal kimliğine normalleştirir; `slack:`, `group:` ve `mpim:` ön ekleri başlatmanın başarısız olmasına neden olur.
Kullanıcı ilkesi girdileri kararlı Slack kullanıcı kimliklerini kullanmalıdır; adlar, kısa adlar, görünen adlar ve e-posta adresleri başlatmanın başarısız olmasına neden olur. Kimlikler, Slack'in standart büyük harfli ön ekini ve gövdesini kullanmalıdır (örneğin `C0123456789` veya `U0123456789`); küçük harfli ve kısa benzerleri başlatmanın başarısız olmasına neden olur. Kurumsal hesaplar
`dangerouslyAllowNameMatching` özelliğini etkinleştiremez. Kurumsal hesaplar genel
`mentionPatterns.mode` değerini ayarlayabilir, ancak yalın Slack kanal kimlikleri çalışma alanıyla nitelendirilmediğinden ve çalışma alanları arasında yeniden kullanılabildiğinden `mentionPatterns.allowIn` ve
`mentionPatterns.denyIn` başlatmanın başarısız olmasına neden olur. Çalışma alanı kurulumları mevcut kapsamlı bahsetme kalıbı davranışını korur. Kabul edilen her çalışma alanı, Slack kimlikleri çakışsa bile ayrı yönlendirme, oturum, transkript, yinelenenleri ayıklama, geçmiş ve önbellek kimliği alır. `message` akışı içinde sıradan kullanıcı mesajları ve kullanıcı tarafından oluşturulan `file_share` olayları desteklenir; diğer mesaj alt türleri yetkilendirme veya sistem olayı işlemeden önce reddedilir.

Kurumsal DM'ler ya devre dışı bırakılmalı (`dm.enabled=false` veya
`dmPolicy="disabled"`) ya da `dmPolicy="open"` ile açıkça açılmalı ve değişmez `"*"` değerini içeren etkin bir hesap `allowFrom` ayarına sahip olmalıdır. Boş bir izin listesi veya `"*"` olmadan kullanıcıya özgü kimlikler başlatmanın başarısız olmasına neden olur. Slack kullanıcı kimlikleri bu yetkilendirme depolarında çalışma alanıyla nitelendirilmediğinden eşleştirme ve kullanıcı başına DM izin listeleri reddedilir. Kanal ve gönderen ilkesi kanal mesajlarına uygulanmaya devam eder.

## Kurulum

```bash
openclaw plugins install @openclaw/slack
```

`plugins install`, Plugin'i kaydeder ve etkinleştirir. Aşağıdaki Slack uygulaması ve kanal ayarlarını yapılandırana kadar hiçbir işlem yapmaz. Genel Plugin kurulum kuralları için [Pluginler](/tr/tools/plugin) bölümüne bakın.

## Hızlı kurulum

Bu bölümdeki manifestler, çalışma alanı kapsamlı bir kurulum oluşturur. Enterprise Grid kuruluş kurulumu için bunun yerine özel
[kuruluş genelindeki manifesti ve iş akışını](#enterprise-grid-org-wide-installs) kullanın.

<Tabs>
  <Tab title="Socket Mode (varsayılan)">
    <Steps>
      <Step title="Yeni bir Slack uygulaması oluşturun">
        [api.slack.com/apps](https://api.slack.com/apps/new) sayfasını açın → **Create New App** → **From a manifest** → çalışma alanınızı seçin → aşağıdaki manifestlerden birini yapıştırın → **Next** → **Create**.

        <CodeGroup>

```json Recommended
{
  "display_information": {
    "name": "OpenClaw",
    "description": "OpenClaw için Slack bağlayıcısı"
  },
  "features": {
    "bot_user": { "display_name": "OpenClaw", "always_online": true },
    "app_home": {
      "home_tab_enabled": true,
      "messages_tab_enabled": true,
      "messages_tab_read_only_enabled": false
    },
    "agent_view": {
      "agent_description": "OpenClaw, Slack Agent View konuşmalarını OpenClaw aracılarının kullanımına bağlar.",
      "suggested_prompts": [
        { "title": "Neler yapabilirsiniz?", "message": "Bana hangi konularda yardımcı olabilirsiniz?" },
        {
          "title": "Bu kanalı özetle",
          "message": "Bu kanaldaki son etkinlikleri özetle."
        },
        { "title": "Yanıt taslağı hazırla", "message": "Bir yanıt taslağı hazırlamama yardım et." }
      ]
    },
    "slash_commands": [
      {
        "command": "/openclaw",
        "description": "OpenClaw'a mesaj gönder",
        "should_escape": false
      }
    ]
  },
  "oauth_config": {
    "scopes": {
      "bot": [
        "app_mentions:read",
        "assistant:write",
        "channels:history",
        "channels:read",
        "chat:write",
        "commands",
        "emoji:read",
        "files:read",
        "files:write",
        "groups:history",
        "groups:read",
        "im:history",
        "im:read",
        "im:write",
        "mpim:history",
        "mpim:read",
        "mpim:write",
        "pins:read",
        "pins:write",
        "reactions:read",
        "reactions:write",
        "usergroups:read",
        "users:read"
      ]
    }
  },
  "settings": {
    "socket_mode_enabled": true,
    "event_subscriptions": {
      "bot_events": [
        "app_home_opened",
        "app_mention",
        "app_context_changed",
        "channel_rename",
        "member_joined_channel",
        "member_left_channel",
        "message.channels",
        "message.groups",
        "message.im",
        "message.mpim",
        "pin_added",
        "pin_removed",
        "reaction_added",
        "reaction_removed"
      ]
    }
  }
}
```

```json Minimal
{
  "display_information": {
    "name": "OpenClaw",
    "description": "OpenClaw için Slack bağlayıcısı"
  },
  "features": {
    "bot_user": { "display_name": "OpenClaw", "always_online": true },
    "app_home": {
      "home_tab_enabled": true,
      "messages_tab_enabled": true,
      "messages_tab_read_only_enabled": false
    },
    "agent_view": {
      "agent_description": "OpenClaw, Slack Agent View konuşmalarını OpenClaw aracılarının kullanımına bağlar.",
      "suggested_prompts": [
        { "title": "Neler yapabilirsiniz?", "message": "Bana hangi konularda yardımcı olabilirsiniz?" },
        {
          "title": "Bu kanalı özetle",
          "message": "Bu kanaldaki son etkinlikleri özetle."
        },
        { "title": "Yanıt taslağı hazırla", "message": "Bir yanıt taslağı hazırlamama yardım et." }
      ]
    },
    "slash_commands": [
      {
        "command": "/openclaw",
        "description": "OpenClaw'a mesaj gönder",
        "should_escape": false
      }
    ]
  },
  "oauth_config": {
    "scopes": {
      "bot": [
        "app_mentions:read",
        "assistant:write",
        "channels:history",
        "channels:read",
        "chat:write",
        "commands",
        "groups:history",
        "groups:read",
        "im:history",
        "im:read",
        "im:write",
        "users:read"
      ]
    }
  },
  "settings": {
    "socket_mode_enabled": true,
    "event_subscriptions": {
      "bot_events": [
        "app_home_opened",
        "app_mention",
        "app_context_changed",
        "message.channels",
        "message.groups",
        "message.im"
      ]
    }
  }
}
```

        </CodeGroup>

        <Note>
          **Önerilen**, Slack Plugin'inin tüm özellikleriyle eşleşir: App Home, eğik çizgi komutları, dosyalar, tepkiler, sabitlenenler, grup DM'leri ve emoji/kullanıcı grubu okumaları. Çalışma alanı ilkesi kapsamları kısıtladığında **Asgari** seçeneğini kullanın; DM'leri, kanal/grup geçmişini, bahsetmeleri ve eğik çizgi komutlarını kapsar ancak dosyaları, tepkileri, sabitlenenleri, grup DM'sini (`mpim:*`), `emoji:read` ve `usergroups:read` özelliklerini çıkarır. Kapsam başına gerekçeler ve ek eğik çizgi komutları gibi ilave seçenekler için [Manifest ve kapsam denetim listesi](#manifest-and-scope-checklist) bölümüne bakın.
        </Note>

        Slack uygulamayı oluşturduktan sonra:

        - **Basic Information -> App-Level Tokens -> Generate Token and Scopes**: `connections:write` ekleyin, kaydedin ve App-Level Token değerini kopyalayın.
        - **Install App -> Install to Workspace**: Bot User OAuth Token değerini kopyalayın.

      </Step>

      <Step title="OpenClaw'ı yapılandırın">

        Önerilen SecretRef kurulumu:

```bash
export SLACK_APP_TOKEN=slack-app-token-example
export SLACK_BOT_TOKEN=slack-bot-token-example
cat > slack.socket.patch.json5 <<'JSON5'
{
  channels: {
    slack: {
      enabled: true,
      mode: "socket",
      appToken: { source: "env", provider: "default", id: "SLACK_APP_TOKEN" },
      botToken: { source: "env", provider: "default", id: "SLACK_BOT_TOKEN" },
    },
  },
}
JSON5
openclaw config patch --file ./slack.socket.patch.json5 --dry-run
openclaw config patch --file ./slack.socket.patch.json5
```

        Ortam değişkeni yedeği (yalnızca varsayılan hesap):

```bash
SLACK_APP_TOKEN=slack-app-token-example
SLACK_BOT_TOKEN=slack-bot-token-example
```

      </Step>

      <Step title="Gateway'i başlatın">

```bash
openclaw gateway
```

      </Step>
    </Steps>

  </Tab>

  <Tab title="HTTP İstek URL'leri">
    <Steps>
      <Step title="Yeni bir Slack uygulaması oluşturun">
        [api.slack.com/apps](https://api.slack.com/apps/new) sayfasını açın → **Create New App** → **From a manifest** → çalışma alanınızı seçin → aşağıdaki manifestlerden birini yapıştırın → `https://gateway-host.example.com/slack/events` değerini genel Gateway URL'nizle değiştirin → **Next** → **Create**.

        <CodeGroup>

```json Recommended
{
  "display_information": {
    "name": "OpenClaw",
    "description": "OpenClaw için Slack bağlayıcısı"
  },
  "features": {
    "bot_user": { "display_name": "OpenClaw", "always_online": true },
    "app_home": {
      "home_tab_enabled": true,
      "messages_tab_enabled": true,
      "messages_tab_read_only_enabled": false
    },
    "agent_view": {
      "agent_description": "OpenClaw, Slack Agent View konuşmalarını OpenClaw aracılarının kullanımına bağlar.",
      "suggested_prompts": [
        { "title": "Neler yapabilirsiniz?", "message": "Bana hangi konularda yardımcı olabilirsiniz?" },
        {
          "title": "Bu kanalı özetle",
          "message": "Bu kanaldaki son etkinlikleri özetle."
        },
        { "title": "Yanıt taslağı hazırla", "message": "Bir yanıt taslağı hazırlamama yardım et." }
      ]
    },
    "slash_commands": [
      {
        "command": "/openclaw",
        "description": "OpenClaw'a mesaj gönder",
        "should_escape": false,
        "url": "https://gateway-host.example.com/slack/events"
      }
    ]
  },
  "oauth_config": {
    "scopes": {
      "bot": [
        "app_mentions:read",
        "assistant:write",
        "channels:history",
        "channels:read",
        "chat:write",
        "commands",
        "emoji:read",
        "files:read",
        "files:write",
        "groups:history",
        "groups:read",
        "im:history",
        "im:read",
        "im:write",
        "mpim:history",
        "mpim:read",
        "mpim:write",
        "pins:read",
        "pins:write",
        "reactions:read",
        "reactions:write",
        "usergroups:read",
        "users:read"
      ]
    }
  },
  "settings": {
    "event_subscriptions": {
      "request_url": "https://gateway-host.example.com/slack/events",
      "bot_events": [
        "app_home_opened",
        "app_mention",
        "app_context_changed",
        "channel_rename",
        "member_joined_channel",
        "member_left_channel",
        "message.channels",
        "message.groups",
        "message.im",
        "message.mpim",
        "pin_added",
        "pin_removed",
        "reaction_added",
        "reaction_removed"
      ]
    },
    "interactivity": {
      "is_enabled": true,
      "request_url": "https://gateway-host.example.com/slack/events",
      "message_menu_options_url": "https://gateway-host.example.com/slack/events"
    }
  }
}
```

```json Minimal
{
  "display_information": {
    "name": "OpenClaw",
    "description": "OpenClaw için Slack bağlayıcısı"
  },
  "features": {
    "bot_user": { "display_name": "OpenClaw", "always_online": true },
    "app_home": {
      "home_tab_enabled": true,
      "messages_tab_enabled": true,
      "messages_tab_read_only_enabled": false
    },
    "agent_view": {
      "agent_description": "OpenClaw, Slack Agent View konuşmalarını OpenClaw ajanlarına bağlar.",
      "suggested_prompts": [
        { "title": "Neler yapabilirsin?", "message": "Bana hangi konularda yardımcı olabilirsin?" },
        {
          "title": "Bu kanalı özetle",
          "message": "Bu kanaldaki son etkinlikleri özetle."
        },
        { "title": "Bir yanıt taslağı hazırla", "message": "Bir yanıt taslağı hazırlamama yardım et." }
      ]
    },
    "slash_commands": [
      {
        "command": "/openclaw",
        "description": "OpenClaw'a mesaj gönder",
        "should_escape": false,
        "url": "https://gateway-host.example.com/slack/events"
      }
    ]
  },
  "oauth_config": {
    "scopes": {
      "bot": [
        "app_mentions:read",
        "assistant:write",
        "channels:history",
        "channels:read",
        "chat:write",
        "commands",
        "groups:history",
        "groups:read",
        "im:history",
        "im:read",
        "im:write",
        "users:read"
      ]
    }
  },
  "settings": {
    "event_subscriptions": {
      "request_url": "https://gateway-host.example.com/slack/events",
      "bot_events": [
        "app_home_opened",
        "app_mention",
        "app_context_changed",
        "message.channels",
        "message.groups",
        "message.im"
      ]
    },
    "interactivity": {
      "is_enabled": true,
      "request_url": "https://gateway-host.example.com/slack/events",
      "message_menu_options_url": "https://gateway-host.example.com/slack/events"
    }
  }
}
```

        </CodeGroup>

        <Note>
          **Önerilen**, Slack plugininin tüm özellikleriyle eşleşir; **Minimal**, kısıtlayıcı çalışma alanları için dosyaları, tepkileri, sabitlenenleri, grup DM'lerini (`mpim:*`), `emoji:read` ve `usergroups:read` öğelerini çıkarır. Her kapsamın gerekçesi için [Manifest ve kapsam denetim listesine](#manifest-and-scope-checklist) bakın.
        </Note>

        <Info>
          Üç URL alanının (`slash_commands[].url`, `event_subscriptions.request_url` ve `interactivity.request_url` / `message_menu_options_url`) tümü aynı OpenClaw uç noktasına yönelir. Slack'in manifest şeması bunların ayrı ayrı adlandırılmasını gerektirir, ancak OpenClaw yük türüne göre yönlendirme yaptığından tek bir `webhookPath` (varsayılan `/slack/events`) yeterlidir. `slash_commands[].url` içermeyen eğik çizgi komutları HTTP modunda sessizce hiçbir işlem yapmaz.
        </Info>

        Slack uygulamayı oluşturduktan sonra:

        - **Basic Information → App Credentials**: istek doğrulaması için **Signing Secret** değerini kopyalayın.
        - **Install App -> Install to Workspace**: Bot User OAuth Token değerini kopyalayın.

      </Step>

      <Step title="OpenClaw'ı yapılandır">

        Önerilen SecretRef kurulumu:

```bash
export SLACK_BOT_TOKEN=slack-bot-token-example
export SLACK_SIGNING_SECRET=...
cat > slack.http.patch.json5 <<'JSON5'
{
  channels: {
    slack: {
      enabled: true,
      mode: "http",
      botToken: { source: "env", provider: "default", id: "SLACK_BOT_TOKEN" },
      signingSecret: { source: "env", provider: "default", id: "SLACK_SIGNING_SECRET" },
      webhookPath: "/slack/events",
    },
  },
}
JSON5
openclaw config patch --file ./slack.http.patch.json5 --dry-run
openclaw config patch --file ./slack.http.patch.json5
```

        <Note>
        Çok hesaplı HTTP için benzersiz webhook yolları kullanın

        Kayıtların çakışmaması için her hesaba farklı bir `webhookPath` (varsayılan `/slack/events`) verin.
        </Note>

      </Step>

      <Step title="Gateway'i başlat">

```bash
openclaw gateway
```

      </Step>
    </Steps>

  </Tab>
</Tabs>

## Kullanıcı kimliği (gerçek bir kişi olarak paylaşım yapma)

Kullanıcı kimliği, OpenClaw'ın Slack uygulamasını yetkilendiren insan adına içerik okumasına ve paylaşmasına olanak tanır. `userToken` işlem yapan kimliktir; eşlik eden bir Slack uygulaması, Events API trafiğini Socket Mode veya bir HTTP Request URL üzerinden taşır. Eşlik eden uygulamanın bot kullanıcısına veya bot token'ına ihtiyacı yoktur.

Eşlik eden uygulamayı şu şekilde kurun:

1. **OAuth & Permissions -> User Token Scopes** altında, kullanıcı kapsamlı şu izinleri ekleyin:

   - geçmiş: `channels:history`, `groups:history`, `im:history`, `mpim:history`
   - konuşma arama: `channels:read`, `groups:read`, `im:read`, `mpim:read`
   - kişiler: `users:read`
   - paylaşım yapma: `chat:write` (mesajlar yetkilendiren kullanıcı adına paylaşılır)
   - DM açma: `im:write`, `mpim:write`

2. **Event Subscriptions -> Subscribe to events on behalf of users** altında, şu kullanıcı olaylarını ekleyin. Bunları yalnızca bot olayları listesine eklemeyin:

   - `message.channels`
   - `message.groups`
   - `message.im`
   - `message.mpim`

3. Bir olay taşıma yöntemi seçin:

   - **Socket Mode:** Socket Mode'u etkinleştirin ve `connections:write` ile uygulama düzeyinde bir token oluşturun. Bunu `appToken` olarak yapılandırın.
   - **HTTP Request URL:** Event Subscriptions öğesini herkese açık OpenClaw Slack uç noktasına yönlendirin ve **Basic Information -> App Credentials -> Signing Secret** değerini kopyalayın. Bunu `signingSecret` olarak yapılandırın.

4. Uygulamayı yükleyin veya yeniden yükleyin, hedeflenen insan olarak yetkilendirin ve elde edilen kullanıcı OAuth token'ını `userToken` içine kopyalayın.

Socket Mode yapılandırması:

```json5
{
  channels: {
    slack: {
      identity: "user",
      userToken: "<xoxp>",
      appToken: "<xapp>",
    },
  },
}
```

HTTP Request URL yapılandırması:

```json5
{
  channels: {
    slack: {
      identity: "user",
      mode: "http",
      userToken: "<xoxp>",
      signingSecret: "<signing-secret>",
      webhookPath: "/slack/events",
    },
  },
}
```

<Warning>
  DM'ler ve grup DM'leri yalnızca yukarıdaki kullanıcı kapsamlı olay aboneliği üzerinden çalışır. Bir bot, insanlar arasındaki bir 1:1 DM'ine katılamaz veya mevcut bir grup DM'ine eklenemez. Eşlik eden uygulama görünmez bir altyapıdır: diğer Slack üyeleri mesajları bir OpenClaw botundan değil, yetkilendiren insandan görür.
</Warning>

OpenClaw, çözümlenen insan kimliğinin yazdığı kullanıcı kapsamlı mesaj olaylarını otomatik olarak bırakır; böylece gönderdiği mesajlar kendi kendine yanıtları tetiklemez.

## Socket Mode taşıma ayarları

OpenClaw, Socket Mode için Slack SDK istemcisinin pong zaman aşımını varsayılan olarak 15 saniyeye ayarlar. Taşıma ayarlarını yalnızca çalışma alanına veya ana makineye özgü ayarlama gerektiğinde geçersiz kılın:

```json5
{
  channels: {
    slack: {
      mode: "socket",
      socketMode: {
        clientPingTimeout: 20000,
        serverPingTimeout: 30000,
        pingPongLoggingEnabled: false,
      },
    },
  },
}
```

Bunu yalnızca Slack websocket pong/sunucu ping zaman aşımlarını günlüğe kaydeden veya olay döngüsünde bilinen tıkanmaların bulunduğu ana makinelerde çalışan Socket Mode çalışma alanları için kullanın. `clientPingTimeout`, SDK bir istemci pingi gönderdikten sonraki pong bekleme süresidir; `serverPingTimeout`, Slack sunucu pinglerinin bekleme süresidir. Uygulama mesajları ve olayları taşıma canlılığı sinyalleri değil, uygulama durumu olarak kalır.

Notlar:

- `socketMode`, HTTP Request URL modunda yok sayılır.
- Temel `channels.slack.socketMode` ayarları, geçersiz kılınmadıkları sürece tüm Slack hesaplarına uygulanır. Hesaba özgü geçersiz kılmalar `channels.slack.accounts.<accountId>.socketMode` kullanır; bu bir nesne geçersiz kılması olduğundan, ilgili hesap için istediğiniz tüm soket ayarlama alanlarını ekleyin.
- Yalnızca `clientPingTimeout` için bir OpenClaw varsayılanı (`15000`) vardır. `serverPingTimeout` ve `pingPongLoggingEnabled`, yalnızca yapılandırıldıklarında Slack SDK'ya aktarılır.
- Socket Mode yeniden başlatma geri çekilmesi yaklaşık 2 saniyede başlar ve yaklaşık 30 saniyede üst sınıra ulaşır. Kurtarılabilir başlatma, başlatmayı bekleme ve bağlantı kesilmesi hataları, kanal durana kadar yeniden denenir. Geçersiz kimlik doğrulama, iptal edilmiş token'lar veya eksik kapsamlar gibi kalıcı hesap ve kimlik bilgisi hataları sonsuza kadar yeniden denenmek yerine hızla başarısız olur.

## Manifest ve kapsam denetim listesi

Temel Slack uygulama manifesti, Socket Mode ve HTTP Request URL'leri için aynıdır. Yalnızca `settings` bloğu (ve eğik çizgi komutunun `url` değeri) farklıdır.

Temel manifest (Socket Mode varsayılanı):

```json
{
  "display_information": {
    "name": "OpenClaw",
    "description": "OpenClaw için Slack bağlayıcısı"
  },
  "features": {
    "bot_user": { "display_name": "OpenClaw", "always_online": true },
    "app_home": {
      "home_tab_enabled": true,
      "messages_tab_enabled": true,
      "messages_tab_read_only_enabled": false
    },
    "agent_view": {
      "agent_description": "OpenClaw, Slack Agent View konuşmalarını OpenClaw ajanlarına bağlar.",
      "suggested_prompts": [
        { "title": "Neler yapabilirsin?", "message": "Bana hangi konularda yardımcı olabilirsin?" },
        {
          "title": "Bu kanalı özetle",
          "message": "Bu kanaldaki son etkinlikleri özetle."
        },
        { "title": "Bir yanıt taslağı hazırla", "message": "Bir yanıt taslağı hazırlamama yardım et." }
      ]
    },
    "slash_commands": [
      {
        "command": "/openclaw",
        "description": "OpenClaw'a mesaj gönder",
        "should_escape": false
      }
    ]
  },
  "oauth_config": {
    "scopes": {
      "bot": [
        "app_mentions:read",
        "assistant:write",
        "channels:history",
        "channels:read",
        "chat:write",
        "commands",
        "emoji:read",
        "files:read",
        "files:write",
        "groups:history",
        "groups:read",
        "im:history",
        "im:read",
        "im:write",
        "mpim:history",
        "mpim:read",
        "mpim:write",
        "pins:read",
        "pins:write",
        "reactions:read",
        "reactions:write",
        "usergroups:read",
        "users:read"
      ]
    }
  },
  "settings": {
    "socket_mode_enabled": true,
    "event_subscriptions": {
      "bot_events": [
        "app_home_opened",
        "app_mention",
        "app_context_changed",
        "channel_rename",
        "member_joined_channel",
        "member_left_channel",
        "message.channels",
        "message.groups",
        "message.im",
        "message.mpim",
        "pin_added",
        "pin_removed",
        "reaction_added",
        "reaction_removed"
      ]
    }
  }
}
```

**HTTP Request URLs modu** için `settings` öğesini HTTP varyantıyla değiştirin ve her eğik çizgi komutuna `url` ekleyin. Herkese açık URL gereklidir:

```json
{
  "features": {
    "slash_commands": [
      {
        "command": "/openclaw",
        "description": "OpenClaw'a mesaj gönder",
        "should_escape": false,
        "url": "https://gateway-host.example.com/slack/events"
      }
    ]
  },
  "settings": {
    "event_subscriptions": {
      "request_url": "https://gateway-host.example.com/slack/events",
      "bot_events": [
        "app_home_opened",
        "app_mention",
        "app_context_changed",
        "channel_rename",
        "member_joined_channel",
        "member_left_channel",
        "message.channels",
        "message.groups",
        "message.im",
        "message.mpim",
        "pin_added",
        "pin_removed",
        "reaction_added",
        "reaction_removed"
      ]
    },
    "interactivity": {
      "is_enabled": true,
      "request_url": "https://gateway-host.example.com/slack/events",
      "message_menu_options_url": "https://gateway-host.example.com/slack/events"
    }
  }
}
```

### Ek manifest ayarları

Yukarıdaki varsayılanları genişleten farklı özellikleri kullanıma sunun.

Varsayılan bildirim, Slack Uygulama Ana Sayfası **Home** sekmesini etkinleştirir ve `app_home_opened` olayına abone olur. Bir çalışma alanı üyesi Home sekmesini açtığında OpenClaw, `views.publish` ile güvenli bir varsayılan Home görünümü yayımlar; hiçbir konuşma yükü veya özel yapılandırma eklenmez. Tek eğik çizgi komutu modu etkinleştirildiğinde komut ipucu `channels.slack.slashCommand.name` kullanır; yerel komutları kullanan veya eğik çizgi komutu kullanmayan kurulumlarda bu ipucu gösterilmez. Slack DM'leri için **Messages** sekmesi etkin kalır. Yeni uygulamalar `features.agent_view`, `assistant:write` ve `app_context_changed` aracılığıyla Slack Agent View kullanır. Görünür her Agent View kökü kendi OpenClaw ileti dizisi oturumuna yönlendirilir ve Slack'in sıralı etkin görünüm varlıkları aracıya yalnızca güvenilmeyen bağlam olarak ulaşır.

Zaten `features.assistant_view` kullanan mevcut uygulamalar geçerli bildirimlerini koruyabilir. OpenClaw, bu kurulumlar için `assistant_thread_started` ve `assistant_thread_context_changed` olaylarını işlemeye devam eder. Slack, Assistant View'dan Agent View'a geçişi geri döndürülemez hâle getirir ve sonrasında kullanıcıların tam yenileme yapmasını gerektirir; bu nedenle tüm çalışma alanını taşımayı planlayana kadar mevcut bir uygulamada `assistant_view` değerini değiştirmeyin.

<AccordionGroup>
  <Accordion title="İsteğe bağlı yerel eğik çizgi komutları">

    Bazı ayrıntılar dikkate alınarak tek bir yapılandırılmış komut yerine birden çok [yerel eğik çizgi komutu](#commands-and-slash-behavior) kullanılabilir:

    - `/status` komutu ayrılmış olduğundan `/status` yerine `/agentstatus` kullanın.
    - Bir Slack uygulamasında aynı anda en fazla 25 eğik çizgi komutu kaydedilebilir (Slack platformu sınırı).

    OpenClaw, etkinleştirilen yerel komutlar için işleyiciler kaydeder; ancak Slack bildirim girdileri yöneticiler tarafından yönetilmeye devam eder ve çalışma zamanında eşitlenmez. Bildirime `/login` değerini manuel olarak ekleyin; aşağıdaki örnek, 25 komut sınırında kalmak için isteğe bağlı `/side` diğer adı yerine bunu içerir. `/login` her yerde gösterilebilir, ancak eşleştirme kodlarını yalnızca özel sohbetlerde veya Web Kullanıcı Arayüzü'nde verir.

    Mevcut `features.slash_commands` bölümünüzü [kullanılabilir komutların](/tr/tools/slash-commands#command-list) bir alt kümesiyle değiştirin:

    <Tabs>
      <Tab title="Socket Mode (varsayılan)">

```json
{
  "slash_commands": [
    {
      "command": "/new",
      "description": "Yeni bir oturum başlat",
      "usage_hint": "[model]"
    },
    {
      "command": "/reset",
      "description": "Geçerli oturumu sıfırla"
    },
    {
      "command": "/compact",
      "description": "Oturum bağlamını sıkıştır",
      "usage_hint": "[instructions]"
    },
    {
      "command": "/stop",
      "description": "Geçerli çalıştırmayı durdur"
    },
    {
      "command": "/session",
      "description": "İleti dizisi bağlama süresinin dolmasını yönet",
      "usage_hint": "boşta <duration|off> veya azami yaş <duration|off>"
    },
    {
      "command": "/think",
      "description": "Düşünme düzeyini ayarla",
      "usage_hint": "<level>"
    },
    {
      "command": "/verbose",
      "description": "Ayrıntılı çıktıyı aç veya kapat",
      "usage_hint": "on|off|full"
    },
    {
      "command": "/fast",
      "description": "Hızlı modu göster veya ayarla",
      "usage_hint": "[status|on|off]"
    },
    {
      "command": "/reasoning",
      "description": "Akıl yürütme görünürlüğünü aç veya kapat",
      "usage_hint": "[on|off|stream]"
    },
    {
      "command": "/elevated",
      "description": "Yükseltilmiş modu aç veya kapat",
      "usage_hint": "[on|off|ask|full]"
    },
    {
      "command": "/exec",
      "description": "Çalıştırma varsayılanlarını göster veya ayarla",
      "usage_hint": "host=<auto|sandbox|gateway|node> security=<deny|allowlist|full> ask=<off|on-miss|always> node=<id>"
    },
    {
      "command": "/approve",
      "description": "Bekleyen onay isteklerini onayla veya reddet",
      "usage_hint": "<id> <decision>"
    },
    {
      "command": "/model",
      "description": "Modeli göster veya ayarla",
      "usage_hint": "[name|#|status]"
    },
    {
      "command": "/models",
      "description": "Sağlayıcıları/modelleri listele",
      "usage_hint": "[provider] [page] [limit=<n>|size=<n>|all]"
    },
    {
      "command": "/help",
      "description": "Kısa yardım özetini göster"
    },
    {
      "command": "/commands",
      "description": "Oluşturulan komut kataloğunu göster"
    },
    {
      "command": "/tools",
      "description": "Geçerli aracının şu anda neleri kullanabileceğini göster",
      "usage_hint": "[compact|verbose]"
    },
    {
      "command": "/agentstatus",
      "description": "Kullanılabilir olduğunda sağlayıcı kullanımı/kotası dâhil çalışma zamanı durumunu göster"
    },
    {
      "command": "/tasks",
      "description": "Geçerli oturumun etkin/yakın tarihli arka plan görevlerini listele"
    },
    {
      "command": "/context",
      "description": "Bağlamın nasıl oluşturulduğunu açıkla",
      "usage_hint": "[list|detail|json]"
    },
    {
      "command": "/whoami",
      "description": "Gönderen kimliğinizi göster"
    },
    {
      "command": "/skill",
      "description": "Bir beceriyi adına göre çalıştır",
      "usage_hint": "<name> [input]"
    },
    {
      "command": "/btw",
      "description": "Oturum bağlamını değiştirmeden bir yan soru sor",
      "usage_hint": "<question>"
    },
    {
      "command": "/login",
      "description": "Codex oturum açma işlemini eşleştir",
      "usage_hint": "[codex|openai]"
    },
    {
      "command": "/usage",
      "description": "Kullanım alt bilgisini denetle veya maliyet özetini göster",
      "usage_hint": "off|tokens|full|cost"
    }
  ]
}
```

      </Tab>
      <Tab title="HTTP İstek URL'leri">
        Yukarıdaki Socket Mode ile aynı `slash_commands` listesini kullanın ve her girdiye `"url": "https://gateway-host.example.com/slack/events"` ekleyin. Örnek:

```json
{
  "slash_commands": [
    {
      "command": "/new",
      "description": "Yeni bir oturum başlat",
      "usage_hint": "[model]",
      "url": "https://gateway-host.example.com/slack/events"
    },
    {
      "command": "/help",
      "description": "Kısa yardım özetini göster",
      "url": "https://gateway-host.example.com/slack/events"
    }
  ]
}
```

        Bu `url` değerini listedeki her komutta tekrarlayın.

      </Tab>
    </Tabs>

  </Accordion>
  <Accordion title="İsteğe bağlı yazarlık kapsamları (yazma işlemleri)">
    Giden iletilerin varsayılan Slack uygulama kimliği yerine etkin aracı kimliğini (özel kullanıcı adı ve simge) kullanmasını istiyorsanız `chat:write.customize` bot kapsamını ekleyin.

    Emoji simgesi kullanırsanız Slack, `:emoji_name:` söz dizimini bekler.

  </Accordion>
  <Accordion title="İsteğe bağlı kullanıcı belirteci kapsamları (okuma işlemleri)">
    `channels.slack.userToken` yapılandırırsanız tipik okuma kapsamları şunlardır:

    - `channels:history`, `groups:history`, `im:history`, `mpim:history`
    - `channels:read`, `groups:read`, `im:read`, `mpim:read`
    - `users:read`
    - `reactions:read`
    - `pins:read`
    - `emoji:read`
    - `search:read` (Slack arama okumalarına bağlıysanız)

  </Accordion>
</AccordionGroup>

## Belirteç modeli

- Bot kimliği (varsayılan), Socket Mode için `botToken` + `appToken`; HTTP modu için ise `botToken` + `signingSecret` gerektirir.
- Kullanıcı kimliği, Socket Mode için `userToken` + `appToken`; HTTP modu için ise `userToken` + `signingSecret` gerektirir. Bot belirteci kullanmaz.
- Aktarma modu, `botToken` ile birlikte `relay.url`, `relay.authToken` ve `relay.gatewayId` gerektirir; uygulama belirteci veya imzalama sırrı kullanmaz.
- `botToken`, `appToken`, `signingSecret`, `relay.authToken` ve `userToken` düz metin
  dizelerini veya SecretRef nesnelerini kabul eder.
- Yapılandırma belirteçleri, ortam geri dönüşünü geçersiz kılar.
- `SLACK_BOT_TOKEN`, `SLACK_APP_TOKEN` ve `SLACK_USER_TOKEN` ortam geri dönüşlerinin her biri yalnızca varsayılan hesap için geçerlidir.
- `userToken` varsayılan olarak salt okunur davranışı (`userTokenReadOnly: true`) kullanır.

Durum anlık görüntüsü davranışı:

- Slack hesap incelemesi, kimlik bilgisi başına `*Source` ve `*Status`
  alanlarını (`botToken`, `appToken`, `signingSecret`, `userToken`) izler.
- Durum `available`, `configured_unavailable` veya `missing` değeridir.
- `configured_unavailable`, hesabın SecretRef veya satır içi olmayan başka bir sır kaynağı aracılığıyla yapılandırıldığı, ancak geçerli komut/çalışma zamanı yolunun gerçek değeri
  çözümleyemediği anlamına gelir.
- HTTP modunda `signingSecretStatus` dâhil edilir. Socket Mode, bot kimliği için
  `botTokenStatus` + `appTokenStatus`, kullanıcı kimliği için ise
  `userTokenStatus` + `appTokenStatus` kullanır.

<Tip>
Bot kimliği için eylemler ve dizin okumaları isteğe bağlı bir kullanıcı belirtecini tercih edebilir; `userTokenReadOnly: false` geri dönüşe izin vermediği sürece yazma işlemleri bot belirtecini kullanmaya devam eder. `identity: "user"` için okuma ve yazma işlemleri her zaman `userToken` kullanır.
</Tip>

## Eylemler ve geçitler

Slack eylemleri `channels.slack.actions.*` tarafından denetlenir.

Geçerli Slack araçlarında kullanılabilen eylem grupları:

| Grup       | Varsayılan |
| ---------- | ---------- |
| messages   | etkin      |
| reactions  | etkin      |
| pins       | etkin      |
| memberInfo | etkin      |
| emojiList  | etkin      |

Geçerli Slack ileti eylemleri arasında `send`, `upload-file`, `download-file`, `read`, `edit`, `delete`, `pin`, `unpin`, `list-pins`, `member-info` ve `emoji-list` bulunur. `download-file`, gelen dosya yer tutucularında gösterilen Slack dosya kimliklerini kabul eder ve görüntüler için görüntü önizlemelerini, diğer dosya türleri içinse yerel dosya meta verilerini döndürür.

## Erişim denetimi ve yönlendirme

<Tabs>
  <Tab title="DM politikası">
    `channels.slack.dmPolicy`, DM erişimini denetler. `channels.slack.allowFrom`, standart DM izin listesidir.

    - `pairing` (varsayılan)
    - `allowlist`
    - `open` (`channels.slack.allowFrom` değerinin `"*"` içermesini gerektirir)
    - `disabled`

    DM bayrakları:

    - `dm.enabled` (varsayılan olarak true)
    - `channels.slack.allowFrom`
    - `dm.allowFrom` (eski)
    - `dm.groupEnabled` (grup DM'lerinde varsayılan olarak false)
    - `dm.groupChannels` (isteğe bağlı MPIM izin listesi)

    Çoklu hesap önceliği:

    - `channels.slack.accounts.default.allowFrom` yalnızca `default` hesabına uygulanır.
    - Adlandırılmış hesaplar, kendi `allowFrom` değerleri ayarlanmamışsa `channels.slack.allowFrom` değerini devralır.
    - Adlandırılmış hesaplar `channels.slack.accounts.default.allowFrom` değerini devralmaz.

    Eski `channels.slack.dm.policy` ve `channels.slack.dm.allowFrom`, uyumluluk amacıyla hâlâ okunur. `openclaw doctor --fix`, erişimi değiştirmeden yapabildiğinde bunları `dmPolicy` ve `allowFrom` değerlerine taşır.

    DM'lerde eşleştirme `openclaw pairing approve slack <code>` kullanır.

  </Tab>

  <Tab title="Kanal politikası">
    `channels.slack.groupPolicy`, kanal işlemeyi denetler:

    - `open`
    - `allowlist`
    - `disabled`

    Kanal izin listesi `channels.slack.channels` altında bulunur ve yapılandırma anahtarları olarak **kararlı Slack kanal kimliklerini kullanmalıdır** (örneğin `C12345678`).

    Çalışma zamanı notu: `channels.slack` tamamen eksikse (yalnızca ortam değişkenleriyle kurulum), çalışma zamanı `groupPolicy="allowlist"` değerine geri döner ve (`channels.defaults.groupPolicy` ayarlanmış olsa bile) bir uyarı kaydeder.

    Ad/kimlik çözümleme:

    - kanal izin listesi girdileri ve DM izin listesi girdileri, belirteç erişimi izin verdiğinde başlangıçta çözümlenir
    - çözümlenemeyen kanal adı girdileri yapılandırıldıkları biçimde korunur ancak varsayılan olarak yönlendirmede yok sayılır
    - gelen yetkilendirme ve kanal yönlendirme varsayılan olarak önce kimliğe göre yapılır; doğrudan kullanıcı adı/kısa ad eşleştirmesi `channels.slack.dangerouslyAllowNameMatching: true` gerektirir

    <Warning>
    Ada dayalı anahtarlar (`#channel-name` veya `channel-name`), `groupPolicy: "allowlist"` altında **eşleşmez**. Kanal araması varsayılan olarak önce kimliğe göre yapıldığından, ada dayalı bir anahtar hiçbir zaman başarıyla yönlendirilmez ve o kanaldaki tüm mesajlar sessizce engellenir. Bu durum, kanal anahtarının yönlendirme için gerekli olmadığı ve ada dayalı bir anahtarın çalışıyor gibi göründüğü `groupPolicy: "open"` davranışından farklıdır.

    Anahtar olarak her zaman Slack kanal kimliğini kullanın. Kimliği bulmak için: Slack'te kanala sağ tıklayın → **Copy link** — kimlik (`C...`) URL'nin sonunda görünür.

    Doğru:

    ```json5
    {
      channels: {
        slack: {
          groupPolicy: "allowlist",
          channels: {
            C12345678: { enabled: true, requireMention: true },
          },
        },
      },
    }
    ```

    Yanlış (`groupPolicy: "allowlist"` altında sessizce engellenir):

    ```json5
    {
      channels: {
        slack: {
          groupPolicy: "allowlist",
          channels: {
            "#eng-my-channel": { enabled: true, requireMention: true },
          },
        },
      },
    }
    ```
    </Warning>

  </Tab>

  <Tab title="Bahsetmeler ve kanal kullanıcıları">
    Kanal mesajları varsayılan olarak bahsetme koşuluna tabidir.

    Bahsetme kaynakları:

    - açık uygulama bahsetmesi (`<@botId>`)
    - bot kullanıcısı ilgili kullanıcı grubunun üyesiyse Slack kullanıcı grubu bahsetmesi (`<!subteam^S...>`); `usergroups:read` gerektirir
    - bahsetme regex kalıpları (`agents.entries.*.groupChat.mentionPatterns`, geri dönüş olarak `messages.groupChat.mentionPatterns`)
    - botun kendi Slack mesajına verilen yanıtlar (`implicitMentions.replyToBot`)
    - botun katıldığı ileti dizilerindeki takip mesajları (`implicitMentions.threadParticipation`)

    Kanal başına denetimler (`channels.slack.channels.<id>`; adlar yalnızca başlangıç çözümlemesi veya `dangerouslyAllowNameMatching` aracılığıyla):

    - `requireMention`
    - `ignoreOtherMentions`
    - `replyToMode` (`off|first|all|batched`; bu kanal için hesap/sohbet türü yanıt modunu geçersiz kılar)
    - `users` (izin listesi)
    - `allowBots`
    - `skills`
    - `systemPrompt`
    - `tools`, `toolsBySender`
    - `toolsBySender` anahtar biçimi: `channel:`, `id:`, `e164:`, `username:`, `name:` veya `"*"` joker karakteri
      (önek içermeyen eski anahtarlar hâlâ yalnızca `id:` ile eşlenir)

    `ignoreOtherMentions` (varsayılan `false`), başka bir kullanıcıdan veya kullanıcı grubundan bahsedip bu bottan bahsetmeyen kanal mesajlarını atar. DM'ler ve grup DM'leri (MPIM'ler) etkilenmez. Filtre, `auth.test` üzerinden çözümlenmiş bir bot kullanıcı kimliği gerektirir; bu kimlik kullanılamıyorsa (örneğin yalnızca kullanıcı belirtecine sahip bir kimlik), geçit açık kalır ve mesajlar değiştirilmeden geçer.

    `allowBots`, kanallar ve özel kanallar için ihtiyatlı davranır: bot tarafından oluşturulan oda mesajları yalnızca gönderen bot o odanın `users` izin listesinde açıkça yer alıyorsa veya `channels.slack.allowFrom` içindeki en az bir açık Slack sahibi kimliği o anda odanın üyesiyse kabul edilir. Joker karakterler ve görünen ad biçimindeki sahip girdileri, sahip bulunma koşulunu karşılamaz. Sahip bulunma denetimi Slack `conversations.members` kullanır; uygulamanın oda türü için uygun okuma kapsamına sahip olduğundan emin olun (herkese açık kanallar için `channels:read`, özel kanallar için `groups:read`). Üye araması başarısız olursa OpenClaw, bot tarafından oluşturulan oda mesajını atar.

    Kabul edilen, bot tarafından oluşturulmuş Slack mesajları ortak [bot döngüsü korumasını](/tr/channels/bot-loop-protection) kullanır. Varsayılan bütçe için `channels.defaults.botLoopProtection` yapılandırın, ardından bir çalışma alanı veya kanal farklı bir sınır gerektirdiğinde `channels.slack.botLoopProtection` ya da `channels.slack.channels.<id>.botLoopProtection` ile geçersiz kılın.

  </Tab>
</Tabs>

## İleti dizileri, oturumlar ve yanıt etiketleri

- DM'ler `direct`; kanallar `channel`; MPIM'ler ise `group` olarak yönlendirilir.
- Slack yönlendirme bağlamaları, ham eş kimliklerinin yanı sıra `channel:C12345678`, `user:U12345678` ve `<@U12345678>` gibi Slack hedef biçimlerini kabul eder.
- Varsayılan `session.dmScope=main` ile sıradan Slack DM'leri ana ajan oturumunda birleştirilir. Agent View kökleri ve mevcut Assistant View ileti dizileri, `:thread:<threadTs>` oturumları olarak yalıtılmış kalır.
- Kanal oturumları: `agent:<agentId>:slack:channel:<channelId>`.
- Sıradan üst düzey kanal mesajları, `replyToMode` değeri `off` olmasa bile kanal başına oturumda kalır.
- Slack kanalı, MPIM, Agent View ve Assistant View ileti dizisi yanıtları, oturum son ekleri için üst Slack `thread_ts` değerini kullanır (`:thread:<threadTs>`). Sıradan DM yanıt dizileri, temel DM oturumunda bir kullanıcı arayüzü olanağı olarak kalır.
- OpenClaw, görünür bir Slack ileti dizisi başlatması beklenen uygun bir üst düzey kanal kökünü `agent:<agentId>:slack:channel:<channelId>:thread:<rootTs>` içine başlangıç verisi olarak ekler; böylece kök ve sonraki ileti dizisi yanıtları aynı OpenClaw oturumunu paylaşır. Bu, `app_mention` olayları, açık bot veya yapılandırılmış bahsetme kalıbı eşleşmeleri ve `requireMention: false` olup `replyToMode` değeri `off` olmayan kanallar için geçerlidir.
- `channels.slack.thread.historyScope` varsayılanı `thread`; `thread.inheritParent` varsayılanı `false` değeridir.
- `channels.slack.thread.initialHistoryLimit`, yeni bir ileti dizisi oturumu başladığında mevcut ileti dizisinden kaç mesajın getirileceğini denetler (varsayılan `20`; devre dışı bırakmak için `0` olarak ayarlayın).
- `channels.slack.implicitMentions.replyToBot`, botun kendi mesajına verilen bir yanıtın bahsetme geçidini atlayıp atlamayacağını denetler (varsayılan `true`).
- `channels.slack.implicitMentions.threadParticipation`, botun yanıt verdiği bir ileti dizisindeki takip mesajlarının bahsetme geçidini atlayıp atlamayacağını denetler (varsayılan `true`). Bu takip mesajlarında yeni ve açık bir bahsetme gerektirmek için `false` olarak ayarlayın. `openclaw doctor --fix`, eski `channels.slack.thread.requireExplicitMention` anahtarını bu pozitif standart bayrağa taşır.
- Hesap geçersiz kılmaları `channels.slack.accounts.<id>.implicitMentions`; ortak varsayılanlar `channels.defaults.implicitMentions` altında bulunur.

Yanıt ileti dizisi denetimleri:

- `channels.slack.channels.<id>.replyToMode`: Slack kanalı/özel kanal mesajları için kanal başına geçersiz kılma
- `channels.slack.replyToMode`: `off|first|all|batched` (varsayılan `off`)
- `channels.slack.replyToModeByChatType`: `direct|group|channel` başına
- doğrudan sohbetler için eski geri dönüş: `channels.slack.dm.replyToMode`

Elle belirtilen yanıt etiketleri desteklenir:

- `[[reply_to_current]]`
- `[[reply_to:<id>]]`

`message` aracından açık Slack ileti dizisi yanıtları göndermek için `replyBroadcast: true` değerini `action: "send"` ile ve `threadId` ya da `replyTo` değerini ayarlayarak Slack'ten ileti dizisi yanıtını üst kanalda da yayımlamasını isteyin. Bu, Slack'in `chat.postMessage` `reply_broadcast` bayrağıyla eşlenir ve medya yüklemelerinde değil, yalnızca metin veya Block Kit gönderimlerinde desteklenir.

Bir `message` araç çağrısı Slack ileti dizisi içinde çalışıp aynı kanalı hedeflediğinde OpenClaw, normalde geçerli hesap, sohbet türü veya kanal başına `replyToMode` ayarına göre mevcut Slack ileti dizisini devralır. Otomatik yanıtlar ve aynı kanala yönelik `send` ya da `upload-file` çağrıları aynı kanal başına geçersiz kılmayı kullanır. Bunun yerine üst kanalda yeni bir mesaj göndermeyi zorlamak için `action: "send"` veya `action: "upload-file"` üzerinde `topLevel: true` ayarlayın. `threadId: null` aynı üst düzey devre dışı bırakma seçeneği olarak kabul edilir.

<Note>
`replyToMode="off"`, açık `[[reply_to_*]]` etiketleri dâhil olmak üzere isteğe bağlı giden Slack yanıt ileti dizilerini devre dışı bırakır. Agent View ve Assistant View, Slack tarafından yönetilen ileti dizisi deneyimleridir; bu nedenle yanıtları ve durumları bu ayardan bağımsız olarak görünür kökte kalır. Diğer gelen Slack ileti dizisi oturumlarını düzleştirmez. Bu davranış, açık etiketlerin `"off"` modunda da uygulandığı Telegram'dan farklıdır. Slack ileti dizileri mesajları kanaldan gizlerken Telegram yanıtları satır içinde görünür kalır.
</Note>

## Onay tepkileri

`ackReaction`, OpenClaw gelen bir mesajı işlerken bir onay emojisi gönderir. `ackReactionScope`, bu emojinin gerçekte _ne zaman_ gönderileceğini belirler.

Varsayılan olarak onay emojisi sabit kalırken Slack'in yerel ajan/asistan ileti dizisi durumu, dönüşümlü yükleme mesajlarıyla ilerlemeyi gösterir. Bunun yerine kuyruğa alındı/düşünüyor/araç/tamamlandı/hata tepki yaşam döngüsünü etkinleştirmek için `messages.statusReactions.enabled: true` ayarlayın.

### Emoji (`ackReaction`)

Çözümleme sırası:

- `channels.slack.accounts.<accountId>.ackReaction`
- `channels.slack.ackReaction`
- `messages.ackReaction`
- ajan kimliği emojisi geri dönüşü (`agents.entries.*.identity.emoji`, aksi takdirde `"eyes"` / 👀)

Notlar:

- Slack kısa kodlar bekler (örneğin `"eyes"`).
- Slack hesabı için veya genel olarak tepkiyi devre dışı bırakmak üzere `""` kullanın.

### Kapsam (`messages.ackReactionScope`)

Slack sağlayıcısı kapsamı `messages.ackReactionScope` üzerinden okur (varsayılan `"group-mentions"`). Şu anda Slack hesabı veya Slack kanalı düzeyinde geçersiz kılma yoktur; değer Gateway genelinde geçerlidir.

Değerler:

- `"all"`: ortam oda olayları dâhil olmak üzere DM'lerde ve gruplarda tepki ver.
- `"direct"`: yalnızca DM'lerde tepki ver.
- `"group-all"`: ortam oda olayları dışında her grup mesajına tepki ver (DM'ler hariç).
- `"group-mentions"` (varsayılan): gruplarda yalnızca bottan bahsedildiğinde (veya katılımı etkinleştirmiş grup bahsedilebilirlerinde) tepki ver. **DM'ler hariç tutulur.**
- `"off"` / `"none"`: hiçbir zaman tepki verme.

<Note>
Varsayılan kapsam (`"group-mentions"`), doğrudan mesajlarda veya ortam oda olaylarında onay tepkilerini tetiklemez. Yapılandırılmış `ackReaction` değerini (örneğin `"eyes"`) gelen Slack DM'lerinde ve sessiz oda olaylarında görmek için `messages.ackReactionScope` değerini `"all"` olarak ayarlayın. `messages.ackReactionScope`, Slack sağlayıcısı başlatılırken okunur; bu nedenle değişikliğin etkili olması için Gateway'in yeniden başlatılması gerekir.
</Note>

```json5
{
  messages: {
    ackReaction: "eyes",
    ackReactionScope: "all", // DM'lerde ve gruplarda tepki ver
  },
}
```

## Metin akışı

`channels.slack.streaming`, canlı önizleme davranışını denetler:

- `off`: canlı önizleme akışını devre dışı bırak.
- `partial` (varsayılan): önizleme metnini en son kısmi çıktıyla değiştir.
- `block`: parçalı önizleme güncellemelerini sona ekle.
- `progress`: oluşturma sırasında ilerleme durumu metnini göster, ardından son metni gönder.
- `streaming.preview.toolProgress`: taslak önizlemesi etkinken araç/ilerleme güncellemelerini düzenlenen aynı önizleme mesajına yönlendir (varsayılan: `true`). Araç/ilerleme mesajlarını ayrı tutmak için `false` ayarlayın.
- `streaming.preview.commandText` / `streaming.progress.commandText`: ham komut/çalıştırma metnini gizlerken kompakt araç ilerleme satırlarını korumak için `status` olarak ayarlayın (varsayılan: `raw`).

Kompakt ilerleme satırlarını korurken ham komut/çalıştırma metnini gizleyin:

```json
{
  "channels": {
    "slack": {
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

`channels.slack.streaming.nativeTransport`, `channels.slack.streaming.mode` değeri `partial` olduğunda Slack'in yerel metin akışını denetler (varsayılan: `true`).

Slack'in yerel ilerleme görev kartları, ilerleme modu için isteğe bağlıdır. Çalışma sürerken Slack'e özgü bir plan/görev kartı göndermek ve tamamlandığında aynı görev kartını güncellemek için `channels.slack.streaming.progress.nativeTaskCards` değerini `true` olarak `channels.slack.streaming.mode="progress"` ile ayarlayın. Bu bayrak olmadan ilerleme modu, taşınabilir taslak önizleme davranışını korur.

- Yerel metin akışının ve Slack asistan ileti dizisi durumunun görünmesi için bir yanıt ileti dizisi kullanılabilir olmalıdır. İleti dizisi seçimi yine `replyToMode` kuralını izler.
- Kanal, grup sohbeti ve üst düzey DM kökleri, yerel akış kullanılamadığında veya yanıt ileti dizisi bulunmadığında normal taslak önizlemesini kullanmaya devam edebilir.
- Üst düzey Slack DM'leri varsayılan olarak ileti dizisi dışında kalır; bu nedenle Slack'in ileti dizisi tarzındaki yerel akış/durum önizlemesini göstermez. Bunun yerine OpenClaw, DM'de bir taslak önizlemesi gönderir ve düzenler.
- Medya ve metin dışı yükler normal teslimata geri döner.
- Medya/hata nihai sonuçları bekleyen önizleme düzenlemelerini iptal eder; uygun metin/blok nihai sonuçları yalnızca önizlemeyi yerinde düzenleyebildiklerinde gönderilir.
- Akış yanıtın ortasında başarısız olursa OpenClaw, kalan yükler için normal teslimata geri döner.

Slack yerel metin akışı yerine taslak önizlemesini kullanın:

```json5
{
  channels: {
    slack: {
      streaming: {
        mode: "partial",
        nativeTransport: false,
      },
    },
  },
}
```

Slack yerel ilerleme görev kartlarını etkinleştirin:

```json5
{
  channels: {
    slack: {
      streaming: {
        mode: "progress",
        progress: {
          nativeTaskCards: true,
          render: "rich",
        },
      },
    },
  },
}
```

Eski anahtarlar:

- `channels.slack.streamMode` (`replace | status_final | append`), `channels.slack.streaming.mode` için eski bir diğer addır.
- boolean `channels.slack.streaming`, `channels.slack.streaming.mode` ve `channels.slack.streaming.nativeTransport` için eski bir diğer addır.
- Üst düzey `channels.slack.chunkMode` ve `channels.slack.nativeStreaming`, `channels.slack.streaming.chunkMode` ve `channels.slack.streaming.nativeTransport` için eski diğer adlardır.
- Eski diğer adlar çalışma zamanında okunmaz; kalıcı Slack akış yapılandırmasını standart anahtarlarla yeniden yazmak için `openclaw doctor --fix` komutunu çalıştırın.

## Yazıyor tepkisi geri dönüşü

`typingReaction`, OpenClaw bir yanıtı işlerken gelen Slack mesajına geçici bir tepki ekler ve çalıştırma tamamlandığında bunu kaldırır. Bu özellik en çok, varsayılan bir "yazıyor..." durum göstergesi kullanan ileti dizisi yanıtlarının dışında yararlıdır.

Çözümleme sırası:

- `channels.slack.accounts.<accountId>.typingReaction`
- `channels.slack.typingReaction`

Notlar:

- Slack kısa kodlar bekler (örneğin `"hourglass_flowing_sand"`).
- Tepki, mümkün olan en iyi şekilde uygulanır ve yanıt ya da hata yolu tamamlandıktan sonra otomatik olarak temizlenmeye çalışılır.

## Sesli giriş

Bugün Slack'te OpenClaw ile konuşmak için OpenClaw uygulamasına bir Slack ses klibi gönderin. Slackbot'un dikte mikrofonu, uygulama API'si olmayan ve Slack'e ait ayrı bir özelliktir.

- **[Slackbot sesli diktesi](https://slack.com/help/articles/202026038-How-to-use-Slackbot)** kullanıcının özel Slackbot görüşmesinde bulunur. Slack, kaydı bir Slackbot istemine dönüştürür ancak Events API aracılığıyla üçüncü taraf Slack uygulamalarına ses dosyası, dikte olayı, istem veya giriş kaynağı işaretçisi göndermez. OpenClaw Slack Plugin'i bunu etkinleştiremez veya alamaz.
- **[Slack ses klipleri](https://slack.com/help/articles/4406235165587-Record-audio-and-video-clips-in-Slack)**, OpenClaw DM'sinde, kanalında veya ileti dizisinde gönderilebilen, Slack'te depolanan dosyalardır. OpenClaw, erişilebilir bir klibi bot belirteciyle indirir, Slack'in klip MIME meta verilerini normalleştirir ve ortak [ses transkripsiyonu işlem hattına](/tr/nodes/audio) gönderir. Önerilen uygulama manifesti, gerekli `files:read` kapsamını içerir.

Ses klipleri ile Slackbot diktesinin gizlilik açısından farklı anlamları vardır: Klipler Slack dosya saklama politikasına tabidir ve OpenClaw bunları transkripsiyon için indirir; Slack ise dikte sesinin depolanmadığını belirtir.

`requireMention: true` bulunan bir kanalda, altyazısız bir ses klibi yapılandırılmış bir bahsetme kalıbının seslendirilmesiyle eşiği karşılayabilir (`agents.entries.*.groupChat.mentionPatterns`, yoksa `messages.groupChat.mentionPatterns` kullanılır). OpenClaw, klibi indirmeden veya yazıya dökmeden önce göndereni yetkilendirir; ardından yalnızca transkript eşleşirse klibi kabul eder. Başarısız veya eşleşmeyen spekülatif bir transkript, indirilen kliple birlikte atılır; kanal geçmişinde saklanmaz. Yerel Slack `@bot` kimliği konuşmadan çıkarılamaz; bu nedenle seslendirilen bir ad kalıbı yapılandırın veya yazılı bir bahsetme ekleyin. Transkript yankılama etkinse yankı yalnızca kabulden sonra gönderilir.

## Medya, parçalara ayırma ve teslimat

<AccordionGroup>
  <Accordion title="Gelen ekler">
    Slack dosya ekleri, Slack tarafından barındırılan özel URL'lerden indirilir (belirteçle kimliği doğrulanan istek akışı) ve getirme başarılı olduğunda ve boyut sınırları izin verdiğinde medya deposuna yazılır. Dosya yer tutucuları, aracıların özgün dosyayı `download-file` ile getirebilmesi için Slack `fileId` değerini içerir.

    İndirmeler sınırlı boşta kalma ve toplam zaman aşımları kullanır. Slack dosyasının alınması duraklar veya başarısız olursa OpenClaw mesajı işlemeye devam eder ve dosya yer tutucusuna geri döner.

    Çalışma zamanındaki gelen boyut üst sınırı, `channels.slack.mediaMaxMb` tarafından geçersiz kılınmadığı sürece varsayılan olarak `20MB` değeridir.

  </Accordion>

  <Accordion title="Giden metin ve dosyalar">
    - Metin parçaları `channels.slack.textChunkLimit` kullanır (varsayılan `8000`, Slack'in kendi mesaj uzunluğu sınırıyla kısıtlanır)
    - `channels.slack.streaming.chunkMode="newline"` önce paragrafa göre bölmeyi etkinleştirir
    - Dosya gönderimleri Slack yükleme API'lerini kullanır ve ileti dizisi yanıtları (`thread_ts`) içerebilir
    - Uzun dosya açıklamaları, yükleme yorumu olarak Slack açısından güvenli ilk metin parçasını kullanır ve kalan parçaları takip mesajları olarak gönderir
    - Giden medya üst sınırı, yapılandırılmışsa `channels.slack.mediaMaxMb` değerini izler; aksi takdirde kanal gönderimleri medya işlem hattındaki MIME türü varsayılanlarını kullanır

  </Accordion>

  <Accordion title="Teslimat hedefleri">
    Tercih edilen açık hedefler:

    - DM'ler için `user:<id>`
    - Kanallar için `channel:<id>`

    Yalnızca metin/blok içeren Slack DM'leri doğrudan kullanıcı kimliklerine gönderilebilir; dosya yüklemeleri ve ileti dizisi gönderimleri ise bu yollar somut bir görüşme kimliği gerektirdiğinden önce Slack görüşme API'leri aracılığıyla DM'yi açar.

  </Accordion>
</AccordionGroup>

## Komutlar ve eğik çizgi davranışı

Eğik çizgi komutları Slack'te tek bir yapılandırılmış komut veya birden fazla yerel komut olarak görünür. Komut varsayılanlarını değiştirmek için `channels.slack.slashCommand` yapılandırmasını ayarlayın:

- `enabled: false`
- `name: "openclaw"`
- `sessionPrefix: "slack:slash"`
- `ephemeral: true`

```txt
/openclaw /help
```

Yerel komutlar, Slack uygulamanızda [ek manifest ayarları](#additional-manifest-settings) gerektirir ve bunun yerine genel yapılandırmalardaki `channels.slack.commands.native: true` veya `commands.native: true` ile etkinleştirilir.

- Yerel komut otomatik modu Slack için **kapalıdır**; bu nedenle `commands.native: "auto"`, Slack yerel komutlarını etkinleştirmez.

```txt
/help
```

Yerel bağımsız değişken menüleri öncelik sırasına göre aşağıdakilerden biri olarak oluşturulur:

- Yeterince kısa 3-5 seçenek: taşma ("...") menüsü
- 100'den fazla seçenek ve zaman uyumsuz seçenek filtreleme kullanılabilir: harici seçim
- 1-2 seçenek veya kodlanmış değeri seçim için fazla uzun olan herhangi bir seçenek: düğme blokları
- Diğer durumlarda (6-100 seçenek veya zaman uyumsuz filtreleme olmadan 100'den fazla seçenek): menü başına 100 seçenek olacak şekilde parçalara ayrılmış statik seçim menüsü

```txt
/think
```

Eğik çizgi oturumları `agent:<agentId>:slack:slash:<userId>` gibi yalıtılmış anahtarlar kullanır ve komut yürütmelerini yine `CommandTargetSessionKey` kullanarak hedef görüşme oturumuna yönlendirir.

## Yerel grafikler

Slack'in herkese açık [`data_visualization` Block Kit bloğu](https://docs.slack.dev/reference/block-kit/blocks/data-visualization-block/)
mesajlarda çizgi, çubuk, alan ve pasta grafiklerini oluşturur. OpenClaw, taşınabilir
`presentation` `chart` bloğunu bu yerel biçime eşler; normal
`chat:write` mesaj erişiminin ötesinde ek OAuth kapsamı, dosya yükleme,
görüntü oluşturucu veya Slack yapılandırması gerekmez.

```json
{
  "blocks": [
    {
      "type": "chart",
      "chartType": "bar",
      "title": "Üç aylık gelir",
      "categories": ["Q1", "Q2"],
      "series": [{ "name": "Gelir", "values": [120, 145] }],
      "xLabel": "Çeyrek"
    }
  ]
}
```

Slack'in sınırları yerel oluşturmadan önce uygulanır:

- Başlık ve isteğe bağlı eksen etiketleri: 50 karakter
- Pasta: 1-12 pozitif dilim
- Çizgi/çubuk/alan: benzersiz adlandırılmış 1-12 seri ve 1-20 ortak kategori
- Dilim, kategori ve seri etiketleri: 20 karakter
- Her seri, her kategori için bir sonlu değer içermelidir; pasta dışındaki değerler
  negatif olabilir

Her yerel grafik ayrıca ekran okuyucular, bildirimler, oturum yansıtma ve bloğu
oluşturamayan istemciler için üst düzey bir metin gösterimi taşır. Diğer OpenClaw
kanallarına yapılan standart sunum gönderimleri, yerel grafik desteğini bildirmedikleri
sürece aynı belirlenimci grafik verilerini metin olarak alır. Aşamalı kullanıma sunma
sırasında Slack grafiği `invalid_blocks` ile reddederse OpenClaw, reddedilen yerel
veri bloklarını kaldırır, varsa diğer denetimleri korur ve eksiksiz grafik gösterimini
görünür metin olarak gönderir.

Slack şu anda mesaj başına en fazla iki `data_visualization` bloğunu kabul eder. Bir
sunum ikiden fazla geçerli grafik içerdiğinde OpenClaw bunların sırasını korur ve
her mesajda en fazla iki grafik olacak şekilde takip mesajlarında yerel oluşturmaya
devam eder.

Slack'in [geliştirici duyurusu](https://docs.slack.dev/changelog/2026/06/16/block-kit-data-visualization-block/)
bloğu uygulamalara yönelik bir Block Kit özelliği olarak belgeler ve ücretli plan
kısıtlaması yayımlamaz. Business+/Enterprise uygunluk ifadesi, bir uygulamanın önceden
yapılandırılmış bir Block Kit grafiği göndermesinden ayrı olan Slackbot'un otomatik
yapay zekâ grafik üretimi için geçerlidir. Grafikler yalnızca mesaj bloklarıdır;
App Home, modal veya Canvas içeriği değildir.

## Yerel tablolar

Slack'in mevcut [`data_table` Block Kit bloğu](https://docs.slack.dev/reference/block-kit/blocks/data-table-block/)
mesajlarda yapılandırılmış satırları ve sütunları oluşturur. OpenClaw, açık bir
taşınabilir `presentation` `table` bloğunu `data_table` biçimine eşler; Slack'in
eski [`table` bloğunu](https://docs.slack.dev/reference/block-kit/blocks/table-block/)
kullanmaz. Normal `chat:write` mesaj erişiminin ötesinde ek OAuth kapsamı veya
Slack yapılandırması gerekmez.

```json
{
  "blocks": [
    {
      "type": "table",
      "caption": "Açık işlem hattı",
      "headers": ["Hesap", "Aşama", "ARR"],
      "rows": [
        ["Acme", "Kazanıldı", 125000],
        ["Globex", "İnceleme", 82000]
      ],
      "rowHeaderColumnIndex": 0
    }
  ]
}
```

OpenClaw, başlık ve dize hücrelerini Slack `raw_text` hücrelerine eşler. Sayısal
hücreler `raw_number` biçimine eşlenir ve sonlu sayısal değer, yerel sıralama ve
filtreleme için korunur. `rowHeaderColumnIndex`, mevcut olduğunda, sıfır tabanlı bu
sütunu Slack satır başlıkları olarak işaretler.

Slack'in yayımlanan `data_table` sınırları yerel oluşturmadan önce uygulanır:

- 1-20 sütun
- 1-100 veri satırı ve başlık satırı
- Her satırda aynı sayıda hücre
- Tek bir mesajdaki tüm tablo hücrelerinde toplam en fazla 10.000 karakter

Mesaj toplam karakter sınırı içinde kaldığı sürece birden fazla geçerli tablo bloğu
yerel olarak oluşturulabilir. Yerel zarf içinde oluşturulamayan bir tablo, satırları
veya hücreleri kaybetmek yerine eksiksiz belirlenimci metne dönüşür. Bu metin tek bir
Slack mesajını aşarsa gönderimler ve eğik çizgi yanıtları sıralı metin parçaları
kullanır. Tablo düzenlemeleri, mevcut bir mesajdaki satırları sessizce kesmek yerine
açık bir boyut hatasıyla başarısız olur.

Taşınabilir sunumdan üretilen her yerel tablo ayrıca ekran okuyucular, bildirimler, oturum yansıtma ve
bloğu işleyemeyen istemciler için üst düzey bir metin gösterimi taşır. Ham grafik ve tablo değerleri
yedek gösterimde değişmez kalır; böylece `<@U123>` gibi hücre verileri bir Slack bahsine dönüşmez.
Slack, yerel grafik veya tablo bloklarını `invalid_blocks` ile reddederse OpenClaw,
sınırlandırılmış tek bir kurtarma adımında tüm yerel veri bloklarını kaldırır, düğmeler ve seçimler gibi
geçerli eşdüzey blokları korur ve Slack biçimlendirmesi devre dışı bırakılmış olarak grafik ve tablo
metninin tamamını görünür biçimde gönderir. Eğik çizgi komutu teslimi, komut genelinde
Slack'in beş çağrılık `response_url` bütçesini izler. Her yanıt grubundan önce,
kalan çağrılara sığan eksiksiz bir plan seçer veya bu grubu göndermeden önce başarısız olur.

Yalnızca açık `presentation` tablo blokları yerel tablolara yükseltilir.
Markdown dikey çizgi tabloları, yazıldıkları biçimde metin olarak kalır; OpenClaw tablo
yapısını veya hücre türlerini tahmin etmez. Mevcut güvenilir Slack yerel üreticileri,
ham blokları `channelData.slack.blocks` üzerinden geçirmeye devam edebilir; OpenClaw geçerli ham
`data_table` hücrelerinden yedek metin türetirken, hatalı özel bloklar
başlıklarına veya genel Block Kit yedek gösterimine indirgenebilir. Taşınabilir ajan, CLI
ve plugin çıktısı `presentation` kullanmalıdır.

## Etkileşimli yanıtlar

Slack, ajan tarafından oluşturulan etkileşimli yanıt denetimlerini işleyebilir ancak bu özellik varsayılan olarak devre dışıdır.
Yeni ajan, CLI ve plugin çıktıları için paylaşılan
`presentation` düğmelerini veya seçim bloklarını tercih edin. Bunlar aynı Slack etkileşim
yolunu kullanırken diğer kanallarda da uygun bir yedek gösterime indirgenir.

Genel olarak etkinleştirin:

```json5
{
  channels: {
    slack: {
      capabilities: {
        interactiveReplies: true,
      },
    },
  },
}
```

Veya yalnızca bir Slack hesabı için etkinleştirin:

```json5
{
  channels: {
    slack: {
      accounts: {
        ops: {
          capabilities: {
            interactiveReplies: true,
          },
        },
      },
    },
  },
}
```

Etkinleştirildiğinde ajanlar, kullanımdan kaldırılmış yalnızca Slack'e özgü yanıt yönergelerini yine de yayımlayabilir:

- `[[slack_buttons: Approve:approve, Reject:reject]]`
- `[[slack_select: Choose a target | Canary:canary, Production:production]]`

Bu yönergeler Slack Block Kit'e derlenir ve tıklamaları veya seçimleri
mevcut Slack etkileşim olayı yolu üzerinden geri yönlendirir. Bunları eski
istemler ve Slack'e özgü kaçış yolları için koruyun; yeni taşınabilir denetimler için
paylaşılan sunumu kullanın.

Yönerge derleyici API'leri de yeni üretici kodları için kullanımdan kaldırılmıştır:

- `compileSlackInteractiveReplies(...)`
- `parseSlackOptionsLine(...)`
- `isSlackInteractiveRepliesEnabled(...)`
- `buildSlackInteractiveBlocks(...)`

Slack tarafından işlenen yeni denetimler için `presentation` yüklerini ve
`buildSlackPresentationBlocks(...)` kullanın.

Notlar:

- Bu, Slack'e özgü eski kullanıcı arayüzüdür. Diğer kanallar Slack Block
  Kit yönergelerini kendi düğme sistemlerine dönüştürmez.
- Etkileşimli geri çağırma değerleri, ajan tarafından yazılmış ham değerler değil, OpenClaw tarafından oluşturulan opak belirteçlerdir.
- Oluşturulan etkileşimli bloklar Slack Block Kit sınırlarını aşacaksa OpenClaw, geçersiz bir blok yükü göndermek yerine özgün metin yanıtına geri döner.

### Plugin'e ait modal gönderimleri

Etkileşimli bir işleyici kaydeden Slack pluginleri, OpenClaw yükü
ajanın görebildiği sistem olayı için sıkıştırmadan önce modal
`view_submission` ve `view_closed` yaşam döngüsü olaylarını da alabilir. Bir Slack modalı
açarken şu yönlendirme kalıplarından birini kullanın:

- `callback_id` değerini `openclaw:<namespace>:<payload>` olarak ayarlayın.
- Ya da mevcut bir `callback_id` değerini koruyun ve modalın `private_metadata` alanına `pluginInteractiveData:
"<namespace>:<payload>"` yerleştirin.

İşleyici, `ctx.interaction.kind` değerini `view_submission` veya
`view_closed` olarak, normalleştirilmiş `inputs` değerini ve Slack'ten gelen tam ham
`stateValues` nesnesini alır. Yalnızca geri çağırma kimliğine göre yönlendirme, plugin işleyicisini
çağırmak için yeterlidir; modalın ayrıca ajanın görebildiği bir sistem olayı üretmesi gerektiğinde
mevcut modal `private_metadata` kullanıcı/oturum yönlendirme alanlarını ekleyin. Ajan,
sıkıştırılmış ve gizli bilgileri çıkarılmış bir `Slack interaction: ...` sistem olayı alır. İşleyici
`systemEvent.summary`, `systemEvent.reference` veya `systemEvent.data` döndürürse bu
alanlar söz konusu sıkıştırılmış olaya eklenir; böylece ajan, form yükünün tamamını görmeden
plugin'e ait depolamaya başvurabilir.

## Slack'te yerel onaylar

Slack, Web kullanıcı arayüzüne veya terminale geri dönmek yerine etkileşimli düğmeler ve etkileşimlerle yerel bir onay istemcisi olarak çalışabilir.

- Exec ve plugin onayları, Slack'e özgü Block Kit istemleri olarak işlenebilir.
- `channels.slack.execApprovals.*`, yerel exec onay istemcisini etkinleştirme ve DM/kanal yönlendirme yapılandırması olarak kalır.
- Exec onayı DM'leri `channels.slack.execApprovals.approvers` veya `commands.ownerAllowFrom` kullanır.
- Slack, kaynak oturum için yerel onay istemcisi olarak etkinleştirildiğinde veya `approvals.plugin` kaynak Slack oturumuna ya da bir Slack hedefine yönlendirme yaptığında, plugin onayları Slack'e özgü düğmeleri kullanır.
- Plugin onayı DM'leri, `channels.slack.allowFrom` içindeki Slack plugin onaylayanlarını, adlandırılmış hesap `allowFrom` değerini veya hesabın varsayılan rotasını kullanır.
- Onaylayan yetkilendirmesi uygulanmaya devam eder: yalnızca exec onaylayabilenler, aynı zamanda plugin onaylayanı olmadıkça plugin isteklerini onaylayamaz.

Bu, diğer kanallarla aynı paylaşılan onay düğmesi yüzeyini kullanır. Slack uygulama ayarlarınızda `interactivity` etkinleştirildiğinde onay istemleri, konuşmanın doğrudan içinde Block Kit düğmeleri olarak işlenir.
Bu düğmeler mevcut olduğunda birincil onay kullanıcı deneyimi bunlardır; OpenClaw
yalnızca araç sonucu sohbet onaylarının kullanılamadığını veya tek yolun manuel onay olduğunu belirtiyorsa
manuel bir `/approve` komutu eklemelidir.

Yapılandırma yolu:

- `channels.slack.execApprovals.enabled`
- `channels.slack.execApprovals.approvers` (isteğe bağlıdır; mümkün olduğunda `commands.ownerAllowFrom` değerine geri döner)
- `channels.slack.execApprovals.target` (`dm` | `channel` | `both`, varsayılan: `dm`)
- `agentFilter`, `sessionFilter`

`enabled` ayarlanmamışsa veya `"auto"` ise ve en az bir
exec onaylayanı çözümlenirse Slack, yerel exec onaylarını otomatik olarak etkinleştirir. Slack plugin
onaylayanları çözümlendiğinde ve istek yerel istemci filtreleriyle eşleştiğinde Slack, bu yerel istemci
yolu üzerinden yerel plugin onaylarını da işleyebilir. Slack'i yerel onay istemcisi olarak açıkça devre
dışı bırakmak için `enabled: false` ayarlayın. Onaylayanlar çözümlendiğinde yerel onayları zorla
etkinleştirmek için `enabled: true` ayarlayın. Slack exec onaylarını devre dışı bırakmak,
`approvals.plugin` üzerinden etkinleştirilen yerel Slack plugin onayı teslimini devre dışı bırakmaz;
plugin onayı teslimi bunun yerine Slack plugin onaylayanlarını kullanır.

Açık bir Slack exec onayı yapılandırması olmadığındaki varsayılan davranış:

```json5
{
  commands: {
    ownerAllowFrom: ["slack:U12345678"],
  },
}
```

Açık Slack yerel yapılandırması yalnızca onaylayanları geçersiz kılmak, filtre eklemek veya
kaynak sohbete teslimi etkinleştirmek istediğinizde gereklidir:

```json5
{
  channels: {
    slack: {
      execApprovals: {
        enabled: true,
        approvers: ["U12345678"],
        target: "both",
      },
    },
  },
}
```

Paylaşılan `approvals.exec` yönlendirmesi ayrıdır. Bunu yalnızca exec onayı istemlerinin ayrıca
diğer sohbetlere veya açık bant dışı hedeflere yönlendirilmesi gerektiğinde kullanın. Paylaşılan
`approvals.plugin` yönlendirmesi de ayrıdır; Slack yerel teslimi, yalnızca Slack plugin onayı
isteğini yerel olarak işleyebildiğinde bu yedek gösterimi engeller.

Aynı sohbet içindeki `/approve`, komutları zaten destekleyen Slack kanallarında ve DM'lerde de çalışır. Eksiksiz onay yönlendirme modeli için [Exec onayları](/tr/tools/exec-approvals) bölümüne bakın.

## Olaylar ve işletim davranışı

- Mesaj düzenlemeleri/silmeleri sistem olaylarına eşlenir.
- İleti dizisi yayınları ("Also send to channel" ileti dizisi yanıtları) normal kullanıcı mesajları olarak işlenir.
- Tepki ekleme/kaldırma olayları sistem olaylarına eşlenir.
- Üye katılma/ayrılma, kanal oluşturma/yeniden adlandırma ve sabitleme ekleme/kaldırma olayları sistem olaylarına eşlenir.
- İsteğe bağlı iletişim durumu yoklaması, gözlemlenen bir insan katılımcının `away` durumundan `active` durumuna geçişini, katılımcının en son etkin olan uygun Slack oturumuna eşleyebilir. Varsayılan olarak kapalıdır.
- `configWrites` etkinleştirildiğinde `channel_id_changed` kanal yapılandırma anahtarlarını taşıyabilir.
- Kanal konusu/amacı meta verileri güvenilmeyen bağlam olarak değerlendirilir ve yönlendirme bağlamına eklenebilir.
- Agent View `app_context` varlıkları Slack uygunluk sırasına göre doğrulanır ve yalnızca yapılandırılmış güvenilmeyen bağlam olarak sunulur; bağlamın atlanması, eski varlıkları yeniden kullanmak yerine dönüşü temizler.
- İleti dizisi başlatıcısı ve ilk ileti dizisi geçmişi bağlamı yerleştirme işlemleri, geçerli olduğunda yapılandırılmış gönderen izin listelerine göre filtrelenir.
- Blok eylemleri, kısayollar ve modal etkileşimleri, zengin yük alanlarına sahip yapılandırılmış `Slack interaction: ...` sistem olayları yayımlar:
  - blok eylemleri: seçilen değerler, etiketler, seçici değerleri ve `workflow_*` meta verileri
  - genel kısayollar: geri çağırma ve aktör meta verileri; aktörün doğrudan oturumuna yönlendirilir
  - mesaj kısayolları: geri çağırma, aktör, kanal, ileti dizisi ve seçili mesaj bağlamı
  - yönlendirilmiş kanal meta verileri ve form girdileri içeren modal `view_submission` ve `view_closed` olayları

Slack uygulama yapılandırmanızda genel veya mesaj kısayolları tanımlayın ve boş olmayan herhangi bir geri çağırma kimliği kullanın. OpenClaw, eşleşen kısayol yüklerini onaylar, diğer Slack etkileşimleriyle aynı DM/kanal gönderen politikasını uygular ve arındırılmış olayı yönlendirilen ajan oturumu için kuyruğa alır. Tetikleyici kimlikleri ve yanıt URL'leri ajan bağlamından çıkarılır.

### İletişim durumu olayları

Slack, iletişim durumu değişikliklerini Events API veya Socket Mode üzerinden göndermez. Bunun yerine OpenClaw, mesajları normal Slack erişim ve yönlendirme denetimlerinden geçen insan katılımcılar için [`users.getPresence`](https://docs.slack.dev/reference/methods/users.getPresence/) yoklaması yapabilir.

```json5
{
  channels: {
    slack: {
      presenceEvents: { mode: "auto" },
      channels: {
        C0123456789: { presenceEvents: { mode: "on" } },
        C0987654321: { presenceEvents: { mode: "off" } },
      },
    },
  },
}
```

- `off` (varsayılan): iletişim durumu zamanlayıcısı veya Slack API çağrısı yoktur.
- `auto`: son 24 saat içinde etkin olan, en fazla 8 gözlemlenen insan katılımcıya sahip DM'leri, MPIM'leri ve Slack ileti dizilerini izler. Üst düzey kanal oturumları hariç tutulur.
- `on`: aynı konuşmaları katılımcı sınırı olmadan izler ve üst düzey kanal oturumlarını dahil eder. Bir kanalı zorla dahil etmek veya hariç tutmak için kanal başına geçersiz kılma kullanın.

OpenClaw, Slack hesabı başına dakikada en fazla 45 benzersiz kullanıcıyı yoklar, ilk sonucu ajanı uyandırmadan başlangıç değeri olarak kaydeder ve yalnızca gözlemlenen bir `away` durumundan `active` durumuna geçişte uyandırır. Kişi birkaç ileti dizisine katılsa bile Slack hesabı ve kullanıcı başına kalıcı 8 saatlik bir bekleme süresi uygulanır. Olay yalnızca söz konusu kişinin en son etkin olan uygun konuşmasına yönlendirilir ve ajana kısa bir selamlama gönderip göndermemeye karar vermeden önce belleğe/wiki'ye ve bilinen saat dilimi bağlamına başvurmasını söyler. Ajan sessiz kalabilir.

Bot belirteci, önerilen manifestte zaten bulunan `users:read` kapsamına ihtiyaç duyar. İletişim durumu olayları, Enterprise Grid kuruluş geneli kurulumlarda kullanılamaz.

## Yapılandırma referansı

Birincil referans: [Yapılandırma referansı - Slack](/tr/gateway/config-channels#slack).

<Accordion title="Önemli Slack alanları">

- mod/kimlik doğrulama: `identity`, `mode`, `enterpriseOrgInstall`, `botToken`, `appToken`, `userToken`, `signingSecret`, `webhookPath`, `accounts.*`
- DM erişimi: `dm.enabled`, `dmPolicy`, `allowFrom` (eski: `dm.policy`, `dm.allowFrom`), `dm.groupEnabled`, `dm.groupChannels`
- uyumluluk anahtarı: `dangerouslyAllowNameMatching` (acil durum; gerekmedikçe kapalı tutun)
- kanal erişimi: `groupPolicy`, `channels.*`, `channels.*.users`, `channels.*.requireMention`, `implicitMentions.*`
- iş parçacığı/geçmiş: `replyToMode`, `replyToModeByChatType`, `thread.*`, `historyLimit`, `dmHistoryLimit`, `dms.*.historyLimit`
- varlıkla uyandırmalar: `presenceEvents.mode`, `channels.*.presenceEvents.mode` (`off|auto|on`; varsayılan `off`)
- teslimat: `textChunkLimit`, `streaming.chunkMode`, `mediaMaxMb`, `streaming`, `streaming.nativeTransport`, `streaming.preview.toolProgress`
- önizlemeler: `unfurlLinks` (varsayılan: `false`), `chat.postMessage` bağlantı/medya önizleme denetimi için `unfurlMedia`; bağlantı önizlemelerini yeniden etkinleştirmek için `unfurlLinks: true` değerini ayarlayın
- işlemler/özellikler: `configWrites`, `commands.native`, `slashCommand.*`, `actions.*`, `userToken`, `userTokenReadOnly`

</Accordion>

## Sorun giderme

<AccordionGroup>
  <Accordion title="Kanallarda yanıt yok">
    Sırasıyla şunları kontrol edin:

    - `groupPolicy`
    - kanal izin listesi (`channels.slack.channels`) — **anahtarlar kanal kimlikleri olmalıdır** (`C12345678`), adlar (`#channel-name`) değil. Kanal yönlendirmesi varsayılan olarak önce kimliği kullandığından, ada dayalı anahtarlar `groupPolicy: "allowlist"` altında sessizce başarısız olur. Bir kimliği bulmak için: Slack'te kanala sağ tıklayın → **Copy link** — URL'nin sonundaki `C...` değeri kanal kimliğidir.
    - `requireMention`
    - kanal başına `users` izin listesi
    - `messages.groupChat.visibleReplies`: normal grup/kanal istekleri varsayılan olarak `"automatic"` değerini kullanır. `"message_tool"` seçeneğini etkinleştirdiyseniz ve günlüklerde `message(action=send)` çağrısı olmadan asistan metni görünüyorsa model, görünür mesaj aracı yolunu kullanmamıştır. Bu modda son metin özel kalır; engellenen yük meta verileri için Gateway ayrıntılı günlüğünü inceleyin veya her normal asistan son yanıtının eski yol üzerinden gönderilmesini istiyorsanız bunu `"automatic"` olarak ayarlayın.
    - `messages.groupChat.unmentionedInbound`: `"room_event"` ise, bahsedilmeden izin verilen kanal sohbetleri ortam bağlamıdır ve aracı `message` aracını çağırmadıkça sessiz kalır. Bkz. [Ortam odası olayları](/tr/channels/ambient-room-events).

```json5
{
  messages: {
    groupChat: {
      visibleReplies: "automatic",
    },
  },
}
```

    Yararlı komutlar:

```bash
openclaw channels status --probe
openclaw logs --follow
openclaw doctor
```

  </Accordion>

  <Accordion title="DM mesajları yok sayılıyor">
    Şunları kontrol edin:

    - `channels.slack.dm.enabled`
    - `channels.slack.dmPolicy` (veya eski `channels.slack.dm.policy`)
    - eşleştirme onayları / izin listesi girdileri (`dmPolicy: "open"` yine de `channels.slack.allowFrom: ["*"]` gerektirir)
    - grup DM'leri MPIM işlemeyi kullanır; `channels.slack.dm.groupEnabled` seçeneğini etkinleştirin ve yapılandırılmışsa MPIM'i `channels.slack.dm.groupChannels` içine ekleyin
    - Slack Assistant DM olayları: `drop message_changed` ifadesinden bahseden ayrıntılı günlükler
      genellikle Slack'in mesaj meta verilerinde kurtarılabilir bir insan gönderici
      bulunmayan, düzenlenmiş bir Assistant iş parçacığı olayı gönderdiği anlamına gelir

```bash
openclaw pairing list slack
```

  </Accordion>

  <Accordion title="Socket modu bağlanmıyor">
    Slack uygulama ayarlarında bot ve uygulama token'larını ve Socket Mode'un etkinleştirildiğini doğrulayın.
    App-Level Token için `connections:write` gerekir ve Bot User OAuth Token
    bot token'ı, uygulama token'ıyla aynı Slack uygulamasına/çalışma alanına ait olmalıdır.

    `openclaw channels status --probe --json`, `botTokenStatus` veya
    `appTokenStatus: "configured_unavailable"` gösteriyorsa Slack hesabı
    yapılandırılmıştır ancak mevcut çalışma zamanı SecretRef destekli
    değeri çözümleyememiştir.

    `slack socket mode failed to start; retry ...` gibi günlükler kurtarılabilir
    başlatma hatalarıdır. Eksik kapsamlar, iptal edilmiş token'lar ve geçersiz kimlik doğrulama ise
    hemen başarısız olur. `slack token mismatch ...` günlüğü, bot token'ı ile uygulama token'ının
    farklı Slack uygulamalarına ait göründüğü anlamına gelir; Slack uygulaması kimlik bilgilerini düzeltin.

  </Accordion>

  <Accordion title="HTTP modu olayları almıyor">
    Şunları doğrulayın:

    - imzalama sırrı
    - Webhook yolu
    - Slack Request URL'leri (Events + Interactivity + Slash Commands)
    - HTTP hesabı başına benzersiz `webhookPath`
    - herkese açık URL'nin TLS'yi sonlandırdığı ve istekleri Gateway yoluna ilettiği
    - Slack uygulamasının `request_url` yolunun `channels.slack.webhookPath` ile tam olarak eşleştiği (varsayılan `/slack/events`)

    Hesap anlık görüntülerinde `signingSecretStatus: "configured_unavailable"` görünüyorsa
    HTTP hesabı yapılandırılmıştır ancak mevcut çalışma zamanı
    SecretRef destekli imzalama sırrını çözümleyememiştir.

    Yinelenen `slack: webhook path ... already registered` günlüğü, iki HTTP
    hesabının aynı `webhookPath` değerini kullandığı anlamına gelir; her hesaba farklı bir yol verin.

  </Accordion>

  <Accordion title="Yerel/eğik çizgi komutları çalışmıyor">
    Şunlardan hangisini amaçladığınızı doğrulayın:

    - Slack'te kayıtlı eşleşen eğik çizgi komutlarıyla yerel komut modu (`channels.slack.commands.native: true`)
    - veya tek eğik çizgi komutu modu (`channels.slack.slashCommand.enabled: true`)

    Slack, eğik çizgi komutlarını otomatik olarak oluşturmaz veya kaldırmaz. `commands.native: "auto"`, Slack yerel komutlarını etkinleştirmez; `true` kullanın ve Slack uygulamasında eşleşen komutları oluşturun. HTTP modunda her Slack eğik çizgi komutu Gateway URL'sini içermelidir. Socket Mode'da komut yükleri websocket üzerinden gelir ve Slack `slash_commands[].url` değerini yok sayar.

    Ayrıca `commands.useAccessGroups`, DM yetkilendirmesi, kanal izin listeleri
    ve kanal başına `users` izin listelerini kontrol edin. Slack, engellenen
    eğik çizgi komutu göndericileri için geçici hatalar döndürür; bunlar arasında şunlar bulunur:

    - `This channel is not allowed.`
    - `You are not authorized to use this command here.`

  </Accordion>
</AccordionGroup>

## Ek medya referansı

Slack dosya indirmeleri başarılı olduğunda ve boyut sınırları izin verdiğinde Slack, indirilen medyayı aracı turuna ekleyebilir. Ses kliplerinin dökümü oluşturulabilir, görüntü dosyaları medya anlama yolundan geçebilir veya doğrudan görsel işleme yeteneğine sahip bir yanıt modeline iletilebilir; diğer dosyalar ise indirilebilir dosya bağlamı olarak kullanılabilir kalır.

### Desteklenen medya türleri

| Medya türü                     | Kaynak               | Mevcut davranış                                                                  | Notlar                                                                     |
| ------------------------------ | -------------------- | --------------------------------------------------------------------------------- | ------------------------------------------------------------------------- |
| Slack ses klipleri              | Slack dosya URL'si       | İndirilir ve paylaşılan ses dökümü oluşturma üzerinden yönlendirilir                          | `files:read` ve çalışan bir `tools.media.audio` modeli veya CLI gerektirir      |
| JPEG / PNG / GIF / WebP görüntüleri | Slack dosya URL'si       | İndirilir ve görsel işleme yeteneğine sahip kullanım için tura eklenir                   | Dosya başına sınır: `channels.slack.mediaMaxMb` (varsayılan 20 MB)                 |
| PDF dosyaları                      | Slack dosya URL'si       | İndirilir ve `download-file` veya `pdf` gibi araçlar için dosya bağlamı olarak sunulur | Slack'ten gelen PDF'ler otomatik olarak görüntü tabanlı görsel girdiye dönüştürülmez |
| Diğer dosyalar                    | Slack dosya URL'si       | Mümkün olduğunda indirilir ve dosya bağlamı olarak sunulur                              | İkili dosyalar görüntü girdisi olarak değerlendirilmez                               |
| İş parçacığı yanıtları                 | İş parçacığı başlangıç dosyaları | Yanıtta doğrudan medya yoksa kök mesaj dosyaları bağlam olarak yüklenebilir  | Yalnızca dosya içeren başlangıçlar bir ek yer tutucusu kullanır                          |
| Çok dosyalı mesajlar            | Birden fazla Slack dosyası | Her dosya bağımsız olarak değerlendirilir                                              | Slack işlemesi mesaj başına sekiz dosyayla sınırlıdır                     |

### Gelen veri işlem hattı

Dosya ekleri içeren bir Slack mesajı geldiğinde:

1. OpenClaw, bot token'ını kullanarak dosyayı Slack'in özel URL'sinden indirir.
2. Başarılı olduğunda dosya medya deposuna yazılır.
3. İndirilen medya yolları ve içerik türleri gelen bağlama eklenir.
4. Ses klipleri paylaşılan döküm oluşturma işlem hattına yönlendirilir; görüntü işleyebilen model/araç yolları aynı bağlamdaki görüntü eklerini kullanabilir.
5. Diğer dosyalar, bunları işleyebilen araçlar için dosya meta verileri veya medya referansları olarak kullanılabilir kalır.

### İş parçacığı kökü ek devralımı

Bir mesaj bir iş parçacığına geldiğinde (`thread_ts` üst öğesine sahipse):

- Yanıtın kendisinde doğrudan medya yoksa ve eklenen kök mesajda dosyalar varsa Slack, kök dosyaları iş parçacığı başlangıç bağlamı olarak yükleyebilir.
- Kök dosyaları yalnızca yeni veya sıfırlanmış bir iş parçacığı oturumu başlatılırken yüklenir. Sonraki yalnızca metin içeren yanıtlar mevcut oturum bağlamını yeniden kullanır ve kök dosyalarını yeni medya olarak yeniden eklemez.
- Doğrudan yanıt ekleri, kök mesaj eklerine göre önceliklidir.
- Yalnızca dosyaları olan ve metni olmayan bir kök mesaj, geri dönüşün yine de dosyalarını içerebilmesi için bir ek yer tutucusuyla temsil edilir.

### Birden fazla eki işleme

Tek bir Slack mesajı birden fazla dosya eki içerdiğinde:

- Her ek, medya işlem hattı üzerinden bağımsız olarak işlenir.
- İndirilen medya referansları mesaj bağlamında birleştirilir.
- İşleme sırası, olay yükündeki Slack dosya sırasını izler.
- Bir ekin indirilememesi diğerlerini engellemez.

### Boyut, indirme ve model sınırları

- **Boyut sınırı**: Dosya başına varsayılan 20 MB. `channels.slack.mediaMaxMb` üzerinden yapılandırılabilir.
- **Ses dökümü sınırı**: İndirilen dosya bir döküm sağlayıcısına veya CLI'ye gönderildiğinde, seçilen ses işleyebilen `tools.media.models[]` girdisinin `maxBytes` değeri de geçerlidir.
- **İndirme hataları**: Slack'in sunamadığı dosyalar, süresi dolmuş URL'ler, erişilemeyen dosyalar, boyut sınırını aşan dosyalar ve Slack kimlik doğrulama/oturum açma HTML yanıtları, desteklenmeyen biçimler olarak bildirilmek yerine atlanır.
- **Görsel modeli**: Görüntü analizi, görsel işlemeyi desteklediğinde etkin yanıt modelini veya `agents.defaults.imageModel` konumunda yapılandırılan görüntü modelini kullanır.

### Bilinen sınırlar

| Senaryo                                      | Mevcut davranış                                                                   | Geçici çözüm                                                                    |
| --------------------------------------------- | ---------------------------------------------------------------------------------- | ----------------------------------------------------------------------------- |
| Süresi dolmuş Slack dosya URL'si                        | Dosya atlanır; hata gösterilmez                                                       | Dosyayı Slack'e yeniden yükleyin                                                   |
| Ses transkripsiyonu kullanılamıyor               | Klip ekli kalır ancak transkript oluşturulmaz                                | `tools.media.audio` yapılandırın veya desteklenen bir yerel transkripsiyon CLI'si yükleyin  |
| Altyazısız klip, bahsetme geçidini aşmaz | Özel spekülatif transkripsiyondan sonra bırakılır; transkript ve indirme silinir | Söylenen ad için bir bahsetme kalıbı yapılandırın, yazılı bir bot bahsetmesi ekleyin veya DM kullanın |
| Görüntü modeli yapılandırılmamış                   | Görsel ekleri medya referansları olarak depolanır ancak görsel olarak analiz edilmez       | `agents.defaults.imageModel` yapılandırın veya görüntü özellikli bir yanıt modeli kullanın    |
| Çok büyük görseller (varsayılan olarak > 20 MB)        | Boyut sınırına göre atlanır                                                               | Slack izin veriyorsa `channels.slack.mediaMaxMb` değerini artırın                          |
| İletilen/paylaşılan ekler                  | Metin ile Slack'te barındırılan görsel/dosya medyası mümkün olduğunca işlenir                             | Doğrudan OpenClaw ileti dizisinde yeniden paylaşın                                      |
| PDF ekleri                               | Dosya/medya bağlamı olarak depolanır, görsel görüntü analizine otomatik olarak yönlendirilmez        | Dosya meta verileri için `download-file` veya PDF analizi için `pdf` aracını kullanın      |

### İlgili belgeler

- [Medya anlama işlem hattı](/tr/nodes/media-understanding)
- [Ses ve sesli notlar](/tr/nodes/audio)
- [PDF aracı](/tr/tools/pdf)

## İlgili

<CardGroup cols={2}>
  <Card title="Eşleştirme" icon="link" href="/tr/channels/pairing">
    Bir Slack kullanıcısını Gateway ile eşleştirin.
  </Card>
  <Card title="Gruplar" icon="users" href="/tr/channels/groups">
    Kanal ve grup DM davranışı.
  </Card>
  <Card title="Kanal yönlendirme" icon="route" href="/tr/channels/channel-routing">
    Gelen mesajları aracılara yönlendirin.
  </Card>
  <Card title="Güvenlik" icon="shield" href="/tr/gateway/security">
    Tehdit modeli ve sağlamlaştırma.
  </Card>
  <Card title="Yapılandırma" icon="sliders" href="/tr/gateway/configuration">
    Yapılandırma düzeni ve önceliği.
  </Card>
  <Card title="Eğik çizgi komutları" icon="terminal" href="/tr/tools/slash-commands">
    Komut kataloğu ve davranışı.
  </Card>
</CardGroup>
