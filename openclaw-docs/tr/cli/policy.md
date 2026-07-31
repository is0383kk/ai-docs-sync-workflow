---
read_when:
    - OpenClaw ayarlarını yazılmış bir policy.jsonc dosyasına göre denetlemek istiyorsunuz
    - Doctor lintinde politika bulguları istiyorsunuz
    - Denetim kanıtı için bir politika doğrulama karmasına ihtiyacınız var
summary: '`openclaw policy` uyumluluk denetimleri için CLI başvurusu'
title: Politika
x-i18n:
    generated_at: "2026-07-26T22:41:56Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 63e4faeab8dd6535e3d517439d3f58cdc167b6b7fade808a6482742ec9b5acf1
    source_path: cli/policy.md
    workflow: 16
---

# `openclaw policy`

`openclaw policy`, paketle gelen Policy plugin tarafından sağlanır. Mevcut OpenClaw ayarları üzerinde çalışan kurumsal bir
uyumluluk katmanıdır; ikinci bir yapılandırma sistemi değildir.
Gereksinimleri `policy.jsonc` içinde tanımlarsınız; OpenClaw etkin çalışma alanını
kanıt olarak gözlemler; Policy, sapmaları `doctor --lint` aracılığıyla bildirir. Policy,
araç çağrılarını zorunlu kılmaz veya istek sırasında çalışma zamanı davranışını yeniden yazmaz ve
`auth-profiles.json` gibi ajan başına kimlik bilgisi depolarını tasdik etmez.

Policy; yapılandırılmış kanalları, MCP sunucularını, model sağlayıcılarını, ağ SSRF
duruşunu, giriş/kanal erişimini, Gateway erişilebilirliğini ve Node komut duruşunu,
tanımlanmış mesaj yönlendirme yoklamalarını,
ajan çalışma alanı erişimini, korumalı alan duruşunu, veri işleme duruşunu, gizli bilgi
sağlayıcısı/kimlik doğrulama profili duruşunu ve yönetişim altındaki araç meta verilerini (`TOOLS.md`) denetler. Bir
çalışma alanında "Telegram etkinleştirilmemelidir" veya "yönetişim altındaki araçlar risk ve
sahip meta verilerini bildirmelidir" gibi kalıcı ve denetlenebilir bir beyan gerektiğinde bunu kullanın. Tasdik veya
sapma algılama olmadan yalnızca yerel davranış gerekiyorsa düz
yapılandırma yeterlidir.

## Hızlı başlangıç

```bash
openclaw plugins enable policy
```

Plugin, `policy.jsonc` eksik olduğunda bile etkin kalır; böylece doctor,
denetimleri sessizce atlamak yerine eksik yapıtı bildirebilir.

`policy.jsonc` dosyasını elle oluşturun; mevcut ayarlardan oluşturulmaz. Her
üst düzey bölüm bir kural ad alanıdır: bir denetim yalnızca altında somut bir kural
bulunduğunda çalışır (desteklenmeyen bölümler veya anahtarlar sessizce yok sayılmak yerine
`policy/policy-jsonc-invalid` olarak başarısız olur). Desteklenen
tüm bölümleri kapsayan asgari örnek:

```jsonc
{
  "channels": {
    "denyRules": [
      {
        "id": "no-telegram",
        "when": { "provider": "telegram" },
        "reason": "Telegram bu çalışma alanı için onaylanmamıştır.",
      },
    ],
  },
  "mcp": {
    "servers": {
      "allow": ["docs"],
      "deny": ["untrusted"],
    },
  },
  "models": {
    "providers": {
      "allow": ["openai", "anthropic"],
      "deny": ["openrouter"],
    },
  },
  "network": {
    "privateNetwork": {
      "allow": false,
    },
  },
  "routing": {
    "requireBindings": true,
    "requireConfiguredChannels": true,
    "probes": [
      {
        "id": "family-dm",
        "route": {
          "channel": "imessage",
          "peer": { "kind": "direct", "id": "+15555550123" },
        },
        "expect": {
          "agentId": "family",
          "matchedBy": ["binding.peer"],
        },
      },
    ],
  },
  "ingress": {
    "session": {
      "requireDmScope": "per-channel-peer",
    },
    "channels": {
      "allowDmPolicies": ["pairing", "allowlist", "disabled"],
      "denyOpenGroups": true,
      "requireMentionInGroups": true,
    },
  },
  "gateway": {
    "exposure": {
      "allowNonLoopbackBind": false,
      "allowTailscaleFunnel": false,
    },
    "auth": {
      "requireAuth": true,
      "requireExplicitRateLimit": true,
    },
    "controlUi": {
      "allowInsecure": false,
    },
    "remote": {
      "allow": false,
    },
    "http": {
      "denyEndpoints": ["chatCompletions", "responses"],
      "requireUrlAllowlists": true,
    },
    "nodes": {
      "denyCommands": ["system.run"],
    },
  },
  "agents": {
    "workspace": {
      "allowedAccess": ["none", "ro"],
      "denyTools": ["exec", "process", "write", "edit", "apply_patch"],
    },
  },
  "dataHandling": {
    "sensitiveLogging": {
      "requireRedaction": true,
    },
    "telemetry": {
      "denyContentCapture": true,
    },
    "retention": {
      "requireSessionMaintenance": true,
    },
    "memory": {
      "denySessionTranscriptIndexing": true,
    },
  },
  "secrets": {
    "requireManagedProviders": true,
    "denySources": ["exec"],
    "allowInsecureProviders": false,
  },
  "auth": {
    "profiles": {
      "requireMetadata": ["provider", "mode"],
      "allowModes": ["api_key", "token"],
    },
  },
  "execApprovals": {
    "requireFile": true,
    "defaults": { "allowSecurity": ["deny"] },
    "agents": {
      "allowSecurity": ["deny", "allowlist"],
      "allowAutoAllowSkills": false,
      "allowlist": { "expected": ["deploy", "status"] },
    },
  },
  "tools": {
    "requireMetadata": ["risk", "sensitivity", "owner"],
    "profiles": {
      "allow": ["messaging", "minimal"],
    },
    "fs": {
      "requireWorkspaceOnly": true,
    },
    "exec": {
      "allowSecurity": ["deny", "allowlist"],
      "requireAsk": ["always"],
      "allowHosts": ["sandbox"],
    },
    "elevated": {
      "allow": false,
    },
    "denyTools": ["group:runtime", "group:fs"],
  },
}
```

Aşağıdaki kural tablolarından kolayca anlaşılamayan genel notlar:

- Geri döngü dışı bağlamaları reddederken `gateway.bind` değerini belirtmemek,
  çalışma zamanı varsayılanını kabul ettiğiniz anlamına gelir; katı uyumluluk için `gateway.bind: "loopback"` değerini ayarlayın.
- Salt okunur bir ajan için geçerli varsayılanlarda/ajanda korumalı alan `mode` değerini
  `all` veya `non-main`, `workspaceAccess` değerini ise `none` veya `ro` olarak ayarlayın. Eksik veya
  `off` korumalı alan modu, salt okunur bir politikayı karşılamaz.
- `agents.workspace.denyTools`; `exec`, `process`, `write`, `edit` ve
  `apply_patch` değerlerini kabul eder. Yapılandırmadaki araç reddetme grupları `group:fs` (dosya değiştirme) ve
  `group:runtime` (kabuk/işlem) eşdeğer duruşu karşılar.
- Yürütme onayı denetimleri, yalnızca bir
  `execApprovals` kuralı mevcut olduğunda canlı `exec-approvals.json` yapıtını okur; eksik veya geçersiz bir yapıt
  yapay bir başarılı sonuç değil, gözlemlenemeyen kanıttır.
- Gizli bilgi ve kimlik doğrulama profili kanıtları yalnızca sağlayıcı/kaynak duruşunu ve
  SecretRef meta verilerini kaydeder; ham değerleri asla kaydetmez. Policy,
  `auth-profiles.json` gibi ajan başına kimlik bilgisi depolarını okumaz veya tasdik etmez.
