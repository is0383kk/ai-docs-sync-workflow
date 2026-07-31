---
read_when:
    - Model seçme veya değiştirme, takma adları yapılandırma
    - Model yük devretme / "Tüm modeller başarısız oldu" hatalarını ayıklama
    - Kimlik doğrulama profillerini ve bunların nasıl yönetileceğini anlama
sidebarTitle: Models FAQ
summary: 'SSS: model varsayılanları, seçimi, takma adları, değiştirme, yük devretme ve kimlik doğrulama profilleri'
title: 'SSS: modeller ve kimlik doğrulama'
x-i18n:
    generated_at: "2026-07-27T00:01:25Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 0c46d99352c5e51af5917c6f62b897dfa4030cb0201b8235e28f2f81f2954544
    source_path: help/faq-models.md
    workflow: 16
---

Model ve kimlik doğrulama profili hakkında SSS. Kurulum, oturumlar, Gateway, kanallar ve
sorun giderme için ana [SSS](/tr/help/faq) sayfasına bakın.

## Modeller: varsayılanlar, seçim, takma adlar, değiştirme

<AccordionGroup>
  <Accordion title='"Varsayılan model" nedir?'>
    Şununla ayarlanır:

    ```text
    agents.defaults.model.primary
    ```

    Modeller `provider/model` referanslarıdır (örnek: `openai/gpt-5.5`,
    `anthropic/claude-sonnet-4-6`). `provider/model` değerini her zaman açıkça ayarlayın. Sağlayıcıyı
    belirtmezseniz OpenClaw önce bir takma ad eşleşmesi, ardından bu model kimliği
    için yapılandırılmış sağlayıcılar arasında benzersiz bir eşleşme arar ve son
    olarak yapılandırılmış varsayılan sağlayıcıya geri döner (kullanımdan kaldırılmış
    uyumluluk yolu). Bu sağlayıcı artık yapılandırılmış varsayılan modele sahip
    değilse OpenClaw eski bir varsayılanı kullanmak yerine yapılandırılmış ilk
    sağlayıcıya/modele geri döner.

  </Accordion>

  <Accordion title="Hangi modeli önerirsiniz?">
    Özellikle araçları kullanabilen veya güvenilmeyen girdiler alan aracılar için
    sağlayıcı yığınınızın sunduğu en güçlü ve en yeni nesil modeli kullanın —
    daha zayıf veya aşırı nicemlenmiş modeller istem enjeksiyonuna ve güvenli
    olmayan davranışlara karşı daha savunmasızdır (bkz. [Güvenlik](/tr/gateway/security)).
    Daha ucuz modelleri aracı rolüne göre rutin/düşük riskli sohbetlere yönlendirin.

    Modelleri aracı bazında yönlendirin ve uzun görevleri paralelleştirmek için alt
    aracılar kullanın (her alt aracı kendi token'larını tüketir). Bkz.
    [Modeller](/tr/concepts/models), [Alt aracılar](/tr/tools/subagents),
    [MiniMax](/tr/providers/minimax) ve [Yerel modeller](/tr/gateway/local-models).

  </Accordion>

  <Accordion title="Yapılandırmamı silmeden modelleri nasıl değiştirebilirim?">
    Yalnızca model alanlarını değiştirin; yapılandırmanın tamamını değiştirmekten kaçının.

    - sohbette `/model` (oturum bazında, bkz. [Eğik çizgi komutları](/tr/tools/slash-commands))
    - `openclaw models set ...` (yalnızca model yapılandırmasını günceller)
    - `openclaw configure --section model` (etkileşimli)
    - `~/.openclaw/openclaw.json` içindeki `agents.defaults.model` değerini doğrudan düzenleyin

    RPC düzenlemeleri için önce `config.schema.lookup` ile inceleyin (normalleştirilmiş
    yol, yüzeysel şema belgeleri, alt öğe özetleri), ardından kısmi bir nesneyle
    `config.apply` yerine `config.patch` tercih edin. Yapılandırmanın
    üzerine yazdıysanız yedekten geri yükleyin veya onarmak için
    `openclaw doctor` komutunu çalıştırın.

    Belgeler: [Modeller](/tr/concepts/models), [Yapılandırma](/tr/cli/configure),
    [Yapılandırma](/tr/cli/config), [Doctor](/tr/gateway/doctor).

  </Accordion>

  <Accordion title="Kendi barındırdığım modelleri (llama.cpp, vLLM, Ollama) kullanabilir miyim?">
    Evet — en kolay yol Ollama'dır. Hızlı kurulum:

    1. Ollama'yı `https://ollama.com/download` adresinden yükleyin
    2. Yerel bir model çekin; ör. `ollama pull gemma4`
    3. Bulut modelleri için de `ollama signin` komutunu çalıştırın
    4. `openclaw onboard` komutunu çalıştırın, `Ollama` seçeneğini ve ardından `Local` veya `Cloud + Local` seçeneğini belirleyin

    `Cloud + Local`, bulut modellerinin yanı sıra yerel Ollama modellerinizi
    de sunar; `kimi-k2.5:cloud` gibi bulut modellerinin yerel olarak çekilmesi
    gerekmez. Elle değiştirmek için: `openclaw models list`, ardından
    `openclaw models set ollama/<model>`.

    Daha küçük/yoğun şekilde nicemlenmiş modeller istem enjeksiyonuna karşı daha
    savunmasızdır. Araç erişimi olan tüm botlarda büyük modeller kullanın; yine de
    küçük modeller kullanıyorsanız korumalı alanı ve katı araç izin listelerini
    etkinleştirin.

    Belgeler: [Ollama](/tr/providers/ollama), [Yerel modeller](/tr/gateway/local-models),
    [Model sağlayıcıları](/tr/concepts/model-providers), [Güvenlik](/tr/gateway/security),
    [Korumalı alan](/tr/gateway/sandboxing).

  </Accordion>

  <Accordion title="Modelleri anında (yeniden başlatmadan) nasıl değiştirebilirim?">
    `/model <name>` ifadesini bağımsız bir mesaj olarak gönderin. Numaralı
    seçici (`/model`, `/model
    list`, `/model 3`), oturum
    geçersiz kılmasını temizleyen `/model default` ve uç nokta/API modu
    ayrıntılarını gösteren `/model status` dahil tam komut listesi için
    [Eğik çizgi komutları](/tr/tools/slash-commands) sayfasına bakın.

    `@profile` ile oturum başına belirli bir kimlik doğrulama profilini zorunlu kılın:

    ```text
    /model opus@anthropic:default
    /model opus@anthropic:work
    ```

    `@profile` ile ayarlanan bir profil sabitlemesini kaldırmak için
    `/model` komutunu son ek olmadan yeniden çalıştırın (ör.
    `/model anthropic/claude-opus-4-6`) veya `/model` içinden varsayılanı seçin.
    Etkin kimlik doğrulama profilini doğrulamak için `/model status` kullanın.

  </Accordion>

  <Accordion title="İki sağlayıcı aynı model kimliğini sunuyorsa /model hangisini kullanır?">
    `/model provider/model` tam olarak bu sağlayıcı rotasını seçer. Örneğin model
    kimliği eşleşse bile `qianfan/deepseek-v4-flash` ve `deepseek/deepseek-v4-flash` farklı
    referanslardır — OpenClaw, yalnızca kimlik eşleşmesine dayanarak sağlayıcıları
    sessizce değiştirmez.

    Kullanıcının seçtiği bir `/model` referansı geri dönüş konusunda
    katıdır: söz konusu sağlayıcı/model kullanılamaz hâle gelirse yanıt
    `agents.defaults.model.fallbacks` seçeneğine geri dönmek yerine görünür biçimde başarısız
    olur. Yapılandırılmış geri dönüş zincirleri; yapılandırılmış varsayılanlar,
    Cron işi birincil modelleri ve otomatik seçilen geri dönüş durumu için
    geçerli olmaya devam eder. Oturum geçersiz kılması olmayan bir çalıştırmanın
    geri dönüş kullanmasına izin verildiğinde OpenClaw önce istenen
    sağlayıcıyı/modeli, ardından yapılandırılmış geri dönüşleri ve son olarak
    yapılandırılmış birincil modeli dener; dolayısıyla yinelenen çıplak model
    kimlikleri doğrudan varsayılan sağlayıcıya geri atlamaz.

    Bkz. [Modeller](/tr/concepts/models) ve [Model yük devretme](/tr/concepts/model-failover).

  </Accordion>

  <Accordion title="Günlük görevler için GPT 5.5, kodlama için Codex 5.5 kullanabilir miyim?">
    Evet — model seçimi ve çalışma zamanı seçimi birbirinden ayrıdır:

    - **Yerel Codex kodlama aracısı:** `agents.defaults.model.primary` değerini
      `openai/gpt-5.5` olarak ayarlayın. ChatGPT/Codex abonelik kimlik doğrulaması için `openclaw models auth login --provider
      openai` ile oturum açın.
    - **Aracı döngüsü dışındaki doğrudan OpenAI API görevleri:** görseller,
      yerleştirmeler, konuşma, gerçek zamanlı işlemler ve aracı olmayan diğer
      OpenAI API yüzeyleri için `OPENAI_API_KEY` yapılandırın.
    - **OpenAI aracısı API anahtarıyla kimlik doğrulama:** sıralı bir
      `openai` API anahtarı profiliyle `/model openai/gpt-5.5`.
    - **Alt aracılar:** kodlama görevlerini kendi `openai/gpt-5.5`
      modeline sahip Codex odaklı bir aracıya yönlendirin.

    Bkz. [Modeller](/tr/concepts/models) ve [Eğik çizgi komutları](/tr/tools/slash-commands).

  </Accordion>

  <Accordion title="GPT 5.5 için hızlı modu nasıl yapılandırabilirim?">
    - **Oturum başına:** `openai/gpt-5.5` kullanırken `/fast on` gönderin.
    - **Model başına varsayılan:** `agents.defaults.models["openai/gpt-5.5"].params.fastMode` değerini
      `true` olarak ayarlayın.
    - **Otomatik zaman sınırı:** `/fast auto` veya `params.fastMode: "auto"`,
      zaman sınırına kadar yeni model çağrılarını hızlı çalıştırır; ardından
      sonraki yeniden deneme, geri dönüş, araç sonucu veya devam çağrılarını
      hızlı mod olmadan çalıştırır. Zaman sınırı varsayılan olarak 60 saniyedir;
      modeldeki `params.fastAutoOnSeconds` ile geçersiz kılın.

    ```json5
    {
      agents: {
        defaults: {
          models: {
            "openai/gpt-5.5": {
              params: {
                fastMode: "auto",
                fastAutoOnSeconds: 30,
              },
            },
          },
        },
      },
    }
    ```

    Hızlı mod, yerel OpenAI Responses isteklerinde `service_tier = "priority"` değerine
    eşlenir; mevcut `service_tier` değerleri korunur ve hızlı mod
    `reasoning` veya `text.verbosity` değerini yeniden yazmaz.
    Oturumdaki `/fast` geçersiz kılmaları yapılandırma
    varsayılanlarından önceliklidir.

    [Düşünme ve hızlı mod](/tr/tools/thinking) sayfasına ve [OpenAI](/tr/providers/openai)
    sağlayıcı sayfasındaki Gelişmiş yapılandırma altında yer alan Hızlı mod
    bölümüne bakın.

  </Accordion>

  <Accordion title='"Model ... is not allowed" iletisini görüp ardından neden yanıt alamıyorum?'>
    `agents.defaults.modelPolicy.allow` boş değilse `/model`, oturum geçersiz kılmaları
    ve `--model` için **izin listesi** olur. Bu listenin dışında bir
    model seçildiğinde normal yanıt yerine şu döndürülür:

    ```text
    Model override "provider/model" is not allowed by agents.defaults.modelPolicy.allow.
    ```

    Düzeltme: tam modeli veya `"provider/*"` gibi bir sağlayıcı joker
    karakterini belirtilen `modelPolicy.allow` listesine ekleyin, bu listeyi
    kaldırın/boşaltın ya da `/model list` içinden bir model seçin. Komut
    ayrıca `--runtime codex` içeriyorsa önce izin listesini güncelleyin,
    ardından aynı `/model provider/model --runtime codex` komutunu yeniden deneyin.

  </Accordion>

  <Accordion title='"Unknown model: minimax/MiniMax-M3" iletisini neden görüyorum?'>
    Eski bir OpenClaw sürümü kullanıyorsanız önce yükseltin (veya kaynaktan
    `main` çalıştırın) ve Gateway'i yeniden başlatın —
    `MiniMax-M3` henüz yüklü sürümünüzün kataloğunda olmayabilir. Aksi
    hâlde MiniMax sağlayıcısı yapılandırılmamıştır (sağlayıcı girdisi veya
    kimlik doğrulama profili bulunamamıştır), dolayısıyla model çözümlenemez.
    Eksiksiz düzeltme kontrol listesi, sağlayıcı/model kimliği tablosu ve
    yapılandırma bloğu örneği için [MiniMax](/tr/providers/minimax) sağlayıcı
    sayfasındaki Sorun Giderme bölümüne bakın.

  </Accordion>

  <Accordion title="Varsayılan olarak MiniMax'i, karmaşık görevler için OpenAI'ı kullanabilir miyim?">
    Evet. MiniMax'i varsayılan olarak kullanın ve modelleri oturum bazında
    değiştirin — geri dönüşler "zor görevler" için değil, hatalar içindir;
    dolayısıyla `/model` veya ayrı bir aracı kullanın.

    **A seçeneği: oturum bazında değiştirin**

    ```json5
    {
      env: { MINIMAX_API_KEY: "sk-...", OPENAI_API_KEY: "sk-..." },
      agents: {
        defaults: {
          model: { primary: "minimax/MiniMax-M3" },
          models: {
            "minimax/MiniMax-M3": { alias: "minimax" },
            "openai/gpt-5.5": { alias: "gpt" },
          },
        },
      },
    }
    ```

    Ardından `/model gpt`.

    **B seçeneği: ayrı aracılar** — Aracı A varsayılan olarak MiniMax'i,
    Aracı B varsayılan olarak OpenAI'ı kullanır; aracıya göre yönlendirin veya
    değiştirmek için `/agent` kullanın.

    Belgeler: [Modeller](/tr/concepts/models), [Çok Aracılı Yönlendirme](/tr/concepts/multi-agent),
    [MiniMax](/tr/providers/minimax), [OpenAI](/tr/providers/openai).

  </Accordion>

  <Accordion title="opus / sonnet / gpt yerleşik kısayollar mı?">
    Evet — yalnızca hedef model `agents.defaults.models` içinde mevcut olduğunda
    uygulanan yerleşik kısa adlardır:

    | Takma ad | Çözümlendiği değer |
    | --- | --- |
    | `opus` | `anthropic/claude-opus-5` |
    | `sonnet` | `anthropic/claude-sonnet-5` |
    | `gpt` | `openai/gpt-5.4` |
    | `gpt-mini` | `openai/gpt-5.4-mini` |
    | `gpt-nano` | `openai/gpt-5.4-nano` |
    | `gemini` | `google/gemini-3.1-pro-preview` |
    | `gemini-flash` | `google/gemini-3-flash-preview` |
    | `gemini-flash-lite` | `google/gemini-3.1-flash-lite` |

    Aynı ada sahip kendi takma adınız yerleşik olanı geçersiz kılar.

  </Accordion>

  <Accordion title="Model kısayollarını (takma adları) nasıl tanımlayabilir/geçersiz kılabilirim?">
    Takma adlar `agents.defaults.models.<modelId>.alias` konumunda bulunur:

    ```json5
    {
      agents: {
        defaults: {
          model: { primary: "anthropic/claude-opus-4-6" },
          models: {
            "anthropic/claude-opus-4-6": { alias: "opus" },
            "anthropic/claude-sonnet-4-6": { alias: "sonnet" },
          },
        },
      },
    }
    ```

    Ardından `/model sonnet` (veya desteklendiğinde `/<alias>`) bu
    model kimliğine çözümlenir.

  </Accordion>

  <Accordion title="OpenRouter veya Z.AI gibi diğer sağlayıcılardan modelleri nasıl ekleyebilirim?">
    OpenRouter (token başına ödeme; çok sayıda model):

    ```json5
    {
      agents: {
        defaults: {
          model: { primary: "openrouter/anthropic/claude-sonnet-4-6" },
          models: { "openrouter/anthropic/claude-sonnet-4-6": {} },
        },
      },
      env: { OPENROUTER_API_KEY: "sk-or-..." },
    }
    ```

    Z.AI (GLM modelleri):

    ```json5
    {
      agents: {
        defaults: {
          model: { primary: "zai/glm-5.1" },
          models: { "zai/glm-5.1": {} },
        },
      },
      env: { ZAI_API_KEY: "..." },
    }
    ```

    Başvurulan bir sağlayıcı/model için sağlayıcı anahtarının eksik olması
    çalışma zamanında kimlik doğrulama hatasına yol açar (ör. `No API key found for provider "zai"`).

    **Yeni bir aracı ekledikten sonra sağlayıcı için API anahtarı bulunamadı**

    Yeni bir aracının kimlik doğrulama deposu boştur — kimlik doğrulama aracı
    bazındadır ve şurada saklanır:

    ```text
    ~/.openclaw/agents/<agentId>/agent/auth-profiles.json
    ```

    Düzeltme: `openclaw agents add <id>` komutunu çalıştırıp sihirbazda kimlik doğrulamayı yapılandırın veya
    ana aracının deposundan yalnızca taşınabilir statik `api_key`/`token`
    profillerini kopyalayın. OAuth için yeni aracı kendi hesabına ihtiyaç
    duyduğunda oturum açın. Tüm `agentDir` yeniden kullanım ve kimlik
    bilgisi paylaşımı kuralları için [Çok Aracılı Yönlendirme](/tr/concepts/multi-agent)
    bölümüne bakın — aracılar arasında asla `agentDir` yeniden kullanmayın.

  </Accordion>
</AccordionGroup>

## Model yük devretme ve "Tüm modeller başarısız oldu"

<AccordionGroup>
  <Accordion title="Yük devretme nasıl çalışır?">
    İki aşama:

    1. Aynı sağlayıcı içinde **kimlik doğrulama profili rotasyonu**.
    2. `agents.defaults.model.fallbacks` içindeki bir sonraki modele **model yedek geçişi**.

    Başarısız profillere bekleme süreleri (üstel geri çekilme) uygulanır;
    böylece bir sağlayıcı hız sınırına takıldığında veya geçici olarak
    başarısız olduğunda OpenClaw yanıt vermeye devam eder.

    Hız sınırı grubu yalnızca `429` öğesinden fazlasını kapsar:
    `Too many concurrent
    requests`, `ThrottlingException`, `concurrency limit reached`,
    `workers_ai
    ... quota limit exceeded`, `resource exhausted` ve dönemsel kullanım penceresi
    sınırlarının (`weekly/monthly limit reached`) tümü yük devretmeyi gerektiren hız
    sınırları sayılır.

    Faturalandırma yanıtları her zaman `402` değildir ve bazı
    `402` öğeleri faturalandırma hattı yerine geçici/hız sınırı
    grubunda kalır. `401`/`403` üzerindeki açık
    faturalandırma metni yine faturalandırmaya yönlendirilebilir; sağlayıcıya
    özgü metin eşleştiriciler (ör. OpenRouter `Key limit exceeded`) yalnızca
    kendi sağlayıcıları kapsamında kalır. Yeniden denenebilir bir kullanım
    penceresi veya kuruluş/çalışma alanı harcama sınırı gibi görünen bir
    `402` (`daily limit reached, resets tomorrow`, `organization spending limit exceeded`), uzun süreli
    faturalandırma devre dışı bırakması olarak değil, `rate_limit`
    olarak değerlendirilir.

    Bağlam taşması hataları yedek geçiş yolunun tamamen dışında kalır —
    `request_too_large`, `input exceeds the maximum number of tokens`, `input token count exceeds the maximum number of input tokens`,
    `input is
    too long for the model` veya `ollama error: context length exceeded` gibi imzalar, model yedek
    geçişini ilerletmek yerine Compaction/yeniden deneme sürecine gider.

    Genel sunucu hatası metninin kapsamı, "içinde bilinmeyen/hata geçen her
    şey" ifadesinden daha dardır. Yük devretme sinyali sayılan sağlayıcı
    kapsamlı geçici biçimler şunlardır: yalın Anthropic `An unknown error occurred`,
    yalın OpenRouter `Provider returned error`, `Unhandled stop reason:
    error` gibi durdurma
    nedeni hataları, geçici sunucu metni (`internal
    server error`,
    `unknown error, 520`, `upstream error`, `backend error`) içeren JSON
    `api_error` yükleri ve sağlayıcı bağlamı eşleştiğinde
    `ModelNotReadyException` gibi sağlayıcı meşgul hataları. `LLM request failed
    with an unknown error.`
    gibi genel dahili yedek geçiş metni ölçülü kalır ve tek başına yedek
    geçişi tetiklemez.

  </Accordion>

  <Accordion title='"anthropic:default profili için kimlik bilgisi bulunamadı" ne anlama gelir?'>
    `anthropic:default` kimlik doğrulama profili kimliği, beklenen kimlik
    doğrulama deposunda hiçbir kimlik bilgisine sahip değildir.

    **Düzeltme kontrol listesi:**

    - Profillerin nerede bulunduğunu doğrulayın — güncel:
      `~/.openclaw/agents/<agentId>/agent/auth-profiles.json`; eski:
      `~/.openclaw/agent/*` (`openclaw doctor` tarafından taşınır).
    - Gateway'in ortam değişkeninizi yüklediğini doğrulayın. Yalnızca
      kabuğunuzda ayarlanan `ANTHROPIC_API_KEY`, systemd/launchd üzerinden
      çalıştırılan bir Gateway'e ulaşmaz — bunu `~/.openclaw/.env` içine
      koyun veya `env.shellEnv` özelliğini etkinleştirin.
    - Doğru aracıyı düzenlediğinizi doğrulayın — çok aracılı kurulumlarda
      birden fazla `auth-profiles.json` dosyası bulunur.
    - Yapılandırılmış modelleri ve sağlayıcı kimlik doğrulama durumunu
      görmek için `openclaw models status` komutunu çalıştırın.

    **"anthropic profili için kimlik bilgisi bulunamadı" (e-posta son eki yok) için:**

    Çalıştırma, Gateway'in bulamadığı bir Anthropic profiline sabitlenmiştir.

    - Claude CLI kullanın: Gateway ana makinesinde `openclaw models auth login --provider anthropic
      --method cli --set-default`
      komutunu çalıştırın.
    - Bunun yerine bir API anahtarı tercih edin: Gateway ana makinesindeki
      `~/.openclaw/.env` içine `ANTHROPIC_API_KEY` yerleştirin, ardından eksik
      profili zorunlu kılan sabitlenmiş sıralamayı temizleyin:

      ```bash
      openclaw models auth order clear --provider anthropic
      ```

    - Uzak mod: kimlik doğrulama profilleri dizüstü bilgisayarınızda değil,
      Gateway makinesinde bulunur — komutları orada çalıştırdığınızı doğrulayın.

  </Accordion>

  <Accordion title="Neden Google Gemini'ı da deneyip başarısız oldu?">
    Model yapılandırmanız Google Gemini'ı yedek olarak içeriyorsa (veya bir
    Gemini kısaltmasına geçtiyseniz), OpenClaw yedek geçiş sırasında onu
    dener. Google kimlik bilgileri yapılandırılmadığında `No API key found for provider
    "google"`
    oluşur. Düzeltme: Google kimlik doğrulaması ekleyin veya Google
    modellerini `agents.defaults.model.fallbacks`/takma adlardan kaldırın.

    **LLM isteği reddedildi: düşünme imzası gerekli (Google Antigravity)**

    Neden: oturum geçmişinde imzasız düşünme blokları vardır (çoğunlukla
    durdurulmuş/kısmi bir akıştan kaynaklanır); Google Antigravity düşünme
    bloklarında imza gerektirir. OpenClaw, Google Antigravity Claude için
    imzasız düşünme bloklarını kaldırır; sorun yine de görünürse yeni bir
    oturum başlatın veya söz konusu aracı için `/thinking off` ayarlayın.

  </Accordion>
