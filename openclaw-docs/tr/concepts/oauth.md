---
read_when:
    - OpenClaw OAuth'ı uçtan uca anlamak istiyorsunuz
    - Token geçersiz kılma / oturum kapatma sorunlarıyla karşılaşıyorsunuz
    - Claude CLI veya OAuth kimlik doğrulama akışlarını istiyorsunuz
    - Birden fazla hesap veya profil yönlendirmesi istiyorsanız
summary: 'OpenClaw''da OAuth: token alışverişi, depolama ve çoklu hesap kalıpları'
title: OAuth
x-i18n:
    generated_at: "2026-07-26T23:57:11Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 3ef94af0601b7d57bb7e2d53c3d8231708b401251eca7dc1bb1e7e4fc09b46da
    source_path: concepts/oauth.md
    workflow: 16
---

OpenClaw, bunu sunan sağlayıcılar için OAuth'u ("abonelik kimlik doğrulaması") destekler;
özellikle **OpenAI Codex (ChatGPT OAuth)** ve **Anthropic Claude CLI yeniden kullanımı**.
Anthropic için pratik ayrım şöyledir:

- **Anthropic API anahtarı**: normal Anthropic API faturalandırması.
- **OpenClaw içinde Anthropic Claude CLI / abonelik kimlik doğrulaması**: Anthropic çalışanları
  bu kullanıma yeniden izin verildiğini bize bildirdi; bu nedenle Anthropic
  yeni bir politika yayımlamadığı sürece OpenClaw, Claude CLI yeniden kullanımını ve
  `claude -p` kullanımını bu entegrasyon için onaylanmış kabul eder. Üretimde Anthropic
  kullanımı için API anahtarıyla kimlik doğrulama hâlâ önerilen daha güvenli
  yoldur.

OpenClaw, hem OpenAI API anahtarıyla kimlik doğrulamayı hem de ChatGPT/Codex OAuth'u
standart sağlayıcı kimliği `openai` altında saklar. Eski `openai-codex:*` profil kimlikleri ve
`auth.order.openai-codex` girdileri, `openclaw doctor --fix` tarafından
onarılmış eski durumdur; yeni yapılandırma için `openai:*` profil kimliklerini ve
`auth.order.openai` kullanın.

Bu sayfada şunlar ele alınır:

- OAuth **belirteç değişiminin** nasıl çalıştığı (PKCE)
- belirteçlerin nerede **saklandığı** (ve nedeni)
- **birden fazla hesabın** nasıl yönetileceği (profiller + oturum başına geçersiz kılmalar)

Kendi OAuth veya API anahtarı akışını sağlayan sağlayıcı Plugin'leri
aynı giriş noktası üzerinden çalışır:

```bash
openclaw models auth login --provider <id>
```

## Belirteç havuzu (neden var)

OAuth sağlayıcıları genellikle her oturum açma/yenileme işleminde yeni bir yenileme belirteci oluşturur.
Bazı sağlayıcılar, aynı kullanıcı/uygulama için yeni bir yenileme belirteci
verildiğinde önceki yenileme belirtecini geçersiz kılar. Pratik belirti: OpenClaw _ve_
Claude Code / Codex CLI üzerinden oturum açıldığında bunlardan birinin daha sonra
rastgele oturumu kapatılır.

Bunu azaltmak için OpenClaw, kimlik doğrulama profili deposunu bir **belirteç havuzu** olarak kullanır:

- çalışma zamanı, kimlik bilgilerini her agent için tek bir yerden okur
- birden fazla profil bir arada bulunabilir ve belirlenimci biçimde yönlendirilebilir
- harici CLI yeniden kullanımı sağlayıcıya özeldir: OpenClaw bir sağlayıcının yerel OAuth
  profilinin sahibi olduktan sonra yerel yenileme belirteci standart kabul edilir. Bu yerel
  yenileme belirteci reddedilirse OpenClaw, harici CLI belirteç
  malzemesine geri dönmek yerine profilin yeniden kimlik doğrulaması gerektirdiğini bildirir.
  Codex CLI önyüklemesi daha da sınırlıdır: yalnızca OpenClaw o
  sağlayıcının OAuth'una sahip olmadan önce boş bir `openai:default` tarzı
  profili başlangıç verileriyle doldurabilir; bundan sonra OpenClaw'un gerçekleştirdiği yenilemeler standart kalır
- durum/başlangıç yolları, harici CLI keşfini önceden
  yapılandırılmış sağlayıcı kümesiyle sınırlar; böylece tek sağlayıcılı bir
  kurulumda ilgisiz bir CLI oturum açma deposu yoklanmaz

## Depolama (belirteçlerin bulunduğu yer)

Gizli bilgiler, `auth-profiles.json` mantıksal adıyla anahtarlanmış şekilde her agent için ayrı tutulur
(alttaki depo agent'ın SQLite veritabanıdır; JSON adı uyumluluk
ve araçlarda gösterim amacıyla korunur):