- Veri işleme kanıtı yalnızca yapılandırma düzeyindeki duruştur (maskeleme modu,
  telemetri yakalama anahtarı, oturum bakım modu, transkript dizinleme
  ayarı). Günlükleri, telemetri dışa aktarımlarını, transkriptleri veya
  bellek dosyalarını incelemez ve temiz bir sonuç, bunlarda kişisel veri veya
  gizli bilgi bulunmadığını kanıtlamaz.
- Yönlendirme yoklamaları, OpenClaw'ın çalışma zamanı bağlama çözümleyicisini yeniden kullanır. Yönlendirme kanıtı
  yalnızca yoklama kimliğini, çözümlenen ajanı, eşleşme türünü ve maskelenmiş bağlama
  meta verilerini kaydeder. Eş, hesap, lonca, takım veya rol tanımlayıcılarını asla kaydetmez.
  Bir yönlendirme bölümü eklemek, politika ve tasdik
  karmalarını bilinçli olarak değiştirir; yönlendirme içermeyen politikalar mevcut kanıt biçimlerini korur.

### Politika kuralı başvurusu

Aşağıdaki her kural isteğe bağlıdır; bir denetim yalnızca kural mevcut olduğunda çalışır.
Gözlemlenen durum, mevcut OpenClaw yapılandırması veya çalışma alanı meta verileridir.

#### Kapsamlı katmanlar

Belirli ajanlar veya kanallar üst düzey temel çizgiden daha katı bir politikaya
ihtiyaç duyduğunda `scopes.<scopeName>` kullanın. Kapsam adı yalnızca bir etikettir; eşleştirme,
kapsam içindeki seçiciyi kullanır. Katmanlar birikimlidir: genel kural çalışmaya devam eder
ve kapsamlı kural aynı kanıta karşı kendi bulgusunu ekleyebilir.

| Seçici     | Desteklenen bölümler                                                             | Kullanım amacı                                          |
| ------------ | ------------------------------------------------------------------------------ | ------------------------------------------------- |
| `agentIds`   | `tools`, `agents.workspace`, `sandbox`, `dataHandling.memory`, `execApprovals` | Bir veya daha fazla çalışma zamanı ajanı daha katı kurallara ihtiyaç duyduğunda.   |
| `channelIds` | `ingress.channels`                                                             | Bir veya daha fazla kanal daha katı giriş kurallarına ihtiyaç duyduğunda. |

Bir `agentIds` girdisi `agents.entries.*` içinde mevcut değilse OpenClaw,
kapsamlı kuralı atlamak yerine söz konusu çalışma zamanı
ajan kimliği için devralınan genel/varsayılan duruşa göre değerlendirir.

```jsonc
{
  "tools": {
    "exec": {
      "allowHosts": ["sandbox", "node"],
    },
  },
  "sandbox": {
    "requireMode": ["all", "non-main"],
  },
  "scopes": {
    "release-workspace": {
      "agentIds": ["release-agent", "review-agent"],
      "agents": {
        "workspace": {
          "allowedAccess": ["none", "ro"],
        },
      },
    },
    "release-lockdown": {
      "agentIds": ["release-agent"],
      "tools": {
        "exec": {
          "allowHosts": ["sandbox"],
          "allowSecurity": ["deny", "allowlist"],
          "requireAsk": ["always"],
        },
        "denyTools": ["exec", "process", "write", "edit", "apply_patch"],
      },
      "sandbox": {
        "requireMode": ["all"],
        "allowBackends": ["docker"],
      },
      "dataHandling": {
        "memory": {
          "denySessionTranscriptIndexing": true,
        },
      },
    },
    "shell-sandbox": {
      "agentIds": ["shell-agent"],
      "sandbox": {
        "allowBackends": ["openshell"],
        "containers": {
          "requireReadOnlyMounts": false,
        },
      },
    },
    "telegram-ingress": {
      "channelIds": ["telegram"],
      "ingress": {
        "channels": {
          "allowDmPolicies": ["pairing"],
          "denyOpenGroups": true,
          "requireMentionInGroups": true,
        },
      },
    },
  },
}
```

Yukarıdaki gibi, her kapsam farklı bir alanı yönetiyorsa aynı ajan birden fazla kapsamda
yer alabilir. Aynı ajan için yinelenen kapsamlı bir alan eşit derecede veya
daha kısıtlayıcı olmalıdır; daha zayıf bir yinelenen iddia reddedilir (izin listeleri
alt kümeler, ret listeleri üst kümelerdir; gerekli Boole değerleri sabittir).

Kapsayıcı duruş kuralları (`sandbox.containers.*`) yalnızca
eşleşen ajanın korumalı alan arka ucunun sunabildiği kanıtlara göre denetlenir. Bir arka uç,
kendisi için etkinleştirdiğiniz bir kuralı gözlemleyemiyorsa Policy başarılı saymak yerine
`policy/sandbox-container-posture-unobservable` bildirir; kapsayıcı
kurallarını bunları sunabilen bir arka uç kullanan ajan gruplarıyla sınırlayın.

Üst düzey `ingress.session.requireDmScope` genel kalır; `session.dmScope`,
kanala atfedilebilir bir kanıt değildir, bu nedenle `channelIds` ile kapsamlandırılamaz.

`policy.jsonc` içinde bulunan her kapsam geçerli ve uygulanabilir olmalıdır.

#### Kanallar

| Politika alanı                         | Gözlemlenen durum                          | Kullanım amacı                                                     |
| ------------------------------------ | --------------------------------------- | ------------------------------------------------------------ |
| `channels.denyRules[].when.provider` | `channels.*` sağlayıcısı ve etkin durumu | `telegram` gibi bir sağlayıcının yapılandırılmış kanallarını reddetmek için. |
| `channels.denyRules[].reason`        | Bulgu iletisi ve düzeltme ipucu bağlamı | Sağlayıcının neden reddedildiğini açıklamak için.                          |

#### MCP sunucuları

| Politika alanı        | Gözlemlenen durum      | Kullanım amacı                                                   |
| ------------------- | ------------------- | ---------------------------------------------------------- |
| `mcp.servers.allow` | `mcp.servers.*` kimlikleri | Yapılandırılmış her MCP sunucusunun bir izin listesinde bulunmasını zorunlu kılmak için. |
| `mcp.servers.deny`  | `mcp.servers.*` kimlikleri | Yapılandırılmış belirli MCP sunucusu kimliklerini reddetmek için.                   |

#### Model sağlayıcıları

| Politika alanı             | Gözlemlenen durum                                   | Kullanım amacı                                                                        |
| ------------------------ | ------------------------------------------------ | ------------------------------------------------------------------------------- |
| `models.providers.allow` | `models.providers.*` kimlikleri ve seçilen model başvuruları | Yapılandırılmış sağlayıcıların ve seçilen model başvurularının onaylı sağlayıcıları kullanmasını zorunlu kılmak için. |
| `models.providers.deny`  | `models.providers.*` kimlikleri ve seçilen model başvuruları | Yapılandırılmış sağlayıcıları ve seçilen model başvurularını sağlayıcı kimliğine göre reddetmek için.               |

#### Ağ

| İlke alanı                   | Gözlemlenen durum                      | Kullanılacağı durum                                                           |
| ------------------------------ | ----------------------------------- | ------------------------------------------------------------------ |
| `network.privateNetwork.allow` | Özel ağ SSRF kaçış yolları | Özel ağ erişiminin devre dışı kalmasını zorunlu kılmak için `false` olarak ayarlayın. |

#### İleti yönlendirme

| İlke alanı                        | Gözlemlenen durum                                      | Kullanılacağı durum                                                               |
| ----------------------------------- | --------------------------------------------------- | ---------------------------------------------------------------------- |
| `routing.requireBindings`           | ACP bağlamaları hariç kanal rota bağlamaları      | En az bir ileti yönlendirme bağlamasını zorunlu kılın.                          |
| `routing.requireConfiguredChannels` | Bağlama kanal kimlikleri ve yapılandırılmış `channels.*` kimlikleri | Eski veya yanlış yazılmış bağlama kanal kimliklerini tespit edin.                        |
| `routing.probes[].route`            | Genel OpenClaw rota çözümleyicisi                  | İleti göndermeden temsili bir gelen rotayı açıklayın.     |
| `routing.probes[].expect.agentId`   | Çözümlenen aracı kimliği                                   | Rotanın incelenen aracıya ulaşmasını zorunlu kılın.                         |
| `routing.probes[].expect.matchedBy` | Çözümleyici eşleşme türü                                 | Eş, hesap, kanal veya incelenen başka bir bağlama özgüllüğünü zorunlu kılın. |

