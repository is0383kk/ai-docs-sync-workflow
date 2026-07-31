---
read_when:
    - OpenClaw'u yeni başlayanlara tanıtma
summary: OpenClaw, herhangi bir işletim sisteminde çalışan, yapay zekâ ajanları için çok kanallı bir gateway'dir.
title: OpenClaw
x-i18n:
    generated_at: "2026-07-27T00:02:41Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 0ce948d12d4b4fcbde2597f9b33f50b99c4f677b69e0f5d72677b2f6683291f3
    source_path: index.md
    workflow: 16
---

# OpenClaw 🦞

<p align="center">
    <img
        src="/assets/openclaw-hero-light.png"
        alt="OpenClaw"
        width="500"
        class="dark:hidden"
    />
    <img
        src="/assets/openclaw-hero-dark.png"
        alt="OpenClaw"
        width="500"
        class="hidden dark:block"
    />
</p>

> _"PUL PUL DÖK! PUL PUL DÖK!"_ — Muhtemelen bir uzay ıstakozu

<p align="center">
  <strong>Discord, Google Chat, iMessage, Matrix, Microsoft Teams, Signal, Slack, Telegram, WhatsApp, Zalo ve daha fazlasındaki yapay zekâ ajanları için tüm işletim sistemlerinde çalışan Gateway.</strong><br />
  Mesaj gönderin, cebinizden bir ajan yanıtı alın. Kanal plugin'leri, WebChat ve mobil Node'lar genelinde tek bir Gateway çalıştırın.<br />
  Kâr amacı gütmeyen <a href="https://openclaw.org">OpenClaw Vakfı</a> tarafından açık olarak geliştirilmektedir.
</p>

<Columns>
  <Card title="Başlayın" href="/tr/start/getting-started" icon="rocket">
    OpenClaw'u kurun ve Gateway'i dakikalar içinde çalıştırın.
  </Card>
  <Card title="İlk Kurulumu Çalıştırın" href="/tr/start/wizard" icon="list-checks">
    `openclaw onboard` ve eşleştirme akışlarıyla yönlendirmeli kurulum.
  </Card>
  <Card title="Bir Kanal Bağlayın" href="/tr/channels" icon="message-circle">
    Her yerden sohbet etmek için Discord, Signal, Telegram, WhatsApp ve daha fazlasını bağlayın.
  </Card>
  <Card title="Denetim Arayüzünü Açın" href="/tr/web/control-ui" icon="layout-dashboard">
    Sohbet, yapılandırma ve oturumlar için tarayıcı kontrol panelini başlatın.
  </Card>
</Columns>

## Belgelere göz atın

Mobil tarayıcılar, tam masaüstü sekme çubuğu olmadan bölüm menüsünü gösterebilir. Sayfa gövdesinden aynı üst düzey belge alanlarına ulaşmak için
bu merkez bağlantılarını kullanın.

<Columns>
  <Card title="Başlayın" href="/tr" icon="rocket">
    Genel bakış, tanıtım, ilk adımlar ve kurulum kılavuzları.
  </Card>
  <Card title="Kurulum" href="/tr/install" icon="download">
    Kurulum yolları, güncellemeler, kapsayıcılar, barındırma ve gelişmiş kurulum.
  </Card>
  <Card title="Kanallar" href="/tr/channels" icon="messages-square">
    Mesajlaşma kanalları, eşleştirme, yönlendirme, erişim grupları ve kanal kalite güvencesi.
  </Card>
  <Card title="Ajanlar" href="/tr/concepts/architecture" icon="bot">
    Mimari, oturumlar, bağlam, bellek ve çoklu ajan yönlendirmesi.
  </Card>
  <Card title="Yetenekler" href="/tr/tools" icon="wand-sparkles">
    Araçlar, Skills, cron, webhook'lar ve otomasyon yetenekleri.
  </Card>
  <Card title="ClawHub" href="/clawhub" icon="store">
    Plugin pazarı, yayımlama, seçki oluşturma ve güven rehberliği.
  </Card>
  <Card title="Modeller" href="/tr/providers" icon="brain">
    Sağlayıcılar, model yapılandırması, yük devretme ve yerel model hizmetleri.
  </Card>
  <Card title="Platformlar" href="/tr/platforms" icon="monitor-smartphone">
    macOS, Windows, iOS, Android, Node'lar ve web yüzeyleri.
  </Card>
  <Card title="Gateway ve Operasyonlar" href="/tr/gateway" icon="server">
    Gateway yapılandırması, güvenlik, tanılama ve operasyonlar.
  </Card>
  <Card title="Başvuru" href="/tr/cli" icon="terminal">
    CLI başvurusu, şemalar, RPC, sürüm notları ve şablonlar.
  </Card>
  <Card title="Yardım" href="/tr/help" icon="life-buoy">
    Sorun giderme, sık sorulan sorular, test, tanılama ve ortam denetimleri.
  </Card>
