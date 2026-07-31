---
read_when:
    - Kimlik doğrulama profili rotasyonu, bekleme süreleri veya model geri dönüşü davranışını tanılama
    - Kimlik doğrulama profilleri veya modeller için yük devretme kurallarını güncelleme
    - Oturum modeli geçersiz kılmalarının yedek modele geçme yeniden denemeleriyle nasıl etkileşime girdiğini anlama
sidebarTitle: Model failover
summary: OpenClaw kimlik doğrulama profillerini nasıl dönüşümlü kullanır ve modeller arasında nasıl geri dönüş yapar
title: Model yük devretme mekanizması
x-i18n:
    generated_at: "2026-07-26T22:44:11Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 3dfedbc85038eebb5be056a7b3ffa3275b4329a0b0d791e1a2b4701cbaa4b595
    source_path: concepts/model-failover.md
    workflow: 16
---

OpenClaw hataları iki aşamada işler:

1. Mevcut sağlayıcı içinde **kimlik doğrulama profili rotasyonu**.
2. `agents.defaults.model.fallbacks` içindeki sonraki modele **model geri dönüşü**.

## Çalışma zamanı akışı

<Steps>
  <Step title="Oturum durumunu çözümle">
    Etkin oturum modelini ve kimlik doğrulama profili tercihini çözümle.
  </Step>
  <Step title="Aday zincirini oluştur">
    Model aday zincirini mevcut model seçiminden ve bu seçim kaynağının geri dönüş politikasından oluştur. Yapılandırılmış varsayılanlar, cron işi birincil modelleri ve otomatik seçilen geri dönüş modelleri yapılandırılmış geri dönüşleri kullanabilir; açık kullanıcı oturumu seçimleri katıdır.
  </Step>
  <Step title="Mevcut sağlayıcıyı dene">
    Mevcut sağlayıcıyı kimlik doğrulama profili rotasyonu/bekleme süresi kurallarıyla dene.
  </Step>
  <Step title="Yük devretmeye uygun hatalarda ilerle">
    Bu sağlayıcının seçenekleri yük devretmeye uygun bir hatayla tükenirse sonraki model adayına geç.
  </Step>
  <Step title="Mevcut tur için geri dönüşü kullan">
    Kazanan geri dönüş adayını, oturumun seçili sağlayıcısını/modelini değiştirmeden çalıştır.
  </Step>
  <Step title="Güvenli salt aşırı yük tükenmesini yeniden dene">
    Her aday yalnızca sağlayıcılar aşırı yüklü olduğu için başarısız olursa ve henüz hiçbir araç yürütme veya asistan çıktısı başlamadıysa tura özgü zincirin tamamını üstel geri çekilmeyle en fazla 10 kez yeniden dene. Kullanıcının sessizce beklememesi için 30 saniye sonra tek bir durum bildirimi gönder.
  </Step>
  <Step title="Tükenirse FallbackSummaryError oluştur">
    Her aday başarısız olursa deneme başına ayrıntıları ve biliniyorsa en yakın bekleme süresi bitişini içeren bir `FallbackSummaryError` oluştur.
  </Step>
</Steps>

Geri dönüş yürütmesi tura özgüdür. Yanıt çalıştırıcısı yalnızca geri dönüş bildirim durumunu kalıcılaştırır; böylece `/status` ve geçiş bildirimleri seçili modeli yanıt veren modelden ayırt edebilir. Geri dönüşü sonraki turun model seçimi olarak kalıcılaştırmaz.

## Seçim kaynağı politikası

Seçim kaynağı, geri dönüş zincirine izin verilip verilmediğini belirler:

