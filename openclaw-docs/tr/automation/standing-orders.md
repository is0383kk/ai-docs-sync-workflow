---
read_when:
    - Görev başına istem gerektirmeden çalışan otonom aracı iş akışlarını ayarlama
    - Aracının bağımsız olarak yapabilecekleriyle insan onayı gerektirenlerin tanımlanması
    - Net sınırlar ve eskalasyon kurallarıyla çok programlı ajanları yapılandırma
summary: Otonom aracı programları için kalıcı işletim yetkisini tanımlayın
title: Daimî talimatlar
x-i18n:
    generated_at: "2026-07-26T23:48:30Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 9e7ad622efe734facc9dc3716f5ee7f57ed3923499db78730bda234a5c62ad80
    source_path: automation/standing-orders.md
    workflow: 16
---

Daimi talimatlar, tanımlanmış programlar için agent'ınıza **kalıcı işletim yetkisi** verir. Her görev için agent'ı yönlendirmek yerine kapsamı, tetikleyicileri ve eskalasyon kuralları açıkça belirlenmiş programlar tanımlarsınız; agent da bu sınırlar içinde özerk olarak yürütür: "Haftalık rapor senin sorumluluğunda. Her cuma derle, gönder ve yalnızca bir şeyler yanlış görünüyorsa eskale et."

## Neden daimi talimatlar

**Daimi talimatlar olmadan:** her görev için agent'ı yönlendirmeniz gerekir, rutin işler unutulur veya gecikir ve darboğaz siz olursunuz.

**Daimi talimatlarla:** agent, tanımlanmış sınırlar içinde özerk olarak yürütür, rutin işler zamanında gerçekleşir ve yalnızca istisnalar ile onaylar için sürece dahil olursunuz.

## Nasıl çalışırlar

Daimi talimatlar, [agent çalışma alanı](/tr/concepts/agent-workspace) dosyalarınızda tanımlanır. Önerilen yaklaşım, agent'ın bunları her zaman bağlamında bulundurması için talimatları doğrudan `AGENTS.md` içine (her oturumda otomatik olarak eklenir) dahil etmektir. Daha büyük yapılandırmalar için bunları `standing-orders.md` gibi özel bir dosyaya da yerleştirip `AGENTS.md` içinden bu dosyaya başvurabilirsiniz.

Her program şunları belirtir:

1. **Kapsam** - agent'ın neleri yapmaya yetkili olduğu
2. **Tetikleyiciler** - ne zaman yürütüleceği (zamanlama, olay veya koşul)
3. **Onay eşikleri** - harekete geçmeden önce nelerin insan onayı gerektirdiği
4. **Eskalasyon kuralları** - ne zaman durup yardım isteneceği

Agent, bu talimatları çalışma alanı başlangıç dosyaları aracılığıyla her oturumda yükler (otomatik eklenen dosyaların tam listesi için [Agent Çalışma Alanı](/tr/concepts/agent-workspace) bölümüne bakın) ve zamana dayalı yaptırım için [Cron işleri](/tr/automation/cron-jobs) ile birlikte bunlara göre yürütür.

<Tip>
Her oturumda yüklenmelerini garanti etmek için daimi talimatları `AGENTS.md` içine koyun. Çalışma alanı başlangıcı `AGENTS.md`, `SOUL.md`, `TOOLS.md`, `IDENTITY.md`, `USER.md`, `HEARTBEAT.md`, `BOOTSTRAP.md` ve `MEMORY.md` dosyalarını otomatik olarak ekler; ancak alt dizinlerdeki herhangi bir dosyayı eklemez.
</Tip>

## Bir daimi talimatın anatomisi

```markdown
## Program: Haftalık Durum Raporu

**Yetki:** Verileri derle, rapor oluştur, paydaşlara ilet
**Tetikleyici:** Her cuma saat 16.00'da (Cron işi aracılığıyla uygulanır)
**Onay eşiği:** Standart raporlar için yok. Anormallikleri insan incelemesi için işaretle.
**Eskalasyon:** Veri kaynağı kullanılamıyorsa veya metrikler olağan dışı görünüyorsa (normdan >2σ sapma)

### Yürütme adımları

1. Yapılandırılmış kaynaklardan metrikleri al
2. Önceki hafta ve hedeflerle karşılaştır
3. Reports/weekly/YYYY-MM-DD.md içinde rapor oluştur
4. Özeti yapılandırılmış kanal üzerinden ilet
5. Tamamlanma kaydını Agent/Logs/ içine yaz

### YAPILMAMASI gerekenler

- Raporları harici taraflara gönderme
- Kaynak verileri değiştirme
- Metrikler kötü görünüyorsa teslimatı atlama; doğru şekilde raporla
```