Yoklama kimlikleri benzersiz olmalıdır. Bir rota `channel`, isteğe bağlı `accountId`,
`peer`, `parentPeer`, `guildId`, `teamId` ve `memberRoleIds` öğelerini destekler. Eş türleri
`direct`, `group` ve `channel` şeklindedir. `matchedBy`, `binding.peer`, `binding.account`, `binding.channel`
veya `default` dahil olmak üzere bir ya da daha fazla çalışma zamanı
eşleşme türü içerebilir.

Yönlendirme denetimleri yalnızca uygunluk denetimleridir. Başlatma,
ileti teslimi, bağlama önceliği veya geri dönüş davranışını değiştirmezler. Bir bağlamanın otomatik olarak değiştirilmesi
özel iletileri yeniden yönlendirebileceğinden bulgular operatör
incelemesi gerektirir.

#### Giriş ve kanal erişimi

| İlke alanı                              | Gözlemlenen durum                                                 | Kullanılacağı durum                                                           |
| ----------------------------------------- | -------------------------------------------------------------- | ------------------------------------------------------------------ |
| `ingress.session.requireDmScope`          | `session.dmScope`                                              | İncelenmiş bir doğrudan ileti yalıtım kapsamını zorunlu kılın.                 |
| `ingress.channels.allowDmPolicies`        | `channels.*.dmPolicy` ve eski kanal DM ilkesi alanları      | Yalnızca incelenmiş doğrudan ileti kanal ilkelerine izin verin.               |
| `ingress.channels.denyOpenGroups`         | Kanal, hesap ve grup giriş ilkesi                     | Yapılandırılmış kanallar ve hesaplar için açık grup girişini reddedin.      |
| `ingress.channels.requireMentionInGroups` | Kanal, hesap, grup, sunucu ve iç içe bahsetme kapısı yapılandırması | Grup girişi açık veya bahsetme kapılı olduğunda bahsetme kapılarını zorunlu kılın. |

#### Gateway

| İlke alanı                            | Gözlemlenen durum                                 | Kullanılacağı durum                                                                             |
| --------------------------------------- | ---------------------------------------------- | ------------------------------------------------------------------------------------ |
| `gateway.exposure.allowNonLoopbackBind` | `gateway.bind`                                 | Geri döngü Gateway bağlamasını zorunlu kılmak için `false` olarak ayarlayın.                                  |
| `gateway.exposure.allowTailscaleFunnel` | Tailscale sunma/tünelleme Gateway duruşu         | Tailscale Funnel erişimini reddetmek için `false` olarak ayarlayın.                                    |
| `gateway.auth.requireAuth`              | `gateway.auth.mode`                            | Devre dışı bırakılmış Gateway kimlik doğrulamasını reddetmek için `true` olarak ayarlayın.                                       |
| `gateway.auth.requireExplicitRateLimit` | `gateway.auth.rateLimit`                       | Açık kimlik doğrulama hız sınırı yapılandırmasını zorunlu kılmak için `true` olarak ayarlayın.                            |
| `gateway.controlUi.allowInsecure`       | Control UI güvenli olmayan kimlik doğrulama/cihaz/kaynak geçişleri | Güvenli olmayan Control UI erişim geçişlerini reddetmek için `false` olarak ayarlayın.                         |
| `gateway.remote.allow`                  | Uzak Gateway modu/yapılandırması                     | Uzak Gateway modunu reddetmek için `false` olarak ayarlayın.                                          |
| `gateway.http.denyEndpoints`            | Gateway HTTP API uç noktaları                     | `chatCompletions` veya `responses` gibi uç nokta kimliklerini reddedin.                          |
| `gateway.http.requireUrlAllowlists`     | Gateway HTTP URL getirme girdileri                  | URL getirme girdilerinde URL izin listelerini zorunlu kılmak için `true` olarak ayarlayın.                         |
| `gateway.nodes.denyCommands`            | `gateway.nodes.commands.deny`                  | `system.run` gibi tam Node komut kimliklerinin OpenClaw yapılandırmasında reddedilmesini zorunlu kılın. |

`gateway.nodes.denyCommands`, tam ve büyük/küçük harfe duyarlı bir ilke reddetme üst kümesi kuralıdır.
İlkenin, ayrıcalıklı Node komutlarının OpenClaw yapılandırması tarafından açıkça
reddedildiğini kanıtlaması gerektiğinde bunu kullanın. Ayrıcalıklı bir Node komutuna kasıtlı olarak izin veren
bir dağıtım, yalnızca `gateway.nodes.commands.allow` öğesine güvenmek yerine incelemeden sonra
`policy.jsonc` öğesini güncellemelidir.

#### Aracı çalışma alanı

| İlke alanı                     | Gözlemlenen durum                                                                           | Kullanılacağı durum                                                                                 |
| -------------------------------- | ---------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------- |
| `agents.workspace.allowedAccess` | `agents.defaults.sandbox.workspaceAccess` ve `agents.entries.*.sandbox.workspaceAccess` | Yalnızca `none` veya `ro` gibi korumalı alan çalışma alanı erişim değerlerine izin verin.                       |
| `agents.workspace.denyTools`     | Genel ve aracı başına araç reddetme yapılandırması                                                    | Değişiklik araçlarının (`exec`, `process`, `write`, `edit`, `apply_patch`) reddedilmesini zorunlu kılın. |

#### Korumalı alan duruşu

| İlke alanı                                          | Gözlemlenen durum                                          | Kullanılacağı durum                                                       |
| ----------------------------------------------------- | ------------------------------------------------------- | -------------------------------------------------------------- |
| `sandbox.requireMode`                                 | `agents.defaults.sandbox.mode` ve aracı başına mod       | Yalnızca `all` veya `non-main` gibi incelenmiş korumalı alan modlarına izin verin. |
| `sandbox.allowBackends`                               | `agents.defaults.sandbox.backend` ve aracı başına arka uç | Yalnızca `docker` gibi incelenmiş korumalı alan arka uçlarına izin verin.         |
| `sandbox.containers.denyHostNetwork`                  | Kapsayıcı destekli korumalı alan/tarayıcı ağ modu           | Ana makine ağ modunu reddedin.                                        |
| `sandbox.containers.denyContainerNamespaceJoin`       | Kapsayıcı destekli korumalı alan/tarayıcı ağ modu           | Başka bir kapsayıcının ağ ad alanına katılmayı reddedin.              |
| `sandbox.containers.requireReadOnlyMounts`            | Kapsayıcı destekli korumalı alan/tarayıcı bağlama modu             | Bağlamaların salt okunur olmasını zorunlu kılın.                                |
| `sandbox.containers.denyContainerRuntimeSocketMounts` | Kapsayıcı destekli korumalı alan/tarayıcı bağlama hedefleri          | Kapsayıcı çalışma zamanı yuvası bağlamalarını reddedin.                          |
| `sandbox.containers.denyUnconfinedProfiles`           | Kapsayıcı güvenlik profili duruşu                      | Sınırlandırılmamış kapsayıcı güvenlik profillerini reddedin.                   |
| `sandbox.browser.requireCdpSourceRange`               | Korumalı alan tarayıcısı CDP kaynak aralığı                        | Tarayıcı CDP erişiminin bir kaynak aralığı bildirmesini zorunlu kılın.        |

İlke, eksik `sandbox.mode` öğesini örtük varsayılanı olan `off` olarak değerlendirir; bu nedenle
`sandbox.requireMode`, yeni veya yapılandırılmamış bir korumalı alanı
`["all"]` gibi bir izin listesinin dışında bildirir.

#### Veri İşleme

