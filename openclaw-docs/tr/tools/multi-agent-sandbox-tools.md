---
read_when: You want per-agent sandboxing or per-agent tool allow/deny policies in a multi-agent gateway.
sidebarTitle: Multi-agent sandbox and tools
status: active
summary: Ajan başına sandbox ve araç kısıtlamaları, öncelik sırası ve örnekler
title: Çoklu ajan korumalı alanı ve araçları
x-i18n:
    generated_at: "2026-07-26T23:06:25Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 0e07d07c30b844be1e1d93db62fcdaab72c47a5248367559642a959bf09ad193
    source_path: tools/multi-agent-sandbox-tools.md
    workflow: 16
---

Çok aracılı bir kurulumdaki her aracı, genel sandbox ve araç politikasını geçersiz kılabilir. Bu sayfa aracı başına yapılandırmayı, öncelik kurallarını ve örnekleri kapsar.

<CardGroup cols={3}>
  <Card title="Sandbox kullanımı" href="/tr/gateway/sandboxing">
    Arka uçlar ve modlar — eksiksiz sandbox başvurusu.
  </Card>
  <Card title="Sandbox, araç politikası ve elevated karşılaştırması" href="/tr/gateway/sandbox-vs-tool-policy-vs-elevated">
    "Bu neden engellendi?" sorununu giderin.
  </Card>
  <Card title="Elevated modu" href="/tr/tools/elevated">
    Güvenilir göndericiler için elevated çalıştırma.
  </Card>
</CardGroup>

<Warning>
Kimlik doğrulama aracı kapsamında tutulur: her aracının `~/.openclaw/agents/<agentId>/agent/openclaw-agent.sqlite` içinde kendi `agentDir` kimlik doğrulama deposu vardır. `agentDir` öğesini aracılar arasında asla yeniden kullanmayın. Aracılar yerel bir profile sahip olmadığında varsayılan/ana aracının kimlik doğrulama profillerini okuyabilir, ancak OAuth yenileme token'ları ikincil aracı depolarına kopyalanmaz. Kimlik bilgilerini elle kopyalarsanız yalnızca taşınabilir statik `api_key` veya `token` profillerini kopyalayın.
</Warning>

---

## Yapılandırma örnekleri

<AccordionGroup>
  <Accordion title="Örnek 1: Kişisel aracı + kısıtlı aile aracısı">
    ```json
    {
      "agents": {
        "list": [
          {
            "id": "main",
            "default": true,
            "name": "Personal Assistant",
            "workspace": "~/.openclaw/workspace",
            "sandbox": { "mode": "off" }
          },
          {
            "id": "family",
            "name": "Family Bot",
            "workspace": "~/.openclaw/workspace-family",
            "sandbox": {
              "mode": "all",
              "scope": "agent"
            },
            "tools": {
              "allow": ["read", "message"],
              "deny": ["exec", "write", "edit", "apply_patch", "process", "browser"],
              "message": {
                "crossContext": {
                  "allowWithinProvider": false,
                  "allowAcrossProviders": false
                }
              }
            }
          }
        ]
      },
      "bindings": [
        {
          "agentId": "family",
          "match": {
            "provider": "whatsapp",
            "accountId": "*",
            "peer": {
              "kind": "group",
              "id": "120363424282127706@g.us"
            }
          }
        }
      ]
    }
    ```

    **Sonuç:**

    - `main` aracısı: ana sistemde çalışır, tüm araçlara erişebilir.
    - `family` aracısı: Docker'da çalışır (aracı başına bir konteyner), yalnızca `read` ve mevcut konuşmaya mesaj gönderimlerine erişebilir.

  </Accordion>
  <Accordion title="Örnek 2: Paylaşılan sandbox kullanan iş aracısı">
    ```json
    {
      "agents": {
        "list": [
          {
            "id": "personal",
            "workspace": "~/.openclaw/workspace-personal",
            "sandbox": { "mode": "off" }
          },
          {
            "id": "work",
            "workspace": "~/.openclaw/workspace-work",
            "sandbox": {
              "mode": "all",
              "scope": "shared",
              "workspaceRoot": "/tmp/work-sandboxes"
            },
            "tools": {
              "allow": ["read", "write", "apply_patch", "exec"],
              "deny": ["browser", "gateway", "discord"]
            }
          }
        ]
      }
    }
    ```
  </Accordion>
  <Accordion title="Örnek 2b: Genel kodlama profili + yalnızca mesajlaşma aracısı">
    ```json
    {
      "tools": { "profile": "coding" },
      "agents": {
        "list": [
          {
            "id": "support",
            "tools": { "profile": "messaging", "allow": ["slack"] }
          }
        ]
      }
    }
    ```

    **Sonuç:**

    - varsayılan aracılar kodlama araçlarını alır.
    - `support` aracısı yalnızca mesajlaşma içindir (+ Slack aracı).

  </Accordion>
  <Accordion title="Örnek 3: Aracı başına farklı sandbox modları">
    ```json
    {
      "agents": {
        "defaults": {
          "sandbox": {
            "mode": "non-main",
            "scope": "session"
          }
        },
        "list": [
          {
            "id": "main",
            "workspace": "~/.openclaw/workspace",
            "sandbox": {
              "mode": "off"
            }
          },
          {
            "id": "public",
            "workspace": "~/.openclaw/workspace-public",
            "sandbox": {
              "mode": "all",
              "scope": "agent"
            },
            "tools": {
              "allow": ["read"],
              "deny": ["exec", "write", "edit", "apply_patch"]
            }
          }
        ]
      }
    }
    ```
  </Accordion>