</AccordionGroup>

## Kimlik doğrulama profilleri: nedir ve nasıl yönetilir?

İlgili: [/concepts/oauth](/tr/concepts/oauth) (OAuth akışları, belirteç depolama, çok hesaplı kalıplar)

<AccordionGroup>
  <Accordion title="Kimlik doğrulama profili nedir?">
    Bir sağlayıcıya bağlı, şu konumda depolanan adlandırılmış bir kimlik
    bilgisi kaydıdır (OAuth veya API anahtarı):

    ```text
    ~/.openclaw/agents/<agentId>/agent/auth-profiles.json
    ```

    Kayıtlı profilleri gizli bilgileri dökmeden inceleyin:
    `openclaw models auth
    list` (isteğe bağlı olarak `--provider <id>` veya
    `--json`). [Modeller CLI'si](/tr/cli/models#auth-profiles)
    bölümüne bakın.

  </Accordion>

  <Accordion title="Tipik profil kimlikleri nelerdir?">
    Sağlayıcı ön ekli: `anthropic:default` (e-posta kimliği olmadığında
    yaygındır), OAuth kimlikleri için `anthropic:<email>` veya seçtiğiniz
    özel bir kimlik (ör. `anthropic:work`).

  </Accordion>

  <Accordion title="İlk olarak hangi kimlik doğrulama profilinin deneneceğini denetleyebilir miyim?">
    Evet. `auth.order.<provider>` yapılandırması, sağlayıcı başına rotasyon
    sırasını belirler (yalnızca meta veriler — gizli bilgiler depolanmaz).

    OpenClaw, kısa süreli **bekleme** durumundaki (hız sınırları, zaman
    aşımları, kimlik doğrulama hataları) veya daha uzun süreli **devre dışı**
    durumundaki (faturalandırma/yetersiz kredi) bir profili atlayabilir.
    `openclaw models status
    --json` ile inceleyin ve `auth.unusableProfiles` öğesini kontrol
    edin. Hız sınırı bekleme süreleri model kapsamlı olabilir — bir model
    için bekleme durumundaki profil, aynı sağlayıcıdaki kardeş bir modele
    hizmet vermeye devam edebilir; faturalandırma/devre dışı pencereleri
    profilin tamamını engeller.

    Aracı başına sıra geçersiz kılması ayarlayın (o aracının
    `auth-state.json` öğesinde depolanır):

    ```bash
    # Yapılandırılmış varsayılan aracıyı kullanır (--agent seçeneğini atlayın)
    openclaw models auth order get --provider anthropic

    # Rotasyonu tek bir profile kilitleyin
    openclaw models auth order set --provider anthropic anthropic:default

    # Veya açık bir sıra belirleyin (sağlayıcı içinde yedek geçiş)
    openclaw models auth order set --provider anthropic anthropic:work anthropic:default

    # Geçersiz kılmayı temizleyin (auth.order yapılandırmasına / dönüşümlü dağıtıma geri dönün)
    openclaw models auth order clear --provider anthropic

    # Belirli bir aracıyı hedefleyin
    openclaw models auth order set --provider anthropic --agent main anthropic:default
    ```

    Gerçekte nelerin deneneceğini doğrulayın: `openclaw models status --probe`. Açık bir
    sıralamada yer almayan kayıtlı profil, sessizce denenmek yerine
    `excluded_by_auth_order` bildirir.

  </Accordion>

  <Accordion title="OAuth ile API anahtarı arasındaki fark nedir?">
    - **OAuth / CLI oturum açma**, sağlayıcının desteklediği durumlarda
      genellikle abonelik erişimini kullanır. Anthropic için OpenClaw'ın
      Claude CLI arka ucu, Anthropic'in şu anda abonelik kullanım
      sınırlarından yararlanan Agent SDK/programatik kullanım olarak
      değerlendirdiği Claude Code `claude -p` öğesini kullanır —
      güncel faturalandırma duraklatma durumu ve kaynak bağlantıları için
      [Anthropic](/tr/providers/anthropic) bölümüne bakın.
    - **API anahtarları**, belirteç başına ödeme faturalandırmasını kullanır.

    Sihirbaz; Anthropic Claude CLI, OpenAI Codex OAuth ve API anahtarlarını
    destekler.

  </Accordion>
</AccordionGroup>

## İlgili

- [SSS](/tr/help/faq) — ana SSS
- [SSS — hızlı başlangıç ve ilk çalıştırma kurulumu](/tr/help/faq-first-run)
- [Model seçimi](/tr/concepts/model-providers)
- [Model yük devretme](/tr/concepts/model-failover)
