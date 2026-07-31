---
read_when:
    - Sağlayıcı kimlik bilgileri ve `auth-profiles.json` referansları için SecretRef'leri yapılandırma
    - Üretimde gizli bilgileri güvenli bir şekilde yeniden yükleme, denetleme, yapılandırma ve uygulama
    - Başlatma sırasında hızlı hata verme, etkin olmayan yüzeylerin filtrelenmesi ve bilinen son iyi durum davranışını anlama
sidebarTitle: Secrets management
summary: 'Gizli değer yönetimi: SecretRef sözleşmesi, çalışma zamanı anlık görüntü davranışı ve güvenli tek yönlü temizleme'
title: Gizli bilgilerin yönetimi
x-i18n:
    generated_at: "2026-07-26T23:58:29Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: d10989ebbce367c68d28768244d4e3649028af5ab63c9523974352c270a3c55e
    source_path: gateway/secrets.md
    workflow: 16
---

OpenClaw, desteklenen kimlik bilgilerinin yapılandırmada düz metin olarak tutulmasını gerektirmeyen eklemeli SecretRef'leri destekler.

<Note>
Düz metin kullanılmaya devam edilebilir. SecretRef'ler her kimlik bilgisi için isteğe bağlıdır.
</Note>

<Warning>
Düz metin kimlik bilgileri, `openclaw.json`, `auth-profiles.json`, `.env` veya oluşturulan `agents/*/agent/models.json` dosyaları dahil olmak üzere, aracının inceleyebildiği dosyalarda bulunuyorsa aracı tarafından okunabilir olmaya devam eder. SecretRef'ler bu yerel etki alanını yalnızca desteklenen her kimlik bilgisi taşındıktan ve `openclaw secrets audit --check` hiçbir düz metin kalıntısı olmadığını bildirdikten sonra azaltır.
</Warning>

## Çalışma zamanı modeli

- Gizli değerler, istek yollarında tembel olarak değil, etkinleştirme sırasında istekli olarak bellek içi bir çalışma zamanı anlık görüntüsüne çözümlenir.
- Soğuk Gateway başlatması, yeniden denenebilir bir SecretRef hatasını, bu sahip yalıtımı desteklediğinde bilinen ve Gateway dışındaki bir sahiple sınırlar. Eşlenen sahip sınıfları arasında model sağlayıcıları ve Skills, medya/TTS/cron sağlayıcıları, uygun kimlik doğrulama profilleri, aracı başına bellek, korumalı alan SSH'si, kanal hesapları ve manifestte bildirilen Plugin yolları bulunur. Gateway başlatılır, sahibi yapılandırılmış ancak kullanılamaz olarak kaydeder ve gizli bilgileri çıkarılmış bir bozulma uyarısı yayınlar. Gateway giriş kimlik doğrulaması, yapısal olarak geçersiz referanslar veya çözümlenmiş değerler, hata durumunda kapalı kalan sahipler ve çalışma zamanı sahibi eşlenmemiş referanslar başlatmayı yine başarısız kılar.
- Yeniden yükleme, eşlenen her sahibi bağımsız olarak doğrular ve ardından tek bir atomik anlık görüntü yayımlar. Sağlıklı sahipler yenilenir. Uygun bir başarısız sahip, yalnızca referans kimlikleri, sağlayıcı tanımları ve gizli olmayan eksiksiz sahip sözleşmesi değişmediyse bilinen son iyi değerini korur ve bayat duruma geçer; değiştirilmiş veya yeni bir başarısız sahip soğuk duruma geçer. Katı bir hata yeniden yüklemeyi reddeder ve etkin anlık görüntüyü korur.
- İlke ihlalleri (örneğin SecretRef girdisiyle birleştirilmiş OAuth modundaki bir kimlik doğrulama profili), çalışma zamanı değiştirilmeden önce etkinleştirmeyi başarısız kılar.
- Çalışma zamanı istekleri yalnızca etkin bellek içi anlık görüntüyü okur. Model sağlayıcısı SecretRef kimlik bilgileri, çıkışa kadar işlem yerel gözcü değerleri olarak kimlik doğrulama depolamasından ve akış seçeneklerinden geçer. Giden teslim yolları da (Discord yanıt/ileti dizisi teslimi, Telegram eylem gönderimleri) bu anlık görüntüyü okur ve her gönderimde referansları yeniden çözümlemez.

Bu, gizli değer sağlayıcısı kesintilerini yoğun istek yollarından uzak tutar.

Gateway giriş koruması, yapısal olarak geçersiz yapılandırma veya çözümlenmiş değerler, ilke ihlalleri ve bilinmeyen sahiplik hata durumunda kapalı kalmaya devam eder. Yalıtılmış sahipler hiçbir zaman daha düşük öncelikli bir kimlik bilgisi kaynağına geçmez.

## Çıkış zamanında ekleme (gözcü değerleri)

SecretRef'ler tarafından desteklenen model sağlayıcısı kimlik bilgileri için OpenClaw, model kimlik doğrulaması çözümlenirken opak ve işlem yerel bir gözcü değeri oluşturur. Bu nedenle kimlik doğrulama depolaması, akış seçenekleri, SDK yapılandırması, günlükler, hata nesneleri ve çalışma zamanı iç gözlemlerinin çoğu sağlayıcı kimlik bilgisi yerine `oc-sent-v1-...` gibi bir değer görür. Korumalı model fetch işlemi ve yönetilen yerel sağlayıcı sistem durumu yoklamaları, bilinen gözcü değerlerini her istek işlemden ayrılmadan hemen önce URL ve üstbilgi değerlerinde değiştirir.

Bilinmeyen gözcü değeri biçimli değerler, ağ etkinliğinden önce hata durumunda kapalı kalır. OpenClaw, çözümlenmemiş bir gözcü değerini sağlayıcıya iletmek yerine isteği göndermeyi reddeder. Çözümlenmiş gizli değerler de katmanlı savunma önlemi olarak tam değerli günlük gizleme için kaydedilir.

