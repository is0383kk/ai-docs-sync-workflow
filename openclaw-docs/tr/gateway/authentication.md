---
read_when:
    - Model kimlik doğrulaması veya OAuth süresinin dolmasıyla ilgili hata ayıklama
    - Kimlik doğrulama veya kimlik bilgisi depolamayı belgeleme
summary: 'Model kimlik doğrulaması: OAuth, API anahtarları, Claude CLI''ın yeniden kullanımı ve Anthropic kurulum belirteci'
title: Kimlik doğrulama
x-i18n:
    generated_at: "2026-07-26T23:55:59Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 1fd4bf1c73f41d297638811f568c1b11e920eba3bd1527206cbb760df51531f2
    source_path: gateway/authentication.md
    workflow: 16
---

<Note>
Bu sayfa **model sağlayıcısı** kimlik doğrulamasını (API anahtarları, OAuth, Claude CLI yeniden kullanımı, Anthropic kurulum belirteci) kapsar. **Gateway bağlantısı** kimlik doğrulaması (belirteç, parola, güvenilir proxy) için [Yapılandırma](/tr/gateway/configuration) ve [Güvenilir Proxy Kimlik Doğrulaması](/tr/gateway/trusted-proxy-auth) sayfalarına bakın.
</Note>

OpenClaw, model sağlayıcıları için OAuth'u ve API anahtarlarını destekler. Sürekli çalışan bir Gateway ana makinesi için API anahtarı en öngörülebilir seçenektir; abonelik/OAuth akışları da sağlayıcı hesabı modelinizle eşleştiğinde çalışır.

- Tam OAuth akışı ve depolama düzeni: [/concepts/oauth](/tr/concepts/oauth)
- SecretRef tabanlı kimlik doğrulama (`env`/`file`/`exec` sağlayıcıları): [Gizli Bilgi Yönetimi](/tr/gateway/secrets)
- `models status --probe` tarafından kullanılan kimlik bilgisi uygunluk/neden kodları: [Kimlik Doğrulama Bilgisi Semantiği](/tr/auth-credential-semantics)

## Önerilen kurulum: API anahtarı (herhangi bir sağlayıcı)

1. Sağlayıcı konsolunuzda bir API anahtarı oluşturun.
2. Anahtarı **Gateway ana makinesine** (`openclaw gateway` çalıştıran makineye) yerleştirin:

```bash
export <PROVIDER>_API_KEY="..."
openclaw models status
```

3. Gateway systemd/launchd altında çalışıyorsa daemon'un okuyabilmesi için anahtarı `~/.openclaw/.env` içine yerleştirin:

```bash
cat >> ~/.openclaw/.env <<'EOF'
<PROVIDER>_API_KEY=...
EOF
```

