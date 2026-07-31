---
read_when:
    - Mantis Slack masaüstü QA'sını GitHub'dan veya yerel olarak çalıştırma
    - Yavaş Mantis Slack masaüstü çalıştırmalarında hata ayıklama
    - Kaynak, önceden hazırlanmış veya sıcak kiralama modunu seçme
    - Bir PR'a ekran görüntüsü ve video kanıtı gönderme
summary: 'Mantis Slack masaüstü QA için operatör çalışma kılavuzu: GitHub tetikleme, yerel CLI, hazır VNC kiralamaları, hydrate modları, zamanlama yorumlama, yapıtlar ve hata yönetimi.'
title: Mantis Slack masaüstü çalışma kılavuzu
x-i18n:
    generated_at: "2026-07-26T23:54:31Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: b3e956d99fc43a7b6fe65e2e820812b0e0e8b9e32badd25be27c74d302ab30dc
    source_path: concepts/mantis-slack-desktop-runbook.md
    workflow: 16
---

Mantis Slack masaüstü QA, Linux masaüstü, VNC ile kurtarma, Slack Web, gerçek bir OpenClaw gateway, ekran görüntüleri, videolar ve PR kanıt yorumu gerektiren Slack sınıfı hatalar için gerçek kullanıcı arayüzü hattıdır. Birim testleri veya başsız Slack canlı hattı hatayı kanıtlayamadığında bunu kullanın.

## Depolama modeli

Mantis üç depolama katmanı kullanır:

- **Sağlayıcı imajı** - Crabbox'a aittir ve bulut sağlayıcısı hesabında depolanır.
  Makine yeteneklerini (Chrome/Chromium, ffmpeg, scrot,
  Node/corepack/pnpm, yerel derleme araçları) ve boş önbellek dizinlerini içerir.
- **Sıcak kiralama durumu** - mevcut operatör oturumuna aittir. Kiralama etkin olduğu sürece
  oturum açılmış bir tarayıcı profili, `/var/cache/crabbox/pnpm` ve hazırlanmış bir kaynak
  çalışma kopyası içerebilir.
- **Mantis yapıtları** - OpenClaw çalıştırmasına aittir.
  `.artifacts/qa-e2e/mantis/...` altında bulunur; GitHub Actions bunları yükler ve Mantis
  GitHub App, PR üzerinde satır içi kanıt yorumu yapar.

Gizli bilgileri, tarayıcı çerezlerini, Slack oturum açma durumunu, depo çalışma kopyalarını,
`node_modules` veya `dist/` değerlerini asla bir sağlayıcı imajına yerleştirmeyin.

## GitHub tetikleme

İş akışını `main` üzerinden çalıştırın:

```bash
gh workflow run mantis-slack-desktop-smoke.yml \
  --ref main \
  -f candidate_ref=<trusted-ref-or-sha> \
  -f pr_number=<pr-number> \
  -f scenario_id=slack-canary \
  -f crabbox_provider=aws \
  -f keep_vm=false \
  -f hydrate_mode=source
```

İş akışı canlı kimlik bilgilerini kullandığından `candidate_ref` kısıtlanmıştır: geçerli `main` üst soyuna, bir sürüm etiketine veya
`openclaw/openclaw` içindeki açık bir PR başına çözümlenmelidir.

İş akışı şunları üretir:

- yüklenen yapıt `mantis-slack-desktop-smoke-<run-id>-<attempt>`
- Mantis GitHub App tarafından oluşturulan satır içi PR yorumu
- `slack-desktop-smoke.png`, `slack-desktop-smoke.mp4`
- `slack-desktop-smoke-preview.gif`, `slack-desktop-smoke-change.mp4`
- `mantis-slack-desktop-smoke-summary.json`, `mantis-slack-desktop-smoke-report.md`
- uzak günlükler: `slack-desktop-command.log`, `openclaw-gateway.log`, `chrome.log`, `ffmpeg.log`

PR yorumu, gizli `<!-- mantis-slack-desktop-smoke -->` işaretçisi aracılığıyla yerinde güncellenir.

## Yerel CLI

