---
read_when:
    - OpenClaw'da Anthropic modellerini kullanmak istiyorsunuz
    - Eşleştirilmiş bilgisayarlar arasındaki Claude CLI veya Claude Desktop oturumlarına göz atmak istiyorsunuz
summary: OpenClaw’da API anahtarları veya Claude CLI aracılığıyla Anthropic Claude kullanın
title: Anthropic
x-i18n:
    generated_at: "2026-07-26T23:30:19Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 08b34794352a559d549f7cf0cb88aca9cb537984049367f55be371bd8e0c10f0
    source_path: providers/anthropic.md
    workflow: 16
---

Anthropic, **Claude** model ailesini geliştirir. OpenClaw iki kimlik doğrulama yolu destekler:

- **API anahtarı** - kullanıma dayalı faturalandırmayla doğrudan Anthropic API erişimi (`anthropic/*` modelleri)
- **Claude CLI** - aynı ana makinedeki mevcut bir Claude Code oturumunu yeniden kullanma

## Kullanım ve maliyet takibi

OpenClaw, kullanılabilir Anthropic kimlik bilgisini algılar ve eşleşen kullanım yüzeyini seçer:

- Claude abonelik/kurulum kimlik bilgileri, kota aralıklarını ve isteğe bağlı ek kullanım bütçesini gösterir.
- `ANTHROPIC_ADMIN_KEY` veya `ANTHROPIC_ADMIN_API_KEY`, Control UI **Kullanım** bölümünde sağlayıcının bildirdiği 30 günlük kuruluş maliyetini ve Messages API kullanımını; günlük harcama, token/önbellek toplamları, en çok kullanılan modeller ve maliyet kategorileriyle birlikte gösterir.
- Anthropic sağlayıcı profilinde depolanan bir `sk-ant-admin...` kimlik bilgisi, otomatik olarak Admin API anahtarı olarak algılanır.

