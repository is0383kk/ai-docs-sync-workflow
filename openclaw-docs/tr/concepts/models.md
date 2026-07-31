---
read_when:
    - Model geri dönüş davranışını veya seçim kullanıcı deneyimini değiştirme
    - '"modele izin verilmiyor" veya güncelliğini yitirmiş varsayılan sağlayıcı geri dönüşünde hata ayıklama'
    - models.json birleştirme/gizli bilgi davranışı üzerinde çalışma
sidebarTitle: Models CLI
summary: OpenClaw'un sağlayıcı/model referanslarını, yapılandırma anahtarlarını ve `/model` sohbet komutunu nasıl çözümlediği
title: Modeller CLI'si
x-i18n:
    generated_at: "2026-07-26T23:18:26Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 2cd13a2aae6575bdfeefb477b7fe8be740b77c66cb76454b07d82481f6612152
    source_path: concepts/models.md
    workflow: 16
---

<CardGroup cols={2}>
  <Card title="Model yük devretme" href="/tr/concepts/model-failover">
    Kimlik doğrulama profili rotasyonu, bekleme süreleri ve bunların geri dönüşlerle nasıl etkileşime girdiği.
  </Card>
  <Card title="Model sağlayıcıları" href="/tr/concepts/model-providers">
    Sağlayıcılara hızlı genel bakış ve örnekler.
  </Card>
  <Card title="Models CLI başvurusu" href="/tr/cli/models">
    Eksiksiz `openclaw models` komut ve bayrak başvurusu.
  </Card>
  <Card title="Yapılandırma başvurusu" href="/tr/gateway/config-agents#agent-defaults">
    Model yapılandırma anahtarları, varsayılanlar ve örnekler.
  </Card>
</CardGroup>

Bir model başvurusu (`provider/model`), düşük seviyeli agent çalışma zamanını değil, bir sağlayıcı ve model seçer. Çalışma zamanı ilkesi ayarlanmamışken veya `auto` iken, OpenAI'ın sağlayıcıya ait rota ilkesi yalnızca yazılmış bir istek geçersiz kılması bulunmayan, tam olarak resmî HTTPS Platform Responses veya ChatGPT Responses rotası için Codex'i seçebilir; yalnızca `openai/*` ön eki Codex'i hiçbir zaman seçmez. Completions bağdaştırıcıları, özel uç noktalar ve yazılmış istek davranışı OpenClaw üzerinde kalır. Düz metin kullanan resmî HTTP uç noktaları reddedilir. Bkz. [OpenAI örtük agent çalışma zamanı](/tr/providers/openai#implicit-agent-runtime).

Abonelik Copilot başvuruları (`github-copilot/*`) haricî GitHub Copilot agent çalışma zamanı Plugin'ine dâhil edilebilir, ancak bu yol her zaman açıkça belirtilir (`auto` tarafından hiçbir zaman seçilmez). Çalışma zamanı geçersiz kılmaları, agent'ın veya oturumun tamamında değil, sağlayıcı/model ilkesinde yer alır. Çalışma zamanı seçimi faturalandırmayı belirlemez: OpenAI API anahtarı ile ChatGPT/Codex abonelik kimlik bilgileri birbirinden ayrı kalır. Bkz. [Agent çalışma zamanları](/tr/concepts/agent-runtimes) ve [GitHub Copilot agent çalışma zamanı](/tr/plugins/copilot).

## Seçim sırası

<Steps>
  <Step title="Birincil model">
    `agents.defaults.model.primary` (veya düz dize olarak `agents.defaults.model`).
  </Step>
  <Step title="Geri dönüşler">
    `agents.defaults.model.fallbacks`, sırayla denenir.
  </Step>
  <Step title="Kimlik doğrulama yük devretmesi">
    OpenClaw bir sonraki geri dönüş modeline geçmeden önce sağlayıcı içinde kimlik doğrulama profili rotasyonu gerçekleşir.
  </Step>