</AccordionGroup>

---

## Yapılandırma önceliği

Hem genel (`agents.defaults.*`) hem de aracıya özgü (`agents.entries.*.*`) yapılandırmalar mevcut olduğunda:

### Sandbox yapılandırması

Aracıya özgü ayarlar genel ayarları geçersiz kılar:

```text
agents.entries.*.sandbox.mode > agents.defaults.sandbox.mode
agents.entries.*.sandbox.scope > agents.defaults.sandbox.scope
agents.entries.*.sandbox.workspaceRoot > agents.defaults.sandbox.workspaceRoot
agents.entries.*.sandbox.workspaceAccess > agents.defaults.sandbox.workspaceAccess
agents.entries.*.sandbox.docker.* > agents.defaults.sandbox.docker.*
agents.entries.*.sandbox.browser.* > agents.defaults.sandbox.browser.*
agents.entries.*.sandbox.prune.* > agents.defaults.sandbox.prune.*
```

<Note>
`agents.entries.*.sandbox.{docker,browser,prune}.*`, söz konusu aracı için `agents.defaults.sandbox.{docker,browser,prune}.*` değerini geçersiz kılar (sandbox kapsamı `"shared"` olarak çözümlendiğinde yok sayılır).
</Note>

### Araç kısıtlamaları

Filtreleme sırası şöyledir:

<Steps>
  <Step title="Araç profili">
    `tools.profile` veya `agents.entries.*.tools.profile`.
  </Step>
  <Step title="Sağlayıcı araç profili">
    `tools.byProvider[provider].profile` veya `agents.entries.*.tools.byProvider[provider].profile`.
  </Step>
  <Step title="Genel araç politikası">
    `tools.allow` / `tools.deny`.
  </Step>
  <Step title="Sağlayıcı araç politikası">
    `tools.byProvider[provider].allow/deny`.
  </Step>
  <Step title="Aracıya özgü araç politikası">
    `agents.entries.*.tools.allow/deny`.
  </Step>
  <Step title="Aracı sağlayıcı politikası">
    `agents.entries.*.tools.byProvider[provider].allow/deny`.
  </Step>
  <Step title="Sandbox araç politikası">
    `tools.sandbox.tools` veya `agents.entries.*.tools.sandbox.tools`.
  </Step>
  <Step title="Alt aracı araç politikası">
    Uygunsa `tools.subagents.tools`.
  </Step>
</Steps>

