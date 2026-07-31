---
read_when:
    - Bir ilk kurulum yolu seçme
    - Yeni bir ortam kurma
sidebarTitle: Onboarding Overview
summary: OpenClaw ilk katılım seçeneklerine ve akışlarına genel bakış
title: İlk katılım genel bakışı
x-i18n:
    generated_at: "2026-07-26T23:36:58Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 4bcda1dcfb91f388ca6bef59f9bdf5177571d93c0d89c45025ef837628fa7ba0
    source_path: start/onboarding-overview.md
    workflow: 16
---

OpenClaw, terminal ve macOS uygulamasıyla ilk katılım seçenekleri sunar. Her ikisi de önce çıkarımı yapılandırır:
mevcut yapay zekâ erişimini algılar, canlı bir tamamlamayı zorunlu tutar ve ancak bundan sonra
kalan kurulumu yapılandırmak üzere OpenClaw'ı başlatır. Varsayılan aracısında zaten yapılandırılmış
bir model bulunan, erişilebilir ve yapılandırılmış bir Gateway, ilk katılımı atlayarak
normal aracı kullanıcı arayüzünü açar. Terminal akışı ayrıca ayrıntılı kurulum için tam klasik
sihirbazı da sunar.

## Hangi yolu kullanmalıyım?

|                | CLI ile ilk katılım                         | macOS uygulamasıyla ilk katılım           |
| -------------- | ------------------------------------------- | ----------------------------------------- |
| **Platformlar**  | macOS, Linux, Windows (yerel veya WSL2) | Yalnızca macOS                     |
| **Arayüz**  | Önce çıkarım kurulumu, ardından OpenClaw         | Önce çıkarım kurulumu, ardından OpenClaw |
| **En uygun olduğu durumlar**   | Sunucular, başsız sistemler, tam denetim        | Masaüstü Mac, görsel kurulum      |
| **Otomasyon** | Betikler için `--non-interactive`        | Yalnızca elle                    |
| **Komut**    | `openclaw onboard`                     | Uygulamayı başlatın                 |

Çoğu kullanıcı **CLI ile ilk katılım** seçeneğiyle başlamalıdır — her yerde çalışır ve
en fazla denetimi sağlar.

## İlk katılım neleri yapılandırır?

Yönlendirmeli çıkarım aşaması yalnızca şunları oluşturur:

1. **Model sağlayıcısı ve kimlik doğrulama** — algılanan erişim veya doğrulanmış bir sağlayıcı oturumu,
   API anahtarı ya da belirteç
2. **Doğrulanmış çıkarım** — varsayılan aracının etkin
   modelinde gerçek bir tamamlama

Bu tamamlama başarılı olduktan sonra OpenClaw; çalışma alanını, Gateway'i,
Gateway hizmetini, kanalları, aracıları, plugin'leri ve diğer isteğe bağlı özellikleri yapılandırabilir.

Klasik CLI sihirbazı ayrıca şunları yapılandırabilir:

1. **Kanallar** (isteğe bağlı) — Discord, Feishu, Google Chat, iMessage,
   Mattermost, Microsoft Teams, Telegram, WhatsApp ve diğerleri gibi
   yerleşik ve paketlenmiş sohbet kanalları
2. **Gelişmiş Gateway denetimleri** — uzak mod, ağ ayarları ve arka plan programı seçenekleri

## CLI ile ilk katılım

Herhangi bir terminalde çalıştırın:

```bash
openclaw onboard
```