## Daimi talimatlar ve Cron işleri

Daimi talimatlar, agent'ın **neyi** yapmaya yetkili olduğunu tanımlar. [Cron işleri](/tr/automation/cron-jobs), bunun **ne zaman** gerçekleşeceğini tanımlar. Birlikte çalışırlar:

```text
Daimi Talimat: "Günlük gelen kutusu önceliklendirmesi senin sorumluluğunda"
    ↓
Cron İşi (her gün saat 08.00): "Daimi talimatlara göre gelen kutusu önceliklendirmesini yürüt"
    ↓
Agent: Daimi talimatları okur → adımları yürütür → sonuçları raporlar
```

Cron işi istemi, daimi talimatı yinelemek yerine ona başvurmalıdır:

```bash
openclaw cron add \
  --name daily-inbox-triage \
  --cron "0 8 * * 1-5" \
  --tz America/New_York \
  --timeout-seconds 300 \
  --announce \
  --channel imessage \
  --to "+1XXXXXXXXXX" \
  --message "Daimi talimatlara göre günlük gelen kutusu önceliklendirmesini yürüt. Yeni uyarılar için postayı kontrol et. Her öğeyi ayrıştır, sınıflandır ve kalıcı olarak kaydet. Özeti sahibine bildir. Bilinmeyenleri eskale et."
```

## Örnekler

### Örnek 1: içerik ve sosyal medya (haftalık döngü)

```markdown
## Program: İçerik ve Sosyal Medya

**Yetki:** İçerik taslağı hazırla, gönderileri zamanla, etkileşim raporlarını derle
**Onay eşiği:** İlk 30 gün boyunca tüm gönderiler sahibin incelemesini gerektirir, ardından daimi onay geçerlidir
**Tetikleyici:** Haftalık döngü (pazartesi inceleme → hafta ortasında taslaklar → cuma özeti)

### Haftalık döngü

- **Pazartesi:** Platform metriklerini ve kitle etkileşimini incele
- **Salı-Perşembe:** Sosyal medya gönderilerinin taslaklarını hazırla, blog içeriği oluştur
- **Cuma:** Haftalık pazarlama özetini derle → sahibine ilet

### İçerik kuralları

- Üslup markayla uyumlu olmalıdır (SOUL.md veya marka üslubu kılavuzuna bakın)
- Kamuya yönelik içeriklerde kendini asla yapay zekâ olarak tanımlama
- Mevcut olduğunda metrikleri dahil et
- Kendini tanıtmaya değil, kitleye sağlanan değere odaklan
```

### Örnek 2: finans işlemleri (olayla tetiklenen)

```markdown
## Program: Finansal İşleme

**Yetki:** İşlem verilerini işle, raporlar oluştur, özetler gönder
**Onay eşiği:** Analiz için yok. Öneriler sahibin onayını gerektirir.
**Tetikleyici:** Yeni veri dosyası algılandığında VEYA zamanlanmış aylık döngüde

### Yeni veri geldiğinde

1. Belirlenen girdi dizinindeki yeni dosyayı algıla
2. Tüm işlemleri ayrıştır ve sınıflandır
3. Bütçe hedefleriyle karşılaştır
4. Şunları işaretle: olağan dışı kalemler, eşik aşımları, yeni yinelenen ücretler
5. Belirlenen çıktı dizininde rapor oluştur
6. Özeti yapılandırılmış kanal üzerinden sahibine ilet

### Eskalasyon kuralları

- Tek kalem > $500: anında uyar
- Kategori bütçeyi %20 aşarsa: raporda işaretle
- Tanınmayan işlem: sınıflandırma için sahibine sor
- 2 yeniden denemeden sonra işleme başarısız olursa: hatayı bildir, tahminde bulunma
```

### Örnek 3: izleme ve uyarılar (sürekli)

```markdown
## Program: Sistem İzleme

**Yetki:** Sistem durumunu kontrol et, hizmetleri yeniden başlat, uyarılar gönder
**Onay eşiği:** Hizmetleri otomatik olarak yeniden başlat. Yeniden başlatma iki kez başarısız olursa eskale et.
**Tetikleyici:** Her Heartbeat döngüsünde

### Kontroller

- Hizmet durumu uç noktaları yanıt veriyor
- Disk alanı eşiğin üzerinde
- Bekleyen görevler bayatlamamış (>24 saat)
- Teslimat kanalları çalışıyor

### Yanıt matrisi

| Koşul            | Eylem                    | Eskale edilsin mi?                       |
| ---------------- | ------------------------ | ---------------------------------------- |
| Hizmet çalışmıyor | Otomatik olarak yeniden başlat | Yalnızca yeniden başlatma 2 kez başarısız olursa |
| Disk alanı < %10 | Sahibini uyar             | Evet                                     |
| Bayat görev > 24 sa | Sahibine hatırlat       | Hayır                                    |
| Kanal çevrimdışı | Kaydet ve sonraki döngüde yeniden dene | Çevrimdışı kalma süresi > 2 saatse       |
```