4. Gateway işlemini (veya daemon'u) yeniden başlatın, ardından tekrar kontrol edin:

```bash
openclaw models status
openclaw doctor
```

Ortam değişkenlerini kendiniz yönetmek istemiyorsanız `openclaw onboard`, daemon kullanımı için API anahtarlarını da depolayabilir. Ortam yükleme önceliğinin tamamı (`env.shellEnv`, `~/.openclaw/.env`, systemd/launchd) için [Ortam değişkenleri](/tr/help/environment) sayfasına bakın.

## Anthropic: Claude CLI yeniden kullanımı

Anthropic kurulum belirteciyle kimlik doğrulama desteklenen bir yol olmaya devam eder. Claude CLI yeniden kullanımı (`claude -p` tarzı kullanım) da bu entegrasyon için onaylanmıştır; ana makinede bir Claude CLI oturumu mevcut olduğunda yerel/masaüstü kullanım için tercih edilen yol budur. Uzun süre çalışan Gateway ana makineleri için açık sunucu tarafı faturalandırma denetimiyle birlikte Anthropic API anahtarı hâlâ en öngörülebilir seçenektir.

Claude CLI yeniden kullanımı için ana makine kurulumu:

```bash
# Gateway ana makinesinde çalıştırın
claude auth login
claude auth status --text
openclaw models auth login --provider anthropic --method cli --set-default
```

Bu işlem iki adımdan oluşur: önce ana makinede Claude Code ile Anthropic oturumu açın, ardından OpenClaw'a Anthropic model seçimini yerel `claude-cli` arka ucu üzerinden yönlendirmesini ve eşleşen OpenClaw kimlik doğrulama profilini depolamasını söyleyin.

Gateway hizmeti, `PATH` üzerinde `claude` öğesini çözümleyebilmelidir. Bir dağıtım standart dışı bir
çalıştırılabilir dosya yolu gerektiriyorsa bir sarmalayıcıyı
[CLI arka uç plugini](/tr/plugins/cli-backend-plugins) aracılığıyla kaydedin.

## Elle belirteç girişi

Herhangi bir sağlayıcıyla çalışır; ajan başına SQLite kimlik doğrulama deposuna yazar ve yapılandırmayı günceller:

```bash
openclaw models auth paste-token --provider openrouter
```

OpenClaw, kimlik doğrulama profillerini her ajanın `openclaw-agent.sqlite` konumundan okur. Uç nokta ayrıntıları (`baseUrl`, `api`, model kimlikleri, üst bilgiler, zaman aşımları), kimlik doğrulama profillerinde değil, `openclaw.json` veya `models.json` içindeki `models.providers.<id>` altında bulunmalıdır.

Eski bir kurulumda hâlâ `auth-profiles.json`, `auth-state.json` veya `{ "openrouter": { "apiKey": "..." } }` gibi düz bir şekil varsa bunu SQLite'a aktarmak için `openclaw doctor --fix` komutunu çalıştırın; doctor, zaman damgalı yedekleri özgün JSON dosyalarının yanında tutar.

Bedrock `auth: "aws-sdk"` gibi harici kimlik doğrulama rotaları kimlik bilgisi değildir. Adlandırılmış bir Bedrock rotası için `openclaw.json` içinde `auth.profiles.<id>.mode: "aws-sdk"` değerini ayarlayın; kimlik doğrulama profili deposuna `type: "aws-sdk"` yazmayın. `openclaw doctor --fix`, eski AWS SDK işaretleyicilerini kimlik bilgisi deposundan yapılandırma meta verilerine taşır.

### SecretRef destekli kimlik bilgileri

- `api_key` kimlik bilgileri `keyRef: { source, provider, id }` kullanabilir
- `token` kimlik bilgileri `tokenRef: { source, provider, id }` kullanabilir
- OAuth modundaki profiller SecretRef kimlik bilgilerini reddeder: `auth.profiles.<id>.mode` değeri `"oauth"` ise söz konusu profil için SecretRef destekli `keyRef`/`tokenRef` reddedilir.

## Model kimlik doğrulama durumunu kontrol etme

```bash
openclaw models status
openclaw doctor
```

Otomasyona uygun denetim; süresi dolmuş/eksik olduğunda `1`, süresi dolmak üzere olduğunda `2` koduyla çıkar:

```bash
openclaw models status --check
```

Canlı kimlik doğrulama yoklamaları (kapsamı daraltmak için `--probe-provider`, `--probe-profile`, `--probe-timeout`, `--probe-concurrency` veya `--probe-max-tokens` ekleyin):

```bash
openclaw models status --probe
```

Notlar:

- Yoklama satırları kimlik doğrulama profillerinden, ortam kimlik bilgilerinden veya `models.json` öğesinden gelebilir.
- `auth.order.<provider>` depolanmış bir profili içermiyorsa yoklama, profili denemek yerine o profil için `excluded_by_auth_order` bildirir.
- Kimlik doğrulama mevcutsa ancak OpenClaw söz konusu sağlayıcı için yoklanabilir bir modeli çözümleyemiyorsa yoklama `status: no_model` bildirir.
- Hız sınırı bekleme süreleri model kapsamlı olabilir: bir model için bekleme süresinde olan profil, aynı sağlayıcıdaki kardeş bir modele hizmet vermeye devam edebilir.

İsteğe bağlı operasyon betikleri (systemd/Termux): [Kimlik doğrulama izleme betikleri](/tr/help/scripts#auth-monitoring-scripts).

## API anahtarı döndürme (Gateway)

Bazı sağlayıcılar, bir çağrı sağlayıcının hız sınırına ulaştığında isteği yapılandırılmış alternatif bir anahtarla yeniden dener.

Sağlayıcı başına anahtar öncelik sırası:

1. `OPENCLAW_LIVE_<PROVIDER>_KEY` (tek geçersiz kılma, tek bir anahtara sabitler)
2. `<PROVIDER>_API_KEYS` (virgül/boşluk/noktalı virgülle ayrılmış liste)
3. `<PROVIDER>_API_KEY`
4. `<PROVIDER>_API_KEY_*` (bu ön eke sahip herhangi bir ortam değişkeni)

Google sağlayıcıları (`google`, `google-vertex`) ayrıca `GOOGLE_API_KEY` öğesine geri döner. Birleştirilmiş listedeki yinelenen öğeler kullanımdan önce kaldırılır.

OpenClaw yalnızca hata mesajı şunlardan biriyle eşleştiğinde sonraki anahtara geçer: `rate_limit`, `rate limit`, `429`, `quota exceeded`/`quota_exceeded`, `resource exhausted`/`resource_exhausted` veya `too many requests`. Diğer hatalarda alternatif anahtarlarla yeniden deneme yapılmaz. Tüm anahtarlar başarısız olursa son denemenin nihai hatası döndürülür.

<Note>
`ThrottlingException`, `concurrency limit reached` veya `workers_ai ... quota limit exceeded` gibi sağlayıcıya özgü ifadeler, yukarıdaki API anahtarı döndürme işleminden ayrı bir mekanizma olan **yük devretme/yeniden deneme sınıflandırmasını** (tekrarlanan başarısızlıklarda model veya sağlayıcı değiştirme) yönlendirir.
</Note>

Kaydedilmiş kimlik doğrulamayı kaldırmak, sağlayıcıdaki anahtarı iptal etmez; sağlayıcı tarafında geçersiz kılma gerektiğinde anahtarı sağlayıcı kontrol panelinde döndürün veya iptal edin.

## Gateway çalışırken sağlayıcı kimlik doğrulamasını kaldırma

Sağlayıcı kimlik doğrulamasını Gateway denetim düzlemi üzerinden kaldırdığınızda OpenClaw, söz konusu sağlayıcının kaydedilmiş kimlik doğrulama profillerini siler ve seçili model sağlayıcısı kaldırılan sağlayıcıyla eşleşen etkin sohbet/ajan çalıştırmalarını iptal eder. İptal edilen çalıştırmalar, `stopReason: "auth-revoked"` ile normal iptal/yaşam döngüsü olaylarını yayınlar; böylece bağlı istemciler, kimlik bilgileri kaldırıldığı için çalıştırmanın durduğunu gösterebilir.

## Kullanılacak kimlik bilgisini denetleme

### OpenAI ve eski `openai-codex` kimlikleri

OpenAI API anahtarı profilleri ve ChatGPT/Codex OAuth profilleri, standart sağlayıcı kimliği `openai` değerini kullanır. Yeni yapılandırma için `openai:*` profil kimliklerini ve `auth.order.openai` öğesini kullanın.

Eski yapılandırmada, kimlik doğrulama profili kimliklerinde veya `auth.order.openai-codex` içinde `openai-codex` görürseniz bunu eski geçiş girdisi olarak değerlendirin; yeni `openai-codex` profilleri oluşturmayın. Şunları çalıştırın:

```bash
openclaw doctor --fix
openclaw models auth list --provider openai
```

Doctor, eski `openai-codex:*` profil kimliklerini ve `auth.order.openai-codex` girdilerini standart `openai` rotasına yeniden yazar. OpenAI'a özgü model/çalışma zamanı yönlendirmesi için [OpenAI](/tr/providers/openai) sayfasına bakın.

### Oturum açma sırasında (CLI)

```bash
openclaw models auth login --provider openai --profile-id openai:ritsuko
openclaw models auth login --provider openai --profile-id openai:lain
```

`--profile-id`, aynı sağlayıcıya ait birden fazla OAuth oturumunu tek bir ajan içinde ayrı tutar.

`--force`, seçili ajan dizinindeki söz konusu sağlayıcıya ait kaydedilmiş kimlik doğrulama profillerini siler ve ardından aynı kimlik doğrulama akışını yeniden çalıştırır. Kaydedilmiş bir profil takılı kaldığında, süresi dolduğunda veya yanlış hesaba bağlı olduğunda bunu kullanın. Sağlayıcıdaki kimlik bilgilerini iptal etmez.

```bash
openclaw models auth login --provider anthropic --force
```

### Oturum başına (sohbet komutu)

- `/model <alias-or-id>@<profileId>`, geçerli oturum için belirli bir sağlayıcı kimlik bilgisini sabitler (örnek profil kimlikleri: `anthropic:default`, `anthropic:work`).
- `/model` (veya `/model list`) kompakt bir seçici gösterir; `/model status` tam görünümü (adaylar + sonraki kimlik doğrulama profili ve yapılandırılmışsa sağlayıcı uç noktası ayrıntıları) gösterir.

Hâlihazırda çalışan bir sohbetin kimlik doğrulama sırasını veya profil sabitlemesini değiştirirseniz yeni bir oturum başlatmak için `/new` veya `/reset` gönderin; mevcut oturumlar sıfırlanana kadar geçerli model/profil seçimlerini korur.

### Ajan başına (CLI geçersiz kılması)

Kimlik doğrulama sırası geçersiz kılmaları, söz konusu ajanın SQLite kimlik doğrulama durumunda depolanır:

```bash
openclaw models auth order get --provider anthropic
openclaw models auth order set --provider anthropic anthropic:default
openclaw models auth order clear --provider anthropic
```

Belirli bir ajanı hedeflemek için `--agent <id>` kullanın; yapılandırılmış varsayılan ajanı kullanmak için bunu atlayın. `openclaw models status --probe`, atlanan depolanmış profilleri sessizce atlamak yerine `excluded_by_auth_order` olarak gösterir.

## Sorun giderme

### "Kimlik bilgisi bulunamadı"

**Gateway ana makinesinde** bir Anthropic API anahtarı yapılandırın veya Anthropic kurulum belirteci yolunu ayarlayın, ardından tekrar kontrol edin:

```bash
openclaw models status
```

### Belirtecin süresi dolmak üzere/dolmuş

Hangi profilin süresinin dolmak üzere olduğunu görmek için `openclaw models status` komutunu çalıştırın. Bir Anthropic belirteç profili eksikse veya süresi dolmuşsa kurulum belirteci aracılığıyla yenileyin ya da Anthropic API anahtarına geçin.

## İlgili

- [Gizli bilgi yönetimi](/tr/gateway/secrets)
- [Uzaktan erişim](/tr/gateway/remote)
- [Kimlik doğrulama depolaması](/tr/concepts/oauth)
