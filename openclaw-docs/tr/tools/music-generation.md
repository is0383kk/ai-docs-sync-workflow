---
read_when:
    - Aracı aracılığıyla müzik veya ses oluşturma
    - Müzik üretimi sağlayıcılarını ve modellerini yapılandırma
    - music_generate aracı parametrelerini anlama
sidebarTitle: Music generation
summary: ComfyUI, fal, Google Lyria, MiniMax ve OpenRouter iş akışlarında music_generate aracılığıyla müzik oluşturun
title: Müzik üretimi
x-i18n:
    generated_at: "2026-07-26T23:43:19Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 3f2a8a4a36e47839c7896046a556f7bf84f6c168492e2de46736635fe2a9358e
    source_path: tools/music-generation.md
    workflow: 16
---

`music_generate` aracı, ComfyUI, fal, Google, MiniMax ve OpenRouter tarafından
desteklenen ortak müzik oluşturma özelliği aracılığıyla müzik veya ses oluşturur.

<Note>
`music_generate` yalnızca en az bir müzik oluşturma sağlayıcısı kullanılabilir
olduğunda görünür: açık bir `agents.defaults.mediaModels.music` yapılandırması veya kimlik
doğrulaması yapılandırılmış bir sağlayıcı (örneğin ayarlanmış bir API anahtarı).
</Note>

Oturum destekli ajan çalıştırmalarında `music_generate` bir arka plan görevi
olarak başlar, görev defterindeki ilerlemeyi izler ve ardından parça hazır
olduğunda ajanı uyandırarak kullanıcıyı bilgilendirmesini ve tamamlanan sesi
eklemesini sağlar. Tamamlama ajanı, oturumun görünür yanıt sözleşmesine uyar:
yapılandırıldığında otomatik son yanıt veya oturum ileti aracını gerektirdiğinde
`message(action="send")`. İstekte bulunan oturum etkin değilse ya da uyandırma işlemi
başarısız olursa ve oluşturulan ses hâlâ yanıtta eksikse OpenClaw, yalnızca eksik
sesi içeren eş etkili bir doğrudan geri dönüş gönderir.

## Hızlı başlangıç

<Tabs>
  <Tab title="Ortak sağlayıcı destekli">
    <Steps>
      <Step title="Kimlik doğrulamasını yapılandırın">
        En az bir sağlayıcı için bir API anahtarı ayarlayın — örneğin
        `GEMINI_API_KEY` veya `MINIMAX_API_KEY`.
      </Step>
      <Step title="Varsayılan bir model seçin (isteğe bağlı)">
        ```json5
        {
          agents: {
            defaults: {
              musicGenerationModel: {
                primary: "google/lyria-3-clip-preview",
              },
            },
          },
        }
        ```
      </Step>
      <Step title="Ajandan isteyin">
        _"Neon bir şehirde gece sürüşü hakkında hareketli bir synthpop parçası
        oluştur."_

        Ajan, `music_generate` aracını otomatik olarak çağırır. Araç için
        izin listesi gerekmez.
      </Step>
    </Steps>

    Oturum destekli bir ajan çalıştırması olmadan (doğrudan/yerel bağlamlarda)
    araç satır içi çalışır ve son medya yolunu aynı araç sonucunda döndürür.

  </Tab>
  <Tab title="ComfyUI iş akışı">
    <Steps>
      <Step title="İş akışını yapılandırın">
        `plugins.entries.comfy.config.music` öğesini bir iş akışı JSON'u ve
        istem/çıktı düğümleriyle yapılandırın.
      </Step>
      <Step title="Bulut kimlik doğrulaması (isteğe bağlı)">
        Comfy Cloud için `COMFY_API_KEY` veya `COMFY_CLOUD_API_KEY` ayarlayın.
      </Step>
      <Step title="Aracı çağırın">
        ```text
        /tool music_generate prompt="Yumuşak bant dokusuna sahip sıcak ortam synth döngüsü"
        ```
      </Step>
    </Steps>
  </Tab>
</Tabs>

Örnek istemler:

```text
Yumuşak yaylılar içeren ve vokal içermeyen sinematik bir piyano parçası oluştur.
```

```text
Gün doğumunda bir roket fırlatma hakkında enerjik bir chiptune döngüsü oluştur.
```

