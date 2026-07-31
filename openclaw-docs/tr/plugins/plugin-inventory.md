---
read_when:
    - Bir pluginin çekirdek npm paketiyle birlikte sunulup sunulmayacağına veya ayrı olarak kurulup kurulmayacağına karar veriyorsunuz
    - Paketle birlikte gelen plugin paket meta verilerini veya sürüm otomasyonunu güncelliyorsunuz
    - Kanonik dahili ve harici plugin listesine ihtiyacınız var
summary: Çekirdekle birlikte sunulan, harici olarak yayımlanan veya yalnızca kaynakta tutulan OpenClaw pluginlerinin oluşturulmuş envanteri
title: Plugin envanteri
x-i18n:
    generated_at: "2026-07-26T22:54:00Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 2d835087afbe9d75f883c3db9739f914bedab5ac87a9c20b69c248304b61c594
    source_path: plugins/plugin-inventory.md
    workflow: 16
---

# Plugin envanteri

Bu sayfa `extensions/*/package.json`, `openclaw.plugin.json`
ve kök npm paketi `files` dışlamalarından oluşturulur. Şununla yeniden oluşturun:

```bash
pnpm plugins:inventory:gen
```

## Tanımlar

- **Çekirdek npm paketi:** `openclaw` npm paketine yerleşiktir ve ayrı bir plugin kurulumu olmadan kullanılabilir.
- **Resmî harici paket:** Çekirdek npm paketine dahil edilmeyen, bu resmî envanterde tutulan ve gerektiğinde ClawHub ve/veya npm üzerinden kurulan, OpenClaw tarafından bakımı yapılan plugin.
- **Yalnızca kaynak kod deposu:** Yayımlanan npm yapıtlarına dahil edilmeyen ve kurulabilir bir paket olarak tanıtılmayan, depoya yerel plugin.

Kaynak kod depoları npm kurulumlarından farklıdır: `pnpm install` sonrasında paketlenmiş
pluginler `extensions/<id>` konumundan yüklenir; böylece yerel düzenlemeler ve pakete yerel çalışma alanı
bağımlılıkları kullanılabilir.

## Plugin kurma

Kurulum gerekip gerekmediğini belirlemek için her girdideki kurulum yolunu kullanın. `included in OpenClaw`
belirten pluginler çekirdek pakette zaten mevcuttur.
Resmî harici paketlerin bir kez kurulması ve ardından Gateway'in yeniden başlatılması gerekir.

Örneğin Discord, resmî bir harici pakettir:

```bash
openclaw plugins install @openclaw/discord
openclaw gateway restart
openclaw plugins inspect discord --runtime --json
```

Lansman geçişi sırasında sıradan yalın paket belirtimleri npm'den kurulmaya devam eder.
Açık bir kaynak gerektiğinde `clawhub:@openclaw/discord` veya `npm:@openclaw/discord`
kullanın. Kurulumdan sonra kimlik bilgilerini ve kanal yapılandırmasını eklemek için
[Discord](/tr/channels/discord) gibi plugin kurulum belgelerini izleyin. Güncelleme, kaldırma ve yayımlama
komutları için [Pluginleri yönetme](/tr/plugins/manage-plugins) bölümüne bakın.

Her girdi paketi, dağıtım yolunu ve açıklamayı listeler.

## Çekirdek npm paketi

70 plugin

- **[admin-http-rpc](/tr/plugins/reference/admin-http-rpc)** (`@openclaw/admin-http-rpc`) - OpenClaw'a dahildir. OpenClaw yönetici HTTP RPC uç noktası.

- **[alibaba](/tr/plugins/reference/alibaba)** (`@openclaw/alibaba-provider`) - OpenClaw'a dahildir. Video oluşturma sağlayıcısı desteği ekler.

- **[anthropic](/tr/plugins/reference/anthropic)** (`@openclaw/anthropic-provider`) - OpenClaw'a dahildir. Anthropic modelleri, Claude CLI ve yerel Claude oturum kataloğu.

- **[azure-speech](/tr/plugins/reference/azure-speech)** (`@openclaw/azure-speech`) - OpenClaw'a dahildir. Azure AI Speech metinden konuşmaya dönüştürme (MP3, yerel Ogg/Opus sesli notlar, PCM telefon sesi).

- **[bonjour](/tr/plugins/reference/bonjour)** (`@openclaw/bonjour`) - OpenClaw'a dahildir. Yerel OpenClaw gateway'ini Bonjour/mDNS üzerinden duyurur.

- **[browser](/tr/plugins/reference/browser)** (`@openclaw/browser-plugin`) - OpenClaw'a dahildir. Aracı tarafından çağrılabilen araçlar ekler.

- **[byteplus](/tr/plugins/reference/byteplus)** (`@openclaw/byteplus-provider`) - OpenClaw'a dahildir. OpenClaw'a BytePlus ve BytePlus Plan model sağlayıcısı desteği ekler.