Sağlayıcı bağdaştırıcıları, SDK'larının desteklediği en son ekleme noktasını kullanır:

- Özel fetch seçeneğine sahip SDK'lar OpenClaw'ın korumalı fetch işlevini alır; böylece SDK gözcü değerini korur.
- Özel fetch seçeneği olmayan SDK'lar, istemci oluşturulmadan hemen önce gözcü değerini açar. Plugin'e ait sağlayıcı akışları ve aracı düzenekleri, bu aktarımlar OpenClaw'ın korumalı fetch işlevini paylaşmadığı için son çekirdeğe ait aktarım noktasında gözcü değerini açar.

Gözcü değerleri, model çağrısı zinciri genelinde düz metne maruz kalmayı azaltır ancak işlem yalıtımı sağlamaz. Gerçek değer aynı işlemin belleğinde bulunmaya devam eder ve son bağdaştırıcı sınırında görünür. SecretRef'ler aracılığıyla yapılandırılmamış düz metin ortam kimlik bilgileri bu mekanizmanın dışındadır ve düz metin olarak kalır.

Olay müdahalesi veya uyumluluk sorunlarını giderme sırasında gözcü değeri oluşturmayı devre dışı bırakmak için `OPENCLAW_SECRET_SENTINELS=off` değerini ayarlayın (`0` veya `false` değerlerini de büyük/küçük harfe duyarsız olarak kabul eder). Acil durdurma anahtarı, tam değerli gizleme kaydını devre dışı bırakmaz.

## Aracı erişim sınırı

SecretRef'ler kimlik bilgilerinin yapılandırmada ve oluşturulan model dosyalarında kalıcı olarak saklanmasını önler ancak işlem yalıtımı sınırı değildir. Aracının okuyabildiği bir yolda diskte bırakılan düz metin kimlik bilgisi, API düzeyindeki gizlemeyi atlayarak dosya veya kabuk araçları üzerinden okunabilir olmaya devam eder.

Aracı tarafından erişilebilen dosyaların kapsam dahilinde olduğu üretim dağıtımlarında, taşıma işlemini yalnızca aşağıdakilerin tümü sağlandığında tamamlanmış sayın:

- Desteklenen kimlik bilgileri düz metin değerler yerine SecretRef'leri kullanır.
- Eski düz metin kalıntıları `openclaw.json`, `auth-profiles.json`, `.env` ve oluşturulan `models.json` dosyalarından temizlenir.
- `openclaw secrets audit --check` taşıma işleminden sonra temizdir.
- Desteklenmeyen veya dönüşümlü olarak yenilenen kalan kimlik bilgileri işletim sistemi yalıtımı, kapsayıcı yalıtımı veya harici bir kimlik bilgisi vekil sunucusuyla korunur.

Denetleme/yapılandırma/uygulama iş akışının yalnızca kolaylık sağlayan bir yardımcı değil, güvenlik taşıması kapısı olmasının nedeni budur.

<Warning>
SecretRef'ler, okunabilen herhangi bir dosyayı güvenli hâle getirmez. Yedeklemeler, kopyalanmış yapılandırmalar, eski oluşturulmuş model katalogları ve desteklenmeyen kimlik bilgisi sınıfları; silinene, aracı güven sınırının dışına taşınana veya ayrı olarak yalıtılana kadar üretim gizli değerleri olarak kalır.
</Warning>

## Etkin yüzey filtreleme

SecretRef'ler yalnızca fiilen etkin yüzeylerde doğrulanır:

- **Etkin yüzeyler**: Eşlenen ve yalıtılabilir sahiplerdeki yeniden denenebilir hatalar soğuk veya bayat bozulma durumuna girer. Katı, hata durumunda kapalı kalan, Gateway için gerekli veya eşlenmemiş hatalar başlatmayı/yeniden yüklemeyi engeller.
- **Etkin olmayan yüzeyler**: Çözümlenmemiş referanslar başlatmayı/yeniden yüklemeyi engellemez; ölümcül olmayan bir `SECRETS_REF_IGNORED_INACTIVE_SURFACE` tanılaması yayınlar.

<Accordion title="Etkin olmayan yüzey örnekleri">
- Devre dışı bırakılmış kanal/hesap girdileri.
- Etkinleştirilmiş hiçbir hesabın devralmadığı üst düzey kanal kimlik bilgileri.
- Devre dışı bırakılmış araç/özellik yüzeyleri.
- `tools.web.search.provider` tarafından seçilmeyen web arama sağlayıcısına özgü anahtarlar. Otomatik modda (sağlayıcı ayarlanmamışken) otomatik algılama için bir anahtar çözümlenene kadar anahtarlara öncelik sırasına göre başvurulur; seçimden sonra seçilmeyen sağlayıcı anahtarları etkin değildir.
- Korumalı alan SSH kimlik doğrulama malzemesi (`agents.defaults.sandbox.ssh.identityData`, `certificateData`, `knownHostsData` ve aracı başına geçersiz kılmalar), yalnızca varsayılan aracı veya etkinleştirilmiş bir aracı için geçerli korumalı alan arka ucu `ssh` olduğunda ve korumalı alan modu `off` olmadığında etkindir.
- `gateway.remote.token` / `gateway.remote.password` SecretRef'leri aşağıdakilerden herhangi biri geçerliyse etkindir:
  - `gateway.mode=remote`
  - `gateway.remote.url` yapılandırılmıştır
  - `gateway.tailscale.mode`, `serve` veya `funnel` değeridir
  - Bu uzak yüzeylerin olmadığı yerel modda: belirteç kimlik doğrulaması geçerli olabiliyorsa ve hiçbir ortam/kimlik doğrulama belirteci yapılandırılmamışsa `gateway.remote.token` etkindir; parola kimlik doğrulaması geçerli olabiliyorsa ve hiçbir ortam/kimlik doğrulama parolası yapılandırılmamışsa yalnızca `gateway.remote.password` etkindir.
- `OPENCLAW_GATEWAY_TOKEN` ayarlandığında `gateway.auth.token` SecretRef'i başlatma kimlik doğrulaması çözümlemesi için etkin değildir, çünkü bu çalışma zamanı için ortam belirteci girdisi önceliklidir.

