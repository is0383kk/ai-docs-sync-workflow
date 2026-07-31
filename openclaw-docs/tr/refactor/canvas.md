---
read_when:
    - Canvas barındırıcısının, araçlarının, komutlarının, belgelerinin veya protokol sahipliğinin taşınması
    - Canvas'ın hâlâ çekirdek tarafından sahiplenilip sahiplenilmediğini denetleme
    - Deneysel Canvas plugin PR'ını hazırlama veya inceleme
summary: Canvas'ı çekirdekten çıkarıp paketle birlikte sunulan deneysel bir Plugin'e taşıma planı ve denetim kontrol listesi.
title: Canvas Plugin yeniden düzenlemesi
x-i18n:
    generated_at: "2026-07-26T23:00:20Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: ead3f865ea80acb1e47f45a5ab07acf19a6470035c00c81006b2b1230bedd71e
    source_path: refactor/canvas.md
    workflow: 16
---

# Canvas plugin yeniden düzenlemesi

Canvas az kullanılan ve deneysel bir özelliktir. Onu temel bir özellik olarak değil, paketle birlikte gelen bir plugin olarak ele alın. Temel katman genel Gateway, Node, HTTP, kimlik doğrulama, yapılandırma ve yerel istemci altyapısını koruyabilir; ancak Canvas'a özgü davranış `extensions/canvas` altında bulunmalıdır.

## Amaç

Mevcut eşleştirilmiş Node davranışını koruyarak Canvas sahipliğini `extensions/canvas` konumuna taşıyın:

- ajanın kullanabildiği `canvas` aracı Canvas plugin tarafından kaydedilir
- Canvas Node komutlarına yalnızca Canvas plugin bunları kaydettiğinde izin verilir
- A2UI ana makine/kaynak dosyaları Canvas plugin altında bulunur
- Canvas belge somutlaştırması Canvas plugin altında bulunur
- CLI komut uygulaması Canvas plugin altında bulunur veya plugin'e ait bir çalışma zamanı barrel'ı üzerinden yetki devreder
- belgeler ve plugin envanteri Canvas'ı deneysel ve plugin destekli olarak tanımlar

## Kapsam dışı hedefler

- Bu yeniden düzenleme kapsamında yerel uygulamanın Canvas kullanıcı arayımını yeniden tasarlamayın.
- Ayrı bir ürün kararı Canvas'ın silinmesi gerektiğini belirtmediği sürece iOS, Android veya macOS'tan Canvas protokolü/istemci desteğini kaldırmayın.
- En az bir başka paketle birlikte gelen plugin aynı bağlantı noktasına ihtiyaç duymadığı sürece yalnızca Canvas için geniş kapsamlı bir plugin hizmeti çerçevesi oluşturmayın.

## Mevcut dal durumu

Tamamlananlar:

