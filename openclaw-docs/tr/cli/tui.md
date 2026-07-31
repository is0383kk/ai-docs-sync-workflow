---
read_when:
    - Gateway için bir terminal kullanıcı arayüzü istiyorsunuz (uzaktan kullanıma uygun)
    - Komut dosyalarından url/token/oturum geçirmek istiyorsunuz
    - TUI'yi Gateway olmadan yerel gömülü modda çalıştırmak istiyorsunuz
    - openclaw chat veya openclaw tui --local kullanmak istiyorsunuz
summary: '`openclaw tui` için CLI referansı (Gateway destekli veya yerel gömülü terminal kullanıcı arayüzü)'
title: TUI
x-i18n:
    generated_at: "2026-07-26T22:42:53Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 5406f25bbd22c64867296c15112fafcaf8e1580c759e5fdc81fccfb62ae1e318
    source_path: cli/tui.md
    workflow: 16
---

# `openclaw tui`

Gateway'e bağlı terminal kullanıcı arayüzünü açın veya yerel gömülü
modda çalıştırın.

İlgili kılavuz: [TUI](/tr/web/tui)

## Seçenekler

| Bayrak                       | Varsayılan                                | Açıklama                                                                           |
| ---------------------------- | ----------------------------------------- | ---------------------------------------------------------------------------------- |
| `--local`                    | `false`                                   | Gateway yerine yerel gömülü agent çalışma zamanıyla çalıştırır.                    |
| `--url <url>`                | yapılandırmadaki `gateway.remote.url`     | Gateway WebSocket URL'si.                                                          |
| `--token <token>`            | (yok)                                     | Gerekiyorsa Gateway token'ı.                                                       |
| `--password <pass>`          | (yok)                                     | Gerekiyorsa Gateway parolası.                                                      |
| `--tls-fingerprint <sha256>` | `gateway.remote.tlsFingerprint`           | Sabitlenmiş bir `wss://` Gateway için beklenen TLS sertifikası parmak izi.         |
| `--session <key>`            | `main` (veya kapsam genelse `global`) | Oturum anahtarı. Bir agent çalışma alanında, önek belirtilmedikçe ilgili agent'ı otomatik seçer. |
| `--deliver`                  | `false`                                   | Asistan yanıtlarını yapılandırılmış kanallar üzerinden iletir.                     |
| `--thinking <level>`         | (model varsayılanı)                        | Düşünme düzeyi geçersiz kılması.                                                    |
| `--message <text>`           | (yok)                                     | Bağlandıktan sonra bir başlangıç mesajı gönderir.                                  |
| `--timeout-ms <ms>`          | `agents.defaults.timeoutSeconds`          | Agent zaman aşımı. Geçersiz değerler için uyarı günlüğe kaydedilir ve değerler yok sayılır. |
| `--history-limit <n>`        | `200`                                     | Bağlanırken yüklenecek geçmiş girdileri.                                            |

Takma adlar: `openclaw chat` ve `openclaw terminal`, örtük olarak
`--local` ile bu komutu çağırır.

## Notlar

- `--local`; `--url`, `--token`, `--password` veya `--tls-fingerprint` ile birlikte kullanılamaz.
- `tui`, mümkün olduğunda token/parola kimlik doğrulaması için yapılandırılmış Gateway kimlik doğrulama SecretRef'lerini
  çözümler (`env`/`file`/`exec` sağlayıcıları).
- Açık bir URL veya port belirtilmediğinde `tui`, çalışan Gateway tarafından kaydedilmiş
  etkin yerel Gateway portunu kullanır. Açıkça belirtilen `--url`, `OPENCLAW_GATEWAY_URL`,
  `OPENCLAW_GATEWAY_PORT` ve uzak Gateway yapılandırması önceliğini korur.
- Yapılandırılmış bir agent çalışma alanı dizininden başlatıldığında TUI, oturum anahtarı
  varsayılanı için ilgili agent'ı otomatik seçer (`--session` açıkça
  `agent:<id>:...` olarak belirtilmedikçe).
- Yerel mod, gömülü agent çalışma zamanını doğrudan kullanır. Çoğu yerel araç çalışır,
  ancak yalnızca Gateway'e özgü özellikler kullanılamaz.
- Yerel mod, TUI komut yüzeyine `/auth [provider]` ekler.
- Plugin onay kapıları yerel modda da geçerlidir: onay gerektiren araçlar
  terminalde bir karar girilmesini ister; hiçbir şey sessizce otomatik olarak onaylanmaz.
- Oturum [hedefleri](/tr/tools/goal) alt bilgide görünür ve
  `/goal` ile yönetilebilir.

## Örnekler

```bash
openclaw chat
openclaw tui --local
openclaw tui
openclaw tui --url ws://127.0.0.1:18789 --token <token>
openclaw tui --session main --deliver
openclaw chat --message "Yapılandırmamı belgelerle karşılaştır ve neleri düzeltmem gerektiğini söyle"
# bir agent çalışma alanında çalıştırıldığında ilgili agent'ı otomatik olarak belirler
openclaw tui --session bugfix
```

## Yapılandırma onarım döngüsü

Gömülü agent'ın mevcut yapılandırmayı incelemesi, belgelerle karşılaştırması
ve aynı terminalden onarılmasına yardımcı olması için yerel modu kullanın.

`openclaw config validate` zaten başarısız oluyorsa önce `openclaw configure` veya
`openclaw doctor --fix` komutunu çalıştırın; `openclaw chat`,
geçersiz yapılandırma korumasını atlamaz.

```bash
openclaw chat
```

Ardından TUI içinde:

```text
!openclaw config file
!openclaw docs gateway auth token secretref
!openclaw config validate
!openclaw doctor
```

Hedefli düzeltmeleri `openclaw config set` veya `openclaw configure` ile uygulayın, ardından
`openclaw config validate` komutunu yeniden çalıştırın. Bkz. [TUI](/tr/web/tui) ve
[Yapılandırma](/tr/cli/config).

## İlgili

- [CLI referansı](/tr/cli)
- [TUI](/tr/web/tui)
- [Hedef](/tr/tools/goal)