- **Yapılandırılmış varsayılan**: `agents.defaults.model.primary`, `agents.defaults.model.fallbacks` kullanır.
- **Ajan birincil modeli**: Ajanın model nesnesi kendi `fallbacks` değerini içermediği sürece `agents.entries.*.model` katıdır. Katı davranışı açıkça belirtmek için `fallbacks: []`, ajanı model geri dönüşüne dahil etmek için boş olmayan bir liste kullanın.
- **Çalışma zamanı geri dönüşü**: Geri dönüş adayı yalnızca mevcut tur için geçerlidir. Sonraki tur yeniden seçili birincil modelden başlar. OpenClaw önceden depolanmış `modelOverrideSource: "auto"` girdilerini tanımaya, yapılandırılmış kaynaklarını her 5 dakikada bir yoklamaya ve kaynak düzeldiğinde bunları temizlemeye devam eder. `/new`, `/reset` ve `sessions.reset` da bu girdileri temizler.
- **Kullanıcı oturumu geçersiz kılması**: `/model`, model seçici, `session_status(model=...)` ve `sessions.patch`, `modelOverrideSource: "user"` değerini yazar. Bu, kesin bir oturum seçimidir. Seçili sağlayıcı/model yanıt üretmeden önce başarısız olursa OpenClaw, ilgisiz bir yapılandırılmış geri dönüşten yanıt vermek yerine hatayı bildirir.
- **Eski oturum geçersiz kılması**: Eski oturum girdilerinde `modelOverrideSource` olmadan `modelOverride` bulunabilir. OpenClaw bunları kullanıcı geçersiz kılmaları olarak değerlendirir; böylece açık bir eski seçim sessizce geri dönüş davranışına dönüştürülmez.
- **Cron yük modeli**: Bir cron işinin `payload.model` / `--model` değeri, kullanıcı oturumu geçersiz kılması değil işin birincil modelidir. İş `payload.fallbacks` sağlamadığı sürece yapılandırılmış geri dönüşleri kullanır; `payload.fallbacks: []` cron çalışmasını katı hâle getirir.

OpenClaw, bir tur geri dönüşe geçtiğinde görünür bir bildirim; sonraki bir tur seçili birincil modelde başarıya ulaştığında ise başka bir bildirim gönderir. Kalıcı bildirim durumu, ardışık turlarda aynı seçili/etkin çift kullanıldığında bildirimlerin tekrarlanmasını önlerken model seçimi değişmeden kalır.

## Kimlik doğrulama hatası atlama önbelleği

Varsayılan olarak her yeni tur mevcut geri dönüş yeniden deneme davranışını korur: OpenClaw, kısa süre önce `auth` veya `auth_permanent` ile başarısız olan birincil olmayan adaylar dahil olmak üzere yapılandırılmış her geri dönüş adayını yeniden dener.

Tekrarlanan kimlik doğrulama hatalarını engellemek için şunu etkinleştirin:

```bash
OPENCLAW_FALLBACK_SKIP_TTL_MS=60000
```

Etkinleştirildiğinde OpenClaw, kimlik doğrulama sınıfı bir hatadan sonra birincil olmayan geri dönüş adayı için oturum kimliği, sağlayıcı ve modele göre anahtarlanmış, bellek içi ve oturum kapsamlı bir atlama işaretçisi kaydeder. Birincil adaylar hiçbir zaman atlanmaz; dolayısıyla açık bir kullanıcı model seçimi gerçek kimlik doğrulama hatasını yine gösterir. Önbellek işleme özgüdür ve Gateway yeniden başlatıldığında temizlenir.

Değer, milisaniye cinsinden bir TTL'dir. `0` veya ayarlanmamış olması önbelleği devre dışı bırakır. Pozitif değerler 1 saniye ile 10 dakika arasında sınırlandırılır.

## Kullanıcının görebildiği geri dönüş bildirimleri

Bir oturum otomatik seçilen bir geri dönüşe geçtiğinde OpenClaw aynı yanıt yüzeyinde bir durum bildirimi gönderir:

```text
↪️ Model Geri Dönüşü: <fallback> (seçili <primary>; <reason>)
```

Sonraki bir yoklama başarılı olduğunda ve oturum seçili birincil modele döndüğünde OpenClaw şunu gönderir:

```text
↪️ Model Geri Dönüşü temizlendi: <primary> (önceki <fallback>)
```

Bu bildirimler asistan içeriği değil, operasyonel mesajlardır. Uygun olduğunda yalnızca yan etkili turlar dahil olmak üzere durum değişikliği başına bir kez iletilirler; ancak tura özgü geri dönüş geçişlerinin tekrarlanması bildirimleri yinelemez. İletim, normal kaynak yanıtı engellemesini atlar, ileti dizili kanallarda ilk asistan yanıtı yuvasını tüketmez ve metinden konuşmaya dönüştürme ile taahhüt çıkarımının dışında tutulur.

## Kimlik doğrulama depolaması (anahtarlar + OAuth)

OpenClaw hem API anahtarları hem de OAuth belirteçleri için **kimlik doğrulama profilleri** kullanır.

