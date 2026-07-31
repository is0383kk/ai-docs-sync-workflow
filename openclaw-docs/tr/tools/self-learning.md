---
read_when:
    - OpenClaw'un tamamlanmış konuşmalardan yeniden kullanılabilir prosedürler öğrenmesini istiyorsunuz
    - Otonom beceri önerilerini etkinleştirip etkinleştirmemeye karar veriyorsunuz
    - Kendi kendine öğrenmenin güvenliğini, maliyetini, uygunluk koşullarını veya sorun gidermeyi anlamanız gerekiyor
sidebarTitle: Self-learning
summary: OpenClaw'un düzeltmelerden ve tamamlanmış kapsamlı çalışmalardan yeniden kullanılabilir beceriler önermesini sağlayın
title: Kendi kendine öğrenme
x-i18n:
    generated_at: "2026-07-27T00:21:44Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: b10618c1a64441bdf0ba58f03e02972bdf2b1d59643a78358910594f8139ccb8
    source_path: tools/self-learning.md
    workflow: 16
---

Kendi kendine öğrenme, OpenClaw'un konuşmalardaki yararlı kanıtları bekleyen
[Skill Workshop](/tr/tools/skill-workshop) önerilerine dönüştürmesini sağlar. Model
ağırlıklarını eğitmez, etkin Skills'ı düzenlemez veya agent davranışını sessizce değiştirmez. Öğrenilen her
prosedür, bir operatör inceleyip uygulayana kadar beklemede kalır.

Kendi kendine öğrenme **varsayılan olarak devre dışıdır**. Yalnızca ek bir
arka plan model çalıştırması ve transkript incelemesi çalışma alanınız için uygunsa etkinleştirin.

## Kendi kendine öğrenmeyi etkinleştirme

Control UI'da **Plugins → Workshop** bölümünü açın ve **Kendi kendine öğrenme** seçeneğini etkinleştirin. Değişiklik
hemen yürürlüğe girer; başka bir yapılandırma yazıcısı dosyayı güncellediğinde
Control UI yapılandırma anlık görüntüsünü yeniler ve sayfayı veya Gateway'i yeniden yüklemeden
geçişi yeniden dener.

CLI'ı kullanın:

```bash
openclaw config set skills.workshop.autonomous.enabled true --strict-json
```

Veya `~/.openclaw/openclaw.json` öğesini düzenleyin:

```json5
{
  skills: {
    workshop: {
      autonomous: {
        enabled: true,
      },
    },
  },
}
```

Şununla yeniden devre dışı bırakın:

```bash
openclaw config set skills.workshop.autonomous.enabled false --strict-json
```

Kullanıcı tarafından istenen skill oluşturma, `/learn` ve manuel Skill Workshop işlemleri,
kendi kendine öğrenme devre dışıyken çalışmaya devam eder.

## Geçmiş oturumları manuel olarak inceleme

Manuel geçmiş incelemesi, otonom yakalamaya karşı daha temkinli bir alternatiftir.
Control UI'da **Plugins → Workshop** bölümünü açın ve **Skill fikirleri bul** seçeneğini belirleyin.
Bu, `skills.workshop.autonomous.enabled` öğesini değiştirmez.

Her tarama:

- incelenmemiş en yeni oturumlarla başlar ve geriye doğru ilerler;
- en az altı model turu içeren en fazla 20 kapsamlı oturumu inceler;
- cron, heartbeat, hook, alt agent, ACP, Plugin'e ait ve dahili inceleme
  oturumlarını atlar;
- tanınan gizli bilgileri sansürler ve transkript paketini seçilen agent'ın yapılandırılmış modeline
  göndermeden önce sınırlar;
- otonom deneyim incelemesiyle aynı yüksek ölçütü kullanır; ve
- en fazla üç bekleyen öneri oluşturabilir veya düzeltebilir, hiçbir zaman etkin Skills oluşturmaz.

Workshop, toplam oturum sayısını, tarih kapsamını ve bulunan fikirleri bildirir.
Bir sonraki eski pencere için **Daha önceki çalışmaları tara** seçeneğini belirleyin. İmleç,
uygun geçmişin başlangıcına ulaştığında eylem **Yeni çalışmaları tara** olarak değişir.
OpenClaw, paylaşılan durum veritabanında yalnızca imleç ve kapsam meta verilerini kalıcı hâle getirir;
ikinci bir transkript arşivi oluşturmaz.