| İlke alanı                                        | Gözlemlenen durum                                                                                     | Kullanılacağı durum                                                               |
| --------------------------------------------------- | -------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------- |
| `dataHandling.sensitiveLogging.requireRedaction`    | `logging.redactSensitive`                                                                          | `logging.redactSensitive: "off"` öğesini reddetmek için `true` olarak ayarlayın.              |
| `dataHandling.telemetry.denyContentCapture`         | `diagnostics.otel.captureContent`                                                                  | Telemetri içerik yakalamayı reddetmek için `true` olarak ayarlayın.                     |
| `dataHandling.retention.requireSessionMaintenance`  | `session.maintenance.mode`                                                                         | Etkin oturum bakım modu `enforce` öğesini zorunlu kılmak için `true` olarak ayarlayın. |
| `dataHandling.memory.denySessionTranscriptIndexing` | `memory.qmd.sessions.enabled`, `memory.search.experimental.sessionMemory` ve aracı başına geçersiz kılmalar | Oturum dökümü belleğe dizinlemeyi reddetmek için `true` olarak ayarlayın.       |

#### Gizli bilgiler

| İlke alanı                      | Gözlemlenen durum                                           | Kullanılacağı durum                                                                |
| --------------------------------- | -------------------------------------------------------- | ----------------------------------------------------------------------- |
| `secrets.requireManagedProviders` | Yapılandırma SecretRef'leri ve `secrets.providers.*` bildirimleri | SecretRef'lerin bildirilmiş sağlayıcılara işaret etmesini zorunlu kılmak için `true` olarak ayarlayın.     |
| `secrets.denySources`             | Gizli bilgi sağlayıcı kaynakları ve SecretRef kaynakları            | `exec`, `file` veya yapılandırılmış başka bir kaynak adı gibi kaynakları reddedin. |
| `secrets.allowInsecureProviders`  | Güvenli olmayan gizli bilgi sağlayıcı duruş bayrakları                   | Güvenli olmayan duruşu tercih eden sağlayıcıları reddetmek için `false` olarak ayarlayın.      |

#### Exec onayları

Exec onayı denetimleri çalışma zamanı `exec-approvals.json` yapıtını okur:
varsayılan olarak `~/.openclaw/exec-approvals.json` veya
`OPENCLAW_STATE_DIR` ayarlandığında `$OPENCLAW_STATE_DIR/exec-approvals.json`.
`execApprovals.defaults.*` veya `execApprovals.agents.*` altındaki duruş kuralları
okunabilir yapıt kanıtı gerektirir; eksik veya geçersiz bir yapıt, en iyi çabayla geçiş yerine
gözlemlenemeyen kanıt olarak bildirilir. Okunabilir olduğunda, belirtilmeyen
alanlar çalışma zamanı varsayılanlarını devralır: eksik `defaults.security`, `full` olur ve
eksik aracı güvenliği bu varsayılanı devralır. Kanıt; `defaults`,
`agents.*`, `agents.*.allowlist[].pattern`, isteğe bağlı `argPattern`, etkin
`autoAllowSkills` duruşu ve girdi kaynağını içerir; yuva yolu/belirteci,
`commandText`, `lastUsedCommand`, çözümlenmiş yollar veya zaman damgalarını asla içermez.

| İlke alanı                                  | Gözlemlenen durum                                                                       | Kullanım amacı                                                                            |
| ------------------------------------------- | -------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------- |
| `execApprovals.requireFile`                 | Etkin çalışma zamanı `exec-approvals.json` yolu                                              | Onaylar yapıtının mevcut olmasını ve ayrıştırılabilmesini zorunlu kılmak için `true` olarak ayarlayın.                     |
| `execApprovals.defaults.allowSecurity`      | `defaults.security`, varsayılan olarak `full`                                              | Yalnızca onaylanmış varsayılan onay güvenliği modlarına izin verin.                                    |
| `execApprovals.agents.allowSecurity`        | `agents.*.security`, varsayılanları devralır                                               | Yalnızca aracı başına onaylanmış etkin onay güvenliği modlarına izin verin.                        |
| `execApprovals.agents.allowAutoAllowSkills` | `defaults.autoAllowSkills` ve `agents.*.autoAllowSkills`, çalışma zamanı varsayılanlarını devralır | Örtük beceri CLI onayı olmadan katı manuel izin listelerini zorunlu kılmak için `false` olarak ayarlayın. |
| `execApprovals.agents.allowlist.expected`   | Toplu `agents.*.allowlist[]` kalıbı ve isteğe bağlı argPattern girdileri               | Onaylar izin listesinin incelenen kalıp kümesiyle eşleşmesini zorunlu kılın.                      |

Örnek: Onaylar yapıtını zorunlu kılın, esnek varsayılanları reddedin ve seçili
aracılar için yalnızca incelenmiş yürütme onayı duruşuna izin verin.

```jsonc
{
  "execApprovals": {
    "requireFile": true,
    "defaults": {
      // Güvenlik modları: "deny", "allowlist" veya "full".
      // Bu varsayılan yalnızca sıkı biçimde kısıtlanmış reddetme duruşuna izin verir.
      "allowSecurity": ["deny"],
    },
  },
  "scopes": {
    "restricted-shell": {
      "agentIds": ["family-agent", "groups-agent"],
      "execApprovals": {
        "agents": {
          // Seçili aracılar incelenmiş izin listesi duruşunu kullanabilir, ancak "full" kullanamaz.
          "allowSecurity": ["allowlist"],
          // false, beceri CLI'larının autoAllowSkills tarafından örtük olarak
          // onaylanmak yerine incelenen izin listesinde bulunması gerektiği anlamına gelir.
          "allowAutoAllowSkills": false,
          "allowlist": {
            "expected": [
              // Basit girdi: argPattern içermeyen, tam olarak incelenmiş yürütülebilir dosya kalıbı.
              "travel-hub",
              // Kısıtlı girdi: kalıp ve incelenmiş bağımsız değişken düzenli ifadesi.
              { "pattern": "calendar-cli", "argPattern": "^sync\\b" },
              "/bin/date",
            ],
          },
        },
      },
    },
  },
}
```

#### Kimlik doğrulama profilleri

| İlke alanı                      | Gözlemlenen durum                             | Kullanım amacı                                                                              |
| ------------------------------- | -------------------------------------------- | ------------------------------------------------------------------------------------------ |
| `auth.profiles.requireMetadata` | `auth.profiles.*` sağlayıcı ve mod meta verileri | Yapılandırma kimlik doğrulama profillerinde `provider` ve `mode` gibi meta veri anahtarlarını zorunlu kılın.               |
| `auth.profiles.allowModes`      | `auth.profiles.*.mode`                       | Yalnızca `api_key`, `aws-sdk`, `oauth` veya `token` gibi desteklenen kimlik doğrulama profili modlarına izin verin. |

#### Araç meta verileri

| İlke alanı              | Gözlemlenen durum                 | Kullanım amacı                                                                              |
| ----------------------- | -------------------------------- | ------------------------------------------------------------------------------------------ |
| `tools.requireMetadata` | Yönetilen `TOOLS.md` bildirimleri | Yönetilen araçların `risk`, `sensitivity` veya `owner` gibi meta veri anahtarlarını bildirmesini zorunlu kılın. |

#### Araç duruşu

