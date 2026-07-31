---
read_when:
    - ClawHub'ın ücretsiz promosyon modeli teklifini denemek istiyorsunuz
    - Bir sağlayıcıyı ilk katılım yerine bir promosyon aracılığıyla yapılandırıyorsunuz
summary: '`openclaw promos` için CLI referansı (promosyonlu model tekliflerini listeleme ve talep etme)'
title: Promosyonlar
x-i18n:
    generated_at: "2026-07-26T22:42:24Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 779eab2e9500b7376fabf9accb333e83ff5f84b085d51b7d551b5507b1e73adb
    source_path: cli/promos.md
    workflow: 16
---

# `openclaw promos`

ClawHub'da yayımlanan promosyon model tekliflerini keşfedin ve talep edin. Bir
promosyonu talep etmek, sağlayıcıyı (gerektiğinde kimlik doğrulama ve Plugin) yapılandırır ve
promosyonun modellerini kaydeder; ilk katılımı yeniden çalıştırmaz ve siz istemediğiniz sürece
varsayılan modelinizi değiştirmez.

İlgili:

- Varsayılan model ve geri dönüşler: [Modeller](/tr/cli/models)
- Sağlayıcı kimlik doğrulama kurulumu: [Başlarken](/tr/start/getting-started)

## Komutlar

```bash
openclaw promos list
openclaw promos claim <slug>
openclaw promos claim <slug> --api-key <key> --set-default
```

## `openclaw promos list`

Şu anda etkin olan promosyonları; modelleri, önerilen
varsayılanı, kalan süreyi ve tam talep komutunu içerecek şekilde listeler. `--json` ham
yükü yazdırır.

## `openclaw promos claim <slug>`

Etkin bir promosyonu talep eder:

1. Promosyonu ClawHub'dan getirir ve geçerlilik aralığında olduğunu doğrular.
2. Promosyonun sağlayıcısını, kimlik doğrulama seçimini ve belirtilen Plugin paketlerini
   yüklü OpenClaw sürümünüze göre doğrular. Bilinmeyen kimlikler veya paket uyuşmazlıkları
   reddedilir; bir promosyon, CLI'ın nasıl yapacağını zaten bilmediği hiçbir şeyi
   çalıştırmasını sağlayamaz.
3. Mevcut sağlayıcı kimlik bilgilerinizi varsa yeniden kullanır. Aksi takdirde
   sağlayıcının normal kimlik doğrulama akışını yürütür (önce ücretsiz anahtar için promosyonun kayıt URL'sini
   yazdırır). `--api-key <key>`, `openclaw onboard` etkileşimsiz bayraklarıyla eşleşerek API anahtarı kimlik doğrulamasını
   istemler olmadan tamamlar; anahtarı komut satırından uzak tutmak için bunun yerine
   sağlayıcının ortam değişkenini dışa aktarın
   (örneğin `OPENROUTER_API_KEY`) — mevcut ortam kimlik bilgileri
   otomatik olarak algılanır ve hiçbir bayrak gerekmez.
4. Promosyonun modellerini diğer adlarıyla birlikte kaydeder. Mevcut diğer adların
   üzerine asla yazılmaz.
5. Promosyonun önerilen modelini varsayılanınız olarak ayarlamayı teklif eder —
   `--set-default` soruyu atlar; aksi takdirde varsayılanlarınızla ilgili hiçbir şey
   değişmez.

Promosyonun geçerlilik aralığı sona erdiğinde sağlayıcı, ücretsiz modelleri sunmayı durdurur;
yapılandırmanıza ve kimlik bilgilerinize dokunulmaz. İstediğiniz zaman
`openclaw models set <model>` ile geri dönün.

## `models list` içinde pasif keşif

`openclaw models list`, siz doğrudan ClawHub'a sormadan da promosyonları
gösterir:

- Modellerini yapılandırmadığınız etkin teklifler, tablonun altında
  "Promosyon aracılığıyla kullanılabilir" grubunda ve her biri kendi talep
  komutuyla görünür.
- `promos claim` aracılığıyla kaydettiğiniz modeller, bir `promo` etiketi taşır; bu etiket
  teklifin geçerlilik aralığı sona erdiğinde `promo ended` olarak değişir.
- Yeni bir teklif ilk kez görüldüğünde, tek seferlik bir bildirim
  `openclaw promos list` konumuna yönlendirir. Daha önce listelediğiniz veya talep ettiğiniz teklifler
  bir daha asla duyurulmaz.

Bu işlem, ClawHub'ın barındırılan promosyon akışının yerel olarak önbelleğe alınmış bir kopyasını okur
(normalde koşullu bir istekle günde bir kez veya önbelleğe alınmış
anlık görüntünün süresi dolduğunda daha erken yenilenir; yenileme hataları sessizce atlanır). Eski bir
yenileme en fazla 2.5 saniye bekler ve listelemeyi asla bozmaz. `--json` ve
`--plain` çıktıları makine kullanımına uygun biçimde temiz kalır: promosyon bölümleri veya bildirimleri içermez.
Talep işlemi her zaman canlı ClawHub API'sine göre yeniden doğrulanır; bu nedenle erken geri çekilen
bir teklif, önbelleğe alınmış kopyada hâlâ görünse bile reddedilir.
