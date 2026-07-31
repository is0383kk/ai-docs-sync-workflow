---
read_when:
    - Komut izinleri için auto, ask, allowlist, full veya deny seçimi
    - Codex Guardian tarafından incelenen onayları tools.exec.mode aracılığıyla yapılandırma
    - OpenClaw exec onaylarını ACPX harness izinleriyle karşılaştırma
summary: Ana makinede komut yürütme için izin modları, Codex Guardian onayları ve ACPX çalıştırma düzeneği oturumları
title: İzin modları
x-i18n:
    generated_at: "2026-07-27T00:20:40Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: f580e66508c1f69e868ed26a62d88a675f86a4d1ca738650dc5af82e967f3ac3
    source_path: tools/permission-modes.md
    workflow: 16
---

İzin modları, bir aracının ana makine komutlarını çalıştırmadan, dosyalara yazmadan veya ek erişim için bir arka uç altyapısından istemde bulunmadan önce ne kadar yetkiye sahip olduğunu belirler.

<Note>
  İzin modu `tools.exec.host=auto` öğesinden ayrıdır. `tools.exec.host`
  bir komutun nerede çalışacağını seçer. `tools.exec.mode` ise ana makinede yürütmenin nasıl
  onaylanacağını seçer.
</Note>

## Önerilen varsayılan

Her eksik eşleşmeyi insan istemine dönüştürmeden kullanışlı ana makine erişimine ihtiyaç duyan kodlama aracıları için `auto` kullanın:

```bash
openclaw config set tools.exec.mode auto
openclaw approvals get
openclaw gateway restart
```

Ardından geçerli ilkeyi doğrulayın:

```bash
openclaw exec-policy show
```

## OpenClaw ana makine yürütme modları

`tools.exec.mode`, ana makine `exec` için standartlaştırılmış ilke yüzeyidir. Her mod, temel bir `security` (izin listesi katılığı) ve `ask` (eşleşme olmadığında istemde bulunma) çiftine karşılık gelir:

| Mod        | security / ask          | Davranış                                                                                      | Kullanım durumu                                              |
| ----------- | ----------------------- | --------------------------------------------------------------------------------------------- | ----------------------------------------------------- |
| `deny`      | `deny` / `off`          | Ana makinede yürütmeyi tamamen engeller.                                                                     | Hiçbir ana makine komutuna izin verilmediğinde.                         |
| `allowlist` | `allowlist` / `off`     | Yalnızca izin listesindeki komutları çalıştırır; eşleşmeyenleri sessizce reddeder.                                          | Güvenli olduğu bilinen bir komut kümeniz olduğunda.                    |
| `ask`       | `allowlist` / `on-miss` | İzin listesi eşleşmelerini çalıştırır; eşleşme olmadığında bir insana sorar.                                                 | Her yeni komutun bir insan tarafından incelenmesi gerektiğinde.              |
| `auto`      | `allowlist` / `on-miss` | İzin listesi eşleşmelerini çalıştırır; eşleşmeyenleri insan onayına başvurmadan önce otomatik incelemeye gönderir. | Kodlama oturumları pratik ve korumalı erişime ihtiyaç duyduğunda.        |
| `full`      | `full` / `off`          | Ana makinede yürütmeyi istem olmadan çalıştırır.                                                                | Bu güvenilir ana makine/oturumun onay geçitlerini atlaması gerektiğinde. |

`ask` ve `auto` aynı izin listesi/isteme ayarlarını paylaşır; `auto` ayrıca eşleşmeyenlere kendisi karar veren ve yalnızca güvenli biçimde onay veremediğinde yapılandırılmış insan onayı yoluna başvuran yerel otomatik inceleyiciyi etkinleştirir.

Ana makinede yürütme ilkesinin tamamı, yerel onaylar dosyası, izin listesi şeması, güvenli ikili dosyalar ve yönlendirme davranışı için [Yürütme onayları](/tr/tools/exec-approvals) bölümüne bakın.

## Codex Guardian eşlemesi

Yerel Codex uygulama sunucusu oturumlarında `tools.exec.mode: "auto"`, yerel Codex gereksinimleri izin verdiğinde Codex'i Guardian tarafından incelenen onaylara yönlendirir. Genellikle ortaya çıkan değerler:

| Codex alanı         | Tipik değer     |
| ------------------- | ----------------- |
| `approvalPolicy`    | `on-request`      |
| `approvalsReviewer` | `auto_review`     |
| `sandbox`           | `workspace-write` |

`auto` modu bu ilkeyi yapılandırılmış tüm Codex korumalı alan/onay geçersiz kılmalarına zorla uygular; bu nedenle `approvalPolicy: "never"` ile `sandbox: "danger-full-access"` gibi eski ve güvenli olmayan birleşimleri korumaz. `tools.exec.mode: "deny"` ve `"allowlist"`, Codex uygulama sunucusunun yerel yürütmesini tamamen engeller. `tools.exec.mode: "full"` değerini yalnızca onaysız duruşu bilinçli olarak istediğinizde kullanın.

Uygulama sunucusu kurulumu, kimlik doğrulama sırası ve yerel Codex çalışma zamanı ayrıntıları için [Codex altyapısı](/tr/plugins/codex-harness) bölümüne bakın.

## ACPX altyapı izinleri

ACPX oturumları etkileşimsizdir; dolayısıyla bir TTY izin istemine tıklayamazlar. ACPX, `plugins.entries.acpx.config` altında ayrı altyapı düzeyi ayarlar kullanır:

| Ayar                     | Değerler          | Anlamı                                     |
| --------------------------- | --------------- | ------------------------------------------- |
| `permissionMode`            | `approve-reads` | Yalnızca okumaları otomatik olarak onaylar.                    |
| `permissionMode`            | `approve-all`   | Yazmaları ve kabuk komutlarını otomatik olarak onaylar.     |
| `permissionMode`            | `deny-all`      | Tüm izin istemlerini reddeder.                |
| `nonInteractivePermissions` | `fail`          | İstem gerektiğinde işlemi iptal eder.      |
| `nonInteractivePermissions` | `deny`          | İstemi reddeder ve mümkün olduğunda devam eder. |

ACPX izinlerini OpenClaw yürütme onaylarından ayrı olarak ayarlayın:

```bash
openclaw config set plugins.entries.acpx.config.permissionMode approve-all
openclaw config set plugins.entries.acpx.config.nonInteractivePermissions fail
openclaw gateway restart
```

`approve-all` değerini istemsiz bir altyapı oturumunun ACPX acil durum eşdeğeri olarak kullanın. Kurulum ayrıntıları ve hata modları için [ACP aracıları kurulumu](/tr/tools/acp-agents-setup#permission-configuration) bölümüne bakın.

## Mod seçme

| Amaç                                          | Yapılandırma                                                   |
| --------------------------------------------- | ----------------------------------------------------------- |
| Ana makine komutlarını tamamen engellemek                | `tools.exec.mode: "deny"`                                   |
| Yalnızca güvenli olduğu bilinen komutların çalışmasına izin vermek              | `tools.exec.mode: "allowlist"`                              |
| Her yeni komut biçimi için bir insana sormak       | `tools.exec.mode: "ask"`                                    |
| İnsanlardan önce Codex/OpenClaw otomatik incelemesini kullanmak  | `tools.exec.mode: "auto"`                                   |
| Ana makinede yürütme onaylarını tamamen atlamak             | `tools.exec.mode: "full"` ve eşleşen ana makine onayları dosyası |
| Etkileşimsiz ACPX oturumlarının yazmasını/yürütmesini sağlamak | `plugins.entries.acpx.config.permissionMode: "approve-all"` |

Mod değiştirildikten sonra bir komut hâlâ istemde bulunuyor veya başarısız oluyorsa her iki katmanı da inceleyin:

```bash
openclaw approvals get
openclaw exec-policy show
```

Ana makinede yürütme, OpenClaw yapılandırması ile ana makineye özgü yerel onaylar dosyasından daha katı olan sonucu kullanır. ACPX altyapı izinleri ana makinede yürütme onaylarını gevşetmez; ana makinede yürütme onayları da ACPX altyapı istemlerini gevşetmez.

## İlgili

- [Yürütme onayları](/tr/tools/exec-approvals)
- [Yürütme onayları - gelişmiş](/tr/tools/exec-approvals-advanced)
- [Codex altyapısı](/tr/plugins/codex-harness)
- [ACP aracıları kurulumu](/tr/tools/acp-agents-setup#permission-configuration)
