---
summary: 'OpenClaw''ın yerleşik ajan çalışma zamanını nasıl yapılandırdığı: kod düzeni, sınırlar, kaynak manifestleri ve çalışma zamanı seçimi.'
title: Ajan çalışma zamanı mimarisi
x-i18n:
    generated_at: "2026-07-26T22:37:35Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 3e09ff21b4369a7c102db51e4458ad3ba1e86c9fe43a3a8bff72eef1713d2d51
    source_path: agent-runtime-architecture.md
    workflow: 16
---

OpenClaw, yerleşik ajan çalışma zamanının sahibidir. Çalışma zamanı kodu `src/agents/` altında, model/sağlayıcı aktarımı `src/llm/` altında bulunur ve plugin'lere yönelik sözleşmeler `openclaw/plugin-sdk/*` barrel'ları üzerinden sunulur.

## Çalışma Zamanı Düzeni

| Yol                                 | Sorumluluk                                                                                                                                                                                                                 |
| ----------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `src/agents/embedded-agent-runner/` | Yerleşik deneme döngüsü (`run.ts`, `run/`), model seçimi ve sağlayıcı normalizasyonu (`model*.ts`), sağlayıcı başına istek parametreleri (`extra-params.*`), Compaction, transkript ve oturum bağlantıları. |
| `src/agents/sessions/`              | Oturum kalıcılığı (`session-manager.ts`), kaynak keşfi (`package-manager.ts`, `resource-loader.ts`), oturum içi `extensions` yükleme, istem şablonları, Skills, temalar ve TUI destekli araç işleyicileri (`tools/`). |
| `packages/agent-core/`              | Yeniden kullanılabilir ajan çekirdeği (`@openclaw/agent-core`): ajan döngüsü, çalıştırma düzeneği türleri, iletiler, Compaction yardımcıları, istem şablonları, Skills ve oturum depolama sözleşmeleri. |
| `src/agents/runtime/`               | `@openclaw/agent-core` ile plugin SDK LLM çalışma zamanını bağlayan ve bunu yerel proxy yardımcı programlarıyla birlikte yeniden dışa aktaran OpenClaw cephesi. |
| `src/agents/agent-tools*.ts`        | OpenClaw'a ait araç tanımları, parametre şemaları, araç politikası, araç çağrısı öncesi/sonrası bağdaştırıcıları ve ana makine/sandbox düzenleme araçları. |
| `src/agents/agent-hooks/`           | Yerleşik çalışma zamanı kancaları: Compaction koruması, Compaction talimatları, bağlam budama. |
| `src/agents/harness/`               | Yerleşik ve plugin tarafından kaydedilen çalıştırma düzenekleri için çalıştırma düzeneği kayıt defteri, seçim politikası ve yaşam döngüsü. |
| `src/llm/`                          | Model/sağlayıcı kayıt defteri, aktarım yardımcıları ve sağlayıcıya özgü akış uygulamaları (`src/llm/providers/`). |

## Sınırlar

Çekirdek, yerleşik çalışma zamanını OpenClaw modülleri ve SDK barrel'ları üzerinden çağırır; harici ajan çerçevesi paketleri artık kullanılmaz. Plugin'ler belgelenmiş `openclaw/plugin-sdk/*` giriş noktalarını kullanır ve `src/**` iç bileşenlerini içe aktarmaz.

`@earendil-works/pi-tui` üçüncü taraf bağımlılığı olarak kalır: yerel TUI ve oturum aracı işleyicileri tarafından kullanılan bir terminal bileşeni araç takımıdır. Bunun içselleştirilmesi, ayrı bir kaynak kodu bünyeye alma çalışması gerektirir.

## Manifestler

Kaynak paketleri, `package.json` meta verilerinde OpenClaw kaynaklarını bildirir. Girdiler, paket köküne göreli dosya yolları veya glob kalıplarıdır:

```json
{
  "openclaw": {
    "extensions": ["extensions/index.ts"],
    "skills": ["skills/*.md"],
    "prompts": ["prompts/*.md"],
    "themes": ["themes/*.json"]
  }
}
```

Bir manifestte listelenmeyen kaynak türleri için geleneksel `extensions/`, `skills/`, `prompts/` ve `themes/` dizinlerinin keşfine geri dönülür.

## Çalışma Zamanı Seçimi

- Yerleşik çalışma zamanı kimliği `openclaw` değeridir. Eski diğer ad `pi`, `openclaw` değerine; `codex-app-server` ise `codex` değerine normalleştirilir.
- Plugin çalıştırma düzenekleri ek çalışma zamanı kimlikleri kaydeder (örneğin `codex`).
- Çalışma zamanı politikası, model/sağlayıcı kapsamlı `agentRuntime.id` yapılandırmasıdır (model girdisi sağlayıcı girdisine göre önceliklidir). Ayarlanmamış değer veya `default`, `auto` olarak çözümlenir.
- `auto`, etkin sağlayıcı yolunu destekleyen kayıtlı bir plugin çalıştırma düzeneğini, aksi durumda yerleşik OpenClaw çalışma zamanını seçer. Yalnızca sağlayıcı veya model öneki hiçbir zaman bir çalıştırma düzeneği seçmez.
- OpenAI, yalnızca yazılmış bir istek geçersiz kılması bulunmayan, tam olarak eşleşen resmî bir HTTPS Platform Responses veya ChatGPT Responses yolu için `codex` değerini örtük olarak seçebilir. Completions bağdaştırıcıları, özel uç noktalar ve yazılmış istek davranışına sahip yollar `openclaw` üzerinde kalır; düz metin kullanan resmî HTTP uç noktaları reddedilir. Bkz. [OpenAI örtük ajan çalışma zamanı](/tr/providers/openai#implicit-agent-runtime).

## Model Çalışma Zamanı Nesilleri

Gateway başlangıcı ile yapılandırma, plugin veya kimlik doğrulama yayını, yapılandırılmış her ajan için hazırlanmış bir model çalışma zamanı nesli oluşturur. Her nesil, keşfedilen kimlik doğrulama şablonunu, model kayıt defterini ve yansıtılmış model kataloğunu tek bir atomik anlık görüntü olarak içerir. Ajan çalıştırmaları, değiştirilebilir kimlik doğrulama ve kayıt defteri depolarını bu anlık görüntüden çatallar; göz atma, durum, Cron, doctor, TUI, PDF ve görüntü yolları, dosya sistemi keşfini tekrarlamak yerine yayımlanan kataloğu okur.

Bağımsız gömülü çalışma zamanları, etkinleştirme sınırlarında aynı anlık görüntü biçimini yayımlar. Başarısız veya eski bir nesil, daha yeni bir kısmi nesille birlikte hiçbir zaman sunulmaz; yaşam döngüsü sahibi önce eksiksiz bir yenisini yayımlamalıdır.

## İlgili

- [OpenClaw ajan çalışma zamanı iş akışı](/tr/openclaw-agent-runtime)
- [Ajan çalışma zamanları](/tr/concepts/agent-runtimes)
