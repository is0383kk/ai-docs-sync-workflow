---
read_when:
    - OpenClaw'da "bağlam"ın ne anlama geldiğini anlamak istiyorsunuz
    - Modelin bir şeyi neden "bildiğini" (veya unuttuğunu) ayıklıyorsunuz
    - Bağlam yükünü azaltmak istiyorsunuz (/context, /status, /compact)
summary: 'Bağlam: modelin ne gördüğü, bunun nasıl oluşturulduğu ve nasıl inceleneceği'
title: Bağlam
x-i18n:
    generated_at: "2026-07-26T23:54:27Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 1eb3d342a601a447487640587f746cc80a133ede338a880741f53c3e01f20ed1
    source_path: concepts/context.md
    workflow: 16
---

"Bağlam", **OpenClaw'ın bir çalıştırma için modele gönderdiği her şeydir**. Modelin **bağlam penceresi** (token sınırı) ile sınırlıdır.

Yeni başlayanlar için zihinsel model:

- **Sistem istemi** (OpenClaw tarafından oluşturulur): kurallar, araçlar, Skills listesi, zaman/çalışma zamanı ve enjekte edilen çalışma alanı dosyaları.
- **Konuşma geçmişi**: bu oturumdaki iletileriniz + asistanın iletileri.
- **Araç çağrıları/sonuçları + ekler**: komut çıktısı, dosya okumaları, görüntüler/ses vb.

Bağlam, "bellek" ile _aynı şey değildir_: bellek diskte saklanıp daha sonra yeniden yüklenebilir; bağlam ise modelin geçerli penceresinin içindekilerdir.

## Hızlı başlangıç (bağlamı inceleme)

- `/status` → hızlı "pencerem ne kadar dolu?" görünümü + oturum ayarları.
- `/context list` → nelerin enjekte edildiği + yaklaşık boyutlar (dosya başına + toplamlar).
- `/context detail` → daha ayrıntılı döküm: dosya başına, araç şeması başına boyutlar, Skills girdisi başına boyutlar, sistem istemi boyutu ve sıkıştırılabilir transkript iletisi sayıları.
- `/context map` → geçerli oturumun izlenen bağlam katkılarının WinDirStat tarzı ağaç haritası görüntüsü.
- `/usage tokens` → normal yanıtlara yanıt başına kullanım alt bilgisi ekler.
- `/compact` → pencere alanını boşaltmak için eski geçmişi kompakt bir girdide özetler.

Ayrıca bkz.: [Eğik çizgi komutları](/tr/tools/slash-commands), [Token kullanımı ve maliyetleri](/tr/reference/token-use), [Compaction](/tr/concepts/compaction).

## Örnek çıktı

Değerler modele, sağlayıcıya, araç politikasına ve çalışma alanınızdakilere göre değişir.

### `/context list`

```text
🧠 Bağlam dökümü
Çalışma alanı: <workspaceDir>
Önyükleme üst sınırı/dosya: 12,000 karakter
Korumalı alan: mod=ana olmayan korumalı=false
Sistem istemi (çalıştırma): 38,412 karakter (~9,603 tok) (Proje Bağlamı 23,901 karakter (~5,976 tok))

Enjekte edilen çalışma alanı dosyaları:
- AGENTS.md: TAMAM | ham 1,742 karakter (~436 tok) | enjekte edilen 1,742 karakter (~436 tok)
- SOUL.md: TAMAM | ham 912 karakter (~228 tok) | enjekte edilen 912 karakter (~228 tok)
- TOOLS.md: KESİLDİ | ham 54,210 karakter (~13,553 tok) | enjekte edilen 20,962 karakter (~5,241 tok)
- IDENTITY.md: TAMAM | ham 211 karakter (~53 tok) | enjekte edilen 211 karakter (~53 tok)
- USER.md: TAMAM | ham 388 karakter (~97 tok) | enjekte edilen 388 karakter (~97 tok)
- HEARTBEAT.md: EKSİK | ham 0 | enjekte edilen 0
- BOOTSTRAP.md: TAMAM | ham 0 karakter (~0 tok) | enjekte edilen 0 karakter (~0 tok)

Skills listesi (sistem istemi metni): 2,184 karakter (~546 tok) (12 Skills)
Araçlar: read, edit, write, exec, process, browser, message, sessions_send, …
Araç listesi (sistem istemi metni): 1,032 karakter (~258 tok)
Araç şemaları (JSON): 31,988 karakter (~7,997 tok) (bağlama dahil edilir; metin olarak gösterilmez)
Araçlar: (yukarıdakiyle aynı)

Oturum token'ları (önbelleğe alınmış): toplam 14,250 / ctx=32,000
```