</Steps>

İlgili model yapılandırma yüzeyleri:

- `agents.defaults.models`, takma adları ve model başına ayarları depolar. Bir girdi eklemek model geçersiz kılmalarını kısıtlamaz.
- `agents.defaults.modelPolicy.allow`, isteğe bağlı geçersiz kılma izin listesidir. Tam başvuruları veya `provider/*` ve `provider/namespace/*` gibi sonda yer alan ön ek joker karakterlerini kullanın; herhangi bir modele izin vermek için bunu atlayın veya `[]` olarak ayarlayın. Agent başına `agents.entries.*.modelPolicy.allow`, söz konusu agent için varsayılan ilkenin yerini alır.
- `agents.defaults.utilityModel`; oluşturulan pano oturumu başlıkları, desteklenen kanal ileti dizisi/konu başlıkları ve ilerleme anlatımı gibi kısa dâhilî görevler için isteğe bağlı, daha düşük maliyetli bir modeldir. Agent başına `agents.entries.*.utilityModel` bunu geçersiz kılar. Ayarlanmadığında OpenClaw, varsa birincil sağlayıcının bildirdiği küçük model varsayılanını (OpenAI → `gpt-5.6-luna`, Anthropic → `claude-haiku-4-5`), aksi takdirde agent'ın birincil modelini kullanır; yardımcı yönlendirmeyi devre dışı bırakmak için bunu boş bir dizeye ayarlayın. Farklı bir yardımcı model başarısız olduğunda oluşturulan başlıklar birincil modelle bir kez daha denenir. Pano başlıklarında otomatik yardımcı türetme ve normal geri dönüş, etkin oturum sağlayıcısını ve kimlik doğrulama profilini izler; açıkça belirtilen yardımcı model ise yapılandırılmış sağlayıcısını/kimlik doğrulamasını korur. Boş bir yardımcı model, pano başlığı oluşturmayı değil, yalnızca alternatif küçük model rotasını atlar. Yardımcı görevler ayrı model çağrılarıdır ve seçilen model sağlayıcısına sınırlandırılmış görev içeriği gönderebilir.
- `agents.defaults.imageModel`, yalnızca birincil model görüntüleri kabul edemediğinde kullanılır.
- `agents.defaults.pdfModel`, `pdf` aracı tarafından kullanılır. Ayarlanmamışsa araç önce `imageModel`, ardından çözümlenen oturum/varsayılan modele geri döner.
- `agents.defaults.mediaModels.{image,music,video}`, paylaşılan medya oluşturma araçlarını destekler. Ayarlanmamışsa her araç, kimlik doğrulama destekli bir sağlayıcı varsayılanını çıkarır: önce geçerli varsayılan sağlayıcı, ardından bu yetenek için kayıtlı kalan sağlayıcılar sağlayıcı kimliği sırasıyla kullanılır. Sağlayıcılar arası geri dönüş, sabit varsayılan davranıştır.
- Agent başına `agents.entries.*.model` (ve bağlamalar), `agents.defaults.model` değerini geçersiz kılar — bkz. [Çok agent'lı yönlendirme](/tr/concepts/multi-agent).

Anahtarların tam başvurusu, varsayılanlar ve JSON5 örnekleri: [Yapılandırma başvurusu](/tr/gateway/config-agents#agent-defaults).

## Seçim kaynağı ve geri dönüş katılığı

Aynı `provider/model`, nereden geldiğine bağlı olarak farklı davranır:

| Kaynak                                                                  | Davranış                                                                                                                                                                                                                                                       |
| ----------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Yapılandırılmış varsayılan (`agents.defaults.model.primary`, agent başına birincil) | Normal başlangıç noktasıdır; `agents.defaults.model.fallbacks` kullanır.                                                                                                                                                                                                 |
| Otomatik geri dönüş                                                           | `modelOverrideSource: "auto"` olarak depolanan geçici kurtarma durumu. OpenClaw, özgün birincil modeli düzenli olarak yeniden yoklar, kurtarma gerçekleştiğinde otomatik seçimi temizler ve her durum değişikliğinde geri dönüş/kurtarma geçişlerini bir kez duyurur.                              |
| Kullanıcı oturumu seçimi                                                  | Tam ve katıdır. `/model`, model seçici, `session_status(model=...)` ve `sessions.patch`, `modelOverrideSource: "user"` değerini depolar. Bu sağlayıcıya/modele erişilemezse çalışma başka bir yapılandırılmış modele geçmek yerine görünür biçimde başarısız olur. |
| Cron `--model` / yük `model`                                        | İş başına birincil. İş kendi yükünde `fallbacks` sağlamadığı sürece yapılandırılmış geri dönüşleri kullanmaya devam eder (`fallbacks: []` katı bir çalışmayı zorunlu kılar).                                                                                                                    |

Diğer seçim kuralları:

- `agents.defaults.model.primary` değerini değiştirmek mevcut oturum sabitlemelerini yeniden yazmaz. Durum `This session is pinned to X; config primary Y will apply to new/unpinned sessions.` bildiriyorsa sabitlemeyi temizlemek için `/model default` çalıştırın.
- CLI varsayılan model ve izin listesi seçicileri, yerleşik kataloğun tamamı yerine yalnızca `models.providers.*.models` öğesini listeleyerek `models.mode: "replace"` değerine uyar.
- Control UI model seçici, yapılandırılmış model görünümünü Gateway'den ister. Açıkça belirtilmiş bir `modelPolicy.allow`, sonda yer alan ön ek joker karakteri girdileri de dâhil olmak üzere bunu filtreler; aksi takdirde yapılandırılmış modelleri ve kullanılabilir kimlik doğrulaması olan sağlayıcıları gösterir. Yerleşik kataloğun tamamı, açık gezinme görünümlerine ayrılmıştır (`models.list` ile `view: "all"` veya `openclaw models list --all`).
- Sağlayıcı envanteri kullanıcı arayüzleri, seçici izin listelerini uygulamadan kaynakta tanımlanmış `models.providers.*.models` satırlarını göstermek için `models.list` ile `view: "provider-config"` kullanır.

Tüm işleyiş: [Model yük devretme](/tr/concepts/model-failover).

## Hızlı model ilkesi

- Birincil modelinizi erişebildiğiniz en güçlü, en yeni nesil modele ayarlayın.
- Maliyet/gecikme açısından hassas görevler ve daha düşük önem düzeyindeki sohbetler için geri dönüşleri kullanın.
- Araç etkin agent'lar veya güvenilmeyen girdiler için eski/zayıf model katmanlarından kaçının.

## İlk katılım

```bash
openclaw onboard
```

OpenAI Codex aboneliği OAuth'ı ve Anthropic (API anahtarı veya Claude CLI'ı yeniden kullanma) dâhil olmak üzere yaygın sağlayıcılar için yapılandırmayı elle düzenlemeye gerek kalmadan model ve kimlik doğrulamasını ayarlar.

Yapılandırılmış bir birincil model yokken yeni OpenAI API anahtarı kurulumu
`openai/gpt-5.6` öğesini seçer; yalın doğrudan API kimliği Sol katmanına çözümlenir. Yeni
ChatGPT/Codex OAuth kurulumu tam `openai/gpt-5.6-sol` katalog başvurusunu seçer.
Yeniden kimlik doğrulama, `openai/gpt-5.5` dâhil olmak üzere mevcut açıkça belirtilmiş birincil modeli korur.
GPT-5.6 hesap tarafından kullanılamıyorsa `openai/gpt-5.5` öğesini açıkça seçin;
OpenClaw bunu sessizce alt sürüme düşürmez.

## "Modele izin verilmiyor" (ve yanıtların neden durduğu)

`agents.defaults.modelPolicy.allow` boş değilse `/model`, oturum geçersiz kılmaları ve `--model` için izin listesi hâline gelir. Bu izin listesinin dışındaki bir modelin seçilmesi, normal bir yanıt oluşturulmadan önce sonuç döndürür. Agent başına `agents.entries.*.modelPolicy.allow`, söz konusu agent için varsayılan ilkenin yerini alır.

```text
"provider/model" model geçersiz kılmasına agents.defaults.modelPolicy.allow tarafından izin verilmiyor.
agents.defaults.modelPolicy.allow öğesine "provider/model", "provider/*" veya daha dar bir "provider/namespace/*" ön eki ekleyin ya da herhangi bir modele izin vermek için listeyi kaldırın/boşaltın.
```

Modeli veya bir sağlayıcı joker karakterini adı belirtilen `modelPolicy.allow` anahtarına ekleyerek, bu listeyi kaldırarak/boşaltarak ya da `/model list` içinden bir model seçerek sorunu düzeltin. Reddedilen komut `/model openai/gpt-5.5 --runtime codex` gibi bir çalışma zamanı geçersiz kılması içeriyorsa önce izin listesini düzeltin, ardından aynı komutu yeniden deneyin.

Yerel/GGUF modellerinde izin listesi, örneğin `ollama/gemma4:26b` veya `lmstudio/Gemma4-26b-a4-it-gguf` gibi sağlayıcı ön ekli tam başvuruyu gerektirir — tam dize için `openclaw models list --provider <provider>` öğesini kontrol edin. İzin listesi etkin olduğunda yalın dosya adları veya görünen adlar yeterli değildir.

Her modeli listelemeden sağlayıcıları sınırlamak için sonda yer alan ön ek joker karakteri girdilerini kullanın. Sağlayıcı genelindeki `provider/*`, o sağlayıcının altındaki her modelle eşleşir; `clawrouter/anthropic/*` gibi daha dar bir ön ek yalnızca o ad alanıyla eşleşir:

```json5
{
  agents: {
    defaults: {
      modelPolicy: {
        allow: ["openai/*", "vllm/*"],
      },
    },
  },
}
```

`/model`, `/models` ve model seçiciler daha sonra yalnızca bu sağlayıcılar için keşfedilen kataloğu gösterir ve izin listesi düzenlenmeden yeni modeller görünebilir. Başka bir sağlayıcıdan belirli bir modeli dâhil etmek için tam `provider/model` girdilerini `provider/*` girdileriyle karıştırın.

Takma adlar ve model başına ayarlar içeren örnek izin listesi:

```json5
{
  agents: {
    defaults: {
      model: { primary: "anthropic/claude-sonnet-4-6" },
      modelPolicy: {
        allow: ["anthropic/claude-sonnet-4-6", "anthropic/claude-opus-4-6"],
      },
      models: {
        "anthropic/claude-sonnet-4-6": { alias: "Sonnet" },
        "anthropic/claude-opus-4-6": { alias: "Opus" },
      },
    },
  },
}
```

<Accordion title="İzin listesini açıkça düzenleyin">
Tam listeyi doğrudan ayarlayın:

```bash
openclaw config set agents.defaults.modelPolicy.allow '["openai/gpt-5.4","anthropic/*"]' --strict-json
```

`openclaw models set`, sağlayıcı kurulumu ve `openclaw models aliases add`, `agents.defaults.models` altına girdiler ekleyebilir, ancak `modelPolicy.allow` değerini hiçbir zaman değiştirmez. Bu, model meta verilerini ve takma adları geçersiz kılma ilkesinden bağımsız tutar.
</Accordion>

## Sohbette `/model`

```text
/model
/model list
/model 3
/model openai/gpt-5.4
/model default
/model status
```

- `/model` ve `/model list`, kompakt bir numaralı seçici (model ailesi + kullanılabilir sağlayıcılar) gösterir; `/model <#>` buradan seçim yapar. Discord'da bu, bir Submit adımıyla sağlayıcı/model açılır listelerini açar; Telegram'da seçici seçimleri oturum kapsamındadır ve aracının `openclaw.json` içindeki kalıcı varsayılanını asla yeniden yazmaz. `/models add` kullanımdan kaldırılmıştır ve sohbetten model kaydetmek yerine bir mesaj döndürür.
- `/model`, yeni oturum seçimini hemen kalıcı hale getirir. Araç boştaysa sonraki çalıştırma bunu hemen kullanır; bir çalıştırma zaten etkinse geçiş, bir sonraki temiz yeniden deneme noktası için (veya araç etkinliği ya da yanıt çıktısı zaten başladıysa daha sonraki bir nokta için) kuyruğa alınır.
- `/model default`, yapılandırılmış birincili yeniden devralması için oturum seçimini temizler.
- Kullanıcı tarafından seçilen bir `/model` referansı, söz konusu oturum için katıdır: erişilemez hale gelirse yanıt, `agents.defaults.model.fallbacks` üzerinden sessizce geri dönüş yapmak yerine görünür biçimde başarısız olur. Yapılandırılmış varsayılanlar ve cron işi birincilleri geri dönüş zincirlerini kullanmaya devam eder.
- `/model status` ayrıntılı görünümdür: sağlayıcı başına kimlik doğrulama adayları ve (yapılandırıldığında) sağlayıcı uç noktası `baseUrl` ile `api` modu.
- Model referansları ilk `/` üzerinden bölünerek ayrıştırılır; `provider/model` yazın. Model kimliğinin kendisi `/` içeriyorsa (OpenRouter tarzı), sağlayıcı ön ekini ekleyin; örneğin `/model openrouter/moonshotai/kimi-k2`. Sağlayıcıyı belirtmezseniz OpenClaw şunları dener: (1) takma ad eşleşmesi, (2) tam olarak bu ön eksiz model kimliği için benzersiz yapılandırılmış sağlayıcı eşleşmesi, (3) yapılandırılmış varsayılan sağlayıcı (kullanımdan kaldırılmış geri dönüş) — bu sağlayıcı yapılandırılmış varsayılan modeli artık sunmuyorsa, kaldırılmış sağlayıcıya ait eski bir varsayılanın gösterilmesini önlemek için bunun yerine ilk yapılandırılmış sağlayıcı/model.
- Model referansları küçük harfe normalleştirilir; sağlayıcı kimlikleri ise bunun dışında tam eşleşir, bu nedenle plugin tarafından bildirilen kimliği kullanın.

Komutların tam davranışı ve yapılandırma: [Eğik çizgi komutları](/tr/tools/slash-commands).

## CLI

```bash
openclaw models status
openclaw models list
openclaw models set <provider/model>
openclaw models set-image <provider/model>
openclaw models scan
openclaw models aliases list|add|remove
openclaw models fallbacks list|add|remove|clear
openclaw models image-fallbacks list|add|remove|clear
openclaw models auth list|add|login|paste-api-key|paste-token|setup-token|order
```

Alt komut olmadan `openclaw models`, `models status` için bir kısayoldur; bu komut ayrıca kimlik doğrulama deposu profillerinin OAuth süre sonunu gösterir (varsayılan olarak 24 saat içinde uyarır). Tüm bayraklar, JSON şekilleri ve kimlik doğrulama profili alt komutları: [Models CLI başvurusu](/tr/cli/models).

<AccordionGroup>
  <Accordion title="Tarama (OpenRouter ücretsiz modelleri)">
    `openclaw models scan`, OpenRouter'ın herkese açık ücretsiz model kataloğunu inceler ve adayların araç ve görüntü desteğini canlı olarak sınayabilir. Kataloğun kendisi herkese açık olduğundan yalnızca meta veri taramaları (`--no-probe`) anahtar gerektirmez; canlı sınama ve `--set-default`/`--set-image` bir OpenRouter API anahtarı (kimlik doğrulama profili veya `OPENROUTER_API_KEY`) gerektirir ve anahtar olmadığında yalnızca meta veri çıktısına kapalı biçimde geri döner.

    Sonuçlar şu ölçütlere göre sıralanır: görüntü desteği, ardından araç gecikmesi, bağlam boyutu ve parametre sayısı. TTY'de sınanan sonuçlar etkileşimli bir geri dönüş seçimi ister; etkileşimsiz modun varsayılanları kabul etmesi için `--yes` gerekir.

  </Accordion>
</AccordionGroup>

## Model kayıt defteri (`models.json`)

`models.providers` altında yapılandırılan özel sağlayıcılar, aracı dizini altındaki `models.json` dosyasına yazılır (varsayılan `~/.openclaw/agents/<agentId>/agent/models.json`). Sağlayıcı plugini katalogları, oluşturulan ve pluginin sahip olduğu ayrı katalog parçaları olarak depolanır ve otomatik olarak yüklenir. Bu dosya varsayılan olarak yapılandırmayla birleştirilir; yalnızca yapılandırdığınız sağlayıcıları kullanmak için `models.mode: "replace"` ayarlayın.

<AccordionGroup>
  <Accordion title="Birleştirme modu önceliği">
    Eşleşen sağlayıcı kimlikleri için:

    - Aracının `models.json` dosyasında zaten bulunan boş olmayan bir `baseUrl` üstün gelir.
    - `models.json` içindeki boş olmayan bir `apiKey`, yalnızca söz konusu sağlayıcı mevcut yapılandırma/kimlik doğrulama profili bağlamında SecretRef tarafından yönetilmiyorsa üstün gelir.
    - SecretRef tarafından yönetilen `apiKey` değerleri, çözümlenmiş gizli bilgileri kalıcı hale getirmek yerine kaynak işaretçilerinden yenilenir: ortam referansları için ortam değişkeni adı, dosya/çalıştırma referansları için `secretref-managed`.
    - SecretRef tarafından yönetilen üstbilgi değerleri de ortam referansları için `secretref-env:ENV_VAR_NAME` kullanılarak aynı şekilde yenilenir.
    - `models.json` içindeki boş veya eksik `apiKey`/`baseUrl`, yapılandırmadaki `models.providers` değerine geri döner.
    - Diğer sağlayıcı alanları, yapılandırmadan ve normalleştirilmiş katalog verilerinden yenilenir.

  </Accordion>
</AccordionGroup>

İşaretçilerin kalıcılaştırılmasında kaynak belirleyicidir: OpenClaw, `models.json` dosyasını her yeniden oluşturduğunda — `openclaw agent` gibi komutla yönlendirilen yollar dâhil — işaretçileri çözümlenmiş çalışma zamanı gizli değerlerinden değil, etkin kaynak yapılandırma anlık görüntüsünden (çözümleme öncesi) yazar.

## İlgili

- [Aracı çalışma zamanları](/tr/concepts/agent-runtimes) — OpenClaw, Codex ve diğer aracı döngüsü çalışma zamanları
- [Yapılandırma başvurusu](/tr/gateway/config-agents#agent-defaults) — model yapılandırma anahtarları
- [Görüntü oluşturma](/tr/tools/image-generation) — görüntü modeli yapılandırması
- [Model yük devretmesi](/tr/concepts/model-failover) — geri dönüş zincirleri
- [Model sağlayıcıları](/tr/concepts/model-providers) — sağlayıcı yönlendirmesi ve kimlik doğrulama
- [Models CLI başvurusu](/tr/cli/models) — tüm komut ve bayrak başvurusu
- [Müzik oluşturma](/tr/tools/music-generation) — müzik modeli yapılandırması
- [Video oluşturma](/tr/tools/video-generation) — video modeli yapılandırması