| İlke alanı                      | Gözlemlenen durum                                            | Kullanım amacı                                                                                            |
| ------------------------------- | ----------------------------------------------------------- | -------------------------------------------------------------------------------------------------------- |
| `tools.profiles.allow`          | `tools.profile` ve `agents.entries.*.tools.profile`        | Yalnızca `minimal`, `messaging` veya `coding` gibi araç profili kimliklerine izin verin.                                 |
| `tools.fs.requireWorkspaceOnly` | `tools.fs.workspaceOnly` ve aracı başına `tools.fs` geçersiz kılmaları | Yalnızca çalışma alanıyla sınırlı dosya sistemi aracı duruşunu zorunlu kılmak için `true` olarak ayarlayın.                                         |
| `tools.exec.allowSecurity`      | `tools.exec.security` ve aracı başına yürütme güvenliği           | Yalnızca `deny` veya `allowlist` gibi yürütme güvenliği modlarına izin verin.                                            |
| `tools.exec.requireAsk`         | `tools.exec.ask` ve aracı başına yürütme sorma modu                | `always` gibi bir onay duruşunu zorunlu kılın.                                                               |
| `tools.exec.allowHosts`         | `tools.exec.host` ve aracı başına yürütme ana makine yönlendirmesi           | Yalnızca `sandbox` gibi yürütme ana makine yönlendirme modlarına izin verin.                                                    |
| `tools.elevated.allow`          | `tools.elevated.enabled` ve aracı başına yükseltilmiş duruş     | Yükseltilmiş araç modunun devre dışı kalmasını zorunlu kılmak için `false` olarak ayarlayın.                                           |
| `tools.alsoAllow.expected`      | `tools.alsoAllow` ve aracı başına `tools.alsoAllow`           | Tam `alsoAllow` girdilerini zorunlu kılın ve eksik ya da beklenmeyen ek araç izinlerini bildirin.                 |
| `tools.denyTools`               | `tools.deny` ve `agents.entries.*.tools.deny`              | Yapılandırılmış araç reddetme listelerinin `group:runtime` ve `group:fs` gibi araç kimliklerini veya gruplarını içermesini zorunlu kılın. |

## Denetimleri çalıştırma

Yazım sırasında yalnızca ilke denetimlerini çalıştırın:

```bash
openclaw policy check
openclaw policy check --json
openclaw policy check --severity-min error
```

`policy check` yalnızca ilke denetimi kümesini çalıştırır ve kanıtları, bulguları
ve doğrulama karmalarını yayımlar. Aynı bulgular, İlke Plugin'i
etkinleştirildiğinde `openclaw doctor --lint` içinde de görünür.

Bir operatör ilke dosyasını yazılmış bir temel değerle karşılaştırın:

```bash
openclaw policy compare --baseline official.policy.jsonc
openclaw policy compare --baseline official.policy.jsonc --policy policy.jsonc --json
```

`policy compare`, ilke dosyası söz dizimini ilke dosyası söz dizimine göre denetler;
çalışma zamanı durumunu, kanıtları, kimlik bilgilerini veya gizli değerleri incelemez. Kapsamlı
katmanları yöneten aynı kural meta verilerini kullanır: izin listeleri eşit veya
daha dar kalmalı, reddetme listeleri eşit veya daha geniş kalmalı, zorunlu Boole değerleri
değerlerini korumalı, sıralı dizeler yalnızca yapılandırılmış sıranın daha katı
ucuna doğru ilerleyebilmeli ve tam listeler eşleşmelidir. Temel değer, kuruluş
tarafından yazılmış bir ilke olabilir; denetlenen ilke daha katı değerler veya
ek kurallar ekleyebilir. Üst düzey bir denetlenen kural, eşit ya da daha kısıtlayıcı
olduğunda kapsamlı bir temel değer kuralını karşılayabilir. Kapsam adlarının dosyalar
arasında eşleşmesi gerekmez; karşılaştırma seçiciye (`agentIds`/`channelIds`) ve alana göre anahtarlanır.
Yönlendirme yoklamalarında, her temel değer yoklama kimliği aynı rota ve
beklenen aracıyla kalmalıdır. Denetlenen bir ilke yoklamalar ekleyebilir veya `matchedBy` değerini daraltabilir, ancak
bir yoklamayı kaldırmak, rotasını ya da aracısını değiştirmek veya kabul edilen eşleşme
türlerini genişletmek daha zayıftır.

Temiz karşılaştırma (`--json`):

```json
{
  "ok": true,
  "baselinePath": "official.policy.jsonc",
  "policyPath": "policy.jsonc",
  "rulesChecked": 3,
  "findings": []
}
```

Temiz `policy check --json` çıktısı, bir operatörün veya
denetleyicinin kaydedebileceği kararlı karmaları içerir:

```json
{
  "ok": true,
  "attestation": {
    "policy": {
      "path": "policy.jsonc",
      "hash": "sha256:..."
    },
    "workspace": {
      "scope": "policy",
      "hash": "sha256:..."
    },
    "findingsHash": "sha256:...",
    "attestationHash": "sha256:..."
  },
  "checksRun": 5,
  "checksSkipped": 0,
  "findings": []
}
```

## İlkeyi yapılandırma

İlke yapılandırması `plugins.entries.policy.config` altında bulunur.

```jsonc
{
  "plugins": {
    "entries": {
      "policy": {
        "enabled": true,
        "config": {
          "enabled": true,
          "path": "policy.jsonc",
          "workspaceRepairs": false,
          "expectedHash": "sha256:...",
          "expectedAttestationHash": "sha256:...",
        },
      },
    },
  },
}
```

| Ayar                      | Amaç                                                            |
| ------------------------- | --------------------------------------------------------------- |
| `enabled`                 | `policy.jsonc` mevcut olmadan önce bile ilke denetimlerini etkinleştirin.         |
| `workspaceRepairs`        | `doctor --fix` komutunun ilke tarafından yönetilen çalışma alanı ayarlarını düzenlemesine izin verin. |
| `expectedHash`            | Onaylanmış ilke yapıtı için isteğe bağlı karma kilidi.            |
| `expectedAttestationHash` | Son kabul edilen temiz ilke denetimi için isteğe bağlı karma kilidi.    |
| `path`                    | İlke yapıtının çalışma alanına göre konumu.             |

Plugin yüklü kalırken bir çalışma alanının ilke
denetimlerini devre dışı bırakmak için `plugins.entries.policy.config.enabled` değerini `false` olarak ayarlayın.

## İlke durumunu kabul etme

Örnek JSON çıktısı:

```json
{
  "ok": true,
  "attestation": {
    "checkedAt": "2026-05-10T20:00:00.000Z",
    "policy": {
      "path": "policy.jsonc",
      "hash": "sha256:..."
    },
    "workspace": {
      "scope": "policy",
      "hash": "sha256:..."
    },
    "findingsHash": "sha256:...",
    "attestationHash": "sha256:..."
  },
  "evidence": {
    "channels": [
      {
        "id": "telegram",
        "provider": "telegram",
        "source": "oc://openclaw.config/channels/telegram",
        "enabled": false
      }
    ],
    "mcpServers": [
      {
        "id": "docs",
        "transport": "stdio",
        "source": "oc://openclaw.config/mcp/servers/docs",
        "command": "npx"
      }
    ],
    "modelProviders": [
      {
        "id": "openai",
        "source": "oc://openclaw.config/models/providers/openai"
      }
    ],
    "modelRefs": [
      {
        "ref": "openai/gpt-5.6-sol",
        "provider": "openai",
        "model": "gpt-5.6-sol",
        "source": "oc://openclaw.config/agents/defaults/model"
      }
    ],
    "network": [
      {
        "id": "browser-private-network",
        "source": "oc://openclaw.config/browser/ssrfPolicy/dangerouslyAllowPrivateNetwork",
        "value": false
      }
    ],
    "gatewayExposure": [
      {
        "id": "gateway-bind",
        "kind": "bind",
        "source": "oc://openclaw.config/gateway/bind",
        "value": "loopback",
        "nonLoopback": false,
        "explicit": true
      }
    ],
    "agentWorkspace": [
      {
        "id": "agents-defaults-workspace-access",
        "kind": "workspaceAccess",
        "source": "oc://openclaw.config/agents/defaults/sandbox/workspaceAccess",
        "scope": "defaults",
        "value": "ro",
        "sandboxMode": "all",
        "sandboxModeSource": "oc://openclaw.config/agents/defaults/sandbox/mode",
        "sandboxEnabled": true,
        "explicit": true
      },
      {
        "id": "agents-defaults-tool-exec",
        "kind": "toolDeny",
        "source": "oc://openclaw.config/tools/deny",
        "scope": "defaults",
        "tool": "exec",
        "denied": true,
        "explicit": true
      }
    ],
    "secrets": [
      {
        "id": "vault",
        "kind": "provider",
        "source": "oc://openclaw.config/secrets/providers/vault",
        "providerSource": "env"
      },
      {
        "id": "oc://openclaw.config/models/providers/openai/apiKey",
        "kind": "input",
        "source": "oc://openclaw.config/models/providers/openai/apiKey",
        "provenance": "secretRef",
        "refSource": "env",
        "refProvider": "vault"
      }
    ],
    "authProfiles": [
      {
        "id": "github",
        "source": "oc://openclaw.config/auth/profiles/github",
        "validMetadata": true,
        "provider": "github",
        "mode": "token"
      }
    ],
    "tools": [
      {
        "id": "deploy",
        "source": "oc://TOOLS.md/tools/deploy",
        "line": 12,
        "risk": "critical",
        "sensitivity": "restricted",
        "capabilities": ["IRREVERSIBLE_EXTERNAL"]
      }
    ]
  },
  "checksRun": 30,
  "checksSkipped": 0,
  "findings": []
}
```