- **[canvas](/tr/plugins/reference/canvas)** (`@openclaw/canvas-plugin`) - OpenClaw'a dahildir. Eşleştirilmiş Node'lar için deneysel Canvas denetimi ve A2UI işleme yüzeyleri.

- **[clawrouter](/tr/plugins/reference/clawrouter)** (`@openclaw/clawrouter`) - OpenClaw'a dahildir. OpenClaw'a ClawRouter model sağlayıcısı desteği ekler.

- **[cohere](/tr/plugins/reference/cohere)** (`@openclaw/cohere-provider`) - OpenClaw'a dahildir; npm; ClawHub: `clawhub:@openclaw/cohere-provider`. OpenClaw Cohere sağlayıcı plugini.

- **[comfy](/tr/plugins/reference/comfy)** (`@openclaw/comfy-provider`) - OpenClaw'a dahildir. OpenClaw'a ComfyUI model sağlayıcısı desteği ekler.

- **[copilot-proxy](/tr/plugins/reference/copilot-proxy)** (`@openclaw/copilot-proxy`) - OpenClaw'a dahildir. OpenClaw'a Copilot Proxy model sağlayıcısı desteği ekler.

- **[crabbox](/tr/plugins/reference/crabbox)** (`@openclaw/crabbox-provider`) - OpenClaw'a dahildir. Crabbox CLI tarafından desteklenen bulut çalışanı sağlayıcısı.

- **[cua-computer](/tr/plugins/reference/cua-computer)** (`@openclaw/cua-computer`) - OpenClaw'a dahildir. Windows ve Linux Node ana makineleri için deneysel cua-driver bilgisayar denetimi.

- **[deepgram](/tr/plugins/reference/deepgram)** (`@openclaw/deepgram-provider`) - OpenClaw'a dahildir. Medya anlama sağlayıcısı desteği ekler. Gerçek zamanlı transkripsiyon sağlayıcısı desteği ekler.

- **[document-extract](/tr/plugins/reference/document-extract)** (`@openclaw/document-extract-plugin`) - OpenClaw'a dahildir. Yerel belge eklerinden metin ve yedek sayfa görüntüleri çıkarır.

- **[duckduckgo](/tr/plugins/reference/duckduckgo)** (`@openclaw/duckduckgo-plugin`) - OpenClaw'a dahildir. Web arama sağlayıcısı desteği ekler.

- **[elevenlabs](/tr/plugins/reference/elevenlabs)** (`@openclaw/elevenlabs-speech`) - OpenClaw'a dahildir. Medya anlama sağlayıcısı desteği ekler. Gerçek zamanlı transkripsiyon sağlayıcısı desteği ekler. Metinden konuşmaya dönüştürme sağlayıcısı desteği ekler.

- **[fal](/tr/plugins/reference/fal)** (`@openclaw/fal-provider`) - OpenClaw'a dahildir. OpenClaw'a fal model sağlayıcısı desteği ekler.

- **[file-transfer](/tr/plugins/reference/file-transfer)** (`@openclaw/file-transfer`) - OpenClaw'a dahildir. Özel Node komutları aracılığıyla eşleştirilmiş Node'lardaki dosyaları getirir, listeler ve yazar. 16 MB'a kadar ikili dosyalar için node.invoke üzerinden base64 kullanarak bash stdout kesilmesini atlar.

- **[github-copilot](/tr/plugins/reference/github-copilot)** (`@openclaw/github-copilot-provider`) - OpenClaw'a dahildir. OpenClaw'a GitHub Copilot model sağlayıcısı desteği ekler.

- **[google](/tr/plugins/reference/google)** (`@openclaw/google-plugin`) - OpenClaw'a dahildir. OpenClaw'a Google, Google Gemini CLI ve Google Vertex model sağlayıcısı desteği ekler.

- **[huggingface](/tr/plugins/reference/huggingface)** (`@openclaw/huggingface-provider`) - OpenClaw'a dahildir. OpenClaw'a Hugging Face model sağlayıcısı desteği ekler.

- **[imessage](/tr/plugins/reference/imessage)** (`@openclaw/imessage`) - OpenClaw'a dahildir. OpenClaw mesajlarını gönderip almak için iMessage kanal yüzeyini ekler.

- **[linux-canvas](/tr/plugins/reference/linux-canvas)** (`@openclaw/linux-canvas`) - OpenClaw'a dahildir. OpenClaw Linux masaüstü uygulaması için Canvas işleme köprüsü.

- **[linux-node](/tr/plugins/reference/linux-node)** (`@openclaw/linux-node`) - OpenClaw'a dahildir. Linux Node ana makineleri için masaüstü bildirimleri, kamera yakalama ve konum.