### `/context detail`

```text
🧠 Bağlam dökümü (ayrıntılı)
…
En büyük Skills (istem girdisi boyutu):
- frontend-design: 412 karakter (~103 tok)
- oracle: 401 karakter (~101 tok)
… (+10 Skills daha)

En büyük araçlar (şema boyutu):
- browser: 9,812 karakter (~2,453 tok)
- exec: 6,240 karakter (~1,560 tok)
… (+N araç daha)
```

### `/context map`

En son önbelleğe alınan çalıştırma raporu ile oturum transkriptinden oluşturulan bir görüntü gönderir. Oturumda normal bir ileti henüz çalıştırma raporu üretmediyse `/context map`, bir tahmin oluşturmak yerine kullanılamıyor iletisi döndürür. Dikdörtgen alanı, izlenen istem karakterleriyle orantılıdır:

- konuşma transkripti (kullanıcı iletileri, asistan yanıtları, araç sonuçları, Compaction özetleri) ile yalnızca modele ulaşan tur başına çalışma zamanı bağlamı ve kanca istemi eklemeleri
- enjekte edilen çalışma alanı dosyaları
- temel sistem istemi metni
- Skills istem girdileri
- araç JSON şemaları

Konuşma grubu oturum ilerledikçe büyüdüğünden harita turdan tura değişir; Compaction sonrasında bir özetler kutucuğuna küçülür.

`/context list`, `/context detail` ve `/context json`, önbelleğe alınmış bir çalıştırma raporu olmadığında da istek üzerine oluşturulan bir tahmini inceleyebilir.

## Bağlam penceresine neler dahil edilir?

Modelin aldığı her şey buna dahildir:

- Sistem istemi (tüm bölümler).
- Konuşma geçmişi.
- Araç çağrıları + araç sonuçları.
- Ekler/transkriptler (görüntüler/ses/dosyalar).
- Compaction özetleri ve budama yapıtları.
- Sağlayıcı "sarmalayıcıları" veya gizli üst bilgiler (görünmez, yine de dahil edilir).

## OpenClaw sistem istemini nasıl oluşturur?

Sistem istemi **OpenClaw'a aittir** ve her çalıştırmada yeniden oluşturulur. Şunları içerir:

- Araç listesi + kısa açıklamalar.
- Skills listesi (yalnızca meta veriler; aşağıya bakın).
- Çalışma alanı konumu.
- Zaman (UTC + yapılandırılmışsa dönüştürülmüş kullanıcı zamanı).
- Çalışma zamanı meta verileri (ana makine/işletim sistemi/model/düşünme).
- **Proje Bağlamı** altında enjekte edilen çalışma alanı önyükleme dosyaları.

Tam döküm: [Sistem İstemi](/tr/concepts/system-prompt).

## Enjekte edilen çalışma alanı dosyaları (Proje Bağlamı)

OpenClaw varsayılan olarak sabit bir çalışma alanı dosyaları kümesini (varsa) enjekte eder:

- `AGENTS.md`
- `SOUL.md`
- `TOOLS.md`
- `IDENTITY.md`
- `USER.md`
- `HEARTBEAT.md`
- `BOOTSTRAP.md` (yalnızca ilk çalıştırma)

Büyük dosyalar, dosya başına `agents.defaults.bootstrapMaxChars` kullanılarak kesilir (varsayılan `20000` karakter). OpenClaw ayrıca dosyalar genelinde toplam önyükleme enjeksiyonu için `agents.defaults.bootstrapTotalMaxChars` sınırını uygular (varsayılan `60000` karakter). `/context`, **ham ve enjekte edilen** boyutları ve kesme uygulanıp uygulanmadığını gösterir.

Kesme gerçekleştiğinde çalışma zamanı, Proje Bağlamı altında istem içi bir uyarı bloğu enjekte edebilir. Bunu `agents.defaults.bootstrapPromptTruncationWarning` ile yapılandırın (`off`, `once`, `always`; varsayılan `always`).

## Skills: enjekte edilenler ve istek üzerine yüklenenler

Sistem istemi, kompakt bir **Skills listesi** (ad + açıklama + konum) içerir. Bu listenin gerçek bir ek yükü vardır.

Skills talimatları varsayılan olarak dahil edilmez. Modelin, **yalnızca gerektiğinde** ilgili Skills'in `SKILL.md` dosyasını `read` etmesi beklenir.

## Araçların iki maliyeti vardır

Araçlar bağlamı iki şekilde etkiler:

1. Sistem istemindeki **araç listesi metni** ("Araçlar" olarak gördüğünüz bölüm).
2. **Araç şemaları** (JSON). Bunlar, araçları çağırabilmesi için modele gönderilir. Düz metin olarak görmeseniz de bağlama dahil edilirler.