`attestation.policy.hash` yazılan kural yapıtını tanımlar. `evidence`
denetimlerin kullandığı gözlemlenen OpenClaw durumunu kaydeder ve
`workspace.hash` bu kanıt yükünü tanımlar. `findingsHash` kesin
bulgu kümesini tanımlar. `checkedAt` denetimin ne zaman çalıştığını kaydeder.
`attestationHash` kararlı iddiayı (ilke karması, kanıt karması,
bulgular karması ve temiz/kirli durumu) tanımlar ve `checkedAt` değerini
bilinçli olarak hariç tutar; böylece aynı ilke durumu her zaman aynı tasdik
karmasını üretir. Bu dört değer birlikte tek bir ilke denetiminin denetim
izini oluşturan dörtlüyü meydana getirir.

Bir Gateway veya denetleyici, bir çalışma zamanı eylemini engellemek, onaylamak
ya da açıklamayla işaretlemek için ilke kullanıyorsa son temiz denetimin tasdik
karmasını kaydetmelidir. `checkedAt` denetim günlükları için JSON çıktısında
kalır ancak kararlı karmanın parçası değildir.

İlke durumunu kabul etme yaşam döngüsü:

1. `policy.jsonc` öğesini yazın veya inceleyin.
2. `openclaw policy check --json` komutunu çalıştırın.
3. Temizse `attestation.policy.hash` değerini `expectedHash` olarak kaydedin.
4. `attestation.attestationHash` değerini `expectedAttestationHash` olarak kaydedin.
5. `openclaw doctor --lint` komutunu CI veya sürüm geçitlerinde yeniden çalıştırın.

İlke kuralları kasıtlı olarak değişirse temiz bir denetimdeki kabul edilmiş iki
karmayı da güncelleyin. Yalnızca çalışma alanı ayarları değişirse (ilke aynı
kalırsa), genellikle yalnızca `expectedAttestationHash` değişir.

`agents.workspace` kurallarını etkinleştirmek veya yükseltmek, çalışma alanı
karmasına ve tasdik karmasına `agentWorkspace` kanıtı ekler; etkinleştirdikten
sonra yeni kanıtı inceleyin ve kabul edilmiş tasdik karmalarını yenileyin.
Araç duruşu kurallarını etkinleştirmek veya yükseltmek de
`toolPosture` kanıtını aynı şekilde ekler.

`openclaw policy watch` denetimi yeniden çalıştırır ve mevcut kanıt artık
`expectedAttestationHash` ile eşleşmediğinde bunu bildirir:

```bash
openclaw policy watch --json
```

Tek bir sapma değerlendirmesine ihtiyaç duyan CI veya betiklerde
`--once` kullanın. `--once` olmadan varsayılan olarak her iki
saniyede bir yoklama yapar; aralığı değiştirmek için `--interval-ms` kullanın.

## Bulgular

