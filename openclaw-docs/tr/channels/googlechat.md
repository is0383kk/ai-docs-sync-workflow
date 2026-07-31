---
read_when:
    - Google Chat kanal özellikleri üzerinde çalışma
summary: Google Chat uygulaması destek durumu, özellikleri ve yapılandırması
title: Google Chat
x-i18n:
    generated_at: "2026-07-26T23:49:42Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 9d3fb96564294b57040327bb21ab7331bf8412eb04f879a9c7ea1018ba2bddab
    source_path: channels/googlechat.md
    workflow: 16
---

Google Chat, resmi `@openclaw/googlechat` plugin'i olarak çalışır: Google Chat API Webhook'ları aracılığıyla DM'ler ve alanlar (yalnızca HTTP uç noktası, Pub/Sub yok).

## Kurulum

```bash
openclaw plugins install @openclaw/googlechat
```

Yerel çalışma kopyası (bir git deposundan çalıştırırken):

```bash
openclaw plugins install ./path/to/local/googlechat-plugin
```

## Hızlı kurulum (başlangıç)

1. Bir Google Cloud projesi oluşturun ve **Google Chat API**'yi etkinleştirin.
   - Şuraya gidin: [Google Chat API Kimlik Bilgileri](https://console.cloud.google.com/apis/api/chat.googleapis.com/credentials)
   - Henüz etkin değilse API'yi etkinleştirin.
2. Bir **Service Account** oluşturun:
   - **Create Credentials** > **Service Account** seçeneklerine basın.
   - İstediğiniz bir adı verin (ör. `openclaw-chat`).
   - İzinleri ve sorumluları boş bırakın (**Continue**, ardından **Done**).
3. **JSON anahtarını** oluşturup indirin:
   - Yeni hizmet hesabına tıklayın > **Keys** sekmesi > **Add Key** > **Create new key** > **JSON** > **Create**.
4. İndirilen JSON dosyasını Gateway ana makinenizde saklayın (ör. `~/.openclaw/googlechat-service-account.json`).
5. [Google Cloud Console Chat Configuration](https://console.cloud.google.com/apis/api/chat.googleapis.com/hangouts-chat) sayfasında bir Google Chat uygulaması oluşturun:
   - **Application info** alanını doldurun (uygulama adı, avatar URL'si, açıklama).
   - **Interactive features** seçeneğini etkinleştirin.
   - **Functionality** altında **Join spaces and group conversations** seçeneğini işaretleyin.
   - **Connection settings** altında **HTTP endpoint URL** seçeneğini belirleyin.
   - **Triggers** altında **Use a common HTTP endpoint URL for all triggers** seçeneğini belirleyin ve bunu herkese açık Gateway URL'nizin ardından `/googlechat` gelecek şekilde ayarlayın ([Herkese açık URL](#public-url-webhook-only) bölümüne bakın).
   - **Visibility** altında **Make this Chat app available to specific people and groups in `<Your Domain>`** seçeneğini işaretleyin ve e-posta adresinizi girin.
   - **Save** düğmesine tıklayın.
6. Uygulama durumunu etkinleştirin: sayfayı yenileyin, **App status** alanını bulun, **Live - available to users** olarak ayarlayın ve tekrar **Save** düğmesine tıklayın.
7. OpenClaw'ı hizmet hesabı ve Webhook hedef kitlesiyle yapılandırın (Chat uygulaması yapılandırmasıyla eşleşmelidir):
   - Ortam değişkeni: `GOOGLE_CHAT_SERVICE_ACCOUNT_FILE=/path/to/service-account.json` (yalnızca varsayılan hesap) veya
   - Yapılandırma: [Yapılandırmada öne çıkanlar](#config-highlights) bölümüne bakın. `openclaw channels add --channel googlechat`; `--audience-type`, `--audience`, `--webhook-path` ve `--webhook-url` değerlerini de kabul eder.
8. Gateway'i başlatın. Google Chat, Webhook yolunuza POST isteği gönderir (varsayılan: `/googlechat`).

## Google Chat'e ekleme

Gateway çalışmaya başladıktan ve e-posta adresiniz görünürlük listesine eklendikten sonra:

1. [Google Chat](https://chat.google.com/) adresine gidin.
2. **Direct Messages** yanındaki **+** (artı) simgesine tıklayın.
3. Google Cloud Console'da yapılandırdığınız **App name** değerini arayın.
   - Bot, özel bir uygulama olduğundan Marketplace göz atma listesinde görünmez; adıyla arayın.
4. Botu seçin, **Add** veya **Chat** düğmesine tıklayın ve bir mesaj gönderin.

## Herkese açık URL (yalnızca Webhook)

Google Chat Webhook'ları, herkese açık bir HTTPS uç noktası gerektirir. Güvenlik için internete **yalnızca `/googlechat` yolunu** açın; OpenClaw kontrol panelini ve diğer uç noktaları gizli tutun.

### Seçenek A: Tailscale Funnel (Önerilen)

Özel kontrol paneli için Tailscale Serve, herkese açık Webhook yolu için Funnel kullanın.

1. Gateway'in hangi adrese bağlandığını kontrol edin:

   ```bash
   ss -tlnp | grep 18789
   ```

   IP adresini not edin (ör. `127.0.0.1`, `0.0.0.0` veya bir Tailscale `100.x.x.x` adresi).

2. Kontrol panelini yalnızca tailnet'e açın (8443 numaralı bağlantı noktası):

   ```bash
   # Yerel ana makineye bağlıysa (127.0.0.1 veya 0.0.0.0):
   tailscale serve --bg --https 8443 http://127.0.0.1:18789

   # Yalnızca bir Tailscale IP'sine bağlıysa:
   tailscale serve --bg --https 8443 http://100.x.x.x:18789
   ```

3. Yalnızca Webhook yolunu herkese açın:

   ```bash
   # Yerel ana makineye bağlıysa (127.0.0.1 veya 0.0.0.0):
   tailscale funnel --bg --set-path /googlechat http://127.0.0.1:18789/googlechat

   # Yalnızca bir Tailscale IP'sine bağlıysa:
   tailscale funnel --bg --set-path /googlechat http://100.x.x.x:18789/googlechat
   ```

4. İstenirse bu Node için Funnel'ı etkinleştirmek üzere çıktıda gösterilen yetkilendirme URL'sini ziyaret edin.

5. Doğrulayın:

   ```bash
   tailscale serve status
   tailscale funnel status
   ```

Herkese açık Webhook URL'niz `https://<node-name>.<tailnet>.ts.net/googlechat`; kontrol paneli ise `https://<node-name>.<tailnet>.ts.net:8443/` adresinde yalnızca tailnet erişimine açık kalır. Google Chat uygulaması yapılandırmasında herkese açık URL'yi (`:8443` olmadan) kullanın.

> Not: Bu yapılandırma yeniden başlatmalarda korunur. Daha sonra `tailscale funnel reset` ve `tailscale serve reset` ile kaldırın.

### Seçenek B: Ters Proxy (Caddy)

Yalnızca Webhook yolunu proxy üzerinden iletin:

```caddy
your-domain.com {
    reverse_proxy /googlechat* localhost:18789
}
```

`your-domain.com/` istekleri yok sayılır veya 404 döndürür; `your-domain.com/googlechat` ise OpenClaw'a yönlendirilir.

### Seçenek C: Cloudflare Tunnel

Tünel giriş kurallarını yalnızca Webhook yolunu yönlendirecek şekilde yapılandırın:

- **Path**: `/googlechat` -> `http://localhost:18789/googlechat`
- **Default rule**: HTTP 404 (Not Found)

## Nasıl çalışır?

1. Google Chat, Gateway Webhook yoluna JSON POST istekleri gönderir (yalnızca POST, JSON içerik türü zorunludur ve IP başına hız sınırı uygulanır).
2. OpenClaw, yönlendirmeden önce her isteğin kimliğini doğrular:
   - Chat uygulaması olayları `Authorization: Bearer <token>` taşır; gövdenin tamamı ayrıştırılmadan önce token doğrulanır.
   - Google Workspace Eklentisi olayları, token'ı gövdede (`authorizationEventObject.systemIdToken`) taşır ve doğrulamadan önce daha katı bir kimlik doğrulama öncesi bütçe (16 KB, 3 sn.) kapsamında okunur.
3. Token, `audienceType` + `audience` ile karşılaştırılarak denetlenir:
   - `audienceType: "app-url"` → hedef kitle, HTTPS Webhook URL'nizdir.
   - `audienceType: "project-number"` → hedef kitle, Cloud proje numarasıdır.
   - `app-url` altındaki eklenti token'ları ayrıca `appPrincipal` değerinin uygulamanın sayısal OAuth 2.0 istemci kimliğine (21 basamak, e-posta değil) ayarlanmasını gerektirir; aksi takdirde doğrulama, günlüğe kaydedilen bir uyarıyla başarısız olur.
4. Mesajlar alana göre yönlendirilir:
   - Alanlar, alan başına `agent:<agentId>:googlechat:group:<spaceId>` oturumlarını kullanır; yanıtlar mesaj ileti dizisine gider.
   - DM'ler varsayılan olarak ajanın ana oturumunda birleştirilir; eş başına DM oturumları için `session.dmScope` değerini ayarlayın ([Oturum](/tr/concepts/session) bölümüne bakın).
5. DM erişimi varsayılan olarak eşleştirme kullanır. Bilinmeyen gönderenler bir eşleştirme kodu alır; şu komutla onaylayın:
   - `openclaw pairing approve googlechat <code>`
6. Grup alanları varsayılan olarak @bahsetme gerektirir. Bahsetmeler, uygulamayı hedefleyen Chat `USER_MENTION` ek açıklamalarından algılanır; algılama için uygulamanın kullanıcı kaynağı adı gerekiyorsa `botUser` değerini (ör. `users/1234567890`) ayarlayın.
7. Bir exec veya plugin onayı Google Chat'ten başlatıldığında ve kararlı bir `users/<id>` onaylayıcısı yapılandırıldığında OpenClaw, kaynak alana veya ileti dizisine yerel bir onay kartı (`cardsV2`) gönderir. Kart düğmeleri opak geri çağırma token'ları taşır; manuel `/approve <id> <decision>` istemi yalnızca yerel teslimat kullanılamadığında görünür.

### Gelen iletilerin dayanıklılığı

İstek kimliği doğrulandıktan sonra OpenClaw, eklenti yetkilendirme nesnesini depolama alanından kaldırır ve `200` döndürmeden önce Google Chat `MESSAGE` olaylarını kalıcı bir kuyruğa alır. Kalıcılık hatası `503` döndürerek Google Chat'in, kaybolabilecek bir olayı kabul edilmiş saymak yerine yeniden denemesine olanak tanır.

Bekleyen veya yeniden denenebilir mesajlar Gateway yeniden başlatıldığında korunur, alan başına sıralı kalır ve etkin ya da saklanan tamamlanma kaydı mevcut olduğu sürece yinelenen kuyruk girdilerini engellemek için Google Chat mesaj kaynağı adını kullanır. Mesaj dışı eylemler mevcut ayrık Webhook yolunu kullanmaya devam eder ve bu dayanıklı kuyruk garantisine sahip değildir. Kuyruktan ajana geçiş sınırında teslimat en az bir kez yapılmaya devam eder; bu nedenle aktarım sırasında oluşan bir çökme, bir turun yeniden oynatılmasına yol açabilir.

## Hedefler

Teslimat ve izin listeleri için şu tanımlayıcıları kullanın:

- Doğrudan mesajlar: `users/<userId>` (önerilen).
- Alanlar: `spaces/<spaceId>`.
- Ham e-posta `name@example.com` değiştirilebilir ve yalnızca `channels.googlechat.dangerouslyAllowNameMatching: true` olduğunda izin listesi eşleştirmesi için kullanılır.
- Kullanımdan kaldırıldı: `users/<email>`, e-posta izin listesi girdisi olarak değil, kullanıcı kimliği olarak değerlendirilir.
- `googlechat:`, `google-chat:` ve `gchat:` ön ekleri kabul edilir ve kaldırılır.

## Yapılandırmada öne çıkanlar

```json5
{
  channels: {
    googlechat: {
      enabled: true,
      serviceAccountFile: "/path/to/service-account.json",
      // veya serviceAccountRef: { source: "file", provider: "filemain", id: "/channels/googlechat/serviceAccount" }
      audienceType: "app-url",
      audience: "https://gateway.example.com/googlechat",
      appPrincipal: "123456789012345678901", // yalnızca eklenti doğrulaması; sayısal OAuth istemci kimliği
      webhookPath: "/googlechat",
      botUser: "users/1234567890", // isteğe bağlı; bahsetme algılamasına yardımcı olur
      allowBots: false,
      dmPolicy: "pairing",
      allowFrom: ["users/1234567890"],
      groupPolicy: "allowlist",
      groups: {
        "spaces/AAAA": {
          enabled: true,
          requireMention: true,
          users: ["users/1234567890"],
          systemPrompt: "Yalnızca kısa yanıtlar.",
        },
      },
      typingIndicator: "message",
      mediaMaxMb: 20,
    },
  },
}
```

Notlar:

- Hizmet hesabı kimlik bilgileri: `serviceAccountFile` (yol), `serviceAccount` (satır içi JSON dizesi veya nesnesi) ya da `serviceAccountRef` (ortam değişkeni/dosya SecretRef'i). `GOOGLE_CHAT_SERVICE_ACCOUNT` (satır içi JSON) ve `GOOGLE_CHAT_SERVICE_ACCOUNT_FILE` (yol) ortam değişkenleri yalnızca varsayılan hesaba uygulanır. Çok hesaplı kurulumlar, hesap başına `serviceAccountRef` dahil olmak üzere aynı anahtarlarla `channels.googlechat.accounts.<id>` kullanır.
- `webhookPath` ayarlanmadığında varsayılan Webhook yolu `/googlechat` olur; bunun yerine `webhookUrl` yolu sağlayabilir.
- Grup anahtarları kararlı alan kimlikleri (`spaces/<spaceId>`) olmalıdır. Görünen ad anahtarları kullanımdan kaldırılmıştır ve bu şekilde günlüğe kaydedilir.
- `dangerouslyAllowNameMatching`, izin listeleri için değiştirilebilir e-posta sorumlusu eşleştirmesini yeniden etkinleştirir (acil durum uyumluluk modu); doctor, e-posta girdileri hakkında uyarır.
- Google Chat tepki eylemleri kullanıma sunulmaz. Plugin, hizmet hesabı kimlik doğrulaması kullanırken Google Chat tepki uç noktaları kullanıcı kimlik doğrulaması gerektirir. Mevcut `actions.reactions` yapılandırması uyumluluk için kabul edilir ancak hiçbir etkisi yoktur.
- Yerel onay kartları, tepki olaylarını değil Google Chat `cardsV2` düğme tıklamalarını kullanır. Onaylayıcılar `allowFrom` veya `defaultTo` üzerinden gelir ve kararlı, sayısal `users/<id>` değerleri olmalıdır.
- Mesaj eylemleri yalnızca `send` metnini kullanıma sunar. Google Chat ek yükleme işlemi kullanıcı kimlik doğrulaması gerektirirken bu plugin hizmet hesabı kimlik doğrulaması kullandığından giden dosya yükleme kullanıma sunulmaz.
- `typingIndicator`: `message` (varsayılan), bir `_<Bot> is typing..._` yer tutucusu gönderip bunu ilk yanıtla değiştirir; `none` bunu devre dışı bırakır; `reaction` kullanıcı OAuth'ı gerektirir ve şu anda hizmet hesabı kimlik doğrulaması altında günlüğe kaydedilen bir hatayla `message` değerine geri döner.
- Gelen ekler (mesaj başına ilk ek), Chat API aracılığıyla medya işlem hattına indirilir ve `mediaMaxMb` (varsayılan 20) ile sınırlandırılır.
- Bot tarafından yazılan mesajlar varsayılan olarak yok sayılır. `allowBots: true` ile kabul edilen bot mesajları, paylaşılan [bot döngüsü korumasını](/tr/channels/bot-loop-protection) kullanır: `channels.defaults.botLoopProtection` değerini yapılandırın, ardından `channels.googlechat.botLoopProtection` veya `channels.googlechat.groups.<space>.botLoopProtection` ile geçersiz kılın.

Gizli bilgiler referans ayrıntıları: [Gizli Bilgiler Yönetimi](/tr/gateway/secrets).

## Sorun giderme

### 405 Yönteme İzin Verilmiyor

Google Cloud Logs Explorer aşağıdaki gibi hatalar gösteriyorsa:

```text
durum kodu: 405, neden ifadesi: HTTP hata yanıtı: HTTP/1.1 405 Yönteme İzin Verilmiyor
```

Webhook işleyicisi kaydedilmemiştir. Yaygın nedenler:

1. **Kanal yapılandırılmamış**: `channels.googlechat` bölümü eksik. Şununla doğrulayın:

   ```bash
   openclaw config get channels.googlechat
   ```

   "Config path not found" döndürürse yapılandırmayı ekleyin ([Yapılandırmada öne çıkanlar](#config-highlights) bölümüne bakın).

2. **Plugin etkinleştirilmemiş**: Plugin durumunu kontrol edin:

   ```bash
   openclaw plugins list | grep googlechat
   ```

   "disabled" gösteriyorsa yapılandırmanıza `plugins.entries.googlechat.enabled: true` ekleyin.

3. Yapılandırma değişikliklerinden sonra **Gateway yeniden başlatılmamış**:

   ```bash
   openclaw gateway restart
   ```

Kanalın çalıştığını doğrulayın:

```bash
openclaw channels status
# Şunu göstermelidir: Google Chat default: enabled, configured, ...
```

### Diğer sorunlar

- `openclaw channels status --probe`, kimlik doğrulama hatalarını ve eksik hedef kitle yapılandırmasını gösterir (`audience` ve `audienceType` değerlerinin ikisi de gereklidir).
- Hiç mesaj gelmiyorsa Chat uygulamasının webhook URL'sini ve tetikleyici yapılandırmasını doğrulayın.
- Bahsetme geçidi yanıtları engelliyorsa `botUser` değerini uygulamanın kullanıcı kaynağı adı olarak ayarlayın ve `requireMention` değerini kontrol edin.
- Test mesajı gönderirken `openclaw logs --follow`, isteklerin Gateway'e ulaşıp ulaşmadığını gösterir.

## İlgili

- [Kanallara Genel Bakış](/tr/channels) — desteklenen tüm kanallar
- [Kanal Yönlendirme](/tr/channels/channel-routing) — mesajlar için oturum yönlendirme
- [Gateway yapılandırması](/tr/gateway/configuration)
- [Gruplar](/tr/channels/groups) — grup sohbeti davranışı ve bahsetme geçidi
- [Eşleştirme](/tr/channels/pairing) — doğrudan mesaj kimlik doğrulaması ve eşleştirme akışı
- [Güvenlik](/tr/gateway/security) — erişim modeli ve güçlendirme
