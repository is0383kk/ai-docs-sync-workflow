---
read_when:
    - Bir agent görevi için yalıtılmış bir dal ve çalışma kopyası istiyorsunuz
    - Workboard kartlarını worktree çalışma alanlarıyla yapılandırıyorsunuz
    - OpenClaw tarafından yönetilen bir çalışma ağacını geri yüklemeniz veya temizlemeniz gerekiyor
summary: Otomatik anlık görüntüler ve temizlik ile agent görevlerini yalıtılmış git çalışma kopyalarında çalıştırın
title: Yönetilen çalışma ağaçları
x-i18n:
    generated_at: "2026-07-26T23:37:37Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 98ed2579b7243544dbdb550c4b8a292ccd4ab494fd4a45b2404256691c831401
    source_path: concepts/managed-worktrees.md
    workflow: 16
---

Yönetilen çalışma ağaçları, kaynak deponun içine geçici dizinler yerleştirmeden bir aracı görevine kendi Git dalını ve çalışma kopyasını verir. OpenClaw bunları durum dizini altında oluşturur, paylaşılan durum veritabanına kaydeder ve kaldırmadan önce izlenen ve yok sayılmayan izlenmeyen içeriklerinin anlık görüntüsünü alır.

## Düzen ve adlar

Her çalışma ağacı şu konumda bulunur:

```text
<openclaw-state-dir>/worktrees/<repo-fingerprint>/<name>
```

Depo parmak izi, standart Git ortak dizini ve kaynak URL'si üzerinden hesaplanan SHA-256 karmasının ilk 16 onaltılık karakteridir. Sağlanan ad `[a-z0-9][a-z0-9-]{0,63}` ile eşleşmelidir. Ad belirtilmezse OpenClaw, `wt-` ve ardından sekiz rastgele onaltılık karakterden oluşan bir ad üretir.

OpenClaw, istenen temel referansta `openclaw/<name>` dalını oluşturur. Temel referans belirtilmezse `origin` öğesini getirir, mevcutsa uzak deponun varsayılan dalını kullanır ve depo çevrimdışıysa veya kullanılabilir bir uzak deposu yoksa yerel `HEAD` dalına geri döner.

## Yok sayılan dosyaları sağlama

Seçilen yok sayılan, izlenmeyen dosyaları yeni bir çalışma ağacına kopyalamak için kaynak deponun köküne `.worktreeinclude` ekleyin. Dosya, her satırda bir desen ve `#` açıklamaları olacak şekilde gitignore desen söz dizimini kullanır:

```gitignore
.env.local
fixtures/generated/**
```

Yalnızca Git tarafından hem yok sayılan hem de izlenmeyen olarak bildirilen dosyalar uygundur. İzlenen dosyalar Git aracılığıyla zaten mevcuttur ve bu adımda hiçbir zaman kopyalanmaz. OpenClaw mevcut hedef dosyaların üzerine yazmaz veya onları değiştirmez, sembolik bağlantılı dizinleri izlemez ve kopyalanan dosya kiplerini korur. Yalnızca gerçekten oluşturduğu yolları kaydeder; böylece bildirim dosyasında daha sonra yapılan düzenlemeler, bu dosyaların temizleme korumasından çıkmasına neden olamaz.

## Depo kurulumunu çalıştırma

Kaynak depoda `.openclaw/worktree-setup.sh` mevcut ve çalıştırılabilir durumdaysa OpenClaw, yeni çalışma ağacını geçerli dizin olarak kullanarak bunu çalıştırır. Betik şunları alır:

```text
OPENCLAW_SOURCE_TREE_PATH=<source checkout>
OPENCLAW_WORKTREE_PATH=<managed worktree>
```

Sıfır olmayan bir çıkış, oluşturmayı iptal eder ve yeni çalışma ağacıyla dalı kaldırır. Bu, depoya özgü bir sözleşmedir; bunun için bir OpenClaw yapılandırma anahtarı yoktur.