</Accordion>

## Gateway kimlik doğrulama yüzeyi tanılamaları

`gateway.auth.token`, `gateway.auth.password`, `gateway.remote.token` veya `gateway.remote.password` üzerinde bir SecretRef ayarlandığında, Gateway başlatma/yeniden yükleme günlükleri yüzey durumunu `SECRETS_GATEWAY_AUTH_SURFACE` kodu altında kaydeder:

- `active`: SecretRef geçerli kimlik doğrulama yüzeyinin parçasıdır ve çözümlenmelidir.
- `inactive`: başka bir kimlik doğrulama yüzeyi önceliklidir veya uzak kimlik doğrulama devre dışıdır/etkin değildir.

Günlük girdisi, etkin yüzey ilkesinin kullandığı nedeni içerir.

## İlk katılım referansı ön denetimi

Etkileşimli ilk katılım sırasında SecretRef depolaması seçildiğinde, kaydetmeden önce ön denetim doğrulaması çalıştırılır:

- Ortam referansları: ortam değişkeni adını doğrular ve kurulum sırasında boş olmayan bir değerin görünür olduğunu onaylar.
- Sağlayıcı referansları (`file` veya `exec`): sağlayıcı seçimini doğrular, `id` değerini çözümler ve çözümlenen değer türünü denetler.
- Hızlı başlangıç akışı: `gateway.auth.token` zaten bir SecretRef olduğunda ilk katılım, aynı hızlı hata kapısını kullanarak yoklama/pano önyüklemesinden önce onu (`env`, `file` ve `exec` referansları için) çözümler.

Doğrulama hatası, hatayı gösterir ve yeniden denemenize olanak tanır.

## SecretRef sözleşmesi

Her yerde tek nesne biçimi:

```json5
{ source: "env" | "file" | "exec", provider: "default", id: "..." }
```

<Tabs>
  <Tab title="env">
    ```json5
    { source: "env", provider: "default", id: "OPENAI_API_KEY" }
    ```

    SecretInput alanlarında kısa biçimli dizgiler de kabul edilir:

    ```json5
    "${OPENAI_API_KEY}"
    "$OPENAI_API_KEY"
    ```

    Doğrulama:

    - `provider`, `^[a-z][a-z0-9_-]{0,63}$` ile eşleşmelidir
    - `id`, `^[A-Z][A-Z0-9_]{0,127}$` ile eşleşmelidir

  </Tab>
  <Tab title="file">
    ```json5
    { source: "file", provider: "filemain", id: "/providers/openai/apiKey" }
    ```

    Doğrulama:

    - `provider`, `^[a-z][a-z0-9_-]{0,63}$` ile eşleşmelidir
    - `id`, mutlak bir JSON işaretçisi (`/...`) veya `singleValue` sağlayıcıları için `value` sabit değeri olmalıdır
    - Segmentlerde RFC 6901 kaçış karakterleri: `~`, `~0` olur; `/`, `~1` olur

  </Tab>
  <Tab title="exec">
    ```json5
    { source: "exec", provider: "vault", id: "providers/openai/apiKey#value" }
    ```

    Doğrulama:

    - `provider`, `^[a-z][a-z0-9_-]{0,63}$` ile eşleşmelidir
    - `id`, `^[A-Za-z0-9][A-Za-z0-9._:/#-]{0,255}$` ile eşleşmelidir (`secret#json_key` gibi seçicileri destekler)
    - `id`, eğik çizgiyle ayrılmış yol segmentleri olarak `.` veya `..` içermemelidir (örneğin `a/../b` reddedilir)

  </Tab>
</Tabs>

## Sağlayıcı yapılandırması

Sağlayıcıları `secrets.providers` altında tanımlayın:

```json5
{
  secrets: {
    providers: {
      default: { source: "env" },
      filemain: {
        source: "file",
        path: "~/.openclaw/secrets.json",
        mode: "json", // veya "singleValue"
      },
      vault: {
        source: "exec",
        command: "/usr/local/bin/openclaw-vault-resolver",
        args: ["--profile", "prod"],
        passEnv: ["PATH", "VAULT_ADDR"],
        jsonOnly: true,
      },
      "team-secrets": {
        source: "exec",
        pluginIntegration: {
          pluginId: "acme-secrets",
          integrationId: "secret-store",
        },
      },
    },
    defaults: {
      env: "default",
      file: "filemain",
      exec: "vault",
    },
  },
}
```

<Accordion title="Ortam sağlayıcısı">
- `allowlist` aracılığıyla isteğe bağlı tam ad izin listesi.
- Eksik veya boş ortam değerleri çözümlemeyi başarısız kılar.

</Accordion>

<Accordion title="Dosya sağlayıcısı">
- `path` konumundaki yerel dosyayı okur.
- `mode: "json"` (varsayılan), bir JSON nesnesi yükü bekler ve `id` değerini JSON işaretçisi olarak çözümler.
- `mode: "singleValue"`, `"value"` referans kimliğini bekler ve ham dosya içeriğini döndürür (sondaki yeni satır kaldırılır).
- Yol, sahiplik/izin denetimlerinden geçmelidir; `timeoutMs` (varsayılan 5000) ve `maxBytes` (varsayılan 1 MiB) okumayı sınırlar.
- Windows'ta hata durumunda kapalı kalır: yol için ACL doğrulaması kullanılamıyorsa çözümleme başarısız olur. Yalnızca güvenilir yollar için denetimi atlamak üzere bu sağlayıcıda `allowInsecurePath: true` değerini ayarlayın.

</Accordion>

