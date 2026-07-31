---
read_when:
    - macOS ilk katılım asistanını tasarlama
    - Kimlik doğrulama veya kimlik kurulumunu uygulama
sidebarTitle: 'Onboarding: macOS App'
summary: OpenClaw için ilk çalıştırma kurulum akışı (macOS uygulaması)
title: İlk katılım (macOS uygulaması)
x-i18n:
    generated_at: "2026-07-27T00:17:54Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 55154774886c530de92b2110d367af24e2142fac48b901f288582d8552a6ca10
    source_path: start/onboarding.md
    workflow: 16
---

macOS uygulamasının ilk çalıştırma akışı: Gateway'in nerede çalışacağını seçin, doğrulanmış bir yapay zekâ arka ucuna bağlanın, izinleri verin ve denetimi aracının kendi önyükleme ritüeline devredin.
CLI ilk kurulumu ve iki yolun karşılaştırması için [İlk Kuruluma Genel Bakış](/tr/start/onboarding-overview) bölümüne bakın.

<Steps>
<Step title="macOS uyarısını onaylayın">
<Frame>
<img src="/assets/macos-onboarding/01-macos-warning.jpeg" alt="" />
</Frame>
</Step>
<Step title="Yerel ağları bulma iznini onaylayın">
<Frame>
<img src="/assets/macos-onboarding/02-local-networks.jpeg" alt="" />
</Frame>
</Step>
<Step title="Karşılama ve güvenlik bildirimi">
<Frame caption="Görüntülenen güvenlik bildirimini okuyup buna göre karar verin">
<img src="/assets/macos-onboarding/03-security-notice.png" alt="" />
</Frame>

Güvenlik güven modeli:

- OpenClaw varsayılan olarak kişisel bir aracıdır: tek bir güvenilir operatör sınırı.
- Paylaşılan/çok kullanıcılı kurulumların sıkılaştırılması gerekir: güven sınırlarını ayırın, araç erişimini minimumda tutun ve [Güvenlik](/tr/gateway/security) yönergelerini izleyin.
- Yerel ilk kurulum, yeni yapılandırmaları varsayılan olarak `tools.profile: "coding"` değerine ayarlar; böylece yeni kurulumlar, sınırsız `full` profili olmadan dosya sistemi/çalışma zamanı araçlarını korur.
- Kancalar/Webhook'lar veya diğer güvenilmeyen içerik akışları etkinse güçlü ve modern bir model katmanı kullanın; sıkı araç politikası ve korumalı alan uygulamasını sürdürün.

</Step>
<Step title="Yerel veya Uzak">
<Frame>
<img src="/assets/macos-onboarding/04-choose-gateway.png" alt="" />
</Frame>

**Gateway** nerede çalışır?

- **Bu Mac (Yalnızca yerel):** İlk kurulum, kimlik doğrulamayı yapılandırır ve kimlik bilgilerini yerel olarak yazar.
- **Uzak (SSH/Tailnet üzerinden):** İlk kurulum yerel kimlik doğrulamayı yapılandırmaz;
  kimlik bilgileri Gateway ana makinesinde zaten bulunmalıdır. Uzak Gateway belirteci
  alanı, macOS uygulamasının bu Gateway'e bağlanmak için kullandığı belirteci saklar;
  mevcut `gateway.remote.token` SecretRef değerleri, siz onları
  değiştirene kadar korunur.
- **Daha sonra yapılandır:** Kurulumu atlayın ve uygulamayı yapılandırılmamış halde bırakın.

<Tip>
**Gateway kimlik doğrulama ipucu:**

- Gateway kimlik doğrulama modu, geri döngü bağlamalarında bile varsayılan olarak `token` değerindedir; bu nedenle yerel WS istemcilerinin kimlik doğrulaması yapması gerekir.
- `gateway.auth.mode: "none"` ayarı, tüm yerel işlemlerin bağlanmasına izin verir; bunu yalnızca tamamen güvenilir makinelerde kullanın.
- Birden fazla makineden erişim veya geri döngü dışı bağlamalar için belirteç kullanın.

</Tip>
</Step>
<Step title="CLI">
  Yerel kurulum, genel `openclaw` CLI'ı npm, pnpm veya bun aracılığıyla yükler ve
  ilk olarak npm'i tercih eder. Gateway'in kendisi için önerilen çalışma zamanı
  Node olmaya devam eder. Mevcut uyumlu kurulumlar yeniden kullanılır.
</Step>
<Step title="Yapay zekânızı bağlayın">
  Yapılandırılmış bir aracı modeline zaten sahip olan bağlı bir Gateway, bu
  sayfayı tamamen atlar ve normal aracı kullanıcı arayüzünü açar. OpenClaw ve sağlayıcı kurulumu
  yalnızca yeni veya eksik bir Gateway için çalışır.