| Denetim kimliği                                         | Bulgu                                                                             |
| -------------------------------------------------------- | --------------------------------------------------------------------------------- |
| `policy/policy-jsonc-missing`                            | İlke etkin ancak `policy.jsonc` eksik.                                            |
| `policy/policy-jsonc-invalid`                            | İlke ayrıştırılamıyor veya hatalı biçimlendirilmiş kural girdileri içeriyor.      |
| `policy/policy-hash-mismatch`                            | İlke, yapılandırılmış `expectedHash` ile eşleşmiyor.                               |
| `policy/attestation-hash-mismatch`                       | Mevcut ilke kanıtı artık kabul edilen tasdikle eşleşmiyor.                         |
| `policy/policy-conformance-invalid`                      | Bir temel veya denetlenen ilke dosyasında geçersiz karşılaştırma söz dizimi var.  |
| `policy/policy-conformance-missing`                      | Denetlenen bir ilke dosyasında, temel ilke dosyasının gerektirdiği bir kural eksik. |
| `policy/policy-conformance-weaker`                       | Denetlenen bir ilke dosyasında, temel ilke dosyasındakinden daha zayıf bir değer var. |
| `policy/channels-denied-provider`                        | Etkin bir kanal, kanal ret kuralıyla eşleşiyor.                                   |
| `policy/mcp-denied-server`                               | Yapılandırılmış bir MCP sunucusu ilke tarafından reddediliyor.                    |
| `policy/mcp-unapproved-server`                           | Yapılandırılmış bir MCP sunucusu izin listesinin dışında.                         |
| `policy/models-denied-provider`                          | Yapılandırılmış bir model sağlayıcısı veya model başvurusu, reddedilen bir sağlayıcı kullanıyor. |
| `policy/models-unapproved-provider`                      | Yapılandırılmış bir model sağlayıcısı veya model başvurusu izin listesinin dışında. |
| `policy/network-private-access-enabled`                  | İlke reddettiği hâlde özel ağ SSRF kaçış mekanizması etkin.                       |
| `policy/routing-bindings-required`                       | İlke bir kanal rota bağlaması gerektiriyor ancak hiçbiri yapılandırılmamış.       |
| `policy/routing-binding-channel-unconfigured`            | Bir rota bağlaması, `channels.*` içinde bulunmayan bir kanalı adlandırıyor.      |
| `policy/routing-agent-mismatch`                          | Yazılmış bir rota farklı bir ajana çözümleniyor.                                  |
| `policy/routing-match-kind-mismatch`                     | Yazılmış bir rota, beklenmeyen bir bağlama özgüllüğünde eşleşiyor.                |
| `policy/ingress-dm-policy-unapproved`                    | Bir kanal DM ilkesi, ilke izin listesinin dışında.                                |
| `policy/ingress-dm-scope-unapproved`                     | `session.dmScope`, ilkenin gerektirdiği DM yalıtım kapsamıyla eşleşmiyor.          |
| `policy/ingress-open-groups-denied`                      | İlke açık grup girişini reddettiği hâlde bir kanal grup ilkesi `open`.         |
| `policy/ingress-group-mention-required`                  | İlke gerektirdiği hâlde bir kanal veya grup girdisi bahsetme geçitlerini devre dışı bırakıyor. |
| `policy/gateway-non-loopback-bind`                       | İlke reddettiği hâlde Gateway bağlama duruşu, geri döngü dışı açığa çıkmaya izin veriyor. |
| `policy/gateway-auth-disabled`                           | İlke kimlik doğrulaması gerektirdiği hâlde Gateway kimlik doğrulaması devre dışı. |
| `policy/gateway-rate-limit-missing`                      | İlke gerektirdiği hâlde Gateway kimlik doğrulama hız sınırı duruşu açıkça belirtilmemiş. |
| `policy/gateway-control-ui-insecure`                     | Gateway Denetim Arayüzü'nün güvenli olmayan açığa çıkma geçişleri etkin.           |
| `policy/gateway-tailscale-funnel`                        | İlke reddettiği hâlde Gateway Tailscale Funnel açığa çıkması etkin.               |
| `policy/gateway-remote-enabled`                          | İlke reddettiği hâlde Gateway uzak modu etkin.                                    |
| `policy/gateway-http-endpoint-enabled`                   | İlke tarafından reddedilmesine rağmen bir Gateway HTTP API uç noktası etkin.     |
| `policy/gateway-http-url-fetch-unrestricted`             | Gateway HTTP URL getirme girdisinde gerekli URL izin listesi yok.                 |
| `policy/gateway-node-command-denied`                     | İlke tarafından reddedilen bir Node komutu, OpenClaw yapılandırması tarafından reddedilmiyor. |
| `policy/agents-workspace-access-denied`                  | Ajan korumalı alan modu veya çalışma alanı erişimi, ilke izin listesinin dışında. |
| `policy/agents-tool-not-denied`                          | Bir ajan veya varsayılan yapılandırma, ilkenin gerektirdiği bir aracı reddetmiyor. |
| `policy/tools-profile-unapproved`                        | Yapılandırılmış genel veya ajan bazlı araç profili izin listesinin dışında.       |
| `policy/tools-fs-workspace-only-required`                | Dosya sistemi araçları yalnızca çalışma alanı yolu duruşuyla yapılandırılmamış.    |
| `policy/tools-exec-security-unapproved`                  | Yürütme güvenlik modu, ilke izin listesinin dışında.                              |
| `policy/tools-exec-ask-unapproved`                       | Yürütme sorma modu, ilke izin listesinin dışında.                                 |
| `policy/tools-exec-host-unapproved`                      | Yürütme ana makine yönlendirmesi, ilke izin listesinin dışında.                   |
| `policy/tools-elevated-enabled`                          | İlke reddettiği hâlde yükseltilmiş araç modu etkin.                               |
| `policy/tools-also-allow-missing`                        | Yapılandırılmış bir `alsoAllow` listesinde ilkenin gerektirdiği bir girdi eksik. |
| `policy/tools-also-allow-unexpected`                     | Yapılandırılmış bir `alsoAllow` listesi, ilkenin beklemediği bir girdi içeriyor. |
| `policy/tools-required-deny-missing`                     | Genel veya ajan bazlı araç ret listesi, reddedilmesi gereken bir aracı içermiyor. |
| `policy/sandbox-mode-unapproved`                         | Korumalı alan modu, ilke izin listesinin dışında.                                 |
| `policy/sandbox-backend-unapproved`                      | Korumalı alan arka ucu, ilke izin listesinin dışında.                             |
| `policy/sandbox-container-posture-unobservable`          | Bir kapsayıcı duruş kuralı, bunu gözlemleyemeyen bir arka uç için etkinleştirilmiş. |
| `policy/sandbox-container-host-network-denied`           | Kapsayıcı destekli bir korumalı alan veya tarayıcı, ana makine ağ modunu kullanıyor. |
| `policy/sandbox-container-namespace-join-denied`         | Kapsayıcı destekli bir korumalı alan veya tarayıcı, başka bir kapsayıcının ad alanına katılıyor. |
| `policy/sandbox-container-mount-mode-required`           | Kapsayıcı destekli bir korumalı alan veya tarayıcı bağlaması salt okunur değil.   |
| `policy/sandbox-container-runtime-socket-mount`          | Kapsayıcı destekli bir korumalı alan veya tarayıcı bağlaması, kapsayıcı çalışma zamanı yuvasını açığa çıkarıyor. |
| `policy/sandbox-container-unconfined-profile`            | İlke reddettiği hâlde kapsayıcı korumalı alan profili sınırsız.                   |
| `policy/sandbox-browser-cdp-source-range-missing`        | İlke gerektirdiği hâlde korumalı alan tarayıcısının CDP kaynak aralığı eksik.     |
| `policy/data-handling-redaction-disabled`                | İlke gerektirdiği hâlde hassas günlük kaydı maskelemesi devre dışı.               |
| `policy/data-handling-telemetry-content-capture`         | İlke reddettiği hâlde telemetri içerik yakalama etkin.                            |
| `policy/data-handling-session-retention-not-enforced`    | İlke gerektirdiği hâlde oturum saklama bakımı zorunlu tutulmuyor.                  |
| `policy/data-handling-session-transcript-memory-enabled` | İlke reddettiği hâlde oturum dökümü bellek indeksleme etkin.                      |
| `policy/secrets-unmanaged-provider`                      | Bir yapılandırma SecretRef'i, `secrets.providers` altında bildirilmemiş bir sağlayıcıya başvuruyor. |
| `policy/secrets-denied-provider-source`                  | Bir yapılandırma gizli bilgi sağlayıcısı veya SecretRef, ilke tarafından reddedilen bir kaynak kullanıyor. |
| `policy/secrets-insecure-provider`                       | İlke reddettiği hâlde bir gizli bilgi sağlayıcısı güvenli olmayan duruşu kabul ediyor. |
| `policy/auth-profile-invalid-metadata`                   | Bir yapılandırma kimlik doğrulama profilinde geçerli sağlayıcı veya mod meta verileri eksik. |
| `policy/auth-profile-unapproved-mode`                    | Bir yapılandırma kimlik doğrulama profili modu, ilke izin listesinin dışında.     |
| `policy/exec-approvals-missing`                          | İlke `exec-approvals.json` gerektiriyor ancak yapıt eksik.                        |
| `policy/exec-approvals-invalid`                          | Yapılandırılmış yürütme onayları yapıtı ayrıştırılamıyor.                         |
| `policy/exec-approvals-default-security-unapproved`      | Yürütme onayı varsayılanları, ilke izin listesinin dışında bir güvenlik modu kullanıyor. |
| `policy/exec-approvals-agent-security-unapproved`        | Ajan bazlı etkin yürütme onayı güvenlik modu izin listesinin dışında.             |
| `policy/exec-approvals-auto-allow-skills-enabled`        | İlke reddettiği hâlde bir yürütme onayı ajanı, beceri CLI'larına örtük olarak otomatik izin veriyor. |
| `policy/exec-approvals-allowlist-missing`                | Onaylar izin listesinde ilkenin gerektirdiği bir örüntü eksik.                    |
| `policy/exec-approvals-allowlist-unexpected`             | Onaylar izin listesi, ilkenin beklemediği bir örüntü içeriyor.                    |
| `policy/tools-missing-risk-level`                        | Yönetilen bir araç bildiriminde risk meta verileri eksik.                        |
| `policy/tools-unknown-risk-level`                        | Yönetilen bir araç bildirimi, bilinmeyen bir risk değeri kullanıyor.              |
| `policy/tools-missing-sensitivity-token`                 | Yönetilen bir araç bildiriminde hassasiyet meta verileri eksik.                   |
| `policy/tools-missing-owner`                             | Yönetilen bir araç bildiriminde sahip meta verileri eksik.                        |
| `policy/tools-unknown-sensitivity-token`                 | Yönetilen bir araç bildirimi, bilinmeyen bir hassasiyet değeri kullanıyor.        |

Bir bulgu hem `target` (uyumsuzluk gösteren, gözlemlenen çalışma alanı
öğesi) hem de `requirement` (bunun bir bulgu olmasına neden olan yazılmış kural)
içerebilir. Her ikisi de bugün `oc://` adres dizeleridir ancak alan adları adres
biçimini değil, ilke rolünü tanımlar.

Örnek bulgular:

```json
{
  "checkId": "policy/channels-denied-provider",
  "severity": "error",
  "message": "'telegram' kanalı, reddedilen 'telegram' sağlayıcısını kullanıyor.",
  "source": "policy",
  "path": "openclaw yapılandırması",
  "ocPath": "oc://openclaw.config/channels/telegram",
  "target": "oc://openclaw.config/channels/telegram",
  "requirement": "oc://policy.jsonc/channels/denyRules/#0",
  "fixHint": "Telegram bu çalışma alanı için onaylanmamıştır."
}
```

```json
{
  "checkId": "policy/tools-missing-risk-level",
  "severity": "error",
  "message": "TOOLS.md içindeki 'deploy' aracının açık bir risk sınıflandırması yok.",
  "source": "policy",
  "path": "TOOLS.md",
  "line": 12,
  "ocPath": "oc://TOOLS.md/tools/deploy",
  "target": "oc://TOOLS.md/tools/deploy",
  "requirement": "oc://policy.jsonc/tools/requireMetadata"
}
```