- Gizli bilgiler ve çalışma zamanı kimlik doğrulama yönlendirme durumu `~/.openclaw/agents/<agentId>/agent/openclaw-agent.sqlite` içinde bulunur.
- Yapılandırmadaki `auth.profiles` / `auth.order` **yalnızca meta veri + yönlendirme** içerir (gizli bilgi içermez).
- Yalnızca içe aktarmaya yönelik eski OAuth dosyası: `~/.openclaw/credentials/oauth.json` (ilk kullanımda ajan başına kimlik doğrulama deposuna aktarılır).
- Eski `auth-profiles.json`, `auth-state.json` ve ajan başına `auth.json` dosyaları `openclaw doctor --fix` tarafından içe aktarılır.

Daha fazla ayrıntı: [OAuth](/tr/concepts/oauth)

Kimlik bilgisi türleri:

- `type: "api_key"` → `{ provider, key }`
- `type: "oauth"` → `{ provider, access, refresh, expires, email? }` (bazı sağlayıcılar için + `projectId`/`enterpriseUrl`)
- `type: "token"` → isteğe bağlı olarak süresi dolabilen, statik taşıyıcı tarzı belirteç; OpenClaw bunu yenilemez (`aws-sdk` ve diğer kimlik bilgisi zinciri kimlik doğrulama modları için kullanılır)

## Profil kimlikleri

OAuth oturum açmaları, birden fazla hesabın birlikte bulunabilmesi için ayrı profiller oluşturur.

- Varsayılan: E-posta mevcut değilse `provider:default`.
- E-posta ile OAuth: `provider:<email>` (örneğin `google-antigravity:user@gmail.com`).

Profiller, ajan başına `openclaw-agent.sqlite` kimlik doğrulama profili deposunda bulunur.

## Rotasyon sırası

Bir sağlayıcının birden fazla profili olduğunda OpenClaw şu şekilde bir sıra seçer:

<Steps>
  <Step title="Açık yapılandırma">
    `auth.order[provider]` (ayarlanmışsa).
  </Step>
  <Step title="Yapılandırılmış profiller">
    Sağlayıcıya göre filtrelenmiş `auth.profiles`.
  </Step>
  <Step title="Depolanan profiller">
    Sağlayıcı için ajan başına SQLite kimlik doğrulama profili girdileri.
  </Step>
</Steps>

Açık bir sıra yapılandırılmamışsa OpenClaw dönüşümlü bir sıra kullanır:

- **Birincil anahtar:** profil türü (**önce OAuth, ardından statik belirteç, ardından API anahtarı**).
- **OAuth için ikincil anahtar:** erişim belirteci şu anda kullanılabilir olan profiller,
  erişim belirtecinin süresi dolmuş profillerden önce gelir. Süresi dolmuş OAuth profilleri,
  kullanılabilir eş profil olmadığında çalışma zamanının bunları yenileyebilmesi için uygun kalır.
- **Sonraki anahtar:** `usageStats.lastUsed` (her tür/durum katmanında en eski önce).
- **Bekleme süresindeki/devre dışı profiller**, en yakın sona erme zamanına göre sıralanarak sona taşınır.

### Oturum bağlılığı (önbellek dostu)

OpenClaw, sağlayıcı önbelleklerini sıcak tutmak için **seçilen kimlik doğrulama profilini oturum başına sabitler**. Her istekte rotasyon yapmaz. Sabitlenen profil şu durumlardan birine kadar yeniden kullanılır:

- oturum sıfırlanır (`/new` / `/reset`)
- bir Compaction tamamlanır (Compaction sayısı artar)
- profil bekleme süresindedir/devre dışıdır

`/model …@<profileId>` aracılığıyla yapılan manuel seçim, ilgili oturum için bir **kullanıcı geçersiz kılması** belirler ve yeni bir oturum başlayana kadar otomatik olarak döndürülmez.

<Note>
Otomatik sabitlenen profiller (oturum yönlendiricisi tarafından seçilenler) bir **tercih** olarak değerlendirilir: Önce bunlar denenir, ancak OpenClaw hız sınırlarında/zaman aşımlarında başka bir profile geçebilir. Özgün profil yeniden kullanılabilir olduğunda yeni çalıştırmalar, seçili modeli veya çalışma zamanını değiştirmeden onu yeniden tercih edebilir. Kullanıcı tarafından sabitlenen profiller ilgili profile kilitli kalır; profil başarısız olursa ve model geri dönüşleri yapılandırılmışsa OpenClaw profil değiştirmek yerine sonraki modele geçer.
</Note>

### OpenAI Codex aboneliği ve API anahtarı yedeği

