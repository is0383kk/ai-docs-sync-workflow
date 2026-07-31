---
read_when:
    - Microsoft Teams kanal özellikleri üzerinde çalışma
summary: Microsoft Teams bot desteği durumu, özellikleri ve yapılandırması
title: Microsoft Teams
x-i18n:
    generated_at: "2026-07-26T23:50:35Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 5a4cf686da27e28b58f7afaad8cc837dbddb93219cde0c37285f9f6895f6fb8c
    source_path: channels/msteams.md
    workflow: 16
---

Durum: metin + DM ekleri desteklenir; kanal/grup dosyası göndermek için `sharePointSiteId` + Graph izinleri gerekir (bkz. [Grup sohbetlerinde dosya gönderme](#sending-files-in-group-chats)). Anketler Adaptive Cards aracılığıyla gönderilir. Mesaj eylemleri, önce dosya gönderimleri için açıkça `upload-file` sunar.

## Paketle birlikte gelen plugin

Microsoft Teams, güncel OpenClaw sürümlerinde paketle birlikte gelen bir plugin olarak sunulur; normal paketlenmiş derlemede ayrı kurulum gerekmez.

Paketle birlikte gelen Teams'i içermeyen eski bir derlemede veya özel kurulumda npm paketini doğrudan kurun:

```bash
openclaw plugins install @openclaw/msteams
```

Güncel resmî sürüm etiketini takip etmek için sürüm belirtilmemiş paketi kullanın. Yalnızca tekrarlanabilir bir kurulum gerektiğinde tam bir sürümü sabitleyin.

Yerel çalışma kopyası (bir git deposundan çalıştırırken):

```bash
openclaw plugins install ./path/to/local/msteams-plugin
```

Ayrıntılar: [Plugin'ler](/tr/tools/plugin)

## Hızlı kurulum

[`@microsoft/teams.cli`](https://www.npmjs.com/package/@microsoft/teams.cli), bot kaydını, manifest oluşturmayı ve kimlik bilgisi üretimini tek komutla gerçekleştirir.

**1. Kurun ve oturum açın**

```bash
npm install -g @microsoft/teams.cli@preview
teams login
teams status   # oturum açtığınızı doğrulayın ve kiracı bilgilerinizi görüntüleyin
```

<Note>
Teams CLI şu anda önizleme aşamasındadır. Komutlar ve bayraklar sürümler arasında değişebilir.
</Note>

**2. Bir tünel başlatın** (Teams localhost'a erişemez)

Gerekirse devtunnel CLI'yi kurup kimlik doğrulaması yapın ([başlangıç kılavuzu](https://learn.microsoft.com/en-us/azure/developer/dev-tunnels/get-started)).

```bash
# Tek seferlik kurulum (oturumlar arasında kalıcı URL):
devtunnel create my-openclaw-bot --allow-anonymous
devtunnel port create my-openclaw-bot -p 3978 --protocol auto

# Her geliştirme oturumunda:
devtunnel host my-openclaw-bot
# Uç noktanız: https://<tunnel-id>.devtunnels.ms/api/messages
```

<Note>
Teams, devtunnels ile kimlik doğrulaması yapamadığından `--allow-anonymous` gereklidir. Gelen her bot isteği yine Teams SDK tarafından doğrulanır.
</Note>

Alternatifler: `ngrok http 3978` veya `tailscale funnel 3978` (URL'ler her oturumda değişebilir).

**3. Uygulamayı oluşturun**

```bash
teams app create \
  --name "OpenClaw" \
  --endpoint "https://<your-tunnel-url>/api/messages"
```

Bu komut bir Entra ID (Azure AD) uygulaması oluşturur, bir istemci gizli anahtarı üretir, bir Teams uygulama manifesti (simgelerle birlikte) oluşturup yükler ve Teams tarafından yönetilen bir bot kaydeder (Azure aboneliği gerekmez). Çıktı; `CLIENT_ID`, `CLIENT_SECRET`, `TENANT_ID` ve bir **Teams Uygulama Kimliği** içerir; ayrıca uygulamayı doğrudan Teams'e yüklemeyi teklif eder.

**4. OpenClaw'ı yapılandırın**; bunun için çıktıdaki kimlik bilgilerini kullanın:

```json5
{
  channels: {
    msteams: {
      enabled: true,
      appId: "<CLIENT_ID>",
      appPassword: "<CLIENT_SECRET>",
      tenantId: "<TENANT_ID>",
      webhook: { port: 3978, path: "/api/messages" },
    },
  },
}
```

Alternatif olarak ortam değişkenlerini doğrudan kullanın: `MSTEAMS_APP_ID`, `MSTEAMS_APP_PASSWORD`, `MSTEAMS_TENANT_ID`.

**5. Uygulamayı Teams'e yükleyin**

`teams app create`, uygulamayı yüklemenizi ister; "Install in Teams" seçeneğini belirleyin. Yükleme bağlantısını daha sonra almak için:

```bash
teams app get <teamsAppId> --install-link
```

**6. Her şeyin çalıştığını doğrulayın**

```bash
teams app doctor <teamsAppId>
```

Bot kaydı, AAD uygulama yapılandırması, manifest geçerliliği ve SSO kurulumu genelinde tanılamalar çalıştırır.

Üretim ortamında istemci gizli anahtarları yerine [federe kimlik doğrulamasını](#federated-authentication-certificate-plus-managed-identity) (sertifika veya yönetilen kimlik) değerlendirin.

<Note>
Grup sohbetleri varsayılan olarak engellenir (`channels.msteams.groupPolicy: "allowlist"`). Grup yanıtlarına izin vermek için `channels.msteams.groupAllowFrom` ayarını belirleyin veya herhangi bir üyeye izin vermek için `groupPolicy: "open"` kullanın (bahsetme koşuluyla).
</Note>

## Hedefler

- Teams DM'leri, grup sohbetleri veya kanalları aracılığıyla OpenClaw ile konuşun.
- Yönlendirmeyi deterministik tutun: yanıtlar her zaman geldikleri kanala geri gider.
- Varsayılan olarak güvenli kanal davranışını kullanın (aksi yapılandırılmadıkça bahsetme gerekir).

## Yapılandırma yazımları

Microsoft Teams, varsayılan olarak `/config set|unset` tarafından tetiklenen yapılandırma güncellemelerini yazabilir (`commands.config: true` gerekir).

Şununla devre dışı bırakın:

```json5
{
  channels: { msteams: { configWrites: false } },
}
```

## Erişim denetimi (DM'ler + gruplar)

**DM erişimi**

- Varsayılan: `channels.msteams.dmPolicy = "pairing"`. Bilinmeyen göndericiler onaylanana kadar yok sayılır.
- `channels.msteams.allowFrom`, kararlı AAD nesne kimliklerini veya `accessGroup:core-team` gibi statik gönderici erişim gruplarını kullanmalıdır.
- İzin listeleri için UPN/görünen ad eşleştirmesine güvenmeyin; bunlar değişebilir. OpenClaw, doğrudan ad eşleştirmesini varsayılan olarak devre dışı bırakır; etkinleştirmek için `channels.msteams.dangerouslyAllowNameMatching: true` kullanın.
- Sihirbaz, kimlik bilgileri izin verdiğinde Microsoft Graph aracılığıyla adları kimliklere çözümleyebilir.

**Grup erişimi**

- Varsayılan: `channels.msteams.groupPolicy = "allowlist"` (`groupAllowFrom` eklemediğiniz sürece engellenir). `channels.msteams.groupPolicy` ayarlanmamışsa `channels.defaults.groupPolicy`, paylaşılan varsayılanı geçersiz kılabilir.
- `channels.msteams.groupAllowFrom`, grup sohbetlerinde/kanallarda hangi göndericilerin veya statik gönderici erişim gruplarının tetikleme yapabileceğini denetler (`channels.msteams.allowFrom` ayarına geri döner).
- Herhangi bir üyeye izin vermek için `groupPolicy: "open"` ayarını belirleyin (varsayılan olarak yine bahsetme koşulludur).
- **Tüm** kanalları engellemek için `channels.msteams.groupPolicy: "disabled"` ayarını belirleyin.

Örnek:

```json5
{
  channels: {
    msteams: {
      groupPolicy: "allowlist",
      groupAllowFrom: ["00000000-0000-0000-0000-000000000000", "accessGroup:core-team"],
    },
  },
}
```

**Ekip + kanal izin listesi**

- Ekipleri ve kanalları `channels.msteams.teams` altında listeleyerek grup/kanal yanıtlarının kapsamını belirleyin.
- Anahtar olarak değiştirilebilir görünen adları değil, Teams bağlantılarındaki kararlı Teams konuşma kimliklerini kullanın (bkz. [Ekip ve Kanal Kimlikleri](#team-and-channel-ids-common-gotcha)).
- `groupPolicy="allowlist"` ve bir ekip izin listesi mevcut olduğunda yalnızca listelenen ekipler/kanallar kabul edilir (bahsetme koşuluyla).
- Yapılandırma sihirbazı `Team/Channel` girdilerini kabul eder ve bunları sizin için saklar.
- Başlatma sırasında OpenClaw, ekip/kanal ve kullanıcı izin listesindeki adları kimliklere çözümler (Graph izinleri elverdiğinde) ve eşlemeyi günlüğe kaydeder. Çözümlenemeyen adlar yazıldığı hâliyle tutulur ancak `channels.msteams.dangerouslyAllowNameMatching: true` ayarlanmadıkça yönlendirme için yok sayılır.

Örnek:

```json5
{
  channels: {
    msteams: {
      groupPolicy: "allowlist",
      teams: {
        "My Team": {
          channels: {
            General: { requireMention: true },
          },
        },
      },
    },
  },
}
```

<details>
<summary><strong>Manuel kurulum (Teams CLI olmadan)</strong></summary>

### Nasıl çalışır?

1. Microsoft Teams plugin'inin kullanılabilir olduğundan emin olun (güncel sürümlerde paketle birlikte gelir).
2. Bir **Azure Bot** oluşturun (Uygulama Kimliği + gizli anahtar + kiracı kimliği).
3. Aşağıdaki RSC izinlerini içeren ve bota başvuran bir **Teams uygulama paketi** oluşturun.
4. Teams uygulamasını bir ekibe (veya DM'ler için kişisel kapsama) yükleyin/kurun.
5. `~/.openclaw/openclaw.json` içinde `msteams` ayarını yapılandırın (veya ortam değişkenlerini kullanın) ve Gateway'i başlatın.
6. Gateway, varsayılan olarak `/api/messages` üzerinde Bot Framework Webhook trafiğini dinler.

### 1. Adım: Azure Bot oluşturun

1. [Create Azure Bot](https://portal.azure.com/#create/Microsoft.AzureBot) sayfasına gidin.
2. **Basics** sekmesini doldurun:

   | Alan               | Değer                                                        |
   | ------------------ | ------------------------------------------------------------ |
   | **Bot handle**     | Botunuzun adı, ör. `openclaw-msteams` (benzersiz olmalıdır)  |
   | **Subscription**   | Azure aboneliğinizi seçin                                    |
   | **Resource group** | Yeni oluşturun veya mevcut olanı kullanın                     |
   | **Pricing tier**   | Geliştirme/test için **Free**                                 |
   | **Type of App**    | **Single Tenant** (önerilir; aşağıdaki nota bakın)            |
   | **Creation type**  | **Create new Microsoft App ID**                               |

<Warning>
Yeni çok kiracılı botların oluşturulması 2025-07-31 tarihinden sonra kullanımdan kaldırıldı. Yeni botlar için **Single Tenant** kullanın.
</Warning>

3. **Review + create**, ardından **Create** düğmesine tıklayın (~1-2 dakika).

### 2. Adım: Kimlik bilgilerini alın

1. Azure Bot kaynağı → **Configuration** → **Microsoft App ID** değerini kopyalayın (`appId` değeriniz).
2. **Manage Password** → App Registration → **Certificates & secrets** → **New client secret** → **Value** değerini kopyalayın (`appPassword` değeriniz).
3. **Overview** → **Directory (tenant) ID** değerini kopyalayın (`tenantId` değeriniz).

### 3. Adım: Mesajlaşma uç noktasını yapılandırın

1. Azure Bot → **Configuration**.
2. **Messaging endpoint** ayarını belirleyin:
   - Üretim: `https://your-domain.com/api/messages`
   - Yerel geliştirme: bir tünel kullanın (bkz. [Yerel geliştirme](#local-development-tunneling))

### 4. Adım: Teams kanalını etkinleştirin

1. Azure Bot → **Channels**.
2. **Microsoft Teams** → Configure → Save öğelerine tıklayın.
3. Hizmet Şartları'nı kabul edin.

### 5. Adım: Teams uygulama manifestini oluşturun

- `botId = <App ID>` içeren bir `bot` girdisi ekleyin.
- Kapsamlar: `personal`, `team`, `groupChat`.
- `supportsFiles: true` (kişisel kapsamda dosya işleme için gereklidir).
- RSC izinlerini ekleyin (bkz. [RSC izinleri](#current-teams-rsc-permissions-manifest)).
- Simgeleri oluşturun: `outline.png` (32x32) ve `color.png` (192x192).
- `manifest.json`, `outline.png` ve `color.png` dosyalarını birlikte zip arşivine alın.

### 6. Adım: OpenClaw'ı yapılandırın

```json5
{
  channels: {
    msteams: {
      enabled: true,
      appId: "<APP_ID>",
      appPassword: "<APP_PASSWORD>",
      tenantId: "<TENANT_ID>",
      webhook: { port: 3978, path: "/api/messages" },
    },
  },
}
```

Ortam değişkenleri: `MSTEAMS_APP_ID`, `MSTEAMS_APP_PASSWORD`, `MSTEAMS_TENANT_ID`.

### 7. Adım: Gateway'i çalıştırın

Plugin kullanılabilir olduğunda ve `msteams` yapılandırması kimlik bilgilerini içerdiğinde Teams kanalı otomatik olarak başlar.

</details>

## Federe kimlik doğrulaması (sertifika ve yönetilen kimlik)

OpenClaw, üretim ortamında istemci gizli anahtarlarına alternatif olarak `channels.msteams.authType: "federated"` aracılığıyla **federe kimlik doğrulamasını** destekler. İki yöntem vardır:

### A Seçeneği: Sertifika tabanlı kimlik doğrulaması

Entra ID uygulama kaydınızla kaydedilmiş bir PEM sertifikası kullanın.

**Kurulum:**

1. Bir sertifika oluşturun veya edinin (özel anahtar içeren PEM biçimi).
2. Entra ID → App Registration → **Certificates & secrets** → **Certificates** → ortak sertifikayı yükleyin.

**Yapılandırma:**

```json5
{
  channels: {
    msteams: {
      enabled: true,
      appId: "<APP_ID>",
      tenantId: "<TENANT_ID>",
      authType: "federated",
      certificatePath: "/path/to/cert.pem",
      webhook: { port: 3978, path: "/api/messages" },
    },
  },
}
```

**Ortam değişkenleri:**

- `MSTEAMS_AUTH_TYPE=federated`
- `MSTEAMS_CERTIFICATE_PATH=/path/to/cert.pem`

### B Seçeneği: Azure Managed Identity

Azure altyapısında (AKS, App Service, Azure VM'leri) parolasız kimlik doğrulaması için Azure Managed Identity kullanın.

**Nasıl çalışır?**

1. Bot pod'u/VM'si, yönetilen bir kimliğe (sistem veya kullanıcı tarafından atanmış) sahiptir.
2. Federe kimlik bilgisi, yönetilen kimliği Entra ID uygulama kaydına bağlar.
3. OpenClaw, çalışma zamanında Azure IMDS uç noktasından token almak için `@azure/identity` kullanır.
4. Token, bot kimlik doğrulaması için Teams SDK'ya iletilir.

**Ön koşullar:**

- Yönetilen kimliğin etkinleştirildiği Azure altyapısı (AKS iş yükü kimliği, App Service, VM).
- Entra ID uygulama kaydında oluşturulmuş federasyon kimlik bilgisi.
- Pod/VM'den IMDS'ye (`169.254.169.254:80`) ağ erişimi.

**Yapılandırma (sistem tarafından atanan yönetilen kimlik):**

```json5
{
  channels: {
    msteams: {
      enabled: true,
      appId: "<APP_ID>",
      tenantId: "<TENANT_ID>",
      authType: "federated",
      useManagedIdentity: true,
      webhook: { port: 3978, path: "/api/messages" },
    },
  },
}
```

**Yapılandırma (kullanıcı tarafından atanan yönetilen kimlik):** yukarıdaki bloğa `managedIdentityClientId: "<MI_CLIENT_ID>"` ekleyin.

**Ortam değişkenleri:**

- `MSTEAMS_AUTH_TYPE=federated`
- `MSTEAMS_USE_MANAGED_IDENTITY=true`
- `MSTEAMS_MANAGED_IDENTITY_CLIENT_ID=<client-id>` (yalnızca kullanıcı tarafından atanan)

### AKS İş Yükü Kimliği kurulumu

İş yükü kimliğini kullanan AKS dağıtımları için:

1. AKS kümenizde **iş yükü kimliğini etkinleştirin**.
2. Entra ID uygulama kaydında **bir federasyon kimlik bilgisi oluşturun**:

   ```bash
   az ad app federated-credential create --id <APP_OBJECT_ID> --parameters '{
     "name": "my-bot-workload-identity",
     "issuer": "<AKS_OIDC_ISSUER_URL>",
     "subject": "system:serviceaccount:<NAMESPACE>:<SERVICE_ACCOUNT>",
     "audiences": ["api://AzureADTokenExchange"]
   }'
   ```

3. Uygulama istemci kimliğiyle **Kubernetes hizmet hesabına ek açıklama ekleyin**:

   ```yaml
   apiVersion: v1
   kind: ServiceAccount
   metadata:
     name: my-bot-sa
     annotations:
       azure.workload.identity/client-id: "<APP_CLIENT_ID>"
   ```

4. İş yükü kimliği ekleme işlemi için **pod'u etiketleyin**:

   ```yaml
   metadata:
     labels:
       azure.workload.identity/use: "true"
   ```

5. IMDS'ye (`169.254.169.254`) **ağ erişimine izin verin**: NetworkPolicy kullanılıyorsa 80 numaralı bağlantı noktasında `169.254.169.254/32` için bir çıkış kuralı ekleyin.

### Kimlik doğrulama türlerinin karşılaştırması

| Yöntem               | Yapılandırma                                   | Avantajlar                          | Dezavantajlar                              |
| -------------------- | ---------------------------------------------- | ----------------------------------- | ------------------------------------------ |
| **İstemci gizli anahtarı** | `appPassword`                             | Basit kurulum                       | Gizli anahtar döndürme gerekir, daha az güvenlidir |
| **Sertifika**        | `authType: "federated"` + `certificatePath`        | Ağ üzerinden paylaşılan gizli anahtar yoktur | Sertifika yönetimi ek yükü                 |
| **Yönetilen Kimlik** | `authType: "federated"` + `useManagedIdentity`        | Parolasızdır, yönetilecek gizli anahtar yoktur | Azure altyapısı gerekir                    |

`certificateThumbprint`, `certificatePath` ile birlikte ayarlanabilir ancak şu anda kimlik doğrulama yolu tarafından okunmaz; yalnızca ileriye dönük uyumluluk için kabul edilir.

**Varsayılan:** `authType` ayarlanmadığında OpenClaw, istemci gizli anahtarıyla kimlik doğrulamasını (`appPassword`) kullanır. Mevcut yapılandırmalar değişiklik yapılmadan çalışmaya devam eder.

## Yerel geliştirme (tünelleme)

Teams, `localhost` adresine erişemez. URL'nin oturumlar arasında sabit kalması için kalıcı bir geliştirme tüneli kullanın:

```bash
# Tek seferlik kurulum:
devtunnel create my-openclaw-bot --allow-anonymous
devtunnel port create my-openclaw-bot -p 3978 --protocol auto

# Her geliştirme oturumunda:
devtunnel host my-openclaw-bot
```

Alternatifler: `ngrok http 3978` veya `tailscale funnel 3978` (URL'ler her oturumda değişebilir).

Tünel URL'si değişirse uç noktayı güncelleyin:

```bash
teams app update <teamsAppId> --endpoint "https://<new-url>/api/messages"
```

## Botu test etme

**Tanılamayı çalıştırın:**

```bash
teams app doctor <teamsAppId>
```

Bot kaydını, AAD uygulamasını, manifesti ve SSO yapılandırmasını tek geçişte denetler.

**Test iletisi gönderin:**

1. Teams uygulamasını yükleyin (`teams app get <id> --install-link` içindeki yükleme bağlantısı).
2. Teams'de botu bulun ve doğrudan ileti gönderin.
3. Gelen etkinlik için Gateway günlüklerini denetleyin.

## Ortam değişkenleri

Kimlik doğrulamayla ilgili bu yapılandırma anahtarları, `openclaw.json` yerine ortam değişkenleri aracılığıyla ayarlanabilir (örneğin `groupPolicy` veya `historyLimit` gibi diğer yapılandırma anahtarları yalnızca yapılandırma üzerinden ayarlanabilir):

| Ortam değişkeni                      | Yapılandırma anahtarı     | Notlar                              |
| ------------------------------------ | ------------------------- | ----------------------------------- |
| `MSTEAMS_APP_ID`                   | `appId`        |                                     |
| `MSTEAMS_APP_PASSWORD`                   | `appPassword`        |                                     |
| `MSTEAMS_TENANT_ID`                   | `tenantId`        |                                     |
| `MSTEAMS_AUTH_TYPE`                   | `authType`        | `"secret"` veya `"federated"` |
| `MSTEAMS_CERTIFICATE_PATH`                   | `certificatePath`        | federasyon + sertifika              |
| `MSTEAMS_CERTIFICATE_THUMBPRINT`                   | `certificateThumbprint`        | kabul edilir, kimlik doğrulama için gerekli değildir |
| `MSTEAMS_USE_MANAGED_IDENTITY`                   | `useManagedIdentity`        | federasyon + yönetilen kimlik       |
| `MSTEAMS_MANAGED_IDENTITY_CLIENT_ID`                   | `managedIdentityClientId`        | yalnızca kullanıcı tarafından atanan yönetilen kimlik |

## Üye bilgisi eylemi

OpenClaw, ajanların ve otomasyonların yapılandırılmış bir konuşma için doğrulanmış katılımcı listesi ayrıntılarını çözümleyebilmesi amacıyla Microsoft Teams için Graph destekli bir `member-info` eylemi sunar.

Gereksinimler:

- `ChannelSettings.Read.Group` ve `TeamMember.Read.Group` RSC izinleri (önerilen manifestte zaten bulunur).

Graph kimlik bilgileri yapılandırıldığında eylem kullanılabilir; ayrı bir `channels.msteams.actions.memberInfo` anahtarı yoktur.
Standart kanal aramaları; eşleşen ekip katılımcı listesi kimliğini, görünen adı, e-posta adresini ve rolleri döndürür.
Eylem, mevcut doğrudan ileti veya grup sohbetinde güvenilir gönderenin kararlı kullanıcı kimliğini döndürebilir.
Özel/paylaşılan kanal ve mevcut olmayan sohbet üyesi aramaları ek katılımcı listesi izinleri gerektirir
ve varsayılan izin temeli tarafından reddedilir.

## Geçmiş bağlamı

- `channels.msteams.historyLimit`, isteme kaç adet yakın tarihli kanal/grup iletisinin dahil edileceğini denetler. Önce `messages.groupChat.historyLimit` değerine geri döner, ardından varsayılan olarak 50 değerini kullanır. Devre dışı bırakmak için `0` ayarlayın.
- Getirilen ileti dizisi geçmişi, gönderen izin listelerine (`allowFrom` / `groupAllowFrom`) göre filtrelenir; dolayısıyla ileti dizisi bağlamının başlangıç verileri yalnızca izin verilen gönderenlerin iletilerini içerir.
- Alıntılanan ek bağlamı (bir yanıtın kendi eklerindeki Skype Reply şeması HTML'sinden ayrıştırılır) filtrelenmeden aktarılır; gönderen izin listesi filtresi şu anda yalnızca ileti dizisi geçmişinin başlangıç verilerine uygulanır.
- Doğrudan ileti geçmişi, `channels.msteams.dmHistoryLimit` (kullanıcı sıraları) ile sınırlandırılabilir. Kullanıcı başına geçersiz kılmalar: `channels.msteams.dms["<user_id>"].historyLimit`.

## Geçerli Teams RSC izinleri (manifest)

Bunlar, Teams uygulama manifestimizdeki **mevcut resourceSpecific izinleridir**. Yalnızca uygulamanın yüklü olduğu ekip/sohbet içinde geçerlidir.

**Kanallar için (ekip kapsamı):**

- `ChannelMessage.Read.Group` (Application) - tüm kanal iletilerini @bahsetme olmadan alır
- `ChannelMessage.Send.Group` (Application)
- `Member.Read.Group` (Application)
- `Owner.Read.Group` (Application)
- `ChannelSettings.Read.Group` (Application)
- `TeamMember.Read.Group` (Application)
- `TeamSettings.Read.Group` (Application)

**Grup sohbetleri için:**

- `ChatMessage.Read.Chat` (Application) - tüm grup sohbeti iletilerini @bahsetme olmadan alır

Teams CLI aracılığıyla RSC izinlerini ekleyin:

```bash
teams app rsc add <teamsAppId> ChannelMessage.Read.Group --type Application
```

## Örnek Teams manifesti (bilgileri gizlenmiş)

Gerekli alanları içeren asgari ve geçerli örnek. Kimlikleri ve URL'leri değiştirin.

```json5
{
  $schema: "https://developer.microsoft.com/en-us/json-schemas/teams/v1.23/MicrosoftTeams.schema.json",
  manifestVersion: "1.23",
  version: "1.0.0",
  id: "00000000-0000-0000-0000-000000000000",
  name: { short: "OpenClaw" },
  developer: {
    name: "Your Org",
    websiteUrl: "https://example.com",
    privacyUrl: "https://example.com/privacy",
    termsOfUseUrl: "https://example.com/terms",
  },
  description: { short: "OpenClaw in Teams", full: "OpenClaw in Teams" },
  icons: { outline: "outline.png", color: "color.png" },
  accentColor: "#5B6DEF",
  bots: [
    {
      botId: "11111111-1111-1111-1111-111111111111",
      scopes: ["personal", "team", "groupChat"],
      isNotificationOnly: false,
      supportsCalling: false,
      supportsVideo: false,
      supportsFiles: true,
    },
  ],
  webApplicationInfo: {
    id: "11111111-1111-1111-1111-111111111111",
  },
  authorization: {
    permissions: {
      resourceSpecific: [
        { name: "ChannelMessage.Read.Group", type: "Application" },
        { name: "ChannelMessage.Send.Group", type: "Application" },
        { name: "Member.Read.Group", type: "Application" },
        { name: "Owner.Read.Group", type: "Application" },
        { name: "ChannelSettings.Read.Group", type: "Application" },
        { name: "TeamMember.Read.Group", type: "Application" },
        { name: "TeamSettings.Read.Group", type: "Application" },
        { name: "ChatMessage.Read.Chat", type: "Application" },
      ],
    },
  },
}
```

### Manifest uyarıları (zorunlu alanlar)

- `bots[].botId`, Azure Bot App ID ile **eşleşmelidir**.
- `webApplicationInfo.id`, Azure Bot App ID ile **eşleşmelidir**.
- `bots[].scopes`, kullanmayı planladığınız yüzeyleri (`personal`, `team`, `groupChat`) içermelidir.
- Kişisel kapsamda dosya işleme için `bots[].supportsFiles: true` gereklidir.
- `authorization.permissions.resourceSpecific`, kanal trafiği için kanal okuma/gönderme izinlerini içermelidir.

### Mevcut bir uygulamayı güncelleme

```bash
# Manifesti indirin, düzenleyin ve yeniden yükleyin
teams app manifest download <teamsAppId> manifest.json
# manifest.json dosyasını yerel olarak düzenleyin...
teams app manifest upload manifest.json <teamsAppId>
# İçerik değiştiyse sürüm otomatik olarak artırılır
```

Güncellemeden sonra uygulamayı her ekibe yeniden yükleyin ve önbelleğe alınmış uygulama meta verilerini temizlemek için **Teams'den tamamen çıkıp yeniden başlatın** (yalnızca pencereyi kapatmayın).

<details>
<summary>Manuel manifest güncellemesi (CLI olmadan)</summary>

1. `manifest.json` öğesini yeni ayarlarla güncelleyin.
2. **`version` alanını artırın** (ör. `1.0.0` → `1.1.0`).
3. Manifesti simgelerle birlikte **yeniden zip dosyasına dönüştürün** (`manifest.json`, `outline.png`, `color.png`).
4. Yeni zip dosyasını yükleyin:
   - **Teams Admin Center:** Teams apps → Manage apps → uygulamanızı bulun → Upload new version.
   - **Sideload:** Teams → Apps → Manage your apps → Upload a custom app.

</details>

## Yetenekler: yalnızca RSC ve Graph karşılaştırması

### **Yalnızca Teams RSC** ile (uygulama yüklü, Graph API izni yok)

Çalışanlar:

- Kanal iletilerinin **metin** içeriğini okuma.
- Kanal iletilerinin **metin** içeriğini gönderme.
- **Kişisel (doğrudan ileti)** dosya eklerini alma.

Çalışmayanlar:

- Kanal/grup **görüntü veya dosya içerikleri** (yük yalnızca bir HTML taslağı içerir).
- SharePoint/OneDrive'da depolanan ekleri indirme.
- Canlı Webhook olayının ötesindeki ileti geçmişini okuma.

### **Teams RSC + Microsoft Graph Application izinleri** ile

Eklenenler:

- Barındırılan içeriği (iletilere yapıştırılmış görüntüler) indirme.
- SharePoint/OneDrive'da depolanan dosya eklerini indirme.
- Graph aracılığıyla kanal/sohbet iletisi geçmişini okuma.

### RSC ve Graph API karşılaştırması

| Yetenek                  | RSC izinleri                 | Graph API                                      |
| ------------------------ | ---------------------------- | ---------------------------------------------- |
| **Gerçek zamanlı iletiler** | Evet (webhook aracılığıyla) | Hayır (yalnızca yoklama)                       |
| **Geçmiş iletiler**      | Hayır                        | Evet (geçmiş sorgulanabilir)                   |
| **Kurulum karmaşıklığı** | Yalnızca uygulama manifesti  | Yönetici onayı + token akışı gerektirir        |
| **Çevrimdışı çalışır**   | Hayır (çalışıyor olmalıdır)  | Evet (istenildiği zaman sorgulanabilir)        |

**Özet:** RSC gerçek zamanlı dinleme, Graph API ise geçmişe erişim içindir. Çevrimdışıyken kaçırılan iletileri almak için `ChannelMessage.Read.All` ile Graph API gerekir (yönetici onayı gerektirir).

## Graph destekli medya + geçmiş

Yalnızca kullandığınız Teams kapsamları ve verileri için gereken Microsoft Graph uygulama izinlerini etkinleştirin:

1. Entra ID (Azure AD) **App Registration** → Graph **Application permissions** ekleyin:
   - Kanal ekleri ve kanal geçmişi için `ChannelMessage.Read.All`.
   - Grup sohbeti ekleri ve grup sohbeti geçmişi için `Chat.Read.All`.
   - Ek baytlarının SharePoint/OneDrive depolama alanından indirilmesi gerektiğinde `Files.Read.All`; yalnızca geçmiş kullanan kurulumlarda buna gerek yoktur.
2. Kiracı için **Grant admin consent** işlemini uygulayın.
3. Teams uygulamasının **manifest sürümünü** artırın, yeniden yükleyin ve **uygulamayı Teams'e yeniden kurun**.
4. Önbelleğe alınmış uygulama meta verilerini temizlemek için **Teams'i tamamen kapatıp yeniden başlatın**.

### Kanal/grup dosyası kurtarma (`graphMediaFallback`)

Teams, bir bota gönderilen HTML etkinliğinden dosya işaretçilerini kaldırabilir. Bu durumda Bot Framework etkinliği sıradan bir HTML iletisinden ayırt edilemez; eksiksiz ek referansı yalnızca iletinin Graph kopyasında bulunur.

Yukarıdaki izinleri verdikten sonra geri dönüş mekanizmasını etkinleştirin:

```json5
{
  channels: {
    msteams: {
      graphMediaFallback: true,
    },
  },
}
```

Bu yalnızca kanallar ve grup sohbetleri için geçerlidir. Bir HTML etkinliği doğrudan indirilebilir medya üretmediğinde, sıradan veya yalnızca bahsetme içeren iletiler dâhil olmak üzere bir Graph ileti araması ekler. Mevcut kurulumların otomatik olarak ek Graph trafiği veya izin hatalarıyla karşılaşmaması için varsayılan değer `false` şeklindedir.

**Kullanıcı bahsetmeleri:** @bahsetmeleri, zaten konuşmada bulunan kullanıcılar için doğrudan çalışır. **Mevcut konuşmada bulunmayan** kullanıcıları dinamik olarak aramak ve onlardan bahsetmek için `User.Read.All` (Application) iznini ekleyip yönetici onayı verin.

## Bilinen sınırlamalar

### Webhook zaman aşımları

Teams iletileri HTTP webhook aracılığıyla iletir. OpenClaw bu webhook dinleyicisine sabit HTTP sunucusu
zaman aşımları uygular: 30 sn hareketsizlik, toplam 30 sn istek ve üstbilgileri
almak için 15 sn. İsteğe bağlı gelen medya ve bağlam zenginleştirme, paylaşılan
10 saniyelik bir bütçeye sahiptir. SDK, ham etkinlik kalıcı olarak eklendikten sonra döner;
ajan dönüşü bağımsız olarak işlenir ve yanıtları proaktif biçimde gönderir. İstek
işleme veya kalıcı kabul işlemi aktarım penceresini kaçırırsa Teams
etkinliği yeniden deneyebilir ve giriş mezar taşı yinelenen bir etkinlik kimliğini reddeder.

### Teams bulutu ve hizmet URL'si desteği

SDK destekli bu Teams yolu, Microsoft Teams genel bulutu için canlı ortamda doğrulanmıştır.

Gelen yanıtlarda, gelen Teams SDK dönüş bağlamı kullanılır. Bağlam dışı proaktif işlemler (gönderimler, düzenlemeler, silmeler, kartlar, anketler, dosya onayı iletileri ve kuyruğa alınmış uzun süreli yanıtlar) depolanan `serviceUrl` konuşma referansını kullanır. Genel bulut, varsayılan olarak Teams SDK genel bulut ortamını kullanır ve genel Teams Connector ana makinesindeki depolanmış referanslara izin verir: `https://smba.trafficmanager.net/`.

Genel bulut varsayılandır. Normal genel bulut botları için `channels.msteams.cloud` veya `channels.msteams.serviceUrl` ayarlamanız gerekmez.

Genel olmayan Teams bulutları için `cloud` ve Microsoft yayımladığında eşleşen proaktif sınırı ayarlayın:

- `channels.msteams.cloud`; kimlik doğrulama, JWT doğrulaması, token hizmetleri ve Graph kapsamı için Teams SDK bulut ön ayarını seçer.
- `channels.msteams.serviceUrl`; proaktif gönderimler, düzenlemeler, silmeler, kartlar, anketler, dosya onayı iletileri ve kuyruğa alınmış uzun süreli yanıtlar öncesinde depolanan konuşma referanslarını doğrulamak için kullanılan Bot Connector uç noktası sınırını seçer. USGov ve DoD SDK bulutları için gereklidir. China/21Vianet için OpenClaw, SDK `China` ön ayarını kullanır ve depolanan/yapılandırılan hizmet URL'lerini yalnızca Azure China Bot Framework kanal ana makinelerinde kabul eder.

Microsoft, genel proaktif Bot Connector uç noktalarını Teams proaktif mesajlaşma belgelerinin [Konuşmayı oluşturma](https://learn.microsoft.com/en-us/microsoftteams/platform/bots/how-to/conversations/send-proactive-messages?tabs=dotnet#create-the-conversation) bölümünde yayımlar. Varsa gelen etkinliğin `serviceUrl` değerini kullanın; aksi takdirde aşağıdaki Microsoft tablosunu kullanın.

| Teams ortamı      | OpenClaw yapılandırması                                      | Proaktif `serviceUrl`                           |
| ----------------- | ----------------------------------------------------------- | ----------------------------------------------------- |
| Genel             | bulut/serviceUrl yapılandırması gerekmez                    | `https://smba.trafficmanager.net/teams`                                    |
| GCC               | `serviceUrl` ayarlayın; ayrı bir Teams SDK bulut ön ayarı yoktur | `https://smba.infra.gcc.teams.microsoft.com/teams`                         |
| GCC High          | `cloud: "USGov"` + `serviceUrl`                    | `https://smba.infra.gov.teams.microsoft.us/teams`                                    |
| DoD               | `cloud: "USGovDoD"` + `serviceUrl`                    | `https://smba.infra.dod.teams.microsoft.us/teams`                                    |
| China/21Vianet    | `cloud: "China"`                                          | gelen etkinliğin `serviceUrl` değerini kullanın |

Microsoft'un ayrı bir proaktif hizmet URL'si belgelediği ancak Teams SDK'nın ayrı bir GCC bulut ön ayarı sunmadığı GCC örneği:

```json
{
  "channels": {
    "msteams": {
      "serviceUrl": "https://smba.infra.gcc.teams.microsoft.com/teams"
    }
  }
}
```

GCC High örneği:

```json
{
  "channels": {
    "msteams": {
      "cloud": "USGov",
      "serviceUrl": "https://smba.infra.gov.teams.microsoft.us/teams"
    }
  }
}
```

`channels.msteams.serviceUrl`, desteklenen Microsoft Teams Bot Connector ana makineleriyle sınırlıdır. Bir hizmet URL'si yapılandırıldığında OpenClaw; proaktif gönderimler, düzenlemeler, silmeler, kartlar, anketler veya kuyruğa alınmış uzun süreli yanıtlar çalıştırılmadan önce depolanan konuşmanın `serviceUrl` değerinin aynı ana makineyi kullandığını denetler. Varsayılan genel bulut yapılandırmasında, depolanan bir konuşma genel Teams Connector ana makinesinin dışını gösteriyorsa OpenClaw güvenli biçimde başarısız olur. Depolanan konuşma referansının güncel olması için bulut/hizmet URL'si ayarlarını değiştirdikten sonra konuşmadan yeni bir ileti alın.

Microsoft'un Teams proaktif uç nokta tablosunda China/21Vianet için ayrı bir genel proaktif `smba` URL'si yoktur. Teams SDK'nın Azure China kimlik doğrulama, token ve JWT uç noktalarını kullanması için `cloud: "China"` yapılandırın. Bundan sonra proaktif gönderimler, Azure China Bot Framework kanal sınırında (`*.botframework.azure.cn`) gelen bir China Teams etkinliğinden depolanmış bir konuşma referansı veya açıkça yapılandırılmış bir hizmet URL'si gerektirir. OpenClaw, Graph isteklerini Azure China Graph uç noktası üzerinden yönlendirene kadar Graph destekli Teams yardımcıları `cloud: "China"` için devre dışıdır.

### Biçimlendirme

Teams markdown'ı Slack veya Discord'a göre daha sınırlıdır:

- Temel biçimlendirme çalışır: **kalın**, _italik_, `code`, bağlantılar.
- Karmaşık markdown (tablolar, iç içe listeler) doğru görüntülenmeyebilir.
- Anketler ve anlamsal sunum gönderimleri için Adaptive Cards desteklenir (aşağıya bakın).

## Yapılandırma

Temel ayarlar (paylaşılan kanal kalıpları için [/gateway/configuration](/tr/gateway/configuration) sayfasına bakın):

- `channels.msteams.enabled`: kanalı etkinleştirin/devre dışı bırakın.
- `channels.msteams.appId`, `channels.msteams.appPassword`, `channels.msteams.tenantId`: bot kimlik bilgileri.
- `channels.msteams.cloud`: Teams SDK bulut ortamı (`Public`, `USGov`, `USGovDoD` veya `China`; varsayılan `Public`). USGov/DoD SDK bulutları için `serviceUrl` ile ayarlayın; Çin, SDK ön ayarını ve depolanan Azure China Bot Framework konuşma referanslarını kullanır; Azure China Graph yönlendirmesi kullanıma sunulana kadar Graph destekli yardımcılar devre dışıdır.
- `channels.msteams.serviceUrl`: SDK proaktif işlemleri için Bot Connector hizmet URL'si sınırı. Genel bulut SDK varsayılanını kullanır; GCC (`https://smba.infra.gcc.teams.microsoft.com/teams`), GCC High veya DoD için ayarlayın. Depolanan konuşma referansı 21Vianet tarafından işletilen Teams'den geldiğinde Çin, Azure China Bot Framework kanal ana makinelerini kabul eder.
- `channels.msteams.webhook.port` (varsayılan `3978`).
- `channels.msteams.webhook.path` (varsayılan `/api/messages`).
- `channels.msteams.dmPolicy`: `pairing | allowlist | open | disabled` (varsayılan `pairing`).
- `channels.msteams.allowFrom`: DM izin listesi (AAD nesne kimlikleri önerilir). Graph erişimi kullanılabilir olduğunda sihirbaz, kurulum sırasında adları kimliklere çözümler.
- `channels.msteams.dangerouslyAllowNameMatching`: değiştirilebilir UPN/görünen ad eşleştirmesini ve doğrudan ekip/kanal adı yönlendirmesini yeniden etkinleştiren acil durum anahtarı.
- `channels.msteams.textChunkLimit`: karakter cinsinden giden metin parçası boyutu (varsayılan `4000`; yapılandırılan değer daha yüksek olsa bile kesin üst sınır `4000`).
- `channels.msteams.streaming.chunkMode`: uzunluğa göre parçalara ayırmadan önce boş satırlardan (paragraf sınırlarından) bölmek için `length` (varsayılan) veya `newline`.
- `channels.msteams.mediaAllowHosts`: gelen eklerin ana makineleri için izin listesi (varsayılan olarak Microsoft/Teams etki alanları: Graph, SharePoint/OneDrive, Teams CDN, Bot Framework, Azure Media Services).
- `channels.msteams.mediaAuthAllowHosts`: medya yeniden denemelerinde Authorization üstbilgilerinin eklenmesine yönelik izin listesi (varsayılan olarak Graph + Bot Framework ana makineleri).
- `channels.msteams.graphMediaFallback`: kanal/grup HTML'si dosya işaretçilerini içermediğinde Graph ileti aramalarını etkinleştirin (varsayılan `false`; bkz. [Kanal/grup dosya kurtarma](#channelgroup-file-recovery-graphmediafallback)).
- `channels.msteams.mediaMaxMb`: kanal başına medya boyutu sınırını MB cinsinden geçersiz kılma. Ayarlanmadığında `agents.defaults.mediaMaxMb` değerine geri döner.
- `channels.msteams.requireMention`: kanallarda/gruplarda @bahsetme gerektirir (varsayılan `true`).
- `channels.msteams.replyStyle`: `thread | top-level` (bkz. [Yanıt biçimi](#reply-style-threads-vs-posts)).
- `channels.msteams.teams.<teamId>.replyStyle`: ekip başına geçersiz kılma.
- `channels.msteams.teams.<teamId>.requireMention`: ekip başına geçersiz kılma.
- `channels.msteams.teams.<teamId>.tools`: kanal geçersiz kılması eksik olduğunda kullanılan varsayılan ekip başına araç politikası geçersiz kılmaları (`allow`/`deny`/`alsoAllow`).
- `channels.msteams.teams.<teamId>.toolsBySender`: ekip başına ve gönderen başına varsayılan araç politikası geçersiz kılmaları (`"*"` joker karakteri desteklenir).
- `channels.msteams.teams.<teamId>.channels.<conversationId>.replyStyle`: kanal başına geçersiz kılma.
- `channels.msteams.teams.<teamId>.channels.<conversationId>.requireMention`: kanal başına geçersiz kılma.
- `channels.msteams.teams.<teamId>.channels.<conversationId>.tools`: kanal başına araç politikası geçersiz kılmaları (`allow`/`deny`/`alsoAllow`).
- `channels.msteams.teams.<teamId>.channels.<conversationId>.toolsBySender`: kanal başına ve gönderen başına araç politikası geçersiz kılmaları (`"*"` joker karakteri desteklenir).
- `toolsBySender` anahtarları açık önekler kullanmalıdır: `channel:`, `id:`, `e164:`, `username:`, `name:` (önek içermeyen eski anahtarlar hâlâ yalnızca `id:` ile eşlenir).
- `channels.msteams.authType`: kimlik doğrulama türü - `"secret"` (varsayılan) veya `"federated"`.
- `channels.msteams.certificatePath`: PEM sertifika dosyasının yolu (federe + sertifika kimlik doğrulaması).
- `channels.msteams.certificateThumbprint`: sertifika parmak izi; kabul edilir, kimlik doğrulaması için zorunlu değildir.
- `channels.msteams.useManagedIdentity`: yönetilen kimlik doğrulamasını etkinleştirin (federe mod).
- `channels.msteams.managedIdentityClientId`: kullanıcı tarafından atanan yönetilen kimlik için istemci kimliği.
- `channels.msteams.sharePointSiteId`: grup sohbetlerinde/kanallarda dosya yüklemeleri için SharePoint site kimliği (bkz. [Grup sohbetlerinde dosya gönderme](#sending-files-in-group-chats)).
- `channels.msteams.welcomeCard`, `channels.msteams.groupWelcomeCard`, `channels.msteams.promptStarters`: ilk DM/grup iletişiminde gösterilen karşılama Adaptive Card'ı ve önerilen istem düğmeleri.
- `channels.msteams.responsePrefix`: giden yanıtların başına eklenen metin.
- `channels.msteams.feedbackEnabled` (varsayılan `true`), `channels.msteams.feedbackReflection` (varsayılan `true`), `channels.msteams.feedbackReflectionCooldownMs`: yanıtlarda başparmak yukarı/aşağı geri bildirimi ve olumsuz geri bildirim üzerine düşünme takibi.
- `channels.msteams.sso`, `channels.msteams.delegatedAuth`: SSO destekli akışlar için Bot Framework OAuth bağlantısı ve devredilmiş Graph kapsamları; `sso.enabled: true`, `sso.connectionName` gerektirir.

## Yönlendirme ve oturumlar

- Oturum anahtarları standart aracı biçimini izler (bkz. [/concepts/session](/tr/concepts/session)):
  - Doğrudan iletiler ana oturumu paylaşır (`agent:<agentId>:<mainKey>`).
  - Kanal/grup iletileri konuşma kimliğini kullanır:
    - `agent:<agentId>:msteams:channel:<conversationId>`
    - `agent:<agentId>:msteams:group:<conversationId>`

## Yanıt biçimi: ileti dizileri ve gönderiler

Teams, aynı temel veri modeli üzerinde iki kanal kullanıcı arayüzü biçimine sahiptir:

| Biçim                    | Açıklama                                               | Önerilen `replyStyle` |
| ------------------------ | --------------------------------------------------------- | ------------------------ |
| **Gönderiler** (klasik)      | İletiler, altında ileti dizili yanıtlar bulunan kartlar olarak görünür | `thread` (varsayılan)       |
| **İleti dizileri** (Slack benzeri) | İletiler, Slack'e daha benzer biçimde doğrusal olarak akar                   | `top-level`              |

**Sorun:** Teams API'si bir kanalın hangi kullanıcı arayüzü biçimini kullandığını göstermez. Yanlış `replyStyle` kullanırsanız:

- İleti dizisi biçimindeki bir kanalda `thread` → yanıtlar uygunsuz biçimde iç içe görünür.
- Gönderi biçimindeki bir kanalda `top-level` → yanıtlar ileti dizisi içinde olmak yerine ayrı üst düzey gönderiler olarak görünür.

**Çözüm:** kanalın nasıl ayarlandığına bağlı olarak `replyStyle` değerini kanal başına yapılandırın:

```json5
{
  channels: {
    msteams: {
      replyStyle: "thread",
      teams: {
        "19:abc...@thread.tacv2": {
          channels: {
            "19:xyz...@thread.tacv2": {
              replyStyle: "top-level",
            },
          },
        },
      },
    },
  },
}
```

### Çözümleme önceliği

Bot bir kanala yanıt gönderdiğinde `replyStyle`, en özel geçersiz kılmadan varsayılana doğru çözümlenir. `undefined` olmayan ilk değer geçerli olur:

1. **Kanal başına** - `channels.msteams.teams.<teamId>.channels.<conversationId>.replyStyle`
2. **Ekip başına** - `channels.msteams.teams.<teamId>.replyStyle`
3. **Genel** - `channels.msteams.replyStyle`
4. **Örtük varsayılan** - `requireMention` üzerinden türetilir:
   - `requireMention: true` → `thread`
   - `requireMention: false` → `top-level`

Açık bir `replyStyle` olmadan `requireMention: false` değerini genel olarak ayarlarsanız, gelen ileti bir ileti dizisi yanıtı olsa bile Gönderi biçimindeki kanallarda bahsetmeler üst düzey gönderiler olarak görünür. Beklenmedik durumları önlemek için `replyStyle: "thread"` değerini genel, ekip veya kanal düzeyinde sabitleyin.

Depolanan bir kanal konuşmasına yapılan proaktif gönderimlerde (kuyruğa alınmış araç çağrısı yanıtları, uzun süre çalışan aracılar) aynı ekip/kanal çözümlemesi uygulanır; grup sohbetleri ve kişisel (DM) konuşmalar, `replyStyle` değerinden bağımsız olarak proaktif gönderimler için her zaman `top-level` değerine çözümlenir.

### İleti dizisi bağlamını koruma

`replyStyle: "thread"` geçerliyken ve bottan bir kanal ileti dizisinin içinden @bahsedildiğinde OpenClaw, yanıtın aynı ileti dizisinin içine düşmesi için özgün ileti dizisi kökünü giden konuşma referansına (`19:...@thread.tacv2;messageid=<root>`) yeniden ekler. Bu, hem canlı (aynı tur içindeki) gönderimler hem de Bot Framework tur bağlamı sona erdikten sonra yapılan proaktif gönderimler (ör. uzun süre çalışan aracılar, `mcp__openclaw__message` üzerinden kuyruğa alınmış araç çağrısı yanıtları) için geçerlidir.

İleti dizisi kökü, konuşma referansında depolanan `threadId` değerinden alınır. `threadId` öncesinden kalan eski depolanmış referanslar, `activityId` değerine (konuşmayı en son başlatan gelen etkinlik neyse ona) geri döner; böylece mevcut dağıtımların yeniden başlatılmadan çalışmayı sürdürmesi sağlanır.

`replyStyle: "top-level"` geçerliyken kanal ileti dizilerinden gelen iletiler kasıtlı olarak yeni üst düzey gönderilerle yanıtlanır; ileti dizisi soneki eklenmez. Bu, İleti dizisi biçimindeki kanallar için doğrudur; ileti dizili yanıtlar beklediğiniz yerde üst düzey gönderiler görünüyorsa `replyStyle` o kanal için yanlış ayarlanmıştır.

## Ekler ve görüntüler

**Mevcut sınırlamalar:**

- **DM'ler:** görüntüler ve dosya ekleri Teams bot dosya API'leri üzerinden çalışır.
- **Kanallar/gruplar:** ekler M365 depolama alanında (SharePoint/OneDrive) bulunur. Webhook yükü gerçek dosya baytlarını değil, yalnızca bir HTML taslağını içerir. Kanal eklerini indirmek için **Graph API izinleri gereklidir**.
- Açıkça önce dosya göndermek için `action=upload-file` değerini `media` / `filePath` / `path` ile kullanın; isteğe bağlı `message` eşlik eden metin/yorum olur ve `filename` (veya `title`) yüklenen adı geçersiz kılar.

Graph izinleri olmadan görüntü içeren kanal iletileri yalnızca metin olarak gelir (görüntü içeriğine bot tarafından erişilemez).
OpenClaw varsayılan olarak medyayı yalnızca Microsoft/Teams ana makine adlarından indirir. `channels.msteams.mediaAllowHosts` ile geçersiz kılın (herhangi bir ana makineye izin vermek için `["*"]` kullanın).
Authorization üstbilgileri yalnızca `channels.msteams.mediaAuthAllowHosts` içindeki ana makineler için eklenir (varsayılan olarak Graph + Bot Framework ana makineleri). Bu listeyi sıkı tutun (çok kiracılı soneklerden kaçının).

## Grup sohbetlerinde dosya gönderme

Botlar, yerleşik FileConsentCard akışını kullanarak DM'lerde dosya gönderebilir. **Grup sohbetlerinde/kanallarda dosya gönderme** ek kurulum gerektirir:

| Bağlam                  | Dosyaların gönderilme biçimi                           | Gerekli kurulum                                    |
| ------------------------ | -------------------------------------------- | ----------------------------------------------- |
| **DM'ler**                  | FileConsentCard → kullanıcı kabul eder → bot yükler | Kullanıma hazır olarak çalışır                            |
| **Grup sohbetleri/kanallar** | SharePoint'e yükleme → yerel dosya kartı      | `sharePointSiteId` + Graph izinleri gerektirir |
| **Görüntüler (herhangi bir bağlam)** | Base64 kodlu satır içi                        | Kullanıma hazır olarak çalışır                            |

### Grup sohbetleri neden SharePoint gerektirir?

Botlar bir uygulama kimliği kullanırken Microsoft Graph'ın `/me` kaynağı [oturum açmış bir kullanıcı gerektirir](https://learn.microsoft.com/en-us/graph/api/user-get?view=graph-rest-1.0). Grup sohbetlerinde/kanallarda dosya göndermek için bot, dosyayı bir **SharePoint sitesine** yükler ve bir paylaşım bağlantısı oluşturur.

### Kurulum

1. Entra ID (Azure AD) → App Registration içinde **Graph API izinlerini ekleyin**:
   - `Sites.ReadWrite.All` (Uygulama) - SharePoint'e dosya yükleyin.
   - `ChatMember.Read.All` (Uygulama) - grup sohbeti dosya gönderimleri için en az ayrıcalıklı, kiracı genelinde izin. `Chat.Read.All` de çalışır ve grup sohbeti geçmişi etkinleştirildiğinde bunu zaten kapsar. Sohbet başına alternatif olarak `ChatMember.Read.Chat` [kaynağa özgü onay iznini](https://learn.microsoft.com/en-us/microsoftteams/platform/graph-api/rsc/resource-specific-consent) kullanın.
2. Kiracı için **yönetici onayı verin**.
3. **SharePoint site kimliğinizi alın:**

   ```bash
   # Graph Explorer veya geçerli bir belirteçle curl aracılığıyla:
   curl -H "Authorization: Bearer $TOKEN" \
     "https://graph.microsoft.com/v1.0/sites/{hostname}:/{site-path}"

   # Örnek: "contoso.sharepoint.com/sites/BotFiles" konumundaki bir site için
   curl -H "Authorization: Bearer $TOKEN" \
     "https://graph.microsoft.com/v1.0/sites/contoso.sharepoint.com:/sites/BotFiles"

   # Yanıt şunu içerir: "id": "contoso.sharepoint.com,guid1,guid2"
   ```

4. **OpenClaw'ı yapılandırın:**

   ```json5
   {
     channels: {
       msteams: {
         // ... diğer yapılandırmalar ...
         sharePointSiteId: "contoso.sharepoint.com,guid1,guid2",
       },
     },
   }
   ```

### Paylaşım davranışı

| Bağlam ve izin                                                           | Paylaşım davranışı                                                        |
| ------------------------------------------------------------------------ | ------------------------------------------------------------------------- |
| Kanal + `Sites.ReadWrite.All`                                               | Kuruluş genelinde paylaşım bağlantısı (kuruluştaki herkes erişebilir)      |
| Grup sohbeti + `Sites.ReadWrite.All` + desteklenen bir sohbet üyesi okuma izni | Kullanıcı başına paylaşım bağlantısı (yalnızca sohbet üyeleri erişebilir) |
| Desteklenen bir sohbet üyesi okuma izni olmayan grup sohbeti             | Gönderim güvenli biçimde başarısız olur                                   |

Yalnızca sohbet katılımcıları dosyaya erişebildiğinden kullanıcı başına paylaşım daha güvenlidir. OpenClaw, grup sohbetleri için başarılı bir üye sorgulaması gerektirir; zaman aşımları, aktarım hataları, boş sonuçlar ve Graph API retleri, erişimi kuruluşun geneline genişletmek yerine gönderimin başarısız olmasına neden olur.

### Geri dönüş davranışı

| Senaryo                                                               | Sonuç                                                     |
| ---------------------------------------------------------------------- | --------------------------------------------------------- |
| Grup sohbeti + dosya + yapılandırılmış SharePoint ve üye izinleri      | SharePoint'e yükle, yerel bir dosya kartı gönder          |
| Grup sohbeti + dosya + eksik SharePoint veya üye izinleri              | Uygulanabilir bir yapılandırma hatasıyla başarısız ol      |
| Kanal + dosya + yapılandırılmış `sharePointSiteId`                     | SharePoint'e yükle, yerel bir dosya kartı gönder          |
| Kişisel sohbet + dosya                                                 | FileConsentCard akışı (SharePoint olmadan çalışır)        |
| Herhangi bir bağlam + görüntü                                          | Base64 kodlu satır içi veri (SharePoint olmadan çalışır)  |

### Dosyaların depolandığı konum

Yüklenen dosyalar, yapılandırılmış SharePoint sitesinin varsayılan belge kitaplığındaki bir `/OpenClawShared/` klasöründe depolanır.

## Anketler (Adaptive Cards)

OpenClaw, Teams anketlerini Adaptive Cards olarak gönderir (yerel bir Teams anket API'si yoktur).

- CLI: `openclaw message poll --channel msteams --target conversation:<id> --poll-question "..." --poll-option "..." --poll-option "..."`.
- Oylar, Gateway tarafından OpenClaw Plugin durumu SQLite veritabanında `state/openclaw.sqlite` altında kaydedilir.
- Mevcut `msteams-polls.json` dosyaları, çalışan Plugin tarafından değil, `openclaw doctor --fix` tarafından içe aktarılır.
- Oyların kaydedilmesi için Gateway çevrimiçi kalmalıdır.
- Anketler sonuç özetlerini otomatik olarak yayımlamaz ve henüz bir anket sonuçları CLI'si yoktur.

## Sunum kartları

`message` aracını, CLI'yi veya normal yanıt teslimini kullanarak Teams kullanıcılarına ya da konuşmalarına anlamsal sunum yükleri gönderin. OpenClaw bunları genel sunum sözleşmesinden Teams Adaptive Cards olarak işler.

`presentation` parametresi anlamsal blokları kabul eder. `presentation` sağlandığında mesaj metni isteğe bağlıdır. Düğmeler, Adaptive Card gönderme veya URL eylemleri olarak işlenir. Seçim menüleri Teams işleyicisinde yerel olarak desteklenmediğinden OpenClaw, teslimden önce bunları okunabilir metne dönüştürür.

**Aracı aracı:**

```json5
{
  action: "send",
  channel: "msteams",
  target: "user:<id>",
  presentation: {
    title: "Merhaba",
    blocks: [{ type: "text", text: "Merhaba!" }],
  },
}
```

**CLI:**

```bash
openclaw message send --channel msteams \
  --target "conversation:19:abc...@thread.tacv2" \
  --presentation '{"title":"Merhaba","blocks":[{"type":"text","text":"Merhaba!"}]}'
```

Hedef biçimi ayrıntıları için aşağıdaki [Hedef biçimleri](#target-formats) bölümüne bakın.

## Hedef biçimleri

MSTeams hedefleri, kullanıcılarla konuşmaları ayırt etmek için ön ekler kullanır:

| Hedef türü           | Biçim                            | Örnek                                                                                                  |
| -------------------- | -------------------------------- | ------------------------------------------------------------------------------------------------------ |
| Kullanıcı (kimlikle) | `user:<aad-object-id>`               | `user:40a1a0ed-4ff2-4164-a219-55518990c197`                                                                                     |
| Kullanıcı (adla)     | `user:<display-name>`               | `user:John Smith` (Graph API gerektirir)                                                              |
| Grup/kanal           | `conversation:<conversation-id>`               | `conversation:19:abc123...@thread.tacv2`                                                                                     |
| Grup/kanal (ham)     | `<conversation-id>`               | `19:abc123...@thread.tacv2`, `19:...@unq.gbl.spaces` veya çıplak bir `a:`/`8:orgid:`/`29:` Bot Framework kimliği |

**CLI örnekleri:**

```bash
# Kimliğe göre bir kullanıcıya gönder
openclaw message send --channel msteams --target "user:40a1a0ed-..." --message "Merhaba"

# Görünen ada göre bir kullanıcıya gönder (Graph API sorgulamasını tetikler)
openclaw message send --channel msteams --target "user:John Smith" --message "Merhaba"

# Bir grup sohbetine veya kanala gönder
openclaw message send --channel msteams --target "conversation:19:abc...@thread.tacv2" --message "Merhaba"

# Bir konuşmaya sunum kartı gönder
openclaw message send --channel msteams --target "conversation:19:abc...@thread.tacv2" \
  --presentation '{"title":"Merhaba","blocks":[{"type":"text","text":"Merhaba"}]}'
```

**Aracı aracı örnekleri:**

```json5
{
  action: "send",
  channel: "msteams",
  target: "user:John Smith",
  message: "Merhaba!",
}
```

```json5
{
  action: "send",
  channel: "msteams",
  target: "conversation:19:abc...@thread.tacv2",
  presentation: {
    title: "Merhaba",
    blocks: [{ type: "text", text: "Merhaba" }],
  },
}
```

<Note>
`user:` ön eki olmadan adlar varsayılan olarak grup veya ekip çözümlemesine yönlendirilir. Kişileri görünen adlarıyla hedeflerken her zaman `user:` kullanın.
</Note>

## Proaktif mesajlaşma

- OpenClaw konuşma başvurularını bu noktada depoladığı için proaktif mesajlar yalnızca bir kullanıcı etkileşimde bulunduktan **sonra** gönderilebilir.
- `dmPolicy` ve izin verilenler listesi denetimi için [/gateway/configuration](/tr/gateway/configuration) bölümüne bakın.

## Ekip ve Kanal Kimlikleri (Yaygın Sorun)

Teams URL'lerindeki `groupId` sorgu parametresi, yapılandırmada kullanılan ekip kimliği **DEĞİLDİR**. Bunun yerine kimlikleri URL yolundan çıkarın:

**Ekip URL'si:**

```text
https://teams.microsoft.com/l/team/19%3ABk4j...%40thread.tacv2/conversations?groupId=...
                                    └────────────────────────────┘
                                    Ekip konuşması kimliği (bunun URL kodunu çözün)
```

**Kanal URL'si:**

```text
https://teams.microsoft.com/l/channel/19%3A15bc...%40thread.tacv2/ChannelName?groupId=...
                                      └─────────────────────────┘
                                      Kanal kimliği (bunun URL kodunu çözün)
```

**Yapılandırma için:**

- Ekip anahtarı = `/team/` sonrasındaki yol bölümü (URL kodu çözülmüş, ör. `19:Bk4j...@thread.tacv2`; eski kiracılarda yine geçerli olan `@thread.skype` görünebilir).
- Kanal anahtarı = `/channel/` sonrasındaki yol bölümü (URL kodu çözülmüş).
- OpenClaw yönlendirmesi için `groupId` sorgu parametresini **yok sayın**. Bu, gelen Teams etkinliklerinde kullanılan Bot Framework konuşma kimliği değil, Microsoft Entra grup kimliğidir.

## Özel kanallar

Botların özel kanallardaki desteği sınırlıdır:

| Özellik                      | Standart kanallar | Özel kanallar               |
| ---------------------------- | ----------------- | --------------------------- |
| Bot yükleme                  | Evet              | Sınırlı                     |
| Gerçek zamanlı mesajlar (Webhook) | Evet         | Çalışmayabilir              |
| RSC izinleri                 | Evet              | Farklı davranabilir         |
| @bahsetmeler                 | Evet              | Bot erişilebilirse          |
| Graph API geçmişi            | Evet              | Evet (izinlerle)            |

**Özel kanallar çalışmıyorsa geçici çözümler:**

1. Bot etkileşimleri için standart kanalları kullanın.
2. DM'leri kullanın; kullanıcılar her zaman bota doğrudan mesaj gönderebilir.
3. Geçmiş erişimi için Graph API'yi kullanın (`ChannelMessage.Read.All` gerektirir).

## Sorun giderme

### Yaygın sorunlar

- **Görüntüler kanallarda görünmüyor:** Graph izinleri veya yönetici onayı eksik. Teams uygulamasını yeniden yükleyin ve Teams'i tamamen kapatıp yeniden açın.
- **Kanalda yanıt yok:** varsayılan olarak bahsetmeler gereklidir; `channels.msteams.requireMention=false` ayarını belirleyin veya ekip/kanal başına yapılandırın.
- **Sürüm uyuşmazlığı (Teams hâlâ eski manifesti gösteriyor):** uygulamayı kaldırıp yeniden ekleyin ve yenilemek için Teams'i tamamen kapatın.
- **Webhook'tan 401 Unauthorized:** Azure JWT olmadan elle test yaparken beklenen bir durumdur; uç noktaya erişilebildiğini ancak kimlik doğrulamanın başarısız olduğunu gösterir. Doğru şekilde test etmek için Azure Web Chat'i kullanın.

### Manifest yükleme hataları

- **"Icon file cannot be empty":** manifest, 0 bayt boyutundaki simge dosyalarına başvuruyor. Geçerli PNG simgeleri oluşturun (`outline.png` için 32x32, `color.png` için 192x192).
- **"webApplicationInfo.Id already in use":** uygulama hâlâ başka bir ekipte/sohbette yüklü. Önce uygulamayı bulup kaldırın veya yayılım için 5-10 dakika bekleyin.
- **Yükleme sırasında "Something went wrong":** bunun yerine [https://admin.teams.microsoft.com](https://admin.teams.microsoft.com) üzerinden yükleyin, tarayıcı DevTools'u (F12) → Network sekmesini açın ve gerçek hata için yanıt gövdesini denetleyin.
- **Sideload başarısız oluyor:** "Upload a custom app" yerine "Upload an app to your org's app catalog" seçeneğini deneyin; bu genellikle sideload kısıtlamalarını aşar.

### RSC izinleri çalışmıyor

1. `webApplicationInfo.id` değerinin botunuzun App ID'siyle tam olarak eşleştiğini doğrulayın.
2. Uygulamayı yeniden yükleyin ve ekipte/sohbette yeniden kurun.
3. Kuruluş yöneticinizin RSC izinlerini engelleyip engellemediğini denetleyin.
4. Doğru kapsamı kullandığınızı doğrulayın: ekipler için `ChannelMessage.Read.Group`, grup sohbetleri için `ChatMessage.Read.Chat`.

## Kaynaklar

- [Azure Bot oluşturma](https://learn.microsoft.com/en-us/azure/bot-service/bot-service-quickstart-registration) - Azure Bot kurulum kılavuzu
- [Teams Developer Portal](https://dev.teams.microsoft.com/apps) - Teams uygulamaları oluşturma/yönetme
- [Teams uygulama manifesti şeması](https://learn.microsoft.com/en-us/microsoftteams/platform/resources/schema/manifest-schema)
- [RSC ile kanal mesajlarını alma](https://learn.microsoft.com/en-us/microsoftteams/platform/bots/how-to/conversations/channel-messages-with-rsc)
- [RSC izinleri başvurusu](https://learn.microsoft.com/en-us/microsoftteams/platform/graph-api/rsc/resource-specific-consent)
- [Teams bot dosya işleme](https://learn.microsoft.com/en-us/microsoftteams/platform/bots/how-to/bots-filesv4) (kanal/grup Graph gerektirir)
- [Proaktif mesajlaşma](https://learn.microsoft.com/en-us/microsoftteams/platform/bots/how-to/conversations/send-proactive-messages)
- [@microsoft/teams.cli](https://www.npmjs.com/package/@microsoft/teams.cli) - bot yönetimi için Teams CLI

## İlgili içerikler

- [Kanallara Genel Bakış](/tr/channels) - desteklenen tüm kanallar
- [Eşleştirme](/tr/channels/pairing) - DM kimlik doğrulaması ve eşleştirme akışı
- [Gruplar](/tr/channels/groups) - grup sohbeti davranışı ve bahsetme koşulu
- [Kanal Yönlendirme](/tr/channels/channel-routing) - iletiler için oturum yönlendirmesi
- [Güvenlik](/tr/gateway/security) - erişim modeli ve güvenlik sıkılaştırması
