---
read_when:
    - Kanal durumunun ve son oturum alıcılarının hızlı bir değerlendirmesini istiyorsunuz
    - Hata ayıklama için yapıştırılabilir bir "tümü" durumu istiyorsunuz
summary: '`openclaw status` için CLI referansı (tanılama, yoklamalar, kullanım anlık görüntüleri)'
title: openclaw status
x-i18n:
    generated_at: "2026-07-26T23:36:41Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 52e8076339216f11ddadf35e0ae8e5604322a47a5a9e2ee305468b2624d7cfde
    source_path: cli/status.md
    workflow: 16
---

Kanallar + oturumlar için tanılama.

```bash
openclaw status
openclaw status --all
openclaw status --deep
openclaw status --usage
```

| Bayrak                  | Açıklama                                                                                                               |
| ----------------------- | ---------------------------------------------------------------------------------------------------------------------- |
| `--all`                 | Tam tanılama (salt okunur, yapıştırılabilir). Güvenlik denetimi, plugin uyumluluğu ve bellek vektörü yoklamalarını içerir. |
| `--deep`                | Canlı yoklamalar çalıştırır (WhatsApp Web + Telegram + Discord + Slack + Signal). Güvenlik denetimini de etkinleştirir. |
| `--usage`               | Normalleştirilmiş sağlayıcı kullanım aralıklarını `X% left` olarak yazdırır.                                  |
| `--json`                | Makine tarafından okunabilir çıktı.                                                                                    |
| `--verbose` / `--debug` | Rapordan önce ham Gateway hedef çözümlemesini de yazdırır.                                                             |

Düz `openclaw status`, hızlı salt okunur yolda kalır ve bellek incelemesini
atladığında belleği kullanılamaz olarak göstermek yerine `not checked`
olarak işaretler. Yoğun güvenlik denetimi, plugin uyumluluğu ve bellek vektörü
yoklamaları `openclaw status --all`, `openclaw status --deep`, `openclaw security audit`
ve `openclaw memory status --deep` seçeneklerine bırakılır.

## Oturum ve model çözümlemesi

- Oturum durumu çıktısı, `Execution:` ile `Runtime:` değerlerini birbirinden ayırır. `Execution`
  korumalı alan yoludur (`direct`, `docker/*`); `Runtime` ise
  oturumun `OpenClaw Default`, `OpenAI Codex`, bir CLI
  arka ucu veya `codex (acp/acpx)` gibi bir ACP arka ucu kullanıp kullanmadığını belirtir.
  Sağlayıcı/model/çalışma zamanı ayrımı için
  [Ajan çalışma zamanları](/tr/concepts/agent-runtimes) bölümüne bakın.
- Geçerli oturum anlık görüntüsü seyrek olduğunda `/status`, belirteç
  ve önbellek sayaçlarını en son transkript kullanım günlüğünden tamamlayabilir.
  Sıfır olmayan mevcut canlı değerler, transkript geri dönüş değerlerine göre
  yine önceliklidir.
- Transkript geri dönüşü, canlı oturum girdisinde eksik olduğunda etkin çalışma zamanı
  model etiketini de kurtarabilir. Bu transkript modeli seçilen modelden farklıysa
  durum, bağlam penceresini seçilen model yerine kurtarılan çalışma zamanı
  modeline göre çözümler.
- İstem boyutu hesabında transkript geri dönüşü, oturum meta verileri eksik
  veya daha küçük olduğunda istem odaklı daha büyük toplamı tercih eder; böylece
  özel sağlayıcı oturumları `0` belirteç gösterimine düşmez.
- Bir oturum, yapılandırılmış birincil modelden farklı bir modele sabitlendiğinde
  durum her iki değeri, nedeni (`session override`) ve
  `/model default` ipucunu yazdırır. Yapılandırılmış birincil model, yeni veya
  sabitlenmemiş oturumlara uygulanır; mevcut sabitlenmiş oturumlar temizlenene
  kadar oturum seçimlerini korur.