OpenAI ajan modellerinde kimlik doğrulama ile çalışma zamanı ayrıdır. Kimlik doğrulama bir Codex abonelik profili ile OpenAI API anahtarı yedeği arasında dönebilirken `openai/gpt-*` Codex çalıştırma altyapısında kalır.

Kullanıcıya gösterilen sıra için `auth.order.openai` kullanın:

```json5
{
  auth: {
    order: {
      openai: ["openai:user@example.com", "openai:api-key-backup"],
    },
  },
}
```

Hem ChatGPT/Codex OAuth profilleri hem de OpenAI API anahtarı profilleri için `openai:*` kullanın. Abonelik bir Codex kullanım sınırına ulaştığında OpenClaw, Codex sağlıyorsa kesin sıfırlanma zamanını kaydeder, sıradaki kimlik doğrulama profilini dener ve çalıştırmayı Codex çalıştırma altyapısı içinde tutar. Sıfırlanma zamanı geçtikten sonra abonelik profili yeniden uygun hâle gelir ve sonraki otomatik seçim ona dönebilir.

Yalnızca ilgili oturum için tek bir hesabı/anahtarı zorunlu kılmak istediğinizde kullanıcı tarafından sabitlenen bir profil kullanın. Kullanıcı tarafından sabitlenen profiller kasıtlı olarak katıdır ve sessizce başka bir profile geçmez.

## Bekleme süreleri

Bir profil kimlik doğrulama/hız sınırı hataları (veya hız sınırlamasına benzeyen bir zaman aşımı) nedeniyle başarısız olduğunda OpenClaw profili bekleme süresine alır ve sonraki profile geçer.

<AccordionGroup>
  <Accordion title="Hız sınırı / zaman aşımı grubuna neler girer">
    Bu hız sınırı grubu yalnızca `429` değerinden daha geniştir: `Too many concurrent requests`, `ThrottlingException`, `concurrency limit reached`, `workers_ai ... quota limit exceeded`, `throttled`, `resource exhausted` gibi sağlayıcı mesajlarının yanı sıra `weekly limit reached` veya `monthly limit exhausted` gibi dönemsel kullanım penceresi sınırlarını da içerir.

    Biçim/geçersiz istek hataları genellikle sonlandırıcıdır; çünkü aynı yükü yeniden denemek aynı şekilde başarısız olur. Bu nedenle OpenClaw, kimlik doğrulama profillerini döndürmek yerine bu hataları gösterir. Bilinen yeniden deneme-onarım yolları bunu açıkça etkinleştirebilir: Örneğin Cloud Code Assist araç çağrısı kimliği doğrulama hataları temizlenir ve `allowFormatRetry` politikası üzerinden bir kez yeniden denenir.

    `Unhandled stop reason: error`, `stop reason: error`, `reason: error` ve `Provider finish_reason: error` gibi OpenAI uyumlu **sağlayıcı tarafından tamamlanmış** durdurma/bitiş nedenleri, zaman aşımı olarak değil **`server_error`** (HTTP benzeri durum 500) olarak sınıflandırılır. Model/profil rotasyonu için yük devretmeye uygun kalırlar; ancak tanılama, kullanıcı metnini "LLM isteği zaman aşımına uğradı." şeklinde yeniden yazmak yerine sağlayıcının bitiş nedeni metnini korur. `Provider finish_reason: abort`, `network_error` ve `malformed_response` gibi aktarım biçimli bitiş nedenleri zaman aşımı/yük devretme grubunda kalır (durum 408).

    Kaynak bilinen geçici bir kalıpla eşleştiğinde genel sunucu metni de bu zaman aşımı grubuna girebilir. Örneğin çıplak model çalışma zamanı akış sarmalayıcı mesajı `An unknown error occurred`, her sağlayıcı için yük devretmeye uygun kabul edilir; çünkü paylaşılan model çalışma zamanı, sağlayıcı akışları belirli ayrıntılar olmadan `stopReason: "aborted"` veya `stopReason: "error"` ile sona erdiğinde bunu yayar. `internal server error`, `unknown error, 520`, `upstream error` veya `backend error` gibi geçici sunucu metinleri içeren JSON `api_error` yükleri de yük devretmeye uygun zaman aşımları olarak değerlendirilir.

    OpenRouter'a özgü, yalın `Provider returned error` gibi genel yukarı akış metinleri yalnızca sağlayıcı bağlamı gerçekten OpenRouter olduğunda zaman aşımı olarak değerlendirilir. `LLM request failed with an unknown error.` gibi genel dahili geri dönüş metinleri ihtiyatlı kalır ve tek başına yük devretmeyi tetiklemez.

  </Accordion>
  <Accordion title="SDK yeniden deneme sonrası üst sınırları">
    Bazı sağlayıcı SDK'ları, denetimi OpenClaw'a geri vermeden önce aksi hâlde uzun bir `Retry-After` aralığı boyunca bekleyebilir. Anthropic ve OpenAI gibi Stainless tabanlı SDK'larda OpenClaw, SDK içindeki `retry-after-ms` / `retry-after` beklemelerini varsayılan olarak 60 saniyeyle sınırlar ve daha uzun, yeniden denenebilir yanıtları hemen ileterek bu yük devretme yolunun çalışmasını sağlar. Üst sınırı `OPENCLAW_SDK_RETRY_MAX_WAIT_SECONDS` ile ayarlayın veya devre dışı bırakın; bkz. [Yeniden deneme davranışı](/tr/concepts/retry).
  </Accordion>
  <Accordion title="Model kapsamlı bekleme süreleri">
    Hız sınırı bekleme süreleri model kapsamında da olabilir:

    - OpenClaw, başarısız olan model kimliği bilindiğinde hız sınırı hataları için `cooldownModel` kaydeder.
    - Bekleme süresi farklı bir modelle sınırlıysa aynı sağlayıcıdaki kardeş bir model yine de denenebilir.
    - Faturalandırma/devre dışı bırakma aralıkları modeller genelinde profilin tamamını engellemeye devam eder.

  </Accordion>
