---
read_when:
    - OpenClaw'ın medya özelliklerine genel bir bakış mı arıyorsunuz?
    - Hangi medya sağlayıcısının yapılandırılacağına karar verme
    - Eşzamansız medya oluşturmanın nasıl çalıştığını anlama
sidebarTitle: Media overview
summary: Görüntü, video, müzik, konuşma ve medya anlama yeteneklerine genel bakış
title: Medyaya genel bakış
x-i18n:
    generated_at: "2026-07-27T00:20:33Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 18eb79e6915c5dc8d705bf5cadfcdddecaf7d21a037f102696d4f2bcd41e5bea
    source_path: tools/media-overview.md
    workflow: 16
---

OpenClaw görüntüler, videolar ve müzik oluşturur, gelen medyayı
(görüntüler, ses, video) anlar ve metinden konuşmaya özelliğiyle yanıtları sesli
olarak söyler. Tüm medya yetenekleri araçlarla çalışır: agent, konuşmaya göre
bunları ne zaman kullanacağına karar verir ve her araç yalnızca onu destekleyen
en az bir sağlayıcı yapılandırıldığında görünür.

Canlı konuşma, tek seferlik medya aracı yolu yerine Talk oturumu sözleşmesini
kullanır. Talk'ın üç modu vardır: sağlayıcıya özgü `realtime`, yerel veya akışlı
`stt-tts` ve yalnızca gözlem amaçlı konuşma yakalama için `transcription`. Bu modlar
sağlayıcı kataloglarını, olay zarflarını ve iptal semantiğini telefon,
toplantı, tarayıcı gerçek zamanlı ve yerel bas-konuş istemcileriyle paylaşır.

## Yetenekler

<CardGroup cols={2}>
  <Card title="Görüntü oluşturma" href="/tr/tools/image-generation" icon="image">
    `image_generate` aracılığıyla metin istemlerinden veya referans görüntülerden
    görüntüler oluşturun ve düzenleyin. Sohbet oturumlarında eşzamansızdır — arka
    planda çalışır ve hazır olduğunda sonucu gönderir.
  </Card>
  <Card title="Video oluşturma" href="/tr/tools/video-generation" icon="video">
    `video_generate` aracılığıyla metinden videoya, görüntüden videoya ve videodan
    videoya dönüştürme. Eşzamansızdır — arka planda çalışır ve hazır olduğunda sonucu gönderir.
  </Card>
  <Card title="Müzik oluşturma" href="/tr/tools/music-generation" icon="music">
    `music_generate` aracılığıyla müzik veya ses parçaları oluşturun. Sohbet
    oturumlarında, paylaşılan medya oluşturma görevi yaşam döngüsünde eşzamansızdır.
  </Card>
  <Card title="Metinden konuşmaya" href="/tr/tools/tts" icon="microphone">
    `tts` aracı ve `tts` yapılandırması aracılığıyla giden
    yanıtları konuşma sesine dönüştürün. Eşzamanlıdır.
  </Card>
  <Card title="Medya anlama" href="/tr/nodes/media-understanding" icon="eye">
    Gelen görüntüleri, sesleri ve videoları görsel yetenekli model
    sağlayıcılarını ve özel medya anlama pluginlerini kullanarak özetleyin.
  </Card>
  <Card title="Konuşmadan metne" href="/tr/nodes/audio" icon="ear-listen">
    Gelen sesli mesajları toplu STT veya Voice Call akışlı STT
    sağlayıcıları aracılığıyla yazıya dökün.
  </Card>
</CardGroup>

## Sağlayıcı yetenek matrisi