Kullanılabilir sağlayıcıları/modelleri incelemek için `action: "list"`, etkin
oturum destekli müzik görevini incelemek için ise `action: "status"` kullanın:

```text
/tool music_generate action=list
/tool music_generate action=status
```

Doğrudan oluşturma örneği:

```text
/tool music_generate prompt="Plak dokusu ve hafif yağmur içeren rüya gibi lo-fi hip hop" instrumental=true
```

## Desteklenen sağlayıcılar

| Sağlayıcı  | Varsayılan model              | Referans girdileri | Desteklenen denetimler                                | Kimlik doğrulama                       |
| ---------- | ---------------------------- | ------------------ | ----------------------------------------------------- | -------------------------------------- |
| ComfyUI    | `workflow`                   | En fazla 1 görsel  | İş akışıyla tanımlanan müzik veya ses                  | `COMFY_API_KEY`, `COMFY_CLOUD_API_KEY` |
| fal        | `fal-ai/minimax-music/v2.6`  | Yok                | `lyrics`, `instrumental`, `durationSeconds`, `format` | `FAL_KEY` veya `FAL_API_KEY`            |
| Google     | `lyria-3-clip-preview`       | En fazla 10 görsel | `lyrics`, `instrumental`, `format`                    | `GEMINI_API_KEY`, `GOOGLE_API_KEY` |
| MiniMax    | `music-2.6`                  | Yok                | `lyrics`, `instrumental`, `format` (yalnızca mp3)     | `MINIMAX_API_KEY` veya MiniMax OAuth  |
| OpenRouter | `google/lyria-3-pro-preview` | En fazla 1 görsel  | `lyrics`, `instrumental`, `durationSeconds`, `format` | `OPENROUTER_API_KEY`                     |

