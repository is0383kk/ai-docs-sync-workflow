---
read_when:
    - Tarayıcı otomasyonu için sitelerde oturum açmanız gerekir
    - X/Twitter'da güncellemeler yayımlamak istiyorsunuz
summary: Tarayıcı otomasyonu ve X/Twitter gönderileri için manuel oturum açma işlemleri
title: Tarayıcıda oturum açma
x-i18n:
    generated_at: "2026-07-26T23:02:55Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: bccd363cf7c9611f4687d50a92f7fb3e2fd1c1d67bb27a80c892f7ac58ae1f8f
    source_path: tools/browser-login.md
    workflow: 16
---

## Manuel oturum açma (önerilir)

Bir site oturum açmayı gerektirdiğinde, ana makine tarayıcısının `openclaw`
profilinde manuel olarak oturum açın. Kimlik bilgilerinizi modele vermeyin: otomatik oturum açma işlemleri sıklıkla
bot karşıtı savunmaları tetikler ve hesabın kilitlenmesine neden olabilir.

X/Twitter ve botlara karşı hassas diğer sitelerde hem okuma (arama/ileti dizileri) hem de
gönderi yayımlama için ana makine tarayıcısını (manuel oturum açma) kullanın. Korumalı alan tarayıcı oturumlarının
bot algılamayı tetikleme olasılığı daha yüksektir.

Ana tarayıcı belgelerine dönün: [Tarayıcı](/tr/tools/browser).

## Hangi Chrome profili kullanılır?

OpenClaw, günlük tarayıcı profilinizden ayrı, `openclaw` adlı (turuncu tonlu
arayüze sahip) özel bir Chrome profilini denetler.

Ajan tarayıcı aracı çağrıları için:

- Varsayılan seçim: ajan, yalıtılmış `openclaw` tarayıcısını kullanır.
- Yalnızca mevcut oturumların açık olması önemli olduğunda ve herhangi bir bağlanma istemine
  tıklamak/onay vermek için bilgisayarın başındaysanız `profile="user"` kullanın.
- Birden fazla kullanıcı tarayıcı profiliniz varsa tahmin etmek yerine profili açıkça
  belirtin.

`openclaw` profiline erişmenin iki yolu vardır:

1. Ajandan tarayıcıyı açmasını isteyin, ardından kendiniz oturum açın.
2. CLI aracılığıyla açın:

```bash
openclaw browser start
openclaw browser open https://x.com
```

Varsayılan olmayan bir profil için alt komuttan önce `--browser-profile <name>`
kullanın (varsayılan `openclaw` değeridir):

```bash
openclaw browser --browser-profile <name> open https://x.com
```

## Korumalı alan: ana makine tarayıcısına erişime izin verme

Ajan korumalı alandaysa `browser` araç çağrıları varsayılan olarak ana makineyi değil,
korumalı alan tarayıcısını kullanır. Ajanın bunun yerine ana makine tarayıcısını hedefleyebilmesi için:

```json5
{
  agents: {
    defaults: {
      sandbox: {
        mode: "non-main",
        browser: {
          allowHostControl: true,
        },
      },
    },
  },
}
```

CLI çağrıları her zaman ana makine tarayıcısını hedefler, korumalı alanı asla hedeflemez; dolayısıyla bu ayardan
bağımsız olarak ana makine tarayıcısını kendiniz açabilirsiniz:

```bash
openclaw browser --browser-profile openclaw open https://x.com
```

`sandbox.browser.allowHostControl: true` ayarlandıktan sonra ajanın `browser`
araç çağrıları da ana makineyi hedefleyebilir. Alternatif olarak, güncellemeleri yayımlayan
ajan için korumalı alanı devre dışı bırakın.

## İlgili

- [Tarayıcı](/tr/tools/browser)
- [Linux'ta tarayıcı sorunlarını giderme](/tr/tools/browser-linux-troubleshooting)
- [WSL2'de tarayıcı sorunlarını giderme](/tr/tools/browser-wsl2-windows-remote-cdp-troubleshooting)
