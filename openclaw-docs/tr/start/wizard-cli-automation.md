---
read_when:
    - Betiklerde veya CI'da ilk katılım sürecini otomatikleştiriyorsunuz
    - Belirli sağlayıcılar için etkileşimsiz örneklere ihtiyacınız var
sidebarTitle: CLI automation
summary: OpenClaw CLI için betikleştirilmiş ilk katılım ve aracı kurulumu
title: CLI otomasyonu
x-i18n:
    generated_at: "2026-07-26T23:37:25Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 2a9fd8530379927995641f8033651ff12ada98068f106672e6655a17b8265735
    source_path: start/wizard-cli-automation.md
    workflow: 16
---

Komut dosyasıyla kurulum yapmak için `openclaw onboard --non-interactive` kullanın. Bunun için `--accept-risk` gerekir: etkileşimsiz kurulum, onay istemi olmadan kimlik bilgilerini ve daemon yapılandırmasını yazabilir; bu nedenle bayrak, riskin açıkça kabul edildiğini belirtir.

<Note>
`--json` etkileşimsiz mod anlamına gelmez. Komut dosyaları için `--non-interactive --accept-risk` seçeneğini açıkça iletin.
</Note>

## Temel etkileşimsiz örnek

```bash
openclaw onboard --non-interactive --accept-risk \
  --mode local \
  --auth-choice apiKey \
  --anthropic-api-key "$ANTHROPIC_API_KEY" \
  --secret-input-mode plaintext \
  --gateway-bind loopback \
  --install-daemon \
  --daemon-runtime node \
  --skip-bootstrap \
  --skip-skills
```

Makine tarafından okunabilir bir özet için `--json` ekleyin.

- `--gateway-port` varsayılan olarak `18789` değerini kullanır; yalnızca geçersiz kılmak için iletin.
- `--skip-bootstrap`, kendi çalışma alanını önceden hazırlayan otomasyonlar için varsayılan çalışma alanı dosyalarının oluşturulmasını atlar.
- `--secret-input-mode ref`, düz metin anahtar yerine kimlik doğrulama profilinde ortam değişkeni destekli bir referans (`{ source: "env", provider: "default", id: "<ENV_VAR>" }`) depolar. Etkileşimsiz `ref` modunda sağlayıcının ortam değişkeni, işlem ortamında önceden ayarlanmış olmalıdır: eşleşen ortam değişkeni olmadan satır içi anahtar bayrağı iletmek işlemin hemen başarısız olmasına neden olur.

```bash
openclaw onboard --non-interactive --accept-risk \
  --mode local \
  --auth-choice openai-api-key \
  --secret-input-mode ref
```

## Sağlayıcıya özgü örnekler

