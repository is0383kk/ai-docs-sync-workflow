---
read_when:
    - Bir `.experimental` yapılandırma anahtarı görüyorsunuz ve kararlı olup olmadığını öğrenmek istiyorsunuz
    - Önizleme çalışma zamanı özelliklerini normal varsayılanlarla karıştırmadan denemek istiyorsunuz
    - Şu anda belgelenmiş deneysel bayrakları bulabileceğiniz tek bir yer istiyorsunuz
summary: OpenClaw'daki deneysel bayrakların anlamı ve şu anda hangilerinin belgelendiği
title: Deneysel özellikler
x-i18n:
    generated_at: "2026-07-26T22:43:44Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 6c14b74bbafce77c0d1e1358ad94053675c4aad9e26be78719f58e78f455c3a2
    source_path: concepts/experimental-features.md
    workflow: 16
---

Deneysel özellikler, açık bayrakların arkasındaki önizleme yüzeyleridir. Kararlı bir varsayılan veya uzun ömürlü bir sözleşme edinmeden önce gerçek dünyada daha fazla kullanıma ihtiyaç duyarlar.

- Bir belge dar kapsamlı bir otomatik kurulum kuralını açıklamadığı sürece varsayılan olarak kapalıdır.
- Biçimi ve davranışı, kararlı yapılandırmaya göre daha hızlı değişebilir.
- Zaten kararlı bir yol varsa onu tercih edin.
- Yaygın biçimde kullanıma sunmadan önce daha küçük bir ortamda test edin.

## Şu anda belgelenen bayraklar

