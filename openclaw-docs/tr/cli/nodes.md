---
read_when:
    - Eşleştirilmiş Node'ları (kameralar, ekran, tuval) yönetiyorsunuz
    - İstekleri onaylamanız veya Node komutlarını çağırmanız gerekir
summary: '`openclaw nodes` için CLI referansı (durum, eşleştirme, çağırma, kamera/tuval/ekran/konum/bildirim)'
title: Node'lar
x-i18n:
    generated_at: "2026-07-26T22:41:53Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 53003bcd3d30b0e754aa0717452700595c0cf69d9ecd6301b8a1bf320ea1838a
    source_path: cli/nodes.md
    workflow: 16
---

# `openclaw nodes`

Eşleştirilmiş Node'ları (cihazları) yönetin ve Node yeteneklerini çağırın.

İlgili: [Node'lara genel bakış](/tr/nodes) - [Etkin bilgisayar varlığı](/tr/nodes/presence) - [Kamera Node'ları](/tr/nodes/camera) - [Görüntü Node'ları](/tr/nodes/images)

Her alt komuttaki ortak seçenekler: `--url <url>`, `--token <token>`, `--timeout <ms>` (varsayılan `10000`), `--json`.

## Durum

```bash
openclaw nodes status
openclaw nodes status --connected
openclaw nodes status --last-connected 24h
openclaw nodes list
openclaw nodes describe --node <idOrNameOrIp>
```

`status` ve `list`, hem `--connected` (yalnızca bağlı Node'lar) hem de `--last-connected <duration>` (ör. `24h`, `7d`; yalnızca belirtilen süre içinde bağlanmış Node'lar) kabul eder. `list`, bekleyen ve eşleştirilmiş Node'ları ayrı tablolarda gösterir; eşleştirilmiş satırlar en son bağlantıdan bu yana geçen süreyi (Last Connect) içerir. `status`, Node başına yetenek, sürüm ve son girdi ayrıntılarını tek bir birleştirilmiş tabloda gösterir. Bağlı bir macOS Node'u, son girdiyi yalnızca kullanıcı **Active computer detection** seçeneğini etkinleştirip Accessibility izni verdikten sonra bildirir; en güncel satır `active` ile işaretlenir. Bkz. [Etkin bilgisayar varlığı](/tr/nodes/presence). `describe`, bir Node'un yeteneklerini, izinlerini, etkinliğini ve yürürlükteki/bekleyen çağırma komutlarını yazdırır.

## Eşleştirme

```bash
openclaw nodes pending
openclaw nodes approve <requestId>
openclaw nodes reject <requestId>
openclaw nodes remove --node <id|name|ip>
openclaw nodes rename --node <id|name|ip> --name <displayName>
```

Bu komutlar, Node'un WS `connect` el sıkışmasını denetleyen cihaz eşleştirmesinden (`openclaw devices approve`) ayrı olan, Gateway tarafından yönetilen `node.pair.*` deposunu denetler. İkisinin nasıl ilişkili olduğu hakkında bilgi için [Node'lar](/tr/nodes) bölümüne bakın.

