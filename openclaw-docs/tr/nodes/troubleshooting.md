---
read_when:
    - Node bağlı ancak kamera/tuval/ekran/exec araçları çalışmıyor
    - Node eşleştirmesi ile onaylar arasındaki zihinsel modele ihtiyacınız var
summary: Node eşleştirme, ön planda çalışma gereksinimleri, izinler ve araç hatalarıyla ilgili sorunları giderme
title: Node sorun giderme
x-i18n:
    generated_at: "2026-07-26T23:26:42Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 4a7ee9e48985805e91cd5acfa1b9f6b676b7e67236ce29fe91e2c8d03002e5c4
    source_path: nodes/troubleshooting.md
    workflow: 16
---

Durumda bir Node görünürken Node araçları başarısız oluyorsa bu sayfayı kullanın.

## Komut sıralaması

```bash
openclaw status
openclaw gateway status
openclaw logs --follow
openclaw doctor
openclaw channels status --probe
```

Ardından Node'a özgü kontrolleri çalıştırın:

```bash
openclaw nodes status
openclaw nodes describe --node <idOrNameOrIp>
openclaw approvals get --node <idOrNameOrIp>
```

Sağlıklı çalışma belirtileri:

- Node bağlıdır ve `node` rolü için eşleştirilmiştir.
- `nodes describe`, çağırdığınız yeteneği içerir.
- Çalıştırma onayları, beklenen modu/izin verilenler listesini gösterir.

## Ön planda çalışma gereksinimleri

`canvas.*`, `camera.*` ve `screen.*`, iOS/Android Node'larında yalnızca ön planda çalışır.

Hızlı kontrol ve düzeltme:

```bash
openclaw nodes describe --node <idOrNameOrIp>
openclaw nodes canvas snapshot --node <idOrNameOrIp>
openclaw logs --follow
```

`NODE_BACKGROUND_UNAVAILABLE` görürseniz Node uygulamasını ön plana getirin ve yeniden deneyin.

## İzinler matrisi

| Yetenek                      | iOS                                     | Android                                      | macOS Node uygulaması            | Tipik hata kodu                               |
| ---------------------------- | --------------------------------------- | -------------------------------------------- | -------------------------------- | --------------------------------------------- |
| `camera.snap`, `camera.clip` | Kamera (+ klip sesi için mikrofon)       | Kamera (+ klip sesi için mikrofon)            | Kamera (+ klip sesi için mikrofon) | `*_PERMISSION_REQUIRED`                       |
| `screen.record`              | Ekran Kaydı (+ isteğe bağlı mikrofon)    | Ekran yakalama istemi (+ isteğe bağlı mikrofon) | Ekran Kaydı                    | `*_PERMISSION_REQUIRED`                       |
| `computer.act`               | geçerli değil                           | geçerli değil                                | Erişilebilirlik + Ekran Kaydı     | `COMPUTER_DISABLED`, `ACCESSIBILITY_REQUIRED` |
| `location.get`               | Kullanırken veya Her Zaman (moda bağlı) | Moda göre ön plan/arka plan konumu            | Konum izni                       | `LOCATION_PERMISSION_REQUIRED`                |
| `system.run`                 | geçerli değil (Node ana makine yolu)    | geçerli değil (Node ana makine yolu)          | Çalıştırma onayları gerekli       | `SYSTEM_RUN_DENIED`                           |

## Eşleştirme ve onaylar

Bir Node komutunun başarılı olup olmayacağını üç ayrı geçit denetler:

1. **Cihaz eşleştirmesi**: Bu Node, Gateway'e bağlanabilir mi?
2. **Gateway Node komutu politikası**: RPC komut kimliğine `gateway.nodes.commands.allow` / `gateway.nodes.commands.deny` ve platform varsayılanları tarafından izin veriliyor mu?
3. **Çalıştırma onayları**: Bu Node, belirli bir kabuk komutunu yerel olarak çalıştırabilir mi?

Node eşleştirmesi, komut başına bir onay yüzeyi değil, bir kimlik/güven geçididir. `system.run` için Node başına politika, Gateway eşleştirme kaydında değil, ilgili Node'un çalıştırma onayları dosyasında (`openclaw approvals get --node ...`) bulunur.

Hızlı kontroller:

```bash
openclaw devices list
openclaw nodes status
openclaw approvals get --node <idOrNameOrIp>
openclaw approvals allowlist add --node <idOrNameOrIp> "/usr/bin/uname"
```