<Accordion title="Exec sağlayıcısı">
- Yapılandırılmış mutlak ikili dosya yolunu kabuk kullanmadan doğrudan çalıştırır.
- Varsayılan olarak `command` normal bir dosya olmalıdır, sembolik bağlantı olmamalıdır. Sembolik bağlantı komut yollarına (örneğin Homebrew yönlendirmelerine) izin vermek için `allowSymlinkCommand: true` ayarını etkinleştirin ve yalnızca paket yöneticisi yollarının uygun sayılması için bunu `trustedDirs` (örneğin `["/opt/homebrew"]`) ile birlikte kullanın.
- `timeoutMs` (varsayılan 5000), `noOutputTimeoutMs` (varsayılan olarak `timeoutMs` değerine eşittir), `maxOutputBytes` (varsayılan 1 MiB), `env`/`passEnv` izin listesi ve `trustedDirs` desteklenir.
- `jsonOnly` varsayılan olarak `true` değerini kullanır. `jsonOnly: false` ve istenen tek bir kimlik olduğunda, düz JSON olmayan stdout çıktısı bu kimliğin değeri olarak kabul edilir.
- Windows'ta güvenli biçimde başarısız olur: komut yolu için ACL doğrulaması kullanılamıyorsa çözümleme başarısız olur. Yalnızca güvenilen yollar için denetimi atlamak üzere ilgili sağlayıcıda `allowInsecurePath: true` ayarını etkinleştirin.
- Plugin tarafından yönetilen exec sağlayıcıları, kopyalanmış bir `command`/`args` yerine `pluginIntegration` kullanabilir. OpenClaw, başlatma/yeniden yükleme sırasında geçerli komut ayrıntılarını yüklü Plugin manifestinden çözümler; Plugin devre dışı bırakılmış, kaldırılmış veya güvenilmeyen durumdaysa ya da artık entegrasyonu bildirmiyorsa ilgili sağlayıcıdaki etkin SecretRef'ler güvenli biçimde başarısız olur.

İstek yükü (stdin):

```json
{ "protocolVersion": 1, "provider": "vault", "ids": ["providers/openai/apiKey"] }
```

Yanıt yükü (stdout):

```jsonc
{ "protocolVersion": 1, "values": { "providers/openai/apiKey": "<openai-api-key>" } } // pragma: allowlist secret
```

Kimlik başına isteğe bağlı hatalar:

```json
{
  "protocolVersion": 1,
  "values": {},
  "errors": { "providers/openai/apiKey": { "code": "NOT_FOUND" } }
}
```

`code`, isteğe bağlı ve makine tarafından okunabilir bir tanılama bilgisidir. OpenClaw, tanınan
`NOT_FOUND` ve `AMBIGUOUS_DUPLICATE_KEY` kodlarını sağlayıcı ve ref kimliğiyle birlikte görüntüler. `message` gibi diğer
kodlar ve serbest biçimli alanlar, protocol-v1 uyumluluğu için kabul edilir
ancak çözümleyici çıktısı kimlik bilgisi materyali içerebileceğinden görüntülenmez.

</Accordion>

## Dosya tabanlı API anahtarları

Yapılandırmanın `env` bloğuna `file:...` dizeleri koymayın. Bu blok değişmezdir ve geçersiz kılma yapmaz; bu nedenle `file:...` burada hiçbir zaman çözümlenmez.

Bunun yerine desteklenen bir kimlik bilgisi alanında dosya SecretRef'i kullanın:

```json5
{
  secrets: {
    providers: {
      xai_key_file: {
        source: "file",
        path: "~/.openclaw/secrets/xai-api-key.txt",
        mode: "singleValue",
      },
    },
  },
  models: {
    providers: {
      xai: {
        apiKey: { source: "file", provider: "xai_key_file", id: "value" },
      },
    },
  },
}
```

`mode: "singleValue"` için SecretRef `id`, `"value"` değeridir. `mode: "json"` için `"/providers/xai/apiKey"` gibi mutlak bir JSON işaretçisi kullanın.

SecretRef kabul eden alanlar için [SecretRef Kimlik Bilgisi Yüzeyi](/tr/reference/secretref-credential-surface) bölümüne bakın.

## Exec entegrasyonu örnekleri

Hizmet hesaplarını, paketle birlikte gelen agent becerisini ve sorun gidermeyi kapsayan özel 1Password kılavuzu için [1Password](/tr/gateway/1password) bölümüne bakın.

