---
read_when:
    - Mac menü kullanıcı arayüzünü veya durum mantığını ayarlama
summary: Menü çubuğu durum mantığı ve kullanıcılara gösterilenler
title: Menü çubuğu
x-i18n:
    generated_at: "2026-07-26T23:25:32Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: d53cd15109864b88010f41ccf4c46ea7fff6721bc6632630d83a558084cb2d62
    source_path: platforms/mac/menu-bar.md
    workflow: 16
---

## Gösterilenler

- Mevcut ajan çalışma durumu, menü çubuğu simgesinde ve menünün ilk durum satırında görüntülenir.
- Çalışma etkinken sistem durumu gizlenir; tüm oturumlar boşta olduğunda yeniden görünür.
- Kök düzeyindeki "Bağlam" öğesi, son oturumları kök menüde genişletmek yerine bunları içeren bir alt menü açar.
- Kök menüdeki "Node'lar" bloğu, istemci/iletişim durumu girdilerini değil, yalnızca eşleştirilmiş **cihazları** (`node.list` kaynağından) listeler.
- Sağlayıcı kullanım anlık görüntüleri mevcut olduğunda, Bağlam'ın altında kök düzeyinde bir "Kullanım" bölümü; varsa bunun ardından maliyet ayrıntıları görünür.
- **Hızlı Sohbet**, kayan ana oturum düzenleyicisini açar; mevcut genel kısayolu öğenin yanında görünür.

## Durum modeli

- Kaynak: `WorkActivityStore` (`apps/macos/Sources/OpenClaw/WorkActivityStore.swift`).
- Olaylar, bir `runId` ile birlikte `ControlAgentEvent` olarak gelir; işleyici (`ControlChannel.routeWorkActivity`), olay yükünden `sessionKey` değerini okur ve bu değer yoksa varsayılan olarak `"main"` kullanır.
- Öncelik: Ana oturum (varsayılan olarak `sessionKey == "main"`) her zaman önceliklidir. Ana oturum etkinse durumu hemen gösterilir. Ana oturum boşta ise bunun yerine ana oturum dışındaki en son etkin oturum gösterilir. Depo, etkinliğin ortasında geçiş yapmaz; yalnızca mevcut oturum boşta olduğunda veya ana oturum etkinleştiğinde geçiş yapar.
- Etkinlik türleri:
  - `job`: üst düzey komut yürütme (`state: started|streaming|done|error|...`).
  - `tool`: `name` içeren `phase: start|result`, isteğe bağlı `meta`/`args`.

## IconState numaralandırması (Swift)

- `idle`
- `workingMain(ActivityKind)`
- `workingOther(ActivityKind)`
- `overridden(ActivityKind)` (hata ayıklama geçersiz kılması)

### ActivityKind -> rozet simgesi

`ActivityKind`, bir `ToolKind` (`bash`, `read`, `write`, `edit`, `attach`, `other`) veya yalın bir `job` sarmalar. Her biri, yaratık simgesinin üzerine çizilen bir SF Symbols rozetine eşlenir (`IconState.badgeSymbolName`):

| Tür             | Simge                              |
| --------------- | ---------------------------------- |
| `bash`          | `chevron.left.slash.chevron.right` |
| `read`          | `doc`                              |
| `write`         | `pencil`                           |
| `edit`          | `pencil.tip`                       |
| `attach`        | `paperclip`                        |
| `other` / `job` | `gearshape.fill`                   |

### Görsel eşleme

- `idle`: normal yaratık, rozet yok.
- `workingMain`: simgeli rozet, tam renk tonu (`.primary` belirginliği), bacaklarda "çalışma" animasyonu.
- `workingOther`: simgeli rozet, soluk renk tonu (`.secondary` belirginliği), hızlı hareket yok.
- `overridden`: gerçek etkinlikten bağımsız olarak seçilen simgeyi/renk tonunu kullanır.

## Bağlam alt menüsü

- Kök menü, oturum sayısını/durumunu içeren tek bir "Bağlam" satırı gösterir; bu satır bir alt menü açar (`MenuSessionsInjector`).
- Alt menü başlığı, son 24 saatteki etkin oturum sayısını gösterir.
- Her oturum satırı; token çubuğunu, yaşını, önizlemesini, düşünme/ayrıntılı çıktı açma-kapama denetimini ve sıfırlama, sıkıştırma ve silme eylemlerini korur.
- Yükleme, bağlantı kesintisi ve oturum yükleme hata iletileri Bağlam alt menüsünde görüntülenir.
- Kullanım ve maliyet bölümleri, alt menüyü açmadan bir bakışta görülebilmeleri için Bağlam'ın altında kök düzeyinde kalır.

## Durum satırı metni (menü)

- Çalışma etkinken: `<Session role> · <activity label>` (`MenuContentView` içindeki `"\(roleLabel) · \(activity.label)"`); burada rol etiketi `Main` veya `Other` olur.
- Boştayken: sistem durumu özetine geri döner.

## Olay alımı

- Kaynak: `ControlChannel.routeWorkActivity(from:)` tarafından yönlendirilen kontrol kanalı `agent` olayları.
- Ayrıştırılan alanlar:
  - Başlatma/durdurma için `data.state` ile `stream: "job"`.
  - `data.phase`, `data.name` ve isteğe bağlı `data.meta`/`data.args` ile `stream: "tool"`.
- Araç etiketleri `ToolDisplayRegistry.resolve(name:args:meta:)` kaynağından gelir; çözümlenemeyen adlar ham araç adına geri döner.

## Hata ayıklama geçersiz kılması

- Settings > Debug > "Icon override" seçicisi:
  - `System (auto)` (varsayılan)
  - `Working: main` / `Working: other` (araç türüne göre: bash, okuma, yazma, düzenleme, diğer)
  - `Idle`
- `UserDefaults` altında `openclaw.iconOverride` anahtarıyla saklanır; `IconState.overridden` değerine eşlenir.

## Test kontrol listesi

- Ana oturum işini tetikleyin: Simge hemen değişir ve durum satırı ana etiketi gösterir.
- Ana oturum boştayken ana oturum dışındaki bir oturum işini tetikleyin: Simge/durum ana oturum dışındaki oturumu gösterir ve işlem tamamlanana kadar sabit kalır.
- Başka bir oturum etkinken ana oturumu başlatın: Simge anında ana oturuma geçer.
- Hızlı araç etkinliği artışları: Rozet titremez (tamamlanan bir araç temizlenmeden önce 2 sn. ek süre, `WorkActivityStore.toolResultGrace`).
- Tüm oturumlar boşta olduğunda sistem durumu satırı yeniden görünür.

## İlgili

- [macOS uygulaması](/tr/platforms/macos)
- [Menü çubuğu simgesi](/tr/platforms/mac/icon)