<AccordionGroup>
  <Accordion title="Anthropic API anahtarı örneği">
    ```bash
    openclaw onboard --non-interactive --accept-risk \
      --mode local \
      --auth-choice apiKey \
      --anthropic-api-key "$ANTHROPIC_API_KEY" \
      --gateway-bind loopback
    ```
  </Accordion>
  <Accordion title="Cloudflare AI Gateway örneği">
    ```bash
    openclaw onboard --non-interactive --accept-risk \
      --mode local \
      --auth-choice cloudflare-ai-gateway-api-key \
      --cloudflare-ai-gateway-account-id "your-account-id" \
      --cloudflare-ai-gateway-gateway-id "your-gateway-id" \
      --cloudflare-ai-gateway-api-key "$CLOUDFLARE_AI_GATEWAY_API_KEY" \
      --gateway-bind loopback
    ```
  </Accordion>
  <Accordion title="Gemini örneği">
    ```bash
    openclaw onboard --non-interactive --accept-risk \
      --mode local \
      --auth-choice gemini-api-key \
      --gemini-api-key "$GEMINI_API_KEY" \
      --gateway-bind loopback
    ```
  </Accordion>
  <Accordion title="Mistral örneği">
    ```bash
    openclaw onboard --non-interactive --accept-risk \
      --mode local \
      --auth-choice mistral-api-key \
      --mistral-api-key "$MISTRAL_API_KEY" \
      --gateway-bind loopback
    ```
  </Accordion>
  <Accordion title="Moonshot örneği">
    ```bash
    openclaw onboard --non-interactive --accept-risk \
      --mode local \
      --auth-choice moonshot-api-key \
      --moonshot-api-key "$MOONSHOT_API_KEY" \
      --gateway-bind loopback
    ```
  </Accordion>
  <Accordion title="Ollama örneği">
    ```bash
    openclaw onboard --non-interactive --accept-risk \
      --mode local \
      --auth-choice ollama \
      --custom-model-id "qwen3.5:27b" \
      --gateway-bind loopback
    ```
  </Accordion>
  <Accordion title="OpenCode örneği">
    ```bash
    openclaw onboard --non-interactive --accept-risk \
      --mode local \
      --auth-choice opencode-zen \
      --opencode-zen-api-key "$OPENCODE_API_KEY" \
      --gateway-bind loopback
    ```
    Go kataloğu için `--auth-choice opencode-go --opencode-go-api-key "$OPENCODE_API_KEY"` seçeneğine geçin.
  </Accordion>
  <Accordion title="Synthetic örneği">
    ```bash
    openclaw onboard --non-interactive --accept-risk \
      --mode local \
      --auth-choice synthetic-api-key \
      --synthetic-api-key "$SYNTHETIC_API_KEY" \
      --gateway-bind loopback
    ```
  </Accordion>
  <Accordion title="Vercel AI Gateway örneği">
    ```bash
    openclaw onboard --non-interactive --accept-risk \
      --mode local \
      --auth-choice ai-gateway-api-key \
      --ai-gateway-api-key "$AI_GATEWAY_API_KEY" \
      --gateway-bind loopback
    ```
  </Accordion>
  <Accordion title="Z.AI örneği">
    ```bash
    openclaw onboard --non-interactive --accept-risk \
      --mode local \
      --auth-choice zai-api-key \
      --zai-api-key "$ZAI_API_KEY" \
      --gateway-bind loopback
    ```
  </Accordion>
  <Accordion title="Özel sağlayıcı örneği">
    ```bash
    openclaw onboard --non-interactive --accept-risk \
      --mode local \
      --auth-choice custom-api-key \
      --custom-base-url "https://llm.example.com/v1" \
      --custom-model-id "foo-large" \
      --custom-api-key "$CUSTOM_API_KEY" \
      --custom-provider-id "my-custom" \
      --custom-compatibility anthropic \
      --custom-image-input \
      --gateway-bind loopback
    ```

    `--custom-api-key` isteğe bağlıdır; bazı uç noktalar kimlik doğrulaması gerektirmez. Belirtilmezse ilk katılım işlemi ortamda `CUSTOM_API_KEY` değerini kontrol eder. `--custom-provider-id` isteğe bağlıdır ve belirtilmediğinde temel URL'den otomatik olarak türetilir. `--custom-compatibility` varsayılan olarak `openai` değerini kullanır (diğer değerler: `openai-responses`, `anthropic`).

    OpenClaw, bilinen görüntü modeli kimliği kalıplarından (`gpt-4o`, `claude-3/4`, `gemini`, `-vl`/`vision` son ekleri ve benzerleri) görüntü girdisi desteğini çıkarır. Tanınmayan bir görüntü modeli için bunu zorla etkinleştirmek üzere `--custom-image-input`, yalnızca metni zorunlu kılmak üzere `--custom-text-input` ekleyin.

    `apiKey` değerini `{ source: "env", provider: "default", id: "CUSTOM_API_KEY" }` olarak depolayan referans modu varyantı:

    ```bash
    export CUSTOM_API_KEY="your-key"
    openclaw onboard --non-interactive --accept-risk \
      --mode local \
      --auth-choice custom-api-key \
      --custom-base-url "https://llm.example.com/v1" \
      --custom-model-id "foo-large" \
      --secret-input-mode ref \
      --custom-provider-id "my-custom" \
      --custom-compatibility anthropic \
      --custom-image-input \
      --gateway-bind loopback
    ```

  </Accordion>
</AccordionGroup>

Anthropic kurulum belirteciyle kimlik doğrulama desteklenmeye devam eder ancak yerel bir Claude CLI oturumu mevcutsa OpenClaw, Claude CLI'ın yeniden kullanılmasını tercih eder. Üretim ortamında Anthropic API anahtarını tercih edin.

## Başka bir aracı ekleme

`openclaw agents add <name>`, kendi çalışma alanı, oturumları ve kimlik doğrulama profilleri bulunan ayrı bir aracı oluşturur. `--workspace` olmadan (ve başka bayrak belirtilmeden) çalıştırılması etkileşimli sihirbazı başlatır; `--workspace`, `--model`, `--agent-dir`, `--bind` veya `--non-interactive` seçeneklerinden herhangi birinin iletilmesi komutu etkileşimsiz çalıştırır ve ardından `--workspace` gerektirir.

```bash
openclaw agents add work \
  --workspace ~/.openclaw/workspace-work \
  --model openai/gpt-5.6-sol \
  --bind whatsapp:biz \
  --non-interactive \
  --json
```

Yazdığı yapılandırma anahtarları (yeni aracı kimliği için `agents.entries.*` girdisi):

- `name`
- `workspace`
- `agentDir`
- `model` (yalnızca `--model` iletildiğinde)

Notlar:

- Varsayılan çalışma alanı (etkileşimli sihirbazda `--workspace` belirtilmediğinde): `~/.openclaw/workspace-<agentId>`.
- `--bind <channel[:accountId]>` tekrarlanabilir; gelen mesajları yeni aracıya yönlendirmek için bağlamalar ekleyin (sihirbaz bunu etkileşimli olarak da yapabilir).
- Aracı adı, geçerli bir aracı kimliğine dönüştürülür; `main` ayrılmıştır.

## İlgili belgeler

- İlk katılım merkezi: [İlk Katılım (CLI)](/tr/start/wizard)
- Tam başvuru: [CLI Kurulum Başvurusu](/tr/start/wizard-cli-reference)
- Komut başvurusu: [`openclaw onboard`](/tr/cli/onboard)