<AccordionGroup>
  <Accordion title="1Password CLI">
    ```json5
    {
      secrets: {
        providers: {
          onepassword_openai: {
            source: "exec",
            command: "/opt/homebrew/bin/op",
            allowSymlinkCommand: true, // required for Homebrew symlinked binaries
            trustedDirs: ["/opt/homebrew"],
            args: ["read", "op://Personal/OpenClaw QA API Key/password"],
            passEnv: ["HOME"],
            jsonOnly: false,
          },
        },
      },
      models: {
        providers: {
          openai: {
            baseUrl: "https://api.openai.com/v1",
            models: [{ id: "gpt-5", name: "gpt-5" }],
            apiKey: { source: "exec", provider: "onepassword_openai", id: "value" },
          },
        },
      },
    }
    ```
  </Accordion>
  <Accordion title="Bitwarden Secrets Manager (`bws`)">
    SecretRef kimliklerini Bitwarden Secrets Manager öğe anahtarlarıyla eşlemek için bir çözümleyici sarmalayıcısı kullanın. Depo `scripts/secrets/openclaw-bws-resolver.mjs` dosyasını içerir; bu dosyayı Gateway'i çalıştıran ana makinedeki mutlak ve güvenilen bir yola yükleyin veya kopyalayın.

    Gereksinimler:

    - Bitwarden Secrets Manager CLI (`bws`) Gateway ana makinesinde yüklü olmalıdır.
    - `BWS_ACCESS_TOKEN`, Gateway hizmeti tarafından kullanılabilir olmalıdır.
    - `PATH` çözümleyiciye aktarılmalı veya `BWS_BIN`, mutlak `bws` ikili dosya yoluna ayarlanmalıdır.
    - Kendi barındırdığınız bir Bitwarden örneği kullanılırken ortamda `BWS_SERVER_URL` ayarlanmalıdır.

    ```json5
    {
      secrets: {
        providers: {
          bws: {
            source: "exec",
            command: "/usr/local/bin/openclaw-bws-resolver.mjs",
            passEnv: ["BWS_ACCESS_TOKEN", "BWS_SERVER_URL", "PATH", "BWS_BIN"],
            jsonOnly: true,
          },
        },
      },
      models: {
        providers: {
          openai: {
            baseUrl: "https://api.openai.com/v1",
            models: [{ id: "gpt-5", name: "gpt-5" }],
            apiKey: {
              source: "exec",
              provider: "bws",
              id: "openclaw/providers/openai/apiKey",
            },
          },
        },
      },
    }
    ```

    Çözümleyici, istenen kimlikleri toplu olarak işler, `bws secret list` komutunu çalıştırır ve eşleşen gizli `key` alanlarının değerlerini döndürür. `openclaw/providers/openai/apiKey` gibi exec SecretRef kimliği sözleşmesini karşılayan anahtarlar kullanın; alt çizgi içeren ortam değişkeni tarzı anahtarlar, çözümleyici çalıştırılmadan önce reddedilir. Birden fazla görünür Bitwarden gizli bilgisi istenen anahtarı paylaşıyorsa çözümleyici tahminde bulunmak yerine bu kimliği belirsiz olarak başarısız sayar. Yapılandırmayı güncelledikten sonra çözümleyici yolunu doğrulayın:

    ```bash
    openclaw secrets audit --allow-exec
    ```

  </Accordion>
  <Accordion title="HashiCorp Vault CLI">
    ```json5
    {
      secrets: {
        providers: {
          vault_openai: {
            source: "exec",
            command: "/opt/homebrew/bin/vault",
            allowSymlinkCommand: true, // required for Homebrew symlinked binaries
            trustedDirs: ["/opt/homebrew"],
            args: ["kv", "get", "-field=OPENAI_API_KEY", "secret/openclaw"],
            passEnv: ["VAULT_ADDR", "VAULT_TOKEN"],
            jsonOnly: false,
          },
        },
      },
      models: {
        providers: {
          openai: {
            baseUrl: "https://api.openai.com/v1",
            models: [{ id: "gpt-5", name: "gpt-5" }],
            apiKey: { source: "exec", provider: "vault_openai", id: "value" },
          },
        },
      },
    }
    ```
  </Accordion>
  <Accordion title="password-store (`pass`)">
    SecretRef kimliklerini doğrudan `pass` girdileriyle eşlemek için küçük bir çözümleyici sarmalayıcısı kullanın. Bunu, exec sağlayıcınızın yol denetimlerinden geçen mutlak bir yola, örneğin `/usr/local/bin/openclaw-pass-resolver` konumuna, çalıştırılabilir dosya olarak kaydedin. `#!/usr/bin/env node` shebang'i `node` yolunu çözümleyici işleminin `PATH` değerinden çözümler; bu nedenle `passEnv` içine `PATH` ekleyin. `pass`, bu `PATH` üzerinde değilse üst ortamda `PASS_BIN` ayarını yapın ve bunu `passEnv` içine de ekleyin:

    ```js
    #!/usr/bin/env node
    const { spawnSync } = require("node:child_process");

    let stdin = "";
    process.stdin.setEncoding("utf8");
    process.stdin.on("data", (chunk) => {
      stdin += chunk;
    });
    process.stdin.on("error", (err) => {
      process.stderr.write(`${err.message}\n`);
      process.exit(1);
    });
    process.stdin.on("end", () => {
      let request;
      try {
        request = JSON.parse(stdin || "{}");
      } catch (err) {
        process.stderr.write(`Failed to parse request: ${err.message}\n`);
        process.exit(1);
      }

      const passBin = process.env.PASS_BIN || "pass";
      const values = {};
      const errors = {};

      for (const id of request.ids ?? []) {
        const result = spawnSync(passBin, ["show", id], { encoding: "utf8" });
        if (result.status === 0) {
          values[id] = result.stdout.split(/\r?\n/, 1)[0] ?? "";
        } else {
          errors[id] = { message: (result.stderr || `pass exited ${result.status}`).trim() };
        }
      }

      process.stdout.write(JSON.stringify({ protocolVersion: 1, values, errors }));
    });
    ```

    Ardından exec sağlayıcısını yapılandırın ve `apiKey` değerini `pass` girdi yoluna yönlendirin:

    ```json5
    {
      secrets: {
        providers: {
          pass_store: {
            source: "exec",
            command: "/usr/local/bin/openclaw-pass-resolver",
            passEnv: ["PATH", "HOME", "GNUPGHOME", "GPG_TTY", "PASSWORD_STORE_DIR", "PASS_BIN"],
            jsonOnly: true,
          },
        },
      },
      models: {
        providers: {
          openai: {
            baseUrl: "https://api.openai.com/v1",
            models: [{ id: "gpt-5", name: "gpt-5" }],
            apiKey: {
              source: "exec",
              provider: "pass_store",
              id: "openclaw/providers/openai/apiKey",
            },
          },
        },
      },
    }
    ```

    Gizli bilgiyi `pass` girdisinin ilk satırında tutun veya bunun yerine tam `pass show` çıktısını döndürmek üzere sarmalayıcıyı özelleştirin. Yapılandırmayı güncelledikten sonra hem statik denetimi hem de exec çözümleyici yolunu doğrulayın:

    ```bash
    openclaw secrets audit --check
    openclaw secrets audit --allow-exec
    ```

  </Accordion>
  <Accordion title="sops">
    ```json5
    {
      secrets: {
        providers: {
          sops_openai: {
            source: "exec",
            command: "/opt/homebrew/bin/sops",
            allowSymlinkCommand: true, // required for Homebrew symlinked binaries
            trustedDirs: ["/opt/homebrew"],
            args: ["-d", "--extract", '["providers"]["openai"]["apiKey"]', "/path/to/secrets.enc.json"],
            passEnv: ["SOPS_AGE_KEY_FILE"],
            jsonOnly: false,
          },
        },
      },
      models: {
        providers: {
          openai: {
            baseUrl: "https://api.openai.com/v1",
            models: [{ id: "gpt-5", name: "gpt-5" }],
            apiKey: { source: "exec", provider: "sops_openai", id: "value" },
          },
        },
      },
    }
    ```
  </Accordion>