- `remove`, Node'un eşleştirilmiş rol kaydını iptal eder. Cihaz destekli bir Node için bu işlem, cihaz eşleştirme deposundaki `node` rolünü iptal eder ve Node rolü oturumlarının bağlantısını keser: karma rollü bir cihaz satırını korur ve yalnızca `node` rolünü kaybeder; yalnızca Node rolüne sahip bir cihaz satırı silinir. Ayrıca eşleşen tüm eski Gateway tarafından yönetilen Node eşleştirme kayıtlarını temizler.
- `pending` yalnızca `operator.pairing` kapsamını gerektirir.
- `gateway.nodes.pairing.autoApproveCidrs`, açıkça güvenilen ilk `role: node` cihaz eşleştirmesinde bekleme adımını atlayabilir. Varsayılan olarak kapalıdır; rol yükseltmelerini onaylamaz.
- `gateway.nodes.pairing.sshVerify` (varsayılan olarak açık), Gateway cihaz anahtarını SSH üzerinden Node ana makinesinde doğrulayabildiğinde ilk `role: node` cihaz eşleştirmesini otomatik olarak onaylar; ilk yetenek yüzeyi de aynı adımda onaylanır. Bkz. [Node eşleştirme](/tr/gateway/pairing#ssh-verified-device-auto-approval-default).
- `approve` kapsam gereksinimleri, bekleyen isteğin bildirdiği komutlara göre belirlenir:
  - komutsuz istek: `operator.pairing`
  - sıradan Node komutları: `operator.pairing` + `operator.write`
  - yönetici açısından hassas komutlar (`system.run`, `system.run.prepare`, `system.which`, `browser.proxy`, `fs.listDir` ve `system.execApprovals.get/set`): `operator.pairing` + `operator.admin`
- `remove` kapsamı: `operator.pairing`, operatör olmayan Node satırlarını kaldırabilir; karma rollü bir cihazdaki kendi Node rolünü iptal eden cihaz belirteci çağırıcısı ek olarak `operator.admin` gerektirir.

## Çağırma

```bash
openclaw nodes invoke --node <id> --command system.which --params '{"bins":["uname"]}'
```

Bayraklar:

- `--command <command>` (gerekli): ör. `canvas.eval`.
- `--params <json>`: JSON nesnesi dizesi (varsayılan `{}`).
- `--invoke-timeout <ms>`: Node çağırma zaman aşımı (varsayılan `15000`).
- `--idempotency-key <key>`: isteğe bağlı eşgüçlülük anahtarı.

`system.run` ve `system.run.prepare` burada engellenir; bunun yerine kabuk yürütmesi için `host=node` ile `exec` aracını kullanın. `system.which` kullanımına `invoke` üzerinden izin verilir.

## Bildirim, anlık bildirim, konum, ekran

```bash
openclaw nodes notify --node <id> --title "Build" --body "Done" --priority timeSensitive
openclaw nodes push --node <id> --title "OpenClaw" --environment sandbox
openclaw nodes location get --node <id> --accuracy precise
openclaw nodes screen record --node <id> --duration 10s --fps 10 --out ./clip.mp4
```

- `notify`, `system.notify` bildiren bir Node'da yerel bildirim gönderir; buna macOS, iOS, Android ve doğrudan watchOS Node'ları dahildir. Doğrudan watchOS teslimatı için OpenClaw'ın etkin olması gerekir. `--title` veya `--body` gerektirir. Seçenekler: `--sound <name>`, `--priority <passive|active|timeSensitive>`, `--delivery <system|overlay|auto>` (varsayılan `system`), `--invoke-timeout <ms>` (varsayılan `15000`).
- `push`, bir iOS Node'una APNs test anlık bildirimi gönderir. Seçenekler: `--title <text>` (varsayılan `OpenClaw`), `--body <text>`, algılanan APNs ortamını geçersiz kılmak için `--environment <sandbox|production>`.
- `location get`, Node'un geçerli konumunu alır. Seçenekler: `--max-age <ms>` (önbelleğe alınmış bir konum tespitini yeniden kullanır), `--accuracy <coarse|balanced|precise>`, `--location-timeout <ms>` (varsayılan `10000`), `--invoke-timeout <ms>` (varsayılan `20000`).
- `screen record`, kısa bir klip kaydeder ve kaydedilen yolu yazdırır (veya `--json` ile JSON yazar). Seçenekler: `--screen <index>` (varsayılan `0`), `--duration <ms|10s>` (varsayılan `10000`), `--fps <fps>` (varsayılan `10`), `--no-audio`, `--out <path>`, `--invoke-timeout <ms>` (varsayılan `120000`).

Kamera ve Canvas komutlarının kendi belgeleri vardır: [Kamera Node'ları](/tr/nodes/camera), [Canvas](/tr/platforms/mac/canvas). Canvas, paketle birlikte sunulan deneysel Canvas Plugin'i tarafından uygulanır; çekirdek, `openclaw nodes canvas` öğesini uyumluluk bağlama noktası olarak korur.

## İlgili

- [CLI başvurusu](/tr/cli)
- [Node'lar](/tr/nodes)