</Columns>

## OpenClaw nedir?

OpenClaw; favori sohbet uygulamalarınızı — Discord, Google Chat, iMessage, Matrix, Microsoft Teams, Signal, Slack, Telegram, WhatsApp, Zalo ve kanal plugin'leri aracılığıyla daha fazlasını — yapay zekâ kodlama ajanlarına bağlayan, **kendi sunucunuzda barındırılan bir gateway'dir**. Kendi makinenizde (veya bir sunucuda) tek bir Gateway işlemi çalıştırırsınız ve bu işlem, mesajlaşma uygulamalarınız ile her zaman kullanılabilir bir yapay zekâ asistanı arasında köprü olur.

**Kimler içindir?** Verilerinin denetiminden vazgeçmeden veya barındırılan bir hizmete bağımlı olmadan, her yerden mesaj gönderebilecekleri kişisel bir yapay zekâ asistanı isteyen geliştiriciler ve ileri düzey kullanıcılar için.

**Onu farklı kılan nedir?**

- **Kendi sunucunuzda barındırılır**: donanımınızda, kurallarınıza göre çalışır
- **Çok kanallı**: tek bir Gateway, yapılandırılan tüm kanal plugin'lerine aynı anda hizmet verir
- **Ajan odaklı**: araç kullanımı, oturumlar, bellek ve çoklu ajan yönlendirmesiyle kodlama ajanları için geliştirilmiştir
- **Açık kaynaklı**: MIT lisanslı, topluluk odaklı

**Neye ihtiyacınız var?** Node 24.15+ (önerilen), uyumluluk için Node 22 LTS (`22.22.3+`) veya Node 25.9+, seçtiğiniz sağlayıcıdan bir API anahtarı ve 5 dakika. En iyi kalite ve güvenlik için kullanılabilir en güçlü, en yeni nesil modeli kullanın.

## Nasıl çalışır?

```mermaid
flowchart LR
  A["Sohbet uygulamaları + plugin'ler"] --> B["Gateway"]
  B --> C["OpenClaw ajanı"]
  B --> D["CLI"]
  B --> E["Web Denetim Arayüzü"]
  B --> F["macOS uygulaması"]
  B --> G["iOS ve Android Node'ları"]
```

Gateway; oturumlar, yönlendirme ve kanal bağlantıları için tek doğruluk kaynağıdır.

## Temel yetenekler

<Columns>
  <Card title="Çok kanallı gateway" icon="network" href="/tr/channels">
    Tek bir Gateway işlemiyle Discord, iMessage, Signal, Slack, Telegram, WhatsApp, WebChat ve daha fazlası.
  </Card>
  <Card title="Plugin kanalları" icon="plug" href="/tr/tools/plugin">
    Kanal plugin'leri Matrix, Nostr, Twitch, Zalo ve daha fazlasını ekler; resmî plugin'ler isteğe bağlı olarak kurulur.
  </Card>
  <Card title="Çoklu ajan yönlendirmesi" icon="route" href="/tr/concepts/multi-agent">
    Ajan, çalışma alanı veya gönderici başına yalıtılmış oturumlar.
  </Card>
  <Card title="Medya desteği" icon="image" href="/tr/nodes/images">
    Görsel, ses ve belge gönderip alın.
  </Card>
  <Card title="Web Denetim Arayüzü" icon="monitor" href="/tr/web/control-ui">
    Sohbet, yapılandırma, oturumlar ve Node'lar için tarayıcı kontrol paneli.
  </Card>
  <Card title="Mobil Node'lar" icon="smartphone" href="/tr/nodes">
    Canvas, kamera ve ses özellikli iş akışları için iOS ve Android Node'larını eşleştirin.
  </Card>