</AccordionGroup>

## MCP sunucusu ortam değişkenleri

`plugins.entries.acpx.config.mcpServers` aracılığıyla yapılandırılan MCP sunucusu ortam değişkenleri SecretInput kabul ederek API anahtarlarının ve token'ların düz metin yapılandırmanın dışında tutulmasını sağlar:

```json5
{
  plugins: {
    entries: {
      acpx: {
        enabled: true,
        config: {
          mcpServers: {
            github: {
              command: "npx",
              args: ["-y", "@modelcontextprotocol/server-github"],
              env: {
                GITHUB_PERSONAL_ACCESS_TOKEN: {
                  source: "env",
                  provider: "default",
                  id: "MCP_GITHUB_PAT",
                },
              },
            },
          },
        },
      },
    },
  },
}
```

Düz metin dize değerleri çalışmaya devam eder. `${MCP_SERVER_API_KEY}` gibi ortam şablonu referansları ve SecretRef nesneleri, MCP sunucusu işlemi başlatılmadan önce Gateway etkinleştirmesi sırasında çözümlenir. Diğer SecretRef yüzeylerinde olduğu gibi çözümlenemeyen referanslar, yalnızca `acpx` Plugin'i fiilen etkin olduğunda etkinleştirmeyi engeller.

## Sandbox SSH kimlik doğrulama materyali

Temel `ssh` sandbox arka ucu, SSH kimlik doğrulama materyali için SecretRef'leri de destekler:

```json5
{
  agents: {
    defaults: {
      sandbox: {
        mode: "all",
        backend: "ssh",
        ssh: {
          target: "user@gateway-host:22",
          identityData: { source: "env", provider: "default", id: "SSH_IDENTITY" },
          certificateData: { source: "env", provider: "default", id: "SSH_CERTIFICATE" },
          knownHostsData: { source: "env", provider: "default", id: "SSH_KNOWN_HOSTS" },
        },
      },
    },
  },
}
```

Çalışma zamanı davranışı:

- OpenClaw, bu referansları her SSH çağrısında tembel olarak değil, korumalı alan etkinleştirilirken çözümler.
- Çözümlenen değerler, kısıtlayıcı dosya izinleriyle (`0o600`) geçici bir dizine yazılır ve oluşturulan SSH yapılandırmasında kullanılır.
- Geçerli korumalı alan arka ucu `ssh` değilse (veya korumalı alan modu `off` ise), bu referanslar etkin olmayan durumda kalır ve başlatmayı engellemez.

## Desteklenen kimlik bilgisi yüzeyi

Standart olarak desteklenen ve desteklenmeyen kimlik bilgileri [SecretRef Kimlik Bilgisi Yüzeyi](/tr/reference/secretref-credential-surface) bölümünde listelenmiştir.

<Note>
Çalışma zamanında oluşturulan veya döndürülen kimlik bilgileri ile OAuth yenileme malzemeleri, salt okunur SecretRef çözümlemesinin dışında özellikle tutulur.
</Note>

## Gerekli davranış ve öncelik

- Referansı olmayan alan: değişmez.
- Referansı olan alan: etkinleştirme sırasında etkin yüzeylerde gereklidir.
- Hem düz metin hem de referans varsa desteklenen öncelik yollarında referans önceliklidir.
- Karartma belirteci `__OPENCLAW_REDACTED__`, dahili yapılandırma karartma/geri yükleme işlemleri için ayrılmıştır ve gönderilen yapılandırma verilerinde değişmez değer olarak kullanılması reddedilir.

Uyarı ve denetim sinyalleri:

- `SECRETS_REF_OVERRIDES_PLAINTEXT` (çalışma zamanı uyarısı)
- `REF_SHADOWED` (`auth-profiles.json` kimlik bilgileri `openclaw.json` referanslarından öncelikli olduğunda denetim bulgusu)

Google Chat `serviceAccount`, satır içi JSON veya bir SecretRef kabul eder. Doctor, kullanımdan kaldırılmış kardeş `serviceAccountRef` alanını, ayarlanmamışsa bu standart alana taşır.

## Etkinleştirme tetikleyicileri

Gizli bilgi etkinleştirmesi şu durumlarda çalışır:

- Başlatma (ön kontrol ve son etkinleştirme)
- Yapılandırmayı yeniden yükleme, çalışırken uygulama yolu
- Yapılandırmayı yeniden yükleme, yeniden başlatma denetimi yolu
- `secrets.reload` aracılığıyla elle yeniden yükleme
- Gateway yapılandırma yazma RPC ön kontrolü (`config.set` / `config.apply` / `config.patch`); düzenlemeleri kalıcılaştırmadan önce gönderilen yapılandırma yükündeki etkin yüzey SecretRef'lerini doğrular

Etkinleştirme sözleşmesi:

- Başarı durumunda anlık görüntü atomik olarak değiştirilir.
- Katı bir başlatma hatası Gateway'in başlatılmasını iptal eder.
- Soğuk başlatma sırasında, eşlenmiş ve yalıtılabilir Gateway dışı bir sahip için yeniden denenebilir bir çözümleme hatası oluşursa anlık görüntü, yalnızca tam olarak o sahip yapılandırılmış-kullanılamaz durumda olacak şekilde yayımlanabilir. Bu sahibe yönelik istekler `SECRET_SURFACE_UNAVAILABLE` ile başarısız olur; model sağlayıcısı sahipleri, açık bir referans başarısız olduktan sonra ortam veya kimlik doğrulama profili kimlik bilgilerine geri dönmez.
- Yeniden yükleme ve yeniden başlatma denetimi, uygun eşlenmiş sahipleri yalıtır. Sağlayıcı tanımları ve gizli olmayan eksiksiz sahip sözleşmesi değişmemiş olan, kimliği değişmeyen referanslar tam olarak bilinen son iyi değerlerini bayat olarak korur; değiştirilen veya yeni yapılandırılan çözümlenmemiş referanslar yalnızca ilgili sahip için soğuk olarak yayımlanır. Katı bir yeniden yükleme hatası, daha önce etkin olan anlık görüntüyü korur.
- `config.set`, `config.apply` ve `config.patch`, yalıtılabilir sahipler için sözdizimsel olarak geçerli çözümlenmemiş referansları kabul eder ve karartılmış bir `degradedSecretOwners` raporu döndürür. Gateway giriş kimlik doğrulaması, yapısal olarak geçersiz yapılandırma veya çözümlenmiş değerler, ilke ihlalleri ve bilinmeyen sahipler disk değişikliğinden önce yine reddedilir.
- Sağlıklı kardeş sahipler, başka bir sahip soğuk veya bayat olsa bile normal şekilde çözümlenir ve yayımlanır.
- Giden bir yardımcı/araç çağrısına çağrı başına açık bir kanal belirteci sağlamak SecretRef etkinleştirmesini tetiklemez; etkinleştirme noktaları başlatma, yeniden yükleme ve açık `secrets.reload` olarak kalır.

