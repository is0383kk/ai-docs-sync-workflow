---
read_when: You want a dedicated explanation of sandboxing or need to tune agents.defaults.sandbox.
sidebarTitle: Sandboxing
status: active
summary: 'OpenClaw korumalı alanının çalışma biçimi: modlar, kapsamlar, çalışma alanı erişimi ve imajlar'
title: Korumalı Alan Kullanımı
x-i18n:
    generated_at: "2026-07-26T23:21:18Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: a3668dc512a8ff30732290ee68e9dd29a3a2e9c106e6e39077a97bfbd90098f7
    source_path: gateway/sandboxing.md
    workflow: 16
---

OpenClaw, etki alanını azaltmak için araç yürütmeyi bir korumalı alan arka ucunda çalıştırabilir. Korumalı alan varsayılan olarak kapalıdır ve `agents.defaults.sandbox` (genel) veya `agents.entries.*.sandbox` (ajan başına) tarafından denetlenir. Gateway işlemi her zaman ana sistemde kalır; etkinleştirildiğinde yalnızca araç yürütme korumalı alana taşınır.

<Note>
Bu kusursuz bir güvenlik sınırı değildir ancak model saçma bir şey yaptığında dosya sistemi ve işlem erişimini önemli ölçüde sınırlar.
</Note>

## Korumalı alanda çalıştırılanlar

- Araç yürütme: `exec`, `read`, `write`, `edit`, `apply_patch`, `process` vb.
- İsteğe bağlı korumalı alan tarayıcısı (`agents.defaults.sandbox.browser`).

Korumalı alanda çalıştırılmayanlar:

- Gateway işleminin kendisi.
- `tools.elevated` aracılığıyla korumalı alan dışında çalışmasına açıkça izin verilen tüm araçlar. Yükseltilmiş yürütme, korumalı alanı atlar ve yapılandırılmış çıkış yolunda çalışır (varsayılan olarak `gateway`; yürütme hedefi `node` olduğunda ise `node`). Korumalı alan kapalıysa yürütme zaten ana sistemde çalıştığından `tools.elevated` hiçbir şeyi değiştirmez. Bkz. [Yükseltilmiş Mod](/tr/tools/elevated).

## Modlar, kapsam ve arka uç

Üç bağımsız ayar korumalı alan davranışını denetler:

| Ayar    | Anahtar                           | Değerler                     | Varsayılan |
| ------- | --------------------------------- | ---------------------------- | ---------- |
| Mod     | `agents.defaults.sandbox.mode`    | `off`, `non-main`, `all`     | `off`    |
| Kapsam  | `agents.defaults.sandbox.scope`   | `agent`, `session`, `shared` | `agent`  |
| Arka uç | `agents.defaults.sandbox.backend` | `docker`, `ssh`, `openshell` | `docker` |

**Mod**, korumalı alanın ne zaman uygulanacağını denetler:

- `off`: korumalı alan kullanılmaz.
- `non-main`: ajanın ana oturumu dışındaki tüm oturumları korumalı alanda çalıştırır. Ana oturum anahtarı her zaman `agent:<agentId>:main` değeridir (`session.scope`, `"global"` olduğunda ise `global`); yapılandırılamaz. Grup/kanal oturumları kendi anahtarlarını kullanır; bu nedenle her zaman ana olmayan oturum sayılır ve korumalı alanda çalıştırılır.
- `all`: her oturum bir korumalı alanda çalışır.

**Kapsam**, kaç kapsayıcı/ortam oluşturulacağını denetler:

- `agent`: ajan başına bir kapsayıcı.
- `session`: oturum başına bir kapsayıcı.
- `shared`: korumalı alandaki tüm oturumların paylaştığı tek bir kapsayıcı (bu kapsam altında ajan başına `docker`/`ssh`/`browser` geçersiz kılmaları yok sayılır).

**Arka uç**, korumalı alandaki araçları hangi çalışma zamanının yürüteceğini denetler. SSH'ye özgü yapılandırma `agents.defaults.sandbox.ssh` altında, OpenShell'e özgü yapılandırma ise `plugins.entries.openshell.config` altında bulunur.

|                     | Docker                           | SSH                            | OpenShell                                           |
| ------------------- | -------------------------------- | ------------------------------ | --------------------------------------------------- |
| **Çalıştığı yer**   | Yerel kapsayıcı                  | SSH ile erişilebilen herhangi bir ana sistem | OpenShell tarafından yönetilen korumalı alan |
| **Kurulum**         | `scripts/sandbox-setup.sh`       | SSH anahtarı + hedef ana sistem | OpenShell plugin'i etkin |
| **Çalışma alanı modeli** | Bağlama veya kopyalama      | Uzak sistemin esas alındığı (bir kez başlangıç verileriyle doldurulur) | `mirror` veya `remote` |
| **Ağ denetimi**     | `docker.network` (varsayılan: yok) | Uzak ana sisteme bağlı | OpenShell'e bağlı |
| **Tarayıcı korumalı alanı** | Desteklenir              | Desteklenmez                   | Henüz desteklenmiyor |
| **Bağlama noktaları** | `docker.binds`             | Yok                            | Yok |
| **En uygun olduğu durumlar** | Yerel geliştirme, tam yalıtım | İş yükünü uzak bir makineye aktarma | İsteğe bağlı çift yönlü eşitlemeye sahip yönetilen uzak korumalı alanlar |