Soğuk kaynak kanıtı:

```bash
pnpm openclaw qa mantis slack-desktop-smoke \
  --provider aws \
  --class standard \
  --gateway-setup \
  --credential-source convex \
  --credential-role maintainer \
  --provider-mode live-frontier \
  --model openai/gpt-5.4 \
  --alt-model openai/gpt-5.4 \
  --scenario slack-canary \
  --hydrate-mode source
```

VNC ile kurtarma için VM'yi koruyun:

```bash
pnpm openclaw qa mantis slack-desktop-smoke \
  --provider aws \
  --class standard \
  --gateway-setup \
  --scenario slack-canary \
  --keep-lease
```

VNC'yi açın:

```bash
crabbox vnc --provider aws --id <cbx_id> --open
```

Sıcak bir kiralamayı yeniden kullanın:

```bash
pnpm openclaw qa mantis slack-desktop-smoke \
  --provider aws \
  --lease-id <cbx_id-or-slug> \
  --gateway-setup \
  --scenario slack-canary \
  --hydrate-mode source
```

`--hydrate-mode prehydrated` seçeneğini yalnızca yeniden kullanılan uzak çalışma alanında zaten
`node_modules` ve derlenmiş bir `dist/` bulunduğunda kullanın; aksi takdirde Mantis güvenli biçimde başarısız olur.

Yerel Slack onay kullanıcı arayüzünü kanıtlayın:

```bash
pnpm openclaw qa mantis slack-desktop-smoke \
  --provider aws \
  --class standard \
  --approval-checkpoints \
  --credential-source convex \
  --credential-role maintainer \
  --hydrate-mode source
```

`--approval-checkpoints` ile `--gateway-setup` birbirini dışlar. Açık bir onay kontrol noktası `--scenario` iletmediğiniz sürece
isteğe bağlı `slack-approval-exec-native` ve `slack-approval-plugin-native`
senaryolarını çalıştırır; diğer Slack senaryoları VM başlamadan önce reddedilir. Slack QA çalıştırıcısı,
gözlemlediği gerçek Slack API mesajından her kontrol noktası JSON dosyasını yazar; ardından
uzak izleyici bu mesajı
`approval-checkpoints/<scenario>-pending.png` ve
`approval-checkpoints/<scenario>-resolved.png` içine işler. Herhangi bir
kontrol noktası JSON'u, mesaj kanıtı, onay JSON'u veya işlenmiş ekran görüntüsü eksik
ya da boşsa çalıştırma başarısız olur.

Soğuk GitHub Actions kiralamalarında Slack Web çerezleri bulunmaz; bu nedenle tarayıcı yakalamaları
Slack oturum açma ekranına ulaşabilir. Onay kontrol noktası kanıtı için
`slack-desktop-smoke.png` yerine işlenmiş kontrol noktası görüntülerine ve Slack QA yapıtlarına güvenin. Yalnızca tarayıcı ekran görüntüsünün
Slack Web'i göstermesi gerektiğinde, Slack Web'de elle oturum açılmış bir profile sahip ve korunan sıcak bir kiralama kullanın.

## Hazırlama modları

| Mod          | Kullanım durumu                                  | Uzak davranış                                                                       | Ödünleşim                                                 |
| ------------- | ----------------------------------------- | ------------------------------------------------------------------------------------- | -------------------------------------------------------- |
| `source`      | Normal PR kanıtı, soğuk makineler, CI        | VM içinde `pnpm install --frozen-lockfile --prefer-offline` ve `pnpm build` çalıştırır | En yavaş, en güçlü kaynak çalışma kopyası kanıtı                 |
| `prehydrated` | Yeniden kullanılan bir kiralamayı bilinçli olarak hazırladığınızda | Mevcut `node_modules` ve `dist/` gerektirir; kurulum/derlemeyi atlar                     | Hızlıdır, ancak yalnızca operatör denetimindeki sıcak kiralamalar için geçerlidir |