- **[litellm](/tr/plugins/reference/litellm)** (`@openclaw/litellm-provider`) - OpenClaw'a dahildir. OpenClaw'a LiteLLM model sağlayıcısı desteği ekler.

- **[llm-task](/tr/plugins/reference/llm-task)** (`@openclaw/llm-task`) - OpenClaw'a dahildir. İş akışlarından çağrılabilen yapılandırılmış görevler için yalnızca JSON kullanan genel LLM aracı.

- **[lmstudio](/tr/plugins/reference/lmstudio)** (`@openclaw/lmstudio-provider`) - OpenClaw'a dahildir. OpenClaw'a LM Studio model sağlayıcısı desteği ekler.

- **[logbook](/tr/plugins/reference/logbook)** (`@openclaw/logbook`) - OpenClaw'a dahildir. Otomatik çalışma günlüğü: eşleştirilmiş bir Node'dan düzenli ekran görüntüleri yakalar ve bunları gününüzün incelenebilir bir zaman çizelgesine dönüştürür.

- **[memory-core](/tr/plugins/reference/memory-core)** (`@openclaw/memory-core`) - OpenClaw'a dahildir. Aracı tarafından çağrılabilen araçlar ekler.

- **[memory-wiki](/tr/plugins/reference/memory-wiki)** (`@openclaw/memory-wiki`) - OpenClaw'a dahildir. OpenClaw için kalıcı wiki derleyicisi ve Obsidian uyumlu bilgi kasası.

- **[meta](/tr/plugins/reference/meta)** (`@openclaw/meta-provider`) - OpenClaw'a dahildir; npm; ClawHub: `clawhub:@openclaw/meta-provider`. OpenClaw'a Meta model sağlayıcısı desteği ekler.

- **[microsoft](/tr/plugins/reference/microsoft)** (`@openclaw/microsoft-speech`) - OpenClaw'a dahildir. Metinden konuşmaya dönüştürme sağlayıcısı desteği ekler.

- **[microsoft-foundry](/tr/plugins/reference/microsoft-foundry)** (`@openclaw/microsoft-foundry`) - OpenClaw'a dahildir. OpenClaw'a Microsoft Foundry model sağlayıcısı desteği ekler.

- **[migrate-claude](/tr/plugins/reference/migrate-claude)** (`@openclaw/migrate-claude`) - OpenClaw'a dahildir. Claude Code ve Claude Desktop talimatlarını, MCP sunucularını, becerileri ve güvenli yapılandırmayı OpenClaw'a aktarır.

- **[migrate-hermes](/tr/plugins/reference/migrate-hermes)** (`@openclaw/migrate-hermes`) - OpenClaw'a dahildir. Hermes yapılandırmasını, anılarını, becerilerini ve desteklenen kimlik bilgilerini OpenClaw'a aktarır.

- **[minimax](/tr/plugins/reference/minimax)** (`@openclaw/minimax-provider`) - OpenClaw'a dahildir. OpenClaw'a MiniMax ve MiniMax Portal model sağlayıcısı desteği ekler.

- **[mistral](/tr/plugins/reference/mistral)** (`@openclaw/mistral-provider`) - OpenClaw'a dahildir. OpenClaw'a Mistral model sağlayıcısı desteği ekler.

- **[novita](/tr/plugins/reference/novita)** (`@openclaw/novita-provider`) - OpenClaw'a dahildir. OpenClaw'a Novita, Novita AI ve Novitaai model sağlayıcısı desteği ekler.

- **[nvidia](/tr/plugins/reference/nvidia)** (`@openclaw/nvidia-provider`) - OpenClaw'a dahildir. OpenClaw'a NVIDIA model sağlayıcısı desteği ekler.

- **[oc-path](/tr/plugins/reference/oc-path)** (`@openclaw/oc-path`) - OpenClaw'a dahildir. oc:// çalışma alanı dosya adresleme için openclaw path CLI'sini ekler.

- **[ollama](/tr/plugins/reference/ollama)** (`@openclaw/ollama-provider`) - OpenClaw'a dahildir. OpenClaw'a Ollama ve Ollama Cloud model sağlayıcısı desteği ekler.

- **[onepassword](/tr/plugins/reference/onepassword)** (`@openclaw/onepassword`) - OpenClaw'a dahildir. Onay politikası ve SQLite denetim geçmişi bulunan, özenle seçilmiş 1Password gizli bilgi aracısı.

- **[open-prose](/tr/plugins/reference/open-prose)** (`@openclaw/open-prose`) - OpenClaw'a dahildir. /prose eğik çizgi komutuna sahip OpenProse VM beceri paketi.

- **[openai](/tr/plugins/reference/openai)** (`@openclaw/openai-provider`) - OpenClaw'a dahildir. OpenClaw'a OpenAI model sağlayıcısı desteği ekler.

