---
read_when:
    - Telemetri / gizlilik kontrolleri üzerinde çalışılıyor
    - Hangi verilerin toplandığına ilişkin sorular
summary: ClawHub CLI tarafından toplanan kurulum telemetrisi ve bunun nasıl devre dışı bırakılacağı.
x-i18n:
    generated_at: "2026-07-26T23:14:10Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: a02bb1c76fea3105255235f6314ade73f260f692d6eb1b41f8001dc84db6ded7
    source_path: clawhub/telemetry.md
    workflow: 16
---

# Telemetri

ClawHub, toplu skill ve plugin yükleme sayılarını hesaplamak için asgari düzeyde CLI telemetrisi kullanır.

## Telemetri ne zaman toplanır?

Telemetri yalnızca şu durumlarda gönderilir:

- CLI'da oturum açmış olmanız.
- `clawhub install <slug>` komutunu çalıştırmanız veya kimliği doğrulanmış bir
  `openclaw plugins install clawhub:<package>` yüklemesini tamamlamanız.
- Telemetrinin **devre dışı bırakılmamış** olması (aşağıdaki “Nasıl devre dışı bırakılır?” bölümüne bakın).

Oturum açmadıysanız hiçbir şey bildirilmez.

## Neleri topluyoruz?

Bir skill veya plugin yüklendikten ve yerel yükleme kaydı kalıcı olarak saklandıktan sonra CLI,
mümkün olan en iyi şekilde tek bir yükleme olayı gönderir.

Olay şunları içerir:

- Yüklenen skill slug'ı veya standart plugin paket adı.
- `version`: biliniyorsa yüklenen sürüm.

### Neleri _toplamıyoruz_?

- Klasör yolları veya klasörlerden türetilen tanımlayıcılar.
- Dosya içerikleri.
- Çalıştırma başına günlükler, istemler veya diğer CLI çıktıları.

## Yükleme sayıları

ClawHub, skill'ler için şunları tutar:

- `installsAllTime`: skill için en az bir CLI yüklemesi bildiren benzersiz kullanıcılar.
- `installsCurrent`: bir yükleme bildiren ve telemetri verilerini silmemiş benzersiz
  kullanıcılar.

ClawHub, plugin'ler için her kullanıcı ve paket tarafından bildirilen ilk başarılı yüklemeyi sayar.
Tekrarlanan yüklemeler ve güncellemeler, toplu yükleme sayısını artırmadan kayıtlı sürümü
yeniler.

## Şeffaflık ve kullanıcı denetimleri

Herkes yalnızca **toplu yükleme sayaçlarını** görür.

Hesabınızı silmek, telemetri verilerinizi de siler ve bu verilerin yükleme sayaçlarına katkısını
kaldırır.

## Telemetri nasıl devre dışı bırakılır?

Ortam değişkenini ayarlayın:

```bash
export CLAWHUB_DISABLE_TELEMETRY=1
```

Bu ayar yapıldığında CLI, yükleme telemetrisi göndermez.