Oturumlar yalnızca OpenClaw sahipliklerini kanıtlayabildiğinde ve
harici hook içeriğini hariç tutabildiğinde taranır. Yükseltmeden sonra, yükseltme öncesindeki mevcut transkript
yerel olarak sınıflandırılabilir ancak çalıştırma başına kaynak bilgisi bulunmayan, döndürülmüş yükseltme öncesi transkriptler
atlanır. Yeni transkriptler bu kaynak bilgisini döndürme sonrasında korur.

Manuel taramalar yine model sağlayıcısı maliyetine yol açar ve uygun konuşma
içeriğini yapılandırılmış sağlayıcıya gönderir. Bunları yalnızca söz konusu inceleme
çalışma alanının gizlilik ve veri işleme gereksinimleriyle uyumlu olduğunda kullanın.

## OpenClaw neleri öğrenebilir?

Kendi kendine öğrenmenin iki temkinli yolu vardır:

1. **Doğrudan talimatlar ve düzeltmeler.** OpenClaw, “bundan sonra,” “bir dahaki sefere”
   gibi kalıcı ifadeleri ve başarısız bir yaklaşıma yönelik düzeltmeleri algılar.
   Kendi kendine öğrenme etkinleştirildiğinde bu sinyalleri başka bir istem beklemeden
   bekleyen önerilere dönüştürebilir. Bu belirlenimci yol, ilişkili talimatları
   en fazla üç öneride gruplayabilir, yazılabilir bir çalışma alanı skill'ini hedefleyebilir
   veya kendi ilişkili bekleyen önerisini düzeltebilir. Ayrıca tamamlanmayı değerlendirmek yerine
   kullanıcının talimatlarını yakaladığı için başarısız turlardan sonra da çalışır.
2. **Deneyim incelemesi.** Başarılı ve kapsamlı bir ön plan turunun ardından
   OpenClaw, tamamlanan çalışmayı yeniden kullanılabilir bir kurtarma tekniği veya
   gelecekteki en az iki model ya da araç gidiş gelişini ortadan kaldıracak
   kararlı bir prosedür açısından inceleyebilir.

İyi adaylar şunlardır:

- tekrarlanan araç veya model hatalarından sonra güvenilir bir kurtarma;
- yinelenen bir hatayı önleyen, açıkça anlaşılmayan bir sıralama kısıtlaması;
- tekrarlanan keşif gerektiren kararlı, çok adımlı bir iş akışı; veya
- gelecekteki birden fazla çağrıyı önleyecek yeniden kullanılabilir bir ön kontrol.

İnceleyici; rutin başarılı çalışmalar, tek seferlik istekler,
kişisel bilgiler, basit tercihler, geçici ortam hataları, genel
öneriler, desteklenmeyen olumsuz iddialar ve gizli bilgiler için öneride bulunmamalıdır.

## Deneyim incelemesinin çalıştığı zaman

Deneyim incelemesi bilinçli olarak geciktirilmiş ve sınırlandırılmıştır:

- Ön plan turu başarıyla tamamlanmalıdır.
- Geçerli tur en az on model yinelemesi içermelidir.
- Cron, heartbeat, bellek, taşma, hook, alt agent ve inceleme oturumları
  hariç tutulur.
- Ön plan çalıştırması bir sağlayıcıyı ve modeli çözümlemiş olmalı ve gerçekten
  `skill_workshop` erişimine sahip olmuş olmalıdır.
- OpenClaw tamamlamadan sonra 30 saniye bekler. Aynı oturumdaki daha sonraki bir ön plan tamamlaması
  bu sessiz süreyi yeniden başlatır.
- Herhangi bir agent veya yanıt çalıştırması hâlâ etkinse inceleme 30 saniye daha bekler.
- Aynı anda yalnızca bir deneyim incelemesi çalışır.
- Gecikmeli inceleme, işleme özgü Gateway çalışmasıdır. Gateway,
  boşta kalma penceresi boyunca çalışmaya devam etmelidir; tek seferlik yerel ve CLI destekli çalışma zamanları,
  bunu planlamak için yeterli yörünge ve araç kullanılabilirliği bağlamını korumaz.