- **[opencode](/tr/plugins/reference/opencode)** (`@openclaw/opencode-provider`) - OpenClaw'a dahildir. OpenClaw'a OpenCode model sağlayıcısı desteği ekler.

- **[opencode-go](/tr/plugins/reference/opencode-go)** (`@openclaw/opencode-go-provider`) - OpenClaw'a dahildir. OpenClaw'a OpenCode Go model sağlayıcısı desteği ekler.

- **[openrouter](/tr/plugins/reference/openrouter)** (`@openclaw/openrouter-provider`) - OpenClaw'a dahildir. OpenClaw'a OpenRouter model sağlayıcısı desteği ekler.

- **[policy](/tr/plugins/reference/policy)** (`@openclaw/policy`) - OpenClaw'a dahildir. Çalışma alanı uyumluluğu için politika destekli doctor denetimleri ekler.

- **[reef](/tr/plugins/reference/reef)** (`@openclaw/reef`) - OpenClaw'a dahildir. Korumalı, uçtan uca şifrelenmiş claw kanalı.

- **[runway](/tr/plugins/reference/runway)** (`@openclaw/runway-provider`) - OpenClaw'a dahildir. Video oluşturma sağlayıcısı desteği ekler.

- **[senseaudio](/tr/plugins/reference/senseaudio)** (`@openclaw/senseaudio-provider`) - OpenClaw'a dahildir. Medya anlama sağlayıcısı desteği ekler.

- **[sglang](/tr/plugins/reference/sglang)** (`@openclaw/sglang-provider`) - OpenClaw'a dahildir. OpenClaw'a SGLang model sağlayıcısı desteği ekler.

- **[synthetic](/tr/plugins/reference/synthetic)** (`@openclaw/synthetic-provider`) - OpenClaw'a dahildir. OpenClaw'a Synthetic model sağlayıcısı desteği ekler.

- **[teams-meetings](/tr/plugins/reference/teams-meetings)** (`@openclaw/teams-meetings`) - OpenClaw'a dahildir. Microsoft Teams toplantılarına Chrome tarayıcı konuğu olarak katılır.

- **[telegram](/tr/plugins/reference/telegram)** (`@openclaw/telegram`) - OpenClaw'a dahildir. OpenClaw mesajlarını gönderip almak için Telegram kanal yüzeyini ekler.

- **[together](/tr/plugins/reference/together)** (`@openclaw/together-provider`) - OpenClaw'a dahildir. OpenClaw'a Together model sağlayıcısı desteği ekler.

- **[tts-local-cli](/tr/plugins/reference/tts-local-cli)** (`@openclaw/tts-local-cli`) - OpenClaw'a dahildir. Metinden konuşmaya dönüştürme sağlayıcısı desteği ekler.

- **[vault](/tr/plugins/reference/vault)** (`@openclaw/vault`) - OpenClaw'a dahildir. HashiCorp Vault SecretRef sağlayıcı entegrasyonu.

- **[vllm](/tr/plugins/reference/vllm)** (`@openclaw/vllm-provider`) - OpenClaw'a dahildir. OpenClaw'a vLLM model sağlayıcısı desteği ekler.

- **[volcengine](/tr/plugins/reference/volcengine)** (`@openclaw/volcengine-provider`) - OpenClaw'a dahildir. OpenClaw'a Volcengine ve Volcengine Plan model sağlayıcısı desteği ekler.

- **[voyage](/tr/plugins/reference/voyage)** (`@openclaw/voyage-provider`) - OpenClaw'a dahildir. Bellek gömme sağlayıcısı desteği ekler.

- **[vydra](/tr/plugins/reference/vydra)** (`@openclaw/vydra-provider`) - OpenClaw'a dahildir. OpenClaw'a Vydra model sağlayıcısı desteği ekler.

- **[web-readability](/tr/plugins/reference/web-readability)** (`@openclaw/web-readability-plugin`) - OpenClaw'a dahildir. Yerel HTML web getirme yanıtlarından okunabilir makale içeriğini ayıklar.

- **[webhooks](/tr/plugins/reference/webhooks)** (`@openclaw/webhooks`) - OpenClaw'a dahildir. Harici otomasyonu OpenClaw TaskFlow'larına bağlayan kimliği doğrulanmış gelen webhook'lar.

- **[workboard](/tr/plugins/reference/workboard)** (`@openclaw/workboard`) - OpenClaw'a dahildir. Aracıların sahip olduğu sorunlar ve oturumlar için pano çalışma tahtası.

- **[xai](/tr/plugins/reference/xai)** (`@openclaw/xai-plugin`) - OpenClaw'a dahildir. OpenClaw'a xAI model sağlayıcısı desteği ekler.

- **[xiaomi](/tr/plugins/reference/xiaomi)** (`@openclaw/xiaomi-provider`) - OpenClaw'a dahildir. OpenClaw'a Xiaomi ve Xiaomi Token Plan model sağlayıcısı desteği ekler.

