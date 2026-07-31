---
read_when:
    - Yerel Docker yerine bulut tarafından yönetilen korumalı alanlar istiyorsunuz
    - OpenShell pluginini kuruyorsunuz
    - Yansıtma ve uzak çalışma alanı modları arasında seçim yapmanız gerekir
summary: OpenShell'i OpenClaw aracıları için yönetilen bir korumalı alan arka ucu olarak kullanın
title: OpenShell
x-i18n:
    generated_at: "2026-07-26T22:46:45Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: bf5c33912bd0db759a01cf58ea26712a8ada68c0804bf16f69f1f7cdd496828c
    source_path: gateway/openshell.md
    workflow: 16
---

OpenShell, yönetilen bir sandbox arka ucudur: OpenClaw, Docker container'larını
yerel olarak çalıştırmak yerine sandbox yaşam döngüsünü uzak ortamları
hazırlayan ve komutları SSH üzerinden yürüten `openshell` CLI'ye devreder.

Plugin, genel [SSH arka ucuyla](/tr/gateway/sandboxing#ssh-backend) aynı SSH
aktarımını ve uzak dosya sistemi köprüsünü yeniden kullanır; ayrıca OpenShell
yaşam döngüsünü (`sandbox create/get/delete/ssh-config`) ve isteğe bağlı `mirror`
çalışma alanı eşitleme modunu ekler.

## Ön koşullar

- OpenShell plugin'i yüklü (`openclaw plugins install @openclaw/openshell-sandbox`)
- `openshell` CLI, `PATH` üzerinde (veya
  `plugins.entries.openshell.config.command` aracılığıyla özel bir yolda)
- Sandbox erişimine sahip bir OpenShell hesabı
- Ana makinede çalışan OpenClaw Gateway

## Hızlı başlangıç

```bash
openclaw plugins install @openclaw/openshell-sandbox
```

```json5
{
  agents: {
    defaults: {
      sandbox: {
        mode: "all",
        backend: "openshell",
        scope: "session",
        workspaceAccess: "rw",
      },
    },
  },
  plugins: {
    entries: {
      openshell: {
        enabled: true,
        config: {
          from: "openclaw",
          mode: "remote",
        },
      },
    },
  },
}
```

Gateway'i yeniden başlatın. Bir sonraki ajan turunda OpenClaw bir OpenShell
sandbox'ı oluşturur ve araç yürütmesini bunun üzerinden yönlendirir. Şunlarla
doğrulayın:

```bash
openclaw sandbox list
openclaw sandbox explain
```

## Çalışma alanı modları

Bu, en önemli OpenShell kararıdır.

### mirror (varsayılan)

`plugins.entries.openshell.config.mode: "mirror"`, **yerel çalışma alanını
kanonik** tutar:

- `exec` öncesinde OpenClaw, yerel çalışma alanını sandbox'a eşitler.
- `exec` sonrasında OpenClaw, uzak çalışma alanını yeniden yerele eşitler.
- Dosya araçları sandbox köprüsünden geçer, ancak turlar arasında yerel
  ortam doğruluk kaynağı olarak kalır.

Geliştirme iş akışları için idealdir: OpenClaw dışındaki yerel düzenlemeler
bir sonraki yürütmede görünür ve sandbox, Docker arka ucuna benzer şekilde
davranır.

Ödünleşim: Her yürütme turunda yükleme + indirme maliyeti.

### remote

`mode: "remote"`, **OpenShell çalışma alanını kanonik** hâle getirir:

- İlk sandbox oluşturulduğunda OpenClaw, uzak çalışma alanını yerelden
  yalnızca bir kez başlangıç verileriyle doldurur.
- Bundan sonra `exec`, `read`, `write`, `edit` ve `apply_patch`
  doğrudan uzak çalışma alanında çalışır. OpenClaw, uzak değişiklikleri
  yerele **eşitlemez**.
- İstem sırasındaki medya okumaları çalışmaya devam eder (dosya/medya araçları
  sandbox köprüsü üzerinden okur).

Uzun süre çalışan ajanlar ve CI için idealdir: tur başına ek yük daha düşüktür
ve ana makinedeki yerel düzenlemeler uzak durumun sessizce üzerine yazamaz.

<Warning>
İlk başlangıç verileri aktarıldıktan sonra ana makinede OpenClaw dışında düzenlenen dosyalar uzak sandbox tarafından görülemez. Yeniden başlangıç verileri aktarmak için `openclaw sandbox recreate` komutunu çalıştırın.
</Warning>

### Mod seçimi

|                          | `mirror`                   | `remote`                  |
| ------------------------ | -------------------------- | ------------------------- |
| **Kanonik çalışma alanı**  | Yerel ana makine                 | Uzak OpenShell          |
| **Eşitleme yönü**       | Çift yönlü (her yürütmede) | Tek seferlik başlangıç verisi aktarımı             |
| **Tur başına ek yük**    | Daha yüksek (yükleme + indirme) | Daha düşük (doğrudan uzak işlemler) |
| **Yerel düzenlemeler görünür mü?** | Evet, sonraki yürütmede          | Hayır, yeniden oluşturulana kadar        |
| **En uygun kullanım**             | Geliştirme iş akışları      | Uzun süre çalışan ajanlar, CI   |

## Yapılandırma referansı

Tüm OpenShell yapılandırması `plugins.entries.openshell.config` altında bulunur:

| Anahtar                       | Tür                     | Varsayılan       | Açıklama                                                                            |
| ------------------------- | ------------------------ | ------------- | -------------------------------------------------------------------------------------- |
| `mode`                    | `"mirror"` veya `"remote"` | `"mirror"`    | Çalışma alanı eşitleme modu                                                                    |
| `command`                 | `string`                 | `"openshell"` | `openshell` CLI'nin yolu veya adı                                                    |
| `from`                    | `string`                 | `"openclaw"`  | İlk oluşturma için sandbox kaynağı                                                   |
| `gateway`                 | `string`                 | ayarlanmamış         | OpenShell gateway adı (üst düzey `--gateway`)                                         |
| `gatewayEndpoint`         | `string`                 | ayarlanmamış         | OpenShell gateway uç noktası (üst düzey `--gateway-endpoint`)                            |
| `policy`                  | `string`                 | ayarlanmamış         | Sandbox oluşturma için OpenShell politika kimliği                                               |
| `providers`               | `string[]`               | `[]`          | Sandbox oluşturulurken eklenen sağlayıcı adları (yinelenenler kaldırılır, giriş başına bir `--provider` bayrağı) |
| `gpu`                     | `boolean`                | `false`       | GPU kaynakları iste (`--gpu`)                                                        |
| `autoProviders`           | `boolean`                | `true`        | Oluşturma sırasında `--auto-providers` (false olduğunda `--no-auto-providers`) geçir            |
| `remoteWorkspaceDir`      | `string`                 | `"/sandbox"`  | Sandbox içindeki birincil yazılabilir çalışma alanı                                          |
| `remoteAgentWorkspaceDir` | `string`                 | `"/agent"`    | Ajan çalışma alanının bağlama yolu (çalışma alanı erişimi `rw` olmadığında salt okunur)               |
| `timeoutSeconds`          | `number`                 | `120`         | `openshell` CLI işlemlerinin zaman aşımı                                                 |

`remoteWorkspaceDir` ve `remoteAgentWorkspaceDir` mutlak yollar olmalı ve yönetilen
`/sandbox` veya `/agent` kökleri altında kalmalıdır; diğer mutlak yollar
reddedilir.

Sandbox düzeyindeki ayarlar (`mode`, `scope`, `workspaceAccess`) diğer
arka uçlarda olduğu gibi `agents.defaults.sandbox` altında bulunur. Tam matris için
[Sandbox Kullanımı](/tr/gateway/sandboxing) sayfasına bakın.

## Örnekler

### En küçük remote kurulumu

```json5
{
  agents: {
    defaults: {
      sandbox: {
        mode: "all",
        backend: "openshell",
      },
    },
  },
  plugins: {
    entries: {
      openshell: {
        enabled: true,
        config: {
          from: "openclaw",
          mode: "remote",
        },
      },
    },
  },
}
```

### GPU ile mirror modu

```json5
{
  agents: {
    defaults: {
      sandbox: {
        mode: "all",
        backend: "openshell",
        scope: "agent",
        workspaceAccess: "rw",
      },
    },
  },
  plugins: {
    entries: {
      openshell: {
        enabled: true,
        config: {
          from: "openclaw",
          mode: "mirror",
          gpu: true,
          providers: ["openai"],
          timeoutSeconds: 180,
        },
      },
    },
  },
}
```

### Özel gateway ile ajan başına OpenShell

```json5
{
  agents: {
    defaults: {
      sandbox: { mode: "off" },
    },
    list: [
      {
        id: "researcher",
        sandbox: {
          mode: "all",
          backend: "openshell",
          scope: "agent",
          workspaceAccess: "rw",
        },
      },
    ],
  },
  plugins: {
    entries: {
      openshell: {
        enabled: true,
        config: {
          from: "openclaw",
          mode: "remote",
          gateway: "lab",
          gatewayEndpoint: "https://lab.example",
          policy: "strict",
        },
      },
    },
  },
}
```

## Yaşam döngüsü yönetimi

```bash
# Tüm sandbox çalışma ortamlarını listele (Docker + OpenShell)
openclaw sandbox list

# Geçerli politikayı incele
openclaw sandbox explain

# Yeniden oluştur (uzak çalışma alanını siler, sonraki kullanımda yeniden başlangıç verileri aktarır)
openclaw sandbox recreate --all
```

`remote` modu için yeniden oluşturma özellikle önemlidir: bu kapsamın
kanonik uzak çalışma alanını siler ve sonraki kullanımda yerelden yeni bir
çalışma alanına başlangıç verileri aktarılır. `mirror` modunda yerel
ortam kanonik kaldığından yeniden oluşturma, esas olarak uzak yürütme ortamını
sıfırlar.

Şunlardan herhangi birini değiştirdikten sonra yeniden oluşturun:

- `agents.defaults.sandbox.backend`
- `plugins.entries.openshell.config.from`
- `plugins.entries.openshell.config.mode`
- `plugins.entries.openshell.config.policy`

## Güvenliği sağlamlaştırma

Mirror modu dosya sistemi köprüsü, yerel çalışma alanı kökünü sabitler ve her
okuma, yazma, mkdir, kaldırma ve yeniden adlandırma işleminden önce kanonik
yolları (realpath aracılığıyla) yeniden denetleyerek yolun ortasındaki sembolik
bağlantıları reddeder. Bir sembolik bağlantı değişimi veya yeniden bağlanan
çalışma alanı, dosya erişimini yansıtılan ağacın dışına yönlendiremez.

## Mevcut sınırlamalar

- Sandbox tarayıcısı OpenShell arka ucunda desteklenmez.
- Bağlamalar yapılandırılmışsa `sandbox.docker.binds` OpenShell'e uygulanmaz;
  sandbox oluşturma başarısız olur.
- `sandbox.docker.*` altındaki Docker'a özgü çalışma ortamı ayarları (`env` dışında)
  yalnızca Docker arka ucuna uygulanır.

## Nasıl çalışır?

1. OpenClaw, sandbox adı için (yapılandırılmış tüm
   `--gateway`/`--gateway-endpoint` değerleriyle) `sandbox get` komutunu çalıştırır; bu başarısız olursa
   `sandbox create` ile bir tane oluşturur ve ayarlandıklarında `--name`, `--from`, `--policy`,
   etkin olduğunda `--gpu`, `--auto-providers`/`--no-auto-providers` ve yapılandırılmış her
   sağlayıcı için bir `--provider` bayrağı geçirir.
2. OpenClaw, SSH bağlantı ayrıntılarını almak için sandbox adına
   `sandbox ssh-config` komutunu çalıştırır.
3. Çekirdek, SSH yapılandırmasını geçici bir dosyaya yazar ve genel SSH arka
   ucuyla aynı uzak dosya sistemi köprüsü üzerinden bir SSH oturumu açar.
4. `mirror` modunda: yürütmeden önce yerelden uzağa eşitle, çalıştır, ardından yeniden yerele eşitle.
5. `remote` modunda: oluşturma sırasında bir kez başlangıç verileri aktar, ardından doğrudan
   uzak çalışma alanında çalış.

## İlgili konular

- [Sandbox Kullanımı](/tr/gateway/sandboxing) - modlar, kapsamlar ve arka uç karşılaştırması
- [Sandbox, Araç Politikası ve Yükseltilmiş Arasındaki Farklar](/tr/gateway/sandbox-vs-tool-policy-vs-elevated) - engellenen araçlarda hata ayıklama
- [Çok Ajanlı Sandbox ve Araçlar](/tr/tools/multi-agent-sandbox-tools) - ajan başına geçersiz kılmalar
- [Sandbox CLI](/tr/cli/sandbox) - `openclaw sandbox` komutları