## Docker arka ucu

Korumalı alan etkinleştirildiğinde varsayılan arka uç Docker'dır. Araçları ve korumalı alan tarayıcılarını Docker daemon soketi (`/var/run/docker.sock`) üzerinden yerel olarak çalıştırır; yalıtım Docker ad alanlarıyla sağlanır.

Varsayılanlar: `network: "none"` (dışarı giden bağlantı yok), `readOnlyRoot: true`, `capDrop: ["ALL"]`, imaj `openclaw-sandbox:bookworm-slim`.

Ana sistem GPU'larını kullanıma açmak için `agents.defaults.sandbox.docker.gpus` değerini (veya ajan başına geçersiz kılmayı) `"all"` ya da `"device=GPU-uuid"` gibi bir değere ayarlayın. Bu değer Docker'ın `--gpus` bayrağına aktarılır ve NVIDIA Container Toolkit gibi uyumlu bir ana sistem çalışma zamanı gerektirir.

<Warning>
**Docker dışından Docker (DooD) kısıtlamaları**

OpenClaw Gateway'i Docker kapsayıcısı olarak dağıtırsanız ana sistemin Docker soketini kullanarak eşdüzey korumalı alan kapsayıcılarını yönetir (DooD). Bu, bir yol eşleme kısıtlaması getirir:

- **Yapılandırma ana sistem yollarını gerektirir**: `openclaw.json` `workspace`, dahili Gateway kapsayıcı yolunu değil, **ana sistemin mutlak yolunu** (ör. `/home/user/.openclaw/workspaces`) içermelidir. Docker daemon, yolları Gateway'in kendi ad alanına göre değil, ana sistem işletim sistemi ad alanına göre değerlendirir.
- **Eşleşen birim eşlemesi gerekir**: Gateway işlemi, heartbeat ve köprü dosyalarını da bu `workspace` yoluna yazar. Aynı ana sistem yolunun Gateway kapsayıcısının içinden de doğru çözümlenmesi için Gateway kapsayıcısına aynı birim eşlemesini (`-v /home/user/.openclaw:/home/user/.openclaw`) verin. Eşleşmeyen eşlemeler, Gateway heartbeat bilgisini yazmaya çalıştığında `EACCES` olarak görünür.
- **Codex kod modu**: Bir OpenClaw korumalı alanı etkinken OpenClaw, korumalı alan araç ilkesi gerekli araçları kullanıma sunmadığı ve deneysel korumalı alan yürütme sunucusu yolunu seçmediğiniz sürece, o tur için Codex app-server'ın yerel Code Mode özelliğini, kullanıcı MCP sunucularını ve uygulama destekli plugin yürütmesini devre dışı bırakır (bunlar OpenClaw korumalı alan arka ucundan değil, Gateway ana sistemindeki app-server işleminden çalışır). Kabuk erişimi daha sonra `sandbox_exec` ve `sandbox_process` gibi OpenClaw korumalı alanı destekli araçlar üzerinden yönlendirilir. Ana sistem Docker soketini ajan korumalı alan kapsayıcılarına veya özel Codex korumalı alanlarına bağlamayın. Tam davranış için [Codex Harness](/tr/plugins/codex-harness) bölümüne bakın.

Docker korumalı alan modu etkin olan Ubuntu/AppArmor ana sistemlerinde Codex app-server `workspace-write` kabuk yürütmesi, korumalı alan kapsayıcısının içinde ayrıcalıksız kullanıcı ad alanları gerektirir ve hizmet kullanıcısı bunları oluşturamadığında kabuk başlatılmadan önce başarısız olabilir. Docker korumalı alanının dışarı giden bağlantıları devre dışı olduğunda (`network: "none"`, varsayılan), ayrıcalıksız bir ağ ad alanı da gerekir. Yaygın belirtiler: `bwrap: setting up uid map: Permission denied` ve `bwrap: loopback: Failed RTM_NEWADDR: Operation not permitted`. `openclaw doctor` komutunu çalıştırın; Codex bwrap ad alanı yoklaması hatası bildirirse gerekli ad alanlarını OpenClaw hizmet işlemine veren bir AppArmor profilini tercih edin. `kernel.apparmor_restrict_unprivileged_userns=0`, güvenlik açısından ödünleri olan ana sistem genelinde bir geri dönüş seçeneğidir; yalnızca bu ana sistem güvenlik duruşu kabul edilebilirse kullanın.
</Warning>