<AccordionGroup>
  <Accordion title="Öncelik kuralları">
    - Her düzey araçları daha fazla kısıtlayabilir, ancak önceki düzeylerde reddedilen araçlara yeniden izin veremez.
    - `agents.entries.*.tools.sandbox.tools` ayarlanmışsa söz konusu aracı için `tools.sandbox.tools` değerinin yerini alır.
    - `agents.entries.*.tools.profile` ayarlanmışsa söz konusu aracı için `tools.profile` değerini geçersiz kılar.
    - Sağlayıcı araç anahtarları, `provider` (ör. `google-antigravity`) veya `provider/model` (ör. `openai/gpt-5.4`) kabul eder.

  </Accordion>
  <Accordion title="Boş izin listesi davranışı">
    Bu zincirdeki herhangi bir açık izin listesi çalıştırmayı çağrılabilir araç olmadan bırakırsa OpenClaw, istemi modele göndermeden önce durur. Bu davranış bilinçlidir: `agents.entries.*.tools.allow: ["query_db"]` gibi eksik bir araçla yapılandırılan aracı, `query_db` öğesini kaydeden Plugin etkinleştirilene kadar belirgin biçimde başarısız olmalı; yalnızca metin aracısı olarak devam etmemelidir.
  </Accordion>
</AccordionGroup>

