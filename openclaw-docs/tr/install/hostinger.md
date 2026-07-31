---
read_when:
    - Hostinger'da OpenClaw kurulumu
    - OpenClaw için yönetilen bir VPS arıyorsanız
    - Hostinger 1-Click OpenClaw'u Kullanma
summary: OpenClaw'u Hostinger'da barındırın
title: Hostinger
x-i18n:
    generated_at: "2026-07-27T00:02:58Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 7dc49e741f8581928553e2426ed91f92df6e7b0c31dd8780c0d6e891a07be263
    source_path: install/hostinger.md
    workflow: 16
---

Kalıcı bir OpenClaw Gateway'i [Hostinger](https://www.hostinger.com/openclaw) üzerinde, **1-Click** yönetilen dağıtım olarak veya kendi yönettiğiniz bir **VPS** kurulumu olarak çalıştırın.

## Ön koşullar

- Hostinger hesabı ([kaydolun](https://www.hostinger.com/openclaw))
- Yaklaşık 5-10 dakika

## Seçenek A: 1-Click OpenClaw

Hostinger altyapıyı, Docker'ı ve otomatik güncellemeleri yönetir. Çalışan bir örneğe ulaşmanın en hızlı yoludur.

<Steps>
  <Step title="Satın alın ve başlatın">
    1. [Hostinger OpenClaw sayfasından](https://www.hostinger.com/openclaw) bir Managed OpenClaw planı seçin ve satın alma işlemini tamamlayın.

    <Note>
    Satın alma sırasında, önceden satın alınmış ve OpenClaw'a anında entegre edilen **Ready-to-Use AI** kredilerini seçebilirsiniz. Böylece diğer sağlayıcılarda harici hesaplara veya API anahtarlarına gerek kalmaz. Hemen sohbet etmeye başlayabilirsiniz. Alternatif olarak kurulum sırasında Anthropic, OpenAI, Google Gemini veya xAI anahtarınızı sağlayabilirsiniz.
    </Note>

  </Step>

  <Step title="Bir mesajlaşma kanalı seçin">
    Bağlanmak için bir veya daha fazla kanal seçin:

    - **WhatsApp** -- kurulum sihirbazında gösterilen QR kodunu tarayın.
    - **Telegram** -- [BotFather](https://t.me/BotFather) tarafından verilen bot token'ını yapıştırın.

  </Step>

  <Step title="Kurulumu tamamlayın">
    Örneği dağıtmak için **Finish** düğmesine tıklayın. Hazır olduğunda OpenClaw panosuna hPanel'deki **OpenClaw Overview** üzerinden erişin.
  </Step>

</Steps>

## Seçenek B: VPS üzerinde OpenClaw

Sunucu üzerinde daha fazla kontrol sağlar. Hostinger, OpenClaw'ı VPS'nize Docker aracılığıyla dağıtır; hPanel'deki **Docker Manager** üzerinden yönetirsiniz.

<Steps>
  <Step title="Bir VPS satın alın">
    1. [Hostinger OpenClaw sayfasından](https://www.hostinger.com/openclaw) bir OpenClaw on VPS planı seçin ve satın alma işlemini tamamlayın.

    <Note>
    Satın alma sırasında **Ready-to-Use AI** kredilerini seçebilirsiniz. Bunlar önceden satın alınmış ve OpenClaw'a anında entegre edilmiştir; böylece diğer sağlayıcılarda harici hesaplara veya API anahtarlarına ihtiyaç duymadan sohbet etmeye başlayabilirsiniz.
    </Note>

  </Step>

  <Step title="OpenClaw'ı yapılandırın">
    VPS hazırlandıktan sonra yapılandırma alanlarını doldurun:

    - **Gateway token** -- otomatik oluşturulur; daha sonra kullanmak üzere kaydedin.
    - **WhatsApp number** -- ülke koduyla birlikte numaranız (isteğe bağlı).
    - **Telegram bot token** -- [BotFather](https://t.me/BotFather) tarafından verilen token (isteğe bağlı).
    - **API keys** -- yalnızca satın alma sırasında Ready-to-Use AI kredilerini seçmediyseniz gereklidir.

  </Step>

  <Step title="OpenClaw'ı başlatın">
    **Deploy** düğmesine tıklayın. Çalışmaya başladıktan sonra **Open** düğmesine tıklayarak hPanel'den OpenClaw panosunu açın.
  </Step>

</Steps>

Günlükler, yeniden başlatmalar ve güncellemeler hPanel'deki Docker Manager arayüzünden yürütülür. Güncellemek üzere en son imajı çekmek için Docker Manager'da **Update** düğmesine basın.

## Kurulumunuzu doğrulayın

Bağladığınız kanalda asistanınıza "Merhaba" gönderin. OpenClaw yanıt verir ve ilk tercihlerinizi belirlemenize yardımcı olur.

## Sorun giderme

**Pano yüklenmiyor** -- Kapsayıcının hazırlanmayı tamamlaması için birkaç dakika bekleyin, ardından hPanel'deki Docker Manager günlüklerini kontrol edin.

**Docker kapsayıcısı sürekli yeniden başlıyor** -- Docker Manager günlüklerini açın ve yapılandırma hatalarını (eksik token'lar, geçersiz API anahtarları) arayın.

**Telegram botu yanıt vermiyor** -- DM eşleştirmesi gerekiyorsa bilinmeyen bir gönderici yanıt yerine kısa bir eşleştirme kodu alır. Kodu OpenClaw panosundaki sohbetten veya kapsayıcıya kabuk erişiminiz varsa `openclaw pairing approve telegram <CODE>` ile onaylayın. Bkz. [Eşleştirme](/tr/channels/pairing).

## Sonraki adımlar

- [Kanallar](/tr/channels) -- Telegram, WhatsApp, Discord ve daha fazlasını bağlayın
- [Gateway yapılandırması](/tr/gateway/configuration) -- tüm yapılandırma seçenekleri

## İlgili içerikler

- [Kuruluma genel bakış](/tr/install)
- [VPS barındırma](/tr/vps)
- [DigitalOcean](/tr/install/digitalocean)