## Oturum çalışma ağaçları

Git destekli bir klasörden çalışma ağacı oturumuyla yalıtılmış bir sohbet başlatın: Control UI'daki Yeni oturum sayfasında bir Gateway kaynak klasörü seçmek için **Konum** seçicisini kullanın, ardından **Çalışma ağacı** seçeneğini belirleyin (isteğe bağlı bir temel dal ve çalışma ağacı adıyla). Bu seçenek yalnızca Gateway, seçilen klasörün bir Git çalışma kopyası olduğunu doğruladıktan sonra görünür; sıradan klasörler doğrudan çalışır ve Git yalıtımı denetimi göstermez. Etkin araç çalışma alanı Git destekliyse iOS aynı seçeneği Sohbet eylemlerinden, Android ise Yeni Sohbet'in yanında sunar.

Kodlama araçları, geçerli görevin dışında doğrulanmış takip çalışmaları tespit ettiklerinde `spawn_task` çağrısını da yapabilir. Control UI hiçbir şey başlatmadan bir öneri çipi gösterirken Gateway destekli bir TUI aynı eylemleri içeren etkileşimli bir istem gösterir. **Çalışma ağacında başlat** seçeneğinin belirlenmesi, önerilen projeden oturuma ait yeni bir çalışma ağacı oluşturur ve bağımsız istemi ilk tur olarak gönderir; önerinin kapatılması depoya dokunmaz. Öneriler ve kimlikleri geçicidir ve Gateway yeniden başlatıldığında korunmaz.

OpenClaw bu araçları yalnızca işlem yapılabilir bir Gateway kullanıcı arayüzüne sahip operatör oturumlarına sunar. Kanal oturumları ve yerel/gömülü TUI oturumları, bu yüzeyler taşınabilir ve türü belirlenmiş bir görev eylemi sözleşmesine sahip olana kadar bunları almaz.

Ortaya çıkan yönetilen çalışma ağacının sahibi oturumdur ve bu oturumdaki her araç çalıştırması onun çalışma kopyasını kullanır. Çalışma alanı bir depo alt dizini olduğunda çalışma ağacı depo köküne sabitlenir ve oturum, bunun içindeki eşleşen alt dizinden çalışır. Oturum çalışma ağacı oluşturma işlemi yöntemin `operator.write` kapsamını kullanır, ancak depo kodunu çalıştırdıkları için depo çalışma kopyası kancaları ve `.openclaw/worktree-setup.sh` adımı yalnızca `operator.admin` çağıranları için çalışır; `.worktreeinclude` sağlama işlemi yine de her çağırana uygulanır. Oturumun silinmesi, çalışma ağacını yalnızca kayıpsız biçimde yapılabiliyorsa kaldırır. Değişiklik içeren çalışma ağaçları veya gönderilmemiş commit'lere sahip dallar kullanılabilir durumda kalır; saatlik temizleme, son oturum etkinliğini çalışma ağacı etkinliği sayarak oturum çalışma ağaçlarının anlık görüntüsünü 7 gün boyunca boşta kaldıktan sonra alır. Kaldırılan çalışma ağaçları, aşağıda açıklandığı gibi anlık görüntülerinden geri yüklenebilir durumda kalır.

`sessions.create`, doğrudan başka bir Gateway klasöründe çalışmak, kaynak çalışma kopyasını `worktree: true` ile birlikte seçmek veya eşleştirilmiş bir Node'un çalışma dizinini ayarlamak için mutlak bir `cwd` içerebilir. Açıkça belirtilen her ana makine yolu `operator.admin` gerektirir; sıradan çalışma ağacı sohbeti oluşturma işlemi `operator.write` olarak kalır ve yapılandırılmış çalışma alanına sabitlenmeye devam eder.