GitHub Actions, VM çalıştırmasından önce her zaman aday çalışma kopyasını hazırlar.
pnpm deposu; işletim sistemi, Node sürümü ve kilit dosyasına göre önbelleğe alınır. VM'deki `source` çalıştırması da
mevcut olduğunda `/var/cache/crabbox/pnpm` değerini yeniden kullanır.

## Zamanlama yorumu

`mantis-slack-desktop-smoke-report.md` aşama zamanlamalarını içerir:

- `crabbox.warmup` - bulut sağlayıcısının önyüklenmesi, masaüstü/tarayıcı hazırlığı, SSH.
- `crabbox.inspect` - kiralama meta verisi araması.
- `credentials.prepare` - Convex kimlik bilgisi kiralamasının alınması.
- `crabbox.remote_run` - eşitleme, tarayıcının başlatılması, OpenClaw kurulumu/derlemesi veya
  hazırlama doğrulaması, gateway başlatma, ekran görüntüsü ve video yakalama.
- `artifacts.copy` - VM'den geri rsync aktarımı.

Crabbox sıfır olmayan bir uzak durum döndürdüğünde, ancak Mantis OpenClaw gateway
kurulumunun tamamlandığını veya Slack QA komutunun başarıyla çıktığını kanıtlayan meta verileri kopyaladığında,
`crabbox.remote_run` içinde `accepted` gösterilebilir.
`accepted` değerini başarısız bir senaryo olarak değil, açıklamalı başarı olarak değerlendirin.

Bir çalıştırma yavaşsa:

- Isınma baskınsa: daha iyi bir Crabbox sağlayıcı imajını önceden oluşturun veya terfi ettirin.
- `source` içinde `remote_run` baskınsa: sıcak bir kiralama kullanın, pnpm deposunun
  yeniden kullanımını iyileştirin veya makine ön koşullarını sağlayıcı imajına taşıyın.
- `prehydrated` içinde `remote_run` baskınsa: uzak çalışma alanı gerçekte
  hazır değildir veya gateway/tarayıcı/Slack kurulumu yavaştır.
- Yapıt kopyalama baskınsa: video boyutunu ve yapıt dizininin içeriğini inceleyin.

## Kanıt kontrol listesi

İyi bir PR yorumu şunları gösterir:

- senaryo kimliği ve aday SHA
- GitHub Actions çalıştırma URL'si ve yapıt URL'si
- satır içi onay kontrol noktası ekran görüntüsü veya oturum açılmış
  sıcak bir kiralamadan alınan Slack Web ekran görüntüsü
- mevcut olduğunda satır içi animasyonlu önizleme
- tam MP4 ve kırpılmış MP4 bağlantıları
- başarı/başarısızlık durumu ve raporun zamanlama özeti

Ekran görüntülerini veya videoları depoya işlemeyin. Bunları GitHub
Actions yapıtlarında veya PR yorumunda tutun.

## Hata yönetimi

İş akışı VM çalıştırmasından önce başarısız olursa önce Actions işini inceleyin.
Tipik nedenler: güvenilmeyen `candidate_ref`, eksik ortam gizli bilgileri veya
aday kurulum/derleme hatası.

VM çalıştırması başarısız olur ancak ekran görüntüleri geri kopyalanırsa şunları inceleyin:

```bash
cat mantis-slack-desktop-smoke-report.md
cat mantis-slack-desktop-smoke-summary.json
cat slack-desktop-command.log
cat openclaw-gateway.log
cat chrome.log
cat ffmpeg.log
```

Çalıştırma kiralamayı koruduysa rapordaki `crabbox vnc ...`
komutuyla VNC'yi açın, ardından işiniz bittiğinde kiralamayı durdurun:

```bash
crabbox stop --provider aws <cbx_id-or-slug>
```

Slack oturumunun süresi dolduysa korunan bir kiralamada VNC üzerinden oturumu düzeltin ve
`--lease-id` ile yeniden çalıştırın. Bu tarayıcı profilini bir sağlayıcı imajına yerleştirmeyin.

## İlgili

- [QA'ya genel bakış](/tr/concepts/qa-e2e-automation)
- [Slack kanalı](/tr/channels/slack)
- [Test](/tr/help/testing)