## Bozulma ve kurtarma sinyalleri

Sağlıklı bir durumdan sonra yeniden yükleme sırasında etkinleştirme başarısız olduğunda OpenClaw, tek seferlik sistem olayları ve günlük kodları yayımlayarak bozulmuş gizli bilgiler durumuna girer:

- `SECRETS_RELOADER_DEGRADED`
- `SECRETS_RELOADER_RECOVERED`

Davranış:

- Bozulmuş: sağlıklı sahipler yenilenir, bayat sahipler bilinen son iyi değeri korur ve soğuk sahipler kullanılamaz durumda kalır.
- Kurtarıldı: bir sonraki başarılı etkinleştirmeden sonra bir kez yayımlanır.
- Zaten bozulmuş durumdayken yinelenen hatalar günlük uyarıları oluşturur ancak olayı yeniden yayımlamaz.
- Katı bir başlatma hatası hiçbir zaman bozulma olayı yayımlamaz çünkü çalışma zamanı hiçbir zaman etkinleşmemiştir. Soğuk sahiplerle başarılı bir başlatma, sahip bozulmasını günlüğe kaydeder ancak yeniden yükleyici olayı yayımlamaz.
- Referans kapsamlı başlatma ve yeniden yükleme hataları, etkilenen her sahip için yapılandırılmış bir `SECRETS_DEGRADED` uyarısı yayımlar. Sağlayıcı kapsamlı kesintiler, sağlayıcı hatasını sahip başına yinelemek yerine sağlayıcıyı ve etkilenen sahiplerin eksiksiz listesini içeren tek bir `SECRETS_PROVIDER_DEGRADED` uyarısı yayımlar. Uyarılar karartılmış bir neden, `cold` veya `stale` sahip durumu ve `openclaw secrets reload` yeniden deneme ipucunu içerir. Çözümlenmiş değerleri veya SecretRef kimliklerini hiçbir zaman içermez.
- `openclaw doctor`, soğuk ve bayat sahipleri etkilenen yapılandırma yolları, karartılmış neden ve yeniden deneme yönergeleriyle listeler.

## Komut yolu çözümlemesi

Komut yolları, bir Gateway anlık görüntü RPC'si aracılığıyla desteklenen SecretRef çözümlemesini etkinleştirebilir. İki genel davranış geçerlidir:

<Tabs>
  <Tab title="Katı komut yolları">
    Örneğin `openclaw memory` uzak bellek yolları ve uzak paylaşılan gizli bilgi referanslarına ihtiyaç duyduğunda `openclaw qr --remote`. Etkin anlık görüntüden okurlar ve gerekli bir SecretRef kullanılamadığında hızla başarısız olurlar.
  </Tab>
  <Tab title="Salt okunur komut yolları">
    Örneğin `openclaw status`, `openclaw status --all`, `openclaw channels status`, `openclaw channels resolve`, `openclaw security audit` ve salt okunur doctor/yapılandırma onarım akışları. Bunlar da etkin anlık görüntüyü tercih eder ancak hedeflenen bir SecretRef kullanılamadığında işlemi iptal etmek yerine kısıtlı çalışır.

    Salt okunur davranış:

    - Gateway çalışırken bu komutlar önce etkin anlık görüntüden okur.
    - Gateway çözümlemesi eksikse veya Gateway kullanılamıyorsa ilgili komut yüzeyi için hedefli bir yerel geri dönüş denerler.
    - Hedeflenen bir SecretRef hâlâ kullanılamıyorsa komut, kısıtlı salt okunur çıktıyla ve referansın yapılandırılmış ancak bu komut yolunda kullanılamaz olduğunu belirten açık bir tanılamayla devam eder.
    - Bu kısıtlı davranış yalnızca komuta özeldir; çalışma zamanı başlatma, yeniden yükleme veya gönderme/kimlik doğrulama yollarını zayıflatmaz.

  </Tab>
</Tabs>

Diğer notlar:

- Arka uç gizli bilgisi döndürüldükten sonra anlık görüntünün yenilenmesi `openclaw secrets reload` tarafından gerçekleştirilir.
- Bu komut yollarının kullandığı Gateway RPC yöntemi: `secrets.resolve`.

## Denetim ve yapılandırma iş akışı

Varsayılan operatör akışı:

<Steps>
  <Step title="Geçerli durumu denetleyin">
    ```bash
    openclaw secrets audit --check
    ```
  </Step>
  <Step title="SecretRef'leri yapılandırın ve uygulayın">
    ```bash
    openclaw secrets configure --apply
    ```
  </Step>
  <Step title="Yeniden denetleyin">
    ```bash
    openclaw secrets audit --check
    ```
  </Step>
</Steps>

Yeniden denetim temiz sonuçlanana kadar geçişi tamamlanmış kabul etmeyin. Denetim, depolanan düz metin değerleri hâlâ bildiriyorsa çalışma zamanı API'leri karartılmış değerler döndürse bile ajan erişimi riski devam eder.

