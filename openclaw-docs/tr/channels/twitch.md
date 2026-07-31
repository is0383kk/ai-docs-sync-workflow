---
read_when:
    - OpenClaw için Twitch sohbet entegrasyonunu ayarlama
sidebarTitle: Twitch
summary: 'Twitch sohbet botu: kurulum, kimlik bilgileri, erişim denetimi, token yenileme'
title: Twitch
x-i18n:
    generated_at: "2026-07-26T22:35:46Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: d827c742ded5fd0b071443dead27b975e2414419b0facb486d7f9c0c9800b060
    source_path: channels/twitch.md
    workflow: 16
---

Twurple istemcisi aracılığıyla Twitch'in sohbet (IRC) arayüzü üzerinden Twitch sohbet desteği. OpenClaw, bir Twitch bot hesabıyla oturum açar, yapılandırılmış her hesap için bir kanala katılır ve o kanalda yanıt verir.

## Kurulum

Twitch, resmî bir Plugin olarak sunulur; çekirdek kurulumun parçası değildir.

<Tabs>
  <Tab title="npm kayıt defteri">
    ```bash
    openclaw plugins install @openclaw/twitch
    ```
  </Tab>
  <Tab title="Yerel çalışma kopyası">
    ```bash
    openclaw plugins install ./path/to/local/twitch-plugin
    ```
  </Tab>
</Tabs>

`plugins install`, Plugin'i kaydeder ve etkinleştirir. `openclaw onboard` veya `openclaw channels add` sırasında Twitch seçildiğinde gerektiğinde kurulur. Güncel sürümü takip etmek için yalnızca paket adını kullanın; tam sürümü yalnızca tekrarlanabilir kurulumlar için sabitleyin. OpenClaw 2026.4.10 veya daha yenisini gerektirir.

