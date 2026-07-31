---
read_when:
    - Yükseltilmiş mod varsayılanlarını, izin listelerini veya eğik çizgi komutu davranışını ayarlama
    - Korumalı alan içindeki ajanların ana sisteme nasıl erişebildiğini anlama
summary: 'Yükseltilmiş yürütme modu: korumalı alandaki bir agent üzerinden komutları korumalı alanın dışında çalıştırma'
title: Yükseltilmiş mod
x-i18n:
    generated_at: "2026-07-27T00:20:21Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 40627217acf56122acfc48b689be1b9e2c61d889fe698e9c3c8fd91270d4a6cf
    source_path: tools/elevated.md
    workflow: 16
---

Bir aracı korumalı alan içinde çalıştığında, `exec` komutları korumalı alan ortamıyla sınırlıdır. **Yükseltilmiş mod**, yapılandırılabilir onay geçitleriyle aracının bunun yerine korumalı alandan çıkıp dışında komut çalıştırmasına olanak tanır.

<Info>
  Yükseltilmiş mod, yalnızca aracı **korumalı alandayken** davranışı değiştirir. Korumalı alanda olmayan aracılar için exec zaten ana makinede çalışır.
</Info>

## Yönergeler

Yükseltilmiş modu eğik çizgi komutlarıyla oturum bazında denetleyin:

| Yönerge        | İşlevi                                                                                                                    |
| ---------------- | ------------------------------------------------------------------------------------------------------------------------------- |
| `/elevated on`   | Yapılandırılmış ana makine yolunda korumalı alanın dışında çalıştırır, onayları korur                                                             |
| `/elevated ask`  | `on` ile aynıdır (takma ad)                                                                                                            |
| `/elevated full` | Yapılandırılmış ana makine yolunda korumalı alanın dışında çalıştırır ve mod/ana makine onay politikası zaten izin vericiyse onayları atlar |
| `/elevated off`  | Korumalı alanla sınırlı yürütmeye döner                                                                                            |

`/elev on|off|ask|full` olarak da kullanılabilir.

Geçerli düzeyi görmek için bağımsız değişken olmadan `/elevated` gönderin.

## Nasıl çalışır?

<Steps>
  <Step title="Kullanılabilirliği denetleyin">
    Yükseltilmiş mod yapılandırmada etkinleştirilmiş ve gönderen izin listesinde olmalıdır:

    ```json5
    {
      tools: {
        elevated: {
          enabled: true,
          allowFrom: {
            discord: ["user-id-123"],
            whatsapp: ["+15555550123"],
          },
        },
      },
    }
    ```

  </Step>

  <Step title="Düzeyi ayarlayın">
    Oturumun varsayılanını ayarlamak için yalnızca yönerge içeren bir ileti gönderin:

    ```
    /elevated full
    ```

    Ya da satır içinde kullanın (yalnızca o iletiye uygulanır):

    ```
    /elevated on dağıtım betiğini çalıştır
    ```

  </Step>

  <Step title="Komutlar korumalı alanın dışında çalışır">
    Yükseltilmiş mod etkinken `exec` çağrıları korumalı alandan çıkar. Etkin ana makine varsayılan olarak
    `gateway`, yapılandırılmış/oturum exec hedefi
    `node` olduğunda ise `node` olur. `full` modunda, çözümlenen exec
    modu/ana makine onay politikası zaten tamamen izin vericiyse exec onayları atlanır (security `full`,
    ask `off`); aksi takdirde normal onay politikası uygulanmaya devam eder.
    `on`/`ask` modunda yapılandırılmış onay kuralları her zaman uygulanır.
  </Step>
</Steps>

## Çözümleme sırası

1. İletideki **satır içi yönerge** (yalnızca o iletiye uygulanır)
2. **Oturum geçersiz kılması** (yalnızca yönerge içeren bir ileti gönderilerek ayarlanır)
3. **Genel varsayılan** (yapılandırmadaki `agents.defaults.elevatedDefault`)

## Kullanılabilirlik ve izin listeleri

- **Genel geçit**: `tools.elevated.enabled` (`true` olmalıdır)
- **Gönderen izin listesi**: kanal başına listeler içeren `tools.elevated.allowFrom`
- **Aracı başına geçit**: `agents.entries.*.tools.elevated.enabled` (yalnızca daha fazla kısıtlayabilir; hem genel hem de aracı başına geçit `true` olmalıdır)
- **Aracı başına izin listesi**: `agents.entries.*.tools.elevated.allowFrom` (gönderen hem genel hem de aracı başına izin listesiyle eşleşmelidir)
- **Kanal tarafından sağlanan yedek izin listesi**: kanal plugin'leri, `tools.elevated.allowFrom.<provider>` yapılandırılmadığında kullanılan bir SDK bağdaştırıcı kancası aracılığıyla isteğe bağlı olarak yedek izin listesi sağlayabilir. Şu anda paketlenmiş hiçbir kanal bu kancayı uygulamadığından pratikte bugün her sağlayıcı için açık bir `tools.elevated.allowFrom.<provider>` girdisi gerekir.
- **Tüm geçitler geçilmelidir**; aksi takdirde yükseltilmiş mod kullanılamıyor kabul edilir

İzin listesi girdi biçimleri:

| Önek                  | Eşleştiği değer                         |
| ----------------------- | ------------------------------- |
| (yok)                  | Gönderen kimliği, E.164 veya From alanı |
| `name:`                 | Gönderenin görünen adı             |
| `username:`             | Gönderenin kullanıcı adı                 |
| `tag:`                  | Gönderen etiketi                      |
| `id:`, `from:`, `e164:` | Açık kimlik hedefleme     |

## Yükseltilmiş modun denetlemedikleri

- **Araç politikası**: `exec` araç politikası tarafından reddedilirse yükseltilmiş mod bunu geçersiz kılamaz.
- **Ana makine seçimi politikası**: yükseltilmiş mod, `auto` değerini serbest bir ana makineler arası geçersiz kılmaya dönüştürmez. Yapılandırılmış/oturum exec hedefi kurallarını kullanır ve yalnızca hedef zaten `node` olduğunda `node` değerini seçer.
- **`/exec` özelliğinden ayrıdır**: `/exec` yönergesi, yetkili gönderenler için oturum başına exec varsayılanlarını (host, security, ask, node) ayarlar ve yükseltilmiş mod gerektirmez.

<Note>
  Bash sohbet komutu (`!` öneki; `/bash` takma adı), kendi `tools.bash.enabled` bayrağına ek olarak `tools.elevated` özelliğinin etkinleştirilmesini gerektiren ayrı bir geçittir. Yükseltilmiş modun devre dışı bırakılması, `!` kabuk komutlarını da engeller.
</Note>

## İlgili konular

<CardGroup cols={2}>
  <Card title="Exec aracı" href="/tr/tools/exec" icon="terminal">
    Aracıdan kabuk komutu yürütme.
  </Card>
  <Card title="Exec onayları" href="/tr/tools/exec-approvals" icon="shield">
    `exec` için onay ve izin listesi sistemi.
  </Card>
  <Card title="Korumalı alana alma" href="/tr/gateway/sandboxing" icon="box">
    Gateway düzeyinde korumalı alan yapılandırması.
  </Card>
  <Card title="Korumalı Alan, Araç Politikası ve Yükseltilmiş Mod" href="/tr/gateway/sandbox-vs-tool-policy-vs-elevated" icon="scale-balanced">
    Üç geçidin bir araç çağrısı sırasında nasıl birleştiği.
  </Card>
</CardGroup>
