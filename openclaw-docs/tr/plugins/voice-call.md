---
read_when:
    - OpenClaw'dan dışarıya sesli arama yapmak istiyorsunuz
    - Sesli arama pluginini yapılandırıyor veya geliştiriyorsunuz
    - Telefon görüşmelerinde gerçek zamanlı sese veya akışlı transkripsiyona ihtiyacınız var
sidebarTitle: Voice call
summary: İsteğe bağlı gerçek zamanlı ses ve akışlı transkripsiyon özellikleriyle Twilio, Telnyx veya Plivo üzerinden giden sesli aramalar yapın ve gelen sesli aramaları kabul edin
title: Sesli arama plugin'i
x-i18n:
    generated_at: "2026-07-26T23:29:42Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 79f09f7b5cb99aace0960e283723d4f4408afa5f5dacd71f3c527fa62859f56f
    source_path: plugins/voice-call.md
    workflow: 16
---

Bir plugin aracılığıyla OpenClaw için sesli aramalar: giden bildirimler, çok turlu
konuşmalar, tam çift yönlü gerçek zamanlı ses, akışlı transkripsiyon ve
izin listesi politikalarıyla gelen aramalar.

**Sağlayıcılar:** `mock` (geliştirme, ağ yok), `plivo` (Voice API + XML aktarımı +
GetInput konuşması), `telnyx` (Call Control v2), `twilio` (Programmable Voice +
Media Streams).

<Note>
Voice Call plugin'i **Gateway işleminin içinde** çalışır. Uzak bir
Gateway kullanılıyorsa plugin, Gateway'in çalıştığı makineye kurulup
yapılandırılmalı ve ardından yüklenmesi için Gateway yeniden başlatılmalıdır.
</Note>

## Hızlı başlangıç

