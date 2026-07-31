---
description: Real-world OpenClaw projects from the community
read_when:
    - Gerçek OpenClaw kullanım örnekleri aranıyor
    - Topluluk projesi öne çıkanları güncelleniyor
summary: OpenClaw tarafından desteklenen topluluk yapımı projeler ve entegrasyonlar
title: Vitrin
x-i18n:
    generated_at: "2026-07-26T23:41:23Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 64af6f1da52ebdccff82fe2cdb0f7a5f0cd57627b08ee796369e2933f47fbae4
    source_path: start/showcase.md
    workflow: 16
---

Topluluk tarafından geliştirilen OpenClaw projeleri: PR inceleme döngüleri, mobil uygulamalar, ev otomasyonu, ses sistemleri, geliştirici araçları ve bellek iş akışları; Telegram, WhatsApp, Discord ve terminallerde sohbet odaklı olarak geliştirildi.

<Info>
**Öne çıkarılmak ister misiniz?** Projenizi [Discord'daki #self-promotion kanalında](https://discord.gg/clawd) paylaşın veya [X'te @openclaw hesabını etiketleyin](https://x.com/openclaw).
</Info>

## Discord'dan yeniler

Kodlama, geliştirici araçları, mobil ve sohbet odaklı ürün geliştirme alanlarındaki son öne çıkanlar.

<CardGroup cols={2}>

<Card title="Dropage ile anında HTML dağıtımı" icon="cloud-arrow-up" href="https://clawhub.ai/jiantoucn/skills/dropage-deploy">
  **@jiantoucn** • `deploy` `hosting` `skill`

Agent'ınıza "bu HTML'yi dağıt" deyin ve yaklaşık bir saniye içinde herkese açık bir URL alın. Sayfaların süresi bir saat sonra kendiliğinden dolar — sunucu, yapılandırma ve kayıt gerekmez.
</Card>

<Card title="Dolandırıcılık karşıtı URL denetleyicisi" icon="shield-halved" href="https://clawhub.ai/phishguard-niki/anti-scam-guard">
  **@phishguard-niki** • `security` `phishing` `skill`

Herhangi bir URL yapıştırın ve sonucu alın. 38 kaynaktan (PhishTank, OpenPhish, CERT.PL ve daha fazlası) 2.5M+ dolandırıcılık alan adı yerel olarak eşleştirilir; böylece tarama geçmişi makineden asla çıkmaz.
</Card>

<Card title="Ürün tasarımı akıl yürütme Skills'leri" icon="pen-ruler" href="https://clawhub.ai/monikazapisekstudio/skills/socratic-dialog">
  **@monikazapisekstudio** • `product` `reasoning` `skills`

Ürün çalışmaları için üçlü: [Sokratik Diyalog](https://clawhub.ai/monikazapisekstudio/skills/socratic-dialog) yanıtlamadan önce soruyu çapraz sorgular, [Kano Modeli Stratejisti](https://clawhub.ai/monikazapisekstudio/skills/kano-model-strategist) özellikleri yerini hak edenlere göre sınıflandırır ve [Okunaklı Agent Çıktısı](https://clawhub.ai/monikazapisekstudio/skills/legible-agent-output) agent çıktısını sade bir dille yeniden yazar.
</Card>

<Card title="Alt agent'lar için posta kutusu aracısı" icon="inbox" href="https://clawhub.ai/albzhu/skills/miab-broker">
  **@albzhu** • `multi-agent` `async` `skill`

Alt agent'lar çalışırken orkestratörlerin boşta kalmasını önler: sonuçların üst agent'ı engellemek yerine bir posta kutusuna ulaştığı eşzamansız geri çağırma mekanizması.
</Card>

<Card title="Düşük RAM'li makineler için lite-mode" icon="feather" href="https://clawhub.ai/skills/lite-mode">
  **@mirajmahmudul** • `performance` `skill`

OpenClaw'ı 2-4 GB belleğe sahip makinelerde kullanılabilir tutar: boş belleği denetler ve makine takas alanını kullanmaya başlamadan önce ağır özellikleri azaltır. [GitHub'daki kaynak](https://github.com/mirajmahmudul/openclaw-lite-mode).
</Card>

<Card title="tokenomics maliyet izleyicisi" icon="coins" href="https://github.com/ncz-os/tokenomics">
  **@ncz-os** • `devtools` `costs` `tokens`

Bir NVIDIA mühendisi tarafından geliştirilen, birinci sınıf OpenClaw desteğine sahip token maliyeti izleyicisi: agent harcamalarınızın model ve oturum bazında tam olarak nereye gittiğini görün.
</Card>

<Card title="Excalidraw diyagram oluşturucu" icon="shapes" href="https://x.com/swiftlysingh/status/2009684853827281070">
  **@swiftlysingh** • `diagrams` `excalidraw` `devtools`

Sohbette bir diyagramı açıklayın ve programlı olarak oluşturulmuş bir Excalidraw taslağı alın.
</Card>

<Card title="GA4 analiz Skill'i" icon="chart-column" href="https://x.com/jdrhyne/status/2012028725710192741">
  **@jdrhyne** • `analytics` `ga4` `skill`

OpenClaw'a kendi Google Analytics sorgu aracını oluşturttu, ardından bunu paketleyip ClawHub'da yayımladı.
</Card>

<Card title="ClawEval model sıralamaları" icon="ranking-star" href="https://github.com/AIgenteur/ClawEval">
  **@AIgenteur** • `evals` `models` `devtools`

"GPU'm için hangi LLM?" sorusunu yanıtlamak üzere modelleri 59 agent rolünde karşılaştırır. Yerel model seçiminde topluluğun favorilerinden biridir.
</Card>

<Card title="Music Craft" icon="music" href="https://clawhub.ai/luischarro/music-craft">
  **@luischarro** • `music` `generation` `skill`

Sağlayıcıdan bağımsız şarkı üretimi: tek seferlik istem vermek yerine parçayı planlayın, şarkı sözlerini yapılandırın ve yetersiz sonuçları gözden geçirin. BPM, ton, yapı ve mashup denetimi sunan bir [MiniMax varyantı](https://clawhub.ai/luischarro/music-craft-minimax) içerir.
</Card>

<Card title="PR İncelemesinden Telegram Geri Bildirimine" icon="code-pull-request" href="https://x.com/i/status/2010878524543131691">
  **@bangnokia** • `review` `github` `telegram`

OpenCode değişikliği tamamlar ve bir PR açar; OpenClaw farkları inceler, önerilerle birlikte net bir birleştirme kararıyla Telegram üzerinden yanıt verir.

  <img src="/assets/showcase/pr-review-telegram.jpg" alt="Telegram üzerinden iletilen OpenClaw PR inceleme geri bildirimi" />
</Card>

<Card title="Dakikalar İçinde Şarap Mahzeni Skill'i" icon="wine-glass" href="https://x.com/i/status/2010916352454791216">
  **@prades_maxime** • `skills` `local` `csv`

"Robby"den (@openclaw) yerel bir şarap mahzeni Skill'i istendi. Örnek bir CSV dışa aktarımı ve depolama yolu talep eder, ardından Skill'i oluşturup test eder (örnekte 962 şişe).

  <img src="/assets/showcase/wine-cellar-skill.jpg" alt="OpenClaw'ın CSV'den yerel şarap mahzeni Skill'i oluşturması" />
</Card>

<Card title="Tesco Alışveriş Otopilotu" icon="cart-shopping" href="https://x.com/i/status/2009724862470689131">
  **@marchattonhere** • `automation` `browser` `shopping`

Haftalık yemek planı, düzenli alınan ürünler, teslimat aralığı ayırma, siparişi onaylama. API yok, yalnızca tarayıcı denetimi.

  <img src="/assets/showcase/tesco-shop.jpg" alt="Sohbet üzerinden Tesco alışveriş otomasyonu" />
</Card>

<Card title="SNAG ekran görüntüsünden Markdown'a" icon="scissors" href="https://github.com/am-will/snag">
  **@am-will** • `devtools` `screenshots` `markdown`

Bir ekran bölgesini kısayol tuşuyla seçin, Gemini görüntü işleme özelliğini kullanın ve anında panonuza Markdown alın.

  <img src="/assets/showcase/snag.png" alt="SNAG ekran görüntüsünden Markdown'a dönüştürme aracı" />
</Card>

<Card title="Agent'lar Kullanıcı Arayüzü" icon="window-maximize" href="https://releaseflow.net/kitze/agents-ui">
  **@kitze** • `ui` `skills` `sync`

Agent'lar, Claude, Codex ve OpenClaw genelindeki Skills'leri ve komutları yönetmek için masaüstü uygulaması.

  <img src="/assets/showcase/agents-ui.jpg" alt="Agent'lar Kullanıcı Arayüzü uygulaması" />
</Card>

<Card title="Telegram sesli notları (papla.media)" icon="microphone" href="https://papla.media/docs">
  **Topluluk** • `voice` `tts` `telegram`

papla.media TTS'yi sarmalar ve sonuçları Telegram sesli notları olarak gönderir (rahatsız edici otomatik oynatma olmadan).

  <img src="/assets/showcase/papla-tts.jpg" alt="TTS'den Telegram sesli not çıktısı" />
</Card>

<Card title="CodexMonitor" icon="eye" href="https://clawhub.ai/odrobnik/skills/codexmonitor">
  **@odrobnik** • `devtools` `codex` `brew`

Yerel OpenAI Codex oturumlarını listelemek, incelemek ve izlemek için Homebrew ile yüklenen yardımcı araç (CLI + VS Code).

  <img src="/assets/showcase/codexmonitor.png" alt="ClawHub'da CodexMonitor" />
</Card>

<Card title="Bambu 3D Yazıcı Denetimi" icon="print" href="https://clawhub.ai/tobiasbischoff/skills/bambu-cli">
  **@tobiasbischoff** • `hardware` `3d-printing` `skill`

BambuLab yazıcılarını denetleyin ve sorunlarını giderin: durum, işler, kamera, AMS, kalibrasyon ve daha fazlası.

  <img src="/assets/showcase/bambu-cli.png" alt="ClawHub'daki Bambu CLI Skill'i" />
</Card>

<Card title="Viyana ulaşımı (Wiener Linien)" icon="train" href="https://clawhub.ai/hjanuschka/skills/wienerlinien">
  **@hjanuschka** • `travel` `transport` `skill`

Viyana toplu taşıması için gerçek zamanlı kalkışlar, aksaklıklar, asansör durumu ve rota oluşturma.

  <img src="/assets/showcase/wienerlinien.png" alt="ClawHub'daki Wiener Linien Skill'i" />
</Card>

<Card title="ParentPay okul yemekleri" icon="utensils">
  **@George5562** • `automation` `browser` `parenting`

ParentPay üzerinden Birleşik Krallık okul yemeklerinin otomatik rezervasyonu. Tablo hücrelerine güvenilir biçimde tıklamak için fare koordinatlarını kullanır.
</Card>

<Card title="R2 yükleme (Dosyalarımı Bana Gönder)" icon="cloud-arrow-up" href="https://clawhub.ai/julianengel/skills/r2-upload">
  **@julianengel** • `files` `r2` `presigned-urls`

Cloudflare R2/S3'e yükleyin ve güvenli, önceden imzalanmış indirme bağlantıları oluşturun. Uzak OpenClaw örnekleri için kullanışlıdır.

  <img src="/assets/showcase/r2-upload.png" alt="ClawHub'daki R2 yükleme Skill'i" />
</Card>

<Card title="Telegram üzerinden iOS uygulaması" icon="mobile">
  **@coard** • `ios` `xcode` `app-store`

Haritalar ve ses kaydı içeren eksiksiz bir iOS uygulaması geliştirildi ve tamamen Telegram sohbeti üzerinden App Store dağıtımına hazırlandı.
</Card>

<Card title="Oura Ring sağlık asistanı" icon="heart-pulse">
  **@AS** • `health` `oura` `calendar`

Oura Ring verilerini takvim, randevular ve spor salonu programıyla bütünleştiren kişisel yapay zekâ sağlık asistanı.

  <img src="/assets/showcase/oura-health.png" alt="Oura Ring sağlık asistanı" />
</Card>

<Card title="Kev'in Rüya Takımı (14+ agent)" icon="robot" href="https://github.com/adam91holt/orchestrated-ai-articles">
  **@adam91holt** • `multi-agent` `orchestration`

Codex çalışanlarına görev devreden bir Opus 4.5 orkestratörüyle tek bir Gateway altında 14+ agent. Agent korumalı alanı için [teknik yazıya](https://github.com/adam91holt/orchestrated-ai-articles) ve [Clawdspace'e](https://github.com/adam91holt/clawdspace) bakın.
</Card>

<Card title="Linear CLI" icon="terminal" href="https://github.com/Finesssee/linear-cli">
  **@NessZerra** • `devtools` `linear` `cli`

Agent tabanlı iş akışlarıyla (Claude Code, OpenClaw) bütünleşen Linear CLI'sı. Sorunları, projeleri ve iş akışlarını terminalden yönetin.
</Card>

<Card title="Beeper CLI" icon="message" href="https://github.com/blqke/beepcli">
  **@jules** • `messaging` `beeper` `cli`

Beeper Desktop üzerinden mesajları okuyun, gönderin ve arşivleyin. Agent'ların tüm sohbetlerinizi (iMessage, WhatsApp ve daha fazlası) tek bir yerden yönetebilmesi için Beeper'ın yerel MCP API'sini kullanır.
</Card>

</CardGroup>

## Otomasyon ve iş akışları

Zamanlama, tarayıcı denetimi, destek döngüleri ve ürünün "görevi benim için yap" tarafı.

<CardGroup cols={2}>

<Card title="Winix hava temizleyici denetimi" icon="wind" href="https://x.com/antonplex/status/2010518442471006253">
  **@antonplex** • `automation` `hardware` `air-quality`

Claude Code, temizleyici denetimlerini keşfedip doğruladı; ardından oda hava kalitesini yönetme görevini OpenClaw devraldı.

  <img src="/assets/showcase/winix-air-purifier.jpg" alt="OpenClaw üzerinden Winix hava temizleyici denetimi" />
</Card>

<Card title="Güzel gökyüzü kamera çekimleri" icon="camera" href="https://x.com/signalgaining/status/2010523120604746151">
  **@signalgaining** • `automation` `camera` `skill`

Bir çatı kamerası tarafından tetiklenir: gökyüzü güzel göründüğünde OpenClaw'dan fotoğraf çekmesini isteyin. Bir Skill tasarladı ve çekimi gerçekleştirdi.

  <img src="/assets/showcase/roof-camera-sky.jpg" alt="OpenClaw tarafından çekilen çatı kamerası gökyüzü görüntüsü" />
</Card>

<Card title="Görsel sabah bilgilendirme sahnesi" icon="robot" href="https://x.com/buddyhadry/status/2010005331925954739">
  **@buddyhadry** • `automation` `briefing` `telegram`

Zamanlanmış bir istem, bir OpenClaw personası aracılığıyla her sabah tek bir sahne görseli (hava durumu, görevler, tarih, favori gönderi veya alıntı) oluşturur.
</Card>

<Card title="Padel kortu rezervasyonu" icon="calendar-check" href="https://github.com/joshp123/padel-cli">
  **@joshp123** • `automation` `booking` `cli`

Playtomic müsaitlik denetleyicisi ve rezervasyon CLI'sı. Bir daha boş kort fırsatını kaçırmayın.

  <img src="/assets/showcase/padel-screenshot.jpg" alt="padel-cli ekran görüntüsü" />
</Card>

<Card title="Muhasebe kabul süreci" icon="file-invoice-dollar">
  **Topluluk** • `automation` `email` `pdf`

E-postalardan PDF'leri toplar, belgeleri vergi danışmanı için hazırlar. Aylık muhasebe otomatik pilotta.
</Card>

<Card title="Koltuktan geliştirici modu" icon="couch" href="https://davekiss.com">
  **@davekiss** • `telegram` `migration` `astro`

Netflix izlerken Telegram üzerinden kişisel sitesinin tamamını yeniden oluşturdu — Notion'dan Astro'ya geçti, 18 gönderiyi taşıdı, DNS'i Cloudflare'e aktardı. Dizüstü bilgisayarını hiç açmadı.
</Card>

<Card title="İş arama ajanı" icon="briefcase">
  **@attol8** • `automation` `api` `skill`

İş ilanlarını arar, CV anahtar kelimeleriyle eşleştirir ve ilgili fırsatları bağlantılarıyla döndürür. JSearch API kullanılarak 30 dakikada oluşturuldu.
</Card>

<Card title="Jira Skills oluşturucu" icon="diagram-project" href="https://x.com/jdrhyne/status/2008336434827002232">
  **@jdrhyne** • `jira` `skill` `devtools`

OpenClaw, Jira'ya bağlandı ve ardından anında yeni bir Skills oluşturdu (henüz ClawHub'da bulunmadan önce).
</Card>

<Card title="Telegram üzerinden Todoist Skills" icon="list-check" href="https://x.com/iamsubhrajyoti/status/2009949389884920153">
  **@iamsubhrajyoti** • `todoist` `skill` `telegram`

Todoist görevlerini otomatikleştirdi ve OpenClaw'ın Skills'i doğrudan Telegram sohbetinde oluşturmasını sağladı.
</Card>

<Card title="TradingView analizi" icon="chart-line">
  **@bheem1798** • `finance` `browser` `automation`

Tarayıcı otomasyonu aracılığıyla TradingView'a giriş yapar, grafiklerin ekran görüntülerini alır ve talep üzerine teknik analiz gerçekleştirir. API gerekmez — yalnızca tarayıcı denetimi.
</Card>

<Card title="Araç pazarlığı (4.200 $ tasarruf)" icon="car-side" href="https://x.com/astuyve/status/2014147784098681217">
  **@astuyve** • `negotiation` `email` `automation`

OpenClaw'ı araç bayileriyle pazarlığa bıraktı: karşılıklı görüşmeleri yürüttü ve fiyatı 4.200 $ düşürdü.
</Card>

<Card title="Uçuş check-in otomatik pilotu" icon="plane-departure" href="https://x.com/armanddp/status/2008767951340794245">
  **@armanddp** • `travel` `email` `automation`

E-postadaki bir sonraki uçuşu bulur, çevrimiçi check-in işlemini tamamlar ve pencere kenarı koltuğu seçer — hava yolu uygulaması gerekmez.
</Card>

<Card title="Sigorta talebi başvurusu" icon="file-signature" href="https://x.com/avi_press/status/2013066316467560521">
  **@avi_press** • `automation` `insurance` `browser`

Özerk olarak sigorta talebinde bulundu ve takip randevusunu planladı.
</Card>

<Card title="Idealista emlak Skills'i" icon="building" href="https://x.com/quifago/status/2012458753786859872">
  **@quifago** • `real-estate` `api` `skill`

Emlak sorguları ve değerlemeleri için Idealista API CLI, ajanın sohbet üzerinden ev arayabilmesi amacıyla Skills olarak paketlendi.
</Card>

<Card title="Bahçecilik işletmesi arka ofisi" icon="seedling" href="https://news.ycombinator.com/item?id=47783940">
  **@mjsweet** • `automation` `email` `invoicing`

İş emirleri için Gmail'i izler, Telegram üzerinden gönderilen mülk fotoğraflarını analiz eder, çok sayfalı LaTeX teklif PDF'leri hazırlar ve Xero üzerinden fatura keser.
</Card>

<Card title="Slack otomatik desteği" icon="slack">
  **@henrymascot** • `slack` `automation` `support`

Bir şirketin Slack kanalını izler, faydalı yanıtlar verir ve bildirimleri Telegram'a iletir. İstenmeden, dağıtılmış bir uygulamadaki üretim hatasını özerk olarak düzeltti.
</Card>

</CardGroup>

## Bilgi ve bellek

Kişisel veya ekip bilgisini dizine ekleyen, arayan, hatırlayan ve bu bilgi üzerinde akıl yürüten sistemler.

<CardGroup cols={2}>

<Card title="xuezh Çince öğrenimi" icon="language" href="https://github.com/joshp123/xuezh">
  **@joshp123** • `learning` `voice` `skill`

OpenClaw üzerinden telaffuz geri bildirimi ve çalışma akışları sunan Çince öğrenme motoru.

  <img src="/assets/showcase/xuezh-pronunciation.jpeg" alt="xuezh telaffuz geri bildirimi" />
</Card>

<Card title="X gönderisi analiz işlem hattı" icon="hashtag" href="https://x.com/andrewjiang/status/2008388427180630155">
  **@andrewjiang** • `analysis` `x` `pipeline`

En iyi 100 X hesabındaki 4 milyon gönderiyi çekip sorgulanabilir bir analiz işlem hattına dönüştürdü.
</Card>

<Card title="Laboratuvar sonuçlarını Notion'a aktarma" icon="flask" href="https://x.com/danpeguine/status/2013388700479058068">
  **@danpeguine** • `health` `notion` `organization`

Yıllara yayılan kan tahlili sonuçlarını yapılandırılmış bir Notion veri tabanında düzenledi.
</Card>

<Card title="Obsidian ikinci beyin" icon="book" href="https://notesbylex.com/openclaw-the-missing-piece-for-obsidians-second-brain">
  **@lexandstuff** • `obsidian` `whatsapp` `memory`

Tüm belleği sürüm denetimli bir Obsidian kasasında markdown olarak saklanan, WhatsApp'taki günlük yardımcı: kalori ve egzersiz takibi, yapılacaklar listeleri ve günlük yaşam yönetimi.
</Card>

<Card title="Aile tarihi botu" icon="people-roof" href="https://news.ycombinator.com/item?id=47783940">
  **@brtkwr** • `telegram` `memory` `family`

Bir aile Telegram grup sohbetinde yer alır, 50'den fazla akrabanın hikâyelerini belgeler ve bilgiye dayalı takip soruları sorar — ana dili Nepalce olanlara Nepalce yanıt verir.
</Card>

<Card title="WhatsApp bellek kasası" icon="vault">
  **Topluluk** • `memory` `transcription` `indexing`

WhatsApp dışa aktarımlarının tamamını içe alır, 1.000'den fazla sesli notu yazıya döker, git günlükleriyle çapraz kontrol eder ve bağlantılı markdown raporları üretir.
</Card>

<Card title="Karakeep anlamsal araması" icon="magnifying-glass" href="https://github.com/jamesbrooksco/karakeep-semantic-search">
  **@jamesbrooksco** • `search` `vector` `bookmarks`

Qdrant ile OpenAI veya Ollama gömmelerini kullanarak Karakeep yer imlerine vektör araması ekler.
</Card>

<Card title="Inside-Out-2 belleği" icon="brain">
  **Topluluk** • `memory` `beliefs` `self-model`

Oturum dosyalarını önce anılara, ardından inançlara ve son olarak gelişen bir benlik modeline dönüştüren ayrı bir bellek yöneticisi.
</Card>

</CardGroup>

## Ses ve telefon

Öncelikle konuşmaya dayalı giriş noktaları, telefon köprüleri ve yoğun yazıya dökme içeren iş akışları.

<CardGroup cols={2}>

<Card title="Pebble Ring tek dokunuşla ses" icon="ring" href="https://x.com/thekitze/status/2014765279650189578">
  **@thekitze** • `voice` `wearable` `hardware`

Pebble Ring'e tek dokunuş, OpenClaw ile sesli görüşme başlatır — giyilebilir bir cihazdan ajan erişimi.
</Card>

<Card title="İçerik üreticisi medya stüdyosu" icon="clapperboard" href="https://x.com/cedric_chee/status/2014608153393168425">
  **@cedric_chee** • `media` `tts` `transcription`

Sohbet içinde eksiksiz bir medya stüdyosu: Codex 5.2 ve MiniMax'e bağlanmış TTS, yazıya dökme ve tarayıcı otomasyonu.
</Card>

<Card title="Action Button telsizi" icon="walkie-talkie" href="https://x.com/i/status/2072766510053888497">
  **@buddyhadry** • `voice` `ios` `mobile`

iPhone Action Button, OpenClaw'a bağlandı: basın, konuşun ve ajan telsiz gibi sesli yanıt versin.
</Card>

<Card title="Clawdia telefon köprüsü" icon="phone" href="https://github.com/alejandroOPI/clawdia-bridge">
  **@alejandroOPI** • `voice` `vapi` `bridge`

Vapi sesli yardımcısından OpenClaw HTTP köprüsüne bağlantı. Ajanınızla neredeyse gerçek zamanlı telefon görüşmeleri.
</Card>

<Card title="OpenRouter yazıya dökme" icon="microphone" href="https://clawhub.ai/obviyus/skills/openrouter-transcribe">
  **@obviyus** • `transcription` `multilingual` `skill`

OpenRouter (Gemini ve diğerleri) üzerinden çok dilli ses yazıya dökme. ClawHub'da kullanılabilir.

  <img src="/assets/showcase/openrouter-transcribe.png" alt="ClawHub'daki OpenRouter yazıya dökme Skills'i" />
</Card>

</CardGroup>

## Altyapı ve dağıtım

OpenClaw'ın çalıştırılmasını ve genişletilmesini kolaylaştıran paketleme, dağıtım ve entegrasyonlar.

<CardGroup cols={2}>

<Card title="Home Assistant eklentisi" icon="home" href="https://github.com/ngutman/openclaw-ha-addon">
  **@ngutman** • `homeassistant` `docker` `raspberry-pi`

SSH tüneli desteği ve kalıcı durum ile Home Assistant OS üzerinde çalışan OpenClaw Gateway.
</Card>

<Card title="Home Assistant Skills'i" icon="toggle-on" href="https://clawhub.ai/homeofe/skills/openclaw-homeassistant">
  **@homeofe** • `homeassistant` `skill` `automation`

Home Assistant cihazlarını doğal dil aracılığıyla denetleyin ve otomatikleştirin.

  <img src="/assets/showcase/homeassistant.png" alt="ClawHub'daki Home Assistant Skills'i" />
</Card>

<Card title="macOS menü çubuğu yöneticisi" icon="desktop" href="https://x.com/MagiMetal/status/2009424267801485362">
  **@MagiMetal** • `macos` `swift` `ui`

Hızlı denetimlerle ajan durumunu gösteren yerel Swift menü çubuğu uygulaması.
</Card>

<Card title="Nix paketleme" icon="snowflake" href="https://github.com/openclaw/nix-openclaw">
  **@openclaw** • `nix` `packaging` `deployment`

Tekrarlanabilir dağıtımlar için her şey dâhil, Nix'e uyarlanmış OpenClaw yapılandırması.
</Card>

<Card title="CalDAV takvimi" icon="calendar" href="https://clawhub.ai/asleep123/skills/caldav-calendar">
  **@asleep123** • `calendar` `caldav` `skill`

khal ve vdirsyncer kullanan takvim Skills'i. Kendi sunucunuzda barındırılan takvim entegrasyonu.

  <img src="/assets/showcase/caldav-calendar.png" alt="ClawHub'daki CalDAV takvim Skills'i" />
</Card>

</CardGroup>

## Ev ve donanım

OpenClaw'ın fiziksel dünyadaki yönü: evler, sensörler, kameralar, elektrikli süpürgeler ve diğer cihazlar.

<CardGroup cols={2}>

<Card title="Kendi oluşturduğu HomePod Skills'i" icon="volume-high" href="https://x.com/localghost/status/2014763987683225685">
  **@localghost** • `homepod` `discovery` `skill`

OpenClaw, yerel ağdaki HomePod'ları buldu ve onları denetlemek için kendine bir Skills yazdı.
</Card>

<Card title="35 $'lık holo küp arayüzü" icon="cube" href="https://x.com/andrewjiang/status/2013140793649734032">
  **@andrewjiang** • `hardware` `display` `fun`

Masa üzerinde ajanın fiziksel yüzü olarak kullanılan ucuz bir holografik küp.
</Card>

<Card title="GoHome otomasyonu" icon="house-signal" href="https://github.com/joshp123/gohome">
  **@joshp123** • `home` `nix` `grafana`

Arayüz olarak OpenClaw'ı kullanan Nix'e özgü ev otomasyonu ve Grafana panoları.

  <img src="/assets/showcase/gohome-grafana.png" alt="GoHome Grafana panosu" />
</Card>

<Card title="Roborock elektrikli süpürge" icon="robot" href="https://github.com/joshp123/gohome/tree/main/plugins/roborock">
  **@joshp123** • `vacuum` `iot` `plugin`

Roborock robot süpürgenizi doğal konuşma yoluyla denetleyin.

  <img src="/assets/showcase/roborock-screenshot.jpg" alt="Roborock durumu" />
</Card>

</CardGroup>

## Topluluk projeleri

Tek bir iş akışının ötesine geçerek daha geniş ürünlere veya ekosistemlere dönüşen çalışmalar.

<CardGroup cols={2}>

<Card title="StarSwap pazaryeri" icon="star" href="https://star-swap.com/">
  **Topluluk** • `marketplace` `astronomy` `webapp`

Eksiksiz astronomi ekipmanı pazaryeri. OpenClaw ekosistemiyle ve bu ekosistem etrafında oluşturuldu.
</Card>

<Card title="Clinch ajan müzakere protokolü" icon="handshake" href="https://clawhub.ai/publicstringapps/clinch">
  **@publicstringapps** • `protocol` `p2p` `skill`

Ajanlar arası açık müzakere: ajanınız diğer Node'larla anlaşmalar, programlar ve hizmet sözleşmeleri için pazarlık eder ve sonucu kriptografik olarak imzalar — yalnızca onaylar veya reddedersiniz.
</Card>

</CardGroup>

## Projenizi gönderin

<Steps>
  <Step title="Paylaşın">
    [Discord'daki #self-promotion kanalında](https://discord.gg/clawd) paylaşın veya [@openclaw hesabına tweet atın](https://x.com/openclaw).
  </Step>
  <Step title="Ayrıntıları ekleyin">
    Ne yaptığını açıklayın, depoya veya demoya bağlantı verin ve varsa bir ekran görüntüsü paylaşın.
  </Step>
  <Step title="Öne çıkın">
    Öne çıkan projeleri bu sayfaya ekleyeceğiz.
  </Step>
</Steps>

## İlgili

- [Başlarken](/tr/start/getting-started)
- [OpenClaw](/tr/start/openclaw)
- [openclaw.ai üzerindeki tam X vitrini](https://openclaw.ai/showcase/)
