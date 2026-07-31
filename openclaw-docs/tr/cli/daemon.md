---
read_when:
    - Betiklerde hâlâ `openclaw daemon ...` kullanıyorsunuz
    - Hizmet yaşam döngüsü komutlarına ihtiyacınız var (install/start/stop/restart/status)
summary: '`openclaw daemon` için CLI başvurusu (Gateway hizmet yönetimi için eski takma ad)'
title: Arka Plan Hizmeti
x-i18n:
    generated_at: "2026-07-26T22:41:04Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 629852ebf3efe86dedc4c84f6ddc9349b25ddde832df5d78521641fe4b137658
    source_path: cli/daemon.md
    workflow: 16
---

# `openclaw daemon`

Gateway hizmet yönetimi için eski takma ad. `openclaw daemon ...`, `openclaw gateway ...` ile aynı hizmet denetimi komutlarına eşlenir. Güncel belgeler ve örnekler için [`openclaw gateway`](/tr/cli/gateway) tercih edilmelidir.

## Kullanım

```bash
openclaw daemon status
openclaw daemon install
openclaw daemon start
openclaw daemon stop
openclaw daemon restart
openclaw daemon uninstall
```

## Alt komutlar ve seçenekler

| Alt komut  | Seçenekler                                                                                          |
| ----------- | ------------------------------------------------------------------------------------------------ |
| `status`    | `--url`, `--token`, `--password`, `--timeout`, `--no-probe`, `--require-rpc`, `--deep`, `--json` |
| `install`   | `--port`, `--runtime <node>`, `--token`, `--wrapper <path>`, `--force`, `--json`                 |
| `uninstall` | `--json`                                                                                         |
| `start`     | `--json`                                                                                         |
| `stop`      | `--json`, `--disable` (yalnızca launchd: bir sonraki başlatmaya kadar KeepAlive/RunAtLoad'u kalıcı olarak devre dışı bırakır) |
| `restart`   | `--force`, `--safe`, `--skip-deferral`, `--wait <duration>`, `--json`                            |

- `status`: hizmetin kurulum durumunu (launchd/systemd/schtasks) gösterir ve Gateway sağlığını yoklar.
- `install`: hizmeti kurar; `--force` mevcut bir kurulumu yeniden kurar/üzerine yazar.
- `restart --safe`: çalışan Gateway'den etkin işler için ön kontrol yapmasını ve işler tamamlandıktan sonra, 5 dakikayla sınırlı tek bir birleştirilmiş yeniden başlatma planlamasını ister. Bu süre dolduğunda yeniden başlatma yine de zorlanır. Düz `restart` doğrudan hizmet yöneticisini kullanır; `--force` anında geçersiz kılma seçeneğidir.
- `restart --safe --skip-deferral`: etkin iş erteleme kapısını atlar; böylece engelleyiciler bildirilse bile Gateway hemen yeniden başlatılır. `--safe` gerektirir.

## Notlar

- `status`, mümkün olduğunda yoklama kimlik doğrulaması için yapılandırılmış kimlik doğrulama SecretRef'lerini çözümler. Gerekli bir SecretRef çözümlenemezse `status --json`, `rpc.authWarning` bildirir; `--token`/`--password` değerlerini açıkça iletin veya önce gizli bilgi kaynağını çözümleyin. Yoklama bunun dışında başarılı olduğunda çözümlenmemiş kimlik doğrulama uyarıları bastırılır.
- `status --deep`, Gateway benzeri diğer hizmetler için en iyi çabaya dayalı sistem düzeyinde bir tarama ekler (temizleme ipuçlarını yazdırır; makine başına bir Gateway önerisi geçerliliğini korur) ve yapılandırma doğrulamasını Plugin duyarlı modda çalıştırarak hızlı varsayılan yolun atladığı Plugin manifest uyarılarını gösterir.
- Linux systemd kurulumlarında token sapması denetimleri hem `Environment=` hem de `EnvironmentFile=` birim kaynaklarını inceler.
- Token sapması denetimleri, birleştirilmiş çalışma zamanı ortamını (önce hizmet komutu ortamı, ardından işlem ortamı) kullanarak `gateway.auth.token` SecretRef'lerini çözümler. Token kimlik doğrulaması fiilen etkin değilse (`password`/`none`/`trusted-proxy` seçeneklerinden `gateway.auth.mode` veya parola öncelik kazanabilecekken ayarlanmamışsa), yapılandırma token'ının çözümlenmesi atlanır.
- `install`, SecretRef tarafından yönetilen bir `gateway.auth.token` değerinin çözümlenebilir olduğunu doğrular ancak çözümlenen değeri hiçbir zaman hizmet ortamı meta verilerine kalıcı olarak kaydetmez; çözümleyemezse kurulum güvenli biçimde başarısız olur.
- Hem `gateway.auth.token` hem de `gateway.auth.password` yapılandırılmışsa ve `gateway.auth.mode` ayarlanmamışsa `install`, mod açıkça ayarlanana kadar işlemi engeller.
- macOS'ta `install`, gizli bilgileri `EnvironmentVariables` içine gömmek yerine LaunchAgent plist'lerini ve oluşturulan ortam dosyasını/sarmalayıcıyı yalnızca sahibin erişimine açık tutar (`0600`/`0700` modu).
- Tek bir ana makinede birden fazla Gateway çalıştırırken bağlantı noktalarını, yapılandırmayı/durumu ve çalışma alanlarını birbirinden ayırın. Bkz. [Birden fazla Gateway](/tr/gateway#multiple-gateways-same-host).

## İlgili

- [CLI referansı](/tr/cli)
- [Gateway operasyon kılavuzu](/tr/gateway)