- Eşleştirme eksik: Önce Node cihazını onaylayın.
- `nodes describe` içinde bir komut eksik: Gateway Node komutu politikasını ve Node'un bağlanırken bu komutu gerçekten bildirmiş olup olmadığını kontrol edin.
- Eşleştirme sorunsuz ancak `system.run` başarısız oluyor: İlgili Node'daki çalıştırma onaylarını/izin verilenler listesini düzeltin.

Onay destekli `host=node` çalıştırmalarında Gateway, yürütmeyi hazırlanmış standart `systemRunPlan` öğesine de bağlar. Daha sonraki bir çağıran, onaylanan çalıştırma iletilmeden önce komutu, cwd'yi veya oturum meta verilerini değiştirirse Gateway, düzenlenmiş yüke güvenmek yerine çalıştırmayı onay uyuşmazlığı nedeniyle reddeder.

## Yaygın Node hata kodları

| Kod                                    | Anlamı                                                                                                                                                                                  |
| -------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `NODE_BACKGROUND_UNAVAILABLE`          | Uygulama arka plandadır; ön plana getirin.                                                                                                                                               |
| `CAMERA_DISABLED`                      | Node ayarlarında kamera anahtarı devre dışıdır.                                                                                                                                          |
| `*_PERMISSION_REQUIRED`                | İşletim sistemi izni eksik veya reddedilmiştir.                                                                                                                                          |
| `LOCATION_DISABLED`                    | Konum modu kapalıdır.                                                                                                                                                                    |
| `LOCATION_PERMISSION_REQUIRED`         | İstenen konum modu için izin verilmemiştir.                                                                                                                                              |
| `LOCATION_BACKGROUND_UNAVAILABLE`      | Uygulama arka plandadır ancak yalnızca Kullanırken izni vardır.                                                                                                                          |
| `COMPUTER_DISABLED`                    | macOS uygulamasında **Allow Computer Control** seçeneğini etkinleştirin, ardından eşleştirme güncellemesini onaylayın.                                                                   |
| `ACCESSIBILITY_REQUIRED`               | macOS Sistem Ayarları'nda geçerli OpenClaw uygulama paketine Erişilebilirlik izni verin.                                                                                                 |
| `SYSTEM_RUN_DENIED: approval required` | Çalıştırma isteği açık onay gerektirir.                                                                                                                                                  |
| `SYSTEM_RUN_DENIED: allowlist miss`    | Komut, izin verilenler listesi modu tarafından engellendi. Windows Node ana makinelerinde `cmd.exe /c ...` gibi kabuk sarmalayıcı biçimleri, soru akışı üzerinden onaylanmadıkları sürece izin verilenler listesi modunda liste dışı kabul edilir. |

## Hızlı kurtarma döngüsü

```bash
openclaw nodes status
openclaw nodes describe --node <idOrNameOrIp>
openclaw approvals get --node <idOrNameOrIp>
openclaw logs --follow
```

Sorun hâlâ sürüyorsa:

- Cihaz eşleştirmesini yeniden onaylayın.
- Node uygulamasını yeniden açın (ön planda).
- İşletim sistemi izinlerini yeniden verin.
- Çalıştırma onayı politikasını yeniden oluşturun/ayarlayın.

Bilgisayar denetimi için ayrıca görsel yetenekli bir aracının `computer` aracını sunduğunu, `screen.snapshot` işleminin Ekran Kaydı izniyle başarılı olduğunu ve `/phone status` öğesinin amaçladığınız geçici veya kalıcı Gateway yetkilendirmesini gösterdiğini doğrulayın. Bir `gateway.nodes.commands.deny` girdisi her zaman `gateway.nodes.commands.allow` ayarını geçersiz kılar.

## İlgili konular

- [Node'lara genel bakış](/tr/nodes)
- [Kamera Node'ları](/tr/nodes/camera)
- [Konum komutu](/tr/nodes/location-command)
- [Bilgisayar kullanımı](/tr/nodes/computer-use)
- [Çalıştırma onayları](/tr/tools/exec-approvals)
- [Gateway eşleştirmesi](/tr/gateway/pairing)
- [Gateway sorun giderme](/tr/gateway/troubleshooting)
- [Kanal sorun giderme](/tr/channels/troubleshooting)
