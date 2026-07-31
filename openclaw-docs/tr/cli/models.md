---
read_when:
    - Varsayılan modelleri değiştirmek veya sağlayıcı kimlik doğrulama durumunu görüntülemek istiyorsunuz
    - Kullanılabilir modelleri/sağlayıcıları taramak ve kimlik doğrulama profillerinde hata ayıklamak istiyorsunuz
summary: '`openclaw models` için CLI başvurusu (durum/listeleme/ayarlama/tarama, takma adlar, geri dönüşler, kimlik doğrulama)'
title: Modeller
x-i18n:
    generated_at: "2026-07-26T23:53:59Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: f7405c25694f04afe9c3029a8af64ae3ae7e1bdcf4c4ac31b8b84ff512d6a90e
    source_path: cli/models.md
    workflow: 16
---

# `openclaw models`

Model keşfi, tarama ve yapılandırma (varsayılan model, yedekler, kimlik doğrulama profilleri).

İlgili:

- Sağlayıcılar + modeller: [Modeller](/tr/providers/models)
- Model seçimi kavramları + `/models` eğik çizgi komutu: [Modeller kavramı](/tr/concepts/models)
- Sağlayıcı kimlik doğrulama kurulumu: [Başlarken](/tr/start/getting-started)

## Yaygın komutlar

```bash
openclaw models status
openclaw models list
openclaw models set <model-or-alias>
openclaw models set-image <model-or-alias>
openclaw models scan
```

`status` ve `auth` alt komutları, yapılandırılmış bir agent'ı hedeflemek için `--agent <id>` kabul eder; `list`, `scan`, `aliases` ve `fallbacks`/`image-fallbacks` her zaman yapılandırılmış varsayılan agent'ı kullanır; `set`/`set-image` ise `--agent` seçeneğini doğrudan reddeder. Belirtilmediğinde, `--agent` desteğine sahip komutlar ayarlanmışsa `OPENCLAW_AGENT_DIR` değerini, aksi takdirde yapılandırılmış varsayılan agent'ı kullanır.

### Durum

`openclaw models status`, çözümlenmiş varsayılanı/yedekleri ve kimlik doğrulamaya genel bakışı gösterir. Codex gibi plugin'e ait agent çalışma zamanlarında, sahip plugin'in etkin olup olmadığını ve başlangıç yükü doğrulamasını geçip geçmediğini de denetler. Geçerli kimlik bilgilerine sahip ancak çalışma zamanı kullanılamayan bir rota, `usable` yerine `status: unavailable` bildirir; JSON çıktısı ayrı `authStatus`, `runtimeStatus` ve sınırlandırılmış çalışma zamanı tanılamalarını içerir. Sağlayıcı kullanım anlık görüntüleri mevcut olduğunda OAuth/API anahtarı durumu bölümü, sağlayıcı kullanım pencerelerini ve kota anlık görüntülerini içerir. Geçerli kullanım penceresi sağlayıcıları: Anthropic, GitHub Copilot, Gemini CLI, OpenAI, MiniMax, Xiaomi ve z.ai. Kullanım kimlik doğrulaması, mevcut olduğunda sağlayıcıya özgü kancalardan gelir; aksi takdirde OpenClaw, kimlik doğrulama profilleri, ortam veya yapılandırmadaki eşleşen OAuth/API anahtarı kimlik bilgilerine başvurur.

`--json` çıktısında `auth.providers`, ortam/yapılandırma/depo bilgilerini dikkate alan sağlayıcı genel bakışıdır; `auth.oauth` ise yalnızca kimlik doğrulama deposu profil sağlığıdır.

Seçenekler:

| Bayrak                      | Etki                                                                                                                                   |
| ------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------- |
| `--json`                  | JSON çıktısı; standart çıktı `jq` içine aktarılabilir durumda kalsın diye kimlik doğrulama profili, sağlayıcı ve başlangıç tanılamaları standart hataya gider.                            |
| `--plain`                 | Düz metin çıktısı.                                                                                                                       |
| `--check`                 | Kimlik doğrulamanın süresi dolmak üzereyse/dolmuşsa veya seçili bir agent çalışma zamanı kullanılamıyorsa sıfırdan farklı bir kodla çıkar: `1` = kullanılamıyor/süresi dolmuş/eksik, `2` = süresi dolmak üzere. |
| `--probe`                 | Yapılandırılmış kimlik doğrulama profillerinin canlı yoklaması. Gerçek istekler gönderir; token tüketebilir ve hız sınırlarını tetikleyebilir.                                       |
| `--probe-provider <name>` | Yalnızca bir sağlayıcıyı yokla.                                                                                                                 |
| `--probe-profile <id>`    | Belirli kimlik doğrulama profili kimliklerini yokla (yinelenebilir veya virgülle ayrılmış).                                                                             |
| `--probe-timeout <ms>`    | Yoklama başına zaman aşımı.                                                                                                                       |
| `--probe-concurrency <n>` | Eş zamanlı yoklamalar.                                                                                                                       |
| `--probe-max-tokens <n>`  | Yoklama için azami token sayısı (mümkün olan en iyi şekilde).                                                                                                          |
| `--agent <id>`            | Yapılandırılmış agent kimliği; `OPENCLAW_AGENT_DIR` değerini geçersiz kılar.                                                                                     |

Yoklama satırları kimlik doğrulama profillerinden, ortam kimlik bilgilerinden veya `models.json` kaynağından gelebilir. Yoklama durumu grupları: `ok`, `auth`, `rate_limit`, `billing`, `timeout`, `format`, `unknown`, `no_model`.

Bir yoklama model çağrısına hiç ulaşmadığında beklenebilecek yoklama ayrıntısı/neden kodları:

- `excluded_by_auth_order`: depolanmış bir profil vardır ancak açık `auth.order.<provider>` bunu dışarıda bırakmıştır; bu nedenle yoklama, profili denemek yerine dışlanmayı bildirir.
- `missing_credential`, `invalid_expires`, `expired`, `unresolved_ref`: profil mevcuttur ancak uygun veya çözümlenebilir değildir.
- `ineligible_profile`: profil başka bir nedenle sağlayıcı yapılandırmasıyla uyumsuzdur.
- `no_model`: sağlayıcı kimlik doğrulaması mevcuttur ancak OpenClaw bu sağlayıcı için yoklanabilir bir model adayı çözümleyememiştir.

OpenAI ChatGPT/Codex OAuth sorunlarını giderirken `openclaw models status`, `openclaw models auth list --provider openai` ve `openclaw config get agents.defaults.model --json`, bir agent'ın yerel Codex çalışma zamanı aracılığıyla `openai/*` için kullanılabilir bir `openai` OAuth profiline sahip olup olmadığını doğrulamanın en hızlı yoludur. Bkz. [OpenAI sağlayıcı kurulumu](/tr/providers/openai#check-and-recover-codex-oauth-routing).

### Liste

`openclaw models list` salt okunurdur: yapılandırmayı, kimlik doğrulama profillerini, mevcut katalog durumunu ve sağlayıcıya ait katalog satırlarını okur ancak `models.json` değerini hiçbir zaman yeniden yazmaz.

Seçenekler: `--all` (tam katalog), `--local` (yerel modellerle sınırla), `--provider <id>`, `--json`, `--plain`.

Notlar:

- `Auth` sütunu salt okunurdur. OpenAI gibi sağlayıcıya ait model rotalarında her satırın API/temel URL rotasını, etkin `auth.order` içindeki uygun profillerle, ortam/yapılandırma kimlik bilgileriyle ve çözümlenmiş komut kapsamlı SecretRef'lerle eşleştirir. Somut bir OpenAI satırı, rota ilkesi kullanılamadığında sağlayıcı düzeyindeki kimlik doğrulamayı ödünç almak yerine bilinmiyor durumunda kalır; yalnızca sağlayıcıya yönelik eski denetimler ve diğer sağlayıcılar, sağlayıcı düzeyindeki davranışı korur. Plugin sentetik kimlik doğrulama meta verileri yalnızca çalışma zamanı yeteneğine ilişkin bir ipucudur, yerel hesap kimlik doğrulamasının kanıtı değildir; bu nedenle hesaba bağımlı rotalar, olumlu kayıt defteri kanıtı olmadan bilinmiyor durumunda kalır. Komut; sağlayıcı çalışma zamanını yüklemez, anahtar zinciri sırlarını okumaz, sağlayıcı API'lerini çağırmaz veya kesin yürütme hazırlığını kanıtlamaz.
- `models list --all --provider <id>`, henüz bu sağlayıcıyla kimlik doğrulaması yapmamış olsanız bile plugin manifestlerinden veya paketlenmiş sağlayıcı katalog meta verilerinden gelen, sağlayıcıya ait statik katalog satırlarını içerebilir. Bu satırlar, eşleşen kimlik doğrulama yapılandırılana kadar kullanılamıyor olarak görünmeye devam eder.
- `models list`, sağlayıcı katalog keşfi yavaşken kontrol düzleminin yanıt vermeye devam etmesini sağlar. Varsayılan ve yapılandırılmış görünümler, kısa bir beklemeden sonra yapılandırılmış veya sentetik model satırlarına başvurur ve keşfin arka planda tamamlanmasına izin verir. Tam olarak keşfedilmiş kataloga ihtiyaç duyduğunuzda ve sağlayıcı keşfini beklemeye hazır olduğunuzda `--all` kullanın.
- Geniş `models list --all`, sağlayıcı çalışma zamanı ek kancalarını yüklemeden manifest katalog satırlarını kayıt defteri satırlarının üzerine birleştirir. Sağlayıcıya göre filtrelenmiş hızlı manifest yolları yalnızca `static` olarak işaretlenmiş sağlayıcıları kullanır; `refreshable` olarak işaretlenmiş sağlayıcılar kayıt defteri/önbellek destekli kalır ve manifest satırlarını ek olarak iliştirirken, `runtime` olarak işaretlenmiş sağlayıcılar kayıt defteri/çalışma zamanı keşfinde kalır.
- `models list`, yerel model meta verilerini ve çalışma zamanı sınırlarını birbirinden ayrı tutar. Tablo çıktısında `Ctx`, etkin bir çalışma zamanı sınırı yerel bağlam penceresinden farklı olduğunda `contextTokens/contextWindow` gösterir; bir sağlayıcı bu sınırı sunduğunda JSON satırları `contextTokens` içerir.
- Sağlayıcıya ait rotalarda `models list`, mantıksal bir sağlayıcı/model satırını seçili rotaya yansıtır. `Input` ve `Ctx` yalnızca tam olarak eşleşen fiziksel rota katalog satırından gelir ve açıkça yapılandırılmış mantıksal geçersiz kılmalar en son uygulanır; çözümlenmemiş rota seçimi, kardeş rota meta verilerini ödünç almak yerine bilinmeyen yetenek alanları gösterir.
- `models list --provider <id>`, `moonshot` veya `openai` gibi sağlayıcı kimliğine göre filtreler. `Moonshot AI` gibi etkileşimli sağlayıcı seçicilerindeki görüntüleme etiketlerini kabul etmez.
- Model başvuruları **ilk** `/` üzerinden bölünerek ayrıştırılır. Model kimliği `/` içeriyorsa (OpenRouter tarzı), sağlayıcı önekini ekleyin (örnek: `openrouter/moonshotai/kimi-k2`).
- Sağlayıcıyı belirtmezseniz OpenClaw, girdiyi önce bir takma ad, ardından tam olarak bu model kimliği için benzersiz bir yapılandırılmış sağlayıcı eşleşmesi olarak çözümler ve ancak bundan sonra bir kullanımdan kaldırma uyarısıyla yapılandırılmış varsayılan sağlayıcıya başvurur. Bu sağlayıcı artık yapılandırılmış varsayılan modeli sunmuyorsa OpenClaw, kaldırılmış bir sağlayıcıya ait eskimiş varsayılanı göstermek yerine ilk yapılandırılmış sağlayıcı/model çiftine başvurur.
- `models status`, gizli olmayan yer tutucuları (örneğin `OPENAI_API_KEY`, `secretref-managed`, `minimax-oauth`, `oauth:chutes`, `ollama-local`) sır olarak maskelemek yerine kimlik doğrulama çıktısında `marker(<value>)` gösterebilir.

### Varsayılanı / görüntü modelini ayarlama

```bash
openclaw models set <model-or-alias>
openclaw models set-image <model-or-alias>
```

`set`, `agents.defaults.model.primary` değerini; `set-image` ise `agents.defaults.imageModel.primary` değerini yazar. Her ikisi de `provider/model` veya yapılandırılmış bir takma ad kabul eder. `set`, yeni seçilen model gerektiriyorsa Codex/Copilot çalışma zamanı plugin kurulumlarını da onarır; `set-image` bunu yapmaz. İki komut da `--agent` kabul etmez; her zaman agent varsayılanlarını yazarlar.

