---
read_when:
    - OpenClaw'u bir makineden kaldırmak istiyorsunuz
    - Gateway hizmeti kaldırma işleminden sonra hâlâ çalışıyor
summary: OpenClaw'u tamamen kaldırma (CLI, hizmet, durum, çalışma alanı)
title: Kaldırma
x-i18n:
    generated_at: "2026-07-26T22:50:30Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 84f01dc11defe6f19c89232375e48bad383b2e71379f47f43e759d3d7bb908b5
    source_path: install/uninstall.md
    workflow: 16
---

İki yol:

- `openclaw` hâlâ yüklüyse **Kolay yol**.
- CLI kaldırılmış ancak hizmet hâlâ çalışıyorsa **Hizmeti elle kaldırma**.

## Kolay yol (CLI hâlâ yüklü)

Önerilen: yerleşik kaldırıcıyı kullanın:

```bash
openclaw uninstall
```

`--workspace` seçeneğini de belirlemediğiniz sürece durum verileri kaldırılırken yapılandırılmış çalışma alanı dizinleri korunur.

Nelerin kaldırılacağını önizleyin (güvenli):

```bash
openclaw uninstall --dry-run --all
```

Etkileşimsiz (otomasyon / npx). Dikkatle ve yalnızca kapsamları doğruladıktan sonra kullanın:

```bash
openclaw uninstall --all --yes --non-interactive
npx -y openclaw uninstall --all --yes --non-interactive
```

Bayraklar: `--service`, `--state`, `--workspace`, `--app` ayrı kapsamları seçer; `--all` dördünü birden seçer.

Elle uygulanan adımlar (aynı sonuç):

1. Gateway hizmetini durdurun:

```bash
openclaw gateway stop
```

2. Gateway hizmetini kaldırın (launchd/systemd/schtasks):

```bash
openclaw gateway uninstall
```

3. Durum verilerini ve yapılandırmayı silin:

```bash
rm -rf "${OPENCLAW_STATE_DIR:-$HOME/.openclaw}"
```

`OPENCLAW_CONFIG_PATH` için durum dizininin dışında özel bir konum belirlediyseniz o dosyayı da silin.
Durum dizini içindeki `~/.openclaw/workspace` gibi bir çalışma alanını korumak istiyorsanız `rm -rf` komutunu çalıştırmadan önce onu başka bir yere taşıyın veya durum dizininin içeriğini seçerek silin.

4. Çalışma alanınızı silin (isteğe bağlı, ajan dosyalarını kaldırır):

```bash
rm -rf ~/.openclaw/workspace
```

5. CLI kurulumunu kaldırın (kullandığınız yöntemi seçin):

```bash
npm rm -g openclaw
pnpm remove -g openclaw
bun remove -g openclaw
```

6. macOS uygulamasını yüklediyseniz:

```bash
rm -rf /Applications/OpenClaw.app
```

Notlar:

- Profiller (`--profile` / `OPENCLAW_PROFILE`) kullandıysanız her durum dizini için 3. adımı tekrarlayın (varsayılanlar: `~/.openclaw-<profile>`).
- Uzak modda durum dizini **Gateway ana makinesinde** bulunur; bu nedenle 1-4. adımları orada da uygulayın.

## Hizmeti elle kaldırma (CLI yüklü değil)

Gateway hizmeti çalışmaya devam ediyor ancak `openclaw` bulunamıyorsa bunu kullanın.

### macOS (launchd)

Varsayılan etiket `ai.openclaw.gateway` şeklindedir (veya bir profille `ai.openclaw.<profile>`):

```bash
launchctl bootout gui/$UID/ai.openclaw.gateway
rm -f ~/Library/LaunchAgents/ai.openclaw.gateway.plist
```

Bir profil kullandıysanız etiketi ve plist adını `ai.openclaw.<profile>` ile değiştirin.

### Linux (systemd kullanıcı birimi)

Varsayılan birim adı `openclaw-gateway.service` şeklindedir (veya `openclaw-gateway-<profile>.service`). Çok eski kurulumlardan yükseltilmiş makinelerde yeniden adlandırma öncesindeki bir `clawdbot-gateway.service` birimi hâlâ bulunabilir; `openclaw uninstall` / `openclaw gateway uninstall` bunu otomatik olarak algılar ve kaldırır.

```bash
systemctl --user disable --now openclaw-gateway.service
rm -f ~/.config/systemd/user/openclaw-gateway.service
systemctl --user daemon-reload
```

### Windows (Zamanlanmış Görev)

Varsayılan görev adı `OpenClaw Gateway` şeklindedir (veya `OpenClaw Gateway (<profile>)`).
Görev, durum dizininizin altında penceresiz bir `gateway.vbs` betiği başlatır; bu betik de
`gateway.cmd` komutunu çalıştırır; ikisini de kaldırın.

```powershell
schtasks /Delete /F /TN "OpenClaw Gateway"
Remove-Item -Force "$env:USERPROFILE\.openclaw\gateway.cmd" -ErrorAction SilentlyContinue
Remove-Item -Force "$env:USERPROFILE\.openclaw\gateway.vbs" -ErrorAction SilentlyContinue
```

Bir profil kullandıysanız eşleşen görev adını ve `~\.openclaw-<profile>` altındaki `gateway.cmd` /
`gateway.vbs` dosyalarını silin.

## Normal kurulum ve kaynak kod deposundan çalıştırma

### Normal kurulum (install.sh / npm / pnpm / bun)

`https://openclaw.ai/install.sh` veya `install.ps1` kullandıysanız CLI, `npm install -g openclaw@latest` ile yüklenmiştir.
`npm rm -g openclaw` ile kaldırın (veya bu yöntemle yüklediyseniz `pnpm remove -g` / `bun remove -g` kullanın).

### Kaynak kod deposundan çalıştırma (git clone)

Bir depo çalışma kopyasından çalıştırıyorsanız (`git clone` + `openclaw ...` / `bun run openclaw ...`):

1. Depoyu silmeden **önce** Gateway hizmetini kaldırın (yukarıdaki kolay yolu veya hizmeti elle kaldırma yöntemini kullanın).
2. Depo dizinini silin.
3. Durum verilerini ve çalışma alanını yukarıda gösterildiği gibi kaldırın.

## İlgili

- [Kuruluma genel bakış](/tr/install)
- [Geçiş kılavuzu](/tr/install/migrating)