- Kimlik doğrulama profilleri (OAuth + API anahtarları + isteğe bağlı değer düzeyi referansları):
  `~/.openclaw/agents/<agentId>/agent/auth-profiles.json`
- Eski uyumluluk dosyası: `~/.openclaw/agents/<agentId>/agent/auth.json`
  (statik `api_key` girdileri keşfedildiğinde temizlenir)

Yalnızca eski verileri içe aktarma dosyası (hâlâ desteklenir ancak ana depo değildir):

- `~/.openclaw/credentials/oauth.json` (ilk kullanımda kimlik doğrulama profili deposuna aktarılır)

Yukarıdakilerin tümü ayrıca `$OPENCLAW_STATE_DIR` değerine (durum dizini geçersiz kılması) uyar. Tam başvuru: [/gateway/configuration-reference#auth-storage](/tr/gateway/configuration-reference#auth-storage)

Statik gizli bilgi referansları ve çalışma zamanı anlık görüntüsü etkinleştirme davranışı için [Gizli Bilgi Yönetimi](/tr/gateway/secrets) bölümüne bakın.

İkincil bir agent'ın yerel kimlik doğrulama profili yoksa OpenClaw, varsayılan/ana
agent deposundan geçişli okuma yoluyla devralmayı kullanır; okuma sırasında ana
agent'ın deposunu klonlamaz. OAuth yenileme belirteçleri özellikle hassastır: bazı
sağlayıcılar kullanımdan sonra yenileme belirteçlerini döndürdüğü veya geçersiz kıldığı için normal
kopyalama akışları bunları varsayılan olarak atlar. Bağımsız bir hesaba ihtiyaç
duyan agent için ayrı bir OAuth oturum açma işlemi yapılandırın.

## Anthropic Claude CLI yeniden kullanımı

OpenClaw, Anthropic Claude CLI yeniden kullanımını ve `claude -p` yolunu onaylanmış bir
kimlik doğrulama yolu olarak destekler. Ana makinede zaten yerel bir Claude oturumunuz varsa
ilk katılım/yapılandırma işlemi bunu doğrudan yeniden kullanabilir. Anthropic kurulum belirteci
desteklenen bir belirteçle kimlik doğrulama yolu olarak kullanılmaya devam eder; ancak OpenClaw,
mevcut olduğunda Claude CLI yeniden kullanımını tercih eder.

<Warning>
Anthropic'in herkese açık Claude Code belgeleri, doğrudan Claude Code kullanımının
Claude abonelik sınırları içinde kaldığını belirtir ve Anthropic çalışanları,
OpenClaw tarzı Claude CLI kullanımına yeniden izin verildiğini bize bildirdi. Bu nedenle OpenClaw,
Anthropic yeni bir politika yayımlamadığı sürece Claude CLI yeniden kullanımını ve
`claude -p` kullanımını bu entegrasyon için onaylanmış kabul eder.

Anthropic'in güncel doğrudan Claude Code planı belgeleri için [Claude Code'u
Pro veya Max planınızla
kullanma](https://support.claude.com/en/articles/11145838-using-claude-code-with-your-pro-or-max-plan)
ve [Claude Code'u Team veya Enterprise
planınızla kullanma](https://support.anthropic.com/en/articles/11845131-using-claude-code-with-your-team-or-enterprise-plan/)
bölümlerine bakın.

OpenClaw'da abonelik tarzında başka seçenekler istiyorsanız [OpenAI
Codex](/tr/providers/openai), [Qwen Cloud Coding
Plan](/tr/providers/qwen), [MiniMax Coding Plan](/tr/providers/minimax)
ve [Z.AI / GLM Coding Plan](/tr/providers/zai) bölümlerine bakın.
</Warning>

## OAuth değişimi (oturum açma nasıl çalışır)

OpenClaw'un etkileşimli oturum açma akışları `openclaw/plugin-sdk/llm.ts` içinde uygulanır ve sihirbazlara/komutlara bağlanır.

### Anthropic kurulum belirteci

Akışın yapısı:

1. Claude Code bulunan herhangi bir makinede `claude setup-token` komutunu çalıştırarak belirteci oluşturun, ardından OpenClaw'dan Anthropic kurulum belirteci veya belirteç yapıştırma işlemini başlatın
2. OpenClaw, elde edilen Anthropic kimlik bilgisini bir kimlik doğrulama profilinde saklar
3. model seçimi `anthropic/...` üzerinde kalır
4. mevcut Anthropic kimlik doğrulama profilleri geri alma/sıralama denetimi için kullanılabilir kalır

### OpenAI Codex (ChatGPT OAuth)

OpenAI Codex OAuth, OpenClaw iş akışları dâhil olmak üzere Codex CLI dışında kullanım için açıkça desteklenir.

Oturum açma komutu standart OpenAI sağlayıcı kimliğini kullanır:

```bash
openclaw models auth login --provider openai
```

Tek bir agent içinde birden fazla ChatGPT/Codex OAuth hesabı için
`--profile-id openai:<name>` kullanın. Yeni profillerde `openai-codex:<name>` kullanmayın.
Doctor, bu eski ön eki çakışmasız bir `openai:*` profil kimliğine taşır;
profil kimliklerini `auth.order` veya `/model ...@<profileId>` içine kopyalamadan önce
onarımdan sonra `openclaw models auth list --provider openai` komutunu çalıştırın.

Akışın yapısı (PKCE):

1. bir PKCE doğrulayıcısı/sınaması ve rastgele bir `state` oluşturun
2. `https://auth.openai.com/oauth/authorize?...` adresini açın (kapsam:
   `openid profile email offline_access`)
3. `http://localhost:1455/auth/callback` üzerindeki geri çağrıyı yakalamayı deneyin
   (geri çağrı ana makinesi varsayılan olarak `localhost` değerini kullanır ve yalnızca geri döngü ana makinelerini kabul eder;
   `OPENCLAW_OAUTH_CALLBACK_HOST` ile geçersiz kılın)
4. geri çağrı ulaşmadan önce bir kod yapıştırabiliyorsanız (veya
   uzak/başsız bir ortamdaysanız ve geri çağrı bağlanamıyorsa), bunun yerine yönlendirme URL'sini/kodunu
   yapıştırın - elle yapıştırma işlemi tarayıcı geri çağrısıyla yarışır ve ilk
   tamamlanan kazanır
5. kodu `https://auth.openai.com/oauth/token` adresinde değiştirin
6. erişim belirtecinden `accountId` değerini çıkarın ve `{ access, refresh, expires, accountId }` bilgisini saklayın

Sihirbaz yolu `openclaw onboard` → kimlik doğrulama seçeneği `openai` şeklindedir.

## Yenileme + süre sonu

Profiller bir `expires` zaman damgası saklar. Çalışma zamanında:

- `expires` gelecekteyse saklanan erişim belirtecini kullanın
- süresi dolmuşsa yenileyin (bir dosya kilidi altında) ve saklanan kimlik bilgilerinin üzerine yazın
- ikincil bir agent devralınmış bir ana-agent OAuth profilini okursa
  yenileme işlemi, yenileme belirtecini ikincil agent deposuna kopyalamak yerine
  ana agent deposuna geri yazar
- harici olarak yönetilen CLI kimlik bilgileri (Claude CLI, sınırlı Codex CLI önyüklemesi;
  bkz. [Belirteç havuzu](#the-token-sink-why-it-exists)), kopyalanmış bir yenileme belirtecini
  harcamak yerine yeniden okunur. Yönetilen bir yenileme başarısız olursa OpenClaw,
  harici CLI belirteç malzemesini döndürmek yerine etkilenen profilin
  yeniden kimlik doğrulaması gerektiğini bildirir.

Yenileme akışı otomatiktir; genellikle belirteçleri elle yönetmeniz gerekmez.

## Birden fazla hesap (profiller) + yönlendirme

İki yöntem vardır:

### 1) Tercih edilen: ayrı agent'lar

"Kişisel" ve "iş" hesaplarının hiçbir zaman etkileşime girmemesini istiyorsanız yalıtılmış agent'lar (ayrı oturumlar + kimlik bilgileri + çalışma alanı) kullanın:

```bash
openclaw agents add work
openclaw agents add personal
```

Ardından kimlik doğrulamayı her agent için ayrı ayrı yapılandırın (sihirbaz) ve sohbetleri doğru agent'a yönlendirin.

### 2) Gelişmiş: tek bir agent içinde birden fazla profil

Kimlik doğrulama profili deposu, aynı sağlayıcı için birden fazla profil kimliğini destekler.
Hangisinin kullanılacağını seçin:

- yapılandırma sıralaması aracılığıyla genel olarak (`auth.order`)
- `/model ...@<profileId>` aracılığıyla oturum başına

Örnek (oturum geçersiz kılması):

- `/model Opus@anthropic:work`

Mevcut profil kimliklerini şu komutla listeleyin:

```bash
openclaw models auth list --provider <id>
```

İlgili belgeler:

- [Model yük devretme](/tr/concepts/model-failover) (döndürme + bekleme süresi kuralları)
- [Eğik çizgi komutları](/tr/tools/slash-commands) (komut yüzeyi)

## İlgili

- [Kimlik doğrulama](/tr/gateway/authentication) - model sağlayıcısı kimlik doğrulamasına genel bakış
- [Gizli bilgiler](/tr/gateway/secrets) - kimlik bilgisi depolama ve SecretRef
- [Yapılandırma Başvurusu](/tr/gateway/configuration-reference#auth-storage) - kimlik doğrulama yapılandırma anahtarları