### Korumalı alan tarayıcısı

- Tarayıcı aracı ihtiyaç duyduğunda korumalı alan tarayıcısı otomatik olarak başlatılır (CDP'nin erişilebilir olmasını sağlar). `agents.defaults.sandbox.browser.autoStart` (varsayılan `true`) ve `autoStartTimeoutMs` (varsayılan 12s) aracılığıyla yapılandırın.
- Korumalı alan tarayıcı kapsayıcıları, genel `bridge` ağı yerine özel bir Docker ağı (`openclaw-sandbox-browser`) kullanır. `agents.defaults.sandbox.browser.network` ile yapılandırın.
- `agents.defaults.sandbox.browser.cdpSourceRange`, bir CIDR izin listesiyle kapsayıcı sınırındaki CDP girişini kısıtlar (örneğin `172.21.0.1/32`).
- noVNC gözlemci erişimi varsayılan olarak parola korumalıdır; OpenClaw, yerel bir önyükleme sayfası sunan ve noVNC'yi URL parçasındaki parolayla (sorgu dizesinde veya üstbilgi günlüklerinde değil) açan kısa ömürlü bir belirteç URL'si oluşturur.
- `agents.defaults.sandbox.browser.allowHostControl` (varsayılan `false`), korumalı alandaki oturumların ana sistem tarayıcısını açıkça hedeflemesine olanak tanır.
- İsteğe bağlı izin listeleri `target: "custom"` kullanımını denetler: `allowedControlUrls`, `allowedControlHosts`, `allowedControlPorts`.

## SSH arka ucu

`exec`, dosya araçları ve medya okumalarını SSH ile erişilebilen herhangi bir makinede korumalı alanda çalıştırmak için `backend: "ssh"` kullanın.

```json5
{
  agents: {
    defaults: {
      sandbox: {
        mode: "all",
        backend: "ssh",
        scope: "session",
        workspaceAccess: "rw",
        ssh: {
          target: "user@gateway-host:22",
          workspaceRoot: "/tmp/openclaw-sandboxes",
          strictHostKeyChecking: true,
          updateHostKeys: true,
          identityFile: "~/.ssh/id_ed25519",
          certificateFile: "~/.ssh/id_ed25519-cert.pub",
          knownHostsFile: "~/.ssh/known_hosts",
          // Ya da yerel dosyalar yerine SecretRefs / satır içi içerikler kullanın:
          // identityData: { source: "env", provider: "default", id: "SSH_IDENTITY" },
          // certificateData: { source: "env", provider: "default", id: "SSH_CERTIFICATE" },
          // knownHostsData: { source: "env", provider: "default", id: "SSH_KNOWN_HOSTS" },
        },
      },
    },
  },
}
```

Varsayılanlar: `command: "ssh"`, `workspaceRoot: "/tmp/openclaw-sandboxes"`, `strictHostKeyChecking: true`, `updateHostKeys: true`.

- **Yaşam döngüsü**: OpenClaw, `sandbox.ssh.workspaceRoot` altında kapsam başına bir uzak kök oluşturur. Oluşturma veya yeniden oluşturma sonrasındaki ilk kullanımda, bu uzak çalışma alanına yerel çalışma alanından bir kez başlangıç verileri aktarır. Bundan sonra `exec`, `read`, `write`, `edit`, `apply_patch`, istem medya okumaları ve gelen medya hazırlama işlemleri SSH üzerinden doğrudan uzak çalışma alanında çalışır. OpenClaw, uzak değişiklikleri yerel çalışma alanına otomatik olarak eşitlemez.
- **Kimlik doğrulama materyali**: `identityFile`/`certificateFile`/`knownHostsFile` mevcut yerel dosyalara başvurur. `identityData`/`certificateData`/`knownHostsData`, normal gizli bilgiler çalışma zamanı anlık görüntüsü aracılığıyla çözümlenen, `0600` moduyla geçici dosyalara yazılan ve SSH oturumu sona erdiğinde silinen satır içi dizeleri veya SecretRefs değerlerini kabul eder. Aynı öğe için hem `*File` hem de `*Data` çeşidi ayarlanmışsa o oturumda `*Data` önceliklidir.
- **Uzak sistemin esas alınmasının sonuçları**: İlk başlangıç verisi aktarımından sonra uzak SSH çalışma alanı gerçek korumalı alan durumu hâline gelir. Başlangıç verisi aktarımı adımından sonra OpenClaw dışında ana sistemde yapılan yerel düzenlemeler, korumalı alanı yeniden oluşturana kadar uzak sistemde görünmez. `openclaw sandbox recreate`, kapsam başına uzak kökü siler ve sonraki kullanımda yerelden yeniden başlangıç verileri aktarır. Tarayıcı korumalı alanı bu arka uçta desteklenmez ve `sandbox.docker.*` ayarları bu arka uç için geçerli değildir.

## OpenShell arka ucu

Araçları OpenShell tarafından yönetilen uzak bir ortamda korumalı alanda çalıştırmak için `backend: "openshell"` kullanın. OpenShell, genel SSH arka ucuyla aynı SSH aktarımını ve uzak dosya sistemi köprüsünü yeniden kullanır; bunlara OpenShell yaşam döngüsünü (`sandbox create/get/delete/ssh-config`) ve isteğe bağlı bir `mirror` çalışma alanı eşitleme modunu ekler.

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
          mode: "remote", // mirror | remote
        },
      },
    },
  },
}
```

`mode: "mirror"` (varsayılan), yerel çalışma alanını kanonik tutar: OpenClaw, `exec` öncesinde yerel çalışma alanını sandbox'a eşitler ve sonrasında değişiklikleri geri eşitler. `mode: "remote"`, uzak çalışma alanını yerelden bir kez başlangıç verileriyle doldurur, ardından `exec`/`read`/`write`/`edit`/`apply_patch` işlemlerini geri eşitleme yapmadan doğrudan uzak çalışma alanında çalıştırır; başlangıç verileri aktarıldıktan sonra yapılan yerel düzenlemeler, `openclaw sandbox recreate` işlemini gerçekleştirene kadar görünmez. `scope: "agent"` veya `scope: "shared"` altında bu uzak çalışma alanı aynı kapsamda paylaşılır. Mevcut sınırlamalar: sandbox tarayıcısı henüz desteklenmemektedir ve `sandbox.docker.binds` bu arka uç için geçerli değildir.

`openclaw sandbox list`/`recreate`/prune işlemlerinin tümü OpenShell çalışma zamanlarını Docker çalışma zamanlarıyla aynı şekilde ele alır; prune mantığı arka uca duyarlıdır.

Tüm ön koşullar, yapılandırma başvurusu, çalışma alanı modu karşılaştırması ve yaşam döngüsü ayrıntıları için [OpenShell](/tr/gateway/openshell) sayfasına bakın.

## Çalışma alanı erişimi

`agents.defaults.sandbox.workspaceAccess`, sandbox'ın neleri görebileceğini denetler:

| Değer            | Davranış                                                                                  |
| ---------------- | ----------------------------------------------------------------------------------------- |
| `none` (varsayılan) | Araçlar, `~/.openclaw/sandboxes` altında yalıtılmış bir sandbox çalışma alanı görür.                    |
| `ro`             | Ajan çalışma alanını `/agent` konumuna salt okunur olarak bağlar (`write`/`edit`/`apply_patch` devre dışı bırakılır). |
| `rw`             | Ajan çalışma alanını `/workspace` konumuna okuma/yazma erişimiyle bağlar.                                    |

OpenShell arka ucunda `mirror` modu, exec çalıştırmaları arasında kanonik kaynak olarak yine yerel çalışma alanını kullanır; `remote` modu, ilk başlangıç verisi aktarımından sonra kanonik kaynak olarak uzak OpenShell çalışma alanını kullanır ve `workspaceAccess: "ro"`/`"none"` yazma davranışını aynı şekilde kısıtlamaya devam eder.

Gelen medya, etkin sandbox çalışma alanına (`media/inbound/*`) kopyalanır.

<Note>
**Skills**: `read` aracı sandbox kök dizinine bağlıdır. `workspaceAccess: "none"` ile OpenClaw, uygun becerileri okunabilmeleri için sandbox çalışma alanına (`.../skills`) yansıtır. `"rw"` ile çalışma alanı becerileri `/workspace/skills` konumundan okunabilir; uygun yönetilen, paketle gelen veya plugin becerileri ise oluşturulan salt okunur `/workspace/.openclaw/sandbox-skills/skills` yolunda somutlaştırılır.
</Note>

## Tek bir ajan için birden çok klasör

Sandbox'ta çalışan bir ajanın birincil çalışma alanından daha fazlasına ihtiyacı olduğunda Docker bind bağlamalarını kullanın. Her girdi, bir ana makine klasörünü açıkça belirtilmiş bir erişim moduyla bir container yoluna eşler:

```text
host-directory:container-directory:ro
host-directory:container-directory:rw
```

- `ro`, bağlanan klasörü sandbox içinde salt okunur yapar.
- `rw`, sandbox içindeki araçların ve süreçlerin ana makine klasörünü değiştirmesine izin verir.
- Container yolu, ajanın kullandığı yoldur. Ana makine yolları otomatik olarak açığa çıkarılmaz.

Bu örnek, `research` ajanına yazılabilir bir birincil çalışma alanı, `/reference` konumunda salt okunur başvuru materyali ve `/drafts` konumunda ayrı bir yazılabilir çıktı klasörü sağlar:

```json5
{
  agents: {
    defaults: {
      sandbox: {
        mode: "all",
        scope: "agent",
      },
    },
    list: [
      {
        id: "research",
        workspace: "/srv/openclaw/research-workspace",
        sandbox: {
          workspaceAccess: "rw",
          docker: {
            binds: ["/srv/shared/reference:/reference:ro", "/srv/shared/drafts:/drafts:rw"],
            // Bu kaynaklar ajan çalışma alanının dışında olduğu için gereklidir.
            dangerouslyAllowExternalBindSources: true,
          },
        },
      },
    ],
  },
}
```

`workspaceAccess` ve bind modları birbirinden bağımsızdır:

| Ayar                          | Denetlediği davranış                                                                    |
| -------------------------------- | --------------------------------------------------------------------------- |
| `workspaceAccess: "none"`        | Yalıtılmış bir sandbox çalışma alanı kullanır; ajan çalışma alanını açığa çıkarmaz.    |
| `workspaceAccess: "ro"`          | Ajan çalışma alanını `/agent` konumuna salt okunur olarak bağlar.                           |
| `workspaceAccess: "rw"`          | Ajan çalışma alanını `/workspace` konumuna okuma/yazma erişimiyle bağlar.                      |
| `docker.binds` girdisi `:ro`/`:rw` | Yalnızca yapılandırılmış container yolundaki ilgili ek ana makine klasörünü denetler. |

`workspaceAccess` değerini değiştirmek, ek bir bind bağlamasını `ro` modundan `rw` moduna veya tersine değiştirmez. Genel ve ajan başına `docker.binds` değerleri birleştirilir. Ajan başına bind bağlamaları için `scope: "agent"` veya `"session"` kullanmaya devam edin; `scope: "shared"`, ajan başına tüm Docker geçersiz kılmalarını yok sayar ve yalnızca genel bind bağlamalarını kullanır.

Bind bağlamaları, desteklenen çok klasörlü sınırdır; çünkü Docker, container'ın dosya sistemi görünümünü bağlama yalıtımıyla oluşturur ve `ro`/`rw` modu sandbox içindeki her sürece uygulanır. Bu sınır, her OpenClaw kod yolunda dosya yolu yetkilendirme denetimlerini çoğaltmadan `exec`, dosya sistemi araçları, alt süreçler ve kütüphaneleri kapsar. Ana makine tarafındaki bir yol izin listesi, izin verilen bir kabuk veya bağımlılık dosyalara doğrudan erişebildiğinde aynı eksiksiz sınırı sağlayamaz.

İsteğe bağlı `dangerouslyAllowExternalBindSources`, yalnızca çalışma alanı köklerinin dışındaki kaynaklara izin verir. OpenClaw'ın engellenen sistem, kimlik bilgisi, Docker soketi, sembolik bağlantı üst dizini veya ayrılmış hedef denetimlerini devre dışı bırakmaz. En küçük klasörü tercih edin, yazma gerekmediği sürece `ro` kullanın ve bağlamaları değiştirdikten sonra sandbox'ı yeniden oluşturun:

```bash
openclaw sandbox recreate --agent research
```

### Diğer bind davranışları

`agents.defaults.sandbox.docker.binds`, genel bağlamaları yapılandırır. Biçim aynı `host:container:mode` biçimidir (örneğin `"/home/user/source:/source:rw"`).

`agents.defaults.sandbox.browser.binds`, ek ana makine dizinlerini yalnızca **sandbox tarayıcısı** container'ına bağlar. Ayarlandığında (`[]` dâhil), tarayıcı container'ı için `docker.binds` değerinin yerini alır; belirtilmediğinde tarayıcı container'ı `docker.binds` değerine geri döner.

```json5
{
  agents: {
    defaults: {
      sandbox: {
        docker: {
          binds: ["/home/user/source:/source:ro", "/var/data/myapp:/data:ro"],
        },
      },
    },
    list: [
      {
        id: "build",
        sandbox: {
          docker: {
            binds: ["/mnt/cache:/cache:rw"],
          },
        },
      },
    ],
  },
}
```

<Warning>
**Bind güvenliği**

- Bind bağlamaları sandbox dosya sistemini atlar: ana makine yollarını belirlediğiniz modla (`:ro` veya `:rw`) açığa çıkarır.
- OpenClaw, tehlikeli bind kaynaklarını varsayılan olarak engeller: sistem yolları (`/etc`, `/proc`, `/sys`, `/dev`, `/root`, `/boot`), Docker soketi dizinleri (`/run`, `/var/run` ve bunların `docker.sock` varyantları) ve yaygın ana dizin kimlik bilgisi kökleri (`~/.aws`, `~/.cargo`, `~/.config`, `~/.docker`, `~/.gnupg`, `~/.netrc`, `~/.npm`, `~/.ssh`).
- Doğrulama, kaynak yolunu normalleştirir ve ardından engellenen yolları ve izin verilen kökleri yeniden denetlemeden önce en derindeki mevcut üst dizin üzerinden tekrar çözümler; böylece son yaprak henüz mevcut olmasa bile sembolik bağlantı üst dizininden kaçışlar güvenli biçimde başarısız olur (ör. `run-link` burayı gösteriyorsa `/workspace/run-link/new-file` yine `/var/run/...` olarak çözümlenir).
- Ayrılmış container bağlama noktalarını (`/workspace`, `/agent`) gölgeleyen bind hedefleri de varsayılan olarak engellenir; `agents.defaults.sandbox.docker.dangerouslyAllowReservedContainerTargets: true` ile geçersiz kılın.
- Çalışma alanı/ajan çalışma alanı izin listesindeki köklerin dışındaki bind kaynakları varsayılan olarak engellenir; `agents.defaults.sandbox.docker.dangerouslyAllowExternalBindSources: true` ile geçersiz kılın. İzin verilen kökler de aynı şekilde kanonikleştirilir; bu nedenle sembolik bağlantı çözümlemesinden önce yalnızca izin listesinin içindeymiş gibi görünen bir yol, izin verilen köklerin dışında olduğu için yine reddedilir.
- Hassas bağlamalar (gizli bilgiler, SSH anahtarları, hizmet kimlik bilgileri), kesinlikle gerekli olmadıkça `:ro` olmalıdır.
- Çalışma alanına yalnızca okuma erişimi gerekiyorsa `workspaceAccess: "ro"` ile birlikte kullanın; bind modları bağımsız kalır.
- Bind bağlamalarının araç politikası ve yükseltilmiş exec ile nasıl etkileşime girdiğini öğrenmek için [Sandbox, Araç Politikası ve Yükseltilmiş Erişim Karşılaştırması](/tr/gateway/sandbox-vs-tool-policy-vs-elevated) sayfasına bakın.

</Warning>

## İmajlar ve kurulum

Varsayılan Docker imajı: `openclaw-sandbox:bookworm-slim`

<Note>
**Kaynak kod deposu ile npm kurulumu karşılaştırması**

`scripts/sandbox-setup.sh`, `scripts/sandbox-common-setup.sh` ve `scripts/sandbox-browser-setup.sh` yardımcı betikleri yalnızca bir [kaynak kod deposundan](https://github.com/openclaw/openclaw) çalıştırıldığında kullanılabilir. npm paketine dâhil edilmezler.

OpenClaw'ı `npm install -g openclaw` aracılığıyla yüklediyseniz bunun yerine aşağıda gösterilen satır içi `docker build` komutlarını kullanın.
</Note>

<Steps>
  <Step title="Varsayılan imajı oluşturun">
    Kaynak kod deposundan:

    ```bash
    scripts/sandbox-setup.sh
    ```

    npm kurulumundan (kaynak kod deposu gerekmez):

    ```bash
    docker build -t openclaw-sandbox:bookworm-slim - <<'DOCKERFILE'
    FROM debian:bookworm-slim
    ENV DEBIAN_FRONTEND=noninteractive
    RUN apt-get update && apt-get install -y --no-install-recommends \
      bash ca-certificates curl git jq python3 ripgrep \
      && rm -rf /var/lib/apt/lists/*
    RUN useradd --create-home --shell /bin/bash sandbox
    USER sandbox
    WORKDIR /home/sandbox
    CMD ["sleep", "infinity"]
    DOCKERFILE
    ```

    Varsayılan imaj Node içermez. Bir beceri Node'a (veya başka çalışma zamanlarına) ihtiyaç duyuyorsa özel bir imaj oluşturun ya da `sandbox.docker.setupCommand` aracılığıyla yükleyin (ağ çıkışı + yazılabilir kök dizin + root kullanıcısı gerektirir).

    `openclaw-sandbox:bookworm-slim` eksik olduğunda OpenClaw, sessizce düz `debian:bookworm-slim` kullanmaz. Varsayılan imajı hedefleyen sandbox çalıştırmaları, imajı oluşturana kadar bir oluşturma talimatıyla hemen başarısız olur; çünkü paketle gelen imaj, sandbox yazma/düzenleme yardımcıları için `python3` içerir.

  </Step>
  <Step title="İsteğe bağlı: ortak imajı oluşturun">
    Yaygın araçları (örneğin `curl`, `jq`, Node 24, pnpm, `python3` ve `git`) içeren daha işlevsel bir sandbox imajı için:

    Kaynak kod deposundan:

    ```bash
    scripts/sandbox-common-setup.sh
    ```

    npm kurulumundan, önce varsayılan imajı oluşturun (yukarıya bakın), ardından depodaki [`scripts/docker/sandbox/Dockerfile.common`](https://github.com/openclaw/openclaw/blob/main/scripts/docker/sandbox/Dockerfile.common) dosyasını kullanarak ortak imajı bunun üzerine oluşturun.

    Ardından `agents.defaults.sandbox.docker.image` değerini `openclaw-sandbox-common:bookworm-slim` olarak ayarlayın.

  </Step>
  <Step title="İsteğe bağlı: sandbox tarayıcı imajını oluşturun">
    Kaynak kod deposundan:

    ```bash
    scripts/sandbox-browser-setup.sh
    ```

    npm kurulumundan, depodaki [`scripts/docker/sandbox/Dockerfile.browser`](https://github.com/openclaw/openclaw/blob/main/scripts/docker/sandbox/Dockerfile.browser) dosyasını kullanarak oluşturun.

  </Step>
</Steps>

Docker sandbox container'ları varsayılan olarak **ağ olmadan** çalışır. `agents.defaults.sandbox.docker.network` ile geçersiz kılın.

<AccordionGroup>
  <Accordion title="Sandbox tarayıcısının Chromium varsayılanları">
    Paketle gelen sandbox tarayıcı imajı, container ortamındaki iş yükleri için temkinli Chromium başlangıç bayrakları uygular:

    - `--remote-debugging-address=127.0.0.1`
    - `--remote-debugging-port=<derived from OPENCLAW_BROWSER_CDP_PORT>`
    - `--user-data-dir=${HOME}/.chrome`
    - `--no-first-run`
    - `--no-default-browser-check`
    - `--disable-dev-shm-usage`
    - `--disable-background-networking`
    - `--disable-breakpad`
    - `--disable-crash-reporter`
    - `--no-zygote`
    - `--metrics-recording-only`
    - `--password-store=basic`
    - `--use-mock-keychain`
    - `--headless=new`, `browser.headless` etkinleştirildiğinde.
    - `--no-sandbox --disable-setuid-sandbox`, `browser.noSandbox` etkinleştirildiğinde.
    - Varsayılan olarak `--disable-3d-apis`, `--disable-gpu`, `--disable-software-rasterizer`; bu grafik sağlamlaştırma bayrakları, GPU desteği olmayan container'lara yardımcı olur. İş yükünüz WebGL veya diğer 3B özellikleri gerektiriyorsa `OPENCLAW_BROWSER_DISABLE_GRAPHICS_FLAGS=0` değerini ayarlayın.
    - Varsayılan olarak `--disable-extensions`; uzantılara dayalı akışlar için `OPENCLAW_BROWSER_DISABLE_EXTENSIONS=0` değerini ayarlayın.
    - Varsayılan olarak `--renderer-process-limit=2`; `OPENCLAW_BROWSER_RENDERER_PROCESS_LIMIT=<N>` tarafından denetlenir; `0` ise Chromium'un varsayılanını korur.

    Farklı bir çalışma zamanı profiline ihtiyacınız varsa özel bir tarayıcı görüntüsü kullanın ve kendi giriş noktanızı sağlayın. Yerel (container dışı) Chromium profillerinde ek başlangıç bayrakları eklemek için `browser.extraArgs` kullanın.

  </Accordion>
  <Accordion title="Ağ güvenliği varsayılanları">
    - `network: "host"` engellenir.
    - `network: "container:<id>"` varsayılan olarak engellenir (ad alanına katılarak kuralları aşma riski).
    - Acil durum geçersiz kılma seçeneği: `agents.defaults.sandbox.docker.dangerouslyAllowContainerNamespaceJoin: true`.

  </Accordion>
</AccordionGroup>

Docker kurulumları ve container'laştırılmış Gateway burada bulunur: [Docker](/tr/install/docker)

Docker Gateway dağıtımlarında `scripts/docker/setup.sh`, korumalı alan yapılandırmasını başlatabilir. Bu yolu etkinleştirmek için `OPENCLAW_SANDBOX=1` (veya `true`/`yes`/`on`) değerini ayarlayın. Soket konumunu `OPENCLAW_DOCKER_SOCKET` ile geçersiz kılın. Tam kurulum ve ortam değişkeni başvurusu: [Docker](/tr/install/docker#agent-sandbox).

## setupCommand (tek seferlik container kurulumu)

`setupCommand`, korumalı alan container'ı oluşturulduktan sonra **bir kez** çalışır (her çalıştırmada değil). Container içinde `sh -lc` aracılığıyla yürütülür.

Yollar:

- Genel: `agents.defaults.sandbox.docker.setupCommand`
- Agent başına: `agents.entries.*.sandbox.docker.setupCommand`

<AccordionGroup>
  <Accordion title="Yaygın sorunlar">
    - Varsayılan `docker.network`, `"none"` değeridir (dışarıya trafik yoktur), dolayısıyla paket kurulumları başarısız olur.
    - `docker.network: "container:<id>"`, `dangerouslyAllowContainerNamespaceJoin: true` gerektirir ve yalnızca acil durumlar içindir.
    - `readOnlyRoot: true` yazma işlemlerini engeller; `readOnlyRoot: false` değerini ayarlayın veya özel bir görüntüyü önceden oluşturun.
    - Paket kurulumları için `user` root olmalıdır (`user` değerini atlayın veya `user: "0:0"` olarak ayarlayın).
    - Korumalı alan yürütmesi, ana makinenin `process.env` değerini **devralmaz**. Skill API anahtarları için `agents.defaults.sandbox.docker.env` (veya özel bir görüntü) kullanın.
    - `agents.defaults.sandbox.docker.env` içindeki değerler, açık Docker container ortam değişkenleri olarak aktarılır. Docker daemon erişimi olan herkes, bunları `docker inspect` gibi Docker meta veri komutlarıyla inceleyebilir. Bu meta veri ifşası kabul edilebilir değilse özel bir görüntü, bağlanmış gizli dosya veya başka bir gizli bilgi sağlama yolu kullanın.

  </Accordion>
</AccordionGroup>

## Araç politikası ve kaçış yolları

Araç izin/reddetme politikaları, korumalı alan kurallarından önce uygulanmaya devam eder. Bir araç genel olarak veya agent başına reddedilmişse korumalı alan kullanımı onu yeniden etkinleştirmez.

`tools.elevated`, `exec` aracını korumalı alan dışında çalıştıran açık bir kaçış yoludur (varsayılan olarak `gateway`; exec hedefi `node` olduğunda ise `node`). `/exec` yönergeleri yalnızca yetkili gönderenler için geçerlidir ve oturum başına kalıcıdır; `exec` özelliğini kesin olarak devre dışı bırakmak için araç politikası reddini kullanın (bkz. [Korumalı Alan, Araç Politikası ve Yükseltilmiş Mod Karşılaştırması](/tr/gateway/sandbox-vs-tool-policy-vs-elevated)).

Hata ayıklama:

- `openclaw sandbox list`; korumalı alan container'larını, durumu, görüntü eşleşmesini, yaşı, boşta kalma süresini ve ilişkili oturumu/agent'ı gösterir.
- `openclaw sandbox explain [--session <key>] [--agent <id>]`; geçerli korumalı alan modunu, ana makine çalışma alanını, çalışma zamanı çalışma dizinini, Docker bağlamalarını, araç politikasını ve düzeltme yapılandırma anahtarlarını inceler. `workspaceRoot` alanı, yapılandırılmış korumalı alan kökü olarak kalır; `effectiveHostWorkspaceRoot`, etkin çalışma alanının gerçekte nerede bulunduğunu gösterir.
- `openclaw sandbox recreate [--all | --session <key> | --agent <id>] [--browser] [--force]`, bir sonraki kullanımda geçerli yapılandırmayla yeniden oluşturulmaları için container'ları/ortamları kaldırır.
- “Bu neden engellendi?” yaklaşımı için [Korumalı Alan, Araç Politikası ve Yükseltilmiş Mod Karşılaştırması](/tr/gateway/sandbox-vs-tool-policy-vs-elevated) bölümüne bakın.

## Çoklu agent geçersiz kılmaları

Her agent, korumalı alanı ve araçları geçersiz kılabilir: `agents.entries.*.sandbox` ve `agents.entries.*.tools` (ayrıca korumalı alan araç politikası için `agents.entries.*.tools.sandbox.tools`). Öncelik sırası için [Çoklu Agent Korumalı Alanı ve Araçları](/tr/tools/multi-agent-sandbox-tools) bölümüne bakın.

## En küçük etkinleştirme örneği

```json5
{
  agents: {
    defaults: {
      sandbox: {
        mode: "non-main",
        scope: "session",
        workspaceAccess: "none",
      },
    },
  },
}
```

## İlgili konular

- [Çoklu Agent Korumalı Alanı ve Araçları](/tr/tools/multi-agent-sandbox-tools) -- agent başına geçersiz kılmalar ve öncelik sırası
- [OpenShell](/tr/gateway/openshell) -- yönetilen korumalı alan arka ucu kurulumu, çalışma alanı modları ve yapılandırma başvurusu
- [Korumalı alan yapılandırması](/tr/gateway/config-agents#agentsdefaultssandbox)
- [Korumalı Alan, Araç Politikası ve Yükseltilmiş Mod Karşılaştırması](/tr/gateway/sandbox-vs-tool-policy-vs-elevated) -- “Bu neden engellendi?” sorusunda hata ayıklama
- [Güvenlik](/tr/gateway/security)