`sessions.create`, temel referansı ve çalışma ağacı adını seçmek için `worktree: true` ile birlikte `worktreeBaseRef` ve `worktreeName` değerlerini de kabul eder (dal `openclaw/<name>` olur); her ikisi de `operator.write` düzeyinde kalır. Oluşturulan çalışma ağacı, oluşturma sonucunda döndürülür ve oturum satırında `worktree: { id, branch, repoRoot }` olarak kalıcılaştırılır; böylece oturum listeleri çalışma kopyasını ve dalı gösterebilir. Bir oturum silinirken değişiklik içeren korunan bir çalışma kopyası sessizce geride bırakılmak yerine `worktreePreserved` olarak bildirilir.

## Anlık görüntüler, temizleme ve geri yükleme

Kaldırma işlemi önce izlenen ve yok sayılmayan izlenmeyen dosyaları içeren yapay bir commit oluşturur, ardından bunu `refs/openclaw/snapshots/<id>` konumuna sabitler. Yok sayılan dosyalar hiçbir zaman depo nesne veritabanına girmez. OpenClaw yalnızca gerçekten sağladığı yok sayılan dosyaları parçalı paylaşılan durum veritabanı satırlarında saklar; `.worktreeinclude` daha sonra değişse veya kaybolsa bile kaydedilmiş yol kümesi yetkili kaynak olmaya devam eder. Geri yükleme, bu baytları değişmez anlık görüntüden okur ve tam kiplerini yeniden uygular. Kaydedilmiş bir yolun anlık görüntüsü artık güvenli biçimde alınamıyorsa otomatik temizleme canlı çalışma ağacını korur. Anlık görüntü oluşturma başarısız olursa kaldırma durur. Açıkça belirtilen zorla silme işlemi anlık görüntü olmadan devam edebilir.

OpenClaw şu temizleme kurallarını uygular:

- Çalıştırma sonunda yalnızca `git status --porcelain` boşsa ve `git log HEAD --not --remotes --oneline` gönderilmemiş commit bulmazsa çalışma ağacını kaldırır. Aksi takdirde yalnızca etkinlik kilidini serbest bırakır.
- Saatlik temizleme, kilitli olmayan ve Workboard'a veya oturuma ait çalışma ağaçlarının anlık görüntüsünü 7 günden uzun süre boşta kaldıklarında, değişiklik içerseler bile alır ve bunları kaldırır. Elle oluşturulan çalışma ağaçları hiçbir zaman otomatik olarak kaldırılmaz.
- Anlık görüntü kayıtları 30 gün boyunca geri yüklenebilir durumda kalır. Ardından temizleme, anlık görüntü referansını ve kayıt satırını siler.
- Canlı bir OpenClaw işlem kilidi ve yabancı veya tanınmayan herhangi bir Git çalışma ağacı kilidi, çalışma ağacını çöp toplamaya karşı korur.

Geri yükleme, özgün anlık görüntü öncesi commit'te `openclaw/<name>` öğesini yeniden oluşturur, ardından anlık görüntü farklılıklarını hazırlanmamış değişiklikler ve izlenmeyen dosyalar olarak yeniden meydana getirir. Bu, yapay anlık görüntü commit'inin dal geçmişine girmesini önler. Anlık görüntü referansı kaynak bilgisi olarak kayıtlı kalır.

## CLI

```bash
openclaw worktrees list [--json]
openclaw worktrees create <repo-root> [--name <name>] [--base-ref <ref>] [--json]
openclaw worktrees remove <id> [--force] [--json]
openclaw worktrees restore <id> [--json]
openclaw worktrees gc [--json]
```

Ayarlar altındaki Control UI **Çalışma ağaçları** sayfası aynı eylemlerin yanı sıra temel dal seçicisiyle oluşturma işlevi sağlar, her çalışma ağacının sahibini (elle oluşturulan, Workboard veya sohbetine yönlendiren bir bağlantıyla sahip oturum) gösterir ve kaldırma işlemi başarısız bir anlık görüntü bildirdiğinde zorla yeniden deneme seçeneği sunar.

## Gateway yöntemleri