Yönlendirmeli akış mevcut yapay zekâ erişimini algılar, adayları sırayla canlı olarak
sınar ve başarısızlık durumunda bir sonrakine geçer. Algılama seçenekleri tükendiğinde önce OpenAI,
Anthropic, xAI (Grok), Google ve OpenRouter'ı gösterir. **Daha Fazla…** seçeneği,
sağlayıcı gruplarında kalan sağlayıcıları; bölgeler, planlar ve desteklenen
tarayıcı, cihaz, API anahtarı veya belirteç yöntemleriyle birlikte ikinci bir menüde içerir. Modeli
ve kimlik bilgisini yalnızca başarılı bir tamamlamadan sonra kaydeder; ardından çalışma alanını,
Gateway'i, kanalları, aracıları, plugin'leri ve diğer isteğe bağlı özellikleri
yapılandırmak üzere OpenClaw'ı başlatır. **Şimdilik Atla**, OpenClaw'ı başlatmadan çıkar. Akış içinde
klasik sihirbaza geçiş yoktur; bunun yerine klasik sihirbazı kullanmak istediğinizde çıkın ve
`openclaw onboard --classic` komutunu çalıştırın.

Çıkarım başarılı olduktan sonra OpenClaw, kanal kurulumunu maskeli bir terminal
sihirbazına devredebilir. Yönlendirmeli veya klasik sağlayıcı kurulumunu açmaz; model
sağlayıcısını ya da kimlik doğrulamasını değiştirmek için OpenClaw'dan çıkın ve
`openclaw onboard` komutunu çalıştırın.

Ayrıntılı model/kimlik doğrulama, kanal, skill, uzak Gateway veya içe aktarma kurulumu için
`openclaw onboard --classic` kullanın. `--install-daemon` eklemek de klasik
akışı seçer ve arka plan hizmetini tek adımda yükler. Çıkarımla ilgili olmayan etkileşimli
kurulum ve onarım için `openclaw
openclaw` kullanın. `openclaw
onboard --modern`, aynı canlı çıkarım
geçidini kullanan bir uyumluluk diğer adıdır.

Tam başvuru: [İlk katılım (CLI)](/tr/start/wizard)
CLI komut belgeleri: [`openclaw onboard`](/tr/cli/onboard)

## macOS uygulamasıyla ilk katılım

OpenClaw uygulamasını açın. Yapılandırılmış yerel veya uzak Gateway erişilebilirse
ve varsayılan aracının zaten yapılandırılmış bir modeli varsa uygulama, ilk katılımı
ve OpenClaw'ı atlayıp normal aracı kullanıcı arayüzünü hemen açar.

Yeni veya eksik bir Gateway için ilk çalıştırma akışı mevcut yapay zekâ
erişimini (Claude Code, Codex veya API anahtarları) algılar, en iyi
seçeneği canlı olarak sınar ve yalnızca gerçek bir yanıttan sonra kaydeder — başarısız olduğunda otomatik olarak
diğer seçeneklere geçer ve hiçbir şey bulunamazsa doğrulanmış bir elle API anahtarı girme adımı
sunar. Hassas kimlik bilgileri maskeli giriş kullanır. Çıkarım başarılı olduğunda OpenClaw başlar ve
kalan yapılandırmaya yardımcı olur.

Gemini CLI, kurulumdan sonra normal aracılar için kullanılabilir olmaya devam eder ancak
araçsız yoklamayı zorunlu kılamadığı için bu çıkarım geçidinde sunulmaz.

Tam başvuru: [İlk katılım (macOS Uygulaması)](/tr/start/onboarding)

## Özel veya listelenmemiş sağlayıcılar

Sağlayıcınız listelenmiyorsa `openclaw onboard --classic` komutunu çalıştırın,
**Özel Sağlayıcı** seçeneğini belirleyin ve şunları girin:

- Uç nokta uyumluluğu: OpenAI uyumlu (`/chat/completions`), OpenAI Responses uyumlu (`/responses`), Anthropic uyumlu (`/messages`) veya bilinmeyen (üçünü de yoklar ve otomatik algılar)
- Temel URL ve API anahtarı (uç nokta gerektirmiyorsa API anahtarı isteğe bağlıdır)
- Model kimliği ve isteğe bağlı model diğer adı

Birden fazla özel uç nokta birlikte kullanılabilir — her biri kendi uç nokta kimliğini alır.

## İlgili

- [Başlarken](/tr/start/getting-started)
- [CLI kurulum başvurusu](/tr/start/wizard-cli-reference)