- **[zoom-meetings](/plugins/reference/zoom-meetings)** (`@openclaw/zoom-meetings`) - OpenClaw'a dahildir. Zoom toplantılarına Chrome tarayıcısı konuğu olarak katılır.

## Resmî harici paketler

72 plugin

- **[acpx](/tr/plugins/reference/acpx)** (`@openclaw/acpx`) - npm; ClawHub. Plugin tarafından yönetilen oturum ve aktarım yönetimine sahip OpenClaw ACP çalışma zamanı arka ucu.

- **[amazon-bedrock](/tr/plugins/reference/amazon-bedrock)** (`@openclaw/amazon-bedrock-provider`) - npm; ClawHub. Model keşfi, gömmeler ve koruma mekanizması desteğine sahip OpenClaw Amazon Bedrock sağlayıcı plugini.

- **[amazon-bedrock-mantle](/tr/plugins/reference/amazon-bedrock-mantle)** (`@openclaw/amazon-bedrock-mantle-provider`) - npm; ClawHub. OpenAI uyumlu model yönlendirmesi için OpenClaw Amazon Bedrock Mantle sağlayıcı plugini.

- **[anthropic-vertex](/tr/plugins/reference/anthropic-vertex)** (`@openclaw/anthropic-vertex-provider`) - npm; ClawHub. Google Vertex AI üzerindeki Claude modelleri için OpenClaw Anthropic Vertex sağlayıcı plugini.

- **[arcee](/tr/plugins/reference/arcee)** (`@openclaw/arcee-provider`) - npm; ClawHub: `clawhub:@openclaw/arcee-provider`. OpenClaw'a Arcee model sağlayıcısı desteği ekler.

- **[baseten](/plugins/reference/baseten)** (`@openclaw/baseten-provider`) - npm; ClawHub: `clawhub:@openclaw/baseten-provider`. OpenClaw Baseten sağlayıcı plugini.

- **[brave](/tr/plugins/reference/brave)** (`@openclaw/brave-plugin`) - npm; ClawHub. Web araması için OpenClaw Brave Search sağlayıcı plugini.

- **[cerebras](/tr/plugins/reference/cerebras)** (`@openclaw/cerebras-provider`) - npm; ClawHub: `clawhub:@openclaw/cerebras-provider`. OpenClaw'a Cerebras model sağlayıcısı desteği ekler.

- **[chutes](/tr/plugins/reference/chutes)** (`@openclaw/chutes-provider`) - npm; ClawHub: `clawhub:@openclaw/chutes-provider`. OpenClaw'a Chutes model sağlayıcısı desteği ekler.

- **[clickclack](/tr/plugins/reference/clickclack)** (`@openclaw/clickclack`) - npm; ClawHub: `clawhub:@openclaw/clickclack`. OpenClaw mesajlarını göndermek ve almak için Clickclack kanal yüzeyini ekler.

- **[cloudflare-ai-gateway](/tr/plugins/reference/cloudflare-ai-gateway)** (`@openclaw/cloudflare-ai-gateway-provider`) - npm; ClawHub: `clawhub:@openclaw/cloudflare-ai-gateway-provider`. OpenClaw'a Cloudflare AI Gateway model sağlayıcısı desteği ekler.

- **[codex](/tr/plugins/reference/codex)** (`@openclaw/codex`) - npm; ClawHub. Codex uygulama sunucusu çalıştırma altyapısı ve yerel oturum kataloğu.

- **[copilot](/tr/plugins/reference/copilot)** (`@openclaw/copilot`) - npm; ClawHub: `clawhub:@openclaw/copilot`. GitHub Copilot aracı çalışma zamanını kaydeder.

- **[deepinfra](/tr/plugins/reference/deepinfra)** (`@openclaw/deepinfra-provider`) - npm; ClawHub: `clawhub:@openclaw/deepinfra-provider`. OpenClaw'a DeepInfra model sağlayıcısı desteği ekler.

- **[deepseek](/tr/plugins/reference/deepseek)** (`@openclaw/deepseek-provider`) - npm; ClawHub: `clawhub:@openclaw/deepseek-provider`. OpenClaw'a DeepSeek model sağlayıcısı desteği ekler.

- **[diagnostics-otel](/tr/plugins/reference/diagnostics-otel)** (`@openclaw/diagnostics-otel`) - npm; ClawHub: `clawhub:@openclaw/diagnostics-otel`. Metrikler, izler ve günlükler için OpenClaw tanılama OpenTelemetry dışa aktarıcısı.

- **[diagnostics-prometheus](/tr/plugins/reference/diagnostics-prometheus)** (`@openclaw/diagnostics-prometheus`) - npm; ClawHub: `clawhub:@openclaw/diagnostics-prometheus`. Çalışma zamanı metrikleri için OpenClaw tanılama Prometheus dışa aktarıcısı.

