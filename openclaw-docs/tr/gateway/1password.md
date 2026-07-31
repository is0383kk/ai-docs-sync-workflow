---
read_when:
    - API anahtarlarını openclaw.json dosyasından çıkarıp 1Password içine taşımak istiyorsunuz
    - Gateway'i başsız modda çalıştırıyorsunuz ve op için hizmet hesabı kimlik doğrulamasına ihtiyacınız var
    - Ajanların op CLI ile gizli bilgileri okumasını veya eklemesini istiyorsunuz
summary: Gateway gizli bilgilerini 1Password CLI ile çözümleyin ve ajanların paketle birlikte gelen 1password skill'ini kullanmasına izin verin
title: 1Password
x-i18n:
    generated_at: "2026-07-26T23:20:07Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 5bb14944f0b3ce1ee3f90bf666a53e8673e7a9861e3e138a5fabe9c8e070cbd7
    source_path: gateway/1password.md
    workflow: 16
---

OpenClaw, **1Password** ile üç bağımsız şekilde eşleşir:

- **Yapılandırma sırları:** `openclaw.json` içindeki herhangi bir [SecretRef](/tr/gateway/secrets) alanı, çalışma zamanında `op` CLI aracılığıyla çözümlenebilir; böylece API anahtarları hiçbir zaman yapılandırma dosyasında bulunmaz.
- **Ajan iş akışları:** paketle birlikte gelen `1password` skill'i, ajanlara kendi görevleri için `op` ile oturum açmayı ve sırları okumayı veya enjekte etmeyi öğretir.
- **Tarayıcıda oturum açma:** `claude-cli` arka ucu, [Claude için 1Password](https://support.1password.com/1password-claude/) ile Claude Code'un Chrome entegrasyonunu kullanabilir; böylece ajan, parola modele veya OpenClaw'a hiç ulaşmadan web sitelerinde oturum açabilir.

## Gereksinimler

- Gateway ana makinesine yüklenmiş [1Password CLI](https://developer.1password.com/docs/cli/get-started/) (`op`) (macOS'te `brew install 1password-cli`).
- `op` için bir kimlik doğrulama modu:
  - **Hizmet hesabı** (başsız Gateway'ler için önerilir): Gateway hizmet ortamında `OP_SERVICE_ACCOUNT_TOKEN` dışa aktarılmalıdır. Masaüstü uygulaması ve etkileşimli oturum açma gerekmez.
  - **Masaüstü uygulaması entegrasyonu**: 1Password uygulaması, CLI entegrasyonu etkinleştirilmiş olarak aynı makinede çalışır. İlk çağrılar Touch ID'yi veya sistem kimlik doğrulamasını tetikleyebilir.
  - **Bağımsız oturum açma**: `op signin` her oturumda istem gösterir. Skill aracılığıyla ajanlar için kullanılabilir ancak başsız bir Gateway'de yapılandırma sırlarının çözümlenmesine uygun değildir.

## Yapılandırma sırlarını op ile çözümleme

Bir `op://vault/item/field` referansıyla `op read` çalıştıran bir exec sır sağlayıcısı tanımlayın, ardından SecretRef destekli herhangi bir alanı buna yönlendirin:

```json5
{
  secrets: {
    providers: {
      onepassword_openai: {
        source: "exec",
        command: "/opt/homebrew/bin/op",
        allowSymlinkCommand: true, // Homebrew sembolik bağlantılı ikili dosyaları için gereklidir
        trustedDirs: ["/opt/homebrew"],
        args: ["read", "op://Personal/OpenClaw QA API Key/password"],
        passEnv: ["HOME"],
        jsonOnly: false,
      },
    },
  },
  models: {
    providers: {
      openai: {
        baseUrl: "https://api.openai.com/v1",
        models: [{ id: "gpt-5", name: "gpt-5" }],
        apiKey: { source: "exec", provider: "onepassword_openai", id: "value" },
      },
    },
  },
}
```

Parçaların birlikte çalışma şekli:

- `command` mutlak bir yol olmalıdır; `trustedDirs` bu yolun dizinini güvenilir olarak işaretler ve Homebrew, `op` öğesini sembolik bağlantı olarak yüklediği için `allowSymlinkCommand` gereklidir.
- `args`, `op://vault/item/field` referansını aynen taşır. OpenClaw, `op://` şemasını kendisi ayrıştırmaz; bunu `op` ikili dosyası çözümler.
- `passEnv`, listelenen değişkenleri Gateway ortamından iletir. Masaüstü uygulaması entegrasyonu için `HOME` gerekir; hizmet hesaplarında ayrıca Gateway hizmet ortamında `OP_SERVICE_ACCOUNT_TOKEN` bulunmalıdır (bunu `passEnv` öğesine ekleyin veya belirtecin yapılandırma dosyasından okunabilmesini kabul ediyorsanız yalnızca `env` aracılığıyla ayarlayın).
- Tek değerli çıktı için `id: "value"` değerini koruyun. `jsonOnly: true` ve bir JSON yükü kullanıldığında alanları bunun yerine bir JSON işaretçisi kimliğiyle adresleyin.
- Her sır için bir sağlayıcı girdisi kullanmak, referansların denetlenebilir kalmasını sağlar; sağlayıcıları tüketicilerine göre adlandırın (`onepassword_openai`, `onepassword_telegram`).

Çözümleme sırası, önbelleğe alma ve hata semantiği için [Gateway sırları](/tr/gateway/secrets) bölümüne; SecretRef kabul eden tüm alanlar için [SecretRef Kimlik Bilgisi Yüzeyi](/tr/reference/secretref-credential-surface) bölümüne bakın.

## Başsız Gateway'ler için hizmet hesabı kurulumu

1. 1Password hesabınızda bir hizmet hesabı oluşturun ve bu hesaba yalnızca Gateway'in ihtiyaç duyduğu kasa öğeleri için okuma erişimi verin.
2. `OP_SERVICE_ACCOUNT_TOKEN` öğesini Gateway hizmetine sağlayın (launchd plist, systemd birimi veya kapsayıcı ortamı).
3. `"OP_SERVICE_ACCOUNT_TOKEN"` öğesini sağlayıcının `passEnv` listesine ekleyin.
4. Gateway ana makinesi ortamından doğrulayın: `op whoami`, istem göstermeden hizmet hesabını yazdırmalıdır.

Hizmet hesabı okumalarında kasanın `op://` referansında açıkça adlandırılması gerekir. Hesabın kapsamını sıkı tutun; bu bir taşıyıcı kimlik bilgisidir.

## Ajanlar için 1password skill'i

OpenClaw, ajanları yetkin `op` operatörlerine dönüştüren bir `1password` skill'i içerir: kullanılabilir kimlik doğrulama modunu (hizmet hesabı, masaüstü uygulaması entegrasyonu veya bağımsız oturum açma) algılar, herhangi bir şeyi okumadan önce `op whoami` ile erişimi doğrular ve sır değerlerini diske yazmak yerine `op run` / `op inject` kullanımını tercih eder. Skill, `op` ikili dosyasını gerektirir ve bu dosya eksikse Homebrew ile yükleme seçeneği sunar.

Ajanlar bunu kendi iş akışlarında, örneğin görevin ortasında bir dağıtım belirteci okumak veya bir komuta ortam değişkenleri enjekte etmek için kullanır. Yapılandırma sırlarının çözümlenmesinden bağımsızdır; Gateway, herhangi bir skill kullanılmadan SecretRef'leri çözümler.

## Claude için 1Password ile tarayıcıda oturum açma

[Claude için 1Password](https://support.1password.com/1password-claude/), Claude'un oturum açma isteğinde bulunmasına olanak tanırken 1Password tarayıcı uzantısı kimlik bilgisini şifreli bir kanal üzerinden doğrudan sayfaya doldurur. Sır hiçbir zaman model bağlamına, transkripte veya OpenClaw'a girmez. OpenClaw, Claude Code'un Chrome entegrasyonu etkinleştirilmiş olarak [`claude-cli` arka ucunu](/tr/gateway/cli-backends#claude-cli-specifics) çalıştırdığında, ajan görevleri gerçek bir oturum açılmış oturum gerektiren web siteleri için bu akışı kullanabilir.

Arka ucun kendisine ek olarak gerekenler:

- Chrome'un, bağlı [Claude in Chrome extension](https://code.claude.com/docs/en/chrome) uzantısının, 1Password masaüstü uygulamasının ve 1Password tarayıcı uzantısının (her ikisi de 8.12.28 veya üzeri) bulunduğu bir macOS Gateway ana makinesi.
- Doğrudan bir Anthropic planında (Pro, Max, Team veya Enterprise) oturum açmış Claude Code. Chrome entegrasyonu Amazon Bedrock, Google Cloud veya diğer üçüncü taraf sağlayıcılar üzerinden kullanılamaz.
- Anthropic tarafındaki tek seferlik 1Password bağlantısı: Claude için 1Password, [1Password kılavuzunda](https://support.1password.com/1password-claude/) açıklanan Claude masaüstü uygulaması veya uzantı akışı üzerinden kurulur ve şu anda bir macOS beta sürümüdür. 1Password Business'ta bir yönetici önce Policies altında "Allow AI agents to autofill for users" seçeneğini etkinleştirmelidir; Anthropic Team/Enterprise planlarında da bir Owner etkinleştirene kadar entegrasyon kapalı olarak sunulur.
- Claude başlatma bağımsız değişkenlerine `--chrome` ekleyen bir [CLI arka uç plugin'i](/tr/plugins/cli-backend-plugins); paketle birlikte gelen arka uç Chrome'u etkinleştirmez.
- Gateway ana makinesinde bir kişi: her kimlik bilgisi kullanımı, orada onaylanan bir 1Password istemi gösterir (örneğin Touch ID ile). Kısıtlayıcı bir exec ilkesi altında, tarayıcı aracı çağrıları da önce OpenClaw onayları olarak kanalınıza iletilir.

Bunu OpenClaw'a bağlamadan önce parçaları Gateway ana makinesindeki etkileşimli bir oturumda doğrulayın: `claude --chrome` komutunu çalıştırın, uzantının bağlandığını onaylayın ve `claude-in-chrome` araçlarının kimlik bilgisi araçlarını içerdiğini kontrol edin. Orada görünmüyorlarsa OpenClaw üzerinden de görünmezler.

Tek kullanımlık parolalar aynı sayfada 1Password tarafından doldurulur; doğrulama kodlarını veya parolaları asla sohbet üzerinden iletmeyin. Onay ve tarayıcı Gateway ana makinesinde bulunduğundan başsız veya uzak Gateway'ler bugün bu akışı kullanamaz.

## Güvenlik notları

- Exec sağlayıcıları aracılığıyla çözümlenen sır değerleri Gateway belleğinde kalır; yapılandırma anlık görüntüleri ve `config.get` yanıtları SecretRef alanlarını maskeler.
- Sır değerlerini asla `openclaw.json`, günlüklere veya sohbete yerleştirmeyin. Öğe adlarını yapılandırmada, değerleri 1Password'de tutun.
- 1Password denetim izi, her hizmet hesabı okumasını göstererek anahtar döndürmeyi ve olay incelemesini uygulanabilir hâle getirir.

## Sorun giderme

- `command not found` veya işlem başlatma hataları: mutlak `op` yolunu kullanın ve dizinini `trustedDirs` içine ekleyin.
- `op` çözümleniyor ancak okumalar sembolik bağlantı hatalarıyla başarısız oluyor: Homebrew yüklemeleri için `allowSymlinkCommand: true` ayarlayın.
- `account is not signed in`: hizmet hesapları için `OP_SERVICE_ACCOUNT_TOKEN` öğesinin Gateway hizmetine ulaştığını ve `passEnv` içinde listelendiğini doğrulayın; masaüstü entegrasyonu için uygulamanın çalıştığını ve kilidinin açık olduğunu doğrulayın.
- İlk okumalar yavaşsa: sağlayıcıdaki `timeoutMs` değerini yükseltin; `op` soğuk başlatmaları, yoğun ana makinelerde katı zaman aşımı sürelerini aşabilir.
