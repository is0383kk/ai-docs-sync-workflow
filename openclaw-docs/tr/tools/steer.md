---
read_when:
    - Bir aracı zaten çalışırken /steer veya /tell kullanma
    - /steer ile /queue modlarının karşılaştırılması
    - Geçerli çalıştırmayı mı yoksa bir ACP oturumunu mu yönlendireceğinize karar verme
sidebarTitle: Steer
summary: Kuyruk modunu değiştirmeden etkin bir çalışmayı yönlendirme
title: Yönlendir
x-i18n:
    generated_at: "2026-07-27T00:09:50Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: d420e14982d52520e415103ffa6d86923fad6f13c43ff7741ebbd8dde0d0073f
    source_path: tools/steer.md
    workflow: 16
---

`/steer` önce zaten etkin olan bir çalıştırmaya yönlendirme göndermeyi dener. Bu,
"bu çalıştırma hâlâ devam ederken onu ayarla" durumları içindir. Geçerli çalışma zamanı
yönlendirmeyi kabul edemiyorsa OpenClaw, mesajı bırakmak yerine
normal bir istem olarak gönderir.

## Geçerli oturum

Geçerli oturumun etkin çalıştırmasını hedeflemek için üst düzey `/steer` kullanın:

```text
/steer daha küçük yamayı tercih et ve testleri odaklı tut
/tell sonraki araç çağrısını yapmadan önce özetle
```

Davranış:

- Yalnızca geçerli oturumun etkin çalıştırmasını hedefler.
- Oturumun `/queue` modundan bağımsız çalışır.
- Oturum boştayken veya etkin çalıştırma yönlendirmeyi kabul edemediğinde aynı
  mesajla normal bir tur başlatır.
- Etkin çalışma zamanının yönlendirme yolunu kullanır; böylece model, yönlendirmeyi
  desteklenen bir sonraki çalışma zamanı sınırında görür.

## Yönlendirme ve kuyruk karşılaştırması

`/queue steer`, bir çalıştırma etkinken gelen normal mesajların etkin çalıştırmayı
yönlendirmeyi denemesini sağlar. `/steer <message>`, kayıtlı `/queue` ayarından
bağımsız olarak komutun mesajını desteklenen bir sonraki çalışma zamanı sınırında etkin
çalıştırmaya eklemeyi deneyen açık bir komuttur. Bu ekleme kullanılamadığında komut
öneki kaldırılır ve `<message>` normal bir istem olarak devam eder.

Açık `/steer` (ve `/tell`) komutu Gateway desteklidir.
`openclaw chat` veya `openclaw tui --local` içinde `/queue steer` seçin ve
yönlendirmeyi normal bir mesaj olarak gönderin; gömülü çalışma zamanı, bir Gateway
komutunu iletmeden aynı yönlendirme politikasını uygular.

Şunları kullanın:

- Etkin çalıştırmayı hemen yönlendirmek istediğinizde `/steer <message>`.
- Gelecekteki normal mesajların varsayılan olarak etkin çalıştırmaları
  yönlendirmesini istediğinizde `/queue steer`.
- Gelecekteki normal mesajların etkin çalıştırmayı yönlendirmek yerine sonraki
  bir turu beklemesi gerektiğinde `/queue collect` veya `/queue followup`.
- En yeni mesajın etkin çalıştırmayı yönlendirmek yerine onun yerini alması
  gerektiğinde `/queue interrupt`.

Kuyruk modları ve yönlendirme sınırları için [Komut kuyruğu](/tr/concepts/queue) ve
[Yönlendirme kuyruğu](/tr/concepts/queue-steering) bölümlerine bakın.

## Alt aracılar

Üst düzey `/steer`, geçerli oturumun etkin çalıştırmasını hedefler. Alt aracılar
üst/istekte bulunan oturumlarına geri bildirimde bulunur; `/subagents` yalnızca
görünürlük içindir.

## ACP oturumları

Hedef bir ACP düzenek oturumu olduğunda `/acp steer` kullanın:

```text
/acp steer --session agent:main:acp:codex yeniden üretimi daralt
```

ACP oturumu seçimi ve çalışma zamanı davranışı için [ACP aracıları](/tr/tools/acp-agents)
bölümüne bakın.

## İlgili

- [Eğik çizgi komutları](/tr/tools/slash-commands)
- [Komut kuyruğu](/tr/concepts/queue)
- [Yönlendirme kuyruğu](/tr/concepts/queue-steering)
- [Alt aracılar](/tr/tools/subagents)