Admin API maliyet geçmişi, Anthropic'in [Kullanım ve Maliyet API'sinden](https://platform.claude.com/docs/en/manage-claude/usage-cost-api) gelir. Bu, OpenClaw'ın oturum verilerinden türetilen tahmini maliyetinden ayrı olan gerçek sağlayıcı faturalandırmasıdır.

<Warning>
OpenClaw'ın Claude CLI arka ucu, yüklü Claude Code CLI'ı
etkileşimsiz yazdırma modunda (`claude -p`) çalıştırır. Anthropic'in güncel Claude Code belgeleri
bu modu Agent SDK/programatik kullanım olarak tanımlar. Anthropic'in 15 Haziran 2026
tarihli destek güncellemesi, duyurulan ayrı Agent SDK faturalandırma değişikliğini
duraklattı: Claude Agent SDK, `claude -p` ve üçüncü taraf uygulama kullanımı hâlâ oturum açılmış
aboneliğin kullanım sınırlarından düşer ve daha önce duyurulan aylık Agent SDK
kredisi, Anthropic bu planı gözden geçirirken kullanılamaz.

Etkileşimli Claude Code da oturum açılmış Claude planının sınırlarından düşer.
API anahtarıyla kimlik doğrulama, doğrudan kullandıkça öde faturalandırmasıdır ve bu plana bağlı değildir.
Uzun süre çalışan gateway ana makineleri, paylaşılan otomasyon ve öngörülebilir üretim
harcamaları için bir Anthropic API anahtarı kullanın.

Anthropic'in güncel destek makaleleri, bir OpenClaw sürümü olmadan bu
davranışı değiştirebilir:

- [Claude Code CLI referansı](https://code.claude.com/docs/en/cli-usage)
- [Claude Agent SDK'yı Claude planınızla kullanma](https://support.claude.com/en/articles/15036540-use-the-claude-agent-sdk-with-your-claude-plan)
- [Claude Code'u Pro veya Max planınızla kullanma](https://support.claude.com/en/articles/11145838-use-claude-code-with-your-pro-or-max-plan)
- [Claude Code'u Team veya Enterprise planınızla kullanma](https://support.claude.com/en/articles/11845131-using-claude-code-with-your-team-or-enterprise-plan)
- [Claude Code maliyetlerini yönetme](https://code.claude.com/docs/en/costs)

</Warning>

## Başlarken

<Tabs>
  <Tab title="API anahtarı">
    **En uygun olduğu durum:** standart API erişimi ve kullanıma dayalı faturalandırma.

    <Steps>
      <Step title="API anahtarınızı alın">
        [Anthropic Console](https://console.anthropic.com/) içinde bir API anahtarı oluşturun.
      </Step>
      <Step title="İlk kurulumu çalıştırın">
        ```bash
        openclaw onboard
        # seçin: Anthropic API key
        ```

        Ya da anahtarı doğrudan iletin:

        ```bash
        openclaw onboard --anthropic-api-key "$ANTHROPIC_API_KEY"
        ```
      </Step>
      <Step title="Modelin kullanılabilir olduğunu doğrulayın">
        ```bash
        openclaw models list --provider anthropic
        ```
      </Step>
    </Steps>

    ### Yapılandırma örneği

    ```json5
    {
      env: { ANTHROPIC_API_KEY: "example-anthropic-key-not-real" },
      agents: { defaults: { model: { primary: "anthropic/claude-opus-5" } } },
    }
    ```

  </Tab>

  <Tab title="Claude CLI">
    **En uygun olduğu durum:** ayrı bir API anahtarı olmadan mevcut bir Claude CLI oturumunu yeniden kullanma.

    <Steps>
      <Step title="Claude CLI'ın kurulu ve oturumun açık olduğundan emin olun">
        Şununla doğrulayın:

        ```bash
        claude --version
        ```
      </Step>
      <Step title="İlk kurulumu çalıştırın">
        ```bash
        openclaw onboard
        # seçin: Claude CLI
        ```

        OpenClaw, mevcut Claude CLI kimlik bilgilerini algılar ve yeniden kullanır.
      </Step>
      <Step title="Modelin kullanılabilir olduğunu doğrulayın">
        ```bash
        openclaw models list --provider anthropic
        ```
      </Step>
    </Steps>

    <Note>
    Claude CLI arka ucunun kurulum ve çalışma zamanı ayrıntıları [CLI Arka Uçları](/tr/gateway/cli-backends) bölümündedir.
    </Note>

    <Warning>
    Claude CLI'ın yeniden kullanılması, OpenClaw işleminin Claude CLI oturumuyla aynı ana makinede
    çalışmasını gerektirir. Docker kurulumları bir konteyner ana dizinini kalıcı hâle getirebilir ve
    Claude Code oturumunu orada açabilir; bkz.
    [Docker'da Claude CLI arka ucu](/tr/install/docker#claude-cli-backend-in-docker).
    [Podman](/tr/install/podman) gibi diğer konteyner kurulumları, ana makinenin
    `~/.claude` dizinini kuruluma veya çalışma zamanına bağlamaz; burada bir Anthropic API anahtarı kullanın veya
    [OpenAI Codex](/tr/providers/openai) gibi OpenClaw tarafından yönetilen OAuth'a sahip
    bir sağlayıcı seçin.
    </Warning>

    ### Kurulum token'ı alma

    Claude Code'un kurulu olduğu herhangi bir makinede `claude setup-token` komutunu çalıştırın. Bu komut,
    `sk-ant-oat01-` ile başlayan uzun ömürlü bir token yazdırır.

    İlk kurulum sırasında macOS uygulamasında **Connect with an API key or token** altında
    **Anthropic setup-token** seçeneğini belirleyerek token'ı yapıştırın veya şunu kullanın:

    ```bash
    openclaw models auth login --provider anthropic --method setup-token
    ```

    ### Yapılandırma örneği

    Standart Anthropic model referansını ve bir CLI çalışma zamanı geçersiz kılmasını tercih edin:

    ```json5
    {
      agents: {
        defaults: {
          model: { primary: "anthropic/claude-opus-5" },
          models: {
            "anthropic/claude-opus-5": {
              agentRuntime: { id: "claude-cli" },
            },
          },
        },
      },
    }
    ```

    Eski `claude-cli/claude-opus-4-7` model referansları uyumluluk amacıyla
    çalışmaya devam eder, ancak yeni yapılandırma sağlayıcı/model seçimini
    `anthropic/*` olarak tutmalı ve yürütme arka ucunu sağlayıcı/model çalışma zamanı politikasına yerleştirmelidir.

    ### Faturalandırma ve `claude -p`

    OpenClaw, Claude CLI çalıştırmaları için Claude Code'un etkileşimsiz `claude -p` yolunu
    kullanır. Anthropic şu anda bu yolu Agent SDK/programatik kullanım olarak değerlendirir:

    - Anthropic'in 15 Haziran 2026 tarihli destek güncellemesi, daha önce duyurulan
      ayrı Agent SDK kredi planını duraklattı.
    - Abonelik planındaki Claude Agent SDK, `claude -p` ve üçüncü taraf uygulama kullanımı
      hâlâ oturum açılmış aboneliğin kullanım sınırlarından düşer.
    - Daha önce duyurulan aylık Agent SDK kredisi, Anthropic bu planı
      gözden geçirirken kullanılamaz.
    - Console/API anahtarı oturumları kullandıkça öde API faturalandırmasını kullanır ve
      abonelik Agent SDK kredisini almaz.

    Duraklatma bildirimi için Anthropic'in [Agent SDK planı
    makalesine](https://support.claude.com/en/articles/15036540-use-the-claude-agent-sdk-with-your-claude-plan), abonelik davranışı içinse Claude Code'un
    [Pro/Max](https://support.claude.com/en/articles/11145838-use-claude-code-with-your-pro-or-max-plan)
    ve
    [Team/Enterprise](https://support.claude.com/en/articles/11845131-use-claude-code-with-your-team-or-enterprise-plan)
    planı makalelerine bakın.

    Anthropic, bir OpenClaw sürümü olmadan Claude Code faturalandırma ve hız sınırı
    davranışını değiştirebilir. Faturalandırmanın öngörülebilirliği önemli olduğunda `claude auth status`, `/status` ve
    Anthropic'in bağlantı verilen belgelerini kontrol edin.

    <Tip>
    Paylaşılan üretim otomasyonu için Claude CLI yerine bir Anthropic API anahtarı
    kullanın. OpenClaw ayrıca [OpenAI Codex](/tr/providers/openai),
    [Qwen Cloud](/tr/providers/qwen), [MiniMax](/tr/providers/minimax) ve
    [Z.AI / GLM](/tr/providers/zai) kaynaklı abonelik tarzı seçenekleri destekler.
    </Tip>

  </Tab>
</Tabs>

## Bilgisayarlar arasındaki Claude oturumları

Paketle gelen Anthropic Plugin'i, normal oturumlar kenar çubuğuna bir **Claude Code**
grubu ekler. Satırlar normal Sohbet bölmesinde açılır. Gateway ve bağlı node
ana makinelerindeki arşivlenmemiş Claude Code oturumlarını keşfeder:

- Claude CLI oturumları geçerli proje dizini kayıtlarından gelir. Dizine eklenmemiş
  transkriptler için sınırlı bir meta veri geri dönüşü, `~/.claude/projects/` altındaki eşzamanlı, yan zincir olmayan
  etkileşimli (`cli`) ve başsız Agent SDK CLI (`sdk-cli`) oturumlarını tanır.
- Claude Desktop oturumları, meta verileri aynı Claude Code oturum kimliğine işaret ettiğinde
  Desktop başlığını, etkinlik zamanını ve arşiv durumunu kullanır.
- Yalnızca CLI oturumunda arşiv bayrağı bulunmadığından, transkripti
  mevcut olduğu sürece görünür kalır.

Keşif için ek OpenClaw yapılandırması gerekmez. Anthropic Plugin'i
paketle gelir ve varsayılan olarak etkindir; yerel `~/.claude/projects/` dizini mevcut olduğunda yerel bir macOS node'u
salt okunur Claude oturum komutlarını duyurur.
Bu komutlar ilk kez göründüğünde node eşleştirme yükseltmesini onaylayın.

Kenar çubuğu satırları Gateway veya eşleştirilmiş node ana makinelerine göre gruplandırır ve her
ana makinenin en yeni sınırlı sayfasını, ilgili bilgisayar yanıt verir vermez gösterir. Ana makine bağlantısı
değişikliklerinden sonra, sayfa yeniden odağa geldiğinde ve görünürken en fazla
30 saniyede bir yeniden uzlaştırma yapar; böylece OpenClaw dışında oluşturulan Claude oturumları
yeniden yükleme olmadan görünür. Değişen bir katalog daha hızlı bir takip geçişini tetikler. Daha fazla
geçmişi bulunan her ana makine için sonraki sayfayı eklemek üzere bir katalog grubunun altındaki **Daha fazla
oturum yükle** seçeneğini kullanın; eklenen satırlar görünür kalır ve yenilemeler boyunca aynı derinliğe
kadar yeniden getirilir. Katalog istemcileri `sessions.catalog.list` kullanır; bir satır açılırken
`sessions.catalog.read` kullanılır.

Terminal denetimini devralma, `claude` konumunu hizmet/daemon PATH'inden önce sahip ana makine kullanıcısının oturum açma kabuğu
PATH'inden çözümler. Bu, uygulamadan başlatılan oturumların operatörün normal bir terminalde
kullandığı Claude CLI ile uyumlu kalmasını sağlar.

Bir satır seçildiğinde önce en yeni transkript sayfası okunur. **Daha eski transkript
öğelerini yükle**, opak bir bayt imlecini izler ve tüm geçmişi yüklemek yerine
JSONL dosyasından başka bir sınırlı bölümü okur. Normal kullanıcı, asistan,
akıl yürütme, araç çağrısı ve araç sonucu içeriği korunur. Node/Gateway güvenlik tavanından
büyük olan tekil bir öğe, kesilmiş olarak açıkça işaretlenir.

Gateway'e yerel bir `claude-cli` satırında normal oluşturucuya metin yazmak,
`sessions.catalog.continue` çağrısını yapar. OpenClaw yerel katalog kaydını yeniden çözümler,
modele kilitli yerel bir oturum oluşturur veya yeniden kullanır, en fazla 200 görünür
öğeyi ya da 512 KiB'ı içe aktarır ve Claude CLI bağlamasını başlatır. İlk tur
`--fork-session` ile sürdürülür; Claude çatala yeni bir oturum kimliği atar, böylece sonraki turlar
çatalı kullanır ve kaynak oturum değiştirilmeden kalır.

Başsız bir node ana makinesi de aşağıdaki node'a yerel ayarı etkinleştirip
node ana makinesini yeniden başlatarak Claude CLI satırlarının sürdürülebilmesini sağlayabilir:

```json5
{
  nodeHost: {
    agentRuns: {
      claude: { enabled: true },
    },
  },
}
```

Node, yalnızca ayar etkinleştirildiğinde ve yerel `claude` yürütülebilir dosyası
çözümlendiğinde `agent.cli.claude.run.v1` duyurusunu yapar. OpenClaw, söz konusu node'daki katalog
kaydını yeniden çözümler, aynı sınırlı geçmişi içe aktarır ve benimsenen
oturumu node'a ve katalog tarafından bildirilen çalışma dizinine bağlar. Her turda
o node'un Claude dosyalarını ve oturumunu kullanarak node'un gerçek `claude -p` işlemi çalıştırılır.
Node'un exec onay politikası uygulanmaya devam eder; Gateway katılımı zorlayamaz.

Node sürdürme v1 yalnızca tek seferliktir. Gateway geri döngü MCP yapılandırmasını ve
Gateway Skills Plugin'i bağımsız değişkenlerini dışarıda bırakır, bir Gateway transkriptinden yeniden başlangıç vermez ve
eklerle görselleri reddeder. Claude Desktop satırları salt görüntülenebilir olarak kalır. Yerel
macOS uygulama node'ları da uygulama çalıştırma komutunu duyurana kadar salt görüntülenebilir olarak kalır.

<Note>
Eşleştirilmiş node Claude oturumları, başsız node açıkça
`agent.cli.claude.run.v1` duyurmadığı sürece salt okunur kalır. OpenClaw hiçbir zaman Claude Desktop
meta verilerini değiştirmez veya Claude oturumlarını arşivlemez. Sayfa, kimliği doğrulanmış
`node.invoke` kullandığı için yazma kapsamına sahip bir operatör bağlantısı gerektirir; listeleme ve okuma,
sürdürmenin etkin olduğu bir node'da bile salt okunur kalır.
</Note>

Node komutu ve güvenlik sınırı için [Node'lar: Claude oturumları ve transkriptleri](/tr/nodes#claude-sessions-and-transcripts) bölümüne bakın.

## Düşünme varsayılanları (Claude Opus 5, Sonnet 5, Mythos 5, Fable 5, 4.8 ve 4.6)

`anthropic/claude-opus-5` varsayılan olarak `high` efor düzeyinde uyarlamalı düşünmeyi kullanır.
Düşünmeyi devre dışı bırakmak için `/think off`, modelin daha yüksek
yerel efor düzeyleri için ise `/think xhigh|max` kullanın. Anthropic bu modelde
bu istek özelliklerini desteklemediğinden OpenClaw, Opus 5 için manuel düşünme
bütçelerini, özel örnekleme parametrelerini, asistan ön doldurmalarını ve Priority Tier'ı
kullanmaz. Katalog; modelin 1.000.000 tokenlık bağlam penceresini, 128.000 tokenlık çıktı
sınırını, görüntü girdisini ve `$5/$25` girdi/çıktı fiyatlandırmasını yayımlar.

`anthropic/claude-sonnet-5` aynı uyarlamalı düşünme varsayılanlarını ve istek
kısıtlamalarını kullanır. Katalog, 31 Ağustos 2026'ya kadar Anthropic'in başlangıç
`$2/$10` girdi/çıktı fiyatlandırmasını kullanır; standart `$3/$15`
fiyatlandırması 1 Eylül 2026'da başlar.

`anthropic/claude-fable-5` her zaman uyarlamalı düşünmeyi kullanır ve varsayılan eforu
`high` düzeyidir. Anthropic bu modelde düşünmenin devre dışı bırakılmasına
izin vermediğinden `/think off` ve `/think minimal`, bunun yerine
`low` efor düzeyine eşlenir. Anthropic, düşünmenin etkin olduğu herhangi
bir istekte sıcaklık geçersiz kılmasını reddettiğinden OpenClaw ayrıca Fable 5
isteklerinde özel sıcaklık değerlerini kullanmaz.

`anthropic/claude-mythos-5`, aynı sürekli etkin uyarlamalı düşünme sözleşmesine sahip
sınırlı erişimli bir modeldir. OpenClaw varsayılan olarak `high` kullanır,
`/think off` ve `/think minimal` değerlerini `low` değerine eşler
ve çağıranın seçtiği örnekleme parametrelerini kullanmaz. Katalog; modelin 1.000.000
tokenlık bağlam penceresini, 128.000 tokenlık çıktı sınırını, görüntü girdisini ve
`$10/$50` girdi/çıktı fiyatlandırmasını yayımlar.

Claude Opus 4.8, OpenClaw'da düşünmeyi varsayılan olarak kapalı tutar. Uyarlamalı
düşünmeyi `/think high|xhigh|max` ile açıkça etkinleştirdiğinizde OpenClaw,
Anthropic'in Opus 4.8 efor değerlerini gönderir; Claude 4.6 modelleri (Opus 4.6 ve Sonnet 4.6)
varsayılan olarak `adaptive` kullanır.

Her mesaj için `/think:<level>` ile veya model parametrelerinde geçersiz kılın:

```json5
{
  agents: {
    defaults: {
      models: {
        "anthropic/claude-opus-5": {
          params: { thinking: "high" },
        },
      },
    },
  },
}
```

<Note>
İlgili Anthropic belgeleri:
- [Uyarlamalı düşünme](https://platform.claude.com/docs/en/build-with-claude/adaptive-thinking)
- [Genişletilmiş düşünme](https://platform.claude.com/docs/en/build-with-claude/extended-thinking)

</Note>

## Güvenlik reddi durumunda yedek modele geçiş (Claude Fable 5)

<Warning>
Claude Fable 5 kullanmak, Claude Opus 4.8'i de kullanmak anlamına gelir. Fable 5,
bir isteği reddedebilen güvenlik sınıflandırıcılarıyla sunulur ve Anthropic'in
onayladığı kurtarma yöntemi, söz konusu isteği `claude-opus-4-8` modeline
yanıtlatmaktır. OpenClaw, doğrudan API anahtarı isteklerinde bunu otomatik olarak
etkinleştirir; dolayısıyla bazı Fable istekleri Claude Opus 4.8 olarak yanıtlanır
ve ücretlendirilir. Politikanız veya bütçeniz Opus tarafından yanıtlanan istekleri
kabul edemiyorsa `anthropic/claude-fable-5` seçmeyin.
</Warning>

### Bunun var olma nedeni

Fable 5 sınıflandırıcıları, kısıtlı alanlardaki isteklerde `stop_reason: "refusal"`
döndürür ve zararsız çalışmalara yakın konularda da yanlış pozitif sonuç verir
(güvenlik araçları, yaşam bilimleri veya modelden ham akıl yürütmesini yeniden
üretmesini istemek gibi). Yedek model olmadan, başka bir Claude modeli isteği
memnuniyetle yanıtlayabilecek olsa bile istek bir hatayla sonlanır; Anthropic'in
kendi ret mesajı, API entegratörlerine bir yedek model yapılandırmalarını söyler.

### Nasıl çalışır?

1. `anthropic/claude-fable-5` modeline yönelik her doğrudan API anahtarı isteğinde OpenClaw,
   Anthropic'in sunucu tarafı yedek modele geçiş kabulünü gönderir:
   `server-side-fallback-2026-06-01` beta başlığı ile
   `fallbacks: [{"model": "claude-opus-4-8"}]`. Claude Opus 4.8, Anthropic'in Fable 5 için izin verdiği
   tek yedek hedeftir.
2. Yedek modele geçişi yalnızca güvenlik sınıflandırıcısının reddi tetikler. Hız sınırları,
   aşırı yüklenmeler ve sunucu hataları eskisiyle tamamen aynı şekilde davranır ve
   OpenClaw'ın normal [model yük devretme](/tr/concepts/model-failover) sürecinden geçer.
3. Kurtarma aynı çağrı içinde gerçekleşir. Herhangi bir çıktıdan önceki ret, gecikme
   dışında görünmez; yanıtın tamamı Opus 4.8'den gelir. Akış ortasında ret durumunda,
   kısmi metin yedek modelin devam edeceği önek olarak tutulurken reddedilen modelin
   akıl yürütmesi ve araç çağrıları Anthropic'in yeniden oynatma kuralları uyarınca
   atılır (bunlar geri yansıtılmamalı veya yürütülmemelidir).
4. Claude Opus 4.8 de reddederse istek, bu özellikten önce olduğu gibi reddi
   hata olarak gösterir.

Yedek modele geçiş Anthropic API düzeyinde gerçekleştiğinden `claude-opus-4-8`
yapılandırılmış model listenizde veya yedek zincirinizde bulunmak zorunda değildir;
Fable özellikli bir API anahtarı her zaman Opus'u sunabilir.

### Gözlemlenebilirlik ve faturalandırma

- Yedek model tarafından yanıtlanan bir istek, asistan mesajında
  `fromModel` ve `toModel` adlarını belirten bir `provider_fallback`
  tanılaması kaydeder ve mesajın `responseModel` alanı `claude-opus-4-8` bildirir.
- Anthropic her denemeyi ayrı ücretlendirir: çıktıdan önceki ret ücretsizdir ve kurtarma
  Claude Opus 4.8 tarifeleriyle ücretlendirilir (şu anda Fable 5 tarifelerinin yarısı).
  OpenClaw'ın istek başına maliyet tahmini, buna uygun olarak yedek model tarafından
  yanıtlanan istekleri Opus tarifeleriyle fiyatlandırır.
- Akış ortasındaki ret ayrıca Anthropic tarafında daha önce akışla gönderilmiş Fable
  bölümünü ücretlendirir; bu bölüm API'nin deneme başına kullanımında bildirilir ancak
  OpenClaw'ın istek başına tahminine dahil edilmez.

### Kapsam

`api.anthropic.com` hedefine karşı API anahtarı kimlik doğrulamasıyla
`anthropic/claude-fable-5` için geçerlidir. OAuth (Claude CLI aboneliğinin yeniden kullanımı),
proxy temel URL'leri, Bedrock, Vertex ve Foundry istekleri değişmez ve bu ortamlarda
retleri hata olarak göstermeye devam eder.

Canlı olarak doğrulandı: Fable 5'ten ham düşünce zincirini yeniden üretmesini isteyen
zararsız bir istem, yedekler olmadan gönderildiğinde `category: "reasoning_extraction"` ile reddedilir;
aynı istem OpenClaw üzerinden gönderildiğinde ise `provider_fallback` tanılamasının
eklendiği, Opus tarafından yanıtlanan normal bir yanıt döndürür.

Temel davranış için Anthropic'in [retler ve yedek modele geçiş
kılavuzuna](https://platform.claude.com/docs/en/build-with-claude/refusals-and-fallback)
bakın.

## İstem önbelleğe alma

OpenClaw, API anahtarı kimlik doğrulamasında Anthropic'in istem önbelleğe alma özelliğini destekler.

| Değer               | Önbellek süresi | Açıklama                                      |
| ------------------- | --------------- | --------------------------------------------- |
| `"short"` (varsayılan) | 5 dakika        | API anahtarı kimlik doğrulamasında otomatik uygulanır |
| `"long"`            | 1 saat           | Genişletilmiş önbellek                        |
| `"none"`            | Önbelleğe alma yok | İstem önbelleğe almayı devre dışı bırakır     |

```json5
{
  agents: {
    defaults: {
      models: {
        "anthropic/claude-opus-4-6": {
          params: { cacheRetention: "long" },
        },
      },
    },
  },
}
```

<AccordionGroup>
  <Accordion title="Aracı başına önbellek geçersiz kılmaları">
    Model düzeyindeki parametreleri temel değerleriniz olarak kullanın, ardından belirli aracıları `agents.entries.*.params` aracılığıyla geçersiz kılın:

    ```json5
    {
      agents: {
        defaults: {
          model: { primary: "anthropic/claude-opus-4-6" },
          models: {
            "anthropic/claude-opus-4-6": {
              params: { cacheRetention: "long" },
            },
          },
        },
        list: [
          { id: "research", default: true },
          { id: "alerts", params: { cacheRetention: "none" } },
        ],
      },
    }
    ```

    Yapılandırma birleştirme sırası:

    1. `agents.defaults.models["provider/model"].params`
    2. `agents.entries.*.params` (eşleşen `id`, anahtara göre geçersiz kılar)

    Bu, bir aracının uzun ömürlü bir önbelleği korumasına, aynı modeldeki başka bir aracının ise ani yoğunluklu/düşük yeniden kullanımlı trafik için önbelleğe almayı devre dışı bırakmasına olanak tanır.

  </Accordion>

  <Accordion title="Bedrock Claude notları">
    - Bedrock'taki Anthropic Claude modelleri (`amazon-bedrock/*anthropic.claude*`) yapılandırıldığında `cacheRetention` aktarımını kabul eder.
    - Anthropic dışı Bedrock modelleri çalışma zamanında `cacheRetention: "none"` değerine zorlanır.
    - API anahtarı akıllı varsayılanları, açık bir değer ayarlanmadığında Bedrock'taki Claude referansları için `cacheRetention: "short"` değerini de başlangıç değeri olarak belirler.

  </Accordion>
</AccordionGroup>

## Gelişmiş yapılandırma

<AccordionGroup>
  <Accordion title="Hızlı mod">
    OpenClaw'ın paylaşılan `/fast` anahtarı, `api.anthropic.com` hedefine yönelik doğrudan API anahtarı trafiğinde Anthropic'in `service_tier` alanını ayarlar.

    | Komut | Eşlendiği değer |
    |-------|-----------------|
    | `/fast on` | `service_tier: "auto"` |
    | `/fast off` | `service_tier: "standard_only"` |

    ```json5
    {
      agents: {
        defaults: {
          models: {
            "anthropic/claude-sonnet-4-6": {
              params: { fastMode: true },
            },
          },
        },
      },
    }
    ```

    <Note>
    - Yalnızca API anahtarıyla yapılan doğrudan `api.anthropic.com` isteklerine uygulanır. OAuth/abonelik tokenı isteklerine ve proxy rotalarına hiçbir zaman `service_tier` alanı eklenmez.
    - Açık `serviceTier` veya `service_tier` parametreleri, ikisi de ayarlandığında `/fast` değerini geçersiz kılar.
    - Claude Opus 5 ve Sonnet 5, Priority Tier'ı desteklemediğinden OpenClaw bu modeller için `service_tier` değerini kullanmaz.
    - Priority Tier kapasitesi olmayan hesaplarda `service_tier: "auto"`, `standard` olarak çözümlenebilir.

    </Note>

  </Accordion>

  <Accordion title="Medya anlama (görüntü ve PDF)">
    Paketle gelen Anthropic plugini, görüntü ve PDF anlama desteğini kaydeder. OpenClaw,
    yapılandırılmış Anthropic kimlik doğrulamasından medya yeteneklerini otomatik olarak
    çözümler; ek yapılandırma gerekmez.

    | Özellik           | Değer                  |
    | ----------------- | ---------------------- |
    | Varsayılan model  | `claude-opus-5`       |
    | Desteklenen girdi | Görüntüler, PDF belgeleri |

    Bir konuşmaya görüntü veya PDF eklendiğinde OpenClaw, bunu otomatik olarak
    Anthropic medya anlama sağlayıcısı üzerinden yönlendirir.

  </Accordion>

  <Accordion title="1M bağlam penceresi">
    Claude Opus 5, Sonnet 5, Mythos 5 ve Fable 5, tam olarak
    1.000.000 tokenlık bir girdi penceresine sahiptir ve 128.000'e kadar çıktı tokenını destekler.
    Anthropic'in 1M bağlam penceresi, uyarlamalı düşünmeye sahip Claude 4.x modellerinde de genel kullanıma sunulmuştur:
    Opus 4.8,
    Opus 4.7, Opus 4.6 ve Sonnet 4.6. OpenClaw bu modelleri
    otomatik olarak boyutlandırır; `params.context1m` gerekmez:

    ```json5
    {
      agents: {
        defaults: {
          models: {
            "anthropic/claude-opus-5": {},
            "anthropic/claude-sonnet-5": {},
            "anthropic/claude-mythos-5": {},
            "anthropic/claude-opus-4-6": {},
          },
        },
      },
    }
    ```

    Eski yapılandırmalar `params.context1m: true` değerini tutabilir; bu modeller için
    zararsız ve etkisizdir ve OpenClaw artık kullanımdan kaldırılmış
    `context-1m-2025-08-07` beta başlığını hiçbir koşulda göndermez. Bu değere sahip eski
    `anthropicBeta` yapılandırma girdileri, istek başlığı çözümlemesi sırasında
    kaldırılır ve desteklenmeyen eski Claude modelleri normal bağlam pencerelerinde kalır.

    `params.context1m: true`, Claude CLI arka ucu (`claude-cli/*`) için de aynı şekilde
    davranır: genel kullanıma uygun Opus ve Sonnet modelleri zaten 1M pencereyi otomatik
    olarak aldığından parametre burada da isteğe bağlıdır.

    <Warning>
    Anthropic kimlik bilgilerinizde uzun bağlam erişimi gerektirir. OAuth/abonelik tokenı kimlik doğrulaması gerekli Anthropic beta başlıklarını korur ancak OpenClaw, eski yapılandırmada kalmışsa kullanımdan kaldırılmış 1M beta başlığını kaldırır.
    </Warning>

  </Accordion>

  <Accordion title="Claude Opus 5 1M bağlamı">
    `anthropic/claude-opus-5` ve `claude-cli` varyantı varsayılan olarak 1M bağlam
    penceresine sahiptir; `params.context1m: true` gerekmez.
  </Accordion>
</AccordionGroup>

## Sorun giderme

<AccordionGroup>
  <Accordion title="401 hataları / token aniden geçersiz">
    Anthropic token kimlik doğrulamasının süresi dolar ve token iptal edilebilir. Yeni kurulumlarda bunun yerine bir Anthropic API anahtarı kullanın.
  </Accordion>

  <Accordion title='"anthropic" sağlayıcısı için API anahtarı bulunamadı'>
    Anthropic kimlik doğrulaması **agent başınadır**; yeni agent'lar ana agent'ın anahtarlarını devralmaz. İlgili agent için ilk katılımı yeniden çalıştırın (veya Gateway ana makinesinde bir API anahtarı yapılandırın), ardından `openclaw models status` ile doğrulayın.
  </Accordion>

  <Accordion title='"anthropic:default" profili için kimlik bilgisi bulunamadı'>
    Hangi kimlik doğrulama profilinin etkin olduğunu görmek için `openclaw models status` komutunu çalıştırın. İlk katılımı yeniden çalıştırın veya bu profil yolu için bir API anahtarı yapılandırın.
  </Accordion>

  <Accordion title="Kullanılabilir kimlik doğrulama profili yok (tümü bekleme süresinde)">
    `auth.unusableProfiles` için `openclaw models status --json` değerini kontrol edin. Anthropic hız sınırı bekleme süreleri modele özgü olabilir; dolayısıyla aynı gruptaki başka bir Anthropic modeli hâlâ kullanılabilir. Başka bir Anthropic profili ekleyin veya bekleme süresinin dolmasını bekleyin.
  </Accordion>
</AccordionGroup>

<Note>
Daha fazla yardım: [Sorun giderme](/tr/help/troubleshooting) ve [SSS](/tr/help/faq).
</Note>

## İlgili

<CardGroup cols={2}>
  <Card title="Model seçimi" href="/tr/concepts/model-providers" icon="layers">
    Sağlayıcıları, model referanslarını ve yük devretme davranışını seçme.
  </Card>
  <Card title="CLI arka uçları" href="/tr/gateway/cli-backends" icon="terminal">
    Claude CLI arka uç kurulumu ve çalışma zamanı ayrıntıları.
  </Card>
  <Card title="İstem önbelleğe alma" href="/tr/reference/prompt-caching" icon="database">
    İstem önbelleğe almanın sağlayıcılar genelinde nasıl çalıştığı.
  </Card>
  <Card title="OAuth ve kimlik doğrulama" href="/tr/gateway/authentication" icon="key">
    Kimlik doğrulama ayrıntıları ve kimlik bilgilerini yeniden kullanma kuralları.
  </Card>
</CardGroup>