Araç politikaları, birden fazla araca genişletilen `group:*` kısaltmalarını destekler. Tam liste için [Araç grupları](/tr/gateway/sandbox-vs-tool-policy-vs-elevated#tool-groups-shorthands) bölümüne bakın.

Aracı başına elevated geçersiz kılmaları (`agents.entries.*.tools.elevated`), belirli aracılar için elevated çalıştırmayı daha fazla kısıtlayabilir. Ayrıntılar için [Elevated modu](/tr/tools/elevated) bölümüne bakın.

---

## Tek aracıdan geçiş

<Tabs>
  <Tab title="Önce (tek aracı)">
    ```json
    {
      "agents": {
        "defaults": {
          "workspace": "~/.openclaw/workspace",
          "sandbox": {
            "mode": "non-main"
          }
        }
      },
      "tools": {
        "sandbox": {
          "tools": {
            "allow": ["read", "write", "apply_patch", "exec"],
            "deny": []
          }
        }
      }
    }
    ```
  </Tab>
  <Tab title="Sonra (çok aracılı)">
    ```json
    {
      "agents": {
        "list": [
          {
            "id": "main",
            "default": true,
            "workspace": "~/.openclaw/workspace",
            "sandbox": { "mode": "off" }
          }
        ]
      }
    }
    ```
  </Tab>
</Tabs>

<Note>
Eski `agents.defaults.*`/`agents.entries.*.*` yapılandırma anahtarları (`sandbox.perSession`, `agentRuntime`, `embeddedPi` gibi) `openclaw doctor` tarafından taşınır; bundan sonra `agents.defaults` + `agents.entries` kullanmayı tercih edin.
</Note>

---

## Araç kısıtlama örnekleri

<Tabs>
  <Tab title="Salt okunur aracı">
    ```json
    {
      "tools": {
        "allow": ["read"],
        "deny": ["exec", "write", "edit", "apply_patch", "process"]
      }
    }
    ```
  </Tab>
  <Tab title="Dosya sistemi araçları devre dışıyken kabuk çalıştırma">
    ```json
    {
      "tools": {
        "allow": ["read", "exec", "process"],
        "deny": ["write", "edit", "apply_patch", "browser", "gateway"]
      }
    }
    ```

    <Warning>
    Bu politika OpenClaw dosya sistemi araçlarını devre dışı bırakır, ancak `exec` yine de bir kabuktur ve seçilen ana sistemin veya sandbox dosya sisteminin izin verdiği her yere dosya yazabilir. Salt okunur bir aracı için `exec` ve `process` öğelerini reddedin ya da kabuk erişimini `agents.defaults.sandbox.workspaceAccess: "ro"` veya `"none"` gibi sandbox dosya sistemi denetimleriyle birleştirin.
    </Warning>

  </Tab>
  <Tab title="Yalnızca iletişim">
    ```json
    {
      "tools": {
        "sessions": { "visibility": "tree" },
        "allow": ["sessions_list", "sessions_send", "sessions_history", "session_status"],
        "deny": ["exec", "write", "edit", "apply_patch", "read", "browser"]
      }
    }
    ```

    Bu profildeki `sessions_history`, ham bir transkript dökümü yerine yine de sınırlandırılmış ve arındırılmış bir hatırlama görünümü döndürür. Asistan hatırlaması; düşünme etiketlerini, `<relevant-memories>` iskeletini, düz metin araç çağrısı XML yüklerini (`<tool_call>...</tool_call>`, `<function_call>...</function_call>`, `<tool_calls>...</tool_calls>`, `<function_calls>...</function_calls>` ve kesilmiş araç çağrısı blokları dâhil), indirgenmiş araç çağrısı iskeletini, sızdırılmış ASCII/tam genişlikli model kontrol token'larını ve hatalı MiniMax araç çağrısı XML'ini redaksiyon/kısaltma işleminden önce kaldırır.

  </Tab>
</Tabs>

---

## Yaygın tuzak: "non-main"

<Warning>
`agents.defaults.sandbox.mode: "non-main"`, aracı kimliğini değil oturum anahtarını ana oturum anahtarıyla karşılaştırır (her zaman `"main"`; `session.mainKey` kullanıcı tarafından yapılandırılamaz ve OpenClaw diğer tüm değerler için uyarı verip bunları yok sayar). Grup/kanal oturumları her zaman kendi anahtarlarını alır; bu nedenle ana olmayan oturumlar olarak değerlendirilir ve sandbox içine alınır. Bir aracının hiçbir zaman sandbox içine alınmamasını istiyorsanız `agents.entries.*.sandbox.mode: "off"` ayarını yapın.
</Warning>

---

## Test

Çok aracılı sandbox ve araçları yapılandırdıktan sonra:

<Steps>
  <Step title="Aracı çözümlemesini denetleyin">
    ```bash
    openclaw agents list --bindings
    ```
  </Step>
  <Step title="Sandbox konteynerlerini doğrulayın">
    ```bash
    docker ps --filter "name=openclaw-sbx-"
    ```
  </Step>
  <Step title="Araç kısıtlamalarını test edin">
    - Kısıtlanmış araçları gerektiren bir mesaj gönderin.
    - Aracının reddedilen araçları kullanamadığını doğrulayın.

  </Step>
  <Step title="Günlükleri izleyin">
    ```bash
    openclaw logs --follow | grep -E "routing|sandbox|tools"
    ```
  </Step>
</Steps>

---

## Sorun giderme

<AccordionGroup>
  <Accordion title="`mode: 'all'` değerine rağmen aracı sandbox içine alınmıyor">
    - Bunu geçersiz kılan genel bir `agents.defaults.sandbox.mode` olup olmadığını denetleyin.
    - Aracıya özgü yapılandırma önceliklidir; bu nedenle `agents.entries.*.sandbox.mode: "all"` ayarını yapın.

  </Accordion>
  <Accordion title="Engelleme listesine rağmen hâlâ kullanılabilen araçlar">
    - [Tam filtreleme sırasını](#tool-restrictions) kontrol edin: profil → sağlayıcı profili → genel politika → sağlayıcı politikası → aracı politikası → aracı sağlayıcı politikası → korumalı alan → alt aracı.
    - Her düzey yalnızca daha fazla kısıtlama uygulayabilir; erişimi yeniden veremez.
    - Adım adım hata ayıklama için [Korumalı alan, araç politikası ve yükseltilmiş mod karşılaştırması](/tr/gateway/sandbox-vs-tool-policy-vs-elevated) bölümüne bakın.

  </Accordion>
  <Accordion title="Kapsayıcı aracı başına yalıtılmıyor">
    - Varsayılan `scope`, `"agent"` şeklindedir (her aracı kimliği için bir kapsayıcı).
    - Her oturum için bir kapsayıcı kullanmak üzere `scope: "session"`, aracılar arasında tek bir kapsayıcıyı yeniden kullanmak için ise `scope: "shared"` değerini ayarlayın.

  </Accordion>
</AccordionGroup>

---

## İlgili

- [Yükseltilmiş mod](/tr/tools/elevated)
- [Çok aracılı yönlendirme](/tr/concepts/multi-agent)
- [Korumalı alan yapılandırması](/tr/gateway/config-agents#agentsdefaultssandbox)
- [Korumalı alan, araç politikası ve yükseltilmiş mod karşılaştırması](/tr/gateway/sandbox-vs-tool-policy-vs-elevated) — "bu neden engellendi?" hata ayıklaması
- [Korumalı alan kullanımı](/tr/gateway/sandboxing) — tam korumalı alan başvurusu (modlar, kapsamlar, arka uçlar, kalıplar)
- [Oturum yönetimi](/tr/concepts/session)