| Yöntem               | Amaç                                                                 |
| -------------------- | ----------------------------------------------------------------------- |
| `worktrees.list`     | Etkin ve geri yüklenebilir çalışma ağacı kayıtlarını listeler.                            |
| `worktrees.branches` | Temel referans seçicileri için bir deponun yerel ve uzak dallarını listeler.    |
| `worktrees.create`   | Adlandırılmış yönetilen bir çalışma ağacı oluşturur veya yeniden kullanır.                               |
| `worktrees.remove`   | Bir çalışma ağacının anlık görüntüsünü alır ve onu kaldırır. Zorla kaldırma işlemleri `snapshotError` bildirir. |
| `worktrees.restore`  | Kaldırılmış bir çalışma ağacını anlık görüntüsünden geri yükler.                           |
| `worktrees.gc`       | Boşta kalma, sahipsiz öğe ve saklama temizliğini şimdi çalıştırır.                            |

`worktrees.list`, `operator.read` gerektirir ve değişiklik yapan yöntemler `operator.admin` gerektirir. `worktrees.branches`, yapılandırılmış araç çalışma alanları için `operator.write` gerektirirken diğer tüm ana makine yolları `operator.admin` gerektirir (`sessions.create` cwd sınırıyla eşleşir). Yalnızca mevcut referansları okur ve hiçbir zaman getirme işlemi yapmaz; yalnızca uzakta bulunan dallar uzak depo niteleyicisiyle (`origin/feature-a`) döner, böylece döndürülen her ad temel referans olarak çözümlenir. Yeni Oturum ayrıca bu yöntemden türü belirlenmiş bir depo durumu isteyebilir; düz bir dizin veya kullanılamayan çalışma kopyası, kullanıcı arayüzünü bir hata dizesinden Git özelliğini çıkarsamaya zorlamak yerine hiç dal döndürmez.

## Workboard çalışma alanları

Paketle birlikte sunulan [Workboard Plugin](/tr/plugins/workboard), bir kart çalışma alanını yönetilen çalışma ağacı olarak oluşturabilir:

```json
{
  "kind": "worktree",
  "path": "/absolute/path/to/source-checkout",
  "branch": "main"
}
```

`path`, kaynak Git çalışma kopyasını tanımlar. `branch` isteğe bağlıdır ve temel referans olur. Tam ana makine erişimine sahip bir çağıran için Workboard, `wb-<card-id>` öğesini oluşturur veya yeniden kullanır, alt aracı yönetilen çalışma kopyasını çalışma dizini olarak kullanarak çalıştırır ve çözümlenen yolu ve dalı karta geri yazar. Gateway istemcileri, tam ana makine üzerinde oluşturma için `operator.admin` gerektirir. Çalıştırma sonunda Workboard, çalışma kopyasını yalnızca kayıpsız olduğu kesin biçimde kanıtlanabiliyorsa kaldırır; değişiklik içeren çalışmalar veya gönderilmemiş commit'ler kullanılabilir durumda kalır.

Çalışma alanına bağlı bir çağıran için `path` ve depo kökü, hedef araç çalışma alanıyla tam olarak eşleşmelidir. Workboard daha sonra doğrudan bu dizinde çalışır ve ana makine üzerinde yönetilen bir çalışma ağacı oluşturmak yerine bir dizin çalışma alanı kaydeder. Hedef, aynı çalışma alanı için yazılabilir ve paylaşılmayan bir Docker korumalı alanı kullanmalı, canlı konteyner karması istenen bağlamalar ve politikayla eşleşmeli ve yükseltilmiş yürütme, ana makine denetimi, ana makine genelindeki oturumlar, kalıcı ana makine/Node yürütmesi veya sınıflandırılmamış Plugin ve MCP araçları sunmamalıdır. Hedef politikası veya canlı konteyner daha geniş kapsamlıysa görev gönderimi kartı sahipsiz bırakır ve uyumsuz durumu bildirir.