Gateway hazır olduğunda ilk kurulum, hâlihazırda sahip olduğunuz yapay zekâ erişimini arar:
Claude Code veya Codex oturumu, `OPENAI_API_KEY` / `ANTHROPIC_API_KEY` ya da
erişilebilir bir Ollama veya LM Studio sunucusunda önceden yüklü, ölçülen etkili bağlamı
en az 16K olan araç kullanabilen bir model. Algılama, macOS uygulaması bir Linux
Gateway'e bağlandığında da dâhil olmak üzere Gateway ana makinesinde çalışır. En iyi
seçenek gerçek bir tamamlama ile test edilir ve yalnızca yanıt verdikten
sonra kaydedilir; test başarısız olduğunda uygulama otomatik olarak sonraki seçeneği
dener ve önceki seçeneğin neden başarısız olduğunu gösterir. Birden fazla seçenek bulunursa
devam etmeden önce aralarında geçiş yapabilirsiniz. Otomatik yerel keşif hiçbir zaman
bir modeli çekmez veya indirmez.

Gateway ana makinesinde Claude CLI oturumu yokken Claude aboneliği kullanmak için
Claude Code'un yüklü olduğu herhangi bir makinede `claude setup-token` komutunu çalıştırın, ardından
yazdırılan belirteci **Connect with an API key or token** altındaki
**Anthropic setup-token** alanına yapıştırın.

Yüklü Gemini CLI, Antigravity, Pi ve OpenCode CLI'ları, yeniden kullanılabilir yönlendirmeli kurulum çıkarım yolu
olarak seçilemediklerinde bağlam amacıyla gösterilir.
Gemini ve Antigravity, araçsız çıkarım yoklamasını zorunlu kılamaz. Pi ve
OpenCode, kurulum çıkarım yolları yerine tam kapsamlı aracı sistemleridir; bunların
oturum entegrasyonları ayrı çalışma zamanı ve Plugin kurulumu gerektirir.

Sağlayıcının kendi OAuth veya cihaz eşleştirme akışı üzerinden de oturum açabilirsiniz.
Yerleşik seçenekler arasında OpenAI/ChatGPT, OpenRouter, GitHub Copilot, Google
Gemini CLI, xAI, MiniMax Global ve CN ile Chutes bulunur. Liste, sabit bir uygulama listesinden
değil Gateway'in etkin metin çıkarımı sağlayıcı Plugin'lerinden gelir;
böylece başka bir sağlayıcı, sağlayıcıya özgü macOS kodu eklemeden katılabilir.

Manuel anahtar/belirteç seçici aynı sağlayıcı kayıt defterini kullanır. Her yöntemde
sağlayıcı, başlangıç modelini ve yapılandırmasını sunar; OpenClaw, kimlik doğrulama
profilini saklamadan önce kimlik bilgisini aynı canlı testle doğrular. Bir arka uç
başarılı olana kadar İleri kilitli kalır; böylece ilk aracı sohbeti çalışan bir çıkarım
olmadan başlatılamaz. Bu canlı kontrol başarılı olduktan sonra OpenClaw,
kalan çalışma alanını, Gateway'i, kanalları ve diğer isteğe bağlı özellikleri
yapılandırmaya yardımcı olabilir. OpenClaw kısa bir seçenek listesi sunduğunda uygulama
yerel seçenek kartlarını gösterir; birini seçmek seçimi gönderir ve **Şimdilik atla**
seçimi her zaman isteğe bağlı bırakır. OpenClaw'a daha sonra
Ayarlar → OpenClaw altından da erişilebilir.
</Step>
<Step title="Anıları içe aktarın (algılandığında gösterilir)">
Yerel bir Gateway için ilk kurulum, desteklenen yapay zekâ araçlarındaki anıları
Mac'te kontrol eder: Claude Code otomatik belleği, Codex birleştirilmiş anıları ve Hermes bellek
dosyaları. Herhangi biri bulunduğunda bu sayfa, her kaynağı bellek sayısıyla birlikte listeler
ve seçilen kaynakları dizinlenmiş hatırlama amacıyla aracı çalışma alanındaki
`memory/imports/` altına aktarmanıza olanak tanır. Daha önce içe aktarılmış dosyalar atlanır ve
içe aktarılacak hiçbir şey olmadığında sayfa hiç görünmez. Atlamak güvenlidir;
panodaki Bellek içe aktarma sayfası, dosya başına denetimle aynı içe aktarma işlemini daha sonra
sunar.
</Step>
<Step title="İzinler">

<Frame caption="OpenClaw'a hangi izinleri vermek istediğinizi seçin">
<img src="/assets/macos-onboarding/05-permissions.png" alt="" />
</Frame>

İlk kurulum şu işlemler için TCC izinleri ister: Otomasyon (AppleScript), Bildirimler, Erişilebilirlik, Ekran Kaydı, Mikrofon, Konuşma Tanıma, Kamera ve Konum.

</Step>
<Step title="Tamamlayın">
  Çıkarım başarılı olduktan sonra OpenClaw, kalan isteğe bağlı kurulumu devralır ve
  sizi normal aracı sohbetine aktarabilir. İzin adımlarının tamamlanması
  aynı sohbeti açar; uygulama, OpenClaw'dan önce bir çalışma alanı oluşturmaz veya ayrı bir
  aracı kurulum konuşması başlatmaz. Aracının ilk gerçek etkileşimi sırasında
  Gateway ana makinesinde neler olduğunu öğrenmek için
  [Önyükleme](/tr/start/bootstrapping) bölümüne bakın.
</Step>
</Steps>

## İlgili

- [İlk kuruluma genel bakış](/tr/start/onboarding-overview)
- [Başlarken](/tr/start/getting-started)