```json
{
  "checkId": "policy/mcp-unapproved-server",
  "severity": "error",
  "message": "'remote' MCP sunucusu ilke izin listesinde değil.",
  "source": "policy",
  "path": "openclaw yapılandırması",
  "ocPath": "oc://openclaw.config/mcp/servers/remote",
  "target": "oc://openclaw.config/mcp/servers/remote",
  "requirement": "oc://policy.jsonc/mcp/servers/allow"
}
```

```json
{
  "checkId": "policy/models-unapproved-provider",
  "severity": "error",
  "message": "'anthropic/claude-sonnet-4.7' model başvurusu, onaylanmamış 'anthropic' sağlayıcısını kullanıyor.",
  "source": "policy",
  "path": "openclaw yapılandırması",
  "ocPath": "oc://openclaw.config/agents/defaults/model/fallbacks/#0",
  "target": "oc://openclaw.config/agents/defaults/model/fallbacks/#0",
  "requirement": "oc://policy.jsonc/models/providers/allow"
}
```

```json
{
  "checkId": "policy/network-private-access-enabled",
  "severity": "error",
  "message": "'browser-private-network' ağ ayarı, özel ağ erişimine izin veriyor.",
  "source": "policy",
  "path": "openclaw yapılandırması",
  "ocPath": "oc://openclaw.config/browser/ssrfPolicy/dangerouslyAllowPrivateNetwork",
  "target": "oc://openclaw.config/browser/ssrfPolicy/dangerouslyAllowPrivateNetwork",
  "requirement": "oc://policy.jsonc/network/privateNetwork/allow"
}
```

```json
{
  "checkId": "policy/gateway-non-loopback-bind",
  "severity": "error",
  "message": "Gateway bağlama ayarı 'gateway-bind', geri döngü dışı erişime izin veriyor.",
  "source": "policy",
  "path": "openclaw config",
  "ocPath": "oc://openclaw.config/gateway/bind",
  "target": "oc://openclaw.config/gateway/bind",
  "requirement": "oc://policy.jsonc/gateway/exposure/allowNonLoopbackBind"
}
```

```json
{
  "checkId": "policy/gateway-node-command-denied",
  "severity": "error",
  "message": "Gateway node komutu 'system.run' ilke tarafından reddediliyor ancak OpenClaw yapılandırması tarafından reddedilmiyor.",
  "source": "policy",
  "path": "openclaw config",
  "ocPath": "oc://openclaw.config/gateway/nodes/commands/deny",
  "target": "oc://openclaw.config/gateway/nodes/commands/deny",
  "requirement": "oc://policy.jsonc/gateway/nodes/denyCommands",
  "fixHint": "'system.run' değerini gateway.nodes.commands.deny listesine ekleyin veya incelemeden sonra ilkeyi güncelleyin."
}
```

```json
{
  "checkId": "policy/agents-workspace-access-denied",
  "severity": "error",
  "message": "agents.defaults sandbox workspaceAccess değeri 'rw' ilke tarafından izin verilmiyor.",
  "source": "policy",
  "path": "openclaw config",
  "ocPath": "oc://openclaw.config/agents/defaults/sandbox/workspaceAccess",
  "target": "oc://openclaw.config/agents/defaults/sandbox/workspaceAccess",
  "requirement": "oc://policy.jsonc/agents/workspace/allowedAccess"
}
```

## Onarım

`doctor --lint` ve `policy check` salt okunurdur.

`doctor --fix`, yalnızca
`workspaceRepairs` açıkça etkinleştirildiğinde ilke tarafından yönetilen çalışma alanı ayarlarını düzenler; aksi takdirde denetimler neleri
onaracaklarını bildirir ve ayarları değiştirmeden bırakır.

Bu sürümde onarım, `channels.denyRules` tarafından reddedilen kanalları devre dışı bırakabilir ve
aşağıda listelenen otomatik daraltma onarımlarını uygulayabilir. Geçerli bir kural
çalışma alanı yapılandırmasını değiştirebileceğinden, `workspaceRepairs` seçeneğini yalnızca
ilke dosyası incelendikten sonra etkinleştirin:

- genel bir ilke yükseltilmiş araçları yasakladığında `tools.elevated.enabled=false` değerini ayarla
- ilke bu araçların reddedilmesini gerektirdiğinde eksik zorunlu ret araç kimliklerini `tools.deny` veya
  `agents.entries.*.tools.deny` öğesine ekle
- güvenli olmayan `gateway.controlUi.*` anahtarlarını `false` olarak ayarla
- ilke uzak Gateway modunu reddettiğinde `gateway.mode=local` değerini ayarla
- ilke Gateway HTTP API uç noktalarını reddettiğinde bildirilen `gateway.http.endpoints.*.enabled` yollarını `false` olarak ayarla
- ilke açık grup girişini reddettiğinde bildirilen kanal girişi `groupPolicy` yollarını `allowlist` olarak ayarla
- ilke grup bahsetmelerini gerektirdiğinde bildirilen kanal girişi `requireMention` yollarını `true` olarak ayarla
- ilke hassas günlük kaydı
  redaksiyonunu gerektirdiğinde `logging.redactSensitive=tools` değerini ayarla
- ilke telemetri içeriği yakalamayı reddettiğinde `diagnostics.otel.captureContent=false` değerini veya
  nesne biçimindeki telemetri yakalama ayarları için
  `diagnostics.otel.captureContent.enabled=false` değerini ayarla

Kapsamlı yükseltilmiş araç onarımları yalnızca algılama amaçlıdır. Bulgu, paylaşılan
günlük kaydı veya telemetri yapılandırmasını bildirdiğinde kapsamlı veri işleme onarımları da
atlanır; çünkü paylaşılan ayarın değiştirilmesi, kapsamlı ilke hedefinden
daha fazlasını etkiler.

Bulgu devralınan kök `tools.deny` değerini bildirdiğinde kapsamlı zorunlu ret onarımları
atlanır; çünkü gerekli aracın kök yapılandırmasına eklenmesi,
kapsamlı ilke hedefinden daha fazlasını etkiler. Aracıya özgü zorunlu ret onarımları,
bildirilen `agents.entries.*.tools.deny` yolunu güncelleyebilir.

Bulgu devralınan `channels.defaults.*` değerini bildirdiğinde kapsamlı kanal girişi onarımları
atlanır; çünkü paylaşılan kanal varsayılanının değiştirilmesi,
kapsamlı ilke hedefinden daha fazlasını etkiler. Otomatik onarım doğru uç nokta URL
izin listesi değerlerini seçemediğinden, Gateway HTTP URL getirme izin listesi bulguları
manuel olarak kalır.

Gateway bağlama ve node komutu bulguları inceleme gerektirmeye devam eder.
`policy/gateway-non-loopback-bind` veya `policy/gateway-node-command-denied`
bir yapılandırma yoluyla eşleştirilebildiğinde, `doctor --fix` önerilen
`gateway.bind` veya `gateway.nodes.commands.deny` değişikliğini atlanmış önizleme
rehberliği olarak bildirir. Değişikliği uygulamaz ve bir operatör yapılandırmayı
veya ilkeyi inceleyip güncelleyene kadar bulgu onarılmış sayılmaz.

```jsonc
{
  "plugins": {
    "entries": {
      "policy": {
        "config": {
          "workspaceRepairs": true,
        },
      },
    },
  },
}
```

## Çıkış kodları

| Komut          | `0`                                                    | `1`                                                                 | `2`                          |
| ---------------- | ------------------------------------------------------ | ------------------------------------------------------------------- | ---------------------------- |
| `policy check`   | Eşikte bulgu yok.                          | Bir veya daha fazla bulgu eşiği karşıladı.                             | Bağımsız değişken veya çalışma zamanı hatası. |
| `policy compare` | İlke dosyası en az temel çizgi kadar katıdır. | İlke dosyası geçersiz, eksik veya temel çizgi kurallarından daha zayıf. | Bağımsız değişken veya çalışma zamanı hatası. |
| `policy watch`   | Bulgu yok ve kabul edilen karma güncel.              | Bulgular mevcut veya kabul edilen tasdik güncel değil.                    | Bağımsız değişken veya çalışma zamanı hatası. |

## İlgili

- [Doctor lint modu](/tr/cli/doctor#lint-mode)
- [Yol CLI'si](/tr/cli/path)
