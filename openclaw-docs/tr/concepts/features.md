---
read_when:
    - OpenClaw'ın desteklediği her şeyin tam listesini istiyorsunuz
summary: Kanallar, yönlendirme, medya ve kullanıcı deneyimi genelinde OpenClaw yetenekleri.
title: Özellikler
x-i18n:
    generated_at: "2026-07-26T23:15:06Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 5bc3ebdd87a0f6ea0f3d75d029bf7cae469ecd9db84a165bd47c4896936fe303
    source_path: concepts/features.md
    workflow: 16
---

## Öne Çıkanlar

<Columns>
  <Card title="Kanallar" icon="message-square" href="/tr/channels">
    Tek bir Gateway ile Discord, iMessage, Signal, Slack, Telegram, WhatsApp, WebChat ve daha fazlası.
  </Card>
  <Card title="Pluginler" icon="plug" href="/tr/tools/plugin">
    Resmî pluginler tek bir kurulum komutuyla Matrix, Nextcloud Talk, Nostr, Twitch, Zalo ve onlarca başka hizmet ekler.
  </Card>
  <Card title="Yönlendirme" icon="route" href="/tr/concepts/multi-agent">
    Yalıtılmış oturumlarla çoklu ajan yönlendirmesi.
  </Card>
  <Card title="Medya" icon="image" href="/tr/nodes/images">
    Görseller, sesler, videolar, belgeler ve görsel/video üretimi.
  </Card>
  <Card title="Uygulamalar ve kullanıcı arayüzü" icon="monitor" href="/tr/platforms">
    Windows Hub, tarayıcı Control UI'ı, macOS menü çubuğu uygulaması ve mobil nodelar.
  </Card>
  <Card title="Mobil nodelar" icon="smartphone" href="/tr/nodes">
    Eşleştirme, sesli iletişim/sohbet ve kapsamlı cihaz komutları sunan iOS ve Android nodeları.
  </Card>
</Columns>

## Tam liste

**Kanallar:**

- iMessage, Telegram ve WebChat temel kurulumla birlikte sunulur; diğer tüm kanallar `openclaw plugins install @openclaw/<id>` ile (veya
  `openclaw onboard` / `openclaw channels add` sırasında isteğe bağlı olarak) kurulan
  resmî pluginlerdir
- Resmî plugin kanalları: Discord, Feishu, Google Chat, IRC, LINE, Matrix, Mattermost,
  Microsoft Teams, Nextcloud Talk, Nostr, QQ Bot, Raft, Signal, Slack, SMS, Synology Chat,
  Tlon, Twitch, Voice Call, WhatsApp, Zalo ve Zalo Personal
- OpenClaw deposu dışında sürdürülen harici plugin kanalları: WeChat, Yuanbao ve Zalo ClawBot
- Bahsetmeye dayalı etkinleştirme ile grup sohbeti desteği
- İzin listeleri ve eşleştirme ile DM güvenliği

**Ajan:**

- Araç akışı sağlayan yerleşik ajan çalışma zamanı
- Çalışma alanı veya gönderen başına yalıtılmış oturumlarla çoklu ajan yönlendirmesi
- Oturumlar: doğrudan sohbetler paylaşılan `main` içinde birleştirilir; gruplar yalıtılır
- Uzun yanıtlar için akış ve parçalara ayırma

**Kimlik doğrulama ve sağlayıcılar:**

- 35'ten fazla model sağlayıcısı (Anthropic, OpenAI, Google ve daha fazlası)
- OAuth üzerinden abonelik kimlik doğrulaması (ör. OpenAI Codex)
- Özel ve kendi sunucunuzda barındırılan sağlayıcı desteği (vLLM, SGLang, Ollama, llama.cpp, LM Studio ve
  OpenAI veya Anthropic ile uyumlu herhangi bir uç nokta)

**Medya:**

- Gelen ve giden görseller, sesler, videolar ve belgeler
- Paylaşılan görsel ve video üretme özelliği yüzeyleri
- Sesli notları yazıya dökme
- Birden fazla sağlayıcıyla metinden sese dönüştürme

**Uygulamalar ve arayüzler:**

- WebChat ve tarayıcı Control UI'ı
- macOS menü çubuğu yardımcı uygulaması
- Eşleştirme, Canvas, kamera, ekran kaydı, konum ve ses özelliklerine sahip iOS nodeu
- Eşleştirme, sohbet, ses, Canvas, kamera ve cihaz komutlarına sahip Android nodeu

**Araçlar ve otomasyon:**

- Tarayıcı otomasyonu, yürütme ve korumalı alan
- Web araması (Brave, DuckDuckGo, Exa, Firecrawl, Gemini, Grok, Kimi, MiniMax Search, Ollama Web Search, Perplexity, SearXNG, Tavily)
- Cron işleri ve Heartbeat zamanlaması
- Skills, pluginler ve iş akışı işlem hatları (Lobster)

## İlgili

<CardGroup cols={2}>
  <Card title="Deneysel özellikler" href="/tr/concepts/experimental-features" icon="flask">
    Henüz varsayılan yüzeyde kullanıma sunulmamış, isteğe bağlı özellikler.
  </Card>
  <Card title="Ajan çalışma zamanı" href="/tr/concepts/agent" icon="robot">
    Ajan çalışma zamanı modeli ve çalıştırmaların nasıl yönlendirildiği.
  </Card>
  <Card title="Kanallar" href="/tr/channels" icon="message-square">
    Telegram, WhatsApp, Discord, Slack ve daha fazlasını tek bir Gateway üzerinden bağlayın.
  </Card>
  <Card title="Pluginler" href="/tr/tools/plugin" icon="plug">
    OpenClaw'u genişleten resmî ve harici pluginler.
  </Card>
</CardGroup>