`configure` sırasında uygulamak yerine bir plan kaydederseniz yeniden denetimden önce bu kayıtlı planı `openclaw secrets apply --from <plan-path>` ile uygulayın.

<AccordionGroup>
  <Accordion title="secrets audit">
    Bulgular şunları içerir:

    - Depolanan düz metin değerler (`openclaw.json`, `auth-profiles.json`, `.env` ve oluşturulan `agents/*/agent/models.json`).
    - Oluşturulan `models.json` girdilerindeki düz metin hassas sağlayıcı başlığı kalıntıları.
    - Çözümlenmemiş referanslar.
    - Öncelik gölgelemesi (`auth-profiles.json` değerlerinin `openclaw.json` referanslarından öncelikli olması).
    - Eski kalıntılar (`auth.json`, OAuth hatırlatıcıları).

    Exec notu: Denetim, komut yan etkilerinden kaçınmak için varsayılan olarak exec SecretRef çözümlenebilirlik denetimlerini atlar. Denetim sırasında exec sağlayıcılarını çalıştırmak için `openclaw secrets audit --allow-exec` kullanın.

    Başlık kalıntısı notu: Hassas sağlayıcı başlığı algılaması, ada dayalı sezgisel yöntem kullanır (yaygın kimlik doğrulama/kimlik bilgisi başlığı adları ve `authorization`, `x-api-key`, `token`, `secret`, `password` ve `credential` gibi parçalar).

  </Accordion>
  <Accordion title="secrets configure">
    Şunları yapan etkileşimli yardımcı:

    - Önce `secrets.providers` öğesini yapılandırır (`env`/`file`/`exec`, ekleme/düzenleme/kaldırma).
    - Bir ajan kapsamı için `openclaw.json` içindeki desteklenen gizli bilgi taşıyan alanların yanı sıra `auth-profiles.json` öğesini seçmenizi sağlar.
    - Hedef seçicide doğrudan yeni bir `auth-profiles.json` eşlemesi oluşturabilir.
    - SecretRef ayrıntılarını (`source`, `provider`, `id`) alır.
    - Ön kontrol çözümlemesini çalıştırır ve hemen uygulayabilir.

    Exec notu: `--allow-exec` ayarlanmadıkça ön kontrol, exec SecretRef denetimlerini atlar. Doğrudan `configure --apply` içinden uyguluyorsanız ve plan exec referansları/sağlayıcıları içeriyorsa uygulama adımı için de `--allow-exec` ayarını koruyun.

    Yararlı modlar:

    - `openclaw secrets configure --providers-only`
    - `openclaw secrets configure --skip-provider-setup`
    - `openclaw secrets configure --agent <id>`

    `configure` uygulama varsayılanları:

    - Hedeflenen sağlayıcılar için `auth-profiles.json` içindeki eşleşen statik kimlik bilgilerini temizler.
    - `auth.json` içindeki eski statik `api_key` girdilerini temizler.
    - Geçerli durum ve etkin yapılandırmanın `.env` dosyalarından eşleşen bilinen gizli bilgi satırlarını temizler (iki yol eşleştiğinde yinelenenler kaldırılır).

  </Accordion>
  <Accordion title="secrets apply">
    Kayıtlı bir planı uygulayın:

    ```bash
    openclaw secrets apply --from /tmp/openclaw-secrets-plan.json
    openclaw secrets apply --from /tmp/openclaw-secrets-plan.json --allow-exec
    openclaw secrets apply --from /tmp/openclaw-secrets-plan.json --dry-run
    openclaw secrets apply --from /tmp/openclaw-secrets-plan.json --dry-run --allow-exec
    ```

    Exec notu: `--allow-exec` ayarlanmadıkça deneme çalıştırması exec denetimlerini atlar; `--allow-exec` ayarlanmadıkça yazma modu, exec SecretRef'leri/sağlayıcıları içeren planları reddeder.

    Katı hedef/yol sözleşmesi ayrıntıları ve kesin ret kuralları için [Gizli Bilgileri Uygulama Planı Sözleşmesi](/tr/gateway/secrets-plan-contract) bölümüne bakın.

  </Accordion>
</AccordionGroup>

## Tek yönlü güvenlik ilkesi

<Warning>
OpenClaw, geçmiş düz metin gizli bilgi değerlerini içeren geri alma yedeklerini özellikle yazmaz.
</Warning>

Güvenlik modeli:

- Yazma modundan önce ön kontrol başarılı olmalıdır.
- Kaydetmeden önce çalışma zamanı etkinleştirmesi doğrulanır.
- Uygulama, dosyaları atomik dosya değiştirme yöntemiyle günceller ve hata durumunda mümkün olan en iyi şekilde geri yükler.

## Eski kimlik doğrulama uyumluluğu notları

Statik kimlik bilgileri için çalışma zamanı artık düz metin eski kimlik doğrulama depolamasına bağlı değildir.

- Çalışma zamanı kimlik bilgisi kaynağı, çözümlenmiş bellek içi anlık görüntüdür.
- Eski statik `api_key` girdileri bulunduğunda temizlenir.
- OAuth ile ilgili uyumluluk davranışı ayrı kalır.

## Web kullanıcı arayüzü notu

Bazı SecretInput birleşimlerini ham düzenleyici modunda yapılandırmak, form modundakinden daha kolaydır.

## İlgili

- [Kimlik Doğrulama](/tr/gateway/authentication) - kimlik doğrulama kurulumu
- [CLI: gizli bilgiler](/tr/cli/secrets) - CLI komutları
- [Vault SecretRef'leri](/tr/plugins/vault) - HashiCorp Vault sağlayıcı kurulumu
- [Ortam Değişkenleri](/tr/help/environment) - ortam önceliği
- [SecretRef Kimlik Bilgisi Yüzeyi](/tr/reference/secretref-credential-surface) - kimlik bilgisi yüzeyi
- [Gizli Bilgileri Uygulama Planı Sözleşmesi](/tr/gateway/secrets-plan-contract) - plan sözleşmesi ayrıntıları
- [Güvenlik](/tr/gateway/security) - güvenlik duruşu