Ön plan yanıtı öğrenme nedeniyle hiçbir zaman geciktirilmez. Başarısız veya uygun olmayan
bir tur deneyim incelemesini başlatmaz ancak doğrudan kullanıcı düzeltmeleri,
otonomi devre dışıyken yine de öneri olarak sunulabilir.

## İnceleyicinin aldığı içerik

Arka plan inceleyicisi yalnızca en son kullanıcı mesajından başlayarak geçerli turu
alır. İşlenen yörünge 60,000 karakterle sınırlandırılır;
gerektiğinde OpenClaw ilk mesajı ve en yeni kanıtları korur ve
atlanmış orta kısmı işaretler.

İnceleyici, çözümlenen sağlayıcıyı ve modeli yeniden kullanır. Bu kimlik kullanılabilir olduğunda ön plan
kimlik doğrulama profilini yeniden kullanır ve model geri dönüşlerini devre dışı bırakır. Bu nedenle
inceleme, yapılandırılmış sağlayıcıda ek bir model çalıştırması başlatır.
Bu çalıştırma, bir öneriyi incelerken veya taslak hâline getirirken birden fazla sağlayıcı isteği yapabilir.
Sağlayıcı fiyatlandırması ve veri işleme koşulları, ön plan turunda olduğu gibi geçerlidir.

OpenClaw başlamadan önce geçerli çalışma zamanı yapılandırmasını yeniden yükler ve
özgün konuşmanın etkin sandbox ve araç politikasını yeniden denetler. Çalıştırma
sandbox içindeyse, politika artık `skill_workshop` öğesine izin vermiyorsa veya gerekli çalışma zamanı bilgileri
eksikse inceleme güvenli biçimde başarısız olur ve hiçbir şey oluşturmaz.

<Warning>
  Kendi kendine öğrenmenin etkinleştirilmesi, geçerli turdaki araç
  girdileri ve sonuçları dâhil olmak üzere uygun konuşma içeriğinin ek bir inceleme için seçilen model
  sağlayıcısına gönderilmesine izin verir. Bu incelemenin
  veri işleme gereksinimlerini ihlal edeceği bir çalışma alanında etkinleştirmeyin.
</Warning>

## Öneri güvenliği

İnceleyici, bilinçli olarak daraltılmış bir araç
yüzeyine sahip yalıtılmış bir oturumda çalışır:

- Yalnızca Workshop önerilerini listeleyebilir veya inceleyebilir ve bekleyen bir
  öneri oluşturabilir ya da düzeltebilir.
- Etkin bir skill'i güncelleyemez, bir öneriyi uygulayamaz, reddedemez, karantinaya alamaz,
  mesaj gönderemez veya genel agent araçlarını kullanamaz.
- Model yeniden denemeleri arasında tek bir değişiklik bütçesi paylaşılır; bu nedenle bir inceleme
  en fazla bir öneri oluşturabilir veya düzeltebilir.
- İncelenen yörünge, arka plan agent'ına yönelik talimatlar olarak değil,
  güvenilmeyen kanıt olarak değerlendirilir.
- Skill Workshop, öneri içeriğini tarar ve öneri durumu yazılmadan önce
  tanınan değişmez kimlik bilgilerini reddeder.

`maxPending`, `maxSkillBytes`,
destek dosyası kısıtlamaları, tarayıcı denetimleri ve yalnızca çalışma alanına yazma dâhil olmak üzere normal Workshop sınırları geçerliliğini korur.
`approvalPolicy: "auto"` ayarı, arka plan inceleyicisine
yaşam döngüsü eylemlerine erişim vermez.

## Öğrenilen önerileri inceleme

Kendi kendine öğrenme, manuel Workshop kullanımıyla aynı bekleyen önerileri üretir.
Uygulamadan önce bunları inceleyin:

```bash
openclaw skills workshop list
openclaw skills workshop inspect <proposal-id>
openclaw skills workshop apply <proposal-id>
```

Yararlı ancak hazır olmayan önerileri düzeltin, reddedin veya karantinaya alın:

```bash
openclaw skills workshop revise <proposal-id> --proposal ./PROPOSAL.md
openclaw skills workshop reject <proposal-id> --reason "Çok spesifik"
openclaw skills workshop quarantine <proposal-id> --reason "Güvenlik incelemesi gerekiyor"
```

Etkin bir `SKILL.md` yazan tek işlem uygulamadır. Eksiksiz yaşam döngüsü ve depolama
modeli için [Skill Workshop](/tr/tools/skill-workshop) bölümüne bakın.

## Yapılandırma

| Ayar                                      | Varsayılan | Kendi kendine öğrenme etkisi                                                                                                      |
| ------------------------------------------ | -------- | --------------------------------------------------------------------------------------------------------------------------------- |
| `skills.workshop.autonomous.enabled`       | `false`  | Doğrudan düzeltme yakalamayı ve gecikmeli deneyim incelemesini etkinleştirir.                                                      |
| `skills.workshop.approvalPolicy`           | `"auto"` | Agent tarafından başlatılan normal yaşam döngüsü eylemlerinin onay istemlerini denetler; arka plan inceleyicisinin izinlerini genişletmez. |
| `skills.workshop.maxPending`               | `50`     | Çalışma alanı başına bekleyen ve karantinaya alınmış önerileri sınırlar.                                                           |
| `skills.workshop.maxSkillBytes`            | `40000`  | Öneri gövdesi boyutunu bayt cinsinden sınırlar.                                                                                    |
| `skills.workshop.allowSymlinkTargetWrites` | `false`  | Yalnızca uygulama davranışını etkiler; kendi kendine öğrenmenin kendisi etkin skill hedeflerine değil, öneri durumuna yazar.         |

Eksiksiz şema, aralıklar ve ilişkili skill ayarları için
[Skills yapılandırması](/tr/tools/skills-config#workshop-skills-workshop) bölümüne bakın.

## Sorun giderme

### Uzun bir turdan sonra öneri görünmüyor

Aşağıdakilerin tümünü denetleyin:

1. `skills.workshop.autonomous.enabled`, etkin Gateway yapılandırmasında `true` değerindedir.
2. Tur başarılı olmuş ve en son kullanıcı mesajından sonra en az on model yinelemesi
   içermiştir.
3. Konuşma; zamanlanmış, bellek, hook veya alt agent çalıştırması değil, normal bir ön plan çalıştırmasıydı.
4. Özgün çalıştırma `skill_workshop` erişimine sahipti ve sandbox içinde değildi.
5. Sistem gecikmeli inceleme için yeterince uzun süre boşta kaldı.
6. Uzun süre çalışan Gateway işlemi boşta kalma penceresi boyunca etkin kaldı;
   tek seferlik yerel bir komut gecikmeli incelemeyi beklemez.

Uygun bir inceleme yine de hiçbir öneri üretmeyebilir. Kanıtlar
yeniden kullanılabilir prosedür eşiğini aşmadığında öneride bulunulmaması beklenen sonuçtur.

### Doctor, Workshop aracının gizli olduğunu bildiriyor

Kendi kendine öğrenme etkinleştirildiğinde `openclaw doctor`, varsayılan
agent'ın etkin araç politikasının `skill_workshop` öğesine izin verip vermediğini denetler. Bildirilen
`tools.allow` veya `tools.alsoAllow` değişikliğini uygulayın ya da kendi kendine öğrenmeyi devre dışı bırakın.

### Çok fazla düşük değerli öneri görünüyor

Kendi kendine öğrenmeyi devre dışı bırakın ve `/learn` veya açık Workshop isteklerini kullanmaya devam edin:

```bash
openclaw config set skills.workshop.autonomous.enabled false --strict-json
```

Özellik devre dışı bırakıldıktan sonra bekleyen öneriler incelenmeye devam edebilir. Kendi kendine
öğrenmeyi devre dışı bırakmak bunları uygulamaz, reddetmez veya silmez.

## İlgili

- öneri incelemesi, onay ve depolama için [Skill Atölyesi](/tr/tools/skill-workshop)
- elle yazılan skill'ler ve `SKILL.md` yapısı için [Skill oluşturma](/tr/tools/creating-skills)
- tüm `skills.*` ayarları için [Skills yapılandırması](/tr/tools/skills-config)
- Atölye ve küratör komutları için [Skills CLI](/tr/cli/skills)