Ayrıntılar: [Plugin'ler](/tr/tools/plugin)

## Hızlı kurulum

<Steps>
  <Step title="Plugin'i kurun">
    Yukarıdaki [Kurulum](#install) bölümüne bakın.
  </Step>
  <Step title="Bir Twitch bot hesabı oluşturun">
    Bot için ayrı bir Twitch hesabı oluşturun (veya mevcut bir hesabı kullanın).
  </Step>
  <Step title="Kimlik bilgilerini oluşturun">
    [Twitch Token Generator](https://twitchtokengenerator.com/) aracını kullanın:

    - **Bot Token** öğesini seçin
    - `chat:read` ve `chat:write` kapsamlarının seçili olduğunu doğrulayın
    - **Client ID** ve **Access Token** değerlerini kopyalayın

  </Step>
  <Step title="Twitch kullanıcı kimliğinizi bulun">
    Bir kullanıcı adını Twitch kullanıcı kimliğine dönüştürmek için [https://www.streamweasels.com/tools/convert-twitch-username-to-user-id/](https://www.streamweasels.com/tools/convert-twitch-username-to-user-id/) aracını kullanın.
  </Step>
  <Step title="Token'ı yapılandırın">
    - Ortam değişkeni: `OPENCLAW_TWITCH_ACCESS_TOKEN=...` (yalnızca varsayılan hesap)
    - Veya yapılandırma: `channels.twitch.accessToken`

    Her ikisi de ayarlanmışsa yapılandırma önceliklidir (ortam değişkeni yalnızca varsayılan hesap için yedek olarak kullanılır).

  </Step>
  <Step title="Gateway'i başlatın">
    ```bash
    openclaw gateway run
    ```
  </Step>
</Steps>

<Warning>
Yetkisiz kullanıcıların botu tetiklemesini önlemek için erişim denetimi (`allowFrom` veya `allowedRoles`) ekleyin. `requireMention` varsayılan olarak `true` değerindedir.
</Warning>

Asgari yapılandırma:

```json5
{
  channels: {
    twitch: {
      enabled: true,
      username: "openclaw", // Botun Twitch hesabı (kimlik doğrulaması yapar)
      accessToken: "oauth:abc123...", // OAuth erişim token'ı (veya OPENCLAW_TWITCH_ACCESS_TOKEN ortam değişkenini kullanın)
      clientId: "xyz789...", // Token Generator'dan alınan istemci kimliği
      channel: "yourchannel", // Hangi Twitch kanalının sohbetine katılacağı (zorunlu)
      allowFrom: ["123456789"], // (önerilir) Yalnızca Twitch kullanıcı kimliğiniz
    },
  },
}
```

## Nedir?

- Gateway'in sahip olduğu bir Twitch kanalıdır.
- Belirlenimci yönlendirme: yanıtlar her zaman mesajın geldiği Twitch kanalına geri gönderilir.
- Katılınan her kanal, yalıtılmış bir grup oturumu anahtarıyla `agent:<agentId>:twitch:group:<channel>` eşlenir.
- `username` botun hesabıdır (kimlik doğrulaması yapan), `channel` ise katılınacak sohbet odasıdır. Her hesap girdisi tam olarak bir kanala katılır.
- Token'lar `oauth:` önekiyle veya öneksiz çalışır; OpenClaw her iki biçimi de normalleştirir (kurulum sihirbazı `oauth:` biçimini bekler).

## Gelen iletilerin dayanıklılığı

OpenClaw, kabul edilen her Twitch sohbet iletisini normal dağıtımdan önce kalıcı olarak kuyruğa alır. Bekleyen veya yeniden denenebilir iletiler Gateway yeniden başlatıldığında korunur, yapılandırılmış kanal için sıralı kalır ve etkin ya da saklanan tamamlanma kaydı mevcut olduğu sürece yinelenen kuyruk girdilerini engellemek için Twitch'in ileti kimliğini kullanır.

Twitch sohbeti, istemci kabul ettikten sonra bir `PRIVMSG` öğesini yeniden oynatmaz. Bu, yerel kabul ile dağıtım arasındaki çökme aralığını korur ancak kalıcı kabulden önce kaçırılan iletileri kurtaramaz. Kuyruğa ekleme işleminin kendisi başarısız olursa OpenClaw hatayı günlüğe kaydeder; yeniden bağlanmak Twitch'ten bu iletiyi yeniden göndermesini istemez.

## Token yenileme (isteğe bağlı)

[Twitch Token Generator](https://twitchtokengenerator.com/) tarafından oluşturulan token'lar OpenClaw tarafından yenilenemez; süreleri dolduğunda yeniden oluşturun (birkaç saat geçerlidirler; uygulama kaydı gerekmez).

Otomatik yenileme için [Twitch Developer Console](https://dev.twitch.tv/console) üzerinden kendi uygulamanızı oluşturun ve şunları ekleyin:

```json5
{
  channels: {
    twitch: {
      clientSecret: "your_client_secret",
      refreshToken: "your_refresh_token",
    },
  },
}
```

Her ikisi de ayarlandığında Plugin, token'ları süreleri dolmadan yenileyen ve her yenilemeyi günlüğe kaydeden bir yenilemeli kimlik doğrulama sağlayıcısı kullanır. `refreshToken` olmadan `token refresh disabled (no refresh token)` günlüğe kaydedilir; `clientSecret` olmadan statik (yenilenmeyen) bir token'a geri dönülür.

## Çoklu hesap desteği

Hesap başına kimlik bilgileriyle `channels.twitch.accounts` kullanın. Ortak kalıp için [Yapılandırma](/tr/gateway/configuration) bölümüne bakın.

Örnek (iki kanalda tek bot hesabı):

```json5
{
  channels: {
    twitch: {
      accounts: {
        channel1: {
          username: "openclaw",
          accessToken: "oauth:abc123...",
          clientId: "xyz789...",
          channel: "yourchannel",
        },
        channel2: {
          username: "openclaw",
          accessToken: "oauth:def456...",
          clientId: "uvw012...",
          channel: "secondchannel",
        },
      },
    },
  },
}
```

<Note>
Her hesap girdisi kendi `accessToken` değerine ihtiyaç duyar (ortam değişkeni yalnızca varsayılan hesabı kapsar). Bir hesap tam olarak bir kanala katılır; dolayısıyla iki kanala katılmak iki hesap gerektirir. `channels.twitch.defaultAccount`, hangi hesabın varsayılan olduğunu seçer.
</Note>

## Erişim denetimi

`allowFrom`, Twitch kullanıcı kimliklerinden oluşan katı bir izin listesidir. Ayarlandığında `allowedRoles` yok sayılır; bunun yerine rol tabanlı erişim kullanmak için `allowFrom` ayarını kaldırılmış bırakın.

**Kullanılabilir roller:** `"moderator"`, `"owner"`, `"vip"`, `"subscriber"`, `"all"`.

<Tabs>
  <Tab title="Kullanıcı kimliği izin listesi (en güvenli)">
    ```json5
    {
      channels: {
        twitch: {
          accounts: {
            default: {
              allowFrom: ["123456789", "987654321"],
            },
          },
        },
      },
    }
    ```
  </Tab>
  <Tab title="Rol tabanlı">
    ```json5
    {
      channels: {
        twitch: {
          accounts: {
            default: {
              allowedRoles: ["moderator", "vip"],
            },
          },
        },
      },
    }
    ```
  </Tab>
  <Tab title="@bahsetme gereksinimini devre dışı bırak">
    Varsayılan olarak `requireMention`, `true` değerindedir. İzin verilen tüm iletilere yanıt vermek için:

    ```json5
    {
      channels: {
        twitch: {
          accounts: {
            default: {
              requireMention: false,
            },
          },
        },
      },
    }
    ```

  </Tab>
</Tabs>

<Note>
**Neden kullanıcı kimlikleri?** Kullanıcı adları değişebilir ve kimliğe bürünmeye olanak tanıyabilir. Kullanıcı kimlikleri kalıcıdır.

Kendi kimliğinizi [kullanıcı adından kimliğe dönüştürücü](https://www.streamweasels.com/tools/convert-twitch-username-to-user-id/) ile bulun.
</Note>

## Sorun giderme

Önce tanılama komutlarını çalıştırın:

```bash
openclaw doctor
openclaw channels status --probe
```

<AccordionGroup>
  <Accordion title="Bot iletilere yanıt vermiyor">
    - **Erişim denetimini kontrol edin:** Kullanıcı kimliğinizin `allowFrom` içinde olduğundan emin olun veya test etmek için geçici olarak `allowFrom` öğesini kaldırıp `allowedRoles: ["all"]` değerini ayarlayın.
    - **Bahsetme geçidini kontrol edin:** `requireMention: true` (varsayılan) kullanıldığında iletiler botun kullanıcı adından @bahsetmelidir.
    - **Botun kanalda olduğunu kontrol edin:** Bot yalnızca `channel` içinde belirtilen kanala katılır.

  </Accordion>
  <Accordion title="Token sorunları">
    "Bağlanılamadı" veya kimlik doğrulama hataları:

    - `accessToken` değerinin OAuth erişim token'ı değeri olduğunu doğrulayın (`oauth:` öneki isteğe bağlıdır)
    - Token'ın `chat:read` ve `chat:write` kapsamlarına sahip olduğunu kontrol edin
    - Token yenileme kullanılıyorsa `clientSecret` ve `refreshToken` değerlerinin ayarlandığını doğrulayın

  </Accordion>
  <Accordion title="Token yenileme çalışmıyor">
    Yenileme olayları için günlükleri kontrol edin:

    ```text
    mybot için ortam değişkeni token kaynağı kullanılıyor
    123456 kullanıcısının erişim token'ı yenilendi (14400s içinde süresi dolacak)
    ```

    `token refresh disabled (no refresh token)` görürseniz:

    - `clientSecret` sağlandığından emin olun
    - `refreshToken` sağlandığından emin olun

  </Accordion>
</AccordionGroup>

## Yapılandırma

### Hesap yapılandırması

<ParamField path="username" type="string" required>
  Bot kullanıcı adı (kimlik doğrulaması yapan hesap).
</ParamField>
<ParamField path="accessToken" type="string" required>
  `chat:read` ve `chat:write` kapsamlarına sahip OAuth erişim token'ı (varsayılan hesap için yapılandırma veya ortam değişkeni).
</ParamField>
<ParamField path="clientId" type="string" required>
  Twitch İstemci Kimliği (Token Generator'dan veya uygulamanızdan). Şemada isteğe bağlıdır ancak bağlanmak için zorunludur.
</ParamField>
<ParamField path="channel" type="string" required>
  Katılınacak kanal.
</ParamField>
<ParamField path="enabled" type="boolean" default="true">
  Bu hesabı etkinleştirin.
</ParamField>
<ParamField path="clientSecret" type="string">
  İsteğe bağlı: otomatik token yenileme için.
</ParamField>
<ParamField path="refreshToken" type="string">
  İsteğe bağlı: otomatik token yenileme için.
</ParamField>
<ParamField path="expiresIn" type="number">
  Token'ın saniye cinsinden sona erme süresi (yenileme takibi).
</ParamField>
<ParamField path="obtainmentTimestamp" type="number">
  Token'ın alındığı zaman damgası (yenileme takibi).
</ParamField>
<ParamField path="allowFrom" type="string[]">
  Kullanıcı kimliği izin listesi. Ayarlandığında roller yok sayılır.
</ParamField>
<ParamField path="allowedRoles" type='Array<"moderator" | "owner" | "vip" | "subscriber" | "all">'>
  Rol tabanlı erişim denetimi.
</ParamField>
<ParamField path="requireMention" type="boolean" default="true">
  Botu tetiklemek için @bahsetme gerektirir.
</ParamField>
<ParamField path="responsePrefix" type="string">
  Bu hesap için giden yanıt öneki geçersiz kılma değeri.
</ParamField>

### Sağlayıcı seçenekleri

- `channels.twitch.enabled` - Kanal başlangıcını etkinleştirir/devre dışı bırakır
- `channels.twitch.username` / `accessToken` / `clientId` / `channel` - Basitleştirilmiş tek hesap yapılandırması (örtük `default` hesabı; `accounts.default` üzerinde önceliklidir)
- `channels.twitch.accounts.<accountName>` - Çoklu hesap yapılandırması (yukarıdaki tüm hesap alanları)
- `channels.twitch.defaultAccount` - Hangi hesap adının varsayılan olduğu
- `channels.twitch.markdown.tables` - Markdown tablosu oluşturma modu (`off` | `bullets` | `code` | `block`)

Tam örnek:

```json5
{
  channels: {
    twitch: {
      enabled: true,
      username: "openclaw",
      accessToken: "oauth:abc123...",
      clientId: "xyz789...",
      channel: "yourchannel",
      clientSecret: "secret123...",
      refreshToken: "refresh456...",
      allowFrom: ["123456789"],
      accounts: {
        second: {
          username: "mybot",
          accessToken: "oauth:def456...",
          clientId: "uvw012...",
          channel: "your_channel",
          enabled: true,
          expiresIn: 14400,
          obtainmentTimestamp: 1706092800000,
          allowedRoles: ["moderator"],
        },
      },
    },
  },
}
```

## Araç eylemleri

Aracı, ileti aracının `send` eylemi üzerinden Twitch iletileri gönderebilir:

```json5
{
  channel: "twitch",
  action: "send",
  to: "#mychannel",
  message: "Hello Twitch!",
}
```

`to` isteğe bağlıdır ve varsayılan olarak hesabın yapılandırılmış `channel` değerini kullanır.

## Güvenlik ve operasyonlar

- **Token'ları parolalar gibi ele alın** - token'ları asla git'e işlemeyin.
- Uzun süre çalışan botlar için **otomatik token yenilemeyi kullanın**.
- Erişim denetimi için kullanıcı adları yerine **kullanıcı kimliği izin listelerini kullanın**.
- Token yenileme olayları ve bağlantı durumu için **günlükleri izleyin**.
- **Token kapsamlarını en düşük düzeyde tutun** - yalnızca `chat:read` ve `chat:write` kapsamlarını isteyin.
- **Takılırsanız**: başka hiçbir işlemin oturumun sahibi olmadığını doğruladıktan sonra Gateway'i yeniden başlatın.

## Sınırlar

- Mesaj başına **500 karakter**; daha uzun yanıtlar sözcük sınırlarından parçalara ayrılır.
- Markdown gönderilmeden önce kaldırılır (Twitch sohbeti düz metindir; yeni satırlar boşluklara dönüşür).
- OpenClaw kendi başına herhangi bir hız sınırlaması eklemez; Twitch hız sınırlarını Twurple sohbet istemcisi yönetir.

## İlgili

- [Kanal Yönlendirme](/tr/channels/channel-routing) — iletiler için oturum yönlendirmesi
- [Kanallara Genel Bakış](/tr/channels) — desteklenen tüm kanallar
- [Gruplar](/tr/channels/groups) — grup sohbeti davranışı ve bahsetme denetimi
- [Eşleştirme](/tr/channels/pairing) — DM kimlik doğrulaması ve eşleştirme akışı
- [Güvenlik](/tr/gateway/security) — erişim modeli ve sağlamlaştırma