</Columns>

## Hızlı başlangıç

<Steps>
  <Step title="OpenClaw'u kurun">
    ```bash
    npm install -g openclaw@latest
    ```
  </Step>
  <Step title="İlk kurulumu yapın ve hizmeti kurun">
    ```bash
    openclaw onboard --install-daemon
    ```
  </Step>
  <Step title="Sohbet edin">
    Tarayıcınızda Denetim Arayüzünü açın ve bir mesaj gönderin:

    ```bash
    openclaw dashboard
    ```

    Alternatif olarak bir kanal bağlayın ([Telegram](/tr/channels/telegram) en hızlısıdır) ve telefonunuzdan sohbet edin.

  </Step>
</Steps>

Tam kurulum ve geliştirme ortamı yönergelerine mi ihtiyacınız var? [Başlangıç](/tr/start/getting-started) sayfasına bakın.

## Kontrol paneli

Gateway başladıktan sonra tarayıcı Denetim Arayüzünü açın.

- Yerel varsayılan: [http://127.0.0.1:18789/](http://127.0.0.1:18789/)
- Uzaktan erişim: [Web yüzeyleri](/tr/web) ve [Tailscale](/tr/gateway/tailscale)

<p align="center">
  <img src="/whatsapp-openclaw.jpg" alt="OpenClaw" width="420" />
</p>

## Yapılandırma (isteğe bağlı)

Yapılandırma `~/.openclaw/openclaw.json` konumundadır.

- **Hiçbir şey yapmazsanız**, OpenClaw paketle birlikte gelen OpenClaw ajan çalışma zamanını kullanır; doğrudan mesajlar ajanın ana oturumunu paylaşır ve her grup sohbeti kendi oturumuna sahip olur.
- Erişimi kısıtlamak istiyorsanız `channels.whatsapp.allowFrom` ve (gruplar için) bahsetme kurallarıyla başlayın.

Örnek:

```json5
{
  channels: {
    whatsapp: {
      allowFrom: ["+15555550123"],
      groups: { "*": { requireMention: true } },
    },
  },
  messages: { groupChat: { mentionPatterns: ["@openclaw"] } },
}
```

## Buradan başlayın

<Columns>
  <Card title="Belge merkezleri" href="/tr/start/hubs" icon="book-open">
    Kullanım durumuna göre düzenlenmiş tüm belgeler ve kılavuzlar.
  </Card>
  <Card title="Yapılandırma" href="/tr/gateway/configuration" icon="settings">
    Temel Gateway ayarları, token'lar ve sağlayıcı yapılandırması.
  </Card>
  <Card title="Uzaktan erişim" href="/tr/gateway/remote" icon="globe">
    SSH ve tailnet erişim kalıpları.
  </Card>
  <Card title="Kanallar" href="/tr/channels/telegram" icon="message-square">
    Discord, Feishu, Microsoft Teams, Telegram, WhatsApp ve daha fazlası için kanala özgü kurulum.
  </Card>
  <Card title="Node'lar" href="/tr/nodes" icon="smartphone">
    Eşleştirme, Canvas, kamera ve cihaz eylemleriyle iOS ve Android Node'ları.
  </Card>
  <Card title="Yardım" href="/tr/help" icon="life-buoy">
    Yaygın düzeltmeler ve sorun giderme başlangıç noktası.
  </Card>
</Columns>

## Daha fazla bilgi edinin

<Columns>
  <Card title="Tam özellik listesi" href="/tr/concepts/features" icon="list">
    Tüm kanal, yönlendirme ve medya yetenekleri.
  </Card>
  <Card title="Çoklu ajan yönlendirmesi" href="/tr/concepts/multi-agent" icon="route">
    Çalışma alanı yalıtımı ve ajan başına oturumlar.
  </Card>
  <Card title="Güvenlik" href="/tr/gateway/security" icon="shield">
    Token'lar, izin listeleri ve güvenlik denetimleri.
  </Card>
  <Card title="Sorun giderme" href="/tr/gateway/troubleshooting" icon="wrench">
    Gateway tanılaması ve yaygın hatalar.
  </Card>
  <Card title="Hakkında ve katkıda bulunanlar" href="/tr/reference/credits" icon="info">
    Projenin kökenleri, katkıda bulunanlar ve lisans.
  </Card>
</Columns>