### Tarama

`models scan`, OpenRouter'ın herkese açık `:free` kataloğunu okur ve adayları yedek olarak kullanılmak üzere sıralar. Kataloğun kendisi herkese açık olduğundan yalnızca meta veri taramaları OpenRouter anahtarı gerektirmez.

OpenClaw varsayılan olarak araç ve görüntü desteğini canlı model çağrılarıyla yoklamaya çalışır. Yapılandırılmış bir OpenRouter anahtarı yoksa komut yalnızca meta veri çıktısına başvurur ve `:free` modellerinin yoklamalar ve çıkarım için yine de `OPENROUTER_API_KEY` gerektirdiğini açıklar.

Seçenekler:

- `--no-probe` (yalnızca meta veri; yapılandırma/sır araması yapılmaz)
- `--min-params <b>`
- `--max-age-days <days>`
- `--provider <name>`
- `--max-candidates <n>`
- `--timeout <ms>` (katalog isteği ve yoklama başına zaman aşımı)
- `--concurrency <n>`
- `--yes`
- `--no-input`
- `--set-default`
- `--set-image`
- `--json`

`--set-default` ve `--set-image` canlı yoklamalar gerektirir; yalnızca meta veri içeren tarama sonuçları bilgilendirme amaçlıdır ve yapılandırmaya uygulanmaz.

## Takma adlar

```bash
openclaw models aliases list [--json] [--plain]
openclaw models aliases add <alias> <model-or-alias>
openclaw models aliases remove <alias>
```

Takma adlar, model girdisi başına `agents.defaults.models.<key>.alias` olarak depolanır. `add`, önce `<model-or-alias>` değerini standart bir sağlayıcı/model anahtarına çözümler; bu nedenle bir takma adı başka bir takma adla eşlemek zincir oluşturmak yerine hedefini değiştirir.
Takma ad eklemek `agents.defaults.modelPolicy.allow` değerini değiştirmez veya model geçersiz kılmalarını kısıtlamaz.

## Yedekler

```bash
openclaw models fallbacks list [--json] [--plain]
openclaw models fallbacks add <model-or-alias>
openclaw models fallbacks remove <model-or-alias>
openclaw models fallbacks clear
```

`agents.defaults.model.fallbacks` değerini yönetir. `openclaw models image-fallbacks list|add|remove|clear`, aynı alt komut yapısıyla paralel `agents.defaults.imageModel.fallbacks` listesini yönetir.

## Kimlik doğrulama profilleri

```bash
openclaw models auth add
openclaw models auth list [--provider <id>] [--json]
openclaw models auth login --provider <id>
openclaw models auth login --provider openai --profile-id openai:work
openclaw models auth login-github-copilot
openclaw models auth paste-api-key --provider <id>
openclaw models auth setup-token --provider <id>
openclaw models auth paste-token --provider <id>
openclaw models auth order get --provider <id>
openclaw models auth order set --provider <id> <profileIds...>
openclaw models auth order clear --provider <id>
```

`models auth add`, etkileşimli kimlik doğrulama yardımcısıdır. Seçtiğiniz sağlayıcıya bağlı olarak bir sağlayıcı kimlik doğrulama akışı (OAuth/API anahtarı) başlatabilir veya belirteci elle yapıştırmanız için size yol gösterebilir.

`models auth list`, seçilen aracı için kaydedilmiş kimlik doğrulama profillerini belirteç, API anahtarı veya gizli OAuth bilgilerini yazdırmadan listeler. `openai` gibi tek bir sağlayıcıya göre filtrelemek için `--provider <id>`, betiklerde kullanmak içinse `--json` kullanın.

`models auth login`, bir sağlayıcı Plugin'inin kimlik doğrulama akışını (OAuth/API anahtarı) çalıştırır. Hangi sağlayıcıların yüklü olduğunu görmek için `openclaw plugins list` kullanın. `login`; oturum açma sırasında adlandırılmış profilleri destekleyen sağlayıcılar için `--profile-id <id>` (aynı sağlayıcıdaki birden fazla oturumu ayrı tutmak için bunu kullanın), belirli bir kimlik doğrulama yöntemi seçmek için `--method <id>`, `--method device-code` kısayolu olarak `--device-code`, sağlayıcının önerdiği varsayılan modeli uygulamak için `--set-default` ve önce o sağlayıcıya ait mevcut profilleri kaldırmak için `--force` kabul eder (önbelleğe alınmış bir OAuth profili takılı kaldığında veya hesap değiştirmek istediğinizde kullanın).