- Birden fazla ajan yapılandırıldığında çıktı, ajan başına oturum depolarını
  içerir.

## Kullanım ve kota

- `--usage`, normalleştirilmiş sağlayıcı kullanım aralıklarını `X% left` olarak yazdırır.
- MiniMax'in ham `usage_percent` / `usagePercent` alanları kalan kotayı
  gösterdiğinden OpenClaw, görüntülemeden önce bunları tersine çevirir; mevcut
  olduklarında sayı tabanlı alanlar önceliklidir. `model_remains` yanıtları
  sohbet modeli girdisini tercih eder, gerektiğinde zaman damgalarından aralık
  etiketini türetir ve plan etiketine model adını ekler.
- Model fiyatlandırması yenileme hataları, isteğe bağlı fiyatlandırma uyarıları
  olarak gösterilir. Bunlar Gateway'in veya kanalların sağlıksız olduğu
  anlamına gelmez.

## Genel bakış ve güncelleme durumu

- Genel bakış, kullanılabilir olduğunda Gateway + node ana makine hizmetinin
  kurulum/çalışma zamanı durumunu, ayrıca kısa Gateway işlem çalışma süresini
  ve ana makine sistem çalışma süresini içerir.
- Genel bakış, güncelleme kanalını + git SHA'sını (kaynak kod
  çalışma kopyaları için) içerir.
- Güncelleme bilgileri Genel Bakış bölümünde gösterilir; bir güncelleme varsa
  durum, `openclaw update` komutunu çalıştırma ipucunu yazdırır
  (bkz. [Güncelleme](/tr/install/updating)).

## Gizli değerler

- Çalışan Gateway, başlangıçtan, yeniden yüklemeden veya bir yapılandırma yazımından kaynaklanan yalıtılmış herhangi bir SecretRef sahibine sahipse durum, JSON'da `degradedSecretOwners` değerini ve insan tarafından okunabilir çıktıda **Bozulmuş gizli değerler** genel bakış satırını içerir. Her girdi sahibin adını, bozulma durumunu (`cold` veya `stale`), yapılandırma yollarını ve sansürlenmiş nedeni belirtir. Soğuk sahipler kullanılamaz; eski sahipler bilinen son iyi değerlerle çalışmaya devam eder.
- Salt okunur durum yüzeyleri (`status`, `status --json`, `status --all`),
  mümkün olduğunda hedefledikleri yapılandırma yolları için desteklenen
  SecretRef değerlerini çözümler.
- Desteklenen bir kanal SecretRef'i yapılandırılmış ancak geçerli komut yolunda
  kullanılamıyorsa durum salt okunur kalır ve çökmek yerine bozulmuş çıktı
  bildirir. İnsan tarafından okunabilir çıktı, "yapılandırılmış belirteç bu
  komut yolunda kullanılamıyor" gibi uyarılar gösterir; JSON çıktısı ise
  `secretDiagnostics` değerini içerir.
- Komuta yerel SecretRef çözümlemesi başarılı olduğunda durum, çözümlenmiş
  anlık görüntüyü tercih eder ve geçici "gizli değer kullanılamıyor" kanal
  işaretlerini nihai çıktıdan temizler.
- `status --all`, rapor oluşturmayı durdurmadan gizli değer tanılamalarını
  (okunabilirlik için kısaltılmış olarak) özetleyen bir Gizli Değerler genel bakış
  satırı ve tanılama bölümü içerir.

## Bellek

`status --json --all`, `plugins.slots.memory` tarafından seçilen etkin bellek plugin'i
çalışma zamanından bellek ayrıntılarını bildirir. Özel bellek plugin'leri,
yerleşik `memory.search.enabled` özelliğini devre dışı bırakıp yine de
kendi dosya, parça, vektör ve FTS durumlarını bildirebilir.

## İlgili

- [CLI referansı](/tr/cli)
- [Doctor](/tr/gateway/doctor)