`/context detail`, en büyük araç şemalarını dökümler; böylece hangilerinin baskın olduğunu görebilirsiniz.

## Komutlar, direktifler ve "satır içi kısayollar"

Eğik çizgi komutları Gateway tarafından işlenir. Birkaç farklı davranış vardır:

- **Bağımsız komutlar**: yalnızca `/...` içeren bir ileti komut olarak çalıştırılır.
- **Direktifler**: `/think`, `/fast`, `/verbose`, `/trace`, `/reasoning`, `/elevated`, `/exec`, `/model`, `/queue`, model iletiyi görmeden önce kaldırılır.
  - Yalnızca direktif içeren iletiler oturum ayarlarını kalıcı hâle getirir.
  - Normal bir iletideki satır içi direktifler, ileti başına ipuçları olarak işlev görür.
- **Satır içi kısayollar** (yalnızca izin listesindeki göndericiler): normal bir iletideki belirli `/...` token'ları hemen çalıştırılabilir (örnek: "hey /status") ve model kalan metni görmeden önce kaldırılır.

Ayrıntılar: [Eğik çizgi komutları](/tr/tools/slash-commands).

## Oturumlar, Compaction ve budama (neler kalıcıdır?)

İletiler arasında nelerin kalıcı olduğu mekanizmaya bağlıdır:

- **Normal geçmiş**, politika tarafından Compaction uygulanana/budanana kadar oturum transkriptinde kalır.
- **Compaction**, bir özeti transkripte kalıcı olarak ekler ve son iletileri olduğu gibi tutar.
- **Budama**, bağlam penceresinde alan açmak için eski araç sonuçlarını _bellekteki_ istemden çıkarır ancak oturum transkriptini yeniden yazmaz; tam geçmiş diskte incelenmeye devam edilebilir.

Belgeler: [Oturum](/tr/concepts/session), [Compaction](/tr/concepts/compaction), [Oturum budama](/tr/concepts/session-pruning).

OpenClaw varsayılan olarak birleştirme ve
Compaction için yerleşik `legacy` bağlam motorunu kullanır. `kind: "context-engine"` sağlayan bir Plugin yükleyip
`plugins.slots.contextEngine` ile seçerseniz OpenClaw bağlam
birleştirmeyi, `/compact` işlemini ve ilgili alt ajan bağlamı yaşam döngüsü kancalarını bunun yerine o
motora devreder. `ownsCompaction: false`, eski
motora otomatik olarak geri dönmez; etkin motor yine de `compact()` işlevini doğru biçimde uygulamalıdır. Tam
takılıp çıkarılabilir arayüz, yaşam döngüsü kancaları ve yapılandırma için
[Bağlam Motoru](/tr/concepts/context-engine) sayfasına bakın.

## `/context` gerçekte neyi raporlar?

`/context`, varsa en son **çalıştırma sırasında oluşturulan** sistem istemi raporunu tercih eder:

- `System prompt (run)` = son gömülü (araç kullanabilen) çalıştırmadan yakalanır ve oturum deposunda kalıcı olarak saklanır.
- `System prompt (estimate)` = çalıştırma raporu bulunmadığında (veya rapor oluşturmayan bir CLI arka ucu üzerinden çalıştırıldığında) anında hesaplanır.

Her iki durumda da boyutları ve en büyük katkıları raporlar; sistem isteminin veya araç şemalarının tamamını dökmez. Ayrıntılı modda ayrıca oturum transkriptini Compaction tarafından kullanılan aynı gerçek konuşma iletisi koşuluyla karşılaştırır; böylece yüksek istem/önbellek kullanımını sıkıştırılabilir konuşma geçmişinden ayırt etmek kolaylaşır.

## İlgili konular

<CardGroup cols={2}>
  <Card title="Bağlam motoru" href="/tr/concepts/context-engine" icon="puzzle-piece">
    Plugin'ler aracılığıyla özel bağlam enjeksiyonu.
  </Card>
  <Card title="Compaction" href="/tr/concepts/compaction" icon="compress">
    Uzun konuşmaları model penceresi içinde tutmak için özetleme.
  </Card>
  <Card title="Sistem istemi" href="/tr/concepts/system-prompt" icon="message-lines">
    Sistem isteminin nasıl oluşturulduğu ve her turda neleri enjekte ettiği.
  </Card>
  <Card title="Ajan döngüsü" href="/tr/concepts/agent-loop" icon="arrows-rotate">
    Gelen iletiden son yanıta kadar tam ajan yürütme döngüsü.
  </Card>
</CardGroup>