<Steps>
  <Step title="Plugin'i yükleyin">
    <Tabs>
      <Tab title="npm'den">
        ```bash
        openclaw plugins install @openclaw/voice-call
        ```
      </Tab>
      <Tab title="Yerel bir klasörden (geliştirme)">
        ```bash
        PLUGIN_SRC=./path/to/local/voice-call-plugin
        openclaw plugins install "$PLUGIN_SRC"
        cd "$PLUGIN_SRC" && pnpm install
        ```
      </Tab>
    </Tabs>

    Güncel sürüm etiketini takip etmek için yalın paketi kullanın. Yalnızca
    yeniden üretilebilir bir kurulum gerektiğinde tam bir sürümü sabitleyin. Daha
    sonra plugin'in yüklenmesi için Gateway'i yeniden başlatın.

  </Step>
  <Step title="Sağlayıcıyı ve Webhook'u yapılandırın">
    Yapılandırmayı `plugins.entries.voice-call.config` altında ayarlayın (aşağıdaki
    [Yapılandırma](#configuration) bölümüne bakın). En azından şunlar gereklidir: `provider`, sağlayıcı
    kimlik bilgileri, `fromNumber` ve herkese açık olarak erişilebilen bir Webhook URL'si.
  </Step>
  <Step title="Kurulumu doğrulayın">
    ```bash
    openclaw voicecall setup
    openclaw voicecall setup --json
    ```

    Plugin'in etkinliğini, sağlayıcı kimlik bilgilerini, Webhook erişimini ve
    yalnızca bir ses modunun (`streaming` veya `realtime`) etkin olduğunu denetler.

  </Step>
  <Step title="Temel testi çalıştırın">
    ```bash
    openclaw voicecall smoke
    openclaw voicecall smoke --to "+15555550123"
    ```

    Her ikisi de varsayılan olarak deneme çalıştırmasıdır. Kısa bir giden
    bildirim araması yapmak için `--yes` ekleyin:

    ```bash
    openclaw voicecall smoke --to "+15555550123" --yes
    ```

  </Step>
</Steps>

<Warning>
Twilio, Telnyx ve Plivo için kurulumun **herkese açık bir Webhook URL'sine** çözümlenmesi gerekir.
`publicUrl`, tünel URL'si, Tailscale URL'si veya sunma yedeği
geri döngüye ya da özel ağ alanına çözümlenirse kurulum, operatör
Webhook'larını alamayan bir sağlayıcıyı başlatmak yerine başarısız olur.
</Warning>

## Yapılandırma

`enabled: true` ancak seçilen sağlayıcının kimlik bilgileri eksikse Gateway
başlangıcı, eksik anahtarları içeren bir kurulum-tamamlanmadı uyarısı kaydeder ve
çalışma zamanını başlatmayı atlar. Komutlar, RPC çağrıları ve ajan araçları kullanıldığında
eksik yapılandırmanın tam karşılığını döndürmeye devam eder.

<Note>
Voice Call kimlik bilgileri SecretRef'leri kabul eder. `plugins.entries.voice-call.config.twilio.authToken`, `plugins.entries.voice-call.config.realtime.providers.*.apiKey`, `plugins.entries.voice-call.config.streaming.providers.*.apiKey` ve `plugins.entries.voice-call.config.tts.providers.*.apiKey` standart SecretRef yüzeyi üzerinden çözümlenir; bkz. [SecretRef kimlik bilgisi yüzeyi](/tr/reference/secretref-credential-surface).
</Note>

```json5
{
  plugins: {
    entries: {
      "voice-call": {
        enabled: true,
        config: {
          provider: "twilio", // veya "telnyx" | "plivo" | "mock"
          fromNumber: "+15550001234", // veya Twilio için TWILIO_FROM_NUMBER
          toNumber: "+15550005678",
          sessionScope: "per-phone", // per-phone | per-call
          numbers: {
            "+15550009999": {
              inboundGreeting: "Silver Fox Cards, size nasıl yardımcı olabilirim?",
              responseSystemPrompt: "Kısa ve öz yanıt veren bir beyzbol kartı uzmanısınız.",
              tts: {
                providers: {
                  openai: { speakerVoice: "alloy" },
                },
              },
            },
          },

          twilio: {
            accountSid: "ACxxxxxxxx",
            authToken: "...",
            // region: "ie1", // isteğe bağlı: us1 | ie1 | au1; varsayılan us1'dir
          },
          telnyx: {
            apiKey: "...",
            connectionId: "...",
            // Mission Control Portal'dan Telnyx Webhook genel anahtarı
            // (Base64; TELNYX_PUBLIC_KEY aracılığıyla da ayarlanabilir).
            publicKey: "...",
          },
          plivo: {
            authId: "MAxxxxxxxxxxxxxxxxxxxx",
            authToken: "...",
          },

          // Webhook sunucusu
          serve: {
            port: 3334,
            path: "/voice/webhook",
          },

          // Webhook güvenliği (tüneller/proxy'ler için önerilir)
          webhookSecurity: {
            allowedHosts: ["voice.example.com"],
            trustedProxyIPs: ["100.64.0.1"],
          },

          // Herkese açık erişim (birini seçin)
          // publicUrl: "https://example.ngrok.app/voice/webhook",
          // tunnel: { provider: "ngrok" },
          // tailscale: { mode: "funnel", path: "/voice/webhook" },

          outbound: {
            defaultMode: "notify", // notify | conversation
          },

          streaming: { enabled: true /* yalnızca Twilio; bkz. Akışlı transkripsiyon */ },
          realtime: { enabled: false /* bkz. Gerçek zamanlı sesli konuşmalar */ },
        },
      },
    },
  },
}
```

### Yapılandırma başvurusu

Yukarıda gösterilmeyen, `plugins.entries.voice-call.config` altındaki üst düzey anahtarlar:

| Anahtar                         | Varsayılan   | Notlar                                                                                             |
| ------------------------------- | ------------ | -------------------------------------------------------------------------------------------------- |
| `enabled`                       | `false`      | Ana açma/kapatma anahtarı.                                                                         |
| `inboundPolicy`                 | `"disabled"` | `disabled` \| `allowlist` \| `pairing` \| `open`. Bkz. [Gelen aramalar](#inbound-calls).           |
| `allowFrom`                     | `[]`         | `inboundPolicy: "allowlist"` için E.164 izin listesi.                                              |
| `maxDurationSeconds`            | `300`        | Yanıtlanma durumundan bağımsız olarak uygulanan, arama başına kesin süre sınırı.                    |
| `staleCallReaperSeconds`        | `120`        | Bkz. [Eski arama temizleyicisi](#stale-call-reaper). `0` bunu devre dışı bırakır.                  |
| `silenceTimeoutMs`              | `800`        | Klasik (gerçek zamanlı olmayan) akış için konuşma sonu sessizlik algılama.                          |
| `transcriptTimeoutMs`           | `180000`     | Bir turdan vazgeçmeden önce arayanın transkripsiyonunu beklemek için azami süre.                    |
| `ringTimeoutMs`                 | `30000`      | Giden aramalar için çalma zaman aşımı.                                                              |
| `maxConcurrentCalls`            | `1`          | Bu sınırı aşan giden aramalar reddedilir.                                                          |
| `outbound.notifyHangupDelaySec` | `3`          | Bildirim modunda otomatik kapatmadan önce TTS sonrasında beklenecek saniye.                         |
| `skipSignatureVerification`     | `false`      | Yalnızca yerel test içindir; üretimde asla etkinleştirmeyin.                                       |
| `store`                         | ayarlanmamış | Varsayılan `$OPENCLAW_STATE_DIR/voice-calls` yolunu geçersiz kılar (normalde `~/.openclaw/voice-calls`). |
| `agentId`                       | `"main"`     | Yanıt oluşturma ve oturum depolama için kullanılan ajan.                                           |
| `responseModel`                 | ayarlanmamış | Klasik (gerçek zamanlı olmayan) yanıtlar için varsayılan modeli geçersiz kılar.                     |
| `responseSystemPrompt`          | oluşturulan  | Klasik yanıtlar için özel sistem istemi.                                                           |
| `responseTimeoutMs`             | `30000`      | Klasik yanıt oluşturma zaman aşımı (ms).                                                           |

Twilio varsayılan olarak US1 REST uç noktasını kullanır. Aramaları desteklenen
ABD dışı bir Bölgede işlemek için `twilio.region` değerini `ie1` veya `au1` olarak ayarlayın ve
o Bölgeye ait kimlik bilgilerini kullanın. Bkz.
[Twilio'nun ABD dışı REST API kılavuzu](https://www.twilio.com/docs/global-infrastructure/using-the-twilio-rest-api-in-a-non-us-region).

<AccordionGroup>
  <Accordion title="Sağlayıcı erişimi ve güvenlik notları">
    - Twilio, Telnyx ve Plivo'nun tümü **herkese açık olarak erişilebilen** bir Webhook URL'si gerektirir.
    - `mock` yerel bir geliştirme sağlayıcısıdır (ağ çağrısı yoktur).
    - Telnyx, `skipSignatureVerification` true olmadığı sürece `telnyx.publicKey` (veya `TELNYX_PUBLIC_KEY`) gerektirir.
    - `skipSignatureVerification` yalnızca yerel test içindir.
    - Ücretsiz ngrok katmanında `publicUrl` değerini tam ngrok URL'sine ayarlayın; imza doğrulaması her zaman zorunludur.
    - `tunnel.allowNgrokFreeTierLoopbackBypass: true`, yalnızca `tunnel.provider="ngrok"` olduğunda ve `serve.bind` geri döngü olduğunda (ngrok yerel ajanı), geçersiz imzalı Twilio Webhook'larına izin verir. Yalnızca yerel geliştirme içindir.
    - Ücretsiz ngrok katmanı URL'leri değişebilir veya geçiş sayfası davranışı ekleyebilir; `publicUrl` değişirse Twilio imzaları başarısız olur. Üretim için kararlı bir alan adı veya Tailscale funnel tercih edin.

  </Accordion>
  <Accordion title="Akış bağlantısı sınırları">
    - `streaming.preStartTimeoutMs` (varsayılan `5000`), hiçbir zaman geçerli bir `start` çerçevesi göndermeyen soketleri kapatır.
    - `streaming.maxPendingConnections` (varsayılan `32`), kimliği doğrulanmamış başlangıç öncesi toplam soket sayısını sınırlar.
    - `streaming.maxPendingConnectionsPerIp` (varsayılan `4`), kaynak IP başına kimliği doğrulanmamış başlangıç öncesi soket sayısını sınırlar.
    - `streaming.maxConnections` (varsayılan `128`), tüm açık medya akışı soketlerini (bekleyen + etkin) sınırlar.

  </Accordion>
  <Accordion title="Eski yapılandırma geçişleri">
    Yapılandırma ayrıştırma, bu eski anahtarları otomatik olarak normalleştirir ve
    yerine geçen yolu belirten bir uyarı kaydeder; uyumluluk katmanı gelecekteki bir
    sürümde (`2026.6.0`) kaldırılacağından, kaydedilmiş yapılandırmayı
    standart şekle yeniden yazmak için `openclaw doctor --fix` çalıştırın:

    - `provider: "log"` → `provider: "mock"`
    - `twilio.from` → `fromNumber`
    - `streaming.sttProvider` → `streaming.provider`
    - `streaming.openaiApiKey` → `streaming.providers.openai.apiKey`
    - `streaming.sttModel` → `streaming.providers.openai.model`
    - `streaming.silenceDurationMs` → `streaming.providers.openai.silenceDurationMs`
    - `streaming.vadThreshold` → `streaming.providers.openai.vadThreshold`
    - `realtime.agentContext.includeSystemPrompt` kaldırılmıştır (gerçek zamanlı bağlam artık oluşturulan ajan istemini kullanır)

  </Accordion>
</AccordionGroup>

## Oturum kapsamı

Voice Call varsayılan olarak `sessionScope: "per-phone"` kullanır; böylece
aynı arayanın tekrarlanan aramalarında konuşma belleği korunur. Her
operatör aramasının yeni bir bağlamla başlaması gerektiğinde `sessionScope: "per-call"` ayarlayın;
örneğin aynı telefon numarasının farklı toplantıları temsil edebileceği resepsiyon,
rezervasyon, IVR veya Google Meet köprü akışlarında.

Voice Call, oluşturulan oturum anahtarlarını yapılandırılmış ajan ad alanında
(`agent:<agentId>:voice:*`) depolar. Açıkça belirtilen ham entegrasyon anahtarları
aynı ad alanına çözümlenir: standart bir `agent:<configuredAgentId>:*` anahtarı bu
sahibi korur ve çekirdek `session.mainKey`/genel kapsam takma adlandırmasına uyar; yabancı veya
hatalı biçimlendirilmiş `agent:*` girdisi, yapılandırılmış ajan altında opak bir anahtar
olarak kapsamlandırılır; `global` ve `unknown` genel sentinel değerleri olarak kalır.

## Gerçek zamanlı sesli konuşmalar

`realtime`, canlı arama sesi için tam çift yönlü gerçek zamanlı bir ses sağlayıcısı seçer.
Bu, sesi yalnızca gerçek zamanlı transkripsiyon
sağlayıcılarına ileten `streaming` özelliğinden ayrıdır.

<Warning>
`realtime.enabled`, `streaming.enabled` ile birleştirilemez. Her arama için
bir ses modu seçin.
</Warning>

Geçerli çalışma zamanı davranışı:

- `realtime.enabled`, Twilio ve Telnyx için desteklenir.
- `realtime.provider` isteğe bağlıdır. Ayarlanmamışsa Voice Call, kaydedilen ilk gerçek zamanlı ses sağlayıcısını kullanır.
- Paketle birlikte gelen gerçek zamanlı ses sağlayıcıları: sağlayıcı pluginleri tarafından kaydedilen Google Gemini Live (`google`) ve OpenAI (`openai`).
- Sağlayıcının sahip olduğu ham yapılandırma `realtime.providers.<providerId>` altında bulunur.
- Voice Call, paylaşılan `openclaw_agent_consult` gerçek zamanlı aracını varsayılan olarak kullanıma sunar. Arayan kişi daha derin akıl yürütme, güncel bilgi veya normal OpenClaw araçları istediğinde gerçek zamanlı model bu aracı çağırabilir.
- `realtime.consultPolicy`, gerçek zamanlı modelin `openclaw_agent_consult` aracını ne zaman çağırması gerektiğine ilişkin isteğe bağlı yönlendirme ekler.
- `realtime.agentContext.enabled` varsayılan olarak kapalıdır. Etkinleştirildiğinde Voice Call, oturum kurulumu sırasında gerçek zamanlı sağlayıcı talimatlarına sınırlandırılmış bir aracı kimliği ve seçili çalışma alanı dosyalarından oluşan bir kapsül ekler.
- `realtime.fastContext.enabled` varsayılan olarak kapalıdır. Etkinleştirildiğinde Voice Call, danışma sorusu için önce dizine alınmış bellek/oturum bağlamında arama yapar ve yalnızca `realtime.fastContext.fallbackToConsult` doğruysa tam danışma aracısına geri dönmeden önce bu parçaları `realtime.fastContext.timeoutMs` içinde gerçek zamanlı modele döndürür.
- `realtime.provider` kaydedilmemiş bir sağlayıcıyı gösteriyorsa veya hiçbir gerçek zamanlı ses sağlayıcısı kaydedilmemişse Voice Call bir uyarı kaydeder ve pluginin tamamını başarısız kılmak yerine gerçek zamanlı medyayı atlar.
- `realtime.enabled` doğru olduğunda `inboundPolicy`, `"disabled"` olmamalıdır; `validateProviderConfig` bu birleşimi reddeder.
- Danışma oturumu anahtarları, mevcut olduğunda depolanan çağrı oturumunu yeniden kullanır; ardından yapılandırılmış `sessionScope` değerine geri döner (varsayılan olarak `per-phone` veya yalıtılmış çağrılar için `per-call`).

### Araç politikası

`realtime.toolPolicy`, danışma çalıştırmasını denetler:

| Politika           | Davranış                                                                                                                                 |
| ---------------- | ---------------------------------------------------------------------------------------------------------------------------------------- |
| `safe-read-only` | Danışma aracını kullanıma sunar ve normal aracıyı `read`, `web_search`, `web_fetch`, `x_search`, `memory_search` ve `memory_get` ile sınırlar. |
| `owner`          | Danışma aracını kullanıma sunar ve normal aracının normal aracı araç politikasını kullanmasına izin verir.                                                      |
| `none`           | Danışma aracını kullanıma sunmaz. Özel `realtime.tools` yine de gerçek zamanlı sağlayıcıya iletilir.                               |

`realtime.consultPolicy` yalnızca gerçek zamanlı model talimatlarını denetler:

| Politika        | Yönlendirme                                                                                        |
| ------------- | ----------------------------------------------------------------------------------------------- |
| `auto`        | Varsayılan istemi korur ve danışma aracının ne zaman çağrılacağına sağlayıcının karar vermesine izin verir.              |
| `substantive` | Basit konuşma bağlantılarını doğrudan yanıtlar; gerçekler, bellek, araçlar veya bağlamdan önce danışır. |
| `always`      | Her önemli yanıttan önce danışır.                                                        |

### Aracı ses bağlamı

Ses köprüsünün, sıradan dönüşlerde tam bir aracı danışma gidiş dönüşünün
maliyetini üstlenmeden yapılandırılmış OpenClaw aracısı gibi duyulması
gerektiğinde `realtime.agentContext` seçeneğini etkinleştirin. Bağlam kapsülü,
gerçek zamanlı oturum oluşturulduğunda bir kez eklenir; bu nedenle dönüş
başına gecikme eklemez. `openclaw_agent_consult` çağrıları yine de tam OpenClaw
aracısını çalıştırır ve araç çalışmaları, güncel bilgiler, bellek aramaları
veya çalışma alanı durumu için kullanılmalıdır.

```json5
{
  plugins: {
    entries: {
      "voice-call": {
        config: {
          agentId: "main",
          realtime: {
            enabled: true,
            provider: "google",
            toolPolicy: "safe-read-only",
            consultPolicy: "substantive",
            agentContext: {
              enabled: true,
              maxChars: 6000,
              includeIdentity: true,
              includeWorkspaceFiles: true,
              files: ["SOUL.md", "IDENTITY.md", "USER.md"],
            },
          },
        },
      },
    },
  },
}
```

### Gerçek zamanlı sağlayıcı örnekleri

<Tabs>
  <Tab title="Google Gemini Live">
    Varsayılanlar: `realtime.providers.google.apiKey`, `GEMINI_API_KEY`
    veya `GOOGLE_API_KEY` üzerinden API anahtarı; model `gemini-3.1-flash-live-preview`;
    ses `Kore`. Daha uzun, yeniden bağlanabilir çağrılar için
    `sessionResumption` ve `contextWindowCompression` varsayılan olarak açıktır.
    Telefon sesiyle daha hızlı söz sırası geçişlerini ayarlamak için
    `silenceDurationMs`, `startSensitivity` ve `endSensitivity` kullanın.

    ```json5
    {
      plugins: {
        entries: {
          "voice-call": {
            config: {
              provider: "twilio",
              inboundPolicy: "allowlist",
              allowFrom: ["+15550005678"],
              realtime: {
                enabled: true,
                provider: "google",
                instructions: "Kısa konuş. Daha kapsamlı araçları kullanmadan önce openclaw_agent_consult aracını çağır.",
                toolPolicy: "safe-read-only",
                consultPolicy: "substantive",
                consultThinkingLevel: "low",
                consultFastMode: true,
                agentContext: { enabled: true },
                providers: {
                  google: {
                    apiKey: "${GEMINI_API_KEY}",
                    model: "gemini-3.1-flash-live-preview",
                    speakerVoice: "Kore",
                    silenceDurationMs: 500,
                    startSensitivity: "high",
                  },
                },
              },
            },
          },
        },
      },
    }
    ```

  </Tab>
  <Tab title="OpenAI">
    ```json5
    {
      plugins: {
        entries: {
          "voice-call": {
            config: {
              realtime: {
                enabled: true,
                provider: "openai",
                providers: {
                  openai: { apiKey: "${OPENAI_API_KEY}" },
                },
              },
            },
          },
        },
      },
    }
    ```
  </Tab>
</Tabs>

Sağlayıcıya özgü gerçek zamanlı ses seçenekleri için
[Google sağlayıcısı](/tr/providers/google) ve
[OpenAI sağlayıcısı](/tr/providers/openai) bölümlerine bakın.

## Akışlı transkripsiyon

`streaming`, Twilio Media Streams'i gerçek zamanlı bir transkripsiyon sağlayıcısına bağlar.
Klasik akış yolu `provider: "twilio"` gerektirir; Telnyx, Plivo veya mock ile
yapılan yapılandırma reddedilir. Telnyx canlı sesi bunun yerine ayrı olarak
kimliği doğrulanmış `realtime.enabled` yolunu kullanır.

Geçerli çalışma zamanı davranışı:

- `streaming.provider` isteğe bağlıdır. Ayarlanmamışsa Voice Call, kaydedilen ilk gerçek zamanlı transkripsiyon sağlayıcısını kullanır.
- Paketle birlikte gelen gerçek zamanlı transkripsiyon sağlayıcıları: sağlayıcı pluginleri tarafından kaydedilen Deepgram (`deepgram`), ElevenLabs (`elevenlabs`), Mistral (`mistral`), OpenAI (`openai`) ve xAI (`xai`).
- Sağlayıcının sahip olduğu ham yapılandırma `streaming.providers.<providerId>` altında bulunur.
- Twilio, kabul edilmiş bir akış `start` iletisi gönderdikten sonra Voice Call akışı hemen kaydeder, sağlayıcı bağlanırken gelen medyayı transkripsiyon sağlayıcısı üzerinden kuyruğa alır ve ilk karşılamayı yalnızca gerçek zamanlı transkripsiyon hazır olduktan sonra başlatır.
- `streaming.provider` kaydedilmemiş bir sağlayıcıyı gösteriyorsa veya hiçbir sağlayıcı kaydedilmemişse Voice Call bir uyarı kaydeder ve pluginin tamamını başarısız kılmak yerine medya akışını atlar.

### Akış sağlayıcısı örnekleri

<Tabs>
  <Tab title="OpenAI">
    Varsayılanlar: API anahtarı `streaming.providers.openai.apiKey` veya
    `OPENAI_API_KEY`; model `gpt-4o-transcribe`; `silenceDurationMs: 800`;
    `vadThreshold: 0.5`.

    ```json5
    {
      plugins: {
        entries: {
          "voice-call": {
            config: {
              streaming: {
                enabled: true,
                provider: "openai",
                streamPath: "/voice/stream",
                providers: {
                  openai: {
                    apiKey: "sk-...", // OPENAI_API_KEY ayarlanmışsa isteğe bağlıdır
                    model: "gpt-4o-transcribe",
                    silenceDurationMs: 800,
                    vadThreshold: 0.5,
                  },
                },
              },
            },
          },
        },
      },
    }
    ```

  </Tab>
  <Tab title="xAI">
    Varsayılanlar: API anahtarı `streaming.providers.xai.apiKey` veya `XAI_API_KEY`
    (ikisi de ayarlanmamışsa bir xAI OAuth kimlik doğrulama profiline geri döner);
    uç nokta `wss://api.x.ai/v1/stt`; kodlama `mulaw`; örnekleme hızı
    `8000`; `endpointingMs: 800`; `interimResults: true`.

    ```json5
    {
      plugins: {
        entries: {
          "voice-call": {
            config: {
              streaming: {
                enabled: true,
                provider: "xai",
                streamPath: "/voice/stream",
                providers: {
                  xai: {
                    apiKey: "${XAI_API_KEY}", // XAI_API_KEY ayarlanmışsa isteğe bağlıdır
                    endpointingMs: 800,
                    language: "en",
                  },
                },
              },
            },
          },
        },
      },
    }
    ```

  </Tab>
</Tabs>

## Çağrılar için TTS

Voice Call, çağrılarda konuşma akışı için çekirdek `tts`
yapılandırmasını kullanır. Plugin yapılandırması altında **aynı yapıyla**
geçersiz kılabilirsiniz; `tts` ile derinlemesine birleştirilir.

```json5
{
  tts: {
    provider: "elevenlabs",
    providers: {
      elevenlabs: {
        speakerVoiceId: "pMsXgVXv3BLzUgSXRplE",
        modelId: "eleven_multilingual_v2",
      },
    },
  },
}
```

<Warning>
**Microsoft konuşma hizmeti, sesli çağrılarda yok sayılır.** Telefon sentezi,
telefon hedefli çıktı uygulayan bir sağlayıcı gerektirir; Microsoft konuşma
sağlayıcısı bunu uygulamadığından çağrılarda atlanır ve bunun yerine geri
dönüş zincirindeki diğer sağlayıcılar denenir.
</Warning>

Davranış notları:

- Plugin yapılandırmasındaki eski `tts.<provider>` anahtarları (`openai`, `elevenlabs`, `microsoft`, `edge`) `openclaw doctor --fix` tarafından onarılır; kaydedilen yapılandırma `tts.providers.<provider>` kullanmalıdır.
- Twilio medya akışı etkinleştirildiğinde çekirdek TTS kullanılır; aksi takdirde çağrılar sağlayıcıya özgü seslere geri döner.
- Bir Twilio medya akışı zaten etkinse Voice Call, TwiML `<Say>` seçeneğine geri dönmez. Bu durumda telefon TTS'i kullanılamıyorsa oynatma isteği iki oynatma yolunu karıştırmak yerine başarısız olur.
- Telefon TTS'i ikincil bir sağlayıcıya geri döndüğünde Voice Call, hata ayıklama için sağlayıcı zincirini (`from`, `to`, `attempts`) içeren bir uyarı kaydeder.
- Twilio araya girme veya akış kapatma işlemi bekleyen TTS kuyruğunu temizlediğinde, kuyruğa alınan oynatma istekleri oynatmanın tamamlanmasını bekleyen arayanları askıda bırakmak yerine sonuçlandırılır.

### TTS örnekleri

<Tabs>
  <Tab title="Yalnızca çekirdek TTS">
```json5
{
  tts: {
    provider: "openai",
    providers: {
      openai: { speakerVoice: "alloy" },
    },
  },
}
```
  </Tab>
  <Tab title="ElevenLabs ile geçersiz kılma (yalnızca aramalar)">
```json5
{
  plugins: {
    entries: {
      "voice-call": {
        config: {
          tts: {
            provider: "elevenlabs",
            providers: {
              elevenlabs: {
                apiKey: "elevenlabs_key",
                speakerVoiceId: "pMsXgVXv3BLzUgSXRplE",
                modelId: "eleven_multilingual_v2",
              },
            },
          },
        },
      },
    },
  },
}
```
  </Tab>
  <Tab title="OpenAI modeliyle geçersiz kılma (derin birleştirme)">
```json5
{
  plugins: {
    entries: {
      "voice-call": {
        config: {
          tts: {
            providers: {
              openai: {
                model: "gpt-4o-mini-tts",
                speakerVoice: "marin",
              },
            },
          },
        },
      },
    },
  },
}
```
  </Tab>
</Tabs>

## Gelen aramalar

Gelen arama politikası varsayılan olarak `disabled` değerindedir. Gelen aramaları etkinleştirmek için şunu ayarlayın:

```json5
{
  inboundPolicy: "allowlist",
  allowFrom: ["+15550001234"],
  inboundGreeting: "Merhaba! Nasıl yardımcı olabilirim?",
}
```

<Warning>
`inboundPolicy: "allowlist"`, düşük güvenceli bir arayan kimliği denetimidir. Plugin,
sağlayıcı tarafından sunulan `From` değerini normalleştirir ve `allowFrom` ile karşılaştırır.
Webhook doğrulaması, sağlayıcı teslimatının gerçekliğini ve yükün bütünlüğünü doğrular,
ancak PSTN/VoIP arayan numarasının sahipliğini **kanıtlamaz**. 
`allowFrom` değerini güçlü arayan kimliği olarak değil, arayan kimliği filtreleme olarak değerlendirin.
</Warning>

Otomatik yanıtlar ajan sistemini kullanır. `responseModel`,
`responseSystemPrompt` ve `responseTimeoutMs` ile ayarlayın.

### Numara başına yönlendirme

Tek bir Voice Call plugin'i birden fazla telefon numarasına gelen aramaları aldığında
ve her numaranın farklı bir hat gibi davranması gerektiğinde `numbers` kullanın. Örneğin,
bir numara gündelik bir kişisel asistan kullanırken başka bir numara kurumsal
bir kişilik, farklı bir yanıt ajanı ve farklı bir TTS sesi kullanabilir.

Rotalar, sağlayıcı tarafından sunulan ve aranan `To` numarasından seçilir. Anahtarlar
E.164 numaraları olmalıdır. Bir arama geldiğinde Voice Call, eşleşen
rotayı bir kez çözümler, eşleşen rotayı arama kaydında saklar ve bu
etkin yapılandırmayı karşılama, klasik otomatik yanıt yolu, gerçek zamanlı
danışma yolu ve TTS oynatma için yeniden kullanır. Hiçbir rota eşleşmezse genel Voice Call
yapılandırması kullanılır. Giden aramalar `numbers` kullanmaz; aramayı
başlatırken giden hedefi, mesajı ve oturumu açıkça iletin.

Rota geçersiz kılmaları şu anda şunları destekler:

- `inboundGreeting`
- `tts`
- `agentId`
- `responseModel`
- `responseSystemPrompt`
- `responseTimeoutMs`

`tts` rota değeri, genel Voice Call `tts` yapılandırması üzerine derinlemesine birleştirilir; böylece
genellikle yalnızca sağlayıcı sesini geçersiz kılabilirsiniz:

```json5
{
  inboundGreeting: "Ana hattan merhaba.",
  responseSystemPrompt: "Varsayılan sesli asistansınız.",
  tts: {
    provider: "openai",
    providers: {
      openai: { speakerVoice: "coral" },
    },
  },
  numbers: {
    "+15550001111": {
      inboundGreeting: "Silver Fox Cards, nasıl yardımcı olabilirim?",
      responseSystemPrompt: "Kısa ve öz yanıt veren bir beyzbol kartı uzmanısınız.",
      tts: {
        providers: {
          openai: { speakerVoice: "alloy" },
        },
      },
    },
  },
}
```

### Sözlü çıktı sözleşmesi

Otomatik yanıtlar için Voice Call, sistem istemine
`{"spoken":"..."}` JSON yanıtı gerektiren katı bir sözlü çıktı sözleşmesi ekler. Voice Call
konuşma metnini korumalı biçimde ayıklar:

- Akıl yürütme/hata içeriği olarak işaretlenen yükleri yok sayar.
- Doğrudan JSON'ı, kod çitli JSON'ı veya satır içi `"spoken"` anahtarlarını ayrıştırır.
- Düz metne geri döner ve muhtemel planlama/meta giriş paragraflarını kaldırır.

Bu, sözlü oynatmanın arayana yönelik metne odaklanmasını sağlar ve
planlama metninin sese sızmasını önler.

### Konuşma başlatma davranışı

Giden `conversation` aramaları için ilk mesaj işleme, canlı
oynatma durumuna bağlıdır:

- Araya girme kuyruğunu temizleme ve otomatik yanıt, yalnızca ilk karşılama etkin olarak seslendirilirken engellenir.
- İlk oynatma başarısız olursa arama `listening` durumuna döner ve ilk mesaj yeniden deneme için kuyrukta kalır.
- Twilio akışı için ilk oynatma, ek gecikme olmadan akış bağlantısında başlar.
- Araya girme, etkin oynatmayı durdurur ve kuyrukta olup henüz oynatılmayan Twilio TTS girdilerini temizler. Temizlenen girdiler atlanmış olarak çözümlenir; böylece sonraki yanıt mantığı, hiçbir zaman oynatılmayacak sesi beklemeden devam edebilir.
- Gerçek zamanlı sesli konuşmalar, gerçek zamanlı akışın kendi açılış sırasını kullanır. Voice Call, bu ilk mesaj için eski bir `<Say>` TwiML güncellemesi **göndermez**; böylece giden `<Connect><Stream>` oturumları bağlı kalır.

### Twilio akış bağlantısı kesilme ek süresi

Bir Twilio medya akışının bağlantısı kesildiğinde Voice Call, aramayı
otomatik olarak sonlandırmadan önce **2000 ms** bekler:

- Akış bu süre içinde yeniden bağlanırsa otomatik sonlandırma iptal edilir.
- Ek süre sonrasında hiçbir akış yeniden kaydolmazsa etkin aramaların takılı kalmasını önlemek için arama sonlandırılır.

## Bayat arama temizleyici

Hiçbir zaman yanıtlanmayan ve hiçbir zaman canlı konuşma durumuna ulaşmayan aramaları,
örneğin sağlayıcının hiçbir zaman sonlandırıcı Webhook göndermediği bildirim modundaki
aramaları sonlandırmak için `staleCallReaperSeconds` (varsayılan **120**) kullanın. Devre dışı bırakmak için
`0` olarak ayarlayın.

Temizleyici her 30 saniyede bir çalışır ve yalnızca
`answeredAt` zaman damgası olmayan ve halihazırda sonlandırılmış veya canlı
(`speaking`/`listening`) durumda bulunmayan aramaları sonlandırır; böylece yanıtlanan konuşmalar bu zamanlayıcı
tarafından hiçbir zaman temizlenmez. `maxDurationSeconds` (varsayılan 300), çok uzun
süren yanıtlanmış aramaları sonlandıran ayrı sınırdır.

Operatörlerin çalma/yanıtlama Webhook'larını yavaş teslim edebildiği bildirim tarzı
akışlarda, yavaş ancak normal aramaların erken temizlenmemesi için
`staleCallReaperSeconds` değerini varsayılanın üzerine çıkarın; `120`-`300` saniye makul bir üretim
aralığıdır.

```json5
{
  plugins: {
    entries: {
      "voice-call": {
        config: {
          maxDurationSeconds: 300,
          staleCallReaperSeconds: 120,
        },
      },
    },
  },
}
```

## Webhook güvenliği

Gateway'in önünde bir proxy veya tünel bulunduğunda Plugin, imza doğrulaması için
genel URL'yi yeniden oluşturur. Bu seçenekler, hangi
iletilen üstbilgilere güvenileceğini denetler:

<ParamField path="webhookSecurity.allowedHosts" type="string[]">
  İletme üstbilgilerinden gelen izin verilenler listesindeki ana makineler.
</ParamField>
<ParamField path="webhookSecurity.trustForwardingHeaders" type="boolean">
  İzin verilenler listesi olmadan iletilen üstbilgilere güvenin.
</ParamField>
<ParamField path="webhookSecurity.trustedProxyIPs" type="string[]">
  İletilen üstbilgilere yalnızca isteğin uzak IP'si listeyle eşleştiğinde güvenin.
</ParamField>

Ek korumalar:

- Webhook **yeniden oynatma koruması** Twilio, Telnyx ve Plivo için etkindir. Yeniden oynatılan geçerli Webhook istekleri onaylanır ancak yan etkileri uygulanmaz.
- Twilio konuşma sıraları, `<Gather>` geri çağırmalarında sıra başına bir belirteç içerir; böylece eski/yeniden oynatılan konuşma geri çağırmaları daha yeni bir bekleyen döküm sırasını karşılayamaz.
- Kimliği doğrulanmamış Webhook istekleri, sağlayıcının gerekli imza üstbilgileri eksik olduğunda gövde okunmadan önce reddedilir.
- Voice Call Webhook'u, imza doğrulamasından önce paylaşılan kimlik doğrulama öncesi gövde okuma profilini (en fazla 64 KB gövde, 5 saniyelik okuma zaman aşımı) ve anahtar başına devam eden istek sınırını (varsayılan olarak anahtar başına 8 eşzamanlı istek) kullanır.

Kararlı bir genel ana makine örneği:

```json5
{
  plugins: {
    entries: {
      "voice-call": {
        config: {
          publicUrl: "https://voice.example.com/voice/webhook",
          webhookSecurity: {
            allowedHosts: ["voice.example.com"],
          },
        },
      },
    },
  },
}
```

## CLI

```bash
openclaw voicecall call --to "+15555550123" --message "OpenClaw'dan merhaba"
openclaw voicecall start --to "+15555550123"   # call için takma ad
openclaw voicecall continue --call-id <id> --message "Sorunuz var mı?"
openclaw voicecall speak --call-id <id> --message "Bir dakika"
openclaw voicecall dtmf --call-id <id> --digits "ww123456#"
openclaw voicecall end --call-id <id>
openclaw voicecall status --call-id <id>
openclaw voicecall tail
openclaw voicecall latency                      # günlüklerden sıra gecikmesini özetle
openclaw voicecall expose --mode funnel
```

Gateway zaten çalışırken operasyonel `voicecall` komutları,
CLI'nin ikinci bir Webhook sunucusuna bağlanmaması için Gateway'in sahip olduğu Voice Call çalışma zamanına
devredilir. Hiçbir Gateway'e erişilemiyorsa komutlar bağımsız
CLI çalışma zamanına geri döner.

`latency`, varsayılan Voice Call depolama yolundan `calls.jsonl` okur. Farklı bir günlüğü
belirtmek için `--file <path>`, analizi son N kayıtla (varsayılan 200) sınırlamak için
`--last <n>` kullanın. Çıktı; sıra gecikmesi ve dinleme-bekleme süreleri için minimum/maksimum/ortalama,
p50 ve p95 değerlerini içerir.

## Ajan aracı

Araç adı: `voice_call`.

| Eylem          | Bağımsız değişkenler                                       |
| --------------- | ------------------------------------------ |
| `initiate_call` | `message`, `to?`, `mode?`, `dtmfSequence?` |
| `continue_call` | `callId`, `message`                        |
| `speak_to_user` | `callId`, `message`                        |
| `send_dtmf`     | `callId`, `digits`                         |
| `end_call`      | `callId`                                   |
| `get_status`    | `callId`                                   |

Voice Call Plugin'i, eşleşen bir ajan becerisiyle birlikte gelir.

## Gateway RPC

| Yöntem                      | Argümanlar                                                             | Notlar                                                                     |
| --------------------------- | ---------------------------------------------------------------- | ------------------------------------------------------------------------- |
| `voicecall.initiate`        | `to?`, `message`, `mode?`, `sessionKey?`, `requesterSessionKey?` | `to` belirtilmediğinde `toNumber` yapılandırmasına geri döner.                     |
| `voicecall.start`           | `to`, `message?`, `mode?`, `dtmfSequence?`, `sessionKey?`        | `initiate` ile aynıdır, ancak bağlantı öncesi `dtmfSequence` değerini de kabul eder.           |
| `voicecall.continue`        | `callId`, `message`                                              | Tur sonuçlanana kadar engeller; dökümü döndürür.                   |
| `voicecall.continue.start`  | `callId`, `message`                                              | Eşzamansız değişken: hemen bir `operationId` döndürür.                      |
| `voicecall.continue.result` | `operationId`                                                    | Bekleyen bir `voicecall.continue.start` işleminin sonucunu yoklar.      |
| `voicecall.speak`           | `callId`, `message`                                              | Beklemeden konuşur; `realtime.enabled` olduğunda gerçek zamanlı köprüyü kullanır. |
| `voicecall.dtmf`            | `callId`, `digits`                                               |                                                                           |
| `voicecall.end`             | `callId`                                                         |                                                                           |
| `voicecall.status`          | `callId?`                                                        | Tüm etkin çağrıları listelemek için `callId` değerini belirtmeyin.                                   |

`dtmfSequence` yalnızca `mode: "conversation"` ile geçerlidir; bildirim modundaki çağrılar,
bağlantı sonrası rakamlara ihtiyaç duyuyorsa çağrı oluşturulduktan sonra
`voicecall.dtmf` kullanmalıdır.

## Sorun giderme

### Kurulum Webhook erişimini açamıyor

Kurulumu Gateway'i çalıştıran ortamdan çalıştırın:

```bash
openclaw voicecall setup
openclaw voicecall setup --json
```

`twilio`, `telnyx` ve `plivo` için `webhook-exposure` yeşil olmalıdır. Yapılandırılmış
bir `publicUrl`, yerel veya özel ağ alanını işaret ettiğinde yine başarısız olur;
çünkü operatör bu adreslere geri çağrı yapamaz.
`publicUrl` olarak `localhost`, `127.0.0.1`, `0.0.0.0`, `10.x`, `172.16.x`-`172.31.x`,
`192.168.x`, `169.254.x`, `fc00::/7`, `fd00::/8` veya operatör sınıfı NAT
aralıklarını kullanmayın.

Twilio bildirim modundaki giden çağrılar, başlangıç `<Say>` TwiML'lerini doğrudan
çağrı oluşturma isteğinde gönderir; dolayısıyla ilk sesli mesaj, Twilio'nun
Webhook TwiML'ini getirmesine bağlı değildir. Durum geri çağrıları, konuşma çağrıları,
bağlantı öncesi DTMF, gerçek zamanlı akışlar ve bağlantı sonrası çağrı denetimi için
herkese açık bir Webhook yine de gereklidir.

Herkese açık erişim yollarından birini kullanın:

```json5
{
  plugins: {
    entries: {
      "voice-call": {
        config: {
          publicUrl: "https://voice.example.com/voice/webhook",
          // veya
          tunnel: { provider: "ngrok" },
          // veya
          tailscale: { mode: "funnel", path: "/voice/webhook" },
        },
      },
    },
  },
}
```

Yapılandırmayı değiştirdikten sonra Gateway'i yeniden başlatın veya yeniden yükleyin, ardından şunları çalıştırın:

```bash
openclaw voicecall setup
openclaw voicecall smoke
```

`--yes` iletmediğiniz sürece `voicecall smoke` bir deneme çalıştırmasıdır.

### Sağlayıcı kimlik bilgileri başarısız oluyor

Seçilen sağlayıcıyı ve gerekli kimlik bilgisi alanlarını kontrol edin:

- Twilio: `twilio.accountSid`, `twilio.authToken` ve `fromNumber` veya
  `TWILIO_ACCOUNT_SID`, `TWILIO_AUTH_TOKEN` ve `TWILIO_FROM_NUMBER`.
- Telnyx: `telnyx.apiKey`, `telnyx.connectionId`, `telnyx.publicKey` ve
  `fromNumber` veya `TELNYX_API_KEY`, `TELNYX_CONNECTION_ID` ve
  `TELNYX_PUBLIC_KEY`.
- Plivo: `plivo.authId`, `plivo.authToken` ve `fromNumber` veya
  `PLIVO_AUTH_ID` ve `PLIVO_AUTH_TOKEN`.

Kimlik bilgileri Gateway ana makinesinde bulunmalıdır. Yerel bir kabuk profilini
düzenlemek, çalışan bir Gateway yeniden başlatılana veya ortamı yeniden yüklenene
kadar onu etkilemez.

### Çağrılar başlıyor ancak sağlayıcı Webhook'ları ulaşmıyor

Sağlayıcı konsolunun herkese açık Webhook URL'sini tam olarak işaret ettiğini doğrulayın:

```text
https://voice.example.com/voice/webhook
```

Ardından çalışma zamanı durumunu inceleyin:

```bash
openclaw voicecall status --call-id <id>
openclaw voicecall tail
openclaw logs --follow
```

Yaygın nedenler:

- `publicUrl`, `serve.path` yolundan farklı bir yolu işaret ediyor.
- Gateway başlatıldıktan sonra tünel URL'si değişti.
- Bir proxy isteği iletiyor ancak ana makine/protokol başlıklarını kaldırıyor veya yeniden yazıyor.
- Güvenlik duvarı veya DNS, herkese açık ana makine adını Gateway dışında bir yere yönlendiriyor.
- Gateway, Voice Call Plugin'i etkinleştirilmeden yeniden başlatıldı.

Gateway'in önünde bir ters proxy veya tünel olduğunda
`webhookSecurity.allowedHosts` değerini herkese açık ana makine adına ayarlayın ya da
bilinen bir proxy adresi için `webhookSecurity.trustedProxyIPs` kullanın.
`webhookSecurity.trustForwardingHeaders` seçeneğini yalnızca proxy sınırı
denetiminiz altındaysa kullanın.

### İmza doğrulaması başarısız oluyor

Sağlayıcı imzaları, OpenClaw'ın gelen istekten yeniden oluşturduğu herkese açık
URL'ye göre denetlenir. İmzalar başarısız olursa:

- Sağlayıcının Webhook URL'sinin şema, ana makine ve yol dâhil olmak üzere `publicUrl` ile tam olarak eşleştiğini doğrulayın.
- ngrok ücretsiz katman URL'leri için tünelin ana makine adı değiştiğinde `publicUrl` değerini güncelleyin.
- Proxy'nin özgün ana makine ve protokol başlıklarını koruduğundan emin olun veya `webhookSecurity.allowedHosts` yapılandırmasını ayarlayın.
- Yerel testler dışında `skipSignatureVerification` seçeneğini etkinleştirmeyin.

### Google Meet Twilio katılımları başarısız oluyor

Google Meet, Twilio çevirmeli katılımları için bu Plugin'i kullanır. Önce Voice
Call'u doğrulayın:

```bash
openclaw voicecall setup
openclaw voicecall smoke --to "+15555550123"
```

Ardından Google Meet aktarımını açıkça doğrulayın:

```bash
openclaw googlemeet setup --transport twilio
```

Voice Call yeşil olduğu hâlde Meet katılımcısı hiç katılmıyorsa Meet'in
çevirmeli numarasını, PIN'ini ve `--dtmf-sequence` değerini kontrol edin. Telefon çağrısı sağlıklı
olabilirken toplantı, yanlış bir DTMF dizisini reddedebilir veya yok sayabilir.

Google Meet, Twilio telefon ayağını bağlantı öncesi DTMF dizisiyle
`voicecall.start` üzerinden başlatır. PIN'den türetilen diziler, Google Meet
Plugin'inin `voiceCall.dtmfDelayMs` değerini (varsayılan **12000 ms**) baştaki Twilio
bekleme rakamları olarak içerir; çünkü Meet çevirmeli istemleri geç ulaşabilir. Ardından Voice Call,
giriş selamlaması istenmeden önce gerçek zamanlı işleme geri yönlendirir.

Canlı aşama izlemesi için `openclaw logs --follow` kullanın. Sağlıklı bir Twilio Meet
katılımı şu sırayla günlüğe kaydedilir:

- Google Meet, Twilio katılımını Voice Call'a devreder.
- Voice Call, bağlantı öncesi DTMF TwiML'ini depolar.
- Twilio başlangıç TwiML'i tüketilir ve gerçek zamanlı işlemeden önce sunulur.
- Voice Call, Twilio çağrısı için gerçek zamanlı TwiML sunar.
- Google Meet, DTMF sonrası gecikmenin ardından `voicecall.speak` ile giriş konuşmasını ister.

`openclaw voicecall tail` kalıcı çağrı kayıtlarını göstermeye devam eder; çağrı durumu ve
dökümler için kullanışlıdır, ancak her Webhook/gerçek zamanlı geçiş
orada görünmez.

### Gerçek zamanlı çağrıda konuşma yok

Yalnızca bir ses modunun etkinleştirildiğini doğrulayın: `realtime.enabled` ve
`streaming.enabled` aynı anda doğru olamaz.

Gerçek zamanlı Twilio/Telnyx çağrıları için ayrıca şunları doğrulayın:

- Bir gerçek zamanlı sağlayıcı Plugin'i yüklenmiş ve kaydedilmiş.
- `realtime.provider` ayarlanmamış veya kayıtlı bir sağlayıcıyı adlandırıyor.
- Sağlayıcı API anahtarı Gateway işlemi tarafından kullanılabiliyor.
- `openclaw logs --follow`, gerçek zamanlı TwiML'in sunulduğunu, gerçek zamanlı köprünün başladığını ve başlangıç selamlamasının kuyruğa alındığını gösteriyor.

## İlgili

- [Konuşma modu](/tr/nodes/talk)
- [Metinden konuşmaya](/tr/tools/tts)
- [Sesle uyandırma](/tr/nodes/voicewake)