</AccordionGroup>

Normal (faturalandırma dışı, kalıcı kimlik doğrulama dışı) bekleme süreleri, profilin son hata sayısına göre ölçeklenir:

- 1\. hata: 30 saniye
- 2\. hata: 1 dakika
- 3\. ve sonraki hatalar: 5 dakika (üst sınır)

Profilin yerleşik hata aralığı geçtikten sonra sayaçlar sıfırlanır.

Durum, ajan başına SQLite kimlik doğrulama durumunda `usageStats` altında saklanır:

```json
{
  "usageStats": {
    "provider:profile": {
      "lastUsed": 1736160000000,
      "cooldownUntil": 1736160600000,
      "errorCount": 2
    }
  }
}
```

## Faturalandırma nedeniyle devre dışı bırakmalar

Faturalandırma/kredi hataları (örneğin "yetersiz kredi" / "kredi bakiyesi çok düşük") yük devretmeyi gerektiren durumlar olarak değerlendirilir, ancak genellikle geçici değildir. OpenClaw, kısa bir bekleme süresi yerine profili **devre dışı** olarak işaretler (daha uzun bir geri çekilme süresiyle) ve sonraki profile/sağlayıcıya geçer.

<Note>
Faturalandırmayı andıran her yanıt `402` değildir ve her HTTP `402` burada değerlendirilmez. Bir sağlayıcı bunun yerine `401` veya `403` döndürse bile OpenClaw, açık faturalandırma metnini faturalandırma yolunda tutar; ancak sağlayıcıya özgü eşleştiriciler bunlara sahip olan sağlayıcıyla sınırlı kalır (örneğin OpenRouter `403 Key limit exceeded`).

Bu arada geçici `402` kullanım aralığı ve kuruluş/çalışma alanı harcama sınırı hataları, ileti yeniden denenebilir görünüyorsa (örneğin `weekly usage limit exhausted`, `daily limit reached, resets tomorrow` veya `organization spending limit exceeded`) `rate_limit` olarak sınıflandırılır. Bunlar, uzun faturalandırma nedeniyle devre dışı bırakma yolu yerine kısa bekleme/yük devretme yolunda kalır.
</Note>

Yüksek güvenilirlikte kalıcı kimlik doğrulama hataları (iptal edilmiş/devre dışı bırakılmış anahtarlar, devre dışı bırakılmış çalışma alanları) benzer bir devre dışı bırakma yoluna girer; ancak bazı sağlayıcılar olaylar sırasında kimlik doğrulama hatasına benzeyen yükleri geçici olarak sunduğundan, faturalandırmaya kıyasla çok daha kısa sürede düzelir.

Durum, ajan başına SQLite kimlik doğrulama durumunda saklanır:

```json
{
  "usageStats": {
    "provider:profile": {
      "disabledUntil": 1736178000000,
      "disabledReason": "billing"
    }
  }
}
```

Aşırı yük ve hız sınırı hataları, faturalandırma bekleme sürelerinden daha agresif biçimde işlenir: OpenClaw varsayılan olarak aynı sağlayıcıda bir kimlik doğrulama profili yeniden denemesine izin verir, ardından beklemeden yapılandırılmış sonraki model geri dönüşüne geçer.

## Model geri dönüşü

Bir sağlayıcının tüm profilleri başarısız olursa OpenClaw, `agents.defaults.model.fallbacks` içindeki sonraki modele geçer. Bu; profil rotasyonunu tüketen kimlik doğrulama hataları, hız sınırları ve zaman aşımları için geçerlidir (diğer hatalar geri dönüşü ilerletmez). Yeterli ayrıntı sunmayan sağlayıcı hataları da geri dönüş durumunda hassas biçimde etiketlenir: `empty_response`, sağlayıcının kullanılabilir bir ileti veya durum döndürmediği anlamına gelir; `no_error_details`, sağlayıcının açıkça `Unknown error (no error details in response)` döndürdüğü anlamına gelir; `unclassified` ise OpenClaw'ın ham önizlemeyi koruduğu ancak henüz hiçbir sınıflandırıcının bununla eşleşmediği anlamına gelir.

`ModelNotReadyException` gibi sağlayıcının meşgul olduğunu belirten sinyaller aşırı yük grubuna girer ve hız sınırlarıyla aynı, bir rotasyonun ardından geri dönüş politikasını izler (yukarıdaki varsayılanlar tablosuna bakın).

Tüm aday zinciri yalnızca aşırı yük hataları nedeniyle tükenirse yanıt çalıştırıcısı, aynı turda zinciri en fazla 10 kez yeniden dener. Tam tur yeniden denemesine yalnızca araç yürütülmesi veya asistan çıktısı başlamadan önce izin verilir; böylece gözlemlenebilir bir işlemden sonra aşırı yük oluşursa yinelenen değişiklikler veya iletiler önlenir. Geri çekilme 2.5 saniyeden başlar ve iki katına çıkarak 30 saniyelik üst sınıra ulaşır. Tur 30 saniyedir bekliyorsa OpenClaw bir kez geçici durum bildirimi gönderir: `The AI service is temporarily overloaded. I’m still retrying; this may take a few minutes.` Yeniden deneme ve varsa kazanan geri dönüş yalnızca o tura özgü kalır; normal geçici sunucu hataları ayrı tek yeniden deneme politikalarını korur.

Bir çalıştırma yapılandırılmış varsayılan birincilden, bir cron işi birincilinden, açık geri dönüşleri olan bir ajan birincilinden veya otomatik seçilmiş bir geri dönüş geçersiz kılmasından başladığında OpenClaw, eşleşen yapılandırılmış geri dönüş zincirinde ilerleyebilir. Açık geri dönüşleri olmayan ajan birincilleri ve açık kullanıcı seçimleri (örneğin `/model ollama/qwen3.5:27b`, model seçici, `sessions.patch` veya tek seferlik CLI sağlayıcı/model geçersiz kılmaları) katıdır: söz konusu sağlayıcıya/modele ulaşılamazsa veya yanıt üretilmeden önce başarısız olursa OpenClaw, ilgisiz bir geri dönüşten yanıt vermek yerine hatayı bildirir.

### Aday zinciri kuralları

OpenClaw, aday listesini hâlihazırda istenen `provider/model` ile yapılandırılmış geri dönüşlerden oluşturur.

<AccordionGroup>
  <Accordion title="Kurallar">
    - İstenen model her zaman ilk sıradadır.
    - Açıkça yapılandırılmış geri dönüşlerde yinelenenler kaldırılır ancak model izin listesine göre filtreleme yapılmaz. Bunlar açık operatör niyeti olarak değerlendirilir.
    - Geçerli çalıştırma aynı sağlayıcı ailesindeki yapılandırılmış bir geri dönüşü zaten kullanıyorsa OpenClaw, yapılandırılmış zincirin tamamını kullanmayı sürdürür.
    - Açık bir geri dönüş geçersiz kılması sağlanmadığında, istenen model farklı bir sağlayıcı kullansa bile yapılandırılmış geri dönüşler yapılandırılmış birincilden önce denenir.
    - Geri dönüş çalıştırıcısına açık bir geri dönüş geçersiz kılması sağlanmadığında yapılandırılmış birincil sona eklenir; böylece önceki adaylar tükendiğinde zincir normal varsayılana geri dönebilir.
    - Bir çağıran `fallbacksOverride` sağladığında çalıştırıcı yalnızca istenen modeli ve bu geçersiz kılma listesini kullanır. Boş bir liste model geri dönüşünü devre dışı bırakır ve yapılandırılmış birincilin gizli bir yeniden deneme hedefi olarak eklenmesini önler.

  </Accordion>
