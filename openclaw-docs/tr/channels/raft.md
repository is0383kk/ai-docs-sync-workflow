---
read_when:
    - OpenClaw'ı bir Raft çalışma alanına bağlamak istiyorsunuz
    - Bir Raft Harici Aracısı yapılandırıyorsunuz
    - Raft uyandırma iletiminde hata ayıklıyorsunuz
sidebarTitle: Raft
summary: Raft CLI uyandırma köprüsü aracılığıyla Raft Harici Ajan desteği
title: Raft
x-i18n:
    generated_at: "2026-07-26T23:49:51Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 454d92d764a4ec3b0ec52467cba254dcad795870e04d1d32d4cf65d8b451a0de
    source_path: channels/raft.md
    workflow: 16
---

Raft, yerel Raft CLI aracılığıyla bir OpenClaw ajanını bir Raft External Agent'a bağlar. Raft, Gateway'e kimliği doğrulanmış uyandırma ipuçları gönderir; ardından ajan, mesajları kontrol etmek ve göndermek için Raft CLI aracını kullanır. Yalnızca doğrudan sohbet desteklenir (gruplar desteklenmez).

## Kurulum

Raft, resmi bir harici Plugin'dir. Gateway ana makinesine kurun:

```bash
openclaw plugins install @openclaw/raft
openclaw gateway restart
```

Ayrıntılar: [Plugin'ler](/tr/tools/plugin)

## Ön Koşullar

- External Agent içeren bir Raft çalışma alanı.
- Raft CLI aracının OpenClaw Gateway ile aynı ana makinede, hizmetin
  `PATH` konumunda kurulu olması.
- Oturumu önceden açılmış ve söz konusu External Agent ile ilişkilendirilmiş
  bir Raft CLI profili.

Plugin, Raft kimlik bilgilerini saklamaz; Raft CLI bu kimlik doğrulamasını kendi profilinde tutar.

## Yapılandırma

Profili yapılandırmada ayarlayın:

```json5
{
  channels: {
    raft: {
      enabled: true,
      profile: "openclaw",
    },
  },
}
```

Varsayılan hesap için bunun yerine Gateway ortamında `RAFT_PROFILE` ayarlayabilirsiniz:

```bash
RAFT_PROFILE=openclaw
```

Tek bir Gateway birden fazla Raft External Agent'a bağlandığında adlandırılmış bir hesap kullanın:

```json5
{
  channels: {
    raft: {
      accounts: {
        support: {
          profile: "support-agent",
        },
        engineering: {
          profile: "engineering-agent",
        },
      },
    },
  },
}
```

Etkileşimli kurulum aynı profili kaydeder:

```bash
openclaw channels add --channel raft
```

## Çalışma biçimi

Gateway başlatıldığında Plugin:

1. Geçici bir bağlantı noktasında yalnızca geri döngüden erişilebilen bir HTTP uyandırma uç noktası açar.
2. Söz konusu uç nokta ve işlem başına bir belirteç ile `raft --profile <profile> agent bridge` başlatır.
3. Yerel köprüden yalnızca kimliği doğrulanmış, içeriksiz ve yeniden oynatma kimliğine sahip uyandırma ipuçlarını kabul eder.
4. Her uyandırma yükünde `eventId`, `attemptId`, `messageId`, `delivery_id`,
   `wake_id` veya `id` değerlerinden birinin bulunmasını zorunlu kılar.
5. Yeniden denenen uyandırma teslimlerini, Gateway yeniden başlatmaları da dâhil olmak üzere, köprü olay kimliğine göre 24 saat boyunca yinelenenlerden arındırır.
6. Geçerli köprü için kararlı bir çalışma zamanı oturumu ve Raft CLI protokolü için boş bir etkinlik boşaltma grubu döndürür.
7. Kabul edilen her uyandırma için seri hâle getirilmiş bir OpenClaw ajan turu başlatır.

Raft teslimatlarının yeniden denenmesini ve yeniden bağlanmayı köprü yönetir. OpenClaw turu, kopyalanmış bir Raft mesaj gövdesi değil, yalnızca bir uyandırma bildirimi alır. Bekleyen mesajları okumak ve yanıtını göndermek için CLI aracını kullanır:

```bash
raft --profile openclaw message check
raft --profile openclaw message send
```

<Note>
Raft, anlık mesaj aktarımı değildir. OpenClaw, modelin nihai metnini köprü üzerinden otomatik olarak geri göndermez; bu nedenle ajan, bir uyandırmayı işledikten sonra Raft CLI aracını kullanmalıdır.
</Note>

## Doğrulama

OpenClaw'ın CLI aracını bulabildiğini ve yapılandırılmış bir profile sahip olduğunu kontrol edin:

```bash
openclaw channels status --probe
openclaw plugins inspect raft --runtime --json
```

Ardından Raft External Agent'a bir mesaj gönderin. Gateway günlüğünde önce Raft köprüsünün başlatıldığı, ardından gelen bir uyandırma gösterilmelidir. Ajan, bekleyen mesajlarını kontrol etmek için yapılandırılmış Raft profilini kullanmalıdır.

## Sorun Giderme

<AccordionGroup>
  <Accordion title="Raft CLI eksik">
    Raft CLI aracını Gateway ana makinesine kurun ve hizmetin `PATH` konumunda
    `raft` kullanılabilir hâle getirin. `raft --help` ile doğrulayın, ardından Gateway'i yeniden başlatın.
  </Accordion>
  <Accordion title="Köprü hemen kapanıyor">
    Yapılandırılmış profilde oturum açıldığını ve profilin amaçlanan
    Raft External Agent'a ait olduğunu doğrulayın. CLI tanılamasını görmek için
    doğrudan `raft --profile <profile> agent bridge` çalıştırın.
  </Accordion>
  <Accordion title="Bir uyandırma geliyor ancak Raft yanıtı gönderilmiyor">
    Ajan Raft CLI aracını çağırmadığında bu beklenen bir durumdur. Uyandırma
    köprüsü mesaj gövdelerini veya otomatik nihai yanıtları taşımaz. Ajanın
    araç politikasını kontrol edin ve `raft --profile <profile>
    message check` ile `message send` çalıştırabildiğinden emin olun.
  </Accordion>
</AccordionGroup>

## Kaynaklar

- [Raft](https://raft.build/)
- [Raft belgeleri](https://docs.raft.build/welcome/)
- [Hermes Raft entegrasyonu](https://hermes-agent.nousresearch.com/docs/user-guide/messaging/raft)