MiniMax, aynı modelleri paylaşan iki sağlayıcı kimliği kaydeder: API anahtarıyla
kimlik doğrulama için `minimax`, OAuth için `minimax-portal`. Model
referansları kimlik doğrulama yolunu izler (`minimax/music-2.6` ile
`minimax-portal/music-2.6` karşılaştırması); bkz.
[MiniMax](/tr/providers/minimax#music-generation).

fal ayrıca varsayılan MiniMax destekli modelinin yanında `fal-ai/ace-step/prompt-to-audio`
(wav, şarkı sözü yok, enstrümantal geçiş düğmesi yok) ve
`fal-ai/stable-audio-25/text-to-audio` (wav, yalnızca istem) seçeneklerini de sunar. Google'ın
varsayılan `lyria-3-clip-preview` modeli yalnızca mp3 çıktısı verir;
`lyria-3-pro-preview` ise wav biçimini de destekler. MiniMax ayrıca
`music-2.6-free`, `music-cover` ve `music-cover-free` seçeneklerini
sunar. OpenRouter ayrıca `google/lyria-3-clip-preview` seçeneğini sunar.

### Yetenek matrisi

`music_generate`, sözleşme testleri ve ortak canlı tarama tarafından kullanılan
açık kip sözleşmesi:

| Sağlayıcı  | `generate` | `edit` | Düzenleme sınırı | Ortak canlı hatlar                                                       |
| ---------- | :--------: | :----: | ----------------- | ----------------------------------------------------------------------- |
| ComfyUI    |     ✓      |   ✓    | 1 görsel          | Ortak taramada yer almaz; `extensions/comfy/comfy.live.test.ts` kapsamındadır |
| fal        |     ✓      |   —    | Yok               | `generate`                                                      |
| Google     |     ✓      |   ✓    | 10 görsel         | `generate`, `edit`                                  |
| MiniMax    |     ✓      |   —    | Yok               | `generate`                                                      |
| OpenRouter |     ✓      |   ✓    | 1 görsel          | `generate`, `edit`                                  |

## Araç parametreleri

<ParamField path="prompt" type="string" required>
  Müzik oluşturma istemi. `action: "generate"` için gereklidir.
</ParamField>
<ParamField path="action" type='"generate" | "status" | "list"' default="generate">
  `"status"` geçerli oturum görevini döndürür; `"list"`
  sağlayıcıları inceler.
</ParamField>
<ParamField path="model" type="string">
  Sağlayıcı/model geçersiz kılma değeri (ör. `google/lyria-3-pro-preview`,
  `comfy/workflow`).
</ParamField>
<ParamField path="lyrics" type="string">
  Sağlayıcı açık şarkı sözü girdisini desteklediğinde isteğe bağlı şarkı sözleri.
</ParamField>
<ParamField path="instrumental" type="boolean">
  Sağlayıcı desteklediğinde yalnızca enstrümantal çıktı isteyin.
</ParamField>
<ParamField path="image" type="string">
  Tek referans görsel yolu veya URL'si.
</ParamField>
<ParamField path="images" type="string[]">
  Birden fazla referans görsel (destekleyen sağlayıcılarda en fazla 10).
</ParamField>
<ParamField path="durationSeconds" type="number">
  Sağlayıcı süre ipuçlarını desteklediğinde saniye cinsinden hedef süre.
</ParamField>
<ParamField path="format" type='"mp3" | "wav"'>
  Sağlayıcı desteklediğinde çıktı biçimi ipucu.
</ParamField>
<ParamField path="filename" type="string">Çıktı dosya adı ipucu.</ParamField>

<Note>
Tüm sağlayıcılar tüm parametreleri desteklemez. OpenClaw, gönderimden önce girdi
sayıları gibi kesin sınırları yine de doğrular. Bir sağlayıcı süreyi destekliyor
ancak istenen değerden daha kısa bir üst sınır kullanıyorsa OpenClaw değeri en
yakın desteklenen süreye sınırlar. Seçilen sağlayıcı veya model gerçekten
desteklenmeyen isteğe bağlı ipuçlarını yerine getiremiyorsa bunlar bir uyarıyla
yok sayılır. Araç sonuçları uygulanan ayarları bildirir; `details.normalization`,
istenen değer ile uygulanan değer arasındaki tüm eşlemeleri yakalar.
</Note>

Sağlayıcı istek zaman aşımları yalnızca operatör yapılandırmasıdır. OpenClaw,
yapılandırıldığında `agents.defaults.mediaModels.music.timeoutMs` kullanır, 120000ms altındaki değerleri
120000ms değerine yükseltir ve aksi durumda sağlayıcı istekleri için varsayılan
olarak 300000ms kullanır.

## Eşzamansız davranış

Oturum destekli müzik oluşturma işlemi bir arka plan görevi olarak çalışır:

- **Arka plan görevi:** `music_generate` bir arka plan görevi oluşturur, başlatıldı/görev
  yanıtını hemen döndürür ve tamamlanan parçayı daha sonra bir takip ajanı
  iletisinde gönderir.
- **Yinelenenleri önleme:** Bir görev `queued` veya `running`
  durumundayken aynı oturumdaki sonraki `music_generate` çağrıları başka bir
  oluşturma başlatmak yerine görev durumunu döndürür. Açıkça denetlemek için
  `action: "status"` kullanın. Yakın zamanda tamamlanmış eşleşen bir istek de
  2 dakika boyunca yinelenen istek olarak elenir.
- **Durum sorgulama:** `openclaw tasks list` veya `openclaw tasks show <taskId>`,
  kuyruğa alınmış, çalışan ve sonlanmış durumları inceler.
- **Tamamlama uyandırması:** OpenClaw, modelin kullanıcıya yönelik takip iletisini
  kendisinin yazabilmesi için aynı oturuma dahili bir tamamlama olayı ekler.
- **İstem ipucu:** Aynı oturumdaki sonraki kullanıcı/manuel turlar, bir müzik görevi
  zaten devam ediyorsa küçük bir çalışma zamanı ipucu alır; böylece model
  `music_generate` aracını düşünmeden tekrar çağırmaz.
- **Oturumsuz geri dönüş:** Gerçek bir ajan oturumu olmayan doğrudan/yerel bağlamlar,
  işlemi satır içi çalıştırır ve son ses sonucunu aynı turda döndürür.

### Görev yaşam döngüsü

Müzik görevi, genel görev kayıt defteriyle aynı durumları sunar
(`timed_out`, `cancelled` ve `lost` dâhil tam durum
makinesi için bkz. [Arka plan görevleri](/tr/automation/tasks#task-lifecycle)).
Çoğu müzik çalıştırması şu durumlardan geçer:

| Durum       | Anlamı                                                                                         |
| ----------- | ---------------------------------------------------------------------------------------------- |
| `queued`    | Görev oluşturuldu; sağlayıcının kabul etmesi bekleniyor.                                       |
| `running`   | Sağlayıcı işliyor (sağlayıcıya ve süreye bağlı olarak genellikle 30 saniye ile 3 dakika arası). |
| `succeeded` | Parça hazır; ajan uyanır ve parçayı konuşmaya gönderir.                                        |
| `failed`    | Sağlayıcı hatası veya zaman aşımı; ajan hata ayrıntılarıyla uyanır.                             |

Durumu CLI üzerinden denetleyin:

```bash
openclaw tasks list
openclaw tasks show <taskId>
openclaw tasks cancel <taskId>
```

## Yapılandırma

### Model seçimi

```json5
{
  agents: {
    defaults: {
      musicGenerationModel: {
        primary: "google/lyria-3-clip-preview",
        fallbacks: ["fal/fal-ai/minimax-music/v2.6", "minimax/music-2.6"],
      },
    },
  },
}
```

### Sağlayıcı seçim sırası

OpenClaw sağlayıcıları şu sırayla dener:

1. Araç çağrısındaki `model` parametresi (ajan bir tane belirtirse).
2. Yapılandırmadaki `musicGenerationModel.primary`.
3. Sırayla `musicGenerationModel.fallbacks`.
4. Yalnızca kimlik doğrulama destekli sağlayıcı varsayılanlarını kullanan otomatik algılama:
   - müzik oluşturma da sunuyorsa önce geçerli varsayılan metin modeli
     sağlayıcısı;
   - ardından kalan kayıtlı müzik oluşturma sağlayıcıları, sağlayıcı
     kimliğine göre alfabetik sırayla.

Bir sağlayıcı başarısız olursa sonraki aday otomatik olarak denenir. Tümü başarısız
olursa hata, her denemenin ayrıntılarını içerir.

Kimliği doğrulanmış sağlayıcılar arasında otomatik geri dönüş her zaman etkindir.
Çağrı başına `model` belirleyici olmaya devam eder.

## Sağlayıcı notları

<AccordionGroup>
  <Accordion title="ComfyUI">
    İş akışı odaklıdır ve yapılandırılmış grafiğin yanı sıra istem/çıktı
    alanlarının node eşlemesine bağlıdır. Birlikte sunulan `comfy` plugin'i,
    müzik oluşturma sağlayıcı kayıt defteri üzerinden paylaşılan
    `music_generate` aracına bağlanır.
  </Accordion>
  <Accordion title="fal">
    Paylaşılan sağlayıcı kimlik doğrulama yolu üzerinden fal model uç noktalarını kullanır.
    Birlikte sunulan sağlayıcı varsayılan olarak `fal-ai/minimax-music/v2.6` kullanır ve istemden sese
    istekleri için `fal-ai/ace-step/prompt-to-audio` ile
    `fal-ai/stable-audio-25/text-to-audio` modellerini de sunar.
    Şarkı sözleri ve enstrümantal mod yalnızca MiniMax modellerinde kullanılabilir; diğer iki
    model yalnızca istem destekler.
  </Accordion>
  <Accordion title="Google (Lyria 3)">
    Lyria 3 toplu oluşturmayı kullanır. Birlikte sunulan mevcut akış;
    istemi, isteğe bağlı şarkı sözü metnini ve isteğe bağlı referans görsellerini destekler.
    Varsayılan `lyria-3-clip-preview` modeli yalnızca mp3 çıktısı verir;
    `lyria-3-pro-preview` modeli ayrıca wav biçimini destekler.
  </Accordion>
  <Accordion title="MiniMax">
    Toplu `music_generation` uç noktasını kullanır. İstemi, isteğe bağlı
    şarkı sözlerini, enstrümantal modu ve `minimax` API anahtarıyla kimlik doğrulama
    veya `minimax-portal` OAuth üzerinden mp3 çıktısını destekler. Ayrıca `music-2.6-free`,
    `music-cover` ve `music-cover-free` modellerini sunar.
  </Accordion>
  <Accordion title="OpenRouter">
    Akış etkinleştirilmiş şekilde OpenRouter sohbet tamamlama ses çıktısını kullanır.
    Birlikte sunulan sağlayıcı varsayılan olarak `google/lyria-3-pro-preview` kullanır ve ayrıca
    `openrouter/google/lyria-3-clip-preview` modelini sunar.
  </Accordion>
</AccordionGroup>

## Doğru yolu seçme

- Model seçimi, sağlayıcı yük devretme ve yerleşik eşzamansız
  görev/durum akışı istediğinizde **paylaşılan sağlayıcı destekli** yolu kullanın.
- Özel bir iş akışı grafiğine veya paylaşılan ve birlikte sunulan müzik
  özelliğinin parçası olmayan bir sağlayıcıya ihtiyaç duyduğunuzda **Plugin yolu (ComfyUI)** kullanın.

ComfyUI'ye özgü davranışlarda hata ayıklıyorsanız
[ComfyUI](/tr/providers/comfy) sayfasına bakın. Paylaşılan sağlayıcı davranışında hata
ayıklıyorsanız [fal](/tr/providers/fal), [Google (Gemini)](/tr/providers/google),
[MiniMax](/tr/providers/minimax) veya [OpenRouter](/tr/providers/openrouter) ile başlayın.

## Sağlayıcı yetenek modları

Paylaşılan müzik oluşturma sözleşmesi açık mod bildirimlerini destekler:

- Yalnızca istemle oluşturma için `generate`.
- İstek bir veya daha fazla referans görseli içerdiğinde `edit`.

Yeni sağlayıcı uygulamaları açık mod bloklarını tercih etmelidir:

```typescript
capabilities: {
  generate: {
    maxTracks: 1,
    supportsLyrics: true,
    supportsFormat: true,
  },
  edit: {
    enabled: true,
    maxTracks: 1,
    maxInputImages: 1,
    supportsFormat: true,
  },
}
```

`maxInputImages`, `supportsLyrics` ve
`supportsFormat` gibi eski düz alanlar, düzenleme desteğini duyurmak için **yeterli değildir**.
Sağlayıcılar; canlı testlerin, sözleşme testlerinin ve paylaşılan `music_generate`
aracının mod desteğini belirlenimci biçimde doğrulayabilmesi için
`generate` ve `edit` değerlerini açıkça bildirmelidir.

## Canlı testler

Paylaşılan ve birlikte sunulan sağlayıcılar (fal, Google, MiniMax,
OpenRouter) için isteğe bağlı canlı kapsam:

```bash
OPENCLAW_LIVE_TEST=1 pnpm test:live -- extensions/music-generation-providers.live.test.ts
```

Aynı test dosyasını çalıştıran eşdeğer depo sarmalayıcısı:

```bash
pnpm test:live:media:music
```

Bu canlı dosya, varsayılan olarak saklanan kimlik doğrulama profillerinden önce
önceden dışa aktarılmış sağlayıcı ortam değişkenlerini kullanır ve sağlayıcı düzenleme
modunu etkinleştirdiğinde hem `generate` hem de bildirilmiş `edit`
kapsamını çalıştırır. Güncel kapsam:

- `google`: `generate` ve `edit`
- `fal`: yalnızca `generate`
- `minimax`: yalnızca `generate`
- `openrouter`: `generate` ve `edit`
- `comfy`: paylaşılan sağlayıcı taramasının parçası olmayan ayrı Comfy canlı kapsamı

Birlikte sunulan ComfyUI müzik yolu için isteğe bağlı canlı kapsam:

```bash
OPENCLAW_LIVE_TEST=1 COMFY_LIVE_TEST=1 pnpm test:live -- extensions/comfy/comfy.live.test.ts
```

Comfy canlı dosyası, ilgili bölümler yapılandırıldığında Comfy görsel ve video
iş akışlarını da kapsar.

## İlgili

- [Arka plan görevleri](/tr/automation/tasks) — ayrılmış `music_generate` çalıştırmaları için görev izleme
- [ComfyUI](/tr/providers/comfy)
- [Yapılandırma referansı](/tr/gateway/config-agents#agent-defaults) — `musicGenerationModel` yapılandırması
- [Google (Gemini)](/tr/providers/google)
- [MiniMax](/tr/providers/minimax)
- [Modeller](/tr/concepts/models) — model yapılandırması ve yük devretme
- [Araçlara genel bakış](/tr/tools)