| Yüzey                   | Anahtar                                                                                       | Şu durumda kullanın                                                                                                                        | Daha fazla bilgi                                                                       |
| ----------------------- | --------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------ | -------------------------------------------------------------------------------------- |
| Yerel model çalışma zamanı | `agents.defaults.experimental.localModelLean`, `agents.entries.*.experimental.localModelLean` | Daha küçük veya daha katı bir yerel arka uç, OpenClaw'ın tam varsayılan araç yüzeyini işleyemiyorsa                                          | [Yerel Modeller](/tr/gateway/local-models)                                                |
| Codex yürütme düzeneği  | `plugins.entries.codex.config.appServer.experimental.sandboxExecServer`                       | Yerel Codex app-server 0.132.0 veya daha yeni bir sürümün Code Mode'u devre dışı bırakmak yerine OpenClaw korumalı alan destekli bir exec-server'ı hedeflemesini istiyorsanız | [Codex yürütme düzeneği başvurusu](/tr/plugins/codex-harness-reference#sandboxed-native-execution) |
| Yapılandırılmış planlama aracı | `tools.experimental.planTool`                                                                 | Uyumlu çalışma zamanlarında ve kullanıcı arayüzlerinde çok adımlı iş takibi için yapılandırılmış `update_plan` aracının sunulmasını istiyorsanız | [Gateway yapılandırma başvurusu](/tr/gateway/config-tools#toolsexperimental)               |
| Code Mode               | `tools.codeMode.enabled`                                                                      | Gizli bir OpenClaw araç kataloğuna kod tarafından yönetilen kompakt erişim istiyorsanız                                                    | [Code Mode](/tr/tools/code-mode)                                                          |
| Swarm                   | `tools.swarm.enabled`                                                                         | Code Mode betiklerinin sınırlı alt ajan gruplarını paralel olarak yönetmesini istiyorsanız                                                 | [Swarm](/tools/swarm)                                                                  |

## Control UI Laboratuvarları

Control UI anahtarı bulunan deneyleri yönetmek için **Settings → Agents & Tools → Labs** bölümünü açın. Bir laboratuvarı etkinleştirmek veya devre dışı bırakmak, kurallı Gateway
yapılandırmasına hemen yama uygular; sayfa yalnızca özellik gerektiriyorsa
yeniden başlatma ipucu gösterir.

Code Mode ve Swarm, şu anda kullanıma sunulmuş Labs girdileridir. Her iki anahtar da
mevcut doğrulanmış yapılandırma anahtarlarını yazar ve normalde Gateway'i yeniden
başlatmadan sonraki ajan çalıştırmalarında etkili olur.

## Yerel model yalın modu

`agents.defaults.experimental.localModelLean: true`, her turda ağır isteğe bağlı araçları ajanın doğrudan yüzeyinden kaldırır: `browser`, `cron`, `message`, `image_generate`, `music_generate`, `video_generate`, `tts` ve `pdf`. Açıkça izin verilen veya teslimat için gereken araçlar kullanılabilir durumda kalır; ancak Tool Search bunları doğrudan sunmak yerine kataloğuna ekleyebilir. Yalın mod ayrıca `tools.toolSearch` önceden ayarlanmamışsa plugin/MCP/istemci kataloglarını varsayılan olarak yapılandırılmış Tool Search'e (`tool_search`, `tool_describe`, `tool_call`) ayarlar. Bunu tek bir ajanla sınırlandırmak için `agents.entries.*.experimental.localModelLean` kullanın.

İlk katılım sırasında doğrulanmış bir `ollama` veya `lmstudio` çıkarım rotası, bu değer yoksa `agents.defaults.experimental.localModelLean: true` değerini otomatik olarak ayarlar. OpenClaw, ayarın ilk katılımdan geldiğini kaydeder; böylece daha sonra doğrulanan yerel olmayan bir rota yalnızca otomatik ayarı kaldırır. Açıkça yapılandırılmış bir `true` veya `false` korunur. Kendi kendine barındırılan diğer sağlayıcılar ve OpenAI uyumlu sağlayıcılar, model adlarından veya URL'lerden çıkarılmaz.

Tool Search'ü zaten genel olarak ayarlıyorsanız OpenClaw bu yapılandırmaya dokunmaz. Yalın modun Tool Search varsayılanını devre dışı bırakmak için `tools.toolSearch: false` ayarlayın.

Yapılandırılmış `tools` modunda yalın çalıştırmalar, kodlama için ayarlanmış yerel modellerin alışık oldukları kabuk yolunu seçmeye devam edebilmesi için `exec` aracını Tool Search denetimlerinin yanında doğrudan görünür tutar. Bu yalnızca şema görünürlüğünü değiştirir: normal araç politikası, korumalı alan ve exec onayları uygulanmaya devam eder. Açık `code` ve `directory` modları normal Compaction davranışlarını korur.

### Neden bu araçlar

Bu araçlar en büyük açıklamalara, en geniş parametre biçimlerine veya küçük bir modelin normal kodlama ve konuşma yolundan sapmasına yol açma olasılığına sahiptir. Bağlamı küçük veya daha katı OpenAI uyumlu bir arka uçta bu, şunlar arasındaki farktır:

- Araç şemalarının isteme sığması ile konuşma geçmişine yer bırakmaması.
- Modelin doğru aracı seçmesi ile çok fazla benzer şema nedeniyle hatalı araç çağrıları üretmesi.
- Chat Completions bağdaştırıcısının yapılandırılmış çıktı sınırları içinde kalması ile araç çağrısı yükünün boyutu nedeniyle 400 hatası alınması.

Bunların kaldırılması yalnızca doğrudan araç listesini kısaltır. Model yine `read`, `write`, `edit`, `exec`, `apply_patch`, görüntü anlama, web arama/getirme (yapılandırıldığında), bellek ve oturum/ajan araçlarına sahiptir. `tools.toolSearch: false` ayarlanmadığı sürece ek kataloglara Tool Search üzerinden erişilebilir; açık araç izinleri, yalın bir ajanın daraltılmış bir iş akışına yeniden katılmasını sağlayabilir.

### Ne zaman açılmalı

Modelin Gateway ile iletişim kurabildiğini, ancak tam ajan turlarının hatalı davrandığını doğruladıktan sonra yalın modu etkinleştirin:

1. `openclaw infer model run --gateway --model <ref> --prompt "Reply with exactly: pong"` başarılı olur.
2. Normal bir ajan turu hatalı araç çağrıları, aşırı büyük istemler veya modelin araçlarını yok sayması nedeniyle başarısız olur.
3. `localModelLean: true` değerinin değiştirilmesi hatayı giderir.

### Ne zaman kapalı bırakılmalı

Arka ucunuz tam varsayılan çalışma zamanını sorunsuz işliyorsa bunu kapalı bırakın. Bu, daha küçük bir araç yüzeyine ihtiyaç duyan yerel yığınlar için geçici bir çözümdür; barındırılan modeller veya yeterli kaynağa sahip yerel sistemler için varsayılan değildir.

Yalın mod; `tools.profile`, `tools.allow`/`tools.deny` veya model `compat.supportsTools: false` kaçış yolunun yerini almaz. Belirli bir ajanda kalıcı olarak daha dar bir araç yüzeyi için bu kararlı ayarları tercih edin.

### Etkinleştirme

```json5
{
  agents: {
    defaults: {
      experimental: {
        localModelLean: true,
      },
    },
  },
}
```

Yalnızca bir ajan için:

```json5
{
  agents: {
    list: [
      {
        id: "local",
        model: "lmstudio/gemma-4-e4b-it",
        experimental: {
          localModelLean: true,
        },
      },
    ],
  },
}
```

Bayrağı değiştirdikten sonra Gateway'i yeniden başlatın. Yalın filtreleme; `tools.allow` veya `tools.alsoAllow` ile açıkça korumadığınız sürece `browser`, `cron`, `message`, `image_generate`, `music_generate`, `video_generate`, `tts` ve `pdf` öğelerini kaldırır; Tool Search, korunan araçları doğrudan sunmak yerine yine de kataloğuna ekleyebilir.

## Deneysel, gizli demek değildir

Deneysel bir özellik, kararlı görünen bir varsayılan ayarın arkasına gizlenmek yerine belgelerde ve yapılandırma yolunda açıkça deneysel olarak belirtilmelidir.

## İlgili

- [Özellikler](/tr/concepts/features)
- [Sürüm kanalları](/tr/install/development-channels)