- `extensions/canvas` içine paketle birlikte gelen plugin paketi eklendi.
- `extensions/canvas/openclaw.plugin.json` eklendi.
- Ajanın `canvas` aracı `src/agents/tools/canvas-tool.ts` konumundan `extensions/canvas/src/tool.ts` konumuna taşındı.
- `createCanvasTool` için temel katmandaki kayıt `src/agents/openclaw-tools.ts` üzerinden kaldırıldı.
- Canvas ana makine uygulaması `src/canvas-host` konumundan `extensions/canvas/src/host` konumuna taşındı.
- `extensions/canvas/runtime-api.ts`; testler, paketleme ve harici genel Canvas yardımcıları için plugin'e ait uyumluluk barrel'ı olarak korundu.
- Canvas belge somutlaştırması `src/gateway/canvas-documents.ts` konumundan `extensions/canvas/src/documents.ts` konumuna taşındı.
- Canvas CLI uygulaması ve A2UI JSONL yardımcıları `extensions/canvas/src/cli.ts` içine taşındı.
- Canvas ana makine URL'si ve kapsamlı yetenek yardımcıları `extensions/canvas/src` içine taşındı.
- Canvas Node komutu varsayılanları sabit kodlanmış temel katman listelerinden çıkarılarak plugin `nodeInvokePolicies` içine taşındı.
- `plugins.entries.canvas.config.host` konumuna plugin'e ait Canvas ana makine yapılandırması eklendi.
- Canvas ve A2UI HTTP sunumu, Canvas plugin HTTP rotası kaydının arkasına taşındı.
- Plugin'e ait HTTP rotaları için genel plugin WebSocket yükseltme yönlendirmesi eklendi.
- Canvas'a özgü Gateway ana makine URL'si ve Node yeteneği kimlik doğrulaması, genel olarak barındırılan plugin yüzeyi ve Node yeteneği yardımcılarıyla değiştirildi.
- Canvas belge URL'lerinin, temel katmanın Canvas belge iç bileşenlerini içe aktarması yerine Canvas plugin üzerinden çözümlenmesi için plugin'e ait barındırılan medya çözümleyicileri eklendi.
- Canvas'ın üst komut yolunu elle yazmadan `openclaw nodes canvas` öğesini plugin'e ait bir Node özelliği olarak bildirebilmesi için `api.registerNodeCliFeature(...)` eklendi.
- `src/**` içindeki üretim `extensions/canvas/runtime-api.js` içe aktarımları kaldırıldı.
- A2UI paket kaynağı `apps/shared/OpenClawKit/Tools/CanvasA2UI` konumundan `extensions/canvas/src/host/a2ui-app` konumuna taşındı.
- A2UI derleme/kopyalama uygulaması `extensions/canvas/scripts` altına taşındı ve kök derleme bağlantıları genel, paketle birlikte gelen plugin varlık kancalarıyla değiştirildi.
- Eski çalışma zamanı üst düzey `canvasHost` yapılandırma diğer adı kaldırıldı.
- `openclaw doctor --fix` komutunun eski `canvasHost` yapılandırmalarını `plugins.entries.canvas.config.host` biçimine yeniden yazması için Canvas doctor geçişi korundu.
- Eski ajan Canvas protokolü uyumluluğu, Gateway protokolü v4 kapsamında kaldırıldı. Yerel istemciler ve Gateway'ler artık yalnızca `pluginSurfaceUrls.canvas` ile `node.pluginSurface.refresh` kullanır; kullanımdan kaldırılan `canvasHostUrl`, `canvasCapability` ve `node.canvas.capability.refresh` yolu bu deneysel yeniden düzenlemede kasıtlı olarak desteklenmez.
- Oluşturulan plugin envanteri Canvas'ı içerecek şekilde güncellendi.
- `docs/plugins/reference/canvas.md` konumuna plugin referans belgeleri eklendi.

Temel katmanın sahipliğinde kaldığı bilinen Canvas yüzeyleri:

- `apps/` altındaki yerel uygulama Canvas işleyicileri hâlâ kasıtlı olarak Canvas plugin yüzeyini kullanıyor
- `apps/` altındaki yerel uygulama Canvas protokolü/istemci işleyicileri
- yayımlanan artefakt çıktısı, geriye dönük uyumlu çalışma zamanı araması için hâlâ `dist/canvas-host/a2ui` kullanıyor; ancak kopyalama adımı artık plugin'e ait

## Hedef yapı

`extensions/canvas` şunların sahibi olmalıdır:

- plugin manifesti ve paket meta verileri
- ajan aracı kaydı
- Node çağırma komutu politikası
- Canvas ana makinesi ve A2UI çalışma zamanı
- Canvas A2UI paket kaynağı ve varlık derleme/kopyalama betikleri
- Canvas belgesi oluşturma ve varlık çözümleme
- Canvas CLI uygulaması
- Canvas belgeleri sayfası ve plugin envanteri girdisi

Temel katman yalnızca genel bağlantı noktalarının sahibi olmalıdır:

- plugin keşfi ve kaydı
- genel ajan aracı kayıt defteri
- genel Node çağırma politikası kayıt defteri
- genel Gateway HTTP/kimlik doğrulaması ve WebSocket yükseltme yönlendirmesi
- genel barındırılan plugin yüzeyi URL çözümlemesi
- genel barındırılan medya çözümleyicisi kaydı
- genel Node yeteneği aktarımı
- genel yapılandırma altyapısı
- genel paketle birlikte gelen plugin varlık kancası keşfi

Yerel uygulamalar, Canvas komut işleyicilerini protokol istemcileri olarak koruyabilir. Bunlar plugin çalışma zamanının sahibi değildir.

## Geçiş adımları

1. `plugins.entries.canvas.config.host` öğesini plugin'e ait yapılandırma yüzeyi olarak ele alın.
2. Belgeleri Canvas'ın deneysel ve paketle birlikte gelen bir plugin olarak tanımlanacağı şekilde güncelleyin.
3. Odaklanmış Canvas testlerini, plugin envanteri kontrollerini, plugin SDK API kontrollerini ve çalışma zamanı sınırlarından etkilenen derleme/tür kapılarını çalıştırın.

## Denetim kontrol listesi

Yeniden düzenlemeyi tamamlanmış olarak değerlendirmeden önce:

- `rg "src/canvas-host|../canvas-host"` hiçbir etkin kaynak içe aktarımı döndürmez.
- `rg "canvas-tool|createCanvasTool" src` temel katmana ait hiçbir Canvas aracı uygulaması bulmaz.
- `rg "canvas.present|canvas.snapshot|canvas.a2ui" src/gateway` genel plugin politikası testleri dışında sabit kodlanmış hiçbir izin listesi varsayılanı bulmaz.
- `rg "extensions/canvas/runtime-api" src --glob '!**/*.test.ts'` boştur.
- `rg "canvas-documents" src` boştur.
- `rg "registerNodesCanvasCommands|nodes-canvas" src` boştur; Canvas plugin, iç içe plugin CLI meta verileri aracılığıyla `openclaw nodes canvas` kaydını yapar.
- `rg "createCanvasHostHandler|handleA2uiHttpRequest" src/gateway` hiçbir Gateway çalışma zamanı sahipliği döndürmez.
- `rg "apps/shared/OpenClawKit/Tools/CanvasA2UI|canvas-a2ui-copy|extensions/canvas/src/host/a2ui" scripts .github package.json` yalnızca uyumluluk sarmalayıcılarını veya plugin'e ait yolları bulur.
- `pnpm plugins:inventory:check` başarılı olur.
- `pnpm plugin-sdk:api:check` başarılı olur veya oluşturulan API sözleşmesi kayıtları kasıtlı olarak güncellenip incelenir.
- Hedeflenen Canvas testleri başarılı olur.
- Canvas ana makinesi/A2UI yolları için değişen işlem hattı testleri başarılı olur.
- PR gövdesi Canvas'ın deneysel ve plugin destekli olduğunu açıkça belirtir.

## Doğrulama komutları

Yineleme sırasında hedefli yerel kontroller kullanın:

```sh
pnpm test extensions/canvas/src/host/server.test.ts extensions/canvas/src/host/server.state-dir.test.ts extensions/canvas/src/host/file-resolver.test.ts
pnpm test src/gateway/server.plugin-node-capability-auth.test.ts src/gateway/server-import-boundary.test.ts
pnpm test extensions/canvas/src/config-migration.test.ts src/commands/doctor-legacy-config.migrations.test.ts
pnpm test test/scripts/changed-lanes.test.ts test/scripts/build-all.test.ts extensions/canvas/scripts/bundle-a2ui.test.ts test/scripts/bundled-plugin-assets.test.ts extensions/canvas/scripts/copy-a2ui.test.ts src/infra/run-node.test.ts
pnpm tsgo:extensions
pnpm plugins:inventory:check
pnpm plugin-sdk:api:check
```

Çalışma zamanı barrel'ı, gecikmeli içe aktarma, paketleme veya yayımlanan plugin yüzeyleri değişirse göndermeden önce `pnpm build` çalıştırın.