- **[diffs](/tr/plugins/reference/diffs)** (`@openclaw/diffs`) - npm; ClawHub. Aracılar için OpenClaw salt okunur fark görüntüleyici plugini ve dosya işleyicisi.

- **[diffs-language-pack](/tr/plugins/reference/diffs-language-pack)** (`@openclaw/diffs-language-pack`) - npm; ClawHub: `clawhub:@openclaw/diffs-language-pack`. Varsayılan fark görüntüleyici kümesinin dışındaki diller için sözdizimi vurgulaması ekler.

- **[discord](/tr/plugins/reference/discord)** (`@openclaw/discord`) - npm; ClawHub. Kanallar, doğrudan mesajlar, komutlar ve uygulama olayları için OpenClaw Discord kanal plugini.

- **[exa](/tr/plugins/reference/exa)** (`@openclaw/exa-plugin`) - npm; ClawHub: `clawhub:@openclaw/exa-plugin`. Web araması sağlayıcısı desteği ekler.

- **[featherless](/tr/plugins/reference/featherless)** (`@openclaw/featherless-provider`) - npm; ClawHub: `clawhub:@openclaw/featherless-provider`. OpenClaw Featherless AI sağlayıcı plugini.

- **[feishu](/tr/plugins/reference/feishu)** (`@openclaw/feishu`) - npm; ClawHub. Sohbetler ve iş yeri araçları için OpenClaw Feishu/Lark kanal plugini (topluluk tarafından @m1heng'in bakımıyla sürdürülür).

- **[firecrawl](/tr/plugins/reference/firecrawl)** (`@openclaw/firecrawl-plugin`) - npm; ClawHub: `clawhub:@openclaw/firecrawl-plugin`. Aracıların çağırabileceği araçlar ekler. Web getirme sağlayıcısı desteği ekler. Web araması sağlayıcısı desteği ekler.

- **[fireworks](/tr/plugins/reference/fireworks)** (`@openclaw/fireworks-provider`) - npm; ClawHub: `clawhub:@openclaw/fireworks-provider`. OpenClaw'a Fireworks model sağlayıcısı desteği ekler.

- **[gmi](/tr/plugins/reference/gmi)** (`@openclaw/gmi-provider`) - npm; ClawHub: `clawhub:@openclaw/gmi-provider`. OpenClaw GMI Cloud sağlayıcı plugini.

- **[google-meet](/tr/plugins/reference/google-meet)** (`@openclaw/google-meet`) - npm; ClawHub. Chrome veya Twilio aktarımları üzerinden aramalara katılmak için OpenClaw Google Meet katılımcı plugini.

- **[googlechat](/tr/plugins/reference/googlechat)** (`@openclaw/googlechat`) - npm; ClawHub. Alanlar ve doğrudan mesajlar için OpenClaw Google Chat kanal plugini.

- **[gradium](/tr/plugins/reference/gradium)** (`@openclaw/gradium-speech`) - npm; ClawHub: `clawhub:@openclaw/gradium-speech`. Metinden sese sağlayıcı desteği ekler.

- **[groq](/tr/plugins/reference/groq)** (`@openclaw/groq-provider`) - npm; ClawHub: `clawhub:@openclaw/groq-provider`. OpenClaw'a Groq model sağlayıcısı desteği ekler.

- **[inworld](/tr/plugins/reference/inworld)** (`@openclaw/inworld-speech`) - npm; ClawHub: `clawhub:@openclaw/inworld-speech`. Inworld akışlı metinden sese dönüştürme (MP3, OGG_OPUS, PCM telefon iletişimi).

- **[irc](/tr/plugins/reference/irc)** (`@openclaw/irc`) - npm; ClawHub: `clawhub:@openclaw/irc`. OpenClaw mesajlarını göndermek ve almak için IRC kanal yüzeyini ekler.

- **[kilocode](/tr/plugins/reference/kilocode)** (`@openclaw/kilocode-provider`) - npm; ClawHub: `clawhub:@openclaw/kilocode-provider`. OpenClaw'a Kilocode model sağlayıcısı desteği ekler.

- **[kimi](/tr/plugins/reference/kimi)** (`@openclaw/kimi-provider`) - npm; ClawHub: `clawhub:@openclaw/kimi-provider`. OpenClaw'a Kimi ve Kimi Coding model sağlayıcısı desteği ekler.

- **[line](/tr/plugins/reference/line)** (`@openclaw/line`) - npm; ClawHub. LINE Bot API sohbetleri için OpenClaw LINE kanal plugini.

- **[llama-cpp](/tr/plugins/reference/llama-cpp)** (`@openclaw/llama-cpp-provider`) - npm; ClawHub. node-llama-cpp üzerinden yerel GGUF metin çıkarımı ve gömmeler.

- **[lobster](/tr/plugins/reference/lobster)** (`@openclaw/lobster`) - npm; ClawHub. Türü belirlenmiş işlem hatları ve sürdürülebilir onaylar için Lobster iş akışı aracı plugini.

- **[longcat](/tr/plugins/reference/longcat)** (`@openclaw/longcat-provider`) - npm; ClawHub: `clawhub:@openclaw/longcat-provider`. OpenClaw LongCat sağlayıcı plugini.

- **[matrix](/tr/plugins/reference/matrix)** (`@openclaw/matrix`) - ClawHub: `clawhub:@openclaw/matrix`; npm. Odalar ve doğrudan mesajlar için OpenClaw Matrix kanal plugini.

- **[mattermost](/tr/plugins/reference/mattermost)** (`@openclaw/mattermost`) - npm; ClawHub: `clawhub:@openclaw/mattermost`. OpenClaw mesajlarını göndermek ve almak için Mattermost kanal yüzeyini ekler.

- **[memory-lancedb](/tr/plugins/reference/memory-lancedb)** (`@openclaw/memory-lancedb`) - npm; ClawHub. Otomatik hatırlama, otomatik yakalama ve vektör araması özelliklerine sahip, LanceDB destekli OpenClaw uzun süreli bellek plugini.

- **[moonshot](/tr/plugins/reference/moonshot)** (`@openclaw/moonshot-provider`) - npm; ClawHub: `clawhub:@openclaw/moonshot-provider`. OpenClaw'a Moonshot model sağlayıcısı desteği ekler.

- **[msteams](/tr/plugins/reference/msteams)** (`@openclaw/msteams`) - npm; ClawHub. Bot görüşmeleri için OpenClaw Microsoft Teams kanal plugini.

- **[mxc](/tr/plugins/reference/mxc)** (`@openclaw/mxc-sandbox`) - npm; ClawHub. MXC aracılığıyla işletim sistemi düzeyinde korumalı alan araç yürütmesi: Komutları, yapılandırılmış MXC ilke dosyalarına sahip bir Windows ProcessContainer içinde çalıştırır.

- **[nextcloud-talk](/tr/plugins/reference/nextcloud-talk)** (`@openclaw/nextcloud-talk`) - npm; ClawHub. Görüşmeler için OpenClaw Nextcloud Talk kanal plugini.

- **[nostr](/tr/plugins/reference/nostr)** (`@openclaw/nostr`) - npm; ClawHub. NIP-04 ile şifrelenmiş doğrudan mesajlar için OpenClaw Nostr kanal plugini.

- **[openshell](/tr/plugins/reference/openshell)** (`@openclaw/openshell-sandbox`) - npm; ClawHub. Yansıtılmış yerel çalışma alanları ve SSH komut yürütmesiyle NVIDIA OpenShell CLI için OpenClaw korumalı alan arka ucu.

- **[parallel](/tr/tools/parallel-search)** (`@openclaw/parallel-plugin`) - npm; ClawHub: `clawhub:@openclaw/parallel-plugin`. Web araması sağlayıcısı desteği ekler.

- **[perplexity](/tr/plugins/reference/perplexity)** (`@openclaw/perplexity-plugin`) - npm; ClawHub: `clawhub:@openclaw/perplexity-plugin`. Web araması sağlayıcısı desteği ekler.

- **[pixverse](/tr/plugins/reference/pixverse)** (`@openclaw/pixverse-provider`) - npm; ClawHub: `clawhub:@openclaw/pixverse-provider`. OpenClaw PixVerse video oluşturma sağlayıcı plugini.

- **[qianfan](/tr/plugins/reference/qianfan)** (`@openclaw/qianfan-provider`) - npm; ClawHub: `clawhub:@openclaw/qianfan-provider`. OpenClaw'a Qianfan model sağlayıcısı desteği ekler.

- **[qqbot](/tr/plugins/reference/qqbot)** (`@openclaw/qqbot`) - npm; ClawHub. Grup ve doğrudan mesaj iş akışları için OpenClaw QQ Bot kanal plugini.

- **[qwen](/tr/plugins/reference/qwen)** (`@openclaw/qwen-provider`) - npm; ClawHub: `clawhub:@openclaw/qwen-provider`. OpenClaw'a Qwen, Qwen Cloud, Model Studio, DashScope, Qwen Token Plan ve Bailian Token Plan model sağlayıcısı desteği ekler.

- **[raft](/tr/plugins/reference/raft)** (`@openclaw/raft`) - npm; ClawHub. Güvenli CLI uyandırma köprüleri için OpenClaw Raft kanal plugini.

- **[searxng](/tr/plugins/reference/searxng)** (`@openclaw/searxng-plugin`) - npm; ClawHub: `clawhub:@openclaw/searxng-plugin`. Web araması sağlayıcısı desteği ekler.

- **[signal](/tr/plugins/reference/signal)** (`@openclaw/signal`) - npm; ClawHub: `clawhub:@openclaw/signal`. OpenClaw mesajlarını göndermek ve almak için Signal kanal yüzeyini ekler.

- **[slack](/tr/plugins/reference/slack)** (`@openclaw/slack`) - npm; ClawHub. Kanallar, doğrudan mesajlar, komutlar ve uygulama olayları için OpenClaw Slack kanal plugini.

- **[sms](/tr/plugins/reference/sms)** (`@openclaw/sms`) - npm; ClawHub: `clawhub:@openclaw/sms`. OpenClaw metin mesajları için Twilio SMS kanal plugini.

- **[stepfun](/tr/plugins/reference/stepfun)** (`@openclaw/stepfun-provider`) - npm; ClawHub: `clawhub:@openclaw/stepfun-provider`. OpenClaw'a StepFun ve StepFun Plan model sağlayıcısı desteği ekler.

- **[synology-chat](/tr/plugins/reference/synology-chat)** (`@openclaw/synology-chat`) - npm; ClawHub. OpenClaw kanalları ve doğrudan mesajları için Synology Chat kanal plugini.

- **[tavily](/tr/plugins/reference/tavily)** (`@openclaw/tavily-plugin`) - npm; ClawHub: `clawhub:@openclaw/tavily-plugin`. Aracıların çağırabileceği araçlar ekler. Web arama sağlayıcısı desteği ekler.

- **[tencent](/tr/plugins/reference/tencent)** (`@openclaw/tencent-provider`) - npm; ClawHub: `clawhub:@openclaw/tencent-provider`. OpenClaw'a Tencent TokenHub ve Tencent Tokenplan model sağlayıcısı desteği ekler.

- **[tlon](/tr/plugins/reference/tlon)** (`@openclaw/tlon`) - npm; ClawHub. Sohbet iş akışları için OpenClaw Tlon/Urbit kanal plugini.

- **[tokenjuice](/tr/plugins/reference/tokenjuice)** (`@openclaw/tokenjuice`) - npm; ClawHub: `clawhub:@openclaw/tokenjuice`. Exec ve bash aracı sonuçlarını Tokenjuice indirgeyicileriyle sıkıştırır.

- **[twitch](/tr/plugins/reference/twitch)** (`@openclaw/twitch`) - npm; ClawHub. Sohbet ve moderasyon iş akışları için OpenClaw Twitch kanal plugini.

- **[venice](/tr/plugins/reference/venice)** (`@openclaw/venice-provider`) - npm; ClawHub: `clawhub:@openclaw/venice-provider`. OpenClaw'a Venice model sağlayıcısı desteği ekler.

- **[vercel-ai-gateway](/tr/plugins/reference/vercel-ai-gateway)** (`@openclaw/vercel-ai-gateway-provider`) - npm; ClawHub: `clawhub:@openclaw/vercel-ai-gateway-provider`. OpenClaw'a Vercel AI Gateway model sağlayıcısı desteği ekler.

- **[voice-call](/tr/plugins/reference/voice-call)** (`@openclaw/voice-call`) - npm; ClawHub. Twilio, Telnyx ve Plivo telefon aramaları için OpenClaw sesli arama plugini.

- **[whatsapp](/tr/plugins/reference/whatsapp)** (`@openclaw/whatsapp`) - ClawHub: `clawhub:@openclaw/whatsapp`; npm. WhatsApp Web sohbetleri için OpenClaw WhatsApp kanal plugini.

- **[zai](/tr/plugins/reference/zai)** (`@openclaw/zai-provider`) - npm; ClawHub: `clawhub:@openclaw/zai-provider`. OpenClaw'a Z.AI model sağlayıcısı desteği ekler.

- **[zalo](/tr/plugins/reference/zalo)** (`@openclaw/zalo`) - npm; ClawHub. Bot ve webhook sohbetleri için OpenClaw Zalo kanal plugini.

- **[zalouser](/tr/plugins/reference/zalouser)** (`@openclaw/zalouser`) - npm; ClawHub. Yerel zca-js entegrasyonu üzerinden çalışan OpenClaw Zalo Kişisel Hesap plugini.

## Yalnızca kaynak kod deposundan kullanım

2 plugin

- **[qa-channel](/tr/plugins/reference/qa-channel)** (`@openclaw/qa-channel`) - yalnızca kaynak kod deposundan kullanım. OpenClaw mesajlarını göndermek ve almak için QA Channel yüzeyini ekler.

- **[qa-lab](/tr/plugins/reference/qa-lab)** (`@openclaw/qa-lab`) - yalnızca kaynak kod deposundan kullanım. Özel hata ayıklayıcı kullanıcı arayüzü ve senaryo çalıştırıcısı içeren OpenClaw QA laboratuvarı plugini.
