---
read_when:
    - CLI'dan exec onaylarını düzenlemek istiyorsunuz
    - Gateway veya Node ana makinelerindeki izin listelerini yönetmeniz gerekir
    - Sohbet arayüzü olmadan bekleyen bir onayı listelemeniz veya sonuçlandırmanız gerekiyor
summary: '`openclaw approvals` ve `openclaw exec-policy` için CLI başvurusu'
title: Onaylar
x-i18n:
    generated_at: "2026-07-26T22:40:16Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: f8b6f198af718d7b058498dbb960a1eb68ced601e1cd9205070b7199688552d2
    source_path: cli/approvals.md
    workflow: 16
---

# `openclaw approvals`

**Yerel ana makine**, **gateway ana makinesi** veya bir **node ana makinesi** için exec onaylarını yönetin. Hedef bayrağı verilmediğinde komutlar diskteki yerel onay dosyasını okur/yazar. Gateway'i hedeflemek için `--gateway`, belirli bir node'u hedeflemek için `--node <id|name|ip>` kullanın.

Takma ad: `openclaw exec-approvals`

İlgili: [Exec onayları](/tr/tools/exec-approvals), [Node'lar](/tr/nodes)

## `openclaw exec-policy`

`openclaw exec-policy`, istenen `tools.exec.*` yapılandırmasıyla yerel ana makine onay dosyasını tek adımda eşitlenmiş durumda tutan, **yalnızca yerelde** kullanılan kolaylık komutudur:

```bash
openclaw exec-policy show
openclaw exec-policy show --json

openclaw exec-policy preset yolo
openclaw exec-policy preset cautious --json

openclaw exec-policy set --host gateway --security full --ask off --ask-fallback full
```

Ön ayarlar (`yolo`, `cautious`, `deny-all`) `host`, `security`, `ask` ve `askFallback` öğelerini birlikte uygular. `set` yalnızca ilettiğiniz bayrakları uygular; kabul edilen her değer doğrulanır (`--host auto|sandbox|gateway|node`, `--security deny|allowlist|full`, `--ask off|on-miss|always`, `--ask-fallback deny|allowlist|full`).

Kapsam:

- Yerel yapılandırma dosyasıyla yerel onay dosyasını birlikte günceller; ilkeyi Gateway'e veya bir node ana makinesine göndermez.
- `--host node` reddedilir: node exec onayları çalışma zamanında node'dan alındığı için yerel `exec-policy` bunları eşitleyemez. Bunun yerine `openclaw approvals set --node <id|name|ip>` kullanın.
- `exec-policy show`, `host=node` kapsamlarını yerel onay dosyasından etkili bir ilke türetmek yerine çalışma zamanında node tarafından yönetilen olarak işaretler.

Uzak ana makine onayları için doğrudan `openclaw approvals set --gateway` veya `openclaw approvals set --node <id|name|ip>` kullanın.

## Yaygın komutlar

```bash
openclaw approvals get
openclaw approvals get --node <id|name|ip>
openclaw approvals get --gateway
openclaw approvals pending
openclaw approvals resolve <id> <allow-once|allow-always|deny>
```

`get`, hedef için etkili exec ilkesini gösterir: istenen `tools.exec` ilkesi, ana makinenin onay dosyası ilkesi ve birleştirilmiş etkili sonuç. Windows yardımcı uygulaması gibi ana makineye özgü bir ilkesi olan node'lar, OpenClaw onay dosyası ilke hesaplamasını uygulamak yerine bu ilkeyi doğrudan gösterir.

Dosya tabanlı node'larda birleştirilmiş görünüm, ana makine tarafından çözümlenmiş bir ilke anlık görüntüsü gerektirir. Eski node'lar, Gateway'in istenen ilkesinin ana makinede de geçerli olduğunu varsaymak yerine etkili ilkeyi kullanılamıyor olarak gösterir.

<Note>
Oturum başına `/exec` geçersiz kılmaları dahil edilmez. Geçerli varsayılanlarını incelemek için ilgili oturumda `/exec` çalıştırın.
</Note>

Öncelik:

- Ana makine onay dosyası, uygulanabilir tek doğruluk kaynağıdır.
- İstenen `tools.exec` ilkesi amacı daraltabilir veya genişletebilir ancak etkili sonuç ana makine kurallarından türetilir.
- `--node`, node ana makinesi onay dosyasını Gateway `tools.exec` ilkesiyle birleştirir (her ikisi de çalışma zamanında uygulanır).
- Gateway yapılandırması kullanılamıyorsa CLI, node onayları anlık görüntüsüne geri döner ve nihai çalışma zamanı ilkesinin hesaplanamadığını belirtir.

## Bekleyen onaylar

Gateway'deki bekleyen exec, plugin ve OpenClaw sistem aracısı onaylarını listeleyin:

```bash
openclaw approvals pending
openclaw approvals pending --json
```

Onay kayıtları aksi durumda istekte bulunanı/inceleyeni filtrelemeye devam ettiği için eksiksiz numaralandırma ve buna karşılık gelen operatör genelindeki `resolve` akışı `operator.admin` kullanır. Çözümleme ayrıca özel `operator.approvals` kapsamını ister. Standart CLI operatör izni her iki kapsamı da içerir; kısıtlı bir üçüncü taraf istemcisi yalnızca bu komutu taklit etmek için yönetici izni istememelidir.

İnsanlarca okunabilir çıktı; onay türünü, aracı/oturum ilişkilendirmesini, isteğin yaşını, sona ermesine kalan süreyi, kısaltılmış bir komutu veya özeti ve kabuktan bağımsız bir `id64_<base64url>` kimlik belirtecini gösterir. Her eksiksiz belirteci ve kayıpsız biçimde kaçış uygulanmış isteği içeren bir `Full request text` bloğu her zaman kompakt tablonun ardından gelir; böylece terminal genişliğine göre kısaltma, bir son eki veya çözümleme için gereken belirteci gizleyemez. Eksiksiz belirteci `resolve` içine kopyalayın. Diğer alanlardaki güvenli olmayan terminal karakterleri, görünür Unicode kaçışları olarak gösterilir. JSON çıktısı, betikler için özgün ham `id`, `summary`, `createdAtMs` ve `expiresAtMs` değerlerini koruyarak `approvals` altında normalleştirilmiş girdiler döndürür; ham kimlikler, ayrılmış `id64_` görüntüleme belirteci ön ekini kullanmadıkları sürece `resolve` tarafından kabul edilmeye devam eder.

Sağlanan bir `id64_` değeri hem değişmez bir ham kimlikle hem de başka bir onayın kodu çözülmüş görüntüleme belirteciyle eşleşirse CLI, yanlış isteği çözümleme riskini almak yerine bunu belirsiz olduğu gerekçesiyle reddeder.

Bir onayı tam kimliğiyle çözümleyin:

```bash
openclaw approvals resolve <id> allow-once
openclaw approvals resolve <id> allow-always
openclaw approvals resolve <id> deny --reason "Bakım sırasında beklenmiyor"
```

CLI, türünü seçmek için birleşik onay kaydını okur, istenen kararı kaydın izin verilen kararlarıyla karşılaştırır ve ardından birleşik çözümleyiciyi çağırır. İlk başarılı karar `0` ile çıkar. Kaydedilen kararı yinelemek de `0` ile çıkar ve `already resolved (same decision)` bildirir. Çelişen bir karar, eksik onay, süresi dolmuş onay veya bu onay türü için kullanılamayan bir karar, anlaşılır bir hata yazdırır ve sıfırdan farklı bir kodla çıkar.

`--reason`, CLI onayına yerel bir not ekler. Geçerli Gateway onay kaydında serbest metin biçiminde bir çözümleme nedeni alanı bulunmadığından bu not kalıcılaştırılmaz veya diğer onay yüzeylerine gönderilmez.

## Onayları bir dosyadan değiştirme

```bash
openclaw approvals set --file ./exec-approvals.json
openclaw approvals set --stdin <<'EOF'
{ version: 1, defaults: { security: "full", ask: "off", askFallback: "full" } }
EOF
openclaw approvals set --node <id|name|ip> --file ./exec-approvals.json
openclaw approvals set --gateway --file ./exec-approvals.json
```

`set` yalnızca katı JSON'u değil, JSON5'i de kabul eder. `--file` veya `--stdin` seçeneklerinden birini kullanın; ikisini birlikte kullanmayın.

Ana makineye özgü Windows node'ları kendi ilke biçimlerini kullanır:

```bash
openclaw approvals set --node <id|name|ip> --stdin <<'EOF'
{
  defaultAction: "deny",
  rules: [{ pattern: "hostname", action: "allow" }]
}
EOF
```

CLI önce node'un geçerli karmasını okur ve güncellemeyle birlikte gönderir; böylece eşzamanlı yerel düzenlemelerin üzerine yazılmak yerine işlem reddedilir. Bu işlem node'un tüm kural listesini değiştirdiği için `rules` gereklidir; `defaultAction` isteğe bağlıdır. Yerel ilkesinin devre dışı olduğunu bildiren bir node uzaktan yapılandırılamaz; önce ilkeyi o ana makinede etkinleştirin veya yapılandırın. Ana makineye özgü ilkeler `allowlist add|remove` yardımcılarını desteklemez.

## "Asla sorma" / YOLO örneği

Exec onaylarında hiçbir zaman durmaması gereken bir ana makine için ana makine onaylarının varsayılanlarını `full` + `off` olarak ayarlayın:

```bash
openclaw approvals set --stdin <<'EOF'
{
  version: 1,
  defaults: {
    security: "full",
    ask: "off",
    askFallback: "full"
  }
}
EOF
```

OpenClaw onay dosyası sunan node'lar için aynı gövdeyi `openclaw approvals set --node <id|name|ip> --stdin` ile kullanın. Ana makineye özgü node'lar, yukarıda gösterilen sahiplerine özgü biçimi gerektirir.

Bu yalnızca **ana makine onay dosyasını** değiştirir. İstenen OpenClaw ilkesini uyumlu tutmak için ayrıca şunları ayarlayın:

```bash
openclaw config set tools.exec.host gateway
openclaw config set tools.exec.mode full
```

`tools.exec.host=gateway` burada açıkça belirtilmiştir çünkü `host=auto` hâlâ "varsa korumalı alan, aksi takdirde gateway" anlamına gelir: YOLO yönlendirmeyle değil, onaylarla ilgilidir. Yapılandırılmış bir korumalı alan olsa bile ana makinede exec çalıştırmak istediğinizde `gateway` (veya `/exec host=gateway`) kullanın.

Atlanan `askFallback` varsayılan olarak `deny` değerini alır. Asla sormama davranışını koruması gereken kullanıcı arayüzsüz bir ana makineyi yükseltirken `askFallback: "full"` değerini açıkça ayarlayın.

Aynı amaç için yalnızca yerel makinede kullanılabilen yerel kısayol:

```bash
openclaw exec-policy preset yolo
```

## İzin listesi yardımcıları

```bash
openclaw approvals allowlist add "~/Projects/**/bin/rg"
openclaw approvals allowlist add --agent main --node <id|name|ip> "/usr/bin/uptime"
openclaw approvals allowlist add --agent "*" "/usr/bin/uname"

openclaw approvals allowlist remove "~/Projects/**/bin/rg"
```

## Yaygın seçenekler

`get`, `set` ve `allowlist add|remove` seçeneklerinin tümü şunları destekler:

- `--node <id|name|ip>` (kimliği, adı, IP'yi veya kimlik ön ekini çözümler; `openclaw nodes` ile aynı çözümleyiciyi kullanır)
- `--gateway`
- paylaşılan node RPC seçenekleri: `--url`, `--token`, `--timeout`, `--json`

Hedef bayrağı verilmemesi, diskteki yerel onay dosyasının kullanılması anlamına gelir.

`allowlist add|remove`, `--agent <id>` seçeneğini de destekler (varsayılan olarak `"*"` değerini alır ve tüm aracılara uygulanır).

Bekleyen istekler canlı Gateway durumu olduğundan `pending` ve `resolve` her zaman Gateway'i kullanır. Paylaşılan Gateway bağlantı seçenekleri `--url`, `--token` ve `--timeout` desteklenir; `pending` ayrıca `--json` seçeneğini de destekler.

## Notlar

- Node ana makinesi `system.execApprovals.get/set` özelliğini duyurmalıdır (macOS uygulaması, başsız node ana makinesi veya Windows yardımcı uygulaması).
- Onay dosyaları OpenClaw durum dizininde ana makine başına depolanır: `$OPENCLAW_STATE_DIR/exec-approvals.json`; değişken ayarlanmamışsa `~/.openclaw/exec-approvals.json`.

## İlgili

- [CLI başvurusu](/tr/cli)
- [Exec onayları](/tr/tools/exec-approvals)
