---
read_when:
    - Hangi ortam değişkenlerinin hangi sırayla yüklendiğini bilmeniz gerekir
    - Gateway'de eksik API anahtarlarında hata ayıklıyorsunuz
    - Sağlayıcı kimlik doğrulamasını veya dağıtım ortamlarını belgeliyorsunuz
summary: OpenClaw'un ortam değişkenlerini nereden yüklediği ve öncelik sırası
title: Ortam değişkenleri
x-i18n:
    generated_at: "2026-07-26T23:22:37Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: db9990dea5df7731e54c8d442f4704bd4d6e0caf6f2c2fdea32d2583cd41128c
    source_path: help/environment.md
    workflow: 16
---

OpenClaw, ortam değişkenlerini birden çok kaynaktan alır. Kural şudur: **mevcut değerleri asla geçersiz kılma**.
Çalışma alanı `.env` dosyaları daha düşük güvenilirliğe sahip bir kaynaktır: OpenClaw, öncelik sırasını uygulamadan önce çalışma alanı `.env` içindeki sağlayıcı kimlik bilgilerini ve korumalı çalışma zamanı kontrollerini yok sayar.

## Öncelik (en yüksekten en düşüğe)

1. **İşlem ortamı** (Gateway işleminin üst kabuktan/daemon'dan zaten aldığı değerler).
2. **Geçerli çalışma dizinindeki `.env`** (dotenv varsayılanı; geçersiz kılmaz; sağlayıcı kimlik bilgileri ve korumalı çalışma zamanı kontrolleri yok sayılır).
3. `~/.openclaw/.env` konumundaki **genel `.env`** (`$OPENCLAW_STATE_DIR/.env` olarak da bilinir; sağlayıcı API anahtarları için önerilir; geçersiz kılmaz).
4. `~/.openclaw/openclaw.json` içindeki **yapılandırma `env` bloğu** (yalnızca eksikse uygulanır).
5. **İsteğe bağlı oturum açma kabuğu içe aktarımı** (`env.shellEnv.enabled` veya `OPENCLAW_LOAD_SHELL_ENV=1`), yalnızca eksik beklenen anahtarlara uygulanır.

Varsayılan durum dizinini kullanan yeni Ubuntu kurulumlarında OpenClaw, genel `.env` sonrasında `~/.config/openclaw/gateway.env` öğesini de uyumluluk için geri dönüş seçeneği olarak değerlendirir. Her iki dosya da mevcutsa ve birbiriyle uyuşmuyorsa OpenClaw, `~/.openclaw/.env` öğesini korur ve bir uyarı yazdırır.

Yapılandırma dosyası hiç yoksa 4. adım atlanır; etkinleştirilmişse kabuk içe aktarımı yine çalışır.

## Operatörlere yönelik desteklenen değişkenler

Aşağıdaki değişkenler, operatörler için desteklenen ortam sözleşmesidir. Belgelenmemiş `OPENCLAW_*` değişkenleri dahili uygulama ayrıntılarıdır ve bildirimde bulunulmadan kaldırılabilir.

### Yollar ve örnekler

| Değişken                 | Amaç                                                           |
| ------------------------ | ----------------------------------------------------------------- |
| `OPENCLAW_HOME`          | OpenClaw yol varsayılanları için kullanılan ana dizini geçersiz kılar.      |
| `OPENCLAW_STATE_DIR`     | Değiştirilebilir durum dizinini geçersiz kılar.                             |
| `OPENCLAW_CONFIG_PATH`   | Etkin yapılandırma dosyası yolunu geçersiz kılar.                             |
| `OPENCLAW_WORKSPACE_DIR` | Varsayılan aracı çalışma alanını geçersiz kılar.                             |
| `OPENCLAW_PROFILE`       | Adlandırılmış bir profili ve onun yalıtılmış varsayılanlarını seçer.                 |
| `OPENCLAW_GIT_DIR`       | Geliştirme kanalı güncellemeleri tarafından kullanılan kaynak kod deposunu geçersiz kılar. |
| `OPENCLAW_INCLUDE_ROOTS` | `$include` öğesinin ek köklerden çözümlenmesine izin verir.                |

### Gateway ve kimlik doğrulama

| Değişken                    | Amaç                                                         |
| --------------------------- | --------------------------------------------------------------- |
| `OPENCLAW_GATEWAY_URL`      | İstemciler tarafından kullanılan uzak Gateway URL'sini geçersiz kılar.                |
| `OPENCLAW_GATEWAY_PORT`     | Yerel Gateway bağlantı noktasını geçersiz kılar.                                |
| `OPENCLAW_GATEWAY_TOKEN`    | Gateway sunucuları ve istemcileri için belirteçle kimlik doğrulama sağlar.    |
| `OPENCLAW_GATEWAY_PASSWORD` | Gateway sunucuları ve istemcileri için parolayla kimlik doğrulama sağlar. |

### Sağlayıcı kimlik bilgileri

Çekirdek ve paketle gelen sağlayıcı Plugin'leri aşağıdaki kimlik bilgisi ve sağlayıcı seçimi değişkenlerini tanır. İşlem genelinde tek bir değer yerine kapsamlı kimlik bilgilerine ihtiyaç duyduğunuzda her sağlayıcının yapılandırmasını veya SecretRef alanlarını tercih edin.

`AI_GATEWAY_API_KEY`, `ANTHROPIC_ADMIN_API_KEY`, `ANTHROPIC_ADMIN_KEY`, `ANTHROPIC_API_KEY`, `ANTHROPIC_OAUTH_TOKEN`, `ARCEEAI_API_KEY`, `AZURE_OPENAI_API_KEY`, `AZURE_SPEECH_API_KEY`, `AZURE_SPEECH_KEY`, `AZURE_SPEECH_REGION`, `BASETEN_API_KEY`, `BRAVE_API_KEY`, `BYTEPLUS_API_KEY`, `BYTEPLUS_SEED_SPEECH_API_KEY`, `CEREBRAS_API_KEY`, `CHUTES_API_KEY`, `CHUTES_OAUTH_TOKEN`, `CLAWROUTER_API_KEY`, `CLOUDFLARE_AI_GATEWAY_API_KEY`, `CODEX_API_KEY`, `COHERE_API_KEY`, `COMFY_API_KEY`, `COMFY_CLOUD_API_KEY`, `COPILOT_GITHUB_TOKEN`, `DASHSCOPE_API_KEY`, `DEEPGRAM_API_KEY`, `DEEPINFRA_API_KEY`, `DEEPSEEK_API_KEY`, `ELEVENLABS_API_KEY`, `EXA_API_KEY`, `FAL_API_KEY`, `FAL_KEY`, `FEATHERLESS_API_KEY`, `FIRECRAWL_API_KEY`, `FIREWORKS_API_KEY`, `GCLOUD_PROJECT`, `GEMINI_API_KEY`, `GH_TOKEN`, `GITHUB_TOKEN`, `GMI_API_KEY`, `GOOGLE_API_KEY`, `GOOGLE_APPLICATION_CREDENTIALS`, `GOOGLE_CLOUD_API_KEY`, `GOOGLE_CLOUD_LOCATION`, `GOOGLE_CLOUD_PROJECT`, `GRADIUM_API_KEY`, `GROQ_API_KEY`, `HF_TOKEN`, `HUGGINGFACE_HUB_TOKEN`, `INWORLD_API_KEY`, `KILOCODE_API_KEY`, `KIMICODE_API_KEY`, `KIMI_API_KEY`, `LITELLM_API_KEY`, `LM_API_TOKEN`, `LONGCAT_API_KEY`, `MINIMAX_API_KEY`, `MINIMAX_CODE_PLAN_KEY`, `MINIMAX_CODING_API_KEY`, `MINIMAX_OAUTH_TOKEN`, `MISTRAL_API_KEY`, `MODELSTUDIO_API_KEY`, `MODEL_API_KEY`, `MOONSHOT_API_KEY`, `NOVITA_API_KEY`, `NVIDIA_API_KEY`, `OLLAMA_API_KEY`, `OPENAI_ADMIN_KEY`, `OPENAI_API_KEY`, `OPENCODE_API_KEY`, `OPENCODE_ZEN_API_KEY`, `OPENROUTER_API_KEY`, `PARALLEL_API_KEY`, `PERPLEXITY_API_KEY`, `PIXVERSE_API_KEY`, `QIANFAN_API_KEY`, `QWEN_API_KEY`, `QWEN_TOKEN_PLAN_API_KEY`, `RUNWAYML_API_SECRET`, `RUNWAY_API_KEY`, `SENSEAUDIO_API_KEY`, `SGLANG_API_KEY`, `SPEECH_KEY`, `SPEECH_REGION`, `STEPFUN_API_KEY`, `SYNTHETIC_API_KEY`, `TAVILY_API_KEY`, `TOGETHER_API_KEY`, `TOKENHUB_API_KEY`, `TOKENPLAN_API_KEY`, `VENICE_API_KEY`, `VLLM_API_KEY`, `VOLCANO_ENGINE_API_KEY`, `VOLCENGINE_TTS_API_KEY`, `VOLCENGINE_TTS_APPID`, `VOLCENGINE_TTS_TOKEN`, `VOYAGE_API_KEY`, `VYDRA_API_KEY`, `XAI_API_KEY`, `XIAOMI_API_KEY`, `XIAOMI_TOKEN_PLAN_API_KEY`, `XI_API_KEY`, `ZAI_API_KEY` ve `Z_AI_API_KEY`.

Yüklü üçüncü taraf Plugin'ler, Plugin bildirimlerinde ek kimlik bilgisi değişkenleri tanımlayabilir; bu değişkenler, çekirdek OpenClaw değişkenleri değil, bunları bildiren Plugin'in sözleşmeleridir.

### Günlük kaydı ve tanılama

| Değişken                             | Amaç                                                       |
| ------------------------------------ | ------------------------------------------------------------- |
| `OPENCLAW_LOG_LEVEL`                 | Dosya ve konsol günlük düzeylerini geçersiz kılar.                         |
| `OPENCLAW_DEBUG_MODEL_TRANSPORT`     | Model aktarımı zamanlama tanılamasını etkinleştirir.                    |
| `OPENCLAW_DEBUG_MODEL_PAYLOAD`       | Düzenlenmiş model yükü tanılamasını seçer.                    |
| `OPENCLAW_DEBUG_SSE`                 | SSE zamanlamasını veya olay önizleme tanılamasını seçer.                  |
| `OPENCLAW_DEBUG_CODE_MODE`           | Kod modu yüzeyi tanılamasını etkinleştirir.                         |
| `OPENCLAW_DIAGNOSTICS`               | Adlandırılmış tanılama bayraklarını etkinleştirir veya `0` ile tüm bayrakları devre dışı bırakır. |
| `OPENCLAW_DIAGNOSTICS_TIMELINE_PATH` | Zaman çizelgesi tanılaması için JSONL yolunu seçer.               |
| `OPENCLAW_DIAGNOSTICS_EVENT_LOOP`    | Zaman çizelgesi tanılamasına olay döngüsü örnekleri ekler.               |

### Özellik ve çalışma zamanı geçişleri

| Değişken                             | Amaç                                                                      |
| ------------------------------------ | ---------------------------------------------------------------------------- |
| `OPENCLAW_LOAD_SHELL_ENV`            | Eksik beklenen değişkenleri oturum açma kabuğundan içe aktarır.                      |
| `OPENCLAW_SHELL_ENV_TIMEOUT_MS`      | Oturum açma kabuğu içe aktarma zaman aşımını ayarlar.                                          |
| `OPENCLAW_EXEC_SHELL_SNAPSHOT`       | `0` ile exec kabuk anlık görüntülerini devre dışı bırakır.                                       |
| `OPENCLAW_OFFLINE`                   | Sabitlenmiş aracı yardımcı ikili dosyalarının indirilmesini önler.                           |
| `OPENCLAW_BROWSER_HEADLESS`          | Yönetilen tarayıcı başlatmalarını arayüzlü (`0`) veya arayüzsüz (`1`) olmaya zorlar.               |
| `OPENCLAW_DISABLE_BONJOUR`           | Bonjour duyurusunu açık (`0`) veya kapalı (`1`) olmaya zorlar.                             |
| `OPENCLAW_NO_AUTO_UPDATE`            | Güncellemelerin otomatik uygulanmasını devre dışı bırakır.                                            |
| `OPENCLAW_ALLOW_INSECURE_PRIVATE_WS` | Acil durum geçersiz kılması olarak güvenilen özel DNS `ws://` bağlantılarına izin verir.     |
| `OPENCLAW_ALLOW_MULTI_GATEWAY`       | Durum başına sahiplik kilitlerini korurken birden fazla Gateway işleminin çalışmasına izin verir. |
| `OPENCLAW_SKIP_CHANNELS`             | Sorun giderme için Gateway'i kanal aktarımları olmadan başlatır.            |
| `OPENCLAW_THEME`                     | TUI paletini `light` veya `dark` olmaya zorlar.                                  |

## Sağlayıcı kimlik bilgileri ve çalışma alanı `.env`

Sağlayıcı API anahtarlarını yalnızca bir çalışma alanı `.env` içinde tutmayın. OpenClaw, bilinen tüm sağlayıcı kimlik doğrulama ortam değişkenleri (örneğin `GEMINI_API_KEY`, `GOOGLE_API_KEY`, `XAI_API_KEY`, `MISTRAL_API_KEY`, `GROQ_API_KEY`, `DEEPSEEK_API_KEY`, `PERPLEXITY_API_KEY`, `BRAVE_API_KEY`, `TAVILY_API_KEY`, `EXA_API_KEY`, `FIRECRAWL_API_KEY`) dahil olmak üzere çok sayıda sağlayıcı kimlik bilgisi ve uç nokta yönlendirme anahtarını, ayrıca `_API_HOST`, `_BASE_URL`, `_ENDPOINT` veya `_HOMESERVER` ile biten tüm anahtarları ve `OPENCLAW_*`, `CLAWHUB_*`, `ANTHROPIC_API_KEY_*` ve `OPENAI_API_KEY_*` ad alanlarının tamamını çalışma alanı `.env` dosyalarından engeller.

Bunun yerine sağlayıcı kimlik bilgileri için şu güvenilen kaynaklardan birini kullanın:

- Bir kabuk, launchd/systemd birimi, kapsayıcı sırrı veya CI sırrı gibi Gateway işlem ortamı.
- `~/.openclaw/.env` veya `$OPENCLAW_STATE_DIR/.env` konumundaki genel çalışma zamanı dotenv dosyası.
- `~/.openclaw/openclaw.json` içindeki yapılandırma `env` bloğu.
- `env.shellEnv.enabled` veya `OPENCLAW_LOAD_SHELL_ENV=1` etkinleştirildiğinde isteğe bağlı oturum açma kabuğu içe aktarımı.

Daha önce sağlayıcı anahtarlarını veya uç nokta yönlendirme değerlerini yalnızca bir çalışma alanı `.env` içinde sakladıysanız bunları yukarıdaki güvenilen kaynaklardan birine taşıyın. Çalışma alanı `.env`, kimlik bilgisi, uç nokta yönlendirmesi, ana makine geçersiz kılması veya `OPENCLAW_*` çalışma zamanı kontrolü olmayan sıradan proje değişkenlerini sağlamaya devam edebilir.

Güvenlik gerekçesi için [Çalışma alanı `.env` dosyaları](/tr/gateway/security#workspace-env-files) bölümüne bakın.

## Yapılandırma `env` bloğu

Satır içi ortam değişkenlerini ayarlamanın eşdeğer iki yolu (ikisi de mevcut değerleri geçersiz kılmaz):

```json5
{
  env: {
    OPENROUTER_API_KEY: "sk-or-...",
    vars: {
      GROQ_API_KEY: "gsk-...",
    },
  },
}
```

Yapılandırma `env` bloğu yalnızca düz metin dize değerlerini kabul eder. `file:...` değerlerini
genişletmez; örneğin `XAI_API_KEY: "file:secrets/xai-api-key.txt"`
sağlayıcılara tam olarak bu dize biçiminde aktarılır.

Dosya destekli sağlayıcı anahtarları için bunu destekleyen kimlik bilgisi alanında bir SecretRef kullanın:

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

Desteklenen alanlar için [Sır Yönetimi](/tr/gateway/secrets) ve
[SecretRef kimlik bilgisi yüzeyi](/tr/reference/secretref-credential-surface) bölümlerine
bakın.

## Kabuk ortamı içe aktarımı

`env.shellEnv`, oturum açma kabuğunuzu çalıştırır ve yalnızca **eksik** beklenen anahtarları içe aktarır:

```json5
{
  env: {
    shellEnv: {
      enabled: true,
      timeoutMs: 15000,
    },
  },
}
```

Ortam değişkeni eşdeğerleri:

- `OPENCLAW_LOAD_SHELL_ENV=1`
- `OPENCLAW_SHELL_ENV_TIMEOUT_MS=15000` (varsayılan `15000`)

## Exec kabuk anlık görüntüleri

Windows dışındaki Gateway ana makinelerinde bash ve zsh `exec` komutları varsayılan olarak bir başlangıç anlık görüntüsü kullanır.
Bu yolu devre dışı bırakmak için Gateway işlem ortamında `OPENCLAW_EXEC_SHELL_SNAPSHOT=0` ayarlayın.
`false`, `no` ve `off` değerleri de bunu devre dışı bırakır. Çağrı başına `exec.env` değerleri,
anlık görüntüleri açıp kapatamaz veya anlık görüntü önbelleğini yeniden yönlendiremez.

## Çalışma zamanında eklenen ortam değişkenleri

OpenClaw ayrıca oluşturulan alt işlemlere bağlam işaretçileri ekler:

- `OPENCLAW_SHELL=exec`: `exec` aracı üzerinden çalıştırılan komutlar için ayarlanır.
- `OPENCLAW_SHELL=acp-client`: ACP köprü sürecini başlattığında `openclaw acp client` için ayarlanır.
- `OPENCLAW_SHELL=tui-local`: yerel TUI `!` kabuk komutları için ayarlanır.
- `OPENCLAW_CLI=1`: CLI giriş noktası tarafından başlatılan alt süreçler için ayarlanır.

Bunlar çalışma zamanı işaretleyicileridir (gerekli kullanıcı yapılandırması değildir). Bağlama özgü kurallar uygulamak için
kabuk/profil mantığında kullanılabilirler.

## UI ortam değişkenleri

- `OPENCLAW_THEME=light`: terminalinizin arka planı açıksa açık TUI paletini zorlar.
- `OPENCLAW_THEME=dark`: koyu TUI paletini zorlar.
- `COLORFGBG`: terminaliniz bunu dışa aktarıyorsa OpenClaw, TUI paletini otomatik seçmek için arka plan rengi ipucunu kullanır.

## Yapılandırmada ortam değişkeni ikamesi

`${VAR_NAME}` söz dizimini kullanarak yapılandırma dizesi değerlerinde ortam değişkenlerine doğrudan başvurabilirsiniz:

```json5
{
  models: {
    providers: {
      "vercel-gateway": {
        apiKey: "${VERCEL_GATEWAY_API_KEY}",
      },
    },
  },
}
```

Tüm ayrıntılar için [Yapılandırma: Ortam değişkeni ikamesi](/tr/gateway/configuration-reference#env-var-substitution) bölümüne bakın.

## Secret ref'leri ile `${ENV}` dizeleri

OpenClaw, ortam değişkenlerine dayalı iki kalıbı destekler:

- Yapılandırma değerlerinde `${VAR}` dize ikamesi.
- Gizli bilgi referanslarını destekleyen alanlar için SecretRef nesneleri (`{ source: "env", provider: "default", id: "VAR" }`).

Her ikisi de etkinleştirme sırasında süreç ortamından çözümlenir. SecretRef ayrıntıları [Gizli Bilgi Yönetimi](/tr/gateway/secrets) bölümünde belgelenmiştir.
Yapılandırmanın `env` bloğunun kendisi SecretRef'leri veya `file:...`
kısa gösterim değerlerini çözümlemez.

## Yolla ilgili ortam değişkenleri

| Değişken                 | Amaç                                                                                                                                                                                                                                 |
| ------------------------ | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `OPENCLAW_HOME`          | Dahili OpenClaw yol varsayılanları (`~/.openclaw/`, aracı dizinleri, oturumlar, kimlik bilgileri, yükleyici ilk kurulumu ve varsayılan geliştirme çalışma kopyası) için kullanılan ana dizini geçersiz kılar. OpenClaw özel bir hizmet kullanıcısı olarak çalıştırılırken kullanışlıdır. |
| `OPENCLAW_STATE_DIR`     | Durum dizinini geçersiz kılar (varsayılan: `~/.openclaw`).                                                                                                                                                                                   |
| `OPENCLAW_CONFIG_PATH`   | Yapılandırma dosyası yolunu geçersiz kılar (varsayılan: `~/.openclaw/openclaw.json`).                                                                                                                                                                    |
| `OPENCLAW_INCLUDE_ROOTS` | `$include` yönergelerinin yapılandırma dizini dışındaki dosyaları çözümleyebileceği dizinlerin yol listesi (varsayılan: yoktur; `$include` yapılandırma diziniyle sınırlıdır). Tilde genişletilir.                                                         |

## Aracı yardımcı araç indirmeleri

OpenClaw'ın sabitlenmiş `fd` ve `ripgrep` yardımcı ikili dosyalarını indirmesini
engellemek için `OPENCLAW_OFFLINE=1` ayarlayın. OpenClaw araçları dizinindeki mevcut yardımcılar
ve çalışan sistem ikili dosyaları kullanılmaya devam edebilir; eksik bir yardımcı, ağ isteğini
tetiklemek yerine kullanılamaz durumda kalır.

## Günlük Kaydı

| Değişken                         | Amaç                                                                                                                                                                                      |
| -------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `OPENCLAW_LOG_LEVEL`             | Hem dosya hem de konsol için günlük düzeyini geçersiz kılar (ör. `debug`, `trace`). Yapılandırmadaki `logging.level` ve `logging.consoleLevel` değerlerinden önceliklidir. Geçersiz değerler bir uyarıyla yok sayılır. |
| `OPENCLAW_DEBUG_MODEL_TRANSPORT` | Genel hata ayıklama günlüklerini etkinleştirmeden `info` düzeyinde hedefli model istek/yanıt zamanlama tanılamaları yayınlar.                                                                                  |
| `OPENCLAW_DEBUG_MODEL_PAYLOAD`   | Model yükü tanılamaları: `summary`, `tools` veya `full-redacted`. `full-redacted` sınırlandırılır ve hassas verilerden arındırılır ancak istem/ileti metni içerebilir.                                               |
| `OPENCLAW_DEBUG_SSE`             | Akış tanılamaları: ilk/tamamlandı zamanlaması için `events`, hassas verilerden arındırılmış ilk beş SSE olayını dahil etmek için `peek`.                                                                                 |
| `OPENCLAW_DEBUG_CODE_MODE`       | Sağlayıcı araçlarının gizlenmesi ve kompakt denetim/doğrudan zorlaması dahil olmak üzere kod modu model yüzeyi tanılamaları.                                                                                  |

### `OPENCLAW_HOME`

Ayarlandığında `OPENCLAW_HOME`, dahili OpenClaw yol varsayılanları için sistem ana dizininin (`$HOME` / `os.homedir()`) yerini alır. Buna varsayılan durum dizini, yapılandırma yolu, aracı dizinleri, kimlik bilgileri, yükleyici ilk kurulum çalışma alanı ve `openclaw update --channel dev` tarafından kullanılan varsayılan geliştirme çalışma kopyası dahildir.

**Öncelik:** `OPENCLAW_HOME` > `$HOME` > `USERPROFILE` > Android'de Termux `PREFIX` ana dizin geri dönüşü > `os.homedir()`

**Örnek** (macOS LaunchDaemon):

```xml
<key>EnvironmentVariables</key>
<dict>
  <key>OPENCLAW_HOME</key>
  <string>/Users/user</string>
</dict>
```

`OPENCLAW_HOME`, tilde içeren bir yol olarak da ayarlanabilir (ör. `~/svc`); bu yol kullanılmadan önce aynı işletim sistemi ana dizin geri dönüş zinciri kullanılarak genişletilir.

`OPENCLAW_STATE_DIR`, `OPENCLAW_CONFIG_PATH` ve `OPENCLAW_GIT_DIR` gibi açık yol değişkenleri yine önceliklidir. Kabuk başlangıç dosyası algılama, paket yöneticisi kurulumu ve ana makine `~` genişletmesi gibi işletim sistemi hesabı görevleri gerçek sistem ana dizinini kullanmaya devam edebilir.

## nvm kullanıcıları: web_fetch TLS hataları

Node.js, sistem paket yöneticisi yerine **nvm** aracılığıyla kurulduysa yerleşik `fetch()`,
modern kök CA'ları (Let's Encrypt için ISRG Root X1/X2, DigiCert Global Root G2 vb.) eksik olabilen
nvm'nin paketlenmiş CA deposunu kullanır. Bu, çoğu HTTPS sitesinde `web_fetch` işleminin `"fetch failed"` hatasıyla başarısız olmasına neden olur.

Linux'ta OpenClaw, nvm'yi otomatik olarak algılar ve düzeltmeyi gerçek başlangıç ortamında uygular:

- `openclaw gateway install`, `NODE_EXTRA_CA_CERTS` değerini systemd hizmet ortamına yazar
- `openclaw` CLI giriş noktası, Node başlamadan önce `NODE_EXTRA_CA_CERTS` ayarlanmış olarak kendisini yeniden çalıştırır

**Elle düzeltme (eski sürümler veya doğrudan `node ...` başlatmaları için):**

OpenClaw'ı başlatmadan önce değişkeni dışa aktarın:

```bash
export NODE_EXTRA_CA_CERTS=/etc/ssl/certs/ca-certificates.crt
openclaw gateway run
```

Bu değişken için yalnızca `~/.openclaw/.env` dosyasına yazmaya güvenmeyin; Node,
`NODE_EXTRA_CA_CERTS` değerini süreç başlatılırken okur.

## Eski ortam değişkenleri

OpenClaw yalnızca `OPENCLAW_*` ortam değişkenlerini okur. Önceki sürümlerdeki eski
`CLAWDBOT_*` ve `MOLTBOT_*` önekleri sessizce
yok sayılır.

Başlangıçta Gateway sürecinde hâlâ bunlardan biri ayarlıysa OpenClaw, algılanan önekleri
ve toplam sayıyı listeleyen tek bir Node kullanımdan kaldırma uyarısı (`OPENCLAW_LEGACY_ENV_VARS`)
yayınlar. Eski öneki `OPENCLAW_` ile değiştirerek her değeri yeniden adlandırın
(örneğin `CLAWDBOT_GATEWAY_TOKEN` yerine `OPENCLAW_GATEWAY_TOKEN`); eski adların hiçbir etkisi yoktur.

## İlgili Konular

- [Gateway yapılandırması](/tr/gateway/configuration)
- [SSS: ortam değişkenleri ve .env yükleme](/tr/help/faq#env-vars-and-env-loading)
- [Modellere genel bakış](/tr/concepts/models)
