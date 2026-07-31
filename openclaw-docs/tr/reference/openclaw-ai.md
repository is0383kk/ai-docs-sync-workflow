---
read_when:
    - OpenClaw'ın model aktarımlarını başka bir uygulamada yeniden kullanmak istiyorsunuz
    - packages/ai veya AI aktarım ana makine portlarını değiştiriyorsunuz
    - openclaw sürümünün kök paket dışında npm'de neler yayımladığını inceliyorsunuz
summary: '@openclaw/ai npm paketi: yeniden kullanılabilir model aktarımları, yalıtılmış çalışma zamanları ve ana sistem politika portları'
title: '@openclaw/ai paketi'
x-i18n:
    generated_at: "2026-07-27T00:00:11Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 610057caae0a9bbf9f74074cda75fc40c0b9aa9d3441f8263151f08f1a3f35a8
    source_path: reference/openclaw-ai.md
    workflow: 16
---

`@openclaw/ai`, OpenClaw'ın model yürütme katmanının yayımlanabilir
kitaplık biçimidir: sağlayıcıdan bağımsız ileti/araç/akış sözleşmeleri, doğrulama,
tanılama, olay akışları, yalıtılmış bir çalışma zamanı kayıt defteri ve sekiz
yerleşik API ailesi (Anthropic Messages, OpenAI Completions, OpenAI
Responses, Azure OpenAI Responses, ChatGPT/Codex Responses, Google Generative
AI, Google Vertex, Mistral Conversations) için tembel bağdaştırıcılar.

Her sürümde kök `openclaw` paketiyle birlikte, aynı sürüme sabitlenmiş
olarak yayımlanır ve geçişli bağımlılık ağacının kurulum sırasında kilitlenmesi
için kendi `npm-shrinkwrap.json` dosyasına sahiptir. `openclaw` kurulunca
eşleşen `@openclaw/ai` otomatik olarak kurulur; kitaplık kullanıcıları
herhangi bir OpenClaw uygulama kodu olmadan doğrudan buna bağımlı olabilir.

## Hızlı başlangıç

```js
import { createLlmRuntime } from "@openclaw/ai";
import { registerBuiltInApiProviders } from "@openclaw/ai/providers";

const runtime = createLlmRuntime();
registerBuiltInApiProviders(runtime.registry);

const stream = runtime.streamSimple(model, { messages }, { apiKey });
for await (const event of stream) {
  if (event.type === "text_delta") process.stdout.write(event.delta);
}
const result = await stream.result();
```

Çalıştırılabilir bir sürüm, depodaki `examples/ai-chat` konumunda bulunur.

## Tasarım sözleşmesi

- **Varsayılan olarak örnek kapsamlıdır.** Paketin içe aktarılması,
  küresel olarak hiçbir şeyi kaydetmez. `createApiRegistry()` / `createLlmRuntime()`
  yalıtılmış örnekler döndürür; `registerBuiltInApiProviders(registry)`, bir kayıt defterini yerleşik
  taşımalara dahil eder. Sağlayıcı SDK modülleri ilk kullanımda tembel olarak yüklenir.
- **Ana makine politikası paketlenmez, dışarıdan sağlanır.**
  İstek getirme koruması (örneğin SSRF politikası), araç sonucu yeniden oynatma
  metnindeki gizli bilgilerin maskelenmesi, OpenAI katı araç varsayılanları ve
  tanılama günlük kaydı, `configureAiTransportHost` ile yapılandırılan `AiTransportHost`
  portlarıdır. Kitaplığın varsayılanları etkisizdir; OpenClaw, gerçek
  uygulamalarını akış cephesine kurar.
- **Tek bir olay akışı kimliği.** `@openclaw/ai/event-stream`, OpenClaw
  çekirdeği, agent-core ve harici kullanıcılar tarafından paylaşılan standart
  `EventStream` oluşturucusudur.
- **`internal/*` alt yolları API değildir.** Bunlar
  OpenClaw uygulamasının kendisi için bulunur ve herhangi bir semver garantisi taşımaz.
- Sağlayıcı kimlikleri, kimlik bilgileri, model katalogları,
  yeniden denemeler ve yük devretme uygulamanın sorumluluğunda kalır. OpenClaw
  bunları bu paketin çevresinde katmanlandırır; bir kitaplık kullanıcısı doğrudan
  bir `Model` nesnesi ve seçenekler sağlar.

## Alt yol dışa aktarımları

| Alt yol          | İçerik                                                                         |
| ---------------- | ------------------------------------------------------------------------------ |
| `.`              | Sözleşmeler, `createApiRegistry`, `createLlmRuntime`, `configureAiTransportHost` |
| `./providers`    | `registerBuiltInApiProviders`, `resetApiProviders`                             |
| `./types`        | Model/ileti/araç/akış türleri                                                  |
| `./validation`   | Araç bağımsız değişkeni doğrulaması                                             |
| `./diagnostics`  | Tanılama sözleşmeleri                                                           |
| `./event-stream` | Paylaşılan `EventStream` uygulaması                                            |
| `./internal/*`   | OpenClaw'a özel dahili kullanım, semver garantisi yok                           |