</AccordionGroup>

### Hangi hatalar geri dönüşü ilerletir?

<Tabs>
  <Tab title="Şu durumlarda devam eder">
    - kimlik doğrulama hataları
    - hız sınırları ve bekleme süresinin tükenmesi
    - aşırı yük/sağlayıcının meşgul olması hataları
    - zaman aşımı biçimindeki yük devretme hataları
    - faturalandırma nedeniyle devre dışı bırakmalar
    - `LiveSessionModelSwitchError`; eski, kalıcı bir modelin dış yeniden deneme döngüsü oluşturmaması için bir yük devretme yoluna normalleştirilir
    - hâlâ kalan adaylar varken tanınmayan diğer hatalar

  </Tab>
  <Tab title="Şu durumlarda devam etmez">
    - zaman aşımı/yük devretme biçiminde olmayan açık iptaller
    - Compaction/yeniden deneme mantığı içinde kalması gereken bağlam taşması hataları (örneğin `request_too_large`, `input token count exceeds the maximum number of input tokens`, `input exceeds the maximum number of tokens`, `input too long for the model` veya `ollama error: context length exceeded`)
    - hiç aday kalmadığında oluşan son bilinmeyen hata
    - Claude Fable 5 güvenlik retleri; doğrudan API anahtarı istekleri bunları bunun yerine sağlayıcı düzeyinde, Anthropic'in sunucu tarafında `claude-opus-4-8` modeline geri dönüşü aracılığıyla işler (bkz. [Anthropic](/tr/providers/anthropic#safety-refusal-fallback-claude-fable-5))

  </Tab>
</Tabs>

### Bekleme süresinde atlama ve yoklama davranışı

Bir sağlayıcının tüm kimlik doğrulama profilleri zaten bekleme süresindeyken OpenClaw, bu sağlayıcıyı otomatik olarak sonsuza kadar atlamaz. Her aday için ayrı karar verir:

<AccordionGroup>
  <Accordion title="Aday başına kararlar">
    - Kalıcı kimlik doğrulama hataları sağlayıcının tamamının hemen atlanmasına neden olur.
    - Faturalandırma nedeniyle devre dışı bırakmalar genellikle atlanır; ancak yeniden başlatma olmadan kurtarma sağlanabilmesi için birincil aday belirli aralıklarla yine de yoklanabilir.
    - Birincil aday, sağlayıcı başına sınırlamayla bekleme süresinin bitimine yakın yoklanabilir.
    - Hata geçici görünüyorsa (`rate_limit`, `overloaded` veya bilinmeyen), aynı sağlayıcıdaki geri dönüş kardeşleri bekleme süresine rağmen denenebilir. Bu, özellikle hız sınırı model kapsamında olduğunda ve kardeş bir model hemen toparlanabildiğinde önemlidir.
    - Tek bir sağlayıcının sağlayıcılar arası geri dönüşü geciktirmemesi için geçici bekleme yoklamaları, her geri dönüş çalıştırmasında sağlayıcı başına bir kezle sınırlandırılır.

  </Accordion>
</AccordionGroup>

## Oturum geçersiz kılmaları ve canlı model değiştirme

Oturum modeli değişiklikleri paylaşılan durumdur. Etkin çalıştırıcı, `/model` komutu, Compaction/oturum güncellemeleri ve canlı oturum uzlaştırması aynı oturum girdisinin bölümlerini okur veya yazar. Geri dönüş yürütmesi model seçimi alanlarına yazmaz; dolayısıyla yeniden denerken daha yeni bir manuel seçimin yerini alamaz.

Canlı model değiştirme şu kuralları izler:

- Yalnızca kullanıcı tarafından açıkça yapılan model değişiklikleri bekleyen bir canlı geçişi işaretler. Buna `/model`, `session_status(model=...)` ve `sessions.patch` dahildir.
- Geri dönüş rotasyonu, Heartbeat geçersiz kılmaları veya Compaction gibi sistem tarafından yapılan model değişiklikleri kendi başına hiçbir zaman bekleyen bir canlı geçişi işaretlemez.
- Kullanıcı tarafından yapılan model geçersiz kılmaları, geri dönüş politikası açısından tam seçim olarak değerlendirilir; bu nedenle ulaşılamayan seçili bir sağlayıcı, `agents.defaults.model.fallbacks` tarafından maskelenmek yerine hata olarak gösterilir.
- Çalışma zamanı geri dönüş adayları yalnızca o tura özgü kalır. Sonraki tur, önceki çalıştırma sırasında yapılan manuel seçim de dâhil olmak üzere geçerli seçili modelden başlar.
- Daha önce saklanan otomatik geri dönüş geçersiz kılmaları desteklenmeye devam eder: OpenClaw, bunların yapılandırılmış kaynağını düzenli aralıklarla yoklar ve kaynak düzeldiğinde geçersiz kılmayı temizler; `/new`, `/reset` ve `sessions.reset`, otomatik kaynaklı geçersiz kılmaları hemen temizler.
- Kullanıcı yanıtları, geri dönüş geçişlerini ve geri dönüş temizlendikten sonraki düzelmeyi her durum değişikliğinde bir kez bildirir. Aynı seçili/etkin çifte sahip yinelenen turlarda bildirim tekrarlanmaz.
- `/status`, seçili modeli ve geri dönüş durumu farklıysa etkin geri dönüş modelini ve nedenini gösterir.
- Canlı oturum uzlaştırması, eski çalışma zamanı modeli alanları yerine kalıcı oturum geçersiz kılmalarını tercih eder.
- Bir canlı geçiş hatası etkin geri dönüş zincirindeki daha sonraki bir adayı gösteriyorsa OpenClaw, önce ilgisiz adaylarda ilerlemek yerine doğrudan bu seçili modele atlar.

Etkin çalıştırma, seçtiği adayı doğrudan taşır. Canlı uzlaştırma bu adayı yalnızca bekleyen açık bir kullanıcı geçişi için değiştirir; dolayısıyla geçici bir geri dönüş geçersiz kılmasına veya geri almaya gerek yoktur.

## Gözlemlenebilirlik ve hata özetleri

`runWithModelFallback(...)`, günlükleri ve kullanıcıya yönelik bekleme süresi iletilerini besleyen deneme başına ayrıntıları kaydeder:

- denenen sağlayıcı/model
- neden (`rate_limit`, `overloaded`, `billing`, `auth`, `model_not_found` ve benzer yük devretme nedenleri)
- isteğe bağlı durum/kod
- insan tarafından okunabilir hata özeti

Yapılandırılmış `model_fallback_decision` günlükleri ayrıca bir aday başarısız olduğunda, atlandığında veya daha sonraki bir geri dönüş başarılı olduğunda düz `fallbackStep*` alanlarını içerir. Bu alanlar, denenen geçişi açıkça belirtir (`fallbackStepFromModel`, `fallbackStepToModel`, `fallbackStepFromFailureReason`, `fallbackStepFromFailureDetail`, `fallbackStepFinalOutcome`); böylece günlük ve tanılama dışa aktarıcıları, son geri dönüş de başarısız olsa bile birincil hatayı yeniden oluşturabilir.

Her aday başarısız olduğunda OpenClaw, `FallbackSummaryError` hatasını oluşturur. Dış yanıt yürütücüsü bunu, "tüm modeller geçici olarak hız sınırına tabi" gibi daha belirgin bir mesaj oluşturmak ve biliniyorsa en yakın bekleme süresi bitişini eklemek için kullanabilir.

Bu bekleme süresi özeti modele duyarlıdır:

- denenen sağlayıcı/model zinciriyle ilgisiz, model kapsamlı hız sınırları yok sayılır
- kalan engel eşleşen, model kapsamlı bir hız sınırıysa OpenClaw, söz konusu modeli hâlâ engelleyen son eşleşen bitiş zamanını bildirir

## İlgili yapılandırma

Şunlar için [Gateway yapılandırması](/tr/gateway/configuration) sayfasına bakın:

- `auth.profiles` / `auth.order`
- `agents.defaults.model.primary` / `agents.defaults.model.fallbacks`
- `agents.defaults.imageModel` yönlendirmesi

Daha kapsamlı model seçimi ve yedek modele geçiş genel bakışı için [Modeller](/tr/concepts/models) sayfasına bakın.