<Note>
Bu tablo, özel medya oluşturma, TTS ve STT pluginlerini kapsar. Birçok
sohbet modeli sağlayıcısı (Anthropic, Google, OpenAI ve diğerleri) gelen
medyayı yanıt modelleri aracılığıyla da anlar; sağlayıcıların tam listesi için
[Medya anlama](/tr/nodes/media-understanding#provider-support-matrix) bölümüne bakın.
</Note>

| Sağlayıcı         | Görüntü | Video | Müzik | TTS | STT | Gerçek zamanlı ses | Medya anlama |
| ----------------- | :-----: | :---: | :---: | :-: | :-: | :----------------: | :----------: |
| Alibaba           |         |   ✓   |       |     |     |                    |              |
| Azure Speech      |         |       |       |  ✓  |     |                    |              |
| BytePlus          |         |   ✓   |       |     |     |                    |              |
| ComfyUI           |    ✓    |   ✓   |   ✓   |     |     |                    |              |
| Deepgram          |         |       |       |     |  ✓  |                    |              |
| DeepInfra         |    ✓    |   ✓   |       |  ✓  |  ✓  |                    |      ✓       |
| ElevenLabs        |         |       |       |  ✓  |  ✓  |                    |              |
| fal               |    ✓    |   ✓   |   ✓   |     |     |                    |              |
| Google            |    ✓    |   ✓   |   ✓   |  ✓  |  ✓  |         ✓          |      ✓       |
| Gradium           |         |       |       |  ✓  |     |                    |              |
| Inworld           |         |       |       |  ✓  |     |                    |              |
| LiteLLM           |    ✓    |       |       |     |     |                    |              |
| Yerel CLI         |         |       |       |  ✓  |     |                    |              |
| Microsoft         |         |       |       |  ✓  |     |                    |              |
| Microsoft Foundry |    ✓    |       |       |     |     |                    |              |
| MiniMax           |    ✓    |   ✓   |   ✓   |  ✓  |     |                    |              |
| Mistral           |         |       |       |     |  ✓  |                    |              |
| OpenAI            |    ✓    |   ✓   |       |  ✓  |  ✓  |         ✓          |      ✓       |
| OpenRouter        |    ✓    |   ✓   |   ✓   |  ✓  |  ✓  |                    |      ✓       |
| PixVerse          |         |   ✓   |       |     |     |                    |              |
| Qwen              |         |   ✓   |       |     |     |                    |      ✓       |
| Runway            |         |   ✓   |       |     |     |                    |              |
| SenseAudio        |         |       |       |     |  ✓  |                    |              |
| Together          |         |   ✓   |       |     |     |                    |              |
| Volcengine        |         |       |       |  ✓  |     |                    |              |
| Vydra             |    ✓    |   ✓   |       |  ✓  |     |                    |              |
| xAI               |    ✓    |   ✓   |       |  ✓  |  ✓  |                    |      ✓       |
| Xiaomi MiMo       |         |       |       |  ✓  |     |                    |              |

<Note>
Buradaki **gerçek zamanlı ses**, sağlayıcıya özgü çift yönlü gerçek zamanlı iletişim
(Talk `realtime` modu; ör. Gemini Live veya OpenAI Realtime API) anlamına gelir —
şu anda bunu yalnızca Google ve OpenAI kaydeder. Deepgram, ElevenLabs, Mistral, OpenAI ve xAI
ayrıca Voice Call akışlı STT'yi (tek yönlü sesten metne) ayrı olarak kaydeder; aşağıdaki
[Konuşmadan metne ve Voice Call](#speech-to-text-and-voice-call) bölümüne bakın.
xAI Realtime ses, üst kaynakta bulunan bir yetenektir ancak paylaşılan gerçek zamanlı ses
sözleşmesi bunu temsil edebilene kadar OpenClaw'da kaydedilmez.
</Note>

## Eşzamansız ve eşzamanlı

| Yetenek          | Mod          | Nedeni                                                                                                            |
| ---------------- | ------------ | ---------------------------------------------------------------------------------------------------------------- |
| Görüntü          | Eşzamansız   | Sağlayıcı işlemesi bir sohbet turundan daha uzun sürebilir; oluşturulan ekler paylaşılan tamamlama yolunu kullanır. |
| Metinden konuşmaya | Eşzamanlı  | Sağlayıcı yanıtları saniyeler içinde döner; yanıt sesine eklenir.                                                 |
| Video            | Eşzamansız   | Sağlayıcı işlemesi 30 s ile birkaç dakika sürer; yavaş kuyruklar yapılandırılan zaman aşımına kadar çalışabilir.   |
| Müzik            | Eşzamansız   | Videoyla aynı sağlayıcı işleme özelliğine sahiptir.                                                              |

Eşzamansız araçlarda OpenClaw, isteği sağlayıcıya gönderir, hemen bir görev
kimliği döndürür ve işi görev defterinde izler. İş çalışırken agent diğer
mesajlara yanıt vermeye devam eder. Sağlayıcı işlemi tamamladığında OpenClaw,
oturumun normal görünür yanıt modu üzerinden kullanıcıyı bilgilendirebilmesi
için agent'ı oluşturulan medya yollarıyla uyandırır: yapılandırıldığında
otomatik nihai yanıt teslimi veya oturum mesaj aracını gerektirdiğinde
`message(action="send")`. İstek sahibi oturum etkin değilse ya da etkin uyandırması
başarısız olursa ve oluşturulan medyanın bir kısmı hâlâ tamamlama yanıtında
eksikse OpenClaw yalnızca eksik medyayı içeren eşgüçlü bir doğrudan yedek
gönderim yapar. Tamamlama yanıtıyla zaten teslim edilmiş medya yeniden gönderilmez.

## Konuşmadan metne ve Voice Call

Deepgram, DeepInfra, ElevenLabs, Google, Groq, Mistral, OpenAI, OpenRouter,
SenseAudio ve xAI, yapılandırıldıklarında gelen sesleri toplu
`tools.media.audio` yolu üzerinden yazıya dökebilir. Bahsetme denetimi veya
komut ayrıştırma için bir sesli notu önceden denetleyen kanal pluginleri,
yazıya dökülmüş eki gelen bağlamda işaretler; böylece paylaşılan medya anlama
geçişi aynı ses için ikinci bir STT çağrısı yapmak yerine bu dökümü yeniden
kullanır.

Deepgram, ElevenLabs, Mistral, OpenAI ve xAI ayrıca Voice Call akışlı STT
sağlayıcılarını kaydeder; böylece canlı telefon sesi, tamamlanmış bir kayıt
beklenmeden seçilen satıcıya iletilebilir.

Canlı kullanıcı konuşmaları için [Talk modunu](/tr/nodes/talk) tercih edin. Toplu ses
ekleri medya yolunda kalır; tarayıcı gerçek zamanlı sesi, yerel bas-konuş,
telefon ve toplantı sesleri Talk olaylarını ve Gateway tarafından döndürülen
oturum kapsamlı katalogları kullanmalıdır.

## Sağlayıcı eşlemeleri (satıcıların yüzeylere dağılımı)

<AccordionGroup>
  <Accordion title="Google">
    Görüntü, video, müzik, toplu TTS, toplu STT, arka uç gerçek zamanlı ses ve
    medya anlama yüzeyleri.
  </Accordion>
  <Accordion title="OpenAI">
    Görüntü, video, toplu TTS, toplu STT, Voice Call akışlı STT, arka uç
    gerçek zamanlı ses ve bellek gömme yüzeyleri.
  </Accordion>
  <Accordion title="DeepInfra">
    Sohbet/model yönlendirme, görüntü oluşturma/düzenleme, metinden videoya,
    toplu TTS, toplu STT, görüntü medyası anlama ve bellek gömme yüzeyleri.
    DeepInfra ayrıca yeniden sıralama, sınıflandırma, nesne algılama ve diğer
    yerel model türlerini sunar; OpenClaw henüz bu kategoriler için bir
    sağlayıcı sözleşmesine sahip olmadığından bu plugin bunları kaydetmez.
  </Accordion>
  <Accordion title="xAI">
    Görüntü, video, arama, kod yürütme, toplu TTS, toplu STT ve Voice Call
    akışlı STT. xAI Realtime ses, üst kaynakta bulunan bir yetenektir ancak
    paylaşılan gerçek zamanlı ses sözleşmesi bunu temsil edebilene kadar
    OpenClaw'da kaydedilmez.
  </Accordion>
</AccordionGroup>

## İlgili

- [Görüntü oluşturma](/tr/tools/image-generation)
- [Video oluşturma](/tr/tools/video-generation)
- [Müzik oluşturma](/tr/tools/music-generation)
- [Metinden konuşmaya](/tr/tools/tts)
- [Medya anlama](/tr/nodes/media-understanding)
- [Ses Node'ları](/tr/nodes/audio)
- [Talk modu](/tr/nodes/talk)