## Yürüt-doğrula-raporla kalıbı

Daimi talimatlar, katı yürütme disipliniyle birleştirildiğinde en iyi sonucu verir. Daimi talimattaki her görev şu döngüyü izlemelidir:

1. **Yürüt** - Asıl işi yap (yalnızca talimatı aldığını belirtme)
2. **Doğrula** - Sonucun doğru olduğunu teyit et (dosya mevcut, mesaj iletildi, veriler ayrıştırıldı)
3. **Raporla** - Sahibine ne yapıldığını ve neyin doğrulandığını bildir

```markdown
### Yürütme kuralları

- Her görev Yürüt-Doğrula-Raporla düzenini izler. İstisna yoktur.
- "Bunu yapacağım" demek yürütme değildir. Yap, ardından raporla.
- Doğrulama olmadan "Tamamlandı" demek kabul edilemez. Kanıtla.
- Yürütme başarısız olursa: yaklaşımı ayarlayarak bir kez yeniden dene.
- Yine başarısız olursa: tanıyla birlikte hatayı bildir. Asla sessizce başarısız olma.
- Asla süresiz yeniden deneme; en fazla 3 girişimden sonra eskale et.
```

Bu kalıp, en yaygın agent başarısızlık biçimini önler: görevi tamamlamadan yalnızca alındığını bildirmek.

## Çok programlı mimari

Birden fazla konuyu yöneten agent'lar için daimi talimatları sınırları açıkça belirlenmiş ayrı programlar olarak düzenleyin:

```markdown
## Program 1: [Alan A] (Haftalık)

...

## Program 2: [Alan B] (Aylık + İstek Üzerine)

...

## Program 3: [Alan C] (Gerektikçe)

...

## Eskalasyon Kuralları (Tüm Programlar)

- [Ortak eskalasyon ölçütleri]
- [Programların tamamında geçerli olan onay eşikleri]
```

Her programda şunlar bulunmalıdır:

- Kendi **tetikleme sıklığı** (haftalık, aylık, olay odaklı, sürekli)
- Kendi **onay eşikleri** (bazı programlar diğerlerinden daha fazla gözetim gerektirir)
- Açık **sınırlar** (agent, bir programın nerede bitip diğerinin nerede başladığını bilmelidir)

## En iyi uygulamalar

### Yapılması gerekenler

- Dar kapsamlı yetkiyle başlayın ve güven arttıkça kapsamı genişletin
- Yüksek riskli eylemler için açık onay eşikleri tanımlayın
- "YAPILMAMASI gerekenler" bölümlerini dahil edin; sınırlar da izinler kadar önemlidir
- Zamana dayalı güvenilir yürütme için Cron işleriyle birleştirin
- Daimi talimatlara uyulduğunu doğrulamak için agent günlüklerini haftalık olarak inceleyin
- İhtiyaçlarınız geliştikçe daimi talimatları güncelleyin; bunlar yaşayan belgelerdir

### Kaçınılması gerekenler

- İlk günden geniş kapsamlı yetki vermek ("en iyi olduğunu düşündüğün şeyi yap")
- Eskalasyon kurallarını atlamak; her programın bir "ne zaman durulup sorulacağı" maddesine ihtiyacı vardır
- Agent'ın sözlü talimatları hatırlayacağını varsaymak; her şeyi dosyaya koyun
- Konuları tek bir programda karıştırmak; ayrı alanlar için ayrı programlar kullanın
- Cron işleriyle uygulamayı unutmak; tetikleyicileri olmayan daimi talimatlar öneriye dönüşür

## İlgili konular

- [Otomasyon](/tr/automation): tüm otomasyon mekanizmalarına genel bakış.
- [Cron işleri](/tr/automation/cron-jobs): daimi talimatlar için zamanlama uygulaması.
- [Kancalar](/tr/automation/hooks): agent yaşam döngüsü olayları için olay odaklı betikler.
- [Webhook'lar](/tr/automation/cron-jobs#webhooks): gelen HTTP olay tetikleyicileri.
- [Agent çalışma alanı](/tr/concepts/agent-workspace): otomatik eklenen başlangıç dosyalarının tam listesi (`AGENTS.md`, `SOUL.md` vb.) dahil olmak üzere daimi talimatların bulunduğu yer.
