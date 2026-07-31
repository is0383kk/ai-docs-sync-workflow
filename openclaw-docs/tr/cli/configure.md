---
read_when:
    - Kimlik bilgilerini, cihazları veya ajan varsayılanlarını etkileşimli olarak ayarlamak istiyorsunuz
summary: '`openclaw configure` için CLI başvurusu (etkileşimli yapılandırma istemleri)'
title: Yapılandırın
x-i18n:
    generated_at: "2026-07-26T23:51:56Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 5980d06e75a5df9e5269d0ef78431f730d6f5fd050dca74784ef3426fb0433d8
    source_path: cli/configure.md
    workflow: 16
---

# `openclaw configure`

Mevcut bir kurulumda hedeflenen değişiklikler için etkileşimli istemler: kimlik bilgileri, cihazlar, ajan varsayılanları, Gateway, kanallar, Plugin'ler, Skills ve sistem durumu denetimleri.

Tam yönlendirmeli ilk çalıştırma süreci için `openclaw onboard` veya `openclaw setup`, yalnızca temel yapılandırma/çalışma alanı için `openclaw setup --baseline`, yalnızca kanal hesabı kurulumu gerektiğinde ise `openclaw channels add` kullanın.

<Tip>
Alt komut olmadan `openclaw config` aynı sihirbazı açar. Etkileşimsiz düzenlemeler için `openclaw config get|set|unset` kullanın.
</Tip>

## Seçenekler

`--section <section>`: yinelenebilir bölüm filtresi. Kullanılabilir bölümler:

`workspace`, `model`, `web`, `gateway`, `daemon`, `channels`, `plugins`, `skills`, `health`

```bash
openclaw configure
openclaw configure --section web
openclaw configure --section model --section channels
openclaw configure --section gateway --section daemon
```

`gateway`, `daemon` veya `health` seçildiğinde (ya da `--section` olmadan tam sihirbaz çalıştırıldığında), Gateway'in nerede çalışacağı sorulur ve `gateway.mode` güncellenir. Üçünü de atlayan bölüm filtreleri, Gateway modu istemi olmadan doğrudan istenen kuruluma geçer. Uzak Gateway modu seçildiğinde uzak yapılandırma yazılır ve işlem hemen sonlandırılır; Plugin kurulumları gibi yalnızca yerelde gerçekleştirilen adımlar çalıştırılmaz.

<Note>
`openclaw configure` etkileşimli bir terminal gerektirir (hem stdin hem de stdout TTY olmalıdır). Böyle bir terminal olmadığında, kısmen çalışmak yerine eşdeğer etkileşimsiz `openclaw config get|set|patch|validate` komutlarını yazdırır ve hatayla sonlanır.
</Note>

## Model bölümü

<Note>
**Model**, açık `agents.defaults.modelPolicy.allow` listesi için çoklu seçim içerir (`/model` içinde ve model seçicide gösterilenler). Sağlayıcı kapsamlı kurulum seçenekleri, yapılandırmada zaten bulunan ilgisiz sağlayıcıların yerini almak yerine seçilen modelleri mevcut listeyle birleştirir. Model başına takma adlar ve parametreler `agents.defaults.models` altında kalır; bu girdiler tek başlarına model geçersiz kılmalarını kısıtlamaz.

Yapılandırma üzerinden sağlayıcı kimlik doğrulaması yeniden çalıştırıldığında, sağlayıcının kimlik doğrulama adımı kendi önerilen varsayılan modelini içeren bir yapılandırma yaması döndürse bile mevcut `agents.defaults.model.primary` korunur. Bir sağlayıcının eklenmesi veya kimliğinin yeniden doğrulanması, mevcut birincil modelinizin yerini almadan o sağlayıcının modellerini kullanılabilir hâle getirir. Varsayılan modeli bilerek değiştirmek için `openclaw models auth login --provider <id> --set-default` veya `openclaw models set <model>` kullanın.
</Note>

Yapılandırma bir sağlayıcı kimlik doğrulama seçeneğinden başlatıldığında, varsayılan model ve model ilkesi seçicileri otomatik olarak bu sağlayıcıyı tercih eder. Volcengine ve BytePlus gibi eşleştirilmiş sağlayıcılarda aynı tercih, bunların kodlama planı varyantlarıyla da eşleşir (`volcengine-plan/*`, `byteplus-plan/*`). Tercih edilen sağlayıcı filtresi boş bir liste oluşturacaksa yapılandırma, boş bir seçici göstermek yerine filtresiz kataloğa geri döner.

## Web bölümü

`openclaw configure --section web` bir web arama sağlayıcısı seçer ve kimlik bilgilerini yapılandırır. Bazı sağlayıcılar, sağlayıcıya özgü ek adımlar gösterir:

- **Grok**, aynı xAI OAuth profili veya API anahtarıyla isteğe bağlı `x_search` kurulumu sunabilir ve bir `x_search` modeli seçmenize olanak tanıyabilir.
- **Kimi**, Moonshot API bölgesini (`api.moonshot.ai` veya `api.moonshot.cn`) ve varsayılan Kimi web arama modelini sorabilir.

## Diğer notlar

- Yerel yapılandırma yazıldıktan sonra, seçilen kurulum yolu gerektiriyorsa yapılandırma seçilen indirilebilir Plugin'leri kurar. Uzak Gateway yapılandırması yerel Plugin paketlerini kurmaz.
- Kanal odaklı hizmetler (Slack/Discord/Matrix/Microsoft Teams), kurulum sırasında kanal/oda izin listelerini sorar. Ad veya kimlik girebilirsiniz; sihirbaz mümkün olduğunda adları kimliklere dönüştürür.
- Arka plan programı kurulum adımını çalıştırırsanız belirteç kimlik doğrulaması bir belirteç gerektirir. `gateway.auth.token` SecretRef tarafından yönetiliyorsa yapılandırma SecretRef'i doğrular, ancak çözümlenen düz metin belirteç değerlerini gözetmen hizmetinin ortam meta verilerinde kalıcı olarak saklamaz; SecretRef çözümlenemezse yapılandırma, uygulanabilir düzeltme yönergeleri sunarak arka plan programı kurulumunu engeller.
- Hem `gateway.auth.token` hem de `gateway.auth.password` yapılandırılmışsa ve `gateway.auth.mode` ayarlanmamışsa yapılandırma, modu açıkça ayarlayana kadar arka plan programı kurulumunu engeller.

## İlgili

- [CLI başvurusu](/tr/cli)
- [Yapılandırma](/tr/gateway/configuration)
- Yapılandırma CLI'si: [Yapılandırma](/tr/cli/config)