`models auth login-github-copilot`, `models auth login --provider github-copilot --method device` (GitHub cihaz akışı) için bir kısayoldur; mevcut bir profilin üzerine istem göstermeden yazmak için `--yes` kabul eder.

Kimlik doğrulama sonuçlarını yapılandırılmış belirli bir aracı deposuna yazmak için `openclaw models auth --agent <id> <subcommand>` kullanın. Üst `--agent` bayrağı; `add`, `list`, `login`, `paste-api-key`, `setup-token`, `paste-token`, `login-github-copilot` ve `order get`/`set`/`clear` tarafından dikkate alınır.

OpenAI modelleri için `--provider openai`, varsayılan olarak ChatGPT/Codex hesabıyla oturum açar. Yalnızca genellikle Codex abonelik sınırları için yedek olarak bir OpenAI API anahtarı profili eklemek istediğinizde `--method api-key` kullanın. Eski OpenAI Codex ön ekli kimlik doğrulama/profil durumunu `openai` biçimine taşımak için `openclaw doctor --fix` çalıştırın.

Örnekler:

```bash
openclaw models auth login --provider openai --set-default
openclaw models auth login --provider openai --method api-key
openclaw models auth paste-api-key --provider openai
openclaw models auth list --provider openai
```

Notlar:

- `paste-api-key`, başka bir yerde oluşturulan API anahtarlarını kabul eder, anahtar değerini ister ve `--profile-id` iletmediğiniz sürece varsayılan profil kimliği `<provider>:manual` içine yazar. Otomasyonda anahtarı standart girdiye aktarın; örneğin `printf "%s\n" "$OPENAI_API_KEY" | openclaw models auth paste-api-key --provider openai`.
- `setup-token` ve `paste-token`, belirteçle kimlik doğrulama yöntemleri sunan sağlayıcılar için genel belirteç komutları olarak kalır.
- `setup-token`, etkileşimli bir TTY gerektirir ve sağlayıcının belirteçle kimlik doğrulama yöntemini çalıştırır (sağlayıcı böyle bir yöntem sunuyorsa varsayılan olarak `setup-token` yöntemini kullanır).
- `paste-token`, `--provider` gerektirir, varsayılan olarak belirteç değerini ister ve `--profile-id` iletmediğiniz sürece varsayılan profil kimliği `<provider>:manual` içine yazar. Otomasyonda, sağlayıcı kimlik bilgilerinin kabuk geçmişinde veya işlem listelerinde görünmemesi için belirteci bağımsız değişken olarak iletmek yerine standart girdiye aktarın.
- `paste-token --expires-in <duration>`, `365d` veya `12h` gibi göreli bir süreden mutlak bir belirteç sona erme zamanı kaydeder.
- `openai` için OpenAI API anahtarları ile ChatGPT/OAuth belirteç malzemeleri farklı kimlik doğrulama biçimleridir. `sk-...` OpenAI API anahtarları için `paste-api-key`, yalnızca belirteçle kimlik doğrulama malzemeleri için `paste-token` kullanın.
- Anthropic: `setup-token`/`paste-token`, `anthropic` için desteklenen OpenClaw kimlik doğrulama yollarıdır; ancak OpenClaw, kullanılabilir olduğunda ana makinedeki Claude CLI'ı (`claude -p`) yeniden kullanmayı tercih eder.
- `auth order get/set/clear`, bir sağlayıcı için aracı başına kimlik doğrulama profili sırası geçersiz kılmasını yönetir ve `auth-state.json` içinde saklanır (`auth.order.<provider>` yapılandırma anahtarından ayrıdır). `set`, öncelik sırasına göre bir veya daha fazla profil kimliği alır; `clear`, yapılandırma/döngüsel sıralamaya geri döner.

## İlgili

- [CLI başvurusu](/tr/cli)
- [Model seçimi](/tr/concepts/model-providers)
- [Model yük devretmesi](/tr/concepts/model-failover)
